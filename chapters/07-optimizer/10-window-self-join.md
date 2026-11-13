# 7.10 — `WINDOW_SELF_JOIN`: turning a broadcast aggregate into a `GROUP BY` + join

*Source: `src/optimizer/window_self_join.cpp`, `src/include/duckdb/optimizer/window_self_join.hpp`.
`OptimizerType::WINDOW_SELF_JOIN`, pass #14 in §7.4's pipeline listing
(`optimizer.cpp:345`, "*Replaces some window computations with self-joins*").
Tests: `test/sql/optimizer/test_window_self_join.test`,
`test/sql/optimizer/window_self_join_optimizer_csv.test`. DiDi correspondence:
"Turning Window Computations Into Self-Joins", `window_self_join`
[10, slide 16].*

*This part also covers a related pass, `SELF_JOIN_AGGREGATE`
(pass #3, `src/optimizer/self_join_aggregate.cpp`). It goes the other way: it
turns a self-join into a `GROUP BY`. See §7.10.11.*

## 7.10.1 The shape of the problem

A window function like `SUM(l.extendedprice) OVER (PARTITION BY l.suppkey)`
does not, semantically, need anything a window function's general machinery
provides. It doesn't need rows sorted by an `ORDER BY` key (there isn't one).
It doesn't need a sliding frame that grows or shrinks row by row (the frame
is always "the whole partition," for every row). It needs exactly one number
per distinct value of `l.suppkey` — the sum of `extendedprice` for that
supplier — broadcast back onto every row that has that `suppkey`. That is
precisely what `GROUP BY l.suppkey` followed by a join back onto the
original rows computes, and it is a much narrower, much better-understood
computational problem than "general window function evaluation": DuckDB's
hash-aggregate operator (Chapter 14) builds one hash table over the group
keys and never needs to look at row order at all, where the general window
evaluator (Chapter 15's sorting chapter, forward-referenced here because it's
the mechanism this pass lets a query skip) has to sort or partition its input
by `PARTITION BY`/`ORDER BY` before it can process a single row, purely
because the general case — `RANK()`, a sliding `ROWS BETWEEN 3 PRECEDING`
frame, `LAG()` — genuinely does depend on row order.

`WindowSelfJoinOptimizer` (`window_self_join.cpp`) is the pass that notices
when a specific window function in a specific query is *not* using any of
that generality, and rewrites it into the cheaper shape directly, at the
logical-plan level, before the physical plan generator (Chapter 9) ever gets
a chance to hand it to the window-function executor at all.

A note on test conventions before diving in: this pair of `.test` files does
not use the paired-label `query I nosort <label>` idiom this book uses
elsewhere to prove two plans are byte-identical. Instead, every test that
wants to assert *whether the rewrite fired* runs `EXPLAIN` and matches the
`physical_plan` output against a `<REGEX>` — `.*Join Type: INNER.*` when the
rewrite is expected to have replaced the window with a join, `.*WINDOW.*`
when it's expected to have left a genuine window operator in the physical
plan. Separate, unlabeled `query` blocks alongside each `EXPLAIN` check the
actual result rows, so every test case answers two questions at once: did the
rewrite fire, and did it produce the right answer either way.

## 7.10.2 Eligibility, precisely: `CanOptimize(w_expr, w_expr0)`

The core question this pass has to answer for a given `BoundWindowExpression`
is: is this window function's frame *provably* equivalent to "the whole
partition, regardless of row order"? The answer lives in
`WindowSelfJoinOptimizer::CanOptimize` at `window_self_join.cpp:145`, and it
is worth walking every branch, because the precise conditions are more
interesting than "no `ORDER BY`" — and get some of them wrong if you guess.

```cpp
// src/optimizer/window_self_join.cpp:145
bool WindowSelfJoinOptimizer::CanOptimize(const BoundWindowExpression &w_expr,
                                          const BoundWindowExpression &w_expr0) const {
	if (w_expr.GetExpressionType() != ExpressionType::WINDOW_AGGREGATE) {
		return false;
	}
```

**Gate 1 — only `WINDOW_AGGREGATE`.** DuckDB's `ExpressionType` enum
(`src/include/duckdb/common/enums/expression_type.hpp:91-106`) gives every
window-function *kind* its own value: `WINDOW_AGGREGATE = 110` for an actual
aggregate function used with `OVER (...)`, and then two completely separate
families with their own numbers — the ranking family
(`WINDOW_RANK`, `WINDOW_RANK_DENSE`, `WINDOW_NTILE`, `WINDOW_PERCENT_RANK`,
`WINDOW_CUME_DIST`, `WINDOW_ROW_NUMBER`, values 120-125) and the navigational
family (`WINDOW_FIRST_VALUE`, `WINDOW_LAST_VALUE`, `WINDOW_LEAD`,
`WINDOW_LAG`, `WINDOW_NTH_VALUE`, `WINDOW_FILL`, values 130-135). This first
line categorically excludes every one of those eleven — `ROW_NUMBER()`,
`RANK()`, `LAG()`, `LEAD()`, and the rest never even reach the frame
reasoning below, because their `ExpressionType` isn't `WINDOW_AGGREGATE` in
the first place. This directly answers part (b) of what this pass covers:
the *function* allowlist is not a curated list of "simple" aggregates —
**it's every registered aggregate function in the whole catalog** (`SUM`,
`COUNT`, `AVG`, `MIN`, `MAX`, `LIST`/`ARRAY_AGG`, `STRING_AGG`, a
user-defined aggregate, a nested `AVG(AVG(x))` sitting on top of an existing
`GROUP BY` — anything that binds as `ExpressionType::WINDOW_AGGREGATE`,
which is every aggregate function used with `OVER (...)`), gated by nothing
function-name-specific. The exclusion is purely categorical: rank/ordinal
functions and navigational (`LEAD`/`LAG`/`FIRST_VALUE`/...) functions are a
different `ExpressionType` altogether and are rejected before any
function-identity check ever runs.

```cpp
	//	We can only accept ORDER BY clauses if the frame is the entire partition
	//	In that case, we will have to move the ordering clauses into the aggregate.
	//	ROWS framing is excluded because the frame depends on physical row position even without ORDER BY.
	switch (w_expr.WindowStart()) {
	case WindowBoundary::UNBOUNDED_PRECEDING:
		break;
	case WindowBoundary::CURRENT_ROW_RANGE:
	case WindowBoundary::CURRENT_ROW_GROUPS:
		if (!w_expr.OrderBy().empty()) {
			return false;
		}
		break;
	default:
		return false;
	}

	switch (w_expr.WindowEnd()) {
	case WindowBoundary::UNBOUNDED_FOLLOWING:
		break;
	case WindowBoundary::CURRENT_ROW_RANGE:
	case WindowBoundary::CURRENT_ROW_GROUPS:
		if (!w_expr.OrderBy().empty()) {
			return false;
		}
		break;
	default:
		return false;
	}
```

**Gate 2 — the frame, boundary by boundary.** This is the part the task
brief specifically asked to get right rather than guess at, and the answer
turns out to hinge on a detail of `WindowBoundary`'s own enum layout
(`src/include/duckdb/parser/expression/window_expression.hpp:18-31`):

```cpp
enum class WindowBoundary : uint8_t {
	INVALID = 0,
	UNBOUNDED_PRECEDING = 1,
	UNBOUNDED_FOLLOWING = 2,
	CURRENT_ROW_RANGE = 3,
	CURRENT_ROW_ROWS = 4,
	EXPR_PRECEDING_ROWS = 5,
	EXPR_FOLLOWING_ROWS = 6,
	EXPR_PRECEDING_RANGE = 7,
	EXPR_FOLLOWING_RANGE = 8,
	CURRENT_ROW_GROUPS = 9,
	EXPR_PRECEDING_GROUPS = 10,
	EXPR_FOLLOWING_GROUPS = 11
};
```

`UNBOUNDED_PRECEDING` and `UNBOUNDED_FOLLOWING` are **single, frame-type-agnostic
values** — there is no `UNBOUNDED_PRECEDING_ROWS` vs.
`UNBOUNDED_PRECEDING_RANGE` distinction, because "unbounded" means the same
thing no matter whether the frame clause says `ROWS`, `RANGE`, or `GROUPS`:
every row in the partition is included, full stop. `CURRENT ROW` and the
bounded `<expr> PRECEDING`/`<expr> FOLLOWING` forms, by contrast, *do* get
three separate enum values each, one per frame type, because their meaning
genuinely differs: `ROWS BETWEEN ... AND CURRENT ROW` means "up to this
row's physical position in the partition," while `RANGE`/`GROUPS BETWEEN ...
AND CURRENT ROW` means "up to (and including) every row that's a peer of the
current row under the `ORDER BY` key" — a value-based notion, not a
position-based one.

That single enum fact is what makes the switch above precise rather than
approximate. Walking it exactly:

- **`WindowStart() == UNBOUNDED_PRECEDING`** (or, symmetrically,
  **`WindowEnd() == UNBOUNDED_FOLLOWING`**) is accepted *unconditionally* —
  `break` falls straight through with no further check, **regardless of
  whether the frame clause says `ROWS`, `RANGE`, or `GROUPS`**, and
  regardless of whether an `ORDER BY` is present. This directly answers the
  brief's open question about `RANGE BETWEEN UNBOUNDED PRECEDING AND
  UNBOUNDED FOLLOWING`: yes, it's eligible — but so, less obviously, is
  **`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`**, since
  nothing in this switch can tell the two apart once both boundaries are the
  frame-type-agnostic "unbounded" value. Test case #12 in
  `test_window_self_join.test` proves exactly this: `list(train_number)
  OVER (PARTITION BY date ORDER BY train_number DESC ROWS BETWEEN UNBOUNDED
  PRECEDING AND UNBOUNDED FOLLOWING)` optimizes to an `INNER` join, `ROWS`
  keyword and all.
- **`CURRENT_ROW_RANGE` / `CURRENT_ROW_GROUPS`** are accepted **only when
  `w_expr.OrderBy().empty()`** — i.e., only when the window has *no* `ORDER
  BY` clause at all. This is not an arbitrary carve-out: under `RANGE` or
  `GROUPS` framing, "current row" means "every row in the current row's peer
  group," and a peer group is defined by equality on the `ORDER BY` key. With
  no `ORDER BY` key at all, every row in the partition is trivially a peer of
  every other row (there is nothing to distinguish them), so `RANGE`/`GROUPS
  BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` with no `ORDER BY` reduces,
  semantically, to "the whole partition" — exactly the same answer
  `UNBOUNDED FOLLOWING` would have given. The moment an actual `ORDER BY` is
  present, though, "current row's peer group" becomes a genuine subset of
  the partition that depends on where a specific row's value falls in the
  order — a running/cumulative computation this pass cannot fake with a
  single group-wide aggregate — so the `if (!w_expr.OrderBy().empty()) return
  false;` guard fires. The `foreach framing RANGE GROUPS` loop in the test
  file (lines 255-266) exercises exactly the no-`ORDER BY` case (`PARTITION
  BY grp {framing} BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, no `ORDER
  BY` anywhere) and confirms it optimizes; test case #6 (`count(*) OVER
  (PARTITION BY date ORDER BY train_number)` — the *default* frame once an
  `ORDER BY` is present is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT
  ROW`) confirms the opposite once an `ORDER BY` actually exists.
