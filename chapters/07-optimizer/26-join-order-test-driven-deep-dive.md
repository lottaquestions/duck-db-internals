# Chapter 7, Part 26 — Join Ordering, Test-Driven: A Tour of `test/optimizer/joins/`

## 7.26.0 What this part is, and what it deliberately isn't

Chapter 8 already did the conceptual work: the join hypergraph (§8.3), the
`JoinRelationSet` interning trie and the user-visible 12-table
`approximate_join_order_threshold` (§8.4), the DPhyp `EmitCsg`/`EnumerateCsgRec`/`EnumerateCmpRec` search itself
(§8.5), the `cost(S ⋈ T) = |S ⋈ T| + cost(S) + cost(T)` cost model and where its
cardinalities actually come from (§8.6), a full worked three-relation trace
(§8.7), the greedy `SolveJoinOrderApproximately` fallback (§8.8), and
build-side/probe-side assignment (§8.9). None of that is repeated here. This
part exists because DuckDB's own test suite devotes an entire directory,
`test/optimizer/joins/` — 39 files at the top level, plus a five-file
`left_join_reordering/` subdirectory and a one-file `issues/` subdirectory,
45 files in total — plus three more files at `test/optimizer/*.test` directly,
to *exercising* that machinery against real, adversarial, and regression-derived
SQL. Chapter 8's Query C is a clean three-relation chain built to teach the
algorithm; this directory is what happens when the algorithm meets TPC-DS-scale
queries, fuzzer output, NaN-valued floats, ASOF joins, LATERAL UNNEST, and eight
years of accumulated GitHub issue numbers. Reading through it in bulk is the
best evidence available that DPhyp's elegance in the textbook case (§8.7) has
to survive a much messier set of real inputs, and it is worth seeing exactly
how it does.

Every file below was read in full — not inferred from its name — because
several of these names turn out to describe something quite different from
what they sound like (`cross_join_and_unnest_dont_work.test` is, somewhat
delightfully, a test that everything *does* work; `test_tpcds_pushdown.test`
turns out to contain no SQL at all). Where a name and a body disagree, this
part says so explicitly, in keeping with the front matter's policy that
source — here, the actual test body — outranks a plausible-sounding guess.

The material is grouped by what each file is actually testing, not
alphabetically:

- §7.26.1 — robustness/liveness at scale (`tpcds_nofail.test` and its siblings)
- §7.26.2 — cardinality-estimation quality (the "denominator" machinery)
- §7.26.3 — cross products: avoided, required, and correctly reconstructed
- §7.26.4 — column bindings surviving reordering
- §7.26.5 — non-inner joins and reorderability constraints
- §7.26.6 — correlated subqueries and delim-joins meeting join order
- §7.26.7 — floating-point equality (a correctness story, not a cardinality one)
- §7.26.8 — ASOF JOIN's asymmetric semantics
- §7.26.9 — build side / probe side, briefly, pointing at Part 15
- §7.26.10 — the one file that's essentially empty
- §7.26.11 — what to carry forward

## 7.26.1 Robustness at scale: `tpcds_nofail.test`

**Category: liveness/robustness, not plan-quality.** The file's own
description states its purpose in one sentence: *"TPCDS queries that force the
Join optimizer to emit 10000+ pairs. Tests the approx algorithm throws no
errors."* This is the single clearest confirmation, from the test suite
itself, of the second escape hatch described in Chapter 8 §8.5 —
`TryEmitPair`'s `if (pairs >= 10000) { ... return false; }` — actually mattering
in practice, not merely existing as defensive code.

`tpcds_nofail.test` runs `CALL dsdgen(sf=0.01)` (a tiny, 1%-scale-factor TPC-DS
data generation) and then executes, as a plain `statement ok` (no result
checked — the point is that it *runs*, not what it returns), a genuinely
enormous query: a CTE (`cs_ui`) joining `catalog_sales` and `catalog_returns`,
feeding into a second CTE (`cross_sales`) that joins **eighteen** relations
(`store_sales`, `store_returns`, `cs_ui`, three separate references to
`date_dim` aliased `d1`/`d2`/`d3`, `store`, `customer`, two references to
`customer_demographics`, `promotion`, two references to
`household_demographics`, two references to `customer_address`, two references
to `income_band`, and `item`), and finally self-joins `cross_sales` against
itself (`cs1`, `cs2`). This is a real TPC-DS query shape (recognizable as a
variant of TPC-DS query 23/related cross-sales analysis queries), and eighteen
relations is comfortably past both the 12-table `approximate_join_order_threshold`
default *and* — because several of those eighteen relations connect to each
other through more than one predicate (the `date_dim` triple-join, the
demographics/address/income-band chains) — almost certainly past the
10,000-candidate-pair cutoff even before the table-count threshold would have
kicked in on its own. The second query in the file, a `CASE`-heavy aggregate
over `store_sales` with five independent scalar subqueries, is a smaller,
different stress shape: five decorrelated subqueries against the same base
table, each contributing its own relation to whatever join tree the optimizer
ultimately builds around the outer `reason` table scan.

Neither query's *plan* is asserted at all — no `EXPLAIN`, no expected row
count. The entire test is a liveness check: does DPhyp's exact search correctly
detect it's overrun its budget, fall back to `SolveJoinOrderApproximately`
(Chapter 8 §8.8) cleanly, and hand back *some* valid, executable plan, rather
than throwing an exception, hanging, or crashing? This is exactly the right
kind of test to have for the two numeric escape hatches in
`plan_enumerator.cpp` — a threshold and a candidate-pair cap are easy to get
right in isolation and easy to get wrong under the combinatorial pressure of a
query nobody hand-wrote for the test suite. `update_nodes_in_full_path.test_slow`
and `updating_the_join_node_hash_map_has_no_errors.test_slow` (§7.26.4) are
close cousins of this same idea, using fuzzer-generated rather than
TPC-DS-derived trees to apply the same kind of pressure from a different angle.

### The greedy fallback in detail

The greedy fallback, `SolveJoinOrderApproximately`, is the code that
`tpcds_nofail.test` exists to stress, so it is worth showing in detail. The
idea: start with every base relation as its own set. At each step, join the
two sets with the lowest cost. Repeat until one set is left. On top of that
simple loop, the code does three things. It first tries the join order the
query already had. It scores each pair of relations once, not once per step.
And it tries both orientations of a pair. The exact search and its
10,000-pair budget (`plan_enumerator.cpp:243`) are as Chapter 8 describes.

**Step 0: keep the incoming order as a candidate.** A short loop at the top
of the function does this:

```cpp
// src/optimizer/join_order/plan_enumerator.cpp:408-420
bool PlanEnumerator::SolveJoinOrderApproximately() {
	// Keep the incoming join order as a candidate for the greedy search.
	for (auto &op : query_graph_manager.GetJoinOperators()) {
		auto &left = op->left_relations.get();
		auto &right = op->right_relations.get();
		if (plans.find(left) == plans.end() || plans.find(right) == plans.end()) {
			continue;
		}
		auto &connections = GetConnections(left, right);
		if (!connections.empty()) {
			EmitPair(left, right, connections);
		}
	}
```

`GetJoinOperators()` (an accessor, `query_graph_manager.cpp:260`) returns
the joins that the query had before reordering. For each one whose two input
sets already have a plan, the loop calls `EmitPair`. `EmitPair` puts the plan
in the DP table (the `plans` map) only if it is cheaper than what is there.
So the plan the user wrote is always one of the options. The greedy search
can still pick something better, but it cannot lose a good written order
just because its first greedy step went another way.

