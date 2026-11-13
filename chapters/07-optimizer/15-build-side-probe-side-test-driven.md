# 7.15 `BUILD_SIDE_PROBE_SIDE`, test by test

## 7.15.1 What Chapter 8 §8.9 already nailed down

As Chapter 8 §8.9 established, picking a join *order* (DPhyp, Chapter 8's main
subject) and picking which of a join's two children becomes the **build** side
are two separate decisions made by two separate passes, run ten positions
apart in the pipeline (`JOIN_ORDER` at #17, `BUILD_SIDE_PROBE_SIDE` at #27 —
Chapter 7 §7.4). By DuckDB convention the **right-hand child of a logical
join is always the build side**; `BuildProbeSideOptimizer::TryFlipJoinChildren`
(`src/optimizer/build_probe_side_optimizer.cpp:235`) decides, for every join
node bottom-up, whether the plan it inherited from `JOIN_ORDER` already has the
cheaper-to-build side on the right, and swaps the two children (`FlipChildren`,
`build_probe_side_optimizer.cpp:66`) when it doesn't. Chapter 8 walked through
the core of the cost function, `BuildProbeSideOptimizer::GetBuildSize`
(`build_probe_side_optimizer.cpp:188`) — row width (with penalties for
`VARCHAR`/`LIST`/`ARRAY` columns and for sheer column count) times estimated
cardinality — and the one-line swap rule built on top of it:
`if (right_side_build_cost > left_side_build_cost) { swap = true; }`
(`build_probe_side_optimizer.cpp:291`). It also flagged, without going deep,
three refinements living in the same file: a DML row-id preference, a dynamic
join-filter bonus, and a right-deep tie-break penalty.

This section does not re-derive any of that. Its job is to hold the heuristic
up against real, runnable evidence — one C++ unit test that exercises the
optimizer's internal API directly (and, in doing so, exposes a fourth
refinement Chapter 8 had no room for at all: a **recursive-CTE carve-out** that
overrides the cost heuristic completely), two `.test` SQL files built
specifically to pin down the width-vs-cardinality tradeoff and the row-id/final-
projection tie-break, a `.test` file that puts exact numbers on the "dynamic
join filter" bonus using constants lifted straight from the source, and a
scan of the rest of `test/optimizer/joins/` to be honest about which of that
directory's 45 files are actually about build/probe selection and which only
sound like they are.

One point of bookkeeping before diving in: several of the tests below use
`EXPLAIN (FORMAT JSON) ... query II ... <REGEX>:...` rather than the
`query I nosort <label>` paired-EXPLAIN idiom used elsewhere in this book to
prove two plans are byte-identical. That pairing convention doesn't appear
anywhere in the build/probe-side test files this section covers — none of
them assert "these two EXPLAINs match each other," they assert "this one
EXPLAIN's *physical* child order matches this pattern." The `FORMAT JSON`
choice is deliberate and stated in-file (`wide_build_skinny_probe.test:49`):
JSON serializes an operator's children as an ordered array, so a regex over
the JSON text can check *which child came first* — i.e., which side is probe
(serialized first, conventionally the left/streamed child) and which is build
(serialized second, the right/materialized child) — independent of how
`EXPLAIN`'s default box-drawing renderer happens to lay two sibling boxes out
side by side on the page.

## 7.15.2 The C++ unit test: `test/optimizer/build_probe_side_optimizer.cpp`

This is a genuine unit test, not a `.test` SQL file — it `#include`s
`duckdb/optimizer/build_probe_side_optimizer.hpp` directly, builds a
`BuildProbeSideOptimizer` by hand, and calls `VisitOperator` on a plan it
built and mutated itself, which is a level of precision a SQL-level `EXPLAIN`
regex can't match: it can point at the exact `LogicalOperator*` before and
after the pass runs and assert pointer identity, not just "the plan looks
roughly like this."

The single `TEST_CASE`,
*"Build/probe optimization preserves correlated recursive domain build
sides"* (`[optimizer][build_probe_side]`), is entirely about a fourth
refinement to the heuristic that neither Chapter 8 nor §7.15.1 above has
mentioned yet, because it isn't really a *cost* refinement at all — it's a
correctness carve-out that **overrides the cost heuristic outright** for one
specific, synthetic plan shape.

### The setup: what a correlated recursive CTE's decorrelated plan looks like

The test's first query is a correlated subquery whose inner subquery is a
`WITH RECURSIVE`:

```sql
-- test/optimizer/build_probe_side_optimizer.cpp:48-60
SELECT c.v, (
    SELECT count(*) FROM (
        WITH RECURSIVE t(i) AS (
            SELECT i FROM (VALUES (0)) a(i), range(1000) b(j)
            UNION ALL
            SELECT i + 1 FROM t WHERE i < c.v
        )
        SELECT * FROM t
    )
)
FROM (VALUES (2), (3)) c(v)
```