- **`CURRENT_ROW_ROWS`** has no case of its own in either switch, so it falls
  to `default: return false;` — **unconditionally**, whether or not an
  `ORDER BY` is present. This is the literal meaning of the inline comment
  *"ROWS framing is excluded because the frame depends on physical row
  position even without ORDER BY"*: under `ROWS` framing, "current row"
  always means "this row's physical/positional index within the partition,"
  which is meaningful even with no `ORDER BY` at all (rows still have *some*
  storage/arrival order), and that positional dependency is exactly the kind
  of thing a `GROUP BY` aggregate — which has no concept of row position —
  cannot reproduce. Test case #13 (`sum(seq) OVER (PARTITION BY grp ROWS
  BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`) is the regression covering
  this exact branch, and its `EXPLAIN` shows `.*WINDOW.*`, confirming the
  rewrite correctly declines. (This is worth spelling out because a version
  of this pass's history literally had a bug here: commit `ce171d5e68`,
  *"Issue #21592: Window Self-Join Framing — Don't apply the optimisation for
  ROWS framing,"* is the commit that introduced this exact carve-out and its
  test coverage, implying an earlier revision of this pass did *not*
  distinguish `ROWS`' current-row case from `RANGE`/`GROUPS`' and would have
  mis-optimized this shape.)
- **Any bounded offset** — `EXPR_PRECEDING_ROWS`, `EXPR_FOLLOWING_ROWS`,
  `EXPR_PRECEDING_RANGE`, `EXPR_FOLLOWING_RANGE`, `EXPR_PRECEDING_GROUPS`,
  `EXPR_FOLLOWING_GROUPS` (i.e., anything shaped like `3 PRECEDING` or `1
  FOLLOWING`) — falls to `default: return false;` in both switches. A
  sliding frame is, definitionally, not "the whole partition," and this pass
  makes no attempt to handle it.

So the precise answer to "is `ORDER BY` ever allowed": **yes**, but only in
the narrow case where the frame's *both* boundaries are the frame-type-agnostic
`UNBOUNDED PRECEDING`/`UNBOUNDED FOLLOWING` pair — at which point the
`ORDER BY` genuinely has no effect on *which rows* belong to the frame (every
row does, regardless of order), even though, as §7.10.4 covers, the `ORDER
BY` can still matter for what value an order-*sensitive* aggregate like
`LIST` computes over those rows, which is exactly why `TranslateAggregate`
goes out of its way to preserve it.

```cpp
	if (w_expr.Partitions().empty()) {
		return false;
	}
	if (w_expr.WindowExclude() != WindowExcludeMode::NO_OTHER) {
		return false;
	}
	if (!w_expr.PartitionsAreEquivalent(w_expr0)) {
		return false;
	}

	if (!IsOrderableDistinctAggregate(w_expr)) {
		return false;
	}

	return true;
}
```

**Gate 3 — `PARTITION BY` is mandatory.** `w_expr.Partitions().empty()`
rejects a window function with no `PARTITION BY` clause at all — `count(*)
OVER ()` in test case #5, which stays a genuine `LogicalWindow` (`.*WINDOW.*`
in `EXPLAIN`). There is nothing structurally impossible about rewriting a
partition-less window function the same way (it would be a `GROUP BY ()`,
a single group covering the whole table, joined back onto every row), but
this pass simply doesn't attempt that case — every eligible rewrite groups
by at least one real key.

**Gate 4 — no `EXCLUDE` clause.** `WindowExcludeMode` (`NO_OTHER`,
`CURRENT_ROW`, `GROUP`, `TIES`) governs SQL's `EXCLUDE CURRENT ROW`/`EXCLUDE
GROUP`/`EXCLUDE TIES` frame modifiers, each of which removes specific rows
*relative to the current row* from an otherwise-eligible frame. Since a
`GROUP BY` aggregate has no notion of "the current row" at all — it computes
one value per group, full stop — none of the three non-default exclude modes
can be reproduced this way, and only `NO_OTHER` (the default, "exclude
nothing") passes.

**Gate 5 — every window expression on this `LogicalWindow` must share the
same partition key.** `PartitionsAreEquivalent(w_expr0)` is checked against a
reference expression, `w_expr0` — the *first* window expression on this
operator (bound in `OptimizeInternal` below) — because a single `LogicalWindow`
node can carry several window functions computed side by side (one QUALIFY
or SELECT list can reference several `OVER (...)` clauses at once), and this
pass builds exactly *one* `GROUP BY` + join to replace the whole node, not
one per window expression. If two window expressions on the same node
partitioned by different keys, there would be no single `GROUP BY` that
could serve both, so the entire node is left alone unless *every* window
expression on it agrees on the partition key. Test case #10 (`max(...) OVER
(PARTITION BY date, train_number) = 100 AND count(*) OVER (PARTITION BY
date, train_number) = 1`) exercises the matching-partitions case — both
aggregates end up computed by the *same* `LogicalAggregate`, as two separate
output columns, joined back once.

**Gate 6 — `DISTINCT` + order-sensitivity: `IsOrderableDistinctAggregate`.**
This last gate (`window_self_join.cpp:54-80`) handles a genuinely subtle
correctness case: an aggregate that is both marked `DISTINCT` *and*
order-dependent (an aggregate like `LIST`/`ARRAY_AGG`, whose *value* depends
on the order its arguments are fed in, not just their multiset). `DISTINCT`
deduplicates by argument value *before* any ordering is applied; if the sort
key used isn't determined by (isn't a "functional dependency" of) the
already-deduplicated arguments, the order that survives deduplication is
ambiguous — which row's copy of a duplicated value "wins" the sort position
is not well-defined. `IsOrderableDistinctAggregate` first short-circuits
`true` for the overwhelmingly common case (the aggregate isn't order-dependent
at all, or isn't `DISTINCT`), and only for the narrow remaining case walks
`BaseSelectBinder::IsFunctionallyDependent` over every `ORDER BY`/`ArgOrders`
key to confirm each one is derivable from the aggregate's own arguments.
`test_window_self_join.test`'s case #12 has exactly the positive/negative
pair proving this: `list(DISTINCT train_number) OVER (PARTITION BY date
ORDER BY train_number DESC ...)` optimizes (the `ORDER BY` key,
`train_number`, *is* the aggregate's own argument — trivially functionally
dependent), while `list(DISTINCT train_number) OVER (PARTITION BY
train_number ORDER BY date DESC ...)` does not (the `ORDER BY` key, `date`,
is unrelated to the deduplicated argument, `train_number` — the check fails
and the window is correctly left alone, `.*WINDOW.*` in `EXPLAIN`).

A window aggregate can have two orderings at once. One is the *argument*
ordering inside the call, as in `list(x ORDER BY x)`. The other is the
*frame* ordering in `OVER (... ORDER BY i ...)`. Both orderings must pass the
check, unless the frame ordering is *redundant*. Checking only one of them is
not enough (DuckDB issue #27814).

The helper that decides "redundant" is `FrameOrderIsRedundant`
(`window_self_join.cpp:35-52`):

```cpp
//	Rows that tie on the argument ordering have equal arguments, so the frame ordering cannot change the result
static bool FrameOrderIsRedundant(const BoundWindowExpression &w_expr) {
	if (w_expr.ArgOrders().empty()) {
		return false;
	}

	vector<reference<Expression>> order_refs;
	order_refs.reserve(w_expr.ArgOrders().size());
	for (auto &order : w_expr.ArgOrders()) {
		order_refs.emplace_back(*order.expression);
	}
	for (auto &arg : w_expr.GetChildren()) {
		if (!BaseSelectBinder::IsFunctionallyDependent(arg, order_refs)) {
			return false;
		}
	}
	return true;
}
```

In plain words: if the argument ordering fully decides the argument values,
then two rows that tie on the argument ordering also have the same argument.
Swapping them changes nothing in the output. So the frame ordering cannot
matter, and the check can skip it. `IsOrderableDistinctAggregate` reads
(`window_self_join.cpp:66-77`):

```cpp
	bool in_args = true;
	for (auto &order : w_expr.ArgOrders()) {
		in_args &= BaseSelectBinder::IsFunctionallyDependent(order.expression, arg_refs);
	}
	if (FrameOrderIsRedundant(w_expr)) {
		return in_args;
	}
	for (auto &order : w_expr.OrderBy()) {
		in_args &= BaseSelectBinder::IsFunctionallyDependent(order.expression, arg_refs);
	}
