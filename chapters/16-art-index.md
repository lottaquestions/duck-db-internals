# Chapter 16 — The ART Index

## 16.1 Two indexes, not one

Every chapter so far has been about the parts of DuckDB that run on *every*
query: parse it, bind it, plan it, optimize it, execute it as a sequence of
vectorized scans and joins. This chapter is about the one part of the engine
that exists specifically to let a query *skip* most of that work — to answer
"find the row(s) where `custkey = 12345`" without touching every row of
`customer`.

DuckDB actually ships two different mechanisms for this, and it is easy to
conflate them because both get called "indexes" informally. They are not the
same kind of thing at all:

1. **Zonemaps** (also called min-max indexes) are baked into the columnar
   storage format itself (Chapter 17 covers row groups and storage in depth).
   Every column is divided into row groups of 120,000 (122,880) rows, and
   every row group carries a `(min, max)` pair per column. `PhysicalTableScan`
   consults these pairs to skip whole row groups a predicate can't possibly
   match. Zonemaps are **automatic** — every column of every table has them,
   there is no `CREATE ZONEMAP` statement — and they are **coarse-grained**:
   they can only decide "maybe this row group has a match" or "definitely
   not," never "here is exactly row 4,819,201." Their usefulness depends
   entirely on whether the table happens to be physically ordered by the
   column in question; an unordered column's row groups all span the full
   value domain, and the zonemap can't skip anything.
2. **Adaptive Radix Tree (ART) indexes** are separate, tree-shaped data
   structures that live alongside the table, not inside it. They are **not**
   automatic — they exist only when a `CREATE INDEX` statement asks for one,
   or implicitly because a `PRIMARY KEY`, `UNIQUE`, or `FOREIGN KEY`
   constraint needs one to enforce itself. They are **fine-grained**: given a
   value, an ART lookup finds the exact row ID(s) that hold it, in time
   proportional to the *bit-length of the key*, not the number of rows in the
   table. This chapter is entirely about this second mechanism.

The rest of this section works out, with real numbers, exactly when an ART
lookup is worth using instead of a sequential scan — because the honest
answer is "not always," and DuckDB's own course material demonstrates a case
where using the index makes a query *slower*.

### Selectivity: the formula and the intuition

