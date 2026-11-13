# Chapter 19 — End-to-End Traces

## 19.1 What this chapter assembles, and how to read it

Every chapter from 3 through 18 did one stage of the pipeline in depth, against
whichever of the book's running queries best exercised that stage. That was
the right way to *learn* each stage, but it means the full journey of any one
query has never, until now, appeared in one place. This chapter closes that
gap. It introduces no new architecture, no new class, no new optimizer pass —
everything asserted here was already proven, with source citations, somewhere
in Chapters 2 through 18. What this chapter adds is only the connective
tissue: the same query, followed continuously from SQL text to result rows,
with a citation at every step pointing back to the chapter that actually did
the verification.

Three queries are traced:

- **§19.2 — Query A**, the join/aggregate/order/limit query, traced through
  every stage Chapters 3, 4, 5, 7, 8, 9, 10, 11, 12, 13, 14, and 15 already
  worked out for it individually, plus the live `EXPLAIN ANALYZE` run
  Chapter 2 captured against a real instance.
- **§19.3 — Query B**, the correlated scalar subquery, traced through binding
  and decorrelation exactly as Chapters 4–6 proved it, and then carried
  *forward* into optimization, physical planning, and execution — stages no
  earlier chapter walked for this specific query. Every claim in that second
  half is explicitly marked as reasoned forward from the general rules
  Chapters 7–13 established, not as an independent re-verification.
- **§19.4 — a third query**, a primary-key point lookup, chosen specifically
  because it is the one shape neither Query A nor Query B ever takes: an ART
  index lookup (§16.10 states this outright — Query A "never uses an ART
  index anywhere in its plan"). Chapter 16 already carries a fully worked
  byte-by-byte trace of almost this exact query; this section reuses that
  trace directly and reasons forward, at lighter confidence, about the stages
  around it that no chapter traced explicitly for this query.

A citation convention is used throughout: **"(Ch. N §M — pulled directly)"**
means the diagram, number, or code excerpt is reproduced from that section
with no new derivation; **"(reasoned forward from Ch. N, following Ch. M's
rules)"** means this chapter is the first place that specific claim appears,
built by applying an already-established general mechanism to a query that
chapter never happened to walk through itself. One place below turned up a
genuine cross-chapter inconsistency while this chapter's diagrams were being
assembled; it is called out explicitly, in a footnote-style aside with a
stated resolution (§19.2.7), rather than silently picking one answer.

## 19.2 Query A, start to finish

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

### 19.2.1 Parse (Ch. 3 §3.7 — pulled directly)

The PEG parser reduces the query text to a token stream and matches it
against `select.gram`/`expression.gram`, producing one `SelectStatement`
whose `SelectNode` holds: a `select_list` of `[ColumnRefExpression(["o",
"custkey"]), FunctionExpression("sum", [ColumnRefExpression(["l",
"extendedprice"])])]` (aliased `"total"`); a `from_table` that is a `JoinRef`
of two `BaseTableRef`s (`orders AS o`, `lineitem AS l`) with `condition` a
`ComparisonExpression(COMPARE_EQUAL, o.orderkey, l.orderkey)`; a
`where_clause` of `ComparisonExpression(COMPARE_LESSTHAN, l.shipdate,
ConstantExpression(DATE '1998-01-01'))`; a one-entry `groups` list
(`o.custkey`); and result modifiers for `ORDER BY total DESC` and `LIMIT 10`.
Nothing here has touched a catalog — the parser doesn't yet know `orders` or
`lineitem` exist, or that `sum` is a real aggregate rather than a typo.

### 19.2.2 Bind and logical plan (Ch. 4 §4.7, Ch. 5 §5.6 — pulled directly)