```

Tests near the end of `test_window_self_join.test` show the three cases,
using `frame_ties`, a table of 8 rows with `p = 1`, `i = 0..7` and
`x = i % 4`:

| Window expression | Frame ORDER BY redundant? | Rewritten? |
|---|---|---|
| `list(DISTINCT x ORDER BY x) OVER (PARTITION BY p ORDER BY i ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)` | yes: `ORDER BY x` decides `x` | yes, `INNER` join |
| `list(x ORDER BY x) OVER (PARTITION BY p ORDER BY i DESC ROWS ...)` | yes; not `DISTINCT` anyway | yes, `INNER` join |
| `list(DISTINCT x ORDER BY x % 2) OVER (PARTITION BY p ORDER BY i DESC ROWS ...)` | no: `x % 2` does not decide `x`, and `i` is not a function of `x` | no (`<!REGEX>:.*Join Type: INNER.*`) |

The third query returns `[2, 0, 3, 1]` on every row. The values 0 and 2 tie
on `x % 2 = 0`. The frame ordering `i DESC` breaks the tie: 2 comes first,
because its first row in that order (`i = 6`) comes before 0's (`i = 4`).
With `DISTINCT`, which
copy of a value "counts" for the order is not well defined, so the pass
leaves this window alone.

## 7.10.3 Eligibility of the *input* below the window: `CanOptimize(const LogicalOperator&)`

A second, structurally separate eligibility check governs what the window
function is allowed to sit *on top of* (`window_self_join.cpp:195`):

```cpp
bool WindowSelfJoinOptimizer::CanOptimize(const LogicalOperator &op) {
	switch (op.type) {
	case LogicalOperatorType::LOGICAL_GET:
	case LogicalOperatorType::LOGICAL_EXPRESSION_GET:
	case LogicalOperatorType::LOGICAL_PROJECTION:
	case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
	case LogicalOperatorType::LOGICAL_DUMMY_SCAN:
	case LogicalOperatorType::LOGICAL_FILTER:
		if (!op.children.empty()) {
			return CanOptimize(*op.children[0]);
		}
		return true;
	default:
		break;
	}
	return false;
}
```

This recurses down `children[0]` only, and only through six operator types —
`LogicalGet`, `LogicalExpressionGet`, `LogicalProjection`,
`LogicalAggregate` (a plain `GROUP BY`), `LogicalDummyScan`, and
`LogicalFilter` — bottoming out successfully once it reaches one of those
types with no children left (a leaf scan). **Anything else anywhere in that
chain — a join, a set operation, another `LogicalWindow`, an `ORDER BY`/
`LIMIT` — makes this return `false` for the whole subtree**, and the window
sitting above it is left untouched. Concretely, this means:

- A window function sitting directly on a table scan, optionally behind a
  `WHERE` filter and/or a narrowing projection, is eligible (test case #11:
  `WHERE date > ... QUALIFY ...` still optimizes, since `FILTER` is on the
  allowed list).
- A window function sitting on top of an *already-computed* `GROUP BY`
  aggregate is eligible too — `LOGICAL_AGGREGATE_AND_GROUP_BY` is on the
  list — which is exactly what makes `test_window_self_join.test`'s `avg(avg(
  base_salary)) OVER (PARTITION BY dept_name) FROM foo GROUP BY dept_name`
  case optimize: the window's input is a `LogicalAggregate`, not a scan, and
  that's still an accepted shape.
- A window function sitting on top of *another, unrewritten* window operator
  is **not** eligible, because `LOGICAL_WINDOW` isn't in the switch's
  case list. This matters for the file's last correctness test — a
  three-deep stack of window functions (`COUNT(*) OVER (PARTITION BY id)`,
  then `LAG(total, 1, 0) OVER (PARTITION BY id)` on top of that, then
  `SUM(x) OVER (PARTITION BY id)` on top of *that*). Because
  `OptimizeInternal` recurses bottom-up (`window.children[0] =
  OptimizeInternal(std::move(window.children[0]), replacer)` runs *before*
  this window's own eligibility is checked — see §7.10.5), the innermost
  `COUNT(*)` window — sitting directly on a plain table scan — gets rewritten
  into a join first. The middle window, `LAG`, is a `WINDOW_LEAD` expression
  type, not `WINDOW_AGGREGATE`, so Gate 1 above rejects it outright and it
  stays a genuine `LogicalWindow`. The outermost `SUM(x) OVER (PARTITION BY
  id)` then finds its own child is that untouched `LogicalWindow` (from
  `LAG`) — an operator type this switch doesn't recognize — so it, too, is
  left alone. The test has no `EXPLAIN` assertion at all; it exists purely
  to confirm that a *partial* rewrite (one window converted, two left as
  windows) still produces the right value (`1  10.0`), which is a real
  correctness property worth testing given how much column-binding
  bookkeeping is involved in stitching a rewritten and an unrewritten window
  back together in the same plan.

## 7.10.4 Building the replacement plan: `OptimizeInternal`

Once both eligibility checks pass for a `LogicalWindow`, `OptimizeInternal`
(`window_self_join.cpp:213`) builds the replacement. It's worth reading in
the order the code actually executes it, because the sequencing (copy the
child *before* mutating anything, replace bindings locally before replacing
them globally) is what keeps a plan with two live copies of the same
subtree — and, in the nested-window case above, a mix of rewritten and
unrewritten windows — internally consistent.

**Step 0 — the volatile guard, unconditionally, before anything else:**

```cpp
	//	We cannot perform self-join when there are volatile functions below us.
	if (VolatileExpressionCounter::HasVolatiles(window)) {
		return op;
	}
```

`VolatileExpressionCounter` (`window_self_join.cpp:16-33`) is a
`LogicalOperatorVisitor` that walks `window` — which, by default visitor
behavior, includes every expression in every operator *below* it too — and
counts any expression whose `IsVolatile()` returns true. The reason this has
to be checked at all: the rewrite is about to build *two* independent copies
of the window's entire input subtree (one feeding the join directly, one
feeding the new `GROUP BY` via a deep copy), and a volatile expression —
`random()`, `nextval('some_sequence')`, `current_timestamp` inside a context
where it's re-evaluated per row — is not safe to evaluate twice and expect
the same answer, or even a *consistent* pair of answers, from the two
copies. `test_window_self_join.test`'s final block is the regression for
exactly this: `SELECT g, n, sum(n) OVER (PARTITION BY g) s FROM (SELECT
nextval('seq') g, i n FROM t)` — if the rewrite fired here, the sequence
would be advanced once by the copy feeding the join's left side and a
*second* time by the copy feeding the new aggregate, and the two sides'
`g` values would disagree, breaking the join entirely (rows would fail to
find their partition's aggregate row, or find the wrong one). Because the
guard fires, this window is left as a genuine `LogicalWindow`, a single
sequence advance per row is guaranteed, and the expected output (`g` running
`1..6` in lockstep with `n` running `0..5`) comes out correct.

**Step 1 — re-verify both eligibility checks** (`CanOptimize(*window.children[0])`
for the input shape, then `CanOptimize(w_expr, w_expr0)` for every window
expression on the node) — covered in full above.

**Step 2 — deep-copy the entire input subtree**, reusing machinery built for
a different pass entirely:

```cpp
	unique_ptr<LogicalOperator> copy_child;
	// Reuse the LogicalOperatorDeepCopy from CTE inlining, as the copying requirements are similar (copying an
	// operator subtree and replacing table indices)
	LogicalOperatorDeepCopy deep_copy(optimizer.binder, nullptr);
	try {
		copy_child = deep_copy.DeepCopy(window.children[0]);
	} catch (std::exception &ex) {
		// failed to copy the LHS - cannot run this optimizer
		return op;
	}
```

`LogicalOperatorDeepCopy` (`src/planner/logical_operator_deep_copy.cpp`) is
worth understanding precisely, because it explains both this pass's
robustness guard and, as §7.10.6 covers, why a CSV-backed scan gets its own
dedicated regression test. `LogicalOperatorDeepCopy::DeepCopy` does two
things in sequence:

```cpp
// src/planner/logical_operator_deep_copy.cpp:13
unique_ptr<LogicalOperator> LogicalOperatorDeepCopy::DeepCopy(unique_ptr<duckdb::LogicalOperator> &op) {
	auto copy = op->Copy(binder.context);
	VisitOperator(*copy);
	TableBindingReplacer replacer(table_idx_replacements, parameter_data);
	replacer.VisitOperator(*copy);
	return copy;
}
```

First, `op->Copy(binder.context)` — the base `LogicalOperator::Copy`,
**not** any hand-written per-operator clone method. And `LogicalOperator::Copy`
(`src/planner/logical_operator.cpp:283`) is, itself, a full **binary
serialize/deserialize round-trip**:

```cpp
unique_ptr<LogicalOperator> LogicalOperator::Copy(ClientContext &context) const {
	MemoryStream stream(Allocator::Get(context));
	SerializationOptions options;
	options.storage_compatibility = StorageCompatibility::Latest();
	BinarySerializer serializer(stream, options);
	// ...
	this->Serialize(serializer);
	// ...
	auto op_copy = BinaryDeserializer::Deserialize<LogicalOperator>(stream, context, parameters);
	return op_copy;
}
```

Every operator in the subtree — and every scan's bind data (whatever a
`TableFunction`'s `Serialize`/`Deserialize` pair chooses to persist) — has to
survive a genuine serialize-then-deserialize cycle for this to work. This is
exactly why the surrounding `try`/`catch` exists (per commit `62b6bc7581`,
*"Fix #21391: try / catch around LogicalOperator::Copy"*): if any operator
or bind-data object in the subtree isn't fully serializable,
`Serialize`/`Deserialize` throws (typically a `NotImplementedException`,
per `LogicalOperator::Copy`'s own catch clause), and rather than letting that
exception escape the optimizer entirely, this pass catches it and simply
declines the rewrite — a correctness-neutral fallback, not a crash.

Because `op->Copy()` alone would produce a structurally identical copy —
*same* `TableIndex` values throughout, which would make every column
binding in the copy collide with the original the moment both trees coexist
in the same plan — `LogicalOperatorDeepCopy::VisitOperator` walks the fresh
copy and, for every operator type that owns a `TableIndex` (`LogicalGet`,
`LogicalProjection`, `LogicalAggregate`'s three indices, `LogicalDummyScan`,
and so on — the exhaustive list is in
`src/planner/logical_operator_deep_copy.cpp:87-201`), generates a *brand-new*
index via `binder.GenerateTableIndex()` and records the old→new mapping.
`TableBindingReplacer` then does a second, full pass rewriting every
`BoundColumnRefExpression` in the copy to point at the new indices instead of
the old ones. The result: `copy_child` is a subtree that computes exactly
the same rows as `window.children[0]`, addressable by entirely fresh,
non-colliding table indices — which is precisely the same requirement
`CTE_INLINING` (§7.4, pass #2) has when it inlines the same CTE definition at
two different reference sites, which is why this pass's own comment
explicitly says it's reusing that machinery rather than maintaining a
parallel, hand-rolled version (as, per commit `dec05a1616`, *"Remove manual
operator remapping and use already existing deep copy,"* an earlier revision
of this pass in fact did).

**Step 3 — build a *local* binding replacer**, mapping the *original* child's
column bindings onto the *copy*'s:

```cpp
	ColumnBindingReplacer local_replacer;
	auto old_bindings = window.children[0]->GetColumnBindings();
	auto new_bindings = copy_child->GetColumnBindings();
	vector<ReplacementBinding> binding_replacements;
	for (idx_t i = 0; i < old_bindings.size(); ++i) {
		binding_replacements.emplace_back(old_bindings[i], new_bindings[i]);
	}
	local_replacer.replacement_bindings = std::move(binding_replacements);
