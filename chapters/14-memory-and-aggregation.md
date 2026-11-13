# Chapter 14 — Memory Management & Grouped Aggregation

This chapter is built primarily from **DiDi Chapter 3, "Managing Memory + Grouped
Aggregation"** [5] (slides dated April 7, 2026), cross-checked wherever
possible against the `v2.0-cyanoptera` source tree at commit `cb5673685153`.
Slide citations below refer to that PDF's own page numbers (each slide is
numbered in its footer, `01` through `16`).

## 14.1 The memory hierarchy, restated as a design constraint

Chapter 2 (§2.1) already introduced the latency table that motivates almost
every mechanism in this book, borrowed from the same source this chapter draws
on — DiDi Chapter 3's own opening slide:

```
 Memory        Actual Latency     "Human Scale"      Typical Size
 CPU L2 cache        2.8 ns             1 second       ¼ – 16 MB
 RAM               ≈ 100 ns            36 seconds      4 – 256 GB
 SSD             50 – 150 µs           5 – 15 hours    ½ – 4 TB
 HDD                1 – 10 ms         4 – 40 days       1 – 16 TB
```

Chapter 2 used this table to explain why a query engine's inner loop is so
sensitive to memory access patterns. This chapter is about the other half of
the same slide's argument [5, slide 02]: faster memory is *always*
drastically smaller than slower memory, by construction — there is no
engineering trick that fixes this, because the physical mechanisms that make
SRAM fast are the same mechanisms that make it expensive and therefore scarce.
DiDi states the conclusion as bluntly as the slide allows: **"We will not be
able to build cache-only DBMSs."** Given that constraint, a database engine has
exactly one lever left: decide, continuously and automatically, what currently
counts as "hot" and deserves to live in RAM, and what can be pushed down to
disk without breaking correctness. Three policy statements fall directly out
of this [5, slide 02]:

- Keep persistent database data on secondary storage (disk).
- During query processing, try to keep all of the currently relevant ("hot")
  data in RAM.
- In general, try to process every query as high up this hierarchy as
  possible.

Everything else in this chapter — `memory_limit`, the three-way memory pool,
streaming, spilling, row-major intermediates, German strings, and the whole of
`HASH_GROUP_BY` — is machinery built to satisfy those three sentences
mechanically, without the query author having to think about them.

### `memory_limit`: how much RAM DuckDB is willing to use

DuckDB's answer to "how much of this precious resource do I get to use" is a
single setting, `memory_limit`, defaulting to **80% of host RAM** [5, slide 03]. This is not a rule of thumb quoted secondhand — it is exactly what
the source computes:

```cpp
// src/main/config.cpp:675-684
void DBConfig::SetDefaultMaxMemory() {
	auto memory = GetSystemAvailableMemory(*file_system);
	if (memory == DBConfigOptions().maximum_memory) {
		// If GetSystemAvailableMemory returned the default, use it as is
		options.maximum_memory = memory;
	} else {
		// Otherwise, use 80% of the available memory
		options.maximum_memory = memory * 8 / 10;
	}
}
```

