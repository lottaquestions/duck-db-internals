# Chapter 2 — The Query Performance Spectrum

Chapter 1 drew the pipeline every later chapter dissects: parser, binder,
optimizer, physical plan generator, execution engine, storage. Before going
stage by stage, it is worth spending one chapter on a question that pipeline
diagram doesn't answer at all: *why does any of this engineering matter?* A
correct, five-line Python script and DuckDB's C++ execution engine compute
the same answer to the same SQL-shaped question. The only difference between
them is wall-clock time — and, as this chapter's benchmark shows, that
difference spans **three orders of magnitude** for a query as simple as
summing one column. Every mechanism this book documents in detail later
(vectorized batches in Chapter 10, pipelines and parallelism in Chapter 11,
compressed columnar storage in Chapters 14–18) exists to close some specific
piece of that three-orders-of-magnitude gap. This chapter's job is to make
the gap itself vivid and measured, one implementation at a time, before any
of those mechanisms are explained — so that later chapters read as answers to
a question you've already watched be asked.

This chapter leans more heavily on secondary source material than most of the
rest of the book: the worked benchmark, its numbers, and its teaching device
(the same query, implemented seven times, each time faster) are lifted
directly from Torsten Grust's DiDi course — specifically **DiDi Chapter 2,
"The Query Performance Spectrum"** [4]. Those slides credit Thomas
Neumann's TUM lecture "Foundations in Data Engineering" as the origin of the
exercise.
Every number quoted below is Grust's own measurement on his development
machine (a 2023 MacBook Pro, Apple M2 Max, 12 CPU cores), not a number this
book invented or re-measured; where this chapter's own DuckDB output is shown
(Section 2.6), that part *was* generated locally against a live database,
with a citation and a caveat explaining exactly how.

## 2.1 Why this chapter comes before any C++: the memory hierarchy

Every optimization in the rest of this book is, at bottom, an attempt to keep
the CPU working with data that is as close to it as possible, for as long as
possible, and to avoid re-doing work that a previous stage already did. The
reason that goal dominates database engineering — more than almost any other
kind of software engineering — is a set of latency numbers that most
programmers know only vaguely, but that a DBMS designer has to internalize
precisely. DiDi Chapter 3, "Managing Memory + Grouped Aggregation," opens
with exactly this table [5, slide 2], and it is worth borrowing
here because it *is* the reason Chapter 1's pipeline is shaped the way it is:

```
 Memory        Actual Latency     "Human Scale"      Typical Size
 CPU L2 cache        2.8 ns             1 second       ¼ – 16 MB
 RAM               ≈ 100 ns            36 seconds      4 – 256 GB
 SSD             50 – 150 µs           5 – 15 hours    ½ – 4 TB
 HDD                1 – 10 ms         4 – 40 days       1 – 16 TB
```

The "Human Scale" column is the pedagogically useful trick here: it rescales
an L2 cache hit to "1 second" and then asks how long the other levels would
take on that same stretched-out clock. An L2 hit is a second. A RAM access —
one hundred times slower in absolute terms — becomes 36 seconds: you'd have
time to read this paragraph twice. An SSD access becomes five to fifteen
*hours*: an entire workday, gone, waiting for one random read. An HDD access
becomes four to forty *days*. Nothing in ordinary application programming
prepares you for gaps this large, because ordinary application code rarely
sits in a tight enough loop, over enough rows, for the gap to matter. A query
engine's inner loop runs millions to billions of times per query. At that
scale, an accidental extra RAM access per row, multiplied by six million
rows, is the difference between a query that finishes while you're still
looking at the screen and one that doesn't.

This table is also why the seven-box pipeline in Chapter 1 keeps mentioning,
almost in passing, things like "buffered pages," "in-memory residency," and
"projection pushdown." Those aren't incidental features. They are the direct
consequence of one policy, stated explicitly in DiDi Chapter 3 [5]: *keep
persistent data on secondary memory, but during query processing, try to keep
all of the currently relevant ("hot") data in RAM, and in general, try to
process every query as high up this hierarchy as possible.* Chapters 14–18
(storage, buffer management, compression) are the detailed mechanics of that
policy; this chapter is the motivating evidence for why the policy exists at
all.

## 2.2 The floor before any code runs: bandwidth limits

Latency explains why an individual memory access is expensive; bandwidth
explains the *floor* on how fast a query that has to touch a lot of data can
possibly be, no matter how well the code is written. DiDi Chapter 2 states
this floor explicitly, before writing a single line of code, using the same
machine's measured read bandwidths at each level of the storage hierarchy
[4, slide 4]:

```
 Storage (Primary / Secondary)   Read Bandwidth   Time to Read 720 MB
 Networked storage (2.5 GbE)         2.5 GB/s            0.28 s
 External USB-C SSD (2 TB)           800 MB/s            0.90 s
 Internal NVMe SSD (2 TB)              5 GB/s            0.14 s
 DRAM (64 GB)                         21 GB/s            0.03 s
```

720 MB is not an arbitrary number: it is the size of this chapter's benchmark
input (Section 2.3). Divide the file size by the bandwidth of whatever
storage tier holds it, and you get the last column — a query-time floor set
purely by physics, before counting a single CPU cycle spent parsing,
comparing, or adding. The note attached to this slide is the whole point of
the exercise: "we will not reach these limits [because CPU cost is ignored
here] — let us try to get close." Section 2.3 is the story of six
increasingly serious attempts to get close to that floor, and the seventh
attempt (DuckDB) doing something more interesting than "get close" — it
sidesteps the floor entirely, for reasons Section 2.3.7 explains.

## 2.3 One benchmark query, seven implementations

The benchmark [4, slide 3] is deliberately trivial: read the
`lineitem` table from TPC-H scale factor 1 as a raw CSV file — 6+ million
rows, 16 columns, about 720 MB on disk — and sum the integer `quantity`
values in column 5:

```
1|155190|7706|1|17|21168.23|0.04|0.02|N|O|1996-03-13|...⏎
1|67310|7311|2|36|45983.16|0.09|0.06|N|O|1996-04-12|...⏎
1|63700|3701|3|8|13309.60|0.10|0.02|N|O|1996-01-29|...⏎
⋮
[6+ million more rows]
```