**Each pair is scored once.** A simple loop would call `EmitPair` on every
connected pair in every step. With `r` relations that is about `r²` calls
per step and `r` steps. On one 100-aggregate query (issue #26865) such a
loop took 21.5 s of 23.5 s of planning time. So the code keeps a list of
scored candidates:

```cpp
// src/optimizer/join_order/plan_enumerator.cpp:429-463
	// in every step of the algorithm, we greedily pick the join between the to-be-joined relations that has the
	// smallest cost. Pair costs are deterministic while the query graph is unchanged, so we evaluate each pair once
	// and keep the viable candidates in a list, instead of re-emitting every pair in every step
	struct GreedyCandidate {
		optional_ptr<JoinRelationSet> left;
		optional_ptr<JoinRelationSet> right;
		double cost;
	};
	vector<GreedyCandidate> candidates;
	auto EmitOrientation = [&](JoinRelationSet &left, JoinRelationSet &right) -> optional_ptr<DPJoinNode> {
		auto &connection = GetConnections(left, right);
		if (connection.empty()) {
			return nullptr;
		}
		return EmitPair(left, right, connection);
	};
	auto EmitCandidate = [&](JoinRelationSet &left, JoinRelationSet &right) {
		// non-commutative joins (e.g. LEFT, SEMI, ANTI) are only valid in one orientation
		auto node = EmitOrientation(left, right);
		if (!node) {
			node = EmitOrientation(right, left);
		}
		if (node) {
			candidates.push_back({&left, &right, node->cost});
		}
	};
	auto InitializeCandidates = [&]() {
		candidates.clear();
		for (idx_t i = 0; i < join_relations.size(); i++) {
			for (idx_t j = i + 1; j < join_relations.size(); j++) {
				EmitCandidate(join_relations[i], join_relations[j]);
			}
		}
	};
	InitializeCandidates();
```

After a merge, the loop drops only the candidates that used one of the two
merged sets. Then it scores the new pairs, each made of one remaining set
and the new merged set:

```cpp
// src/optimizer/join_order/plan_enumerator.cpp:525-538
		idx_t kept_candidates = 0;
		for (idx_t candidate_idx = 0; candidate_idx < candidates.size(); candidate_idx++) {
			auto &candidate = candidates[candidate_idx];
			if (candidate.left == best_left || candidate.left == best_right || candidate.right == best_left ||
			    candidate.right == best_right) {
				continue;
			}
			candidates[kept_candidates++] = candidate;
		}
		candidates.resize(kept_candidates);
		for (auto &relation : join_relations) {
			EmitCandidate(relation, new_set);
		}
		join_relations.push_back(new_set);
```

The list is rebuilt from scratch only when `ActivateRequiredCrossProducts()`
adds edges to the graph (`plan_enumerator.cpp:477-481`), because new edges can
change which pairs connect. Ties between equal-cost candidates are broken in
the same order a full re-scan of all pairs would use, so the list gives the
same plans as the simple loop.

**Both orientations are tried.** `EmitCandidate` falls back to
`EmitOrientation(right, left)`, as shown above. A `LEFT`, `SEMI` or `ANTI`
join is valid in one direction only. If a pair were asked only one way round
(say, always with the newly merged set on the right), a valid pair could be
rejected. With no candidate left, the optimizer would give up and keep the
original tree. The test for this is in
`test/optimizer/common_subplan_projection_map.test:85-111`. It sets
`approximate_join_order_threshold=2` to force the greedy path, joins an
`ANTI`-join view to itself through two `LEFT JOIN`s, and checks that the
repeated view is still materialized once as a common subplan
(`__common_subplan.*CTE Scan.*CTE Scan`).

**Faster set checks.** Every `JoinRelationSet` has a bitmask next to its
sorted array of relation indexes:

```cpp
// src/optimizer/join_order/join_relation_set.cpp:12-24
static constexpr idx_t BITMASK_WORD_SIZE = sizeof(idx_t) * 8;

JoinRelationSet::JoinRelationSet(unsafe_unique_array<RelationIndex> relations, idx_t count)
    : relations(std::move(relations)), count(count) {
	if (count != 0) {
		// the relations are sorted, so the last one determines the number of words
		bitmask.resize(this->relations[count - 1].index / BITMASK_WORD_SIZE + 1, 0);
		for (idx_t i = 0; i < count; i++) {
			auto index = this->relations[i].index;
			bitmask[index / BITMASK_WORD_SIZE] |= idx_t(1) << (index % BITMASK_WORD_SIZE);
		}
	}
}
```

`IsSubset` and `Intersects` compare 64-bit words instead of walking the
two sorted arrays. The bitmask is a `vector<idx_t>`, so it grows by one word
per 64 relations. It does not limit the number of relations.

## 7.26.2 Cardinality estimation quality: the denominator machinery

Chapter 8 §8.6 covered *where* the leaf cardinalities feeding
`cost(S ⋈ T) = |S ⋈ T| + cost(S) + cost(T)` come from (table filters, HLL
distinct counts, the `DEFAULT_SELECTIVITY = 0.2` fallback) and worked one
three-relation join by hand. This directory's cardinality-estimation files are
about a narrower, harder question §8.6 didn't have room for: once you have two
or more relations joining on possibly-composite, possibly-transitive,
possibly-mixed equality/inequality predicates, what number do you divide by to
turn "product of the two input cardinalities" into "estimated output
cardinality"? That number is what `src/optimizer/join_order/cardinality_estimator.cpp`
calls, in its own vocabulary, the **denominator** — and getting it right or
wrong is worth up to an order of magnitude in join-order quality, because a
badly-estimated denominator feeds directly into the `EmitPair`/`cost_model.cpp`
comparison that Chapter 8 §8.7 walked through.

### `denominator_cardinality_estimation.test` — the denominator's edge cases, one at a time

This is the richest single file in the directory for cardinality-estimation
internals, and its six blocks each isolate one specific way the denominator
computation can go wrong, verified against real source (`cardinality_estimator.cpp:668-880`):

**Block 1 — disconnected components merged by a cross product.** Four tables
(`dc_a`/1000, `dc_b`/100, `dc_c`/500, `dc_d`/50) joined as two independent
pairs (`dc_a ⋈ dc_b`, `dc_c ⋈ dc_d`) with *no* predicate connecting the two
pairs to each other — the query's `FROM` clause literally juxtaposes them with
a comma. This is a genuinely disconnected join hypergraph (two separate
connected components), so the plan *must* contain an explicit `CROSS_PRODUCT`
node (this is the same "no predicate at all connects two halves of the query"
situation that §7.26.3's `required_cross_product_activation.test` studies from
the opposite angle — that file proves a cross product is *avoided* when
predicates exist to connect the graph; this block proves the denominator
estimate stays correct when one genuinely can't be avoided). The comment notes
the final estimate "must include selectivity from both joined pairs" —
i.e., the two pairs' respective denominators both have to be reflected, not
just one of them, in the combined estimate for the whole four-relation
cross-joined result (`~5,000` rows: `(1000×100/100) × (500×50/50) = 1000 × 500`,
scaled by each pair's own FK/PK-style selectivity).

**Block 2 — an equality class discovered out of arrival order.** Four
same-cardinality tables (`te_a`..`te_d`, 1000 rows each) joined by predicates
including one `IS NOT DISTINCT FROM` mixed in with ordinary `=`, arranged so
that the union-find over equality predicates (Chapter 8 §8.3's "equality
classes" aside — `QueryGraphManager::BuildPredicateModel`) discovers that
`te_c.k`/`te_d.k` and `te_a.k`/`te_b.k` are actually the *same* transitive
equality class only after processing edges in a specific order. The comment:
"All equality edges must receive the final component id" — the bug this
guards is a stale/partial equivalence-class id being left on some edges if the
union-find's path-compression or component-merging happens to visit them
before the merge that unifies two previously-separate classes, which would
make `ClassifyDenominatorEdge`'s `REDUNDANT_TRANSITIVE_EQUALITY` case
(`cardinality_estimator.cpp:682-699`) miss some of the edges that really are
redundant, double-counting their selectivity.

**Block 3 — non-equality predicates get their own domain group.** A three-way
join where one edge is a plain equality (`ne_a.x = ne_b.x`) and the other is a
range predicate (`ne_b.y < ne_c.y`), verified with a `PIECEWISE_MERGE_JOIN`
regex (Chapter 8 §8.10's decision tree — a lone inequality condition routes
there). The comment is explicit about what must *not* happen: a non-equality
predicate "should still add selectivity, but they must not merge into
equality domains" — i.e., `ne_b.y < ne_c.y`'s contribution to the denominator
has to be tracked as its own thing (`DenominatorEdgeKind::NON_EQUALITY_SELECTIVITY`,
`cardinality_estimator.cpp:690-691`), not folded into whatever transitive
equality class `ne_b.y` might otherwise participate in, because an inequality
and an equality on the same column mean fundamentally different things about
how selective the join is.

**Block 4 — a column split across two domain groups at once.** This is the
most subtle block: a star-join shape (`sg_wss` fact table joined separately to
two dimension-like tables `sg_d1`/`sg_d2` on the *same* column `w`, then the
two resulting subqueries joined to each other on an *expression*,
`w1 = w2 - 52`). The comment states the actual historical failure mode
directly: *"Otherwise the expression predicate keeps an infinite denominator,
estimates ~0 rows, and is chosen before the local equality joins."* Read that
literally: if the expression predicate's column (`w`) doesn't get its distinct
count updated in *both* the equality-closure domain group and the
separate expression-predicate domain group, the expression edge's denominator
estimate can come out unbounded (nothing constrains it), which — because
`EmitPair`/`cost_model.cpp` (Chapter 8 §8.6) picks whichever candidate the cost
model rates cheapest — would make DPhyp *prefer* joining on the wildly
underestimated expression predicate first, over the two much more sensible
local equi-joins against the dimension tables. Getting a shared column's
distinct count propagated into every domain group it participates in, not just
whichever one happens to be visited first, is what keeps this from happening.

**Block 5 — expression equality predicates stay graph predicates, not
equality-closure predicates.** A near-identical shape (`ee_a.x + 1 = ee_b.y`,
`ee_b.y = ee_c.z`) makes the companion point: an expression-valued equality
(`x + 1 = y`, not `x = y`) must still participate as a real hyperedge in the
join graph — Chapter 8 §8.3's DPhyp pruning only works because every candidate
pair is checked against real edges — but it must **not** be folded into the
transitive-equality-class union-find the way a plain column-to-column `=`
would be, since `ee_a.x`'s domain and `ee_b.y`'s domain aren't actually the
same set of values (one is shifted by 1). The `EXPLAIN` regex
(`HASH_JOIN.*x \+ 1.*HASH_JOIN`) confirms the expression edge is still planned
as an ordinary hash join, just never merged into anyone's equivalence class.

**Block 6 — a redundant transitive edge must not double-penalize.** A
four-relation snowflake (`tr_customer`/`tr_supplier`/`tr_nation`/`tr_region`,
TPC-H-flavored) where `c_nationkey = s_nationkey`, `s_nationkey = n_nationkey`
form a transitive chain that *implies* `c_nationkey = n_nationkey` even though
no such predicate is written. The comment: without recognizing this implied
edge as redundant, "the estimator can replace a small domain with the full
table cardinality and badly underestimate fanout" — this is precisely what
`unused_edge_tdoms` (`cardinality_estimator.cpp:722,746`) and the final
`auto denom_multiplier = 1.0 + static_cast<double>(state.unused_edge_tdoms.size());`
(`cardinality_estimator.cpp:880`)
exist to prevent: every edge whose equality class was **already** applied
(`EquivalenceGroupApplied`, `cardinality_estimator.cpp:677-680`) is classified
`REDUNDANT_TRANSITIVE_EQUALITY` and specifically *excluded* from
`unused_edge_tdoms`, so it contributes nothing extra to `denom_multiplier`.
This block's expected estimate (`~1,200` rows) is exactly the FK/PK-style
`|customer| × |supplier| × |nation| × |region| / (their respective domains)`,
with no accidental extra halving from a phantom redundant edge.

The file's last two blocks (LEFT-join fanout, and an empty-relation edge case)
are best read alongside §7.26.5's left-join-specific material — they establish
that a `LEFT JOIN`'s denominator computation must never let the output shrink
*below* the preserved side's cardinality even when the matching inner join
would be much larger (`lf_dim`/10 rows LEFT JOIN `lf_fact`/10,000 rows still
correctly estimates and returns `10,000`, not `10`), and that an entirely empty
join (`empty_inner_a ⋈ empty_inner_b`, both zero rows) mustn't produce a
divide-by-zero total-domain denominator that then poisons a `LEFT JOIN`
sitting on top of it.

### `fkpk_composite_key_ce.test` — the *composite*-key version of the same problem

Where the previous file's blocks are mostly about single-column equality
classes, `fkpk_composite_key_ce.test` is specifically about **multi-column**
(composite) foreign-key/primary-key joins, and it does something the book's
front matter explicitly values: its own header comment narrates the exact
before/after of a real fix, in the file itself.

```
# test/optimizer/joins/fkpk_composite_key_ce.test:115-123 (comment, verbatim)
# Before the fix: MarkEdgeEquivalences was called before CreateHyperGraphEdges,
# so left_binding/right_binding were unset and all filters failed the validity
# check. No equivalence index was ever assigned. The transitive edge
# ps_suppkey=s_suppkey then landed in unused_edge_tdoms, doubling the
# denominator and halving the estimate (~300K instead of ~600K).
#
# After the fix: equivalence indices are correctly assigned, the transitive
# edge is recognised as redundant and skipped, and the FK/PK multi-key cap
# keeps the estimate at ~|li| rows (~600K).
```

This is worth pausing on because it names a genuine *ordering* bug in the
optimizer's own construction sequence — `MarkEdgeEquivalences` running before
`CreateHyperGraphEdges` meant every predicate's `left_binding`/`right_binding`
were still unset at the moment equivalence indices needed to be assigned, so
*no* predicate ever got tagged as belonging to an equality class at all, and
the `unused_edge_tdoms`/`denom_multiplier` machinery from
`denominator_cardinality_estimation.test`'s Block 6 (which depends entirely on
correctly recognizing an edge as `REDUNDANT_TRANSITIVE_EQUALITY`) had nothing
to work with — every edge looked "unused," `denom_multiplier` came out as `2`
instead of `1`, and the join's estimated cardinality was silently halved. The
test's TPC-H-Q9-flavored setup (`ps`/`partsupp`, `su`/`supplier`, `li`/`lineitem`,
with a genuine composite primary key on `ps` — `(ps_suppkey, ps_partkey)`, each
pair appearing exactly once by construction) is built specifically so that the
transitive edge `ps_suppkey = s_suppkey` (implied by
`ps.ps_suppkey = li.l_suppkey` and `su.s_suppkey = li.l_suppkey` both being
present) is exactly the kind of edge this bug would have mis-flagged.

The rest of the file works through the **composite-key cap** itself
(`CardinalityEstimator::ApplyJoinPairCap`, `cardinality_estimator.cpp:585-613`)
in isolation from the equivalence-class bug:

- **The cap correctly recognizes a true composite key** (`ck_a`/`ck_b`, both
  keyed on `(x, y)` with `x` and `y` in lockstep, 1000 distinct pairs each):
  the estimate lands at exactly `~1,000` rows, not the naive
  `1000 × 1000 / (domain of x) / (domain of y)` an estimator treating the two
  conditions as *independent* would compute (which would badly underestimate,
  since `x` and `y` are perfectly correlated here, not independent).
- **The cap doesn't override a genuinely tighter estimate.** When one
  same-pair key is already more selective than the FK/PK floor would suggest
  (`ck_sel_a`/`ck_sel_b`, filtered to `x < 100` first), the comment states the
  guard directly: *"the composite cap should not weaken that estimate or
  suppress the second condition."* `ApplyJoinPairCap`'s own guard,
  `if (cap > 0 && first_d < cap && first_d > 0)` (`cardinality_estimator.cpp:606`),
  only *raises* a distinct-count estimate up to the FK/PK floor when it's
  currently *below* that floor — it's a floor, not an override, so a more
  selective, already-correct estimate is left alone.

  The two `ck_sel` `EXPLAIN` checks (`fkpk_composite_key_ce.test:81,88`)
  accept any estimate from 100 to 299 (`~[12][0-9]{2} rows`), not one exact
  number, because sampled distinct counts (`distinct_statistics.cpp`) can
  move the estimate a little. What matters is that the cap does not push the
  estimate up to the FK/PK floor.
- **The cap explicitly refuses to apply to broad fact-to-fact joins.** The
  `ff_a`/`ff_b` block (100,000 rows each, both with only 1,000 distinct
  `(x, y)` pairs — i.e., heavy duplication per key, unlike a real PK) is a
  negative case: the comment explains that "if both inputs have many rows per
  observed single-column domain, the row-count cap would make a broad
  fact-to-fact join look like a key lookup and hide useful local filters" —
  this is exactly what `stats.CanApplyCap(cap)`
  (`cardinality_estimator.cpp:597-599`) exists to check, per its own inline
  comment: *"Avoid applying it to broad fact-to-fact joins where the base row
  count is much larger than any observed single-column join domain; in that
  case row count is a poor proxy for composite NDV."* The expected result here
  is the full `10,000,000`-row cross-multiplication, not a falsely FK/PK-capped
  smaller number.
- **The composite key can appear *indirectly*, via a prior join.** The final
  `ps4`/`su4`/`li4` block chains `ps4.ps_suppkey = su4.s_suppkey` and
  `su4.s_suppkey = li4.l_suppkey` (already establishing `ps4 ⋈ su4` on
  `suppkey`) with `ps4.ps_partkey = li4.l_partkey` — the comment explains that
  because `ps.suppkey = su.suppkey` was already applied earlier in the plan,
  the remaining `li ⋈ ps` join on `partkey` **implies** a second, composite
  `(suppkey, partkey)` equality even though no single predicate states it
  directly, and "without that, the estimate is roughly 4x too high because
  ps4 has four suppliers per part." The expected estimate (`~60,000`, matching
  the actual row count exactly) confirms the cap correctly reconstructs this
  implied composite relationship rather than treating `partkey` as an
  independent, single-column join.

### `better_ce_estimates_for_bad_join_conditions.test`

A short, three-table file (`t1`/`t2`, 2,000 rows each; `t3`, 8,000 rows, with
two foreign keys into `t1`/`t2`) joined on `t3.t1_id_FK = t1.id AND
t3.t2_id_FK = t2.id`, alongside an *inequality* filter (`t1.name_ != t2.name_`)
that connects `t1` and `t2` directly — but only as a `!=` predicate, which
carries almost no selectivity information (nearly every pair of rows satisfies
`!=`, since `name_` only has 700 distinct values across 2,000 rows). The test's
single assertion is a negative `EXPLAIN` regex:
`<!REGEX>:.*NESTED_LOOP_JOIN.*` — i.e., *no* nested-loop join anywhere in the
plan. Read against Chapter 8 §8.10's physical-join decision tree, this is
checking that a poorly-selectivity-estimated (or, per the file's own
description, simply *badly written*) join condition doesn't accidentally push
the optimizer toward routing the `t1`/`t2` connection through a
`NESTED_LOOP_JOIN` (the fallback for a predicate with no usable equality or
enough range conditions) when a much cheaper join order exists that never has
to materialize the `t1 × t2` relationship via `!=` at all — the two genuinely
useful FK/PK equalities against `t3` are enough to connect the whole graph, and
a well-estimated plan should route through those instead, leaving `!=` to be
evaluated as a residual filter rather than as the join edge that decides plan
shape.

### `not_equal_join_cardinality_estimation.test` — how selective is `<>`?

`<>` and `IS DISTINCT FROM` join conditions have their own estimate. If the
estimator treated them like `<` and `>`, it would think such joins return
far fewer rows than they do.

This file is about the same kind of condition as the previous one: a
join on "not equal". Here is the function that turns one join condition into
a factor of the denominator:

```cpp
// src/optimizer/join_order/cardinality_estimator.cpp:445-463
// Apply the denominator multiplier for a given comparison type and effective distinct count.
static double ApplyComparisonRatio(double base_denom, ExpressionType comparison_type, double effective_d) {
	switch (comparison_type) {
	case ExpressionType::COMPARE_EQUAL:
	case ExpressionType::COMPARE_NOT_DISTINCT_FROM:
		return base_denom * effective_d;
	case ExpressionType::COMPARE_NOTEQUAL:
	case ExpressionType::COMPARE_DISTINCT_FROM:
		return effective_d > 1 ? base_denom * (effective_d / (effective_d - 1)) : base_denom;
	case ExpressionType::COMPARE_LESSTHANOREQUALTO:
	case ExpressionType::COMPARE_LESSTHAN:
	case ExpressionType::COMPARE_GREATERTHANOREQUALTO:
	case ExpressionType::COMPARE_GREATERTHAN:
		// Assume this blows up, but use the tdom to bound it a bit.
		return base_denom * pow(effective_d, 2.0 / 3.0);
	default:
		return base_denom;
	}
}
```

`effective_d` is the number of distinct join-key values (the "tdom", or total
domain). Dividing by `tdom^(2/3)`, as the inequality group does, would be
wrong for `<>`. If a key has `d` distinct values, about
`1/d` of all pairs are equal, so `<>` keeps the other `(d - 1)/d`. Dividing by
`d / (d - 1)` gives exactly that.

The test makes it concrete. `t1` and `t2` each have 1,000 rows with 100
distinct values (`range % 100`):

```sql
-- test/optimizer/joins/not_equal_join_cardinality_estimation.test:13-16
query II
EXPLAIN (FORMAT JSON) SELECT * FROM t1 JOIN t2 ON t1.a <> t2.b;
----
logical_opt	<REGEX>:.*"name":\s*"COMPARISON_JOIN".*"Estimated Cardinality":\s*"9[0-9]{5}".*
```

The cross product has 1,000 × 1,000 = 1,000,000 pairs. With the `<>` rule the
estimate is `1,000,000 / (100/99)` = 990,000, a six-digit number starting
with 9, which is what the regex wants. The real count, checked at the end of
the file, is also `990000`. With the inequality rule the estimate would be
`1,000,000 / 100^(2/3)`, about 46,000: more than 20 times too small. The same
check is made for `IS DISTINCT FROM`. A third check confirms that `<` uses
the inequality rule and gets a five-digit estimate. An estimate that is too
small can make the optimizer pick a nested loop join or the wrong build
side. That is the same risk `better_ce_estimates_for_bad_join_conditions.test`
guards against.

### `always_true_scan_filter_estimate.test` — a filter that removes nothing

This file checks a small rule in the leaf estimates. If the column
statistics prove that a scan filter is true for every row, the filter must
not lower the row estimate. The file turns off `statistics_propagation`, so
the filter stays in the scan and only the estimator sees it:

```sql
-- test/optimizer/joins/always_true_scan_filter_estimate.test:17-20
query II
EXPLAIN (FORMAT JSON) SELECT * FROM input WHERE id IS NOT NULL;
----
logical_opt	<REGEX>:.*"name":\s*"SEQ_SCAN".*"Estimated Cardinality":\s*"1000".*
```

`id` has no NULLs, so `id IS NOT NULL` is always true and the estimate stays
at 1,000. The nullable column `nullable_id` still gets the normal guess of
200 (the default selectivity of 0.2). The rule is four lines in the code
that reads table filters for the join-order optimizer:

```cpp
// src/optimizer/relation_statistics/relation_statistics_scan.cpp:145-154
			if (column_statistics) {
				auto &filter = ExpressionFilter::GetExpressionFilter(entry.Filter(), "ExtractGetStats");
				if (filter.column_indexes.size() <= 1 &&
				    filter.CheckStatistics(context, *column_statistics) == FilterPropagateResult::FILTER_ALWAYS_TRUE) {
					continue;
				}
				cardinality_after_filters =
				    MinValue(cardinality_after_filters,
				             InspectTableFilter(base_table_cardinality, entry.Filter(), *column_statistics));
			}
```

### `left_join_reordering/left_join_cardinality_estimation.test`

This file (grouped here rather than in §7.26.5 because its content is
overwhelmingly about *cardinality*, not reordering *legality*) directly
exercises the same LEFT-join floor `denominator_cardinality_estimation.test`'s
last block touched, but with a specific historical wrong formula named in its
own comments:

```
-- With the old product formula, D = D_id * D_x = 10K * 5K = 50M >> |medium|=10K, so
-- left_join_extra would be ~5000 to compensate — astronomically large intermediate values.
-- With min(D, |RHS|), denom = min(10K, 10K) = 10K, estimate = |large| * |medium| / 10K = |large|.
```

The old formula multiplied the distinct-count estimates of *every* condition
in a multi-condition `LEFT JOIN`'s `ON` clause together to get a combined
denominator (`D = D_id × D_x`), which for two independent-looking columns can
wildly overshoot the RHS table's actual row count (`50M` vs. `10K` rows) — and
because a `LEFT JOIN`'s estimator needs to add back a compensating
"`left_join_extra`" term precisely when the computed denominator undershoots
the true output floor (this is the concrete implementation detail behind
Chapter 8 §8.6's `GetLeftJoinInputCost` correction — that description covered
*why* LEFT joins need special cost treatment; this is what the actual number
being computed looks like when it goes wrong), an inflated denominator forces
an equally inflated compensating term. The fix — capping the combined
denominator at `min(D, |RHS|)` rather than letting the raw product run away —
keeps two independent equality conditions on the same RHS from ever producing
an estimate that implies more distinct key combinations than the RHS actually
has rows. The file's remaining block (an inequality condition, `l_x < m_x`,
mixed with an equality condition in the same `ON` clause) is a narrower
regression check that this mixed-predicate-class case doesn't crash or
regress below the `|LHS|` floor either.

