# Chapter 11 — Pipelines and Parallelism

## 11.1 Two models for running a plan

Chapter 9 leaves us holding a `PhysicalOperator` tree. Chapter 1 already showed
what that tree looks like for Query A:

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

The question this chapter answers is: what actually *calls* these operators,
in what order, on how many threads? There are, broadly, two answers in the
history of database engines, and DuckDB deliberately picked the less famous
one.

**The classic answer is the Volcano/iterator model** (Goetz Graefe [29]):
every operator exposes one method, conventionally called `GetChunk()` or
`Next()`, and the *parent* calls it on the *child* to pull one chunk (or one
tuple, in the original row-at-a-time formulation) at a time. Execution starts
at the root — "give me a chunk" — and that call recurses all the way down to
the leaves before any row moves. This is a **pull** model: control flow starts
at the top of the tree and control passes downward through function calls;
data then flows back upward through return values. It is the model
PostgreSQL still uses today (its executor's `ExecProcNode()` dispatch is a
direct descendant of the original Volcano design), and it is the model nearly
every database textbook teaches first, because recursion over a tree of
operators is a genuinely elegant way to describe a query.

**DuckDB does not use this model.** Since release 0.3.0 (October 2021), DuckDB
runs a **push-based pipeline-driver** model instead: a driver loop external to
every operator pulls chunks from a *source*, pushes each chunk through a chain
of *operators* in a fixed order, and hands the surviving chunk to a *sink*.
Control flow lives in the driver, not recursively distributed across operator
implementations. Data still flows upward (source to sink), matching the plan
diagrams' arrows — but the *calls* that make it flow no longer originate at
the root and recurse downward. They originate at a driver loop that sits
logically beside the pipeline, not above it.

Why does this matter enough to open a chapter with it? Because the
difference is easiest to see — and the reason DuckDB switched is easiest to
justify — by picking the one operator that the pull model handles worst: a
hash join.

### 11.1.1 A hash join, two ways

A hash join has two distinct phases with two distinct kinds of input: it must
first consume its *build* side entirely (inserting every row into a hash
table), and only then can it consume its *probe* side (looking up each probe
row against the finished table). Under the pull model, both phases have to be
squeezed into the *same* method, because there is only one method —
`GetChunk()` — and the parent calls it without knowing or caring which phase
the join is currently in.

```cpp
// illustrative — Volcano-style pull model, NOT DuckDB's code
class HashJoin : public PhysicalOperator {
    unique_ptr<HashTable> ht;
    bool built = false;

    DataChunk *GetChunk() override {
        if (!built) {
            // Build phase: drain the ENTIRE build child before
            // this operator can return a single probe-side row.
            // This loop is invisible to the caller — GetChunk()
            // just doesn't return for a while.
            DataChunk *build_chunk;
            while ((build_chunk = build_child->GetChunk()) != nullptr) {
                ht->Insert(*build_chunk);
            }
            built = true;
        }
        // Probe phase: now, and only now, start pulling from the
        // probe child and looking rows up in ht.
        DataChunk *probe_chunk = probe_child->GetChunk();
        if (probe_chunk == nullptr) {
            return nullptr; // exhausted
        }
        return ht->Probe(*probe_chunk);
    }
};
```

The `built` flag is the tell. `GetChunk()` has to encode "have I finished
building yet?" *and* "give me the next probe result" as two different control
paths inside one function, because the caller has no way to ask for one
without risking getting the other. Two fundamentally different concerns —
one-time, blocking, all-input-at-once construction, and per-call, streaming,
one-chunk-at-a-time lookup — are entangled inside a single recursive call.
This is exactly the DiDi course's complaint [8] about the pull model applied to
joins: build and probe are "entangled, hard to parallelize," because nothing
about the interface separates them. You cannot hand the build loop to eight
threads and the probe loop to eight different threads without the operator
itself growing thread-pool logic, mutexes around `built`, and a way to block
callers until the build is done — none of which the Volcano interface gives
you for free. Every join operator has to reinvent this parallelization
machinery internally, and get it right, on its own.

Now the push model:

```cpp
// illustrative — push-based pipeline model, DuckDB's actual shape
// (real Sink/Combine/Finalize/Execute signatures follow in §11.3)
class HashJoinBuild : public PhysicalOperator {
    SinkResultType Sink(DataChunk &chunk, GlobalSinkState &gstate) const override {
        // Just insert. No loop, no phase-tracking. The DRIVER decided
        // to call this — the operator doesn't ask "am I still building?"
        gstate.ht.Insert(chunk);
        return SinkResultType::NEED_MORE_INPUT;
    }
    SinkFinalizeType Finalize(GlobalSinkState &gstate) const override {
        // Called exactly once, single-threaded, after the driver has
        // confirmed every thread's Sink calls are done.
        gstate.ht.FinalizeBuild();
        return SinkFinalizeType::READY;
    }
};

class HashJoinProbe : public PhysicalOperator {
    OperatorResultType Execute(DataChunk &input, DataChunk &chunk,
                               GlobalOperatorState &gstate) const override {
        // No knowledge of "build" exists here at all. By the time any
        // thread calls this, gstate.ht is already finalized — a
        // separate PIPELINE guaranteed that (§11.6).
        chunk.Reference(gstate.ht.Probe(input));
        return OperatorResultType::NEED_MORE_INPUT;
    }
};
```

`Sink` and `Execute` are each doing exactly one thing, with no internal
control flow of their own — no loops, no phase flags, no "have I started yet"
state. That is only possible because the *build* and *probe* halves of the
join have been split into two separate physical operators sharing one piece
of state, running as two separate **pipelines**, with an explicit dependency
between them: the probe pipeline is not even allowed to start until the build
pipeline's `Finalize` has returned. Parallelizing the build phase now just
means running many threads' `Sink` calls concurrently (safe, because
`Sink`'s contract already promises that — see §11.3) and letting a single
thread run `Finalize` once everyone is done. None of that thread-pool logic
lives inside `HashJoinBuild` or `HashJoinProbe` — it lives in the driver and
the pipeline dependency graph, which is precisely the point of this chapter.

## 11.2 Pipeline and MetaPipeline, precisely

**A `Pipeline` is a linear chain: exactly one source, zero or more
mid-pipeline operators, and exactly one sink.** It is the unit that a single
thread runs from end to end without needing to consult any other pipeline —
which is why the DiDi course's diagram for "parallelism in a pipeline" draws
it as one box on the left (source), a stack of identical boxes in the middle
(operators, one instance per thread), and one box on the right (sink)
[8, slide 11]:

```
 —flow→   ┌──────┐  ┌──────┐┌──────┐┌──────┐  ┌──────┐
          │Source│◄─┤ Op   ├┤ Op   ├┤ Op   ├─►│ Sink │  } a pipeline
          └──────┘  └──────┘└──────┘└──────┘  └──────┘
                     └──────┘└──────┘└──────┘
                     └──────┘└──────┘└──────┘   (one row of operator
                                                  instances per thread)
```

Sources and sinks are "parallelism-aware" — they carry explicit interfaces
for splitting and merging work across threads (§11.3, §11.5). Mid-pipeline
operators, by contrast, need not be parallelism-aware at all: `PhysicalFilter`
or `PhysicalProjection` have no idea how many threads are running copies of
them, because each thread's copy only ever sees the chunk handed to it by its
own thread's driver loop [8, slide 11].

The real class matches this shape closely
(`src/include/duckdb/parallel/pipeline.hpp:111` onward):

```cpp
// src/include/duckdb/parallel/pipeline.hpp:111
class Pipeline : public enable_shared_from_this<Pipeline> {
	friend class Executor;
	friend class PipelineExecutor;
	friend class PipelineEvent;
	friend class PipelineFinishEvent;
	friend class PipelineBuildState;
	friend class MetaPipeline;

public:
	explicit Pipeline(Executor &execution_context);

	Executor &executor;

public:
	ClientContext &GetClientContext();

	void AddDependency(shared_ptr<Pipeline> &pipeline);
	void AddDataflowDependency(shared_ptr<Pipeline> &pipeline);
	//! The dependencies of this pipeline on pipelines of other MetaPipelines
	vector<weak_ptr<Pipeline>> GetDependencies() const;
	//! The dependencies of this pipeline on pipelines of the same MetaPipeline
	const vector<PipelineDependency> &GetIntraDependencies() const { ... }
	// ...
	//! Returns a list of all operators (including source and sink) involved in this pipeline
	vector<reference<PhysicalOperator>> GetOperators();
	// ...
};
```