The recursive term (`WHERE i < c.v`) references the outer column `c.v`, so this
is a *correlated* recursive CTE — Chapter 6's `FlattenDependentJoins`
decorrelation machinery has to run on it. Per the inline comment the test's
production code carries (`build_probe_side_optimizer.cpp:450-451`):
*"`FlattenDependentJoins` appends the correlation domain to the recursive
anchor"* — meaning the anchor term of the recursive CTE (`SELECT i FROM
(VALUES (0)) a(i), range(1000) b(j)`) gets a **new cross product** wrapped
around it after decorrelation, whose right-hand child is the *domain*: the set
of distinct correlated values (`c.v`'s values, here `{2, 3}`) that the
recursive computation has to be repeated for.

The test locates this structure directly rather than trusting `EXPLAIN` to
render it legibly:

```cpp
// test/optimizer/build_probe_side_optimizer.cpp:61-70
auto recursive_plan = PlanBuildProbeQuery(connection, correlated_query);
auto recursive_cte = FindBuildProbeOperator(*recursive_plan, LogicalOperatorType::LOGICAL_RECURSIVE_CTE);
REQUIRE(recursive_cte);
REQUIRE(recursive_cte->Cast<LogicalCTE>().correlated_columns.size() == 1);
recursive_plan->ResolveOperatorTypes();
auto anchor = std::move(recursive_cte->children[0]);
auto anchor_cross = anchor.get();
REQUIRE(anchor_cross->type == LogicalOperatorType::LOGICAL_CROSS_PRODUCT);
auto nested_cross = FindBuildProbeOperator(*anchor_cross->children[0], LogicalOperatorType::LOGICAL_CROSS_PRODUCT);
REQUIRE(nested_cross);
```

So the recursive CTE's anchor child is, itself, a `LOGICAL_CROSS_PRODUCT`
(`anchor_cross`) — the correlation-domain attachment — and *its* left child
contains a second, ordinary `LOGICAL_CROSS_PRODUCT` (`nested_cross`) which is
just the literal `(VALUES (0)) a(i), range(1000) b(j)` from the SQL. Two cross
products, nested, only one of which is the synthetic domain attachment.

Rather than trust the estimated cardinalities the planner happened to produce
for `(VALUES (0))` (1 row) and `range(1000)` (1000 rows) — which would make
this test fragile to unrelated changes in cardinality estimation — the test
pins them deliberately to values chosen to make the *naive* memory-footprint
heuristic want to swap both cross products:

```cpp
// test/optimizer/build_probe_side_optimizer.cpp:36-40
static void SetAsymmetricCrossProductEstimates(LogicalOperator &cross_product) {
	REQUIRE(cross_product.type == LogicalOperatorType::LOGICAL_CROSS_PRODUCT);
	cross_product.children[0]->SetEstimatedCardinality(1);
	cross_product.children[1]->SetEstimatedCardinality(1000);
}
```

Cardinality 1 on the left, 1000 on the right, applied to *both* cross
products (`anchor_cross` and `nested_cross`). Under the plain `GetBuildSize`
rule from §7.15.1, a cross product with these cardinalities has
`right_side_build_cost` (1000 rows' worth of row width) vastly exceeding
`left_side_build_cost` (1 row's worth), so `TryFlipJoinChildren` should want
to swap children on *both* nodes — flipping the domain attachment's own
children would put the single-row anchor-computation side on the right
(build) and the *correlation domain* on the left (probe).

### What the test proves: the domain attachment is exempt, the nested cross product is not

```cpp
// test/optimizer/build_probe_side_optimizer.cpp:82-89
SetAsymmetricCrossProductEstimates(*anchor_cross);
SetAsymmetricCrossProductEstimates(*nested_cross);
auto anchor_domain = anchor_cross->children[1].get();
auto nested_left = nested_cross->children[0].get();
BuildProbeSideOptimizer recursive_optimizer(*connection.context, *recursive_plan);
recursive_optimizer.VisitOperator(*recursive_plan);
REQUIRE(anchor_cross->children[1].get() == anchor_domain);
REQUIRE(nested_cross->children[1].get() == nested_left);
```

Two assertions, and they prove two *different* things:

1. **`anchor_cross->children[1].get() == anchor_domain`** — the domain
   attachment's own children did **not** move, despite the asymmetric
   cardinalities that should have triggered a swap. The correlation domain
   (`children[1]`, cardinality artificially set to 1000 — the *expensive*
   side under the naive rule) stayed exactly where it was.
2. **`nested_cross->children[1].get() == nested_left`** — the *nested*,
   ordinary cross product **did** flip: `nested_left` was captured as
   `nested_cross->children[0]` *before* optimization, and after
   `VisitOperator` runs, that same operator is now sitting at
   `children[1]`. The naive cost rule fired normally here.

The mechanism behind assertion 1 is not `GetRecursiveProbeSidePreference` (the
`RecursiveProbeSidePreference::{NONE,KEEP,SWAP}` machinery Chapter 8 didn't
have room for either — see §7.15.2.1 below) — that function only ever
inspects `LOGICAL_COMPARISON_JOIN` nodes, and both operators here are
`LOGICAL_CROSS_PRODUCT`. The real mechanism is upstream, in
`BuildProbeSideOptimizer::VisitOperator`'s handling of the recursive CTE node
itself:

```cpp
// src/optimizer/build_probe_side_optimizer.cpp:446-460
if (op.type == LogicalOperatorType::LOGICAL_RECURSIVE_CTE) {
	auto &cte = op.Cast<LogicalCTE>();
	auto domain_attachment = FindCorrelatedDomainAttachment(cte);
	if (domain_attachment) {
		// FlattenDependentJoins appends the correlation domain to the recursive anchor. Optimize both inputs while
		// preserving the orientation of this generated attachment.
		VisitOperatorChildren(*domain_attachment);
	} else {
		VisitOperator(*op.children[0]);
	}
	active_recursive_cte_indexes.push_back(cte.table_index);
	VisitOperator(*op.children[1]);
	active_recursive_cte_indexes.pop_back();
	return;
}
```

The load-bearing detail is `VisitOperatorChildren(*domain_attachment)` versus
`VisitOperator(*op.children[0])`. `VisitOperator` is the method whose `switch`
statement (§7.15.1's `case LOGICAL_CROSS_PRODUCT: TryFlipJoinChildren(op);`)
decides whether *this* node gets flip-tested. `VisitOperatorChildren` — the
base `LogicalOperatorVisitor`'s recursion helper — visits a node's *children*
without ever subjecting the node itself to that `switch`. So when
`FindCorrelatedDomainAttachment` (`build_probe_side_optimizer.cpp:397`) proves
the anchor's top cross product really is the synthetic domain attachment
(by walking down through any intervening projections and checking that the
right child's bindings are exactly the CTE's `correlated_columns`, in the
same order — `build_probe_side_optimizer.cpp:397-443`), the pass recurses into
`nested_cross` and `anchor_domain` *individually* — both of which remain
eligible for their own, ordinary flip decisions — but the cross product node
that ties them together, `anchor_cross`, is walked past without ever calling
`TryFlipJoinChildren` on it at all. It is structurally unreachable by the
cost heuristic, not merely "protected" by a preference check.

That is exactly what the two assertions show: `nested_cross` (an ordinary
child of the exempted node) still flips normally under adverse cardinalities;
`anchor_cross` (the exempted node itself) cannot flip no matter what
cardinalities are attached to it, because `VisitOperator` is never called on
it in the first place.

### The rest of the test: the exemption is narrow, not a blanket "never touch recursion" rule

The remaining two-thirds of the test case are control conditions, and they're
worth walking through because they establish exactly how narrow the carve-out
is:

```cpp
// test/optimizer/build_probe_side_optimizer.cpp:95-113
const string recursive_query = R"(
    WITH RECURSIVE t(i, j) AS (
        SELECT i, j FROM (VALUES (1)) a(i), range(1000) b(j)
        UNION ALL
        SELECT i, j FROM t WHERE false
    )
    SELECT * FROM t
)";
auto uncorrelated_plan = PlanBuildProbeQuery(connection, recursive_query);
auto uncorrelated_cte = FindBuildProbeOperator(*uncorrelated_plan, LogicalOperatorType::LOGICAL_RECURSIVE_CTE);
REQUIRE(uncorrelated_cte);
auto uncorrelated_cross =
    FindBuildProbeOperator(*uncorrelated_cte->children[0], LogicalOperatorType::LOGICAL_CROSS_PRODUCT);
