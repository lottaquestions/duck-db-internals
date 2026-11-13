# Chapter 7, Part 14 — `UNUSED_COLUMNS` and `COLUMN_LIFETIME`

Both of these passes are, on first glance, "delete columns nobody needs." That
description is accurate enough to make it easy to assume they're the same
pass split across two files for no particular reason. They are not. Read
closely, `src/optimizer/remove_unused_columns.cpp` and
`src/optimizer/column_lifetime_analyzer.cpp` answer two genuinely different
questions:

- **`UNUSED_COLUMNS`** (`RemoveUnusedColumns`, `optimizer.cpp:401`/`:517`,
  §7.4's pass #23/#38) answers: **is this column, in its entirety, read by
  *anything* in the rest of the query — yes or no?** If the answer is no, the
  column is deleted outright: erased from a `LogicalGet`'s `column_ids`
  (meaning the storage layer never opens that column's segments at all — see
  §14.1.3 and the forward reference to Chapter 17 §17.2.2), erased from a
  `LogicalProjection`'s expression list, erased from an aggregate's output
  expressions, and so on, with every remaining `BoundColumnRefExpression`
  that pointed at a *later* column in the same list re-indexed down to close
  the gap.
- **`COLUMN_LIFETIME`** (`ColumnLifetimeAnalyzer`, `optimizer.cpp:419`/`:525`,
  §7.4's pass #26/#39) answers a strictly narrower question about columns
  that already survived `UNUSED_COLUMNS` — i.e. columns that *are* read by
  something, somewhere: **past which operator, exactly, does this column stop
  being needed?** It never deletes a column from any operator's own
  expression list. It only ever installs (or extends) a `projection_map` —
  `LogicalFilter::projection_map`, `LogicalOrder::projection_map`,
  `LogicalComparisonJoin::left_projection_map`/`right_projection_map` — that
  tells that *one specific operator* which of its child's columns to actually
  carry forward into its own output, letting a column that a filter or a join
  condition still needs internally stop propagating immediately afterward
  instead of riding all the way to the final projection.

Put concretely: if a column is used by an early `WHERE` predicate and by
nothing else in the whole query, `UNUSED_COLUMNS` cannot delete it — it
*is* used, somewhere, so by `UNUSED_COLUMNS`'s binary yes/no test it survives
completely intact, all the way down to (and including) the base-table scan.
`COLUMN_LIFETIME` is the pass that then notices the filter is the *last*
operator to need it, and arranges for the filter's own `projection_map` to
stop forwarding it past that point, so every operator above the filter simply
never carries it. This is exactly the hypothesis this chapter set out to
verify — reading both source files confirms it precisely, with one
refinement worth flagging up front: `COLUMN_LIFETIME` doesn't compute an
abstract "lifetime interval" per column and then decide a drop-off point in
the abstract. It's much more mechanical than that — for a small, fixed set
of operator types (`LogicalFilter`, `LogicalOrder`, equality
`LogicalComparisonJoin`s, and `LogicalDistinct` with `DISTINCT ON`), it asks
"of my child's output columns, which ones does *everything I've seen so far,
above me* actually reference?", and prunes at exactly that operator, one
level at a time. There's no separate global lifetime-interval data
structure — the "interval" only ever exists implicitly, as the accumulated
answer to that same local question asked again at every qualifying operator
on the way down.

## 14.1 `UNUSED_COLUMNS` — binary reachability, all the way to the scan

### 14.1.1 The two-mode traversal: `ANALYZE` then `APPLY`, per subtree

`RemoveUnusedColumns` (`src/optimizer/remove_unused_columns.cpp`,
`src/include/duckdb/optimizer/remove_unused_columns.hpp`) is a subclass of
`BaseColumnPruner`, which is itself a `LogicalOperatorVisitor`. Its
constructor makes the traversal direction explicit right away:

```cpp
// src/optimizer/remove_unused_columns.cpp:60
RemoveUnusedColumns::RemoveUnusedColumns(Optimizer &optimizer)
    : optimizer(optimizer), binder(optimizer.binder), context(optimizer.context), everything_referenced(true),
      root(*this) {
}
```

The traversal is **top-down**, and `everything_referenced` starting `true`
at the root is the mechanism that makes that concrete: the plan's root
operator (whatever `SELECT`, `INSERT ... RETURNING`, or `COPY` sits at the
very top) implicitly needs every column it's handed, so nothing gets pruned
there. As the visitor descends (`RemoveUnusedColumns::VisitOperator`,
`remove_unused_columns.cpp:496`), each operator type gets a case in one large
switch, and most cases follow the same shape: figure out which of *this*
operator's own output columns are actually referenced (using a member map,
`column_references : column_binding_map_t<ReferencedColumn>`, populated by
everything visited *above* this point in the tree), delete the ones that
aren't, fix up the ones that are, and then recurse into the child with a
*fresh* `column_references` map — because a child's `ColumnBinding`
namespace (its `TableIndex`) is generally unrelated to its parent's, so
"what's referenced" has to be recomputed relative to each operator's own
output, not carried as one giant cross-tree set.

The two-mode aspect (`RemoveUnusedColumnsMode::ANALYZE` vs. `::APPLY`,
`remove_unused_columns.hpp:88`) is a local wrinkle inside that top-down walk,
not a second top-to-bottom traversal of the whole tree. `LOGICAL_PROJECTION`
is the clearest example (`remove_unused_columns.cpp:743`):

```cpp
// src/optimizer/remove_unused_columns.cpp:743
case LogicalOperatorType::LOGICAL_PROJECTION: {
    auto &proj = op.Cast<LogicalProjection>();
    if (analyze) {
        RemoveUnusedColumns remove(*this, false);
        for (idx_t i = 0; i < proj.expressions.size(); i++) {
            auto binding = ColumnBinding(proj.table_index, ProjectionIndex(i));
            if (everything_referenced || column_references.find(binding) != column_references.end()) {
                remove.VisitExpression(&proj.expressions[i]);
            }
        }
        remove.VisitOperator(op.children[0]);
        return;
    }
    if (!everything_referenced) {
        CheckPushdownExtract(op);
        auto old_expression_count = proj.expressions.size();
        ClearUnusedExpressions(proj.expressions, proj.table_index);
        RewriteExpressions(proj, old_expression_count);
        // ...
    }
    // then recurse into the children of this projection
    RemoveUnusedColumns remove(*this, false);
    remove.VisitOperatorExpressions(op);
    remove.VisitOperator(op.children[0]);
    return;
}
```

In `APPLY` mode (the mode every top-level invocation actually runs in —
`RemoveUnusedColumnsMode mode = RemoveUnusedColumnsMode::APPLY;` is the
default member initializer at `remove_unused_columns.hpp:150`, and no code
path in `optimizer.cpp` ever constructs one in `ANALYZE` mode directly), a
projection first deletes any of its own expressions nothing above needs
(`ClearUnusedExpressions` — §14.1.2), then visits the *surviving* expressions
(`VisitOperatorExpressions`, which walks every kept expression and, via
`BaseColumnPruner::VisitReplace(BoundColumnRefExpression&, ...)`, records
each `BoundColumnRefExpression` it finds into a *new* `column_references`
map scoped to the child), then recurses into the child using that new map.
`ANALYZE` mode only appears as a small, local **lookahead**, spawned by a
handful of operators (`LOGICAL_PROJECTION`, `LOGICAL_AGGREGATE_AND_GROUP_BY`,
`LOGICAL_UNION`, `LOGICAL_MATERIALIZED_CTE`, `LOGICAL_RECURSIVE_CTE`) that
need to know something about a *deeper* subtree's actual column usage before
they can safely decide what to keep in their *own* expression list — most
concretely, a `MATERIALIZED_CTE`'s left-hand side (the CTE's definition)
can't be pruned until every reference site (every `LogicalCTERef` anywhere
in the right-hand side, however deep) has reported back which of the CTE's
columns it actually reads; `ANALYZE` mode is how that lookahead is performed
without yet mutating anything. So: one overall top-down pass, with a small,
scoped, throwaway analysis sub-walk used by a few operator types to see
"what does my subtree need" before committing to a decision about their own
output — not two full passes over the whole tree.

### 14.1.2 `ClearUnusedExpressions` — deleting a column *is* re-indexing everything after it

The task hypothesis going in was that a separate helper (`ColumnBindingReplacer`)
is what shifts every surviving column's index down once an earlier column is
deleted. Reading the source shows this is only half right, and the half
that's right belongs to a different helper.

`RemoveUnusedColumns::ClearUnusedExpressions` (a template, `remove_unused_columns.cpp:232`)
is what actually does the deleting-and-reindexing, for the operator whose own
list is being pruned, in the same pass, using information the top-down walk
already collected:

```cpp
// src/optimizer/remove_unused_columns.cpp:232 (elided)
template <class T>
void RemoveUnusedColumns::ClearUnusedExpressions(vector<T> &list, TableIndex table_idx, bool replace) {
	D_ASSERT(mode == RemoveUnusedColumnsMode::APPLY);
	idx_t new_col_idx = 0;
	// entries that are referred to are moved to the front of the list, the rest is erased at the end
	idx_t kept_count = 0;
	for (idx_t col_idx = 0; col_idx < list.size(); col_idx++) {
		auto current_binding = ColumnBinding(table_idx, ProjectionIndex(col_idx));
		// ...
		auto entry = column_references.find(current_binding);
		if (entry == column_references.end()) {
			// ...
			// this entry is not referred to, erase it from the set of expressions
			continue;
		}
		if (kept_count != col_idx) {
			list[kept_count] = std::move(list[col_idx]);
		}
		kept_count++;
		// ...
		bool should_replace = false;
		if (col_idx != new_col_idx) {
			should_replace = true;
		}
		// ...
		if (should_replace) {
			// column is used but the ColumnBinding has changed because of removed columns
			auto new_binding = ColumnBinding(table_idx, ProjectionIndex(new_col_idx));
			auto created_bindings = ReplaceBinding(current_binding, new_binding);
			// ...
			new_col_idx += created_bindings;
		}
		// ...
	}
	list.erase(list.begin() + NumericCast<int64_t>(kept_count), list.end());
}
```

> **Note:** the loop does not erase each unused entry on the spot. Calling
> `list.erase_at(col_idx)` once per dropped column would shift every later
> entry left each time, so pruning many columns from a wide list would cost
> quadratic time. Instead the loop moves each kept entry to the front
> (`kept_count`) and cuts off the tail with one `erase` at the end. Because
> nothing moves during the loop, `col_idx` is always the entry's original
> position.

Walk this at the level of "why does this work": `column_references` was
populated *before* this operator's own list is touched, by everything above
it in the tree — it's a map from this operator's `ColumnBinding`s to a
`ReferencedColumn`, and critically, `ReferencedColumn::bindings` is
`vector<reference<BoundColumnRefExpression>>` (`remove_unused_columns.hpp:65`)
— **live references to the actual `BoundColumnRefExpression` objects sitting
in ancestor operators' expression trees that pointed at this column.** So
when `ClearUnusedExpressions` decides column index 3 survives but needs to
become index 1 (because indices 1 and 2 were deleted), it doesn't need to
walk back up the tree to find who referenced index 3 — `BaseColumnPruner::ReplaceBinding`
(`remove_unused_columns.cpp:207`) already has that list on hand, and mutates
each one directly:

```cpp
// src/optimizer/remove_unused_columns.cpp:207
idx_t BaseColumnPruner::ReplaceBinding(ColumnBinding current_binding, ColumnBinding new_binding) {
	auto colrefs = column_references.find(current_binding);
	// ...
	for (auto &colref_p : col.bindings) {
		auto &colref = colref_p.get();
		colref.BindingMutable() = new_binding;
	}
	// ...
}
```

That's the actual re-indexing mechanism for *this* operator's own pruning
decision: direct, targeted mutation of exactly the `BoundColumnRefExpression`
nodes the top-down analysis already found, using bindings gathered on the
way down — no generic tree-wide rewrite pass required, because the pass
never lost track of *where* every reference to this column lives.

So what is `ColumnBindingReplacer` (`src/optimizer/column_binding_replacer.cpp`)
actually for? Reading it end to end: it's a small, **general-purpose**
utility — a `LogicalOperatorVisitor` holding a flat list of
`ReplacementBinding{old_binding, new_binding}` pairs
(`column_binding_replacer.hpp`), whose `VisitReplace(BoundColumnRefExpression&, ...)`
(`column_binding_replacer.cpp:159`) rewrites any matching binding it finds,
anywhere in whatever subtree it's pointed at:

```cpp
// src/optimizer/column_binding_replacer.cpp:216 (elided)
unique_ptr<Expression> ColumnBindingReplacer::VisitReplace(BoundColumnRefExpression &expr,
                                                           unique_ptr<Expression> *expr_ptr) {
	if (use_index) {
		auto replacement = FindReplacement(expr.Binding());
		if (replacement) {
			expr.BindingMutable() = replacement->new_binding;
			// ...
		}
		return nullptr;
	}
	for (auto &replacement : replacement_bindings) {
		if (expr.Binding() != replacement.old_binding) {
			continue;
		}
		expr.BindingMutable() = replacement.new_binding;
		// ...
	}
	return nullptr;
}
```

> **Two speed-ups and one traversal detail** in `ColumnBindingReplacer`:
>
> - **A lookup index.** `BuildIndex()` (`column_binding_replacer.cpp:124`)
>   builds a hash map from each `old_binding` to its slot in
>   `replacement_bindings`. With the index, each column reference costs one
>   hash lookup instead of a scan of the whole list. The index is only used
>   when it gives the same answer as the scan. The scan applies the
>   replacements in order, so `a → b` followed by `b → c` turns `a` into `c`.
>   A single lookup cannot do that. So if any `new_binding` is also an
>   `old_binding`, `BuildIndex` gives up and the scan stays.
> - **Scoped replacement graphs.** `BindingReplacementGraph::ScopedToBoundary`
>   (`column_binding_replacer.cpp:331`) trims a graph of replacements down
>   to the edges that can still be seen at one operator's output. Without
>   this, the delim-join CTE rewriter would merge every child's graph into its
>   parent's, and the graphs would grow with every level. For a plan with 200
>   columns and 20 correlated `EXISTS` subqueries, that is the difference
>   between about 47 s and 0.4 s in a debug build.
> - **No projection-map remapping while recursing.** `VisitOperator`
>   (`column_binding_replacer.cpp:195`) walks `op.children` itself. It does
>   not call `VisitOperatorChildren`, which would also remap projection
>   maps. The comment says why: *"Binding replacement keeps child
>   layouts intact, so projection maps are left untouched"*.
>
> None of this changes *which* bindings get replaced. The role of the class
> is described next.

`RemoveUnusedColumns` reaches for it in exactly the cases where the direct
`ReplaceBinding` mechanism above *doesn't* apply, because the rewrite needs
to cross a boundary the live-reference map doesn't reach across. The clearest
example is materialized-CTE pruning
(`LOGICAL_MATERIALIZED_CTE`, `remove_unused_columns.cpp:845`): once the CTE's
left-hand side (its definition) has some columns dropped and renumbered, every
`LogicalCTERef` on the right-hand side — however many times the CTE is
scanned, however deep each scan site is buried — needs its own bindings
patched to match. That renumbering is computed by a separate small visitor,
`CTERefPruner` (`remove_unused_columns.hpp:191`), which walks the right-hand
side once and produces the exact `ReplacementBinding` list; a
`ColumnBindingReplacer` is then built from that list and run over the
right-hand side to apply it everywhere at once:

```cpp
// src/optimizer/remove_unused_columns.cpp:946 (comments elided)
CTERefPruner cte_ref_pruner(cte.table_index, referenced_columns_in_rhs);
cte_ref_pruner.VisitOperator(*cte.children[1]);
for (auto &replacement : cte_ref_pruner.binding_replacements) {
	root.projection_map_replacements[replacement.old_binding] = {replacement.new_binding};
	RecordSecureViewReplacement(replacement.old_binding, replacement.new_binding);
}
ApplySecureViewReplacements();

ColumnBindingReplacer column_binding_replacer;
column_binding_replacer.replacement_bindings = cte_ref_pruner.binding_replacements;
column_binding_replacer.VisitOperator(*cte.children[1]);
```

The recursive-CTE path (`RewriteRecursiveCTEReferences`, `remove_unused_columns.cpp:170`)
follows the identical shape. So, to state the correction plainly: the
hypothesis that `ColumnBindingReplacer` is *the* mechanism behind "shift
every later reference down by one" for an ordinary pruned projection or
filter is not what the source shows — that reindexing is handled inline, via
the live `bindings` references already sitting in `column_references`.
`ColumnBindingReplacer` is the tool reached for when the set of expressions
that need patching can't be enumerated that way in advance — because they
live on the far side of a CTE reference, materialized-view boundary, or (as
`column_lifetime_analyzer.cpp` reuses it for, see §14.2.3) an entirely
separate debug-only verification pass — and so has to be found by a second,
independent tree walk instead.

### 14.1.3 `LogicalGet` — where this pass earns its keep

Every other operator `UNUSED_COLUMNS` prunes saves *some* CPU work — one
fewer expression to evaluate in a projection, one fewer key to hash in an
aggregate. `LogicalGet` is qualitatively different, because of what a
`LogicalGet`'s `column_ids` field actually controls: not "compute this
expression or don't," but "ask the storage layer to touch this column's
on-disk bytes, or don't ask at all." `RemoveUnusedColumns::RemoveColumnsFromLogicalGet`
(`remove_unused_columns.cpp:1301`) is the function that turns a pruning
decision into that instruction:

```cpp
// src/optimizer/remove_unused_columns.cpp:1301 (elided)
void RemoveUnusedColumns::RemoveColumnsFromLogicalGet(LogicalGet &get, unique_ptr<LogicalOperator> &op_ref) {
	if (everything_referenced) {
		return;
	}
	if (!get.function.projection_pushdown) {
		return;
	}
	// ... build new_column_ids from whichever of old_column_ids survive column_references ...
	if (new_column_ids.empty()) {
		// this generally means we are only interested in whether or not anything exists in the table
		// (e.g. EXISTS(SELECT * FROM tbl)) in this case, we just scan the row identifier column as it
		// means we do not need to read any of the columns
		auto any_column = get.GetAnyColumn();
		original_ids.emplace_back(any_column);
		new_column_ids.emplace_back(any_column);
	}
	// ...
	get.SetColumnIds(std::move(new_column_ids));
```