### `left_join_reordering/left_join_composite_domain.test` — composite keys under a LEFT JOIN

The FK/PK composite-key cap from `fkpk_composite_key_ce.test` also applies
to `LEFT JOIN`s.

A *composite key* is a key made of more than one column, such as
`(a, b)`. In this file, `composite_left` has 2,048 rows and
`composite_right` has 1,024 rows. In both tables the first column has 256
distinct values and the second has 4. Neither column is unique on its own,
but the pair `(x, y)` is unique in `composite_right`. So
`composite_left LEFT JOIN composite_right ON a = x AND b = y` returns exactly
2,048 rows: each left row matches at most one right row.

```sql
-- test/optimizer/joins/left_join_reordering/left_join_composite_domain.test:24-28
query II
EXPLAIN SELECT * FROM composite_left
LEFT JOIN composite_right ON a = x AND b = y;
----
physical_plan	<REGEX>:.*HASH_JOIN[^~]*~2.048 rows.*
```

To get this right, the estimator must see that `a = x` and `b = y` together
form one key. A step, `QueryGraphManager::BuildLeftEqualityPairs`
(`query_graph_manager.cpp:108`), collects the `=` and
`IS NOT DISTINCT FROM` conditions of each `LEFT` join into groups. It also
reads equalities hidden in the residual `ON` expression. The LEFT-join
denominator code then applies the same cap as the inner-join case:

```cpp
// src/optimizer/join_order/cardinality_estimator.cpp:489-499
	auto &inner_numerator = set_manager.Union(*left.numerator_relations, *right.numerator_relations);
	auto inner_denom = CalculateInnerJoinDenom(GetEffectiveDenom(left.denom) * GetEffectiveDenom(right.denom), filter);
	auto composite_pair = predicate.GetCompositeJoinPair();
	if (composite_pair) {
		auto cap = GetJoinPairCap(*composite_pair);
		CompositeJoinPairStats stats;
		stats.RegisterDistinctCount(filter.GetDistinctCount());
		if (stats.CanApplyCap(cap) && filter.GetDistinctCount() > 0 && cap > filter.GetDistinctCount()) {
			inner_denom = inner_denom / filter.GetDistinctCount() * cap;
		}
	}
```

The rest of the file checks the edges of this rule, one block each:

- writing the condition the other way round (`x = a`) or using
  `IS NOT DISTINCT FROM` gives the same estimate;
- a projection or an alias between the scan and the join keeps the column
  statistics;
- reusing one column twice (`a = x AND a = y`, or `a = x AND b = x`) is *not*
  a composite key, so the estimate stays higher (`~3,000` to `~9,999` rows);
- an inequality, or an expression with no direct column statistics, is not
  part of the key;
- broad many-to-many joins are still excluded, as in `fkpk_composite_key_ce.test`;
- equalities that are chained or written on one side of the `ON` clause still
  count as one key part;
- NULLs and rows with no match still keep every left row.

### `get_cardinality_from_limit.test_slow`

**Category: cardinality estimation from a non-obvious source.** A single
`EXPLAIN` against `logical_opt` output (`pragma explain_output='optimized_only'`)
confirms a very specific claim: joining a 40,000-row table against
`(SELECT * FROM t_right LIMIT 10000)` (where `t_right` itself has ten million
rows) produces a plan whose `SEQ_SCAN...LIMIT...SEQ_SCAN` shape reflects the
`LIMIT`'s row cap, not the underlying table's true size — i.e., the relation
manager (Chapter 8 §8.3) building the leaf cardinality for this side of the
join reads the `LogicalLimit`'s constant `10000` directly as a hard cardinality
ceiling, rather than falling back to `t_right`'s ten-million-row base
statistic (which would badly mis-cost any join built on top of it). This is a
small, specific instance of a broader theme this directory keeps returning to:
cardinality estimation quality depends on noticing *every* place a plan shape
already tells you something concrete about row counts (a `LIMIT`, a table
filter, an HLL sketch), not just falling back to raw table statistics by
default.

### `parquet_minmax_cardinality_estimation.test` — zonemap-informed estimates

**Category: storage-layer statistics feeding join-order cardinality.** This
file is the clearest bridge in the whole directory between Chapter 8's join
ordering and Chapter 17's storage-engine/zonemap material — it's really a
zonemap-selectivity test wearing a join-order-directory location because its
end consumer is exactly the leaf-cardinality machinery
(`RelationStatisticsHelper::ExtractGetStats`, Chapter 8 §8.6) that feeds
DPhyp. Every block writes a small Parquet file and checks an `EXPLAIN` row
estimate:

- **A single-value equality filter** (`i = 42` against 10,000 sequential
  integers) estimates `~1 row` — Parquet row-group min/max metadata alone
  (no HLL, no separate stats sidecar) is enough to know a range filter against
  a column with a wide, gap-free domain matches almost nothing.
- **The same logic for `TIMESTAMP_MS`** — confirming the min/max-based
  estimate isn't an integer-only special case; it works for a temporal type's
  physical representation too.
- **A join on a *sparse* key** (`k = range × 1000`, only 10 distinct values
  spread across a domain of 10,000) estimates `~10 rows` for the join — the
  min/max range alone (`0` to `9000`) can't distinguish "10 sparse values" from
  "10,000 dense values" on its own; this block is really checking that the
  estimator uses the *file's actual row count* as the distinct-count proxy
  when there's no better signal, rather than assuming a dense domain from the
  min/max span (which would drastically overestimate the join's fan-out if it
  assumed all 9,001 possible integer values were present).
- **Four separate integer-overflow guard blocks** (`BIGINT`, `UBIGINT`,
  `HUGEINT`, `UHUGEINT`, each with one row pinned to the type's minimum value
  and one to its maximum, mixed into 100 otherwise-sequential rows) all assert
  the *same* `~20 rows` estimate for `i = 42`. This is a narrow but real
  correctness point: computing a selectivity fraction from a min/max range
  (`(42 - min) / (max - min)`-style reasoning, or an equivalent bucketed
  estimate) has to avoid a signed/unsigned overflow or precision loss when
  `max - min` for a `HUGEINT` or `UHUGEINT` column is itself a number that
  doesn't fit cleanly back into the arithmetic used to compute the estimate.
  Four parallel blocks, one per at-risk type, is the test author's way of
  making sure the fix (or the original bug) wasn't type-specific.

