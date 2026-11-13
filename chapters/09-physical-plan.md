# Chapter 9 — Physical Plan Generation

Chapter 5 spent its entire length establishing one point and refusing to let go
of it: a logical plan is *declarative*. A `LogicalComparisonJoin` node says "the
rows of my left child and my right child should be combined wherever
`o.orderkey = l.orderkey` holds" and stops there — it does not choose an
algorithm. A `LogicalAggregate` says "group these rows by `o.custkey` and
compute `SUM(l.extendedprice)` per group" and stops there too. Chapters 7 and 8
covered the ~45-pass optimizer that rewrites this tree's *shape* (relocating
filters, choosing a join order, fusing `ORDER BY` + `LIMIT` into `LogicalTopN`)
without ever changing its *vocabulary* — every rewrite still produces one of
the same 47-or-58 `LogicalOperator` subclasses Chapter 5 catalogued, because
none of those passes commit to an algorithm either.

This chapter is about the one step in the whole pipeline whose entire job is
to stop deferring. `PhysicalPlanGenerator` (`src/execution/physical_plan_generator.cpp`)
walks the optimized logical tree and, for every node, manufactures a concrete
C++ object: `LogicalComparisonJoin` becomes an actual `PhysicalHashJoin` (or
`PhysicalPiecewiseMergeJoin`, or `PhysicalIEJoin`, or `PhysicalNestedLoopJoin`
— Chapter 8 covered the *criteria*; §9.6 below shows exactly where those
criteria turn into a `new`-equivalent `Make<PhysicalHashJoin>(...)` call).
`LogicalAggregate` becomes one of four different concrete aggregate operators
depending on what the `GROUP BY` columns and aggregate functions actually look
like. `LogicalTopN` becomes `PhysicalTopN` directly. Every node in the tree
that survives this step is no longer a statement of intent — it is a specific
object, with a specific execution strategy, ready to be handed to the
`Executor` (Chapter 11) and run on some number of threads.

Put in the vocabulary Chapter 1 introduced: the logical plan says orders and
lineitem should be joined on `orderkey`. The physical plan says a
`PhysicalHashJoin` will do it, with `orders` built into an in-memory hash
table first (the *build side*) and `lineitem` streamed against it (the *probe
side*), with as many worker threads as the `TaskScheduler` gives this query.
Everything in this chapter is about how DuckDB gets from the first sentence to
the second.

## 9.1 The base class: `PhysicalOperator`

Every concrete physical operator this chapter discusses derives from one
class, and it is worth reading in full before looking at any subclass, exactly
the way Chapter 5 read `LogicalOperator` in full first:

```cpp
// src/include/duckdb/execution/physical_operator.hpp:42-72
//! PhysicalOperator is the base class of the physical operators present in the execution plan.
class PhysicalOperator {
public:
	static constexpr const PhysicalOperatorType TYPE = PhysicalOperatorType::INVALID;

public:
	PhysicalOperator(PhysicalPlan &physical_plan, PhysicalOperatorType type, vector<LogicalType> types,
	                 idx_t estimated_cardinality);
	virtual ~PhysicalOperator() {
	}

	//! Deleted copy constructors.
	PhysicalOperator(const PhysicalOperator &other) = delete;
	PhysicalOperator &operator=(const PhysicalOperator &) = delete;

	//! The child operators.
	ArenaLinkedList<reference<PhysicalOperator>> children;
	//! The physical operator type.
	PhysicalOperatorType type;
	//! The return types.
	vector<LogicalType> types;
	//! The estimated cardinality.
	idx_t estimated_cardinality;

	//! The global sink state. Published under `lock` by Pipeline::ResetSink on a worker;
	//! a reader racing pipeline initialization must hold `lock` to observe it safely
	unique_ptr<GlobalSinkState> sink_state;
	//! The global operator state.
	unique_ptr<GlobalOperatorState> op_state;
	//! Lock for (re)setting any of the operator states.
	mutex lock;
	// ...
};
```

Five fields worth walking through one at a time, because — same as Chapter
5's treatment of `LogicalOperator` — every concrete subclass in this chapter
is defined almost entirely by what it puts in these fields and which virtual
methods it overrides.

### 9.1.1 `children`: reference-wrapped, arena-owned — not `unique_ptr`

The very first field is where the biggest structural difference from Chapter
5 lives, and it's worth being precise about it because it's easy to skim past.
`LogicalOperator::children` is `vector<unique_ptr<LogicalOperator>>`
(`src/include/duckdb/planner/logical_operator.hpp:46`) — the tree *owns* its
nodes, each parent holding its children by unique, exclusive pointer.
`PhysicalOperator::children` is `ArenaLinkedList<reference<PhysicalOperator>>`
— a linked list of *reference wrappers*, owning nothing. The actual storage
for every `PhysicalOperator` in the tree lives somewhere else entirely: a
`PhysicalPlan` arena (§9.4.3), and the tree of `children` links is just a set
of pointers into that arena.

This is not a stylistic preference; it follows directly from what changed
between the two trees. A `LogicalOperator` tree can be copied (`Copy()`,
Chapter 5 §5.2.4), serialized and deserialized for a prepared statement,
mutated in place by ~45 independent optimizer passes, and — for a CTE
referenced more than once — the same logical subtree needs to appear at
multiple points in the final tree as genuinely independent objects. All of
that calls for exclusive, movable ownership: `unique_ptr` is the right tool.

A physical plan has none of those requirements. By the time
`PhysicalPlanGenerator` runs, there is no more rewriting to do, no more
serialization to worry about (a physical plan is never itself the thing a
prepared statement serializes — the *logical* plan is, and it's re-planned
into a fresh physical tree on every `EXECUTE`), and the tree's entire lifetime
is exactly one query's execution, start to finish. Given that narrower
lifetime, arena ownership is strictly cheaper: allocating every operator out
of one bump-pointer `ArenaAllocator` (`PhysicalPlan::arena`, §9.4.3) is faster
than a `unique_ptr`'s heap allocation per node, and letting `children` hold
bare references rather than ownership means the same operator object can
legitimately be *pointed to* from more than one place (a duplicate-eliminated
join's delimiter scans, `PhysicalDelimJoin::delim_scans` in §9.3.2, are
exactly this — several parts of the tree reference the same underlying
`PhysicalColumnDataScan` without any of them owning it) — something a
`unique_ptr`-owned tree structurally cannot express without reaching for a
`shared_ptr` or a raw pointer escape hatch. Arena ownership is the physical
layer solving "share cheaply within one query's lifetime" the way `unique_ptr`
solved "own exclusively, rewrite freely, serialize on demand" for the logical
layer — two different ownership models for two answers to two different
questions.

### 9.1.2 `type`, `types`, `estimated_cardinality`

These three carry over from `LogicalOperator` almost unchanged in spirit.
`type` is the `PhysicalOperatorType` tag (§9.2) that lets code recover the
concrete subclass from a bare `PhysicalOperator&` without `dynamic_cast` —
same job `LogicalOperatorType` did. `types` is the resolved output schema,
filled in during the same `ResolveOperatorTypes()` walk (§9.4.1) that Chapter
5 covered for the logical tree. `estimated_cardinality` is the row-count
estimate the join-order optimizer (Chapter 8) computed — copied across from
the logical operator it was created from (`PlanInternal`, §9.4.3, does this
copy explicitly: `physical_plan->Root().estimated_cardinality =
op.estimated_cardinality`), not recomputed. Nothing about cardinality
estimation happens fresh at this stage; the physical plan generator only
*reads* the number the optimizer already produced.

### 9.1.3 `sink_state`, `op_state`, `lock` — a preview of Chapter 11

The last three fields are the ones `LogicalOperator` has no analogue for at
all, and they exist for a reason this chapter can only gesture at:
`PhysicalOperator` is not just a static description of an algorithm, it is
also where that algorithm's *mutable runtime state* lives once execution
starts. `sink_state` (`unique_ptr<GlobalSinkState>`) is where a
`PhysicalHashJoin` keeps its hash table once the build side has finished
scanning `orders`; `op_state` (`unique_ptr<GlobalOperatorState>`) is
analogous state for a non-sink operator; `lock` (a `mutex`) guards the moment
either of those gets (re)set, because more than one worker thread can race to
initialize a pipeline's operators simultaneously. The comment directly on the
field is specific about exactly this hazard: `sink_state` is "published under
`lock` by `Pipeline::ResetSink` on a worker; a reader racing pipeline
initialization must hold `lock` to observe it safely"
(`src/include/duckdb/execution/physical_operator.hpp:66-67`).

This chapter deliberately does not explain the Sink/Source/Operator protocol
those states belong to, or when `Pipeline::ResetSink` actually runs, or why a
race is even possible — that is Chapter 11's entire subject, in full. All this
chapter needs you to notice is *that the fields exist on every physical
operator, from the moment `PhysicalPlanGenerator` constructs it*, and *why*
they have no logical-plan counterpart: a `LogicalOperator` never runs — it's
pure description, evaluated by later stages, and never itself needs a place
to keep a hash table. A `PhysicalOperator` is the thing that *will* run, and
these three fields are where "running" leaves its footprints.

### 9.1.4 `InitializeOutputChunk`: which output columns get their own memory

Between two operators in a pipeline sits a `DataChunk` that holds the first
operator's output (Chapter 10 covers `DataChunk` itself). The pipeline executor
creates these chunks once, before execution starts. The simple approach is to
give every column of every intermediate chunk its own buffer of
`STANDARD_VECTOR_SIZE` values. But many operators never write into those
buffers. A filter, for example, only *slices* its input: it points each output
column at the input column and adds a selection vector. A projection that just
passes a column through only *references* the input column. Such buffers would
be allocated and never used.

So `PhysicalOperator` has a virtual method that lets each operator decide this
for itself:

```cpp
// src/include/duckdb/execution/physical_operator.hpp:94-97
	//! Initialize the vectors this operator uses for its output
	virtual void InitializeOutputChunk(Allocator &allocator, DataChunk &chunk) const {
		chunk.Initialize(allocator, types);
	}
```

The default allocates every column. The pipeline executor calls this
method on the operator that *produces* each intermediate chunk
(`src/parallel/pipeline_executor.cpp:65`). Several operators override it and
pass a `vector<bool>` to `DataChunk::Initialize`, where `false` means "do not
allocate a buffer for this column":

```cpp
// src/execution/operator/filter/physical_filter.cpp:36-39
void PhysicalFilter::InitializeOutputChunk(Allocator &allocator, DataChunk &chunk) const {
	// every output column only references or slices the input
	chunk.Initialize(allocator, types, vector<bool>(types.size(), false));
}
```

```cpp
// src/execution/operator/projection/physical_projection.cpp:34-48
void PhysicalProjection::InitializeOutputChunk(Allocator &allocator, DataChunk &chunk) const {
	// column references only reference their input vector. Only computed columns need a buffer
	vector<bool> initialize;
	bool any_computed = false;
	for (auto &expression : select_list) {
		const bool computed = expression->GetExpressionClass() != ExpressionClass::BOUND_REF;
		initialize.push_back(computed);
		any_computed = any_computed || computed;
	}
	if (!any_computed) {
		chunk.InitializeEmpty(types);
		return;
	}
	chunk.Initialize(allocator, types, initialize);
}
```

