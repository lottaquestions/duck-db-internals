# Chapter 6 — Subquery Decorrelation

## 6.1 The problem this chapter solves

Take Query B from the front matter:

```sql
SELECT c.name,
       (SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey) AS n_orders
FROM customer c;
```

The subquery in the `SELECT` list references `c.custkey` — a column that belongs
to the *outer* query, not to `orders`. This is a **correlated subquery**: its
result depends on which outer row is currently being produced. Every database
textbook introduces exactly this shape of query, and every textbook then
introduces the same first, naive execution strategy for it, usually under the
name **Apply operator** or **correlated nested-loop**: for each row of
`customer`, substitute that row's `custkey` into the subquery and run it fresh.

```
for c_row in scan(customer):
    n = execute("SELECT COUNT(*) FROM orders o WHERE o.custkey = " || c_row.custkey)
    emit(c_row.name, n)
```

This is correct, and it is also the reason correlated subqueries have a bad
reputation for being slow. If `customer` has a million rows, this strategy
scans (or index-probes) `orders` up to a million times — once per outer row,
with no way for the engine to notice that the millionth scan of `orders` and
the first scan of `orders` are doing related work. There is no batching, no
opportunity for a single hash join or a single grouped aggregation over all of
`orders` at once; the query planner's entire toolbox for making a filter or a
join or an aggregate cheap (Chapter 7, Chapter 8) is unavailable, because from
the physical operator's point of view there are a million *unrelated*
single-row queries, not one query. Asymptotically, a query that should cost
something like O(|customer| + |orders|) — one pass over each table, joined
once — instead costs O(|customer| × cost(orders subquery)), which for anything
larger than toy data is unusable.

