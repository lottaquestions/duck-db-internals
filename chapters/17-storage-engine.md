# Chapter 17 — Storage Engine

## 17.1 One file, divided into blocks

Every chapter so far has treated `orders` and `lineitem` as things that simply
*exist* — a `LogicalGet` names them, a `PhysicalTableScan` reads from them, and
nothing before this chapter has asked what "reading from them" actually touches
on disk. This chapter answers that question.

DuckDB's default persistence format is a **single file**. A `.duckdb` file
contains, in one place: the catalog (every schema, table, view, and function
definition), every table's row data, every index, and (alongside it, as a
separate file) the write-ahead log. This is a deliberate, load-bearing design
choice, not an accident of implementation convenience — it's the same choice
SQLite makes, and it's the opposite of what most server-oriented relational
databases do. Postgres, for comparison, stores each relation (each table, each
index) as its own file (or set of 1 GB segment files) inside a per-database
directory — `orders` and `lineitem` would each get their own path on disk, and
the catalog lives in yet other files. DuckDB deliberately gives up the
operational conveniences that per-relation files buy you (e.g. `cp`-ing a
single table out of the data directory) in exchange for a database that is
literally one file you can copy, email, or commit to a repository — appropriate
for an embedded, single-node analytical engine where "the database" and "a data
file" are meant to feel like the same kind of object.

It's worth being explicit that this single-file model is not the only storage
architecture DuckDB itself now supports. **DuckLake** — DuckDB Labs' own
lakehouse format, which separates data into open files (Parquet) on object
storage from a small SQL-backed catalog tracking those files — is DuckDB's own
answer to "what if you don't want everything in one file," and it is
increasingly relevant to how people deploy DuckDB in practice. Per the front
matter's scope note (§ "A note on scope and honesty about uncertainty"), this
book is scoped to the query-processing core and its supporting single-file
storage/transaction layer; DuckLake is out of scope here and gets no further
treatment in this book. Everything below describes the single-file engine that
backs a plain `ATTACH 'file.duckdb'` or the default in-process database.

### 17.1.1 The block manager

Inside that one file, storage is divided into fixed-size **blocks** — the same
unit of allocation SQLite and every buffer-pool-based RDBMS uses. The default
block size is set by a single constant:

```cpp
// src/include/duckdb/storage/storage_info.hpp:33
#define DEFAULT_BLOCK_ALLOC_SIZE 262144ULL
```

262144 bytes is 256 KiB. A few lines later, the same header derives the
*usable* block size by subtracting a small per-block header:

```cpp
// src/include/duckdb/storage/storage_info.hpp:64-68
constexpr static idx_t DEFAULT_BLOCK_HEADER_SIZE = sizeof(idx_t);
// ...
//! The default block size.
constexpr static idx_t DEFAULT_BLOCK_SIZE = DEFAULT_BLOCK_ALLOC_SIZE - DEFAULT_BLOCK_HEADER_SIZE;
```

This block size is configurable per-database (within bounds — `Storage::
MIN_BLOCK_ALLOC_SIZE` is 16 KiB, `Storage::MAX_BLOCK_ALLOC_SIZE` is 256 KiB, and
`VerifyBlockAllocSize` enforces both the range and a power-of-two requirement),
but 256 KiB is what an ordinary `duckdb file.duckdb` gets unless you override it
at `ATTACH` time.

The interface for reading, writing, and allocating blocks is the abstract
`BlockManager` base class:

```cpp
// src/include/duckdb/storage/block_manager.hpp:31
class BlockManager {
public:
    BlockManager() = delete;
    BlockManager(BufferManager &buffer_manager, const optional_idx block_alloc_size_p,
                 const optional_idx block_header_size_p);
    virtual ~BlockManager() = default;
    //! The buffer manager
    BufferManager &buffer_manager;
public:
    BufferManager &GetBufferManager() const { return buffer_manager; }
    //! Creates a new block inside the block manager
    virtual unique_ptr<Block> ConvertBlock(block_id_t block_id, FileBuffer &source_buffer) = 0;
    // ...
};
```

`BlockManager` is deliberately an interface, not just an implementation detail —
its abstract-base-class shape is what lets DuckDB's `BufferManager` (§17.3) sit
above storage-format-agnostic code, and lets alternative storage backends
(in-memory databases, or extensions attaching other file formats) implement the
same contract. The concrete implementation that backs a normal `.duckdb` file is
`SingleFileBlockManager` (`src/storage/single_file_block_manager.cpp`). It owns:

- **A free list of reusable block IDs.** When a block's contents become dead
  (superseded by a newer version written during a checkpoint, or a dropped
  table), its block ID goes onto an in-memory `free_list` set, persisted in the
  file header. `GetFreeBlockIdInternal` (`src/storage/single_file_block_manager.cpp:913`)
  pulls a reusable ID off that free list before ever growing the file with a
  brand-new block:
  ```cpp
  // src/storage/single_file_block_manager.cpp:913 (paraphrased structure)
  block_id_t SingleFileBlockManager::GetFreeBlockIdInternal(FreeBlockType type) {
      lock_guard<mutex> lock(block_lock);
      block_id_t block_id;
      if (!free_list.empty()) {
          block_id = *free_list.begin();
          free_list.erase(free_list.begin());
      } else {
          block_id = max_block++;
      }
      // ...
  }
  ```
  This free-list-plus-grow-on-demand scheme is what makes DuckDB's writes
  MVCC-friendly by construction (more in §17.6 and Chapter 18): a block is never
  overwritten in place while a reader might still be looking at its old
  contents. A write that changes data writes a *new* block and only later,
  once nothing needs the old version, returns the old block ID to the free
  list for reuse.
- **`magic_bytes.cpp`**, which sniffs a file's first bytes to decide what kind
  of file it's looking at before trying to open it as a DuckDB database —
  including catching the common mistake of pointing DuckDB at a SQLite or
  Parquet file:
  ```cpp
  // src/storage/magic_bytes.cpp:16-20 (paraphrased)
  static DataFileType ClassifyMagicBytes(const char *buffer) {
      if (memcmp(buffer, "SQLite format 3\0", 16) == 0) {
          return DataFileType::SQLITE_FILE;
      }
      if (memcmp(buffer, "PAR1", 4) == 0) {
          // ... Parquet
      }
      // ...
  }
  ```
  This is also where DuckDB's own storage-format version number is checked, so
  that an old build refuses to (mis)open a file written by a newer,
  format-incompatible DuckDB rather than silently corrupting it.

  The `StorageVersion` enum (`src/include/duckdb/storage/storage_info.hpp:134-148`)
  names v2.0.0 and v2.0.1. Both write storage version 69, so a v2.0.1 file is
  the same format as a v2.0.0 file. DuckDB also protects *older* readers. A
  file can keep an old storage version (below v1.5.0, which is 68) so that
  older DuckDB releases can still open it. Metadata blocks form chains: each
  block starts with a pointer to the next block. Readers before v1.4.2 follow
  that pointer as soon as they enter a block. If a checkpoint by a new DuckDB
  freed the next block and reused it for something else, an old reader would
  follow the pointer into garbage. So for such files, the `MetadataManager`
  also keeps the next block of every block it keeps (`keep_next_blocks`,
  `src/storage/metadata/metadata_manager.cpp:436-477`).