For example, the projection `SELECT a, b + 1, c` has three output columns. Only
`b + 1` is computed, so only the middle column gets a buffer; `a` and `c` will
point at the input vectors. `PhysicalHashJoin` does the same for its probe-side
output columns (`src/execution/operator/join/physical_hash_join.cpp:2256-2267`),
and `PhysicalStreamingLimit` and `PhysicalStreamingSample` override it too.
`test/sql/projection/reference_only_output.test` checks that such outputs still
give correct results for empty inputs, NULLs, lists and structs.

## 9.2 The enum: `PhysicalOperatorType`

Every `PhysicalOperator::type` field holds one value from this enum:

```cpp
// src/include/duckdb/common/enums/physical_operator_type.hpp:18-138 (re-flowed: one value per source line, condensed here for page width)
enum class PhysicalOperatorType : uint8_t {
    INVALID,
    ORDER_BY, LIMIT, STREAMING_LIMIT, LIMIT_PERCENT, TOP_N, WINDOW, UNNEST,
    UNGROUPED_AGGREGATE, HASH_GROUP_BY, PERFECT_HASH_GROUP_BY, PARTITIONED_AGGREGATE,
    LIMITED_DISTINCT, FILTER, PROJECTION, COPY_TO_FILE, BATCH_COPY_TO_FILE,
    RESERVOIR_SAMPLE, STREAMING_SAMPLE, STREAMING_WINDOW, PIVOT, COPY_DATABASE,

    // Scans
    TABLE_SCAN, DUMMY_SCAN, COLUMN_DATA_SCAN, CHUNK_SCAN, RECURSIVE_CTE_SCAN,
    RECURSIVE_RECURRING_CTE_SCAN, CTE_SCAN, DELIM_SCAN, EXPRESSION_SCAN, POSITIONAL_SCAN,

    // Joins
    BLOCKWISE_NL_JOIN, NESTED_LOOP_JOIN, HASH_JOIN, CROSS_PRODUCT, PIECEWISE_MERGE_JOIN,
    IE_JOIN, LEFT_DELIM_JOIN, RIGHT_DELIM_JOIN, POSITIONAL_JOIN, ASOF_JOIN,

    // SetOps
    UNION, RECURSIVE_CTE, RECURSIVE_KEY_CTE, CTE,

    // Updates
    INSERT, BATCH_INSERT, DELETE_OPERATOR, UPDATE, MERGE_INTO,

    // Schema
    CREATE_TABLE, CREATE_TABLE_AS, BATCH_CREATE_TABLE_AS, CREATE_INDEX, ALTER,
    CREATE_SEQUENCE, CREATE_VIEW, CREATE_SCHEMA, CREATE_MACRO, DROP, PRAGMA,
    TRANSACTION, CREATE_TYPE, CREATE_TRIGGER, ATTACH, DETACH,

    // Helpers
    EXPLAIN, EXPLAIN_ANALYZE, EMPTY_RESULT, EXECUTE, PREPARE, VACUUM, EXPORT, SET,
    SET_VARIABLE, LOAD, INOUT_FUNCTION, RESULT_COLLECTOR, RESET, EXTENSION,
    VERIFY_VECTOR, UPDATE_EXTENSIONS, CONNECT, DISCONNECT, EXTERNAL_RESOURCE,

    // Secret
    CREATE_SECRET,

    // Physical-only recursive CTE specialization. Keep appended to preserve existing enum values.
    RECURSIVE_KEY_JOIN, SECURE_VIEW,
    // Source of a MERGE INTO action pipeline. Keep appended to preserve existing enum values.
    MERGE_ACTION_SOURCE,
};
```

The first thing to notice, sitting right there in every value above, is a
small naming-convention difference from Chapter 5 that's easy to trip over if
you go looking for these values in a debugger or a log line:
**`PhysicalOperatorType` values have no `PHYSICAL_` prefix.**
`LogicalOperatorType::LOGICAL_COMPARISON_JOIN` becomes
`PhysicalOperatorType::HASH_JOIN`, not `PHYSICAL_HASH_JOIN` — the class is
named `PhysicalHashJoin`, but its type tag is bare `HASH_JOIN`. There's no
documented rationale for the asymmetry in the header; it's simply the
convention this enum has always used, and it's worth internalizing now so a
bare `HASH_JOIN` string in an `EXPLAIN` plan or a log doesn't read as
unfamiliar later.

The category groupings themselves track Chapter 5's `LogicalOperatorType`
groupings closely, which is itself informative — it means the physical layer
is organized around the same conceptual seams the logical layer was, even
though the *values within* each category don't correspond one-to-one:

- **General operators** (the unlabeled first block): sort/limit/top-n
  (`ORDER_BY`, `LIMIT`, `STREAMING_LIMIT`, `LIMIT_PERCENT`, `TOP_N`),
  `WINDOW`/`UNNEST`, the four aggregate strategies (`UNGROUPED_AGGREGATE`,
  `HASH_GROUP_BY`, `PERFECT_HASH_GROUP_BY`, `PARTITIONED_AGGREGATE` — §9.3.1
  explains why there are four), `LIMITED_DISTINCT`, `FILTER`, `PROJECTION`,
  the copy-out operators, sampling, `PIVOT`, and `COPY_DATABASE`. Notice this
  single flat category already covers more ground than
  `LogicalOperatorType`'s equivalent "general" block did — the logical layer
  had one `LOGICAL_AGGREGATE_AND_GROUP_BY` value for all `GROUP BY`
  processing; the physical layer needs four values, because "how to compute a
  `GROUP BY`" is exactly the kind of algorithmic decision the logical layer
  is designed never to make.
- **Scans**: `TABLE_SCAN` is joined by several specialized, non-table
  sources (`DUMMY_SCAN`, `COLUMN_DATA_SCAN`, `CHUNK_SCAN`, the recursive-CTE
  scan variants, `CTE_SCAN`, `DELIM_SCAN`, `EXPRESSION_SCAN`,
  `POSITIONAL_SCAN`) — §9.3.3 covers what each is for.
- **Joins**: ten values, directly reflecting Chapter 8's algorithm-selection
  criteria materialized into distinct classes — §9.3.2 and §9.6 cover this
  category in the most depth of any in the chapter, because it's the category
  where "logical says what, physical says how" is most visible.
- **SetOps**: `UNION`, the two recursive-CTE variants, `CTE`.
- **Updates**: DuckDB's write path — `INSERT`, `BATCH_INSERT`,
  `DELETE_OPERATOR`, `UPDATE`, `MERGE_INTO` — mirroring Chapter 5's DML
  category almost value-for-value.
- **Schema**: DDL — table/index/view/schema/macro/type/trigger creation,
  `ALTER`, `DROP`, `PRAGMA`, `TRANSACTION`, `ATTACH`/`DETACH` — again mirroring
  Chapter 5's DDL category closely, because DDL statements carry almost no
  algorithmic choice for the physical layer to make; a `PhysicalCreateTable`
  is barely more than a thin wrapper around the same bound `CreateInfo` the
  logical node already held.
- **Helpers**: the catch-all for anything that isn't relational computation
  in the classical sense — `EXPLAIN`/`EXPLAIN_ANALYZE`, `EMPTY_RESULT`,
  prepared-statement plumbing (`EXECUTE`, `PREPARE`), `VACUUM`, `EXPORT`,
  session-setting statements (`SET`, `SET_VARIABLE`, `RESET`, `LOAD`), table
  functions used for side effects (`INOUT_FUNCTION`), and — the one every
  query actually terminates in — `RESULT_COLLECTOR` (§9.3.10). `EXTENSION`
  and `VERIFY_VECTOR` (§9.4.5) also live here.
- **Secret**: `CREATE_SECRET`, alone in its own category exactly as it was in
  `LogicalOperatorType`.
- **Appended physical-only specializations**: `RECURSIVE_KEY_JOIN` and
  `SECURE_VIEW`, then `MERGE_ACTION_SOURCE` — each with a header comment
  insisting it stay appended rather than sorted into its "natural" category,
  for the identical wire-format-stability reason Chapter 5 §5.3 explained for
  `LogicalOperatorType`'s numeric gaps: an enum value's ordinal is load-bearing
  somewhere downstream, and inserting a new value in the "obviously correct"
  alphabetical or categorical position would silently renumber everything
  after it.

## 9.3 Concrete operators, by source directory

`src/include/duckdb/execution/operator/` groups the concrete `PhysicalOperator`
subclasses into directories that mirror the enum's categories almost exactly.
This section walks each directory in turn, the same way Chapter 5 walked
`LogicalOperator`'s concrete subclasses one at a time — except here the
organizing question for every class is not "what relational concept does this
represent" (the logical layer's question) but **"what algorithm does this
implement, and under what conditions is it chosen over its siblings."**

### 9.3.1 `aggregate/` — four different answers to "compute a `GROUP BY`"

```
src/include/duckdb/execution/operator/aggregate/
  physical_hash_aggregate.hpp        PhysicalHashAggregate
  physical_perfecthash_aggregate.hpp PhysicalPerfectHashAggregate
  physical_partitioned_aggregate.hpp PhysicalPartitionedAggregate
  physical_ungrouped_aggregate.hpp   PhysicalUngroupedAggregate
  physical_limited_distinct.hpp      PhysicalLimitedDistinct
  physical_window.hpp                PhysicalWindow
  physical_streaming_window.hpp      PhysicalStreamingWindow
```

A single `LogicalAggregate` node (Chapter 5 §5.4.6) can turn into any of four
different physical classes, and the selection logic
(`src/execution/physical_plan/plan_aggregate.cpp`) is worth reading close to
verbatim, because it's a clean, self-contained example of exactly the kind of
decision this whole chapter is about — and because Chapter 14's full
hash-aggregation algorithm assumes you already know why you ended up looking
at *that* chapter's material in the first place.

**No groups at all → `PhysicalUngroupedAggregate`.** `SELECT SUM(x) FROM t`
with no `GROUP BY` is checked first and separately, because "compute one row
of aggregate state, no hashing, no grouping key at all" is a strictly simpler
problem than the general case and gets its own dedicated operator:

```cpp
// src/execution/physical_plan/plan_aggregate.cpp:278-284
if (op.groups.empty() && op.grouping_sets.size() <= 1) {
	// no groups: use the dedicated ungrouped aggregate path
	auto &group_by = Make<PhysicalUngroupedAggregate>(op.types, std::move(op.expressions), op.estimated_cardinality,
	                                                  op.distinct_validity);
	group_by.children.push_back(plan);
	return group_by;
}
```

**Groups, and the input is already partitioned by them →
`PhysicalPartitionedAggregate`.** This is the least intuitive of the four, and
worth being precise about because it's easy to mis-guess what triggers it —
it is *not* simply "running in parallel," despite Chapter 1's shallow preview
saying exactly that. The real trigger,
`CanUsePartitionedAggregate`/`HasSingleValuePartitions`
(`src/execution/physical_plan/plan_aggregate.cpp:32-45, 48-129`), walks back
up through the child plan (through `PhysicalProjection`/`PhysicalFilter`
nodes specifically) looking for a `PhysicalTableScan` whose data is already
guaranteed to be grouped by exactly the `GROUP BY` columns — the situation
that arises, for instance, scanning the state table of a recursive CTE that's
already partitioned by its own key columns. When that guarantee holds, no
aggregate needs to coordinate across partitions at all — each partition's
group-by columns take a single value, so a much cheaper, non-hashing
aggregation strategy applies. **This condition does not hold for Query A** —
the output of a hash join between `orders` and `lineitem` carries no such
guarantee about `custkey`'s distribution — so Query A never actually reaches
this operator; §9.7 explains where the boundary falls concretely.

