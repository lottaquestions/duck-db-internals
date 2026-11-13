# Chapter 10 — Vectors and Data Chunks

## 10.1 Why batches, and why 2048 rows

Chapter 1 already gave away the punch line for this chapter in passing: "each
pipeline runs by pulling `DataChunk`s — batches of up to 2048 rows across all
needed columns — from its source, pushing each chunk through a chain of
operators." This chapter is what that sentence expands into once you take it
seriously, because almost nothing in Chapters 11–13 makes sense without it.
`DataChunk` and `Vector` are the single most load-bearing pair of types in the
whole engine — every operator's `Execute`, `Sink`, and `GetData` method takes
one or produces one, and the design decisions baked into their C++ layout are
the reason DuckDB is fast in the specific way it's fast.

Start with the two alternatives DuckDB deliberately did not choose, because
the batch model only makes sense as a reaction to both of their failure modes.

**Row-at-a-time interpretation** — the classic Volcano/iterator model, where
`Next()` is called once per output row and every operator does its work on a
single tuple before handing it to its parent — has an overhead problem that
has nothing to do with the actual arithmetic being performed. Evaluating
`l.shipdate < DATE '1998-01-01'` is one comparison instruction. But reaching
that comparison in a row-at-a-time interpreter usually means a virtual
function call into the expression tree, a type dispatch, and a `Next()` call
up through however many stacked iterators sit between the scan and this
predicate — for *every single row*. When there are only microseconds of real
work to do per row, framework overhead like this dominates the actual cost,
often by an order of magnitude. This is the "interpretation overhead" problem,
and it scales with the number of *rows*, not the number of *values*.

**Column-at-a-time full materialization** — the opposite extreme, where each
operator fully computes an entire intermediate column before the next operator
starts (this is roughly what a system like MonetDB's classic vectorized
execution did at large batch sizes, and it's the natural thing to do if you
think in terms of columnar storage without also thinking about cache) — solves
the interpretation-overhead problem (one function call handles a whole
column), but trades it for a different one: **an intermediate the size of the
whole table doesn't fit in cache.** If `lineitem` has six million rows, a
fully materialized intermediate column of `extendedprice` values is six
million times 8 bytes — 48 MB — before you've even gotten to the next
operator. That's already past the "¼ – 16 MB" typical L2 cache size cited in
Chapter 2's memory-hierarchy table (§2.1), meaning every operator in the chain
re-reads the *previous* operator's output from RAM, not L2 — paying something
close to the "≈100 ns / 36-second-human-scale" RAM latency Chapter 2 spent a
whole chapter justifying you should avoid, on every single column, at every
single operator boundary.

DuckDB's answer sits between these two: **process a fixed-size batch of rows,
across every needed column, at a time** — call one function per batch instead
of per row (fixing the interpretation-overhead problem), but keep that batch
small enough that the whole thing — not just one column, the *entire
horizontal slice* — stays resident in L2 while every operator in the pipeline
touches it (fixing the materialization problem). The batch size that makes
this work is `STANDARD_VECTOR_SIZE`:

```cpp
// src/include/duckdb/common/vector_size.hpp:16-20
#define DEFAULT_STANDARD_VECTOR_SIZE 2048U
...
#ifndef STANDARD_VECTOR_SIZE
#define STANDARD_VECTOR_SIZE DEFAULT_STANDARD_VECTOR_SIZE
#endif
```

