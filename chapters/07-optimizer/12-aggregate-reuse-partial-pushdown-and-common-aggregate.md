# 7.12 `AGGREGATE_REUSE`, `PARTIAL_AGGREGATE_PUSHDOWN`, and `COMMON_AGGREGATE`

This file goes deep on three of the pipeline's ~45 passes (§7.4's numbered list),
all three nominally about "not doing aggregate work twice," but landing at three
genuinely different scopes of that idea:

- **`AGGREGATE_REUSE`** (pass #18) — reusing a *finished aggregate a completely
  separate part of the query already computed*, not just sharing one hash
  table across sibling functions in one `GROUP BY`.
- **`PARTIAL_AGGREGATE_PUSHDOWN`** (pass #19) — restructuring a
  `GROUP BY`-over-a-join so a cut-down aggregate runs *below* the join instead
  of the join running on unreduced rows.
- **`COMMON_AGGREGATE`** (pass #37) — deduplicating two aggregate function
  calls *within one `LogicalAggregate`* that turn out, after everything
  upstream of this pass has already run, to be requesting the literal same
  computation.

None of these three passes explains or reimplements the hash-based grouped
aggregation algorithm itself — that is entirely Chapter 14's subject
(`GroupedAggregateHashTable`, the salted pointer-table probe, the two-phase
Sink/Combine/Finalize/GetData lifecycle). What all three passes here have in
common is that they operate on the *plan* that feeds into that machinery,
deciding whether it needs to run at all, where it needs to run, and how many
distinct group-by computations it needs to run — never touching how one
`GROUP BY` gets executed once the decision to run it has been made. Every
"decompose/combine/finalize" concept below (`ExportAggregateFunction`,
`combine_aggr`, `HasStateCombineCallback`) is the *plan-time* analogue of
Chapter 14 §14.6–14.7's *runtime* Phase-1/Phase-2 combine step: the same idea
— a partial accumulator state can be merged with another partial accumulator
state of the same aggregate to produce the accumulator state you'd have
gotten from seeing all the rows at once — reapplied across plan boundaries
(joins, CTE materialization points, `LEFT`/`RIGHT` outer joins) rather than
across worker threads.

## 7.12.1 `AGGREGATE_REUSE` — reusing a finished aggregate from elsewhere in the query

**Purpose, in one sentence:** when the plan already computes a grouped
aggregate as a byproduct of evaluating some other part of the query — a
`SEMI`/`ANTI` join built to check `IN`/`EXISTS`, or a separately materialized
CTE — and a `GROUP BY` somewhere else in the query needs that exact same
per-key aggregate value, `AggregateReuseOptimizer` rewrites the second
computation to consume the first one's result instead of re-scanning and
re-aggregating from the base tables.

This is worth stating precisely up front because the name invites the wrong
first guess. It is **not** about `SELECT dept, SUM(salary), AVG(salary),
COUNT(*) FROM emp GROUP BY dept` sharing one hash table across its three
aggregate *functions* — that sharing is not an optimization decision at all;
it falls straight out of how `GroupedAggregateHashTable` is built (Chapter
14 §14.4's `TupleDataLayout` gives every group's row a fixed shape with one
running accumulator slot per aggregate function, computed in the same single
pass over the input regardless of how many aggregate functions the query
asks for). There is no separate optimizer pass needed to make that case fast,
so there is nothing for `AGGREGATE_REUSE` to do there.

What this pass actually targets is the more surprising case: **two
syntactically separate aggregations, in different parts of the plan tree,
that happen to compute overlapping work over the same underlying rows.**
The class's own doc comment names both shapes it looks for directly:

```cpp
// src/include/duckdb/optimizer/aggregate_reuse.hpp:18
//! Reuses grouped aggregate work already available through SEMI joins or materialized CTEs.
class AggregateReuseOptimizer : public LogicalOperatorVisitor {
```

Two independent sub-mechanisms live behind that one comment, in two files:
`TryReuseSemiAggregate` (`src/optimizer/aggregate_reuse.cpp`) and
`TryReuseMaterializedAggregate` (`src/optimizer/materialized_aggregate_reuse.cpp`).
The visitor tries them on each operator it reaches, bottom-up. It tries the
materialized-CTE path first. It tries the SEMI-join path only when the first
one did not fire:

```cpp
// src/optimizer/aggregate_reuse.cpp:533-538
void AggregateReuseOptimizer::VisitOperator(unique_ptr<LogicalOperator> &op) {
	LogicalOperatorVisitor::VisitOperator(op);
	if (!TryReuseMaterializedAggregate(op)) {
		TryReuseSemiAggregate(op);
	}
}
```

(The materialized-CTE path is tried first. This section still explains the
SEMI-join path first, because it is the simpler one.)

### Sub-mechanism 1: reusing an aggregate exposed by a filtering SEMI join

Consider this shape — an `IN` subquery next to an ordinary `GROUP BY` over
the same fact table:

```sql
SELECT o.k, d.label, sum(f.v)
FROM reuse_owner o JOIN reuse_dim d ON o.dim_id = d.id JOIN reuse_fact f USING (k)
WHERE o.k IN (SELECT k FROM reuse_fact GROUP BY k HAVING sum(v) > 25)
GROUP BY o.k, d.label
```

(`test/optimizer/aggregate_reuse.test:31-34`, against `reuse_dim`/`reuse_owner`/
`reuse_fact` tables the same file defines at lines 5–21.) Decorrelated, the
`IN` subquery becomes a `SEMI` join: `reuse_owner` (and whatever it's already
joined to) on the left, and on the right a projection over `GROUP BY k;
SUM(v)` filtered by `HAVING sum(v) > 25`, projecting only `k` back out. That
right-hand aggregate computes `SUM(v)` **per key**, purely to decide which
keys pass the `HAVING` check — and then the outer query goes on to compute
`SUM(f.v)` **grouped by the same key**, by scanning and re-joining
`reuse_fact` a second time. Without this pass, `reuse_fact` is scanned twice
for exactly the same per-key sum; the baseline `EXPLAIN` (with the pass
disabled, `SET disabled_optimizers = 'aggregate_reuse'`, line 28) confirms
this directly with a physical-plan regex that requires the string
`reuse_fact` to appear twice (line 36).

**The eligibility check — `GetCandidate`** (`aggregate_reuse.cpp:167-216`) —
is the exact condition under which the *outer* aggregate is even a candidate
to reuse *something*:

```cpp
// src/optimizer/aggregate_reuse.cpp:167-176 (elided)
static optional<AggregateJoinCandidate> GetCandidate(LogicalAggregate &aggregate, LogicalComparisonJoin &join) {
	if (join.join_type != JoinType::INNER || join.children.size() != 2 || join.conditions.size() != 1 ||
	    join.HasProjectionMap() || join.HasArbitraryConditions() || aggregate.expressions.size() != 1 ||
	    !HasCompleteGrouping(aggregate)) {
		return nullopt;
	}
	auto &condition = join.conditions[0];
	if (!condition.IsComparison() || condition.GetComparisonType() != ExpressionType::COMPARE_EQUAL ||
	    condition.GetLHS().GetReturnType() != condition.GetRHS().GetReturnType()) {
		return nullopt;
	}
```

Three requirements worth calling out precisely: **exactly one aggregate
expression** (`aggregate.expressions.size() != 1` bails), a **single-equality
inner join** directly underneath the aggregate, and `HasCompleteGrouping`
(`aggregate_reuse.cpp:148-165`) — no `ROLLUP`/`CUBE`/multiple grouping sets,
just one plain, complete `GROUP BY` list. Then `GetCandidate` rejects the
outer aggregate outright if it's `DISTINCT`, volatile, already state-exported,
carries a `FILTER`/`ORDER BY`, takes more than one argument, or its function
lacks *any* of `HasGetStateTypeCallback`/`HasStateCombineCallback`/
`HasStateFinalizeCallback` (`aggregate_reuse.cpp:183-191`) — i.e. it must be
a genuinely combinable aggregate in Chapter 14's sense, the same three
callbacks the runtime's own Phase-1/Phase-2 merge relies on. Finally it
determines which join child is the **"owner"** side (the side the outer
`GROUP BY`'s keys trace back to — here, `reuse_owner ⋈ reuse_dim`) and which
is the **"probe"** side (the side the aggregate's sole argument traces back
to — here, `reuse_fact`), by trying both assignments (`aggregate_reuse.cpp:192-214`).

**Finding the donor — `PromoteReusableAggregate`** (`aggregate_reuse.cpp:357-518`)
then walks down through the owner side looking for a `SEMI` join whose left
key matches the owner-side join key, and whose right (domain) side, after
descending through allowed `LOGICAL_PROJECTION`/`LOGICAL_FILTER` operators,
bottoms out at a `FindUnaryAggregate` (`aggregate_reuse.cpp:258-268`) — the
decorrelated `HAVING` subquery's own `GROUP BY k; SUM(v)` node. It checks,
via `SameReusableAggregate` (`aggregate_reuse.cpp:343-355`), that one of that
inner aggregate's expressions is **exactly** the outer aggregate: same
function, same return type, same state-export mode, no `DISTINCT`/`FILTER`/
`ORDER BY` on either side, and — critically — the same `TableColumnOrigin`
(`aggregate_reuse_internal.hpp:13-16`, traced through
`GetTableColumnOrigin`/`GetDirectReferenceIndex`, all the way back to the
literal `(TableCatalogEntry, ColumnIndex)` pair) for both the join key and
the aggregate's argument. This origin-tracing is the correctness backbone of
the whole pass: it is not enough for two `SUM(v)` calls to look syntactically
alike — they have to be provably summing *the same physical column of the
same physical table*, with no filter anywhere on the path (`allow_filtered`
defaults to `false` in the equality check) that could have changed which
rows contribute to one sum but not the other.

Once a match is found, the pass does four things, in order (still inside
`PromoteReusableAggregate`, `aggregate_reuse.cpp:470-498`):

1. **Exports** the inner aggregate's state (`ExportAggregateFunction::Bind`)
   instead of finalizing it to a plain scalar — the same export mechanism
   §7.12.2 uses for join pushdown.
2. **Finalizes every *existing* consumer** of that raw finalized value
   (`FinalizeAggregateConsumers`, `aggregate_reuse.cpp:318-341`) — here, the
   `HAVING sum(v) > 25` filter itself, which still needs the ordinary,
   finalized number to compare against `25`, even though the aggregate
   expression underneath it now produces raw state.
3. **Promotes the raw state** as a new, additional output column through
   every intervening projection/filter (`PromoteAggregateBinding`,
   `aggregate_reuse.cpp:270-304`), so it survives out to the `SEMI` join's
   right-hand output even though nothing originally asked for it there.
4. **Flips the join from `SEMI` to `INNER`** (`aggregate_reuse.cpp:494`) —
   because the outer query now genuinely needs to *read* a value the right
   side carries, not just check for the right side's existence.

**Reassembling the outer aggregate.** Back in `TryReuseSemiAggregate`
(`aggregate_reuse.cpp:549-647`), the exposed per-key state becomes the sole
input to a **new, minimal aggregate**: `GROUP BY <owner-side groups>;
combine_aggr(<exported state>)` (`aggregate_reuse.cpp:598-618`), followed by
a projection that finalizes the combined state back into the aggregate's
normal return type (`aggregate_reuse.cpp:630-637`). `combine_aggr` is the
same generic, single-argument state-merge function §7.12.2 builds on
(`CombineAggrFun`, `src/include/duckdb/function/aggregate/distributive_functions.hpp:104-112`) — here bound with
one argument, no multiplicity.

Why a fresh **`combine`** step, rather than simply projecting the promoted
state straight through as the answer? Because the (owner ⋈ domain) join,
now `INNER`, can emit **more than one row per key** if the owner side has
duplicate keys — and each such row carries an identical copy of the
per-key exported state. `combine_aggr`, applied once per row the join
emits, adds those copies together exactly as many times as the owner side
repeats that key. This is provably correct precisely because the outer
`GROUP BY`'s key is the *same* key the join condition equates: joining N
owner rows sharing key `k` to the one domain row for `k`, then re-grouping
by `k` and summing the (per-key, already-summed) payload N times, gives the
same answer as joining N owner rows to *all* the individual fact rows for
`k` and summing their raw values once — because every one of those N owner
rows would, in the un-rewritten plan, pull in the *entire* same set of
`reuse_fact` rows for `k` and contribute the same total. The test suite
checks this explicitly with a duplicated owner table:

```sql
-- test/optimizer/aggregate_reuse.test:169-176
CREATE TABLE reuse_duplicate_owner AS SELECT * FROM reuse_owner UNION ALL SELECT * FROM reuse_owner;

SELECT o.k, sum(f.v)
FROM reuse_duplicate_owner o JOIN reuse_fact f USING (k)
WHERE o.k IN (SELECT k FROM reuse_fact GROUP BY k HAVING sum(v) > 25)
GROUP BY o.k
```
with expected result `1  60` / `3  180` (`aggregate_reuse.test:188-195`) — exactly
double the single-owner-row answer (`1  30` / `3  90`, line 75-76), because
`combine_aggr` adds the same 30/90 state to itself once per duplicate owner
row. `EXPLAIN`'s physical plan confirms the mechanism directly, matching
`combine_aggr` in its operator list (line 186) while showing no repeated
`reuse_fact` scan (line 178) and no surviving `SEMI` join anywhere in the
plan (`aggregate_reuse.test:56`, `<!REGEX>:.*Join Type: SEMI.*`).

### Walking the rest of `aggregate_reuse.test`'s SEMI-join cases

The file (`test/optimizer/aggregate_reuse.test`) is deliberately adversarial
about every place this rewrite could silently produce a wrong answer if any
one of the guards above were missing. Worth reading each rejection as a
correctness argument, not just a "doesn't fire" note:

- **NULL propagation.** `WHERE o.k IN (SELECT k FROM reuse_fact GROUP BY k
  HAVING sum(v) IS NULL)` — `reuse_fact`'s `k=4` row has `v = NULL`, so
  `SUM(v) IS NULL` passes only for `k=4`. The rewrite still applies, and the
  reused, re-finalized `SUM` correctly stays `NULL` all the way through the
  new inner join and combine step (`4  d  NULL`, lines 79-85) — no accidental
  coercion to `0` anywhere along the export/combine/finalize chain.
- **A non-unique grouping key is fine.** `GROUP BY d.label` (dropping `o.k`
  from the `GROUP BY` list entirely) still reuses the aggregate — `label`
  isn't unique per key (`k=1` and `k=3` both map to `label='x'`), but the
  pass only requires that every group expression trace back to the *owner*
  side of the join, not that it be the join key itself; two keys' reused
  states simply get `combine_aggr`'d together under the shared label,
  correctly summing to `120` (`30 + 90`, lines 88-102).
- **`DISTINCT` blocks reuse outright.** `count(DISTINCT f.w)` against a
  domain `HAVING count(DISTINCT w) > 1` still re-scans `reuse_fact` twice
  (line 108-114) — `GetCandidate` rejects any `IsDistinct()` outer aggregate
  unconditionally (`aggregate_reuse.cpp:184`), because a `DISTINCT`
  aggregate's state has a fundamentally different combine semantics (it has
  to track which raw values it has already seen, not just an additive
  accumulator) that this pass doesn't attempt to reason about.
- **Every outer aggregate must be reusable, or none are.** `sum(f.v),
  count(*)` together — `GetCandidate` requires `aggregate.expressions.size()
  == 1` (line 118-123), so a `LogicalAggregate` with two output expressions
  is rejected wholesale, even though `sum(f.v)` alone would have qualified.
- **Wrong argument, wrong function, filtered domain, filtered join
  condition** — `sum(f.w)` vs. the domain's `sum(v)` (different column
  origin), `max(f.v)` vs. `sum(v)` (different function), an added `WHERE
  w >= 2` inside the subquery, or an added `f.w >= 2` join condition on the
  *outer* join — all four independently break `SameReusableAggregate`'s
  origin/function equality or `GetCandidate`'s "one equality condition, no
  arbitrary conditions" requirement, and all four correctly fall back to
  scanning `reuse_fact` twice (lines 126-157).
- **The SEMI join must consume the group key itself.** `WHERE o.k IN
  (SELECT CAST(count(*) AS INTEGER) FROM reuse_fact GROUP BY k HAVING
  sum(v) > 25)` projects `count(*)`, not `k`, out of the domain subquery —
  so the value the `SEMI` join actually equates against isn't the aggregate's
  own group key, and `PromoteReusableAggregate`'s `traced_group`/
  `projected_key` check (`aggregate_reuse.cpp:446-451`) correctly refuses to
  treat this as the same domain (lines 160-166).

### Sub-mechanism 2: reusing an entire materialized CTE's join result

The second file, `materialized_aggregate_reuse.cpp`, is the more dramatic
case the task set out to find: it doesn't just reuse one aggregate function's
value — it recognizes that a **correlated aggregate's entire join graph** is
a strict subset of a **separately materialized CTE's** join graph (missing
exactly one relation), and rewrites the correlated aggregate to scan the
already-materialized CTE directly instead of re-scanning and re-joining every
table the CTE already joined.

The canonical test case is TPC-H's "minimum cost supplier" shape:

```sql
-- test/optimizer/aggregate_reuse.test:233-247 (elided for width)
WITH outer_rows AS MATERIALIZED (
  SELECT p.k, p.label, f.cost, s.name, s.address, s.phone
  FROM payload_part p, payload_fact f, payload_supplier s, payload_region r
  WHERE p.k = f.k AND s.id = f.supplier_id AND s.region_id = r.id
    AND p.size = 15 AND p.kind = 'BRASS' AND r.name = 'EUROPE'
), minimum_cost AS (
  SELECT f.k, min(f.cost) AS cost
  FROM payload_fact f, payload_supplier s, payload_region r
  WHERE s.id = f.supplier_id AND s.region_id = r.id AND r.name = 'EUROPE'
    AND f.k IN (SELECT k FROM outer_rows)
  GROUP BY f.k
)
SELECT o.name, o.address, o.phone, o.k, o.label, o.cost
FROM outer_rows o JOIN minimum_cost m ON o.k = m.k AND o.cost = m.cost
```

`outer_rows` joins **four** relations (`part`, `fact`, `supplier`, `region`)
under three filters; `minimum_cost` joins **three** of those same four
relations (`fact`, `supplier`, `region`, with the *same* `region.name =
'EUROPE'` filter and the *same* `supplier`↔`region` edge) and restricts
`f.k` to `outer_rows`'s own keys via a correlated `IN` — which, structurally,
is exactly the "domain semi join" shape `TryReuseSemiAggregate` recognizes,
just against a materialized CTE (`GetDomainCTERef`/`FindDomainSemi`,
`materialized_aggregate_reuse.cpp:131-237`) instead of an inline subquery.
The rewrite's insight: `minimum_cost`'s three-relation join is `outer_rows`'s
four-relation join **minus exactly the `part` relation**, and the
`f.k IN (SELECT k FROM outer_rows)` correlation is exactly the edge that
`part.k = fact.k` supplied in `outer_rows`. If every other join edge and
every other per-table filter on the shared relations matches — verified by
building a `RelationGraph` (sources + edges) for each side
(`CollectInnerGraph`, `materialized_aggregate_reuse.cpp:67-129`) and
requiring the producer's graph to have exactly one relation the core
doesn't (`producer_graph.sources.size() != core_graph.sources.size() + 1`,
`materialized_aggregate_reuse.cpp:446`), every shared edge to match
(`SameEdge`, line 59-65), and every shared table's filters to match
literally (`SourceFiltersMatch`, lines 307-341, via `Expression::Equals` on
each `ExpressionFilter`) — then `minimum_cost`'s `GROUP BY f.k; MIN(cost)`
can be rewritten to scan `outer_rows` directly (a `LogicalCTERef`, built at
`materialized_aggregate_reuse.cpp:531-539`) instead of re-scanning
`payload_fact`/`payload_supplier`/`payload_region` and rejoining them.
`EXPLAIN`'s `logical_opt` output confirms this exactly as it did for the
SEMI case: disabled, `payload_fact` appears twice (line 248); enabled, it
appears once (line 269).

