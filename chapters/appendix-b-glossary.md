# Appendix B — Glossary

This appendix collects, in one alphabetized list, every acronym, class name, and
term of art used across the book. Entries are deliberately terse — this is a
reference, not a restatement of the chapters it points back to. Where a term
gets its full treatment somewhere else, the entry says where to go.

A note on how "class name" entries are written: DuckDB's C++ vocabulary is not
decorative. A `LogicalGet` and a `PhysicalTableScan` are different things
representing different moments in the same query's life, and this glossary
tries to say precisely which moment each class belongs to, rather than
gesturing at "the scan thing."

---

### Academic venue abbreviations (CIDR, ICDE, SIGMOD, VLDB, BTW)

Conference/journal abbreviations this book cites when attributing an
algorithm to its source paper. **VLDB** — Very Large Data Bases (both the
conference and, for the version cited in Chapter 13, its journal, PVLDB)
[13]. **SIGMOD** — ACM's Special Interest Group on
Management of Data conference, where Moerkotte and Neumann published the DPhyp
join-enumeration algorithm in "Dynamic Programming Strikes Back" (Chapter 8)
[20]. **ICDE** — IEEE International Conference on Data
Engineering, where Leis, Kemper and Neumann published the Adaptive Radix Tree
(Chapter 16) [33]. **CIDR** — Conference on Innovative Data Systems
Research, where MonetDB/X100's vectorized-execution paper appeared
(Chapter 13) [27]. **BTW** — *Datenbanksysteme für Business,
Technologie und Web*, the German-language database conference where Neumann
and Kemper published "Unnesting Arbitrary Queries" (Chapter 6)
[12], the paper `FlattenDependentJoins` directly
implements.

### AdaptiveFilter

A runtime mechanism (`src/execution/adaptive_filter.cpp`) that reorders the
conjuncts of a compound `WHERE` predicate based on which ones are observed to
reject the most rows, cheapest-and-most-selective first — without generating
new code, only choosing an evaluation order at runtime. See Chapter 12 §12.12.

### ALP / ALPRD (Adaptive Lossless floating-Point compression)

DuckDB's default compression scheme for `DOUBLE`/`FLOAT` columns
(`alp/`, `alprd.cpp`), superseding Patas/Chimp for the common case
[36]. See Chapter 17 §17.4.7.

### ART (Adaptive Radix Tree)

A span-8, adaptively-node-sized radix tree used for DuckDB's secondary
indexes (`PRIMARY KEY`/`UNIQUE`/`FOREIGN KEY` enforcement and explicit
`CREATE INDEX`). Originates from Leis, Kemper and Neumann's 2013 ICDE paper
[33]; DuckDB's implementation lives under `src/execution/index/art/`. Unlike a
zonemap, an ART lookup finds exact row IDs in time proportional to key
bit-length, not table size — but it is not automatic the way zonemaps are;
it exists only when something asks for it. Full treatment, including key
encoding and the four adaptive node types, in Chapter 16.


### Async results

How DuckDB v2.0 hands a query's rows to the client. Worker threads build the
result into a buffer, and the client reads from it, with back-pressure. A
`ResultFormat` (see below) decides the shape of the output. Three visible
effects: the planner does not force non-`SELECT` statements to materialize (an `INSERT ... RETURNING` can stream); closing an unfinished
result aborts its statement and, in autocommit, rolls it back; and a
connection that still holds an unfinished result refuses new statements with
`RESOURCE_IN_USE`. Main PRs: #25727, #25801, #26314, #26747, #26880.

---

### Bare-label keyword

A keyword that may be used as a column alias without `AS`, listed in
`src/parser/peg/grammar/keywords/bare_label_keyword.list` (the same idea as
in Postgres). A keyword not on the list needs `AS`. For example,
`SELECT d + 1 day` is a syntax error rather than an alias named `day`.
See Chapter 3.

### BindContext

The per-scope structure the `Binder` populates and consults to resolve a
bare name (`o.custkey`, `orders`) to something concrete — a table, a column, a
CTE. It is also the mechanism behind "did you mean" suggestions on unresolved
names. See Chapter 4 §4.3.

### Binder

The class (`src/planner/binder.cpp` and `src/planner/binder/`) that performs
name resolution *and*, as a byproduct of resolving names, directly constructs
the initial `LogicalOperator` tree — there is no separate "bound but
unplanned" intermediate representation in DuckDB, unlike the generic textbook
pipeline. One `Binder` exists per lexical scope (a subquery gets its own,
chained to its parent). See Chapter 4.

### Bitpacking