The pieces are all there in the member list: an `Executor &` (the query this
pipeline belongs to — Chapter 1's diagram already noted one `Executor` per
running query), a way to add dependencies on *other* pipelines
(`AddDependency` / `AddDataflowDependency`, expanded in §11.6), and a
`GetOperators()` accessor that returns source, operators, and sink together as
one ordered list — confirming that "source, operators, sink" is not just an
informal description but the literal shape the class stores.

**The schedulable unit that actually runs a `Pipeline`** is `PipelineTask`, a
thin wrapper that owns one `PipelineExecutor` (§11.4) and gets handed to the
`TaskScheduler` (§11.7):

```cpp
// src/include/duckdb/parallel/pipeline.hpp:29-64
enum class PipelineInputMode : uint8_t { SCHEDULED_SOURCE, EXTERNAL_INPUT };
enum class PipelineDependencyType : uint8_t { REQUIRED, OPTIONAL_DEPENDENCY };

//! A dependency of a Pipeline on another Pipeline within the same MetaPipeline
struct PipelineDependency {
	PipelineDependency(Pipeline &pipeline_p, PipelineDependencyType type_p);
	weak_ptr<Pipeline> pipeline;
	PipelineDependencyType type;
};

class PipelineTask : public ExecutorTask {
	static constexpr const idx_t PARTIAL_CHUNK_COUNT = 50;

public:
	explicit PipelineTask(Pipeline &pipeline_p, shared_ptr<Event> event_p);
	Pipeline &pipeline;
	unique_ptr<PipelineExecutor> pipeline_executor;
	optional_idx reserved_batch_index;
	// ...
};
```

`PipelineDependencyType` is worth flagging now, because it resurfaces in
§11.6: a `REQUIRED` dependency means "the dependent pipeline literally cannot
start correctly" (the hash join build/probe relationship from §11.1), whereas
`OPTIONAL_DEPENDENCY` exists for weaker orderings that affect scheduling
preference without being a correctness requirement.

### 11.2.1 MetaPipeline: several pipelines, one shared sink

A single `Pipeline` cannot express a hash join by itself, because a hash join
needs *two* pipelines (build and probe) that both ultimately feed the same
piece of state — the join's `GlobalSinkState`, holding the hash table. That
grouping is what `MetaPipeline` is for: **a DAG of `Pipeline`s that all share
one sink operator's global state.**

```cpp
// src/include/duckdb/parallel/meta_pipeline.hpp:17-55
enum class MetaPipelineType : uint8_t {
	REGULAR = 0,   //! The shared sink is regular
	JOIN_BUILD = 1 //! The shared sink is a join build
};

//! MetaPipeline represents a set of pipelines that all have the same sink
class MetaPipeline : public enable_shared_from_this<MetaPipeline> {
	//! We follow these rules when building:
	//! 1. For joins, build out the blocking side before going down the probe side
	//!     - The current streaming pipeline will have a dependency on it (dependency across MetaPipelines)
	//!     - Unions of this streaming pipeline will automatically inherit this dependency
	//! 2. Build child pipelines last (e.g., Hash Join becomes source after probe is done: scan HT for FULL OUTER JOIN)
	//!     - 'last' means after building out all other pipelines associated with this operator
	//!     - The child pipeline automatically has dependencies (within this MetaPipeline) on:
	//!         * The 'current' streaming pipeline
	//!         * And all pipelines that were added to the MetaPipeline after 'current'
public:
	MetaPipeline(Executor &executor, PipelineBuildState &state, optional_ptr<PhysicalOperator> sink,
	             MetaPipelineType type = MetaPipelineType::REGULAR);
	// ...
	shared_ptr<Pipeline> &GetBasePipeline();
	void GetPipelines(vector<shared_ptr<Pipeline>> &result, bool recursive);
	// ...
};
```

That doc comment is dense but precise, and it is worth reading twice: rule 1
is the build-before-probe ordering from §11.1 stated as a build-time
invariant, not just a runtime fact — the physical plan generator (Chapter 9)
constructs the build-side `MetaPipeline` and wires the probe-side pipeline's
dependency on it *while building the plan*, before any thread runs anything.
Rule 2 is a detail this chapter will come back to in §11.5: a hash join can
become a *source* in its own right (for `FULL OUTER JOIN`, to emit build-side
rows that were never matched by any probe row), and when that happens the
"scan the hash table" pipeline is deliberately built *last*, with dependencies
on every other pipeline already added to that `MetaPipeline`.

For Query A's join (`orders` build, `lineitem` probe — no outer join, so no
third "scan unmatched build rows" pipeline), the `MetaPipeline` shape is:

```
                     MetaPipeline (sink = PhysicalHashJoin)
                     shared: HashJoinGlobalSinkState (the hash table)
        ┌───────────────────────────┬──────────────────────────────┐
        │                            │                               │
┌───────▼────────┐          ┌────────▼─────────┐            (dependency:
│ Pipeline (build) │          │ Pipeline (probe)  │◄───────── probe pipeline
│ source: TableScan │          │ source: TableScan  │           REQUIRES the
│   (orders)         │          │   (lineitem)        │           build pipeline's
│ sink: HashJoin.Sink│          │ operators: HashJoin  │           Finalize to
│   (build phase)    │          │   .Execute (probe)   │           have already run
└───────────────────┘          │ sink: (next pipeline's│
                                 │   sink — see §11.9)   │
                                 └──────────────────────┘
```

Both pipelines' Sink/Combine/Finalize calls read and write the *same*
`HashJoinGlobalSinkState` object — that is the entire meaning of "sharing a
sink" here. The build pipeline's `Finalize` (§11.3) is what turns the
raw, per-thread hash tables accumulated during `Sink`/`Combine` into one
probeable structure; the probe pipeline is not scheduled until that call has
returned (§11.6 shows the exact dependency bookkeeping that enforces this).

## 11.3 The Sink / Source / Operator interface

`PhysicalOperator` (`src/include/duckdb/execution/physical_operator.hpp`) is
where all three roles a physical operator can play — mid-pipeline operator,
source, sink — are defined as virtual methods. Every concrete operator
(`PhysicalHashJoin`, `PhysicalTableScan`, `PhysicalHashAggregate`, ...)
overrides whichever subset applies to it; an operator can be none of these
(pure pass-through), a source only, a sink only, or — as §11.5 covers —
*both* a sink and a source at once.

### 11.3.1 Operator interface (mid-pipeline transform)

```cpp
// src/include/duckdb/execution/physical_operator.hpp:105-112
public:
	// Operator interface
	virtual unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) const;
	virtual unique_ptr<GlobalOperatorState> GetGlobalOperatorState(ClientContext &context) const;
	virtual bool ResetGlobalOperatorState(ClientContext &context, GlobalOperatorState &state) const;
	virtual OperatorResultType Execute(ExecutionContext &context, DataChunk &input, DataChunk &chunk,
	                                   GlobalOperatorState &gstate, OperatorState &state) const;
	virtual OperatorFinalizeResultType FinalExecute(ExecutionContext &context, DataChunk &chunk,
	                                                GlobalOperatorState &gstate, OperatorState &state) const;
```

`Execute` is the one every "trivially parallel" operator implements —
`PhysicalFilter` discards rows, `PhysicalProjection` evaluates expressions,
each writing its result into `chunk` and returning `NEED_MORE_INPUT`
(consumed everything, driver should fetch the next input chunk),
`HAVE_MORE_OUTPUT` (this input chunk produced more output than fits in one
result chunk — the driver must call `Execute` again with the *same* input
before advancing), or `FINISHED` (this operator will never produce output
again — e.g. a `LIMIT` deciding it has already emitted enough rows, which
lets the whole pipeline shut down early).

`FinalExecute` is rarer: it exists for operators that buffer rows internally
and need one last call, after the normal input stream is exhausted, to flush
whatever they were holding onto (`RequiresFinalExecute()` reports whether a
given operator needs this — the pipeline driver checks it explicitly, as seen
in `PipelineExecutor::TryFlushCachingOperators`,
`src/parallel/pipeline_executor.cpp:161`).