**The one restriction that makes this safe under duplication, and why it's
conservative rather than exact.** Unlike the SEMI-join path, this rewrite has
**no combine step** — it just re-points the correlated aggregate's `FROM`
clause at the materialized CTE and reruns an ordinary `GROUP BY` over it.
That means if the "extra" relation (`payload_part` here) can supply more
than one matching row per key — i.e. it can fan out the join — a
duplication-sensitive aggregate like `SUM`/`COUNT`/`AVG` would silently
double-count. The pass's guard against this sits, unconditionally, at the
very top of `TryReuseMaterializedAggregate`:

```cpp
// src/optimizer/materialized_aggregate_reuse.cpp:420-424
auto &bound_aggregate = expression->Cast<BoundAggregateExpression>();
if (!bound_aggregate.IsDistinct() &&
    bound_aggregate.Function().GetDistinctDependent() != AggregateDistinctDependent::NOT_DISTINCT_DEPENDENT) {
	return false;
}
```

This reuses the exact enum §7.5 of the overview already introduced for a
different rule (`DistinctAggregateOptimizer`, which drops a redundant
`DISTINCT` off `MIN`/`MAX` because de-duplicating their input never changes
the answer): `AggregateDistinctDependent::NOT_DISTINCT_DEPENDENT` means "this
aggregate's result is a function of the *set* of input values, not the
*multiset*" — and that is precisely the property that makes the aggregate
safe to run over a join that might fan out through an unrelated extra
relation, since a fan-out only ever duplicates values, never introduces new
ones. `MIN`/`MAX` qualify; `SUM`/`COUNT`/`AVG` do not, unless the query
explicitly marked them `DISTINCT` (which independently makes them
duplicate-immune). The test file demonstrates the resulting asymmetry with a
literally duplicated `payload_part`:

```sql
-- test/optimizer/aggregate_reuse.test:298-334 (elided)
CREATE TABLE payload_unproven_part AS SELECT * FROM payload_part UNION ALL SELECT * FROM payload_part;
-- outer_rows now joins through a doubled payload_unproven_part...
-- minimum_cost AS (SELECT f.k, min(f.cost) AS cost FROM payload_fact f WHERE f.k IN (SELECT k FROM outer_rows) GROUP BY f.k)
```
`min(f.cost)` still reuses the materialized `outer_rows` correctly (its own
answer is unaffected by the duplication — `logical_opt` shows `payload_fact`
only once, line 314) even though `outer_rows` itself now emits duplicate
rows per key (the final join result legitimately doubles: `1 1` / `1 1` /
`2 9`×4, lines 328-334, tracking the doubled `part` fan-out exactly). A
sibling test with **two** replication-invariant aggregates together
(`min(f.cost) AS minimum, max(f.cost) AS maximum`) confirms the rewrite
composes (line 337-350), while the **exact same query shape with
`sum(f.cost)`** in place of `min`/`max` is rejected outright — `logical_opt`
shows `payload_fact` twice (lines 369-382), because `SUM`'s
`GetDistinctDependent()` is `DISTINCT_DEPENDENT` and the guard above fires.

