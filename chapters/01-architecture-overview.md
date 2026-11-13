# Chapter 1 — Architecture Overview

## 1.1 The pipeline, precisely

The seven-box diagram from the front matter is the right mental model, but it
hides two things DuckDB does differently from the generic textbook version, and
both are worth knowing before you read anything else in this book:

```
   SQL text
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ PARSER                                                       │
│  src/parser/peg/  (native PEG parser, v2.0.0 and later)      │
│  produces: vector<unique_ptr<SQLStatement>>                  │
└──────────────────────────┬───────────────────────────────────┘
                            │  SQLStatement (untyped AST:
                            │  ColumnRefExpression, BaseTableRef, ...)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ BINDER  (also builds the initial Logical Plan — see 1.2)     │
│  src/planner/binder.cpp, src/planner/binder/                 │
│  produces: BoundStatement { plan, types, names }              │
└──────────────────────────┬───────────────────────────────────┘
                            │  unique_ptr<LogicalOperator> tree
                            │  (unoptimized — see Chapter 5)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ OPTIMIZER                                                     │
│  src/optimizer/optimizer.cpp                                  │
│  ~45 fixed-order passes; rewrites the tree in place            │
└──────────────────────────┬───────────────────────────────────┘
                            │  unique_ptr<LogicalOperator> tree
                            │  (optimized, same node vocabulary)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ PHYSICAL PLAN GENERATOR                                       │
│  src/execution/physical_plan_generator.cpp                    │
│  produces: PhysicalOperator tree, owned by a PhysicalPlan arena │
└──────────────────────────┬───────────────────────────────────┘
                            │  PhysicalOperator tree
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ EXECUTION ENGINE                                               │
│  src/parallel/  (Pipeline, MetaPipeline, TaskScheduler)        │
│  src/execution/expression_executor*  (per-vector interpreter)  │
│  a vectorized, push-based, multi-threaded interpreter —        │
│  NOT a query compiler. See Chapter 13.                         │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
                      result rows
```

**Difference #1 — Binder and Logical Plan are the same step, not two.** In the
textbook diagram, "Binder / Planner" and "Logical Plan" are drawn as separate
boxes connected by an arrow, which suggests the binder produces some intermediate,
not-yet-a-plan representation that a separate planning step then turns into a
logical plan. DuckDB doesn't have that intermediate representation. The `Binder`
class's own methods — specifically the family of files named `plan_*.cpp` inside
`src/planner/binder/query_node/` and `src/planner/binder/tableref/` — directly
construct `LogicalGet`, `LogicalProjection`, `LogicalComparisonJoin`, and so on
*as a byproduct of binding a name*. There is no separate "bound but unplanned"
tree. The struct the binder actually hands off,
[`BoundStatement`](../research/duckdb-source-architecture-recon.md) (§2e of the
research notes; canonical source at
`src/include/duckdb/planner/bound_statement.hpp`), already contains a fully
formed `unique_ptr<LogicalOperator> plan`:

```cpp
// src/include/duckdb/planner/bound_statement.hpp
struct BoundStatement {
    unique_ptr<LogicalOperator> plan;
    vector<LogicalType> types;
    vector<Identifier> names;
    ExtraBoundInfo extra_info;
};
```

So the diagram's "Binder / Planner" and "Logical Plan" boxes are, in DuckDB, one
piece of code producing one struct. Chapter 4 covers the binder; Chapter 5 covers
the tree shape it produces; they are two chapters about two aspects of the same
step, not two sequential steps.

**Difference #2 — most subquery decorrelation happens *before* the optimizer, not
inside it.** A correlated subquery — anything shaped like `WHERE x IN (SELECT ...
correlated_column ...)` or a correlated scalar subquery in the `SELECT` list —
cannot, in DuckDB, be executed by any physical operator directly. There is no
"correlated nested-loop" physical operator waiting at the bottom of the pipeline
to re-evaluate a subquery once per outer row. Instead, the binder represents a
correlated subquery with a special logical node, `LogicalDependentJoin`. Right
after binding, before the optimizer's first pass runs, the planner
(`Planner::CreatePlan`) rewrites it away using an algorithm called
`FlattenDependentJoins` (`src/planner/subquery/flatten_dependent_join.cpp`).
By the time the optimizer's first pass sees the plan, every
`LogicalDependentJoin` is already gone, replaced by an ordinary join tree.
Flattening first builds that tree around a *delim join*. In the code, a delim
join is a `LogicalComparisonJoin` whose operator type is
`LogicalOperatorType::LOGICAL_DELIM_JOIN`; there is no separate
`LogicalDelimJoin` class. With
default settings, one more planner step then turns each delim join into
plain joins over materialized CTEs (Chapter 6 §6.4.1), so the optimizer
never sees a delim join at all. `FlattenDependentJoins::DecorrelateIndependent`
calls `DelimJoinCTERewriter::Rewrite` right after flattening
(`flatten_dependent_join.cpp:281-292`). The setting that controls this,
`delim_join_as_cte`, defaults to `true` and is marked deprecated. The
optimizer's `deliminator.cpp` and
`unnest_rewriter.cpp` passes, and a matching clean-up inside the CTE step,
then get a chance to simplify it further.