An ART lookup for a predicate `p` (`col = val`, or a range) doesn't always
return one row. If `col` isn't unique, one key can map to many row IDs, and
those row IDs point at arbitrary positions in the base table's row-major
sequence of row groups. Following them means the table scan operator jumps
around in storage rather than reading it front-to-back — and "jumping
around" defeats the sequential-access pattern that makes DuckDB's vectorized
columnar scan so fast in the first place (Chapter 10 covers `DataChunk`s and
the batch-at-a-time execution model that sequential scanning exploits;
Chapter 17 covers exactly how a row group's columns are laid out in storage).
Whether an index lookup is a net win depends on **selectivity**:

```
0 ≤ sel(p) ≝ ( SELECT count(*) FILTER (p) FROM t ) / ( SELECT count(*) FROM t ) ≤ 1
```

Low `sel(p)` (few matching rows out of many) means few row IDs to
dereference, so the "jump around" cost is small and the index wins easily.
High `sel(p)` means many row IDs, and at some point the cost of chasing all
of them one-by-one through the base table's storage exceeds the cost of just
reading the whole table sequentially and filtering as you go — because a
sequential vectorized scan over compressed columnar data, batch by batch, is
about as fast as main-memory bandwidth allows, and a random-access row-ID
dereference pays a cache-miss (or worse) per row.

DuckDB actually encodes this tradeoff as a real, tunable threshold rather
than leaving it to intuition. `TryScanIndex` only *offers* the index scan
path at all, and the settings `index_scan_max_count` (default 2048) and
`index_scan_percentage` (default 0.001) gate whether a `TABLE_SCAN` node
becomes an `Index Scan` or a `Sequential Scan`:

```sql
-- expected rows <= MAX(index_scan_max_count, index_scan_percentage * |t|)
--                                            └───────────────────┘
--                                       maximum acceptable selectivity
```

The DiDi course material (Grust, U. Tübingen, chapter 5 companion file
`material/05/016-art.sql` [11]) runs exactly this experiment against a
150-million-row TPC-H `orders` table (scale factor 100) and reports concrete
numbers worth internalizing:

- Building `CREATE INDEX orders_clerk_idx ON orders(o_clerk)` (a **non-unique**
  text column) costs roughly **3.2 GB** of working memory
  (`duckdb_memory()` reports `ART_INDEX` at 3,237,216,256 bytes) — an ART
  index is not free, and it has to be maintained on every subsequent insert,
  update, and delete to that column.
- A selective point lookup, `WHERE o_clerk = 'Clerk#000051887'` (1,518
  matching rows out of 150 million — `sel(p) ≈ 10⁻⁵`), runs via `Index Scan`
  in about **0.004 s**. Forcing a `Sequential Scan` of the same query (by
  setting both thresholds to 0) takes **0.109 s** — roughly **27× slower**.
  The index is a clear win here.
- The same material then builds a second, deliberately adversarial table: 100
  million rows where each distinct key value `i` occurs **40,000 times**
  (`sel(p) = 40,000 / 100,000,000 ≈ 4×10⁻⁴`, still numerically "low," but
  40,000 row IDs is a lot to chase). `WHERE i = 1904` with the index
  *disabled* runs in **0.011 s**. With the index *enabled*, `EXPLAIN ANALYZE`
  reports an `Index Scan` retrieving all 40,000 rows in **0.028 s** — the
  index lookup itself is instant, but dereferencing 40,000 scattered row IDs
  into the column chunks costs more than just scanning the (still fairly
  small) table sequentially would have. **Index usage slows the query down.**

That last result is the whole point of the selectivity formula: an ART index
turns "read N contiguous cache lines" into "look up a key, then perform N
essentially-random single-row fetches," and there is a crossover point past
which the second thing is worse than the first. DuckDB's default thresholds
are a coarse, static guess at where that crossover usually sits; they are not
a guarantee, as the experiment above shows directly.

### When ART indexes get created

```sql
-- Manual:
CREATE [UNIQUE] INDEX name ON table (column [, column, …]);
DROP  INDEX [IF EXISTS] name;

-- Implicit (constraint enforcement) — the DDL statement below creates a
-- unique ART index on t(c) behind the scenes:
CREATE TABLE t (…, c … PRIMARY KEY, …);   -- also: UNIQUE
CREATE TABLE s (…, d … REFERENCES t(c), …);
```

An index key may be an expression, such as `CREATE INDEX i ON t(hash(s))`.
The binder is strict about which expressions it allows. An index is stored on
disk and used by later sessions. So its key must give the same bytes for the
same row every time, in every session. The binder rejects three kinds of key
expression (`IsValidIndexExpression`,
`src/planner/expression_binder/index_binder.cpp:45-64`):

- expressions that depend on the session or the query, such as `now()`,
  `getvariable('x')`, `current_setting('threads')`, or a cast that reads the
  `TimeZone` setting;
- lambda functions, because the index's expression executor has no client
  context to run them with;
- expressions with side effects.

The error is "Index keys cannot contain expressions with side effects".
`test/sql/index/art/create_index_session_dependent.test` shows each case.
The same check also runs when a stored index is loaded. A stored key that
fails it raises a `DataCorruptionException` ("Invalid index ...").

Every `INSERT INTO t VALUES (…, val, …)` or `UPDATE t SET c = val` against a
constrained column performs an ART lookup (`c = val`) internally, to check
the constraint before the write is allowed to commit. This is worth knowing
because it means an ART index's presence isn't only a `SELECT`-side cost —
inserts and updates against a `PRIMARY KEY`/`UNIQUE`/`FOREIGN KEY` column pay
index-maintenance cost on every write, which the same DiDi material quantifies [11]:
an `UPDATE` that changes 4.4 million indexed keys took **~117 s** with the
index present versus **~7.5 s** after `DROP INDEX` removed it (with some
parallelism gained back, since index maintenance in this checkout is
single-threaded per update). Index design is a real tradeoff, not a free
speed-up.

## 16.2 Where this data structure comes from

The tree structure itself — span-based traversal, adaptively-sized inner
nodes, path compression, lazy expansion — is not a DuckDB invention. It is the
**Adaptive Radix Tree (ART)** from Viktor Leis, Alfons Kemper, and Thomas
Neumann [33]. DuckDB's implementation
(`src/execution/index/art/`) is a from-scratch C++ implementation of that
paper's ideas, adapted to DuckDB's specific storage model (its nodes are
persisted through the same buffer-manager-backed block allocator that stores
table data, via `FixedSizeAllocator`/`FixedSizeBuffer` in
`src/execution/index/fixed_size_allocator.cpp` — an ART index's nodes are not
some separate off-heap structure; they live in ordinary on-disk blocks).

DuckDB does not blindly trust the on-disk form of an index. When
`FixedSizeBuffer::LoadFromDisk` reads a buffer of ART nodes, it checks that the
stored offset and size fit inside one block. If they do not, it throws a
`DataCorruptionException` instead of copying past the end of the block
(`src/execution/index/fixed_size_buffer.cpp:168-175`). It also zeroes the
unused tail of the new buffer. A related check makes sure that an index pointer
points to an allocated segment before following it
(`test/sql/index/art/storage/test_art_corrupt_pointer.test`).

It is worth flagging one likely point of confusion up front, because the same
research group (TUM, Munich) shows up twice in this book for two *completely
unrelated* reasons. Chapter 13 is dedicated to HyPer/Umbra-style **query
compilation** — generating and JIT-compiling machine code per query — which
is also Neumann's work (with Kemper), and which DuckDB explicitly does *not*
do (Chapter 13 proves this exhaustively). ART has nothing to do with query
compilation; it is an **indexing** data structure, orthogonal to whether the
engine that uses it compiles queries or interprets them. DuckDB adopted ART
while remaining firmly in the vectorized-interpreter lineage the rest of this
book documents — don't let "same research group" become "same idea."

## 16.3 Key encoding: turning arbitrary values into comparable bytes

An ART is a tree over **bytes**, not over typed SQL values, and it never
calls a comparison operator at lookup time the way a B-tree or binary search
tree would. Instead, every value that goes into the tree is first mapped to a
byte sequence with one specific property: **plain, unsigned,
byte-by-byte lexicographic comparison of the encoded bytes must produce the
same ordering as SQL's own ordering of the original values.** Get that
mapping right once, up front, and the tree itself never needs to know
anything about the original type again — every node's logic is just "look at
byte number `depth`, branch on its value." (This is the same total-order
byte-encoding idea Chapter 15's sort-key normalization uses to build a flat
sort key for external sorting — applied here not to produce one comparable
byte string per row, but to build the branching structure of a tree.)

`ARTKey::CreateKey` (`src/execution/index/art/art_key.cpp:66–110`) dispatches
on `PhysicalType` to the encoding rule for each type. Here's what each rule
actually does, and why.

### Unsigned integers

No transformation needed beyond fixing the byte order. `Radix::EncodeData`
for an unsigned type is a plain `Store` — the type's natural binary
representation already sorts correctly under unsigned comparison, as long as
the most significant byte comes first (encoded keys are always big-endian;
DuckDB's native in-memory representation is little-endian on essentially all
target platforms, so a byte-swap happens on encode).

### Signed integers: the sign-bit flip

Two's-complement signed integers do **not** sort correctly under plain
unsigned byte comparison, because the sign bit — bit 7 of the top byte — is
set for *negative* numbers and clear for non-negative ones, which is exactly
backwards from what an unsigned comparison wants (it treats a set top bit as
"larger"). The fix is one XOR:

```cpp
// src/include/duckdb/common/radix.hpp:45-47
static inline uint8_t FlipSign(uint8_t key_byte) {
    return key_byte ^ 128;
}
```

```cpp
// src/include/duckdb/common/radix.hpp:166-173
template <class T>
void Radix::EncodeSigned(data_ptr_t dataptr, T value) {
    using UNSIGNED = typename MakeUnsigned<T>::type;
    UNSIGNED bytes;
    Store<T>(value, data_ptr_cast(&bytes));
    Store<UNSIGNED>(BSwapIfLE(bytes), dataptr);
    dataptr[0] = FlipSign(dataptr[0]);
}
```