2048 is not a round number chosen for aesthetic reasons — it's chosen for
arithmetic reasons that follow directly from Chapter 2's table. A `BIGINT`
column at 2048 rows is 16 KB. Even a moderately wide horizontal slice —
say a dozen fixed-width columns of Query A's `lineitem` scan — sits at a few
hundred KB, comfortably inside the "¼ – 16 MB" L2 range for the whole
`DataChunk`, not just one column of it. That's the design intent in one
sentence: **a `DataChunk` is sized so an entire batch of every column an
operator needs can live in L2 cache simultaneously**, so a pipeline of several
operators chained together on one chunk pays L2 latency (Chapter 2's "1
second" on the human-scale clock) at every step instead of RAM latency
("36 seconds") or worse.

**The morsel.** One `DataChunk` is the unit an *operator* consumes; a
**morsel** is the coarser-grained unit of work handed to one *thread* — DiDi's
term for a batch of consecutive chunks a single worker pulls and processes
before going back to the scheduler for more, sized (per DiDi Chapter 7,
"Vectorized Query Execution" [9]) at 60 chunks, i.e. 122,880 rows. This chapter
only needs the definition, not the mechanics: how morsels get assigned to
threads, how a `Pipeline` is broken into `PipelineTask`s, and how DuckDB's
scheduler decides which thread runs which morsel of which table are all
Chapter 11's job. One honesty note worth flagging now rather than later: the
literal word "morsel" does not appear as a symbol anywhere in this source
tree. The closest real quantity is `PipelineTask::PARTIAL_CHUNK_COUNT`:

```cpp
// src/include/duckdb/parallel/pipeline.hpp:49
static constexpr const idx_t PARTIAL_CHUNK_COUNT = 50;
```

used at `src/parallel/pipeline.cpp:48` as the number of chunks a
`PipelineExecutor` processes in one call before yielding control back to the
scheduler for a cooperative check (can this task be paused, is there
higher-priority work, etc.). It is a related idea — a bounded run of chunks
processed together — but it is a *scheduling quantum*, not necessarily the
same number DiDi's slides use for the pedagogical "morsel," and Chapter 11 is
where that distinction (and the actual scheduling algorithm) gets sorted out
properly. Nothing else in this chapter depends on getting that number exactly
right; what matters here is only the shape: **chunk = one operator call's
worth of rows, morsel = several chunks' worth of rows given to one thread at a
time, pipeline = how morsels flow through the whole operator tree (Ch. 11).**

## 10.2 `DataChunk` and `Vector`, precisely

A `DataChunk` is a **horizontal slice** of a relation: one `Vector` per output
column, and every `Vector` in the chunk has the same logical row count. Here
is the class, with the parts that matter for this chapter kept in and the rest
elided:

```cpp
// src/include/duckdb/common/types/data_chunk.hpp:44-202 (elided)
class DataChunk {
public:
	//! The vectors owned by the DataChunk.
	vector<Vector> data;

public:
	inline idx_t size() const {
		if (count.IsValid()) {
			return count.GetIndex();
		}
		return DeriveSize();
	}
	inline idx_t ColumnCount() const {
		return data.size();
	}
	//! The number of rows the vectors of this chunk were initialized to hold
	DUCKDB_API idx_t GetCapacity() const;
	//! Verify all child vectors have the expected cardinality
	void CheckCardinality(idx_t count_p);
	//! Sets the cardinality of all child vectors of this chunk
	void SetChildCardinality(idx_t count_p);
	//! Sets only the logical cardinality; child vector sizes must be synchronized separately.
	void SetCardinalityUnsafe(idx_t count_p) {
		this->count = count_p;
	}
	...
	//! Returns the uncompressed size of the data elements stored in this data chunk
	idx_t GetDataSize() const;
private:
	optional_idx count;
	//! Vector caches, used to store data when ::Initialize is called
	vector<VectorCache> vector_caches;
};
```

`vector<Vector> data` (`data_chunk.hpp:51`) is the whole story of "horizontal
slice": a `DataChunk` is literally nothing but a list of `Vector`s plus one
shared row count. `size()` (`data_chunk.hpp:54-59`) — how many logical rows the
chunk currently holds — is stored once, in `count`, not once per vector; every
`Vector` in `data` is expected to agree with it (`CheckCardinality`,
declared at `data_chunk.hpp:66`, is the debug-mode assertion of exactly that
invariant). The class comment above the definition
(`data_chunk.hpp:26-43`) states the design intent directly: a `DataChunk`
"holds a set of vectors that all have the same length," and critically, "the
underlying vectors can become referencing vectors to other chunks as well
(i.e. in the case an operator does not alter the data, such as a `Filter`
operator which only adds a selection vector)" — which is exactly §10.4's
subject, and exactly what Query A's `WHERE l.shipdate < DATE '1998-01-01'`
does at the bottom of §10.8.

`Vector` is the single-column batch:

```cpp
// src/include/duckdb/common/types/vector.hpp:35-45, 181-241 (elided)
class Vector {
public:
	...
	//! Create a vector of size capacity (non-standard)
	DUCKDB_API explicit Vector(LogicalType type, idx_t capacity = STANDARD_VECTOR_SIZE,
	                           VectorDataInitialization initialize = VectorDataInitialization::UNINITIALIZED);
	...
public:
	// Getters
	inline VectorType GetVectorType() const {
		auto &buffer_ref = GetBufferRef();
		if (!buffer_ref) {
			return VectorType::FLAT_VECTOR;
		}
		return buffer_ref->GetVectorType();
	}
	inline const LogicalType &GetType() const {
		return type;
	}
	...
protected:
	//! The type of the elements stored in the vector (e.g. integer, float)
	LogicalType type;
	//! The main buffer holding the data of the vector
	mutable buffer_ptr<VectorBuffer> buffer;
};
```

Two fields, full stop: `type` (a `LogicalType` — what SQL type this column
is: `INTEGER`, `VARCHAR`, `DECIMAL(12,2)`, ...) and `buffer` (a
`buffer_ptr<VectorBuffer>` — where the actual bytes live and how they're
physically laid out). Everything else on `Vector` is a method that operates
on those two fields. `GetVectorType()` (`vector.hpp:182-188`) is worth reading
carefully: the physical representation isn't stored on `Vector` at all — it's
a property of whichever `VectorBuffer` subclass `buffer` currently points at
(`buffer_ref->GetVectorType()`), and if there's no buffer yet, the vector
defaults to reporting `FLAT_VECTOR`. This is the mechanism §10.3 is about:
"what kind of vector is this" is a question you answer by asking the buffer,
and the buffer can be swapped out for a different subclass entirely — turning
a flat vector into a dictionary vector, say — without the `Vector` object
itself moving in memory or changing identity. `DataChunk::SetCardinality`
(deprecated in favor of `SetChildCardinality`, `data_chunk.hpp:73-83`) and
`DataChunk::ToUnifiedFormat` (`data_chunk.hpp:147`, wrapping the per-`Vector`
version covered in §10.6) round out the chunk-level API surface that
operators actually call.

Picture the whole thing for Query A's post-join, pre-aggregate chunk — three
columns (`custkey`, `orderkey` in the join key, `extendedprice`), 2048 rows
each, all sharing one `count`:

```
DataChunk  (count = 2048)
┌─────────────────────────────────────────────────────────────────┐
│ data[0]: Vector  type=INTEGER   (o.custkey)                     │
│ data[1]: Vector  type=INTEGER   (l.orderkey)                    │
│ data[2]: Vector  type=DECIMAL   (l.extendedprice)                │
└─────────────────────────────────────────────────────────────────┘
      │                  │                    │
      ▼                  ▼                    ▼
 ┌───────────┐     ┌───────────┐        ┌───────────┐
 │ buffer     │     │ buffer     │        │ buffer     │   ← each Vector's
 │ (VectorBuffer   │ (VectorBuffer   │ (VectorBuffer   │     own physical
 │  subclass,│     │  subclass,│        │  subclass,│     layout — may
 │  tagged by│     │  tagged by│        │  tagged by│     differ per
 │  VectorType)     │  VectorType)     │  VectorType)      column! (§10.3)
 └───────────┘     └───────────┘        └───────────┘
```

The row count (2048, or fewer for the last chunk of a scan) is a property of
the *chunk*; the physical representation (flat array? dictionary? constant?)
is a property of each individual *vector*, independently. This independence —
one column can be a flat array of real bytes while its neighbor in the same
chunk is a constant broadcast of one value — is what §10.3 is entirely about.

### 10.2.1 Columns that start with no buffer

`DataChunk::Initialize` normally gives each column its own `VectorCache`: a
reusable buffer of `STANDARD_VECTOR_SIZE` values that the column is reset to
before each new batch. One overload takes a `vector<bool>` and skips that for the
columns marked `false`:

```cpp
// src/common/types/data_chunk.cpp:53-74
void DataChunk::Initialize(Allocator &allocator, const vector<LogicalType> &types, const vector<bool> &initialize,
                           idx_t capacity) {
	D_ASSERT(types.size() == initialize.size());
	D_ASSERT(data.empty());

	for (idx_t i = 0; i < types.size(); i++) {
		// We copy the type here so we don't create another reference to the same LogicalTypeInfo
		// Otherwise, threads will constantly increment/decrement the atomic ref count to the same shared_ptr
		// This is necessary to avoid heavy contention on the atomic on many-core machines
		// Note that for nested types, there will still be contention on the atomic(s) one level down,
		// because this is a shallow copy (only copies LogicalTypeInfo to depth=1)
		auto copied_type = types[i].Copy();
		if (!initialize[i]) {
			data.emplace_back(copied_type, nullptr);
			vector_caches.emplace_back();
			continue;
		}

		VectorCache cache(allocator, copied_type, capacity);
		data.emplace_back(cache);
		vector_caches.push_back(std::move(cache));
	}
}
```

A column marked `false` is a `Vector` with a type but no buffer. Recall from
above that such a vector reports `FLAT_VECTOR`. It is meant to be filled by
*referencing* or *slicing* another vector, never by writing into it. The
engine uses this a lot: the operators that only reference or slice their input
(`PhysicalFilter`, a pass-through `PhysicalProjection`, the probe-side columns
of `PhysicalHashJoin`, and others) ask for output chunks with such columns
(Chapter 9 §9.1.4, `InitializeOutputChunk`). Three small rules make these
chunks safe to reuse:

- When the chunk is reset for the next batch, a column with no cache drops
  its buffer again (`VectorCache::ResetFromCache`,
  `src/common/types/vector_cache.cpp:130-137`). So it does not keep pointing at
  the previous batch's data.
- `DataChunk::size()` on an initialized chunk whose columns are all unset
  returns `0` instead of throwing (`DataChunk::DeriveSize`,
  `data_chunk.cpp:77-89`).
- `DataChunk::GetCapacity()` (`data_chunk.cpp:91-100`) returns the
  capacity of the first column that has a cache, or `STANDARD_VECTOR_SIZE` if
  no column has one.

The comment in the excerpt also mentions `LogicalTypeInfo`. A `LogicalType` is
not an id plus a `shared_ptr` to extra type information. It is a single reference to an intrusively reference-counted
`LogicalTypeInfo` object (`src/include/duckdb/common/types.hpp:283-285`), which
holds the type id, the physical type and the reference count. Types without
parameters, such as `INTEGER`, share one "immortal" info object that is never
counted or freed. This is why the `Vector`'s `type`
field stays cheap to copy, and why `Initialize` still copies each type once per
chunk: copying a parameterized type such as a `STRUCT` creates a fresh info
object (enum types are the exception and stay shared, `src/common/types.cpp:1482-1489`),
so threads do not all update the same reference count.

## 10.3 `VectorType`: six physical representations of one logical batch

`Vector::GetType()` tells you the *logical* SQL type (does this column hold
`INTEGER`s?). `Vector::GetVectorType()` tells you something orthogonal: the
*physical encoding* currently used to store those 2048 logical values. The
enum, verbatim:

```cpp
// src/include/duckdb/common/enums/vector_type.hpp:15-22
enum class VectorType : uint8_t {
	FLAT_VECTOR,       // Flat vectors represent a standard uncompressed vector
	FSST_VECTOR,       // Contains string data compressed with FSST
	CONSTANT_VECTOR,   // Constant vector represents a single constant
	DICTIONARY_VECTOR, // Dictionary vector represents a selection vector on top of another vector
	SEQUENCE_VECTOR,   // Sequence vector represents a sequence with a start point and an increment
	SHREDDED_VECTOR    // Shredded variant vector
};
```

Four of these (`FLAT`, `CONSTANT`, `DICTIONARY`, `SEQUENCE`) are the classic
set found in essentially every vectorized engine descended from this lineage
(X100/VectorWise [27], and MonetDB before it). The other two — `FSST_VECTOR` and
`SHREDDED_VECTOR` — are DuckDB-specific additions. `SHREDDED_VECTOR` is young
enough that its history is worth stating up front. `git log` on
`src/common/vector/shredded_vector.cpp` shows its introducing commit, "Initial
implementation of emitting shredded vectors" (`51692629c3`, Feb 10 2026). It is
machinery for the `VARIANT` type. It is not in any v1.5.x release (`v1.5.6`,
tagged Sep 27 2026, does not contain it). DuckDB **v2.0.0** (tagged Oct 20 2026)
is the first release that contains it, with the same six-value `VectorType`
enum shown above. The code that reads a serialized shredded vector checks its
layout: the first child must be the canonical unshredded `VARIANT` struct, and
the second (typed) child may not itself contain a `VARIANT`
(`src/common/vector/shredded_vector.cpp:71-78`).

Each variant is a different answer to the same question: "given up to 2048
logical values, what physical bytes do I actually store, and what do I do to
read logical value `i` back out?" To make the four classic ones concrete,
here's the same 8-row logical vector (shrunk from 2048 for readability) —
`[10, 10, 10, 20, 10, 30, 10, 40]` — shown in each of the four representations
that could plausibly produce exactly that sequence of logical values:

```
Logical vector (8 rows):    [10, 10, 10, 20, 10, 30, 10, 40]

FLAT_VECTOR — one physical value per logical row, plain array:
  physical data:  [10, 10, 10, 20, 10, 30, 10, 40]
  logical[i] = physical data[i]                       (no indirection)

CONSTANT_VECTOR — used only when every value truly is 10, e.g. a
projected literal `SELECT 42, a FROM t`:
  physical data:  [10]                                (one slot)
  logical[i] = physical data[0]   for every i          (broadcast)

DICTIONARY_VECTOR — a small SelectionVector indexing into a child vector,
e.g. after `WHERE a = 10` kept 5 of 8 original rows without copying them:
  child (original) data: [10, 99, 10, 10, 20, 10, 30, 40, 10]  ← untouched
  selection vector:      [0, 2, 3, 5, 8]
  logical[i] = child data[selection vector[i]]
    → logical[0]=child[0]=10, logical[1]=child[2]=10,
      logical[2]=child[3]=10, logical[3]=child[5]=10,
      logical[4]=child[8]=10                          (5 rows, all 10 —
                                                        a differently-shaped
                                                        example than the
                                                        8-row one above, to
                                                        show the indirection)

SEQUENCE_VECTOR — start + increment, e.g. a synthetic row-number column
`0, 1, 2, ..., 7`:
  start = 0, increment = 1
  logical[i] = start + i * increment                  (no array at all!)
```

**`FLAT_VECTOR`** is the default and the fallback everything else eventually
reduces to: one physical value stored per logical row, in a plain contiguous
array. It's what `Vector::GetVectorType()` reports when a vector has no
buffer at all (`vector.hpp:184-186`), and it's the representation every table
scan ultimately produces at the leaves once storage decompression has run
(Chapter 16 covers on-disk compression; by the time a `PhysicalTableScan`
hands a chunk upward, string/numeric compression schemes have already been
decoded into flat arrays, except where `FSST_VECTOR` specifically declines to
do that — see below).

**`CONSTANT_VECTOR`** represents a single physical value standing in for every
logical row. The textbook trigger is a literal in a projection —
`SELECT o.custkey, 42 AS answer FROM orders o` — where the `42` column is
computed exactly once and doesn't need 2048 (or 6 million) copies of the same
4 bytes sitting in memory; one value plus an implied row count is enough. The
same representation also shows up as the output of an aggregate broadcast
back over its group (a `SUM(...) OVER (PARTITION BY ...)` window result before
the window frame varies it) and, relevantly for Query A, as the natural
representation of a scalar produced once per chunk and needed by every row in
that chunk, such as a `NULL` check against a single bound parameter.

A constant vector keeps two sizes apart: its *logical* size (how many rows it
stands for, for example 2048) and its *physical* capacity (always one entry).
DuckDB keeps this split strict. Making a constant slice of a flat vector creates
a buffer with exactly one physical
entry and sets the logical size separately:

```cpp
// src/common/vector/flat_vector.cpp:107-115
buffer_ptr<VectorBuffer> StandardVectorBuffer::ConstantSliceInternal(const LogicalType &type, count_t count) {
	// a constant has one physical entry - the logical count is tracked separately from the physical capacity
	auto result = make_buffer<StandardVectorBuffer>(data_ptr, count_t(1ULL), type_size);
	result->GetValidityMask().Set(0, validity.RowIsValid(0));
	result->SetVectorType(VectorType::CONSTANT_VECTOR);
	result->SetVectorSize(count);
	result->AddAuxiliaryData(make_uniq<VectorBufferHolder>(shared_from_this()));
	return result;
}
```

In the same spirit, `Vector::Reserve` does nothing for a constant vector
(`src/common/types/vector.cpp:315-319`), and `ConstantVector::SetNull` reuses
the vector's buffer only when no other vector shares it; otherwise it starts a
fresh buffer, so turning one vector into a `NULL` constant cannot change the
data another vector is reading (`src/common/vector/constant_vector.cpp`).

**`DICTIONARY_VECTOR`** is the one to understand most precisely, because it's
the representation that makes filters and joins cheap, and it's built
directly on top of `SelectionVector` — the two are joined at the hip rather
than being independent concepts. `Vector::Dictionary()`
(`vector.hpp:87-96`) is the method that turns a vector into one; the
underlying buffer is:

```cpp
// src/include/duckdb/common/vector/dictionary_vector.hpp:34-39, 101-104
//! The DictionaryBuffer holds a selection vector and a reference to a DictionaryEntry
class DictionaryBuffer : public VectorBuffer {
public:
	explicit DictionaryBuffer(const SelectionVector &sel, idx_t sel_count, buffer_ptr<DictionaryEntry> entry_p);
	...
private:
	SelectionVector sel_vector;
	buffer_ptr<DictionaryEntry> entry;
};
```

So a `DICTIONARY_VECTOR` is precisely a pair: a `SelectionVector` (the index
array) plus a `DictionaryEntry` (`dictionary_vector.hpp:15-32`, whose only
essential field is `Vector data` — the *child* vector actually holding the
physical bytes). Reading logical value `i` out of a dictionary vector means
`child.data[sel_vector[i]]` — the "dictionary" in the name is not a
deduplication structure the way it might be in a Parquet or ORC string
dictionary; it's DuckDB's term for "a vector that references another vector's
data indirectly, through an index array." The concrete scenario where these
arise constantly is exactly the one the `DataChunk` class comment already
named: a filter operator that doesn't copy surviving rows into a fresh array,
but instead builds a `SelectionVector` of which row indices survived and lets
every column of the chunk reference the original data through it. §10.4 and
§10.8 work through this in full using Query A's own `WHERE` clause.

A `DictionaryEntry` can also cache the hashes of its child vector
(`cached_hashes`), so that a hash join that sees the same dictionary in many
chunks hashes each distinct value only once. This cache is filled exactly once
through a `OnceFlag` (`mutable OnceFlag cached_hashes_once;`,
`dictionary_vector.hpp:30`), so readers do not have to take a mutex.

**`SEQUENCE_VECTOR`** stores no array at all — just a `start` and an
`increment`:

```cpp
// src/include/duckdb/common/vector/sequence_vector.hpp:15,19-20
class SequenceBuffer : public VectorBuffer {
	...
	int64_t start;
	int64_t increment;
```

and `Vector::Sequence()` (`src/common/types/vector.cpp:505-507`) is a
three-line function that just replaces the buffer with one of these. Reading
logical value `i` is `start + i * increment` — computed on the fly, nothing
to look up. The canonical use is a synthetic row-number-like column: a
`rowid` produced by a scan, or the counter DuckDB's `generate_series` /
positional operators hand out — anywhere the values are already known to
follow an arithmetic progression and materializing them would be pure waste.
Because it carries no per-row data pointer, it's also the one representation
that operators *cannot* read generically (there's nothing to point a data
pointer at) — which is exactly why, as §10.6 shows, it gets flattened before
any generic operator touches it.

**`FSST_VECTOR`** holds string data compressed with FSST [28] (a
symbol-table-based string compression scheme; the compression mechanics
themselves are Chapter 17's territory). The reason it earns its own
`VectorType` rather than always being decompressed back to `FLAT_VECTOR`
before use is a genuine performance trick: for a large class of string
operations — equality comparison, prefix comparison, `LIKE 'foo%'`, even
certain hash computations — you can operate **directly on the compressed
bytes**, because FSST is a byte-order-preserving symbol substitution: two
FSST-compressed strings compare equal (or in the same prefix order) as
compressed byte sequences if and only if the original decompressed strings
would. Comparing compressed bytes directly means a filter like
`WHERE l.comment LIKE 'foo%'` can be evaluated **without ever fully
decompressing the column**, which avoids a decompress-then-compare double
pass over every string in the vector — you'd otherwise pay the decompression
cost for strings that fail the predicate and get thrown away one line later.
`VectorFSSTStringBuffer` (`src/include/duckdb/common/vector/fsst_vector.hpp:15-46`)
carries both the compressed string data and the decoder needed for the
(comparatively rare) cases where full decompression genuinely is required.

**`SHREDDED_VECTOR`** is the newcomer. It first shipped in v2.0.0 (see above),
so it has had the least time in real use of the six. It exists to support
`VARIANT`, DuckDB's semi-structured type. A `VARIANT` value's *canonical*
on-wire representation, spelled out directly in a source comment, is a fully
generic self-describing layout:

```cpp
// src/include/duckdb/common/types/variant_iterator.hpp:37-45
// A VARIANT vector is either stored in its canonical "unshredded" layout:
//     STRUCT(
//         keys VARCHAR[],
//         children STRUCT(key_id UINTEGER, value_id UINTEGER)[],
//         values STRUCT(type_id UTINYINT, byte_offset UINTEGER)[],
//         data BLOB
//     )
// or as a SHREDDED_VECTOR which combines a typed ("shredded") representation with the leftover
// "unshredded" component for everything that did not fit the shredded schema.
```

That canonical layout is maximally flexible (any value of any shape can be
encoded as a blob plus an index describing its structure) but maximally slow
to query — every access has to walk the generic `keys`/`children`/`values`
indirection rather than reading a typed column directly. "Shredding" is the
optimization: DuckDB inspects a batch of `VARIANT` values, infers a common
structural shape from a sample, and — where a value actually matches that
inferred shape — re-encodes it into ordinary, typed, columnar sub-vectors
(the "shredded" part), falling back to the original generic encoding (the
"unshredded" part) only for the values that don't conform. The actual
debug/test entry point makes the two-part structure concrete:

```cpp
// src/storage/table/variant/variant_shredding.cpp:840-841, 852-856
	//! Derive the shredding schema from the *first* value only - subsequent values that don't match it
	//! will be partially shredded (i.e. fall back to the unshredded/overlay component)
	...
	//! Shred into a STRUCT(unshredded, shredded) and wrap it as a SHREDDED_VECTOR
	Vector shredded_struct(shredded_struct_type, count);
	ShredVariantData(variant, shredded_struct, count);
	FlatVector::SetSize(shredded_struct, count_t(count));
	variant.Shred(shredded_struct, count);
```

So a `SHREDDED_VECTOR` is, physically, a `STRUCT(unshredded, shredded)` (two
child vectors — one holding the fallback generic encoding, one holding the
typed columns matching the inferred schema) wrapped by a
`ShreddedVectorBuffer` (`src/include/duckdb/common/vector/shredded_vector.hpp:15-46`)
that tags the whole thing with `VectorType::SHREDDED_VECTOR` so downstream
code knows to unpack it that way. The upside this buys, when it applies, is
exactly what shredding always buys for semi-structured data (this is the same
idea Parquet/BigQuery-style "shredding" of nested/repeated columns is named
after): operators that only care about one consistently-typed field of a
`VARIANT` column (e.g. `variant_col.age::INTEGER` where most rows really do
carry an integer `age`) can read a typed, flat sub-vector directly instead of
paying the generic-layout indirection cost for every row, every time. Beyond
that — the exact conditions under which shredding kicks in for a real query
plan, how deeply nested shapes are handled, and how stable this machinery is
expected to be — the source doesn't give enough to say more with confidence,
and this book won't guess.

One rule about *when shredding is refused* is easy to state. `VARIANT` object keys are case-sensitive: `'a'` and
`'A'` are two different keys. But the shredded part is a `STRUCT`, and `STRUCT`
field names are case-insensitive. So if one object column has two keys that
differ only in case, they cannot both become struct fields. The shredding-type
builder checks for this with a case-insensitive set of field names, and
gives up on shredding that object when it finds a clash:

```cpp
// src/storage/table/variant/variant_shredding.cpp:449-452
			if (!field_names.emplace(entry.first).second) {
				//! STRUCT field names are case-insensitive, but VARIANT object keys are case-sensitive.
				return false;
			}
```

`test/issues/general/test_24298.test` stores `{'a': 1}` and `{'A': 2}` in one
`VARIANT` column and runs `FORCE CHECKPOINT`. The column stays unshredded
(`vector_type(v)` is `FLAT_VECTOR`), and both keys keep their own values. If the
clash is inside a nested object (`{'outer': {'a': 1}, 'other': 10}` and
`{'outer': {'A': 2}, 'other': 20}`), the column is still `SHREDDED`: only
`outer` falls back to the unshredded part, and `other` is shredded normally.
The test is named after GitHub issue 24298, which reported this key clash.

## 10.4 `SelectionVector`: indirection instead of copying

`SelectionVector` is, physically, nothing more than an array of indices:

```cpp
// src/include/duckdb/common/types/selection_vector.hpp:31-32, 136-142, 176-180
struct SelectionVector {
	...
	inline idx_t get_index(idx_t idx) const {
		return sel_vector ? get_index_unsafe(idx) : idx;
	}
	inline idx_t get_index_unsafe(idx_t idx) const {
		D_ASSERT(idx < capacity);
		return sel_vector[idx];
	}
	...
private:
	sel_t *sel_vector;
	buffer_ptr<SelectionData> selection_data;
	idx_t capacity;
};
```

Note `get_index`'s fallback (`selection_vector.hpp:136-138`): if `sel_vector`
is null, `get_index(idx)` just returns `idx` back unchanged — an "empty"
selection vector is the identity permutation, at zero storage cost. This is
the same "absence means cheap default" pattern §10.5's validity mask uses,
and it means a freshly-scanned, unfiltered chunk pays nothing for this
mechanism until something actually needs it.

The connection to `DICTIONARY_VECTOR` (§10.3) is direct and worth restating
precisely, because the two concepts are easy to conflate: a `SelectionVector`
is the general-purpose index array; `DICTIONARY_VECTOR` is *one specific use*
of a `SelectionVector` — the case where a `Vector`'s buffer is a
`DictionaryBuffer` holding both a `SelectionVector` and a reference to the
child vector the indices point into (`dictionary_vector.hpp:101-104`, quoted
in full in §10.3). But `SelectionVector`s are used more broadly than that:
`DataChunk::Slice(sel, count)` (`data_chunk.hpp:149`) applies one across
*every* column of a chunk at once, and `Vector::Slice(sel, count)`
(`vector.hpp:88`) can turn any vector — flat, constant, whatever it currently
is — into a dictionary vector over itself by installing exactly this kind of
buffer.

Here is where that actually happens, in the operator that produces it most
directly for Query A — `PhysicalFilter`, evaluating
`l.shipdate < DATE '1998-01-01'`:

```cpp
// src/execution/operator/filter/physical_filter.cpp:13-20, 45-56
class FilterState : public CachingOperatorState {
public:
	explicit FilterState(ExecutionContext &context, Expression &expr)
	    : executor(context.client, expr), sel(STANDARD_VECTOR_SIZE) {
	}

	ExpressionExecutor executor;
	SelectionVector sel;
	...
};

OperatorResultType PhysicalFilter::ExecuteInternal(ExecutionContext &context, DataChunk &input, DataChunk &chunk,
                                                   GlobalOperatorState &gstate, OperatorState &state_p) const {
	auto &state = state_p.Cast<FilterState>();
	idx_t result_count = state.executor.SelectExpression(input, state.sel);
	if (result_count == input.size()) {
		// nothing was filtered: skip adding any selection vectors
		chunk.Reference(input);
	} else if (result_count > 0) {
		chunk.Slice(input, state.sel, result_count);
	}
	return OperatorResultType::NEED_MORE_INPUT;
}
```

`ExpressionExecutor::SelectExpression` writes the surviving row *indices*
(not the surviving row *values*) into `state.sel`, and returns how many rows
survived. `DataChunk::Slice(input, state.sel, result_count)` then does this,
per column:

```cpp
// src/common/types/data_chunk.cpp:387-400
void DataChunk::Slice(const DataChunk &other, const SelectionVector &sel, idx_t count_p, idx_t col_offset) {
	D_ASSERT(other.ColumnCount() <= col_offset + ColumnCount());
	SelCache merge_cache;
	for (idx_t c = 0; c < other.ColumnCount(); c++) {
		if (other.data[c].GetVectorType() == VectorType::DICTIONARY_VECTOR) {
			// already a dictionary! merge the dictionaries
			data[col_offset + c].Reference(other.data[c]);
			data[col_offset + c].Slice(sel, count_p, merge_cache);
		} else {
			data[col_offset + c].Slice(other.data[c], sel, count_p);
		}
	}
	count = count_p;
}
```

Every output column becomes a reference to the corresponding input column,
sliced by `sel` — which, per the header comment quoted in §10.2
(`data_chunk.hpp:152`), means "turning all Vectors into Dictionary Vectors,
using `sel`." **Not one byte of `l.orderkey` or `l.extendedprice` gets
copied.** The 2048-row `lineitem` chunk that came out of the scan is left
completely untouched in memory; the filter's entire output is a small index
array (`result_count` indices, each `idx_t`) plus a handful of `Vector`
objects that now point at that array instead of holding flat data of their
own. And the branch just above it — `if (other.data[c].GetVectorType() ==
VectorType::DICTIONARY_VECTOR)` — shows this composes: if the input was
*already* a dictionary vector (say, because an earlier operator in the same
pipeline had already filtered), the new selection vector is merged with the
existing one (`Vector::Slice(sel, count, cache)` on an already-dictionary
vector, `vector.hpp:90`) rather than stacking indirection two layers deep.

The performance case for this is the same one Chapter 2 already built for
memory access in general: without a selection vector, every filter, every
join, every semi-join anti-match would need to allocate a fresh output buffer
and physically copy every surviving value into it — an O(n) write, per
operator, per chunk, on data that (per §10.1) was specifically sized to sit in
L2 so that operators *wouldn't* need to pay for extra memory traffic between
each other. A selection vector turns that O(n) copy into an O(1) reference
plus a much smaller O(result_count) index array, and — because the technique
composes across operator boundaries — a whole *chain* of filtering/joining
operators can each logically narrow the working set down further and further
while all of them still point at the same original physical bytes the scan
produced. The underlying `lineitem` data is only ever materialized (copied
into a fresh flat buffer) when something downstream genuinely needs a flat
layout it can write into directly — which is precisely what `Vector::Flatten`
(`vector.hpp:111-117`) is for, and precisely what §10.6's unified-format
conversion sometimes has to fall back to.

## 10.5 The validity mask: a NULL bitmap, and a fast path for "no NULLs at all"

Every `Vector` (well — every one whose buffer type supports it; see below)
carries a `ValidityMask` recording, per row, whether the value is `NULL`. The
representation is a bit per row, not a byte per row:

```cpp
// src/include/duckdb/common/types/validity_mask.hpp:58-68 (elided)
template <typename V>
struct TemplatedValidityMask {
	using ValidityBuffer = TemplatedValidityData<V>;
public:
	static constexpr const idx_t BITS_PER_VALUE = ValidityBuffer::BITS_PER_VALUE;   // 64, for validity_t = uint64_t
	static constexpr const idx_t STANDARD_ENTRY_COUNT = (STANDARD_VECTOR_SIZE + (BITS_PER_VALUE - 1)) / BITS_PER_VALUE;
	static constexpr const idx_t STANDARD_MASK_SIZE = STANDARD_ENTRY_COUNT * sizeof(V);
	...
protected:
	V *validity_mask;
	buffer_ptr<ValidityBuffer> validity_data;
	idx_t capacity;
};
```

With `V = validity_t = uint64_t` (`validity_mask.hpp:50`), a full
`STANDARD_VECTOR_SIZE` (2048) mask is `2048 / 64 = 32` entries of 8 bytes
each — **256 bytes** to carry NULL-ness for the entire chunk's worth of one
column, versus 2048 bytes for a byte-per-row `bool[]`. This is a straight
memory-density win (an 8× reduction just from packing), which matters
directly for §10.1's whole argument: every byte a `ValidityMask` doesn't need
is a byte more of the column's real data that fits in L2 alongside it. It
also plays to a strength general-purpose CPUs already have: testing 64 rows'
validity at once is a single 64-bit load plus (for the "are they all valid"
question) a single comparison against `MAX_ENTRY`
(`validity_mask.hpp:162-165`), rather than 64 separate byte loads.

The more interesting design decision, though, is what happens when a vector
has **no NULLs at all** — which, for a lot of real columns (primary keys, `NOT
NULL` columns, columns after a completed `WHERE col IS NOT NULL` filter), is
the common case, not the exception. Rather than allocating a mask and setting
every bit to 1, DuckDB skips allocating the mask at all:

```cpp
// src/include/duckdb/common/types/validity_mask.hpp:82-87, 197-209
inline bool CannotHaveNull() const {
	return !validity_mask;
}
inline bool CanHaveNull() const {
	return validity_mask;
}
...
//! Returns true if a row is valid (i.e. not null), false otherwise
inline bool RowIsValid(idx_t row_idx) const {
	...
	if (!validity_mask) {
		return true;
	}
	return RowIsValidUnsafe(row_idx);
}
```

A null `validity_mask` pointer *means* "this vector cannot have NULLs" —
`RowIsValid` for any row returns `true` unconditionally, without ever
touching memory beyond the pointer check itself. `SetValid` likewise no-ops
when `validity_mask` is null (`validity_mask.hpp:227-231`: "if
`CannotHaveNull()` we don't need to do anything — the row is already valid");
only `SetInvalid` (marking something *as* NULL for the first time) forces the
lazy allocation (`validity_mask.hpp:256-258`, calling `Initialize(capacity)`).
So the common all-valid case pays literally zero bytes and zero allocations
for its NULL tracking, and the cost only appears the moment a vector actually
acquires its first `NULL` — a textbook "pay only for what you use" design,
and one more instance of the same pattern the selection vector's
`get_index` fallback used in §10.4: absence of the auxiliary structure is
itself a cheap, correct default.

## 10.6 The unified representation: one generic loop instead of 6^N specialized ones

Here is the combinatorics problem §10.3–10.5 have been building toward.
Consider a generic binary operator — equality comparison, say, for a hash
join's build-side key column against its probe-side key column. Each of the
two input vectors can independently be any of the six `VectorType`s from
§10.3. If the comparison kernel had to be written as a direct, hand-specialized
loop for every *pairing* of input formats, an operator with two inputs would
need on the order of 6 × 6 = 36 specialized loop bodies just to cover every
combination — and an *N*-ary expression (a `CASE` with several branches, a
multi-column comparison) would need up to 6^N. That is obviously not
something anyone writes or maintains by hand, and it isn't what DuckDB does.

Instead, every operator that needs to read a vector generically first calls
`Vector::ToUnifiedFormat()`, converting *any* input format into one shared
shape — a data pointer plus a selection vector plus a validity mask — and
then runs exactly **one** generic loop over that shape, regardless of what the
original vector's `VectorType` was:

```cpp
// src/common/types/vector.cpp:465-474
void Vector::ToUnifiedFormat(UnifiedVectorFormat &format) const {
	format.physical_type = GetType().InternalType();
	auto vtype = GetVectorType();
	if (vtype != VectorType::FLAT_VECTOR && vtype != VectorType::CONSTANT_VECTOR &&
	    vtype != VectorType::DICTIONARY_VECTOR) {
		// FSST/SEQUENCE/SHREDDED: flatten first so the buffer can provide unified format
		Flatten();
	}
	Buffer().ToUnifiedFormat(format);
}
```

and the shape it produces:

```cpp
// src/include/duckdb/common/vector/unified_vector_format.hpp:22-35
struct UnifiedVectorFormat {
	...
	const SelectionVector *sel;
	const_data_ptr_t data;
	ValidityMask validity;
	SelectionVector owned_sel;
	PhysicalType physical_type;
	...
};
```

The method's own doc comment (`vector.hpp:121-126`) states the design intent
almost as plainly as this chapter has been trying to: "the most common vector
types (flat, constant & dictionary) can be converted to the canonical format
'for free'" — meaning no data copy, just a `sel`/`data` pointer pair
constructed from whatever the buffer already has (a flat vector's own data
pointer plus an identity selection; a constant vector's single value pointer
plus a selection that maps every logical index to slot 0; a dictionary
vector's *actual* `SelectionVector`, reused as-is). Everything **else** —
`SEQUENCE_VECTOR` (no data pointer exists to hand out — it's computed
on-the-fly per §10.3), `FSST_VECTOR` (compressed bytes an arbitrary generic
loop can't interpret), and `SHREDDED_VECTOR` (a nested struct, not a single
flat array) — gets materialized into a real flat array first, via
`Flatten()`, before the buffer can hand back a unified view at all. That's
the concrete meaning of "sequence vectors are first flattened" from this
chapter's brief: it's this exact `if` on line 468 of `vector.cpp`, and it
applies identically to the other two non-"free" formats.

The origin-story comment is worth keeping, because it explains why this
mechanism has an unusual name: "`ToUnifiedFormat` was originally called
`Orrify`, as a tribute to Orri Erling who came up with it"
(`vector.hpp:126`) — a nod to the VectorWise/X100 lineage this whole design
descends from. `RecursiveToUnifiedFormat` (`vector.hpp:129-133`,
`vector.cpp:476-503`) extends the same idea down into nested types, walking
`LIST`/`ARRAY`/`STRUCT` children and recursing:

```cpp
// src/common/types/vector.cpp:480-503
void Vector::RecursiveToUnifiedFormat(const Vector &input, RecursiveUnifiedVectorFormat &data) {
	input.ToUnifiedFormat(data.unified);
	data.logical_type = input.GetType();

	if (input.GetType().InternalType() == PhysicalType::LIST) {
		auto &child = ListVector::GetChild(input);
		data.children.emplace_back();
		RecursiveToUnifiedFormat(child, data.children.back());

	} else if (input.GetType().InternalType() == PhysicalType::ARRAY) {
		...
	} else if (input.GetType().InternalType() == PhysicalType::STRUCT) {
		auto &children = StructVector::GetEntries(input);
		for (idx_t i = 0; i < children.size(); i++) {
			data.children.emplace_back();
		}
		for (idx_t i = 0; i < children.size(); i++) {
			RecursiveToUnifiedFormat(children[i], data.children[i]);
		}
	}
}
```

so a generic recursive equality/hash kernel over an arbitrarily nested type
gets exactly one uniform shape at every level of the nesting too, rather than
combinatorial explosion multiplying across both "vector format" *and*
"nesting depth."

**Query A's own hash join is where this earns its keep**, and it's worth
seeing the real call site because it isn't just theory — the exact scenario
§10.4 built (a filtered, dictionary-encoded `lineitem` chunk) is what arrives
here. `JoinHashTable::InitializeScanStructure`, run on the probe side's key
columns before every lookup:

```cpp
// src/execution/join_hashtable.cpp:1138,1152-1153
void JoinHashTable::InitializeScanStructure(ScanStructure &scan_structure, DataChunk &keys,
                                            TupleDataChunkState &key_state,
                                            optional_ptr<const SelectionVector> &current_sel) {
	...
	// first prepare the keys for probing
	TupleDataCollection::ToUnifiedFormat(key_state, keys);
	scan_structure.count = PrepareKeys(keys, key_state.vector_data, current_sel, scan_structure.sel_vector, false);
```

`keys` here is the probe-side `DataChunk` — for Query A, the join column
`l.orderkey`, quite possibly still a `DICTIONARY_VECTOR` if the shipdate
filter ran in the same pipeline just above the join (exactly §10.8's trace).
`ToUnifiedFormat` on that dictionary vector is one of the "for free" cases:
no copy, just handing the join's key-preparation code the existing
`data`/`sel` pair the filter already built. The probe path even special-cases
this further, before falling all the way through to the fully generic path,
by checking the *incoming* vector's format directly:

```cpp
// src/execution/join_hashtable.cpp:1162-1168, 1175-1187
//! Per-chunk fast-reject so non-dictionary chunks skip the TryProbeDictionary call entirely
static bool LHSChunkIsDictionaryEligible(const Vector &lhs_key) {
	if (lhs_key.GetVectorType() != VectorType::DICTIONARY_VECTOR) {
		return false;
	}
	return DictionaryVector::DictionarySize(lhs_key).IsValid();
}
...
void JoinHashTable::Probe(ScanStructure &scan_structure, DataChunk &keys, TupleDataChunkState &key_state,
                          ProbeState &probe_state, optional_ptr<Vector> precomputed_hashes) {
	// dispatch into the compressed-vector fast paths when the operator gated them on. precomputed_hashes
	// is incompatible: the caller hashed the full N-row chunk; the fast paths re-hash a sliced D-row chunk.
	if (probe_state.dict_state && !precomputed_hashes) {
		auto &lhs_key = keys.data[0];
		if (LHSChunkIsConstant(lhs_key) && TryProbeConstant(scan_structure, keys, key_state, probe_state)) {
			return;
		}
		if (LHSChunkIsDictionaryEligible(lhs_key) && TryProbeDictionary(scan_structure, keys, key_state, probe_state)) {
			return;
		}
	}
	...
```

This is the same idea one level up: even the generic unified-format path is
itself something DuckDB tries to avoid paying for when a cheaper,
format-specific shortcut exists (probing once per *distinct dictionary
value* instead of once per row, when the dictionary is small — Chapter 8's
join-implementation chapter is the right place for the full mechanics of
`TryProbeDictionary`/`TryProbeConstant`). The unified format isn't the fast
path; it's the **correct, always-available fallback** that every operator can
rely on when no special case applies. That reliance is tested directly:
`PhysicalPlanGenerator` can wrap the whole physical tree in a debug
`PhysicalVerifyVector` operator (`src/execution/physical_plan_generator.cpp:69`)
that deliberately re-represents every intermediate vector as
dictionary/constant/etc. between operators, specifically to shake out bugs in
code that forgot to go through `ToUnifiedFormat` and instead assumed one
particular physical layout.

## 10.7 Nested types: struct-of-vectors, and lists as shape plus one flattened child

`STRUCT` and `LIST` don't get bespoke physical encodings the way scalar types
do; they get **columnar decomposition applied recursively**, using the same
`Vector`/`DataChunk` machinery this whole chapter has already covered.

A `STRUCT` is stored as a **struct of vectors**: each field of the struct gets
its own independent child `Vector`, with its own independent validity mask
(a struct value can be non-NULL overall while one of its fields is NULL, or
vice versa — the two validity masks are tracked separately). The buffer:

```cpp
// src/include/duckdb/common/vector/struct_vector.hpp:16-21, 66-77
class VectorStructBuffer : public VectorBuffer {
public:
	explicit VectorStructBuffer(const LogicalType &struct_type, capacity_t capacity = capacity_t(STANDARD_VECTOR_SIZE));
	VectorStructBuffer(vector<Vector> children, capacity_t capacity);
	...
private:
	ValidityMask validity;
	//! child vectors used for nested data
	vector<Vector> children;
	//! The capacity of the struct
	idx_t capacity;
};

struct StructVector {
	DUCKDB_API static const vector<Vector> &GetEntries(const Vector &vector);
	DUCKDB_API static vector<Vector> &GetEntries(Vector &vector);
};
```

`children` is `vector<Vector>` — literally a list of ordinary `Vector`
objects, each of which can independently be `FLAT`, `DICTIONARY`, `CONSTANT`,
or anything else from §10.3, exactly as if it were a top-level column. A
`STRUCT(a INTEGER, b VARCHAR)` column is, physically, two completely ordinary
vectors (an `INTEGER` one and a `VARCHAR` one) sitting inside one
`VectorStructBuffer`, plus the struct's own top-level validity mask. Reading
field `a` of every row is: get `StructVector::GetEntries(vec)[0]`, and you're
holding an ordinary `Vector` you already know how to read.

`LIST` (and `ARRAY`) split responsibilities the other way: **shape is kept
separate from data**. Each row's list doesn't get its own little array —
instead, one flattened child vector holds every list element from the
*entire chunk*, concatenated end to end, and a separate array of
`{offset, length}` pairs (one per row) tells you which slice of that single
flattened child belongs to which row:

```cpp
// src/include/duckdb/common/types.hpp:52
struct list_entry_t { // NOLINT: mimic std casing
	...
	uint64_t offset;
	uint64_t length;
};
```

```cpp
// src/include/duckdb/common/vector/list_vector.hpp:17-18, 30-53, 73-74
class VectorListBuffer : public StandardVectorBuffer {
public:
	explicit VectorListBuffer(Allocator &allocator, capacity_t capacity, unique_ptr<Vector> vector);
	...
public:
	Vector &GetChild() {
		return *child;
	}
	...
	idx_t GetChildSize() const {
		return child->size();
	}
	...
private:
	//! child vectors used for nested data
	unique_ptr<Vector> child;
};

struct ListVector {
	...
	//! Gets a reference to the underlying child-vector of a list
	DUCKDB_API static const Vector &GetChild(const Vector &vector);
	DUCKDB_API static Vector &GetChildMutable(Vector &vector);
	//! Gets the total size of the underlying child-vector of a list
	DUCKDB_API static idx_t GetListSize(const Vector &vector);
	DUCKDB_API static void SetListSize(Vector &vec, idx_t size);
	...
};
```

The top-level `LIST` vector itself is, physically, a flat/constant/dictionary
array of `list_entry_t {offset, length}` structs — one per logical row — and
`VectorListBuffer::child` (`list_vector.hpp:74`) is the single shared
`Vector` holding every element from every row's list, back to back. Row `i`'s
elements are `child[entry[i].offset .. entry[i].offset + entry[i].length)`.
Picture a `LIST(INTEGER)` column with rows `[1,2]`, `[]`, `[3]`:

```
Top-level LIST vector (3 rows, list_entry_t array):
  row 0: {offset=0, length=2}
  row 1: {offset=2, length=0}
  row 2: {offset=2, length=1}

Single flattened child vector (shared across the whole chunk):
  [ 1, 2, 3 ]
    ^  ^  ^
    │  │  └── row 2's one element (offset 2, length 1)
    │  └───── row 0's second element
    └──────── row 0's first element (offset 0)
```

This is the same underlying idea structs use (decompose into independent,
ordinary vectors and reuse everything this chapter already built), applied to
a shape — variable-length, per-row — that a fixed-width array can't represent
directly without either padding to the longest row (wasteful) or storing
shape and payload together (which would defeat the point of columnar
storage). Both decompositions are why `RecursiveToUnifiedFormat` (§10.6) has
to exist and has to recurse: a generic operator over a `LIST(STRUCT(...))`
column needs the unified-format treatment applied at every one of these
levels — the top-level shape vector, the struct's field vectors, and (if a
field is itself a list) further down still — before a single generic
comparison/hash loop can run over the whole nested shape uniformly.

## 10.8 Query A traced: `lineitem`'s filter and join through this chapter's lens

Put the last four sections together over Query A's actual shape:

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

From Chapter 1's physical plan, the filter on `l.shipdate` was pushed into the
`lineitem` scan itself by the optimizer's `filter_pushdown` pass, and the join
is a `PhysicalHashJoin` with `lineitem` as the probe side. Walk one 2048-row
chunk of `lineitem` through:

```
1. PhysicalTableScan(lineitem), filter shipdate < DATE '1998-01-01' pushed in
   ─────────────────────────────────────────────────────────────────────────
   Storage hands back a chunk. orderkey/extendedprice arrive as FLAT_VECTOR
   (decompressed from on-disk storage, Ch. 16) — or, if the row group's
   zonemap for shipdate already proved the whole group qualifies/disqualifies,
   the filter may not even need row-level evaluation for that chunk. Assume
   the general case: some rows in this chunk pass, some don't.

2. Row-level shipdate filter evaluated
   ─────────────────────────────────────────────────────────────────────────
   ExpressionExecutor::SelectExpression writes surviving row *indices* into
   a SelectionVector (physical_filter.cpp:48, §10.4) — say 1,400 of 2048
   rows pass. DataChunk::Slice(input, sel, 1400) (data_chunk.cpp:387-400)
   turns every output column — orderkey, extendedprice — into a
   DICTIONARY_VECTOR: same underlying FLAT_VECTOR data as step 1, now
   referenced through a 1,400-entry index array. No copy of either column.

        orderkey:        DICTIONARY_VECTOR over the original flat data
        extendedprice:    DICTIONARY_VECTOR over the original flat data
                          (both share the *same* SelectionVector, since the
                           whole chunk was sliced together)

3. PhysicalHashJoin, probe side (lineitem) reaches JoinHashTable::Probe
   ─────────────────────────────────────────────────────────────────────────
   keys.data[0] is the orderkey DICTIONARY_VECTOR from step 2.
   LHSChunkIsDictionaryEligible(lhs_key) (join_hashtable.cpp:1163-1168)
   checks exactly this and, if the join's dict_state fast path is enabled,
   tries TryProbeDictionary — probing once per *distinct* dictionary value
   rather than once per of the 1,400 rows (Chapter 8's territory).
   Otherwise, InitializeScanStructure calls
   TupleDataCollection::ToUnifiedFormat(key_state, keys) (line 1152, §10.6):
   because DICTIONARY_VECTOR is one of the "free" formats, this reuses the
   existing selection vector + underlying data pointer rather than
   flattening — the generic hash-probe loop then runs identically whether
   orderkey arrived flat, dictionary, or (hypothetically) constant.

4. extendedprice reaches the SUM aggregate (post-join)
   ─────────────────────────────────────────────────────────────────────────
   Whatever survived the join's row-matching is itself expressed as another
   selection vector over the already-dictionary-encoded extendedprice column
   (§10.4's "already a dictionary! merge the dictionaries" branch,
   data_chunk.cpp:391-394) — chains of indirection collapse into one
   SelectionVector rather than stacking. PhysicalHashAggregate's SUM state
   update then reads extendedprice through the same ToUnifiedFormat
   machinery §10.6 described, entirely oblivious to how many operators up
   the chain actually touched the physical bytes underneath.
```

The throughline: from the moment the `shipdate` predicate first discards
rows, `orderkey` and `extendedprice` are never physically copied again on
their way into the hash table build/probe or the `SUM` accumulator — only
small `SelectionVector` index arrays get built, merged, and consulted. That
is the concrete payoff of everything in §10.3–10.6: cheap logical filtering
that composes across operator boundaries, and a uniform way (`ToUnifiedFormat`)
for every one of those downstream operators to read the result without
needing to special-case how many layers of filtering happened before it.

## 10.9 What this sets up

Everything from here forward assumes this chapter's vocabulary without
re-explaining it. Chapter 11 covers how chunks get grouped into morsels and
morsels get scheduled across threads — the piece deliberately deferred in
§10.1. Chapter 12's `ExpressionExecutor` is the thing actually calling
`ToUnifiedFormat` and iterating unified formats inside every scalar and
aggregate expression evaluation this chapter only sketched
(`SelectExpression` in §10.4, the `SUM` update in §10.8). Chapter 13's case
that DuckDB interprets rather than compiles per query rests on exactly the
mechanism in §10.6: the "different, hyper-specialized inner loop for every
combination of operator, type, and vector representation" Chapter 1 promised
is precisely the small, fixed set of unified-format-consuming loops this
chapter has now shown in their natural habitat — compiled once, when DuckDB
itself is built, selected at runtime by nothing more than a `VectorType`
tag and a `switch`.