```

This matters because the expressions that will become the new aggregate's
`GROUP BY` keys and aggregate arguments are built by *copying* the window's
own `partitions`/argument expressions — and those, as originally bound,
reference the **original** child's column bindings, not the deep copy's.
`local_replacer` is applied *only* to the newly built `LogicalAggregate`
(`local_replacer.VisitOperator(*agg_op)`, a few lines below), rebinding its
group/aggregate expressions to point at `copy_child`'s fresh table indices —
since `copy_child`, not the original, is what actually feeds this aggregate.

**Step 4 — build the `GROUP BY` aggregate itself:**

```cpp
	auto aggregate_index = optimizer.binder.GenerateTableIndex();
	auto group_index = optimizer.binder.GenerateTableIndex();

	vector<unique_ptr<Expression>> groups;
	vector<unique_ptr<Expression>> aggregates;

	// Create Aggregate Operator
	for (auto &part : partitions) {
		auto part_copy = part->Copy();
		groups.push_back(std::move(part_copy));
	}

	for (auto &expr : window.expressions) {
		auto &w_expr = expr->Cast<BoundWindowExpression>();
		aggregates.emplace_back(TranslateAggregate(optimizer.GetContext(), w_expr));
	}

	// args: group_index, aggregate_index, ...
	auto agg_op = make_uniq<LogicalAggregate>(group_index, aggregate_index, std::move(aggregates));

	agg_op->groups = std::move(groups);
	agg_op->children.push_back(std::move(copy_child));
	local_replacer.VisitOperator(*agg_op);
	agg_op->ResolveOperatorTypes();
```

The `groups` are literal copies of the window's own `PARTITION BY`
expressions. Each `aggregates` entry comes from `TranslateAggregate`
(`window_self_join.cpp:82-131`), which builds a plain `BoundAggregateExpression`
out of the window expression's own function, arguments, `FILTER` clause, and
`DISTINCT`-ness — discarding the `PARTITION BY`/frame machinery entirely
(that's what the surrounding `GROUP BY` + join now handles) but, critically,
**preserving `ORDER BY` when the aggregate function is itself order-dependent**:

```cpp
// src/optimizer/window_self_join.cpp:103-130
	const auto aggr_ordered = (agg_func.GetOrderDependent() == AggregateOrderDependent::ORDER_DEPENDENT);
	auto result = make_uniq<BoundAggregateExpression>(std::move(agg_func), std::move(children), std::move(filter),
	                                                  std::move(bind_info), aggr_type);
	result->SetReturnType(w_expr.GetReturnType());

	if (!aggr_ordered) {
		//	ORDER BY is a NOP, so drop it.
		return std::move(result);
	}

	if (w_expr.ArgOrders().empty() && w_expr.OrderBy().empty()) {
		return std::move(result);
	}

	//	The frame ordering breaks ties in the argument ordering
	result->GetOrderBysMutable() = make_uniq<BoundOrderModifier>();
	auto &orders = result->GetOrderBysMutable()->orders;
	for (auto &order : w_expr.ArgOrders()) {
		orders.emplace_back(order.Copy());
	}
	if (FrameOrderIsRedundant(w_expr)) {
		return std::move(result);
	}
	for (auto &order : w_expr.OrderBy()) {
		orders.emplace_back(order.Copy());
	}

	return std::move(result);
```

Two details of this function matter for correctness.

1. **Tie-breakers are kept.** The function copies the argument ordering *and*
   the frame ordering, not just one of them. The real window operator sorts
   by the frame ordering first, and then does a stable sort by the argument
   ordering. So the frame ordering decides the order of rows that tie on the
   argument ordering. A rewrite that dropped it could return tied rows in a
   different order (DuckDB issue #27814). So the new aggregate is ordered by
   the argument ordering, *then* the frame ordering (unless
   `FrameOrderIsRedundant` says the frame ordering cannot matter).
2. **The result type is kept.** The line
   `result->SetReturnType(w_expr.GetReturnType())` gives the aggregate the
   type of the window expression it replaces. Without it, a windowed `sum`
   over a `DECIMAL` could fail with *"Failed to bind column reference ...
   inequal types (DECIMAL(38,2) != HUGEINT)"*. Here is why. A filter that
   reads several bindings makes a statistics step run before this pass
   (`FilterStatisticsOptimizer`, called right after `FILTER_PUSHDOWN` at
   `optimizer.cpp:292-295`).
   That step can swap the `DECIMAL` sum for `sum_no_overflow`, which returns
   `HUGEINT`, while the window expression stays `DECIMAL(38,2)`. Taking the
   type from the swapped function would give the wrong type.

Two tests at the end of `test_window_self_join.test` pin both details.
The first uses a table with two rows that tie on the argument ordering:

```sql
-- test/sql/optimizer/test_window_self_join.test
CREATE TABLE arg_ties(id INT, key_s VARCHAR, payload_s VARCHAR, ord INT);
INSERT INTO arg_ties VALUES (1, 'b', 'b1_ord1', 1), (1, 'b', 'b2_ord2', 2);

SELECT id,
	STRING_AGG(payload_s, ',' ORDER BY key_s ASC) OVER (
		PARTITION BY id
		ORDER BY ord DESC
		ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
	) AS res
FROM arg_ties
ORDER BY id, ord;
----
1	b2_ord2,b1_ord1
1	b2_ord2,b1_ord1
```

Both rows have `key_s = 'b'`, so `ORDER BY key_s` ties. The frame ordering
`ord DESC` breaks the tie, so `b2_ord2` comes first. The second test is the
`DECIMAL` query:

```sql
CREATE TABLE decimals AS SELECT i % 7 AS k, (i % 50 + 1)::DECIMAL(12,2) AS d FROM range(1000) t(i);

SELECT (SELECT max(s) FROM (SELECT sum(d) OVER (PARTITION BY k) AS s FROM decimals)),
       (SELECT count(*) FROM range(10) r(x) WHERE x = 3 OR x IN (SELECT 1));
----
3664.00	2
```

This is the mechanism that reconciles "the frame ignores `ORDER BY`
entirely" with "the *aggregate*'s answer can still depend on order" for a
function like `LIST`: DuckDB's aggregate functions can carry their own
internal `ORDER BY` (the same feature that makes `list(x ORDER BY y)` a
valid, order-preserving aggregate outside of any window context at all), and
`TranslateAggregate` simply relocates the window's `ORDER BY` onto the
translated aggregate's own ordering clause. The resulting `BoundAggregateExpression`
computes `LIST(train_number ORDER BY train_number DESC)` as a plain,
order-respecting `GROUP BY` aggregate — one list per partition, correctly
ordered internally, with no window machinery involved at all. (When the
call has both an argument ordering and a frame ordering, the frame ordering
is appended after the argument ordering as a tie-breaker; see the list
above. Per its own
git history, commit `df711b905d`, *"Internal #6979: Window SelfJoin OrderBy
— Convert ORDER BYs with unbounded frames to ordered aggregates,"* is
precisely where this capability was added; it postdates the pass's original
`count(*)`-only scaffold, discussed in §7.10.7.)

`agg_op->ResolveOperatorTypes()` recomputes the aggregate's own output
types, and a defensive check follows:

```cpp
	if (agg_op->types.size() <= agg_op->groups.size()) {
		throw InternalException("LogicalAggregate types size mismatch");
	}
```

— an internal-bug guard (an aggregate with zero actual aggregate outputs
would be a construction error in this pass itself, not a state a user query
could trigger), not a user-facing error path.

**Step 5 — build the join, and this is where the shape of the rewrite is
decided:**

```cpp
	auto join = make_uniq<LogicalComparisonJoin>(JoinType::INNER);
	for (size_t i = 0; i < partitions.size(); ++i) {
		auto left_expr = partitions[i]->Copy();
		auto right_expr = make_uniq<BoundColumnRefExpression>(partitions[i]->GetReturnType(),
		                                                      ColumnBinding(group_index, ProjectionIndex(i)));
		join->conditions.push_back(
		    JoinCondition(std::move(left_expr), std::move(right_expr), ExpressionType::COMPARE_NOT_DISTINCT_FROM));
	}

	join->children.push_back(std::move(original_child));
	join->children.push_back(std::move(agg_op));
	join->ResolveOperatorTypes();
```

## 7.10.5 Why an `INNER` join — and specifically `IS NOT DISTINCT FROM`, not `=`

This directly answers the hint in the task brief, and the source confirms
the reasoning rather than contradicting it: the join is
`make_uniq<LogicalComparisonJoin>(JoinType::INNER)` — an ordinary equi-style
comparison join, **not** a `SEMI` join, `MARK` join, or anything else.

The reason is exactly the cardinality argument the brief anticipated. This
pass needs **every original row back**, and each one needs to carry its
*own group's* aggregate value as an actual output column — not merely "does
a matching group exist" (which is all a `SEMI` join could tell you, and a
`SEMI` join, by definition, projects only the *left* side's columns; it
structurally cannot attach a column computed on the right side, which is
exactly what `supp_total` or `total` in these queries needs to be). The
`LogicalAggregate` on the join's right side produces **exactly one row per
distinct partition-key combination** — that's what `GROUP BY` guarantees —
so for every original row on the left, the `INNER` join finds *exactly one*
matching row on the right (the one whose group key equals its own partition
key) and the join's output is one row per *original* row, each now carrying
its group's aggregate value alongside every column it already had. This is a
many-to-one "broadcast" join, not a fan-out: the multiplicity comes entirely
from the left (original-row) side, and the right (aggregate) side never
produces more than one match per group, so row count is preserved exactly —
which is the property this rewrite depends on for correctness (a window
function's output has the same cardinality as its input, one extra column
per row, and the join has to reproduce that exactly).

The join condition uses `ExpressionType::COMPARE_NOT_DISTINCT_FROM` — SQL's
`IS NOT DISTINCT FROM`, i.e. null-safe equality — rather than plain
`COMPARE_EQUAL`, and this is not an arbitrary choice either. `GROUP BY`
groups every `NULL` partition key into a single group (SQL's `GROUP BY`
treats all `NULL`s as equal to each other for grouping purposes), but a
plain `=` comparison never returns true when either side is `NULL` — `NULL
= NULL` evaluates to `NULL` (falsy), not `TRUE`. If the join condition used
ordinary `=`, every original row whose partition key was `NULL` would fail
to find *any* matching row in the aggregate — including the aggregate row
that `GROUP BY` computed specifically for the `NULL` group — and, because the
join is `INNER`, those rows would simply vanish from the result. `IS NOT
DISTINCT FROM` is exactly the null-safe operator that makes `NULL` join to
`NULL`, restoring parity with `GROUP BY`'s own null-grouping semantics. This
is precisely what test case #8 in `test_window_self_join.test` is a
regression for — its own comment states the intent directly (*"This tests
that COMPARE_NOT_DISTINCT_FROM is used for join conditions... NULL = NULL
returns NULL (falsy), but NULL IS NOT DISTINCT FROM NULL returns TRUE"*), and
the test data (two rows with `category = NULL`, sharing a partition, plus one
row with a genuinely unique `category = 'B'`) is constructed specifically so
that a plain-`=` join would silently and incorrectly drop the `NULL`-keyed
rows from an `INNER` join result, which the test's expected output (only
`5, 'B'` — the `NULL`-keyed rows correctly grouped together and correctly
excluded by the `= 1` predicate since their count is 2, not because they
fell out of the join) would catch immediately if the join condition
regressed to plain equality.

**Step 6 — redirect every reference to the old window's output columns**,
and hand the whole join back in place of the original `LogicalWindow`:

```cpp
	for (idx_t column_index = 0; column_index < window.expressions.size(); ++column_index) {
		ColumnBinding old_binding(window.window_index, ProjectionIndex(column_index));
		ColumnBinding new_binding(aggregate_index, ProjectionIndex(column_index));
		replacer.replacement_bindings.emplace_back(old_binding, new_binding);
	}

	return std::move(join);
