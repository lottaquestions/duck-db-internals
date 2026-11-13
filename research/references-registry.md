# Reference registry (working file, not part of the book)

> **Status (2026-11-13):** the chapters now use final IEEE numbers, not keys. The
> numbered list is in `chapters/appendix-c-references.md` (copy in
> `references-numbered.md`), and the verified entries are in `references-ieee.md`.
> To add a source later: add its verified entry to the end of the list in
> Appendix C with the next free number, and cite it by that number.

Chapters cite sources with a **key** written as `[@key]`, for example
`... as Neumann shows [@neumann2011compiling].` A slide or page goes after the
key: `[@didi06, slide 9]`. A final pass replaces every key with its IEEE number
(`[1]`, `[1, slide 9]`), numbered in order of first use from Chapter 0 to the last
appendix, and writes the reference chapter.

To cite a source that is not listed here, add an entry to
`research/refs-pending/<your-chapter-id>.md` with: the key you used, authors,
title, venue/publisher, year, volume/issue/pages if known, URL and DOI if known.
Do not invent details you could not see.

Do NOT cite the DuckDB source tree or its test files with keys. Those keep their
inline `path/to/file.cpp:LINE` form. The source tree as a whole has a key
(`duckdb-source`) for the places that cite "the DuckDB source code" as a work.

## Academic papers

| key | details (to be verified and set in IEEE style by the final pass) |
|---|---|
| moerkotte2008dpstrikes | G. Moerkotte, T. Neumann, "Dynamic programming strikes back," SIGMOD 2008 |
| neumann2015unnesting | T. Neumann, A. Kemper, "Unnesting arbitrary queries," BTW 2015, Hamburg |
| neumann2011compiling | T. Neumann, "Efficiently compiling efficient query plans for modern hardware," PVLDB 4(9):539–550, 2011 |
| boncz2005x100 | P. Boncz, M. Zukowski, N. Nes, "MonetDB/X100: Hyper-pipelining query execution," CIDR 2005 |
| leis2013art | V. Leis, A. Kemper, T. Neumann, "The adaptive radix tree: ARTful indexing for main-memory databases," ICDE 2013 |
| leis2014morsel | V. Leis, P. Boncz, A. Kemper, T. Neumann, "Morsel-driven parallelism: A NUMA-aware query evaluation framework for the many-core age," SIGMOD 2014 |
| graefe1994volcano | G. Graefe, "Volcano—An extensible and parallel query evaluation system," IEEE TKDE 6(1), 1994 |
| selinger1979access | P. G. Selinger et al., "Access path selection in a relational database management system," SIGMOD 1979 |
| khayyat2015iejoin | Z. Khayyat et al., "Lightning fast and space efficient inequality joins," PVLDB 8(13), 2015 |
| boncz2020fsst | P. Boncz, T. Neumann, V. Leis, "FSST: Fast random access string compression," PVLDB 13(11), 2020 |
| pelkonen2015gorilla | T. Pelkonen et al., "Gorilla: A fast, scalable, in-memory time series database," PVLDB 8(12), 2015 |
| liakos2022chimp | P. Liakos, K. Papakonstantinopoulou, Y. Kotidis, "Chimp: Efficient lossless floating point compression for time series databases," PVLDB 15(11), 2022 |
| afroozeh2023alp | A. Afroozeh, L. X. Kuffó, P. Boncz, "ALP: Adaptive lossless floating-point compression," Proc. ACM Manag. Data 1(4), 2023 |
| neumann2020umbra | T. Neumann, M. Freitag, "Umbra: A disk-based system with in-memory performance," CIDR 2020 (source of "German strings") |
| raasveldt2019duckdb | M. Raasveldt, H. Mühleisen, "DuckDB: An embeddable analytical database," SIGMOD 2019 (demo) |
| odeh2012mergepath | S. Odeh, O. Green, Z. Mwassi, O. Shmueli, Y. Birk, "Merge path — Parallel merging made simple," IPDPS Workshops 2012 |

## DiDi course (Torsten Grust, University of Tübingen)

| key | details |
|---|---|
| didi01 | DiDi chapter 1 slides, https://blobs.duckdb.org/slides/DiDi-01.pdf |
| didi02 | DiDi chapter 2 slides, "The Query Performance Spectrum", https://blobs.duckdb.org/slides/DiDi-02.pdf |
| didi03 | DiDi chapter 3 slides, "Managing Memory + Grouped Aggregation", https://blobs.duckdb.org/slides/DiDi-03.pdf |
| didi04 | DiDi chapter 4 slides, Sorting, https://blobs.duckdb.org/slides/DiDi-04.pdf |
| didi05 | DiDi chapter 5 slides, ART index, https://blobs.duckdb.org/slides/DiDi-05.pdf |
| didi06 | DiDi chapter 6 slides, Pipelining, https://blobs.duckdb.org/slides/DiDi-06.pdf |
| didi07 | DiDi chapter 7 slides, "Vectorized Query Execution", https://blobs.duckdb.org/slides/DiDi-07.pdf |
| didi08 | DiDi chapter 8 slides, "Query Rewriting and Optimization", https://blobs.duckdb.org/slides/DiDi-08.pdf |
| didi-repo | DiDi companion repository, https://github.com/DBatUTuebingen/DiDi |
| didi-library | DuckDB library page for the course, https://duckdb.org/library/design-and-implementation-of-duckdb-internals/ |
| explain-visualizer | DuckDB Explain Visualizer, https://github.com/DBatUTuebingen/duckdb-explain-visualizer |

## Software and other sources

| key | details |
|---|---|
| duckdb-source | DuckDB source code, https://github.com/duckdb/duckdb, branch v2.0-cyanoptera, commit used by this edition (see Chapter 0) |
| libpg-query | pganalyze, libpg_query, https://github.com/pganalyze/libpg_query |
| tum-queryopt-ch3 | TUM query optimization lecture notes, chapter 3, https://db.in.tum.de/teaching/ws1415/queryopt/chapter3.pdf |