Real TPC-H queries are far more complex, but this one line — "sum a column"
— is complex enough to expose every implementation choice this chapter cares
about: how you read bytes off disk, how you find field and record boundaries
inside those bytes, how you convert text to a number, and how you spread the
work across CPU cores. Every implementation below computes the same answer
(153,078,795); only the wall-clock time and the throughput (MB/s or GB/s of
raw CSV processed) change.

### 2.3.1 ➊ awk

`awk` is a line-oriented, pattern-action interpreter: it reads input line by
line, matches each line against a sequence of patterns, and runs the
corresponding `{ ... }` action. The whole query fits in four lines
[4, slide 5]:

```awk
BEGIN { FS = "|"           # delimiter in the CSV is '|'
        sum = 0 }          # matches the first line: reset sum
      { sum = sum + $5 }   # matches every line: add 5th column
END   { print sum }        # matches the last line: print the result
```

```
$ time ./001-sum-quantity.awk lineitem.csv
153078795
        1.58 real     1.43 user     0.14 sys
```

Measured wall-clock time: **≈1.60 s**, a throughput of **471 MB/s**. `time`'s
three numbers matter here and recur for every implementation below: `real`
is elapsed wall-clock time, `user` is time spent executing application code,
`sys` is time spent inside the OS on the process's behalf (system calls).
For `awk`, `user` (1.43 s) accounts for almost all of `real` (1.58 s) — this
implementation is **CPU-bound**, not I/O-bound. It cannot even keep up with
plain SSD read bandwidth (Section 2.2's 800 MB/s–5 GB/s range), which means
the OS's page cache holding the file in DRAM buys `awk` nothing: the
bottleneck is the interpreter re-parsing and re-tokenizing every line, not
the storage device underneath it.

### 2.3.2 ➋ Python

Python is a more general-purpose scripting language, compiled to bytecode
and then interpreted, but the structure is the same as the `awk` script
[4, slide 7]:

```python
total = 0
with open(sys.argv[1], 'r') as file:
    for line in file:
        total = total + int(line.split('|', 5)[4])
print(total)
```

Measured time: **≈2.75 s**, throughput **275 MB/s** — slower than `awk`, and
still purely CPU-bound. Nothing here does anything conceptually different
from the `awk` version; both spend their time in an interpreter loop,
re-splitting and re-parsing the same bytes on every single row.

### 2.3.3 ➌ C, the direct transcription (`getline`)

Moving to a compiled language is the first real architectural change. The
initial C version is a line-by-line transcription of the same logic: read a
line with POSIX `getline(3)`, walk forward through four `|` delimiters with
`strchr(3)`, and convert the text after the fourth delimiter with `atoi(3)`
[4, slide 8]:

```c
while (getline(&line, &linecap, file) > 0) {
    char *delim = line;
    for (column = 1; column < 5; column++) {
        delim = strchr(delim, '|');
        delim++;
    }
    sum = sum + atoi(delim);
}
```

Measured time: **≈0.50 s**, throughput **1.5 GB/s** — a 3× improvement over
`awk` for free, just from removing interpreter overhead. But `perf`/
Instruments profiling [4, slide 9] shows this is *still*
CPU-bound, and shows exactly where the CPU time goes:

```
 C library routine        % of CPU time
 (all other C code)             16 %
 getdelim (≡ getline)           58 %  ███████████▌
 atoi                           14 %  ██▌
 memchr (≡ strchr)              12 %  ██
```

Fifty-eight percent of the time is inside `getdelim`, the function
`getline` calls internally — reading the file one line at a time means one
buffered-I/O call, one buffer-management decision, and one dynamic
reallocation check *per row*, six million times. Reading the file
line-by-line is itself the bottleneck; the fix is to stop doing that and
instead read the whole file into memory with a single system call, then
impose line structure on it in application code.

### 2.3.4 ➍ C with `mmap(2)`

`mmap(2)` maps an entire file directly into the process's virtual address
space with one system call — no buffered reads, no per-line I/O at all
[4, slide 10]:

```c
data = (char *)mmap(NULL, size, PROT_READ, MAP_SHARED, file, 0);
```

```
   lineitem.csv (disk)                    mapped region (DRAM)
  ┌───────────────────┐                  ┌───────────────────┐
  │ 1|155190|...⏎      │  ── mmap(2) ──▶  │ 1|155190|...⏎      │
  │      720 MB file   │                  │  720 MB, page-in   │
  └───────────────────┘                  │  on first touch     │
                                          └───────────────────┘
```

If the file is larger than available DRAM, the OS pages it in on demand;
otherwise, once a page has been touched once, it is resident in the OS page
cache and `mmap`'s pointer refers directly to that cached copy — no copy into
a separate application buffer at all. The catch: with a raw pointer into a
memory block instead of a `FILE*`, `strchr`'s null-terminated-string
assumption no longer applies cleanly across arbitrary field boundaries, so
this first `mmap` version finds `\n` by hand, one byte at a time
[4, slide 11]:

```c
column = 1;
while (data < end) {
    switch (*data) {
    case '|':
        column++;
        break;
    case '\n':
        /* error: line has too few columns */
        break;
    }
    data++;

    if (column < 5)
        continue;                 /* keep scanning for column 5 */

    sum = sum + atoi(data);       /* convert up to the next '|' or '\n' */

    column = 1;                   /* next line starts at column 1 */
    while (*data++ != '\n')
        ;                          /* skip the remainder of this line */
}
```

Measured time depends on whether the OS file-system cache is already warm
[4, slide 12]:

```
 OS file-system cache state    Query time    Throughput
 cold (first read from disk)      1.6 s        471 MB/s
 warm (already cached in DRAM)   0.42 s        1.8 GB/s
```

Cold-cache performance is identical to `awk`'s — unsurprising, since a cold
`mmap` still has to wait on the same disk read `awk` did. But warm-cache
performance already beats the `getline` version by a wide margin: no
per-line system calls, no dynamic buffer reallocation, just a pointer walk
through memory the OS has already staged. Profiling this version, though,
turns up a *new* bottleneck [4, slide 12]:

```
 C code fragment / function     % of CPU time
 (all other C code)                   25.5 %
 atoi                                 21.4 %
 while (*data++ != '\n')              53.1 %
```

Byte-by-byte scanning for `\n` now dominates. Swapping the hand-written loop
for libc's own `strchr(data, '\n')` — which is itself hand-optimized,
typically using exactly the kind of word-at-a-time trick Section 2.3.5
builds from scratch — brings the time down to **0.27 s** (**2.8 GB/s**),
already exceeding the internal-NVMe-SSD floor from Section 2.2's table. The
natural question DiDi poses at this point [4, slide 12] is: *how can `strchr`
be so efficient?* — which motivates writing the same trick out by hand.

### 2.3.5 ➎ C with `mmap(2)` + block-wise newline search

Modern CPUs operate on 64-bit (8-byte) words natively. Instead of asking
"is this byte a `\n`?" six million times, one byte at a time, ask "does this
*8-byte word* contain a `\n` anywhere in it?" and advance by a full word at a
time whenever the answer is no. The core of this is a well-known
bit-twiddling technique — DiDi cites the Stanford *Bit Twiddling Hacks*
collection's section "Determine if a word has a byte equal to n" for it
[4, slide 13, fn. 4], [15] — and it is worth deriving, not just
quoting, because the same "test 8 bytes in one comparison" idea recurs all
over DuckDB's own vectorized execution engine (Chapter 10) in spirit, if not
in this exact form.

The building block is `HAS_ZERO(x)`: given a 64-bit word `x`, does *any* of
its 8 bytes equal `0x00`?

```c
/* HAS_ZERO(x): does the 64-bit word x contain a zero byte anywhere in it? */
#define HAS_ZERO(x) (((x) - 0x0101010101010101ULL) & ~(x) & 0x8080808080808080ULL)
```

The trick has three moving parts, one per sub-term:

- `(x) - 0x0101010101010101`: subtracts 1 from *every byte lane*
  simultaneously (`0x01` repeated across all 8 bytes). If a byte lane
  was `0x00`, this subtraction *borrows* from the next-higher byte, which
  flips that lane's high bit (`0x80`) on — a zero byte always produces a
  byte result with bit 7 set (either `0xFF` from the borrow, or a smaller
  value with bit 7 set, depending on what it borrowed from). Nonzero byte
  lanes may also end up with bit 7 set after subtracting 1 (e.g. `0x80 - 1 =
  0x7F`, no — but `0x81 - 1 = 0x80`, yes), so this term alone is not proof of
  a zero byte; it is only a *necessary* condition.
- `~(x)`: the bitwise complement of the original word. A byte lane that was
  actually `0x00` becomes `0xFF` here — its high bit is set. A byte lane
  that was *not* zero has some bit set in the original, so its complement
  cannot be `0xFF`... but it might still happen to have bit 7 set (e.g.
  `0x7F` complements to `0x80`).
- `& 0x8080808080808080`: masks down to just the high bit of each lane.

The reason `AND`-ing all three terms together works, and neither of the
first two alone would, is that the *only* way a byte lane can have bit 7 set
in **both** `(x - 0x0101...01)` and `~x` simultaneously is if that lane was
originally `0x00`: any other starting value either fails to borrow far
enough in the subtraction, or fails to have its complement's high bit set,
or both. (This is exactly the derivation the Stanford Bit Twiddling Hacks
page [15] gives for "determine if a word has a byte equal to n," specialized to
`n = 0`.) The result has `0x80` in exactly the byte lanes that were zero in
`x`, and `0x00` everywhere else — a ready-made bitmask of *which* lane
matched.

Finding a specific byte other than `0x00` — here, `\n` (`0x0a`) — needs one
more step: XOR the target byte, broadcast across all 8 lanes, into `x`
first. XOR-ing a lane with its own value produces `0x00` in that lane and
something nonzero everywhere else, which turns "find `\n`" into "find a
zero byte" and lets `HAS_ZERO` do the rest:

```c
/* HAS_NL(x): find '\n' (0x0a) in the 64-bit-wide character block x */
#define HAS_NL(x) (HAS_ZERO((x) ^ 0x0a0a0a0a0a0a0a0aULL))
```

Wired into the scanning loop, one word (8 bytes) replaces 8 individual
byte comparisons per iteration (adapted from [4, slides 13–14];
the byte-lane-to-offset lookup below is reconstructed here using a portable
compiler builtin rather than the original slide's unrolled `if`-ladder, for
clarity):

```c
while (data < end) {
    /* ... locate column 5 and sum(atoi(data)) exactly as in 2.3.4 ... */

    const uint64_t *block = (const uint64_t *)data;
    uint64_t hit;
    while (!(hit = HAS_NL(*block)))
        block++;                       /* whole 8-byte word had no '\n' in it */

    data = (const char *)block;
    /* `hit` has 0x80 set in exactly the lane(s) that held '\n'; on a
       little-endian CPU, lane 0 is the least-significant byte, so counting
       trailing zero bits and dividing by 8 gives that lane's byte offset */
    data += __builtin_ctzll(hit) / 8;
    while (*data++ != '\n')
        ;                               /* consume the rest of this line */
}
```

Measured time (warm cache): **≈0.28 s**, throughput **2.8 GB/s** — matching,
not beating, the hand-tuned `strchr` result from Section 2.3.4. That's the
point of the exercise: `strchr` in glibc *already does* essentially this
trick internally (often with wider SIMD registers, not just 64-bit words),
so reimplementing it by hand doesn't buy extra speed — it buys understanding
of *why* `strchr` is fast, and a template for word-at-a-time scanning that
generalizes past "find one specific byte" to the kinds of predicate
evaluation Chapter 10's vectorized `Vector`/`SelectionVector` machinery does
at a larger scale.

### 2.3.6 ➏ C with `mmap(2)` + threads

Every implementation so far has used one CPU core. Grust's machine has 12
(M2 Max). The next step splits the memory-mapped file into 12 partitions —
each boundary snapped to the nearest `\n` so no row is split across two
partitions — and spawns one `pthread` per partition, each independently
summing its own chunk of column 5 [4, slide 16]:

```
              ┌─────────────┐
   thread 1   │ sum_chunk() │───▶ partial sum
              └─────────────┘
                    ⋮                     (12 threads, one per core)
              ┌─────────────┐
main() ──────▶│ sum_chunk() │───▶ partial sum ──▶ pthread_join × 12 ──▶ total
              └─────────────┘
```

The threads share a small, explicit struct — no shared mutable state beyond
what's needed to hand each thread its own slice of the mapped file and get
its answer back [4, slide 17]:

```c
struct chunk {
    pthread_t thread;   /* thread ID */
    char      *data;    /* pointer to chunk start */
    char      *end;     /* pointer to chunk end */
    int       sum;      /* sum of this partition */
};
typedef struct chunk chunk_t;

void *sum_chunk(void *arg)
{
    chunk_t *chunk = (chunk_t *)arg;    /* this thread's argument */
    char *data = chunk->data;
    char *end  = chunk->end;

    /* ... sum this chunk, exactly as in sum-quantity-mmap.c (2.3.4/2.3.5) ... */

    chunk->sum = sum;                   /* write the result back */
    return NULL;
}
```

`main()` calls `pthread_create(..., sum_chunk, &chunk[i])` once per
partition, then `pthread_join(...)` on each, and adds the 12 thread-local
`sum` fields together for the final answer.

Measured time (warm cache, 12 threads): **≈0.04 s**, aggregate throughput
**18.8 GB/s** — a 40× improvement over the single-threaded `mmap`+bit-trick
version, and now within shouting distance of this machine's 21 GB/s DRAM
read bandwidth ceiling from Section 2.2. Profiling shows each thread's
`sum_chunk()` call takes roughly the same time as every other thread's,
which means the file was split into roughly equal-sized partitions — the
single most important condition for parallel speedup to actually scale with
core count, and a preview of exactly the load-balancing problem Chapter 11
covers in depth for DuckDB's own morsel-driven parallelism (splitting a
table's row groups into "morsels" so that every thread gets roughly equal
work, rather than one thread finishing early and idling while another is
still busy).