```

Note this uses `replacer` — the *outer-scope* `ColumnBindingReplacer` passed
by reference through the whole recursive `OptimizeInternal` call chain
(distinct from the `local_replacer` used only inside step 3-4) — because
whatever expression, anywhere above this `LogicalWindow` in the tree,
referenced `(window.window_index, i)` (the window's `i`-th output column)
needs to be redirected to `(aggregate_index, i)` instead, and that
referencing expression could be arbitrarily far up the tree (inside a
`QUALIFY`-derived filter directly above, or inside a projection, an `ORDER
BY`, or another operator several levels higher). `Optimize`, the public entry
point (`window_self_join.cpp:136`), collects every such replacement across
every rewritten window in the plan and applies them **once**, in a single
whole-tree pass, right at the end:

```cpp
unique_ptr<LogicalOperator> WindowSelfJoinOptimizer::Optimize(unique_ptr<LogicalOperator> op) {
	ColumnBindingReplacer replacer;
	op = OptimizeInternal(std::move(op), replacer);
	if (!replacer.replacement_bindings.empty()) {
		replacer.VisitOperator(*op);
	}
	return op;
}
```

## 7.10.6 Why this is (usually) cheaper: the honest version

The intuition, stated carefully rather than over-claimed: DuckDB's general
window-function executor — `PhysicalWindow`/`PhysicalStreamingWindow`, from
`LogicalWindow` (Chapter 5 §5.4.7; physical operator choice in Chapter 9
§9.3.1) — has to handle the *entire* generality of the SQL window-function
model: arbitrary `PARTITION BY`/`ORDER BY` combinations, sliding frames that
change contents row by row, and order-dependent functions like `RANK()` or
`LAG()`. To do that correctly in the general case, its execution model
(covered end to end in Chapter 11's sink/source pipeline treatment) needs its
input properly partitioned and, for anything order-sensitive, **sorted** by
the `PARTITION BY`/`ORDER BY` keys first — sorting being the specific,
concrete mechanism Chapter 15 covers, and a real cost even when it's a
partition-local sort rather than a whole-table one. Even for a partition-only
aggregate with no real `ORDER BY` dependency, the general window evaluator
still has to build partitions, track frame boundaries per row (even a
constant "whole partition" frame), and drive its own per-row output logic —
machinery built to be correct across the whole feature surface, not
specifically tuned for "one aggregate value per group, broadcast to every
row."

A `GROUP BY` computed by `PhysicalHashAggregate` (Chapter 14), by contrast,
needs **no sort at all** — it builds a hash table keyed on the group columns
and folds every row into its group's running aggregate state as it streams
past, in whatever order rows happen to arrive. The join that broadcasts the
result back is an ordinary hash join (Chapter 9's `PhysicalHashJoin`
machinery): build a hash table over the (typically much smaller) aggregate
result — one row per group — and probe it once per original row.
Structurally, this replaces "sort/partition + per-row frame bookkeeping"
with "one hash-table build over a *small* (one row per group) input, plus
one hash-table probe per original row" — and the probe side of a hash join
is exactly the kind of embarrassingly-parallel, streaming operation the rest
of DuckDB's execution engine is built to do well.

It would be an overclaim to say this is *always* strictly cheaper — a
sufficiently well-tuned, partition-broadcast-specialized window
implementation could in principle match or beat a hash-aggregate-plus-join
for this exact shape, and this book has no controlled microbenchmark to cite
either way. What can be said with confidence, grounded in what's actually in
the source rather than asserted from outside it: DuckDB's own team built and
ships a dedicated optimizer pass specifically to perform this substitution
whenever it's semantically safe to do so (rather than, say, tuning the
window executor's own partition-broadcast fast path instead), which is
itself the strongest available signal that, for DuckDB's *particular*
implementations of both operators, the `GROUP BY` + join path measurably
wins often enough to be worth a dedicated, unconditional (no cost-based
"maybe don't" escape hatch, unlike, say, `TOP_N`'s cardinality-based decline
in §7.6.4) rewrite pass. No source comment in `window_self_join.cpp` states
an exact percentage or microbenchmark number, and this book is not going to
manufacture one; the honest statement is the structural argument above (no
sort, streaming hash aggregation, a hash-join probe instead of per-row frame
bookkeeping) plus the fact of the pass's unconditional existence.

One place this book's own secondary source is worth correcting explicitly,
in keeping with the front matter's stated policy: DiDi's own slide on this
pass, *"Turning Window Computations Into Self-Joins"* [10, slide 16]
(dated April 7 2026 against "DuckDB version 1.5" per its own pass-listing
slide), illustrates the rewrite with SQL pseudocode that reads:

```sql
WITH frame(part, agg) AS (
    SELECT c AS part, agg(e)
    FROM t
    GROUP BY c
)
SELECT agg
FROM t SEMI JOIN frame
    ON (c IS NOT DISTINCT FROM part)
```

Taken as literal SQL, this is not actually well-formed: a genuine `SEMI
JOIN` can only ever project columns from its left side (`t`), and `agg` is a
column of `frame`, the right side — you cannot `SELECT agg` out of a real
semi-join at all. Read charitably, "`SEMI JOIN`" here is DiDi's informal
shorthand for "keep only `t`'s rows that find a match, and carry `agg`
along" rather than a literal invocation of relational semi-join semantics —
but it's exactly the discrepancy this book's sourcing policy exists to catch
and call out rather than silently repeat: **the real `window_self_join.cpp`
builds `JoinType::INNER`, not a semi join**, precisely because an actual
semi join structurally cannot deliver what the rewrite needs (the
right-hand aggregate's own column, attached to the output). The `IS NOT
DISTINCT FROM` half of DiDi's illustration, on the other hand, matches the
real source exactly (§7.10.5's own citation of `COMPARE_NOT_DISTINCT_FROM`).
Separately, DiDi's slide states the eligibility condition more narrowly than
the checked-out source actually implements — *"frames that are exclusively
specified by `PARTITION BY`"* — where the real `v2.0-cyanoptera` source (per
§7.10.2's exhaustive walk of `CanOptimize`) also accepts an `ORDER BY`
whenever the frame's own boundaries make it irrelevant. The slide deck's
header names "DuckDB version 1.5" as its baseline. This book's checkout is
v2.0.0 plus 1,041 commits (front matter, "What version of DuckDB, precisely"). So this is
exactly the kind of
gap the front matter already anticipated: DiDi's slide describes an earlier,
simpler cut of this pass, and the `ORDER BY`-tolerant generalization (commit
`df711b905d`, "Internal #6979: Window SelfJoin OrderBy") is one of several
capability additions that landed afterward. Neither correction diminishes
DiDi's slide as a teaching artifact — the core mechanic (`GROUP BY` + join
back on the partition key, using null-safe equality) is exactly right — but
the source, not the slide, is what this book's own stated methodology
requires trusting when the two disagree.

## 7.10.7 Where this pass came from: `CountWindowElimination`

One curiosity worth flagging, both because it's directly visible in the
checked-out source and because it explains this pass's shape: the header
file itself, `src/include/duckdb/optimizer/window_self_join.hpp`, still
carries its *original* file-path comment at the top:

```cpp
//===----------------------------------------------------------------------===//
//                         DuckDB
//
// duckdb/optimizer/count_window_elimination.hpp
//
//
//===----------------------------------------------------------------------===//
```

— a leftover from before the file (and the class, and the optimizer pass)
were renamed. The git history in this checkout confirms the whole arc:
the earliest commit touching this file is literally titled *"feat(optimizer):
Scaffold CountWindowElimination rule"* (`1c329ee1c7`), followed by *"feat(optimizer):
Implement CountWindowElimination logic"* and *"Issue #20076: Implement
CountWindowElimination Logic"* — i.e., this pass began life narrowly scoped
to exactly the shape test case #1 in `test_window_self_join.test` still
exercises, `count(*) OVER (PARTITION BY ...) = N`, the "does this row belong
to a group of exactly size `N`" idiom. From there, a sequence of commits
progressively generalized it: *"Internal #6974: Window Self-Join Files"*,
*"Internal #6975: Window Inner Self-Join"* (settling on `INNER`, not `SEMI`,
as the join type), *"Internal #6976: Window Self-Join Predicate"* (beyond a
literal `= N` predicate to any `QUALIFY`/`WHERE` shape), *"Internal #6977:
Window Self-Join Aggregate"* (generalizing from `COUNT(*)` specifically to
any `WINDOW_AGGREGATE`), *"Internal #6978: Window Self-Join Multiple"*
(multiple window expressions sharing one `LogicalWindow`, §7.10.2's Gate 5),
and *"Internal #6979: Window SelfJoin OrderBy"* (the `ORDER BY`-with-whole-partition-frame
generalization walked in §7.10.2). Later commits hardened it: *"Fix #21391:
try / catch around LogicalOperator::Copy"* (§7.10.4's Step 2 guard), *"Issue
#21592: Window Self-Join Framing"* (the `ROWS`-framing carve-out, §7.10.2),
and *"Remove manual operator remapping and use already existing deep
copy"* — the refactor that gave the pass its current, `LogicalOperatorDeepCopy`-based
implementation in place of an earlier, hand-rolled column-rebinding
mechanism. Two later fixes kept the frame `ORDER BY` as a tie-breaker
(issue #27814) and kept the `DECIMAL` type of a windowed sum, both covered in
§7.10.4. The pass's present generality — any aggregate, any compatible
frame shape, multiple window expressions at once, `DISTINCT`+order-dependent
correctness checking — is the accumulated result of that whole sequence, not
a single design that shipped complete; the header comment is simply the one
piece of the old name nobody has yet updated.

## 7.10.8 The CSV-specific test, and why it likely exists on its own

`test/sql/optimizer/window_self_join_optimizer_csv.test` is short — four
statements, testing exactly one thing:

```sql
-- test/sql/optimizer/window_self_join_optimizer_csv.test
statement ok
COPY (
    SELECT 'a' AS x, 10 AS a, 20 AS b, 30 AS c
    UNION ALL BY NAME
    SELECT 'b' AS x, 11 AS a, 21 AS b, 31 AS c
    UNION ALL BY NAME
    SELECT 'c' AS x, 12 AS a, 22 AS b, 32 AS c
) TO '{TEST_DIR}/window_self_join_src.csv'

