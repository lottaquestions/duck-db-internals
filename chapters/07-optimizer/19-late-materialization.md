# 7.19 `LATE_MATERIALIZATION` — Scan Narrow, Sort Narrow, Fetch the Wide Columns Late

Pass #33 in §7.4's list, `src/optimizer/late_materialization.cpp` (+ the shared
`src/optimizer/late_materialization_helper.cpp`), gets one line there:
*"rewrites the scan to carry row IDs through the narrow operators and defer
fetching the wide payload columns until a late semi-join reconstructs them."*
This section is where that line gets earned — read against the actual C++,
against five real test files that between them encode four separate GitHub
issues' worth of hard-won correctness lessons, and against the storage model
Chapter 17 already built, because this pass genuinely cannot be understood
without it.

## 7.19.1 The idea, before any code

Take a query shaped like the one this chapter keeps coming back to:

```sql
SELECT customer_id, name, address, email, notes, segment, referrer,
       signup_channel, lifetime_value, last_purchase_date
FROM customers
WHERE last_purchase_date > '2020-01-01'
ORDER BY total_spend DESC
LIMIT 10;
```

Ten columns in the `SELECT` list, most of them wide (`VARCHAR` names,
addresses, free-text `notes`). But the `WHERE` clause only touches
`last_purchase_date`, and the `ORDER BY` only touches `total_spend`. Read
literally, syntax-first, the way Chapter 5's binder would produce it: scan
all ten (or however many) columns for every row in `customers`, filter on
`last_purchase_date`, feed *all ten wide columns of every surviving row* into
a bounded top-10 heap keyed on `total_spend`, and only then, once the heap
has converged, emit the winners. If `customers` has a million rows and 10
survive to the final answer, every one of those ten output columns gets
read, decompressed, and copied into the sort/limit machinery's internal
buffers roughly a million times over — even though 999,990 of those copies
are guaranteed to be discarded the moment a better row bumps them out of the
heap.

`LATE_MATERIALIZATION`'s whole idea is to notice that the *sort key* and the
*filter key* are narrow (one or two skinny columns) even when the *payload*
is wide, and to split the plan into two scans of the same table instead of
one:

- A **narrow scan** that reads only what the filter/sort/limit actually
  needs — `last_purchase_date`, `total_spend`, and a **row ID** — and does
  all the expensive work (filtering, comparing, heap maintenance) on that
  narrow representation.
- A **wide scan**, of the *same* table, that reads the full `SELECT` list
  (all ten columns) plus a row ID, but never runs through the filter or the
  heap at all.
- A **join between the two, keyed on row ID**, that keeps only the wide
  scan's rows whose row ID also came out of the narrow side's winning set —
  i.e., exactly the 10 rows that survived the filter/sort/limit, and nothing
  else.

The payoff, in Chapter 2's vocabulary: this is "stops needing to touch most
of the bytes... at all" (§2.4) applied one layer deeper than Chapter 2's own
example. Chapter 2 showed that a *materialized* DuckDB table beats `read_csv`
because it never re-parses unwanted columns; this pass is the optimizer
arranging, for a *single* scan of a *single* table, for the wide columns to
never enter the hot filter/sort/limit path in the first place — narrowing
what §17.2.2 calls "narrowed projection" not just at the `SELECT`-list level
(which `filter_pushdown`/`remove_unused_columns` already handle — Chapter 7
§7.4 #6, #23) but *temporally*, so the wide columns are read once, late, for
survivors only, instead of once per row through the entire pipeline.

## 7.19.2 What actually triggers it — and a correction to a plausible-sounding assumption

It would be natural to assume `LATE_MATERIALIZATION` triggers on any of
"filter, sort, or `GROUP BY`" — a scan feeding some narrowing operator whose
own inputs are narrower than the scan's own output. Reading
`LateMaterialization::Optimize` directly rules half of that assumption out:

```cpp
// src/optimizer/late_materialization.cpp:420
unique_ptr<LogicalOperator> LateMaterialization::Optimize(unique_ptr<LogicalOperator> op) {
	switch (op->type) {
	case LogicalOperatorType::LOGICAL_LIMIT: {
		// ...
	}
	case LogicalOperatorType::LOGICAL_TOP_N: {
		// ...
	}
	case LogicalOperatorType::LOGICAL_SAMPLE: {
		// ...
	}
	default:
		break;
	}
	for (auto &child : op->children) {
		child = Optimize(std::move(child));
	}
	return op;
}
```

Three entry points, full stop: `LOGICAL_LIMIT`, `LOGICAL_TOP_N` (the fused
`ORDER BY ... LIMIT` operator from `TOP_N`, pass #32, which runs immediately
before this one at `optimizer.cpp:459` vs. `optimizer.cpp:465`), and
`LOGICAL_SAMPLE` for a row-count (not percentage) sample. There is no
`LOGICAL_AGGREGATE` case anywhere in this switch. A bare `GROUP BY` with no
`ORDER BY`/`LIMIT` above it never reaches `TryLateMaterialization` at all —
this pass simply does not fire for it. This isn't an oversight worth
correcting so much as a structural non-issue: a `LogicalAggregate`'s own
output is, definitionally, only its `GROUP BY` keys and aggregate results —
anything else in a `SELECT` list next to a `GROUP BY` has to be functionally
dependent on the grouping key or wrapped in an aggregate to be legal SQL in
the first place, so there's no scenario where a `LogicalAggregate` reads a
wide column it doesn't already need for its own grouping/aggregation. The
"wide `SELECT` list, narrow filter/sort key" shape this pass exploits is
specific to `LIMIT`/`TOP_N`/`SAMPLE`, where an operator can pass a wide,
unaggregated row straight through to the output based on a comparison that
only looked at one or two of its columns — exactly the shape a `GROUP BY`
structurally cannot produce.

### The row-count gate: no column-count or width threshold at all

The setting that gates this whole pass is a **row-count** cap on the *output*
size — not a column count, not an estimated byte width, not any measure of
"how wide is wide":

```cpp
// src/include/duckdb/main/settings.hpp:1868
struct LateMaterializationMaxRowsSetting {
	using RETURN_TYPE = idx_t;
	static constexpr const char *Name = "late_materialization_max_rows";
	static constexpr const char *Description =
	    "The maximum amount of rows in the LIMIT/SAMPLE operator for which we trigger late materialization.";
	static constexpr const char *InputType = "UBIGINT";
	static constexpr bool IsDebug = false;
	static constexpr bool IsDeprecated = false;
	static constexpr const char *DefaultValue = "50";
	// ...
};
```

Default **50**. For `LOGICAL_TOP_N`, this caps `top_n.limit` directly
(`late_materialization.cpp:449`); for `LOGICAL_SAMPLE`, it caps the
row-count sample size (`:463`); for `LOGICAL_LIMIT`, it caps the constant
`LIMIT` value, *unless* `OptimizeLargeLimit` grants a special exception:

```cpp
// src/optimizer/late_materialization.cpp:391
bool LateMaterialization::OptimizeLargeLimit(LogicalLimit &limit, idx_t limit_val, bool has_offset) {
	if (!has_offset && !Settings::Get<PreserveInsertionOrderSetting>(optimizer.context)) {
		// we avoid optimizing large limits if preserve insertion order is false
		// since the limit is executed in parallel anyway
		return false;
	}
	static constexpr const idx_t LIMIT_MAX_VAL = 1000000;
	if (limit_val > LIMIT_MAX_VAL) {
		return false;
	}
	// we only support large limits if they are directly below the source
	reference<LogicalOperator> current_op = *limit.children[0];
	while (current_op.get().type != LogicalOperatorType::LOGICAL_GET) {
		if (current_op.get().type != LogicalOperatorType::LOGICAL_PROJECTION) {
			return false;
		}
		current_op = *current_op.get().children[0];
	}
	// if there are any filters we shouldn't do large limit optimization
	auto &get = current_op.get().Cast<LogicalGet>();
	if (get.table_filters.HasFilters()) {
		return false;
	}
	return true;
}
```

So a plain, unconditional `LIMIT N` (no `WHERE`, no `ORDER BY`, sitting
directly on a `Get`/`Projection` chain with nothing else in between) gets a
1,000,000-row allowance instead of 50 — the comment explains why this is safe
specifically for *that* shape: "the row-ids are not consecutive [in general,
so] doing the join can worsen performance," but for a plain `LIMIT` with no
filter, the surviving row IDs genuinely *are* consecutive (rows
`[0, N)` in whatever order the scan naturally produces them), which is
exactly the case where the row-ID join is cheap. `has_offset` matters too:
optimizing a small, offset-free `LIMIT` "really only makes sense if we have
an offset" (`:437-440`) — a bare `LIMIT 10` with no `OFFSET` and no sort key
doesn't need any narrowing at all, since a plain scan can already stop after
10 rows without reading the rest of the table.