One more hook decides how the driver allocates the chunk an operator writes
into. The driver does not simply allocate a full buffer for every output
column. It asks the operator:

```cpp
// src/include/duckdb/execution/physical_operator.hpp:94-97
	//! Initialize the vectors this operator uses for its output
	virtual void InitializeOutputChunk(Allocator &allocator, DataChunk &chunk) const {
		chunk.Initialize(allocator, types);
	}
```

The default allocates every column. But some operators never write
new values into a column. They only point at (reference) the input vector.
`PhysicalProjection` is the clearest case. In `SELECT s, i AS j FROM t`,
both output columns are plain column references (`BOUND_REF`). Each output
vector just references an input vector, so a buffer for it is wasted memory:

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

`PhysicalFilter`, `PhysicalStreamingLimit`, `PhysicalStreamingSample` and
`PhysicalHashJoin` override the hook too. A filter, for example, only slices
its input, so it asks for no buffers at all. The driver calls the hook in
`PipelineExecutor`'s constructor and in `PipelineExecutor::InitializeChunk`
(`src/parallel/pipeline_executor.cpp:65` and `:899`).
`test/sql/projection/reference_only_output.test` checks that reference-only
projections still give correct results, also when a batch is empty.

### 11.3.2 Source interface

```cpp
// src/include/duckdb/execution/physical_operator.hpp:145-178
public:
	// Source interface
	virtual unique_ptr<LocalSourceState> GetLocalSourceState(ExecutionContext &context,
	                                                         GlobalSourceState &gstate) const;
	virtual unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext &context) const;
	virtual unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext &context,
	                                                           const OperatorPartitionInfo &partition_info) const;

protected:
	virtual SourceResultType GetDataInternal(ExecutionContext &context, DataChunk &chunk,
	                                         OperatorSourceInput &input) const;

public:
	SourceResultType GetData(ExecutionContext &context, DataChunk &chunk, OperatorSourceInput &input) const;

	virtual OperatorPartitionData GetPartitionData(ExecutionContext &context, DataChunk &chunk,
	                                               GlobalSourceState &gstate, LocalSourceState &lstate,
	                                               const OperatorPartitionInfo &partition_info) const;

	virtual bool IsSource() const {
		return false;
	}

	virtual bool ParallelSource() const {
		return false;
	}

	//! Whether this source creates partitioned work that is not bounded by its input chunks
	virtual bool HasSourceTasks() const {
		return false;
	}

	//! How this source manages parallelism
	virtual TableFunctionParallelism SourceParallelism() const;
```

`GetLocalSourceState`/`GetGlobalSourceState` split source state the same way
sink state splits (below): a `GlobalSourceState` shared read-mostly by every
thread scanning this source, and one `LocalSourceState` per thread holding
whatever cursor or buffer that thread privately needs to make progress
without contending with other threads. `GetData` is the public entry point
the driver calls; it wraps the protected `GetDataInternal` (a `NON_VIRTUAL_
INTERFACE`-style split that lets the base class do bookkeeping —
`StartOperator`/`EndOperator` profiling timestamps — around every call
without every subclass having to remember to do it).

`IsSource()`, `ParallelSource()`, `HasSourceTasks()`, and `SourceParallelism()`
are the levers that determine *how* this particular source gets split across
threads — full treatment in §11.5, since that is the mechanism the outside
world calls "morsel-driven parallelism" (§11.8) even though nothing in this
header uses that word.

### 11.3.3 Sink interface

This is the interface with the richest doc comments in the whole header, and
they are worth reading verbatim because they are the actual parallel-safety
contract every sink operator must honor:

```cpp
// src/include/duckdb/execution/physical_operator.hpp:202-228
public:
	// Sink interface

	//! The sink method is called constantly with new input, as long as new input is available. Note that this method
	//! CAN be called in parallel, proper locking is needed when accessing data inside the GlobalSinkState.
	virtual SinkResultType Sink(ExecutionContext &context, DataChunk &chunk, OperatorSinkInput &input) const;
	//! The combine is called when a single thread has completed execution of its part of the pipeline, it is the final
	//! time that a specific LocalSinkState is accessible. This method can be called in parallel while other Sink() or
	//! Combine() calls are active on the same GlobalSinkState.
	virtual SinkCombineResultType Combine(ExecutionContext &context, OperatorSinkCombineInput &input) const;
	//! (optional) function that will be called before Finalize
	//! For now, its only use is to to communicate memory usage in multi-join pipelines through TemporaryMemoryManager
	virtual void PrepareFinalize(ClientContext &context, GlobalSinkState &sink_state) const;
	//! The finalize is called when ALL threads are finished execution. It is called only once per pipeline, and is
	//! entirely single threaded.
	//! If Finalize returns SinkResultType::Finished, the sink is marked as finished
	virtual SinkFinalizeType Finalize(Pipeline &pipeline, Event &event, ClientContext &context,
	                                  OperatorSinkFinalizeInput &input) const;
	//! For sinks with RequiresBatchIndex set to true, when a new batch starts being processed this method is called
	//! This allows flushing of the current batch (e.g. to disk)
	virtual SinkNextBatchType NextBatch(ExecutionContext &context, OperatorSinkNextBatchInput &input) const;
	//! Called after NextBatch when the pipeline minimum advances without subsequent input for this local sink state
	virtual SinkNextBatchType UpdateMinBatchIndex(ExecutionContext &context, OperatorSinkNextBatchInput &input) const;

	virtual unique_ptr<LocalSinkState> GetLocalSinkState(ExecutionContext &context) const;
	virtual unique_ptr<GlobalSinkState> GetGlobalSinkState(ClientContext &context) const;
```

Three phases, three different safety contracts:

- **`Sink`** — called once per incoming chunk, *repeatedly*, by every thread
  running this pipeline, for as long as chunks keep arriving. The comment is
  explicit that this **can be called in parallel**: any state touched inside
  `Sink` that isn't private to one thread's `LocalSinkState` needs its own
  locking (`GlobalSinkState` is fair game to mutate, but only under a lock the
  operator manages itself — DuckDB does not take a lock for you here).
- **`Combine`** — called exactly once per thread, at the moment that thread
  has seen the last chunk it will ever see for this pipeline. It is the
  *last* point at which that thread's `LocalSinkState` is legally touchable —
  after `Combine` returns, that state is gone. Critically, the comment notes
  `Combine` **can run concurrently with other threads' `Sink` or `Combine`
  calls** on the *same* `GlobalSinkState` — thread A finishing early and
  calling `Combine` does not have to wait for thread B, still mid-`Sink`, to
  catch up. This is a genuinely subtle guarantee: it means `Combine`'s own
  merge-into-global-state step must itself be safe to run concurrently with
  other threads' inserts, not just with other `Combine` calls.
- **`Finalize`** — called exactly once, after `Combine` has been called by
  *every* thread that ran `Sink` at all, and is **entirely single-threaded**.
  This is where whatever the global state accumulated gets turned into
  whatever shape downstream pipelines expect to read. Returning
  `SinkFinalizeType::NO_OUTPUT_POSSIBLE` (say, because the sink knows in
  advance no row can survive — a build side that turns out to be empty, for
  instance) lets the pipeline driver skip executing the dependent pipeline
  entirely (`PipelineExecutor`'s constructor checks exactly this,
  `src/parallel/pipeline_executor.cpp:71-75`).
- **`NextBatch`** exists for a different concern entirely: order-sensitive
  sinks (writing Parquet files, `COPY ... TO`) that need to know when the
  *batch* of rows currently arriving has rolled over to the next one, so they
  can flush what they were accumulating for the previous batch before new
  rows for a new batch start arriving out of order across threads.

### 11.3.4 Two running illustrations: HashJoin's build side, HashAggregate

**`PhysicalHashJoin`'s build side** puts real flesh on this contract. `Sink`
resolves the join keys for the incoming chunk and inserts it into a
thread-local hash table — no loop, no phase tracking, exactly the shape
promised in §11.1:

```cpp
// src/execution/operator/join/physical_hash_join.cpp:896-927
SinkResultType PhysicalHashJoin::Sink(ExecutionContext &context, DataChunk &chunk, OperatorSinkInput &input) const {
	auto &gstate = input.global_state.Cast<HashJoinGlobalSinkState>();
	auto &lstate = input.local_state.Cast<HashJoinLocalSinkState>();

	// resolve the join keys for the right chunk
	lstate.join_keys.Reset();
	lstate.join_key_executor.Execute(chunk, lstate.join_keys);
	// ...
	// build the HT
	lstate.hash_table->Build(lstate.append_state, lstate.join_keys, lstate.payload_chunk);

	return SinkResultType::NEED_MORE_INPUT;
}
```

