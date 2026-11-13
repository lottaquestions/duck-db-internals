# Appendix C — References

This appendix lists every outside work the book cites. It uses the IEEE
reference style.

## How citations work in this book

- A number in square brackets, such as [10], points to entry 10 in the list
  below.
- A locator may follow the number: [10, slide 27] means slide 27 of entry 10,
  and [12, Sec. 3] means section 3 of entry 12.
- Numbers are given in the order that works are first cited, reading the book
  from Chapter 0 to Appendix B. A work keeps its number everywhere after that.
- Online sources show the date they were last checked ("accessed").
- DuckDB source files and test files are **not** in this list. They are cited
  inline as `path/to/file.cpp:LINE`, so you can open them directly. Entry [1]
  names the exact source tree those paths refer to.

All entries were checked online on 2026-11-13. Paper details come from the
publisher's DOI record (via Crossref) or from the paper PDF itself. Slide titles
come from the DiDi course page [2].

## Reference list

[1] DuckDB contributors, *DuckDB*, source code, branch v2.0-cyanoptera, commit cb5673685153 (v2.0.0-1041-gcb567368515), Nov. 12, 2026. [Online]. Available: https://github.com/duckdb/duckdb (accessed Nov. 13, 2026).

[2] T. Grust, "Design and implementation of DuckDB internals," DuckDB Library, Mar. 19, 2026. [Online]. Available: https://duckdb.org/library/design-and-implementation-of-duckdb-internals/ (accessed Nov. 13, 2026).

[3] T. Grust, "Welcome & setup," lecture slides, ch. 1 of *Design and Implementation of DuckDB Internals (DiDi)*, Univ. of Tübingen, Tübingen, Germany, 2026. [Online]. Available: https://blobs.duckdb.org/slides/DiDi-01.pdf (accessed Nov. 13, 2026).

[4] T. Grust, "The query performance spectrum," lecture slides, ch. 2 of *Design and Implementation of DuckDB Internals (DiDi)*, Univ. of Tübingen, Tübingen, Germany, 2026. [Online]. Available: https://blobs.duckdb.org/slides/DiDi-02.pdf (accessed Nov. 13, 2026).

[5] T. Grust, "Managing memory + grouped aggregation," lecture slides, ch. 3 of *Design and Implementation of DuckDB Internals (DiDi)*, Univ. of Tübingen, Tübingen, Germany, 2026. [Online]. Available: https://blobs.duckdb.org/slides/DiDi-03.pdf (accessed Nov. 13, 2026).

[6] T. Grust, "Sorting large tables," lecture slides, ch. 4 of *Design and Implementation of DuckDB Internals (DiDi)*, Univ. of Tübingen, Tübingen, Germany, 2026. [Online]. Available: https://blobs.duckdb.org/slides/DiDi-04.pdf (accessed Nov. 13, 2026).

[7] T. Grust, "The ART of indexing," lecture slides, ch. 5 of *Design and Implementation of DuckDB Internals (DiDi)*, Univ. of Tübingen, Tübingen, Germany, 2026. [Online]. Available: https://blobs.duckdb.org/slides/DiDi-05.pdf (accessed Nov. 13, 2026).

[8] T. Grust, "Query execution plans and pipelining," lecture slides, ch. 6 of *Design and Implementation of DuckDB Internals (DiDi)*, Univ. of Tübingen, Tübingen, Germany, 2026. [Online]. Available: https://blobs.duckdb.org/slides/DiDi-06.pdf (accessed Nov. 13, 2026).

[9] T. Grust, "Vectorized query execution," lecture slides, ch. 7 of *Design and Implementation of DuckDB Internals (DiDi)*, Univ. of Tübingen, Tübingen, Germany, 2026. [Online]. Available: https://blobs.duckdb.org/slides/DiDi-07.pdf (accessed Nov. 13, 2026).

[10] T. Grust, "Query rewriting and optimization," lecture slides, ch. 8 of *Design and Implementation of DuckDB Internals (DiDi)*, Univ. of Tübingen, Tübingen, Germany, 2026. [Online]. Available: https://blobs.duckdb.org/slides/DiDi-08.pdf (accessed Nov. 13, 2026).