The *structural* eligibility check, independent of the row-count gate, lives
in `TryLateMaterialization` and is worth reading exactly because it is **not**
a width or column-count threshold either — it's a single inequality over
column *sets*:

```cpp
// src/optimizer/late_materialization.cpp:216
column_binding_set_t required_columns;
for (auto &entry : column_references) {
	required_columns.insert(entry.first);
}
for (auto &filter_entry : get.table_filters) {
	required_columns.insert(ColumnBinding(get.table_index, filter_entry.GetIndex()));
}
if (required_columns.size() >= get.GetColumnIds().size()) {
	// we do not benefit from late materialization
	// every column the scan reads is already required upstream (Top-N/Limit/etc) or by its pushed-down filters
	return false;
}
```

`column_references` was populated by walking every expression the
`LIMIT`/`TOP_N`/`SAMPLE` chain and any `Filter`/`Projection` operators
between it and the `Get` actually reference (`VisitOperatorExpressions`,
called at `:170` and again inside the `Projection`/`Filter` cases of the
walk at `:174-213`). If that set — plus whatever columns the scan's own
`table_filters` already need — covers *every* column the scan was going to
read anyway, there is nothing to strip out, and the pass bails. There is no
separate check for "and only if there are at least N columns" or "only if
the dropped columns are estimated to be at least K bytes wide" — dropping
even a single unneeded column is enough to trigger the rewrite, as long as
the row-count gate above is also satisfied. "Wide" in this pass's own
vocabulary just means "not referenced by the filter/sort/limit key or the
scan's pushed-down filters" — nothing about actual byte width ever enters
the decision.

Three more structural gates round out `TryLateMaterialization`, each worth
naming precisely:

- **The walk from `LIMIT`/`TOP_N`/`SAMPLE` down to the `Get` only tolerates
  `LOGICAL_PROJECTION` and `LOGICAL_FILTER` in between** (`:172-213`) — hit
  anything else (an `Aggregate`, a `Join`, a `Window`) and the `default:
  return false` fires. This is the same structural fact that rules out
  `GROUP BY` from a different angle: even a `GROUP BY` sitting *below* a
  `LIMIT`/`TOP_N` blocks the rewrite, because the walk can't step through it.
- **Volatile projection expressions bail out entirely** (`:180-184`):
  `if (expr->IsVolatile()) { return false; }` — reconstructing a wide row via
  a second scan means re-evaluating that row's projection expressions a
  second time (once on the narrow RHS side conceptually, though as
  §7.19.3 shows the *wide* copy is really where re-evaluation happens); a
  volatile expression like `random()` would then produce a *different* value
  the second time, which is observably wrong.
- **`get.function.late_materialization` must be `true`** (`:228-231`) — this
  is an opt-in flag on the table function itself, defaulting to `false`
  (`src/function/table_function.cpp:32`). A full-text search of `src/`
  and `extension/` for every assignment of this flag turns up exactly three
  hits: DuckDB's own table scan
  (`scan_function.late_materialization = true;` —
  `src/function/table/table_scan.cpp:1307`), the `ATTACH`ed-DuckDB-database
  reader (`read_duckdb.late_materialization = true;` —
  `src/function/table/read_duckdb.cpp:551`), and the Parquet reader
  (`read_parquet.late_materialization = true;` —
  `extension/parquet/read_single_parquet_file.cpp:554`). The Parquet flag
  sits on the single-file reader. The multi-file `read_parquet` function copies
  it from there (`src/common/multi_file/table_function_multi_file.cpp:893`),
  so Parquet scans are eligible. Nothing under
  `read_csv`'s table function registration sets it — a plain CSV scan is
  never eligible for this rewrite, full stop, regardless of the query shape.
  This is a reasonable design boundary even if the source doesn't spell out
  the reasoning explicitly: a row-ID-keyed re-scan needs the storage layer to
  answer "give me exactly these row IDs back," and CSV's row-oriented,
  no-index text format doesn't have that capability the way row-group/
  column-segment storage and Parquet's row-group-indexed layout do.
- **A row-count sample already pushed down onto the `Get` blocks it**
  (`:232-236`): `if (get.extra_info.sample_options &&
  !get.extra_info.sample_options->is_percentage) { return false; }` — this
  is a *different* sampling mechanism from the `LOGICAL_SAMPLE` operator case
  handled by the outer switch; it's a sample that `SAMPLING_PUSHDOWN` (pass
  #31, which runs immediately before `TOP_N` and `LATE_MATERIALIZATION`) has
  already relocated *into* the scan itself. The comment is direct about why:
  "the sample scan is already fast and creating a semi-join would duplicate
  the full table scan" — a row-count-limited sample scan already stops early
  on its own; wrapping it in a second, wide, unbounded scan plus a join would
  make things strictly worse, not better.

## 7.19.3 The exact rewrite: one scan becomes two, joined by row ID

Once every gate above passes, `TryLateMaterialization` performs a genuinely
intricate piece of tree surgery. It's worth walking in the order the code
actually executes it, because the ordering is load-bearing (comments in the
source flag exactly this in two places).

**Step 1 — fetch the row-ID columns this scan's table function can supply.**
Every eligible table function also registers a `get_row_id_columns`
callback:

```cpp
// src/optimizer/late_materialization.cpp:237-251
if (!get.function.get_row_id_columns) {
	throw InternalException("Function supports late materialization but not get_row_id_columns");
}
row_id_column_ids = get.function.get_row_id_columns(optimizer.context, get.bind_data.get());
// ...
for (auto &col_id : row_id_column_ids) {
	auto entry = get.virtual_columns.find(col_id);
	// ...
	row_id_columns.push_back(entry->second);
}
```

