# How DuckDB Works: An Internals Reference

This is a from-source internals reference for DuckDB, built by reading the actual
DuckDB C++ source tree, running/reading its SQL test suite, and cross-checking
everything against Torsten Grust's **DiDi** ("Design and Implementation of DuckDB
Internals") course at the University of Tübingen — the only known academic course
built specifically around DuckDB's own codebase, with direct input from the DuckDB
Labs team. Where DiDi and the source agree, we cite both. Where the source has moved
on since DiDi's slides were written (DuckDB moves fast), the source wins, and the
discrepancy is called out explicitly.

> **Disclaimer: this book was generated with the assistance of AI.** It was written
> from the DuckDB source code, its test suite and
> the sources in [Appendix C](chapters/appendix-c-references.md). Every claim
> carries a citation you can check, but the book may still contain mistakes.
> Where it disagrees with the DuckDB source, trust the source.

## Which DuckDB is this document about?

The source is the DuckDB GitHub repository, https://github.com/duckdb/duckdb, branch
**`v2.0-cyanoptera`**, commit `cb5673685153` (2026-11-12). You can browse that
exact commit at https://github.com/duckdb/duckdb/tree/cb5673685153159d22bf1042ab444273c9ad268a.
`git describe --tags` gives `v2.0.0-1041-gcb567368515`. That means **DuckDB
v2.0.0** (tagged 2026-10-20) plus 1,041 later commits.

