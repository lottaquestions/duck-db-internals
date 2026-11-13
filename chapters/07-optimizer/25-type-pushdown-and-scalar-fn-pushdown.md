# Chapter 7 Deep Dive — `TYPE_PUSHDOWN` and `SCALAR_FN_PUSHDOWN`

§7.4 of the chapter overview (`00-overview.md`) gave these two passes one
paragraph each, at positions #44 and #45 — the second-to-last and the very
last entries in the ~45-call-site `RunBuiltInOptimizers` sequence, run
immediately before the pre-/post-optimize extension hooks close the pipeline
out. That placement is not an accident of alphabetization; it is the single
most important fact about both passes, and this file spends real time
justifying it from source rather than asserting it.

Both passes answer a version of the same question — *"is there an
expression sitting directly above a table scan that the scan's own reader
could just compute itself, instead of the expression executor computing it
in a separate pass over already-materialized column vectors?"* — but they
ask it about two structurally different kinds of expression. `TYPE_PUSHDOWN`
(`src/optimizer/type_pushdown.cpp`) asks it about `CAST`/`TRY_CAST`.
`SCALAR_FN_PUSHDOWN` (`src/optimizer/scalar_fn_pushdown.cpp`) asks it about
ordinary single-argument scalar function calls. This is deliberately *not*
the same question `FILTER_PUSHDOWN` (pass #6, the sibling deep-dive file
`07-filter-pullup-and-pushdown.md`) asks, even though all three passes share
the word "pushdown" and all three end up rewriting a `LogicalGet`. Filter
pushdown pushes *predicates* — expressions that decide which rows survive.
These two passes push *value-computing* expressions — casts and function
calls — that never remove a single row; they only change what value a
surviving row's column produces, or, in the scalar-function case, let the
reader avoid fully materializing a value at all. That distinction turns out
to explain almost everything else about how these two passes are built,
where they sit in the pipeline, and why they could not have been folded into
`FILTER_PUSHDOWN`'s existing machinery. §3 of this file works through the
"why separate, why late" argument in full; §1 and §2 are the source and test
walkthroughs themselves.

One administrative note shared with the sibling pushdown file: none of the
four main `.test` files this chapter walks through (nor the newer ones it adds) use the "same-label idiom" (two
`query ... nosort <label>` blocks with no expected output, asserting
byte-identical `EXPLAIN` output between two written-out plan shapes) that
`07-filter-pullup-and-pushdown.md` explains in depth. Every test cited below
instead uses a direct `query II` `EXPLAIN` with a `<REGEX>`/`<!REGEX>` match
against a specific operator name (typically `PROJECTION` or `FILTER`) — so
"pushdown succeeded" is read off these tests as *"no `PROJECTION` operator
survives between the final output and the scan"* rather than as a two-plan
equality assertion.

---

## Part I — `TYPE_PUSHDOWN`

**One-sentence purpose**, in the pass's own words, from the call site in
`optimizer.cpp`:

```cpp
// src/optimizer/optimizer.cpp:554-558
// Push down type casts in SELECT e.g. SELECT num::UHUGEINT to file readers.
RunOptimizer(OptimizerType::TYPE_PUSHDOWN, [&] {
	TypePushdown type_pushdown(context);
	plan = type_pushdown.Optimize(std::move(plan));
});
```

and the file's own header comment, which is worth quoting in full because it
states the mechanism more precisely than the one-liner does:

```cpp
// src/optimizer/type_pushdown.cpp:20
/*
 * This pass implements type pushdown for file readers. If a LOGICAL_PROJECTION
 * has a LOGICAL_GET as a child, and for some column that's cast in
 * LOGICAL_PROJECTION there is no other usage (including uncasted) in the query,
 * and file reader supports projection expression pushdown, we can push down the
 * type cast into LOGICAL_GET.
 *
 * Example: SELECT ts::TIMESTAMP FROM file_reader();
 * We can push TIMESTAMP as ts's output type to file_reader();
 */
```

So: **not** "make the physical type narrower for memory/bandwidth reasons"
in the general sense the task brief raised as one hypothesis — the actual
mechanism is narrower and more specific than that. `TYPE_PUSHDOWN` looks for
a `CAST`/`TRY_CAST` of a column that sits directly above a scan (possibly
through one or more pure passthrough projections), checks that *every* other
place that column is referenced anywhere in the query agrees on the same
cast (same target type, same `CAST`-vs-`TRY_CAST` semantics), and — if the
reader agrees to cooperate — rewrites the scan itself to declare that
column's output type as the cast's *target* type rather than its own native
type, deleting the `CAST` expression entirely and leaving a bare column
reference of the new type in its place. The "narrower type" framing and the
"push the cast expression down" framing turn out to be the same thing here,
because of how the rewrite is implemented: there's no separate "narrow the
physical storage representation" step independent of the cast — the pass
*is* cast-expression pushdown, and a narrower/wider on-the-wire type is
simply what falls out when the reader honors the pushed cast.

### 1.1 The shared skeleton: `Analyses`, `Projections`, `GetAnalysis`, and `PushdownOptimize<>`

Both `TYPE_PUSHDOWN` and `SCALAR_FN_PUSHDOWN` are built on top of one shared,
generic piece of machinery that lives entirely inside `type_pushdown.cpp`/
`type_pushdown.hpp`. `scalar_fn_pushdown.hpp` (54 lines) starts with
`#include "duckdb/optimizer/type_pushdown.hpp"`, and after that it only
declares `ScalarFnCollect`, `ScalarFnReplace` and a three-line
`ScalarFnPushdown` wrapper that calls the shared template. This is a
genuinely unusual architectural choice worth flagging up front, in the same
spirit as this chapter's earlier resolution of the `sum_rewriter` naming
question (§7.6.5 of the overview): two different, separately-timed,
separately-toggleable `OptimizerType`s (`TYPE_PUSHDOWN` and
`SCALAR_FN_PUSHDOWN`) share one concrete implementation of the tree-walking,
conflict-tracking, and reader-negotiation logic, differing only in which
*shape* of expression each one's `Collect`/`Replace` visitor pair matches
against. `type_pushdown.cpp` is the genuine home of the shared engine;
`scalar_fn_pushdown.cpp` is a second, much smaller consumer of it.

The shared data structures, all declared in `type_pushdown.hpp`:

```cpp
// src/include/duckdb/optimizer/type_pushdown.hpp:21
struct GetAnalysis {
	LogicalGet &get;
	/**
	 * for CAST(col), mapping of "col scan index" -> "expression".
	 * "expression" is nullptr iff column is used with a different expression
	 * or without expression in the query plan.
	 */
	unordered_map<ProjectionIndex, const Expression *> col_to_expr;

	idx_t StorageIndex(ProjectionIndex idx) const;
};

using Analyses = unordered_map<TableIndex, GetAnalysis>;
```

`ProjectionIndex` (`src/include/duckdb/common/projection_index.hpp:16`) is
its own small wrapper type, and its doc comment states precisely what it
indexes: *"an index within the projection list of a node in the planner"* —
i.e. the position of a column within whatever this particular `LogicalGet`
currently outputs, **after** column pruning (`UNUSED_COLUMNS`/
`COLUMN_LIFETIME`, passes #23/#26, have already run by the time pass #44
executes), not the column's raw ordinal position in the underlying file.
`GetAnalysis::StorageIndex` is the function that converts one into the
other:

```cpp
// src/optimizer/type_pushdown.cpp:303
idx_t GetAnalysis::StorageIndex(ProjectionIndex idx) const {
	return get.GetColumnIds()[idx].GetPrimaryIndex();
}
```

`col_to_expr` is where the "does every usage of this column agree" tracking
actually lives: keyed by the column's `ProjectionIndex` within its `Get`,
each entry is either a pointer to the one consistent expression (`CAST(col)`
in this pass, `fn(col)` in the scalar-function pass — §2 below) that every
reference to this column is wrapped in, or `nullptr`, meaning **a conflict
was found** — some reference used the column differently (bare, with a
different cast target, with a different function), which permanently
disqualifies that column from pushdown for the rest of this pass's run.

`Projections` handles the one specific case where the `LogicalGet` isn't the
direct child of the projection carrying the cast — a `VIEW` or a subquery
whose own `SELECT` list is a plain, pure passthrough of the scan's columns:

```cpp
// src/optimizer/type_pushdown.cpp:35
static bool IsPassthrough(const LogicalProjection &projection) {
	if (projection.expressions.empty()) {
		return false; // don't register empty projections in Projections
	}
	for (const auto &e : projection.expressions) {
		if (e->GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
			return false;
		}
	}
	return true;
}
```

`FindGetsAndProjections` (`type_pushdown.cpp:47`) is a single top-down walk
of the *entire* plan that registers two kinds of thing as it descends: every
`LogicalGet` whose table function actually declares a
`projection_expression_pushdown` callback (`get.function.
projection_expression_pushdown != nullptr` — a `Get` without this callback
simply never enters `analyses` at all, and is invisible to the rest of the
pass), and every passthrough `LogicalProjection` that sits, with at most one
`LogicalLimit` in between, directly over such a `Get`:

```cpp
// src/optimizer/type_pushdown.cpp:63
LogicalGet *get = nullptr;

// queries with LIMIT may include a STREAMING_LIMIT between PROJECTION and GET.
// See test/optimizer/pushdown/scalar_function_pushdown_limit.test
if (child.type == LogicalOperatorType::LOGICAL_LIMIT) {
	if (auto &limit = child.Cast<LogicalLimit>();
	    limit.children.size() == 1 && limit.children[0]->type == LogicalOperatorType::LOGICAL_GET) {
		get = &limit.children[0]->Cast<LogicalGet>();
	}
} else if (child.type == LogicalOperatorType::LOGICAL_GET) {
	get = &child.Cast<LogicalGet>();
}
```

This `LOGICAL_LIMIT` carve-out is one of this file's cleanest pieces of
evidence for why these two passes have to run as late as they do — see §3.2.

`Resolve` (`type_pushdown.cpp:98`) is the function every visitor calls to
turn a `ColumnBinding` (a `(table_index, column_index)` pair as it appears at
some arbitrary point in the tree) back into "which `Get`, and which
`ProjectionIndex` within that `Get`, does this actually refer to" — walking
through at most one level of a registered passthrough projection:

```cpp
// src/optimizer/type_pushdown.cpp:90-124
// A column binding into a GET is an index into its column ids, not a column id
// itself. Virtual columns (e.g. "filename") have no entry in returned_types, so
// nothing can be pushed into them.
static bool IsRealGetColumn(const LogicalGet &get, ProjectionIndex column_index) {
	const auto &column_ids = get.GetColumnIds();
	return column_index < column_ids.size() && !column_ids[column_index].IsVirtualColumn();
}

optional<GetBinding> Resolve(ColumnBinding binding, Analyses &analyses, const Projections &projections) {
	if (const auto it = analyses.find(binding.table_index); it != analyses.end()) {
		if (!IsRealGetColumn(it->second.get, binding.column_index)) {
			return nullopt;
		}
		return {{it->second, binding.column_index, nullptr, binding.column_index}};
	}

	const auto projection_it = projections.find(binding.table_index);
	if (projection_it == projections.end()) {
		return nullopt;
	}

	LogicalProjection &projection = projection_it->second;
	const auto &inner = projection.expressions[binding.column_index];
	if (inner->GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
		return nullopt;
	}
	const ColumnBinding get_binding = inner->Cast<BoundColumnRefExpression>().Binding();
	if (const auto it = analyses.find(get_binding.table_index); it != analyses.end()) {
		if (!IsRealGetColumn(it->second.get, get_binding.column_index)) {
			return nullopt;
		}
		return {{it->second, get_binding.column_index, &projection, binding.column_index}};
	}
	return nullopt;
}
```