[11] Database Systems Research Group, Univ. of Tübingen, "DiDi: Dissecting the duck's innards—A DuckDB-based course on the design and implementation of database system internals," GitHub repository. [Online]. Available: https://github.com/DBatUTuebingen/DiDi (accessed Nov. 13, 2026).

[12] T. Neumann and A. Kemper, "Unnesting arbitrary queries," in *Datenbanksysteme für Business, Technologie und Web (BTW)*, Hamburg, Germany, 2015, *Lecture Notes in Informatics*, vol. P-241. Bonn, Germany: Gesellschaft für Informatik, pp. 383–402.

[13] T. Neumann, "Efficiently compiling efficient query plans for modern hardware," *Proc. VLDB Endow.*, vol. 4, no. 9, pp. 539–550, 2011, doi: 10.14778/2002938.2002940.

[14] T. Neumann and M. J. Freitag, "Umbra: A disk-based system with in-memory performance," in *Proc. 10th Conf. Innovative Data Syst. Res. (CIDR)*, Amsterdam, The Netherlands, 2020. [Online]. Available: https://www.cidrdb.org/cidr2020/papers/p29-neumann-cidr20.pdf

[15] S. E. Anderson, "Bit twiddling hacks," Stanford Univ., Stanford, CA, USA. [Online]. Available: https://graphics.stanford.edu/~seander/bithacks.html (accessed Nov. 13, 2026).

[16] Database Systems Research Group, Univ. of Tübingen, "DuckDB Explain Visualizer (DEV)," GitHub repository. [Online]. Available: https://github.com/DBatUTuebingen/duckdb-explain-visualizer (accessed Nov. 13, 2026). Hosted instance: https://db.cs.uni-tuebingen.de/explain

[17] Dalibo, *PEV2: Postgres Explain Visualizer 2*, source code. [Online]. Available: https://github.com/dalibo/pev2 (accessed Nov. 13, 2026).

[18] B. Ford, "Packrat parsing: Simple, powerful, lazy, linear time," in *Proc. 7th ACM SIGPLAN Int. Conf. Functional Program. (ICFP)*, Pittsburgh, PA, USA, 2002, pp. 36–47, doi: 10.1145/581478.581483.

[19] T. Neumann, "Improving unnesting of complex queries," in *Datenbanksysteme für Business, Technologie und Web (BTW)*, Bamberg, Germany, 2025. Bonn, Germany: Gesellschaft für Informatik, pp. 25–47, doi: 10.18420/BTW2025-01.

[20] G. Moerkotte and T. Neumann, "Dynamic programming strikes back," in *Proc. ACM SIGMOD Int. Conf. Manage. Data (SIGMOD)*, Vancouver, BC, Canada, 2008, pp. 539–552, doi: 10.1145/1376616.1376672.

[21] W. P. Yan and P.-Å. Larson, "Eager aggregation and lazy aggregation," in *Proc. 21st Int. Conf. Very Large Data Bases (VLDB)*, Zurich, Switzerland, 1995, pp. 345–357. [Online]. Available: https://www.vldb.org/conf/1995/P345.PDF

[22] G. Moerkotte, P. Fender, and M. Eich, "On the correct and complete enumeration of the core search space," in *Proc. ACM SIGMOD Int. Conf. Manage. Data (SIGMOD)*, New York, NY, USA, 2013, pp. 493–504, doi: 10.1145/2463676.2465314.

[23] Chair for Database Systems, Tech. Univ. of Munich, "Join ordering," lecture slides, ch. 3 of *Query Optimization*, winter term 2014/15. [Online]. Available: https://db.in.tum.de/teaching/ws1415/queryopt/chapter3.pdf (accessed Nov. 13, 2026).

[24] P. G. Selinger, M. M. Astrahan, D. D. Chamberlin, R. A. Lorie, and T. G. Price, "Access path selection in a relational database management system," in *Proc. ACM SIGMOD Int. Conf. Manage. Data (SIGMOD)*, Boston, MA, USA, 1979, pp. 23–34, doi: 10.1145/582095.582099.