DiDi's own footnote on this slide contrasts it with a well-known alternative
default: PostgreSQL ships with `shared_buffers` set to a fixed **128 MB**,
regardless of how much RAM the host actually has. DuckDB's 80%-of-host-RAM
default is only sensible *because* of a second design decision this chapter
comes back to repeatedly: DuckDB shares an address space with its host process
(it's an in-process library, not a standalone server), so it has to behave
well both when it's the only thing using the machine and when it's embedded in
a process doing a hundred other things — which is why the fraction is
"available," not "total," and why (as §14.2 shows) DuckDB refuses to
pre-allocate a fixed-size buffer pool the way a dedicated database server can
afford to.

## 14.2 One memory pool, three allocation categories

A naive reading of "keep 80% of RAM for query processing" suggests a single
undifferentiated arena that anything can allocate from. DuckDB instead
recognizes three *kinds* of memory demand that behave so differently under
memory pressure that treating them identically would be actively harmful
[5, slides 06–08]:

```
                     Non-Paged Intermediate    Paged Intermediate      Buffered Base-Table Data
 Size                flexible                 typical 32–256KB          256KB
                                               (max 2^62 bytes)
 Lifetime             query                    query or session          across queries
                                                                          (database session)
 Spillable?           no                       yes                       (no write-back needed)
```

The reason for three categories, not one, is that each answers a different
question about what happens when memory gets tight:

1. **Non-paged allocations.** Small, fragmented bits of memory — "as used in
   pointer-based data structures" [5, slide 06]. These **cannot be
   spilled at all**, for a structural reason, not a policy choice: something
   elsewhere in the process holds a raw C++ pointer directly into this memory,
   and that pointer is scattered arbitrarily across other data structures with
   no central registry of who holds it. There is no way to move the
   underlying bytes to disk and later bring them back at a *different* address
   without invalidating every one of those pointers, and no way to find and
   fix up all of them. The `GroupedAggregateHashTable`'s own pointer array is
   exactly this kind of allocation — it is fetched through the general-purpose
   buffer allocator, not through the page/block machinery described next, and
   every one of its slots holds a raw `data_ptr_t` into row data elsewhere:

   ```cpp
   // src/execution/aggregate_hashtable.cpp:347-350  (GroupedAggregateHashTable::Resize)
   auto new_hash_map = buffer_manager.GetBufferAllocator().TryAllocateHuge(size * sizeof(ht_entry_t));
   capacity = size;
   hash_map = std::move(new_hash_map);
   entries = reinterpret_cast<ht_entry_t *>(hash_map.get());
   ```

   This allocation still counts against `memory_limit` (it goes through the
   buffer manager's memory accounting), but it is not wrapped in a `Pin()`
   /`Unpin()` handle the way a page of row data is — there is nothing that
   could safely relocate it.

   The pointer array is allocated only in `Resize`, with
   `TryAllocateHuge`. The new array is allocated *before* the old one is
   released. If there is not enough memory, the call throws an
   out-of-memory error and the old array is still there. §14.6 shows why
   this matters: the sink may try to grow the array during a query, and
   it must be able to back off when the memory is not there.

2. **Paged allocations.** Blocks of contiguous data, typically 32–256KB,
   holding parts of tabular data structures — hash table row storage, sort
   buffers, join build sides. These **can** be spilled, because every access
   to this memory goes through one level of indirection (a `BlockHandle`, a
   `TupleDataChunkPart`) rather than a bare pointer. When memory is tight, the
   buffer manager can write such a block out, mark it unpinned, and reload it
   later at a *different* address — because the one place that referenced it
   directly (the handle) gets updated, and everything else follows the handle
   rather than a raw pointer. §14.4–14.5 return to exactly this mechanism for
   row and string data.

3. **Buffered base-table pages.** Data read from the persistent `.db` file,
   cached in RAM in case a later query needs the same page again. These
   **never need to be written back on eviction**, for a reason that has
   nothing to do with cleverness: they are a pure read cache over bytes that
   already exist, correctly, on disk. Evicting one is just "forget the copy" —
   there is nothing to reconcile. (This describes the common case of
   unmodified table data such as this chapter's read-mostly `orders`/
   `lineitem` example; a page holding an uncommitted write does need to reach
   disk eventually, through the WAL/checkpoint machinery Chapters 16–18
   cover — but the read cache described here is deliberately *not* where that
   durability guarantee lives.)

The source's real evidence for this three-way split isn't a single named
enum — it shows up as two related, finer-grained mechanisms that jointly
implement it. First, every buffer-managed allocation carries a `MemoryTag` for
accounting purposes, and `BASE_TABLE` and `HASH_TABLE` are two of its sixteen
values — proof that the engine tracks "this memory is cached table data" and
"this memory is an intermediate's working set" as genuinely distinct
categories, not an afterthought:

```cpp
// src/include/duckdb/common/enums/memory_tag.hpp
enum class MemoryTag : uint8_t {
	BASE_TABLE = 0,
	HASH_TABLE = 1,
	PARQUET_READER = 2,
	CSV_READER = 3,
	ORDER_BY = 4,
	ART_INDEX = 5,
	COLUMN_DATA = 6,
	METADATA = 7,
	OVERFLOW_STRINGS = 8,
	IN_MEMORY_TABLE = 9,
	// ... 6 more categories (ALLOCATOR ... OBJECT_CACHE), then UNKNOWN = 16
};
```

Second, and more directly tied to *eviction behavior specifically*, every
buffered block carries a `FileBufferType`, and the buffer pool's eviction
queue treats the three non-tiny types with three different priorities:

```cpp
// src/storage/buffer/buffer_pool.cpp:15-27
static idx_t FileBufferTypeToEvictionQueueTypeIdx(const FileBufferType &type) {
	switch (type) {
	case FileBufferType::BLOCK:
	case FileBufferType::EXTERNAL_FILE:
		return 0; // Evict these first (cheap, just free)
	case FileBufferType::MANAGED_BUFFER:
		return 1; // Then these (have to write to storage)
	case FileBufferType::TINY_BUFFER:
		return 2; // Evict tiny buffers last (last resort)
	default:
		throw InternalException("Unknown FileBufferType in FileBufferTypeToEvictionQueueTypeIdx");
	}
}
```

Read the comments literally and DiDi's three categories fall out directly:
`BLOCK`/`EXTERNAL_FILE` pages (base-table pages read from the `.db` file, or
from an external Parquet/CSV file) are evicted *first*, precisely because they
are "cheap, just free" — no write-back required, matching category 3 exactly.
`MANAGED_BUFFER` pages (paged intermediates that own real, still-needed
working data) are evicted only *after* that, because eviction here really does
mean "have to write to storage" first — matching category 2. This is real
source confirming, at the level of actual eviction-queue priority (not just
prose), that the three categories aren't a teaching simplification layered
over one undifferentiated pool — they get measurably different treatment by
the one component (`BufferPool`) responsible for deciding what to throw away
first when memory runs out.

```
  memory_limit  (default: 80% of host RAM)
  ┌──────────────────────────────────────────────────────────────────┐
  │  non-paged          paged intermediates       buffered base-table │
  │  (hash array,       (sort buffers, hash        pages (.db file    │
  │  small pointer      table row storage,         blocks, read       │
  │  structures)        join build sides)          cache)             │
  │                                                                    │
  │  spill: NEVER       spill: YES, to disk        spill: N/A —        │
  │  (raw pointers      (handle indirection        never written back  │
  │  everywhere)        makes relocation safe)     (pure read cache)   │
  └──────────────────────────────────────────────────────────────────┘
        evict priority:        2nd                       1st (cheapest)
```

DiDi's own framing of *why this matters* is worth quoting directly, because it
states the contrast with "conventional" database engineering precisely: **"In
contrast to many DBMSs that require a pre-configuration of buffer size (# of
pages) or memory size for intermediates"** [5, slide 08]. A server-style
DBMS with a fixed `shared_buffers` has already drawn a line, at startup, between
"memory for caching table pages" and "memory for everything else" — and that
line doesn't move no matter what a given query needs. DuckDB's single unified
pool, sliced dynamically into these three categories on demand, means a
query with a huge `GROUP BY` and no repeat table scans can claim almost all of
`memory_limit` for paged intermediates, while a query that scans the same
table repeatedly can claim almost all of it for base-table caching — with no
restart, no reconfiguration, and (per the footnote already quoted in §14.1)
no wasted memory sitting idle when DuckDB's connection is idle inside a larger
host process.

## 14.3 Streaming vs. spilling: two different answers to "too much data"

Given a fixed `memory_limit`, a query whose input is larger than that limit
has to be handled somehow. DuckDB has two structurally different answers, and
which one applies to a given operator is decided by whether that operator is a
**pipeline breaker** — vocabulary Chapter 11 develops in full, but enough of
which is needed here to draw the line correctly.

**Streaming** is the answer for anything that can process one
`DataChunk` — Chapter 10's term for a batch of up to 2048 rows across all
needed columns — at a time and immediately hand its output chunk downstream,
never needing to look at a row again once it's been passed along. DiDi lists
the concrete operator shapes that qualify [5, slide 04]:

- Simple `WHERE` filters and `SELECT` projections.
- Ungrouped aggregates, or `GROUP BY` where the number of distinct groups is
  small (this is exactly where `PERFECT_HASH_GROUP_BY`, §14.8, becomes
  relevant — a small, bounded group count is a *streamable* aggregation).
- `LIMIT H` for small `H`.
- Reading from one format and writing to another (CSV → Parquet, `COPY`).

```
   ┍━┯━┯━┑
   │=│▇│▇│  ← one 2048-row vector at a time
   │=│▇│▇│     (=: irrelevant, projected-away column)
   │=│▇│▇│
   └─┴─┴─┘
     │
     ▼  A: route this one chunk through the query plan
  [ FILTER ] → [ PROJECT ] → [ ... ] → result
     │
     ▼  B: memory requirement at any instant is just
          (input chunk) + (output chunk) + δ (small
          per-thread intermediate state) — NOT the
          whole table.
```

The consequence DiDi draws out explicitly [5, slide 04] is the one worth
remembering: **such queries can run over larger-than-memory tables even under
a very constraining `memory_limit`**, because at no point does the engine
attempt to hold more than a handful of chunks in memory simultaneously per
thread. This is the mechanism, forward-referenced from Chapter 1 §1.3, that
lets a `PhysicalTableScan → PhysicalFilter → PhysicalProjection` chain run
against a table many times the size of RAM without anything in this chapter's
memory-pool machinery ever being invoked.

**Spilling** is the other answer, needed exactly when an operator is a *sink*
— a pipeline-breaking operator that must accumulate state across the *entire*
input before it can produce its first output row. DiDi's list of concrete
triggers [5, slide 05] is the list every later section of this chapter
returns to:

- `GROUP BY` where the grouping criteria create many groups (a large hash
  table).
- `count(DISTINCT column)` with many distinct values (also a hash table under
  the hood).
- Joining two tables that are both larger than memory (the build side's hash
  table).
- `ORDER BY` over larger-than-memory input.
- A window function with a "sweeping frame" evaluated over larger-than-memory
  input.

Every one of these needs to see the whole input before any output can be
correct — you cannot know whether `custkey` 42's running total is final until
every row that might belong to group 42 has been processed, so there is no
chunk-at-a-time shortcut available; the accumulated state (a hash table, a
sort buffer) can legitimately outgrow `memory_limit`, and when it does,
DuckDB "temporarily uses out-of-core memory on disk to hold (parts of) large
intermediates" — DB lingo: **spilling** [5, slide 05]. Everything from
§14.4 onward is the detailed mechanics of spilling, for the specific case of
hash-based grouped aggregation.

### Where spilled data goes, and how much of it is allowed

Spilled blocks are written to files in the *temporary directory*
(`temp_directory`). A second setting, `max_temp_directory_size`, caps how many
bytes those files may hold in total. By default the cap is 90% of the free
disk space where the temporary directory lives. When a spill would go past
the cap, the query fails with an out-of-memory error that starts with
"failed to offload data block of size ..." and names the setting.

The check lives in `TemporaryFileManager::IncreaseSizeOnDisk`. It reserves the
space with an atomic compare-and-swap loop, so two threads that
spill at the same moment cannot both pass the check and together go over the
cap:

```cpp
// src/storage/temporary_file_manager.cpp:647-661 (message text elided)
void TemporaryFileManager::IncreaseSizeOnDisk(idx_t bytes) {
	auto current_size_on_disk = size_on_disk.load();
	do {
		if (current_size_on_disk + bytes > max_swap_space) {
			// ...
			throw OutOfMemoryException(R"(failed to offload data block of size %s (%s/%s used).
This limit was set by the 'max_temp_directory_size' setting.
// ...
		}
	} while (!size_on_disk.compare_exchange_weak(current_size_on_disk, current_size_on_disk + bytes));
}
```

The setting can be given at two times. It can be set with `SET` on an open
database, or passed in the startup configuration (for example
`duckdb.connect(config={...})` in Python). A value from the startup
configuration is stored in `DBConfig`, so the buffer manager reads it from
there when it is created:

```cpp
// src/storage/standard_buffer_manager.cpp:80-84
	// 'max_temp_directory_size' set before the database was created is only stored in the config
	auto &config = DBConfig::GetConfig(db);
	if (config.options.maximum_swap_space != DConstants::INVALID_INDEX) {
		temporary_directory.maximum_swap_space = config.options.maximum_swap_space;
	}
```

The C++ test "Test max_temp_directory_size set at startup is enforced"
(`test/api/test_api.cpp:192`) opens a database with the cap set to `0KiB` in
the startup config and checks that a query that must spill fails with
"failed to offload data block". It guards against a bug where a startup value
was stored but never read, so the default 90%-of-disk cap applied.

## 14.4 Row-major layout for paged intermediates

Base tables in DuckDB are stored column-wise (Chapter 17 covers this in
depth): each column's values sit contiguously, which is exactly what lets a
scan that only needs `l_quantity` out of `lineitem`'s sixteen columns (Chapter
2 §2.3.7) skip reading the other fifteen entirely. Paged *intermediate* data
structures — the working set of a hash table, a sort buffer, a join's build
side — make the opposite choice: **row-major layout** [5, slide 09].
This looks, at first glance, like giving back exactly the locality advantage
column storage bought. It isn't, because the access pattern is different:

- A table scan touches *a few columns, across many rows* — column-major keeps
  those few columns' bytes contiguous and skips the rest.
- A sort, a hash probe, or a join comparison touches *all (or most) columns of
  one candidate row at a time* — you need the whole row together to decide
  "does this row match," "how does this row compare to that one," or "what's
  this row's full payload to emit downstream." Row-major layout is what keeps
  that whole-row access cache-local; scattering the same row's columns across
  sixteen separate column arrays would mean sixteen separate cache lines
  touched per comparison instead of one.

DiDi states this directly: **"Co-locating the columns of a row helps row
comparison during sorting, hashing, or joins (column access locality)"**
[5, slide 09]. The layout that makes this efficient needs two more
properties: rows of *fixed* size, and column offsets *known in advance*, so
that finding column `k` of row `r` on a page is pure arithmetic — no scanning,
no length-prefix chasing:

```
 ┌ page ──────────────────────┐    row #r is located
 │┊col0┊col1┊col2┊  ...  ┊    │      at offset r × w
 │┊col0┊col1┊col2┊  ...  ┊    │      within the page
 │┊col0┊col1┊col2┊  ...  ┊    │
 └──────── w bytes ───────────┘
              ↑    ↑    ↑
           column offsets within a row
     (row width w and offsets are known at
      query-plan generation time — computed
      ONCE per TupleDataLayout, not per row)
```

This is exactly `TupleDataLayout`, the real DuckDB class backing every
`GroupedAggregateHashTable`, sort buffer, and hash-join build side:

```cpp
// src/include/duckdb/common/types/row/tuple_data_layout.hpp:72-104 (excerpted)
//! Returns the total width required for each row, including padding
inline idx_t GetRowWidth() const {
	return row_width;
}
//! Returns the offset to the start of the data
inline idx_t GetDataOffset() const {
	return flag_width;
}
//! Returns the offset to the start of the aggregates
inline idx_t GetAggrOffset() const {
	return flag_width + data_width;
}
//! Returns the column offsets into each row
inline const vector<idx_t> &GetOffsets() const {
	return offsets;
}
```

`row_width`, `flag_width` (the validity/null bitmap, stored up front),
`data_width`, `aggr_width`, and the per-column `offsets` vector are all
computed *once*, when the layout is built from the operator's known types at
plan-generation time — precisely DiDi's "row size and column offsets are known
at query plan generation time: store these only once per layout" [5, slide 09]. For `GROUP BY o.custkey; SUM(l.extendedprice)`, that layout has (in
order): a validity bitmap, the group column `custkey` at a fixed offset, and
then — starting at `GetAggrOffset()` — the running `SUM` accumulator for that
group, at another fixed offset. Every group's row on every page has the exact
same shape, which is what lets the hash table's inner probing loop (§14.6)
jump straight to a candidate row's group-key bytes with one multiply and one
add, no matter how many rows are already in the table.

