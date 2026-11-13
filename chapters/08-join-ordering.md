# Chapter 8 — Join Ordering

## 8.1 Why this needs an entire chapter

Given a `FROM` clause with `n` tables and enough join predicates to connect them,
there is more than one way to compute the same result. `A JOIN B JOIN C` can be
executed as `(A ⋈ B) ⋈ C` or as `A ⋈ (B ⋈ C)` or, if a predicate connects `A` and
`C` directly, as `(A ⋈ C) ⋈ B` — all three return the same rows, but on real data
their costs can differ by orders of magnitude, because joining a 200-row filtered
table against a 6-million-row table first, then joining the *result* against a
second 6-million-row table, is a completely different amount of work than joining
the two 6-million-row tables together first and filtering afterward. Picking the
tree shape (and, at every node in that tree, which side is used as the algorithm's
build input — §8.9) is the job of the pass this chapter is about:
`OptimizerType::JOIN_ORDER`. It is pass #17 of the optimizer's fixed ~45-pass
pipeline, counted the way Chapter 7's pass inventory counts
(`src/optimizer/optimizer.cpp:364`).

Before getting into how DuckDB solves this, it's worth being precise about *why*
it can't simply try every tree shape and keep the cheapest one — "just enumerate
them all" is the obvious algorithm, and the reason no serious query optimizer does
it this way is a clean piece of combinatorics.

**Counting join-tree shapes.** A join of `k` relations needs `k − 1` binary join
operations arranged into a full binary tree with `k` leaves (one leaf per
relation). The number of distinct ways to shape such a tree — i.e., the number of
distinct full parenthesizations of `k` factors, ignoring for now *which* relation
sits at which leaf — is a Catalan number. The Catalan numbers are defined by the
recurrence

```
C₀ = 1
Cₙ = Σ (for k = 0 to n−1) Cₖ · Cₙ₋ₖ₋₁          (n ≥ 1)
```

which says: to build a full binary tree with `n + 1` leaves, pick where the root's
two subtrees split (`k` leaves on the left, `n − k` on the right), and multiply the
number of ways to shape each side. Because `Cₙ` counts trees with `n + 1` leaves, a
join of `k` relations has `C_{k−1}` possible shapes. Tabulating `Cₙ` for `n = 0`
through `10` (computed directly from the recurrence above — this is the standard
Catalan sequence, and it matches the growth curve DiDi's join-ordering slide deck
uses to make the same point [10]):

| `n` | `Cₙ` | relations `k = n+1` | shapes for a `k`-relation join |
|----:|-----:|---------------------:|-------------------------------:|
|  0  |     1 |  1  |     1 |
|  1  |     1 |  2  |     1 |
|  2  |     2 |  3  |     2 |
|  3  |     5 |  4  |     5 |
|  4  |    14 |  5  |    14 |
|  5  |    42 |  6  |    42 |
|  6  |   132 |  7  |   132 |
|  7  |   429 |  8  |   429 |
|  8  |  1430 |  9  |  1430 |
|  9  |  4862 | 10  |  4862 |
| 10  | 16796 | 11  | 16796 |

That table alone already makes the point — an 11-relation join has 16,796 distinct
*shapes* before you've even decided which of the 11 relations goes in which leaf.
And "shapes" undercounts the real search space, because the optimizer also has to
decide which specific relation occupies which leaf position. The number of
distinct, fully-labeled join trees over `k` *distinguishable* relations (shapes ×
leaf-assignments) is `C_{k-1} · k!`, which has the closed form `(2(k-1))! /
(k-1)!`. For `k = 10` relations, that's `18! / 9! = 17,643,225,600` — over 17
billion distinct join trees, for a query that TPC-H would consider unremarkable.
Naive enumeration is not an option past a handful of tables, and this is true even
before accounting for the fact that most of those 17 billion trees are physically
absurd (they'd require joining two tables with no predicate connecting them —
a cross product — somewhere in the middle of the plan). The rest of this chapter
is about the two ideas DuckDB actually uses instead: prune the search to only the
trees a *hypergraph of join predicates* says are connected (§8.3–§8.5), and when
even that search space is too large, fall back to a cheap greedy heuristic instead
of giving up on ordering altogether (§8.8).

## 8.2 Query C: this chapter's worked example

The front matter's Query A (the `orders ⋈ lineitem` aggregate) only has one join,
so it never exercises anything this chapter cares about — with one join there is
nothing to *order*. This chapter introduces a third running example, **Query C**,
a TPC-H-flavored three-way join written with implicit (comma) join syntax, exactly
the shape that forces a real ordering decision:

```sql
-- Query C
SELECT c.id
FROM customer c, orders o, lineitem l
WHERE c.id = o.custkey
  AND o.orderkey = l.orderkey
  AND c.nation = 'FR';
```

Three base relations (`customer`, `orders`, `lineitem`), two equi-join predicates
connecting them in a chain (`customer — orders — lineitem`, with no predicate
directly relating `customer` and `lineitem`), and a single-table filter on
`customer` that — as we'll see in §8.6 — matters just as much to the final
decision as the join graph's shape does, because it changes one leaf's estimated
cardinality before the DP search ever starts.

## 8.3 The join hypergraph

By the time `JoinOrderOptimizer::Optimize` (`src/optimizer/join_order/join_order_optimizer.cpp:57`)
gets control, the binder has already produced an ordinary logical plan — a tree of
`LogicalGet`, `LogicalComparisonJoin`, and `LogicalFilter` nodes, joined however the
`FROM`/`WHERE` clause happened to nest them. The join-order optimizer's first job
is to tear that structure apart and rebuild it as a graph: `QueryGraphManager::Build`
walks the plan, pulls out every base relation it can legally reorder as a
`SingleJoinRelation` (`relation_manager.cpp`), and pulls out every join predicate
as a `FilterInfo`/`JoinPredicate` (`filter_info.cpp`, `join_predicate.cpp`). Once
every relation has been assigned a `RelationIndex` (a plain, zero-based integer:
`customer = 0`, `orders = 1`, `lineitem = 2`, in the order the relation manager
happens to encounter them for Query C), `QueryGraphManager::CreateHyperGraphEdges`
(`src/optimizer/join_order/query_graph_manager.cpp:404`) turns the predicates into
graph edges:

```cpp
// src/optimizer/join_order/query_graph_manager.cpp:419-432
for (auto predicate_ref : predicate_model.GetGraphPredicates()) {
	auto &predicate = predicate_ref.get();
	D_ASSERT(predicate.GetLeftSetOptional() && predicate.GetRightSetOptional());
	auto operator_index = filters_and_bindings[predicate.GetIndex()]->source_operator_index;
	if (operator_index.IsValid()) {
		D_ASSERT(operator_index.GetIndex() < join_operators.size());
		if (JoinOrderConflictDetector::MustApplyAsOperator(*join_operators[operator_index.GetIndex()])) {
			continue;
		}
	}
	query_graph.CreateEdge(predicate.GetLeftSet(), predicate.GetRightSet(), predicate);
	query_graph.CreateEdge(predicate.GetRightSet(), predicate.GetLeftSet(), predicate);
	ConnectGraphComponents(predicate.GetLeftSet(), predicate.GetRightSet());
}
```

For Query C this produces exactly the graph you'd sketch on paper: two nodes
connected by an edge for each equi-join predicate, and no edge at all between
`customer` and `lineitem`, because no predicate mentions both:

```
   customer ────────── orders ────────── lineitem
   (0)         c.id =    (1)   o.orderkey=   (2)
               o.custkey        l.orderkey
```