`Combine` is where each thread's private hash table gets handed over to the
shared collection — under a lock, exactly as the doc comment demands, and
explicitly designed so a thread that never received any input (an empty
`LocalSinkState`) can bail out cheaply instead of merging nothing:

```cpp
// src/execution/operator/join/physical_hash_join.cpp:934-952
SinkCombineResultType PhysicalHashJoin::Combine(ExecutionContext &context, OperatorSinkCombineInput &input) const {
	auto &gstate = input.global_state.Cast<HashJoinGlobalSinkState>();
	auto &lstate = input.local_state.Cast<HashJoinLocalSinkState>();

	// Under deferred layout, a thread that never received a Sink chunk has no sink_collection to flush
	const bool has_layout = lstate.append_state_initialised;
	if (has_layout) {
		lstate.hash_table->GetSinkCollection().FlushAppendState(lstate.append_state);
	}
	annotated_lock_guard<annotated_mutex> guard(gstate.lock);
	if (!has_layout) {
		// nothing to merge — drop the empty thread-local hash table
		gstate.active_local_states--;
	} else if (lstate.keep_hash_table) {
		gstate.local_hash_tables.push_back(*lstate.hash_table);
	} else {
		gstate.owned_local_hash_tables.push_back(std::move(lstate.hash_table));
		gstate.local_hash_tables.push_back(*gstate.owned_local_hash_tables.back());
	}
	// ...
}
```

And `Finalize`, single-threaded, is where the *decision* the rest of the join
depends on gets made — whether the combined hash table fits comfortably in
the memory budget or needs to spill to disk as an external join:

```cpp
// src/execution/operator/join/physical_hash_join.cpp:2015-2027
SinkFinalizeType PhysicalHashJoin::Finalize(Pipeline &pipeline, Event &event, ClientContext &context,
                                            OperatorSinkFinalizeInput &input) const {
	auto &sink = input.global_state.Cast<HashJoinGlobalSinkState>();
	auto &ht = *sink.hash_table;

	// the filter covers the whole build side once Sink is done
	ht.SealBloomFilter();

	sink.temporary_memory_state->UpdateReservation(context);
	sink.external = sink.temporary_memory_state->GetReservation() < sink.total_size;
	if (sink.external) {
		// For external join we reduce the load factor, this may even prevent the external join altogether
		ht.load_factor = JoinHashTable::EXTERNAL_LOAD_FACTOR;
		// ...
```

Notice what is *not* here: no waiting, no polling other threads, no check of
"has everyone called `Sink` yet." By the time any thread's `Finalize` call
happens, the pipeline driver and its `Event` bookkeeping (§11.6) have already
guaranteed every `Combine` call finished. The operator gets to assume its
single-threaded moment simply *is* single-threaded.

The `SealBloomFilter()` call needs a word. A Bloom filter is a small bit array
that can say "this key is surely not in the table". A join that is expected to spill to disk fills its Bloom filter during
`Sink`, on every thread that runs `Sink`. `SealBloomFilter` marks the filter complete
(`src/include/duckdb/execution/join_hashtable.hpp:655-660`). It belongs in
`Finalize` for the same reason as the memory decision: only here is it
certain that every `Sink` call has added its keys. A filter that is not yet
sealed is never pushed into the probe-side scan, because it could wrongly
drop rows.

