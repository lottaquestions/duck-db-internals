# 7.24 `JOIN_FILTER_PUSHDOWN` — a filter the optimizer cannot compute

**One-sentence purpose.** `JoinFilterPushdownOptimizer` (pass #42, the last
structural pass in §7.4's list, `src/optimizer/join_filter_pushdown_optimizer.cpp`)
does not compute a filter at all — it wires up the *plumbing* so that, while a
`PhysicalHashJoin`'s build side is being hashed at execution time, the range
(and, as this checkout's source shows, sometimes a Bloom filter or an exact
membership set) of the join key's actual values gets recorded and handed to
the probe side's table scan as an extra, execution-time-only predicate, so
that a probe row provably outside the build side's key domain never reaches
the hash-table lookup at all.

That single sentence hides the entire reason this pass is interesting enough
to deserve its own chapter-length treatment: **every other pass in §7.4's
list rewrites the plan using information available while the plan is still a
plan.** `JOIN_FILTER_PUSHDOWN` is different in kind, not just in mechanism —
the values that make the filter useful (the build side's actual min, actual
max, actual distinct-value set) provably do not exist until the query is
already running. This section is the full source walkthrough — optimizer
side, execution side, and the cast-safety machinery that makes crossing a
probe/build type mismatch safe — followed by every real test file the task
list named, walked end to end, and a precise, code-cited account of why this
pass is the one place in the whole optimizer where `EXPLAIN` and `EXPLAIN
ANALYZE` can show genuinely different plans rather than the same plan with
extra numbers stapled on.

## 7.24.1 Where this sits, and what "after the dust has settled" means literally

`optimizer.cpp:542-546` (§7.4's pass #42 citation) shows the entire call site:

```cpp
// src/optimizer/optimizer.cpp:542-546
// perform join filter pushdown after the dust has settled
RunOptimizer(OptimizerType::JOIN_FILTER_PUSHDOWN, [&]() {
	JoinFilterPushdownOptimizer join_filter_pushdown(*this);
	join_filter_pushdown.VisitOperator(*plan);
});
```

"After the dust has settled" is not a throwaway phrase. By position #42, every
pass that could still change a join's shape has already run: `JOIN_ORDER`
(#17) has fixed which relations join in which order; `BUILD_SIDE_PROBE_SIDE`
(#27) has fixed which child of each join is build and which is probe (and, as
§7.24.9 below shows, that decision loops back to influence this very pass);
`UNUSED_COLUMNS`/`COLUMN_LIFETIME` (#23/#26, re-run at #38/#39) have already
pruned and re-projected every column set; `STATISTICS_PROPAGATION` (#34) has
already run its own, purely *static*, min/max propagation. `JoinFilterPushdownOptimizer`
needs the join tree, the join keys' final expressions, and every column's
final type to already be frozen, because everything it does is about
recording *how to reconstruct* an expression later — get any of those wrong
and the reconstruction breaks. Running any earlier would mean rebuilding this
bookkeeping every time a later pass touched the tree; running last means it's
built exactly once, against the plan shape that will actually reach the
physical-plan generator.

## 7.24.2 `VisitOperator` and `GenerateJoinFilters`: the entry point

The whole optimizer object is a `LogicalOperatorVisitor` with one override:

```cpp
// src/optimizer/join_filter_pushdown_optimizer.cpp:447-453
void JoinFilterPushdownOptimizer::VisitOperator(LogicalOperator &op) {
	if (op.type == LogicalOperatorType::LOGICAL_COMPARISON_JOIN) {
		// comparison join - try to generate join filters (if possible)
		GenerateJoinFilters(op.Cast<LogicalComparisonJoin>());
	}
	LogicalOperatorVisitor::VisitOperator(op);
}
```

Every `LogicalComparisonJoin` in the tree gets one call to `GenerateJoinFilters`
— there is no cross-join bookkeeping, no global reasoning about which of
several joins in a query is "most worth" a dynamic filter (that decision, as
§7.24.9 shows, was already made earlier, by `BUILD_SIDE_PROBE_SIDE`'s
`DynamicFilterBuildBonus`). Each join is considered entirely on its own.

`GenerateJoinFilters` (`join_filter_pushdown_optimizer.cpp:348-445`) is the
whole pass's real body. Reading it top to bottom:

**Step 1 — the join-type gate, first thing, unconditionally:**

```cpp
// src/optimizer/join_filter_pushdown_optimizer.cpp:348-351
void JoinFilterPushdownOptimizer::GenerateJoinFilters(LogicalComparisonJoin &join) {
	if (!JoinFilterPushdownUtil::JoinTypeIsSupported(join.join_type)) {
		return;
	}
```

This is the single line that keeps LEFT and FULL OUTER joins safe, and it's
worth showing in full before anything else, because §7.24.10 is built entirely
on top of it:

```cpp
// src/optimizer/join_filter_pushdown_optimizer.cpp:129-145
bool JoinFilterPushdownUtil::JoinTypeIsSupported(JoinType join_type) {
	switch (join_type) {
	case JoinType::MARK:
	case JoinType::SINGLE:
	case JoinType::LEFT:
	case JoinType::OUTER:
	case JoinType::ANTI:
	case JoinType::RIGHT_ANTI:
		// cannot generate join filters for these join types
		// mark/single - cannot change cardinality of probe side
		// left/outer always need to include every row from probe side
		// FIXME: anti/right_anti - we could do this, but need to invert the join conditions
		return false;
	default:
		return true;
	}
}
```

Six exclusions, three distinct reasons, all in the source's own comments:
`MARK`/`SINGLE` must emit **exactly one output row per probe row** (a
three-valued mark, or a single matched-or-NULL scalar) — dropping a probe row
before the join runs would silently change the output's *row count*, which
these join types must never do, independent of whether that row would have
matched. `LEFT`/`OUTER` must **preserve every probe-side row**, matched or
not (NULL-extending the unmatched ones) — §7.24.10 works through exactly why a
build-side-derived range filter would corrupt this. `ANTI`/`RIGHT_ANTI` are
flagged `FIXME`: the pass *could* in principle push a filter for these (an
anti join's output is exactly the probe rows that *don't* match, so a filter
derived from the build side's key range would need to be logically inverted
— "keep only rows outside this range" — which the current code has no
representation for), so this is a known, named gap rather than an oversight.
`INNER`, `SEMI`, and (as §7.24.10 shows) `RIGHT` all fall through to `default:
return true` — for all three, a probe row that provably can't match anything
on the build side is legitimately absent from the output anyway, so dropping
it early changes *when* it's discarded, never *whether* it's discarded.

**Step 2 — reorder conditions, then walk each comparison condition looking
for a pushdown-eligible LHS expression:**

```cpp
// src/optimizer/join_filter_pushdown_optimizer.cpp:352-388
// re-order conditions here - otherwise this will happen later on and invalidate the indexes we generate
PhysicalComparisonJoin::ReorderConditions(join.conditions);
auto pushdown_info = make_uniq<JoinFilterPushdownInfo>();

vector<JoinFilterPushdownColumn> pushdown_columns;
for (idx_t cond_idx = 0; cond_idx < join.conditions.size(); cond_idx++) {
	auto &cond = join.conditions[cond_idx];
	if (!cond.IsComparison()) {
		continue;
	}
	switch (cond.GetComparisonType()) {
	case ExpressionType::COMPARE_EQUAL:
	case ExpressionType::COMPARE_LESSTHAN:
	case ExpressionType::COMPARE_LESSTHANOREQUALTO:
	case ExpressionType::COMPARE_GREATERTHAN:
	case ExpressionType::COMPARE_GREATERTHANOREQUALTO:
		break;
	case ExpressionType::COMPARE_NOT_DISTINCT_FROM:
		// TODO: Need OR predicates
		continue;
	default:
		continue;
	}

	JoinFilterPushdownColumn pushdown_col;
	if (!JoinFilterPushdownUtil::PushdownJoinFilterExpression(cond.GetLHS(), pushdown_col)) {
		continue;
	}

	pushdown_col.join_filter_idx = pushdown_info->join_condition.size();
	pushdown_columns.push_back(pushdown_col);
	pushdown_info->join_condition.push_back(cond_idx);
}
if (pushdown_columns.empty()) {
	// could not generate any filters - bail-out
	return;
}
```

`ReorderConditions` runs first specifically so that the indices this pass is
about to compute (`join_condition`, an index into `join.conditions`) stay
valid — physical join execution reorders equality conditions ahead of
inequality ones for its own reasons, and if that reorder happened *after*
this pass had already recorded index `2` as "the `orderkey` condition,"
index `2` might mean something else by the time execution reads it back.
`COMPARE_NOT_DISTINCT_FROM` (NULL-safe equality) is explicitly excluded with
a `TODO`, not silently mishandled — this is the exact fact
`07-filter-pullup-and-pushdown.md` §2.10 already leaned on when explaining
why `SimplifyNullSafeSemiJoinConditions` downgrades a mark join's NULL-safe
condition back to plain `COMPARE_EQUAL` once it becomes an ordinary semi
join: plain equality is what unlocks this pass; NULL-safe equality opts a
join out of it silently, by simply never matching any case in this `switch`.

**`cond.GetLHS()`, not `cond.GetRHS()`.** By convention (and because
`ReorderConditions` just ran), a `JoinCondition`'s left-hand side references
the *left* child of the join — i.e., the probe side — and its right-hand
side references the build side. `PushdownJoinFilterExpression` is being
asked, here, "can this **probe-side** expression be traced back to a plain
scanned column, possibly through some number of casts?" — because the filter
is going to be installed on the probe side's scan. The min/max *values*, by
contrast (Step 4 below), get computed from `cond.GetRHS()` — the build side.

**Step 3 — `PushdownJoinFilterExpression`: recording, not evaluating, a cast
chain.** This is the function that decides eligibility at the *expression*
level, and it is worth reading in full because §7.24.6 (`runtime_filter_cast.cpp`)
is entirely about what happens with the information it records:

```cpp
// src/optimizer/join_filter_pushdown_optimizer.cpp:27-43
static bool IsJoinFilterPushdownIntegralType(const LogicalType &type) {
	return type.IsIntegral() && GetTypeIdSize(type.InternalType()) <= GetTypeIdSize(PhysicalType::INT64);
}

static bool IsJoinFilterPushdownIntegralCast(const LogicalType &src, const LogicalType &tgt) {
	return IsJoinFilterPushdownIntegralType(src) && IsJoinFilterPushdownIntegralType(tgt);
}

static bool IsJoinFilterPushdownVariantIntegralCast(const LogicalType &src, const LogicalType &tgt) {
	if (src.id() == LogicalTypeId::VARIANT) {
		return IsJoinFilterPushdownIntegralType(tgt);
	}
	if (tgt.id() == LogicalTypeId::VARIANT) {
		return IsJoinFilterPushdownIntegralType(src);
	}
	return false;
}
```

`IsJoinFilterPushdownIntegralType` caps at 64 bits — `HUGEINT` (128-bit) is
explicitly too wide, which §7.24.7's cast test exercises directly. Note
precisely what these two helpers gate: **casts**, not columns in general. A
bare column reference of *any* orderable type — `VARCHAR`, `DOUBLE`, whatever
— is unconditionally eligible (see the recursive function below); the
integral-only restriction applies exclusively to the *cast-crossing* case,
where the probe side's raw storage type differs from the type the join
condition actually compares in. §7.24.7's `DOUBLE`- and §7.24.11's
`VARCHAR`-keyed test cases both hinge on this distinction, and it's an easy
point to misread the pass as being "integral-keys-only," which it is not.

```cpp
// src/optimizer/join_filter_pushdown_optimizer.cpp:45-109
struct JoinFilterExpressionPath {
	ColumnBinding binding;
	JoinFilterPushdownMode mode = JoinFilterPushdownMode::RECONSTRUCT_EXPRESSION;
	vector<RuntimeFilterCastStep> casts;
};

static bool PushdownJoinFilterExpressionInternal(const Expression &expr, JoinFilterExpressionPath &path) {
	const auto &return_type = expr.GetReturnType();
	if (return_type.IsNested() && return_type.id() != LogicalTypeId::VARIANT) {
		// nested columns are not supported for pushdown
		return false;
	}
	if (return_type.id() == LogicalTypeId::INTERVAL) {
		// interval is not supported for pushdown
		return false;
	}
	switch (expr.GetExpressionClass()) {
	case ExpressionClass::BOUND_COLUMN_REF: {
		// column-ref - pass through the new column binding
		auto &colref = expr.Cast<BoundColumnRefExpression>();
		path.binding = colref.Binding();
		return true;
	}
	case ExpressionClass::BOUND_FUNCTION: {
		if (!BoundCastExpression::IsCast(expr)) {
			auto &function_expr = expr.Cast<BoundFunctionExpression>();
			if (function_expr.Function().GetName() != "variant_normalize" || function_expr.GetChildren().size() != 1) {
				return false;
			}
			return PushdownJoinFilterExpressionInternal(*function_expr.GetChildren()[0], path);
		}
		// We allow pushing through integral casts and integral/VARIANT casts.
		const auto &bound_cast = expr.Cast<BoundFunctionExpression>();
		const auto &src = BoundCastExpression::Child(bound_cast).GetReturnType();
		const auto &tgt = bound_cast.GetReturnType();
		const bool integral_cast = IsJoinFilterPushdownIntegralCast(src, tgt);
		const bool variant_integral_cast = IsJoinFilterPushdownVariantIntegralCast(src, tgt);
		if (!integral_cast && !variant_integral_cast) {
			return false;
		}
		if (!PushdownJoinFilterExpressionInternal(BoundCastExpression::Child(bound_cast), path)) {
			return false;
		}
		if (variant_integral_cast) {
			if (tgt.id() == LogicalTypeId::VARIANT) {
				path.mode = JoinFilterPushdownMode::STORAGE_ONLY;
				path.casts.clear();
			} else if (path.mode == JoinFilterPushdownMode::RECONSTRUCT_EXPRESSION) {
				path.casts.emplace_back(tgt, BoundCastExpression::IsTryCast(bound_cast)
				                                 ? RuntimeFilterCastMode::TRY_CAST
				                                 : RuntimeFilterCastMode::DEFAULT_CAST);
			}
			return true;
		}
		if (path.mode == JoinFilterPushdownMode::RECONSTRUCT_EXPRESSION) {
			path.casts.emplace_back(tgt, BoundCastExpression::IsTryCast(bound_cast)
			                                 ? RuntimeFilterCastMode::TRY_CAST
			                                 : RuntimeFilterCastMode::DEFAULT_CAST);
		}
		return true;
	}
	default:
		return false;
	}
}
```

This is a recursive walk *down* an expression tree, one hop at a time,
stopping the instant it hits anything other than "a plain column reference"
or "a cast (or a `variant_normalize` call) wrapping something this same
function already accepted." A bare `BOUND_COLUMN_REF` is the base case and
always succeeds, at any type — nothing here restricts column type. A
`BOUND_FUNCTION` is accepted only if it's a cast (or the single-argument
`variant_normalize` helper), and only if that specific cast is either a
plain integral-to-integral cast or a VARIANT-to/from-integral cast; anything
else (a cast to `VARCHAR`, a cast through `DOUBLE`, a genuinely computed
expression like `x + 1`) returns `false` and the whole condition is dropped
from consideration — not degraded, dropped. This is the mechanism, not a
heuristic: a query whose join key involves, say, string concatenation on the
probe side structurally cannot reach the cast-crossing branch at all, so
there's no "unsafe cast" case to worry about for it in the first place —
§7.24.6's cast-safety analysis only ever has to reason about the narrow
integral (and VARIANT-integral) space this function admits.

Each accepted cast pushes a `RuntimeFilterCastStep` (target type + cast mode)
onto `path.casts`, in evaluation order, unless the cast is a VARIANT-narrowing
one, in which case `path.mode` flips to `STORAGE_ONLY` and any previously
recorded casts are discarded — §7.24.8's VARIANT test walks a concrete case
of this switch. By the time the recursion bottoms out at the original
`BOUND_COLUMN_REF`, `path` holds: which column, in what order to re-apply
which casts to get from that column's stored value back to the value the
join condition actually compares, and whether that reconstruction is even
considered safe to attempt (`RECONSTRUCT_EXPRESSION`) or must fall back to a
cruder, storage-domain-only comparison (`STORAGE_ONLY`).

**Step 4 — set up the min/max aggregate expressions, one pair per pushdown
column, bound against the *build*-side (RHS) expression:**

```cpp
// src/optimizer/join_filter_pushdown_optimizer.cpp:411-437
const auto compute_aggregates_anyway =
    join.join_type == JoinType::INNER && join.conditions.size() == 1 && pushdown_info->join_condition.size() == 1 &&
    join.conditions[0].IsComparison() && join.conditions[0].GetComparisonType() == ExpressionType::COMPARE_EQUAL &&
    TypeIsIntegral(join.conditions[0].GetRHS().GetReturnType().InternalType());
if (pushdown_info->probe_info.empty() && !compute_aggregates_anyway) {
	// no table sources found in which we can push down filters
	return;
}

// set up the min/max aggregates for each of the filters
auto &context = optimizer.GetContext();
for (auto &join_condition : pushdown_info->join_condition) {
	for (const auto &aggr_name : {MinFun::Name, MaxFun::Name}) {
		FunctionBinder function_binder(context);
		vector<unique_ptr<Expression>> aggr_children;
		aggr_children.push_back(join.conditions[join_condition].GetRHS().Copy());
		// the one-argument overload of the min/max set is the plain aggregate, min(x, n) takes two
		auto aggr = GetBuiltinAggregateFunction(context, aggr_name, {aggr_children[0]->GetReturnType()});
		auto aggr_expr = function_binder.BindAggregateFunction(std::move(aggr), std::move(aggr_children), nullptr,
		                                                       AggregateType::NON_DISTINCT);
		if (aggr_expr->GetChildren().size() != 1) {
			// min/max with collation - not supported
			return;
		}
		pushdown_info->min_max_aggregates.push_back(std::move(aggr_expr));
	}
}
```

`compute_aggregates_anyway` is a genuinely separate reason to still build the
min/max aggregates even when **no** table scan was found to push a filter
into: an `INNER` join with a single integral equality condition might still
qualify for a **perfect hash join** (a collision-free, dense-array hash table
keyed directly by the integer value, covered in Chapter 9), and that decision
(`CanUsePerfectHashJoin`, back in `physical_hash_join.cpp:279-289`) itself
needs the build side's min/max — so this pass computes those aggregates
purely to keep that door open, independent of whether a *probe-side filter*
will ever exist. This is the one place `JoinFilterPushdownInfo` can end up
attached to a join with an *empty* `probe_info` list but a *non-empty*
`min_max_aggregates` list — worth remembering when reading §7.24.4's Sink/
Finalize code, which is written to handle exactly that case.

The `if (aggr_expr->GetChildren().size() != 1) return;` guard is a second,
narrower safety valve: DuckDB's bound `MIN`/`MAX` aggregate functions
sometimes bind with a second, hidden "collation" child when the column has a
non-default collation attached — and the comment says outright, this pass
gives up entirely rather than try to reason about a collated min/max.
§7.24.11's `nop_collation` test is the one case in this chapter's test list
that specifically confirms the *opposite*, common case still works: the
**NOP** ("no-op"/binary) collation binds a plain, one-child `MIN`/`MAX` like
any uncollated comparison, so it sails through this check.

**Step 5 — attach the pushdown info to every `LogicalGet` target found, and
record whether the build side itself is already filtered:**

```cpp
// src/optimizer/join_filter_pushdown_optimizer.cpp:389-406, 438-444
// recurse the query tree to find the LogicalGets in which we can push the filter info
vector<PushdownFilterTarget> pushdown_filter_targets;
GetPushdownFilterTargets(*join.children[0], pushdown_columns, pushdown_filter_targets, true);
for (auto &target : pushdown_filter_targets) {
	auto &get = target.get;
	// pushdown info can be applied to this LogicalGet - push the dynamic table filter set
	if (!get.dynamic_filters) {
		get.dynamic_filters = make_shared_ptr<DynamicTableFilterSet>();
	}

	JoinFilterPushdownFilter get_filter;
	get_filter.dynamic_filters = get.dynamic_filters;
	get_filter.columns = std::move(target.columns);
	if (get.has_estimated_cardinality) {
		get_filter.estimated_scan_cardinality = get.estimated_cardinality;
	}
	pushdown_info->probe_info.push_back(std::move(get_filter));
}
// ...
if (!pushdown_info->probe_info.empty()) {
	const idx_t child_idx = join.children[1]->type == LogicalOperatorType::LOGICAL_DELIM_GET ? 0 : 1;
	pushdown_info->build_side_has_filter = IsFiltering(join.children[child_idx]);
}

// set up the filter pushdown in the join itself
join.filter_pushdown = std::move(pushdown_info);
```

`get.dynamic_filters = make_shared_ptr<DynamicTableFilterSet>()` is the
single most important line in this whole file to hold onto for §7.24.5:
it allocates the (initially *empty*) container that will, at execution time,
receive the actual runtime predicate — and it's a `shared_ptr` copied both
into `get_filter.dynamic_filters` (which `join.filter_pushdown` owns) and
left sitting on the `LogicalGet` itself, so that when the physical plan
generator later turns this same `LogicalGet` into a `PhysicalTableScan`
(`plan_get.cpp:236`, `cast_table_scan.dynamic_filters = op.dynamic_filters;`),
*the same underlying object* is reachable from both the hash join's
execution-time machinery and the table scan's `EXPLAIN`-rendering code.
That one shared object, still empty at this point in the optimizer, is the
entire bridge between planning and execution this pass exists to build.

`GetPushdownFilterTargets` — walking `join.children[0]`, the probe side, down
to a base-table scan through whatever intervening operators don't invalidate
the pushdown — is covered next.

## 7.24.3 `GetPushdownFilterTargets`: reaching through the probe side's own plan shape

```cpp
// src/optimizer/join_filter_pushdown_optimizer.cpp:162-324 (abridged to the cases that matter;
// case bodies other than the first ones are summarized in /* */ comments)
void JoinFilterPushdownOptimizer::GetPushdownFilterTargets(LogicalOperator &op,
                                                           vector<JoinFilterPushdownColumn> columns,
                                                           vector<PushdownFilterTarget> &targets,
                                                           bool cross_secure_views) {
	auto &probe_child = op;
	switch (probe_child.type) {
	case LogicalOperatorType::LOGICAL_LIMIT:
	case LogicalOperatorType::LOGICAL_TOP_N:
		// LIMIT/TOP_N determines which rows are part of the probe side before the join.
		// Pushing a join filter below it can change which rows survive the limit/offset.
		break;
	case LogicalOperatorType::LOGICAL_ORDER_BY:
	case LogicalOperatorType::LOGICAL_DISTINCT:
	case LogicalOperatorType::LOGICAL_CROSS_PRODUCT:
		// does not affect probe side - recurse into left child
		// FIXME: we can probably recurse into more operators here (e.g. window, unnest)
		GetPushdownFilterTargets(*probe_child.children[0], std::move(columns), targets, cross_secure_views);
		break;
	case LogicalOperatorType::LOGICAL_FILTER:
	case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
		GetPushdownFilterTargets(*probe_child.children[0], std::move(columns), targets, cross_secure_views);
		break;
	case LogicalOperatorType::LOGICAL_ANY_JOIN: {
		auto &join = probe_child.Cast<LogicalAnyJoin>();
		// Only remove left rows where that cannot introduce null-extended rows or change volatile predicates.
		if (join.join_type != JoinType::INNER || join.condition->IsVolatile()) {
			break;
		}
		GetPushdownFilterTargets(*join.children[0], std::move(columns), targets, cross_secure_views);
		break;
	}
	case LogicalOperatorType::LOGICAL_UNNEST: { /* bails if the filter targets the unnest index itself */ }
	case LogicalOperatorType::LOGICAL_EXCEPT:
	case LogicalOperatorType::LOGICAL_INTERSECT:
	case LogicalOperatorType::LOGICAL_UNION: { /* rewrites bindings per-child, recurses into each */ }
	case LogicalOperatorType::LOGICAL_GET: { /* found it - see below */ }
	case LogicalOperatorType::LOGICAL_PROJECTION: { /* rewrite through the projection's expressions */ }
	case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY: { /* only if the filter targets a GROUP BY key */ }
	case LogicalOperatorType::LOGICAL_SECURE_VIEW:
		if (!cross_secure_views) {
			break;
		}
		// a join filter enters the view as a pushed WHERE filter would, and needs no barrier: it cannot fail, as a cast
		// that can fail is a TRY_CAST that keeps the rows it cannot convert for the join
		// (PreserveRuntimeFilterCastErrors)
		GetPushdownFilterTargets(*probe_child.children[0], std::move(columns), targets, cross_secure_views);
		break;
	default:
		// unsupported child type
		break;
	}
}
```

The shape here is a whitelist, not a blacklist: every operator type not
explicitly handled falls through to `default: break;` and the search for a
pushable `LogicalGet` simply stops on that branch, silently. The two cases
worth dwelling on:

**`LOGICAL_LIMIT`/`LOGICAL_TOP_N` block recursion entirely**, with a comment
that states the hazard directly: a `LIMIT`/`OFFSET` (or a fused `TOP_N`)
determines *which* rows count as "the probe side" in the first place — the
rows a `LIMIT 5` keeps are whichever five happen to arrive first from its
child, in whatever order the plan produces them. If a dynamic filter were
pushed *below* the `LIMIT`, it could change which five rows survive to be
limited (by removing some candidates from the pool before the `LIMIT` ever
sees them), which is a different, order-dependent kind of wrongness from the
mark/single/outer cases: it isn't that the *filter's answer* is wrong, it's
that *evaluating it earlier changes something the plan's semantics depend on
the identity of*. §7.24.10 walks the exact test (`test_inner_join_filter_pushdown.test`'s
`ordered_probe`/`boolean_keys` case) that exercises this specific `break;`.

Two more entries in the whitelist need a closer look, and so does the
`cross_secure_views` parameter.

**`LOGICAL_ANY_JOIN`.** An "any join" is a join whose
condition is an arbitrary expression, not a list of comparisons. The walk
goes through one into its left child, but only when the join is
`INNER` and its condition is not volatile (a volatile expression, such as
one calling `random()`, may give a different answer each time it runs).
Why these two rules: removing left rows below an `INNER` join only removes
output rows that the outer hash join would drop anyway. Below a `LEFT`
any-join, a removed left row could turn into a NULL-extended row instead.
And if the condition is volatile, running it on fewer rows can change its
results. The test is
`test/optimizer/pushdown/join_filter_pushdown_any_join.test`. Its first
case builds an any-join with a scalar list:

```sql
-- test/optimizer/pushdown/join_filter_pushdown_any_join.test:32-37
EXPLAIN ANALYZE
SELECT sum(id)
FROM (SELECT * FROM fact WHERE list_contains((SELECT list(k) FROM keys), k)) f
JOIN days USING(d);
----
analyzed_plan	<REGEX>:.*BLOCKWISE_NL_JOIN.*Dynamic Filters:.*d IN.*
```

`list_contains(..., k)` against an uncorrelated subquery becomes an
`INNER` any-join (a `BLOCKWISE_NL_JOIN` in the physical plan) sitting
between the hash join on `d` and the Parquet scan of `fact`. The walk
passes through that join, so the `d IN (...)` filter from the two-row
`days` table reaches the scan. Later cases in the file check
that a volatile any-join condition blocks the walk, and that a
null-extending join still does.

**`LOGICAL_SECURE_VIEW`.** A *secure view*
(`CREATE SECURE VIEW`) is a view whose hidden rows must not influence
what a user can observe. Its plan contains a `LOGICAL_SECURE_VIEW`
operator that most optimizer rules treat as a wall. The walk passes
through it, but only when the `cross_secure_views` argument is
`true`. `GenerateJoinFilters` passes `true`; the default is `false`. The
header explains why other callers keep the wall
(`join_filter_pushdown_optimizer.hpp:27-28`): they act on what they find,
for example a target's estimated cardinality, and that must not depend on
rows the view hides. `TopN::PushdownDynamicFilters` (described below) is
one such caller, so a Top-N boundary filter still stops at a secure view.
The source comment explains why a join filter is safe to send inside: it
cannot raise an error, because any cast that could fail becomes a
`TRY_CAST` that keeps the row for the join to judge (§7.24.6). The tests
are `test/sql/optimizer/secure_view_join_filter_pushdown.test` (for
example, joining `big_secure` with `dim ... WHERE d.yr = 42` must not
read all 1,000,000 rows inside the view) and
`test/sql/optimizer/secure_view_join_order.test`. Join ordering can also
see through secure views; that part belongs to the join-order chapter.

**A smaller detail** sits in the `LOGICAL_GET` case below. If the
scan's table function has a `supports_pushdown_type` callback and it
declines a column, that column is skipped (`continue`) instead of being
sent to the scan. The target is dropped only if no column is left
(`accepted_columns.empty()`).

**`LOGICAL_GET` is the success case**, and it re-validates the column's type
one more time before accepting it:

```cpp
// src/optimizer/join_filter_pushdown_optimizer.cpp:238-282
case LogicalOperatorType::LOGICAL_GET: {
	// found LogicalGet
	auto &get = probe_child.Cast<LogicalGet>();
	if (!get.function.filter_pushdown) {
		// filter pushdown is not supported - no need to consider this node
		return;
	}
	get.ResolveOperatorTypes();
	const auto bindings = get.GetColumnBindings();
	vector<JoinFilterPushdownColumn> accepted_columns;
	for (auto &filter : columns) {
		if (filter.probe_column_index.table_index != get.table_index) {
			// the filter does not apply to the probe side here - bail-out
			return;
		}
		if (get.function.supports_pushdown_type) {
			auto column_id = get.GetColumnIndex(filter.probe_column_index).GetPrimaryIndex();
			if (!get.function.supports_pushdown_type(*get.bind_data, column_id)) {
				continue;
			}
		}
		// add storage type to columns
		for (idx_t i = 0; i < bindings.size(); i++) {
			if (filter.probe_column_index.column_index == bindings[i].column_index) {
				filter.storage_type = get.types[i];
				break;
			}
		}
		D_ASSERT(filter.storage_type != LogicalType::INVALID);
		const auto reconstructed_type = filter.runtime_filter_casts.empty()
		                                    ? filter.storage_type
		                                    : filter.runtime_filter_casts.back().target_type;
		if (filter.storage_type.id() == LogicalTypeId::VARIANT &&
		    filter.mode == JoinFilterPushdownMode::RECONSTRUCT_EXPRESSION &&
		    reconstructed_type.id() == LogicalTypeId::VARIANT) {
			return;
		}
		accepted_columns.push_back(std::move(filter));
	}
	if (accepted_columns.empty()) {
		return;
	}
	targets.emplace_back(get, std::move(accepted_columns));
	break;
}
```

This is where `filter.storage_type` — the type the column *actually has on
disk/in the scanned relation* — first gets recorded, and it's a distinct
concept from the type the join condition compares in: a `probe_int` table
storing `INTEGER` values, projected up to `BIGINT` for the join, has
`storage_type = INTEGER` and a recorded cast chain ending at `BIGINT`. The
`VARIANT`-storage / `VARIANT`-reconstructed-type check is the one case where
this function refuses a column *for a reason other than plan shape*: if,
after walking every cast, the value would still be reconstructed as a raw,
un-shredded `VARIANT` (rather than an extracted integral value), there's
nothing comparable to build a range filter against, and the whole target is
abandoned — this is the "mixed VARIANT" case §7.24.8's variant-cast test
exercises with an explicit `<!REGEX>` assertion.

**`LOGICAL_UNION`/`LOGICAL_EXCEPT`/`LOGICAL_INTERSECT` rewrite bindings
per-child and recurse into each branch independently** — each branch of a
`UNION ALL` is scanned by its own `LogicalGet`, possibly with its own,
different storage type (§7.24.7's `t_union_int`/`t_union_small` case has
`INT` on one branch and `SMALLINT` on the other), so the recursion has to
re-derive `filter.storage_type` and the cast chain separately for each
branch rather than reusing whatever the first branch computed.

**`IsFiltering`**, used both by `GenerateJoinFilters` (to record
`build_side_has_filter`, consulted later at Finalize time by
`CanUseBloomFilter`) and, as `15-build-side-probe-side-test-driven.md`
already showed, by `BuildProbeSideOptimizer::DynamicFilterBuildBonus`, is a
short recursive predicate:

```cpp
// src/optimizer/join_filter_pushdown_optimizer.cpp:326-346
bool JoinFilterPushdownOptimizer::IsFiltering(const unique_ptr<LogicalOperator> &op) {
	switch (op->type) {
	case LogicalOperatorType::LOGICAL_GET: {
		auto &get = op->Cast<LogicalGet>();
		return get.table_filters.HasFilters();
	}
	case LogicalOperatorType::LOGICAL_FILTER: {
		return true;
	}
	case LogicalOperatorType::LOGICAL_TOP_N: {
		return true;
	}
	default:
		for (const unique_ptr<LogicalOperator> &child : op->children) {
			if (IsFiltering(child)) {
				return true;
			}
		}
		return false;
	}
}
```

Both `IsFiltering` and `GetPushdownFilterTargets` are declared `static`/public
on the class (`join_filter_pushdown_optimizer.hpp:29-32`) specifically so
other passes can reuse them without re-implementing this traversal —
`18-topn-and-topn-window-elimination.md` §7.18.1.4 already documented the
second reuse: `TopN::PushdownDynamicFilters` calls
`JoinFilterPushdownOptimizer::GetPushdownFilterTargets` directly to install
its own, independent runtime filter (a tightening top-`N` boundary rather
than a fixed build-side range) using the exact same probe-side traversal
logic. It's the same "reach through Filter/Projection/Aggregate/SetOp down
to a `LogicalGet`" machinery serving two different runtime-filter producers.
There are two more callers: `BuildProbeSideOptimizer` (`build_probe_side_optimizer.cpp:120`,
to estimate how big a filter's target scan is) and the row-group pruner
(`row_group_pruner.cpp:138`). All three pass no `cross_secure_views`
argument, so for them a secure view still ends the walk (see
`LOGICAL_SECURE_VIEW` above).

## 7.24.4 The execution-side object, tied to Chapter 11's Sink/Combine/Finalize

A note on file layout before the walkthrough, in this book's own stated
sourcing-honesty style: the task that produced this chapter named
`src/execution/operator/join/join_filter_pushdown.cpp` as a source file to
read. **That file does not exist in this checkout.** `JoinFilterPushdownInfo`,
`JoinFilterGlobalState`, and `JoinFilterLocalState` are declared in
`src/include/duckdb/execution/operator/join/join_filter_pushdown.hpp`, but
every one of their methods (`GetGlobalState`, `GetLocalState`, `Sink`,
`Combine`, `Finalize`, `FinalizeMinMax`, `FinalizeFilters`, `PushInFilter`,
`CanUseInFilter`/`CanUseBloomFilter`/`CanUsePrefixRangeFilter`,
`DeferRuntimeFilter`, `TryRegisterPrefixRangeFilter`) is *defined* inside
`src/execution/operator/join/physical_hash_join.cpp`, interleaved directly
with `PhysicalHashJoin`'s own `Sink`/`Combine`/`Finalize`. This isn't a
version-drift correction the way a couple of others in this book have been —
there's no missing `.cpp` file to blame on a rename; the header simply
declares an API whose only client, `PhysicalHashJoin`, implements it in its
own translation unit. Every citation below is to `physical_hash_join.cpp`
for that reason.

**Header, first, for the API shape:**

```cpp
// src/include/duckdb/execution/operator/join/join_filter_pushdown.hpp:75-82, 98-115
struct JoinFilterPushdownFilter {
	//! The dynamic table filter set where to push filters into
	shared_ptr<DynamicTableFilterSet> dynamic_filters;
	//! The columns for which we should generate filters
	vector<JoinFilterPushdownColumn> columns;
	//! Target scan estimate for filter costing when intervening operators reduce the probe cardinality.
	idx_t estimated_scan_cardinality = 0;
};

// ...
struct JoinFilterPushdownInfo {
	//! The join condition indexes for which we compute the min/max aggregates
	vector<idx_t> join_condition;
	//! The probes to push the filter into
	vector<JoinFilterPushdownFilter> probe_info;
	//! Min/Max aggregates
	vector<unique_ptr<Expression>> min_max_aggregates;
	//! Whether the build side has a filter -> we might be able to push down a bloom filter into the probe side
	bool build_side_has_filter;

public:
	unique_ptr<JoinFilterGlobalState> GetGlobalState(ClientContext &context, const PhysicalOperator &op) const;
	unique_ptr<JoinFilterLocalState> GetLocalState(JoinFilterGlobalState &gstate) const;

	void Sink(DataChunk &chunk, JoinFilterLocalState &lstate) const;
	void Combine(JoinFilterGlobalState &gstate, JoinFilterLocalState &lstate) const;
	unique_ptr<DataChunk> Finalize(ClientContext &context, JoinFilterGlobalState &gstate,
	                               const PhysicalComparisonJoin &op, optional_ptr<JoinHashTable> ht = nullptr) const;
	// ...
};
```

The names line up one-to-one with the Sink/Combine/Finalize contract Chapter
11 §11.3 established for every `PhysicalOperator` that acts as a sink; this
is deliberate — `JoinFilterPushdownInfo` is not itself a `PhysicalOperator`,
it's a helper object `PhysicalHashJoin` *delegates to*, at each of the three
phases, from inside its own `Sink`/`Combine`/`Finalize` overrides. If you
haven't read Chapter 11 §11.3.4's walkthrough of `PhysicalHashJoin`'s own
Sink/Combine/Finalize (build-side hash-table construction, thread-local
tables, single-threaded finalize), the rest of this section assumes it —
this book deliberately does not re-derive Sink/Combine/Finalize's general
contract here, only the one operator-specific slice of it this pass adds.

**`GetGlobalState`/`GetLocalState` — allocated once per query execution
(global) and once per thread (local), holding a running min/max aggregate
state:**

```cpp
// src/execution/operator/join/physical_hash_join.cpp:291-302
unique_ptr<JoinFilterGlobalState> JoinFilterPushdownInfo::GetGlobalState(ClientContext &context,
                                                                         const PhysicalOperator &op) const {
	// clear any previously set filters
	// we can have previous filters for this operator in case of e.g. recursive CTEs
	for (auto &info : probe_info) {
		info.dynamic_filters->ClearFilters(op);
	}
	auto result = make_uniq<JoinFilterGlobalState>();
	result->global_aggregate_state =
	    make_uniq<GlobalUngroupedAggregateState>(BufferAllocator::Get(context), min_max_aggregates);
	return result;
}
```

```cpp
// src/execution/operator/join/physical_hash_join.cpp:605-609
unique_ptr<JoinFilterLocalState> JoinFilterPushdownInfo::GetLocalState(JoinFilterGlobalState &gstate) const {
	auto result = make_uniq<JoinFilterLocalState>();
	result->local_aggregate_state = make_uniq<LocalUngroupedAggregateState>(*gstate.global_aggregate_state);
	return result;
}
```

`GlobalUngroupedAggregateState`/`LocalUngroupedAggregateState` are the same
machinery a scalar `MIN(x)`-with-no-`GROUP BY`-clause aggregate uses
elsewhere in the engine — this pass doesn't invent a new min/max
implementation, it reuses the ordinary ungrouped-aggregate state machine,
seeded with the `min_max_aggregates` expression list §7.24.2 built (one
`MIN` and one `MAX` bound aggregate per pushdown-eligible join condition).
The `ClearFilters` call at the top exists for exactly one named reason —
recursive CTEs, where the same `PhysicalHashJoin` operator object is
`Reset()` and re-driven through another Sink/Combine/Finalize cycle; without
clearing, a stale filter computed from a *previous* iteration's build data
could linger on the shared `DynamicTableFilterSet` and be applied against
the *current* iteration's probe rows.

**`HashJoinLocalSinkState`'s constructor wires the local filter state in
alongside the local hash table**, and `HashJoinGlobalSinkState`'s
constructor does the same for the global state — both gated on `op.filter_pushdown`
being non-null (i.e., on `GenerateJoinFilters` having actually attached a
`JoinFilterPushdownInfo` to this join at optimize time):

```cpp
// src/execution/operator/join/physical_hash_join.cpp:635-637 (HashJoinLocalSinkState ctor)
if (op.filter_pushdown) {
	local_filter_state = op.filter_pushdown->GetLocalState(*gstate.global_filter_state);
}
```

**Sink — every incoming build-side chunk feeds the min/max aggregate,
alongside (not instead of) the ordinary hash-table insert:**

```cpp
// src/execution/operator/join/physical_hash_join.cpp:885-894
void JoinFilterPushdownInfo::Sink(DataChunk &chunk, JoinFilterLocalState &lstate) const {
	// if we are pushing any filters into a probe-side, compute the min/max over the columns that we are pushing
	for (idx_t pushdown_idx = 0; pushdown_idx < join_condition.size(); pushdown_idx++) {
		auto join_condition_idx = join_condition[pushdown_idx];
		for (idx_t i = 0; i < 2; i++) {
			idx_t aggr_idx = pushdown_idx * 2 + i;
			lstate.local_aggregate_state->Sink(chunk, join_condition_idx, aggr_idx, chunk.size());
		}
	}
}
```

...called from exactly the line Chapter 11 §11.3.4 already reproduced as
`PhysicalHashJoin::Sink`'s body, one step before the actual hash-table insert:

```cpp
// src/execution/operator/join/physical_hash_join.cpp:896-928 (elided to the filter-pushdown line)
SinkResultType PhysicalHashJoin::Sink(ExecutionContext &context, DataChunk &chunk, OperatorSinkInput &input) const {
	auto &gstate = input.global_state.Cast<HashJoinGlobalSinkState>();
	auto &lstate = input.local_state.Cast<HashJoinLocalSinkState>();

	// resolve the join keys for the right chunk
	lstate.join_keys.Reset();
	lstate.join_key_executor.Execute(chunk, lstate.join_keys);

	if (filter_pushdown && !gstate.skip_filter_pushdown) {
		filter_pushdown->Sink(lstate.join_keys, *lstate.local_filter_state);
	}
	// ... payload handling, lazy layout publication, then:
	// build the HT
	lstate.hash_table->Build(lstate.append_state, lstate.join_keys, lstate.payload_chunk);

	return SinkResultType::NEED_MORE_INPUT;
}
```

Every one of Chapter 11's guarantees about `Sink` apply unmodified: this runs
once per incoming chunk, potentially concurrently across every thread
scanning the build side, and — because `local_aggregate_state` is per-thread
(from `GetLocalState`, called once per `HashJoinLocalSinkState`) — no
locking is needed here, exactly as no locking was needed for each thread's
private hash table in Chapter 11's version of this same function. This is
**the Sink phase**: the min/max running totals are per-thread and partial,
not yet combined, and absolutely not yet a usable filter.

**Combine — one thread's partial min/max gets merged into the shared,
global aggregate state, under the same lock guarding the hash-table
handoff:**

```cpp
// src/execution/operator/join/physical_hash_join.cpp:930-932
void JoinFilterPushdownInfo::Combine(JoinFilterGlobalState &gstate, JoinFilterLocalState &lstate) const {
	gstate.global_aggregate_state->Combine(*lstate.local_aggregate_state);
}
```

Called from `PhysicalHashJoin::Combine` — though notice it is *not* called
from inside the `annotated_lock_guard` block Chapter 11 §11.3.4 quoted;
`JoinFilterPushdownInfo::Combine` isn't shown taking that lock either, which
means the underlying `GlobalUngroupedAggregateState::Combine` call must
itself be safe to call concurrently across threads (the ordinary
ungrouped-aggregate combine path is built for exactly this — many threads
merging into one shared running total is the normal parallel-aggregation
case, not something specific to this pass). This is **the Combine phase**:
by the time every thread's `Combine` has run, the *global* aggregate state
holds the true min and true max across the *entire* build side — but it is
still sitting inside an aggregate-state object, not yet extracted into a
`Value`, and — critically — not yet installed as a filter anywhere a scan
could read it.

**Finalize — the moment the running aggregate becomes a concrete range, and
the moment a filter actually gets installed:**

```cpp
// src/execution/operator/join/physical_hash_join.cpp:1598-1609
unique_ptr<DataChunk> JoinFilterPushdownInfo::FinalizeMinMax(JoinFilterGlobalState &gstate) const {
	// finalize the min/max aggregates
	vector<LogicalType> min_max_types;
	for (auto &aggr_expr : min_max_aggregates) {
		min_max_types.push_back(aggr_expr->GetReturnType());
	}
	auto final_min_max = make_uniq<DataChunk>();
	final_min_max->Initialize(Allocator::DefaultAllocator(), min_max_types);

	gstate.global_aggregate_state->Finalize(*final_min_max);
	return final_min_max;
}
```

`GlobalUngroupedAggregateState::Finalize` is the aggregate-state machine's
own terminal step — the same call any plain, no-`GROUP BY` `SELECT MIN(x)`
would make to turn its running state into an actual output row. Here, that
output row *is* the build side's true min and true max, as of the exact
instant every build-side thread's Sink and Combine have completed. This
function is called from exactly one place — `PhysicalHashJoin::Finalize`
— and `PhysicalHashJoin::Finalize` is, per Chapter 11 §11.3.4, single-threaded
and guaranteed to run only after every thread's `Combine` has returned. The
min/max values this function produces are therefore, provably, the complete,
final answer — not a partial or racy snapshot.

`FinalizeFilters` (`physical_hash_join.cpp:1876-2006`, the pass's largest
function) is what turns that `DataChunk` of raw min/max values into an
actual, installed predicate. The single-value case (build side collapsed to
one distinct key) and the range case are handled differently, and — in this
checkout — the range case has grown considerably beyond a plain min/max
comparison:

```cpp
// src/execution/operator/join/physical_hash_join.cpp:1931-2002 (abridged)
if (Value::NotDistinctFrom(min_val, max_val)) {
	// min = max - single value: a one-sided comparison filter
	auto filter_expr = CreateJoinFilterComparisonExpression(context, pushdown_column, cmp, min_val, ...);
	if (filter_expr) {
		info.dynamic_filters->PushFilter(op, filter_col_idx, make_uniq<ExpressionFilter>(std::move(filter_expr)));
	}
} else {
	// min != max
	if (cmp != ExpressionType::COMPARE_EQUAL) {
		CreateDynamicMinMaxFilters(op, info, context, pushdown_column, filter_col_idx, cmp, min_val, max_val, ...);
		continue;
	}
	// equality: try an exact IN-list filter, then a prefix-range/Bloom filter, falling back to min/max
	bool pushed_in_filter = false;
	if (CanUseInFilter(context, ht, cmp)) {
		pushed_in_filter = PushInFilter(context, info, pushdown_column, *ht, op, filter_idx, filter_col_idx);
	}
	if (can_emit_prf && span < SMALL_EXACT_PRF_BITS && TryRegisterPrefixRangeFilter(...)) { continue; }
	if (can_emit_prf) { /* larger, Bloom-sized prefix-range filter */ }
	if (!pushed_in_filter) {
		CreateDynamicMinMaxFilters(op, info, context, pushdown_column, filter_col_idx, cmp, min_val, max_val, ...);
	}
	if (allow_bloom_filters && can_emit_runtime_filters && ht && gstate &&
	    CanUseBloomFilter(info, context, op, cmp, ht)) {
		if (ht->HasPublishableBloomFilter()) {
			// a complete filter was accumulated during Sink (external joins build it there)
			DeferRuntimeFilter(DeferredRuntimeFilterType::BLOOM_FILTER, op, info, pushdown_column,
			                   filter_col_idx, *gstate);
		} else if (allow_prefix_range_filters) {
			// like a prefix-range filter, a bloom filter may be built during finalize when the
			// finalize materializes the whole build side - an external finalize does not
			ht->SetBuildBloomFilter(true);
			DeferRuntimeFilter(DeferredRuntimeFilterType::BLOOM_FILTER, op, info, pushdown_column,
			                   filter_col_idx, *gstate);
		}
	}
}
```

The book's task framing (and DiDi's own slide [10], echoed in the overview's
§7.6.5) describes this pass as a **min/max range filter**, and that remains
the mechanism's conceptual core and the one every test in §7.24.7–7.24.8
actually observes — but this checkout's `FinalizeFilters` shows the current
source has generalized considerably beyond it: for an equality join key with
a small enough build side, `CanUseInFilter` (`physical_hash_join.cpp:1472-1476`,
gated by the `DynamicOrFilterThresholdSetting`) tries an **exact IN-list**
filter (scanning the build side's actual key column via `PushInFilter`,
`:1784-1834`, and installing a literal `x IN (v1, v2, ...)` predicate — a
strictly tighter filter than any range could be, when the build side has
few enough distinct values). Failing that, `CanUsePrefixRangeFilter`/
`TryRegisterPrefixRangeFilter` build a **bit-vector prefix-range filter**
(a range-of-bits sketch, sized either exactly or to a Bloom-filter-sized
budget depending on how wide the key's span is) — and separately,
`CanUseBloomFilter` may additionally register a genuine **Bloom filter**
(`DeferRuntimeFilter`), published once the hash table itself finishes
building via `PublishDeferredRuntimeFilters`
(`physical_hash_join.cpp:1698-1726`). Only when none of the pricier options
apply does the code fall back to the plain two-sided `>=min AND <=max`
comparison (`CreateDynamicMinMaxFilters`, `:1836-1870`) this chapter's tests
actually exercise end to end. This chapter follows the task's framing and
keeps the running example to the min/max case throughout, but it would be
inaccurate to leave the impression that min/max is the *only* mechanism live
in the current source — it is the baseline every fancier filter degrades to
when its own preconditions aren't met.

`PhysicalHashJoin::Finalize` calls this in two different places depending on
whether the join went **external** (spilled to disk — hash table too large
for the memory budget) or stayed **in-memory**:

```cpp
// src/execution/operator/join/physical_hash_join.cpp:1353-1369 (external path)
//! Computes and registers the runtime filters of an external join. Both external paths call this exactly once,
//! after the whole build side has been merged into the global hash table and before the finalize event is
//! scheduled. Every input is accumulated during Sink, so it does not matter which partitions are materialized.
static void FinalizeExternalRuntimeFilters(ClientContext &context, const PhysicalHashJoin &op,
                                           HashJoinGlobalSinkState &sink) {
	if (!op.filter_pushdown || sink.skip_filter_pushdown) {
		return;
	}
	auto &ht = *sink.hash_table;
	if (ht.GetSinkCollection().Count() == 0) {
		return;
	}
	auto filter_min_max = op.filter_pushdown->FinalizeMinMax(*sink.global_filter_state);
	// prefix-range filters are built from the materialized hash table, which only ever holds one round
	op.filter_pushdown->FinalizeFilters(context, op, std::move(filter_min_max), &ht, true, false,
	                                    sink.global_filter_state.get());
}
```

```cpp
// src/execution/operator/join/physical_hash_join.cpp:2092-2127 (in-memory path, abridged)
Value min;
Value max;
unique_ptr<DataChunk> filter_min_max = nullptr;

if (filter_pushdown && !sink.skip_filter_pushdown && ht.Count() > 0) {
	filter_min_max = filter_pushdown->FinalizeMinMax(*sink.global_filter_state);
	min = filter_min_max->data[0].GetValue(0);
	max = filter_min_max->data[1].GetValue(0);
} else if (TypeIsIntegral(conditions[0].GetRHS().GetReturnType().InternalType())) {
	// ...
}
// ... perfect-hash-join decision uses min/max here too ...
if (filter_min_max) {
	filter_pushdown->FinalizeFilters(context, *this, std::move(filter_min_max), &ht, !use_perfect_hash, true,
	                                 sink.global_filter_state.get());
	if (use_perfect_hash) {
		ht.BuildPrefixRangeFilter();
		PublishDeferredRuntimeFilters(context, ht, *sink.global_filter_state);
	}
}
```

The last two `bool` arguments of `FinalizeFilters` are `allow_bloom_filters`
and `allow_prefix_range_filters`. Two details worth pulling out: the
external path disables prefix-range filters outright (they're only offered
when the whole build side is known in-memory at once), and the in-memory
path disables Bloom filters specifically when a **perfect hash join** ends
up being used — a perfect hash join already gives collision-free,
dense-array probing directly keyed by the integer value, so a probabilistic
Bloom pre-filter in front of it would be pure overhead with nothing left to
save.

Bloom filters for **external** joins need special care. An external join
builds its hash table in rounds: each round loads only some partitions into
memory. A Bloom filter filled during finalize would only see the partitions
of the *first* round. Probe rows whose key lived in a later round's
partition would fail the filter and be silently dropped (issue #25702). So
the Bloom filter is built in a different place, by this rule:

- An **in-memory** join builds its Bloom filter during finalize,
  which then sees the whole build side. That is the
  `else if (allow_prefix_range_filters)` branch above: only the in-memory
  path passes `true` there.
- A join that is **likely to go external** fills a Bloom filter during
  `Sink`, where every build row passes exactly once. "Likely external"
  means the estimated build size is more than a quarter of the memory
  budget, or `debug_force_external` is set
  (`JoinIsLikelyExternal`, `physical_hash_join.cpp:394-405`, with
  `LIKELY_IN_MEMORY_BUILD_MEMORY_FRACTION = 4` at `:391`;
  `ShouldBuildBloomFilterInSink`, `:412-436`; and
  `InitializeBloomFilter`, `:479-488`). `ShouldBuildBloomFilterInSink`
  also repeats the estimate-based checks of `CanUseBloomFilter`, since
  the real build count is not known yet. All threads insert into one
  shared filter (`HashJoinLocalSinkState`'s constructor calls
  `hash_table->SetBloomFilter(gstate.hash_table->GetBloomFilter())`).
- A filter is published only if it is *sealed* (complete) and still
  useful. `CreateRuntimeFilterExpression` checks
  `ht.HasPublishableBloomFilter()` and returns no filter otherwise
  (`physical_hash_join.cpp:1668-1672`). So a wrong guess is safe both
  ways: a join that spills unexpectedly runs without a Bloom filter, and
  a join that unexpectedly stays in memory publishes the complete
  sink-built filter.

The regression test is
`test/sql/outofcore/out_of_core_hash_join_bloom_filter_pushdown.test_slow`:
two 3,000,000-row tables with 25 distinct keys, `memory_limit='60MB'` and
one thread, so the join spills. `SELECT count(*) FROM t WHERE a IN (SELECT
b FROM u)` must return all `3000000` rows.

The cost check for Bloom and prefix-range filters looks at the *target
scan*, not only at the join's direct probe child.
`CanUseBloomFilter` compares the build size with the probe size and gives
up when the build side is larger (`BUILD_TO_PROBE_RATIO_THRESHOLD = 1.0`,
`physical_hash_join.cpp:386`). The problem: a selective operator between
the join and the scan (say, a semi join) can make the probe child's
estimate small, even though the filter will run on the much bigger scan
below it. So the optimizer copies the target scan's estimate into the
field `JoinFilterPushdownFilter::estimated_scan_cardinality`, and
`CanUseBloomFilter` uses the larger of the two numbers, but only when the
build side is itself filtered:

```cpp
// src/execution/operator/join/physical_hash_join.cpp:1507-1511
	auto probe_cardinality = op.children[0].get().estimated_cardinality;
	if (build_side_has_filter) {
		probe_cardinality = MaxValue(info.estimated_scan_cardinality, probe_cardinality);
	}
	const double build_to_probe_ratio = static_cast<double>(build_count) / static_cast<double>(probe_cardinality);
```

`test/optimizer/pushdown/join_filter_target_cardinality.test` shows the
effect. `probe p SEMI JOIN gate g` leaves an estimated few hundred rows,
fewer than the 8,000 build rows that pass `WHERE b.keep`. With the target
scan's 20,000 rows in the ratio, the plan shows `k IN PRF(k)` (a
prefix-range filter) or, for sparse keys, `k IN BF(k)` (a Bloom filter).
Without `WHERE b.keep` the build side is unfiltered, only the probe child's
estimate counts, and neither filter appears.

Either way, by the time
`Finalize` returns, `get.dynamic_filters` — the very object §7.24.2 allocated
empty at optimize time — has had `PushFilter` called on it, and is no longer
empty. **This is the phase boundary this whole chapter has been building
toward: before this line runs, `dynamic_filters->HasFilters()` is false;
after it, it's true, on the exact same object the probe side's table scan
holds a `shared_ptr` to.** §7.24.5 is entirely about what that flip means for
`EXPLAIN`.

## 7.24.5 Why `EXPLAIN` and `EXPLAIN ANALYZE` genuinely differ here

Chapter 2 §2.5 established the general mechanism precisely: `EXPLAIN` stops
after the physical plan generator and renders the `PhysicalOperator` tree
without ever executing it; `EXPLAIN ANALYZE` executes the query to
completion first, then renders the *same kind* of tree, annotated with
per-node timing and row counts collected by the `OperatorProfiler` machinery
bracketing every `Sink`/`Execute`/`GetData` call. For most of the forty
passes in this chapter, that distinction only changes the *numbers*
attached to a plan whose *shape and extra_info* were already fully decided
before a single row moved — an `EXPLAIN` of a query with a pushed-down
static filter already shows that filter's `Filters: ...` text, because the
`LogicalGet`'s `table_filters` were finalized by `FILTER_PUSHDOWN` (pass #6)
long before physical-plan generation, let alone execution, ever happens.

`JOIN_FILTER_PUSHDOWN` breaks that pattern, and the mechanism is a single,
literal `if` inside `PhysicalTableScan::ParamsToString`, the function that
supplies a table scan's own portion of an `EXPLAIN` tree's `extra_info`:

```cpp
// src/execution/operator/scan/physical_table_scan.cpp:361-367
if (function.filter_pushdown && table_filters) {
	result["Filters"] = GetFilterInfo(*table_filters);
}

if (function.filter_pushdown && dynamic_filters && dynamic_filters->HasFilters()) {
	result["Dynamic Filters"] = GetFilterInfo(*dynamic_filters->GetFinalTableFilters(*this, nullptr));
}
```

`table_filters` (the `"Filters"` row) is a plain member, set once at physical
plan construction and never touched again — it's exactly as static as every
other pass's output, and `EXPLAIN` and `EXPLAIN ANALYZE` render it
identically. `dynamic_filters` (the `"Dynamic Filters"` row) is the
`shared_ptr<DynamicTableFilterSet>` traced all the way back to
`join_filter_pushdown_optimizer.cpp:396`'s `make_shared_ptr<DynamicTableFilterSet>()`
and copied, unchanged, into the `PhysicalTableScan` at
`plan_get.cpp:236` (`cast_table_scan.dynamic_filters = op.dynamic_filters;`).
Its `HasFilters()` (`table_filter_set.cpp:472-475`) is a plain, lock-guarded
check of whether anything has ever called `PushFilter` on it —

```cpp
// src/planner/table_filter_set.cpp:472-475
bool DynamicTableFilterSet::HasFilters() const {
	lock_guard<mutex> l(lock);
	return !filters.empty();
}
```

— and, per §7.24.4, the **only** call to `PushFilter` on this object happens
inside `JoinFilterPushdownInfo::FinalizeFilters`, which is only ever reached
from `PhysicalHashJoin::Finalize`, which is only ever reached by the pipeline
scheduler once the build side's `Sink`/`Combine` phases have actually run
against real data. **A plain `EXPLAIN` never executes the query, so
`PhysicalHashJoin::Finalize` never runs, so `PushFilter` is never called, so
`dynamic_filters->HasFilters()` is false when `ParamsToString` checks it, so
the `"Dynamic Filters"` row is never added to the tree at all** — not
merely present-but-empty, structurally absent, because the surrounding `if`
guards the entire map insertion. `EXPLAIN ANALYZE`, by contrast, drives the
query to completion first (Chapter 2 §2.5's own words: "*does* execute the
query — fully, to completion") and *then* calls the same `TreeRenderer`
machinery against the *same, already-mutated* `PhysicalOperator` objects —
by the time `ParamsToString` runs, `PhysicalHashJoin::Finalize` has already
called `PushFilter` on this exact `DynamicTableFilterSet`, so `HasFilters()`
is now true, and the row appears with the real, concrete filter expression's
own `ToString()` rendering as its value.

This is precisely the same mechanism every `join_filter_pushdown_cast*.test`
file and the `nop_collation` test check directly — see §7.24.7's
test-format note — and it is worth stating the contrast with Chapter 2's own
illustration explicitly, in the spirit of this book's stated policy of
flagging discrepancies rather than smoothing over them. Chapter 2 §2.6's
worked `EXPLAIN ANALYZE` tree for Query A shows the join-filter information
attached directly to the `HASH_JOIN` node itself, as `Build Min: 0` /
`Build Max: 99999` fields — but that illustration was generated, by its own
stated caveat, against a locally installed **stable v1.1.3** build, "since
the `v2.0-cyanoptera` checkout examined elsewhere in this book was not
compiled in this pass." A search of this checkout's entire `src/` tree for
the literal strings `"Build Min"` or `"Build Max"` finds nothing — and
`PhysicalHashJoin::ParamsToString` (`physical_hash_join.cpp:2923-2948`) emits
only `"Join Type"`, `"Conditions"`, and the estimated cardinality, with no
runtime-filter field at all. In this checkout, the *only* place this pass's
runtime effect surfaces textually in an `EXPLAIN`-family tree is the probe
side's own `TABLE_SCAN` node, as the `"Dynamic Filters"` row shown above —
not the join node. Whether v1.1.3 attached this information to the join
node as a since-removed feature, or whether the "Build Min"/"Build Max"
labels were themselves this book's own illustrative simplification of an
older, differently-shaped mechanism, this checkout's source doesn't say —
flagged here, per this book's policy, rather than guessed at. What *is*
certain, and directly verifiable in both versions' architecture, is the
underlying claim both the overview and this chapter make: the information is
runtime-derived, and a plan rendered before execution structurally cannot
carry it.

## 7.24.6 `runtime_filter_cast.cpp`: making a probe/build type mismatch safe

§7.24.2 showed `PushdownJoinFilterExpressionInternal` recording a chain of
`RuntimeFilterCastStep`s whenever the probe-side expression is a cast (or
chain of casts) over a plain column. `runtime_filter_cast.cpp` is the
(small — 86 lines) file that decides, given such a chain, whether it's safe
to trust *directly*, or whether it must be re-evaluated, per row, at
execution time with the engine's ordinary cast machinery. The load-bearing
function is `RuntimeFilterCastCanFail`:

```cpp
// src/optimizer/runtime_filter_cast.cpp:9-30
bool RuntimeFilterCastUtil::RuntimeFilterCastCanFail(const LogicalType &source_type, const LogicalType &target_type) {
	if (source_type == target_type) {
		return false;
	}
	if (source_type.id() == LogicalTypeId::VARIANT || target_type.id() == LogicalTypeId::VARIANT ||
	    !source_type.IsIntegral() || !target_type.IsIntegral()) {
		return true;
	}

	const auto source_size = GetTypeIdSize(source_type.InternalType());
	const auto target_size = GetTypeIdSize(target_type.InternalType());
	if (source_size > target_size) {
		return true;
	}
	if (source_type.IsSigned() == target_type.IsSigned()) {
		return false;
	}
	if (source_type.IsSigned()) {
		return true;
	}
	return source_size >= target_size;
}
```

Read as a decision table, this is exactly "is this integral cast a provably
total, order-preserving injection?":

| source → target | same signedness | signed → unsigned | unsigned → signed |
|---|---|---|---|
| **widening** (source strictly smaller) | safe (`false`) | unsafe (`true`) — a negative source value has no unsigned image | safe (`false`) only if strictly wider (`source_size >= target_size` is `false`) |
| **same size** | safe (`false`) | unsafe | unsafe (`source_size >= target_size` is `true`) |
| **narrowing** (source strictly larger) | unsafe (`true`, caught by the size check before signedness is even considered) | unsafe | unsafe |

Any cast involving `VARIANT`, or either side non-integral, is conservatively
`true` (can fail) unconditionally — this function never tries to reason
about whether a `VARIANT`-typed value's dynamic content happens to be safe;
that reasoning, per §7.24.2, is handled entirely upstream by
`PushdownJoinFilterExpressionInternal` refusing to record a cast chain at
all for anything outside the integral/VARIANT-integral space in the first
place. Two independent layers of defense, not one: the optimizer only ever
lets an integral-to-integral or VARIANT-to/from-integral cast *into* the
cast-chain machinery to begin with (§7.24.2's `IsJoinFilterPushdownIntegralCast`/
`IsJoinFilterPushdownVariantIntegralCast`), and, only within that already
narrowed space, `RuntimeFilterCastCanFail` further distinguishes the subset
that's provably safe to trust blindly from the subset that must be
re-verified per row.

**What "can fail" actually gates**, downstream:

```cpp
// src/optimizer/runtime_filter_cast.cpp:67-84
unique_ptr<Expression> RuntimeFilterCastUtil::CreateRuntimeFilterInputExpression(ClientContext &context,
                                                                                 const JoinFilterPushdownColumn &column,
                                                                                 bool &preserves_cast_errors) {
	unique_ptr<Expression> input = make_uniq<BoundReferenceExpression>(column.storage_type, idx_t(0));
	auto source_type = column.storage_type;
	for (auto &cast : column.runtime_filter_casts) {
		const auto cast_can_fail = RuntimeFilterCastCanFail(source_type, cast.target_type);
		const auto is_try_cast = cast.mode == RuntimeFilterCastMode::TRY_CAST || cast_can_fail;
		if (source_type != cast.target_type) {
			input = BoundCastExpression::AddCastToType(context, std::move(input), cast.target_type, is_try_cast);
		}
		preserves_cast_errors |= cast.mode == RuntimeFilterCastMode::DEFAULT_CAST && cast_can_fail;
		source_type = cast.target_type;
	}
	return input;
}
```

This function does not translate the build side's min/max *values* across
the type boundary by arithmetic — it never, anywhere, computes "what would
this BIGINT max value be, numerically, in INTEGER space." Instead it builds
a literal `CAST(...)` **expression**, re-applying the exact same cast chain
the original join condition used, rooted at a fresh reference to the raw
`storage_type` scan value — and hands that expression to the ordinary
`ExpressionExecutor` to run, per probe row, at scan time, comparing its
result against the (already-in-the-post-cast-domain) min/max constants.
Correctness is delegated entirely to DuckDB's existing, independently
correct cast implementation, not to any bespoke bounds-translation
arithmetic this pass would otherwise have to get right on its own. When
`cast_can_fail` is true, `is_try_cast` is forced true regardless of the
originally recorded cast mode — a `TRY_CAST` returns `NULL` for an
out-of-range value instead of raising an error, so a probe row whose raw
value simply doesn't fit the comparison domain (like the cast test's
`9999999999`-vs-`INT32` cases) is treated as "doesn't match the filter, drop
it" rather than crashing the whole query over a row the filter itself was
only ever supposed to prune, not evaluate authoritatively.

**The one correctness hazard this machinery exists specifically to avoid** —
and the direct answer to "what could go wrong with a non-monotone/failing
cast" — is *not* that the filter would silently give the wrong numeric
answer (DuckDB's cast implementations don't do C-style bit-reinterpreting
wraparound; an out-of-range numeric cast always either succeeds with the
correct value or fails/returns NULL, never silently wraps to an unrelated
number). The real hazard is the same one `07-optimizer/00-overview.md` §7.6.1
already named for `REORDER_FILTER`'s `CanThrow()` guard: **an optimization
must never change which rows cause the query to raise an error.** If the
*original*, un-pushed-down join condition uses a `DEFAULT` (non-try) cast
that would legitimately throw a `Conversion Error` for some probe row — and
the runtime filter's own reconstruction of that same cast chain used a
silent `TRY_CAST` instead and just quietly dropped that row as
"non-matching" — the query would go from *correctly erroring* to *silently
returning fewer rows*, which is a strictly worse outcome than either
"correct" or "loudly wrong." `CreateRuntimeFilterInputExpression`'s
`preserves_cast_errors` output flag exists to prevent exactly this:

```cpp
// src/execution/operator/join/physical_hash_join.cpp:1561-1571
static unique_ptr<Expression> PreserveRuntimeFilterCastErrors(unique_ptr<Expression> filter_expr,
                                                              unique_ptr<Expression> null_check_input) {
	if (!null_check_input) {
		return filter_expr;
	}
	// A dynamic filter must not suppress an error raised by the original join cast.
	auto null_check =
	    ExpressionFilter::CreateNullCheckExpression(std::move(null_check_input), ExpressionType::OPERATOR_IS_NULL);
	return make_uniq<BoundConjunctionExpression>(ExpressionType::CONJUNCTION_OR, std::move(null_check),
	                                             std::move(filter_expr));
}
```

Whenever a cast in the chain was recorded as a `DEFAULT_CAST` (meaning: the
*original* query's own cast is allowed to throw) but is flagged
`cast_can_fail`, the runtime filter's own copy of the input expression is
wrapped `IS NULL OR <filter>` — so a probe row whose reconstruction would
have failed is never silently excluded by the *filter*; it's let through
(matched, conservatively) to reach the join's own condition, where the
*original*, un-pushed-down cast will run, in the normal course of query
execution, and throw exactly the error it always would have. `join_filter_pushdown_cast.test`'s
final three cases (§7.24.7) are exactly regression tests for this: they
assert `statement error ... Conversion Error: ... out of range` for a query
whose join key requires a narrowing cast that overflows, confirming the
runtime filter's presence didn't quietly turn that error into a wrong,
silent answer.

## 7.24.7 Test-format note, then `join_filter_pushdown_cast.test` end to end

**How these tests actually check the pass fired** — and this is the
divergence from the same-label paired-`EXPLAIN` idiom `07-filter-pullup-and-pushdown.md`
documented for `FILTER_PUSHDOWN` — is exactly the mechanism §7.24.5 just
derived from source: an `EXPLAIN ANALYZE` block whose `analyzed_plan` output
is matched against a regex for the literal string `Dynamic Filters`, either
required (`<REGEX>:.*Dynamic Filters.*`) or forbidden
(`<!REGEX>:.*Dynamic Filters.*`):

```
query II
EXPLAIN ANALYZE SELECT COUNT(*) FROM (SELECT a::BIGINT AS a FROM probe_int) t JOIN build_bigint ON t.a = build_bigint.b;
----
analyzed_plan	<REGEX>:.*Dynamic Filters.*
```

There is no static plan to compare against, and no second query whose plan
this one must byte-match — because, per §7.24.5, a plain (non-`ANALYZE`)
`EXPLAIN` of the same query would *never* show this text at all, so there is
nothing meaningful to pair it against. `PRAGMA explain_output='OPTIMIZED_ONLY'`
(seen at the top of this file and several others) narrows the `EXPLAIN`
output to just the optimized logical/physical tree rather than every
intermediate stage, keeping the regex target small; `SET disabled_optimizers
TO 'compressed_materialization'` disables an *unrelated* pass (#34's
compressed-materialization sub-feature — the same one the VARIANT test
relies on being *enabled*, see below) specifically because, per the file's
own comment, that pass would otherwise rewrite some of these casts into
internal `__internal_compress_integral_*` functions before this pass ever
sees them, changing which cases actually exercise the cast-chain logic this
file is testing.

Walking the file's cases, grouped by what each one pins down:

**Basic downcast (lines 15-38).** `probe_int` stores `INT`; the query casts
it up to `BIGINT` in a projection before joining against `build_bigint`
(genuinely `BIGINT`). The `Dynamic Filters` row is required; `COUNT(*) = 4`
(all four build values are representable in `INT`, so all four legitimately
match). This is the baseline case every mechanism section above describes: a
safe, same-signedness widening cast (`INT`→`BIGINT`), `RuntimeFilterCastCanFail`
returns `false`, no `TRY_CAST` needed anywhere.

**Out-of-range build values are dropped, not miscounted (lines 40-53).**
`build_overflow` adds `9999999999` — too large for `INT32`. The filter is
still generated (`<REGEX>` required); `COUNT(*) = 2`, i.e. the two
in-range build values (`5`, `10`) still match, and the overflowing one
simply never matches anything (there's no `probe_int` row large enough to
equal it, filter or no filter) — this is confirming ordinary join
correctness survives an overflowing build value, independent of the filter.

**All build values overflow: the filter itself disappears (lines 55-67).**
`build_all_overflow` has *only* out-of-range values. Here the assertion
flips to `<!REGEX>` — **no** `Dynamic Filters` row at all. This is
`FinalizeFilters`'s cast-to-storage-type step
(`physical_hash_join.cpp:1905-1913`) failing outright: both `min_val` and
`max_val`, cast down to the probe's `INT` storage type via
`DefaultTryCastAs`, come back empty (`!cast_min || !cast_max`), and the code
`continue`s past this column without ever calling `PushFilter` — so
`dynamic_filters` stays empty, `HasFilters()` stays false, and the row is
absent. `COUNT(*) = 0` regardless (nothing in `probe_int`'s 0-99 range could
ever equal either overflowing value) — confirming the *query* stays correct
even when the *optimization* declines to engage at all.

**Narrower probe types — `TINYINT` and `SMALLINT` (lines 69-102).** Same
shape, smaller storage type: a build value that overflows `TINYINT`/`SMALLINT`
specifically gets excluded from the match count (`2` matches out of 3
plausible, `1` match out of 2) while the filter is still generated — proving
this isn't special-cased to any one integer width; `IsJoinFilterPushdownIntegralType`'s
"up to 64 bits" ceiling and `RuntimeFilterCastCanFail`'s size comparison both
operate generically across every integral width DuckDB has.

**NULLs on the build side (lines 104-116).** `build_with_nulls` includes a
`NULL`. The filter is still generated; `COUNT(*) = 2` (the two non-`NULL`
build values, unaffected — `MIN`/`MAX` aggregates ignore `NULL` inputs the
same way any ordinary `MIN`/`MAX` aggregate does, so a `NULL` in the build
side neither breaks nor widens the recorded range).

**Push-through-`GROUP BY` (lines 118-130), no `EXPLAIN` assertion, just a
correctness check.** The probe side is `SELECT x::BIGINT AS xb FROM
probe_for_agg GROUP BY x::BIGINT` — this is the `LOGICAL_AGGREGATE_AND_GROUP_BY`
branch of `GetPushdownFilterTargets` (§7.24.3): the filter column must
resolve to one of the aggregate's *group* expressions (not an aggregate
value), and that group expression is itself run back through
`PushdownJoinFilterExpression` recursively — here, the cast happens *inside*
the `GROUP BY` key expression itself, one more layer for the traversal to
compose through correctly.

**Non-integral cast (lines 132-147) — the important negative-space case.**
Both `probe_double` and `build_double` are plain `DOUBLE` columns, with *no
cast at all* on either side. The filter is still generated
(`<REGEX>` required) — confirming, concretely, the point §7.24.2 made in
the abstract: the integral-only restriction gates *cast-crossing*
reconstructions specifically, not the pushdown mechanism as a whole. A bare
column reference of a non-integral, non-cast type sails straight through
`PushdownJoinFilterExpressionInternal`'s `BOUND_COLUMN_REF` base case with
no type check at all.

**`HUGEINT`: the 64-bit ceiling actually fires (lines 149-164).**
`build_hugeint` is genuinely `HUGEINT` (128-bit); the probe side is cast up
to `HUGEINT` to match. Here the assertion flips to `<!REGEX>` — no filter —
which is `IsJoinFilterPushdownIntegralType`'s explicit
`GetTypeIdSize(...) <= GetTypeIdSize(PhysicalType::INT64)` ceiling rejecting
the cast outright at optimize time, before any of the runtime cast-safety
machinery even gets involved. `COUNT(*) = 2` regardless — correctness is
never at stake, only whether the optimization engages.

**Signed build values against an unsigned probe (lines 166-182).**
`probe_ubigint` is `UBIGINT`; `build_neg` (`BIGINT`) includes `-1`. The
filter is generated; only `5` and `10` match (`-1` cannot cast to `UBIGINT`
at all). This is `RuntimeFilterCastCanFail`'s signed-to-unsigned branch
(`source_type.IsSigned()` → unconditionally `true`) forcing a `TRY_CAST`
reconstruction rather than a blind range comparison — exactly the row in
§7.24.6's decision table that would be **wrong** if trusted naively (a raw
`UBIGINT` value has no representable negative counterpart to compare
against a negative endpoint at all).

**Cast directly inside the join condition, no wrapping projection (lines
184-219).** Confirms `PushdownJoinFilterExpressionInternal` reaches a cast
sitting straight inside `LogicalComparisonJoin.conditions[i].GetLHS()`, not
only casts that got hoisted into a separate `LogicalProjection` first — the
two representations reach the same function, `GenerateJoinFilters` calls
`PushdownJoinFilterExpression` on `cond.GetLHS()` directly regardless of
which shape produced it. The trailing case — inserting one more,
wildly out-of-range `9999999999` build value and re-running — flips the
assertion to `<!REGEX>` (the whole-column cast-to-storage-type step fails
again, exactly as in the "all build values overflow" case above, just with
one in-range value mixed in this time), while `COUNT(*)` stays `2`: the
comment "*All build values in range: all match*" is best read as asserting
that overall query correctness is unaffected by whether the optimization
engages, not that the new, overflowing value itself matches anything —
worth flagging explicitly rather than over-reading, per this book's
sourcing policy.

**`UNION ALL`, each branch keeping its own storage type (lines 221-248).**
`t_union_int` (`INT`) and `t_union_small` (`SMALLINT`) feed one probe side
via `UNION ALL`; both get the filter pushed independently, each cast
correctly for its own branch's storage type — this is
`GetPushdownFilterTargets`'s `LOGICAL_UNION` case rewriting column bindings
per child and recursing into each one separately (§7.24.3), made concrete:
a single logical filter fans out into two, differently-cast physical
filters, one per `UNION` branch's own `LogicalGet`.

**The final four cases (lines 250-301) are the correctness guard for
§7.24.6's error-preservation machinery**, and they're worth reading as a
matched pair. `CAST(CAST(p.k AS INTEGER) AS BIGINT) = b.k` with a probe row
of `3000000000` (too large for `INTEGER`) is asserted `statement error ...
Conversion Error: ... out of range` — the query, filter pushdown or not,
must still error, because the user wrote a `DEFAULT` (non-try) cast that
doesn't fit. Swapping in `CAST(TRY_CAST(p.k AS INTEGER) AS BIGINT)` instead
— now the user's own cast tolerates the overflow by returning `NULL` — is
asserted to return `count(*) = 2` cleanly, no error, the offending row
simply and correctly excluded. The last two cases repeat the *first*
(error-asserting) shape through a `LogicalProjection`-composed cast chain
and through an aggregate `GROUP BY` key, confirming the same
error-preservation guarantee holds regardless of which of
`GetPushdownFilterTargets`'s recursive branches (`LOGICAL_PROJECTION`,
`LOGICAL_AGGREGATE_AND_GROUP_BY`) assembled the cast chain.

## 7.24.8 `join_filter_pushdown_cast_variant.test` and the VARIANT-specific cases

This file adds `VARIANT` (DuckDB's semi-structured, dynamically-typed
column type) into the same cast-chain machinery, and its very first comment
flags a dependency worth stating precisely: *"Direct fully shredded VARIANT
joins are lowered to native integral casts by compressed materialization."*
Per §7.4's pipeline listing, `COMPRESSED_MATERIALIZATION` is a sub-feature
of `STATISTICS_PROPAGATION` (pass #34) — eight positions *before*
`JOIN_FILTER_PUSHDOWN` (#42) — and it can rewrite a `VARIANT` column that
turns out to be **fully shredded** (every value stored as the same,
statically-known integral type underneath the dynamic wrapper) into that
native integral representation directly. By the time this pass runs, a
fully-shredded `probe_direct_variant`/`build_direct_variant` pair has
frequently *already stopped being a `VARIANT` cast problem at all* — it's
reduced to the same plain integral-cast case §7.24.7 walked through, just
reached via a different starting SQL shape. This is exactly why
`variant_minimum_shredding_size = 0` is set at the top of the file (forcing
shredding to apply even to tiny test tables) and why a `CHECKPOINT`/`restart`
sequence appears midway through — to force the shredded, on-disk
representation to actually persist and be re-read, rather than possibly
staying an in-memory, unshredded value for the whole test.

**Direct fully-shredded VARIANT (lines 61-69).** `EXPLAIN ANALYZE` requires
`Dynamic Filters`; `COUNT(*) = 4`. Structurally identical to
`join_filter_pushdown_cast.test`'s basic downcast case, reached through
`VARIANT` syntax that's been lowered away before this pass ever sees it.

**Mismatched shredded widths need a shared type (lines 77-90).**
`probe_variant_smallint` shreds to `SMALLINT`, `build_variant_bigint` to
`BIGINT`. The `EXPLAIN (FORMAT yaml)` assertion just above the `EXPLAIN
ANALYZE` check pins down the *exact* reconstructed expression:

```
logical_opt	<REGEX>:.*CAST\(CAST\(a AS SMALLINT\) AS BIGINT\).*CAST\(b AS BIGINT\).*
```

— proving the join key gets normalized to a shared type (`BIGINT`, the
wider of the two) via a **two-step** cast: extract from `VARIANT` as
`SMALLINT` first, then widen to `BIGINT`. This is `PushdownJoinFilterExpressionInternal`'s
recursive case composing two `RuntimeFilterCastStep`s in order, exactly as
described in §7.24.2 — and it's a genuinely useful confirmation that the
pass's cast-chain bookkeeping handles more than one hop correctly, not just
the single-cast cases §7.24.7 mostly showed.

**NULL semantics survive VARIANT lowering (lines 92-100).** One `NULL`
value on each side; plain `=` yields `count = 1` (NULLs never match under
plain equality, VARIANT-lowered or not), while `IS NOT DISTINCT FROM` yields
`count = 2` (the NULL-to-NULL pair now counts as a match too) — and recall
from §7.24.2 that `COMPARE_NOT_DISTINCT_FROM` conditions are explicitly
excluded from filter generation altogether (the `TODO: Need OR predicates`
`continue` in `GenerateJoinFilters`'s condition loop) — so this second query
is, implicitly, also confirming that a NULL-safe join still returns the
*correct* answer even with no dynamic filter engaged for it at all, exactly
the same "correctness survives the optimization declining to fire"
discipline every negative case in this chapter follows.

**VARIANT probe cast down to a plain integral build column (lines 102-110).**
The `RECONSTRUCT_EXPRESSION` mode from §7.24.2's `JoinFilterExpressionPath`
comes alive here directly: extracting an integral value *out of* a
`VARIANT` and casting it to plain `INT` is accepted as a normal,
reconstructable cast step (`IsJoinFilterPushdownVariantIntegralCast` with
`src.id() == VARIANT`), unlike the reverse direction.

**Mixed-type VARIANT: structurally excluded, not merely declined (lines
112-120).** `probe_mixed_variant` stores a `VARCHAR` in roughly one row in
ten and an `INT` in the rest — genuinely **not** fully shredded, so
`COMPRESSED_MATERIALIZATION` cannot lower it to a single native type, and
the column really is dynamically typed at the storage layer. `EXPLAIN
ANALYZE` asserts `<!REGEX>` — no `Dynamic Filters` — which is
`GetPushdownFilterTargets`'s `LOGICAL_GET` case's own VARIANT check firing
(§7.24.3): `filter.storage_type.id() == VARIANT` and, since there's no
statically-known integral shredded type to reconstruct through, the
`reconstructed_type` is *also* still `VARIANT`, and the function bails
(`return;`) rather than accepting a target it has no comparable value to
build a range filter against. `COUNT(*) = 4` regardless, computed the
ordinary, un-accelerated way.

## 7.24.9 `join_filter_build_side.test`

This test belongs, in full, to `15-build-side-probe-side-test-driven.md`
§7.15.5 — its `jf_probe`/`jf_build` setup, the `WHERE b.keep` predicate's
role, and the exact `DynamicFilterBuildBonus` arithmetic
(`build_probe_side_optimizer.cpp:132-153`, constants at `:24-28`) that keeps
the ten-`VARCHAR`-column `jf_build` on the build side specifically *because*
its filtered ~10-row cardinality makes it a uniquely cheap, uniquely
effective source for a dynamic filter over `jf_probe`'s one million rows.
This chapter doesn't re-derive that arithmetic — the one addition worth
making here is naming, precisely, which piece of *this* pass's own code that
worked example leans on: `IsFiltering` (§7.24.3), called from
`JoinFilterPushdownOptimizer::IsFiltering` at
`join_filter_pushdown_optimizer.cpp:328-331`
(`get.table_filters.HasFilters()` for a `LogicalGet`), is the exact function
`DynamicFilterBuildBonus`'s first guard calls to decide whether a candidate
build side is even worth considering for the bonus at all — the two passes
share not just a design intuition but this one literal function call.

## 7.24.10 Outer-join safety: `test_inner_join_filter_pushdown.test`, `test_right_join_filter_pushdown.test`, `test_left_join_filter_pushdown.test`, `test_semianti_join_filter_pushdown.test`

**None of these four files assert anything about `EXPLAIN` output at all** —
no `Dynamic Filters` regex, no plan shape check. Every query in them is a
plain `query ... rowsort` result-correctness check. That is itself the
point: `JoinTypeIsSupported` (§7.24.2) is a single, per-join-operator gate
keyed only on `join.join_type` — it doesn't inspect how complicated the
`ON` clause is, how many extra conjuncts it carries, or whether some of
those conjuncts reference only one side. These four files are a battery of
increasingly elaborate `ON`-clause shapes — a plain equi-join, an equi-join
plus a build-side-only conjunct, an equi-join plus a probe-side-only
conjunct, a constant `true`/`false` conjunct, a genuinely cross-column
arithmetic conjunct (`l.amount + r.budget > 1100`) — run against every join
type this pass treats differently, to confirm the type-level gate alone is
sufficient: no combination of extra conjuncts can smuggle a filter past it
for a join type where one shouldn't exist, and none of them accidentally
suppress a filter for a join type where one safely could.

**`test_inner_join_filter_pushdown.test`** is the baseline: `INNER JOIN`
falls through `JoinTypeIsSupported`'s `default: return true`, so every one of
its cases is, in principle, filter-pushdown-eligible, and every result is
exactly what an ordinary (non-accelerated) inner join would produce — the
filter, when it fires, can only ever *remove* rows the join would have
discarded anyway. One case is worth pulling out on its own because it ties
directly back to a specific branch this chapter already read in source:

```sql
-- test/sql/join/inner/test_inner_join_filter_pushdown.test:78-82
WITH c AS (
	SELECT flag FROM ordered_probe ORDER BY ord LIMIT 1
)
SELECT * FROM c INNER JOIN boolean_keys ON flag = flag_key;
----
```

(Nothing follows the `----` line: the expected result is no rows.)

`ordered_probe` has rows `(true,1)`, `(true,2)`, `(false,3)`; `ORDER BY ord
LIMIT 1` keeps only the first, `(true,1)`; `boolean_keys` holds only
`(false)`. If a dynamic filter derived from `boolean_keys`'s min/max
(collapsed to the single value `false`, since it's a one-row table) were
pushed *underneath* the `LIMIT`, it would filter `ordered_probe` down to only
its `false`-flagged row *before* the `ORDER BY ... LIMIT 1` ever ran — which
would then keep `(false,3)` as "the first row" instead of the true first row,
`(true,1)`, silently changing which row the `LIMIT` selects. The empty
result actually observed is exactly what happens when the filter is *not*
pushed below the `LIMIT`: the true first row, `(true,1)`, correctly reaches
the join, correctly fails to match `flag_key = false`, and the join
correctly returns nothing. This is `GetPushdownFilterTargets`'s
`LOGICAL_LIMIT`/`LOGICAL_TOP_N` `break;` (§7.24.3) doing its job, made
concrete and observable rather than merely asserted from the comment above
it in the source.

**`test_left_join_filter_pushdown.test`** exercises `LEFT JOIN`, which
`JoinTypeIsSupported` excludes unconditionally
(`join_type == JoinType::LEFT` is in the exclusion `switch`, §7.24.2).
Concretely, take its first case:

```sql
-- test/sql/join/left_outer/test_left_join_filter_pushdown.test:17-25
SELECT * FROM left_table l
LEFT JOIN right_table r ON l.id = r.id AND l.val > 1 AND r.category = 1;
----
1	1	50	NULL	NULL	NULL
2	1	75	NULL	NULL	NULL
3	2	60	3	1	1500
4	2	90	NULL	NULL	NULL
5	99	100	NULL	NULL	NULL
```

`right_table` here has `id`s `{1, 2, 3}`; suppose (contrary to fact — this
is the hypothetical the exclusion prevents) the pass *did* generate a
dynamic filter from `right_table.id`'s build-side range `[1, 3]` and pushed
`l.id BETWEEN 1 AND 3` onto `left_table`'s scan, exactly the way it safely
would for an `INNER` join. `left_table` has no row with `id` outside `[1,3]`
in this particular data, so this specific example wouldn't even expose the
bug — but change `left_table` to include, say, `(6, 1, 10)`, and the
difference becomes exact: the correct `LEFT JOIN` output must still include
`6 1 10 NULL NULL NULL` (row 6 has no match, but a `LEFT JOIN` preserves it,
NULL-extended) — while a probe-side prefilter keyed to the build side's
`[1,3]` range would silently discard row 6 *before the join ever saw it*,
and it would vanish from the output entirely, with no NULL-extension, no
error, nothing — a straightforwardly wrong, silently-undercounted result.
That is the concrete failure mode `join_type == JoinType::LEFT` being in
`JoinTypeIsSupported`'s exclusion list prevents, and it is categorically
different from — and a strict superset of the risk in — the `LIMIT`/`TOP_N`
case above: there, pushing early changed *which* row the query kept; here,
it would make the query drop a row the join's own semantics guarantee must
survive, no matter what.

This is also where `07-filter-pullup-and-pushdown.md`'s coverage of the
*static* `FILTER_PUSHDOWN` pass (#6) is worth cross-referencing directly,
because both passes face the identical hazard and respond to it with
genuinely different tools. That chapter's §2.7–2.8 (`pushdown_left_join.cpp`,
`pushdown_outer_join.cpp`) describe a filter that *is* known at plan time
(a `WHERE`-clause predicate, or a coalesced-equal-keys pattern) being pushed
across a `LEFT`/`FULL OUTER JOIN` boundary only after `NotNullExpressionAnalyzer`
or an equivalent proof shows the predicate provably rejects every NULL a
NULL-extending join would otherwise introduce — a fine-grained, per-predicate
proof obligation, because the pass has the predicate's actual logic in hand
to reason about. `JOIN_FILTER_PUSHDOWN` has no such proof available to it —
the filter it would generate doesn't exist yet, isn't known until execution,
and by construction can only ever be "the build side's observed key range,"
which is provably *never* a subset of "values that would have been NULL
anyway." Lacking any predicate to reason about, the pass responds with the
blunter, coarser-grained tool available to it: refuse to run *at all* for
any join type where the probe side's cardinality must be fully preserved,
checked once, per join operator, by `join_type` alone (§7.24.2's
`JoinTypeIsSupported`) — a single boolean gate standing in for what the
static pass does with a full per-predicate NULL-rejection proof.

**`test_right_join_filter_pushdown.test`** looks, from its name and its
correct, classic RIGHT-JOIN-shaped output (every `right_table` row present,
`left_table` rows without a match simply absent from the *left* columns),
like it ought to be testing the `JoinType::RIGHT` branch of
`JoinTypeIsSupported`'s `switch` — recall that branch is *not* in the
exclusion list, and falls through to `default: return true`. Reading the
binder's own join-planning code turns up a genuinely surprising fact worth
stating plainly, in this book's stated style of flagging what a superficial
reading would get wrong: **by default, this query never reaches that branch
of the `switch` as a `RIGHT` join at all.**

```cpp
// src/planner/binder/tableref/plan_joinref.cpp:466-470
if (!has_dependent_condition && ref.type == JoinType::RIGHT && ref.ref_type != JoinRefType::ASOF &&
    !Optimizer::OptimizerDisabled(context, OptimizerType::BUILD_SIDE_PROBE_SIDE)) {
	// Binding follows the syntactic order. Normalize finalized RIGHT joins only after both sides and ON are bound.
	ref.type = JoinType::LEFT;
	std::swap(left, right);
}
```

Every ordinary `A RIGHT JOIN B` is rewritten, at **bind time** — during
logical-plan construction, well before *any* of the ~45 passes in this
chapter's pipeline even begin — into `B LEFT JOIN A` with the two children
swapped, provided `BUILD_SIDE_PROBE_SIDE` (pass #27) isn't explicitly
disabled. By the time `JoinFilterPushdownOptimizer::VisitOperator` (pass
#42) ever sees this join, its `join_type` is `LEFT`, not `RIGHT` — and
`LEFT` is excluded. So `test_right_join_filter_pushdown.test`, despite its
name and its `RIGHT JOIN` SQL syntax, is — in this checkout's default
configuration — testing exactly the same exclusion as
`test_left_join_filter_pushdown.test`, just entered through a bind-time
normalization rather than through literal `LEFT JOIN` syntax. The semantic
reasoning is symmetric either way: a `RIGHT JOIN`'s *preserved* side is
`right_table`, syntactically written last but, after this rewrite, playing
the role of the *new* left child of an internal `LEFT JOIN` — the exact
child role `JoinTypeIsSupported`'s exclusion is protecting. Whether the
`default: return true` case for a literal, un-normalized `JoinType::RIGHT`
is ever reachable in practice — only if a user explicitly runs `SET
disabled_optimizers = 'build_side_probe_side'` first, which turns off both
this normalization *and* the cost-based build/probe swap `15-build-side-probe-side-test-driven.md`
covers — is a real, narrow, verifiable code path this chapter flags rather
than dismisses, but it is not the path this particular test file's queries
take by default.

**`test_semianti_join_filter_pushdown.test`** is the one file in this list
that exercises a genuine *split* within `JoinTypeIsSupported`'s exclusion
list: `SEMI` falls through to `default: return true` (eligible), while
`ANTI` is explicitly excluded, with its own distinct reasoning
(`FIXME: anti/right_anti - we could do this, but need to invert the join
conditions`) rather than the "preserve every probe row" reasoning `LEFT`/
`OUTER` get. A `SEMI` join's output is exactly the probe rows that *do*
match something on the build side — a probe row provably outside the build
side's key range provably has no match, so discarding it early is exactly
the same "changes when, never whether" reasoning that makes `INNER`
eligible. An `ANTI` join's output is the *complement* — exactly the probe
rows that *don't* match — so the very same build-side range, applied
naively as an inclusion filter, would discard the *wrong* rows entirely
(everything the range *would* have matched, when the query actually wants
everything that *doesn't*); making this safe would require generating an
inverted predicate ("keep only rows outside `[min, max]`"), which is
precisely what the `FIXME` names as unimplemented rather than unsafe in
principle. The file's own results confirm this split holds in practice:
every `SEMI JOIN` query returns exactly the ordinary, correct semi-join
result (rows of `left_table` with a qualifying match), and every `ANTI
JOIN` query returns exactly the ordinary, correct anti-join result (rows of
`left_table` with no qualifying match) — with no test in the file asserting
anything about which of the two internally engaged a dynamic filter, again
because these are correctness batteries, not plan-shape assertions.

## 7.24.11 `nop_collation_join_filter_pushdown.test`

```sql
-- test/sql/collate/nop_collation_join_filter_pushdown.test:23-26
EXPLAIN ANALYZE SELECT * FROM s, t WHERE a = b COLLATE C;
----
analyzed_plan	<REGEX>:.*Dynamic Filters:.*
```

`s.a` is `TEXT`, 10,000 rows; `t.b` is `TEXT`, one row (`'val_3'`). `COLLATE
C` is DuckDB's "NOP" collation — the identity/binary collation, meaning
plain byte comparison with no locale-aware transform applied — and the
point of this test is exactly the one §7.24.7's `DOUBLE` case already made
from a different angle: **the min/max mechanism itself has no integral-type
restriction.** A `VARCHAR` equi-join key, with no cast at all, produces a
`MIN`/`MAX` pair over strings exactly the way `MinFun`/`MaxFun` would for
any orderable type, and `FinalizeFilters`' single-value collapse case
(`t.b`'s one row makes `min = max = 'val_3'`) generates a plain, one-sided
`a = 'val_3'` equality filter on `s`'s scan — pruning 10,000 candidate rows
down to whichever ones equal a single known string, before the join runs at
all. `SET disabled_optimizers TO 'column_lifetime,compressed_materialization'`
disables two unrelated passes purely to keep the plan simple enough for the
regex to target reliably; neither has any bearing on whether this pass
itself engages. The one thing this test does *not* exercise is a collated
(non-NOP) `MIN`/`MAX` — recall §7.24.2's own guard, `if
(aggr_expr->GetChildren().size() != 1) { return; }`, which is precisely the
non-NOP-collation case this test's choice of `COLLATE C` is careful to
avoid triggering.

## 7.24.12 Summary: what this pass is, in the vocabulary the rest of this chapter uses

Every other named pass in §7.4's pipeline answers "what should this plan
look like" once, at optimize time, using only information the plan itself
already carries. `JOIN_FILTER_PUSHDOWN` answers a structurally different
question — "what plumbing does this plan need so that a value which won't
exist until execution can still reach the right place once it does" — and
every piece of its machinery follows from that one framing: the optimizer
side (`join_filter_pushdown_optimizer.cpp`) records *how* to reconstruct a
probe-side expression and *where* the eventual filter should land, never a
value; the execution side, riding inside `PhysicalHashJoin`'s own
Sink/Combine/Finalize exactly as Chapter 11 §11.3 describes for any sink,
turns a per-thread partial aggregate into a global one and then, in
`Finalize` alone, turns that aggregate into an actual, installed predicate on
a `shared_ptr<DynamicTableFilterSet>` the probe side has been holding empty
since optimize time; `runtime_filter_cast.cpp` is the narrow, carefully
bounded layer making sure that reconstruction survives a probe/build type
mismatch without ever letting the optimization change which rows a query
matches or which rows cause it to error; and the `EXPLAIN`/`EXPLAIN ANALYZE`
divergence this chapter opened with is not a documentation quirk but a
direct, mechanical consequence of one `if (dynamic_filters->HasFilters())`
check reading state that a plan which was never executed can never have
populated. The `JoinTypeIsSupported` gate, finally, is what keeps all of
this from ever running somewhere it would be unsafe to — not by reasoning
about any specific query's predicates, the way the *static* `FILTER_PUSHDOWN`
pass must, but by refusing outright, for six join types, before any of the
rest of this machinery is even built.