Chapter 6 is dedicated to this, because it is one of
DuckDB's most academically well-known pieces of engineering (it directly
implements Neumann and Kemper's "Unnesting Arbitrary Queries"
[12]) and because
getting the planner/optimizer split right matters if you ever go looking for
"where does DuckDB decorrelate subqueries" in the source — the honest answer is
"mostly in `src/planner/`, not `src/optimizer/`," which surprises people who
assume optimizer work only happens in the box literally named `optimizer/`.

## 1.2 The bigger picture: where this pipeline sits inside the whole engine

The seven-stage pipeline is what happens to *one query*. It sits inside a larger
set of long-lived components that exist independent of any single query:

```
┌───────────────────────────────────────────────────────────────────────┐
│                              DuckDB (Process)                          │
│                                                                          │
│  ┌────────────────┐   ┌────────────────┐   ┌───────────────────────┐  │
│  │  Catalog        │   │  ClientContext  │   │  TaskScheduler         │  │
│  │  (databases,    │   │  (per-          │   │  (shared thread pool,  │  │
│  │  schemas,       │   │  connection     │   │  used by ALL           │  │
│  │  tables,        │   │  state: active  │   │  concurrent queries)   │  │
│  │  functions,     │   │  transaction,   │   └───────────┬───────────┘  │
│  │  indexes)       │   │  settings,      │                │              │
│  └───────┬────────┘   │  prepared       │                │              │
│          │             │  statements)    │                │              │
│          │             └───────┬────────┘                │              │
│          │                     │  1 per query: runs the    │              │
│          │                     │  7-stage pipeline above    │              │
│          │                     ▼                            ▼              │
│          │           ┌──────────────────────────────────────────────┐    │
│          │           │           Executor (1 per running query)       │    │
│          │           │  owns this query's MetaPipelines, schedules    │    │
│          │           │  its PipelineTasks onto the shared scheduler   │    │
│          │           └───────────────────┬──────────────────────────┘    │
│          │                               │                                 │
│          ▼                               ▼                                 │
│  ┌────────────────┐            ┌───────────────────────┐                 │
│  │  BufferManager  │◄───────────┤  DataTable / row groups │                 │
│  │  (page cache,   │            │  (columnar storage,     │                 │
│  │  eviction,      │            │  compression, zonemaps) │                 │
│  │  spilling)      │            └───────────┬────────────┘                 │
│  └───────┬────────┘                        │                              │
│          │                                  │                              │
│          ▼                                  ▼                              │
│  ┌────────────────┐            ┌───────────────────────┐                 │
│  │ BlockManager    │            │ TransactionManager /    │                 │
│  │ (on-disk        │            │ UndoBuffer (MVCC)       │                 │
│  │ .duckdb file,   │            └───────────────────────┘                 │
│  │ WAL)            │                                                       │
│  └────────────────┘                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

A few things worth internalizing from this picture before we go stage by stage:

- **The `Catalog` is consulted continuously, not just once.** The parser doesn't
  touch it at all (parsing is purely syntactic — it doesn't know whether `orders`
  is a real table). The binder is the first stage that looks anything up in the
  catalog (resolving `orders` to an actual `TableCatalogEntry`, resolving a
  function name to a concrete overload). The optimizer's statistics-propagation
  pass reads column statistics that live alongside catalog metadata. The physical
  plan generator reads index metadata off the catalog when deciding whether a
  filter can use an ART index. None of this is a separate "catalog stage" in the
  seven-box diagram — it's a service every stage calls into as needed.
- **One `TaskScheduler`, many queries.** The thread pool that actually runs
  vectorized execution (Chapter 11) is process-wide (well: per-`DuckDB` instance),
  shared by every concurrent query on every concurrent connection. A single
  query's `Executor` breaks its physical plan into `Pipeline`s and submits
  `PipelineTask`s to this shared scheduler; it does not own dedicated threads.
- **Storage and transactions are the foundation everything else stands on**,
  but they are also the part of the system a single query interacts with the
  least directly — a `PhysicalTableScan` operator asks the `DataTable` for
  chunks and gets them, without needing to know anything about block managers,
  compression schemes, or undo buffers. Chapters 14–18 cover this foundation;
  they're placed *after* the query-processing chapters specifically because you
  need the vocabulary from execution (chunks, vectors, pipelines) before storage
  internals will make sense, even though storage is, in a layering sense, "below"
  execution.

## 1.3 One query, all seven stages, at a glance

This section is deliberately shallow — a preview, not a proof. Every claim here is
expanded with full evidence in its own chapter. The point is to let you see the
shape of the whole journey once before diving into any one leg of it.

Take Query A from the front matter:

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

**Stage 1 — Parser.** The PEG parser (Chapter 3; DuckDB v1.5.x and earlier use
the older `libpg_query` parser instead, described in Appendix A, but it builds
the same kind of tree) tokenizes the query text, then
matches it against the `select.gram` grammar rule (with `expression.gram` handling
`SUM(l.extendedprice)`, the comparison in `WHERE`, and the join condition). The
output is one `SelectStatement`, whose `node` is a `SelectNode` holding:
a `from_table` of type `JoinRef` (itself holding two `BaseTableRef`s, `orders
AS o` and `lineitem AS l`, plus the join condition as a `ComparisonExpression`), a
`where_clause` (`ComparisonExpression` comparing a `ColumnRefExpression` against a
constant), a `select_list` (a `ColumnRefExpression` for `o.custkey` and a
`FunctionExpression` for `sum(l.extendedprice)`), a `groups` list, and result
modifiers for `ORDER BY` and `LIMIT`. Nothing here has been checked against a
catalog yet — the parser doesn't know or care whether `orders` exists, or whether
`extendedprice` is numeric.

**Stage 2 — Binder.** The binder (Chapter 4) resolves `orders`/`lineitem` against
the catalog, assigns each a `TableIndex`, and registers their columns in the
`BindContext`. `o.custkey` becomes a `BoundColumnRefExpression` carrying a
`ColumnBinding{table_index, column_index}` — from this point on, nothing in the
plan refers to columns by name again. `sum(l.extendedprice)` is bound against the
`sum` aggregate function's catalog entry. Simultaneously (Difference #1 above),
the binder constructs the logical plan directly: a `LogicalGet` for each table, a
`LogicalComparisonJoin` for the `JOIN ... ON`, a `LogicalFilter` for the `WHERE`,
a `LogicalAggregate` for the `GROUP BY`/`SUM`, and a `LogicalOrder` for the
`ORDER BY`, with the `LIMIT` folded in as a `LogicalLimit`. The unoptimized tree
looks like:

```
LogicalLimit (10)
  └─ LogicalOrder (total DESC)
       └─ LogicalAggregate (GROUP BY o.custkey; SUM(l.extendedprice))
            └─ LogicalFilter (l.shipdate < DATE '1998-01-01')
                 └─ LogicalComparisonJoin (o.orderkey = l.orderkey)
                      ├─ LogicalGet (orders)
                      └─ LogicalGet (lineitem)