- **`partial_block_manager.cpp`**, which handles allocation at *sub-block*
  granularity: several small column segments (a short row group's tail, a
  handful of index leaf nodes) can be packed into one shared 256 KiB block
  rather than each claiming a whole block for a few kilobytes of data. Its
  header states the intent directly:
  ```cpp
  // src/include/duckdb/storage/partial_block_manager.hpp:89-92
  //! Enables sharing blocks across some scope. Scope is whatever we want to share
  //! blocks across. It may be an entire checkpoint or just a single row group.
  //! In any case, they must share a block manager.
  class PartialBlockManager {
  ```

  Every compressed segment is first built in a buffer of a full block
  (256 KiB). When the segment is small, the checkpoint copies it into a shared
  partial block. A partial block can collect thousands of tiny segments of a
  few dozen bytes each. If each segment kept pointing at its own 256 KiB buffer
  until the partial block was written to disk, thousands of such buffers would
  stay alive for no reason, and loading such data could spill or fail with an
  out-of-memory error. So `ColumnCheckpointState::FlushSegmentInternal` calls
  `ColumnSegment::UsePackedCopy` right after the copy
  (`src/storage/table/column_checkpoint_state.cpp:184`). The segment reads from
  the copy in the partial block from then on, and its own buffer is freed.
  `test/sql/storage/partial_blocks/partial_block_tail_memory.test` covers it.

One level up, `storage_manager.cpp` is the per-attached-database orchestrator
that owns *a* `BlockManager` (plus the catalog, the WAL, and — as of this
checkout — row-group sizing):

```cpp
// src/storage/storage_manager.cpp:458 (context)
idx_t row_group_size = DEFAULT_ROW_GROUP_SIZE;
```

`storage_manager.cpp` is where "attach this file" becomes "here is a
`BlockManager`, a `Catalog`, and a `WriteAheadLog`, all wired together" — it's
the thing `ATTACH 'orders.duckdb'` ultimately calls into.

## 17.2 Row groups and column segments: the columnar storage model

Below the block manager sits the part of DuckDB people usually mean when they
call it a "columnar database": tables are stored column-by-column, not
row-by-row, and that decision is made once, at the storage layer, and inherited
by everything above it.

### 17.2.1 Row groups

A table's data is divided horizontally into **row groups** — contiguous runs of
rows, each stored (and compressed, and checkpointed) mostly independently of
the others. The size is a single constant:

```cpp
// src/include/duckdb/storage/storage_info.hpp:26
//! The standard row group size
#define DEFAULT_ROW_GROUP_SIZE 122880ULL
```

122,880 is not an arbitrary round number — it's exactly 60 × `STANDARD_VECTOR_SIZE`
(2,048, the vector size fixed in Chapter 10):

```cpp
// src/include/duckdb/common/vector_size.hpp:16-20
#define DEFAULT_STANDARD_VECTOR_SIZE 2048U
// ...
#ifndef STANDARD_VECTOR_SIZE
#define STANDARD_VECTOR_SIZE DEFAULT_STANDARD_VECTOR_SIZE
#endif
```

122880 / 2048 = 60 exactly, and `storage_info.hpp` even has a compile-time
assertion enforcing that a row group size must be a whole number of vectors:

```cpp
// src/include/duckdb/storage/storage_info.hpp:404
#if (DEFAULT_ROW_GROUP_SIZE % STANDARD_VECTOR_SIZE != 0)
```

So one row group is 60 execution-sized vectors' worth of rows — a size chosen
to be large enough that per-row-group bookkeeping (zonemaps, §17.5; compressed
segment headers) is cheap relative to the data it describes, but small enough
that a single row group's worth of one column comfortably fits in a handful of
256 KiB blocks and can be skipped, scanned, or checkpointed as an independent
unit of work in parallel with other row groups.

`RowGroup` (`src/storage/table/row_group.cpp`, header at
`src/include/duckdb/storage/table/row_group.hpp`) is the class representing one
such horizontal slice:

```cpp
// src/storage/table/row_group.cpp:42-45
RowGroup::RowGroup(RowGroupCollection &collection_p, idx_t count)
    : SegmentBase<RowGroup>(count), collection(collection_p), version_info(nullptr), deletes_is_loaded(false),
      allocation_size(0), row_id_is_loaded(false), row_number_is_loaded(false), has_changes(false) {
	Verify();
}
```

A whole table is a `RowGroupCollection` (`src/storage/table/row_group_collection.cpp`)
— an ordered sequence of `RowGroup`s, each internally holding one `ColumnData`
per table column (`src/storage/table/column_data.cpp`,
`src/storage/table/standard_column_data.cpp` for fixed-width types, plus
`struct_column_data.cpp`, `list_column_data.cpp`, `array_column_data.cpp` for
nested types). Each `ColumnData` is, in turn, a sequence of `ColumnSegment`s —
the actual on-disk compressed byte ranges:

```cpp
// src/include/duckdb/storage/table/column_segment.hpp:38-45
enum class ColumnSegmentType : uint8_t { TRANSIENT, PERSISTENT };

class ColumnSegment : public SegmentBase<ColumnSegment> {
public:
	//! Construct a column segment.
	ColumnSegment(DatabaseInstance &db, shared_ptr<BlockHandle> block, const ColumnSegmentType segment_type,
	              const idx_t count, const CompressionFunction &function_p, BaseStatistics statistics,
	              const block_id_t block_id_p, const idx_t offset, const idx_t segment_size_p,
	              unique_ptr<ColumnSegmentState> segment_state_p = nullptr,
	              optional<uint32_t> data_size_in_bytes = optional<uint32_t>());
```

Note the fields on the constructor: a `CompressionFunction` (which scheme
compresses this segment — §17.4), a `BaseStatistics` (this segment's zonemap —
§17.5), and a `block_id_t` (which physical block, from the block manager in
§17.1, backs it). This is the whole hierarchy in one sentence: **a file is
blocks; a table is row groups; each row group is one `ColumnData` per column;
each `ColumnData` is a sequence of `ColumnSegment`s; each `ColumnSegment` is
compressed bytes living inside one or a few blocks.**

```
┌───────────────────────────────────────────────────────────────────────┐
│ orders.duckdb  (one file)                                              │
│                                                                          │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐   ...         │
│  │ Block #0       │  │ Block #1       │  │ Block #2       │            │
│  │ (256 KiB)      │  │ (256 KiB)      │  │ (256 KiB)      │            │
│  └───────────────┘  └───────────────┘  └───────────────┘              │
│         ▲                    ▲                   ▲                     │
│         └────────────────────┴───────────────────┴── BlockManager      │
│                                                        (§17.1)          │
└───────────────────────────────────────────────────────────────────────┘
                                 │  table "lineitem" = RowGroupCollection
                                 ▼
┌───────────────────────────────────────────────────────────────────────┐
│ RowGroup 0        rows [0, 122880)                                     │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌────────────┐│
│  │ ColumnData     │ │ ColumnData     │ │ ColumnData     │ │  ...       ││
│  │ l_orderkey     │ │ l_quantity     │ │ l_shipdate     │ │            ││
│  │  └─ segment(s) │ │  └─ segment(s) │ │  └─ segment(s) │ │            ││
│  │   BIGINT,       │ │   DECIMAL,      │ │   DATE,         │ │            ││
│  │   bitpacked     │ │   bitpacked FOR │ │   min/max stats │ │            ││
│  │   (§17.4)       │ │   (§17.4)       │ │   (§17.5)       │ │            ││
│  └───────────────┘ └───────────────┘ └───────────────┘ └────────────┘│
├───────────────────────────────────────────────────────────────────────┤
│ RowGroup 1        rows [122880, 245760)          ...                   │
└───────────────────────────────────────────────────────────────────────┘
```