DuckDB's default compression scheme for integer columns: within a block,
subtract a common minimum, then pack each residual using only as many bits as
the largest residual needs. Implements frame-of-reference encoding — see
**FOR** below for the disambiguation of that term. See Chapter 17 §17.4.2.

### BlockManager

The abstract interface for reading, writing, and allocating fixed-size disk
blocks (256 KiB by default) that make up a `.duckdb` file. See Chapter 17
§17.1.1.

### BoundStatement

The struct a `Binder` ultimately produces: `{ plan, types, names, extra_info
}`, where `plan` is already a fully-formed `unique_ptr<LogicalOperator>`. Its
existence is the concrete evidence that DuckDB fuses "Binder" and "Logical
Plan" into a single pipeline stage. Defined at
`src/include/duckdb/planner/bound_statement.hpp`. See Chapter 1 §1.1 and
Chapter 4 §4.5.

### BufferManager

DuckDB's analogue of an OS page cache: pins/unpins disk blocks into
addressable memory, evicts under memory pressure, and spills to temporary
files when needed. `StandardBufferManager` is the concrete implementation. See
Chapter 17 §17.3.

---

### Checkpoint

The operation (explicit `CHECKPOINT` or implicit, policy-driven) that
consolidates a database's live, possibly-scattered transactional state back
into a compact set of fresh blocks, and is also what lets prior Write-Ahead
Log entries be discarded. A checkpoint always starts a new WAL file, even when the old one is empty. It also first binds every index
that still has buffered WAL changes; if that fails, an automatic checkpoint is
skipped and the WAL is kept, and an explicit `CHECKPOINT` fails. See
Chapter 17 §17.6 and Chapter 18 §18.4.

### ColumnBinding

A `{table_index, column_index}` pair — the currency every reference to a
column uses throughout the entire logical-plan layer, from the moment the
binder produces a `BoundColumnRefExpression` onward. Nothing in a bound plan
ever refers to a column by name again. See Chapter 4 §4.3.

### ColumnSegment

The compressed-bytes unit backing one column's data within one row group; a
`RowGroup` owns one `ColumnData` per column, and each `ColumnData` is a
sequence of `ColumnSegment`s, each carrying its own compression scheme and
zonemap. See Chapter 17 §17.2.1.

### Compression schemes, other (Patas, Chimp, Roaring bitmaps, zstd, dictionary encoding)

Rounding out DuckDB's compression framework alongside bitpacking/FOR, FSST,
ALP, and RLE (each with its own entry in this glossary): **Patas** and
**Chimp** [35] are floating-point schemes in the Gorilla
lineage [34], now generally superseded by ALP for the common
case; **dictionary encoding** replaces
repeated distinct values with small integer codes into a value table;
**Roaring bitmaps** compress sparse boolean/bitmap data; **zstd** is the
honest general-purpose fallback for data that doesn't fit any specialized
scheme well. See Chapter 17 §17.4.


### CompressionSegmentReader

A small, non-owning view over one block or page that checks every read stays
inside it (`src/include/duckdb/storage/compression/compression_segment_reader.hpp`).
The readers of dictionary, FSST, `DICT_FSST`, Zstd and Roaring segments
are built on it, so a corrupt file raises a `DataCorruptionException` instead of reading past a buffer. See
Chapter 17 §17.4.

### ConsumerToken