## 14.5 German strings: fixed rows meet variable-length data

Fixed-width rows are easy to arrange for `custkey` (an integer) but not for
`text`, `blob`, or nested types, whose length varies row to row. DuckDB's
answer — the same one popularized by Umbra (TU Munich) [14] and, per DiDi's
footnote, now used by Apache Arrow and Polars too — is to keep the *row* fixed
size regardless, and push variable-length payloads out to separate pages,
referenced from the row by a small, fixed-size header [5, slide 10]:

```
 short string (≤ 12 bytes): stored entirely inline, no pointer at all
 ┌──────────┬────────────────────────┐
 │ len (4B) │   up to 12 raw bytes    │
 └──────────┴────────────────────────┘
      20 bytes total, no indirection

 long string (> 12 bytes): 4-byte prefix + pointer to a separate page
 ┌──────────┬──────────┬─────────────┐
 │ len (4B) │ prefix(4B)│ pointer (8B)│──────▶  full string bytes,
 └──────────┴──────────┴─────────────┘          on a separate
      16 bytes total                            "variable data page"
```

This is not a paraphrase — it is exactly `string_t`, the struct every VARCHAR
value in DuckDB's execution engine is represented by, right down to the
byte counts:

```cpp
// src/include/duckdb/common/types/string_type.hpp
struct string_t {
public:
	static constexpr idx_t PREFIX_BYTES = 4 * sizeof(char);
	static constexpr idx_t INLINE_BYTES = 12 * sizeof(char);
	static constexpr idx_t HEADER_SIZE = sizeof(uint32_t) + PREFIX_BYTES;
	...
private:
	union {
		struct {
			uint32_t length;
			char prefix[4];
			char *ptr;
		} pointer;
		struct {
			uint32_t length;
			char inlined[12];
		} inlined;
	} value;
};
```

`IsInlined()` is simply `GetSize() <= INLINE_LENGTH` (12 bytes) — the decision
of which union member is "live" is made purely from the length that's already
sitting in the first 4 bytes, with no separate tag bit needed. This design
wins for two independent reasons, both of which matter for a real OLAP
workload:

1. **Most real strings are short.** Country codes, single-word categorical
   columns, short names, order statuses (`'O'`/`'F'`) — the overwhelming
   majority of string columns queried in practice fit in 12 bytes. For all of
   those, `string_t` needs *zero* pointer chases to read the value: the bytes
   are sitting right there in the row, in the same cache line the row's other
   columns already occupied. Compare this to a naive design where every
   string is a pointer to a heap allocation — every single string comparison
   or copy would cost a cache-missing dereference, even for `'F'`.
2. **Even for long strings, many operations never need to dereference the
   pointer.** The 4-byte prefix is stored redundantly — once inline for short
   strings, once *alongside the pointer* for long ones — specifically so that
   equality and ordering comparisons can often be decided from the header
   alone. The real comparison code makes this explicit:

```cpp
// src/include/duckdb/common/types/string_type.hpp
static inline bool Equals(const string_t &a, const string_t &b) {
	uint64_t a_bulk_comp = Load<uint64_t>(const_data_ptr_cast(&a));
	uint64_t b_bulk_comp = Load<uint64_t>(const_data_ptr_cast(&b));
	if (a_bulk_comp != b_bulk_comp) {
		// Either length or prefix are different -> not equal
		return false;
	}
	// they have the same length and same prefix!
	a_bulk_comp = Load<uint64_t>(const_data_ptr_cast(&a) + 8u);
	b_bulk_comp = Load<uint64_t>(const_data_ptr_cast(&b) + 8u);
	if (a_bulk_comp == b_bulk_comp) {
		// either they are both inlined (so compare equal) or point to the same string
		return true;
	}
	if (!a.IsInlined()) {
		// 'long' strings of the same length -> compare pointed value
		if (memcmp(a.value.pointer.ptr, b.value.pointer.ptr, a.GetSize()) == 0) {
			return true;
		}
	}
	return false;
}
```