### 17.2.2 Why this is what makes projection pushdown cheap

Because every column's segments are physically separate byte ranges — not
interleaved fields inside one shared row layout — a scan that only needs some
columns can skip the bytes belonging to the others *entirely*, without reading
them off disk (or, if they're cached, without even touching them in RAM).
Chapter 2's `awk`-vs-DuckDB benchmark measured exactly this effect from the
outside: `awk`'s line-oriented `getline` loop has to tokenize every field of
every line of `lineitem.csv` to compute a sum over one column, because a
row-major text format physically interleaves all of a row's fields together —
there is no way to "skip" `l_shipdate` or `l_comment` when all you want is
`l_quantity`. DuckDB's storage layer removes that constraint at the source: a
query like

```sql
SELECT SUM(l_quantity) FROM lineitem;
```

only ever asks the `ColumnData` for `l_quantity` for its segments; the
`l_shipdate`, `l_comment`, and every other column's segments are never opened,
never decompressed, and (if evicted from the buffer manager, §17.3) never even
paged in from disk. This is why Chapter 1 §1.3's physical plan for Query A
shows the pushed-down filter landing directly at the `lineitem` `LogicalGet` —
the optimizer's `filter_pushdown` pass (Chapter 7) is arranging for the scan to
apply `l.shipdate < DATE '1998-01-01'` and, separately, for the projection to
be narrowed to only the columns the rest of the query actually references
(`l.orderkey`, `l.extendedprice`, `l.shipdate` — not, say, `l.comment` or
`l.tax`), and the row-group/column-segment storage model documented here is
what turns "narrowed projection" from a plan-level intention into an actual
reduction in bytes read.

### 17.2.3 `DataTable` and `LocalStorage`: the abstraction a scan actually talks to

None of this is visible to a `PhysicalTableScan` directly. The operator asks a
`DataTable` (`src/storage/data_table.cpp`) for chunks; `DataTable` owns the
table's committed `RowGroupCollection` and is the facade every access path
(scans, appends, updates, deletes, index maintenance) goes through. Rows
inserted or updated by a transaction that hasn't committed yet don't live in
the committed `RowGroupCollection` at all — they live in a separate,
per-transaction structure:

```cpp
// src/include/duckdb/transaction/local_storage.hpp:142
//! The LocalStorage class holds appends that have not been committed yet
class LocalStorage {
```

A scan run by the same transaction that did the appending has to merge rows
from both places (the committed row groups, plus this transaction's own
`LocalStorage`); a scan run by a *different*, concurrent transaction sees only
the committed row groups as of its own snapshot. This is the seam where storage
(this chapter) meets MVCC (Chapter 18's `UndoBuffer`/`DuckTransaction`
machinery) — `LocalStorage` and `optimistic_data_writer.cpp` are the
storage-side half of "uncommitted rows exist somewhere, and readers need to
know whether they can see them"; Chapter 18 covers the transaction-visibility
half in full.

## 17.3 The buffer manager: DuckDB's page cache

Blocks on disk are 256 KiB of bytes; a query needs them as addressable memory.
The **buffer manager** is the layer in between — DuckDB's analogue of an
operating system's page cache, purpose-built for the block manager's needs.

`BufferManager` (`src/include/duckdb/storage/buffer_manager.hpp:27`) is,
again, an abstract interface, with `StandardBufferManager` as the concrete
implementation used by ordinary DuckDB databases:

```cpp
// src/include/duckdb/storage/buffer_manager.hpp:27-64 (elided)
class BufferManager {
public:
	//! Allocate block-based memory.
	//! The block manager provides the block size, and the block header size.
	//! Returns the BlockHandle managing the registered memory.
	virtual shared_ptr<BlockHandle> AllocateMemory(MemoryTag tag, BlockManager *block_manager,
	                                               bool can_destroy = true) = 0;
	//! Pin a block handle.
	virtual BufferHandle Pin(shared_ptr<BlockHandle> &handle) = 0;
	//! Unpin a block handle.
	virtual void Unpin(shared_ptr<BlockHandle> &handle) = 0;

	//! Returns the currently allocated memory.
	virtual idx_t GetUsedMemory() const = 0;
	//! Returns the maximum available memory.
	virtual idx_t GetMaxMemory() const = 0;
	//! Returns the currently used swap space.
	virtual idx_t GetUsedSwap() const = 0;
	// ...
};
```

**Pinning and unpinning** are the vocabulary this chapter shares with Chapter
14's memory-pool discussion, and it's worth being explicit that they are the
same mechanism seen from two different angles. Chapter 14 covers the *memory
accounting* view: the buffer pool's categories (paged/buffered base-table
pages vs. everything else) and how the global memory limit is partitioned
across concurrently running operators. This chapter is the *storage* view of
the identical mechanism: a `ColumnSegment` isn't directly readable memory until
something calls `Pin()` on its `BlockHandle`, which either finds it already
resident (a cache hit — no I/O) or loads it from the block manager (a cache
miss — one disk read of up to one block); `Unpin()` marks it as no longer
actively needed by that caller, making it *eligible* for eviction without
requiring it to be evicted immediately. A segment can be pinned by more than
one concurrent scan at once (each pin is a reference, not a lock).

### 17.3.1 Eviction

`BufferPool` (`src/include/duckdb/storage/buffer/buffer_pool.hpp:40`) is the
component that decides what to evict when memory is tight:

```cpp
// src/include/duckdb/storage/buffer/buffer_pool.hpp:38-46
//! The BufferPool is in charge of handling memory management for one or more databases. It defines memory limits
//! and implements priority eviction among all users of the pool.
class BufferPool {
	friend class BlockMemory;
	friend class BlockHandle;
	friend class BlockManager;
	friend class BufferManager;
	friend class StandardBufferManager;
```

Its eviction candidates live in an `EvictionQueue`, backed by a lock-free
concurrent queue (`moodycamel::ConcurrentQueue`) rather than a strict doubly
linked LRU list:

```cpp
// src/storage/buffer/buffer_pool.cpp:61-63
typedef duckdb_moodycamel::ConcurrentQueue<BufferEvictionNode> eviction_queue_t;

struct EvictionQueue {
```

Practically, this makes DuckDB's eviction policy **approximately LRU**, not
textbook-exact LRU: blocks are pushed onto the eviction queue roughly in the
order they become unpinned, `EvictBlocks`/`EvictBlocksInternal`
(`src/storage/buffer/buffer_pool.cpp:362`) pop from the front when memory needs
reclaiming, and a periodic `Purge`/`PurgeIteration` pass sweeps out stale
entries whose underlying block has since been re-pinned or already evicted
(tracked via a monotonic `handle_sequence_number` on each `BufferEvictionNode`
rather than by mutating a linked list under a global lock). This is the usual
tradeoff every high-throughput concurrent cache makes: give up exact recency
ordering in exchange for eviction and un-eviction that don't serialize every
thread through one lock.

### 17.3.2 Why base-table pages never need write-back

An evicted page belonging to a table that hasn't been modified since it was
last read from disk needs **no write-back** — its bytes in memory are, by
construction, identical to its bytes on disk, because that's exactly where
they were loaded from and nothing has touched them since. The buffer manager
can simply drop such a page and reload it later if needed; this is a *pure
read cache* over immutable on-disk data, in the same sense an OS page cache
over a read-only mmap needs no write-back.