### `relation_statistics_bindings.test`

**Category: cardinality/statistics bookkeeping correctness across non-trivial
plan shapes.** Three scenarios, each checking that a relation's *statistics*
stay correctly lined up with its *output column bindings* even after the plan
shape changes those bindings' positions:

1. An aggregate (`GROUP BY k`, producing both a group-key binding and a
   separate aggregate-result binding namespace) wrapped in a `MATERIALIZED`
   CTE and then re-projected in a different column order — the comment flags
   the exact risk: "Reordering the CTE projection must not make their shared
   column indexes collide."
2. The same layout, but with `NOT MATERIALIZED` (forcing CTE inlining — a
   Chapter 7 `CTE_INLINING` pass concern, §7.4 pass #2/#12) plus a filter —
   checking the binding layout survives inlining, not just materialization.
3. A `PIVOT` (an "unsupported output-changing operator," per the comment) used
   as one side of a join: the comment states the actual invariant being
   protected — *"Its child can still be optimized, but the parent join must
   not receive guessed output-column statistics"* — i.e., when the relation
   manager (Chapter 8 §8.3) hits an operator whose output shape it genuinely
   can't reason about statistically, it must degrade gracefully (treat it as
   an opaque boundary with honest, non-fabricated statistics) rather than
   silently attaching plausible-looking but wrong numbers to columns a `PIVOT`
   invented.

### `estimated_cardinalities_are_in_logical_plan.test` — the architectural claim

**Category: architecture, directly tied to Chapter 5.** This file's whole
point is narrower and more structural than the others in this section: a
three-relation chain join (`t1 ⋈ t2 ⋈ t3`, deliberately sized so `t2`/`t3` each
have their own distinct row-count profile) is explained with
`PRAGMA explain_output=OPTIMIZED_ONLY`, and the single assertion is a regex
against the **`logical_opt`** output (not `physical_plan`) confirming that a
`COMPARISON_JOIN` node's rendered `EXPLAIN` output includes an estimated row
count (`~[0-9]+`) *at the logical-plan stage*, immediately after optimization,
before physical planning has even begun. This is the exact confirmation
Chapter 5's `estimated_cardinality`/`has_estimated_cardinality` fields on
`LogicalOperator` are for: the join-order optimizer doesn't merely *use*
cardinality estimates transiently inside its own DP table (Chapter 8 §8.6/8.7)
and then discard them — it writes the winning estimate back onto the
reconstructed `LogicalComparisonJoin` node itself, where it survives as
genuine logical-plan state, visible in `EXPLAIN`, and available to every later
pass and to the physical plan generator (this is precisely the
`op.children[0]->has_estimated_cardinality` check Chapter 8's own tree
inherits and that §7.26.8 below shows a physical-planning decision — ASOF
join's loop-vs-operator choice — actually reading at plan-generation time).
`join_reorder_optimizer.test` (next section) and `any_join_condition_pruning.test`
both live at this same `test/optimizer/*.test` root level rather than inside
`joins/` — a filing detail worth noting only because it means all three of
this part's root-level files are testing something slightly more structural/
architectural than a typical `joins/`-directory file, which tends to be about
one specific join-order or cardinality *behavior*.

## 7.26.3 Cross products: avoided, required, and correctly reconstructed

Chapter 8 §8.3–§8.5 established that DuckDB's hypergraph construction means a
candidate join pair is *never even proposed* to the DP table unless a real
edge connects the two relation-sets — cross products aren't something DPhyp
weighs and rejects as too expensive; they're structurally absent from the
search unless nothing else can connect the graph. This section's files probe
the boundary of that guarantee from several angles: when is a cross product
truly unavoidable, does the optimizer correctly recognize that boundary, and
does reconstructing the final plan tree around a cross product stay correct
when the cross product spans more than the simple two-relation case Chapter 8
worked through.

### `required_cross_product_activation.test`

**Category: cross-product necessity, both directions.** This file states its
own thesis in its description: *"Activate explicit cross products only when
they are needed to connect the predicate graph."* Two fact tables (`f1`, `f2`,
10,000 rows each) and two trivial one-row dimension tables (`d1`, `d2`) are
joined with a *syntactic* cross product in the `FROM` list (`FROM d1, d2, f1,
f2`) but a fully-connecting `WHERE` clause (`f1.dk = d1.k AND f2.dk = d2.k AND
f1.id = f2.id`). Run across both the exact search and the greedy fallback
(`foreach threshold 0 30` — recall Chapter 8 §8.4's `approximate_join_order_threshold`
default of 12; `threshold=0` forces every query in this file through
`SolveJoinOrderApproximately` regardless of table count, `threshold=30` lets
the exact search run since there are only four relations), the resulting plan
must contain **no** `CROSS_PRODUCT` node at all — the syntactic comma-joins in
the `FROM` clause are just SQL surface syntax; the join hypergraph the
predicates actually describe is fully connected, and DPhyp (or its greedy
fallback) must recognize that and build ordinary joins throughout. The file
then flips the scenario: `SELECT count(*) FROM d1, d2` with *no* connecting
predicate at all *must* produce an explicit `CROSS_PRODUCT` — there is no other
legal way to compute this query. Finally, with
`SET debug_force_no_cross_product = true` (Chapter 8 §8.6's cost-model excerpt
already showed this exact setting being read in `SolveJoinOrder`), the
fully-connected four-relation query still succeeds (no cross product was ever
needed), but the genuinely-disconnected `d1, d2` query now fails outright with
`Invalid Input Error: Query requires a cross-product, but 'force_no_cross_product'` —
proving the debug flag is a hard refusal, not merely a cost penalty nudging
the search away from cross products when an alternative exists.

### `no_cross_product_reordering.test`

**Category: a specific, real correctness boundary — not merely "avoid
reordering a cross product that's already there."** The file's own
description is precise about what it actually guards: *"cannot create cross
product between LHS and RHS of semi/anti joins."* The query is
`select * from t1, t2 where a in (select * from t3)` — an `IN`-subquery over
`t3` (100 rows) correlated against `t1` (1,000 rows), combined with a plain
cross-join-syntax `t1, t2` (`t2` has exactly 1 row). The correct plan, per the
file's own annotated (and then explicitly marked *incorrect*) diagram, joins
`t1` against the `SEMI`-joined subquery result first (`HASH_JOIN...SEMI`), and
only *then* cross-joins in `t2` — the asserted regex is literally
`CROSS_PRODUCT.*HASH_JOIN.*SEMI`, i.e., the cross product happens **above**
the semi join, not below or beside it. The file's ASCII diagram of the
*wrong* plan shows exactly the mistake being guarded against: a
`CROSS_PRODUCT` sitting directly *underneath* the `SEMI` join's own LHS,
splitting `t1` and `t2` in a way that puts a relation the semi join's
predicate needs (`t1`, via `a`) on one side of a cross product boundary that
the semi join's own reconstruction then has to reach across — the point being
that a semi/anti join's LHS and RHS are not freely interchangeable with
whatever cross-product boundary the optimizer might otherwise be tempted to
introduce for cost reasons, because a `SEMI`/`ANTI` join, unlike an ordinary
inner join, has a specific, asymmetric relationship between its two sides
(only the LHS survives into the output; the RHS is purely a filter), and that
asymmetry has to be preserved through whatever the join-order search decides
to bushy-ify around it.

### `single_sided_predicate_reconstruction.test`

**Category: correctness of plan *reconstruction*, specifically at cross
products spanning more than two relations.** The file's description states
the guard directly: *"Join-order reconstruction must not fail when a
cross-product spans more than two relations."* Every block here has the same
shape: four (or five) relations, where one join predicate (`c1.x > 0`, or
`c2.x > 3`, or `c2.x >= 4`) is **single-sided** — it mentions only one
relation (`c1`/`c2`), never the relation it's syntactically an `INNER JOIN...
ON` clause of (`d1`/`d2`). A predicate like this carries zero information
about how `c1` and `d1` relate to each other; it's really just a filter on
`c1` that happens to be spelled inside a join's `ON` clause. Once the
optimizer recognizes this (Chapter 8's hypergraph never gets an edge for a
predicate that doesn't reference both sides of the join it sits on), the
"real" join graph reduces to `{a1, b1}` cross `{c1, d1}` — an unfiltered
cross product between two, not one, relation-sets, each itself containing more
than one base relation. `JoinRelationSet`s spanning multiple relations,
Chapter 8 §8.4's actual, sorted-array-backed representation (the sorted
array also carries a bitmask for fast subset checks; see §7.26.1),
are exactly what has to be correctly reconstructed back into
`LogicalCrossProduct`/`LogicalComparisonJoin` nodes here — the risk this test
guards against is a reconstruction bug specific to a cross product whose
*endpoints* are themselves multi-relation sets rather than single leaves,
since that's a structurally different (and more complex) case than a cross
product between two lone base tables. Three variants — symmetric
cardinalities, asymmetric cardinalities (explicitly noted as freeing the cost
model to "pick a different bushy plan"), and a fifth relation widening the
connected component further — stress this reconstruction path from several
different resulting tree shapes, each checked against an exact expected
`count(*)`.

### `no_duplicate_elimination_join.test`

**Category: binding correctness at composite/multi-relation hyperedges** — a
close cousin of the previous file, but built specifically around the
multi-relation hyperedge case Chapter 8 §8.3 flagged as the honest limit of
its own claim (edges whose endpoints are `JoinRelationSet`s of size greater
than one, arising from expression-valued predicates like
`test1.range + test4.range = test2.range`, which forces `{test1, test4}` to be
treated as one joint endpoint before it can connect to `test2`). Five tables
(`test0`..`test4`) are connected by predicates whose right-hand-side ASCII
comments in the test file itself annotate the intended hyperedges directly:

```sql
-- test/optimizer/joins/no_duplicate_elimination_join.test:22-25
WHERE test1.range + test4.range = test2.range  -- Edge: (test1,test4) ⋈ test2
  AND test1.range + test4.range = test3.range  -- Edge: (test1,test4) ⋈ test3
  AND test1.range = test4.range                -- Edge: test1 ⋈ test4
  AND test1.range = test0.range;               -- Edge: test1 ⋈ test0