Concretely, for 8-bit values `1` and `-1`:

```
        two's complement          sign-bit flipped (XOR 0x80)
  1  =  00000001  (0x01)     →     10000001  (0x81)
 -1  =  11111111  (0xff)     →     01111111  (0x7f)
```

Before flipping, `0xff > 0x01` under unsigned comparison, so `-1` would sort
*after* `1` — wrong. After flipping the sign bit of just the top byte,
`0x7f < 0x81`, so `-1` sorts *before* `1` — correct. The intuition: flipping
the sign bit maps the two's-complement range `[-2^(n-1), 2^(n-1)-1]` onto the
unsigned range `[0, 2^n-1]` by shifting every value up by `2^(n-1)` — which is
precisely what's needed to turn "most negative value has the largest raw bit
pattern" into "most negative value has the smallest encoded bit pattern."
`EncodeSigned` only ever flips `dataptr[0]`, the byte holding the sign bit
after the big-endian byte-swap — every lower byte is left untouched, because
those bytes already compare correctly against each other once the sign byte
has decided which "half" of the number line a value falls in.

### IEEE-754 floats: sign flip *plus* a conditional full complement

Floats need one more step than integers, and it's worth being explicit about
why. IEEE-754's bit layout is `[sign][exponent][mantissa]`, and for
**non-negative** floats, larger numeric value means larger *unsigned* bit
pattern — exponent and mantissa both increase monotonically with magnitude,
same as an integer. So for non-negative floats, a sign-bit flip alone (same
trick as signed integers) is enough. But for **negative** floats, magnitude
still *increases* the exponent/mantissa bits (a very negative number has a
large exponent, same as a very positive number would), and the sign bit is
already `1` — so among negative floats, a *larger*-magnitude (i.e., more
negative, i.e., numerically smaller) value has a *larger* raw unsigned bit
pattern. That's exactly backwards: unsigned comparison would sort the most
negative float last among negatives, not first. Flipping only the sign bit
doesn't fix this, because the sign bit is the *only* bit the naive
integer-style fix touches — the exponent/mantissa bits, which are what's
actually reversed, are left alone. The extra step of complementing **every**
bit (`~x`, only for originally-negative values) reverses the relative order
of the exponent/mantissa bits too, which is exactly what's needed to put
negatives back in the right relative order:

```cpp
// src/include/duckdb/common/radix.hpp:49-78 (EncodeFloat; EncodeDouble at
// line 103 is the same algorithm for 64-bit values)
static inline uint32_t EncodeFloat(float x) {
    uint32_t buff;
    // ... zero / NaN / ±infinity special cases elided ...
    buff = Load<uint32_t>(const_data_ptr_cast(&x));
    if ((buff & (1U << 31)) == 0) { //! +0 and positive numbers
        buff |= (1U << 31);
    } else {          //! negative numbers
        buff = ~buff; //! complement 1
    }
    return buff;
}
```

Working this out for the values in DuckDB's own course material
(`material/05/017-encode-float.c` [11], which implements and
self-checks this exact algorithm), π = 3.1415 and e = 2.718:

```
            raw IEEE-754 bits                     encoded (comparable) bits
  π       01000000010010010000111001010110   →   11000000010010010000111001010110
 -π       11000000010010010000111001010110   →   00111111101101101111000110101001
  e       01000000001011011111001110110110   →   11000000001011011111001110110110
 -e       11000000001011011111001110110110   →   00111111110100100000110001001001
```

And the encoded (rightmost column) bit patterns, compared as plain unsigned
32-bit integers, now satisfy every ordering SQL expects:
`e < π` ✓ (`0xc02df3b6 < 0xc0490e56`), `-π < π` ✓, and — the one that
actually exercises the extra complement step — `-π < -e` ✓
(`0x3fb6f1a9 < 0x3fd20c49`, i.e. the more-negative value has the smaller
encoded pattern, as required). Try comparing the two *raw* IEEE-754 bit
patterns for `-π` and `-e` directly and you'll find they come out backwards
(`0xc0490e56 > 0xc02df3b6`, i.e. -π's raw pattern is larger even though -π is
numerically smaller) — which is exactly the bug this second complement step
exists to fix.

### Strings

DuckDB's course material [7] describes string encoding as
using ICU's `ucol_getSortKey()` to map the UTF-8 byte sequence to a
locale-aware, byte-comparable sort key, ending with a NUL terminator. That
claim is worth checking against this checkout's actual code, because it
turns out to be **wrong for this branch** — a good example of the book's own
stated policy that the source wins over teaching material when they
disagree. `ARTKey::CreateARTKey<string_t>` does not touch ICU at all:

```cpp
// src/execution/index/art/art_key.cpp:20-49
template <>
ARTKey ARTKey::CreateARTKey(ArenaAllocator &allocator, string_t value) {
    auto string_data = const_data_ptr_cast(value.GetData());
    auto string_len = value.GetSize();

    // We escape \00 and \01.
    idx_t escape_count = 0;
    for (idx_t i = 0; i < string_len; i++) {
        if (string_data[i] <= 1) {
            escape_count++;
        }
    }

    idx_t key_len = string_len + escape_count + 1;
    auto key_data = allocator.Allocate(key_len);

    // Copy over the data and add escapes.
    idx_t pos = 0;
    for (idx_t i = 0; i < string_len; i++) {
        if (string_data[i] <= 1) {
            // Add escape.
            key_data[pos++] = '\01';
        }
        key_data[pos++] = string_data[i];
    }

    // End with a null-terminator.
    key_data[pos] = '\0';
    return ARTKey(key_data, key_len);
}
```

So the actual encoding is: copy the raw UTF-8 bytes through unchanged (DuckDB
compares strings byte-for-byte, i.e. by Unicode code point order via UTF-8's
own byte-comparable encoding — not by any locale-aware collation), escaping
any literal `0x00` or `0x01` byte in the value by prefixing it with `0x01`
(so a real embedded NUL can never be confused with the terminator, and a real
embedded `0x01` can never be confused with an escape marker), then appending
a `0x00` terminator. (A search across `src/` for `ucol_getSortKey`, `ICU`, or
any collation type confirms zero hits inside `execution/index/art/` — the
ICU-collation claim doesn't apply to string ART keys in this codebase at
all. It's possible the DiDi slide is describing either an older
implementation or a different, collation-sensitive code path elsewhere; on
this checkout's actual `art_key.cpp`, plain byte-order comparison is what
governs ART string ordering.)