REQUIRE(uncorrelated_cross);
SetAsymmetricCrossProductEstimates(*uncorrelated_cross);
auto uncorrelated_left = uncorrelated_cross->children[0].get();
BuildProbeSideOptimizer uncorrelated_optimizer(*connection.context, *uncorrelated_plan);
uncorrelated_optimizer.VisitOperator(*uncorrelated_plan);
REQUIRE(uncorrelated_cross->children[1].get() == uncorrelated_left);
```

This second `WITH RECURSIVE` is *not* correlated (`WHERE false` in the
recursive term references nothing from an outer query — there is no outer
query). `LogicalCTE::correlated_columns` is empty, so
`FindCorrelatedDomainAttachment` bails out on its very first line
(`build_probe_side_optimizer.cpp:398-400`: `if (cte.correlated_columns.empty())
return nullptr;`), the `if (domain_attachment)` branch is never taken, and
`VisitOperator(*op.children[0])` runs instead — the anchor's cross product
*is* subjected to the normal `switch`, and with the same asymmetric
cardinalities as before, it flips (`uncorrelated_left`, captured pre-optimization
at `children[0]`, ends up at `children[1]`). A recursive CTE, by itself, buys
no protection; only a *correlated* one with a real domain attachment does.

The final control is the plain baseline, with no CTE at all:

```cpp
// test/optimizer/build_probe_side_optimizer.cpp:115-122
auto ordinary_plan = PlanBuildProbeQuery(connection, "SELECT * FROM (VALUES (1)) a(i), range(1000) b(j)");
auto ordinary_cross = FindBuildProbeOperator(*ordinary_plan, LogicalOperatorType::LOGICAL_CROSS_PRODUCT);
REQUIRE(ordinary_cross);
SetAsymmetricCrossProductEstimates(*ordinary_cross);
auto ordinary_left = ordinary_cross->children[0].get();
BuildProbeSideOptimizer ordinary_optimizer(*connection.context, *ordinary_plan);
ordinary_optimizer.VisitOperator(*ordinary_plan);
REQUIRE(ordinary_cross->children[1].get() == ordinary_left);
```

Same flip, same assertion shape — this is the sanity check that proves the
flip mechanism itself works as expected on an unremarkable cross product,
which is what makes the *first* assertion (the one that does **not** flip)
meaningful rather than a fluke of a broken test.

Between the two decorrelation queries, the test also checks something this
book cares about deeply: that the exemption didn't just avoid a crash, it
avoided a *silent correctness regression*. After running the optimizer on the
correlated plan, the test actually executes the original query and checks
real output:

```cpp
// test/optimizer/build_probe_side_optimizer.cpp:90-93
auto correlated_result = connection.Query(correlated_query);
REQUIRE_NO_FAIL(*correlated_result);
CHECK_COLUMN(correlated_result, 0, {2, 3});
CHECK_COLUMN(correlated_result, 1, {3000, 4000});
```

`c.v = 2` recurses `i` from `0` up to `1` inclusive across `1000` starting
rows (`i` from `(VALUES(0)), range(1000)` — 1000 rows at `i=0`, then the
recursive step adds 1 while `i < 2`, so two more generations of 1000 rows
each survive before the anchor's `i=0` rows fail `i<2`... the exact row
count, `3000`, is what the test hard-asserts, and it is exactly what would go
*wrong* first if the correlation domain's position in the plan tree were
disturbed — decorrelation's whole mechanism depends on the domain relation
being where `FlattenDependentJoins` put it, since that position is what lets
the recursive evaluator know which per-value slice of the working table
belongs to which correlated input row. A cost-based pass that "helpfully"
flipped this cross product because one side's cardinality estimate happened
to look big would not just make the plan slower — per this test, it would
require the recursive step's other decorrelation-era invariants to keep
holding regardless of orientation, which is precisely the risk the
`domain_attachment` exemption is written to foreclose before it can happen.

### 7.15.2.1 What this unit test does *not* exercise, for completeness

`build_probe_side_optimizer.cpp` (the source file, not the test) contains a
second, independent recursive-CTE-aware mechanism that this specific unit
test never reaches: `GetRecursiveProbeSidePreference`
(`build_probe_side_optimizer.cpp:357`), returning
`RecursiveProbeSidePreference::{NONE, KEEP, SWAP}`, consulted at the very top
of `TryFlipJoinChildren` — before any cost computation happens at all — but
**only for `LOGICAL_COMPARISON_JOIN` nodes**:

```cpp
// src/optimizer/build_probe_side_optimizer.cpp:357-360
RecursiveProbeSidePreference BuildProbeSideOptimizer::GetRecursiveProbeSidePreference(const LogicalOperator &op) const {
	if (active_recursive_cte_indexes.empty() || op.type != LogicalOperatorType::LOGICAL_COMPARISON_JOIN) {
		return RecursiveProbeSidePreference::NONE;
	}
```

This is a *performance* heuristic, not the correctness carve-out above: if an
ordinary comparison join somewhere inside a recursive CTE's recursive term
references the CTE's own working table (checked via
`ContainsActiveRecursiveReference`, walking down for a `LOGICAL_CTE_REF` whose
`cte_index` is on the currently-active stack — `active_recursive_cte_indexes`,
pushed and popped exactly around the recursive term's `VisitOperator` call in
the snippet above), and the *other* side of that join is safe to keep
resident across iterations (no `RIGHT`/`OUTER`/`RIGHT_SEMI`/`RIGHT_ANTI` join
type that would `PropagatesBuildSide`, and no
`LOGICAL_DELIM_GET`/`LOGICAL_DELIM_JOIN`/`LOGICAL_DEPENDENT_JOIN` beneath it —
`ContainsCorrelationSensitiveOperators`), then the pass prefers whichever
orientation lets that non-recursive side stay the build side across
iterations, `KEEP`ing or `SWAP`ping to get there rather than reconsulting
`GetBuildSize` at all. Because this unit test's decorrelated plan uses cross
products, not comparison joins, at every point where a recursive reference
sits alongside a domain, this second mechanism is real, present in the same
file, and — as far as this book's evidence goes — untested by anything in
`test/optimizer/build_probe_side_optimizer.cpp` or, on this search, by any
`.test` file either. It's flagged here rather than guessed at, per this
book's sourcing policy (front matter, "sources and how they're weighted").

## 7.15.3 `wide_build_skinny_probe.test`: proving width beats row count

This is the one `.test` file whose name is a direct, on-point claim about the
heuristic, and it is worth taking at face value and then verifying with real
arithmetic from `GetBuildSize` itself.

```sql
-- test/optimizer/joins/wide_build_skinny_probe.test:5-45
statement ok
create table skinny as select range a, range b, range c from range(10000);

statement ok
create table wide as select
	range pk,
	(range::VARCHAR || '1111') a,
	(range::VARCHAR || '2222') b,
	-- ... 32 more VARCHAR columns, listed exhaustively in the file ...
	(range::VARCHAR || '0044') kk from range(8000);

# wide should be the probe side (serialized first), skinny should be on the build side
# use json format so the child order is checked independently of the rendered box layout
query II
explain (format json) select * from wide w, skinny s where w.pk=s.a;
----
physical_plan	<REGEX>:.*wide.*skinny.*
```

`skinny` has three `BIGINT` columns and **10,000 rows** — more rows than
`wide`'s **8,000**. `wide` has one `BIGINT` primary key and **34 `VARCHAR`**
columns. If the heuristic were "the side with fewer rows is the build side" —
the intuition a reader might reach for before reading `GetBuildSize` — `wide`
(8,000 < 10,000) would look like the obvious build-side candidate. The test's
comment says the opposite is correct: `wide` should stay the **probe** side
and `skinny` — the side with *more* rows — should be the **build** side. The
file's own name states the generalization directly: it isn't row count that
decides this, it's **width**.

### Doing the arithmetic `GetBuildSize` actually does

`GetBuildSize` (`build_probe_side_optimizer.cpp:188-221`) builds a real
`TupleDataLayout` for each side's row shape plus one appended `HASH` column,
applies the `VARCHAR`/`LIST`/`ARRAY` and per-column penalties, and adds a flat
hash-table overhead — reproduced here in full since Chapter 8 only showed it
elided:

```cpp
// src/optimizer/build_probe_side_optimizer.cpp:188-221
double BuildProbeSideOptimizer::GetBuildSize(vector<LogicalType> types, const idx_t cardinality) {
	// Row width in the hash table
	types.push_back(LogicalType::HASH);
	auto tuple_layout = TupleDataLayout();
	tuple_layout.Initialize(types, TupleDataValidityType::CAN_HAVE_NULL_VALUES);
	auto row_width = tuple_layout.GetRowWidth();

	for (const auto &type : types) {
		TypeVisitor::VisitReplace(type, [&](const LogicalType &visited_type) {
			// Penalty for variable-size types (we don't have statistics here yet)
			switch (visited_type.InternalType()) {
			case PhysicalType::VARCHAR:
				row_width += 8;
				break;
			case PhysicalType::LIST:
			case PhysicalType::ARRAY:
				row_width += 32;
				break;
			default:
				break;
			}

			// Penalty for number of (recursive) columns
			row_width += COLUMN_COUNT_PENALTY;

			return visited_type;
		});
	}

	// There is also a cost of NextPowerOfTwo(count * 2) * sizeof(ht_entry_t) per tuple in the hash table
	// This is a not a smooth cost function, so instead we do the average, which is ~3 * sizeof(ht_entry_t)
	row_width += 3 * sizeof(ht_entry_t);

	return MaxValue<double>(static_cast<double>(row_width) * static_cast<double>(cardinality), sizeof(ht_entry_t));
}
```

Note the last line. The multiply is done in `double`, and the result is
never smaller than `sizeof(ht_entry_t)` (8). Why does this matter? A
cardinality estimate can be 0. Then both sides could cost 0, and the swap
test `right > left` would compare 0 with 0. Every non-empty hash table costs
at least one table slot, so the floor keeps the cost above zero. For the
tables in this chapter the floor never applies: their costs are far above 8.

Every constant here is checkable: `COLUMN_COUNT_PENALTY = 2`
(`src/include/duckdb/optimizer/build_probe_side_optimizer.hpp:31`),
`ht_entry_t` (`src/include/duckdb/execution/ht_entry.hpp:27-97`) wraps exactly
one `hash_t value` member, so `sizeof(ht_entry_t) == 8`, and
`TupleDataLayout`'s row-level validity mask is a
`TemplatedValidityMask<uint8_t>` (`tuple_data_layout.hpp:23`) — one bit per
column, rounded up to whole bytes — so its size in bytes is
`ceil(column_count / 8)` (`tuple_data_layout.cpp:50`,
`ValidityBuffer::EntryCount`).

**`skinny`** — 3 `BIGINT` columns + the appended `HASH` column = 4 columns,
all fixed-width 8 bytes:

- Validity flag bytes: `ceil(4 / 8) = 1`
- Data bytes: `4 columns × 8 bytes = 32`
- `TupleDataLayout` row width: `1 + 32 = 33`
- Column-count penalty: `4 columns × 2 = 8` (no `VARCHAR`, so no width penalty)
- Hash-table overhead: `3 × 8 = 24`
- **Total row width: `33 + 8 + 24 = 65` bytes**
- **`GetBuildSize(skinny, 10000) = 65 × 10000 = 650,000`**

**`wide`** — 1 `BIGINT` (`pk`) + 34 `VARCHAR` columns + the appended `HASH`
column = 36 columns. `VARCHAR`'s physical size is `sizeof(string_t) == 16`
bytes (`src/common/types.cpp:484-485`):

- Validity flag bytes: `ceil(36 / 8) = 5`
- Data bytes: `pk (8) + 34 × string_t (16) = 8 + 544 = 552`, plus the `HASH`
  column's 8 bytes: `552 + 8 = 560`
- `TupleDataLayout` row width: `5 + 560 = 565`
- Column-count penalty: `36 columns × 2 = 72`
- `VARCHAR` width penalty: `34 columns × 8 = 272`
- Hash-table overhead: `3 × 8 = 24`
- **Total row width: `565 + 72 + 272 + 24 = 933` bytes**
- **`GetBuildSize(wide, 8000) = 933 × 8000 = 7,464,000`**

`wide`'s per-row cost (933 bytes) is roughly **14× `skinny`'s** (65 bytes) —
driven almost entirely by 34 `string_t` columns at 16 bytes apiece plus their
8-byte penalty, versus `skinny`'s 3 plain `BIGINT`s — and that ratio survives
`wide`'s 20%-smaller row count by more than an order of magnitude:
`7,464,000` versus `650,000`, a difference of roughly **11.5×** in total
estimated build-side footprint. The FROM clause in this query already lists
`wide` first (so it starts, by convention, as the left/probe child before any
flip is even considered) and `skinny` second (the right/build child by
convention). Plugging into `TryFlipJoinChildren`'s swap test
(`build_probe_side_optimizer.cpp:291`) with `right_side_build_cost =
GetBuildSize(skinny, 10000) = 650,000` and `left_side_build_cost =
GetBuildSize(wide, 8000) = 7,464,000`:

```
right_side_build_cost > left_side_build_cost
      650,000        >     7,464,000     →  false
```

No swap. Both children in this query happen to have no residual filters
(`IsFiltering` on either scan returns `false`, so `DynamicFilterBuildBonus`
— §7.15.5 — contributes nothing here), and neither side has joins beneath it
(so the `PREFER_RIGHT_DEEP_PENALTY` tie-break Chapter 8 mentioned doesn't
apply either) — this test isolates the plain width-times-cardinality
comparison with nothing else in play. The natural FROM-clause order already
happened to be optimal, and the test's real value is proving the pass
*correctly declines to disturb it* even though a row-count-only reading of
"smaller is build" would have called for exactly the opposite arrangement.
§7.15.6 below constructs the mirror-image query — the one where the natural
order *is* wrong — using these same two tables and the same arithmetic, to
show the actual flip happening.

## 7.15.4 `prefer_final_projected_columns_on_probe_side.test`: a distinct, narrower heuristic

This file's name promises something conceptually different from "smaller
build side wins": a preference for keeping, on the probe side, whichever
child's columns are the ones that actually survive to the operator that
consumes the join's output. Reading `build_probe_side_optimizer.cpp` closely
shows this preference is real, but considerably narrower in its *current*
implementation than the test's name suggests — and the test itself is
constructed in a way that only exercises the narrow case, which is worth
being precise about.

```sql
-- test/optimizer/prefer_final_projected_columns_on_probe_side.test:5-34
statement ok
create table t(ts_start timestamptz, ts_stop timestamptz, id text);

statement ok
with dates as (
  select '2023-01-01'::timestamp + i * interval '1 DAY' as x
    from generate_series(0, 999) as t(i)
),
ids as (
  select 'id_' || lpad(i::text, 4, '0') as y
  from generate_series(0, 999) as t(i)
)
insert into t(ts_start, ts_stop, id)
  select d.x, null, i.y from dates d, ids i;

# the probe side (serialized first) is the sequential scan, the build side is the projection
# whose child is the window over the sequential scan
# use json format so the child order is checked independently of the rendered box layout
query II
explain (format json) update t as this
set ts_stop = next.ts_start_next
from (
select id, ts_start, LEAD(ts_start) over (partition by id order by ts_start)
	as ts_start_next
from t
) as next
where this.id=next.id and this.ts_start=next.ts_start;
----
physical_plan	<REGEX>:.*HASH_JOIN.*SEQ_SCAN.*PROJECTION.*WINDOW.*SEQ_SCAN.*
```

`t` has exactly 1000 rows (`generate_series(0, 999)` cross-joined 1000×1000
then inserted only where `d.x` and `i.y` line up positionally — the `INSERT`
is really just zipping `dates` and `ids` by construction, so `t` ends up with
1000 rows). The `UPDATE ... FROM` rewrites into a join between `this` (a
direct scan of `t`, aliased) and `next` (a derived table that scans `t` again
and computes a `LEAD(...) OVER (PARTITION BY id ORDER BY ts_start)` window
function over it) — **both sides scan the same 1000-row table**, so their
estimated cardinalities are identical: `1000 == 1000`.

That equality is exactly the condition `TryFlipJoinChildren` checks for its
final, narrowest tie-break — one Chapter 8 didn't cover at all:

```cpp
// src/optimizer/build_probe_side_optimizer.cpp:295-312
// swap for preferred on probe side
if (rhs_cardinality == lhs_cardinality && !preferred_on_probe_side.empty()) {
	// inspect final bindings, we prefer them on the probe side
	auto bindings_left = left_child.GetColumnBindings();
	auto bindings_right = right_child.GetColumnBindings();
	auto bindings_in_left = ComputeOverlappingBindings(bindings_left, preferred_on_probe_side);
	auto bindings_in_right = ComputeOverlappingBindings(bindings_right, preferred_on_probe_side);
	// (if the sides are planning to be swapped AND
	// if more projected bindings are in the left (meaning right/build side after the swap)
	// then swap them back. The projected bindings stay in the left/probe side.)
	// OR
	// (if the sides are planning not to be swapped AND
	// if more projected bindings are in the right (meaning right/build)
	// then swap them. The projected bindings are swapped to the left/probe side.)
	if ((swap && bindings_in_left > bindings_in_right) || (!swap && bindings_in_right > bindings_in_left)) {
		swap = !swap;
	}
}
```

This only fires on an exact cardinality tie — which is precisely the
situation this `UPDATE` produces — and it re-decides the swap based on which
side holds more bindings from `preferred_on_probe_side`. The name of that
member variable is where the "final projected columns" framing gets earned:
but in this checkout, `preferred_on_probe_side` is populated **exactly once**,
in the constructor, and **exclusively** with row-id bindings:

```cpp
// src/optimizer/build_probe_side_optimizer.cpp:54-64
BuildProbeSideOptimizer::BuildProbeSideOptimizer(ClientContext &context, LogicalOperator &op) : context(context) {
	vector<ColumnBinding> updating_columns, current_op_bindings;
	auto bindings = op.GetColumnBindings();
	vector<ColumnBinding> row_id_bindings;
	// If any column bindings are a row_id, there is a good chance the statement is an insert/delete/update statement.
	// As an initialization step, we travers the plan and find which bindings are row_id bindings.
	// When we eventually do our build side probe side optimizations, if we get to a join where the left and right
	// cardinalities are the same, we prefer to have the child with the rowid bindings in the probe side.
	GetRowidBindings(op, preferred_on_probe_side);
	op.ResolveOperatorTypes();
}
```

`GetRowidBindings` (`build_probe_side_optimizer.cpp:31-52`) walks the *entire*
plan looking for any `LogicalGet` whose `column_ids` include a
`col_id.IsRowIdColumn()` entry — which is exactly what an `UPDATE`/`DELETE`
target scan carries, since the physical update operator needs each row's
storage row-id to know which physical row to overwrite. In this query, that's
`this` — the scan feeding the `UPDATE`'s target, not `next` (the side that
computes the `LEAD` window function and produces the *new* `ts_stop` value,
but has no reason to carry a row-id).

So the mechanism the test's title generalizes to "final projected columns"
is, concretely, in this source tree: *whichever side of a cardinality tie
carries the row-id(s) an enclosing `INSERT`/`UPDATE`/`DELETE` needs at the
top of the plan wins the probe side.* Row-ids are a paradigm case of exactly
the kind of column the general framing describes — needed by an operator far
above the join, not needed as a join key, and expensive to be forced to carry
as extra payload width inside a build-side hash table when they could
instead simply be read straight off the probe side's streamed input chunk
without ever being hashed or copied into a payload row at all. But as
written, the *implementation* is narrower than "any column that survives to
the final projection" — it is specifically rowid-bindings-only, gated behind
an exact-cardinality-tie precondition. A join between two unrelated,
equal-cardinality tables where the *non-DML* final `SELECT` list happened to
only reference one side's columns would **not**, on this reading of the
source, get any preferential probe-side treatment from this code path —
there's no general "final projection reachability" bindings set being built
anywhere else in this file. This is exactly the kind of distinction this
book's sourcing policy asks to be stated plainly rather than smoothed over:
the test's plain-English name is a fair *motivating description* of the
mechanism, but the mechanism it actually tests is the DML-row-id special
case, not a fully general final-column-affinity heuristic.

The `EXPLAIN` regex confirms the prediction this analysis makes:
`HASH_JOIN.*SEQ_SCAN.*PROJECTION.*WINDOW.*SEQ_SCAN` — the join's first
serialized child (probe) is a bare `SEQ_SCAN` (this — the row-id-carrying
scan), and its second child (build) is a `PROJECTION` over a `WINDOW` over a
second `SEQ_SCAN` (next — the side computing `LEAD`). Without the row-id
tie-break, a coin-flip on an exact cardinality tie could just as easily have
put the more expensive `PROJECTION`+`WINDOW` subtree on the probe side
instead — cheap to build (the raw `GetBuildSize` cost is roughly the same on
both sides here, since both ultimately scan the same base table's columns
plus one extra window-function output column) but this test isn't really
about that cost being different; it's about what breaks the tie when
`GetBuildSize` alone can't decide.

## 7.15.5 `join_filter_build_side.test`: the dynamic-filter bonus, with the real threshold constants

Chapter 8 §8.9 named `DynamicFilterBuildBonus` and
`JoinFilterBuildSideHeuristics` in one sentence and moved on. This file is
built specifically to exercise that bonus, and its two tables' row counts
line up suspiciously well with the bonus function's own named constants —
worth checking directly rather than assumed.

```sql
-- test/optimizer/joins/join_filter_build_side.test:8-45
statement ok
CREATE TABLE jf_probe AS SELECT range::INTEGER AS id FROM range(1000000);

statement ok
CREATE TABLE jf_build AS SELECT
	range::INTEGER AS id,
	range < 10 AS keep,
	(range::VARCHAR || '1111') a,
	(range::VARCHAR || '2222') b,
	(range::VARCHAR || '3333') c,
	(range::VARCHAR || '4444') d,
	(range::VARCHAR || '5555') e,
	(range::VARCHAR || '6666') f,
	(range::VARCHAR || '7777') g,
	(range::VARCHAR || '8888') h,
	(range::VARCHAR || '9999') i,
	(range::VARCHAR || '0000') j
FROM range(100000);

statement ok
ANALYZE jf_probe;

statement ok
ANALYZE jf_build;

statement ok
PRAGMA explain_output='physical_only';

# jf_build is wide enough that the normal build-side cost prefers jf_probe.
# The filter on jf_build can generate a dynamic filter for the large jf_probe scan,
# so jf_build should remain on the build side.
query II
EXPLAIN SELECT *
FROM jf_probe p
JOIN jf_build b ON p.id = b.id
WHERE b.keep;
----
physical_plan	<REGEX>:.*HASH_JOIN.*jf_probe.*jf_build.*
```

`jf_probe` has **1,000,000** rows and one narrow `INTEGER` column;
`jf_build` has **100,000** rows, ten `VARCHAR` columns, and a `keep` column
defined as `range < 10` — exactly **10** of its 100,000 rows satisfy it. The
`FROM`-clause order already puts `jf_probe` on the left (probe, by
convention) and `jf_build` on the right (build, by convention) — the same
starting orientation as §7.15.3's `wide_build_skinny_probe.test`. The test's
own comment states outright that *without* the dynamic-filter mechanism, the
plain `GetBuildSize` comparison would want to flip this: `jf_build`'s ten
`VARCHAR` columns make its per-row cost far higher than `jf_probe`'s single
`INTEGER` column, echoing §7.15.3's `wide`/`skinny` case almost exactly. The
`WHERE b.keep` predicate is what changes the outcome, via
`DynamicFilterBuildBonus` (`build_probe_side_optimizer.cpp:132-155`),
reproduced here in full since Chapter 8 only named it:

```cpp
// src/optimizer/build_probe_side_optimizer.cpp:132-155
static double DynamicFilterBuildBonus(LogicalComparisonJoin &join, const idx_t probe_idx, const idx_t build_idx,
                                      const idx_t build_cardinality) {
	if (!JoinFilterPushdownOptimizer::IsFiltering(join.children[build_idx])) {
		return 1;
	}

	idx_t max_target_cardinality = 0;
	for (auto &cond : join.conditions) {
		if (!cond.IsComparison() || cond.GetComparisonType() != ExpressionType::COMPARE_EQUAL) {
			continue;
		}
		auto &probe_expr = probe_idx == 0 ? cond.GetLHS() : cond.GetRHS();
		max_target_cardinality =
		    MaxValue(max_target_cardinality, MaxDynamicFilterTargetCardinality(*join.children[probe_idx], probe_expr));
	}
	if (max_target_cardinality < JoinFilterBuildSideHeuristics::MIN_FILTER_TARGET_CARDINALITY) {
		return 1;
	}
	if (build_cardinality > 0 &&
	    build_cardinality > max_target_cardinality / JoinFilterBuildSideHeuristics::MAX_BUILD_TO_TARGET_RATIO) {
		return 1;
	}
	return static_cast<double>(max_target_cardinality) / static_cast<double>(MaxValue<idx_t>(build_cardinality, 1));
}
```

And the constants it reads from
(`src/optimizer/build_probe_side_optimizer.cpp:25-29`):

```cpp
struct JoinFilterBuildSideHeuristics {
	static constexpr idx_t MIN_FILTER_TARGET_CARDINALITY = 1000000;
	static constexpr idx_t MAX_BUILD_TO_TARGET_RATIO = 64;
	static constexpr idx_t SEMI_JOIN_FILTER_TARGET_RATIO = 3;
};
```

Walking through the function for this exact query:

1. `IsFiltering(join.children[build_idx])` — the build side is `jf_build`,
   and `WHERE b.keep` gets pushed down (`FILTER_PUSHDOWN`, Chapter 7 §7.4
   pass #6, runs at position #6, well before `BUILD_SIDE_PROBE_SIDE` at #27)
   into a `table_filters` entry on `jf_build`'s `LogicalGet`, so
   `IsFiltering` — which for a `LOGICAL_GET` checks exactly
   `get.table_filters.HasFilters()`
   (`src/optimizer/join_filter_pushdown_optimizer.cpp:328-331`) — returns
   `true`. The function does not bail out at the first guard.
2. **`max_target_cardinality`** is computed by walking the *probe* side
   (`jf_probe`) looking for a pushdown target reachable from the equality
   condition's probe-side expression (`p.id`) — which resolves straight to
   `jf_probe`'s own `LogicalGet`, whose estimated cardinality is its full row
   count: **1,000,000**.
3. **The first threshold check**: `max_target_cardinality <
   MIN_FILTER_TARGET_CARDINALITY` → `1,000,000 < 1,000,000` → **false** — this
   query's probe-side row count was deliberately chosen to sit exactly at,
   not comfortably above, the `1,000,000`-row constant that gates whether a
   dynamic filter is even considered worth generating at all. Below this
   threshold, the bonus function returns `1` (no bonus, no behavior change)
   regardless of everything else — a small scan simply isn't worth building a
   runtime min/max filter for.
4. **The second threshold check** (the "is the build side itself cheap
   enough to be worth trusting as a filter domain" guard): `build_cardinality
   > max_target_cardinality / MAX_BUILD_TO_TARGET_RATIO` → is `jf_build`'s
   *post-filter* estimated cardinality (driven down toward the true value of
   10 by the `ANALYZE jf_build` statement the test runs explicitly, giving
   the cardinality estimator real column statistics on `keep` to work from)
   greater than `1,000,000 / 64 = 15,625`? With a filtered cardinality on the
   order of 10, this is comfortably false — the build side stays a candidate
   for the bonus.
5. **The bonus itself**: `max_target_cardinality / max(build_cardinality, 1)`
   — with `max_target_cardinality = 1,000,000` and a filtered
   `build_cardinality` on the order of `10`, this divides `right_side_build_cost`
   by a factor on the order of **100,000×** before it's ever compared against
   `left_side_build_cost` in the swap test (`right_side_build_cost /=
   DynamicFilterBuildBonus(join, 0, 1, rhs_cardinality);` —
   `build_probe_side_optimizer.cpp:261`). Whatever `jf_build`'s raw,
   ten-`VARCHAR`-column `GetBuildSize` cost actually is (large, by the same
   arithmetic §7.15.3 walked through), dividing it by ~100,000 pushes it far
   below `jf_probe`'s cost (`GetBuildSize` on a single-`INTEGER`-column,
   1,000,000-row table), and the swap test's inequality — the same one-line
   rule from §7.15.1 — comes out false. `jf_build` stays the build side.

The `EXPLAIN` regex, `HASH_JOIN.*jf_probe.*jf_build.*`, confirms exactly
this: `jf_probe` serialized first (still the probe side), `jf_build` second
(still the build side) — no flip, exactly as the test's comment predicts,
and exactly what the bonus arithmetic above predicts independently. The
underlying physical payoff, not directly asserted by this `.test` file but
stated as the motivating reason in its own comment and in `JOIN_FILTER_PUSHDOWN`'s
own inline comment (Chapter 7 §7.4, pass #42): keeping the narrow,
ten-row-after-filtering `keep`-domain on the build side lets that domain's
min/max be pushed down as a *dynamic, execution-time* range filter onto
`jf_probe`'s scan — pruning the 1,000,000-row probe scan down to just the
rows that could possibly match one of `jf_build`'s ten surviving `id`s,
before a single hash probe happens. Flipping the join to satisfy the raw
row-width heuristic alone would still produce a *correct* plan, but it would
throw away a large-scan-pruning opportunity purely because ten `VARCHAR`
columns look expensive in isolation — which is exactly the miscalibration
`DynamicFilterBuildBonus` exists to correct.

## 7.15.6 A full worked example: the same two tables, the other way around

§7.15.3 showed a query where the natural, FROM-clause-order plan was already
optimal and `BUILD_SIDE_PROBE_SIDE` correctly left it alone. To see an actual
flip happen, with the same two tables and the exact same per-row arithmetic,
it only takes reversing which one is listed first:

```sql
CREATE TABLE skinny AS SELECT range a, range b, range c FROM range(10000);
CREATE TABLE wide AS SELECT range pk, /* 34 VARCHAR columns */ ... FROM range(8000);

SELECT * FROM skinny s, wide w WHERE s.a = w.pk;
```

The only change from §7.15.3's query is which table is written first in the
`FROM` clause. With a single equi-join and no other relations to reorder,
`JOIN_ORDER` (Chapter 8) has nothing to search over and hands
`BUILD_SIDE_PROBE_SIDE` a plan whose left/right children mirror the SQL text
exactly: `skinny` on the left, `wide` on the right. By the right-child-is-
build convention, that makes **`wide` the build side** — the *wrong* choice,
per the same arithmetic computed in §7.15.3.

**Before `BUILD_SIDE_PROBE_SIDE` runs** (the plan `JOIN_ORDER` hands
forward — `wide` on the right/build side purely because it was written
second in the `FROM` clause):

```
┌───────────────────────────────┐
│ PhysicalHashJoin               │  s.a = w.pk
└──────┬────────────────┬───────┘
       │                │
┌──────▼──────┐  ┌──────▼──────────┐
│ SEQ_SCAN     │  │ SEQ_SCAN         │
│ skinny       │  │ wide             │
│ (probe)      │  │ (build)          │
│ 3×BIGINT     │  │ 1×BIGINT+34×VCHR │
│ ~10,000 rows │  │ ~8,000 rows      │
└─────────────┘  └─────────────────┘
```

`GetBuildSize` on the current build side (`wide`, right child):
`933 bytes/row × 8,000 rows = 7,464,000`.
`GetBuildSize` on the current probe side, evaluated as a build-side
*candidate* (`skinny`, left child): `65 bytes/row × 10,000 rows = 650,000`.
`TryFlipJoinChildren`'s swap test:

```
right_side_build_cost > left_side_build_cost
      7,464,000       >      650,000        →  true  →  swap
```

`FlipChildren` (`build_probe_side_optimizer.cpp:66-99`) runs:
`std::swap(op.children[0], op.children[1])`, the join's `join_type` is
inverted via `InverseJoinType` (a no-op here — this is a plain `INNER` join,
and `INNER`'s inverse is `INNER`,
`src/common/enums/join_type.cpp:30-31`), and every comparison condition is
rewritten with its two sides swapped and its comparison operator flipped
(`FlipComparisonExpression`) so that `s.a = w.pk` becomes `w.pk = s.a` —
the same predicate, restated so its left-hand side matches the operator's
new left (now-probe) child. `left_projection_map`/`right_projection_map` are
swapped too, so whichever columns each side was already set to project stay
attached to the correct physical side after the swap.

**After `BUILD_SIDE_PROBE_SIDE` runs:**

```
┌───────────────────────────────┐
│ PhysicalHashJoin               │  w.pk = s.a
└──────┬────────────────┬───────┘
       │                │
┌──────▼──────────┐  ┌──▼──────────┐
│ SEQ_SCAN         │  │ SEQ_SCAN     │
│ wide             │  │ skinny       │
│ (probe)          │  │ (build)      │
│ 1×BIGINT+34×VCHR │  │ 3×BIGINT     │
│ ~8,000 rows      │  │ ~10,000 rows │
└─────────────────┘  └─────────────┘
```

This is, byte for byte, the same physical shape §7.15.3's real
`wide_build_skinny_probe.test` asserts for its own (already-natural-ordered)
query — `wide` streamed as the probe side, `skinny`'s narrow rows hashed into
the build-side table — confirming that `BUILD_SIDE_PROBE_SIDE` converges on
the *same* optimal physical arrangement regardless of which of the two
syntactically equivalent `FROM`-clause orderings a query happened to be
written in. `PhysicalHashJoin`'s `Sink` phase now buffers `skinny`'s 10,000
narrow rows (a hash table on the order of `65 × 10,000 ≈ 63` KiB of row
storage, before the hash table's own bucket-pointer array) once, and
`wide`'s 8,000 much-wider rows stream through as probes without ever being
copied into hash-table payload storage at all — exactly the outcome the
per-row-width arithmetic in §7.15.3 predicts is cheaper, arrived at here via
an actual flip rather than by the natural order already being correct.

## 7.15.7 Scanning the rest of `test/optimizer/joins/`: what's actually about build/probe, and what only sounds like it

`test/optimizer/joins/` holds 45 files in total, counting its
`left_join_reordering/` subdirectory. Grepping
the directory for build/probe-side vocabulary in a file's name, its
`# description:` line, or its body turns up four matches. Only two of them
test the pass: `join_filter_build_side.test` (§7.15.5) and
`wide_build_skinny_probe.test` (§7.15.3). The other two,
`left_join_reordering/complex_left_joins.test:6` and
`left_join_reordering/test_reordering_left_joins.test:8`, do the opposite.
They turn the pass off with `pragma disabled_optimizers='build_side_probe_side';`.
This keeps the child order that `JOIN_ORDER` chose, so the tests can check
the join order alone. Every other file whose name might suggest cardinality-and-build-
side relevance turns out, on inspection, to be squarely about **join-order**
cardinality estimation — DPhyp's DP-table cost feed, Chapter 8's subject —
rather than which side of an already-chosen join becomes build or probe.
Being precise about that distinction rather than forcing a build/probe
framing onto files that don't support one:

- **`better_ce_estimates_for_bad_join_conditions.test`** — despite its
  generic-sounding name, its own `# description:` states the point plainly:
  *"In the join order optimizer queries need to have the correct bindings."*
  Its query joins three tables on a mix of an inequality (`t1.name_ !=
  t2.name_`) and two equalities, and asserts only that the physical plan does
  **not** contain a `NESTED_LOOP_JOIN` (`<!REGEX>:.*NESTED_LOOP_JOIN.*`) — a
  claim about **join-order/algorithm selection** (getting a cardinality
  estimate for the inequality-joined pair right enough that DPhyp doesn't
  back itself into a plan shape needing a nested-loop fallback), not about
  which child of any one join becomes the build side.
- **`denominator_cardinality_estimation.test`** — its own description:
  *"Join cardinality denominator assembly and predicate grouping."* Both its
  scenarios (disconnected join components merged by a cross product; a
  union-find-style component-merging edge case) assert row-count estimates
  embedded directly in `EXPLAIN` output (`~5.000 rows`) and a final
  `SELECT count(*)` value (`5000`) — this is DPhyp's cost-model input being
  checked for correctness, not build/probe placement.
- **`fkpk_composite_key_ce.test`** — its description: *"Cardinality
  estimation for composite FK/PK joins with transitive equality conditions,"*
  built explicitly to *"mimic TPC-H Q9 structure"* (the file's own comment
  block walks through exactly how `partsupp`/`supplier`/`lineitem`-shaped
  composite keys are constructed so the join count comes out to a provable
  `600,000`). Every assertion is an estimated-row-count regex or a `count(*)`
  check — again, join-order cardinality feed, not build-side selection.
- **`get_cardinality_from_limit.test_slow`** — description: *"when a limit is
  encountered, use the limit to know the cardinality."* Checks that a
  `LIMIT 10000` subquery's cardinality estimate (used by the join-order cost
  model) reflects the limit rather than the underlying 10-million-row table's
  full size (`logical_opt` regex: `SEQ_SCAN.*LIMIT.*SEQ_SCAN`) — cardinality
  estimation feeding join order, not a build/probe decision.