**Groups, all small integer types, distinct-value range fits a bit budget →
`PhysicalPerfectHashAggregate`.** `CanUsePerfectHashAggregate`
(`src/execution/physical_plan/plan_aggregate.cpp:131-247`) is the most
elaborate of the four gates. It requires every `GROUP BY` column to be an
integer physical type (`INT8` through `UINT64` — line 140-153: anything else
returns `false` immediately), then, using column statistics if available or a
conservative type-range fallback for small types without stats
(lines 154-172), computes exactly how many bits are needed to address every
possible value in the observed min/max range (`RequiredBitsForValue`,
lines 18-25) — and bails out if that range exceeds `2^32` (line 224) or if the
running total of bits-per-group exceeds a configurable threshold,
`PerfectHtThresholdSetting` (line 234). When it fits, DuckDB can address every
group by a direct array index computed from the group values themselves —
"perfect" in the sense of a perfect hash function, no collisions, no probing
— rather than an actual hash table. `GROUP BY o.custkey` alone, with
`custkey` a `BIGINT` (`INT64`, satisfies line 144) and a plausible customer-key
range with statistics available, is a real, common candidate for this path in
practice; whether Query A actually lands here versus the general
`PhysicalHashAggregate` depends on whether the optimizer's
statistics-propagation pass (Chapter 7) attached usable min/max stats to
`o.custkey` by the time this check runs — §9.7 makes the concrete call for
Query A's worked example.

**Everything else → `PhysicalHashAggregate`,** the general-purpose fallback
every other case falls through to:

```cpp
// src/execution/physical_plan/plan_aggregate.cpp:306-311
auto &group_by = Make<PhysicalHashAggregate>(context, op.types, std::move(op.expressions), std::move(op.groups),
                                             std::move(op.grouping_sets), std::move(op.grouping_functions),
                                             op.estimated_cardinality, group_validity, op.distinct_validity);
group_by.children.push_back(plan);
return group_by;
```

It handles arbitrary group types, multiple grouping sets (`ROLLUP`/`CUBE`),
distinct aggregates, and everything else the three specialized paths above
explicitly excluded — the price for that generality is an actual hash table
over group keys, radix-partitioned across threads for parallelism. Chapter
14 is dedicated in full to how that hash table, and its partitioning and
combine/finalize protocol, actually works; this chapter's job stops at "here
is why the physical plan generator handed this particular query to this
particular class."

`PhysicalWindow`/`PhysicalStreamingWindow` (from `LogicalWindow`, Chapter 5
§5.4.7) and `PhysicalLimitedDistinct` round out the directory. Neither is part
of Query A's path.

`PhysicalLimitedDistinct` is easy to mistake for an optimizer fusion like
`LogicalTopN`. In fact no optimizer pass creates it. The *physical plan generator* creates it, in `plan_limit.cpp`, after
it has planned the child of a `LIMIT`. If that child is an aggregate with group
keys but no aggregate functions (a `SELECT DISTINCT`, or a `GROUP BY` with no
`SUM`/`COUNT`/...), and the limit is at most 100,000
(`LIMITED_DISTINCT_THRESHOLD`, `src/execution/physical_plan/plan_limit.cpp:186`),
the generator swaps the aggregate for a `PhysicalLimitedDistinct`. That operator
can stop reading input once it has found enough distinct groups. The `LIMIT`
node stays on top to apply the exact `LIMIT`/`OFFSET`.

This rewrite replaces a `PhysicalPerfectHashAggregate` as well as a
`PhysicalHashAggregate`. There is one
exception. A perfect-hash aggregate knows how many bits its keys need, so it
knows the most groups there can be: `2^total_bits`. If that number is not larger
than the limit, the limit can never cut the scan short, so the perfect hash
table is kept:

```cpp
// src/execution/physical_plan/plan_limit.cpp:105-119
static bool CanUseLimitedDistinct(const PhysicalPerfectHashAggregate &perfect_aggregate, idx_t limit) {
	if (perfect_aggregate.groups.empty() || !perfect_aggregate.aggregates.empty()) {
		return false;
	}
	// the group bounds cap the number of groups: when the limit cannot be reached a scan that stops early gains
	// nothing over the perfect hash table
	idx_t total_bits = 0;
	for (const auto &bits : perfect_aggregate.required_bits) {
		total_bits += bits;
	}
	if (total_bits < sizeof(idx_t) * 8 && (idx_t(1) << total_bits) <= limit) {
		return false;
	}
	return true;
}
```

`test/sql/aggregate/distinct/distinct_limit.test` shows both sides:

```sql
EXPLAIN SELECT i % 8 FROM range(1000000) t(i) GROUP BY 1 LIMIT 100;
-- PERFECT_HASH_GROUP_BY: the key needs 4 bits, so at most 2^4 = 16 groups
-- can exist, and LIMIT 100 is never reached
EXPLAIN SELECT i % 8 FROM range(1000000) t(i) GROUP BY 1 LIMIT 5;
-- LIMITED_DISTINCT: 16 > 5, so it can stop after 5 groups
```

### 9.3.2 `join/` — the deepest hierarchy in the physical layer

```
                         PhysicalOperator
                                │
                        CachingPhysicalOperator  (§9.3.5)
                                │
                          PhysicalJoin  ── PhysicalCrossProduct
                                │              (also Caching-derived,
                    ┌───────────┼───────────┐   but not Join-derived)
                    │           │           │
         PhysicalComparisonJoin │  PhysicalBlockwiseNLJoin
                    │           │
      ┌─────────────┼───────────────────────┐
      │             │                       │
PhysicalHashJoin  PhysicalRangeJoin   PhysicalAsOfJoin
                    │
        ┌───────────┴───────────┐
        │                       │
PhysicalPiecewiseMergeJoin  PhysicalIEJoin
```

```cpp
// src/include/duckdb/execution/operator/join/physical_join.hpp:17-51 (elided: method bodies)
//! PhysicalJoin represents the base class of the join operators
class PhysicalJoin : public CachingPhysicalOperator {
public:
	static constexpr const PhysicalOperatorType TYPE = PhysicalOperatorType::INVALID;

public:
	PhysicalJoin(PhysicalPlan &physical_plan, LogicalOperator &op, PhysicalOperatorType type, JoinType join_type,
	             idx_t estimated_cardinality);

	JoinType join_type;
	// ...
};
```

```cpp
// src/include/duckdb/execution/operator/join/physical_comparison_join.hpp:20-33
//! PhysicalJoin represents the base class of the join operators
class PhysicalComparisonJoin : public PhysicalJoin {
public:
	PhysicalComparisonJoin(PhysicalPlan &physical_plan, LogicalOperator &op, PhysicalOperatorType type,
	                       vector<JoinCondition> cond, JoinType join_type, idx_t estimated_cardinality);

	vector<JoinCondition> conditions;
	//! Non-comparison predicate for additional filtering (can be nullptr)
	unique_ptr<Expression> predicate;
	//! Scans where we should push generated filters into (if any)
	unique_ptr<JoinFilterPushdownInfo> filter_pushdown;
	// ...
};
```

Every join with a `JoinCondition`-list representation — hash-joinable
equalities, range comparisons, or a mix — is a `PhysicalComparisonJoin`
subclass, carrying that condition list forward unchanged from
`LogicalComparisonJoin` (Chapter 5 §5.4.4 introduced the same
`JoinCondition{left, right, comparison}` shape at the logical level).
`PhysicalHashJoin` handles the equality case; `PhysicalRangeJoin` is the
shared base for the two range-comparison strategies,
`PhysicalPiecewiseMergeJoin` (sort both sides, merge) and `PhysicalIEJoin`
(the inequality-join algorithm for two-or-more range predicates, e.g.
`BETWEEN`-style joins on two columns at once); `PhysicalAsOfJoin` gets its own
class specifically for the "most-recent-at-or-before" semantics Chapter 5
§5.4.4 introduced — recall that at the *logical* level, `ASOF JOIN` needed no
dedicated subclass (it was an ordinary `LogicalComparisonJoin` with a special
enum tag); at the *physical* level it does get one, because "most recent
match" genuinely is a distinct execution algorithm, not just a distinct
condition shape. `PhysicalBlockwiseNLJoin` sits directly under `PhysicalJoin`
rather than `PhysicalComparisonJoin`, because it exists precisely for
conditions that *aren't* representable as a clean `JoinCondition` list — the
physical-layer analogue of `LogicalAnyJoin` (Chapter 5's fallback for
`ON a.x + a.y > b.z`-shaped conditions).

`PhysicalCrossProduct` (unconditional, every row against every row) derives
from `CachingPhysicalOperator` directly, alongside `PhysicalPositionalJoin`
(row *i* against row *i*, DuckDB's `POSITIONAL JOIN` — Chapter 5 §5.4.4's
`LogicalUnconditionalJoin` pairing carried straight through to the physical
layer).

The **Delim-join trio** — `PhysicalDelimJoin`, `PhysicalLeftDelimJoin`,
`PhysicalRightDelimJoin` — is the physical counterpart of Chapter 6's entire
decorrelation machinery. Recall `LogicalDependentJoin` (Chapter 5 §5.4.5) is a
deliberately temporary marker, gone by the time the optimizer's first pass
even runs, rewritten by `FlattenDependentJoins` into an ordinary join tree.

What that tree looks like depends on the setting `delim_join_as_cte`. It
defaults to `true` (`src/common/settings.json`; the setting is marked
deprecated). With it on, `FlattenDependentJoins::DecorrelateIndependent`
(`src/planner/subquery/flatten_dependent_join.cpp:281-292`) calls
`DelimJoinCTERewriter::Rewrite` right after flattening. That step turns every
delim join into plain joins over two materialized CTEs (`__duckdb_delim_<n>`
and `__duckdb_delim_dedup_<n>`), and `VerifyNoDelim`
(`src/planner/subquery/delim_join_cte_rewriter.cpp:33-45`) checks that no
`LOGICAL_DELIM_JOIN` or `LOGICAL_DELIM_GET` is left (Chapter 6 §6.4.1). So with
default settings the physical planner never calls `PlanDelimJoin`. It builds a
`PhysicalCTE` for each materialized CTE, `CTE_SCAN`s to read it back, a hash
aggregate for the duplicate elimination, and ordinary hash joins.

With `SET delim_join_as_cte = false`, the tree is built around a delim join: a
`LogicalComparisonJoin` whose type is `LOGICAL_DELIM_JOIN` (there is no
`LogicalDelimJoin` class). `PhysicalDelimJoin` is what that logical node
becomes. The delim-join trio below is what you get only with this setting off.

The class:

```cpp
// src/include/duckdb/execution/operator/join/physical_delim_join.hpp:17-28
//! PhysicalDelimJoin represents a join where either the LHS or RHS will be duplicate eliminated and pushed into a
//! PhysicalColumnDataScan in the other side. Implementations are PhysicalLeftDelimJoin and PhysicalRightDelimJoin
class PhysicalDelimJoin : public PhysicalOperator {
public:
	PhysicalDelimJoin(PhysicalPlan &physical_plan, PhysicalOperatorType type, vector<LogicalType> types,
	                  PhysicalOperator &original_join, PhysicalOperator &distinct,
	                  const vector<const_reference<PhysicalOperator>> &delim_scans, idx_t estimated_cardinality,
	                  optional_idx delim_idx);

	PhysicalOperator &join;
	PhysicalHashAggregate &distinct;
	vector<const_reference<PhysicalOperator>> delim_scans;
	// ...
};
```