Binding proceeds `FROM` → `WHERE` → `SELECT`/`GROUP BY` → `ORDER BY`/`LIMIT`,
constructing the logical plan as a byproduct of each step (Ch. 1 Difference
#1; Ch. 4 §4.5). `bind_basetableref.cpp` resolves `orders` and `lineitem`
against the catalog, assigning each a distinct `TableIndex` and directly
constructing a `LogicalGet`; `bind_joinref.cpp` binds `o.orderkey =
l.orderkey` and `plan_joinref.cpp` wraps it in a `LogicalComparisonJoin`.
`WHERE l.shipdate < DATE '1998-01-01'` binds against the join's combined
scope and becomes a `LogicalFilter`. `SUM(l.extendedprice)` resolves through
`FunctionBinder` to the `sum` aggregate's catalog entry and, together with
`GROUP BY o.custkey`, drives a `LogicalAggregate` followed by a
`LogicalProjection`. `ORDER BY total DESC` resolves `total` against the
projection's alias map (not as an independent expression) and becomes a
`LogicalOrder`; `LIMIT 10` becomes a `LogicalLimit`. No `LogicalDependentJoin`
is ever built for this query — there is no correlation anywhere in it.

The unoptimized tree, with Chapter 5 §5.6's exact `ColumnBinding`s (`orders`
assigned `table_index = 0`, `lineitem` assigned `table_index = 1`, matching
real TPC-H column order: `orderkey`/`custkey` are `orders`' columns 0/1;
`extendedprice`/`shipdate` are `lineitem`'s columns 5/10):

```
┌──────────────────────────────────────────────────────────────────────────┐
│ LOGICAL_LIMIT  (LogicalLimit)                                             │
│   limit = 10, offset = 0                                                  │
└──────────────────────────────────┬───────────────────────────────────────┘
┌───────────────────────────────────▼───────────────────────────────────────┐
│ LOGICAL_ORDER_BY  (LogicalOrder)                                          │
│   orders = [ { expr: ColumnBinding{2,1} ("total"), ASCENDING: false } ]    │
└──────────────────────────────────┬───────────────────────────────────────┘
┌───────────────────────────────────▼───────────────────────────────────────┐
│ LOGICAL_AGGREGATE_AND_GROUP_BY  (LogicalAggregate)                        │
│   groups      = [ ColumnBinding{0,1} ]        -- o.custkey                │
│   expressions = [ SUM(ColumnBinding{1,5}) ]    -- SUM(l.extendedprice)    │
└──────────────────────────────────┬───────────────────────────────────────┘
┌───────────────────────────────────▼───────────────────────────────────────┐
│ LOGICAL_FILTER  (LogicalFilter)                                           │
│   expressions = [ ColumnBinding{1,10} < DATE '1998-01-01' ]  -- l.shipdate│
└──────────────────────────────────┬───────────────────────────────────────┘
┌───────────────────────────────────▼───────────────────────────────────────┐
│ LOGICAL_COMPARISON_JOIN  (LogicalComparisonJoin, join_type = INNER)       │
│   conditions = [ {0,0} ("orderkey") = {1,0} ("orderkey") ]                │
└──────────────┬─────────────────────────────────────┬─────────────────────┘
┌──────────────▼───────────────┐         ┌────────────▼───────────────────┐
│ LOGICAL_GET (table_index = 0) │         │ LOGICAL_GET (table_index = 1)  │
│   orders — table_filters: {}  │         │   lineitem — table_filters: {} │
└───────────────────────────────┘         └──────────────────────────────┘
```

This *is* `BoundStatement::plan` — the literal output of binding, not a
translation of some prior representation (Ch. 4 §4.5.1, Ch. 1 Difference #1).

### 19.2.3 Optimize (Ch. 7 §7.7, Ch. 8 §8.2/§8.9 — pulled directly)

Chapter 7 walked all 45 numbered `RunOptimizer` call sites against this
exact tree (see the pass-numbering aside below).
Roughly three dozen are no-ops for Query A — no CTEs, no `IN` list, no
correlated subquery for `DELIMINATOR`/`UNNEST_REWRITER` to clean up, no
window functions, only one `GROUP BY` key, only one join edge for `JOIN_ORDER`
to consider (Chapter 8 confirms this directly: Query A "only has one join, so
it never exercises anything [Chapter 8] cares about — with one join there is
nothing to *order*"; Chapter 8 introduces its own three-way Query C
specifically because Query A can't exercise DPhyp at all). Three passes do
real, visible work:

- **`FILTER_PUSHDOWN`** (#6) — `l.shipdate < DATE '1998-01-01'` references
  only `lineitem` columns, so it travels straight through the join and
  becomes a `table_filters` entry on `lineitem`'s own `LogicalGet`, where
  storage-layer zonemaps (Ch. 17 §17.5) can use it to skip whole row groups.
- **`UNUSED_COLUMNS`/`COLUMN_LIFETIME`** (#23/#26, and again at #38/#39 —
  see the pass-numbering aside below) — prune each `LogicalGet`'s exposed
  columns down to exactly what survives: `orders` to `[orderkey, custkey]`,
  `lineitem` to `[orderkey, extendedprice]`, with `shipdate` consumed
  entirely by the pushed-down filter and never entering the output chunk.
- **`TOP_N`** (#32) — `ORDER BY total DESC LIMIT 10` satisfies every
  condition in `TopN::CanOptimize` (constant limit, no offset, well under
  both the 0.7%-of-cardinality and 5000-row absolute thresholds), so
  `LogicalOrder`+`LogicalLimit` fuse unconditionally into one `LogicalTopN`.

`STATISTICS_PROPAGATION` (#34) and `BUILD_SIDE_PROBE_SIDE` (#27) also do
real, but invisible-to-the-tree-shape, work: the former propagates
`lineitem.shipdate`'s min/max/histogram to estimate the filter's selectivity
(feeding Chapter 9's physical-operator choice and Chapter 14's
memory-budgeting, without proving anything always-true/always-false here);
the latter decides which side of the join becomes the hash table's build
side.

> **A pass-numbering aside.** Chapter 7 §7.4 is this book's single
> authoritative, line-cited enumeration of the fixed 45-pass pipeline
> (`optimizer.cpp:253`–`561`). It counts `EXPRESSION_REWRITER` as #1 and
> leaves out `REMOTE_PUSHDOWN` and the extension hooks. Every chapter uses its
> numbers. The passes this chapter uses are: `FILTER_PUSHDOWN` #6 (line 286),
> `DELIMINATOR` #10 (line 316), `UNNEST_REWRITER` #22 (line 395),
> `UNUSED_COLUMNS` #23, `COLUMN_LIFETIME` #26, `BUILD_SIDE_PROBE_SIDE` #27
> (line 426), `TOP_N` #32 (line 459), `STATISTICS_PROPAGATION` #34 (line 476),
> and the second `UNUSED_COLUMNS`/`COLUMN_LIFETIME` pair #38/#39.
>
> Two passes in the pipeline do nothing for any query in this chapter:
>
> - **`SELF_JOIN_AGGREGATE`** (#3, `optimizer.cpp:268`) runs right after the
>   first `CTE_INLINING`. Its source comment says it rewrites "a self-join on
>   equal keys and one inequality, read only through its keys by a parent
>   that ignores duplicates, into GROUP BY keys HAVING min(x) <> max(x) over
>   one scan." It must run before filter pushdown, which would hide the
>   pattern.
> - **`REDUNDANT_ORDER_KEYS`** (#21, `optimizer.cpp:389`) runs after
>   `JOIN_ELIMINATION`. It drops `ORDER BY` keys that follow a unique prefix.
>   For example, if `id` is unique, then in `ORDER BY id, name` the key `name`
>   can never break a tie, so it is dropped. It works out which column sets
>   are unique from the operators below (`ApplyFunctionalDependencies` in
>   `src/optimizer/remove_redundant_order_keys.cpp`).
>
> Neither changes Query A, Query B or the point lookup: none of them has a
> self-join, and Query A's `ORDER BY` has a single key.

After optimization:

```
LogicalTopN (ORDER BY total DESC LIMIT 10)
  └─ LogicalAggregate (GROUP BY o.custkey; SUM(l.extendedprice))
       └─ LogicalComparisonJoin (o.orderkey = l.orderkey)
            ├─ LogicalGet (orders — columns: orderkey, custkey)
            └─ LogicalGet (lineitem — columns: orderkey, extendedprice;
                            filter pushed into scan: shipdate < DATE '1998-01-01')
```

### 19.2.4 Physical plan (Ch. 9 §9.4.2, §9.6 — pulled directly)

`ColumnBindingResolver` walks this tree converting every
`{table_index,column_index}` `ColumnBinding` into a flat
`BoundReferenceExpression{index}` — a positional offset into whatever
`DataChunk` will actually arrive at runtime, since a `PhysicalOperator` never
carries table indices (Ch. 5 §5.5.3, Ch. 9 §9.4.2). Walking it stage by
stage, exactly as Chapter 9 §9.6 did:

- `LogicalGet(orders)` → `bindings = [{0,0}, {0,1}]`.
- `LogicalGet(lineitem)` → `bindings = [{1,0}, {1,5}]` (local positions —
  `extendedprice` is `lineitem`'s *pruned-output* column 1, even though it's
  column 5 of the full table).
- `LogicalComparisonJoin` (the hand-written binary-join case): left resolved
  against `[{0,0},{0,1}]` (`o.orderkey` → `BoundReferenceExpression{0}`),
  right resolved independently against `[{1,0},{1,5}]` (`l.orderkey` →
  `BoundReferenceExpression{0}`, before the concatenation shift), then the
  two lists are concatenated left-first: `bindings = [{0,0}, {0,1}, {1,0},
  {1,5}]` — position 0 = `orders.orderkey`, 1 = `orders.custkey`, 2 =
  `lineitem.orderkey`, 3 = `lineitem.extendedprice`.
- `LogicalAggregate`: `groups = [{0,1}]` → position 1 →
  `BoundReferenceExpression{1}`; `SUM`'s argument `{1,5}` → position 3 →
  `BoundReferenceExpression{3}`. Output bindings become `[{2,0}, {2,1}]`.
- `LogicalTopN`: sort key `{2,1}` ("total") → position 1 →
  `BoundReferenceExpression{1}`.

The join has a pure equality condition, so `PlanComparisonJoin`'s decision
tree (Ch. 8 §8.10, Ch. 9 §9.5) routes it straight to `PhysicalHashJoin` — no
range/merge/IE-join logic ever gets consulted, since `has_equality` is
checked first and is unconditionally true here. The aggregate has one
integer `GROUP BY` key and one combinable `SUM`; §9.6 walks
`CanUsePartitionedAggregate` (rejected — the join isn't one of the
partition-preserving pass-through cases) and `CanUsePerfectHashAggregate`
(depends on whether statistics propagation proved a tight `custkey` range —
resolved empirically below, in §19.2.7). `LogicalTopN` becomes `PhysicalTopN`
one-to-one. The final tree, capped by a `PhysicalResultCollector`:

```
┌───────────────────────────────────────────────────────────────────┐
│ PhysicalResultCollector                                            │
└──────────────────────────────┬────────────────────────────────────┘
┌───────────────────────────────▼───────────────────────────────────┐
│ PhysicalTopN            ORDER BY total DESC LIMIT 10                │
│   sort key: BoundReferenceExpression{1}                            │
└──────────────────────────────┬────────────────────────────────────┘
┌───────────────────────────────▼───────────────────────────────────┐
│ PhysicalHashAggregate   GROUP BY custkey{1}; SUM(extendedprice{3})  │
└──────────────────────────────┬────────────────────────────────────┘
┌───────────────────────────────▼───────────────────────────────────┐
│ PhysicalHashJoin        condition: left{0} = right{0}               │
│   build: orders   probe: lineitem                                   │
└──────────────┬─────────────────────────────────┬───────────────────┘
┌──────────────▼───────────────┐    ┌──────────────▼──────────────────┐
│ PhysicalTableScan (orders)    │    │ PhysicalTableScan (lineitem)      │
│   projects: [orderkey,custkey]│    │   projects: [orderkey,extendedprice]
│                                │    │   table_filters: shipdate < ...    │
└───────────────────────────────┘    └────────────────────────────────────┘
```

### 19.2.5 Pipelines (Ch. 11 §11.9 — pulled directly, with its own caveat kept)

Four sinks break this tree into four pipelines, in a strict linear
dependency chain (Chapter 11's own stated caveat is preserved here: this
decomposition is "this book's own reasoned reconstruction, applying the
Sink/Source/Operator rules," not a transcript of a live `EXPLAIN ANALYZE`
run against this checkout):

```
Pipeline 1 (sink: PhysicalHashJoin, build)
  source: PhysicalTableScan(orders)
  sink:   PhysicalHashJoin.Sink   (build the hash table)
        │ REQUIRED — Pipeline 2 needs the finalized hash table
        ▼
Pipeline 2 (sink: PhysicalHashAggregate)
  source: PhysicalTableScan(lineitem, shipdate filter pushed in)
  op:     PhysicalHashJoin.Execute   (probe — mid-pipeline here, not a sink)
  sink:   PhysicalHashAggregate.Sink (accumulate groups)
        │ REQUIRED — Pipeline 3 needs every group finalized
        ▼
Pipeline 3 (sink: PhysicalTopN)
  source: PhysicalHashAggregate.GetData (sink-and-source, §11.5.1)
  sink:   PhysicalTopN.Sink (bounded top-10 heap)
        │ REQUIRED — Pipeline 4 needs the finished top-10
        ▼
Pipeline 4 (sink: PhysicalResultCollector)
  source: PhysicalTopN.GetData
  sink:   PhysicalResultCollector.Sink
```

No two pipelines here can run concurrently with each other — all available
parallelism is *within* a pipeline (many threads over disjoint row groups of
`orders` in Pipeline 1, of `lineitem` in Pipeline 2), which Chapter 11 draws
out as the general "invest cores in one pipeline" lesson for a query this
shape.

### 19.2.6 Vectors, expressions, execution (Ch. 10 §10.8, Ch. 12 §12.13, Ch. 15 §15.7 — pulled directly)

Following one 2048-row chunk of `lineitem` through Pipeline 2 (Ch. 10 §10.8):
the scan hands back `orderkey`/`extendedprice` as `FLAT_VECTOR`s (or skips
row-level work entirely if the row group's `shipdate` zonemap already
resolved it); the pushed-down `shipdate` predicate runs through
`ExpressionExecutor::SelectExpression`, writing surviving indices into a
`SelectionVector` — say 1,400 of 2048 rows pass — and `DataChunk::Slice`
turns both output columns into `DICTIONARY_VECTOR`s over the same original
data, no copy. The hash join's probe side (`orderkey`, still dictionary-
encoded) reaches `JoinHashTable::Probe`, where `ToUnifiedFormat` treats
`DICTIONARY_VECTOR` as one of the "free" formats and reuses the existing
selection vector rather than flattening. Whatever survives the join is
itself another selection vector layered over the already-dictionary-encoded
`extendedprice`, and `PhysicalHashAggregate`'s `SUM` state update reads it
through the same uniform machinery, oblivious to how many filtering layers
came before.

At the expression level (Ch. 12 §12.13): `l.shipdate < DATE '1998-01-01'`
is a `BoundFunctionExpression` around `<`, dispatched (as a pushed-down
`TableFilter`) through the comparison function's `Select` callback rather
than its value-producing path; `o.custkey` in the final projection is a bare
`BoundReferenceExpression`, handled by `execute_reference.cpp` with no
arithmetic at all; `SUM(l.extendedprice)` is *not* dispatched through
`ExpressionExecutor::Execute`'s switch at all — `BOUND_AGGREGATE` isn't one
of that switch's cases — because an aggregate carries per-group accumulator
state across chunks, a fundamentally different evaluation shape than
`ExpressionExecutor`'s "read a chunk, produce a chunk, forget everything"
(Ch. 14 covers the accumulator machinery itself). None of this evaluation is
compiled per-query: Chapter 13's exhaustive negative evidence establishes
that DuckDB has no JIT anywhere in this path, and every one of the
specialized inner loops invoked above (`BinaryExecutor::Execute<int32_t,
int32_t, bool, GreaterThan>` for the date comparison, the `SUM`
accumulator's own templated update loop) was compiled once, when DuckDB
itself was built.

`PhysicalTopN`'s bounded heap reuses a piece of Chapter 15's sorting
machinery rather than ignoring it: `TopNHeap` bounds its candidate set to
exactly `limit + offset = 10` entries regardless of how many `(custkey,
total)` groups feed into it, and compares candidates using the same
normalized, byte-comparable `sort_key` encoding `CreateSortKeyHelpers`
exposes (Ch. 15 §15.4, §15.7) — it does *not* run Chapter 15's full
thread-local-run-plus-merge machinery (that's for Chapter 15's own
unbounded-sort running example, Query C — no shortcut exists there because
every row genuinely has to land somewhere in a total order).

### 19.2.7 Ground truth: `EXPLAIN ANALYZE`, and resolving one discrepancy (Ch. 2 §2.6 — pulled directly)

Chapter 2 §2.6 ran this exact query, on synthetic `orders`/`lineitem` tables
(100,000 orders, ~320,000 qualifying `lineitem` rows), against a real,
locally built stable DuckDB (v1.1.3) — and states explicitly that "Chapter
19's full end-to-end trace of Query A leans on it directly." The live tree:

```
TOP_N            Top: 10  Order By: sum(l.extendedprice) DESC        (10 rows)
  PROJECTION     __internal_decompress_integral_bigint(#0,0), #1     (1000 rows)
    PERFECT_HASH_GROUP_BY   Groups: #0   Aggregates: sum(#1)         (1000 rows)
      PROJECTION   custkey, extendedprice                            (320648 rows)
        PROJECTION __internal_compress_integral_{uinteger,usmallint}(...) (320648 rows)
          HASH_JOIN  Join Type: INNER   Conditions: orderkey=orderkey
                      Build Min: 0  Build Max: 99999                  (320648 rows, 0.02s)
            TABLE_SCAN orders     Projections: orderkey, custkey     (100000 rows)
            TABLE_SCAN lineitem   Projections: orderkey, extendedprice
                       Filters: shipdate < '1998-01-01'::DATE AND
                                shipdate IS NOT NULL                  (320648 rows)
```

Everything this section and §19.2.4 derived matches, node for node: the
filter really is on `lineitem`'s `TABLE_SCAN` leaf; both scans really do
project down to exactly the two needed columns; the join really is a
`HASH_JOIN` with `orders` on the build side (`Build Min`/`Max` of `0`/`99999`
matches `orders.custkey`'s actual range, and `Total Time: 0.0261s` is
overwhelmingly dominated by the join's own `0.02s`, exactly as predicted by
"most of a hash join's cost is the one-time build+probe, not anything
downstream of it").

> **A resolved discrepancy: `HASH_GROUP_BY` vs. `PERFECT_HASH_GROUP_BY`.**
> Chapter 9 §9.6's worked physical trace explicitly *assumed*, for
> expository purposes, "that no such statistics were available" for
> `custkey`, and so walked `CanUsePerfectHashAggregate` failing and fell
> through to a plain `PhysicalHashAggregate`. This live run shows the
> opposite: the optimizer's `STATISTICS_PROPAGATION` pass *did* prove a
> tight, known range for `custkey` (visible directly on the join's own
> `Build Min: 0` / `Build Max: 99999` annotation), so
> `CanUsePerfectHashAggregate` succeeds and the real operator is
> `PERFECT_HASH_GROUP_BY` — confirmed independently by Chapter 14 §14.9,
> which cites this exact same live trace as the reason "Chapter 2 §2.6's
> live `EXPLAIN ANALYZE` trace of this book's own Query A showed
> `PERFECT_HASH_GROUP_BY`, not `HASH_GROUP_BY`." Per this chapter's stated
> policy of preferring whichever source did the deeper, most
> source-verified analysis: a live run against a real instance outranks a
> hypothetical "assume statistics aren't available" branch taken purely for
> illustration, so `PERFECT_HASH_GROUP_BY` is this chapter's answer for
> what Query A's aggregate operator actually is. Chapter 9's own
> `PhysicalHashAggregate`/`HASH_GROUP_BY` walk remains correct as a
> description of *what happens when the statistics aren't available* — a
> real and useful branch of the decision tree — it just isn't the branch
> this specific query, against this specific data, actually takes.

Two `PROJECTION` nodes the hand-drawn diagrams above don't show also appear
in the live tree, compressing/decompressing join-key integer widths
(`__internal_compress_integral_uinteger`/`usmallint`,
`__internal_decompress_integral_bigint`) — Chapter 2 flags these explicitly
as a case where "a real `EXPLAIN ANALYZE` tree is routinely a few nodes
busier than the simplified plan diagrams this book draws by hand," and
defers chasing down exactly why until the reader has Chapters 10 and 17's
vocabulary. This chapter doesn't chase it further either, for the same
reason: it belongs to storage-level integer-width compression (Ch. 17
§17.4.2), not to anything this chapter's assembly needs to resolve.

## 19.3 Query B, start to finish

```sql
SELECT c.name,
       (SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey) AS n_orders
FROM customer c;
```

### 19.3.1 Parse (Ch. 3 §3.8 — pulled directly)

The outer query parses as an ordinary `SimpleSelect`. The interesting part is
`(SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey)`, which matches
`SubqueryReference <- Parens(SelectStatementInternal)` — the entire inner
query is parsed recursively with the same machinery §19.2.1 just used,
producing its own complete `SelectNode`. The result is a plain
`SubqueryExpression` with `subquery_type = SubqueryType::SCALAR`. Critically,
**nothing here is marked "correlated"** — `c.custkey` inside the subquery's
`WHERE` clause looks, syntactically, exactly like any other qualified column
reference. Correlation is a binding-time, scope-dependent fact, not a
syntactic one; the parser's tree for Query B genuinely has no field that
could record it yet.

### 19.3.2 Bind: where the correlation is discovered (Ch. 4 §4.8, Ch. 5 §5.7 — pulled directly)

Binding the outer `SELECT`-list's subquery expression creates a *child*
`Binder` (`Binder::CreateBinder`, parent = the outer `Binder`) with its own,
independent `BindContext`, populated only from the subquery's own `FROM
orders o`. `o.custkey` resolves locally, the ordinary way. `c.custkey` does
not — the child scope never registered alias `c` — and
`ExpressionBinder::BindExpression` falls through to `BindInEnclosingScope`,
which walks the `parent` link back to the outer `Binder` and resolves
`c.custkey` there, at `Depth() = 1`. `ExtractCorrelatedExpressions` notices
`Depth() > 0` and records the reference on the child `Binder`'s own
`correlated_columns` list — and a non-empty `correlated_columns` list at the
end of subquery binding is exactly the boolean (`BoundSubqueryExpression::
IsCorrelated()`) that routes `Binder::PlanSubquery` toward
`PlanCorrelatedSubquery` instead of the uncorrelated path Query A's shape
would have taken had it had any subquery at all. `PlanCorrelatedSubquery`
builds the marker node:

```
┌───────────────────────────────────────────────────────────────────────────┐
│ LOGICAL_PROJECTION   expressions = [ c.name{0,1}, n_orders{1,0} ]          │
└──────────────────────────────────────┬────────────────────────────────────┘
┌───────────────────────────────────────▼───────────────────────────────────┐
│ LOGICAL_DEPENDENT_JOIN (join_type = INNER)                                 │
│   correlated_columns = [ ColumnBinding{0,0} ]  -- c.custkey                │
└──────────────┬───────────────────────────────────────┬────────────────────┘
┌──────────────▼───────────────┐         ┌──────────────▼──────────────────┐
│ LOGICAL_GET (customer c)      │         │ LOGICAL_AGGREGATE  count(*)/()   │
│  table_index = 0               │         │  aggregate_index = 1             │
└────────────────────────────────┘         └──────────────┬───────────────────┘
                                                              │
                                            ┌──────────────────▼─────────────┐
                                            │ LOGICAL_FILTER                  │
                                            │  o.custkey{3,0} = c.custkey{0,0}│
                                            │  (CORRELATED — reaches outside  │
                                            │   this subtree entirely)        │
                                            └──────────────────┬─────────────┘
                                                                 │
                                            ┌──────────────────▼─────────────┐
                                            │ LOGICAL_GET (orders, idx = 3)    │
                                            └─────────────────────────────────┘
```

This node is explicitly documented as transient (`logical_dependent_join.hpp`
calls it "planned but not yet flattened"). By the time `Binder::Bind` returns
a `BoundStatement`, this is what sits in the tree. Flattening comes next, in
`Planner::CreatePlan`, still before the optimizer's first pass.

### 19.3.3 Decorrelation: `FlattenDependentJoins` (Ch. 6 §6.3–§6.6 — pulled directly)

This is the star of Query B's trace, and Chapter 6 walked it in exhaustive
detail.

Which tree comes out of flattening depends on the setting
`delim_join_as_cte`. It defaults to `true` (`src/common/settings.json`; the
setting is marked deprecated). With it on,
`FlattenDependentJoins::DecorrelateIndependent`
(`src/planner/subquery/flatten_dependent_join.cpp:281-292`) calls
`DelimJoinCTERewriter::Rewrite` right after flattening. That step rewrites the
delim join into plain joins over two materialized CTEs, and `VerifyNoDelim`
(`delim_join_cte_rewriter.cpp:33-45`) checks that no delim join is left
(Chapter 6 §6.4.1).

- **The default (CTE) form.** The optimizer starts from this tree: `customer`
  is stored once in a CTE `__duckdb_delim_<n>`; D is a real
  `GROUP BY c.custkey` over that CTE, stored in a second CTE
  `__duckdb_delim_dedup_<m>`; and the joins read both CTEs through
  `LogicalCTERef`s. `DELIMINATOR` (#10) finds no delim join. The matching
  clean-up (`GeneratedDedupRefEliminator`, `GeneratedDomainJoinEliminator`)
  has already run inside the CTE step. The physical plan has `CTE`/`CTE_SCAN`
  operators and ordinary hash joins, not a
  `PhysicalLeftDelimJoin`/`PhysicalRightDelimJoin`. Which joins the CTE step
  removes for Query B was not checked against a live plan (this book had no
  built binary of this checkout).
- **The delim-join form.** With `SET delim_join_as_cte = false`, the delim
  join stays in the tree. This is the form Chapter 6 explains in detail, so
  the tree below, and the delim-join parts of §19.3.4, §19.3.5, §19.3.6 and
  §19.3.7, trace Query B with the setting off.

The four algebraic steps below are the same in both forms.

The rewrite proceeds in four algebraic steps:

1. **Duplicate-eliminate the parameter.** `D := δ[d: c.custkey](customer)`.
   The dependent join is rewritten so its left input shrinks from "every row
   of `customer`" to "every *distinct* `custkey` value."
2. **Push ⋈ᵈ through the aggregate.** `D ⋈ᵈ Γ[count(*)]/() (Q)` becomes
   `Γ[count(*)]/(d) (D ⋈ᵈ Q)` — the correlation column joins the aggregate's
   own `GROUP BY` list, converting per-row "Apply" semantics into an
   ordinary, single-pass grouped aggregation.
3. **Push ⋈ᵈ through the filter.** `D ⋈ᵈ σ[o.custkey=d](orders)` commutes
   freely to `σ[o.custkey=d](D ⋈ᵈ orders)` — filters don't care how many
   distinct parameters produced the rows in front of them.
4. **Eliminate ⋈ᵈ.** With nothing left below referencing `d` except an
   equality predicate against it, the dependent join becomes an ordinary
   join, and that predicate folds directly into the join condition:
   `D ⋈[o.custkey=d] orders`.

The final, ordinary join tree (Ch. 6 §6.3):

```
┌───────────────────────────────┐
│ LogicalProjection              │  π [c.name, n_orders]
└──────────────┬─────────────────┘
┌──────────────▼─────────────────┐
│ LogicalComparisonJoin           │  c.custkey = d   (LEFT — see below)
└──────┬────────────────┬────────┘
┌──────▼──────┐   ┌─────▼───────────────────────────┐
│ LogicalGet    │   │ LogicalAggregate  count(*)/(d)    │
│ customer c    │   └──────────────┬───────────────────┘
└──────────────┘                  │
                     ┌─────────────▼─────────────────────┐
                     │ LOGICAL_DELIM_JOIN                  │  o.custkey = d
                     └──────┬───────────────────┬─────────┘
                 ┌──────────▼─────────┐  ┌──────▼──────┐
                 │ "D": customer's      │  │ LogicalGet   │
                 │ correlation column   │  │ orders o     │
                 │ (duplicate_eliminated│  └─────────────┘
                 │  _columns metadata,  │
                 │  §6.4 — no reified   │
                 │  δ node exists)       │
                 └──────────────────────┘
```

Two details from Chapter 6 are load-bearing for anything reasoned forward in
the next sections: **first**, `LOGICAL_DELIM_JOIN` is not its own C++ class —
it is a `LogicalComparisonJoin` whose `type` field is set to the
`LOGICAL_DELIM_JOIN` enum value and which carries an extra
`duplicate_eliminated_columns` field (here, one expression: `c.custkey`); the
actual computation of D is deferred all the way to the *physical* plan, where
`PhysicalPlanGenerator::PlanDelimJoin` builds a real `PhysicalHashAggregate`
(confusingly field-named `distinct`) to compute it. **Second**, the *outer*
join (`customer ⋈ Γ`) is forced to `LEFT`, not `INNER`, by `PushDownAggregate`'s
null-propagation check: `COUNT`'s `PropagatesNullValues()` is false, so an
`INNER` join would silently drop
every customer with zero matching orders instead of reporting `n_orders =
0`. This is the single fact in this whole trace that would be easiest to get
wrong by analogy with Query A's plain `INNER` join, and Chapter 6 catches it
in running code, not just in prose.

The decision is made in two steps inside `PushDownAggregate`:

```cpp
// src/planner/subquery/flatten_dependent_join.cpp:910-913
for (auto &expr : plan->expressions) {
	auto &aggr_expr = expr->Cast<BoundAggregateExpression>();
	propagate_null_values &= !aggr_expr.IsVolatile() && aggr_expr.PropagatesNullValues();
}
```

```cpp
// src/planner/subquery/flatten_dependent_join.cpp:945-949
// dropping missing keys is only equivalent when the single recovered row would be all-NULL:
// repeated empty sets need one row per set and GROUPING() outputs are non-NULL constants
auto drop_missing_keys =
    !any_join && propagate_null_values && empty_set_count == 1 && aggr.grouping_functions.empty();
auto join_type = drop_missing_keys ? JoinType::INNER : JoinType::LEFT;
```

For Query B, `COUNT(*)` does not propagate `NULL`s, so `propagate_null_values`
becomes false, `drop_missing_keys` is false, and the join type is `LEFT`. The
code also treats volatile aggregates and `GROUPING SETS` with more than one
empty set with care, but Query B has neither.

> **Pass numbers for this section.** `DELIMINATOR` is pass **#10**
> (`optimizer.cpp:316`) and `UNNEST_REWRITER` is pass **#22**
> (`optimizer.cpp:395`), in Chapter 7 §7.4's numbering, which Chapter 6
> §6.10 uses too; see the aside in §19.2.3.

### 19.3.4 Optimizer passes on the decorrelated tree (reasoned forward from Ch. 6 §6.10, following Ch. 7/Ch. 8's general rules)

No chapter runs Query B specifically through the full ~45-pass pipeline the
way Chapter 7 §7.7 did for Query A. What follows applies the general rules
those chapters already established to Query B's specific post-decorrelation
shape. It assumes `SET delim_join_as_cte = false`, so that the delim join
reaches the optimizer; with default settings it does not (see §19.3.3).

`DELIMINATOR` (#10) gets a real, non-trivial input for the first time in this
book: a genuine `LOGICAL_DELIM_JOIN` sits in the tree. §6.6's "decoupling"
rule is directly relevant here and is worth flagging as a distinct,
lower-confidence aside rather than folding it into the main trace (see
§19.3.7 below) — the safer, more conservative assumption for the rest of
this section is that the delim join survives `DELIMINATOR` intact, exactly
as §19.3.3 drew it. `UNNEST_REWRITER` (#22) has nothing to do here — its
target is `LogicalUnnest`/window-adjacent shapes this query doesn't have.
`UNUSED_COLUMNS`/`COLUMN_LIFETIME` prune `customer` down to `[custkey,
name]` and `orders` down to `[custkey]` (no other column of either table is
referenced anywhere in this plan). `JOIN_ORDER`'s DPhyp search treats the
`LOGICAL_DELIM_JOIN` as a single opaque relation in the *outer* hypergraph
rather than dissolving it — this is the specific, source-confirmed behavior
Chapter 6 §6.10 documents (`RelationManager::ExtractJoinRelations`, its
`LOGICAL_DELIM_JOIN` case at `relation_manager.cpp:733-794`), including the detail that DuckDB tried the
more aggressive "reorder freely" alternative once and reverted it after it
produced worse plans in practice. With only two joins total, chained rather
than branching, there is only one order to consider either way —
structurally the same non-event `JOIN_ORDER` was for Query A's single join,
just for a different underlying reason (opacity, not absence of alternatives).
`BUILD_SIDE_PROBE_SIDE` (#27) applies its usual `GetBuildSize` cost estimate
independently to each of the two comparison joins; for the outer `LEFT` join,
`customer` sits on the left (the side a `LEFT` join must preserve) and the
grouped-`orders` aggregate result sits on the right — already the natural
build-side candidate, since a grouped result is generally no larger than the
table it was grouped from, so there is no structural reason to expect a flip
here, though this chapter does not have a live run to confirm it either way
for this specific query, unlike §19.2.7's Query A check.

### 19.3.5 Physical plan (reasoned forward from Ch. 6 §6.4, following Ch. 8 §8.10/Ch. 9 §9.5's decision tree)

Like §19.3.4, this section assumes `SET delim_join_as_cte = false`. With
default settings there is no `PlanDelimJoin` call; see
§19.3.3.

Both remaining joins carry pure equality conditions (`c.custkey = d` and
`o.custkey = d`), so `PlanComparisonJoin`'s decision tree routes both to the
hash-join family exactly as it did for Query A's single join in §19.2.4 —
`has_equality` is checked first, unconditionally true for both, and neither
range-join branch is ever reached. The inner one, tagged `LOGICAL_DELIM_JOIN`,
is planned by `PhysicalPlanGenerator::PlanDelimJoin` rather than the plain
`PlanComparisonJoin` path: it first builds the underlying hash join via
`PlanComparisonJoin` (Ch. 6 §6.4's own code citation, `plan_delim_join.cpp:
27-34`), then wraps it as a `PhysicalLeftDelimJoin`/`PhysicalRightDelimJoin`
(the `LEFT`/`RIGHT` prefix records which side carries the duplicate-eliminated
data, via the logical join's `delim_flipped` flag — Ch. 6 §6.4 names this
without pinning down the exact direction for a query this book never
specifically drove `EXPLAIN` against), and builds a `PhysicalHashAggregate`
(the join's own `distinct` field) grouping on `duplicate_eliminated_columns =
[c.custkey]`, wired to feed the `PhysicalColumnDataScan` (`DELIM_SCAN`) that
descends from `LogicalDelimGet` in the join's now-elaborated tree. The outer
join is a plain `PhysicalHashJoin`, `LEFT` type, `customer` on the (preserved)
left, the grouped-`orders` aggregate on the right. `COUNT(*)` grouped by an
integer key, `custkey`, is structurally the same shape §19.2.4/§19.2.7
resolved for `SUM(l.extendedprice)` grouped by `custkey` — whether it
likewise becomes `PERFECT_HASH_GROUP_BY` rather than plain `HASH_GROUP_BY`
depends on whether `STATISTICS_PROPAGATION` proves an equally tight range for
`custkey` here, which this chapter cannot confirm without a live run of this
specific query (unlike Query A, no earlier chapter captured one for Query B),
so it is left as a genuinely open, correctly-flagged uncertainty rather than
assumed either way.

```
┌───────────────────────────────────────────────────────────────────┐
│ PhysicalResultCollector                                            │
└──────────────────────────────┬────────────────────────────────────┘
┌───────────────────────────────▼───────────────────────────────────┐
│ PhysicalProjection      c.name, n_orders                            │
└──────────────────────────────┬────────────────────────────────────┘
┌───────────────────────────────▼───────────────────────────────────┐
│ PhysicalHashJoin (LEFT)   condition: c.custkey = d                  │
│   probe (preserved): customer   build: grouped-orders aggregate     │
└──────────────┬─────────────────────────────────┬───────────────────┘
┌──────────────▼───────────────┐    ┌─────────────▼──────────────────────┐
│ PhysicalTableScan (customer)  │    │ PhysicalHashAggregate                │
│   projects: [custkey, name]   │    │   GROUP BY d; COUNT(*)                │
│                                │    └─────────────┬──────────────────────┘
│                                │    ┌─────────────▼──────────────────────┐
│                                │    │ PhysicalLeftDelimJoin (or Right —    │
│                                │    │  §6.4's delim_flipped, unconfirmed   │
│                                │    │  for this specific query)             │
│                                │    │   condition: o.custkey = d            │
│                                │    └───────┬───────────────────┬─────────┘
│                                │  ┌──────────▼─────────┐  ┌──────▼──────┐
│                                │  │ PhysicalHashAggregate │  │ Physical    │
│                                │  │  "distinct": GROUP BY  │  │ TableScan   │
│                                │  │  c.custkey (computes D)│  │ (orders)    │
│                                │  │  fed from customer's   │  │ projects:   │
│                                │  │  own scan/pipeline      │  │  [custkey]  │
│                                │  └────────────────────────┘  └─────────────┘
└────────────────────────────────┘
```

### 19.3.6 Pipelines and execution (reasoned forward from Ch. 11's Sink/Source rules, Ch. 10/12/13's general vector/expression mechanics)

Applying Chapter 11's rules mechanically: every binary hash join is a sink
against its build child and an operator against its probe child (§11.1,
§11.3.4); every `PhysicalHashAggregate`'s `Finalize` gates the next pipeline,
exactly as it did for Query A's `GROUP BY o.custkey` in §19.2.5. That gives,
at minimum, a chain shaped like: (1) scan `customer`, feed the "distinct"
aggregate that computes D → (2) scan `orders`, probe/build the delim join
against D, feed the `COUNT(*)` aggregate grouped by `d` → (3) read the
finished per-`custkey` counts, probe/build the outer `LEFT` join against the
`customer` scan → (4) project and collect. This chapter states plainly what
it cannot pin down at this depth: **the exact wiring by which the "distinct"
D-computing pipeline shares (or independently re-derives) `customer`'s rows
relative to the outer join's own `customer` scan is a detail Chapter 6 §6.4
established exists (via `plan_delim_join.cpp`'s `distinct` field and the
`PhysicalColumnDataScan`/`DELIM_SCAN` wiring) but did not diagram at
pipeline granularity, and no earlier chapter drew a pipeline diagram for any
delim-join-bearing query at all.** The four-stage shape above is the most
direct application of Chapter 11's general rules to Query B's tree; it should
be read with the same "reasoned reconstruction, not a transcript" caveat
Chapter 11 attached to its own Query A diagram (§19.2.5), one level more
cautiously.

At the vector/expression level, nothing here needs new machinery beyond what
Chapters 10, 12, and 13 already established for Query A: `c.custkey`/
`o.custkey` are ordinary `BoundReferenceExpression`s flowing through
`DataChunk`s exactly as `orderkey` did in §19.2.6; the equality conditions on
both joins dispatch through the same `BOUND_FUNCTION`/`Equals` path Chapter
12 §12.6 walked for `int64` equality; and `COUNT(*)` sits outside
`ExpressionExecutor::Execute`'s switch for the identical reason `SUM` did in
§19.2.6 — `BOUND_AGGREGATE` isn't a case that switch handles, because
per-group accumulator state has to outlive any single chunk. None of it is
compiled per-query, per Chapter 13's general claim, restated rather than
re-proven here.

### 19.3.7 A flagged aside: does the delim join survive at all?

With default settings the answer is simply no: the planner has already
replaced it with joins over CTEs before the optimizer starts (see
§19.3.3). The rest of this section asks the narrower question for
`SET delim_join_as_cte = false`. The CTE step's own eliminators target the
same redundancy for the default form (Chapter 6 §6.4.1, Chapter 7 part 13
§13.3.6).

Chapter 6 §6.6 documents, with a direct code citation
(`deliminator.cpp:230-238`, `RemoveJoinWithDelimGet`), a "decoupling"
optimization: if the RHS of a `LOGICAL_DELIM_JOIN` already produces, for
free, a column equal to the correlation parameter wherever it matters, the
join against the materialized D can be deleted outright, compensated by one
`IS NOT NULL` filter. Query B's inner delim join condition is `o.custkey =
d` — after that condition holds, `o.custkey` *is* `d`, by definition, for
every surviving row. This is exactly the shape `RemoveJoinWithDelimGet`
targets. If it fires, the entire "distinct" `PhysicalHashAggregate` and the
delim join disappear, per §6.6's second, physical-level citation
(`plan_delim_join.cpp:36-41`: "no duplicate eliminated scans in the delim
side... just push the normal join"), and Query B's inner subtree collapses
to something much plainer:

```
Γ[n_orders: count(*)]/(o.custkey) (σ[o.custkey IS NOT NULL](orders))
```
— i.e., simply `SELECT custkey, COUNT(*) FROM orders WHERE custkey IS NOT
NULL GROUP BY custkey`, `LEFT JOIN`ed against `customer` directly on
`c.custkey = o.custkey`, with no delim machinery anywhere in the final
physical tree at all.

This chapter flags this explicitly as the *more likely* real outcome rather
than the conservative tree drawn in §19.3.5, precisely because Chapter 6
proves the mechanism exists and cites the exact condition it targets — but
whether `RemoveJoinWithDelimGet`'s specific redundancy check recognizes
*this particular* expression shape (an equality condition on the delim join
itself, versus the more elaborate nested-subquery shapes Chapter 6's own
prose examples describe) is not something any earlier chapter confirmed
against a live plan, and this chapter declines to assert it as settled
without one. Both trees are offered because the honest state of knowledge,
assembled from what's actually been verified so far, is "very plausibly this
simpler shape, unconfirmed" — exactly the kind of stated uncertainty the
front matter commits to rather than a papered-over guess.

## 19.4 A third query: a primary-key point lookup through the ART index

```sql
SELECT * FROM customer WHERE custkey = 12345;
```

Neither Query A nor Query B ever touches an ART index — §16.10 states this
outright for Query A ("never uses an ART index anywhere in its plan; it
relies entirely on zonemaps and vectorized sequential scanning"), and Query B
has no equality predicate against an indexed column anywhere in its own
plan either. This query is chosen specifically because Chapter 16 §16.8
already carries a fully worked, byte-by-byte trace of almost this exact
query, assuming `custkey` is declared `PRIMARY KEY` — that assumption is kept
here too, so this section can reuse §16.8 directly rather than inventing a
parallel version that might drift from it.

### 19.4.1 Parse, bind, logical plan, and filter pushdown (reasoned forward from Ch. 3/Ch. 4/Ch. 7's general rules)

No earlier chapter parsed or bound this specific query, so this stage is
reasoned forward rather than pulled from a citation — but nothing about it
needs new machinery. `SELECT * FROM customer WHERE custkey = 12345` matches
`select.gram`'s ordinary `SimpleSelect` shape exactly the way Query A's outer
query did (§19.2.1): one `BaseTableRef("customer")`, a `where_clause` of
`ComparisonExpression(COMPARE_EQUAL, ColumnRefExpression(["custkey"]),
ConstantExpression(12345))`, and a `select_list` of a single `StarExpression`.
Binding resolves `customer` against the catalog, assigns it a `TableIndex`,
constructs a `LogicalGet`, and binds `custkey = 12345` against it exactly the
way Query A's `l.shipdate < DATE '1998-01-01'` was bound in §19.2.2 — a
`BoundComparisonExpression` wrapped in a `LogicalFilter`. The unoptimized
tree is the simplest shape this book has traced anywhere:

```
LogicalProjection (*)
  └─ LogicalFilter (custkey = 12345)
       └─ LogicalGet (customer)
```

`FILTER_PUSHDOWN` (Ch. 7 §7.4 pass #6) does exactly what it did for
`lineitem`'s `shipdate` predicate in §19.2.3: since the predicate references
only `customer`'s own column, it moves directly into `customer`'s
`LogicalGet::table_filters`, leaving:

```
LogicalProjection (*)
  └─ LogicalGet (customer — table_filters: custkey = 12345)
```

### 19.4.2 Physical plan and the ART lookup itself (Ch. 16 §16.8 — pulled directly)

`LogicalGet(customer)` becomes a `PhysicalTableScan` exactly as `orders` and
`lineitem` did in §19.2.4 — there is no separate `PhysicalIndexScan` operator
class in this checkout. The index consultation happens *inside* the same
scan machinery, at table-function-init time: `table_scan.cpp`'s `TryScanIndex`
(§16.10's own citation, `table_scan.cpp:926-940`) inspects whether the
pushed-down `TableFilter` is a single-column equality/range predicate against
a column that has an ART, and — because `custkey`'s `PRIMARY KEY` constraint
gave it an implicit, unique ART — it is. The scan then bypasses sequential
scanning and zonemap-based row-group skipping entirely in favor of an actual
tree descent:

**Step 1 — encode the key.** `12345` as a big-endian, sign-bit-flipped
32-bit integer (Ch. 16 §16.3, `Radix::EncodeSigned`) is the 4-byte sequence
`[0x80, 0x00, 0x30, 0x39]`.

**Step 2 — descend, one byte per level.** Byte `0x80` resolves at the root's
node (a `Node4`/`Node16`/etc. depending on how many live children exist at
that level — Ch. 16 §16.5's four adaptive node types); a path-compressed
`Prefix` node may absorb a shared byte like `0x00` with a single comparison
rather than a child-array lookup (§16.7's lazy expansion and path
compression); the final byte, `0x39`, lands on a `LEAF_INLINED` node —
because `custkey` is declared `UNIQUE`/`PRIMARY KEY`, §16.6's rule applies:
no separate leaf allocation exists at all, and the matching row ID is
unpacked directly out of the `NodePtr`'s own bit pattern via `GetRowId()`.

**Step 3 — row ID to row.** The ART's job ends at a row ID; turning it into
an actual `(custkey, name, address, ...)` tuple is the storage layer's job
(Ch. 17 §17.2.1's row-group/column-chunk model) — locate which row group
holds that row ID, then fetch the requested columns' values at the right
offset within it. Because this query's `SELECT *` needs every column,
projection pushdown (which pruned Query A's scans down to two or three
columns each, §19.2.3) has nothing to prune here — every column gets fetched
for this one row.

The physical tree is the shallowest in this chapter:

```
┌─────────────────────────────┐
│ PhysicalResultCollector      │
└──────────────┬───────────────┘
┌───────────────▼───────────────┐
│ PhysicalProjection (*)         │
└──────────────┬────────────────┘
┌───────────────▼───────────────┐
│ PhysicalTableScan (customer)   │  table_filters: custkey = 12345
│   index lookup via ART          │  (TryScanIndex — bypasses zonemap
│   (implicit PK index)           │   skipping entirely for this scan)
└────────────────────────────────┘
```

### 19.4.3 Pipelines and vectors (reasoned forward from Ch. 11's general Sink/Source rules, Ch. 10's `DataChunk` model)

This is the simplest pipeline shape this chapter has assembled: one sink
(`PhysicalResultCollector`), one source (`PhysicalTableScan`), no
intermediate operators at all — there is no join, no aggregate, no sort to
break the tree into more than a single pipeline, unlike either Query A's four
or Query B's several. The `DataChunk` this scan produces holds, at most, one
row — because `custkey` is `PRIMARY KEY`, exactly zero or one row can ever
match — so the whole apparatus of `SelectionVector`s slicing thousands of
rows down to hundreds (§19.2.6's `lineitem` trace) has nothing to do here:
the chunk that reaches `PhysicalResultCollector` is either empty or a single
fully-materialized row, no vectorized batching payoff available or needed at
this cardinality. This is the concrete instance of the general point §16.1's
selectivity math and §16.10's OLAP-vs-OLTP framing both make in the
abstract: an ART lookup is worth using exactly when the alternative is
touching far more rows than the answer needs, and a query that returns at
most one row out of a customer table is the maximally favorable case for
that trade.

## 19.5 Reading map

For a reader who arrived here first and wants to go deeper on any single leg
of the journeys just assembled, this is the one best chapter per stage:

| Stage | Best chapter | What it proves that this chapter only cited |
|---|---|---|
| Parsing (PEG grammar, tokenizer, AST shape) | Ch. 3 | The grammar rules and transformer pattern behind every `SelectNode`/`SubqueryExpression` used above |
| Binding and scoping (`BindContext`, correlation detection) | Ch. 4 | `ColumnBinding`, `BindInEnclosingScope`, and exactly how `Depth() > 0` gets discovered |
| Logical operator vocabulary | Ch. 5 | Every `LogicalOperator` subclass's fields, `ColumnBindingResolver`'s full mechanics |
| Subquery decorrelation | Ch. 6 | `FlattenDependentJoins`'s full rewrite-rule derivation and the `deliminator.cpp`/`plan_delim_join.cpp` code this chapter only cited |
| The fixed optimizer pipeline | Ch. 7 | The authoritative, line-cited ~45-pass ordering; the passes' individual mechanics |
| Join ordering, build/probe side, join algorithm choice | Ch. 8 | DPhyp, the join hypergraph, the full `PlanComparisonJoin` decision tree |
| Physical operator selection | Ch. 9 | `ColumnBindingResolver`'s coordinate-system change and every `PhysicalOperator` subclass |
| Vectors, `DataChunk`, `SelectionVector`, dictionary encoding | Ch. 10 | Why filtering composes cheaply across operator boundaries |
| Pipelines, `MetaPipeline`, Sink/Source/Operator, the scheduler | Ch. 11 | How a physical tree becomes concurrently-runnable tasks |
| Expression evaluation, `ExpressionExecutor` dispatch | Ch. 12 | The exact template-specialization/dispatch chain, `AdaptiveFilter` |
| Interpretation vs. compilation | Ch. 13 | The exhaustive case that DuckDB is not a JIT-compiling engine |
| Memory, hash-based grouped aggregation | Ch. 14 | Sink → Combine → Finalize → GetData for `GROUP BY`, and `PERFECT_HASH_GROUP_BY`'s exact criteria |
| Sorting and Top-N | Ch. 15 | `CreateSortKeyHelpers`, the full unbounded-sort machinery `PhysicalTopN` deliberately avoids |
| The ART index | Ch. 16 | Key encoding, the four adaptive node types, path compression, and what ART deliberately can't do |
| Storage (row groups, compression, zonemaps) | Ch. 17 | How a row ID becomes a row, and how zonemaps make Query A's filter pushdown pay off |
| Transactions and MVCC | Ch. 18 | What none of this chapter's three read-only queries needed, but any concurrent writer would |
| Methodology: reading real `EXPLAIN`/`EXPLAIN ANALYZE` output | Ch. 2 §2.5–2.6 | How to get a live check on any claim this chapter — or any chapter — makes about a specific plan |