A token from the vendored moodycamel `ConcurrentQueue`
(`src/include/duckdb/parallel/concurrentqueue.hpp`). Each query puts its
tasks in its own producer sub-queue. Each worker thread dequeues with a
`ConsumerToken` (PR #26809), which makes it rotate across producers, so
one busy query cannot starve the others. See **TaskScheduler** and
Chapter 11 §11.7.

### CSE (Common Subexpression Elimination)

The optimizer pass (`COMMON_SUBEXPRESSIONS`, `src/optimizer/cse_optimizer.cpp`)
that recognizes an identical expression appearing more than once within a
single operator's expression list and computes it only once. Distinguished in
the book from two other, larger-scoped "don't compute it twice" mechanisms —
see Chapter 7 §7.6.7 for all three, precisely scoped.

### CTE (Common Table Expression)

A `WITH` clause. Represented in the logical plan by three cooperating node
types (the "CTE trio"): `LogicalMaterializedCTE` (computes the body once,
`LogicalCTERef` nodes read the shared result), `LogicalRecursiveCTE` (`WITH
RECURSIVE`'s fixed-point iteration), and `LogicalCTERef` itself (a leaf
referencing either). See Chapter 5 §5.4.12.

---

### DataChunk

A batch of up to `STANDARD_VECTOR_SIZE` (2048) rows, held as one `Vector` per
needed column — the unit that actually flows between operators during
execution. `GetCapacity()` tells how many rows its
vectors were set up to hold. See Chapter 10 §10.2.


### DataCorruptionException

The exception DuckDB raises when data it reads from disk (a segment, a WAL
entry, an index buffer, a serialized plan) fails a validity check
(`src/include/duckdb/common/exception.hpp`). Many readers check their input
and raise this exception instead of crashing on a corrupt or hostile file.

### DDL (Data Definition Language)

Statements that alter schema rather than data — `CREATE TABLE`, `CREATE
INDEX`, `ALTER`, `DROP`. Represented by their own logical-plan family
(`LogicalCreateTable`, `LogicalCreateIndex`, `LogicalAlter`, `LogicalDrop`,
Chapter 5 §5.4.14) and physical-plan family (the `schema/` directory,
Chapter 9 §9.3.8); DDL nodes carry almost no relational structure and are
mostly thin plumbing around a catalog mutation.

### Decorrelation

The general process of rewriting a correlated subquery (one referencing a
column from an enclosing query) into an equivalent join tree that ordinary
physical operators can execute — DuckDB has no physical operator that
re-evaluates a subquery once per outer row. In DuckDB this happens almost
entirely in the *planner*, right after binding (`Planner::CreatePlan` calls
`FlattenDependentJoins`), before the optimizer's first pass ever runs — a
direct implementation of Neumann and Kemper's "Unnesting Arbitrary Queries"
[12]. Full treatment in Chapter 6.

### DelimJoin

Shorthand for "duplicate-eliminated join" — the join shape decorrelation
produces when a correlated subquery's LHS needs to be de-duplicated before
being joined against, so the subquery body isn't logically re-evaluated once
per duplicate outer key. `LOGICAL_DELIM_JOIN` is not a distinct C++ class
(it's assembled from an ordinary `LogicalComparisonJoin` plus bookkeeping
fields); on the physical side it *does* get a dedicated class,
`PhysicalDelimJoin`, paired with `PhysicalColumnDataScan`s reading from a
`LogicalDelimGet`. "DELIM" is stated directly in the source as standing for
duplicate elimination. See Chapter 6 §6.4.

With default settings, delim joins live only for a moment. Right after
flattening, `DelimJoinCTERewriter` turns each one into plain joins over two
materialized CTEs, `__duckdb_delim_<n>` (the outer side) and
`__duckdb_delim_dedup_<n>` (its distinct keys). So the optimizer and the
physical planner see no `LOGICAL_DELIM_JOIN`, and `EXPLAIN` shows `CTE` and
`CTE_SCAN` instead of `DELIM_JOIN` and `DELIM_SCAN`. The delim-join form
appears only with `SET delim_join_as_cte = false` (a deprecated setting).
See Chapter 6 §6.4.1.


### Deserialization policy

An optional rule set given to a deserializer that forbids some logical types
and literal kinds while values are read
(`src/include/duckdb/common/serializer/deserialization_policy.hpp`, PR
#28416). For example, `json_deserialize_sql` uses it to reject `POINTER`
types.

### DML (Data Manipulation Language)

Statements that write data — `INSERT`, `UPDATE`, `DELETE`, `MERGE INTO`.
Logical family: `LogicalInsert`/`LogicalUpdate`/`LogicalDelete`/
`LogicalMergeInto` (Chapter 5 §5.4.13); physical family: the `persistent/`
directory (Chapter 9 §9.3.7).

### DP (Dynamic Programming, join-ordering sense)

Not "for loop" DP — here, the technique of building up optimal join orders
for larger sets of relations from already-computed optimal orders for smaller
subsets, memoized in a DP table keyed by connected subsets of the join
hypergraph. DuckDB's concrete algorithm for this is DPhyp (below); when the
search space exceeds a size threshold, DuckDB falls back to a greedy
heuristic instead. See Chapter 8 §8.5–§8.8.

### DPhyp

The specific dynamic-programming-over-connected-subgraphs join-enumeration
algorithm DuckDB implements (`src/optimizer/join_order/plan_enumerator.cpp`),
from Moerkotte and Neumann's "Dynamic Programming Strikes Back"
[20]. It
enumerates only join trees that the join hypergraph (below) says are
connected, rather than all possible orderings. See Chapter 8 §8.5.

### DuckTransaction

The concrete class answering "what is this transaction allowed to see right
now" — it carries a `SnapshotView` and consults it via `Sees()` checks as a
scan walks a row's version chain. See Chapter 18 §18.2.

---

### EXPLAIN (SQL)

A form of `EXPLAIN` (PR #26062) that returns the
*optimized* logical plan written back as runnable SQL. Each
`LogicalOperator` has a `ToSQL` method for this; the code lives in
`src/planner/sql_export/`. The output is not the original query: optimizer
rewrites show, and values known at plan time may appear as literals. See
Chapter 5 and Chapter 7.

### ExpressionExecutor

The class that interprets a bound expression tree (`WHERE` predicates, join
conditions, `SELECT`-list computations) once per `DataChunk`, dispatching
through per-expression-kind evaluation code compiled once when DuckDB itself
was built. It is the concrete evidence, expanded fully in Chapter 13, that
DuckDB does not generate or JIT-compile code per query. See Chapter 12.

---

### FIRST set (PEG matcher)

For a grammar rule, the set of tokens it can start with (plus whether it can
match nothing). The grammar compiler computes these sets
(`src/parser/peg/matcher_first_set.cpp`, PR #26451), and the matcher skips any
alternative that cannot start at the current token instead of trying it and
backtracking. The PR measured a 2.2× faster median parse with this pruning.
See Chapter 3.

### FOR (Frame of Reference)

In this book, always the *compression* term, never a loop: subtracting a
block's common minimum value ("the frame of reference") from every value in
the block so each residual can be packed in fewer bits than the column's
native width. DuckDB's own source uses this exact vocabulary
(`BitpackingMode::FOR`, `BitpackingMode::DELTA_FOR`). See **Bitpacking**
above and Chapter 17 §17.4.2.

### FSST (Fast Static Symbol Table)

DuckDB's own string-compression scheme (co-designed by DuckDB Labs, published
at VLDB [28]), compressing strings against a small per-block symbol table while
keeping compressed bytes directly comparable/prefix-searchable without full
decompression. Has a dedicated `VectorType` (`FSST_VECTOR`) so compressed
string data can stay encoded end-to-end through execution. See Chapter 17
§17.4.4.

---

### German strings

The fixed-size, variable-length-friendly string row layout (popularized by
Umbra [14], also since adopted by Arrow and Polars): short strings (≤ 12 bytes)
are stored entirely inline in the fixed-size row; longer strings store a
small inline header (length + prefix) plus a pointer to an out-of-line
variable-data page. This is what lets DuckDB keep intermediate rows
fixed-width even though `text`/`blob` columns aren't. See Chapter 14 §14.5.

---

### Hypergraph (join-ordering sense)

The graph DuckDB's join-order optimizer builds out of a query's `FROM`/`WHERE`
clause before searching for an order: nodes are base relations (or, at
non-reorderable boundaries like outer joins, whole already-planned subtrees),
and edges are join predicates. It's a *hyper*graph rather than a plain graph
specifically because an edge's endpoints are `JoinRelationSet`s that can each
contain more than one base relation. See Chapter 8 §8.3.

---

### IEJoin (IE-join)

`PhysicalIEJoin`, a `PhysicalRangeJoin` subclass implementing Khayyat et
al.'s "Lightning fast and space efficient inequality joins" algorithm
[26]: both inputs are sorted and
swept against each other to evaluate a predicate with two independent
inequalities and no exploitable equality, in close to `O((n+m)log(n+m)+k)`
rather than a naive nested loop's `O(n·m)`. Its simpler sibling for a single
range condition is `PhysicalPiecewiseMergeJoin`. See Chapter 8 §8.10.


### IntegerPartitionedTupleData

A row container (`src/common/types/row/integer_partitioned_tuple_data.cpp`) that partitions rows on the value of a small integer
key itself: one partition per value, one for `NULL`s and a sorted overflow
partition. `HashedSort` uses it for partitioned `COPY` and
`OVER (PARTITION BY ...)` when the only key is an integer column with known
min/max and there is no `ORDER BY`; every partition then holds one key and
needs no sort. See Chapter 15.

---

### JIT compilation

Just-in-time compilation of a query into machine code specific to that one
query's plan — the technique HyPer and Umbra use (via the produce/consume
model [13]; Umbra [14]) and which DuckDB deliberately does **not** use anywhere in its core
query-processing pipeline. Chapter 13 is dedicated to answering "does DuckDB
do this" exhaustively, with negative evidence, and to explaining the
tradeoff DuckDB accepts instead.

---

### Legacy parser extension

An out-of-tree extension, `legacy_parser`, that brings back a
Postgres-style parser with v1.5 behavior. `SET enable_legacy_parser = true`
autoloads it (PR #26479). It is a best-effort fallback; the PEG parser is the
default. See Chapter 3 and Appendix A.

### LogicalAggregate

The logical-plan node representing `GROUP BY` and aggregate computation
(including grouping sets) as a single node type, regardless of how many
grouping columns or aggregate functions are involved. See Chapter 5 §5.4.6.

### LogicalComparisonJoin

The logical-plan node for an ordinary (non-dependent) join with a
comparison-based condition — what a `JOIN ... ON` becomes once bound, before
the physical planner picks a concrete join algorithm. See Chapter 5 §5.4.4.

### LogicalDependentJoin

The marker node the binder builds for a correlated subquery, before
`FlattenDependentJoins` rewrites it away. By the time the optimizer's first
pass runs, no `LogicalDependentJoin` should remain in the tree. See Chapter 5
§5.4.5 and Chapter 6.

### LogicalFilter

The logical-plan node for a `WHERE`/`HAVING`-style predicate applied to its
child's output. One of the optimizer's most active passes,
`filter_pushdown`, exists specifically to move these as close to the data
source as it legally can. See Chapter 5 §5.4.2.

### LogicalGet

The logical-plan node for *any* data source — not just a base table scan,
but table functions, `read_csv`, etc. — anything that produces rows from
outside the plan itself. See Chapter 5 §5.4.1.

### LogicalOperator

The abstract base class for every node in a logical plan: carries `type`,
`children`, `expressions`, output `types`, and an `estimated_cardinality`.
Produced directly by the binder (see **BoundStatement**), rewritten in place
by the optimizer's ~45 fixed passes, and finally walked by the
`PhysicalPlanGenerator` to produce a `PhysicalOperator` tree. See Chapter 5.

---

### Merge Path

The algorithm (a textbook two-way idea [32], generalized by
DuckDB to K-way) that
computes, via binary search, exactly where a parallel merge's output would be
split among worker threads *without ever performing the merge* — letting
independent threads each produce a contiguous, coordination-free slice of a
globally sorted output. See Chapter 15 §15.6.2–§15.6.3.

### MetaPipeline

A group of one or more `Pipeline`s that share a common sink — e.g., a hash
join's build-side pipeline and any pipelines feeding into the same hash
table. See Chapter 11 §11.2.1.

### Morsel ("morsel-driven parallelism")

The database-literature term (Leis et al., describing HyPer
[31]) for
handing out disjoint, unclaimed chunks of scan work to whichever thread asks
next, rather than statically pre-partitioning up front. DuckDB's actual
mechanism (row-group- and file-split-granularity work-stealing via
`GetPartitionData`/`SourceParallelism`) matches this closely enough that the
outside world applies the label — but the word "morsel" itself occurs
**zero** times in DuckDB's own `src/execution` or `src/parallel` source.
The one place it surfaces is a gloss in the DiDi
teaching slides [8, slide 14], not DuckDB's own vocabulary. See
Chapter 11 §11.8 for the full, honest accounting.

### MVCC (Multi-Version Concurrency Control)

The general concurrency-control family DuckDB implements as snapshot
isolation via row-versioning: readers and writers never block each other
because a reader sees a frozen, consistent snapshot of the database while
writers append new versions rather than overwriting in place. See Chapter 18.

---

### Node4 / Node16 / Node48 / Node256

The four adaptive internal-node representations of DuckDB's ART
implementation, chosen by `NodePtr::GetInternalNodeType`
(`src/execution/index/art/node.cpp`) based purely on how many children a node
currently has. `Node4` and `Node16` are instances of the template
`BaseNode<CAPACITY, TYPE>` (`base_node.hpp`); `Node48` and `Node256` have
their own files: `Node4`/`Node16` use linear or
binary/SIMD-scanned key arrays, `Node48` uses a 256-entry byte→slot index
plus a compact 48-pointer array, and `Node256` indexes 256 child pointers
directly by key byte. Nodes upgrade and downgrade representation as children
are added or removed. See Chapter 16 §16.5.

---

### Optimizer

The class (`src/optimizer/optimizer.cpp`) driving DuckDB's fixed, ~45-pass,
run-once, fixed-order rewrite pipeline over a `LogicalOperator` tree — a
deliberate architectural choice (no cost-based pass-selection loop; a pass
that's already run doesn't get a second chance) covered in full in Chapter 7.
Two of its passes, `SELF_JOIN_AGGREGATE` and `REDUNDANT_ORDER_KEYS`, have
their own entries.

---

### ParsedExpression

The parser's expression AST base class (`src/parser/expression/`) — e.g.
`ColumnRefExpression` (a raw, unresolved dotted-name chain like `["o",
"custkey"]`), `FunctionExpression` (name + arguments, with no distinction yet
between scalar and aggregate), `ComparisonExpression`, `SubqueryExpression`.
Purely syntactic: nothing here has been checked against a catalog. See
Chapter 3 §3.6.


### PassthroughStatement

A statement type (`src/include/duckdb/parser/statement/passthrough_statement.hpp`) used after `CONNECT` to a database that has its own
grammar. Every statement except `DISCONNECT` is matched by a tiny grammar
(one or more tokens) and sent as text to that database, unparsed by DuckDB
(PR #25990). See Chapter 3.

### PEG (Parsing Expression Grammar)

The parsing formalism DuckDB's current, native, hand-written parser is built
on (`src/parser/peg/`), replacing the historical `libpg_query`-derived parser
(covered, for readers on a v1.x release, in Appendix A). Since the v2.0.0
release (2026-10-20) the PEG parser is the parser of a stable DuckDB too. A PEG grammar
resolves ambiguity deterministically via ordered choice, unlike a classic
context-free grammar. See Chapter 3.


### PEGRuleId

A generated enum (`src/include/duckdb/parser/peg/peg_rule_id.hpp`) with one
value per grammar rule. Generated transformer code refers to rules by this
id instead of by name; rules added by grammar extensions get ids above
`PEGRuleId::BUILTIN_COUNT`. See Chapter 3.

### PhysicalHashAggregate

The general-case physical operator for `GROUP BY`, implementing two-phase
parallel hash aggregation (thread-local pre-aggregation, then partition-wise
combination). Narrower conditions can instead select
`PhysicalPerfectHashAggregate` (a small, known-range key needs no hashing at
all) or `PhysicalPartitionedAggregate`. See Chapter 14 §14.6–§14.9.

### PhysicalHashJoin

The physical operator chosen for an equality-predicate `LogicalComparisonJoin`
— build a hash table from one side (typically the smaller, the "build side"),
probe it with the other ("probe side"). See Chapter 9 §9.3.2 and Chapter 8
§8.9–§8.10 for how build/probe side and join algorithm are chosen.

### PhysicalOperator

The abstract base class for every node in a physical plan: arena-owned
(`children` are reference-wrapped, not `unique_ptr`, unlike `LogicalOperator`),
and implementing up to three roles — Sink, Source, Operator — depending on
what it does. Produced from a `LogicalOperator` tree by the
`PhysicalPlanGenerator`. See Chapter 9.

### PhysicalPlanGenerator

The class (`src/execution/physical_plan_generator.cpp`) that walks an
optimized `LogicalOperator` tree and, for each node, dispatches to a
`plan_*.cpp` file that picks and constructs a concrete `PhysicalOperator` —
e.g., deciding whether a `LogicalComparisonJoin` becomes a
`PhysicalHashJoin`, a merge join, or an `IEJoin`. See Chapter 9 §9.4–§9.5.


### PhysicalResultSink

The root operator of every plan that produces rows
(`src/include/duckdb/execution/operator/helper/physical_result_sink.hpp`).
With **Async results**, it feeds the result buffer through a `ResultFormat`.
There are no separate batch or Arrow collector operators (PR #25801). See
Chapter 9 §9.3.10.

### PhysicalTableScan

The physical operator that reads rows from a base table's row groups
(applying any pushed-down filter during the scan itself), one of several
specialized siblings in the physical plan's `scan/` directory. See Chapter 9
§9.3.3.

### PhysicalTopN

The physical operator executing a fused `ORDER BY ... LIMIT`
(`LogicalTopN`), using a bounded top-k structure instead of a full sort. See
Chapter 7 §7.6.4 and Chapter 9.

### Pipeline

A linear chain of operators between one source and one sink, decomposed from
a physical plan tree — e.g., a hash join's build side (scan → build hash
table sink) is one pipeline, and its probe side (scan → probe → next sink)
is a dependent second pipeline. `Pipeline`s, not the raw operator tree, are
the actual unit of execution scheduling. See Chapter 11 §11.2.

### PipelineExecutor

The driver that pulls `DataChunk`s through one pipeline's operator chain on
a given thread, calling each operator's `Execute`/`Sink`/`GetData` as
appropriate. See Chapter 11 §11.4.

### Produce/consume model

Thomas Neumann's 2011 code-generation model [13], in which each operator
in a query plan contributes a fragment of generated code to one function
chained end to end, so a tuple never leaves a CPU register unnecessarily.
The foundational technique behind HyPer/Umbra-style compiling engines —
and explicitly **not** what DuckDB's `ExpressionExecutor`/`PipelineExecutor`
do; DuckDB interprets a generic operator tree instead. See Chapter 13.

---

### Query A / Query B / Query C (running examples)

The book's three recurring worked queries, introduced so a reader can watch
one query's shape change across chapters rather than re-orienting each time.
Query A (join + `GROUP BY` + `ORDER BY ... LIMIT`, from the front matter,
traced end to end in Chapter 19) has no correlation; Query B (a correlated
scalar subquery, introduced in Chapter 6) exercises decorrelation; Query C
(introduced in Chapter 8) has enough joins to make join ordering
non-trivial.

### QueryNode

The parser's representation of one `SELECT`/set-operation/recursive-CTE
block (`SelectNode`, `SetOperationNode`, `RecursiveCTENode`), holding a
`select_list`, `from_table` (a `TableRef`), `where_clause`, and grouping/
result-modifier fields — still pre-catalog, pre-binding. See Chapter 3 §3.6.

---

### Read-ahead (scan read-ahead)

Starting the I/O for data a scan will need soon, while it still works on
earlier data (`src/include/duckdb/parallel/scan_read_ahead.hpp`). Its memory
budget is capped at `memory_limit/8`, the files it keeps open count against a
shared limit, and it skips row groups that a Top-N boundary excludes. No
chapter covers read-ahead in depth.

### REDUNDANT_ORDER_KEYS

An optimizer pass (`src/optimizer/remove_redundant_order_keys.cpp`,
PR #25878). In
`ORDER BY id, payload`, if `id` is already unique (a primary key, a unique
`NOT NULL` column, or a `GROUP BY` key), `payload` cannot change the order,
so the pass drops it. See Chapter 7.

### RESOURCE_IN_USE

The error (`DUCKDB_ERROR_RESOURCE_IN_USE = 44` in `duckdb.h`) a connection
returns when a new statement is submitted while a previous result has not
finished. See **Async results**.

### ResultFormat

The interface (`src/include/duckdb/main/result_format.hpp`) that turns the
rows a query produces into *result units* on the worker threads. Chunks
(`DataChunk`s) are the default format; Arrow is `ArrowFormat`. See PRs
#25727 and #25801, and **Async results**.

### RLE (Run-Length Encoding)

A compression scheme storing a consecutive run of identical values once
alongside a run length, well-suited to sorted or naturally-repetitive
columns. See Chapter 17 §17.4.5.

### RowGroup

A horizontal slice of a table (by default sized to a fixed multiple of
`STANDARD_VECTOR_SIZE`) — the unit at which zonemaps are computed, columns
are compressed independently, and parallel scans/checkpoints are divided.
See Chapter 17 §17.2.1.

---

### SelectionVector

An array of row indices providing indirection instead of copying — e.g., a
filter writes surviving row indices into a `SelectionVector` rather than
physically compacting matching rows. An "empty" selection vector is the
identity permutation at zero storage cost. The underlying mechanism behind
`DICTIONARY_VECTOR` (see **VectorType**). See Chapter 10 §10.4.


### SELF_JOIN_AGGREGATE

An optimizer pass (`src/optimizer/self_join_aggregate.cpp`, PR #28259). It replaces a
self-join on equal keys plus one `<>`, whose parent only asks which keys
exist (for example through `IN (...)`), with
`GROUP BY keys HAVING min(x) <> max(x)` over one scan. The motivating query
is TPC-DS q95. See Chapter 7.

### Sink / Source / Operator (execution interface roles)

The three roles a `PhysicalOperator` can implement: **Source** produces
chunks (a scan); **Operator** transforms a chunk mid-pipeline (a filter,
projection); **Sink** consumes chunks and accumulates state (a hash join's
build side, a hash aggregate). One physical operator can implement more than
one role — a hash join is both a Sink (build side) and an Operator (probe
side). See Chapter 11 §11.3.

### SnapshotView

The small struct (`{transaction_id, visibility_bound}`) that answers, for any
row version, "can this transaction see this?" — the concrete mechanism
behind snapshot isolation. See Chapter 18 §18.2.

### SQLStatement

The parser's top-level output type (`vector<unique_ptr<SQLStatement>>` is
what `Parser::statements` holds) — `SelectStatement`, `InsertStatement`,
`CreateStatement`, and so on, one per parsed statement, handed directly to
the `Binder`. See Chapter 3 §3.6.


### Storage version

The number that tells which DuckDB releases can read a database file
(`StorageVersion` in `src/include/duckdb/storage/storage_info.hpp`, with
`src/storage/version_map.json`). v2.0.0 and v2.0.1 both use storage version
69. See Chapter 17.

---

### TableRef

The parser's representation of a `FROM`-clause source before binding:
`BaseTableRef` (bare table name), `JoinRef` (left/right `TableRef`s plus a
condition), `SubqueryRef`, `TableFunctionRef`, and others. See Chapter 3
§3.6.

### TaskScheduler

The process-wide (per-`DuckDB`-instance) thread pool that actually runs
vectorized execution, shared across every concurrent query on every
concurrent connection; a query's `Executor` breaks its plan into `Pipeline`s
and submits tasks to this shared pool rather than owning dedicated threads.
Worker threads take tasks with a `ConsumerToken`, so they rotate fairly
between queries. See Chapter 1 §1.2 and Chapter 11 §11.7.


### TransformFramePlan

A static record, written by the transformer generator, that describes how to
transform one grammar rule: which child rules to visit and where to put
their results (`src/include/duckdb/parser/peg/transformer/peg_transformer.hpp`,
PR #27849). Three shared functions interpret these records instead of one
generated function per rule ("plan-driven trampolines"). See Chapter 3.

---

### UnboundIndex

An index loaded from disk but not yet usable for lookups
(`src/include/duckdb/execution/index/unbound_index.hpp`). It is bound (turned
into an `ART`) on first use. While unbound, WAL replay buffers index changes
in it. A checkpoint binds such indexes first (PR #26814), so those changes
are not lost. See Chapter 18 (Chapter 16 does not cover index
binding).

### UndoBuffer

The append-only, in-memory log every open transaction owns, holding
previous versions of rows it has touched — the actual mechanism (not the
WAL) providing isolation and rollback. Entries get a real timestamp on
`COMMIT` rather than being deleted, and are physically undone on `ROLLBACK`.
See Chapter 18 §18.3.

---

### v2.0.0

The release tag `v2.0.0` (commit `d54cbbe3df1`, 2026-10-20) on branch
`v2.0-cyanoptera`. It is the first release with the PEG parser. The commit
this book describes, cb5673685153, is `v2.0.0-1041-gcb567368515`: 1,041
commits after this tag.

### ValidityMask

A NULL bitmap attached to a `Vector`, with a fast path for the common "no
NULLs at all" case so that path costs nothing extra. See Chapter 10 §10.5.

### Vector

The physical container for one column's worth of up to 2048 values within a
`DataChunk`. A `Vector`'s *logical* type (e.g. `INTEGER`) is independent of
its current *physical* encoding, which is its `VectorType`. See Chapter 10
§10.2.

### Vectorized execution

Processing data in batches (DuckDB's `DataChunk`s of up to 2048 rows) through
tight, type-specialized inner loops, rather than one row at a time
(the classic Volcano model [29]) or one compiled function per
query (the HyPer/Umbra produce/consume model [13]). The
batch-at-a-time design follows MonetDB/X100 [27]. This is DuckDB's actual execution
strategy — batching and cache-conscious, specialized-but-precompiled loops,
not per-query code generation. See Chapter 12 and Chapter 13.

### VectorType

The enum describing a `Vector`'s current physical encoding, independent of
its logical SQL type — six variants: `FLAT_VECTOR` (standard, uncompressed),
`CONSTANT_VECTOR` (one value, implicitly repeated), `DICTIONARY_VECTOR` (a
`SelectionVector` over another vector's distinct values), `SEQUENCE_VECTOR`
(start + increment, no stored values), `FSST_VECTOR` (FSST-compressed string
data, still queryable without full decompression), and `SHREDDED_VECTOR`
(a shredded variant-type representation). See Chapter 10 §10.3.

---

### WAL (Write-Ahead Log)

The on-disk mechanism providing *durability* (a committed change survives a
crash) — a distinct concern from the in-memory `UndoBuffer`'s isolation and
rollback. Replayed at startup according to one of four `WALInitState`s;
made unnecessary for old entries once a `CHECKPOINT` folds their effect into
the main file. Replay truncates a broken last entry instead
of recovering uncommitted data, and checks `ALTER` entries and index sizes.
See Chapter 18 §18.4.

---

### Zonemap

Per-`ColumnSegment` min/max (and related) statistics that let a scan skip an
entire row group without reading it, when a filter predicate can prove no
row in that group could match. Coarser-grained than an ART index (it can't
do point lookups) but automatic — every column segment has one, with no
`CREATE INDEX` required. See Chapter 16 §16.1 and Chapter 17 §17.5.