The first 8-byte load already covers `length` + `prefix` together as one
machine word; a mismatch there — different lengths, or the same length but
different first four bytes — proves inequality with **one comparison and no
pointer dereference at all**. Only when the header genuinely can't decide the
answer does the code fall through to `memcmp` on the pointed-to bytes. The
ordering comparison (`GreaterThan`) does the same thing for `<`/`>`: the
4-byte prefix, byte-swapped into a comparable integer via
`GetPrefixIntegerComparable()`, resolves most real-world comparisons (sorting
distinct customer names, for instance) without ever following the pointer —
exactly the same principle Chapter 2 §2.3.5 built up from scratch for
byte-scanning (do the cheap, coarse check first; only pay for the expensive,
precise check when the cheap one is inconclusive).

### Spilling variable-data pages: the rebasing problem

Row pages holding `string_t` headers are ordinary paged intermediates
(§14.2's category 2) and can be spilled. But a *long* string's pointer is an
absolute address into a *separate* variable-data page — and if that page gets
spilled and later reloaded, nothing guarantees it comes back at the same
address. Every pointer stored in every row that referenced the old address is
now dangling. DiDi describes the fix as: store the page's base address as
metadata at the head of the page itself; when the page is reloaded at a new
base address, walk every row that references it and rewrite the pointer,
lazily, on next dereference [5, slide 11]. This is not a simplification
— the real allocator that owns row and heap pages does exactly this, and says
so in its own comment:

```cpp
// src/common/types/row/tuple_data_allocator.cpp:441-460
// Check if heap block has changed - re-compute the pointers within each row if so. Independent scans may read
// the same chunk concurrently, so the comparison and the update of the stored pointer are both done under the
// part lock, and this scan uses the pointer it pinned rather than re-reading the stored one
data_ptr_t base_heap_ptr;
if (recompute && pin_state.properties != TupleDataPinProperties::ALREADY_PINNED) {
	base_heap_ptr = GetBaseHeapPointer(pin_state, part);
	lock_guard<mutex> guard(part.lock);
	const auto old_base_heap_ptr = part.base_heap_ptr;
	if (old_base_heap_ptr != base_heap_ptr) {
		// ... RecomputeHeapPointers walks every affected row and rewrites its pointer
		// relative to (base_heap_ptr - old_base_heap_ptr)
		part.base_heap_ptr = base_heap_ptr;
	}
} else {
	base_heap_ptr = part.base_heap_ptr;
}
```

Two scans can read the same chunk at the same time. So the compare and the
update both happen under the lock, and each scan keeps the pointer it pinned
itself (`base_heap_ptr`) instead of reading the shared field again later. One
scan cannot see a half-updated base address from the other.

`part.base_heap_ptr` is exactly the metadata DiDi's slide describes — the base
address recorded once per page/part — and this check runs inside
`InitializeChunkStateInternal`, which executes whenever a chunk of rows is
about to be scanned or otherwise accessed. That is the "lazily, on next
dereference" part: nothing eagerly walks every spilled page the moment it's
paged back in; the fix-up happens exactly once per page, the next time
something actually needs to read from it, and every affected row's pointer is
shifted by the same `new_base - old_base` delta in one pass. The mechanism
`RecomputeHeapPointers` performs is the general form of exactly this delta
shift.

```
  before spill:                        after spill + reload at new address:

  base = A                              base = B  (B ≠ A, chosen by the
  ┌ variable data page ┐                ┌ variable data page ┐   allocator when
  │ "customer one" ...  │                │ "customer one" ...  │   reloading — not
  └─────────────────────┘                └─────────────────────┘   guaranteed == A)

  row's pointer: A + Δ                  on next dereference:
                                         new pointer = B + Δ
                                         (Δ = old_ptr - old_base,
                                          recomputed once, in bulk,
                                          for every row on the page)
```

## 14.6 Hash-based grouped aggregation, Phase 1: thread-local pre-aggregation

`GROUP BY` is the canonical pipeline-breaking sink this chapter has been
building toward, and DuckDB executes it — by default — through a physical
operator named `HASH_GROUP_BY` (`PhysicalHashAggregate` in the plan-generator
vocabulary Chapter 9 introduces), backed by a class called
`GroupedAggregateHashTable`. DiDi's own running example for this section
scales the problem up past what fits comfortably in memory [5, slide 12]:

```sql
SELECT l_orderkey, count(*)
FROM   lineitem             -- TPC-H (sf = 100): 600,000,000 rows
GROUP BY l_orderkey         -- creates 150,000,000(!) groups
```

This book's own running Query A has the same shape at smaller scale:

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

`GROUP BY o.custkey; SUM(l.extendedprice)` is used as the concrete running
example for the rest of this section: one integer group key, one running-sum
aggregate state per group, fed by the `HASH_JOIN`'s output.

DuckDB's design goals for this operator are stated explicitly [5, slide 13]: use every CPU core, and don't assume the hash table's entries fit
in memory — treat them as a paged intermediate and let the memory manager
spill them like anything else. The algorithm that achieves both is split into
two phases: **Phase 1, thread-local pre-aggregation**, one hash table per
morsel-processing thread; and **Phase 2, partition-wise aggregation**, one
thread per output partition, covered in §14.7.

### The hash-splitting scheme

Each thread reads an input **morsel** — DiDi's term for roughly 100,000 rows
handed to one thread at a time (there are typically many more morsels than
threads, so an idle thread just grabs the next one; this is the same
load-balancing idea Chapter 2 §2.3.6 built by hand with `pthread`s, and the
same "morsel" vocabulary Chapter 11 formalizes for pipeline scheduling).
Within a thread, every input row's group key is hashed to a 64-bit value, and
that hash is split into three fields [5, slides 14–15]:

```
  hash(key) = h              64 bits total
              ┌──────────────┬─────────────┬──────────────────┐
              │  salt        │  partition # │   array index     │
              │  16 bits      │  ≤ 12 bits   │  (remaining bits)  │
              │  (upper)      │  (radix bits) │  (lower)           │
              └──────────────┴─────────────┴──────────────────┘
```

This is precisely what the real bit layout enforces, down to the comment
explaining *why* the salt gets the top 16 bits specifically:

```cpp
// src/include/duckdb/common/radix_partitioning.hpp:25-47 (excerpted)
//! 4096 partitions ought to be enough to go out-of-core properly
static constexpr const idx_t MAX_RADIX_BITS = 12;

//! Radix bits begin after uint16_t because these bits are used as salt in the aggregate HT
static inline constexpr idx_t Shift(idx_t radix_bits) {
	return (sizeof(hash_t) - sizeof(uint16_t)) * 8 - radix_bits;
}
```

`MAX_RADIX_BITS = 12` is exactly DiDi's "≤ 12 bits" partition field, and the
comment confirms the 16-bit salt occupies the top of the word by design, with
up to 12 radix (partition) bits taken from immediately below it. The
remaining low bits of the hash select a slot in the thread-local **array**
(the hash table's pointer table): capacity starts at `2 × STANDARD_VECTOR_SIZE`
(4096 slots) and grows by doubling as the table fills, so the exact number of
"array index" bits used varies with how full the table has gotten — DiDi's
worked example (17 bits, 131,072 slots) is one concrete size along that
growth curve, not a fixed constant. Salt and pointer are packed together into
one 64-bit `ht_entry_t`, not stored as two separate fields:

```cpp
// src/include/duckdb/execution/ht_entry.hpp:34-36
//! Upper 16 bits are salt, lower 48 bits are the pointer
static constexpr const hash_t SALT_MASK = 0xFFFF000000000000;
static constexpr const hash_t POINTER_MASK = 0x0000FFFFFFFFFFFF;
```

### Why the salt matters: rejecting a non-match without a pointer chase

The array slot found via the low bits of the hash might already be occupied
by an *unrelated* key that happened to hash to the same slot — a collision.
The naive way to check "is this really my key" is to follow the slot's
pointer out to the row on the data page and compare the full group key
there — but that pointer very likely lands on a cache line nothing nearby has
touched, so a wrong guess costs a full cache miss for nothing. The salt exists
to make the *common* case (this slot's occupant is not my key) cheap: compare
the candidate's 16-bit salt against the slot's stored salt *first*, entirely
within the already-cache-resident array, before ever following the pointer.
Two different keys landing in the same array slot is far more likely than
those same two keys *also* sharing the same 16-bit salt, so a salt mismatch
lets the probe move on immediately — no cache miss paid for a comparison that
was going to fail anyway. This is exactly the same idea as Chapter 2 §2.3.5's
prefix-based `HAS_ZERO`/`HAS_NL` trick and §14.5's string-prefix comparison: do
the cheap, coarse check first, and only pay for the expensive, precise
comparison when the cheap one couldn't resolve the question. The real inner
loop makes the ordering explicit — salt compare, then (only on a salt match)
follow the pointer and compare full keys, and only on an actual salt
collision without a probing conflict does it fall through to linear probing:

```cpp
// src/execution/aggregate_hashtable.cpp:795-831  (GroupedAggregateHashTableInnerLoop, elided)
template <bool HAS_SEL, bool CREATE_MISSING>
static void GroupedAggregateHashTableInnerLoop(ht_entry_t *const entries, const idx_t capacity, const hash_t bitmask,
                                               const hash_t *const hash_salts, uint64_t *const ht_offsets,
                                               const SelectionVector *const sel_vector, const idx_t remaining_entries,
                                               SelectionVector &empty_vector, SelectionVector &compare_vector,
                                               idx_t &empty_count, idx_t &compare_count) {
	for (idx_t i = 0; i < remaining_entries; i++) {
		const auto index = HAS_SEL ? sel_vector->get_index_unsafe(i) : i;
		const auto salt = hash_salts[index];
		auto &ht_offset = ht_offsets[index];

		idx_t inner_iteration_count;
		for (inner_iteration_count = 0; inner_iteration_count < capacity; inner_iteration_count++) {
			auto &entry = entries[ht_offset];
			if (!entry.IsOccupied()) {
				if (CREATE_MISSING) {
					entry.SetSalt(salt);
				}
				empty_vector.set_index(empty_count++, index);
				break;
			}
			if (DUCKDB_LIKELY(entry.GetSalt() == salt)) { // Matching salt: compare groups
				compare_vector.set_index(compare_count++, index);
				break;
			}
			// Linear probing
			SaltIncrementAndWrap(ht_offset, salt, bitmask);
		}
	}
}
```

Only rows whose salt matched get pushed into `compare_vector` for the
(relatively expensive) full group-key comparison against the row on the data
page; every other row was resolved — occupied-by-someone-else, keep probing,
or found an empty slot — using only bytes already sitting in the array.
**Linear probing** (`SaltIncrementAndWrap`) resolves genuine collisions by
walking forward through the array, using bits of the salt itself to decide
the step size rather than always stepping by one — spreading collision chains
out instead of clustering them.

### The ⅔-full spill trigger, and why resetting is safe

DiDi frames the spill decision this way [5, slide 15]: **"When hash
slots are ⅔ full and collisions become frequent: (1) unpin data pages — the
memory manager may evict these; (2) empty the hash array, continue (reset OK
because of Phase 2)."** The ⅔ figure is not an approximation — it is the exact
inverse of the hash table's load factor:

```cpp
// src/include/duckdb/execution/aggregate_hashtable.hpp:87
constexpr static double LOAD_FACTOR = 1.5;
```

`ResizeThreshold(capacity) = capacity / LOAD_FACTOR = capacity / 1.5`, which
is `capacity × 0.667` — DiDi's "⅔ full," exactly. When a thread's local hash
table reaches this threshold, it has three choices: grow the pointer
array, keep it but stop looking rows up, or **abandon** its current array and
row pages. The code that chooses is the end of `RadixPartitionedHashTable::Sink`:

```cpp
// src/execution/radix_partitioned_hashtable.cpp:648-674  (RadixPartitionedHashTable::Sink, excerpted)
	// Row width and aggregate state size can increase without filling the pointer table.
	const auto total_size = ht.GetSizeInBytes();
	UpdateSinkReservation(context.client, gstate, ht, total_size);
	const auto memory_pressure =
	    total_size > gstate.GetThreadLimit() || (gstate.spill_plan && gstate.StatePressureExceeded(ht));
	if (!memory_pressure &&
	    ht.Count() + STANDARD_VECTOR_SIZE < GroupedAggregateHashTable::ResizeThreshold(lstate.local_sink_capacity)) {
		return;
	}

	if (!memory_pressure) {
		if (RadixHTAdaptivity::TryGrow(gstate, lstate)) {
			return;
		}
		RadixHTAdaptivity::MaybeSkipLookups(gstate, lstate);
	}

	if (gstate.number_of_threads > RadixHTConfig::GROW_STRATEGY_THREAD_THRESHOLD || gstate.external) {
		// Keep the materialized rows and clear only the local pointer table
		// We don't do this when running with 1 or 2 threads, it only makes sense when there's many threads
		gstate.any_abandoned = true;
		ht.Abandon();
		if (!memory_pressure) {
			RadixHTAdaptivity::MaybeResumeLookups(gstate, lstate);
		}
	}
```

Read it top to bottom:

1. **Memory pressure** is checked on every chunk, not only when the
   array is ⅔ full. `GetThreadLimit()` is this thread's share of the
   operator's memory reservation: the reservation divided by the number of
   threads, times 0.8 (`src/include/duckdb/execution/radix_ht_sink_state.hpp:100-102`).
   If the table's total size is above it, the thread abandons at once, even
   if the array is not full. Wide rows or big aggregate states can use a lot
   of memory without filling the array; that is what the first comment
   warns about.
2. **No pressure, array ⅔ full: try to grow.** `RadixHTAdaptivity::TryGrow`
   (`src/execution/radix_ht_adaptivity.cpp:9-78`) only runs with more than
   two threads. It uses a HyperLogLog sketch (a small structure that
   estimates how many *distinct* keys it has seen) to guess how many groups
   the table really holds. If many input rows were stored as *duplicate*
   group rows (because earlier abandons forgot that the group already
   existed), a bigger array would have saved that work. The rule is: grow
   only if the bytes of those duplicate rows are more than 4 times the size
   of the new array (`GROWTH_DUPLICATE_BYTES_FACTOR = 4.0`). Growth must
   also fit the thread's memory limit; if it does not, the new size is
   halved until it does, or growth is skipped. If the allocation still
   fails with an out-of-memory error, the thread turns HLL off and does not
   try again.
3. **No pressure, growth not worth it: maybe skip lookups.**
   `MaybeSkipLookups` looks at the cycle that just ended. If fewer than 5%
   of the input rows found an existing group (`MINIMUM_SMALL_TABLE_REDUCTION
   = 0.05`), the lookups are wasted work: nearly every key is new. Then the
   table *skips lookups*: it just appends each row as a new group row and
   leaves all merging to Phase 2. This is only tried after at least
   `LOOKUP_SAMPLE_SIZE` (1,048,576) input rows.
4. **Otherwise abandon.** After abandoning, `MaybeResumeLookups`
   turns lookups back on once another 1,048,576 rows went by with lookups
   skipped, so the table can notice if the data changed.

`test/sql/aggregate/group/test_group_by_skip_lookups.test` runs a query where
nearly every group is unique (1,100,000 groups for 1,200,000 rows, with three
threads), and checks that the result is still correct.
`test/sql/aggregate/external/budgeted_local_growth.test_slow` checks results
when local tables grow or fall back to external aggregation under
`memory_limit='512MB'`.

So with many threads, abandoning at ⅔ full is the fallback, not the only
choice. The policy for growing and skipping lookups lives in its own class,
`RadixHTAdaptivity` (`src/include/duckdb/execution/radix_ht_adaptivity.hpp`).

`Abandon()` is DiDi's "unpin and reset," verbatim:

```cpp
// src/execution/aggregate_hashtable.cpp:166-184
void GroupedAggregateHashTable::Abandon() {
	if (radix_bits >= UNPARTITIONED_RADIX_BITS_THRESHOLD) {
		// Flush/unpin unpartitioned data and append to partitioned data
		if (unpartitioned_data) {
			unpartitioned_data->FlushAppendState(state.unpartitioned_append_state);
			unpartitioned_data->Unpin();                       // <- "unpin data pages"
			unpartitioned_data->Repartition(context, *partitioned_data);
		}
		InitializeUnpartitionedData();
	}

	// Start over
	ClearPointerTable();                                        // <- "empty hash array"
	count = 0;
	adaptivity.BeginCycle();
	// ...
}
```

`adaptivity.BeginCycle()` starts a fresh "cycle" for the
statistics that `MaybeSkipLookups` reads at the end of the next cycle.

Resetting is safe — losing all trace of which rows this thread had already
aggregated together so far — precisely because **Phase 2 re-aggregates
everything from scratch anyway**. A thread's Phase-1 hash table was never
the *final* answer for any group; it was only ever a way to cut down on
volume before Phase 2 does the authoritative merge. Two partial rows for
`custkey = 42` sitting in two different abandoned batches, still tagged with
the same partition number, will both get picked up and correctly summed
together in Phase 2 (§14.7) — nothing about correctness depended on Phase 1
having produced one row per group per thread; it only needed to produce
*some* reduction in row count, which even repeatedly abandoning and restarting
still achieves over the morsel stream as a whole.

## 14.7 Hash-based grouped aggregation, Phase 2: partition-wise aggregation

Every row a Phase-1 thread appends to a data page carries the partition
number computed from its hash's middle bits (§14.6). Phase 2's job is to take
all rows tagged with the same partition number — scattered across however
many thread-local batches Phase 1 produced — and merge them into one
authoritative per-partition hash table, in parallel across partitions
[5, slide 16]:

```
  thread-local partitions       exchange        thread-local HTs        output
  (from many Phase-1 threads)                   (one per partition,
                                                  one thread each)
  #0 [ data page ] ┐
  #1 [ data page ] ┼──╲                                                ┌───┐
  #0 [ data page ] ┼───╲──▶  merge rows      ┌───────────┐    scan     │███│
  #1 [ data page ] ┼───╱     tagged #0  ───▶ │ fresh HT  │───────────▶ │███│
   ⋮               ┘  ╱      into one HT     │ (partition │  as morsel │███│
                     ╱                        │    #0)     │  downstream└───┘
                                              └───────────┘
```

DiDi's instruction to the reader is exactly the "one thread per partition"
scheduling rule, together with the sizing rule that decides *how many*
partitions to create: **"Run one thread per partition. Choose bits for the
partition number such that the hash table for a fully aggregated partition
fits memory (create more but smaller partitions [if needed])"** [5, slide 16]. This is a recursive sizing decision, not a one-shot choice made at
the start of the query: DuckDB starts with a small number of radix bits and
*grows* them — meaning it doubles the number of partitions, and therefore
halves each partition's expected size — only if the current partitioning
isn't proving sufficient. The real code names this growth step explicitly:

```cpp
// src/include/duckdb/execution/radix_ht_sink_state.hpp:82
static constexpr idx_t REPARTITION_RADIX_BITS = 2;

// src/execution/radix_partitioned_hashtable.cpp:606-612  (MaybeRepartition)
const auto row_size_per_partition =
    ht.GetMaterializedCount() * ht.GetPartitionedData().GetLayout().GetRowWidth() / partition_count;
if (row_size_per_partition >
    LossyNumericCast<idx_t>(config.BLOCK_FILL_FACTOR * static_cast<double>(gstate.block_alloc_size))) {
	// We crossed our block filling threshold, try to increment radix bits
	config.SetRadixBits(current_radix_bits + config.REPARTITION_RADIX_BITS);
}
```

and `RadixPartitionedHashTable::MaxThreads` makes the "fits memory" sizing
constraint from DiDi's slide concrete: it estimates how many partitions'
worth of memory the current reservation can afford
(`usable_memory / max_partition_size`), and only schedules that many threads
onto the finalize/scan phase at once, rather than assuming every partition
can run concurrently regardless of size:

```cpp
// src/execution/radix_partitioned_hashtable.cpp:881-901
idx_t RadixPartitionedHashTable::MaxThreads(GlobalSinkState &sink_p) const {
	...
	// This many partitions will fit given our reservation (at least 1)
	const auto partitions_fit = MaxValue<idx_t>(usable_memory / sink.max_partition_size, 1);
	return MinValue<idx_t>(partitions_fit, max_threads);
}
```

When a thread is assigned a partition, it builds one fresh
`GroupedAggregateHashTable` and `Combine`s every thread-local batch tagged
with that partition number into it — this is the actual re-aggregation from
scratch that makes §14.6's "reset is safe" argument correct:

```cpp
// src/execution/radix_partitioned_hashtable.cpp:1109-1111  (RadixHTLocalSourceState::Finalize)
// Now combine the uncombined data using this thread's HT
ht->Combine(*partition.data, &partition.combined_chunks);
```

And the moment that merge finishes, the same thread immediately switches to
scanning its own freshly-finalized partition and streams the result out as a
new chunk to whatever operator sits downstream — no separate barrier, no
waiting for every other partition to finish first:

```cpp
// src/execution/radix_partitioned_hashtable.cpp:1153-1155
// This thread will scan the partition
task = RadixHTSourceTaskType::SCAN;
scan_status = RadixHTScanStatus::INIT;
```

This is exactly DiDi's closing line for Phase 2: **"When a partition
finalize ends, immediately scan its output data pages. These become a morsel
to be fed to the downstream plan"** [5, slide 16] — and it is also
where this chapter's vocabulary rejoins Chapter 11's: a fully aggregated
partition's output chunk is a `DataChunk`, produced by a `GetData` call, fed
directly into the next operator's pipeline exactly the way any other morsel
would be, with no special case for "this data used to be a hash table."

## 14.8 `PERFECT_HASH_GROUP_BY`: when hashing isn't needed at all

Everything in §14.6–14.7 exists to handle collisions safely and cheaply — the
salt, linear probing, the whole spill/re-aggregate cycle. All of that machinery
is unnecessary overhead if DuckDB can *prove, at optimization time*, that the
grouping key only ever takes on a small, known range of values: a dense array
indexed directly by the key's (small) integer value has no collisions to
resolve, by construction, so there is nothing for a salt or a probe sequence
to do. DiDi's footnote flags exactly this specialization: **"If DuckDB
statically knows that no collisions will occur (e.g., when the grouping key
has few distinct values like `lineitem.l_returnflag`), the efficient
alternative operator `PERFECT_HASH_GROUP_BY` is used"** [5, slide 12,
fn. 5].

The real eligibility check lives in the physical plan generator, and it's
worth reading in full because it shows exactly how conservative this
optimization has to be to stay safe:

```cpp
// src/execution/physical_plan/plan_aggregate.cpp:131-247  (CanUsePerfectHashAggregate, elided)
static bool CanUsePerfectHashAggregate(ClientContext &context, LogicalAggregate &op, vector<idx_t> &bits_per_group) {
	if (op.grouping_sets.size() > 1 || !op.grouping_functions.empty()) {
		return false;
	}
	idx_t perfect_hash_bits = 0;
	for (idx_t group_idx = 0; group_idx < op.groups.size(); group_idx++) {
		auto &group = op.groups[group_idx];
		auto &stats = op.group_stats[group_idx];

		switch (group->GetReturnType().InternalType()) {
		case PhysicalType::INT8:
		case PhysicalType::INT16:
		case PhysicalType::INT32:
		case PhysicalType::INT64:
		case PhysicalType::UINT8:
		case PhysicalType::UINT16:
		case PhysicalType::UINT32:
		case PhysicalType::UINT64:
			break;
		default:
			// we only support simple integer types for perfect hashing
			return false;
		}
		// ... (constructs [type_min, type_max] stats when the column has none but is a small type)
		if (!NumericStats::HasMinMax(nstats)) {
			return false;
		}
		// ... (computes `range` = max - min, as a hugeint_t, per integer width)
		// bail out on any range bigger than 2^32
		if (range >= NumericLimits<int32_t>::Maximum()) {
			return false;
		}
		range += 2;
		// figure out how many bits we need
		idx_t required_bits = RequiredBitsForValue(UnsafeNumericCast<uint32_t>(range));
		bits_per_group.push_back(required_bits);
		perfect_hash_bits += required_bits;
		// check if we have exceeded the bits for the hash
		if (perfect_hash_bits > Settings::Get<PerfectHtThresholdSetting>(context)) {
			// too many bits for perfect hash
			return false;
		}
	}
	for (auto &expression : op.expressions) {
		auto &aggregate = expression->Cast<BoundAggregateExpression>();
		if (aggregate.IsDistinct() || !aggregate.Function().HasStateCombineCallback()) {
			// distinct aggregates are not supported in perfect hash aggregates
			return false;
		}
	}
	return true;
}
```

This is exactly why Chapter 2 §2.6's live `EXPLAIN ANALYZE` trace of this
book's own Query A showed `PERFECT_HASH_GROUP_BY`, not `HASH_GROUP_BY`, for
`GROUP BY o.custkey`: the optimizer's statistics-propagation pass had already
proven a tight, known integer range for `custkey` (`Build Min: 0` /
`Build Max: 99999` on the join feeding it, in that chapter's synthetic data),
which is exactly the min/max/range check `CanUsePerfectHashAggregate` performs.
The `bits_per_group`/`required_bits` computed here become the dense array's
index width directly — no hash function, no salt, no linear probing, because
the array slot *is* the (offset) group value, so two different groups can
never collide on the same slot by definition. Where the range is unknown
(unbounded VARCHAR keys, an `int64` column with no useful statistics, or a
range that's simply too wide to allocate a dense array for), the check fails
and the operator falls back to ordinary `HASH_GROUP_BY` — this is a strictly
*better* operator exactly when it applies, and it is inapplicable precisely
when the grouping key's cardinality can't be bounded cheaply in advance.

### What if the statistics are wrong?

The whole plan rests on the min/max statistics. If a file's statistics are
corrupt — say they claim the key is between 300 and 400, but the data holds
600 — a group id falls outside the dense array. Writing to that slot would
write to memory the array does not own. So `PerfectAggregateHashTable` checks
every group id before it uses it:

```cpp
// src/execution/perfect_aggregate_hashtable.cpp:126-131
static inline void CheckGroupInRange(uint64_t group, idx_t total_groups) {
	if (group >= total_groups) {
		throw DataCorruptionException("Perfect hash aggregate: aggregate group %llu exceeded total groups %llu.", group,
		                              total_groups);
	}
}
```

There are two paths that turn group ids into addresses: the ordinary per-row
path and a *clustered* path (used when runs of equal keys can be updated
together). Both are checked. The clustered path checks each
clustered id with `CheckGroupInRange`. But the clusterer masks the ids, and
the per-row addresses still use the raw ids. So a second check ORs all raw
ids together: the array size is a power of two, so if any id is too large,
the OR is too large as well
(`src/execution/perfect_aggregate_hashtable.cpp:206-218`).
`test/sql/aggregate/group/perfect_hash_corrupt_stats_range.test` loads a
database whose statistics claim `[300, 400]` and expects:

```
Data Corruption Error: Perfect hash aggregate: aggregate group 301 exceeded total groups 128.
```

## 14.9 Hash-based vs. sort-based grouping, and `PhysicalPartitionedAggregate`

DiDi notes, as an aside, that grouped aggregation can also be implemented by
*sorting* on the grouping column and then merging adjacent equal keys — and
that this is "profitable if the query also contains an `ORDER BY
l_orderkey` clause," a case DB literature calls an **interesting order**:
grouping comes almost for free once the data is already ordered by the
grouping key for some *other* reason [5, slide 12, fn. 4].

It would be tempting to look for exactly this in DuckDB's own optimizer — an
"interesting order" pass that notices `ORDER BY custkey` upstream of
`GROUP BY custkey` and swaps in a sort-merge grouping strategy instead of
hashing. Searching the physical plan generator turns up no such thing: there
is no `SORT_GROUP_BY`, `MERGE_GROUP_BY`, or equivalent operator anywhere in
DuckDB's operator vocabulary, and `plan_aggregate.cpp` never consults an
`ORDER BY` clause when choosing between `PhysicalHashAggregate`,
`PhysicalPerfectHashAggregate`, or the `PhysicalUngroupedAggregate` fallback.
Honestly stated: **this book found no evidence DuckDB exploits an "interesting
order" coming from an explicit `ORDER BY` clause to avoid hashing** —
`HASH_GROUP_BY` (with the `PERFECT_HASH_GROUP_BY` specialization of §14.8) is,
as far as the source shows, the only grouping strategy DuckDB's optimizer
picks from for an arbitrary `GROUP BY`.

What the source *does* show is a related but distinct specialization:
`PhysicalPartitionedAggregate`, chosen not from an `ORDER BY` clause but from
the *scan*'s own reported partitioning:

```cpp
// src/execution/physical_plan/plan_aggregate.cpp:32-46
static bool CanUsePartitionedAggregate(ClientContext &context, LogicalAggregate &op,
                                       PhysicalOperator &child, vector<column_t> &partition_columns) {
	...
	return PhysicalPlanGenerator::HasSingleValuePartitions(context, op.groups, child, partition_columns);
}
```

`HasSingleValuePartitions` walks down through any `PROJECTION`/`FILTER`
operators sitting above the scan and asks the underlying table function
(e.g. a Hive-partitioned Parquet read, where each distinct partition-column
value lives in its own subdirectory) whether it can guarantee
`TablePartitionInfo::SINGLE_VALUE_PARTITIONS` — that consecutive rows coming
out of the scan all share one grouping-key value before the value changes and
never repeats. When that guarantee holds, DuckDB can group with a
`PhysicalPartitionedAggregate` that never needs a shared hash table across
threads at all, because the source itself already delivered rows
pre-clustered by group. This *is* an "interesting order" argument in spirit
— structure already present in the input is exploited to skip hashing
entirely — but the interesting property comes from **the table function's
physical partitioning of the data on disk**, not from an `ORDER BY` clause in
the query. Chapter 1 §1.3's shorthand — `PhysicalHashAggregate` "or
`PhysicalPartitionedAggregate` under parallel execution" — is worth refining
with this in hand: the deciding factor isn't parallelism itself, it's whether
`HasSingleValuePartitions` can prove the scan already hands out one
partition-value's rows at a time; Query A's synthetic `orders`/`lineitem`
tables carry no such guarantee, so `GROUP BY o.custkey` never takes this path
in this book's worked example.

### Ordered and `DISTINCT` aggregates: two subtle cases

Query A's `SUM` is the simple case. Two kinds of aggregate need more than a
running value per group: an *ordered* aggregate, such as
`string_agg(x, ',' ORDER BY y)`, and a `DISTINCT` aggregate, such as
`count(DISTINCT x)`. Each has one subtle case that DuckDB must handle with care.

**Sharing a `DISTINCT` branch.** The optimizer pass
`MultiStageAggregateRewriter` (`src/optimizer/multi_stage_aggregate_rewriter.cpp`)
rewrites `DISTINCT` aggregates into several stages. First the input is made
distinct, then the aggregate runs over it. Several `DISTINCT` aggregates over
the same argument can share one de-duplicated branch. But an `ORDER BY`
expression that is not an argument becomes an extra de-duplication key. For
example, in `STRING_AGG(DISTINCT u, ',' ORDER BY u)` over a `UNION` column,
the values `'1'` (text) and `1` (integer) are both `1` as `VARCHAR`, but
`ORDER BY u` still tells them apart. Putting that aggregate into a shared
branch would change which rows the *other* aggregates in the branch see as
duplicates. So two aggregates share a branch only when those extra `ORDER BY`
keys match:

```cpp
// src/optimizer/multi_stage_aggregate_rewriter.cpp:767-769
			if (ArgumentsEqualIncludingCollation(aggregate, other) && Expression::ListEquals(keys, set.keys) &&
			    Expression::Equals(aggregate.GetFilter(), other.GetFilter()) &&
			    OrderExpressionsEqual(order_expressions, set.order_expressions)) {
```

`test/sql/aggregate/distinct/rewrite/distinct_aggregate_rewrite_order_by_keys.test`
runs `STRING_AGG(DISTINCT u, ',')` and `STRING_AGG(DISTINCT u, ',' ORDER BY u)`
side by side. For group 1 the first must give `[1]` (one distinct text
value) and the second `1,1` (two distinct union values).

**Finalizing many ordered-aggregate states.** An ordered aggregate buffers
its input per group, then sorts and finalizes. The internal sort can only
number up to 2^16 groups in one call. One call may have to finalize more
states than that (for example a window function where one hash group holds
more than 65,535 partitions). Then the group numbers would overflow and some
results would come out wrong or `NULL`. So `SortedAggregateFunction::Finalize`
cuts the work into batches of at most 65,535 states:

```cpp
// src/function/aggregate/sorted_aggregate_function.cpp:546-559
	static void Finalize(Vector &states, AggregateFinalizeInputData &finalize_input_data, Vector &result, idx_t count,
	                     const idx_t offset) {
		//	Internally we can only handle 2^16 rows, so chunk them up
		const idx_t group_limit = NumericLimits<uint16_t>::Maximum();
		if (count <= group_limit) {
			FinalizeGroups(states, finalize_input_data, result, count, offset);
			return;
		}
		for (idx_t group_offset = 0; group_offset < count; group_offset += group_limit) {
			const auto group_count = MinValue<idx_t>(group_limit, count - group_offset);
			Vector group_states(states, group_offset, group_offset + group_count);
			FinalizeGroups(group_states, finalize_input_data, result, group_count, offset + group_offset);
		}
	}
```

Each batch gets its own slice of `states` and writes its results at
`offset + group_offset`. `test/sql/window/test_window_constant_aggregate.test_slow`
covers this, including a check that counts `NULL` results.

## 14.10 Query A end to end: `GROUP BY o.custkey` through Sink → Combine → Finalize → GetData

Putting every mechanism in this chapter together against this book's own
running query:

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

```
PhysicalTopN  (ORDER BY total DESC LIMIT 10)
  └─ PhysicalHashAggregate  (or PhysicalPerfectHashAggregate — §14.8 — when
     │                        o.custkey's range is statically known and small,
     │                        as Chapter 2 §2.6's live EXPLAIN ANALYZE showed)
     └─ PhysicalHashJoin
          ├─ PhysicalTableScan (orders)
          └─ PhysicalTableScan (lineitem, filter pushed into scan)
```

- **Sink** (Phase 1, §14.6). Each worker thread pulls a ~100,000-row morsel of
  join output — `(custkey, extendedprice)` pairs — and calls
  `RadixPartitionedHashTable::Sink`, which owns a thread-local
  `GroupedAggregateHashTable`. Every row's `custkey` is hashed; the low bits
  pick an array slot, the salt short-circuits non-matches, and either an
  existing group's running `SUM` is updated in place or a fresh row (group
  key + zero-initialized accumulator) is appended to a data page tagged with
  that hash's partition number. If this thread's local table crosses the
  ⅔-full `ResizeThreshold`, its data pages are unpinned and its pointer array
  is cleared (`Abandon()`) — safe, because nothing downstream yet trusts any
  thread's Phase-1 state to be a finished per-group total. (With
  more than two threads and no memory pressure, the thread may first try to
  grow its pointer array, or may stop doing lookups when nearly every
  `custkey` it sees is new — §14.6.)
- **Combine** (transition into Phase 2, §14.7). Once every thread has
  finished sinking its share of the join's output, `Combine` gathers up every
  thread-local, still-unmerged batch of rows, grouped by the partition number
  each row was tagged with in Phase 1 — no aggregation happens yet here, only
  the bookkeeping that makes each partition's full set of contributing rows
  visible to whichever single thread will finalize it.
- **Finalize** (Phase 2 proper). One thread per partition builds a fresh
  `GroupedAggregateHashTable`, sized so this specific partition's data is
  expected to fit in memory (recursively repartitioned into more, smaller
  partitions beforehand if a first estimate wouldn't fit — §14.7), and
  `Combine`s every row tagged with that partition into it. This is the
  authoritative re-aggregation: every partial `SUM` contributed by every
  thread for a given `custkey`, wherever it came from, is finally merged into
  one row per `custkey`.
- **GetData** (streamed to `PhysicalTopN`). The instant a partition finishes
  finalizing, the same thread switches to scanning it and emits its rows as
  ordinary `DataChunk`s — a fresh morsel from this operator's point of view —
  directly into `PhysicalTopN`, which only ever needs to retain the current
  best 10 rows by `total`, never materializing the full grouped result. Other
  partitions finalize and stream out on other threads concurrently; there is
  no single global barrier where "all groups are ready" has to hold before
  any row can leave `PhysicalHashAggregate`.

This is precisely the Sink/Combine/Finalize/GetData shape Chapter 11
formalizes for every sink operator in the engine, and precisely the
distinction Chapter 9 draws between `PhysicalHashAggregate` (the general case
this section just traced) and `PhysicalPartitionedAggregate` (§14.9's
specialization for a source that already guarantees one group's rows arrive
contiguously, which Query A's synthetic tables don't qualify for) — or
`PhysicalPerfectHashAggregate` (§14.8) when `o.custkey`'s range is small and
known, which is the operator Chapter 2 §2.6 actually observed for this exact
query against its synthetic dataset.

## 14.11 Where this goes next

This chapter answered the question Chapter 2 raised and deferred: once "keep
hot data in RAM, spill the rest" is stated as policy, what does DuckDB
actually build to make that true? The answer turned out to be layered:

- A single memory pool, sized by `memory_limit` (80% of host RAM by default),
  split into three categories whose spill/eviction rules differ because *what
  can safely be relocated* differs — non-paged data can't be, paged
  intermediates can be (because access always goes through a level of
  indirection), and buffered base-table pages don't even need write-back
  (because they were never the authoritative copy in the first place).
- Two different responses to "more data than fits": streaming (for anything
  that can process one chunk at a time and never look back) and spilling (for
  sinks — `GROUP BY`, `ORDER BY`, joins, `count(DISTINCT)`, sweeping-frame
  window functions — that must see the whole input before producing any
  output).
- A row-major layout for paged intermediates specifically, because sorting,
  hashing, and joining all compare *whole rows* at a time, in contrast to
  Chapter 17's column-major base-table storage, which is read a few columns
  at a time.
- A German-strings-style encoding (`string_t`) that avoids a pointer chase for
  the common case (short strings) and avoids one even for many operations on
  long strings (prefix-only comparisons), plus a lazy pointer-rebasing scheme
  for the rare case where a spilled variable-data page comes back at a new
  address.
- A two-phase, fully parallel, spill-aware algorithm for `HASH_GROUP_BY`
  itself — thread-local pre-aggregation with a salted, linearly-probed hash
  table that's free to reset under memory pressure, followed by
  partition-wise re-aggregation that streams each finished partition
  downstream the moment it's ready — plus two narrower specializations
  (`PERFECT_HASH_GROUP_BY` for statically-bounded keys,
  `PhysicalPartitionedAggregate` for already-partitioned sources) that skip
  parts of this machinery entirely when it's provably safe to do so. Ordered
  and `DISTINCT` aggregates add their own care on top (§14.9).

Chapter 9 (§9.3.1) picks up the physical-plan-generation side of this — exactly which
statistics and shape checks decide between `PhysicalHashAggregate`,
`PhysicalPerfectHashAggregate`, and `PhysicalPartitionedAggregate` for a given
`LogicalAggregate`. Chapter 11 generalizes the Sink/Combine/Finalize/GetData
lifecycle traced in §14.10 to every other sink operator in the engine —
`ORDER BY`, hash joins, `count(DISTINCT)` — each of which spills using the
same paged-intermediate machinery this chapter described, just with a
different row layout and a different merge step in place of Phase 2's
partition-wise re-aggregation. Chapters 16–18 pick up the other side of
§14.2's three-category split: how buffered base-table pages actually get
written, checkpointed, and recovered — the durability story this chapter
deliberately set aside to focus on pure in-memory-vs-spilled behavior.
