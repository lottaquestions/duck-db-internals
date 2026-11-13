# Chapter 7, Part 5 — CTE Inlining, CTE Filter Pushdown, and Common Subplan Detection

## 7.9 Overview: three passes, one recurring theme

§7.4 of the chapter overview gave each of these passes a one-paragraph
summary and placed them precisely in the ~45-call-site pipeline:
`CTE_INLINING` at #2 and #12, `CTE_FILTER_PUSHER` at #7, and `COMMON_SUBPLAN`
at #28. This part goes back to the real source
(`src/optimizer/cte_inlining.cpp`, `src/optimizer/cte_filter_pusher.cpp`,
`src/optimizer/common_subplan_optimizer.cpp`) and the real test suite for
all three, plus one logical-operator-level test
(`materialized_cte_sibling_column_pruning.test`) that is really about a
different pass but is CTE-specific enough to belong here.

All three passes share one recurring correctness theme, worth stating up
front because it will recur, in a slightly different shape, in every
section below: **a plan rewrite that would be sound if a piece of work
happened exactly once becomes unsound the moment that work is duplicated,
if the work is not actually side-effect-free and repeatable.** `CTE_INLINING`
duplicates a CTE's definition across every reference site when it inlines;
`CTE_FILTER_PUSHER` moves a filter from one reference site into a
definition that other, differently-filtered reference sites also read;
`COMMON_SUBPLAN` does the exact reverse of inlining — it takes two
*already-duplicated* copies of the same computation and collapses them back
into one, on the theory that they were interchangeable all along. Every one
of the three passes has to answer, in its own way, "is this actually safe to
duplicate / relocate / collapse," and the answer routes through the same
handful of concerns each time: is any part of it volatile (`random()`,
`nextval()`, anything `IsVolatile()` reports true for), does every consumer
that needs the work actually get it, and — for filters specifically — could
relocating a conjunct that `CanThrow()` change whether an error surfaces.

## 7.10 `CTE_INLINING` — inline or materialize, decided per CTE

**Purpose, in one sentence:** for every `WITH`-clause common table
expression the binder unconditionally wraps in a `LogicalMaterializedCTE`
node, `CTE_INLINING` decides whether to substitute the CTE's definition
directly at each reference site (as if it had been written as a repeated
subquery) or to leave it materialized — computed once, scanned via
`LogicalCTERef`/`CTE_SCAN` wherever it's used.

### 7.10.1 The binder always builds a materialized CTE; the optimizer decides its fate

It's worth confirming this precisely before looking at the decision logic,
because it settles where authority for the inline/materialize choice
actually lives. `Binder::FinishCTE` (`src/planner/binder/query_node/bind_cte_node.cpp:180-216`)
is the only place a `LogicalMaterializedCTE` gets constructed for an
ordinary `WITH` clause, and it runs unconditionally for every referenced,
non-side-effecting CTE:

```cpp
// src/planner/binder/query_node/bind_cte_node.cpp:211
auto root = make_uniq<LogicalMaterializedCTE>(bound_cte.ctename, bound_cte.setop_index, result.types.size(),
                                              std::move(cte_query), std::move(cte_child), bound_cte.materialized);
```