[25] T. Ebergen, "Join order optimization with (almost) no statistics," M.S. thesis, Vrije Universiteit Amsterdam, Amsterdam, The Netherlands, 2022. [Online]. Available: https://blobs.duckdb.org/papers/tom-ebergen-msc-thesis-join-order-optimization-with-almost-no-statistics.pdf

[26] Z. Khayyat *et al.*, "Lightning fast and space efficient inequality joins," *Proc. VLDB Endow.*, vol. 8, no. 13, pp. 2074–2085, 2015, doi: 10.14778/2831360.2831362.

[27] P. Boncz, M. Zukowski, and N. Nes, "MonetDB/X100: Hyper-pipelining query execution," in *Proc. 2nd Biennial Conf. Innovative Data Syst. Res. (CIDR)*, Asilomar, CA, USA, 2005, pp. 225–237. [Online]. Available: https://www.cidrdb.org/cidr2005/papers/P19.pdf

[28] P. Boncz, T. Neumann, and V. Leis, "FSST: Fast random access string compression," *Proc. VLDB Endow.*, vol. 13, no. 12, pp. 2649–2661, 2020, doi: 10.14778/3407790.3407851.

[29] G. Graefe, "Volcano—An extensible and parallel query evaluation system," *IEEE Trans. Knowl. Data Eng.*, vol. 6, no. 1, pp. 120–135, Feb. 1994, doi: 10.1109/69.273032.

[30] M. Raasveldt (Mytherin), "Move to push-based execution model," DuckDB GitHub issue #1583, Apr. 8, 2021. [Online]. Available: https://github.com/duckdb/duckdb/issues/1583 (accessed Nov. 13, 2026).

[31] V. Leis, P. Boncz, A. Kemper, and T. Neumann, "Morsel-driven parallelism: A NUMA-aware query evaluation framework for the many-core age," in *Proc. ACM SIGMOD Int. Conf. Manage. Data (SIGMOD)*, Snowbird, UT, USA, 2014, pp. 743–754, doi: 10.1145/2588555.2610507.

[32] S. Odeh, O. Green, Z. Mwassi, O. Shmueli, and Y. Birk, "Merge path—Parallel merging made simple," in *Proc. IEEE 26th Int. Parallel Distrib. Process. Symp. Workshops PhD Forum (IPDPSW)*, Shanghai, China, 2012, pp. 1611–1618, doi: 10.1109/IPDPSW.2012.202.

[33] V. Leis, A. Kemper, and T. Neumann, "The adaptive radix tree: ARTful indexing for main-memory databases," in *Proc. IEEE 29th Int. Conf. Data Eng. (ICDE)*, Brisbane, QLD, Australia, 2013, pp. 38–49, doi: 10.1109/ICDE.2013.6544812.

[34] T. Pelkonen *et al.*, "Gorilla: A fast, scalable, in-memory time series database," *Proc. VLDB Endow.*, vol. 8, no. 12, pp. 1816–1827, 2015, doi: 10.14778/2824032.2824078.

[35] P. Liakos, K. Papakonstantinopoulou, and Y. Kotidis, "Chimp: Efficient lossless floating point compression for time series databases," *Proc. VLDB Endow.*, vol. 15, no. 11, pp. 3058–3070, 2022, doi: 10.14778/3551793.3551852.

[36] A. Afroozeh, L. X. Kuffo, and P. Boncz, "ALP: Adaptive lossless floating-point compression," *Proc. ACM Manage. Data*, vol. 1, no. 4, pp. 1–26, 2023, doi: 10.1145/3626717.

[37] S. Chambi, D. Lemire, O. Kaser, and R. Godin, "Better bitmap performance with Roaring bitmaps," *Softw.: Pract. Exper.*, vol. 46, no. 5, pp. 709–719, 2016, doi: 10.1002/spe.2325.

[38] pganalyze, *libpg_query: C library for accessing the PostgreSQL parser outside of the server environment*, source code. [Online]. Available: https://github.com/pganalyze/libpg_query (accessed Nov. 13, 2026).

## Notes on the sources

These notes say why each main source matters to the book, and where it is used.

