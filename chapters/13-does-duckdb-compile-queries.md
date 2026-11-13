# Chapter 13 — Does DuckDB Compile Queries to Machine Code?

## 13.1 A reasonable question, and why it's reasonable

Ask anyone who follows modern OLAP engine design what makes a query engine
fast, and somewhere in the answer you will hear the word "compiled." Since
Thomas Neumann's 2011 VLDB paper, *"Efficiently Compiling Efficient Query
Plans for Modern Hardware"* [13], a whole line of systems has taken
the position that the fastest way to execute a SQL query is to first turn it
into a program, specifically for that query, and then run the program. Not
"interpret a generic plan tree that happens to represent this query" — *compile*
that plan into LLVM IR or straight machine code, link it into the running
process, and jump to it. Neumann's paper introduced the **produce/consume**
model for doing this: each operator in a query plan contributes a fragment of
code to a single generated function, chained together so that a tuple moves
from the bottom of the plan to the top without ever leaving a CPU register if
it doesn't have to. HyPer, the system Neumann built this idea into at TUM, and
Umbra [14], the successor system from the same research group, both compile every
query this way, as a matter of course, before running it.

This is not an obscure corner of database research. It is one of the most
influential ideas in query engine design of the last fifteen years, and it is
the reason "does this engine compile queries to machine code?" has become a
completely standard question to ask about any new, fast, research-adjacent
database system. DuckDB gets asked this question constantly, and for a
specific, structural reason that goes beyond "it's fast and modern": **DuckDB's
own execution model is a direct descendant of the same 2011 lineage.** Chapter
11 covers this in full, but the short version matters here — DuckDB decomposes
a physical plan into `Pipeline`s, breaks each pipeline into `PipelineTask`s,
and pushes `DataChunk`s through a chain of operators from source to sink,
scheduled dynamically across worker threads by a shared `TaskScheduler`. That
push-based, pipelined, parallel execution model — including the "pipeline
breaker" concept for anything that has to fully materialize before its
consumer can start (a hash join's build side, a sort, a final aggregation) —
is architecturally the same idea Neumann's 2011 paper and the morsel-driven
parallelism work that followed it (Leis, Boncz, Kemper and Neumann
[31]) popularized.

So the honest starting position for this chapter is: DuckDB and HyPer/Umbra
share an architectural grandparent. It would be *unsurprising*, not
far-fetched, if DuckDB also compiled queries to machine code — that is exactly
what its academic cousins do, and DuckDB borrowed the other half of their
design. The job of this chapter is to nail down, with the same level of
source-level rigor as every other chapter in this book, exactly which half of
that inheritance DuckDB kept and which half it deliberately declined — because
getting this exactly right is the single most common point of confusion a
reader coming from the HyPer/Umbra world will have about DuckDB, and it is a
completely reasonable point of confusion to have.

## 13.2 The answer, stated plainly, up front

**No.** DuckDB does not compile SQL queries into machine code, LLVM IR, or any
other form of query-specific generated code, at any stage of query processing.
There is no code generator anywhere in its execution path, no JIT compiler
linked into the binary, and no vendored compiler-construction library of any
kind in its dependency tree. DuckDB is a **vectorized, tree-walking,
bytecode-free interpreter**. Every SQL query it ever runs is executed by the
same fixed set of C++ functions that shipped inside the `duckdb` binary the
day it was built — a query never causes new executable code to come into
existence.

That is the claim. The rest of this chapter exists to prove it exhaustively —
first by evidence of absence (Section 13.3), then by ruling out two things
that look like exceptions but aren't (Section 13.4), then by evidence of
presence: showing concretely what DuckDB does instead, and why that is not
merely "the same idea in different clothes" (Section 13.5). Section 13.6
brings in Torsten Grust's own framing of this exact question. Section 13.7
turns the comparison into a real engineering tradeoff, not just a scorecard.
Section 13.8 places both traditions in their correct academic lineage. Section
13.9 closes with a summary built to be quoted on its own.

## 13.3 The exhaustive negative evidence

If DuckDB generated and compiled code at query time, that machinery would have
to live somewhere in `src/`, and it would have to leave a paper trail: either
a call into a JIT compiler library, or hand-rolled machine-code emission, or
at the very least a source directory with a name like `codegen/` or `jit/`.
None of that exists. The following greps were run against the source tree
(`v2.0-cyanoptera`, commit `cb5673685153`, which is after the `v2.0.0`
release). Each grep
covers the *entire* `src/` tree — parser, planner, optimizer, execution, and parallel
scheduling all included, not just the execution engine, in case query-specific
code generation were somehow triggered earlier in the pipeline:

```
$ grep -rniI "\bjit\b" --include=*.cpp --include=*.hpp src
→ 0 hits

$ grep -rniI "asmjit" --include=*.cpp --include=*.hpp src
→ 0 hits

$ grep -rniI "codegen|code.gen|code generation" --include=*.cpp --include=*.hpp src
→ 1 hit, and it is a comment, not code:
    src/include/duckdb/common/sorting/sort_key.hpp:36:
    //! amount of code generated for sorting, merging, scattering/gathering and the range joins.

$ grep -rniI "computed goto|&&label|goto \*" --include=*.cpp --include=*.hpp src
→ 0 hits

$ grep -rniI "llvm" --include=*.cpp --include=*.hpp src
→ 3 hits, all incidental:
    src/include/duckdb/common/thread_annotation.hpp:11
        (a documentation link to Clang's thread-safety-analysis docs)
    src/include/duckdb/common/compatible_with_ipp.hpp:12-13
        (a comment noting a shared_ptr implementation was adapted from
         llvm-project's libcxx — borrowed STL-adjacent code, unrelated
         to query execution)
```

Each of these five patterns was chosen because it corresponds to a distinct,
plausible way a compiling query engine would announce itself in source, and
each is worth taking one at a time:

- **`jit` / `asmjit`.** The most direct possible signal. A system that JIT-compiles
  query plans either rolls its own JIT (HyPer targets LLVM's JIT infrastructure;
  a lighter-weight alternative some systems use is the `asmjit` library, a
  small x86/ARM machine-code assembler meant exactly for building JITs) or at
  minimum uses the word "JIT" in comments explaining what it's doing. Zero
  hits for either pattern, anywhere in `src/`.
- **`codegen` / "code generation."** The generic term for the whole technique,
  independent of which backend a system targets. The single hit that does
  turn up is a comment in `sort_key.hpp` (§17 covers this file's actual job:
  packing sort keys into a compact binary layout) worrying about *how much C++
  template instantiation the sort-key machinery produces when the compiler
  builds DuckDB* — i.e., ordinary, ahead-of-time C++ compilation of DuckDB's
  own source, the same sense in which any large templated C++ codebase
  "generates a lot of code." This is not runtime codegen. It is precisely the
  DBMS-build-time specialization Section 13.5 characterizes properly.
- **`computed goto` / `&&label` / `goto \*`.** This one deserves its own aside,
  because it is the pattern most likely to produce a false negative if you
  don't know what you're looking for. "Computed goto" (GCC/Clang's
  `&&label`-address-and-`goto *ptr` extension) is the technique bytecode
  interpreters — CPython's main loop since 3.11, Ruby's YARV, many Lua
  implementations — use to replace a `switch`-based dispatch loop with direct
  jumps between opcode handlers, because it lets the branch predictor learn
  per-opcode transition patterns instead of always predicting from one shared
  `switch` dispatch site. It is worth naming explicitly here because it is
  *itself still an interpreter technique*, not a compiler — a computed-goto
  interpreter still walks a fixed program (bytecode, or in DuckDB's case a
  bound expression tree) with pre-existing handler code; it just dispatches
  between handlers more cheaply than a `switch` does. If this grep had turned
  up hits, it would not have meant DuckDB compiles queries — it would only
  have meant DuckDB's interpreter dispatch is a slightly more advanced flavor
  of interpreter dispatch. It didn't turn up hits: as Section 13.5 shows,
  DuckDB's dispatch is a plain `switch`, the most ordinary interpreter-dispatch
  mechanism there is, with no computed-goto optimization layered on top of it
  either.
- **`llvm`.** The most direct possible signal for the HyPer-style approach
  specifically, since HyPer compiles to LLVM IR and JITs it via LLVM's own
  MCJIT/ORC infrastructure. All three hits are unrelated to query execution —
  one is a documentation URL in a comment, two are attribution comments for a
  `shared_ptr` implementation borrowed from LLVM's own C++ standard library
  (`libcxx`), which is ordinary open-source code reuse, not evidence of an
  LLVM dependency used for compilation.

Circumstantial but strong supporting evidence comes from what the checkout
*doesn't* vendor. `third_party/` is where DuckDB bundles third-party code it
depends on but doesn't want to require the host system to provide. A compiling
query engine needs a compiler-construction backend somewhere in its dependency
graph — LLVM (HyPer), a custom lightweight code generator (Umbra, see Section
13.8), or at minimum something like `libgccjit` or `asmjit`. The actual
contents of `third_party/` in this checkout:

```
brotli          fastpforlib     jaro_winkler   pdqsort       thrift
catch           fmt             jemalloc       re2           utf8proc
concurrentqueue fsst            lz4            ska_sort      vergesort
fast_float      httplib         mbedtls        skiplist      yyjson
                hyperloglog     miniz          snappy        zstd
                imdb                           parquet
                                pcg            tdigest
```

Not one compiler-construction library among them. What *is* there sorts
cleanly into a handful of categories, and every one of them is consistent with
"vectorized interpreter with excellent low-level engineering," none consistent
with "compiling query engine": compression codecs (`brotli`, `lz4`, `snappy`,
`zstd`, `miniz`, `fastpforlib`, `fsst` — the storage layer's compression
schemes, Chapter 17); data-interchange formats (`parquet`, `thrift`, `yyjson`);
a general-purpose allocator (`jemalloc`) and a lock-free concurrent queue for
the task scheduler (`concurrentqueue`); string/number utilities (`fmt`,
`re2`, `utf8proc`, `fast_float`, `jaro_winkler`); approximate-aggregate
sketches (`hyperloglog`, `tdigest`); sort algorithms (`pdqsort`, `ska_sort`,
`vergesort`); a fast PRNG (`pcg`); and a test framework (`catch`). If DuckDB
generated and compiled query-specific code, *something* in this list would
need to be a code-generation backend. Nothing is.

There is also no source directory that would house such a thing — no
`src/execution/jit/`, no `src/execution/codegen/`, nothing analogous to what
you'd expect to find in HyPer's or Umbra's own source trees. The absence of a
place for the machinery to live is as telling as the absence of the machinery
itself.

One hit *outside* `src/` is worth a sentence, so that a reader who widens the
grep is not misled. `extension/extension_build_tools.cmake:188` has the comment
"Resolve archive LLVM IR into native code for non-LTO linkers." This is part
of building DuckDB, not running it. With link-time optimization (LTO), the
C++ compiler leaves LLVM IR in the object files of a static library. That
CMake step turns the IR into normal machine code once, after the library is
built (`duckdb_make_native_lto_archive`, lines 189-211). It is the same
compile-once-at-build-time work as every other compiler step. It runs on the
build machine, and it never sees a SQL query.

## 13.4 Two things that look like exceptions, and aren't

Two pieces of vocabulary in DuckDB's own source and its own teaching material
are easy to mis-read as evidence *for* runtime code generation. Both are worth
addressing directly, because a careless skim of either is exactly the kind of
thing that manufactures a false "actually, DuckDB *does* compile queries"
conclusion.

**`adaptive_filter.hpp`.** DuckDB genuinely has a feature called "adaptive"
execution, and it genuinely lives in `src/include/duckdb/execution/`. Read
past the name, though, and it is not code generation in any sense:

```cpp
// duckdb/execution/adaptive_filter.hpp
struct AdaptiveFilterState {
    TimePoint monotonic_start;
};
enum class AdaptiveFilterSource : uint8_t { INITIAL, SEEDED };
```

`AdaptiveFilter` is a small piece of runtime bookkeeping that reorders the
conjuncts of a multi-clause `WHERE` predicate (`a > 10 AND b < 100 AND ...`)
into cheapest-first order, based on each conjunct's *observed* selectivity
during execution — a conjunct that has been filtering out most rows lately
gets evaluated before one that rarely does, so the total number of
per-conjunct evaluations goes down. This is a real, useful optimization, and
it is fully described in Chapter 12. But nothing about it generates code: the
set of possible operations was already fixed and compiled into the binary;
`AdaptiveFilter` only changes the *order* in which the interpreter calls
already-existing functions. A reordered interpreter is still an interpreter.

**"Compiling Tight Loops."** Torsten Grust's own DiDi Chapter 7 slides
(*"Vectorized Query Execution"*) [9] contain a section literally titled "Compiling
Tight Loops," followed by assembly-code listings generated with `clang -O2`
and a reference to Matt Godbolt's Compiler Explorer. Read in isolation, that
heading is exactly the kind of phrase that could be mistaken for "DuckDB
compiles its query loops." It is not that. That section of the course is a
pedagogical device: it takes a small, hand-written C function —
`PROJECT_sub_int_col_int_col`, doing exactly what a DuckDB projection operator
does for `col1 - col2` — and *ahead of time*, as a teaching exercise, compiles
it once with `clang` to show the reader what assembly a well-written tight
loop over a `STANDARD_VECTOR_SIZE`-sized (2048-element) vector actually looks
like: register assignment, the loop-unrolled and SIMD-vectorized versions
built up slide by slide, the `__restrict` pointer-aliasing hint, and so on.
This is Grust *compiling a standalone illustrative C snippet with a stock
compiler, once, before the lecture*, to explain what kind of machine code
DuckDB's own hand-written, statically-compiled tight loops resemble. It has
nothing to do with DuckDB generating or compiling any code at query time —
if anything, it is evidence for the *opposite* claim, since the whole point
of the exercise is to show that the loop shapes DuckDB's `ExecuteGenericLoop`-
style templates already produce, once, at DBMS build time, are as tight as
hand-optimized C.

## 13.5 The positive evidence: what an interpreter actually looks like

Absence of a JIT is not, by itself, proof of what DuckDB does instead. Chapter
12 covers the `ExpressionExecutor` in full depth; this section re-derives just
enough of it to make the crux distinction of this chapter concrete, because
that distinction is easy to state and easy to get subtly wrong.

`ExpressionExecutor` is the class responsible for evaluating a bound SQL
expression — `l.shipdate < DATE '1998-01-01'`, `o.custkey`, `sum(l.extendedprice)`,
anything that isn't itself a whole operator — against a `DataChunk`:

```cpp
// src/include/duckdb/execution/expression_executor.hpp
//! ExpressionExecutor is responsible for executing a set of expressions and
//! storing the result in a data chunk
class ExpressionExecutor {
public:
    // ...
    //! Execute the set of expressions with the given input chunk and store
    //! the result in the output chunk
    DUCKDB_API void Execute(DataChunk *input, DataChunk &result);
};
```

Underneath `Execute`, the dispatch mechanism (confirmed directly in
`src/execution/expression_executor.cpp`, where this exact `switch` appears at
three times: in state initialization (`InitializeState`), in the ordinary
execution path (`Execute`), and in the `WHERE`-clause path (`Select`)) is about as plain as interpreter dispatch gets:

```cpp
switch (expr.GetExpressionClass()) {
case ExpressionClass::BOUND_REF:        ...
case ExpressionClass::BOUND_CASE:       ...
case ExpressionClass::BOUND_CONJUNCTION: ...
case ExpressionClass::BOUND_CONSTANT:   ...
case ExpressionClass::BOUND_FUNCTION:   ...
case ExpressionClass::BOUND_LAMBDA:     ...
case ExpressionClass::BOUND_OPERATOR:   ...
case ExpressionClass::BOUND_PARAMETER:  ...
// ...
}
```
`src/execution/expression_executor.cpp:234` (and again at 279, 322)

Each branch dispatches into a per-expression-kind file
(`execute_comparison.cpp`, `execute_function.cpp`, `execute_case.cpp`, and so
on), and *those* functions, in turn, dispatch again — by value type, by vector
representation — down to a templated inner loop. Take the one-line expression
`i = 42` from DiDi Chapter 7 [9, slide 11] ("Deep Dive Into DuckDB C++ Code"),
which follows DuckDB 1.4's code:
`ExpressionExecutor::Execute` matches `COMPARE_EQUAL` in the switch above,
calls `VectorOperations::Equals(left, right, result, count)`, which calls
`ComparisonExecutor::Execute<duckdb::Equals>`, which switches again on the
left vector's physical type (`PhysicalType::INT64`, in this example) to reach
`ComparisonExecutor::TemplatedExecute<int64_t, Equals>`, which finally calls
`BinaryExecutor::ExecuteSwitch<int64_t, int64_t, bool, ..., Equals>` — a
function *template*, instantiated once per `(left type, right type, result
type, operator)` combination, that either takes a fast constant-folding path
or falls through to `ExecuteGeneric`, converts both vectors to DuckDB's
unified representation (Chapter 10), and runs a tight, `__restrict`-annotated
loop calling `Equals::Operation(...)` once per element. This is, end to end, a
tree walk with a runtime `switch` at every node, terminating in calls to
functions that were all fully written and fully compiled before this query
(or any query) ever existed.

In `v2.0-cyanoptera` the names on this path are different, but the kind of path
is not. There is no `COMPARE_EQUAL` case in the switch. A comparison is a
`BOUND_FUNCTION`, so it goes through `execute_function.cpp`. From there it
reaches `VectorOperations::Equals`, then `TryPrimitiveComparisonExecute`
(a switch on the physical type), then `BinaryExecutor::Execute`, then
`ScalarExecutor::ExecuteInternal` (a switch on the vector representation),
and finally the templated `ScalarExecutor::ExecuteFlat` loop. Chapter 12
§12.5–§12.6 walks this exact chain with line numbers. Every step is still a
runtime choice among functions that already exist in the binary.

This is the crux of the whole chapter, and it deserves to be drawn out as its
own diagram, because the two architectures look deceptively similar from a
distance — both end up running tight, type-specialized machine code — and
differ entirely in *when* that specialized code came into existence:

```
COMPILING ENGINE (HyPer / Umbra), per Neumann 2011's produce/consume model:

  SQL query
     │
     ▼
  optimize
     │
     ▼
  GENERATE new C++ / IR / machine code specific to THIS query's plan
     │          (happens at QUERY TIME — adds latency here, once per query,
     │           but the generated function has zero interpretation overhead
     ▼           left inside it)
  compile / JIT that generated code
     │
     ▼
  execute the generated function


DUCKDB, vectorized tree-walking interpreter:

  ┌─────────────────────────────────────────────────────────────────────┐
  │ DBMS BUILD TIME — once, ever, when `duckdb` itself is compiled:      │
  │   C++ templates (BinaryExecutor::ExecuteSwitch<L,R,RES,OP>, and      │
  │   dozens like it) get instantiated by the C++ compiler for every     │
  │   (operator × type × vector-representation) combination the         │
  │   codebase declares, and baked into the duckdb binary as ordinary,   │
  │   already-compiled machine code.                                    │
  └─────────────────────────────┬─────────────────────────────────────────┘
                                 │
  ┌─────────────────────────────▼─────────────────────────────────────────┐
  │ QUERY TIME — every query, no code generation occurs here:            │
  │   SQL query → optimize → physical plan of PhysicalOperators →        │
  │   ExpressionExecutor walks the BOUND EXPRESSION TREE and, via a      │
  │   runtime switch on ExpressionClass (then again on PhysicalType,     │
  │   then again on VectorType), SELECTS which of the pre-existing,     │
  │   already-compiled functions to call for each node — once per       │
  │   DataChunk, i.e. once per up-to-2048 rows, not once per row —      │
  │   and calls it.                                                     │
  └───────────────────────────────────────────────────────────────────────┘
```

Both architectures end up running a tight, specialized loop for `int64 = 42`.
The compiling engine gets there by *writing a new function into existence*,
specific to this query, this session, right now. DuckDB gets there by having
*already written every function it could plausibly need*, as a C++ template,
years before this particular query was typed, and merely picking the right
one at runtime with a `switch`. The template specialization visible all over
`execution_executor/` and `common/vector_operations/` looks, from a distance,
like the same idea as per-query codegen — a different, hyper-specialized inner
loop for every `(operator, type, representation)` combination — and Chapter 1
§1.4 already flagged this resemblance as the thing most likely to mislead a
reader. It is not the same idea. Those specialized loops are all compiled
*once*, at DBMS-build time; the query does nothing but select among them.

## 13.6 In Grust's own words

DiDi Chapter 7 ("Vectorized Query Execution") [9] states this distinction
explicitly, in the middle of exactly the material re-derived in Section 13.5
above. The slide walks through the `switch`-on-representation code a naive
implementation of `l + r` would need, then poses the problem DuckDB actually
faces — controlling the resulting volume of C++ generated by that
combinatorial `switch` nesting — and gives two answers:

> DuckDB aims to control the resulting C++ code volume:
> 1. Convert vectors `l`, `r` to unified representation, process these by
>    generic loops (cuts lower levels of the decision tree).
> 2. Use C++ templates to generate code at *DBMS compile time*.

attached to point 2 is footnote 3, which is the single most load-bearing
sentence in this whole chapter, quoted here exactly as it appears on the slide:

> A different breed of DBMS — for example Umbra — generates and compiles
> C++/IR/machine code at *query translation time*. These query-specific code
> fragments are then linked with the DBMS and invoked at runtime.

([9, slide 10, fn. 3] — verified verbatim against the slide PDF, fetched
for this chapter.)

Read that footnote twice, because both halves matter. The phrase "*at query
translation time*" (italicized on the slide itself) is doing the same job the
diagram in Section 13.5 does with boxes and arrows: it names the moment the
compiling engine's code generation happens, and contrasts it — by construction,
since it's a footnote hung off "DBMS compile time" — against DuckDB's own
templates, which generate code once, when *DuckDB itself* is built, not once
per query. Umbra is named explicitly as the "different breed" that does the
former; DuckDB is the subject of the sentence the footnote is attached to, and
it does the latter. This is Grust, in direct communication with DuckDB Labs
and reading the same source this book reads, stating precisely the boundary
this chapter has spent three sections proving from the code side.

The same course's concluding material on this topic — after walking through
branching vs. branch-free `FILTER` code, five-stage instruction pipelines,
branch misprediction penalties, and branch-prediction heuristics — lands on
exactly the same anti-per-query-specialization philosophy as its own
practical takeaway, in the deck's own words:

> DBMS code aims to avoid branch mispredictions in tight loops: Prefer
> branch-less implementations of operator logic, or at least reduce the
> number of random/hard-to-predict branches.

([9, slide 29], "Avoiding Branch Mispredictions.")

Put the footnote and this closing recommendation side by side and the
one-sentence test this whole course is building toward becomes explicit: an
engine in the HyPer/Umbra tradition solves the "avoid overhead in the tight
loop" problem by *generating a new, branch-free, type-specific function per
query*; DuckDB solves the identical problem by writing branch-free, type-specific
functions **once**, as C++ templates, and letting the compiler that builds
DuckDB itself do the specialization — which is exactly the sense in which,
to answer the question this chapter opened with in one line, DuckDB is a
system that does not generate code from SQL queries.

## 13.7 What DuckDB gets instead — a real tradeoff, not a consolation prize

Declining to compile each query is not free, and it's worth being honest
about what it costs and what it buys, rather than treating this as DuckDB
quietly losing a feature HyPer has.

**The cost: interpretation overhead is amortized, not eliminated.** Every time
`ExpressionExecutor::Execute` walks the bound expression tree for a
`DataChunk`, it pays the cost of that `switch`-based dispatch: branch
prediction on the `ExpressionClass`/`PhysicalType`/`VectorType` switches,
virtual-dispatch-adjacent overhead for `PhysicalOperator` calls, and the
general cost of not having a single, straight-line function that goes from
input columns to output columns with no decision points in between. A
compiling engine pays this dispatch cost exactly once, at query-compile time,
and then never again — the generated function *is* the straight-line path,
with every type and representation baked in as a compile-time fact, not a
runtime `switch` outcome. DuckDB pays a version of this dispatch cost on
*every* `DataChunk`. The saving grace is the denominator: because DuckDB
processes up to 2,048 rows per `Execute` call (Chapter 10), that per-call
dispatch overhead is divided by up to 2,048 rows, not paid once per row the
way a naive row-at-a-time interpreter would. A compiling engine's per-row
dispatch cost is zero; DuckDB's is (dispatch cost) / (up to 2048) — not zero,
but small enough, in practice, to be dwarfed by the actual arithmetic and
memory-access work the templated inner loop does over that same batch.

**The other side of the ledger: query-time compilation is not free either.**
A compiling engine's zero-per-row-dispatch advantage is bought with a cost
DuckDB never pays at all: the latency of generating and compiling (or JITing)
a fresh program *every time a new query shows up*, even a trivial one, even
one that will run once and never again. This is where the two engines'
target workloads diverge sharply enough that "which is faster" stops having a
single answer. HyPer and Umbra are built with a workload in mind where queries
are often long-running, executed repeatedly, or expensive enough that paying
compilation latency once is trivially amortized by the execution time it
saves. DuckDB's actual target workloads — embedded analytics inside another
application, interactive data-science notebooks, ad-hoc exploratory SQL typed
by a human waiting for the result, a script that runs a different `SELECT` on
every invocation — are workloads dominated by *short*, *frequently changing*
queries, often issued once. For that workload, paying tens or hundreds of
milliseconds of code-generation-and-compilation latency on every single query
— *before* execution even starts — is a real, user-visible cost, and it is
paid on every query regardless of how small the result set turns out to be.
Section 2.3's benchmark chapter already demonstrated the shape of this
argument in miniature, for a completely different overhead: DuckDB's
materialized-table query ran in 0.002 s partly *because* nothing about running
that query involved generating and compiling a program first. Declining to
compile is, for DuckDB's actual users, a defensible design choice on its own
terms — not merely the option left over after deciding not to build a
compiler.

**What claws back the difference.** Four mechanisms, each covered fully
elsewhere in this book, do the work a compiling engine's code generator would
otherwise be doing, without paying query-time compilation latency to get
there:

- **C++ template specialization at DBMS-build time** (Section 13.5, Chapter
  12) gives most of the same per-type, per-representation efficiency a code
  generator would produce — the difference is *who* writes the specialization
  logic (a human, once, as a template) and *when* the compiler instantiates it
  (once, when DuckDB is built) rather than *what* the specialization looks
  like once it's running.

  The build-time nature of these specializations is visible in the source in
  one more way. Each template specialization costs space in the binary.
  DuckDB has a build option, `SMALLER_BINARY` (`CMakeLists.txt:704`), that
  drops chosen specializations when DuckDB is compiled. Each one is a named
  feature in `src/include/duckdb/common/smaller_binary.hpp`, guarded in the
  code by `#if !DUCKDB_SMALLER_BINARY(feature)`. For example,
  `comparison_select` drops the special `Select` callback of the comparison
  functions, and `primitive_comparison_execute` drops the fast path for
  primitive types in `TryPrimitiveComparisonExecute`. A trimmed build then
  takes a more generic, already-compiled path. A third feature,
  `scalar_executor_constant`, covers the all-constant case of
  `ScalarExecutor` (Chapter 12 §12.6). This option only makes sense
  in an interpreter. The set of specialized loops is fixed when DuckDB is
  built, and the builder can choose to make it smaller. No query can add one
  back.
- **SIMD, branchless code, and explicit loop unrolling** (Chapter 12, and
  demonstrated by hand in Section 2.3.5's `HAS_NL` bit-trick) close most of
  the remaining gap between "interpreted dispatch, then a tight specialized
  loop" and "one fused, branch-free generated function" — because once
  dispatch has happened, the loop itself is every bit as tight, vectorized,
  and branch-free as anything a code generator would emit; DiDi Chapter 7's
  own assembly listings (Section 13.4) are the direct evidence for this.
- **Software prefetching** for the access patterns that can't be made
  predictable any other way (Chapter 12) hides memory latency the same way a
  compiling engine's generated code would have to hide it too — this is a
  hardware-latency problem neither architecture escapes for free.
- **The storage layer** (Chapters 14–18) reduces how much data ever needs
  interpreting over in the first place — compression, zonemaps, and
  projection/filter pushdown mean the `ExpressionExecutor` is walking its
  tree over far fewer, far smaller `DataChunk`s than a naive reading of the
  SQL would suggest, which is a savings a per-query code generator gets no
  special credit for either, since it has to touch the same reduced data.

Put together, this is not "DuckDB is slower because it doesn't compile, but
that's an acceptable price." It is closer to: DuckDB identified which parts
of the compiling-engine advantage actually matter for its target workloads
(low per-batch dispatch overhead, tight specialized inner loops) and got
those by moving the specialization decision to DBMS-build time instead of
query time — while deliberately not paying the one cost (query-time
compilation latency) that its actual users would feel on every single query.

## 13.8 Two lineages, precisely placed

It is worth being precise about where each tradition comes from, because a
reader who wants to read further should know which paper describes which
system, and because the two traditions are easy to conflate — which is,
itself, probably the single most common source of the exact question this
chapter answers.

**The compiling-engine tradition** begins with Thomas Neumann's 2011 VLDB
paper, *"Efficiently Compiling Efficient Query Plans for Modern Hardware"*
[13], which introduced the produce/consume code-generation model and
implemented it in **HyPer** (Neumann and Alfons Kemper's group at TU München),
compiling generated query plans to LLVM IR and JIT-compiling that IR at query
time. **Umbra**, the same research group's successor system, continues this
tradition but with a notable refinement: rather than paying LLVM's own
(comparatively heavyweight) compilation latency on every query, Umbra compiles
to a custom, lightweight intermediate representation of its own design,
specifically to bring query-time compilation latency down — this detail is
mentioned tentatively here, since this book's primary sourcing is the DuckDB
tree rather than Umbra's own, but it is consistent with what DiDi's footnote 3
says Umbra does ("generates and compiles C++/IR/machine code at query
translation time") and with Umbra's own public description of itself as
targeting low compilation latency without giving up compiled-code performance.

**The contrasting vectorized-execution tradition** traces to Peter Boncz,
Marcin Zukowski, and Niels Nes's 2005 CIDR paper, *"MonetDB/X100:
Hyper-Pipelining Query Execution"* [27], and its
commercial descendant, Vectorwise (later Actian Vector). This tradition's
answer to "how do you get compiled-code-like performance without compiling
per query" is exactly the batching/templating/SIMD combination Sections 13.5
and 13.7 describe: process data in cache-resident batches, using
pre-specialized inner loops selected at runtime rather than generated at
runtime.

DuckDB's **execution model** sits squarely in the second, MonetDB/X100
lineage — vectorized, batch-at-a-time, runtime-dispatched, pre-compiled
operator functions, exactly as Sections 13.3–13.5 demonstrate. Its
**pipelining and parallelism model** (Chapter 11) is, at the same time,
architecturally descended from the *first* lineage — push-based execution,
pipeline breakers, and dynamic per-chunk work distribution functionally
equivalent to what the literature calls "morsel-driven" parallelism (a term,
worth noting, that does not itself appear anywhere in DuckDB's own source —
it is an external label the community applies to DuckDB's `TaskScheduler`
design, not a name DuckDB's code uses for itself). DuckDB borrowed the
Neumann lineage's *pipelining* idea and declined its *compilation* idea. That
is a coherent, deliberate design position, not an inconsistency — but stating
it as "DuckDB uses Neumann-style execution" without immediately qualifying
*which half* is exactly the kind of imprecise phrasing that produces the
question this chapter set out to answer.

## 13.9 Summary

DuckDB does not compile SQL queries to machine code, LLVM IR, or any other
query-specific generated form, at any point in its pipeline. The two strongest
pieces of evidence for this are, first, an exhaustive case-insensitive grep
across the entirety of `src/` for `jit`, `asmjit`, `codegen`, computed-goto
patterns, and `llvm` — zero real hits across all five patterns, corroborated
by a `third_party/` directory that vendors compression codecs, data formats,
allocators, sort algorithms, and a test framework, but no compiler-construction
library of any kind — and second, Torsten Grust's own DiDi course material
stating the boundary explicitly: DuckDB uses "C++ templates to generate code
at *DBMS compile time*," in direct, footnoted contrast to "a different breed
of DBMS — for example Umbra" that "generates and compiles C++/IR/machine code
at *query translation time*." What DuckDB does instead, in one sentence: it is
a vectorized, tree-walking interpreter that amortizes per-batch dispatch
overhead across up to 2,048 rows at a time, and gets its per-type,
per-representation efficiency from C++ templates that the compiler
specializes once, when DuckDB itself is built, rather than from a code
generator that specializes a fresh function once per query. It kept the
Neumann/HyPer lineage's push-based, pipelined execution model (Chapter 11);
it declined that same lineage's query-time code generation, in favor of the
MonetDB/X100/Vectorwise tradition's batching-plus-specialization approach
(Chapter 12) — a choice that trades a small, batch-amortized per-`DataChunk`
dispatch cost for the complete absence of per-query compilation latency,
which is the right trade for the short, ad-hoc, frequently changing queries
DuckDB is actually built to run.

---

**References cited in this chapter**

- Neumann, produce/consume compilation [13].
- Boncz, Zukowski and Nes, MonetDB/X100 [27].
- Grust, DiDi Chapter 7, "Vectorized Query Execution" [9].
- Leis et al., morsel-driven parallelism [31].
- Neumann and Freitag, Umbra [14].
