# Chapter 15 — Sorting Large Tables

## 15.1 Sorting is not an `ORDER BY` implementation detail

It is tempting to file "sorting" under `ORDER BY` and move on — DuckDB's own
storage model reinforces the temptation, since a table is a bag of row groups
with no promised row order at all (Chapter 14). But by the time a query plan
reaches execution, sorting turns out to be one of the most cross-cutting pieces
of infrastructure in the whole engine. At least five distinct SQL features
route through the same sort machinery this chapter describes:

1. **`ORDER BY` itself** — the obvious case: `SELECT * FROM t ORDER BY a, b`
   needs the output rows in a specific total order.
2. **Ordered aggregates** — `list(x ORDER BY y)`, `string_agg(x, ',' ORDER BY
   y)`, and friends need their inputs sorted *before* the aggregate function
   ever sees them, because the aggregate's result depends on the order values
   arrive in, not just which values arrive.
3. **Window functions** — `row_number() OVER (ORDER BY x)`,
   `SUM(x) OVER (PARTITION BY p ORDER BY x)`, and every other window function
   with an `ORDER BY` clause need their frame defined over a specific row
   order per partition; DuckDB computes this by sorting each partition (or the
   whole input, if there's no `PARTITION BY`) before evaluating the window
   function over it.
4. **`ASOF JOIN`** — an as-of join ("find the most recent `orders` row at or
   before each `quotes` row's timestamp") is defined entirely in terms of
   comparing rows along a sort order; DuckDB's `ASOF JOIN` physical operator
   sorts (or merge-joins against an already-sorted side) using this same
   machinery.
5. **Deterministic `LIMIT`/`OFFSET` behavior** — without an `ORDER BY`, SQL
   makes no promises about which rows a `LIMIT` returns. Once a query *does*
   pair `LIMIT`/`OFFSET` with `ORDER BY`, the engine needs a real, total order
   to make that pairing deterministic — which is exactly the case this
   chapter's running example ends on (§15.7).

None of this is decoration. It's the reason DuckDB doesn't get to have one
small, ORDER-BY-shaped sort routine tucked away in a corner — it needs one fast,
general, parallel, spill-to-disk-capable sorting facility that every one of
these five features can call into. The source layout reflects this: the core
machinery lives under `src/common/sort/` (not `src/execution/operator/order/`),
with names like `Sort`, `FullSort`, `HashedSort`, `SortedRun`, and
`SortedRunMerger` — infrastructure classes, not one operator's private
implementation detail.

**A caveat worth stating up front, in the spirit of this book's honesty
policy:** DuckDB's sorting code was rewritten in a large change that shipped in
**v1.4.0, September 2025**. This book is checked against `cb5673685153`, more
than a year later. The DiDi course, whose Chapter 4 this chapter leans on
heavily, says so explicitly: "*To this day, DuckDB's internal sorting routines
are on the workbench and still undergo active development*" [6, slide 3]. That
is still true. Names like `SortedRunMerger`, `SortKeyType`, the memory-tracked
merge buffer (§15.5.4), and the K-way merge-boundary algorithm in §15.6.3 are
current at this commit. Treat them as "true at `cb5673685153`", not as a stable
API.

## 15.2 Two queries, two very different machines

Query A, this book's running join/aggregate query, ends in an `ORDER BY ...
LIMIT`:

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

Chapter 1 already flagged that DuckDB's optimizer fuses this query's
`LogicalOrder` + `LogicalLimit` pair into a single `LogicalTopN`, which becomes
`PhysicalTopN` (Chapter 7 covers the `top_n` optimizer pass that does the
fusing). It's worth being explicit about *why*, and about what that means for
this chapter: **`PhysicalTopN` does not use the two-phase merge sort this
chapter is about.** If you only ever need the top 10 rows out of what might be
tens of millions of `(custkey, total)` groups, fully sorting all of them and
then throwing away everything past row 10 is wasted work — you never need a
total order over the whole input, only enough of one to know which 10 rows are
the winners. `PhysicalTopN` instead maintains a **bounded candidate heap** of
at most `limit + offset` entries (`heap_size(limit + offset)`,
`src/execution/operator/order/physical_top_n.cpp:181`) and — interestingly,
this reuses a mechanism from later in this very chapter, see §15.7 — compares
candidates using the same normalized sort-key encoding that `create_sort_key()`
exposes at the SQL level. Query A is the right query to introduce *why* this
chapter's machinery exists, but the wrong query to *walk through* it with,
because Query A's `LIMIT` routes around it entirely.

So this chapter needs its own running example — one with no `LIMIT`, where the
engine genuinely has no choice but to materialize a full global order over
everything:

```sql
-- Query C — the chapter's running example: a full, unbounded sort
SELECT *
FROM lineitem
ORDER BY shipdate, orderkey;
```

`lineitem` at TPC-H scale factor 100 is 600 million rows. There is no shortcut
here: every row has to end up somewhere in a single, globally-ordered output,
and the engine can't know where until it has looked at (nearly) everything.
This is the query this chapter's Phase 1 and Phase 2 sections trace through.

## 15.3 Two-phase merge sort, at a glance

DuckDB implements a variant of a classic **two-phase merge sort**, and it
parallelizes *both* phases across all `T` available CPU cores — not just the
first one, which is the easy part to parallelize:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 1 — thread-local sort  (T threads, fully independent)          │
│                                                                        │
│  thread 0: read 1/T of lineitem → normalize keys → sort  → run₀       │
│  thread 1: read 1/T of lineitem → normalize keys → sort  → run₁       │
│  thread 2: read 1/T of lineitem → normalize keys → sort  → run₂       │
│    ...                                                                │
│  thread T-1: read 1/T of lineitem → normalize keys → sort → runT₋₁    │
└──────────────────────────────┬────────────────────────────────────────┘
                                │  T independently-sorted runs
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 2 — parallel T-way merge                                       │
│                                                                        │
│  Precompute non-overlapping boundaries that split the T runs into      │
│  T output partitions, so each thread can merge "its" partition          │
│  independently, with no coordination once boundaries are known.        │
│                                                                        │
│  thread 0 → output partition 0 (globally first slice of final order)   │
│  thread 1 → output partition 1                                          │
│    ...                                                                │
│  thread T-1 → output partition T-1 (globally last slice)                │
└─────────────────────────────────────────────────────────────────────┘
```

(This picture is redrawn from [6, slide 4] — "*DuckDB implements a
variant of a two-phase merge sort strategy that utilizes all available CPU
cores in both phases*" — using this book's own box-drawing convention.)

The rest of this chapter is organized around the single cleverest idea that
makes both phases fast — **key normalization** (§15.4) — and then walks Phase 1
(§15.5) and Phase 2 (§15.6) each in depth, grounded in the real
`src/common/sort/` source.

## 15.4 Key normalization: one comparison instead of many

### 15.4.1 The problem: multi-column comparators are slow and don't vectorize

Query C's `ORDER BY shipdate, orderkey` is already a two-column sort. Real
`ORDER BY` clauses routinely mix ascending and descending columns and different
null placements, e.g.:

```sql
SELECT * FROM lineitem
ORDER BY l_shipdate DESC NULLS FIRST, l_quantity;
--                   ▲▲▲▲ ▲▲▲▲▲▲▲▲▲▲   ▲
--         defaults: ASC  NULLS LAST   (lexicographic: shipdate dominates)
```

([6, slide 2]; DuckDB's session-level defaults for the parts left
unspecified are configurable — `SET default_order = {ASC, DESC}` and
`SET default_null_order = {NULLS_FIRST, NULLS_LAST,
NULLS_FIRST_ON_ASC_LAST_ON_DESC, NULLS_LAST_ON_ASC_FIRST_ON_DESC}` — but the
out-of-the-box default is `ASC NULLS LAST`.)

The obvious way to compare two rows under a clause like this is a **cascading
comparator**: compare `shipdate` first; if equal, fall through to compare
`orderkey`; and at each step, dispatch on the column's type to pick the right
`<` operator, negate the result if that column is `DESC`, and special-case
`NULL`. This is exactly the kind of code every database textbook shows, and
it's exactly the kind of code that's slow in practice: it's branchy (a
type dispatch and a direction check per column, per comparison), it can't be
vectorized (the number of columns actually compared varies row to row,
depending on where the first difference falls), and every one of the `T` sort
threads in Phase 1 executes it billions of times over a 600-million-row table.

### 15.4.2 The solution: normalize every key into one comparable byte sequence

DuckDB's answer is **key normalization**: transform every row's sort key —
however many columns, however many different directions and null orders —
into a single byte sequence, constructed so that **one plain lexicographic
byte-array comparison on the transformed sequence gives exactly the same
answer the original multi-column comparator would have given.** Once that's
done, the sort algorithm itself never needs to know it's sorting a
`(shipdate, orderkey)` pair with mixed directions — it just needs a `<`
operator over byte strings (or, as §15.4.4 shows, over a few `uint64_t`
words), which is both branch-free and something every off-the-shelf sort
algorithm already knows how to do.

Three encoding tricks make the transformed byte sequence behave correctly, and
all three are visible, verbatim, in `src/function/scalar/create_sort_key.cpp`
— the file that implements `create_sort_key()`, the scalar function that
exposes this exact machinery at the SQL level (§15.4.3):

**Trick 1 — a leading sentinel byte encodes NULL placement.** Every encoded
value is prefixed by one byte that says "this value is NULL" or "this value is
present," and which of the two byte values means which is swapped depending on
`NULLS FIRST` vs `NULLS LAST`:

```cpp
// src/function/scalar/create_sort_key.cpp:98-116
struct SortKeyVectorData {
    static constexpr data_t NULL_FIRST_BYTE = 1;
    static constexpr data_t NULL_LAST_BYTE = 2;
    ...
    null_byte = NULL_FIRST_BYTE;
    valid_byte = NULL_LAST_BYTE;
    if (modifiers.null_type == OrderByNullType::NULLS_LAST) {
        std::swap(null_byte, valid_byte);
    }
```

Under `NULLS FIRST`, a NULL row gets sentinel byte `1` and a present value gets
`2` — so byte-lexicographic order puts every NULL row before every non-NULL
row, regardless of what the non-NULL value actually is, because `1 < 2` and
that one byte is compared before anything else. Under `NULLS LAST` the two
bytes swap, flipping which side NULLs land on. This is the "one leading
sentinel byte" trick the task summary anticipated, confirmed verbatim in
source.

**Trick 2 — bitwise-inverting every byte reverses `DESC` order.** A column
marked `DESC` needs the *opposite* of what plain lexicographic order would
give it. DuckDB's trick: flip every bit of every encoded byte for that column
(`~byte`), including the sentinel byte from Trick 1. Flipping every bit of a
byte string exactly reverses its lexicographic order (`a < b` before flipping
implies `~a > ~b` after), so a single boolean, computed once per column and
threaded through the whole encoding, is enough to make `DESC` "just work"
against the same plain byte-comparison:

```cpp
// src/function/scalar/create_sort_key.cpp:602-611
struct SortKeyConstructInfo {
    SortKeyConstructInfo(OrderModifiers modifiers_p, ...) {
        flip_bytes = modifiers.order_type == OrderType::DESCENDING;
    }
    ...
    bool flip_bytes;
```

Every encoding routine in the file — fixed-width numeric encoding, string
encoding, blob encoding, even the end-of-list delimiter for nested types — is
templated on this `FLIP_BYTES` flag and applies `~` to its output bytes when
it's set (e.g. `SortKeyWord::FlipByte<FLIP_BYTES>`, used throughout). Because
the flip is applied to *every* byte the column contributes, including its
NULL/valid sentinel, `DESC NULLS FIRST` and `DESC NULLS LAST` both come out
correct without special-casing the interaction between the two tricks.

**Trick 3 — signed integers get their sign bit flipped, after a byte-swap to
big-endian.** This is the trick that makes numeric columns work at all under
byte-lexicographic comparison. A little-endian machine stores `int32_t` values
byte-reversed relative to their numeric magnitude, so the raw bytes of `-5`
and `3` don't compare in numeric order; and even in big-endian byte order,
two's-complement negative numbers have their sign bit set, which makes them
look *larger* than positive numbers under an unsigned byte comparison — the
opposite of what's needed. DuckDB's `Radix::EncodeSigned` fixes both problems
in one pass — byte-swap to big-endian, then flip just the sign bit of the
first byte:

```cpp
// src/include/duckdb/common/radix.hpp:166-172
template <class T>
void Radix::EncodeSigned(data_ptr_t dataptr, T value) {
    using UNSIGNED = typename MakeUnsigned<T>::type;
    UNSIGNED bytes;
    Store<T>(value, data_ptr_cast(&bytes));
    Store<UNSIGNED>(BSwapIfLE(bytes), dataptr);
    dataptr[0] = FlipSign(dataptr[0]);   // FlipSign(b) is just b ^ 128
}
```

Worked example, `int32_t`: `-5` is `0xFFFFFFFB` in two's complement.
Byte-swapped to big-endian: `FF FF FF FB`. Flip the sign bit of the first
byte (`FF ^ 0x80 = 0x7F`): the encoded key is `7F FF FF FB`. Now encode `3`
(`0x00000003`): big-endian is `00 00 00 03`; flipping the first byte's sign bit
(`00 ^ 0x80 = 0x80`) gives `80 00 00 03`. Comparing the two byte strings as
plain unsigned bytes: `7F... < 80...`, so `-5` sorts before `3` — correctly,
even though nothing about the comparison knew these were signed integers. This
is the identical family of trick used to make ART's index keys
byte-comparable (Chapter 16 covers that in full) — flipping the sign bit so
two's-complement's "negative numbers look big" property gets cancelled out
before a plain unsigned byte comparison is applied.

Strings get a fourth, narrower trick worth a one-line mention: every byte of
the string is incremented by one before encoding (`input_data[pos] + 1`,
`create_sort_key.cpp:251`), specifically so that no *content* byte can ever
collide with the reserved `0x00` delimiter byte appended after the string —
the same delimiter byte that Trick 2's `~` flip turns into `0xFF` under
`DESC`, still consistently the largest byte value either way.

### 15.4.3 `create_sort_key()`: the same mechanism, exposed to SQL

DuckDB doesn't keep this trick as a private implementation detail — it exposes
it directly as a scalar function, `create_sort_key()`, documented in
`src/function/scalar/generic/functions.json` as:

> **create_sort_key(key1, ...)** — *"Constructs a binary-comparable sort key
> based on a set of input parameters and sort qualifiers."*
> Example: `create_sort_key('A', 'DESC')`

The function takes an alternating list of `(value, sort_specifier)` pairs —
so a realistic call over two columns with mixed direction looks like:

```sql
SELECT create_sort_key(shipdate, 'ASC', orderkey, 'DESC') AS key
FROM lineitem
ORDER BY key;
```

`ORDER BY key` here gives exactly the same row order as
`ORDER BY shipdate ASC, orderkey DESC` would — because `key` *is* the
normalized byte sequence from §15.4.2, and DuckDB's binder actually rewrites
plain `ORDER BY` clauses into calls to this same underlying construction logic
internally. If every input column has a small, fixed-width type and the
combined width fits in 8 bytes, `create_sort_key()`'s bind logic
(`CreateSortKeyBind`, `create_sort_key.cpp:65-92`) even sets its return type to
plain `BIGINT` instead of `BLOB` — the small-key fast path described next.

### 15.4.4 `FixedSortKey`: packing short keys into `uint64_t` words

A CPU compares two 64-bit integers with a single instruction; comparing two
arbitrary-length byte arrays needs a loop (or at best a wide SIMD `memcmp`).
For the common case — a sort key that, once normalized, is short — DuckDB
avoids the byte-array comparison entirely by packing the normalized bytes into
one or more `uint64_t` "parts" and comparing those integers directly:

```cpp
// src/include/duckdb/common/sorting/sort_key.hpp:344-349 (NO_PAYLOAD_FIXED_16 shown)
template <>
struct SortKey<SortKeyType::NO_PAYLOAD_FIXED_16>
    : FixedSortKey<SortKey<SortKeyType::NO_PAYLOAD_FIXED_16>, false> {
    static constexpr idx_t PARTS = 2;
    static constexpr idx_t INLINE_LENGTH = 16;
    uint64_t part0;
    uint64_t part1;
};
```

and the comparison really is just chained integer comparisons, generated
recursively over however many parts the key has:

```cpp
// src/include/duckdb/common/sorting/sort_key.hpp:149-154
template <idx_t REMAINING>
static bool SortKeyLessThan(const uint64_t *const &lhs, const uint64_t *const &rhs) {
    return (*lhs < *rhs) || ((*lhs == *rhs) && SortKeyLessThan<REMAINING - 1>(lhs + 1, rhs + 1));
}
template <>
inline bool SortKeyLessThan<1>(const uint64_t *const &lhs, const uint64_t *const &rhs) {
    return *lhs < *rhs;
}
```

A source comment on this exact tradeoff is worth quoting directly, because it
puts real numbers on "make the common case fast" rather than leaving it as an
assertion — measuring `ORDER BY` over 20 million rows, trimmed (fewer
`SortKeyType` variants compiled in) vs. not:

```
ORDER BY 1x INTEGER  NO_PAYLOAD_FIXED_8  kept                1.06x slower if trimmed
ORDER BY 1x BIGINT   NO_PAYLOAD_FIXED_16 -> forced to FIXED_32  1.56x slower if trimmed
ORDER BY 2x BIGINT   NO_PAYLOAD_FIXED_24 -> forced to FIXED_32  1.26x slower if trimmed
ORDER BY 3x BIGINT   NO_PAYLOAD_FIXED_32 kept                1.03x slower if trimmed
```
(`src/include/duckdb/common/sorting/sort_key.hpp:32-47`)

In other words, when a key that would naturally fit in 16 or 24 bytes gets
forced up to the next wider fixed layout (or, in a further-trimmed build, all
the way down to the fully variable-length `BLOB`-backed key), real `ORDER BY`
queries measurably slow down — up to 1.6–2.4x in the most aggressive
trim tested. The width split into `NO_PAYLOAD_FIXED_8` through `_32` and
`PAYLOAD_FIXED_16` through `_32` (`SortKeyType` enum, `sort_key.hpp:19-30`) —
where, for instance, `PAYLOAD_FIXED_24` is exactly a two-word normalized key
(16 bytes, `PARTS = 2`) plus an 8-byte pointer back to the row's payload
(`sort_key.hpp:391-397`) — exists *because* of this — it's a direct, source-level
example of trading a modest amount of generated code (the comment estimates
~70 KB of Release-build code per `SortKeyType` variant, ~623 KB for all nine
combined) for real, measured throughput on the hot path.

One more detail worth naming because it's genuinely counter-intuitive: the
comment in `FixedSortKey::Construct` (`sort_key.hpp:189-192`) notes that the
normalized bytes aren't actually *stored* in the byte-comparable order derived
in §15.4.2 — they're stored byte-swapped once more, specifically so that the
`part0`/`part1`/... fields can be compared as native machine integers (which,
on a little-endian CPU, compare in the *opposite* byte order from what a
`memcmp` would need). The externally-visible behavior — "lexicographic byte
comparison on the logical key" — is unchanged; it's an internal representation
choice, undone again (`ByteSwap()`) whenever a key needs to be decoded back
into its original typed value.

Decoding a key does not always give back the exact original value, though.
Floating-point keys are the example. `-0.0` and `0.0` must compare as equal, so
they get the same encoded bytes. If the sort decoded such a key to produce its
output, every `-0.0` would come back as `0.0`. So `Sort::Sort` does not read a
`FLOAT` or `DOUBLE` column (or a nested type that contains one) back from the
key. It also copies the column into the payload, which keeps the original bits:

```cpp
// src/common/sort/sort.cpp:79-84
//! The sort key encoding for FLOAT/DOUBLE is not injective around zero (-0.0 and 0.0 encode
//! identically, since they must compare equal), so re-decoding the key would turn -0.0 into 0.0.
//! Route such columns through the payload instead, where the original bits are preserved.
const bool key_encoding_is_lossy = TypeVisitor::Contains(input_col_type, LogicalTypeId::FLOAT) ||
                                   TypeVisitor::Contains(input_col_type, LogicalTypeId::DOUBLE);
if (it != input_column_to_key.end() && !key_encoding_is_lossy) {
```

`test/sql/order/order_by_negative_zero.test` checks it. The table has the rows
`(1, -0.0)` and `(2, 0.0)`. After `ORDER BY v, id`, the query
`SELECT v, 1.0 / v` must return `-0.0, -inf` and `0.0, inf`, so the first row
keeps its sign.

## 15.5 Phase 1: thread-local sort

### 15.5.1 Sink, Combine, Finalize — the standard DuckDB shape, applied to sorting

Query C's `ORDER BY shipdate, orderkey` sink follows the same
Sink/Combine/Finalize shape every DuckDB pipeline sink follows (Chapter 11
covers this pattern generally). Each of the `T` worker threads gets its own
`LocalSinkState` (`FullSortLocalSinkState`,
`src/common/sort/full_sort.cpp:97`), which owns exactly one thread-local
`SortedRun` — a normalized-key `TupleDataCollection` plus, if the query has a
payload beyond the key, a second `TupleDataCollection` holding the actual row
data (`SortedRun::Sink`, `src/common/sort/sorted_run.cpp`). As a thread pulls
its share of `lineitem`'s row groups through the pipeline, every `DataChunk`
gets run through the exact key-normalization logic from §15.4 and appended to
that thread's `SortedRun`. At `Finalize`, each thread's accumulated `SortedRun`
gets sorted independently and in parallel — no cross-thread coordination is
needed for this step at all, which is what makes Phase 1 straightforward to
parallelize compared to Phase 2 (§15.6).

Rows aren't accumulated into one contiguous array — they're accumulated onto a
chain of fixed-size **256KB data pages** (`DEFAULT_BLOCK_ALLOC_SIZE = 262144`,
`src/include/duckdb/storage/storage_info.hpp:33` — the same page size
Chapter 14 introduces for row-major storage of intermediates generally). A
600-million-row `lineitem` sort, split across, say, 16 threads, means each
thread's `SortedRun` is built from many such pages chained together, not one
giant allocation.

### 15.5.2 A custom iterator so a page chain looks like one array

This immediately creates a problem for handing the accumulated rows to an
off-the-shelf sorting algorithm: every general sort implementation (the C++
standard library's, and the three third-party ones DuckDB uses — see §15.5.3)
is written against the `std::iterator` random-access interface, which assumes
the underlying data is one contiguous, uniformly-addressable array. A chain of
256KB pages is not that. DuckDB's answer is a custom iterator,
`BlockIteratorState`, that presents a page chain *as if* it were one
contiguous array of sort keys, translating a flat logical index into a
`(page index, offset within page)` pair on every access:

```cpp
// src/include/duckdb/common/types/row/block_iterator.hpp:96-110 (abbreviated)
class BlockIteratorState<BlockIteratorStateType::IN_MEMORY>
    : public BlockIteratorStateBase<...> {
public:
    void RandomAccessInternal(idx_t &block_idx, idx_t &tuple_idx, const idx_t &index) const {
        block_idx = fast_mod.Div(index);      // which 256KB page
        tuple_idx = fast_mod.Mod(index, block_idx); // offset within that page
    }
    template <class T>
    T &GetValueAtIndex(const idx_t &block_idx, const idx_t &tuple_idx) const {
        return reinterpret_cast<T *const>(block_ptrs[block_idx])[tuple_idx];
    }
```

with `Increment`/`Decrement`/`Add`/`Subtract` implemented on top of that
translation (`block_iterator.hpp:44-70`) to satisfy the full random-access
iterator contract — including the fast-path division (`FastMod`) needed
because "which page is index `i` on" is a division and a modulo on every
single random access, and a naive integer division there would be expensive
enough to matter at this scale.

**Why this is necessary, not just tidy:** a single 256KB page holds only about
10,000 24-byte sort keys (`262144 / 24 ≈ 10,922`) — and 24 bytes is a real,
common key width in this codebase, as §15.4.4 already showed with
`PAYLOAD_FIXED_24`. Ten thousand keys is nowhere near a long enough run for a
general sorting algorithm's runtime characteristics to be attractive — some of
the very cache and branch-prediction benefits that make an algorithm like
Ska Sort or pdqsort fast depend on it operating over a genuinely long
contiguous run, not being restarted from scratch every 10,000 elements. The
iterator is what lets DuckDB hand these algorithms one logical run spanning
however many hundred pages a thread accumulated, instead of one run per page.

### 15.5.3 The three-algorithm cascade

With a random-access iterator over `SortKey` values in hand (comparable either
via `operator<` on their packed `uint64_t` words, §15.4.4, or via `memcmp` for
the variable-length key layouts), DuckDB doesn't reach for one general sorting
algorithm — it combines three, vendored under `third_party/` and wired
together as a decision cascade in
`src/common/sort/sorted_run.cpp:247-260`:

```cpp
// src/common/sort/sorted_run.cpp:255-260 (TemplatedSort)
SortMergeBuffer merge_buffer(BufferAllocator::Get(context), key_data.Count(), sizeof(SORT_KEY));
const BufferedMergePolicy<SORT_KEY> merge(merge_buffer);
const auto fallback = [&ska_extract_key, &merge](const BLOCK_ITERATOR &fb_begin, const BLOCK_ITERATOR &fb_end) {
    duckdb_ska_sort::ska_sort(fb_begin, fb_end, ska_extract_key, merge);
};
duckdb_vergesort::vergesort(begin, end, std::less<SORT_KEY>(), fallback, merge);
```

§15.5.4 explains the first two lines and the extra `merge` argument.

```
                    ┌─────────────────────────────┐
                    │  vergesort(begin, end, ...)  │   cheap pre-scan
                    └───────────────┬───────────────┘
                                    │
                does the data contain long already-sorted
                or reverse-sorted runs?
                                    │
                ┌───────────────────┴───────────────────┐
               yes                                       no
                │                                         │
   merge the detected runs directly           ┌───────────────────────┐
   (near-linear, no radix pass needed)         │  ska_sort (radix on   │
                                                │  first 64 bits of key)│
                                                └───────────┬────────────┘
                                                             │
                                          do the first 64 bits already
                                          discriminate most rows?
                                                             │
                                            ┌─────────────────┴─────────────────┐
                                           yes                                  no
                                            │                                    │
                                  done — near-linear radix buckets     ┌──────────────────────┐
                                                                        │ pdqsort_branchless    │
                                                                        │ (pattern-defeating     │
                                                                        │  quicksort, general    │
                                                                        │  fallback)             │
                                                                        └──────────────────────┘
```

Each stage's job, and why the cascade is ordered this way:

- **Vergesort** runs first because it's the cheapest possible win: real-world
  data is very often partially sorted already (`lineitem` scanned in
  insertion order is already close to sorted by `orderkey`; a table freshly
  loaded in `shipdate` order is close to sorted by `shipdate`), and detecting
  a long already-sorted (or exactly reverse-sorted) run costs a linear scan,
  not a full sort. Where such runs exist, vergesort merges them directly and
  never invokes the more expensive stages at all. This is a pure win-or-do-no-
  harm layer: on data with no exploitable order, its pre-scan cost is small
  relative to what follows.
- **Ska Sort** is a radix sort keyed on the first 64 bits of the *already
  normalized* key (`SkaExtractKey::operator()` literally returns `key.part0`,
  `src/common/sort/sorted_run.cpp:214`) — an American Flag / byte-at-a-time
  radix pass that's `O(n)` rather than `O(n log n)`, and very fast, whenever
  those leading 8 bytes already discriminate most rows from each other.
  Because §15.4's key normalization puts the *dominant* sort column's bytes
  first, this is exactly the common case for a well-chosen `ORDER BY` — Query
  C's leading column, `shipdate`, has enough distinct values that its encoded
  bytes alone separate most rows.
- **pdqsort** (pattern-defeating quicksort) is the general-purpose fallback,
  used both as vergesort's own fallback for chunks it can't detect order in,
  and — this is a detail confirmed directly in the vendored source, not just
  DiDi's slides [6] — *inside* DuckDB's fork of Ska Sort itself. Ska Sort uses
  pdqsort in two places. First, a bucket with fewer than 128 rows goes
  straight to `pdqsort_branchless` (`StdSortIfLessThanThreshold`,
  `third_party/ska_sort/ska_sort.hpp:1097-1107`). For so few rows a radix pass
  does not pay for itself. Second, a bucket where every row has the *same*
  leading 64 bits cannot be split by the radix pass at all. Many `lineitem`
  rows share one `shipdate`, for example, and only `orderkey`, the key's *next*
  word, tells them apart. Such a bucket goes to `StdSortFallback`
  (`ska_sort.hpp:1084-1095`). That runs vergesort with the full comparison,
  and vergesort falls back to pdqsort. Small buckets skip vergesort because,
  as the source comment says, below 128 elements vergesort "would hand the
  whole range to pdqsort without merging anything". `StdSortFallback` also
  takes the merge policy from §15.5.4, so its merges use the same tracked
  buffer.

The engineering logic behind the cascade, stated plainly: **try the cheapest,
most-specialized technique first, and only pay for a more general algorithm
when the specialized one can't finish the job.** Vergesort is nearly free and
exploits order that's already there; Ska Sort is linear and exploits the fact
that a normalized key's leading bytes usually discriminate well; pdqsort is
the correctness backstop that every input, however adversarial, still
terminates correctly and reasonably fast against. No single one of the three
is chosen as "the" sort algorithm — the combination is.

### 15.5.4 The merge buffer: scratch memory that counts against the limit

Vergesort finds sorted runs and then merges them two at a time. The obvious
tool for that is `std::inplace_merge` from the C++ standard library. But that
function asks `operator new` for a temporary buffer of up to half the keys, and
DuckDB's memory limit (Chapter 14) does not see memory that comes from
`operator new`. In some window-function sorts that hidden memory reached a
131 MB peak and 28 GB of allocation churn, all invisible to `memory_limit`.

So the sort allocates this scratch space itself, through the buffer allocator,
before it starts. The class is `SortMergeBuffer`:

```cpp
// src/include/duckdb/common/sorting/buffered_merge.hpp:19-44
//! Scratch buffer for merging sorted runs: the merge is correct without it, so the allocation must never fail a sort.
//! Starting at half the keys, the size is halved until the buffer manager can provide it (or it becomes too small to
//! be worth having).
class SortMergeBuffer {
public:
	//! Number of keys the buffer holds at most: the smaller side of a merge is at most half the keys
	static idx_t MaximumKeyCount(idx_t key_count) {
		return key_count / 2 + 1;
	}
	// ...
	SortMergeBuffer(Allocator &allocator, idx_t key_count, idx_t key_size) : count(0) {
		static constexpr idx_t MINIMUM_KEYS = 1024;
		for (idx_t keys = MaximumKeyCount(key_count); keys >= MINIMUM_KEYS; keys /= 2) {
			try {
				data = allocator.Allocate(keys * key_size);
				count = keys;
				return;
			} catch (OutOfMemoryException &) {
				// try a smaller buffer
			}
		}
	}
```

Two ideas are in this code:

- **The buffer is optional.** A merge gives the right answer without any
  buffer; it is only slower. So the constructor never fails. It tries half the
  keys, then a quarter, then an eighth, and so on. If even 1,024 keys do not
  fit, it gives up and keeps an empty buffer. Then `BufferedMerge` merges in
  place, with rotations and no extra memory.
- **One buffer serves every merge in the sort.** A `BufferedMergePolicy`
  (`buffered_merge.hpp:174-190`) wraps the buffer. `TemplatedSort` passes it to
  vergesort and to Ska Sort (the excerpt in §15.5.3). Ska Sort passes it on to
  `StdSortFallback`, so the nested vergesort inside a radix bucket uses the
  same buffer. The merge step of Phase 2 does the same
  (`sorted_run_merger.cpp:659-664`).

Because the buffer is real, counted memory, the sink must plan for it. A
thread-local run is measured with `SortedRun::SizeInBytesForSorting()`,
which is the run's size plus the largest buffer its sort could ask for
(`src/common/sort/sorted_run.cpp:457-460`). `TryFinishSink` compares that
number, not the bare run size, with the per-thread budget
(`src/common/sort/sort.cpp:227-233`). A run that would not fit together with
its buffer is sorted and closed sooner.

## 15.6 Phase 2: parallel T-way merge

### 15.6.1 Why the merge itself needs to be parallel, not just Phase 1

After Phase 1, the engine holds `T` independently-sorted runs (one per
thread) that together cover all of `lineitem`. A textbook merge sort would now
merge all `T` runs with one thread, using a single min-heap of size `T`
scanning across them — correct, but a hard bottleneck: however well Phase 1
was parallelized across 16 or 32 cores, Phase 2 would be back down to one core
doing all the work, for a table with hundreds of millions of rows. DuckDB
avoids this by parallelizing the merge itself: precompute, ahead of time and
without doing any of the actual merging, where each of `T` independent
"segments" of the final output begins and ends in each of the `T` input runs —
then let `T` threads merge their segments fully independently, each one
producing a contiguous slice of the final output with zero coordination
needed once the boundaries are fixed. The algorithm that computes those
boundaries without ever performing the merge is called **Merge Path**
[32].

### 15.6.2 Merge Path, the textbook two-way idea

The clearest way to build intuition for Merge Path is the two-array case.
Take two small sorted arrays:

```
A = [1, 3, 5, 7, 9]           (5 elements)
B = [2, 4, 6, 8, 10, 12]      (6 elements)
```

The ordinary merge algorithm walks both arrays left to right, at each step
taking whichever of the two current elements is smaller, and produces:

```
1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 12
```

Think of this walk as a **staircase path** through an 11-total-element output
space: every step is either "take from A" (move right) or "take from B" (move
down). The key realization behind Merge Path: **you can find exactly where
that staircase is after emitting any given number `k` of output elements
without ever walking the staircase from the start** — by binary search.

Suppose we want to know: what are the first 6 elements of the merged output?
Equivalently: for some split `i` elements taken from A and `6 - i` elements
taken from B, is that split *valid* — meaning everything on the "taken" side
of both arrays is `≤` everything on the "not yet taken" side? A split `i` is
valid exactly when `A[i-1] ≤ B[6-i]` and `B[6-i-1] ≤ A[i]` (with out-of-range
indices treated as `-∞`/`+∞`). Because both arrays are sorted, moving `i` up by
one always moves one of these two comparisons from "OK" to "not OK" and the
other from "not OK" to "OK" — the valid split point is unique, and binary
search over `i` finds it in `O(log(min(|A|,|B|)))` comparisons instead of
walking all 6 steps:

```
 A-index i →   0    1    2    3    4    5
              ┌────┬────┬────┬────┬────┬────┐
              │ -∞ │  1 │  3 │  5 │  7 │  9 │
              └────┴────┴────┴────┴────┴────┘
                         try i=3: A[2]=5, B[6-3-1]=B[2]=6  → 5 ≤ 6  OK
                                  B[6-3]=B[3]=8, A[3]=7     → 8 ≤ 7  FAILS
                         → need MORE from A: try i=4
                         i=4: A[3]=7, B[6-4-1]=B[1]=4       → 7 ≤ 4  FAILS
                         → too much from A: try i=3 was closer... converge at i where both hold

           the binary search converges on i=4, 6-i=2:
           take A[0..3] = [1,3,5,7]  and B[0..1] = [2,4]
           merged: 1,2,3,4,5,7 — sorted, and exactly the first 6 elements ✓.
```

That's the whole idea: **a binary search along the "cross-diagonal" of a
conceptual (never materialized) merge matrix locates a valid split point for
any target output length**, without running the merge up to that point. Do
this once per desired boundary — say, 8 evenly-spaced boundaries for 8
threads — and each of the 8 (A-range, B-range) segment pairs between
consecutive boundaries can be merged completely independently, in parallel,
each producing one contiguous chunk of the final output.

### 15.6.3 DuckDB's actual K-way generalization

The two-array case above generalizes cleanly to two arrays; DuckDB's actual
problem is **K-way**, with one input run per sort thread (16, 32, however many
cores were used in Phase 1) rather than two. The real implementation, in
`SortedRunMergerLocalState::TemplatedComputePartitionBoundaries`
(`src/common/sort/sorted_run_merger.cpp:484-567`, explicitly commented "//
Computes upper partition boundaries using K-way Merge Path" at line 104), does
not do a literal per-pair binary search generalized to K arrays — it uses a
different, and just as elegant, iterative scheme that converges to the same
kind of boundary. Worth walking through, because it's real committed source,
not a paraphrase:

```cpp
// src/common/sort/sorted_run_merger.cpp:527-561 (abbreviated)
while (total_remaining != 0) {
    const idx_t k = active_run_idxs.size();

    // For each still-active run, propose taking an equal (k-way) share of
    // what's left, capped by how much that run actually has remaining...
    auto min_idx = active_run_idxs[0];
    auto min_delta = ComputeBoundaryDelta(total_remaining, k, run_boundaries[min_idx]);
    reference<SORT_KEY> min_value = run_iterators[min_idx][run_boundaries[min_idx].begin + min_delta - 1];
    for (idx_t i = 1; i < active_run_idxs.size(); i++) {
        const auto &active_run_idx = active_run_idxs[i];
        const auto run_delta = ComputeBoundaryDelta(total_remaining, k, run_boundaries[active_run_idx]);
        auto &run_value = run_iterators[active_run_idx][run_boundaries[active_run_idx].begin + run_delta - 1];
        if (run_value < min_value.get()) {          // ...and commit only the SMALLEST proposal
            min_idx = active_run_idx;
            min_delta = run_delta;
            min_value = run_value;
        }
    }
    run_boundaries[min_idx].begin += min_delta;      // safe: nothing in any other
                                                      // run's untaken portion can be smaller
    if (run_boundaries[min_idx].begin == run_boundaries[min_idx].end) {
        active_run_idxs.erase(...);                  // this run is now fully assigned
    }
    total_remaining -= min_delta;
}
```

The idea: at each step, every still-undetermined ("active") run proposes an
equal-share candidate cut — "if I take my fair 1/k share of what's still
needed, my cut point would land on this key value." Whichever run's proposed
cut value is the *smallest* across all active runs is guaranteed safe to
commit immediately, because by construction no other active run has anything
smaller left to contribute before that value — committing it can never
violate global order. That run's boundary advances by its proposed delta
(shrinking `total_remaining`), and the process repeats — a handful of
iterations, not one per element, because each iteration commits roughly a
`1/k` share of the remaining work. This is the same *problem* the classic
2-way Merge Path solves — find non-overlapping, order-respecting boundaries
across the input runs without performing the merge — generalized to K runs at
once via iterative fair-share elimination rather than a literal per-pair
binary search. Both approaches share the essential property that makes them
worth having at all: **the cost of finding a boundary is far smaller than the
cost of actually merging up to it.**

The chapter's source material (the recovered pages of DiDi-04 [6]) is explicit
that this boundary-finding step is cheap relative to the real merge work, but
the deck's own text cuts off right at the question it's building to — "*How
can all CPU cores contribute equally during this merge?*" [6, slide 9],
the last recovered page) — before stating a number. A commonly-repeated figure
in secondary discussion of this design is that the boundary computation costs
no more than a couple of percent of total merge time; this book could not
independently confirm an exact percentage either in the accessible slide pages
or in source-level profiling comments, so it's reported here only as an
unconfirmed figure, not a source-backed fact — consistent with this book's
policy on secondary-literature claims (see the front matter). What *is* directly
observable in the source is the shape of the algorithm that makes such a claim
plausible: no data is moved and no comparisons beyond `O(k)` per elimination
step are performed while finding boundaries; all the real per-row work happens
afterward, in `TemplatedMergePartition`, once boundaries are fixed.

One more real detail worth surfacing: adjacent partitions share work. A
partition's `begin` boundary is exactly the previous partition's computed
`end` boundary, so `ComputePartitionBoundaries`
(`sorted_run_merger.cpp:386-459`) checks whether an earlier partition already
computed the value it needs and reuses it (`begin_computed`,
guarded by a per-partition lock) instead of recomputing from scratch — letting
partitions be picked up by whichever thread becomes free next, rather than
statically pinning partition `i` to thread `i`.

### 15.6.4 Batch indices: reconstructing global order from out-of-order chunks

Once boundaries are fixed, `T` threads merge their own partitions and emit
`DataChunk`s independently — and, because threads finish at different times,
those chunks can reach a downstream operator in any order. What guarantees the
*final* output is still globally ordered is a **batch index** attached to
every emitted chunk: `SortedRunMerger::GetPartitionData` tags each chunk with
its owning partition's index —

```cpp
// src/common/sort/sorted_run_merger.cpp:831-836
OperatorPartitionData SortedRunMerger::GetPartitionData(...) const {
    auto &lstate = lstate_p.Cast<SortedRunMergerLocalState>();
    return OperatorPartitionData(lstate.partition_idx.GetIndex());
}
```

— and because partition `0` is, by construction, defined as exactly the first
`partition_size` elements of the *final* merged order, partition `1` the next
`partition_size`, and so on (§15.6.3), this index number *is* the chunk's
position in global order. A downstream consumer never has to inspect a
chunk's contents to know where it belongs — it just needs to reassemble chunks
by ascending partition index, however out-of-sequence they actually arrived
from however many parallel merge threads produced them.

### 15.6.5 `HashedSort`: when a partitioned sort can skip sorting

Two features sort by a *partition key* rather than for a final order:
`OVER (PARTITION BY p)` in window functions, and `COPY ... (PARTITION_BY (p))`,
which writes one directory per value of `p`. Both use `HashedSort`
(`src/common/sort/hashed_sort.cpp`). It splits rows into bins by the hash of
`p`, then sorts each bin by `p` so that all rows of one partition sit next to
each other. `HashedSort` can skip the sort for a bin that turns out to hold just
one key value. It has to track the keys in each bin to find that out.

There is also a cheaper route. Suppose there is exactly one partition key, it is
an integer column, there is no `ORDER BY`, and the statistics give its minimum
and maximum. Then the set of possible keys is known before any row arrives.
`HashedSort` can give each value its own partition and skip hashing, key
tracking and sorting:

```cpp
// src/common/sort/hashed_sort.cpp:681-686
//	A single integer partition key with known bounds is partitioned on directly: every group is single-key
if (can_bypass_single_key_sort && partition_key_count == 1) {
	integer_partitioning = IntegerPartitionedTupleData::TryBoundedKey(
	    client, partition_key_types[0], partitions[0].stats.get(), integer_partition_min, integer_partition_count);
}
track_single_keys = can_bypass_single_key_sort && !integer_partitioning;
```

(`can_bypass_single_key_sort` is true when there is at least one partition key
and no `ORDER BY`, `hashed_sort.cpp:652`.)
`IntegerPartitionedTupleData` makes `max - min + 1` partitions for the values,
plus one for `NULL` keys and one "overflow" partition for keys outside the
bounds. Only the overflow partition can hold more than one key, so only it is
ever sorted. `TryBoundedKey` refuses when the range is too wide. The cap is 256
values, and less when the per-thread memory cannot hold two blocks per
partition (`src/common/types/row/integer_partitioned_tuple_data.cpp:57-100`).

A small example from `test/sql/copy/partitioned/integer_partitioned_write.test`:

```sql
CREATE TABLE t AS SELECT i % 8 AS p, i AS v, 'row ' || i AS s FROM range(20000) tbl(i);
COPY t TO '{TEST_DIR}/int_part1' (FORMAT PARQUET, PARTITION_BY (p));
```

The statistics of `p` are `[0, 7]`. So `HashedSort` makes 8 value partitions
(plus the `NULL` and overflow ones), and each of the 2,500 rows with `p = 3`
goes straight into partition 3. The test then checks that every file holds
exactly one value of `p`. The same file tests negative bounds (`(i % 7) - 3`)
too.

## 15.7 Back to Query A: why the top-k path skips all of this

§15.2 promised a return to Query A's `ORDER BY total DESC LIMIT 10` once the
full machinery was on the table, and the reason to come back is that
`PhysicalTopN` turns out to *reuse* a piece of this chapter rather than
ignoring it wholesale. `TopNHeap` bounds its candidate set to exactly
`limit + offset` entries (`heap_size(limit + offset)`,
`physical_top_n.cpp:181`) — for Query A, 10 entries, no matter how many
millions of `(custkey, total)` groups feed into it — and compares candidates
using `string_t sort_key` fields built with the very same
`CreateSortKeyHelpers` machinery §15.4 walked through
(`#include "duckdb/function/create_sort_key.hpp"`,
`physical_top_n.cpp:8`; `TopNEntry::operator<` just compares `sort_key`
strings, `physical_top_n.cpp:33`). What Query A's plan *doesn't* do is any of
§15.5's thread-local-run-plus-256KB-page-chain construction or §15.6's
parallel K-way merge: there's no need to materialize a full sorted run per
thread when a size-10 heap, updated as rows stream past, already holds the
answer. The two mechanisms share an idea (normalize the key once, compare
cheaply forever after) but solve genuinely different problems — full,
unbounded materialization of a global order (this chapter, Query C) versus
maintaining a small, continuously-updated set of current winners (Chapter 7's
`top_n` fusion pass, Query A) — and conflating them is a common
misunderstanding this book can now rule out with the actual source in hand.

The threads of a parallel top-N share a *boundary value*: the sort key of the
current 10th-best row. A thread can drop any input row that is worse than the
boundary without adding it to its heap. A thread does not wait for the lock to
read the newest boundary. It only tries the lock
(`TopNBoundaryValue::TryGetBoundaryValue`,
`src/execution/operator/order/physical_top_n.cpp:60-67`). If another thread
holds it, the thread keeps using its own older copy of the boundary. An older
boundary is still safe, because it is only less strict. It filters fewer rows
but never a wrong one.

## 15.8 Bridge to the next chapters

This chapter deliberately borrowed vocabulary from two directions: Chapter 14
supplied the 256KB data-page building block that Phase 1's `SortedRun`s are
built from, and Chapter 7 supplied the `top_n` optimizer pass that routes
Query A away from this chapter's machinery entirely. Two forward pointers are
worth planting explicitly. First, §15.4.2's sign-bit-flip and
bitwise-inversion tricks for byte-comparable encoding are not unique to
sorting — the same family of encoding recurs, for the same underlying reason
(making a typed comparison collapse into a plain byte-lexicographic one), in
the design of ART index keys, which Chapter 16 covers in full. Second,
window functions' `ORDER BY` clause (§15.1, item 3) and `ASOF JOIN` (item 4)
both call into this same sorting infrastructure from their own physical
operators — a detail this chapter named but did not trace end to end, left
for whichever later chapter takes up window functions and joins over ordered
data directly.