This only holds because of how DuckDB *writes*, which is the same free-list
discipline from §17.1: a write never mutates an already-checkpointed block's
bytes in place while a reader might be looking at them. Instead, new or
modified data goes into **freshly allocated blocks** (drawn from the free list
or grown at the end of the file), and old blocks are only returned to the free
list — and thus only become reusable — once nothing needs them anymore. So the
picture is: reading is cache-only (pin/unpin, no write-back, ever, for
unmodified data); writing always produces new blocks; and the moment at which
"new blocks replace old ones as the source of truth" is precisely a
**checkpoint** (§17.6) or a transaction commit, both of which interact with the
write-ahead log and MVCC machinery that Chapter 18 covers in full. The buffer
manager itself stays simple — "cache blocks, evict without write-back" —
specifically because the write path upstream of it was designed to make that
simplicity safe.

## 17.4 Compression

Compression is where DuckDB's storage engine does its most concrete, most
inspectable engineering, and it's organized around one shared framework rather
than eleven unrelated ad-hoc schemes.

### 17.4.1 The analyze / compress / scan framework

Every compression scheme in `src/storage/compression/` implements the same
`CompressionFunction` interface — a bundle of function pointers for three
phases, documented directly in the header:

```cpp
// src/include/duckdb/function/compression_function.hpp:125-135
//===--------------------------------------------------------------------===//
// Analyze
//===--------------------------------------------------------------------===//
//! The analyze functions are used to determine whether or not to use this compression method
//! The system first determines the potential compression methods to use based on the physical type of the column
//! After that the following steps are taken:
//! 1. The init_analyze is called to initialize the analyze state of every candidate compression method
//! 2. The analyze method is called with all of the input data in the order in which it must be stored.
//!    analyze can return "false". In that case, the compression method is taken out of consideration early.
//! 3. The final_analyze method is called, which should return a score for the compression method

//! The system then decides which compression function to use based on the analyzed score (returned from final_analyze)
typedef unique_ptr<AnalyzeState> (*compression_init_analyze_t)(ColumnData &col_data, PhysicalType type);
typedef bool (*compression_analyze_t)(AnalyzeState &state, const Vector &input);
typedef idx_t (*compression_final_analyze_t)(AnalyzeState &state);
```

So this is not a heuristic bolted onto the side of one scheme — it's the
*entry point every scheme is required to have*: DuckDB genuinely samples a
column segment's actual data (via `analyze`), lets every applicable scheme for
that column's physical type score itself (`final_analyze`), and picks a
winner, before it ever compresses a byte for real. `Compress`/`Scan` are the
other two phases of the same interface (`compression_init_compression_t`,
`compression_compress_data_t`, `compression_scan_vector_t`, and friends,
immediately following in the same header) — one shared shape, eleven different
bodies. `standard_compression_state.cpp` provides a reusable base
(`StandardCompressionState`) that most schemes build their `CompressionState`
on top of, handling the boilerplate of creating and pinning a new
transient `ColumnSegment` and flushing a finished one:

```cpp
// src/storage/compression/standard_compression_state.cpp:23-31
void StandardCompressionState::CreateAndPinNewSegment() {
	auto compressed_segment = CreateNewSegment();
	current_segment = std::move(compressed_segment);

	auto &buffer_manager = BufferManager::GetBufferManager(current_segment->GetDatabase());
	handle = buffer_manager.Pin(current_segment->GetBlockHandle());
}
```

Note the direct link back to §17.3: even *building* a compressed segment goes
through the same `Pin()` call an ordinary scan uses.

### 17.4.2 Bitpacking (+ hugeint variant)

`bitpacking.cpp` implements **frame-of-reference (FOR) encoding with minimal
bit-width packing** — the default workhorse for integer columns. The idea,
concretely: within a block of values, subtract a common minimum (the "frame of
reference"), then store each residual using only as many bits as the largest
residual needs, rather than the type's full native width.

Take twelve `l_orderkey`-like values: `100000, 100003, 100001, 100007, 100002,
100004, 100000, 100006, 100005, 100001, 100003, 100002`. As 32-bit integers
that's 48 bytes. The minimum is 100000; subtracting it from every value gives
residuals `0, 3, 1, 7, 2, 4, 0, 6, 5, 1, 3, 2` — every residual fits in 3 bits
(`0`–`7`). Store the frame-of-reference value (`100000`) once, then twelve
3-bit residuals (36 bits, rounded up to 5 bytes) instead of twelve 32-bit
values (48 bytes) — roughly a 9× reduction for this made-up block, and the
general shape of the win any time a column's values cluster in a narrow range
relative to their type's full width (auto-incrementing keys, small deltas,
low-magnitude measures).

The source uses exactly this vocabulary — `BitpackingMode::FOR` and
`BitpackingMode::DELTA_FOR` (FOR applied to consecutive *deltas* rather than
raw values, for monotonic or near-monotonic columns) sit alongside `CONSTANT`
and `CONSTANT_DELTA` fast paths for degenerate blocks:

```cpp
// src/include/duckdb/storage/compression/bitpacking.hpp:15
enum class BitpackingMode : uint8_t { INVALID, AUTO, CONSTANT, CONSTANT_DELTA, DELTA_FOR, FOR };
```

and the compressor's own comments confirm the frame-of-reference terminology
directly:

```cpp
// src/storage/compression/bitpacking.cpp:349-351 (comments, in context)
// FOR (frame of reference).
// ...
// Aligned bitpacking width.
```

`bitpacking_hugeint.cpp` is the same algorithm specialized for 128-bit
`HUGEINT`/`UHUGEINT` columns, where the frame-of-reference arithmetic has to be
done carefully in 128-bit space.

### 17.4.3 Dictionary encoding

`src/storage/compression/dictionary/` (plus `dictionary_compression.cpp`)
implements classic dictionary encoding for low-cardinality columns: the
distinct values that actually occur are stored once, in a per-segment
dictionary, and each row stores a small bit-packed index into that dictionary
instead of a full copy of the value. A `status` column with three distinct
string values repeated across 122,880 rows benefits enormously — three strings
plus 122,880 tiny (2-bit) indices, instead of 122,880 full string copies.

The cardinality tradeoff isn't a hardcoded percentage cutoff — it falls out
naturally from a space check the analyzer runs incrementally as it walks the
column's values:

```cpp
// src/storage/compression/dictionary/analyze.cpp:30-38
bool DictionaryAnalyzeState::CalculateSpaceRequirements(bool new_string, idx_t string_size) {
	if (!new_string) {
		return DictionaryCompression::HasEnoughSpace(current_tuple_count + 1, current_unique_count, current_dict_size,
		                                             current_width, info.GetBlockSize());
	}
	next_width = BitpackingPrimitives::MinimumBitWidth(current_unique_count + 2); // 1 for null, one for new string
	return DictionaryCompression::HasEnoughSpace(current_tuple_count + 1, current_unique_count + 1,
	                                             current_dict_size + string_size, next_width, info.GetBlockSize());
}
```

Every new distinct value both grows the raw dictionary bytes *and* can push
the per-row index width up a bit (bitpacked via the same `MinimumBitWidth`
helper §17.4.2 uses), and `HasEnoughSpace` is checked against the segment's
fixed block budget. As distinct-value count climbs toward the row count, both
terms of that check grow until the segment stops fitting — which is the
concrete, mechanical reason dictionary encoding "stops paying off" as
cardinality approaches the row count: at the limit (every value distinct), the
dictionary is as large as an uncompressed column plus the overhead of the
index array on top, strictly worse than storing values directly.

