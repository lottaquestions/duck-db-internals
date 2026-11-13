# Appendix C — Bibliography

This appendix collects, in one place, every external source the preceding
chapters cite: academic papers, the DiDi course material, the DuckDB source
tree and SQL test suite, and everything else. Nothing here is new — every
claim below is traceable to a specific place in Chapters 1–18 — this is a
compilation pass, not a fresh literature search. Two housekeeping notes
before the list itself:

- **Chapter 19** ("end-to-end traces," per the front matter's own description)
  does not yet exist in `chapters/` as of this writing — only Chapters 0
  through 18 are present. Nothing below is drawn from it.
- **Appendix A**, which Chapter 3 and the front matter both forward-reference
  for the historical `libpg_query` parser architecture, likewise does not yet
  exist as a file. The `libpg_query` citation in §C.4 is assembled from what
  Chapter 3 already says about it, so that it's ready and waiting when
  Appendix A is actually written.

Entries are grouped into four sections: academic papers, DiDi course
material, DuckDB source-and-test citation conventions, and other external
sources. Within each section, entries are ordered by the chapter that first
introduces them.

## C.1 Academic Papers

**Guido Moerkotte and Thomas Neumann. "Dynamic Programming Strikes Back."
*Proceedings of the 2008 ACM SIGMOD International Conference on Management of
Data* (SIGMOD 2008).**
Introduces the DPccp/DPhyp reformulation of dynamic-programming join
ordering as an enumeration over *connected subgraphs* of the query's join
hypergraph, rather than a blind enumeration of all `2ⁿ` relation subsets —
the paper that makes exhaustive DP-based join ordering tractable again for
non-trivial join graphs. Used in **Chapter 8** (§8.4 onward), which
identifies `src/optimizer/join_order/plan_enumerator.cpp` as a direct,
source-cited implementation of this algorithm (the source file's own comment
names the paper). Referenced again in **Chapter 6** (§6.9), purely to
distinguish it from Neumann & Kemper's 2015 paper below — both are
Neumann co-authored optimizer papers, but they solve different problems
(join ordering vs. dependent-join decorrelation), and Chapter 6 is careful
not to conflate them.

**Thomas Neumann and Alfons Kemper. "Unnesting Arbitrary Queries."
*Datenbanksysteme für Business, Technologie und Web* (BTW 2015), Hamburg,
Germany.**
The paper behind DuckDB's dependent-join decorrelation machinery: represent
a correlated subquery as a dependent-join operator over a duplicate-free
parameter domain, then push that operator down through projection,
aggregation, and selection (via a small set of algebraic equivalences) until
it can be replaced by an ordinary join. Used as the **primary source for
Chapter 6** (Subquery Decorrelation), which argues at length (§6.9) that
`FlattenDependentJoins` is a direct implementation of this paper's technique,
not merely "inspired by" it, and cites DuckDB's own source comment
(`src/planner/subquery/flatten_dependent_join.cpp`'s enclosing directory) as
corroboration. Also referenced in **Chapter 1** (§1, architecture overview)
and **Chapter 9** (§9, physical planning) wherever the book needs to name the
specific mechanism DuckDB uses to avoid re-evaluating a correlated subquery
per outer row.

**Thomas Neumann. "Efficiently Compiling Efficient Query Plans for Modern
Hardware." *Proceedings of the VLDB Endowment*, 4(9):539–550, 2011.**
Introduces the produce/consume code-generation model that underlies HyPer
and Umbra's query-compilation architecture — generate and JIT-compile
machine code per query, rather than interpreting a plan tree of virtual-call
operators. Used as the **primary source for Chapter 13** ("Does DuckDB
Compile Queries?"), whose entire argument is that DuckDB deliberately does
*not* implement this model, despite sharing the same research lineage
(morsel-driven parallelism, vectorization) with the systems that do. Also
referenced in **Chapter 1**, where the book's opening pipeline diagram flags
query compilation as the alternative architecture DuckDB's execution stage
could have chosen and didn't.

**Peter Boncz, Marcin Zukowski, and Niels Nes. "MonetDB/X100: Hyper-Pipelining
Query Execution." *Proceedings of the 2005 Conference on Innovative Data
Systems Research* (CIDR 2005).**
The paper that introduced the vectorized/batched execution model — process
data in fixed-size batches of a few thousand values per column, using tight,
type-specialized primitive loops, without going as far as generating new
machine code per query. Used as the **primary source for Chapter 13**,
which places DuckDB squarely in this tradition (rather than the
produce/consume compiling tradition above) as its central thesis. Also
referenced in **Chapter 10** (Vectors and Chunks), which credits this paper's
lineage (X100/VectorWise, and MonetDB before it) for the general idea of a
`DataChunk` as a horizontal slice of columns processed together.

**Viktor Leis, Alfons Kemper, and Thomas Neumann. "The Adaptive Radix Tree:
ARTful Indexing for Main-Memory Databases." *Proceedings of the 29th IEEE
International Conference on Data Engineering* (ICDE 2013).**
The paper defining the Adaptive Radix Tree (ART) data structure: span-based
traversal, adaptively-sized inner nodes (Node4/16/48/256), path compression,
and lazy expansion. Used as the **primary source for Chapter 16** (The ART
Index), which treats `src/execution/index/art/` as a from-scratch C++
implementation of this paper's ideas, adapted to DuckDB's own
buffer-manager-backed, on-disk block storage model rather than a pure
in-memory structure. Chapter 16 (§16.2) is explicit that this is a
*completely different* piece of TU München research from the
produce/consume compilation work above, despite sharing two authors
(Neumann, Kemper) with it — ART is an indexing structure, orthogonal to
whether the engine that uses it compiles or interprets queries.

**Viktor Leis, Peter Boncz, Alfons Kemper, and Thomas Neumann. "Morsel-Driven
Parallelism: A NUMA-Aware Query Evaluation Framework for the Many-Core Age."
*Proceedings of the 2014 ACM SIGMOD International Conference on Management
of Data* (SIGMOD 2014).**
The paper that coined "morsel-driven parallelism" for HyPer's scheme of
handing out disjoint, unclaimed units of work to whichever thread asks next,
rather than statically pre-partitioning data across a fixed set of threads.
Used in **Chapter 11** (§11.8, "A terminology aside") specifically to make a
negative point: the mechanism this paper names is functionally close to what
DuckDB's own `GetPartitionData`/`SourceParallelism`/`HasSourceTasks`
machinery does, but the term "morsel" itself does not appear anywhere in
DuckDB's source (`src/execution`, `src/parallel`) — the chapter verifies this
by direct search and treats "morsel-driven parallelism" as an outside label
applied to DuckDB by comparison, not DuckDB's own vocabulary for itself.

**Goetz Graefe. "Volcano — An Extensible and Parallel Query Evaluation
System." *IEEE Transactions on Knowledge and Data Engineering*, 6(1), 1994.**
The canonical statement of the pull-based iterator/Volcano execution model
(`Next()`/`GetChunk()` calls flowing from parent to child, data flowing back
up through return values) that most database textbooks and PostgreSQL's own
executor implement. Used in **Chapter 11** (§11.1) as the explicit baseline
DuckDB's own push-based pipeline-driver design is contrasted against — the
chapter frames DuckDB's choice as one of two historically available answers
to "what actually calls these operators," with Volcano being the more
famous, PostgreSQL-descended one DuckDB deliberately did not pick.

**P. G. Selinger, M. M. Astrahan, D. D. Chamberlin, R. A. Lorie, and T. G.
Price. "Access Path Selection in a Relational Database Management System."
*Proceedings of the 1979 ACM SIGMOD International Conference on Management
of Data* (SIGMOD 1979).**
The System R optimizer paper establishing the classical bottom-up,
subset-enumerating dynamic-programming approach to join ordering. Used in
**Chapter 8** (§8.4) purely as the historical baseline that Moerkotte &
Neumann's DPhyp paper (above) improves on — Chapter 8 uses it to explain
precisely what "strikes back" in "Dynamic Programming Strikes Back" is
responding to (DP itself wasn't the part that didn't scale; blind subset
enumeration was).

**Zuhair Khayyat et al. "Lightning Fast and Space Efficient Inequality
Joins." *Proceedings of the VLDB Endowment*, 8(11), 2015.** (Cited in-text as
"Lightning-fast inequality joins.")
Introduces the IE-join algorithm: sort both join inputs by one of two
inequality predicates, then sweep a second, permutation-ordered pass over the
other predicate to find qualifying pairs without a full nested-loop scan.
Used in **Chapter 8** (§8.8) to explain what `PhysicalIEJoin`/
`PhysicalRangeJoin` implement, and why DuckDB's join-order optimizer routes a
predicate list with at least two independent inequalities (and no equality)
to this operator instead of the hash-join workhorse.

## C.2 DiDi Course Material

All eight decks are by **Torsten Grust**, Database Systems group, University
of Tübingen, from the course *"Design and Implementation of DuckDB
Internals"* (DiDi). Slides are hosted at `blobs.duckdb.org/slides/DiDi-0{1..8}.pdf`;
the companion GitHub repository, which holds worked SQL examples and
supporting material referenced alongside individual slides, is
`github.com/DBatUTuebingen/DiDi`. The book's front matter (Chapter 0) and
several chapters independently confirm that this course was built by reading
DuckDB's own source, its Discord, and by talking directly with the DuckDB
Labs team — which is why it's treated as primary teaching material rather
than secondary commentary throughout this book, and why, per Chapter 0's own
weighting policy, the source tree still wins whenever a DiDi slide and the
current source tree disagree (this happens at least once — see the ART
string-collation discrepancy noted under DiDi-05 below).

Chapters cite these decks under a mix of naming styles — some give an
explicit on-slide chapter title, others cite only a chapter number or slide
number. The list below reconciles that into one consistent numbering; where
no chapter cited an explicit title for a given deck, that's stated rather
than invented.

- **DiDi-01.** Not cited by any of Chapters 1–18 in this book. No title or
  content can be reconciled from in-text evidence.
- **DiDi-02, "The Query Performance Spectrum."** Explicitly titled in-text.
  Used as the **primary source for Chapter 2**, which lifts its entire
  worked benchmark (the same query implemented seven times, each faster than
  the last) directly from this deck, measured on Grust's own development
  machine (2023 MacBook Pro, Apple M2 Max). Chapter 2 also notes this deck
  itself credits Thomas Neumann's TUM lecture "Foundations in Data
  Engineering" as the exercise's origin.
- **DiDi-03, "Managing Memory + Grouped Aggregation."** Explicitly titled
  in-text. Used as the **primary source for Chapter 14**, cited extensively
  by slide number throughout (spill triggers, row-major intermediate layout,
  radix-partitioned out-of-core hashing). Also cited once by **Chapter 2**
  (§2.1), which borrows this deck's memory-hierarchy latency table (slide 02)
  to motivate why the rest of the book cares about cache locality at all.
- **DiDi-04, Sorting.** No explicit on-slide chapter title is quoted by the
  citing chapter. Used as the **primary source for Chapter 15**, which
  redraws its two-phase (partition, then merge) external-sort diagram from
  this deck's slide 04 and quotes its stated design goal verbatim: *"DuckDB
  implements a variant of a two-phase merge sort strategy that utilizes all
  available CPU cores in both phases."*
- **DiDi-05, The ART Index.** No explicit on-slide chapter title is quoted by
  the citing chapter; Chapter 16 identifies it only as "the DiDi course
  material... chapter 5" and cites its companion repo file
  `material/05/016-art.sql` directly for a 150-million-row TPC-H benchmark.
  Used as the **primary source for Chapter 16** for ART lookup pseudocode,
  node-size figures (DiDi rounds Node256's byte size to "2 kB"), and
  quantified insert/update overhead numbers. Chapter 16 (§16.7) also flags
  one place where this deck and the current source tree diverge: DiDi
  describes ICU-collation-aware string key encoding for ART, but a direct
  search of `execution/index/art/` in this book's checkout finds no ICU
  collation code path at all — the chapter treats this as a case where either
  an older DuckDB version or a different code path is being described, and
  defers to the source over the slide, per Chapter 0's stated source-weighting
  policy.
- **DiDi-06, Pipelining.** No explicit on-slide chapter title is quoted by the
  citing chapter. Used as the **primary source for Chapter 11**, cited
  repeatedly by slide number for the pipeline-driver sketch, the
  build/probe-side-selection heuristic ("DuckDB chooses the smaller input to
  be on the `BUILD` side," slide 9), and the generic root sink DiDi calls
  `QUERY` (slide 8).
- **DiDi-07, "Vectorized Query Execution."** Explicitly titled in-text (also
  once informally described as covering "Compiling Tight Loops" and
  "Avoiding Branch Mispredictions" as named sub-sections/slides within it).
  Used as the **primary source for Chapter 13**, most notably for a
  verbatim-checked footnote distinguishing DuckDB's vectorized-interpreter
  approach from a "different breed of DBMS — for example Umbra," which
  compiles at query-translation time. Also cited by **Chapter 12**
  (Expression Evaluation, for its "Mixed-Mode Selection" slide) and
  **Chapter 10** (Vectors and Chunks, for vector sizing).
- **DiDi-08, "Query Rewriting and Optimization."** Explicitly titled in-text.
  Used as the **primary source for Chapter 6** (dependent-join
  decorrelation — the deck's pages 26–34 are cited directly, including its
  own algebraic notation for the dependent-join operator `D`), and drawn on
  again, unnumbered but by evident subject match, in **Chapter 8**'s
  join-ordering material ("DiDi's join-ordering slide deck," the
  build-side-selection heuristic).

**Companion tool (not part of the numbered deck sequence):** the
**DuckDB Explain Visualizer**, `github.com/DBatUTuebingen/duckdb-explain-visualizer`,
a web-based `EXPLAIN (FORMAT JSON)` plan-tree renderer built by Grust's group
(hosted at `db.cs.uni-tuebingen.de/explain`), adapted from the
Postgres-focused `pev2` project. Cited in **Chapter 2** (§2.6) as a more
legible alternative to a terminal-rendered ASCII plan tree for large plans.

## C.3 DuckDB Source & Tests

**Source tree.** Every source-level claim, file excerpt, and `file:line`
citation in this book is checked against one specific checkout:
`/home/nick.othieno/Github/duckdb`, branch `v2.0-cyanoptera`, commit
`d8a1bd4f4fbccdf3d22d8f1fe27a94c11c805d0a` (confirmed directly against that
checkout via `git rev-parse HEAD` while preparing this appendix; the front
matter's own short-form `d8a1bd4f4f` is this same commit). This is a
pre-release development branch, roughly 14,600 commits past the last stable
release line (`v1.5.x`), and — per Chapter 0 and Chapter 3 — its parser was
rewritten from a `libpg_query`-derived grammar to a hand-written PEG parser
one day before this commit was taken. This book does not repeat every
individual `file:line` citation here; there are hundreds, scattered
throughout Chapters 1–18, and they belong next to the prose they support, not
in a reference list. What belongs here is the anchor: if a citation in this
book ever looks stale against a newer DuckDB checkout, this is the exact
commit to diff against first.

**Research notes.** The raw research notes behind this book — the working
document assembled while first reading the source tree, before any chapter
prose was written — live at
`research/duckdb-source-architecture-recon.md`. Chapter 18 cites it directly
at least once (its own §7b–7c, for `DuckTransaction`/`UndoBuffer` MVCC
machinery); it is the provenance trail for material in this book that isn't
independently re-derived from a source excerpt inline, and is the right place
to look if a claim in the finished chapters needs to be traced back to *why*
it was believed true in the first place.

**SQL test suite.** DuckDB's own test suite (`test/sql/**/*.test`,
`test/optimizer/**/*.test`) is cited throughout by path, in the convention
`test/sql/<area>/<subarea>/<test_name>.test` — for example
`test/sql/subquery/scalar/test_unnest_subquery.test` (Chapter 6),
`test/sql/index/art/multi_column/test_art_multi_predicate.test` (Chapter 16),
`test/sql/catalog/did_you_mean.test` (Chapter 4), and
`test/sql/explain/explain_formats.test` (Chapter 2). Nearly every worked SQL
example in this book is either lifted directly from a `.test` file at that
path or written in the same style and checked against the same behavior a
`.test` file would check, specifically so a reader can go find the original
and run it. Any test cited this way can be run locally, against the same
checkout named above, with `make unittest` or the built `duckdb` CLI's own
test runner — no separate tooling beyond what ships in the DuckDB source tree
itself is required.

## C.4 Other External Sources

**libpg_query** (vendored fork; upstream project maintained by Lukas Fittl /
pganalyze as a standalone transpiler of PostgreSQL's own bison/flex grammar
into a freestanding C parsing library — this attribution is supplied here for
reference; Chapter 3's own text identifies the tool only generically, as "a
vendored fork of `libpg_query`... that transpiles PostgreSQL's bison/flex
grammar into a standalone C library," without naming the maintaining
organization). This is the parser architecture every released DuckDB version
through `v1.5.x` actually runs, and which this book's Chapter 3 explicitly
sets aside in favor of documenting the PEG parser that replaced it one commit
before this book's checkout was taken. Chapter 3 and the front matter both
forward-reference **Appendix A** as the place this architecture will be
documented in full; as noted at the top of this appendix, that file does not
yet exist in this tree.

**PostgreSQL's own grammar/parser**, referenced only informally and
comparatively in **Chapter 3** (§3.1) — as the LALR, bison/flex-based
architecture that both `libpg_query` and Postgres itself use, and that
DuckDB's PEG parser deliberately departs from. No specific PostgreSQL
documentation page or version is cited by name in the chapter text, so none
is invented here.

**Floating-point and string compression schemes named, but not fully cited,
in Chapter 17** (Storage Engine):

- **FSST (Fast Static Symbol Table).** Chapter 17 (§17.4.4) describes it as
  "DuckDB's own string-compression scheme (co-designed by DuckDB Labs and
  published as a VLDB paper)" but gives no author names, title, or year — the
  chapter cites the *fact* of publication, not the paper itself. No full
  citation is given in-text, so none is fabricated here.
- **Gorilla.** Chapter 17 (§17.4.6) attributes the XOR-against-reference-value
  compression idea to "Facebook's Gorilla paper" by name only — again, no
  author, title, or year appears in the chapter text.
- **Chimp.** Described in the same section as "a later refinement" of
  Gorilla, also named only, with no full citation given.
- **Patas.** Described as DuckDB's own variant within the same general
  Gorilla/Chimp family; the chapter explicitly declines to assert anything
  about a specific external paper for Patas at all — it is presented purely
  as source-derived (`patas.cpp`), not as an implementation of any named
  external publication.
- **ALP (Adaptive Lossless floating-Point compression).** Notably different
  from the three schemes above: Chapter 17 (§17.4.7) describes ALP entirely
  from its own algorithm header and source comments, and — unlike its
  treatment of FSST, Gorilla, and Chimp — never uses the words "paper" or
  "published" in connection with it at all. Whatever academic lineage ALP
  has, this book's own text doesn't invoke one, so this bibliography doesn't
  either.