Two things worth pulling out of this. First, the guard
`if (!get.function.projection_pushdown) return;` — pruning a `LogicalGet`'s
column list is conditional on the underlying `TableFunction` actually
*supporting* projection pushdown at all (real tables and Parquet/CSV readers
do; some table functions don't expose column-level narrowing and simply
return everything every time regardless of what the optimizer wants). Second,
the `new_column_ids.empty()` fallback: a query like `EXISTS(SELECT * FROM tbl)`
needs to know only whether *any* row exists, not the value of any column —
`UNUSED_COLUMNS` correctly prunes every real column away, and rather than
leave the scan with zero columns (which several downstream assumptions don't
tolerate), it falls back to `get.GetAnyColumn()`, a synthetic row-identifier
column that lets the scan still report row existence/count without reading a
single real column's bytes off disk.

This is exactly the mechanism Chapter 17 §17.2.2 describes from the storage
side: because a `ColumnData`'s segments for `l_comment` are a physically
separate byte range from `l_quantity`'s segments, a `LogicalGet` whose
`column_ids` never lists `l_comment` means the scan operator never opens
that column's segments, never decompresses them, and — if they were ever
paged out — never even pages them back in from disk. `UNUSED_COLUMNS`
reaching all the way down to a `LogicalGet` is the one point in this entire
pipeline where a purely logical, syntactic decision ("nothing references this
column") turns directly into a **physical I/O reduction**, not just fewer
CPU cycles spent on an intermediate row. Every other pruning decision this
chapter covers saves work on rows already in memory; this one can mean bytes
never leave the disk (or the buffer manager's cache) at all.

The rest of `RemoveColumnsFromLogicalGet` is bookkeeping to keep that
decision consistent: pushed-down `table_filters` reference columns by
`ProjectionIndex` into the *old* `column_ids` list, so once `column_ids` is
renumbered, every `TableFilterSet` entry (and every multi-column expression
filter) has to be remapped to the *new* indices
(`old_to_new_pos`, `remapped_filters`, `remove_unused_columns.cpp:1436`–`1465`) —
and `get.projection_ids` (which of the *filtered* columns actually need to
survive into the operator's own output, as opposed to being read purely to
satisfy a filter that's then dropped, `get.function.filter_prune`,
`remove_unused_columns.cpp:1468`–`1493`) gets rebuilt from scratch against the
new numbering too. None of this changes *what* gets pruned; it's what makes
the pruning decision safe to apply without corrupting the scan's own filter
evaluation.

*(This checkout's `remove_unused_columns.cpp` also carries a large amount of
machinery this section has deliberately not gone deep on — `PushdownExtractSupport`,
`WritePushdownExtractColumns`, `CheckPushdownExtract`, and the
`ReferencedColumn::struct_extracts`/`child_columns` fields. This is
struct-field-level projection pushdown: pushing a `s.c1` struct-extract
itself down into the scan's `column_ids`, so that even a **sub-field** of a
wide `STRUCT` column that's never fully read doesn't get pulled off disk.
It's the same "reach the scan" principle as above, one level more granular,
and §14.1.5's `deliver_child_columns.test` walkthrough exercises it directly
— but a full mechanical treatment of struct pushdown-extract belongs to
whichever future chapter covers `STRUCT`/`VARIANT` physical layout, not
here.)*

**Pushdown-extract details a user can see.** We do not cover
pushdown-extract in full. But a few of its rules change what a user sees, so
they are listed here.

**A cast that can fail is not pushed into the scan.** When the pass pushes a field read like `g.d` into the scan, it can also
push a cast on top of it (`g.d::DATE`). The scan then hands back the field
already cast. The catch: the scan casts **every row**, including rows a
`WHERE`, a join or a `LIMIT` above would have thrown away. So a cast that
fails on one of those rows would make the whole query fail (DuckDB issue
#28617). `TryGetCastChild` guards against this with one extra test:

```cpp
// src/optimizer/remove_unused_columns.cpp:1834
static bool TryGetCastChild(unique_ptr<Expression> &expr, optional_ptr<unique_ptr<Expression>> &child) {
	if (expr->GetExpressionType() != ExpressionType::OPERATOR_CAST) {
		return false;
	}
	D_ASSERT(BoundCastExpression::IsCast(*expr));
	auto &cast = expr->Cast<BoundFunctionExpression>();
	//! A pushed-down cast runs on every row of the child, including rows that are later filtered out
	if (BoundCastExpression::IsTryCast(cast) || cast.CanThrow()) {
		return false;
	}
	child = BoundCastExpression::ChildMutable(cast);
	return true;
}
```

A cast that can throw stays where the query wrote it. The field read
itself is still pushed down; only the cast stays up. The same `CanThrow()`
test also keeps a throwing cast inside `TRY(...)` from being pushed out of
the `TRY` (DuckDB issue #25236). The test
`test/sql/optimizer/struct_extract_cast_filter.test` shows both sides:

```sql
-- test/sql/optimizer/struct_extract_cast_filter.test:6 (statement/query headers elided)
CREATE TABLE dates(s VARCHAR);
INSERT INTO dates VALUES ('2023-01-01'), ('');
CREATE TABLE structs AS SELECT {'d': s} AS g FROM dates;

SELECT g.d::DATE FROM structs WHERE length(g.d) > 0
----
2023-01-01

statement error
SELECT g.d::DATE FROM structs
----
Conversion Error

-- a cast that cannot throw is still pushed into the scan
CREATE TABLE ints AS SELECT {'i': i::INTEGER} AS g FROM range(3) t(i);
EXPLAIN SELECT g.i::BIGINT FROM ints
----
physical_plan	<!REGEX>:.*CAST.*
```

The empty string `''` is not a valid `DATE`. The `WHERE length(g.d) > 0`
removes that row. If the scan cast `''` before the filter ran, the query
would fail. Instead it returns `2023-01-01`. Without the `WHERE`, the
query still fails, as it should. `INTEGER` to `BIGINT` can never fail, so
that cast still goes into the scan, and no `CAST` shows in the plan. The
same file also checks the cases from the issue: a CTE, a subquery in the
`WHERE`, an `IN (SELECT ...)` and a `LIMIT`.

**A `VARIANT` field inside a `STRUCT` can be read through a projection.**
When the pass pulls a field read up through a projection, it has to build
the read again as an expression. `ConstructStructExtractFromPath` does this
one level at a time. A `STRUCT` level is addressed by position, but a
`VARIANT` level is addressed by a key name. So a `VARIANT` level becomes a
`variant_extract` call by key, which makes a query like
`SELECT x.v.k FROM (SELECT s AS x FROM t)` work:

```cpp
// src/optimizer/remove_unused_columns.cpp:1125 (elided)
if (type_iter.get().id() == LogicalTypeId::VARIANT) {
	// a VARIANT field is addressed by key, and is itself a VARIANT
	arguments[1] = make_uniq<BoundConstantExpression>(Value(path_iter.get().GetFieldName()));
	target = BindBuiltinScalarFunction(context, VariantExtractFun::Name, std::move(arguments));
} else {
	auto child_index = path_iter.get().GetPrimaryIndex();
	// ...
	// the constant argument selects the key or the index overload of struct_extract
	target = BindBuiltinScalarFunction(context, StructExtractFun::Name, std::move(arguments));
}
```

The test `test/sql/types/struct/struct_variant_projection_pushdown.test`
reads `x.v.k` and `x.v.n.x` through `(SELECT id, s AS x FROM t)`, where
`s.v` is a `VARIANT`. A related cast rule applies too.
`ExtractCastCanBePushedDown` (`remove_unused_columns.cpp:1677`) keeps a cast
to `JSON` on top when the field is a `VARIANT`. The reason, in the source
comment: when the `VARIANT` is stored shredded (split into typed columns), a
pushed-down cast would run on the shredded value. A `VARIANT` turned into
JSON and a shredded `VARCHAR` parsed as JSON are not the same thing.

**Two reads of one struct field with different types are both kept.**
`ReferencedColumn::AddPath` keeps a set of field paths per column. When a
shorter (parent) path is added, it may delete a longer (child) path under it,
since reading the parent gives you the child for free. That is wrong when
the parent path carries a cast. In
`SELECT s.payload.x AS a, s.payload::VARCHAR AS b FROM t`, the path
`s.payload` is read as a `VARCHAR`, so it does not hold an `x` to read. So
`AddPath` deletes a child path only if the new parent path really matches
it, types included (`remove_unused_columns.cpp:1783`–`1798`). The test is
`test/sql/optimizer/remove_unused_columns_nested_struct_cast.test`.

### 14.1.4 Joins, `DISTINCT`, `UNION`, and CTEs: worth a paragraph each

A handful of the non-default cases in `RemoveUnusedColumns::VisitOperator`
are specific enough to be worth calling out on their own, because each one
answers a "wait, but what about—" question a reader would otherwise be left
with.

**Equi-joins fold RHS references onto the LHS before pruning anything**
(`LOGICAL_COMPARISON_JOIN`/`LOGICAL_DELIM_JOIN`/`LOGICAL_ASOF_JOIN`,
`remove_unused_columns.cpp:586`). For an `INNER` join with an equality
condition `X = Y` where both sides are plain column references (and neither
is floating-point, since `+0.0`/`-0.0` are equal-but-distinct), any reference
to `Y` anywhere above the join is rewritten to reference `X` instead, and
`Y`'s own entry is erased from `column_references` outright:

```cpp
// src/optimizer/remove_unused_columns.cpp:617 (elided)
// comparison join between two bound column refs
// we can replace any reference to the RHS (build-side) with a reference to the LHS (probe-side)
for (auto &entry : colrefs->second.bindings) {
	auto &colref = entry.get();
	colref.BindingMutable() = lhs_col.Binding();
	AddBinding(colref);
}
column_references.erase(rhs_col.Binding());
```

Since `o.orderkey = l.orderkey` guarantees the two values are equal on every
row that survives the join, there is never a reason to carry *both* copies
through — one is redundant by construction, and folding every downstream
reference onto the LHS column before running the rest of this pass means the
RHS copy is never even considered "referenced," so it's free to be pruned
(from the base scan, if nothing else needs it) exactly like any other unused
column.

**Plain `DISTINCT` needs every column; `DISTINCT ON` needs only its own
target list.** `LOGICAL_DISTINCT` (`remove_unused_columns.cpp:814`) checks
`distinct.distinct_type`: for `DISTINCT ON`, it breaks out of the switch
without forcing `everything_referenced`, so the normal pruning logic applies
based on whatever the `distinct_targets`/`order_by` and the rest of the
query actually reference; for plain `DISTINCT`, `everything_referenced` is
forced to `true` for this operator's child (restored afterward), because
row-uniqueness is a whole-row comparison — dropping any column would change
which rows collapse together. §14.1.6 shows exactly this distinction driving
`distinct_on_projection_pushdown.test`.

**`UNION ALL` can drop columns; `UNION`/`INTERSECT`/`EXCEPT` (with implicit
dedup) cannot**, for the identical whole-row-comparison reason
(`LOGICAL_UNION`, `remove_unused_columns.cpp:637`; `LOGICAL_EXCEPT`/`LOGICAL_INTERSECT`,
`:734`, unconditionally preserve everything). A `UNION ALL` is pure
concatenation with no deduplication, so an unreferenced column is exactly as
prunable as it would be anywhere else — the pass even pushes a fresh
narrowing `LogicalProjection` under each branch when the branches' surviving
column sets need reconciling to a common shape (`:702`–`719`).

**Materialized CTEs need every reference site accounted for before the
definition can be pruned** (`LOGICAL_MATERIALIZED_CTE`, `remove_unused_columns.cpp:845`).
`GatherCTEScans` first finds every `LogicalCTERef` anywhere in the CTE's
continuation (its "right-hand side") and records them as
`expected_readers`; each reference site, when actually visited, marks itself
into `seen_readers` and contributes its own referenced-column set. Only once
`expected_readers == seen_readers` — every reader has actually been visited —
is it safe to compute the union of what every reader needs and prune the
CTE's definition to exactly that column set; visiting only *some* readers and
pruning based on partial information could silently drop a column a
not-yet-visited reader still needs, so the code explicitly throws an
`InternalException` if it ever finds itself in that state
(`remove_unused_columns.cpp:883`–`886`) rather than risk it.

### 14.1.5 Test walkthrough: `test/optimizer/unused_columns/`

**`cte_unused_columns.test`** — *"Test that unused columns are removed from
materialized CTEs."* Four scenarios, in order of increasing structural
complexity:

```sql
-- test/optimizer/unused_columns/cte_unused_columns.test:19
EXPLAIN
WITH test AS MATERIALIZED (SELECT * FROM o_order)
SELECT col0, col6 FROM test;
----
logical_opt	<!REGEX>:.*col[1-5].*
```

`o_order` has seven columns (`col0`..`col6`); only `col0` and `col6` are
selected from the single reference site. The `<!REGEX>` assertion — a
*negative* match — checks that the optimized logical plan's textual `EXPLAIN`
output contains none of `col1` through `col5` anywhere at all, confirming
the CTE's materialization itself was narrowed from seven columns to two, not
just that the final `SELECT` list was narrow (which would be true even if
the CTE still materialized all seven and a `LogicalProjection` above it threw
five away — the assertion is specifically checking that the *CTE definition
itself* only shows `col0`/`col6`). The second scenario adds a second
reference site with a *different* narrow column need
(`(SELECT col1 FROM test) t2`), and checks that the CTE prunes down to the
*union* of both readers' needs (`col0`, `col1`, `col6` — hence only `col2`
through `col5` are asserted absent, one fewer excluded column than the first
case) — this is `GatherCTEScans`/`seen_readers` in action: pruning waits for
both reference sites, then computes their combined requirement. The third
and fourth scenarios (`cte_fanout`, referenced twice via `UNION ALL`) confirm
the same union-of-readers behavior holds when the CTE is scanned by multiple
branches of a set operation rather than multiple subqueries, and a final
`query I` (no `EXPLAIN`) checks the *actual result* (`9`, i.e.
`(1+2)+(1+2)+(1+2)`) to confirm the pruning didn't silently change what the
query computes — the aggregate check every `EXPLAIN`-shape test in this
suite implicitly needs somewhere, so at least one variant in the file
supplies it.

**`deliver_child_columns.test`** — *"deliver child columns in RemoveUnusedColumns
optimizer."* This test's name is the give-away for what it's actually
checking: not whole-column pruning, but the struct pushdown-extract mechanism
flagged in §14.1.3 — a `view` sitting between a `LogicalGet` and its
consumer, and whether that view still "delivers" (forwards) only the
specific struct *fields* its own consumer needs, rather than the whole
struct. The test loops (`foreach source t read_parquet(...)`) over a real
table and a Parquet file read of the same data, and walks through several
view shapes:

```sql
-- test/optimizer/unused_columns/deliver_child_columns.test:26
create or replace view test as select * from {source};
-- ...
explain select s.c1 from test;
----
physical_plan	<REGEX>:.*Projections:.*s.c1.*
```

Here the view itself does no narrowing (`SELECT *`), but the *consumer*
(`s.c1 from test`) only needs one field of the `STRUCT` column `s` — and the
physical plan's scan `Projections:` list shows `s.c1` specifically, not the
whole struct, confirming the field-level narrowing survived through an
unprojected view. A later variant makes the view *itself* reference all of
`s.c1`, `s.c2`, and the whole struct `s`:

```sql
-- test/optimizer/unused_columns/deliver_child_columns.test:46
create or replace view test as select s.c1, s.c2, s from {source};
-- consumer selects only s.c2, s.c1 (not the bare 's')
explain select s.c2, s.c1 from test;
----
physical_plan	<REGEX>:.*Projections:.*s.c1.*
```

The comment directly above this block in the test file states the point
plainly: *"Because the child projection only references certain fields,
projection pushdown gets applied regardless"* — i.e. even though the view's
own definition looks like it needs the whole struct (it projects bare `s`),
because the *outer* query never actually selects bare `s`, `RemoveUnusedColumns`'s
`CheckPushdownExtract`/`WritePushdownExtractColumns` machinery can still
narrow all the way to individual fields once it reruns having seen what's
truly needed above. The final variant in the loop nests a struct-extract
inside `upper(s.c1)`, confirming the pushdown recognizes a field access
buried inside another function call, not just a bare `s.c1` at the top of
a `SELECT` list. A second `foreach` block (from `test_structs2.parquet`
onward) checks the same mechanism against `struct_pack`-constructed structs
built *inside* the query rather than read directly off a table — confirming
the pushdown-extract logic operates on the logical shape of the expression
tree, not merely on a `LogicalGet`'s raw output columns.

**`distinct_on_projection_pushdown.test`** — *"Test `DISTINCT ON` `ORDER BY`."*
This is the test that directly exercises the `DISTINCT`-vs-`DISTINCT ON`
distinction from §14.1.4:

```sql
-- test/optimizer/unused_columns/distinct_on_projection_pushdown.test:8
query I nosort res_1
SELECT col0 FROM (
    SELECT DISTINCT ON (floor(col0)) * FROM t0 ORDER by col0 DESC
);
----
1000 values hashing to 796a92aea33b9601284e2a4ac9e6587e

query I nosort res_1
SELECT col0 FROM (
    SELECT DISTINCT ON (floor(col0)) col0 FROM t0 ORDER by col0 DESC
);
----
```

This pair is worth pausing on for the sqllogictest mechanics themselves, not
just the SQL: **`query I nosort res_1` appearing twice with the same label
(`res_1`), where only the first occurrence spells out an expected value
(here, a hash of the 1000-row result) and the second leaves the expected
section empty, is the test-harness's built-in equivalence check** — the
second block's actual output is compared against the *first* block's
recorded result under that label, not against a value written in the file.
So this pair is a compact, direct proof that `DISTINCT ON (floor(col0)) *`
(selecting every column of `t0`, five of which are never used anywhere) and
`DISTINCT ON (floor(col0)) col0` (selecting only `col0` explicitly) produce
byte-identical output — i.e. that `UNUSED_COLUMNS`/`COLUMN_LIFETIME` pruning
`col1` through `col5` out of the `SELECT *` version doesn't change a single
row of the result, because `DISTINCT ON`'s row-selection is governed
entirely by its `(floor(col0))` target list and `ORDER BY col0 DESC`, never
by which *other* columns happen to be present. The `EXPLAIN` block
immediately after confirms *why* it's safe: the optimized logical plan
excludes `col1`–`col5` entirely (`<!REGEX>:.*col[1-5].*`), meaning this
isn't merely "the extra columns happened not to affect row selection" — they
were never carried past the scan at all. This is the concrete case the
`LOGICAL_DISTINCT` code in §14.1.4 exists for: had `DISTINCT ON` been treated
the same as plain `DISTINCT` (forcing `everything_referenced = true`), this
`<!REGEX>` assertion would fail, because all five otherwise-unused columns
would still be scanned and carried through, just to be thrown away by an
equality comparison `DISTINCT ON` was never actually going to perform on
them.

**`unused_window_columns.test`** (`test/optimizer/unused_window_columns.test`,
group `[optimizer]`, not `[unused_columns]`, but directly exercising the
`LOGICAL_WINDOW` case at `remove_unused_columns.cpp:500`) is worth a mention
here for one specific reason: window functions are the one place
`RemoveUnusedColumns` has to reason about **volatility**, not just
reachability. The source comment at the top of that case states the rule
(`remove_unused_columns.cpp:502`) — *"Preserve volatile evaluation within the
window expression, even when its output is unused. Column references do not
propagate volatility from their producers."* The test file checks this one
rule many times over. It uses `nextval()` on a real `CREATE SEQUENCE` as a
side effect you can observe:

```sql
-- test/optimizer/unused_window_columns.test:200
CREATE SEQUENCE unused_window_sequence;

query I
SELECT count(*) FROM (
    SELECT sum(nextval('unused_window_sequence')) OVER () AS unused_s FROM range(3)
) t;
----
3

query I
SELECT currval('unused_window_sequence');
----
3
```

The window output `unused_s` is never read by anything (only `count(*)` is
selected), so an over-eager pass might drop the whole window computation as
dead — but `nextval()` has an externally-observable side effect (the
sequence's counter actually advances), and DuckDB's contract is that a
volatile expression's number of evaluations must not change just because its
*result* went unused. The test confirms the sequence still advanced by
exactly 3 (once per row) even though the window's numeric output was
discarded — i.e. the window *operator* survives (it must, to preserve
volatile evaluation count), but its *output column* is still pruned from
everything above it. Later cases in the same file extend this to volatility
hiding inside the frame bounds, `FILTER`, the aggregate's own `ORDER BY`
argument, and even nested inside a lambda passed to `list_transform` — each
one checked the same way, by asserting a sequence's `currval()` afterward
rather than by inspecting `EXPLAIN` text, since the whole point is behavior
invisible to the plan shape.

> **Note:** a window's `ORDER BY` and `PARTITION BY` cannot be volatile at
> all. The binder rejects them (DuckDB issue #24830). So the cases in this
> file with `nextval()` inside `ORDER BY` and `PARTITION BY` expect an error:
>
> ```sql
> -- test/optimizer/unused_window_columns.test:236
> # Volatile ordering is rejected before the unused window could be pruned.
> statement ok
> CREATE SEQUENCE window_order_sequence;
>
> statement error
> SELECT count(*) FROM (SELECT row_number() OVER (ORDER BY nextval('window_order_sequence')) AS unused_w FROM range(3) t(i)) t;
> ----
> ORDER BY window expressions cannot be volatile
> ```
>
> The `PARTITION BY` case fails the same way with *"PARTITION BY window
> expressions cannot be volatile"*. The error comes before the pruning code
> runs.

**`materialized_cte_sibling_column_pruning.test`** (group `[optimizer]`) is a
regression test for a specific interaction: a materialized CTE forcing
`everything_referenced = true` on its own *input* (because pruning it safely
requires having seen every reader, per §14.1.4) must not leak that
conservatism sideways into an *unrelated* join sibling. Its query nests a
`LATERAL` join against a one-row materialized CTE (`t`) alongside a second,
independent join input (`b`) that carries a genuinely unused
`struct_pack(id := id) AS unused_payload` column:

```sql
-- test/optimizer/materialized_cte_sibling_column_pruning.test:26
EXPLAIN SELECT b.value
FROM (SELECT id AS k, 0 AS domain FROM v) a
JOIN LATERAL (
    WITH t AS MATERIALIZED (SELECT a.k AS n)
    SELECT n FROM t
) q ON true
JOIN (
    SELECT b.k, b.domain, b.value
    FROM (
        SELECT id AS k, 0 AS domain,
               struct_pack(id := id) AS unused_payload,
               id AS value
        FROM v
    ) b
) b ON b.k = q.n AND b.domain IS NOT DISTINCT FROM a.domain
WHERE a.k = 1;
----
logical_opt	<!REGEX>:.*unused_payload.*
```

The title states the property directly: *"CTE input preservation must not
prevent pruning unrelated join inputs."* Because `everything_referenced`
in `RemoveUnusedColumns` is a per-branch flag threaded through a *tree* of
`RemoveUnusedColumns` instances (each operator that needs its own scope
spawns `RemoveUnusedColumns remove(*this, some_bool)` — see the constructor
overload at `remove_unused_columns.cpp:65` that copies the parent's shared
state but takes its own `is_root`), forcing conservatism for the
`b`-side's own `unused_payload` — even for the sibling `q`/`t` branch's CTE.
The test runs the same query twice, under two different `disabled_optimizers`
settings (`''` and `'join_order,build_side_probe_side'`), specifically to
check the property survives regardless of what shape `JOIN_ORDER` gives the
tree — a good example of a test written to be robust to *which* structural
passes ran, by only asserting on the one column-pruning fact it actually
cares about, at both the logical (`logical_opt`) and physical (`physical_only`)
`EXPLAIN` stages, plus two `query T`/`query I` blocks confirming that
*explicitly* selecting `b.unused_payload` (whole struct) and
`b.unused_payload.id` (one field) both still work correctly when the column
genuinely *is* referenced — i.e. this isn't a test that the column can never
be produced, only that it's dropped precisely when, and only when, nothing
needs it.

### 14.1.6 The C++ unit tests: `test/optimizer/remove_unused_columns.cpp`

Unlike every file above, `test/optimizer/remove_unused_columns.cpp` (304
lines) is a Catch2 C++ test file, not a `.test` sqllogictest script — it
calls `RemoveUnusedColumns` directly against hand-built `LogicalOperator`
trees rather than going through the parser and full optimizer pipeline. Its
harness (`ApplyColumnPruning`, `remove_unused_columns.cpp:37`) confirms
directly, in code, exactly the single-call, `APPLY`-mode invocation this
section has been describing:

```cpp
// test/optimizer/remove_unused_columns.cpp:37 (elided)
static void ApplyColumnPruning(Connection &connection, unique_ptr<LogicalOperator> &plan) {
	// ...
	Optimizer optimizer(*binder, *connection.context);
	RemoveUnusedColumns remove_unused(optimizer);
	remove_unused.VisitOperator(plan);
	// ...
}
```

One `RemoveUnusedColumns` object, one `VisitOperator` call — the whole
`ANALYZE`/`APPLY` machinery described in §14.1.1 happens *inside* that single
call, as scoped lookaheads spawned by whichever operators need them, not as
two separate driver invocations.

The seven `TEST_CASE`s exercise `projection_map` bookkeeping specifically —
i.e. they're testing the *`LogicalFilter`/projection-map* side of pruning
(a synthetic three-column `LogicalProjection` feeding a `LogicalFilter` that
already carries a `projection_map`), not `LogicalGet` scans:

- *"Column pruning removes dead projection map entries"* and *"...remaps
  retained projection map entries"* check the base case both directions:
  a filter's `projection_map` entry pointing at a column nothing above needs
  gets dropped from the map outright; one pointing at a column that *does*
  survive gets remapped to that column's new (shifted) index.
- *"...follows replacements before matching projection map bindings"* and
  *"...distinguishes removed projection map entries from shifted bindings"*
  are the sharper cases: they construct filters whose `projection_map`
  entries interact with columns that get *renumbered* (not just
  deleted) by the surrounding pruning, checking that the map-remapping logic
  (`RemapPrunedProjectionMap`, `remove_unused_columns.cpp:322`) correctly
  tells apart "this map entry pointed at a column that's now gone" from
  "this map entry pointed at a column that's still there, just at a new
  index" — exactly the distinction §14.1.2 traced through
  `ClearUnusedExpressions`/`ReplaceBinding`, now checked directly against
  `LogicalFilter::projection_map` rather than inferred from `EXPLAIN` text.
- *"Column pruning retains a minimum layout for removed projection map
  entries"* builds a case where the *outer* projection needs nothing from
  the filter at all (a bare `Value::INTEGER(42)` constant is all it
  selects) and checks the filter still ends up with a well-formed,
  single-entry `projection_map` rather than an empty or invalid one —
  the same "never leave zero columns behind" concern §14.1.3 covered for
  `LogicalGet`'s `GetAnyColumn()` fallback, exercised here for
  `LogicalFilter` instead.
- The final two cases (*"Recursive CTE pruning does not reapply projection
  map replacements"*, *"...distinguishes removed projection map entries
  from shifted bindings"*) go through the full parser/binder pipeline
  (`PlanUnusedColumnsQuery`) with a real `WITH RECURSIVE` query, locate the
  `LogicalFilter` sitting inside the recursive CTE's continuation branch via
  a small tree-search helper (`FindUnusedColumnsOperator`), deliberately
  pre-populate that filter's `projection_map` before pruning runs, and check
  it comes out correctly adjusted afterward — confirming the CTE-specific
  binding-replacement path (`RewriteRecursiveCTEReferences`,
  §14.1.2) and the ordinary `ClearUnusedExpressions` path don't corrupt each
  other's bookkeeping when both act on the same operator in the same pass.

## 14.2 `COLUMN_LIFETIME` — where the drop-off point is, for a column that already survived

### 14.2.1 A flat set, not a per-column referenced-info struct

`ColumnLifetimeAnalyzer` (`src/optimizer/column_lifetime_analyzer.cpp`,
`src/include/duckdb/optimizer/column_lifetime_analyzer.hpp`) is a much
smaller class than `RemoveUnusedColumns`, and the size difference reflects
the narrower question it answers. Where `RemoveUnusedColumns` tracks a rich
`ReferencedColumn` per binding (live expression references, struct-extract
paths, pushdown-extract eligibility), `ColumnLifetimeAnalyzer` tracks nothing
but a flat set:

```cpp
// src/include/duckdb/optimizer/column_lifetime_analyzer.hpp:44
column_binding_set_t column_references;
```

— membership only: "has anything above this point referenced this
`ColumnBinding`, yes or no." It never deletes anything from an operator's own
expression list; the only thing it ever mutates is a `projection_map` field
on one of four operator types.

The traversal is top-down, exactly like `RemoveUnusedColumns`, and
`everything_referenced` plays the identical role: `true` at the root
(`ColumnLifetimeAnalyzer(optimizer, root, /*is_root=*/true)`, constructed at
`optimizer.cpp:420`), and re-scoped to `true` for subtrees below an operator
that (like `RemoveUnusedColumns`'s handling of plain `DISTINCT`,
`UNION`/`EXCEPT`/`INTERSECT`, and materialized/recursive CTEs) can't safely
narrow what its child produces.

### 14.2.2 The mechanical pattern: snapshot before, install after

Four operator types actually install a `projection_map`:
`LOGICAL_COMPARISON_JOIN` and `LOGICAL_DELIM_JOIN` (both are
`LogicalComparisonJoin` objects; there is no `LogicalDelimJoin` class), only
when the join has at least one equality condition and
`PreferRangeJoinsSetting` isn't set (line 86–89),
`LogicalOrder`, and `LogicalFilter`. All four follow one identical shape,
shown here for `LOGICAL_FILTER` (`column_lifetime_analyzer.cpp:175`):

```cpp
// src/optimizer/column_lifetime_analyzer.cpp:175
case LogicalOperatorType::LOGICAL_FILTER: {
	auto &filter = op.Cast<LogicalFilter>();
	if (everything_referenced) {
		break;
	}

	// filter, figure out which columns are not needed after the filter
	column_binding_set_t unused_bindings;
	ExtractUnusedColumnBindings(op.children[0]->GetColumnBindings(), unused_bindings);

	StandardVisitOperator(op);

	// then generate the projection map
	GenerateProjectionMap(op.children[0]->GetColumnBindings(), unused_bindings, filter.projection_map);

	return;
}
```

Read the three steps in order, because the order is the entire mechanism:

1. **`ExtractUnusedColumnBindings`** checks, for each of the filter's
   *child's* output bindings, whether it's already present in
   `column_references` — the set accumulated so far by everything strictly
   **above** this filter in the tree (this call happens *before* the filter's
   own predicate is visited, so the predicate's own column usage is not yet
   in that set). Anything not found is provisionally "unused" — meaning
   "nothing above this filter needs it," which is exactly the question a
   filter's own *output* projection map needs answered; it says nothing
   about whether the filter's *predicate* needs it, because the predicate
   reads the child's raw bindings directly and was never going to be
   affected by this operator's own output projection map regardless.
2. **`StandardVisitOperator(op)`** (`column_lifetime_analyzer.cpp:48`) does
   two things in a fixed order: `VisitOperatorExpressions(op)` — walk the
   filter's *own* predicate expressions, adding every column they reference
   into `column_references` — and *then* `VisitOperatorChildren(op)` —
   recurse into the child (and everything below it), using this now-updated
   `column_references`.
3. **`GenerateProjectionMap`** builds the actual `projection_map` from the
   *frozen* `unused_bindings` snapshot taken in step 1 — deliberately not
   recomputed after step 2, so a column referenced only by the filter's own
   predicate and by nothing above it still counts as prunable from the
   filter's *output*, even though it was just added to `column_references`
   in step 2 for the sake of everything *below*.

This is precisely the "earliest safe drop-off point" mechanism the chapter
set out to verify: a column needed by an early filter and nothing else gets
dropped from that filter's own output projection map, immediately, rather
than continuing to ride through every operator above it. The equality-join
case (`column_lifetime_analyzer.cpp:76`) is the same three-step shape applied
twice, once per side:

```cpp
// src/optimizer/column_lifetime_analyzer.cpp:91
column_binding_set_t lhs_unused;
column_binding_set_t rhs_unused;
ExtractUnusedColumnBindings(op.children[0]->GetColumnBindings(), lhs_unused);
ExtractUnusedColumnBindings(op.children[1]->GetColumnBindings(), rhs_unused);

StandardVisitOperator(op);

if (op.type != LogicalOperatorType::LOGICAL_ASOF_JOIN) {
	GenerateProjectionMap(op.children[0]->GetColumnBindings(), lhs_unused, comp_join.left_projection_map);
}
GenerateProjectionMap(op.children[1]->GetColumnBindings(), rhs_unused, comp_join.right_projection_map);
```

A join key referenced only by the join's own equality condition (Query A's
`o.orderkey = l.orderkey`, once `UNUSED_COLUMNS`'s RHS-onto-LHS folding from
§14.1.4 has already collapsed the two copies to one that both sides still
internally read) is exactly the shape this drops from the join's *output*:
the physical hash-join operator still reads it raw, off both input chunks,
to build and probe the hash table — the projection map only decides which
columns get materialized into the join's output chunk once a match is
found, and a pure join key with no life above the join doesn't need to be
one of them.

### 14.2.3 `Verify`/`AddVerificationProjection` — a debug-only, self-checking harness

One more piece of `ColumnLifetimeAnalyzer` worth citing precisely because it
reuses `ColumnBindingReplacer` for a genuinely different purpose than the
CTE-rewriting use in §14.1.2: `Verify` (`column_lifetime_analyzer.cpp:198`),
gated behind `Settings::Get<DebugVerificationProjectionSetting>`. When
enabled, before visiting a filter/order/join, it wraps that operator's
child in a synthetic `LogicalProjection` that interleaves every real column
with a `NULL` placeholder column, using `ColumnBindingReplacer` to rewrite
every *existing* reference to the old bindings so they now point at the new
projection's (shuffled) output positions:

```cpp
// src/optimizer/column_lifetime_analyzer.cpp:223 (elided)
void ColumnLifetimeAnalyzer::AddVerificationProjection(unique_ptr<LogicalOperator> &child) {
	// If our child has columns [i, j], we will generate a projection like so [NULL, j, NULL, i, NULL]
	// ...
	ColumnBindingReplacer replacer;
	// ... populate replacer.replacement_bindings with old_binding -> new (shuffled) binding ...
	replacer.stop_operator = child.get();
	replacer.VisitOperator(root);
	// ...
}
```

The point of interleaving `NULL` placeholders and shuffling real columns'
positions is to make any *subsequent* bug in the pass's own projection-map
math loudly visible: if a projection map's index bookkeeping is ever off by
one, this deliberately-scrambled layout turns that into a `NULL` appearing
where a real value was expected, or an out-of-bounds read, rather than a
silent wrong-but-plausible-looking value — precisely the same category of
bug `must_visit_operator_expressions_first.test` (§14.2.4) is written to
catch, except this harness manufactures many more chances to trip over such
a bug on every debug build's test run, rather than relying on one
specifically hand-crafted query. `ColumnBindingReplacer`'s appearance here is
the second confirmation of the §14.1.2 correction: it's a generic
old-binding-to-new-binding rewrite tool, reached for here to patch *every*
outstanding reference to a child's bindings after wrapping it in a
projection, exactly as it was reached for to patch every `LogicalCTERef`
after a CTE's definition was renumbered — never the mechanism behind an
operator's *own* pruning-driven reindexing, which (§14.1.2) is handled
inline via live expression references instead.

### 14.2.4 Case study: `must_visit_operator_expressions_first.test`

This test's name states the invariant directly, and reading
`StandardVisitOperator` shows exactly what it's guarding:

```cpp
// src/optimizer/column_lifetime_analyzer.cpp:48
void ColumnLifetimeAnalyzer::StandardVisitOperator(LogicalOperator &op) {
	VisitOperatorExpressions(op);
	VisitOperatorChildren(op);
}
```

**Expressions first, children second — never the other way around.** Every
non-default case in `ColumnLifetimeAnalyzer::VisitOperator` that installs a
projection map ultimately calls `StandardVisitOperator(op)` to do the actual
descent (§14.2.2's step 2), rather than hand-rolling its own
expressions-then-children order — meaning this ordering isn't an
incidental detail of one operator's implementation, it's the one traversal
invariant every operator relies on holding.

Here is why getting it backwards would be a genuine correctness bug, not
merely a missed optimization, worked through concretely against the actual
test query:

```sql
-- test/optimizer/column_lifetime_analyzer/must_visit_operator_expressions_first.test:6
create table t5 as select (range + 1000) a5, range b5,
    (range + 50)::INT::VARCHAR || '__suffix__' c5 from range(50);
create table t1 as select range::INT a1, (range + 45)::INT b1,
    (range)::INT::VARCHAR || '__suffix__' c1 from range(900);

select a5, c1 from t1, t5 where b5=b1;
----
1045	0__suffix__
1046	1__suffix__
1047	2__suffix__
1048	3__suffix__
1049	4__suffix__
```

`t1` and `t5` each have three columns; only `a5` and `c1` are ever selected,
and `b5`/`b1` exist purely to drive the `WHERE b5 = b1` equi-join condition
(itself lowered into a proper `LogicalComparisonJoin` by `FILTER_PUSHDOWN`
and `JOIN_ORDER`, long before `COLUMN_LIFETIME` runs). By the time
`COLUMN_LIFETIME` sees this tree, `UNUSED_COLUMNS` has already deleted `a1`
and `c5` completely (nothing anywhere references them) — but `b1` and `b5`
*are* referenced, by the join condition itself, so they survive
`UNUSED_COLUMNS` fully intact all the way up from each `LogicalGet`. It's
`COLUMN_LIFETIME`'s job to notice the join is the *last* place either of
them is needed and drop them from the join's own output — i.e. this query is
constructed so that its correctness depends entirely on the equi-join case
in §14.2.2 getting the three-step ordering right.

Now trace what would go wrong under the *opposite* ordering — visiting an
operator's children before its own expressions. Consider what happens one
level below the join, at whatever operator actually exposes `b1`/`b5` to it
(here, directly the `LogicalGet`s, since nothing else sits between the scan
and the join once earlier passes have run). The join's own condition
(`b5 = b1`) is the *only* thing in the entire query that references `b1` and
`b5` at all — nothing above the join does, and nothing below the scan does
either. If the traversal visited the join's children (recursing down into
each side, and whatever pruning decisions happen along the way) *before*
registering the join condition's own use of `b1`/`b5` into `column_references`,
then at the moment anything below the join needed to answer "is this binding
referenced by anything already seen," the answer for `b1`/`b5` would
incorrectly be **no** — because the one and only thing that references them,
the join condition sitting directly above, hasn't been visited yet. Any
operator between the join and the scan that installs its own projection map
based on that (wrong) answer would then build a projection map that drops
`b1`/`b5` from what it forwards upward — cutting the join condition off
from a column it still needs to evaluate `b5 = b1` against. That is not a
missed optimization; it is a dangling `ColumnBinding`, the same failure mode
`Verify`/`AddVerificationProjection` (§14.2.3) is built to surface loudly in
debug builds, and in a release build it is either a hard crash resolving the
binding or — worse — a silently wrong join result.

This is exactly why the test asserts on the **query's actual output rows**,
not on `EXPLAIN` plan shape (contrast every other test in this section,
which mostly assert `<REGEX>`/`<!REGEX>` against `logical_opt` or
`physical_plan` text): a traversal-order bug of this kind doesn't
necessarily change the *shape* of the plan an `EXPLAIN` would show at all —
projection maps are invisible in ordinary `EXPLAIN` output, and a
subtly-wrong one can still produce a plan that *looks* identical while
quietly reading the wrong column index or dropping a binding the join
needed. The only way to catch that class of bug is to actually run the
query and check the rows come back right — which is precisely, and only,
what this test does.

### 14.2.5 The rest of `test/optimizer/column_lifetime_analyzer/`

**`no_unnecessary_projections.test`** — *"Test column lifetime."* This test
demonstrates the flip side of §14.2.2's mechanism: that narrowing a join's
output via its own `projection_map` field means the physical plan never
needs a *separate* `PhysicalProjection` operator bolted on afterward just to
throw away a no-longer-needed join key:

```sql
-- test/optimizer/column_lifetime_analyzer/no_unnecessary_projections.test:12
explain analyze SELECT rhs.j FROM rhs SEMI JOIN lhs USING (i);
----
analyzed_plan	<REGEX>:.*RIGHT_SEMI.*

explain analyze SELECT rhs.j FROM rhs SEMI JOIN lhs USING (i);
----
analyzed_plan	<!REGEX>:.*HASH JOIN.*PROJECTION.*
```

`rhs` (10 rows) `SEMI JOIN lhs` (100 rows) `USING (i)`, selecting only
`rhs.j`. A `SEMI` join only ever exposes the "existence-tested" side's
columns (here, `rhs`'s) — `i` itself is the join key and, once the query
only selects `j`, has no life past the join at all. The first assertion
confirms the physical operator is a `RIGHT_SEMI` join (the physical planner,
Chapter 9, flips which side physically builds the hash table based on
relative sizes, independent of which side is the *logical* left/right); the
second — a negative match for `HASH JOIN` immediately followed by
`PROJECTION` anywhere in the analyzed plan text — confirms that dropping `i`
from the join's output didn't require inserting a distinct projection
operator to do it. This is the concrete payoff of `ColumnLifetimeAnalyzer`'s
own header comment (`column_lifetime_analyzer.hpp:20`: *"ensures that
columns are removed from the plan when no longer required"*) and
`optimizer.cpp`'s inline comment on the pass (*"creates projection maps so
unused columns are projected out early"*) — "projected out" here means
*via the join operator's own built-in projection map field*, not via a
second operator materializing a narrower copy of the join's rows.

**`summary_column_lifetime.test`** — *"Test column lifetime analyzer with
SUMMARY (internal issue #4138)."* This one is a minimal regression smoke
test, not an `EXPLAIN`-shape assertion:

```sql
-- test/optimizer/column_lifetime_analyzer/summary_column_lifetime.test:6
create table data as select * from range(0,4000) tbl(col);
SELECT * FROM summary((SELECT col FROM data ORDER BY col));
```

Being honest about what this test does and doesn't reveal: it has no
`EXPLAIN` block and no expected-row check beyond `statement ok` — its only
assertion is that this shape runs at all, without erroring or crashing. The
query's construction is nonetheless a recognizable stress case for the
`LOGICAL_ORDER_BY` branch in §14.2.2: the `ORDER BY col` sits directly
beneath the `summary()` table function (a table-in/table-out function
consuming the subquery's result), and `col` is simultaneously the sort key
*and* the entire projected output — the degenerate case where
`unused_bindings` (§14.2.2's step 1) should legitimately come back empty
(nothing to drop, since the one column present is needed both as the sort
key and as the only output). The test's name and its cited internal issue
number (`#4138`) are the only concrete evidence tying it to a specific past
failure in this exact interaction; this book's sourcing policy is to say so
plainly rather than reconstruct a specific historical bug mechanism that the
test file itself doesn't state.

## 14.3 Why each pass runs twice

Recall the two positions from §7.4 of the overview chapter: `UNUSED_COLUMNS`
at #23 (`optimizer.cpp:401`) and again at #38 (`:517`); `COLUMN_LIFETIME` at
#26 (`:419`) and again at #39 (`:525`). Between the first and second
occurrence of each, the pipeline runs (in order): `DUPLICATE_GROUPS`,
`COMMON_SUBEXPRESSIONS`, `COLUMN_LIFETIME` (its *first* run, sitting between
the two `UNUSED_COLUMNS` runs), `BUILD_SIDE_PROBE_SIDE`, `COMMON_SUBPLAN`,
`LIMIT_PUSHDOWN`, `ROW_GROUP_PRUNER`, `SAMPLING_PUSHDOWN`, `TOP_N`,
`LATE_MATERIALIZATION`, `STATISTICS_PROPAGATION`, `PROJECTION_PLACEMENT`,
`TOP_N_WINDOW_ELIMINATION`, and `COMMON_AGGREGATE` — then `UNUSED_COLUMNS`'s
second run, then `COLUMN_LIFETIME`'s second run immediately after that.

The reasons stated in the source are narrower, and more precise, than
"a lot happened in between, so recheck everything." `optimizer.cpp` names
them in a comment right above the conditional rerun:

```cpp
// src/optimizer/optimizer.cpp:513
// statistics propagation removes filters, join conditions and aggregate children that it proves redundant.
// this can leave columns in a scan that nothing references any more - prune them again. so does a top-N rewrite
// with late materialization, which reads the payload in a second scan
if (removed_expressions || topn_used_late_materialization) {
	RunOptimizer(OptimizerType::UNUSED_COLUMNS, [&]() {
		ClearProjectionMaps(*plan);
		RemoveUnusedColumns unused(*this);
		unused.VisitOperator(plan);
	});
}
```

The rerun has two triggers: `removed_expressions` and
`topn_used_late_materialization`. The second is explained below, after the
first.

`removed_expressions` is set from `StatisticsPropagator::HasRemovedExpressions()`
(`optimizer.cpp:474`, `:481`, `:491`) — and it's set *twice*, because
`STATISTICS_PROPAGATION` itself runs its core `PropagateStatistics` pass,
then (if the plan contains anything worth it) a cost-based
`MultiStageAggregateRewriter` pass runs using the statistics just computed,
and if *that* changes anything, statistics get propagated a **second** time
right there, inside the same pipeline position, before this pass's timer
even closes. Either propagation pass can prove a filter always-true (drop
the conjunct), an always-false subtree (collapse to `LogicalEmptyResult`),
or an aggregate input provably redundant — and any of those deletions can
mean a column that was still legitimately "used" the first time
`UNUSED_COLUMNS` ran (#23, well before `STATISTICS_PROPAGATION` even starts)
is now referenced by nothing at all. The rerun exists specifically to catch
*that* category of newly-orphaned column — one whose only reference was an
*expression* that statistics propagation itself deleted — not to redo work
`UNUSED_COLUMNS` already did correctly the first time on expressions that
are still present. `ClearProjectionMaps(*plan)` running immediately before
the rerun (`optimizer.cpp:518`) is there because the projection maps
`COLUMN_LIFETIME` (#26) and `PROJECTION_PLACEMENT` (#35) built in the
meantime may themselves reference expressions that no longer exist after
`STATISTICS_PROPAGATION`'s deletions — clearing them first means
`UNUSED_COLUMNS`'s second run starts from a clean slate rather than trying
to reconcile stale projection-map indices against a tree that's since
changed shape.

**The second trigger: a top-N window rewrite that used late
materialization.** `TOP_N_WINDOW_ELIMINATION` (#36) turns a "top rows per
group" query into a grouped `arg_max`-style aggregate (§7.18 covers it). With
late materialization, the rewrite reads the table twice. The first scan finds
the winning rows and needs only the partition and order columns, plus the
row id. The second scan fetches the other columns, only for the winning row
ids. Take the query from `test/optimizer/topn_window_elimination_late_materialization_scan.test`:

```sql
-- test/optimizer/topn_window_elimination_late_materialization_scan.test:6
CREATE TABLE t AS SELECT i % 50 AS a, i AS b, md5(i::VARCHAR) AS c FROM range(10000) t(i);
-- ...
EXPLAIN SELECT a, b, c FROM t WHERE a IN (1, 2, 3, 4) QUALIFY row_number() OVER (PARTITION BY a ORDER BY b DESC) <= 2
----
physical_plan	<REGEX>:.*Projections: a, b  .*
```

The first scan should read only `a` and `b`, not `c`. But the rewrite runs
*after* the first `UNUSED_COLUMNS`, so without a second pruning run nothing
would prune the new first scan. For a table where `c` is a 768-byte string,
that is the difference between loading about 740 MiB of table data and
5.2 MiB. So `TopNWindowElimination` records whether any rewrite used late
materialization:

```cpp
// src/include/duckdb/optimizer/topn_window_elimination.hpp:44
//! Whether a rewrite used late materialization, which leaves the input of its aggregate to be pruned afterwards
bool UsedLateMaterialization() const {
	return any_late_materialization;
}
```

`optimizer.cpp:504` copies that into `topn_used_late_materialization`, and
the rerun of `UNUSED_COLUMNS` fires on it. The rerun calls
`ClearProjectionMaps` first, so it can prune below the join that the rewrite
built.

`COLUMN_LIFETIME`'s second run (#39) is **not** gated by `removed_expressions`
at all — it runs unconditionally, immediately after `UNUSED_COLUMNS`'s
(conditional) second run, and the inline comment at that call site is
simply "See #26" territory: having just rebuilt (or, if neither trigger
fired, left untouched) the column set via `UNUSED_COLUMNS`, the projection
maps that decide *where* each surviving column's carry-forward stops need to
be rebuilt too. When the rerun ran, `ClearProjectionMaps` just wiped whatever
`COLUMN_LIFETIME` and `PROJECTION_PLACEMENT` had built earlier in the
pipeline. When it did not run, the maps may still be out of date after the
passes in between. This second run is best understood as bringing
`COLUMN_LIFETIME`'s bookkeeping back in sync with the tree, not as
discovering some qualitatively new class of pruning opportunity of its own.

Two passes that ran *between* the two `UNUSED_COLUMNS`/`COLUMN_LIFETIME`
occurrences deserve a flagged note here, because the reasoning below is this
book's own — the source comments cited above name `STATISTICS_PROPAGATION`
and the top-N window rewrite, and no inline comment anywhere in
`optimizer.cpp` attributes the rerun to the `LATE_MATERIALIZATION` pass or
`PROJECTION_PLACEMENT`:

- **`LATE_MATERIALIZATION`** (#33, immediately before `STATISTICS_PROPAGATION`)
  rewrites certain scans to carry row IDs through narrow intermediate
  operators and defer fetching wide payload columns until a late semi-join
  reconstructs them (§7.4's description). It's plausible this reshaping
  could, in principle, change what an operator's "output columns" even
  *are* in a way that exposes a fresh pruning opportunity — but nothing in
  the source or its comments confirms this actually happens in practice for
  this pipeline's ordering, and the code's own stated trigger for the
  rerun is `removed_expressions` specifically, which `LATE_MATERIALIZATION`
  never sets. This book is not asserting `LATE_MATERIALIZATION` is a cause
  of the rerun; it's flagging the hypothesis and stating plainly that the
  source doesn't support it as one. Do not confuse this pass with the
  second trigger above. That trigger is late materialization done *inside*
  `TOP_N_WINDOW_ELIMINATION` (#36), which runs *after*
  `STATISTICS_PROPAGATION`. The `LATE_MATERIALIZATION` pass (#33) does not set
  it.
- **`PROJECTION_PLACEMENT`** (#35) is different: it runs *inside* the same
  gated block as `STATISTICS_PROPAGATION` (guarded by `propagated_statistics`,
  not `removed_expressions`, at `optimizer.cpp:484`), using the freshly
  computed statistics map to decide where a narrowing projection should
  physically sit — but it is itself a consumer of `COLUMN_LIFETIME`'s and
  `UNUSED_COLUMNS`'s prior output (§7.4's #35 description: *"the placement
  decision that the column-lifetime analysis and statistics propagation set
  up but don't themselves execute"*), not a producer of a new column-pruning
  fact for those earlier passes to react to. It runs inside the same
  `if (propagated_statistics)` block (`optimizer.cpp:484`, guarding both the
  cost-based aggregate rewrite and, at `:493`, `PROJECTION_PLACEMENT`
  itself) rather than being gated by `removed_expressions` the way the
  `UNUSED_COLUMNS` rerun is. Reasoning from the code's actual data flow,
  `PROJECTION_PLACEMENT`'s role in the rerun is therefore best understood as
  one more reason `COLUMN_LIFETIME`'s projection maps need rebuilding at #39
  (since `PROJECTION_PLACEMENT` moves things around after #26's maps were
  built) — a consequence of the unconditional `COLUMN_LIFETIME` rerun
  already covered above, not an independent trigger of its own.

The source documents two causal chains, end to end. The first:
`STATISTICS_PROPAGATION` (possibly twice, via the cost-based aggregate
rewrite loop) proves some expression redundant and deletes it → a column
that expression was the sole reader of becomes newly, provably unused →
`UNUSED_COLUMNS`'s second run (gated on exactly this condition) removes it
→ `COLUMN_LIFETIME`'s second run (unconditional) rebuilds the projection
maps against the now-current tree. The second: `TOP_N_WINDOW_ELIMINATION`
builds a new scan with late materialization that reads more columns than it
needs → the same `UNUSED_COLUMNS` rerun prunes that scan → `COLUMN_LIFETIME`
rebuilds the maps. Everything else that ran in between —
`DUPLICATE_GROUPS`, `COMMON_SUBEXPRESSIONS`, `BUILD_SIDE_PROBE_SIDE`,
`COMMON_SUBPLAN`, `LIMIT_PUSHDOWN`, `ROW_GROUP_PRUNER`, `SAMPLING_PUSHDOWN`,
`TOP_N`, a top-N window rewrite *without* late materialization,
`COMMON_AGGREGATE` — is either a
consumer of the narrowed column set these two passes already produced once
(§7.7's Query A trace shows `BUILD_SIDE_PROBE_SIDE` consuming exactly this),
or structurally incapable of orphaning a column the way an expression
*deletion* can, since none of them delete an expression outright the way
`STATISTICS_PROPAGATION`'s empty-result/redundant-filter proofs do.