`LogicalCTE`'s constructor (`src/include/duckdb/planner/operator/logical_cte.hpp:26-32`)
pushes its two arguments as `children[0]` ("top", the CTE's own definition)
and `children[1]` ("bottom", the rest of the plan that consumes it) in that
order, and `LogicalMaterializedCTE::GetColumnBindings()` /
`ResolveTypes()` both delegate to `children[1]` — the node's *externally
visible* schema is the schema of the query that uses the CTE, confirming
`children[1]` really is "everything downstream," not just the immediate
consumers. There is no bind-time inlining shortcut for a "simple" CTE
anywhere in this file; every reference becomes a `LogicalCTERef` under a
shared `LogicalMaterializedCTE`, and the entire inline-vs-materialize
decision is made later, entirely inside `src/optimizer/cte_inlining.cpp`.

DuckDB can export an optimized plan as SQL (`EXPLAIN (SQL)`). A caller of
`LogicalPlanSQLExporter::ExportWithSources` can replace parts of the plan with
its own relations. Each replaced part is written out as a CTE. By default that
CTE is `CTE_MATERIALIZE_DEFAULT`
(`src/include/duckdb/planner/logical_plan_sql_exporter.hpp:52`). So the same
`CTE_INLINING` rules in this section decide whether it is inlined. A caller
that needs the relation to run exactly once must ask for
`CTE_MATERIALIZE_ALWAYS`. A volatile source, such as one that calls
`nextval(...)`, still runs once under the default. If it is referenced once,
inlining just moves it. If it is referenced more than once, point 7 below
(`PreventInlining`) keeps it materialized. The C++ test
`test/sql_export/sql_export_plan_sources.cpp` ("Source replacement
materialization is selected per boundary") checks both settings.

### 7.10.2 The decision, read straight from source

`CTEInlining::TryInlining` (`src/optimizer/cte_inlining.cpp:135-234`)
recurses into a node's children *first* ("traverse children first, so we
can inline the deepest CTEs first" — line 143), then, if the node itself is
a `LOGICAL_MATERIALIZED_CTE`, runs the following checks **in exactly this
order**, each one capable of returning early and short-circuiting
everything below it:

```cpp
// src/optimizer/cte_inlining.cpp:148-234 (condensed to the control flow)
auto &cte = op->Cast<LogicalMaterializedCTE>();
auto ref_count = CountCTEReferences(*op, cte.table_index);

if (ref_count == 0) {
    if (cte.children[0]->HasSideEffects()) return;         // must still execute
    op = std::move(op->children[1]); return;                // drop it entirely
}
if (cte.children[0]->HasSideEffects()) return;              // never inline a side-effecting CTE

if (ContainsDelimGet(*cte.children[0]) &&
    HasCTEReferenceBelowDelimJoin(*op->children[1], cte.table_index)) return;  // DELIM_GET scoping hazard

if (cte.materialize == CTEMaterialize::CTE_MATERIALIZE_ALWAYS) return;         // explicit MATERIALIZED wins

if (ref_count == 1) {
    Inline(op->children[1], *op, /*requires_copy=*/false);   // no copy needed
    return;
}
// ref_count > 1
if (cte.materialize == CTEMaterialize::CTE_MATERIALIZE_NEVER) {
    Inline(op->children[1], *op, /*requires_copy=*/true);    // NOT MATERIALIZED forces it
    return;
}
PreventInlining prevent_inlining;
prevent_inlining.VisitOperator(*op->children[0]);
if (prevent_inlining.prevent_inlining) return;               // any volatile expr anywhere in the definition

if (CTEInlining::EndsInAggregateOrDistinct(*op->children[0])) return;          // mimics binder's default heuristic

bool is_cheap_to_inline = op->children[0]->type == LogicalOperatorType::LOGICAL_EMPTY_RESULT ||
                          op->children[0]->type == LogicalOperatorType::LOGICAL_CTE_REF ||
                          EndsInDummyScan(*op->children[0]);

auto base_table_references = CountBaseTableReferences(*op->children[0]);
if (!is_cheap_to_inline && base_table_references > 2 && base_table_references * ref_count > 10) return;

if (is_cheap_to_inline || ContainsLimit(*op->children[1])) {
    Inline(op->children[1], *op, /*requires_copy=*/true);
    return;
}
// otherwise: fall through, do nothing — stays materialized
```

Every one of these conditions is worth naming precisely, because the
research recon this chapter started from described the decision loosely as
"reference count / hints / volatility / expected size / emptiness," and the
real ordering — which check runs first and which short-circuits the rest —
turns out to matter for several of the test cases below.

1. **Unreferenced, side-effect-free CTE (`ref_count == 0`)**: dropped
   entirely — `op = std::move(op->children[1])` discards the whole
   `LogicalMaterializedCTE` node, definition included. `HasSideEffects()`
   is checked *first*, though, because a data-modifying CTE body (an
   `INSERT`/`UPDATE`/`DELETE`/`MERGE`/`COPY` written inside a `WITH` clause)
   must still run even if nothing reads its result — that's the entire
   point of a side-effecting CTE (`Binder::FinishCTE`, lines 183-193, does
   the matching bind-time check: a side-effecting CTE is force-bound even
   when unreferenced, rather than silently dropped the way a non-effecting
   one is).
2. **Unconditional side-effect check for any `ref_count`**
   (`cte.children[0]->HasSideEffects()` a second time, outside the
   `ref_count == 0` branch): a side-effecting CTE is *never* inlined,
   however many times it's referenced — the comment is explicit about why:
   *"the LOGICAL_MATERIALIZED_CTE guarantees that it executes exactly once
   and before the query side."* Inlining would duplicate the DML.
3. **`DelimGet`-under-nested-`DelimJoin` safety check**: if the CTE's own
   definition already contains a `LOGICAL_DELIM_GET` (this happens when a
   correlated-subquery decorrelation, Chapter 6's `FlattenDependentJoins`,
   left a duplicate-eliminating scan *inside* the CTE body itself — the
   materialized-CTE-based dedup path Chapter 6 §6.9 documents) **and** some
   reference to this same CTE sits *below another* `DELIM_JOIN` elsewhere
   in the tree, inlining is refused. The comment spells out the hazard
   precisely: *"Inlining a CTE that already contains a DELIM_GET stays safe
   while all matching CTE scans remain outside DELIM_JOIN subtrees, but
   once a scan is nested below another DELIM_JOIN the inlined DELIM_GETs
   can attach to the wrong duplicate-elimination source."* Duplicating the
   `DelimGet` by copying the CTE body would create two `DelimGet`s that both
   need to bind to the *same* duplicate-elimination source, and a
   downstream `DelimJoin` nesting could make that binding ambiguous or
   simply wrong.
4. **Explicit `MATERIALIZED` hint (`CTE_MATERIALIZE_ALWAYS`)**: this check
   runs *before* the `ref_count == 1` branch below it — which means an
   explicit `MATERIALIZED` hint overrides even a single-reference CTE that
   would otherwise be trivially, unconditionally inlined. §7.10.3's case 4
   is a real test that depends on exactly this ordering.
5. **`ref_count == 1`**: inlined unconditionally, with `requires_copy =
   false` — there is only one reference, so `Inline()` can move the
   definition directly into place rather than deep-copying it.
6. **`ref_count > 1`, explicit `NOT MATERIALIZED` hint
   (`CTE_MATERIALIZE_NEVER`)**: forced to inline regardless of cost —
   *"this CTE is referenced multiple times, but we are not allowed to
   materialize it, we have to inline it if possible"* (the comment's own
   words). No cost heuristic below this point ever gets consulted; the
   hint is absolute (short of `Inline()` itself failing to deep-copy an
   operator it doesn't know how to copy — see point 9).
7. **Volatility check (`PreventInlining`)**: only reached for
   `ref_count > 1` with the *default* materialization policy. `PreventInlining`
   (`cte_inlining.cpp:268-292`) walks every expression in the CTE's own
   definition and sets `prevent_inlining = true` the moment it finds a
   `BOUND_FUNCTION` expression whose `IsVolatile()` is true — with one
   explicit, named exception: the `error` function is exempt (`if
   (bound_function.Function().GetName() == "error") return;` — calling
   `error(...)` twice, once per inlined copy, is exactly as observable as
   calling it once, since either way the query fails with that message,
   so there's no correctness reason to block inlining over it). Any other
   volatile call — `random()`, `uuid()`, a sequence's `nextval()` — blocks
   inlining outright: duplicating the CTE's body across every reference
   site would mean each reference independently re-evaluates the volatile
   call, silently changing what used to be *one* draw from `random()`
   (shared by every consumer of the materialized result) into *N*
   independent draws.
8. **`EndsInAggregateOrDistinct`**: if the CTE's definition's outermost
   operator chain (following single-child operators down) bottoms out at a
   `LOGICAL_AGGREGATE_AND_GROUP_BY`, `LOGICAL_DISTINCT`, or `LOGICAL_WINDOW`,
   inlining is refused — recomputing an aggregation or a window function
   once per reference site is exactly the kind of duplicated work
   materialization exists to avoid, and the comment says this explicitly
   mimics "the behavior of the CTE materialization in the binder" (i.e., a
   long-standing DuckDB heuristic that an aggregating CTE is a natural
   materialization boundary). This same helper, `CTEInlining::EndsInAggregateOrDistinct`,
   is `public static` specifically so `CommonSubplanOptimizer::ShouldMaterialize`
   (§7.12.2) can reuse it — the "ends in an aggregate ⇒ worth materializing"
   heuristic isn't duplicated logic, it's the *same* function called from
   two different passes.
9. **`is_cheap_to_inline`**: true if the CTE's own definition is *already*
   a `LOGICAL_EMPTY_RESULT`, is itself just a `LOGICAL_CTE_REF` (a CTE whose
   body is nothing but a reference to another CTE), or `EndsInDummyScan`
   (bottoms out at a `LOGICAL_DUMMY_SCAN`, a `LOGICAL_EMPTY_RESULT`, or
   another `LOGICAL_CTE_REF` through a chain of single-child operators).
   These are all shapes cheap enough that duplicating them costs
   essentially nothing at runtime — an empty result stays empty no matter
   how many times it's copied. §7.10.5 walks through exactly how a CTE's
   definition *becomes* `LOGICAL_EMPTY_RESULT` by the time this check runs,
   which turns out to be the whole reason the pass needs to run twice.
10. **The size guard**: `base_table_references = CountBaseTableReferences(*op->children[0])`
    counts `LogicalGet` nodes anywhere in the definition (recursively,
    unconditionally — unlike `ContainsLimit`/`EndsInDummyScan`, this walk
    does *not* stop at multi-child operators, so it correctly counts every
    base table under a join too). If the definition is **not** cheap to
    inline, references **more than two** base tables, **and** the product
    `base_table_references * ref_count` exceeds `10`, inlining is refused
    outright — a moderately large multi-table CTE referenced several times
    would duplicate a correspondingly large amount of scan-and-join work if
    inlined, and past this threshold that cost is judged to outweigh
    whatever benefit inlining brings (mainly, letting later per-reference
    filter pushdown/column pruning specialize each copy to what that
    specific reference actually needs). Note the guard's shape: a
    two-table CTE referenced a hundred times is *never* blocked by this
    check (`base_table_references > 2` is false), no matter how large
    `ref_count` gets — the threshold is about join-fan-out per copy, not
    about total reference count.
11. **The `LIMIT`-driven final decision**: `if (is_cheap_to_inline ||
    ContainsLimit(*op->children[1]))` — `ContainsLimit`
    (`cte_inlining.cpp:55-68`) is a narrow structural check: it returns true
    if the operator itself is a `LOGICAL_LIMIT`/`LOGICAL_TOP_N`, and
    otherwise only recurses when the operator has *exactly one* child,
    which means it silently gives up (`return false`) the moment it hits
    any join, set operation, or other multi-child operator — a `LIMIT`
    *inside* one branch of a join that consumes the CTE is invisible to
    this check. Applied to `op->children[1]` (the whole consuming side of
    the query, not just the immediate reference site), this asks: is there
    a `LIMIT`/`TOP_N` sitting directly above an unbroken single-child chain
    that eventually reaches (or contains) this CTE's use? The reasoning,
    straight from the inline comment: *"CTEs require full materialization
    before the CTE scans begin, LIMIT and TOP_N operators cannot abort the
    materialization, even if only a part of the CTE result is needed."* A
    materialized CTE is an all-or-nothing barrier — even if the consuming
    query only wants the first 10 rows, the CTE's *entire* result still has
    to be computed and buffered before any `CTE_SCAN` can start producing
    rows for the `LIMIT` to trim. Inlining removes that barrier: each
    inlined copy becomes an ordinary part of the plan that the physical
    executor can push a `LIMIT` into and stop early on, exactly like Query
    A's `TOP_N` fusion (§7.6.4/§7.7) can stop scanning early once the
    bounded heap is full. So: **if it's cheap, inline it (materializing an
    empty/dummy/CTE-ref result buys nothing); otherwise, only inline it if
    doing so would let a `LIMIT` short-circuit work that a shared
    materialization would otherwise force to run to completion.** If
    neither holds, the function falls through without touching `op` at
    all — the CTE simply stays materialized. This is worth stating plainly
    because it's easy to miss: **a multi-referenced CTE with no `LIMIT`
    downstream and a body too expensive to call "cheap" is left
    materialized by default**, even if it's small enough to have sailed
    past the base-table-count guard. Materialization, not inlining, is the
    fallback outcome for the "otherwise unremarkable" multi-reference case.

`CTEInlining::Inline` (`cte_inlining.cpp:236-266`) is the mechanical part:
it recurses through `op->children[1]` looking for `LOGICAL_CTE_REF` nodes
matching this CTE's index, and at each one, if `requires_copy` is true, runs
`LogicalOperatorDeepCopy::DeepCopy` on the definition to produce an
independent copy (catching `NotImplementedException` — some operator shapes
genuinely cannot be deep-copied, in which case `Inline` returns `false` and
`TryInlining` leaves the CTE materialized rather than half-inlining it),
then wraps the (copied or moved) definition in a `LogicalProjection` that
re-establishes the `LogicalCTERef`'s original output bindings. (This
wrapper is built by one call, `LogicalProjection::CreateIdentity(cteref.table_index, ...)`.
It is a projection that passes every column through unchanged, under the
`CTE_REF`'s table index.) Every
matching `CTE_REF` gets its own independent copy when `requires_copy` is
true — there is no "keep the original for the last reference, copy for the
rest" special case; an `N`-times-referenced CTE inlined this way produces
`N` independent copies of its definition, not `N - 1` copies plus the
original.

### 7.10.3 Why it runs twice — verified, not guessed

§7.4's pipeline listing offered a plausible-sounding but unverified guess
for why `CTE_INLINING` needs a second run: "the first run happens before
join reordering and grouping-set lowering have had a chance to change what
'references this CTE' even means." Direct source inspection turns up a
sharper, fully verified mechanism for at least one concrete and testable
case, and it's worth walking through end to end because it also explains
exactly what runs *between* the two calls.

`Optimizer::RunBuiltInOptimizers` calls `CTE_INLINING` at
`optimizer.cpp:260` (pass #2) and again at `optimizer.cpp:333` (pass #12).
Between them, in order: `SELF_JOIN_AGGREGATE` (see the overview's §7.4.1), `AGGREGATE_FUNCTION_REWRITER`, `FILTER_PULLUP`,
**`FILTER_PUSHDOWN`**, an unlabeled `FilterStatisticsOptimizer` step (see
§7.11.2 — it's gated and usually a no-op), `CTE_FILTER_PUSHER`,
`REGEX_RANGE`, `IN_CLAUSE`, `DELIMINATOR`, `GROUPING_SETS` plus an
unconditional `MultiStageAggregateRewriter` pass.

The load-bearing one is `FILTER_PUSHDOWN`. `FilterPushdown::PushdownFilter`
(`src/optimizer/pushdown/pushdown_filter.cpp:9-24`) doesn't just relocate
filters — it also has its own lightweight, local unsatisfiability check,
independent of the much larger `STATISTICS_PROPAGATION` pass that runs at
#34:

```cpp
// src/optimizer/pushdown/pushdown_filter.cpp:16-20
for (auto &expression : filter.expressions) {
    if (AddFilter(std::move(expression)) == FilterResult::UNSATISFIABLE) {
        // filter statically evaluates to false, strip tree
        return make_uniq<LogicalEmptyResult>(std::move(op));
    }
}
```

`AddFilter` is backed by `FilterCombiner` (`filter_combiner.cpp`), which can
recognize a filter expression that's a constant `FALSE`, a constant `NULL`,
or a combination its own local reasoning can already prove impossible — and
when it does, `FilterPushdown` replaces the `LogicalFilter` with a
`LogicalEmptyResult` *right there*, at pass #6, long before
`STATISTICS_PROPAGATION` (#34) would otherwise have proven the same thing
from column statistics. `FilterPushdown` recurses into a `LogicalMaterializedCTE`'s
children exactly like any other operator, so this fold applies just as
readily *inside* a CTE's own definition as anywhere else in the tree.

Put those two facts together: a CTE whose body contains a provably
unsatisfiable filter (`WHERE NULL`, `WHERE FALSE`, or an expression pass #1's
`ConjunctionSimplificationRule`/`ConstantOrNullSimplification` has already
reduced to a constant `NULL`/`FALSE` predicate — expression-level folding
doesn't remove the `LogicalFilter` *node*, only simplifies what's inside
it, so the structural collapse to `LogicalEmptyResult` still has to wait for
something that acts on operators, not just expressions) is, at the *first*
`CTE_INLINING` call (#2), still a live `LogicalFilter` — `is_cheap_to_inline`
is false, and if there's no `LIMIT` on the consuming side either, the CTE
stays materialized. Then `FILTER_PUSHDOWN` (#6) runs, reaches that filter,
and — via the mechanism just shown — replaces it with `LogicalEmptyResult`.
By the time the *second* `CTE_INLINING` call (#12) runs, `op->children[0]->type`
is now literally `LOGICAL_EMPTY_RESULT`, `is_cheap_to_inline` flips to true,
and the CTE — previously refused inlining on cost/`LIMIT` grounds — is now
inlined unconditionally, copy included.

This is a precise, source-verified instance of the general principle the
second call exists for: **a pass that runs between the two `CTE_INLINING`
calls can change a CTE's own cost profile (or, more generally, its
reference count, via `DELIMINATOR`'s dependent-join cleanup, or its shape,
via `GROUPING_SETS`' lowering) in a way that only the second call gets a
chance to react to.** §7.10.5 walks through the exact test file this
mechanism was built (and is tested) for.

### 7.10.4 `test/optimizer/cte_inlining.test`, case by case

Every case in this file uses `PRAGMA explain_output = OPTIMIZED_ONLY` and
checks the post-optimization logical plan text for the presence or absence
of the string `CTE` (via `<REGEX>`/`<!REGEX>`), against a table `a(i
integer)` holding one row, `(42)`.

**Case 1 — single reference, default hint:**

```sql
explain with cte1 as (Select i as j from a) select * from cte1;
----
logical_opt	<!REGEX>:.*CTE.*
```

`ref_count == 1`, no explicit hint, so per point 5 above this is inlined
unconditionally — the plan after optimization has no `CTE`/`CTE_SCAN`
operator at all, just `a`'s scan projected directly to `j`.

**Case 2 — single reference, explicit `NOT MATERIALIZED`:**

```sql
explain with cte1 as NOT MATERIALIZED (Select i as j from a) select * from cte1;
----
logical_opt	<!REGEX>:.*CTE.*
```

The test's own comment calls this "multiple references... not inlined by
default, but can be forced," which reads oddly against a query with exactly
one reference — but the point still stands precisely, just via a different
branch: since `ref_count == 1`, this would be inlined *regardless* of the
`NOT MATERIALIZED` hint (point 5 fires before the hint is even consulted
for the `> 1` case). What this case actually pins down is that an explicit
`NOT MATERIALIZED` hint on a single-reference CTE doesn't somehow force
materialization instead — inlining still happens, exactly as if no hint had
been given at all, because `CTE_MATERIALIZE_ALWAYS` is the *only* hint value
checked before the `ref_count == 1` branch (point 4); `CTE_MATERIALIZE_NEVER`
is only consulted inside the `ref_count > 1` branch, and is simply never
reached here.

**Case 3 — chained CTEs, both single-reference:**

```sql
explain
with
  cte1 as (select i as j from a),
  cte2 as (select * from cte1)
select * from cte2;
----
logical_opt	<!REGEX>:.*cte1.*
```

The bound tree is `LogicalMaterializedCTE(cte1, LogicalMaterializedCTE(cte2,
main))`, with `cte2`'s own definition containing a `LogicalCTERef` back to
`cte1`. `TryInlining`'s children-first recursion reaches `cte2`'s node
before `cte1`'s: `cte2` has `ref_count == 1` (the main query references it
once) and is inlined, which splices `cte2`'s definition (`select * from
cte1`, itself still containing the `CTE_REF` to `cte1`) directly into the
main query. Recursion then unwinds to `cte1`'s own `LogicalMaterializedCTE`
node, and *now* — after `cte2` has already been inlined into `cte1`'s
consuming side — `CountCTEReferences(*op, cte1.table_index)` finds exactly
one reference (the one that used to live inside `cte2`'s definition, now
spliced directly under `cte1`). `cte1` is inlined too. Both CTEs vanish;
the deepest-first traversal order is precisely what makes this cascade work
in a single pass, without needing a third `CTE_INLINING` call.

**Case 4 — mixed hints, proving hint order beats reference count:**

```sql
explain
with
  cte1 as MATERIALIZED (select i as j from a),
  cte2 as NOT MATERIALIZED (select * from cte1)
select * from cte2;
----
logical_opt	<REGEX>:.*CTE.*
```

`cte2` (`NOT MATERIALIZED`, referenced once by the main query) is inlined
via `ref_count == 1` exactly as in case 1 — the hint doesn't matter, since
that branch never checks it. This splices `cte2`'s body (`select * from
cte1`, containing a `CTE_REF` to `cte1`) into the main query, so `cte1` now
also has `ref_count == 1`. Naively, that would suggest `cte1` should inline
too — but `cte1` carries an explicit `MATERIALIZED` hint
(`CTE_MATERIALIZE_ALWAYS`), and point 4 in §7.10.2's ordering checks that
hint **before** the `ref_count == 1` branch ever gets a chance to run. So
`cte1` stays materialized regardless of how many references it ends up
with. This is the test that pins down the exact ordering claim from
§7.10.2: an explicit `MATERIALIZED` hint overrides even a trivially
inlinable single-reference CTE.

**Case 5 — unreferenced CTE:**

```sql
explain
with
  cte1 as (select i as j from a)
select * from a;
----
logical_opt	<!REGEX>:.*CTE.*
```

`cte1` is never referenced by the main query (`select * from a`, not `from
cte1`). `ref_count == 0`, the definition has no side effects, so
`TryInlining` drops the whole `LogicalMaterializedCTE` node —
`op = std::move(op->children[1])` — and the CTE's definition (and the scan
of `a` inside it) never executes at all.

### 7.10.5 `test/optimizer/cte_statistics_propagation.test` — one computation, shared statistics

**Mechanism, verified against `src/optimizer/statistics/operator/propagate_cte.cpp`:**
a materialized CTE is evaluated exactly once, so its column and row-count
statistics are computed exactly once too, and shared by every reference
site — rather than each `LogicalCTERef` independently re-deriving statistics
for "whatever the CTE happens to produce."

```cpp
// src/optimizer/statistics/operator/propagate_cte.cpp:7-32
unique_ptr<NodeStatistics> StatisticsPropagator::PropagateStatistics(LogicalMaterializedCTE &op, ...) {
    // the CTE is evaluated once, so the statistics of its definition hold for every reference to it
    auto definition_node_stats = PropagateStatistics(op.children[0]);
    ...
    cte_stats_map[op.table_index] = std::move(cte_stats);
    // the operator emits whatever the query referencing the CTE emits
    return PropagateStatistics(op.children[1]);
}

unique_ptr<NodeStatistics> StatisticsPropagator::PropagateStatistics(LogicalCTERef &op, ...) {
    if (op.is_recurring) {
        // the recurring table is filled during execution - the definition does not describe it
        return nullptr;
    }
    auto entry = cte_stats_map.find(op.cte_index);
    ...
    // a reference scans the whole CTE, so it emits exactly as many rows as the definition
    return make_uniq<NodeStatistics>(*cte_stats.node_stats);
}
```

Every test case in this file is a direct consequence of that
once-computed, shared-statistics design:

**Case 1 — a non-nullable definition makes a downstream `IS NOT NULL` join
condition provably always true, so it's removed:**

```sql
CREATE TABLE t(i INTEGER, j INTEGER);
INSERT INTO t VALUES (1, 10), (2, 20), (3, 30);
EXPLAIN WITH c AS MATERIALIZED (SELECT * FROM t) SELECT * FROM t l JOIN c ON l.i + c.i IS NOT NULL
----
logical_opt	<!REGEX>:.*IS NOT NULL.*
```

`t.i` has no nulls, so `c`'s definition (`SELECT * FROM t`) propagates
`CANNOT_HAVE_NULL_VALUES` for its own `i` column into `cte_stats_map`; the
`CTE_REF` for `c` inherits that stat directly, and the join condition
`l.i + c.i IS NOT NULL` folds away as always-true.

**Case 2 — the same shape, but the definition scans a table that genuinely
can produce nulls, so the condition survives:**

```sql
CREATE TABLE nullable(i INTEGER, j INTEGER);
INSERT INTO nullable VALUES (1, 10), (NULL, 20);
EXPLAIN WITH c AS MATERIALIZED (SELECT * FROM nullable) SELECT * FROM t l JOIN c ON l.i + c.i IS NOT NULL
----
logical_opt	<REGEX>:.*IS NOT NULL.*
```

Same mechanism, opposite conclusion — this pair of cases is the file's own
"but only if it really cannot produce NULL" comment, and demonstrates that
the CTE-statistics shortcut isn't a blanket assumption of non-nullability;
it faithfully reflects whatever the definition's *own* statistics say.

**Case 3 — min/max propagate too, folding an unreachable comparison and
proving the query returns zero rows:**

```sql
EXPLAIN WITH c AS MATERIALIZED (SELECT * FROM t) SELECT * FROM t l JOIN c ON l.i + c.i > 1000
----
logical_opt	<!REGEX>:.*1000.*

WITH c AS MATERIALIZED (SELECT * FROM t) SELECT count(*) FROM t l JOIN c ON l.i + c.i > 1000
----
0
```

`t.i`'s known max is 3, so `l.i + c.i` can never exceed 6; the `> 1000`
comparison is provably always false, and the join collapses (per
`HandleJoinNeverMatches`, §7.10 note in the overview's §7.6.2) to zero rows
— exactly what the plain `count(*)` query confirms.

**Case 4 — the CTE's propagated cardinality feeds a downstream aggregate
overflow proof, not just filter folding:**

```sql
CREATE TABLE small_vals(v BIGINT);
INSERT INTO small_vals SELECT (i * 1000000000)::BIGINT FROM range(1000) s(i);
WITH c AS MATERIALIZED (SELECT v FROM small_vals) SELECT sum(v) FROM c
----
499500000000000

CREATE TABLE big_vals(v BIGINT);
INSERT INTO big_vals SELECT 4000000000000000000::BIGINT FROM range(5) s(i);
WITH c AS MATERIALIZED (SELECT v FROM big_vals) SELECT sum(v) FROM c
----
20000000000000000000
```

The test's own comment is precise about what this checks: the CTE
definition's propagated *cardinality* (row count) reaches the `sum(v)`
reference through exactly the `cte_stats.node_stats` channel shown above,
which is what lets the aggregate-function rewriter (the plan-level
assertions for the actual `sum` → `sum_no_overflow` rewrite live in a
different file, `test/optimizer/test_aggregate_function_rewrite.test`)
prove whether or not the sum can overflow a 64-bit accumulator. This file
only checks that results stay correct on both sides of that overflow
boundary — 1000 rows of `i * 10^9` sums safely inside `int64`, five rows of
`4×10^18` whose sum (`2×10^19`) genuinely does overflow a signed 64-bit
accumulator, both computed correctly regardless of which summation strategy
gets chosen.

**Case 5 — the same statistics-sharing applies to the deduplicated,
decorrelation-generated CTE a `LATERAL` join produces, not just an
explicit, user-written `WITH` clause:**

```sql
EXPLAIN SELECT * FROM t l, LATERAL (SELECT * FROM t r WHERE r.i + l.i IS NOT NULL ORDER BY r.i + l.i LIMIT 2)
----
logical_opt	<!REGEX>:.*IS NOT NULL.*
```

Chapter 6 §6.9 documents the decorrelation path that materializes a
deduplicated domain of correlation keys as a CTE; this test confirms that
CTE gets exactly the same statistics treatment as an ordinary, hand-written
`WITH` clause — `propagate_cte.cpp` doesn't distinguish "user-written" from
"compiler-generated" materialized CTEs, and the results (each customer's
first two orders by `r.i + l.i`, non-null since `t.i` has no nulls) confirm
correctness end to end.

**Case 6 — the recurring table of a `WITH RECURSIVE` CTE explicitly opts
out of this whole mechanism:**

```sql
WITH RECURSIVE r(i) AS (SELECT 1 UNION ALL SELECT i + 1 FROM r WHERE i < 10) SELECT i FROM r WHERE i > 5 ORDER BY i
----
6
7
8
9
10

WITH RECURSIVE r(i) AS (SELECT 1 UNION ALL SELECT i + 1 FROM r WHERE i < 10)
SELECT count(*) FROM r WHERE i IS NOT NULL
----
10
```

The `is_recurring` guard at the very top of `PropagateStatistics(LogicalCTERef&, ...)`
(`return nullptr` immediately, before even consulting `cte_stats_map`) is
the reason this case is in the file at all: the *recurring* reference
inside a recursive CTE's own recursive term reads a table that grows
*during execution*, one working-table iteration at a time — its anchor
term's statistics (from the `SELECT 1` base case) describe none of that
growth, so reusing them would be actively wrong. The correctness check here
is that both queries still return the right answer (6 through 10; count 10)
even though this specific CTE reference gets *no* statistics shortcut at
all and has to be reasoned about the ordinary way.

### 7.10.6 `test/sql/cte/cte_inlining_empty_result.test` — inlining a CTE that turns out to be empty

This file is the concrete regression test for the exact mechanism
§7.10.3 derived from source: a CTE whose body collapses to
`LOGICAL_EMPTY_RESULT` (via `FILTER_PUSHDOWN`'s own unsatisfiability check,
running between the two `CTE_INLINING` calls) becomes "cheap to inline" by
the second call, and gets inlined — copied once per reference — rather than
staying behind a shared materialization barrier.

**Setup:**

```sql
CREATE TABLE typed_scalars(id INT);
CREATE TABLE typed_data_2(flag BOOLEAN);
INSERT INTO typed_scalars VALUES (1);
INSERT INTO typed_data_2 VALUES (TRUE);
```

**Case 1 — a genuinely elaborate always-false/always-null filter, referenced
twice, through a `ROLLUP`:**

```sql
WITH q4 AS (
  SELECT TRY_CAST(DATE '0001-01-01' AS TIMESTAMP) AS c0
  FROM typed_scalars t2 JOIN typed_data_2 t3 ON (t3.flag <= TRUE)
  WHERE (t3.flag AND NULL)
  ORDER BY c0 ASC NULLS FIRST
  LIMIT 61
)
SELECT t5.c0, MAX(2147483647), MAX(COALESCE(CAST('nan' AS DOUBLE), -2147483648))
FROM q4 t5 JOIN q4 t6 ON t5.c0 = t6.c0
WHERE (9223372036854775807 <> -9223372036854775808)
GROUP BY ROLLUP (t5.c0, t5.c0);
----
NULL	NULL	NULL
```

`t3.flag AND NULL` can only ever evaluate to `NULL` or `FALSE` — never
`TRUE` — for any boolean value of `t3.flag`; `q4`'s own filter is
unconditionally unsatisfiable, entirely independent of what data
`typed_data_2` actually holds. `q4` is referenced twice (`t5`, `t6`); at the
first `CTE_INLINING` pass it's still a live `LogicalFilter` and stays
materialized. `FILTER_PUSHDOWN` then reaches that filter, `FilterCombiner`
classifies it `UNSATISFIABLE`, and `q4`'s body collapses to
`LOGICAL_EMPTY_RESULT`. At the second `CTE_INLINING` pass,
`is_cheap_to_inline` is now true, so `q4` is inlined — deep-copied once for
`t5` and once for `t6` — and both copies are independently, trivially
empty. The single output row of `NULL NULL NULL` is exactly what an empty
input grouped by `ROLLUP` produces (a `ROLLUP` over zero input rows still
emits the grand-total row, with every aggregate evaluating over zero rows —
`NULL` for every `MAX`).

**Case 2 — the minimal version of the same shape:**

```sql
WITH q AS (SELECT id AS c0 FROM typed_scalars WHERE NULL LIMIT 61)
SELECT t5.c0 FROM q t5 JOIN q t6 ON t5.c0 = t6.c0;
----
(no rows)
```

Same mechanism with a literal `WHERE NULL`: unsatisfiable by
`FilterCombiner`'s simplest case, collapsed to `LOGICAL_EMPTY_RESULT` by
`FILTER_PUSHDOWN`, then inlined (copied twice) by the second `CTE_INLINING`
call. Both copies of the join input are empty, so the inner join between
them produces zero rows.

**Case 3 — the same shape with a third reference and an `ORDER BY`:**

```sql
WITH q AS (SELECT id AS c0 FROM typed_scalars WHERE NULL ORDER BY 1 LIMIT 61)
SELECT t5.c0 FROM q t5 JOIN q t6 ON t5.c0 = t6.c0 JOIN q t7 ON t5.c0 = t7.c0;
----
(no rows)
```

Confirms the same mechanism scales to `ref_count == 3` — `Inline()` has no
special case for "more than two references," it simply deep-copies the
(empty) definition once per matching `LOGICAL_CTE_REF` it finds while
recursing through the consuming side.

**Case 4 — the crucial contrast case: no unsatisfiable filter, so the CTE
stays materialized, and the join still produces a correct, non-empty
result:**

```sql
WITH q AS (SELECT id AS c0 FROM typed_scalars LIMIT 61)
SELECT t5.c0 FROM q t5 JOIN q t6 ON t5.c0 = t6.c0;
----
1
```

With no `WHERE` clause at all, there is nothing for `FILTER_PUSHDOWN` to
prove unsatisfiable — `q`'s definition never becomes `LOGICAL_EMPTY_RESULT`
at any point in the pipeline, so `is_cheap_to_inline` stays false at both
`CTE_INLINING` calls. There's also no `LIMIT` on the *consuming* side
(`ContainsLimit(*op->children[1])` stops at the two-child join, exactly as
described in §7.10.2 point 11 — the `LIMIT 61` lives *inside* `q`'s own
definition, which this check never looks at). Neither disjunct in the final
`if` holds, so `q` falls through the whole decision tree untouched and
stays **materialized**. The join is computed against one shared,
materialized copy of `q` (one row, `id = 1`), and correctly returns that one
row twice-joined-with-itself, i.e. one row. This is the pairing this
chapter promised in its introduction: cases 1 through 3 show correctness
surviving the *inlined-and-collapsed* path, and case 4 shows correctness
holding just as well on the *materialized* path the decision tree falls
back to whenever emptiness genuinely can't be proven.

**Cases 5 and 6 — the physical-plan confirmation, using DuckDB's own
same-query-twice idiom to check two independent structural properties:**

```sql
SET threads=1;
SET profiling_renderer_settings = MAP {'operator_casing': 'upper'};
PRAGMA explain_output='physical_only';

EXPLAIN WITH q AS (SELECT id AS c0 FROM typed_scalars WHERE NULL LIMIT 61)
SELECT t5.c0 FROM q t5 JOIN q t6 ON t5.c0 = t6.c0;
----
physical_plan	<REGEX>:.*EMPTY_RESULT.*EMPTY_RESULT.*

EXPLAIN WITH q AS (SELECT id AS c0 FROM typed_scalars WHERE NULL LIMIT 61)
SELECT t5.c0 FROM q t5 JOIN q t6 ON t5.c0 = t6.c0;
----
physical_plan	<!REGEX>:.*CTE.*
```

Note this is not the "same-label" idiom mentioned elsewhere in this
chapter's sibling files (two `query I nosort <label>` blocks whose
*outputs* must come out byte-identical) — here the *query* is repeated
identically, but the two `EXPLAIN` blocks assert two different, independent
regex properties against that one plan: the physical plan must contain the
string `EMPTY_RESULT` **twice** (one per independently-inlined, independently
collapsed copy of `q` — confirming the two-copy duplication this section
derived, not a single shared empty result), and it must contain the string
`CTE` **zero** times (confirming the CTE really was fully inlined away —
recall from §7.10 that `LogicalCTERef`'s physical form is always a
`PhysicalColumnDataScan` tagged `PhysicalOperatorType::CTE_SCAN`,
`src/execution/physical_plan/plan_recursive_cte.cpp:195-229` — so a
materialized survivor would be structurally impossible to hide from this
check).

## 7.11 `CTE_FILTER_PUSHER` — pushing a filter back into a materialized CTE's own definition

**Purpose, in one sentence:** for a CTE that `CTE_INLINING` left
materialized, `CTEFilterPusher` looks for a filter applied to a
*reference* of that CTE and, when it can prove doing so can't silently drop
rows some other reference needs, moves that filter into the CTE's own
*definition* instead — so the filtering happens once, during
materialization, rather than once per reference after the fact.

### 7.11.1 The safety conditions, read straight from source

`CTEFilterPusher::Optimize` (`src/optimizer/cte_filter_pusher.cpp:23-54`)
first calls `FindCandidates` to build, for every materialized CTE in the
plan, a `MaterializedCTEInfo` recording every reference to it:

```cpp
// src/optimizer/cte_filter_pusher.cpp:83-113 (structure only)
if (op.type == LOGICAL_MATERIALIZED_CTE) {
    cte_info_map.insert(key, make_uniq<MaterializedCTEInfo>(op));   // all_cte_refs_are_filtered starts true
} else if (op.type == LOGICAL_FILTER && op.children[0]->type == LOGICAL_CTE_REF) {
    info->filters.push_back(op);
    info->references.push_back(cte_ref);
} else if (op.type == LOGICAL_CTE_REF) {
    info->all_cte_refs_are_filtered = false;    // an unfiltered reference exists
    info->references.push_back(cte_ref);
}
```

`all_cte_refs_are_filtered` starts `true` for a fresh candidate and is
flipped to `false` the moment **any** reference is found without a
`LogicalFilter` sitting directly on top of it. This is the load-bearing
safety flag, checked twice: once as a cheap pre-filter in the main loop
(skip a candidate outright if `!all_cte_refs_are_filtered &&
!has_filter_dependency`), and again, more carefully, inside
`CanPushFilter`:

```cpp
// src/optimizer/cte_filter_pusher.cpp:65-81
bool CTEFilterPusher::CanPushFilter(const MaterializedCTEInfo &info) {
	if (info.all_cte_refs_are_filtered) {
		return true;
	}
	if (info.filters.size() != 1 || !HasValidDependency(info)) {
		return false;
	}
	auto &cte = info.materialized_cte.Cast<LogicalMaterializedCTE>();
	D_ASSERT(info.filters[0].get().children[0]->type == LogicalOperatorType::LOGICAL_CTE_REF);
	auto &filtered_ref = info.filters[0].get().children[0]->Cast<LogicalCTERef>();
	return filtered_ref.table_index == cte.filter_dependency->row_scan;
}

bool CTEFilterPusher::HasValidDependency(const MaterializedCTEInfo &info) {
	auto &cte = info.materialized_cte.Cast<LogicalMaterializedCTE>();
	return cte.filter_dependency && cte.filter_dependency->MatchesConsumers(info.references);
}
```

The "exactly two references, one row scan and one domain scan" count lives
in its own method, so the CTE join-filter step (§7.11.4) can reuse it:

```cpp
// src/planner/operator/logical_materialized_cte.cpp:6
bool CTEFilterDependency::MatchesConsumers(const vector<reference<LogicalCTERef>> &references) const {
	if (references.size() != 2) {
		return false;
	}
	idx_t row_scans = 0;
	idx_t domain_scans = 0;
	for (auto &ref : references) {
		row_scans += ref.get().table_index == row_scan;
		domain_scans += ref.get().table_index == domain_scan;
	}
	return row_scans == 1 && domain_scans == 1;
}
```

This is the exact, narrow answer to the "referenced multiple times with
different filters" correctness question this chapter opened with:

- **If every reference to the CTE carries some filter** (however different
  those filters are from each other), pushing is safe — `CanPushFilter`
  returns `true` immediately. This is *not* about the filters being
  identical; it's about no reference being *unfiltered*. §7.11.3's second
  test case pushes two genuinely different filters (`digest < '4'` and
  `digest > 'c'`) from two separate references into the same shared
  definition, safely, because the pass combines them with `OR` rather than
  picking one arbitrarily — a row survives the combined predicate if *any*
  reference would have wanted it, which is exactly the union every
  individual reference is entitled to.
- **If at least one reference is unfiltered**, pushing *anything* would be
  unsound in general: an unfiltered reference needs every row, and any
  filter pushed into the shared definition — even one derived by `OR`-ing
  every filtered reference together — could still exclude rows the
  unfiltered reference actually needs (since "unfiltered" is a strictly
  weaker constraint than any nonempty disjunction). The pass refuses,
  *unless* a very specific, narrower structural guarantee holds:
  `cte.filter_dependency` is set, there are **exactly two** references,
  **exactly one** filtered reference, and the two references map
  one-to-one onto a `row_scan`/`domain_scan` pair recorded in
  `CTEFilterDependency`, with the *filtered* reference specifically being
  the `row_scan` side (not the `domain_scan` side — the check would refuse
  the reverse assignment). This `filter_dependency` is a planner-established
  guarantee from decorrelation (Chapter 6): the comment on
  `CTEFilterDependency` itself (`src/include/duckdb/planner/operator/logical_materialized_cte.hpp:18-21`)
  states it precisely — *"Retaining outer rows preserves every domain
  partition needed for their results. The domain scan has no consumers
  independent of the outer-row scan."* In other words: the domain scan's
  entire job is to enumerate which distinct correlation keys exist, and
  every one of those keys is guaranteed, by construction, to also be needed
  by the row scan — so filtering the row scan can never orphan a domain-scan
  key that isn't *also* going to be discarded by the row scan itself. This
  is a compiler-internal invariant, not something a hand-written query can
  set up by accident; `filter_dependency` only ever gets populated by the
  decorrelation machinery that produces this specific row-scan/domain-scan
  CTE shape in the first place.

`PushFilterIntoCTE` (`cte_filter_pusher.cpp:115-185`) then builds the
combined predicate and applies two more, narrower safety checks per
individual conjunct — the same volatility/throwing concerns that recur
throughout this file:

```cpp
// src/optimizer/cte_filter_pusher.cpp:136-162 (condensed)
bool all_conjuncts_repeatable = true;
for (auto &expr : filter.get().expressions)
    all_conjuncts_repeatable &= !expr->IsVolatile() && !ExpressionBarrier::Contains(*expr);

unique_ptr<Expression> inner_expr;
for (auto &filter_expr : filter.get().expressions) {
    if (filter_expr->IsVolatile() || ExpressionBarrier::Contains(*filter_expr) ||
        (!all_conjuncts_repeatable && filter_expr->CanThrow())) {
        continue;    // drop this conjunct from what gets pushed
    }
    ... // AND this conjunct into inner_expr
}
if (!inner_expr) return;   // an unrestricted consumer makes the OR-disjunction true; abort the whole push
```

A **volatile** conjunct (`random() > 0.5`) is never copied into the pushed
predicate — evaluating it once, at materialization time, instead of once
per reference (or worse, moving its evaluation site entirely) would change
how many times, and on what values, it runs. An **`ExpressionBarrier`**-wrapped
conjunct is skipped for the same class of reason — that marker exists
specifically to pin an expression to its original location against exactly
this kind of relocation. And if any conjunct in the *same* filter is
volatile or barriered, then even a conjunct that merely `CanThrow()` (a
cast or division that might error) is *also* excluded — mirroring
`REORDER_FILTER`'s guard from §7.6.1: relocating a throwing conjunct away
from a volatile sibling that used to run alongside it could change whether,
or on what data, the error surfaces. Finally, if dropping unsafe conjuncts
leaves *one* of the filtered references with no safe conjunct left at all,
the whole push is aborted — the comment says it plainly: *"An unrestricted
consumer makes the disjunction true"* — a reference that, after safety
filtering, contributes nothing to the `OR` makes the combined predicate
trivially true for every row, which is the same as not filtering at all,
so there's no point (and no safety issue) in going further, but there's
also no benefit, so the function just returns.

The resulting predicate is wrapped in a fresh `LogicalFilter` directly
above the CTE's own definition, re-run through the 37 expression rewrite
rules (`optimizer.rewriter.VisitOperator`), and then handed to a *full*
`FilterPushdown::Rewrite` call — with
`ProjectionMode::PRESERVE_COMPUTED_EXPRESSIONS`, which is what keeps an
already-materialized expensive computation (§7.11.3's `md5(...)` case) from
being duplicated as a second, independently-recomputed copy just because a
predicate now references its output.

### 7.11.2 The missing piece: `FilterStatisticsOptimizer`

One of this file's test cases (§7.11.3, case 7 below) can only be explained
by a pass that has no line of its own in §7.4's numbered list at all:
`FilterStatisticsOptimizer` (`src/optimizer/filter_statistics.cpp`), invoked
directly inside `Optimizer::RunBuiltInOptimizers` right after
`FILTER_PUSHDOWN` (#6) and right before `CTE_FILTER_PUSHER` (#7):

```cpp
// src/optimizer/optimizer.cpp:293-296
if (!CTEContainsDML(*plan)) {
    FilterStatisticsOptimizer filter_statistics(*this);
    filter_statistics.Optimize(plan);
}
```

Unlike every other step this chapter has covered, this call is not wrapped
in its own `RunOptimizer(OptimizerType::..., ...)` — it has no dedicated
name, no individual profiler timer, and doesn't appear in
`duckdb_optimizers()`. It's gated, though, by a real structural condition:

```cpp
// src/optimizer/filter_statistics.cpp:53-56
void FilterStatisticsOptimizer::Optimize(unique_ptr<LogicalOperator> &plan) {
    if (ContainsDelimJoin(*plan) || !HasMultiBindingFilter(*plan)) {
        return;
    }
```

It only does anything at all if the plan contains a `LogicalFilter` whose
expression references columns from **more than one table binding** (and
contains no `DelimJoin` — decorrelation's own machinery is left alone
here). When that gate passes, it runs a *real* `StatisticsPropagator` —
the same class `STATISTICS_PROPAGATION` (#34) uses, just invoked here in a
narrower `FILTER_SIMPLIFICATION` mode, much earlier in the pipeline, and
labeled (for profiling purposes) as if it *were* `STATISTICS_PROPAGATION`:

```cpp
// src/optimizer/filter_statistics.cpp:58-73
optimizer.RunOptimizer(OptimizerType::STATISTICS_PROPAGATION, [&]() {
    StatisticsPropagator propagator(optimizer, plan, StatisticsPropagationMode::FILTER_SIMPLIFICATION);
    propagator.PropagateStatistics(plan);
    filter_bindings_changed = propagator.FilterBindingsChanged();
});
if (!filter_bindings_changed) return;
optimizer.RunOptimizer(OptimizerType::FILTER_PUSHDOWN, [&]() {
    FilterPushdown filter_pushdown(optimizer);
    ...
    plan = filter_pushdown.Rewrite(std::move(plan));
});
```

If, and only if, propagating statistics in this narrow mode actually
*simplifies* the filter enough to change which table bindings it
references (`filter_bindings_changed`), a *second* `FilterPushdown` pass
runs immediately, to relocate the now-simpler filter to wherever it
belongs given its new, smaller set of bindings. This exists precisely for
the case where a filter is written as a cross-relation disjunction, but
real column statistics can prove one disjunct is unconditionally false —
collapsing the `OR` down to something that references only *one* relation,
and is therefore eligible to be pushed (by the immediately following
`CTE_FILTER_PUSHER`, §7.11.1) somewhere a purely syntactic pushdown could
never have placed it.

### 7.11.3 `test/sql/cte/materialized/cte_filter_pusher.test`, case by case

**Case 1 — nested materialized CTEs, both single-reference, filters
cascade inward in one pass:**

```sql
WITH
  a(x) AS MATERIALIZED (SELECT * FROM generate_series(1, 10)),
  b(x) AS MATERIALIZED (SELECT * FROM a WHERE x < 8)
SELECT * FROM b WHERE x % 3 = 1 ORDER BY x;
----
1
4
7
```

`FindCandidates`'s deepest-first ordering (candidates are processed via
`ctes.rbegin()`/`rend()` over an insertion-ordered map — `a` is bound
outermost and inserted first, so reverse iteration reaches `b` first) means
`b`'s filter `x % 3 = 1` is pushed into `b`'s own definition first.
`PushFilterIntoCTE`'s own `FilterPushdown::Rewrite` call, run immediately as
part of that same push, pushes the newly-added `x % 3 = 1` straight through
`b`'s definition and merges it with the pre-existing `x < 8` filter already
sitting on `a`'s `CTE_REF` inside `b`'s body — producing one combined
`x < 8 AND x % 3 = 1` filter directly above `a`'s reference, in the same
call. `cte_info_map` is then rebuilt (the comment explains why: "Changes to
the plan otherwise break the non-`unique_ptr` references") and the *next*
loop iteration processes `a`, now seeing that combined filter on its own
(now single) reference, and pushes it into `generate_series(1, 10)`'s own
definition. End state: `a`'s materialized result already only contains
`{1, 4, 7}` (values under 8, congruent to 1 mod 3).

**Case 2 — "computed materialized outputs must not be copied into the
combined consumer filter," two references, two different filters, both
filtered:**

```sql
EXPLAIN WITH cte AS MATERIALIZED (
	SELECT i, md5(i::VARCHAR) AS digest FROM range(100) t(i)
)
SELECT count(*) FROM cte WHERE digest < '4'
UNION ALL
SELECT count(*) FROM cte WHERE digest > 'c';
----
logical_opt	<!REGEX>:.*md5.*md5.*

WITH cte AS MATERIALIZED (...) SELECT count(*) FROM cte WHERE digest < '4' UNION ALL SELECT count(*) FROM cte WHERE digest > 'c'
----
25
32
```

Both references *are* filtered (just with different predicates), so
`all_cte_refs_are_filtered` is true and pushing proceeds unconditionally,
combining `digest < '4'` and `digest > 'c'` with `OR`. `digest` is a
*computed* column (`md5(i::VARCHAR)`), not a plain column reference, so
`FilterPushdown`'s `PRESERVE_COMPUTED_EXPRESSIONS` mode keeps the pushed
comparison referencing the already-materialized `digest` output rather than
re-deriving it as a fresh, independent `md5(i::VARCHAR)` expression
computed a second time inline inside the filter — hence the assertion that
`md5` appears in the optimized plan text only once, not twice. Counts (25,
32) confirm the combined filter is applied correctly against real data.

**Case 3 — a plain column rename still pushes all the way to the base
scan:**

```sql
CREATE TABLE cte_filter_pushdown_source AS SELECT i FROM range(100) t(i);
EXPLAIN WITH cte AS MATERIALIZED (SELECT i AS alias FROM cte_filter_pushdown_source)
SELECT * FROM cte WHERE alias = 42;
----
logical_opt	<REGEX>:.*Filters: i = 42.*
```

Unlike case 2's `md5(...)`, `alias` here is a pure rename of `i` — nothing
computed, nothing that needs preserving above a projection — so the pushed
`alias = 42` predicate travels all the way down to become a real table
filter on the base `LogicalGet`, phrased in terms of the underlying column
name `i`.

**Case 4 — projection splitting: cheap filter input and expensive unrelated
output get separated:**

```sql
EXPLAIN WITH cte AS MATERIALIZED (
	SELECT i + 1 AS x, md5(i::VARCHAR) AS payload FROM range(100) t(i)
)
SELECT sum(length(payload)) FROM cte WHERE x < 10;
----
logical_opt	<REGEX>:(?s).*Expressions: x,.*payload.*Expressions: x < 10.*Expressions: x, i.*
```

`FilterPushdown`'s general projection-splitting logic — inherited "for
free" here, since `PushFilterIntoCTE`'s last step is a plain
`FilterPushdown::Rewrite` call, the same class the plan-level
`FILTER_PUSHDOWN` pass uses — splits the original single projection into an
inner one computing just `x` (plus `i`, carried along for later), a filter
`x < 10` directly above that cheap inner projection, and an outer
projection that only computes the expensive `md5(i::VARCHAR)` payload for
rows that already survived the cheap filter. Result: 288 (nine rows with
`i + 1 < 10`, each contributing a 32-character md5 hex digest:
`9 × 32 = 288`).

**Case 5 — the ordinary, non-CTE control case, to show what "full" pushdown
looks like without the CTE-specific `PRESERVE_COMPUTED_EXPRESSIONS`
constraint:**

```sql
EXPLAIN SELECT sum(length(payload))
FROM (SELECT i + 1 AS x, md5(i::VARCHAR) AS payload FROM range(100) t(i)) sq
WHERE x < 10;
----
logical_opt	<REGEX>:(?s).*Expressions: payload.*Expressions: \(i \+ 1\) < 10.*Range.*
```

An ordinary derived subquery has no other consumer that might also need
the intermediate `x` column computed exactly once, so plain `FILTER_PUSHDOWN`
is free to fully re-derive the predicate in terms of the scan's own input
(`(i + 1) < 10`) and push it right down to the `Range` table function —
more aggressive than case 4's CTE version, precisely because there's no
multi-consumer constraint to respect.

**Case 6 — NULLs and duplicates survive the projection split intact:**

```sql
CREATE TABLE cte_projection_split_source(i INTEGER, payload VARCHAR);
INSERT INTO cte_projection_split_source VALUES (1, 'a'), (1, 'b'), (NULL, 'c'), (2, NULL);
WITH cte AS MATERIALIZED (SELECT i + 1 AS x, upper(payload) AS payload FROM cte_projection_split_source)
SELECT coalesce(x, -1), coalesce(payload, 'NULL') FROM cte WHERE x = 2 OR x IS NULL;
----
-1	C
2	A
2	B
```

A regression-style correctness check specifically for the projection-split
rewrite: the `i = NULL` row correctly matches `x IS NULL` (not silently
dropped), and the two `i = 1` rows (duplicate `i`, different `payload`)
both survive as two separate output rows, confirming the split doesn't
accidentally deduplicate.

**Case 7 — statistics can turn a cross-relation filter into one local to a
single CTE reference:**

```sql
CREATE TABLE cte_filter_context(x INTEGER);
INSERT INTO cte_filter_context VALUES (10), (20), (NULL);
CHECKPOINT;
EXPLAIN (FORMAT JSON) WITH cte AS MATERIALIZED (SELECT i AS k, i % 10 AS v FROM range(1000000000) t(i))
SELECT count(*) FROM cte c, cte_filter_context s
WHERE c.v IN (1, 2, 3, 4, 5, 6) OR s.x > 100;
----
logical_opt	<REGEX>:(?s).*"name": "CTE".*"name": "COMPARISON_JOIN".*"name": "RANGE".*"Join Type": "MARK".*"Expressions": "IN \(\.\.\.\)".*"name": "CTE_SCAN".*

WITH cte AS MATERIALIZED (SELECT i AS k, i % 10 AS v FROM range(100) t(i))
SELECT count(*) FROM cte c, cte_filter_context s WHERE c.v IN (1, 2, 3, 4, 5, 6) OR s.x > 100
----
180
```

As written, `c.v IN (1..6) OR s.x > 100` spans two relations (`c` and `s`)
— exactly the `HasMultiBindingFilter` trigger §7.11.2 described. Since
`cte_filter_context` was `CHECKPOINT`ed with values `{10, 20, NULL}`, its
persisted statistics show `max(x) = 20 < 100`, so `FilterStatisticsOptimizer`'s
early `StatisticsPropagator` run proves `s.x > 100` always false, and
`SimplifyFilter` collapses the whole disjunction down to just
`c.v IN (1..6)` — now a filter local to `c` alone. `filter_bindings_changed`
is true, so a fresh `FilterPushdown` relocates that now-single-relation
filter directly onto `c`'s `CTE_REF`, and *then* `CTE_FILTER_PUSHER` (#7)
finds exactly the structural pattern it needs (`LOGICAL_FILTER` directly
over `LOGICAL_CTE_REF`) and pushes `c.v IN (1..6)` into `cte`'s own
definition. The plan-level `IN_CLAUSE` pass (#9), which runs *after*
`CTE_FILTER_PUSHER` (#7) in the fixed pipeline, then finds that predicate
wherever it now lives — inside the CTE's own definition — and rewrites it
into the `MARK`-join-against-a-constant-chunk shape §7.6.6 describes,
which is exactly what the JSON regex's `"Join Type": "MARK"` /
`"Expressions": "IN (...)"` fragment, nested *inside* the `"name": "CTE"`
subtree, confirms. This is a clean illustration of how passes running later
in the fixed, one-shot pipeline (§7.2) can still act on structural changes
an earlier pass made — not because either pass re-runs, but because
`IN_CLAUSE` was always going to walk the whole tree looking for its own
pattern, wherever that pattern happens to live by the time #9 gets its
turn.

**Case 8 — the direct negative control: add one unfiltered reference to the
exact same CTE, and the identical push is refused:**

```sql
EXPLAIN (FORMAT JSON) WITH cte AS MATERIALIZED (SELECT i AS k, i % 10 AS v FROM range(1000000000) t(i))
SELECT count(*) FROM cte c, cte_filter_context s WHERE c.v IN (1, 2, 3, 4, 5, 6) OR s.x > 100
UNION ALL
SELECT count(*) FROM cte;
----
logical_opt	<!REGEX>:(?s).*"name": "CTE".*"name": "COMPARISON_JOIN".*"name": "RANGE".*"Join Type": "MARK".*"Expressions": "IN \(\.\.\.\)".*"name": "CTE_SCAN".*
```

Identical query, plus one extra, entirely unfiltered `SELECT count(*) FROM
cte`. The same statistics-driven simplification still collapses the first
reference's filter down to `c.v IN (1..6)` — but now `all_cte_refs_are_filtered`
is false (the new reference has no filter above its `CTE_REF` at all), and
`cte.filter_dependency` is null (this is an ordinary user-written `WITH`
clause, not a decorrelation-generated row/domain-scan pair), so
`CanPushFilter` refuses. The exact same regex that matched in case 7 is
now asserted absent — a direct, source-grounded confirmation of the "an
unfiltered reference blocks the push" rule from §7.11.1, using the same
query as its own contrast case.

A final `PRAGMA disable_optimizer` / `PRAGMA enable_optimizer` bracket
re-runs case 7's smaller-scale query with every optimization off and checks
the same golden value (180) — the test suite's other standard correctness
idiom (distinct from the "same label, byte-identical output" idiom used
elsewhere in this book's material): run once through the fully optimized
path, once with the optimizer disabled entirely, and confirm the answer
doesn't move.

### 7.11.4 Pushing join keys into a correlation CTE

`CTE_FILTER_PUSHER` has a second step. After it pushes ordinary filters, it
runs a separate class, `CTEJoinFilterPusher`
(`src/optimizer/cte_join_filter_pusher.cpp`). DuckDB PR #26158 ("Push
join-derived key restrictions into correlation CTEs") describes the idea. The
step is deliberately narrow, as the conditions below show.

**The problem.** Decorrelating a `LATERAL` join (Chapter 6) can build a
materialized CTE with two readers. One reader, the *row scan*, reads the outer
rows. The other, the *domain scan*, reads the distinct correlation keys. Each key
starts one run of the inner query (for example, one recursive CTE per key).
§7.11.1 showed that a *filter* on the row scan can be pushed into this CTE. But
sometimes the restriction is not a filter. It is a *join*: the row scan is
joined with a small, already-filtered input. Without this step, the inner
query still runs for every key, even for keys the join will throw away.

**What the step does.** The step only looks at CTEs with a valid
`CTEFilterDependency` (the same `MatchesConsumers` check as above). Then:

1. `CTEJoinFilterCollector::Collect`
   (`src/optimizer/cte_join_filter_collector.cpp:208`, `VisitOperator`) walks the
   plan and reads it only. At each **inner** comparison join it takes the `=`
   and `IS NOT DISTINCT FROM` conditions. One side of the join is the *source*
   of keys. The source is either a reference to another materialized CTE, or a
   simple child (a scan, maybe under a filter or projection) that can be
   copied. The other side must lead down to the row scan of a target CTE. The
   result is a list of `CTEJoinFilter` candidates.
2. `CTEJoinFilterPusher::Optimize` (`cte_join_filter_pusher.cpp:67`) chooses at
   most one candidate per target CTE, using estimates:

   ```cpp
   // src/optimizer/cte_join_filter_pusher.cpp:68
   	// Bound the hash build and require a substantial reduction in retained keys.
   	static constexpr double MAX_SOURCE_RATIO = 0.25;
   	static constexpr double MAX_KEY_RATIO = 0.5;
   ```

   The source may have at most 25% as many rows as the CTE input. The source's
   distinct keys must be at most half of the target's distinct keys. A source
   CTE must be an *ancestor* of the target, so it is already materialized when
   the target runs. A copied source is accepted only if the *unfiltered* scan
   estimate is not bigger than the target (`CanAffordCTEJoinSource`,
   `cte_join_filter_pusher.cpp:18`). This last check exists because a filtered
   estimate could let a huge table be scanned twice.
3. It then puts a `SEMI` join on top of the CTE's definition:

   ```cpp
   // src/optimizer/cte_join_filter_pusher.cpp:202
   		auto join = make_uniq<LogicalComparisonJoin>(JoinType::SEMI);
   		for (idx_t i = 0; i < best->comparisons.size(); i++) {
   			auto source_col = best->source_keys[i].column.GetIndex();
   			auto target_col = best->target_keys[i].column.GetIndex();
   			join->conditions.emplace_back(
   			    best->target_keys[i].BindExpression(cte.children[0]->types[target_col], target_bindings[target_col]),
   			    best->source_keys[i].BindExpression(source_types[source_col], source_bindings[source_col]),
   			    best->comparisons[i]);
   		}
   ```

   The original join stays in place. The semi join only removes rows whose key
   can never match, so row counts and `NULL` behavior do not change.

A sketch of the result, for a target CTE `c` and a small ancestor CTE `o`:

```
 before                                after
 MATERIALIZED_CTE c                    MATERIALIZED_CTE c
   definition: SCAN person               definition: SEMI JOIN (key = o.key)
   ...                                      ├─ SCAN person
   INNER JOIN (o.id = c.id)                 └─ CTE_SCAN o      (added reader of o)
     ├─ CTE_SCAN o                        ...
     └─ CTE_SCAN c (row scan)             INNER JOIN (o.id = c.id)   (unchanged)
```

The collector stops at operators that could change which rows survive. A key
may be a deterministic expression over one column (`CTEJoinFilterKey`), such as
the `STRUCT` field `i.node.k` that the same test file checks for issue #27072.
A `SEMI` restriction is also refused when the target has other
readers: `MatchesConsumers` still requires exactly the row scan and the domain
scan.

**The test.** `test/optimizer/pushdown/cte_domain_join_filter.test` builds
`person(id, selected)` with 1,000 rows, three of them `selected`. A view
`nested_walk` runs a `WITH RECURSIVE ... USING KEY` walk inside a `LATERAL`, per
person. The test reads the profiler's JSON and adds up the rows the
`RECURSIVE_CTE` operators produce (`recursive_rows()`), and counts `SEMI` joins
(`semi_joins()`):

```sql
-- test/optimizer/pushdown/cte_domain_join_filter.test
SELECT max(x) FROM nested_walk WHERE id=3;               -- 5
SELECT recursive_rows();                                 -- 3      (one key walked)

SELECT count(*), sum(x), max(x) FROM nested_walk WHERE selected;  -- 6  74  53
SELECT recursive_rows();                                 -- 9      (three keys walked)
SELECT semi_joins();                                     -- 1      (the added semi join)

SELECT count(*),sum(x) FROM nested_walk;                 -- 2000  500500
SELECT recursive_rows();                                 -- 3000   (all 1,000 keys)
SELECT semi_joins();                                     -- 0      (no reduction, no semi join)
```

With `WHERE selected`, only the three selected keys are walked (9 rows) instead
of all 1,000 (3,000 rows). Without a filter, nothing would be gained, so no semi
join is added. Other cases in the file check the refusals: a materialized CTE
that is also read on its own (`(SELECT sum(x) FROM q)`) is not restricted, and a
`DISTINCT ON` above the walk blocks the semi join, because `DISTINCT ON` must pick
its row before any key is removed.

Two more small details of `CTEFilterPusher::Optimize`
(`cte_filter_pusher.cpp:23-54`): it returns at once when the plan has no
materialized CTE, and it calls `FilterPushdown::CheckMarkToSemi` before
pushing. The source comment says why: *"Scalar producer pushdown must preserve markers
referenced by CTE consumers."* That is, a `MARK` join whose marker column is read
by a CTE reader must not be turned into a `SEMI` join (see the part on filter
pushdown, §2.10.1 of `07-filter-pullup-and-pushdown.md`).

## 7.12 `COMMON_SUBPLAN` — detecting duplicate work anywhere in the plan

**Purpose, in one sentence:** independent of any `WITH` clause the user
wrote, `CommonSubplanOptimizer` finds whole plan subtrees — not just single
expressions — that are syntactically identical after a canonicalization
step, and, when doing so is judged worthwhile, materializes one occurrence
as a synthetic `LogicalMaterializedCTE` and replaces every occurrence
(including the original) with a `CTE_SCAN` onto it.

### 7.12.1 What "syntactically identical" actually means

The honest answer, verified directly against
`src/optimizer/common_subplan_optimizer.cpp`, is: **not literally identical
expression trees** — there's a real normalization step first, and the
actual equality test is byte-equality of a *canonicalized, serialized*
representation, not a structural tree comparison.

Two independent renumbering passes happen before anything is compared.
First, table indices: `PlanSignatureTableIndexMap::Initialize` assigns each
distinct table index appearing anywhere under an operator a fresh,
canonical index starting at a fixed offset (`CANONICAL_TABLE_INDEX_OFFSET =
10000000000000`, `common_subplan_optimizer.cpp:516`), so the arbitrary,
globally-unique indices the binder happened to assign don't distinguish two
subtrees that are otherwise identical in shape. Second, expression-level
cleanup: while converting an operator's expressions to canonical form,
every expression has its **alias cleared** and its **query location
cleared** (`expr.ClearAlias(); expr.SetQueryLocation(optional_idx());`,
`common_subplan_optimizer.cpp:500-501`) before serialization — meaning
`SELECT a + 1` and `SELECT a + 1 AS foo` serialize identically, and two
copies of the same expression written at different source-text offsets
still match. Only *after* this canonicalization does the actual comparison
happen: `PlanSignature::Create` serializes the canonicalized operator with
`BinarySerializer` into a shared `MemoryStream`
(`common_subplan_optimizer.cpp:573-606`), and the resulting byte range *is*
the signature — `GetSignature()` just returns those bytes as a string
(`common_subplan_optimizer.cpp:636-638`). `PlanSignature::Equals`
(`common_subplan_optimizer.cpp:620-633`) compares two subtrees' signature
bytes directly, then recurses into their respective children's signatures
too — so "identical" really does mean identical all the way down, just
identical *after* the alias/table-index/column-index normalization, not
identical in the original, binder-assigned numbering.

A `PlanSignatureHash` (`common_subplan_optimizer.cpp:710-714`) combines each
node's own hash with its children's hashes recursively, giving the
`unordered_map<..., PlanSignatureHash, PlanSignatureEquality>` bucketing
(`subplan_map_t`, `common_subplan_optimizer.cpp:739`) a cheap pre-filter
before falling back to the byte-exact `Equals` check for real.

**Not every operator type is even eligible.** `PlanSignature::OperatorIsSupported`
(`common_subplan_optimizer.cpp:640-697`) is an explicit allow-list — plain
projections, filters, aggregates, windows, unnest, limit, order, top-N,
distinct, pivot, expression-get, dummy-scan, and every plain join/set-operation
type are eligible; the following are deliberately excluded (per the
source's own inline comment listing them): `COPY_TO_FILE`, `SAMPLE`,
`COPY_DATABASE`, `DELIM_GET`, `EMPTY_RESULT`, `CTE_REF`, generic
`JOIN`/`DELIM_JOIN`/`DEPENDENT_JOIN`, `RECURSIVE_CTE`, `MATERIALIZED_CTE`,
and `EXTENSION_OPERATOR`.

Two of the eligibility rules are exactly the "cannot safely share a
non-serializable computation" concern this section was specifically asked
to track down, and both are worth quoting precisely:

```cpp
// common_subplan_optimizer.cpp:666-677
case LogicalOperatorType::LOGICAL_GET: {
    auto &get = op.Cast<LogicalGet>();
    if (get.bind_data && !get.function.HasSerializationCallbacks() && get.parameters.empty()) {
        // Without serialization callbacks, the serialized form carries only the call parameters
        // (see LogicalGet::Serialize). A scan without positional parameters - e.g., one created through
        // an attached catalog - keeps its identity solely in the bind data, and its options do not
        // identify what it reads, so equal serialized bytes cannot prove that two scans read the same
        // table.
        return false;
    }
    return true;
}
case LogicalOperatorType::LOGICAL_CHUNK_GET:
    // Avoid serializing massive amounts of data (this is here because of the "Test TPCH arrow roundtrip" test)
    return op.Cast<LogicalColumnDataGet>().collection->Count() < 1000;
```

A `LogicalGet` whose bind data is opaque to serialization *and* has no
positional call parameters to serve as a fallback identity check is refused
candidacy outright — its serialized bytes would be indistinguishable from
any other scan built the same way, which is not the same thing as
*actually reading the same data*. A `LogicalColumnDataGet` (an in-memory
constant chunk — the shape `IN_CLAUSE`'s mark-join rewrite produces, among
others) is only eligible below 1000 rows, specifically to avoid the cost of
serializing and byte-comparing a potentially enormous inlined constant just
to check for duplication.

Named parameters do not count as an identity. Most scans carry their default
options as named parameters. Options such as `header = true` say *how* to
read, not *what* to read. So two such scans could serialize the same bytes and
still read different tables. The check looks only at positional parameters.

When the canonical table-index map renumbers the table indices of an
operator's children, it uses `GetExpandedColumnBindings`
(`common_subplan_optimizer.cpp:22`), not `GetColumnBindings`. That helper
looks through `FILTER`, `ORDER_BY`, `LIMIT`, `TOP_N` and `DISTINCT`, and adds
the extra columns of `WINDOW` and `UNNEST` and all join outputs. A join's
`GetColumnBindings` respects its projection map, which can hide columns. If
the map used `GetColumnBindings`, two subtrees whose children differed *only* in their projection maps could
get the same signature and be merged by mistake.
`test/optimizer/common_subplan_projection_map.test` guards against that bug.
It builds a view `v` with two `LEFT JOIN`s and `CASE` columns `a_amt` and
`b_amt`:

```sql
select (select sum(a_amt) from v where typ in ('A','B')),
       (select sum(b_amt) from v where typ in ('A','B'));
-- 11.00   22.00      (with the bug, both columns could read the same subplan)
```

Two *identical* subqueries are shared: the same file expects `CTE` in the
plan for `sum(a_amt)` twice.

And separately from all of this: **volatility is checked per expression,
during the same canonicalization walk that renumbers table indices** —
`expr.IsVolatile()` sets `can_materialize = false`
(`common_subplan_optimizer.cpp:509-511`) for the whole enclosing operator,
and `PlanSignature::Create` returns `nullptr` (no signature at all) the
moment `can_materialize` is false, whatever the serialized bytes would have
looked like. A subtree containing `random()` never even enters the
signature map — it can't be matched against anything, including another,
byte-for-byte-identical occurrence of the exact same `random()` call.

### 7.12.2 Deciding whether a duplicate is worth materializing

Finding two byte-identical subtrees is necessary but not sufficient —
`CommonSubplanFinder::FilterSubplans` (`common_subplan_optimizer.cpp:855-916`)
prunes candidates first, on structural grounds:

- a signature covering only one operator (`OperatorCount() == 1`) is
  dropped — too small to be worth a `CTE_SCAN`'s overhead;
- a signature with only one matching occurrence is dropped — nothing to
  deduplicate;
- **redundant-with-parent pruning**: if this subtree's own parent *also*
  forms a matching signature group, and materializing the parent would
  cover at least as much duplicated work (`parent_count >= subplan_count`,
  each measured as occurrences × operator-count-per-occurrence), the
  smaller, nested subtree is dropped in favor of letting the larger,
  encompassing parent absorb the deduplication — no point creating two
  nested materialized CTEs when the outer one already covers the inner
  one's duplication;
- a signature group where one occurrence's own canonical bindings contain
  a duplicate is dropped outright — a sign of a nested self-similar
  subplan the pass isn't prepared to reason about safely;
- finally, since different occurrences of the "same" canonical subtree
  might expose different *subsets* of live columns downstream, `FilterSubplans`
  picks a **primary occurrence**: whichever occurrence's own bindings are a
  superset of every binding *any* occurrence needs, preferring the widest
  such occurrence, and bails on the whole group if no single occurrence can
  supply every required binding.

The primary-occurrence choice is a method, `SelectPrimarySubplan`, and it runs
twice. `FilterSubplans` calls it as described. `ConvertSubplansToCTEs` calls it
again just before it builds each CTE. By then, a *nested* common subplan may
already have been replaced by `CTE_SCAN`s, so an occurrence may no longer
expose all the bindings it had. If no occurrence can supply them all, the group
is skipped. `test/sql/optimizer/test_common_subplan_nested_bindings.test`
guards against a crash in this case. It uses a five-branch `UNION ALL` over a
`QUALIFY ROW_NUMBER()` view, joined to itself.

Only after this pruning does `CommonSubplanOptimizer::ShouldMaterialize`
(`common_subplan_optimizer.cpp:1242-1245`) apply an actual cost model:

```cpp
bool ShouldMaterialize(const SubplanInfo &subplan_info) const {
    auto &subplan = subplan_info.subplans[0].op.get();
    return CTEInlining::EndsInAggregateOrDistinct(*subplan) || IsSelectiveMultiTablePlan(subplan);
}
```

`EndsInAggregateOrDistinct` is the *exact same* function
`CTEInlining` uses (§7.10.2, point 8) — a repeated aggregate/`DISTINCT`/window
computation is always judged worth sharing. Otherwise,
`IsSelectiveMultiTablePlan` (`common_subplan_optimizer.cpp:1261-1283`)
requires a known estimated cardinality, requires the subtree to span **at
least two base tables** (a single-table repeated scan-and-filter, unless it
ends in an aggregate, is not automatically shared by this rule), and then:
materialize unconditionally if the subtree's own estimated cardinality is
`≤ 2048` rows; otherwise, only materialize if that cardinality is under
**half** of the largest contributing base table's own cardinality — i.e.,
the subtree has to actually be *selective* relative to its biggest input,
not just a repeated near-full join that wouldn't save much by being shared.
Because `COMMON_SUBPLAN` (pass #28) runs after `JOIN_ORDER` (#17) but
*before* the big `STATISTICS_PROPAGATION` pass (#34), the `estimated_cardinality`
this check reads is the lighter-weight estimate the join-order search
itself computed (`src/optimizer/join_order/cardinality_estimator.cpp`) —
the same correction the chapter overview made in §7.6.2 about which
cardinality estimator feeds which downstream decision applies here too.

### 7.12.3 Assembling the replacement

`ConvertSubplansToCTEs` (`common_subplan_optimizer.cpp:918-1107`) builds a
synthetic `LogicalMaterializedCTE`, named `__common_subplan_<n>`, at the
**lowest common ancestor** of every matching occurrence
(`subplan_info.lowest_common_ancestor`), using the primary occurrence
selected in §7.12.2 as the materialized definition, and replaces every
occurrence — including the primary one itself — with a `LogicalCTERef`
(wrapped in a thin re-projection when an individual occurrence's column
order or subset differs from the shared, widened materialization). The new
CTE is constructed with `CTEMaterialize::CTE_MATERIALIZE_DEFAULT`
(`common_subplan_optimizer.cpp:1067-1069`) — but since `COMMON_SUBPLAN` (#28)
runs strictly after both `CTE_INLINING` call sites (#2 and #12) in the fixed
pipeline, no later pass ever reconsiders whether a synthetic
common-subplan CTE should be inlined instead. Given the one-shot,
never-re-entered architecture §7.2 established for the whole pipeline, a
CTE this pass creates is, in effect, always materialized in the plan
DuckDB actually executes — the `CTE_MATERIALIZE_DEFAULT` tag is
structurally correct but practically moot.

Because a plan can contain more than one common-subplan replacement, and
one replacement's materialized definition might itself need to sit
*downstream* of another synthetic CTE for pipeline-scheduling reasons, the
tail of `ConvertSubplansToCTEs` (`common_subplan_optimizer.cpp:1081-1099`)
walks down through consecutive `LogicalMaterializedCTE` children performing
what its own comment calls "CTE pushdown," reordering newly-created CTEs
relative to each other until their nesting order is one the pipeline
scheduler can actually satisfy — this is exactly what §7.12.5's case with
one common subplan containing another is built to exercise.

The whole thing is skipped entirely when the plan contains a DML CTE
(`optimizer.cpp:432-439`, `if (!CTEContainsDML(*plan)) { RunOptimizer(COMMON_SUBPLAN, ...); }`)
— the same guard, and the same reasoning (table statistics captured at plan
time can already be stale across a data-modifying CTE boundary), that also
gates `STATISTICS_PROPAGATION` itself.

### 7.12.4 `test/optimizer/common_subplan_nonserializable.cpp` — the C++ unit test

This is the concrete, executable version of the "non-serializable subplan"
edge case §7.12.1 derived from `OperatorIsSupported`'s `LOGICAL_GET`
special case. The test registers a custom table function, `nonserial_tf`,
whose bind data is explicitly built to resist exactly the kind of
byte-identity sharing `COMMON_SUBPLAN` would otherwise attempt:

```cpp
// test/optimizer/common_subplan_nonserializable.cpp:16-27
struct NonSerializableBindData : public TableFunctionData {
	unique_ptr<FunctionData> Copy() const override {
		throw InternalException("Copy not supported for NonSerializableBindData");
	}
	bool Equals(const FunctionData &other) const override {
		return false;
	}
	bool SupportStatementCache() const override {
		return false;
	}
};
```

and whose actual scan is stateful in a way that would make silent
deduplication produce a visibly wrong answer: it returns exactly one row
(`42`) the *first* time it's invoked, and zero rows every time after (a
`state.emitted` flag flips after the first call). The query:

```cpp
// test/optimizer/common_subplan_nonserializable.cpp:84-92
const string query = "SELECT lhs.c, rhs.c "
                     "FROM (SELECT count(*) AS c FROM nonserial_tf()) lhs "
                     "CROSS JOIN (SELECT count(*) AS c FROM nonserial_tf()) rhs";
auto result = con.Query(query);
REQUIRE(result->Collection().GetValue(0, 0).GetValue<int64_t>() == 1);
REQUIRE(result->Collection().GetValue(1, 0).GetValue<int64_t>() == 1);
```

Two syntactically identical `SELECT count(*) FROM nonserial_tf()`
subqueries. If `COMMON_SUBPLAN` had (incorrectly) deduplicated them into one
shared, once-materialized computation, the second `CTE_SCAN` would read the
*same* result as the first — `1, 1` either way, coincidentally, since both
counts happen to be 1 regardless of whether the function ran once or
twice. So the test's real assertion is the structural one, checked
directly against the optimized `EXPLAIN` text:

```cpp
// test/optimizer/common_subplan_nonserializable.cpp:94-96
REQUIRE_NO_FAIL(con.Query("PRAGMA explain_output='optimized_only'"));
const auto explain = GetExplainPlan(con, query);
REQUIRE(!StringUtil::Contains(explain, "__common_subplan_"));
```

No synthetic `__common_subplan_*` CTE name appears anywhere — confirming
the `LogicalGet` for `nonserial_tf()` (opaque bind data, no serialization
callbacks, no call parameters to fall back on) was correctly refused
signature candidacy by `OperatorIsSupported`, and the two invocations were
left as two genuinely independent scans, each free to call the underlying
table function on its own.

### 7.12.5 `test/optimizer/common_subplan.test_slow`, case by case

**The basic positive case:**

```sql
select t1.s + t2.s
from (select sum(range) s from range(10)) t1,
     (select sum(range) s from range(10)) t2
----
90

explain select t1.s + t2.s from (select sum(range) s from range(10)) t1, (select sum(range) s from range(10)) t2
----
logical_opt	<REGEX>:.*CTE.*
```

Two byte-identical `sum(range) FROM range(10)` subtrees, each ending in an
aggregate — `ShouldMaterialize` says yes via `EndsInAggregateOrDistinct`
alone, independent of any cardinality threshold. `45 + 45 = 90` confirms
correctness.

**The volatility control, same shape, `random()` instead of a deterministic
sum:**

```sql
explain select t1.s + t2.s
from (select sum(random()) s from range(10)) t1,
     (select sum(random()) s from range(10)) t2
----
logical_opt	<!REGEX>:.*CTE.*
```

Identical structure, but `random()` is volatile — `can_materialize` is
false for the whole subtree during canonicalization, no signature is ever
created, and the two calls are correctly left as two independent
`sum(random())` computations, exactly as they must be for the two halves of
the sum to actually be two independent draws.

**A common subplan appearing both inside a hand-written materialized CTE
and outside it — two separate CTEs, not one shared with the user's:**

```sql
with cte as materialized (select sum(range) s from range(10))
select t1.s + t2.s from cte t1, (select sum(range) s from range(10)) t2
----
90

explain with cte as materialized (select sum(range) s from range(10)) ...
----
logical_opt	<REGEX>:.*CTE.*CTE.*
```

`COMMON_SUBPLAN` doesn't try to fold its own synthetic CTE into an
already-existing, user-written one — `MATERIALIZED_CTE` is explicitly
excluded from `OperatorIsSupported`'s allow-list (§7.12.1), so a
materialized CTE node is never itself a signature candidate. Instead, the
*definition inside* the user's CTE (`sum(range) s from range(10)`) is
recognized as identical to the standalone occurrence outside it, and a
*second*, synthetic CTE gets created to share that inner computation — the
`CTE.*CTE` regex confirms two separate CTE nodes appear, not one.

**Ten-way `UNION ALL`, one shared join computed once, ten different column
projections read from it:**

```sql
create table build as select range a, range*2 b, range*3 c, range*4 d from ...;
create table probe as select range e, range*2 f, range*3 g, range*4 h from ...;
create view my_view as from probe join build on (build.a = probe.e);
set disabled_optimizers='unused_columns';

explain
select greatest(a, b, c, d, e, f, g, h) from my_view
union all select greatest(b, f) from my_view
union all select greatest(b, g) from my_view
-- ... seven more variants, each reading a different column subset
----
logical_opt	<REGEX>:(?:.*CTE_SCAN){10}.*$
```

Ten separate query blocks, each selecting a *different* subset of the
underlying join's eight columns — this is exactly why `UNUSED_COLUMNS` has
to be disabled for this test (its own comment explains: without disabling
it, per-branch column pruning would make the ten `probe JOIN build`
occurrences stop being byte-identical, since each would keep only the
columns its own branch reads). With column pruning out of the way, all ten
occurrences of the join canonicalize identically, get deduplicated into
one shared materialization, and each of the ten `UNION ALL` branches reads
from it via its own `CTE_SCAN` — ten scans, one join actually computed.

**Nested common subplans — one candidate contains another, requiring two
CTEs correctly ordered relative to each other:**

```sql
explain
select count(*) from (select distinct range from range(10))
union all
select count(*) from (select distinct range from range(10))
union all
select * from (select distinct range from range(10))
----
logical_opt	<REGEX>:(?:.*CTE){2}.*(?:.*CTE_SCAN){4}.*
```

The comment states the intent directly: "One common subplan contains
another, we get 2 CTEs and 3 CTE scans... this also tests that we set up
the CTEs in the correct order for scheduling pipeline dependencies" — this
is precisely the "CTE pushdown" reordering logic from §7.12.3, confirmed
by a real, executable test rather than only by the source comment
describing it.

**Three levels of nesting:**

```sql
explain
select distinct range from range(10)
union all
select distinct range from range(10)
union all
select range % 2 as range from (select distinct range from range(10)) group by range
union all
select range % 2 as range from (select distinct range from range(10)) group by range
union all
select count(*) from (select range % 2 as range from (select distinct range from range(10)) group by range)
union all
select count(*) from (select range % 2 as range from (select distinct range from range(10)) group by range)
----
logical_opt	<REGEX>:(?:.*CTE){3}.*(?:.*CTE_SCAN){8}.*
```

Confirms the nesting/reordering mechanism scales past two levels: three
distinct nested common subplans, correctly ordered, eight total scan
sites.

**The same shape, with joins instead of aggregates, and a deliberately
"wonky" join condition:**

```sql
-- note that we use a wonky join condition here
-- otherwise statistics propagation and filter pushdown create slight plan differences
select r1.range from range(10) r1 join range(1) r2 on (r1.range::varchar = r2.range::varchar)
union all
select r1.range from range(10) r1 join range(1) r2 on (r1.range::varchar = r2.range::varchar)
union all
... join range(2) r3 on (r2.range::varchar = r3.range::varchar)
union all
... join range(2) r3 on (r2.range::varchar = r3.range::varchar)
union all
... join range(2) r3 on (r1.range::varchar = r3.range::varchar) join range(3) r4 on (r3.range::varchar = r4.range::varchar)
union all
... join range(2) r3 on (r1.range::varchar = r3.range::varchar) join range(3) r4 on (r3.range::varchar = r4.range::varchar)
----
logical_opt	<REGEX>:(?:.*CTE){4}.*(?:.*CTE_SCAN){8}.*
```

The test's own comment about the `::varchar` casts is worth repeating,
since it's a good illustration of how sensitive byte-exact signature
matching is to *other* passes running first: an ordinary integer-typed
join condition here would let `STATISTICS_PROPAGATION`/`FILTER_PUSHDOWN`
simplify or reshape each of the six near-identical query blocks slightly
differently (depending on subtly different cardinality estimates feeding
join reordering), which would make their canonicalized signatures diverge
even though the *SQL* is identical — so the test deliberately routes the
join condition through a `varchar` cast specifically to keep every
occurrence's post-optimization shape identical enough for `COMMON_SUBPLAN`
to still find them, four distinct shared shapes yielding eight total scans.

**A common subplan shared between a hand-written materialized CTE and code
outside it, with explicit ordering preserved:**

```sql
explain with cte as materialized (select sum(range) s from range(10))
select * from cte
union all
select sum(range) s from range(10)
----
logical_opt	<REGEX>:(?:.*CTE){2}.*(?:.*CTE_SCAN){3}.*
```

The comment: "this also tests that we don't reorder materialized CTEs that
were created by hand" — the CTE-pushdown reordering logic from §7.12.3 only
ever reorders *synthetic* CTEs relative to each other (tracked via
`min_cte_idx`/`cte_index` bounds, `common_subplan_optimizer.cpp:1089`); a
user's own, hand-written `MATERIALIZED` CTE is never a candidate for that
reordering, only for having its *inner* computation matched against other
occurrences the way the earlier "both inside and outside" case showed.

**Real TPC-DS queries, confirming the pass fires on genuine analytical SQL,
not just constructed examples:**

```sql
require tpcds
call dsdgen(sf=0.01)
-- q27, q44, q61: each contains a subquery repeated verbatim (or, for q61,
-- two large, nearly-identical join graphs computing related aggregates)
----
logical_opt	<REGEX>:.*CTE.*
```

TPC-DS query 44 in particular repeats an entire correlated-subquery-bearing
aggregate (`avg(ss_net_profit)` over `store_sales` filtered on
`ss_store_sk = 4`, with a `HAVING` clause referencing its own nested
scalar subquery) twice, once for an ascending rank and once for a
descending one; query 61 computes two large, structurally near-identical
join aggregates (`promotional_sales` and `all_sales`) that share almost
their entire join graph, differing only in one extra join and filter for
the promotional case. Both are exactly the shape `COMMON_SUBPLAN` exists to
catch — not contrived test SQL, but real analytical queries where the same
underlying computation is genuinely, unavoidably written twice by a human
author.

### 7.12.6 `test/optimizer/common_subplan_nested_cte.test` — canonical identity of generated bindings

This file's own comment states its purpose precisely: "Extracting a nested
common subplan creates new CTE reader bindings. A later common subplan must
retain the canonical identities of those generated bindings." It runs a
graph-shaped, mutual-friends-style query over `WITH RECURSIVE` CTEs chained
several levels deep (`interest_source` → `interested_people` →
`friends_of_interested`, self-joined), and specifically brackets the query
inside a `foreach do '' 'column_lifetime'` loop with
`debug_verify_column_bindings = true` set:

```sql
-- The first COLUMN_LIFETIME pass narrows the extracted subplan to live columns, so this shape does not
-- form by default. Disabling it exposes the defect, and per-pass verification reports the pass at fault.
```

The point: once `COMMON_SUBPLAN` extracts one nested common subplan into a
synthetic CTE, the new `LogicalCTERef` bindings it generates
(`generated_binding_map`, `common_subplan_optimizer.cpp:1045-1049`) need
their own canonical identity tracked correctly, so that a *second*,
outer-level common-subplan detection — one whose candidate subtree happens
to include a leg that is itself already a reference to the first synthetic
CTE — doesn't get confused about whether two occurrences are really the
same underlying computation. Under the normal pipeline, `COLUMN_LIFETIME`
(pass #26, which runs immediately before `COMMON_SUBPLAN` at #28) narrows
each extracted subplan down to only its live columns before `COMMON_SUBPLAN`
even sees it, which happens to prevent this specific shape from forming at
all — running the query with `column_lifetime` explicitly disabled removes
that incidental protection and directly exercises the canonical-binding
bookkeeping this test is really about. Both loop iterations (pass enabled,
pass disabled) must produce the same correct result — two mutual-friend
pairs, each with a mutual-friend count of 1 — confirming the bookkeeping
holds up whether or not an earlier pass happens to mask the scenario.

### 7.12.7 `test/sql/optimizer/test_common_subplan_table_filters.test` — table filters must not blur base-column identity

This file targets a specific, narrow corner of the canonicalization step:
a pushed-down table filter on a `LogicalGet` is keyed by *column index*, and
two filters on *different* base columns can end up with the same
projection index in different queries — canonicalization has to keep track
of which underlying column a filter actually binds to, not just its
positional index, or two structurally similar-looking but semantically
different filtered scans could be wrongly matched.

```sql
CREATE TABLE csp_filter_key(a INTEGER, b INTEGER);
INSERT INTO csp_filter_key VALUES (1, 0), (2, 2), (3, 0), (4, 4);

-- Filters on different base columns can have the same projection index
SELECT 'a', count(*) FROM csp_filter_key WHERE a > 1
UNION ALL SELECT 'b', count(*) FROM csp_filter_key WHERE b > 1;
----
a	3
b	2

-- IN filters on different base columns must not share a subplan
SELECT 'a', count(*) FROM csp_filter_key WHERE a IN (2, 3, 4)
UNION ALL SELECT 'b', count(*) FROM csp_filter_key WHERE b IN (2, 3, 4);
----
a	3
b	2

-- Genuinely identical filtered subplans remain equivalent
SELECT 'x', count(*) FROM csp_filter_key WHERE a > 1
UNION ALL SELECT 'y', count(*) FROM csp_filter_key WHERE a > 1;
----
x	3
y	3
```

The correctness checks (run first, without inspecting `EXPLAIN` at all)
establish that the *results* are right regardless of what the optimizer
does internally — `a > 1` matches rows `(2,2),(3,0),(4,4)` = 3, `b > 1`
matches `(2,2),(4,4)` = 2. The file then confirms the *mechanism* directly:

```sql
PRAGMA explain_output='optimized_only';
EXPLAIN SELECT 'a', count(*) FROM csp_filter_key WHERE a > 1
UNION ALL SELECT 'b', count(*) FROM csp_filter_key WHERE b > 1;
----
logical_opt	<!REGEX>:.*CTE.*

EXPLAIN SELECT 'x', count(*) FROM csp_filter_key WHERE a > 1
UNION ALL SELECT 'y', count(*) FROM csp_filter_key WHERE a > 1;
----
logical_opt	<REGEX>:.*CTE.*
```

`a > 1` and `b > 1` are structurally similar (same shape of comparison,
same literal) but bind to genuinely different base columns — no CTE is
created, confirming the canonicalization correctly distinguishes them
rather than being fooled by matching projection indices. `a > 1` compared
against itself (under different top-level string labels `'x'`/`'y'`, which
canonicalization also has to see through, exactly as §7.12.1 described for
aliases) *does* produce a shared CTE — the genuinely identical case.

### 7.12.8 `test_common_subplan_cte_binding_order.test` and `test_common_subplan_projection_maps.test`

`test_common_subplan_cte_binding_order.test` ("Materialized common subplans
keep the same projected shape as their CTE refs") builds a small,
realistic view hierarchy — deduplicating views over `orders`/`line_items`
using `QUALIFY row_number() OVER (...) = 1`, feeding into a `sku_agg`
CTE inside an `order_lifecycle` view — and checks that a query referencing
`order_lifecycle` twice, through two structurally different wrapping shapes
(one via a nested `SELECT amount, order_type FROM order_lifecycle`
sub-select feeding a `GROUP BY`, one via a direct `WHERE is_net_order`
filter), still produces the correct joined counts and revenue
(`other 1 75`, `widget_order 2 80`, both with `total_orders = 2`) once
`COMMON_SUBPLAN` (or the underlying view-inlining and column-lifetime
machinery it interacts with) has done whatever restructuring it does — a
realistic-shaped regression test rather than a minimal, hand-built
counterexample, confirming the mechanism survives ordinary view nesting
and window-function-based deduplication idioms, not just synthetic
single-operator subtrees.

`test_common_subplan_projection_maps.test` ("Common subplan materialization
invalidates stale projection maps") is the regression test for the exact
`ClearProjectionMaps` calls found in `ConvertSubplansToCTEs`
(`common_subplan_optimizer.cpp:964-965`, targeted at the primary subplan
when its required, union-of-all-occurrences binding set differs from what
that occurrence originally exposed; and `common_subplan_optimizer.cpp:1105`,
globally, whenever any subplan was actually converted). Its own comment
states the mechanism precisely: replacing subplan occurrences with
`CTE_SCAN`s changes the underlying column bindings that a join's or
filter's positional projection map (built by `COLUMN_LIFETIME`, #26,
*before* `COMMON_SUBPLAN` runs at #28) refers to by index — an un-cleared,
stale projection map built against the pre-replacement bindings would
silently read the wrong column after replacement. The test constructs a
realistic upsert-plus-snapshot-plus-anti-join view pattern (four parallel
`_upserts`/`_snapshot`/`_deletes` table triples feeding `UNION ALL`/`ANTI
JOIN` views) specifically shaped to trigger the stale-projection-map
scenario, and checks that results stay correct once `COMMON_SUBPLAN` has
run — a good illustration of how a rewrite three passes upstream
(`COLUMN_LIFETIME`) can leave state (a projection map keyed by column
position) that a much later pass (`COMMON_SUBPLAN`) has to know to
invalidate rather than silently reuse, and of the general principle from
§7.2 that in a one-shot, never-re-entered pipeline, every pass has to be
individually careful about what state earlier passes left lying around for
it.

## 7.13 `MATERIALIZED_CTE` at the logical-operator level: `materialized_cte_sibling_column_pruning.test`

This test's own name and description point at a different pass than the
three this chapter is centrally about — "CTE input preservation must not
prevent pruning unrelated join inputs" is squarely `UNUSED_COLUMNS`/
`COLUMN_LIFETIME` territory (covered in full depth in this chapter's
sibling file on unused-column pruning) — but it's specifically about how
that pruning interacts with a *materialized* CTE sitting inside a `LATERAL`
join, which is CTE-specific enough to belong here as a brief pointer.

```sql
CREATE TABLE v(id INTEGER PRIMARY KEY);
INSERT INTO v VALUES (1), (2), (3), (4);

foreach disabled '' 'join_order,build_side_probe_side'
SET disabled_optimizers = {disabled};

EXPLAIN SELECT b.value
FROM (SELECT id AS k, 0 AS domain FROM v) a
JOIN LATERAL (
    WITH t AS MATERIALIZED (SELECT a.k AS n)
    SELECT n FROM t
) q ON true
JOIN (
    SELECT b.k, b.domain, b.value
    FROM (
        SELECT id AS k, 0 AS domain,
               struct_pack(id := id) AS unused_payload,
               id AS value
        FROM v
    ) b
) b ON b.k = q.n AND b.domain IS NOT DISTINCT FROM a.domain
WHERE a.k = 1;
----
logical_opt	<!REGEX>:.*unused_payload.*
```

The query deliberately buries a `MATERIALIZED` CTE (`t`) *inside* a
`LATERAL` subquery correlated to `a.k`, alongside an entirely unrelated
join input (`b`) that computes an unused, wide `struct_pack(...)` column.
The materialized CTE's own bookkeeping — the correlation columns it has to
preserve, the guarantees Chapter 6's decorrelation machinery attaches to it
— must not accidentally force the *unrelated* join input's unused struct
column to survive pruning too; the two are independent, and the test
confirms `unused_payload` is absent from both the logical (`optimized_only`)
and physical (`physical_only`) explain output. It's run twice, with
`join_order`/`build_side_probe_side` alternately enabled and disabled (`foreach
disabled ...`), specifically "to exercise the original join order as well
as the default optimizer order" — confirming the pruning result doesn't
depend on which join shape the plan happens to settle into. Correctness
checks follow: `b.value` (the live, needed column) still returns `1`
correctly, and a *separate* query that actually reads `b.unused_payload`
(both the whole struct, `{'id': 1}`, and a single field, `.id → 1`) confirms
the column is preserved correctly whenever something genuinely does need
it — the negative case (case 1) is paired with a positive case proving the
optimizer isn't just always dropping that column, only dropping it when
nothing reads it.

## 7.14 Summary and cross-references

`CTE_INLINING` is the deepest of the three passes covered here, and its
decision procedure — verified directly from `cte_inlining.cpp:135-234` — is
an ordered sequence of short-circuiting checks, not a flat scoring
function: unreferenced-and-side-effect-free removal, an unconditional
side-effect veto, a `DelimGet`-under-nested-`DelimJoin` safety check, an
explicit `MATERIALIZED` hint veto (which this chapter confirmed, via a real
test, beats even a trivially-inlinable single reference), unconditional
inlining at `ref_count == 1`, a forced inline for an explicit `NOT
MATERIALIZED` hint at higher reference counts, a volatility veto, an
aggregate/`DISTINCT`/window veto, a base-table-count-times-reference-count
cost cutoff, and finally a decision that turns on whether the CTE's body is
already cheap (most importantly, already provably empty) or whether a
downstream `LIMIT` could exploit early termination that a shared
materialization barrier would otherwise block. The pass runs twice for a
concretely verifiable reason, not a vague one: `FILTER_PUSHDOWN`'s own
lightweight unsatisfiability detection, running between the two calls, can
turn a CTE's definition into `LOGICAL_EMPTY_RESULT` — flipping
`is_cheap_to_inline` from false to true — and only the second call gets a
chance to act on that.

`CTE_FILTER_PUSHER` answers the "referenced multiple times with different
filters" question precisely (and also pushes join keys into decorrelation
CTEs as a semi join, §7.11.4): different filters across references are fine
to combine with `OR` and push, as long as *every* reference has *some*
filter; a single unfiltered reference blocks the push entirely, with one
narrow, compiler-generated exception (a decorrelation-produced
row-scan/domain-scan pair, tracked via `CTEFilterDependency`) where the
planner has already guaranteed the unfiltered side's needs are a subset of
the filtered side's. An unlabeled, easy-to-miss inline step,
`FilterStatisticsOptimizer`, running between `FILTER_PUSHDOWN` and
`CTE_FILTER_PUSHER`, turns out to be the mechanism behind a real test case
where a cross-relation filter becomes push-eligible only after real
column statistics prove one of its disjuncts always false.

`COMMON_SUBPLAN`'s notion of "syntactically identical" is canonicalized,
serialized-byte equality — table indices, column indices, and even
expression aliases get normalized away before two subtrees are compared —
restricted to an explicit allow-list of operator types, with volatile
expressions and a small number of genuinely non-serializable shapes (an
opaque, parameter-less table function's bind data; an oversized inlined
constant chunk) excluded from candidacy entirely rather than merely
excluded from matching. Whether a genuine duplicate is worth
materializing is answered by the same `EndsInAggregateOrDistinct` helper
`CTE_INLINING` uses, or by a cardinality-and-selectivity heuristic that
only fires above two base tables.

Everything in this part depended on reading the fixed pipeline's exact
call order from `optimizer.cpp`, exactly as §7.2 and §7.4 set up — several
of the concrete mechanisms here (why `CTE_INLINING` needs a second call,
why `CTE_FILTER_PUSHER`'s test case 7 works at all) are only explicable
because a specific *other* pass happens to run in the gap between two calls
to the pass under discussion, which is the one-shot, no-fixpoint
architecture from §7.2 showing up as a genuine, testable consequence rather
than an abstract design note.
