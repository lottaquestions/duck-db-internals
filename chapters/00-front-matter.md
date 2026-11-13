# Chapter 0 — Front Matter & How to Read This Book

> **Disclaimer: this book was generated with Claude.** It was written by Claude,
> an AI model made by Anthropic, working from the DuckDB source code, the DuckDB
> test suite and the sources listed in Appendix C. Claude read and cited the
> source directly, and the citations let you check every claim yourself. Even
> so, it may contain mistakes. Where this book and the DuckDB source disagree,
> trust the source.

## Why this book exists

DuckDB is small enough to read end to end, and popular enough that a lot of people
want to. This book is an attempt to actually do that reading, once, carefully, and
write down what's there — not what the marketing blog posts say is there, not what
a five-year-old paper said was there, but what the C++ source tree says, checked
against the runnable SQL test suite, and checked again against a university course
built specifically to teach DuckDB's own internals.

It is organized around one diagram, which you have probably already seen if you've
read any database textbook:

```
   SQL
    │
    ▼
 Parser
    │
    ▼
 Binder / Planner
    │
    ▼
 Logical Plan
    │
    ▼
 Optimizer
    │
    ▼
 Physical Plan
    │
    ▼
 Execution
```

Every database textbook draws roughly this picture. What's interesting — and what
this book spends most of its pages on — is what each box actually *is* inside
DuckDB specifically, which of the boxes DuckDB draws in an unusual place compared
to other systems (spoiler: DuckDB fuses "Binder" and "Logical Plan" into one step,
and it does most of its subquery-decorrelation work *before* the box labeled
"Optimizer" even starts), and what happens once you cross the line into
"Execution" — which turns out to be the single most misunderstood box in the
entire diagram, because a very well-known and technically excellent line of
research (HyPer, Umbra) does something *completely different* there than DuckDB
does, and it is very easy to assume DuckDB works the way that other, famous line of
research works. It does not. Chapter 13 is entirely dedicated to nailing down,
with exhaustive source-level evidence, exactly what DuckDB does instead and why.

## Sources and how they're weighted

Three kinds of source material went into this book, and they're not weighted
equally:

1. **The DuckDB C++ source tree itself** [1]
   (the GitHub repository https://github.com/duckdb/duckdb, commit `cb5673685153` on branch
   `v2.0-cyanoptera`, browsable at https://github.com/duckdb/duckdb/tree/cb5673685153159d22bf1042ab444273c9ad268a). This is ground truth. Every
   claim in this book that could be checked against the source was checked against
   the source. Where a code excerpt appears, it is copied verbatim (re-flowed for
   page width only) and cited with a `file:line` reference so you can go look at it
   yourself.
2. **The DiDi course** ("Design and Implementation of DuckDB Internals," Torsten
   Grust, University of Tübingen [2]). It has eight slide decks
   [3]–[10] and a
   companion repository of SQL examples [11]. This is the only teaching
   material we're aware of that was built by reading DuckDB's own source, DuckDB's
   own Discord, and by talking directly to the DuckDB Labs team. Where DiDi explains
   *why* a design choice was made, or gives a worked numeric example (timings,
   memory sizes, node byte counts), we use it — and cite the specific chapter and
   slide. Where DiDi's slides describe something that has since changed in the
   source (DuckDB moves fast; the PEG parser rewrite that this book's Chapter 3
   documents may have happened *after* some DiDi slides were written), the source wins and
   the discrepancy is called out.
3. **The DuckDB SQL test suite** (`test/sql/**/*.test`, `test/optimizer/**/*.test`).
   Nearly every worked example query in this book is either lifted directly from a
   `.test` file or was written in the same style and checked against the same
   engine behavior a `.test` file would check. Test names are cited so you can find
   the original and run it yourself with `make unittest` or the built `duckdb` CLI.

A fourth kind of source — general secondary literature about DuckDB (blog posts,
conference talks, Hacker News comments) — deliberately went into *shaping which
questions to ask*, but no claim in this book rests on it alone. If you've read
elsewhere that DuckDB does something, and this book says otherwise, trust this
book's citations and go check the source yourself; that's the whole point of
writing it this way.

## Conventions used throughout

**Diagrams are ASCII/box-drawing, not images.** This keeps them next to the prose
that explains them, keeps them diffable, and means nothing ever goes stale because
an image file got lost. A typical plan diagram looks like this (this is not yet
explained — it's just here to establish the convention):

```
┌─────────────────────────────┐
│ PhysicalResultCollector      │  root — gathers final output
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│ PhysicalTopN                  │  ORDER BY total DESC LIMIT 10
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│ PhysicalHashAggregate          │  GROUP BY o.custkey
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│ PhysicalHashJoin               │  o.orderkey = l.orderkey
└──────┬────────────────┬───────┘
       │                │
┌──────▼──────┐  ┌──────▼──────┐
│ TableScan    │  │ TableScan   │
│ orders       │  │ lineitem    │
└─────────────┘  └─────────────┘
```

Arrows point in the direction data flows *up* through the tree at execution time
(child produces rows, parent consumes them), which is the direction every DuckDB
`EXPLAIN` output uses. Where a diagram needs to show something other than a plan
tree (a memory layout, a state machine, a byte layout), the convention is
explained locally.

**Code excerpts are verbatim.** Every fenced ` ```cpp ` block that isn't explicitly
marked "illustrative" or "pseudocode" is copied from the actual source tree, with
a `path/to/file.cpp:LINE` citation directly above or below it. Long functions are
sometimes elided with `// ...`, but never in a way that changes what the code
does — only unrelated branches, comments, or error-handling boilerplate are cut.

**Every path is relative to the DuckDB repository root.** A citation such as
`src/optimizer/optimizer.cpp:73` means line 73 of that file at commit
`cb5673685153`. To open it on GitHub, put the path after
`https://github.com/duckdb/duckdb/blob/cb5673685153159d22bf1042ab444273c9ad268a/`
and add `#L73` for the line:
<https://github.com/duckdb/duckdb/blob/cb5673685153159d22bf1042ab444273c9ad268a/src/optimizer/optimizer.cpp#L73>.
Test files work the same way.

**Two running example queries** recur across chapters, so you can watch the same
query change shape as it moves through the pipeline instead of re-orienting to a
new query every chapter:

*Query A — the join/aggregate/order/limit query* (used from Chapter 1 onward,
fully traced end to end in Chapter 19):

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

This is a small TPC-H-flavored query (it uses TPC-H's `orders`/`lineitem` schema
and column names, though it is not one of the 22 canonical TPC-H queries) chosen
because it touches almost every stage this book cares about: a scan, a filter, an
equi-join (hash-joinable), a `GROUP BY` (hash aggregation), and an `ORDER BY ...
LIMIT` (which DuckDB's optimizer fuses into a single Top-N operator — see
Chapter 7). It deliberately does *not* contain a correlated subquery, because that
needs its own query and its own chapter.

*Query B — the correlated subquery* (used from Chapter 6 onward):

```sql
SELECT c.name,
       (SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey) AS n_orders
FROM customer c;
```

This is the smallest query that forces DuckDB's planner to build a dependent join
and then decorrelate it — the mechanism covered in depth in Chapter 6.

Where a chapter needs a third example to illustrate something neither query
touches (window functions, `ART` indexes, recursive CTEs, `ASOF JOIN`), it
introduces one locally and says so.

## What version of DuckDB, precisely

This is worth restating from the README because it changes what Chapter 3 looks
like.

The source tree is on branch `v2.0-cyanoptera` at commit `cb5673685153`, dated
2026-11-12. `git describe --tags` names it `v2.0.0-1041-gcb567368515`. In plain
words: DuckDB **v2.0.0** was tagged on 2026-10-20, and this commit is 1,041
commits after that tag. So this book describes v2.0.0 plus about a month of
later fixes. It is a development branch, not a release.

**Which parser runs where.** The biggest architectural change on this branch is
the SQL parser. DuckDB removed its old parser, which was derived from
`libpg_query`, on 2026-04-20 (commit `b791d0b5f74`). It merged a hand-written
native PEG parser on 2026-04-25 (PR #22194, commit `8140496820f`).

The releases split like this:

- **v1.5.0 to v1.5.6** come from a separate 1.5 release branch. They still use
  the `libpg_query` parser. v1.5.6 (2026-09-27) is the last of them.
  Appendix A describes this parser.
- **v2.0.0 and later** use the PEG parser. Chapter 3 describes it. No
  `libpg_query` code is left in the source tree this book uses.

Every chapter
after Chapter 4 (Binder) describes architecture that predates and survives this
change untouched — the parser rewrite only changes what feeds the binder, not
anything the binder, optimizer, physical planner, or execution engine do with it.

## A note on scope and honesty about uncertainty

DuckDB has an enormous surface area — extensions, a Python/R/Java/Node/Go/Rust/WASM
client ecosystem, a SQL dialect with hundreds of functions, a growing lakehouse
story (DuckLake), Arrow interop, and more. This book is scoped deliberately to the
**query-processing core**: how a piece of SQL text becomes rows, and how the
storage and transaction layers underneath make that fast and correct. It does not
cover client-language bindings, the extension-loading mechanism in general (only
the specific optimizer/parser extension hooks that matter to the core pipeline),
or DuckLake. Within that scope, every factual claim is sourced; where the source
was ambiguous or a question genuinely couldn't be answered with confidence (this
happened rarely — noted inline when it does), that uncertainty is stated rather
than papered over with a guess.