```

(This particular query has no correlated subquery, so `LogicalDependentJoin` and
`FlattenDependentJoins` — Chapter 6 — never come into play. Query B, in Chapter 6,
is where that machinery is demonstrated.)

**Stage 3 — Optimizer.** The optimizer (Chapter 7) runs its fixed ~45-pass
pipeline over this tree. For this query, the passes that actually change anything
include: `filter_pushdown` (pushes `l.shipdate < ...` down so it's evaluated as
early as possible — ideally right at `lineitem`'s scan, as a predicate the storage
layer's zonemaps can use to skip row groups, rather than after the join);
`join_order` (with only one join here, this pass has nothing to reorder, but it
still runs — Chapter 8 covers what it does when there are more tables, and
covers the detail, easy to guess wrong, that join ordering uses its own
self-contained cardinality estimates rather than waiting on the general
`statistics_propagation` pass, which actually runs *later* in the fixed
pipeline — see Chapter 7 for the exact order); `statistics_propagation`
(propagates min/max/cardinality/nullability estimates more broadly through the
tree, catching further constant-folding and pruning opportunities); and
`top_n` (fuses the `LogicalOrder` + `LogicalLimit` pair into a single
`LogicalTopN`, because computing a full sort of every group just to keep the top
10 is wasteful — a bounded top-k structure is cheaper).

Two other passes are worth naming here. `REDUNDANT_ORDER_KEYS` drops
`ORDER BY` keys that come after a prefix of keys that is already unique
(`src/optimizer/remove_redundant_order_keys.cpp`). For example, after
`GROUP BY a`, the key `a` is unique, so in `ORDER BY a, b` the `b` can never
break a tie. `SELF_JOIN_AGGREGATE` rewrites a self-join of one table on equal
keys and one inequality into `GROUP BY keys HAVING min(x) <> max(x)` over a
single scan, when the parent reads only the keys and ignores duplicates
(`src/include/duckdb/optimizer/self_join_aggregate.hpp:15-17`).
`src/optimizer/optimizer.cpp` has 48 `RunOptimizer(...)` call sites.
Chapter 7 §7.4 numbers 45 of them as the main pipeline (some
passes run twice). It leaves out `REMOTE_PUSHDOWN` and the two extension
hooks. `SELF_JOIN_AGGREGATE` is pass #3 and `REDUNDANT_ORDER_KEYS` is
pass #21. Neither pass changes Query A: its `ORDER BY` has a single key,
and it has no self-join.

After optimization:

```
LogicalTopN (ORDER BY total DESC LIMIT 10)
  └─ LogicalAggregate (GROUP BY o.custkey; SUM(l.extendedprice))
       └─ LogicalComparisonJoin (o.orderkey = l.orderkey)
            ├─ LogicalGet (orders)
            └─ LogicalGet (lineitem, filter pushed into scan: shipdate < ...)