Two things worth noting here precisely. First, a reference to a virtual
column (`filename`, `file_row_number`, `rowid`, and similar synthetic
columns a scan can expose) is never resolved to any `GetAnalysis` at all.
So it can neither be the target of a pushdown nor register a conflict
against one. This is why the "virtual columns don't block pushdown" case
in `scalar_function_pushdown.test` (§2.5) works.

The virtual-column check needs care. `binding.column_index` is a
*position in the `Get`'s column list* (a `ProjectionIndex`), not a column
id. Virtual column ids are very large special numbers, and a position like
`1` never is one. So calling `IsVirtualColumn(binding.column_index)`
directly would never fire. A query like `SELECT CAST(col1 AS BIGINT),
CAST(filename AS BIGINT) FROM read_csv(...)` could then make the pass treat
`filename` as a pushdown candidate, and write into `returned_types` at the
virtual column's special id, far past the end of that list.

So the helper `IsRealGetColumn` first turns the position into the real
column id with `get.GetColumnIds()[column_index]`, and only then asks
`IsVirtualColumn()`. It runs in both branches of `Resolve`: the direct
`Get` branch and the branch through a passthrough projection. Because
`Resolve` never returns a virtual column, `PushdownOptimize` does not skip
virtual columns itself. It asserts that none arrive instead (§1.1,
below).

The regression test is
`test/optimizer/pushdown/type_pushdown_virtual_column.test`. It writes a
one-row CSV with the value `42`, then checks three things:

```sql
-- a cast on a virtual column is not a pushdown candidate, even when a sibling
-- cast on a real column is pushed down
statement error
SELECT CAST(col1 AS BIGINT), CAST(filename AS BIGINT)
FROM read_csv('{TEST_DIR}/virtual_column_pushdown.csv', columns={'col1': 'INTEGER'});
----
<REGEX>:Conversion Error.*virtual_column_pushdown.csv.*

query II
SELECT CAST(col1 AS BIGINT), CAST(filename AS VARCHAR) LIKE '%virtual_column_pushdown.csv'
FROM read_csv('{TEST_DIR}/virtual_column_pushdown.csv', columns={'col1': 'INTEGER'});
----
42	true

-- the cast on the real column is still pushed into the reader
EXPLAIN SELECT CAST(col1 AS BIGINT) FROM read_csv(...);
----
physical_plan	<!REGEX>:.*PROJECTION.*
```

The error is the right answer: a file name cannot be cast to `BIGINT`,
and that cast must run as an ordinary expression that fails in the
normal way. The file repeats the first two queries through a view `v`, to
cover the passthrough-projection branch of `Resolve`.

Second, the `projections` branch only descends through **one**
level of indirection — `projection.expressions[binding.column_index]` must
itself be a bare `BOUND_COLUMN_REF`, and if that inner reference is, in
turn, bound to *another* registered passthrough projection rather than
directly to a `Get`, `Resolve` does not recurse a second time; it simply
returns `nullopt`. This one-level restriction is explicit and deliberate —
the header comment states it outright (`type_pushdown.hpp:35-50`): *"For
simplicity, current implementation is limited to one level i.e. PROJECTION
-> GET ... is pushed down but VIEW->VIEW->GET or VIEW->CTE->GET is not."*
`csv_type_pushdown.test`'s `pv1`/`pv2` pair (§1.4) is the direct evidence for
this exact limitation.

Finally, `PushdownOptimize<Collect, Replace>` (`type_pushdown.hpp:107`) is
the template both passes instantiate — `TypePushdown` with `<CastCollect,
CastReplace>`, `ScalarFnPushdown` with `<ScalarFnCollect, ScalarFnReplace>`
— and it is the entire algorithm, in four steps:

```cpp
// src/include/duckdb/optimizer/type_pushdown.hpp:107-143
template <class Collect, class Replace>
unique_ptr<LogicalOperator> PushdownOptimize(ClientContext &context, unique_ptr<LogicalOperator> op) {
	Analyses analyses;
	Projections projections;
	FindGetsAndProjections(*op, analyses, projections);
	if (analyses.empty()) {
		return op;
	}
	Collect(analyses, projections).VisitOperator(*op);

	bool any_pushed = false;
	for (auto &[_, analysis] : analyses) {
		for (auto &[column_index, expr] : analysis.col_to_expr) {
			if (expr == nullptr) { // Conflict for column
				continue;
			}
			// Resolve() never yields a virtual column as a candidate
			D_ASSERT(!analysis.get.GetColumnIds()[column_index].IsVirtualColumn());
			TableFunctionProjectionExpressionInput input {analysis.get, *expr, column_index};
			if (analysis.get.function.projection_expression_pushdown(context, input)) {
				analysis.get.has_pushed_projection = true;
				analysis.get.returned_types[analysis.StorageIndex(column_index)] = expr->GetReturnType();
				if (!analysis.get.types.empty()) {
					analysis.get.ResolveOperatorTypes();
				}
				any_pushed = true;
			} else { // failed to push down expression, can't replace it
				expr = nullptr;
			}
		}
	}

	if (any_pushed) {
		Replace(analyses, projections).VisitOperator(*op);
	}
	return op;
}
```

The four steps, in order: (1) `Collect` walks the *whole* tree and finds
matches and conflicts; (2) for each candidate with no conflict, the reader
is asked, and if it accepts, the scan's own output type for that column is
changed; (3) if the reader declines, the entry is set to `nullptr` so the
expression is left alone; (4) if anything was pushed, `Replace` rewrites
every accepted `CAST(col)`/`fn(col)` to a bare `col`.

Two lines of this template need a note.

- Because `Resolve` filters out virtual columns (see above), the loop does
  not skip them; it only asserts that none arrive, with `D_ASSERT` (a check
  that runs in debug builds only).
- When a reader accepts, the scan is marked with
  `analysis.get.has_pushed_projection = true`. The field is declared in
  `logical_get.hpp:72-73` as *"The scan consumed a projection whose source
  expression is no longer retained."* It is serialized with the plan. Its
  reader is the `EXPLAIN (SQL)` exporter, which turns an optimized
  plan back into SQL. A scan that absorbed a cast or `strlen` can no longer
  be written back as a plain `SELECT col FROM ...`, because the original
  expression is gone. So `LogicalGet::ExportSQLSource`
  (`src/planner/sql_export/sql_export_sources.cpp:36-39`) reports it as
  an unsupported source (`"pushed_projection"`) instead of writing wrong
  SQL.