**`PhysicalHashAggregate`** (built out in full in Chapter 14 — this is
intentionally a light preview) follows the identical three-phase shape for a
completely different reason to be a sink: `Sink` accumulates each thread's own
partial hash table of `GROUP BY` groups; `Combine` merges those partial
per-thread tables into one shared table (or, above a size threshold,
repartitions them for a parallel merge — Chapter 14's subject); `Finalize`
does whatever final aggregate-function post-processing is needed (e.g.
converting a running sum-of-squares into a variance) before the *aggregate
itself* turns around and becomes a *source* for the next pipeline — which is
exactly the "some operators are both sink and source" case §11.5 covers.

## 11.4 The pipeline driver: who owns control flow

DiDi's own slides make the key architectural claim explicit before showing
any code: pipeline execution is driven by **one core loop, run by each
thread**, that owns *all* control flow — fetching from the source, pushing
through operators in order, and deciding whether the result reaches the sink
[8, slide 13]:

> In DuckDB, pipeline execution is driven by a core loop [...] run by each
> thread. This loop maintains operator state (thread-local as well as
> global), receives data chunks from operators (or the source) and pushes
> these chunks towards the downstream operator (or the sink), [and] contains
> control flow that detects whether **①** the source is exhausted or **②** an
> operator outputs an empty chunk.

The course then gives an explicit pseudocode "sketch" of this loop, itself
modeled after a real DuckDB maintainer's explanation
in DuckDB GitHub issue #1583 [30] — this is not an academic reconstruction, it is
DuckDB's own team describing DuckDB's own design [8, slide 14]:

```
// DiDi Chapter 6, slide 14 — "Sketch of DuckDB's Pipeline Driver
// (Run by each Thread)", reproduced faithfully from the slide

var operator[]     pipeline[0...P]        // pipeline to execute (operator count P+1)
var state[]        states[0...P]          // thread-local + global operator state
var data_chunk[]   intermediates[0...P-1] // rows output by source/operators (intermediate results)

source ← pipeline[0]                      // pipeline source and sink
sink   ← pipeline[P]                      // (pipeline[1...P-1] are operators)

for i in 0...P:                           // initial operator states
  states[i] ← pipeline[i].GetOperatorState(...)
for i in 0...P-1:                         // allocate intermediate chunks
  intermediates[i] ← data_chunk.Initialize(pipeline[i].return_type)

while true:
  intermediates[0] ← source.GetData(states[0], ...)   // get next data chunk from thread's morsel
  if (intermediates[0].size = 0)                       // no more rows?
    return FINISHED                                    //   yes, bail out

  reached_sink ← true
  for i in 1...P-1:                                    // execute operators in pipeline order
    intermediates[i] ← pipeline[i].Execute(intermediates[i-1], states[i])
    if (intermediates[i].size = 0)                      // operator reduced data chunk to size 0?
      reached_sink ← false                              //   yes, restart at source with next chunk
      break

  if reached_sink:                                       // did non-empty data chunk reach the sink?
    sink.Sink(intermediates[P-1], states[P], ...)         //   yes, collect rows at the sink
```

Read this loop once, closely: it fetches one chunk from the source; if that
chunk came back empty, the source is exhausted and the whole pipeline is
`FINISHED`. Otherwise it walks the chunk through every mid-pipeline operator
in order, in a straight `for` loop with no recursion — if *any* operator
reduces the chunk to zero rows (a `FILTER` that rejected everything, a
`PROBE` that matched nothing), the loop abandons this chunk immediately,
skips the sink for this iteration, and goes straight back to the source for
the next chunk. Only a chunk that survives every operator intact reaches
`sink.Sink()`. The driver, and only the driver, knows about "exhausted" and
"empty chunk" — no operator implementation needs to reason about either.

**This is the pattern real DuckDB code implements**, and it is worth seeing
the actual C++, because it confirms the pseudocode is not a simplification
that glosses over something essential — it is a faithful summary of a real,
if considerably more defensive, implementation. `PipelineExecutor::
ExecutePushInternal` is the function that pushes one chunk through the
operator chain and then, if it survived, into the sink:

```cpp
// src/parallel/pipeline_executor.cpp:560-612
OperatorResultType PipelineExecutor::ExecutePushInternal(DataChunk &input, ExecutionBudget &chunk_budget,
                                                         idx_t initial_idx) {
	D_ASSERT(pipeline.sink);
	if (input.size() == 0) {
		return OperatorResultType::NEED_MORE_INPUT;
	}

	// this loop will continuously push the input chunk through the pipeline as long as:
	// - the OperatorResultType for the Execute is HAVE_MORE_OUTPUT
	// - the Sink doesn't block
	// - the ExecutionBudget has not been depleted
	OperatorResultType result = OperatorResultType::HAVE_MORE_OUTPUT;
	do {
		if (&input != &final_chunk) {
			final_chunk.Reset();
			if (pipeline.operators.empty()) {
				final_chunk.Reference(input);
				result = OperatorResultType::NEED_MORE_INPUT;
			} else {
				// Execute and put the result into 'final_chunk'
				result = Execute(input, final_chunk, initial_idx);
				if (result == OperatorResultType::FINISHED) {
					return OperatorResultType::FINISHED;
				}
			}
		} else {
			result = OperatorResultType::NEED_MORE_INPUT;
		}
		auto &sink_chunk = final_chunk;
		if (sink_chunk.size() > 0) {
			StartOperator(*pipeline.sink);
			OperatorSinkInput sink_input {*pipeline.sink->sink_state, *local_sink_state, interrupt_state};
			auto sink_result = Sink(sink_chunk, sink_input);
			EndOperator(*pipeline.sink, nullptr);

			if (sink_result == SinkResultType::BLOCKED) {
				return OperatorResultType::BLOCKED;
			} else if (sink_result == SinkResultType::FINISHED) {
				FinishProcessing();
				return OperatorResultType::FINISHED;
			}
		}
		if (result == OperatorResultType::NEED_MORE_INPUT) {
			return OperatorResultType::NEED_MORE_INPUT;
		}
	} while (chunk_budget.Next());
	return result;
}
```

`if (sink_chunk.size() > 0)` guarding the call to `Sink` is the real-code
counterpart of the pseudocode's `if reached_sink: sink.Sink(...)` — a chunk
that got reduced to zero rows anywhere along the operator chain never reaches
the sink at all. The walk through the operator chain itself lives in a
sibling method:

```cpp
// src/parallel/pipeline_executor.cpp:744 (excerpted; full loop handles
// resumption of "HAVE_MORE_OUTPUT" operators via an explicit stack —
// elided here since that bookkeeping isn't part of the conceptual model)
OperatorResultType PipelineExecutor::Execute(DataChunk &input, DataChunk &result, idx_t initial_idx) {
	// ...
	while (true) {
		auto &current_chunk = /* result, or an intermediate chunk */;
		auto &prev_chunk = /* input, or the previous operator's chunk */;
		auto &current_operator = pipeline.operators[current_idx - 1].get();

		auto result = current_operator.Execute(context, prev_chunk, current_chunk,
		                                       *current_operator.op_state, *intermediate_states[...]);
		if (result == OperatorResultType::FINISHED) {
			FinishProcessing(NumericCast<int32_t>(current_idx));
			return OperatorResultType::FINISHED;
		}

		if (current_chunk.size() == 0) {
			// no output from this operator! go back and try to pull
			// data from the source again
			GoToSource(current_idx, initial_idx);
			continue;
		} else {
			current_idx++; // we got output! continue to the next operator
		}
	}
}
```

Why is the real function so much longer than the eleven-line pseudocode
sketch? Because production code has to handle things the teaching pseudocode
correctly leaves out to keep the idea visible: an operator or sink can return
`BLOCKED` (asynchronous I/O in flight — a Parquet read waiting on disk, a
remote table function waiting on a network response) and the *entire* driver
loop needs to suspend and resume later without losing its place, which is
what `in_process_operators` (a stack of "resume here" indices),
`remaining_sink_chunk`, and the whole `PipelineExecuteResult::INTERRUPTED`
plumbing exist for. An operator can also return `HAVE_MORE_OUTPUT` — more
output than fits in one chunk from one input chunk — which needs the *same*
input chunk re-fed on the next iteration rather than fetching a fresh one.
None of that changes the shape this section opened with: **the driver, not
any operator, decides when the source is exhausted, when a chunk dies
mid-pipeline, and when a sink gets called** — the extra code is entirely in
service of doing that correctly across blocking I/O and partial output, not a
different model.

Sometimes a blocked scan cannot *park* (step aside and let the thread do
other work). Then the I/O tasks attached to its `BLOCKED` result
(`AsyncResult`) run right where the scan is. They do not run one after the
other. The overload `AsyncResult::ExecuteTasksSynchronously(ClientContext &)`
(`src/parallel/async_result.cpp:175`), runs them side by side on a
`TaskExecutor`, and the calling thread helps. Scans reach it through
`TableFunctionInput::HandleBlocked(ClientContext &, AsyncResult &)`
(`src/include/duckdb/function/table_function.hpp:195-196`). The Parquet
reader, the multi-file wrapper, the table scan and `PhysicalTableScan`'s
fallback use it. For a remote Parquet file this means several column reads
go out at once instead of one round trip each. The model in this section does
not change; only the waiting is shorter.

The payoff, stated plainly: because `PipelineExecutor` owns every one of
these decisions, `Sink` and `Execute` implementations across the whole
codebase get to be almost stateless from a *control-flow* perspective. They
never check "is this the first call," "has my sibling operator finished,"
or "should I ask my child for more input" — all of that has already been
decided for them by the time they are invoked. §11.1's contrast between the
tangled Volcano `HashJoin::GetChunk()` and the trivial `Sink`/`Execute` split
is not a simplified illustration; it is the literal architectural
consequence of moving control flow out of the operators and into this one
driver loop.

## 11.5 Source parallelism: splitting the scan across threads

A `Pipeline`'s source is where DuckDB decides *how many threads can usefully
start at once*, before a single row has moved. The relevant hooks, all on
`PhysicalOperator` (§11.3.2), are:

```cpp
virtual bool IsSource() const { return false; }
virtual bool ParallelSource() const { return false; }
//! Whether this source creates partitioned work that is not bounded by its input chunks
virtual bool HasSourceTasks() const { return false; }
//! How this source manages parallelism
virtual TableFunctionParallelism SourceParallelism() const;
virtual OperatorPartitionData GetPartitionData(ExecutionContext &context, DataChunk &chunk,
                                               GlobalSourceState &gstate, LocalSourceState &lstate,
                                               const OperatorPartitionInfo &partition_info) const;
```

`IsSource()` is the simplest: does this operator sit at the start of a
pipeline at all. `ParallelSource()` answers whether *more than one thread* is
allowed to call `GetData` concurrently against the same `GlobalSourceState`
at all — `PhysicalTableScan` returns `true` unconditionally unless the
underlying table function is a table-in/table-out function fed only a single
input row, which by construction cannot be split:

```cpp
// src/execution/operator/scan/physical_table_scan.cpp:402-413
bool PhysicalTableScan::ParallelSource() const {
	if (!function.function) {
		// table in-out functions cannot be executed in parallel as part of a PhysicalTableScan
		// since they have only a single input row
		return false;
	}
	return true;
}

TableFunctionParallelism PhysicalTableScan::SourceParallelism() const {
	return function.parallelism;
}
```

`SourceParallelism()` reports *which kind* of parallelism the underlying
table function manages, via an enum with three values
(`src/include/duckdb/function/table_function.hpp:31-36`):

```cpp
//! Controls how a table function manages parallelism.
enum class TableFunctionParallelism : uint8_t {
	SELF_MANAGED_PARALLELISM = 0, //! Source handles its own parallelism (default)
	SEQUENTIAL = 1,               //! Sequential source, benefits from external parallelization
	FORCE_SINGLE_THREADED = 2     //! Sequential source, prefers single-threaded execution
};
```

A base-table scan (`PhysicalTableScan` over `DataTable`, e.g. `orders` and
`lineitem` in Query A) is `SELF_MANAGED_PARALLELISM`: the table's own row
groups are the natural unit of splittable work, and the scan's underlying
table function knows how to divide those row groups among threads without
the pipeline driver needing to understand row-group internals at all. That
division is exposed through `GetPartitionData`, which delegates straight to
the table function's own `get_partition_data` callback:

```cpp
// src/execution/operator/scan/physical_table_scan.cpp:258-268
OperatorPartitionData PhysicalTableScan::GetPartitionData(ExecutionContext &context, DataChunk &chunk,
                                                          GlobalSourceState &gstate_p, LocalSourceState &lstate,
                                                          const OperatorPartitionInfo &partition_info) const {
	D_ASSERT(SupportsPartitioning(partition_info));
	D_ASSERT(function.get_partition_data);
	auto &gstate = gstate_p.Cast<TableScanGlobalSourceState>();
	auto &state = lstate.Cast<TableScanLocalSourceState>();
	TableFunctionGetPartitionInput input(bind_data.get(), state.local_state.get(), gstate.global_state.get(),
	                                     partition_info);
	return function.get_partition_data(context.client, input);
}
```

Concretely, for a scan over `lineitem` (a table with, say, dozens of row
groups on disk — Chapter 15 covers row-group storage in depth): each thread
running the probe pipeline asks the shared `TableScanGlobalSourceState` for
"the next unclaimed row group," scans it, produces chunks from it, and asks
for another when it runs dry — a form of dynamic load balancing where no
thread is pre-assigned a fixed slice of the table up front. This is the
concrete mechanism §11.8 is about to name explicitly.

`HasSourceTasks()` covers a different case: a source whose splittable work
*isn't* neatly bounded by the chunks it will eventually emit — e.g. a source
that first has to discover *how much* work exists (globbing a directory of
Parquet files, say) before that work can be handed out. `HasSourceTasks()`
lets such a source say "I create partitioned work units independent of my
output chunking," which the scheduler treats differently from ordinary
chunk-bounded scan parallelism.

### 11.5.1 An operator that is both sink and source

Not every pipeline-breaking operator disappears after its `Finalize`. Some
become the *source* of the very next pipeline, reading back out of the state
they just finished building. DiDi's course states this as a general rule
about parallelism-aware sources, using exactly the operator this book has
been building toward using as its own running illustration
[8, slide 11]:

> Some operators are sinks *and* sources: `HASH_GROUP_BY` serves rows from
> its aggregate hash table.

`PhysicalHashAggregate` (Chapter 14 in full) is the sink for the pipeline
that consumes `GROUP BY` input — its `Sink`/`Combine`/`Finalize` build the
grouped hash table exactly as §11.3.4 sketched. Once `Finalize` has run, that
*same* operator becomes the source of the *next* pipeline: its
`GetDataInternal` scans the finished hash table's buckets and emits one
chunk of finished groups at a time, parallelizable across threads the same
way a table scan's row groups are (each thread claims a disjoint slice of
buckets to drain). This dual role is why a `MetaPipeline`'s "child pipeline
built last" rule from §11.2.1 exists in the first place: the source-side
pipeline for `HASH_GROUP_BY` (or, per that same doc comment, a `FULL OUTER
JOIN`'s hash join scanning for unmatched build rows) has to be added to the
`MetaPipeline` *after* every other pipeline that depends on the shared sink
having finished, precisely because reading the finished state safely
requires that state to already be finished.

## 11.6 Pipeline dependencies and Events

Every dependency this chapter has described so far — probe waits for build,
`HASH_GROUP_BY`'s source pipeline waits for its own sink pipeline — has to be
enforced by *something*, or two threads could race: one thread starting the
probe pipeline while another is still mid-`Sink` on the build side. That
enforcement is the job of `PipelineDependency` (already introduced in
§11.2) together with a family of `Event` subclasses.

`PipelineDependencyType` distinguishes two strengths of ordering
(`src/include/duckdb/parallel/pipeline.hpp:31`):

```cpp
enum class PipelineDependencyType : uint8_t { REQUIRED, OPTIONAL_DEPENDENCY };
```

- **`REQUIRED`** — the dependent pipeline is not merely *slower* without the
  dependency having run first, it is *incorrect*: the probe pipeline reading
  an unfinished hash table would return wrong join results, not just stale
  ones. The scheduler will not schedule a single task from a `REQUIRED`-
  dependent pipeline until every pipeline it depends on has finished.
- **`OPTIONAL_DEPENDENCY`** — an ordering preference that doesn't affect
  correctness if violated, used for cases where running one pipeline before
  another is merely a good idea (better cache locality, avoiding redundant
  work) rather than a hard requirement.

`Event` (`src/include/duckdb/parallel/event.hpp`) is the base class for the
actual "is this stage of the DAG done yet" bookkeeping:

```cpp
// src/include/duckdb/parallel/event.hpp:19-40
class Event : public enable_shared_from_this<Event> {
public:
	explicit Event(Executor &executor);
	virtual ~Event() = default;

public:
	virtual void Schedule() = 0;
	//! Whether an event with no scheduled tasks should automatically finish after scheduling
	virtual bool AutoFinishWithoutTasks() const { return true; }
	//! Called right after the event is finished
	virtual void FinishEvent() { }
	//! Called after the event is entirely finished
	virtual void FinalizeFinish() { }

	void FinishTask();
	void Finish();
	// ...
};
```

`PipelineInitializeEvent`, `PipelineFinishEvent`, and `PipelineCompleteEvent`
each specialize this for one moment in a pipeline's life (all under
`src/include/duckdb/parallel/`): initializing a pipeline's global states
before any task runs against it, running that pipeline's `Combine`/`Finalize`
once every `PipelineTask` for it has finished, and marking the whole
pipeline — and, transitively, anything that only depended on it — as
complete so dependent pipelines' events can themselves fire. `Executor`
(§11.7) is the object that actually owns this whole event DAG for one query
and drives it forward as tasks report back to the scheduler.

