# 7.17 `LIMIT_PUSHDOWN`, `ROW_GROUP_PRUNER`, and `SAMPLING_PUSHDOWN` — three ways to make a scan see fewer rows

Chapter 7's overview (`00-overview.md`, §7.4) lists forty-five `RunOptimizer` call
sites in exact pipeline order. Three of them sit
back-to-back, around the thirtieth slot, right after the structural join/aggregate/CSE machinery has settled
and right before `TOP_N` fuses `ORDER BY` + `LIMIT` into a single operator:

```
29. LIMIT_PUSHDOWN       — optimizer.cpp:442, src/optimizer/limit_pushdown.cpp
30. ROW_GROUP_PRUNER      — optimizer.cpp:447, src/optimizer/row_group_pruner.cpp
31. SAMPLING_PUSHDOWN     — optimizer.cpp:453, src/optimizer/sampling_pushdown.cpp
32. TOP_N                 — optimizer.cpp:459, src/optimizer/topn_optimizer.cpp
```

That ordering is not an accident of alphabetization, and this file's first job
is to explain why it's the only ordering that makes sense. All three passes
this file covers are, in one way or another, about **making a scan produce
fewer rows, or produce its rows in a more useful order, before anything
downstream has to touch them** — but they attack the problem from three
genuinely different angles: `LIMIT_PUSHDOWN` is a purely syntactic tree
rewrite that never looks at a single byte of table data; `ROW_GROUP_PRUNER` is
a heuristic bet on physical data layout that consults real zonemaps at
*execution* time, not at optimization time; and `SAMPLING_PUSHDOWN` turns a
logical `SAMPLE`/`TABLESAMPLE` operator into an annotation the storage-layer
scan itself can act on, for one specific sampling method only. All three are
small files — `limit_pushdown.cpp` is 53 lines, `sampling_pushdown.cpp` is 58,
and `row_group_pruner.cpp` is 212 — and all three reward being read in full,
which is what this file does, alongside every real test case the source tree
ships for them.

A brief note on this file's test citations before starting: none of the test
cases below use the two-`query I nosort <label>`-blocks convention (a
byte-identical paired-`EXPLAIN` proof of a rewrite) that recurs elsewhere in
this chapter's test suite — every test here instead asserts a single `EXPLAIN`
against a `<REGEX>` pattern, so that convention doesn't need re-explaining in
this particular file.

## 7.17.1 `LIMIT_PUSHDOWN` — a narrow, purely syntactic swap

**One-sentence purpose:** move a `LogicalLimit` that sits directly above a
plain `LogicalProjection` to sit directly *below* it instead, when doing so is
provably safe and probably cheap enough to be worth it — because a projection
cannot change how many rows flow through it, only what each row looks like.

### The whole pass, read in full

`limit_pushdown.cpp` is short enough to reproduce as a whole rather than in
fragments, and it's worth doing so because every one of its four eligibility
conditions is visible in fourteen lines:

```cpp
// src/optimizer/limit_pushdown.cpp:8
bool LimitPushdown::CanOptimize(duckdb::LogicalOperator &op) {
	if (op.type == LogicalOperatorType::LOGICAL_LIMIT &&
	    op.children[0]->type == LogicalOperatorType::LOGICAL_PROJECTION) {
		auto &limit = op.Cast<LogicalLimit>();

		if (limit.offset_val.Type() == LimitNodeType::EXPRESSION_PERCENTAGE ||
		    limit.offset_val.Type() == LimitNodeType::EXPRESSION_VALUE) {
			// Offset cannot be an expression
			return false;
		}

		if (limit.limit_val.Type() == LimitNodeType::CONSTANT_VALUE && limit.limit_val.GetConstantValue() < 8192) {
			// Push down only when limit value is smaller than 8192.
			// when physical_limit is introduced, it will end a parallel pipeline
			// restrict the limit value to be small so that remaining operations run fast without parallelization.
			if (limit.offset_val.Type() == LimitNodeType::CONSTANT_VALUE && limit.offset_val.GetConstantValue() > 0) {
				// If we push the limit below the projection, the offset rows are discarded before the
				// projection runs, so a volatile expression produces different values for the rows we keep.
				auto &projection = op.children[0]->Cast<LogicalProjection>();
				for (auto &expr : projection.expressions) {
					if (expr->IsVolatile()) {
						return false;
					}
				}
			}
			return true;
		}
	}
	return false;
}
```

Four conditions, in the order the code checks them:

1. **Shape**: the operator has to literally be a `LogicalLimit` whose
   `children[0]` is a `LogicalProjection` — nothing else qualifies. An
   `ORDER BY` sitting between the `LIMIT` and the projection blocks this
   immediately, which is exactly why Query A (Chapter 7's running example,
   `... GROUP BY o.custkey ORDER BY total DESC LIMIT 10`) is a documented
   no-op for this pass in the overview's §7.7 — `LogicalLimit`'s child there
   is `LogicalOrder` (soon to be fused into `LogicalTopN`), not a projection.
2. **No expression offset**: `OFFSET` is allowed, but only as a
   compile-time-constant row count (`LimitNodeType::CONSTANT_VALUE` or
   `UNSET`) — an `OFFSET` computed from an expression (`EXPRESSION_VALUE`/
   `EXPRESSION_PERCENTAGE`) blocks the rewrite outright, presumably because
   the pass has no static value to reason about at all for the volatility
   check that follows.
3. **A small, constant `LIMIT`**: `limit_val` must be a compile-time constant
   *and* strictly less than `8192`. `< 8192`, not `<= 8192` — a `LIMIT 8192`
   query stays exactly where it started (§7.17.1's test walkthrough below
   confirms this at the boundary with `LIMIT 8192` over `range(8193)`). The
   inline comment gives the actual engineering reason, not just "it's
   arbitrary": `PhysicalLimit` is a pipeline-breaking operator — it has to
   synchronize across every parallel worker to agree on exactly which global
   rows survive — so moving it earlier in the tree moves that
   synchronization point earlier too. That's fine when the limit is small
   (whatever runs after the limit, now on the far side of a barrier, only
   ever sees a small number of rows and doesn't need to be parallelized to be
   fast), but pushing an unbounded-feeling limit below a projection would
   mean re-synchronizing on a potentially large row count in a spot the
   planner didn't originally put a pipeline breaker, for no corresponding
   benefit.
4. **Volatility, but only when `OFFSET > 0`**: if the constant offset is
   greater than zero, every expression in the projection being pushed under
   is checked with `IsVolatile()`, and *any* volatile expression vetoes the
   whole rewrite. This is the subtlest of the four conditions and the one
   the test suite specifically exercises (below): pushing the limit below
   the projection means the `OFFSET` rows get discarded *before* the
   projection ever computes anything for them, instead of *after*. For a
   pure expression that makes no observable difference. For `nextval('seq')`
   it changes which sequence values the surviving rows actually see, because
   in the un-pushed plan the discarded rows still flow through the
   projection (and still consume sequence values) before the limit throws
   them away, while in the pushed plan they never reach the projection at
   all and never consume a sequence value.

### The rewrite itself, and why it's actually a *local* loop, not one-shot

```cpp
// src/optimizer/limit_pushdown.cpp:39
unique_ptr<LogicalOperator> LimitPushdown::Optimize(unique_ptr<LogicalOperator> op) {
	if (CanOptimize(*op)) {
		auto projection = std::move(op->children[0]);
		op->children[0] = std::move(projection->children[0]);
		projection->SetEstimatedCardinality(op->estimated_cardinality);
		projection->children[0] = std::move(op);
		swap(projection, op);
	}
	for (auto &child : op->children) {
		child = Optimize(std::move(child));
	}
	return op;
}
```

The four-line body inside the `if` is a standard tree rotation: the
projection is lifted out, the limit's new child becomes whatever used to sit
*under* the projection, the projection's new child becomes the limit itself,
and the two nodes swap places in the variable that will be returned. What's
worth noticing is what happens *next*: the function falls through to its
own recursive loop over `op->children`, and by this point `op` is the
projection (post-swap) and its only child is the limit, whose only child is
whatever used to be two levels down. If that turns out to be *another* plain
projection, `CanOptimize` fires again on the very next recursive call, and the
limit migrates one more level down. This is the same architectural point the
overview makes about `ExpressionRewriter`'s inner rule loop in §7.5 — the
plan-level pipeline in `optimizer.cpp` only invokes `LIMIT_PUSHDOWN` once as
pass #29, but *within* that one invocation, `LimitPushdown::Optimize`'s own
recursion is perfectly happy to walk a `LIMIT` down through an arbitrarily
long stack of consecutive plain projections in one pass. This is exactly what
the test suite's "recursive limit pushdown" case demonstrates, below.

Also worth noting from the eligibility check and the rewrite together: there
is no branch anywhere in this file for `LogicalSetOperation` (`UNION`,
`UNION ALL`, `INTERSECT`, `EXCEPT`), no branch for `LogicalFilter`, and no
branch for anything except `LogicalProjection`. **`LIMIT_PUSHDOWN` does not
push a `LIMIT` through a `UNION ALL`, and it does not push through a
`FILTER`.** This is worth stating explicitly because it's an intuitive guess
— "surely each branch of a `LIMIT`-ed `UNION ALL` only needs to produce up to
N rows" is a real optimization some other systems perform — but it is simply
not what this file's source does. `CanOptimize`'s very first check requires
`op.children[0]->type == LogicalOperatorType::LOGICAL_PROJECTION`; anything
else, including a set operation, fails that check on the first line and the
function returns `false` before reaching any of the other three conditions.
If a `UNION ALL` branch happens to end in a projection immediately under a
plain `SELECT`'s outer projection, that inner branch's own limit pushdown (if
any exists in the query) is handled independently wherever it occurs in that
branch's own subtree — but there is no code path in this file that reasons
about a `LIMIT` sitting above a `UNION` at all.

### The test suite, case by case

`test/optimizer/limit_pushdown.test` uses `PRAGMA explain_output =
OPTIMIZED_ONLY` and inspects the `logical_opt` column, so every `<REGEX>`
below is being matched against the post-optimizer logical tree, top to bottom.