### 2.3.7 ➐ SQL / DuckDB

The DuckDB CLI version of the same query needs one `read_csv` call and one
`sum()`:

```sql
.timer on
SELECT sum(l_quantity)
FROM   read_csv('lineitem.csv', header = false,
                 names = [..., 'l_quantity', ...]);
```

```
┌──────────────────┐
│ sum(l_quantity)  │
├──────────────────┤
│    153078795     │
└──────────────────┘
Run Time (s): real 0.448   user 3.322090   sys 0.133819
```

Measured time: **≈0.45 s** real — already beating every single-threaded
implementation above, and doing it while parsing a full CSV file from
scratch, with no bespoke bit-trick byte scanning written by hand at all
[4, slide 20]. The `user` time (3.32 s) being more than 7×
larger than `real` (0.45 s) is the tell: DuckDB's CSV reader is itself
multi-threaded, splitting the file into chunks (much like Section 2.3.6's
hand-rolled version) and parsing them in parallel across cores, automatically,
with no `pthread_create` calls written by the query author.

That 0.45 s figure is DuckDB parsing raw CSV text *every single time the
query runs*, which is a fundamentally different task from the numbers
quoted in Section 2.1's framing slide, which asked readers to "understand
how DuckDB achieves 0.002s for our query." Getting to that number requires
one additional step this benchmark's other six implementations never take:
loading the data *once* into DuckDB's own native columnar storage format,
and only then running (and re-running) the aggregate query against it:

```sql
CREATE TABLE lineitem AS SELECT * FROM read_csv('lineitem.csv', ...);
-- (this CREATE TABLE step pays the 0.45s-ish CSV-parsing cost once)

SELECT sum(l_quantity) FROM lineitem;
-- subsequent runs of this query: ≈0.002s
```

It is worth being explicit about why 0.002 s is not "the same 720 MB, just
read faster" — because taken at face value, 720 MB in 0.002 s would be
360 GB/s, over 17× *this machine's own DRAM bandwidth ceiling* from Section
2.2's table, which would be physically impossible. The resolution is that,
once materialized, this query touches nowhere near 720 MB. `lineitem` has 16
columns; the query only needs one of them, `l_quantity`, stored as a typed,
fixed-width integer column. Six million 4-byte integers is about 24 MB — not
720 MB — and DuckDB's optimizer (Chapter 7's `filter_pushdown` and its
sibling passes) has already arranged, at plan time, for the physical
`PhysicalTableScan` to fetch *only* that one column (`Projections:
l_quantity` is exactly the annotation Section 2.6 shows attached to a real
`TABLE_SCAN` node below). Reading 24 MB at anything close to this machine's
21 GB/s DRAM bandwidth, split across the same 12 cores Section 2.3.6 used
by hand, comfortably lands in the low milliseconds — which is exactly what
"0.002 s" is. The three orders of magnitude promised at the top of this
chapter, restated precisely, come from **four independent, additive
effects**, each of which some earlier implementation in this section
achieved in isolation but never all at once:

1. **No re-parsing.** Every non-DuckDB implementation above re-tokenizes and
   re-converts the same text to numbers on every run. Once loaded, DuckDB's
   `l_quantity` column is already a native integer array — parsing happened
   exactly once, at `CREATE TABLE` time, never again.
2. **Projection pushdown.** The query only asks for one of `lineitem`'s 16
   columns. None of the awk/Python/C implementations in this section were
   even given a `lineitem` with 16 columns to begin with — they were handed
   a pre-stripped, `quantity`-only CSV, silently getting this benefit for
   free. A real 16-column table makes the contrast sharper, not smaller.
3. **In-memory residency.** Once loaded, the relevant column lives resident
   in RAM (Section 2.1's "keep hot data at the top of the hierarchy"
   policy), not paged in from disk on demand.