The terminator is the important design detail. Without it, one value's
encoding could be a
**literal byte-prefix** of another's. `"AB"` encodes to bytes `[0x41, 0x42]`;
`"ABC"` encodes to `[0x41, 0x42, 0x43]` — the first is a prefix of the
second. In a radix tree, a leaf terminates a path; if `"AB"`'s path is
exactly the first two bytes of `"ABC"`'s path, then the position where
`"AB"`'s value should live is *also* the branching point that leads onward to
`"ABC"` — the tree can't represent both "this is a complete value" and "this
is an internal node with a child" at the same byte position without extra
bookkeeping. Every value must end with a byte that can never occur as a
non-terminal path byte for another value; appending `0x00` (and guaranteeing,
via the escape step above, that `0x00` never appears anywhere else in an
encoded key) achieves exactly that: **prefix-freedom**. No encoded key is
ever a byte-for-byte prefix of another encoded key.

### Composite keys and NULL

Composite (multi-column) keys are built by encoding each column
independently and concatenating the byte sequences — `ARTKey::Concat`
(`src/execution/index/art/art_key.cpp:146-152`) does exactly this, copying
one key's bytes immediately after another's into a single buffer. `NULL`
values widen the encoded representation (an extra leading byte/bit
distinguishing "NULL" from every possible non-NULL encoding) so that NULL
sorts to a single well-defined position without colliding with any real
value's byte sequence.

## 16.4 Span-8 traversal: why ART height depends on *k*, not *n*

An ART doesn't compare whole keys against each other the way a binary search
tree's node does. Instead, it consumes the encoded key **one byte at a
time**: at each level, the next 8 bits of the key select which child to
descend into. This "8 bits per level" choice is called the **span** (denoted
`s` in the ART paper [33]), and DuckDB fixes `s = 8` throughout.

For a key of `k` bits, this gives a tree with exactly `⌈k/8⌉` levels of inner
nodes — independent of how many distinct keys `n` are actually stored. A
32-bit integer key always takes 4 levels to resolve, whether the index holds
10 rows or 10 billion. Both lookup and insert are therefore `O(k)`.

Contrast this with a comparison-based binary search tree over the same `n`
keys: a BST has height `O(log₂ n)`, and each level costs a full key
comparison, which for `k`-bit keys is itself `O(k)` in the worst case (you
may need to look at most or all of the key's bytes to determine which side of
the comparison it falls on). So BST lookup is `O(k · log₂ n)` overall, versus
ART's `O(k)`. The crossover point is exactly where the research notes for
this book state it: ART wins in height once `n > 2^(k/s)` — with `s = 8`,
`n > 2^(k/8)`. For a 32-bit key, `2^(32/8) = 2^4 = 16`; past a mere 16
entries, a span-8 ART is already shallower than a balanced BST would be, and
the gap widens fast as `n` grows further (a BST needs 10¹² entries before
`log₂ n` reaches 40, while the ART's height for a 32-bit key never moves off
4). This is the concrete payoff of "optimize for a large span": bigger `s`
means fewer levels for the same key width, which is why ART chooses the
largest span practical to implement — and DuckDB's choice of a full byte
(`s = 8`) is exactly that: the largest span for which "index a node by the
next chunk of the key" is still a cheap array/table lookup rather than
needing to synthesize a giant fan-out from smaller pieces.

But a bigger span has an obvious problem, which is the entire reason "radix
tree" needed the word "adaptive" bolted onto it.

## 16.5 The four adaptive node types

The naive way to implement span-8 traversal is a fixed array of `2⁸ = 256`
child pointers per inner node — index directly by the next key byte, get
`O(1)` lookup. But most such arrays would be mostly `NULL`: a table with a
few thousand distinct byte-prefixes at some tree level doesn't need 256
slots at every node on that level, and a full 256-pointer array costs
`256 × 8 bytes = 2048 bytes` whether it's 90% full or 90% empty. Doing this
for every inner node of every level would make the tree enormous.

ART's answer is to keep the span fixed at 8 bits (so traversal stays cheap
and height stays `O(k)`) but let each **individual node**'s internal
representation adapt to how many children it actually has, upgrading and
downgrading its representation as children are added and removed. DuckDB
implements exactly the four node sizes from the original paper, chosen by
`NodePtr::GetInternalNodeType`:

```cpp
// src/execution/index/art/node.cpp:353-362
NType NodePtr::GetInternalNodeType(idx_t count) {
    if (count <= Node4::CAPACITY) {
        return NType::NODE_4;
    } else if (count <= Node16::CAPACITY) {
        return NType::NODE_16;
    } else if (count <= Node48::CAPACITY) {
        return NType::NODE_48;
    }
    return NType::NODE_256;
}
```

```
Node4                 Node16                Node48                    Node256
┌───┬───┬───┬───┐      ┌───┬───┬─┈─┬───┐    ┌───────────────┐          ┌──────────────┐
│key│key│key│key│      │key│key│┈┈┈│key│    │  child_index  │          │              │
│ 0 │ 2 │ 3 │255│      │ 0 │ 2 │┈┈┈│255│    │ [256 × uint8] │          │  children     │
├───┼───┼───┼───┤      ├───┼───┼─┈─┼───┤    │  byte → slot  │          │ [256×NodePtr] │
│ptr│ptr│ptr│ptr│      │ptr│ptr│┈┈┈│ptr│    ├───────────────┤          │  direct index  │
│ 0 │ 1 │ 2 │ 3 │      │ 0 │ 1 │┈┈┈│15 │    │   children    │          │  by key byte   │
└───┴───┴───┴───┘      └───┴───┴─┈─┴───┘    │ [48 × NodePtr]│          │               │
 linear scan            binary/SIMD          slot → pointer  │          │               │
 (≤4 children)          scan (5-16)          ├───────────────┤          └──────────────┘
                                              (17-48 children)          (49-256 children)
```

