# Chapter 7, Part 20 — STATISTICS_PROPAGATION

## 7.20.1 Where this pass actually sits, and the distinction it is easy to blur

`00-overview.md` §7.6.2 already corrected one popular misconception about this
pass — that it feeds `JOIN_ORDER`'s cost model — with a source-level citation:
`STATISTICS_PROPAGATION` is pass **#34** in the fixed pipeline
(`optimizer.cpp:476`), running *after* `JOIN_ORDER` (#17) and
`BUILD_SIDE_PROBE_SIDE` (#27), not before either. Chapter 8 §8.6 established
the pass that actually feeds DPhyp's cost model: a separate, purpose-built
estimator in `src/optimizer/join_order/cardinality_estimator.cpp`, built on
top of `RelationStatisticsHelper` (`src/optimizer/relation_statistics/`),
which runs *inside* the `JOIN_ORDER` pass itself and is gone by the time
`STATISTICS_PROPAGATION` starts.

This file is about the *other* one — the one that actually owns the name
`StatisticsPropagator` in the source tree (`src/optimizer/statistics_propagator.cpp`,
plus the twenty-two per-operator/per-expression handler files under
`src/optimizer/statistics/operator/` and `src/optimizer/statistics/expression/`,
plus `src/optimizer/filter_statistics.cpp`). It is worth restating the
division of labor bluntly, because this file's own test corpus (and two of
the files this chapter was specifically asked to cross-reference,
`test/optimizer/estimated_cardinalities_are_in_logical_plan.test` and
`test/optimizer/estimated_cardinalities_through_struct_extract.test`) will
otherwise tempt you to mix the two up:

| | `RelationStatisticsHelper` (Ch. 8) | `StatisticsPropagator` (this file) |
|---|---|---|
| Runs during | `JOIN_ORDER`, pass #17 | `STATISTICS_PROPAGATION`, pass #34 |
| Answers | "How many rows will this relation/join produce?" | "What does this **column** provably contain — range, nullability — and can that prove a predicate's truth value?" |
| Tracks per column | `DistinctCount` (HLL- or min/max-derived, feeding join-size formulas) | min/max (`NumericStatsData`/`StringStatsData`), a coarse can-have-null/can-have-no-null pair |
| Consumed by | DPhyp's dynamic-programming search, `BuildProbeSideOptimizer` | `ReplaceWithEmptyResult`, `ConstantOrNull` folding, the cost-based second pass of `MultiStageAggregateRewriter`, `PROJECTION_PLACEMENT` |
| Lives once query starts executing? | No — thrown away after `JOIN_ORDER` | No — thrown away after `STATISTICS_PROPAGATION`/`PROJECTION_PLACEMENT` |
| Source | `src/optimizer/relation_statistics/*.cpp` | `src/optimizer/statistics_propagator.cpp`, `src/optimizer/statistics/{operator,expression}/*.cpp`, `src/optimizer/filter_statistics.cpp` |

Both are, in the loosest sense, "statistics passes." They are different code,
answering different questions, at different points in the fixed pipeline,
for different downstream consumers, and they do not share so much as a
`BaseStatistics` object between them — `RelationStatisticsHelper` builds its
own `RelationStats`/`RelationColumnStats`/`DistinctCount` structures from
scratch (`relation_statistics.cpp:5-10`), and by the time
`STATISTICS_PROPAGATION` runs, none of that survives. Keep this table in
mind through §7.20.16, where two of this file's namesake tests turn out to
belong to the *other* column of it.

## 7.20.2 The general algorithm: a bottom-up switch over `LogicalOperatorType`

`StatisticsPropagator::PropagateStatistics` (`statistics_propagator.cpp:72-146`)
is the dispatcher. It is a plain `switch` over `node.type` that casts to the
concrete operator subclass and calls an overload; operator types it doesn't
recognize fall through to `PropagateChildren`, which just recurses into every
child and returns no statistics of its own (`statistics_propagator.cpp:45-51`):

```cpp
// src/optimizer/statistics_propagator.cpp:72-146
unique_ptr<NodeStatistics> StatisticsPropagator::PropagateStatistics(LogicalOperator &node,
                                                                     unique_ptr<LogicalOperator> &node_ptr) {
	propagation_path.emplace_back(node_ptr);
	unique_ptr<NodeStatistics> result;
	switch (node.type) {
	case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
		result = PropagateStatistics(node.Cast<LogicalAggregate>(), node_ptr);
		break;
	// ... LOGICAL_FILTER, LOGICAL_GET, LOGICAL_PROJECTION, every join type,
	// LOGICAL_LIMIT, set operations, LOGICAL_ORDER_BY, LOGICAL_WINDOW,
	// LOGICAL_CTE_REF, LOGICAL_MATERIALIZED_CTE, LOGICAL_SECURE_VIEW,
	// LOGICAL_COPY_TO_FILE, LOGICAL_CROSS_PRODUCT, LOGICAL_POSITIONAL_JOIN
	case LogicalOperatorType::LOGICAL_UNNEST:
		result = PropagateChildren(node, node_ptr);
		// unnest emits its input row once per unnested entry
		ResetAdditiveStatistics(node.children[0]->GetColumnBindings());
		break;
	default:
		result = PropagateChildren(node, node_ptr);
	}

	if (mode == StatisticsPropagationMode::FULL &&
	    !optimizer.OptimizerDisabled(OptimizerType::COMPRESSED_MATERIALIZATION)) {
		// compress data based on statistics for materializing operators
		CompressedMaterialization compressed_materialization(optimizer, *root, propagation_path, statistics_map);
		compressed_materialization.Compress(node_ptr);
	}

	propagation_path.pop_back();
	return result;
}
```

Two things in this excerpt need a word. `propagation_path` is a stack of the
operators on the path from the root down to the current node.
`CompressedMaterialization` uses it to rebind only the ancestors of an
operator it changed, instead of sweeping the whole plan. The `LOGICAL_UNNEST`
case resets "additive" statistics; the end of §7.20.3 explains what those are.

The constructor takes `unique_ptr<LogicalOperator> &root`, not a plain
reference. The pass can replace the root operator itself (for example, a
root `LIMIT` that it removes). Because it holds the owning pointer, the
propagator then sees the new root, not a freed one
(`test/optimizer/statistics/statistics_root_replaced.test`).

Every overload follows the same shape: recurse into the child(ren) first
(`node_stats = PropagateStatistics(child)`), *then* do this operator's own
work using whatever statistics the recursion populated into a single shared
map, `column_binding_map_t<unique_ptr<BaseStatistics>> statistics_map`
(declared on the `StatisticsPropagator` instance, one map for the whole
tree walk — `statistics_propagator.hpp:169`). That is the entire mechanism of
"bottom-up propagation": there is no separate tree pass to collect base
statistics and a second pass to push them through filters — the single
recursive walk *is* both, because a parent operator's handler runs its own
logic only after its child's handler has already populated
`statistics_map` with everything the child knows. A `BoundColumnRefExpression`
looks itself up in this map and does nothing else
(`propagate_columnref.cpp:6-13`); a `LogicalProjection` re-derives statistics
for each of *its own* output expressions and re-inserts them into the map
under *its own* table index (`propagate_projection.cpp:6-23`), so a column
renamed or recomputed by a projection gets a fresh map entry rather than
inheriting one under a stale binding.

The four operator families the task asked about, concretely:

- **`LogicalGet`** (`propagate_get.cpp:167-243`) is the tree's *source* of
  statistics — it doesn't derive anything from a child (it has none); it
  pulls per-column `BaseStatistics` straight out of the table function's own
  `statistics`/`statistics_extended` callback (for a real table, this is
  ultimately backed by the storage layer's per-segment min/max — Chapter 17
  §17.5 covers where those numbers actually come from) and inserts one entry
  per scanned column into `statistics_map`, keyed by `(get.table_index,
  ProjectionIndex(i))`.
- **`LogicalFilter`** (`propagate_filter.cpp:317-355`) *narrows*: it
  propagates through the child first, then calls `HandleFilter` on each of
  its own conjuncts, which (a) re-propagates statistics through the
  condition expression, (b) classifies the result — always true, always
  false, true-or-null, false-or-null, or "no pruning possible" — and (c), in
  the last case, calls `UpdateFilterStatistics` to tighten the *column's*
  statistics in `statistics_map` based on what surviving this filter now
  implies (`x > 100` sets the column's known minimum to 100 going forward).
- **`LogicalJoin`** (`propagate_join.cpp:301-373`; the per-condition work is in
  the `LogicalComparisonJoin` overload, `:122-254`) *combines*: cardinality
  multiplies (inner/semi) or takes special handling per join type, and for
  each equality/inequality condition, `PropagateComparison` checks whether
  the two sides' ranges can even overlap at all — with a genuinely
  interesting extra step covered in §7.20.7 below, where a *narrowed* range
  discovered on one side of a join condition is turned into an actual new
  `LogicalFilter` on the *other* side.
- **`LogicalProjection`**/general expressions (`propagate_projection.cpp`,
  the whole `src/optimizer/statistics/expression/` directory) propagate
  *through* functions when the function's behavior is either fully constant,
  fully foldable, or annotated as monotonic in its arguments — `x + 1`'s
  output range is the input range's min and max, each shifted by 1, and this
  is not a special case for `+` specifically; it's the general
  `PropagateMonotoneBounds` mechanism described in §7.20.6.

## 7.20.3 What statistics this pass actually tracks — read from source, not assumed

`BaseStatistics` (`src/include/duckdb/storage/statistics/base_statistics.hpp:64-176`)
is the type every one of the files above traffics in, and its private data
members settle every question about what is and isn't tracked:

```cpp
// src/include/duckdb/storage/statistics/base_statistics.hpp:163-176
private:
	LogicalType type = LogicalType::INVALID;
	//! Whether or not the segment can contain NULL values
	bool has_null;
	//! Whether or not the segment can contain values that are not null
	bool has_no_null;
	//! estimate that one may have even if distinct_stats==nullptr
	idx_t distinct_count;
	//! Numeric and String stats
	union { NumericStatsData numeric_data; StringStatsData string_data; ... } stats_union;
```

Four things worth being precise about, since a shallow summary of
this pass could easily get any of them wrong:

1. **Min/max**, for numeric types, is `NumericStatsData` — `has_min`/`has_max`
   booleans plus a `NumericValueUnion min`/`max` (`numeric_stats.hpp:27-36`).
   For strings it's `StringStatsData` (`string_stats.hpp:33-58`) — but the
   min/max stored there are *truncated* to `CURRENT_MAX_STRING_MINMAX_SIZE`
   (12 bytes) with an explicit `StringStatsType` tag (`NO_STATS`/
   `EMPTY_STATS`/`EXACT_STATS`/`TRUNCATED_STATS`) recording whether the
   stored bound is exact or merely a safe truncated prefix. When it's
   truncated, comparison logic (`GetStringStatsMax` in
   `propagate_comparison.cpp:15-25`) doesn't just use the truncated bytes
   as-is for an upper bound — it calls `StringUtil::FindNextPrefix` to widen
   a truncated max to the next byte sequence that's still guaranteed to be
   `>=` every value with that prefix, so a truncated bound never produces a
   false "always true"/"always false" conclusion.
2. **Nullability is a two-bit flag, not a fraction.** `has_null` and
   `has_no_null` are independent booleans — DuckDB tracks *can this column
   contain a NULL* and *can it contain a non-NULL value*, as two separate
   claims, not a null-count or null-percentage. All four combinations are
   meaningful: `(false, true)` is "definitely never null," `(true, false)`
   is "definitely always null" (an all-`NULL` column — `statistics_is_null.test`
   exercises this exact case, see §7.20.14), `(true, true)` is the ordinary
   "could be either" case, and `(false, false)` describes an empty
   relation. There is no `NullFraction()`/`NullPercentage()` API anywhere in
   this type; anyone reasoning about a "null selectivity" number for this
   pass's purposes is reasoning about something DuckDB's optimizer does not
   compute here.
3. **String length is tracked, separately from the min/max string values
   themselves** — `StringStatsData` also carries `has_min_string_length`/
   `min_string_length`, `has_max_string_length`/`max_string_length`, and
   `has_total_string_length`/`total_string_length` (byte counts, not
   codepoint counts — `string_stats.hpp:44-53`), plus `has_unicode`. This is
   exactly the "string-specific statistic" the task asked about, and
   `statistics_string_length.test` (§7.20.14) is built entirely around
   exercising it, byte-length semantics and all.
4. **Distinct count is tracked on `BaseStatistics` (`distinct_count`,
   `base_statistics.cpp:16,55,65,329`, populated from the storage layer's
   `DistinctStatistics` HyperLogLog sketch — `distinct_statistics.hpp:18-53`),
   but `StatisticsPropagator` itself never narrows it.** Grep the entire
   `src/optimizer/statistics/` directory for `SetDistinctCount` and you find
   nothing — every call site is in `src/storage/table/table_statistics.cpp`
   (populating the field when statistics are first read off a segment) or
   inside `BaseStatistics::FromConstant` (`base_statistics.cpp:646`, setting
   it to exactly 1 for a literal). `Copy()`/`Merge()` both carry the field
   through mechanically (`CopyBase`, `base_statistics.cpp:326-329`), so a
   column's approximate distinct count survives a projection or a `COALESCE`
   unchanged, but nothing in this pass ever *recomputes* it after a filter
   narrows a range or a join combines two relations — that recomputation is
   exactly what `RelationStatisticsHelper::EstimateDistinctCardinality`
   (`relation_statistics_helper.cpp:93-104`) does, in the *other* system,
   for the *other* consumer (Ch. 8). If you were expecting this pass to
   maintain a live, continuously-refined distinct-count estimate as it walks
   the tree, the source says otherwise: distinct count rides along as
   inert baggage here, and the pass's actual analytical currency is min/max
   plus the two nullability bits.

Cardinality — "how many rows will this subtree produce" — is tracked
separately again, at the *node* level rather than per-column, as
`NodeStatistics` (`node_statistics.hpp:16-34`): `has_estimated_cardinality`/
`estimated_cardinality` plus `has_max_cardinality`/`max_cardinality` (an
upper bound distinct from the point estimate). `PropagateStatistics`'s
per-operator overloads return one of these for every operator, and the
combinators are worth naming since they show up constantly: `AddCardinalities`
(set operations, `src/optimizer/statistics/operator/propagate_set_operation.cpp:6-28`),
`MultiplyCardinalities` (joins and cross products,
`propagate_join.cpp:276-299`), `MaxCardinalities` (positional joins,
`propagate_join.cpp:375-383`) — every one of them is careful to bail to
`nullptr` rather than silently under- or over-count once either operand
lacks a cardinality estimate or a `Hugeint`-checked multiply/add would
overflow `int64_t` (`propagate_join.cpp:279-297`).

### Additive statistics are reset when rows are duplicated

Most statistics stay true when a row is copied. If `x` is in `[1, 9]`, it is
still in `[1, 9]` after a join repeats some rows. But one statistic is a
*sum over rows*: `total_string_length`, the total byte length of all strings
in a column (point 3 above). The code calls such a statistic **additive**.
If a join emits the one dimension row 1,000 times, the real total is 1,000
times larger, and the stored total is wrong.

`BaseStatistics::ResetAdditiveStatistics`
(`src/storage/statistics/base_statistics.cpp:184-203`) clears
`has_total_string_length`, and recurses into the children of `LIST`, `ARRAY`,
`STRUCT` and `VARIANT` statistics. The propagator calls it, through a small
helper, for every column that passes through an operator that can emit an
input row more than once:

```cpp
// src/optimizer/statistics_propagator.cpp:53-62
void StatisticsPropagator::ResetAdditiveStatistics(const vector<ColumnBinding> &bindings) {
	// additive statistics (e.g. total_string_length) describe a fixed row population - an operator that can emit an
	// input row more than once invalidates them for all pass-through columns
	for (auto &binding : bindings) {
		auto entry = statistics_map.find(binding);
		if (entry != statistics_map.end() && entry->second) {
			entry->second->ResetAdditiveStatistics();
		}
	}
}
```

The callers are: a cross product (both sides,
`propagate_cross_product.cpp:11-13`); `INNER`, `LEFT`, `RIGHT` and `OUTER`
joins (both sides) and `SINGLE` joins (the right side)
(`propagate_join.cpp:328-338`); `UNNEST` (`statistics_propagator.cpp:128-132`);
and the group columns of an aggregate with more than one grouping set, where a
key appears once per set (`propagate_aggregate.cpp:486-494`).
`test/optimizer/statistics/statistics_additive_duplication.test` checks each
case with `SET debug_verify_stats=true`, which compares the planned statistics
with the real data at run time. For example:

```sql
-- test/optimizer/statistics/statistics_additive_duplication.test
CREATE TABLE dim AS SELECT 1::BIGINT AS k, 'Package was damaged' AS d;
CREATE TABLE fact AS SELECT 1::BIGINT AS k, range AS amt FROM range(1000);

SELECT max(len(s.d)) FROM (SELECT struct_pack(amt := amt, d := d) AS s FROM fact, dim WHERE fact.k = dim.k);
----
19
```

`struct_pack` copies the child statistics into its result. Without the reset,
the verification would find a total string length of 19 bytes for a column
that really holds 1,000 copies of the string.

## 7.20.4 The pass runs *twice*, and the second time isn't in §7.4's numbered list

`RunBuiltInOptimizers` invokes `StatisticsPropagator` in `FULL` mode at pass
#34, as documented. But `optimizer.cpp:293-296`, sitting between `FILTER_PUSHDOWN`
(#6) and `CTE_FILTER_PUSHER` (#7), contains a second, easy-to-miss
invocation that never shows up in §7.4's pipeline listing under its own
name, because it's not wrapped in its own `RunOptimizer(OptimizerType::...)`
call at the top level — `FilterStatisticsOptimizer` invokes
`STATISTICS_PROPAGATION`'s timer *internally*, one layer down:

```cpp
// src/optimizer/optimizer.cpp:293-296
if (!CTEContainsDML(*plan)) {
	FilterStatisticsOptimizer filter_statistics(*this);
	filter_statistics.Optimize(plan);
}
```

`FilterStatisticsOptimizer::Optimize` (`filter_statistics.cpp:53-74`) is
narrowly gated: it does nothing unless the plan (a) contains no
`LOGICAL_DELIM_JOIN` (`ContainsDelimJoin`, `filter_statistics.cpp:41-51`;
with the default `delim_join_as_cte = true` the planner has already turned
every delim join into joins over CTEs, so in practice this check only
matters when that setting is off — see Chapter 6 §6.4.1) and (b) contains at least one `LogicalFilter` whose
expression references columns from *more than one table binding* —
`HasMultiBindingFilter`/`HasMultipleBindings` (`filter_statistics.cpp:15-39`)
walk every filter's `BoundColumnRefExpression`s and collect the distinct
`table_index`es they touch. A plain single-table `WHERE i > 5` never
triggers this early pass at all; `WHERE a.x = b.y` does. When it fires, it
runs a `StatisticsPropagator` in `StatisticsPropagationMode::FILTER_SIMPLIFICATION`
— a mode that skips the general min/max propagation and instead just tracks,
per filter, which table bindings the filter referenced *before* propagation
versus *after* (`HandleFilter`, `propagate_filter.cpp:299-315`, calling
`SimplifyFilter` to fold constant `AND`/`OR` children and then comparing
`GetFilterBindings(*condition)` before and after). If simplification changed
which tables a filter touches — for instance, a conjunct referencing table
`b` folded away to a constant, leaving a filter that used to need both `a`
and `b` now needing only `a` — `filter_bindings_changed` is set, and
`FilterStatisticsOptimizer` immediately re-runs `FilterPushdown` so the
now-single-table filter can be pushed all the way to that table's own scan,
something the *first* `FilterPushdown` pass (#6, which already ran once
before this) had no way to do, because at that point the filter genuinely
still spanned two tables. This is precisely the mechanism
`statistics_filter_multicolumn.test` exercises (§7.20.14): it isn't really
testing whole-tree statistics propagation so much as this specific,
early, narrowly-scoped re-entry point that exists *only* to unlock pushdown
for multi-table filters whose cross-table dependency turns out to be provably
vacuous.

## 7.20.5 Filter and comparison propagation: `PropagateComparison`, `UpdateFilterStatistics`, `BETWEEN`

The comparison logic is centralized in
`StatisticsPropagator::PropagateComparison(const BaseStatistics&, const BaseStatistics&, ExpressionType)`
(`propagate_comparison.cpp:93-147`), which special-cases string statistics
(comparing truncation-aware min/max, `propagate_comparison.cpp:96-119`) and
otherwise dispatches on `PhysicalType` for every numeric-ish type (including
`INTERVAL`) into `PropagateValueComparison`
(`propagate_comparison.cpp:36-91`) — a pure function of four `Value` bounds
and a comparison type that returns one of five `FilterPropagateResult`s:
`FILTER_ALWAYS_TRUE`, `FILTER_ALWAYS_FALSE`, `FILTER_TRUE_OR_NULL`,
`FILTER_FALSE_OR_NULL`, or `NO_PRUNING_POSSIBLE`. The `_OR_NULL` variants
exist specifically because ranges alone can prove *value-level* truth
without knowing whether the columns involved can also be `NULL` — that
extra bit is threaded in by the caller as `has_null`, and it's what makes the
difference between `x = 5` being unconditionally provable false (if `x`'s
column has no nulls at all) versus only provable "false, unless the row is
NULL" (`FILTER_FALSE_OR_NULL`, if it does).

`ClassifyFilter` (`propagate_filter.cpp:231-249`) also treats a filter
condition that folded to a constant `NULL` as `FILTER_FALSE_OR_NULL`. A
`NULL` condition keeps no row, just like `false`, so the filter's input can
be replaced by an empty result. `statistics_all_null_comparison.test` shows
it: over a column that is `NULL` in every row,
`WHERE split_part(version, '.', 1) <> '0'` folds to `NULL`, and the scan is
replaced by an empty result.

`UpdateFilterStatistics` (`propagate_filter.cpp` — three overloads,
`:51-160`) is the half of the mechanism that goes the other direction:
*given* that a comparison against a constant survived (i.e., wasn't provably
always-true or -false), what does that let you conclude about the column
itself, going forward? For `X < c`/`X <= c` it clamps the column's *max* down
to `c`; for `X > c`/`X >= c` it clamps the *min* up to `c`; for `X = c` it
sets both min and max to `c` (`propagate_filter.cpp:65-86`). This is the
literal, direct implementation of "a filter narrows statistics based on
what its predicate implies," and it composes: a `BETWEEN` is handled by
`PropagateBetween` (`propagate_between.cpp:9-69`) as two ordinary
comparisons against the same input (lower-bound and upper-bound), each run
through the exact same `PropagateComparison`/classification machinery,
with careful case analysis for every combination of "always true," "always
false," and "true/false-or-null" between the two halves — including the
one DuckDB fuzzer-derived edge case explicitly called out in
`statistics_between.test` (§7.20.14): a `BETWEEN` whose *both* bounds are
provably always-true still degenerates the physical filter away entirely,
even though this particular fold happens during *physical* planning rather
than the logical `STATISTICS_PROPAGATION` pass itself.

There's also a two-sided variant, `UpdateFilterStatistics(BaseStatistics&,
BaseStatistics&, ExpressionType)` (`propagate_filter.cpp:89-160`), used when
a comparison relates *two columns* rather than a column and a constant —
this is the mechanism `statistics_join.test`/`statistics_join_multicolumn.test`
exercise (§7.20.14), and it's worth reading its own inline comment because
it explains the reasoning better than paraphrase would:

```cpp
// src/optimizer/statistics/operator/propagate_filter.cpp:106-125
case ExpressionType::COMPARE_LESSTHAN:
case ExpressionType::COMPARE_LESSTHANOREQUALTO:
	// LEFT < RIGHT OR LEFT <= RIGHT
	// we know that every value of left is smaller (or equal to) every value in right
	// i.e. if we have left = [-50, 250] and right = [-100, 100]
	// we know that left.max is AT MOST equal to right.max
	// because any value in left that is BIGGER than right.max will not pass the filter
	if (NumericStats::Max(lstats) > NumericStats::Max(rstats)) {
		NumericStats::SetMax(lstats, NumericStats::Max(rstats));
	}
	// we also know that right.min is AT MOST equal to left.min
	// because any value in right that is SMALLER than left.min will not pass the filter
	if (NumericStats::Min(rstats) < NumericStats::Min(lstats)) {
		NumericStats::SetMin(rstats, NumericStats::Min(lstats));
	}
	// so in our example, the bounds get updated as follows:
	// left: [-50, 100], right: [-50, 100]
	break;
```

Note carefully what this *doesn't* do: it never touches `lstats.min` or
`rstats.max` in the `<` case — a `LEFT < RIGHT` filter tells you nothing new
about `LEFT`'s minimum or `RIGHT`'s maximum, only that `LEFT`'s effective
maximum can't exceed `RIGHT`'s pre-filter maximum and vice versa for the
minimum. This asymmetry is exactly what makes
`statistics_filter_multicolumn.test`'s range-join cases work out the way
they do (§7.20.14).

## 7.20.6 Function and expression propagation: constants, monotone bounds, `CASE`, `COALESCE`

Three distinct mechanisms cover "propagate through functions," in order of
how a `BoundFunctionExpression` is actually handled
(`PropagateExpression(BoundFunctionExpression&, ...)`, `propagate_function.cpp:238-278`):

1. **A per-function statistics callback**, if the function registered one
   (`func.Function().HasStatisticsCallback()`) — this is an escape hatch for
   functions whose output range genuinely can't be derived generically (a
   cast is exactly this case, covered in §7.20.8; so is `count()`,
   §7.20.9).
2. **`PropagateConstantInputs`** (`propagate_function.cpp:89-119`): if every
   argument reduces to a single known `Value` — either genuinely foldable
   (`child.IsFoldable()`), or statistically forced to one value because its
   statistics prove a single min==max (or an all-`NULL` column, handled via
   `TryInferConstantBounds`, `propagate_function.cpp:60-86`, which
   explicitly excludes `±0.0` for floats since equal min/max doesn't imply
   an identical bit pattern for signed zero, and excludes `INTERVAL`
   entirely) — the function is evaluated once, directly, against those
   constants, and the *exact* result becomes the derived statistics
   (`BaseStatistics::FromConstant`). `Lambda`-taking functions are excluded,
   since a lambda argument is a placeholder, not a value to substitute.
3. **`PropagateMonotoneBounds`** (`propagate_function.cpp:123-224`) is the
   general "`x + 1`" mechanism the task asked about, and it is considerably
   more general than just addition: any scalar function that has registered
   per-argument `ArgProperties` with a `Monotonicity` annotation
   (`Monotonicity::CONSTANT`/increasing/decreasing) is evaluated **at the
   corners of its arguments' known ranges** — for each argument, if its
   annotated monotonicity is increasing, its low corner feeds the "compute
   the output's low bound" call and its high corner feeds the "compute the
   output's high bound" call; a decreasing argument gets its corners
   swapped between those two calls; a `CONSTANT`-annotated argument (an
   argument the function's *value* doesn't depend on the sign of — used for
   things like a fixed base) contributes the same value to both. The two
   evaluations (`TryEvaluateAtConstants`, computing `f` at the "all lo
   corners" and "all hi corners" argument vectors,
   `propagate_function.cpp:14-25`) become the derived `NumericStats::SetMin`/
   `SetMax`. A corner evaluation that produces `NULL` or `NaN`
   (`is_unusable`, `propagate_function.cpp:194-207`) makes the whole
   derivation bail rather than derive a wrong bound — this is exactly why
   `statistics_numeric.test`'s `year(infinity)` and `sqrt`/`ln` of a
   negative-including range both come back as "no derived bounds" rather
   than a bound built from a garbage corner value (§7.20.14). If any
   argument isn't annotated as monotonic at all, the whole function bails
   with no derived statistics — this is a strictly opt-in mechanism, keyed
   off explicit per-function annotations, not a generic interval-arithmetic
   engine that works for arbitrary scalar functions.

These mechanisms guard their inputs in four ways:

1. **Unresolved parameters.** If an argument is a prepared-statement parameter
   whose type was reset to `SQLNULL` (filter pushdown can do this), or an
   argument's type is not yet complete, the function gets no statistics at
   all. A parameter has no value until execution.
2. **Empty statistics.** `TryInferConstantBounds` (`:60-86`) does not read
   min/max from statistics that say "no non-`NULL` value". Such bounds do not
   describe any real value.
3. **Stability.** `PropagateConstantInputs` and `PropagateMonotoneBounds`
   refuse only `VOLATILE` functions, so a function that is
   `CONSISTENT_WITHIN_QUERY` can be folded or bounded too. A foldable
   argument is evaluated with `TryEvaluateScalar`, so an argument that throws
   makes the propagation give up instead of failing the whole query at
   planning time.
4. **`-infinity` inside a range.** For `DATE` and timestamp values, the
   smallest finite value sorts below `-infinity`. So a column range can hold
   `-infinity` *between* its bounds, and a monotone function such as `year()`
   returns `NULL` there. `PropagateFiniteMonotoneBounds` (`:226-236`) checks
   for this with `HoldsNegativeInfinity` (`:28-43`) and bails.
   `statistics_monotone_infinity.test` covers it.

Not every function that is monotone in math is marked monotone.
`statistics_exp_libm.test` explains why `exp` is not: the C library's `exp`
can return a smaller result for a slightly larger `double`.
`monotonicity_annotations.test` checks the annotations of other functions.

`CASE` (`propagate_case.cpp:6-20`) propagates by *merging* — not narrowing —
statistics: the `ELSE` branch's statistics start the accumulator, and every
`WHEN`/`THEN` branch's `THEN`-statistics get `Merge`d in (union of ranges,
OR of nullability flags), regardless of whether the corresponding `WHEN`
condition is itself statically foldable. This is why
`statistics_case.test`'s first query — `CASE WHEN i=2 THEN i+1 ELSE i+2 END`,
over a column with no nulls — can be proven never `NULL` (every branch's
merged statistics say "cannot be null"), while its second query (one branch
literally `ELSE NULL`) cannot, since `Merge` correctly propagates the
`ELSE`'s `has_null=true` into the merged result even though the `WHEN`
condition it's guarded by is data-dependent, not foldable.

`COALESCE` (`propagate_operator.cpp:47-85`, an `OPERATOR_COALESCE`
`BoundOperatorExpression`) does something sharper than merging: it
*rewrites the expression itself* as it goes. If a child's statistics prove
it can never be non-`NULL` (`!child_stats[i]->CanHaveNoNull()`) — an
all-`NULL` column, or a literal `NULL` — that child is spliced out of the
`COALESCE`'s argument list entirely (it could never be the value that wins),
setting `removed_expressions = true`; if a child's statistics prove it can
*never* be `NULL`, every argument *after* it is spliced out instead, since a
`COALESCE` short-circuits at the first guaranteed-non-null argument. This is
why `statistics_coalesce.test`'s `COALESCE(i, 17)=17` folds to
`EMPTY_RESULT` while `i` has no nulls (the `17` is unreachable, so the
comparison becomes `i=17`, provably false against `i`'s known `[1,3]`
range) but stops folding the instant a `NULL` is inserted into `i`
(§7.20.14). `IS NULL`/`IS NOT NULL` on any expression fold the same way
directly off the two nullability bits (`propagate_operator.cpp:86-117`) —
`x IS NULL` becomes `false` outright if `x`'s statistics say
`!CanHaveNull()`, and `true` outright if they say `!CanHaveNoNull()` (the
all-`NULL`-column case).

Before it folds, the `IS NULL`/`IS NOT NULL` rule asks `CanFoldNullCheck`
(`propagate_operator.cpp:26-29`). Folding removes the child expression from
the plan, so it is skipped when the child is volatile (`random()` must still
run), or when it contains a cast of a variable-size value to `VARIANT`. Such
a cast can throw at run time, even as a `TRY_CAST`, when the encoded row is
too large, and folding would hide that error.

Conjunctions (`BoundConjunctionExpression`, `propagate_conjunction.cpp:67-126`)
propagate through every child and then, for any child that turns out to be
foldable (having been rewritten to a literal `TRUE`/`FALSE`/`NULL` by its own
propagation), either prune that child out of the `AND`/`OR` list (a `TRUE` in
an `AND`, a `FALSE` in an `OR` — logically inert, drop it) or collapse the
*entire* conjunction to a constant (a `FALSE` in an `AND`, a `TRUE` in an
`OR` — logically decisive, nothing else in the list matters). Notice this
lives in the *expression* layer, entirely separate from `SimplifyFilter`
(§7.20.4's `FILTER_SIMPLIFICATION`-mode-only helper in
`propagate_filter.cpp:262-297`) — the two look similar but serve different
callers: `PropagateExpression(BoundConjunctionExpression&)` runs
unconditionally, in every mode, for every conjunction anywhere in the tree
(a `CASE WHEN a AND b`, a join condition list, a top-level filter);
`SimplifyFilter` is specifically about detecting whether a *filter's* table
bindings changed as a side effect, for the early multi-column-filter
re-pushdown described in §7.20.4.

**Redundant `constant_or_null` conjuncts are removed.**
`constant_or_null(true, x)` means "`true`, unless `x` is `NULL`". The
propagator builds such expressions itself, for example when a `BETWEEN` bound
is always true for every non-`NULL` value (§7.20.11). Inside an `AND`, such a
conjunct is often useless: if another conjunct already rejects the row when
`x` is `NULL`, the `NULL` check adds nothing. `SimplifyConstantOrNull`
(`propagate_conjunction.cpp:13-65`) removes the conjunct in that case. It runs
on the children of every `AND` and on the expression list of every
`LogicalFilter` (`propagate_filter.cpp:352`).

It is careful. It does nothing if any conjunct can throw, is volatile, or
contains an `ExpressionBarrier`, because removing a check lets more rows reach
the other predicates. And it keeps a `constant_or_null` whose argument is a
`NULL` constant. When it removes a conjunct, it sets `removed_expressions`, so
the later `UNUSED_COLUMNS` run can prune columns that only the removed check
read.

```sql
-- test/optimizer/statistics/constant_or_null_conjunction.test
CREATE TABLE quantities AS SELECT i::INTEGER AS q FROM range(1, 101) t(i);
INSERT INTO quantities VALUES (NULL);
CHECKPOINT;

EXPLAIN SELECT count(*) FROM quantities WHERE q BETWEEN 1 AND 20;
----
logical_opt	<!REGEX>:.*constant_or_null\(.*
```

`q >= 1` is true for every non-`NULL` `q`, so it becomes
`constant_or_null(true, q)`. But `q <= 20` already rejects `NULL`, so the
`constant_or_null` is dropped, and the plan keeps only `q <= 20`.
`constant_or_null_conjunction_effects.test` checks that the evaluation effects
(errors, volatile calls) are kept.

## 7.20.7 Join propagation, and the sideways filter it can synthesize

§7.20.2 already named the headline mechanics — cardinality combination,
per-condition range-overlap checks feeding `HandleJoinNeverMatches`/
`HandleJoinAlwaysMatches` (`propagate_join.cpp:16-107`, which know, per join
type, exactly what "the condition never holds" or "the condition always
holds" degenerates to: an inner or semi join with a never-true condition
becomes `LogicalEmptyResult` outright; a left/right outer join with a
never-true condition keeps its preserved side and replaces the *other* side
with an empty result, since every row on the preserved side still emits with
NULLs; an inner join whose condition is *always* true degenerates to a plain
`LogicalCrossProduct`, since the join predicate was never actually filtering
anything). One further mechanism is worth calling out in full, because it's
the single most surprising piece of this whole pass and none of the earlier
chapter material mentions it: `CreateFilterFromJoinStats`
(`propagate_join.cpp:430-478`).

After a join condition's two sides get run through `UpdateFilterStatistics`
(the two-sided overload from §7.20.5), the condition's columns may now have
*narrower* ranges than they did before the join — exactly the `i: [0,25]`
after `i=j AND i=k` example the source comment at
`propagate_join.cpp:199-205` walks through. For `INNER`/`SEMI` joins on an
integral `BoundColumnRefExpression` (checked directly —
`expr->GetExpressionType() != ExpressionType::BOUND_COLUMN_REF` bails,
`propagate_join.cpp:438`), `CreateFilterFromJoinStats` compares the
column's statistics *before* and *after* this narrowing and, if the range
genuinely shrank on either end, **synthesizes an actual new
`LogicalFilter`** — a real `>=`/`<=` `BoundComparisonExpression` against the
new bound — and splices it directly above that side's child operator
(`propagate_join.cpp:465-472`). It then immediately re-runs `FilterPushdown`
on just that child (`propagate_join.cpp:475-476`), so the brand-new,
derived-from-nowhere-in-the-original-SQL filter gets pushed all the way down
to that side's own base-table scan in the same optimizer invocation, rather
than only becoming pushable the *next* time the fixed, no-fixpoint pipeline
(§7.2) happens to run — which, per that architecture, it never does within
a single query.

This is a genuinely different mechanism from the *dynamic*, execution-time
join filter pushdown covered in §7.6.5 (`JOIN_FILTER_PUSHDOWN`, pass #42,
which records a hash join's build-side min/max range *at execution time* and
pushes it as a runtime probe-side pre-filter). `CreateFilterFromJoinStats`
is entirely static, plan-time, and derived purely from the two sides'
*compile-time-known* statistics — a real, physical `LogicalFilter` node
appears in the optimized plan, not a runtime-computed range. Both mechanisms
exist because they solve overlapping but distinct problems: this one can
fire even when neither side's actual runtime data distribution is knowable
until execution (it only needs the two *statistics* ranges to provably
shrink), and it can push a derived predicate all the way into a scan's own
`table_filters`, unlocking storage-layer zonemap skipping the dynamic
mechanism (which only prunes at the join probe, not at the scan) cannot.
`statistics_join_multicolumn.test`'s range-join cases (§7.20.14) are
precisely `CreateFilterFromJoinStats` in action.

A few smaller join rules complete the picture:

- **A join that always matches keeps its output columns.** When
  `HandleJoinAlwaysMatches` (`propagate_join.cpp:63-107`) turns a join into a
  cross product, it keeps exactly the columns the join emitted, using the
  join's projection maps (`statistics_join_always_matches_layout.test`).
- **`RIGHT_SEMI` joins** let one side's statistics filter the other, also
  below a `DISTINCT` (`right_semi_join_filter.test`).
- **Join conditions after the first** cannot claim `NOT NULL` in the
  verification statistics just because an earlier condition rejected `NULL`.
  `statistics_join_arbitrary_condition.test` covers joins that mix comparison
  conditions with arbitrary ones.
- **The early `FILTER_SIMPLIFICATION` run** (§7.20.4) counts a join range
  it inferred only once (`statistics_early_join_range.test`).

## 7.20.8 Casts, and the signed/unsigned nuance the task asked for real care on

Casts are propagated through a per-cast-function statistics callback,
`CastStatistics::Propagate` (`src/function/cast/cast_statistics.cpp:254-256`),
which every generically-bound cast gets registered with automatically unless
it supplies something more specific
(`cast_function_set.cpp:80-82,107-112`) — critically, **`CAST` and `TRY_CAST`
share the exact same registered cast function and therefore the exact same
statistics callback** for a given (source, target) type pair; the only
difference between them is in the *executor*, at the moment an individual
value actually fails to convert (`CAST` throws, `TRY_CAST` emits `NULL`).
`CastStatistics::TryPropagate` (`cast_statistics.cpp:231-252`) special-cases
`VARIANT`, `GEOMETRY`, and `ARRAY→LIST` casts, and otherwise falls through to
`CanPropagate` (`cast_statistics.cpp:83-177`) — a whitelist over
`PhysicalType` (every integer width, signed and unsigned, plus `FLOAT`/
`DOUBLE`; anything else, including `ENUM`, is refused) and a set of
temporal-type-specific refusals (timestamp-with-timezone casts are session
-timezone-dependent and would diverge from the propagator's static,
UTC-only `Value::DefaultTryCastAs`, so they're excluded outright —
`cast_statistics.cpp:150-170`, citing issue #22235 in its own comment).

Once a cast is deemed eligible, the actual range derivation
(`StatisticsOperationsNumericNumericCast`, `cast_statistics.cpp:56-81`) is
almost embarrassingly direct: it takes the input's *literal* min and max
`Value`s and calls `Value::DefaultTryCastAs(target)` on each one,
individually. **If either cast fails — for any reason, including overflow
— the entire propagation bails with `nullptr`.** There is no separate
"signed/unsigned" code path, no special-cased wraparound modeling, and
critically, *no attempt to clamp an out-of-range corner to the target
type's own domain bounds*. This is the load-bearing detail behind every one
of this chapter's signed/unsigned test files:

- **`statistics_unsigned_cast.test`**: `id::BIGINT > 100` folds to
  `EMPTY_RESULT` identically whether `id` is a signed `INT` or a `UINTEGER`
  with the same `[0, 99]` value range, because in *both* cases, casting the
  literal min (`0`) and max (`99`) up to `BIGINT` succeeds trivially — a
  widening cast, unsigned-to-signed included, never overflows when the
  source range actually fits, so `CastStatistics::TryPropagate` derives
  `BIGINT[0, 99]` either way, and `100` is outside it.
- **`statistics_unsigned_arithmetic_pruning.test`**: `x + 100::UINTEGER > 100000::UINTEGER`
  over `x UINTEGER NOT NULL` ranging `[0, 10239]` doesn't even touch the
  cast machinery — `+` is registered with increasing-monotonicity
  `ArgProperties` on both arguments, so `PropagateMonotoneBounds`
  (§7.20.6) derives `y ∈ [100, 10339]` directly, and `10339 < 100000` folds
  the whole predicate away. The point this test actually demonstrates is
  narrower than "unsigned arithmetic is special": it's that the generic
  monotone-bounds machinery works identically over unsigned physical types,
  since `NumericValueUnion`/`NumericStats::Min`/`Max` are type-parametric
  and carry no assumption about signedness one way or the other.
- **`statistics_try_cast.test`**: casting a small, always-in-range `INTEGER`
  column to `TINYINT` (`i::TINYINT`) provably never fails, so the ordinary
  `CAST`'s derived statistics say "cannot be null," and `i::TINYINT IS NULL`
  folds to `false` at the logical-plan level. `TRY_CAST(i AS TINYINT) IS NULL`
  does **not** fold identically, even over the exact same values — because,
  per the shared-callback point above, `TRY_CAST`'s statistics callback is
  the same `CastStatistics::Propagate`, which derives the exact same
  provably-non-null range statistics; the test's own `EXPLAIN` assertions
  confirm the *ordinary* cast's `IS NULL` disappears while the
  `TRY_CAST`'s stays in the plan. Read together with the source, the
  honest explanation is that `TRY_CAST`'s statistics callback derives a
  range exactly as confidently as `CAST`'s does when the corner values
  cast cleanly — the difference the test is really pinning down is that
  DuckDB does *not* special-case `TRY_CAST`'s "no exception, NULL instead"
  semantics into a permissive nullable-by-default statistics fallback; it
  runs the identical value-based corner-cast logic either way, and only the
  *executor's* handling of an actual out-of-range row differs.
- **`stats_non_numeric_cast.test`** and **`optimize_unsigned_vs_signed_ints.test`**
  are both, on inspection, correctness regression guards rather than
  optimizer-pruning showcases — the former checks that an `OR`-of-`AND`s
  filter over `VARCHAR` equality still returns the right two rows with
  `expression_rewriter` disabled (guarding against a historical bug where
  disabling that pass broke a downstream assumption this pass's filter
  handling made); the latter checks that `TRY_CAST(i - 4 AS UINT32) < 2`
  over `i ∈ [0,9]` (so `i - 4 ∈ [-4, 5]` in signed `BIGINT` arithmetic
  before the cast) returns exactly `{4, 5}`. This second case is precisely
  where the "bail rather than clamp" design in
  `StatisticsOperationsNumericNumericCast` earns its keep: the input range's
  minimum, `-4`, fails to cast to `UINT32` (a negative value has no unsigned
  representation), so the *entire* cast propagation bails with no derived
  statistics at all — it never substitutes a wrapped or clamped value in
  place of the failed corner, which is exactly what would be needed to
  produce a silently wrong range (and a silently wrong pruning decision)
  under C-style unsigned-wraparound reasoning. DuckDB's actual `CAST`/
  `TRY_CAST` semantics never wrap; the propagator's unconditional bail on a
  failed corner cast is what keeps it consistent with that.

`StatisticsOperationsNumericNumericCast` (`cast_statistics.cpp:56-81`) has
two more steps. First, if the input can only be `NULL`, it returns empty
statistics of the target type instead of looking at min/max. Second, for
`DATE` and timestamp sources it calls `WidenByInfinities`
(`cast_statistics.cpp:31-54`). In these types `-infinity` is stored as a
value that sorts *above* the smallest finite value. So a range like
`[smallest finite, -infinity]` is not what it looks like, and casting just
the two corners can give the wrong bounds. `WidenByInfinities` also casts
any infinity that lies inside the range and widens the result to include
it; if that cast fails, propagation bails as for any other corner.
`statistics_cast_infinity.test` covers it.

## 7.20.9 Aggregate propagation: `count(x) → count_star()`, and a zonemap-driven fold that skips the scan entirely

Two mechanisms live inside `LogicalAggregate` handling
(`propagate_aggregate.cpp` — both the operator-level file at
`src/optimizer/statistics/operator/propagate_aggregate.cpp` and the
expression-level `propagate_aggregate.cpp` under `expression/`, which
propagates through an individual `BoundAggregateExpression`'s own
statistics callback).

**`count(x) → count_star()`.** `count`'s aggregate function registers
`CountPropagateStats` as its statistics callback
(`src/function/aggregate/distributive/count.cpp:278-300`):

```cpp
// src/function/aggregate/distributive/count.cpp:278
unique_ptr<BaseStatistics> CountPropagateStats(ClientContext &context, BoundAggregateExpression &expr,
                                               AggregateStatisticsInput &input) {
	if (!expr.IsDistinct() && !input.child_stats[0].CanHaveNull()) {
		// count on a column without null values: use count star
		expr.FunctionMutable() = BoundAggregateFunction(CountStarFun::GetFunction());
		expr.GetChildrenMutable().clear();
	}
	return nullptr;
}
```

If the counted column's propagated statistics prove it can never be `NULL`,
`count(x)` and `count(*)` are provably identical (nothing was ever going to
be excluded by `count`'s NULL-skipping semantics), so the expression is
rewritten in place — its function swapped to `count_star`, and its one
child argument dropped entirely. This is the rewrite behind
`statistics_count_not_null_pruning.test`'s and
`statistics_count_not_null_join_pruning.test`'s core claims (§7.20.14), and
it's the direct cause of `removed_expressions = true` at the call site
(`propagate_aggregate.cpp` expression file, `:24`) — which is exactly the
signal `UNUSED_COLUMNS`'/`COLUMN_LIFETIME`'s *second* run (§7.4's #38/#39,
covered in full in the sibling deep-dive
`14-unused-columns-and-column-lifetime.md`) is watching for: a column that
used to be read only by an aggregate that no longer reads it becomes
provably dead, and gets pruned out of the scan on the rerun.

**The zonemap-driven aggregate short-circuit, `TryExecuteAggregates`**
(`propagate_aggregate.cpp`, operator file, `:176-476`) is the more dramatic
mechanism: for a `LogicalAggregate` with **no `GROUP BY`** whose every
aggregate is exactly `count_star()`, `min(col)`, or `max(col)` (over a
column reached through zero or more simple projections and/or safe
widening casts down to a bare `LogicalGet`), this pass can, in the right
circumstances, **compute the aggregate's answer directly from row-group
/partition-level statistics and delete the scan from the plan entirely** —
replacing the whole subtree with a `LogicalExpressionGet` holding literal
constant(s). It works off a *coarser*, partition-oriented statistics
structure than the rest of this file (`PartitionStatistics`/
`PartitionRowGroup`, `function/partition_stats.hpp:27-46` — not
`BaseStatistics`), obtained through a dedicated table-function callback,
`get.function.get_partition_stats`. Each partition's usability is gated on
two independent exactness flags: `stats.count_type == CountType::COUNT_APPROXIMATE`
bails the count fold; `!prg->MinMaxIsExact(storage_index) ||
prg->HasPendingWrites()` bails the min/max fold specifically
(`propagate_aggregate.cpp:121-159`, `TryGetValueFromStats`) — the two axes
are checked separately because a row deleted mid-transaction can only ever
*shrink* a count exactly (DuckDB's version-chain visibility tracking makes
"how many rows are currently visible" cheap and precise even for
uncommitted deletes), while it can *invalidate* a cached min or max outright
if the deleted row happened to be the extreme value, and no cheap
incremental way exists to know the new extreme without an actual rescan.
Where a `WHERE` clause has already attached table filters to the scan, this
pass even partitions the *partitions themselves* into "provably entirely
excluded," "provably entirely included," and "genuinely needs a scan," and
where some but not all partitions can be pre-computed, it does a genuinely
elegant partial fold: the table function is told (`SetPartitionsToScan`) to
scan only the undetermined partitions, and a `LogicalProjection` above the
now-reduced aggregate *combines* the pre-computed constant with the live
scan's own aggregate result (`pre_count + count_star_from_scan` for
`count`; `COALESCE(least(pre_min, live_min), pre_min)` for `min`, — an
expression the pass literally constructs and binds itself,
`propagate_aggregate.cpp:407-461`).

This mechanism, not `StatisticsPropagator`'s ordinary min/max narrowing, is
what `test/optimizer/partition_stats_exactness.test` and
`test/optimizer/partition_stats_prepared.test` are really testing —
covered in full in §7.20.15, because the *prepared-statement* half of that
story turns out to be a genuinely subtle, carefully-designed correctness
mechanism worth its own section.

**An ungrouped aggregate always returns one row.** `SELECT count(*) FROM t`
returns one row even when `t` is empty. So the aggregate's node statistics
say exactly that, `NodeStatistics(1, 1)`, instead of passing on the input's
cardinality (`propagate_aggregate.cpp:549-554`). With the input's estimate,
an empty input would give an estimate of zero rows, and a `LIMIT` above such
an aggregate in a join could be dropped by mistake
(`statistics_join_ungrouped_aggregate_limit.test`).

## 7.20.10 The rest, briefly: set operations, window, CTE, secure views

- **Set operations** (`propagate_set_operation.cpp`): `UNION ALL`
  (`PropagateUnion`, `:30-75`) merges each output column's statistics
  across every branch (`Merge`, so nullability and range both widen to
  cover every branch) and *adds* cardinalities. `EXCEPT` conservatively
  reuses the LHS's statistics unchanged (`:103-106` — "as in the worst
  case, nothing is filtered out"); `INTERSECT` does too, with an explicit
  `FIXME` in the source acknowledging the tighter, correct answer (the
  actual intersection of both sides' ranges) is not yet implemented
  (`:108-111`) — `statistics_setop.test`'s own comments make the same
  point, and deliberately scope its `INTERSECT` assertions to only the
  provably-safe LHS-derived claims (§7.20.14).
- **`LogicalWindow`** (`propagate_window.cpp:10-76`) propagates through
  `PARTITION BY`/`ORDER BY` expressions, frame boundary expressions, and —
  for an aggregate window function — synthesizes a temporary
  `BoundAggregateExpression` purely so it can reuse the aggregate's own
  statistics callback (the same `count`→`count_star` mechanism from
  §7.20.9 applies to a windowed `count(x)` too, not just a grouped one).
- **Materialized CTEs** (`propagate_cte.cpp`) propagate once through the
  CTE's definition and cache the result (`CTEStatistics`, keyed by
  `table_index`) so every `LogicalCTERef` reusing that CTE gets the same
  statistics without re-deriving them — correct, since a materialized CTE's
  definition genuinely executes once regardless of how many times it's
  referenced.
- **Secure views** (`propagate_secure_view.cpp:6-47`) enforce a real
  security boundary, not just a bookkeeping one: statistics for any column
  the view's *own* body reads but does not itself *emit* are deliberately
  erased before returning (`leaked_bindings`, `:27-39`) — otherwise a
  filter written outside the view, against a column the view never
  exposes, could still be pruned or proven using statistics about rows the
  view was specifically designed to hide.
- **`COPY ... TO ... (PARTITION_BY ...)`**: `PropagateStatistics` for
  `LogicalCopyToFile` stores the statistics of each partition column in
  `partition_stats`, so the partitioned write can choose how to partition.

## 7.20.11 The payoff, precisely, and how it feeds what comes after

`ReplaceWithEmptyResult` (`statistics_propagator.cpp:41-43`) is the single
mechanical move behind every "prove this predicate is unconditionally false
and collapse the subtree" claim in this chapter — it doesn't inspect
anything itself; every call site above (`propagate_filter.cpp`,
`propagate_get.cpp`, `propagate_join.cpp`) has already done the statistical
reasoning and simply wraps whatever's left in a `LogicalEmptyResult`. Once
that marker exists anywhere in the tree, it is `EMPTY_RESULT_PULLUP`
(§7.4's pass #13, `src/optimizer/empty_result_pullup.cpp`) — running
*before* `STATISTICS_PROPAGATION` in the fixed sequence, so it only catches
empty results an *earlier* pass produced, not this one's — that is
responsible for propagating "this input is empty" further up through
parent operators that would themselves necessarily still produce zero rows.
Because `EMPTY_RESULT_PULLUP` runs before this pass in the pipeline, an
empty result `StatisticsPropagator` itself produces late in the tree walk
does *not* get a second pullup pass to bubble it further upward within the
same query optimization — it stays exactly where `ReplaceWithEmptyResult`
put it, correct but not maximally hoisted.

`ExpressionRewriter::ConstantOrNull` (used throughout
`propagate_comparison.cpp`, `propagate_between.cpp`) is the softer sibling
of the same idea, for the "provably true/false *unless* an input is NULL"
case — it wraps the original input expressions (preserving their
NULL-producing side effects) around a literal boolean, so the expression
still evaluates to `NULL` on a row where the real inputs are `NULL`, while
short-circuiting the actual comparison logic on every other row.

00-overview.md §7.6.2 already covers, correctly, this pass's two
downstream consumers once it finishes: a second, `COST_BASED`-policy run of
`MultiStageAggregateRewriter` using the just-built statistics map, and — only
if statistics actually propagated — `PROJECTION_PLACEMENT` (#35), which
decides where in the operator chain a narrowing projection should physically
be materialized. Nothing in this file's source reading contradicts that;
it's restated here only to keep this file self-contained about where its
own output goes, without re-deriving that section's argument from scratch.

## 7.20.12 A full worked example, bottom-up, node by node

Take the task's own suggested shape and make it concrete:

```sql
CREATE TABLE t(x INTEGER);
INSERT INTO t SELECT range FROM range(500000);  -- x ∈ [0, 499999], no NULLs
SELECT * FROM t WHERE x > 1000000;
```

Reading `t`'s own `STATS()`-visible statistics before any predicate is
applied (the debug-only `stats(...)` scalar function `statistics_numeric.test`
uses throughout is exactly `BaseStatistics::ToStruct()` surfaced at
runtime, backed by the same `DebugVerifyStatsSetting`/`SetVerificationStats`
machinery described below) would show `x`'s base-table statistics as
`min=0, max=499999, has_null=false, has_no_null=true`. The bottom-up walk:

```
Step 1 — LogicalGet(t)                       [propagate_get.cpp:167]
┌───────────────────────────────────────────────────────┐
│ statistics_map[(t.table_index, x)] =                    │
│   NumericStats { min: 0, max: 499999,                    │
│                  has_null: false, has_no_null: true }    │
│ node_stats = NodeStatistics(estimated≈500000,             │
│                              max=500000)                  │
└───────────────────────────────────────────────────────┘
                          │  (bottom-up: child propagated first)
                          ▼
Step 2 — LogicalFilter(x > 1000000)          [propagate_filter.cpp:317]
  HandleFilter(x > 1000000):
    PropagateExpression(condition)
      → PropagateComparison(x's stats, {min:1000000,max:1000000})
      → PropagateValueComparison([0,499999], [1000000,1000000], COMPARE_GREATERTHAN)
          lmax(499999) < rmin(1000000)  →  FILTER_ALWAYS_FALSE
    ClassifyFilter → FILTER_ALWAYS_FALSE
┌───────────────────────────────────────────────────────┐
│ prune_result == FILTER_ALWAYS_FALSE                      │
│   → ReplaceWithEmptyResult(node_ptr)                     │
│   → node_ptr = make_uniq<LogicalEmptyResult>(...)        │
│   → return NodeStatistics(0, 0)                          │
└───────────────────────────────────────────────────────┘
```

The resulting `logical_opt`-stage `EXPLAIN` (matching this book's diagram
convention, arrows pointing the direction data would have flowed at
execution time) collapses to a single node:

```
┌─────────────────────────────┐
│ LogicalEmptyResult            │  x > 1000000 proven unsatisfiable
│ (types: [x INTEGER])           │  against x ∈ [0, 499999]
└─────────────────────────────┘
```

No scan, no filter, no rows read — this is
`statistics_filter.test_slow`'s pattern exactly (§7.20.14), just with round
numbers instead of the test's `{1,2,3}`-valued tables, and it is the same
`FilterPropagateResult::FILTER_ALWAYS_FALSE` → `ReplaceWithEmptyResult` path
whether the base statistics come from an ordinary table scan (as here) or
survive through an intervening `LogicalLimit`/subquery, as several of the
`statistics_filter.test_slow` and `statistics_between.test` cases
specifically go on to check.

## 7.20.13 The test corpus, organized

Forty-four dedicated files live under `test/optimizer/statistics/` — the
largest single-directory test corpus this book has covered for any one
optimizer pass. Thirty of them fall into the broad categories below. The
other fourteen each guard one specific rule described earlier in this part,
and are listed after the table. Categorized by what each one is actually
probing (read in full, not guessed from filenames):

| Category | Files |
|---|---|
| **Numeric range filter pruning** (fold a predicate to `EMPTY_RESULT`/keep it, from `x`'s known min/max) | `statistics_filter.test_slow`, `statistics_filter_pruning.test`, `statistics_numeric.test`, `statistics_numeric_comparison_pruning.test`, `statistics_between.test`, `statistics_least_greatest.test` |
| **Filter mechanics across plan boundaries / multi-column filters** | `statistics_filter_multicolumn.test`, `statistics_filter_context.test`, `statistics_removed_filter_column_pruning.test` |
| **Nullability reasoning** | `statistics_is_null.test`, `statistics_null_comparison.test`, `statistics_case_convert.test` |
| **Join-specific propagation** | `statistics_join.test`, `statistics_join_multicolumn.test`, `statistics_join_null.test` |
| **Aggregate propagation** | `statistics_aggregate.test`, `statistics_count_not_null_pruning.test`, `statistics_count_not_null_join_pruning.test` |
| **Set operations** | `statistics_setop.test` |
| **Expression-specific propagation (`CASE`/`COALESCE`)** | `statistics_case.test`, `statistics_coalesce.test` |
| **Nested types (`STRUCT`)** | `statistics_struct.test`, and the struct cases inside `statistics_filter_pruning.test` |
| **String-specific statistics (min/max, length, prefix/suffix, `CONTAINS`)** | `statistics_varchar.test`, `statistics_string_length.test`, `statistics_suffix.test`, `statistics_contains.test`, `string_expression_filter_pruning.test` |
| **Casts, including signed/unsigned nuance** | `statistics_try_cast.test`, `statistics_unsigned_cast.test`, `statistics_unsigned_arithmetic_pruning.test` |
| **Case-conversion functions** | `statistics_case_convert.test` *(also listed above — spans both categories)* |

For completeness, one sentence each for the thirty files in the table, in
the same order `ls` returns them:

- `statistics_aggregate.test` — statistics inside `GROUP BY` groups, plus
  `SUM`/`AVG` overflow-widening behavior for `BIGINT`/`DECIMAL`/`HUGEINT`.
- `statistics_between.test` — `BETWEEN` folding via `PropagateBetween`,
  across both direct-scan and subquery-with-`LIMIT` plan shapes.
- `statistics_case.test` — `CASE` statistics merging and its `NULL`
  interaction with `IS NULL`.
- `statistics_case_convert.test` — `lower`/`upper` propagate nullability
  (and, for unicode, *only* nullability, not byte length) through
  `strlen`/`length`.
- `statistics_coalesce.test` — `COALESCE` argument pruning driven by
  per-argument nullability proofs.
- `statistics_contains.test` — row-group pruning for `contains()` using
  string max-length statistics (a Chapter 17 storage-layer payoff, not a
  logical-plan fold — see §7.20.14's framing note).
- `statistics_count_not_null_join_pruning.test` — `count(x) → count_star()`
  surviving every join type and interacting correctly with projection maps.
- `statistics_count_not_null_pruning.test` — the core `count(x) →
  count_star()` rewrite, `FILTER (WHERE ...)` interactions, and
  which-column-survives-pruning behavior with each dependent optimizer
  toggled off individually.
- `statistics_filter.test_slow` — the flagship range-pruning suite, run
  over every integer/float type via `foreach type`.
- `statistics_filter_context.test` — `OR`-disjunction pruning where one
  disjunct is statistically false-or-null, and how that interacts with
  `MARK` joins created by a large `IN (...)` list.
- `statistics_filter_multicolumn.test` — the early `FilterStatisticsOptimizer`
  re-pushdown mechanism (§7.20.4), plus `CreateFilterFromJoinStats`
  (§7.20.7) via range joins.
- `statistics_filter_pruning.test` — `total_rows_scanned` profiling proof
  that arithmetic-through-filter pruning (`a*4+4 > 1000000`) actually skips
  reading rows, across `INTEGER`, `DOUBLE`, and `STRUCT` columns.
- `statistics_is_null.test` — `IS [NOT] NULL` folding off the two
  nullability bits, including the all-`NULL`-column and outer-join-introduced-
  `NULL` cases.
- `statistics_join.test` — `HandleJoinNeverMatches`/`HandleJoinAlwaysMatches`
  across inner, left, and semi/`ANY` joins.
- `statistics_join_multicolumn.test` — `CreateFilterFromJoinStats`'s
  disjoint-vs-overlapping-range distinction on two-column join conditions.
- `statistics_join_null.test` — why a provably-always-true join condition
  *cannot* degrade to a `CROSS_PRODUCT` once either side can contain `NULL`.
- `statistics_least_greatest.test` — `least`/`greatest`'s monotone-bounds
  propagation, including the all-nullable-inputs fallback case.
- `statistics_null_comparison.test` — `ConstantOrNull` vs. outright pruning,
  depending on whether the provably-decided comparison sits inside a
  `WHERE` clause or a `SELECT` list.
- `statistics_numeric.test` — the `PropagateMonotoneBounds` showcase:
  `abs`/`floor`/`sqrt`/`ln`/`log2`/`log10`/`degrees`/`radians`/`year`, their
  NaN/overflow/infinite-corner bail-out behavior, and `DECIMAL`/`HUGEINT`
  arithmetic statistics.
- `statistics_numeric_comparison_pruning.test` — row-group-level `!=`/`=`
  pruning (a Chapter 17 payoff) across numeric, temporal, boolean, `VARCHAR`,
  `UUID`, and `ENUM` types, including the disjoint-vs-shared-truncated-prefix
  string distinction.
- `statistics_removed_filter_column_pruning.test` — the general claim that a
  column whose *only* reference was an expression this pass proved
  removable is subsequently pruned from the scan by `UNUSED_COLUMNS`.
- `statistics_setop.test` — `UNION ALL` merge, `EXCEPT`/`INTERSECT`'s
  deliberately conservative LHS-only propagation.
- `statistics_string_length.test` — `length`/`strlen` byte-vs-codepoint
  statistics, unicode worst-case bounding, and `length_grapheme`'s
  deliberate exclusion from the strengthened bound.
- `statistics_struct.test` — `IS NULL` and equality folding through
  `STRUCT` field access.
- `statistics_suffix.test` — `suffix()` row-group pruning via string
  min/max/length, including `NOT suffix(...)`'s negated-null semantics.
- `statistics_try_cast.test` — `TRY_CAST`'s statistics callback derives
  ranges exactly as confidently as `CAST`'s when corners cast cleanly.
- `statistics_unsigned_arithmetic_pruning.test` — monotone-bounds
  propagation through `+` over `UINTEGER` columns.
- `statistics_unsigned_cast.test` — signed vs. unsigned source columns
  casting up to `BIGINT` fold identically when both corners fit.
- `statistics_varchar.test` — baseline correctness for `length`/`strlen`/
  `strpos`/`upper`/`lower`/`substr` over ASCII and unicode `VARCHAR` values
  (a functional-correctness anchor more than a pruning showcase).
- `string_expression_filter_pruning.test` — `replace(s,'z','z')='...'`-style
  string-expression filters still generate pruning-friendly predicates, and
  a failed-constant-evaluation-on-empty-input safety check.

The other fourteen files each pin down one rule. Each file's
`# description:` line names the bug or property it guards:

- `constant_or_null_conjunction.test`,
  `constant_or_null_conjunction_effects.test` — removing redundant
  `constant_or_null` conjuncts, and keeping their effects (§7.20.6).
- `monotonicity_annotations.test`, `statistics_exp_libm.test`,
  `statistics_monotone_infinity.test` — which functions are marked monotone,
  and `-infinity` inside a range (§7.20.6).
- `right_semi_join_filter.test`, `statistics_early_join_range.test`,
  `statistics_join_always_matches_layout.test`,
  `statistics_join_arbitrary_condition.test` — the smaller join rules
  (§7.20.7).
- `statistics_additive_duplication.test` — resetting additive statistics
  when rows are duplicated (§7.20.3).
- `statistics_all_null_comparison.test` — a filter that folds to `NULL`
  (§7.20.5).
- `statistics_cast_infinity.test` — casts of `DATE` and timestamp ranges
  that hold an infinity (§7.20.8).
- `statistics_join_ungrouped_aggregate_limit.test` — an ungrouped aggregate
  returns one row (§7.20.9).
- `statistics_root_replaced.test` — the pass replacing the root operator
  (§7.20.2).

## 7.20.14 Deep walkthroughs: fifteen of the thirty, in real depth

**`statistics_filter.test_slow`.** Eleven numeric types
(`utinyint`…`double`), each run through the identical battery: `=`, `>`,
`>=`, `<`, `<=`, each once out-of-range (against a table of `{1,2,3}`, an
out-of-range value like `i=0` or `i=4`) and once in-range, checked twice —
once as a direct scan (where `FILTER_PUSHDOWN` has already turned the
predicate into a table filter before `STATISTICS_PROPAGATION` even runs,
so `PropagateTableFilter`/`propagate_get.cpp:29-62` is the code path
exercised) and once wrapped in `(SELECT * FROM integers LIMIT 10)` (where
the intervening `LogicalLimit` blocks pushdown, so the ordinary
`LogicalFilter` path, `propagate_filter.cpp:317`, is exercised instead).
Both paths reach the identical `EMPTY_RESULT`/no-`EMPTY_RESULT` conclusion
for every type and every comparison — this is precisely the test asserting
that `NumericValueUnion`'s per-physical-type min/max comparison logic is
correct across every width and signedness DuckDB has, not just `INTEGER`.

**`statistics_filter_multicolumn.test`.** The join-condition-narrowing case
from §7.20.5's source excerpt, made concrete: `integers.i ∈ [1,3]`,
`integers2.i ∈ [2,4]`. After `JOIN ... USING (i)` narrows both sides (via
the two-sided `UpdateFilterStatistics` overload) to `i ∈ [2,3]`, `i=1` and
`i=4` both fold away (outside `[2,3]`), `i>2`/`i<3` do not (both touch
`{2,3}`'s interior). The range-join half (`i2.i < i1.i`) exercises the
asymmetric narrowing from §7.20.5's code excerpt directly: `i1`'s new
minimum only rises to match `i2`'s *pre-narrowing* minimum (2), and `i2`'s
new maximum only falls to match `i1`'s *pre-narrowing* maximum (3) — giving
`i1 ∈ [2,3]`, `i2 ∈ [2,3]`, from which `i2=4` and `i1=1` fold but `i2=3`/
`i1=2` do not, matching the asymmetric-clamp reasoning exactly.

**`statistics_filter_context.test`.** The subtlest file in the whole
corpus. Its central claim: `price > 100` against a column whose known
statistics prove it's always false-or-null (`price`'s range never reaches
100) behaves *differently* depending on whether it sits, unguarded, as a
top-level filter conjunct/disjunct versus inside a *join condition*. In a
row-filter context, `FILTER_FALSE_OR_NULL` for one disjunct of an `OR` still
lets the *other* disjuncts decide the row's fate on their own — a `NULL`
can never make an `OR` chain pass by itself. So
`substring(zip,1,5) IN (...) OR state='CA' OR price > 100` simplifies its
`price > 100` disjunct away entirely before the `IN`-list-to-join-conversion
(pass #9, `IN_CLAUSE`) even runs, letting that whole predicate become a
single-table filter on `address` alone and get pushed below the join,
*before* the large `IN` list gets turned into a `MARK` join
(`logical_opt <REGEX>...INNER.*IN \(\.\.\.\).*MARK.*filter_context_address...`
— the test's own `EXPLAIN` assertion). But the file also demonstrates the
opposite discipline explicitly: `SELECT state = 'CA' OR price > 100`
(no `WHERE`, this expression is the *output*) must keep the `NULL`
observable via `ConstantOrNull`, since a `SELECT`-list expression's `NULL`
is a real, user-visible result, not a row-admission decision — and a join
*condition* (`ON flag OR v > 100`) gets the same `constant_or_null`
treatment as a `SELECT`-list expression, not the filter-context treatment,
because a join's `ON` clause has its own NULL semantics (a `NULL` condition
still routes the row through an outer join's NULL-extended path) distinct
from a row filter's "false or null both reject the row" rule. Three
different NULL-handling contexts, three different foldings, from the
identical underlying `FILTER_FALSE_OR_NULL` classification.

**`statistics_filter_pruning.test`.** Proves the pruning is real, not just
`EXPLAIN`-cosmetic, by tracking `query.total_rows_scanned` through
`enable_profiling='json'` around `a*4+4 > 1000000` (`a ∈ [-150000, 149999]`)
— a chain of two monotone functions composed (`*4` then `+4`, both
increasing), correctly bounded to `[-599996, 599996]`, entirely below
`1000000`, folding the whole predicate to `EMPTY_RESULT` and letting the
physical scan skip reading rows it would otherwise have to. The `STRUCT`
variant (`a.b + 5 > 1000000`, and again via `struct_extract`/
`struct_extract_at`) proves the exact same monotone-bounds machinery works
identically once a value is reached through struct field access — the
propagator doesn't need struct-specific logic here because `struct_extract`
is just another `BoundFunctionExpression` whose statistics callback (in
`struct_stats.cpp`) forwards the child field's own `BaseStatistics`
directly.

**`statistics_removed_filter_column_pruning.test`.** Uses a custom
`json_serialize_plan`-based macro (`scan_columns`) to count a scan's
"`FULL_READ`" column entries directly from the serialized plan, precisely
because — as its own comment explains — a column read only to feed a
filter is projected out of the scan by `COLUMN_LIFETIME` immediately, so it
never shows up in `EXPLAIN`'s `Projections:`/`Filters:` text at all; the
only way to see it is to serialize the plan and count. It systematically
walks every place this pass can prove a filter/join-condition column
unreachable — an always-true single-column filter (`propagate_get.cpp`'s
table-filter path), an always-true filter *without* pushdown
(`propagate_filter.cpp`/`propagate_conjunction.cpp`), a `COALESCE` argument
made unreachable (`propagate_operator.cpp`), and an always-true join
condition, across every one of `INNER`/`LEFT`/`RIGHT`/`FULL`/`SEMI`/`ANTI`
— and confirms, in a second loop, that disabling `statistics_propagation`,
`unused_columns`, or `column_lifetime` individually never changes the
*answer*, only whether the pruning happens at all. This is exactly the
cross-pass dependency §7.4 describes for passes #38/#39 rerunning after
#34 — this test is that dependency's correctness proof.

**`statistics_numeric.test`.** The `PropagateMonotoneBounds` reference
file. Its `STATS(...)` idiom — `SELECT s.min, s.max FROM (SELECT
STATS(expr) s FROM t LIMIT 1)` — surfaces the *literal derived*
`BaseStatistics` as a runtime struct value, via the same
`BaseStatistics::ToStruct()` machinery that backs the
`DebugVerifyStatsSetting`-driven runtime self-check described in §7.20.16.
Its most instructive individual cases: `abs(i)` derives `[0, max(|min|,
|max|)]` only once the column's range actually straddles zero (the source
comment "the call to abs can no longer be removed" tracks a real behavior
change — once `i`'s range includes both signs, `abs`'s output range is no
longer identical to `i`'s own, so the projection can't fold `abs` away
entirely); `year(dt)` over an `infinity`-valued date corner returns `NULL`
at that corner, and `is_unusable` correctly bails rather than deriving a
`NULL`-tainted bound; `sqrt`/`ln` over a range including negative values
hit the same bail, since DuckDB's `sqrt`/`ln` of a negative number is
NaN-or-error, not a real corner value.

**`statistics_between.test`.** Every `FILTER_ALWAYS_TRUE`/`FALSE`/
`TRUE_OR_NULL`/`FALSE_OR_NULL` combination `PropagateBetween`
(`propagate_between.cpp:9-69`) branches on, made concrete: a `BETWEEN`
whose lower bound is trivially true rewrites to a plain `<=` upper-bound
comparison (`i BETWEEN 0 AND 2` → `i <= 2`, once `i`'s known minimum is 0);
one whose upper bound is trivially true rewrites to `>=`; one whose lower
bound exceeds its upper bound (`BETWEEN 3 AND 2`) folds to `EMPTY_RESULT`
unconditionally, independent of the column's own statistics, since no value
can ever satisfy `x>=3 AND x<=2`. It also documents, by its own comment
citing two DuckDB-fuzzer issue numbers, that a `BETWEEN` whose both bounds
are provably always-true only fully disappears from the *physical* plan
(not the logical one) — a subtlety worth taking at face value rather than
assuming is a documentation error.

**`statistics_is_null.test`.** The full nullability-bit truth table: no
nulls in the base column → both `IS NULL`/`IS NOT NULL` fold statically;
some-but-not-all nulls → neither folds; a filter that removes nulls as a
side effect (`WHERE i>0`) makes the *subsequent* `IS NULL` foldable even
though the base column itself has nulls, since `UpdateFilterStatistics`'s
`CANNOT_HAVE_NULL_VALUES` side effect (§7.20.5) applies regardless of
comparison direction; and — genuinely subtle — a `LEFT JOIN ... ON (false)`
degrades (via `HandleJoinNeverMatches`) to a cross product against a
NULL-only right side whose *columns*' statistics get explicitly marked
`CAN_HAVE_NULL_VALUES` by the join-null-introduction logic in
`propagate_join.cpp:353-370`, so `i2.i IS NULL` folds to always-true; but a
`FULL OUTER JOIN ... ON (false)` marks *both* sides nullable, so `i1.i IS
NULL` (querying the *left*, normally-non-null side) correctly does **not**
fold, since a full outer join really can introduce nulls on either side
regardless of what the base tables contained.

**`statistics_join.test` / `statistics_join_null.test`.** Companion files
proving the same always-matches/never-matches join logic (§7.20.7) with and
without nulls present. The pairing is the whole point: `i1.i < i2.i` over
non-nullable `{1,2,3}`/`{4,5,6}` degrades all the way to a plain
`CROSS_PRODUCT` (provably always true, no nulls to worry about); the exact
same condition over the *nullable* variant of those tables (`{1,2,NULL}`/
`{4,5,NULL}`) does **not** degrade, because — as `statistics_join_null.test`'s
own comment states outright — doing so would be unsound unless the
optimizer *also* pushed an `IS NOT NULL` filter onto both sides first, which
this pass's `HandleJoinAlwaysMatches` deliberately does not attempt.

**`statistics_join_multicolumn.test`** and **`statistics_least_greatest.test`**
are covered in §7.20.7 and §7.20.6/§7.20.8 respectively, in full, as the
flagship examples of `CreateFilterFromJoinStats` and
`PropagateMonotoneBounds` — no further repetition needed here beyond noting
that `statistics_least_greatest.test`'s "two nullable columns, no constant"
case specifically exercises the `output_can_have_null` accumulation across
every argument in `PropagateMonotoneBounds` (§7.20.6), not just the
single-nullable-plus-constant case.

**`statistics_count_not_null_pruning.test` / `_join_pruning.test`.**
§7.20.9's `CountPropagateStats` rewrite, exercised against every
complication that could plausibly break it: a `FILTER (WHERE ...)` clause
over a *different* column (rewrites cleanly, the filter column alone
survives), a `FILTER` over the *same* counted column (does **not** rewrite
to `count_star`, since the filter condition still needs to read that
column's actual values — the test's own `EXPLAIN` regex confirms `user_id`
survives in that case), an `ORDER BY` inside the aggregate call (count is
declared `NOT_ORDER_DEPENDENT`, so the ordering is irrelevant and the
rewrite still fires), every join type, a materialized CTE referenced
through a `UNION ALL` that itself triggers `COMMON_SUBPLAN`, and — the
`_pruning.test` file's own explicit point — non-nullness proven purely by a
*pushed-down filter*, not by the column's own declared nullability,
propagating correctly into the count-star rewrite even though the scan no
longer even projects the column by the time the rewrite runs.

**`statistics_setop.test`.** `EXCEPT`/`INTERSECT`'s conservative,
LHS-only propagation, made concrete: `integers2 EXCEPT integers3`
(`[4,6]` minus a set including `NULL`) still reports `[4,6]`, *not* the
tighter true post-subtraction range, because `PropagateStatistics
(LogicalSetOperation&)`'s `EXCEPT` branch (`propagate_set_operation.cpp:103-106`)
literally just copies the LHS's stats unchanged — the test's own comments
call this out as a known, deliberate conservatism rather than an oversight,
and scope every assertion to claims that hold true regardless (`i=7` still
folds, since `7` is outside even the *unrefined* `[4,6]`; `i=5` correctly
does *not* fold, since `5` is inside it).

**`statistics_string_length.test`.** The string-byte-length statistics
showcase in full. Fixed-width strings (every value exactly 15 characters)
let `length(v) > 30` fold to `EMPTY_RESULT` outright. Variable-width data
demonstrates genuine row-group-level pruning (a Chapter 17 payoff, using the
same min/max-string-length fields from §7.20.3) via
`EXPLAIN ANALYZE`'s `Row Groups Scanned: 1 / 2`. The unicode cases are the
most careful: since every UTF-8 codepoint takes at most four bytes, a
row-group whose *minimum byte length* is 40 can be proven to contain
nothing shorter than `ceil(40/4)=10` *characters* — the file's own comment
walks the arithmetic, including an exact rounding-boundary case (41 bytes
implying at least 11 characters) — and it explicitly, separately verifies
that this byte-derived strengthening is applied **only** to `length()`
(codepoint counting) and never to `length_grapheme()` (grapheme-cluster
counting, where a single cluster can span arbitrarily many codepoints and
bytes, breaking the `bytes/4` bound entirely) — the
`length_grapheme(v) < 10` case over 19-codepoint, 37-byte clusters
deliberately scans both row groups, confirming the stronger bound was
correctly withheld from that function.

## 7.20.15 `partition_stats_exactness.test` and `partition_stats_prepared.test`: exactness, and why a prepared statement can't go stale here

`partition_stats_exactness.test` is the correctness contract behind
§7.20.9's `TryExecuteAggregates` fold, worked through methodically across
storage-layer state transitions: a freshly-appended, never-deleted table
keeps `count(*)`/`min`/`max` exact (no `Sequential Scan` in the physical
plan) across a `CHECKPOINT` and even a full `restart` (append-only
operations carry no deletion uncertainty at all). An **in-process delete**
(uncommitted, same transaction) keeps `count(*)` exact — DuckDB's
version-chain visibility tracking makes "how many rows are currently
committed-and-visible" cheap and precise even mid-transaction — but flips
`min`/`max` to *inexact* (a real `Sequential Scan` reappears in the plan),
because the deleted row might have *been* the cached minimum or maximum,
and there is no cheap incremental way to know the new extreme value without
actually looking. After a full `restart`, *persisted* deletes go a step
further and make even `count(*)` inexact — the comment is explicit that
"persisted deletes are not loaded during optimization," so a fresh process
genuinely cannot know the discounted count without a scan. A `BEGIN`/
`DELETE`/`EXPLAIN`/`ROLLBACK` sequence inside the same test confirms
`count(*)` is never foldable to a *literal* the instant there's an
uncommitted delete at all sitting in the *current* transaction being
optimized (`physical_plan <REGEX>Sequential Scan` right after the `DELETE`,
even before any `CHECKPOINT`) — reconciling this with the "count stays
exact across a pending delete" claim a few lines earlier requires reading
closely: the fold happens off *partition-level* aggregated statistics that
are themselves periodically recomputed, and an in-flight `DELETE` inside an
still-open, uncommitted transaction is exactly the case `HasPendingWrites()`
(`function/partition_stats.hpp:32`) is built to signal — a genuinely open
question this book flags honestly rather than resolves with more confidence
than the source supports: the exact boundary between "pending write blocks
even count's exactness" and "count survives a pending delete" traces to
`PartitionRowGroup::HasPendingWrites()`/`CountType` accounting inside the
storage layer proper (Chapter 17's territory), not to anything in
`src/optimizer/statistics/`.

`partition_stats_prepared.test` is short — eleven lines of SQL — and is the
sharpest, most carefully-designed test in this entire corpus for exactly
the correctness question the task asked to be investigated with real care:
**does a prepared statement's statistics-driven fold go stale across
multiple `EXECUTE`s of the same prepared plan, if the underlying data (or
the surrounding transaction) changes in between?**

```sql
CREATE TABLE prepared_count_rollback AS SELECT range::INTEGER AS i FROM range(11);
BEGIN;
DELETE FROM prepared_count_rollback WHERE i = 0;
PREPARE prepared_count_stmt AS SELECT count(*) FROM prepared_count_rollback;
EXECUTE prepared_count_stmt;  -- 10
ROLLBACK;
EXECUTE prepared_count_stmt;  -- 11
```

The naive worry: if `PREPARE` folded `count(*)` into a literal `10` (using
`TryExecuteAggregates`, since the pending delete still lets count stay
exact per the exactness test above), and `EXECUTE` simply reused that
already-optimized plan verbatim, the second `EXECUTE` — after `ROLLBACK`
restores the eleventh row — would incorrectly still print `10`. The test
demonstrates it prints `11` instead, correctly. The mechanism is *not*
narrowly about `STATISTICS_PROPAGATION` re-deriving fresh statistics each
time — DuckDB's default `EXECUTE` path, `Binder::Bind(ExecuteStatement&)`
(`src/planner/binder/statement/bind_execute.cpp:16-107`), explicitly
reuses the *cached*, already-bound-and-optimized `PreparedStatementData`
without any rebind at all unless `PreparedStatementData::RequireRebind`
says otherwise (`bind_execute.cpp:70-89`) — parameter type mismatches or a
catalog version change are the ordinary triggers
(`prepared_statement_data.cpp:96-131`). Neither applies here: no catalog
object changed shape, and this prepared statement takes no parameters at
all.

What actually forces a full rebind-and-reoptimize on *every single*
`EXECUTE` of this specific statement is a blanket, conservative rule set at
the very top of `Optimizer::Optimize`, immediately after the mandatory
aggregate lowering and before any of the ~45 heuristic passes run at all:

```cpp
// src/optimizer/optimizer.cpp:585-592
this->plan = std::move(plan_p);
if (plan->type == LogicalOperatorType::LOGICAL_PREPARE) {
	auto &prepared = plan->Cast<LogicalPrepare>().prepared;
	// Optimizers can embed the current database state in the executable plan.
	if (!prepared->properties.read_databases.empty() && ContainsDataSource(*plan)) {
		prepared->properties.always_require_rebind = true;
	}
}
```

Any `PREPARE`d statement whose plan (a) actually reads from a real
database (`read_databases` non-empty) and (b) contains at least one real
`LogicalGet` (`ContainsDataSource`, `optimizer.cpp:187-197`, a plain
recursive tree walk) is unconditionally marked
`always_require_rebind = true` — and, correspondingly, its `LogicalPrepare`
physical-plan generator (`plan_prepare.cpp:8-17`) deliberately *skips*
building and caching a physical plan at `PREPARE` time at all whenever that
flag is set. `RequireRebind` (`prepared_statement_data.cpp:107-109`) checks
that flag *first*, before any parameter or catalog-version check, and
returns `true` unconditionally when it's set — which sends `bind_execute.cpp`
down its `ATTEMPT_TO_REBIND` branch (`:82-89`) every time: a brand-new
`Planner` re-parses, re-binds, and — critically — **re-runs the entire
optimizer, including `STATISTICS_PROPAGATION` and `TryExecuteAggregates`,
from scratch**, against whatever the database and transaction actually look
like *at that specific `EXECUTE` call*.

The comment on that four-line block states its own rationale plainly:
*"Optimizers can embed the current database state in the executable plan."*
This is not a narrow fix aimed at `TryExecuteAggregates` specifically — it's
a deliberate, blanket safety valve covering *every* pass in this chapter
capable of baking a fact about the database's current contents into a
plan: a constant fold, an `EMPTY_RESULT` proof, a cardinality estimate used
to pick a join algorithm, this pass's own aggregate short-circuit. Rather
than trying to enumerate every pass that could be responsible and rebind
only when one of *them* specifically fired, DuckDB simply refuses the
"prepare the physical plan once, execute it cheaply many times" optimization
for any statement that touches a real table at all, and accepts the cost of
a full re-bind-and-re-optimize on every `EXECUTE` in exchange. That is
precisely why `partition_stats_prepared.test` passes: the *first* `EXECUTE`
re-optimizes fresh (inside the still-open transaction, seeing the pending
delete, correctly folding to `10`); the `ROLLBACK` changes the transaction's
visible state; the *second* `EXECUTE` re-optimizes fresh yet again (this
time seeing no delete at all), and `TryExecuteAggregates` folds to a fresh,
correct `11` rather than replaying a stale cached constant. The staleness
risk this section set out to investigate is real in principle — and DuckDB's
answer to it is not a targeted fix inside `StatisticsPropagator` at all, but
a general, plan-shape-based refusal to cache physical plans for any
data-touching prepared statement in the first place.

## 7.20.16 `estimated_cardinalities_*.test`: filed here, owned by Chapter 8

Both `estimated_cardinalities_are_in_logical_plan.test` and
`estimated_cardinalities_through_struct_extract.test` sit in
`test/optimizer/`, immediately alongside this pass's own dedicated
`statistics/` subdirectory, and their names invite exactly the confusion
§7.20.1's table exists to head off. Reading both in full confirms they
belong to the *other* system:

`estimated_cardinalities_are_in_logical_plan.test` asserts that a three-way
equi-join's `EXPLAIN` output shows a non-trivial `~N` cardinality
annotation on its `LOGICAL_COMPARISON_JOIN` node
(`logical_opt <REGEX>...COMPARISON_JOIN.*a = b.*~[0-9]+...`). That
annotation is `LogicalOperator::estimated_cardinality`, and for a join
node specifically, it is set by `JOIN_ORDER`'s DPhyp search using
`RelationStatisticsHelper`/`cardinality_estimator.cpp` (Chapter 8 §8.6),
not by anything `StatisticsPropagator` computes — `STATISTICS_PROPAGATION`
runs seventeen passes *later* and never revisits a join's own
`estimated_cardinality` field once `BUILD_SIDE_PROBE_SIDE` (#27) has
already used it to pick a hash-join build side.

`estimated_cardinalities_through_struct_extract.test` is the more elaborate
of the two: it builds every TPC-DS table as a single-column `STRUCT`-wrapped
table (literally `CREATE TABLE {tbl}_struct AS SELECT {tbl}_normal {tbl}
FROM {tbl}_normal`), rebuilds every ordinary TPC-DS view as a
`SELECT struct_field AS struct_field, ... FROM {tbl}_struct` projection over
that one struct column, and asserts TPC-DS query 64's seventeen-way join
never shows a degenerate `~0` cardinality estimate anywhere in its plan.
The mechanism that keeps this correct lives in
`RelationStatisticsHelper::ExtractProjectionStats`
(`relation_statistics_helper.cpp:66-89`), via a small recursive helper,
`GetChildColumnBinding` (`:25-63`): for any projection expression, it walks
down through arbitrary wrapping function calls (a `struct_extract`/`.field`
access included, since it isn't a zero-argument function and isn't a
special-cased expression class, so it falls to the generic
`EnumerateChildren` recursion at the bottom of the function) until it finds
a `BoundColumnRefExpression` underneath, and reuses *that* underlying
column's own `distinct_count` for the projected output — rather than
falling back to a "no column reference found, assume worst case" default
that would otherwise make every one of these struct-wrapped views look, to
the cardinality estimator, like it produces a column with no useful
distinct-value information at all. Like the previous test, this is squarely
`RelationStatisticsHelper`/Chapter 8's mechanism, exercised through a
deliberately elaborate TPC-DS fixture; it is filed in this chapter's
material only because its filename and its neighbors in `test/optimizer/`
make it easy to assume otherwise, and getting that attribution right is
exactly the discipline §7.20.1 asked to be kept honest about.

## 7.20.17 Edge cases and limitations, recapped

- **Distinct count is inert cargo, not a live estimate**, in this pass —
  §7.20.3's point restated as a limitation: if you need a distinct-count
  number that's actually been *refined* by a filter or a join, that number
  lives in the other system (Ch. 8), not here.
- **String statistics can be truncated**, and every comparison against a
  truncated max correctly widens to the next representable prefix
  (`FindNextPrefix`) rather than risk an unsound conclusion — but this also
  means two truncated prefixes that happen to be *equal* prove nothing
  about the full strings' relative order, which is exactly why
  `statistics_numeric_comparison_pruning.test`'s "equal truncated prefix"
  case must fall back to scanning every row group (§7.20.14).
- **Casts bail rather than clamp** on any out-of-range corner — a
  deliberate, conservative design choice (§7.20.8) that trades some missed
  pruning opportunities (a cast whose *actual* runtime values would all
  fit, but whose statically-known corner value doesn't) for the guarantee
  that no fold ever silently assumes wraparound semantics DuckDB's real
  `CAST`/`TRY_CAST` don't have.
- **`EXCEPT`/`INTERSECT` propagate deliberately loose statistics** — the
  source's own `FIXME` (§7.20.10) — so pruning opportunities a *tighter*,
  correctly-intersected range would unlock are left on the table, in
  exchange for never claiming a range narrower than what's actually
  possible.
- **The pass runs conditionally at all**: both the main #34 invocation and
  the early `FilterStatisticsOptimizer` re-entry (§7.20.4) are skipped
  outright whenever the plan contains a DML statement inside a CTE
  (`CTEContainsDML`) — table statistics captured mid-DML can already be
  stale relative to what the DML itself is about to write, and this pass
  declines to reason about them at all in that shape, rather than risk
  propagating a fact about to become false.
- **A prepared statement that touches real data never benefits from this
  pass's plan-time work being cached across executions** (§7.20.15) — every
  `EXECUTE` re-derives fresh statistics from scratch, by design, precisely
  because this pass (among others) is exactly the kind of thing that could
  otherwise embed a fact about the database that stops being true.