**DuckDB source code [1].** This is ground truth for every claim.
This book uses commit `cb5673685153` (2026-11-12) on branch
`v2.0-cyanoptera`. `git describe --tags` gives `v2.0.0-1041-gcb567368515`. If a `file:line`
citation looks wrong against a newer checkout, diff against `cb5673685153` first. The
SQL tests in `test/sql/` and `test/optimizer/` are part of this source tree.
Most worked SQL examples in the book come from a `.test` file there.

**The DiDi course [2].** Torsten Grust's course at the University
of Tübingen. It was built by reading DuckDB's source and by talking to the
DuckDB team. The book uses its eight slide decks as teaching material:

- Deck 1, "Welcome & setup" [3]: course setup.
- Deck 2, "The query performance spectrum" [4]: main source for
  Chapter 2.
- Deck 3, "Managing memory + grouped aggregation" [5]: main source for
  Chapter 14.
- Deck 4, "Sorting large tables" [6]: main source for Chapter 15.
- Deck 5, "The ART of indexing" [7]: main source for Chapter 16.
- Deck 6, "Query execution plans and pipelining" [8]: main source for
  Chapter 11.
- Deck 7, "Vectorized query execution" [9]: main source for Chapter 13,
  also used in Chapters 10 and 12.
- Deck 8, "Query rewriting and optimization" [10]: main source for
  Chapter 6, also used in Chapters 7 and 8.

The companion repository [11] holds the SQL examples that go with the
slides. When a slide and the source code disagree, the book follows the source
and says so.

**Query execution models.** Graefe's Volcano paper [29] is the
classic "pull" model that Chapter 11 contrasts with DuckDB's "push" pipelines.
Boncz, Zukowski and Nes [27] introduced vectorized execution, the
model DuckDB follows (Chapters 10 and 13). Neumann [13]
introduced compiling each query to machine code, the model DuckDB does not
follow (Chapter 13). Umbra [14] is a later compiling system.
Leis et al. [31] named "morsel-driven parallelism"; Chapter 11
shows that DuckDB does similar work but does not use that name.

**Query optimization.** Selinger et al. [24] is the classic
dynamic-programming join orderer. Moerkotte and Neumann [20]
give the DPhyp algorithm that DuckDB implements (Chapter 8). Moerkotte, Fender
and Eich [22] define which reorderings of outer joins
are valid; a DuckDB test cites one of its figures (Chapter 7, Part 26). The TUM
lecture slides [23] cover join ordering for teaching. Ebergen's
thesis [25] explains DuckDB's cardinality estimator. Neumann
and Kemper [12] describe the subquery unnesting method that
Chapter 6 traces in the source; Neumann's later paper
[19] extends it. Yan and Larson [21] describe
eager aggregation, the idea behind partial aggregate pushdown (Chapter 7,
Part 12).

**Operators and data structures.** Khayyat et al. [26] describe
the IE join for inequality conditions (Chapter 8). Odeh et al.
[32] describe Merge Path, used for parallel merging in sorting
(Chapter 15). Leis, Kemper and Neumann [33] define the Adaptive Radix
Tree (Chapter 16). Chambi et al. [37] describe Roaring bitmaps
(Chapter 17).

**Compression (Chapter 17).** FSST [28] compresses strings. Gorilla
[34] and Chimp [35] compress floating-point
time series by XOR with an earlier value. ALP [36] compresses
floating-point numbers by turning them into integers. Patas is DuckDB's own
variant in the Gorilla/Chimp family; the book knows of no paper for it.

**Parsing.** Ford [18] defines packrat parsing, the memoized PEG
parsing that DuckDB's new parser uses (Chapter 3). `libpg_query`
[38] is the PostgreSQL-derived parser library that DuckDB used up to
v1.5.x (Appendix A).

**Tools and other material.** The DuckDB Explain Visualizer
[16] draws query plans in a browser. It is adapted from PEV2
[17] (Chapter 2). "Bit twiddling hacks" [15] is the source
of a byte-search trick in the DiDi performance example (Chapter 2). DuckDB issue
#1583 [30] is where DuckDB's move to push-based execution was
planned (Chapter 11).