This is a deliberately conservative design choice worth naming explicitly:
DuckDB does not attempt to *prove* the extra relation joins 1:1 with the
core relations (e.g. via a primary-key/foreign-key constraint on
`payload_part.k`, which — had it existed and been provably unique — would
have made `SUM` equally safe here). It uses one blanket, syntactic
eligibility test (`NOT_DISTINCT_DEPENDENT`, or explicit `DISTINCT`) that is
always safe regardless of the extra relation's actual cardinality, at the
cost of never applying this rewrite to a duplication-sensitive aggregate even
in the cases where a fan-out proof would have made it safe. The file's final
negative test (lines 385-398, and 403-410) reinforces the same principle
from the opposite direction: if the *core* relation's own filter set differs
between the two sides (`f.cost >= 7` present in `minimum_cost` but absent
from `outer_rows`'s scan of the same table), `SourceFiltersMatch` correctly
refuses the rewrite regardless of which aggregate function is involved,
because the two sides would then be aggregating over genuinely different
row sets.

## 7.12.2 `PARTIAL_AGGREGATE_PUSHDOWN` — pre-aggregating below a join

`PartialAggregatePushdown` is the single largest file in this trio
(`src/optimizer/partial_aggregate_pushdown.cpp`, 1246 lines) because it is
not one rewrite but **three independent sub-mechanisms**, tried in this
fixed order against every `LogicalAggregate`-directly-over-`LogicalComparisonJoin`
shape the visitor finds (`PartialAggregatePushdown::VisitOperator`,
`partial_aggregate_pushdown.cpp:1178-1185`):

```cpp
// src/include/duckdb/optimizer/partial_aggregate_pushdown.hpp:27-33
//! Double-eager (Yan & Larson "eager group-by-count"): pre-aggregate BOTH join inputs by the join key and
//! reconstruct aggregates above by repeating each side's state by the other side's row count.
bool TryDoubleEagerPushdown(unique_ptr<LogicalOperator> &op);
//! Pre-aggregate count on the non-preserved side of a LEFT or RIGHT join.
bool TryOwnerOuterCountPushdown(unique_ptr<LogicalOperator> &op);
//! One-sided eager group-by: push a partial aggregate (via state export) below the join on one side only.
bool TryPushdownAggregate(unique_ptr<LogicalOperator> &op);
```

All three are gated behind the same shape check, `GetPushdownOperators`
(`partial_aggregate_pushdown.cpp:235-253`): the operator directly under the
`LogicalAggregate` must be a `LogicalComparisonJoin` of `INNER`/`LEFT`/`RIGHT`
type, with no projection map, no `DelimGet` anywhere underneath (so
post-decorrelation join trees Chapter 6 built are left alone), and — this
matters for one test below — **a non-empty `GROUP BY` list**
(`aggr->groups.empty()` bails, line 240). A bare, groupless `count(*)` is
therefore never in scope for any of the three mechanisms here; that case
belongs to §7.12.2's closing aside on a completely different pass.

Before walking each mechanism, it's worth resolving the exact question this
book's task set out to answer precisely: **how does this file treat `AVG`,**
which — unlike `SUM`, `COUNT`, `MIN`, `MAX` — cannot be correctly
reconstructed from a single partial scalar without also carrying a partial
count?

### The `AVG` question, resolved: two routes, and both are safe

An earlier pass may split `AVG` into `SUM` and `COUNT`. But it splits a
grouped `AVG` only when the split saves work. A lone grouped `AVG` reaches this
pass unchanged, and this pass pushes it down as a native `avg` state.

`PARTIAL_AGGREGATE_PUSHDOWN` is pass #19 in the fixed pipeline (§7.4). Pass
#4, `AGGREGATE_FUNCTION_REWRITER`, already ran fifteen passes earlier. One of
its rules, `AvgRewriteRule` (§7.6.5 of the overview names it; full code at
`src/optimizer/aggregate_function_rewriter.cpp:70-171`), can rewrite `AVG(x)`
into `SUM(x)` plus a second `COUNT(x)` aggregate. The division moves to a
projection above the aggregate.

The rule does not fire on every numeric `AVG`. It has a second gate,
`ShouldRewrite`, which the rewriter calls for each
matching `AVG` (`aggregate_function_rewriter.cpp:406`):

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:95-142 (elided)
bool ShouldRewrite(const LogicalAggregate &aggr, Expression &expr) override {
	if (aggr.groups.empty()) {
		return true;
	}
	if (optimizer.OptimizerDisabled(OptimizerType::COMMON_AGGREGATE)) {
		return false;
	}
	...
	const auto can_share_count =
	    !optimizer.OptimizerDisabled(OptimizerType::STATISTICS_PROPAGATION) && is_not_null(child);
	for (auto &other : aggr.expressions) {
		// Identical AVGs already share their native state.
		if (expr.Equals(*other)) {
			continue;
		}
		if (sum->Equals(*other) || count->Equals(*other)) {
			return true;
		}
		if (!can_share_count || other->IsVolatile()) {
			continue;
		}
		...
		auto &children = other->Cast<BoundAggregateExpression>().GetChildren();
		if (children.empty() || (children.size() == 1 && is_not_null(*children[0]))) {
			return true;
		}
	}
	return false;
}
```

In plain words:

- An **ungrouped** `AVG` (no `GROUP BY`) is always split.
- A **grouped** `AVG(x)` is split only if the same `GROUP BY` already has a
  `SUM(x)` or `COUNT(x)` it can share. It can also share a `COUNT(*)`, or the
  count of another `AVG`, when `x` is known to be never `NULL`. The split only
  pays off because `COMMON_AGGREGATE` (§7.12.3) later merges the duplicate
  `SUM` or `COUNT`. So the rule does nothing when `COMMON_AGGREGATE` is
  disabled.
- Otherwise the grouped `AVG` stays as one `avg` call.

The test file `test/optimizer/avg_rewrite_profitability.test` shows each
case. A lone grouped `AVG` keeps `avg` in the plan (lines 15-18). Adding
`sum(x)` makes the `avg` disappear (lines 38-41). A scalar `AVG` is still
split (lines 169-173).

So this pass sees `AVG` in one of two forms. Both are handled by the same
generic machinery:

1. **Split form.** Two ordinary `BoundAggregateExpression`s, `SUM` and
   `COUNT`. Each one passes `IsSupportedAggregate`
   (`partial_aggregate_pushdown.cpp:95-117`) on its own, and each one is
   exported and combined like any other aggregate (`ExportAggregateFunction::Bind`,
   `HasStateCombineCallback`, `HasStateFinalizeCallback`). The division in the
   projection above sees two correct totals, so it computes the correct
   average.
2. **Native form.** One `avg` call. Its accumulator state already holds both a
   count and a sum: `AvgState` has the two fields `count` and `value`
   (`extension/core_functions/aggregate/algebraic/avg.cpp:14-20`). Because the
   state declares a `STATE_TYPE`, the generic aggregate builder installs a
   state-type callback for it
   (`src/include/duckdb/function/aggregate_function.hpp:956`).
   `IsSupportedAggregate` only asks for that callback. So the pass exports the
   `avg` state below the join and combines it above the join. The partial count
   travels inside the state. Nothing special is needed.

The test suite checks that the result is the same with the pass on and off:

```sql
-- test/optimizer/partial_aggregate_pushdown.test:204-223 (disabled/enabled pair, elided)
SELECT round(sum(avg_m)::DOUBLE) FROM (
  SELECT d_a, d_b, d_c, d_d, avg(measure)::DOUBLE AS avg_m
  FROM fact JOIN dim USING (dim_key)
  GROUP BY d_a, d_b, d_c, d_d
)
----
49500
```
The result is identical whether `partial_aggregate_pushdown` is disabled or
enabled. Note that this query has a lone grouped `AVG`, so it reaches
this pass in the native form. The un-skipped stress test later in the same
file (`pap_fact`/`pap_dim`, lines 279–330) runs `sum(v), count(v), min(v),
max(v), avg(w)::DOUBLE` *together*, over data built with duplicate dimension
rows and `NULL` measure values. It checks the enabled and disabled result
sets for exact equality with `EXCEPT ALL` in both directions (lines 303-319).
Zero rows differ.

So the worry "does DuckDB need to carry the sum and the count together through
the partial step for AVG?" has a simple answer. Yes, and it already does:
either as two separate aggregates, or inside the one `avg` state. The pass
only needs aggregates whose state it can export and combine.

There is one variant of "average" this file still rejects by name:
**`decimal_average`**. It is a separate catalog function
(`DecimalAverageFun`, `src/include/duckdb/function/aggregate/distributive_functions.hpp:94`,
code in `src/function/aggregate/distributive/decimal_avg.cpp`). It computes an
exact `DECIMAL` average and returns a `DECIMAL`. You get it only by calling
`decimal_average(...)` by name. A plain `avg` over a `DECIMAL` column does
**not** become `decimal_average`: `BindDecimalAvg` keeps the name `avg` and
returns a `DOUBLE`
(`extension/core_functions/aggregate/algebraic/avg.cpp:535-545`). Both `IsSupportedAggregate` and
the double-eager mechanism's `DECanRepeatAggregateState` check the name and
reject it:

```cpp
// src/optimizer/partial_aggregate_pushdown.cpp:95-107 (elided)
static bool IsSupportedAggregate(const BoundAggregateExpression &expr) {
	if (expr.IsDistinct() || expr.GetFilter() || expr.GetOrderBys()) {
		return false;
	}
	if (expr.Function().GetName() == "decimal_average") {
		return false;
	}
	for (auto &child : expr.GetChildren()) {
		if (IsAggregateStateFromFunction(child->GetReturnType(), "decimal_average")) {
			return false;
		}
	}
```
The test file checks this. An explicit `decimal_average(f.m)` call gets no
`combine_aggr` at all (`partial_aggregate_pushdown.test:1067-1070`). The plan
computes it the ordinary way, over the joined rows. The two queries just
before it, `sum(f.m)` and `avg(f.m)` over the same `DECIMAL(38,2)` column
(lines 1057-1065), check something narrower: they only assert that the
*two-argument* `combine_aggr` of the double-eager mechanism is absent. That
is the `INT128` rule explained under sub-mechanism B below, not the
`decimal_average` rule.

### Sub-mechanism A: `TryPushdownAggregate` — one-sided star-schema pushdown

**Purpose:** for a `GROUP BY` immediately over a two-way join, where every
aggregate's argument lives on one side (the "aggregate"/fact side) and the
group-by keys trace back to the other side (the "dimension" side), plus at
least one join-key column — push a smaller, pre-aggregated version of the
fact side below the join, so the join itself processes far fewer rows.

The eligibility analysis, `AnalyzePushdown`
(`partial_aggregate_pushdown.cpp:427-449`), is a chain of six checks worth
naming individually because each corresponds to a real correctness or
cost-based reason to decline:

1. **`FindAggregateSide`** (lines 302-331): every aggregate's argument must
   trace to the *same* side of the join (a `count_star` with no children is
   skipped, contributing no constraint). Mixed-side aggregates disqualify
   the whole rewrite — there is no single "aggregate side" to push.
2. **`JoinPreservesAggregateSide`** (lines 414-425): for an `INNER` join,
   any side may be pushed; for `LEFT`/`RIGHT`, only the *preserved* side may
   be — pushing the *non*-preserved side would pre-aggregate rows before the
   outer join has had a chance to introduce the `NULL`s a correct
   aggregate-over-outer-join must see. The test file checks this exact
   correctness boundary with `sum(d.w)` where `d` is the *non*-preserved
   dimension side of a `LEFT JOIN` — the pushdown never fires
   (`partial_aggregate_pushdown.test:161-175`), on both `LEFT` and `RIGHT`
   phrasings of the same join.
3. **`ContainsAggregateInput`** (lines 207-217): the side to be pushed must
   not already contain another aggregate — no nesting through this path.
4. **A non-empty dimension side** and **`PassesCardinalityHeuristic`**
   (lines 333-354) — the cost gate. Three named constants drive it
   (`PartialAggregatePushdownHeuristics`, lines 33-38):
   `MIN_AGGREGATE_TO_DIMENSION_RATIO = 4` (the fact side must be at least
   4× the dimension side's estimated cardinality — otherwise there's no
   real fan-out to reduce), and `MIN_JOIN_RETENTION = 0.7` (the join must
   retain at least 70% of the aggregate side's estimated rows — if a highly
   selective join predicate is about to discard most of the fact side
   anyway, pre-aggregating it first just duplicates work the join was going
   to do for free). The stress test's "excluded shapes" section confirms
   the retention guard directly: `WHERE d1 = 3 AND d2 = 4` — a highly
   selective filter on the dimension side — is grouped with `count(*)`,
   `FILTER`, and `DISTINCT` as one of the shapes this pass is "intentionally
   excluded or guarded" against (`partial_aggregate_pushdown.test:388-408`),
   even though correctness there is verified by an `EXCEPT ALL` diff, not by
   an `EXPLAIN` regex — this is a cost decision, not a correctness one.
5. **`ValidateJoinConditions`** (lines 381-394): every join condition must
   be a plain equality with a direct column reference on the aggregate
   side.
6. **`HasDimensionGroup`** (lines 396-412): at least one `GROUP BY` column
   must actually come from the dimension side — otherwise there's nothing
   the join is adding to the grouping that pre-aggregating the fact side
   alone would already capture below the join.

Then, separately, **`PassesLowerGroupHeuristic`**
(`partial_aggregate_pushdown.cpp:469-472`) gates the *shape* of the
pre-aggregate itself: the lower (pushed) aggregate's group list — join keys
plus at most `MAX_EXTRA_LOWER_GROUPS = 1` extra fact-side column
(`CollectLowerSideGroups`, lines 273-300) — must be *strictly smaller* than
the original `GROUP BY` list. Two tests demonstrate exactly why this
matters: grouping only by the dimension join key itself
(`GROUP BY dim.dim_key`, `test.test:76-81`) makes the lower aggregate's
group list the same size as the upper one's — no reduction, so the pass
declines (`<!REGEX>:.*combine_aggr.*`); and replacing one grouping key with
a different one of the same width (`GROUP BY d_a` instead of the original
four columns, `test.test:84-90`) similarly fails to shrink the group
cardinality enough to offset running two aggregate passes instead of one.

There is one more gate: for an `INNER` join, the pass gives up if any
aggregate argument can raise an error. See "A guard for A and B" at the end
of §7.12.2.

**What actually gets built**, once eligible
(`TryPushdownAggregate`, lines 1196-1244; helpers `BuildLowerGroupMap`,
`BindPushdownAggregates`, `CreateLowerAggregate`, `CreateJoin`,
`CreateUpperAggregate`, `BuildFinalProjection`):

```
before:                              after:
LogicalAggregate                     LogicalProjection  (finalize state -> scalar)
  (GROUP BY dim cols;                  └─ LogicalAggregate
   SUM/COUNT/MIN/MAX(fact.x))              (GROUP BY dim cols;
  └─ LogicalComparisonJoin                  combine_aggr(partial))
       (fact.key = dim.key)               └─ LogicalComparisonJoin
       ├─ fact scan                            (fact_lower.key = dim.key)
       └─ dim scan                            ├─ LogicalAggregate  (fact side)
                                              │    (GROUP BY key[, +1 extra];
                                              │     exported SUM/COUNT/MIN/MAX(x))
                                              │    └─ fact scan
                                              └─ dim scan
```

`BindPushdownAggregates` (lines 474-499) exports each aggregate's state via
`ExportAggregateFunction::Bind`, and pairs it with a `combine_aggr` (1-arg
form) at the upper level — the same generic export/combine pair
§7.12.1's SEMI-reuse mechanism uses, confirming this is one shared idiom
across both passes, not two independent implementations of the same idea.

**`ROLLUP`/`GROUPING SETS` survive the rewrite**, because
`CreateUpperAggregate` copies `aggr.grouping_sets`/`grouping_functions`
verbatim onto the *upper* aggregate (lines 570-571) — the lower,
pushed-down aggregate always groups by the full (non-rolled-up) key set, and
the upper aggregate, sitting above the reconstituted join, is the one that
still expands into multiple grouping-set rows exactly as the original query
asked. The test file checks this at real scale (`pap_fact`/`pap_dim`,
`GROUP BY ROLLUP(d1, d2, d3, d4)`, lines 335-386): enabled and disabled
results diff to zero rows in both directions.

**Recursive application.** `VisitOperator` deliberately re-visits a
rewritten subtree once any of the three mechanisms fires
(`partial_aggregate_pushdown.cpp:1181-1184`, "Revisit rewritten subtrees so
nested aggregate-over-join shapes can be pushed too"), which is what lets an
**n-way** fact/fact/fact join get pushed one join at a time, recursively:

```sql
-- test/optimizer/partial_aggregate_pushdown.test:820-824
SELECT f1.keyid AS keyid, count(*) AS c, sum(m1) AS s1, sum(m2) AS s2, sum(m3) AS s3,
       min(m1) AS mn, max(m3) AS mx, count(m2) AS cm
FROM nw_f1 f1 JOIN nw_f2 f2 USING (keyid) JOIN nw_f3 f3 USING (keyid)
GROUP BY f1.keyid
```
verified exact (integer measures) via `EXCEPT ALL` both ways (lines 838-853),
and `EXPLAIN` on the two-aggregate subset confirms `combine_aggr` appears at
all (line 859-865) — worth noting this is the one place in §7.4's fixed,
run-once pipeline architecture where a pass *does* effectively get a second
shot at its own output within a single invocation, by design, rather than
relying on the pipeline looping back around.

**A genuine test-format aside, worth explaining once since it recurs
throughout this file:** several blocks pair two `query ... rowsort <label>`
statements under the *same* label with **no literal expected rows written
in the file at all** (e.g. `partial_aggregate_pushdown.test:977` and `:993`,
both tagged `de_num_ab`). This is the sqllogictest harness's own paired-label
mechanism: two blocks sharing a label are asserted to produce byte-identical
result sets *against each other*, without the test author writing out the
actual values — the same underlying idea as the book's own convention of
pairing a disabled-optimizer run against an enabled one, just expressed as
one label instead of an explicit `EXCEPT ALL` query. Wherever this pattern
appears below, read it as "these two queries must agree, values unwritten."

**Projection fusion.** A computed expression between the aggregate and the
join (e.g. a subquery projecting `c * 2.0 AS c2`) would normally block
`GetPushdownOperators`, since its immediate child is a `LogicalProjection`,
not a `LogicalComparisonJoin`. `FuseInterveningProjections`
(`partial_aggregate_pushdown.cpp:1137-1176`) inlines any chain of
non-volatile projections sitting directly above the join into the
aggregate's own group/expression list first, so the join becomes the
aggregate's direct child again and the rest of the pass can proceed
normally. The test comments this precisely: "Without the fusion this query
gets no pushdown at all" (`test.test:764-766`), and separately flags that
this specific fused case's plan shape is **vector-size-dependent** — its
estimated group count straddles the 512/2048 boundary the pass's own
gating heuristic checks, so the test is guarded with `require vector_size
2048` (lines 1373-1387) and deliberately placed last in the file so every
size-independent check above it still runs at every configured vector size.

### Sub-mechanism B: `TryDoubleEagerPushdown` — pre-aggregating *both* sides

**Purpose:** when *neither* side of the join is a small dimension — both
sides have substantial fan-out per join key — pre-aggregate *both* sides
down to (join key, `count_star`, exported partial states) tuples, join the
two much smaller pre-aggregates instead of the original tables, and
reconstruct each original aggregate by combining its own side's exported
state with **the other side's row count as an explicit multiplicity**. The
header cites the technique's provenance directly: *"Double-eager (Yan &
Larson `eager group-by-count`)"* (`partial_aggregate_pushdown.hpp:27-28`).
The technique comes from Yan and Larson's paper on eager and lazy
aggregation [21].

The cost gate here is a genuine estimation, `DEEstimateCollapse`
(`partial_aggregate_pushdown.cpp:672-687`): it estimates the join's number
of distinct keys as `ndv = (c0 * c1) / cj` (clamped to at most either side's
own cardinality), then requires **both** sides' cardinality to be at least
`min_collapse × ndv`, where `min_collapse` is 2 for a "small" key domain (≤
`2 × STANDARD_VECTOR_SIZE` distinct keys) or 8 for a larger one
(`DoubleEagerHeuristics`, lines 666-670) — i.e. both inputs must show real,
substantial repetition per key before the two extra aggregate passes plus
their join plus the final combine are judged worth it over just running the
original join once. The file's own "modest collapse" negative test makes
this concrete: two 30,000-row tables joining through a 5,000-row dimension
(6 rows per key per side) fail this ratio and the plan shows no
`combine_aggr` (`test.test:576-604`), while the main double-eager
demonstration (`de_a`/`de_b`, 100 rows per key each through a 50-row
dimension) clears it easily (lines 457-514).

The **reconstruction step** is the part worth reading precisely, because
it's the one place `combine_aggr`'s *two-argument* form is used
(`Parameters = "arg,multiplicities"`, `distributive_functions.hpp:106`):

```cpp
// src/optimizer/partial_aggregate_pushdown.cpp:852-864 (elided)
for (auto &de : aggregates) {
	vector<unique_ptr<Expression>> arg;
	arg.push_back(partial_ref(de));
	arg.push_back(cnt_ref(1 - de.side));
	auto upper = DEBindCombineAggr(optimizer.context, std::move(arg));
	...
```

The first argument, `partial_ref(de)`, is this side's exported partial state.
The second, `cnt_ref(1 - de.side)`, is the other side's precomputed
`count_star`.

Every original aggregate's argument lived on one side only; that side's
pre-aggregated partial state is combined **once per row the other side
contributed for that key** — exactly reproducing the row multiplicity the
original, un-pushed join would have produced (`count_a × count_b` rows per
key, each carrying the same underlying value from whichever side didn't
change). A dedicated test stresses this with genuinely different
per-key multiplicities and a holistic, state-heavy aggregate (`median`),
specifically to make "incorrect state repetition observable":

```sql
-- test/optimizer/partial_aggregate_pushdown.test:519-538 (elided)
CREATE TABLE de_list_multiplicity AS SELECT CASE WHEN i<30 THEN 1 ELSE 2 END AS k, 1 AS g FROM range(50) _(i);
CREATE TABLE de_list_values       AS SELECT CASE WHEN i<20 THEN 1 ELSE 2 END AS k,
       CASE WHEN i<20 THEN 1 ELSE 42 END AS v FROM range(40) _(i);

SELECT m.g, median(v.v)
FROM de_list_multiplicity m INNER JOIN de_list_values v USING (k)
GROUP BY m.g
```
`median`, `quantile_cont`, `mad`, and `list`/`list_count`/`list_sum` all
appear in the same follow-up query (lines 542-554) and match, on and off,
via `combine_aggr` — this is the concrete demonstration that this
mechanism is not restricted to arithmetic-decomposable functions the way
`SUM`/`COUNT` are: **any** aggregate whose accumulator state supports
combine/finalize (`HasStateCombineCallback`/`HasStateFinalizeCallback`) can
ride this same generic machinery, including genuinely holistic ones like
`MEDIAN` that keep their entire input as an internal sorted structure — the
"decomposability" this pass actually requires is a property of the
*internal accumulator state*, not of the finalized scalar value, which is
exactly why the naive SQL intuition ("SUM/COUNT/MIN/MAX decompose, AVG needs
care, MEDIAN can't decompose at all") doesn't map cleanly onto what this
code checks.

**The one real, source-verified type restriction — and a genuine,
unexplained asymmetry.** `DECanRepeatAggregateState`
(`partial_aggregate_pushdown.cpp:689-704`) additionally excludes `sum`/`avg`
whose argument's physical type is `PhysicalType::INT128` (signed
`HUGEINT`, or `DECIMAL(38,*)`, which is stored as a 128-bit integer) from
this *specific* mechanism:

```cpp
// src/optimizer/partial_aggregate_pushdown.cpp:689-704 (elided)
static bool DECanRepeatAggregateState(const BoundAggregateExpression &aggr) {
	auto &name = aggr.Function().GetName();
	if (name == "decimal_average") {
		return false;
	}
	...
	if ((name != "sum" && name != "avg") || aggr.GetChildren().empty()) {
		return true;
	}
	auto &type = aggr.GetChildren()[0]->GetReturnType();
	return type.InternalType() != PhysicalType::INT128;
}
```
The test file's "Exact no-headroom sum states are not repeated" section
demonstrates the reason directly: a `HUGEINT` column engineered with values
near the type's own maximum magnitude (`85000000000000000000000000000000`)
sums correctly *once* (`test.test:932-936`), but **double-eager specifically**
is skipped for both `sum` and `avg` over it (lines 938-948, `<!REGEX>` on the
two-argument `combine_aggr` form) — because this mechanism's combine step
literally *repeats* a partial sum's contribution once per matching row on
the other side, and an accumulator already near `INT128`'s own headroom has
no margin left to survive being effectively multiplied. `MIN`/`MAX` over the
same `HUGEINT` column, by contrast, **do** still use double-eager (lines
950-954) — repetition can't overflow a value that's just being compared,
never summed. Note that `TryPushdownAggregate` (the one-sided mechanism
above) has *no* such `INT128` restriction, because its `combine_aggr` call
is always the plain, 1-argument form — no multiplicity, no repetition, no
overflow risk to guard against.

One more concrete, source-verified detail this book states rather than
smooths over: the same test file shows **`UHUGEINT`** (unsigned 128-bit)
`sum` *does* engage double-eager (`test.test:1086-1089`,
`<REGEX>:.*Aggregates: combine_aggr...`) even though it is exactly as wide
as the excluded signed `HUGEINT`. `DECanRepeatAggregateState`'s check is
literally `type.InternalType() != PhysicalType::INT128` — `UHUGEINT`'s
physical type is a *different* enumerator (unsigned 128-bit), so it simply
never matches this specific exclusion. Whether this is a deliberate
judgment that unsigned 128-bit addition has enough headroom in practice, or
an asymmetry nobody has revisited, isn't something the source states either
way — this book flags it as observed, precisely reproduced from both the
code and the test, rather than guessing at an intent behind it.

### Sub-mechanism C: `TryOwnerOuterCountPushdown` — `COUNT` below a preserved-side outer join

**Purpose:** for `COUNT`/`COUNT(*)` grouped by the single column that is the
*preserved* ("owner") side of a `LEFT`/`RIGHT` join, pre-aggregate a partial
count on the *other* ("fact"/event) side before the join, then recombine
with a plain `SUM(COALESCE(partial_count, 0))` above the join — correctly
giving an owner row with **no** matching fact rows a count of `0`, exactly
as an ordinary `LEFT JOIN` + `COUNT` would.

This is the narrowest of the three mechanisms (single grouping column only,
`aggr.groups.size() != 1` bails at `partial_aggregate_pushdown.cpp:988`;
`COUNT`/`COUNT(*)` only, checked by literal function-name string at lines
1020-1026), but it is also the mechanism most worth reading closely, because
it does **not** use the export/`combine_aggr` machinery at all:

```cpp
// src/optimizer/partial_aggregate_pushdown.cpp:1056-1059 (elided)
vector<unique_ptr<Expression>> lower_aggregates;
lower_aggregates.push_back(aggregate.Copy());
auto lower = make_uniq<LogicalAggregate>(lower_group_index, lower_aggregate_index, std::move(lower_aggregates));
lower->groups.push_back((*aggregate_key_expr)->Copy());
```
`aggregate.Copy()` is the original `count`/`count_star` call. It is not
exported.
`COUNT`'s own **finalized** value — an ordinary `BIGINT` — is already
directly re-summable: `SUM` of per-key counts computed over disjoint subsets
of rows equals the count over their union, by elementary arithmetic, with
no accumulator-state export needed at all. The only genuine subtlety this
mechanism has to handle is the **outer join's own NULL-padding**: an owner
row with no matching fact rows produces, after a `LEFT JOIN`, one row whose
fact-side columns are all `NULL` — and the pre-aggregated partial count for
a key that never appears on the fact side is *absent* from the lower
aggregate entirely, not present-and-zero. The upper aggregate handles this
with an explicit `COALESCE`:

```cpp
// src/optimizer/partial_aggregate_pushdown.cpp:1075-1077
auto count_value = make_uniq<BoundOperatorExpression>(ExpressionType::OPERATOR_COALESCE, count_type);
count_value->GetChildrenMutable().push_back(std::move(count_ref));
count_value->GetChildrenMutable().push_back(make_uniq<BoundConstantExpression>(Value::BIGINT(count_star ? 1 : 0)));
```
— `COALESCE(partial_count, 0)` for `COUNT(column)`, or `COALESCE(partial_count,
1)` for `COUNT(*)` (an owner row with zero matching fact rows still
contributes exactly one output row of the outer join itself, so `COUNT(*)`'s
"missing" case is `1`, not `0` — precisely why the pass tracks `count_star`
as a distinct case at all). `EXPLAIN`'s physical plan makes this visible
directly, matching a very specific fingerprint:

```sql
-- test/optimizer/partial_aggregate_pushdown.test:1159-1164
EXPLAIN SELECT o.k, count(e.event_id)
FROM outer_count_owner o LEFT JOIN outer_count_events e ON o.k = e.k
GROUP BY o.k
----
physical_plan	<REGEX>:(?s).*COALESCE.*Hash Join.*Hash Group By.*
```
— a `COALESCE` sitting *above* the hash join, which itself sits above a
*genuine* `Hash Group By` (the pre-aggregated partial count on the event
side) — and correctness is checked both against a hand-computed answer
(`1  30` / `2  30` / `3  30` / `4  0`, lines 1169-1177, where owner `k=4` has
no matching events at all) and, more rigorously, against the *same query run
with the entire optimizer disabled* (`PRAGMA disable_optimizer`, lines
1188-1207) via a bidirectional `EXCEPT ALL` diff.

The remaining eligibility guards are worth naming because each maps to a
concrete test:

- **Owner-side statistics must be reliable.** The pass requires an *exact*
  or *HLL*-sourced distinct-count estimate for the join key on the
  aggregate side (`key_stats->distinct_count.source`,
  `partial_aggregate_pushdown.cpp:1036-1041`) and the same
  `MIN_AGGREGATE_TO_DIMENSION_RATIO`/`MIN_JOIN_RETENTION` cost gates
  §7.12.2 sub-mechanism A already introduced. A synthetic "sparse events"
  table, where the owner join would discard most event rows, correctly
  fails the retention guard (`test.test:1307-1313`); a synthetic key domain
  where most values are unique (`CASE WHEN i < 7000 THEN 1 ELSE i END`)
  correctly fails the aggregate-to-dimension ratio (lines 1315-1325),
  because the "dimension" here is really a near-unique key domain, not a
  small owner table.
- **Owner uniqueness is *not* required.** A duplicated owner table
  (`outer_count_unproven`, built the same `UNION ALL` way §7.12.1's
  duplicate-owner test did) still gets the correct, doubled answer (`1  60`
  / `2  60` / `3  60` / `4  0`, lines 1262-1270) — the upper `SUM` naturally
  preserves owner-row multiplicity exactly the way §7.12.1's `combine_aggr`
  did, just via ordinary `SUM` rather than an aggregate-state combine,
  because `COUNT`'s finalized value is already summable.
- **`NULL` owner keys behave exactly as an ordinary `LEFT JOIN` would.** A
  `NULL` owner key can never equal anything by SQL's own equality
  semantics, so it always gets `count = 0` in both the rewritten and
  un-rewritten plan, with no special-casing needed in the rewrite itself
  (`test.test:1334-1349`).
- **A composite `GROUP BY`, `DISTINCT`, or a duplication-sensitive
  aggregate all correctly decline.** `GROUP BY o.k, o.label` (more than one
  grouping column), `count(DISTINCT e.event_id)`, and `sum(e.event_id)` in
  place of `count` are each rejected by one of the guards named above (single
  grouping column, non-`DISTINCT`, `COUNT`-only). Each test confirms no
  `COALESCE` appears in the resulting plan (lines 1351-1370).
- **An extra event-side condition does not block the rewrite.**

  The test expects the rewrite to fire for `ON o.k = e.k AND e.keep`
  (`partial_aggregate_pushdown.test:1327-1332`, `<REGEX>:.*COALESCE.*`).
  `e.keep` reads only the event side. Filter pushdown moves such a condition
  out of the `LEFT JOIN` and into the event-side input before this pass runs,
  so the join is left with the single key equality. Whether the rewrite then
  fires depends only on the cost gates above, which read estimated row
  counts. (This book infers the reason from the test. It did not trace the
  cardinality code.)

### A guard for A and B: an argument that can throw stays above an inner join

Without this guard, the pass could make a correct query fail with an error
(DuckDB issue #28568).

**The problem.** Pushing an aggregate below a join also pushes its argument
expressions. The lower aggregate then runs on *every* row of its side. An
inner join would have removed some of those rows. If the argument can raise
an error, it could then raise it on a row the query never meant to touch. The
test file `test/optimizer/partial_aggregate_pushdown_throwing_argument.test`
shows this with a cast:

```sql
-- test/optimizer/partial_aggregate_pushdown_throwing_argument.test:5-10, 32-36
CREATE TABLE dim AS SELECT i AS id, i % 5 AS a, i % 3 AS b FROM range(100) t(i) WHERE i <> 50;

# The fact rows with id = 50 have no dimension row and hold a value that does not convert to INTEGER.
CREATE TABLE fact AS SELECT i % 100 AS id, CASE WHEN i % 100 = 50 THEN 'oops' ELSE (i % 7)::VARCHAR END AS v FROM range(10000) t(i);

# An argument that can throw keeps the aggregate above the inner join.
EXPLAIN SELECT a, b, sum(CAST(v AS INTEGER)) FROM fact JOIN dim ON fact.id = dim.id GROUP BY a, b
----
physical_plan	<!REGEX>:.*combine_aggr.*
```

The rows with `id = 50` hold `'oops'`. The join removes them, because `dim`
has no row with `id = 50`. So the query as written never casts `'oops'`. But
an unguarded pushdown would build this plan:

```
Perfect Hash Group By: combine_aggr(#2)
└─ Hash Join (INNER) id = #0
   ├─ Seq Scan dim
   └─ Perfect Hash Group By: sum(#1) EXPORT_STATE
      └─ Projection: #0, CAST(v AS INTEGER)
         └─ Seq Scan fact
```

Here the cast runs on all of `fact`, before the join. The query would fail
with `Conversion Error: Could not convert string 'oops' to INT32`. The same
would happen with arithmetic that can overflow, such as `sum(v * 2)` when an
unmatched row holds the `BIGINT` maximum.

**The guard.** Before it pushes an aggregate below an **inner** join, the pass
asks whether any aggregate argument can throw:

```cpp
// src/optimizer/partial_aggregate_pushdown.cpp:194-205
// Below an inner join a throwing argument would also run on the rows the join removes
static bool AggregateArgumentsCanThrow(Optimizer &optimizer, unique_ptr<LogicalOperator> &op) {
	auto &aggr = op->Cast<LogicalAggregate>();
	for (auto &expr : aggr.expressions) {
		for (auto &child : expr->Cast<BoundAggregateExpression>().GetChildren()) {
			if (ArgumentCanThrow(optimizer, op, *child)) {
				return true;
			}
		}
	}
	return false;
}
```

It is called in two places:

- in `TryPushdownAggregate` (sub-mechanism A), only for an `INNER` join
  (`partial_aggregate_pushdown.cpp:1215-1218`);
- in `TryDoubleEagerPushdown` (sub-mechanism B)
  (`partial_aggregate_pushdown.cpp:930-932`). Double-eager only handles
  inner joins.

A `LEFT` or `RIGHT` join that keeps the aggregate side is not affected. That
join keeps every row of that side anyway, so the argument runs on the same
rows with or without the pushdown. The source says so in a comment:

```cpp
// src/optimizer/partial_aggregate_pushdown.cpp:1215-1218
// A preserved side of an outer join keeps every row, so the argument is evaluated on all of them either way.
if (join->join_type == JoinType::INNER && AggregateArgumentsCanThrow(optimizer, op)) {
	return false;
}
```

Sub-mechanism C pushes only `count`/`count_star` over a plain column, so it
has no argument that can throw.

**Using statistics to keep good pushdowns.** A plain "can this expression
throw?" check would be too strict. For example, `price * qty` over `DECIMAL`
columns is marked as able to overflow, but the column values may be far too
small to ever overflow. So `ArgumentCanThrow` takes a second look:

```cpp
// src/optimizer/partial_aggregate_pushdown.cpp:170-192
// An argument that can throw is safe when the bounds of the base table columns it references prove it cannot fail
static bool ArgumentCanThrow(Optimizer &optimizer, unique_ptr<LogicalOperator> &aggr, const Expression &argument) {
	if (!argument.CanThrow()) {
		return false;
	}
	column_binding_map_t<unique_ptr<BaseStatistics>> column_statistics;
	ExpressionIterator::VisitExpression<BoundColumnRefExpression>(argument, [&](const BoundColumnRefExpression &ref) {
		if (column_statistics.find(ref.Binding()) != column_statistics.end()) {
			return;
		}
		auto stats = GetBaseColumnStatistics(optimizer.context, *aggr->children[0], ref.Binding());
		if (stats) {
			column_statistics[ref.Binding()] = std::move(stats);
		}
	});
	if (column_statistics.empty()) {
		return true;
	}
	auto copy = argument.Copy();
	StatisticsPropagator propagator(optimizer, aggr);
	propagator.PropagateExpressionStatistics(copy, std::move(column_statistics));
	return copy->CanThrow();
}
```

Step by step:

1. If the expression cannot throw at all, it is safe.
2. Otherwise, for each column the expression reads, find the statistics of
   the base table column behind it. `GetBaseColumnStatistics`
   (`partial_aggregate_pushdown.cpp:128-168`) walks down through projections
   to the `LogicalGet` and asks the table function for statistics. It marks
   the result as "can be `NULL`", because an outer join above the scan can
   add `NULL`s.
3. If no statistics are found, assume the worst: it can throw.
4. Otherwise run statistics propagation on a copy of the expression, with
   those column bounds as input. Propagation can clear the "can throw" mark
   when the bounds prove the operation is safe (for example, a multiplication
   whose result always fits). Check the copy again.

`CanThrow()` is also exact for some functions at bind time. `DECIMAL` `+`,
`-` and `*` widen their result type so that they cannot overflow, and say
so; they stay marked only when the width is clamped at the `DECIMAL` maximum.
A `DOUBLE` division under the default IEEE semantics returns `inf` or `nan`
instead of an error, so it does not count as able to throw either. (This
part is in the binder and function code, not in this pass. It keeps the
pushdown firing for common TPC-DS queries.)

The test file walks through each case. These keep the aggregate above the
join (`<!REGEX>:.*combine_aggr.*`):

- `sum(CAST(v AS INTEGER))` over the `'oops'` data (lines 32-36);
- `sum(v * 2)` where an unmatched row holds the `BIGINT` maximum
  (lines 93-96);
- a `DOUBLE` division with `ieee_floating_point_ops = false`, and integer
  division (from line 137);
- a clamped `DECIMAL(10,0) * DECIMAL(10,0)` where an unmatched row holds a
  value whose square does not fit (from line 220);
- the same clamped arithmetic over a `UNION ALL` input, where no base table
  statistics can be found (from line 259).

These are still pushed down (`<REGEX>:.*combine_aggr.*`):

- `sum(TRY_CAST(v AS INTEGER))`, because `TRY_CAST` returns `NULL` instead
  of failing (lines 38-42);
- the same `CAST` under a `LEFT JOIN` that keeps `fact` (lines 63-67);
- `sum(v)`, a plain column (lines 98-102);
- unclamped `DECIMAL` arithmetic and IEEE `DOUBLE` division (from line 104);
- clamped `DECIMAL` arithmetic whose column bounds prove it cannot overflow
  (lines 154-158, and from line 237), also through a filter below the join
  (lines 253-257).

The double-eager path gets its own check of a throwing argument (from line
161). That block disables `join_filter_pushdown` first, because a runtime
join filter would hide the error on the unmatched rows.

A second test, `test/optimizer/partial_aggregate_pushdown_serialized_plan.test`,
checks that a pushed-down `sum(price - discount)` over `DECIMAL` columns still
has `combine_aggr` after the plan is serialized and read back
(`debug_verify_serializer`). This matters because the "cannot overflow"
decision is made at bind time and must survive the round trip.

## 7.12.3 `COMMON_AGGREGATE` — deduplicating identical aggregate calls within one `GROUP BY`

**Purpose, in one sentence:** if a single `LogicalAggregate` requests the
same aggregate computation more than once — the same function, the same
arguments in the same order, the same `DISTINCT`/`FILTER`/`ORDER BY`
modifiers — `CommonAggregateOptimizer` computes it once and re-points every
reference at the single surviving copy.

This is the smallest of the three files (`src/optimizer/common_aggregate_optimizer.cpp`,
85 lines) and the most mechanically direct. `ExtractCommonAggregates`
(lines 46-83) walks one `LogicalAggregate`'s expression list once, keyed by
an `expression_map_t<ProjectionIndex>` (`aggregate_remap`, line 47):

```cpp
// src/optimizer/common_aggregate_optimizer.cpp:49-81 (elided)
for (idx_t i = 0; i < aggr.expressions.size(); i++) {
	ProjectionIndex original_index(i + total_erased);
	ProjectionIndex new_index(i);
	// volatile aggregates must not be deduplicated: each call is independent
	if (aggr.expressions[i]->IsVolatile()) {
		...
		continue;
	}
	auto entry = aggregate_remap.find(*aggr.expressions[i]);
	if (entry == aggregate_remap.end()) {
		// aggregate does not exist yet: add it to the map
		aggregate_remap[*aggr.expressions[i]] = new_index;
		...
	} else {
		// aggregate already exists! we can remove this entry
		total_erased++;
		aggr.expressions.erase_at(i);
		i--;
		// we need to remap any references to this aggregate so they point to the other aggregate
		ColumnBinding original_binding(aggr.aggregate_index, original_index);
		ColumnBinding new_binding(aggr.aggregate_index, entry->second);
		aggregate_map[original_binding] = new_binding;
	}
}
```

**How "identical" is decided.** `expression_map_t` is keyed by
`Expression::Hash()`/`Expression::Equals()` (`src/include/duckdb/parser/expression_map.hpp:20-32`)
— the same generic map type Chapter 7's `COMMON_SUBEXPRESSIONS` (`cse_optimizer.cpp`)
and `COMMON_SUBPLAN` passes use, at their respective, narrower/wider scopes.
For a `BoundAggregateExpression` specifically, the override compares the
**aggregate type** (plain vs. `DISTINCT`), the **bound function identity**,
**every child expression pairwise, in argument order**
(`bound_aggregate_expression.cpp:91-100` — no permutation is attempted),
the **collation of each child's type** (new, see below), the **`FILTER`
clause**, the **state-export mode**, the **bind data**, and the **`ORDER BY`
modifier**:

```cpp
// src/planner/expression/bound_aggregate_expression.cpp:74-111
bool BoundAggregateExpression::Equals(const BaseExpression &other_p) const {
	if (!Expression::Equals(other_p)) {
		return false;
	}
	auto &other = other_p.Cast<BoundAggregateExpression>();
	if (other.aggr_type != aggr_type) {
		return false;
	}
	if (other.function != function) {
		return false;
	}
	if (children.size() != other.children.size()) {
		return false;
	}
	if (!Expression::Equals(other.filter, filter)) {
		return false;
	}
	for (idx_t i = 0; i < children.size(); i++) {
		if (!Expression::Equals(*children[i], *other.children[i])) {
			return false;
		}
		// expression equality ignores the return type, but a collation lives in the return type and decides
		// which argument values compare equal - two aggregates that disagree on it are not the same aggregate
		if (!children[i]->GetReturnType().EqualsIncludingCollation(other.children[i]->GetReturnType())) {
			return false;
		}
	}
	if (state_export_mode != other.state_export_mode) {
		return false;
	}
	if (!FunctionData::Equals(bind_info.get(), other.BindInfo().get())) {
		return false;
	}
	if (!BoundOrderModifier::Equals(order_bys, other.order_bys)) {
		return false;
	}
	return true;
}
```

> **Why the collation check.** Plain expression equality ignores the return
> type. A collation, such as `NOCASE`, lives in the return type. So without
> the collation check inside the loop, `list(DISTINCT s COLLATE NOCASE)` and
> `list(DISTINCT s)` could look like the same aggregate, and one could be
> merged into the other. They are not the
> same: with `NOCASE`, `'a'` and `'A'` are duplicates; without it, they are
> not. The test `test/sql/collate/collate_distinct_aggregate.test:25-30`
> checks this directly:
>
> ```sql
> SELECT len(list(DISTINCT s COLLATE NOCASE)), len(list(DISTINCT s)),
>        len(list(DISTINCT s COLLATE NOCASE))
> FROM tbl;
> ----
> 3	5	3
> ```
>
> The first and third calls are identical and may share one computation. The
> second one must stay separate. The same file also checks `ROW` and `STRUCT`
> arguments that contain a collated field.

Directly answering the task's question: **this is exact, ordered
syntactic-tree equality on the already-bound expression, not a
commutativity-aware comparison.** `SUM(a + b)` and `SUM(b + a)` are two
different `Expression` trees under this check (children compared
positionally, no sorting), so unless *some other, earlier pass* has already
canonicalized their argument order (the arithmetic-simplification rules in
§7.5 handle some constant/non-constant reordering, but nothing in the fixed
pipeline generally sorts the operands of an arbitrary commutative
expression), `COMMON_AGGREGATE` would not recognize them as the same
aggregate. What it *does* recognize as trivially identical is exactly what
you'd expect from bind-time canonicalization rather than any cleverness in
this file: `COUNT(*)` and `COUNT()` bind to the same zero-argument
`count_star` catalog function at parse/bind time, so they are syntactically
identical `BoundAggregateExpression`s before this pass ever runs.

**The one genuinely non-obvious identity this pass benefits from, sourced
from a completely different pass.** The test file's very first case claims
`COUNT(*)`, `COUNT()`, *and* `COUNT(i)` all fold into one aggregate:

```sql
-- test/sql/optimizer/expression/test_common_aggregate.test:18-21
EXPLAIN SELECT COUNT(*), COUNT(), COUNT(i) FROM integers;
----
logical_opt	<!REGEX>:.*AGGREGATE.*count.*count.*
```
`COUNT(i)` is, on its face, a genuinely different bound function
(`count`, with one child) from `count_star` — so this can't be
`COMMON_AGGREGATE` alone. The missing piece is `STATISTICS_PROPAGATION`
(pass #34, which runs three passes *before* `COMMON_AGGREGATE` at #37): the
`count` aggregate function registers its own statistics callback that
rewrites itself into `count_star` in place, whenever the input column is
provably non-`NULL`:

```cpp
// src/function/aggregate/distributive/count.cpp:278-286
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
With no `NULL`s yet inserted into `integers.i`, propagated statistics prove
`i` can't be `NULL`, `CountPropagateStats` mutates `COUNT(i)` into a
genuine, zero-child `count_star` node *in place*, and by the time
`COMMON_AGGREGATE` runs three passes later, all three expressions really
are syntactically identical. The test proves this staging directly by
inserting a `NULL` and re-running the identical query:

```sql
-- test/sql/optimizer/expression/test_common_aggregate.test:40-47
INSERT INTO integers VALUES (NULL);
EXPLAIN SELECT COUNT(*), COUNT(), SUM(i), COUNT(i), SUM(i) / COUNT(i) FROM integers;
----
logical_opt	<REGEX>:.*AGGREGATE.*count.*count.*   -- now TWO counts survive
```
— once a `NULL` exists, `CanHaveNull()` is true, `CountPropagateStats`
leaves `COUNT(i)` alone, and `COMMON_AGGREGATE` correctly keeps it as a
second, genuinely distinct aggregate alongside the merged `COUNT(*)`/
`COUNT()`, since `COUNT(i)` (`3`) and `COUNT(*)` (`4`) now legitimately
differ (line 49-52: `4  4  6  3  2`).

**Scoping to a single `LogicalAggregate`, deliberately.** `VisitOperator`
(`common_aggregate_optimizer.cpp:16-34`) starts a **fresh**
`CommonAggregateOptimizer` (a fresh, empty `aggregate_map`) at every
`UNION`/`EXCEPT`/`INTERSECT`/`LogicalProjection` boundary it crosses — so
two aggregates that happen to look identical but sit in *different*
`GROUP BY`s (say, two branches of a `UNION ALL`, or a correlated aggregate
inside a subquery versus one outside it) are never merged across that
boundary, even if they're textually identical, because they may be
computed over entirely different input rows. Deduplication only ever
happens *within* one `LogicalAggregate`'s own expression list.

**A trap: readers that find columns by position.** When this pass drops a
duplicate aggregate, every column after it moves one slot to the left. The
pass fixes the readers through `aggregate_map`, which maps old *bindings*
(table index, column index) to new ones. That works for any parent that
reads columns by binding. It does not work for a parent that reads columns
by *position*. A materialized CTE definition is one such case: its readers
read "column 0, column 1, ..." of the CTE result.

> **A positional reader needs a projection in between.** The
> `GROUPING_SETS` pass can turn an aggregate directly into the definition of
> a materialized CTE. Decorrelation may add a second, identical `count(*)`
> to that aggregate. If `COMMON_AGGREGATE` merged the two there, the CTE
> would get one column fewer, but its readers would still read the old
> positions. The query would fail with `INTERNAL Error: Attempted to access
> index 2 within vector of size 2`. The guard is in `GROUPING_SETS`, not in
> this pass. It puts an identity projection on top of the aggregate before it becomes the CTE
> definition:
>
> ```cpp
> // src/optimizer/grouping_sets_optimizer.cpp:436-438
> // cap the aggregate with a projection so later binding remaps cannot change the CTE's column layout
> auto cte_definition =
>     LogicalProjection::CreateIdentity(optimizer.binder.GenerateTableIndex(), std::move(level.aggregate));
> ```
>
> The projection reads the aggregate by binding, so the remap reaches it. The
> CTE's readers read the projection by position, and its column count never
> changes. As a side effect, the two `count(*)` calls can still be merged.
> The test `test/optimizer/common_aggregate_positional_consumers.test`
> checks both points. Its `LATERAL` query over `GROUPING SETS ((), ())`
> returns `7 0`, `7 0`, `8 0`, `8 0` (lines 59-73), and its plan has exactly
> one `count_star() EXPORT_STATE` aggregate (lines 30-57).

**A four-way merge, and a real subtlety with `DISTINCT`-derived counts.**
The second test block shows this isn't limited to pairs:

```sql
-- test/sql/optimizer/expression/test_common_aggregate.test:29-32
EXPLAIN SELECT COUNT(*), COUNT(), SUM(i), COUNT(i), SUM(i) / COUNT(i) FROM integers;
----
logical_opt	<!REGEX>:.*AGGREGATE.*count.*count.*
```
Five expressions in the `SELECT` list reference only **two** distinct
aggregates (one merged count, one sum) once `COUNT(*)`/`COUNT()`/`COUNT(i)`
collapse and the final `SUM(i)/COUNT(i)` division just references the same
merged count and sum a second and third time — `COMMON_AGGREGATE` only ever
needs to keep the *aggregate* computations unique; a plain scalar
expression that *reads* an already-computed aggregate's result multiple
times is `COMMON_SUBEXPRESSIONS`'s job (pass #25, a strictly narrower scope
per §7.6.7 of the overview), not this pass's.

The file's final case is worth a brief, honest note rather than a full
re-derivation: a window function computing `MAX(SUM(n)) OVER (PARTITION BY
name)` alongside a plain `GROUP BY`-level `SUM(n)` in the same `SELECT`
list (`test_common_aggregate.test:75-85`) exercises the interaction between
this pass and window-function planning (a `SUM(n)` used both as a genuine
`GROUP BY` aggregate and as the input to a window aggregate computed over
the grouped result) — the file's own comment calls it out as generating "a
CAE inside a CTE," and correctness is checked purely by result value (`a 3
1.0` / `b 3 1.0`) rather than by an `EXPLAIN` regex. This book did not trace
the window-planning machinery that produces that particular intermediate
CTE shape in the depth the rest of this file's claims were verified to —
noted here rather than asserted past what was actually confirmed.

## 7.12.4 Summary: three passes, one shared idiom underneath

Despite operating at three different scopes — one query's separate parts
(`AGGREGATE_REUSE`), one `GROUP BY`'s relationship to the join beneath it
(`PARTIAL_AGGREGATE_PUSHDOWN`), and one `LogicalAggregate`'s own expression
list (`COMMON_AGGREGATE`) — the first two passes turn out to share a single
underlying mechanism: an aggregate function's **exported accumulator
state** (`ExportAggregateFunction::Bind`, gated on
`HasGetStateTypeCallback`/`HasStateCombineCallback`/`HasStateFinalizeCallback`)
and a generic **`combine_aggr`** function that merges two (or, with an
explicit multiplicity argument, N-times-repeated) partial states into one.
That is exactly the plan-time counterpart of Chapter 14 §14.7's Phase-2
partition-wise re-aggregation, which merges thread-local partial hash
tables the identical way at runtime — the same idea, reapplied wherever the
*plan*, not just the *scheduler*, has split one logical aggregation into
pieces that need to be reassembled correctly: across a `SEMI`-join
byproduct, across a join a fact table sits below, or across worker threads.
`COMMON_AGGREGATE`, by contrast, needs none of that machinery — its job is
entirely syntactic, deciding two aggregate expressions are the literal same
computation before either one is ever run, which is also exactly why it can
be the simplest and shortest of the three files.

One lesson applies to all three passes. Moving or merging an aggregate is
only safe if it does not change *which rows* each expression sees, *which
values* count as equal, or *where* each column sits.
`PARTIAL_AGGREGATE_PUSHDOWN` refuses to run a throwing argument on rows an
inner join would have removed. `COMMON_AGGREGATE` treats two aggregates with
different collations as different. And a positional reader above a merged
aggregate gets a projection in between,
so its columns do not move.