```

The file's own description is the useful part: *"In the join order optimizer
queries need to have the correct bindings"* — despite the filename's
suggestion of duplicate elimination, this isn't about DELIM-joins at all
(contrast with §7.26.6); it's a plain, non-correlated five-way join whose real
content is exercising `RelationManager`'s bookkeeping (Chapter 8 §8.3) when a
composite two-relation set (`{test1, test4}`) has to be treated as a single
hyperedge endpoint for two *separate* downstream edges (to `test2` and to
`test3`) at once, plus a second query variant that adds a *third* such
composite edge (`(test1,test3) ⋈ test0`) layering yet another multi-relation
endpoint onto an already-multi-relation-endpoint graph. `pragma
debug_force_no_cross_product=true` at the top confirms the whole point: this
five-relation graph must be fully connectable without any cross product at
all, purely through composite hyperedges — if the relation manager's
bindings for a composite set get scrambled anywhere along the way, the
optimizer would either produce wrong results silently or fail to find a
connected plan and incorrectly fall back to (or error out over) a cross
product it shouldn't need.

### `join_reorder_optimizer.test` (root-level `test/optimizer/`)

**Category: DPhyp liveness on a connected hypergraph, plus reordering-adjacent
correctness in an unrelated feature.** The file's own description: *"Make
sure we can emit a vaild [sic] join order by DPhyp if hypergraph is
connected."* Four tables (`t1`..`t4`, each derived from the last with a few
extra duplicate rows inserted) are joined three different ways under
`debug_force_no_cross_product=true`: a three-edge open chain
(`t1.c1=t2.c1 AND t2.c2=t3.c2 AND t3.c3=t4.c3`), the same chain closed into a
cycle by adding `t4.c4=t1.c4`, and a variant where the last predicate is
replaced by a multi-column arithmetic expression
(`t1.c1 + t2.c2 + t3.c3 = 3 * t4.c4`) — the third case forcing all four
relations onto one side of a single hyperedge, similar in spirit to
`no_duplicate_elimination_join.test`'s composite endpoints, but arriving there
from a genuinely different predicate shape (a single four-relation expression
rather than several separate two/three-relation edges). All three are only
`statement ok EXPLAIN` — liveness, not plan-shape assertions. The file's final
two blocks are a bit of an outlier for a join-ordering test file: a
`generate_subscripts`/`UNNEST`-driven word-search-puzzle CTE chain with a
self-referencing `cross join` between a `grid` CTE and a `search` CTE, run
once without and once with an explicit `row_id` column — these don't obviously
connect to join *ordering* at all; they read as a fuzzer- or user-report-
derived regression appended to this file because it happened to also touch
DPhyp's handling of a `CROSS JOIN` inside a correlated `EXISTS` subquery,
rather than because the file's original author designed them to test the same
thing as the four `t1`..`t4` blocks above them. This is a good, honest example
of how a `.test` file's contents can accrete unevenly over a project's history
— not every block in a file necessarily shares the same design intent as the
blocks around it.

### `any_join_condition_pruning.test` (root-level `test/optimizer/`)

**Category: join-condition folding meeting cross-product/empty-result
recognition — directly adjacent to `EMPTY_RESULT_PULLUP` and the `ANY_JOIN`
mechanism.** `ANY_JOIN` is DuckDB's representation for a join whose condition
is an arbitrary boolean-valued expression that the optimizer can't turn into a
`JoinCondition` comparison (here, `array_cosine_similarity(t1.a, t2.a) IS NOT
NULL` — a function call, not a column comparison). This file checks what
happens when such a condition **folds to a compile-time constant** via the
`t` table's fixed values: an `ANY_JOIN` whose condition is always true
degenerates into a plain `CROSS_PRODUCT` (confirmed by both a positive
`CROSS_PRODUCT` regex and a negative `<!REGEX>:.*ANY_JOIN.*` check — the
`ANY_JOIN` node genuinely disappears from the plan, it doesn't merely get
folded away in spirit), while a condition that's always `NULL`/false is
pruned all the way to an `EMPTY_RESULT` node — the join subtree is proven, at
optimization time, to be incapable of producing any row, and collapses
entirely (this is the same `EMPTY_RESULT_PULLUP` family of reasoning Chapter
7's pipeline listing covers as pass #13, here triggered from a join-condition
angle rather than a `WHERE`-filter angle). The file's last block is the
important boundary case: this always-true-condition-to-cross-product
degeneration is explicitly **not** applied to join types that "also emit
non-matching rows" — an `ANTI` or `LEFT` join whose condition is a constant
`IS NOT NULL` still has to actually emit rows for non-matches (an `ANTI` join
keeps every LHS row that *doesn't* match; a `LEFT` join keeps every LHS row
regardless), which a bare `CROSS_PRODUCT` rewrite would get wrong — so the
degeneration is correctly scoped to join types where "always matches" and "is
a cross product" really are equivalent (inner-style joins), and left alone
everywhere else.

### `test_issue_5265.test_slow` — one cross product, not two

**Category: a real regression case study.** This file's setup is a full
TPC-H `dbgen(sf=0.1)` six-relation revenue-by-nation query
(`region ⋈ nation ⋈ supplier ⋈ lineitem ⋈ orders ⋈ customer`, with one
composite condition — `c.c_custkey = o.o_custkey AND c.c_nationkey =
s.s_nationkey` — linking `customer` back to *both* `orders` and `supplier` at
once) plus one extra, deliberately trivial join:
`JOIN (SELECT 1 AS dummy) single_row ON 1 = 1`. That final join's condition
references no column at all — it's a comparison of two constants — so it
contributes no edge to the join hypergraph whatsoever (Chapter 8 §8.3's
hyperedges are built strictly from predicates connecting two relation-sets;
a constant-only comparison connects nothing). `single_row` is therefore a
disconnected, one-row relation that has no choice but to be attached via an
explicit cross product somewhere in the final plan — but the file's own
comment (*"this should run quickly"*) plus its `EXPLAIN` assertion,
`<!REGEX>:.*CROSS_PRODUCT.*CROSS_PRODUCT.*` (i.e., **at most one**
`CROSS_PRODUCT` anywhere in the plan, not two), together read as a regression
guard against a historical reconstruction bug where a disconnected,
single-relation component like `single_row` could end up attached to the main
join tree via more than one redundant cross-product node instead of exactly
one — needlessly duplicating work (and, per the "should run quickly" comment,
apparently making the query noticeably slower) for a join that contributes
nothing but a single extra column nobody even selects.

## 7.26.4 Column bindings must survive reordering

DPhyp doesn't just decide *which* relations join in *which* order — once it's
decided, `JoinOrderOperator`/`QueryGraphManager::Reconstruct` (Chapter 8 §8.7's
closing line) has to rebuild an actual operator tree and rewrite every column
reference in the rest of the plan to point at wherever that column ended up
living in the new tree. Get this rewrite wrong, and a query doesn't crash — it
silently returns the *wrong column's values* under a correct-looking column
name. This section's files are specifically about that hazard.

**`order_optimizer_bindings.test`** joins a table function's output against
itself: `summary((select 5)) tbl1(i) JOIN summary((select 5)) tbl2(i) ON
tbl1.i=tbl2.i`, run across both the exact and greedy thresholds
(`foreach threshold 0 30`). `summary(...)` is a table function whose output
schema isn't a plain base-table column list, and the `tbl1(i)`/`tbl2(i)`
syntax renames only its *first* output column, leaving the rest to their
default names — exactly the kind of relation whose column bindings are most
at risk of getting scrambled if the relation manager (Chapter 8 §8.3) doesn't
track a table function's full output schema correctly through join
reconstruction. The description states the point plainly: *"queries need to
have the correct bindings."*

**`test_logical_get_with_unnest.test`** makes the same point about a
different relation shape: its description is *"Test unnest as logical Get.
Make sure relation manager has all column bindings."* The query cross-joins a
table against `UNNEST(cc.number)` and, in the same `FROM` clause, a
`CROSS JOIN LATERAL` subquery reading `json_each(cc.my_json)`. Both `UNNEST`
and `json_each` are represented as `LogicalGet` nodes that, unlike a plain
base-table scan, have their *own* child expressions/operators feeding them —
the risk this guards is `RelationManager::AddRelation` (Chapter 8 §8.3) only
capturing a subset of such a `LogicalGet`'s actual output bindings (missing
whichever bindings come from its non-trivial child), which would leave the
join-order optimizer's later column-rewrite with a hole to fall into.

**`issues/internal_6248.test`**, filed under the directory's own `issues/`
subfolder, is this exact same failure mode traced to a concrete internal
issue number. Its description: *"Extract all child bindings of logical gets
if they have children. Usually the case with table functions."* The
reproduction — a `json_each`/`json_group_array`-based correlated aggregate
joined against a second table on a hyphenated, quoted column name
(`"company-id"`) — is deliberately awkward on two independent axes at once
(a table-function `LogicalGet` with children, *and* an identifier that needs
quoting), which is a realistic way for a customer-reported bug to surface a
binding gap that a cleaner, hand-written regression test might not have
stumbled onto.

**`update_nodes_in_full_path.test_slow`** and
**`updating_the_join_node_hash_map_has_no_errors.test_slow`** are this
section's stress-test analogues to §7.26.1's `tpcds_nofail.test` — both are
fuzzer-derived, deeply nested join trees over `dbgen`-generated TPC-H tables,
mixing `INNER`/`LEFT` joins with correlated scalar subqueries sitting directly
inside `ON` clauses (`ON ((SELECT l_linestatus FROM main.lineitem LIMIT 1
OFFSET 2) IS NULL)`). Both check that the plan-enumerator's internal
per-relation-set bookkeeping — the `JoinRelationSet` interning trie and the DP
table keyed on it (Chapter 8 §8.4) — doesn't corrupt or error out under a
tree shape no hand-written test would ever produce. The second file is
particularly telling about *why* fuzzer output like this matters: its first
query is a `statement error` (the fuzzer-generated `ON ((SELECT NULL))`
condition legitimately returns more than one row against a multi-row table,
correctly triggering `More than one row returned by a subquery used as an
expression`), and the *second* query is the identical tree re-run after
`SET scalar_subquery_error_on_multiple_rows=false` — proving the underlying
join-order machinery handles the same adversarial tree shape cleanly whether
or not that particular business-logic safety check is engaged, i.e., that the
safety check and the plan-enumeration robustness are genuinely independent
concerns.

Three more files in this directory guard the same "rebuild the tree
correctly" job: `filter_projection_map_reconstruction.test`,
`zero_column_filter_reconstruction.test` and
`join_elimination_condition_reference.test`. Each one guards against a real
bug.

**`filter_projection_map_reconstruction.test`** (issue #28157) is about the operator *above* the reordered joins.
Join ordering often starts below a `LogicalFilter`. That filter stays in
place, and the new join tree is put under it. A filter can carry a
*projection map*: a list of positions that says which of its child's columns
it passes on. The new join tree may output its columns in a different order.
If the map were not updated, the filter would pick the wrong columns by
position. The test's first query would then fail with
`INTERNAL Error: Failed to bind column reference "i"`:

```sql
-- test/optimizer/joins/filter_projection_map_reconstruction.test:17-25
query I
SELECT count(*) FROM (
	SELECT b.i <> ANY (SELECT i FROM s)
	FROM t AS a
	LEFT JOIN t AS b ON a.b = b.b AND b.i IN (SELECT i FROM s)
	WHERE (SELECT 3) = 3 OR a.i IS NULL
);
----
3
```

`QueryGraphManager::Reconstruct` remembers the old child's column
bindings before the rebuild. After it puts the new tree in place, it remaps
the parent by binding, not by position:

```cpp
// src/optimizer/join_order/query_graph_manager.cpp:566-567
	// the new join tree can emit its columns in a different order: preserve the parent's output layout
	ColumnBindingRewrite::ApplyToChild(*parent, 0, std::move(old_child_bindings), BindingReplacementGraph());