4. **Parallel threads**, exactly as in Section 2.3.6, dividing whatever work
   remains by the number of available cores.

Each of these four effects is the subject of its own later chapter — (1) and
(3) are Chapters 14–18 (storage, compression, buffer management), (2) is
Chapter 7 (optimizer) together with Chapter 9 (physical plan generation),
and (4) is Chapter 11 (pipelines and parallelism) — but seeing all four
named explicitly, against a benchmark you've just watched climb from 1.60 s
to 0.002 s one mechanism at a time, is the reason this chapter exists before
any of them.

## 2.4 Summary: the whole spectrum at a glance

```
 Implementation                          Query Time    Throughput   Bottleneck removed
 awk                                        1.60 s       471 MB/s   —
 Python                                     2.75 s       275 MB/s   —
 C, getline (2.3.3)                        0.50 s       1.5 GB/s   interpreter overhead
 C, mmap (2.3.4, warm cache)                0.42 s       1.8 GB/s   per-line syscalls
 C, mmap + strchr (2.3.4)                   0.27 s       2.8 GB/s   byte-wise '\n' scan
 C, mmap + HAS_NL bit-trick (2.3.5)         0.28 s       2.8 GB/s   (same, done by hand)
 C, mmap + 12 pthreads (2.3.6)              0.04 s      18.8 GB/s   single-core ceiling
 SQL, DuckDB read_csv (2.3.7, cold)        ~0.45 s       ~1.6 GB/s  (parses in parallel
                                                                     automatically)
 SQL, DuckDB, materialized table (2.3.7)   ~0.002 s      n/a *      re-parsing, unwanted
                                                                     columns, disk residency
```

`*` — throughput isn't a meaningful figure for the materialized case, precisely
because the whole point is that it no longer processes 720 MB of anything;
see Section 2.3.7's four-effects breakdown.

Two lessons run through every row of this table, and both recur, by name, in
essentially every later chapter of this book:

- **Removing overhead beats adding cleverness, up to a point.** The jump from
  `awk`/Python to C, and from line-by-line `getline` to a single `mmap`, are
  both cases of *stopping doing unnecessary work* (re-interpreting bytecode,
  making a system call per line) rather than doing the necessary work more
  cleverly. Only once that overhead is gone does clever bit-level scanning
  (2.3.5) or explicit parallelism (2.3.6) have anything left to improve.
- **Bandwidth, not cleverness, is the eventual ceiling — and parallelism is
  the only lever that moves the ceiling itself.** Sections 2.3.4 and 2.3.5
  arrive at the same 2.8 GB/s number by two different routes because both
  are, in the end, one core doing a linear scan over cached memory: they've
  hit the single-core memory-bandwidth wall, not a wall specific to their
  algorithm. The only implementation in this whole progression that beats
  that wall is the one that stops using one core (2.3.6) — or the one that
  stops needing to touch most of the bytes in the file at all (2.3.7,
  materialized).

## 2.5 `EXPLAIN` and `EXPLAIN ANALYZE`

Everything in Sections 2.3–2.4 was measured from *outside* the program, with
`time` and a sampling profiler. SQL query engines carry an equivalent
facility built *into* the engine itself, precisely so a query author doesn't
need `perf` or Instruments to see where a query's time goes. DiDi Chapter 2
closes on exactly this note ("Interlude: SQL EXPLAIN" [4, slide 21]), framing it
as the natural next question after watching wall-clock time swing by 800×
across Section 2.3: *if DuckDB can be this fast, and this slow, on the same
data, how do you tell which one you're getting, for a query you didn't write
seven versions of by hand?*

`EXPLAIN` and `EXPLAIN ANALYZE` are that facility, and they map directly onto
two different stages of Chapter 1's pipeline:

