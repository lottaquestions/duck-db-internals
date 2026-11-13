# Chapter 7, Part 18 — TOP_N, ROW_NUMBER_REWRITER, and TOP_N_WINDOW_ELIMINATION

## 7.18.0 Three passes, one family of idioms

This part covers three of the ~45 passes §7.4 catalogued, grouped together
because all three exist to answer a version of the same question — "the user
only wants the first few rows of something that's ordered; can we avoid
materializing and ordering the rest?" — at three different scopes:

- **`TOP_N`** (pass #32, `src/optimizer/topn_optimizer.cpp`) answers it at
  *global* scope: a plain `ORDER BY ... LIMIT N` over the whole result.
- **`ROW_NUMBER_REWRITER`** (pass #43, `src/optimizer/row_number_rewriter.cpp`)
  answers a much narrower question that only looks like it's about ordering:
  a bare, unpartitioned, unordered `ROW_NUMBER() OVER ()` that just wants
  sequential integers, computed over a base-table scan that already has a
  cheap way to produce them.
- **`TOP_N_WINDOW_ELIMINATION`** (pass #36,
  `src/optimizer/topn_window_elimination.cpp`) answers it at *per-group*
  scope: the classic "top N rows per partition" idiom, written as
  `ROW_NUMBER() OVER (PARTITION BY ...) <= N` and filtered with `QUALIFY` or
  an outer `WHERE`.

All three produce a plan that never runs the general-purpose window-function
evaluator or the general-purpose sort, when a narrower, purpose-built
mechanism would give the identical answer for less work. None of the three is
about *how* rows get physically ranked once the decision to rank them is
made — that's Chapter 15's `TopNHeap` and the vectorized window-aggregate
machinery, both left alone here. This part is about the optimizer-level
*decision* to avoid needing that machinery at all, and — for
`TOP_N_WINDOW_ELIMINATION` specifically — a genuinely large, regression-driven
test corpus that's worth walking through close to exhaustively, because it
maps out just how many different plan shapes a seemingly simple SQL idiom can
arrive in.

A methodological note before diving in: unlike some of this chapter's other
deep-dive parts, this test corpus does not lean on the `query I nosort <label>`
paired-EXPLAIN idiom (two blocks sharing a label, proving a rewrite fired by
checking their outputs hash identically) — none of the nine files this part
draws on use it. Instead, correctness here is overwhelmingly proven
*differentially*: run the query once with the optimizer on, once with
`SET disabled_optimizers = 'top_n_window_elimination'`, and either diff the
result sets directly (`EXCEPT`) or compare a boolean column of hand-computed
invariants (`count(*) = ... AND min(rn) = ... AND max(rn) = ...`). Given how
much plan-shape branching this pass does — struct-packing vs. not, late
materialization vs. not, semi-join vs. inner-join reconstruction — a
byte-identical EXPLAIN comparison would be far too brittle a tool; the tests
instead pin down *that the answer doesn't change*, and use a smaller number of
`<REGEX>` assertions on `EXPLAIN` output to pin down *which* plan shape was
chosen when that specific shape is the point of the test.

## 7.18.1 `TOP_N` — fusing `ORDER BY ... LIMIT` into a single operator

**Purpose, in one sentence:** recognize an `ORDER BY` immediately followed by
a `LIMIT` (with an optional, constant `OFFSET`) and fuse the two separate
`LogicalOrder` + `LogicalLimit` operators into one `LogicalTopN` operator, so
the physical layer can hand the query to `PhysicalTopN`'s bounded top-k heap
(Chapter 15, §15.7) instead of a full two-phase sort followed by a row-count
truncation.

### 7.18.1.1 The exact fusion condition

The whole gate lives in `TopN::CanOptimize`
(`src/optimizer/topn_optimizer.cpp:22`), and it is considerably narrower than
"there's an ORDER BY somewhere below a LIMIT." Reading it clause by clause:

```cpp
// src/optimizer/topn_optimizer.cpp:22
bool TopN::CanOptimize(LogicalOperator &op, optional_ptr<ClientContext> context) {
	if (op.type == LogicalOperatorType::LOGICAL_LIMIT) {
		auto &limit = op.Cast<LogicalLimit>();

		if (limit.limit_val.Type() != LimitNodeType::CONSTANT_VALUE) {
			// we need LIMIT to be present AND be a constant value for us to be able to use Top-N
			return false;
		}
		if (limit.offset_val.Type() == LimitNodeType::EXPRESSION_VALUE) {
			// we need offset to be either not set (i.e. limit without offset) OR have offset be
			return false;
		}
		// ...
```

**The `LIMIT` must be a plain constant — not a parameter, not `LIMIT N
PERCENT`, and not absent.** `LimitNodeType` (`bound_result_modifier.hpp:70`)
has five states: `UNSET`, `CONSTANT_VALUE`, `CONSTANT_PERCENTAGE`,
`EXPRESSION_VALUE`, `EXPRESSION_PERCENTAGE`. The check above accepts exactly
one of them. This directly answers the two questions worth nailing down with
certainty rather than guessing at:

- **A parameterized `LIMIT`** (`LIMIT $1`, or any limit whose value isn't
  foldable to a constant by the time this pass runs) binds as
  `EXPRESSION_VALUE`, which fails the check outright — a prepared query with a
  placeholder `LIMIT` never gets `TOP_N` fusion and always pays for a full
  sort, no matter how small the eventual limit turns out to be at execution
  time.
- **`LIMIT N PERCENT`** binds as `CONSTANT_PERCENTAGE`, also rejected — this
  makes sense mechanically as well as by-the-code: a percentage limit's row
  count isn't known until the child's cardinality is known, which for a
  streaming, bounded top-k heap would mean the heap's bound couldn't be fixed
  in advance the way a plain numeric `LIMIT` fixes it.
- **`OFFSET` may be absent (`UNSET`) or a constant (`CONSTANT_VALUE`), but not
  an expression.** `LIMIT 4 OFFSET 3` is fine; `LIMIT 4 OFFSET (SELECT ...)`
  is not.
- **`OFFSET` with no `LIMIT` at all never fuses**, and the reason is visible
  directly in the type check: with no `LIMIT` clause present, `limit_val`'s
  type is `UNSET`, not `CONSTANT_VALUE`, so the very first `if` in the
  function already returns `false` — there's no special-case "offset-only"
  branch to reason about; it's simply that the required precondition
  (a constant limit) doesn't hold.

### 7.18.1.2 The cardinality-based fallback to full sort

The overview chapter (§7.4, pass #32) flagged this as "conditional on a cost
heuristic, not automatic" without pinning the exact numbers down. The real
check is a *two-part* condition — both parts must hold, or fusion proceeds
regardless of how large the limit is relative to the child:

```cpp
// src/optimizer/topn_optimizer.cpp:36
auto child_op = op.children[0].get();
if (context) {
	// estimate child cardinality if the context is available
	child_op->EstimateCardinality(*context);
}

if (child_op->has_estimated_cardinality) {
	// only check if we should switch to full sorting if we have estimated cardinality
	auto constant_limit = static_cast<double>(limit.limit_val.GetConstantValue());
	if (limit.offset_val.Type() == LimitNodeType::CONSTANT_VALUE) {
		constant_limit += static_cast<double>(limit.offset_val.GetConstantValue());
	}
	auto child_card = static_cast<double>(child_op->estimated_cardinality);

	// if the limit is > 0.7% of the child cardinality, sorting the whole table is faster
	bool limit_is_large = constant_limit > 5000;
	if (constant_limit > child_card * 0.007 && limit_is_large) {
		return false;
	}
}
```

Both conditions are computed against `limit + offset` (the actual number of
rows the top-k heap would have to retain, not just the number ultimately
returned to the caller):

1. **`constant_limit > child_card * 0.007`** — the effective limit exceeds
   0.7% of the child's estimated cardinality, and
2. **`constant_limit > 5000`** — the effective limit is bigger than 5000 rows
   outright.

**Both must be true simultaneously** for `TOP_N` to decline and leave the
plan as a separate `LogicalOrder` + `LogicalLimit` (which the physical planner
then turns into a full sort, Chapter 15, followed by a plain row-count
truncation). A limit that's a large *fraction* of a small table but numerically
tiny (say, `LIMIT 100` against a 10,000-row child — 1% of the table, but only
100 absolute rows) still fuses, because `limit_is_large` is false. Symmetrically,
an enormous limit against an enormous table where the limit is still a tiny
*fraction* of the whole (say, `LIMIT 6000` against a 100-million-row table)
still fuses, because `6000 > child_card * 0.007` is false even though
`6000 > 5000` is true. The heuristic is specifically trying to catch the case
where the top-k heap would end up holding a large *absolute* number of
candidates that is *also* not a small slice of the input — the case where a
bounded heap's per-row comparison-and-maybe-insert overhead (`O(log k)` per
row, `k = limit + offset`) stops being cheaper than the two-phase merge sort's
`O(n log n)` over the whole input with `k` large enough that the constants
favor the full sort.

The `test/optimizer/topn/topn_optimizer.test` file's last two cases are a
direct, worked confirmation of exactly this arithmetic:

```sql
-- test/optimizer/topn/topn_optimizer.test:70-81
create table large_table as select range a from range(100000);

-- LIMIT 100: 100 > 5000 is false -> limit_is_large is false -> fuses regardless
EXPLAIN select * from large_table order by a limit 100;
----
logical_opt	<REGEX>:.*TOP_N.*

-- LIMIT 10000: 10000 > 5000 (true) AND 10000 > 100000*0.007=700 (true) -> declined
EXPLAIN select * from large_table order by a limit 10000;
----
logical_opt	<!REGEX>:.*TOP_N.*
```

100,000 rows, and the crossover between "still worth a bounded heap" and
"just sort the whole thing" sits somewhere between `LIMIT 100` and
`LIMIT 10000` — consistent with `0.007 * 100000 = 700` as the fractional
threshold once the limit also clears the flat 5000-row floor.

One more detail this heuristic depends on: `EstimateCardinality` is called
*on the child* right there in `CanOptimize`, not read from a
pre-computed field — this pass runs early (#32 in the pipeline, before
`STATISTICS_PROPAGATION` at #34), so it has to ask for its own cardinality
estimate on demand rather than relying on a number some earlier pass already
attached to the tree.

### 7.18.1.3 Locating the `ORDER BY`: projections are transparent, everything else isn't

After the limit/offset/cardinality checks pass, `CanOptimize` still has to
confirm there's actually an `ORDER BY` to fuse with, and it does so by walking
straight down through any chain of `LogicalProjection`s sitting directly under
the `LIMIT`:

```cpp
// src/optimizer/topn_optimizer.cpp:56
while (child_op->type == LogicalOperatorType::LOGICAL_PROJECTION) {
	D_ASSERT(!child_op->children.empty());
	child_op = child_op->children[0].get();
}

return child_op->type == LogicalOperatorType::LOGICAL_ORDER_BY;
```

Any number of plain projections between the `LIMIT` and the `ORDER BY` are
transparent to this check — which is exactly why a CTE reference chain like
`cte1 → cte2 → cte3` (each adding its own `SELECT *, ... AS alias` projection
layer) still fuses in the test file's fourth case, and why `SELECT * FROM (...
ORDER BY x) AS cte LIMIT 10` fuses even though the subquery boundary itself
compiles down to a projection. Anything *other* than a projection interposed
between `LIMIT` and `ORDER BY` — a filter, a join, an aggregate — blocks the
match outright, because the loop only knows how to see through
`LOGICAL_PROJECTION` nodes.

`Optimize()` (`topn_optimizer.cpp:159`) does the actual surgery once
`CanOptimize` says yes: it walks down through the same projection chain,
collecting each projection it passes so it can rebuild them afterward, splices
the `LogicalOrder`'s child directly under a new `LogicalTopN` (constructed from
the order's sort keys plus the limit and offset constants), estimates the new
node's cardinality as `min(limit_val, child's own estimate)` — the fused node
can never produce more rows than the limit asked for, so that's a strictly
better upper bound than whatever the child's own estimate was — and then
re-wraps the result in the collected projections, outermost last:

```cpp
// src/optimizer/topn_optimizer.cpp:186-193
auto topn = make_uniq<LogicalTopN>(std::move(order_by.orders), limit_val, offset_val);
topn->unpruned_offset = limit.unpruned_offset;
topn->AddChild(std::move(order_by.children[0]));
auto cardinality = limit_val;
if (topn->children[0]->has_estimated_cardinality && topn->children[0]->estimated_cardinality < limit_val) {
	cardinality = topn->children[0]->estimated_cardinality;
}
topn->SetEstimatedCardinality(cardinality);
```

### 7.18.1.4 A bonus mechanic: dynamic filter pushdown through `LogicalTopN`

Once a `LogicalTopN` exists, `TopN::Optimize` immediately tries a second,
separate rewrite on it — `PushdownDynamicFilters`
(`topn_optimizer.cpp:66`) — worth covering here even though §7.4's one-liner
doesn't mention it, because it's a genuinely distinct piece of mechanism
riding along with the same pass. If the sort key is a single, plain
`BoundColumnRefExpression` of a numeric or `VARCHAR` type (any compound sort
key, or a sort key that's an expression rather than a bare column, disqualifies
this immediately), the pass looks for a base-table scan the column traces back
to through `JoinFilterPushdownOptimizer::GetPushdownFilterTargets` and, if it
finds one whose cast chain back to the raw scanned column is order-preserving
(no `TRY_CAST`, no non-integral cast — checked via `RuntimeFilterCastUtil`),
installs a **dynamic filter**: a placeholder comparison (`C < boundary`,
`C <= boundary`, `C > boundary`, or `C >= boundary`, direction chosen from
`ASCENDING`/`DESCENDING` and whether there's more than one sort key) pushed
straight into the scan's `table_filters`, with its boundary constant
initialized to the type's minimum value and — critically — actually *updated
at execution time* as the top-k heap's current worst-surviving candidate
tightens. This is a runtime filter in the same family as the one
`JOIN_FILTER_PUSHDOWN` (#42) installs for hash joins: the `LogicalTopN`'s own
child scan gets to skip rows it can already prove could never make it into
the top-`N`, well before those rows would otherwise reach the heap at all.
`NULLS FIRST` orderings get an `OR IS NULL` wrapped around the pushed
comparison, since a `NULL` sort key can still be a winner under that null
ordering and must not be filtered away by a plain numeric bound.

### 7.18.1.5 Walking `topn_optimizer.test` and `complex_top_n.test`

`test/optimizer/topn/topn_optimizer.test`, case by case:

- **Standalone `LIMIT`** (`SELECT i FROM integers LIMIT 4`) — no `ORDER BY`
  exists at all, so `CanOptimize`'s final check (walk down to an
  `ORDER BY`) fails; the plan keeps a plain `LOGICAL_LIMIT`.
- **Standalone `ORDER BY`** (no `LIMIT`) — there's no `LogicalLimit` node in
  the tree for `CanOptimize` to even be called on (it only ever fires when
  `op.type == LOGICAL_LIMIT`); the plan keeps `LOGICAL_ORDER_BY`.
- **`ORDER BY i LIMIT 4`** — both a constant limit and an `ORDER BY`
  immediately below it: fuses to `TOP_N`.
- **`ORDER BY i LIMIT 4 OFFSET 3`** — `offset_val` is `CONSTANT_VALUE`, which
  is explicitly one of the two accepted states: still fuses.
- **`ORDER BY i OFFSET 3`** (no `LIMIT`) — as derived above from
  `LimitNodeType::UNSET`, does not fuse; the offset is applied by a plain
  `LogicalLimit` with no numeric limit, sitting over a full `LogicalOrder`.
- **`SELECT * FROM (SELECT * FROM range(100000000) AS _(x) ORDER BY x) AS cte
  LIMIT 10`** — the subquery boundary is a projection, which the
  down-traversal sees through: fuses, even at a hundred million rows,
  because `LIMIT 10` is nowhere near the 5000-row floor.
- **The four-level CTE chain** (`cte1` with the `ORDER BY`, `cte2` and `cte3`
  each adding derived columns as more `SELECT *, ...` layers, `LIMIT 4` on
  the outermost reference) — a deliberately deep stress test of "any number
  of interposed projections is still transparent": still fuses. This case is
  gated behind `require no_alternative_verify`, flagging that DuckDB's debug
  alternate-verification machinery (which re-runs plans through a second code
  path to catch inconsistencies) is known to choke on a projection chain this
  deep, and the test suite explicitly opts out of it rather than working
  around it.
- **`large_table`, 100,000 rows, `LIMIT 100` vs. `LIMIT 10000`** — the
  cardinality-fallback pair already walked through in §7.18.1.2.

`test/optimizer/topn/complex_top_n.test` is doing something different from
the mechanics-focused first file: it's a **correctness stress test under
plan complexity**, not an enumeration of fusion-eligibility edge cases. Both
of its queries bury an `ORDER BY ... LIMIT 50 OFFSET 50` three layers deep
inside a CTE (`CTE`) that itself feeds three further `LEFT JOIN`ed
subqueries, over four generated tables (`CUSTOMERVIEW`, `OrderView`,
`OrderItemView`, `CREDITCARDVIEW`) built with `random()` under a fixed
`seed 0.42` for reproducibility. The first query variant is a plain `query
III` result-hash check (`351 values hashing to
4cba82d74d151aef44c26dc8693f5dca`) — proving the *answer* survives whatever
rewrite fired, without asserting anything about the plan shape. The second
variant (same query, minus `CUSTOMER_ID` from the `ORDER BY` list) is an
`EXPLAIN` check asserting `physical_plan` — not `logical_opt` — contains
`TOP_N`, i.e. that the fused `LogicalTopN` really does survive all the way
through physical-plan generation into a `PhysicalTopN` node, even sitting
under three subsequent joins and reached through a materialized CTE
reference. Between the two files, the pairing is deliberate: the first file
proves the *mechanics* (which exact syntactic shapes fuse), and the second
proves the mechanics still hold up once the fused node is embedded in a
plan complex enough that a less careful implementation might have lost
track of it somewhere in the join tree.

## 7.18.2 `ROW_NUMBER_REWRITER` — mapping onto a storage-layer virtual column

**Purpose, in one sentence:** recognize a bare `ROW_NUMBER() OVER ()` — no
`PARTITION BY`, no `ORDER BY`, nothing but sequential numbering — sitting
directly over a base-table scan, and replace the entire `LogicalWindow`
operator with a reference to a `row_number` **virtual column** the storage
layer can already produce directly, instead of running the general window-
function evaluator to compute the same sequence the hard way.

### 7.18.2.1 What "virtual column" means here, concretely

This is not a metaphor for "computed lazily" — it's a literal, storage-layer
mechanism. `COLUMN_IDENTIFIER_ROW_NUMBER` is a real, reserved column
identifier (`src/include/duckdb/common/constants.hpp:46`), and
`RowGroup`'s column-fetch path has a dedicated branch for it
(`src/storage/table/row_group.cpp:168`), backed by its own concrete
`ColumnData` subclass, `RowNumberColumnData`
(`src/storage/table/row_number_column_data.cpp:7`) — a column that doesn't
store bytes on disk at all; it derives every value on demand from the row
group's own position within the table and the current transaction's
visibility state (which is exactly why the heavy second half of
`row_number_virtual_column.test` is a battery of `BEGIN`/`INSERT`/`DELETE`/
`ROLLBACK` cases: correct row numbering has to account for transaction-local
insertions, deletions that make earlier row numbers vanish mid-scan, and a
snapshot reader's view staying stable across a concurrent writer's commits).
A `LogicalGet`'s `virtual_columns` map is how a scan advertises which of
these it can supply, and this pass's entire job is to check for that
capability, then splice a reference to it into the plan instead of a real
window computation.

### 7.18.2.2 The eligibility gate

`RowNumberRewriter::CanOptimize` (`src/optimizer/row_number_rewriter.cpp:10`)
is short, and every clause in it maps directly onto one of the test file's
sections:

```cpp
// src/optimizer/row_number_rewriter.cpp:10
bool RowNumberRewriter::CanOptimize(LogicalOperator &op) {
	if (op.type != LogicalOperatorType::LOGICAL_WINDOW) {
		return false;
	}
	if (op.expressions.size() != 1) {
		return false;
	}

	auto &expression = op.expressions[0];
	if (expression->GetExpressionType() != ExpressionType::WINDOW_ROW_NUMBER) {
		return false;
	}
	auto &window_expr = expression->Cast<BoundWindowExpression>();
	if (!window_expr.Partitions().empty() || !window_expr.OrderBy().empty()) {
		return false;
	}

	// Should be followed by a get
	auto &window_ch = op.children[0];
	if (window_ch->type != LogicalOperatorType::LOGICAL_GET) {
		return false;
	}

	auto &get = window_ch->Cast<LogicalGet>();

	// the get must support the row_number virtual column
	if (get.virtual_columns.find(COLUMN_IDENTIFIER_ROW_NUMBER) == get.virtual_columns.end()) {
		return false;
	}

	// only rewrite for a base-table scan
	if (!get.GetTable()) {
		return false;
	}

	// cannot rewrite if there are table filters pushed into the scan
	if (get.table_filters.HasFilters()) {
		return false;
	}

	return true;
}
```

Five independent conditions, every one of them load-bearing:

1. **Exactly one window expression, and it must be `WINDOW_ROW_NUMBER`.**
   Not `RANK()`, not `DENSE_RANK()`, not a second window function riding
   alongside it in the same `LogicalWindow` — only a lone, bare
   `ROW_NUMBER()`.
2. **No `PARTITION BY`, no `ORDER BY`.** The virtual column numbers rows in
   storage/scan order, with no notion of grouping or an alternate sort — the
   moment either clause is present, this is a different, more general
   computation the virtual column cannot answer.
3. **The window's immediate child must be a `LogicalGet`**, not a filter, not
   a projection, not a join — nothing may sit between the window and the
   scan.
4. **That scan must actually expose a `row_number` virtual column.** This is
   the guard the inline comment at line 41 explains in detail: a table
   function like `read_duckdb` *also* populates a `virtual_columns` entry
   under this same identifier, but with **file-local** semantics (numbering
   rows within whatever file it's currently reading) rather than the SQL
   standard's per-query window semantics. Mapping `ROW_NUMBER() OVER ()`
   directly onto that column would silently return the wrong numbers for
   such a scan — this is exactly what issue #23586 was about, and the fifth
   condition (`get.GetTable()`, i.e., "this is a genuine base-table scan, not
   an arbitrary table function") is the fix: it restricts the whole rewrite
   to real, catalog-backed tables, whose row-number virtual column really is
   backed by `RowNumberColumnData`'s transaction-aware, whole-scan-relative
   semantics.
5. **No table filters may already be pushed into the scan.** The row-number
   virtual column counts *absolute* row positions as the underlying storage
   sees them; if a filter has trimmed which of those rows the scan actually
   surfaces, "row 47 of the storage" and "the 47th row this scan yields" stop
   being the same number, and the rewrite would silently return wrong
   row-numbers for anything past the first filtered-out row.

### 7.18.2.3 The rewrite itself: replacing an operator with a wider `LogicalGet`

`RewriteGet` (`row_number_rewriter.cpp:71`) does the actual splice: it adds
`COLUMN_IDENTIFIER_ROW_NUMBER` to the `LogicalGet`'s column-id list (via
`AddColumnId`), appends `BIGINT` to the scan's output types, and then — this
is the interesting part — builds a **column-binding remap** rather than
rewriting every downstream expression by hand. Every column binding the old
`LogicalWindow` used to produce is mapped either straight through to the
underlying scan's existing bindings, or, for the position that used to be the
window's `ROW_NUMBER()` output, onto the scan's newly-appended virtual
column:

```cpp
// src/optimizer/row_number_rewriter.cpp:89-100
for (idx_t i = 0; i < old_window_bindings.size(); i++) {
	ColumnBinding target_binding;
	if (i < child_bindings.size() - 1) {
		// Map existing columns through
		target_binding = child_bindings[i];
	} else {
		// Map the window expression to the virtual ROW_NUMBER column
		target_binding = child_bindings.back();
	}
	replacer.replacement_bindings.emplace_back(old_window_bindings[i], target_binding);
}
```

The `LogicalWindow` node itself is then discarded outright
(`return std::move(window.children[0])` — the function's return value is the
*scan*, not the window), and a single `ColumnBindingReplacer` walks the rest
of the plan tree afterward, retargeting every reference to what used to be
the window's output column onto the scan's new virtual column directly. The
net effect: `EXPLAIN` on a rewritten query shows no `WINDOW` operator at all —
just a wider `LogicalGet`.

### 7.18.2.4 Walking both test files

`test/optimizer/row_number_virtual_column.test` has the widest single-file
coverage of any pass in this part, so it's worth going through it in
thematic chunks rather than line by line:

- **The virtual column is not user-visible as a name.**
  `SELECT a, row_number FROM foo` fails to bind at all
  (`Binder Error: Referenced column "row_number" not found in FROM clause`),
  and `SELECT a AS row_number FROM foo` is just an ordinary column aliased to
  that string — the rewrite's target is a reserved internal identifier, never
  something SQL text can name directly; it only ever gets wired in by this
  optimizer pass, after a real `ROW_NUMBER() OVER ()` call has already bound.
- **A `DELETE` before the query changes the answer correctly.** After
  `delete from foo where a < 5000` on a 130,000-row table,
  `row_number() over(), a, rowid from foo order by a desc limit 1` returns
  `125000 | 129999 | 129999` — 125,000 surviving rows after the delete, and
  the row-number for the last one by insertion order lines up with its
  `rowid`. This is the load-bearing case for condition 5 above: no filter is
  pushed down here (it's an `ORDER BY ... LIMIT` over the whole table, fused
  separately by `TOP_N`), so the rewrite still applies and still gets the
  post-delete count right, because the virtual column itself, not a static
  precomputed sequence, is transaction-aware.
- **The rewrite survives being used alongside a join, unnesting, and
  `CASE`.** `EXPLAIN SELECT t.row_num, t.a, t2.c FROM (SELECT ..., ROW_NUMBER()
  OVER () AS row_num FROM test) t JOIN test2 t2 ON t.b = t2.b LIMIT 10` still
  shows no `WINDOW` node — the binding replacement propagates the virtual
  column reference straight through the join above it. `CASE WHEN
  row_number() OVER () = 2 THEN ... END` and `SELECT row_number() OVER (),
  unnest(num) FROM numbers` both correctly produce `1111/2222/1111` and
  `1/1/1/1` respectively (`unnest` runs *after* the window in this shape, so
  every unnested output row of the same source row shares the same row
  number 1 — the source table has one row, so the virtual column of the base
  scan really is a constant 1 here) — while the second numbers test (row
  number computed *after* an already-unnested subquery) correctly counts
  1/2/3/4, since in that shape the scan being numbered is the four-row
  unnested intermediate, not the one-row `numbers` table.
- **A join purely keyed on two independent row-numbered subqueries** still
  eliminates both windows: `EXPLAIN SELECT f.row_num AS foo_row, b.row_num AS
  bar_row, ... FROM (SELECT row_number() OVER () AS row_num, a FROM foo) f
  JOIN (SELECT row_number() OVER () AS row_num, b FROM bar) b ON f.row_num =
  b.row_num` shows no `WINDOW` — each side's rewrite is entirely independent
  of the other's, since `CanOptimize` only ever looks at one `LogicalWindow`
  and its immediate `LogicalGet` child at a time.
- **Multi-column, multi-typed tables** (`INT`/`DATE`/`VARCHAR`) confirm the
  rewrite is agnostic to what else the scan projects, and to where in the
  `SELECT` list `ROW_NUMBER()` appears — first, last, or in the middle of the
  output column list all produce correct numbering, because the binding
  remap operates purely on `ColumnBinding` identity, not projection position.
- **The negative case, twice over.** `filter_test` with a `WHERE col1 > 2`
  filter — even one that will be pushed down into the scan by
  `FILTER_PUSHDOWN` (pass #6) long before this pass (#43, near the very end
  of the pipeline) ever runs — keeps the `WINDOW` operator in the plan, and
  the *correct* answer is a freshly-renumbered sequence starting at 1 for the
  surviving rows (`gamma→1, delta→2, epsilon→3`), which is exactly what the
  general window evaluator computes and exactly what the virtual column
  would have gotten wrong (it would have reported each surviving row's
  absolute, pre-filter storage position instead). The same shape repeated
  inside a transaction with transaction-local inserted rows confirms the
  guard holds regardless of whether the filtered-out or filtered-in rows are
  committed or local.
- **`read_duckdb`, deliberately kept unrewritten.** The block introduced by
  the issue #23586 comment attaches a fresh on-disk database, creates a
  two-row table in it, detaches it, and re-reads it through
  `read_duckdb(...)` — a table *function*, not a catalog-backed base table.
  `ROW_NUMBER() OVER ()` over that scan correctly returns `1, 2` (SQL window
  semantics), and the `EXPLAIN` directly beside it confirms the `WINDOW`
  operator survives — condition 4's `get.GetTable()` check is precisely what
  keeps this scan on the general, correct path instead of wrongly reaching
  for `read_duckdb`'s own file-local row-number virtual column.
- **The parallel-scan stress test (issue #25553).** A `foreach`-driven matrix
  over `{parallelism setting} × {thread count} × {row count straddling
  vector-size boundaries: 2047, 2048, 2049, 2050, 130000}` checks
  `count(*) = row_count AND min(rn) = 1 AND max(rn) = row_count AND
  count(DISTINCT rn) = row_count` after creating the table fresh each time —
  i.e., that parallel scan assignments split across exactly a vector-size
  boundary (2048 is DuckDB's default vector size) still hand out a dense,
  gap-free, duplicate-free sequence of row numbers, which is a real
  correctness hazard for a rewrite whose whole premise is deriving numbers
  from *storage position*, when storage positions get divided across
  parallel scan tasks. Each variant is immediately followed by an `EXPLAIN`
  assertion (`<!REGEX>:.*WINDOW.*`) confirming the fast path stayed engaged
  throughout — "the correctness check must not be passing because a
  correctness bug silently forced a fallback to the slow, always-correct
  path" is exactly the failure mode a differential test like this is
  designed to catch, and the explicit `EXPLAIN` alongside it rules that out.
  The same matrix is then run again after a `DELETE FROM rn_batches WHERE i
  < 5` (mid-table deletion), inside a `BEGIN`/`ROLLBACK` (transaction-local
  inserts that must vanish again), and after deleting the *entire visible
  prefix* (`DELETE ... WHERE i < 2048`, exercising the "the base rows this
  assignment starts numbering from have all just become invisible" edge
  case) — the `coalesce(min(rn), 1)`/`coalesce(max(rn), 0)` pattern in that
  last block exists specifically to make the assertion still type-check when
  the row count could legitimately be zero.
- **Snapshot isolation.** The final two-connection block
  (`con1`/`con2`) opens a transaction on `con1`, has `con2` delete most of the
  table and then insert a fresh, larger range while `con1`'s transaction is
  still open, and checks that `con1`'s row-numbering — still driven by the
  virtual column, confirmed by the surrounding structure of the test — sees
  neither the concurrent delete nor the concurrent insert until after its own
  `COMMIT`. This is the single clearest piece of evidence in the whole
  corpus that `RowNumberColumnData` is genuinely wired into DuckDB's MVCC
  visibility machinery, not just a cache of the table's current physical row
  count.

`test/optimizer/row_number_virtual_column_local.test` is a smaller,
narrower companion focused specifically on **transaction-local writes without
a commit in between**, run for four separate table shapes: local inserts
alone, local deletes alone, local inserts and deletes combined inside the
same open transaction, and a multi-column table with a `WHERE` filter that
(as established above) must keep using the general window path even here.
None of its cases include an `EXPLAIN` assertion — its entire job is pure
result-correctness under transaction-local churn, treating the plan-shape
question as already settled by the first file and using this one purely to
harden the row-numbering *values* against every local mutation ordering that
matters.

## 7.18.3 `TOP_N_WINDOW_ELIMINATION` — the general, per-group top-N rewrite

**Purpose, in one sentence:** recognize the "top N rows per group" SQL idiom
— `ROW_NUMBER() OVER (PARTITION BY ...) <op> N`, filtered immediately
afterward, most commonly spelled with `QUALIFY` — and replace the whole
`LogicalFilter → [projections] → LogicalWindow` sandwich with a
`LogicalAggregate` using DuckDB's `min`/`max`/`arg_min`/`arg_max` family of
aggregates (optionally their list-returning `_n` variants for `N > 1`),
avoiding the cost of assigning and then discarding a row number for every
row in every partition.

### 7.18.3.1 Relationship to `ROW_NUMBER_REWRITER`: not overlapping, not a
generalization — a different axis entirely

The task of reading both files carefully enough to state this precisely
turns up an answer that's easy to get subtly wrong by pattern-matching on
names alone: **these two passes are not variations on the same idea, and
`TOP_N_WINDOW_ELIMINATION` is not "the general case of which
`ROW_NUMBER_REWRITER` is the special case."** They solve genuinely different
plan shapes, triggered by genuinely different, mutually exclusive
preconditions:

- `ROW_NUMBER_REWRITER` requires the window to have **no `PARTITION BY` and
  no `ORDER BY`** (`row_number_rewriter.cpp:23`) — it exists for exactly the
  case where the numbering carries no semantic ranking information at all,
  just "which position did this row land in during the scan."
- `TOP_N_WINDOW_ELIMINATION` requires the opposite: an `ORDER BY` is
  mandatory (`window_expr.OrderBy().size() != 1` is an immediate rejection,
  `topn_window_elimination.cpp:679`) — the whole point is a rewrite that
  *depends on* there being a genuine per-partition ranking to replace with a
  targeted `min`/`max`/`arg_min`/`arg_max`.

A query could never satisfy both passes' preconditions at once: one demands
an absent `ORDER BY`/`PARTITION BY`, the other demands a present `ORDER BY`.
They also run at very different points in the pipeline — `#36` for
`TOP_N_WINDOW_ELIMINATION`, right after `STATISTICS_PROPAGATION`, versus `#43`
for `ROW_NUMBER_REWRITER`, near the very end — and `TOP_N_WINDOW_ELIMINATION`
explicitly requires the `core_functions` extension to be loaded
(`topn_window_elimination.cpp:222`, since `arg_min`/`arg_max`/`arg_min_n`/
`arg_max_n` and their `_nulls_last` variants all live in that extension —
`extension/core_functions/aggregate/distributive/functions.json`), a
dependency `ROW_NUMBER_REWRITER` doesn't share at all, since it never invokes
any aggregate function — it only rewires column bindings. The honest
one-sentence relationship: they are siblings that both attack the general
"eliminate an unnecessary window computation" problem, aimed at two disjoint
special cases of `ROW_NUMBER()`'s use, not a specific/general pair.

### 7.18.3.2 The eligibility gate, in full

`TopNWindowElimination::CanOptimize`
(`src/optimizer/topn_window_elimination.cpp:551`) is the longest single
eligibility check covered in this part, and every clause maps onto a test
this section will come back to.

`CanOptimize` is a public `static` method that returns `optional_idx` (an
index that may be empty). An empty value means "no". Otherwise the value is
the effective limit `N`, the number of rows kept per group (so `rn < 3` gives
2). Because the method is `static`, it does not use any column-reference
state of the pass. A small helper, `ColumnReferenceExtractor`
(`topn_window_elimination.cpp:542-549`), collects the column references of
one expression at a time. The pass class itself derives from
`LogicalOperatorVisitor`.

**The filter shape.** The pattern must start at a `LogicalFilter` with
**exactly one** expression (`filter.expressions.size() != 1` rejects
immediately — a filter with several `AND`-ed conditions, only one of which
mentions `ROW_NUMBER()`, does not qualify at all), and that expression must be
one of exactly three comparison types against a `BIGINT` constant:

```cpp
// src/optimizer/topn_window_elimination.cpp:563-566 (switch), annotated below —
// the per-case "// rn <= N, ..." comments are this book's, not the source's
switch (comparison) {
case ExpressionType::COMPARE_LESSTHANOREQUALTO:  // rn <= N,  N >= 1, N < MIN_MAX_N_MAX_VALUE
case ExpressionType::COMPARE_LESSTHAN:           // rn <  N,  N >= 2, N-1 < MIN_MAX_N_MAX_VALUE
case ExpressionType::COMPARE_EQUAL:              // rn =  N,  N must be exactly 1 (TODO: generalize)
```

(The numeric bounds noted per case are enforced a few lines further down,
inside each case's own body at `topn_window_elimination.cpp:586-616`, not on
the `switch` line itself.)

`rn <= N` and `rn < N` are both accepted (the effective limit for `<` is
`N - 1`, computed later in `ExtractOptimizerParameters` at
`topn_window_elimination.cpp:916-919`, and also returned by `CanOptimize`
itself), which is exactly what lets
`QUALIFY 2 > row_number() OVER(...)` fuse alongside the more common
`<=`/`QUALIFY rn <= N` spelling — `BoundComparisonExpression::Left`/`Right`
here are purely positional (`GetChildren()[0]`/`[1]`,
`src/function/scalar/comparison/comparison.cpp:227-233`), so by the time this
check runs, `2 > row_number()`'s two operands must already sit in the same
`[row-number-reference, constant]` order the `<=` spelling produces natively.
This book traced that reordering back through the expression binder
(`bind_comparison_expression.cpp`, which preserves source-text child order
with no swap) and through the expression-rewriter rules without finding the
specific step responsible for it, so — per this book's policy of flagging
rather than papering over a gap — exactly *where* the swap happens is left
unresolved here; what the test file settles beyond doubt is *that* it
happens, for both `=` and `>`/`<` spellings, consistently enough that this
pass's purely-positional `Left`/`Right` check works correctly regardless of
which side of the comparison the SQL text put the constant on. `rn = N` is
accepted **only for
`N = 1`** — the inline `// TODO: Handle other values` at line 609 is explicit
that `rn = 5`, say, is not (yet) supported, even though semantically "give me
exactly the 5th-ranked row of each partition" is a well-defined operation the
`arg_min`/`arg_max` machinery this rewrite reaches for doesn't directly
support without also computing (and discarding) ranks 1 through 4.

**`MIN_MAX_N_MAX_VALUE` — the exact cap.** Both the `<=` and `<` branches
check the effective limit against `MIN_MAX_N_MAX_VALUE`, a constant defined
once (`src/include/duckdb/function/aggregate/minmax_n_helpers.hpp:22`):

```cpp
static constexpr int64_t MIN_MAX_N_MAX_VALUE = 1000000;
```

The reason this specific number exists here at all: the rewrite hands its
effective limit straight to the `min`/`max`/`arg_min`/`arg_max` aggregates as
their own `n` argument (`CreateAggregateOperator`,
`topn_window_elimination.cpp:373-375`), and those aggregates' own
implementation independently enforces the same cap
(`src/function/aggregate/distributive/minmax.cpp:513-514`, throwing
`InvalidInputException` past it) — so `CanOptimize` is pre-checking a
constraint the aggregate would otherwise discover for itself, at execution
time, as a hard error. Declining the rewrite above the cap and falling back
to the general window plan is strictly the correct behavior: the general
window evaluator has no such ceiling, so a query asking for the top million-
and-first rows of a partition still has to work, just without this
optimization's help.

**The left side must trace, through any chain of plain projections, straight
down to a `LogicalWindow` whose only (or every-identical-copy) expression is
a bare `WINDOW_ROW_NUMBER`.** The traversal (`topn_window_elimination.cpp:625-641`)
walks down through `LogicalProjection`s exactly as `TOP_N`'s own
traversal did, but with stricter checks at each hop. The column being traced
must resolve to **exactly one** underlying column reference
(`column_references.size() != 1` aborts the whole match). And the projection
expression that produces it must itself be a plain column reference:

```cpp
// src/optimizer/topn_window_elimination.cpp:632-637
// The filter has to name the window column itself. An expression that only mentions it, such as
// row_number + 10, is a different predicate, and following the inner reference would drop that expression.
auto &proj_expr = *projection.expressions[current_column_ref.column_index];
if (proj_expr.GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
	return cant_optimize;
}
```

So a projection that computes the filtered column as anything other than a
rename of one upstream column (arithmetic, a function call) breaks the chain.

Why is the second check needed? `row_number() + 10` has exactly one column
reference inside it, so it passes the "exactly one reference" test. Without
the second check, the walk would follow that reference down to the window
and lose the `+ 10`. A filter `rn2 <= 1` on `rn2 = row_number() + 10` would
be rewritten as "keep rank 1", and would return a row whose `rn2` is 11
(issue #27811).
`test/sql/optimizer/topn_window_elimination_computed_filter.test` pins
this down: `WHERE rn2 <= 1` returns no rows, and `WHERE rn2 <= 11` returns
the one row with `rn2 = 11`.

Once the walk reaches an actual `LogicalWindow`, several further conditions all have to hold simultaneously:

- Every `PARTITION BY` expression must itself be a plain column reference
  (`ExpressionClass::BOUND_COLUMN_REF`) — a computed partition key like
  `PARTITION BY g + 1` disqualifies the match (this specific edge case is
  exercised directly by `topn_window_elimination_late_materialization.test`'s
  "value-transforming partition projection" case, §7.18.3.6 below — note that
  case still gets the window *eliminated*, just without late materialization,
  because the partition-key check here is about the window's own expression,
  which is fine, while it's the *late-materialization* eligibility check
  elsewhere that additionally requires the projected group column itself to
  be reconstructable).
- If the `LogicalWindow` carries more than one expression, every expression
  past the first must be `Equals()` to the first — this is what lets a query
  project `ROW_NUMBER()` more than once (e.g. once as the filtered rank and
  once again in the output list) without breaking eligibility, as long as
  they're syntactically the same window call.
- The one window expression must literally be `WINDOW_ROW_NUMBER` — `RANK()`
  and `DENSE_RANK()` are different `ExpressionType`s entirely and are never
  matched by this check at all; this rewrite has no path for them, in this
  version of the source.
- **`ArgOrders()` must be empty.** DuckDB's `ROW_NUMBER(ORDER BY ...) OVER
  (...)` syntax lets a windowed `ROW_NUMBER` carry its *own*, separate
  ordering argument, independent of the window's own `ORDER BY` clause —
  and if one is present, the rewrite bails immediately, because everything
  downstream (`CreateAggregateOperator`) assumes the window's `ORDER BY` is
  the only ranking signal that determines which row wins.
  `topn_window_elimination_argument_order.test` (§7.18.3.4) is built
  specifically to exercise this guard.
- **Exactly one `ORDER BY` entry**, and it must be plain `ASCENDING` or
  `DESCENDING` with **`NULLS_LAST`** null ordering specifically — a window
  `ORDER BY ... NULLS FIRST` is outside this rewrite's scope entirely (there's
  no branch anywhere in `CreateAggregateExpression` for a nulls-first
  variant; the aggregate family it reaches for only comes in default and
  nulls-last flavors).

### 7.18.3.3 What the plan actually turns into

Once eligibility is confirmed, the rewrite doesn't just swap one operator for
another — it rebuilds a small sub-tree, and which exact shape comes out
depends on how many "extra" (non-group, non-order-key) columns the query
actually needs from the winning row:

- **`ExtractOptimizerParameters`** (`topn_window_elimination.cpp:909`)
  computes the effective `limit` (subtracting 1 for the `<` case), whether
  row numbers themselves are referenced downstream
  (`include_row_number`, via `BindingsReferenceRowNumber`), whether the
  payload needs `STRUCT_PACK`ing (more than one extra column) or fits in a
  `SINGLE_COLUMN`, and whether the values may be `NULL`
  (`params.can_be_null`). The last one is not really a decision. It is
  always `true`:

  ```cpp
  // src/optimizer/topn_window_elimination.cpp:926-929
  // The statistics of a column binding are those at the top of the plan: a filter above the window (e.g. on the
  // result of a decorrelated LATERAL join) can have narrowed them to non-NULL although the input of the window
  // contains NULLs. They cannot tell whether the window input can be NULL, so keep the NULL-preserving variants.
  params.can_be_null = true;
  ```

  So the rewrite always picks the `NULL`-preserving (`_nulls_last`) form of
  `arg_min`/`arg_max` when it uses the `arg_` family.

  Why not use column statistics to pick the cheaper plain aggregates? The
  statistics map from `STATISTICS_PROPAGATION` holds one entry per column
  binding, and that entry describes the column at the *top* of the plan. A
  filter above the window can make a column non-`NULL` there, while the
  window's own input still holds `NULL`s.
  `test/optimizer/topn_window_elimination_filter_above_null.test` shows the
  danger. `r` holds `(1, 1, NULL)` and `(2, 1, -5)`. The query
  `SELECT * FROM l JOIN LATERAL (SELECT r.v FROM r WHERE r.k = l.k ORDER BY r.id LIMIT 1) r ON true WHERE r.v <= 0`
  becomes a top-1-per-group rewrite under a filter `r.v <= 0`. That filter
  makes `v` look non-`NULL`. The plain `arg_min` skips rows whose payload
  (`arg`) is `NULL`. With it, the winner would be `-5` (id 2) instead of
  `NULL` (id 1), and the query would return `1, -5` instead of no rows. The
  correct winner is the `NULL` row, which the filter then removes.
- **`CreateAggregateExpression`** (`topn_window_elimination.cpp:335`) picks
  the actual function name by string concatenation:
  `("arg_" if it needs an argument else "") + ("min" or "max", from
  ASCENDING/DESCENDING) + ("_nulls_last" if nullable and using the arg_
  variant)` — giving exactly the function family
  `min`/`max`/`arg_min`/`arg_max`/`arg_min_nulls_last`/`arg_max_nulls_last`
  (or their `_n`-suffixed, `N`-argument forms once `params.limit > 1` adds a
  third argument to the call) that `functions.json` registers under
  `core_functions`. Since `can_be_null` is always `true`, the `arg_` forms
  are always the `_nulls_last` ones. Plain `min`/`max` is still used when only
  the sort key itself is needed and `limit` is 1. `change_to_arg` (line 341) is a nice detail: even a query
  whose payload is *only* the sort key itself (no other projected columns)
  still has to switch from plain `min`/`max` to the `arg_` family the moment
  the sort key can be `NULL` and `limit > 1`, because plain `min`/`max`
  simply discard `NULL`s rather than ranking them last, which would silently
  change which rows survive a multi-row top-N.
- **`CreateAggregateOperator`** builds the actual `LogicalAggregate`, with
  `GROUP BY` set directly to the window's own partition expressions (moved,
  not copied) — and, for the completely unpartitioned, `LIMIT 1` case,
  synthesizes a constant `TRUE` group column specifically so that an *empty*
  input still produces the aggregate's well-defined "no rows" behavior rather
  than accidentally behaving like a `GROUP BY ()` aggregate that always emits
  one row even over zero input (the `empty_topn` case in
  `topn_window_elimination.test_slow`, §7.18.3.5, exists to prove this).
- **`TryCreateUnnestOperator`** only fires when `limit > 1`: a `LIMIT 1`
  query's `arg_min`/`arg_max` result already *is* the single winning value,
  no further unpacking needed, while `limit > 1` produces a `LIST`-typed
  result from the `_n` aggregate variant that has to be exploded back into
  one row per surviving rank via `LogicalUnnest` — and, if row numbers are
  referenced downstream, a synthetic `unnest(generate_series(1,
  array_length(...)))` is unnested alongside it to reproduce the original
  per-row rank values without ever running the real window evaluator.
- **`CreateProjectionOperator`** re-assembles the group columns (in their
  original order), the payload (either a direct column reference, for the
  single-column case, or a run of `struct_extract` calls unpacking the
  packed struct), and the row number if needed.
- Finally, **late materialization** (§7.18.3.6) is attempted *before* the
  aggregate is built whenever the payload would otherwise require
  `STRUCT_PACK`ing more than one column together, as a way to avoid ever
  materializing that struct at all.

### 7.18.3.4 `topn_window_elimination_argument_order.test` — the `ArgOrders()` guard, isolated

This file exists to isolate exactly one of the eligibility conditions above:
`ROW_NUMBER(ORDER BY id DESC) OVER (PARTITION BY g ORDER BY p ASC ROWS
BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) <= 1` ranks each row by
`id DESC` — an ordering argument entirely separate from the window's own
`ORDER BY p ASC` — and the correct answer (`(3,1,3)` and `(5,2,2)`, i.e. the
highest-`id` row in each partition, *not* the highest-`p` row) only comes out
right because the general window evaluator, not this rewrite, computes it:
`window_expr.ArgOrders()` is non-empty here, so `CanOptimize` bails before
ever reaching the point where it would (incorrectly) treat the window's
`ORDER BY p` as the deciding order. The file's second query
(`ROW_NUMBER(ORDER BY id DESC) OVER (PARTITION BY g ORDER BY p ASC) = 1`,
default growing frame instead of the explicit unbounded one) makes the
"argument order, not window order, decides who is rank 1 under a growing
frame" behavior concrete: with the default frame, every row is momentarily
first-by-`id` at the instant it enters the frame, so *every* row in each
partition satisfies `= 1` — the test's expected output is literally every
input row unfiltered. Both shapes correctly bypass this optimization
entirely; no `EXPLAIN` assertion is even needed in this file, because the
result values themselves are diagnostic enough (a spuriously-applied rewrite
would have produced the wrong small subset of rows instead of the whole
table, or the wrong single winner instead of the true `id`-maximal one).

### 7.18.3.5 `topn_window_elimination.test_slow` — the main corpus, walked in depth

This file is the pass's stress-test backbone, organized loosely around a
reusable `window_fun(table_name, col_names, sort_order, topn)` macro that
parameterizes four axes at once (`{sort_order} ∈ {ASC, DESC}`, `{topn} ∈ {1,
3}`, and which of several fixture tables and column-list shapes get plugged
in), plus a parallel `lateral_join` macro proving the same rewrite applies
underneath a correlated `LATERAL` top-N join, not just underneath a plain
`QUALIFY`.

**Regression: an empty-projection-map join side must still get its row ID
appended** (lines 23-53). `topn_filter_l`/`topn_filter_r` are joined, filtered
by a `greatest(...)` comparison, then top-1'd per `c.id`. The `EXPLAIN`
assertion (`COMPARISON_JOIN.*rowid = rowid.*AGGREGATE`) pins down that late
materialization's reconstruction join really does fire here — the comment
above it explains the actual bug this guards: when a join side already has an
*empty* projection map (meaning "project every column, unchanged"), naively
appending a new row-ID column to that side without updating the map would
leave the row ID silently unreachable from above the join, since an empty map
means "everything," but only as of when the map was last examined — so the
fix has to explicitly recognize "empty means already-everything, including
whatever I just added" rather than trying to enumerate the new column into a
map that was deliberately left unenumerated.

**Duplicate row IDs from an inner join** (lines 60-81) is the file's
introduction to the `row_ids_may_have_duplicates` mechanism covered fully in
§7.18.3.7 below — a single `topn_dup_l` row joined against two `topn_dup_r`
rows sharing the same key produces two source rows with the *same*
`topn_dup_l` row ID, and the top-2 rewrite must keep both, not collapse them
via a semi-join that would only keep one. The `EXPLAIN` regex is
deliberately an *alternative* (`INNER.*rowid = rowid.*AGGREGATE` **or**
`arg_max(struct_pack(...`) — the comment explains why: DuckDB's debug build
runs extra column-lifetime verification passes that can conservatively push
the rewrite back to the struct-packing fallback instead of late
materialization, and both outcomes are equally correct, so the test accepts
either rather than over-fitting to one specific debug-vs-release plan shape.

**Empty input must not fabricate a row** (lines 83-104). An unpartitioned
`row_number() OVER (ORDER BY s) <= 1` over a genuinely empty table returns
zero rows — proving the synthetic constant-`TRUE` group column
(§7.18.3.3) doesn't accidentally make the rewritten aggregate behave like an
unconditional `GROUP BY ()`. The second variant is subtler: the *base table*
has one row, but a `WHERE s < random()` runtime-evaluated filter (comparing
against `random()`, which is always in `[0, 1)`, against a stored `s = 1`)
rejects it, so the window's *input* is empty even though the table isn't —
and the rewrite still correctly returns nothing.

**Nullable ordering expressions** (lines 105-173, and reprised standalone in
`topn_window_elimination_nullable_order.test`, walked in §7.18.3.8) —
`NULLIF(id % 3, 0) DESC NULLS LAST` turns a `NOT NULL` column into a
possibly-`NULL` ranking key, and the corresponding row must still be kept
(NULLS LAST, not dropped). Since `can_be_null` is always `true`
(§7.18.3.3), the `NULL`-preserving aggregates keep these rows. Two
further variants nail down that "always-`NULL`" ordering keys
(`NULLIF(id, id)`, i.e. an expression that's `NULL` for every row) don't
fabricate a spurious `NULL` payload column where none should exist, and that
`TRY_CAST`-introduced nulls behave identically to `NULLIF`-introduced ones.

**`NaN` sorts after all finite floats** (lines 175-186) — `ORDER BY val ASC
NULLS LAST` over `{0.0, NaN, -1.0}` keeps `{-1.0, 0.0}` for a top-2, treating
`NaN` as effectively "larger than everything finite" under ascending order —
this is IEEE-754-adjacent floating-point total-ordering behavior the
rewrite's underlying `min`/`arg_min` aggregates have to get right on their
own; the test is here specifically to confirm the rewrite doesn't disturb
that behavior relative to the general window path.

**The main parameterized matrix** (lines 188-345) is the heart of the file:
for every `{ASC, DESC} × {1, 3}` combination, it checks six increasingly
elaborate plan shapes all get the window eliminated
(`<!REGEX>:.*FILTER.*WINDOW.*`) — plain `min`/`max` (one group column only),
`arg_min`/`arg_max` without struct-packing (one extra payload column), the
same with struct-packing (two-plus extra columns), and all three again over
`tbl_with_null` (a table with actual `NULL`s in its struct/list columns,
proving the null-handling machinery holds even when nullability comes from
real data rather than an ordering expression) — followed by a join-underneath
case that still eliminates the window, and a `lateral_join` macro invocation
proving the same elimination reaches a `LATERAL` correlated top-N join. Two
narrower cases inside the same loop are worth calling out specifically:
projecting a join column *from the side not selected for late materialization*
(`col_names = ['grp','a','b','x']`) falls back to the struct-packing/inner-
join alternative-regex pattern seen earlier, while projecting a join column
*from the selected side alongside another table's column*
(`['grp','a','b','y']`) explicitly asserts late materialization does **not**
apply (`<!REGEX>:.*COMPARISON_JOIN.*rowid = rowid.*AGGREGATE.*`) — a direct,
deliberate negative-space test proving the pass correctly declines late
materialization once a projected column genuinely spans both join sides in a
way `CanUseLateMaterialization`'s single-table-of-origin requirement
(§7.18.3.6) can't satisfy. Every one of these ten plan-shape checks is then
independently re-verified for **result correctness**: the loop materializes
each shape's output into its own table with the optimizer enabled, then reruns
the identical query with `disabled_optimizers = 'top_n_window_elimination'`
and asserts an `EXCEPT` between the two produces zero rows — ten separate
differential checks per `{sort_order, topn}` combination, twenty in total
across the outer loop.

**Vector-valued payloads and nearest-neighbor-style queries** (lines 347-396)
push the rewrite through `list_distance`-ranked top-7 and top-1 lateral
self-joins over 1,000 three-dimensional vectors — proving the rewrite's
column-tracing logic handles `LIST` payload types and a `LATERAL`-driven
per-row nearest-neighbor pattern, again checked purely differentially against
the same query with the pass disabled.

**Non-strict `<` and reversed-operand comparisons** (lines 398-459) directly
exercises the same reversed-operand behavior flagged in §7.18.3.2 as an open
question:
`QUALIFY 1 = row_number() OVER(...)` and `QUALIFY 2 > row_number() OVER(...)`
both still eliminate the window regardless of which side of the comparison
the SQL text put the constant on, while `QUALIFY 0 = row_number() OVER(...)`
(asking for rank
zero, which can never be satisfied — `bigint_value != 1` rejects it per the
`COMPARE_EQUAL` branch) and `QUALIFY 1 > row_number() OVER(...)` (asking for
rank *less than* 1, i.e. `<= 0` — rejected by the `COMPARE_LESSTHAN` branch's
`bigint_value < 2` check) both correctly **keep** the general `WINDOW`
operator, since neither is a satisfiable top-N request the aggregate rewrite
knows how to represent, but the general window path still returns the
(empty) correct answer for a filter that can never be true.

**Several numbered GitHub-issue regressions round out the file**: a
`GROUP BY col1, col0, col0` fuzz-tested query unrelated in spirit but included
because it previously tripped up CI; a nearest-neighbor array-distance query
over multiple partitions with duplicate values; an aggregate computed
*downstream* of a top-N-per-partition subquery (`avg(b) ... GROUP BY c`,
checked three equivalent ways — `QUALIFY`, an explicit `WHERE rn = ...`
subquery, and the same shape again through a CTE — confirming the rewrite is
indifferent to which of the three equivalent SQL surface forms produced the
same underlying `LogicalFilter → LogicalWindow` plan shape); a left-
projection-map corruption regression (#21650) where a windowed subquery joins
back onto one of its own two source tables; a late-materialization-plus-cast
regression (#21604) with a materialized CTE reference used on both sides of a
self-join; an `EXCEPT` between two structurally similar but not identical
top-N-per-group subqueries (#confirming no accidental binding aliasing
between the two independently-optimized branches of the `EXCEPT`); the
`parse_filename` late-materialization guard (already covered in depth in
§7.18.3.6 via its own dedicated test file, and reprised here against a
`/User/hawkfish/...`-style path fixture); and, finally, a small cluster of
`ORDER BY NULL` / `ORDER BY <constant>` cases (issue #24363) proving that even
a *degenerate*, always-equal ordering key — `ORDER BY NULL`, `ORDER BY 1`,
`ORDER BY 1 + NULL` — still produces a stable, sequential 1..N numbering
within each partition (ties broken by *some* consistent order, since
`ROW_NUMBER()` never permits genuine ties) rather than either erroring or
collapsing every tied row onto the same rank.

### 7.18.3.6 Late materialization: why it exists, what it guards against, and why Parquet gets its own file

The struct-packing path (`TopNPayloadType::STRUCT_PACK`) exists because the
underlying `arg_min`/`arg_max` aggregates only carry a single value alongside
the ranking key — so the moment a query needs *more than one* extra column
from the winning row, the naive rewrite has to pack all of them together into
one struct value with `struct_pack(...)`, carry that struct through the
aggregate, and unpack it again afterward with a run of `struct_extract` calls.
That's genuinely wasteful when the source is a plain table or file scan: it
means constructing and destructuring a struct for every group, for columns
the aggregate doesn't actually need to *rank* anything by — it only needs
them attached to whichever row wins.

**Late materialization is the alternative**: instead of packing the payload
columns into the aggregate state at all, carry only a compact **row ID**
through the aggregate (as the `arg_min`/`arg_max` value, or the single element
of an `arg_min_n`/`arg_max_n` list), and reconstruct the actual payload
afterward with a join back onto the base table, keyed on that row ID —
exactly the same trick Chapter 7's separate `LATE_MATERIALIZATION` pass (§7.4,
pass #33) uses for ordinary filter/sort/`LIMIT` plans, applied here to a
top-N-per-group aggregate instead. `CanUseLateMaterialization`
(`topn_window_elimination.cpp:947`) is the gate, and it's considerably more
conservative than "any base-table scan qualifies":

- **The payload's origin must trace back to a single table**, walking down
  through projections, filters, and a restricted set of join shapes
  (`INNER`/`SEMI`/`ANTI` only — never an outer join, since an outer join's
  unmatched side could hand back `NULL` row IDs that don't correspond to any
  real row to reconstruct from) until it hits an actual `LogicalGet`
  (`IsRecreatableByLateMaterialization`, §7.18.3.6.1 next).
- **The base scan's function must itself support late materialization**
  (`logical_get.function.late_materialization` and `get_row_id_columns` must
  both be present) — this is a scan-type capability, not something every
  `LogicalGet` automatically has.
- **It must actually shrink the row-ID footprint**: if the number of row-ID
  columns the base scan needs is not strictly smaller than the number of
  original payload arguments (`rhs_rowid_column_idxs.size() >= args.size()`,
  `topn_window_elimination.cpp:1174`), late materialization is declined —
  there's no point trading one struct for a differently-shaped one that isn't
  actually narrower.
- **Every projected column along the path must be
  `IsRecreatableByLateMaterialization`** (`topn_window_elimination.cpp:201`):
  a bare column reference, or a single non-`TRY_CAST` cast directly wrapping a
  bare column reference. Anything else — a scalar function call
  (`parse_filename(path)`), a `TRY_CAST` (whose `NULL`-on-failure semantics a
  strict reconstruction cast would silently discard, turning a `NULL` payload
  into a thrown conversion error instead), or a cast-of-a-cast (whose
  intermediate, possibly-lossy conversion a single reconstruction cast
  couldn't reproduce) — disqualifies the whole attempt, and the rewrite falls
  all the way back to ordinary struct-packing instead of half-reconstructing
  a wrong value.

`topn_window_elimination_late_materialization.test` is built entirely around
proving that last bullet holds for every value-transforming shape someone
might reasonably write: `parse_filename(path)` (issue #22849 — the file's
lead case, and the source of the comment
`// Late materialization recreates a payload by re-scanning the base table
column and re-applying a single default (strict) cast` at
`topn_window_elimination.cpp:195-200`), `upper(v)`, plain arithmetic
(`n + 1`), a value-transforming *partition* expression (`g + 1` — rejected
for late materialization specifically because the reconstruction join would
otherwise have to re-derive the group key itself from a raw column, not just
the payload), a cast wrapping a function call, a `TRY_CAST` (issue #24609 —
where the failure mode would otherwise turn a would-be-`NULL` winning value
into a hard `Conversion Error`), and a doubly-nested cast (also #24609 — where
a naive single reconstruction cast would silently skip the intermediate
truncation, e.g. `CAST(CAST(x AS INTEGER) AS DOUBLE)` needs the integer
truncation to actually happen, not just a direct `DOUBLE → DOUBLE` no-op).
Every one of these cases pairs an `EXPLAIN` proving the window still gets
eliminated (`<!REGEX>:.*FILTER.*WINDOW.*`) with a second `EXPLAIN` proving
late materialization specifically does **not** engage
(`<!REGEX>:.*COMPARISON_JOIN.*rowid = rowid.*`), then a differential result
check against `disabled_optimizers = 'top_n_window_elimination'` — the same
"prove the plan shape and prove the answer, separately" pattern seen
throughout this corpus. A companion positive case (`CAST(a AS TIMESTAMPTZ)`
directly on a base column, no function wrapping) confirms a *plain* cast of a
raw column is still eligible and does get the join-based reconstruction —
the boundary being drawn precisely at "a single non-`TRY_CAST` cast directly
over a column reference," neither more permissive nor more restrictive.

**A cast chain spread over several projections.** `IsRecreatableByLateMaterialization`
looks at one projection expression at a time. So a column cast once in each of two
stacked projections could pass: each projection on its own holds "a single cast
over a column". But late materialization re-applies only one cast, from the base
column to the final type, so the intermediate cast would be lost. To stop
this, `CanUseLateMaterialization` remembers, per payload column, whether it has already
passed a cast on the way down (`column_is_cast`):

```cpp
// src/optimizer/topn_window_elimination.cpp:991-996
if (BoundCastExpression::IsCast(*projection.expressions[projection_idx])) {
	if (column_is_cast[i]) {
		return false;
	}
	column_is_cast[i] = true;
}
```

A second cast for the same column means "no late materialization". The test
for this (issue #27812,
`topn_window_elimination_late_materialization.test:253-277`) casts an `INT`
`42` to `BOOLEAN` in one subquery and back to `INT` in the next. The correct
answer is `1` (`42` is `true`, and `true` is `1`). A single re-applied cast
from the base column would have returned `42`.

**A row id that is also a payload column.** The reconstruction join
(`ConstructJoin`) expects the row-id columns to be the *last* columns of its
left input. If the query itself also selects `rowid`, that column is already
in the scan, somewhere before the end, so the join would read the wrong
column. The pass forces a projection that re-appends the row ids at the end whenever a
row-id column is also one of the scan's columns
(`topn_window_elimination.cpp:1137-1150`). The tests (issue #27813,
`topn_window_elimination_late_materialization.test:279-362`) check that
`SELECT id, p1, rowid ... WHERE rn <= 1` returns the winning row's own
`rowid`, with `rowid` first or last in the select list, and for `rn <= 2`.

**Pruning the first scan again.** With late materialization, the plan scans
the table twice. The first scan feeds the aggregate, which only needs the
partition column, the order column, and the row id. The second scan fetches
the payload for the winning rows. The payload columns must then be removed
from the first scan. So the pass records whether any rewrite used late
materialization (`UsedLateMaterialization()`), and the optimizer then runs
`UNUSED_COLUMNS` one more time:

```cpp
// src/optimizer/optimizer.cpp:512-521
// statistics propagation removes filters, join conditions and aggregate children that it proves redundant.
// this can leave columns in a scan that nothing references any more - prune them again. so does a top-N rewrite
// with late materialization, which reads the payload in a second scan
if (removed_expressions || topn_used_late_materialization) {
	RunOptimizer(OptimizerType::UNUSED_COLUMNS, [&]() {
		ClearProjectionMaps(*plan);
		RemoveUnusedColumns unused(*this);
		unused.VisitOperator(plan);
	});
}
```

`test/optimizer/topn_window_elimination_late_materialization_scan.test` checks
it. For `SELECT a, b, c FROM t WHERE a IN (1, 2, 3, 4) QUALIFY row_number()
OVER (PARTITION BY a ORDER BY b DESC) <= 2`, the plan must contain the
`rowid = rowid` join, and one scan must show `Projections: a, b` only.

Without this extra run, the first scan would read every selected column. On
a 1.5-million-row table with a 768-byte string column `c`, that is the
difference between loading 740 MiB of table data and 5.2 MiB.

**Why Parquet gets its own dedicated `.test_slow` file** rather than folding
into the general one: a Parquet scan's row-identifying virtual column is a
different concrete mechanism than an in-memory table's (`file_row_number`,
tied to the physical layout of one or more Parquet files, rather than
DuckDB's native storage row groups), so every one of the general file's plan-
shape assertions is worth re-running end to end against `read_parquet(...)`
specifically to confirm late materialization's row-ID plumbing works
correctly against *that* scan type too, not just the native table scan the
main file exercises. The file's lead regression case makes the reason
concrete rather than aspirational: `projected_get.parquet` is scanned with a
`WHERE NOT excluded` filter already pushed down into the `LogicalGet` *before*
this pass ever runs, and the comment directly above it explains the bug this
guards against — *"the pushed-down filter creates a gap in the GET's
projection ids before the row ids are appended"* — i.e., if the row-ID
columns this pass appends to the scan's projection list are appended
naively, without accounting for the fact that an already-pushed-down filter
may have left non-contiguous holes in the projection-id numbering, the
row-ID columns can end up bound to the wrong physical column index.
Re-running the same `min`/`arg_min`/struct-packing/join-projection matrix as
the general file, purely against Parquet, is what actually exercises this —
proving the fix generalizes across scan types rather than being a narrower,
in-memory-table-specific patch.

#### 7.18.3.6.1 Row multiplicity: semi-join vs. inner-join reconstruction

One more late-materialization wrinkle deserves its own mention, because it's
the subject of an entire dedicated test file
(`topn_window_elimination_multiplicity.test`) and a specific field on
`TopNWindowEliminationParameters`: **a row ID is not necessarily unique in
the stream the aggregate reads.** If a join sits between the base scan and
the window (an `INNER` join specifically — `CanUseLateMaterialization` sets
`params.row_ids_may_have_duplicates = true` the moment it walks through one,
`topn_window_elimination.cpp:1013-1017`), the very same base-table row can
appear multiple times in the aggregate's input, each occurrence potentially
ranking differently depending on what it was joined against. Reconstructing
"the payload for row ID 7" with a plain **semi-join** back onto the base
table would silently *deduplicate* those occurrences — collapsing what should
have been two winning rows sharing the same row ID down to one. `ConstructJoin`
(`topn_window_elimination.cpp:1293`) picks between a `SEMI` join (cheap, but
only correct when row IDs are provably unique — the ordinary case of a plain
scan with no join underneath) and an `INNER` join (preserves each duplicate
occurrence's own row, at the cost of an actual join rather than a cheap
membership check) based on exactly this flag, additionally forced to `INNER`
whenever row numbers themselves are projected (since two duplicate-row-ID
occurrences must still be told apart by *which* rank each individually
earned). `topn_window_elimination_multiplicity.test` walks through: a
duplicated row ID surviving an unpartitioned top-2 and a partitioned top-2
alike; row numbers themselves surviving correctly per duplicate occurrence;
the duplication originating from *either* join side (not just the
conventionally-expected one); an explicit `EXPLAIN` proving the
reconstruction join is **not** `SEMI` the moment duplication is possible; and,
as a contrasting control, a plain scan/filter/projection path (no join at
all, hence no duplication risk) confirmed to still take the cheaper `SEMI`
path. This is the multiplicity story `topn_window_elimination_group_stats.test`
and the CTE regression below both incidentally touch as well, but this file
is where it's isolated and proven directly.

### 7.18.3.7 `topn_window_elimination_group_stats.test` — statistics scoping, not multiplicity

Despite superficially resembling the multiplicity file (both involve a join
above a partitioned top-N-per-group subquery), this one is testing something
narrower and specifically about *where the rewritten aggregate's group
statistics come from*. The setup: `w` has 40 distinct partition-key values
(`(range * 7) % 40`), and the query joins the windowed subquery against a
four-row table `k`, filtering the join to only four of those forty groups.
Naively, one might expect the aggregate that replaces the window to inherit
whatever narrowed group-key statistics apply *after* the join — but the
aggregate physically sits *below* the join in the rewritten plan, and must
see statistics describing its own actual input (all 40 partitions), not
statistics narrowed by something that hasn't executed yet by the time the
aggregate runs. `CreateAggregateOperator`'s handling of `group_stats`
(`topn_window_elimination.cpp:394-404`) is written specifically to source
these statistics from the window's own `PartitionsStats()` — a snapshot of
what the statistics propagator determined about the window's *input*, before
any join above it had a chance to narrow anything — rather than from the
optimizer's global, join-aware statistics map, and the inline comment says as
much directly: *"The global statistics map cannot be used here: its entries
are narrowed by operators above the window."* Getting this wrong wouldn't
produce an incorrect *result* — the aggregate's `GROUP BY` still groups
correctly regardless of what statistics it's handed — but it could enable an
incorrect **perfect-hash-aggregation** decision downstream (a physical-plan
choice that assumes a tight, correct bound on the number of distinct group
values, purely as a performance optimization) if the narrowed, post-join
statistics were used instead and under-counted how many groups the aggregate
itself actually has to handle. The test's two queries are plain result
checks (`count(*) = 4`, and the four winning `(g, v)` pairs by value) —
correctness of the *count* and *values* is really a proxy here for "the
rewrite didn't quietly break under a scenario where its statistics
plumbing could plausibly have been wired to the wrong scope," which is not
something an `EXPLAIN` assertion alone could catch nearly as directly as
getting the actual row values right.

### 7.18.3.8 The CTE mixed-partition-types regression: a case study

`issue_topn_window_elimination_cte_mixed_partition_types.test` is the richest
single case study in this corpus, and it's worth reconstructing the failure
mode it guards against rather than just reporting that a fix exists.

The reproduction's `ranked` CTE partitions by **two columns of different
types together** — `PARTITION BY n.name, d.date`, a `VARCHAR` and a `DATE` —
over a plan shape assembled from a `VALUES`-based CTE, a `DISTINCT` subquery,
a `CROSS JOIN`, and a `LEFT JOIN`, before the outer query selects only a
*subset* of the ranked CTE's columns (`SELECT name, date FROM ranked WHERE rn
= 1` in the first variant, then the **column order swapped**,
`SELECT date, name FROM ranked WHERE rn = 1`, in the second). The comment
right at the top names the exact hazard precisely: *"TopN window elimination
should preserve mixed-type partition bindings through CTE/projection
rewrites."* The failure this guards against is a **binding-tracking bug, not
a computation bug**: the rewrite's `GenerateAggregatePayload` and
`TraverseProjectionBindings` machinery (§7.18.3.3, and the `group_bindings`
map built at `topn_window_elimination.cpp:705-716`) has to correctly track,
through every intervening CTE reference and projection, *which* output
column of the final projected result corresponds to *which* partition-key
expression of the original window — and it has to get this right
independently of (a) which of the two partition columns is a `VARCHAR` versus
a `DATE`, and (b) what order the outer `SELECT` lists them in, which need not
match the order they were partitioned by at all. A binding-remap bug here
wouldn't necessarily error out loudly — it could just as easily silently swap
which value ends up under which output column, returning `A, 2022-02-01` in
the `name` position when `2022-02-01` (a date) belongs in the `date`
position, which is exactly the kind of quiet correctness bug a plan-shape
`EXPLAIN` assertion would never catch and only a full result comparison
(`query TT ... ---- A 2022-01-01 / A 2022-02-01`, run once per column order)
can. Running the identical CTE shape twice, with only the outer projection's
column order swapped between the two `query TT` blocks, is precisely the
right differential structure to isolate an ordering/binding bug from a value
bug: if the rewrite's binding tracking silently transposed the two partition
columns anywhere along its rewrite path, the second block's expected output
(`date` first, `name` second) would come back with the columns still in the
*first* block's order, a mismatch a naive "does it error" test would never
surface.

The remaining two cases in the same file are further, independently-numbered
regressions bundled into it because they share the same root symptom class —
top-N-per-group elimination interacting with a subquery/CTE boundary — rather
than being examples of the mixed-partition-types bug specifically: issue
#21788 wraps the ranked CTE's `WHERE` clause around a correlated `IN
(SELECT ...)` subquery over a *second* table joined only through that
subquery, and issue #21820 exercises the equivalent shape spelled with
`QUALIFY` directly in a `CREATE TABLE ... SELECT ... FROM ...`, rather than
through an intermediate CTE at all — both simple existence checks (the
queries must merely execute and return the expected single row) rather than
column-order-sensitive checks, confirming the fix for the mixed-partition-
type binding bug didn't regress these two simpler, previously-already-working
shapes in the process.

### 7.18.3.9 Signed zeros in a `FLOAT`/`DOUBLE` partition column

The rewrite turns `PARTITION BY d` into `GROUP BY d`. A `GROUP BY` puts values
that compare equal into one group. For `FLOAT` and `DOUBLE`, `0.0 = -0.0` is
true, so both zeros land in the same group. If a selected partition column
were read back *from the group*, as the group key, every row of that group
would get the sign of whichever zero the group saw first. You can see the
damage with `1 / d`: it is `inf` for `0.0` and `-inf` for `-0.0`. `NaN` would
lose its sign the same way.

`GenerateAggregatePayload` avoids this. A `FLOAT` or `DOUBLE` partition
column, or a nested type that contains one, is not mapped to its group. It
is carried in the payload instead, so each kept row brings its own value.
This is the same `group_bindings` map that §7.18.3.8 relies on:

```cpp
// src/optimizer/topn_window_elimination.cpp:705-716
// Remember order of group columns to recreate that order in new bindings later.
// A FLOAT or DOUBLE group holds both 0.0 and -0.0, so such a column is carried in the payload instead.
column_binding_map_t<idx_t> group_bindings;
for (idx_t i = 0; i < window_expr.PartitionsMutable().size(); i++) {
	auto &expr = window_expr.PartitionsMutable()[i];
	const auto &type = expr->GetReturnType();
	if (TypeVisitor::Contains(type, LogicalTypeId::FLOAT) || TypeVisitor::Contains(type, LogicalTypeId::DOUBLE)) {
		continue;
	}
	auto column_references = ColumnReferenceExtractor::GetReferences(&expr);
	group_bindings[column_references.begin()->first] = i;
}
```

The window is still eliminated; only where the value comes from changes.
`test/optimizer/topn_window_elimination_negative_zero.test` shows both parts:

```sql
-- test/optimizer/topn_window_elimination_negative_zero.test:8-27
CREATE TABLE t AS FROM (VALUES (0.0::DOUBLE, 1, 1), (-0.0::DOUBLE, 2, 1), (1.5::DOUBLE, 3, 2),
                               (-0.0::DOUBLE, 4, 2), (-1.0::DOUBLE, 5, 1), (0.0::DOUBLE, 6, 2)) v(d, x, g);

EXPLAIN SELECT x, d FROM (SELECT x, d, row_number() OVER (PARTITION BY d ORDER BY x) rn FROM t) WHERE rn <= 10;
----
logical_opt	<!REGEX>:.*WINDOW.*

SELECT x, d::VARCHAR, (1 / d)::VARCHAR FROM (SELECT x, d, row_number() OVER (PARTITION BY d ORDER BY x) rn FROM t)
WHERE rn <= 10 ORDER BY x;
----
1	0.0	inf
2	-0.0	-inf
3	1.5	0.6666666666666666
4	-0.0	-inf
5	-1.0	-1.0
6	0.0	inf
```

Read from the group, rows 2 and 4 would come back as `0.0` with `inf`. The
same file repeats the check for `rn = 1`, for `QUALIFY` with a second
partition column `g`, and for `REAL`. This has a cost: when a query selects
both the `ORDER BY` column and such a partition column, the payload grows,
and a quick `rn = 1` benchmark runs 1.5 to 2.5 times slower.

One smaller robustness guard belongs here too. `OptimizeInternal` calls
`filter.ResolveOperatorTypes()` before it reads the filter's types
(`topn_window_elimination.cpp:279-283`). When `STATISTICS_PROPAGATION` is
disabled, those types might not be filled in yet, and the pass would read
past the end of the list. `test/optimizer/topn_window_elimination_nearest_setop.test`
reproduces it with `SET disabled_optimizers='statistics_propagation'` and an
`EXCEPT` and a `UNION` of two `NEAREST` joins.

## 7.18.4 Summary: where each pass leaves the plan

Putting all three back side by side, in terms of what `EXPLAIN` shows once
each has done its work:

- **`TOP_N`**: a plain `ORDER BY ... LIMIT [OFFSET]` collapses from two
  operators (`LogicalOrder` + `LogicalLimit`) into one (`LogicalTopN`),
  unless the limit is a parameter, a percentage, or large enough relative to
  the child's estimated cardinality (`> 5000` rows **and** `> 0.7%` of the
  child) that a full sort is heuristically cheaper — in which case the plan
  is left exactly as the binder produced it, for Chapter 15's two-phase
  merge sort to handle in full.
- **`ROW_NUMBER_REWRITER`**: a bare `ROW_NUMBER() OVER ()` directly over an
  unfiltered base-table scan disappears entirely — no `LogicalWindow` node
  survives — replaced by a wider `LogicalGet` reading one extra, storage-
  native virtual column. Any `PARTITION BY`, `ORDER BY`, intervening
  operator, non-base-table scan, or pushed-down filter keeps the general
  window evaluator in the plan instead.
- **`TOP_N_WINDOW_ELIMINATION`**: a `ROW_NUMBER() OVER (PARTITION BY ...
  ORDER BY single_col [ASC|DESC] NULLS LAST) <op> N` immediately filtered
  (typically via `QUALIFY`) collapses into a `LogicalAggregate` using
  `min`/`max`/`arg_min`/`arg_max` (or their `_n`/`_nulls_last` variants),
  optionally followed by a `LogicalUnnest` (for `N > 1`) and, where eligible,
  a row-ID reconstruction join in place of struct-packing multiple payload
  columns together — with the join itself chosen `SEMI` or `INNER` depending
  on whether the rows feeding the aggregate could carry duplicate row IDs.
  An explicit `ROW_NUMBER(ORDER BY ...)` argument order, a `RANK()`/
  `DENSE_RANK()` call, a compound partition-key expression, a filter on an
  expression over the row number (such as `row_number() + 10`), `NULLS FIRST`
  ordering, an effective limit at or past 1,000,000, or an `= N` filter for
  any `N` other than 1 all keep the general window path instead. A `FLOAT`
  or `DOUBLE` partition column is carried in the payload rather than read
  from the group, so each row keeps the sign of its own zero (§7.18.3.9).

Between them, the three passes cover almost the entire practical surface of
"the user wrote a window function whose only real purpose was picking
winners out of an ordering" — leaving genuinely general window computation
(real aggregation *over* a frame, `RANK()`/`DENSE_RANK()`, multi-column or
`NULLS FIRST` orderings, non-1 exact-rank filters) to the general-purpose
window-function evaluator this part deliberately did not need to describe.