### 17.4.4 FSST and dict_fsst

`fsst.cpp` implements **FSST (Fast Static Symbol Table)**, a string
compression scheme published by Boncz, Neumann and Leis [28]. The
core idea: build a small, per-block symbol table mapping common
short byte sequences (frequent substrings — not whole values, sub-string
fragments) to single-byte codes, then replace occurrences of those substrings
with their codes. Unlike a dictionary encoding (§17.4.3), FSST works even when
every *value* is distinct, because it compresses shared substructure *within
and across* strings rather than deduplicating whole values.

The property that makes FSST specifically interesting to a query engine,
rather than merely "a good string compressor," is that FSST-compressed bytes
can often be operated on **without fully decompressing them first** — notably
equality comparisons, and in favorable cases prefix comparisons, can be
evaluated directly against the compressed representation, because the same
symbol table maps a matching plaintext substring to the same code on both
sides. This is the same idea Chapter 10 introduces from the vector-execution
side as `FSST_VECTOR` — a first-class `VectorType` whose data stays encoded end
to end through parts of the execution pipeline instead of being eagerly
expanded to plain strings; this chapter is where that representation actually
originates, as bytes read straight off disk.

`dict_fsst.cpp` combines the two techniques: distinct strings are FSST-encoded
once into a shared per-segment dictionary, and rows store bit-packed indices
into that dictionary — dictionary encoding's "store the value once" plus
FSST's "and compress that one copy" for the tail of not-quite-duplicate
strings a pure dictionary wouldn't catch. Its own header comment lays out the
on-disk layout precisely:

```cpp
// src/storage/compression/dict_fsst.cpp:13-30 (comment, elided)
/*
Data layout per segment:
+-----------------------------------------------------+
|                  Header                             |
+-----------------------------------------------------+
|             Selection Buffer               |
|      tuple index -> index buffer idx       |
+--------------------------------------------+
|               Index Buffer                 |
|  string_index -> offset in the dictionary  |
+--------------------------------------------+
|                Dictionary                  |
|      the string data without lengths       |
+--------------------------------------------+
|             FSST Symbol Table (opt)        |
+--------------------------------------------+
*/
```

### 17.4.5 RLE (run-length encoding)

`rle.cpp` stores a consecutive run of identical values once, alongside a run
length, instead of storing the value once per row. It's a natural fit for
columns that are sorted, or that are naturally repetitive by construction
(a `status` column written in large batches of the same value, a foreign key
column scanned in an order that happens to cluster equal parent keys
together). The implementation is careful about IEEE-754 edge cases when
comparing floating-point values for run continuation — it compares raw bit
patterns rather than using `==` directly, because IEEE equality treats `+0.0`
and `-0.0` as equal even though their bit patterns differ:

```cpp
// src/storage/compression/rle.cpp:16-27
template <class T>
bool RLEValueEqual(const T &lhs, const T &rhs) {
	return lhs == rhs;
}

template <>
bool RLEValueEqual(const float &lhs, const float &rhs) {
	// IEEE == collapses +0 and -0; compare the bit pattern instead.
	return Load<uint32_t>(const_data_ptr_cast(&lhs)) == Load<uint32_t>(const_data_ptr_cast(&rhs));
}
```

### 17.4.6 Patas and Chimp

`chimp/` and `patas.cpp` are floating-point-specific compression schemes drawn
from the published Gorilla/Chimp family of time-series compression research:
Facebook's Gorilla [34], and Chimp [35], a
later refinement of it. The general
idea behind that family: rather than compressing each float independently,
successive values in the stream are **XORed against a reference value**
(typically the previous value, or a small ring buffer of recent values), and
because floating-point measurements in a column often change little from one
row to the next, that XOR result tends to have long runs of leading and
trailing zero bits — so the encoding stores only the sparse, differing middle
bits plus counts of the leading/trailing zeros it elided.

DuckDB's own Chimp implementation confirms this shape directly in source, not
just as a description of the literature:

```cpp
// src/include/duckdb/storage/compression/chimp/algorithm/chimp128.hpp:105-142 (elided)
CHIMP_TYPE xor_result;
// ...
auto tempxor_result = (CHIMP_TYPE)in ^ reference_value;
trailing_zeros = CountZeros<CHIMP_TYPE>::Trailing(tempxor_result);
// ...
if (xor_result == 0) {
    // ...
}
auto leading_zeros_raw = CountZeros<CHIMP_TYPE>::Leading(xor_result);
```

— an XOR against a reference value, followed by counting leading and trailing
zero bits, is exactly the Gorilla/Chimp family's core mechanism
[34], [35]. `Patas`
(`patas.cpp`) is a related, DuckDB-specific float scheme in the same
family of ideas (backward-reference-based packed encoding of the same kind of
XOR residual); its exact internal packing differs from Chimp's, but this book
doesn't assert stronger claims about Patas's precise bit layout than the
comments in `patas.cpp` support — the honest characterization is "the
general Gorilla/Chimp family idea, DuckDB's own variant."

### 17.4.7 ALP and ALPRD

`alp/` and `alprd.cpp` implement **ALP (Adaptive Lossless floating-Point
compression)**, from the paper by Afroozeh, Kuffó and Boncz [36] —
described in the source as DuckDB's newer default scheme for
`DOUBLE`/`FLOAT` columns, ahead of Patas/Chimp for the common case. ALP's core
idea, readable directly in its algorithm header: try to represent each
floating-point value as an **integer, scaled by a per-vector exponent/factor
pair**, so that the original float can be reconstructed exactly by dividing
back out — and for the (typically small) minority of values where that
round-trip doesn't reproduce the original bits exactly, store them as explicit
**exceptions** alongside their position, rather than falling back to storing
every value uncompressed:

```cpp
// src/include/duckdb/storage/compression/alp/algorithm/alp.hpp:67-88 (elided)
struct AlpCompressionData {
	AlpCompressionData() : vector_encoding_indices(0, 0), exceptions_count(0), bit_width(0) {
	}
	// ...
	AlpConstants::EXCEPTIONS_COUNT_TYPE exceptions_count;
	T exceptions[AlpConstants::ALP_VECTOR_SIZE];
	AlpConstants::EXCEPTION_POSITION_TYPE exceptions_positions[AlpConstants::ALP_VECTOR_SIZE];
};
```

and the encoder explicitly detects and patches those mismatches per vector:

```cpp
// src/include/duckdb/storage/compression/alp/algorithm/alp.hpp:318-345 (elided)
//! We detect exceptions using a predicated comparison
auto is_exception = (decoded_value != actual_value);
compression_data.exceptions_positions[exceptions_idx] = UnsafeNumericCast<uint16_t>(i);
exceptions_idx += is_exception;
// ...
// Replacing that first non exception value on the vector exceptions
```

The encoded integers are then themselves bitpacked with frame-of-reference
(§17.4.2) — so ALP is, in effect, "find a scaling that turns most of this
float column into a small-range integer column, bitpack that, and list the
few values the scaling couldn't represent exactly." Its win condition is data
that's genuinely decimal-precision-bounded in practice (prices, percentages,
sensor readings rounded to N digits) even though it's stored as an IEEE
double — real-world floating-point columns are very often exactly this kind of
data. `ALPRD` (`alprd.cpp`, `alprd_analyze.hpp`/`alprd_compress.hpp`/etc.) is a
related variant tuned for a different distribution of inputs (the source
splits candidate encodings across both `alp/` and `alprd/` and lets the
analyze step, §17.4.1, pick whichever scores better for a given segment); this
book doesn't go further than that into ALPRD's specific distinguishing
criteria, since the source comments don't spell out the split as explicitly as
they do the base ALP exception mechanism above. The ALP paper itself names the
variant ALP_rd ("real doubles") and presents it as the scheme for doubles with
too many significant digits to scale into small integers [36].