The order of the last two steps of a task matters. A `PipelineTask` tells
its event "I am done" by calling `event->FinishTask()`. When the last task
of a pipeline does that, another thread may start the sink's `Finalize` right
away. So the task must let go of everything it still holds *before* it
reports:

```cpp
// src/parallel/pipeline.cpp:69-73
	// release the local states and their buffer pins before the sink can be finalized
	pipeline_executor.reset();
	event->FinishTask();
	return TaskExecutionResult::TASK_FINISHED;
}
```

`pipeline_executor.reset()` destroys the task's `PipelineExecutor`, and with
it the thread-local operator states. Those states can hold buffer *pins*. A
pin keeps a memory block in place while someone reads it (Chapter 14
covers the buffer manager). If a pin is still alive while `Finalize` runs,
`Finalize` may fail. For example, an `INSERT` into a table created in the same
transaction flushes its partial blocks in `Finalize`. With the two lines in the
other order, the last task could report "done" while it still held pins, and
the flush could fail with "BlockManager::ConvertToPersistent in destructive mode
- cannot be called for block 3 as old_block has multiple readers active".
`test/sql/parallelism/intraquery/test_transaction_local_parallelism.test` is a
test that could hit this.

For Query A's join alone, the dependency picture looks like this — two
pipelines, one `REQUIRED` edge, matching §11.2.1's `MetaPipeline` diagram:

```
   Pipeline 1 (build: scan orders → HashJoin.Sink)
        │
        │  REQUIRED  (Finalize of 1 must run before any task of 2 starts)
        ▼
   Pipeline 2 (probe: scan lineitem → HashJoin.Execute → next sink)
```

§11.9 extends this same diagram across Query A's *entire* plan, where the
chain grows to four pipelines.

## 11.7 The TaskScheduler: one thread pool, many queries

Every `PipelineTask` produced by every `Executor`, for every concurrently
running query on every connection, is submitted to a single, process-wide
`TaskScheduler` (`src/include/duckdb/parallel/task_scheduler.hpp`). This is
worth stating plainly because it corrects an intuitive but wrong mental
model: a query does **not** get its own dedicated pool of threads. Chapter
1's system diagram already flagged this ("One `TaskScheduler`, many
queries"); this chapter is where the mechanism behind that claim lives.

The scheduler's worker threads pull work from a lock-free, multiple-producer
multiple-consumer queue — the vendored `moodycamel::ConcurrentQueue`
(`third_party/concurrentqueue`, wrapped by
`src/include/duckdb/parallel/concurrentqueue.hpp`, per
`duckdb-source-architecture-recon.md` §6b). Any thread with spare capacity —
regardless of which query's `Executor` produced the task — can pop the next
`PipelineTask` off this shared queue. This is what makes the scheduler a
genuinely shared resource rather than a per-query abstraction with a
misleading name: ten simultaneous connections running ten different queries
are, under load, all drawing from the same worker pool, competing for the
same cores.

### 11.7.1 Fair turns between queries: the `ConsumerToken`

A shared queue raises a fairness question. If one heavy query keeps adding
tasks, can a short query that arrived first still get a turn?

Inside the queue, each query has its own sub-queue, called a *producer*
(the query enqueues through a `ProducerToken`). When a worker asks for "any
task", `moodycamel` has to pick a producer first. Without extra help it looks
at only the first three non-empty producers and takes the one with the most
tasks. New producers are put at the front of that list, and empty producers
are reused from the front too. So a newer query could be served ahead of an
older one again and again.