For each type: exact memory layout, lookup algorithm, and why the design
earns its keep.

**Node4** — `key[4]` (an array of up to 4 key bytes, kept in sorted order) and
`children[4]` (the matching child pointers), from `BaseNode<4, NType::NODE_4>`
(`src/include/duckdb/execution/index/art/base_node.hpp:17-33,161-176`). A
lookup is a linear scan of at most 4 bytes — cheap in absolute terms, and
because the array is tiny it typically lives in a single cache line anyway,
so "linear scan" here isn't the liability it would be for a larger array.
Size: `count (1B) + key[4] (4B) + children[4] (4 × 8B = 32B) ≈ 37 bytes`
(the `NodePtr`/`IndexPointer` type is a single 64-bit tagged value —
`src/include/duckdb/execution/index/index_pointer.hpp:38-44` — so every
child pointer costs exactly 8 bytes across all four node types).

**Node16** — same shape as Node4, scaled up: `key[16]` and `children[16]`
(`base_node.hpp:178-196`). DuckDB's course material
(`material/05/018-simd-compare.c` [11]) demonstrates the point of this size class
directly: modern SIMD instruction sets can compare several key bytes against
the target byte in parallel — the example loads 4 keys and the target byte
into 128-bit registers and does the comparison with one `vceqq_s32`/
`_mm_cmpeq_epi32` instruction, rather than 4 separate branches. (The
technique generalizes: 16 one-byte keys fit two 128-bit — or one 256-bit —
SIMD registers, letting the whole array be compared in one or two
instructions instead of up to 16 sequential comparisons; where SIMD isn't
used, a plain binary search over the sorted key array works too, since 16
elements is still small enough that binary search's log₂ 16 = 4 comparisons
beats a full 256-wide direct index.) Size: `1B + 16B + (16 × 8B = 128B) ≈
145 bytes`.

**Node48** — this is where storing keys directly becomes wasteful: searching
even a sorted 48-entry array, or committing to a direct 256-wide array just
to hold 48 pointers, are both worse options. DuckDB's actual struct
(`src/include/duckdb/execution/index/art/node48.hpp:36-39`):

```cpp
private:
    uint8_t count;
    uint8_t child_index[Node256::CAPACITY];  // 256 slots
    NodePtr children[CAPACITY];              // 48 slots
```

Lookup (`Node48::GetChildNode`, `node48.hpp:88-97`) uses the key byte
directly as an index into the 256-entry `child_index` array; the value found
there (or `EMPTY_MARKER = 48` if absent) is itself the index into the
compact 48-slot `children` array. Two dependent array reads, no scanning
and no branching search — but the point of the design is the space this
saves versus a direct 256-pointer array:

```
640 B  =  256 B (child_index: 256 × 1 byte)  +  48 × 8 B (children)
2048 B =  256 × 8 B                                       (a full Node256)
```

**Node256** — for 49–256 children, DuckDB gives up on saving space and just
uses a direct 256-entry pointer array (`src/include/duckdb/execution/index/art/node256.hpp:32-34`):
`uint16_t count; NodePtr children[256];` — lookup is `children[byte]`,
nothing else. Size: `2B + 256 × 8B = 2050 bytes` (DiDi's slides round this to
"2 kB" [7]). Past 48 children, the indirection Node48 uses to save space no
longer pays for itself relative to just committing to the full array, both in
extra memory-access latency (two dependent loads instead of one) and in the
`child_index` array's own 256-byte fixed overhead.

**Growth and shrink thresholds — the "adaptive" part.** A node changes
representation type on insertion/deletion overflow/underflow: a Node4 that
receives its 5th child becomes a Node16 (`Node4::GrowNode16` /
`Node16::GrowNode4`, `base_node.hpp:175,193`); a Node16 growing past 16
becomes a Node48 (`Node16::GrowNode48`); a Node48 growing past 48 becomes a
Node256 (`Node48::GrowNode16`/`ShrinkNode256`, `node48.hpp:158-160`). This
checkout also builds in hysteresis against thrashing at the boundary:
shrinking happens at a *lower* count than growing (`Node48::SHRINK_THRESHOLD
= 12`, `Node256::SHRINK_THRESHOLD = 36` — `node48.hpp:29`, `node256.hpp:25`
— rather than shrinking the instant count drops back to 4/16/48), so a node
whose child count oscillates right around a boundary doesn't flip node types
back and forth on every insert/delete pair.

## 16.6 Leaves: from the textbook picture to what this checkout actually does

DuckDB's course material [7] describes leaves as mirroring the inner-node
adaptive-fanout idea one level down: single-entry leaves for unique values,
and `Leaf7`/`Leaf15`/`Leaf256` multi-entry leaves — holding row IDs instead of
child pointers — for non-unique indexes where one key maps to several row
IDs. That's the textbook ART design, and it's a useful mental model to start
from. The real source in this checkout (`v2.0-cyanoptera`) uses something more
specialized, and it's worth walking through the actual design
rather than just the mirrored-node picture, because it's a nice piece of
engineering in its own right.

`Leaf` (`src/include/duckdb/execution/index/art/leaf.hpp:17-22`) documents
three cases directly in its own header comment:

```cpp
// There are three types of leaves.
// 1. LEAF_INLINED: Inlines a row ID in a NodePtr.
// 2. LEAF: Deprecated. A list of Leaf nodes containing row IDs.
// 3. Nested leaves indicated by gate nodes. If an ART key contains multiple
//    row IDs, then we use the row IDs as keys and create a nested ART
//    behind the gate node. As row IDs are always unique, these nested ARTs
//    never contain duplicates themselves.
```

Case 1 is an optimization that goes further than "a leaf holding one row ID"
— when a key is unique (the overwhelmingly common case for a `PRIMARY
KEY`/`UNIQUE` index, and often true even for non-unique ones), the row ID is
packed **directly into the parent `NodePtr`'s own bit pattern**
(`NodePtr::SetRowId`, `node.hpp:196-202` packs it into bits 8–63 of the
pointer). There is no separate leaf node at all — no extra allocation, no
extra pointer dereference to reach it. Reaching a `LEAF_INLINED` node *is*
having the answer.