### 17.4.8 Roaring bitmaps

`src/storage/compression/roaring/` implements a **Roaring-bitmap**-style
compressed encoding (Roaring bitmaps are from Chambi, Lemire, Kaser and Godin
[37]), primarily for validity masks and boolean-heavy columns —
the storage-layer counterpart to the validity-mask representation Chapter 10
covers at the vector level. Rather than one flat bit per row regardless of
whether that bit is ever set, data is split into fixed-size containers, and
each container picks the cheapest of three representations for its own
content:

```cpp
// src/include/duckdb/storage/compression/roaring/roaring.hpp:28-109 (elided)
//! The amount of values that are encoded per container
// ...
res.container_type = ContainerType::RUN_CONTAINER;
// ...
res.container_type = ContainerType::ARRAY_CONTAINER;
// ...
static ContainerMetadata BitsetContainer(uint16_t container_size) {
	res.container_type = ContainerType::BITSET_CONTAINER;
```

A container that's a long unbroken run of set (or unset) bits — very common
for a `NOT NULL` column, which is "all valid" for its entire length — becomes a
`RUN_CONTAINER` (a start/length pair, near-free); a container with only a
handful of set bits among mostly-unset ones becomes an `ARRAY_CONTAINER` (list
the few set positions); only a container with a genuinely dense, unpredictable
mix of 0s and 1s falls back to a literal `BITSET_CONTAINER`. This maps neatly
onto real validity masks: an all-`NOT NULL` column's validity mask is one big
run container practically for free, while a column with a handful of scattered
`NULL`s among mostly-present values gets an array container listing just those
positions.

### 17.4.9 zstd

`zstd.cpp` wraps `third_party/zstd` as a general-purpose, dictionary-free block
compressor, available as a fallback or supplement for data (typically longer
strings, or `BLOB`s) that don't fit the profile any of the more specialized
schemes above target well. Its own layout comment shows it operating at
per-vector granularity within a segment, tracking each vector's compressed and
uncompressed size independently so a scan can seek directly to the vector it
needs:

```cpp
// src/storage/compression/zstd.cpp:17-26 (elided)
/*
Data layout per segment:
+--------------------------------------------+
|            Vector Metadata                 |
|   int64_t  page_id[]
|   uint32_t page_offset[]
|   uint64_t uncompressed_size[]
|   uint64_t compressed_size[]
*/
```

### 17.4.10 The honest fallback, and degenerate-case shortcuts

`uncompressed.cpp` / `fixed_size_uncompressed.cpp` / `validity_uncompressed.cpp`
are exactly what they sound like: when the analyze step (§17.4.1) can't find a
scheme whose `final_analyze` score beats simply storing the type's raw bytes,
storing the raw bytes wins — there's no shame in it in the framework's design;
it's one more candidate scored on equal footing with everything else, and for
genuinely high-entropy, no-structure data it's often correctly the winner.

Two more files handle cases that aren't really "compression" so much as
recognizing there's *nothing to store at all*:

- `numeric_constant.cpp` — an entire segment (up to all 122,880 rows of a row
  group) being one single repeated value collapses to storing that one value
  plus a count, with no per-row data whatsoever (a strict, degenerate special
  case of RLE with exactly one run).
- `empty_validity.cpp` — a segment that is entirely valid (no `NULL`s at all)
  or entirely `NULL` needs no validity bitmap at all:
  ```cpp
  // src/storage/compression/empty_validity.cpp:5-11
  CompressionFunction EmptyValidityCompressionFun::GetFunction(PhysicalType type) {
      return EmptyValidityCompression::CreateFunction();
  }

  bool EmptyValidityCompressionFun::TypeIsSupported(const PhysicalType physical_type) {
      return physical_type == PhysicalType::BIT;
  }
  ```
  This is the storage-layer twin of Chapter 10's `ConstantVector`/all-valid
  fast paths at the vector-representation level — the same "don't materialize
  what you don't need to" instinct, expressed on disk instead of in a
  `DataChunk`.

Constant compression has to watch for the same `-0.0` problem that §17.4.5's
RLE code avoids. A segment is stored as a constant when its statistics say
`max <= min`. But `-0.0 == 0.0`, so a segment holding both `-0.0` and `0.0`
could look constant, and after a checkpoint every value would come back with
one sign. Two rules prevent this. When `NumericStats::UpdateValue` sees a zero,
it prefers `-0.0` as the minimum and `0.0` as the maximum, so a mix of signs
shows up in the statistics. And `NumericStats::IsConstant` treats a zero range
as constant only when both bounds have the same sign
(`src/storage/statistics/numeric_stats.cpp:374-394`).
`test/sql/storage/compression/constant/constant_negative_zero.test` checks that
a table with the rows `-0.0` and `0.0` is not stored as `Constant`, while a
table of only `-0.0` (or only `0.0`) still is.

### 17.4.11 Reading compressed bytes safely

A compressed segment holds numbers that the reader must trust: a count of
dictionary entries, a bit width, an offset to the symbol table, and so on. If
the file is damaged, or crafted by an attacker, those numbers can point
outside the segment. A reader that trusts them reads (or writes) memory it does
not own.

DuckDB has a helper for this, `CompressionSegmentReader`. It is a bounds-checked
view over the bytes of one
segment:

```cpp
// src/include/duckdb/storage/compression/compression_segment_reader.hpp:22-31
//! A non-owning, bounds-checked view over [0, size) of a block or page, with a byte position.
//! It validates physical byte ranges, not row or codec invariants.
class CompressionSegmentReader {
public:
	CompressionSegmentReader(const_data_ptr_t data, idx_t size, const char *context)
	    : data(data), size(size), position(0), context(context) {
	}

	//! Returns a reader over the recorded segment bytes, or the remaining block bytes for legacy segments.
	static CompressionSegmentReader FromSegment(const BufferHandle &handle, const ColumnSegment &segment,
```

Each `Read<T>()`, `ReadBytes(n)` or `ReadArray<T>(n)` checks that the bytes it
asks for lie inside `[0, size)`. If not, it throws instead of reading. The
readers of bitpacking, RLE, the uncompressed formats, ALP, ALPRD, Chimp, Patas,
dictionary, FSST, DICT_FSST, Roaring and zstd all use it.

The note in the class comment matters: the reader checks *byte ranges*, not
the meaning of the values. So each scheme also checks its own rules. For
DICT_FSST the header is read once into a `SegmentLayout`, and its comments say
what was checked:

```cpp
// src/include/duckdb/storage/compression/dict_fsst/decompression.hpp:20-39
struct SegmentLayout {
	//! Compression mode read from disk, validated as mode < DictFSSTMode::COUNT.
	DictFSSTMode mode;
	//! Dictionary entry count read from disk, validated as nonzero.
	//! In FSST_ONLY mode, dict_count - 1 is at least the segment row count.
	//! With zero-width lengths, at most the NULL and empty-string entries are present.
	uint32_t dict_count;
	//! Bit widths read from disk, both validated as at most 32.
	bitpacking_width_t dictionary_indices_width;
	//! String lengths have a nonzero width in FSST modes.
	bitpacking_width_t string_lengths_width;
	//! Dictionary string bytes, FSST-encoded unless mode is DICTIONARY.
	CompressionSegmentReader dictionary;
	//! Serialized FSST symbol table.
	CompressionSegmentReader symbol_table;
	//! Packed per-entry string lengths from disk, decoded lengths are unvalidated.
	unsafe_array_ptr<const uint8_t> string_lengths;
	//! Packed row-to-dictionary indices from disk, decoded indices are unvalidated.
	unsafe_array_ptr<const uint8_t> dictionary_indices;
};
```