So every worker thread dequeues through a `ConsumerToken`
(`src/include/duckdb/parallel/task_scheduler_token.hpp`):

```cpp
// src/include/duckdb/parallel/task_scheduler_token.hpp:41-52
//! Dequeues tasks from any producer, rotating across producers so that every query gets a turn
struct ConsumerToken {
public:
	explicit ConsumerToken(array<unique_ptr<TaskSchedulerQueue>, TASK_SCHEDULER_TYPE_COUNT> &queues);
	~ConsumerToken();

public:
	QueueConsumerToken &GetQueueConsumerToken(TaskSchedulerType pool_type);

private:
	array<unique_ptr<QueueConsumerToken>, TASK_SCHEDULER_TYPE_COUNT> tokens;
};
```

The token remembers which producer the worker is reading from. The worker
keeps taking tasks from that producer. After a fixed number of tasks (the
*quota*), it moves on to the next producer, so every query gets a turn.
`moodycamel`'s default quota is 256. DuckDB lowers it to 16, because one
DuckDB task is much heavier than a typical queue item:

```cpp
// src/parallel/task_scheduler_queue.cpp:18
	static constexpr uint32_t EXPLICIT_CONSUMER_CONSUMPTION_QUOTA_BEFORE_ROTATE = 16;
```

The worker loop creates one token per thread
(`src/parallel/task_scheduler.cpp:200-210`). External threads that help
with tasks (`ExecuteTasks`, used by `duckdb_execute_tasks`) use a token too.
There is no plain `TaskSchedulerQueue::Dequeue` without a token. The effect is
easy to measure: with one worker, a short query sent before three heavy ones
finishes in 64 ms with the token, against 449 ms without it.

A second, smaller detail concerns errors on worker threads. A worker that
throws out of its own bookkeeping (not out of a task) would end the whole
process. So the worker loop catches this. When it cannot allocate, it
waits and continues with the task it holds, up to `MAX_TASK_RETRIES` (1000) times
(`src/parallel/task_scheduler.cpp:187`, `:203-205`). A task error that is an
out-of-memory error is reported to the query as an error instead of
aborting (`test/api/test_task_error_out_of_memory.cpp`,
`test/api/test_worker_exception.cpp`).

This shared-pool design is also why DiDi's course treats "should independent
pipelines run in parallel" as a real, debated design question rather than an
obvious yes. Query A's own plan happens not to raise this question (§11.9
shows its four pipelines form one strict chain), but DiDi's larger worked
example — a three-way join feeding a `HASH_GROUP_BY`, corresponding to
TPC-H Q10 — does have two genuinely independent build-side pipelines before
the rest of the plan can proceed, and the course lays out the tradeoff
explicitly [8, slide 10]:

> Independent pipelines ① ‖ ② can be run in parallel:
> - 👍 This promises reduced latency.
> - 👎 Independent pipelines are rather rare. Does the implementation of the
>   required logic pay off?
> - 👎 Amount of work in pipelines ① and ② may differ significantly.
> - 👎 Reduces cache locality.
> - 💡 Rather invest more cores in a single pipeline.

The conclusion — favor giving more threads to whichever pipeline is
currently runnable over building machinery to run two unrelated pipelines
side by side — is a direct consequence of the shared-scheduler design: extra
cores handed to one wide, well-load-balanced pipeline (many threads, each
independently pulling row groups per §11.5) are close to free, cache-friendly
throughput, whereas splitting those same cores across two pipelines with
unequal amounts of work and no shared working set mostly just adds
bookkeeping and cache misses for a latency win that, per the slide, rarely
materializes in practice.

## 11.8 A terminology aside: "morsel-driven parallelism" is not DuckDB's word

The mechanism §11.5 just described — a source hands out unclaimed,
disjointly-sized chunks of underlying work (row groups, file splits) to
whichever thread asks next, rather than statically pre-partitioning the
table across a fixed number of threads up front — is exactly what the
database-systems literature calls **morsel-driven parallelism**, a term
introduced by Leis et al. in their 2014 paper on the HyPer engine [31]. It would be
reasonable to expect this term to appear somewhere in DuckDB's own execution
or scheduling code, describing exactly the mechanism it names.

It does not. A search of `src/execution` and `src/parallel` — the two
directories that between them hold every class this chapter has discussed —
turns up **zero** occurrences of the word "morsel," in code, comments, or
identifiers. DuckDB's own vocabulary for this mechanism is the one this
chapter has been using throughout: `GetPartitionData`, `SourceParallelism`,
`HasSourceTasks`, row groups, chunks. "Morsel-driven parallelism" is a label
the outside world — blog posts, conference talks, comparison papers — has
applied to DuckDB's design because the *mechanism* matches HyPer's
morsel-driven executor closely enough to invite the comparison. It is not a
name DuckDB's own source code, or its own team, uses for itself.

What makes this genuinely interesting trivia, rather than a pedantic
footnote, is where the word *does* turn up: inside the very pseudocode this
chapter quoted in §11.4. DiDi's slide 14 — written by someone who has read
DuckDB's actual source and talked to the DuckDB Labs team, and whose
pseudocode is explicitly modeled on a DuckDB maintainer's own GitHub issue —
glosses the line `intermediates[0] ← source.GetData(states[0], ...)` with the
comment *"get next data chunk from thread's morsel."* The academic
vocabulary crept into the teaching material's own explanatory gloss, as a
convenient shorthand for readers who already know the term from the wider
literature — even though the code being explained, and the maintainers who
wrote it, never reach for that word themselves. It is a small, honest
illustration of how an outside research vocabulary and a project's internal
vocabulary can diverge while describing the exact same mechanism, sometimes
within a single sentence of each other.

## 11.9 Worked example: Query A decomposed into pipelines

Everything above can now be assembled into one worked decomposition of Query
A's physical plan, following the same reasoning DiDi applies to its own TPC-H
examples [8, slides 9–10]: find every sink (pipeline breaker), and every source,
and read the dependency edges off which sink's `Finalize` each source needs
to have already run.

A reminder of the plan (Chapter 1 / Chapter 9):

```
PhysicalResultCollector
  └─ PhysicalTopN                    (ORDER BY total DESC LIMIT 10)
       └─ PhysicalHashAggregate      (GROUP BY o.custkey; SUM(l.extendedprice))
            └─ PhysicalHashJoin      (o.orderkey = l.orderkey)
                 ├─ PhysicalTableScan (orders)     ← smaller table: build side
                 └─ PhysicalTableScan (lineitem,   ← larger table: probe side
                                       filter pushed into the scan itself)
```