query II rowsort
SELECT
    x, MAX(a) OVER (PARTITION BY x)
FROM '{TEST_DIR}/window_self_join_src.csv'
----
a	10
b	11
c	12
```

There's no comment in the file (its only two commits, per `git log --follow`,
are the original *"Add a test"* and a later mechanical `__TEST_DIR__` →
`{TEST_DIR}` rename) explaining why this couldn't simply have been folded
into `test_window_self_join.test` as one more block against an in-memory
table. Reasoning from what §7.10.4 already established about how this pass
actually copies a subtree, though, points at a specific, plausible answer,
stated here with the same honesty about confidence this book applies
elsewhere: **`LogicalOperator::Copy` is a full binary serialize/deserialize
round-trip**, and that round-trip has to faithfully reconstruct *everything*
a `LogicalGet`'s bind data needs to actually re-scan correctly — including
whatever a table function's own `FunctionData::Serialize`/`Deserialize`
implementation chooses to persist. For an ordinary catalog table, that bind
data is trivial (essentially, "which table, by catalog OID" — re-resolved
cheaply and deterministically on deserialize). A CSV scan's bind data is
categorically richer: CSV has no embedded schema at all, so
`ReadCSVData`/`CSVReaderOptions` (`src/function/table/read_csv.cpp`,
`src/include/duckdb/function/table/read_csv.hpp`) carries the *result of
sniffing the file* — inferred column names and types, delimiter/quote/escape
detection, and multi-file expansion state — none of which exists anywhere
except as this bind data itself, reconstructed once at bind time and never
re-derived. Tellingly, `read_csv.cpp` defines a distinct
`SerializedCSVReaderOptions` type specifically for this purpose
(`read_csv.cpp:39`), suggesting the CSV function's own runtime options
object isn't directly, losslessly serializable as-is and needs its own
dedicated serialization shape. If that round-trip ever dropped or
mis-reconstructed any of the sniffed schema information, the **second**
copy of the scan this pass creates (the one feeding the new `GROUP BY`,
built via `LogicalOperatorDeepCopy`, which is exactly `LogicalOperator::Copy`
plus table-index rebinding) could end up with a different inferred type or
column set than the original scan still feeding the join's other side —
producing a type mismatch, a wrong answer, or a crash, in a way an in-memory
table's trivially-serializable bind data structurally cannot exercise. This
book cannot point to a comment or commit message that states this
motivation explicitly, so it's offered as the most concrete, source-grounded
explanation available rather than a confirmed fact: a CSV-backed scan is the
one common built-in table function whose bind data is rich enough, and
derived from file content rather than catalog metadata, to make "does
`Copy()`'s serialize round-trip faithfully preserve everything a second,
independent scan needs" a real question worth its own dedicated regression,
separate from the in-memory-table-only coverage the rest of the suite
provides.

## 7.10.9 A full worked example, start to finish

None of the tests above show the complete before/after logical-plan shape
end to end with real table indices, so here is a query constructed in the
same spirit as this book's Query A/B (front matter) — call it **Query C**,
introduced here and used only in this section:

```sql
SELECT
    l.suppkey,
    l.extendedprice,
    SUM(l.extendedprice) OVER (PARTITION BY l.suppkey) AS supp_total