That header comment in `query_graph_manager.hpp:59` calls the structure that holds
these edges "a reference to the `QueryGraphEdges` structure that stores edges
between nodes **and hypernodes**" — and the "hyper" part is not decoration. Look
at the loop directly below the one just quoted, which handles join predicates that
live on **non-reorderable** operators (an `OUTER`/`SEMI`/`ANTI` join, or anything
`JoinOrderConflictDetector::MustApplyAsOperator` decides imposes an ordering
constraint DPhyp has to respect):

```cpp
// src/optimizer/join_order/query_graph_manager.cpp:448-467
bool created_edge = false;
for (auto predicate_index : op.costing_predicate_indices) {
	// ...
	query_graph.CreateEdge(op.left_total_set, op.right_total_set, *predicate, op);
	query_graph.CreateEdge(op.right_total_set, op.left_total_set, *predicate, op);
	ConnectGraphComponents(op.left_total_set, op.right_total_set);
	created_edge = true;
}
if (!created_edge) {
	query_graph.CreateEdge(op.left_total_set, op.right_total_set, nullptr, op);
	// ...
}
```

`op.left_total_set` and `op.right_total_set` are not single relations — they are
`JoinRelationSet`s, i.e., whole *sets* of base relations sitting under that
operator's left and right children. An edge here connects two node-*sets* to each
other, not two single nodes. That's the concrete, source-grounded answer to "why a
hypergraph and not a plain graph": in DuckDB's construction, an edge's two
endpoints are `JoinRelationSet`s that can each contain more than one base
relation, and `QueryGraphEdges::CreateEdge` (`query_graph.cpp:61`) is written
generically over `JoinRelationSet` on both sides for exactly this reason — a
"neighbor" in this graph is always a set, even when that set happens to have size
one. Being honest about the boundary of this claim: I did not find, anywhere in
this pass, a single *join predicate expression* that itself references three or
more base relations at once (DuckDB's binder always produces binary comparisons);
the multi-relation endpoints arise specifically from non-reorderable operator
boundaries (outer joins and similar), where an entire already-planned subtree has
to move as one unit. If the DPhyp literature's formal hypergraph model has a
broader justification than this — and the paper cited in §8.5 does generalize the
node/edge relationship abstractly — this chapter's claim is limited to what the
DuckDB source actually does with it.

One more piece of this construction worth knowing before §8.5: equi-join
predicates are also folded into **equality classes** via a small union-find
(`QueryGraphManager::BuildPredicateModel`, `query_graph_manager.cpp:27`). The
union-find lives in its own small class, `JoinEqualityGroups`
(`src/include/duckdb/optimizer/join_order/join_predicate.hpp:96`), so that a
second user can share it (see the LEFT JOIN note in §8.6). If a
query said `a.x = b.x AND b.x = c.x`, the predicate model recognizes `a.x`, `b.x`,
and `c.x` as one transitive equality class rather than two independent edges, which
matters for cardinality estimation (§8.6) on star- and snowflake-shaped join
graphs. Query C's two predicates don't share a column, so they stay two separate
equality classes / two separate edges — the simple chain pictured above is exactly
what gets built.

## 8.4 Relation sets: a sorted array and a bitmask

The DPhyp literature (and most textbook write-ups of it) represents a "set of
relations already joined" as a bitmask — one bit per base relation — because
bitmask union, subset, and intersection tests are single machine instructions,
and because that representation is what makes a dynamic-programming table over
*subsets* of relations tractable at all. It would be easy to assume DuckDB's
`JoinRelationSet` is exactly that. Reading `join_relation_set.hpp` and
`join_relation_set.cpp` directly turns up something more specific:

```cpp
// src/include/duckdb/optimizer/join_order/join_relation_set.hpp:19-34
struct JoinRelationSet {
public:
	JoinRelationSet(unsafe_unique_array<RelationIndex> relations, idx_t count);
public:
	string ToString() const;
	bool Empty() const;
	static bool IsSubset(const JoinRelationSet &super, const JoinRelationSet &sub);
	static bool Intersects(const JoinRelationSet &left, const JoinRelationSet &right);
public:
	unsafe_unique_array<RelationIndex> relations;
	idx_t count;
	//! Bitmask over the relation indexes for fast subset/intersection checks
	vector<idx_t> bitmask;
};
```

A `JoinRelationSet` holds the same set in two forms. The first form is a
**sorted, deduplicated array of `RelationIndex`**. The second form is a
**bitmask**: one bit per relation, stored in a `vector` of 64-bit words, so it
has no fixed size limit. The constructor fills in the bitmask from the array:

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

For example, the set `{customer, orders}` from Query C is `{0, 1}`. Its array is
`[0, 1]` and its bitmask is one word with value `0b011` (decimal 3). `IsSubset`
and `Intersects` (`join_relation_set.cpp:41-62`) work word by word on these
bitmasks:

```cpp
// src/optimizer/join_order/join_relation_set.cpp:41-52
bool JoinRelationSet::IsSubset(const JoinRelationSet &super, const JoinRelationSet &sub) {
	D_ASSERT(sub.count > 0);
	if (sub.count > super.count || sub.bitmask.size() > super.bitmask.size()) {
		return false;
	}
	for (idx_t i = 0; i < sub.bitmask.size(); i++) {
		if (sub.bitmask[i] & ~super.bitmask[i]) {
			return false;
		}
	}
	return true;
}
```

`sub` is a subset of `super` when no bit of `sub` is missing from `super`. That is
the test `sub & ~super == 0`, done once per 64-bit word. For any query with 64 or
fewer relations, this is a single AND. The sorted array is still needed: it is
used to print the set, to walk its members, and as the key for the trie described
next.

The bitmask makes each set test fast. A second trick makes the DP table fast:
`JoinRelationSetManager` interns every set it ever constructs behind a trie
(`JoinRelationTreeNode`, keyed step-by-step on each `RelationIndex` in sorted
order — `join_relation_set.cpp:64-84`), so that **any two places in the algorithm
that arrive at the same set of relations, by whatever path, get back the literal
same `JoinRelationSet*`**. That's what makes the DP table
(`reference_map_t<JoinRelationSet, unique_ptr<DPJoinNode>> plans` in
`plan_enumerator.cpp`) cheap to query: it hashes and compares by pointer identity,
not by walking the array, so "have we already found a plan for exactly this
combination of relations?" is still an `O(1)` lookup — the sharing of subproblems
that makes dynamic programming dynamic programming is preserved. The trie itself
is still keyed on the sorted array, not on the bitmask. The header still carries a
`FIXME` about this: `// FIXME: this structure is inefficient, could use a bitmap
for lookup instead (todo: profile)` — `join_relation_set.hpp:41`. So the bitmask
speeds up the subset and overlap tests, but the lookup "give me the one shared
object for this set" still walks the trie one relation at a time.

A second, separate 64-bit bitmask exists in `PlanEnumerator`'s cache-key helper
for neighbor sets. Unlike the `JoinRelationSet` bitmask above, this one is a
single word, so it has a hard size limit.

```cpp
// src/optimizer/join_order/plan_enumerator.cpp:98-105
static idx_t GetNeighborSetCacheKey(const vector<RelationIndex> &neighbors) {
	idx_t key = 0;
	for (auto &neighbor : neighbors) {
		D_ASSERT(neighbor.index < sizeof(idx_t) * 8);
		key |= idx_t(1) << neighbor.index;
	}
	return key;
}
```

`idx_t` is `uint64_t` (`src/include/duckdb/common/typedefs.hpp:16`), so
`sizeof(idx_t) * 8` is 64. So this one cache assumes that relation numbers stay
below 64. It is not a limit on join ordering as a whole: only the exact search
(`EnumerateCSGRecursive` and `EnumerateCmpRecursive`) uses this cache, and the
`JoinRelationSet` bitmask above has no such limit. Two more caveats are worth
being precise about. First, it's a `D_ASSERT`, which compiles to nothing in a release
build; a 65th relation wouldn't be rejected cleanly, it would silently shift a
`uint64_t` by an out-of-range amount, which is undefined behavior in C++. Second,
and more importantly, **you will not get anywhere near 64 relations before a much
smaller, deliberate, user-visible limit kicks in first**: `PlanEnumerator::SolveJoinOrder`
checks the relation count against a setting before it even tries the exact search:

```cpp
// src/optimizer/join_order/plan_enumerator.cpp:602-610
bool PlanEnumerator::SolveJoinOrder() {
	bool force_no_cross_product = Settings::Get<DebugForceNoCrossProductSetting>(query_graph_manager.context);
	auto swap_to_approximate_threshold =
	    Settings::Get<ApproximateJoinOrderThresholdSetting>(query_graph_manager.context);

	// first try to solve the join order exactly
	bool solved;
	if (query_graph_manager.relation_manager.NumRelations() >= swap_to_approximate_threshold) {
		solved = SolveJoinOrderApproximately();
```

`ApproximateJoinOrderThresholdSetting`'s default is `12`
(`src/include/duckdb/main/settings.hpp:308-317`, documented as "the minimum number
of tables in a join to determine the optimal join order approximately instead of
exactly"). So the practically-binding cap on "how big a join can be before DuckDB
stops trying to find the exact optimum" is **12 tables, by default** — a setting
any user can raise with `SET approximate_join_order_threshold = N`, and it would
take deliberately cranking that setting up past 64 to ever put pressure on the
`D_ASSERT` above. The bitmask limit is real, but it is not the limit that matters
in practice; the threshold setting is.

## 8.5 DPhyp: dynamic programming over connected subgraphs

The exact search — when it's attempted at all — is a direct implementation of a
named published algorithm, and the source says so in its own words:

```cpp
// src/optimizer/join_order/plan_enumerator.cpp:599-601
// the plan enumeration is a straight implementation of the paper "Dynamic Programming Strikes Back" by Guido
// Moerkotte and Thomas Neumannn, see that paper for additional info/documentation bonus slides:
// https://db.in.tum.de/teaching/ws1415/queryopt/chapter3.pdf?lang=de
```

The paper is Moerkotte and Neumann [20]. The slides the
comment links to are TUM lecture notes [23].

**What "strikes back" refers to.** The classical approach to DP-based join
ordering (Selinger et al.'s System R optimizer [24]) builds its DP table by
enumerating every subset of the `n` relations, bottom-up, and for each subset
trying every way to split it into two already-solved smaller subsets. That works,
but it scales with the number of *subsets* (`2ⁿ`), and — crucially for this
algorithm's story — it does that enumeration blindly: most of those `2ⁿ` subsets,
for any join graph that isn't a dense clique, are *disconnected* (they'd require a
cross product to actually join), and the classical approach still has to visit
them to rule them out. By the mid-2000s this had led to a fairly settled belief
that exhaustive DP was impractical once join predicates got complicated — graphs
with cycles, star schemas, outer-join ordering constraints — and that queries
past a small table count needed a switch to heuristic/greedy search instead.
Moerkotte and Neumann's contribution [20] was to show that DP
didn't need to be abandoned for these cases: reformulate the search as directly
enumerating **connected subgraphs of the query's join hypergraph** — never
touching a disconnected subset at all — and the DP table can be built with a
number of table entries proportional to the number of actually-connected
subgraphs, which for realistic (sparse) join graphs is dramatically smaller than
`2ⁿ`. "Dynamic programming strikes back" against the idea that DP itself was the
part that didn't scale — it was the *enumeration strategy* that didn't scale, and
a graph-traversal reformulation fixes it.

**The mechanics, in DuckDB's own function names — which are the paper's own
terminology.** `plan_enumerator.hpp`/`.cpp` implements the paper's `EmitCsg` /
`EnumerateCsgRec` / `EnumerateCmpRec` triplet nearly verbatim (`Csg` = *connected
subgraph*, `Cmp` = its *complement* — the two halves of a candidate join). The
outer driver is `SolveJoinOrderExactly`:

```cpp
// src/optimizer/join_order/plan_enumerator.cpp:385-406
bool PlanEnumerator::SolveJoinOrderExactly() {
	// now we perform the actual dynamic programming to compute the final result
	// we enumerate over all the possible pairs in the neighborhood
	for (idx_t i = query_graph_manager.relation_manager.NumRelations(); i > 0; i--) {
		// for every node in the set, we consider it as the start node once
		auto &start_node = query_graph_manager.set_manager.GetJoinRelation(RelationIndex(i - 1));
		// emit the start node
		if (!EmitCSG(start_node)) {
			return false;
		}
		// initialize the set of exclusion_set as all the nodes with a number below this
		unordered_set<RelationIndex> exclusion_set;
		for (idx_t j = 0; j < i; j++) {
			exclusion_set.emplace(j);
		}
		// then we recursively search for neighbors that do not belong to the banned entries
		if (!EnumerateCSGRecursive(start_node, exclusion_set)) {
			return false;
		}
	}
	return true;
}
```

Each single relation gets a turn as a "seed," from the highest `RelationIndex`
down to the lowest. Two things are happening for every seed:

- `EmitCSG(node)` looks at `node`'s neighbors in the query graph — but *excluding*
  every relation whose index is numerically below `node`'s smallest member. This
  exclusion set is the trick that guarantees every connected subgraph pair gets
  discovered **exactly once** rather than once from each direction: relation 0's
  turn as seed sees the whole graph, relation 1's turn only sees relations ≥ 1,
  and so on, so a pair already reachable from a lower-numbered seed is never
  re-derived from a higher one.
- For each qualifying neighbor, if there's a direct edge, the pair gets handed to
  `TryEmitPair` immediately; either way, `EnumerateCmpRecursive` is called to keep
  growing the *complement* side (the "other" relation-set) bigger — union it with
  more of its own neighbors — checking at each step whether that grown complement
  is already a previously-solved connected subgraph (i.e., already has an entry in
  the `plans` DP table) before proposing it as a join partner.
- `EnumerateCSGRecursive` does the same growth on the **seed's own side**: keep
  extending `node` itself into bigger connected subgraphs (again only using
  not-yet-excluded neighbors), and every time a bigger subgraph is confirmed to
  already be solvable, hand it right back to `EmitCSG` — which is why these three
  functions are mutually recursive.

The load-bearing consequence of all this — the actual pruning that avoids
enumerating cross products — is that **`GetNeighbors` is always called with an
exclusion set, and only relations reachable through an actual query-graph edge are
ever returned** (`query_graph.cpp:118-131`). A pair of relation-sets is never even
considered as a join candidate unless the hypergraph says an edge connects them.
`TryEmitPair` and `EmitPair` do the actual DP-table maintenance once a candidate
pair has cleared that bar:

```cpp
// src/optimizer/join_order/plan_enumerator.cpp:213-217
if (entry == plans.end() || new_cost < old_cost) {
	// the new plan costs less than the old plan. Update our DP table.
	plans[new_set] = std::move(new_plan);
	return plans[new_set].get();
}
```

```cpp
// src/optimizer/join_order/plan_enumerator.cpp:243-248
if (pairs >= 10000) {
	// when the amount of pairs gets too large we exit the dynamic programming and resort to a greedy algorithm
	// FIXME: simple heuristic currently
	// at 10K pairs stop searching exactly and switch to heuristic
	return false;
}
```

That second excerpt is the *other* escape hatch, distinct from the 12-table
threshold in §8.4: even for a query under the table-count threshold, if the
connected-subgraph search still ends up emitting 10,000 candidate pairs (which can
happen for a densely-connected, non-chain join graph — a star schema with many
fact/dimension predicates, say), the exact search is abandoned mid-flight in favor
of the greedy heuristic in §8.8. Both escape hatches exist because DPhyp's
worst-case behavior, while far better than blind subset enumeration, is still
driven by how many connected subgraphs the *specific* join graph has — a query
with the same table count but a denser predicate graph can be much more expensive
to solve exactly than a simple chain.

## 8.6 Cost and cardinality: what actually feeds the DP table

Every time `EmitPair`/`CreateJoinTree` accepts a candidate join of `left` and
`right` into a combined `set`, it needs a number to compare against whatever plan
is already sitting in the DP table for that same `set`. That number comes from
`CostModel::ComputeCost`:

```cpp
// src/optimizer/join_order/cost_model.cpp:37-48
// Currently cost of a join mostly factors in the cardinalities.
// LEFT joins need an explicit RHS input component because their output cardinality preserves the LHS,
// which otherwise makes early LEFT joins over large RHS inputs look almost free.
double CostModel::ComputeCost(DPJoinNode &left, DPJoinNode &right, JoinRelationSet &combination,
                              const vector<reference<NeighborInfo>> &possible_connections) {
	auto join_card = cardinality_estimator.EstimateCardinalityWithSet<double>(combination);
	auto join_cost = join_card;
	if (query_graph_manager.GetPredicateModel().HasLeftJoinPredicates()) {
		join_cost += GetLeftJoinInputCost(cardinality_estimator, possible_connections);
	}
	return join_cost + left.cost + right.cost;
}
```

`cost(S ⋈ T) = |S ⋈ T| + cost(S) + cost(T)`, with a leaf's cost initialized to `0`
in `InitLeafPlans` (`plan_enumerator.cpp:590-591`). Unrolled by induction down to
the leaves, this means the cost of a whole join tree is the **sum, over every
internal node in the tree, of that node's own estimated output cardinality** — the
classic "sum of intermediate result sizes" cost metric (often called `C_out` in
the literature). This is the same family of idea as the simpler, more commonly
taught formula `cost(S ⋈ T) = |S| + |T| + |S ⋈ T|` (charge for reading both
inputs plus producing the output) — DuckDB's version charges only for the output
of every join actually performed, recursively, rather than separately re-charging
for reading each input at every level, but both formulas are ways of saying "favor
trees whose intermediate results stay small," and the ranking they induce over
candidate plans tends to agree in practice. The LEFT-join adjustment on top of
that base formula is a real, source-confirmed refinement, not a simplification:
because a `LEFT JOIN`'s output cardinality is bounded below by its LHS regardless
of how large the RHS is, the naive formula would make joining a huge table on the
RHS of an early `LEFT JOIN` look almost free (the *output* stays small even though
a large table had to be scanned) — `GetLeftJoinInputCost` explicitly adds back in
the cardinality of every distinct RHS relation set involved in a `LEFT JOIN`
predicate, so that cost isn't invisible to the search.

**Where the `|S|`/`|T|` cardinalities themselves come from** is worth tracing
carefully, because it's easy to assume — reasonably, from the pipeline order
described in Chapter 1's preview — that they come from the general
`STATISTICS_PROPAGATION` pass. Reading `optimizer.cpp`'s actual pass order shows
`JOIN_ORDER` at line 364 and `STATISTICS_PROPAGATION` at line 476 — the
statistics-propagation pass runs **after** join ordering, not before it. This
chapter's own investigation surfaces a real refinement to make here: the
cardinalities the join-order DP consumes are self-contained within
`src/optimizer/join_order/` and `src/optimizer/relation_statistics/`, and don't
depend on that later pass at all. `RelationManager::AddRelation` sets each base
relation's `estimated_cardinality` directly from a `RelationStats` object
(`relation_manager.cpp:99-100`), and for a `LogicalGet` those stats come from
`RelationStatisticsHelper::ExtractGetStats`:

```cpp
// src/optimizer/relation_statistics/relation_statistics_scan.cpp:134-168 (elided)
RelationStats RelationStatisticsHelper::ExtractGetStats(LogicalGet &get, ClientContext &context) {
	RelationStats result;
	auto base_table_cardinality = get.EstimateSourceCardinality(context);
	auto cardinality_after_filters = base_table_cardinality;
	// ...
	if (get.table_filters.HasFilters()) {
		bool has_non_optional_filters = false;
		for (auto &entry : get.table_filters) {
			auto &column_index = get.GetColumnIndex(entry.GetIndex());
			auto column_statistics = GetColumnStatistics(get, context, column_index);
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
			// ...
		}
		if (cardinality_after_filters == base_table_cardinality && has_non_optional_filters) {
			cardinality_after_filters =
			    MaxValue<idx_t>(LossyNumericCast<idx_t>(double(base_table_cardinality) * DEFAULT_SELECTIVITY), 1);
		}
		if (base_table_cardinality == 0) {
			cardinality_after_filters = 0;
		}
	}
	result.cardinality = cardinality_after_filters;
	// ...
}
```

Two details in this excerpt are worth a closer look:

- The base row count comes from `get.EstimateSourceCardinality`, not
  `get.EstimateCardinality`. The second function returns a cached estimate if the
  `LogicalGet` already has one (`src/planner/operator/logical_get.cpp:303-309`).
  That cached value may already include the table filters. If the stats were
  extracted twice, the filters were applied twice. The "source" version always
  asks the table function for the raw row count (`logical_get.cpp:311-324`).
- A filter that the column statistics prove is **always true** is skipped. It
  does not shrink the estimate, and it does not trigger the 20% fallback. For
  example, `id IS NOT NULL` on a column whose statistics say it has no NULLs keeps
  all rows. `test/optimizer/joins/always_true_scan_filter_estimate.test` checks
  this on a 1,000-row table: `WHERE id IS NOT NULL` is estimated at `1000` rows,
  but `WHERE nullable_id IS NOT NULL` (a column that has one NULL) is still
  estimated at `200`.

`get.table_filters` is already populated by the time this runs, because
`FILTER_PUSHDOWN` (pass #6, `optimizer.cpp:286`) runs well before `JOIN_ORDER`
(pass #17) and has already pushed `c.nation = 'FR'` down onto `customer`'s
`LogicalGet` as a table filter. `ExtractGetStats` looks up column statistics for
`nation` (an HLL-based distinct count if the catalog has one) and calls
`InspectTableFilter` to shrink the base cardinality accordingly; if no usable
per-column statistic exists for a non-optional filter, it falls back to a flat
`DEFAULT_SELECTIVITY = 0.2` (`relation_statistics/relation_statistics_helper.hpp:32`) — assume any
otherwise-unmeasurable filter keeps 20% of rows. Either way, the number
`InitLeafPlans` hands to the DP table for `customer` already reflects the
`nation = 'FR'` filter, entirely independent of the later, general-purpose
`STATISTICS_PROPAGATION` pass (whose job, per its own module — cited in Chapter 7
— is closer to correctness-oriented min/max propagation and constant folding than
to feeding join-order cardinalities). This is a real, source-checked correction to
the more casual "statistics propagation feeds join order" framing a quick read of
the pipeline order might suggest; the front matter's own policy is that the
source wins when material like this disagrees, so it's called out here rather
than smoothed over.

### 8.6.1 Two special cases in the join estimator

The join estimator divides the product of the input sizes by a **denominator**.
For an equality join, the denominator grows with `tdom`, the estimated number of
distinct join-key values (the "total domain"). The function that sets this
factor per comparison type treats `<>` and `IS DISTINCT FROM` on their own:

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

A `<>` join keeps every pair of rows except the pairs with equal keys. If there
are `tdom` distinct values, about `1/tdom` of the pairs are equal, so about
`1 − 1/tdom` of them survive. Dividing by `tdom / (tdom − 1)` gives exactly that
fraction. If `<>` used the range-join factor `tdom^(2/3)` instead, `<>` joins
would look orders of magnitude smaller than they are, and a too-small estimate
could lead to a nested loop join or the wrong build side.
`test/optimizer/joins/not_equal_join_cardinality_estimation.test` shows it with
two 1,000-row tables whose key has 100 distinct values:

```sql
-- t1.a and t2.b each hold the values 0..99, ten times each
SELECT * FROM t1 JOIN t2 ON t1.a <> t2.b;
-- estimated: a six-digit number starting with 9 (the test matches "9[0-9]{5}")
-- actual count(*): 990000
SELECT * FROM t1 JOIN t2 ON t1.a < t2.b;
-- estimated: a five-digit number, since range joins keep the tdom^(2/3) penalty
```

The math: `1,000 × 1,000 / (100 / 99) = 990,000`, which is the true answer.

The second change is for **LEFT joins on composite keys**. Suppose a LEFT JOIN's
`ON` clause has two equalities, `a = x AND b = y`, between the same two tables.
Neither column alone is unique, but the pair `(a, b)` may be.
`QueryGraphManager::BuildLeftEqualityPairs` (`query_graph_manager.cpp:108`) finds
such pairs. It groups the equalities of each LEFT join with the same
`JoinEqualityGroups` union-find used for inner joins (§8.3) and marks the
predicate as a composite join pair. `CalculateLeftJoinDenomInfo` then uses a
larger "cap" as the key domain (`cardinality_estimator.cpp:491-499`).
`test/optimizer/joins/left_join_reordering/left_join_composite_domain.test`
builds a 2,048-row table and a 1,024-row table whose `(a, b)` and `(x, y)` pairs
have 1,024 distinct values. The test expects the hash join to be estimated at
`~2,048` rows, which is the real result size.

## 8.7 Walking Query C through the DP table

With the hypergraph from §8.3 and the leaf cardinalities from §8.6, here is what
`InitLeafPlans` puts in the DP table before any joining happens. Suppose
`customer` has 150,000 base rows and `nation` has roughly 25 distinct values
(TPC-H's actual scale), `orders` has 1,500,000 rows, and `lineitem` has
6,000,000 rows:

```
DP table after InitLeafPlans:

  {customer}  cost=0   cardinality ≈ 150,000 / 25 ≈ 6,000   (after nation='FR')
  {orders}    cost=0   cardinality = 1,500,000
  {lineitem}  cost=0   cardinality = 6,000,000
```

`SolveJoinOrderExactly` seeds from the highest-indexed relation down. With
`customer=0, orders=1, lineitem=2`, the seed order is `lineitem`, then `orders`,
then `customer`. Each seed's `EmitCSG` looks at its graph neighbors under the
exclusion set built from lower indices:

```
seed = lineitem (2):  neighbors under exclusion{}          → orders (edge exists)
                       EmitPair({lineitem}, {orders})       → candidate join A

seed = orders (1):    neighbors under exclusion{2}          → customer (edge exists)
                       EmitPair({orders}, {customer})       → candidate join B
                       EnumerateCSGRecursive grows {orders} using lineitem (excluded
                       here since seed=1 bans nothing ≥1, lineitem=2 is still a
                       valid neighbor) → considers {orders,lineitem} as a left-hand
                       side, looks for its neighbors (customer) → candidate join C:
                       ({orders,lineitem}, {customer})

seed = customer (0):  neighbors under exclusion{}            → orders (edge exists);
                       lineitem is NOT a direct neighbor of customer (no predicate),
                       so it is only reached via the {orders,lineitem} set already
                       built above.
```

Only two 2-relation sets are ever proposed as join candidates — `{orders,
lineitem}` and `{customer, orders}` — because those are the only pairs the
hypergraph actually connects; `{customer, lineitem}` is never even built as a
join candidate, because `GetNeighbors` never returns it (no edge). This is
DPhyp's pruning in action: a naive subset-based DP would have to construct and
cost all three pairs to find this out, and would additionally have to consider a
`{customer, lineitem}` cross product explicitly *not worth it* rather than never
generating it.

```cpp
// src/optimizer/join_order/cost_model.cpp — cost(S⋈T) = |S⋈T| + cost(S) + cost(T)
```

Costing the two viable 2-relation joins, using the cardinality-estimation
machinery from §8.6 (illustrative numbers, using a simple FK/PK-style
containment assumption for the estimator's total-domain denominator — see the
comment at `cardinality_estimator.cpp:899-905` citing Ebergen's MSc thesis on
almost-statistics-free cardinality estimation [25] for the
actual, more careful formula DuckDB uses here):

```
{orders, lineitem}:
  join_card ≈ 6,000,000            (orders ⋈ lineitem on orderkey is close to 1:1
                                     per lineitem row — lineitem is the larger,
                                     "many" side of a 1:many FK relationship)
  cost = join_card + cost(orders) + cost(lineitem) = 6,000,000 + 0 + 0 = 6,000,000

{customer, orders}:
  join_card ≈ 6,000 × (1,500,000 / 150,000) ≈ 60,000
                                    (customer already filtered to ~6,000 rows by
                                     nation='FR'; each customer has ~10 orders)
  cost = join_card + cost(customer) + cost(orders) = 60,000 + 0 + 0 = 60,000
```

`{customer, orders}` is roughly two orders of magnitude cheaper than `{orders,
lineitem}` as a first join, purely because the `nation = 'FR'` filter already
shrank `customer` before either candidate was costed — this is exactly why §8.6's
point about *when* cardinalities get filtered matters concretely, not just as
trivia. `EmitPair` records both candidates in the DP table regardless (a smaller
subproblem can still be reused inside a larger plan even if it isn't part of the
eventual optimum on its own), and the search continues by growing these
2-relation sets toward the full 3-relation set:

```
{orders, lineitem} + customer → {customer, orders, lineitem}
    cost = |full join| + cost({orders,lineitem}) + cost({customer})
         = 60,000(≈)   + 6,000,000               + 0
         ≈ 6,060,000

{customer, orders} + lineitem → {customer, orders, lineitem}
    cost = |full join| + cost({customer,orders}) + cost({lineitem})
         = 60,000(≈)   + 60,000                  + 0
         ≈ 120,000
```

(The final join's own output cardinality, `|full join|`, is roughly the same
either way — it's determined by the data, not by which order you multiplied the
intermediate joins in — so what actually separates these two candidate trees is
almost entirely the `cost(...)` of whichever 2-relation subplan they build on top
of.) `EmitPair` compares the two candidate costs for the set `{customer, orders,
lineitem}` and keeps the cheaper one, exactly the same "new plan costs less than
the old plan. Update our DP table." logic quoted in §8.5. The winning tree —
filter `customer` down to France first, join that small result against `orders`,
then bring in `lineitem` last — is precisely the plan a human optimizing this
query by hand would pick, and precisely the one DPhyp's exhaustive-but-pruned
search finds without ever having had to consider a `customer ⋈ lineitem` cross
product:

```
              LogicalComparisonJoin (o.orderkey = l.orderkey)
                    │
        ┌───────────┴────────────┐
        │                        │
  LogicalComparisonJoin       LogicalGet (lineitem)
  (c.id = o.custkey)
        │
   ┌────┴─────┐
   │          │
LogicalGet   LogicalGet
(customer,   (orders)
 filter:
 nation='FR')
```

`QueryGraphManager::GenerateJoins` (`query_graph_manager.cpp:643`) turns this
winning `DPJoinNode` tree back into real `LogicalComparisonJoin` and
`LogicalCrossProduct` nodes. `QueryGraphManager::Reconstruct`
(`query_graph_manager.cpp:502`) then splices the new tree back into the plan.
(Despite its name, `join_order_operator.cpp` does not build joins. It holds
`JoinOrderConflictDetector`, which decides which orders are legal for
non-reorderable joins such as LEFT, SEMI and ANTI.)

### 8.7.1 Putting the tree back: what `Reconstruct` and relation extraction preserve

Several rules around the reordered tree make sure that a query's result is the
same as it would be without reordering. Most of them have a regression test that
guards against an earlier bug.

**The parent keeps its column layout.** The new join tree can output its columns
in a different order than the old tree. For example, the old tree may have
output `(a.i, a.b, b.i, b.b)` and the new one `(b.i, b.b, a.i, a.b)`. The
operator just above the tree may pick columns by *position* (a `projection_map`
on a `LogicalFilter`, for example). Such a parent would pick the wrong columns.
So `Reconstruct` records the old child's column bindings first, and rewrites the
parent to match the new order after the swap:

```cpp
// src/optimizer/join_order/query_graph_manager.cpp:564-567
	// have to replace at this node
	(*parent)->children[0] = std::move(join_tree.op);
	// the new join tree can emit its columns in a different order: preserve the parent's output layout
	ColumnBindingRewrite::ApplyToChild(*parent, 0, std::move(old_child_bindings), BindingReplacementGraph());
```

`test/optimizer/joins/filter_projection_map_reconstruction.test` has the query
from a bug report (GitHub issue 28157). It guards against a bug where the query
failed with `INTERNAL Error: Failed to bind column reference "i"`.

**Stacked filters keep their order.** When several `LogicalFilter` nodes sit on
top of each other, the relation manager collects them top-down. Re-applying them
in that order would put the lowest filter (the one that ran first) last. So each
collected chain is reversed:

```cpp
// src/optimizer/join_order/relation_manager.cpp:434-435
	// stacked filters are re-applied in this order: keep the lower (earlier evaluated) filters first
	std::reverse(filter_operators.begin() + NumericCast<int64_t>(filter_chain_start), filter_operators.end());
```

The order matters when an earlier filter protects a later one from an error.
`test/optimizer/expression_rewriter/functions_that_error_are_not_reordered.test`
has this case:

```sql
SELECT * FROM (VALUES (0)) x(a), (VALUES (2)) y(b)
WHERE b < 1 AND get_bit('1'::BIT, b) = 1;
-- empty result: b < 1 must run first, or get_bit throws for b = 2
```

**Volatile filters are a boundary.** A *volatile* expression can give a different
result each time it runs, like `random()` or `nextval('seq')`. A filter that
contains one must run on exactly the rows it would run on without reordering. So
`OperatorIsNonReorderable` treats such a filter like a set operation: the
join order optimizer does not move joins across it.

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

`test/optimizer/joins/zero_column_filter_reconstruction.test` checks that a
`nextval` filter on one join input runs once per input row, before the join
duplicates that row:

```sql
CREATE SEQUENCE leaf_filter_calls;
SELECT count(*)
FROM (SELECT * FROM range(1) t(i) WHERE nextval('leaf_filter_calls') = 1) a
JOIN (VALUES (0), (0)) b(i) USING (i);
-- 2
SELECT currval('leaf_filter_calls');
-- 1   (the filter ran once, not once per joined row)
```

**Filters with no column references are kept.** A filter like `WHERE NULL` or
`WHERE random() > 2` mentions no column, so it belongs to no relation. If such a
filter stayed on the intermediate operator where it was found, it could be lost,
because reconstruction replaces those intermediate operators. So such a filter
gets an empty relation set, and reconstruction attaches it again. The helper
`RelationManager::ExtractFilterBindings`
(`relation_manager.cpp:901-908`) decides which filters may move: a filter with
no bindings may move only if it truly has no column references. The
same test file checks queries such as
`SELECT (SELECT 1) AS a0, (SELECT 2) AS a1 FROM t0 WHERE NULL;`, which must
return no rows.

**Secure views are one relation.** A `LogicalSecureView` (the operator that
wraps the plan of a `CREATE SECURE VIEW`) is an optimization barrier. The relation
manager treats it as a single relation, like an aggregate: it optimizes the
view's inside on its own with a child optimizer, and only the view's own
cardinality estimate crosses the boundary. Column statistics cross only if the
view's `propagate_statistics` flag allows it (`relation_manager.cpp:520-550`).
`test/sql/optimizer/secure_view_join_order.test` joins a fact table behind a
secure view with three dimension tables. The test checks that the reordered plan
has no `CROSS_PRODUCT` and still has its `SECURE_VIEW` node.

**The recursion limit is a constant.** The join order optimizer calls itself
once for each nested non-reorderable operator (a projection, an aggregate, a set
operation, and so on). A deep chain of `EXCEPT`/`INTERSECT` could overflow the
stack. So the depth is capped by a constant, not by a setting:

```cpp
// src/include/duckdb/optimizer/join_order/join_order_optimizer.hpp:24-28
	//! Deepest nesting of child optimizers (one per non-reorderable operator, projection, aggregate, ...) that is still
	//! join-order optimized. Every level recurses through Optimize/Build/ExtractJoinRelations, so a very deep plan
	//! (e.g. a long chain of EXCEPT/INTERSECT) would otherwise overflow the stack long before the plan depth limit is
	//! hit; subplans below this depth keep their original join order, which is always a valid plan.
	static constexpr idx_t MAX_RECURSION_DEPTH = 128;
```

## 8.8 When exact DP gives up: the greedy fallback

Three conditions send the search to `SolveJoinOrderApproximately` instead of
finishing with `SolveJoinOrderExactly` (§8.4, §8.5):

1. the relation count meets `approximate_join_order_threshold` (default 12), so
   the exact search is never tried;
2. the exact search emits 10,000 candidate pairs (`pairs >= 10000` in
   `TryEmitPair`) before it finishes;
3. the exact search finishes but finds no plan that covers all relations, even
   after it adds the cross products the query needs
   (`plan_enumerator.cpp:612-617`).

In every case the fallback is the same algorithm. The source names it
explicitly:

```cpp
// src/optimizer/join_order/plan_enumerator.cpp:408-427
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
	// at this point, we exited the dynamic programming but did not compute the final join order because it took too
	// long instead, we use a greedy heuristic to obtain a join ordering now we use Greedy Operator Ordering to
	// construct the result tree first we start out with all the base relations (the to-be-joined relations)
	vector<reference<JoinRelationSet>> join_relations; // T in the paper
	for (idx_t i = 0; i < query_graph_manager.relation_manager.NumRelations(); i++) {
		join_relations.push_back(query_graph_manager.set_manager.GetJoinRelation(RelationIndex(i)));
	}
```

**Greedy Operator Ordering (GOO)**: start with every base relation as its own
one-relation "set." At every step, look at every currently-remaining pair of
sets. Keep only pairs that have an actual query-graph edge connecting them (the
same `GetConnections`/hypergraph machinery as the exact search — the greedy
fallback does not consider cross products as a first resort). Cost each
connectable pair with the *exact same* `CostModel`/`EmitPair` used by DP. Then
merge the single cheapest pair into one combined set, and repeat. Each step
reduces the number of sets by one. This is far cheaper than the exact search's
possibly exponential work, but it is no longer guaranteed to be optimal: a
locally cheapest pair at step 1 can lock in a globally worse tree than a
different first choice would have.

Three details of the greedy loop matter.

**Each pair is costed once.** Calling `EmitPair` on every remaining pair in every
step would cost `O(r²)` costings per step, `O(r³)` in total for `r` relations.
But a pair's cost does not change while the query graph stays the same. So the
code costs each pair once and keeps the results in a list of
`GreedyCandidate`s:

```cpp
// src/optimizer/join_order/plan_enumerator.cpp:429-438
	// in every step of the algorithm, we greedily pick the join between the to-be-joined relations that has the
	// smallest cost. Pair costs are deterministic while the query graph is unchanged, so we evaluate each pair once
	// and keep the viable candidates in a list, instead of re-emitting every pair in every step
	struct GreedyCandidate {
		optional_ptr<JoinRelationSet> left;
		optional_ptr<JoinRelationSet> right;
		double cost;
	};
	vector<GreedyCandidate> candidates;
```

After a merge of `best_left` and `best_right` into `new_set`, the loop drops every
candidate that used one of the two merged sets. It then costs only the new pairs
`(relation, new_set)` for each remaining relation (`plan_enumerator.cpp:517-538`).
So each step costs `O(r)` new pairs instead of `O(r²)`. The list is rebuilt from
scratch only when `ActivateRequiredCrossProducts` adds edges to the graph,
because new edges can make new pairs valid.

**Each pair is tried in both orientations.** LEFT, SEMI and ANTI joins are not
commutative: `A LEFT JOIN B` is not `B LEFT JOIN A`. `EmitPair` rejects a pair in
the wrong orientation. A merged set is put on the right first. If that
orientation fails, the loop tries the reverse:

```cpp
// src/optimizer/join_order/plan_enumerator.cpp:445-455
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
```

With only one orientation, a valid pair could be rejected. If no candidate was
left, the optimizer would keep the original tree. Commit `4e59c21ce83` describes
the real effect this had: the original tree estimated an `ANTI` join at its full
left input, and that bad estimate stopped the common-subplan optimizer from
sharing a repeated subquery. `test/optimizer/common_subplan_projection_map.test`
runs such a query with `SET approximate_join_order_threshold=2`, to force the greedy
path, and checks that the plan uses `CTE Scan` twice.

**The original join order is a candidate too.** The loop at the top of the
function (quoted above) walks the query's *original* join operators. For each
one whose two inputs already have plans in the DP table, it emits that join with
`EmitPair`. This puts parts of the plan the user wrote into the DP table. When the
greedy loop later builds the same set of relations, `EmitPair` keeps whichever
of the two plans is cheaper (the "Update our DP table" rule from §8.5). So a
greedy choice replaces an original subplan only if the cost model rates it
cheaper.

If, at some step, *no* remaining pair has any connecting edge at all (the join
graph has a genuinely disconnected component — no predicate relates the two
halves of the query), the fallback first tries `ActivateRequiredCrossProducts`.
If that adds nothing, it looks for the cheapest pair it is allowed to cross-join
(`CanCreateCrossProduct`, restricted to what the source calls "inner-join
companion sets"), synthesizes a cross-product edge in the query graph, and
continues. If even that fails, `SolveJoinOrderApproximately` returns `false`, and
`JoinOrderOptimizer::Optimize` falls back to the plan's original, pre-join-order
shape unchanged — always a valid (if unoptimized) plan.

## 8.9 Build side vs. probe side

Picking a join *order* only decides which two relation-sets get joined together
and in what sequence — it doesn't decide, for a given join, which of the two
inputs feeds which side of the physical join algorithm. That's a separate,
later pass, `OptimizerType::BUILD_SIDE_PROBE_SIDE`. It is pass #27 in Chapter
7's numbering, which leaves out `REMOTE_PUSHDOWN` because that pass runs from a
separate method (see Chapter 7 §7.4). Its call is at
`src/optimizer/optimizer.cpp:426`, and its code is in
`src/optimizer/build_probe_side_optimizer.cpp`), and by DuckDB's convention the
**right-hand child of a logical join is always the build side**. The heuristic
this pass uses to decide whether to flip a join's children is, at its core,
exactly the "smaller/more complex → build side" rule DiDi's slides describe [10],
grounded in an actual memory-footprint estimate:

```cpp
// src/optimizer/build_probe_side_optimizer.cpp:188-221 (elided)
double BuildProbeSideOptimizer::GetBuildSize(vector<LogicalType> types, const idx_t cardinality) {
	// Row width in the hash table
	types.push_back(LogicalType::HASH);
	auto tuple_layout = TupleDataLayout();
	tuple_layout.Initialize(types, TupleDataValidityType::CAN_HAVE_NULL_VALUES);
	auto row_width = tuple_layout.GetRowWidth();
	// ... penalties for VARCHAR/LIST/ARRAY columns and column count ...
	// There is also a cost of NextPowerOfTwo(count * 2) * sizeof(ht_entry_t) per tuple in the hash table
	// This is a not a smooth cost function, so instead we do the average, which is ~3 * sizeof(ht_entry_t)
	row_width += 3 * sizeof(ht_entry_t);

	return MaxValue<double>(static_cast<double>(row_width) * static_cast<double>(cardinality), sizeof(ht_entry_t));
}
```

The last line has a floor. The join-order estimator works in `double`, so an
estimate can be below one row (for example `0.3` rows after several selective
joins). Stored as the integer `cardinality`, that becomes `0`, which would give
a build size of `0`. The floor makes the result never smaller than the size of
one hash-table entry, `sizeof(ht_entry_t)`, so a build side is never "free". The
product is also computed in `double`, not as an integer product.

`GetBuildSize` is literally an estimate of *bytes of memory the build side would
occupy* — row width (with penalties for wide/variable-size/nested columns) times
estimated cardinality. `TryFlipJoinChildren` computes this for both children and
swaps them if the side currently on the right is more expensive to build than the
side on the left:

```cpp
// src/optimizer/build_probe_side_optimizer.cpp:288-293
// RHS is build side.
// if right_side metric is larger than left_side metric, then right_side is more costly to build on
// than the lhs. So we swap
if (right_side_build_cost > left_side_build_cost) {
	swap = true;
}
```

**Why this matters differently, but for a related reason, per join algorithm**
(all three of DuckDB's join implementations that have a build/probe distinction —
hash join, nested-loop join, and the sort-based range joins — declare themselves
sinks, meaning every one of them fully consumes its build/right child *before* a
single row of the probe/left child is processed):

- **Hash join** (`PhysicalHashJoin`): the build side is read entirely during the
  `Sink` phase and materialized into an in-memory hash table (keyed on the join
  columns); the probe side then streams through, chunk by chunk, looking up each
  row's key. The hash table's size is dominated by the build side's row count and
  width, so putting the smaller side there directly minimizes both the memory
  footprint and the one-time build cost paid before any output can start.
- **Nested-loop join** (`PhysicalNestedLoopJoin`): also declares
  `IsSink() = true` (`physical_nested_loop_join.hpp:66-68`) and fully buffers its
  right child's rows during `Sink` before the left child is scanned against that
  buffer in `ExecuteInternal`. Every left/probe row re-scans the *entire*
  buffered right/build side (this is what makes it a nested loop rather than a
  hash join), so a smaller build side isn't just cheaper to hold in memory — it
  directly shrinks the inner-loop work repeated once per probe row.
- **Piecewise merge join and IE-join** (`PhysicalPiecewiseMergeJoin`,
  `PhysicalIEJoin`, both `PhysicalRangeJoin` subclasses): both sides get sorted
  (each maintains its own `LocalSortedTable`/`GlobalSortedTable`,
  `physical_range_join.hpp`), and the resulting sorted runs are what get swept
  against each other looking for qualifying pairs. The build/right side's sorted
  run is what stays resident and gets re-walked (piecewise, hence the name, in
  blocks) for each block of the probe side, so the same "keep the smaller,
  more-expensive-to-hold structure on the side that's built once" reasoning
  applies again, on top of — not instead of — the sort cost each side pays
  independently of which side is "build."

A few extra refinements exist in the same file worth knowing about without going
deep on them here: a DML statement's row-id-carrying relation is preferentially
kept on the probe side (`GetRowidBindings`, `build_probe_side_optimizer.cpp:31`);
a join whose build side would enable a dynamic filter to be pushed down into a
much larger probe-side scan can be worth flipping even against the raw
memory-footprint comparison (`DynamicFilterBuildBonus`,
`JoinFilterBuildSideHeuristics` at `build_probe_side_optimizer.cpp:24-28`); and
equal-cost ties are broken in favor of keeping the build side "right-deep" when
the left child already has joins beneath it, so that in-flight tuples aren't
rebuilt (`PREFER_RIGHT_DEEP_PENALTY`, `build_probe_side_optimizer.cpp:279-286`).

## 8.10 Choosing the join algorithm itself

This last decision — which concrete `PhysicalOperator` implements a given
`LogicalComparisonJoin` — is not made by anything under `src/optimizer/join_order/`
at all. It happens one stage later, in the physical plan generator, specifically
in `PhysicalPlanGenerator::PlanComparisonJoin`
(`src/execution/physical_plan/plan_comparison_join.cpp:177`) — Chapter 9 covers
that file and that stage in full. But the *criteria* the decision is made on
belong conceptually to this chapter, because they're a direct continuation of the
join-order and build/probe-side reasoning above (predicate shape, and the
cardinalities the join-order optimizer already estimated), so they're worth
walking through here, with the forward reference to Chapter 9 for exactly where
in the pipeline this code executes.

```cpp
// src/execution/physical_plan/plan_comparison_join.cpp:215-288 (elided)
if (op.conditions.empty()) {
	// no conditions: insert a cross product
	return Make<PhysicalCrossProduct>(op.types, left, right, op.estimated_cardinality);
}

idx_t has_range = 0;
bool has_equality = op.HasEquality(has_range);
bool can_merge = has_range > 0;
bool can_iejoin = has_range >= 2 && recursive_cte_tables.empty();
// ... join_type-specific restrictions on can_merge / can_iejoin ...

if (has_equality && !prefer_range_joins) {
	// pass separately to PhysicalHashJoin
	auto &join = Make<PhysicalHashJoin>(op, left, right, /* ... */);
	return join;
}

idx_t nested_loop_join_threshold = Settings::Get<NestedLoopJoinThresholdSetting>(context);
if (left.estimated_cardinality < nested_loop_join_threshold ||
    right.estimated_cardinality < nested_loop_join_threshold) {
	can_iejoin = false;
	can_merge = false;
}
if (can_merge && can_iejoin) {
	idx_t merge_join_threshold = Settings::Get<MergeJoinThresholdSetting>(context);
	if (left.estimated_cardinality < merge_join_threshold || right.estimated_cardinality < merge_join_threshold) {
		can_iejoin = false;
	}
}

if (can_iejoin) {
	return Make<PhysicalIEJoin>(op, left, right, /* ... */);
}
if (can_merge) {
	// range join: use piecewise merge join
	return Make<PhysicalPiecewiseMergeJoin>(op, left, right, /* ... */);
}
if (PhysicalNestedLoopJoin::IsSupported(op.conditions, op.join_type)) {
	// inequality join: use nested loop
	return Make<PhysicalNestedLoopJoin>(op, left, right, /* ... */);
}
// ... else PhysicalBlockwiseNLJoin, the fully generic fallback ...
```

As a decision tree:

```
                         op.conditions empty?
                              │
                    yes ──────┼────── no
                     │                │
              CROSS_PRODUCT     any equality condition?
                                      │
                          yes ────────┼──────── no
                           │                     │
                       HASH_JOIN         ≥2 range/inequality
                     (the default,        conditions AND no
                      "workhorse" case)    recursive CTE?
                                                  │
                                      yes ────────┼──────── no
                                       │                     │
                              both sides above       exactly one range
                              merge/NL-join                condition?
                              cardinality                   │
                              thresholds?                   │
                                       │           yes ──────┼────── no
                             yes ──────┼────── no   │                │
                              │                │  PIECEWISE_MERGE   comparison(s)
                          IE_JOIN    PIECEWISE      _JOIN           supported by
                                     _MERGE_JOIN                    NESTED_LOOP_JOIN?
                                     (small-input                        │
                                      fallback: NL)              yes ────┼──── no
                                                                   │            │
                                                             NESTED_LOOP   BLOCKWISE_NL_JOIN
                                                                _JOIN      (fully generic)
```

`has_equality` covers the overwhelmingly common case — any `=` condition at all
routes straight to `PhysicalHashJoin`, DuckDB's workhorse join, regardless of what
else is in the predicate list. Only once there's *no* equality does predicate
shape start to matter: `has_range` counts inequality-style comparisons
(`<`, `<=`, `>`, `>=`), and `can_iejoin` requires **at least two** of them (plus no
recursive CTE involvement) — this is the case IE-join was purpose-built for: a
predicate like `a.x < b.x AND a.y > b.y`, two independent inequalities with no
equality anywhere. IE-join (from Khayyat et al.'s "Lightning fast and space
efficient inequality joins" [26], which DuckDB's `PhysicalIEJoin`/`PhysicalRangeJoin` machinery implements)
sorts both inputs by one of the two predicates and sweeps a second, permutation-
ordered pass over the other predicate to find qualifying pairs in close to
`O((n+m) log(n+m) + k)` rather than the `O(n·m)` a naive nested loop would need —
conceptually, sort-merge rather than hash-and-probe, because there's no equality
to hash on. `PhysicalPiecewiseMergeJoin` is the simpler sibling for a *single*
range condition: sort both sides, then sweep once. Both range-join operators are
gated by cardinality thresholds (`NestedLoopJoinThresholdSetting`,
`MergeJoinThresholdSetting`) precisely because sorting has its own fixed cost —
below some input size, a plain nested loop is cheaper than paying for a sort
whose result gets used only once, which is why small-enough inputs disable
`can_merge`/`can_iejoin` and fall through to `NESTED_LOOP_JOIN` instead. The final
fallback, `PhysicalBlockwiseNLJoin`, is what's left for predicate shapes none of
the specialized operators support (an arbitrary boolean expression that isn't
even a simple per-column comparison) — it evaluates the join condition as a
generic expression per candidate row pair, in blocks, and is the slowest of the
five by design; the whole decision tree above exists to avoid falling into it
whenever a faster, more specialized operator is applicable.

## 8.11 What to carry forward

Three ideas from this chapter recur in later chapters and are worth having
crisp: (1) DuckDB never enumerates disconnected relation-set pairs — the query's
join predicates are compiled into a hypergraph before any search starts, and
every candidate the DP search proposes is guaranteed connected by construction;
(2) the DP table's speed comes from two things: *interning* relation sets so
identical sets collapse to one pointer, and a multi-word
bitmask on each set for fast subset and overlap tests (there is no 64-relation
limit on relation sets; only one cache used by the exact search assumes
relation numbers below 64, and a much smaller, user-visible 12-table
threshold governs when the exact search is even attempted);
(3) join ordering, build/probe-side assignment, and physical join-algorithm
selection are three separate passes, running in that order, each consuming the
cardinality estimates and structural decisions the previous one produced —
Chapter 9 picks up immediately after the third of these, at the physical plan
generator that actually instantiates the `PhysicalHashJoin`/`PhysicalIEJoin`/etc.
nodes this chapter's decision tree chooses among.