Its own doc comment states the mechanism in one line: one side gets
duplicate-eliminated (via a `PhysicalHashAggregate`, the `distinct` field
above — notice it's a concrete `PhysicalHashAggregate&`, not a generic
`PhysicalOperator&`, because the delim-join protocol specifically needs
aggregate machinery) and the deduplicated result is pushed into
`PhysicalColumnDataScan`s (`delim_scans`) that get read back on the other
side of the plan — this is precisely the mechanical trick Chapter 6 describes
Neumann and Kemper's algorithm [12] using to avoid re-evaluating a correlated
subquery once per outer row: materialize the distinct correlated values once,
then join against that materialization instead. `delim_scans` being a
`vector<const_reference<PhysicalOperator>>` rather than a single reference is
itself a small, concrete illustration of §9.1.1's point about
reference-wrapped, arena-shared physical operators — the same underlying
`PhysicalColumnDataScan` object can be, and typically is, pointed to from more
than one place in the final tree.

`JoinFilterPushdown`, mentioned in the research material as a shared support
object, is what lets a completed hash join's build side hand a *dynamic*
filter (computed from the actual values seen, not something the static
optimizer could know in advance) back down into the probe side's scan — worth
knowing exists, out of scope to trace further here.

### 9.3.3 `scan/` — `PhysicalTableScan` and its specialized siblings

`PhysicalTableScan` is the workhorse: the physical realization of an ordinary
`LogicalGet` over a real table (or, per Chapter 5 §5.4.1's point that
`LogicalGet` is DuckDB's one general "data source" node, over a table
function like `read_csv`/`read_parquet`/`range` just as easily). It is what
actually calls into the storage layer (Chapters 14–18) to pull `DataChunk`s
out of row groups, applying whatever `table_filters` the optimizer's
filter-pushdown pass attached to the originating `LogicalGet` — this is the
scan that ends up at the leaves of Query A's physical tree in §9.7.

The rest of the directory is specialized, non-table sources for situations
where "read from persistent storage" isn't the right description at all:
`PhysicalColumnDataScan` reads back an in-memory `ColumnDataCollection` — used
by the Delim-join machinery above, by materialized CTEs, and by
`EXPLAIN`/prepared-statement plumbing that needs to hand a pre-computed result
set back into a plan as if it were a fresh source. `PhysicalExpressionScan`
evaluates a literal `VALUES` list (or similar constant-expression source)
directly, with no underlying storage at all. `PhysicalPositionalScan` is the
data-source half of `POSITIONAL JOIN`'s row-by-row-position pairing.
`PhysicalEmptyResult` is the physical realization of `LogicalEmptyResult`
(Chapter 5 — a subtree the optimizer proved statically can never produce any
rows, e.g. a `WHERE FALSE`) and `PhysicalDummyScan` produces exactly one
row with zero columns, the trick that lets a query with no `FROM` clause at
all (`SELECT 1 + 1`) still be represented as a tree with a source at its
leaves rather than needing a special no-source case threaded through every
other operator.

### 9.3.4 `order/` — `PhysicalOrder` vs. `PhysicalTopN`

Directly parallel to Chapter 5 §5.4.9's logical-layer pair.
`PhysicalOrder` performs an actual full sort — every row materialized,
compared, and reordered — the physical realization of a bare `LogicalOrder`
with no accompanying `LIMIT`. `PhysicalTopN` is the physical realization of
`LogicalTopN` (itself, recall, entirely a creation of the optimizer's
pattern-matching, never the binder) and uses the bounded min/max-heap
strategy its logical counterpart's doc comment already promised — holding at
most `limit` elements in memory regardless of how many rows flow through it.
Query A's `ORDER BY total DESC LIMIT 10` reaches `PhysicalTopN` directly,
never `PhysicalOrder` — the fusion already happened one stage earlier, in the
optimizer, so by the time `CreatePlanInternal`'s switch (§9.4.4) sees this
part of the tree, there is only ever one `LogicalTopN` node to translate, not
a `LogicalOrder`/`LogicalLimit` pair to somehow recognize and fuse itself.

### 9.3.5 `filter/` — `PhysicalFilter` and why it's a `CachingPhysicalOperator`

`PhysicalFilter` is the direct physical realization of `LogicalFilter`
(Chapter 5 §5.4.2) — evaluate a predicate against each incoming row, pass
matching rows through, drop the rest. What's worth explaining is *what*
`CachingPhysicalOperator` (§9.1.1 already named it as `PhysicalJoin`'s base
too) actually buys a filter operator specifically:

```cpp
// src/include/duckdb/execution/physical_operator.hpp:324-340 (elided: private members, FinalExecute body)
class CachingPhysicalOperator : public PhysicalOperator {
public:
	static constexpr const idx_t CACHE_THRESHOLD = 64;
	CachingPhysicalOperator(PhysicalPlan &physical_plan, PhysicalOperatorType type, vector<LogicalType> types,
	                        idx_t estimated_cardinality);

	bool caching_supported;

public:
	//! This Execute will prevent small chunks from entering the pipeline, buffering them until a bigger chunk is
	//! created.
	OperatorResultType Execute(ExecutionContext &context, DataChunk &input, DataChunk &chunk,
	                           GlobalOperatorState &gstate, OperatorState &state) const final;
	//! FinalExecute is used here to send out the remainder of the chunk (< STANDARD_VECTOR_SIZE) that we still had
	//! cached.
	OperatorFinalizeResultType FinalExecute(ExecutionContext &context, DataChunk &chunk, GlobalOperatorState &gstate,
	                                        OperatorState &state) const final;
	// ...
};
```

