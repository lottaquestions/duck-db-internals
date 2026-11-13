# Chapter 7 — The Optimizer

**This is the largest chapter in the book, by a wide margin — several hundred
printed-book-equivalent pages across this overview and 27 companion deep-dive
files.** Every one of DuckDB's ~45 named optimizer passes and all 37 rules of
the `EXPRESSION_REWRITER` pass get a test-grounded treatment below (two
passes, `SELF_JOIN_AGGREGATE` and `REDUNDANT_ORDER_KEYS`, are covered in this
overview, in §7.4.1 and §7.4.2): real source walkthroughs with `file:line`
citations, and — the organizing principle of every part file — every claim illustrated by walking
through actual `.test` files from DuckDB's own test suite, reproducing the SQL
and explaining exactly what each test proves and why. Several part files
uncovered genuine discrepancies between this book's working assumptions (or
even DuckDB's own historical documentation/course material) and what the
current source actually does; every one of those is called out explicitly in
the relevant file rather than smoothed over, in keeping with the book's
stated sourcing policy (Chapter 0).

**This file (`00-overview.md`) is the front door**: it covers the optimizer's
architecture, the full ordered pipeline, the `disabled_optimizers`/
`duckdb_optimizers()` introspection tools, the expression-rewriter's rule list,
and Query A traced through the whole pipeline at a level of depth appropriate
for a first read. Every pass gets a one-paragraph-to-a-page treatment here and
then a much deeper, dedicated file below. Read this file first; then use the
table to jump to whichever passes you want to go deeper on.

## Table of contents for this chapter's 27 deep-dive parts