*A note on this section's status, stated as plainly as the front matter
asks:* what follows is **this book's own reasoned reconstruction**, applying
the Sink/Source/Operator rules just covered, not a transcript of an actual
`EXPLAIN ANALYZE` run against this checkout. The build/probe assignment
follows DiDi's own stated rule — "DuckDB chooses the smaller input to be on
the `BUILD` side" [8, slide 9] — and TPC-H's `orders` table is
several times smaller than `lineitem` at any scale factor, so `orders` is
assigned to build. Four operators in this tree are sinks (pipeline
breakers): `PhysicalHashJoin` (build side only), `PhysicalHashAggregate`,
`PhysicalTopN` (DiDi's own `TOP_N` node is drawn as a pipeline breaker with
bars on both sides [8, slide 10] — a bounded top-*k* heap still needs every
input row before it can be certain which 10 rows survive), and
`PhysicalResultCollector` (DiDi's generic root "`QUERY`" sink [8, slide 8]: "Root
operator `QUERY` acts as a sink that collects all incoming rows"). Three of
those four sinks — `HashAggregate`, `TopN`, and implicitly the join's
probe-side hash-table-scan case from §11.5.1 — also double as the *source*
of the next pipeline once their own `Finalize` has run.

That gives four pipelines:

```
Pipeline 1  (MetaPipeline: sink = PhysicalHashJoin, JOIN_BUILD)
┌────────────────────────────────────────────────────────────┐
│ source:     PhysicalTableScan(orders)                       │
│ operators:  (none — no predicate on orders in Query A)       │
│ sink:       PhysicalHashJoin.Sink   (insert into build HT)    │
└────────────────────────────────────────────────────────────┘
        │ REQUIRED: Pipeline 2 cannot start until Pipeline 1's
        │ PhysicalHashJoin.Finalize has built the probeable HT
        ▼
Pipeline 2  (MetaPipeline: sink = PhysicalHashAggregate)
┌────────────────────────────────────────────────────────────┐
│ source:     PhysicalTableScan(lineitem, shipdate < ... pushed│
│             into the scan)                                   │
│ operators:  PhysicalHashJoin.Execute   (probe — mid-pipeline) │
│ sink:       PhysicalHashAggregate.Sink (accumulate groups)    │
└────────────────────────────────────────────────────────────┘
        │ REQUIRED: Pipeline 3 cannot start until Pipeline 2's
        │ PhysicalHashAggregate.Finalize has finished every group
        ▼
Pipeline 3  (MetaPipeline: sink = PhysicalTopN)
┌────────────────────────────────────────────────────────────┐
│ source:     PhysicalHashAggregate.GetData (scan finished       │
│             groups — the sink-AND-source case, §11.5.1)       │
│ operators:  (none)                                             │
│ sink:       PhysicalTopN.Sink  (maintain bounded top-10 heap)  │
└────────────────────────────────────────────────────────────┘
        │ REQUIRED: Pipeline 4 cannot start until Pipeline 3's
        │ PhysicalTopN.Finalize has produced the final sorted 10
        ▼
Pipeline 4  (MetaPipeline: sink = PhysicalResultCollector)
┌────────────────────────────────────────────────────────────┐
│ source:     PhysicalTopN.GetData  (emit the finished top-10)   │
│ operators:  (none)                                             │
│ sink:       PhysicalResultCollector.Sink (gather result rows) │
└────────────────────────────────────────────────────────────┘
```

Dependency chain: **1 < 2 < 3 < 4**, a strict linear order with no
independent, parallelizable pair anywhere in it — unlike DiDi's own
three-way-join example (§11.7), where two build pipelines genuinely can run
side by side, Query A's single two-way join gives the scheduler nothing to
run concurrently *across* pipelines. All of the parallelism available for
this query is *within* each pipeline: many threads scanning disjoint row
groups of `orders` inside Pipeline 1, many threads scanning disjoint row
groups of `lineitem` and independently probing the (by then finalized, safe
to read concurrently) hash table inside Pipeline 2, and so on — precisely
the "invest cores in one pipeline" preference §11.7 quoted DiDi drawing out
as a general design lesson.

Two details worth double-checking against everything above, because they are
exactly the kind of thing that's easy to get backwards:

- **Why is the join only a sink in Pipeline 1, and only an operator (not a
  sink) in Pipeline 2?** Because `PhysicalHashJoin` plays two different
  roles for its two children. Against the build child (`orders`) it behaves
  as a pipeline-ending sink (§11.1, §11.3.4). Against the probe child
  (`lineitem`) it behaves as an ordinary mid-pipeline operator (`Execute`,
  §11.3.1) — probing an already-finalized hash table and passing matched
  rows onward is a stateless-per-call transform from the driver's point of
  view, no different in shape from `PhysicalFilter`. The *same C++ class* is
  a sink from one child's perspective and an operator from the other's,
  which is exactly the split §11.1's side-by-side pseudocode was built to
  motivate.
- **Why does `PhysicalHashAggregate` get its own pipeline as a source
  (Pipeline 3), rather than being read directly out of Pipeline 2?** Because
  its `Finalize` — single-threaded, run once all of Pipeline 2's `Combine`
  calls are done — is the earliest safe moment to start reading finished
  groups back out, and per the `REQUIRED` dependency rule (§11.6) no
  pipeline is allowed to start before the pipeline it depends on has
  reached that point. This is the identical shape to the join's
  build/probe split one level up the tree, just triggered by a `GROUP BY`
  instead of a join condition.

### 11.9.1 Query progress, and the pipeline that never runs

The `Executor` also holds one more pipeline that the diagram above leaves
out. `PhysicalResultCollector` is the root of the plan, so it is also the
*source* of a root pipeline. That pipeline never runs inside the engine.
The client reads the collected rows later, outside the pipeline machinery.

This matters for the progress bar. `Executor::GetPipelinesProgress` adds up
"rows done" and "rows expected" over all pipelines. The root pipeline always
reports 0 done out of 1 expected. (`Pipeline::GetDetailedProgress` raises an
estimate of 0 rows to 1.) So a query whose planner expects N rows could only
reach N / (N + 1). With a small estimate this would be very visible, for
example:

```sql
SET enable_progress_bar = true;
CREATE TABLE tbl AS SELECT i, (i * 7919) % 1000000 AS j FROM range(1000000) t(i);
SELECT i FROM tbl WHERE j = 12345;  -- scans every row; would peak at 50%
```

So the loop skips that pipeline:

```cpp
// src/parallel/executor.cpp:653-665
	for (auto &pipeline : pipelines) {
		if (pipeline->GetSource()->type == PhysicalOperatorType::RESULT_COLLECTOR) {
			// this pipeline only holds the result collector as the root of the plan: it never runs, so including it
			// would add a row that never completes to the total
			continue;
		}
		ProgressData p;
		if (!pipeline->GetProgress(p)) {
			count_invalid++;
		} else {
			progress.Add(p);
		}
	}
```

Without this, a full scan with a selective filter could stop at 50–67% even
after all work was done. The test is in C++ (`test/api/test_progress_bar.cpp`),
because query progress cannot be read from SQL.

## 11.10 Summary and what's next

This chapter has covered, in order: why DuckDB's push-based pipeline-driver
model exists at all and what specifically it fixes about the pull-based
Volcano model for operators like joins (§11.1); the precise `Pipeline` and
`MetaPipeline` classes that give this model its shape, including the
build-before-probe and child-pipelines-built-last invariants baked into
`MetaPipeline`'s own construction rules (§11.2); the full, verbatim
Sink/Source/Operator interface on `PhysicalOperator`, with `PhysicalHashJoin`
and `PhysicalHashAggregate` as running illustrations of the
Sink/Combine/Finalize contract's parallel-safety guarantees (§11.3); the
real `PipelineExecutor` driver loop that owns all control flow, shown
alongside DiDi's own pseudocode sketch to establish that the two describe
the same mechanism at different levels of defensive detail (§11.4); the
source-parallelism hooks (`GetPartitionData`, `SourceParallelism`,
`HasSourceTasks`) that let a scan hand out row-group-sized work dynamically,
and the sink-and-source dual role that `HASH_GROUP_BY` (and, per
`meta_pipeline.hpp`'s own comment, `FULL OUTER JOIN`) plays (§11.5); the
`PipelineDependency`/`Event` bookkeeping that turns "pipeline B needs
pipeline A's `Finalize` to have run" from a hopeful convention into an
enforced scheduling constraint (§11.6); the single, process-wide
`TaskScheduler` that every concurrent query's `Executor` shares, the
`ConsumerToken` that gives every query a fair turn, and the design argument for preferring wide single-pipeline parallelism over
running independent pipelines side by side (§11.7); the honest terminology
gap between "morsel-driven parallelism" as an external label and DuckDB's
own, morsel-free vocabulary (§11.8); and a full, explicitly-reasoned
pipeline decomposition of Query A into its four constituent pipelines and
their strict dependency chain, plus the root pipeline that never runs and is
left out of query progress (§11.9).

What this chapter has deliberately deferred: *how* a chunk moves through any
single operator's `Execute` or `Sink` at the byte level — that is `DataChunk`
and vector representations, Chapter 10's subject, assumed throughout this
chapter as the unit flowing between pipeline stages. *How* an expression like
`o.orderkey = l.orderkey` or `SUM(l.extendedprice)` actually gets evaluated
once a chunk reaches an operator that needs it — that is the
`ExpressionExecutor`, Chapter 12. And *why* DuckDB interprets that expression
tree per chunk rather than compiling it, which this chapter's emphasis on a
generic, reusable driver loop should already make plausible as a design
choice — is argued in full in Chapter 13. Chapter 14 returns to
`PhysicalHashAggregate` and expands this chapter's brief sink/source sketch
of it into DuckDB's complete two-phase, partitioned hash-aggregation
algorithm. Chapter 19 traces Query A through every stage this book covers,
pipelines included, against one single, fully worked run.