A highly selective `WHERE` clause is exactly the situation where this
matters: if 2048 rows come in and only 30 pass the predicate, pushing a
30-row chunk straight downstream to the next operator (an aggregate's hash
probe, a join's probe side) means every downstream operator pays its
per-chunk fixed overhead — resetting SIMD-friendly loop state, dispatching
through vectorized function pointers, updating profiling counters — for a
tiny amount of actual work. `CachingPhysicalOperator`'s `Execute` buffers
those small results instead of forwarding them immediately, only releasing a
chunk downstream once it's accumulated past `CACHE_THRESHOLD` (64) rows or
the operator runs out of input (`FinalExecute` flushes whatever's left).
`PhysicalFilter` inherits this for free; a `PhysicalHashJoin`'s probe side
gets the identical benefit through `PhysicalJoin`'s own inheritance from the
same base.

### 9.3.6 `projection/` — `PhysicalProjection` and its siblings

`PhysicalProjection` is the direct realization of `LogicalProjection`
(Chapter 5 §5.4.3) — evaluate each output expression against the input chunk,
column by column. `PhysicalUnnest` and `PhysicalPivot` realize `LogicalUnnest`
(§5.4.8) and the `PIVOT` construct respectively. `PhysicalTableInOutFunction`
is the physical side of a table function that both consumes rows and
produces them (as opposed to `LogicalGet`'s producer-only table functions) —
this directory is generally the least algorithmically interesting in the
whole physical layer, in the same sense Chapter 5 noted about the logical
`LogicalProjection`: it computes exactly what its expressions say, with no
strategic choice comparable to picking a join algorithm.

### 9.3.7 `persistent/` — DML: `PhysicalInsert`/`Update`/`Delete`/`MergeInto`

Mirrors Chapter 5 §5.4.13's `LogicalInsert`/`LogicalUpdate`/`LogicalDelete`/
`LogicalMergeInto` closely, with one genuinely physical-layer decision worth
naming: `PhysicalInsert` versus `PhysicalBatchInsert`. A large `INSERT ...
SELECT` benefits from committing rows to storage in parallel, batched
independently per worker thread rather than serialized through one thread —
`PhysicalBatchInsert` is that strategy, chosen (via
`PhysicalPlanGenerator::UseBatchIndex`, declared alongside `CreatePlan` in
`physical_plan_generator.hpp:111`) based on whether the source plan can
produce a stable per-row *batch index* to keep concurrently-inserted batches
correctly orderable. `PhysicalCopyToFile`/`PhysicalBatchCopyToFile` is the
same ordinary-versus-batched split applied to `COPY ... TO`.

`PhysicalCopyToFile` is the largest operator in this directory. Most of its
code is for *partitioned* copies (`COPY ... TO ... (PARTITION_BY ...)`). Input
rows are split into partitions either through sorted bins or, for integer keys
with known bounds, directly by key value. A helper class, `CopyBatchSlicer`
(`src/include/duckdb/execution/operator/persistent/copy_batch_slicer.hpp`),
measures row sizes so that a byte limit such as `ROW_GROUP_SIZE_BYTES` can cut
a chunk into slices that fit. Output files can rotate on either
`FILE_SIZE_BYTES` or `ROW_GROUPS_PER_FILE`, and `COPY TO` has a
`RETURN_COLUMN_TYPES` option. The generator also has a public overload,
`CreatePlan(LogicalCopyToFile &op, PhysicalOperator &plan)`
(`physical_plan_generator.hpp:131-132`), which plans a copy on top of a child
plan that is already built. This chapter does not trace the partitioned copy
further.

`PhysicalMergeInto`
and its companion `PhysicalMergeActionSource` (the latter tagged
`MERGE_ACTION_SOURCE` in §9.2's appended-values block) implement `MERGE INTO`'s
per-branch `WHEN MATCHED`/`WHEN NOT MATCHED` action dispatch.

### 9.3.8 `schema/` — DDL

`PhysicalCreateTable`, `PhysicalCreateIndex`, `PhysicalCreateView`,
`PhysicalAlter`, `PhysicalDrop`, `PhysicalAttach`/`PhysicalDetach`, and their
several `CreateSchema`/`CreateSequence`/`CreateType`/`CreateTrigger` siblings
carry the same already-bound `CreateInfo`/`AlterInfo`/etc. structs their
logical counterparts held (Chapter 5 §5.4.14), essentially unchanged, through
to execution. As at the logical layer, there is minimal algorithmic choice
here for the physical planner to make — one paragraph is proportionate to how
much is actually happening in this directory.

### 9.3.9 `set/` — `PhysicalUnion`, `PhysicalRecursiveCTE`, `PhysicalCTE`

`PhysicalUnion` realizes `LogicalSetOperation` (Chapter 5 §5.4.11) — recall
that one logical class backs `UNION`/`EXCEPT`/`INTERSECT` all three; by the
physical layer, though, only `UNION` gets its own dedicated class here
(`EXCEPT`/`INTERSECT` are planned via different, more specialized machinery
not covered by this directory's class list). `PhysicalRecursiveCTE` realizes
`LogicalRecursiveCTE` (Chapter 5 §5.4.12)'s fixed-point iteration directly,
with `PhysicalRecursiveCTEStateScan : public PhysicalColumnDataScan` as the
mechanism for reading back the previous iteration's working set — a second,
concrete example (after the Delim-join trio) of a physical-only subclass
composing an existing scan operator rather than reimplementing scanning from
scratch. `PhysicalCTE`/`PhysicalCTEConsumerSource` do the analogous job for
`LogicalMaterializedCTE`.

### 9.3.10 `helper/` — where "the answer" actually comes from

This is the most heterogeneous directory, but one member deserves to be
called out above all the others, because it answers a question this book has
implicitly deferred since Chapter 1: after all of this — parsing, binding,
optimizing, physical planning, executing pipelines full of hash joins and
aggregates — *where do the final rows a client actually receives come from?*

```cpp
// src/include/duckdb/execution/operator/helper/physical_result_collector.hpp:19-42 (elided: further methods)
//! PhysicalResultCollector is an abstract class that is used to generate the final result of a query
class PhysicalResultCollector : public PhysicalOperator {
public:
	static constexpr const PhysicalOperatorType TYPE = PhysicalOperatorType::RESULT_COLLECTOR;

public:
	PhysicalResultCollector(PhysicalPlan &physical_plan, PreparedStatementData &data);

	StatementType statement_type;
	StatementProperties properties;
	PhysicalOperator &plan;
	vector<Identifier> names;

public:
	static unique_ptr<PhysicalOperator> GetResultCollector(ClientContext &context, PreparedStatementData &data);

public:
	//! The final method used to fetch the query result from this operator
	virtual unique_ptr<QueryResult> GetResult(GlobalSinkState &state) const = 0;

	bool IsSink() const override {
		return true;
	}
	// ...
};
```

This is the answer: `PhysicalResultCollector` is always the root of the
physical tree (§9.4.3 shows exactly where `PlanInternal` sets it as such), it
is a *sink* (`IsSink() const override { return true; }` — Chapter 11 covers
what being a sink means mechanically), and `GetResult` is, quite literally,
the method whose return value becomes the client's `QueryResult`.

The class is abstract, but the built-in engine has only **one** concrete
subclass: `PhysicalResultSink`. There is no separate batch collector; batch
ordering is one of its modes. The factory `GetResultCollector` always builds
one. Only its `ResultOrdering` argument changes:

```cpp
// src/execution/operator/helper/physical_result_collector.cpp:21-33
unique_ptr<PhysicalOperator> PhysicalResultCollector::GetResultCollector(ClientContext &context,
                                                                         PreparedStatementData &data) {
	auto &physical_plan = *data.physical_plan;
	auto &root = physical_plan.Root();

	if (!PhysicalPlanGenerator::PreserveInsertionOrder(context, root)) {
		return make_uniq<PhysicalResultSink>(physical_plan, data, ResultOrdering::UNORDERED);
	}
	if (!PhysicalPlanGenerator::UseBatchIndex(context, root)) {
		return make_uniq<PhysicalResultSink>(physical_plan, data, ResultOrdering::SOURCE_ORDERED);
	}
	return make_uniq<PhysicalResultSink>(physical_plan, data, ResultOrdering::BATCH_INDEX_ORDERED);
}
```

The three orderings (`src/include/duckdb/common/enums/result_ordering.hpp:16-23`)
say how the client sees rows in the right order:

- `UNORDERED`: the query does not need to keep insertion order, so many
  threads add chunks in whatever order they finish.
- `SOURCE_ORDERED`: order matters, but the source cannot number its chunks, so
  one thread adds all chunks in source order.
- `BATCH_INDEX_ORDERED`: order matters, and the source gives every chunk a
  *batch index* (a sequence number). Many threads add chunks, and the sink puts
  them back in order by batch index.

Whether the result is streamed or kept in full is not fixed in the plan.
The sink writes into a buffer. The client's first call decides the buffer's
`ResultLifetime`: a fetch makes it `DRAINING` (a bounded buffer that the client
reads once), and a request for the full result makes it `RETAINED`
(`src/include/duckdb/common/enums/result_lifetime.hpp:14-24`). The sink also
packs rows into "units" of a pluggable `ResultFormat`
(`src/include/duckdb/main/result_format.hpp:77`), so the same sink can build,
for example, ordinary `DataChunk` results or Arrow batches. Memory handling
lives in the result format too, not in a field of `PhysicalResultCollector`.

Every one of this book's `EXPLAIN`-style tree diagrams that ends in
"`PhysicalResultCollector` — root, gathers final output" (Chapter 0's
convention-setting example, and Chapter 1's Query A preview) is pointing at
exactly this class.

The rest of the directory is the "everything else" bucket: `PhysicalLimit`/
`PhysicalStreamingLimit`/`PhysicalLimitPercent` (a bare `LIMIT` with no
`ORDER BY` to fuse with — no need for `PhysicalTopN`'s heap, just "stop after
N rows," with the streaming variant able to short-circuit its upstream
pipeline entirely once N rows are seen), `PhysicalReservoirSample` (the
statistical reservoir-sampling algorithm behind `USING SAMPLE`),
`PhysicalExplainAnalyze` (wraps a real physical tree so that timing and row
counts get collected per-operator during an actual run, for `EXPLAIN
ANALYZE`'s output), and the substantial cluster of session/statement plumbing
operators (`PhysicalVacuum`, `PhysicalPragma`, `PhysicalSet`/
`PhysicalSetVariable`, `PhysicalReset`, `PhysicalPrepare`, `PhysicalExecute`,
`PhysicalTransaction`, `PhysicalConnect`/`PhysicalDisconnect`,
`PhysicalCreateSecret`, `PhysicalSecureView`, `PhysicalExternalResource`).

Last, `PhysicalVerifyVector` — a debug-only operator, never present in a
normal query's physical plan, covered on its own in §9.4.5 because it's
inserted by the physical plan generator itself rather than dispatched from
any `LogicalOperatorType` at all.

## 9.4 `PhysicalPlanGenerator`: the control flow, stage by stage

Everything above described the *vocabulary* physical plans are built from.
This section covers the *machine* that does the building — `PhysicalPlanGenerator`
(`src/include/duckdb/execution/physical_plan_generator.hpp`,
`src/execution/physical_plan_generator.cpp`) — reading its control flow in the
order it actually executes.

```cpp
// src/execution/physical_plan_generator.cpp:24-28
unique_ptr<PhysicalPlan> PhysicalPlanGenerator::Plan(unique_ptr<LogicalOperator> op) {
	auto &plan = ResolveAndPlan(std::move(op));
	plan.Verify();
	return std::move(physical_plan);
}
```

`Plan` is the single public entry point — one optimized `LogicalOperator`
tree in, one `PhysicalPlan` (§9.4.3) out, with a `Verify()` sanity check on
the resulting tree before it's handed back. Everything interesting happens
inside `ResolveAndPlan`:

```cpp
// src/execution/physical_plan_generator.cpp:30-54
PhysicalOperator &PhysicalPlanGenerator::ResolveAndPlan(unique_ptr<LogicalOperator> op) {
	auto &profiler = QueryProfiler::Get(context);

	// Resolve the types of each operator.
	{
		auto timer = profiler.StartTimer<MetricPhysicalPlannerResolveTypes>();
		op->ResolveOperatorTypes();
	}

	// Resolve the column references.
	{
		auto timer = profiler.StartTimer<MetricPhysicalPlannerColumnBinding>();
		ColumnBindingResolver resolver;
		resolver.VisitOperator(*op);
	}

	// Create the main physical plan.
	{
		auto timer = profiler.StartTimer<MetricPhysicalPlannerCreatePlan>();
		physical_plan = PlanInternal(*op);
	}

	// Return a reference to the root of this plan.
	return physical_plan->Root();
}
```

Three profiled stages, strictly in order, each individually timed (these
timer names — `MetricPhysicalPlannerResolveTypes`,
`MetricPhysicalPlannerColumnBinding`, `MetricPhysicalPlannerCreatePlan` — are
exactly what shows up broken out in `EXPLAIN ANALYZE`'s planner-time
section, if you've ever wondered where those three line items come from).

### 9.4.1 Stage (a): `ResolveOperatorTypes()`

The very first thing that happens is the same bottom-up type resolution
Chapter 5 §5.2.2 already covered in full for the logical tree — called again,
here, on the *already-optimized* tree, because the optimizer's rewrites
(pushing filters, choosing a join order, fusing `LogicalOrder`+`LogicalLimit`
into `LogicalTopN`) can leave some operators' `types` stale or unset. This
pass re-derives every operator's output schema, children first, bottom-up,
exactly per `LogicalOperator::ResolveOperatorTypes`'s recursive definition
(`src/planner/logical_operator.cpp:133-141`, quoted in full in Chapter 5).
Nothing in this stage looks at column *bindings* at all — only *types* — which
is precisely why it has to run before the next stage rather than after or
alongside it: the next stage needs correct types available to sanity-check
against as it rewrites bindings.

### 9.4.2 Stage (b): `ColumnBindingResolver` — the key moment

This is the single most important transformation in this chapter, and
Chapter 5 §5.5.3 already introduced it as "the bridge into Chapter 9" — this
is that promise redeemed. Its job, restated precisely: convert every
`BoundColumnRefExpression`'s `ColumnBinding{table_index, column_index}` — the
addressing scheme every logical operator has used since Chapter 4's binder
first assigned it — into a flat, positional `BoundReferenceExpression{index}`,
where `index` means nothing more than "how far into the incoming `DataChunk`'s
column list is this value."

*Why this has to happen* is worth restating in this chapter's terms, because
it's the cleanest possible justification for everything §9.1 said about
`PhysicalOperator` carrying no addressing infrastructure of its own:
**physical operators do not carry table indices.** A `PhysicalHashJoin` or
`PhysicalFilter`, at execution time, is handed a `DataChunk` — an anonymous,
flat vector of columns with no memory anywhere in it of "table 3, column 1."
It only knows "evaluate my expression against column *N* of whatever chunk
arrived." A `PhysicalProjection` genuinely has no way to know, and no need to
know, that the value in its input chunk's third column originally came from
`lineitem.extendedprice` rather than some entirely different table three
joins upstream — by the time it runs, that provenance information has already
done its job (letting the binder and optimizer reason about the query
correctly) and has no further use. Carrying it forward into execution would
mean re-resolving a table/column pair to a chunk offset on every single row,
of every single chunk, for every single expression evaluation — a hash
lookup or linear scan repeated billions of times over a large query, when the
answer is exactly the same integer every single time. `ColumnBindingResolver`
does that resolution work *once per expression*, at the plan/physical
boundary, so execution only ever pays for an array index.

The mechanism (`src/execution/column_binding_resolver.cpp`) is a
`LogicalOperatorVisitor` subclass (Chapter 5 §5.5.1) that walks the tree
maintaining one piece of running state — `bindings`, the currently-in-scope
ordered list of `ColumnBinding`s a downstream expression might reference —
and rewrites every `BoundColumnRefExpression` it finds by locating that exact
binding's position in the current list:

```cpp
// src/execution/column_binding_resolver.cpp:243-278 (elided: error-path assertions)
unique_ptr<Expression> ColumnBindingResolver::VisitReplace(BoundColumnRefExpression &expr,
                                                           unique_ptr<Expression> *expr_ptr) {
	D_ASSERT(expr.Depth() == 0);
	// check the current set of column bindings to see which index corresponds to the column reference
	for (idx_t i = 0; i < bindings.size(); i++) {
		if (expr.Binding() == bindings[i]) {
			// ...
			if (verify_only) {
				return nullptr;
			}
			return make_uniq<BoundReferenceExpression>(expr.GetAlias(), expr.GetReturnType(), i);
		}
	}
	// ...
}
```

Most operator kinds fall through to a "general case" — visit children first
(so their own `bindings` context is whatever *they* need), then resolve this
operator's own expressions against the *child's* output bindings, then update
`bindings` to *this* operator's own output for whatever comes next:

```cpp
// src/execution/column_binding_resolver.cpp:233-240
// general case
// first visit the children of this operator
VisitOperatorChildren(op);
// now visit the expressions of this operator to resolve any bound column references
VisitOperatorExpressions(op);
// finally update the current set of bindings to the current set of column bindings
bindings = op.GetColumnBindings();
types = op.types;
```

A binary join is the one case that needs real, hand-written logic rather than
the general fallback, because a join condition legitimately needs to resolve
expressions against *each side's own local bindings* before the two sides'
column lists get concatenated into one combined output:

```cpp
// src/execution/column_binding_resolver.cpp:38-82 (elided: LOGICAL_DELIM_JOIN's near-identical sibling case)
case LogicalOperatorType::LOGICAL_ASOF_JOIN:
case LogicalOperatorType::LOGICAL_COMPARISON_JOIN: {
	auto &comp_join = op.Cast<LogicalComparisonJoin>();

	VisitOperator(*comp_join.children[0]);
	auto left_bindings = bindings;
	auto left_types = types;
	for (auto &cond : comp_join.conditions) {
		if (cond.IsComparison()) {
			VisitExpression(&cond.LeftReference());
		}
	}
	// ...
	VisitOperator(*comp_join.children[1]);
	auto right_bindings = bindings;
	auto right_types = types;
	for (auto &cond : comp_join.conditions) {
		if (cond.IsComparison()) {
			VisitExpression(&cond.RightReference());
		}
	}

	// combine bindings to resolve predicate
	auto combined_bindings = left_bindings;
	combined_bindings.insert(combined_bindings.end(), right_bindings.begin(), right_bindings.end());
	// ...
	bindings = combined_bindings;
	// ... (non-comparison predicate, if any, resolved against combined_bindings here)

	// update to join output bindings
	bindings = op.GetColumnBindings();
	types = op.types;
	return;
}
```

Read this precisely once and the whole "coordinate-system change" framing
Chapter 5 used becomes literal: the *left* child's `orderkey` reference
resolves against a binding list containing only the left child's own output
columns (position within that list alone), the *right* child's `orderkey`
reference resolves against only the right child's own columns, and only
*after* both are separately resolved do the two lists get concatenated —
left's columns first, right's columns appended after — to become the binding
context anything *above* the join sees. That concatenation order is exactly
why a right-side column reference, once resolved, ends up with an index
offset by however many columns the left side contributed: "column 2 of the
combined chunk" is genuinely "column 0 of whatever the right child emits,"
shifted by the left child's column count. §9.6's worked trace makes this
concrete with Query A's actual numbers.

### 9.4.3 Stage (c): `PlanInternal` and the `PhysicalPlan` arena

```cpp
// src/execution/physical_plan_generator.cpp:56-73
unique_ptr<PhysicalPlan> PhysicalPlanGenerator::PlanInternal(LogicalOperator &op) {
	if (!physical_plan) {
		physical_plan = make_uniq<PhysicalPlan>(Allocator::Get(context));
	}
	op.estimated_cardinality = op.EstimateCardinality(context);
	physical_plan->SetRoot(CreatePlan(op));
	physical_plan->Root().estimated_cardinality = op.estimated_cardinality;

	auto debug_verify_vector = Settings::Get<DebugVerifyVectorSetting>(context);
	if (debug_verify_vector != DebugVectorVerification::NONE) {
		if (debug_verify_vector != DebugVectorVerification::DICTIONARY_EXPRESSION &&
		    debug_verify_vector != DebugVectorVerification::VARIANT_VECTOR &&
		    debug_verify_vector != DebugVectorVerification::SHREDDED_VECTOR) {
			physical_plan->SetRoot(Make<PhysicalVerifyVector>(physical_plan->Root(), debug_verify_vector));
		}
	}
	return std::move(physical_plan);
}
```

This is where §9.1.1's arena claim becomes concrete. `PhysicalPlan` itself:

```cpp
// src/include/duckdb/execution/physical_plan_generator.hpp:38-78 (elided: destructor loop body shown, comments retained)
class PhysicalPlan {
public:
	explicit PhysicalPlan(Allocator &allocator) : arena(allocator) {};
	// ...

public:
	template <class T, class... ARGS>
	PhysicalOperator &Make(ARGS &&... args) {
		static_assert(std::is_base_of<PhysicalOperator, T>::value, "T must be a physical operator");
		auto ptr = arena.Make<T>(*this, std::forward<ARGS>(args)...);
		ops.push_back(*ptr);
		return *ptr;
	}

	PhysicalOperator &Root() {
		D_ASSERT(root);
		return *root;
	}
	void SetRoot(PhysicalOperator &op) {
		root = op;
	}
	// ...

private:
	//! The arena allocator storing the physical operator memory.
	ArenaAllocator arena;
	//! References to the physical operators.
	vector<reference<PhysicalOperator>> ops;
	//! The root of the physical plan.
	optional_ptr<PhysicalOperator> root;
};
```

Every single concrete `PhysicalOperator` this chapter has named —
`PhysicalHashJoin`, `PhysicalHashAggregate`, `PhysicalTableScan`, all of them
— comes into existence through this one `Make<T>` template method, which
allocates the object directly out of one `ArenaAllocator` shared by the whole
plan and keeps a bookkeeping reference to it in `ops` (used, among other
things, by the destructor to call each operator's destructor explicitly,
since arena memory doesn't run destructors on its own deallocation). There is
no `new PhysicalHashJoin(...)` anywhere in this codebase's physical planning
path — every construction site funnels through `Make`, which is exactly the
mechanism that makes §9.1.1's "operators live in a `PhysicalPlan` arena
instead" claim literally true rather than just an ownership-model metaphor.

Two housekeeping details in `PlanInternal` worth a sentence each: the
`estimated_cardinality` is explicitly copied from the logical root onto the
physical root, a second time, immediately after `CreatePlan` — because
`CreatePlan(op)`'s recursive descent may have already consumed and
reassigned that field on intermediate nodes, so the *root's* final cardinality
gets re-stamped once, at the very end, to guarantee it reflects the top-level
estimate rather than whatever an inner recursive call left behind. And the
`debug_verify_vector` check at the end is §9.4.5's subject, covered next.

### 9.4.4 Stage (d): `CreatePlanInternal`'s switch — dispatch to the `plan_*.cpp` files

```cpp
// src/execution/physical_plan_generator.cpp:75-82
PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalOperator &op) {
	const auto repeatable = ClassifyLogicalOperatorRepeatability(op) == LogicalOperatorRepeatability::REPEATABLE;
	auto &result = CreatePlanInternal(op);
	if (!repeatable) {
		non_repeatable_operators.insert(result);
	}
	return result;
}
```

`CreatePlan` itself is a thin wrapper — the actual recursive descent is
`CreatePlanInternal`'s big switch on `op.type`, dispatching to one C++
function overload per `LogicalOperatorType`:

```cpp
// src/execution/physical_plan_generator.cpp:84-119 (elided: continues for every LogicalOperatorType)
PhysicalOperator &PhysicalPlanGenerator::CreatePlanInternal(LogicalOperator &op) {
	switch (op.type) {
	case LogicalOperatorType::LOGICAL_GET:
		return CreatePlan(op.Cast<LogicalGet>());
	case LogicalOperatorType::LOGICAL_PROJECTION:
		return CreatePlan(op.Cast<LogicalProjection>());
	case LogicalOperatorType::LOGICAL_EMPTY_RESULT:
		return CreatePlan(op.Cast<LogicalEmptyResult>());
	case LogicalOperatorType::LOGICAL_FILTER:
		return CreatePlan(op.Cast<LogicalFilter>());
	case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
		return CreatePlan(op.Cast<LogicalAggregate>());
	case LogicalOperatorType::LOGICAL_WINDOW:
		return CreatePlan(op.Cast<LogicalWindow>());
	case LogicalOperatorType::LOGICAL_UNNEST:
		return CreatePlan(op.Cast<LogicalUnnest>());
	case LogicalOperatorType::LOGICAL_LIMIT:
		return CreatePlan(op.Cast<LogicalLimit>());
	case LogicalOperatorType::LOGICAL_SAMPLE:
		return CreatePlan(op.Cast<LogicalSample>());
	case LogicalOperatorType::LOGICAL_ORDER_BY:
		return CreatePlan(op.Cast<LogicalOrder>());
	case LogicalOperatorType::LOGICAL_TOP_N:
		return CreatePlan(op.Cast<LogicalTopN>());
	case LogicalOperatorType::LOGICAL_COPY_TO_FILE:
		return CreatePlan(op.Cast<LogicalCopyToFile>());
	case LogicalOperatorType::LOGICAL_DUMMY_SCAN:
		return CreatePlan(op.Cast<LogicalDummyScan>());
	case LogicalOperatorType::LOGICAL_ANY_JOIN:
		return CreatePlan(op.Cast<LogicalAnyJoin>());
	case LogicalOperatorType::LOGICAL_ASOF_JOIN:
	case LogicalOperatorType::LOGICAL_DELIM_JOIN:
	case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
		return CreatePlan(op.Cast<LogicalComparisonJoin>());
	case LogicalOperatorType::LOGICAL_CROSS_PRODUCT:
		return CreatePlan(op.Cast<LogicalCrossProduct>());
	// ... (continues for every LogicalOperatorType)
	}
}
```

`op.Cast<T>()` is the same debug-checked downcast idiom `OperatorState::Cast`
used in §9.1.3 — recover the concrete logical subclass from the base
reference now that `op.type` has already told you which one it is, no
`dynamic_cast` needed. Each `CreatePlan(LogicalX&)` overload is declared as
its own method on `PhysicalPlanGenerator`
(`src/include/duckdb/execution/physical_plan_generator.hpp:135-188` lists all
of them, one per logical operator kind, `LogicalAggregate` through
`LogicalPivot`), and — this is the detail worth dwelling on — **each overload
is implemented in its own source file**, under
`src/execution/physical_plan/`, named `plan_<operator>.cpp`. This book's
research pass counted 51 such files; the checked-out tree today has grown to
55 (`plan_aggregate.cpp` through `plan_window.cpp`, alphabetically) — the
same small, honest "DuckDB moves fast" signal Chapter 5 flagged for
`LogicalOperator`'s subclass count, reproduced here for its physical-planning
mirror.

Reading down that switch statement is, in a very literal sense, reading a
table of contents for "every kind of SQL statement DuckDB's physical layer
knows how to make concrete": `plan_get.cpp` (scans), `plan_projection.cpp`,
`plan_filter.cpp`, `plan_aggregate.cpp` (§9.3.1's decision tree), `plan_window.cpp`,
`plan_limit.cpp`, `plan_order.cpp`, `plan_top_n.cpp`, `plan_comparison_join.cpp`
(§9.5, in full depth — the single most consequential file in this list),
`plan_any_join.cpp`, `plan_asof_join.cpp`, `plan_cross_product.cpp`,
`plan_positional_join.cpp`, `plan_delim_join.cpp`, `plan_distinct.cpp`,
`plan_set_operation.cpp`, `plan_recursive_cte.cpp`, `plan_cte.cpp`,
`plan_insert.cpp`, `plan_update.cpp`, `plan_delete.cpp`, `plan_merge_into.cpp`,
`plan_create.cpp`/`plan_create_table.cpp`/`plan_create_index.cpp`,
`plan_alter.cpp`, `plan_drop.cpp`, `plan_copy_to_file.cpp`,
`plan_copy_database.cpp`, `plan_pivot.cpp`, `plan_sample.cpp`,
`plan_dummy_scan.cpp`, `plan_empty_result.cpp`, `plan_expression_get.cpp`,
`plan_column_data_get.cpp`, `plan_delim_get.cpp`, and a long tail of
DDL/helper-statement plumbing files (`plan_explain.cpp`, `plan_export.cpp`,
`plan_pragma.cpp`, `plan_prepare.cpp`, `plan_execute.cpp`, `plan_set.cpp`,
`plan_reset.cpp`, `plan_load.cpp`, `plan_vacuum.cpp`, `plan_transaction.cpp`,
`plan_attach.cpp`, `plan_detach.cpp`, `plan_connect.cpp`,
`plan_disconnect.cpp`, `plan_create_secret.cpp`, `plan_secure_view.cpp`,
`plan_external_resource.cpp`, `plan_update_extensions.cpp`).

One instructive exception to "one file per enum value" is worth flagging
because it previews §9.5 directly: `LOGICAL_ASOF_JOIN`, `LOGICAL_DELIM_JOIN`,
and `LOGICAL_COMPARISON_JOIN` all route through the *same* switch case and the
*same* `CreatePlan(LogicalComparisonJoin&)` overload — implemented in
`plan_comparison_join.cpp` — which then internally sub-dispatches by
`op.type` a second time, to `PlanAsOfJoin` (living in the separate
`plan_asof_join.cpp`), `PlanComparisonJoin` (staying in
`plan_comparison_join.cpp` itself), or `PlanDelimJoin` (`plan_delim_join.cpp`):

```cpp
// src/execution/physical_plan/plan_comparison_join.cpp:291-302
PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalComparisonJoin &op) {
	switch (op.type) {
	case LogicalOperatorType::LOGICAL_ASOF_JOIN:
		return PlanAsOfJoin(op);
	case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
		return PlanComparisonJoin(op);
	case LogicalOperatorType::LOGICAL_DELIM_JOIN:
		return PlanDelimJoin(op);
	default:
		throw InternalException("Unrecognized operator type for LogicalComparisonJoin");
	}
}
```

The general pattern — one `LogicalOperatorType` value, one `CreatePlan`
overload, one file — is real and holds for the overwhelming majority of
cases; the comparison-join family is the one place it's a two-level dispatch
instead, and it happens to be exactly the two-level dispatch that matters
most for this chapter's next section.

(As a side note on `ClassifyLogicalOperatorRepeatability`, glimpsed in
`CreatePlan`'s wrapper above: it flags operators whose execution has
side effects that make replanning-and-rerunning them unsafe to treat as
interchangeable with a fresh copy — relevant to prepared-statement
re-execution and materialized-CTE reuse, not to ordinary single-shot `SELECT`
queries like Query A, and not pursued further here.)

### 9.4.5 Stage (e): the optional `PhysicalVerifyVector` debug wrap

The last few lines of `PlanInternal` (§9.4.3) check a client setting,
`DebugVerifyVectorSetting`, and — if it names one of the supported
verification modes — wrap the freshly-built root in a `PhysicalVerifyVector`
operator instead of returning it directly. This operator is never present in
an ordinary run; it exists purely for DuckDB's own testing and fuzzing:

```cpp
// src/include/duckdb/execution/operator/helper/physical_verify_vector.hpp:17-42 (elided: further modes, method declarations)
//! The PhysicalVerifyVector operator is a streaming operator that emits the same data as it ingests - but in a
//! different format
// There are different configurations
// * Dictionary: Transform every vector into a dictionary vector where the underlying vector has gaps and is reversed
//       i.e. original: FLAT [1, 2, 3]
//            modified: BASE: [NULL, 3, NULL, 2, NULL, 1]   OFFSETS: [5, 3, 1]
// * Constant: Decompose every DataChunk into single-row constant vectors
// ...
class PhysicalVerifyVector : public PhysicalOperator {
public:
	static constexpr const PhysicalOperatorType TYPE = PhysicalOperatorType::VERIFY_VECTOR;

public:
	PhysicalVerifyVector(PhysicalPlan &physical_plan, PhysicalOperator &child, DebugVectorVerification verification);

	DebugVectorVerification verification;
	// ...
};
```

The idea: DuckDB's vectorized execution engine (Chapters 10–13) supports
several different in-memory representations for the same logical column of
values — flat, constant, dictionary, sequence — and every operator in the
system is supposed to produce correct results regardless of which
representation shows up. `PhysicalVerifyVector` deliberately re-encodes every
`DataChunk` passing through it into one of these alternative, deliberately
awkward shapes (reversed dictionaries with gaps, single-row constant chunks,
lists with shuffled non-contiguous offsets) between two operators that would
otherwise never have exercised that particular representation against each
other — a targeted way to shake out format-specific bugs that ordinary query
workloads would rarely trigger. It's excluded for a few specific verification
modes (`DICTIONARY_EXPRESSION`, `VARIANT_VECTOR`, `SHREDDED_VECTOR`) that have
their own, separate verification paths elsewhere. Nothing about this operator
is reachable from any `LogicalOperatorType` at all — it's the one physical
operator this entire chapter's dispatch machinery never routes to, because
`PlanInternal` inserts it directly, after the switch-based dispatch has
already finished.

## 9.5 Special depth: `plan_comparison_join.cpp` and where join algorithm selection actually happens

Chapter 8 covered the *criteria* by which DuckDB decides a hash join is
preferable to a merge join, or a merge join to `IEJoin`, or any of those to a
plain nested-loop join — equality versus range predicates, cardinality
estimates, configurable thresholds. This section shows exactly where those
criteria stop being criteria and become a concrete C++ object, because it's
easy to read Chapter 8's decision framework and still not know *where in the
source* that framework actually executes. The answer is
`PhysicalPlanGenerator::PlanComparisonJoin`
(`src/execution/physical_plan/plan_comparison_join.cpp:177-289`), one single,
un-recursive function body, read here close to verbatim:

```cpp
// src/execution/physical_plan/plan_comparison_join.cpp:215-289
if (op.conditions.empty()) {
	// no conditions: insert a cross product
	return Make<PhysicalCrossProduct>(op.types, left, right, op.estimated_cardinality);
}

idx_t has_range = 0;
bool has_equality = op.HasEquality(has_range);
bool can_merge = has_range > 0;
bool can_iejoin = has_range >= 2 && recursive_cte_tables.empty();
switch (op.join_type) {
case JoinType::SEMI:
case JoinType::ANTI:
case JoinType::MARK:
	can_merge = can_merge && op.conditions.size() == 1;
	break;
case JoinType::RIGHT_ANTI:
case JoinType::RIGHT_SEMI:
	can_merge = can_merge && op.conditions.size() == 1;
	can_iejoin = false;
	break;
default:
	break;
}

//	TODO: Extend PWMJ to handle all comparisons and projection maps
bool prefer_range_joins = Settings::Get<PreferRangeJoinsSetting>(context);
prefer_range_joins = prefer_range_joins && can_iejoin;
if (has_equality && !prefer_range_joins) {
	// pass separately to PhysicalHashJoin
	auto &join = Make<PhysicalHashJoin>(op, left, right, std::move(op.conditions), op.join_type,
	                                    op.left_projection_map, op.right_projection_map, std::move(op.mark_types),
	                                    op.estimated_cardinality, std::move(op.filter_pushdown));
	return join;
}

// range joins apply both projection maps, the nested loop join only applies the left one of a MARK join
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
	return Make<PhysicalIEJoin>(op, left, right, std::move(op.conditions), op.join_type, op.estimated_cardinality,
	                            std::move(op.filter_pushdown));
}
if (can_merge) {
	// range join: use piecewise merge join
	return Make<PhysicalPiecewiseMergeJoin>(op, left, right, std::move(op.conditions), op.join_type,
	                                        op.estimated_cardinality, std::move(op.filter_pushdown));
}
if (PhysicalNestedLoopJoin::IsSupported(op.conditions, op.join_type)) {
	// inequality join: use nested loop
	D_ASSERT(op.left_projection_map.empty() || op.join_type == JoinType::MARK);
	return Make<PhysicalNestedLoopJoin>(op, left, right, std::move(op.conditions), op.join_type,
	                                    op.estimated_cardinality, std::move(op.filter_pushdown));
}

for (auto &cond : op.conditions) {
	if (cond.IsComparison()) {
		RewriteJoinCondition(cond.RightReference(), left.types.size());
	}
}
D_ASSERT(op.left_projection_map.empty());
auto condition = JoinCondition::CreateExpression(std::move(op.conditions));
return Make<PhysicalBlockwiseNLJoin>(op, left, right, std::move(condition), op.join_type, op.estimated_cardinality);
```

Reading this top to bottom is reading a priority order, and every branch
resolves a question Chapter 8 posed abstractly into a specific answer here:

1. **No join conditions at all** — an unconditional `FROM a, b` or explicit
   `CROSS JOIN` — is the one case with no algorithm to choose between at all:
   `PhysicalCrossProduct`, immediately, before any of the equality/range
   machinery below even runs.
2. **`op.HasEquality(has_range)`** inspects the actual `JoinCondition` list —
   the same structured representation Chapter 5 §5.4.4 introduced at the
   logical level — to determine, in one call, both whether *any* condition is
   an equality comparison and how many are range comparisons (`<`, `>`,
   `BETWEEN`-style). This single call is where the abstract "is this
   hash-joinable" question from Chapter 8 gets its concrete, mechanical
   answer.
3. **Equality present, and range joins aren't explicitly preferred** (a
   session setting, `PreferRangeJoinsSetting`, mostly useful for testing the
   range-join code paths deliberately) → `PhysicalHashJoin`, unconditionally,
   right there at line 244. For an ordinary equi-join like Query A's
   `o.orderkey = l.orderkey`, this is the branch that actually fires — no
   cardinality thresholds, no further comparison against the range-join
   machinery below it. Query A's `LogicalComparisonJoin` becomes a
   `PhysicalHashJoin` at exactly this line.
4. **No usable equality: cardinality gates.** `NestedLoopJoinThresholdSetting`
   and `MergeJoinThresholdSetting` are configurable minimum row-count
   thresholds — if either side of the join is estimated (using
   `estimated_cardinality`, propagated all the way from the optimizer's
   statistics pass through the logical tree, §9.1.2) to be smaller than the
   threshold, the more elaborate range-join machinery is disabled in favor of
   something simpler, on the reasoning that sort/merge overhead isn't worth
   paying for a join this small.
5. **Two-or-more range predicates and no recursive-CTE complication** →
   `PhysicalIEJoin`, DuckDB's dedicated inequality-join algorithm for
   multi-predicate range joins (the canonical use case: `a.start <= b.end AND
   a.end >= b.start`-style interval overlap conditions).
6. **One range predicate** → `PhysicalPiecewiseMergeJoin` — sort both sides,
   merge.
7. **Neither range-join strategy applies, but `PhysicalNestedLoopJoin`
   supports this specific condition/join-type combination** → plain
   nested-loop join. `PhysicalNestedLoopJoin::IsSupported`
   (`src/execution/operator/join/physical_nested_loop_join.cpp:133-161`)
   also requires **at least one comparison condition**. The nested-loop join
   uses its comparisons to find matching pairs, and only then checks the other
   (non-comparison) conditions on those pairs. If a join has no comparison
   left, it has nothing to drive the loop. This can happen when statistics
   propagation proves a comparison always true and removes it. For example,
   in `... JOIN (SELECT 0 b) ON a > b AND a + b > 0` with `a = 1`, the
   comparison `a > b` folds to true and only `a + b > 0` is left
   (`test/optimizer/statistics/statistics_join_arbitrary_condition.test`).
   Such a join goes to branch 8. (The test guards against a bug where such a
   join failed with an INTERNAL error.)
8. **Nothing above applies at all** — the final fallback, reached only for
   conditions too irregular for any of the specialized strategies —
   `PhysicalBlockwiseNLJoin`, after rewriting the right side's column
   references to account for the left side's column count (exactly the
   binding-offset arithmetic §9.4.2 described in the abstract, applied here
   concretely to a condition that's about to be re-expressed as one combined
   boolean expression rather than a structured `JoinCondition` list).

One detail in the excerpt: there is no
`D_ASSERT(op.left_projection_map.empty())` before the cardinality gates. A
*projection map* lists which input columns a join passes on to its output. The
source comment says the range joins (`PhysicalIEJoin`,
`PhysicalPiecewiseMergeJoin`) apply both projection maps, so a non-empty left
map is fine for them. The assertion sits only in the two joins that ignore the
left map: the nested-loop join (except for MARK joins) and the blockwise
nested-loop join.

This is, precisely, the moment Chapter 8's join-selection *criteria* stop
being criteria a cost model reasons about, and become one specific
`Make<PhysicalX>(...)` call — one line of this function, executed once, for
one join in one query's plan.

## 9.6 Query A, physically: the full trace

Chapter 1 previewed this stage with operator names and one-line annotations;
Chapter 5 §5.6 gave the exact unoptimized and optimized *logical* tree. This
section walks the same query the rest of the way, through every stage this
chapter has covered, ending in the concrete physical tree and — the piece
neither earlier chapter could show — the exact `BoundReferenceExpression`
indices `ColumnBindingResolver` produces at each level.

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

Recall the optimized logical tree Chapter 1 §1.3 arrived at — filter pushdown
has already folded `WHERE l.shipdate < ...` directly into `lineitem`'s
`LogicalGet::table_filters`, and `top_n` has already fused `ORDER BY` +
`LIMIT` into one `LogicalTopN`:

```
LogicalTopN (ORDER BY total DESC LIMIT 10)
  └─ LogicalAggregate (GROUP BY o.custkey; SUM(l.extendedprice))
       └─ LogicalComparisonJoin (o.orderkey = l.orderkey)
            ├─ LogicalGet (orders)
            └─ LogicalGet (lineitem, filter pushed into scan: shipdate < ...)
```

Using Chapter 5 §5.6's exact `ColumnBinding`s (`o.orderkey = {0,0}`,
`o.custkey = {0,1}`, `l.orderkey = {1,0}`, `l.extendedprice = {1,5}`,
`l.shipdate = {1,10}` — these match real TPC-H column ordering: `orderkey`
and `custkey` are `orders`' first two columns; `extendedprice` and `shipdate`
are `lineitem`'s sixth and eleventh columns respectively), and noting that by
this point projection pushdown (Chapter 7) has already trimmed each
`LogicalGet`'s exposed column list down to only what's needed further up the
tree — `orders` down to `[orderkey, custkey]`, `lineitem` down to
`[orderkey, extendedprice]`, with `shipdate` consumed entirely inside the
pushed-down `table_filters` and never entering the output chunk at all
(Chapter 7 covers this pruning mechanism in full; this chapter only needs its
*result*, the already-pruned column lists, as input) — here is
`ColumnBindingResolver`'s walk, stage by stage, exactly per §9.4.2's
mechanism:

**`LogicalGet(orders)`.** General case (§9.4.2's fallback branch): `bindings`
becomes `orders`' own `GetColumnBindings()`, i.e. `[{0,0}, {0,1}]` — two
entries, local positions 0 and 1.

**`LogicalGet(lineitem)`.** Same general case, independently:
`bindings = [{1,0}, {1,5}]` — again two entries, local positions 0 and 1
(remember: these are positions *within lineitem's own pruned output*, not
positions within the full 16-column table — `extendedprice` is lineitem's
*local* column 1 here, even though it's column 5 of the full table).

**`LogicalComparisonJoin`.** The hand-written join case from §9.4.2 fires.
Visiting the left child first sets `left_bindings = [{0,0}, {0,1}]`; the join
condition's left reference, `o.orderkey = {0,0}`, matches position 0 in that
list — resolved to `BoundReferenceExpression{0}`. Visiting the right child
sets `right_bindings = [{1,0}, {1,5}]`; the condition's right reference,
`l.orderkey = {1,0}`, matches position 0 in *that* list — resolved to
`BoundReferenceExpression{0}` too, for now, before the offset shift below.
The two lists are then concatenated, left first: `combined_bindings =
[{0,0}, {0,1}, {1,0}, {1,5}]`. There's no non-comparison predicate to resolve
against that combined list here (the join condition is a pure equality,
already resolved above), so the join's own output bindings become exactly
this four-entry combined list — `bindings = op.GetColumnBindings() = [{0,0},
{0,1}, {1,0}, {1,5}]` — the binding context every operator above the join
now inherits. Concretely: position 0 = `orders.orderkey`, position 1 =
`orders.custkey`, position 2 = `lineitem.orderkey`, position 3 =
`lineitem.extendedprice`.

**`LogicalAggregate`.** General case again: children already visited, so this
operator's own expressions are resolved against the join's four-entry
combined bindings above. `groups = [o.custkey = {0,1}]` matches position 1 —
`BoundReferenceExpression{1}`. `expressions = [SUM(l.extendedprice = {1,5})]`
matches position 3 — the `SUM`'s argument becomes
`BoundReferenceExpression{3}`. The aggregate's own output bindings then
become the new context for whatever's above it: `bindings =
[{2,0}, {2,1}]` (Chapter 5 §5.6's `group_index`/`aggregate_index`, both 2) —
two entries: position 0 is the group-by result (`custkey`), position 1 is the
`SUM` result (`total`).

**`LogicalTopN`.** General case: `orders = [{ expr: {2,1} ("total"),
ASCENDING: false }]` resolves against the aggregate's two-entry output —
`{2,1}` matches position 1 — `total`'s sort key becomes
`BoundReferenceExpression{1}`.

Every one of these numbers now feeds directly into `CreatePlanInternal`'s
switch (§9.4.4) and, for the join specifically, `PlanComparisonJoin` (§9.5) —
an equality condition on both sides (`op.HasEquality` returns true, no range
predicates at all) puts this join squarely in §9.5's branch 3, producing a
`PhysicalHashJoin` with no further cardinality-threshold logic consulted at
all. The aggregate has one `GROUP BY` column (`custkey`, a `BIGINT`) and one
non-distinct, combinable `SUM` — §9.3.1's checks run in order:
`CanUsePartitionedAggregate` needs the child plan to already be partitioned by
`custkey` (walking back through `PhysicalHashJoin` — which is not one of the
`PROJECTION`/`FILTER` pass-through cases `HasSingleValuePartitions` walks
through, so the walk fails immediately and this path is rejected);
`CanUsePerfectHashAggregate` could in principle succeed if the optimizer's
statistics pass attached a usable min/max range for `custkey` within the
`PerfectHtThresholdSetting` bit budget — assume, for this trace, that no
such statistics were available (an entirely realistic outcome for a
freshly-loaded table with no `ANALYZE` run) — which falls through to the
general-purpose `PhysicalHashAggregate`. `LogicalTopN` becomes `PhysicalTopN`
directly (§9.3.4 — no decision to make, one-to-one). The whole tree is
capped, as every `SELECT`'s physical tree is, by a `PhysicalResultCollector`
(§9.3.10) that `PlanInternal` never shows explicitly constructing in this
chapter's excerpts because it's assembled by the caller of `Plan()` from
`PreparedStatementData`, one layer above `PhysicalPlanGenerator` itself, via
exactly the `PhysicalResultCollector::GetResultCollector` static factory
quoted in §9.3.10.

The final physical tree:

```
┌───────────────────────────────────────────────────────────────────┐
│ PhysicalResultCollector                                            │  root — gathers final output
└──────────────────────────────┬────────────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────────┐
│ PhysicalTopN                                                       │  sort key: BoundReferenceExpression{1}
│   ORDER BY total DESC LIMIT 10                                     │  ("total", position 1 of aggregate's output)
└──────────────────────────────┬────────────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────────┐
│ PhysicalHashAggregate                                               │  groups: {1}  ("custkey", pos 1 of join output)
│   GROUP BY custkey; SUM(extendedprice)                              │  agg args: {3} ("extendedprice", pos 3)
└──────────────────────────────┬────────────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────────┐
│ PhysicalHashJoin                                                    │  condition: left{0} = right{0}
│   build: orders  probe: lineitem  (Chapter 8: build/probe choice)   │  (each side resolved against its OWN
└──────────────┬─────────────────────────────────┬───────────────────┘   local bindings before concatenation)
               │                                    │
┌──────────────▼───────────────┐    ┌──────────────▼──────────────────┐
│ PhysicalTableScan (orders)    │    │ PhysicalTableScan (lineitem)      │
│   projects: [orderkey,        │    │   projects: [orderkey,             │
│              custkey]         │    │              extendedprice]        │
│   (2 output columns)          │    │   table_filters: shipdate < ...    │
│                                │    │   (2 output columns, shipdate     │
│                                │    │    consumed by the filter only)   │
└───────────────────────────────┘    └────────────────────────────────────┘
```

The chunk-column arithmetic is worth stating explicitly once more, because
it's the whole payoff of this section: the `PhysicalHashAggregate`'s `SUM`
argument, `BoundReferenceExpression{3}`, is *not* "column 3 of `lineitem`" —
`lineitem`'s own pruned scan only emits 2 columns. It is column 3 of the
*4-column chunk `PhysicalHashJoin` produces* (`orders.orderkey`,
`orders.custkey`, `lineitem.orderkey`, `lineitem.extendedprice`, in that
concatenation order) — and that chunk shape, in turn, exists only because
`ColumnBindingResolver` walked the join exactly as §9.4.2 described, resolved
each side against its own local bindings first, and then concatenated left
before right. Nothing about "table 1, column 5" survives anywhere past that
point; every operator from `PhysicalHashJoin` upward only ever sees bare
integer offsets into whatever chunk its child handed it.

## 9.7 Handoff

What `PhysicalPlanGenerator::Plan` returns — a `unique_ptr<PhysicalPlan>`
whose arena holds every operator this chapter has named, rooted at a
`PhysicalResultCollector`, every column reference already a flat chunk
offset, every join and aggregate already committed to one specific algorithm
— is handed to the `Executor` (Chapter 11), which does the one thing this
chapter's tree has been built for and has not yet done at all: run it.
Chapter 11 picks up exactly here, with the question this chapter's `sink_state`/
`op_state`/`lock` fields (§9.1.3) only gestured at — how a static tree of
C++ objects becomes a set of `Pipeline`s, scheduled across worker threads by
the shared `TaskScheduler`, pulling and pushing `DataChunk`s until the
`PhysicalResultCollector` at the root has everything it needs to hand back to
whatever client asked for `custkey` and `total`, ten rows, in order.