| Part | Covers | Passes/rules |
|------|--------|---------------|
| [01](01-expression-rewriter-arithmetic-and-constants.md) | Expression rewriter I | `ConstantFoldingRule`, `ConstantOrderNormalizationRule`, `ArithmeticSimplificationRule`, `MoveConstantsRule`/`MoveUnaryMinusRule`, `DistributivityRule`, `StructExtractStructPackFoldingRule`, `LeastGreatestSimplificationRule` |
| [02](02-expression-rewriter-comparisons-and-logic.md) | Expression rewriter II | `ComparisonSimplificationRule`, `RowComparisonSimplificationRule`, `NotComparisonSimplificationRule`, `ConjunctionSimplificationRule`/`NotConjunctionSimplificationRule`, `CaseSimplificationRule`, `EqualOrNullSimplification`, `PredicateFactoringRule`, `MonotonePreimageRule`, `JoinDependentFilterRule`, `TimeStampComparison` |
| [03](03-expression-rewriter-strings-and-regex.md) | Expression rewriter III | `LikeOptimizationRule`, `StringPrefixRule`/`InstrPrefixRule`, `RegexOptimizationRule`/`RegexpReplaceExtractRule`, `EmptyNeedleRemovalRule`/`NoopReplaceRemovalRule`, `ContainsToInClauseRule`, `ListComprehensionRewriteRule` |
| [04](04-expression-rewriter-enums-dates-and-distinct.md) | Expression rewriter IV | `InClauseSimplificationRule`, `InEnumSimplificationRule`/`EnumCompareSimplificationRule`/`EnumComparisonRule`, `DatePartSimplificationRule`/`DateTruncSimplificationRule`, `OrderedAggregateOptimizer`, `DistinctAggregateOptimizer`/`DistinctWindowedOptimizer` |
| [05](05-cte-inlining-and-common-subplan.md) | CTE inlining & subplan reuse | `CTE_INLINING`, `CTE_FILTER_PUSHER` (including its CTE join-filter step), `COMMON_SUBPLAN` |
| [06](06-aggregate-function-rewriter-and-grouping-sets.md) | Aggregate rewriting I | `AGGREGATE_FUNCTION_REWRITER` (sum/avg/list rewrites), `GROUPING_SETS` |
| [07](07-filter-pullup-and-pushdown.md) | The flagship pass (longest part file) | `FILTER_PULLUP`, `FILTER_PUSHDOWN` |
| [08](08-regex-range-and-in-clause.md) | Plan-level rewrites | `REGEX_RANGE`, `IN_CLAUSE` |
| [09](09-empty-result-and-projection-pullup.md) | Empty-result & projection movement | `EMPTY_RESULT_PULLUP`, `PROJECTION_PULLUP` |
| [10](10-window-self-join.md) | One rich, standalone pass | `WINDOW_SELF_JOIN` |
| [11](11-outer-join-simplification-and-join-elimination.md) | Join semantics simplification | `OUTER_JOIN_SIMPLIFICATION`, `JOIN_ELIMINATION` |
| [12](12-aggregate-reuse-partial-pushdown-and-common-aggregate.md) | Aggregate rewriting II | `AGGREGATE_REUSE`, `PARTIAL_AGGREGATE_PUSHDOWN`, `COMMON_AGGREGATE` |
| [13](13-deliminator-and-unnest-rewriter.md) | Decorrelation cleanup (deep dive beyond Ch.6) | `DELIMINATOR`, `UNNEST_REWRITER` |
| [14](14-unused-columns-and-column-lifetime.md) | Column pruning | `UNUSED_COLUMNS`, `COLUMN_LIFETIME` |
| [15](15-build-side-probe-side-test-driven.md) | Test-driven supplement to Ch.8 §8.9 | `BUILD_SIDE_PROBE_SIDE` |
| [16](16-common-subexpressions.md) | Expression-level dedup | `COMMON_SUBEXPRESSIONS` (CSE) |
| [17](17-limit-pushdown-row-group-pruner-and-sampling.md) | Limit/scan interaction | `LIMIT_PUSHDOWN`, `ROW_GROUP_PRUNER`, `SAMPLING_PUSHDOWN` |
| [18](18-topn-and-topn-window-elimination.md) | Top-K rewrites | `TOP_N`, `ROW_NUMBER_REWRITER`, `TOP_N_WINDOW_ELIMINATION` |
| [19](19-late-materialization.md) | One rich, standalone pass | `LATE_MATERIALIZATION` |
| [20](20-statistics-propagation.md) | The pass with the largest test corpus | `STATISTICS_PROPAGATION` |
| [21](21-projection-placement-and-duplicate-groups.md) | Late-stage cleanup | `PROJECTION_PLACEMENT`, `DUPLICATE_GROUPS` |
| [22](22-reorder-filter.md) | One rich, standalone pass | `REORDER_FILTER` (+ contrast with runtime `AdaptiveFilter`, Ch.12) |
| [23](23-remote-pushdown-partitioned-execution-and-extensions.md) | The optimizer's outside interfaces | `REMOTE_PUSHDOWN`, `PARTITIONED_EXECUTION`, `EXTENSION` hooks |
| [24](24-join-filter-pushdown.md) | A runtime-computed optimization | `JOIN_FILTER_PUSHDOWN` |
| [25](25-type-pushdown-and-scalar-fn-pushdown.md) | Late, reader-specific pushdown | `TYPE_PUSHDOWN`, `SCALAR_FN_PUSHDOWN` |
| [26](26-join-order-test-driven-deep-dive.md) | Test-driven supplement to Ch.8 | `JOIN_ORDER` (the 39-file `test/optimizer/joins/` corpus) |
| [27](27-compressed-materialization.md) | Plan-intermediate compaction | compressed materialization (invoked from within `STATISTICS_PROPAGATION`) |
| this file, §7.4.1 | Self-join to `GROUP BY` (pass #3) | `SELF_JOIN_AGGREGATE` |
| this file, §7.4.2 | Redundant `ORDER BY` keys (pass #21) | `REDUNDANT_ORDER_KEYS` |

Two passes have no part file of their own. `SELF_JOIN_AGGREGATE` (pass #3)
turns a certain kind of self-join into a `GROUP BY`. `REDUNDANT_ORDER_KEYS`
(pass #21) drops `ORDER BY` keys that come after a unique key. §7.4.1 and
§7.4.2 describe both in full.

## 7.1 The plan the binder hands you is often a bad plan

Chapter 5 established what the unoptimized logical plan actually looks like, and
it is worth restating the point bluntly before this chapter explains what fixes
it: **the unoptimized plan is syntax-driven, not cost-driven.** The binder does
not choose a good execution strategy — it mechanically transliterates the SQL
text's clause structure into operators, in the order the clauses were written.

Concretely, for a query with an implicit multi-table `FROM` list or a chain of
explicit `JOIN`s, the binder builds a **left-deep tree that mirrors the order
the tables appear in the `FROM` clause**, nothing more — the first two tables
become the innermost join/cross-product, the third table joins onto that
result, and so on, regardless of which join order would actually be cheap.
Every predicate in the `WHERE` clause becomes a `LogicalFilter` bolted on top of
whatever join tree the `FROM` clause produced, regardless of how selective that
predicate is or how early it could, in principle, be evaluated. Every expression
in the `SELECT` list becomes a `LogicalProjection` at the very top of the tree,
computed only once every join and every filter has already run — including
columns nothing downstream needs, which are still scanned, joined, and carried
through every intermediate operator because the binder has no reason to leave
them out.

This is not a defect in the binder — the binder's job is name resolution and
type checking, not query optimization, and conflating the two would make it
much harder to reason about correctness. But it does mean that if DuckDB
executed the binder's output directly, it would, in the general case, be
needlessly slow: filters would run after joins that could have been avoided
entirely if the filter had pruned rows first; a three-way join with no
selectivity information would be evaluated in whatever order the `FROM` clause
happened to list the tables, which for real schemas is very often the worst
of the `n!` possible orders; and every column ever mentioned anywhere in the
query would ride along through every operator whether or not it survived to
the final result.

This is the entire reason the box in the front matter's seven-stage diagram
labeled "Optimizer" exists, and it's why this is one of the longest chapters
in the book: turning a syntactically faithful but strategically arbitrary
tree into a strategically reasonable one takes roughly forty separate,
specialized rewrites, each responsible for one narrow aspect of "make this
plan not be dumb." None of them, individually, is doing anything as dramatic
as choosing a join order (that's Chapter 8's DPhyp search, and it is the one
place in this list where DuckDB does something recognizable as a cost-based
*search* over alternatives) — most of them are narrow, syntactic-to-semantic
rewrites: push this filter down, fuse this `ORDER BY`+`LIMIT` pair, drop this
column nobody reads, notice this subplan is computed twice. What makes the
optimizer interesting as a piece of engineering is less any single pass and
more the *architecture* that holds all forty of them together, which is the
subject of §7.2.

## 7.2 The architectural choice: fixed passes, run once, in a fixed order

If you've read a paper on Cascades- or Volcano-style query optimization, or
looked at how Postgres's or Calcite's rule-based optimizers work, you might
expect DuckDB's optimizer to be a rule engine: a set of pattern-matching
rewrite rules thrown into a worklist, fired opportunistically wherever they
match, re-triggered whenever an earlier rule changes something nearby, run
until no rule fires anymore (a fixpoint), possibly exploring multiple
alternative plan shapes and picking the cheapest by an explicit cost model.

DuckDB's optimizer is not that. It is architecturally much simpler, and the
simplicity is a deliberate, named tradeoff (this framing comes from Chapter 8
of the DiDi course [10]): **a fixed,
hardcoded sequence of named "specialist" passes, each one a self-contained
class responsible for one aspect of the plan, executed exactly once, in a
fixed order that never changes based on what any earlier pass did.** There is
no worklist, no re-triggering, no fixpoint, and — with the sole, explicitly
carved-out exception of the join-order search in Chapter 8 — no cost-based
enumeration of alternative plan shapes to choose among. Every pass gets one
shot at the plan tree it inherits from the pass before it, does whatever
local rewriting it knows how to do, and hands the tree to the next pass on
the list. If pass #30 makes a change that would have unlocked a rewrite pass
#5 already ran past, pass #5 does not get a second chance — the optimization
opportunity is simply missed, in this run of the query.

This has a concrete, measured tradeoff, and it's worth stating in DiDi's own
terms [10] because it explains a lot about why DuckDB feels fast even on plans it
hasn't optimized as thoroughly as a search-based optimizer theoretically
could: each pass typically costs on the order of **10–50 microseconds**, and
because the sequence is fixed and never re-entered, total optimizer time for
an ordinary query is small and *predictable* — you are not at the mercy of a
search procedure that might, for a sufficiently adversarial plan shape, blow
up combinatorially trying to reach a fixpoint. The price is that DuckDB's
optimizer can, and sometimes does, leave real optimization potential on the
table: a rewrite that would only become applicable after a later pass has
already run is a rewrite DuckDB's optimizer structurally cannot perform in a
single query invocation, because there is no mechanism for a later pass to
ask an earlier one to run again. DPhyp (Chapter 8) is the deliberate,
narrowly-scoped exception to this rule — join ordering really is a search
over alternative plan shapes, costed against cardinality estimates — precisely
because reordering joins well cannot be done as a single one-shot syntactic
rewrite the way, say, constant folding can.

### The driver: `Optimizer` and `RunOptimizer`

The class that owns this sequence is `Optimizer`
(`src/optimizer/optimizer.cpp`). Its constructor does two things: it wires up
`context` and `binder` references, and it populates a member named `rewriter`
— an `ExpressionRewriter` — with the full, fixed list of 37 expression-level
rewrite rules that will run as this chapter's very first pass (§7.5 covers
these in full):

```cpp
// src/optimizer/optimizer.cpp:73
Optimizer::Optimizer(Binder &binder, ClientContext &context) : context(context), binder(binder), rewriter(context) {
	rewriter.rules.push_back(make_uniq<ConstantOrderNormalizationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<ConstantFoldingRule>(rewriter));
	// ... 35 more rules, in a fixed order — see §7.5
```

Every one of the ~45 plan-level passes described in this chapter is invoked
through a single helper, `Optimizer::RunOptimizer`, which is where the "fixed
sequence, individually toggleable, individually profiled" architecture
actually lives in code:

```cpp
// src/optimizer/optimizer.cpp:137
void Optimizer::RunOptimizer(OptimizerType type, const std::function<void()> &callback) {
	context.InterruptCheck();

	if (OptimizerDisabled(type)) {
		// optimizer is marked as disabled: skip
		return;
	}
	auto &profiler = QueryProfiler::Get(context);
	{
		auto optimizer_timer = profiler.StartTimerInternal("optimizer." + StringUtil::Lower(EnumUtil::ToString(type)));
		callback();
	}
	if (plan) {
		Verify(*plan);
	}
}
```

Four things worth noticing in this fifteen-line function, because each one is
architecturally load-bearing:

1. **Every pass is individually named** (`OptimizerType type`) and individually
   **profiled** — `profiler.StartTimerInternal("optimizer." + ...)` means
   `EXPLAIN ANALYZE`'s timing breakdown can, in principle, show you exactly how
   many microseconds `optimizer.filter_pushdown` or `optimizer.join_order` cost
   for a specific query, which is how the "10–50µs per pass" figure in §7.1 is
   actually measured rather than merely asserted.
2. **Every pass is individually disableable** at runtime, checked via
   `OptimizerDisabled(type)` before the callback runs at all — this is the
   mechanism behind `disabled_optimizers`, covered in full in §7.3.
3. **The callback is a closure over the current `plan`**, not a return value
   threaded through some outer combinator — this is what "fixed sequence,
   in place" looks like concretely: each `RunOptimizer(...)` call site
   elsewhere in `optimizer.cpp` closes over `plan` and reassigns it
   (`plan = some_pass.Optimize(std::move(plan))`), so the tree is genuinely
   rewritten in place, one pass at a time, top to bottom of the function.
4. **`Verify(*plan)` runs after every single pass**, not just once at the end
   — in debug/verification builds this walks the tree and re-checks
   invariants (column bindings still resolve, types still line up) after
   *each* of the ~45 rewrites, which is a strong internal consistency check
   given how much churn forty independent passes put a tree through.

`OptimizerDisabled` itself is worth showing too, because it's the other half
of the toggle mechanism and it reveals something specific: disabling *all*
optimization (`PRAGMA disable_optimizer`, i.e. the `EnableOptimizerSetting`
being off) and disabling one *named* pass (`disabled_optimizers`) are checked
through the exact same function, just gated by different settings:

```cpp
// src/optimizer/optimizer.cpp:124
bool Optimizer::OptimizerDisabled(OptimizerType type) {
	return OptimizerDisabled(context, type);
}

bool Optimizer::OptimizerDisabled(ClientContext &context_p, OptimizerType type) {
	if (!Settings::Get<EnableOptimizerSetting>(context_p)) {
		// all optimizes are disabled
		return true;
	}
	auto &config = DBConfig::GetConfig(context_p);
	return config.IsOptimizerDisabled(type);
}
```

The lookup in the set of disabled passes is done by a helper,
`DBConfig::IsOptimizerDisabled` (`src/main/config.cpp:1016`). The helper takes
a lock (`disabled_optimizers_lock`) before it reads the set. So one thread can
change `disabled_optimizers` while another thread is optimizing a query,
without a data race.

One more structural fact worth flagging before the pipeline listing, because
it's easy to miss and matters if you ever go looking for "where does the
optimizer start": **`PRAGMA disable_optimizer` does not disable everything.**
`Optimizer::Optimize` (`src/optimizer/optimizer.cpp:577`) begins by calling
`LowerMandatoryAggregateRewrites` (line 578) *before* it ever checks
`EnableOptimizerSetting`:

```cpp
// src/optimizer/optimizer.cpp:577
unique_ptr<LogicalOperator> Optimizer::Optimize(unique_ptr<LogicalOperator> plan_p) {
	plan_p = LowerMandatoryAggregateRewrites(std::move(plan_p));
	if (!Settings::Get<EnableOptimizerSetting>(context)) {
		CTEFilterPusher::ClearDependencies(*plan_p);
		return plan_p;
	}
	Verify(*plan_p);

	this->plan = std::move(plan_p);
	// ...
	RunBuiltInOptimizers();
	// ...
	Planner::VerifyPlan(context, plan);
```

`LowerMandatoryAggregateRewrites` (line 567) is not one of the ~45 *optional*
heuristic passes this chapter is otherwise about — it lowers `ROLLUP`/`CUBE`/
`GROUPING SETS` into explicit grouping-set plans and applies any aggregate
rewrite the physical-plan generator has no operator for otherwise
(`AggregateRewritePolicy::MANDATORY`, as opposed to the optional,
toggle-gated `UNCONDITIONAL`/`COST_BASED` policies used inside
`RunBuiltInOptimizers`). It runs unconditionally because without it, the
physical plan generator would have nothing valid to turn some plan shapes
into — it's plan *legalization*, not optimization, and the distinction
matters if you're using `PRAGMA disable_optimizer` to inspect an
"unoptimized" plan and are surprised that a `ROLLUP` has already been
expanded.

The main sequence itself — the forty-ish heuristic passes this chapter
covers — lives inside `RunBuiltInOptimizers()` (line 230), called once from
`Optimize()` at line 604, bracketed by two loops over third-party
`OptimizerExtension`s (lines 594 and 606 — pre-optimize hooks run before
`RunBuiltInOptimizers()`, post-optimize hooks run after; both are wrapped in
their own `RunOptimizer(OptimizerType::EXTENSION, ...)` calls, so they get
the same timing/toggle treatment as everything else). §7.4 reproduces that
sequence's exact order.

## 7.3 Introspecting the pipeline: `disabled_optimizers` and `duckdb_optimizers()`

Before walking the ~45 passes one by one, it's worth knowing that DuckDB gives
you two concrete, runnable tools for inspecting this machinery directly,
rather than taking this chapter's word for the pipeline's existence:

**`duckdb_optimizers()`** is a table function that lists every registered
`OptimizerType` by name — this is literally where the string `"join_order"` or
`"filter_pushdown"` used by `disabled_optimizers` comes from:

```sql
-- test/sql/table_function/duckdb_optimizers.test
SELECT * FROM duckdb_optimizers();

SELECT name FROM duckdb_optimizers() WHERE name='join_order';
-- join_order
```

**`SET disabled_optimizers = '...'`** takes a comma-separated list of those
names and turns each corresponding pass's `RunOptimizer(...)` call into a
no-op (recall `OptimizerDisabled` from §7.2 — this is exactly the
`config.options.disabled_optimizers` set it checks). The test suite exercises
both the happy path and the error path when a name doesn't match anything
`duckdb_optimizers()` would list:

```sql
-- test/sql/settings/setting_disabled_optimizer.test
SET disabled_optimizers = '';                                          -- ok, clears the set
SET disabled_optimizers TO 'expression_rewriter';                      -- ok, one pass off
SET disabled_optimizers TO 'expression_rewriter,filter_pushdown,join_order';  -- ok, three off
SELECT current_setting('disabled_optimizers');                         -- ok, echoes back

SET disabled_optimizers TO 'expression_rewriteX';
-- Parser Error: ...Optimizer...not recognized...
SET disabled_optimizers TO 'unknown_optimizer';
-- Parser Error: ...Optimizer...not recognized...
```

And **`PRAGMA disable_optimizer`** (equivalently, `SET disable_optimizer=true`
— the `EnableOptimizerSetting` flag checked in `OptimizerDisabled` at
`optimizer.cpp:129`) turns off the *entire* `RunBuiltInOptimizers()` sequence
in one shot, which is the fastest way to get `EXPLAIN` to show you something
close to Chapter 5's raw, syntax-driven tree (modulo the mandatory
lowering rewrites from §7.2, which still run). Combined, these three tools —
list the passes, disable one by name, disable all of them — are the concrete
mechanism behind every "which pass did that" claim in the rest of this
chapter: if you're unsure whether a plan change you're looking at in
`EXPLAIN` came from `filter_pushdown` or from `statistics_propagation`,
disabling one at a time and re-running `EXPLAIN` will tell you directly.

## 7.4 The full pipeline, in exact execution order

This is the list this chapter exists to make faithful: every pass
`RunBuiltInOptimizers()` invokes, in the exact order it invokes them, with the
source file responsible and — for each one — at least one sentence of real
mechanical explanation rather than just a name. Line numbers are the
`RunOptimizer(OptimizerType::..., ...)` call sites in
`src/optimizer/optimizer.cpp` at commit `cb5673685153`. The list has 45
numbered entries, one for each `RunOptimizer` call in `RunBuiltInOptimizers`.
Together with the `REMOTE_PUSHDOWN` call and the two extension-hook calls
described after the list, `optimizer.cpp` makes 48 `RunOptimizer` calls in
all. A handful of passes appear
twice in the sequence (the optimizer re-runs a cheap bookkeeping pass after a
later, more disruptive pass has had a chance to invalidate its work) — both
occurrences are listed, in order, because the *second* occurrence is not
redundant: it's cleaning up after specific, named passes that ran in between.

Passes marked **(Ch. 8)** or **(Ch. 6)** are covered in full in those
chapters; this chapter gives them only the one-line pointer requested, since
duplicating a full chapter's content here would be redundant.

1. **`EXPRESSION_REWRITER`** — `optimizer.cpp:253`, `src/optimizer/expression_rewriter.cpp`.
   Walks the whole tree applying the fixed 37-rule expression-simplification
   list from §7.5 to every expression, plus a separate
   `ConstantOrNullSimplification` pass run in the same step. Purely
   expression-level — it never changes the *shape* of the operator tree,
   only what's inside each operator's expression lists.
2. **`CTE_INLINING`** — `optimizer.cpp:260` (and again at 333).
   `src/optimizer/cte_inlining.cpp`. Decides, per CTE, whether to inline its
   definition at each reference site instead of materializing it once — the
   decision walks a small tree of conditions (reference count via
   `CountCTEReferences`, explicit `[NOT] MATERIALIZED` hints, whether the CTE
   ends in an aggregate/`DISTINCT`/window operator via
   `EndsInAggregateOrDistinct`, whether a `LIMIT` is present via
   `ContainsLimit`) rather than one simple heuristic. It's run twice because
   the first run happens before join reordering and grouping-set lowering
   have had a chance to change what "references this CTE" even means for some
   plan shapes; the second catches CTEs whose reference count or shape only
   became inlinable after those later rewrites.
3. **`SELF_JOIN_AGGREGATE`** — `optimizer.cpp:268`,
   `src/optimizer/self_join_aggregate.cpp`. No numbered part covers it, so
   it gets a full treatment here, after the list (§7.4.1).
   In short: some queries join a table to itself only to ask "which keys
   have two different values of some column?" This pass answers that
   question with one scan and a `GROUP BY ... HAVING min(x) <> max(x)`
   instead of a join.
4. **`AGGREGATE_FUNCTION_REWRITER`** — `optimizer.cpp:274`,
   `src/optimizer/aggregate_function_rewriter.cpp`. Per its own inline
   comment in `optimizer.cpp`: *"Rewrites `AVG(x)` -> `SUM(x)/COUNT(x)` and
   `SUM(x+C)` -> `SUM(x) + C*COUNT(x)`"* — this is where DiDi's "sum_rewriter"
   [10] actually lives; see the dedicated discussion in §7.6.5, because
   it's not a separately-named `OptimizerType` the way DiDi's slide vocabulary
   might suggest.
5. **`FILTER_PULLUP`** — `optimizer.cpp:280`, `src/optimizer/filter_pullup.cpp`.
   Pulls `LogicalFilter`s *up* through certain operators — most importantly,
   through an equi-join's other side, so that a filter known to hold on one
   join input can be re-derived as an equivalent filter on the other input
   (mechanics and worked example in §7.6.3, alongside its pushdown
   counterpart).
6. **`FILTER_PUSHDOWN`** — `optimizer.cpp:286`, `src/optimizer/filter_pushdown.cpp`
   (using `filter_combiner.cpp` and the seventeen per-operator handler files
   under `src/optimizer/pushdown/` — for aggregates, cross products,
   `DISTINCT`, filters, scans (`pushdown_get.cpp`), each join type (inner,
   left, outer, mark, semi/anti, single), `LIMIT`, projections, secure views,
   set operations, `UNNEST`, and window operators). This is the pass that
   pushes predicates as close to the base-table scan as the plan shape
   allows; full mechanics in §7.6.3.
7. **`CTE_FILTER_PUSHER`** — `optimizer.cpp:299`, `src/optimizer/cte_filter_pusher.cpp`.
   A CTE-specific variant of filter pushdown: derives filters that hold at
   every reference site of a *materialized* CTE and pushes them into the
   CTE's own definition, so a materialized CTE scanned twice with two
   different downstream filters can still have the weaker of the two (or
   their disjunction) applied once, at materialization time.
   At the end of its work, this pass also runs a second helper,
   `CTEJoinFilterPusher` (`src/optimizer/cte_join_filter_pusher.cpp` and
   `cte_join_filter_collector.cpp`). It is called from
   `CTEFilterPusher::Optimize` (`src/optimizer/cte_filter_pusher.cpp:51`).
   It looks for references of a materialized CTE that sit under an inner
   join. It collects the join keys from the other side of those joins.
   When that is cheap enough (fixed ratios in
   `CTEJoinFilterPusher::Optimize`, `cte_join_filter_pusher.cpp:67-70`), it
   restricts the CTE's rows to keys that can still find a match. It runs
   inside the `CTE_FILTER_PUSHER` timer and has no toggle of its own. Part 05
   owns the details.

   Between #6 and #7, `RunBuiltInOptimizers` also runs a
   `FilterStatisticsOptimizer` (`optimizer.cpp:293-296`), skipped when a CTE
   contains DML. The call itself is not wrapped in `RunOptimizer`, so it has
   no name in `duckdb_optimizers()` and no toggle of its own. Its inner steps do
   use `RunOptimizer`, though: `FilterStatisticsOptimizer::Optimize`
   (`src/optimizer/filter_statistics.cpp:53-74`) runs a statistics step under
   `OptimizerType::STATISTICS_PROPAGATION` and, if that changed a filter, a
   second filter pushdown under `OptimizerType::FILTER_PUSHDOWN`. So its time
   is counted under those two names, and disabling either pass also skips
   that step. It also does nothing when the plan contains a
   `LOGICAL_DELIM_JOIN`, which with default settings never happens (Chapter 6
   §6.4.1).
8. **`REGEX_RANGE`** — `optimizer.cpp:305`, `src/optimizer/regex_range_filter.cpp`.
   Recognizes regex patterns that are secretly range/prefix constraints
   (an anchored literal-prefix regex) and rewrites them into a table filter
   the storage layer's zonemaps can use, the same way `LIKE 'foo%'` gets
   turned into a usable range by the expression-rewriter's
   `LikeOptimizationRule` (§7.5).
9. **`IN_CLAUSE`** — `optimizer.cpp:310`, `src/optimizer/in_clause_rewriter.cpp`.
   Rewrites a long, all-foldable `IN (...)` list into a `MARK`-join against a
   `LogicalColumnDataGet` built from the literal values — turning an O(n·m)
   linear scan-and-compare per row into an O(n+m) hash-join-style probe. Full
   mechanics and the exact rewrite threshold in §7.6.6. (Not to be confused
   with `InClauseSimplificationRule`, one of the *expression*-rewriter rules
   from pass #1 that does something narrower and unrelated — see the aside
   in §7.5.)
10. **`DELIMINATOR`** — `optimizer.cpp:316`, `src/optimizer/deliminator.cpp`.
   Post-decorrelation cleanup: removes redundant `DelimJoin`/`DelimGet` pairs
   left behind once `FlattenDependentJoins` (Chapter 6) has already turned a
   correlated subquery into an ordinary join tree. **(Ch. 6)**
   With default settings this pass finds nothing to do. The planner has
   already turned every delim join into joins over materialized CTEs
   (`delim_join_as_cte`, default `true`; Chapter 6 §6.4.1), and the same
   clean-up runs there instead, inside `DelimJoinCTERewriter`. This pass
   only works on delim joins when `delim_join_as_cte = false`. Turning this
   pass off with `disabled_optimizers` still matters, because the CTE step
   checks the same switch (part 13, §13.3.6).
11. **`GROUPING_SETS`** — `optimizer.cpp:322`,
    `src/optimizer/grouping_sets_optimizer.cpp`. Rewrites an aggregate over
    multiple grouping sets (`ROLLUP`/`CUBE`/explicit `GROUPING SETS`) into an
    explicit plan shape the physical aggregate operators can execute — the
    *optional*, heuristic counterpart to the *mandatory* lowering
    `LowerMandatoryAggregateRewrites` already performed in §7.2 before this
    pipeline even started (this instance also runs `MultiStageAggregateRewriter`
    with an `UNCONDITIONAL` policy immediately afterward, gated by the
    separate `DISTINCT_AGGREGATE_REWRITE` toggle — a policy check inline
    rather than its own timed `RunOptimizer` call).
12. **`CTE_INLINING`** again — `optimizer.cpp:333`. See #2.
13. **`EMPTY_RESULT_PULLUP`** — `optimizer.cpp:339`,
    `src/optimizer/empty_result_pullup.cpp`. Pulls a `LogicalEmptyResult`
    marker (produced when an earlier pass, most often statistics propagation,
    has already proven some subtree can never produce rows) *up* through
    parent operators that would themselves necessarily still produce zero
    rows given an empty child — so an entire join, not just its one empty
    input, can collapse to "this returns nothing."
14. **`WINDOW_SELF_JOIN`** — `optimizer.cpp:345`,
    `src/optimizer/window_self_join.cpp`. Per its own inline comment:
    *"Replaces some window computations with self-joins"* — a
    `PARTITION BY`-only aggregate window function (no `ORDER BY` frame) is
    rewritten into an equivalent `GROUP BY` aggregate joined back onto the
    original rows, which lets the hash-aggregate machinery do the work
    instead of the (more general, more expensive) window-function evaluator.
    Guarded by a `VolatileExpressionCounter` check — a window expression with
    volatile subexpressions is left alone, since duplicating it across a
    self-join would change how many times a volatile function evaluates.
15. **`PROJECTION_PULLUP`** — `optimizer.cpp:351`,
    `src/optimizer/projection_pullup.cpp`. Per its inline comment: *"Pull up
    projection from joins"* — hoists a `LogicalProjection` sitting directly
    under a join up above it where doing so doesn't change semantics,
    simplifying the tree the join-order optimizer has to reason about next.
16. **`OUTER_JOIN_SIMPLIFICATION`** — `optimizer.cpp:357`,
    `src/optimizer/outer_join_simplification.cpp`. Per its inline comment:
    *"Simplifies FULL OUTER -> LEFT/RIGHT OUTER -> INNER if NULLs are
    filtered anyway"* — if a filter above a `FULL OUTER JOIN` would discard
    any row where the left (or right) side's columns are `NULL`, then the
    outer-ness on that side was never actually observable, and the join can
    be strictly simplified to a cheaper join type.
17. **`JOIN_ORDER`** — `optimizer.cpp:364`, `src/optimizer/join_order/`. The
    DPhyp-based dynamic-programming join reorderer — the one pass in this
    entire pipeline that is a genuine cost-based search over alternative
    plan shapes rather than a one-shot syntactic rewrite. **(Ch. 8, in full)**
    For a query with only one join condition (Query A, §7.7), there is
    nothing to reorder and this pass is close to a no-op; its real work shows
    up starting at three relations.
18. **`AGGREGATE_REUSE`** — `optimizer.cpp:370`,
    `src/optimizer/aggregate_reuse.cpp` (+ `materialized_aggregate_reuse.cpp`).
    Per its inline comment: *"Reuse exact aggregate payloads exposed by
    filtering SEMI joins"* — when a `SEMI`/`ANTI` join has already computed
    an aggregate as a byproduct of its filtering, and something downstream
    needs that same aggregate value, this pass lets the downstream consumer
    reuse it instead of recomputing it.
19. **`PARTIAL_AGGREGATE_PUSHDOWN`** — `optimizer.cpp:379`,
    `src/optimizer/partial_aggregate_pushdown.cpp`. Per its inline comment:
    *"Pre-aggregate SUM/COUNT below joins when GROUP BY is on dimension
    columns and the fact side dominates cardinality — reduces probe-side
    work"* — a star-schema-shaped optimization: aggregate the large fact
    table down to (dimension key, partial sum/count) pairs *before* joining
    to the small dimension table, instead of joining first and aggregating
    the full, unreduced row count afterward.
20. **`JOIN_ELIMINATION`** — `optimizer.cpp:384`,
    `src/optimizer/join_elimination.cpp`. Removes a join entirely when
    nothing downstream references any column that could only come from the
    eliminated side and a foreign-key/uniqueness guarantee proves the join
    can't change the row count or duplicate rows (e.g. a join purely to
    validate existence, with no columns pulled from the joined-to side and
    no cardinality-changing effect provable from constraints).
21. **`REDUNDANT_ORDER_KEYS`** — `optimizer.cpp:389`,
    `src/optimizer/remove_redundant_order_keys.cpp`. No numbered part covers
    it, so §7.4.2 below describes it in full. In short: in
    `ORDER BY id, payload`, if `id` is unique, no two rows can tie on `id`,
    so `payload` never decides anything. The pass drops it.
22. **`UNNEST_REWRITER`** — `optimizer.cpp:395`, `src/optimizer/unnest_rewriter.cpp`.
    Per its inline comment: *"rewrites UNNESTs in DelimJoins by moving them to
    the projection"* — further post-decorrelation cleanup of the join tree
    `FlattenDependentJoins` produced. **(Ch. 6)**
23. **`UNUSED_COLUMNS`** — `optimizer.cpp:401` (and again at 517),
    `src/optimizer/remove_unused_columns.cpp` (+ `column_binding_replacer.cpp`).
    Walks the tree top-down tracking which columns each operator's *parent*
    actually needs, and prunes any column an operator produces that nothing
    above it reads — including columns from a base-table scan that never
    survive past an early filter, so those columns don't need to be
    materialized off disk at all. Re-run a second time (see #37) after
    statistics propagation has had a chance to remove expressions and
    potentially orphan more columns.
24. **`DUPLICATE_GROUPS`** — `optimizer.cpp:407`,
    `src/optimizer/remove_duplicate_groups.cpp`. Per its inline comment:
    *"Remove duplicate groups from aggregates"* — if the same expression
    appears twice in a `GROUP BY` list (directly, or as a provable functional
    duplicate), the redundant copy is dropped so the hash-aggregate's key
    isn't wider than it needs to be.
25. **`COMMON_SUBEXPRESSIONS`** — `optimizer.cpp:413`,
    `src/optimizer/cse_optimizer.cpp`. Classic CSE, but scoped to *within* a
    single operator's expression list: if `a + 1` is computed twice inside
    one projection or filter's expressions, it's computed once and the
    second occurrence becomes a reference to the first's result. Full
    worked example in §7.6.7.
26. **`COLUMN_LIFETIME`** — `optimizer.cpp:419` (and again at 525),
    `src/optimizer/column_lifetime_analyzer.cpp`. Per its inline comment:
    *"creates projection maps so unused columns are projected out early"* —
    a finer-grained companion to `UNUSED_COLUMNS`: rather than only removing
    columns nothing ever reads, this computes, per operator, the exact point
    in the tree past which a given column is no longer needed, and installs
    a projection map that drops it there rather than carrying it further
    than necessary even when it's still read *somewhere* downstream.
27. **`BUILD_SIDE_PROBE_SIDE`** — `optimizer.cpp:426`,
    `src/optimizer/build_probe_side_optimizer.cpp`. Per its inline comment:
    *"Once we know the column lifetime, we have more information regarding
    what relations should be the build side/probe side"* — decides, for each
    hash join, which input becomes the (materialized, hashed) build side and
    which becomes the (streamed) probe side, now informed by the narrower,
    lifetime-pruned column sets #26 just computed. **(fuller mechanics in
    Ch. 8/Ch. 9, alongside the rest of physical hash-join operator choice)**
28. **`COMMON_SUBPLAN`** — `optimizer.cpp:435` (conditional: skipped if the
    plan contains a DML CTE, since table statistics can be stale across a DML
    boundary), `src/optimizer/common_subplan_optimizer.cpp`. Detects two
    *syntactically identical* plan subtrees (not just identical
    expressions — whole scan+filter+join fragments), materializes the
    subtree once as a synthetic CTE, and replaces both original occurrences
    with `CTE_SCAN`s onto it — skipped for a subplan containing anything
    volatile, since two occurrences of a volatile computation are not
    actually interchangeable. Full worked example in §7.6.7.
29. **`LIMIT_PUSHDOWN`** — `optimizer.cpp:442`, `src/optimizer/limit_pushdown.cpp`.
    Per its inline comment: *"pushes LIMIT below PROJECTION"* — a narrow,
    specific rewrite: a `LIMIT` sitting directly above a plain projection can
    move below it, since a projection can't change row count. It does
    nothing when the limit's immediate child isn't a projection (an
    `ORDER BY`, for instance, blocks it — see Query A in §7.7).
30. **`ROW_GROUP_PRUNER`** — `optimizer.cpp:447`, `src/optimizer/row_group_pruner.cpp`.
    Recognizes an `ORDER BY ... LIMIT N` whose sort key traces back to an
    actual stored column on a `LogicalGet` (walking down through the
    intervening filter/aggregate/projection operators via
    `RowGroupPruner::FindLogicalOrder`/`TryOptimize`), and, when it does,
    arranges to scan that table's row groups in min/max zonemap order rather
    than storage order — so the scan can stop after finding `N` qualifying
    rows in the first few row groups instead of reading the whole table.
    Bails out (returns `false` from `TryOptimize`) if another `LIMIT`
    operator sits between the order and the sort key's source, since that
    could change which rows the reordering is even allowed to affect.
31. **`SAMPLING_PUSHDOWN`** — `optimizer.cpp:453`, `src/optimizer/sampling_pushdown.cpp`.
    Pushes a `TABLESAMPLE` clause's sampling operation down toward the base
    scan, so sampling can be applied while reading rather than after
    materializing the full input.
32. **`TOP_N`** — `optimizer.cpp:459`, `src/optimizer/topn_optimizer.cpp`.
    Fuses an `ORDER BY` immediately followed by a constant `LIMIT` (with a
    constant or absent `OFFSET`) into a single `LogicalTopN` operator, which
    the physical layer implements with a bounded top-k structure instead of
    a full sort. Notably conditional on a cost heuristic, not automatic —
    see the exact code and Query A's application of it in §7.6.4 and §7.7.
33. **`LATE_MATERIALIZATION`** — `optimizer.cpp:465`,
    `src/optimizer/late_materialization.cpp` (+ `late_materialization_helper.cpp`).
    For a scan feeding a filter/sort/`LIMIT`/`GROUP BY` where some scanned
    columns are only needed in the *final* projected output and not by the
    filter/sort/group key itself, rewrites the scan to carry row IDs through
    the narrow operators and defer fetching the wide payload columns until
    a late semi-join reconstructs them — keeping the hot intermediate
    representation narrow. Gated by a row-count cap
    (`LateMaterializationMaxRowsSetting`), since deferring materialization
    only pays off below some row count where the extra semi-join's overhead
    doesn't dominate.
34. **`STATISTICS_PROPAGATION`** — `optimizer.cpp:476` (conditional: skipped
    if the plan contains a DML CTE, whose table statistics captured at plan
    time may already be stale), `src/optimizer/statistics_propagator.cpp`
    (+ `relation_statistics/`, `filter_statistics.cpp`,
    `compressed_materialization.cpp`). Bottom-up propagation of column
    statistics (min/max, distinct count, null fraction) through the whole
    tree — used both to prove some filters are unconditionally true or false
    (`StatisticsPropagator::ReplaceWithEmptyResult`, at
    `statistics_propagator.cpp:41`, wraps a subtree in a
    `LogicalEmptyResult` the moment a filter is proven never satisfiable)
    and, immediately after this pass, to drive a **cost-based** second run
    of the `MultiStageAggregateRewriter` (using the just-computed statistics
    map) plus the `PROJECTION_PLACEMENT` pass below. Full discussion,
    including a worked-example correction to Chapter 1's shallow preview
    about what this pass actually feeds, in §7.6.2.
35. **`PROJECTION_PLACEMENT`** — `optimizer.cpp:493` (only runs if
    statistics were actually propagated in #34), `src/optimizer/projection_placement.cpp`.
    Uses the freshly-computed statistics map to decide where, concretely, in
    the operator chain a narrowing projection should be materialized — the
    placement decision that the column-lifetime analysis (#26) and
    statistics propagation (#34) set up but don't themselves execute.
36. **`TOP_N_WINDOW_ELIMINATION`** — `optimizer.cpp:501`,
    `src/optimizer/topn_window_elimination.cpp`. Per its inline comment:
    *"rewrite row_number window function + filter on row_number to
    aggregate"* — a query shaped like `WHERE row_number() OVER (...) <= N`
    (a per-group top-N idiom) is recognized and rewritten to avoid
    materializing and numbering every row in every partition just to throw
    away everything past rank `N`.
37. **`COMMON_AGGREGATE`** — `optimizer.cpp:508`,
    `src/optimizer/common_aggregate_optimizer.cpp`. Per its inline comment:
    *"remove duplicate aggregates"* — the aggregate-level sibling of CSE
    (#25): if `SUM(x)` is requested twice inside one `LogicalAggregate`
    (directly, or as a sub-computation two different output expressions both
    depend on), it's computed once.
38. **`UNUSED_COLUMNS`** again — `optimizer.cpp:517`. It runs only if one of
    two things happened. Either statistics propagation (#34) removed some
    expressions (`removed_expressions`), or the top-N window rewrite (#36)
    used late materialization (`topn_used_late_materialization`). See #23 —
    the tree-clearing call `ClearProjectionMaps(*plan)` runs first, since the
    projection maps #26/#35 built may reference expressions #34 just deleted.
    The second trigger exists because when #36 uses late materialization, it
    reads the wide payload columns in a second scan. The first scan then holds
    columns that nothing reads any more. The comment at
    `optimizer.cpp:513-515` says so:

    ```cpp
    // statistics propagation removes filters, join conditions and aggregate children that it proves redundant.
    // this can leave columns in a scan that nothing references any more - prune them again. so does a top-N rewrite
    // with late materialization, which reads the payload in a second scan
    if (removed_expressions || topn_used_late_materialization) {
    ```
39. **`COLUMN_LIFETIME`** again — `optimizer.cpp:525`. See #26 — rebuilds
    projection maps once more, now current with everything #34–#38 changed.
40. **`REORDER_FILTER`** — `optimizer.cpp:531`, `src/optimizer/expression_heuristics.cpp`.
    Sorts the conjuncts of every multi-conjunct filter (and every
    `AND`/`OR` `BoundConjunctionExpression`) from cheap-to-evaluate to
    costly-to-evaluate, using a static per-operator/per-type cost heuristic
    (`ExpressionHeuristics::Cost`) — but explicitly refuses to reorder if any
    conjunct `CanThrow()` or carries an `ExpressionBarrier`, since moving a
    conjunct that might raise a cast error ahead of one that filters out the
    rows that would have caused it to throw would change observable
    behavior. Full mechanics and the exact guard code in §7.6.1.
41. **`PARTITIONED_EXECUTION`** — `optimizer.cpp:537`,
    `src/optimizer/partitioned_execution.cpp`. Per its inline comment:
    *"split pipelines into partitions and union them back together"* — plan
    preparation for radix-partitioned parallel execution (this is where, for
    example, the logical groundwork for a `PhysicalPartitionedAggregate`
    referenced back in Chapter 1's preview gets laid down); the actual
    parallel scheduling machinery is Chapter 14's subject.
42. **`JOIN_FILTER_PUSHDOWN`** — `optimizer.cpp:543`,
    `src/optimizer/join_filter_pushdown_optimizer.cpp` (+
    `src/optimizer/runtime_filter_cast.cpp`). Per its own inline comment:
    *"perform join filter pushdown after the dust has settled"* — sets up
    the **dynamic, execution-time** filter described in §7.6.5: a hash
    join's build side will record the min/max of its join key while building
    the hash table, and this pass arranges for that range to be usable as a
    runtime pre-filter on the probe side's scan. Deliberately placed last
    among the structural passes — "after the dust has settled" — because it
    needs the join tree, join keys, and column types to already be in their
    final shape.
43. **`ROW_NUMBER_REWRITER`** — `optimizer.cpp:549`,
    `src/optimizer/row_number_rewriter.cpp`. Per its inline comment:
    *"Rewrite ROW_NUMBER() OVER() window functions to use the row_number
    virtual column"* — a bare, frame-less `ROW_NUMBER()` doesn't need the
    general window-function evaluator at all if the storage layer can supply
    a row-numbering virtual column directly.
44. **`TYPE_PUSHDOWN`** — `optimizer.cpp:555`, `src/optimizer/type_pushdown.cpp`.
    Per its inline comment: *"Push down type casts in SELECT e.g. SELECT
    num::UHUGEINT to file readers"* — lets a file-format reader (Parquet,
    CSV) produce a column already cast to the type the query actually wants,
    instead of the reader's native type followed by a separate cast
    operator.
45. **`SCALAR_FN_PUSHDOWN`** — `optimizer.cpp:561`, `src/optimizer/scalar_fn_pushdown.cpp`.
    Per its inline comment: *"Pushdown scalar functions in SELECT like
    SELECT strlen(str) to readers"* — the scalar-function analogue of #44:
    some readers can compute a scalar function of a column (like a string
    length) during the scan itself, avoiding a separate expression-executor
    pass over the whole column afterward.

Two extension-hook points bracket this entire sequence, each wrapped in its
own `RunOptimizer(OptimizerType::EXTENSION, ...)` call so third-party passes
are timed and toggleable exactly like the built-in ones:
**pre-optimize extension hooks** (`optimizer.cpp:594`, run once per
registered `OptimizerExtension` *before* `RunBuiltInOptimizers()` is called)
and **post-optimize extension hooks** (`optimizer.cpp:606`, run *after*).

### 7.4.1 `SELF_JOIN_AGGREGATE` (#3)

Its enum value is `OptimizerType::SELF_JOIN_AGGREGATE = 47`, and its name in
`duckdb_optimizers()` and `disabled_optimizers` is `self_join_aggregate`
(`src/common/enums/optimizer_type.cpp:62`).

**The problem.** A *self-join* joins a table to itself. TPC-DS query 95 has
one. It wants to know which orders ship from more than one warehouse. It
asks this by pairing every row of an order with every other row of the
same order, and keeping pairs whose warehouses differ. The file's opening
comment shows the query:

```cpp
// src/optimizer/self_join_aggregate.cpp:1
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

An order with 10 rows makes up to 90 pairs in the join. The query only
needs one fact per order: "are there two different warehouses?" That fact
is the same as `min(warehouse) <> max(warehouse)` for the order. So the
pass rewrites the plan like this:

```
Before                                   After
──────                                   ─────
(parent that ignores duplicates,         (same parent)
 e.g. the right side of an IN)             └─ LogicalProjection (K; other
  └─ LogicalProjection (K, a.x, b.x)            columns become typed NULLs)
       └─ LogicalFilter (a.K = b.K               └─ LogicalFilter (min(x) <> max(x))
                         AND a.x <> b.x)              └─ LogicalAggregate (GROUP BY K;
            └─ LogicalCrossProduct                           min(x), max(x))
                 ├─ LogicalGet (T as a)                    └─ LogicalFilter (K IS NOT NULL)
                 └─ LogicalGet (T as b)                         └─ LogicalGet (T)
```

The code that builds the "After" side is at
`self_join_aggregate.cpp:509-560`. It keeps the old projection and its table
index, so operators above it need no change. Each key column now reads from
the group-by. Each other column becomes a typed `NULL`, since nothing above
reads it. The later passes push the `IS NOT NULL` filter into the scan, and
the join order optimizer places the group-by like any other subtree.

**Why the answer is the same.** The source explains it in the same comment
(`self_join_aggregate.cpp:12-17`). In simple terms:

- The join keeps a pair of rows only if both have the same key, and the key
  is not `NULL` (`=` never matches `NULL`).
- It keeps the pair only if both `x` values are not `NULL` and differ.
- So a key appears in the join output exactly when its group has two
  different non-`NULL` values of `x`.
- `min(x)` and `max(x)` skip `NULL`s. They differ exactly when the group has
  two different non-`NULL` values. A group where every `x` is `NULL` gives
  `NULL <> NULL`, which is `NULL`, so the `HAVING` drops it.
- The pass also filters out the `NULL` key group first. In the join, a
  `NULL` key never pairs with anything. And an `IN` list that holds a `NULL`
  turns a `FALSE` answer into `NULL`.

The two plans give the same *set* of keys. They do not give the same
*number* of rows: the join gives one row per pair, the group-by one row per
key. That is why the parent must not care about duplicates.

**Which parents ignore duplicates.** The pass walks from the root down
(`self_join_aggregate.cpp:19-27`). These parents qualify:

- the right side of a `SEMI`, `ANTI` or `MARK` join (and the left side of a
  `RIGHT_SEMI` or `RIGHT_ANTI` join). These only ask "is there a match?";
- a `DISTINCT`;
- a `GROUP BY` whose aggregates are all `min`, `max`, or `DISTINCT`
  aggregates (`AggregateIgnoresDuplicates`, `self_join_aggregate.cpp:94-112`).

Below such a parent, the property passes through projections and filters
without volatile expressions, cross products, and `INNER` joins. Anything
else stops it. A `UNION` stops it too, because a union reads its inputs'
columns by position, and the pass could not see which columns it reads. A
materialized CTE's body qualifies when every reference to the CTE does.

**What the join must look like** (`self_join_aggregate.cpp:29-39`):

- two scans of the same table (or the same repeatable table function with
  equal bind data), with no table filters, time travel, sampling or
  ordinality;
- every join or filter condition is `=` or `<>` between the *same* column
  of the two scans;
- at least one `=` column (these form the key K) and exactly one `<>`
  column (x), with x not part of K;
- K and x are integer or `DATE` columns. For these types, ordering agrees
  with equality. A collated `VARCHAR` might not: two strings can be "equal"
  under a collation and still have a different min and max;
- the columns that are read above the projection must all be key columns.

**Where it runs, and why there.** It runs right after the first
`CTE_INLINING` and before `FILTER_PULLUP`/`FILTER_PUSHDOWN`:

```cpp
// src/optimizer/optimizer.cpp:265
	// Rewrites a self-join on equal keys and one inequality, read only through its keys by a parent that ignores
	// duplicates, into GROUP BY keys HAVING min(x) <> max(x) over one scan. Runs before filter pushdown, which
	// would move the join's filters into the scans and hide the pattern
	RunOptimizer(OptimizerType::SELF_JOIN_AGGREGATE, [&]() {
		SelfJoinAggregate self_join_aggregate(*this);
		plan = self_join_aggregate.Optimize(std::move(plan));
	});
```

If filter pushdown ran first, it would turn `a.o = b.o` into a join
condition and spread the other filters into the scans. The simple shape the
pass looks for (filters over a cross product of two scans) would be gone.

**Worked example.** `test/sql/optimizer/self_join_aggregate.test` runs each
query twice: once with the pass turned off and once with it on. Both runs
must give the same answer:

```sql
-- test/sql/optimizer/self_join_aggregate.test
loop i 0 2

statement ok
SET disabled_optimizers = CASE WHEN {i} = 0 THEN 'self_join_aggregate' ELSE '' END;

-- q95's shape: a CTE read by two INs, one of them through a join with another table
query III
WITH ws_wh AS (SELECT a.o, a.w AS w1, b.w AS w2 FROM ws a, ws b WHERE a.o = b.o AND a.w <> b.w)
SELECT count(*), count(DISTINCT x), sum(x) FROM q
WHERE x IN (SELECT o FROM ws_wh) AND x IN (SELECT k FROM r, ws_wh WHERE k = ws_wh.o)
----
545	140	70583
```

Then it checks the plan. With the pass on, the plan has a `Group By` with
`min(` and `max(`, and only one scan of `ws`:

```sql
query II
EXPLAIN SELECT DISTINCT a.o, a.p FROM ws a, ws b WHERE a.o = b.o AND a.p = b.p AND a.w <> b.w
----
physical_plan	<!REGEX>:[\s\S]*Seq Scan[\s\S]*Seq Scan[\s\S]*
```

This second query has two key columns (`o` and `p`) under a `DISTINCT`.
With `SET disabled_optimizers = 'self_join_aggregate'`, the same `EXPLAIN`
shows two `Seq Scan`s again.

### 7.4.2 `REDUNDANT_ORDER_KEYS` (#21)

Its enum value is `OptimizerType::REDUNDANT_ORDER_KEYS = 46`, and its name is
`redundant_order_keys` (`src/common/enums/optimizer_type.cpp:61`). It was
written for DuckDB issue #24000.

**The idea.** Sorting compares rows key by key. The second key only matters
when two rows tie on the first key. If the first key is *unique* (no two
rows share a value), there are never ties, so every key after it is
useless. The pass removes those keys:

```
ORDER BY id, payload, grp      (id is the PRIMARY KEY)
         ──  ───────────
         │   never used: no two rows have the same id
         unique prefix
                 │
                 ▼
ORDER BY id
```

Fewer sort keys means cheaper comparisons. It can also mean fewer columns.
If `payload` was read only for the sort, the later `UNUSED_COLUMNS` pass
(#23) can then stop reading it. That is why this pass runs *before*
`UNUSED_COLUMNS`. It sits right after `JOIN_ELIMINATION`:

```cpp
// src/optimizer/optimizer.cpp:389
	RunOptimizer(OptimizerType::REDUNDANT_ORDER_KEYS, [&]() {
		RemoveRedundantOrderKeys remove_redundant_order_keys;
		remove_redundant_order_keys.Optimize(*plan);
	});
```

**How it knows a key is unique.** The pass is a `LogicalOperatorVisitor`
called `ApplyFunctionalDependencies`
(`src/optimizer/remove_redundant_order_keys.cpp:19`). (A *functional
dependency* means: "if you know these columns, the other columns are
fixed.") It visits the plan bottom-up. Each operator hands up a list of
column sets that are known to be unique (`result`). The sources of unique
sets are:

- **A table scan** (`VisitLogicalGet`, lines 36-84). It reads the table's
  `UNIQUE` and `PRIMARY KEY` constraints. A plain `UNIQUE` column counts only
  if it is also `NOT NULL`, because a `UNIQUE` column may hold many `NULL`s
  (lines 73-77). It works only for DuckDB's own tables (`IsDuckTable()`), and
  only when all the key columns are actually scanned.
- **A `GROUP BY`** (lines 86-111). Its group columns are unique in its
  output, as long as there is a single grouping set.
- **A `DISTINCT`** (lines 113-131). Its target columns are unique, if they
  are all plain column references.

Some operators keep the sets: filters and limits pass them through
unchanged, and projections pass them on when they forward the columns
unchanged (lines 133-164). Every other operator (a join, for example)
clears the list, because it may duplicate rows:

```cpp
// src/optimizer/remove_redundant_order_keys.cpp:217
	default:
		VisitOperatorChildren(op);
		//	Clear the keys because we don't know if the operator preserves them.
		result.clear();
		break;
```

At an `ORDER BY`, the pass grows a prefix of key columns, one key at a
time. As soon as the prefix covers a full unique set, it cuts every key
after that point:

```cpp
// src/optimizer/remove_redundant_order_keys.cpp:172
	column_binding_set_t prefix;
	for (idx_t i = 0; i + 1 < order.orders.size(); i++) {
		auto &expr = *order.orders[i].expression;
		if (expr.GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
			continue;
		}
		prefix.insert(expr.Cast<BoundColumnRefExpression>().Binding());
		for (auto &unique_set : result) {
			bool covered = true;
			for (auto &binding : unique_set) {
				if (prefix.find(binding) == prefix.end()) {
					covered = false;
					break;
				}
			}
			if (!covered) {
				continue;
			}
			order.orders.erase(order.orders.begin() + NumericCast<int64_t>(i + 1), order.orders.end());
			return;
		}
	}
```

Only plain column references count. An expression over a unique column,
such as `CAST(id AS BOOLEAN)`, is not unique, so it is skipped.

**Worked example.** `test/optimizer/remove_redundant_order_keys.test`:

```sql
-- test/optimizer/remove_redundant_order_keys.test
CREATE TABLE t(id INTEGER PRIMARY KEY, payload VARCHAR, grp INTEGER);

-- a primary key prefix erases the keys behind it, through the filter and the limit around it
EXPLAIN SELECT id FROM t WHERE id + grp > 0 ORDER BY id, payload LIMIT 2;
-- physical_plan  <REGEX>:.*Order By: id ASC +│.*

-- the freed column is pruned by the later passes, so the pass has to run before them
EXPLAIN SELECT id FROM t ORDER BY id, payload;
-- physical_plan  <!REGEX>:.*payload.*

-- keys before the unique prefix stay
EXPLAIN SELECT id FROM t ORDER BY payload, id, grp;
-- physical_plan  <REGEX>:.*Order By: [^\n]*id ASC +│.*

-- a join duplicates the unique column
EXPLAIN SELECT t.id FROM t JOIN t AS t2 ON t.grp = t2.grp ORDER BY t.id, t2.payload;
-- physical_plan  <REGEX>:.*Order By: [^\n]*, [^\n]*│.*
```

In the third query, `payload` comes first and is not unique, so it stays.
Once `id` joins the prefix, the prefix is unique and `grp` is dropped. In the
fourth query, the self-join can repeat each `t.id`, so nothing is dropped.

The same test shows the rule for `UNIQUE` without `NOT NULL`:

```sql
CREATE TABLE u(ukey INTEGER UNIQUE, skey VARCHAR UNIQUE NOT NULL, tiebreak INTEGER);

EXPLAIN SELECT ukey FROM u ORDER BY ukey, tiebreak;   -- both keys stay
EXPLAIN SELECT skey FROM u ORDER BY skey, tiebreak;   -- Order By: skey ASC
```

`ukey` can hold two `NULL`s, which tie, so `tiebreak` still matters.

**A side effect worth knowing.** A dropped key is never evaluated. So an
error or a side effect inside it disappears too.
`test/optimizer/remove_redundant_order_keys_side_effects.test` checks this:

```sql
-- test/optimizer/remove_redundant_order_keys_side_effects.test
-- a dropped key never raises, and never runs its side effect
SELECT count(*) FROM (SELECT id FROM t ORDER BY id, CAST(payload AS INTEGER)) q;
-- 5
SELECT count(*) FROM (SELECT id FROM t ORDER BY id, nextval('seq_elide')) q;
-- 5
SELECT last_value IS NULL FROM duckdb_sequences() WHERE sequence_name = 'seq_elide';
-- true
```

`CAST('a' AS INTEGER)` would fail, but it never runs. The sequence is never
advanced. A key *before* the unique prefix (`ORDER BY nextval('seq_keep'),
id`) is still evaluated. With `SET disabled_optimizers =
'redundant_order_keys'`, the same test expects the cast query to fail with a
`Conversion Error`. So turning this pass off can change whether a query
fails. For the same reason, the test run with the whole optimizer off
(`test/configs/disable_optimizer.json`) skips this test file.

### What's conspicuously absent from this numbered list, and why

Two things worth flagging explicitly, in the spirit of this book's policy of
stating uncertainty rather than papering over it:

**`REMOTE_PUSHDOWN`** is a real, registered `OptimizerType`, but it is not
one of the ~45 passes above — it's invoked from a *different* method,
`Optimizer::OptimizeStatement` (`optimizer.cpp:223`), which operates on a
`unique_ptr<SQLStatement>&` rather than a `LogicalOperator` tree, and only
runs at all when the database has at least one remote catalog attached
(`DatabaseManager::Get(context).GetRemoteCatalogCount() > 0`). It's a
statement-level rewrite for a different input type entirely, not a step in
the plan-level pipeline this section reproduces.

**`MATERIALIZED_CTE`** is declared in the `OptimizerType` enum
(`src/include/duckdb/common/enums/optimizer_type.hpp`), and `duckdb_optimizers()`
and `disabled_optimizers` both know its name — but a full-text search of
`src/` for `OptimizerType::MATERIALIZED_CTE` turns up nothing outside the two
enum-name-mapping tables (`src/common/enum_util.cpp`,
`src/common/enums/optimizer_type.cpp`). Nothing in this checkout's optimizer
actually calls `OptimizerDisabled(OptimizerType::MATERIALIZED_CTE)`. It's
possible this gates behavior somewhere this search didn't reach (a
compilation unit outside `src/optimizer/`, or logic that's been superseded
by `CTE_INLINING`/`COMMON_SUBPLAN` without the old toggle being removed) —
this is flagged rather than guessed at, per this book's sourcing policy.
The two tables are at `enum_util.cpp:4410` and `optimizer_type.cpp:42`.

**`DISTINCT_AGGREGATE_REWRITE`** and **`COMPRESSED_MATERIALIZATION`**, by
contrast, *are* real, live toggles — they're just not wrapped in their own
`RunOptimizer(...)` call sites, and so don't get their own line in the
numbered list or their own profiler timer. `DISTINCT_AGGREGATE_REWRITE` gates
the `MultiStageAggregateRewriter` invocation folded into pass #11
(`optimizer.cpp:329`); `COMPRESSED_MATERIALIZATION` gates a check inside
`STATISTICS_PROPAGATION` itself (`statistics_propagator.cpp:138`) that
decides whether to apply a compressed intermediate representation for
certain materialized results. Both are real, disableable, and listed by
`duckdb_optimizers()` — they're just implemented as inline policy checks
inside a *different* pass's callback rather than as passes of their own.

## 7.5 The expression-rewriter rule set (pass #1, in full)

The full, fixed, ordered list of 37 rules, exactly as pushed in the
`Optimizer` constructor:

```cpp
// src/optimizer/optimizer.cpp:73
Optimizer::Optimizer(Binder &binder, ClientContext &context) : context(context), binder(binder), rewriter(context) {
	rewriter.rules.push_back(make_uniq<ConstantOrderNormalizationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<ConstantFoldingRule>(rewriter));
	rewriter.rules.push_back(make_uniq<StructExtractStructPackFoldingRule>(rewriter));
	rewriter.rules.push_back(make_uniq<DistributivityRule>(rewriter));
	rewriter.rules.push_back(make_uniq<ArithmeticSimplificationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<LeastGreatestSimplificationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<CaseSimplificationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<ConjunctionSimplificationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<DatePartSimplificationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<DateTruncSimplificationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<ComparisonSimplificationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<RowComparisonSimplificationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<InClauseSimplificationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<InEnumSimplificationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<EnumCompareSimplificationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<EqualOrNullSimplification>(rewriter));
	rewriter.rules.push_back(make_uniq<MoveConstantsRule>(rewriter));
	rewriter.rules.push_back(make_uniq<MoveUnaryMinusRule>(rewriter));
	rewriter.rules.push_back(make_uniq<MonotonePreimageRule>(rewriter));
	rewriter.rules.push_back(make_uniq<LikeOptimizationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<StringPrefixRule>(rewriter));
	rewriter.rules.push_back(make_uniq<InstrPrefixRule>(rewriter));
	rewriter.rules.push_back(make_uniq<OrderedAggregateOptimizer>(rewriter));
	rewriter.rules.push_back(make_uniq<DistinctAggregateOptimizer>(rewriter));
	rewriter.rules.push_back(make_uniq<DistinctWindowedOptimizer>(rewriter));
	rewriter.rules.push_back(make_uniq<RegexOptimizationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<RegexpReplaceExtractRule>(rewriter));
	rewriter.rules.push_back(make_uniq<EmptyNeedleRemovalRule>(rewriter));
	rewriter.rules.push_back(make_uniq<NoopReplaceRemovalRule>(rewriter));
	rewriter.rules.push_back(make_uniq<EnumComparisonRule>(rewriter));
	rewriter.rules.push_back(make_uniq<JoinDependentFilterRule>(rewriter));
	rewriter.rules.push_back(make_uniq<TimeStampComparison>(rewriter));
	rewriter.rules.push_back(make_uniq<PredicateFactoringRule>(rewriter));
	rewriter.rules.push_back(make_uniq<ListComprehensionRewriteRule>(rewriter));
	rewriter.rules.push_back(make_uniq<ContainsToInClauseRule>(rewriter));
	rewriter.rules.push_back(make_uniq<NotComparisonSimplificationRule>(rewriter));
	rewriter.rules.push_back(make_uniq<NotConjunctionSimplificationRule>(rewriter));
}
```

That's 37 rules. Immediately after, a debug-only assertion checks that every
rule actually defines a pattern to match against — a rule with no `root`
matcher would silently never fire, which would be a bug worth catching in
debug builds specifically:

```cpp
// src/optimizer/optimizer.cpp:112
#ifdef DEBUG
	for (auto &rule : rewriter.rules) {
		// root not defined in rule
		D_ASSERT(rule->root);
	}
#endif
```

Each rule is a small, independent class under `src/optimizer/rule/*.cpp`,
built around the same two-part shape: a matcher (`root`, built at
construction time from combinators like `ComparisonExpressionMatcher`,
`FunctionExpressionMatcher`, `ConstantExpressionMatcher`) describing the
expression shape it looks for, and an `Apply` method that runs once the
matcher finds a hit. `ExpressionRewriter::VisitOperator` walks the whole tree
once, and at every expression node tries every rule in the list in order,
applying the first (or, depending on the rule, every) match. Unlike the
plan-level pipeline in §7.4, this *inner* loop does re-apply — if one rule's
rewrite exposes a pattern a rule earlier in the list would have matched,
`VisitOperator`'s expression-tree walk still visits the newly-produced
subexpression and rule matching runs against it too. The one-shot,
no-fixpoint architecture described in §7.2 is a property of the ~45
*plan-level* passes, not of this inner expression-rewrite loop specifically
— though the *plan-level* `EXPRESSION_REWRITER` pass as a whole still only
gets to run once, at position #1, before the plan-shaping passes take over.

A handful of these rules are worth seeing concretely, both because they're
illustrative of how the matcher/rule split works in practice and because a
few of them come up again in Query A's trace (§7.7):

**`ConstantFoldingRule`** (`rule/constant_folding.cpp`) is the simplest:
match any foldable, non-constant expression, evaluate it once with the
`ExpressionExecutor`, and replace it with a `BoundConstantExpression` holding
the result.

```cpp
// src/optimizer/rule/constant_folding.cpp:28
unique_ptr<Expression> ConstantFoldingRule::Apply(LogicalOperator &op, vector<reference<Expression>> &bindings,
                                                  bool &changes_made, bool is_root) {
	auto &root = bindings[0].get();
	// ...
	Value result_value;
	if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), root, result_value)) {
		return nullptr;
	}
	// ...
	return make_uniq<BoundConstantExpression>(result_value);
}
```

`1 + 1` folds to `2` before execution ever begins — no vectorized addition
runs at query time for a compile-time-constant expression like this.

**`ComparisonSimplificationRule`** (`rule/comparison_simplification.cpp`)
handles the type-specific case where a comparison provably has a fixed
answer regardless of one side's runtime value — for instance, comparing a
`DATE` against a non-midnight `TIMESTAMP` constant: a `DATE` value is always
exactly midnight, so it can never equal a timestamp with a nonzero
time-of-day component, and the rule folds the comparison directly, careful
to preserve `NULL` propagation:

```cpp
// src/optimizer/rule/comparison_simplification.cpp:30
case ExpressionType::COMPARE_EQUAL:
	// d =  T   -> false, preserving NULL
	replacement = ExpressionRewriter::ConstantOrNull(
	    context, std::move(BoundCastExpression::ChildMutable(cast_expression)), Value::BOOLEAN(false));
	return true;
```

So `some_date_column = TIMESTAMP '2020-01-01 00:00:01'` folds directly to
`some_date_column IS NOT NULL AND FALSE` (the `ConstantOrNull` helper keeps
the column's `NULL`-ness observable) without ever comparing a single row at
runtime.

The helper refuses an *infinite* timestamp (`'infinity'`, `'-infinity'`).
It returns `false` before any of the cases above
(`comparison_simplification.cpp:19-22`). The rule also allows a narrowing
`DECIMAL` cast of the constant when the constant round-trips exactly. Part 02
covers both.

**`CaseSimplificationRule`** (`rule/case_simplification.cpp`) folds any
`WHEN` branch whose condition is itself foldable: `CASE WHEN 1 = 1 THEN 'a'
ELSE 'b' END` becomes just `'a'`, because the rule evaluates the foldable
`WHEN` check, finds it's always true, and moves that branch's `THEN`
expression directly into the `ELSE` slot, discarding every subsequent
branch.

If a foldable `WHEN` condition cannot be evaluated at plan time (for example,
it would raise an error), the rule skips that branch and leaves it for
execution (`case_simplification.cpp:22-27`). So an error is not raised while
planning for a branch that would never run.

**`LikeOptimizationRule`** (`rule/like_optimizations.cpp`) matches on the
`LIKE`/`NOT LIKE`/`GLOB`/`NOT GLOB` operators with a constant pattern, and
classifies the pattern using two helpers, `PatternIsConstant` (no wildcards
at all — degenerates to plain equality) and `PatternIsPrefix` (a trailing
run of wildcards and nothing else). `x LIKE 'foo%'` is recognized by
`PatternIsPrefix` as exactly a prefix constraint — this is the rewrite DiDi
[10] highlights as opening up a *prefix-range scan* opportunity: once a `LIKE`
predicate is known to mean "starts with `foo`," it can be handled the same
way a `>= 'foo' AND < 'fop'` range predicate would be, which a zonemap or an
ART index can actually use, unlike an opaque pattern-match call.

The rule also matches `ILIKE` (`~~*`) and `NOT ILIKE` (`!~~*`), but only to
fold a `NULL` pattern to a `NULL` result. Any other `ILIKE` is left alone
(`like_optimizations.cpp:139-146`). A constant pattern that cannot be
evaluated does not fail planning; the rule just returns. Part 03 covers both.

**`StringPrefixRule`/`InstrPrefixRule`** (`rule/string_prefix.cpp`) catch a
related but syntactically different shape: an explicit prefix-extracting
function call compared to a constant, such as
`substring(name, 1, 3) = 'Bob'` or `left(name, 3) = 'Bob'`. Rather than
computing the three-character substring for every row and comparing it,
the rule rewrites this into a direct range/prefix comparison on `name`
itself — the same storage-layer benefit `LikeOptimizationRule` unlocks for
`LIKE`, reached from a different starting syntax.

**`InClauseSimplificationRule`** (`rule/in_clause_simplification_rule.cpp`) is
worth pausing on because its name is easy to confuse with the *plan-level*
`IN_CLAUSE` pass (#9 in §7.4) — they solve different problems entirely. This
*expression*-level rule fires when the left side of an `IN` list is a cast
of a column, and both the cast and its inverse are provably lossless
(`BoundCastExpression::CastIsInvertible`): `CAST(x AS BIGINT) IN (1, 2, 3)`
gets rewritten to push the cast onto the three *constants* instead, leaving
`x IN (CAST(1 AS <x's original type>), ...)` — so the column itself is
compared in its native type, which means a zonemap or index on `x` can still
be used directly, instead of `x` being hidden inside a cast expression the
storage layer can't see through. The plan-level `IN_CLAUSE` pass, by
contrast, is about turning a *long* `IN` list into a join (§7.6.6) — the two
share a name only by coincidence of English vocabulary.

The rule skips `TRY_CAST`. A `TRY_CAST` turns a failing value into `NULL`,
and that `NULL` would be lost if the cast were moved off the column
(`in_clause_simplification_rule.cpp:27-30`). Part 04 owns this rule.

**`DistinctAggregateOptimizer`** (`rule/distinct_aggregate_optimizer.cpp`)
checks whether an aggregate function marked `DISTINCT` is actually
*distinct-sensitive* — some aggregates (`MIN`, `MAX`) give the same answer
whether or not duplicate inputs are removed first. If
`Function().GetDistinctDependent()` reports `NOT_DISTINCT_DEPENDENT`, the
rule silently clears the `DISTINCT` flag: `MAX(DISTINCT x)` is rewritten to
behave as plain `MAX(x)`, skipping the (potentially expensive) deduplication
step entirely since it could never have changed the answer.

**`RegexOptimizationRule`** (`rule/regex_optimizations.cpp`) inspects a
`regexp_matches(x, pattern)` call whose pattern is a compile-time constant,
and — via the `LikeString` conversion helpers — checks whether that pattern
is actually a plain literal string with no regex metacharacters at all. A
call like `regexp_matches(x, 'abc')` with no special characters anywhere in
`'abc'` doesn't need RE2's full regex engine invoked per row; it's
functionally a `CONTAINS`/`LIKE` check, and the rule rewrites it to one,
which is considerably cheaper per row than compiling and running a regex
match.

An anchor (`^` or `$`) in the *middle* of a pattern stops the rewrite. A
`LIKE` pattern can only express anchors at its two ends
(`regex_optimizations.cpp:137-148`). Part 03 covers this.

## 7.6 Selected passes in depth

The passes below are the ones this chapter is specifically asked to go
deeper on — either because DiDi's course [10] covers their mechanics with worked
numeric detail, or because §7.4's one-liner leaves an open question (the
`sum_rewriter` naming question, most notably) worth resolving in full.

### 7.6.1 `REORDER_FILTER` — cheap-to-costly conjunct ordering

`ExpressionHeuristics` (`src/optimizer/expression_heuristics.cpp`) is the
pass DiDi calls `reorder_filter` [10]; in DuckDB's own `OptimizerType` vocabulary
it's `REORDER_FILTER`, and it runs late (#40 in §7.4's list), after every
structural rewrite that could still add or remove conjuncts has already had
its turn. Its job: given a filter with multiple `AND`-ed (or `OR`-ed)
conjuncts, evaluate the cheap ones first, so a row that fails a cheap check
never pays for an expensive one:

```cpp
// src/optimizer/expression_heuristics.cpp:37
void ExpressionHeuristics::ReorderExpressions(vector<unique_ptr<Expression>> &expressions) {
	// ...
	for (idx_t i = 0; i < expressions.size(); i++) {
		if (expressions[i]->CanThrow() || ExpressionBarrier::Contains(*expressions[i])) {
			// do not allow reordering if an expression can throw, or carries a barrier
			return;
		}
	}

	vector<ExpressionCosts> expression_costs;
	expression_costs.reserve(expressions.size());
	// iterate expressions, get cost for each one
	for (idx_t i = 0; i < expressions.size(); i++) {
		idx_t cost = Cost(*expressions[i]);
		expression_costs.push_back({std::move(expressions[i]), cost});
	}

	// sort by cost and put back in place
	sort(expression_costs.begin(), expression_costs.end());
	for (idx_t i = 0; i < expression_costs.size(); i++) {
		expressions[i] = std::move(expression_costs[i].expr);
	}
}
```

The guard at the top is exactly the caution DiDi's slides flag [10]: if *any*
conjunct in the list `CanThrow()` — a cast that might fail, a division that
might hit zero — the whole list is left untouched, because reordering could
move a row-filtering conjunct (one that would have excluded the
error-triggering row) to *after* the conjunct that throws, turning a query
that used to succeed (because the bad row never reached the throwing check)
into one that errors out. `ReorderExpressions` runs both directly on
multi-expression `LogicalFilter`s and, via `VisitReplace`, on the children of
any `BoundConjunctionExpression` anywhere in the tree — so `WHERE a AND b`
gets reordered even when it isn't sitting directly under a `LogicalFilter`
node (for instance, inside a `CASE WHEN` condition).

### 7.6.2 `STATISTICS_PROPAGATION` — and a correction to Chapter 1's preview

`StatisticsPropagator` (`src/optimizer/statistics_propagator.cpp`) walks the
tree bottom-up, and for every operator type it knows about, derives output
statistics (min/max, distinct-count estimate, null fraction) from its
children's statistics and its own expressions — a `LogicalFilter` on
`x > 100` narrows `x`'s known range on the way out; a projection computing
`x + 1` shifts `x`'s min/max by one; a join combines both sides' relevant
column statistics. Two concrete effects fall out of this:

**Constant-folding-style pruning.** When a filter's known input statistics
prove it can never be satisfied — `WHERE x > 1000000` where `x`'s propagated
maximum is `500` — the pass doesn't just fold the filter's *expression* to
`FALSE` (pass #1 already handled genuinely constant expressions); it
replaces the entire unsatisfiable subtree with a `LogicalEmptyResult`
directly:

```cpp
// src/optimizer/statistics_propagator.cpp:41
void StatisticsPropagator::ReplaceWithEmptyResult(unique_ptr<LogicalOperator> &node) {
	node = make_uniq<LogicalEmptyResult>(std::move(node));
}
```

This is a genuinely data-dependent decision (it depends on what min/max the
column's statistics say, which depends on what's actually stored), unlike
pass #1's rules, which only ever fold expressions whose truth doesn't depend
on any table's contents at all.

**Feeding the cost-based aggregate rewriter and projection placement.**
Immediately after `STATISTICS_PROPAGATION` runs, `RunBuiltInOptimizers`
re-invokes `MultiStageAggregateRewriter` with a `COST_BASED` policy (using
the statistics map this pass just built), and, if that second rewriter pass
changed anything, re-propagates statistics once more before running
`PROJECTION_PLACEMENT` (#35) — both of these genuinely depend on this pass's
output.

Here is the correction worth flagging explicitly, in the spirit of this
book's stated policy that later chapters give full evidence where an
earlier chapter's preview was necessarily shallow: **Chapter 1, §1.3 said
`statistics_propagation` runs early and "feeds the next two passes" (listed
right before `join_order`), estimating the cardinalities that join ordering
needs.** The real, fixed pipeline order in §7.4 shows this isn't what
happens. `STATISTICS_PROPAGATION` is pass #34 — it runs *after* `JOIN_ORDER`
(#17) and `BUILD_SIDE_PROBE_SIDE` (#27), not before either of them. What
actually feeds the DPhyp join-order search's cost model is a separate,
purpose-built, lighter-weight estimator living inside
`src/optimizer/join_order/cardinality_estimator.cpp`, invoked from within
the `JOIN_ORDER` pass itself — not the global, tree-wide statistics
propagator this section describes. The two are easy to conflate because
they're both, in a loose sense, "cardinality estimation," but they're
different code, running at different points in the pipeline, for different
consumers: the join-order optimizer's own estimator exists specifically to
cost alternative join orders during its search (Chapter 8's subject); the
global `STATISTICS_PROPAGATION` pass exists to prove filters vacuous and to
inform the cost-based aggregate rewrite and projection placement that
immediately follow it in the fixed sequence. Chapter 1's preview wasn't
wrong about statistics propagation existing or mattering — it was
necessarily shallow (as its own framing said it would be) about exactly
*which* cardinality numbers feed *which* downstream decision.

### 7.6.3 `FILTER_PULLUP` and `FILTER_PUSHDOWN` — the two-directional filter mechanism

These two passes (#5 and #6, run back-to-back) are, together, the mechanism
behind the single most consequential rewrite an OLAP query typically
benefits from: getting a `WHERE` predicate evaluated as early as possible,
ideally as part of the base-table scan itself.

**`FilterPushdown`** (`src/optimizer/filter_pushdown.cpp`, using
`filter_combiner.cpp` to merge/simplify accumulated filter sets, plus
thirteen per-operator-type handlers under `src/optimizer/pushdown/`)
recursively walks the tree carrying a set of "filters still waiting to be
applied," picking them up at every `LogicalFilter` it encounters and
pushing them down through whatever operator sits below, as far as each
operator's own handler (`pushdown_inner_join.cpp`, `pushdown_get.cpp`, and
so on) allows. An inner join's handler can push a predicate referencing only
one side's columns all the way past the join and onto that side alone; a
predicate referencing both sides has to stay at (or above) the join as an
actual join condition. Reaching a `LogicalGet` is the best case: the
predicate becomes part of the scan's own table filter set, which the
storage layer can use for zonemap-based row-group skipping (Chapter 15) —
this is exactly what happens to `l.shipdate < DATE '1998-01-01'` in Query A
(§7.7).

**`FilterPullup`** (`src/optimizer/filter_pullup.cpp`) runs in the opposite
direction and one step earlier in the pipeline (#5, before pushdown at #6),
and does something pushdown alone can't: for an equi-join, a filter that's
already known to hold on one side can be *pulled up* through the join and
re-derived as an equivalent filter on the *other* side too — so, for a join
on `a.k = b.k` with a known filter `a.k > 100` on the left, `FilterPullup`
lets the optimizer also apply `b.k > 100` on the right side, a predicate
that wasn't written anywhere in the original SQL but is a valid consequence
of the join condition plus the existing filter. Once that derived predicate
exists, the very next pass, `FilterPushdown`, can push it down into `b`'s
own scan exactly as if the user had written it there directly — the two
passes are deliberately sequenced back-to-back so pullup's newly-derived
predicates get a pushdown pass immediately available to act on them, rather
than only being useful the *next* time the fixed pipeline happens to loop
back around (which, per §7.2's architecture, it never does within a single
query).

### 7.6.4 `TOP_N` — fusing `ORDER BY` + `LIMIT`, conditionally

`TopN::CanOptimize` (`src/optimizer/topn_optimizer.cpp:22`) is the exact
condition under which `LogicalOrder` immediately followed by `LogicalLimit`
gets fused into a single `LogicalTopN`, and it's more than just "is there a
constant limit" — it includes a cost heuristic that can decline the fusion
outright:

```cpp
// src/optimizer/topn_optimizer.cpp:22
bool TopN::CanOptimize(LogicalOperator &op, optional_ptr<ClientContext> context) {
	if (op.type == LogicalOperatorType::LOGICAL_LIMIT) {
		auto &limit = op.Cast<LogicalLimit>();

		if (limit.limit_val.Type() != LimitNodeType::CONSTANT_VALUE) {
			// we need LIMIT to be present AND be a constant value for us to be able to use Top-N
			return false;
		}
		if (limit.offset_val.Type() == LimitNodeType::EXPRESSION_VALUE) {
			// we need offset to be either not set (i.e. limit without offset) OR have offset be
			return false;
		}
		// ...
		if (child_op->has_estimated_cardinality) {
			// only check if we should switch to full sorting if we have estimated cardinality
			auto constant_limit = static_cast<double>(limit.limit_val.GetConstantValue());
			// ...
			auto child_card = static_cast<double>(child_op->estimated_cardinality);

			// if the limit is > 0.7% of the child cardinality, sorting the whole table is faster
			bool limit_is_large = constant_limit > 5000;
			if (constant_limit > child_card * 0.007 && limit_is_large) {
				return false;
			}
		}
```

The reasoning behind the 0.7%-of-cardinality-and->5000 guard: a bounded
top-k structure only wins over a plain full sort when the `k` you're keeping
is small relative to the input — past some point, the bookkeeping overhead
of maintaining a top-k heap loses to just sorting everything once. DuckDB's
heuristic only backs off to a full sort when the limit is *both* a sizeable
fraction of the input *and* large in absolute terms (`> 5000`) — a query
asking for `LIMIT 5001` out of ten million rows still gets `LogicalTopN`,
since 5001 rows is nowhere near 0.7% of ten million; only a limit that's
large on *both* axes forgoes the fusion. `LIMIT 10`, as in Query A, is never
close to either threshold, so the fusion is unconditional in practice for
any query with a genuinely small `LIMIT`.

### 7.6.5 The `sum_rewriter` question — resolved

The task that produced this chapter specifically flagged an open question:
DiDi's course material [10] names a `sum_rewriter` pass with the mechanic
"`sum(e + const)` → `sum(e) + const × count(e)`, restricted to integer types
since IEEE-754 float addition isn't associative." The research recon file
covering this chapter's source didn't resolve whether this is its own
`OptimizerType` or folded into something else. Direct inspection of
`src/optimizer/` resolves it: **there is no file or `OptimizerType` named
`sum_rewriter` or `SUM_REWRITER`.** The rewrite lives inside
`src/optimizer/aggregate_function_rewriter.cpp`, as one rule
(`SumRewriteRule`) registered inside the broader `AggregateFunctionRewriter`
— which is `OptimizerType::AGGREGATE_FUNCTION_REWRITER`, pass #4 in §7.4's
list, whose own inline comment in `optimizer.cpp` already names both of its
jobs: *"Rewrites `AVG(x)` -> `SUM(x)/COUNT(x)` and `SUM(x+C)` -> `SUM(x) +
C*COUNT(x)`"*.

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:173
//! SUM(x + C) -> SUM(x) + C * COUNT(x)
class SumRewriteRule : public AggregateRewriteRule {
public:
	explicit SumRewriteRule(Optimizer &optimizer) : AggregateRewriteRule(optimizer) {
		// Match SUM(x + C) or SUM(C + x) on integer types
		auto op = make_uniq<AggregateExpressionMatcher>();
		op->function = make_uniq<SpecificFunctionMatcher>("sum");
		// ...
```

The class comment tracks DiDi's description exactly — restricted to integer
types, precisely because DiDi's stated reason (float addition isn't
associative under IEEE-754, so `sum(x) + n*c` isn't guaranteed bit-identical
to `sum(x + c)` for floating-point `x`) is a correctness constraint DuckDB's
implementation also respects. `SumRewriteRule` is registered alongside the
`AVG` rewrite in the same file
(`rules.push_back(make_uniq<SumRewriteRule>(optimizer));` at
`aggregate_function_rewriter.cpp:506`), both living inside a small internal
rule list this single `OptimizerType` iterates — architecturally similar to
how the 37 expression rules in §7.5 all live inside one `OptimizerType`
(`EXPRESSION_REWRITER`), just for aggregate-shaped patterns instead of
scalar-expression ones. So: DiDi's "sum_rewriter" is real, its mechanic is
accurately described, and its actual home in the source is a rule class
inside pass #4, not a separate numbered pass of its own.

While in this file, it's worth also resolving what DiDi's slides [10] describe
as **`join_filter_pushdown`**'s *dynamic* nature concretely, since it's easy
to misread as a purely logical-plan rewrite: `JoinFilterPushdownOptimizer`
(pass #42, `src/optimizer/join_filter_pushdown_optimizer.cpp`) doesn't
compute a static filter range at plan time at all — min/max isn't known
until the hash table is actually built at execution time. What this pass
does at optimization time is set up the *plumbing*: identifying which join
keys are eligible (`IsJoinFilterPushdownIntegralType`/
`IsJoinFilterPushdownIntegralCast` restrict this to integral types up to
64 bits, plus a specific `VARIANT`-to-integral cast case), and recording,
per `JoinFilterExpressionPath`, how to reconstruct the equivalent expression
on the probe side once a runtime range becomes available. The actual
min/max collection and probe-side pre-filtering happen inside the physical
hash-join operator during execution — which is exactly why, as DiDi notes [10]
and this chapter's requirements restate, this pass's effect is visible in
`EXPLAIN ANALYZE`'s runtime output but invisible in a plain, pre-execution
`EXPLAIN`.

### 7.6.6 `IN_CLAUSE` — the plan-level rewrite (mark join, not the expression rule)

`InClauseRewriter` (`src/optimizer/in_clause_rewriter.cpp`) is the plan-level
pass, distinct from pass #1's `InClauseSimplificationRule` (§7.5). Its
trigger condition, `HasRewritableInClause`, requires the `IN`/`NOT IN`
list to have at least `IN_CLAUSE_REWRITE_THRESHOLD` elements *and* every
element past the first to be foldable (a compile-time constant, not a
subquery or a correlated expression):

```cpp
// src/optimizer/in_clause_rewriter.cpp:16
bool InClauseRewriter::HasRewritableInClause(const Expression &expr) {
	if (expr.GetExpressionClass() == ExpressionClass::BOUND_OPERATOR &&
	    (expr.GetExpressionType() == ExpressionType::COMPARE_IN ||
	     expr.GetExpressionType() == ExpressionType::COMPARE_NOT_IN)) {
		auto &op = expr.Cast<BoundOperatorExpression>();
		if (op.GetChildren().size() >= InClauseRewriter::IN_CLAUSE_REWRITE_THRESHOLD) {
			bool all_foldable = true;
			for (idx_t child_idx = 1; child_idx < op.GetChildren().size(); child_idx++) {
				if (!op.GetChildren()[child_idx]->IsFoldable()) {
					all_foldable = false;
					break;
				}
			}
			if (all_foldable) {
				return true;
			}
		}
	}
	// ...
```

When the threshold is met, the literal values are materialized once as a
`LogicalColumnDataGet` — an in-memory, columnar chunk of constant values,
not a scan of any table — and the original `IN` predicate becomes a `MARK`
join between the query's rows and that constant chunk on equality. This is
exactly DiDi's stated mechanic [10]: a naive `IN (v1, v2, ..., vn)` evaluated
per-row is an O(n·m) linear scan of the list for every one of `m` input
rows; turned into a mark join against an n-row `LogicalColumnDataGet`, it
becomes an O(n+m) hash-build-then-probe, with the usual hash-join
asymptotics replacing linear list-scanning. A short list (below the
threshold) isn't worth the overhead of building a hash table for and is
left as a direct, per-row comparison chain instead — which is also, not
coincidentally, cheap enough for `REORDER_FILTER` (§7.6.1) to reason about
using its simple static cost model.

The pass refuses some places where the mark join would change behavior. It
overrides `VisitExpression` (`in_clause_rewriter.cpp:63-86`):

- inside a `TRY(...)` expression, an `IN` stays an expression. The mark join
  would raise its errors outside the `TRY`;
- inside `AND`/`OR`, `CASE` and `COALESCE`, only the first child is visited.
  The other children run only for some rows. A mark join would evaluate
  them for every row, and could raise an error the original query never
  hits.

Also, a short `IN` list with a volatile left side (for example
`random() IN (...)`) is not expanded into `x = 1 OR x = 2 ...`
(`in_clause_rewriter.cpp:118-120`). That expansion would call the volatile
expression once per list value instead of once. Part 08 owns the details.

### 7.6.7 CSE and common-subplan reuse — three different scopes of "don't compute it twice"

Three passes in §7.4's list are all, in a loose sense, about not
recomputing the same thing twice, but they operate at three genuinely
different scopes, and it's worth being precise about which is which:

- **`COMMON_SUBEXPRESSIONS`** (#25, `src/optimizer/cse_optimizer.cpp`) —
  scope: *within one operator's expression list*. `SELECT a, a+1, a+1 FROM t`
  has `a+1` appearing twice in one `LogicalProjection`'s expression list;
  `CommonSubExpressionOptimizer` recognizes the duplicate, computes it once,
  and has the second occurrence reference the first's already-computed
  result rather than re-evaluating `a+1` a second time per row. This is the
  test suite's own canonical example
  (`test/sql/optimizer/expression/test_cse.test`).
- **`COMMON_AGGREGATE`** (#37, `src/optimizer/common_aggregate_optimizer.cpp`)
  — scope: *aggregate functions within one `LogicalAggregate`*. If two
  different output expressions both need `SUM(x)`, it's computed once and
  shared, the same idea as CSE but specialized to the aggregate-state
  machinery rather than plain scalar expressions.
- **`COMMON_SUBPLAN`** (#28, `src/optimizer/common_subplan_optimizer.cpp`) —
  scope: *entire plan subtrees*, potentially spanning a scan, a filter, and
  a join, not just one expression or one aggregate. It detects two
  syntactically identical subtrees appearing anywhere in the plan (the
  canonical case: the same table scanned and filtered identically on both
  sides of a self-join, or via two otherwise-unrelated references to the
  same derived computation), materializes that subtree once as a synthetic
  CTE (reusing the same machinery `CTE_INLINING`'s counterpart,
  materialization, relies on — the file even includes
  `optimizer/cte_inlining.hpp`), and replaces both original occurrences with
  a `CTE_SCAN` onto the single materialized copy. It explicitly skips any
  subplan containing something volatile, since two "identical" subtrees
  that each call `random()` are not actually interchangeable — deduplicating
  them would silently change the query's answer.

## 7.7 Query A, traced through the full pipeline

Recall Query A from the front matter, and the unoptimized tree Chapter 4
built for it:

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

```
LogicalLimit (10)
  └─ LogicalOrder (total DESC)
       └─ LogicalAggregate (GROUP BY o.custkey; SUM(l.extendedprice))
            └─ LogicalFilter (l.shipdate < DATE '1998-01-01')
                 └─ LogicalComparisonJoin (o.orderkey = l.orderkey)
                      ├─ LogicalGet (orders)
                      └─ LogicalGet (lineitem)
```

Chapter 1's §1.3 previewed which passes matter here, in shallow form; this
section walks all 45 `RunOptimizer` call sites from §7.4 against this
specific tree, stating for each one whether it does anything and, if not,
exactly why not — which is itself useful, since "why didn't a pass fire"
is at least as informative as "what did a pass do" for building real
intuition about this machinery.

**Passes that are no-ops for Query A, and why:**

- **`EXPRESSION_REWRITER`** (#1) runs, but there is nothing foldable to fold
  — `DATE '1998-01-01'` is already bound directly as a date constant, not a
  cast-of-a-string-literal needing `ConstantFoldingRule`'s help, and the
  single-conjunct `WHERE` isn't a shape any of the other 36 rules matches.
- **`CTE_INLINING`** (#2, #12) — no `WITH` clause anywhere in Query A.
- **`SELF_JOIN_AGGREGATE`** (#3) — Query A joins `orders` to
  `lineitem`, two different tables, not a table to itself.
- **`AGGREGATE_FUNCTION_REWRITER`** (#4) — `SUM(l.extendedprice)` is already
  a plain, non-composite `SUM` over a column, not `AVG` and not `SUM(x + C)`
  for a constant `C` — `SumRewriteRule` (§7.6.5) has nothing to match.
- **`CTE_FILTER_PUSHER`**, **`REGEX_RANGE`** — no CTEs, no regex.
- **`IN_CLAUSE`** (#9) — no `IN` list at all, let alone one past the
  rewrite threshold.
- **`DELIMINATOR`**, **`UNNEST_REWRITER`** (#10, #22) — no correlated
  subquery, so `FlattenDependentJoins` (Chapter 6) never ran and there's no
  `DelimJoin`/`DelimGet` pair for either pass to clean up.
- **`GROUPING_SETS`** — a plain `GROUP BY o.custkey`, not `ROLLUP`/`CUBE`/
  explicit grouping sets.
- **`EMPTY_RESULT_PULLUP`** — nothing gets proven empty for this query (see
  `STATISTICS_PROPAGATION` below).
- **`WINDOW_SELF_JOIN`**, **`TOP_N_WINDOW_ELIMINATION`**,
  **`ROW_NUMBER_REWRITER`** — no window functions anywhere in Query A.
- **`PROJECTION_PULLUP`** — there's no `LogicalProjection` sitting directly
  under the join to hoist; the `SELECT` list's projection sits at the very
  top of the tree, above the aggregate and the top-N, untouched by a pass
  that only concerns itself with projections adjacent to joins.
- **`OUTER_JOIN_SIMPLIFICATION`** — the join is already `INNER`; there's no
  outer-join-ness to simplify away.
- **`JOIN_ORDER`** — DPhyp still runs and builds its hypergraph, but with
  exactly one join edge between two relations there is only one possible
  join order; the search space is a single point, and the pass returns the
  same tree shape it was handed. **Chapter 8** covers what changes once a
  third table joins in.
- **`AGGREGATE_REUSE`**, **`PARTIAL_AGGREGATE_PUSHDOWN`**,
  **`JOIN_ELIMINATION`** — no `SEMI`/`ANTI` join to reuse an aggregate
  from; `orders` isn't a dimension table being pre-aggregated into ahead of
  a join in the sense `PARTIAL_AGGREGATE_PUSHDOWN` targets (there's exactly
  one join, and the aggregate sits above it, not restructured around it);
  and every column the join produces is actually needed downstream, so
  there's no join to eliminate.
- **`REDUNDANT_ORDER_KEYS`** (#21) — `ORDER BY total DESC` has only one
  key. The pass needs at least two keys (`order.orders.size() < 2` returns
  early), so there is nothing to drop.
- **`DUPLICATE_GROUPS`** — one `GROUP BY` key, `o.custkey`; nothing
  duplicated.
- **`COMMON_SUBEXPRESSIONS`**, **`COMMON_AGGREGATE`**, **`COMMON_SUBPLAN`**
  — no repeated expression, no repeated aggregate, no two identical
  subtrees anywhere in this small a plan.
- **`LIMIT_PUSHDOWN`** — its trigger is a `LIMIT` sitting directly above a
  plain `LogicalProjection`; here, `LogicalLimit`'s immediate child is
  `LogicalOrder` (soon to become part of `LogicalTopN`), not a projection,
  so the pass has nothing to push through.
- **`ROW_GROUP_PRUNER`** — this pass needs the `ORDER BY` key to trace back
  to an actual stored column on a `LogicalGet`. Here the sort key is
  `total`, the *output of an aggregate* (`SUM(l.extendedprice)`), not a
  column any table's zonemap describes directly — there's no stored min/max
  for "sum of extendedprice per customer" to sort row groups by, so this
  pass can't apply.
- **`SAMPLING_PUSHDOWN`** — no `TABLESAMPLE` clause.
- **`LATE_MATERIALIZATION`** — every column either table contributes
  (`o.custkey`, `o.orderkey`, `l.orderkey`, `l.extendedprice`,
  `l.shipdate`) is consumed immediately, either by the join condition, the
  filter, or the aggregate — there's no wide payload column sitting idle,
  waiting only for the final `SELECT`, for this pass to defer fetching.
- **`REORDER_FILTER`** (#40) — the `WHERE` clause is a single comparison,
  not a multi-conjunct `AND`/`OR`; `ReorderExpressions` only does anything
  when `expressions.size() > 1`.
- **`TYPE_PUSHDOWN`**, **`SCALAR_FN_PUSHDOWN`** — the `SELECT` list has no
  explicit cast and no scalar function of a column for a reader to absorb
  (`SUM` is an aggregate, handled by the aggregate operator, not something
  `TYPE_PUSHDOWN`/`SCALAR_FN_PUSHDOWN` target).

**Passes that actually change something:**

- **`FILTER_PUSHDOWN`** (#6, with `FILTER_PULLUP` at #5 running first but
  finding nothing to pull up here, since there's no *pre-existing* filter
  on the `orders` side for pullup to project across the join onto
  `lineitem`). `l.shipdate < DATE '1998-01-01'` references only `lineitem`
  columns, so the per-operator join handler lets it travel straight past
  the join and become part of `lineitem`'s own `LogicalGet`, where it
  becomes a table filter the storage layer's zonemaps can use to skip
  entire row groups of `lineitem` that are provably all `>= DATE
  '1998-01-01'` (Chapter 15).
- **`UNUSED_COLUMNS`** / **`COLUMN_LIFETIME`** (#23/#26, and their
  second occurrences at #38/#39) — neither `orders` nor `lineitem` is
  scanned with every one of its columns; only `custkey` and `orderkey` from
  `orders`, and `orderkey`, `extendedprice`, and `shipdate` from
  `lineitem`, survive into the scan's projection, because nothing else the
  two tables might contain is referenced anywhere in the join condition,
  filter, aggregate, or final `SELECT` list.
- **`BUILD_SIDE_PROBE_SIDE`** (#27) — decides which of `orders` and
  `lineitem` becomes the hash join's build side; the fuller mechanics of
  that decision (and why the filtered, narrowed `lineitem` scan is not
  automatically the "obvious" choice either way) belong to Chapter 8/9's
  physical-join-operator discussion, but the column set this decision
  reasons over is exactly the narrowed set `UNUSED_COLUMNS`/`COLUMN_LIFETIME`
  just computed.
- **`STATISTICS_PROPAGATION`** (#34) — propagates `lineitem.shipdate`'s
  known min/max/histogram through the `l.shipdate < DATE '1998-01-01'`
  filter to estimate its selectivity, and propagates the resulting
  estimated row counts on through the join and the aggregate. Nothing in
  Query A's filter is provably always-true or always-false against real
  table statistics (unlike the constructed always-false example in
  §7.6.2), so no `LogicalEmptyResult` collapse happens here — this pass's
  contribution to Query A is entirely about cardinality *estimates* feeding
  cost decisions elsewhere (physical operator choice in Chapter 9,
  memory-budgeting decisions in Chapter 14), not about further rewriting
  the tree's shape.
- **`TOP_N`** (#32) — `ORDER BY total DESC LIMIT 10` satisfies every
  condition in `TopN::CanOptimize` (§7.6.4): the limit (`10`) is a constant,
  there's no offset, and `10` is nowhere near either the 0.7%-of-cardinality
  or the >5000-rows-absolute threshold that would decline the fusion — so
  `LogicalOrder` + `LogicalLimit` fuse unconditionally into a single
  `LogicalTopN(ORDER BY total DESC LIMIT 10)`.
- **`JOIN_FILTER_PUSHDOWN`** (#42) — `o.orderkey`/`l.orderkey` is an
  integral join key (TPC-H's `orderkey` columns are integers), which is
  exactly the type `IsJoinFilterPushdownIntegralType` requires — a strong
  candidate for the dynamic runtime filter §7.6.5 describes: as the hash
  join builds its table from (whichever side ends up the build side, per
  `BUILD_SIDE_PROBE_SIDE` above), it can record that side's observed
  `orderkey` min/max and use the range to pre-filter the probe side's rows
  before they even reach the hash-table lookup. Confirming whether this
  fires for a *specific* run of Query A, and in exactly what physical form,
  needs `EXPLAIN ANALYZE` against a real instance — the static plan alone,
  as this chapter has stressed, won't show it.

After all of the above, the tree looks like this — the same "after
optimization" shape Chapter 1 previewed, now with the specific column
pruning made explicit:

```
LogicalTopN (ORDER BY total DESC LIMIT 10)
  └─ LogicalAggregate (GROUP BY o.custkey; SUM(l.extendedprice))
       └─ LogicalComparisonJoin (o.orderkey = l.orderkey)
            ├─ LogicalGet (orders — columns: custkey, orderkey)
            └─ LogicalGet (lineitem — columns: orderkey, extendedprice,
                            shipdate; filter pushed into scan:
                            shipdate < DATE '1998-01-01')
```

Every arrow in this diagram, and the narrowed column lists on both
`LogicalGet`s, are the accumulated effect of exactly two passes doing real
work on this query's *shape* (`FILTER_PUSHDOWN` and `TOP_N`) plus one pass
doing real work purely on the *columns each operator carries*
(`UNUSED_COLUMNS`/`COLUMN_LIFETIME`, together) — everything else in the
45-pass sequence either had no matching pattern in this particular query,
or did real, invisible-to-`EXPLAIN` work feeding a decision some later stage
(the physical plan generator in Chapter 9, the execution engine in Chapter
14) will actually act on. That asymmetry — a handful of visible structural
rewrites, a couple of invisible-until-runtime ones, and roughly three dozen
passes that simply don't apply to this particular small query — is itself
the most honest one-sentence summary of what "a fixed, forty-pass pipeline"
feels like in practice: most queries only exercise a small slice of it, and
the fixed ordering exists precisely so that whichever slice a given query
does exercise, it exercises in a predictable, cheaply-measured order.

## 7.8 Summary: what crosses the line into Chapter 8 and beyond

This chapter covered the architecture (a fixed, one-shot, individually
timed and toggleable sequence of specialist passes — not a fixpoint rule
engine, and, DPhyp aside, not a cost-based search over alternative plan
shapes), reproduced the full 45-call-site pipeline in exact execution
order with real per-pass mechanics, gave the complete 37-rule
expression-rewriter list with concrete before/after examples, resolved the
`sum_rewriter` naming question against the actual source
(`SumRewriteRule` inside `AGGREGATE_FUNCTION_REWRITER`), and traced Query A
through every one of those 45 call sites to show precisely which two
structural passes and which column-pruning pass actually reshape this
book's own running example.

Two of the passes named in §7.4's list got only a one-line pointer here
because they are large enough to need — and already have — their own
chapters: **`JOIN_ORDER`**, whose DPhyp dynamic-programming search over
join orders is the one place in this entire pipeline where DuckDB commits
to genuine cost-based search rather than a one-shot syntactic rewrite, is
**Chapter 8**'s complete subject, including the query hypergraph, the
relation-set bitmask bookkeeping, the DP-table cost model, and the
escape-hatch heuristic for when the search space gets too large.
**`DELIMINATOR`** and **`UNNEST_REWRITER`**, the optimizer's post-decorrelation
cleanup of the join tree `FlattenDependentJoins` (Chapter 6) leaves behind,
belong to **Chapter 6**'s treatment of subquery decorrelation as a whole,
alongside the planner-level machinery that does the actual, academically
well-known unnesting work before the optimizer's first pass even runs.

What the physical plan generator (Chapter 9) inherits from this chapter is
the tree shown at the end of §7.7: the same node vocabulary the binder
originally produced (`LogicalGet`, `LogicalComparisonJoin`,
`LogicalAggregate`, and so on — the optimizer rewrites the tree, it doesn't
introduce a new intermediate representation), but reshaped by up to forty-five
narrow, specialist decisions into something a lot closer to a plan you'd
actually want to run: filters evaluated at the scan, unread columns never
even projected out of storage, and a bounded top-k structure standing in
for a full sort. Chapter 9 picks up from exactly this point, and its first
job is deciding, operator by operator, which concrete `PhysicalOperator`
each of these logical nodes becomes.