A bad value raises a `DataCorruptionException`, which names the problem,
instead of a crash. The other schemes have similar checks: Roaring checks that
its runs and array positions are in bounds and in order; zstd checks its
decompression progress and rejects cycles in its chain of overflow pages;
bitpacked `BOOLEAN` segments must decode to only 0 and 1. The test directory
`test/sql/storage/compression/dict_fsst/` shows the style. Each
`dict_fsst_corrupt_*.test` file (`dict_fsst_corrupt_mode.test`,
`dict_fsst_corrupt_dict_count.test`, `dict_fsst_corrupt_indices_width.test`,
and others) damages one field of a stored segment and expects a clear error.

## 17.5 Zonemaps and row-group skipping

Chapter 16's ART chapter introduces zonemaps briefly, in contrast to what an
ART index buys you beyond them; this section is the deeper, storage-focused
treatment zonemaps deserve on their own.

Every `ColumnSegment` carries a `BaseStatistics` alongside its compressed
bytes (visible in the `ColumnSegment` constructor in §17.2.1). For numeric and
date/time types, the concrete statistics are a plain min/max pair:

```cpp
// src/include/duckdb/storage/statistics/numeric_stats.hpp:27-34
struct NumericStatsData {
	//! Whether or not the value has a max value
	bool has_min;
	//! Whether or not the segment has a min value
	bool has_max;
	//! The minimum value of the segment
	NumericValueUnion min;
	//! The maximum value of the segment
	NumericValueUnion max;
};
```

This min/max pair *is* the zonemap: the cheapest possible summary of "what
values could possibly be in this row group's copy of this column," computed
once when the segment is written and consulted, essentially for free, before
any of its actual data is read.

`RowGroup::CheckZonemap` is the real function that puts this to use during a
scan:

```cpp
// src/storage/table/row_group.cpp:732 (signature)
bool RowGroup::CheckZonemap(optional_ptr<ClientContext> context, ScanFilterInfo &filters, idx_t row_start) {
```

It's called right before a row group's data is touched at all
(`src/storage/table/row_group.cpp:387`, and again at a finer, per-segment
granularity via `CheckZonemapSegments` at line 800, invoked from the scan loop
around line 945-946 with the comment `//! first check the zonemap if we have to
scan this partition`). If a pushed-down filter's bound is provably
inconsistent with a row group's (or segment's) min/max — for instance, the
filter requires `l_shipdate < DATE '1998-01-01'` but that row group's
`l_shipdate` zonemap reports `min = 1998-03-11`, entirely past the cutoff —
`CheckZonemap` returns false, and the *entire row group* (up to 122,880 rows,
across however many blocks its column segments occupy) is skipped without
reading a single one of its bytes. This is filter pushdown (Chapter 7) and
zonemap-based row-group skipping meeting at the storage boundary: the
optimizer's job is to get the filter down onto the scan; this chapter's job is
what the scan does with it once it's there.

A zonemap must also say whether a segment *can* hold `NULL`. A filter such as
`x IS NULL OR x > 2` must keep a segment with `NULL`s even when no value is
above 2. Files written before storage version 2.0 (for example by v1.5.6)
recorded `NULL`s only in the statistics of the separate *validity* segments,
not in the data segments. A reader that checks only the data segments would
see "no `NULL`s, no values above 2" and skip segments that do hold matching
rows. DuckDB v2.0.0 had exactly this bug: with 1,024 ones and 1,024 `NULL`s,
`count(*) ... WHERE x IS NULL OR x > 2` returned 0 instead of 1,024.
`StandardColumnData::InitializeColumn`
(`src/storage/table/standard_column_data.cpp:341-359`) handles it. When a
column was written in the old format (its segments have no recorded byte size)
and its validity says it can hold `NULL`s, every data segment is marked as "may
hold `NULL`". `test/sql/storage/bc/null_rows_v1_5_or_filter.test` opens a
v1.5.6 file and checks such filters.

**When zonemaps help a lot, and when they barely help at all** is entirely a
question of physical correlation between the filtered column and insertion
(or clustering) order — it has nothing to do with the filter's own selectivity
in the abstract:

- If `lineitem` were loaded, or later reorganized (e.g. by `INSERT ... SELECT
  ... ORDER BY l_shipdate`, or reconstructed during a checkpoint that happens
  to preserve rough temporal order — Chapter 7's `row_group_reorderer.cpp` is
  the mechanism a checkpoint can use for this), such that each row group's rows
  cluster into a narrow date range, then a filter like Query A's
  `l.shipdate < DATE '1998-01-01'` is enormously effective: every row group
  whose *entire* date range is after the cutoff gets skipped wholesale, and
  only the (few) row groups actually straddling the boundary need scanning at
  all. In the limit — perfectly sorted data — a selective range filter costs
  roughly one row group's worth of real work, no matter how big the table is.
- If `lineitem`'s rows are in no particular order with respect to
  `l_shipdate` — insertion order unrelated to shipping date, or a table that's
  had years of scattered updates — then almost every row group's min/max
  span will straddle the filter's cutoff regardless of how selective the
  predicate looks in aggregate. `CheckZonemap` still runs, and still correctly
  returns `false` for any row group that happens to fall entirely on the wrong
  side, but with uncorrelated data that almost never happens — nearly every
  row group needs to be opened and scanned row-by-row, and the net effect of
  the zonemap check is close to zero. The predicate is exactly as selective
  either way; what changes is whether that selectivity is *visible at the
  row-group granularity* zonemaps operate at.

This is why physical clustering (via `ORDER BY`-on-load, periodic
reclustering, or a naturally-sequential insertion pattern like an append-only
event log keyed by timestamp) is a real, actionable performance lever for
analytical workloads on DuckDB, independent of anything an index could do —
and also why Chapter 16 introduces the ART index as the tool for the opposite
situation: point lookups and equality/range filters on columns that *aren't*
physically correlated with row-group layout, where a zonemap's row-group-level
granularity is too coarse to help and something with per-value precision is
needed instead.

## 17.6 Checkpointing

A `CHECKPOINT` (issued explicitly, or run implicitly by the storage manager
under its own policy) is what turns the live, possibly-scattered state built
up by ordinary transactional writes back into a compact set of fresh blocks.
`checkpoint_manager.cpp` and `src/storage/checkpoint/` own this:

```cpp
// src/include/duckdb/storage/checkpoint_manager.hpp:45-78 (elided)
//! Wrapper to manage the lifetime of a checkpoint connection and transaction.
// ...
virtual void CreateCheckpoint() = 0;
```

Conceptually, `CreateCheckpoint` walks every table's row groups, writes out any
row groups (or column segments) that have changed since the last checkpoint
into fresh blocks (using `PartialBlockManager`, §17.1.1, to pack small
leftover segments efficiently rather than wasting whole blocks on them),
updates the catalog metadata to point at the new block locations, and then
returns the *old* blocks — the ones nothing points to anymore — to the block
manager's free list. Two effects fall out of that directly: **space is
reclaimed** (dead versions of updated/deleted rows, and blocks made obsolete by
compaction, become reusable rather than sitting around as permanent file
bloat), and **the file is compacted** (data that was scattered across
non-contiguous blocks by a sequence of small transactional writes gets
rewritten more contiguously).

