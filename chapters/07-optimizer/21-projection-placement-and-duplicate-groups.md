# 7.21 — `PROJECTION_PLACEMENT` and `DUPLICATE_GROUPS`: two late-pipeline cleanup specialists

*Sources: `src/optimizer/projection_placement.cpp` (+
`src/include/duckdb/optimizer/projection_placement.hpp`);
`src/optimizer/remove_duplicate_groups.cpp` (+
`src/include/duckdb/optimizer/remove_duplicate_groups.hpp`). Passes #35 and
#24 in §7.4's pipeline listing respectively — `OptimizerType::PROJECTION_PLACEMENT`
(`optimizer.cpp:493`) and `OptimizerType::DUPLICATE_GROUPS`
(`optimizer.cpp:407`). Tests: `test/optimizer/projection_placement.test`,
`test/optimizer/projection_placement_results.test`,
`test/sql/optimizer/test_duplicate_groups_optimizer.test`.*

Both passes covered in this file share a family resemblance that's worth
naming up front, because it's easy to miss if you read them in isolation:
neither one is a *general* rewrite rule discovered by pattern-matching the
SQL text. Both are narrow, late-running specialists that exist to clean up
after — or take advantage of — decisions *other* passes already made earlier
in the fixed pipeline (§7.2). `PROJECTION_PLACEMENT` only runs at all because
an earlier pass (`PROJECTION_PULLUP`, #15) indiscriminately relocated
computed expressions to a position that later turns out to be a bad one once
real cardinality and width statistics are available (#34, immediately
before). `DUPLICATE_GROUPS` only has anything to do because an earlier pass
(`UNUSED_COLUMNS`, #23, immediately before) rewrites certain join-connected
column references to point at the *same* underlying binding, which can turn
a `GROUP BY` list that started out syntactically distinct into one with a
literal, exploitable duplicate. Neither pass would need to exist if the
optimizer were a single fixpoint loop that could re-examine everything after
every change — but per §7.2, it isn't, so DuckDB instead ships a small,
purpose-built pass whose entire job is "notice the specific mess a specific
earlier pass can leave behind, and clean it up."

One administrative note before the source dive, directly following up on
this book's own stated test-format conventions (§7's sibling deep-dive files,
e.g. `07-filter-pullup-and-pushdown.md` §"On the same-label idiom," document
the `query I nosort <label>` paired-label convention that proves two
statements produce byte-identical output). **None of the three `.test` files
this chapter walks through use that idiom at all** — a direct check
(`grep -n nosort` across all three) turns up nothing. That's worth stating
rather than silently omitting, since the idiom's *absence* here is itself
informative: `PROJECTION_PLACEMENT`'s own test suite instead uses a
deliberately different two-file split to get an equivalent kind of
guarantee — covered in full in §21.1.6 below, with the git history to prove
the split was there from the very first commit, not an afterthought.

---

## Part I — `PROJECTION_PLACEMENT`: deciding *where*, not *whether*, to compute an expression

### 21.1.1 What this pass is, in one sentence

`PROJECTION_PLACEMENT` takes a computed expression that already sits
*somewhere* in a `LogicalProjection` — it never invents a projection or a
computation that wasn't already going to happen — and relocates *only its
position in the plan tree*, up or down across intervening filters,
pass-through projections, and inner joins, to whichever position on its
dependency path minimizes an explicit, statistics-driven byte-transport-plus-CPU
cost model; it never changes what value the expression computes, only when
and where, relative to the joins around it, that value gets computed.

### 21.1.2 Why this needs its own pass at all: what `PROJECTION_PULLUP` already did to the plan

It's tempting to assume the binder's own plan — Query written as
`(SELECT i, substring(v, 1, 20) AS s FROM dimension) JOIN fact USING (i)` —
already puts `substring(v, 1, 20)` in the "obviously right" place, directly
below the join, exactly where the SQL subquery syntax put it. And it does,
at the moment the binder hands the plan to the optimizer. But §7.4's
pipeline listing already flagged pass #15, `PROJECTION_PULLUP`
(`src/optimizer/projection_pullup.cpp`), whose entire job — quoting its own
inline comment, reproduced in the overview — is *"Pull up projection from
joins"*: hoisting a `LogicalProjection` sitting directly under a join up
above it, unconditionally, specifically to simplify the tree that pass #17's
join-order search (`JOIN_ORDER`, Chapter 8) has to reason about. That pass
runs with **no cost information about the expression it's hoisting at
all** — it can't have any, because real cardinality and width statistics
aren't available yet at position #15; those only show up at
`STATISTICS_PROPAGATION`, pass #34, nineteen positions later.

So by the time real numbers exist, a computed expression that started life
tucked safely below a join, on the narrow, pre-fanout side, can easily be
sitting *above* the join instead — a structurally correct but potentially
much more expensive position, because everything the expression depends on
now has to be carried, uncomputed, across the join, and the expression
itself may now run once per *post*-join row rather than once per pre-join
row. `PROJECTION_PLACEMENT` is placed at pass #35 — deliberately the very
next structural decision after `STATISTICS_PROPAGATION` (#34) is done, and
explicitly gated on it having actually run (`optimizer.cpp:493`: *"only runs
if statistics were actually propagated"*, per §7.4) — precisely because it
is the *only* point in the fixed pipeline where the optimizer both knows
real numbers and still has a chance to undo an earlier, cost-blind
structural decision. This is the concrete answer to "what does this pass
decide": not *whether* an expression is worth computing (it was always going
to be computed — the SQL asked for it), but *where*, among the finitely many
positions its dependency chain of filters/projections/inner-joins actually
allows, it is cheapest to compute it.

### 21.1.3 The cost primitives: byte width, not just row count

Every decision this pass makes bottoms out in one quantity: an estimated
number of **bytes transported and computed**, not just rows. This is worth
foregrounding because it's the single fact that makes the pass's behavior
predictable rather than mysterious — the same expression, over the same
join, gets placed in opposite positions purely because the *ratio* of input
width to output width, or the *direction* of the join's fanout, changed.

**`GetExpressionWidth`** (`projection_placement.cpp:125`) collects every
distinct, depth-0 `BoundColumnRefExpression` the candidate expression reads,
and sums a per-binding byte width via **`GetBindingWidth`**
(`projection_placement.cpp:104`):

```cpp
// src/optimizer/projection_placement.cpp:104
static bool GetBindingWidth(const ColumnBinding &binding, const LogicalType &type,
                            const column_binding_map_t<unique_ptr<BaseStatistics>> &statistics_map, idx_t &width) {
	auto physical_type = type.InternalType();
	if (TypeIsConstantSize(physical_type)) {
		width = GetTypeIdSize(physical_type);
		return true;
	}
	if (physical_type != PhysicalType::VARCHAR) {
		return false;
	}
	// Variable-width values are only costed when statistics provide a safe upper bound.
	auto entry = statistics_map.find(binding);
	if (entry == statistics_map.end() || !entry->second ||
	    entry->second->GetStatsType() != StatisticsType::STRING_STATS ||
	    !StringStats::HasMaxStringLength(*entry->second)) {
		return false;
	}
	width = GetTypeIdSize(physical_type) + StringStats::MaxStringLength(*entry->second);
	return true;
}
```

A fixed-size physical type (`INTEGER`, `BIGINT`, `DATE`, …) always gets a
width — `GetTypeIdSize` is exact and unconditional. A `VARCHAR`, by contrast,
only gets a width **if `STATISTICS_PROPAGATION` left behind a known maximum
string length** for that specific binding (`StringStats::HasMaxStringLength`).
Any other variable-width physical type (`LIST`, `STRUCT`, …) is refused
outright — `GetBindingWidth` returns `false`, and the whole candidate is
dropped for this expression, because there's no safe upper bound to cost
against. This is exactly the mechanism behind the test comment "Unknown
variable-width results stay above the join" for `lower(v)`: `lower` doesn't
provably preserve a known maximum length the way `substring(v, 1, 20)`'s
*constant* length bound does, so no width — and hence no relocation
decision at all — is possible for it.

**`GetExclusiveInputWidth`** (`projection_placement.cpp:174`) then leaves
out any dependency binding that some **other** expression in the same
projection also reads. Moving one expression only saves the columns that
nothing else still needs. This is the "not credited as a saving when
selected independently" test: `SELECT d.v, n FROM (SELECT i, v, length(v) AS
n FROM placement_wide) d JOIN placement_dimension USING (i)` never moves.
`d.v` is selected on its own, so the wide `v` column must cross the join
whatever happens to `length(v)`.

The count of readers comes from a helper, `CountReadingExpressions`
(`projection_placement.cpp:159`). It walks the projection once and counts,
for each binding, how many expressions read it. An expression that reads
the same column twice (for example `v || v`) still counts once:

```cpp
// src/optimizer/projection_placement.cpp:158
//! The number of expressions of the projection that read each binding
static column_binding_map_t<idx_t> CountReadingExpressions(const LogicalProjection &projection) {
	column_binding_map_t<idx_t> result;
	for (auto &expression : projection.expressions) {
		column_binding_set_t read_bindings;
		ExpressionIterator::VisitExpression<BoundColumnRefExpression>(
		    *expression, [&](const BoundColumnRefExpression &colref) {
			    if (colref.Depth() == 0 && read_bindings.insert(colref.Binding()).second) {
				    result[colref.Binding()]++;
			    }
		    });
	}
	return result;
}

//! The width of the bindings that no other expression of the projection reads
static idx_t GetExclusiveInputWidth(const column_binding_map_t<idx_t> &reading_expressions,
                                    const vector<ExpressionBindingWidth> &binding_widths) {
	idx_t result = 0;
	for (auto &binding_width : binding_widths) {
		auto entry = reading_expressions.find(binding_width.binding);
		if (entry != reading_expressions.end() && entry->second > 1) {
			continue;
		}
		result += binding_width.width;
	}
	return result;
}
```

The candidate expression itself is one of the readers. So a count above 1
means "someone else reads this column too", and the column's width is not
counted as a saving.

The reader counts come from a single counting pass over the projection.
`LiftFirstProfitableExpression` and `OptimizeOperator` build the map once per
projection and pass it to `FindLiftPlacement` and `FindPlacement`. So the
work is one walk per projection, not one walk per (expression, binding)
pair.

**`MINIMUM_COST_RATIO = 0.75`** (`projection_placement.cpp:22`) is the
hysteresis threshold: a candidate position must beat the current position's
estimated cost by at least 25% before the rewrite fires at all. Given every
input to this cost model is itself an *estimate* (propagated statistics,
cardinality estimates from Chapter 8's join-order search), this guards
against the pass thrashing the plan for a marginal, estimation-noise-sized
"win."

**`IsEligibleJoin`** (`projection_placement.cpp:227`) is the barrier that
determines which joins an expression is even allowed to cross:

```cpp
// src/optimizer/projection_placement.cpp:227
static bool IsEligibleJoin(LogicalOperator &op) {
	if (op.type != LogicalOperatorType::LOGICAL_COMPARISON_JOIN && op.type != LogicalOperatorType::LOGICAL_ANY_JOIN) {
		return false;
	}
	return op.Cast<LogicalJoin>().join_type == JoinType::INNER;
}
```

Only `INNER` joins qualify — every other join type (`LEFT`, `RIGHT`,
`FULL`, `SEMI`, `ANTI`, `MARK`, `SINGLE`) is a hard placement barrier, and
every one of those is directly exercised as a negative test in §21.1.4.
`LOGICAL_ANY_JOIN` (a join whose condition isn't a simple equality — an
`OR`-connected or otherwise non-comparison condition) is explicitly included
as long as its `join_type` is still `INNER`, which is exactly why
"Arbitrary inner joins use the same single-child placement rule" is one of
the test cases below.

### 21.1.4 The correctness guards, before any cost math runs

Both `FindLiftPlacement` (`projection_placement.cpp:285`) and `FindPlacement`
(`projection_placement.cpp:763`) open with the identical gate:

```cpp
if (expression.GetExpressionClass() == ExpressionClass::BOUND_COLUMN_REF || expression.IsVolatile() ||
    expression.CanThrow() || !projection.children[0]->has_estimated_cardinality) {
	return false;
}
```

- A bare column reference is skipped outright — there's nothing to
  "compute," so there's no placement decision to make.
- **`IsVolatile()`** — a `random()`, a sequence call, anything whose value
  changes on every evaluation — is a hard barrier, for the obvious reason:
  relocating *where* a volatile expression is evaluated changes *how many
  times* it's evaluated relative to the row counts on either side of a join,
  which changes observable output. The "Volatile expressions are placement
  barriers" test is the direct regression.
- **`CanThrow()`** — a `CAST` that might fail, or any expression that can
  raise an error at runtime — is likewise a hard barrier, for exactly the
  reasoning §7.6.1 already established for `REORDER_FILTER`'s own identical
  guard: moving a possibly-throwing computation across a join that could
  otherwise have filtered out the offending row changes whether the query
  errors at all. The test ("A throwing expression cannot be pulled across a
  filtering join," walked in full in §21.1.7 below) confirms the *error
  message itself* is byte-identical whether `PROJECTION_PLACEMENT` runs or
  `PRAGMA disable_optimizer` is set — the strongest form of "this barrier
  actually holds" a test file can assert.
- **`has_estimated_cardinality`** gates the entire mechanism on the same
  precondition §21.1.2 already explained: no real numbers, no relocation
  decision, full stop.

### 21.1.5 Two phases, two directions: `Lift` and `Placement`

`ProjectionPlacementOptimizer::Optimize` (`projection_placement.cpp:998`)
runs two structurally symmetric but *directionally opposite* sweeps, in
this exact order:

```cpp
// src/optimizer/projection_placement.cpp:998
void ProjectionPlacementOptimizer::Optimize(unique_ptr<LogicalOperator> &plan) {
	vector<PathStep> ancestors;
	while (LiftFirstProfitableExpression(optimizer, statistics_map, plan, plan, ancestors)) {
		plan->ResolveOperatorTypes();
	}
	OptimizeOperator(optimizer, statistics_map, plan);
	plan->ResolveOperatorTypes();
}
```

**Phase 1 — Lift, upward, to a fixed point.** `LiftFirstProfitableExpression`
(`projection_placement.cpp:725`) walks the plan recursively, building up an
`ancestors` path from the root down to each `LogicalProjection` it visits,
and for every expression in that projection asks `FindLiftPlacement`
(`projection_placement.cpp:285`) whether relocating the computation *later*
— higher up the tree, past filters, pass-through projections, and eligible
inner joins — would be cheaper than computing it where it already sits.
Concretely, this is the rewrite that answers: *is it cheaper to carry this
expression's (usually narrow) **input** dependencies across an upcoming
join, and compute the (possibly wide) result only once, after that join,
than to compute the result now and pay to carry the wide result itself
across every intervening operator?* The loop in `Optimize` re-runs this
search from scratch (`while (LiftFirstProfitableExpression(...))`) after
every single successful lift, to a local fixed point — one of the very few
places in this book's optimizer coverage where an individual pass re-fires
itself to convergence *within* its own one turn on the fixed pipeline (§7.2
governs the forty-ish *passes themselves* running exactly once each; nothing
in that architecture prevents one pass's own internal logic from looping).

The clearest illustration is the "selective join" family of test cases in
§21.1.7: `repeat(v, 10)` — an expression that *expands* its input, turning
a bounded 100-byte string into a bounded ~1000-byte one — moves **above** a
selective join (`placement_selective`, which reduces 100 dimension rows down
to far fewer matches), because it's cheaper to let the join discard
non-matching rows first and only expand the *survivors*, rather than
expanding all 100 rows and then discarding most of the (now ten times
wider) expanded results. This is a lift: the expression's evaluation moves
later, past the join, precisely because the join is a *filter* here, not a
fanout.

**Phase 2 — Placement, downward, exhaustively over the whole path.** Once
lifting has converged, `OptimizeOperator` (`projection_placement.cpp:960`)
walks the tree bottom-up and, for every `LogicalProjection`, repeatedly
calls `FindPlacement` (`projection_placement.cpp:763`) for each of its
expressions — this is the *opposite* direction: it asks whether relocating
the computation **earlier**, lower in the tree, closer to the original
scan, across intervening filters, pass-through projections, and eligible
inner joins, would be cheaper. This is the mechanism behind "A bounded
string projection is cheaper to evaluate on the dimension side of a fanout
join": `substring(v, 1, 20)`, hoisted above the join earlier by
`PROJECTION_PULLUP`, gets pushed back down below it, because the join here
*fans out* (100 dimension rows become 100,000 output rows), and evaluating
the (narrow, 20-byte-bounded) substring once per dimension row, before the
fanout, is far cheaper than carrying the full 100-byte `v` through the join
and computing the substring 100,000 times afterward.

`FindPlacement`'s own comment states the key architectural decision
explicitly:

```cpp
// src/optimizer/projection_placement.cpp:801
// Compare every position on the dependency path instead of making a greedy per-join decision.
```

The function doesn't stop at the *first* join it can cross — it walks the
**entire** single-child dependency chain (through `LOGICAL_FILTER`, through
`LOGICAL_PROJECTION` via `RewriteThroughProjection`, through every
`IsEligibleJoin` in sequence, as long as the expression's dependencies
resolve unambiguously to exactly one child at each join — `GetExpressionChild`,
`projection_placement.cpp:754`), collecting one `ExpressionPlacement`
candidate *per join crossed*, and only then picks the single cheapest
candidate among **all** of them, comparing each one's candidate cost against
the *original* position's cost scaled by `MINIMUM_COST_RATIO`. This directly
explains "The cost is evaluated across the complete single-child path
through multiple joins" — a two-join chain (`dimension JOIN fact f1 JOIN
fact f2`) is evaluated as one path with two candidate stopping points, not
as two independent single-join decisions that could each individually fall
short of the 25% hysteresis threshold while a two-hop move still clears it.

**`RewriteThroughProjection`** (`projection_placement.cpp:202`) is the
mechanism that lets a candidate expression's dependency trace *through* an
ordinary pass-through projection, as long as every column it needs is itself
just a bare column reference one level down — this is what makes "Placement
follows direct aliases through ordinary projections as well" work: `CAST(i
AS BIGINT) AS k` sitting between the candidate expression and the join
doesn't block the trace, because `k` is directly and losslessly traceable
back to the underlying `i`.

### 21.1.6 Grouping and rewiring: `PlacementGroup`, `RebuildProjection`, and the binding-replacement machinery

A subtlety worth calling out precisely, because it answers a real
correctness question a naive implementation could get wrong: what happens
when **two different expressions in the same projection** both want to
relocate to the **same target join**? `OptimizeOperator`'s inner loop
(`projection_placement.cpp:969`) collects every expression whose best
placement targets the identical join operator (`RefersToSameObject`) into a
single `PlacementGroup`, and `ApplyPlacementGroup`
(`projection_placement.cpp:883`) rewrites the tree for the **whole group in
one shot** — inserting one shared lower `LogicalProjection` immediately
above the target join that computes every relocated expression side by
side, rather than performing N separate, sequential tree-surgery passes that
would each individually invalidate the column bindings the next one needs to
find. This is exactly the "Multiple aliases of the candidate still compute
the expression only above the join" test: `s AS s1, s AS s2`, both aliases
of the same underlying `repeat(v, 10)` computation, get handled as one
placement group, not two independent relocations that would otherwise risk
computing the expensive `repeat` call twice.

The actual tree surgery — `RebuildProjection` (`projection_placement.cpp:483`)
for the Lift phase, `ApplyPlacementGroup` for the Placement phase — follows
the same pattern in both directions: build a **new** `LogicalProjection`
with a fresh `TableIndex` (`optimizer.binder.GenerateTableIndex()`) at the
target position, copy across every binding that has to survive, install the
relocated expression(s), and then run a whole-subtree
**`ColumnBindingReplacer`** (the same generic binding-rewrite utility
`WINDOW_SELF_JOIN` and `RemoveDuplicateGroups` — see Part II — both reuse) to
redirect every reference to the old binding at every point above the
rewrite.

In the Placement phase, the new lower projection starts as an *identity*
projection: one plain column reference per output column of the target
join. `ApplyPlacementGroup` builds it with a shared helper,
`LogicalProjection::CreateIdentity`, and then appends the moved
expressions:

```cpp
// src/optimizer/projection_placement.cpp:893
auto old_bindings = target->GetColumnBindings();
auto lower_index = optimizer.binder.GenerateTableIndex();
auto lower_projection = LogicalProjection::CreateIdentity(lower_index, std::move(target));
for (auto &placement : group.placements) {
	lower_projection->expressions.push_back(std::move(placement.expression));
}
```

`CreateIdentity` (`src/planner/operator/logical_projection.cpp`) resolves
the child's types, makes one `BoundColumnRefExpression` per child binding,
and puts the child under the new projection.

In the Lift phase, *when* the replacer runs matters. A lift may rebuild
several projections on its path (the source projection, and any
pass-through projection between it and the target join). Each rebuild gives
columns new bindings. `RebuildProjection` only *records* each
`(old binding, new binding)` pair in `binding_replacements`. It also
remembers the lowest projection it rebuilt (`lowest_rebuilt`). The replacer
runs once, at the end of `ApplyLiftPlacement`, after the restore projection
is in place:

```cpp
// src/optimizer/projection_placement.cpp:690
// now that the restore projection reproduces the target's original column layout, every projection map in the
// plan is consistent again: replace the references to the bindings of the rebuilt projections (resolved through
// the chain of rebuilds) everywhere above the lowest rebuilt projection
ColumnBindingReplacer rebuild_replacer;
for (auto &replacement : binding_replacements) {
	auto final_binding = ResolveBinding(replacement.first, binding_replacements);
	if (final_binding != replacement.first) {
		rebuild_replacer.replacement_bindings.emplace_back(replacement.first, final_binding);
	}
}
rebuild_replacer.stop_operator = lowest_rebuilt;
rebuild_replacer.VisitOperator(*root);
```

Each old binding is chased through the whole chain (`ResolveBinding`) to
its final binding. So if rebuild 1 renamed `A → B` and rebuild 2 renamed
`B → C`, every reference to `A` becomes `C` in one step. While the chain is
still being built, a later `RebuildProjection` also uses `ResolveBinding` to
find a dependency whose reference has not been rewritten yet.

Why wait until the end? A `ColumnBindingReplacer` walk asks every operator
above for its column bindings. But the projection maps of the joins above
are only fixed later, step by step. If the replacer ran right after each
rebuild, a join with a stale projection map could index past the end of its
child's new, shorter column list:
`INTERNAL Error: Attempted to access index 5 within vector of size 5`.
§21.1.10 walks the two regression tests for this.

`RemapParentProjectionMap` (`projection_placement.cpp:442`) and
`ExposeBinding` (`projection_placement.cpp:466`, `:879`) handle the fact
that intervening operators may carry a **projection map** (the mechanism
`COLUMN_LIFETIME`, pass #26, installs — §7.4) that has to be kept consistent
with whatever new bindings this rewrite introduces or renames, threading a
new column through an operator's existing narrowing map rather than
silently dropping it. This is the concrete answer to the "Placement does not
depend on column-lifetime projection maps" test: the pass works correctly
whether or not `COLUMN_LIFETIME` has already run, but when it *has*, this
machinery has to cooperate with its projection maps rather than fight them.
`ResolveBinding` (`projection_placement.cpp:422`) — a small binding-chase
helper over a list of `(old, new)` replacement pairs, with an explicit
`InternalException` if it ever detects a cycle — is what lets a *second*,
independent placement group's rewrite still correctly find a binding that a
*first* group's rewrite already renamed, which is precisely "Placements on
different join branches are recomputed after each binding rewrite" and
"Sequential placements preserve indexed outputs above the rewritten join":
placing `l.v`'s substring and `r.v`'s substring in the same query, across
two different join branches, requires each placement's tree surgery to
leave the *other* branch's bindings intact and discoverable.

### 21.1.7 `test/optimizer/projection_placement.test`, walked case by case

This file is entirely `EXPLAIN`-driven — every substantive assertion is a
`query II` block whose second column is a `<REGEX>` or `<!REGEX>` match
against `logical_opt` (the post-optimization logical plan, thanks to
`PRAGMA explain_output = OPTIMIZED_ONLY` at the top of the file). It proves
**where** the rewrite placed things, not what values came out — that's
Part II's job, covered next. Four fixture tables set the stage:
`placement_dimension` (100 rows, a 10-valued grouping column `k`, and a
100-byte padded string `v`), `placement_fact` (100,000 rows, `i` cycling
0–99 — a genuine **fanout** join partner for `dimension`), `placement_selective`
(10 rows — a genuine **filtering** join partner), and `placement_wide` (100
rows with a 2000-byte string).

**Fanout joins push expressions down.** The opening three tests — `substring(v,
1, 20)`, `substring(v, -20, 20)` (negative offset), and
`substring_grapheme(v, 1, 20)` — all joined against `placement_fact`, all
assert the *positive* regex `.*JOIN.*PROJECTION.*Expressions:...s...`
(§21.1.5 already explained: `JOIN` text preceding a `PROJECTION` containing
`s` means the projection sits *below*, as the join's child, in `EXPLAIN`'s
top-down rendering). All three substring variants get the same treatment
because all three are recognized by `PatternIsPrefix`-adjacent statistics
reasoning as producing a **statically bounded** output width (20 characters,
regardless of which substring flavor computes it), which is exactly what
§21.1.3's `GetBindingWidth` needs to cost them at all.

**The pass is independently toggleable, and independent of `column_lifetime`.**
`SET disabled_optimizers = 'projection_placement'` on the identical query
flips the assertion to `<!REGEX>` — with the pass off, the expression stays
wherever `PROJECTION_PULLUP` (and nothing after it) left it, which the test
confirms is *not* below the join. A second block disables `column_lifetime`
instead and re-confirms the *positive* regex still holds — direct evidence
that `PROJECTION_PLACEMENT`'s own tree surgery (§21.1.6) doesn't silently
depend on `COLUMN_LIFETIME` having already built projection maps for it to
piggyback on.

**Correctness alongside plan shape.** Immediately after the toggle tests,
one `query I` actually runs the relocated query
(`WHERE length(s) = 20` over the fanout join) and checks the row count —
`100000` — the one place this particular file steps outside pure `EXPLAIN`
checking, specifically to confirm the relocated computation still produces
rows that satisfy a predicate depending on its own output.

**Payload shrinkage alone can justify relocation, without a large fanout
ratio.** `length(v)` from `placement_wide` (2000-byte strings) joined
against `placement_dimension` (100 rows on both sides — no dramatic
cardinality change at all) still moves down (`<REGEX>`, positive) — because
`GetExclusiveInputWidth` sees a 2000-byte input shrinking to a 4-byte
`INTEGER` output, and that width ratio alone clears `MINIMUM_COST_RATIO`
even with a roughly 1:1 join. The very next test is the negative control
already covered in §21.1.3: `SELECT d.v, n FROM ... JOIN ...` — with `d.v`
independently selected, the width saving evaporates and the placement
doesn't fire (`<!REGEX>`).

**Fixed-size expressions are eligible too.** `i + 1 AS n` — nothing
string-related at all — still relocates below a fanout join
(`<REGEX>`), confirming the cost model isn't string-specific; any
statically-widthed expression qualifies.

**The selective/fanout contrast, side by side.** This is the pass's central
demonstration, and the file states it almost as a minimal pair:
`substring(v, 1, 20)` against `placement_selective` asserts `<!REGEX>`
("must not cause expensive work to move to its larger input" — stays or
moves *above* the filtering join, evaluated only on survivors), while the
same expression against `placement_fact` asserts `<REGEX>` (stays *below*
the fanout join, evaluated once per pre-fanout row). Two further blocks
repeat exactly this contrast for `repeat(v, 10)` (a payload-*expanding*
expression rather than a payload-*shrinking* one): it moves **above**
`placement_selective` (`<!REGEX>` — this is a Lift, §21.1.5) and stays
**below** `placement_fact` (`<REGEX>` — computing the expansion once per
dimension row beats computing it once per fanned-out row, even though the
*result* is now wider than the input).

**An overflow-adjacent edge case, honestly flagged.** `repeat(v, 50000000)`
against `placement_selective` asserts the *positive* regex — the expression
is found **below** the join, the same position as the ordinary fanout case,
not the "moves above a selective join" position its non-overflowing sibling
(`repeat(v, 10)`) took on the identical join. This book's sourcing policy
(front matter, "A note on scope and honesty about uncertainty") calls for
stating this plainly rather than inventing a tidy explanation: the test's
own comment — *"A repeat whose bounded result can exceed the string size
limit remains fallible"* — documents that this call can, in principle, hit
DuckDB's string-size ceiling regardless of where it's evaluated, but tracing
*why* the width-costing math specifically favors the below-join position for
this one enormous, likely-overflowing multiplier (rather than the
above-join position its smaller sibling took) would require following the
`repeat` scalar function's own statistics-propagation rule for maximum
output length — a file this chapter didn't need to open for everything else
it explains, and one this book won't guess at rather than read. What can be
stated with confidence: whichever position is chosen, `PROJECTION_PLACEMENT`
never changes *whether* an overflow can occur, only where the (still
possibly-overflowing) computation happens to run.

**Semantic barriers, one join type at a time.** The remainder of the file is
a systematic sweep confirming §21.1.3's `IsEligibleJoin` barrier list, each
with its own dedicated test: a cross-condition expression that depends on
*both* join sides can't move into either child (`<REGEX>` showing
`PROJECTION.*JOIN` — the whole computation stays above, since it genuinely
needs both inputs simultaneously); an expression referenced *by the join
condition itself* can't cross it either (`d.s = f.i::VARCHAR` — the
`Conditions:` line in `EXPLAIN` still shows `s`, meaning the substring
computing `s` had to stay visible to the condition, not be relocated away
from it); `LEFT`, `RIGHT` (rendered as `LEFT` internally — DuckDB normalizes
`RIGHT JOIN` to a `LEFT JOIN` with swapped children), and `FULL` all keep
the projection above them; `SEMI` and `ANTI` joins are explicitly *not*
cost-placement candidates (`<!REGEX>` — no relocation happens across them at
all, consistent with `IsEligibleJoin`'s hard `join_type == INNER` check);
and `MARK` joins (from an `IN (subquery)`) and `SINGLE` joins (from a scalar
subquery) both remain barriers too. Every one of these is a direct,
one-to-one confirmation of the `IsEligibleJoin` gate read in §21.1.3 — there
is no daylight between what the source checks and what the test suite
independently confirms.

**A robustness test for the Lift loop's own termination.** "A projection
that cannot cross `UNNEST` must not be retried indefinitely" — `unnest([1])
AS u` sitting alongside a `repeat(v, 100) AS s` in the same projection is a
regression against a specific failure mode: `LiftFirstProfitableExpression`
loops until no further lift succeeds, and if an ineligible expression
(anything downstream of an `UNNEST`, an operator type §21.1.5's dependency
walk doesn't recognize) were mishandled, the loop could either spin forever
re-discovering the same non-candidate or silently corrupt the plan trying to
lift past an operator it doesn't understand. The test has no `EXPLAIN`
assertion — it only checks the query terminates and returns the right
aggregate (`1000000`), which is exactly the right shape for a "this used to
hang or crash" regression.

**`NULL`s and duplicate keys produce identical results with the pass on or
off**, exercised via a `foreach do '' 'projection_placement'` loop running
the *same* query with `disabled_optimizers` set to each value in turn — a
small but genuinely useful idiom distinct from the paired-`nosort`-label
one this book's sibling files describe: rather than asserting two plans are
byte-identical, this asserts two *executions*, under two different
optimizer configurations, produce byte-identical *rows*. `placement_values`
deliberately includes a duplicate key (`(1, 'abcd')` twice), a `NULL` value,
and a `NULL` key, specifically to stress the relocation machinery's
handling of `NULL` propagation through the substring/join combination.

**A throwing expression cannot be pulled across a filtering join**
(§21.1.7, promised in §21.1.4). `CAST(v AS INTEGER)` where `placement_cast_input`
contains a genuinely non-numeric string (`'bad'`) is joined against
`placement_cast_match`. The `EXPLAIN` still shows the cast-producing
projection below the join (`<REGEX>` — plausible, since nothing here
actually needs to move for this particular shape to be an interesting test
of the *throwing* behavior rather than the *placement* decision), and the
file's real point is the pair of `statement error` blocks that follow: the
identical `Conversion Error: ...bad...` fires whether the query runs with
the optimizer fully enabled or with `PRAGMA disable_optimizer` in effect —
proof, by direct comparison rather than by reading the `CanThrow()` guard
and trusting it, that this pass's barrier for throwing expressions holds in
practice, not merely in the source.

### 21.1.8 `test/optimizer/projection_placement_results.test`, walked case by case

Where the previous file asks "did the rewrite fire, and where did it put
things," this file asks a strictly narrower question: **whatever the
optimizer decided to do, is the query's actual output still correct?**
Every substantive block here is a bare `query I`/`query II`/`query III`
against literal expected rows — there is not one `EXPLAIN` statement
anywhere in the file. This is the direct, source-verified answer to the
task's own naming-convention question: yes, the two files really do split
exactly along the "plan shape" versus "result correctness" line the names
suggest, and — per §21.1.9 below — this was true from the very first commit
that introduced either file, not a later refactor.

The file re-derives its own small fixture set
(`placement_result_dimension`, `placement_result_fact`) rather than sharing
tables with `projection_placement.test`, and then works through a
condensed version of the same scenarios, purely for their **result**, never
their plan shape:

- The basic fanout-join substring case (`sum(length(s))` → `20000`),
  mirroring the very first `EXPLAIN` test in the sibling file.
- The two-sided-alias case (`substring(l.v, ...)` and `substring(r.v, ...)`
  both relocating across the same self-join on `k`) — `sum(length(ls))`,
  `sum(length(rs))` → `20000`, `30000` — the result-correctness counterpart
  of "Placements on different join branches are recomputed after each
  binding rewrite."
- **"Preserve the first placed expression when a later placement shifts its
  join output"** — a three-way join (`dimension l JOIN dimension r USING
  (k) JOIN fact f ON f.i = l.i`) where `rs` (from `r.v`) and `ls` (from
  `l.v`) both relocate, but the *second* join (against `f`) changes the
  column layout of the first join's output *after* `rs` has already been
  placed there. The comment names exactly the bug class this guards
  against: if the tree surgery for placing `ls` at the second join failed
  to correctly account for `rs`'s already-rewritten bindings from the first
  placement (§21.1.6's `ResolveBinding`/binding-chase machinery), this query
  would either silently return wrong values or throw an internal
  binding-not-found exception. The expected output — `200000`, `300000`,
  scaled up from the two-table version's `20000`/`30000` by `fact`'s
  ten-per-key multiplicity — is exactly what correct, order-independent
  handling of sequential placements should produce.
- The `NULL`/duplicate-key correctness case, re-run here for its actual rows
  rather than its plan shape (same expected output as the sibling file's
  version, `1  ab` four times plus `2  NULL`).
- The `UNNEST` non-termination regression, re-run here purely for its
  numeric result (`1000000`), the mirror of the sibling file's
  plan-agnostic version of the same test.
- **A verification-projection / IEJoin interaction test**, the most
  structurally distinct block in the file: `SET debug_verification_projection
  = true` together with `merge_join_threshold = 0`, `nested_loop_join_threshold
  = 0`, and `prefer_range_joins = true` forces an inequality-join shape
  (`l.start < r.stop AND r.start < l.stop AND l.price + r.bid > 300`) onto
  DuckDB's IEJoin operator specifically, over data deliberately containing
  `NULL` `start`/`stop` values in both `placement_ie_left` and
  `placement_ie_right`. `debug_verification_projection` is a DuckDB internal
  consistency-checking mode that wraps operators in extra verification
  projections purely to catch type or binding mismatches the normal
  execution path might silently tolerate — the comment, *"Verification
  projections preserve IEJoin payload types,"* states directly that this
  test exists to confirm `PROJECTION_PLACEMENT`'s tree surgery around a join
  as structurally unusual as IEJoin (which carries its own specialized
  payload/condition-column bookkeeping, distinct from an ordinary hash
  join's) doesn't corrupt a payload column's *type* along the way — a class
  of bug that would likely only surface under this specific debug
  instrumentation, which is presumably exactly why the test enables it
  explicitly rather than relying on ordinary execution to catch it.

### 21.1.9 Why two files: the split was there from the pass's first commit

This chapter's opening TEST-FORMAT note asked for the answer to be verified,
not assumed — and the git history settles it cleanly. `git log --oneline
--follow --diff-filter=A -- test/optimizer/projection_placement_results.test`
returns exactly one commit: **`66553e6a0d`, "Add cost-aware projection
placement"** — the *same* commit that introduces `projection_placement.cpp`
itself and `test/optimizer/projection_placement.test`. There was never a
single, combined test file that later got split; the pass's author designed
the two-file separation as part of the pass's original design, not as a
later cleanup. A later commit, **`19f36044f1`, "Isolate projection placement
plan tests,"** is worth flagging precisely because its name could easily be
misread as "this is the commit that created the split" — reading its actual
diff shows it does something narrower and unrelated to the file split: it
adds `SET disabled_optimizers = 'compressed_materialization'` throughout
`projection_placement.test` specifically to stop a *different* pass
(`COMPRESSED_MATERIALIZATION`, one of the two policy-check-only toggles
§7.4's overview flagged as having no `RunOptimizer` call site of its own)
from occasionally altering the exact `EXPLAIN` shape these plan-shape
assertions depend on — "isolating" the plan tests from a noisy neighbor
pass's interference, not isolating them into their own file. Two further
commits, **`a24d1b04fa`, "Place expressions across projection boundaries"**
(almost certainly the commit that added `RewriteThroughProjection`'s
alias-tracing capability, §21.1.5) and **`461fa21934`, "Preserve bindings
across sequential projection placements"** (confirmed by its own diff to add
both the "preserve the first placed expression" results test *and* the
"Placements on different join branches" plan test in the same commit, plus
a 23-line change to `projection_placement.cpp` itself), show the pass's
capability growing incrementally after its initial commit, with **both**
test files gaining new coverage together at each step — further evidence
the two-file split is a maintained, load-bearing convention throughout the
pass's history, not an artifact of a single initial commit that was never
revisited.

The practical payoff of the split, independent of its history: a future
change to the *cost model* (which position gets chosen) can be verified
against `projection_placement.test` without touching a single expected
result row, and a future change to the *tree-surgery mechanics* (whether the
relocated computation still produces the right value once moved) is
verified against `projection_placement_results.test` without needing to
hand-maintain a brittle `EXPLAIN` regex for every scenario that isn't
actually about plan shape. It's the same engineering instinct behind
`WINDOW_SELF_JOIN`'s own test-file split discussed in this book's sibling
deep-dive (§7.10.8) — one file for "did the rewrite fire," a structurally
separate one for "is the output still right" — applied here with an actual
two-file (rather than two-block) separation, and confirmed, this time, all
the way back to a single, unambiguous originating commit.

A third place holds tests for this pass: the
directory `test/sql/optimizer/projection_placement/`. It holds two
results-only regression tests, covered next.

### 21.1.10 Two regression tests: lifting through nested joins

Both tests guard the deferred replacer described in §21.1.6. Both are
results-only (no `EXPLAIN`). With an eager replacer, both fail with an
`INTERNAL Error`. They use TPC-DS-like table names
(`store_sales`, `item`, `date_dim`).

**`test/sql/optimizer/projection_placement/lift_through_nested_joins.test`**.
A subquery computes several `CASE` columns over
`store_sales`. The outer query joins it with `item` and then with
`date_dim`, and groups by one of the `CASE` columns:

```sql
SELECT i.i_item_id,
       ss.resource_id,
       MAX(CAST(d.d_date AS TIMESTAMP WITH TIME ZONE)) AS last_seen,
       MIN(CAST(d.d_date AS TIMESTAMP WITH TIME ZONE)) AS first_seen,
       CAST(SUM(ss.ss_quantity) AS VARCHAR) || '/' || CAST(AVG(ss.ss_sales_price) AS VARCHAR) || '/' || CAST(SUM(ss.ss_net_profit) AS VARCHAR) || '/' || COALESCE(ss.resource_id, '-') AS summary
FROM (
    SELECT ss_item_sk, ss_customer_sk, ss_quantity, ss_sales_price, ss_net_profit, ss_sold_date_sk,
           CASE WHEN ss_quantity = 2 THEN 'r' END AS resource_id,
           CASE WHEN ss_net_profit > 0 THEN 'active' END AS status,
           CASE WHEN ss_coupon_amt > 0 THEN 'discounted' END AS pricing_type,
           CASE WHEN regexp_matches(CAST(ss_ticket_number AS VARCHAR), '[0-9]+') THEN 'numeric' END AS ticket_type
    FROM store_sales
) ss
JOIN item i ON ss.ss_item_sk = i.i_item_sk
JOIN date_dim d ON ss.ss_sold_date_sk = d.d_date_sk
GROUP BY i.i_item_id, ss.resource_id
ORDER BY i.i_item_id, ss.resource_id
----
a	r	2000-01-02 00:00:00+00	2000-01-01 00:00:00+00	4/27.5/3.00/r
b	NULL	2000-01-02 00:00:00+00	2000-01-02 00:00:00+00	1/5.0/-1.00/-
```

The Lift phase wants to move a `CASE` expression up past two joins. The
first rebuild narrows the subquery's projection. With an eager replacer,
the walk would reach the lower join while its projection map still pointed
at the old, wider layout.

**`test/sql/optimizer/projection_placement/lift_through_left_join_filter.test`**.
The same shape, with 3,000 generated `store_sales`
rows and one more join: a `LEFT JOIN store_returns` above the two inner
joins. The lift path ends at an inner join, but the stale projection map
sits *above* the end of the path, on the `LEFT JOIN`. Only the restore
projection that `ApplyLiftPlacement` installs last makes that map valid
again. This is why remapping the parent's map before the replacer is not
enough, and why the replacer runs at the very end. The test checks one summary row: `1920` groups, one distinct
`resource_id`, and the min and max of the other columns.

---

## Part II — `DUPLICATE_GROUPS`: collapsing redundant grouping keys, syntactic and functional alike

### 21.2.1 What this pass is, in one sentence

`RemoveDuplicateGroups` finds `GROUP BY` entries that are either the *exact
same underlying column* as another entry already in the list, or a
*deterministic, single-column function* of another entry that's already a
bare column reference in the list, and removes the redundant entry from the
aggregate's actual grouping key — narrowing the hash-aggregate's key width —
while, for the second (functional) case only, inserting a projection above
the aggregate that recomputes the removed expression from the surviving
base column, so every downstream reader still sees the value it originally
asked for.

### 21.2.2 The header comment states the scope precisely — verify it against the code

`src/include/duckdb/optimizer/remove_duplicate_groups.hpp` carries this
class comment, worth quoting in full before anything else, because every
claim in it is directly checkable against `remove_duplicate_groups.cpp` and
this section does exactly that, line by line:

```cpp
// src/include/duckdb/optimizer/remove_duplicate_groups.hpp
//! Removes same-binding duplicate groups (e.g. from Deliminator / RemoveUnusedColumns) and
//! groups that are deterministic functions of a sibling column-ref group (e.g.
//! `GROUP BY x, x-1, cast(x AS BIGINT)`). The latter are recomputed in a projection above.
```

Two claims to verify, and both hold up: (1) there are genuinely **two**
distinct removal mechanisms — same-binding and functionally-derived — not
one generalized "these are equal" test; and (2) the "recomputed in a
projection above" language is literal, not a simplification — §21.2.4 walks
the exact projection-insertion code.

### 21.2.3 `CollectGroupRemovals`: the two passes over the `GROUP BY` list

`VisitAggregate` (`remove_duplicate_groups.cpp:230`) first bails out
entirely — doing nothing at all — in two situations, each checked before any
duplicate-detection logic even runs:

```cpp
// src/optimizer/remove_duplicate_groups.cpp:230
void RemoveDuplicateGroups::VisitAggregate(LogicalAggregate &aggr) {
	if (!aggr.grouping_functions.empty()) {
		return;
	}

	// If there are multiple grouping sets (ROLLUP/CUBE), we cannot remove duplicate groups
	// because the position of groups matters semantically in ROLLUP(col1, col2, col3),
	// even if col1 and col3 reference the same column binding (e.g., after join column replacement)
	if (aggr.grouping_sets.size() > 1) {
		return;
	}
```

**`grouping_functions` non-empty** means the query calls SQL's `GROUPING()`
function, which reports — as a literal bitmask, by *position* — which of a
`ROLLUP`/`CUBE`/`GROUPING SETS` aggregate's grouping columns were actually
part of the specific grouping set that produced a given output row. Silently
dropping one of those positions would corrupt the bitmask `GROUPING()`
reports, regardless of whether the underlying columns happen to be
value-identical — so this pass declines outright the moment `GROUPING()` is
anywhere in the query.

**`grouping_sets.size() > 1`** is `ROLLUP`/`CUBE`/explicit `GROUPING SETS` —
more than one grouping set means position is semantically load-bearing
(`ROLLUP(col1, col2, col3)` produces four progressively-NULLed output rows,
each defined by *which positions* got rolled up), and the comment states the
motivating scenario directly, in the *exact same vocabulary* this chapter
already used for the pass's downstream trigger: "even if col1 and col3
reference the same column binding (e.g., after join column replacement)" —
§21.2.6 traces that mechanism to its actual source.

Once past both guards, `CollectGroupRemovals`
(`remove_duplicate_groups.cpp:60`) runs two separate scans over `aggr.groups`:

**Scan 1 — same-binding duplicates**, syntactic and unconditional:

```cpp
// src/optimizer/remove_duplicate_groups.cpp:63
column_binding_map_t<ProjectionIndex> first_occurrence;
for (auto group_idx : ProjectionIndex::GetIndexes(groups.size())) {
	const auto &group = groups[group_idx];
	if (group->GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
		continue;
	}
	const auto &binding = group->Cast<BoundColumnRefExpression>().Binding();
	auto it = first_occurrence.find(binding);
	if (it == first_occurrence.end()) {
		first_occurrence.emplace(binding, group_idx);
	} else {
		removals.push_back({group_idx, it->second, nullptr});
	}
}
```

Every `GROUP BY` entry that's a bare column reference gets recorded, keyed
by `ColumnBinding`, the first time it's seen; every *subsequent* occurrence
of the *same* binding is scheduled for removal, unconditionally — this is
the literal `GROUP BY x, x` case (or `GROUP BY x, y, x`), and it needs no
extra safety check, because two occurrences of the identical binding are, by
construction, the identical value on every row. No type restriction applies
here — this is the "purely syntactic, always safe" half of the pass.

**Scan 2 — functionally-derived duplicates**, semantic, and considerably
more restricted:

```cpp
// src/optimizer/remove_duplicate_groups.cpp:78
for (auto group_idx : ProjectionIndex::GetIndexes(groups.size())) {
	const auto &group = groups[group_idx];
	if (group->GetExpressionType() == ExpressionType::BOUND_COLUMN_REF) {
		continue;
	}
	if (group->IsFoldable() || !group->IsConsistent()) {
		continue;
	}
	column_binding_set_t colrefs;
	if (!CollectColumnRefBindings(*group, colrefs) || colrefs.size() != 1) {
		continue;
	}
	auto base_it = first_occurrence.find(*colrefs.begin());
	if (base_it == first_occurrence.end()) {
		continue;
	}
	if (!BaseTypeAllowsCollapse(groups[base_it->second.GetIndex()]->GetReturnType())) {
		continue;
	}
	removals.push_back({group_idx, base_it->second, nullptr});
	any_derived = true;
}
```

**This is the direct, source-confirmed answer to whether `DUPLICATE_GROUPS`
handles semantic, not just syntactic, equivalence: yes, but in a
deliberately narrow form — functional dependency on a single, already-grouped
base column, not general expression-equivalence proving.** Walking every
condition:

- Bare column references are skipped (already handled by Scan 1).
- **`IsFoldable()`** — a compile-time constant `GROUP BY` entry — is left
  completely untouched by this pass. A constant grouping key isn't "derived"
  from any column at all, so there's no base to collapse it against, and
  this pass makes no attempt to special-case it (constant-folding elsewhere
  in the pipeline, §7.5, is a different concern entirely).
- **`!IsConsistent()`** — anything containing a volatile function (one
  whose `FunctionStability` is `VOLATILE`, such as `random()`) — is
  rejected. A function that is only stable *within* one query
  (`CONSISTENT_WITHIN_QUERY`, for example one that reads `now()` or the
  `TimeZone` setting) counts as consistent and is allowed, because this
  rewrite happens inside one query. This is the correctness
  precondition for the whole functional-dependency argument to hold at all:
  if `f(x)` isn't guaranteed to return the *same* value every time it's
  handed the *same* `x`, then knowing "this row's `x` matches that group's
  `x`" says nothing about whether `f(x)` also matches, and recomputing it
  later from the surviving `x` could produce a different value than
  whatever was computed the first time.

  `BoundFunctionExpression::IsConsistent()`
  (`src/planner/expression/bound_function_expression.cpp:60-64`) tests
  `function.GetStability() != FunctionStability::VOLATILE`. The separate
  "same value only within this query" question is answered by another
  method, `Expression::RequiresRebind()`, used for things such as index keys
  that outlive one query. For this pass it means a derived group that
  depends on a session setting (for example an ICU date function over `x`
  that reads `TimeZone`) can be collapsed. That is safe, because the setting
  cannot change in the middle of one query.
- **`CollectColumnRefBindings`** (`remove_duplicate_groups.cpp:27`) walks the
  candidate expression's own tree and collects every distinct column
  binding it reads, **bailing the moment a second distinct binding
  appears**. This is the load-bearing restriction that keeps the whole
  mechanism tractable: an expression like `x + y` is never considered,
  because proving `x + y` is redundant given some *other* group entry would
  require reasoning about the joint distribution of two columns, not a
  simple single-column functional dependency. Only expressions that are
  *provably* a function of exactly one already-grouped column qualify —
  `x - 1`, `CAST(x AS BIGINT)`, `x + 0`, `upper(x)` (if `IsConsistent()`),
  all pass this specific gate; `x + y`, `CASE WHEN y THEN x ELSE 0 END`, and
  anything else touching a second binding do not.
- The single surviving binding must match a binding **already recorded in
  Scan 1's `first_occurrence` map** — i.e., the base column this expression
  depends on must *also* appear, somewhere in the same `GROUP BY` list, as
  its own bare, already-grouped entry. This is the crux of why the rewrite
  is safe: if `x` is itself a grouping key, then within any one group, every
  row's `x` value is, by the definition of `GROUP BY`, identical — which
  means `f(x)`, for any deterministic `f`, is *also* identical across every
  row in that group, whether or not `f(x)` is separately listed as a
  grouping key. Adding `f(x)` to the grouping key can therefore never split
  a group that grouping by `x` alone wouldn't already have produced — the
  partitions are provably identical, group for group, with or without
  `f(x)` present. This is precisely "`GROUP BY x, x+0`" generalized from the
  trivial identity function to any single-column deterministic one, and it
  is exactly the reasoning the header comment's own worked example (`GROUP
  BY x, x-1, cast(x AS BIGINT)`) demonstrates.

### 21.2.4 The one subtlety that keeps this from being unconditionally safe: `BaseTypeAllowsCollapse`

The functional-dependency argument in §21.2.3 has exactly one hidden
assumption: that the hash-aggregate's own notion of "same value" (whatever
equality test it uses to decide two rows belong to the same group) agrees
with the notion of equality the *base* column's SQL type actually has. For
almost every type, it does. For three specific type shapes, it provably
doesn't, and `BaseTypeAllowsCollapse` (`remove_duplicate_groups.cpp:45`)
exists entirely to rule those out:

```cpp
// src/optimizer/remove_duplicate_groups.cpp:41
// Bases where SQL `=` doesn't imply bit-equality are unsafe: 0e0 == -0e0 (signbit distinguishes),
// '1 day' == '24 hour' (datepart distinguishes). INTERVAL today gets wrapped in
// normalized_interval(...) by the binder, so it shouldn't reach here, but we exclude it
// defensively in case that wrapping ever changes.
bool BaseTypeAllowsCollapse(const LogicalType &type) {
	return !TypeVisitor::Contains(type, [](const LogicalType &t) {
		switch (t.id()) {
		case LogicalTypeId::FLOAT:
		case LogicalTypeId::DOUBLE:
		case LogicalTypeId::INTERVAL:
			return true;
		default:
			return false;
		}
	});
}
```

The concrete failure mode this guards against: `0.0` and `-0.0` compare
*equal* under IEEE-754 floating-point comparison (and under SQL's `=`), and
a `GROUP BY` on a `DOUBLE` column groups both values together — but they are
**not bit-identical** (the sign bit differs), so a deterministic function
that can observe the bit pattern rather than just the "SQL-equal" value —
`sign(x)`, `1.0 / x` (producing `+Infinity` versus `-Infinity`), or a
bit-inspecting cast — could legitimately produce *different* results for
the two values, even though they landed in the same group. If this pass
collapsed `GROUP BY x, sign(x)` on the strength of "`x` is already grouped,
so `sign(x)` is redundant," and then recomputed `sign(x)` from a single
*representative* `x` value per group, it could silently produce a different
answer than the *original*, per-row evaluation of `sign(x)` would have —
correct-looking SQL, subtly wrong output, and the kind of bug that would be
extraordinarily hard to notice without a `0.0`/`-0.0` pair specifically in
the test data. `INTERVAL` has an analogous problem for a different reason:
DuckDB's interval equality treats `'1 day'` and `'24 hours'` as equal even
though their internal (months, days, microseconds) representations differ —
a function that inspects a specific date-part component rather than the
interval's overall equality class could again disagree across two
"equal" values. The comment's own aside — that `INTERVAL` literals are
already normalized by the binder before reaching this pass, so the carve-out
is currently defensive rather than load-bearing — is itself a small,
honest admission that this exact guard has at least once mattered enough to
add and keep, even if the specific `INTERVAL` half of it isn't currently
reachable through the front door.

`TypeVisitor::Contains` (`src/include/duckdb/common/type_visitor.hpp:21`)
recurses into nested type structure — a `STRUCT` or `LIST` containing a
`DOUBLE` field anywhere inside it also fails this check — so the exclusion
is not merely "the base column's own top-level type," but "anything a
bit-sensitive function could possibly observe by reaching into this value."
Note precisely what this check does **not** need to guard: same-binding
duplicates (Scan 1, §21.2.3) never need it, because two occurrences of the
*literal same binding* are the identical value, bit for bit, by
construction — there's no re-derivation happening, and hence no
representative-value substitution that could go wrong. `BaseTypeAllowsCollapse`
is checked exclusively against the *base* column of a functionally-derived
removal, precisely because that's the only case where a single group's
worth of rows could genuinely differ from each other in ways SQL equality
papers over.

### 21.2.5 What actually happens to the plan: `ApplyGroupRemovals` and the passthrough projection

`ApplyGroupRemovals` (`remove_duplicate_groups.cpp:105`) does the physical
bookkeeping: it compacts `aggr.groups`, dropping every removed index, and
builds `group_binding_map` — a map from **every** original
`ColumnBinding(aggr.group_index, i)` (not just the removed ones) to its
post-compaction replacement, where a *removed* slot's entry points at its
*surviving target's* new binding rather than at nothing. It also remaps
`aggr.grouping_sets` — trivial here, since §21.2.3's first guard already
guaranteed there's exactly one grouping set left to fix up. Before any of
that, `VisitAggregate` throws away the aggregate's per-group statistics:

```cpp
// src/optimizer/remove_duplicate_groups.cpp:249-251
// Earlier statistics (e.g. from the filter simplification) no longer match the groups - the statistics
// propagation that runs after this optimizer computes them again
aggr.group_stats.clear();
```

`group_stats` holds one statistics entry per `GROUP BY` slot. Removing a
slot shifts the positions, so the old entries would describe the wrong
groups. This pass does not remap them. It clears them, and relies on the
full `STATISTICS_PROPAGATION` pass (#34), which runs later, to compute
them again for the new, shorter group list.

Why can the list be non-empty here at all? Because statistics propagation
also runs *early*, in a limited mode. Right after `FILTER_PUSHDOWN`, the
optimizer runs `FilterStatisticsOptimizer` (`optimizer.cpp:293-296`). It
calls the `StatisticsPropagator` in `FILTER_SIMPLIFICATION` mode
(`src/optimizer/filter_statistics.cpp:60`), and that propagator fills
`group_stats` on every aggregate it visits
(`src/optimizer/statistics/operator/propagate_aggregate.cpp:484`).

So `group_stats` is not a sign of pass order: it can be filled before this
pass runs. The pass order is still `DUPLICATE_GROUPS` before the full
`STATISTICS_PROPAGATION`, and that later run is what makes clearing safe.

The regression test is
`test/optimizer/remove_duplicate_groups_with_stats.test`. It groups by the
same column twice under two names, then filters on the aggregate's output,
so the early statistics run sees the aggregate first:

```sql
CREATE TABLE t(x INTEGER, y INTEGER);
INSERT INTO t VALUES (1, 1), (2, 2), (2, 3), (NULL, 4);

SELECT * FROM (SELECT x, x AS x2, sum(y) s FROM t GROUP BY x, x2) WHERE s > x ORDER BY x;
----
2	2	5
```

`GROUP BY x, x2` is a same-binding duplicate (`x2` is just `x`), so Scan 1
removes it. A second query adds `count(*)` and `OR x IS NULL`, and checks
that the `NULL` group (`NULL NULL 4 1`) also survives.

Same-binding removals need nothing further: the retained group already
*is* the value every downstream reference needs, so `ReplaceBinding` (the
`BaseColumnPruner` machinery this class shares with `RemoveUnusedColumns`)
simply redirects every reference to the removed binding onto the survivor's
new binding, and the aggregate's own output shrinks by one column, full
stop.

Functionally-derived removals need one more step, because the *value* a
downstream reader expects (`x - 1`, say) is no longer computed anywhere —
only its base (`x`) survives inside the aggregate. `BuildDerivedProjection`
(`remove_duplicate_groups.cpp:158`) builds a **passthrough projection**
sitting directly above the aggregate: every surviving group and every
aggregate output gets a trivial `BoundColumnRefExpression` passthrough slot,
*except* the derived slot(s), which get the **original expression**,
rebound (via a small, single-purpose `ColumnBindingReplacer`) to read the
base column's *new*, post-compaction binding instead of whatever binding it
originally referenced. This new projection is not spliced into the tree
immediately — `VisitAggregate` runs while the visitor still only holds an
ordinary reference to the `LogicalAggregate`, not the parent's owning
`unique_ptr` slot, so the freshly built projection is stashed in
`pending_projections` (a `reference_map_t<LogicalAggregate,
unique_ptr<LogicalProjection>>`) and only physically inserted once
`VisitOperator`'s post-recursion loop (`remove_duplicate_groups.cpp:211-227`)
reaches the parent and can actually move the child `unique_ptr` into the new
projection's `children` vector. This two-step "compute now, splice once you
own the slot" pattern is a small but genuine piece of C++ ownership
plumbing worth recognizing on its own terms — it's the same reason
`ProjectionPlacementOptimizer` (Part I) always operates via `unique_ptr<LogicalOperator>
&root` references rather than raw `LogicalOperator&`, threading enough
ownership context through the call chain to actually replace a node rather
than merely mutate one in place.

### 21.2.6 Where the duplicates come from, mechanism one: literal SQL

The most obvious source of a same-binding duplicate is a query that simply
writes one — `GROUP BY x, x`, or `GROUP BY x, y, x` — which can arise
innocuously from SQL assembled programmatically (a BI tool or a dbt-style
templating layer building a `GROUP BY` list by concatenating a set of
"dimension" column names that weren't first deduplicated), or from a view or
CTE whose definition already selects the same underlying column twice under
different aliases, both of which then get grouped by at the outer query
level. Scan 1's unconditional handling exists to make this case cheap to
clean up regardless of how the duplicate got there.

### 21.2.7 Where the duplicates come from, mechanism two: `RemoveUnusedColumns`'s join column replacement — traced to the exact lines

The header comment's own parenthetical — "e.g., after join column
replacement" — names a *specific*, findable mechanism, not a vague
gesture, and it's worth tracing to its actual source rather than taking the
comment's word for it, per this book's stated sourcing policy. `RemoveUnusedColumns`
(`src/optimizer/remove_unused_columns.cpp`) is the class implementing pass
#23, `UNUSED_COLUMNS` — the pass immediately before `DUPLICATE_GROUPS` in
the fixed pipeline (§7.4). Its handling of `LOGICAL_COMPARISON_JOIN` (and
the decorrelation-adjacent `LOGICAL_ASOF_JOIN`/`LOGICAL_DELIM_JOIN`)
contains this, for an **`INNER`** join specifically:

```cpp
// src/optimizer/remove_unused_columns.cpp:597-632
// for inner joins with equality predicates in the form of (X=Y)
// we can replace any references to the RHS (Y) to references to the LHS (X)
// this reduces the amount of columns we need to extract from the join hash table
// (except in the case of floating point numbers which have +0 and -0, equal but different).
for (auto &cond : comp_join.conditions) {
	if (!cond.IsComparison() || cond.GetComparisonType() != ExpressionType::COMPARE_EQUAL) {
		continue;
	}
	if (cond.GetLHS().GetExpressionClass() != ExpressionClass::BOUND_COLUMN_REF) {
		continue;
	}
	if (cond.GetRHS().GetExpressionClass() != ExpressionClass::BOUND_COLUMN_REF) {
		continue;
	}
	if (cond.GetLHS().Cast<BoundColumnRefExpression>().GetReturnType().IsFloating()) {
		continue;
	}
	if (cond.GetRHS().Cast<BoundColumnRefExpression>().GetReturnType().IsFloating()) {
		continue;
	}
	// comparison join between two bound column refs
	// we can replace any reference to the RHS (build-side) with a reference to the LHS (probe-side)
	auto &lhs_col = cond.GetLHS().Cast<BoundColumnRefExpression>();
	auto &rhs_col = cond.GetRHS().Cast<BoundColumnRefExpression>();
	// if there are any columns that refer to the RHS,
	auto colrefs = column_references.find(rhs_col.Binding());
	if (colrefs == column_references.end()) {
		continue;
	}
	for (auto &entry : colrefs->second.bindings) {
		auto &colref = entry.get();
		colref.BindingMutable() = lhs_col.Binding();
		AddBinding(colref);
	}
	column_references.erase(rhs_col.Binding());
}
```

This is a *real*, independently interesting rewrite in its own right — for
an inner join whose condition is a plain column-to-column equality
(`t1.col1 = t2.col3`), **every reference anywhere in the plan above the
join to the right-hand column gets rewritten to point at the left-hand
column instead**, so the physical hash join never has to extract the
right-hand column from its build side at all if nothing above still needs
it under its own name — narrower join-hash-table payloads, purely as a side
effect of knowing the two columns are provably equal on every joined row.
The floating-point exclusion (`IsFloating()`, checked on *both* sides,
mirroring §21.2.4's `BaseTypeAllowsCollapse` almost exactly, right down to
citing the same `+0`/`-0` reasoning in its own comment) exists for
precisely the same bit-versus-value-equality concern this chapter already
walked through for `DUPLICATE_GROUPS` itself — two independent pieces of
the optimizer, in two different files, arriving at the identical safety
carve-out for the identical reason.

Here is the concrete payoff for `DUPLICATE_GROUPS`: a query written as
`SELECT col1, col3 FROM t1 JOIN t2 ON t1.col1 = t2.col3 GROUP BY col1, col3`
starts out with `col1` and `col3` as two **genuinely distinct**
`ColumnBinding`s — different tables, different columns, joined by equality,
nothing syntactically duplicated at all. But by the time `UNUSED_COLUMNS`
(#23) has run, every reference to `col3`'s binding — including the one
sitting inside the `GROUP BY` list — has been rewritten to reference
`col1`'s binding instead. What `DUPLICATE_GROUPS` (#24) receives, one pass
later, is a `GROUP BY` list containing the *literal same binding twice* —
not because the SQL said so, but because an entirely different, generically
useful rewrite one step earlier collapsed two provably-equal join columns
into one. This is the pipeline-ordering payoff the header comment's
parenthetical was pointing at, and it is precisely why `DUPLICATE_GROUPS`
sits immediately *after* `UNUSED_COLUMNS` in the fixed sequence rather than
anywhere else: it exists, in part, specifically to mop up this exact
byproduct.

### 21.2.8 `test/sql/optimizer/test_duplicate_groups_optimizer.test`, walked case by case

The file opens with the smallest possible fixture: `t1(col1, col2)` holding
one row `(1, 1)`, and `t2(col3)` holding one row `(1)` — deliberately built
so `t1.col1 = t2.col3` is both true and, after the join, the two columns
carry identical values.

**Three `ROLLUP`/`CUBE`/`GROUPING SETS` blocks, each a regression for the
`grouping_sets.size() > 1` guard, not a positive demonstration of
collapsing.** `GROUP BY rollup(col1, col2, col3)`, `GROUP BY cube(col1,
col2, col3)`, and `GROUP BY grouping sets (col1, col2, col3), (col1, col2),
(col1)` are all run over the join, and all three produce exactly the
NULL-padded row sets standard `ROLLUP`/`CUBE`/`GROUPING SETS` semantics
predict (four rows for `ROLLUP`'s three-column hierarchy, eight for `CUBE`'s
full power set, three for the explicit set list) — with **no** collapsing
of `col1` and `col3` visible anywhere in the output, even though the join
condition makes them equal on every row. This is exactly what §21.2.3's
first guard (`aggr.grouping_sets.size() > 1`) predicts: `VisitAggregate`
returns immediately for all three queries, before either duplicate-detection
scan ever runs, precisely *because* collapsing `col1` and `col3` here — even
though they're provably the same value — would silently corrupt which
*positions* rolled up in each output row. These three blocks exist to prove
the guard holds, not to show the pass doing anything.

**The one genuine positive test**, immediately following, with an explicit
comment naming exactly what it's protecting:

```sql
-- make sure there is only one group and unused columns/duplicate groups still
-- works.
-- if unused columns/duplicate groups combo breaks, each group will be on a separate line
query II
explain select
	col1,
	col3
from t1
	join t2
	on t1.col1 = t2.col3
group by col1, col3;
----
logical_opt	<REGEX>:.*Groups: col1.*
```

No `ROLLUP`/`CUBE` here — a single, ordinary grouping set — which is
exactly what lets `DUPLICATE_GROUPS` actually run. Per §21.2.7, `t2.col3`'s
binding has already been rewritten to `t1.col1`'s binding by the time this
pass sees the plan, making `GROUP BY col1, col3` a literal same-binding
duplicate; Scan 1 catches it, and the `EXPLAIN` output's `Groups:` line
shows **only** `col1` — `col3` has genuinely vanished from the aggregate's
grouping key, not merely been hidden from the `EXPLAIN` renderer. The
comment's own framing — "if unused columns/duplicate groups **combo**
breaks" — is precise, not casual: it's naming the *interaction* between two
passes (`UNUSED_COLUMNS` doing the binding rewrite, `DUPLICATE_GROUPS`
cleaning up what that rewrite left behind) as the thing under test, which is
exactly the "specialist mopping up after an earlier specialist" pattern this
whole chapter opened with.

**A closing `CUBE` correctness check, unrelated to binding collapse.** The
final block builds `t3(a, b, c)` with genuinely duplicate *row values*
(`(1,1,1)` appears twice, alongside `(1,2,2)` and `(1,2,1)`) and runs `SELECT
* FROM t3 GROUP BY cube(a, b, c) ORDER BY all`, checking all sixteen
expected output rows of a three-column `CUBE`'s full power set. This is
worth being honest about, in the same spirit §21.1.7 already applied to the
`repeat(v, 50000000)` edge case: `a`, `b`, and `c` here are three
**genuinely distinct** bindings with no equality relationship between them
at all — there is nothing for `DUPLICATE_GROUPS` to collapse, guard or no
guard, and `grouping_sets.size() > 1` would bail it out regardless. This
block is general `CUBE`-with-duplicate-row-values correctness coverage that
happens to live in this file because of its overall theme, not a
demonstration of this specific pass's mechanism — the kind of distinction
worth drawing out explicitly rather than silently folding into "and this
proves the optimizer handles duplicates," which it doesn't, in the sense
this chapter's mechanism actually operates.

### 21.2.9 The pass's own evolution, read from git history

`remove_duplicate_groups.cpp`'s commit history tells a clean, linear story
of a pass that started narrow and syntactic and was only later generalized
to the functional-dependency case §21.2.3 covers — which matters, because it
confirms the "purely syntactic first, semantic equivalence added
deliberately and later" reading this chapter has been building toward,
rather than the two mechanisms having shipped together as one design from
day one. In rough chronological order (oldest first, per `git log
--oneline --follow`, read newest-to-oldest as usual and reversed here for
narrative order): **`3ed2527fdb`, "Optimize ROLLUP queries"** — an early
commit in the file's history, from the era when this pass's interaction
with `ROLLUP`/`CUBE` was still being worked out; **`32424fb10a`, "don't
remove duplicate groups if there \[are] grouping functions, and don't
compress aggregates if there are duplicate groups"** — the commit that
literally introduces the `grouping_functions.empty()` guard this section
opened with, its own subject line naming the exact correctness concern
(`GROUPING()`'s positional bitmask) §21.2.3 explains; **`14fa2494c8`, "do not
remove unused columns if there are multiple grouping sets. remove duplicate
groups then removes rollup/cube functionality"** (on the *test* file's own
history) — a commit whose subject line is, almost word for word, the bug
report this chapter's §21.2.8 walkthrough describes the `grouping_sets.size()
> 1` guard as preventing. Only *after* this same-binding-only foundation was
solid does **`2487d4f53b`, "Collapse FD-derived GROUP BY groups"** land,
with a commit body that states the generalization in almost exactly this
chapter's own words:

> Extends RemoveDuplicateGroups to strip groups that are deterministic
> functions of a sibling column-ref base and recompute them in a projection
> inserted above the aggregate.

Two follow-up commits harden that generalization rather than extend it
further: **`33a23c258f`, "Batch group removals into single mapping pass"**
(the `ApplyGroupRemovals`/`group_binding_map` machinery replacing what was
presumably an earlier, less consolidated bookkeeping approach), and,
tellingly, **`56883845cd`, "Reject float/interval bases; split helpers"** —
the commit that introduces `BaseTypeAllowsCollapse` itself, arriving
*after* the FD-derived generalization was already in place. Read together,
this is direct evidence that the `0.0`/`-0.0` and interval bit-equality
hazard §21.2.4 walks through was not designed in from the start alongside
the functional-dependency mechanism — it was identified and patched in as a
dedicated follow-up commit, exactly the kind of subtle correctness gap that
tends to surface only once a generalization is already live and someone
specifically goes looking for what it might have broken.

The newest commit to the file follows the same pattern. **`ce0a563a789`,
"Clear stale group statistics when removing duplicate groups"**
(October 2026), replaced the `group_stats` assertion with a `clear()`
(§21.2.5). The assertion had assumed that no statistics exist this early
in the pipeline. The early filter-simplification run of the statistics
propagator showed that assumption to be false.

### 21.2.10 Summary: exactly how far "semantic" goes, restated precisely

To close with the precise, hedge-free answer the task's own framing asked
for: **`DUPLICATE_GROUPS` is not purely syntactic** — it does reach past
literal textual duplication into a genuine, if narrow, notion of semantic
equivalence, via the single-column functional-dependency argument in
§21.2.3. But its semantic reach stops at a specific, deliberately-drawn
boundary, and it's worth being precise about exactly where:

- It **does** collapse `GROUP BY x, x` (trivially — same binding).
- It **does** collapse `GROUP BY x, x - 1`, `GROUP BY x, CAST(x AS BIGINT)`,
  `GROUP BY x, x + 0` — any non-volatile, single-column
  function of an already-grouped bare column — as long as that base
  column's type (recursively, through any `STRUCT`/`LIST` nesting) contains
  no `FLOAT`, `DOUBLE`, or `INTERVAL`.
- It does **not** attempt to prove two *different* base columns are
  provably equal via some other mechanism (a `WHERE` clause, a check
  constraint) unless `UNUSED_COLUMNS`'s join-column-replacement rewrite
  (§21.2.7) has *already* physically unified them into the same binding
  before this pass ever runs — the FD reasoning always terminates at an
  actual bare `BOUND_COLUMN_REF`, never at "these two expressions are
  semantically the same value by some other argument."
- It does **not** attempt multi-column functional dependencies (`x + y`
  provably redundant given some other `f(x, y)`) — `CollectColumnRefBindings`'s
  single-binding requirement rules this out structurally, not as an
  unimplemented extension.
- It does **not** touch anything inside a `ROLLUP`/`CUBE`/`GROUPING SETS`
  aggregate (`grouping_sets.size() > 1`), nor anything a `GROUPING()` call
  depends on, regardless of how syntactically or semantically redundant the
  grouping list might otherwise be.
- It leaves constant (`IsFoldable()`) grouping keys entirely alone, and
  volatile (`!IsConsistent()`) derived expressions entirely
  alone, in both cases simply declining to reason about them rather than
  reasoning about them incorrectly.

That is a precisely-scoped, single-column, deterministic, bit-equality-aware
notion of "these two grouping keys mean the same partition" — considerably
more than pure text matching, considerably less than a general
expression-equivalence prover, and, per §21.2.7, wired into the fixed
pipeline specifically to catch the one concrete byproduct
(`UNUSED_COLUMNS`'s join-column replacement) that the pass's own author
evidently built it to catch in the first place.

---

## Closing synthesis

Both passes in this file earn their place in the pipeline the same way:
neither one is chasing a new optimization opportunity the binder's original
plan didn't already contain the seed of — `PROJECTION_PLACEMENT` only
relocates a computation that `PROJECTION_PULLUP` already, indiscriminately,
moved once before; `DUPLICATE_GROUPS` only collapses a duplicate that either
the original SQL wrote directly or `UNUSED_COLUMNS`'s join-column
replacement manufactured one pass earlier. Both are late, specialist,
statistics-or-structure-aware cleanup passes in a pipeline that, per §7.2,
gives every pass exactly one turn and no mechanism to ask an earlier pass to
run again — which is precisely why DuckDB needs a *dedicated* pass sitting
right after the specific earlier pass whose output it exists to clean up,
rather than folding either cleanup into that earlier pass directly. Reading
them side by side is a small, concrete lesson in how a fixed, non-fixpoint
optimizer pipeline actually gets to behave, in aggregate, as if it were
smarter than any one of its forty individual passes: not by any single pass
reasoning globally, but by careful, source-verifiable sequencing of narrow
passes, each one accounting for exactly what the pass before it could leave
behind.