- **`EXPLAIN <query>`** stops after the physical plan generator (Chapter 1,
  stage 4) and renders the resulting `PhysicalOperator` tree — the query is
  never executed, no rows are produced, and no timing information exists yet
  because none was collected. This is a *planning*-time tool: it shows what
  the optimizer decided to do (which join algorithm, whether a filter got
  pushed into a scan, whether `ORDER BY ... LIMIT` fused into a `PhysicalTopN`
  — see Chapter 1 §1.3's walk through exactly this query) without spending
  any time actually doing it.
- **`EXPLAIN ANALYZE <query>`** *does* execute the query — fully, to
  completion — and renders the same physical operator tree, but now every
  node is annotated with what actually happened when the engine ran it: rows
  produced, elapsed time, and (in more detailed profiling modes) memory used
  and bytes scanned.

The mechanism behind `EXPLAIN ANALYZE`'s annotations is not something bolted
on to `EXPLAIN` after the fact — it's the same `OperatorProfiler` machinery
that backs DuckDB's general-purpose profiling pragmas (`PRAGMA
enable_profiling`), reused. Every time a `PipelineExecutor` pushes or pulls a
`DataChunk` through an operator, the call is bracketed by matching
`StartOperator`/`EndOperator` calls that start and stop a per-operator timer:

```cpp
// src/parallel/pipeline_executor.cpp:902-908
void PipelineExecutor::StartOperator(PhysicalOperator &op) {
	context.client.InterruptCheck();
	context.thread.profiler.StartOperator(&op);
}

void PipelineExecutor::EndOperator(PhysicalOperator &op, optional_ptr<DataChunk> chunk) {
	context.thread.profiler.EndOperator(chunk);
```

These bracket every chunk that flows through the pipeline's source, its
intermediate operators, and its sink (`src/parallel/pipeline_executor.cpp`,
lines 197/200 for intermediate operators, 779/782 and 864/880 for the source,
591/598 for the sink). The accumulated numbers are held per operator in an
`OperatorMetrics` struct — one instance per node in the plan, mirrored into a
`ProfilingNode` tree that shadows the `PhysicalOperator` tree one-for-one:

```cpp
// src/include/duckdb/main/profiler/profiling_node.hpp:16-34
struct OperatorMetrics {
	explicit OperatorMetrics() {
		ResetMetrics();
	}

	string name;
	PhysicalOperatorType operator_type;

	double time;
	idx_t elements_returned;
	idx_t intermediate_size_bytes;
	idx_t system_peak_buffer_manager_memory;
	idx_t system_peak_temp_directory_size;
	idx_t rows_scanned;
	idx_t row_groups_scanned;
	idx_t total_row_groups_to_scan;
	idx_t bytes_scanned;
	//! Whether the scan reports bytes_scanned at all - otherwise the metric is left out rather than reported as 0
	bool bytes_scanned_reported;
	// ... (GetMetrics/ResetMetrics/AddExtraInfo/GatherMetrics/Merge follow)
```

The last two fields work as a pair. `bytes_scanned` counts the bytes a scan
read for the query, for scans that know this number (for example the Parquet
reader, which reports the compressed size of the column chunks it scans). For
other scans `bytes_scanned_reported` stays `false`, and the metric is left out
of the profile instead of showing a misleading `0`.

`time` is exactly the "elapsed time" figure printed under each box in the
text-format tree below; `elements_returned` is the row count. Once the query
finishes, this `ProfilingNode` tree is handed to one of seven interchangeable
`TreeRenderer` subclasses under `src/common/tree_renderer/` — `text`, `json`,
`yaml`, `html`, `graphviz`, `mermaid`, `compact` — selected by `EXPLAIN (...,
FORMAT <name>)`. All seven render the *same* annotated tree; they differ only in
output syntax, which the checkout's own test suite exercises directly
(`test/sql/explain/explain_formats.test`, confirming e.g. that `FORMAT json`
always nests children under a `"children"` key and `FORMAT graphviz` always
emits a `digraph G { ... }` block) and (`test/sql/explain/test_explain_analyze.test`,
confirming that `EXPLAIN ANALYZE` and the parenthesized `EXPLAIN (ANALYZE)`
form are synonyms, and that a plain `EXPLAIN ANALYZE SELECT 42;` still prints
something matching `.*Total Time.*` even with no profiling PRAGMA enabled at
all — profiling for `EXPLAIN ANALYZE` specifically is always on, regardless
of the session-wide `enable_profiling` setting).

The `compact` renderer (`src/common/tree_renderer/compact_tree_renderer.cpp`)
prints one operator per line, indented by depth, with the numbers in
parentheses and the operator's properties after them, all in plain ASCII.
Its header comment says it is "Meant for machine consumers (e.g. AI coding
agents) that read the plan as text, where the box-drawing tree is expensive
and the JSON/YAML formats are verbose." The DuckDB CLI uses it by default
for `EXPLAIN` when it detects that a coding agent is driving it. Anyone can
ask for it. `test/sql/explain/explain_formats.test` checks the shape:

```sql
EXPLAIN (FORMAT compact) SELECT * FROM renderer_plan
----
physical_plan	<REGEX>:(?s).*TOP_N \(est=\d+\).*\n +HASH_JOIN \(est=\d+\) Join Type: INNER; Conditions: .*

EXPLAIN ANALYZE (FORMAT compact) SELECT * FROM renderer_plan
----
analyzed_plan	<REGEX>:(?s).*TOP_N \(est=\d+, rows=\d+, time=[0-9.]+s\).*HASH_JOIN.*
```

So a plain `EXPLAIN` line reads like `HASH_JOIN (est=1000) Join Type:
INNER; Conditions: ...`, and under `ANALYZE` the parentheses also hold the
real row count and time.

### 2.5.1 A third kind of `EXPLAIN`: `EXPLAIN (SQL)`

`ExplainType` has a third value, `EXPLAIN_SQL`, next to `EXPLAIN_STANDARD`
and `EXPLAIN_ANALYZE`
(`src/include/duckdb/parser/statement/explain_statement.hpp:16`).

`EXPLAIN (SQL) <query>` does not draw a tree at all. It returns the
optimized plan written back out as SQL text that DuckDB can run:

```sql
-- test/sql/sql_export/explain_sql_scopes.test
EXPLAIN (SQL) SELECT 1;
----
sql	SELECT CAST(1 AS INTEGER) AS "1"
```

Like plain `EXPLAIN`, it does not run the query. Unlike plain `EXPLAIN`, it
stops at the *logical* plan: it runs the binder and the optimizer, then turns
the optimized `LogicalOperator` tree into SQL. The output shows what the
optimizer did, in a language every reader already knows. The binder accepts
it only for `SELECT` (including `VALUES` and `WITH`), `INSERT`, `UPDATE`,
`DELETE` and `MERGE INTO`
(`src/planner/binder/statement/bind_explain.cpp:10-21`). Chapter 5, §5.5.4
shows how each logical operator writes its own part of the SQL.

### 2.5.2 Other profiler features

The profiler behind `EXPLAIN ANALYZE` has a few more features worth knowing:

- **Profiles of failed queries.** A query that fails or is interrupted
  while it runs still writes its profile. In
  `test/sql/profiling/test_failed_query_profile.test`, `SELECT CASE WHEN i
  = 5000 THEN error('boom') END FROM range(10000) t(i)` fails, and the JSON
  profile file still holds the query text and its operators.
- **Live metrics.** The table function `duckdb_live_query_metrics()`
  shows some counters of a query *while it runs*, one row per connection
  and metric (`test/sql/table_function/duckdb_live_query_metrics.test`).
  The live metrics include `io.total_bytes_read`, `io.total_read_operations`,
  `io.total_read_time` and their write twins, and
  `system.total_bytes_spilled`.
- **I/O metrics.** The profile counts I/O operations and I/O time,
  not only bytes, plus bytes spilled to disk and
  `query.total_bytes_scanned`.
- **Thread safety.** Metrics can be updated from several threads at
  once, and set while the query is still being bound. One use: an
  extension that talks to a remote catalog can record how many catalog
  requests a query made.

## 2.6 Query A, under `EXPLAIN ANALYZE`

Chapter 1 traced Query A —

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

— down to the physical plan it becomes (Chapter 1, §1.3):

```
PhysicalResultCollector
  └─ PhysicalTopN
       └─ PhysicalHashAggregate
            └─ PhysicalHashJoin  (build: orders, probe: lineitem)
                 ├─ PhysicalTableScan (orders)
                 └─ PhysicalTableScan (lineitem, pushed-down filter)
```

`EXPLAIN ANALYZE` renders exactly this shape, with every node now carrying
real row counts and real timings. What follows is a live run of this book's
Query A, against synthetic `orders`/`lineitem` tables built to the same
schema and join relationship (100,000 orders, ~320,000 qualifying lineitem
rows after the `shipdate` filter) — generated locally with an installed
stable DuckDB build (v1.1.3), since the `v2.0-cyanoptera` checkout examined
elsewhere in this book was not compiled in this pass. The plan-tree shape,
operator vocabulary, and the profiling mechanism behind it (Section 2.5) are
unchanged between that version and the checkout, so the *shape* shown here
is what the checkout also produces, even though the specific row counts and
timings below are from a smaller, locally generated dataset rather than the
checkout's own build. (One exception is the names of the JSON fields; see
the note before the JSON excerpt below.)

```
┌────────────────────────────────────────────────┐
│┌──────────────────────────────────────────────┐│
││              Total Time: 0.0261s             ││
│└──────────────────────────────────────────────┘│
└────────────────────────────────────────────────┘
┌───────────────────────────┐
│           QUERY           │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│      EXPLAIN_ANALYZE      │
│    ────────────────────   │
│           0 Rows          │
│          (0.00s)          │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│           TOP_N           │  Top: 10
│    ────────────────────   │  Order By: sum(l.extendedprice) DESC
│          10 Rows          │
│          (0.00s)          │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│         PROJECTION        │  __internal_decompress_integral_bigint(#0,0), #1
│         1000 Rows         │
│          (0.00s)          │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│   PERFECT_HASH_GROUP_BY   │  Groups: #0   Aggregates: sum(#1)
│         1000 Rows         │
│          (0.00s)          │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│         PROJECTION        │  custkey, extendedprice
│        320648 Rows        │
│          (0.00s)          │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│         PROJECTION        │  __internal_compress_integral_uinteger(#0,0),
│        320648 Rows        │  __internal_compress_integral_usmallint(#1,0), ...
│          (0.00s)          │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│         HASH_JOIN         │  Join Type: INNER
│    ────────────────────   │  Conditions: orderkey = orderkey
│      Build Min: 0         │
│      Build Max: 99999     ├──────────────┐
│        320648 Rows        │              │
│          (0.02s)          │              │
└─────────────┬─────────────┘              │
┌─────────────┴─────────────┐┌─────────────┴─────────────┐
│         TABLE_SCAN        ││         TABLE_SCAN        │
│           orders          ││          lineitem         │
│  Projections: orderkey,   ││  Projections: orderkey,   │
│    custkey                ││    extendedprice          │
│                           ││  Filters: shipdate <       │
│                           ││    '1998-01-01'::DATE AND  │
│                           ││    shipdate IS NOT NULL    │
│        100000 Rows        ││        320648 Rows        │
│          (0.00s)          ││          (0.00s)          │
└───────────────────────────┘└───────────────────────────┘
```

Every node and label above is unedited from the actual rendered tree; the
inline `extra_info` (join conditions, filters, projections) is reformatted
onto one line per node purely to fit page width, in place of the renderer's
own multi-line layout inside each box.

A few things worth reading directly off this tree, each of which is a
concrete instance of a claim Chapter 1 made about this exact query in the
abstract — plus one thing the tree shows that Chapter 1's hand-drawn diagram
didn't, because it's an implementation detail only `EXPLAIN ANALYZE` (or
reading the physical plan generator's source) would surface:

- **`PhysicalHashAggregate` became `PERFECT_HASH_GROUP_BY`, not
  `HASH_GROUP_BY`.** Chapter 1 mentioned this operator only as
  `PhysicalHashAggregate`; the *specific* physical operator chosen depends on
  what the optimizer's statistics-propagation pass (Chapter 7) can prove
  about the grouping column. `o.custkey`'s range is known and small (the
  `Build Min: 0` / `Build Max: 99999` annotation on the join below already
  hints at this), so DuckDB can guarantee no hash collisions are possible and
  picks the cheaper, collision-free `PERFECT_HASH_GROUP_BY` variant instead of
  general-purpose linear-probing hashing (this exact optimization — and the
  ordinary `HASH_GROUP_BY` fallback when it doesn't apply — is documented in
  [5, slide 13, fn. 5]). Note for readers on the
  `v2.0-cyanoptera` checkout this book is otherwise written from: the
  `Build Min`/`Build Max` annotation shown here is what the older, locally
  installed v1.1.3 build prints; on this repository's actual checkout, the
  equivalent runtime-computed range filter surfaces differently — as a
  `Dynamic Filters` field on the *probe*-side `TABLE_SCAN` rather than on the
  join node itself, and it only appears once the filter has actually proven
  useful. Chapter 7, Part 24 (`JOIN_FILTER_PUSHDOWN`) traces this exact
  mechanism and its `EXPLAIN`-output surface in full depth.
- **The filter really was pushed into the scan.** `lineitem`'s
  `TABLE_SCAN` box carries `Filters: shipdate < '1998-01-01'::DATE AND
  shipdate IS NOT NULL` directly on the leaf — exactly the `filter_pushdown`
  pass Chapter 1 named, made concrete: the predicate is evaluated once,
  during the scan itself, rather than after the join has already multiplied
  `orders` by `lineitem`.
- **Projection pushdown is visible, not just asserted.** Both `TABLE_SCAN`
  nodes list only the columns actually needed (`orderkey`/`custkey` for
  `orders`; `orderkey`/`extendedprice` for `lineitem`) — the same mechanism
  Section 2.3.7 credited for most of DuckDB's headline 0.002 s figure,
  shown here operating on a join query rather than a bare aggregate.
- **Two extra `PROJECTION` nodes appear immediately around the join** that
  neither Chapter 1's diagram nor an `EXPLAIN`-without-`ANALYZE` reading of
  this plan would necessarily draw attention to: one compresses the join
  keys into smaller unsigned integer types (`__internal_compress_integral_
  uinteger`/`usmallint`) just before the `HASH_JOIN`, and one decompresses a
  `BIGINT` back out (`__internal_decompress_integral_bigint`) on the way to
  `TOP_N`. This book doesn't chase down exactly why the physical plan
  generator inserts these two nodes for this particular query until it has
  the vocabulary to (Chapters 10 and 17), but their presence here is a fair
  warning: a real `EXPLAIN ANALYZE` tree is routinely a few nodes busier than
  the simplified plan diagrams this book draws by hand for exposition.
- **The 0.02 s `HASH_JOIN` line is where nearly all of this query's total
  time went** (`Total Time: 0.0261s` at the root) — every other node reports
  `(0.00s)`. This is exactly the kind of information `EXPLAIN` alone (without
  `ANALYZE`) cannot give you: it can tell you a hash join *will* run here, but
  only `ANALYZE`, by actually running the query, can tell you it's the one
  node worth optimizing first.

Requesting `FORMAT JSON` instead of the default text renderer exposes the
same annotations as machine-readable fields. The excerpt below is the v1.1.3
output.

A note on field names. These v1.1.3 names do not match the checkout. The
checkout names its metrics in `src/common/metrics.json`, grouped
by area: the operator metrics are `type`, `name`, `timing`,
`intermediate_rows`, `extra_info`, `rows_scanned`, `bytes_scanned`, and a
few more. (`tracked_metrics` refers to them as `operator.type` and so on.)
The older names shown below (`operator_type`, `operator_timing`,
`operator_cardinality`) are what the checkout still writes when you set the
deprecated `legacy_metrics_format` to `true`. The mapping is in
`src/main/query_profiler.cpp:946-951`:

```cpp
// src/main/query_profiler.cpp:946-951
	emit_as("type", "operator_type");
	emit_as("name", "operator_name");
	emit_as("timing", "operator_timing");
	emit_as("rows_scanned", "operator_rows_scanned");
	emit_as("intermediate_rows", "operator_cardinality");
	emit_as("intermediate_size_bytes", "result_set_size");
```

Next to the operator *type* (for example `TABLE_SCAN`), the checkout's JSON
profile also gives the operator's *name* (for example `SEQ_SCAN`) in the
`name` metric. `test/sql/pragma/profiling/test_profiling_operator_name.test`
checks that `EXPLAIN (ANALYZE, FORMAT json)` output contains
`"name": "SEQ_SCAN"` followed by `"type": "TABLE_SCAN"`.

The v1.1.3 output:

```json
{
    "operator_type": "TOP_N",
    "operator_cardinality": 10,
    "operator_timing": 0.000027542,
    "extra_info": {
        "Top": "10",
        "Order By": "sum(l.extendedprice) DESC"
    },
    "children": [
        {
            "operator_type": "PROJECTION",
            "operator_cardinality": 1000,
            "operator_timing": 0.000003866,
            "extra_info": { "Projections": ["..."] },
            "children": [ { "operator_type": "PERFECT_HASH_GROUP_BY", "...": "..." } ]
        }
    ]
}
```

This JSON is exactly the input format the University of Tübingen's own
**DuckDB Explain Visualizer** [16] (a web-based tool built
by Grust's Database Systems group, adapted from the Postgres-focused `pev2`
project [17]) consumes to draw exactly the same tree as
Section 2.6's box diagram, bottom-up, with each node's box sized and colored
by its row count and its share of total query time — a more legible view of
the same information for plans deep enough that a terminal-rendered ASCII
tree becomes hard to read at a glance. Pasting the output of `EXPLAIN
(ANALYZE, FORMAT JSON) <query>` directly into that tool reproduces the
annotated plan interactively; it adds no information beyond what Section 2.5's
`OperatorMetrics`/`TreeRenderer` machinery already computed and serialized.

## 2.7 Where this goes next

Every lesson this chapter demonstrated with a hand-written C program has a
direct counterpart, built into DuckDB's execution engine by default, that
later chapters document at the source level:

- **"Stop re-parsing, stop touching bytes you don't need"** (Sections 2.3.4,
  2.3.5, 2.3.7) is the batching and typed-storage story: **Chapter 10**
  covers `DataChunk`/`Vector`, the 2,048-row batches that let an operator
  process many rows per function call instead of one, and the typed,
  columnar in-memory representation that means a value is converted from
  text to its native type exactly once, not once per query.
- **"Split the work across cores, with roughly equal partitions"** (Section
  2.3.6) is DuckDB's own parallelism story: **Chapter 11** covers how a
  physical plan is decomposed into `Pipeline`s, how `Pipeline`s are broken
  into `PipelineTask`s and scheduled onto the shared `TaskScheduler`
  introduced in Chapter 1 §1.2, and how DuckDB's morsel-driven approach to
  splitting a table's row groups aims for the same "every thread gets about
  the same amount of work" property Section 2.3.6's hand-rolled 12-thread
  version needed to get right by hand.
- **"Keep hot data at the top of the memory hierarchy, and never write back
  what you didn't change"** (Section 2.1's policy statement) is the storage
  story: **Chapters 14–18** cover the buffer manager, block manager,
  compression schemes, and zonemaps that make it possible for a
  `PhysicalTableScan` to skip entire row groups without reading them, and for
  a materialized table's columns to sit resident in RAM in a form small
  enough (Section 2.3.7's 24 MB, not 720 MB) that "process everything in
  DRAM" is actually achievable for real workloads, not just this chapter's
  toy benchmark.

The `EXPLAIN ANALYZE` tree from Section 2.6 is also a preview of a
methodology, not just a one-off illustration: it is the same tool used
throughout the rest of this book, on the same two running queries, whenever
a later chapter needs to show — rather than merely assert — that some
specific optimizer pass, join algorithm, or storage mechanism actually fired
for a given query. Chapter 19's full end-to-end trace of Query A leans on it
directly.