```

**`zero_column_filter_reconstruction.test`** (issue #27451) is about filters that use no column at all,
such as `WHERE NULL` or `WHERE random() > 2`. Join ordering takes filters off
the operators it replaces and puts them back on the new tree. If it skipped
any filter with no relation bindings, such a filter would stay on an
operator that is then thrown away, and the filter would be lost. The test turns
off `filter_pushdown` so that such a filter stays between the cross products
of `SELECT (SELECT 1) AS a0, (SELECT 2) AS a1 FROM t0 WHERE NULL`. That query
must return no rows. A helper says whether a filter may move:

```cpp
// src/optimizer/join_order/relation_manager.cpp:901-908
bool RelationManager::ExtractFilterBindings(const Expression &expression, unordered_set<RelationIndex> &bindings) {
	const auto can_reorder = ExtractBindings(expression, bindings);
	if (!bindings.empty()) {
		return true;
	}
	// without bindings the filter can only move if it references no columns at all
	return can_reorder && !ExpressionContainsColumnRef(expression);
}
```

A filter with no column references gets an empty relation set, and the
rebuild puts it back on top. A filter that reads an internal column the
optimizer does not track (the code comment's example is `limit_rownum`) still
stays where it is.

The same file covers *volatile* filters. A volatile expression can give
a different answer each time it runs, like `random()` or `nextval()`. Moving
it to a place where it sees more rows, or different rows, changes the result.
So a `LogicalFilter` with a volatile expression is a wall that join
ordering does not cross. The optimizer still reorders the joins below it:

```cpp
// src/optimizer/join_order/relation_manager.cpp:172-181
static bool OperatorIsNonReorderable(const LogicalOperator &op) {
	switch (op.type) {
	case LogicalOperatorType::LOGICAL_FILTER:
		// Preserve the input rows on which volatile predicates are evaluated.
		for (auto &expression : op.expressions) {
			if (expression->IsVolatile()) {
				return true;
			}
		}
		return false;
```

The test uses a sequence to count how often a filter runs. Here the filter
must run once, on the single row of `range(1)`, before the join doubles that
row:

```sql
-- test/optimizer/joins/zero_column_filter_reconstruction.test:84-98
# a volatile filter on a leaf must run before the join duplicates its input
statement ok
CREATE SEQUENCE leaf_filter_calls;

query I
SELECT count(*)
FROM (SELECT * FROM range(1) t(i) WHERE nextval('leaf_filter_calls') = 1) a
JOIN (VALUES (0), (0)) b(i) USING (i);
----
2

query I
SELECT currval('leaf_filter_calls');
----
1
```

If the filter moved above the join, it would run twice, and the second call
would return 2, so `= 1` would drop a row. Other blocks check that two
identical volatile filters are both kept (`currval` is 2), and that a
volatile filter between two joins stays between them.

**`join_elimination_condition_reference.test`** is not about join *ordering*. It guards the `JOIN_ELIMINATION`
pass, which removes a join whose output nobody reads. In this TPC-H-shaped
query, a `SINGLE` join from a `LIMIT 1` scalar subquery feeds the `ON`
condition of a `LEFT JOIN` above it. Elimination must see that the condition
reads that output and keep the join. The expected result has three rows.

Two related rules are tested outside this directory. Join ordering keeps
the order of stacked filters when it rebuilds them; Part 22 §6.1 walks
through it, since it matters for filters that can throw. Join ordering also
handles a `SECURE VIEW`: the view
becomes one relation, its inside is optimized on its own, and only its own
estimate (plus column domains, if the view may share statistics) crosses the
view boundary (`relation_manager.cpp:520-522`). Its tests are
`test/sql/optimizer/secure_view_join_order.test` and
`test/sql/optimizer/secure_view_build_side.test_slow`.

## 7.26.5 Non-inner joins and reorderability constraints

Chapter 8 mentioned, but didn't dwell on, the fact that not every join
operator is freely reorderable inside DPhyp's search — the hypergraph
construction (§8.3) explicitly carves out an exception for operators
`JoinOrderConflictDetector::MustApplyAsOperator` flags, whose entire
already-planned subtree has to move through the search as one indivisible
unit rather than being decomposed relation-by-relation. `LEFT`/`RIGHT`/`FULL
OUTER` joins are the headline case (an outer join's semantics depend on
*which* side is NULL-extended, so a predicate above it can't always be
treated as if it applied to the join's raw inputs), and `SEMI`/`ANTI` joins
are a related but distinct case (§7.26.3's `no_cross_product_reordering.test`
already showed one angle of this). This section's five files — four of them
concentrated in the `left_join_reordering/` subdirectory, which exists
specifically for this topic — work through that boundary in real depth.

### `left_join_reordering/complex_left_joins.test`

This is the largest, most systematically organized file in the whole
directory: a small, four-table fixture (`t1(a): {1,2,3,4}`, `t2(b): {1,2,3}`,
`t3(c): {2,3,4}`, `t4(d): {2,3}`) run through dozens of combinations —
`(t1 ⨝ t2) ⟕ t3` vs. `(t1 ⟕ t2) ⨝ t3`, chained `LEFT JOIN`s, `LEFT` mixed with
`SEMI`/`ANTI` (via `IN`/`EXISTS`/`NOT IN`/`NOT EXISTS`), inequality
predicates, arithmetic (`a + b = 4`) predicates, `AND true`/`AND false`
conditions on a `LEFT JOIN`'s `ON` clause, and multi-condition `ON` clauses —
each with an exact expected result set, run with `build_side_probe_side`
disabled (so the *shape* of the reordering, not incidental build/probe
flipping, is what's being checked). The organizing principle running through
every block is the same one Chapter 8's non-reorderable-operator aside
gestures at but doesn't spell out concretely: an `INNER` join placed *after* a
`LEFT JOIN` can be freely reordered underneath it **only if its predicate
references solely the `LEFT JOIN`'s LHS** (the always-preserved side); if it
references the `LEFT JOIN`'s RHS (the nullable side) at all, that `INNER` join
is pinned above the `LEFT JOIN` and can't be pulled below it, because doing so
would let the `INNER` join's predicate silently discard the very NULL-extended
rows the `LEFT JOIN` exists to keep. Two comment-annotated blocks state this
in the file's own words: *"forces LOJ to be preserved"* (RHS-referencing
case) vs. *"can be freely reordered"* (LHS-only case). A later block on the
same theme (*"A post-LEFT SEMI join must not be reconstructed as a filter,
which would preserve RHS duplicates"*) generalizes the same idea to `SEMI`
joins sitting above a `LEFT JOIN`: a `SEMI` join's whole point is to collapse
potential RHS duplicates down to at most one output row per matching LHS row,
and if reconstruction ever "simplified" a post-`LEFT`-join `SEMI` join into an
ordinary filter expression instead of keeping it a real `SEMI` join node, that
deduplication guarantee would quietly evaporate.

### `left_join_reordering/non_inner_join_edge_constraints.test`

Where `complex_left_joins.test` is almost entirely *correctness-of-output*
testing (does the result set match, for many plan shapes), this file is more
explicitly about the *enumeration* logic itself — several blocks read like
they were built directly from named cases in the join-ordering literature.
One block cites its source outright:

```
# test/optimizer/joins/left_join_reordering/non_inner_join_edge_constraints.test:86
# Figure 14 from Moerkotte et al. (SIGMOD 2013): applying the ANTI join above the LEFT join is invalid.
```

— a direct reference to the same research lineage Chapter 8 §8.5 already
cited for DPhyp itself (Moerkotte and Neumann's "Dynamic Programming Strikes
Back" [20]), here pointing at a *different* paper on the
same general problem [22, Fig. 14]
(reordering outer/semi/anti joins correctly, an area with its own dedicated
literature beyond the base DPhyp algorithm), used to justify a specific
enumeration constraint rather than the search structure itself. The file runs
every block twice — once under the exact search, once under the greedy
fallback (`foreach threshold 0 30`, exactly as in §7.26.3's
`required_cross_product_activation.test`) — which is itself a meaningful
design choice: a non-inner-join ordering *constraint* (as opposed to a pure
cost heuristic) has to be enforced identically by both `SolveJoinOrderExactly`
and `SolveJoinOrderApproximately` (Chapter 8 §8.5/§8.8), since a query that
happens to cross the 12-table threshold can't be allowed to suddenly produce
wrong results just because it took the greedy code path instead of the exact
one. Other blocks in this file check that a `SEMI`/`ANTI` join's LHS spanning
*several* relations at once still respects the same constraint (echoing
§7.26.3's `single_sided_predicate_reconstruction.test` theme of multi-relation
endpoints), that an explicit `CROSS JOIN` scope stays available as a distinct
plan option even when sibling `LEFT JOIN`s share both of its inputs, and — one
particularly pointed comment — that "conflict simplification must not split
an INNER occurrence whose complete condition is constrained by a LEFT join,"
i.e., the constraint-tracking machinery has to treat an `INNER` join's *whole*
condition as the unit that's either safely movable or pinned, not
accidentally split its clauses and move only some of them.

### `left_join_reordering/test_reordering_left_joins.test`

The oldest-feeling and most narratively told file of the four — its opening
comment, *"make sure no illegal left join reordering happens,"* is followed by
a labeled sequence of algebra-annotated cases (`(a ⟕ b) ⨝ c ON b.col = c.col`,
explicitly marked *"This was previously crashing with infinite recursion /
stack overflow"*) that read as a fix's own regression trail, one case at a
time, rather than a from-scratch systematic sweep like
`complex_left_joins.test`. A recurring, carefully-varied sub-theme here is
**post-`LEFT`-join filters that reference the nullable side**: several blocks
construct a filter like `WHERE l.v IS DISTINCT FROM r.v` or `r.v IS NOT
DISTINCT FROM l.v` sitting above a `LEFT JOIN`, and the comment states the
hazard directly — *"If this filter is reconstructed before the LEFT join, the
NULL-extended rows disappear."* This is a sharp, concrete instance of the
same general principle `complex_left_joins.test` established more broadly: a
`WHERE`-clause predicate that touches the nullable side of a `LEFT JOIN` has
to act as a hard ordering barrier during reconstruction, staying above the
join rather than being folded down into it or past it, precisely because
NULL-extended rows are exactly the rows such a predicate is often written to
detect (`... IS DISTINCT FROM ...` is a very natural way to write "the right
side didn't match"), and losing that barrier would filter out the rows the
query is actually asking for. The file's final block is worth flagging as a
direct sequel to Chapter 8 §8.9's build/probe-side material: a `LEFT JOIN`
against a small "optional" dimension table, joined after a highly selective
`INNER` join against an even smaller dimension, checks (via an `EXPLAIN`
regex) that the `LEFT JOIN` isn't planned *before* the selective inner join
just because it's syntactically written that way — the comment notes the
final physical plan "may be reconstructed as a RIGHT join after build/probe
swapping" (exactly Chapter 8 §8.9's right-child-is-always-build-side
convention), but the *logical* ordering constraint — the optional table stays
above the selective join, not interleaved arbitrarily with it — must hold
regardless of that later physical-side flip.

### `test_must_apply_residual_predicate.test`

**Category: a genuine correctness requirement, not an optimization.** A
"residual predicate," in this context, is a join condition that a specific
join algorithm can't fully discharge through its primary matching mechanism
(equality hashing, for a hash join) and that therefore has to be checked as an
extra, per-matched-pair condition after the primary match — DuckDB's hash join
supports exactly this: one or more equality conditions used to build/probe the
hash table, plus additional conditions (inequalities, or, as here, a
single-sided filter folded into an outer join's `ON` clause) checked as a
residual on each candidate pair the hash lookup produces. The file's own
description names the exact bug (issue 24690): *"Residual predicate belonging
to a must-apply join operator must not be consumed as an unused residual."*
Concretely: `e1 LEFT JOIN e2 ON (e1.x = e2.x) RIGHT JOIN e3 ON (e1.x = e3.x
AND e3.x > 0), e4 WHERE e4.x = e1.x` embeds a single-sided predicate
(`e3.x > 0`) directly inside a `RIGHT JOIN`'s `ON` clause, alongside its
equi-condition. Because outer-join `ON`-clause predicates can't legally be
relocated the way an ordinary inner-join filter could (pushing `e3.x > 0`
below the join instead of applying it exactly at the join would change which
RHS rows count as "matched," and therefore which rows get NULL-extended) —
this predicate is genuinely "must-apply": it has to be evaluated exactly at
this join operator, not before it, not after it, and not dropped. The bug
being guarded against is the join-order reconstruction pipeline treating this
kind of predicate as an ordinary, independently-relocatable "leftover" filter
it was free to move or, worse, silently drop as apparently redundant — the
first four blocks (against genuinely empty tables, so every variant trivially
evaluates to `count(*) = 0` regardless of whether the bug is present) are
really checking that this specific plan shape **plans and executes without an
internal error at all**; the fifth block, against tables with real data
(`t1`..`t4` of sizes 4/9/7/3), is the one block whose expected answer
(`count(*) = 1`) is actually sensitive to whether `t3.x > 1` is correctly
applied — and it passes, confirming the fix holds on non-trivial data too, not
just on the empty-table shape that would pass even with the bug present.

### `many_joins_and_one_non_reorderable_join.test_slow`

**Category: mixing a non-reorderable boundary with an internally-reorderable
subtree.** `test0 LEFT JOIN (SELECT test1.range AS the_range FROM test1,
test2, test3) other ON other.the_range = test0.range` puts a plain,
unfiltered three-way cross product (`test1, test2, test3` — no `WHERE` clause
at all connects them to each other) on the nullable side of a `LEFT JOIN`.
This tests something specifically *not* covered by
`left_join_reordering/`'s other files: it's not asking whether the `LEFT
JOIN`'s boundary is respected (that's the previous three files' whole point)
— it's asking whether the optimizer can correctly handle a **non-reorderable
join whose own child is itself a nontrivial join-order problem**. Internally,
`test1`/`test2`/`test3` genuinely have no connecting predicate at all, so
DPhyp's greedy-cross-product path (Chapter 8 §8.8's "no remaining pair has any
connecting edge" case) has to decide how to pair them up as its own
sub-problem — but whatever it decides, the *result* of that sub-problem has to
present itself to the enclosing `LEFT JOIN` as a single, correctly-bounded
unit that the `LEFT JOIN`'s own non-reorderable handling (Chapter 8 §8.3's
`JoinOrderConflictDetector::MustApplyAsOperator`) can treat as one opaque
child, without any of `test1`/`test2`/`test3` individually leaking out from
under the `LEFT JOIN` boundary to be reordered against `test0` directly. This
is the "mix of reorderable and non-reorderable joins in one query" case in its
most literal form: full freedom *inside* the non-reorderable operator's
subtree, zero freedom to cross its boundary.

### `pushdown_semi_anti.test`

**Category: SEMI/ANTI joins as ordinary, reorderable hyperedges (post a named
fix).** Two `EXPLAIN`s of the identical query
(`tbl1, tbl2 WHERE b IN (SELECT * FROM tbl3) AND tbl1.a = tbl2.b`) —
one with the full optimizer pipeline, one with `join_order` itself disabled
(`set disabled_optimizers='statistics_propagation,join_order'`) — check
literally opposite orderings of the same two operator kinds:
`INNER.*SEMI` (optimized) vs. `SEMI.*INNER` (unoptimized, matching however the
binder mechanically nested the `IN`-subquery's `SEMI` join around whatever the
`FROM` clause structure happened to produce). This is a small, concrete proof
that `SEMI` joins genuinely participate in DPhyp's search as reorderable
hyperedges — not merely tolerated as a fixed non-reorderable boundary the way
this section's `LEFT`/`RIGHT`/`FULL OUTER` material has been about — and it
connects directly to a comment already surfaced in §7.26.6's
`delim_join_dont_explode.test_slow`: *"Since PR #12290, we can reorder
DELIM_SCANS... Now that we reorder semi joins the ordering of the sequential
scans has also changed."* Both files are evidence of the same underlying
capability (reordering `SEMI` joins, added by that PR) observed from two
different angles.

### `join_bug.test`

**Category: an honestly untriaged fuzzer regression.** This file's own
description is refreshingly direct: *"unsure yet."* Its first query nests
`SEMI`/`INNER`/`NATURAL` joins several levels deep over a single self-joined
table (`v00`), with the comment *"don't delete numerator relations"* attached
— read plainly, this is guarding against an optimizer bug where a relation
serving as the "numerator" side of a nested `SEMI` join (i.e., a relation
whose only role is filtering, feeding into yet another `SEMI` join one level
up) was being incorrectly eliminated from the plan altogether, rather than
correctly retained as a filtering step. Its second query — *"Asof joins should
swap"* — is a `NATURAL JOIN` chained into an `ASOF JOIN` (`ta02 NATURAL JOIN
ta03 ASOF JOIN ta04 USING (c01)`), checking that whatever child-swapping logic
applies to an `ASOF` join (its build/probe or LHS/RHS orientation — recall
Chapter 8 §8.9's general build-side/probe-side machinery, and §7.26.8 below on
ASOF's own asymmetric behavior) still functions correctly when the join
immediately below it isn't a plain base-table scan but the result of a prior
`NATURAL JOIN`. Neither query result is checked against expected output —
both are `statement ok`, liveness-only, in the same spirit as §7.26.1's
`tpcds_nofail.test`. It's worth naming plainly, per this book's policy of
being honest about what's actually known rather than retrofitting false
confidence: not every regression test in a mature codebase carries a fully
worked-out narrative behind it. Some exist purely because a fuzzer found an
input that broke something, the fix's author confirmed the fix worked, and
left the test in the suite with an honest "unsure yet" rather than inventing a
tidier story after the fact.

## 7.26.6 Correlated subqueries and delim-joins meeting join order

Chapter 6 covered `FlattenDependentJoins`, the `LOGICAL_DELIM_JOIN`/
`LogicalDelimGet` machinery, and the `Deliminator`/`UnnestRewriter` optimizer
passes that clean up after decorrelation, in full — including the precise
point (§6.7) that `Deliminator` and `UnnestRewriter` are optimizer-level
cleanup of an already-decorrelated tree, not decorrelation itself, and that
`Deliminator` runs at pipeline position #10 (before `JOIN_ORDER` at
position #17 — Chapter 7 §7.4's numbering). This section is not a second
explanation of any of that; it's about what happens when the join-order
optimizer, running *after* decorrelation and *after* `Deliminator`'s first
cleanup pass, encounters the `DELIM_SCAN`/`DELIM_JOIN` structure that
decorrelation leaves behind.

Note that with default settings the join-order optimizer does not meet a
`LOGICAL_DELIM_JOIN` at all. The setting `delim_join_as_cte` defaults to
`true`, so right after flattening the planner turns every delim join into
plain joins over materialized CTEs (Chapter 6 §6.4.1). Join ordering then
sees ordinary comparison joins and `LogicalCTERef` leaves. The
`LOGICAL_DELIM_JOIN` case in `RelationManager::ExtractJoinRelations`
(Chapter 6 §6.10) only runs when `delim_join_as_cte = false`. That is why
several tests below accept either a `DELIM_SCAN` or a
`CTE Name: __duckdb_delim` pattern. None of the tests in this section set
`delim_join_as_cte`, so a default run of each one takes the CTE form.

**`joins_with_correlated_subqueries.test`** is the file this part was asked
to hold up against Chapter 6 explicitly, and the cross-reference is direct:
its query is a `COALESCE` wrapping a genuinely correlated scalar subquery
(referencing `parent.x` from the enclosing `df parent` row) that itself
contains a `CROSS JOIN` between two further-correlated subqueries (`prev`,
ordered descending and limited to find the nearest preceding non-null row, and
`next`, ordered ascending and limited to find the nearest following one) — a
linear-interpolation-style "fill forward/backward" idiom. This is precisely
the shape Chapter 6 §6.3's `FlattenDependentJoins` walkthrough describes: a
correlation parameter (`parent.x`) threaded through a `LIMIT`-bearing
subquery, which §6.6's "decoupling" discussion already flags as one of the
harder cases for decorrelation to get right cleanly. The join-order-relevant
content here is what happens *after* `FlattenDependentJoins` has done its
work: the two now-decorrelated, `LIMIT`-bearing subqueries (`prev`/`next`) are
cross-joined, and that cross join — like every cross product this directory
studies — has to be recognized by the join-order optimizer as a genuinely
required one (nothing connects `prev` and `next` to each other directly; they
only jointly participate in the outer `COALESCE` expression), correctly costed
using the `LIMIT`-derived cardinality reasoning `get_cardinality_from_limit.test_slow`
(§7.26.2) already demonstrated, and correctly reconstructed as one of the
DELIM-side subtrees a `LOGICAL_DELIM_JOIN` carries (or, in the default CTE
form, as part of the subtree under the generated CTEs). The test itself only
checks that the query runs (`statement ok`, no result set) — its value, per
its own description citing an internal issue number, is as a regression
against whatever specific interaction between `LIMIT`-bearing correlated
subqueries, a `CROSS JOIN` between them, and join-order reconstruction that
issue originally reported.

**`delim_left_join_pushdown.test`** is new, join-order-adjacent territory
beyond what Chapter 6 covers: its own description flags the risk directly —
*"Verify left-join pushdown into lateral joins is not over-eager."* The query
is a `LEFT JOIN` against a correlated, aggregate-producing subquery
(`array_agg` over rows matching `c.u_id = ANY(t.u_ids)`), which the binder
represents as a dependent join needing decorrelation. The specific hazard: the
general filter-pushdown machinery (Chapter 7 §7.4 pass #6,
`FILTER_PUSHDOWN`) knows how to push filters through the *preserved* side of a
`LEFT JOIN`, and separately knows how to push filters into a lateral/dependent
join's correlated side — but combining both movements naively could push a
filter meant to apply *after* the `LEFT JOIN`'s NULL-extension has already
happened (the file's two blocks both filter on `ts.names`, the subquery's own
output column, which is only meaningfully non-NULL after the join) down
*into* the lateral subquery's correlated evaluation instead, which would
change which rows the `LEFT JOIN` correctly reports as unmatched (`NULL`)
versus which rows the pushed-down filter incorrectly excludes before the
`LEFT JOIN` even gets to NULL-extend them. Both blocks check the correct,
non-over-eager output directly (`c.id = 4`, whose `u_id` is `NULL` and whose
subquery therefore correctly returns no rows, still appears correctly in the
second block's output as `4 | NULL`, not silently dropped).

**`delim_join_dont_explode.test_slow`** already surfaced its own most useful
sentence above (§7.26.5's `pushdown_semi_anti.test` discussion): a comment
explicitly narrating a plan-shape *change*, not a regression in the usual
sense — *"Since PR #12290, we can reorder DELIM_SCANS... Now that we reorder
semi joins the ordering of the sequential scans has also changed"* — which is
exactly the two-shared-EXPLAIN-label pattern this book's test-format note
describes (a plan-shape assertion whose expected regex was deliberately
updated alongside a real optimizer capability change, with the comment
preserving *why* the old expectation no longer holds, rather than silently
overwriting it). The file's actual query — an `EXISTS` subquery correlated
through a modulo condition (`medium_1.fk_to_big % 7 = bt.id % 7`) against a
1,000-row outer table — is this section's clearest illustration of "don't
explode": before decorrelation and `Deliminator`'s cleanup were both working
correctly together, a correlated `EXISTS` like this risked being evaluated in
a way whose cost scaled with `|outer| × |subquery|` rather than being properly
turned into a single decorrelated join, and the file's name is a direct
statement of that risk (this is precisely the failure mode a `DelimJoin`'s
whole design, per Chapter 6 §6.4, exists to avoid — materializing D once via
a real `PhysicalHashAggregate`, rather than re-evaluating the correlated
subquery once per outer row). The `EXPLAIN` assertion's `<REGEX>` alternation
(`.*HASH_JOIN.*DELIM_SCAN.*SEQ_SCAN.*` **or** `.*CTE Name:.*__duckdb_delim.*
HASH_JOIN.*SEQ_SCAN.*CTE_SCAN.*`) is itself worth noting as a second instance
of the same test-format idiom the front matter flagged — two structurally
different but semantically equivalent plan shapes are both accepted, because
which one DuckDB actually produces for this query can legitimately vary with
unrelated optimizer changes (whether the DELIM side ends up represented as a
raw `DELIM_SCAN` or gets further rewritten into a materialized CTE), and the
test is written to be robust to that rather than pinned to one specific
rendering.

**`delim_join_with_in_has_correct_results.test`** is a small, sharp
correctness point, not a join-order-search behavior at all: `t0 IN (SELECT u0
FROM u WHERE t0 = 4)` over a table `t` with one `NULL` row and one `4` row,
against a table `u` containing only two `NULL` rows. Per standard SQL
three-valued `IN` semantics (already covered generally, but worth restating
precisely here because this test pins the exact result down): when `t0 = 4`,
the correlated subquery's `WHERE` clause matches, but its *result set* is
`{NULL, NULL}` — comparing `4 IN (NULL, NULL)` correctly yields `NULL` (can't
prove `4` equals nothing, can't disprove it either — SQL's `IN` returns `NULL`
rather than `false` when at least one comparison is indeterminate and none is
a definite match). When `t0` is `NULL`, the subquery's own correlation
predicate (`t0 = 4`) itself evaluates to `NULL`, so the subquery's `WHERE`
clause filters out every row of `u`, leaving `u0` genuinely empty — and `NULL
IN (<empty set>)` is unambiguously `false`, regardless of what `NULL`
comparisons would otherwise do, because an empty `IN` list has no candidate to
be indeterminate about. The file's own description states exactly this edge
case: *"An IN expression should return false when the IN list is empty."*
This is a `DelimJoin`-family test (the `IN`-subquery here is exactly the shape
`FlattenDependentJoins`/`Deliminator` process) but its content is really about
NULL-handling correctness surviving the decorrelation rewrite intact, not
about join *ordering* per se.

**`test_delim_join_with_cross_product_in_rhs.test`** names its own hazard
precisely: *"Verify that a delim join with a correlated column in the RHS of a
cross product (on the RHS of the delim GET) is properly bound."* `t3, t1
INNER JOIN (SELECT t3.c0) AS subQuery1 ON (t1.c0 > t3.c0::DOUBLE)` places a
correlated reference to `t3.c0` inside a subquery that itself sits on the RHS
of what the binder initially represents as a cross product (`t3, t1 ...`)
before decorrelation restructures it — precisely the kind of nested,
multiply-correlated shape where a `LogicalDelimGet`'s binding (Chapter 6 §6.4)
has to correctly track which enclosing `LOGICAL_DELIM_JOIN` it belongs to and
which specific correlation column it's standing in for, even when the path
from the `DelimGet` back up to its owning `DelimJoin` passes through an
intervening cross product rather than a direct parent-child edge.

**`lateral_cross_join.test`** is a smaller, purely correctness-of-`to_string`/
result-shape test: a `WITH`-CTE (`date_range`, an aggregate with no `GROUP
BY`, hence guaranteed exactly one row) is `CROSS JOIN`ed against a fact table
and then `LEFT JOIN`ed against a mapping table with a compound predicate
mixing an equality (case-insensitive, via `upper(...)`) and a `BETWEEN`-style
range check spanning the CTE's two aggregate columns. Its description —
*"test to string of complex lateral cross join"* — suggests its original
purpose was specifically about the plan (or expression) `ToString()`
representation of this particular nested shape rendering correctly, rather
than about join-order search behavior; it's included here as the directory's
representative case for "does a `CROSS JOIN` immediately followed by a
`LEFT JOIN`, both against subquery/CTE-shaped relations rather than plain
base tables, plan and print correctly," which is a real, if narrower, sibling
concern to this section's other files.

**`filter_on_subquery_with_aggregate.test`** rounds out this section with a
fuzzer-derived case (its description: *"some fuzzer issues"*) that isn't
really about correlation at all — a `WITH`-CTE filters and then self-joins
against `MAX(A)` computed from itself in a scalar subquery
(`WHERE A = (SELECT MAX(A) FROM cte)`) — but is grouped here because its shape
(a CTE referenced twice, once directly and once inside a scalar subquery
computed *from* it) sits at the same general intersection of CTE-handling
(`CTE_INLINING`, Chapter 7 §7.4 passes #2/#12) and subquery-decorrelation
machinery this section's other files exercise, even though its specific
content is closer to a plain aggregate/CTE-reference-counting correctness
check than to join reordering proper.

## 7.26.7 Floating-point equality: a correctness story, not a cardinality one

**`join_on_doubles.test`** and **`join_on_floats.test`** (identical in
structure, one per type) are worth being precise about, because their
filenames and directory location invite a natural but, on close reading of the
actual test body, **incorrect** guess: that they're about cardinality
estimation being harder for floating-point columns because exact
equality-based distinct-count reasoning doesn't transfer cleanly to a
floating-point domain. That is not what these files test. Both files' own
description states the real subject directly: *"tests for #16901 / #16965"* —
two GitHub issue numbers — and the bodies contain no `EXPLAIN`, no
cardinality assertion, and no statistics-related setup at all. What they
actually check is a join **correctness** edge case: `x` holds `{0.0, NaN}`,
`y` holds `{-0.0, -NaN}`, and every join variant (`INNER`, both directions,
`RIGHT`, `LEFT`) on `a = b` is expected to match `0.0` with `-0.0` **and**
`NaN` with `-NaN`.

Tracing this against the actual source confirms a specific, two-part
mechanism, not a cardinality-estimation story at all:

1. **DuckDB defines a SQL-specific total order for floating-point equality
   that differs from raw IEEE-754 `==`.**
   `src/common/vector_operations/comparison_operators.cpp:24-40` specializes
   `Equals::Operation` for `float`/`double`:
   ```cpp
   // src/common/vector_operations/comparison_operators.cpp:24-30
   template <class T>
   static bool EqualsFloat(T left, T right) {
       if (DUCKDB_UNLIKELY(Value::IsNan(left) && Value::IsNan(right))) {
           return true;
       }
       return left == right;
   }
   ```
   Under raw IEEE-754, `NaN == NaN` is `false` — by design, NaN compares
   unequal to everything, including itself. DuckDB's SQL semantics
   deliberately override this (the same choice PostgreSQL makes, for the same
   reason: a total order is needed for `GROUP BY`/`DISTINCT`/sorting/joining
   to behave sensibly on data containing NaN), treating **any** NaN bit
   pattern — `Value::IsNan` only checks `std::isnan`, so this is agnostic to
   NaN's sign bit or payload bits — as equal to any other NaN. The `0.0`/`-0.0`
   half of the test needs no special case at all: IEEE-754 already defines
   signed zero as comparing equal under plain `==`, so `left == right` alone
   already returns `true` for `0.0 == -0.0` before the NaN branch is ever
   reached. A matching override exists for ordering too
   (`GreaterThanFloat`/`GreaterThanEqualsFloat`, same file, lines 42-95): NaN
   is defined to compare as **greater than every other value**, a specific,
   documented total-order convention, not an arbitrary implementation
   accident.
2. **The hash function used for hash-join bucketing is kept consistent with
   that same equality relation.** A hash join's build phase (Chapter 8 §8.9's
   build-side machinery) buckets rows by `Hash()`, and any two values the SQL
   `Equals::Operation` considers equal *must* hash identically, or a hash join
   would never even attempt the residual per-pair comparison that would
   otherwise have confirmed the match. `Hash(float)`/`Hash(double)`
   (`src/common/types/hash.cpp:32-58`) apply a
   `FloatingPointEqualityTransform` before hashing the raw bit pattern:
   ```cpp
   // src/common/types/hash.cpp:32-41
   template <class T>
   struct FloatingPointEqualityTransform {
       static void OP(T &val) {
           if (val == (T)0.0) {
               // Turn negative zero into positive zero
               val = (T)0.0;
           } else if (std::isnan(val)) {
               val = std::numeric_limits<T>::quiet_NaN();
           }
       }
   };
   ```
   `-0.0` is canonicalized to `+0.0`, and *every* NaN bit pattern (including a
   negative-signed NaN, which is what `-NaN` literally is at the bit level) is
   canonicalized to one single, fixed `quiet_NaN()` representation before
   hashing — so `0.0` and `-0.0` land in the same hash bucket, and every NaN
   variant lands in the *same* bucket as every other NaN variant, agreeing
   exactly with the `Equals::Operation` override above. Without this, a hash
   join could easily place `0.0` and `-0.0` (or two differently-signed NaNs)
   in different buckets purely because their raw bit patterns differ, and
   never even reach the row-level equality check that would have correctly
   matched them.

Read this way, `join_on_doubles.test`/`join_on_floats.test` are genuinely
join-adjacent (they live in this directory because the mechanism they test is
exercised specifically *through* a join predicate, across all four join
directions), but the actual subject is **join equality correctness for
special floating-point values**, powered by a coordinated `Equals`/`Hash`
pair that both diverge from bare IEEE-754 semantics in the same,
deliberate way — not a distinct-count or cardinality-estimation concern at
all. It's a useful, concrete example of this book's general policy: a
plausible-sounding hypothesis about what a test must be checking, based on its
name and location, is worth stating and then actually verifying against the
file and the source it exercises, because sometimes — as here — the real
answer is a different, and in this case rather more interesting, mechanism
entirely.

## 7.26.8 ASOF JOIN: asymmetric semantics meet the optimizer

**`asof_join_adds_rows.test`** and **`cross_join_and_unnest_dont_work.test`**
need to be discussed together, because a direct `diff` shows they are, byte
for byte, the same file — every line identical except the `# name:` header
comment. This is worth stating plainly rather than writing two independent
narratives that happen to describe identical content: at some point two
differently-motivated bug reports (one framed around ASOF's row-count
behavior, one framed around a cross-join/`UNNEST` combination "not working")
were evidently reproduced with the *same* underlying repro case, and both
copies survived into the test suite under their own names — a small, honest
artifact of how a real, actively-developed test suite accumulates, not a
carefully unified design.

The content itself is genuinely two related things bundled into one file.
**Part one** builds a `small_probe` table (6 timestamped rows) cross-joined
with a trivial one-row `child_join` table and filtered
(`select * from small_probe, child_join where c=sp_const`), then `ASOF JOIN`s
the result against a `large_build` table (16 rows) on
`lb_const = sp_const AND a < b`, run inside `foreach threshold 0 32` over
`PRAGMA asof_loop_join_threshold = {threshold}`. Reading
`PhysicalPlanGenerator::PlanAsOfJoin` directly explains exactly what this
loop exercises:

```cpp
// src/execution/physical_plan/plan_asof_join.cpp:386-396
		const idx_t asof_join_threshold = Settings::Get<AsofLoopJoinThresholdSetting>(context);
		if (has_predicate || (op.children[0]->has_estimated_cardinality && lhs_cardinality < asof_join_threshold)) {
			auto result = PlanAsOfLoopJoin(op, left, right, join_cardinality);
			if (result) {
				return *result;
			}
			if (has_predicate) {
				//	PhysicalAsOfJoin only evaluates the comparisons
				throw NotImplementedException("Unsupported ASOF JOIN: the join predicate cannot be evaluated together "
				                              "with these join conditions");
			}
		}
```

Three details of this code are worth a note. First, `PlanAsOfLoopJoin` gets
a `join_cardinality` argument. `PlanAsOfJoin` computes it with
`op.EstimateCardinality(context)` *before* it plans the children
(`plan_asof_join.cpp:373-374`). Planning a child uses up the bind data of
its scans, so estimating afterwards could crash. Second, `PlanAsOfLoopJoin`
returns `nullptr` (no loop plan) when a bound aggregate argument is not a
plain column reference. Third, if the join has an extra predicate and no
loop plan can be built, DuckDB throws `NotImplementedException`. It must not
fall through to the sort-based operator, which does not check that
predicate.

`asof_loop_join_threshold` (default `64`, documented as *"the maximum number
of rows we need on the left side of an ASOF join to use a nested loop
join"*) directly reads `op.children[0]->has_estimated_cardinality` — this is
the exact same `LogicalOperator`-level field Chapter 5 defines and
§7.26.2's `estimated_cardinalities_are_in_logical_plan.test` proved is
genuinely populated and readable at this stage, here consumed by a *physical*
planning decision rather than by the join-order optimizer itself. With
`threshold = 0`, `lhs_cardinality < 0` is never true for an unsigned
`idx_t`, so the general, sort/IEJoin-style ASOF operator is always chosen;
with `threshold = 32`, the LHS here (`small_probe` cross `child_join`,
filtered — six rows) is comfortably below `32`, so the specialized
nested-loop `PlanAsOfLoopJoin` path is chosen instead. Running the identical
query and expecting the identical, exact result set under both settings is a
direct proof that DuckDB's **two independent physical implementations** of
`ASOF JOIN` agree — including when the ASOF join's LHS isn't a plain
base-table scan but the filtered output of a cross product, which is
precisely the case that makes `has_estimated_cardinality`'s plumbing through
an intervening `CROSS_PRODUCT`/`LogicalFilter` pair matter at all (if that
plumbing were broken, one of the two thresholds could silently plan the wrong
physical operator, or crash outright, rather than merely producing a
different but still-correct plan).

**Part two** is a considerably more elaborate CTE chain combining `ASOF LEFT
JOIN` with `LEFT JOIN LATERAL UNNEST(... ->> '$.amp[*].k')` — a correlated,
JSON-array-unnesting lateral join, decorrelated by exactly the
`FlattenDependentJoins`/`UnnestRewriter` machinery Chapter 6 covers — followed
by a *second* `ASOF LEFT JOIN` against a further table. This is where the
filename `cross_join_and_unnest_dont_work.test` earns its honesty-worth-
flagging status: read at face value, the name promises a documented
*limitation*; the actual body is a regression test proving this exact
combination (`ASOF` + correlated `LATERAL UNNEST` + a second `ASOF LEFT JOIN`)
**does** produce the correct two-row result. The most defensible reading,
consistent with everything actually in the file, is that both names describe
the same historical bug from two angles that were both true *before* the fix
landed: the combination used not to work (hence one name), and — because ASOF
JOIN's matching semantics are inherently asymmetric (each LHS/probe row
matches **at most one** RHS/build row — the single nearest qualifying
row — unlike an ordinary equi-join, which can fan out to every matching RHS
row), a bug in how that asymmetry interacted with a cross-product-shaped or
LATERAL-UNNEST-shaped probe side could plausibly have manifested as spurious
*extra* output rows rather than a clean failure (hence the other name,
`asof_join_adds_rows`). Both files, being identical, now serve as the same
regression guard against that class of bug recurring, verified against two
different physical ASOF implementations at once via the threshold loop.

## 7.26.9 Build side / probe side, briefly

Two files in this directory are squarely about build-side/probe-side
assignment (Chapter 8 §8.9) rather than about join *ordering* per se, and
Chapter 7 Part 15 (`15-build-side-probe-side-test-driven.md`) already gives
both genuinely full, source-grounded treatment — reproducing them again here
would be pure duplication, so this section is deliberately a pointer, not a
retelling.

**`wide_build_skinny_probe.test`** joins a 10,000-row narrow table (`skinny`,
three plain `INTEGER` columns) against an 8,000-row table with over thirty
wide `VARCHAR` columns (`wide`), and checks — via a JSON-formatted `EXPLAIN`
regex asserting `wide` is serialized before `skinny` in the physical plan's
child order — that `BuildProbeSideOptimizer::GetBuildSize`
(`build_probe_side_optimizer.cpp:188-222`, cited in full in Ch. 8 §8.9) chooses
the *narrower* table as the build side even though it isn't the smaller
table by row count, because build-side cost is a function of estimated
per-row byte width times cardinality, not row count alone. Part 15 §7.15.3
works through this file's exact byte-width arithmetic (comparing `wide`'s
~933-byte rows against `skinny`'s ~65-byte rows) in full.

**`join_filter_build_side.test`** is Part 15's §7.15.5 subject: a filtered
build-side table (`jf_build`, with a selective `keep` predicate) is kept on
the build side even though the plain memory-footprint heuristic alone would
otherwise favor the much smaller `jf_probe`, because — per the
`DynamicFilterBuildBonus`/`JoinFilterBuildSideHeuristics` refinement Chapter
8 §8.9 names but doesn't detail — a filtered build side can produce a
tighter dynamic min/max range to push down as a runtime pre-filter into the
probe side's scan (Chapter 7 §7.4 pass #42, `JOIN_FILTER_PUSHDOWN`), which is
worth more than the raw byte-width comparison alone would suggest.

## 7.26.10 `test_tpcds_pushdown.test`: an (almost) empty file

Worth stating plainly rather than silently skipping: `test_tpcds_pushdown.test`
contains exactly three lines — a `# name:` header, a `# group: [joins]` line,
and nothing else. No SQL, no setup, no assertion. Its filename (and, by
extension, its evident original intent — "filter pushdown interacting with
join ordering at TPC-DS scale") is the only information this file actually
carries in this checkout; whatever body it once had, or was meant to have, is
not present at commit `cb5673685153`. This is included here rather than
silently omitted for the same reason `join_bug.test`'s "unsure yet" was kept
in §7.26.5: an honest accounting of a real test directory includes its
placeholders and its stubs, not only its fully-realized files.

## 7.26.11 What to carry forward

Three things from this directory are worth having crisp, on top of everything
Chapter 8 already established about the algorithm itself: **(1)** the
denominator/cardinality-estimation machinery this part's §7.26.2 walked
through in depth is not a monolithic formula — it's a layered system of
equality-class union-finds, per-domain-group tracking, and an explicit
FK/PK composite-key cap, each with its own, separately-testable failure
modes, several of which trace back to real, named historical bugs (the
`MarkEdgeEquivalences`-before-`CreateHyperGraphEdges` ordering bug in
`fkpk_composite_key_ce.test` chief among them); **(2)** "reorderable" is not a
binary property of a join node — `SEMI`/`ANTI` joins genuinely participate in
DPhyp's search as ordinary hyperedges (since PR #12290), while `LEFT`/`RIGHT`/
`FULL OUTER` joins impose real, asymmetric constraints on what may cross their
boundary during enumeration, and a single query can freely mix fully-
reorderable and rigidly-non-reorderable subtrees, provided the boundary
between them is respected exactly; **(3)** a mature test suite's regression
files are not uniformly polished teaching examples — `join_bug.test`'s
"unsure yet," `test_tpcds_pushdown.test`'s empty body, and
`asof_join_adds_rows.test`/`cross_join_and_unnest_dont_work.test`'s
byte-identical duplication are all, in their own way, more honest evidence of
how a real optimizer's correctness gets defended over years of production use
than a from-scratch, pedagogically tidy test file could be — and reading them
this closely is the only way to tell the difference between what a test's
name promises and what its body actually proves.

One more point is worth adding. **(4)** Many tests in this directory guard
the *rebuild* step, not the search.
After DPhyp or the greedy fallback picks an order, join ordering must put
back every filter it took off (also filters with no columns), keep volatile
filters on the rows they were written for, keep the order of stacked
filters, and keep the column layout that the operator above expects. The
search itself is kept fast by scoring each pair once (§7.26.1), and the
estimates have special rules for `<>`, for always-true scan filters and for
composite `LEFT JOIN` keys (§7.26.2).
