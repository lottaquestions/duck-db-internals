# Chapter 7 Deep Dive — FILTER_PULLUP and FILTER_PUSHDOWN

§7.6.3 of the chapter overview (`00-overview.md`) gave this pair of passes a
few paragraphs: pushdown walks the tree carrying filters downward as far as
each operator's semantics allow, pullup runs one step earlier and derives new
filters on the *other* side of an equi-join so pushdown has something to push
there too, and the payoff — a predicate landing on a `LogicalGet` where it
becomes a table filter the storage layer can use for zonemap-based row-group
skipping — was illustrated with Query A's `l.shipdate < DATE '1998-01-01'`.
That was necessarily the shallow version. This file is the deep one: every
operator-specific handler under `src/optimizer/pushdown/` read in full, the
`FilterCombiner`'s equivalence-class machinery that makes the "transitive
filter" trick possible, the exact coalesce-rewriting code that lets a filter
cross a `FULL OUTER JOIN` boundary without corrupting NULL-extended rows, and
roughly twenty real `.test` files walked through with their actual SQL,
actual `EXPLAIN` regexes, and — where the test's own name promises a negative
result — an explanation of exactly which correctness boundary the test is
guarding.

Two administrative notes before the source dive:

**On the "same-label idiom."** Several of the tests below use a pattern where
two `query II nosort <label>` blocks share the same label and no expected
output is written under either `----`. This is `sqllogictest`'s way of
asserting that two *different* SQL statements produce byte-identical
`EXPLAIN` output — the harness runs both, and the test fails if their outputs
diverge, without anyone having to hand-write what that output should look
like. It shows up constantly in this file's evidence because it is the
natural way to prove *"the optimizer's automatic pushdown produced exactly
the plan you'd get by writing the pushdown yourself, by hand, in the SQL
text."* A sibling deep-dive file may cover the idiom in more depth elsewhere
in this book's ~27-part optimizer treatment; here it is enough to recognize
the shape (`nosort <label>` appearing twice, both blocks empty) and read it
as an equality assertion between two plans.