- **`relation_statistics_bindings.test`** — description: *"Relation
  statistics remain aligned with output bindings through projections,
  aggregates and CTEs."* Every assertion here is either a correctness check
  on query results after a `MATERIALIZED`/`NOT MATERIALIZED` CTE has been
  inlined or not, or a plan-shape assertion that a three-relation join stays
  a proper join tree rather than degrading to a `CROSS_PRODUCT`
  (`<!REGEX>:.*CROSS_PRODUCT.*`) — about whether DPhyp's relation-statistics
  bookkeeping (Chapter 8 §8.4's "relation sets" bookkeeping) survives binding
  renumbering through aggregates/CTEs/`PIVOT`, not about build/probe
  orientation of the resulting joins.
- **`parquet_minmax_cardinality_estimation.test`** — description: *"Use
  Parquet min/max as a local join-order distinct-count proxy."* The name
  says it outright: this is a distinct-count proxy for **join-order** cost
  estimation, sourced from Parquet row-group statistics rather than
  DuckDB's own `ANALYZE`-collected statistics.

All six of the files above are legitimate, well-constructed cardinality-
estimation tests — and every one of them ultimately feeds the same DPhyp
cost model Chapter 8 covers in depth. None of them assert anything about
which child of a join ends up on the build side versus the probe side; that
distinction is decided ten pipeline positions later, by a different pass,
reading different information (`GetBuildSize`'s row-width arithmetic, not
DPhyp's cardinality-times-selectivity cost function). Presenting them as
build/probe-side evidence would have been the forced framing this book's
sourcing policy asks to avoid; they belong in Chapter 8's evidence base, not
here.

Six more test files in this directory follow the same pattern. None of them
is about build/probe placement:

- `always_true_scan_filter_estimate.test`: a scan predicate that is proven
  always true must not lower the estimated row count.
- `filter_projection_map_reconstruction.test`: the projection map of a
  filter above a reordered join tree must follow the new column order.
- `join_elimination_condition_reference.test`: join elimination must keep a
  join whose output a join condition higher up still reads.
- `not_equal_join_cardinality_estimation.test`: a `<>` or
  `IS DISTINCT FROM` join is estimated as nearly a cross product.
- `zero_column_filter_reconstruction.test`: a filter that reads no column
  must survive when the join order is rebuilt.
- `left_join_reordering/left_join_composite_domain.test`: composite equality
  domains for `LEFT` joins.

Each one checks the join-order side of Chapter 8, not this pass.