**Setup:**
```sql
CREATE TABLE integers(i INTEGER, j INTEGER);
INSERT INTO integers VALUES (1,1), (2,2), (3, 3), (4,4);
```

**Case 1 — the basic swap:**
```sql
EXPLAIN SELECT i FROM integers LIMIT 4
----
logical_opt	<REGEX>:.*PROJECTION.*LIMIT.*
```
The regex requires `PROJECTION` to appear *before* `LIMIT` in the rendered
tree, and `EXPLAIN` renders top-down (root first), so this is asserting the
tree is now `PROJECTION → LIMIT → ...` — the limit has moved below the
projection. `LIMIT 4` is a constant well under `8192`, there's no `OFFSET` at
all (so the volatility check never triggers), and `integers`' `i` column is a
plain column reference, not a volatile expression, so every condition in
`CanOptimize` is trivially satisfied.

**Case 2 — recursive pushdown through two stacked projections:**
```sql
EXPLAIN SELECT i FROM (SELECT i, i+1 FROM integers) LIMIT 4
----
logical_opt	<REGEX>:.*PROJECTION.*PROJECTION.*LIMIT.*
```
Two projections now separate the limit's original position from the base
scan: the outer `SELECT i` and the inner `SELECT i, i+1`. The regex requires
both `PROJECTION`s to print *before* `LIMIT`, meaning the limit has migrated
past both of them, exactly as the "the rewrite itself" discussion above
predicted from re-reading `Optimize`'s recursive fall-through.
```sql
SELECT i FROM (SELECT i, i+1 FROM integers) LIMIT 4 OFFSET 2
----
3
4
```
This immediately-following data test isn't just re-confirming the shape —
it's confirming the rewrite didn't change the *answer* while it changed the
*shape*: rows 3 and 4 (0-indexed offset 2 into the 4-row table) still come
out, whether the limit+offset is applied once at the top or, after the
rewrite, effectively pushed down alongside the recursion.

**Case 3 — `OFFSET` with no `LIMIT`: nothing to push:**
```sql
EXPLAIN SELECT i FROM integers OFFSET 4
----
logical_opt	<REGEX>:.*LIMIT.*PROJECTION.*
```
Here the regex is inverted from case 1 — `LIMIT` appears *first*, meaning it
stayed on top. This is the first condition in `CanOptimize` doing its job
silently: an `OFFSET`-only clause with no `LIMIT` count leaves `limit_val` at
`LimitNodeType::UNSET`, not `CONSTANT_VALUE`, so the `if (limit.limit_val...`
branch's own condition never even evaluates to true — `CanOptimize` falls
through to `return false` at the very end of the function. There's an
intuitive reason this is the right call even setting the source aside: an
`OFFSET` with no `LIMIT` doesn't bound how many rows the query eventually
produces at all, so there's no row-count reduction for pushing the limit
below the projection to buy — the entire rationale in condition 3's
pipeline-breaker discussion above (small limit, small unparallelized
remainder) simply doesn't apply when there's no upper bound on the remainder.

**Case 4 — `LIMIT` and `OFFSET` together, ordinary case:**
```sql
EXPLAIN SELECT i FROM integers LIMIT 4 OFFSET 2
----
logical_opt	<REGEX>:.*PROJECTION.*LIMIT.*
```
```sql
SELECT i FROM integers LIMIT 4 OFFSET 2
----
3
4
```
`OFFSET 2` is a constant `> 0`, so the volatility check does run this time —
but `i` is a plain column reference, not volatile, so the pushdown proceeds
exactly as case 1 did. Only two rows come back (the table only has four rows
total and `OFFSET 2 LIMIT 4` can supply at most two of them), which matches
what the un-rewritten query would have produced too — same point as case 2's
data check, correctness preserved across the rewrite.

**Case 5 — the `8192` boundary, hit exactly:**
```sql
EXPLAIN SELECT i + 1 FROM range(8193) t(i) LIMIT 8192
----
logical_opt	<REGEX>:.*LIMIT.*PROJECTION.*
```
`range(8193)` supplies exactly one more row than the limit asks for, and the
limit itself is `8192` — landing precisely on the threshold `CanOptimize`
checks with a strict `<`. `8192` is not `< 8192`, so the whole condition-3
branch is skipped and `CanOptimize` returns `false` at the bottom of the
function, leaving `LIMIT` on top of `PROJECTION` exactly as case 3 did. This
is the test suite proving the boundary is inclusive-on-the-refusing-side —
`LIMIT 8191` over the same query would push down; `LIMIT 8192` does not.

**Case 6 and 7 — the volatility guard, proven both structurally and by data:**
```sql
CREATE SEQUENCE s START 1;

EXPLAIN SELECT nextval('s') AS v FROM integers LIMIT 1 OFFSET 1
----
logical_opt	<REGEX>:.*LIMIT.*PROJECTION.*
```
`OFFSET 1` is a constant `> 0`, and the single projected expression is
`nextval('s')`, which `Expression::IsVolatile()` reports as volatile — so
`CanOptimize`'s inner loop finds a volatile expression and returns `false`
immediately, and `LIMIT` stays above `PROJECTION`. The very next statement
turns this structural fact into an observable-correctness proof:
```sql
CREATE TABLE offset_result AS SELECT nextval('s') AS v FROM integers LIMIT 1 OFFSET 1;

SELECT v FROM offset_result
----
2
```
`integers` has four rows, so an un-pushed plan calls `nextval('s')` once for
the first row (consuming sequence value `1`, immediately discarded by the
`OFFSET`), then once more for the second row (consuming sequence value `2`,
which survives past the `OFFSET` and satisfies the `LIMIT 1`) — hence the
observed `2`. Had the guard been missing and the limit been pushed below the
projection anyway, the plan would instead discard the *raw, pre-projection*
first row via the pushed-down limit/offset before `nextval('s')` ever runs
for it, and the surviving row would compute `nextval('s')` for the first
time — yielding `1` instead of `2`. The guard exists precisely to keep this
kind of side-effect-visible-through-`OFFSET` case correct, and the test
suite checks the actual returned value, not just the plan shape, to prove it.

### Where this sits relative to the rest of the pipeline