A checkpoint also writes which rows are deleted. For each vector of 2,048 row
slots, `ChunkVectorInfo::Write` stores "nothing deleted", "everything
deleted", or a bitmask of the deleted slots.

`Write`, `HasDeletes` and `CompressVersionIds` take a `row_count` and look
only at the slots that hold rows. The slots past `row_count` are written as
"not deleted" (`src/storage/table/chunk_info.cpp:810-835`). This matters in one
case. Suppose a table holds 100 rows, so its first vector is only partly used.
A transaction appends 500 more rows into that vector, and then its commit fails
(in the test, because another transaction altered the table first). The
rollback cuts the vector back to 100 rows, but the unused slots after them
still carry the failed transaction's insert ID. A checkpoint that looked at all
2,048 slots would see that ID and write those slots as deleted. Rows appended
into the same slots later would then be treated as deleted after a restart.
`test/sql/storage/revert_append_partial_vector.test` guards against this.

This chapter deliberately stops at "what a checkpoint does to blocks and row
groups" — how a checkpoint interacts with the write-ahead log (why a
checkpoint lets old WAL entries be discarded), and how it relates to which
transactions' writes are visible when it runs, is squarely Chapter 18's
territory (`write_ahead_log.cpp`, `wal_replay.cpp`, and the
`DuckTransaction`/`UndoBuffer` MVCC machinery from the research notes' §7b–7c),
and is covered there in full.

## 17.7 Temporary files: the storage-layer machinery behind spilling

Chapter 14 describes *when* an operator needs to spill — a hash join's build
side, or a sort, whose intermediate state has grown larger than the memory
budget the buffer pool (§17.3) can grant it. `temporary_file_manager.cpp` and
`temporary_memory_manager.cpp` are the concrete machinery that makes spilling
actually happen at the storage layer once that decision is made.

`TemporaryFileManager` is the piece that owns actual files on disk for spilled
data:

```cpp
// src/include/duckdb/storage/temporary_file_manager.hpp:285-290 (context)
//===--------------------------------------------------------------------===//
// TemporaryFileManager
//===--------------------------------------------------------------------===//
class TemporaryFileManager {
	friend struct BlockIndexManager;
	friend class TemporaryFileHandle;
```

When an operator's block of intermediate state (a partition of a hash table
too large to keep resident, a run of a sort that doesn't fit in memory) is
evicted under memory pressure, `TemporaryFileManager` is what actually writes
those bytes to a temp file and — later, when the operator needs that data back
to resume its work — reads them back in. It reuses the same `BufferManager`
allocation path (§17.3) that ordinary table blocks use
(`BufferManager::AllocateTemporaryMemory`), so from the buffer pool's point of
view a spilled hash-table partition and a base-table column segment are both
just memory-tagged, pinnable/unpinnable allocations — the difference is only
where their bytes live when evicted (a temp file, versus the permanent
`.duckdb` file's own blocks).

Three details shape how spilling behaves.

- **The size limit is enforced atomically.** The setting
  `max_temp_directory_size` caps the total size of the temp files (by default
  90% of the free disk space). `IncreaseSizeOnDisk` reserves the bytes with a
  compare-and-swap loop, so two threads cannot both pass the check and
  together go past the limit (`src/storage/temporary_file_manager.cpp:647-661`).
- **A limit given at startup is used.** A `max_temp_directory_size` passed in
  the startup config (for example `duckdb.connect(config={...})` in Python) is
  read by the buffer manager when the database starts, just like a later `SET`
  (Chapter 14 §14.3).
- **Small segments can be spilled too.** A column segment that is still being
  filled (a *transient* segment) starts small. A small segment either lives
  inside a shared full-size block, which can be evicted to a temp file, or
  gets a tiny buffer of its own, which can never be evicted. The shared block
  is passed down to every column: top-level columns, validity columns, and the
  children of `STRUCT`, `LIST`, `ARRAY` and `VARIANT` columns. Without that,
  an in-memory database could not load much more than `memory_limit`, even
  with a temp directory. (In DuckDB v2.0.0 only top-level columns got the
  shared block, and loading 20 million rows with two structs and a list into a
  256 MB in-memory database failed with out-of-memory.) The test
  `test/sql/storage/temp_directory/offload_small_transient_segments.test`
  loads 8 million rows with a nullable integer, a struct and a list into an
  in-memory database with `memory_limit='32MB'`.

`TemporaryMemoryManager` sits one layer up, deciding *how much* memory each
concurrently running operator's spillable state is allowed to hold onto before
being asked to spill more:

```cpp
// src/include/duckdb/storage/temporary_memory_manager.hpp:69-71
//! TemporaryMemoryManager is a one-of class owned by the buffer pool that tries to dynamically assign memory
//! to concurrent states, such that their combined memory usage does not exceed the limit
class TemporaryMemoryManager {
```

— i.e., it's the arbiter that answers "given N concurrent operators each
wanting a big chunk of memory for their hash table or sort buffer, and one
global memory limit, how much does each get before it has to start spilling to
the `TemporaryFileManager`." Chapter 14 covers the policy questions (how much
each operator is granted, and why); this section is simply where to look for
the code that turns "please spill" into bytes actually leaving RAM and landing
on disk, and back again.

## 17.8 Summary

The physical picture this chapter builds, end to end: one `.duckdb` file,
divided into 256 KiB blocks (`DEFAULT_BLOCK_ALLOC_SIZE`) managed by a
`BlockManager`/`SingleFileBlockManager` that reuses freed blocks rather than
overwriting live ones in place; each table divided into 122,880-row row groups
(`DEFAULT_ROW_GROUP_SIZE`, exactly 60 execution vectors), each row group
storing every column as an independently compressed `ColumnSegment` — the
physical fact behind "columnar storage" and behind why projection pushdown
(Chapter 2, Chapter 7) actually saves bytes read, not just plan-tree
complexity. A `BufferManager`/`BufferPool` sits between those blocks and
anything that wants to read them, pinning and unpinning them into an
approximately-LRU cache that needs no write-back for unmodified base-table
data, because writes always land in fresh blocks (Chapter 18 covers why, via
the WAL and MVCC). Eleven-odd compression schemes — bitpacking, dictionary,
FSST/dict_fsst, RLE, Patas/Chimp, ALP/ALPRD, Roaring, zstd, and the
uncompressed/degenerate fallbacks — all plug into one shared analyze/compress/
scan interface (`CompressionFunction`), so the "pick the best scheme" decision
is a real, data-driven scoring competition, not a fixed per-type rule, and
their readers check every byte range and header value they read from disk
(§17.4.11). Each
segment's min/max zonemap lets a scan skip entire row groups without reading
them (§17.5) whenever the filtered column happens to be physically correlated
with row-group layout — the storage-layer payoff for Chapter 7's filter
pushdown, and the coarser-grained sibling to Chapter 16's ART index.
Checkpointing periodically compacts all of this back into a tidy set of fresh
blocks, and the same block/buffer-manager machinery, borrowed rather than
duplicated, is what backs Chapter 14's spill-to-disk story for operators that
outgrow their memory budget. Chapter 18 picks up from here: the write-ahead
log and the MVCC/undo-buffer machinery that make all of these writes durable
and correct under concurrent transactions.