**On source layout.** `FILTER_PUSHDOWN` (`OptimizerType::FILTER_PUSHDOWN`,
pass #6) is driven by `src/optimizer/filter_pushdown.cpp` plus
`src/optimizer/filter_combiner.cpp`, and dispatches to **thirteen**
per-operator-type handler files under `src/optimizer/pushdown/`:
`pushdown_aggregate.cpp`, `pushdown_cross_product.cpp`,
`pushdown_distinct.cpp`, `pushdown_filter.cpp`, `pushdown_get.cpp`,
`pushdown_inner_join.cpp`, `pushdown_left_join.cpp`, `pushdown_limit.cpp`,
`pushdown_mark_join.cpp`, `pushdown_outer_join.cpp`,
`pushdown_projection.cpp`, `pushdown_secure_view.cpp`,
`pushdown_semi_anti_join.cpp`, `pushdown_set_operation.cpp`,
`pushdown_single_join.cpp`, `pushdown_unnest.cpp`, `pushdown_window.cpp` —
that's actually seventeen distinct files once every join type is counted
separately (inner, left, outer, mark, single, semi/anti each get their own
file, `pushdown_get`/`pushdown_filter`/`pushdown_projection`/etc. round out
the rest); every one of them is read and explained below.
`FILTER_PULLUP` (pass #5, one step *earlier* in the fixed pipeline) is
`src/optimizer/filter_pullup.cpp` plus five small per-operator files under
`src/optimizer/pullup/`.

---

## Part I — `FILTER_PUSHDOWN`: the general algorithm

### 1.1 The `Rewrite` dispatch and the `Filter` bag

`FilterPushdown::Rewrite` (`src/optimizer/filter_pushdown.cpp:108`) is the
single entry point, and every one of the operator-specific methods below is
reached from its `switch`:

```cpp
// src/optimizer/filter_pushdown.cpp:118
switch (op->type) {
case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
	return PushdownAggregate(std::move(op));
case LogicalOperatorType::LOGICAL_FILTER:
	return PushdownFilter(std::move(op));
case LogicalOperatorType::LOGICAL_CROSS_PRODUCT:
	return PushdownCrossProduct(std::move(op));
case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
case LogicalOperatorType::LOGICAL_ANY_JOIN:
case LogicalOperatorType::LOGICAL_ASOF_JOIN:
case LogicalOperatorType::LOGICAL_DELIM_JOIN:
	return PushdownJoin(std::move(op));
case LogicalOperatorType::LOGICAL_PROJECTION:
	return PushdownProjection(std::move(op));
case LogicalOperatorType::LOGICAL_INTERSECT:
case LogicalOperatorType::LOGICAL_EXCEPT:
case LogicalOperatorType::LOGICAL_UNION:
	return PushdownSetOperation(std::move(op));
case LogicalOperatorType::LOGICAL_DISTINCT:
	return PushdownDistinct(std::move(op));
case LogicalOperatorType::LOGICAL_ORDER_BY:
	// we can just push directly through these operations without any rewriting
	RewriteChild(*op, 0);
	return op;
// ... LOGICAL_MATERIALIZED_CTE, LOGICAL_GET, LOGICAL_LIMIT, LOGICAL_WINDOW,
//     LOGICAL_UNNEST, LOGICAL_SECURE_VIEW handled similarly
default:
	return FinishPushdown(std::move(op));
}
```

An instance of `FilterPushdown` carries a `vector<unique_ptr<Filter>> filters`
— predicates collected on the way down but not yet applied — and a
`FilterCombiner combiner` that normalizes and deduplicates them. Every
`Filter` (`FilterPushdown::Filter`, `filter_pushdown.hpp:39`) records which
`TableIndex` bindings it touches (`bindings`, populated by `ExtractBindings`
via `LogicalJoin::GetExpressionBindings`) and whether it carries a **barrier**
(`has_barrier`, from `ExpressionBarrier::Contains` — see §1.3 below). The
`default` case, `FinishPushdown` (`filter_pushdown.cpp:317`), is the fallback
for any operator type this file doesn't special-case (window functions
reached the `LOGICAL_WINDOW` branch specifically, but e.g. `SAMPLE`,
`PIVOT`-derived operators, and anything else not explicitly listed lands
here): it recurses into every child with a fresh `FilterPushdown` instance
(so filters never leak sideways between siblings) and then calls
`PushFinalFilters`, which wraps whatever couldn't be pushed further in a
`LogicalFilter` sitting directly on top.

Many handlers recurse into a child with `RewriteChild(*op, child_idx)`
(`filter_pushdown.cpp:327`) rather than with a bare
`op->children[i] = Rewrite(...)`. Some operators (joins, mostly) carry a
*projection map*: a list of child column positions they keep. Pushdown can
change a child's output columns, for example when a projection is split.
`RewriteChild` saves the child's column bindings before the rewrite and then
remaps the parent's projection map onto the child's new layout
(`LogicalOperatorVisitor::RemapProjectionMap`).

### 1.2 `AddFilter`, the combiner, and `GenerateFilters`

`AddFilter` (`filter_pushdown.cpp:217`) is where an incoming predicate first
meets the machinery that will end up doing the clever transitive-derivation
work covered in Part III. It calls `combiner.AddConjuncts`
(`filter_combiner.cpp:118`): every filter is split by top-level `AND` first
(`LogicalFilter::SplitPredicates`), then each conjunct is fed to
`combiner.AddFilter`. The combiner doesn't just store expressions in a list —
it builds **equivalence classes** of expressions known to be equal to each
other (from `x = y` conditions) and, per class, a bucket of known constant
comparisons (from `x > 10`, `x <= 20`, etc.). `GenerateFilters`
(`filter_pushdown.cpp:224`) is the inverse operation: it asks the combiner to
emit its accumulated knowledge back out as a fresh list of `Filter` objects —
and because the combiner may have *derived* new comparisons that were never
written in the original SQL (this is Part III's subject), the filter list
that comes back out of `GenerateFilters` can be strictly larger, and strictly
more useful to downstream pushdown, than the list that went in.

### 1.3 The barrier mechanism: what may never move where it changes observable behavior

`ExpressionBarrier` (`src/include/duckdb/planner/expression/expression_barrier.hpp`)
marks an expression node as "may throw, or has an externally observable
side effect" — the class comment states the invariant precisely: *"a barred
expression is never moved to a position where it would be evaluated on rows
that the operators around it filter out, and it is always evaluated after the
filters that sit next to it."* `FilterPushdown::BarrierCanPassThrough`
(`filter_pushdown.cpp:82`) enumerates the only operator types a barred filter
is allowed to pass through unchanged — `LOGICAL_FILTER`, `LOGICAL_PROJECTION`,
`LOGICAL_ORDER_BY`, `LOGICAL_GET`, `LOGICAL_SECURE_VIEW` — because none of
those can remove rows on their own. The very top of `Rewrite`
(`filter_pushdown.cpp:110`) checks this before doing anything else: if the
current operator *can* remove rows (a join, an aggregate with `HAVING`-style
filtering, a `LIMIT`, …) and there are barred filters still pending, they get
peeled off with `ExtractBarrierFilters` and re-attached as an ordinary
`LogicalFilter` sitting *above* that row-removing operator, guaranteeing they
never see a row the operator below would otherwise have discarded first.
`AddLogicalFilter` (`filter_pushdown.cpp:237`) additionally
`stable_partition`s a filter's conjuncts so barred ones always evaluate last
within the same `LogicalFilter` node, after every non-barred sibling conjunct
— consistent with `REORDER_FILTER`'s own guard (§7.6.1 of the overview),
which refuses to reorder past a `CanThrow()` or barrier-carrying conjunct for
the identical reason.

### 1.4 `FinishPushdown` and `PushFinalFilters`

Two small helpers do most of the "give up gracefully" work.
`PushFinalFilters` (`filter_pushdown.cpp:263`) takes whatever remains in
`filters` and wraps the operator in a fresh `LogicalFilter`; `FinishPushdown`
(`filter_pushdown.cpp:317`) is that plus "first recurse into every child with
a brand-new `FilterPushdown` instance." Both patterns recur throughout every
operator-specific handler covered below — "push what you can, then finish
with `FinishPushdown`/`PushFinalFilters` for the rest" is the uniform shape
every handler follows.

---

## Part II — every per-operator pushdown handler

### 2.1 `LOGICAL_FILTER` — `pushdown_filter.cpp`

The simplest handler in the file: gather the filter's own expressions into
`AddFilter`, discard the `LogicalFilter` node itself, and recurse into its
child — *unless* the filter carries a `HasProjectionMap()` (some filters are
also given a projection map so they can narrow columns as a side effect; in
that case `FinishPushdown` is used instead, since merging the projection map
semantics with pushdown's own rewriting isn't handled here). This is why a
chain of `WHERE` clauses effectively "disappears" from the optimized plan
whenever pushdown succeeds all the way to the scan — you can see this
directly in `table_filter_pushdown.test`:

```sql
-- test/optimizer/pushdown/table_filter_pushdown.test
EXPLAIN SELECT k FROM integers where j=5
-- logical_opt   <!REGEX>:.*FILTER.*        -- no LogicalFilter survives at all
```

versus a case where part of the predicate can't be pushed all the way into
a table filter:

```sql
EXPLAIN SELECT k FROM integers where i+j > 10 and j = 5 and i = k+1
-- logical_opt   <REGEX>:.*FILTER.*         -- a LogicalFilter remains
```

`i+j > 10` and `i = k+1` are multi-column expressions the scan's table-filter
representation can't natively encode (table filters are single-column;
see §4 below), so those two conjuncts survive as a `LogicalFilter` directly
above the scan, while `j = 5` — single-column, a plain comparison against a
foldable constant — gets absorbed entirely into the scan's own `table_filters`
and vanishes from the printed plan.

### 2.2 `LOGICAL_GET` — `pushdown_get.cpp` — the payoff operator

This is the handler responsible for the entire reason this pass exists: it's
where an accumulated filter either becomes part of the scan's own
`table_filters` (enabling `RowGroup::CheckZonemap`,
`src/storage/table/row_group.cpp:732` — see Chapter 17 §17.5 for the
zonemap mechanism itself) or, failing that, becomes a `LogicalFilter` sitting
directly on top of the `LogicalGet` (still cheap, but no longer eligible for
row-group skipping).

**Column-ref alias normalization first.** Before anything else,
`NormalizeColumnRefAliases` (`pushdown_get.cpp:22`) walks every pending filter
and, for any `BoundColumnRefExpression` bound to this `LogicalGet`'s
`table_index`, rewrites its alias back to the scan's own column name. The
comment explains exactly why this exists: if `CTE_INLINING` has collapsed
`WITH cte AS (SELECT col AS other FROM reader()) SELECT * WHERE other > 0
FROM cte` into a single scan, the filter arriving at `pushdown_get.cpp`
still carries the alias `other` — and a table function's
`pushdown_complex_filter` callback would receive a filter naming a column
that doesn't exist from its own point of view. This is exactly what
`test/optimizer/filter_pushdown_alias.cpp` — a C++ unit test, not a
`.test` SQL file — is built to catch. It registers a synthetic table
function whose `pushdown_complex_filter` callback records the alias of any
`BoundColumnRefExpression` bound to its own scan's `table_index`, runs

```sql
EXPLAIN SELECT * FROM (SELECT col AS other FROM alias_inspect()) sub WHERE other IS NOT NULL
```

and asserts the callback saw `col` — the scan's real column name — not
`other`, the alias the SQL text actually used. Without
`NormalizeColumnRefAliases`, a table-function author writing a
`pushdown_complex_filter` callback that pattern-matches on column name (a
CSV or Parquet reader's own filter logic does exactly this) would silently
fail to recognize its own column whenever the query happened to alias it.

**Two extension points, in order.** If the scan's `TableFunction` declares
`pushdown_complex_filter`, that callback gets first refusal over the *entire*
remaining filter list (`pushdown_get.cpp:134`) — this is the CSV/Parquet
reader's own filter-pushdown hook, entirely separate from the generic
`table_filters` mechanism described next, and it's allowed to consume some
filters, rewrite others, and leave the rest. Only *after* that does
`FilterCombiner::GenerateTableScanFilters` (`pushdown_get.cpp:165`) run,
converting whatever's left into the storage layer's generic `TableFilterSet`
— per-column, per-comparison-type filter objects the row-group zonemap
machinery consumes directly.

**Why `WITH ORDINALITY` and table in/out functions block partial pushdown.**
Two guards worth calling out precisely
(`pushdown_get.cpp:124` and `:149`): a scan that `assigns_ordinality` (i.e.
numbers its output rows via `WITH ORDINALITY`) refuses *any* filter pushdown
into the function itself, because pushing a filter into the function's own
row generation would renumber which rows survive *before* their original
ordinal position is assigned — the ordinality has to be computed against
every row the function *would* have produced, not the post-filter subset.
Similarly, a table in/out function (one that both consumes and produces
rows) can't accept a *partial* type-based filter pushdown
(`requires_partial_pushdown`), because there's no way to apply a filter to
only some of the columns flowing through such a function without
misinterpreting its input/output contract.

**Barrier filters at the scan get one extra restriction.**
`PushdownBarrierFilters` (`pushdown_get.cpp:45`) only pushes a barred filter
into the scan if (a) every other filter has *also* successfully landed in the
scan (`barrier_filters.empty() || !filters.empty()` guards this — if any
non-barred filter remains above the scan, the barred one must stay there too,
so evaluation order is preserved), and (b) the scan is a genuine DuckDB
table (`table->IsDuckTable()`) — an *external* scan (an extension-provided
table function) might evaluate a pushed expression against values a
DuckDB-native constant-folding pass would never construct (e.g. a partition
constant that no surviving row actually has), so barred expressions are kept
conservatively out of any scan DuckDB doesn't fully control.

### 2.3 `LOGICAL_PROJECTION` — `pushdown_projection.cpp` — the classic "rewrite references" case

This is the textbook example the task description names explicitly: pushing
a filter below a projection requires rewriting every column reference in the
filter from "the projection's output column" to "whatever expression the
projection computed to produce that output column." `ReplaceProjectionBindings`
(`pushdown_projection.cpp:57`) does exactly this — for every
`BoundColumnRefExpression` in the filter, it looks up
`proj.GetExpression(colref.Binding())` and substitutes a full copy of that
expression in place of the column reference.

Three cases are distinguished per filter, and the distinction matters:

1. **Volatile or throwing (and not barred).** `IsVolatile` walks the
   filter's column references and checks whether any of the *projection
   expressions they'd be rewritten into* is itself volatile
   (`pushdown_projection.cpp:13`) — if `SELECT random() AS r ... WHERE r <
   0.5` were rewritten naively, pushing the filter below the projection would
   mean `random()` gets evaluated once *per filter check* rather than once
   per row before the check, changing how many times a volatile function
   actually runs. Such filters are held back as a `remain_expressions` list
   and re-attached above the projection with `AddLogicalFilter`.
2. **`PRESERVE_COMPUTED_EXPRESSIONS` mode and the filter references a
   computed (non-bare-column) projection expression.** Ordinary
   `FILTER_PUSHDOWN` (pass #6) always constructs its `FilterPushdown`
   instances with the default `ALLOW_COMPUTED_EXPRESSIONS` mode, so this
   branch is dormant for the main pass — it's `CTE_FILTER_PUSHER`
   (`src/optimizer/cte_filter_pusher.cpp:181`) that opts into
   `PRESERVE_COMPUTED_EXPRESSIONS`, specifically because a filter derived at
   *one* reference site of a materialized CTE and pushed toward the CTE's
   shared definition must not silently duplicate an expensive computed
   expression once per reference site. When this mode is active,
   `SplitProjection` (`pushdown_projection.cpp:111`) inserts a *new*, lower
   projection that computes the shared expression exactly once, and rewrites
   the filter (and the original projection) to reference that lower
   projection's output instead of re-deriving the computation inline.
3. **The ordinary case.** The filter's column references get rewritten via
   `ReplaceProjectionBindings` and handed to a fresh `child_pushdown`, which
   recurses into the projection's child — this is what lets a filter written
   against a `SELECT`-list alias reach all the way down to the base table:
   `WHERE total > 100` on `SELECT l_quantity * l_price AS total FROM
   lineitem` becomes, below the projection, `WHERE l_quantity * l_price >
   100` — a filter the scan may or may not be able to use directly (a
   multi-column expression like this one generally lands as a `LogicalFilter`
   rather than a scan-level table filter — see §4's `TryPushdownGenericExpression`
   contrast for the cases that *can* still make it into the scan).

> **Two more reasons keep a filter above the projection.** The handler decides
> with two small lambdas:
>
> ```cpp
> // src/optimizer/pushdown/pushdown_projection.cpp:173
> 	auto stays_on_top = [&](const Filter &f) {
> 		return IsVolatile(proj, *f.filter) || ReferencesCompressExpression(proj, *f.filter) ||
> 		       (f.filter->CanThrow() && !f.has_barrier);
> 	};
> 	auto splits_off = [&](const Filter &f) {
> 		return projection_mode == ProjectionMode::PRESERVE_COMPUTED_EXPRESSIONS &&
> 		       ReferencesComputedExpression(proj, *f.filter);
> 	};
> 	// barred expressions may only cross the projection if every other filter does: a filter kept on top of the
> 	// projection, or split off below it, would otherwise be evaluated after them
> 	bool hold_back_barriers = false;
> 	for (auto &filter : filters) {
> 		if (!filter->has_barrier && (stays_on_top(*filter) || splits_off(*filter))) {
> 			hold_back_barriers = true;
> 		}
> 	}
> ```
>
> - **A filter over a compressed column stays on top**
>   (`ReferencesCompressExpression`, `pushdown_projection.cpp:26`). Compressed
>   materialization (part 27) puts `compress` calls into projections. A compress
>   expression is only valid at the place it was put, so a filter must not copy
>   it lower down. `test/optimizer/pushdown/filter_pushdown_compressed_projection.test`
>   builds a join whose derived filter would otherwise be inlined through such a
>   projection, and checks the sums (`95495000`) with and without the other
>   passes that can hide the problem.
> - **Barred filters wait for their neighbours.** A barred filter must run *after* the filters next to it (§1.3). If any other
>   filter stays above the projection (or is split off below it), the barred
>   filters stay above too. `test/sql/optimizer/secure_view_pushdown.test`
>   checks this with a secure view.

### 2.4 `LOGICAL_CROSS_PRODUCT` — `pushdown_cross_product.cpp` — where joins get manufactured from filters

A cross product's handler is the piece every join handler below eventually
delegates to (inner joins convert themselves *into* a cross product
specifically to reuse this logic — see §2.6). Given the accumulated filter
list, each filter's bindings are checked with `JoinSide::GetJoinSide` against
the left and right child's table-index sets:

- **`LEFT` or `RIGHT` only** — pushed straight into that side's own
  `FilterPushdown` instance.
- **`BOTH` or `NONE`** — this predicate references columns from *both*
  sides (or neither, e.g. a scalar subquery result), so it becomes a
  candidate **join condition**: `join_expressions.push_back(...)`. If there
  *are* such expressions, `LogicalComparisonJoin::ExtractJoinConditions` and
  `LogicalComparisonJoin::CreateJoin` turn the cross product into a genuine
  `INNER` join built directly from those predicates — this is precisely how
  `WHERE a.k = b.k` (written as an ordinary `WHERE`-clause equality between
  two cross-joined tables) becomes an honest equi-join operator rather than
  staying a cross product with a filter bolted on top, which matters because
  only a real join operator is visible to `JOIN_ORDER`'s DPhyp search
  (Chapter 8) as something reorderable.
- **A special carve-out for `IN`-clause candidates**
  (`InClauseRewriter::HasRewritableInClause`): a `BOTH`/`NONE` filter that
  looks like a long, foldable `IN (...)` list is deliberately *not* folded
  into a join condition here — it's deferred (`deferred_filters`) and
  re-attached as an ordinary `LogicalFilter` on top of the result instead, so
  the later, dedicated `IN_CLAUSE` pass (#9 in the fixed pipeline, §7.6.6 of
  the overview) still gets a chance to turn it into its own purpose-built
  mark join rather than having this handler's generic join-condition logic
  claim it first.

> **Transitive join bounds.** Before it sorts the filters into left, right and
> join, the handler may run them through the `FilterCombiner` once more:
>
> ```cpp
> // src/optimizer/pushdown/pushdown_cross_product.cpp:73
> 		// Recombine useful join bounds before distributing filters to the inputs.
> 		if (HasTransitiveJoinBounds(filters)) {
> 			if (PushFilters() == FilterResult::UNSATISFIABLE) {
> 				return make_uniq<LogicalEmptyResult>(std::move(op));
> 			}
> 			GenerateFilters();
> 		}
> ```
>
> `HasTransitiveJoinBounds` (`pushdown_cross_product.cpp:15`) returns `true` when
> two things are both present: a one-table filter that compares a column with a
> constant (for example `i > 4999`, or a `BETWEEN`), and a two-table `<`, `<=`,
> `>` or `>=` comparison whose right side is a column of that same table (for
> example `j >= i`). Then `PushFilters` + `GenerateFilters` let the combiner
> derive the new bound `j > 4999` (Part III, §3.3). That bound touches only one
> side, so it can go down into that side's scan as a table filter, where a
> zonemap can use it. Without this step the inequality would become a join
> condition before the combiner ever saw both facts together.

### 2.5 `LOGICAL_COMPARISON_JOIN` / `LOGICAL_ANY_JOIN` / `LOGICAL_ASOF_JOIN` / `LOGICAL_DELIM_JOIN` — `PushdownJoin` dispatch

`FilterPushdown::PushdownJoin` (`filter_pushdown.cpp:169`) is the shared
entry point for every join type; it computes `left_bindings`/`right_bindings`
once via `LogicalJoin::GetTableReferences` and then dispatches by
`join.join_type`:

```cpp
// src/optimizer/filter_pushdown.cpp:179
	// every branch keeps the join's projection maps on the operator it returns: the join itself when it survives,
	// the re-created join in PushdownInnerJoin, or a filter carrying the map when a side replaces the join
	switch (join.join_type) {
	case JoinType::OUTER:
		return PushdownOuterJoin(std::move(op), left_bindings, right_bindings);
	case JoinType::INNER:
		//	AsOf joins can't push anything into the RHS, so treat it as a left join
		if (op->type == LogicalOperatorType::LOGICAL_ASOF_JOIN) {
			return PushdownLeftJoin(std::move(op), left_bindings, right_bindings);
		}
		return PushdownInnerJoin(std::move(op), left_bindings, right_bindings);
	case JoinType::LEFT:
		return PushdownLeftJoin(std::move(op), left_bindings, right_bindings);
	case JoinType::MARK:
		return PushdownMarkJoin(std::move(op), left_bindings, right_bindings);
	case JoinType::SINGLE:
		return PushdownSingleJoin(std::move(op), left_bindings, right_bindings);
	case JoinType::SEMI:
	case JoinType::ANTI:
		return PushdownSemiAntiJoin(std::move(op));
	default:
		// unsupported join type: stop pushing down
		return FinishPushdown(std::move(op));
	}
```

(Each handler keeps the join's projection maps itself, as the comment says.)

The inline comment on the `ASOF` special case is worth pausing on: an `ASOF
JOIN` matches each left row to the *nearest preceding* right row by an
inequality condition, and DuckDB's ASOF implementation can only exploit that
ordering property from one particular physical join strategy — pushing a
filter into its right-hand input the way an ordinary inner join would risks
removing rows the ASOF matching needs to see (a row that gets filtered out
might have been the *nearest preceding* match for some surviving left row).
Treating it as a left join is the conservative fix: `PushdownLeftJoin`
already knows how to push into the left side only, and to leave the right
side untouched unless a special null-rejection proof (§2.7) licenses more.

### 2.6 Inner join — `pushdown_inner_join.cpp` — converting to a cross product to reuse §2.4's logic

`PushdownInnerJoin` doesn't have its own filter-side-classification logic at
all — it *manufactures* one. Every one of the join's own conditions (for a
comparison join, each `JoinCondition`; for an `LOGICAL_ANY_JOIN`, its single
arbitrary `condition` expression) is turned into an ordinary filter via
`AddFilter`, the join is then physically rebuilt as a `LogicalCrossProduct`
over the *same two children*, and the whole thing is handed to
`PushdownCrossProduct` (§2.4). This is exactly why an inner join's equality
condition participates in the `FilterCombiner`'s equivalence-class machinery
identically to a `WHERE`-clause predicate above the join — as far as the
combiner is concerned, `a.k = b.k` written as an explicit `JOIN ... ON`
condition and `a.k = b.k` written as a `WHERE` clause over a cross product
are the same kind of object by the time this handler is done with them. That
equivalence is the mechanical basis for §5's transitive-filter derivation.

`PushdownInnerJoin` also keeps the join's projection
maps. When `PushdownCrossProduct` hands back a rebuilt comparison join, the old
join's `left_projection_map`/`right_projection_map` are moved onto it and remapped
to the children's new column layout (`pushdown_inner_join.cpp:57`).

One structural subtlety: a `LOGICAL_DELIM_JOIN` with `JoinType::INNER` is
diverted first (`filter_pushdown.cpp` — see `PushFiltersIntoDelimJoin`,
`filter_pushdown.cpp:272`), since a delim join's LHS and RHS have a
correlation relationship `FlattenDependentJoins` (Chapter 6) set up that
ordinary cross-product logic doesn't understand; that helper instead
checks, filter by filter, whether *all* of a filter's bindings are satisfiable
by one specific child's own output bindings (not the join-side-partition
logic `JoinSide::GetJoinSide` uses elsewhere) before copying it down into
that child. With default settings this branch is not reached: the planner
has already turned every delim join into plain joins over materialized CTEs
(`delim_join_as_cte` defaults to `true`; Chapter 6 §6.4.1). It matters when
`delim_join_as_cte = false`.

### 2.7 Left join — `pushdown_left_join.cpp` — the "prove it's really an inner join" trick

A `LEFT JOIN`'s asymmetry is exactly the point of this handler. Filters
whose bindings match the **left** side alone are always safe to push into
the left child — the left side's rows all survive a left join regardless of
what the right side contributes, so narrowing the left input can only
narrow, never corrupt, the join's output.

Filters touching the **right** side (or both sides) are a different story,
and this is where `FilterRemovesNull` (`pushdown_left_join.cpp:77`) earns its
keep. The function copies the filter, replaces every `BoundColumnRefExpression`
bound to a right-side table with a typed `NULL` constant
(`ReplaceColRefWithNull`), runs the constant-folding expression rewriter over
the result, and checks whether the folded value comes out `FALSE` or `NULL`.
If it does, the filter is provably false for every row the left outer join
would have NULL-extended on the right — which means the outer join's entire
reason for preserving those NULL-extended rows is moot, *because this filter
above it would discard them anyway*. In that case:

```cpp
// src/optimizer/pushdown/pushdown_left_join.cpp:157-169
if (FilterRemovesNull(optimizer.context, optimizer.rewriter, filters[i]->filter.get(), right_bindings)) {
	// the filter removes NULL values, turn it into an inner join
	join.join_type = JoinType::INNER;
	// ... move all filters back into the pending list ...
	return PushdownInnerJoin(std::move(op), left_bindings, right_bindings);
}
```

the `LEFT` join is rewritten to `INNER` in place, and the whole operator is
re-dispatched through `PushdownInnerJoin` (§2.6) — which means it *also*
gets the full symmetric pushdown treatment an inner join enjoys, including
having its own join condition folded into the `FilterCombiner`'s equivalence
classes. This is the single most important correctness-preserving rewrite in
this handler: a naive implementation that pushed a right-side filter into an
outer join's right child *without* this proof would be wrong in general
(§Part IV works through exactly why), but proving the filter rejects every
NULL-extended row first makes the rewrite airtight rather than merely
convenient.

> **Note:** `FilterRemovesNull` folds the `NULL`-substituted filter with
> `ExpressionExecutor::TryEvaluateScalar`, not `EvaluateScalar`
> (`pushdown_left_join.cpp:96`). If folding *throws* (for example a division by
> zero that only shows up once the column is `NULL`), that conjunct proves
> nothing and is skipped. So the error does not escape at planning time for a
> query that would never evaluate that expression (test
> `test/optimizer/constant_argument_errors.test`).

If the join's own conditions are unsatisfiable (`has_unsatisfiable_condition`)
and the join is still `LEFT` (the null-rejection proof above didn't fire, or
wasn't attempted for a non-comparison condition), a different fix applies:
`CreateDummyRHS` (`pushdown_left_join.cpp:46`) manufactures a dummy,
all-`NULL` right-hand side with the correct column types and shape — the
left outer join semantics still say "every left row survives, right side
columns come out NULL for it," and if the right side can never actually
match (its join condition provably fails), that's exactly what a
cross-product against synthetic all-`NULL` rows also produces, cheaper. The
`FilterCombiner filter_combiner` local to this function
(`pushdown_left_join.cpp:119`) plays the same role as `PushdownOuterJoin`'s
coalesce mechanism will (§2.8): it seeds itself with the join's own
comparison conditions *and* every left-side filter, so that if a join
condition is `i = a` and there's a filter `i = 500` on the left, the combiner
can derive `a = 500` and offer it to the right side as a genuinely new,
independently pushable predicate (`filter_combiner.GenerateFilters`,
`pushdown_left_join.cpp:183`) — the comment in the source states this
explicitly: *"this happens if, e.g. a join condition is (i=a) and there is a
filter (i=500), we can then push the filter (a=500) into the RHS."*

### 2.8 Outer (`FULL OUTER`) join — `pushdown_outer_join.cpp` — the coalesced-equal-keys mechanism

This is the deepest, most surgical piece of correctness logic in the entire
file, and it deserves the full walkthrough the task asks for. A `FULL OUTER
JOIN` preserves *both* sides' unmatched rows — there is no "preserved side"
the way a `LEFT`/`RIGHT` join has one, so the null-rejection trick from §2.7
(rewrite to `INNER`) only applies in the degenerate case where a filter
rejects NULLs from *both* sides simultaneously. `PushdownOuterJoin`
(`pushdown_outer_join.cpp:174`) handles a narrower but very common real-world
shape instead: a `WHERE` clause written against `COALESCE(a.k, b.k)` — the
idiomatic way SQL authors write "the join key, whichever side actually has
a matching row" after a `FULL OUTER JOIN ... USING (k)`.

**The core insight, made precise.** For a `FULL OUTER JOIN ON a.k = b.k`,
*every row the join produces* satisfies one of exactly three shapes: (1) both
sides matched, so `a.k = b.k` and `COALESCE(a.k, b.k)` equals either one; (2)
only the left side is present, so `b.k` (and everything else on the right)
is `NULL`, and `COALESCE(a.k, b.k) = a.k`; (3) only the right side is
present, symmetric with `a.k`. In every one of these three cases, a predicate
`P(COALESCE(a.k, b.k))` is true for that output row **if and only if**
`P(a.k)` is true for shape (1)/(2) rows and `P(b.k)` is true for shape
(1)/(3) rows — because whichever side is actually populated for that row is
exactly the value `COALESCE` picked. So `P(COALESCE(a.k, b.k))` on the join's
output is provably equivalent to pushing `P(a.k)` into the left child and
`P(b.k)` into the right child **independently**, each computed entirely
within that child, with no reference to the other side at all — a
correctness argument that has nothing to do with proving the filter rejects
NULLs (§2.7's trick) and everything to do with the coalesce operator's own
definition.

**The matching code.** `PushDownFiltersOnCoalescedEqualJoinKeys`
(`pushdown_outer_join.cpp:96`) builds a lookup from every possible
"flattened coalesce of an equi-join condition's two sides" — `FlattenedCoalesce`
(`pushdown_outer_join.cpp:17`) exists specifically because
`coalesce(a, coalesce(b, c))` and `coalesce(a, b, c)` are the same function
semantically but different expression trees, so both orderings
(`coalesce(l, r)` and `coalesce(r, l)`) are registered against the same join
condition. Then, for each pending filter, it checks: does this filter contain
a `coalesce(...)` subexpression matching one of those registered join-key
pairs, and — critically — is the filter, once every occurrence of that
coalesce subexpression is replaced by a placeholder constant, a *pure
function of that coalesce value alone* (`ExprIsFunctionOnlyOf`,
`pushdown_outer_join.cpp:78`)? If a filter also references some *other*
column (not part of the coalesced pair), the rewrite is unsound — replacing
the coalesce with `a.k` on the left ignores whatever the other column
contributes on the right side's absent rows — so such filters are correctly
left alone. If it passes, `ReplaceIn` (`pushdown_outer_join.cpp:65`)
substitutes the join condition's raw left expression for the coalesce on one
copy, the raw right expression on another, and each copy is separately
pushed down (`pushdown_left`/`pushdown_right` callbacks).

**Walking `pushdown_filter_on_coalesced_equal_outer_join_keys.test` end to
end.** This is the dedicated test, and it's built almost entirely out of
same-label pairs — proving "the automatic rewrite" and "the SQL author's own
manual per-side filter" produce byte-identical optimized plans:

```sql
-- test/optimizer/pushdown/pushdown_filter_on_coalesced_equal_outer_join_keys.test
-- label: single_join
EXPLAIN SELECT id
FROM (SELECT id FROM t1) FULL OUTER JOIN (SELECT id FROM t2) USING (id)
WHERE id >= 2
----
-- (same label, no expected output — this is the assertion the two match)
EXPLAIN SELECT id
FROM (SELECT id FROM t1 WHERE id >= 2) FULL OUTER JOIN (SELECT id FROM t2 WHERE id >= 2) USING (id)
----
```

Because this query uses `USING (id)`, DuckDB's binder itself already
resolves the shared output column `id` to a `COALESCE(t1.id, t2.id)`-shaped
projection above the join — so `WHERE id >= 2` really is, under the hood, a
filter on that coalesce expression, and the test is directly exercising the
mechanism just described. The `single_join_isnull_filter` label pair repeats
this with `WHERE id IS NULL` — proving the mechanism also handles the
"neither side matched, or one side's key was itself NULL" case correctly,
not just simple range predicates. The `multiple_joins` label pair chains
*two* `FULL OUTER JOIN ... USING (id)` operators and shows the same
single-filter-becomes-three-independent-pushed-filters rewrite still fires
across both joins. `multiple_occurrences_of_the_same_coalesced_join_keys`
pushes a three-way `OR` (`id >= 2 OR id IN (1, 4) OR id IS NULL`) through —
proving `ExprIsFunctionOnlyOf`'s scan correctly recognizes the coalesce
expression appearing *multiple times inside one predicate tree* as still a
single, consistent substitution target.

The test also directly proves the coalesce ordering doesn't matter
(`left_right_coalesce` label, both `COALESCE(t1.id, t2.id) >= 2` and
`COALESCE(t2.id, t1.id) >= 2` optimize identically — exactly the reason
`FlattenedCoalesce::Of` registers both orderings), and it proves the
mechanism generalizes past bare column references to **any function of the
two join keys**, as long as both sides apply the *same* function:

```sql
-- nullif_func_join_keys
EXPLAIN SELECT *
FROM (SELECT id, a FROM t4) AS t4
FULL OUTER JOIN (SELECT id, a FROM t5) AS t5
ON nullif(t4.id, t4.a) = nullif(t5.id, t5.a)
WHERE coalesce(nullif(t4.id, t4.a), nullif(t5.id, t5.a)) < 3;
-- optimizes identically to writing nullif(id,a) < 3 on each side separately
```

and even a `LIST`-valued join key (`[t4.id, t4.a] = [t5.id, t5.a]`, with a
`coalesce(...)[0] < 4` filter on the composite key) is handled, since the
mechanism operates purely on expression-tree equality, not on any
type-specific logic.

**The negative cases, and why they're negative.** Three tests in this file
prove the boundary is respected, not just the happy path:

```sql
-- "should not pushdown filter on coalesced keys that also depend on other columns"
EXPLAIN SELECT id FROM t1 FULL OUTER JOIN t4 USING (id) WHERE id = a;
-- logical_opt   <!REGEX>:.*SEQ_SCAN.*Filters.*   -- no scan-level filter appears
```

`id = a` references `a` — a column that is *not* part of the coalesced pair
— so `ExprIsFunctionOnlyOf` correctly refuses the rewrite: substituting `t1`'s
raw `id` for the coalesce on the left branch would silently drop the
constraint's dependency on `a` for rows where `t4`'s side is the one that's
actually absent, and the resulting per-side filters would no longer be
equivalent to the original joined predicate.

```sql
-- "should not pushdown single filter containing coalesced keys from different join conditions"
EXPLAIN SELECT id FROM t4 FULL OUTER JOIN t5 USING (id, a) WHERE id IN (a, 1)
-- logical_opt   <!REGEX>:.*SEQ_SCAN.*Filters.*
```

`id IN (a, 1)` mixes two *different* join keys (`id` and `a`) in one
predicate — there's no single coalesce-of-a-join-condition this matches
against, so the lookup in `join_cond_by_coalesced_join_keys` simply never
fires for it.

```sql
-- "should not pushdown a volatile filter"
EXPLAIN SELECT id FROM t1 FULL OUTER JOIN t2 USING (id) WHERE trunc(random() * id) >= 2
-- logical_opt   <!REGEX>:.*SEQ_SCAN.*Filters.*
```

The volatility guard at the very top of the per-filter loop
(`filter->IsVolatile() || filter->CanThrow()`, `pushdown_outer_join.cpp:125`)
refuses to even attempt the rewrite here — the standard reasoning applies
identically to this mechanism as to every other pushdown decision: splitting
a volatile filter into two independently-evaluated copies (one per side)
would double how many times `random()` actually runs relative to evaluating
it once against the joined output.

**When no coalesce match is found at all,** `PushdownOuterJoin` simply calls
`FinishPushdown` (`pushdown_outer_join.cpp:191`) — an ordinary `FULL OUTER
JOIN` with a filter that doesn't fit this narrow, provably-safe shape gets
*no* pushdown at all through this handler; the filter is left sitting above
the join as a plain `LogicalFilter`, which is always correct (if
conservative) since evaluating the original predicate against the join's
actual output, after both sides have been fully NULL-extended and combined,
can never be wrong.

### 2.9 Semi/Anti join — `pushdown_semi_anti_join.cpp` — asymmetric by construction

A `SEMI` join returns left rows that *have* a match on the right (without
duplicating them per match, and without ever projecting right-side columns);
`ANTI` returns left rows that *don't*. Because neither join type ever
projects a right-side column, and because the right side's role is purely
"does at least one match exist," `PushdownSemiAntiJoin`
(`pushdown_semi_anti_join.cpp:12`) pushes *all* currently-pending filters
into the **left** child unconditionally (`RewriteChild(*op, 0)` —
note this reuses this instance's own `Rewrite` directly,
not a filtered subset, since every pending filter's bindings can only ever
be resolvable against columns the final output actually exposes, which for
a semi/anti join is exactly the left side's columns) and pushes nothing
external into the right side at all — `right_pushdown.Rewrite` only ever
sees filters that were *already inside* the right subtree before this
handler ran, never anything pushed down from above the join.

`table/pushdown_semi_anti_join.test`'s own inline comment on one of its
cases makes the asymmetry explicit:

```sql
-- test/optimizer/pushdown_semi_anti_join.test
-- filter is not pushed down to right side, otherwise answer is incorrect
select * from t2 semi join t3 on (t2.a + 70 + t3.b = t2.b**t3.a) where a < 10;
----
1   9
```

Pushing `a < 10` into `t3` would be a category error — `a` is `t2`'s column,
not `t3`'s, so this particular test is really guarding against a binding
mix-up rather than a semantic subtlety, but the broader point the test file
demonstrates across many cases is that semi/anti join results are
**existence facts about the left side**, and it would be actively wrong to
let a right-side-only filter narrow which right rows are visible during the
existence check while *also* letting some other part of the plan believe the
right side still has its original cardinality — since a semi/anti join
throws away the right side's row identity entirely, only whether-a-match-
exists is preserved.

The rest of the test file is a long, systematic sweep of **empty-result
short-circuiting**:

```sql
query II
explain select a from t0 semi join (select * from t1 where 1 = 0) on a = b;
----
logical_opt   <!REGEX>:.*SEMI.*         -- SEMI JOIN with an empty RHS has no matches: whole join is empty

query II
explain select * from (select 42 where 1=0) tmp0(a) ANTI JOIN (select 42) tmp1(b) on (a=b);
----
logical_opt   <!REGEX>:.*EXCEPT.*       -- left_empty && ANTI -> empty result (nothing to "not match")

query II
explain select * from t0 ANTI JOIN (select * from t1 where 1=0) on (a=b);
----
logical_opt   <!REGEX>:.*ANTI.*         -- right_empty && ANTI -> every left row survives: just return t0
```

This maps directly onto the handler's own logic
(`pushdown_semi_anti_join.cpp:20-49`): once both children have been
recursively rewritten, if either side collapsed to a `LogicalEmptyResult`
(most often because `STATISTICS_PROPAGATION` or an earlier pushdown pass
proved a subtree vacuous), the semi/anti semantics let the handler resolve
the *whole join* without touching the query at execution time at all —
`ANTI` with an empty right side degenerates to "return the left child"
(line 43), while every other
combination (`SEMI` with either side empty, `ANTI` with the left side empty)
collapses to `LogicalEmptyResult`.

> **Note:** the "return the left child" branch does not return the child bare.
> It calls `LogicalJoin::PromoteJoinSide(std::move(op->children[0]), std::move(join.left_projection_map))`.
> The comment reads *"nothing to match against: every left row passes, keep the
> join's output layout"*. If the `ANTI` join had a projection map, its parent
> expects only the mapped columns, in the mapped order. `PromoteJoinSide` keeps
> that layout (with a projection on top when needed) (test `test/sql/optimizer/anti_join_empty_rhs_projection_map.test`).

### 2.10 Mark join — `pushdown_mark_join.cpp` — converting MARK to SEMI/ANTI, and the safety check that gates it

A `MARK` join is how DuckDB implements a scalar subquery boolean result — the
`b IN (SELECT x FROM t)` shape produces a join whose *right-hand output* is a
single synthetic boolean column (the "marker," bound at `mark_index`) rather
than any of the right side's real columns, and that boolean records "did
this left row have a match." `PushdownMarkJoin` (`pushdown_mark_join.cpp:28`)
looks for exactly one very specific, very safe rewrite opportunity: **if the
only thing anything ever does with the marker is check its truthiness in a
filter, the mark join is behaviorally identical to a semi (or anti) join**,
which is strictly cheaper to execute (no boolean column to materialize and
carry around; the physical operator can short-circuit on first match).

```cpp
// src/optimizer/pushdown/pushdown_mark_join.cpp:58-69
if (filters[i]->filter->GetExpressionType() == ExpressionType::BOUND_COLUMN_REF && convert_mark_joins &&
    comp_join.convert_mark_to_semi) {
	// filter just references the marker: turn into semi join
	join.join_type = JoinType::SEMI;
	filters.erase_at(i);
	i--;
	continue;
}
```

`WHERE marker` (a bare reference, no negation, no further computation)
becomes `SEMI`; `WHERE NOT marker`, *conditioned on every join comparison
being a `null_values_are_equal` comparison* (`COMPARE_DISTINCT_FROM`/
`COMPARE_NOT_DISTINCT_FROM` — the shape a correlated `EXISTS` compiles to,
where NULL-handling has already been made total so the mark's result is
never itself `NULL`), becomes `ANTI`. Both conversions are gated by a second
flag, `comp_join.convert_mark_to_semi`, which is where `CheckMarkToSemi`
(§2.10.1 next) comes in.

#### 2.10.1 `CheckMarkToSemi` — the safety walk that runs *before* pushdown

`FilterPushdown::CheckMarkToSemi` (`filter_pushdown.cpp:33`) is invoked once,
at the very top of the `FILTER_PUSHDOWN` pass's callback
(`optimizer.cpp:289`), *before* `Rewrite` itself runs — it's a separate,
top-down, read-only pre-pass over the *original*, pre-pushdown tree. Its job:
walk down from the root, tracking which table-index bindings are referenced
by some operator *above* each mark join, and disable the semi/anti
conversion whenever the marker's own binding shows up in that set:

```cpp
// src/optimizer/filter_pushdown.cpp:55-65
case LogicalOperatorType::LOGICAL_COMPARISON_JOIN: {
	auto &join = op.Cast<LogicalComparisonJoin>();
	if (join.join_type != JoinType::MARK) {
		break;
	}
	// if an operator above the mark join includes the mark join index,
	// then the mark join cannot be converted to a semi join
	if (referenced_bindings.find(join.mark_index) != referenced_bindings.end()) {
		join.convert_mark_to_semi = false;
	}
	break;
}
```

The subtlety is in what counts as "referenced." `ExpressionsBecomeFilters`
(`filter_pushdown.cpp:19`) excludes an operator's *own* expressions from
counting against it if that operator's expressions are themselves destined
to become ordinary filters during pushdown anyway (a `LogicalFilter`, or an
`INNER`-type join/any-join/delim-join) — because in that case, referencing
the marker there is just the ordinary `WHERE marker` shape the conversion
exists to handle in the first place, and doesn't disqualify anything. What
*does* disqualify the conversion is an operator that consumes the marker's
**value**, not just its truthiness-as-a-filter — most concretely, a
projection that outputs the marker as one of the query's actual result
columns, since converting to `SEMI` would delete that column entirely (a
semi join, by definition, never has a marker output at all).

`no_mark_to_semi_if_mark_index_is_projected.test` is built exactly around
this:

```sql
-- test/optimizer/pushdown/no_mark_to_semi_if_mark_index_is_projected.test
WITH Example AS (
    SELECT
        c.my_key,
        (c.parent_key IN (SELECT my_key FROM BaseData)) AS parentExists,
        p.my_key IS NOT NULL AS parentExists2,
    FROM BaseData AS c
    LEFT JOIN BaseData AS p ON c.parent_key = p.my_key
)
SELECT *
FROM Example
WHERE parentExists
----
```

The `IN (SELECT ...)` subquery compiles to a `MARK` join whose boolean
result is `parentExists`. That value is used in **two** places: it's filtered
on (`WHERE parentExists`, which alone would license the `SEMI` conversion),
*and* it's selected as an ordinary output column of the query (`SELECT * ...
parentExists ...` propagates it all the way to the final projection). Because
the top-level projection is an operator that would *keep* the marker's
binding as one of its own referenced bindings (it doesn't fall under
`ExpressionsBecomeFilters`'s exemption — a projection's expressions don't
become filters), `CheckMarkToSemi`'s walk finds `mark_index` in
`referenced_bindings` at that projection and sets
`convert_mark_to_semi = false` before `Rewrite` ever runs. The test confirms
this directly:

```sql
EXPLAIN
WITH Example AS ( ... )
SELECT * FROM Example WHERE parentExists
----
logical_opt   <REGEX>:.*MARK.*      -- the MARK join survives
```
```sql
EXPLAIN
WITH Example AS ( ... )
SELECT * FROM Example WHERE parentExists
----
logical_opt   <!REGEX>:.*SEMI.*     -- and it is specifically NOT converted to SEMI
```

Without this guard, converting to `SEMI` here would silently make
`parentExists` unavailable to the outer `SELECT *` — not a wrong answer in
the sense of corrupting rows, but a plan that could no longer even be
constructed correctly, since the column the query asks for would no longer
exist anywhere in the tree.

A second, unconditional case lives right next to this one:
`filter_pushdown_into_subquery.test`'s title states it plainly — *"When
Aggregate and Group By operators contain a mark join index, Mark join ->
Semi join cannot happen."* Its concrete repro is a mark join whose result is
consumed **inside a parent join's own condition**, not filtered above it at
all:

```sql
-- test/optimizer/filter_pushdown/filter_pushdown_into_subquery.test
WITH vals(x) AS (VALUES (1))
SELECT l.a, r.b
FROM (VALUES (1), (2)) l(a)
LEFT JOIN (VALUES (1), (3)) r(b)
  ON l.a = r.b AND r.b IN (SELECT x FROM vals)
ORDER BY ALL;
----
1	1
2	NULL
```

Here the marker feeds directly into the outer `LEFT JOIN`'s own condition
(`AND r.b IN (...)`), which is structurally a `LOGICAL_DELIM_JOIN` case in
`CheckMarkToSemi`'s `switch` — and that branch disables the conversion
*unconditionally*, independent of any binding-set check, with a comment
citing the actual regression this guards: *"Duplicate-eliminated correlated
subqueries must keep MARK semantics; converting to SEMI can drop correlation
for nested RHS shapes (issue #22267)."* The subtlety here is one level more
severe than the projected-column case: a delim join's correlation machinery
depends on the marker's value flowing correctly through nested structure that
a semi join's simpler execution model isn't built to preserve, so this path
never even attempts the binding-overlap check — it disables the conversion
outright for every `MARK`-typed delim join, regardless of what's above it.

> **Note:** with default settings this query does not reach `CheckMarkToSemi`
> as a `LOGICAL_DELIM_JOIN`. `delim_join_as_cte` defaults to `true`, so no
> `LOGICAL_DELIM_JOIN` reaches the optimizer (Chapter 6 §6.4.1). The
> `LOGICAL_DELIM_JOIN` branch (`filter_pushdown.cpp:46-52`) only runs when
> that setting is `false`. The test itself only checks the result rows, so it
> passes in both forms.

Finally, when a mark-to-semi conversion *does* succeed, one more
NULL-semantics cleanup runs immediately afterward:
`SimplifyNullSafeSemiJoinConditions` (`pushdown_mark_join.cpp:10`). A `MARK`
join's conditions are sometimes built with `COMPARE_NOT_DISTINCT_FROM`
(NULL-safe equality) specifically so the mark's three-valued result (true /
false / unknown) can be computed correctly. Once the join becomes an
ordinary `SEMI` join, that three-valued subtlety is gone — a semi join is a
simple existence check — so if either join key is provably `NOT NULL`
(`NotNullExpressionAnalyzer::IsNotNull`), the condition is downgraded back to
plain `COMPARE_EQUAL`. This isn't just cosmetic: plain equality unlocks the
existing runtime join-filter infrastructure (`JOIN_FILTER_PUSHDOWN`, pass #42
— the min/max runtime filter described in §7.6.5 of the overview), which
doesn't have a code path for null-safe comparisons; NULL-safe equality would
silently opt this join out of that optimization if left unconverted.

### 2.11 Single join — `pushdown_single_join.cpp` — the scalar-subquery cousin of a left join

A `SINGLE` join is DuckDB's operator for an uncorrelated scalar subquery
(`SELECT (SELECT max(x) FROM t) ...`) — like a left join, it must preserve
exactly one output row per left input row (never zero, never more than one),
so its pushdown handler is structurally the simplest asymmetric case in the
file: push every left-bound filter into the left child, push *nothing* into
the right child at all (`pushdown_single_join.cpp:15-24` — there isn't even
a coalesce-style rewrite attempted here, since a `SINGLE` join's right side
producing a scalar is a fundamentally different shape from a `FULL OUTER
JOIN`'s per-row NULL extension), and finish with `PushFinalFilters` for
whatever's left (typically both-side or right-only filters, which must wait
until after the join actually runs).

### 2.12 Aggregate — `pushdown_aggregate.cpp` — pushing below `GROUP BY`, one grouping set at a time

Pushing a filter below an aggregate is safe only when the filter is
expressible entirely in terms of the *pre-aggregation* rows — i.e. it's a
predicate on a `GROUP BY` key, not on an aggregate function's result (a
`HAVING`-style filter on `SUM(x) > 100` can never be pushed below the
aggregate, since the sum doesn't exist yet at that point in the tree).
`PushdownAggregate` (`pushdown_aggregate.cpp:38`) checks this precisely:

- A filter whose bindings include `aggr.aggregate_index` (an aggregate
  function's own output) or `aggr.groupings_index` (the `GROUPING()`
  meta-function's output) is left alone entirely.
- A filter on a plain group column can only be pushed if the query has at
  least one grouping set at all (`aggr.groups.empty()` blocks pushdown
  entirely for a `GROUP BY ()`/plain aggregate with no groups), and — this
  is the part that matters for `ROLLUP`/`CUBE`/`GROUPING SETS` — the filter
  must reference columns present in **every** grouping set
  (`pushdown_aggregate.cpp:70-81`): a `ROLLUP(a, b)` expands into multiple
  grouping sets (`{a,b}`, `{a}`, `{}`), and a filter on `b` can't be pushed
  below the aggregate if one of those expanded grouping sets doesn't even
  include `b` as a grouping key — the filter would be applied to rows that,
  for that particular grouping set, haven't been grouped by `b` at all.
- Volatility is checked the same way as the projection handler
  (`IsVolatile`, `pushdown_aggregate.cpp:12`): if the group-key expression
  the filter's column reference maps back to is itself volatile, the filter
  stays above the aggregate.
- A filter that is volatile *itself* also stays
  above the aggregate, even if it only touches group keys
  (`f.filter->IsVolatile() || IsVolatile(aggr, *f.filter)`,
  `pushdown_aggregate.cpp:85`). The comment says why: *"a volatile filter is
  evaluated once per group, not once per input row"*. Pushing
  `HAVING nextval('s1') <= p + 1` below the `GROUP BY` would call `nextval` once
  per input row instead of once per group.

### 2.13 Distinct — `pushdown_distinct.cpp` — plain `DISTINCT` yes, `DISTINCT ON` not yet

`PushdownDistinct` (`pushdown_distinct.cpp:7`) is a short dispatch: plain
`DISTINCT` (no explicit `ON` clause) lets every non-volatile filter pass
straight through, because removing duplicate rows and filtering rows commute
freely regardless of order — `DISTINCT(filter(R)) = filter(DISTINCT(R))`
always, since filtering can only ever remove rows, never merge or split what
"duplicate" means. `DISTINCT ON`, however, is explicitly left unhandled
(`FinishPushdown`) — its own inline comment says "no pushdown through
`DISTINCT ON` (yet?)" and cites a real regression, `#19327`: `DISTINCT ON`
picks a specific *representative* row per group (by some `ORDER BY`), so a
filter evaluated *before* that selection could change which row within a
group would have won the "keep one" competition, versus a filter evaluated
*after* — the two orders are not equivalent in general, unlike plain
`DISTINCT`.

> **Volatile filters are the exception.** Plain `DISTINCT` does not let
> *volatile* filters through. Above the `DISTINCT`, a filter runs once per
> distinct row. Below it, it would run once per input row, including
> duplicates. So `PushdownDistinct` takes the volatile filters out first and puts them back on top with
> `AddLogicalFilter` (comment: *"volatile filters stay above the DISTINCT, where
> they run once per distinct row"*). `FilterPullup::PullupDistinct` has the
> matching rule: a volatile predicate that was pulled up stays *below* the
> `DISTINCT` (`filter_pullup.cpp:125`).
> `test/optimizer/pushdown/volatile_filter_aggregate_distinct.test` counts
> `nextval` calls:
>
> ```sql
> -- dd holds a = 1, 1, 2, 2, 3
> SELECT count(*) FROM (SELECT DISTINCT a FROM dd) q WHERE nextval('s4') <= 2;
> -- 2
> SELECT currval('s4');
> -- 3          (one call per distinct value, not five)
>
> -- agg holds four rows with p = 1
> SELECT SUM(x) FROM agg GROUP BY p HAVING nextval('s1') <= p + 1;
> -- 100
> SELECT currval('s1');
> -- 1          (one call for the one group)
> ```

### 2.14 Set operations (`UNION`/`INTERSECT`/`EXCEPT`) — `pushdown_set_operation.cpp` — rewrite-and-recurse, then empty-result propagation

Every set operation's children logically share one output binding
(`setop.table_index`), even though physically each child has its own,
different bindings — so `ReplaceSetOpBindings`
(`pushdown_set_operation.cpp:14`) rewrites a *copy* of each pending filter
per child, mapping the shared set-op-level column binding onto that
specific child's real binding, before recursing into it. This is why
pushing a filter through a `UNION`/`INTERSECT`/`EXCEPT` fans a single filter
out into as many independently-pushed copies as there are children — each
child gets to push its own copy as far down as its own shape allows,
completely independently of what the sibling children do.

`pushdown_set_op.test` is the evidence file, and its most interesting
coverage is what happens *after* pushdown when one side collapses to an
empty result — because `INTERSECT`/`EXCEPT` (unlike `UNION`) have
asymmetric emptiness rules that this handler must get exactly right:

```sql
-- intersect is empty if EITHER side is empty
explain select 42 intersect select 42 where 1=0;
----
logical_opt   <!REGEX>:.*INTERSECT.*    -- collapses to LogicalEmptyResult
explain select 42 where 1=0 intersect select 42;
----
logical_opt   <!REGEX>:.*INTERSECT.*    -- same, regardless of which side is empty

-- except is empty if the LEFT side is empty (nothing to subtract from)
explain select 42 where 1=0 except select 42;
----
logical_opt   <!REGEX>:.*EXCEPT.*

-- but except with an empty RIGHT side just returns the left side untouched
explain select 42 except all select 42 where 1=0;
----
logical_opt   <!REGEX>:.*EXCEPT.*       -- optimized away entirely, becomes the left child
```

The handler's own code encodes exactly this asymmetry
(`pushdown_set_operation.cpp:74-104`): `INTERSECT` treats *either* side being
empty as "the whole result is empty" (an intersection with nothing is
nothing), `EXCEPT` treats an empty *left* side the same way (subtracting
from nothing yields nothing) but treats an empty *right* side as "return the
left child verbatim" (subtracting nothing changes nothing) — and when it
does that, it has to re-point the surviving child's `LogicalProjection`
table index at the set operation's own `table_index`
(`pushdown_set_operation.cpp:93-95`), since the caller above expects to see
the set operation's bindings, not the individual child's.

### 2.15 `UNNEST` — `pushdown_unnest.cpp` — blocking on the unnest's own output index

A filter can be pushed below a `LOGICAL_UNNEST` node exactly when it doesn't
reference the unnested column itself — `PushdownUnnest`
(`pushdown_unnest.cpp:9`) checks each filter's bindings against
`unnest.unnest_index` and, if the filter touches it, holds it back as a
`remain_expressions` entry re-attached above the unnest with
`AddLogicalFilter`. This is exactly the right boundary: `UNNEST` is
row-multiplying (one input row with a 3-element list becomes three output
rows), so a filter on the *unnested* value fundamentally can't be evaluated
until after that multiplication has happened, while a filter on some other,
un-unnested column of the same row is safe to push straight through
unchanged, since it applies identically to every row the unnesting produces
from that one input row.

`pushdown_unnest_into_cte.test` demonstrates the positive case reaching all
the way to the scan, through both a bare `UNNEST(...)` and the equivalent
`generate_subscripts(...)`/`UNNEST(...)` pairing used to also expose the
list index:

```sql
-- test/optimizer/pushdown/pushdown_unnest_into_cte.test
EXPLAIN SELECT id1, element
FROM (SELECT id1, UNNEST(somelist) AS element FROM tbl2) tmp
WHERE id1=10;
----
logical_opt   <REGEX>:.*UNNEST.*SEQ_SCAN.*Filters.*
```

`id1=10` doesn't touch the unnested `somelist` values at all, so it sails
through the `LOGICAL_UNNEST` node and lands as a genuine scan-level
`Filters:` entry beneath it — the `UNNEST` operator still sits above the
scan in the printed plan (unnesting still has to happen), but every row it
processes has already been narrowed to `id1=10` before it ever reaches the
unnest step.

### 2.16 Window — `pushdown_window.cpp` — pushdown gated on partition-key membership, not `ORDER BY`

A window function's frame is defined per-partition, and a filter can only be
pushed below a `LOGICAL_WINDOW` node if it can be proven the filter's
selectivity is *entirely determined before* partitioning even matters — the
narrowest, always-safe case is a filter that references only columns that
are **`PARTITION BY` keys of every window expression in the node** (plural —
a single `LogicalWindow` can carry several distinct window functions with
different `PARTITION BY` clauses at once). `CanPushdownFilter`
(`pushdown_window.cpp:14`) checks exactly this: for each window expression's
partition-binding set, is this filter's binding set a subset of it? Only if
*every* window expression's partition passes does the filter qualify
(`filter_on_all_partitions`, line 25) — a filter safe with respect to one
window function's partitioning but not another's can't be pushed, because
doing so would change the rows visible to the *other* function's frame
computation.

Two additional hard blocks apply before any of this: a window expression
with **no** `PARTITION BY` at all disqualifies pushdown entirely
(`pushdown_window.cpp:48-53` — with no partitioning, every row in the input
potentially affects every other row's frame, e.g. an unbounded running
total, so nothing can be pushed without risking changing which rows
contribute to which computed value), and a **volatile** filter is excluded
regardless of partition membership (`pushdown_window.cpp:82`).

`pushdown_window_partition_filter.test` is an unusually rich test file
because it demonstrates both the positive case and three distinct negative
cases side by side, with actual result-set comparisons (not just `EXPLAIN`
regexes) proving the semantic boundary, not just the syntactic one:

```sql
-- can't push down c!='B': RANK()'s values for surviving rows DEPEND on
-- the existence of the c='B' rows, even though c='B' rows themselves
-- get filtered out afterward
select * from (
	select a, c, sum(b) OVER (PARTITION BY c), rank() OVER (PARTITION BY d ORDER BY b)
	from partition_and_rank_me
) where c != 'B' order by all;
----
A	A	30	1
A	A	30	2
D	C	110	5
D	C	110	6
```

Notice the `rank()` values here are `1, 2, 5, 6` — *not* renumbered to
`1, 2, 3, 4` — precisely because `rank() OVER (PARTITION BY d ...)` and the
filter's own predicate (`c != 'B'`) partition on **different** columns
(`d` vs `c`). The filter can't be pushed below the window, so `rank()`
computes its values against the *full*, unfiltered partition-by-`d` input
first, and only afterward does the filter remove the `c='B'` rows — leaving
gaps in the rank sequence that prove the filter really did run after window
evaluation, not before. Contrast this with a filter that *is* on the
partition column shared by every window expression:

```sql
select * from (
	select a, b, c, sum(a) OVER (PARTITION BY b, c), sum(b) OVER (PARTITION BY a, c) from t2
) where c = 20;
----
-- logical_opt   <REGEX>:.*WINDOW.*c = 20.*   -- filter appears pushed down, inside the WINDOW's own subtree
```

and one more subtlety the file specifically calls out: even a filter that
*syntactically* mentions only partition columns doesn't automatically
qualify if it's wrapped in an `AND` with a non-partition column —

```sql
-- One filter clause is on the partitioned column but the filter clause is an
-- AND conjunction, so we don't push that down.
select * from (
	select a, c, sum(b) OVER (PARTITION BY c), rank() OVER (PARTITION BY c ORDER BY b)
	from partition_and_rank_me
) where (c = 'B' AND a = 'D') order by all;
----
D	B	70	2       -- note: sum(b) computed over the FULL c='B' partition (70), not just a='D' rows
```

versus the same predicate connected with `OR` instead of `AND`, which *does*
qualify, because `FilterPushdown::ExtractFilterBindings` collects bindings
from the whole conjunct as a single unit, and an `OR`-combined predicate
where every branch is a comparison against the exact same window-partition
column (recognized here structurally, not deeply reasoned about) is treated
as pushable in a way a mixed-column `AND` is not — the two results (with and
without manual pushdown) are shown to be identical in the test, confirming
the rewrite really is safe for that specific `OR` shape.

### 2.17 `LIMIT` — `pushdown_limit.cpp` — the "generally NOT safe" case, reduced to one narrow exception

This is the shortest handler in the file, and it is short specifically
*because* almost nothing is safe here: pushing a filter below a `LIMIT`
would change *which* rows the limit keeps (a `LIMIT 10` after a filter keeps
the first 10 *surviving* rows; a `LIMIT 10` before the same filter keeps
some *different* set of up to 10 rows, of which possibly none survive the
filter at all) — so `PushdownLimit` (`pushdown_limit.cpp:9`) implements
exactly one special case and defers everything else to `FinishPushdown`
(which, recall, means "recurse into children with a *fresh* pushdown
instance, and stop"): a syntactically constant `LIMIT 0` collapses the
entire operator straight to `LogicalEmptyResult`, since a limit of zero rows
needs no input at all, regardless of what any filter above or below it
would have computed. Everything else — `LIMIT 10`, `LIMIT n` for a bound
parameter, `LIMIT` with an `OFFSET` — blocks pushdown through it entirely,
which is exactly the negative case the task description asks to confirm:
"pushing below a LIMIT is generally NOT safe since it would change which
rows survive the limit," and DuckDB's implementation agrees completely,
special-casing only the one instance (`LIMIT 0`) where there's provably no
row-selection behavior left to disturb.

### 2.18 Secure view — `pushdown_secure_view.cpp` — barriers, frozen cardinality, and reported (not executed) filters

A `LogicalSecureView` is DuckDB's boundary node for a row/column-security
view — the query author sees the view's declared output, but the actual
operators computing that output are deliberately hidden from `EXPLAIN` and
from the caller's ability to reason about the view's internals. Three
distinct correctness concerns interact here:

1. **Cardinality must be frozen *before* anything is pushed in.**
   (`pushdown_secure_view.cpp:38-43`) If the caller pushes a highly selective
   filter into the view, the view's *own* estimated cardinality (used by
   `JOIN_ORDER`'s cost model, Chapter 8) must still reflect what the view
   would produce *before* that caller-supplied filter narrows it — otherwise
   a caller with a very selective predicate could make the view boundary
   itself look artificially cheap, corrupting cost estimation for anything
   else joined against it. Freezing is skipped only for a view that
   explicitly `propagate_statistics` — i.e. one that already exposes its
   internal contents' real statistics honestly, so there's nothing to
   protect by freezing.
2. **A pushed filter is recorded for `EXPLAIN` purposes, whether or not it
   survives.** `secure_view.pushed_filters.push_back(expr->ToString())`
   (`pushdown_secure_view.cpp:56`) happens for *every* filter pushed toward
   the view, regardless of what happens to it afterward — this is how
   `EXPLAIN` can report "a filter was pushed into this view" as a single
   summary line at the boundary node, without ever having to reveal the
   internal operators the filter actually landed on.
3. **Barrier-wrapping is applied defensively, not just passed through.**
   (`pushdown_secure_view.cpp:57`) Even a filter that *wasn't already* marked
   as a barrier gets wrapped in one here if `ExpressionBarrier::Required`
   says evaluating it is externally observable — because once a filter
   crosses into a secure view's internals, the caller has no visibility into
   what row-removing operators might sit between the view's boundary and
   wherever the filter ends up, so the same "never evaluate on rows an
   operator below already removed" guarantee (§1.3) has to be established
   fresh at this boundary, rather than trusted to have already held on the
   way in.

---

## Part III — the `FilterCombiner`'s equivalence classes: pushdown's cleverest trick

Every operator handler above eventually funnels its filters through the same
`FilterCombiner` (`src/optimizer/filter_combiner.cpp`), and this is where the
single cleverest piece of logic in the entire pass lives: **deriving a brand
new predicate that was never written anywhere in the original SQL, purely
from an equality plus an existing constraint.**

### 3.1 Equivalence classes: `x = y` merges two buckets

`AddBoundComparisonFilter`'s "comparison between two non-scalars" branch
(`filter_combiner.cpp:1151-1191`) is what happens when the combiner sees an
equality between two column expressions (`x = y`, whether that equality came
from a `WHERE` clause or — as §2.6 established — from an inner join's own
condition, converted into a filter identically): `GetEquivalenceSet` assigns
(or looks up) a small integer ID per distinct expression, and when `x` and
`y` turn out to belong to *different* IDs, the combiner literally merges the
smaller bucket's members and known constant-comparison facts into the
larger one's:

```cpp
// src/optimizer/filter_combiner.cpp:1173-1190
auto &left_bucket = equivalence_map.find(left_equivalence_set)->second;
auto &right_bucket = equivalence_map.find(right_equivalence_set)->second;
for (auto &right_expr : right_bucket) {
	equivalence_set_map[right_expr] = left_equivalence_set;
	left_bucket.push_back(right_expr);
}
for (auto &right_constant : right_constant_bucket) {
	if (AddConstantComparison(left_constant_bucket, right_constant) == FilterResult::UNSATISFIABLE) {
		return FilterResult::UNSATISFIABLE;
	}
}
```

After this merge, `x` and `y` are, as far as every subsequent filter the
combiner processes is concerned, *interchangeable members of the same
class*. `GenerateFilters` (`filter_combiner.cpp:162`) is what turns that
internal bookkeeping back into concrete, emittable predicates: for every
member of a merged equivalence class, it re-derives an equality against
*every other member* (`entries[i] = entries[k]` for all pairs) and a
comparison against *every known constant fact* attached to the class
(`entries[i] <cmp> constant`, for every recorded `<`, `<=`, `>`, `>=`, `=`
fact) — including facts that arrived from a completely different member of
the class than the one being emitted for. This is exactly how, given
`a.k = b.k` (an equality, merging `a.k` and `b.k` into one class) and
`a.k > 100` (a constant fact attached to that same class via whichever
member it was originally stated against), `GenerateFilters` emits **both**
`a.k > 100` *and* `b.k > 100` as independent output filters — even though
only one of the two was ever written by the query's author.

### 3.2 `pullup_filters.test`'s equality-derivation cases, read against this mechanism

`pullup_filters.test` is titled around `FILTER_PULLUP` (Part IV explains that
pass properly), but a good third of its cases are really demonstrating
*this* equivalence-class mechanism, once pullup has done its job of exposing
a deeply-nested filter at the same tree level as a join condition:

```sql
-- test/optimizer/pullup_filters.test
## INNER JOIN: pull up a single filter in cross product from LHS
EXPLAIN SELECT * FROM (SELECT * FROM vals1, vals2 WHERE i=5) tbl1, (SELECT * FROM vals1, vals2) tbl2 WHERE tbl1.i=tbl2.i
----
physical_plan   <REGEX>:.*= 5.*= 5.*
```

Here `i=5` is written only on `tbl1`'s side, three levels of subquery deep.
`FILTER_PULLUP` (Part IV) hoists it up to sit alongside `tbl1.i=tbl2.i`
before `FILTER_PUSHDOWN` runs; once both are ordinary filters at the same
level, `AddBoundComparisonFilter`'s scalar-comparison branch attaches the
constant fact `i = 5` to `tbl1.i`'s equivalence class, the join condition
`tbl1.i = tbl2.i` merges `tbl2.i` into that same class, and `GenerateFilters`
emits `= 5` **twice** — once per side — which is exactly the `.*= 5.*= 5.*`
the regex is checking for in the physical plan (both `tbl1` and `tbl2`'s
scans end up with their own, independently pushed `i = 5` table filter).

### 3.3 Non-equality transitivity: `AddTransitiveFilters` and `transitive_filters.test`

Equality merging (§3.1) is only half the story — `AddTransitiveFilters`
(`filter_combiner.cpp:1308`) handles the strictly harder case of a
**non-equality** comparison between two column expressions, e.g. `j >= i`
where `i` and `j` are different, non-equal columns. This can't simply merge
two equivalence classes into one (they genuinely denote different values),
but it *can* still derive a new constant fact by chaining the inequality
through whatever constant facts are already known about one side:

```cpp
// src/optimizer/filter_combiner.cpp:1342 (the comment that heads the loop)
	// read every constant filters already inserted for the right scalar variable
	// and see if we can create new transitive filters, e.g., there is already a filter i > 10,
	// suppose that we have now the j >= i, then we can infer a new filter j > 10
```

> **Volatile and throwing comparisons derive nothing.** The top of
> `AddTransitiveFilters` has this check:
>
> ```cpp
> // src/optimizer/filter_combiner.cpp:1312
> 	// Derived bounds must not duplicate volatile or throwing expressions.
> 	if (comparison.IsVolatile() || comparison.CanThrow()) {
> 		return FilterResult::UNSUPPORTED;
> 	}
> ```
>
> A derived bound is a *copy* of part of the original comparison. If that
> comparison calls `nextval(...)` or a cast that can fail, a copy would run it
> again, or on other rows. `test/sql/filter/transitive_filter_bounds.test`
> checks this:
>
> ```sql
> SELECT i FROM range(5) t(i) WHERE i > 0 AND nextval('transitive_calls') >= i ORDER BY i;
> -- 1 2 3 4
> SELECT last_value BETWEEN 4 AND 5 FROM duckdb_sequences() WHERE sequence_name='transitive_calls';
> -- true       (nextval ran once per row, not once more for a derived bound)
> ```

`transitive_filters.test` walks through every combination of "which
comparison operator connects the two columns" crossed with "which comparison
operator connects one column to a constant," checking that the *correct*
strictness (`>` vs `>=`) survives the chain:

```sql
-- col0=5 AND col1>=col0  =>  col1 >= 5   (equality doesn't weaken the bound)
EXPLAIN SELECT * FROM (VALUES(5,5)) tbl(col0, col1) WHERE col0=5 AND col1 >= col0
----
logical_opt   <REGEX>:.*col1 >= 5.*

-- col0>5 AND col1>=col0  =>  col1 > 5    (strict > wins over non-strict >=)
EXPLAIN SELECT * FROM (VALUES(5,5)) tbl(col0, col1) WHERE col0>5 AND col1 >= col0
----
logical_opt   <REGEX>:.*col1 > 5.*

-- col0>=5 AND col1>=col0  =>  col1 >= 5  (both non-strict: result stays non-strict)
EXPLAIN SELECT * FROM (VALUES(5,5)) tbl(col0, col1) WHERE col0>=5 AND col1 >= col0
----
logical_opt   <REGEX>:.*col1 >= 5.*
```

and, symmetrically, the reversed clause order (`col1 >= col0 AND col0=5`,
rather than `col0=5 AND col1 >= col0`) is checked to derive the identical
result — proving the combiner's derivation is order-independent, as an
equivalence-class-based mechanism should be, regardless of which conjunct
the parser happened to see first. The file also directly tests
unsatisfiability detection falling out of the *same* machinery:

```sql
EXPLAIN SELECT * FROM (VALUES(1),(2),(3)) tbl(a) WHERE a > 1 AND a <= 1;
----
logical_opt   <REGEX>:.*EMPTY_RESULT.*
```

`AddConstantComparison` (`filter_combiner.cpp:76`) is what catches this: when
a new constant fact (`a <= 1`) is compared against an already-recorded fact
in the same equivalence class (`a > 1`), `CompareValueInformation` can
determine the pair is jointly impossible, returning
`ValueComparisonResult::UNSATISFIABLE_CONDITION` — at which point the entire
subtree gets short-circuited to `LogicalEmptyResult` immediately, without
ever reaching the scan or running the query at all.

> **Multi-hop bounds.** The file's "Complex transitive filters" section checks
> bounds that travel across a cross product and along a chain of columns:
>
> ```sql
> -- test/optimizer/transitive_filters.test
> EXPLAIN SELECT * FROM (VALUES(5000,5000)) tbl1(i,j), (VALUES(NULL, NULL, 5000, 5000)) tbl2(A, B, k,l)
> WHERE i>4999 AND j>=i AND k>=j;
> -- logical_opt   <REGEX>:.*(j > 4999.*k > 4999|k > 4999.*j > 4999).*
>
> EXPLAIN SELECT * FROM (VALUES(5000,5000)) tbl1(i,j), (VALUES(NULL, NULL, 5000, 5000)) tbl2(A, B, k,l)
> WHERE i<4999 AND k<=j AND j<=i AND l<=k;
> -- logical_opt   <REGEX>:.*(k < 4999.*l < 4999|l < 4999.*k < 4999).*
> ```
>
> `i > 4999` and `j >= i` give `j > 4999`; then `k >= j` gives `k > 4999`. The
> cross-product step from §2.4 is what lets the bound reach `tbl2`'s side.
> A last case checks that a column is not "mirrored" onto itself:
> `WHERE i < 1 AND abs(i) = 1` must not show `abs(i) = 1` twice.

---

## Part IV — `FILTER_PULLUP`: exposing buried filters so pushdown can reach them

### 4.1 Why this pass exists at all, precisely

Part III's equivalence-class trick only works on filters that `FilterCombiner`
actually gets to see *together, at the same level of the tree* — and
`FilterPushdown` only accumulates filters as it walks **downward** from
whatever a parent has already collected. If a selective filter is sitting
several operators *below* a join, on one child's own subtree, ordinary
top-down pushdown never brings it back up to be compared against the join's
own condition — pushdown only ever pushes what it's carrying *down*, it has
no mechanism to look *up* into a child it hasn't recursed into yet, discover
a filter there, and combine it with something from a sibling. `FilterPullup`
exists specifically to close this gap, one pipeline step *before* pushdown
runs (`FILTER_PULLUP` is pass #5, `FILTER_PUSHDOWN` is pass #6,
`optimizer.cpp:280-291`) — it hoists a filter that's buried inside one join
child up to sit as an ordinary sibling of the join's own condition, so that
when `FILTER_PUSHDOWN` runs immediately afterward, both are visible to the
*same* `FilterCombiner` instance at the *same* tree level, and Part III's
machinery gets a chance to do its work on a case it otherwise never would
have seen.

### 4.2 The mechanics: `can_pullup`, `PullupBothSide`/`PullupFromLeft`, and turning a join into a filtered cross product

`FilterPullup::Rewrite` (`filter_pullup.cpp:13`) dispatches on operator type
much like pushdown does, but the traversal direction and default posture are
inverted: a fresh `FilterPullup` instance starts with `can_pullup = false`
(`filter_pullup.hpp:20`), meaning by default nothing gets pulled anywhere —
`FinishPullup` (`filter_pullup.cpp:149`) just recurses into children with
*their own* fresh, `can_pullup=false` instances and stops. `can_pullup`
only flips to `true` at specific "there is a fork here worth exploiting"
points: `PullupBothSide` (`pullup_both_side.cpp:5`, used for `INNER` joins
and cross products) constructs `left_pullup`/`right_pullup` with
`can_pullup=true` explicitly, recurses each side with that permissive
instance, and merges whatever each side hoisted into one combined list;
`PullupFromLeft` (`pullup_from_left.cpp:7`, used for `LEFT`/`SEMI`/`ANTI`
joins, matching the exact same asymmetry pushdown itself respects for those
join types) allows pullup **only** from the left side, refusing to pull
anything the right side exposed — correctly mirroring the fact that only
the left side's rows are unconditionally preserved by those join types.

`PullupInnerJoin` (`filter_pullup.cpp:66`) is where the "turn the join into
a filtered cross product" step happens, and it deliberately mirrors
`PushdownInnerJoin`'s own "convert to cross product" trick from §2.6, just
run in the opposite direction: it collects whatever `PullupBothSide` hoisted
from each child, *adds the join's own conditions to that same list*
(`expressions.push_back(JoinCondition::CreateExpression(...))`,
`filter_pullup.cpp:89`), rebuilds the two children as a plain
`LogicalCrossProduct`, and either (a) if this `FilterPullup` instance's own
`can_pullup` is true, hands the combined expression list further up to
*its* parent as `filters_expr_pullup` (deferring the actual `LogicalFilter`
construction to whoever asked for the pullup), or (b) if not — this
instance is the top of a pullup chain, with nowhere further to hoist to —
wraps the cross product in a genuine `LogicalFilter` right here
(`GeneratePullupFilter`). Either way, the join's own condition ends up
sitting in the *same* expression list as whatever was pulled up from
below it — which is precisely the setup `FILTER_PUSHDOWN`'s subsequent
`FilterCombiner` needs to perform the equivalence-class merge from Part III.

### 4.3 Reading `pullup_filters.test`'s join-order case as end-to-end proof

The file's final section is worth reading as a single, complete story
rather than an isolated `EXPLAIN` check, because it demonstrates the whole
point of sequencing pullup immediately before pushdown — not merely that a
filter gets duplicated onto both sides of a join (§3.2 already covered
that), but that the **derived** filter changes a *later* pass's decision
(`JOIN_ORDER`, Chapter 8):

```sql
-- test/optimizer/pullup_filters.test
create table orders as select range o_orderkey from range(10)
create table lineitem as select range % 10 l_orderkey from range(100)

-- even though we are explicitly joining on l_orderkey both times,
-- we can derive that o1.o_orderkey = o2.o_orderkey
-- once we've derived this, the join order optimizer finds it should join
-- o1 with o2 before joining with lineitem rather than joining lineitem
-- with o1 and o2 directly

explain
select count(*) from lineitem l
join orders o1 on (l.l_orderkey = o1.o_orderkey)
join orders o2 on (l.l_orderkey = o2.o_orderkey)
----
logical_opt   <REGEX>:.*lineitem.*orders.*orders.*
```

Nothing in the SQL text ever states `o1.o_orderkey = o2.o_orderkey`
directly — both conditions are written against `lineitem` — but the
combined pullup-then-pushdown machinery derives it transitively (this is
Part III's equality-merge mechanism again: `l.l_orderkey` is in the same
equivalence class as both `o1.o_orderkey` and `o2.o_orderkey`, and
`GenerateFilters` emits the cross-equality between every pair in a merged
class, not just against constants), and that derived, previously-nonexistent
join condition is what lets `JOIN_ORDER`'s cost-based search discover that
joining the two small `orders` copies to each other *first* — before either
one touches the (ten-times larger) `lineitem` — is cheaper than the
left-deep, SQL-text order the binder would otherwise have handed it. The
test proves this is genuinely `FILTER_PULLUP`'s contribution, not some
side effect of ordinary pushdown alone, by disabling it and showing the
join order regresses to the naive one:

```sql
-- if we disable the FilterPullup, we get the original join order again:
-- orders first (joined last), then lineitem, then orders
set disabled_optimizers to 'filter_pullup'

explain
select count(*) from lineitem l
join orders o1 on (l.l_orderkey = o1.o_orderkey)
join orders o2 on (l.l_orderkey = o2.o_orderkey)
----
logical_opt   <REGEX>:.*orders.*lineitem.*orders.*
```

This one paired test is, in miniature, the entire argument for why this
book treats `FILTER_PULLUP` and `FILTER_PUSHDOWN` as a single combined
mechanism rather than two unrelated passes that merely happen to run
back-to-back: pullup's *only* observable effect, in this example, is on a
completely different pass three steps later in the fixed pipeline
(`JOIN_ORDER`, pass #17) — pullup itself never chooses a join order, and
pushdown alone (with pullup disabled) never discovers the derived equality
in the first place, because the filter that would let it do so is buried on
one child's subtree and never rises to be seen alongside the join
condition.

### 4.4 `PullupProjection`, `PullupSetOperation`, and where pullup declines to act

Two remaining pullup handlers are worth a shorter mention for completeness.
`PullupProjection` (`pullup/pullup_projection.cpp:78`) rewrites a hoisted
filter's column bindings to point at the projection's own output columns —
the mirror image of `ReplaceProjectionBindings` in pushdown (§2.3) — but
with an important extra safety check for operators that **cannot add new
output columns** (`INTERSECT`, `EXCEPT`, `DISTINCT`): if hoisting a filter
through the projection would require exposing a new column the projection
didn't already compute (`copy_proj_expressions.size() > proj.expressions.size()`,
`pullup_projection.cpp:66`), the pullup is reverted right there
(`RevertFilterPullup`) rather than silently changing the set operation's
column count.

> **Fallible predicates stay below.** `PullupProjection` also reverts the
> pullup when any pulled-up predicate `CanThrow()`
> (`pullup/pullup_projection.cpp:83`). The source comment: *"Keep fallible predicates and their guards below the projection."*
> Think of `WHERE value <> 'bad'` below a join on `value::BIGINT + 1`. The
> filter is a *guard*: it removes the rows the cast would fail on. If pullup
> moved the guard and the cast apart, the cast could run on `'bad'` and throw.
> `test/optimizer/pullup_join_expression_semantics.test` builds this shape
> (`cast_rows`, `overflow_rows`). With the guard on, the query must return rows;
> with the guard off, it must still raise `Conversion Error` or
> `Overflow in addition`. It runs once more with `filter_pushdown`, `join_order`
> and `projection_pullup` disabled, so that no later pass can repair a broken
> pullup by accident.

`PullupSetOperation` (`pullup/pullup_set_operation.cpp:19`)
handles `INTERSECT`/`EXCEPT` specifically — `INTERSECT` pulls from both
sides (`PullupBothSide`, matching the fact that either side missing a row
means the intersection lacks it too), `EXCEPT` pulls only from the left
(`PullupFromLeft`, matching that only the left side's rows can ever survive
an `EXCEPT`) — the identical asymmetry pushdown's own set-operation handling
(§2.14) respects, arrived at independently for the opposite traversal
direction.

---

## Part V — the storage-layer payoff: from `LogicalFilter` to a scan's `table_filters`

Everything above describes *how far down the tree* a predicate travels.
This part is about the second, independent question: once a predicate
reaches a `LogicalGet`, does it become a genuine, storage-aware
`TableFilterSet` entry (unlocking `RowGroup::CheckZonemap`,
`src/storage/table/row_group.cpp:732`, and the row-group-skipping behavior
Chapter 17 §17.5 covers in full), or does it merely stop descending and sit
as an ordinary `LogicalFilter` directly above the scan — functionally
correct, but invisible to zonemap pruning? `FilterCombiner::GenerateTableScanFilters`
(`filter_combiner.cpp:1069`) is the dividing line, and a good chunk of the
test corpus exists specifically to draw it precisely.

### 5.1 Prefix rewriting: `LIKE`, `starts_with`, `^@`, `prefix()` all funnel into the same range rewrite

`TryPushdownPrefixFilter` (`filter_combiner.cpp:490`) and
`TryPushdownLikeFilter` (`filter_combiner.cpp:556`) both do the same
essential thing pass #1's `LikeOptimizationRule`/`StringPrefixRule` do at
the expression level (§7.5 of the overview) — recognize that a fixed-prefix
string match is really a **range constraint** — but they do it at the
scan-filter level, producing two `ExpressionFilter`s (a `>=` lower bound and
a `<` upper bound) instead of leaving the pattern-match call opaque to the
storage layer:

```cpp
// src/optimizer/filter_combiner.cpp:518-526
auto lower_bound = CreateComparisonExpression(*func.GetChildren()[0], ExpressionType::COMPARE_GREATERTHANOREQUALTO,
                                              Value(prefix_string));
table_filters.PushFilter(filter_idx, make_uniq<ExpressionFilter>(std::move(lower_bound)));
if (Utf8Proc::FindNextLegalUTF8(prefix_string)) {
	auto upper_bound = CreateComparisonExpression(*func.GetChildren()[0], ExpressionType::COMPARE_LESSTHAN, Value(prefix_string));
	table_filters.PushFilter(filter_idx, make_uniq<ExpressionFilter>(std::move(upper_bound)));
	return FilterPushdownResult::PUSHED_DOWN_FULLY;
}
```

`starts_with_pushdown.test` proves all three syntactic spellings of "prefix
match" — `starts_with(s, ...)`, `s ^@ ...`, and `prefix(s, ...)` — converge
on the identical rewrite, and proves the payoff directly in
`EXPLAIN ANALYZE`'s *runtime* output, not just the logical plan:

```sql
-- test/optimizer/pushdown/starts_with_pushdown.test
EXPLAIN ANALYZE SELECT count(*) FROM t WHERE starts_with(s, '00024999')
----
analyzed_plan   <REGEX>:(?s).*s >= '00024999'.*s < '0002499:'.*Row Groups Scanned: 1 / 5.*
```

`Utf8Proc::FindNextLegalUTF8` is what computes `'0002499:'` as the upper
bound (`:` is the byte immediately after `9` in ASCII — the tightest
possible exclusive upper bound for "everything starting with `00024999`");
`Row Groups Scanned: 1 / 5` is the zonemap payoff made completely concrete —
of five row groups in the table, only the one whose min/max zonemap range
actually straddles this narrow prefix window needed to be opened at all.
`like_prefix_boundary.test` stress-tests the exact UTF-8 boundary
computation this depends on, using deliberately chosen boundary code points
(`chr(127)`, the boundary between one- and two-byte UTF-8 encoding, and
`chr(2047)`, the boundary between two- and three-byte encoding) to make sure
the "next legal UTF-8 string after this prefix" computation doesn't
mis-round at exactly the byte-length transition.

> **A `NULL` pattern is left alone** by both functions.
>
> - `TryPushdownPrefixFilter` returns `NO_PUSHDOWN` when the constant is
>   `NULL` (`filter_combiner.cpp:507`; comment: *"prefix(col, NULL) is NULL and
>   filters out every row - leave the filter in place, do not push it down"*).
> - `TryPushdownLikeFilter` also returns `NO_PUSHDOWN` for `col LIKE NULL`.
>   `col LIKE NULL` is `NULL` for every row, so it must keep no rows at all. A
>   pushed `col IS NOT NULL` filter would keep the non-`NULL` ones. The normal
>   filter removes every row instead (this guards against a bug reported in
>   issue #26433).
>
> The tests: `test/optimizer/pushdown/prefix_pushdown_null_constant.test`
> (`SELECT * FROM t WHERE s ^@ nullif(NULL::VARCHAR, s)` returns no rows) and
> `test/sql/filter/test_like_null.test`
> (`SELECT count(*) FROM t WHERE c1 ILIKE (CASE CAST(NULL AS BLOB) WHEN NULL THEN c1 END)`
> returns `0`).

### 5.2 `OR`-clause pushdown: `TryPushdownOrClause`, and how far it reaches

`TryPushdownOrClause` (`filter_combiner.cpp:705`) only fires when *every*
branch of a top-level `OR` is a plain comparison between the *same* column
and a constant (`table_or_pushdown.test`'s enormous sweep is essentially an
exhaustive proof of this boundary) — the moment any branch involves a
different column, a function call, or a nested `CASE`, the whole `OR` is
declined (`return FilterPushdownResult::NO_PUSHDOWN`) and stays as an
ordinary expression filter instead:

```sql
-- test/optimizer/pushdown/table_or_pushdown.test
EXPLAIN SELECT * FROM integers WHERE a=1 OR a>3 OR a<5
----
physical_plan   <REGEX>:.*SEQ_SCAN.*Filters: a=1 OR a>3 OR a<5|.*     -- pure single-column OR: pushed

EXPLAIN SELECT * FROM integers WHERE a=1 OR b=2 AND (a>3 OR b<5)
----
physical_plan   <!REGEX>:.*SEQ_SCAN.*Filters.*    -- mixes columns a and b: not pushed at all

EXPLAIN SELECT * FROM integers WHERE concat(a=1, b=1)='truetrue' OR a=2 ORDER by a
----
physical_plan   <!REGEX>:.*SEQ_SCAN.*Filters.*    -- a function call inside a branch: not pushed
```

The file's most interesting single case is the "AND priority" test — an
`AND` conjunction where only *one* conjunct is pushable and pushdown must
still leave the whole `OR` sub-expression correctly attached to the scan
without accidentally splitting it apart:

```sql
EXPLAIN SELECT * FROM integers WHERE (a<2 OR a>3) AND (a=1 OR a=4)
----
physical_plan   <REGEX>:.*SEQ_SCAN.*Filters.*a.*<.*2.*OR.*a.*>.*3.*AND.*a.*=.*1.*OR.*a.*=.*4.*
```

both `OR` clauses independently qualify (each is single-column, pure
comparisons against constants), and both land as scan-level filters
connected by `AND` — proving the mechanism composes across multiple `OR`
groups joined by `AND`, not just a single top-level `OR`.

> **Dense equality `OR`s are pushed exactly.** In general an `OR` is pushed as
> an *optional* filter (only used for zonemap pruning), and the original `OR`
> stays as a `FILTER` above the scan. But if every branch is `col = constant` and the constants form a dense range
> (`IsDenseRange`, for example `-1, 0, 1, 2`), the `OR` becomes two exact
> filters, `col >= -1` and `col <= 2`, and the `FILTER` goes away. The same
> helper, `TryPushdownDenseInFilter` (`filter_combiner.cpp:630`), also does
> this for a dense `IN` list. `test/sql/optimizer/plan/dense_or_filter_pushdown.test`:
>
> ```sql
> EXPLAIN (FORMAT JSON) SELECT x FROM dense_or WHERE x = 2 OR -1 = x OR x = 1 OR x = 0
> -- physical_plan   <!REGEX>:.*FILTER.*     (exact range, no residual filter)
> EXPLAIN (FORMAT JSON) SELECT x FROM dense_or WHERE x = 1 OR x = 3
> -- physical_plan   <REGEX>:.*FILTER.*      (a gap: the exact OR must stay)
> ```
>
> `table_or_pushdown.test`'s "numeric statistics" cases show the usual shape
> for a non-dense `OR`: an `optional:` filter on the scan, used only to
> prune row groups, plus a real `FILTER` above it. For example
> `EXPLAIN ANALYZE SELECT a FROM integers WHERE a<2 OR a>4` must match
> `FILTER.*a < 2.*OR.*a > 4.*2 rows.*TABLE_SCAN.*Filters.*optional:`.

### 5.3 Type-relaxed temporal casts and `constant_or_null`

Two smaller mechanisms round out the scan-filter story. `TryPushdownTemporalCastFilter`
(`filter_combiner.cpp:948`) handles a filter written against a *cast* of a
temporal column (`WHERE ts >= NOW()::TIMESTAMP - INTERVAL 1 YEAR` in
`table_filter_pushdown.test`, where `ts` is stored as `TIMESTAMP` and
compared after some arithmetic) by computing a **relaxation margin** per
source/target type pair (`GetTemporalCastMargin`, `filter_combiner.cpp:803`
— 2 hours for `TIMESTAMP`↔`TIMESTAMPTZ` DST ambiguity, 24 hours for a cast
that drops time-of-day precision entirely, 1 second for sub-second rounding
loss) and widening the pushed bound by that margin, so the scan-level filter
is a conservative *superset* of the true predicate (never wrongly excludes a
qualifying row, at the cost of occasionally admitting a few extra
candidates a downstream, exact re-check then discards). (A "cast" between
two types with the same type id gets margin 0 only for real temporal types. For other types with the same id, such as two
different `DECIMAL` or `UNION` types, the function returns `-1`, meaning "no
safe margin".) `constant_or_null`
(`test_constant_or_null_pushdown.test`) is the inverse of a normal
constant-folding scenario — it's a function whose entire purpose is *NULL
propagation bookkeeping* (used by subquery decorrelation, Chapter 6, to
preserve "no correlated match" semantics), and the test confirms it still
survives, visibly, as its own scan-level filter (`Filters: ... constant_or_null ...`)
rather than being silently discarded or folded away, since its NULL-ness is
exactly the information later stages of the plan depend on.

### 5.4 Generic expressions: `TryPushdownGenericExpression` and the `scalar_fn_pushdown`/`type_pushdown` frontier

Beyond the specialized recognizers above (prefix, `LIKE`, `OR`, temporal
casts), `TryPushdownGenericExpression` (`filter_combiner.cpp:447`) is the
catch-all: if the scan's `TableFunction` declares `pushdown_expression`, and
the candidate filter references exactly one column (or, for a Parquet/DuckDB
table specifically, several columns of the *same* physical row — handled via
`TryCreateMultiColumnExpressionFilter` and a `PUSHED_DOWN_PARTIALLY` result,
which still lets statistics propagate even though it can't fully replace
the filter), the raw expression itself — not a decomposed comparison — is
handed to the reader to evaluate natively. This is a structurally different
mechanism from `SCALAR_FN_PUSHDOWN` (pass #45) and `TYPE_PUSHDOWN` (pass
#44, both covered only briefly in the overview's §7.4 listing), which push a
**projected output expression** (a `SELECT`-list function call or cast) into
the reader rather than a filter — but `scalar_function_pushdown.test` is
useful evidence here too, because it demonstrates the single-usage
constraint both mechanisms share: any *second*, differently-shaped use of
the same source column blocks the pushdown outright.

```sql
-- test/optimizer/pushdown/scalar_function_pushdown.test
EXPLAIN SELECT strlen(str) FROM '{TEST_DIR}/pe-pushdown.parquet';
----
physical_plan   <!REGEX>:.*PROJECTION.*      -- strlen computed inside the Parquet reader itself

EXPLAIN SELECT strlen(str) FROM '{TEST_DIR}/pe-pushdown.parquet' WHERE strlen(str) > 2;
----
physical_plan   <REGEX>:.*PROJECTION.*       -- a filter on strlen(str) blocks pushdown of the SELECT-list strlen(str)

EXPLAIN SELECT strlen(str) FROM '{TEST_DIR}/pe-pushdown.parquet' WHERE len(str) > 2;
----
physical_plan   <REGEX>:.*PROJECTION.*       -- even a *different* function (len, not strlen) of the same column conflicts
```

The reasoning is the same "single, unambiguous rewrite target" constraint
`InClauseSimplificationRule` and `TryPushdownGenericExpression`'s
multi-column carve-out both respect in their own ways: once `str` is
consumed by two structurally different expressions (`strlen(str)` in
`SELECT` and `len(str) > 2` in `WHERE`), there's no longer one single
rewrite of "replace this column with this reader-computed value" that
satisfies both usages simultaneously, so the optimizer declines to push
either and falls back to an ordinary post-scan `PROJECTION`/`FILTER`. The
`scalar_function_pushdown_limit.test` file adds one further check: the
pushdown still fires even when the query also has a `LIMIT` sitting above
it — proving `SCALAR_FN_PUSHDOWN` (pass #45) and `LIMIT_PUSHDOWN`
(pass #29) don't interfere with each other despite operating on adjacent
parts of the same tree.

### 5.5 `csv_type_pushdown.test` and `csv_pushdown.test`: reader-level pushdown has a narrower ceiling than Parquet's

Contrasting the CSV reader against the scan-filter mechanism above is
instructive precisely because it shows a *lower* ceiling for what a given
source format can accept, without any change to the pass itself — the
optimizer offers the same rewrite; the CSV reader's `TableFunction` simply
doesn't implement every hook a columnar format like Parquet can:

```sql
-- test/optimizer/pushdown/csv_type_pushdown.test
-- Column is used casted and uncasted: conflict, no pushdown (same rule as scalar_fn above)
EXPLAIN SELECT column00, column00::UTINYINT FROM read_csv(...);
----
physical_plan   <REGEX>:.*PROJECTION.*

-- Column is used uncasted with filter on casted value.
-- CSV reader doesn't support CAST expression pushdown.
EXPLAIN SELECT column00 FROM read_csv(...)
WHERE column00::UTINYINT > 0;
----
physical_plan   <REGEX>:.*FILTER.*     -- stops at an ordinary LogicalFilter above the scan, not a scan-level table filter

-- CAST and TRY_CAST conflict in NULL-semantics: not pushed down even though
-- both nominally target the same column, because CAST throws on overflow
-- and TRY_CAST silently returns NULL — the two are not interchangeable
EXPLAIN SELECT column00::UTINYINT, TRY_CAST(column00 AS UTINYINT)
FROM read_csv(...);
----
physical_plan   <REGEX>:.*CAST.*
```

and `csv_pushdown.test` demonstrates the CSV reader's pushdown ceiling from
the *other* direction — it happily supports column **projection** pushdown
(reading only the one column actually referenced, skipping type conversion
entirely for every column that's never touched: *"conversion is skipped if
we don't read the value — so even with the incorrect type specified this
still works"* is the file's own comment, proving a column whose declared
type would actually fail to parse can still be silently ignored as long as
nothing downstream reads it) — but nowhere in this file does a `WHERE`
clause reach the CSV reader as a genuine table filter the way it reaches a
Parquet or native DuckDB scan; the CSV reader's contract with the optimizer
is narrower by design, and the CAST-pushdown test above shows the exact
line: `TYPE_PUSHDOWN` (pass #44) can push a plain, unconditional column cast
into the reader, but the moment that same cast also needs to gate a *filter*
(rather than just transform an output value), the CSV reader's
`filter_pushdown`/`pushdown_complex_filter` hooks simply don't cover that
combination, and the optimizer correctly falls back to an ordinary
`LogicalFilter` rather than guessing.

### 5.6 `distinct_from_pushdown.test` — `IS [NOT] DISTINCT FROM` inside an `OR` still reaches the scan

A short but precise confirmation that NULL-safe comparison operators are
covered by the same `OR`-clause machinery as ordinary comparisons — recall
`TryPushdownOrClause`'s explicit handling of `COMPARE_DISTINCT_FROM`/
`COMPARE_NOT_DISTINCT_FROM` against a `NULL` constant
(`filter_combiner.cpp:760-778`, converting them to `IS [NOT] NULL` table
filters rather than a literal, always-false `x = NULL` comparison):

```sql
-- test/optimizer/pushdown/distinct_from_pushdown.test
select * from test where tst is not distinct from 'a' or tst is not distinct from 'b';
----
                                    -- empty: neither branch matches 'tst'

select * from test where tst is distinct from 'a' or tst is distinct from 'b';
----
tst                                 -- correctly returns the row: it's distinct from at least one of the two constants
```

### 5.7 `remove_unused_filter_column.test` — the column-pruning payoff pushdown sets up for a later pass

`remove_unused_filter_column.test` isn't, strictly, a `FILTER_PUSHDOWN` test
at all — its regex checks the **absence** of a column from a scan's
physical output list, which is `UNUSED_COLUMNS`/`COLUMN_LIFETIME`'s job
(passes #23/#26, §7.4 of the overview) — but it's worth including here
because it's the clean demonstration of the second-order benefit pushdown
sets up:

```sql
-- test/optimizer/remove_unused_filter_column.test
explain select i from test where j=0
----
physical_plan   <!REGEX>:│ +j +│     -- column j does not appear anywhere in the physical SEQ_SCAN's own output
```

Once `j=0` has become a scan-level table filter (exactly the §5.1-§5.4
mechanism), the query's actual `SELECT` list (just `i`) never needs `j` to
survive as an *output* column of the scan at all — the filter is evaluated
entirely inside the scan's own table-filter machinery, against bytes that
never need to be materialized into the projected result. This is exactly
why filter pushdown and column pruning are usually discussed together even
though they're formally separate passes: a filter that reaches a scan as a
table filter doesn't just get to skip whole row groups via the zonemap
(§17.5) — the column it filters on doesn't even need to be *decoded* for
rows inside the row groups that do get scanned, past whatever the filter
evaluator itself needs to read to compute the comparison, and once the
filter's already been applied at that layer, `UNUSED_COLUMNS` is free to
drop it from the scan's declared output entirely if nothing else downstream
reads it.

---

## Part VI — volatility, side effects, and the negative test corpus

### 6.1 `filter_cannot_pushdown.test` — the volatility boundary, drawn precisely

This file's name promises a negative result, and its actual content is
entirely about **volatility**, not about join semantics or row-order
concerns — every case is built around `random()` or `nextval()`, and the
test file is a careful demonstration of exactly *how far* a volatile
expression's non-pushability propagates through neighboring, non-volatile
predicates in the same conjunction:

```sql
-- test/optimizer/pushdown/filter_cannot_pushdown.test
-- t is empty, so 'a > 1' is always false regardless of any volatile predicate
-- alongside it - the whole thing collapses to EMPTY_RESULT
explain select rnd, a from (select random() as rnd, a from t) where rnd < 0.3 and a > 1;
----
logical_opt   <REGEX>:.*EMPTY_RESULT.*

-- once t has a row, a > 1 can be independently proven/refuted and the
-- volatile rnd < 0.3 conjunct is left completely alone
explain select rnd, a from (select random() as rnd, a from t) where rnd < 0.3 and a > 1;
----
logical_opt   <!REGEX>:.*a > 1.*      -- 'a > 1' itself no longer shows up as a *separate* filter:
                                      -- it was pushed all the way to the scan, while rnd < 0.3 was not
```

The last case in the file is the sharpest: a `HAVING` clause on a
`nextval()`-derived `GROUP BY` key (`group by 1 having g > 1`) is shown to
still filter correctly — `g > 1` legitimately stays *above* the aggregate
(§2.12's aggregate handler already refuses to push a filter on the
aggregate's own bindings), and the sequence's side effect (advancing the
counter) is confirmed, via an actual materialized table and a follow-up
`SELECT`, to have fired exactly once per underlying row rather than once
per filter evaluation — proof that the volatility guard isn't merely
blocking a plan-shape regex from matching, but is actually preventing a
real, observable double-increment bug.

### 6.2 `lateral_projection_evaluation.test` — counting side-effect calls through a `LATERAL` join and a recursive CTE

This file pushes the "how many times did a side-effecting expression
actually run" question about as far as it can reasonably go: a `LATERAL`
join against a `WITH RECURSIVE` walk (each of 100 vertices generates its own
8-row recursive walk), with a `nextval('projected_calls')` counter inside
the outer projection, filtered down to a single vertex (`WHERE id = 3`)
*after* the projection:

```sql
-- test/optimizer/pushdown/lateral_projection_evaluation.test
SELECT sum(v), count(*) FROM (SELECT id, nextval('projected_calls') AS v FROM walk_rows) q WHERE id = 3;
----
36	8

SELECT currval('projected_calls');
----
8
```

The filter `id = 3` correctly narrows the *input* rows the projection ever
runs against — down to exactly the 8 rows the recursive walk produces for
vertex 3 — before `nextval` is evaluated for any of them, which the final
`currval('projected_calls')` check confirms directly: the sequence advanced
by exactly 8, not by `100 * 8 = 800` (which is what would have happened had
the filter been evaluated only *after* the volatile projection ran across
every vertex's walk). This is the same principle as §6.1's guard, but
exercised through a genuinely complex plan shape (`LATERAL` plus recursive
CTE plus sequence) rather than a synthetic minimal repro — it's the pass's
own correctness invariant (a barred/volatile expression is never evaluated
on more rows than the final result actually needed) surviving contact with
one of the more structurally involved constructs in the whole SQL surface.

---

## Summary: the boundary this pass draws, restated precisely

Pulling every operator handler in Part II together into one statement: a
predicate may be pushed past an operator boundary exactly when doing so is
provably a no-op with respect to *which rows the final result contains and
what values it computes for them* — never merely "when it looks likely to
help." The concrete boundary each operator draws:

- **Safe unconditionally, no rewriting needed:** `ORDER BY`, plain
  `DISTINCT` (§2.13) — neither changes row identity or count in a way a
  filter's position could affect. (A *volatile* filter does not cross a plain
  `DISTINCT`, because it would run once per duplicate row.)
- **Safe, but requires column-reference rewriting:** `PROJECTION` (§2.3),
  `UNNEST` for non-unnested columns (§2.15), set operations via shared
  bindings (§2.14) — the filter is semantically pushable, but the *symbols*
  it's written against must be translated to whatever the child operator
  actually computes.
- **Safe only for one side, asymmetrically:** `LEFT`/`SINGLE`/`ASOF` joins
  (§2.7, §2.11), `SEMI`/`ANTI` joins (§2.9) — the preserved side accepts
  anything; the other side accepts nothing, *unless* a specific proof (null
  rejection, §2.7) licenses converting the operator to a symmetric one
  first.
- **Safe only under a narrow, specifically provable shape:** `FULL OUTER
  JOIN`'s coalesced-key rewrite (§2.8), window functions restricted to
  every expression's shared partition keys (§2.16), aggregates restricted to
  columns present in every grouping set (§2.12), mark-to-semi/anti
  conversion gated on the marker never being consumed as a value elsewhere
  (§2.10).
- **Unsafe by construction, with at most one degenerate exception:**
  `LIMIT` (§2.17, `LIMIT 0` only), `DISTINCT ON` (§2.13, not implemented at
  all as of this checkout).
- **Blocked regardless of operator shape:** any filter with a volatile
  subexpression (`random()`, `nextval()` — §6.1, §6.2, and the volatility
  guards repeated independently in nearly every handler in Part II), and any
  filter wrapped in an `ExpressionBarrier` until it reaches an operator
  proven not to remove rows (§1.3).

And layered on top of every one of those rules is the equivalence-class
machinery from Part III — not a rule about *where* a filter may go, but
about *what new filters can be manufactured* from the ones already in hand,
fed by `FILTER_PULLUP`'s job (Part IV) of first making sure a buried filter
and a join condition ever end up in the same `FilterCombiner`'s view at all.
Together, the two passes are less "push things down" and "pull things up"
as separate ideas than a single two-phase procedure for maximizing how much
of a query's `WHERE` clause — written, derived, or transitively implied — a
`LogicalGet` ends up seeing directly, which is the one thing that actually
lets the storage engine's zonemaps (Chapter 17 §17.5) do their job.