`LIMIT_PUSHDOWN` runs at position #29, one slot before `ROW_GROUP_PRUNER`
(#30) and three slots before `TOP_N` (#32). The ordering matters in one
concrete way: because `TOP_N` hasn't run yet, a `LogicalLimit` whose child
really is a plain `LogicalOrder` (as opposed to a plain `LogicalProjection`)
is still a `LogicalLimit`/`LogicalOrder` pair at this point in the pipeline,
not yet a fused `LogicalTopN` — which is exactly the shape `ROW_GROUP_PRUNER`,
the next pass, is looking for. `LIMIT_PUSHDOWN` itself has no interaction
with `LogicalOrder` at all (its `CanOptimize` only recognizes a
`LogicalProjection` child), so it simply does nothing for an `ORDER BY ...
LIMIT` query and lets the next two passes have their turn at the unmodified
`LIMIT`/`ORDER BY` pair.

## 7.17.2 `ROW_GROUP_PRUNER` — betting on physical layout to skip whole row groups

**One-sentence purpose:** for an `ORDER BY <column> LIMIT N` whose sort key
traces back to an actual stored column on a base-table scan, arrange for that
table's row groups to be *visited* in the zonemap order most likely to
surface the top-`N` rows early, and — when the data's physical layout
actually cooperates — prove that later row groups can be skipped
**without reading a single byte of them**, not merely reordered.

This is the most involved of the three passes in this file, and it's worth
being precise about what "involved" means here: the optimizer-side file,
`row_group_pruner.cpp`, is only 212 lines and does no more than *recognize*
the pattern and hand off a small options struct; essentially all of the
actual cleverness — the multimap of row groups keyed by zonemap bound, the
overlap-based proof that lets whole row groups be excluded, the NULL-ordering
bookkeeping — lives in `src/storage/table/row_group_reorderer.cpp`, which
runs once per query at *execution* time, not at plan time. This section
covers both halves, because neither one makes sense read in isolation.

### Recognizing the pattern: `RowGroupPruner::TryOptimize`

```cpp
// src/optimizer/row_group_pruner.cpp:28
bool RowGroupPruner::TryOptimize(LogicalOperator &op) const {
	optional_idx row_limit;
	optional_idx row_offset;

	if (op.type != LogicalOperatorType::LOGICAL_LIMIT) {
		return false;
	}

	auto &logical_limit = op.Cast<LogicalLimit>();
	GetLimitAndOffset(logical_limit, row_limit, row_offset);
	auto logical_order = FindLogicalOrder(logical_limit);
	if (!logical_order) {
		return false;
	}
```

The very first requirement is the same `LOGICAL_LIMIT` check `LimitPushdown`
starts with, but everything downstream diverges immediately.
`FindLogicalOrder` (`row_group_pruner.cpp:105`) walks down from the limit's
child through any number of intervening plain projections (exactly the
recursion `LIMIT_PUSHDOWN` itself performs, reused here for tree-walking
rather than rewriting) and requires what it lands on to be a genuine
`LogicalOrder`, with two further constraints on the *primary* (first) sort
key:

```cpp
// src/optimizer/row_group_pruner.cpp:105
optional_ptr<LogicalOrder> RowGroupPruner::FindLogicalOrder(const LogicalLimit &logical_limit) const {
	reference<LogicalOperator> current_op = *logical_limit.children[0];
	while (current_op.get().type == LogicalOperatorType::LOGICAL_PROJECTION) {
		current_op = *current_op.get().children[0];
	}

	if (current_op.get().type != LogicalOperatorType::LOGICAL_ORDER_BY) {
		return nullptr;
	}

	auto &logical_order = current_op.get().Cast<LogicalOrder>();
	auto order_column_type = logical_order.orders[0].expression->GetReturnType();
	if (!order_column_type.IsNumeric() && !order_column_type.IsTemporal() &&
	    order_column_type != LogicalType::VARCHAR) {
		return nullptr;
	}

	if (logical_order.orders[0].expression->GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
		return nullptr;
	}

	return logical_order;
}
```

Two things to notice here, both of which answer questions the task of
writing this section was specifically asked to resolve against real source
rather than guessed at:

- **Only `orders[0]`, the first `ORDER BY` key, is ever consulted.** A
  multi-key `ORDER BY a, b LIMIT N` is examined only through `a`'s lens —
  `b` never enters into whether this pass fires or how it reorders row
  groups. This makes sense once you see what the reorderer does with the
  single key it's given: sorting row groups by `a`'s zonemap bound can never
  correctly account for a secondary tiebreak on `b`, so the pass simply
  doesn't try; it treats `ORDER BY a, b` the same as `ORDER BY a` would be
  treated for the purposes of *this* pass, on the (sound) logic that `a`'s
  own zonemap-driven reordering is still valid regardless of what `b` does
  to break ties *within* a fixed value of `a`.
- **The sort key must be a plain, direct `BOUND_COLUMN_REF`** — not an
  expression, not a cast, not a function call. `ORDER BY total DESC` in
  Query A (Chapter 7's running example, where `total` is `SUM(l.extendedprice)`,
  the output of an aggregate) fails this check immediately, which the
  overview's §7.7 states as a conclusion; this is the exact line of code
  that conclusion rests on. `ORDER BY col_type`, where `col_type` is limited
  to numeric, temporal, or `VARCHAR` — `order_column_type.IsNumeric() ...`
  — also excludes, for instance, `BOOLEAN`, `BLOB`, and nested types from
  ever qualifying (a `BOOLEAN` zonemap of min/max `{false, true}` genuinely
  carries almost no ordering information to exploit anyway).

### The intervening-operator walk: reordering is allowed under more than pruning is

Once a candidate `LogicalOrder` is found, `TryOptimize` walks *down* from it
toward the base table, checking every operator along the way:

```cpp
// src/optimizer/row_group_pruner.cpp:43
// Only reorder row groups if there are no additional limit operators since they could modify the order
reference<LogicalOperator> current_op = *logical_order;
while (!current_op.get().children.empty()) {
	if (current_op.get().children.size() > 1) {
		return false;
	}

	const auto op_type = current_op.get().type;
	if (op_type == LogicalOperatorType::LOGICAL_LIMIT) {
		return false;
	}
	if (op_type == LogicalOperatorType::LOGICAL_FILTER ||
	    op_type == LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY ||
	    op_type == LogicalOperatorType::LOGICAL_DISTINCT) {
		row_limit.SetInvalid();
		row_offset.SetInvalid();
	}
	current_op = *current_op.get().children[0];
}
```

Three distinct outcomes fall out of this loop, and the distinction between
them is exactly the honesty-about-limits point this file was asked to get
right:

1. **A second `LOGICAL_LIMIT` anywhere between the order and the base table
   is an unconditional bail-out** (`return false`, the whole optimization is
   abandoned for this candidate). The comment states why: an intervening
   limit could already have decided which subset of rows survives, and
   reordering the *base table's* row-group visiting order retroactively
   would be reasoning about a row set that isn't the one the intervening
   limit actually operated on.
2. **A `LOGICAL_FILTER`, `LOGICAL_AGGREGATE_AND_GROUP_BY`, or
   `LOGICAL_DISTINCT` between the order and the scan invalidates the
   *limit/offset-based pruning*, but not the reordering.** Both
   `row_limit` and `row_offset` are set to `optional_idx::Invalid()` right
   here — meaning by the time the options struct reaches the storage layer,
   there is no `N` to try to prove "enough rows found" against — but the
   loop keeps walking, and a `LogicalGet` further down can still receive a
   *pure reordering* instruction with no limit attached. The reasoning: a
   `DISTINCT` or an intervening aggregate means the number of *rows the
   scan produces* is no longer the same as the number of *final result
   rows* the `ORDER BY ... LIMIT` cares about, so bounding the scan by `N`
   raw rows could easily undercount how many raw rows are actually needed
   to produce `N` distinct/aggregated output rows — but visiting the row
   groups whose zonemaps suggest they contain the extreme values *first* is
   still a good bet regardless, since whatever survives the filter/dedup
   will still tend to show up earlier in that visiting order than it would
   in arbitrary storage order.
3. **No such operator at all**, as in a plain `ORDER BY col LIMIT N` with
   nothing but projections in between — `row_limit`/`row_offset` survive
   intact all the way to the options struct, and the strongest form of the
   optimization (provable row-group skipping) becomes possible.

Immediately after this loop, one more condition narrows things further, and
it's checked *again* independently of the filter/aggregate/distinct check
above:

```cpp
// src/optimizer/row_group_pruner.cpp:73
if (logical_get->table_filters.HasFilters()) {
	// If there are filters, we only order the row groups but do not prune
	row_limit.SetInvalid();
	row_offset.SetInvalid();
}
```

If the base-table scan itself already carries a pushed-down `WHERE` filter
(from `FILTER_PUSHDOWN`, pass #6, which ran many passes earlier), the same
downgrade happens: reordering survives, limit-based pruning doesn't. The
reasoning is the same shape as case 2 above — with a filter present, the
number of rows a given row group *contributes to the final answer* is no
longer simply "however many rows are in it," since some fraction get
filtered out, and a fraction that varies unpredictably per row group means
the reorderer can no longer *prove* a running total of qualifying rows the
way it can when every visible row in a row group is guaranteed to count.

### Reaching the base table through the join-filter-pushdown column tracer

`FindLogicalGet` (`row_group_pruner.cpp:129`) is the one piece of this pass
that is, source-verified, a genuine reuse of machinery built for a different
job — `JOIN_FILTER_PUSHDOWN` (pass #42, `join_filter_pushdown_optimizer.cpp`)
already has a general-purpose helper,
`JoinFilterPushdownOptimizer::GetPushdownFilterTargets`, whose whole job is
tracing a column reference *down* through a chain of intervening operators
(`LOGICAL_FILTER`, `LOGICAL_ORDER_BY`, `LOGICAL_DISTINCT`,
`LOGICAL_CROSS_PRODUCT`, `LOGICAL_PROJECTION`, `LOGICAL_AGGREGATE_AND_GROUP_BY`,
set operations, and finally `LOGICAL_GET`) to find which raw, stored column a
given expression ultimately reads from — built originally so a hash join's
runtime min/max filter can be reconstructed on the probe side. `ROW_GROUP_PRUNER`
calls this exact same function, packaged as a single-column lookup, purely to
answer "does this `ORDER BY` key trace back to a real stored column, and if
so, which storage index":

```cpp
// src/optimizer/row_group_pruner.cpp:129
optional_ptr<LogicalGet> RowGroupPruner::FindLogicalGet(const LogicalOrder &logical_order,
                                                        ColumnIndex &column_index) const {
	const auto &primary_order = logical_order.orders[0];
	auto &colref = primary_order.expression->Cast<BoundColumnRefExpression>();

	JoinFilterPushdownColumn column;
	column.probe_column_index = colref.Binding();
	vector<JoinFilterPushdownColumn> columns {std::move(column)};
	vector<PushdownFilterTarget> pushdown_targets;
	JoinFilterPushdownOptimizer::GetPushdownFilterTargets(*logical_order.children[0], std::move(columns),
	                                                      pushdown_targets);

	if (pushdown_targets.empty()) {
		return nullptr;
	}

	D_ASSERT(pushdown_targets.size() == 1);
	auto &pushed_column = pushdown_targets[0].columns[0];
	if (pushed_column.mode != JoinFilterPushdownMode::RECONSTRUCT_EXPRESSION ||
	    RuntimeFilterCastUtil::RuntimeFilterUsesTryCast(pushed_column)) {
		// the sort key reaches the scan through a cast chain that does not preserve the raw column's
		// ordering (an explicit TRY_CAST, or a cast that can throw) - ordering row groups by the raw
		// column statistics would be incorrect - bail out
		return nullptr;
	}

	auto &logical_get = pushdown_targets.front().get;

	if (!logical_get.function.set_scan_order) {
		return nullptr;
	}
	// ...
}
```

Three more bail-out conditions live here, on top of everything already
checked: the traced column must reconstruct as `RECONSTRUCT_EXPRESSION`
mode (meaning the path from the raw stored value to the `ORDER BY`
expression is a lossless, order-preserving chain — no `TRY_CAST`, no
throwing cast — because a cast that can silently null out or reorder values
would make "sort row groups by the raw column's zonemap" produce the wrong
answer for the *cast* value the query actually orders by); and, separately,
the underlying table function has to actually support scan reordering at
all, gated by a single boolean field, `set_scan_order`, on `TableFunction`.
A direct search of the source tree confirms only one registration site sets
this field:

```cpp
// src/function/table/table_scan.cpp:1313
scan_function.set_scan_order = SetScanOrder;
```

**This means `ROW_GROUP_PRUNER` only ever applies to DuckDB's own native,
persistent table storage** (the row-group-based storage this book's Chapter
17 documents) — not to a Parquet scan, not to a CSV read, not to any other
registered table function, in this checkout. A `zonemap`-consulting reader
like Parquet has its own, separate min/max-based row-group/page skipping
mechanism (a page index, in Parquet's case), but it does not participate in
*this specific* optimizer pass's reordering machinery, because it never
registers `set_scan_order`.

### The options struct, and the ASC/DESC-to-MIN/MAX mapping

`CreateRowGroupReordererOptions` (`row_group_pruner.cpp:167`) is where the
`ORDER BY` direction becomes a concrete instruction about *which end* of each
row group's zonemap range to prioritize:

```cpp
// src/optimizer/row_group_pruner.cpp:174
const auto order_by = order_type == OrderType::ASCENDING ? OrderByStatistics::MIN : OrderByStatistics::MAX;
```

This is exactly the intuitive rule, made concrete: `ORDER BY col ASC LIMIT N`
wants the smallest values first, so row groups get ranked by their *minimum*
— the row group whose zonemap reports the smallest `min` is visited first,
since it's the one most likely to contain the smallest actual values.
`ORDER BY col DESC LIMIT N` wants the largest values first, so row groups get
ranked by their *maximum* instead. `combined_limit` folds `OFFSET` into the
row count that has to be found before pruning can stop (`row_limit +
row_offset`, when both are present), since the reorderer needs to find
enough qualifying rows to cover *both* the discarded offset prefix and the
retained limit suffix before it can prove later row groups are unreachable.

One additional refinement lives in this function and is worth calling out
because it goes beyond simple limit-pruning: if a nonzero, constant `OFFSET`
is present *and* the underlying table function can supply partition-level
statistics (`logical_get.function.get_partition_stats`), the pass makes a
*second* attempt, this time trying to prove that some number of leading row
groups (in sort order) are entirely and exactly consumed by the offset
itself — i.e., that every one of their rows definitely precedes the offset
window and none of them will ever be scanned at all, letting the *offset*
itself shrink (`logical_limit.offset_val` is rewritten down to the
remainder) while the original, full offset is preserved separately in
`logical_limit.unpruned_offset` for correct reporting. This mechanism is
covered together with `RowGroupReorderer::GetOffsetAfterPruning` below,
since it's the storage-layer half of the same offset-pruning idea.

### The execution-time mechanism: `RowGroupReorderer`

Everything above produces one artifact: a `RowGroupOrderOptions` struct
(column index, `MIN`/`MAX`, ascending/descending, null ordering, numeric-vs-
string, and an optional combined row limit) attached to the `LogicalGet` via
`LogicalGet::SetScanOrder`. **No row group is actually consulted, sorted, or
skipped at optimization time.** `table_scan.cpp:381-387` shows the handoff:
if the bind data carries `order_options`, a `RowGroupReorderer` is
constructed and attached to both the shared and local scan states, and from
that point forward, every place the ordinary sequential scan machinery would
have asked "what's the first row group" or "what's the next row group after
this one" instead asks the reorderer (`scan_state.cpp:232-278`):

```cpp
// src/storage/table/scan_state.cpp:232
optional_ptr<SegmentNode<RowGroup>> ParallelCollectionScanState::GetRootSegment(RowGroupSegmentTree &row_groups) const {
	if (reorderer) {
		return reorderer->GetRootSegment(row_groups);
	}
	return row_groups.GetRootSegment();
}
```

`RowGroupReorderer::GetRootSegment` (`row_group_reorderer.cpp:338`) is where
all of the real work happens, exactly once per scan, lazily, the first time
it's called:

```cpp
// src/storage/table/row_group_reorderer.cpp:351 (elided)
for (auto &row_group : row_groups.SegmentNodes()) {
	auto stats = row_group.GetNode().GetStatistics(options.column_idx);
	// ... classify into null_only_groups / ambiguous_groups / row_group_map (keyed by zonemap bound)
}
```

This is the moment this pass connects directly to Chapter 17's storage
material: `GetStatistics(options.column_idx)` retrieves the same
`BaseStatistics` — the same min/max `NumericStatsData` pair, for numeric and
temporal columns, that Chapter 17 §17.5 describes `RowGroup::CheckZonemap`
consulting — for *this specific row group's* copy of the sort column. Every
row group whose statistics are missing goes into an `ambiguous_groups`
bucket (scanned unconditionally, in original order, appended at the end);
every row group whose column is provably all-`NULL` goes into a
`null_only_groups` bucket (positioned according to `NULLS FIRST`/`NULLS
LAST` semantics); everything else goes into a `multimap<Value,
RowGroupSegmentNodeEntry>` keyed by exactly the bound (`min` for ascending,
`max` for descending) that `CreateRowGroupReordererOptions` selected.

**Reordering alone** — walking this multimap in key order — is what
happens whenever no usable row limit survived the optimizer-side checks
above (an intervening filter/aggregate/distinct, or a filter on the scan
itself). It costs nothing extra at execution beyond the one-time sort of
row-group entries (a handful of entries per table — a row group holds up to
122,880 rows, so even a huge table has relatively few row groups to sort),
and it means whatever consumes the scan's output — most often a
`LogicalTopN`'s bounded-heap physical operator — is fed candidate rows in an
order that's *likely* to expose the true top-`N` early, even though every
row group still eventually gets scanned.

**Provable pruning** — the strictly stronger case, and the one that actually
lets whole row groups go unread — is `AddRowGroups` (an anonymous-namespace
helper in `row_group_reorderer.cpp:46`), which only runs when a row limit
survived. Walking the sorted multimap in order, it maintains a running,
*conservative* count of "tuples definitely qualifying for the retained top-`N`
window" and compares each newly resolved row group's opposite-end zonemap
bound (the row group's `max`, when sorting ascending by `min`, say) against
the *next* row group's key. If the two don't overlap — this row group's own
extremal bound is strictly on the correct side of the next row group's bound
— then every one of this row group's rows is guaranteed to belong to the
"smaller/larger than everything not yet examined" set, and its row count
(via `GetMinimumQualifyingTupleCount`) can be added to the running total with
full confidence. Once that running total reaches the combined limit,
`AddRowGroups` returns `true` and every row group not yet added to
`ordered_row_groups` is simply never scanned:

```cpp
// src/storage/table/row_group_collection.cpp:359
if (state.reorderer) {
	// row groups pruned by the reorderer are never scanned
	idx_t total_rows = 0;
	for (auto &row_group : state.row_groups->SegmentNodes()) {
		total_rows += row_group.GetNode().count;
	}
	auto scan_rows = state.reorderer->ScanRowCount();
	state.skipped_rows = total_rows > scan_rows ? total_rows - scan_rows : 0;
}
```

That comment — *"row groups pruned by the reorderer are never scanned"* — is
the whole mechanism's payoff stated as plainly as the source ever states
anything: given a table with, say, one thousand 122,880-row groups and a
query asking for `ORDER BY col LIMIT 10`, if `col`'s values happen to be
laid out such that the very first (in sort order) handful of row groups'
zonemap ranges don't overlap with the rest of the table's at all, the scan
can stop after examining only those few row groups — a small, fixed number
of row groups' worth of I/O and decompression, independent of how large the
remaining 995 groups are, with zero bytes read from any of them.

Two details of `GetMinimumQualifyingTupleCount` are worth being precise
about rather than hand-waved, since they encode exactly how conservative the
proof has to be in the presence of possible `NULL`s:

```cpp
// src/storage/table/row_group_reorderer.cpp:25
idx_t GetMinimumQualifyingTupleCount(RowGroup &row_group, BaseStatistics &stats, const OrderByColumnType type,
                                     TransactionData transaction) {
	if (!stats.CanHaveNull()) {
		return row_group.GetVisibleRowCount(transaction);
	}
	if (type == OrderByColumnType::NUMERIC) {
		if (!NumericStats::HasMinMax(stats)) {
			return 0;
		}
		if (NumericStats::IsConstant(stats)) {
			return 1;
		}
		return 2;
	}
	// We cannot check if the min/max for StringStats have actually been set. As the strings may be truncated, we
	// also cannot assume that min and max are the same
	return 0;
}
```

If the column's statistics guarantee no `NULL`s are possible at all, the
*entire* visible row count of the row group counts as guaranteed-qualifying
— every row is a comparable, non-null value. If `NULL`s are possible, the
proof falls back to something much more conservative: a numeric column whose
min and max are provably different guarantees *at least two* distinct,
non-null values exist somewhere in the row group (two different values
necessarily means at least two rows), so `2` is added to the running total
regardless of how many rows the group actually has; a numeric column whose
min equals its max (`IsConstant`) guarantees only that *one* non-null value
is known to exist; and a `VARCHAR` column contributes `0` unconditionally,
because — as the comment states plainly — zonemap string bounds can be
truncated, so the code cannot even be sure the reported min and max are
exact enough to guarantee any specific count of real values at all. This is
a strict, safety-first bias: the algorithm would rather under-count how many
rows a row group is known to contribute (and therefore scan a few more row
groups than strictly necessary) than ever over-count and risk pruning away a
row group that could actually have contained a top-`N` result.

`OFFSET`-side pruning (`RowGroupReorderer::GetOffsetAfterPruning`,
`row_group_reorderer.cpp:262`) works over a separate input entirely —
*partition statistics* supplied by the table function
(`get.function.get_partition_stats`), which must report an **exact** row
count and an **exact** (not approximate) min/max per partition
(`MinMaxIsExact`) before this path even engages — and asks a narrower
question: can a *contiguous prefix* of row groups, in sort order, be proven
to fall entirely before the offset window, so those row groups need never be
scanned and the remaining `OFFSET` can shrink by their exact row count? This
is strictly more conservative than the limit-side proof (it demands *exact*
statistics, not merely a min/max bound, because an approximate count could
under- or over-shoot the offset by an amount that would silently return the
wrong row), and it can only prune a *prefix* — the moment a row group with an
uncertain (possibly-`NULL`-containing, when `NULLS_FIRST` is in effect) or
overlapping range is encountered, the search for prunable leading row groups
stops there, because offset pruning can only ever remove a leading run, not
scattered groups throughout the table.

### `NULL` handling: three buckets, ordered by `NULLS FIRST`/`NULLS LAST`

`GetRootSegment`'s final assembly step treats `null_only_groups`,
`ambiguous_groups`, and the sorted `row_group_map` differently depending on
where `NULL`s are supposed to sort:

```cpp
// src/storage/table/row_group_reorderer.cpp:372
if (options.null_order == OrderByNullType::NULLS_FIRST) {
	AppendRowGroups(null_only_groups, options.leading_null_group_offset, ordered_row_groups);
	AppendRowGroups(ambiguous_groups, 0, ordered_row_groups);
	SetRowGroupVector(row_group_map, options.row_limit, options.row_group_offset, options.order_type,
	                  options.column_type, ordered_row_groups, transaction);
} else {
	auto pruned_by_limit =
	    SetRowGroupVector(row_group_map, options.row_limit, options.row_group_offset, options.order_type,
	                      options.column_type, ordered_row_groups, transaction);
	AppendRowGroups(ambiguous_groups, 0, ordered_row_groups);
	if (!pruned_by_limit || options.order_type != OrderType::ASCENDING) {
		AppendRowGroups(null_only_groups, 0, ordered_row_groups);
	}
}
```

With `NULLS FIRST`, every all-`NULL` row group is visited before any
ordered, non-null row group — matching the semantics of `NULLS FIRST`
directly at the row-group level, not just within a row group — and
`ambiguous_groups` (uncertain statistics) are placed immediately after,
conservatively, since a group with no trustworthy stats might itself contain
leading `NULL`s that need to appear early. With `NULLS LAST` (the default),
the ordered groups are tried first, and null-only groups are appended at the
very end — *unless* the limit was already satisfied by earlier non-null
groups on an ascending scan, in which case null-only groups are correctly
omitted entirely (`!pruned_by_limit || options.order_type !=
OrderType::ASCENDING` — this condition is worth reading carefully: null-only
groups are skipped only when the scan *was* pruned by the limit *and* the
order is ascending, i.e., exactly the situation where the `NULL`s, sorting
last, could never have been part of the top-`N` window anyway).

### The test suite: what's actually in `zonemaps.test`, versus what tests `ROW_GROUP_PRUNER` for real

This is the point in this section where source-checking corrects an
assumption rather than confirming one. `test/optimizer/zonemaps.test` is a
real, substantial test file, and it genuinely exercises zonemap-consulting
machinery — but reading it in full shows it is not testing the pass this
section has just walked through. Every case in it asserts a
`physical_plan` regex containing the literal string `Filters:` (or
`<!REGEX>` for its absence), which is the physical scan operator's rendering
of its `TableFilterSet` — the pushed-down predicate that Chapter 17 §17.5
describes `RowGroup::CheckZonemap` consulting to skip a row group *whose
statistics are inconsistent with a filter*, not the reordered-scan-order
mechanism `ROW_GROUP_PRUNER` installs for an `ORDER BY ... LIMIT`. In other
words: `zonemaps.test` is really testing `FILTER_PUSHDOWN`/`filter_combiner`
— the machinery that turns `WHERE b in (1,2,3)` or `WHERE (2 < a and 7 > a)
or (a > 5 and a < 6)` into a single, tight `TableFilterSet` range the scan
can hand to `CheckZonemap` — and it's worth walking through in its own
right, both because it *is* zonemap-related and because it's the mechanism
`ROW_GROUP_PRUNER`'s own filter-present bail-out (§7.17.2, above) is written
in terms of.

Every case in the file has two parts: an `EXPLAIN` with a regex on the
filter text, and then a `count(*)` query. So each case checks the answer as
well as the plan.

**Setup:**
```sql
-- test/optimizer/zonemaps.test:9
create temporary table t as
  select range a, length(range::varchar) b, mod(range,10000) c, 5::BIGINT d, 10000 e
  from range(100000);
```

(`d` is cast to `BIGINT`, so it is not a plain `INTEGER`.)

**Case 1 — an `IN` list collapses to a usable range filter:**
```sql
-- test/optimizer/zonemaps.test:12-20
explain select count(*) from t where b in (1,2,3) ;
----
physical_plan	<REGEX>:.*Filters:.*b <= 3.*

select count(*) from t where b in (1,2,3) ;
----
1000
```
`b`'s three literal values are next to each other (1, 2, 3). The filter
combiner merges them into one range, so the scan gets a `b <= 3` bound in its
`Filters:` list. This is the predicate `CheckZonemap` tests each row group's
`b`-column min/max against. The count, 1000, is the number of values in
`range(100000)` with 1 to 3 digits (0 to 999).

**Case 2 — an already-range-shaped predicate passes through directly:**
```sql
-- test/optimizer/zonemaps.test:22-30
explain select count(*) from t where b <=3 and b>=0;
----
physical_plan	<REGEX>:.*Filters:.*b <= 3.*
```

**Case 2b — an `IN` list or `OR` with gaps becomes an *optional* filter:**
```sql
-- test/optimizer/zonemaps.test:35-53
explain select count(*) from t where b in (1, 3) ;
----
physical_plan	<REGEX>:.*Filters:.*optional:.*b IN.*1, 3.*

explain select count(*) from t where b = 1 or b = 3 ;
----
physical_plan	<REGEX>:.*Filters:.*optional:.*b = 1.*OR.*b = 3.*
```
The values 1 and 3 are not next to each other, so the combiner does not
make a tight range. The scan shows the `IN` list (or the `OR`) itself,
marked `optional:`. An optional filter is a
hint: the scan may use it to skip row groups by their zonemaps, but the exact
filter still runs above the scan. Both queries return 910 (the 10 one-digit
values 0-9 plus the 900 three-digit values 100-999).

**Case 3 — transitive filters:**
```sql
-- test/optimizer/zonemaps.test:56-74
explain select count(*) from t where b < 5 and d = 5
----
physical_plan	<REGEX>:.*Filters:.*b < 5.*

explain select count(*) from t where d = 5 and b < d
----
physical_plan	<REGEX>:.*Filters:.*b < 5.*
```
The second query's predicate, `b < d`, isn't a filter on a constant at all —
but `d = 5` fixes `d`'s value, and the filter combiner can substitute that
known constant into `b < d` to derive `b < 5`, a bound the scan's zonemap
check on `b` can actually use, even though the original SQL never wrote a
literal `5` next to `b`. (The regex checks for `b < 5` itself, not just
for some `Filters:` line.)

**Case 4 — zonemap-usable filters propagated across a join:**
```sql
-- test/optimizer/zonemaps.test:78-93
create temporary table t1 as select range a, length(range::VARCHAR) nationkey from range(100000);
create table t2 as select range nationkey, concat('Nation_',range) n_name from range(25);

explain select count(*) from t1 join t2 using (nationkey)
where t2.n_name = 'Nation_1' and t1.nationkey = 1;
----
physical_plan	<REGEX>:.*Filters:.*Filters:.*
```
Two `Filters:` occurrences are required here — one per side of the join —
because `FILTER_PULLUP`/`FILTER_PUSHDOWN` (§7.6.3 in the overview) derive a
filter on `t2.nationkey` from the equi-join condition plus the known
`t1.nationkey = 1` filter, so *both* base scans end up with a usable,
zonemap-consultable predicate, even though the original query only wrote a
literal filter on one side directly and an equality filter on the other
side's joined-on column indirectly.
```sql
-- test/optimizer/zonemaps.test:117-125
explain select count(*) from t1 join t2 using (nationkey) where t2.nationkey =2 or n_name= 'Nation_2'
----
physical_plan	<!REGEX>:.*Filters:.*
```
An `OR` across two *different* columns (`nationkey` and `n_name`) cannot be
collapsed into a single per-column range the way an `OR` across the *same*
column's ranges could — there is no single interval of `nationkey` values
*and* no single interval of `n_name` values that captures "nationkey = 2 OR
n_name = 'Nation_2'" without over- or under-approximating, so no
`TableFilterSet` gets attached to either scan at all, and the negative
regex confirms it.

The last join case, `t2.n_name in ('Nation_1', 'Nation_2')`
(`zonemaps.test:127-135`), expects an `optional:` `IN` filter on `n_name`,
the same shape as Case 2b.

**Case 5 — the `OR`-of-ranges battery across every scalar type**
(`tinyint`, `smallint`, `integer`, `bigint`, `double`, `real`, `varchar`).
The table `test` holds 23 rows (`range(23)`, with `e` and `f` set to
`range / 2.0`). The queries are a long run of
`(2 < a and 7 > a) or (a > 5 and a < 6)`-shaped predicates, written with the
operands flipped and with strict and inclusive bounds mixed.

The regexes expect the scan to keep the whole `OR` as written, not one
folded range such as `a>=2 AND a<=7`. For example:

```sql
-- test/optimizer/zonemaps.test:147-155
explain select count(*) from test where (2 < a and 7 > a) or (a >5 and a < 6)
----
physical_plan	<REGEX>:.*Filters:.*2 < a.*7 > a.*OR.*a > 5.*a < 6.*

select count(*) from test where (2 < a and 7 > a) or (a >5 and a < 6)
----
4
```

So the filter does reach the scan, as one `OR` filter that zonemaps can
check branch by branch. But the filter combiner does not merge the two
ranges into one interval. The real proof of correctness is the `count(*)`
after each `EXPLAIN`. Section 5.2 (`zonemaps.test:187-315`) runs the same
four bound shapes over `a`–`d` with `foreach col a b c d`, over `e` and `f`
(where the bounds print as `2.0`, `7.0`, and so on), and over the `VARCHAR`
column `g` (string comparison, so `'10'` sorts between `'1'` and `'2'`).

**Case 5.3 — casting, and a `DATE` zonemap surviving a string-literal `IN`
list:**
```sql
-- test/optimizer/zonemaps.test:317-337
CREATE TABLE test_date(a DATE);
INSERT INTO test_date VALUES ('1995-12-01'), ('1995-12-10');

EXPLAIN SELECT count(*) FROM test_date WHERE a = '1995-12-01';
----
physical_plan	<REGEX>:.*Filters:.*a = '1995-12-01'::DATE.*

EXPLAIN SELECT count(*) FROM test_date WHERE a IN ('1995-12-01', '1995-12-02');
----
physical_plan	<REGEX>:.*Filters:.*optional:.*a IN.*'1995-12-01'::DATE.*'1995-12-02'::DATE.*
```
The string literals are cast to `DATE` once, at plan time (matching
`InClauseSimplificationRule`'s own cast-invertibility reasoning from the
overview's §7.5). The list does not fold into a range. It stays an `IN`
list of `DATE` constants, again marked `optional:`. Either way, `a`'s zonemap, a plain min/max pair of
`DATE` values per Chapter 17 §17.5, is checked against `DATE` values, so the
storage layer never needs to parse strings.

**Case 5.4 — `NULL`s inside an `IN` list:**
```sql
CREATE TABLE t0(c0 BOOLEAN);
INSERT INTO t0 VALUES (false), (NULL);

SELECT t0.c0 FROM t0 WHERE t0.c0 IN (false, NULL);
----
0
```
This last case is a pure correctness check, not a plan-shape assertion —
`IN` with a `NULL` in the list follows three-valued SQL logic (a value can
match one of the non-null alternatives and return true, but can never be
proven *not* to be in the list just because it isn't `NULL` itself unless it
positively matches something), and the single surviving row (`false`) proves
the zonemap/filter-combining machinery upstream of this never silently
mishandles `NULL` into producing a wrong answer, whatever plan shape it
settles on.

**The tests that actually exercise `ROW_GROUP_PRUNER`** live under
`test/sql/topn/`, not `test/optimizer/`, and are worth walking through in
their own right, both because they're the real evidence for everything
described above and because every one of them is a regression test for a
specific edge case in `RowGroupReorderer`'s `NULL`/offset bookkeeping — which
means each test's *name* is close to a plain-English description of the bug
it once caught.

**`test_top_n_offset_no_limit_pruner.test`** — *"RowGroupPruner must not
truncate scan when LIMIT is unset"*:
```sql
CREATE TABLE t AS SELECT range AS id FROM range(300000);
-- 300,000 rows -> 3 default-size row groups (122880 / 122880 / 54240)

SELECT COUNT(*) FROM (SELECT id FROM t ORDER BY id OFFSET 0)
----
300000

SELECT COUNT(*) FROM (SELECT id FROM t ORDER BY id OFFSET 200000)
----
100000

SELECT MAX(id) FROM (SELECT id FROM t ORDER BY id OFFSET 200000)
----
299999
```
An `ORDER BY id OFFSET 200000` with **no `LIMIT` at all** still qualifies for
`GetLimitAndOffset` to attach an `optional_idx` offset (since
`LimitNodeType::UNSET` for the limit maps to a genuinely absent, "scan
everything" bound — `row_limit` stays invalid throughout), but the comment
records this test existed because of a real regression where an unset limit
was, at some point, mishandled as if it meant "limit of zero," truncating
the scan to nothing. The third row group (index range
`[245760, 300000)`) has to survive intact for `MAX(id) = 299999` to come
back correctly.

**`test_top_n_nulls_row_group_pruner.test`** — *"Test row group pruning with
NULLS LAST and offset across mixed row groups"*:
```sql
CREATE TABLE integers AS
  SELECT NULL::BIGINT AS i FROM range(122881)
  UNION ALL
  SELECT range::BIGINT AS i FROM range(122880);
restart

SELECT i FROM integers ORDER BY i NULLS LAST LIMIT 1 OFFSET 122880;
----
NULL
```
122,881 `NULL` rows (spilling across a row-group boundary by exactly one
row) followed by 122,880 non-null rows means the non-null values occupy
positions `0` through `122879` once sorted `NULLS LAST` — so `OFFSET 122880`
lands *exactly* on the first `NULL` that sorts after all of them, and the
`restart` forces the table through a real checkpoint/reload so the test
exercises genuine on-disk row groups and real persisted zonemaps rather than
in-memory ones.

**`test_top_n_distinct_row_group_pruner.test`** — *"DISTINCT should
invalidate row group pruning limits for persisted top-n plans"*:
```sql
ATTACH '...' AS db (ROW_GROUP_SIZE 2048)
CREATE TABLE db.t(i BIGINT);
INSERT INTO db.t SELECT 0 FROM range(2048);
INSERT INTO db.t SELECT 1 FROM range(2048);
restart

SELECT DISTINCT i FROM db.t ORDER BY 1 LIMIT 2
----
0
1
```
This is the data-level proof of the `LOGICAL_DISTINCT`-invalidates-pruning
branch in `TryOptimize` discussed above: with two row groups, each entirely
one constant value (`0` and `1` respectively), a naive limit-pruned scan
that stopped after finding `2048` "qualifying" rows in the first row group
alone (all of them `0`) would never even look at the second row group, and
`DISTINCT i ... LIMIT 2` would wrongly return only `0`. Because
`LOGICAL_DISTINCT` sits between the `ORDER BY` and the scan, `row_limit` was
correctly invalidated at plan time, the scan still visits both row groups
(in the right order), and `0, 1` comes back correctly.

**`test_top_n_nulls_first_row_group_reorderer_edge_cases.test`** and
**`test_top_n_nulls_row_group_pruner_extensive.test`** are large, deliberately
combinatorial regression suites — small (`2048`-row) row groups built with
`ROW_GROUP_SIZE 2048` specifically so a handful of `INSERT`s can produce many
distinct row groups per test, each engineered to be **null-only**, **mixed
null/non-null in a specific split**, or **scattered-null** — checking every
combination of `NULLS FIRST`/`NULLS LAST`, ascending/descending, `BIGINT`
and `VARCHAR`, with and without an `OFFSET` that lands precisely on a
row-group boundary. A representative case:
```sql
CREATE TABLE db.mixed_ints AS
  SELECT i FROM repeat(NULL::BIGINT, 1023) t(i)
  UNION ALL
  SELECT i FROM repeat(1::BIGINT, 1025) t(i);
-- (one 2048-row row group: 1023 NULLs followed by 1025 copies of the value 1)

EXPLAIN ANALYZE SELECT i FROM db.mixed_ints ORDER BY i NULLS FIRST LIMIT 1
----
analyzed_plan	<REGEX>:.*Table Scan.*2,048 rows.*

SELECT i FROM db.mixed_ints ORDER BY i NULLS FIRST LIMIT 1 OFFSET 1023
----
1
```
Note this `EXPLAIN ANALYZE` case asserts the scan touches *all* `2,048` rows
of the table's single row group — this is a `NULLS FIRST` case where the
group is neither provably all-null nor provably all-non-null, so it lands in
the `ambiguous_groups` bucket described above and gets scanned in full
rather than pruned, and the test is confirming exactly that (not confirming
a successful prune) — `EXPLAIN ANALYZE`'s real row count is being used here
precisely as this section earlier described: as the ground truth for whether
pruning actually happened, since a plain (pre-execution) `EXPLAIN` can't show
it. The suite's `require no_alternative_verify` and `SET threads TO 1`
directives, present in both files, exist because DuckDB's test harness's
"alternative verify" mode independently re-runs queries with some optimizers
disabled to catch correctness bugs the optimizer's presence might otherwise
mask — and one of the very things that mode disables is exactly the
offset-pruning behavior these tests are trying to pin down, so the tests
opt out of that mode explicitly rather than have it silently no-op the
mechanic under test.

**`test/issues/general/test_21160.test`** rounds out the picture with a tiny,
single-row-then-growing table exercising the boundary where the sort column
is **entirely** `NULL` for the only row that matches a `WHERE` clause, both
with and without a `LIMIT`, and both `NULLS FIRST`/`NULLS LAST` — plus a
comment worth quoting directly because it states a real, documented
limitation in the optimizer's own coverage: *"Explicit NULLS FIRST — NULLs
appear before non-nulls (optimizer skips reorderer for this case)"* — i.e.,
for at least this test's specific shape, the reorderer path is bypassed
entirely for `NULLS FIRST` and the query falls back to an ordinary sort,
which is correct (just not accelerated) rather than exercising the
pruning path.

### Honesty about the limits: what happens when physical layout doesn't cooperate

Every mechanism above is described in terms of "provably qualifying" and
"provably excluded" — the reorderer never guesses. When a table's physical
row-group layout has no relationship at all to the `ORDER BY` column's
values (an `id` column that increases with insertion order but an `ORDER BY
some_unrelated_score_column` that was scattered essentially at random across
row groups by whatever order rows happened to be inserted in), the multimap
built from each row group's zonemap bound still gets constructed and sorted
— that part of the work is unconditional and cheap — but every row group's
range will tend to overlap heavily with its neighbors', so `AddRowGroups`'s
overlap check will almost never find a clean non-overlapping boundary early,
and the running "guaranteed qualifying" total will climb far more slowly
than the row groups actually visited. In the worst case, no row group is
ever excludable at all, and the scan degrades to visiting every row group —
in the *reordered* order rather than storage order, which is harmless, not
wrong, but which also buys nothing beyond the ordinary correctness a full
scan-and-sort would have provided anyway. This is, precisely, the same
"depends entirely on physical correlation, not on the filter's abstract
selectivity" point Chapter 17 §17.5 makes about `CheckZonemap` itself: a
range filter on a column correlated with row-group layout skips almost
everything; the identical filter on an uncorrelated column skips almost
nothing, at exactly the same cost of checking. `ROW_GROUP_PRUNER` is the
`ORDER BY ... LIMIT` sibling of that same underlying fact about zonemaps —
it is a heuristic bet on data layout, cheap to attempt and safe to lose,
never a guarantee.

## 7.17.3 `SAMPLING_PUSHDOWN` — pushing `SYSTEM` sampling into the scan itself

**One-sentence purpose:** when a `LogicalSample` sits directly above a bare
`LogicalGet` and the sampling method is `SYSTEM` (not `BERNOULLI`, not
`RESERVOIR`), fold the sampling decision into the scan's own bind data so the
scan reads only the fraction of the table the sample actually needs, instead
of reading the whole table and discarding rows afterward.

### The whole pass, and exactly which sampling methods qualify

```cpp
// src/optimizer/sampling_pushdown.cpp:9
unique_ptr<LogicalOperator> SamplingPushdown::Optimize(unique_ptr<LogicalOperator> op) {
	if (op->type == LogicalOperatorType::LOGICAL_SAMPLE && !op->children.empty() &&
	    op->children[0]->type == LogicalOperatorType::LOGICAL_GET && op->children[0]->children.empty()) {
		auto &sample_op = op->Cast<LogicalSample>();
		auto &get = op->children[0]->Cast<LogicalGet>();
		const auto &sample_options = *sample_op.sample_options;
		const bool has_filters = get.table_filters.HasFilters() || get.dynamic_filters;
		const bool can_push_system_sample =
		    get.function.sampling_pushdown && sample_options.method == SampleMethod::SYSTEM_SAMPLE && !has_filters;

		if (can_push_system_sample) {
			// ...
		}
	}
	for (auto &child : op->children) {
		child = Optimize(std::move(child));
	}
	return op;
}
```

Four conditions gate the pushdown, all visible in `can_push_system_sample`'s
construction:

1. **Shape**: `LOGICAL_SAMPLE` directly above a `LOGICAL_GET` whose own
   `children` list is empty — a genuine, bare base-table scan with nothing
   else already sitting under the sample. A join or a filtered subquery
   directly under the sample fails this immediately (the derived-table test
   case below shows exactly this).
2. **The table function supports it at all**: a boolean field,
   `function.sampling_pushdown`, gates this per-reader — a direct search
   shows only DuckDB's own table scan sets it:
   ```cpp
   // src/function/table/table_scan.cpp:1306
   scan_function.sampling_pushdown = true;
   ```
3. **The sampling method is exactly `SYSTEM_SAMPLE`.** `SampleMethod` has
   three values — `SYSTEM_SAMPLE`, `BERNOULLI_SAMPLE`, `RESERVOIR_SAMPLE`
   (`src/include/duckdb/parser/parsed_data/sample_options.hpp:18`) — and only
   the first is ever eligible here; the condition checks it by name, with no
   fallback path for the other two. The test suite (below) confirms both
   `BERNOULLI` and `RESERVOIR` sampling keep their standalone sample operator
   sitting above an ordinary, full sequential scan, unconditionally.
4. **No filters already on the scan**: `get.table_filters.HasFilters() ||
   get.dynamic_filters` — a static, pushed-down `WHERE` predicate *or* a
   dynamic (runtime, e.g. join-filter-pushdown-installed) filter both
   disqualify the rewrite. The reasoning: `SYSTEM` sampling works by
   including or excluding whole vectors/chunks as they're read off disk,
   independent of their contents; sampling *before* a filter runs would
   produce a sample of the table's raw population, while sampling *after*
   the filter (the un-pushed-down behavior) produces a sample of the
   *filtered* population — genuinely different distributions unless the
   filter happens to be a no-op, so DuckDB conservatively refuses to
   conflate the two rather than silently changing what "a `25%` sample"
   means once a `WHERE` clause is also present.

### Row-count sampling: a probabilistic push plus an exact-count safety net

`SYSTEM(0.1%)`-style percentage sampling pushes down with nothing more than
handing the `SampleOptions` struct straight to the scan
(`get.extra_info.sample_options = std::move(sample_op.sample_options)`), but
`USING SAMPLE 2 ROWS (system)`-style *row-count* sampling needs one extra
step, because `SYSTEM` sampling is fundamentally a per-chunk probabilistic
decision, not an exact-count mechanism:

```cpp
// src/optimizer/sampling_pushdown.cpp:20
const bool is_row_count_sampling = !sample_options.is_percentage;
int64_t row_limit = 0;
if (is_row_count_sampling) {
	if (!sample_op.sample_options->seed.IsValid()) {
		auto &random_engine = RandomEngine::Get(context);
		sample_op.sample_options->SetSeed(random_engine.NextSeed32());
	}
	// For row-count sampling, calculate the sampling rate based on estimated cardinality.
	// Use EstimateCardinality which can query table function stats if has_estimated_cardinality is not set.
	row_limit = sample_options.sample_size.GetValue<int64_t>();
	const idx_t estimated_card = get.EstimateCardinality(context);
	if (estimated_card > 0) {
		sample_op.sample_options->sample_rate =
		    static_cast<double>(row_limit) / static_cast<double>(estimated_card);
	} else {
		sample_op.sample_options->sample_rate = 1.0;
	}
}

get.extra_info.sample_options = std::move(sample_op.sample_options);
op = std::move(op->children[0]);

if (is_row_count_sampling) {
	// Wrap with LIMIT to ensure exact row count and enable early stopping.
	// The pushdown sampling may oversample due to the distributed chunk-based
	// approach, so LIMIT ensures we stop as soon as the target is reached.
	auto limit = make_uniq<LogicalLimit>(BoundLimitNode::ConstantValue(row_limit), BoundLimitNode());
	limit->children.push_back(std::move(op));
	op = std::move(limit);
}
```

`2 ROWS` is converted to an *estimated* per-chunk inclusion probability
(`sample_rate = 2 / estimated_cardinality`), because there's no way to know
in advance exactly which chunks, sampled independently, will land on exactly
`2` surviving rows — this is a probabilistic estimate, not an exact
computation, and the inline comment names the consequence plainly: the
pushed-down sample "may oversample." Rather than trying to compute a
perfectly calibrated rate (which would require knowing the table's exact
size and distribution in advance, defeating the point of sampling instead of
scanning), the pass accepts the imprecision and **wraps the rewritten scan
in a fresh `LogicalLimit(row_limit)`** to guarantee the query still returns
exactly the requested row count — and, as a direct consequence, to let
execution stop reading further chunks the moment enough sampled rows have
accumulated, which is itself an early-stopping benefit layered on top of the
I/O-reduction benefit sampling-at-the-scan already provides. `LogicalSample`
itself is discarded entirely in this branch (`op = std::move(op->children[0])`
replaces the sample node with its own child, the now-annotated `LogicalGet`,
before the `LogicalLimit` wrapper is added on top) — there is no
stand-alone `PhysicalSample`/`plan_sample.cpp` lowering left to run for this
subtree at all, which is exactly the detail one of the test cases below
specifically guards against regressing.

One more detail of the excerpt: the seed. A row-count sample with no
`REPEATABLE (seed)` clause still gets a fixed seed here, so that every thread
of the scan uses the same one. The seed comes from the connection's
`RandomEngine`, through `NextSeed32`:

```cpp
// src/common/random_engine.cpp:101-104
uint32_t RandomEngine::NextSeed32() {
	auto seed = NextRandomInteger();
	return explicitly_seeded ? seed : seed ^ static_cast<uint32_t>(EntropySeed());
}
```

If the user set a seed (`SET seed`, `setseed()`), the engine's next number is
used as before, so results repeat. If not, the number is mixed (XOR) with
fresh entropy from the operating system.

Why mix in fresh entropy? A connection's `RandomEngine` is seeded once, when
the connection is made. If the process is then copied (by `fork()`, or by
restoring a VM snapshot), every copy holds the same engine state. Without
the entropy, every copy would draw the same "random" sample.
`plan_sample.cpp`, where an unpushed `LogicalSample` gets its seed, does the
same thing.

### The test suite: `test/optimizer/sampling_pushdown.test`

**Setup:**
```sql
CREATE TABLE integers1(i INTEGER, j INTEGER);
CREATE TABLE integers2(i INTEGER, j INTEGER);
INSERT INTO integers1 VALUES (1,1), (2,2), (3, 3), (4,4);
INSERT INTO integers2 VALUES (1,1), (2,2), (3, 3), (4,4);
```

**Case 1 — percentage `SYSTEM` sampling pushes into the scan, both syntaxes:**
```sql
EXPLAIN SELECT i FROM integers1 tablesample system(0.1%)
----
physical_plan	<REGEX>:.*SEQ_SCAN.*System: 0.1%.*

EXPLAIN SELECT i FROM integers1 using sample system(0.1%)
----
physical_plan	<REGEX>:.*SEQ_SCAN.*System: 0.1%.*
```
Both `TABLESAMPLE` (the SQL-standard-flavored syntax) and `USING SAMPLE`
(DuckDB's own alternate syntax) parse to the same `LogicalSample`, and both
end up rendered as an annotation directly on the `SEQ_SCAN` physical
operator itself — no separate sample operator remains in the tree at all.

**Case 2 — a filter blocks the pushdown, and the negative check is explicit:**
```sql
EXPLAIN SELECT i FROM (SELECT i FROM integers1 WHERE i > 1) USING SAMPLE system(25%)
----
physical_plan	<REGEX>:.*STREAMING_SAMPLE.*System: 25.*%.*

EXPLAIN SELECT i FROM (SELECT i FROM integers1 WHERE i > 1) USING SAMPLE system(25%)
----
physical_plan	<!REGEX>:.*SEQ_SCAN.*System: 25.*%.*
```
Same query, asserted twice — once for what *is* in the plan
(`STREAMING_SAMPLE`, the physical operator for a non-pushed-down sample,
sitting above the filtered scan), and once, with `<!REGEX>`, for what is
*not*: no `SEQ_SCAN` anywhere carries the `System: 25%` annotation. This
matches `has_filters` disqualifying the rewrite exactly as described above —
`WHERE i > 1` is already a pushed-down `TableFilterSet` on the scan by the
time `SAMPLING_PUSHDOWN` (pass #31) runs, since `FILTER_PUSHDOWN` (pass #6)
ran many passes earlier.

**Case 3 — each side of a join gets its own independent pushdown:**
```sql
EXPLAIN SELECT * FROM integers1 tablesample system(0.1%), integers2 tablesample system(0.1%)
----
physical_plan	<REGEX>:.*SEQ_SCAN.*System: 0.1%.*
```
Comma-separated `FROM`-list syntax attaches each `TABLESAMPLE` clause
directly to its own table reference — so each one is independently a bare
`LogicalSample` over a bare `LogicalGet`, *before* the cross product or join
combines them, and both push down on their own merits, entirely independent
of each other.

**Case 4 — `BERNOULLI` and `RESERVOIR`: categorically ineligible:**
```sql
EXPLAIN SELECT i FROM integers1 tablesample bernoulli(0.1%)
----
physical_plan	<REGEX>:.*Bernoulli.*SEQ_SCAN.*

EXPLAIN SELECT i FROM integers1 tablesample reservoir(0.1%)
----
physical_plan	<REGEX>:.*RESERVOIR_SAMPLE.*SEQ_SCAN.*
```
Both keep their own standalone physical sample operator (`Bernoulli`,
`RESERVOIR_SAMPLE`) sitting above a completely ordinary `SEQ_SCAN`, because
the method check in `can_push_system_sample` excludes both categorically —
this isn't a data-dependent decision the way the filter check is; it's a
hard, method-name equality test that only `SYSTEM_SAMPLE` ever satisfies.

**Case 5 — a derived table (or any non-bare `LogicalGet`) blocks pushdown:**
```sql
EXPLAIN SELECT * FROM integers1, integers2 where integers1.i = integers2.i USING SAMPLE SYSTEM(25%)
----
physical_plan	<REGEX>:.*System.*SEQ_SCAN.*
```
Here the `USING SAMPLE` clause applies to the *joined* result as a whole,
so the `LogicalSample`'s immediate child is a `LogicalComparisonJoin`, not a
bare `LogicalGet` — the very first structural check in `Optimize` fails, and
the sample stays exactly where the binder put it, above the join.

**Case 6 — row-count sampling, and the injected `LIMIT`'s exact-count
guarantee:**
```sql
EXPLAIN SELECT i FROM integers1 USING SAMPLE 2 ROWS (system)
----
physical_plan	<REGEX>:.*SEQ_SCAN.*System: 2 rows.*
```
```sql
# Row-count system sampling: SamplingPushdown injects a LogicalLimit, but LogicalSample is
# removed so plan_sample.cpp is not applied for that subtree. The physical plan must not
# stack two LIMIT operators (would duplicate Limit: N in the rendered plan).
EXPLAIN SELECT i FROM integers1 USING SAMPLE 2 ROWS (system)
----
physical_plan	<!REGEX>:.*Limit:.*Limit:.*
```
The scan itself carries `System: 2 rows` (the row-count annotation, distinct
from a percentage), and the negative assertion is a direct regression guard
for exactly the "`LogicalSample` discarded, `LogicalLimit` injected" behavior
walked through in the source above — without that discard, a stray, already-
lowered sample-related limit rendering could combine with the newly injected
`LogicalLimit` to print `Limit:` twice in the same plan, which this test
exists specifically to catch.

**Case 7 — filtered row-count sampling: same no-pushdown behavior as the
percentage case:**
```sql
EXPLAIN SELECT i FROM (SELECT i FROM integers1 WHERE i > 1) USING SAMPLE 2 ROWS (system)
----
physical_plan	<REGEX>:.*STREAMING_SAMPLE.*System: 2 rows.*

EXPLAIN SELECT i FROM (SELECT i FROM integers1 WHERE i > 1) USING SAMPLE 2 ROWS (system)
----
physical_plan	<!REGEX>:.*SEQ_SCAN.*System: 2 rows.*
```
Identical reasoning to case 2 — the `has_filters` check doesn't distinguish
between percentage and row-count sampling; a filtered scan blocks both
equally.

**Cases 8 and 9 — alternate syntaxes and `REPEATABLE`, same pushdown:**
```sql
EXPLAIN SELECT i FROM integers1 USING SAMPLE system(2 ROWS)
----
physical_plan	<REGEX>:.*SEQ_SCAN.*System: 2 rows.*

EXPLAIN SELECT i FROM integers1 USING SAMPLE 2 ROWS (system, 42)
----
physical_plan	<REGEX>:.*SEQ_SCAN.*System: 2 rows.*
```
Both `2 ROWS (system)` and `system(2 ROWS)` are the same sampling request
written with the method name in a different position, and adding an
explicit seed (`42`, for `REPEATABLE`) doesn't disturb the pushdown decision
at all — `sample_op.sample_options->seed.IsValid()` in the source above only
*assigns* a fresh random seed when none was given; an explicit seed is
simply carried through unchanged.

### `test/optimizer/sampling_pushdown.test_slow`: the `.test_slow` convention, and what changes at scale

DuckDB's test suite draws a real, load-bearing distinction between a
`.test` file and a `.test_slow` file: the `_slow` suffix marks a test as
excluded from the default, fast `make unittest` run and reserved for
extended or nightly CI passes, precisely because it does something
expensive enough (here: `CALL DBGEN(sf=0.1)`, generating actual TPC-H data,
and running `EXPLAIN ANALYZE` — a real execution, not just a plan
inspection) that running it on every single test invocation would slow down
the ordinary development feedback loop for no proportionate benefit.

The `.test_slow` variant reproduces the *same* set of scenarios as
`.test` — pushdown succeeds for percentage `SYSTEM` sampling, succeeds
across a join, fails for `BERNOULLI`/`RESERVOIR`, fails for a
derived table — but changes what's actually being proven in three
concrete ways:

1. **Real, disk-backed TPC-H data instead of four in-memory rows.**
   `lineitem` at scale factor `0.1` is large enough that "the scan reads
   fewer rows" is an empirically checkable, not merely structurally
   asserted, claim — which is exactly why every assertion in this file uses
   `EXPLAIN ANALYZE` (the `analyzed_plan` column, which reflects real
   execution row counts) rather than plain `EXPLAIN`'s `physical_plan`.
   ```sql
   require tpch
   CALL DBGEN(sf=0.1);

   EXPLAIN ANALYZE SELECT count(*) FROM lineitem tablesample system(0.1%)
   ----
   analyzed_plan	<REGEX>:.*TABLE_SCAN.*System: 0.1%.*
   ```
2. **A real multi-table-scan join, at scale:**
   ```sql
   EXPLAIN ANALYZE SELECT count(*) FROM lineitem tablesample system(0.1%), orders tablesample system(0.1%)
   ----
   analyzed_plan	<REGEX>:.*TABLE_SCAN.*System: 0.1%.*
   ```
   confirming the same independent-per-side pushdown the small `.test` file's
   case 3 already asserted structurally, this time against `lineitem` and
   `orders` at a size where actually executing the query (rather than merely
   planning it) is a meaningful thing to check at all.
3. **The negative cases carry over unchanged in spirit, at scale:**
   ```sql
   EXPLAIN ANALYZE SELECT count(*) FROM lineitem tablesample bernoulli(0.1%)
   ----
   analyzed_plan	<REGEX>:.*Bernoulli.*TABLE_SCAN.*

   EXPLAIN ANALYZE SELECT count(*) FROM lineitem tablesample reservoir(0.1%)
   ----
   analyzed_plan	<REGEX>:.*RESERVOIR_SAMPLE.*TABLE_SCAN.*

   EXPLAIN ANALYZE SELECT count(*) FROM lineitem, orders where l_orderkey = o_orderkey USING SAMPLE SYSTEM(25%)
   ----
   analyzed_plan	<REGEX>:.*System.*TABLE_SCAN.*
   ```
   — the same three disqualifying shapes (wrong method, wrong method again,
   sample-above-a-join) reproduced against `lineitem`/`orders` rather than
   `integers1`/`integers2`.

Put simply: the `.test` file exists to check the *optimizer's decision* is
correct — a fast, structural, plan-shape check that belongs in every
ordinary test run; the `.test_slow` file exists to check that decision holds
up under conditions closer to what actually motivates the optimization in
the first place (a real, disk-resident, multi-million-row table, where "does
the scan actually read less" is a claim worth spending the DBGEN generation
time and the real execution time to verify, but not a claim worth paying
that cost for on every single `cargo`/`make`-triggered test run during
ordinary development).

## 7.17.4 How the three passes actually interact, in one query

It's worth closing this file by tracing all three passes against a single
query that touches every one of them, since none of the individual test
files does this in combination:

```sql
SELECT i FROM my_table WHERE i > 100 ORDER BY i LIMIT 5;
```

- **`LIMIT_PUSHDOWN`** (#29) looks at the tree first and finds nothing to do
  — `LogicalLimit`'s child here is `LogicalOrder`, not a plain
  `LogicalProjection`, so `CanOptimize`'s very first condition fails.
- **`ROW_GROUP_PRUNER`** (#30) looks next, finds the `LogicalOrder` beneath
  the limit, confirms `i` is a direct `BOUND_COLUMN_REF` of a numeric type,
  walks down toward the base table finding nothing but the scan itself in
  between — but then hits the base-scan's own `table_filters.HasFilters()`
  check: `WHERE i > 100` is already sitting on the scan as a pushed-down
  filter (from `FILTER_PUSHDOWN`, many passes earlier), so `row_limit`/
  `row_offset` both get invalidated right there. The row groups still get
  reordered by `i`'s ascending zonemap `min` — cheap, and still likely
  useful — but no row group can be *proven* skippable purely from the limit,
  since some unknown fraction of any given row group's rows might fail the
  `i > 100` filter, and the reorderer has no way to know in advance how many
  will survive.
- **`SAMPLING_PUSHDOWN`** (#31) has nothing to do at all — there's no
  `SAMPLE`/`TABLESAMPLE` clause anywhere in this query.
- **`TOP_N`** (#32) runs last of the four and fuses the still-intact
  `LogicalOrder`+`LogicalLimit` pair into a single `LogicalTopN(ORDER BY i
  LIMIT 5)`, exactly as it would have regardless of what the previous three
  passes did, since none of them touched the tree's *shape* — only the
  `LogicalGet`'s scan-order annotation changed.

The net effect: at execution time, the physical `TopN` operator's bounded
top-5 heap gets fed rows from a scan that visits row groups in ascending-`i`
order (helpful, because rows likely to end up in the true top-5 tend to
arrive early) but still has to visit every row group that could contain any
row satisfying `i > 100` (no row group gets skipped outright, because the
filter's presence blocked the provable-pruning path). Change the query to
drop the `WHERE` clause entirely — `SELECT i FROM my_table ORDER BY i LIMIT
5` — and the *same* `ROW_GROUP_PRUNER` pass, on the *same* table, now finds
`table_filters.HasFilters()` false, keeps `row_limit` intact, and can, if the
data's physical layout cooperates, stop after the very first row group and
never touch the rest of the table at all. Same pass, same eligibility
checks, two very different outcomes — entirely a function of what else is in
the query, which is exactly the point this file has tried to make concrete
with real source and real tests throughout: `ROW_GROUP_PRUNER` never
degrades to wrong, only ever to no-better-than-a-plain-scan, and precisely
which of those two outcomes you get depends on facts about your query and
your table's layout that the pass checks explicitly rather than assumes.