Case 3 is the interesting one for non-unique keys: rather than a fixed set
of specialized "hold N row IDs directly" leaf-size classes bolted onto the
tree, DuckDB builds **another, nested ART** rooted at that point, keyed by
the row IDs themselves (which are always unique, so this nested tree never
has to solve the duplicate-key problem again). The parent node's pointer is
marked with a **gate bit** (`NodePtr::AND_GATE = 0x80`,
`node.hpp:118-119,204-219`) to say "what's below here is a nested tree over
row IDs, not a continuation of the outer key." `Node7Leaf`, `Node15Leaf`, and
`Node256Leaf` (`base_leaf.hpp:80-118`, `node256_leaf.hpp`) are the *leaf
level of that nested tree* — they hold, not row IDs directly, but the final
differentiating **byte** of a row ID (since row IDs sharing a byte-prefix are
themselves path-compressed through ordinary `Prefix` nodes, same as any
other ART content). `Node256Leaf` in particular is literally a 256-bit
presence bitmask (`validity_t mask[256 / BITS_PER_VALUE]`,
`node256_leaf.hpp:33-34`) rather than an array of anything — at 256 possible
final bytes, "is this byte present" is cheaper to store as one bit each than
as any kind of array.

This is a genuine divergence from the flatter Leaf7/Leaf15/Leaf256-hold-
row-IDs-directly picture the course slides sketch, and it's worth being
precise about which parts of the "textbook ART" picture survive: the
adaptive small→large fanout idea absolutely does (that's exactly what
`Node7Leaf`/`Node15Leaf`/`Node256Leaf` are — a 3-way, not 4-way, size ladder
at the leaf level, since a leaf byte only ever needs "present or not," never
a further child pointer). What's different is that non-unique keys don't get
a flat array of row IDs at all — they get routed into a second, recursive ART
instance, entered through a gate. It's a more general and more re-usable
mechanism (the same `Node4`/`Node16`/`Node48`/`Node256`/`Prefix` machinery
handles both the outer, value-keyed tree and the inner, row-ID-keyed one)
at the cost of being a level more indirect to explain than "leaves just hold
an array of row IDs."

## 16.7 Lazy expansion and path compression

Two related optimizations reduce tree height and memory further, both aimed
at removing inner nodes that don't earn their keep — most valuable when keys
are long (many possible levels) but the actual key space is sparse (most of
those levels would otherwise have exactly one child).

**Lazy expansion**: don't materialize an inner node at all until two leaves
actually need to be told apart. If a tree currently holds only keys with a
shared prefix `"duck"`, there is no reason to spend four levels of Node4s
just to re-arrive at a single leaf — collapse that whole run into the leaf
itself (or, per §16.6, directly into an inlined row ID) and only *expand*
the path into real inner nodes the moment a second key that shares less of
the prefix (say, one sharing only `"du"`) needs to be inserted.

**Path compression**: separately, remove inner nodes that end up with
exactly one child (which lazy expansion mostly prevents from being created
in the first place, but which can still arise from deletions or from keys
diverging deep into what was previously a single path). There are two
classic variants, differing in what gets stored at the node that survives
the compression and how a mismatch gets detected:

- **Pessimistic**: store the actual skipped byte sequence in the surviving
  node. On every descent through it, compare those stored bytes directly
  against the corresponding slice of the search key. A mismatch is detected
  immediately, at that node, without needing to go any further down the
  tree. Cost: the node needs storage proportional to how many bytes it
  skips.
- **Optimistic**: store only a *count* of how many bytes were skipped, not
  their values. Descent skips that many key-bytes without checking them at
  all, and only compares the full key against the value actually found at
  the eventual leaf — so a key that would have mismatched partway through
  the skipped run isn't caught until the leaf is reached and dereferenced.
  Cost: none in extra per-node storage, but a lookup for a key that doesn't
  exist can pay for a full, wasted descent to a leaf before discovering that.

DuckDB implements the **pessimistic** variant. Each `Prefix` node
(`src/include/duckdb/execution/index/art/prefix.hpp:18-29`) stores up to
`art.PrefixCount()` actual skipped bytes plus a count, chaining multiple
`Prefix` nodes (`Prefix::New`, `prefix.cpp:55-67`) if a compressed run is
longer than one node's capacity — and `art.PrefixCount()` is itself computed
per-index from the combined size of the indexed columns' encoded keys
(`ART::SetPrefixCount`, `src/execution/index/art/art.cpp:1159-1188`), so a
`Prefix` node's byte capacity is tuned to the actual key width rather than
being one universal constant. Verification happens exactly where the
pessimistic scheme predicts — inline, during descent, byte-by-byte, in the
real lookup routine:

```cpp
// src/include/duckdb/execution/index/art/art_operator.hpp:24-58
static OptionalNodePtr Lookup(const ART &art, const NodePtr &node, const ARTKey &key, idx_t depth) {
    NodePtr current(node);

    while (current.HasMetadata()) {
        if (current.IsAnyLeaf() || current.GetGateStatus() == GateStatus::GATE_SET) {
            return current;
        }

        if (current.GetType() == NType::PREFIX) {
            ConstNodeHandle handle(art, current);
            auto data = handle.GetPtr();
            NodePtr child = ConstPrefixHandle::ChildRef(art, handle);
            for (idx_t i = 0; i < data[art.PrefixCount()]; i++) {
                if (data[i] != key[depth]) {
                    return OptionalNodePtr();
                }
                depth++;
            }
            current = child;
            continue;
        }

        D_ASSERT(depth < key.len);
        auto child = current.GetChildNode(art, key[depth]);
        if (!child) {
            return OptionalNodePtr();
        }

        current = child.Get();
        D_ASSERT(current.HasMetadata());
        depth++;
    }

    return OptionalNodePtr();
}
```

Compare this against DiDi's own pseudocode for exactly this algorithm
([7], slide "ART: Index Lookup (Lazy Expansion + Pessimistic Path
Compression)" — the course explicitly teaches the pessimistic variant,
because that's the one this engine implements):