For DuckDB's own tables this resolves through `TableScanGetRowIdColumns`
(`table_scan.cpp:1225`, registered at `:1312`) to the scan's virtual `rowid` column — the same
virtual column Chapter 16's ART discussion introduces as the thing an index
leaf ultimately hands back (§16.8's `LEAF_INLINED`/`GetRowId()` walkthrough).
Almost always this is a single column (`row_id_columns.size() == 1`), though
the code is written generically over a *vector* of row-ID columns
throughout, because some table functions (multi-file Parquet reads, in
particular) key a row by more than one coordinate.

**Step 2 — build the LHS: a fresh, *wide* copy of the original scan.**

```cpp
// src/optimizer/late_materialization_helper.cpp:7
unique_ptr<LogicalGet> LateMaterializationHelper::CreateLHSGet(const LogicalGet &rhs, Binder &binder) {
	auto table_index = binder.GenerateTableIndex();
	auto new_get = make_uniq<LogicalGet>(table_index, rhs.function, rhs.bind_data->Copy(), rhs.returned_types,
	                                     rhs.names, rhs.virtual_columns);
	new_get->GetMutableColumnIds() = rhs.GetColumnIds();
	// ... copies projection_ids, parameters, named_parameters, input_table_types/names
	for (auto &filter_entry : rhs.table_filters) {
		auto &expr_filter = filter_entry.Filter().Cast<ExpressionFilter>();
		if (ExpressionFilter::ContainsInternalFunction(*expr_filter.expr, DynamicFilterScalarFun::NAME)) {
			continue;
		}
		// ... wrap a copy of the filter in CreateOptionalFilterExpression and push it onto the LHS too
	}
	return new_get;
}
```

This is deliberately a **second, independent `LogicalGet`** over the same
table (a fresh `TableIndex` via `GenerateTableIndex()`, so its column
bindings never collide with the original's) — not a modification of the
existing one. It starts out with *exactly the same column set* as the
original scan (`GetMutableColumnIds() = rhs.GetColumnIds()`); this LHS gets
narrowed later, but not here. The one thing it does *not* copy blindly is
`table_filters`: any filter built around the internal `DynamicFilterScalarFun`
marker is explicitly skipped (`continue`). §7.19.5's first test walks through
exactly why — a dynamic filter (the kind `ROW_GROUP_PRUNER`, pass #30, or
`TOP_N`'s own boundary-tracking machinery installs, whose bound changes at
*runtime* as the scan progresses) would be captured with a **stale** bound if
copied onto a second, independently-executing scan; every other, ordinary
static filter *is* copied, but wrapped as an **optional** filter via
`CreateOptionalFilterExpression` (`src/planner/filter/table_filter_functions.cpp:76`)
— a filter category the storage layer treats as pruning-only. §7.19.4 covers
precisely what that buys, down to the row-group check code.

`GetOrInsertRowIds` (the other function `late_materialization_helper.cpp`
exports) then adds the row-ID column(s) to this LHS `Get`, reusing an
already-projected row ID if one happens to already be there, or appending a
fresh one and bumping `projection_ids`/`types` to match:

```cpp
// src/optimizer/late_materialization_helper.cpp:31
vector<ProjectionIndex> LateMaterializationHelper::GetOrInsertRowIds(LogicalGet &get,
                                                                     const vector<column_t> &row_id_column_ids,
                                                                     const vector<TableColumn> &row_id_columns) {
	// ... search column_ids for an existing match; else push a new ColumnIndex(row_id_column_id)
	// and, if projection_ids/types are non-empty, extend those too
}
```

This same helper function is not private to this one pass — §7.19.5.3 covers
a second, independent caller of exactly this pair of functions.

**Step 3 — build the final projection list *before* touching the RHS.**
The comment in the source is explicit about the ordering constraint: *"we do
this before constructing the RHS because that alters the original plan"*
(`:268-269`). The final `SELECT`-list expressions — the ones the query
actually asked for — get copied from the original root `LogicalProjection`
if one exists (or synthesized directly from the LHS `Get`'s own columns if
the query has no projection above the scan at all), and then every column
reference inside them is rewritten, one intervening operator at a time, to
point at the **LHS**'s bindings instead of the original chain's
(`ReplaceExpressionReferences`, `:152-157`, driven by `GetExpression`,
`:130-150`). This is the single most consequential decision in the whole
pass: **the wide output columns get re-evaluated fresh, from the LHS `Get`**,
not carried through from wherever the original plan computed them. §7.19.5.3
is entirely about the sharp edge this creates for anything more complex than
a plain column reference.

**Step 4 — decide the final sort order.** For a `LOGICAL_TOP_N` root, the
final result still needs to come out sorted by the *original* `ORDER BY`
key(s) — the semi-join below does not preserve any particular order, so the
pass re-emits `top_n.orders` as a fresh `LogicalOrder` sitting *above* the
whole reconstructed tree (`:295-301`). For `LOGICAL_LIMIT`/`LOGICAL_SAMPLE`,
there's no meaningful output order to restore, so instead the pass orders by
**row ID** (`:302-309`) — a detail that matters for one specific reason:
sorting by row ID after the join, rather than leaving the join's output
order arbitrary, is what makes `LIMIT ... OFFSET` behave correctly and
deterministically (row ID order is the closest thing to "the order the table
was stored in" that a `LIMIT`/`OFFSET` without an explicit `ORDER BY` is
implicitly relying on).

**Step 5 — construct the RHS: the *old* pipeline, relabeled, with a row-ID
column threaded through.** `ConstructRHS` (`:28-79`) walks back down from the
`LIMIT`/`TOP_N`/`SAMPLE` node to the `Get` — the exact same chain that was
just read from, now mutated in place — and pushes the row-ID column up
through any intervening `Projection` (as a new projected expression) or
`Filter` (extended into its `projection_map` if it has one) it finds along
the way. The comment here is equally direct about what does *not* happen
yet: *"this is essentially the old pipeline, but with the `rowid` column
added... we don't [remove unused columns] here yet though — we do this in a
later step using the `RemoveUnusedColumns` optimizer"* (`:313-315`). At this
exact point in the rewrite, the RHS is **not yet narrow** — it's the entire
original wide scan-through-filter-through-projection chain, still reading
every column it always did, just with one more column (row ID) tacked on.

**Step 6 — the semi-join itself, and the table-index handoff.** The RHS's
column bindings get relabeled onto a brand-new `TableIndex`
(`optimizer.binder.GenerateTableIndex()`, `:323`), propagated down through
the RHS's own operators by `ReplaceTopLevelTableIndex`
(`:81-121`, walking through `Projection`/`Get`/`TopN`/`Filter`/`Sample`/
`Limit` nodes specifically — anything else throws `InternalException`, which
is the pass's own internal consistency check that the walk really did stay
inside the shape §7.19.2 already restricted eligibility to). Then, finally,
the join:

```cpp
// src/optimizer/late_materialization.cpp:332
auto join = make_uniq<LogicalComparisonJoin>(JoinType::SEMI);
join->children.push_back(std::move(lhs));   // children[0] — the wide, fresh Get
join->children.push_back(std::move(op));    // children[1] — the narrow-to-be, old pipeline

for (idx_t r_idx = 0; r_idx < row_id_columns.size(); r_idx++) {
	JoinCondition condition(
	    /* lhs row-id ref */, /* rhs row-id ref */, ExpressionType::COMPARE_EQUAL);
	join->conditions.push_back(std::move(condition));
}
```

This is a **`SEMI` join** — LHS rows are kept exactly when a matching row-ID
exists on the RHS, with no duplication and no RHS columns surviving into the
join's own output. §7.19.4 is entirely about what this becomes physically
and why the roles (which side is "small," which side actually gets scanned
sequentially) work out the way they do.

**Step 7 — wrap in the final projection (and, for `TOP_N`, the final
order), then let `RemoveUnusedColumns` do the actual narrowing.** The
join gets wrapped in the `final_proj_list` from Step 3 (re-pointed, one more
time, at the join's now-final table index) and, for the `TOP_N` case, the
`final_orders` from Step 4 sits *above* that projection (since the
projection strips the row-ID column the ordering no longer needs); for
`LIMIT`/`SAMPLE`, the order sits *below* the final projection instead, since
that ordering is *by* row ID, a column the final projection is about to
drop. Only at the very end does the pass call out to a different optimizer
entirely:

```cpp
// src/optimizer/late_materialization.cpp:385-387
RemoveUnusedColumns unused_optimizer(optimizer);
unused_optimizer.VisitOperator(op);
return true;
```

This is the moment the RHS side actually becomes narrow. `RemoveUnusedColumns`
(pass #23/#38 in §7.4's own numbered list, `src/optimizer/remove_unused_columns.cpp`)
walks the *entire, freshly reconstructed* tree top-down and prunes any column
an operator produces that nothing above it reads. Since the final
`SELECT`-list projection now reads its wide columns exclusively from the
**LHS** (Step 3's rewiring), and the RHS's only remaining consumer is the
join condition (which only needs the row-ID column) plus whatever the
original filter/sort key genuinely required, every wide payload column the
RHS chain still nominally reads gets pruned at this step — right down to the
`LogicalGet` at the bottom of the RHS chain, whose own `column_ids` shrink to
match. **This is the single place in the whole rewrite where "narrow RHS"
transitions from a description of intent to an actual, physically enforced
property of the tree** — everything before this point in `TryLateMaterialization`
is plumbing to get the column bindings and table indices to line up so that
this generic, unrelated pass can do the pruning correctly.

## 7.19.4 What "join back in" actually means, physically

A `LogicalComparisonJoin(JoinType::SEMI)` with a single equality condition on
an integral column reaches `PhysicalPlanGenerator::CreatePlan` (Chapter 9)
and, because the condition `has_equality` and merge/IE-join aren't preferred
by default, gets built directly as a `PhysicalHashJoin`
(`src/execution/physical_plan/plan_comparison_join.cpp:242-247`):

```cpp
if (has_equality && !prefer_range_joins) {
	auto &join = Make<PhysicalHashJoin>(op, left, right, std::move(op.conditions), op.join_type, /* ... */);
	return join;
}
```

This is directly observable, not inferred — every one of §7.19.5's tests
that checks the plan shape does so by grepping `EXPLAIN`'s physical output
for exactly `HASH_JOIN.*SEMI`. It's worth stating the negative fact plainly,
because it's easy to assume otherwise: **there is no specialized "fetch by
row-ID list" physical join operator anywhere in this codebase.** This pass
does not invent new execution machinery; it reuses `PhysicalHashJoin`,
off the shelf, exactly as it would be used for any other equi-join anywhere
else in a plan. That's a deliberate contrast worth drawing against Chapter
16's ART discussion, which covers a genuinely *different* mechanism for
"turn a row ID into a row": an ART leaf hands back a row ID, and "turning
row ID `4,918,204` into the actual tuple... is a job for the storage layer:
locate which row group contains that row ID, then fetch the corresponding
value out of each requested column's data for that row group at the right
offset" (Chapter 16 §16.8) — a direct, positional, O(1)-per-column-per-row
fetch used for point lookups (index probes, `UPDATE`/`DELETE` by row ID).
`LATE_MATERIALIZATION`'s row-ID reconciliation is not that mechanism. It is
a *join*: a second full (or zonemap-prunable) sequential scan of the base
table, correlated against a small hash table, not a batch of direct
positional row fetches.

### Which side builds, which side probes

`PhysicalHashJoin`'s own source pins this down precisely — `children[0]` is
the probe side, `children[1]` is the build side:

```cpp
// src/execution/operator/join/physical_hash_join.cpp:458, :465
build_side_multi_source = BuildSideHasMultipleSources(op.children[1].get());
// ...
probe_types = op.children[0].get().GetTypes();
```

The rewrite in §7.19.3 Step 6 pushes `lhs` (the wide, fresh `Get`) as
`children[0]` and `op` (the narrow-after-`RemoveUnusedColumns`, filtered/
sorted/limited chain) as `children[1]`. So: **the narrow, row-count-capped
side (≤50 rows by default, or the `TOP_N`/`LIMIT`/`SAMPLE` bound) is the
build side** — a tiny in-memory hash table over a handful of row IDs — and
**the wide side is the probe side**, streamed through the join sequentially,
tested one row at a time against that tiny hash table, with only matching
rows surviving. This is the sensible assignment (build the small side, probe
with the large one), and it falls out of the rewrite's own child ordering
without needing `BUILD_SIDE_PROBE_SIDE` (pass #27, which already ran earlier
in the pipeline and never gets a second look at this newly-created join, per
§7.2's one-shot architecture) to intervene.

### Three separate, stacking reasons the wide (probe) side doesn't have to read everything

Because the wide side is a genuine sequential probe scan, the honest
answer to "does this really turn a million wide reads into ten?" is
**it depends on how tightly the surviving row IDs cluster** — but there are
three independent, source-verified mechanisms that push the answer toward
"very few reads" whenever they apply, and it's worth naming all three rather
than treating this as one trick:

**1. Structural (unconditional): the narrow side reads zero wide-column
bytes, guaranteed.** `RemoveUnusedColumns` (§7.19.3 Step 7) physically prunes
the wide payload columns out of the RHS `Get`'s own `column_ids`. The RHS
scan — the one that does the actual filtering/sorting/comparing — never
opens, decompresses, or pages in a single wide `ColumnSegment` (§17.2.1),
regardless of table size, row-ID clustering, or anything else. This half of
the win is not probabilistic.

**2. Static, plan-time: the copied "optional" filter enables zonemap
pruning on the wide side too.** §7.19.3 Step 2 showed every non-dynamic
`table_filter` from the original `WHERE` clause gets copied onto the LHS,
wrapped via `CreateOptionalFilterExpression`
(`src/planner/filter/table_filter_functions.cpp:76`). "Optional" here means
exactly what Chapter 17 §17.5 already established zonemaps do generically —
`RowGroup::CheckZonemap` runs this filter against a row group's min/max
*before* any bytes are read, and once a row group survives that check, the
filter is marked `SetFilterAlwaysTrue` so it is **not** re-evaluated
per-row inside the row group (`src/storage/table/row_group.cpp:788-791`,
guarded by `ExpressionFilter::IsRootNonSelectivityOptionalFilter`) — its only
job on the LHS is row-group-level pruning; correctness never depends on it,
since the semi-join's row-ID match is what actually filters correctly. This
is exactly the "physical clustering of the filtered column" story Chapter 17
§17.5 already told in full: how much this specific mechanism helps is a
question of whether `last_purchase_date` (in this section's running example)
happens to correlate with row-group layout, not of the filter's abstract
selectivity.

**3. Dynamic, at runtime: row ID itself gets a free, exact row-group check
— and, plausibly, a min/max value pushed from the build side.**
`RowGroup::CheckZonemap` special-cases any filter whose target is the row-ID
column specifically:

```cpp
// src/storage/table/row_group.cpp:778-781
if (base_column_index.IsRowIdColumn()) {
	// the row ids in this row group span exactly [row_start, row_start + count)
	prune_result = CheckRowIdFilter(filter, row_start, row_start + count);
}
```

Unlike an ordinary column zonemap, this check needs no stored per-segment
statistics at all — a row group's row-ID range is always exactly
`[row_start, row_start + count)`, computable for free from the row group's
own bookkeeping. This is *exactly* the fact Chapter 16 leans on when it says
row IDs correspond directly to physical row-group position: a row group's
row IDs are, by construction, a contiguous slice of the whole table's row-ID
space, laid out in the same order the table itself is laid out in (§17.2.1).
Whether a *runtime* min/max range over the actual surviving row IDs from the
join's build side gets pushed down onto this exact check is the one claim in
this section this book cannot fully verify with a dedicated test: Chapter 9
§9.3.2 documents that "a completed hash join's build side [can] hand a
dynamic filter... back down into the probe side's scan," and
`JoinFilterPushdownUtil::JoinTypeIsSupported`
(`src/optimizer/join_filter_pushdown_optimizer.cpp:129-145`) explicitly
allows `JoinType::SEMI` (it's absent from the unsupported-types list, so it
falls into the `default: return true` branch) — meaning the freshly-created
semi-join from this pass is, generically, eligible for exactly this
treatment from `JOIN_FILTER_PUSHDOWN` (pass #42, which runs after this pass
in the fixed sequence and is not restricted to joins the optimizer created
before it ran). No test in §7.19.5's suite specifically asserts a runtime row-ID
filter appears in a late-materialization-created join's `EXPLAIN ANALYZE`
output, so this is stated as *structurally eligible and mechanically
plausible*, not as directly observed.

Put together: for a plain `LIMIT`/`OFFSET`/row-count `SAMPLE` — where the
surviving row IDs are, by construction, a small contiguous range — reasons 2
and 3 both bite hard, and "a handful of reads instead of the whole table" is
close to literally true. For a `TOP_N` (`ORDER BY ... LIMIT`) whose sort key
is *uncorrelated* with physical row-group layout, the surviving row IDs can
be scattered arbitrarily across the whole table's row-ID range, in which case
reasons 2 and 3 individually may buy little — but reason 1 is completely
unconditional. Even in that worst case, the guaranteed win is: the narrow
RHS side never reads a single wide byte, and the wide LHS side reads its wide
columns via one plain sequential scan feeding a cheap, single-column-key hash
probe, rather than having those same wide columns copied into (and compared
inside) a heap-based `PhysicalTopN`'s internal candidate storage a million
times over. This is why §7.19.6's worked example is careful to state the win
in terms of processing/materialization avoided unconditionally, with the
wide-read-avoidance framing reserved for the cases (§7.19.5.1's `WHERE
grp = 1` test, and the `LIMIT`/`OFFSET` Parquet cases in §7.19.5.5) where it
is directly demonstrated.

## 7.19.5 Walking the test suite

Five real files, spanning four different concerns, exercise this pass. Two
of them are pure regression tests for bugs the fuzzer and real GitHub issues
found; reading them is the fastest way to understand exactly where this
rewrite's sharp edges are.

### 7.19.5.1 `test/sql/optimizer/late_materialization_table_filters.test` — filters on the LHS, done right and done wrong

This file builds a real 100,000-row table (`lm_test`, with `grp` cycling
through 100 groups and five `md5`-hashed `VARCHAR` "payload" columns standing
in for wide data) specifically so `grp = 1` narrows to exactly 1,000
matching rows, spread evenly across the table by construction ("Insert
100K rows... interleaved to spread rowids"). Five scenarios, each isolating
one specific correctness property from §7.19.3's rewrite:

1. **Baseline: `WHERE grp = 1 AND ts BETWEEN ... ORDER BY ts LIMIT 20` must
   return identical results with `late_materialization_max_rows = 100`
   (pass enabled, since the `LIMIT 20 ≤ 100`) and with it set to `0` (pass
   effectively disabled).** Both the plan check (`EXPLAIN` must show
   `HASH_JOIN.*SEMI`) and the row-for-row output are asserted, including a
   direct check that a wide `payload1` column survives the round-trip intact
   (`payload1 = md5(id::VARCHAR || 'p1')`) — this is the test proving Step 3
   of §7.19.3 (re-evaluating the `SELECT` list against the LHS) reconstructs
   the exact original value, not some corrupted or stale one.
2. **`ORDER BY ts LIMIT 3` with no `WHERE` at all: the dynamic boundary
   filter must *not* be copied to the LHS.** The comment states the bug this
   guards directly: *"The `TOP_N` optimizer pushes a dynamic boundary filter
   into the RHS scan. If this filter were copied to the LHS, it would use a
   stale boundary and incorrectly filter out rows."* This is precisely the
   `DynamicFilterScalarFun::ContainsInternalFunction` skip in `CreateLHSGet`
   (§7.19.3 Step 2) — `TOP_N`'s own dynamic filter narrows its bound
   progressively as the heap fills, which is correct behavior for the *one*
   scan it's attached to, but would be a frozen, wrong snapshot if blindly
   copied onto a second, independent scan.
3. **`WHERE grp = 1 ORDER BY ts LIMIT 3` (filter on a *different* column from
   the sort key): the `grp` filter is copied to the LHS as an optional
   filter; the `EXPLAIN` regex explicitly checks for
   `SEQ_SCAN.*optional.*grp.*` on the LHS side.** This is §7.19.4's reason 2
   in action, directly asserted in a test rather than merely argued for.
4. **`WHERE ts >= '2024-03-01' ORDER BY ts LIMIT 3` (filter on the *same*
   column as the sort key): the whole column's filter is skipped on the LHS,
   not just the dynamic part.** The comment is explicit about the tradeoff:
   *"When `WHERE` and `TOP_N` dynamic filter are on the same column, they are
   ANDed together. The entire column filter is skipped on the LHS because it
   contains a dynamic filter. This loses zone-map pruning for the `WHERE`
   condition on that column, but guarantees correctness."* This is a
   deliberate, documented conservatism: rather than trying to split a
   combined static-AND-dynamic filter on one column into "the static half is
   safe to copy, the dynamic half isn't," the pass drops the whole thing —
   correct, at the cost of losing the zonemap benefit for that one column in
   that one case.
5. **`WHERE rowid = 5 ORDER BY ts LIMIT 1`: a filter on the `rowid` virtual
   column itself must be copied to the LHS like any other filter**
   (`late_materialization_table_filters.test:225-243`). The plan must match
   `HASH_JOIN.*SEMI.*SEQ_SCAN.*optional.*rowid.*`, and the query must return
   the row `6, 7, 2024-01-01 00:06:00`. `rowid` is not one of the table's
   real columns. So `CreateLHSGet` cannot look up a filter column's type in
   `rhs.returned_types[col_idx]`, a list that only holds the real columns: a
   virtual column's index would point past its end. It asks
   `rhs.GetColumnType(column_ids[...])` instead, which knows about virtual
   columns (`src/planner/operator/logical_get.cpp:211-223`).

### 7.19.5.2 `test/sql/optimizer/late_materialization_pushdown_extract.test` — interaction with struct-field pushdown, issue #24252

`STRUCT` columns get their own optimizer trick — pulling a `s.a = 1` style
predicate out into a pushdown-extract column at the scan, with the parent
struct column itself potentially pruned if nothing else needs it. This
file's core case: `SELECT s, x FROM lm_pushdown_extract WHERE s.a = 1 ORDER
BY id LIMIT 1` — struct-field pushdown and column pruning have already
happened on the RHS scan *before* `TryLateMaterialization`'s eligibility walk
runs, which means the LHS's projection-ID bookkeeping has to be rebuilt
against an already-pruned column set, not the original, unpruned one. The
comment names the exact bug this regression-tests: *"The RHS scan of the
late materialization rewrites the struct filter into a pushdown-extract
column and prunes column `x`. The projection ids must be rebuilt against the
pruned column set, otherwise resolving the rowid column fails (issue
#24252)."* — a `GetOrInsertRowIds` bookkeeping bug specifically triggered by
struct pushdown having already changed what "the scan's column set" means by
the time this pass runs. A second sub-case in the same file — a filter like
`s.a + s.b = 12` that references *two* different extracted struct fields at
once — cannot be pushed into the scan as a single pushdown-extract predicate
at all, so it stays a plain `LogicalFilter` above the scan; the test confirms
the scan then simply "falls back to outputting all its columns in scan
order," and that late materialization still produces the right answer even
in that degraded (non-pushed-down) case.

### 7.19.5.3 `test/sql/optimizer/topn_window_elimination_late_materialization.test` — a second pass, sharing this pass's own plumbing

This file is not testing `LATE_MATERIALIZATION` (`optimizer.cpp:465`)
itself — every query in it is a `QUALIFY row_number() OVER (PARTITION BY
... ORDER BY ...) = 1` idiom, which is `TOP_N_WINDOW_ELIMINATION`'s territory
(pass #36, §7.4), a pass that runs *after* `LATE_MATERIALIZATION` in the
fixed sequence and rewrites a per-group top-N window computation into a
`GROUP BY`-shaped plan instead. Its full rewrite mechanics are in §7.18.3
(`18-topn-and-topn-window-elimination.md`). What belongs *here* is that `TOP_N_WINDOW_ELIMINATION`
reaches directly into `late_materialization_helper.cpp` — the file this
section is explicitly also covering — and calls
`LateMaterializationHelper::CreateLHSGet` and
`LateMaterializationHelper::GetOrInsertRowIds` itself
(`src/optimizer/topn_window_elimination.cpp:1183-1184, 1261-1269`) to build
its *own*, independent row-ID-keyed reconstruction of a per-group winner's
full row. `late_materialization_helper.cpp`, in other words, is not private
scaffolding for one pass — it's the shared "build a narrow, row-ID-carrying
twin of a scan" toolkit two separate, unrelated-in-trigger-condition passes
both need.

This shared plumbing comes with a shared sharp edge, and this test file
exists specifically to police it. `TopNWindowElimination::IsRecreatableByLateMaterialization`
states the constraint in its own doc comment, which is worth quoting in
full because it is the single clearest statement in the whole codebase of
*why* a row-ID reconstruction can silently produce wrong answers:

```cpp
// src/optimizer/topn_window_elimination.cpp:195-200
//! Late materialization recreates a payload by re-scanning the base table column and re-applying a single default
//! (strict) cast at the topmost projection. That only reproduces the original value when the projection expression
//! is a plain column reference, optionally wrapped in a single non-try cast. A value-transforming expression
//! (e.g. parse_filename(col)) cannot be rebuilt that way. A try cast would be reconstructed as a strict cast
//! (losing its NULL semantics), and a deeper cast chain would lose its intermediate conversions, so none of those
//! shapes are eligible for late materialization.
bool IsRecreatableByLateMaterialization(const Expression &expr) {
	if (expr.GetExpressionClass() == ExpressionClass::BOUND_COLUMN_REF) {
		return true;
	}
	if (BoundCastExpression::IsCast(expr) && !BoundCastExpression::IsTryCast(expr.Cast<BoundFunctionExpression>())) {
		return BoundCastExpression::Child(expr.Cast<BoundFunctionExpression>()).GetExpressionClass() ==
		       ExpressionClass::BOUND_COLUMN_REF;
	}
	return false;
}
```

This is the exact same structural risk §7.19.3 Step 3 flagged for the
"plain" `LATE_MATERIALIZATION` pass (re-evaluating a `SELECT`-list expression
fresh against the LHS, rather than carrying forward whatever value the
original pipeline computed) — `TOP_N_WINDOW_ELIMINATION`'s version of the
same rewrite just happens to have a named, tested function guarding it,
because the "top-1-per-group" query shape this pass targets makes
value-transforming payload expressions far more common in practice than
they are in a plain `TOP_N`. Every scenario in this test file is a specific,
named regression against a specific way that guard could be (or once was)
wrong:

- **Issue #22849**: `parse_filename(path)` as a `SELECT`-list expression. The
  window computation *is* still eliminated (`FILTER.*WINDOW` does not appear
  in the optimized plan), but the `<!REGEX>` check on the *next* line asserts
  the rowid semi-join `COMPARISON_JOIN.*rowid = rowid` does **not** appear
  either — `parse_filename` is a value-transforming function of a base
  column, exactly the case `IsRecreatableByLateMaterialization` returns
  `false` for, so this query correctly falls back to carrying the computed
  `filename` value through the aggregate-based rewrite directly, instead of
  trying (and failing) to reconstruct it via a raw column re-read.
- **Issue #22096** (implicitly, via the "issue #22096 behaviour" comment): a
  *plain* cast of a base column (`c::TIMESTAMPTZ`) *is* eligible — this is
  exactly the second branch of `IsRecreatableByLateMaterialization`, "a
  strict, non-try cast whose child is a plain column reference." The
  `COMPARISON_JOIN.*rowid = rowid` regex here is a positive `<REGEX>` check
  (not `<!REGEX>`), confirming late materialization *does* fire for this
  shape, and the value round-trips correctly.
- **Issue #24609, part one**: `TRY_CAST(val AS INTEGER)`. The window
  elimination itself still happens, but late materialization must not,
  because — as the comment states directly — *"the rowid late-materialization
  path reconstructs the payload with a strict default cast, turning `TRY_CAST`
  failures of winning rows into Conversion Errors."* The test confirms the
  un-castable winning row (`'oops'`) correctly yields `NULL` rather than
  throwing, which is only true because `IsRecreatableByLateMaterialization`'s
  `!BoundCastExpression::IsTryCast(...)` check excluded this shape from the
  row-ID reconstruction path entirely.
- **Issue #24609, part two**: a *nested* cast chain,
  `CAST(CAST(x AS INTEGER) AS DOUBLE)`. Reconstructing this via "one default
  cast at the topmost projection" (as the doc comment puts it) would apply
  only the *outer* `DOUBLE` cast to the raw `x` column, silently skipping the
  intermediate truncation to `INTEGER` — the test confirms the correct,
  truncated value (`4`, not `4.4` from `x = 4.4`) comes out, precisely
  because the nested-cast shape also fails
  `IsRecreatableByLateMaterialization`'s "child is a plain column reference"
  check and falls back to carrying the value through directly.
- **A value-transforming partition-key projection** (`g + 1 AS gp`, used as
  the `PARTITION BY` key itself, not just a `SELECT`-list payload) is checked
  separately and also confirmed to block the row-ID join — the semi-join's
  output columns come from the *LHS* (the base-table re-scan), which has no
  way to reproduce `g + 1` without re-evaluating it, and this pass's own
  `CanUseLateMaterialization` walk (`topn_window_elimination.cpp:947-1017`)
  applies exactly the same `IsRecreatableByLateMaterialization` check to
  partition-key expressions as it does to payload expressions.
- **Issue #27812**: a cast chain spread over
  *two* stacked projections, `CAST(val AS BOOLEAN)` in one subquery and
  `CAST(val_b AS INT)` in the next. Each projection on its own passes
  `IsRecreatableByLateMaterialization`, so the walk also counts casts
  per column and refuses a second one. §7.18.3.6 shows the check.
- **Issue #27813**: a query that selects
  `rowid` as an ordinary payload column. The reconstruction join expects the
  row-id columns at the *end* of its left input, so the rewrite adds a
  projection that puts them there. Also covered in §7.18.3.6.

The same file's `CanUseLateMaterialization` also restricts which *join
types* it will walk through on the way down to the base scan —
`LogicalOperatorType::LOGICAL_COMPARISON_JOIN` is only tolerated for
`INNER`, `SEMI`, or `ANTI` (`:1010-1013`), and an `INNER` join specifically
sets `params.row_ids_may_have_duplicates = true` because *"an inner join can
produce the same base-table row more than once. The selected row IDs must
then be joined back using an inner join to preserve their multiplicity"*
(`:1014-1017`) — a detail worth flagging because it's the one place in either
pass where the join back isn't necessarily a `SEMI` join: `TryPrepareLateMaterialization`
picks `JoinType::INNER` instead of `JoinType::SEMI` specifically when row
IDs might repeat (`:1307-1309`, `use_inner_reconstruction = params.row_ids_may_have_duplicates
|| params.include_row_number`), since a `SEMI` join would silently collapse
duplicate row IDs down to one output row, which is wrong when the original
plan's join could legitimately produce that same base row more than once.

### 7.19.5.4 `test/fuzzer/duckfuzz/late_materialization_filter.test` — a fuzzer-found regression

The whole test is four lines of setup and a five-row table:

```sql
create table tbl(z int, i bool, j bool, k uhugeint);
insert into tbl (i, j, k) values (true, 'true', 3), (NULL, NULL, NULL), (false, 'false', 1);
select k from tbl where i and j limit 30;
```

The description names what the fuzzer actually found: *"NULL input to
`json_extract_string`"* — despite the query above containing no
`json_extract_string` call at all, which is itself informative: `duckfuzz`
generates a much larger randomized query, and the harness's regression-test
distillation process (standard practice across DuckDB's fuzzer test corpus)
reduces the failing case down to the smallest reproducer that still
triggers the underlying bug, which is why the surviving test's own
`WHERE`/`LIMIT` shape (a boolean `AND` filter combined with a row-count
`LIMIT`, over a table containing `NULL`s specifically in the filtered
columns) is what matters, not the originally-reported function name. The
table's second row (`(NULL, NULL, NULL)`) is the load-bearing detail: it
exercises `WHERE i AND j` with a `NULL`-valued boolean conjunct feeding
straight into the row-count-gated `LIMIT 30` path this pass rewrites,
confirming three-valued-logic filter evaluation on the LHS's optional-filter
copy (§7.19.4, reason 2) doesn't misbehave around `NULL` the way the
original bug apparently did.

### 7.19.5.5 `test/sql/copy/parquet/parquet_late_materialization.test` — where the wide-column-avoidance story is sharpest

Parquet is the clearest, most concrete illustration of this whole pass's
motivation, for a reason specific to Parquet's own on-disk format that
Chapter 17's row-group/column-segment model doesn't fully cover on its own:
a Parquet **column chunk** is a unit of *compressed, encoded* bytes — often
dictionary-encoded, RLE-encoded, or (for `VARCHAR`) length-prefixed and
packed — that generally has to be **decoded as a whole unit**, or at least a
whole page within it, before any individual value can be extracted. Unlike
DuckDB's own native storage, where per-vector, per-filter selective
decompression is a first-class internal capability (Chapter 17 §17.4's
compression framework is built around exactly that expectation), a naive
Parquet read of a wide `VARCHAR` column for a chunk containing 999 rows that
are about to be discarded and 1 that survives still has to run the decoder
over (a meaningful fraction of) that whole chunk. Late materialization
sidesteps this at the *plan* level, before the Parquet reader ever gets
involved: if the wide columns never appear in the narrow RHS scan's
projected column list at all (§7.19.3 Step 7's `RemoveUnusedColumns`
pruning), the Parquet reader is never asked to decode those column chunks
for the rows the narrow scan processes — full stop, independent of whatever
selective-decode sophistication that specific reader does or doesn't have
internally.

The test file itself is structured as a systematic tour of exactly which
shapes trigger the rewrite and which don't, checked via `EXPLAIN`'s
`logical_opt` output for the presence or absence of `COMPARISON_JOIN`:

- `SELECT * FROM test ORDER BY j DESC LIMIT 2` and `ORDER BY j, i LIMIT 2` —
  triggers (`<REGEX>`). Multiple sort columns work identically to one.
- `SELECT i FROM test ORDER BY i LIMIT 2` — does **not** trigger
  (`<!REGEX>`), with the comment stating exactly why: *"we cannot remove any
  columns by turning it into a row-id join"* — this is §7.19.2's structural
  `required_columns.size() >= get.GetColumnIds().size()` check in action:
  the only projected column (`i`) is also the sort key, so there is nothing
  to defer.
- `SELECT j FROM test WHERE j > 3 ORDER BY i LIMIT 2` — also does not
  trigger, for the same reason from a different angle: the one projected
  column (`j`) is already required by the pushed-down `WHERE` filter, so
  again nothing to strip out.
- `SELECT * FROM (SELECT i + random() AS i, ...) ORDER BY i LIMIT 2` — does
  not trigger: `i + random()` is volatile, hitting the `IsVolatile()` bail
  from §7.19.2.
- Multiple layers of projection nesting (`SELECT 100 + i i, ... FROM (SELECT
  -i i, ... FROM test)`) and expressions in the `ORDER BY` clause itself
  (`ORDER BY -j DESC`) both still trigger correctly — confirming the
  "flatten the chain of `source_operators`" logic in §7.19.3 Step 3 handles
  more than one intervening projection layer, and that the sort key itself
  is allowed to be a computed expression, not only a bare column.
- `LIMIT 2 OFFSET 2` triggers — this is exactly the "surviving row IDs are a
  small, contiguous range" case §7.19.4 singles out as the one where
  wide-read avoidance is closest to literal.
- `USING SAMPLE 2 ROWS` triggers; `USING SAMPLE 10%` does not — matching
  §7.19.2's `is_percentage` gate precisely, with the comment restating the
  reason directly: *"we can only use joins when we are sampling rows"* (a
  percentage sample can't be expressed as "the surviving rows have these N
  specific row IDs" ahead of time the way a row-count sample can).
- **A filter applied *after* a `TOP_N` subquery**
  (`SELECT * FROM (SELECT * FROM test ORDER BY j DESC LIMIT 2) WHERE i=8`) —
  demonstrating the rewrite composes correctly with an *unrelated* filter
  layered on top of its own output, not just filters that existed
  before the rewrite ran.
- A second copy of the whole test suite re-run against a `VARCHAR`-typed
  version of the same data (`printf('%02d', ...)` instead of integer
  arithmetic) — checking that the row-ID reconstruction correctly handles
  variable-width payload columns, not just fixed-width ones, which matters
  specifically for Parquet's own variable-length encoding.

## 7.19.6 A full worked example, before and after

Take this section's own running query from §7.19.1 and trace it through
both the unoptimized and rewritten shape, in the diagram convention this
book establishes in its front matter (arrows point in the direction data
flows upward at execution time):

```sql
SELECT customer_id, name, address, email, notes, segment, referrer,
       signup_channel, lifetime_value, last_purchase_date
FROM customers
WHERE last_purchase_date > '2020-01-01'
ORDER BY total_spend DESC
LIMIT 10;
```

Assume `customers` has 1,000,000 rows, `last_purchase_date > '2020-01-01'`
survives 400,000 of them, and (crucially, per §7.19.4) `total_spend` is *not*
physically correlated with row-group layout — the hard case, chosen
deliberately rather than the easy `LIMIT`/`OFFSET` case §7.19.5.5 already
covers well.

**Before** — the plan Chapter 5's binder would hand the optimizer, and what
survives if `LATE_MATERIALIZATION` (or the whole optimizer) is disabled:

```
┌─────────────────────────────────────┐
│ PhysicalResultCollector              │  root
└──────────────┬───────────────────────┘
               │
┌──────────────▼───────────────────────┐
│ PhysicalTopN                          │  ORDER BY total_spend DESC LIMIT 10
│ (bounded min-heap, holds ≤10 rows —   │  every one of those ≤10 held rows is
│  but every comparison/replace carries │  a FULL 10-column wide row
│  the full 10-column wide row)         │
└──────────────┬───────────────────────┘
               │  400,000 rows, EACH carrying all 10 output
               │  columns (customer_id, name, address, email,
               │  notes, segment, referrer, signup_channel,
               │  lifetime_value, last_purchase_date)
┌──────────────▼───────────────────────┐
│ PhysicalTableScan  customers          │  table_filters:
│ (SEQ_SCAN)                             │   last_purchase_date > '2020-01-01'
│ projects ALL 10 output columns for     │
│ EVERY row, filtered or not             │
└─────────────────────────────────────┘
```

Every one of the 1,000,000 base rows has its wide columns (`name`,
`address`, `email`, `notes` — the expensive ones) read and decompressed by
the single scan (filter pushdown, §7.4 #6, at least stops the scan from
projecting columns nothing in the query ever mentions — but every column
the query *does* mention, including the nine wide ones only needed in the
final output, gets read for all 1,000,000 rows, not just the 400,000 that
pass the filter, since a `TableFilterSet` predicate only decides which rows
survive *after* the vector's columns are already materialized — Chapter 17
§17.4's compression framework does support some early-exit tricks per
column, but there is no cross-column "skip decoding this column for rows
that already failed a different column's filter" mechanism at play here).
Of the 400,000 rows that pass the filter, all 400,000 full 10-column rows
get pushed into `PhysicalTopN`'s heap-maintenance logic, which compares
(and, on roughly `O(10 · log(400,000/10))` of those 400,000 rows, per the
classic order-statistics bound for a running top-k over unordered input,
actually copies) full wide rows in and out of its bounded storage.

**After** — the plan `LATE_MATERIALIZATION` produces (per §7.19.3):

```
┌─────────────────────────────────────────┐
│ PhysicalProjection                        │  final SELECT list, re-evaluated
│ customer_id, name, address, email, ...    │  fresh against the LHS scan below
└──────────────┬─────────────────────────────┘
               │  10 rows — the survivors
┌──────────────▼─────────────────────────────┐
│ PhysicalHashJoin   SEMI                     │  rowid = rowid
│  build side ◄── children[1] (RHS, narrow)   │
│  probe side ◄── children[0] (LHS, wide)     │
└──────┬─────────────────────────┬────────────┘
       │ (LHS/probe)             │ (RHS/build)
┌──────▼──────────────┐   ┌──────▼──────────────────────┐
│ PhysicalTableScan     │   │ PhysicalTopN                  │  ORDER BY total_spend
│ customers              │   │ (bounded heap over just       │  DESC LIMIT 10
│ (SEQ_SCAN)             │   │  total_spend + rowid — no      │
│ projects: customer_id, │   │  wide columns exist on this    │
│ name, address, email,  │   │  side AT ALL, after            │
│ notes, segment,        │   │  RemoveUnusedColumns pruned    │
│ referrer,               │   │  them out — §7.19.3 Step 7)    │
│ signup_channel,         │   └──────┬──────────────────────┘
│ lifetime_value,        │           │  400,000 rows, EACH carrying
│ last_purchase_date,     │           │  only 2 columns: total_spend, rowid
│ rowid                  │   ┌───────▼──────────────────────┐
│ table_filters:          │   │ PhysicalFilter                 │  last_purchase_date
│  last_purchase_date >   │   │ last_purchase_date >           │  > '2020-01-01'
│  '2020-01-01'  (OPTIONAL │   │ '2020-01-01'                   │
│  — zone-map pruning      │   └──────┬──────────────────────┘
│  only, §7.19.4 reason 2) │          │  1,000,000 rows, EACH
└─────────────────────────┘   ┌───────▼──────────────────────┐
                                │ PhysicalTableScan               │  customers
                                │ customers (SEQ_SCAN)           │  projects ONLY:
                                │ projects: total_spend,          │  total_spend,
                                │ last_purchase_date, rowid        │  last_purchase_date,
                                └─────────────────────────────┘   rowid — the 8 wide
                                                                    output columns are
                                                                    NEVER READ here
```

Reading this bottom-up on the right (RHS/build) side: the narrow scan reads
**3 columns** (`total_spend`, `last_purchase_date`, `rowid`) for all
1,000,000 rows — none of them wide — filters down to 400,000, and runs the
bounded top-10 heap over *those* 3-column rows, never touching `name`,
`address`, `email`, `notes`, or any of the other six wide columns anywhere
in this side of the tree. On the left (LHS/probe) side: the wide scan reads
all 10 output columns plus `rowid` for (up to) 1,000,000 rows, but only
because `total_spend`'s uncorrelated-with-storage-order assumption was
deliberately chosen as the hard case — a correlated sort key, or a `WHERE`
clause narrow enough to prune most row groups via the copied optional filter
(§7.19.5.1's `grp = 1` test), would cut that 1,000,000 down substantially.
Either way, the **guaranteed** win, independent of any physical-clustering
luck, is that the expensive comparison-and-heap-maintenance machinery in
`PhysicalTopN` — the part of this query actually doing repeated, per-row
work rather than one straight-through pass — now operates on 3-column rows
instead of 10-column rows, for all 1,000,000 (or 400,000 post-filter) of
them, and the wide columns get read, once, only by a plain sequential scan
feeding a cheap single-integer-key hash probe whose output is exactly 10
rows.

## 7.19.7 Summary and cross-references

- **Trigger**: exactly three logical operators — `LOGICAL_LIMIT`,
  `LOGICAL_TOP_N`, `LOGICAL_SAMPLE` (row-count only) — never `GROUP BY`,
  because a `LogicalAggregate`'s output columns are, by SQL's own rules,
  already exactly what it needs. Gated by a **row-count** cap on the
  surviving output (`late_materialization_max_rows`, default 50; up to
  1,000,000 for an unconditional, filter-free `LIMIT`), never by a
  column-count or byte-width threshold — the structural test is simply
  "would this actually drop at least one column the scan currently reads."
- **Rewrite shape**: one `LogicalGet` becomes two — a fresh, wide LHS copy
  (`LateMaterializationHelper::CreateLHSGet`) and the original chain,
  relabeled and later pruned down to a genuinely narrow RHS by
  `RemoveUnusedColumns` — joined by a `JoinType::SEMI`
  `LogicalComparisonJoin` on row ID equality, physically realized as an
  ordinary `PhysicalHashJoin` with the narrow side as build and the wide
  side as probe (Chapter 9's build/probe convention, verified directly
  against `physical_hash_join.cpp`).
- **Storage interaction**: not a specialized positional row-ID fetch (that
  mechanism exists, for index probes, and is Chapter 16's territory, not
  this pass's) — a genuine, reused, generic hash join. Wide-column-read
  avoidance stacks three ways: structurally guaranteed on the narrow side
  (§7.19.4 reason 1), zonemap-pruned on the wide side via a copied
  best-effort filter (reason 2, resting on Chapter 17 §17.5's zonemap
  machinery, plus row ID's own free, exact `[row_start, row_start+count)`
  check at `row_group.cpp:778-781`), and plausibly extended at runtime via
  `JOIN_FILTER_PUSHDOWN` (reason 3, stated with appropriate honesty about
  what is and isn't directly test-verified).
- **Shared plumbing, two callers**: `late_materialization_helper.cpp`'s
  `CreateLHSGet`/`GetOrInsertRowIds` aren't private to this pass —
  `TOP_N_WINDOW_ELIMINATION` (§7.4 #36, full mechanics in §7.18.3) calls them directly for its own,
  independently-triggered row-ID reconstruction, and shares this pass's
  exact same sharp edge: a reconstructed payload expression must be a plain
  column reference or a single non-`TRY` cast of one, or the row-ID join-back
  silently reconstructs the wrong value (`IsRecreatableByLateMaterialization`).
- **Chapter 2, one layer deeper**: §2.4's two lessons — "removing overhead
  beats adding cleverness" and a materialized DuckDB table's edge over
  `read_csv` coming from "stopping needing to touch most of the bytes... at
  all" — are exactly what this pass buys for a *single* scan of a *single*
  table: not a cleverer sort, not a cleverer filter, just fewer wide bytes
  and narrower rows moving through the expensive parts of the plan.