DuckDB's answer to this is not "provide a smarter physical operator that does
the Apply-operator strategy but pipelines it well." DuckDB's answer is stronger
and stranger than that: **there is no physical operator in DuckDB capable of
executing a correlated subquery by re-running it per outer row, at all.** Grep
the full `PhysicalOperatorType` enum (`src/include/duckdb/common/enums/physical_operator_type.hpp`,
walked in full in Chapter 1's research notes) and you will not find anything
named `CORRELATED_NESTED_LOOP`, `APPLY`, or similar — every join entry in that
enum (`BLOCKWISE_NL_JOIN`, `NESTED_LOOP_JOIN`, `HASH_JOIN`, `CROSS_PRODUCT`,
`PIECEWISE_MERGE_JOIN`, `IE_JOIN`, `LEFT_DELIM_JOIN`, `RIGHT_DELIM_JOIN`,
`POSITIONAL_JOIN`, `ASOF_JOIN`, plus `RECURSIVE_KEY_JOIN` near the end of the
enum, a physical-only specialization for recursive CTEs) describes a join between two relations that are
each read *once*, not a re-parameterized re-execution of one side per row of
the other. This is confirmed directly by the DiDi course, in the same slide
that introduces `DEPENDENT_JOIN` (the notation for the not-yet-decorrelated
correlated subquery — introduced properly in §6.2):

> "In fact, [DuckDB] does not implement `DEPENDENT_JOIN` as a *physical* plan
> operator. Query decorrelation thus is an absolute must in DuckDB."
> [10, p. 26, fn. 8]

DuckDB's own source makes the same point structurally: `LogicalDependentJoin`,
the logical-plan marker for "this is a correlated subquery, not yet resolved,"
carries a header comment that is unusually blunt about its own temporariness:

```cpp
// duckdb/planner/operator/logical_dependent_join.hpp
//
// logical_dependent_join represents a logical operator for lateral joins that
// is planned but not yet flattened
//
// This construct only exists during planning and should not exist in the plan
// once flattening is complete. Although the same information can be kept in the
// join itself, creating a new construct makes the code cleaner and easier to
// understand.
```
(`src/include/duckdb/planner/operator/logical_dependent_join.hpp:4-12`)

So the algorithm this chapter covers — turning a `LogicalDependentJoin` into an
ordinary join before the optimizer's first pass ever runs — is not an
*optional* optimization DuckDB applies when it can. It is a mandatory
rewrite that every correlated subquery must survive, because the alternative
(leaving it as-is) is a plan DuckDB's execution engine is structurally unable
to run. This is also why, as Chapter 1 flagged, most of the interesting work
happens in `src/planner/`, not `src/optimizer/`: by the time `Optimizer::Optimize`
gets the plan, `LogicalDependentJoin` is already gone.

## 6.2 The marker node and the D ⋈ᵈ Q notation

### `LogicalDependentJoin`

When the binder (Chapter 4) resolves a subquery expression and discovers that
it references a column from an enclosing query — a `BoundColumnRefExpression`
whose `Depth()` is greater than zero, meaning "this column lives more than
zero query-nesting-levels up from here" — it does not try to plan the subquery
in isolation. Instead it builds a `LogicalDependentJoin`, a thin subclass of
`LogicalJoin`:

```cpp
// src/planner/operator/logical_dependent_join.cpp
LogicalDependentJoin::LogicalDependentJoin(JoinType join_type)
    : LogicalJoin(join_type, LogicalOperatorType::LOGICAL_DEPENDENT_JOIN) {
}
```

Its two children are the plan for the *outer* query (the side that produces
the correlation values) and the plan for the *subquery* (the side that still
references those values). It additionally carries a `CorrelatedColumns` list —
one `CorrelatedColumnInfo{ binding, type, name, depth }` per free column
reference the subquery makes into the outer scope
(`src/include/duckdb/planner/binder.hpp:101-118`, collected in the
`CorrelatedColumns` struct that follows it). For Query B, that list has
exactly one entry: the binding for `c.custkey`.

### D ⋈ᵈ Q, in plain English before it's used

DiDi (and the paper it implements — Neumann and Kemper's "Unnesting Arbitrary
Queries" [12], see §6.9) write the not-yet-decorrelated plan as **D ⋈ᵈ Q**. If
you haven't seen this notation before, here is what each symbol means, with
no algebra required yet:

- **D** stands for the *duplicate-free set of correlation parameter values* —
  concretely, for Query B, the distinct values of `customer.custkey` that
  actually occur. You can think of D as "the list of distinct arguments the
  subquery will need to be evaluated for."
- **Q** is the plan for the correlated subquery itself, still containing a
  free reference to the correlation column (here, `o.custkey = c.custkey`,
  where `c.custkey` is the free reference).
- **⋈ᵈ** is the *dependent join*: conceptually, "evaluate Q once for each
  row of D, substituting that row's value in for the free reference, and
  return the union of all those per-row results, tagged with which row of D
  produced them." This is exactly the Apply-operator semantics from §6.1 —
  DiDi states this explicitly: "Nested loop semantics: Evaluate [Q] for each
  row produced by [D]" [10, p. 26].

So D ⋈ᵈ Q is *the same naive strategy* as §6.1, just written algebraically
instead of as a `for` loop. Writing it this way is what makes it possible to
apply algebraic rewrite rules to it — which is the entire content of this
chapter. **Query decorrelation** is the process of algebraically rewriting
D ⋈ᵈ Q, using only equivalence-preserving steps, until no operator in the
rewritten tree still needs D's *distinctness* to be recomputed per row — at
which point ⋈ᵈ can be replaced by an ordinary ⋈, something DuckDB's execution
engine can actually run.

## 6.3 Walking `FlattenDependentJoins` over Query B

The rewrite is implemented by `FlattenDependentJoins`
(`src/planner/subquery/flatten_dependent_join.cpp`). It runs once per
statement, in `Planner::CreatePlan`, right after `Binder::Bind` has returned
the whole plan and before the optimizer's first pass:

```cpp
// src/planner/planner.cpp:254-262
		// A LogicalPlanStatement carries an already planned (and possibly optimized) plan
		if (statement.type != StatementType::LOGICAL_PLAN_STATEMENT) {
			RewriteTriggersToDependent(*this->binder, *this->plan);
			RecursiveDependentJoinPlanner::Plan(*this->binder, this->plan);
			this->plan = FlattenDependentJoins::DecorrelateIndependent(*this->binder, std::move(this->plan));
		}
		D_ASSERT(!ContainsDependentJoin(*this->plan));
		D_ASSERT(VerifyPlannedExpressions(*this->plan));
		D_ASSERT(VerifyCanonicalComparisonJoins(*this->plan));
```

It does not run inside the binder each time a `LogicalDependentJoin` is
built. The binder builds *all* dependent joins first, and this one call then
flattens them all. The `D_ASSERT` checks that none is left.

The `if (statement.type != StatementType::LOGICAL_PLAN_STATEMENT)` guard
skips one case. A `LogicalPlanStatement` wraps a plan that was already planned (and maybe
optimized) elsewhere, so it must not be decorrelated a second time.

This section walks its core algorithm, in the
simplified, per-operator-type form DiDi teaches it in (and that the original
Neumann–Kemper paper presents it in); §6.4 then reconciles this with what the
actual, considerably more general, production C++ does to a real `LogicalOperator`
tree.

**Initial plan.** Query B's subquery (`SELECT COUNT(*) FROM orders o WHERE
o.custkey = c.custkey`) is a filter feeding an ungrouped aggregate. The
unoptimized plan the binder hands to `FlattenDependentJoins` looks like this
(using the box-drawing convention from the front matter, with algebra symbols
alongside each operator's name so this ties directly to the D ⋈ᵈ Q notation):

```
┌───────────────────────────────┐
│ LogicalProjection              │  π [c.name, n_orders]
└──────────────┬─────────────────┘
               │
┌──────────────▼─────────────────┐
│ LogicalDependentJoin  (⋈ᵈ)      │  D ⋈ᵈ Q
└──────┬────────────────┬────────┘
       │                │
┌──────▼──────┐   ┌─────▼──────────────────────┐
│ LogicalGet    │   │ LogicalAggregate  (Γ)       │  n_orders: count(*) / ()
│ customer c    │   └──────────────┬─────────────┘
└──────────────┘                  │
                     ┌─────────────▼─────────────┐
                     │ LogicalFilter  (σ)          │  o.custkey = c.custkey
                     └─────────────┬─────────────┘
                                   │
                     ┌─────────────▼─────────────┐
                     │ LogicalGet                  │
                     │ orders o                    │
                     └─────────────────────────────┘
```

The `LogicalFilter`'s predicate `o.custkey = c.custkey` contains a free
reference to `c.custkey` — this is exactly the correlation `Depth() > 0`
column reference from §6.2, and it is why the join above it had to be built as
a `LogicalDependentJoin` rather than an ordinary join in the first place.

**Step 1 — compute the duplicate-free parameter set D.** The rewrite starts
by observing that `customer` may (and typically will) contain repeated
`custkey` groupings are irrelevant here — actually `custkey` is a primary key,
but the algorithm cannot assume that in general, so it must explicitly
deduplicate. It introduces a duplicate-eliminating projection over the
correlation column:

```
D  :=  δ[d: c.custkey](customer)
```

read as "D is the result of scanning `customer`, projecting out just
`custkey` renamed to `d`, and removing duplicate rows." The dependent join is
then rewritten to join the *original*, undeduplicated outer relation back
against a *new*, smaller dependent join that only needs to run against D:

```
customer ⋈[c.custkey = d] Q   ≡   customer ⋈[c.custkey = d] (D ⋈ᵈ Q)
```

This step alone does not remove ⋈ᵈ — it just shrinks its left input from
"every row of `customer`" to "every *distinct* correlation value `customer`
produces." That is already a real win if `custkey` has few distinct values
relative to `customer`'s row count, but the real payoff comes from what
happens next: pushing ⋈ᵈ down through Q's own operators until D's distinctness
is no longer needed at all.

**Step 2 — push ⋈ᵈ down through the aggregate.** Q's outermost operator is
`Γ [n_orders: count(*)] / ()` — an *ungrouped* aggregate (empty group list).
The rewrite rule for aggregation (stated formally in §6.5) says: pushing a
dependent join down through an aggregate adds the correlation column to that
aggregate's `GROUP BY` list. Concretely:

```
D ⋈ᵈ Γ[n_orders: count(*)] / ()  (σ[o.custkey = d](orders))
   ≡
Γ[n_orders: count(*)] / (d)  (D ⋈ᵈ σ[o.custkey = d](orders))
```

This is the crux of the whole chapter. Instead of "run `COUNT(*) FROM orders
WHERE o.custkey = d` once per row of D," the aggregate now says "compute
`COUNT(*)` **grouped by** `d`, over the whole join of D against `orders`" —
one single grouped aggregation, computed once, in one pass over `orders`,
that happens to produce one output row per distinct `d`. The dependent join
has moved from *above* the aggregate to *below* it, and in doing so the
per-row "Apply" semantics were converted into ordinary relational-algebra
grouping semantics.

**Step 3 — push ⋈ᵈ down through the filter.** What is left below the
aggregate is `σ[o.custkey = d](orders)` — the correlated predicate, now
already rewritten (by the binder's companion pass,
`rewrite_correlated_expressions.cpp`) to reference `d` instead of the
original free `c.custkey`. The filter rewrite rule is the simplest of the
three: a dependent join commutes freely with a filter below it, because the
filter doesn't care whether the row arriving from D is one of many or the
join's only row:

```
D ⋈ᵈ σ[o.custkey = d](orders)   ≡   σ[o.custkey = d](D ⋈ᵈ orders)
```

**Step 4 — eliminate ⋈ᵈ.** After step 3, the dependent join sits directly
above `orders`'s base scan, with nothing below it that still references `d`
except the filter predicate `o.custkey = d` sitting right above it — and that
predicate is *exactly* an equality between D's parameter column and a column
of `orders`. At this point the general elimination rule applies: **a
dependent join whose parameter is not free anywhere below it is just an
ordinary join**, and the filter predicate folds directly into that join's
condition instead of remaining a separate `σ` node:

```
D ⋈ᵈ orders, with filter o.custkey = d   ≡   D ⋈[o.custkey = d] orders
```

**Final plan.** Assembling all four steps gives an entirely ordinary join
tree — no ⋈ᵈ anywhere:

```
┌───────────────────────────────┐
│ LogicalProjection              │  π [c.name, n_orders]
└──────────────┬─────────────────┘
               │
┌──────────────▼─────────────────┐
│ LogicalComparisonJoin           │  c.custkey = d      (ordinary ⋈)
└──────┬────────────────┬────────┘
       │                │
┌──────▼──────┐   ┌─────▼───────────────────────────┐
│ LogicalGet    │   │ LogicalAggregate  (Γ)             │  n_orders: count(*) / (d)
│ customer c    │   └──────────────┬───────────────────┘
└──────────────┘                  │
                     ┌─────────────▼─────────────────────┐
                     │ LogicalComparisonJoin  (LOGICAL_    │  o.custkey = d
                     │ DELIM_JOIN — see §6.4)              │
                     └──────┬───────────────────┬─────────┘
                            │                   │
                 ┌──────────▼─────────┐  ┌──────▼──────┐
                 │ "D": the customer    │  │ LogicalGet   │
                 │ correlation column   │  │ orders o     │
                 │ (see §6.4 for what   │  └─────────────┘
                 │ this really is)      │
                 └──────────────────────┘
```

Every node in this tree is an ordinary `LogicalOperator` that the physical
plan generator (Chapter 9) and every optimizer pass already know how to
handle — the one exception being that the inner join is *tagged* as
`LOGICAL_DELIM_JOIN` rather than a plain `LOGICAL_COMPARISON_JOIN`, which
carries some extra bookkeeping metadata explained next.

## 6.4 What "DELIM" means, and what the real logical plan looks like

**"DELIM" stands for duplicate elimination.** This is stated directly, more
than once, in the source:

```cpp
// src/include/duckdb/planner/operator/logical_comparison_join.hpp:31-34
//! Used for duplicate-eliminated MARK joins
...
//! The set of columns that will be duplicate eliminated from the LHS and pushed into the RHS
vector<unique_ptr<Expression>> duplicate_eliminated_columns;
```
```cpp
// src/include/duckdb/execution/operator/join/physical_delim_join.hpp:17-18
//! PhysicalDelimJoin represents a join where either the LHS or RHS will be
//! duplicate eliminated and pushed into a PhysicalColumnDataScan in the other side.
```
```cpp
// src/include/duckdb/planner/operator/logical_delim_get.hpp:15
//! LogicalDelimGet represents a duplicate eliminated scan belonging to a DelimJoin
```

Notice something important buried in that first excerpt: **`LOGICAL_DELIM_JOIN`
is not its own C++ class.** `FlattenDependentJoins::FinalizeDependentJoin`
(`src/planner/subquery/flatten_dependent_join.cpp:662-771`) builds the final
join like this:

```cpp
auto finalized = make_uniq<LogicalComparisonJoin>(op.join_type, LogicalOperatorType::LOGICAL_DELIM_JOIN);
finalized->conditions = std::move(conditions);
finalized->mark_types = std::move(mark_types);
finalized->duplicate_eliminated_columns = std::move(duplicate_eliminated_columns);
finalized->children = std::move(op.children);
```
(`src/planner/subquery/flatten_dependent_join.cpp:763-767`)

It is a perfectly ordinary `LogicalComparisonJoin`, just constructed with its
`type` field set to the `LOGICAL_DELIM_JOIN` enum value instead of
`LOGICAL_COMPARISON_JOIN`, and carrying the extra `duplicate_eliminated_columns`
field (the D of this specific dependent join — for Query B, one expression:
`c.custkey`). This is a good example of a place where DiDi's teaching algebra
(an explicit `δ` operator materializing D) and DuckDB's actual logical-plan
shape genuinely diverge, and it is worth being precise about the divergence
rather than letting the algebra stand in for the real tree:

- The **logical** tree DuckDB actually builds has *no* explicit
  duplicate-elimination node at all. The LHS child of the `LOGICAL_DELIM_JOIN`
  is simply the *raw* outer relation (`customer`, unaggregated,
  undeduplicated) — the same child it always had. The only trace that D
  exists is the `duplicate_eliminated_columns` list attached to the join node
  as metadata.
- Wherever the RHS subtree needs to read D's values, it does so through a
  `LogicalDelimGet` — a scan-shaped placeholder node
  (`src/planner/operator/logical_delim_get.cpp`) that simply says "read the
  duplicate-eliminated columns of my enclosing `LOGICAL_DELIM_JOIN`'s LHS
  here," without itself doing any deduplication.
- The actual **computation** of D — an honest hash-based DISTINCT over the
  LHS's `duplicate_eliminated_columns` — is deferred all the way to the
  **physical** plan. `PhysicalPlanGenerator::PlanDelimJoin` builds a
  `PhysicalHashAggregate` (confusingly named `distinct` in the field list of
  `PhysicalDelimJoin`, `src/include/duckdb/execution/operator/join/physical_delim_join.hpp:27`)
  that groups on exactly those columns, and wires every `PhysicalColumnDataScan`
  descended from a `LogicalDelimGet` (tagged `DELIM_SCAN`) to read from that
  aggregate's output:
  ```cpp
  // src/execution/physical_plan/plan_delim_join.cpp:26-32
  PhysicalOperator &PhysicalPlanGenerator::PlanDelimJoin(LogicalComparisonJoin &op) {
      // first create the underlying join
      auto &plan = PlanComparisonJoin(op);
      ...
      // duplicate eliminated join
      // first gather the scans on the duplicate eliminated data set from the delim side
  ```
  Even the join-order optimizer (Chapter 8) is aware of this deferral: when it
  needs a cardinality estimate for the LHS of a `LOGICAL_DELIM_JOIN`, it
  builds a throwaway `dummy_aggr` over `duplicate_eliminated_columns` purely to
  get realistic distinct-count statistics, with the comment "create dummy
  aggregation for the duplicate elimination" (`src/optimizer/join_order/relation_manager.cpp:741-748`)
  — it has to fake this, because the real aggregation genuinely does not
  exist yet at the logical-plan stage it is running at.

So: **D is a real, load-bearing algebraic concept, and the DiDi/Neumann-Kemper
algebra's explicit `δ` operator is a completely accurate description of what
the rewrite conceptually does — but DuckDB's concrete logical-plan
representation never reifies D as its own tree node.** It represents "the LHS,
plus a promise (`duplicate_eliminated_columns`) that this specific set of its
columns will be deduplicated by whatever executes this join." This is a
genuinely elegant design choice: it keeps the logical tree simpler (no extra
operator type to special-case through every optimizer pass) while still
giving the physical plan generator everything it needs to build the real
DISTINCT computation later. Finally, the `LEFT`/`RIGHT` prefix on the physical
operators (`PhysicalLeftDelimJoin`, `PhysicalRightDelimJoin`) records which
*side* of the join is the duplicate-eliminated one — controlled by the
logical join's `delim_flipped` flag — a physical-plan-generation detail
that belongs properly to Chapter 9 and is mentioned here only so the naming
doesn't look mysterious later.

### 6.4.1 What the default plan really looks like: DelimJoins become CTEs

With default settings, the `LOGICAL_DELIM_JOIN` above is not the plan the
optimizer receives. One more step runs right after flattening. It rewrites
every DelimJoin into ordinary joins over materialized CTEs. A setting,
`delim_join_as_cte`, can turn this step off, but it is marked *deprecated*
(setting it prints a warning). The CTE form is the one DuckDB intends to keep.

The step is called from `FlattenDependentJoins::DecorrelateIndependent`, the
entry point the planner uses:

```cpp
// src/planner/subquery/flatten_dependent_join.cpp:281-292
unique_ptr<LogicalOperator> FlattenDependentJoins::DecorrelateIndependent(Binder &binder,
                                                                          unique_ptr<LogicalOperator> plan) {
	CorrelatedColumns correlated;
	FlattenDependentJoins flatten(binder, correlated);
	flatten.DecorrelateSubtree(plan, true, {});
	if (Settings::Get<DelimJoinAsCteSetting>(binder.context)) {
		DelimJoinCTERewriter::Rewrite(binder, plan);
	}
	// ANY joins must retain their general shape until enclosing decorrelation and binding rewrites are complete.
	FinalizeAnyJoins(plan);
	return plan;
}
```

`DelimJoinAsCteSetting` defaults to `true` (`src/include/duckdb/main/settings.hpp:1109-1120`,
description: "Rewrite delim joins to materialized CTEs during dependent join
flattening."). `DelimJoinCTERewriter::MaterializeDelimJoinAsCTE`
(`src/planner/subquery/delim_join_cte_rewriter.cpp:2647`) turns each
DelimJoin into this shape (identity projections between the nodes are left
out):

```
LogicalMaterializedCTE  __duckdb_delim_N         ← child 0: the outer side (customer), computed once
 │   └─ (child 1)
 ▼
LogicalMaterializedCTE  __duckdb_delim_dedup_M   ← child 0: D, computed once:
 │                                                   LogicalAggregate GROUP BY c.custkey
 │                                                     └─ LogicalCTERef(__duckdb_delim_N)
 │   └─ (child 1)
 ▼
LogicalComparisonJoin  (type is now plain LOGICAL_COMPARISON_JOIN)
 ├─ LogicalCTERef(__duckdb_delim_N)            ← the outer side, read back
 └─ ... subquery plan ...
       └─ LogicalCTERef(__duckdb_delim_dedup_M) ← was a LogicalDelimGet
```

In plain steps:

1. The join's type is changed from `LOGICAL_DELIM_JOIN` to
   `LOGICAL_COMPARISON_JOIN` (`delim_join_cte_rewriter.cpp:2660`).
2. Every `LogicalDelimGet` under the right side becomes a `LogicalCTERef`
   that reads the dedup CTE (`RewriteDelimScanReferences`, called at line
   2663).
3. The left input moves into a CTE named `__duckdb_delim_<n>` (line 2747),
   and the join reads it back through a `LogicalCTERef`.
4. D itself becomes a real `LogicalAggregate` that groups by the
   duplicate-eliminated columns over that CTE (line 2768). It is stored in a
   second CTE named `__duckdb_delim_dedup_<n>` (lines 2783-2790).

So in the default plan, D *is* an explicit node after all: a `GROUP BY`
inside a CTE. This is much closer to DiDi's `δ` than the DelimJoin form
described in §6.4. The DelimJoin form, with `PhysicalLeftDelimJoin` /
`PhysicalRightDelimJoin` and the `distinct` hash aggregate, is still what you
get with `SET delim_join_as_cte = false`.

The rewriter then checks that nothing was missed. `VerifyNoDelim`
(`delim_join_cte_rewriter.cpp:33-45`) throws `INTERNAL Error: Found
DELIM_JOIN after flattening dependent joins` if any DelimJoin or DelimGet is
left. It also runs its own clean-up passes, `GeneratedDedupRefEliminator` and
`GeneratedDomainJoinEliminator`, which remove joins against the dedup CTE
that add nothing. They do the same job for the CTE form that the optimizer's
`Deliminator` does for the DelimJoin form (§6.6, §6.7). Their time is even
reported under the same profiler key, `optimizer.deliminator`
(`delim_join_cte_rewriter.cpp:31`). The test file
`test/optimizer/deliminator.test` checks both forms; its patterns accept
either `DELIM_GET` or `CTE_SCAN`, and look for `CTE Name: __duckdb_delim`.

## 6.5 The general rewrite rules

Section 6.3 exercised two of the three pushdown rules that make up the core of
`FlattenDependentJoins`; here they are stated in full, algebraically, matching
[10, pp. 30–32] exactly (π, m, a, g are DiDi's letters for "some projection
list," "some aggregate function," "some existing group-by list"):

**Projection.**
```
D ⋈ᵈ πᵐ(Q)   ≡   πᵈᵐ(D ⋈ᵈ Q)
```
*Why it holds:* a projection just narrows or renames columns; it never
inspects more than one row at a time and never needs to know how many
distinct correlation parameters exist. Pushing the join underneath it only
requires remembering to also project out the correlation column `d`, so that
whatever consumes this subtree still has `d` available to eventually join
back against D (or against the next operator up the tree that still needs
it). DuckDB's implementation is `FlattenDependentJoins::PushDownProjection`
(`src/planner/subquery/flatten_dependent_join.cpp:876-904`), whose central
line is a direct transcription of the right-hand side: `AppendCorrelatedColumns`
appends `d` to the projection's existing expression list before recursing.

**Aggregation.**
```
D ⋈ᵈ Γᵐ:ᵃ/ᵍ(Q)   ≡   Γᵐ:ᵃ/ᵈᵍ(D ⋈ᵈ Q)
```
*Why it holds:* an aggregate groups its input rows by `g` and computes `a`
per group; adding `d` to the group-by list simply means "also group by which
distinct correlation parameter produced this row," which is exactly right,
because that's the granularity the enclosing dependent join needs its output
at. This is the transformation §6.3's step 2 walked through concretely — the
implementation is `FlattenDependentJoins::PushDownAggregate`
(`src/planner/subquery/flatten_dependent_join.cpp:906-1087`), and its key line
is `AddDelimColumnsToGroup(aggr, result.bindings)`
(`flatten_dependent_join.cpp:825-835`), which literally appends the
correlation column(s) to `aggr.groups` and to every entry of
`aggr.grouping_sets`.

> **A correctness subtlety worth naming, because Query B hits it directly.**
> `COUNT(*)` returns `0`, not `NULL`, for a group with zero input rows — but
> a plain `INNER` join between D and the newly-grouped aggregate would simply
> *drop* any `d` value that has zero matching `orders` rows, since an inner
> join has nothing to match against. For Query B that would silently make
> customers with no orders vanish from the result instead of showing
> `n_orders = 0`.
>
> DuckDB solves this in two parts. **Part one: keep the missing keys.**
> `PushDownAggregate` joins D (a `LogicalDelimGet`) to the grouped aggregate.
> It only uses an `INNER` join when dropping a missing key cannot change the
> answer:
>
> ```cpp
> // src/planner/subquery/flatten_dependent_join.cpp:945-949
> // dropping missing keys is only equivalent when the single recovered row would be all-NULL:
> // repeated empty sets need one row per set and GROUPING() outputs are non-NULL constants
> auto drop_missing_keys =
>     !any_join && propagate_null_values && empty_set_count == 1 && aggr.grouping_functions.empty();
> auto join_type = drop_missing_keys ? JoinType::INNER : JoinType::LEFT;
> ```
>
> `propagate_null_values` is cleared at the top of the function if any
> aggregate is volatile or does not propagate NULLs
> (`flatten_dependent_join.cpp:910-913`). `COUNT`'s `PropagatesNullValues()`
> is false, because it has a real value on empty input. So for Query B the
> join is a `LEFT` join, and every customer key survives.
>
> **Part two: replace the NULL with the empty-input value.** A `LEFT` join
> gives `NULL` in the `count` column for a customer with no orders, not `0`.
> So `PushDownAggregate` also looks for aggregates whose function has
> `FunctionNullHandling::SPECIAL_HANDLING` (`COUNT` is one,
> `flatten_dependent_join.cpp:971`). For those it does three things:
>
> 1. It adds a hidden `count(*)` "marker" aggregate. The marker is never
>    `NULL` on a real group, so a `NULL` marker means "this row was added
>    by the `LEFT` join".
> 2. It builds a tiny side plan that runs the same aggregate over an empty
>    input (`LogicalEmptyResult`). For `COUNT` that gives `0`.
> 3. It records both in `replacement_map`.
>
> At the end of flattening, `RewriteCorrelatedAggregates` replaces every
> reference to the aggregate result with a `CASE`:
>
> ```cpp
> // src/planner/subquery/rewrite_correlated_expressions.cpp:95
> // Replace this with CASE WHEN the aggregate group is missing THEN EMPTY_RESULT ELSE AGGREGATE END.
> ```
>
> So `n_orders` becomes `CASE WHEN marker IS NULL THEN 0 ELSE count END`.
> `SUM`/`AVG`/`MIN`/`MAX`, whose empty-group value genuinely is `NULL`, need
> neither part. When nothing above them can tell a `NULL` row from a
> missing row, they get the cheaper `INNER` join.
>
> §6.9.1 shows edge cases of this mechanism: repeated empty grouping sets,
> `GROUPING()`, and a `COUNT` that is itself a correlation key.

**Filter.**
```
D ⋈ᵈ σᵖ(Q)   ≡   σᵖ(D ⋈ᵈ Q)
```
*Why it holds:* this is the least surprising of the three — a selection just
drops rows that fail a predicate, and whether the input rows came from one
Apply-style evaluation or from a batch join makes no difference to which
individual rows pass. This is unconditionally true for any predicate `p` that
doesn't itself reference something that changes meaning per-D-row, which is
always the case here since `p`'s free references were already rewritten to
point at `d` by `rewrite_correlated_expressions.cpp` before this rule ever
runs. Implementation: `FlattenDependentJoins::PushDownChild`
(`src/planner/subquery/flatten_dependent_join.cpp:487-504`), the shared
recursion helper every operator-specific rule calls into for its single child.

`FlattenDependentJoins` applies these rules (plus several more, for joins,
cross products, set operations, window operators, and materialized CTEs —
see §6.9) recursively, bottom-up along the correlated side of the tree, until
recursion bottoms out at a leaf that no longer references the correlation
parameter, at which point `DecorrelateSubtreeInternal`
(`src/planner/subquery/flatten_dependent_join.cpp:322-403`) hits its
`LOGICAL_DEPENDENT_JOIN` case, calls `FinalizeDependentJoin`, and — as shown
in §6.4 — turns it into an ordinary, `LOGICAL_DELIM_JOIN`-tagged
`LogicalComparisonJoin`.

## 6.6 Two post-optimizations

DiDi presents two further refinements on top of the core three rules, both
concerned with tidying up artifacts the rewrite otherwise leaves behind.

**(1) Push pulled-up filters back into the join predicate.** When step 3 of
§6.3 turned `σ[o.custkey = d]` into an ordinary predicate sitting just above
the base scan, the *general* algorithm (for a more complex query with
additional non-equality filter clauses) can leave some filter clauses stranded
as a separate `σ` node just outside the (now-ordinary) join, instead of folded
directly into the join's own condition list. DiDi shows this explicitly for
its own TPC-H worked example (a filter `l.xp = m` and a filter `i.ok = d`,
each still a standalone `σ` immediately outside a join after the core
algorithm finishes) and then folds both directly into their enclosing join's
condition ([10, p. 34], "Post-Optimizations ⑥ — Push σ Down Into ⋈"). In
DuckDB's implementation this fold-in isn't a separate later pass at all — it
happens as part of the same `FinalizeDependentJoin` call that eliminates ⋈ᵈ,
via `LogicalComparisonJoin::ExtractJoinConditions`
(`src/planner/subquery/flatten_dependent_join.cpp:757-759`), which walks
exactly this kind of leftover predicate and turns it directly into a
`JoinCondition` rather than leaving a `LogicalFilter` node behind. Where DiDi
teaches this as a distinct step for pedagogical clarity, DuckDB's C++ folds it
into step 4.

**(2) "Decoupling": avoid materializing D at all.** The underlying idea, from
Neumann and Kemper's paper [12]: if the RHS subtree already produces, for free,
some column that is *equal to* D's correlation column wherever it matters,
there is no need to actually join against (and therefore no need to
materialize) D at all — the RHS can be filtered directly against that
already-available column instead, at the cost of one extra `IS NOT NULL`
check to preserve the original join's implicit NULL-filtering behavior. This
chapter can confirm this precisely against DuckDB's source in *two*
independent places, each catching the redundant D-materialization at a
different stage of the pipeline:

- **Logical level**, `src/optimizer/deliminator.cpp`, function
  `RemoveJoinWithDelimGet`: when the RHS of a `LOGICAL_DELIM_JOIN` contains an
  inner join directly against a `LogicalDelimGet` whose join columns are
  provably redundant (equal to columns already reachable another way), it
  deletes that join-with-DelimGet entirely — and compensates for the lost
  implicit NULL-filtering of the deleted equi-join by inserting exactly one
  `IS NOT NULL` filter per condition:
  ```cpp
  // src/optimizer/deliminator.cpp:230-238
  // Only add IS NOT NULL filter for regular equality/inequality comparisons
  // Do NOT add for DISTINCT FROM variants, as they handle NULL correctly
  if (cond.GetComparisonType() != ExpressionType::COMPARE_NOT_DISTINCT_FROM &&
      cond.GetComparisonType() != ExpressionType::COMPARE_DISTINCT_FROM) {
      auto is_not_null_expr =
          make_uniq<BoundOperatorExpression>(ExpressionType::OPERATOR_IS_NOT_NULL, LogicalType::BOOLEAN);
      is_not_null_expr->GetChildrenMutable().push_back(other_side.Copy());
      filter_expressions.push_back(std::move(is_not_null_expr));
  }
  ```
  This is precisely "derive the correlation column from a column already
  produced inside the subquery, at the cost of needing an extra not-NULL
  check" — stated exactly, in running code, not merely in spirit.
- **Physical level**, `PhysicalPlanGenerator::PlanDelimJoin`: even after the
  logical Deliminator pass, if every `LogicalDelimGet` under a
  `LOGICAL_DELIM_JOIN` turns out to have been eliminated by some other means,
  the physical plan generator notices there is nothing left that reads D and
  skips building the DISTINCT-computing `PhysicalHashAggregate` (§6.4)
  entirely, planning the node as a perfectly ordinary join instead:
  ```cpp
  // src/execution/physical_plan/plan_delim_join.cpp:36-41
  if (delim_scans.empty()) {
      // no duplicate eliminated scans in the delim side!
      // in this case we don't need to create a delim join
      // just push the normal join
      return plan;
  }
  ```

So DuckDB doesn't implement "decoupling" as a single named optimizer step the
way `deliminator` and `unnest_rewriter` are named passes — it implements the
*effect* Neumann and Kemper describe, redundantly, at two different points in
the pipeline, each catching cases the other might miss.

## 6.7 The optimizer-level cleanup — NOT the core algorithm

It bears repeating, because it is a common point of confusion: everything in
§6.3–§6.6 happens in the **planner**, right after binding (the
`Planner::CreatePlan` call shown in §6.3), before `Optimizer::Optimize` is
ever called. `src/optimizer/deliminator.cpp` and `src/optimizer/unnest_rewriter.cpp`
are real optimizer passes — they are passes #10 and #22 of the 45 fixed
passes that Chapter 7 §7.4 numbers — but they are
**follow-up cleanup of the join tree decorrelation already produced**, not
decorrelation itself. If you disable both of these passes with
`SET disabled_optimizers = 'deliminator,unnest_rewriter';`, a correlated
subquery still executes correctly — it just executes against a slightly
less-simplified join tree, with some redundant `DelimJoin`/`DelimGet`
structure left in place. Disable `FlattenDependentJoins` and there is, by
construction, no legal plan to execute at all, since no physical operator
exists for `LOGICAL_DEPENDENT_JOIN`.

A note on what these two passes actually see. With default settings, the
DelimJoins have already been rewritten into CTEs (§6.4.1), so `Deliminator`
finds no `LOGICAL_DELIM_JOIN` to work on. Its job for the default plan is done
earlier, by the CTE rewriter's own eliminators. Disabling `deliminator` still
matters, though: the rewriter checks the same switch and skips its own
clean-up when the pass is disabled (`cte_deliminator_enabled`,
`delim_join_cte_rewriter.cpp:2643-2644`). The descriptions of `Deliminator`
below apply to the DelimJoin form (`SET delim_join_as_cte = false`).
`UnnestRewriter` handles both forms; `RewriteInlineCTEDedupCandidate`
(`unnest_rewriter.cpp:391`) is its path for the CTE form.

**`Deliminator`** (`src/optimizer/deliminator.cpp`) scans the plan for every
remaining `LOGICAL_DELIM_JOIN`, and within its RHS subtree looks for joins
against a `LogicalDelimGet` that add no information beyond what's already
implied elsewhere (`Deliminator::FindCandidates` /
`Deliminator::RemoveJoinWithDelimGet`, `deliminator.cpp:103-259`). Section 6.6
already covered its headline behavior — deleting a redundant join-with-DelimGet
and compensating with an `IS NOT NULL` filter. It has one more trick worth
naming: `TrySwitchSingleToLeft` (`deliminator.cpp:426-479`) — DuckDB only ever
constructs `DelimJoin`s with `JoinType::SINGLE` (a join guaranteed to return
at most one matching row per outer row, used for scalar subqueries), but if
the RHS turns out to already be deduplicated by an aggregate whose `GROUP BY`
columns are a superset of the join's columns, the stronger `SINGLE`
uniqueness guarantee is unnecessary and the join can be downgraded to a
cheaper ordinary `LEFT` join.

**`UnnestRewriter`** (`src/optimizer/unnest_rewriter.cpp`) targets a narrower,
UNNEST-specific pattern. Recall from `FlattenDependentJoins::PrepareDependentJoinLeft`
that when a dependent join doesn't need full duplicate elimination
(`!op.perform_delim` — the case for single-row correlated subqueries that
contain `UNNEST`), the LHS is instead given a synthetic `row_number() OVER ()`
window column to use as its per-row key
(`CreateRowNumberWindow`, `flatten_dependent_join.cpp:406-422`, called from
`PrepareDependentJoinLeft` at line 460). This
produces a `LOGICAL_DELIM_JOIN` whose LHS is a `LogicalWindow` and whose RHS is
a chain of projections bottoming out in a `LogicalUnnest` reading from a
`LogicalDelimGet`. `UnnestRewriter::FindCandidates`
(`unnest_rewriter.cpp:122-241`) pattern-matches exactly this shape and rewires
the plan so the `LogicalUnnest` reads directly from the LHS's original
(pre-window) input instead of bouncing through a `DelimGet`, removing the
`LOGICAL_DELIM_JOIN` and the window operator entirely. This is real
plan-tree simplification, not decorrelation — the plan was already
executable without it — but it removes a join and a window function the
"generic" path introduced only as scaffolding.

## 6.8 A second worked example: escalating UNNEST complexity

The DuckDB SQL test suite has a small, deliberately-escalating test that
exercises exactly the boundary this chapter has been walking, three queries
at a time increasing in complexity:

```
# name: test/sql/subquery/scalar/test_unnest_subquery.test
# description: Test unnest in subqueries
# group: [scalar]

# uncorrelated
query I
SELECT (SELECT UNNEST([1]))
----
1
...
# correlated
query I
SELECT (SELECT UNNEST(i)) FROM (VALUES ([1])) tbl(i);
----
1
...
# now with an aggregate
query I
SELECT (SELECT SUM(k) FROM (SELECT UNNEST(i)) tbl(k)) FROM (VALUES ([1, 2, 3])) tbl(i);
----
```

- **Uncorrelated** (`SELECT (SELECT UNNEST([1]))`): the subquery references
  nothing from an enclosing scope at all — there is no free column reference,
  so the binder never builds a `LogicalDependentJoin` in the first place.
  This is the degenerate case where none of this chapter's machinery is
  invoked; the subquery is planned exactly as if it were written at the top
  level.
- **Correlated** (`SELECT (SELECT UNNEST(i)) FROM (VALUES ([1])) tbl(i)`):
  now `i` is a free reference into the outer `tbl`. This is a scalar
  (single-row-producing) correlated subquery whose body is `UNNEST`, which is
  exactly the `!perform_delim` case from §6.7 — `FlattenDependentJoins`
  attaches a `row_number()` window to the LHS instead of a full
  duplicate-eliminating join, and `UnnestRewriter` then simplifies away the
  resulting `LOGICAL_DELIM_JOIN`/window pair, leaving a plain `LogicalUnnest`
  reading straight from `tbl`.
- **Correlated with an aggregate** (`SELECT (SELECT SUM(k) FROM (SELECT
  UNNEST(i)) tbl(k)) FROM (VALUES ([1,2,3])) tbl(i)`): this adds exactly the
  ingredient Query B has and the previous line lacks — a `Γ` above the
  correlated body. Now the aggregation-pushdown rule from §6.5 applies in
  full: the dependent join is pushed through `Γ[SUM(k)]/()`, adding the
  correlation column to its `GROUP BY`, before it can reach the `LogicalUnnest`
  underneath and be eliminated. This is the same escalation DiDi's own TPC-H
  worked example rests on (a filter, then an aggregate, then decorrelation) —
  just demonstrated here with `UNNEST` instead of a `WHERE`-clause equality,
  and confirmed directly by a real, runnable test-suite entry rather than a
  hypothetical.

## 6.9 This is Neumann and Kemper's algorithm, not merely "inspired by" it

The paper is Neumann and Kemper's "Unnesting Arbitrary Queries"
[12].

DuckDB's own source cites this paper by name at the point where correlated
subquery planning starts:

```cpp
// src/planner/binder/query_node/plan_subquery.cpp:524-525
	// for a more in-depth explanation of this code, read the paper "Unnesting Arbitrary Subqueries"
	// also read "Improving Unnesting of Complex Queries"
```

(The comment gets the title slightly wrong: the paper says "Queries", not
"Subqueries". The second paper it names is a later follow-up
[19].) DiDi cites it too [10, p. 27, fn. 9]. The
correspondence is direct, not
loose: the paper's central technique — represent a correlated subquery as a
special dependent-join operator over a duplicate-free parameter domain, then
push that operator down through projection, aggregation, and selection until
it can be replaced by an ordinary join — is implemented essentially
one rewrite-rule-per-operator-type, with the same three core equivalences
this chapter derived in §6.5 as the load-bearing steps. `FlattenDependentJoins`
is not "inspired by" or "in the spirit of" Neumann–Kemper; it *is* an
implementation of it, in the same sense that DuckDB's join-order optimizer
(Chapter 8) *is* an implementation of Moerkotte and Neumann's DPhyp
algorithm (a different paper — "Dynamic Programming Strikes Back"
[20] — worth being careful not to conflate the two, since both are by
Thomas Neumann and both concern query optimization, but they solve different
problems: one turns dependent joins into ordinary joins, the other decides
what order to execute ordinary joins in).

That said, in fairness to how large the production code has grown since the
paper (and since DiDi's simplified teaching presentation of it): the real
`FlattenDependentJoins` handles a considerably larger surface area than the
three rules in §6.5 alone — `SEMI`/`ANTI`/`MARK`/`RIGHT_SEMI`/`RIGHT_ANTI`
join types, correlated references into materialized CTEs (`PushDownCTE`,
`PushDownCTERef`), the rewrite of every DelimJoin into CTEs
(`delim_join_cte_rewriter.cpp`, §6.4.1), full outer
joins (`PushDownFullOuterJoin`, with `COALESCE` to reconcile correlation
columns arriving from either side), and *nested* dependent joins handled via
a `parent`-linked chain of `FlattenDependentJoins` instances
(the constructor at `flatten_dependent_join.cpp:46-64` takes the `parent`;
`FlattenDependentJoins::DecorrelateIndependentSubtree` is at lines 294-301) so that a correlated subquery
inside a correlated subquery gets flattened correctly relative to both
enclosing scopes at once. None of that changes the core equivalences; it is
the same three rewrite rules, applied more times, in more contexts, with more
careful bookkeeping of which correlated column belongs to which enclosing
query. §6.9.1 lists the edge cases that this extra code handles.

### 6.9.1 Edge cases in the production code

The core of `FlattenDependentJoins` is what §6.3–§6.5 describe. Much of the
rest of the code handles edge cases. They follow one pattern: a rewrite rule
that is correct for the common shape would lose or duplicate rows for a
rarer shape, so the code checks for that shape. This section lists the cases
that change what a query returns or which plans DuckDB can build. Each item
names a test file you can read for the exact SQL.

**1. Empty grouping sets that repeat, and `GROUPING()`.** A query can say
`GROUP BY GROUPING SETS ((), ())`. That means "produce the grand total
twice". Inside a correlated subquery, the plain aggregate rule from §6.5
would produce it only once. From `test/sql/subquery/lateral/lateral_empty_grouping_sets.test`:

```sql
SELECT o.k, a.c
FROM (VALUES (7), (8)) AS o(k),
LATERAL (
    SELECT count(*) AS c
    FROM (VALUES (1)) AS i(k)
    WHERE i.k = o.k
    GROUP BY GROUPING SETS ((), ())
) AS a
ORDER BY ALL;
----
7	0
7	0
8	0
8	0
```

`PushDownAggregate` (§6.5) handles this. It counts the empty grouping
sets (`empty_set_count`). When there is more than one, it joins the
recovered rows against a small constant table with one row per empty set
(the `needs_multiplier` block, `flatten_dependent_join.cpp:1060-1084`). It
also gives `GROUPING()` the right value on recovered rows, through the same
marker-and-`CASE` mechanism used for `COUNT`.

**2. A `COUNT` that is itself the correlation key.** Sometimes the value a
nested subquery correlates on is the result of a `COUNT` from the level
above. Then the `CASE WHEN marker IS NULL THEN 0 ...` rewrite from §6.5 would
land inside the DelimJoin's `duplicate_eliminated_columns`. A DelimJoin key
must be a plain column. So
`FlattenDependentJoins::MaterializeCorrelatedAggregateRewrites`
(`flatten_dependent_join.cpp:630-660`) puts an identity projection under the
join's left side when this happens. The `CASE` goes into the projection, and
the key stays a plain column. From
`test/sql/subquery/scalar/test_correlated_aggregate_correlation_key.test`:

```sql
SELECT v, (SELECT (SELECT count(p)) FROM (SELECT s.w AS p FROM s WHERE s.w = t.v) sc0)
FROM t ORDER BY v NULLS LAST;
----
1	1
2	0
NULL	0
```

(with `t(v) = {1, 2, NULL}` and `s(w) = {1}`). The `0`s show that the
recovered `count` flows correctly into the inner subquery.

**3. Correlated `ASOF` joins.** `PushDownJoin` has a shortcut: when only one
side of an `INNER` join needs the correlation domain D, it pushes D into that
side alone. That is wrong for an `ASOF` join. An `ASOF` join picks *at most
one* match per left row from the whole right side, so with D only on the
right, a left row would pick one match across all domains. So
`selects_over_right` (`flatten_dependent_join.cpp:1184-1190`) turns the
right-only shortcut off for `ASOF`. From
`test/sql/subquery/lateral/lateral_asof_join.test`:

```sql
SELECT o.k, q.x, q.n
FROM (VALUES (1), (2)) o(k), LATERAL (
    SELECT v.x, t.n
    FROM (VALUES (2)) v(x)
    ASOF JOIN (SELECT o.k AS n) t ON v.x >= t.n
) q ORDER BY ALL;
----
1	2	1
2	2	2
```

The test guards against a bug where only the second row came back.

**4. A lateral join that reads a CTE.** When the right side of a dependent
join is a CTE that does not use D, `PushDownCTE` moves the CTE above the
join. Decorrelation must then go on with the same join, not restart it: a
restart would prepare the left side twice. That gives an internal error
("Conflicting or cyclic column binding replacement") or, for nested
laterals, wrong counts. From
`test/sql/subquery/lateral/lateral_cte_list_correlation.test`:

```sql
SELECT count(*), sum(n.k)
FROM o, LATERAL (SELECT * FROM (SELECT o.x) p,
                 LATERAL (WITH st(k) AS (VALUES (1), (2))
                          SELECT q.k FROM q WHERE q.k IN (SELECT k FROM st) AND q.s = o.g AND q.k < p.x)) n;
----
250	367
```

The test guards against a bug that returned `2497 3666`.

**5. `error()` does not block duplicate elimination.** Recall from §6.7:
when a subquery contains a volatile expression, DuckDB cannot share one
subquery result between equal outer values. It then numbers the outer rows
with `row_number()` instead of building D (`perform_delim = false`). The
function `error()` is marked volatile, so any subquery that used it would
lose duplicate elimination. But `error()` is deterministic in practice. The
helper `IsVolatileIgnoringError`
(`src/planner/binder/query_node/plan_subquery.cpp:451-472`) skips it when
deciding `perform_delim`. Its comment calls this a stop-gap until `error` no
longer has to be volatile.

Two notes on `perform_delim = false` itself, which §6.7 described only for
`UNNEST`. It is chosen in `PerformDuplicateElimination`
(`plan_subquery.cpp:485-510`) for two reasons: the subquery has volatile
expressions, or a correlated column has a type that cannot be grouped on
cheaply (`PerformDelimOnType`, lines 434-446: any `LIST`, or a `STRUCT` that
contains one). §6.8's `UNNEST(i)` example hits the second reason, because
`i` is a `LIST`.

**6. Smaller cases.** Each has its own test or benchmark:

- Correlated `IN` / `ANY` against a single `TUPLE` can be planned
  (`test/sql/subquery/any_all/test_correlated_tuple_in.test`).
- A correlated `IN` inside the `ON` clause of a `LEFT JOIN` under a MARK
  join keeps its correlated column
  (`test/sql/subquery/any_all/test_correlated_in_left_join_mark.test`).
- A filter on a view is pushed below the `SINGLE` and `MARK` joins built for
  the view's subqueries (`test/sql/subquery/scalar/view_filter_below_delim_join.test`).
- A `UNION ALL` of many uncorrelated subqueries is planned in linear
  time, not quadratic time (benchmark
  `benchmark/micro/optimizer/union_all_subquery_planning.benchmark`).
- `Deliminator::TrySwitchSingleToLeft` (§6.7) refuses to turn a `SINGLE`
  join into a `LEFT` join when the aggregate has more than one grouping set
  (`deliminator.cpp:466-469`), since several grouping sets can emit several
  rows per key.
- In the optimizer, the CTE join filter pusher
  (`src/optimizer/cte_join_filter_collector.cpp`) restricts the
  recursive partitions of CTE-decorrelated lateral joins. Without it, these
  plans are slower than the DelimJoin plans of the 1.5.x releases. This
  belongs to Chapter 7, but it exists because of the CTE form from §6.4.1.

Chapter 4 (§4.8.1) covers one more related case, in the binder: a
correlated subquery may refer to an outer `GROUP BY` *expression*, not
only to an outer column.

## 6.10 Bridge to Chapter 7 and Chapter 8

By the time `Optimizer::Optimize` receives the plan, every `LOGICAL_DEPENDENT_JOIN`
is gone, replaced by ordinary joins. With default settings these are plain
joins over `__duckdb_delim_*` CTEs (§6.4.1). With `delim_join_as_cte = false`
some are tagged `LOGICAL_DELIM_JOIN`, carrying the
`duplicate_eliminated_columns` metadata from §6.4.
Chapter 7 picks up here: `DELIMINATOR` and `UNNEST_REWRITER` are two of the
45 fixed passes it numbers. They are pass #10 and pass #22. Chapter 7 §7.4
does not count `REMOTE_PUSHDOWN`, because that pass runs from a separate
method, `OptimizeStatement`, not from the main pipeline. The whole book uses
§7.4's numbers.

The two passes are at these call sites:
`src/optimizer/optimizer.cpp:316` and `:395`. What matters here
is that — as §6.7 stressed — they clean up what decorrelation left behind
rather than performing decorrelation themselves.

It is tempting to then say "and after that, join ordering (Chapter 8) treats
every join in the tree identically, decorrelated or not" — this is *almost*
true, and worth stating precisely rather than loosely, because the source
draws a real distinction here. `RelationManager::ExtractJoinRelations`
(`src/optimizer/join_order/relation_manager.cpp:733-791`, the
`LOGICAL_DELIM_JOIN` case) does special-case `LOGICAL_DELIM_JOIN` (which, per
§6.4.1, only reaches it when `delim_join_as_cte` is off): rather than dissolving it into the surrounding join
hypergraph the way an ordinary `LOGICAL_COMPARISON_JOIN` is dissolved, it
recursively runs a *fresh, nested* join-order optimization over each of the
DelimJoin's own two children independently — so join ordering absolutely does
apply *inside* a decorrelated subquery's plan — and then adds the DelimJoin
as a single, opaque relation into the *enclosing* hypergraph, rather than
letting its LHS or RHS be freely reordered against unrelated tables at the
outer level. The comment trail in the `LOGICAL_DELIM_GET` case right after it
(`relation_manager.cpp:794-796`) records exactly why: DuckDB tried the more aggressive "reorder freely" version
once, it produced measurably worse plans in practice, and it was reverted —
history that is itself a small, concrete lesson in how far "just treat it
like any other join" can be pushed before it stops being a safe
simplification. So the accurate version of the bridge is: **once decorrelation
produces an ordinary join tree, join ordering optimizes each of that tree's
independently-decorrelated parts exactly as it would any other join
subplan — it just also keeps the join that resulted from decorrelation
together as one unit relative to everything around it**, which is precisely
the guarantee the physical `PhysicalDelimJoin`/`DelimGet` pairing (§6.4) needs
in order to still make sense once execution begins.
