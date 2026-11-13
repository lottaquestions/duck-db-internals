# Chapter 7 — The Optimizer (deep dive): `EMPTY_RESULT_PULLUP` and `PROJECTION_PULLUP`

This file is one of the deep-dive parts fanning out from `00-overview.md`, which
already reproduced the full ~45-call-site pipeline in exact execution order
(§7.4) and gave each pass a one-line pointer. Two of those pointers are
expanded here in full: **`EMPTY_RESULT_PULLUP`** (pass #13,
`optimizer.cpp:339`, `src/optimizer/empty_result_pullup.cpp`) and
**`PROJECTION_PULLUP`** (pass #15, `optimizer.cpp:351`,
`src/optimizer/projection_pullup.cpp`). They sit two slots apart in the fixed
sequence, they're unrelated to each other mechanically, and they're grouped
into one file simply because both are small, self-contained "pull something
upward through the tree" passes that reward the same close a reading — every
claim below is checked against the actual source on this book's checkout
(`cb5673685153`) and, wherever the source's own git history clarifies *why* a
line is there, against that history too.

One format note before either pass starts: neither of this file's two primary
tests (`test/optimizer/projection_pullup.test`,
`test/fuzzer/duckfuzz/empty_result_pullup_delim_joins.test`) uses the
paired-`query I nosort <label>` idiom that proves a rewrite by matching two
labeled `EXPLAIN` blocks byte-for-byte. `projection_pullup.test` proves its
rewrites two different ways instead — an `EXPLAIN ... REGEX` pattern match
against the shape of the *optimized* logical plan (`PRAGMA explain_output =
OPTIMIZED_ONLY`), and, in its closing section, a `foreach do '' 'projection_pullup'`
loop that runs the same query twice, once fully optimized and once with this
one named pass switched off via `disabled_optimizers` (§7.3's mechanism),
checking that both runs return identical rows. That second idiom is arguably
a stronger correctness proof than a paired `EXPLAIN` match, since it exercises
the actual row-producing execution engine, not just the plan shape — it's
flagged here because it recurs through the rest of this chapter's deep dives
wherever a pass's test file wants to prove "this rewrite is invisible at the
result level," and this is as good a place as any to name it once.

## 7.9 `EMPTY_RESULT_PULLUP` — collapsing whole joins once one side is provably empty

**In one sentence:** given a plan where some subtree has *already* been
replaced by a `LogicalEmptyResult` marker (by an earlier pass — this pass
never proves emptiness itself), `EmptyResultPullup` walks the tree bottom-up
and, for each parent operator sitting directly above that marker, decides —
using the operator's own type, and, for joins, the join's `JoinType` —
whether the *parent* is now also guaranteed to produce zero rows, and if so
replaces the parent with a fresh `LogicalEmptyResult` of its own (or, for one
join type, with the parent's surviving non-empty child), so the "this
produces nothing" fact can propagate upward past operators the pass that
originally proved it had no way to reason about.

That first clause — "already been replaced… by an earlier pass" — is worth
sitting with before reading the source, because it's the single most
commonly misread thing about this pass: **`EmptyResultPullup` does not
itself look at column statistics, evaluate any predicate, or prove anything
false.** It has no notion of a value range, a min/max, or a constant. Its
entire job is *reactive*: something else already produced a bare
`LogicalEmptyResult` node somewhere in the tree, and this pass's only
question, at every ancestor of that node, is "given this specific parent
operator's semantics, does an empty child force an empty parent too?" —
which for some operators is always yes, for some join types is yes only
under specific conditions, and for others is provably *no* regardless of
statistics, which is exactly the correctness boundary this section is asked
to get right.

### 7.9.1 Where the emptiness this pass reacts to actually comes from

Before walking `EmptyResultPullup`'s own logic, it's worth establishing what
can possibly hand it a `LogicalEmptyResult` node to react to, since the pass
runs at position #13 — *before* `STATISTICS_PROPAGATION` (#34), the pass
§7.6.2 of the overview singles out as the primary source of
`ReplaceWithEmptyResult` calls. A repo-wide search for every call site that
constructs a `LogicalEmptyResult` node shows the real picture is broader than
"statistics propagation and nothing else":

```
src/optimizer/constant_or_null_simplification.cpp:159   (part of pass #1, EXPRESSION_REWRITER)
src/optimizer/filter_pushdown.cpp:301                    (pass #6)
src/optimizer/pushdown/pushdown_filter.cpp:19            (pass #6's per-operator handlers)
src/optimizer/pushdown/pushdown_get.cpp:159              (pass #6)
src/optimizer/pushdown/pushdown_inner_join.cpp:27,38     (pass #6)
src/optimizer/pushdown/pushdown_cross_product.cpp:76     (pass #6)
src/optimizer/pushdown/pushdown_semi_anti_join.cpp:24,34,45 (pass #6)
src/optimizer/pushdown/pushdown_aggregate.cpp:95         (pass #6)
src/optimizer/pushdown/pushdown_projection.cpp:207,216   (pass #6)
src/optimizer/pushdown/pushdown_unnest.cpp:38,47         (pass #6)
src/optimizer/pushdown/pushdown_secure_view.cpp:62       (pass #6)
src/optimizer/pushdown/pushdown_set_operation.cpp:62,83,100 (pass #6)
src/optimizer/pushdown/pushdown_limit.cpp:13             (pass #6)
src/optimizer/empty_result_pullup.cpp (5 sites)          (this pass, #13 — see below)
src/optimizer/statistics_propagator.cpp:42               (pass #34, ReplaceWithEmptyResult)
src/planner/subquery/flatten_dependent_join.cpp:1002      (binder-time, before the optimizer runs — Ch. 6)
```

In `pushdown_semi_anti_join.cpp`, the branch for an `ANTI` join whose right
side is already empty does not build an empty result. It returns the left
child. It does this through
`LogicalJoin::PromoteJoinSide` (line 43, see §7.9.3), so the join's output
column layout is kept. In `pushdown_cross_product.cpp:76`, on a cross product, filter pushdown first
looks for a constant bound on one table (`i > 4999`) together with a
`<`/`>` comparison that links it to the other table (`k >= i`). When it finds
that shape, it combines all the filters before it splits them between the two
sides, so it can derive new bounds such as `k > 4999`. If the combined filters
contradict each other, it returns an empty result right there.

Two things fall out of this list that matter for reading everything below
correctly. First, **`FILTER_PUSHDOWN` (pass #6) is, by call-site count, the
biggest single producer of `LogicalEmptyResult` nodes in the whole
optimizer** — every one of its thirteen per-operator handlers under
`src/optimizer/pushdown/` has its own `FilterResult::UNSATISFIABLE` escape
hatch (routed through `FilterCombiner::AddFilter`, which detects a
*syntactic* self-contradiction among the filters it's accumulated — `a > 1
AND a <= 1`, or a join condition that's a literal `0` ANDed into a bigger
conjunction — not a data-dependent, table-statistics-based one). Second,
**`FILTER_PUSHDOWN` runs at position #6, seven full positions before
`EMPTY_RESULT_PULLUP` at #13** — so by the time `EmptyResultPullup` gets its
one shot at the tree, it can already be looking at plenty of real
`LogicalEmptyResult` markers left behind by filter pushdown's own
unsatisfiability detection, plus whatever `ConstantOrNullSimplification`
(part of pass #1) folded even earlier. `STATISTICS_PROPAGATION`'s
`ReplaceWithEmptyResult` (§7.6.2 of the overview), by contrast, runs at #34 —
twenty-one positions *after* `EmptyResultPullup` has already had, and used up,
its only turn. Whatever `STATISTICS_PROPAGATION` proves empty for the first
time never gets a second chance to be pulled further up through a
surrounding join by this pass, in this run of the query — a direct
consequence of §7.2's one-shot, no-fixpoint architecture. (§7.9.3 below shows
that `STATISTICS_PROPAGATION` doesn't actually need `EmptyResultPullup` for
its own join-collapsing needs — it re-implements a smaller, parallel version
of the same join-type-aware logic inline, precisely because it can't lean on
a pass that already ran.)

### 7.9.2 Source walkthrough: `EmptyResultPullup::Optimize`

The whole pass is 116 lines (`src/optimizer/empty_result_pullup.cpp`). It
nominally inherits `LogicalOperatorVisitor`
(`src/include/duckdb/optimizer/empty_result_pullup.hpp:16`) but doesn't use
any of that base class's virtual dispatch — it's really just a plain
recursive function wearing the visitor's name:

```cpp
// src/optimizer/empty_result_pullup.cpp:70
unique_ptr<LogicalOperator> EmptyResultPullup::Optimize(unique_ptr<LogicalOperator> op) {
	for (idx_t i = 0; i < op->children.size(); i++) {
		op->children[i] = Optimize(std::move(op->children[i]));
	}
	switch (op->type) {
	...
```

The recursion is unconditionally bottom-up — every child is optimized
*before* the switch on `op->type` runs, for every operator type, with no
exceptions. This matters: it's what makes the "pull up" framing literal
rather than metaphorical. By the time any operator's own case in the switch
runs, every one of its children has already been fully resolved to whatever
it's going to be — including, possibly, a fresh `LogicalEmptyResult` that a
*sibling* pass at a lower level in the tree collapsed into during this same
recursive call. The tree is walked once, leaves first, and emptiness climbs
one level per recursive return, which is exactly what lets a three-deep
nested join tree collapse all the way to the root in a single pass, one
level at a time, as the recursion unwinds.

**The plain pass-through group.** Eight operator types get identical
treatment — if *any* child is a `LogicalEmptyResult`, the whole operator
becomes one too:

```cpp
// src/optimizer/empty_result_pullup.cpp:75
case LogicalOperatorType::LOGICAL_PROJECTION:
case LogicalOperatorType::LOGICAL_FILTER:
case LogicalOperatorType::LOGICAL_DISTINCT:
case LogicalOperatorType::LOGICAL_WINDOW:
case LogicalOperatorType::LOGICAL_GET:
case LogicalOperatorType::LOGICAL_INTERSECT:
case LogicalOperatorType::LOGICAL_PIVOT:
case LogicalOperatorType::LOGICAL_CROSS_PRODUCT: {
	for (auto &child : op->children) {
		if (child->type == LogicalOperatorType::LOGICAL_EMPTY_RESULT) {
			op = make_uniq<LogicalEmptyResult>(std::move(op));
			break;
		}
	}
	return op;
}
```

This grouping is safe precisely because none of these eight operator types
can manufacture rows out of nothing: a `LogicalProjection`, `LogicalFilter`,
`LogicalDistinct`, or `LogicalWindow` each has exactly one child and can only
ever emit as many (or fewer) rows than that child does, so an empty child
forces an empty parent unconditionally. `LOGICAL_CROSS_PRODUCT` and
`LOGICAL_INTERSECT` are multi-child but the same logic holds from either
direction — a cross product with either side empty has nothing to pair, and
an intersection with either side empty has nothing in common with the other.
`LOGICAL_PIVOT` reshapes columns into rows but still can't invent rows a
single upstream child didn't supply. `LOGICAL_GET` is the odd one out at
first glance — a table scan is normally a *leaf*, with zero children, so this
case would appear to be dead weight — but a `LogicalGet` wrapping a
table-in/table-out function (a table function whose argument is itself a
subquery, rather than a `LogicalGet` scanning a base table directly) can
carry a child subplan, and this case exists for that shape.

**`LOGICAL_MATERIALIZED_CTE` — a deliberately different, two-child rule.**
This is not folded into the group above, and the reason is a real,
git-documented correctness fix, not stylistic preference. The current source:

```cpp
// src/optimizer/empty_result_pullup.cpp:91
case LogicalOperatorType::LOGICAL_MATERIALIZED_CTE: {
	D_ASSERT(op->children.size() == 2);
	if (op->children[1]->type == LogicalOperatorType::LOGICAL_EMPTY_RESULT) {
		if (op->children[0]->HasSideEffects()) {
			return op;
		}
		op = make_uniq<LogicalEmptyResult>(std::move(op));
		break;
	}
	return op;
}
```

A `LogicalMaterializedCTE` has exactly two children with two *different*
roles: `children[0]` is the CTE's own definition (the subquery that gets
materialized once), and `children[1]` is the body — the rest of the query
that reads from it. The check is deliberately asymmetric: it looks at
`children[1]` (does the *consuming* side of the tree turn out to need
nothing at all?), not "any child," and it's guarded by
`op->children[0]->HasSideEffects()` — a recursive check
(`src/planner/logical_operator.cpp:96`) that walks the *definition* subtree
looking for `LOGICAL_INSERT`/`LOGICAL_UPDATE`/`LOGICAL_DELETE`/
`LOGICAL_MERGE_INTO`/`LOGICAL_COPY_TO_FILE` anywhere inside it. The reason
this guard exists is concrete and important: a `LogicalEmptyResult` node
that wraps a subtree does not execute that subtree — `src/execution/physical_plan/plan_empty_result.cpp:9`
builds the physical operator from nothing but `op.types` and
`op.estimated_cardinality`; the wrapped-and-discarded child is never turned
into a physical plan at all. So for a query like

```sql
WITH t AS (DELETE FROM some_table RETURNING *)
SELECT * FROM t WHERE FALSE
```

if the body (`SELECT * FROM t WHERE FALSE`) has already collapsed to
`LogicalEmptyResult`, collapsing the *entire* `LogicalMaterializedCTE` down
to a bare `LogicalEmptyResult` as well would silently make the `DELETE`
never run — a wildly wrong thing for an optimizer pass to do to a
data-modifying statement. The `HasSideEffects()` check exists specifically
to refuse that collapse whenever the definition side has a write anywhere
inside it, at any depth. This exact case was a real, separately-fixed bug:
before commit `caf4cf117a` ("Fix empty result pullup for materialized
CTEs"), `LOGICAL_MATERIALIZED_CTE` was folded into the *plain pass-through
group* above — meaning the pass checked "any child empty" (which, for a
two-child CTE node, meant it would just as happily fire on `children[0]`,
the *definition*, turning out empty as on `children[1]`, the body) with no
side-effect guard at all. The fix moved it into its own case, switched the
check to the body specifically, and added the guard.

**The join family — `PullUpEmptyJoinChildren`.** Five operator types are
routed to a shared helper:

```cpp
// src/optimizer/empty_result_pullup.cpp:102
case LogicalOperatorType::LOGICAL_EXCEPT:
case LogicalOperatorType::LOGICAL_ANY_JOIN:
case LogicalOperatorType::LOGICAL_DELIM_JOIN:
case LogicalOperatorType::LOGICAL_ASOF_JOIN:
case LogicalOperatorType::LOGICAL_COMPARISON_JOIN: {
	op = PullUpEmptyJoinChildren(std::move(op));
	break;
}
```

`PullUpEmptyJoinChildren` starts by recovering a single `JoinType` for
whichever of these five operator types it was actually handed —
`LOGICAL_EXCEPT` is synthetically mapped to `JoinType::ANTI` (DuckDB
implements set-difference internally as an anti-join plus deduplication),
and every comparison-join-shaped operator (`LOGICAL_DELIM_JOIN`,
`LOGICAL_ASOF_JOIN`, `LOGICAL_COMPARISON_JOIN`) is cast directly to
`LogicalComparisonJoin` to read its real `join_type`:

```cpp
// src/optimizer/empty_result_pullup.cpp:9
unique_ptr<LogicalOperator> EmptyResultPullup::PullUpEmptyJoinChildren(unique_ptr<LogicalOperator> op) {
	JoinType join_type = JoinType::INVALID;
	D_ASSERT(op->type == LogicalOperatorType::LOGICAL_COMPARISON_JOIN ||
	         op->type == LogicalOperatorType::LOGICAL_ANY_JOIN || op->type == LogicalOperatorType::LOGICAL_DELIM_JOIN ||
	         op->type == LogicalOperatorType::LOGICAL_ASOF_JOIN || op->type == LogicalOperatorType::LOGICAL_EXCEPT);
	switch (op->type) {
	case LogicalOperatorType::LOGICAL_DELIM_JOIN:
	case LogicalOperatorType::LOGICAL_ASOF_JOIN:
	case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
		join_type = op->Cast<LogicalComparisonJoin>().join_type;
		break;
	case LogicalOperatorType::LOGICAL_ANY_JOIN:
		join_type = op->Cast<LogicalAnyJoin>().join_type;
		break;
	case LogicalOperatorType::LOGICAL_EXCEPT:
		join_type = JoinType::ANTI;
		break;
	case LogicalOperatorType::LOGICAL_INTERSECT:
		join_type = JoinType::SEMI;
		break;
	default:
		break;
	}
```

That `LOGICAL_INTERSECT: join_type = JoinType::SEMI;` line is worth flagging
explicitly: it is dead code on this checkout. `LOGICAL_INTERSECT` is never
one of the five types `Optimize`'s outer switch routes into
`PullUpEmptyJoinChildren` in the first place — it's handled entirely by the
plain pass-through group two subsections up, which happens to produce the
same "either side empty ⇒ whole thing empty" answer for `INTERSECT` anyway,
by coincidence of semantics rather than by this line ever actually running.
This is noted here rather than silently corrected because this book's own
stated policy (front matter, §"Sources and how they're weighted") is to
report what the source actually does, including a harmless unreachable
branch, rather than tidy it up in the retelling.

With `join_type` established, a second switch applies the actual
emptiness-propagation rule, and this is the part worth reading slowly,
because it is the exact INNER-vs-OUTER correctness boundary this section was
asked to get right:

```cpp
// src/optimizer/empty_result_pullup.cpp:33
switch (join_type) {
case JoinType::SEMI:
case JoinType::INNER: {
	for (auto &child : op->children) {
		if (child->type == LogicalOperatorType::LOGICAL_EMPTY_RESULT) {
			op = make_uniq<LogicalEmptyResult>(std::move(op));
			break;
		}
	}
	break;
}
// For ANTI joins, if the right child is empty, the whole join collapses to the left child
case JoinType::ANTI: {
	if (op->children[1]->type == LogicalOperatorType::LOGICAL_EMPTY_RESULT &&
	    op->type != LogicalOperatorType::LOGICAL_EXCEPT) {
		op = std::move(op->children[0]);
		break;
	}
	if (op->children[0]->type == LogicalOperatorType::LOGICAL_EMPTY_RESULT) {
		op = make_uniq<LogicalEmptyResult>(std::move(op));
	}
	break;
}
case JoinType::MARK:
case JoinType::SINGLE:
case JoinType::LEFT: {
	if (op->children[0]->type == LogicalOperatorType::LOGICAL_EMPTY_RESULT) {
		op = make_uniq<LogicalEmptyResult>(std::move(op));
	}
	break;
}
default:
	break;
}
```

Reading this case by case, against `JoinType`'s own documented semantics
(`src/include/duckdb/common/enums/join_type.hpp`):

- **`INNER` and `SEMI`: either child empty ⇒ the whole join is empty.** This
  is the easy, symmetric case, and it's why the code loops over *all*
  children rather than singling one out. An `INNER` join can't produce a row
  unless both sides contribute at least one matching pair, so an empty
  either side is fatal to the whole result. `SEMI` (per the enum comment,
  "returns left side row ONLY if it has a join partner") is really an
  existence check against the right side — if the right side is empty there
  are no partners to have, and if the left side is empty there's nothing to
  check partnership for either way — so both directions collapse the same
  way `INNER` does.

- **`ANTI`: the two children are *not* symmetric, and the asymmetry is the
  interesting part.** An `ANTI` join (per the enum comment, "returns left
  side row ONLY if it has NO join partner") keeps a left row exactly when
  the right side has *nothing* matching it. If the right side is empty
  outright, then trivially *every* left row has no partner — the entire
  `ANTI` join is equivalent to just its left child, unfiltered, which is
  exactly what `op = std::move(op->children[0])` does: it doesn't wrap
  anything in `LogicalEmptyResult` at all, it *replaces the whole join node*
  with its left child directly, dropping the join (and its now-provably-
  useless right side) from the tree entirely. If instead the *left* side is
  empty, there's nothing to check for partnership in the first place, and
  the join is trivially empty regardless of the right side — the ordinary
  `LogicalEmptyResult` wrap applies there. The `op->type !=
  LogicalOperatorType::LOGICAL_EXCEPT` guard on the first branch is a second,
  independent correctness carve-out: `EXCEPT` is implemented as an anti-join
  *plus implicit deduplication* (`A EXCEPT B` removes duplicate rows of `A`,
  not just rows of `A` that also appear in `B`), and collapsing the node
  down to a bare, un-deduplicated left child would silently drop that
  deduplication — a genuine correctness bug, not merely a missed
  optimization, for a query like `SELECT 1 UNION ALL SELECT 1 EXCEPT SELECT
  2` if `B` (`SELECT 2`, filtered to nothing relevant) collapsed empty. The
  left-side-empty branch carries no such carve-out, because deduplicating an
  already-empty set is still trivially empty either way. **This exact
  ANTI-join case was itself a bugfix, not present from the pass's first
  version:** the source used to carry only a `// TODO: For ANTI joins, if
  the right child is empty, you can replace the whole join with the left
  child` comment with no implementation, meaning `ANTI` fell through to
  exactly the same "check only `children[0]`" rule as `MARK`/`SINGLE`/`LEFT`
  below — correct for the left-empty direction, silently missing the
  right-empty-collapses-to-left-child optimization entirely — until commit
  `58d4de9834` ("Fix empty result pullup for EXCEPT") implemented it, with
  the `EXCEPT` exclusion built in from that same commit.

- **`MARK`, `SINGLE`, `LEFT`: only the *left* child's emptiness is checked,
  and this absence is the load-bearing part.** A `MARK` join annotates every
  left row with a boolean (does a partner exist or not) without ever
  dropping a left row; `SINGLE` behaves like a `LEFT OUTER JOIN` capped to
  at most one matched partner per left row, `NULL`-padding when none exists;
  and plain `LEFT` is the ordinary SQL `LEFT [OUTER] JOIN`. **All three
  preserve every left row regardless of whether the right side matches
  anything at all** — that's the entire point of the "outer" in "left outer
  join." If the right child (`children[1]`) turns out to be empty, that does
  *not* make the whole join empty: it means every left row simply gets
  `NULL`s (or `false`, for `MARK`) on the right-hand columns instead of a
  match, which is a completely different, non-empty result. The switch
  statement's silence about `children[1]` for these three types is not an
  oversight to be filled in later — it is the correctness boundary itself,
  expressed as an *absence* of a check rather than a present one. Only if
  the *left* side is already empty does the whole thing collapse, because
  with zero left rows to begin with there's nothing left to pad with `NULL`s
  in the first place.

**What's conspicuously absent even from this join-family group.** The switch
has no case at all for `JoinType::RIGHT`, `JoinType::OUTER` (full outer),
`JoinType::RIGHT_SEMI`, or `JoinType::RIGHT_ANTI` — all four fall to
`default: break;`, meaning an already-empty child below a `RIGHT JOIN` or a
`FULL OUTER JOIN` is *never* pulled up any further by this pass, regardless
of which side is empty. Two of those four gaps have a clean, pipeline-order
explanation and two do not, and the two categories are worth telling apart
precisely: `RIGHT_SEMI`/`RIGHT_ANTI` are, per their own doc comment in
`join_type.hpp`, "created by the optimizer when the children of a
semi/anti join need to be switched so that the build side can be the
smaller table" — that switching decision is `BUILD_SIDE_PROBE_SIDE`'s job,
pass **#27**, which runs *fourteen positions after* `EMPTY_RESULT_PULLUP` at
#13. A `RIGHT_SEMI`/`RIGHT_ANTI` join simply cannot exist anywhere in the
tree yet at the point this pass runs — there is nothing missing to fix here,
because there is nothing yet to fix. `RIGHT` and `OUTER` (full) are a
genuinely different story: both can be written directly in SQL (`RIGHT
JOIN`, `FULL OUTER JOIN`) and both can exist in the tree from the moment the
binder produces it, long before `EMPTY_RESULT_PULLUP` ever runs — and yet
neither gets a case here. A `RIGHT JOIN` with an empty *left* child (the
non-preserved side) is, by the exact mirror-image argument used for `LEFT`
above, provably empty too — but this pass simply does not know that. This is
a real, documented (by omission) gap in this pass's coverage on this
checkout, not something explained away by an ordering argument the way the
semi/anti gap is.

### 7.9.3 `StatisticsPropagator`'s own, independent, parallel join-collapsing logic

Because `EMPTY_RESULT_PULLUP` only gets one turn, at #13, and
`STATISTICS_PROPAGATION` — the pass that actually proves most *data-dependent*
emptiness, as opposed to the purely syntactic contradictions
`FILTER_PUSHDOWN` catches earlier — doesn't run until #34, `StatisticsPropagator`
cannot lean on `EmptyResultPullup` at all for its own needs. It re-implements
a smaller version of the same join-type-aware "does emptiness collapse the
whole join, one side, or nothing" logic directly and inline, in
`src/optimizer/statistics/operator/propagate_join.cpp`, triggered the moment
a join *condition* (not an already-empty child — an actual comparison, freshly
proven unsatisfiable from real column min/max statistics) is shown to never
hold:

```cpp
// src/optimizer/statistics/operator/propagate_join.cpp:16
bool StatisticsPropagator::HandleJoinNeverMatches(LogicalJoin &join, unique_ptr<LogicalOperator> &node_ptr) {
	switch (join.join_type) {
	case JoinType::RIGHT_SEMI:
	case JoinType::SEMI:
	case JoinType::INNER:
		// semi or inner join on false; entire node can be pruned
		ReplaceWithEmptyResult(node_ptr);
		return true;

		// If the filter is always false or Null, the join output is exactly one of its sides:
		// promote that side together with its projection map so the output layout stays unchanged.
	case JoinType::RIGHT_ANTI:
		node_ptr = LogicalJoin::PromoteJoinSide(std::move(join.children[1]), std::move(join.right_projection_map));
		return true;
	case JoinType::ANTI:
		node_ptr = LogicalJoin::PromoteJoinSide(std::move(join.children[0]), std::move(join.left_projection_map));
		return true;
	case JoinType::LEFT:
		// anti/left outer join: replace right side with empty node
		ReplaceWithEmptyResult(join.children[1]);
		return true;
	case JoinType::RIGHT:
		// right outer join: replace left side with empty node
		ReplaceWithEmptyResult(join.children[0]);
		return true;
	default:
		// other join types: can't do much meaningful with this information
		// full outer join requires both sides anyway; we can skip the execution of the actual join, but eh
		// mark/single join requires knowing if the rhs has null values or not
		return false;
	}
}
```

The shape of the reasoning is identical to `EmptyResultPullup`'s — `INNER`/
`SEMI` collapse the whole node, `ANTI` collapses to the left child, `LEFT`
only empties the non-preserved side — but notice what's *different*: this
version *does* have a case for `RIGHT` (mirroring `LEFT`, replacing the
*left* side with empty, since a `RIGHT JOIN` preserves its right side) and
for `RIGHT_SEMI`/`RIGHT_ANTI` (`RIGHT_SEMI` sits with `SEMI`/`INNER`;
`RIGHT_ANTI` keeps its *right* child, the side it emits) — both gaps this
section just flagged in `EmptyResultPullup` itself. That's not a contradiction; it's the same
pipeline-order argument from the other direction. By the time
`STATISTICS_PROPAGATION` runs at #34, `BUILD_SIDE_PROBE_SIDE` (#27) has
already had the chance to produce `RIGHT_SEMI`/`RIGHT_ANTI` joins, and a
`RIGHT JOIN` written directly in SQL has been sitting in the tree the whole
time — so a pass running this late genuinely needs to cover more join-type
ground than one running at #13 does. `MARK`/`SINGLE`/`OUTER` (full) fall to
`default: return false;` here too, with an explicit comment owning the
limitation rather than silently omitting it: `MARK`/`SINGLE` need to know
whether the right side can produce `NULL`s specifically (not just whether it
has zero rows) before anything can be concluded, and a full `OUTER` join
needs both sides regardless, so an unsatisfiable condition alone doesn't let
either be skipped.

When an `ANTI` or `RIGHT_ANTI` join can never match, the propagator replaces
the join with one of its sides. It does this through a helper,
`LogicalJoin::PromoteJoinSide`. Just moving the child up would lose the join's
*projection map* — the list that says which of the child's columns the join
passes on. Operators above the join could then read the wrong column.
`RIGHT_ANTI` does not swap its children first; it promotes `children[1]`
directly.

`PromoteJoinSide` is small:

```cpp
// src/planner/operator/logical_join.cpp:87
unique_ptr<LogicalOperator> LogicalJoin::PromoteJoinSide(unique_ptr<LogicalOperator> child,
                                                         vector<ProjectionIndex> projection_map) {
	if (projection_map.empty()) {
		return child;
	}
	auto filter = make_uniq<LogicalFilter>();
	filter->projection_map = std::move(projection_map);
	filter->children.push_back(std::move(child));
	if (filter->children[0]->has_estimated_cardinality) {
		filter->SetEstimatedCardinality(filter->children[0]->estimated_cardinality);
	}
	filter->ResolveOperatorTypes();
	return std::move(filter);
}
```

If the join had no projection map, the child is returned unchanged. If it
had one, the child is wrapped in a `LogicalFilter` with *no conditions*. The
filter does no filtering. It is there only to carry the projection map, so
the operators above see the same columns in the same order as they would
over the join. Filter pushdown's own `ANTI`-with-empty-right-side branch
(`pushdown_semi_anti_join.cpp:43`) uses the same helper. Without it, a
correlated `IN`/`EXISTS` next to such an `ANTI` join could fail with
`INTERNAL Error: Attempted to access index 3 within vector of size 3`
(DuckDB issue #27091).

The companion function, `HandleJoinAlwaysMatches`
(`propagate_join.cpp:64`), handles the opposite direction — a join
*condition* proven unconditionally `TRUE` — by turning `INNER`/`SEMI` joins
into a plain (cheaper) `LogicalCrossProduct`, which is exactly the mechanism
behind the `statistics_join.test` cases in §7.9.5 below whose `EXPLAIN`
output shows `CROSS_PRODUCT` rather than `EMPTY_RESULT`.

> **Note:** the cross product that replaces an always-matching join keeps the
> join's output columns. A cross product emits *every* column of
> both sides. A `SEMI` join emits only its left side, and any join may also
> have a projection map that drops columns. So the code builds a new
> projection map with a helper, `AddProjectedColumns`, and wraps the cross
> product with `PromoteJoinSide`. The test
> `test/optimizer/statistics/statistics_join_always_matches_layout.test`
> covers it. It runs with `SET debug_verify_column_bindings=true` and checks
> queries like this one. The test's own comment says the condition
> `a1.x < b.x` "is always true according to the statistics" (`a` holds 1
> and 2, `b` holds 5):
>
> ```sql
> SELECT sum(c.y), count(*) FROM a a1 SEMI JOIN b ON a1.x < b.x LEFT JOIN a a3 ON true JOIN c ON a3.x = c.x;
> ----
> 60	4
> ```

### 7.9.4 The fuzzer regression: `empty_result_pullup_delim_joins.test`

```sql
-- test/fuzzer/duckfuzz/empty_result_pullup_delim_joins.test
SELECT
	NULL
FROM
main.partsupp AS ref_1
INNER JOIN main.region ON (NULL)
INNER JOIN main.customer AS ref_3 ON
(EXISTS(SELECT NULL FROM (SELECT ref_3.c_acctbal AS c0)))
INNER JOIN
(SELECT
	ref_9.s_phone AS c0,
	ref_8.r_regionkey AS c1,
	ref_9.s_phone AS c2
FROM main.region AS ref_8
RIGHT JOIN
	main.supplier AS ref_9 ON ((1 AND (ref_9.s_comment ~~ ref_9.s_address) AND 0 AND (ref_9.s_name !~~ ref_9.s_name)))
LEFT JOIN main.customer AS ref_10 ON ((ref_10.c_address ~~* ref_10.c_comment))
WHERE (ref_9.s_phone IS NOT NULL) LIMIT 53) AS subq_2
ON ((CASE  WHEN (1) THEN (subq_2.c2) ELSE subq_2.c2 END !~~ ref_1.ps_comment))
```

This is a `duckfuzz`-generated query — the group name and the `test/fuzzer/`
path both say so directly — meaning it was almost certainly not hand-written
to demonstrate anything in particular; it's the minimized remainder of a
much larger, randomly-generated query that used to crash or misbehave, cut
down until only the shape needed to reproduce the bug survived. The `#
description: test why column bindings fail` header is the fuzzer harness's
own terse note-to-self, not a designed teaching example, so this walkthrough
reasons from the structure and the verified source diff rather than from any
narrative the test file itself supplies.

Two SQL shapes inside this query are worth naming before the fix itself,
because they're exactly the kind of pattern the rest of this section already
established can produce an early `LogicalEmptyResult`:

- `INNER JOIN main.region ON (NULL)` — a join condition that is the literal
  constant `NULL`, not merely an expression that might evaluate to `NULL`.
  `ConstantOrNullSimplification` (part of pass #1) or `FILTER_PUSHDOWN`'s
  filter combiner (pass #6) can fold a condition this trivially
  unsatisfiable directly, well before `EMPTY_RESULT_PULLUP` ever runs.
- `RIGHT JOIN main.supplier AS ref_9 ON ((1 AND (ref_9.s_comment ~~
  ref_9.s_address) AND 0 AND (ref_9.s_name !~~ ref_9.s_name)))` — a
  conjunction with a literal `0` ANDed directly into it. `AND`-ing anything
  with a constant `0` (`FALSE`) makes the whole conjunction unconditionally
  `FALSE` regardless of what the other three conjuncts evaluate to — exactly
  the shape `test/optimizer/transitive_filters.test` demonstrates folding to
  `EMPTY_RESULT` via pure syntactic contradiction detection, with no table
  statistics involved at all (§7.9.5 below).

The `EXISTS(SELECT NULL FROM (SELECT ref_3.c_acctbal AS c0))` correlated
subquery, meanwhile, is exactly the shape Chapter 6 covers in full: a
subquery correlated on `ref_3.c_acctbal`, which the planner's
`FlattenDependentJoins` machinery turns into a delim join — a
`LogicalComparisonJoin` whose type is `LOGICAL_DELIM_JOIN` (there is no
separate `LogicalDelimJoin` class). It behaves like an ordinary comparison
join (it carries a real `JoinType`) but additionally
threads the correlated column's distinct values through to one or more
`LogicalDelimGet` operators inside its right-hand subplan, so the
(conceptually per-outer-row) subquery only actually runs once per *distinct*
correlated value rather than once per row. `DELIMINATOR` (pass #10) is the
pass responsible for cleaning up redundant `DelimJoin`/`DelimGet` pairs once
this flattening is done — and it runs at position #10, **three slots before**
`EMPTY_RESULT_PULLUP` at #13.

> **Note:** with default settings, this delim join is *not* what
> `EMPTY_RESULT_PULLUP` sees. The setting `delim_join_as_cte` defaults to `true`, so right after flattening the
> planner turns every delim join into plain joins over materialized CTEs
> (Chapter 6 §6.4.1). The test file does not change that setting, so a
> default run of it reaches `EMPTY_RESULT_PULLUP` with no
> `LOGICAL_DELIM_JOIN` in the tree. The `LOGICAL_DELIM_JOIN` cases below
> matter when `delim_join_as_cte = false`. In a default run, the test
> (`statement ok`) only checks that the CTE form of the same query plans and
> runs without an error.

Here is the fix, verified against the actual git history of this file
(commit `08c01fbbbb`, "pullup empty results through delim joins as well"):

```diff
 unique_ptr<LogicalOperator> EmptyResultPullup::PullUpEmptyJoinChildren(unique_ptr<LogicalOperator> op) {
 	JoinType join_type = JoinType::INVALID;
 	D_ASSERT(op->type == LogicalOperatorType::LOGICAL_COMPARISON_JOIN ||
-	         op->type == LogicalOperatorType::LOGICAL_ANY_JOIN || op->type == LogicalOperatorType::LOGICAL_EXCEPT);
+	         op->type == LogicalOperatorType::LOGICAL_ANY_JOIN ||
+	         op->type == LogicalOperatorType::LOGICAL_DELIM_JOIN ||
+	         op->type == LogicalOperatorType::LOGICAL_EXCEPT);
 	switch (op->type) {
+	case LogicalOperatorType::LOGICAL_DELIM_JOIN:
 	case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
 		join_type = op->Cast<LogicalComparisonJoin>().join_type;
 		break;
@@
 	case LogicalOperatorType::LOGICAL_EXCEPT:
 	case LogicalOperatorType::LOGICAL_ANY_JOIN:
+	case LogicalOperatorType::LOGICAL_DELIM_JOIN:
 	case LogicalOperatorType::LOGICAL_COMPARISON_JOIN: {
 		op = PullUpEmptyJoinChildren(std::move(op));
 		break;
 	}
```

Before this commit, `LOGICAL_DELIM_JOIN` was simply not one of the five
types `Optimize`'s outer switch recognized at all — it fell straight to
`default: break;`, which means the pass's recursive descent still visited
and fully resolved *both* of a `DelimJoin`'s children (that top-of-function
loop, `for (i) op->children[i] = Optimize(...)`, runs unconditionally for
every operator type, `DelimJoin` included), but once one of those children
came back as a bare `LogicalEmptyResult`, the `DelimJoin` node sitting
directly above it was never given the chance to ask "does *my* join type,
given this now-empty child, make me collapse further too?" the way an
ordinary `LogicalComparisonJoin` with the identical `join_type` would have.
A query shape where a correlated `EXISTS` is decorrelated into a `DelimJoin`
sitting inside an otherwise heavily-nested, multi-`INNER JOIN` fuzzer query
like this one — where *other* branches genuinely do collapse to
`LogicalEmptyResult` via the constant-`NULL`/literal-`0` patterns just
described — is exactly the shape where a `DelimJoin` could end up standing
directly on top of an already-empty child while every ordinary comparison
join around it in the same tree keeps collapsing correctly, leaving the
`DelimJoin` as a structural outlier: a join node whose own emptiness-derived
simplification opportunity was silently skipped purely because of its
`LogicalOperatorType`, not because of anything about its `join_type` or its
actual children.

Being precise about what this book can and can't verify here: the exact
downstream symptom that made the fuzzer report "column bindings fail" isn't
spelled out in the commit message, and this book's own sourcing policy
(front matter, §"A note on scope and honesty about uncertainty") is to say so
rather than invent a specific crash trace. What can be said with confidence,
because it's a direct reading of the diff: a `DelimJoin` left standing atop
an empty child, when it should have collapsed further (or should have
disappeared entirely, in the `ANTI`-with-empty-right-child case), is a tree
shape that later passes and the mandatory post-pass `Verify(*plan)` check
(§7.2 of the overview — column-binding-resolution consistency, re-checked
after *every single* `RunOptimizer` call) were never designed to expect from
`EmptyResultPullup`'s own output. Whatever the precise failure mode was on
the original, larger fuzzer-generated query this file is the minimized
remainder of, the fix that resolved it is exactly the one-line-per-switch
addition shown above, and its correctness follows directly from the fact
that a `LOGICAL_DELIM_JOIN` genuinely does carry a real `JoinType` with the
identical semantics `PullUpEmptyJoinChildren` already knew how to reason
about for every other comparison-join-shaped operator — the bug was a gap
in *which operator types get routed to that logic*, not a gap in the logic
itself.

### 7.9.5 Broader test-suite coverage

**`test/optimizer/statistics/statistics_join.test` — the INNER-vs-LEFT split,
explicitly annotated by the test's own author.** Two disjoint three-row
tables:

```sql
CREATE TABLE integers  AS SELECT * FROM (VALUES (1), (2), (3)) tbl(i);
CREATE TABLE integers2 AS SELECT * FROM (VALUES (4), (5), (6)) tbl(i);
```

```sql
-- inner join, join cannot match
EXPLAIN SELECT i1.i FROM integers i1 JOIN integers2 i2 ON i1.i=i2.i ORDER BY 1;
----
logical_opt	<REGEX>:.*EMPTY_RESULT.*
```

Since `integers`' max is `3` and `integers2`'s min is `4`, `i1.i = i2.i` can
never hold — `STATISTICS_PROPAGATION`'s `PropagateComparison` proves this
directly from propagated min/max (this is real, data-dependent proof, unlike
`FILTER_PUSHDOWN`'s purely syntactic contradictions), and
`HandleJoinNeverMatches` (§7.9.3) collapses the whole `INNER` join to
`ReplaceWithEmptyResult(node_ptr)` — the entire node, not merely a child, so
`EmptyResultPullup` itself (which already ran, 21 positions earlier) has
nothing left to do here; the collapse in the final `logical_opt` output is
`STATISTICS_PROPAGATION`'s own doing. The test file's very next case makes
the boundary explicit with its own comment:

```sql
-- left join
-- join cannot match
-- note that the result is not actually empty here; only the RHS is set to the empty result
EXPLAIN SELECT i1.i FROM integers i1 LEFT JOIN integers2 i2 ON i1.i=i2.i ORDER BY 1;
----
logical_opt	<REGEX>:.*EMPTY_RESULT.*
```

The `EMPTY_RESULT` the `EXPLAIN` regex matches here is real, but it's
sitting on the *right* child specifically —
`HandleJoinNeverMatches`'s `case JoinType::LEFT: ReplaceWithEmptyResult(join.children[1]);`
— while the `LEFT JOIN` node itself survives, now feeding from an empty
right side and its own ordinary `LEFT`-join `NULL`-padding logic, which is
exactly why the actual row output two sections later in the same file
confirms all three left rows survive with no rows dropped:

```sql
SELECT i1.i FROM integers i1 LEFT JOIN integers2 i2 ON i1.i=i2.i ORDER BY 1;
----
1
2
3
```

`test/optimizer/statistics/statistics_join_null.test` runs the identical
pair of `INNER`/comparison-operator queries with `NULL` values mixed into
both tables, and adds one more sharp, source-grounded observation as an
inline comment: an `i1.i < i2.i` join that's provably a match for *every
non-null* pairing is still *not* rewritten to a `CROSS_PRODUCT` the way the
null-free version of the same query is, "because there are null values! we
CANNOT convert to a cross product (well… we could, if we would push a `x IS
NOT NULL` filter on both sides) (but that is for another day)" — the test
author's own words, kept verbatim because they're a precise, honest
statement of a known, deliberately-left optimization gap rather than a bug.

**`test/optimizer/any_join_condition_pruning.test` — the same `INNER`/outer
distinction, reached via an `ANY_JOIN` rather than a `COMPARISON_JOIN`.**

```sql
CREATE TABLE t(a FLOAT[3], b FLOAT[3]);
INSERT INTO t VALUES ([1,2,3],[4,5,6]), ([0,1,0],[1,0,0]);

-- an always-true ANY join becomes a cross product
EXPLAIN (FORMAT yaml) SELECT * FROM t t1 JOIN t t2 ON array_cosine_similarity(t1.a, t2.a) IS NOT NULL
----
logical_opt	<REGEX>:.*CROSS_PRODUCT.*

-- ... and one that can never match is pruned entirely
EXPLAIN (FORMAT yaml) SELECT * FROM t t1 JOIN t t2 ON array_cosine_similarity(t1.a, t2.a) IS NULL
----
logical_opt	<REGEX>:.*EMPTY_RESULT.*

SELECT count(*) FROM t t1 JOIN t t2 ON array_cosine_similarity(t1.a, t2.a) IS NULL
----
0
```

`array_cosine_similarity(...)` never returns `NULL` for two non-`NULL` fixed-
size `FLOAT` arrays, so `... IS NULL` is a condition `ClassifyFilter` proves
unconditionally `FILTER_ALWAYS_FALSE` inside `PropagateStatistics(LogicalAnyJoin
&, ...)` (`propagate_join.cpp:256`), routing straight into the same
`HandleJoinNeverMatches` this section already walked through — this is the
`LogicalAnyJoin` path through the identical function, not a separate
mechanism. The file's closing section then demonstrates the same `LEFT`/
`ANTI`-preserve-non-matching-rows principle one more time, with an
always-*true* condition this time, deliberately chosen to show the guard
holds in both truth directions:

```sql
-- an always-true ANY join is not rewritten for join types that also emit non-matching rows
SELECT count(*) FROM l ANTI JOIN r ON array_cosine_similarity([1.0,2.0,3.0]::FLOAT[3], [4.0,5.0,6.0]::FLOAT[3]) IS NOT NULL
----
2

SELECT count(*) FROM l LEFT JOIN r ON array_cosine_similarity([1.0,2.0,3.0]::FLOAT[3], [4.0,5.0,6.0]::FLOAT[3]) IS NOT NULL
----
2
```

An always-`TRUE` condition on an `ANTI` join means *no* left row ever
qualifies (every left row has a "partner," by the always-true condition,
so `ANTI`'s "no partner" requirement is never met) — but `HandleJoinAlwaysMatches`'s
`default: return false;` for `ANTI` (§7.9.3) explicitly declines to
special-case this, with its own comment explaining why: *"anti joins only
produce an empty result when the other side has rows — with no rows to
match against, every tuple survives."* Since `r` here has zero rows
(`CREATE TABLE r(y INTEGER)` with no `INSERT`), the always-true condition is
moot — there's no partner to find regardless of the condition's truth value
— and both `l`'s rows survive the `ANTI` join, matching the `2` returned.

**An empty input does not make an ungrouped aggregate empty.** The
pass-through list in §7.9.2 has no case for
`LOGICAL_AGGREGATE_AND_GROUP_BY`, and that is on purpose. An aggregate with
no `GROUP BY` (an *ungrouped* aggregate) always returns exactly one row,
even when its input is empty: `SELECT count(*) FROM empty_table` returns
`0`, not zero rows. So an empty child below it must *not* be pulled up.

`STATISTICS_PROPAGATION` must respect the same rule in its *row-count*
estimates. If it told the parent that an ungrouped aggregate could return as
many rows as its input, then over an empty table that would be `0`. So the
aggregate reports exactly one row, and it also clears the leftover input count
in `node_stats`, so that a parent join does not pick it up (DuckDB issues
#26401 and #27714).

```cpp
// src/optimizer/statistics/operator/propagate_aggregate.cpp:544
const auto is_ungrouped = aggr.groups.empty();
// after we propagate statistics - try to directly execute aggregates using statistics
TryExecuteAggregates(aggr, node_ptr);

if (is_ungrouped) {
	// Ungrouped aggregate always returns exactly one row, even over empty input
	// node_stats holds the input cardinality: do not leak it to the parent operator
	node_stats.reset();
	return make_uniq<NodeStatistics>(1U, 1U);
}
```

Why would a wrong count matter? With a maximum of `0` rows, the propagator
would think the subtree could never produce a row, so a `LIMIT 1` above it
would look useless and be dropped. With the `LIMIT` gone, the query would
return too many rows. `test/optimizer/statistics/statistics_join_ungrouped_aggregate_limit.test`
shows the join case (table `t` is empty):

```sql
query I
SELECT count(*) FROM (SELECT * FROM range(3) r(i), (SELECT count(*) AS n FROM t) WHERE i >= n LIMIT 1)
----
1
```

`count(*)` over the empty `t` gives one row with `n = 0`. All three rows of
`range(3)` satisfy `i >= 0`, and the `LIMIT 1` keeps one of them. If the
`LIMIT` were dropped, the answer would be `3`.
`test/sql/subquery/exists/test_issue_26401.test` shows the `EXISTS` case:
`SELECT EXISTS (SELECT 1 FROM t WHERE NOT EXISTS (SELECT 1 FROM e))` must
return `true` when `e` is empty.

**`test/optimizer/transitive_filters.test` — the purely syntactic,
statistics-free contradiction path.**

```sql
EXPLAIN SELECT * FROM (VALUES(1),(2),(3)) tbl(a) WHERE a > 1 AND NOT (a > 1);
----
logical_opt	<REGEX>:.*EMPTY_RESULT.*

EXPLAIN SELECT * FROM (VALUES(1),(2),(3)) tbl(a) WHERE a > 1 AND a <= 1;
----
logical_opt	<REGEX>:.*EMPTY_RESULT.*

EXPLAIN SELECT * FROM (VALUES(1),(2),(3)) tbl(a) WHERE a >= 2 AND a < 2;
----
logical_opt	<REGEX>:.*EMPTY_RESULT.*
```

Every one of these is a compile-time-provable contradiction that needs no
knowledge of what's actually stored in `tbl` — `FilterCombiner::AddFilter`
(inside `FILTER_PUSHDOWN`, pass #6, via the `pushdown_filter.cpp`/
`filter_pushdown.cpp:301` unsatisfiability escape hatch) merges the two
range constraints on `a` symbolically and finds the merged range is empty,
long before `STATISTICS_PROPAGATION` or `EMPTY_RESULT_PULLUP` even get a
turn. This is the same mechanical category as the literal-`0`-ANDed-into-a-
join-condition pattern spotted inside §7.9.4's fuzzer query above — proof by
syntax, not by data.

**`test/sql/cte/cte_inlining_empty_result.test` — the interaction with
`CTE_INLINING`'s own inlining criteria.** No sibling file in this chapter
directory currently covers `CTE_INLINING` in depth, so this is worked out
directly from `src/optimizer/cte_inlining.cpp` rather than cross-referenced.
`CTE_INLINING` runs *twice* — #2 and #12 — specifically because a CTE that
wasn't a good inlining candidate the first time around can become one after
later rewrites change its shape (00-overview.md, pass #2's entry). One of the
concrete conditions that makes a CTE "cheap to inline regardless of its
base-table reference count" is written directly into the source:

```cpp
// src/optimizer/cte_inlining.cpp:208
bool is_cheap_to_inline = op->children[0]->type == LogicalOperatorType::LOGICAL_EMPTY_RESULT ||
                          op->children[0]->type == LogicalOperatorType::LOGICAL_CTE_REF ||
                          EndsInDummyScan(*op->children[0]);
```

If a CTE's own definition has *already* collapsed to a bare
`LogicalEmptyResult` — which, by the time the *second* `CTE_INLINING`
occurrence runs at #12, `FILTER_PUSHDOWN` (#6) has had every chance to do,
for a CTE body containing something like `WHERE NULL` — inlining it is
free, so `CTE_INLINING` copies that trivial empty subtree directly into
*every* reference site rather than materializing one shared copy. This is
exactly what the test's physical-plan pattern is checking for:

```sql
WITH q AS (SELECT id AS c0 FROM typed_scalars WHERE NULL LIMIT 61)
SELECT t5.c0 FROM q t5 JOIN q t6 ON t5.c0 = t6.c0;
----
physical_plan	<REGEX>:.*EMPTY_RESULT.*EMPTY_RESULT.*
```

Two separate `EMPTY_RESULT` markers, not one — because `t5` and `t6` are now
two *independently inlined copies* of the same already-empty definition
(this is also why the same query's `<!REGEX>:.*CTE.*` check on the physical
plan passes: nothing about a materialized CTE survives once inlining has
replaced both reference sites outright). Once inlined this way, the `JOIN`
combining `t5`/`t6` is an ordinary `INNER` `LogicalComparisonJoin` whose
*both* children are already `LogicalEmptyResult` — which
`EmptyResultPullup::PullUpEmptyJoinChildren`'s plain `INNER`/`SEMI` rule
(§7.9.2) handles without needing anything CTE-specific at that point; the
CTE-aware part of the story is entirely `CTE_INLINING`'s own
`is_cheap_to_inline` criterion, one pass before this one gets involved at
all.

**`test/fuzzer/duckfuzz/semi_join_has_correct_left_right_relations.test` —
a documented, deliberate side effect on *observable error behavior*, not a
correctness bug in this pass.** This file's own comments are worth quoting
directly, because they say something this section would otherwise have to
reconstruct from scratch: that proving a subtree empty can change whether a
query that would otherwise raise a runtime error does so at all.

```sql
-- This one should throw a Conversion Error. However, FilterPullup now pulls up an unsatisfiable join filter,
-- * there's a lot of those in this query: "... INNER JOIN ... ON (NULL)"
-- so that part of the plan is never executed, and we just get a empty result
query I
SELECT NULL FROM main.all_types AS ref_0 INNER JOIN main.all_types AS ref_1 INNER JOIN main.all_types AS ref_2 ON ((SELECT NULL)) ...
----


-- When we disable FilterPullup and EmptyResultPullup we get the expected error
statement ok
set disabled_optimizers to 'filter_pullup,empty_result_pullup'

statement error
SELECT NULL FROM main.all_types AS ref_0 INNER JOIN main.all_types AS ref_1 INNER JOIN main.all_types AS ref_2 ON ((SELECT NULL)) ...
----
<REGEX>:.*Conversion Error:.*
```

The query contains, buried inside one branch, an expression
(`ref_4."varchar"` used directly as a boolean join condition, among others)
that would raise a `Conversion Error` *if that branch ever actually
executed*. With the optimizer running normally, one of the several
`... INNER JOIN ... ON (NULL)` conditions elsewhere in the same query gets
proven unsatisfiable by `FILTER_PUSHDOWN`'s combiner, collapses to
`LogicalEmptyResult`, and `EMPTY_RESULT_PULLUP` (plus `FILTER_PULLUP`
deriving further unsatisfiable filters from it, per §7.6.3 of the overview)
propagates that emptiness widely enough through the surrounding join tree
that the branch containing the type-erroring expression is pruned away
entirely — it's simply never turned into a physical operator, so it never
gets a chance to throw. Disabling both `filter_pullup` and
`empty_result_pullup` (not `filter_pushdown` — the unsatisfiability
detection that seeds the very first `LogicalEmptyResult` still runs; it's
specifically the *further* propagation of that fact through more of the
tree that the test wants stopped) restores the "expected" error. This is
the exact same category of behavior `REORDER_FILTER`'s `CanThrow()` guard
(§7.6.1 of the overview) exists to prevent for conjunct reordering — but
notably, *no equivalent guard exists here*: `EmptyResultPullup` and
`FilterPullup` will happily discard a branch containing a throwing
expression the moment they can prove that branch contributes nothing to the
final row count, without asking whether discarding it changes whether an
error is observed. The test suite's own two-comment structure (first
comment explains the surprising behavior, second block shows how to turn
the passes off to get the "naive" behavior back) is DuckDB's own way of
saying: this is known, accepted, and not scheduled to be fixed — proving a
branch empty is allowed to make an error message disappear.

### 7.9.6 Summary of this pass's boundaries

Put together, `EMPTY_RESULT_PULLUP`'s coverage on this checkout is:

| Parent shape | Collapses when… |
|---|---|
| `PROJECTION` / `FILTER` / `DISTINCT` / `WINDOW` / `GET` / `INTERSECT` / `PIVOT` / `CROSS_PRODUCT` | any child is empty |
| `MATERIALIZED_CTE` | the **body** (`children[1]`) is empty, *unless* the **definition** (`children[0]`) `HasSideEffects()` |
| `INNER` / `SEMI` (via `COMPARISON_JOIN`/`ANY_JOIN`/`DELIM_JOIN`/`ASOF_JOIN`) | either child is empty |
| `ANTI` (incl. `EXCEPT`) | left child empty ⇒ whole thing empty; right child empty ⇒ collapses to left child (**except** for `EXCEPT`, which needs its own deduplication and cannot skip straight to a raw left child) |
| `MARK` / `SINGLE` / `LEFT` | **only** left child empty ⇒ whole thing empty — right-side emptiness is deliberately *not* checked, because these join types still emit every left row, `NULL`-padded, when the right side has nothing |
| `RIGHT` / `OUTER` (full) | **not handled at all** on this checkout — a real gap |
| `RIGHT_SEMI` / `RIGHT_ANTI` | can't exist yet at this point in the pipeline (`BUILD_SIDE_PROBE_SIDE`, which creates them, runs 14 positions later) |

`STATISTICS_PROPAGATION` (pass #34) needs to reason about join emptiness
too, but twenty positions too late to lean on this pass, so it carries its
own smaller, parallel version of the same logic
(`StatisticsPropagator::HandleJoinNeverMatches`/`HandleJoinAlwaysMatches`) —
one that *does* cover `RIGHT`/`RIGHT_SEMI`/`RIGHT_ANTI`, since by pass #34
all three can genuinely exist in the tree, but still declines `MARK`/
`SINGLE`/full `OUTER` for the same semantic reasons `EmptyResultPullup`
does.

## 7.10 `PROJECTION_PULLUP` — hoisting projections so `JOIN_ORDER` can see clean join trees

**In one sentence:** `ProjectionPullup` walks the tree looking for a
`LogicalProjection` sitting directly beneath an operator it knows how to
pull through — a join, a filter, a set operation, `DISTINCT`, a CTE
boundary, `PIVOT`, or `COPY TO` — and, whenever doing so provably doesn't
change what the query computes, either dissolves the projection entirely
(if every expression in it is a plain column reference) or physically
relocates it to a higher position in the tree (if some expressions are
genuinely computed, but nothing in between still needs to see the computed
value directly), so that the *un*-projected join structure underneath
becomes visible, as a single connected shape, to whatever pass needs to
reason about that structure next.

### 7.10.1 Pipeline position, and correcting a plausible-but-wrong assumption about what it feeds

It's tempting to assume `PROJECTION_PULLUP` exists to expose predicates for
`FILTER_PUSHDOWN` — "pull the projection out of the way so the filter
underneath becomes pushable" reads as a natural pairing, the same way
`FILTER_PULLUP`/`FILTER_PUSHDOWN` are deliberately sequenced back-to-back in
§7.6.3 of the overview. **That assumption is backwards for this pass.**
Checking the actual, exact pipeline order from `00-overview.md`'s §7.4
(all line numbers are `RunOptimizer` call sites in
`src/optimizer/optimizer.cpp` on this checkout):

```
#6  FILTER_PUSHDOWN         optimizer.cpp:286
...
#15 PROJECTION_PULLUP       optimizer.cpp:351
#16 OUTER_JOIN_SIMPLIFICATION  optimizer.cpp:357
#17 JOIN_ORDER              optimizer.cpp:364
```

`FILTER_PUSHDOWN` is pass #6; `PROJECTION_PULLUP` is pass #15. By the time
`PROJECTION_PULLUP` runs, `FILTER_PUSHDOWN` has already had its one turn and
is not coming back — there is nothing left for `PROJECTION_PULLUP` to
"unlock" for it within this run of the query, per the same one-shot,
no-fixpoint architecture (§7.2 of the overview) that makes `CTE_INLINING`
need to run twice to catch what changes in between. What actually sits
*immediately after* `PROJECTION_PULLUP` is `OUTER_JOIN_SIMPLIFICATION` (#16)
and then `JOIN_ORDER` (#17) — DuckDB's DPhyp-based [20], genuinely cost-based
search over alternative join orders, the one pass in this entire pipeline
that explores more than one candidate plan shape (Chapter 8's full subject).

And `JOIN_ORDER`'s own relation-extraction code confirms, directly, that
this is exactly the pass `PROJECTION_PULLUP` exists to help. The join-order
optimizer's `RelationManager` decides which operator types count as an
opaque, atomic "relation" — a leaf it will *not* look inside when building
its join hypergraph — versus which ones it can see straight through to keep
discovering more base tables to reorder:

```cpp
// src/optimizer/join_order/relation_manager.cpp:154
static bool OperatorNeedsRelation(LogicalOperatorType op_type) {
	switch (op_type) {
	case LogicalOperatorType::LOGICAL_PROJECTION:
	case LogicalOperatorType::LOGICAL_EXPRESSION_GET:
	case LogicalOperatorType::LOGICAL_CHUNK_GET:
	case LogicalOperatorType::LOGICAL_GET:
	case LogicalOperatorType::LOGICAL_UNNEST:
	case LogicalOperatorType::LOGICAL_DELIM_GET:
	case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
	case LogicalOperatorType::LOGICAL_WINDOW:
	case LogicalOperatorType::LOGICAL_SAMPLE:
	case LogicalOperatorType::LOGICAL_SECURE_VIEW:
		return true;
	default:
		return false;
	}
}
```

(`LOGICAL_SECURE_VIEW` is on this list too: to the join-order optimizer, a
secure view is one more opaque relation.)

`LOGICAL_PROJECTION` is on that list. And the handler for it
(`relation_manager.cpp:630`) confirms exactly what "needs a relation" means
in practice: rather than reasoning about whatever join structure sits
*beneath* the projection as part of the same, single top-level search, it
spins up a **separate, independent, nested join-order optimizer**
(`optimizer.CreateChildOptimizer()`) to resolve that entire subtree on its
own, in isolation, and then folds the *result* of that nested search back in
as one single, already-fixed, atomic relation for the outer search to place.
Concretely: for `(SELECT a.a, b.b FROM t1 a JOIN t2 b ON a.a = b.b) AS join12
JOIN t3 ON join12.a = t3.c`, if the projection wrapping `t1 JOIN t2` is still
standing when `JOIN_ORDER` runs, DPhyp's *outer* search over
`{join12-as-a-black-box, t3}` never gets the chance to consider joining
`t3` to `t1` first instead — that specific alternative plan shape simply
isn't in the outer search's space at all, because `t1` and `t2` were never
individually exposed to it; `join12` was already collapsed into a single,
pre-decided relation before the outer search even started. Removing the
projection turns this from a two-relation outer search bolted onto an
independently-solved two-relation inner search into a *single* three-
relation search across `t1`, `t2`, and `t3` together — strictly more of the
plan-shape space is reachable, and DPhyp might genuinely find a cheaper
order that the two-independent-searches version structurally could not.
This is the real mechanical payoff `PROJECTION_PULLUP` exists to deliver,
and it's a `JOIN_ORDER`-side benefit specifically, not a `FILTER_PUSHDOWN`
one — the two-position gap between #15 and #17 in the fixed sequence is not
an accident of alphabetization; it's `PROJECTION_PULLUP` clearing the way
immediately before the one pass in the whole pipeline that actually needs
the join tree to look this clean.

### 7.10.2 Source walkthrough: class shape and the `parents` stack

`ProjectionPullup` (`src/optimizer/projection_pullup.cpp`,
`src/include/duckdb/optimizer/projection_pullup.hpp`) is a
`LogicalOperatorVisitor` subclass that, unlike `EmptyResultPullup`, does
lean on real per-instance state: a reference to `optimizer` (needed to call
`GenerateTableIndex()` when synthesizing a brand-new projection node), a
reference to `root` — the *entire* plan's root, not just the current
subtree, because pulling a projection all the way up can require rewiring
bindings anywhere between the projection's original position and the very
top of the tree — and a `parents` stack: the chain of ancestor operators the
current recursive descent has passed through *that this pass knows how to
pull a projection through*, in root-to-here order.

```cpp
// src/include/duckdb/optimizer/projection_pullup.hpp:12
class ProjectionPullup : public LogicalOperatorVisitor {
public:
	explicit ProjectionPullup(Optimizer &optimizer_p, unique_ptr<LogicalOperator> &root)
	    : optimizer(optimizer_p), root(root) {}
	void Optimize(unique_ptr<LogicalOperator> &op);
	void VisitOperator(unique_ptr<LogicalOperator> &op) override;
private:
	Optimizer &optimizer;
	unique_ptr<LogicalOperator> &root;
	vector<reference<LogicalOperator>> parents;
	...
};
```

The dispatch inside `VisitOperator` (`projection_pullup.cpp:247`) sorts
every operator type into one of four buckets:

**1. Set operations (`UNION`, `INTERSECT`, `EXCEPT`) — force a canonical
projection immediately below each child, rather than pulling one up.**

```cpp
// src/optimizer/projection_pullup.cpp:252
case LogicalOperatorType::LOGICAL_INTERSECT:
case LogicalOperatorType::LOGICAL_EXCEPT:
case LogicalOperatorType::LOGICAL_UNION: {
	for (auto &child : op->children) {
		InsertProjectionBelowOp(op, child, true);
	}
	return;
}
```

These three depend on column *order*, not column identity, to line their
two inputs up — `InsertProjectionBelowOp` (`projection_pullup.cpp:41`)
checks whether a child is already a plain `LogicalProjection` and, if not,
synthesizes one (a fresh `LogicalProjection` wrapping a full set of plain
column-ref expressions over the child's current output, with a
freshly-generated table index via `optimizer.binder.GenerateTableIndex()`;
the new node is built by the shared helper
`LogicalProjection::CreateIdentity(proj_index, std::move(child))`), so that both sides of the set operation present a stable, canonical column
list directly beneath the set operation itself — the opposite direction
from what this pass usually does, but for the same underlying reason
(giving whatever runs next, and the physical set-operation operator itself,
a clean, canonical shape to reason about rather than an arbitrary one).

**2. `DISTINCT`, `RECURSIVE_CTE`, `MATERIALIZED_CTE`, `CTE_REF`,
`COPY_TO_FILE`, `PIVOT` — the same forced-barrier treatment, with a
different internal rewiring mode (`stop_at_op=false`).** Mechanically
identical in intent to the set-operation case; these are operators whose own
semantics (deduplication, recursion, materialization boundary, reshaping)
depend on seeing a stable column list directly below them too.

**3. Joins — the actual pull-through logic, with a `MARK`/`SINGLE` bail-out
and a `SEMI`/`ANTI` asymmetry.**

```cpp
// src/optimizer/projection_pullup.cpp:271
case LogicalOperatorType::LOGICAL_ANY_JOIN:
case LogicalOperatorType::LOGICAL_COMPARISON_JOIN: {
	auto &comp_join = op->Cast<LogicalComparisonJoin>();
	if (comp_join.join_type == JoinType::MARK || comp_join.join_type == JoinType::SINGLE) {
		break; // bail
	}
	parents.push_back(*op);
	if (comp_join.join_type == JoinType::SEMI || comp_join.join_type == JoinType::ANTI) {
		VisitChildOfOperatorWithProjectionMap(comp_join.children[0], comp_join.left_projection_map);
		InsertProjectionBelowOp(op, comp_join.children[1], false);
	} else {
		VisitChildOfOperatorWithProjectionMap(comp_join.children[0], comp_join.left_projection_map);
		VisitChildOfOperatorWithProjectionMap(comp_join.children[1], comp_join.right_projection_map);
	}
	PopParents(*op);
	return;
}
```

`MARK`/`SINGLE` joins bail out of this whole case immediately (`break`
drops through to the generic tail at the bottom of `VisitOperator`, which
just recurses into children with a *fresh*, empty `ProjectionPullup`
instance and no live `parents` stack) — this pass makes no attempt to pull a
projection through either of these two join types at all. For `SEMI`/
`ANTI`, the left side is visited normally (pushed onto `parents`, eligible
for pull-through exactly like any other join), but the *right* side gets
`InsertProjectionBelowOp(op, comp_join.children[1], false)` instead of a
normal visit — the same forced-barrier treatment `UNION`/`DISTINCT` get,
not an attempt at pull-through. This asymmetry has a direct semantic reason:
a `SEMI`/`ANTI` join's right side contributes *zero* columns to the join's
own output (both join types only ever emit left-side columns — that's the
entire point of an existence check versus a real join), so there is nothing
to gain from exposing whatever projection sits there, and forcibly keeping a
canonical projection boundary there gives `JOIN_ORDER`'s relation-set
bookkeeping (which, per Chapter 8, treats a `SEMI`/`ANTI` join's right side
as something that must stay tied to the exact relation set it was originally
paired with — not freely reorderable the way plain `INNER` joins are) a
clean, fixed scope for that subtree. The test suite's own comment on this
exact behavior (§7.10.4 below) puts it precisely: "the projection on the RHS
of a semi join is not removed, but the plan is optimized deeper" — the
barrier projection itself stays, but `ProjectionPullup` still recurses
*inside* it (`next.Optimize(child->children[0])`), continuing to look for
pull-up opportunities within that self-contained subtree.

**4. `FILTER` — a normal pull-through boundary**, pushed onto `parents` the
same way a join is, respecting whatever `projection_map` the filter itself
already carries via `VisitChildOfOperatorWithProjectionMap`.

### 7.10.3 The actual pull-up decision, inside the `LOGICAL_PROJECTION` case

This is where the real work happens (`projection_pullup.cpp:306`):

```cpp
case LogicalOperatorType::LOGICAL_PROJECTION: {
	auto &proj = op->Cast<LogicalProjection>();
	auto proj_bindings = proj.GetColumnBindings();
	bool all_column_refs = true;
	column_binding_map_t<unique_ptr<Expression>> projection_map;
	for (idx_t i = 0; i < proj.expressions.size(); i++) {
		projection_map[proj_bindings[i]] = proj.expressions[i]->Copy();
		if (proj.expressions[i]->GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
			all_column_refs = false;
		}
		if (proj.expressions[i]->IsVolatile() || proj.expressions[i]->CanThrow()) {
			ProjectionPullup next(optimizer, root);
			next.Optimize(proj.children[0]);
			return; // bail
		}
	}
	bool can_pull_through = true;
	idx_t pull_up_to_here = parents.size();
	CanPullThrough(projection_map, HasJoinBelow(proj), can_pull_through);
	if (!can_pull_through) {
		ProjectionPullup next(optimizer, root);
		next.Optimize(proj.children[0]);
		return;
	}
	if (pull_up_to_here > 0) {
		if (op->children[0]->type == LogicalOperatorType::LOGICAL_UNNEST) {
			parents.push_back(*op);
			Optimize(op->children[0]);
			PopParents(*op);
			return;
		}
		if (all_column_refs) {
			PullUpColrefProjection(op, proj, proj_bindings);
			return;
		}
		PullUpNonColrefProjection(op, proj, proj_bindings, pull_up_to_here);
		return;
	}
	Optimize(op->children[0]);
	PopParents(*op);
	return;
}
```

Working through the guards in the order they run:

- **Volatile or throwing expressions bail out immediately** — a projection
  containing `random()`, or a cast that might error, is left exactly where
  it is, and the recursion just continues into its child fresh. Pulling a
  volatile expression to a different point in the tree could change how
  many times it's evaluated relative to other operators around it (the same
  concern `WINDOW_SELF_JOIN`'s `VolatileExpressionCounter` guard exists for,
  per §7.4's pass #14 in the overview); pulling a throwing one past an
  operator that used to filter out the rows that would have triggered the
  error is exactly the same category of hazard `REORDER_FILTER`'s
  `CanThrow()` guard exists for (§7.6.1) — and, unlike `EMPTY_RESULT_PULLUP`
  in §7.9.5 above, this pass *does* carry that guard.

- **`CanPullThrough` — the "partial pullup is intentionally not
  implemented" check.** This walks the live `parents` stack from the
  *closest* ancestor outward
  (`for (idx_t i = parents.size(); i > 0; i--)`), and at each level,
  enumerates that ancestor operator's *own* expressions (a join's
  condition, a filter's predicate) looking for any reference back to one of
  this projection's output bindings. If that reference is to anything other
  than a plain, unmodified column ref — i.e., the ancestor is consuming the
  *computed value itself*, not just passing a column through — pull-through
  is refused, for every level from that point outward, and the loop stops
  early. The source is explicit about why this is an all-or-nothing
  decision rather than "pull it up as far as it validly can go, then stop":
  *"Partial pullup is intentionally not implemented. Pulling a projection
  only partially up could leave it in an intermediate state between
  operators. This would reduce the opportunities for join reordering
  without providing any benefit."* — a projection stranded halfway up the
  tree is exactly the kind of obstacle §7.10.1 just showed `JOIN_ORDER`
  can't see through, so a pullup that doesn't clear the projection *all the
  way* out from between two joinable relations wouldn't actually deliver the
  benefit this pass exists for.

- **A throwing filter above keeps the projection in place.**

  `CanPullThrough` takes a flag, `keep_throwing_filters_above`. The call site
  passes `HasJoinBelow(proj)`.

  ```cpp
  // src/optimizer/projection_pullup.cpp:179
  static bool HasJoinBelow(const LogicalOperator &op) {
  	reference<const LogicalOperator> current = op;
  	while (current.get().children.size() == 1) {
  		current = *current.get().children[0];
  	}
  	return current.get().children.size() > 1;
  }

  // Filter pushdown keeps throwing expressions above projections. Removing a projection over a join would let the join
  // order optimizer evaluate them before the joins that remove rows, also when they reference other relations.
  static bool IsThrowingProjectionFilter(const Expression &expr,
                                         const column_binding_map_t<unique_ptr<Expression>> &projection_map) {
  	if (!expr.CanThrow()) {
  		return false;
  	}
  	bool references_projection = false;
  	ExpressionIterator::VisitExpression<BoundColumnRefExpression>(expr, [&](const BoundColumnRefExpression &colref) {
  		if (projection_map.find(colref.Binding()) != projection_map.end()) {
  			references_projection = true;
  		}
  	});
  	return references_projection;
  }
  ```

  Inside `CanPullThrough`, before the column-ref check, each ancestor
  expression is tested first:

  ```cpp
  // src/optimizer/projection_pullup.cpp:212
  LogicalOperatorVisitor::EnumerateExpressions(parent_op, [&](unique_ptr<Expression> *expr) {
  	if (keep_throwing_filters_above && IsThrowingProjectionFilter(**expr, projection_map)) {
  		can_pull_through = false;
  		return;
  	}
  ```

  In plain words: `HasJoinBelow` follows single-child operators down from the
  projection. It answers "is there a join (an operator with two or more
  children) somewhere under this projection?". If yes, and some ancestor
  expression *can throw* (for example an integer `+` that can overflow, or
  `//` that can divide by zero) and reads at least one of the projection's
  columns, the projection stays where it is.

  Why? Filter pushdown (pass #6) never pushes a throwing filter down through
  a projection. That rule keeps the filter from running on rows that a join
  below would have removed. But the projection was the only thing holding
  the filter up. If this pass removed it, `JOIN_ORDER` (pass #17) would see
  one flat join tree and could push the filter below the join after all. That
  would have two bad effects:

  1. **A new error.** A view `v` has an `ANTI` join that removes a row whose
     columns overflow when added. `SELECT count(*) FROM v WHERE a + b > 5`
     would raise an overflow error, because `a + b` would be computed before
     the `ANTI` join.
  2. **Lost sharing.** When a view is used twice and only one use has such a
     filter, the filter would end up inside only one copy. `COMMON_SUBPLAN`
     would then not see two equal copies, so nothing would be materialized
     once and shared.

  Two tests cover this. `test/optimizer/pullup_join_expression_errors.test`
  checks the error case with real rows:

  ```sql
  # Throwing filter referencing another relation must remain above a row-eliminating join
  query I
  SELECT count(*)
  FROM (
  	SELECT a.k, a.v
  	FROM (VALUES (1, 1), (2, 1)) a(k, v)
  	JOIN (VALUES (0), (0), (0), (0)) b(q) ON a.k % 2 = b.q
  ) mixed_v
  JOIN (VALUES (1, 0), (2, 1)) c(k, x) USING (k)
  WHERE v // x > 0;
  ----
  4
  ```

  The inner join keeps only `k = 2` (because `2 % 2 = 0`), four times. For
  `k = 2`, `x = 1`, so `v // x` is `1 // 1 = 1`. The row `k = 1` has `x = 0`,
  and `1 // 0` would throw a division-by-zero error. The inner join removes
  that row first, so the right answer is `4` with no error. Note that the
  filter reads `x` from `c`, a different relation; the check only needs the
  expression to read *one* column of the projection (`v`).

  `test/optimizer/common_subplan_projection_map.test` checks the sharing case.
  It builds `CREATE VIEW orders AS SELECT o.* FROM snap_order o ANTI JOIN
  del_order d USING (order_id)` and joins `orders` with an aggregate over
  `orders` that has `WHERE customer_id + order_id > 5`. The expected physical
  plan must match `<REGEX>:.*__common_subplan.*CTE Scan.*CTE Scan.*`, that is,
  one shared subplan read by two CTE scans.

- **The `UNNEST` guard** — a projection sitting directly above
  `LOGICAL_UNNEST` is deliberately never removed, even when it otherwise
  qualifies, with the reasoning inline in the source: that projection is
  what extracts only the specific fields a query actually needs out of an
  unnested struct; removing it would force every operator between the
  unnest and wherever the fields are finally consumed to carry the *entire*
  struct instead, which the comment names directly as a memory blowup risk.

- **`all_column_refs` → `PullUpColrefProjection`: the cheap path, genuine
  deletion.** When every expression in the projection is a plain column
  reference, there's no computation being relocated at all — the projection
  is pure renaming/reordering, and it can simply be *deleted*, with every
  binding it produced rewritten (via `ColumnBindingReplacer`) to point
  straight at the underlying binding it was forwarding. One subtlety is
  carved out with its own inline comment and a concrete, named regression:

  ```cpp
  // src/optimizer/projection_pullup.cpp:73
  // LOGICAL_DISTINCT sets `everything_referenced = true` in RemoveUnusedColumns
  // for its subtree. The projection above it acts as a binding barrier; removing
  // it lets upstream references point past DISTINCT and breaks column pruning
  // down to READ_PARQUET. Repro: TPC-DS Q54 regresses 4-5x without this guard.
  if (proj.children[0]->type == LogicalOperatorType::LOGICAL_DISTINCT) {
  	ProjectionPullup next(optimizer, root);
  	next.Optimize(proj.children[0]);
  	return;
  }
  ```

  `UNUSED_COLUMNS`' underlying machinery (`RemoveUnusedColumns`, pass #23/#38
  in the overview's §7.4) marks a `DISTINCT`'s entire subtree as
  `everything_referenced = true` — meaning it deliberately switches *off*
  its usual "prune whatever nothing above reads" logic below a `DISTINCT`,
  because a correct `DISTINCT` needs every one of its input columns intact
  to compute the right notion of "duplicate row," not just the columns some
  distant ancestor happens to still read. The projection sitting directly
  *above* the `DISTINCT` is what narrows the column set back down to only
  what's actually needed once the `DISTINCT` itself is done — it's the
  boundary past which "everything" stops being forced. Deleting that
  projection (which is exactly what the ordinary colref pullup path would
  otherwise do unconditionally) removes that boundary, letting the
  `everything_referenced` contamination leak *upward* past where it should
  have stopped, all the way down to wherever the actual base-table scan is —
  in the regression this comment documents by name, a Parquet reader that
  now has to materialize every column instead of only the ones genuinely
  needed, a measured 4–5× slowdown on TPC-DS query 54. This is the clearest
  single piece of evidence in either pass's source that "pull things up
  whenever legal" is not this pass's actual goal — it is deliberately
  *conservative* wherever pulling something up would defeat a different
  pass's own column-pruning boundary elsewhere in the pipeline.

- **Otherwise → `PullUpNonColrefProjection`: relocate the whole projection
  node, and the `LEFT`/`RIGHT`/`OUTER` guard.** When some expressions are
  genuinely computed, the projection *node itself* is physically moved —
  spliced out from its current position and re-inserted `pull_up_to_here`
  levels higher, at `insert_at_node` — with `ColumnBindingReplacer` used to
  keep every intervening operator's references consistent with the move.
  Before doing any of that, though, the function walks the same `parents`
  stack one more time with a narrower, join-type-specific question:

  ```cpp
  // src/optimizer/projection_pullup.cpp:106
  // Do not pull non-colref expressions through outer joins.
  // non-colref expressions on the nullable side of a LEFT/RIGHT/OUTER JOIN must not be pulled above the
  // join. If pulled up, expressions (e.g COALESCE) evaluate after the join and return non-null for
  // unmatched rows instead of null.
  if (parent_op.type == LogicalOperatorType::LOGICAL_COMPARISON_JOIN ||
      parent_op.type == LogicalOperatorType::LOGICAL_ANY_JOIN) {
  	auto &join = parent_op.Cast<LogicalComparisonJoin>();
  	if (join.join_type == JoinType::LEFT || join.join_type == JoinType::RIGHT ||
  	    join.join_type == JoinType::OUTER) {
  		ProjectionPullup next(optimizer, root);
  		next.Optimize(proj.children[0]);
  		return;
  	}
  }
  ```

  `test/optimizer/projection_pullup.test` proves exactly this scenario with
  real rows, and it's worth working through precisely because the direction
  of the bug this guard prevents is easy to get backwards:

  ```sql
  CREATE TABLE tbl1 AS SELECT range AS a FROM range(3);   -- a = 0, 1, 2
  CREATE TABLE tbl2 AS SELECT range AS b FROM range(1);   -- b = 0
  CREATE VIEW v AS SELECT b, COALESCE(b, 0) AS c FROM tbl2;

  SELECT a, v.c FROM tbl1 LEFT JOIN v ON tbl1.a = v.b;
  ----
  0	0
  1	NULL
  2	NULL
  ```

  `v.c` is `COALESCE(b, 0)` — a genuinely computed, non-colref expression,
  sitting inside a view that becomes the *right*, non-preserved side of a
  `LEFT JOIN`. Rows `a=1` and `a=2` have no matching `v.b`, so a correct
  `LEFT JOIN` must `NULL`-pad *every* column `v` would have contributed —
  including `v.c` — since there is, semantically, no row of `v` for those
  two `tbl1` rows to have run `COALESCE` against at all. The expected output
  reflects exactly that: `NULL`, not `0`. If `PullUpNonColrefProjection` were
  allowed to relocate the `COALESCE(b, 0)` expression itself to sit *above*
  the join (rather than leaving it exactly where it is, below the join,
  computed once per real row of `v` before the join's own `NULL`-padding
  logic ever runs), the join would instead need to carry the *raw* column
  `b` through, `NULL`-pad *that* correctly for the two non-matching rows,
  and only then apply `COALESCE(b, 0)` on the other side of the join — at
  which point `COALESCE(NULL, 0)` evaluates to `0`, not `NULL`, silently
  turning "no matching row existed at all" into "a matching row existed with
  `b = NULL`," which is a different, wrong answer. This is precisely what
  the comment names — pulled-up expressions "return non-null for unmatched
  rows instead of null" — and it's the mechanical reason this guard exists
  specifically for the *non-colref* path: a plain column-ref pullup
  (`PullUpColrefProjection`) never has this problem, because it never moves
  any computation across the join boundary at all — it only ever renames
  bindings, and a renamed reference to a `NULL`-padded column is still
  correctly `NULL`.

### 7.10.4 Every case in `test/optimizer/projection_pullup.test`, walked through

The file sets `PRAGMA explain_output = OPTIMIZED_ONLY` and disables four
*other* passes that would otherwise introduce their own projections and
confuse the `EXPLAIN` pattern matches (`common_subplan`, `column_lifetime`,
`compressed_materialization`, `statistics_propagation`) — a reminder that
isolating one pass's visible effect in `EXPLAIN` output often means
explicitly quieting its neighbors, exactly as §7.3 of the overview describes
`disabled_optimizers` being used for.

**Cannot pull up — a non-colref output consumed directly by the join
condition.**

```sql
EXPLAIN SELECT * FROM (SELECT a + 1 as x, b FROM tbl) subq JOIN t1 ON subq.x = t1.a;
----
logical_opt	<REGEX>:.*JOIN.*PROJECTION.*
```

`subq.x` (`= a + 1`, non-colref) is referenced directly by the join's own
condition, `subq.x = t1.a` — exactly the pattern `CanPullThrough` exists to
catch: the join's expression list references a projection output that isn't
a plain colref, so pull-through is refused outright, and the projection
stays put, directly beneath the `JOIN` in the optimized plan.

```sql
EXPLAIN SELECT a, SUM(b) AS sum_b FROM (SELECT a, b FROM tbl WHERE b > 0) GROUP BY a;
----
logical_opt	<REGEX>:.*PROJECTION.*AGGREGATE.*
```

This one is refused for a different, more categorical reason:
`LOGICAL_AGGREGATE_AND_GROUP_BY` does not appear *anywhere* in
`ProjectionPullup::VisitOperator`'s dispatch — not among the join cases, not
among the set-operation/`DISTINCT`/CTE cases, nowhere. It falls straight to
the generic tail at the very bottom of the function (a fresh
`ProjectionPullup` recursing into children with no live `parents` entry for
the aggregate at all), which means this pass makes no attempt whatsoever to
pull anything through an aggregate boundary, in either direction — an
aggregate is a hard wall for this pass, not a case that happens to fail a
narrower check.

```sql
EXPLAIN SELECT * FROM (SELECT CASE WHEN a > 5 THEN a ELSE 0 END as x FROM t1) subq JOIN t2 ON subq.x = t2.b;
----
logical_opt	<REGEX>:.*JOIN.*PROJECTION.*

EXPLAIN SELECT * FROM (SELECT a.a + 1 AS newA, b.b + 2 AS newB, a.a +3 as newAA, b.b + 4 as newBB FROM t1 a JOIN t2 b ON a.a = b.b) AS join12
JOIN (SELECT c.c + 3 AS newC, d.d + 4 AS newD FROM t3 c JOIN t4 d ON c.c = d.d) AS join34 ON join12.newA = join34.newC;
----
logical_opt	<REGEX>:.*JOIN.*PROJECTION.*
```

Both are the same shape as the first case — a `CASE` expression and two
independent `+`-expressions, respectively, each directly consumed by an
outer join's own condition — refused by `CanPullThrough` for the same
reason, on both sides independently in the second query.

**Should pull up — plain column-ref projections, exposing joins directly
to each other.**

```sql
EXPLAIN SELECT * FROM (SELECT a.a, b.b FROM t1 a JOIN t2 b ON a.a = b.b) AS join12
JOIN (SELECT c.c, d.d as newD FROM t3 c JOIN t4 d ON c.c = d.d) AS join34 ON join12.a = join34.c;
----
logical_opt	<!REGEX>:.*JOIN.*PROJECTION.*
```

Both `join12`'s and `join34`'s wrapping projections are pure column refs
(`a.a, b.b` and `c.c, d.d`), so both dissolve via `PullUpColrefProjection`,
leaving `t1 ⋈ t2` and `t3 ⋈ t4` directly adjacent to the outer join with no
projection standing between any of them — exactly the shape §7.10.1
described `JOIN_ORDER` needing, now exposing all four base relations
(`t1`, `t2`, `t3`, `t4`) to a single top-level DPhyp search instead of two
independently pre-solved two-relation searches bolted together afterward.
The next two cases confirm the same dissolution happens correctly with only
one side wrapped:

```sql
EXPLAIN SELECT * FROM (SELECT a.a, b.b FROM t1 a JOIN t2 b ON a.a = b.b) AS join12
JOIN t3 on join12.a = t3.c LIMIT 5;
----
logical_opt	<!REGEX>:.*JOIN.*PROJECTION.*

EXPLAIN SELECT * FROM t1 AS join12 JOIN (SELECT c.c, d.d as newD FROM t3 c JOIN t4 d ON c.c = d.d) AS join34 ON join12.a = join34.c;
----
logical_opt	<!REGEX>:.*JOIN.*PROJECTION.*
```

**Stacked trivial projections collapse to (at most) one, via repeated
re-entry.**

```sql
EXPLAIN SELECT a3 FROM (SELECT a2 AS a3 FROM (SELECT a1 AS a2 FROM (SELECT a AS a1 FROM tbl) AS t0) AS t1) AS t2 WHERE a3 % 2 = 0;
----
logical_opt	<!REGEX>:.*PROJECTION.*PROJECTION.*
```

Three consecutive pure-rename projections nested under a `WHERE` filter.
`FILTER` is itself a pull-through boundary, so the recursion reaches the
innermost projection with a live `parents` entry already pushed for the
filter above it; `PullUpColrefProjection` dissolves one layer and then
explicitly re-enters `Optimize` on what's left (with its own comment: *"Binding
rewrites can make parent projections redundant, and without another pass
they would not be eliminated"*), which is what lets all three stacked
renames collapse down to nothing rather than leaving two of the three
standing after only a single pass over that spot in the tree.

**A CTE reference and a `VALUES`-derived join both dissolve the same way.**

```sql
EXPLAIN with cte_1 as (select * from t1) select * from cte_1 join t2 on a=b limit 3;
----
logical_opt	<!REGEX>:.*JOIN.*PROJECTION.*

EXPLAIN SELECT SUM(v1.i) + SUM(v2.i) AS sum_i, SUM(v1.j) + SUM(v2.j) AS sum_j
FROM (VALUES (1, 10), (2, 5), (3, 4)) AS v1(i, j)
JOIN (VALUES (1, 10), (2, 5), (3, 4)) AS v2(i, j) ON v1.i = v2.i;
----
logical_opt	<!REGEX>:.*AGGREGATION.*JOIN.*PROJECTION.*
```

Whatever trivial colref-only projection the binder attaches to a plain
`SELECT * FROM t1` CTE reference, or to a `VALUES (...) AS v1(i, j)` alias,
dissolves the same way the hand-written column-ref subqueries above did —
exposing the join cleanly beneath the `SUM(...) + SUM(...)` aggregate in the
second case.

**`DISTINCT` and `INTERSECT` are exactly the two documented cases where a
projection deliberately stays.**

```sql
EXPLAIN SELECT DISTINCT a FROM (SELECT a, b FROM (SELECT i AS a, i AS b FROM range(10) t(i)) t1) t2;
----
logical_opt	<REGEX>:.*DISTINCT.*PROJECTION.*

EXPLAIN SELECT * FROM range(5) INTERSECT SELECT * FROM range(5);
----
logical_opt	<REGEX>:.*INTERSECT.*PROJECTION.*
```

The first is precisely the TPC-DS-Q54-guarding special case walked through
in §7.10.3 — the immediate child of the outer `SELECT a`'s projection is a
`LOGICAL_DISTINCT`, so `PullUpColrefProjection` explicitly refuses to
dissolve it, leaving `DISTINCT` directly followed by `PROJECTION` in the
optimized plan on purpose. The second is `LOGICAL_INTERSECT`'s
forced-barrier bucket (§7.10.2's first case) doing exactly what it's
supposed to: `range(5)`'s output gets wrapped in a canonical projection
directly beneath the set operation regardless of whether one was already
there.

**`SEMI` join: LHS dissolves normally, RHS keeps its barrier.**

```sql
EXPLAIN
select * from(select a.a, b.b from (select * from t1) a join (select * from t2) b on a.a = b.b) as join12
semi join(select c.c, d.d as newD from (select * from t3) c join (select * from t4) d on c.c = d.d) as join34 on join12.a = join34.c;
----
logical_opt	<!REGEX>:.*INNER.*PROJECTION.*
```

`join12`, on the semi join's *left* side, is visited exactly like any other
pull-through-eligible child and dissolves the same way it did in the plain-
`JOIN` cases above (exposing `t1 ⋈ t2` directly, hence the check for no
leftover `INNER ... PROJECTION` pattern on that side). `join34`, on the
*right* side, gets the forced-barrier treatment from §7.10.2's join case
instead of a normal visit — its own outer wrapping projection is kept — but
`ProjectionPullup` still recurses *inside* that barrier, which is why the
inner `t3 ⋈ t4` join underneath `join34` is just as cleanly exposed within
its own self-contained scope as `join12`'s is; the comment above this test
in the file states this precisely: the RHS projection isn't removed, but
the plan beneath it is still optimized.

**Real views, non-colref expressions used only as pure passthrough output —
these *do* qualify for pullup, but the pass only decides they can move; a
later pass decides where they finally land.**

```sql
CREATE VIEW v1 AS SELECT i, length(i::VARCHAR) j FROM t1 JOIN t2 using (i);

EXPLAIN SELECT i, j1.j FROM v1 j1 JOIN v1 j2 using (i);
----
logical_opt	<!REGEX>:.*JOIN.*PROJECTION.*

CREATE TABLE pullup_variable(i INTEGER, v VARCHAR);

-- Computed expressions remain eligible for general projection pullup
EXPLAIN SELECT s FROM (SELECT i, substring(v, 1, 20) AS s FROM pullup_variable) p JOIN t2 USING (i);
----
logical_opt	<!REGEX>:.*JOIN.*PROJECTION.*

-- Expression placement is handled separately after statistics propagation
EXPLAIN SELECT n FROM (SELECT i, length(v) AS n FROM pullup_variable) p JOIN t2 USING (i);
----
logical_opt	<!REGEX>:.*JOIN.*PROJECTION.*

-- Variable-size column references can still be removed
EXPLAIN SELECT p.v FROM (SELECT i, v FROM pullup_variable) p JOIN t2 USING (i);
----
logical_opt	<!REGEX>:.*JOIN.*PROJECTION.*
```

In every one of these, the non-colref expression (`length(i::VARCHAR)`,
`substring(v, 1, 20)`, `length(v)`) is joined on a *different*, genuinely
colref column (`i`, via `USING`) and is otherwise only ever read as a plain
passthrough output higher up — nothing in the `parents` stack's own
expression lists references the computed value as anything but a straight
colref, so `CanPullThrough` succeeds and `PullUpNonColrefProjection`
physically relocates each of them above their respective joins. The middle
test's own comment is the important one to take literally: *"Expression
placement is handled separately after statistics propagation."*
`PROJECTION_PULLUP`'s job stops at deciding *whether* and *how far* a
projection may legally rise — it has no cost model and makes no claim about
where the *best* final resting place for a relocated expression is. That
decision belongs to `PROJECTION_PLACEMENT` (pass #35 in the overview's §7.4,
`src/optimizer/projection_placement.cpp`), which runs twenty positions
later, *after* `STATISTICS_PROPAGATION` (#34) has real cardinality estimates
to place a narrowing projection cost-effectively. `PROJECTION_PULLUP` clears
projections out of `JOIN_ORDER`'s way early; `PROJECTION_PLACEMENT` decides,
much later, once real statistics exist, where a projection should
concretely sit.

**A correctness (not merely a shape) test: keeping filter projection maps in
sync across duplicated grouped expressions.**

```sql
SELECT COUNT(*)
FROM pullup_category AS root
LEFT JOIN (
	SELECT product_category_name,
		MIN(DISTINCT address_valuation) AS rp_207,
		MIN(DISTINCT address_valuation) AS rp_208,
		MIN(DISTINCT address_valuation) AS rp_212
	FROM pullup_product, pullup_address
	GROUP BY product_category_name
) AS rp ON root.category_name = rp.product_category_name
WHERE COALESCE(rp.rp_207, root.category_valuation) IN (432, 120, 150, 160, 110, 800, 450, 330);
----
2
```

The comment directly above this test in the file names exactly what it's
guarding: *"Projection pullup has to keep filter projection maps in sync
when pulling up duplicate grouped expressions."* This checks an actual row
count, not merely an `EXPLAIN` pattern, because it's a correctness
regression test — proof that when `ColumnBindingReplacer` rewires bindings
during a pullup, the `LogicalFilter::projection_map` machinery the base
`LogicalOperatorVisitor` uses to track which of a child's columns still
matter after a filter (via `VisitChildOfOperatorWithProjectionMap`,
`RemapProjectionMap`, §7.10.2) stays internally consistent even when the
same underlying expression (`MIN(DISTINCT address_valuation)`) has been
requested three times, under three different output names, feeding a single
downstream `WHERE COALESCE(...) IN (...)` filter.

**The closing correctness loop.**

```sql
foreach do '' 'projection_pullup'
statement ok
set disabled_optimizers to {do}

query III
SELECT * FROM (SELECT a + 1 as x, b FROM tbl) subq JOIN t1 ON subq.x = t1.i ORDER BY 1 LIMIT 3;
----
1	0	1
...
endloop
```

Every structural query the file already demonstrated `EXPLAIN`-shape changes
for gets re-run inside this loop, once with `projection_pullup` fully
enabled and once with it specifically disabled via `disabled_optimizers`
(§7.3 of the overview's mechanism), checking that both runs return
byte-identical rows — the file's own, stronger-than-`EXPLAIN` proof that
every one of the rewrites walked through above changes only the *shape* of
the plan, never the answer.

### 7.10.5 Summary of this pass's boundaries

`PROJECTION_PULLUP`'s legal territory, precisely:

- **Never crosses:** an aggregate boundary (no case at all for
  `LOGICAL_AGGREGATE_AND_GROUP_BY`); a `MARK`/`SINGLE` join (explicit
  `break`); a volatile or throwing expression (explicit bail, recursing
  fresh into the child instead); a non-colref expression that any ancestor
  in the `parents` stack consumes as anything other than a plain colref
  (`CanPullThrough`, with partial pullup explicitly ruled out by design,
  not merely unimplemented); a projection sitting directly above
  `LOGICAL_UNNEST` (explicit guard against forcing every downstream
  operator to carry a whole unnested struct); a non-colref expression on
  the nullable side of a `LEFT`/`RIGHT`/`OUTER` join (explicit guard against
  turning a genuine "no matching row" `NULL` into a computed non-`NULL`
  default); a projection whose immediate child is `LOGICAL_DISTINCT`
  (explicit guard, with a named, measured 4–5× TPC-DS Q54 regression as the
  reason it exists); and an ancestor
  expression that can throw and reads one of the projection's columns, when
  a join sits below the projection (`HasJoinBelow` plus
  `IsThrowingProjectionFilter`, §7.10.3).
- **Forces a canonical projection into place, rather than pulling one out:**
  `UNION`/`INTERSECT`/`EXCEPT` (column-order stability) and `DISTINCT`/
  `RECURSIVE_CTE`/`MATERIALIZED_CTE`/`CTE_REF`/`COPY_TO_FILE`/`PIVOT` (a
  stable column-list boundary at a structurally significant point), plus the
  right-hand side specifically of `SEMI`/`ANTI` joins.
- **Actually pulls through:** `FILTER`, and both sides of every join type
  *except* `MARK`, `SINGLE`, and (on the right only) `SEMI`/`ANTI`,
  dissolving plain colref-only projections outright and physically
  relocating genuinely computed ones when nothing in between still consumes
  them as anything but a passthrough value.
- **What it's actually for:** not exposing predicates for `FILTER_PUSHDOWN`
  (already ran, seven positions earlier, at #6) — exposing clean,
  un-projected join structure for `JOIN_ORDER`'s DPhyp search two positions
  later, at #17, whose own `RelationManager::OperatorNeedsRelation` treats
  any surviving `LOGICAL_PROJECTION` as an opaque relation boundary and
  solves everything beneath it with an independent, already-fixed nested
  search rather than folding it into the same top-level join-reordering
  space.

## 7.11 Closing note: two "pull it up" passes, two very different kinds of upward movement

It's worth naming the contrast between these two passes directly, since they
sit only two positions apart in the fixed sequence and share a verb in their
names but otherwise have almost nothing in common mechanically.
`EMPTY_RESULT_PULLUP` propagates a single, discrete fact — "this subtree
produces zero rows" — through a small, closed set of parent-operator
semantics, and it does so *reactively*: it never proves anything itself,
it only asks, at each level, whether an already-proven fact from below still
holds true one level up, with the entire question resting on getting each
join type's own asymmetric semantics (INNER/SEMI symmetric, ANTI
asymmetric-but-collapsible, LEFT/SINGLE/MARK asymmetric-and-*not*-
collapsible on the wrong side) exactly right — a boundary this pass has, on
this checkout, both a git-documented history of getting wrong and fixing
(the `ANTI`/`EXCEPT` TODO, the `MATERIALIZED_CTE` side-effect bug, the
`DELIM_JOIN` gap this section's fuzzer test caught) and at least one
still-open gap (`RIGHT`/full `OUTER` never handled at all). `PROJECTION_PULLUP`
moves something much larger and more structural — an entire operator node,
along with every downstream reference to its output — and does so
*prospectively*, clearing operators out of the way specifically so that a
later, more consequential pass (`JOIN_ORDER`, two positions on) inherits a
join tree unobstructed by projection nodes it would otherwise have to treat
as opaque, pre-solved leaves. Both passes are small by line count — 116 and
376 lines respectively — and both are exactly the kind of narrow,
single-purpose "specialist" the overview's §7.2 says this whole
architecture is built from: each one very good at exactly one thing, run
once, in a fixed place, and never revisited within the same query.