```

**Stage 4 — Physical Plan Generator.** `PhysicalPlanGenerator` (Chapter 9) walks
this tree and, for every `LogicalOperator`, picks a concrete `PhysicalOperator`.
`LogicalComparisonJoin` with an equality predicate becomes `PhysicalHashJoin`
(Chapter 8 covers when it would instead become a merge join or `IEJoin`).
`LogicalAggregate` with a `GROUP BY` becomes `PhysicalHashAggregate` in the
general case (Chapter 14 covers the full two-phase parallel hash-aggregation
algorithm this runs, and the narrower conditions — such as a scan that already
reports its data partitioned by the grouping column — under which
`PhysicalPartitionedAggregate` or `PhysicalPerfectHashAggregate` get chosen
instead).
`LogicalTopN` becomes `PhysicalTopN` directly (Chapter 9). The two `LogicalGet`s
become two `PhysicalTableScan`s. A `PhysicalResultCollector` is added at the root
to gather the final output rows:

```
PhysicalResultCollector
  └─ PhysicalTopN
       └─ PhysicalHashAggregate
            └─ PhysicalHashJoin  (build: orders, probe: lineitem — or the
                                  reverse; Chapter 8 covers how build/probe
                                  side is chosen)
                 ├─ PhysicalTableScan (orders)
                 └─ PhysicalTableScan (lineitem, pushed-down filter applied
                                       during the scan itself)
```

**Stage 5 — Execution.** This is where the biggest conceptual jump happens, and
it's worth being explicit about it here even before Chapters 10–13 build it up in
full. The physical tree above is *not* compiled into a single function and run.
Instead:

- The tree is decomposed into **pipelines** (Chapter 11) — the hash join's build
  side (scan `orders` → build the hash table, a *sink*) is one pipeline; the probe
  side (scan `lineitem` → probe the hash table → feed the aggregate's sink) is
  another, dependent on the first having finished.
- Each pipeline runs by pulling **`DataChunk`s** (Chapter 10) — batches of up to
  2048 rows across all needed columns — from its source, pushing each chunk
  through a chain of operators, and finally handing it to a sink.
- Every operator that needs to evaluate a SQL expression (the `WHERE` predicate,
  `l.orderkey` in the join condition, `sum(l.extendedprice)`) does so through the
  `ExpressionExecutor` (Chapter 12), which **interprets** the bound expression
  tree once per chunk — it does not generate machine code, LLVM IR, or C++ source
  for this specific query. Chapter 13 is dedicated to proving this claim
  exhaustively and explaining why DuckDB's designers chose this over the
  alternative (compiling each query, à la HyPer/Umbra
[13], [14]).
- Multiple `PipelineTask`s for independent chunks of `orders` and `lineitem` can
  run on different worker threads simultaneously, coordinated by the shared
  `TaskScheduler` — this is where DuckDB gets its parallelism, without any
  query-specific compilation.

The result of all of this: 10 rows, each holding a `custkey` and a `total`,
streamed back to whatever client issued the query.

## 1.4 The one-sentence version of DuckDB's design philosophy

Every chapter after this one is, in some sense, evidence for a single design
philosophy, which is worth stating up front so you can recognize it recurring:
**push as much decision-making as possible to "compile time" of the *database
process*, not to "compile time" of the *query*, and get speed from batching,
compact in-memory representations, and cache-conscious algorithms rather than
from generating and JIT-compiling new machine code per query.** The C++ template
specialization you'll see all over the expression executor (Chapter 12) — a
different, hyper-specialized inner loop for every combination of operator, type,
and vector representation — looks, from a distance, like the same idea as
per-query code generation. It is not: those specialized loops are all compiled
*once*, when DuckDB itself is built, and the query merely selects among them at
runtime with a `switch`. Chapter 13 draws this distinction out in full, because it
is the single most important architectural fact to get right if you're trying to
understand why DuckDB is fast.