```
search(node, key, depth):
 1  if node = NULL
 2      return NULL                    // search failed (findChild() returned NULL)
 3  if isLeaf(node)                    // reached leaf level?
 4      if leafMatches(node, key)      // reached the proper leaf?
 5          return node                //   yes, success
 6      return NULL                    //   no, failure
 7  p ← byteSequence(node)             // byte sequence in inner node
 8  if p ≠ key[depth…depth+len(p)]     // does partial key match byte sequence?
 9      return NULL
10  depth ← depth + len(p)             // skip to next key byte
11  child ← findChild(node, key[depth]) // key byte selects child
12  return search(child, key, depth+1) // descend into subtree
```

The correspondence is direct: DiDi's lines 7–9 (compare stored bytes `p`
against the key, fail immediately on mismatch) are `ARTOperator::Lookup`'s
`for` loop over the `Prefix` node's `data[]`; DiDi's `isLeaf`/`leafMatches`
is `current.IsAnyLeaf()` plus (for the general case, not shown in the
simplified pseudocode) `LookupInLeaf`'s row-ID comparison
(`art_operator.hpp:63` onward) or the nested-ART gate lookup for non-unique
keys (§16.6); DiDi's `findChild` is `GetChildNode`. Both the optimistic
variant and the classic "single leaf per value" picture remain useful as the
*general* ART design space this engine chose one specific point within — but
on this checkout, "pessimistic path compression" isn't just a description of
one design option, it's a description of the actual code.

## 16.8 Worked example: a primary-key point lookup

Take a query in the same spirit as this book's running examples:

```sql
SELECT * FROM customer WHERE custkey = 12345;
```

Assume `custkey` is declared `PRIMARY KEY`, so an implicit, unique ART index
exists on it, and `custkey` is a 32-bit signed integer.

**Step 1 — encode the search key.** `12345` in binary is `0x00003039`.
Converting to big-endian byte order and flipping the sign bit of the
top byte (§16.3, `Radix::EncodeSigned`) gives:

```
  12345 as int32, big-endian, pre-flip:   00 00 30 39
  sign bit of byte 0 flipped (XOR 0x80):  80 00 30 39
```

So the ART key for this lookup is the 4-byte sequence `[0x80, 0x00, 0x30,
0x39]` — one byte consumed per tree level, 4 levels total for a 32-bit key
(§16.4).

**Step 2 — descend the tree.** The following tree is a small, hand-built
illustration — not a dump of any real running index — meant only to show the
byte-by-byte descent mechanism concretely, including a path-compressed run
and lazy expansion, for a handful of nearby `custkey` values that happen to
share the encoded prefix `0x80 0x00 0x30`:

```
                              root (Node4)
                          key bytes: 0x22, 0x80, 0xA1, 0xFF
                                       │
                          byte 0x80 selects ──┐
                                               ▼
                              Prefix node: skip byte 0x00
                              (every key under this subtree
                               starts with 0x80 0x00 — a
                               path-compressed run of length 1)
                                               │
                                               ▼
                              Node16 (this level has 6 live
                              children under the 0x80 0x00
                              prefix; upgraded from Node4)
                          key bytes: … 0x2F, 0x30, 0x31 …
                                       │
                          byte 0x30 selects ──┐
                                               ▼
                              Node4
                          key bytes: 0x37, 0x39, 0x3A, 0x3D
                                       │
                          byte 0x39 selects ──┐
                                               ▼
                              LEAF_INLINED
                              (custkey is UNIQUE — no separate
                               leaf node; the row ID is packed
                               directly into this NodePtr)
                              row_id = 4,918,204
```

Byte 1 (`0x80`) is resolved at the root's Node4. Byte 2 (`0x00`) doesn't even
need a node lookup — it's absorbed by a single-byte path-compressed `Prefix`
node, verified with one comparison rather than a child-array lookup (this is
the payoff of path compression: an inner node that would otherwise hold
exactly one child, for every `custkey` sharing this prefix, is collapsed
away). Byte 3 (`0x30`) is resolved at a Node16 (this subtree happens to have
enough live children at this level to have grown past a Node4). Byte 4
(`0x39`) is resolved at a final Node4, landing on a `LEAF_INLINED` node —
because `custkey` is a `PRIMARY KEY`, this key is guaranteed unique, so per
§16.6 there's no separate leaf allocation at all; the matching row ID,
`4,918,204`, is unpacked directly out of the `NodePtr`'s own bit pattern via
`GetRowId()`.

**Step 3 — from row ID to row.** The ART's job ends the moment it hands back
a row ID — it has no idea what a "row" looks like, and doesn't need to.
Turning row ID `4,918,204` into the actual `(custkey, name, address, …)`
tuple the query asked for is a job for the storage layer: locate which row
group contains that row ID, then fetch the corresponding value out of each
requested column's data for that row group at the right offset (Chapter 17
covers exactly how row groups, column chunks, and compression schemes make
this single-row fetch work). This is precisely the "jump to one row" cost
§16.1's selectivity discussion was pricing — cheap here, because it's exactly
one such fetch, not forty thousand.

## 16.9 Range and prefix queries

An ART is fundamentally a *sorted* structure — a byte-comparable encoding
plus a tree walked in key order means an in-order traversal of the leaves
visits rows in ascending key order — and DuckDB's `Iterator`
(`src/execution/index/art/iterator.cpp`, `src/include/duckdb/execution/index/art/iterator.hpp`)
exploits exactly that for two predicate shapes beyond plain equality.

**`>=` / `BETWEEN`**: perform an ordinary point-style descent to locate the
*lower bound* (`Iterator::LowerBound`, `iterator.hpp:146`), then scan
leaves left-to-right from there via `Iterator::Scan`
(`iterator.hpp:138-139`), stopping once a leaf's key exceeds the upper bound
(or never, for an unbounded `>`). For:

```sql
SELECT * FROM orders WHERE orderkey BETWEEN 100 AND 200;
```