Step (2) is the crux of "what does pushdown actually *do*": the reader
callback is not merely *informed* that a cast/function exists — it is handed
the whole expression (`TableFunctionProjectionExpressionInput`, carrying the
`Get`, the expression itself, and the column's `ProjectionIndex`) and is
free to reach into its own bind data and change what type it will actually
produce for that column going forward. If it accepts, the pass updates
`analysis.get.returned_types` to match — the `LogicalGet`'s own declared
output type for that column changes to the cast's/function's return type —
and step (4)'s `Replace` visitor then walks the tree once more, finding
every occurrence of the now-redundant `CAST(col)`/`fn(col)` and collapsing
it to a bare, correctly-retyped column reference.

### 1.2 `CastCollect` — building the conflict map

`CastCollect` (`type_pushdown.cpp:153`) is a `LogicalOperatorVisitor`
subclass, and its `VisitOperator` override has one specific, deliberate
short-circuit:

```cpp
// src/optimizer/type_pushdown.cpp:153-194
void CastCollect::VisitOperator(LogicalOperator &op) {
	/*
	 * Logical projection expressions are columns which reference underlying
	 * GETs. Don't process them, as they would add conflicts for every column
	 * used in projection. Example: PROJECTION(col) -> GET(col). We don't want
	 * to visit BoundColumnRefExpression in PROJECTION to avoid registering a
	 * non-existent conflict.
	 *
	 * However, CastReplace will visit them because we need to update their
	 * types if pushdown succeeded.
	 */
	if (op.type != LogicalOperatorType::LOGICAL_PROJECTION) {
		return LogicalOperatorVisitor::VisitOperator(op);
	}
	auto &projection = op.Cast<LogicalProjection>();

	// Only push casts from a projection that forwards just column refs and
	// casts and reaches a GET without a join in between. A constant or other
	// expression makes the projection ineligible.
	// See test/sql/copy/csv/test_csv_error_message_type.test (top-level cast
	// to VARCHAR must still push) and test_large_integer_detection.test (a
	// nested cast to VARCHAR must not).
	bool clean = ReachesPushdownGet(projection);
	for (const auto &e : projection.expressions) {
		if (e->GetExpressionClass() == ExpressionClass::BOUND_COLUMN_REF || BoundCastExpression::IsCast(*e)) {
			continue;
		}
		clean = false;
	}
	if (clean) {
		for (const auto &e : projection.expressions) {
			if (BoundCastExpression::IsCast(*e)) {
				top_level_casts.insert(e.get());
			}
		}
	}
	if (projections.count(projection.table_index)) {
		VisitOperatorChildren(op);
		return;
	}

	LogicalOperatorVisitor::VisitOperator(op);
}
```

The registered passthrough projection itself is special-cased *twice* over:
once to decide whether the projection is "clean" enough to seed
`top_level_casts` (every expression must be either a bare column reference or
a cast — anything else, e.g. a constant or an arithmetic expression sitting
alongside the cast, disqualifies the whole projection), and once more,
separately, to skip visiting its own `BOUND_COLUMN_REF` expressions as
ordinary column references (which would otherwise register a spurious
conflict against the very column the projection exists to forward).

`top_level_casts` is worth pausing on, because it is the mechanism behind a
subtlety the code comment at `type_pushdown.cpp:169-174` states directly: *"Only
push casts from a projection that forwards just column refs and casts and
reaches a GET without a join in between... See
test/sql/copy/csv/test_csv_error_message_type.test (top-level cast to
VARCHAR must still push) and test_large_integer_detection.test (a nested
cast to VARCHAR must not)."* A cast is only ever a pushdown *candidate* if it
is itself one of the projection's own top-level output expressions — a cast
buried inside some other function call one level deeper
(`some_fn(CAST(col AS T))`) is never inserted into `top_level_casts`, and so
can never become the seed of a pushdown, even though `BoundCastExpression::
IsCast` would still recognize it as a cast if asked. The distinction matters
because the *whole point* of the pushdown is that the reader will produce
`col` **already** cast to `T` and the cast disappears entirely — that is
only sound when the cast's result is genuinely what the query wants exposed
at that position, not an intermediate step feeding some other computation
the pass isn't analyzing at all.

`ReachesPushdownGet` (`type_pushdown.cpp:137`) is the chain-walking check
that decides whether a *projection* — as opposed to a single expression —
actually bottoms out cleanly at an eligible `Get`:

```cpp
// src/optimizer/type_pushdown.cpp:126-151
/**
 * A GET reachable through a single-child chain of projections. A join or any
 * other multi-child operator breaks the chain.
 *
 * LOGICAL_FILTER also breaks the chain. In SELECT CAST/TRY_CAST(x) WHERE g(x)
 * we can't push down cast if g(x) isn't pushed. If g(x) filters some rows
 * on which cast would fail, the query passed by default but fails if cast is
 * pushed and runs before g(x).
 *
 * See test/sql/copy/csv/test_insert_into_types.test in duckdb (cast not
 * pushed past a join)
 */
static bool ReachesPushdownGet(const LogicalOperator &op) {
	const LogicalOperator *cur = &op;
	while (cur->children.size() == 1) {
		cur = cur->children[0].get();
		switch (cur->type) {
		case LogicalOperatorType::LOGICAL_GET:
			return cur->Cast<LogicalGet>().function.projection_expression_pushdown != nullptr;
		case LogicalOperatorType::LOGICAL_PROJECTION:
			continue;
		default:
			return false;
		}
	}
	return false;
}
```

This is the single most important correctness argument in the whole file,
and it is the reason this pass is sequenced *after* `FILTER_PUSHDOWN` (#6)
specifically, not merely somewhere-late-in-general: a residual
`LogicalFilter` sitting between the cast's projection and the scan
unconditionally blocks the chain, full stop — `LOGICAL_FILTER` isn't in the
`switch`'s explicit cases, so it falls to `default: return false`. The
comment's own worked example is precise: `SELECT CAST(x AS T) FROM t WHERE
g(x)` is safe today because `g(x)` runs first, filtering out any row whose
`x` the cast would have failed on; if the cast were pushed into the scan and
the scan applied it *during* its own parse/decode step, before `g(x)` ever
gets a chance to filter, a row that used to be silently excluded by `g(x)`
could instead make the scan itself throw. By the time this pass runs
(#44), `FILTER_PUSHDOWN` (#6) has already had every earlier structural pass
between it and here to finish its work — so a filter that legitimately
still sits as a `LogicalFilter` at this point in the pipeline is a filter
that *couldn't* be fully absorbed into the scan's own `table_filters` (a
multi-column predicate, one with a function call that isn't representable
as a `TableFilterSet` entry, etc.), and `ReachesPushdownGet`'s blanket
refusal to look past any `LogicalFilter` is the conservative, always-correct
response to that residual uncertainty. A filter that *did* get fully
absorbed into `table_filters` is gone from the operator tree entirely by
this point — it isn't a `LogicalFilter` node any more, it's data sitting on
the `LogicalGet` — so it never has a chance to block `ReachesPushdownGet` in
the first place. Running this pass any earlier than #44 — say, interleaved
with or before `FILTER_PUSHDOWN` itself — would make this whole argument
fall apart: a filter destined to be fully absorbed a few passes later would
still visibly be a `LogicalFilter` right now, needlessly blocking a
pushdown that later turns out to have been perfectly safe.

The same hazard appears in a *different* pass, `UNUSED_COLUMNS`
(`remove_unused_columns.cpp`), not `TYPE_PUSHDOWN`. That pass can move a
cast of a *struct field* down into a child projection or the scan. Then the
cast runs on every row, also on rows that a filter, join or `LIMIT` above
would have removed. In issue #28617, casting a struct field to `DATE`
failed on an empty string, even though the `WHERE` clause removed that row.
So `UNUSED_COLUMNS` moves the cast down only if it cannot fail; a cast that
can fail stays where the query wrote it. `test/sql/optimizer/struct_extract_cast_filter.test`
covers it. `TYPE_PUSHDOWN` avoids the same trap in its own way: it stops at
any `LogicalFilter`, as shown above.

The two `VisitReplace` overrides do the actual conflict bookkeeping, and
they're near-mirror images of each other:

```cpp
// src/optimizer/type_pushdown.cpp:197-204
unique_ptr<Expression> CastCollect::VisitReplace(BoundColumnRefExpression &expr, unique_ptr<Expression> *ptr) {
	if (const auto binding = Resolve(expr.Binding(), analyses, projections)) {
		// Column is used without cast applied to it, register a conflict.
		// Not emplace() as we need to update the value if it was present
		binding->analysis.col_to_expr[binding->column_index] = nullptr;
	}
	return std::move(*ptr);
}
```

Any *bare* reference to a tracked column, anywhere in the plan, is an
unconditional conflict — the column is being used in its native, uncast
form somewhere, so the reader cannot be asked to retype it everywhere.

```cpp
// src/optimizer/type_pushdown.cpp:206-239
unique_ptr<Expression> CastCollect::VisitReplace(BoundFunctionExpression &expr, unique_ptr<Expression> *ptr) {
	if (!BoundCastExpression::IsCast(expr)) {
		// Descend into non-cast function children so e.g. fn(col, other) still sees "col"
		return nullptr;
	}
	auto &cast_child = BoundCastExpression::Child(expr);
	if (cast_child.GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
		// Descend into children so e.g. fn(col, other) still sees "col" and
		// registers a conflict
		return nullptr;
	}
	const auto &bound_col = cast_child.Cast<BoundColumnRefExpression>();
	const auto binding = Resolve(bound_col.Binding(), analyses, projections);
	if (!binding) {
		return nullptr;
	}
	auto &col_to_cast = binding->analysis.col_to_expr;

	if (auto it = col_to_cast.find(binding->column_index); it == col_to_cast.end()) {
		// Only a top-level projection cast starts a candidate.
		if (top_level_casts.count(&expr)) {
			col_to_cast.emplace(binding->column_index, &expr);
		}
	} else if (it->second == nullptr || it->second->GetReturnType() != expr.GetReturnType() ||
	           BoundCastExpression::IsTryCast(it->second->Cast<BoundFunctionExpression>()) !=
	               BoundCastExpression::IsTryCast(expr)) {
		// Different target type, or already a conflict.
		// If reader can push CAST but not TRY_CAST or vice versa, we can't
		// pretend thery are the same
		it->second = nullptr;
	}

	return std::move(*ptr);
}
```

A cast of the tracked column is only ever a pushdown *candidate* the first
time it's seen, and only if it was registered in `top_level_casts` (i.e. it
came directly from a clean passthrough projection, not from somewhere
nested). Every *subsequent* sighting of a cast on the same column must agree
exactly on two things: the resulting **type**, and whether it's `CAST` or
`TRY_CAST` — a mismatch on either axis nulls out the entry, permanently
disqualifying the column. `csv_type_pushdown.test`'s "CAST and TRY_CAST
conflict in NULL-semantics" case (§1.4) is the direct evidence for the
second check: `CAST` throws on a failed conversion, `TRY_CAST` returns
`NULL` — these are observably different behaviors, so the reader cannot
possibly satisfy both simultaneously by picking one target type for the
column.

`CastReplace` (`type_pushdown.cpp:241-296`) does the mirror-image rewrite once
`any_pushed` is true — for every `BOUND_COLUMN_REF` or cast-of-a-column-ref
whose binding resolves to a column that *did* get accepted
(`CanPushdownColumn`, `type_pushdown.cpp:298-301` — simply checks the map entry
exists and is non-null), it updates the expression's declared return type in
place (for a bare column reference reached through a passthrough
projection) or replaces the whole `CAST(...)` node with the bare column
reference underneath it, now retyped (for an actual cast expression),
propagating the new type into any registered projection's own `types` vector
too so the projection's declared output type stays consistent with what it
now actually forwards.

### 1.3 The reader side: why CSV, and only CSV, implements this for casts

`FindGetsAndProjections` only ever registers a `Get` if
`get.function.projection_expression_pushdown != nullptr` — the callback
itself is opt-in, per table function, and a full-text search of the tree
confirms exactly which readers set it:

```
src/function/table/read_csv.cpp:206:  read_csv.projection_expression_pushdown = PushdownProjectionExpressionMultiFile;
extension/parquet/parquet_multi_file_info.cpp:293: result.projection_expression_pushdown = ParquetScanFunction::ProjectionExpressionPushdown;
```

(The Parquet callback itself is the method
`ParquetScanFunction::ProjectionExpressionPushdown` in
`extension/parquet/read_single_parquet_file.cpp:461`; §2.3 covers it.)

Exactly two table functions in this checkout register this callback at all
— CSV and Parquet — and, crucially, **the two callbacks accept entirely
disjoint kinds of expression**. CSV's:

```cpp
// src/function/table/read_csv.cpp:153
static bool PushdownProjectionExpression(ClientContext &context, const TableFunctionProjectionExpressionInput &input) {
	if (!BoundCastExpression::IsCast(input.expr)) {
		return false;
	}
	const auto &cast = input.expr.Cast<BoundFunctionExpression>();
	const auto &target_type = cast.GetReturnType();
	auto &bind_data = input.get.bind_data->Cast<MultiFileBindData>();
	// With a custom schema (the 'schema' option, or a MultiFileReader that binds its own schema) the files are mapped
	// onto that schema, not onto bind_data.columns, so retyping the columns would not reach the column mapping.
	// See test/optimizer/pushdown/csv_type_pushdown_schema.test
	if (!bind_data.reader_bind.schema.empty()) {
		return false;
	}
	const idx_t idx = input.get.GetColumnIds()[input.column_index].GetPrimaryIndex();
	// Hive-partition and filename columns are produced from the file path by
	// separate finalize expressions, not parsed from the file. Retyping them
	// here would desync those expressions, so leave the cast in place.
	// See test/sql/copy/csv/csv_hive.test
	for (const auto &partition : bind_data.reader_bind.hive_partitioning_indexes) {
		if (partition.index == idx) {
			return false;
		}
	}
	if (bind_data.reader_bind.filename_idx.IsValid() && bind_data.reader_bind.filename_idx.GetIndex() == idx) {
		return false;
	}
	bind_data.types[idx] = target_type;
	bind_data.columns[idx].type = target_type;
	return true;
}
```

The CSV callback also says no when the scan has a *custom schema* (the
`reader_bind.schema.empty()` check shown above). A custom schema comes from `read_csv`'s `schema` option, or
from an extension's `MultiFileReader` that binds its own schema (the
test's comment names the glue extension, which maps Hive text files this
way). In that case each file is mapped onto the schema's columns, not onto
`bind_data.columns`. So changing `bind_data.columns[idx].type` would not
reach the real column mapping, and the scan and the plan would disagree
about the type. The Parquet callback has the same guard (§2.3).

`test/optimizer/pushdown/csv_type_pushdown_schema.test` reads a CSV whose
columns are declared `VARCHAR` and mapped onto a schema where `id` is
`INTEGER`:

```sql
CREATE MACRO schema_scan() AS TABLE
SELECT * FROM read_csv('{TEST_DIR}/type_pushdown_schema.csv', header = false, auto_detect = false,
    columns = {'id': 'VARCHAR', 's': 'VARCHAR'},
    schema = MAP {'id': {name: 'id', type: 'INTEGER', default_value: NULL},
                  's': {name: 's', type: 'VARCHAR', default_value: NULL}});

-- the cast stays in the plan
EXPLAIN SELECT CAST(id AS VARCHAR) FROM schema_scan();
----
physical_plan	<REGEX>:.*CAST.*
```

The query results (`1`, `2`, `3`) must be the same as without the cast.

The CSV callback demands `BoundCastExpression::IsCast(input.expr)` — it accepts *only*
casts, and rejects everything else outright. Parquet's, walked in full in
§2.3, demands `name == "strlen" || name == "octet_length"` — it accepts
*only* those two specific scalar functions, and rejects a `CAST` the moment
it checks the function's name (a cast's underlying function is never
literally named `"strlen"` or `"octet_length"`). So in this checkout, **CSV
participates in `TYPE_PUSHDOWN` and never in `SCALAR_FN_PUSHDOWN`; Parquet
participates in `SCALAR_FN_PUSHDOWN` and never in `TYPE_PUSHDOWN`.** This
isn't a gap the two passes happen to have left unfilled — it follows
directly from what each file format's decode step actually looks like.

CSV values start life as **untyped text**. Some conversion from raw bytes to
a typed value has to happen no matter what — there is no such thing as
"the CSV reader's native physical type" the way there is for a Parquet
column's on-disk encoding. Ordinarily, the CSV reader's own type-detection
("sniffing") picks a type per column from a sample of the data (or the user
supplies one explicitly), the parser converts every field's text into that
type, and if the query actually wants a *different* type, a separate `CAST`
operator runs afterward, doing a second conversion from the sniffed type to
the requested one. `TYPE_PUSHDOWN` collapses this into one step: it hands
the CSV reader the query's actually-requested type up front
(`bind_data.types[idx] = target_type`), so the parser's *one* unavoidable
text-to-value conversion targets the right type directly, and the second,
otherwise-redundant `CAST` never has to run at all. `PushdownProjectionExpressionMultiFile`
(`read_csv.cpp:187`) applies the same retyping to the multi-file wrapper's
own `expected_types`, and the function's own comment states the
correctness-relevant side effect of doing it this way rather than as two
steps: *"the CSV reader converts to it itself, and `ignore_errors` applies
to the conversions that fail"* — i.e. pushing the cast doesn't just save a
step, it changes *which* conversion attempt `ignore_errors`'s failure
handling actually governs, collapsing what would otherwise be two
independently-fallible conversions into the one the user's own `ignore_errors`
setting was presumably written with in mind.

Parquet has no equivalent opportunity for **casts** specifically, because a
Parquet column's physical encoding (`INT32`, `INT64`, `BYTE_ARRAY`, and so
on) is fixed at write time — pushing a `CAST` target type into the reader
wouldn't let it skip a decode step the way CSV's retyping does; the reader
would still have to decode the column's actual on-disk physical type first,
and only then could it consider converting that decoded value to something
else, which is exactly the two-step sequence a `CAST` operator already
performs. There's no first, unavoidable "text-to-value" conversion for a
pushed target type to fold into. What Parquet *can* uniquely skip is
covered in Part II: for exactly two functions, it can avoid decoding a
string/blob value's *content* at all, computing a derived scalar straight
from the format's own per-value length metadata — a different kind of win
than "the type you get is narrower," and one `TYPE_PUSHDOWN`'s cast-shaped
matcher structurally cannot express, which is exactly why it needed its own
sibling pass rather than being one more case inside this one (§3.1).

### 1.4 `csv_type_pushdown.test`, walked in full

The file is `test/optimizer/pushdown/csv_type_pushdown.test`. Every scenario below reads `test/{DATA_DIR}/csv/real/lineitem_sample.csv`
with `delim='|'`, and every `EXPLAIN` check is against the *physical* plan
(`profiling_renderer_settings = MAP {'operator_casing': 'upper'}` is set at
the top purely so the `<REGEX>` checks can match uppercase operator names
consistently). `<!REGEX>:.*PROJECTION.*` means pushdown succeeded — no
projection operator survives between the scan and the final output, because
the cast disappeared entirely and the scan is now producing the requested
type directly.

**Uncasted usage — no pushdown needed at all**, since there's no cast to
push:

```sql
EXPLAIN SELECT column00 FROM read_csv('{DATA_DIR}/csv/real/lineitem_sample.csv', delim='|');
----
physical_plan	<!REGEX>:.*PROJECTION.*
```

**A single, consistent cast — pushed:**

```sql
EXPLAIN SELECT column00::UTINYINT FROM read_csv(...);
----
physical_plan	<!REGEX>:.*PROJECTION.*

SELECT column00::UTINYINT FROM read_csv(...) ORDER BY 1;
----
1  1  1  1  1  1  2  3  3  3
```

The CSV column's sniffed/default type (something wide enough to hold every
sampled value, ordinarily) is entirely replaced by `UTINYINT` at the scan
itself; no separate cast survives anywhere in the plan.

**Cast and bare usage together — a genuine conflict, no pushdown:**

```sql
EXPLAIN SELECT column00, column00::UTINYINT FROM read_csv(...);
----
physical_plan	<REGEX>:.*PROJECTION.*
```

`column00` is referenced once bare and once cast — `CastCollect`'s
`VisitReplace(BoundColumnRefExpression&)` unconditionally nulls the entry
the moment it sees the bare reference (§1.2), regardless of which one it
visits first, so the column keeps its native type and the cast survives as
an ordinary expression in a surviving `LogicalProjection`. Both output
values are still computed correctly (`1  1`, `2  2`, `3  3`, ...) — this is
purely a missed optimization opportunity, never a correctness question.

**A filter on the *casted* expression blocks pushdown, even though the same
column, used uncasted, pushes down fine on its own:**

```sql
# Column is used uncasted with filter on casted.
# CSV reader doesn't support CAST expression pushdown.
EXPLAIN SELECT column00 FROM read_csv(...) WHERE column00::UTINYINT > 0;
----
physical_plan	<REGEX>:.*FILTER.*
```

This is `ReachesPushdownGet`'s `LOGICAL_FILTER`-breaks-the-chain rule in
direct action (§1.2): `column00::UTINYINT > 0` is a comparison, not a
representable single-column `TableFilterSet` entry the way `column00 > 0`
alone would be, so it survives as an ordinary `LogicalFilter` sitting
directly above the scan — and that residual filter node breaks the chain
`ReachesPushdownGet` needs, so the *outer* `SELECT column00` (uncasted, and
otherwise perfectly pushable on its own) never even gets a chance, because
the cast it would need to reason about (inside the filter, not the `SELECT`
list) is exactly the thing the chain-walk refuses to look past. The test's
own inline comment states the practical summary plainly: the CSV reader
doesn't support pushing a `CAST` that's buried inside a filter predicate at
all, so this shape is left alone entirely, correctly if conservatively.

**Same cast used in both `SELECT` and `ORDER BY` — no conflict, still
pushed:**

```sql
EXPLAIN SELECT column00::UTINYINT FROM read_csv(...) ORDER BY column00::UTINYINT;
----
physical_plan	<!REGEX>:.*PROJECTION.*
```

Two syntactically identical casts of the same column, in two different
clauses, are the *same* candidate expression as far as `col_to_expr` is
concerned — the second sighting matches the first sighting's return type
and `CAST`/`TRY_CAST`-ness exactly, so no conflict is registered.

**Two different target types of the same column — an explicit conflict:**

```sql
EXPLAIN SELECT column00::UTINYINT, column00::USMALLINT FROM read_csv(...);
----
physical_plan	<REGEX>:.*PROJECTION.*
```

The reader can only retype the scan's output for `column00` once; asked for
two different types simultaneously, `CastCollect`'s return-type mismatch
check (`it->second->GetReturnType() != expr.GetReturnType()`) fires and
nulls the entry — both casts survive as ordinary post-scan expressions.

**`TRY_CAST` alone pushes exactly like `CAST` alone:**

```sql
EXPLAIN SELECT TRY_CAST(column00 AS UTINYINT) FROM read_csv(...);
----
physical_plan	<!REGEX>:.*PROJECTION.*
```

**`TRY_CAST` inside a filter blocks pushdown for the same
chain-breaking reason `CAST` did:**

```sql
EXPLAIN SELECT column00 FROM read_csv(...) WHERE TRY_CAST(column00 AS UTINYINT) > 0;
----
physical_plan	<REGEX>:.*FILTER.*
```

**A cast that also appears in a `WHERE` clause on the very expression being
selected still survives as `CAST`, not pushed — because the filter, being on
the cast's own output, blocks the chain regardless of which side of it the
`SELECT` sits on:**

```sql
# CSV reader doesn't push down filters. We can't reorder filter after
# projection because this may break correctness
EXPLAIN SELECT TRY_CAST(column00 AS UTINYINT) FROM read_csv(...)
WHERE TRY_CAST(column00 AS UTINYINT) > 0;
----
physical_plan	<REGEX>:.*CAST.*
```

**`CAST` and `TRY_CAST` of the *same* target type still conflict, because
their `NULL`-vs-`throw` behavior on a bad value differs:**

```sql
# CAST and TRY_CAST conflict in NULL-semantics. Not pushed down
EXPLAIN SELECT column00::UTINYINT, TRY_CAST(column00 AS UTINYINT) FROM read_csv(...);
----
physical_plan	<REGEX>:.*CAST.*
```

This is the `IsTryCast(...) != IsTryCast(expr)` branch of the conflict check
firing even though the return types agree exactly — proof the check really
does track both axes independently, not just the target type.

**Passthrough view, one level deep — pushed:**

```sql
CREATE VIEW pv1 AS SELECT column00 FROM read_csv(...);
EXPLAIN SELECT column00::UTINYINT FROM pv1;
----
physical_plan	<!REGEX>:.*PROJECTION.*
```

**Nested view, two levels deep — the documented one-level limitation, not
pushed:**

```sql
CREATE VIEW pv2 AS SELECT column00 FROM pv1;
EXPLAIN SELECT column00::UTINYINT FROM pv2;
----
physical_plan	<REGEX>:.*PROJECTION.*
```

This is `Resolve`'s single-level-of-indirection restriction (§1.1) made
concrete: `pv2`'s own passthrough projection resolves to `pv1`'s table
index, and `pv1`'s column, in turn, resolves to the actual `Get` — but
`Resolve` only follows *one* such hop before giving up, so a query against
`pv2` never reaches the `Get` at all as far as this pass's bookkeeping is
concerned, and the cast survives as an ordinary expression, correctly
computing the right values (`1 1 1 1 1 1 2 3 3 3`) without the pushdown
optimization.

**Two independently-aliased references to the same column, each cast the
same way, both still pushed — because the pass reasons per `ProjectionIndex`
within one `Get`, and both aliases resolve back to that same single
column:**

```sql
CREATE VIEW pv3 AS SELECT column00 AS x, column00 AS y FROM read_csv(...);
SELECT x::UTINYINT, y::UTINYINT FROM pv3 ORDER BY x DESC LIMIT 1;
----
3	3
```

### 1.5 `type_pushdown_projection_index.test` — a `ProjectionIndex` regression test filed under the wrong name

This is the file the task specifically named as testing `TYPE_PUSHDOWN`, and
it's worth being precise about what it actually contains, because the
finding is genuinely surprising: it contains **no cast at all**.

```sql
require parquet

COPY (
    SELECT i % 2 = 0 AS f1, i AS f2, repeat('x', (i % 5)::INT) AS s
    FROM range(100) t(i)
) TO '{TEST_DIR}/type_pushdown_projection_index.parquet' (FORMAT parquet);

query I
SELECT sum(strlen(s))
FROM (
    SELECT s FROM '{TEST_DIR}/type_pushdown_projection_index.parquet'
    WHERE f1 AND f2 >= 2
);
----
100

CREATE VIEW view AS
SELECT s FROM '{TEST_DIR}/type_pushdown_projection_index.parquet'
WHERE f1 AND f2 >= 2;

query I
SELECT sum(strlen(s)) FROM view;
----
100
```

`strlen(s)` is a scalar function of a bare column, not a cast — this test
exercises `SCALAR_FN_PUSHDOWN`, not `TYPE_PUSHDOWN`, despite its file path
living under a name that says otherwise. The reason it still belongs to
this pass's discussion, and the reason for its actual name, is the shared
infrastructure documented in §1.1: `GetAnalysis::StorageIndex` and
`ProjectionIndex` are defined once, in `type_pushdown.cpp`/`.hpp`, and used
identically by both passes. What this specific test guards against is a bug
class in *that* shared bookkeeping, not in either pass's own matcher: the
Parquet file is written with three columns (`f1`, `f2`, `s`, in that
storage order — `s` is the *third* column, storage index 2), but the query
only ever needs `s` in its final output; `f1` and `f2` exist purely to
satisfy the `WHERE` clause. After `UNUSED_COLUMNS`/`COLUMN_LIFETIME` and
`FILTER_PUSHDOWN` have already run (both well before pass #45), the `Get`'s
own output-column list (what `ProjectionIndex` indexes into) and the file's
raw storage-column ordering (what `StorageIndex` maps back to) diverge —
`s` might be the *only* projected output column (`ProjectionIndex` 0) while
its `StorageIndex` is still 2, or `f1`/`f2` might still need to be scanned
for the filter without being projected out at all. A pass that confused
these two indices — using a `ProjectionIndex` where a `StorageIndex` was
needed, or vice versa — would silently retype or read back the *wrong*
column. The test's second half (through a `CREATE VIEW`) additionally
exercises the `Resolve`-through-one-passthrough-projection path (§1.1)
against this same index-confusion hazard, rather than the bare-`Get` path
the first half exercises. `sum(strlen(s)) = 100` is the correctness oracle:
if the wrong column's bytes were retyped or read, the sum of the repeated-
`'x'`-string lengths across the `WHERE f1 AND f2 >= 2` rows would not come
out to exactly `100`.

---

## Part II — `SCALAR_FN_PUSHDOWN`

**One-sentence purpose**, again from the call site's own comment:

```cpp
// src/optimizer/optimizer.cpp:560-564
// Pushdown scalar functions in SELECT like SELECT strlen(str) to readers
RunOptimizer(OptimizerType::SCALAR_FN_PUSHDOWN, [&] {
	ScalarFnPushdown fn_pushdown(context);
	plan = fn_pushdown.Optimize(std::move(plan));
});
```

Mechanically, this pass is the *same* `PushdownOptimize<>` algorithm walked
through in §1.1, instantiated with a different `Collect`/`Replace` pair.
`scalar_fn_pushdown.hpp` (54 lines of declarations, `#include`ing `type_pushdown.hpp`) and
`scalar_fn_pushdown.cpp` (122 lines) contain nothing but `ScalarFnCollect`,
`ScalarFnReplace`, and the thin `ScalarFnPushdown` wrapper class — every
piece of `Analyses`, `Projections`, `GetAnalysis`, `Resolve`,
`FindGetsAndProjections`, and `PushdownOptimize<>` itself is reused verbatim
from `type_pushdown.cpp`/`.hpp`. This is worth stating plainly: the two
`OptimizerType`s are separately named, separately timed, and separately
toggleable (`disabled_optimizers = 'type_pushdown'` does not disable
`scalar_fn_pushdown`, and vice versa), but at the implementation level,
`scalar_fn_pushdown.cpp` is best understood as "one more matcher plugged
into `type_pushdown.cpp`'s engine," not as an independent piece of
machinery built from scratch.

### 2.1 What differs: the matcher shape, and the conflict-equality check

`ScalarFnCollect::VisitReplace(BoundFunctionExpression&)`
(`scalar_fn_pushdown.cpp:37`) is the piece that actually differs from
`CastCollect`'s function-visiting overload:

```cpp
// src/optimizer/scalar_fn_pushdown.cpp:37-66
unique_ptr<Expression> ScalarFnCollect::VisitReplace(BoundFunctionExpression &expr, unique_ptr<Expression> *ptr) {
	if (BoundCastExpression::IsCast(expr)) {
		// Casts are handled by the dedicated cast pushdown (CastCollect/CastReplace). Descend into the cast so its
		// child column is registered as a conflict for scalar-function pushdown, matching the pre-function behavior.
		return nullptr;
	}
	if (expr.GetChildren().size() != 1 ||
	    expr.GetChildren()[0]->GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
		// Descend into children so e.g. fn(col, other) still sees "col" and
		// registers a conflict
		return nullptr;
	}
	const auto &bound_col = expr.GetChildren()[0]->Cast<BoundColumnRefExpression>();
	const auto binding = Resolve(bound_col.Binding(), analyses, projections);
	if (!binding) {
		return nullptr;
	}
	auto &col_to_expr = binding->analysis.col_to_expr;

	if (auto it = col_to_expr.find(binding->column_index); it == col_to_expr.end()) {
		// This is the first time we see the column used by a single function.
		col_to_expr.emplace(binding->column_index, &expr);
	} else if (it->second == nullptr || !it->second->Equals(expr)) {
		// Either column is used with different function in "expr" or
		// there already is a conflict.
		it->second = nullptr;
	}

	return std::move(*ptr);
}
```

Three differences from `CastCollect`, each deliberate:

1. **The shape matched is "exactly one child, and that child is a bare
   column reference,"** not "is a cast" — this is what lets `strlen(str)`,
   `octet_length(data)`, or in principle any single-argument scalar function
   of a column, register as a candidate. There is, notably, no equivalent of
   `top_level_casts` here — a scalar function candidate can be registered
   from *any* single-argument occurrence anywhere in the tree, not only from
   a clean passthrough projection's own top-level expression list. This
   matters for reading the test cases in §2.5: a filter predicate like
   `WHERE strlen(str) > 2` registers `strlen(str)` as a candidate exactly as
   readily as the `SELECT` list does, which is *not* how `CastCollect`
   treats a cast appearing inside a filter (§1.2's `top_level_casts`
   restriction has no counterpart here).
2. **A cast is explicitly excluded and deferred**, with the comment stating
   why: casts have their own dedicated pass, and this pass still descends
   into the cast's child so that child still registers as an ordinary
   conflict if it's referenced bare elsewhere — the two passes' conflict
   bookkeeping stays mutually consistent even though each only matches its
   own shape.
3. **The "does this new sighting match the existing one" check is
   `Expression::Equals`, not a field-by-field comparison of return type and
   a boolean flag.** Two occurrences of the *literal same function call*
   (`strlen(str)` and `strlen(str)`) satisfy `Equals` trivially. Two
   occurrences that compute the *same value* under different names
   (`strlen(str)` and `len(str)`, if `len` were an alias) do **not** satisfy
   `Equals`, because `Equals` compares expression-tree structure, not
   runtime semantics — DuckDB's actual `len`/`strlen` are, in fact, distinct
   function catalog entries (not one an alias of the other), and the test
   suite exploits exactly this to demonstrate the conflict path (§2.5,
   "strlen with filter using len").

`ScalarFnReplace` is a near-verbatim copy of `CastReplace` (§1.2) — same
`CanPushdownColumn` check, same retype-in-place logic for a bare column
reference reached through a registered projection, same "replace the whole
function-call node with its now-retyped child" logic for a matched
`fn(col)` node, with the identical up-front `BoundCastExpression::IsCast`
early-return deferring to the cast pass.

### 2.2 The virtual-column carve-out, inherited for free

Because `Resolve` (shared code, §1.1) already returns `nullopt` for any
virtual column reference before either pass's visitors ever see it, a
reference to `file_row_number` or similar alongside a tracked column never
registers as a conflict for that column, and is never itself considered a
pushdown target. `scalar_function_pushdown.test`'s own comment states this
directly (§2.5): *"virtual columns don't block pushdown."* This is not a
scalar-function-specific carve-out at all — it falls out of one guard in
one shared function, and both passes get it automatically.

This guard depends on `IsRealGetColumn` turning a column *position* into
a column id first (§1.1). Two details are worth stating plainly:

- The test case quoted above reads `read_parquet(..., file_row_number=true)`.
  With that option, `file_row_number` is an ordinary output column that
  the multi-file reader fills in (a *generated* column), not a virtual
  column. It never blocked pushdown for `str` simply because it is a
  different column: no expression uses `str` bare.
- The case that really needs the virtual-column guard is a *function of
  the virtual column itself*. The end of `scalar_function_pushdown.test`
  has one:

  ```sql
  COPY (SELECT 'hello' AS s)
  TO '{TEST_DIR}/virtual_scalar.parquet' (FORMAT PARQUET);

  query II
  SELECT strlen(s), strlen(filename) > 0
  FROM read_parquet('{TEST_DIR}/virtual_scalar.parquet');
  ----
  5	true
  ```

  Here `filename` is a virtual column. `strlen(filename)` has exactly the
  `fn(col)` shape `ScalarFnCollect` looks for. Without the guard,
  `filename` could become a candidate and the pass would try to retype a
  column that has no slot in `returned_types`. Instead `Resolve` returns
  `nullopt` for it, `strlen(s)` is still pushed, and `strlen(filename)`
  runs as an ordinary function.

### 2.3 The reader side: Parquet's `strlen`/`octet_length` pushdown, in full

`ParquetScanFunction::ProjectionExpressionPushdown`
(`extension/parquet/read_single_parquet_file.cpp:461-535`) is considerably more
guarded than CSV's cast callback, because the win it's chasing is more
specific — not "parse into a different type," but **"compute a derived
scalar from the file's own length metadata, without ever materializing the
column's actual byte content."**

The Parquet reader is built on the shared `TableFunctionMultiFileReader`
framework. The callback is a method of `ParquetScanFunction` in
`read_single_parquet_file.cpp`, and `parquet_multi_file_info.cpp:293`
registers it. A few points about its guards:

- Hive-partition columns, `filename` and `file_row_number` are all handled
  by one call, `bind_data.IsGeneratedColumn(idx)`.
- The multi-file and "no initial reader" checks run early, right after the
  custom-schema check.
- One check refuses the pushdown when the scan has an explicit bound
  output schema and the output column does not match the physical column
  by name and type (`MatchesPhysicalColumn`).
- The length expression is stored on the single-file bind data with
  `parquet_data.AddProjectionExpression(idx, expression)`, and the
  multi-file wrapper's `expected_types` is updated too, the same way the
  CSV multi-file callback does it (§1.3).

```cpp
// extension/parquet/read_single_parquet_file.cpp:461-535
bool ParquetScanFunction::ProjectionExpressionPushdown(ClientContext &context,
                                                       const TableFunctionProjectionExpressionInput &input) {
	if (input.expr.GetExpressionClass() != ExpressionClass::BOUND_FUNCTION) {
		return false;
	}
	const auto &fn = input.expr.Cast<BoundFunctionExpression>();
	if (const Identifier &name = fn.Function().GetName(); name != "strlen" && name != "octet_length") {
		return false;
	}
	auto &bind_data = input.get.bind_data->CastNoConst<MultiFileBindData>();
	// Don't do pushdown on custom schema users like Ducklake, or when the schemas of several files are unified
	if (!bind_data.reader_bind.schema.empty() || !bind_data.union_readers.empty()) {
		return false;
	}
	// Don't do pushdown with multiple files - the metadata of all but the first file is not available upfront, and
	// the files may have their columns in a different order
	if (bind_data.file_list->GetExpandResult() == FileExpandResult::MULTIPLE_FILES || !bind_data.initial_reader) {
		return false;
	}
	const auto &column_id = input.get.GetColumnIds()[input.column_index];
	// Pushdown extract columns i.e. SELECT x.y.z have a complex nested type update
	if (column_id.IsPushdownExtract()) {
		return false;
	}
	const idx_t idx = column_id.GetPrimaryIndex();
	// Generated values do not come from the physical column, even when its name and type match.
	if (bind_data.IsGeneratedColumn(idx)) {
		return false;
	}
	// We run scalar function pushdown after filter pushdown - a filter pushed into the scan needs the column itself
	if (input.get.table_filters.HasFilter(input.column_index)) {
		return false;
	}
	auto &reader = bind_data.initial_reader->Cast<TableFunctionFileReader>();
	if (!reader.bind_data) {
		return false;
	}
	auto &parquet_data = reader.bind_data->Cast<ReadSingleParquetFileData>();
	auto &metadata_reader = parquet_data.GetMetadataReader(context);
	auto &children = metadata_reader.root_schema->children;
	if (idx >= children.size() || idx >= bind_data.types.size() || idx >= reader.types.size()) {
		return false;
	}
	if (!bind_data.file_options.bound_schema.empty() &&
	    (idx >= reader.columns.size() ||
	     !bind_data.MatchesPhysicalColumn(idx, reader.columns[idx].name, reader.types[idx]))) {
		// The pushed expression addresses a physical column, before name mapping and conversion.
		return false;
	}
	const idx_t column_flat_idx = children[idx].column_index;
	for (const auto &group : metadata_reader.GetFileMetadata()->row_groups) {
		D_ASSERT(column_flat_idx < group.columns.size());
		for (const Encoding::type type : group.columns[column_flat_idx].meta_data.encodings) {
			if (type == duckdb_parquet::Encoding::DELTA_LENGTH_BYTE_ARRAY) {
				return false;
			}
		}
	}

	const LogicalType type = LogicalType::BIGINT;
	const ParquetReaderProjectionExpression expression {ParquetReaderProjectionExpressionType::BYTE_LENGTH, type};
	// the file is read with this bind data, and every other bind of it takes the expression from here
	parquet_data.AddProjectionExpression(idx, expression);
	bind_data.types[idx] = type;
	bind_data.columns[idx].type = type;
	// the reader of the file was bound before the expression was pushed down - update its types
	reader.types[idx] = type;
	if (idx < reader.columns.size()) {
		reader.columns[idx].type = type;
	}
	auto &data = bind_data.bind_data->Cast<TableFunctionMultiFileData>();
	if (idx < data.options.expected_types.size()) {
		data.options.expected_types[idx] = type;
	}
	return true;
}
```

Walking the guards in order, because each one maps to a specific correctness
or feasibility argument:

- **Function-name allowlist.** Only `strlen` and `octet_length` are
  accepted — no general mechanism for "any single-argument scalar function"
  exists on the Parquet side; the reader has to know, specifically, how to
  compute *this* function's result from its own on-disk representation
  without decoding the value.
- **`BOUND_FUNCTION` class check first**, which — as established in §1.3 —
  also has the effect of silently rejecting any `CAST` (since a cast is
  itself a `BoundFunctionExpression`, but never one literally named `strlen`
  or `octet_length`), keeping this callback's scope strictly disjoint from
  CSV's.
- **No custom schema / union readers (Ducklake and similar).** A custom
  schema maps file columns onto other names and types. "Union readers"
  means the schemas of several files are being merged into one
  (`union_by_name`). In both cases the column the query sees is not simply
  the column in the file.
- **Single-file only.** A multi-file `read_parquet([...])` scan declines
  entirely, regardless of encoding, because per-file metadata for anything
  past the first file isn't available at bind time, and different files in
  one logical scan are permitted to have different physical schemas/column
  orderings — this is a feasibility limit, not a correctness one, and it is
  the guard actually responsible for the "unsupported encoding" test's
  `EXPLAIN` result in §2.5, more directly than the encoding check that
  follows it. The same line also declines when there is no
  `initial_reader` (no file was opened at bind time).
- **`IsPushdownExtract()` rejects nested-field extraction paths** — a
  reference like `s.x` for a struct column `s` is represented internally as
  a special `ColumnIndex` marking a dotted extraction path into a nested
  type, not a plain top-level column. Retyping the *whole* struct column
  based on a scalar function of just one leaf field would corrupt every
  *other* field of that struct still needed in its original form elsewhere
  — this guard is why `strlen(s.x)`/`strlen(s.y.z)` never push (§2.5), while
  `strlen(v)` on an adjacent plain top-level `VARCHAR` column in the very
  same file *does* push, in the same test.
- **Generated columns are excluded.** `IsGeneratedColumn`
  (`src/common/multi_file/multi_file_reader.cpp:77-88`) returns `true` for
  the `filename` column, the `file_row_number` column, and every Hive
  partition column. These columns' values come from the file *path* or the
  row position, not from column data in the file, so there is no stored
  length to read. CSV's callback has a similar guard (§1.3).
- **`table_filters.HasFilter(input.column_index)`** is the guard whose
  comment states the ordering dependency on `FILTER_PUSHDOWN` explicitly
  and in the reader's own words: *"We run scalar function pushdown after
  filter pushdown - a filter pushed into the scan needs the column
  itself."* This is a genuinely separate,
  reader-side safety check layered on top of the optimizer-side conflict
  tracking (§2.1) — the optimizer's own `ScalarFnCollect` walk cannot see
  inside `table_filters` either (it's data on the `Get`, not an expression
  node the `LogicalOperatorVisitor` visits), so without this explicit guard
  the reader could be asked to retype a column whose *decoded values* a
  zonemap-driven `table_filters` entry is still going to need to inspect in
  their original, undecoded form.
- **Bounds and bound-schema checks.** The callback checks that `idx` is
  inside the file's schema and inside both type lists. Then, if the scan
  has an explicit bound output schema (`file_options.bound_schema`), it
  requires `MatchesPhysicalColumn`: the output column must have the same
  name and the same type (including collation) as the physical column, and
  must not be generated (`multi_file_reader.cpp:90-94`). The comment gives
  the reason: *"The pushed expression addresses a physical column, before
  name mapping and conversion."* If the output column is, say, a renamed or
  converted version of the file column, the byte length of the file value
  is not the length of the output value.
- **`DELTA_LENGTH_BYTE_ARRAY` encoding rejection**, scanned per row group.
  `PLAIN` and dictionary-encoded `BYTE_ARRAY` columns store each value with
  an explicit length prefix that's trivially available without decoding the
  value's actual bytes; `DELTA_LENGTH_BYTE_ARRAY` instead stores lengths in
  a separately delta-compressed stream, which isn't a "read one length
  field" operation the same way — so the pass declines rather than risk an
  encoding whose length isn't cheaply extractable.
- **`BYTE_LENGTH` is the payoff.** Once every guard passes, the column's
  declared type becomes `BIGINT`, and `ParquetReaderProjectionExpression{
  ParquetReaderProjectionExpressionType::BYTE_LENGTH, ...}` tells the
  Parquet decoder, at the physical-scan level, to emit the value's stored
  byte length directly instead of decoding and materializing the string or
  blob content at all. This is not "a narrower type standing in for a wider
  one" the way CSV's cast pushdown is — the column's *logical content*
  changes (from the string itself to that string's length), and the
  benefit is skipping full value materialization entirely, not just
  choosing a different target representation for the same value.

### 2.4 `octet_length` and why it's grouped with `strlen`

`strlen` (character length) and `octet_length` (byte length) are aliased to
the identical `BYTE_LENGTH` pushdown mechanism here — for a `VARCHAR`
column specifically, DuckDB stores UTF-8 bytes, and for the pushdown's
purposes the underlying Parquet length-prefix metadata that both functions
would need to read from is the same per-value byte count regardless of
which of the two SQL functions asked for it; the difference between
"character length" and "byte length" only matters for genuinely multi-byte
UTF-8 content, and is outside what this narrow, length-prefix-reading
pushdown mechanism needs to distinguish for a `BLOB` (`octet_length` only,
by definition) or for the common case of ASCII-heavy string columns.

### 2.5 `scalar_function_pushdown.test`, walked in full

This is the largest of the four main test files, and its opening comment states
precisely the shared-infrastructure hazard §1.5 already introduced from the
other direction:

```sql
require parquet

# str has storage_index=1. This tests cases where column_index (i.e. 0 in
# SELECT strlen(str)) is different from storage_index.
CREATE TABLE pep_test (before INTEGER, str VARCHAR, after INTEGER);
INSERT INTO pep_test VALUES (0, 'Hello', 0), (1, 'Hi', 1), (2, 'Hey', 2);
COPY (SELECT * FROM pep_test) TO '{TEST_DIR}/pe-pushdown.parquet';
```

`str` is the file's *second* column (storage index 1), but `SELECT
strlen(str) FROM '...'` only ever projects `str` — so its `ProjectionIndex`
within the scan's own output is 0, distinct from its storage index of 1.
Every scenario below is, underneath, also implicitly re-testing that
`StorageIndex`/`ProjectionIndex` distinction the sibling test in §1.5 makes
explicit.

**Basic pushdown, plain and aliased — both succeed:**

```sql
EXPLAIN SELECT strlen(str) FROM '{TEST_DIR}/pe-pushdown.parquet';
----
physical_plan	<!REGEX>:.*PROJECTION.*

SELECT strlen(str) FROM '{TEST_DIR}/pe-pushdown.parquet' ORDER BY strlen(str);
----
2
3
5

EXPLAIN SELECT strlen(str) AS l FROM '{TEST_DIR}/pe-pushdown.parquet';
----
physical_plan	<!REGEX>:.*PROJECTION.*
```

**A filter on the exact same expression does not block pushdown** — because,
unlike `CastCollect`'s `top_level_casts` restriction, `ScalarFnCollect`
treats a matching `strlen(str)` occurring inside a `WHERE` clause as just
another consistent sighting of the same candidate, not as a disqualifying
"different use":

```sql
EXPLAIN SELECT strlen(str) FROM '{TEST_DIR}/pe-pushdown.parquet' WHERE strlen(str) > 2;
----
physical_plan	<REGEX>:.*PROJECTION.*
```

Wait — read this one carefully: the `EXPLAIN` here actually shows
`<REGEX>:.*PROJECTION.*`, i.e. pushdown is **declined**. This looks, at
first glance, like it contradicts the previous paragraph, but it doesn't —
the reason is the Parquet-side guard from §2.3, not the optimizer-side
conflict tracker: `WHERE strlen(str) > 2` gets absorbed by `FILTER_PUSHDOWN`
(#6) into the scan's own `table_filters` before this pass ever runs, and
`ParquetScanFunction::ProjectionExpressionPushdown`'s own `table_filters.HasFilter(input.
column_index)` guard (§2.3) then explicitly declines the pushdown for
exactly this reason — the reader itself refuses to retype a column that a
`table_filters` entry still needs to inspect in its original decoded form.
The test's own subsequent variations confirm the query still computes
correctly either way (`3`, `5` for `WHERE strlen(str) > 2`), just without
the pushdown's benefit. A multi-child variant of the same filter (`WHERE
strlen(str) * 2 > 4`) still finds and matches the nested `strlen(str)`
candidate correctly (the tree walk descends through the multiplication to
find it), and produces the same `3`, `5` result — the arithmetic wrapping
the call doesn't block the *optimizer's* recognition of the call, only the
*reader's* separate `table_filters` guard governs whether the pushdown
still proceeds once that recognition succeeds.

**A different function of the same column used in the filter — an explicit
conflict, at the optimizer level this time, via `Equals`:**

```sql
# strlen with filter using len, no pushdown
SELECT strlen(str) AS l FROM '{TEST_DIR}/pe-pushdown.parquet'
WHERE len(str) * 2 > 4
ORDER BY strlen(str);
----
3
5

# explain: len() in WHERE conflicts with strlen() in SELECT, no pushdown
EXPLAIN SELECT strlen(str) FROM '{TEST_DIR}/pe-pushdown.parquet' WHERE len(str) > 2;
----
physical_plan	<REGEX>:.*PROJECTION.*
```

`len` and `strlen` are DuckDB's own distinct catalog entries (not a plain
alias of one to the other at the expression-tree level, whatever their
runtime semantics happen to coincide on), so `Expression::Equals` correctly
reports them as different expressions, and the second sighting nulls the
column's candidate entry (§2.1) — this time it genuinely is the *optimizer's*
conflict tracker declining the pushdown, not the reader's `table_filters`
guard, since a comparison like `len(str) > 2` is a single-column,
constant-comparison predicate that *is* representable as a `table_filters`
entry and would ordinarily have vanished from the tree the same way
`strlen(str) > 2` did above — but by the time it's absorbed, `str` has
already been visited via `len(str)`'s own single-child-function shape
during `FindGetsAndProjections`'s earlier full-tree walk in a prior
invocation context, registering the conflict regardless.

**Bare column usage anywhere blocks pushdown of a function of that same
column, exactly as in the cast pass:**

```sql
# strlen with bare str: no pushdown
SELECT str, strlen(str) AS l FROM '{TEST_DIR}/pe-pushdown.parquet'
WHERE strlen(str) > 3 ORDER BY str;
----
Hello	5

EXPLAIN SELECT str, strlen(str) FROM '{TEST_DIR}/pe-pushdown.parquet';
----
physical_plan	<REGEX>:.*PROJECTION.*
```

**A function this reader simply doesn't recognize by name never pushes,
even alone:**

```sql
# pushdown of prefix() not supported
SELECT prefix(str, 'H') FROM '{TEST_DIR}/pe-pushdown.parquet' ORDER BY str;
----
true
true
true
```

**Multi-argument uses of the tracked column are conflicts, regardless of
which specific function is involved** — this is the `expr.GetChildren().
size() != 1` branch of `ScalarFnCollect`'s matcher (§2.1) descending into
the multi-arg call's children and finding the bare `str` reference beneath
it:

```sql
# str as part of multi-arg ||, conflict
SELECT strlen(str) AS l, str || '!' AS s FROM '{TEST_DIR}/pe-pushdown.parquet' ORDER BY l;
----
2	Hi!
3	Hey!
5	Hello!

EXPLAIN SELECT strlen(str), str || '!' FROM '{TEST_DIR}/pe-pushdown.parquet';
----
physical_plan	<REGEX>:.*PROJECTION.*

# strlen() and a different multi-arg use of str (substr): no pushdown
SELECT strlen(str) AS l, substr(str, 1, 1) AS c FROM '{TEST_DIR}/pe-pushdown.parquet' ORDER BY l;
----
2	H
3	H
5	H
```

and the same principle applies to multi-arg uses appearing only in a
`WHERE` clause, alongside an otherwise-eligible `strlen(str)` in the
`SELECT` list — proving the "only usage of `str` in `SELECT`" framing some
of the test's own comments use is really shorthand for "only single-arg
usage anywhere in the plan," not literally "only in the `SELECT` clause":

```sql
# pushdown of prefix() not supported as a complex filter, so WHERE is not pushed
# down. Although this is the only usage of str in SELECT, we can't push strlen
# down as there is usage in WHERE. This also tests functions with multiple
# arguments using "str" inside
SELECT strlen(str) FROM '{TEST_DIR}/pe-pushdown.parquet'
WHERE prefix("str", 'H') > 0
ORDER BY strlen(str);
----
2
3
5

EXPLAIN SELECT strlen(str) FROM '{TEST_DIR}/pe-pushdown.parquet' WHERE prefix("str", 'H') > 0;
----
physical_plan	<REGEX>:.*PROJECTION.*
```

**A predicate that fully vanishes into `table_filters` — the mirror image of
the earlier `strlen(str) > 2` result — genuinely does let pushdown succeed,
because the predicate here doesn't reference `str` in any function shape at
all, single- or multi-arg; it's a plain constant comparison directly on the
bare column, exactly the shape `table_filters` was built for, and it never
gives `ScalarFnCollect` anything to register a conflict against in the
first place:**

```sql
# constant comparison is pushed down as a complex filter so WHERE disappears.
# This is only usage of str in SELECT so we push it down
EXPLAIN SELECT strlen(str) FROM '{TEST_DIR}/pe-pushdown.parquet' WHERE str != '';
----
physical_plan	<!REGEX>:.*PROJECTION.*
```

Contrasting this directly against the earlier `WHERE strlen(str) > 2` case
(declined) and this `WHERE str != ''` case (succeeds) is the cleanest single
piece of evidence in the whole test file for the two-layer guard structure
this pass has: a bare-column predicate absorbed into `table_filters` leaves
nothing behind for either the optimizer's conflict tracker *or* the
reader's own `HasFilter` guard to trip on; a *function-of-the-column*
predicate absorbed into `table_filters` still trips the reader's `HasFilter`
guard specifically, because that guard checks by column index, not by
whether the filter happened to reference the column through a function call
or directly.

**A `ConstantFilter` directly equating the column also blocks pushdown, via
the same `HasFilter` mechanism:**

```sql
EXPLAIN SELECT strlen(str) FROM '{TEST_DIR}/pe-pushdown.parquet' WHERE str = 'Hello';
----
physical_plan	<REGEX>:.*PROJECTION.*
```

This one is worth contrasting against the `str != ''` case immediately
above it: `str = 'Hello'` *does* block pushdown, `str != ''` does *not* —
even though both are bare-column constant comparisons that vanish into
`table_filters` in the same way. The difference is what the reader's own
`HasFilter(input.column_index)` guard is actually checking: it fires
whenever *any* `TableFilterSet` entry exists for that column index at all,
regardless of comparison type — so the real distinguishing fact here isn't
about `=` versus `!=` in the abstract, it's that `str = 'Hello'`, a genuine
equality `ConstantFilter`, is recorded in `table_filters` for `str`'s column
index the way any absorbed predicate is, and `HasFilter` correctly reports
it; the `str != ''` case earlier in this same file evidently does **not**
leave a `table_filters` entry the guard detects (whether because inequality
against the empty string doesn't get built into a `ConstantFilter` at all
under this data, or because it's consumed via a different mechanism this
chapter's source-reading did not fully trace) — this is exactly the kind
of edge case worth stating honestly rather than overclaiming: the two
results are real and reproducible from the test file as written, and the
`HasFilter`-triggers-on-any-recorded-filter mechanism is confirmed directly
from source, but this book did not trace far enough into `FilterCombiner`'s
`!=`-specific handling to state with certainty why `str != ''` avoids
tripping it while `str = 'Hello'` does not.

**Nested struct field extraction never pushes, confirming the
`IsPushdownExtract` guard (§2.3) directly:**

```sql
COPY (SELECT {'x': 'a', 'y': 'b'} s, 'hello' v)
TO '{TEST_DIR}/scalar-pushdown-nested-2.parquet' (FORMAT PARQUET, PARQUET_VERSION v1);

# pushdown-extract columns e.g s.x are not pushed down
EXPLAIN SELECT strlen(s.x) FROM '{TEST_DIR}/scalar-pushdown-nested-2.parquet'
----
physical_plan	<REGEX>:.*PROJECTION.*

SELECT strlen(s.x) FROM '{TEST_DIR}/scalar-pushdown-nested-2.parquet';
----
1
```

while an adjacent plain top-level column in the *same* file, `v`, pushes
without issue (`strlen(v)` earlier in the file against a similarly-shaped
nested-struct file, matching `<!REGEX>:.*PROJECTION.*`) — the guard is
specifically about *extraction paths into* a nested value, not about
"anything near a struct column."

**Unsupported multi-file / encoding case:**

```sql
COPY (SELECT 'a'::VARCHAR AS s) TO '{TEST_DIR}/00_scalar_fn_plain.parquet' (FORMAT PARQUET, PARQUET_VERSION v1);
COPY (SELECT range::VARCHAR AS s FROM range(30))
TO '{TEST_DIR}/99_scalar_fn_delta.parquet' (FORMAT PARQUET, PARQUET_VERSION v2, DICTIONARY_SIZE_LIMIT 1);

EXPLAIN SELECT strlen(s)
FROM read_parquet(['{TEST_DIR}/00_scalar_fn_plain.parquet', '{TEST_DIR}/99_scalar_fn_delta.parquet']);
----
physical_plan	<REGEX>:.*PROJECTION.*
```

As §2.3 states, the guard actually responsible for this result, per the
order the checks appear in source, is the multi-file restriction
(`bind_data.file_list->GetExpandResult() == FileExpandResult::MULTIPLE_FILES`)
— any `read_parquet([...])` list with more than one file is declined
outright, before the per-row-group encoding scan for `DELTA_LENGTH_BYTE_ARRAY`
ever runs. The second file's `DICTIONARY_SIZE_LIMIT 1` (forcing a non-plain
encoding once the dictionary overflows) is there to make the test's intent
about encoding safety visible in the file names and comment even though the
multi-file guard alone would already suffice to explain the observed
`EXPLAIN` result for this specific query — both guards exist in source and
are each independently real, but this particular test exercises the
coarser of the two first.

**Pushdown alongside an unrelated filter on a *different* column succeeds
without issue**, confirming the guards are scoped strictly per tracked
column, not per-`Get`:

```sql
COPY (SELECT 'hello world ' || r AS s, r FROM range(3) t(r)) TO '{TEST_DIR}/scalar-pushdown-filter.parquet';

SELECT strlen(s) FROM '{TEST_DIR}/scalar-pushdown-filter.parquet' WHERE r = 2;
----
13
```

`r`'s own filter and `table_filters` entry are on `r`'s column index, not
`s`'s — `ParquetScanFunction::ProjectionExpressionPushdown`'s `HasFilter` check is
indexed by `input.column_index`, so it never sees `r`'s filter at all when
evaluating whether `s` is eligible.

**A function of a virtual column.** The file ends with `SELECT strlen(s), strlen(filename) > 0 FROM
read_parquet(...)`, which must return `5  true`. §2.2 explains what it
guards.

### 2.6 `scalar_function_pushdown_limit.test`

This file is short and exists to exercise exactly one shape — the `LOGICAL_
LIMIT`-between-projection-and-`Get` carve-out already introduced from the
`TYPE_PUSHDOWN` side in §1.1, confirmed here to apply identically to
`SCALAR_FN_PUSHDOWN` because both passes call the *same*
`FindGetsAndProjections` function:

```sql
require parquet

CREATE TABLE pep_test (str VARCHAR);
INSERT INTO pep_test VALUES ('Hello'), ('Hi'), ('Hey');
COPY (SELECT * FROM pep_test) TO '{TEST_DIR}/pe-pushdown-limit.parquet';
CREATE VIEW pe AS SELECT str FROM '{TEST_DIR}/pe-pushdown-limit.parquet';

EXPLAIN (FORMAT json) SELECT strlen(str) FROM pe LIMIT 1;
----
physical_plan	<!REGEX>:.*PROJECTION.*
```

Without the `type_pushdown.cpp:67-71` special case, this query would fail
to push down purely because of an accident of *when* it's checked, not
because of anything semantically wrong with pushing it: `LIMIT_PUSHDOWN`
(pass #29, run at `optimizer.cpp:442`, fifteen passes before this one (#44)) moves
the `LIMIT` below `pe`'s own passthrough view projection wherever doing so
is safe (a plain projection cannot change row count), leaving a
`LogicalLimit` sitting directly between that passthrough projection and the
`LogicalGet` — precisely the shape `FindGetsAndProjections`'s dedicated
`if (child.type == LogicalOperatorType::LOGICAL_LIMIT)` branch exists to
still recognize. Without it, `get` would remain `nullptr` for this query,
`projections.emplace(...)` would never run, and `strlen(str)` would never be
recognized as reaching a pushdown-eligible `Get` at all — not because the
`Get` stopped being eligible, but because an *earlier, unrelated* pass
happened to insert one more node into the chain this pass's shape-matcher
has to walk through. This is a small, self-contained, source-confirmed
illustration of exactly the general argument §3.2 makes at the scale of the
whole pipeline: these two passes' correctness and effectiveness both depend
on the *exact* shape of the tree other, earlier passes leave behind, which
is only ever fully settled once every earlier pass has already run.

---

## Part III — Why two separate, late passes, and why this order

### 3.1 Why not one general "expression pushdown" pass instead of two

It would be reasonable to ask why `TYPE_PUSHDOWN` and `SCALAR_FN_PUSHDOWN`
are two `OptimizerType`s at all, given how much code they share (all of
`type_pushdown.cpp`'s non-`Cast*` machinery). The answer sits entirely in
what each one's `Collect`/`Replace` pair matches, and in what the *reader*
side is willing to do with what it's handed:

- **A cast changes an expression's declared type while (ideally) preserving
  its logical value** — `column00::UTINYINT` still "means" the same
  quantity `column00` meant, just represented (and range-checked) as a
  narrower integer type. Pushing it into CSV works because CSV's decode
  step is, structurally, "parse text into *some* type" regardless — there's
  no additional computation involved beyond picking which type to target.
- **A scalar function computes something genuinely new from a column's
  value** — `strlen(str)` is not "`str`, represented differently"; it's a
  different, derived value entirely. Pushing it into Parquet only works
  because Parquet's on-disk format happens to expose exactly the one piece
  of metadata (`BYTE_LENGTH`, read straight from a length-prefix) two
  specific functions need, without decoding the value's actual content —
  which is a fundamentally different kind of trick than "choose the parse
  target type," and it generalizes to essentially no other scalar function
  short of one that also only needs a value's stored length or a similarly
  cheap piece of per-value metadata.

Because the two mechanisms are asking such different questions of the
reader, and because (per §1.3/§2.3) the two readers that implement either
callback in this checkout each implement *exactly one* of the two and
explicitly reject the other's shape, a single merged pass would gain
nothing structurally — it would still need two disjoint matchers and two
disjoint sets of reader-side semantics, just filed under one name instead
of two. Keeping them separate buys the ordinary benefits every named pass in
this pipeline gets for free (§7.2 of the overview): independent profiling,
independent `disabled_optimizers` toggling, and — as the next section
argues — the ability to sequence one strictly before the other.

### 3.2 Why these two, specifically, run dead last

Every one of the four correctness/feasibility facts this file has traced
back to source converges on the same conclusion: `TYPE_PUSHDOWN` and
`SCALAR_FN_PUSHDOWN` are structurally dependent on the tree already being in
its *final* shape, in a way `FILTER_PUSHDOWN` and most of the other ~45
passes are not.

1. **They depend on `FILTER_PUSHDOWN` having already finished.**
   `ReachesPushdownGet`'s explicit `LOGICAL_FILTER` chain-break (§1.2) and
   Parquet's `table_filters.HasFilter` guard (§2.3, with its own comment
   stating the dependency outright) both require that whatever filter
   pushdown was ever going to do to a given predicate has already happened.
   A filter that's going to end up fully absorbed into `table_filters`
   needs to have actually been absorbed *before* these passes check whether
   a `LogicalFilter` node still blocks their chain-walk; a filter that's
   going to survive as a genuine `LogicalFilter` needs to already be sitting
   there, correctly blocking the chain, rather than these passes trying to
   reason about a not-yet-settled predicate. There is no way to interleave
   this correctly — it requires `FILTER_PUSHDOWN`'s single, one-shot pass
   (#6) to be fully finished, tree-wide, before either of these passes so
   much as starts.
2. **They depend on every other structural rewrite that can change which
   operator sits directly above a `Get` having already run.** `LIMIT_
   PUSHDOWN`'s carve-out (§2.6) is the cleanest single proof of this: an
   earlier pass (#29) can insert a new operator between a passthrough
   projection and its `Get`, and `FindGetsAndProjections`'s shape-matcher
   has to be hand-written to tolerate that one specific insertion or lose
   the opportunity outright. `PROJECTION_PULLUP` (#15), `UNNEST_REWRITER`
   (#22), `COMMON_SUBPLAN` (#28), and `JOIN_ORDER` (#17) can each, in
   principle, change whether a projection ends up sitting directly and
   cleanly above a `Get` at all — running `TYPE_PUSHDOWN`/`SCALAR_FN_
   PUSHDOWN` any earlier risks either missing pushdown opportunities a later
   structural pass would have made available, or reasoning about a
   projection/`Get` adjacency that a later pass is about to destroy.
3. **`UNUSED_COLUMNS`/`COLUMN_LIFETIME` (#23/#26, and their second
   occurrences at #38/#39) must have already fixed each `Get`'s final
   output-column list**, because `ProjectionIndex` — the index both passes'
   entire bookkeeping is keyed on — is defined relative to that list
   (§1.1). Running before column pruning has stabilized would mean
   reasoning about indices that are about to shift underneath the pass.
4. **`TYPE_PUSHDOWN` (#44) must run before `SCALAR_FN_PUSHDOWN` (#45)**,
   for a narrower, purely internal reason: `ScalarFnCollect`/`Replace`
   explicitly decline to match through a `CAST` node, deferring to "the
   dedicated cast pushdown" (§2.1). If a column's only wrapping expression
   is `fn(CAST(col))` — a function applied to a cast, rather than directly
   to a bare column — `SCALAR_FN_PUSHDOWN`'s matcher never even considers
   it, because its single-child check finds a `CAST` node, not a bare
   `BOUND_COLUMN_REF`, immediately underneath `fn`. Running `TYPE_PUSHDOWN`
   first gives the cast a chance to be resolved away (rewritten to a bare,
   now-correctly-typed column reference) before `SCALAR_FN_PUSHDOWN` ever
   looks at that position in the tree — at which point `fn(col)` *is* a
   directly matchable shape. Reversing the order would silently forfeit
   this composition, with no error and no observable symptom beyond a
   missed optimization.

Point 4 is a small-scale, two-pass illustration of exactly the general rule
§7.2 of the chapter overview states about the whole forty-pass pipeline:
*"If pass #30 makes a change that would have unlocked a rewrite pass #5
already ran past, pass #5 does not get a second chance."* Here the distance
is one pass, not twenty-five, and the consequence is a missed pushdown
rather than a missed rewrite of some other kind — but the mechanism is
identical: a fixed, one-shot, no-fixpoint sequence means the *relative*
order of two closely related passes is exactly as load-bearing as the
overall order of the whole pipeline, and DuckDB resolves it here the same
way it resolves it everywhere else — by hand-choosing which of the two
comes first and hard-coding that choice into `RunBuiltInOptimizers`, rather
than by building any machinery that could discover the right order
dynamically.

### 3.3 What separates these two passes from `FILTER_PUSHDOWN`, restated plainly

`FILTER_PUSHDOWN` (pass #6, `07-filter-pullup-and-pushdown.md` in full) is a
recursive, per-operator-type dispatcher — seventeen distinct handler files,
each one encoding a different operator's own semantics for "which filters
may cross me, and under what proof obligation" (a `FULL OUTER JOIN`'s
coalesce reasoning, a `MARK` join's null-safety downgrade, a `SEMI`/`ANTI`
join's asymmetric existence semantics, and so on). It runs early (#6)
precisely because almost everything downstream benefits from filters having
already reached the scan — smaller intermediate cardinalities feed better
join-order decisions, cheaper column-lifetime analysis, and so on.

`TYPE_PUSHDOWN`/`SCALAR_FN_PUSHDOWN` are, by contrast, a single global
tree-wide conflict-tracking walk with no operator-type dispatch at all — the
only "operator semantics" either one needs to know about are "is this a
`LogicalGet` with the right callback," "is this a passthrough
`LogicalProjection`," and (one special case) "is this a `LogicalLimit`
directly above a `LogicalGet`." They run last (#44/#45) precisely because
they are not trying to shrink the amount of data flowing through the plan
the way filter pushdown does — they're trying to avoid a second pass of
work over data that's already going to be scanned in full, by asking the
reader to produce the *final* value directly. That's only a sound question
to ask once the plan's shape, the scan's own column list, and every
predicate's fate have already been settled by everything else in this
chapter — which is exactly why these two passes sit where they sit: the
very last two structural rewrites this book's optimizer chapter covers,
immediately before the extension hooks close the pipeline out.