FROM lineitem l
WHERE l.shipdate >= DATE '1998-01-01';
```

This is deliberately close to Query A's own `lineitem`/`extendedprice` shape
(same schema, same book) but reframed as a window-function query: for every
qualifying line item, attach the *total* `extendedprice` across every line
item from the same supplier, alongside that row's own values — the "broadcast
a per-supplier total onto every one of that supplier's rows" pattern this
whole pass exists to accelerate.

**Before — the plan `WINDOW_SELF_JOIN` receives** (a `LogicalFilter` sits
directly beneath the window, itself sitting on the base scan — exactly the
`FILTER` → `GET` chain §7.10.3 confirmed is an eligible input shape):

```
LogicalProjection (suppkey, extendedprice, supp_total)
  └─ LogicalWindow (window_index=#3; SUM(l.extendedprice) OVER (PARTITION BY l.suppkey))
       └─ LogicalFilter (l.shipdate >= DATE '1998-01-01')
            └─ LogicalGet (lineitem, table_index=#1)
```

**After** — `WindowSelfJoinOptimizer::OptimizeInternal` deep-copies the
*entire* `LogicalFilter` → `LogicalGet` chain (not just the `LogicalGet`
alone — the deep-copy operates on `window.children[0]` as a whole subtree),
assigning it fresh table indices (`#1` becomes `#4`), builds a
`LogicalAggregate` over that copy, and replaces the `LogicalWindow` node with
an `INNER` join between the *original*, untouched chain and the new
aggregate:

```
LogicalProjection (suppkey, extendedprice, supp_total)
  └─ LogicalComparisonJoin (INNER; l.suppkey IS NOT DISTINCT FROM group_key)
       ├─ LogicalFilter (l.shipdate >= DATE '1998-01-01')            [original — table_index #1, untouched]
       │    └─ LogicalGet (lineitem, table_index=#1)
       └─ LogicalAggregate (group_index=#5, aggregate_index=#6;
                            GROUP BY l.suppkey;  SUM(l.extendedprice))
            └─ LogicalFilter (l.shipdate >= DATE '1998-01-01')       [deep copy — table_index #4, fresh]
                 └─ LogicalGet (lineitem, table_index=#4)
```

Every reference in the top `LogicalProjection` to the window's old output
column, `(window_index=#3, 0)` — the `supp_total` expression — has been
rewritten by the final, whole-tree `ColumnBindingReplacer` pass to point at
`(aggregate_index=#6, 0)` instead, the new aggregate's own `SUM` output
column, exactly as §7.10.5's Step 6 describes.

Two things worth pointing out about this specific shape, both direct
consequences of mechanics covered above:

- `lineitem` is genuinely scanned **twice** in the rewritten plan — once
  through the original chain (table index `#1`), once through the deep copy
  (table index `#4`) — each independently applying the `shipdate` filter.
  This is the concrete cost this rewrite is trading against the general
  window evaluator's sort/partition cost: an extra scan-and-filter pass over
  `lineitem`, versus one sort. Which one wins depends on how selective the
  filter is and how much of `lineitem` survives it — exactly the kind of
  judgment call §7.10.6 declined to resolve with an invented number, and
  exactly why filter pushdown (§7.6.3, passes #5/#6) already ran *before*
  this pass in the fixed pipeline (§7.10.10 below) — both scans get the
  `shipdate` predicate pushed into their own `LogicalGet`, so neither copy
  ever reads a row that couldn't possibly qualify.
- The new `LogicalAggregate`'s child is the **entire** deep-copied
  `LogicalFilter` → `LogicalGet` chain, not a bare re-scan of `lineitem`
  alone — confirming, in a fully worked example rather than just the test
  suite's terser cases, that `CanOptimize(const LogicalOperator&)`'s
  linear-chain requirement (§7.10.3) really does need to hold for the
  *whole* subtree being duplicated, filter and all.

## 7.10.10 Pipeline position: what has to run before, and why

`WINDOW_SELF_JOIN` is pass #14 in §7.4's fixed sequence
(`optimizer.cpp:345`), which places it **after** `FILTER_PULLUP`/`FILTER_PUSHDOWN`
(#5/#6) and `EMPTY_RESULT_PULLUP` (#13), but **before** `JOIN_ORDER` (#17) —
and both placements matter for exactly the reasons Query C's worked example
already surfaced:

- **Running after filter pushdown, not before it**, means that by the time
  this pass duplicates a `LogicalFilter` → `LogicalGet` chain (as in Query
  C), any filter that could be pushed all the way to the scan already has
  been — both the original chain *and* the deep copy inherit the
  narrower, pushed-down scan, rather than each copy separately needing its
  own later pushdown pass to reach the same state. Had this pass instead run
  *before* filter pushdown, the deep-copied chain would carry an
  un-pushed-down `LogicalFilter`, and the fixed, one-shot pipeline
  architecture (§7.2 — no pass gets a second turn) offers no later chance to
  push it down a second time into the *newly created* copy specifically,
  since filter pushdown has already had its one turn by then. Running this
  pass after guarantees both scans this pass creates start from an
  already-optimized filter position.
- **Running before `JOIN_ORDER`** means the `LogicalComparisonJoin` this pass
  introduces is not invisible to the join-order search — it becomes one more
  join edge DPhyp's hypergraph (Chapter 8) can reason about and place
  optimally relative to whatever other joins the query already had. A query
  combining an explicit join with a partition-only window function (say,
  Query C rewritten to also join `lineitem` against `supplier`) would, after
  this pass runs, present `JOIN_ORDER` with *two* joins to order — the
  original `lineitem`/`supplier` join and this pass's newly introduced
  `lineitem`/(aggregate) join — rather than hiding the second one inside an
  opaque `LogicalWindow` node that `JOIN_ORDER` has no vocabulary to reorder
  around at all. Had the ordering been reversed (`JOIN_ORDER` before
  `WINDOW_SELF_JOIN`), the join this pass introduces would simply never
  participate in that search, missing whatever build-side/probe-side or
  join-order benefit `JOIN_ORDER` and its downstream companions
  (`AGGREGATE_REUSE`, `PARTIAL_AGGREGATE_PUSHDOWN`, `BUILD_SIDE_PROBE_SIDE`)
  could otherwise have applied to it.
- Its position immediately after `EMPTY_RESULT_PULLUP` (#13) and immediately
  before `PROJECTION_PULLUP` (#15) is otherwise unremarkable for this pass
  specifically — neither neighbor's own mechanics depend on window-function
  shapes, and this pass's own output (an ordinary join plus an ordinary
  aggregate) is exactly the vocabulary every later structural pass in the
  pipeline already knows how to reason about, which is precisely the point:
  after this pass runs, nothing downstream needs to know a window function
  was ever there at all.

## 7.10.11 `SELF_JOIN_AGGREGATE`, the rewrite in the other direction

*Pass #3 in §7.4's pipeline listing. Source: `src/optimizer/self_join_aggregate.cpp`,
`src/include/duckdb/optimizer/self_join_aggregate.hpp`.
`OptimizerType::SELF_JOIN_AGGREGATE` (`optimizer_type.hpp:64`, name
`self_join_aggregate` in `disabled_optimizers`). Test:
`test/sql/optimizer/self_join_aggregate.test`.*

`WINDOW_SELF_JOIN` turns a window into a `GROUP BY` plus a self-join.
`SELF_JOIN_AGGREGATE` does nearly the opposite. It finds a self-join that
only exists to answer a group question, and replaces it with one `GROUP BY`
over a single scan. The two passes are not related in code. They sit
together in this part because both trade a self-join for a `GROUP BY`.

### 7.10.11.1 The problem: TPC-DS query 95

The motivating query is TPC-DS q95. The source file opens with it
(`self_join_aggregate.cpp:1-10`):

```cpp
// A self-join of one table on equal keys K and on `a.x <> b.x` for one more column x, read only through K by a parent
// that ignores duplicates, is `GROUP BY K HAVING min(x) <> max(x)` over a single scan. TPC-DS q95's ws_wh is the case:
//
//     WITH ws_wh AS (SELECT ws1.ws_order_number, ws1.ws_warehouse_sk wh1, ws2.ws_warehouse_sk wh2
//                    FROM web_sales ws1, web_sales ws2
//                    WHERE ws1.ws_order_number = ws2.ws_order_number AND ws1.ws_warehouse_sk <> ws2.ws_warehouse_sk)
//     ... ws_order_number IN (SELECT ws_order_number FROM ws_wh)
//
// The join pairs every order's rows with each other only to learn which orders ship from more than one warehouse;
// the group-by reads web_sales once and keeps one row per order.
```

The CTE `ws_wh` joins `web_sales` with itself. For one order with *n* rows,
the join makes up to *n × n* pairs. But the query only uses `ws_wh` inside
`IN (...)`. It only asks "which order numbers appear?". The answer is: the
orders that have at least two different warehouses. A `GROUP BY` finds those
in one pass:

```sql
SELECT ws_order_number FROM web_sales
WHERE ws_order_number IS NOT NULL
GROUP BY ws_order_number
HAVING min(ws_warehouse_sk) <> max(ws_warehouse_sk)
```

The commit message reports q95 at TPC-DS scale factor 3 on 8 threads going
from 45 ms to 17 ms. It also says the gain grows with the number of rows per
key, because the join's output grows with the square of that number.

### 7.10.11.2 Why the rewrite gives the same answer

The rewrite is only safe when two things hold. The output must be the same
*set* of keys, and the parent must not care how often each key appears.

**The same set of keys.** The source states the argument
(`self_join_aggregate.cpp:12-17`):

```cpp
// Why it is exact. The join's rows are the pairs (r1, r2) of rows with the same non-NULL K (`=` drops NULL keys) and
// non-NULL, different x (`<>` drops NULL x). The set of their K values is the set of K values of the non-NULL groups
// that hold two different non-NULL x, which is `min(x) <> max(x)` over the group (min and max skip NULLs; a group
// whose x are all NULL gives NULL <> NULL, which the HAVING drops). So the two produce the same SET of K, and a parent
// that reads only K and ignores how often each row occurs cannot tell them apart. The NULL key group is filtered out
// first: in the join no NULL key pairs with anything, and an IN over a NULL would turn a FALSE marker into NULL.
```

Take the small table from the test file:

```sql
-- test/sql/optimizer/self_join_aggregate.test
CREATE TABLE n (k INTEGER, x INTEGER);
-- key 1: one value and NULLs; key 2: two values and a NULL; key 3: NULLs only; key NULL: two values; key 4: one row
INSERT INTO n VALUES (1, 10), (1, NULL), (1, 10), (2, 20), (2, 21), (2, NULL), (3, NULL), (3, NULL), (NULL, 30), (NULL, 31), (4, 40);
```

Group by group:

```
 k     x values         join a.k=b.k AND a.x<>b.x     min(x) <> max(x)
 ----  ---------------  ----------------------------  ---------------------------
 1     10, NULL, 10     no pair (10 = 10, NULL drops)  10 <> 10   -> false
 2     20, 21, NULL     (20,21), (21,20)               20 <> 21   -> true
 3     NULL, NULL       no pair                        NULL<>NULL -> NULL (dropped)
 NULL  30, 31           no pair (NULL = NULL is NULL)  must be removed by IS NOT NULL
 4     40               no pair                        40 <> 40   -> false
```

Both sides give the set `{2}`. The `NULL` key needs a separate filter. Its
group has two different values, so `min <> max` would be true. But the join
never pairs `NULL` keys, because `NULL = NULL` is not true. So the rewrite
adds `k IS NOT NULL` below the `GROUP BY`. This matters for `NOT IN`. If a
`NULL` were in the subquery result, `x NOT IN (...)` would return `NULL`
instead of `false` for every non-matching row. The test runs each query with
the pass off and on and expects the same rows:

```sql
SELECT k, k IN (SELECT a.k FROM n a, n b WHERE a.k = b.k AND a.x <> b.x) FROM (VALUES (1), (2), (3), (4), (NULL)) v(k) ORDER BY 1
----
1	false
2	true
3	false
4	false
NULL	NULL
```

**A parent that ignores duplicates.** The join gives key 2 twice (two
pairs). The `GROUP BY` gives it once. So the parent must be one where "once"
and "twice" give the same result. The source lists these parents
(`self_join_aggregate.cpp:19-27`):

```cpp
// Which parents ignore duplicates, checked from the root down (Walk): the right side of a SEMI, ANTI or MARK join (and
// the left side of a RIGHT_SEMI or RIGHT_ANTI one), whose rows only decide whether a match exists -- a MARK join's NULL
// included, which depends on whether a NULL occurs, not how often; a DISTINCT, whose targets it reads through
// expressions, a DISTINCT ON passing its other columns through by position, which therefore all count as read; and a
// GROUP BY whose aggregates are min, max or DISTINCT ones (`min(x, n)`, the n smallest values, counts duplicates and
// is not one of them). Below such a parent the property passes through projections and filters without volatile
// expressions, cross products and INNER joins: their output, as a set, depends only on their inputs as sets. Anything
// else stops it, a UNION among them: it reads its inputs' columns by position rather than through expressions, so what
// it reads would not be seen. A materialized CTE's definition ignores duplicates when every one of its references does.
```

An `IN (subquery)` has already become a `MARK` or `SEMI` join by the time the
optimizer runs (Chapter 6), so q95's `IN` is a "right side of a MARK or SEMI
join" parent.

### 7.10.11.3 Finding candidates: `Walk`

The pass works in two steps. First, `Walk` goes down the plan from the root
and carries one flag, `ignores`. The flag says "the parent of this operator
does not care about duplicate rows". Every `LogicalProjection` reached with
`ignores = true` becomes a *candidate*. Here is the core of `Walk`
(`self_join_aggregate.cpp:300-346`):

```cpp
	void Walk(unique_ptr<LogicalOperator> &slot, bool ignores) {
		auto &op = *slot;
		switch (op.type) {
		case LogicalOperatorType::LOGICAL_PROJECTION:
			if (ignores) {
				candidates.push_back(Candidate {slot, nullptr});
			}
			Walk(op.children[0], ignores && !AnyVolatile(op.expressions));
			return;
		case LogicalOperatorType::LOGICAL_FILTER:
			Walk(op.children[0], ignores && !AnyVolatile(op.expressions));
			return;
		case LogicalOperatorType::LOGICAL_CROSS_PRODUCT:
			for (auto &child : op.children) {
				Walk(child, ignores);
			}
			return;
		case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
		case LogicalOperatorType::LOGICAL_ANY_JOIN: {
			const bool deterministic = !JoinConditionsVolatile(op);
			bool left = false;
			bool right = false;
			switch (op.Cast<LogicalJoin>().join_type) {
			case JoinType::INNER:
				left = right = ignores;
				break;
			case JoinType::SEMI:
			case JoinType::ANTI:
			case JoinType::MARK:
				left = ignores;
				right = true;
				break;
			case JoinType::RIGHT_SEMI:
			case JoinType::RIGHT_ANTI:
				left = true;
				right = ignores;
				break;
			default:
				break;
			}
			Walk(op.children[0], left && deterministic);
			Walk(op.children[1], right && deterministic);
			return;
		}
		case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
			Walk(op.children[0], AggregateIgnoresDuplicates(op.Cast<LogicalAggregate>()));
			return;
```

The rest of the switch (`self_join_aggregate.cpp:347-382`) handles three
more cases:

- `LOGICAL_DISTINCT` sets `ignores = true` for its child.
- A materialized CTE walks its *references* first (they are all under its
  second child). A `LOGICAL_CTE_REF` records whether *its* parent ignores
  duplicates. The CTE's definition only counts as a candidate when *every*
  reference ignores duplicates. The test shows a CTE read once by `count(*)`
  and once by `IN`; neither use is rewritten, because the CTE is shared.
- Any other operator (a `UNION`, a `LEFT` join, a window, ...) walks its
  children with `ignores = false`.

`AggregateIgnoresDuplicates` (`self_join_aggregate.cpp:94-112`) accepts a
`GROUP BY` only when every aggregate is `DISTINCT`, or is a one-argument
`min` or `max`. So `SELECT min(o), max(o), count(DISTINCT o) FROM (self-join)`
is rewritten, but `SELECT count(o) FROM (self-join)` is not.

Second, `Run` (`self_join_aggregate.cpp:271-296`) finds which columns of
each candidate are actually read above it. It uses a helper visitor,
`LiveBindings`. Then it calls `Rewrite` on each candidate, deepest first.

### 7.10.11.4 Checking the join: `Rewrite`

`Rewrite` (`self_join_aggregate.cpp:386-566`) checks the shape below the
candidate projection. The conditions, as the source comment states them
(`self_join_aggregate.cpp:29-39`):

```cpp
// What must hold of the join, in the plan as bound (the rewrite runs before filter pushdown, which would move the
// conditions into the scans; the later optimizers then push the new filter into the scan and order the joins around
// the group-by as for any other): a projection over filters over a cross product or an INNER comparison join of two
// scans of the same relation -- the same table, or the same function with equal bind data and marked repeatable, so
// that the two scans return the same rows, read the same way, without table filters, time travel, sampling or
// ordinality -- and every condition of the filters and the join a `=` or `<>` between the same column of the two
// scans. The `=` columns are K, at least one; the `<>` column is x, exactly one and not in K (with two the pairs must
// differ in both, which no per-column test says). K and x are integral or DATE columns, whose ordering agrees with
// their equality; a collated VARCHAR's need not. The projection's columns read above it (through the references of
// the CTE it defines, when it does) must each be one of K: those then come from the group-by and the others, which
// nothing reads, become typed NULLs, so a CTE keeps the columns its references expect.
```

As a checklist:

1. Below the projection: zero or more `LogicalFilter`s, then a
   `LogicalCrossProduct` or an `INNER` `LogicalComparisonJoin`.
2. The join's two children are both `LogicalGet`s of the **same relation**
   (`SameRelation`, `self_join_aggregate.cpp:196-213`). Both must be plain
   scans (`PlainScan`, `:172-175`): no table filters, no `AT` time travel, no
   dynamic filters, no sample, no ordinality. For a table function, both
   scans must have equal parameters and bind data, and the function must be
   marked repeatable. The test shows `duckdb_types()` joined with itself is
   *not* rewritten, because that function is not marked repeatable.
3. Every predicate and join condition is `=` or `<>` between **the same
   column** of the two scans. `a.w <> b.p` (two different columns),
   `a.w <> b.w + 1` (an expression), and `a.v > 5` (one side only) all stop
   the rewrite.
4. At least one `=` column (these are the keys K) and **exactly one** `<>`
   column (this is x), and x is not one of the keys. With two `<>`
   conditions, a pair must differ in *both* columns. `min <> max` per
   column cannot say that.
5. Every compared column is an integer type or `DATE`
   (`OrderMatchesEquality`, `:237-239`). For these types, `min <> max` means
   exactly "there are two different values". The test shows `VARCHAR` and
   `DOUBLE` columns are refused. For a `VARCHAR` with a collation, two
   strings can be "equal" but still have different `min` and `max`.
6. Every column of the projection that is read above must be a key, from
   either scan.

### 7.10.11.5 Building the replacement

When all checks pass, `Rewrite` builds the new subtree
(`self_join_aggregate.cpp:509-547`). It keeps the **left** scan and drops
the right one:

```cpp
		// scan WHERE key IS NOT NULL ... GROUP BY keys [min(x), max(x)] HAVING min(x) <> max(x)
		auto scan_index = left.table_index;
		auto scan = std::move(join.children[0]);
		auto not_null = make_uniq<LogicalFilter>();
		for (auto &key : keys) {
			auto check = make_uniq<BoundOperatorExpression>(ExpressionType::OPERATOR_IS_NOT_NULL, LogicalType::BOOLEAN);
			check->GetChildrenMutable().push_back(
			    make_uniq<BoundColumnRefExpression>(key.type, ColumnBinding(scan_index, key.left)));
			not_null->expressions.push_back(std::move(check));
		}
		not_null->AddChild(std::move(scan));
```

Then it binds `min(x)` and `max(x)` as real aggregate functions, builds a
`LogicalAggregate` that groups by every key, and puts a `LogicalFilter` with
`min(x) <> max(x)` on top. That filter plays the role of a `HAVING` clause.
Finally, the projection keeps its own table index and column types, so
nothing above it needs rebinding (`self_join_aggregate.cpp:549-564`):

```cpp
		// the projection keeps its table index and its columns' types: a read key from the groups, the rest a NULL
		for (idx_t k = 0; k < projection.expressions.size(); k++) {
			auto &expr = projection.expressions[k];
			unique_ptr<Expression> replacement;
			if (key_of[k] != DConstants::INVALID_INDEX) {
				replacement = make_uniq<BoundColumnRefExpression>(
				    expr->GetReturnType(), ColumnBinding(group_index, ProjectionIndex(key_of[k])));
			} else {
				replacement = make_uniq<BoundConstantExpression>(Value(expr->GetReturnType()));
			}
			replacement->SetAlias(expr->GetAlias());
			expr = std::move(replacement);
		}
		// drops the filters, the join and the other scan
		projection.children[0] = std::move(having);
		projection.ResolveOperatorTypes();
```

A read key column now points at the `GROUP BY` output. A column that nothing
reads (such as `wh1` and `wh2` in q95's CTE) becomes a typed `NULL`
constant. The column stays, so a CTE still has the columns its references
expect.

### 7.10.11.6 Worked example: before and after

Take one query from the test file:

```sql
SELECT DISTINCT a.o, a.p FROM ws a, ws b WHERE a.o = b.o AND a.p = b.p AND a.w <> b.w
```

**Before** (the plan as bound; filter pushdown has not run yet):

```
LogicalDistinct (a.o, a.p)                         <- parent ignores duplicates
  └─ LogicalProjection (a.o, a.p)                  <- candidate
       └─ LogicalFilter (a.o = b.o AND a.p = b.p AND a.w <> b.w)
            └─ LogicalCrossProduct
                 ├─ LogicalGet (ws, alias a)
                 └─ LogicalGet (ws, alias b)
```

The keys are `o` and `p`. The single inequality column is `w`. Both
projection columns are keys.

**After** `SELF_JOIN_AGGREGATE`:

```
LogicalDistinct (a.o, a.p)
  └─ LogicalProjection (group.o, group.p)          [same table index as before]
       └─ LogicalFilter (min(w) <> max(w))         ["HAVING"]
            └─ LogicalAggregate (GROUP BY o, p; min(w), max(w))
                 └─ LogicalFilter (o IS NOT NULL, p IS NOT NULL)
                      └─ LogicalGet (ws, alias a)  [one scan; b is gone]
```

The test checks that the physical plan has a `Group By` with `min(` and
`max(`, and that it does **not** have two `Seq Scan`s:

```
query II
EXPLAIN SELECT DISTINCT a.o, a.p FROM ws a, ws b WHERE a.o = b.o AND a.p = b.p AND a.w <> b.w
----
physical_plan	<!REGEX>:[\s\S]*Seq Scan[\s\S]*Seq Scan[\s\S]*
```

With `SET disabled_optimizers = 'self_join_aggregate'`, the same `EXPLAIN`
shows two `Seq Scan`s. The test also runs every rewritten query twice, once
with the pass off and once on, and expects the same answer. For example,
q95's shape over the test's `ws` table returns `545  140  70583` both ways.

### 7.10.11.7 Why it runs before filter pushdown

The pass runs right after the first `CTE_INLINING` and before
`FILTER_PULLUP` and `FILTER_PUSHDOWN`. The comment at its call site says why
(`optimizer.cpp:265-271`):

```cpp
	// Rewrites a self-join on equal keys and one inequality, read only through its keys by a parent that ignores
	// duplicates, into GROUP BY keys HAVING min(x) <> max(x) over one scan. Runs before filter pushdown, which
	// would move the join's filters into the scans and hide the pattern
	RunOptimizer(OptimizerType::SELF_JOIN_AGGREGATE, [&]() {
		SelfJoinAggregate self_join_aggregate(*this);
		plan = self_join_aggregate.Optimize(std::move(plan));
	});
```

Before pushdown, the conditions `a.o = b.o` and `a.w <> b.w` sit in one
`LogicalFilter` above a cross product. That is easy to match. After
pushdown, they would be join conditions, and other filters would be moved
into the scans as table filters. `PlainScan` refuses scans with table
filters, so the pattern would no longer match. Running early also helps the
later passes: filter pushdown then pushes the new `IS NOT NULL` filter into
the single scan, and `JOIN_ORDER` places the small `GROUP BY` result like
any other input.

This is the opposite choice from `WINDOW_SELF_JOIN` (§7.10.10), which runs
*after* filter pushdown on purpose. `WINDOW_SELF_JOIN` *makes* a copy of a
subtree, so it wants filters already pushed into it. `SELF_JOIN_AGGREGATE`
*removes* a copy, so it wants to see the plan before pushdown changes its
shape.

## 7.10.12 Summary

`WINDOW_SELF_JOIN` recognizes a `PARTITION BY`-bearing aggregate window
function whose frame is provably equivalent to "the whole partition" —
either no `ORDER BY` at all (with the frame's `CURRENT ROW` boundary, under
`RANGE`/`GROUPS` framing specifically, folding into "the whole partition"
precisely because there's no order to distinguish rows by), or an `ORDER BY`
whose effect is neutralized by an explicit `UNBOUNDED PRECEDING ...
UNBOUNDED FOLLOWING` frame (any of `ROWS`/`RANGE`/`GROUPS`, since "unbounded"
is a single, frame-type-agnostic boundary value) — and rewrites it into an
equivalent `GROUP BY` aggregate, deep-copied off to one side, joined back
onto every original row via an `INNER` join keyed on the partition columns
with null-safe (`IS NOT DISTINCT FROM`) equality, so that `NULL` partition
keys — which `GROUP BY` collapses into one group — still find their group's
aggregate row rather than silently vanishing from an `INNER` join's output.
The function eligible for this treatment is not a curated allowlist but the
entire aggregate-function catalog, gated purely by `ExpressionType::WINDOW_AGGREGATE`
against the categorically separate ranking and navigational window-function
families; the input the window sits on is restricted to a linear
scan/filter/projection/(already-grouped-)aggregate chain, with a volatile-expression
guard and a defensive `try`/`catch` around the underlying `LogicalOperator::Copy`
serialization round-trip protecting correctness when either condition can't
be guaranteed. What Chapter 8 inherits from this pass, for any query where
it fires, is one more ordinary join for DPhyp to place — which is exactly
why this pass runs before `JOIN_ORDER` rather than after it.

The same part also covers `SELF_JOIN_AGGREGATE` (pass #3, §7.10.11), which
goes the other way. It finds a self-join on equal keys plus one `<>`
condition, read only by a parent that ignores duplicates, and replaces it
with `GROUP BY keys HAVING min(x) <> max(x)` over a single scan. It runs
*before* filter pushdown, because pushdown would hide the pattern, while
`WINDOW_SELF_JOIN` runs *after* it, because it wants filters already pushed
into the subtree it copies.