the index is used to jump straight to the leaf for `orderkey = 100` (or the
first key ≥ 100, if 100 itself isn't present), then the in-order leaf chain
is walked forward, collecting row IDs, until a key greater than 200 is seen.
This is directly exercised by the real test suite: `test/sql/index/art/scan/
test_art_range_scan.test` builds tables specifically sized to force Node48-
and Node256-shaped subtrees (`# node 48` / `# node 256` comments mark inserts
of 19 and 134 keys respectively) and checks `WHERE x > 20` / `WHERE x > 135`
style queries return exactly the expected rows across all four node-type
boundaries — confirming the lower-bound-then-scan mechanism is node-type-
agnostic by construction (every node type implements the same
`GetNextChildNode`/`GetNextChild` "first child ≥ byte" interface used to
resume a scan at any node kind, §16.5).

An index scan first collects the matching row IDs. Then worker tasks fetch the
rows, taking 2,048 row IDs (`STANDARD_VECTOR_SIZE`) at a time. The scan does not
start more threads than there are such batches. It allows one more thread if
the transaction has its own uncommitted rows in local storage
(`DuckIndexScanInitGlobal`, `src/function/table/table_scan.cpp:737-744`). For
example, 5,000 matching row IDs make 3 batches, so at most 3 threads (or 4 with
local rows) work on the scan. Those local rows get their own batch index, after
all the row-ID batches. So they always come out last, which keeps the output
order stable (`test/sql/index/art/scan/test_art_scan_local_rows_batch_index.test`).

**`LIKE 'prefix%'`**: for a query like:

```sql
SELECT * FROM customer WHERE name LIKE 'Smith%';
```

the *fixed prefix* portion of the pattern (`'Smith'`) is used to descend the
tree exactly as a point lookup would, using only those bytes — reaching the
inner node that is the branching point for every key beginning with that
prefix — and then the **entire subtree beneath that node** is scanned,
because every key under it, by construction, shares the prefix. This is a
strictly coarser operation than the `BETWEEN` case: rather than "scan
forward from a lower bound until an upper bound is exceeded" (a linear walk
along the leaf chain), it's "visit every leaf under this one subtree,"
which can be understood as a special case of the same forward-scan
machinery bounded by "still inside the subtree" rather than by an explicit
upper-bound key.

## 16.10 What ART can't do (and why that's a reasonable choice for DuckDB)

DuckDB's own course material [7] is candid about a specific set of gaps, framed
against what an OLTP-oriented engine like PostgreSQL does with its own
B-tree/GiST/GIN indexes:

| Capability | DuckDB ART | PostgreSQL |
|---|---|---|
| Single-column equality/range | Yes | Yes |
| Composite (multi-column) predicate evaluated *by the index itself* | **No** | Yes (multi-column B-tree) |
| Combining two *different* indexed columns' predicates with AND/OR, both via their indexes | **No** | Yes (bitmap index AND/OR) |
| General `LIKE` (not just a literal prefix pattern) | **No** | Yes (GIN/trigram indexes) |
| Accelerating `ORDER BY` off the index's own sorted-leaf order | **No** | Yes |
| Accelerating `MIN`/`MAX` by walking to an extreme leaf | **No** | Yes |

This isn't a claim about some earlier release — it's directly checkable
against this exact `v2.0-cyanoptera` checkout's own source. The scan-pushdown
code that decides whether a `TABLE_SCAN` can use an ART at all says so in its
own comments:

```cpp
// src/function/table/table_scan.cpp:926-940
bool TryScanIndex(const IndexReadHandle<ART> &art, const ColumnList &column_list, TableFunctionInitInput &input,
                  TableFilterSet &filter_set, idx_t max_count, set<row_t> &row_ids) {
    // FIXME: No support for index scans on compound ARTs.
    // See note above on multi-filter support.
    if (art->UnboundExpressionCount() > 1) {
        return false;
    }

    auto index_expr = art->CopyUnboundExpression(0);
    auto indexed_columns = art->GetColumnIds();

    // NOTE: We do not push down multi-column filters, e.g., 42 = a + b.
    if (indexed_columns.size() != 1) {
        return false;
    }
    // ...
```

and only a single `TableFilter` (one column's predicate) is ever handed to
`ART::TryInitializeScan` per scan attempt
(`filter_set.TryGetFilterByColumnIndex`, `table_scan.cpp:970`). The real
test suite confirms the practical consequence directly:
`test/sql/index/art/multi_column/test_art_multi_predicate.test` creates an
index on column `i` only, then runs `WHERE i = 1 AND j = 2` — the query
still returns the right answer, but only because `i = 1` is pushed to the
index scan while `j = 2` is applied afterward as an ordinary residual filter
over whatever rows the index scan returned; the index itself never "sees"
the `j` predicate, and there is no mechanism to intersect two separate
indexes' row-ID sets the way PostgreSQL's bitmap index scan does. `LIKE`
patterns fare similarly: DuckDB's optimizer can rewrite an anchored prefix
pattern into an equivalent range (`o_clerk >= 'Clerk#00005188' AND o_clerk <
'Clerk#00005189'` for `o_clerk LIKE 'Clerk#00005188%'`), which the index
*can* use because it's a range predicate on the single indexed column
(§16.9) — but a general pattern with wildcards in the middle, or without a
fixed literal prefix at all, has no such rewrite and falls back entirely to
sequential scanning.

None of this is a bug to be fixed so much as a reflection of what DuckDB is
*for*. An OLTP engine like PostgreSQL is built around the assumption that a
typical query touches a handful of specific rows, identified by some
combination of predicates across several columns — so it's worth the
engineering cost of composite indexes, bitmap AND/OR across index scans, and
general pattern-matching indexes, because that's where the bulk of its
workload's time goes. An OLAP engine's typical query instead scans and
aggregates a large fraction of a table's rows (this book's own running Query
A — a join, a `GROUP BY`, an `ORDER BY … LIMIT` — never uses an ART index
anywhere in its plan; it relies entirely on zonemaps and vectorized
sequential scanning). Point and range lookups on a single column — the exact
shape ART supports — cover the cases where OLAP workloads *do* benefit from
an index (dimension-table lookups by primary key, selective filters on a
single fact-table column), without paying the implementation and maintenance
cost of a general-purpose, multi-column, pattern-matching index
infrastructure whose benefit, for DuckDB's actual workload mix, would rarely
be worth its cost. §16.1's selectivity math already showed that even the
*one* predicate shape ART fully supports isn't automatically a win — building
out the harder cases for workloads DuckDB isn't primarily designed for would
be solving a problem its users mostly don't have.