**The parser.** This matters most for one chapter. DuckDB removed its old
PostgreSQL-derived parser (`libpg_query`) on 2026-04-20 and merged a
hand-written **native PEG parser** on 2026-04-25 (PR #22194). Releases v1.5.0
to v1.5.6 still use `libpg_query`; v2.0.0 and later use the PEG parser.
**[Chapter 3](chapters/03-parsing.md) documents the PEG parser.**
[Appendix A](chapters/appendix-a-libpg-query.md) documents the old parser for
readers on v1.5.x. Every chapter after parsing describes code that both
parsers feed in the same way.

## How this document is organized

Each chapter is a self-contained Markdown file under [`chapters/`](chapters/). They
build on each other in order (the pipeline is a pipeline), but each is written to be
readable on its own, with its own diagrams, code excerpts, and worked examples.

Diagrams are plain-text/ASCII so they render in any Markdown viewer and stay
byte-identical next to the prose that explains them (no external image files to go
stale). Code excerpts are taken verbatim from the source tree, with file:line
citations, and lightly re-flowed for width only — never semantically altered.

| # | Chapter | Covers |
|---|---------|--------|
| 0 | [Front Matter & How to Read This Book](chapters/00-front-matter.md) | Scope, conventions, the running example queries, source/version notes |
| 1 | [Architecture Overview](chapters/01-architecture-overview.md) | The whole pipeline end to end, one query traced through all seven stages at a glance |
| 2 | [The Query Performance Spectrum](chapters/02-performance-spectrum.md) | Why DuckDB is built the way it is: awk → Python → C → mmap → SIMD → threads → DuckDB, memory hierarchy, `EXPLAIN ANALYZE` |
| 3 | [Parsing](chapters/03-parsing.md) | The native PEG parser: grammar files, tokenizer, packrat matching, transformer pattern, AST types |
| 4 | [Binding & Name Resolution](chapters/04-binding.md) | `Binder`, `BindContext`, `ColumnBinding`, type resolution/casts, the `BoundStatement` |
| 5 | [The Logical Plan](chapters/05-logical-plan.md) | `LogicalOperator`, the full operator taxonomy, the canonical unoptimized plan |
| 6 | [Subquery Decorrelation](chapters/06-subquery-decorrelation.md) | Dependent joins, `FlattenDependentJoins`, Neumann/Kemper "unnesting arbitrary queries" |
| 7 | [The Optimizer](chapters/07-optimizer/00-overview.md) — **expanded, 27-part deep dive** | The fixed pipeline of 45 numbered passes (48 `RunOptimizer` calls) (including the new `SELF_JOIN_AGGREGATE` and `REDUNDANT_ORDER_KEYS` passes) and all 37 expression-rewriter rules, each with its own dedicated, test-grounded chapter part (see the table of contents inside `00-overview.md`) |
| 8 | [Join Ordering](chapters/08-join-ordering.md) | Hypergraphs, DPhyp dynamic programming, cardinality estimation, build/probe side selection, join algorithm selection |
| 9 | [Physical Plan Generation](chapters/09-physical-plan.md) | `PhysicalOperator`, the operator taxonomy, `PhysicalPlanGenerator` |
| 10 | [Vectors and Data Chunks](chapters/10-vectors-and-chunks.md) | `DataChunk`/`Vector`, flat/constant/dictionary/sequence/FSST/shredded representations, unified representation |
| 11 | [Pipelines and Parallelism](chapters/11-pipelines-and-parallelism.md) | `Pipeline`/`MetaPipeline`, Sink/Source/Operator, push vs. the old Volcano pull model, the task scheduler |
| 12 | [Expression Evaluation](chapters/12-expression-evaluation.md) | `ExpressionExecutor`, template-based operator dispatch, SIMD/branchless/unrolling, adaptive filter reordering |
| 13 | [Does DuckDB Compile Queries to Machine Code?](chapters/13-does-duckdb-compile-queries.md) | The definitive answer, with full evidence, and why not — contrast with HyPer/Umbra |
| 14 | [Memory Management & Grouped Aggregation](chapters/14-memory-and-aggregation.md) | The memory pool, streaming vs. spilling, row layout, German strings, hash-based grouped aggregation |
| 15 | [Sorting Large Tables](chapters/15-sorting.md) | Two-phase parallel merge sort, key normalization, Merge Path |
| 16 | [The ART Index](chapters/16-art-index.md) | Zonemaps, Node4/16/48/256, path compression, lazy expansion, range queries |
| 17 | [Storage Engine](chapters/17-storage-engine.md) | Block manager, buffer manager, row groups/column segments, compression schemes |
| 18 | [Transactions & MVCC](chapters/18-transactions-and-mvcc.md) | `DuckTransaction`, undo buffers, WAL, checkpointing |
| 19 | [End-to-End Traces](chapters/19-end-to-end-traces.md) | Several full queries, traced through every stage, plans shown at each step |
| A | [Appendix A: The Historical libpg_query Parser](chapters/appendix-a-libpg-query.md) | The Postgres-derived parser used by releases up to v1.5.6 |
| B | [Appendix B: Glossary](chapters/appendix-b-glossary.md) | Every acronym and term of art used in this book |
| C | [Appendix C: References](chapters/appendix-c-references.md) | Every cited work in IEEE style, with notes on how the book uses each one |

## Sources

All outside works are listed in IEEE style in
[Appendix C: References](chapters/appendix-c-references.md). In short:

- **DuckDB source** [1]: https://github.com/duckdb/duckdb, commit
  `cb5673685153` (branch `v2.0-cyanoptera`).
- **DiDi course** (Torsten Grust, University of Tübingen) [2]:
  eight slide decks [3]–[10]
  and a companion repository [11].
- **DuckDB SQL test suite**: `test/sql/**/*.test` and `test/optimizer/**/*.test`
  in the source tree, used throughout as real, runnable example queries.
- Research notes (kept for provenance, not for reading):
  [`research/duckdb-source-architecture-recon.md`](research/duckdb-source-architecture-recon.md).

## Status

Complete: Chapters 0–19 (Chapter 7 is a 27-part deep dive into the optimizer)
and Appendices A–C. Every `file:line` citation and code excerpt was checked
against commit `cb5673685153`. All outside works are cited with numbered IEEE
references, checked online on 2026-11-13.
