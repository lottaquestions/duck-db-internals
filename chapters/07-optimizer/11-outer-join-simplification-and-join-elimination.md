# 7.11 — `OUTER_JOIN_SIMPLIFICATION` and `JOIN_ELIMINATION`

## 11.0 Where these two passes sit, and why they sit next to each other

§7.4 of the chapter overview lists the full, fixed ~45-call-site pipeline in
exact execution order. Two of those passes are the subject of this deep dive,
and they are numbered #16 and #20 in that list:

```
16. OUTER_JOIN_SIMPLIFICATION — optimizer.cpp:357, src/optimizer/outer_join_simplification.cpp
17. JOIN_ORDER               — optimizer.cpp:364, src/optimizer/join_order/         (Ch. 8)
18. AGGREGATE_REUSE          — optimizer.cpp:370, src/optimizer/aggregate_reuse.cpp
19. PARTIAL_AGGREGATE_PUSHDOWN — optimizer.cpp:379, src/optimizer/partial_aggregate_pushdown.cpp
20. JOIN_ELIMINATION         — optimizer.cpp:384, src/optimizer/join_elimination.cpp
```

Right after `JOIN_ELIMINATION`, as #21, runs `REDUNDANT_ORDER_KEYS`.

`OUTER_JOIN_SIMPLIFICATION` is one call site away from `JOIN_ORDER`
(`optimizer.cpp:357` then `:364`) — seven lines apart in the driver function,
run back-to-back with nothing structural in between. `JOIN_ELIMINATION` runs
four passes later (#20), after `JOIN_ORDER` and after two aggregate-shaped passes
that don't touch join *type* or join *presence* at all. That placement is not
an accident of alphabetical convenience, and §11.1.7 below explains exactly why
the first adjacency (`OUTER_JOIN_SIMPLIFICATION` immediately before
`JOIN_ORDER`) matters mechanically, with the specific DPhyp source code that
makes an `INNER` join reorderable and an `OUTER`/`LEFT`/`RIGHT`/`SEMI`/`ANTI`
join *not* reorderable in the same way.

Both passes are, in a loose one-sentence sense, about "a join doing less than
its full, general job because the specific query around it doesn't need that
generality" — but they attack two structurally different targets, and it is
worth stating the distinction up front before either gets a line of source
code, because the rest of this file is going to spend thousands of words on
mechanics that could otherwise blur together:

- **`OUTER_JOIN_SIMPLIFICATION` changes a join's *type***, without touching
  how many operators are in the tree. A `FULL OUTER JOIN` might become a
  `LEFT JOIN`; a `LEFT JOIN` might become an `INNER JOIN`; a `LEFT JOIN` might
  become an `ANTI` join. The join node is still there, still costs a build
  and a probe, still needs a physical operator chosen for it in Chapter 9 —
  it has just been proven to need fewer of the general "outer join" query
  guarantees than its type nominally promises.
- **`JOIN_ELIMINATION` removes a join node from the tree entirely.** Where it
  fires, the plan that comes out the other side has one fewer join operator
  than the plan that went in — the eliminated side's scan, and everything
  under it, is deleted from the tree, and the join's parent is wired directly
  onto the surviving side.

A join whose type has just been downgraded from `LEFT` to `INNER` by the first
pass is not, in general, eligible for the second pass's removal — becoming
`INNER` doesn't by itself prove anything about cardinality-preserving
uniqueness on either side, which is what `JOIN_ELIMINATION` actually needs (see
§11.2.3). The two passes are complementary, not sequential steps of one bigger
idea, even though they sit close together in the pipeline and both have "join"
and "simplification"-flavored names.

One more scoping note before diving in: the safety boundary around *pushing a
filter through* an outer join — the general question of when a predicate that
looks like it only touches one side of an outer join is actually safe to move
past the join at all — belongs to `FILTER_PUSHDOWN` (pass #6) and
`FILTER_PULLUP` (pass #5), covered in §7.6.3 of the chapter overview
(`00-overview.md`). This file's pass, `OUTER_JOIN_SIMPLIFICATION` (#16), runs
*ten passes later* and solves an adjacent but distinct problem: it doesn't
decide whether a filter can move past a join, it decides whether the join
itself has stopped being observably "outer" once a filter *already sitting
above it* is taken into account. §11.1.6 below walks a real DuckDB test
(`test/optimizer/pushdown/pushdown_filter_on_coalesced_equal_outer_join_keys.test`)
that sits squarely on the seam between the two passes, and is worth reading as
a cross-reference rather than a re-explanation of §7.6.3's material.

---

## 11.1 `OUTER_JOIN_SIMPLIFICATION`

**Purpose, in one sentence:** if a filter (or a set of accumulated
constraints derived from filters, `IS NULL`/`IS NOT NULL` checks, and
`ORDER BY`/aggregate column requirements above a join) proves that every row a
`LEFT`/`RIGHT`/`FULL OUTER` join's null-extension mechanism would produce is
going to be discarded or was never actually needed, that join's declared type
is downgraded to the cheapest type that still produces the same answer —
`FULL OUTER` to `LEFT`/`RIGHT` to `INNER`, or, in the specific case of a
predicate that requires a `LEFT` join's right side to be exactly `NULL`, all
the way to `ANTI`.

The class's own header comment states this precisely:

```cpp
// src/include/duckdb/optimizer/outer_join_simplification.hpp:27
//! Simplifies outer joins if NULL-extended rows are filtered in a way that changes the join semantics
class OuterJoinSimplification : public LogicalOperatorVisitor {
```

and `optimizer.cpp`'s own inline comment at the call site restates the
specific direction of travel:

```cpp
// src/optimizer/optimizer.cpp:355
// Simplifies FULL OUTER -> LEFT/RIGHT OUTER -> INNER if NULLs are filtered anyway
```

Neither comment mentions the `ANTI` rewrite by name, but it is very much part
of what the class does — it is arguably the more consequential of the two
rewrite families this pass performs, since `LEFT JOIN ... WHERE right.key IS
NULL` (the classic "find rows in the left table that have no match in the
right table" idiom) is common enough in real SQL that DuckDB gives it its own
dedicated `ExpressionType::COMPARE_...`-independent code path rather than
relying on the general downgrade logic to reach the same place by accident.

### 11.1.1 The two independent rewrite families, and which runs first

`OuterJoinSimplification::VisitOuterJoin` (`outer_join_simplification.cpp:302`)
is the entry point every `LEFT`/`RIGHT`/`FULL OUTER` (`JoinType::OUTER`) join
passes through, and it tries the two rewrites in a fixed order, falling
through to the second only if the first didn't apply:

```cpp
// src/optimizer/outer_join_simplification.cpp:302
void OuterJoinSimplification::VisitOuterJoin(LogicalComparisonJoin &join, LogicalOperator &op) {
	if (TryConvertLeftToAntiJoin(join)) {
		auto right_bindings = GetRightBindings(join);
		AddRequiredColumns(join.conditions);
		EraseNullRequiredColumns(*join.children[1]);
		VisitOperatorChildren(op);
		// only operators above the join see the right side as NULL, not the right child itself
		MarkEliminatedNullColumns(right_bindings);
		return;
	}

	SimplifyOuterJoinType(join);
	if (join.join_type == JoinType::INNER) {
		VisitOperator(op);
		return;
	}
	// ...
}
```

**Family 1 — `TryConvertLeftToAntiJoin`** (`outer_join_simplification.cpp:201`)
only ever applies to a plain `LEFT` join (checked at the top of the function:
`if (join.join_type != JoinType::LEFT) return false;` — a `RIGHT` or `OUTER`
join never reaches this rewrite at all, which §11.1.5 revisits as a real,
documented limitation). It looks for a very specific shape: some predicate
*above* this join requires one of the right side's own output columns to be
exactly `NULL` (tracked in `null_required_columns`, populated from an
`IS NULL` check — §11.1.2 covers exactly how), and that same column is also
one of the join's *own* equi-join condition columns on the right side:

```cpp
// src/optimizer/outer_join_simplification.cpp:201
bool OuterJoinSimplification::TryConvertLeftToAntiJoin(LogicalComparisonJoin &join) {
	if (join.join_type != JoinType::LEFT) {
		return false;
	}

	auto right_bindings = GetRightBindings(join);
	if (!HasNullRequiredColumns(right_bindings) || HasRequiredColumns(right_bindings)) {
		return false;
	}

	column_binding_set_t match_non_null_rhs;
	for (const auto &condition : join.conditions) {
		if (!condition.IsComparison() || !FiltersNulls(condition.GetComparisonType())) {
			continue;
		}
		ColumnBinding rhs_binding;
		if (!GetColumnBinding(condition.GetRHS(), rhs_binding)) {
			continue;
		}
		match_non_null_rhs.insert(rhs_binding);
	}

	for (const auto &binding : right_bindings) {
		if (null_required_columns.find(binding) == null_required_columns.end()) {
			continue;
		}
		if (match_non_null_rhs.find(binding) == match_non_null_rhs.end()) {
			continue;
		}
		join.join_type = JoinType::ANTI;
		return true;
	}
	return false;
}
```

Reading this as a list of conditions that must *all* hold before a `LEFT`
join becomes `ANTI`:

1. **`!HasRequiredColumns(right_bindings)`** — nothing above this join reads
   *any* actual value from the right side. `HasRequiredColumns` checks the
   `required_columns` set (populated top-down from every projection,
   aggregate, filter, and order-by expression the walk has passed through so
   far — see §11.1.2) against every one of the right child's output
   bindings. If anything upstream still wants, say, `r.value` for real (not
   just to test it against `NULL`), the rewrite is refused outright — an
   `ANTI` join, by definition, never produces the right side's columns at
   all, so keeping a column the query still needs would silently turn it
   into garbage.
2. **`HasNullRequiredColumns(right_bindings)`** — *something* above this join
   does require at least one right-side column to be `NULL`. This is the
   positive trigger: without an `IS NULL` requirement recorded on some
   right-side binding, there's nothing to convert.
3. **The `NULL`-required column has to be one of the join's own equality
   condition columns, specifically a column whose comparison type
   `FiltersNulls`** (i.e., not `IS [NOT] DISTINCT FROM` — see §11.1.2). This
   is `match_non_null_rhs`, built from the join's *own* conditions, and it is
   the piece of reasoning that makes the whole rewrite sound: if the join
   condition is `l.i = r.i`, then a left row that has *no* matching right row
   is exactly the set of rows the `LEFT` join null-extends on `r.i` — and
   "the null-extended rows" is precisely what `ANTI` returns. If the `NULL`
   requirement instead landed on some *other* right-side column not involved
   in the join key (`r.value`, say), that column can legitimately be `NULL`
   on a genuine *match* too (nothing about a `LEFT JOIN`'s semantics
   guarantees `r.value` is non-null on matched rows), so `r.value IS NULL`
   does **not** imply "there was no match," and the rewrite correctly
   declines. §11.1.3 walks a real test case built exactly to exercise this
   distinction.

Once the rewrite fires, `VisitOuterJoin` first walks the join's two children.
Only after that does it call `MarkEliminatedNullColumns(right_bindings)`.
This call records *every* right-side column (not just the one column that
triggered the rewrite) as "known to be `NULL`, and no longer produced by the
join at all". The set feeds the cleanup in `VisitProjection` and
`VisitFilter` (§11.1.4). Both cleanups run on the way back *up* out of the
recursion, so only operators *above* the join see the new facts.

> **Why the marking waits for the children.** If the columns were marked
> *before* the walk went down into the join's children, an `ON` predicate
> on the right side would break. Such a predicate, like `b.n IS NULL` in
> `t a LEFT JOIN t b ON a.i = b.i AND b.n IS NULL`, can end up as a filter
> *inside* the right child. The visitor would then see `b.n` marked as
> "always `NULL`", and `VisitFilter` would delete that filter as if it were
> redundant. But inside the right child, `b.n` is real data, so the join
> would match rows it should not match. The regression test is
> `test/optimizer/outer_join_simplification_anti_on_filter.test`. For the
> table `t(i, n) = {(1,1), (2,NULL), (3,3)}`, the query
> `SELECT a.i FROM t a LEFT JOIN t b ON a.i=b.i AND b.n IS NULL WHERE b.i IS NULL`
> must return `1` and `3`. Only row 2 has a partner whose `n` is `NULL`.

**Family 2 — `SimplifyOuterJoinType`** (`outer_join_simplification.cpp:236`)
is the more general downgrade, tried only when family 1 didn't fire (a
`RIGHT`/`OUTER` join always falls straight through to this family, since
family 1 refuses them unconditionally):

```cpp
// src/optimizer/outer_join_simplification.cpp:236
void OuterJoinSimplification::SimplifyOuterJoinType(LogicalComparisonJoin &join) {
	bool preserves_null_extended_rows[2] = {join.join_type == JoinType::LEFT || join.join_type == JoinType::OUTER,
	                                        join.join_type == JoinType::RIGHT || join.join_type == JoinType::OUTER};
	for (idx_t child_idx = 0; child_idx < 2; child_idx++) {
		for (const auto &binding : join.children[child_idx]->GetColumnBindings()) {
			if (null_filtered_columns.find(binding) != null_filtered_columns.end()) {
				// Rejecting NULLs in one child removes preservation of NULL extended rows for the other child.
				preserves_null_extended_rows[1 - child_idx] = false;
				break;
			}
		}
	}

	if (!preserves_null_extended_rows[0] && !preserves_null_extended_rows[1]) {
		join.join_type = JoinType::INNER;
		return;
	}

	if (preserves_null_extended_rows[0] && !preserves_null_extended_rows[1]) {
		D_ASSERT(join.join_type == JoinType::LEFT || join.join_type == JoinType::OUTER);
		join.join_type = JoinType::LEFT;
	} else if (!preserves_null_extended_rows[0] && preserves_null_extended_rows[1]) {
		D_ASSERT(join.join_type == JoinType::RIGHT || join.join_type == JoinType::OUTER);
		join.join_type = JoinType::RIGHT;
	} else {
		D_ASSERT(join.join_type == JoinType::OUTER);
		join.join_type = JoinType::OUTER;
	}
}
```

Read the indexing carefully, because it is the crux of the whole rewrite and
it is phrased in exactly the inverted way you'd expect from "rejecting NULLs
on one side undoes null-extension on the *other* side": `child_idx == 0` is
the join's left child, `child_idx == 1` is its right child.
`preserves_null_extended_rows[0]` tracks whether the **left** side's
null-extended rows (produced when this is a `RIGHT`/`OUTER` join and the left
side didn't match) are still observable; `preserves_null_extended_rows[1]`
tracks the same for the **right** side (`LEFT`/`OUTER`). The loop then walks
each child's *own* bindings and checks them against `null_filtered_columns` —
and when it finds that a binding belonging to child `child_idx` is
null-filtered (some predicate above the join rejects `NULL` on that exact
column), it flips `preserves_null_extended_rows[1 - child_idx]` to `false` —
the *other* index. This is the whole idea stated as an array-indexing trick:
a predicate that rejects `NULL`s on the **left** side's own columns is only
capable of doing that filtering *at all* on rows the join produced with the
left side genuinely present — and the only rows where the left side is
*absent* (all-`NULL`) are the ones a `RIGHT`/`OUTER` join's null-extension on
the left manufactures. So a filter rejecting `NULL`s on the left implies
those null-extended-on-the-left rows can't have survived to reach this
filter — which is exactly "the right side's null-extension privilege has been
undone," i.e. `preserves_null_extended_rows[1]` (right-null-extension) goes
false. Symmetrically for the right side.

Once both flags are computed: if *neither* side's null-extension is still
observable, the join is a plain `INNER` join in every way that matters — no
row this join could ever produce with one side entirely `NULL` would survive
downstream anyway, so keeping the more expensive outer-join machinery buys
nothing. If exactly one flag survives, the join downgrades to the
corresponding single-sided type (`OUTER`→`LEFT` or `OUTER`→`RIGHT` — the
`D_ASSERT`s at lines 255 and 258 confirm a `LEFT`/`RIGHT` join can only ever
have started with its *own* preservation flag true, so this branch is really
only reachable starting from a `FULL OUTER JOIN` losing one side's
preservation). If both flags survive, nothing changes — the join stays
`OUTER` (line 261's `D_ASSERT(join.join_type == JoinType::OUTER)` documents
that this is the only type that can reach this branch at all).

### 11.1.2 What "a filter rejects `NULL`" actually means, precisely

The task this book set for itself when opening this file was to find the
*exact* condition — not "does it look for `IS NOT NULL`," but what expression
shapes actually get recognized. The answer, read directly out of
`HandleFilterExpression` (`outer_join_simplification.cpp:97`), is broader than
a literal `IS NOT NULL` check, and it is worth walking branch by branch:

```cpp
// src/optimizer/outer_join_simplification.cpp:97
void OuterJoinSimplification::HandleFilterExpression(const Expression &expr) {
	if (expr.GetExpressionClass() == ExpressionClass::BOUND_OPERATOR &&
	    expr.GetExpressionType() == ExpressionType::OPERATOR_IS_NOT_NULL) {
		const auto &is_not_null = expr.Cast<BoundOperatorExpression>();
		HandleExpression(*is_not_null.GetChildren()[0]);
		AddRequiredColumns(expr);
		return;
	}

	if (expr.GetExpressionClass() == ExpressionClass::BOUND_OPERATOR &&
	    expr.GetExpressionType() == ExpressionType::OPERATOR_IS_NULL) {
		const auto &is_null = expr.Cast<BoundOperatorExpression>();
		if (!HandleIsNullExpression(*is_null.GetChildren()[0])) {
			AddRequiredColumns(expr);
		}
		return;
	}

	if (!BoundComparisonExpression::IsComparison(expr)) {
		AddRequiredColumns(expr);
		return;
	}

	AddRequiredColumns(expr);
	if (!FiltersNulls(expr.GetExpressionType())) {
		return;
	}
	const auto &comparison = expr.Cast<BoundFunctionExpression>();
	HandleExpression(BoundComparisonExpression::Left(comparison));
	HandleExpression(BoundComparisonExpression::Right(comparison));
}
```

Four branches, in order:

1. **Literal `IS NOT NULL`** — handled directly, `HandleExpression` records
   the child's column binding in `null_filtered_columns` (this is the case
   everyone expects).
2. **Literal `IS NULL`** — this is the *other* direction, and it feeds
   `null_required_columns` (used by `TryConvertLeftToAntiJoin`, §11.1.1), via
   `HandleIsNullExpression` → `GetNullPreservingColumnBinding`. If that
   helper declines (§11.1.2's cast discussion below), the whole predicate
   falls back to being merely a `required_columns` entry — "keep this
   column around, but don't draw any `NULL`-based conclusion from it."
3. **Anything that isn't a `BoundComparisonExpression` at all** — a `LIKE`, a
   `regexp_matches` call, a scalar UDF, a `CASE` expression, `x IN (...)` —
   is treated as an opaque, column-requiring expression only.
   **This is the direct, source-grounded answer to whether the pass reasons
   about predicates that merely *imply* non-null through some other
   mechanism**: it does not. `x LIKE '%foo%'` is, by ordinary SQL
   three-valued logic, exactly as `NULL`-rejecting as `x = 'foo'` when it
   sits in a `WHERE` clause (any strict function of `NULL` is `NULL`, and
   `WHERE NULL` discards the row) — but `HandleFilterExpression` has no
   branch that reasons about *arbitrary functions* rejecting `NULL`; it only
   recognizes the specific `BoundComparisonExpression` class of node. A
   `LIKE`-based filter above an outer join, however `NULL`-rejecting it
   really is at runtime, will not trigger this pass's downgrade.
4. **A genuine comparison** (`=`, `<>`, `<`, `<=`, `>`, `>=`, `IS [NOT]
   DISTINCT FROM`) — first unconditionally added to `required_columns` (the
   column has to survive regardless), then, **only if
   `FiltersNulls(expr.GetExpressionType())`**, both sides get
   `HandleExpression`'d into `null_filtered_columns`. `FiltersNulls` itself
   is the one-line answer to "does this pass only look for equality, or
   any comparison":

```cpp
// src/optimizer/outer_join_simplification.cpp:158
bool OuterJoinSimplification::FiltersNulls(ExpressionType comparison_type) {
	return comparison_type != ExpressionType::COMPARE_DISTINCT_FROM &&
	       comparison_type != ExpressionType::COMPARE_NOT_DISTINCT_FROM;
}
```

Every ordinary comparison type filters `NULL`s except the two explicitly
`NULL`-safe ones (`IS [NOT] DISTINCT FROM`, which are defined precisely to
*not* three-value-collapse on `NULL`). So **yes — `right_col > 5` marks
`right_col` as null-filtered**, exactly as much as `right_col = 5` or
`right_col <> 5` would; there is nothing equality-specific about this pass.
`HandleExpression` (called on both sides of any qualifying comparison) simply
asks `GetColumnBinding` whether *that side* resolves to a column (unwrapping
casts along the way, `outer_join_simplification.cpp:56`) — a literal constant
like the `5` in `right_col > 5` fails that check and contributes nothing,
while the column side succeeds and gets inserted. A two-sided comparison
between two real columns (`l.a > r.b`) inserts *both* sides.

### 11.1.3 The invertible-cast asymmetry, and why it exists

There are two different helpers for "does this expression resolve to a
column, looking through casts," and they deliberately disagree with each
other:

```cpp
// src/optimizer/outer_join_simplification.cpp:56
bool OuterJoinSimplification::GetColumnBinding(const Expression &expr, ColumnBinding &binding) {
	if (expr.GetExpressionClass() != ExpressionClass::BOUND_COLUMN_REF) {
		if (!BoundCastExpression::IsCast(expr)) {
			return false;
		}
		auto &cast = expr.Cast<BoundFunctionExpression>();
		return GetColumnBinding(BoundCastExpression::Child(cast), binding);
	}
	// ...
}

// src/optimizer/outer_join_simplification.cpp:69
bool OuterJoinSimplification::GetNullPreservingColumnBinding(const Expression &expr, ColumnBinding &binding) {
	if (expr.GetExpressionClass() != ExpressionClass::BOUND_COLUMN_REF) {
		if (!BoundCastExpression::IsCast(expr)) {
			return false;
		}
		auto &cast = expr.Cast<BoundFunctionExpression>();
		if (BoundCastExpression::IsTryCast(cast) ||
		    !BoundCastExpression::CastIsInvertible(BoundCastExpression::SourceType(cast), cast.GetReturnType())) {
			return false;
		}
		return GetNullPreservingColumnBinding(BoundCastExpression::Child(cast), binding);
	}
	// ...
}
```

`GetColumnBinding` (used for the `null_filtered_columns` / `>`-style /
`IS NOT NULL` direction) unwraps *any* cast unconditionally, invertible or
not. `GetNullPreservingColumnBinding` (used for the `null_required_columns` /
`IS NULL` direction, which feeds the `ANTI`-join rewrite) refuses to unwrap a
`TRY_CAST` at all, and refuses a plain `CAST` too unless
`BoundCastExpression::CastIsInvertible` says the specific source-type →
target-type pair is lossless in both directions.

This asymmetry is not an oversight — it is required for soundness, and the
two directions genuinely have different soundness requirements:

- **"Column `IS NOT NULL`" (or any `NULL`-rejecting comparison) surviving a
  cast is always safe to conclude**, invertible or not: if `CAST(r.i AS
  BIGINT)` is not `NULL`, the underlying `r.i` cannot have been `NULL`
  either — a cast of an actual `NULL` value is always `NULL` for any cast
  function DuckDB has, invertible or lossy. No invertibility requirement is
  needed for this direction, which is why `GetColumnBinding` doesn't check
  for it.
- **"Column `IS NULL`" surviving a cast is only safe to conclude backward
  through an *invertible*, non-`TRY` cast.** A `TRY_CAST` can turn a
  perfectly non-null value into a `NULL` result purely because the cast
  itself failed (`TRY_CAST('x' AS INTEGER)` is `NULL` even though `'x'` is
  not `NULL`) — so `TRY_CAST(r.s AS INTEGER) IS NULL` does **not** imply
  `r.s IS NULL`, and treating it as if it did would let the `ANTI`-join
  rewrite silently discard *matched* rows whose cast merely failed. Even a
  plain (non-`TRY`) `CAST` needs the specific source/target pair to be
  *invertible* — a `VARCHAR → INTEGER` cast is not injective in the sense
  this check requires (many strings are not valid integers at all, and the
  runtime behavior for those is to *throw*, not silently succeed), so it is
  excluded even though it isn't a `TRY_CAST`.

§11.1.5 below reproduces the exact test cases DuckDB's own test suite uses to
pin this distinction down, including one where the plain (non-`TRY`) cast
case is refused specifically *because* the underlying data would make the
literal query throw a runtime error on the very row the (declined) rewrite
would have needed to reason about.

### 11.1.4 The traversal: what propagates through which operator, and the September 2026 bug it took to get this right

`OuterJoinSimplification` is a top-down `LogicalOperatorVisitor`: it starts at
the plan's root and walks down, accumulating `required_columns`,
`null_filtered_columns`, and `null_required_columns` as it passes through
each operator, and only decides what to do with a join once it actually
*reaches* that join node (at which point everything accumulated on the path
from the root down to here is available). Each operator type gets its own
handling of exactly what survives the walk downward:

- **`VisitFilter`** (`outer_join_simplification.cpp:365`) splits the filter's
  (possibly `AND`-conjoined) predicates and runs `HandleFilterExpression` on
  each one — this is where §11.1.2's four branches actually get invoked.
  After recursing into its child, it does one more pass over its own
  expressions: any predicate that was itself an `IS NULL` check whose column
  ended up in `eliminated_null_columns` (i.e., the `ANTI`-join rewrite fired
  on the join below and made this exact `IS NULL` check now *always true*,
  since the column literally no longer exists as anything but a constant
  `NULL`) gets erased from the filter outright
  (`filter.expressions.erase_at(i)`, line 380) — the redundant `WHERE r.i IS
  NULL` that triggered the `ANTI` conversion in the first place doesn't
  need to be re-checked once the join itself already guarantees it.
- **`VisitProjection`** (`outer_join_simplification.cpp:331`) does two things:
  before recursing, it forwards `null_filtered_columns`/`null_required_columns`
  membership through any output column that's a direct pass-through column
  reference (so a renamed/re-projected column still carries its
  accumulated `NULL` facts down to whatever's below); after recursing back
  up, for any pass-through column whose *input* binding ended up in
  `eliminated_null_columns`, it replaces that projection's own expression
  with a literal typed `NULL` constant
  (`make_uniq<BoundConstantExpression>(Value(type))`, line 360) — §11.3's
  worked example shows this concretely.
- **`VisitAggregate`** (`outer_join_simplification.cpp:400`) and
  **`VisitUnsupportedOperator`** (`outer_join_simplification.cpp:412`) both
  *fork a brand new `OuterJoinSimplification` instance* rather than
  continuing the current one's accumulated state — an aggregate's grouping
  and aggregate expressions become the *entire* `required_columns` set for a
  fresh sub-walk of its child, and any operator type this visitor doesn't
  explicitly know how to reason through (the `default:` case at
  `outer_join_simplification.cpp:462`, whose comment reads *"Passthrough not
  supported. TODO: could pass through more operators like
  LOGICAL_UNION"*) conservatively starts a fresh sub-walk per child with
  nothing carried over but that child's own raw output bindings as
  `required_columns`. This means an outer join sitting *below* a `UNION`, a
  `DISTINCT`, or any other operator this visitor doesn't special-case gets
  analyzed in isolation, with none of the accumulated `NULL` facts from
  further up the tree available to it — a real, source-confirmed scope
  limit, not a bug: correctness requires this conservatism, since the
  visitor genuinely doesn't know how those operators would transform the
  accumulated constraints.
- **`VisitOrder`/`VisitTopN`** (`outer_join_simplification.cpp:385`/`:392`)
  both add every sort-key expression's columns to `required_columns` before
  going down. An outer join that feeds a sort key can't be simplified in a
  way that would stop producing a column the sort still needs. After that,
  the two differ. `VisitOrder` keeps walking with the same visitor.
  `VisitTopN` starts a *new* visitor for its child, and so does the
  `LOGICAL_LIMIT` case in `VisitOperator`:

```cpp
// src/optimizer/outer_join_simplification.cpp:392
void OuterJoinSimplification::VisitTopN(LogicalTopN &top_n, LogicalOperator &op) {
	for (const auto &order_node : top_n.orders) {
		AddRequiredColumns(*order_node.expression);
	}
	OuterJoinSimplification child_simplification(required_columns);
	child_simplification.VisitOperator(*op.children[0]);
}
```

```cpp
// src/optimizer/outer_join_simplification.cpp:452
	case LogicalOperatorType::LOGICAL_LIMIT: {
		OuterJoinSimplification child_simplification(required_columns);
		child_simplification.VisitOperator(*op.children[0]);
		return;
	}
```

  The new visitor gets a copy of `required_columns`, so no column the query
  still needs is lost. But it starts with empty `null_filtered_columns` and
  `null_required_columns`. So a `NULL` fact found *above* a `LIMIT` or a
  top-N never reaches a join *below* it.

> **Why `LIMIT` and `TOP_N` are barriers.** If they just called
> `VisitOperatorChildren` and passed every fact down, the result could be
> wrong (DuckDB issue #24542). A `LIMIT` picks rows *before* the filter above
> it runs. Take this query from
> `test/issues/general/test_24542.test`:
>
> ```sql
> SELECT * FROM (
>     SELECT l2.a, l2.ak, r2.b
>     FROM l2 LEFT JOIN r2 ON l2.ak = r2.bk
>     ORDER BY b DESC NULLS FIRST, a ASC NULLS LAST
>     LIMIT 2
> ) s
> WHERE s.b <= 20;
> ```
>
> The `LEFT JOIN` gives two rows with `b = NULL` (for `x` and `y`) and one
> matched row with `b = 10` (for `p`). The `NULL` rows sort first, so
> `LIMIT 2` keeps them. Then `s.b <= 20` drops both. The right answer is
> empty. Without the barrier, the pass would see `s.b <= 20` as a
> `NULL`-rejecting filter and turn the `LEFT JOIN` into an `INNER JOIN`
> *below* the limit. Then the limit would only see the matched row, and the
> query would wrongly return `(3, p, 10)`. The test checks that the result is empty with the pass on
> and with it off.

The single most instructive piece of this traversal, though, is a genuine
correctness bug that DuckDB fixed in September 2026 — worth walking through in
full because it is exactly the kind of subtlety that makes "just check if the column is required to be
NULL" harder than it sounds once joins *nest*. Commit `916aeeb64b`
("Do not assume NULL columns on right side of LEFT JOIN (#25211)", merged
2026-09-02, fixing `duckdb/duckdb#24540`) added exactly six lines:

```cpp
// src/optimizer/outer_join_simplification.cpp:189 (added by #25211)
void OuterJoinSimplification::EraseNullRequiredColumns(LogicalOperator &child) {
	for (const auto &binding : child.GetColumnBindings()) {
		null_required_columns.erase(binding);
	}
}
```

called from `VisitOuterJoin` right before descending into whichever side(s)
a join can null-extend:

```cpp
// src/optimizer/outer_join_simplification.cpp:319
AddRequiredColumns(join.conditions);
// NULL values in the NULL-extended side can be introduced by this join itself,
// so columns of that side are not guaranteed to be NULL below the join
if (join.join_type == JoinType::LEFT || join.join_type == JoinType::OUTER) {
	EraseNullRequiredColumns(*join.children[1]);
}
if (join.join_type == JoinType::RIGHT || join.join_type == JoinType::OUTER) {
	EraseNullRequiredColumns(*join.children[0]);
}
```

Here is the bug this fixed, in the DuckDB engineer's own words from the merge
commit: *"OuterJoinSimplification walks the plan top-down, recording columns
that a predicate forces to be NULL in `null_required_columns`. This allows
the optimizer to rewrite LEFT joins into ANTI joins. When an `IS NULL`
predicate sits above a LEFT join, the `IS NULL` is carried into the join's
right child, as a way to assume only NULL values will survive [...] This is
incorrect, however, since the LEFT JOIN will extend non-matching rows and
produce new NULLs."* Concretely: for a query with **two nested `LEFT JOIN`s**
— `a LEFT JOIN (b LEFT JOIN c ON ...) ON ...` — a `WHERE outer.some_col IS
NULL` sitting above the *outer* join would, before this fix, get carried
straight down through the outer join's right child and treated as "this
column is `NULL` everywhere in this subtree," including at the *inner* join
buried inside that subtree. But the outer join's own null-extension is
exactly what can *manufacture* that `NULL` for rows where the inner join's
subtree produced something entirely different (possibly non-`NULL`) — the
"`IS NULL`" fact is true of the *outer join's output*, not of the raw values
flowing through the inner join underneath it. Carrying it down incorrectly
let the optimizer rewrite the *inner* join into an `ANTI` join too, which is
wrong: `test/sql/join/left_outer/left_join_issue_24540.test`
(*"left join do not assume right child is null"*) is the regression test this
fix shipped with, and it is worth reading as a minimal, real repro:

```sql
-- test/sql/join/left_outer/left_join_issue_24540.test
SELECT * FROM (VALUES (1),(2),(NULL)) AS a(x)
LEFT JOIN (SELECT p.y AS y, q.z AS z
           FROM (VALUES (2),(3),(NULL)) AS p(y)
           LEFT JOIN (VALUES (1),(3),(NULL)) AS q(z) ON p.y <> q.z) AS s
ON a.x <> s.y
WHERE s.z IS NULL;
----
NULL	NULL	NULL
```

Note precisely what this fix does **not** touch: it only erases
`null_required_columns` (the `IS NULL`/`ANTI`-conversion direction) on the
join's null-extendable side before descending. `null_filtered_columns` (the
`IS NOT NULL`/downgrade-to-`INNER` direction, used by `SimplifyOuterJoinType`)
needed no equivalent fix, and there's a clean reason why not, worth stating
explicitly since it's the kind of asymmetry a careful reader should expect an
explanation for rather than take on faith: "this column's value, wherever it
ultimately comes from, is provably non-`NULL`" is a fact that remains true no
matter how many further null-extending joins sit *underneath* the point it
was established — if the *final* value reaching the filter is non-`NULL`,
then every join between the filter and the column's origin must have
happened to produce a non-`NULL` value for that specific surviving row,
including any nested join's own null-extension (a null-extending join either
passes a real value through unchanged, or replaces the whole side with
`NULL` — it can never take a real non-`NULL` value and hide it). "This
column's value is exactly `NULL`," by contrast, does *not* survive backward
through an intervening null-extending join, because that join can manufacture
a fresh `NULL` for rows whose *underlying* value below it was never `NULL`
at all — which is precisely the direction of error this fix closed.

### 11.1.5 Every case in `test/optimizer/outer_join_simplification.test`

The file opens with `PRAGMA explain_output = OPTIMIZED_ONLY;`, so every
`EXPLAIN`'s `logical_opt` column shown below is the plan *after* the full
optimizer pipeline has run, and every regex check is a coarse "does the join
type keyword still appear in the printed plan" test:

**`FULL OUTER` → nothing (`<!REGEX>:.*FULL.*` — the word "FULL" no longer
appears):**

```sql
SELECT l.i, r.i FROM range(3) l(i) FULL OUTER JOIN range(2) r(i) ON l.i = r.i
WHERE r.i IS NOT NULL ORDER BY 2;
```

`r.i IS NOT NULL` rejects `NULL`s on the right side's own column → per
`SimplifyOuterJoinType`'s indexing rule, `preserves_null_extended_rows[0]`
(left-side null-extension, produced by a `RIGHT`/`OUTER` join) flips false.
The right side (`preserves_null_extended_rows[1]`) is untouched, so the join
downgrades `OUTER` → `LEFT` — and since the plan printer for a `LEFT` join
doesn't print the literal word "FULL", the `<!REGEX>:.*FULL.*` check passes.
The symmetric case (`l.i IS NOT NULL`, checked against `range(2) l(i) ...
FULL OUTER JOIN range(3) r(i)`) downgrades `OUTER` → `RIGHT` instead, for the
same reason mirrored.

**`FULL OUTER` → `INNER` (`<REGEX>:.*INNER.*`):**

```sql
SELECT l.i, r.i FROM range(3) l(i) FULL OUTER JOIN range(3) r(i) ON l.i = r.i
WHERE l.i IS NOT NULL AND r.i IS NOT NULL ORDER BY 1;
```

Both sides are now `NULL`-rejected, so both `preserves_null_extended_rows`
flags go false, and `SimplifyOuterJoinType` downgrades straight to `INNER`.

**A `LEFT` join with an extra, constant-comparing condition stays `LEFT`
(`<REGEX>:.*LEFT.*`):**

```sql
SELECT l.i, r.i FROM range(3) l(i) LEFT JOIN range(3) r(i) ON l.i = r.i AND r.i = 1
ORDER BY 1;
```

There is no filter above this join at all — the extra `r.i = 1` is part of
the *join condition itself*, not a predicate the visitor's `HandleFilterExpression`
path ever sees, so `null_filtered_columns`/`null_required_columns` stay
empty for `r.i` and nothing downgrades. (This is also a nice illustration of
why the extra condition, sitting inside the join rather than above it, still
correctly produces `0 NULL / 1 1 / 2 NULL` at the result level — every left
row except `i=1` genuinely has no matching right row once `r.i=1` is also
required, which is ordinary `LEFT JOIN` semantics, nothing this pass needs to
touch.)

**A nested `LEFT JOIN` that becomes an `INNER JOIN`'s left child loses its
"LEFT"-ness (`<!REGEX>:.*LEFT.*`):**

```sql
SELECT t.li, x.j
FROM (SELECT l.i AS li, r.i AS ri FROM range(4) l(i) LEFT JOIN range(2) r(i) ON l.i = r.i) t
INNER JOIN range(2) x(j) ON t.ri = x.j
ORDER BY 1;
```

The *outer* `INNER JOIN`'s own condition, `t.ri = x.j`, is itself a
`NULL`-rejecting comparison on `t.ri` — which is a direct pass-through of the
inner `LEFT JOIN`'s `r.i`. `VisitInnerOrSemiJoin`
(`outer_join_simplification.cpp:287`) treats an `INNER`/`SEMI` join's own
conditions exactly like `HandleFilterExpression` treats a `WHERE` predicate:
`AddRequiredColumns` unconditionally, then `HandleExpression` on both sides
when `FiltersNulls` — so `t.ri`, and through `VisitProjection`'s pass-through
propagation, the inner join's `r.i`, lands in `null_filtered_columns` before
the inner `LEFT JOIN` node is ever reached. By the time the walk descends
into the inner join, it downgrades to `INNER` for exactly the same reason as
the two-sided `FULL OUTER` case above.

**The `ANTI`-join family (`<REGEX>:.*ANTI.*`), the pass's signature rewrite:**

```sql
SELECT l.i FROM range(3) l(i) LEFT JOIN range(2) r(i) ON l.i = r.i WHERE r.i IS NULL ORDER BY 1;
```

`r.i IS NULL`, and `r.i` is exactly the join's own equality-condition column
on the right → every condition in §11.1.1's `TryConvertLeftToAntiJoin` walk-
through is satisfied → `LEFT` becomes `ANTI`. The test suite repeats this
same shape through a `(SELECT ...) t` wrapper, through a `VALUES`-sourced
right side selecting a *different* output column (`r.v`, not the join key
itself — proving the rewrite reasons about the join-key binding specifically,
not merely "some column of the right side"), and through two invertible-cast
variants:

```sql
-- both of these still become ANTI:
... LEFT JOIN (VALUES (1), (2)) r(i) ON l.i = r.i        WHERE CAST(r.i AS BIGINT) IS NULL ...
... LEFT JOIN (VALUES (1), (2)) r(i) ON l.i = CAST(r.i AS BIGINT) WHERE CAST(r.i AS BIGINT) IS NULL ...
```

`INTEGER → BIGINT` is a lossless, invertible widening cast, so
`GetNullPreservingColumnBinding` unwraps it in both the join-condition copy
and the filter's copy, and the rewrite fires exactly as if the cast weren't
there.

**Where the cast breaks the rewrite (`<REGEX>:.*LEFT.*` — stays `LEFT`,
correctly declined):**

```sql
... LEFT JOIN (VALUES ('x'), ('2')) r(s) ON l.s = r.s WHERE TRY_CAST(r.s AS INTEGER) IS NULL ...
... LEFT JOIN (VALUES ('x'), ('2')) r(s) ON l.s = r.s WHERE      CAST(r.s AS INTEGER) IS NULL ...
```

Both stay `LEFT`, for the two separate reasons §11.1.3 laid out:
`IsTryCast` refuses the first outright; `VARCHAR → INTEGER` failing
`CastIsInvertible` refuses the second even though it isn't a `TRY_CAST`. The
test suite pairs the first with its actual correct-result check (both `'x'`
— the matched-but-cast-failed row — and `'3'` — the genuinely unmatched row
— show up under `TRY_CAST(...) IS NULL`, exactly the outcome an incorrect
`ANTI` rewrite would have silently broken by dropping the matched `'x'` row),
and pairs the second with a `statement error` check
(`Could not convert string 'x' to INT32`) proving the plain `CAST` version
of this query doesn't even have well-defined result rows to preserve — the
data genuinely makes it throw, which is exactly why a lossy, non-invertible
cast has no business being treated as `NULL`-preserving in the first place.

**Where the `NULL`-required column isn't the join key (`<REGEX>:.*LEFT.*`):**

```sql
SELECT l.i FROM (VALUES (1), (2), (3)) l(i)
LEFT JOIN (VALUES (1, NULL), (2, 5)) r(i, v) ON l.i = r.i
WHERE r.v IS NULL;
```

`r`'s row `(1, NULL)` is a *genuine match* on `i=1` whose `v` column happens
to be `NULL` in the underlying data — `r.v IS NULL` is true both for that
matched row and for the truly-unmatched `l.i=3` row. `r.v` is not part of the
join's equality condition, so `match_non_null_rhs` in
`TryConvertLeftToAntiJoin` never contains it, and the rewrite correctly
declines — converting to `ANTI` here would have wrongly dropped the `i=1`
row from the result (the test's correct-result twin confirms the actual
expected output is `1` and `3`, not just `3`).

**`IS NOT DISTINCT FROM` as the join condition, still convertible:**

```sql
SELECT l.i FROM (VALUES (NULL), (1), (2)) l(i)
LEFT JOIN (VALUES (NULL), (1)) r(i) ON l.i IS NOT DISTINCT FROM r.i
WHERE r.i IS NULL ORDER BY 1 NULLS FIRST;
```

Even though the join condition itself is the explicitly `NULL`-safe
`IS NOT DISTINCT FROM` (which `FiltersNulls` would say does *not* reject
`NULL`s — recall §11.1.2's `COMPARE_NOT_DISTINCT_FROM` exclusion) — the
*filter above the join*, `r.i IS NULL`, is a plain, ordinary `IS NULL`, whose
column resolves cleanly via `GetNullPreservingColumnBinding` (no cast
involved). `TryConvertLeftToAntiJoin`'s own loop over `join.conditions`
(building `match_non_null_rhs`) explicitly *skips* any condition that isn't
comparison-and-`FiltersNulls` — so the `IS NOT DISTINCT FROM` condition
contributes nothing to `match_non_null_rhs`, and this rewrite should, by the
letter of the code, *not* find `r.i` in `match_non_null_rhs`... and yet the
test's expectation is `<REGEX>:.*LEFT.*` (stays `LEFT`, not `ANTI`) — which
is exactly what the code above predicts: since the only join condition is
`IS NOT DISTINCT FROM` (not an ordinary `FiltersNulls`-qualifying
comparison), `match_non_null_rhs` ends up empty, the inner loop over
`right_bindings` never finds a binding satisfying both the
`null_required_columns` check *and* the `match_non_null_rhs` check, and the
rewrite declines. This test case is a precise, deliberate confirmation that
the rewrite's soundness argument — "a `NULL` on this column proves no
match happened, because the join literally matches on this column with
ordinary equality" — genuinely requires *ordinary*, `NULL`-rejecting equality
as the matching mechanism, and an explicitly `NULL`-tolerant condition like
`IS NOT DISTINCT FROM` disqualifies it even though an `IS NULL` filter above
the join is present and would otherwise look like a perfect trigger.

**A CTE-wrapped `GROUP BY` case, still converts:**

```sql
EXPLAIN WITH sales AS (
    SELECT y, l.i, sum(q) AS total_q
    FROM (VALUES (2000, 1, 10), (2000, 2, 20), (2001, 3, 30)) l(y, i, q)
    LEFT JOIN (VALUES (1), (3)) r(i) ON l.i = r.i
    WHERE r.i IS NULL
    GROUP BY y, l.i
)
SELECT * FROM sales WHERE y = 2000 ORDER BY i;
----
logical_opt	<REGEX>:.*ANTI.*
```

Proves the rewrite survives being nested inside a materialized/inlined CTE
definition and underneath a `GROUP BY` — `VisitAggregate`'s fresh sub-walk
(§11.1.4) still sees the `WHERE r.i IS NULL` filter directly beneath it
(the filter sits *between* the `GROUP BY` and the join in the unoptimized
tree, since `WHERE` binds tighter than `GROUP BY`), so the rewrite fires
exactly as in the simplest case.

**Correct-result checks.** The file's second half re-runs a subset of the
above queries as plain `query` blocks (no `EXPLAIN`), checking that the
*optimized* plan's actual output rows match hand-computed expectations —
this is the mechanism this book's front matter calls out as the test suite's
own self-verification: a rewrite this aggressive (changing a `LEFT JOIN`
into an `ANTI` join, which touches which columns even exist in the output)
would be caught here if the plan-shape checks above were passing for the
wrong reason.

**Real-query coverage: TPC-DS Q93 and Q78.** The file closes with two actual
TPC-DS queries (behind `require tpcds`), each proving the rewrite matters
beyond hand-written micro-cases:

```sql
-- TPC-DS Q93 (abbreviated)
... FROM store_sales LEFT OUTER JOIN store_returns
    ON (sr_item_sk = ss_item_sk AND sr_ticket_number = ss_ticket_number), reason
   WHERE sr_reason_sk = r_reason_sk AND r_reason_desc = 'reason 28' ...
----
logical_opt	<!REGEX>:.*(LEFT|RIGHT).*
```

`sr_reason_sk = r_reason_sk` is an ordinary equality filter on
`store_returns`'s own `sr_reason_sk` column, sitting above the `LEFT OUTER
JOIN` — `NULL`-rejecting, and `sr_reason_sk` survives (as a direct
pass-through) from `store_returns`'s own scan, so the `LEFT OUTER JOIN`
downgrades to `INNER`, and the plan text no longer contains either `LEFT` or
`RIGHT` at all.

```sql
-- TPC-DS Q78 (abbreviated)
... FROM web_sales LEFT JOIN web_returns
    ON wr_order_number = ws_order_number AND ws_item_sk = wr_item_sk
   JOIN date_dim ON ws_sold_date_sk = d_date_sk
   WHERE wr_order_number IS NULL ...
----
logical_opt	<REGEX>:.*ANTI.*
```

The canonical "rows from `web_sales` with no matching `web_returns` row"
idiom, at TPC-DS scale — the exact `LEFT JOIN ... WHERE key IS NULL` shape
`TryConvertLeftToAntiJoin` exists for, confirming the rewrite reaches real,
non-synthetic query shapes and not just constructed micro-cases.

### 11.1.6 A neighboring, easily-confused mechanism: filter pushdown through coalesced outer-join keys

§7.6.3 of the chapter overview already covers `FILTER_PULLUP`/`FILTER_PUSHDOWN`
(passes #5/#6) in general. One specific test file under that pair's
jurisdiction, `test/optimizer/pushdown/pushdown_filter_on_coalesced_equal_outer_join_keys.test`,
is worth a brief, deliberately non-duplicating mention here, because it sits
close enough to this file's subject matter to be confused with it: it proves
that `FULL OUTER JOIN t1 USING (id)` — where `id` is a *coalesced* output
column, `COALESCE(t1.id, t2.id)` under the hood — can have a filter like
`WHERE id >= 2` pushed all the way down into *both* underlying scans
(`t1 WHERE id >= 2` and `t2 WHERE id >= 2`, independently), **without
changing the join's type at all**. This is a `FILTER_PUSHDOWN`-family
rewrite, not an `OUTER_JOIN_SIMPLIFICATION` one: the join stays `FULL OUTER`
in every one of that test file's cases (nothing there is asking whether the
join could become `INNER`), and the mechanism is entirely about legally
distributing one filter on the *coalesced* output column into two filters on
the *pre-coalesce* input columns — a rewrite that has to reason correctly
about the fact that `COALESCE(t1.id, t2.id) IS NULL` only holds when *both*
inputs are `NULL` (both scans need the pushed-down `IS NULL` filter), whereas
`COALESCE(t1.id, t2.id) >= 2` only requires *whichever side actually has a
non-null value* to satisfy `>= 2` (still legally pushable to both sides
independently, since a `NULL` value never satisfies `>= 2` on either side
regardless). Two paired `query II nosort <label>` blocks per case are how the
test proves this pushdown is exactly semantically equivalent to writing the
filter by hand on each subquery — for example:

```sql
-- test/optimizer/pushdown/pushdown_filter_on_coalesced_equal_outer_join_keys.test:47
query II nosort single_join
EXPLAIN SELECT id
FROM (SELECT id FROM t1) FULL OUTER JOIN (SELECT id FROM t2) USING (id)
WHERE id >= 2
----

query II nosort single_join
EXPLAIN SELECT id
FROM (SELECT id FROM t1 WHERE id >= 2) FULL OUTER JOIN (SELECT id FROM t2 WHERE id >= 2) USING (id)
----
```

Two `query II nosort <label>` blocks that share the same label (here
`single_join`) are DuckDB's test format for asserting **byte-identical**
paired output with no sort applied and no expected-result section written
out explicitly for either block — the test runner checks that the *first*
block's actual output and the *second* block's actual output are identical
to each other, which is exactly what's needed to prove "the optimizer,
handed the natural query, produces the same `EXPLAIN` text as a human who
manually rewrote the query with the filter already pushed down by hand."
This convention recurs constantly across `test/optimizer/*.test` (at
`cb5673685153`, 22 files under `test/optimizer/` use `nosort`, on 230
lines) and is the mechanism behind essentially every "prove this rewrite
is exactly equivalent to writing it out by hand" check in the whole
optimizer test suite — worth knowing once, since it will reappear
throughout the rest of this chapter's sibling files without needing
re-explanation each time.

The reason this test file is worth flagging here rather than only in a
filter-pushdown-specific file: it is *exactly* the kind of "a filter above an
outer join interacts with the join's `NULL`-producing semantics" reasoning
this file's pass also does — the two mechanisms are neighbors in the same
conceptual territory (both reason about what a filter combined with an outer
join's `NULL`-production implies) but they are genuinely different code,
solving genuinely different problems: one distributes a filter across a
join's inputs without touching the join's type; the other changes the join's
type based on a filter that's already sitting above it. Reading both
together is the fastest way to stop conflating them.

### 11.1.7 Why this pass runs immediately before `JOIN_ORDER`

§7.4 of the chapter overview places `OUTER_JOIN_SIMPLIFICATION` at pass #16
and `JOIN_ORDER` at #17 — adjacent, in that order, with nothing structural
between them. Chapter 8 covers DPhyp's search in full; the piece of that
search worth pulling forward here is exactly the piece that explains *why*
this ordering is deliberate rather than incidental.

`JoinOrderConflictDetector::Build` (`src/optimizer/join_order/join_order_operator.cpp:241`)
decides, for every join operator DPhyp's query-graph construction encounters,
whether that operator's exact left/right relation-set pairing has to be
treated as a single, rigid, non-decomposable unit — a hyperedge DPhyp can
only ever "complete" all at once — or whether the individual base relations
underneath it are free to mix into any other subset the search explores:

```cpp
// src/optimizer/join_order/join_order_operator.cpp:247
op->must_apply_as_operator = op->type != JoinOrderOperatorType::CROSS_PRODUCT &&
                             (op->type != JoinOrderOperatorType::INNER || !op->conflict_rules.empty());
```

An `INNER` join with no conflict rules attached — the ordinary case — gets
`must_apply_as_operator = false`: its relations become individually
reorderable graph edges, free to combine with any other relation DPhyp's
dynamic-programming search considers, in whatever order the cost model
prefers. Anything that is not `CROSS_PRODUCT` and not a conflict-rule-free
`INNER` join — which includes every `LEFT`, `RIGHT`, `OUTER`, `SEMI`, `ANTI`,
and `MARK` join unconditionally — gets `must_apply_as_operator = true`: its
own `left_total_set`/`right_total_set` becomes a single fixed hyperedge that
has to be "completed" as one atomic step (`JoinOrderConflictDetector::IsCompletedBy`,
`join_order_operator.cpp:299`), and everything underneath moves through the
search as a rigid, already-decided subtree rather than a bag of freely
recombinable relations. Chapter 8's own §8.3 independently arrives at the
same fact from the query-graph-construction side, describing exactly this
class of operator as living on "non-reorderable operators (an
`OUTER`/`SEMI`/`ANTI` join, or anything `JoinOrderConflictDetector::MustApplyAsOperator`
decides imposes an ordering constraint DPhyp has to respect)."

Put the two facts together and the pipeline ordering stops being a
coincidence: every `LEFT`/`RIGHT`/`OUTER` join that `OUTER_JOIN_SIMPLIFICATION`
manages to downgrade to `INNER` *before* `JOIN_ORDER`'s query-graph
construction runs is a join that gets to shed a `must_apply_as_operator`
constraint it would otherwise have carried into the search unconditionally —
its relations become free graph edges DPhyp can mix into any join order the
cost model prefers, rather than a pinned, atomic subtree the search has no
choice but to build exactly as written. Running the simplification one pass
earlier is what makes that relaxation available to the very next pass in the
sequence; running it any later (recall §7.2's architecture: there is no
second chance, no fixpoint, no re-triggering) would mean `JOIN_ORDER` had
already committed to treating a join as a rigid `LEFT`-type hyperedge before
this pass ever got the chance to prove it didn't need to be one. An `ANTI`
join conversion doesn't get the same benefit — `ANTI` is exactly as
non-reorderable to DPhyp as `LEFT` was (both get `must_apply_as_operator =
true` unconditionally) — but the plain `INNER`-downgrade family this pass
also performs is a direct, mechanical widening of next-pass's search space,
and its position in the pipeline is what makes that widening actually land
in time to matter.

---

## 11.2 `JOIN_ELIMINATION`

**Purpose, in one sentence:** if nothing anywhere in the rest of the plan
reads a column that could only have come from one side of a join, and that
side is provably guaranteed to contribute *at most one* matching row per row
on the other side (so the join can't duplicate anything and — for a
`LEFT`/`RIGHT`/`SINGLE` join specifically — can't filter anything either),
the join node is deleted outright and replaced by the surviving side alone.

### 11.2.1 A genuinely surprising finding worth stating before anything else

The task that produced this chapter's outline for this pass assumed — very
reasonably, given how the equivalent optimization is usually described in
database-textbook terms — that `JOIN_ELIMINATION` proves its uniqueness
guarantee from **catalog-declared primary-key, foreign-key, or `UNIQUE`
constraint metadata**: "a join against a dimension table on its declared
primary key can't duplicate the fact table's rows, so if nothing selects the
dimension table's columns, drop the join." Reading `join_elimination.cpp`
directly turns up something different, and it's worth stating precisely
rather than assuming the textbook story applies, per this book's own stated
policy on checking claims against the actual source:

**This checkout's `JOIN_ELIMINATION` pass does not consult catalog
constraint metadata at all.** Its own source file says exactly this, in a
comment that doubles as a literal `TODO`:

```cpp
// src/optimizer/join_elimination.cpp:278
if (pipe_info.distinct_groups.empty()) {
	return std::move(pipe_info.root);
}
// 1. TODO: guarantee by primary/foreign key
```

> **Note:** the `TODO` is at `join_elimination.cpp:281`. Nothing else under
> `src/optimizer/` reads primary-key or `UNIQUE` constraints to remove a join
> either.

and the class's own doc comment in the header spells out what actually *does*
prove uniqueness instead, and lists the PK/FK route only as a name inside a
worked-example parenthetical, not as something implemented:

```cpp
// src/include/duckdb/optimizer/join_elimination.hpp:59
// with specific condition we can eliminate a (left/right, semi, inner) join.
// exemplify left/right join eliminaion condition:
// 1. output can only have outer table columns
// 2. join result cannot filter by inner table columns(ex. in where clause/ having clause ...)
// 3. must ensure each outer row can match at most one inner table row, such as:
//  1) inner table join condition is unique(ex. 1. join conditions have inner table's primary key 2. inner table
//  join condition columns contains a whole distinct group) 2) join result columns contains a whole distinct group
```

Git history confirms this isn't a regression or a stripped-down feature —
`git log --diff-filter=A` shows `src/optimizer/join_elimination.cpp` was
**added** on 2025-05-26 (commit `e4aa62d764`, message *"working"*) with no
earlier version of a differently-named file doing PK/FK-based elimination
anywhere in this repository's history. `OptimizerType::JOIN_ELIMINATION` is a
genuinely new pass, built from scratch over more than a year of commits,
entirely around **`DISTINCT`
and `GROUP BY`-provable uniqueness of the query's own written structure** —
not catalog metadata. The `// TODO: guarantee by primary/foreign key` comment
has been present, unresolved, since at least the pass's early shape and is
still there at `cb5673685153`. **A join
against a table with a real, enforced `PRIMARY KEY` or `UNIQUE` constraint on the join key, where nothing
downstream reads that table's columns, will *not* be eliminated by this pass
today unless the query text itself also carries an explicit `DISTINCT` or
`GROUP BY` that the pass's own analysis can trace back to that side.** This
is the honest, source-grounded answer to the task's own question about
"what happens if the constraint isn't actually declared" — the more precise
finding is that it currently doesn't matter whether the constraint is
declared or not, because nothing in this pass looks at the catalog either
way. §11.2.5 revisits this with a real query that a PK-aware optimizer
*should* be able to eliminate but this checkout's `JOIN_ELIMINATION` cannot.

### 11.2.2 The bookkeeping: `PipelineInfo`, `distinct_groups`, and how a `DISTINCT`/`GROUP BY` gets traced back to a table

`JoinElimination` is, like `OuterJoinSimplification`, a tree-walking visitor,
but its walk is structured very differently — it builds a **mirrored tree of
`JoinElimination` sub-objects**, one recursive instance per join and per
multi-child operator, each carrying its own `PipelineInfo`:

```cpp
// src/include/duckdb/optimizer/join_elimination.hpp:31
class PipelineInfo {
public:
	unique_ptr<LogicalOperator> root = nullptr;
	unordered_set<TableIndex> ref_table_ids;
	unordered_map<TableIndex, column_binding_set_t> distinct_groups;

	// pushdown filter condition(ex in table scan operator),
	// if have outer table columns then cannot elimination
	bool has_filter = false;

	optional_ptr<LogicalOperator> join_parent = nullptr;
	idx_t join_index = 0;

public:
	PipelineInfo CreateChild() {
		auto result = PipelineInfo();
		result.ref_table_ids = ref_table_ids;
		result.distinct_groups = distinct_groups;
		return result;
	}
};
```

`distinct_groups` is the heart of the whole pass: a map from a `TableIndex`
to the exact set of that table's own column bindings which some `DISTINCT`
or `GROUP BY` operator, somewhere in the subtree rooted below the current
point, has already proven are jointly unique. Two operator types populate a
fresh entry directly, in `OptimizeChildren`
(`src/optimizer/join_elimination.cpp:22`):

```cpp
// src/optimizer/join_elimination.cpp:43 (LOGICAL_DISTINCT case, abbreviated)
case LogicalOperatorType::LOGICAL_DISTINCT: {
	auto &distinct = op.Cast<LogicalDistinct>();
	if (distinct.distinct_type != DistinctType::DISTINCT) break;   // not plain DISTINCT (e.g. DISTINCT ON) -> skip
	// every distinct_targets[i] must itself be a plain BOUND_COLUMN_REF,
	// and all of them must share the same table_index, or this is skipped
	pipe_info.distinct_groups[table_idx] = std::move(distinct_group);
	break;
}
// src/optimizer/join_elimination.cpp:68 (LOGICAL_AGGREGATE_AND_GROUP_BY case, abbreviated)
case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY: {
	auto &aggr = op.Cast<LogicalAggregate>();
	if (aggr.grouping_sets.size() > 1) break;                     // ROLLUP/CUBE/multiple grouping sets -> skip
	// registers one binding per group-by column, keyed by the aggregate's OWN group_index
	pipe_info.distinct_groups[table_idx] = std::move(distinct_group);
	break;
}
```

Two conditions worth calling out because they're real, source-confirmed
limits on what this pass can even *see*: `DISTINCT ON (...)` is explicitly
excluded (only plain, whole-row `DISTINCT` registers a group), and a
`GROUP BY` with more than one grouping set (`ROLLUP`/`CUBE`/explicit multiple
`GROUPING SETS`) is excluded entirely too — the pass makes no attempt to
reason about which of several grouping sets might apply to which downstream
consumer.

A `LOGICAL_PROJECTION` case (`join_elimination.cpp:84`) then propagates a
registered group **forward** through the tree, but only if the projection's
relevant output columns are *themselves* plain, unrenamed-in-a-way-that-
breaks-tracing column references back to the group's own binding set — any
non-column-reference expression in the projection for one of the tracked
columns aborts the propagation for that group (`could_add = false`). This is
what lets `(SELECT DISTINCT ida AS ida FROM a)` still register as a group
keyed to the *subquery's own* projection table index even though the column
got renamed — but it is also exactly why `(SELECT * FROM a GROUP BY all)`,
which groups on *two* columns (`ida` and `valuea`), only registers a
two-column group, not a one-column one, no matter how the query later
references just one of those columns.

The other side of the bookkeeping, `ref_table_ids`, is populated by a single
`VisitReplace` override that fires on every `BoundColumnRefExpression` the
generic `LogicalOperatorVisitor` machinery walks past during
`VisitOperatorExpressions(op)`:

```cpp
// src/optimizer/join_elimination.cpp:333
unique_ptr<Expression> JoinElimination::VisitReplace(BoundColumnRefExpression &expr, unique_ptr<Expression> *expr_ptr) {
	pipe_info.ref_table_ids.insert(expr.Binding().table_index);
	return nullptr;
}
```

Every table index that appears *anywhere* in an expression the walk has
passed through — a `WHERE`, a `SELECT`, a `GROUP BY` key, an `ORDER BY`, a
join condition on some *other* join — ends up in `ref_table_ids`. This is the
blunt instrument behind requirement 1 in the header comment ("output can only
have outer table columns"): if the side being considered for elimination
appears in `ref_table_ids` at all, elimination is refused outright
(`join_elimination.cpp:263`), no matter how narrow or broad the actual
reference turns out to be.

Note which way these sets flow. A sub-visitor is made by `CreateChildren`,
which calls `PipelineInfo::CreateChild` (the excerpt above). That copies the
parent's `ref_table_ids` and `distinct_groups` into the child. So both sets
flow *down* the tree. A join deep in the plan sees every column read and
every `DISTINCT` group found above it.

There is one place where a join itself reads columns: its own conditions.
The join branch of `OptimizeChildren` registers those
reads in both child sub-visitors before it goes down:

```cpp
// src/optimizer/join_elimination.cpp:23
if (op.type == LogicalOperatorType::LOGICAL_COMPARISON_JOIN) {
	if (!parent) {
		return;
	}
	D_ASSERT(!pipe_info.join_parent);
	pipe_info.join_parent = parent;
	pipe_info.join_index = idx;
	left_child = CreateChildren();
	right_child = CreateChildren();
	// the joins below must keep the columns this join's conditions read
	left_child->VisitOperatorExpressions(op);
	right_child->VisitOperatorExpressions(op);
	left_child->OptimizeInternal(std::move(op.children[0]));
	right_child->OptimizeInternal(std::move(op.children[1]));
	return;
}
```

> **Why the two `VisitOperatorExpressions(op)` lines matter.** Without them,
> a join's own conditions would never reach `ref_table_ids`. Then a join
> lower in the tree could be removed even though a join *above* it read its
> output in an `ON` clause (DuckDB issue #27606). The issue's query has
> `LEFT JOIN tc ON tc.c_custkey = (SELECT o.o_custkey FROM ... LIMIT 1)`.
> The scalar subquery becomes a `SINGLE` join a few levels down. Nothing in
> the `SELECT` list reads it, only the `ON` clause above. If the pass removed
> that `SINGLE` join, the subquery's column would disappear, and the query
> would fail with an internal error. The regression test is
> `test/optimizer/joins/join_elimination_condition_reference.test`.

### 11.2.3 `TryEliminateJoin`: the three ways a join proves it's safe to remove

`TryEliminateJoin` (`join_elimination.cpp:213`) runs bottom-up, once the
top-down `OptimizeChildren` pass has finished building the whole mirrored
tree of sub-visitors and populating each one's `pipe_info`. For a candidate
join, it first narrows down which join types are even eligible at all:

```cpp
// src/optimizer/join_elimination.cpp:236
bool is_output_unique = false;
idx_t inner_idx = 1;
idx_t outer_idx = 0;
switch (join.join_type) {
case JoinType::LEFT:
	break;
case JoinType::SINGLE: {
	is_output_unique = true;
	break;
case JoinType::RIGHT:
	inner_idx = 0;
	outer_idx = 1;
	break;
}
default:
	return std::move(pipe_info.root);
}
```

Only `LEFT`, `RIGHT`, and `SINGLE` joins are ever candidates — `INNER`,
`OUTER`, `SEMI`, `ANTI`, `MARK` all hit the `default:` and are refused
unconditionally (`INNER` and `OUTER` can filter or duplicate the *outer*
side too, which this pass's whole argument doesn't reason about; `SEMI`/
`ANTI`/`MARK` don't preserve the outer side's row shape in the way this
pass's replacement — "just keep the outer child" — assumes). **`SINGLE`
joins get `is_output_unique = true` unconditionally, for free** — this join
type is DuckDB's own internal representation of "a correlated scalar
subquery's join, decorrelated," and its defining guarantee (per the enum
comment: *"returns at most one join partner per entry on the LEFT side"*) is
already exactly the uniqueness property this whole pass exists to prove, so
no `distinct_groups` lookup is even needed for it.

For `LEFT`/`RIGHT`, three more checks gate elimination before uniqueness is
even considered: `inner_child->pipe_info.has_filter` (a table filter
somewhere inside the to-be-eliminated side means that side could be silently
excluding rows in a way the outer side's row *count* depends on — refused),
`join.filter_pushdown` (a dynamic join-filter-pushdown descriptor already
attached to this join, from pass #42's later machinery having been primed
earlier in some intervening re-run context — refused conservatively), and
the `ref_table_ids` check from §11.2.2 (refused if the inner side's own
table index appears anywhere in `ref_table_ids`).

Only once all of those pass does the actual uniqueness proof run, and it is
exactly the two-rule structure the header comment names:

```cpp
// src/optimizer/join_elimination.cpp:283 (rule 2, abbreviated)
if (!is_output_unique) {
	is_output_unique = true;
	vector<ColumnBinding> col_bindings;
	for (auto &condition : join.conditions) {
		if (!condition.IsComparison() || condition.GetComparisonType() != ExpressionType::COMPARE_EQUAL ||
		    condition.GetLHS().GetExpressionType() != ExpressionType::BOUND_COLUMN_REF ||
		    condition.GetRHS().GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
			is_output_unique = false;
			break;
		}
		auto inner_binding = /* the inner side's column of this equality condition */;
		col_bindings.push_back(inner_binding);
	}
	if (is_output_unique && !ContainDistinctGroup(col_bindings)) {
		is_output_unique = false;
	}
}
if (!is_output_unique) {
	// rule 3
	auto outer_bindings = join.children[outer_idx]->GetColumnBindings();
	if (ContainDistinctGroup(outer_bindings)) {
		is_output_unique = true;
	}
}
```

**Rule 2** — every one of the join's conditions must be a plain
column-equals-column equality (any `OR`, any constant comparison, any
non-equality operator anywhere in the condition list aborts the *entire*
rule, not just that one conjunct — `is_output_unique = false; break;` exits
the loop the moment one non-conforming condition is seen). If every
condition qualifies, the *set* of the inner side's own columns referenced
across all of them (deduplicated, since the same inner column can appear in
more than one condition) has to exactly cover some registered
`distinct_groups` entry keyed to that side's table index —
`ContainDistinctGroup` (`join_elimination.cpp:316`) checks this by
intersecting the supplied bindings against the stored group and comparing
sizes, so a *partial* overlap (join key covers only some of a
multi-column `DISTINCT`'s columns) correctly fails.

**Rule 3** — a fallback for exactly the shape §11.2.4 walks through in
detail: if the join's own condition columns don't prove anything, check
whether the **outer (preserved) side's own bindings** already fully cover
some registered `distinct_groups` entry keyed to *that* side's table index.
This isn't a second, independent way to prove "the inner side can't
duplicate anything" — it's a different, equally valid argument that reaches
the same conclusion by a different route: if a `DISTINCT`/`GROUP BY`
*already sitting directly above this join* is going to collapse any
duplicate rows the join might produce right back down to one row per
distinct outer-side combination anyway, then whatever the join does
underneath that `DISTINCT` is unobservable, and the join can be deleted
without the `DISTINCT` operator itself needing to move or change at all.

### 11.2.4 Every case in `test/optimizer/join_elimination.test`

The file sets up three small tables — `a(ida, valuea)`, `b(idb, valueb)`,
`c(idc, valuec)` — each seeded with duplicate keys and `NULL`s specifically
so that a naive "just check if the values look unique" heuristic would be
insufficient; only a structural `DISTINCT`/`GROUP BY` proof is meant to pass.
Every check is a `physical_plan	<REGEX>:.*JOIN.*` / `<!REGEX>:.*JOIN.*` pair —
the *physical* plan (post-Chapter-9) either still contains a join operator or
it doesn't:

**Eliminated (`<!REGEX>` — no `JOIN` in the physical plan):**

```sql
explain select distinct b.* from b left join a on a.ida=b.idb;
```
`SELECT DISTINCT b.*` registers a `distinct_groups` entry keyed to `b`'s
table index covering `{idb, valueb}` — every column `b` has. `a`'s columns
never appear in `ref_table_ids` (only `b.*` is selected). Rule 3 applies
directly: the outer side (`b`)'s own bindings *are* the registered group.
Eliminated.

```sql
explain select distinct b.* from b left join a on a.ida=b.idb where b.valueb=1;
```
Same reasoning — the extra `WHERE b.valueb=1` only adds `b`'s own table
index to `ref_table_ids` (fine, `b` is the outer/preserved side, this check
only blocks the *inner* side), and doesn't touch `a` at all. Still
eliminated.

```sql
explain select b.* from b left join (select distinct ida as ida from a) as a1 on a1.ida=b.idb;
```
Here the `DISTINCT` sits on `a`'s *own* side, and covers exactly `{ida}`.
The join condition `a1.ida = b.idb` references exactly `a1.ida` on the inner
side. Rule 2 applies: the (deduplicated) inner-side condition columns,
`{ida}`, exactly cover the registered group. Eliminated.

```sql
explain select b.* from b left join (select a.ida from a group by ida) as a1 on b.idb=a1.ida;
```
Same shape as the previous case with `GROUP BY` instead of `DISTINCT` —
same registered `{ida}` group, same rule-2 match. Eliminated.

```sql
explain select  b.* from b left join (select distinct ida as ida from a) as a1 on a1.ida=b.idb and a1.ida = b.valueb;
```
Two equality conditions, but both reference the *same* inner-side column
(`a1.ida`, compared against two different outer-side columns) — the
deduplicated inner-side column set is still just `{ida}`, which still
exactly covers the registered `{ida}` group. Eliminated.

```sql
explain select  b.* from b left join (select distinct ida as ida, valuea as valuea from a) as a1
  on a1.ida=b.idb and a1.ida = b.valueb and a1.valuea = b.idb;
```
Now the registered group is `{ida, valuea}` (two-column `DISTINCT`), and the
three conditions between them reference both inner-side columns
(`a1.ida` twice, `a1.valuea` once) — deduplicated inner-side set is
`{ida, valuea}`, exactly matching. Eliminated.

```sql
explain
  select b.* from b left join (select distinct a.ida from a) as a on a.ida=b.idb union all
  select b.*  from b left join (select distinct a.ida from a) as a on a.ida=b.idb;
```
Elimination fires independently on both sides of a `UNION ALL` (no special
handling needed — each branch is walked as its own subtree, and `UNION`'s
multi-child structure is handled by the generic `children`/`children_root`
branching in `OptimizeChildren`, `join_elimination.cpp:152`).

**Not eliminated (`<REGEX>` — `JOIN` still present in the physical plan):**

```sql
explain select distinct b.* from b left join a on a.ida=b.idb where a.valuea=1;
```
`a.valuea=1` puts `a`'s table index into `ref_table_ids` — the inner side is
now referenced by a predicate, so `join_elimination.cpp:263`'s check refuses
elimination before uniqueness is even considered, regardless of `b.*`'s
`DISTINCT`.

```sql
explain select  b.* from b left join (select * from a group by all) as a1 on b.idb=a1.ida;
```
`GROUP BY ALL` groups on *every* non-aggregate column of `a`, registering a
**two-column** group `{ida, valuea}`. The join condition only equates on
`ida` — a strict subset of the registered group — so rule 2's
`ContainDistinctGroup` size check fails (`1 != 2`): grouping on two columns
only proves uniqueness of the *pair*, not of `ida` alone, so a single `ida`
value could still legitimately appear on multiple output rows with
different `valuea` values, each of which would duplicate the matching `b`
row. Correctly refused.

```sql
explain select b.idb,sum(b.valueb) from b left join a on b.idb=a.ida group by b.idb;
explain select b.idb,sum(a.valuea) from b left join a on b.idb=a.ida group by b.idb;
```
Neither is eliminated, and it is worth being precise about *why*, since it's
easy to assume the first one (which never even reads an `a` column) should
qualify. The `GROUP BY b.idb` sits *above* the join, and registers its own
group keyed to the **aggregate operator's own freshly-minted table index** —
a completely different namespace from either `a`'s or `b`'s original scan
table index. Rule 2 looks for a group keyed to `a`'s table index (none
exists); rule 3 looks for a group keyed to `b`'s table index (also none —
the aggregate's synthetic table index is not `b`'s). A `GROUP BY` that
merely *sits above* a join, grouping on the *preserved* side's key, is not
the same evidence as a `DISTINCT`/`GROUP BY` that decorates one specific
*side*'s own subtree directly — this pass only registers and consults groups
by the table index of the operator that actually established them, and an
aggregate above the whole join establishes a group about its own output
rows, not about either input side. (The second query additionally puts `a`'s
table index into `ref_table_ids` via `sum(a.valuea)`, which alone would have
refused it regardless.)

```sql
explain select  b.* from b left join (select distinct ida as ida from a) as a1 on a1.ida=b.idb and a1.ida = 1;
```
The registered group is `{ida}`, but one of the two join conditions,
`a1.ida = 1`, has a constant on one side, not a column reference — rule 2's
per-condition check (`GetRHS().GetExpressionType() != BOUND_COLUMN_REF`)
aborts the *entire* rule the moment it sees this, even though the other
condition (`a1.ida = b.idb`) alone would have satisfied it. This is a real,
source-confirmed conservative gap: logically, adding an *extra* equality
condition can only narrow the matching rows further, never introduce new
duplicates — a human could reason that this join is at least as safe to
eliminate as the version without the constant condition — but
`JoinElimination`'s rule 2 doesn't attempt that reasoning; any
non-column-pair condition anywhere in the list disqualifies the whole rule
for that join, full stop.

```sql
explain select  b.* from b left join (select distinct ida as ida from a) as a1 on a1.ida=b.idb or a1.ida = 1;
```
An `OR`, not an `AND` — this doesn't even survive as a `COMPARE_EQUAL`
`JoinCondition` the same way, so rule 2's `condition.IsComparison()` check
fails outright. Refused.

```sql
explain select b.*, a.* from b left join a on a.ida=b.idb;
explain select b.* from b left join a on a.ida=b.idb+1;
explain select tan(b.idb) from b left join a on a.ida=b.idb;
explain select b.valueb from b left join a on a.ida=b.idb;
explain select b.*, tan(a.ida) from b left join a on a.ida=b.idb;
```
None of these has any `DISTINCT`/`GROUP BY` anywhere, so `pipe_info.distinct_groups`
is empty and every one refuses at the very first check
(`join_elimination.cpp:278`: `if (pipe_info.distinct_groups.empty()) return
std::move(pipe_info.root);`) regardless of what else is or isn't selected —
this is the cleanest illustration that **an ordinary `LEFT JOIN` to a table
whose columns aren't read is *not* eliminated by this pass on its own**; a
witnessing `DISTINCT`/`GROUP BY` is a hard prerequisite, not merely a
convenient one.

**Two nested-join cases that prove rule 3's real scope, precisely:**

```sql
-- "this should eliminate inner join but keep outer one"
explain SELECT c.*
  FROM c LEFT JOIN (SELECT DISTINCT b.* FROM b LEFT JOIN a ON a.ida = b.idb) AS ab
  ON c.idc = ab.idb;
----
physical_plan	<REGEX>:.*JOIN.*
```

Two nested `LEFT JOIN`s: `c LEFT JOIN (b LEFT JOIN a)`. `SELECT DISTINCT
b.*` registers a group `{idb, valueb}` keyed to `b`'s own table index, and
that registration happens *before* the walk descends past the `b LEFT JOIN
a` join, so by the time the inner join (`a` is its inner side) is evaluated,
rule 3 applies directly: the outer/preserved side of *that* join is `b`
itself, whose bindings exactly cover the registered `{idb, valueb}` group —
**the inner `LEFT JOIN a` is eliminated**, leaving `DISTINCT b.*` sitting
directly over `b`'s bare scan. The *outer* join (`c LEFT JOIN ab`) is a
separate question: its own condition only equates on `idb`, a strict subset
of the `{idb, valueb}` group carried by `ab`'s subtree, so rule 2 fails for
it, and rule 3 (checking `c`'s own bindings against any group keyed to `c`)
finds nothing either — the outer join stays. The test's physical plan check
(`<REGEX>:.*JOIN.*` — *some* join remains) is checking the *outer* survivor,
not that nothing happened; the test's own comment (*"this should eliminate
inner join but keep outer one"*) states the intended, and actually observed,
asymmetric outcome directly.

```sql
-- "this should eliminate all join"
explain SELECT c.*
  FROM c LEFT JOIN (SELECT DISTINCT b.* FROM b LEFT JOIN a ON a.ida = b.idb) AS ab
  ON c.idc = ab.idb and c.valuec = ab.valueb;
----
physical_plan	<!REGEX>:.*JOIN.*
```

Identical structure, except the outer join's condition now equates on
*both* `idb` and `valueb` — exactly covering the full `{idb, valueb}` group
`ab`'s subtree carries. Rule 2 now succeeds for the outer join too. Both
joins are eliminated, leaving just `c`'s bare scan under the (still-present)
`DISTINCT` operator wrapping `b`'s bare scan somewhere in what was
previously `ab`'s subtree — but since `ab`'s own join was already eliminated
first (bottom-up evaluation order — recall `TryEliminateJoin` recurses into
`children[0]`/`children[1]` before evaluating the *current* join), by the
time the outer join is evaluated, `ab`'s subtree is already just
`DISTINCT b.* FROM b`, and the outer join disappears too, leaving `c`'s scan
directly.

**The `UNNEST` interaction (issue #25213), proving the pass reasons about
row-multiplying operators correctly:**

```sql
-- one row emitted per element of unnest([1,2,3]) -> 3x row multiplication
select count(*) from probe_keys
left join (select k, unnest([1,2,3]) e from (select distinct k from dup_keys) s) t
on t.k=probe_keys.idb;
----
20
```

`dup_keys` has 4 rows with a duplicate key (`1,1,2,3`); `SELECT DISTINCT k`
registers a group `{k}` covering exactly 3 distinct values. If this group
were carried *past* the `UNNEST` unmodified, rule 2/3 would (wrongly)
conclude the whole subquery `t` is unique per `k`, when in fact `UNNEST`
triples every row — `t` actually has 3 rows per distinct `k` value, not 1.
`OptimizeChildren`'s single-child branch (`join_elimination.cpp:141`)
explicitly guards against exactly this:

```cpp
// src/optimizer/join_elimination.cpp:141
// UNNEST emits multiple rows per input row, so a distinct group found below it does not hold above it
const auto invalidates_distinct = op.type == LogicalOperatorType::LOGICAL_UNNEST;
unordered_map<TableIndex, column_binding_set_t> distinct_groups_above;
if (invalidates_distinct) {
	distinct_groups_above = pipe_info.distinct_groups;
}
OptimizeChildren(*op.children[0], op, idx);
if (invalidates_distinct) {
	pipe_info.distinct_groups = std::move(distinct_groups_above);
}
```

Any `distinct_groups` entries that existed *before* descending into an
`UNNEST`'s child are saved and restored *after* — meaning whatever new
groups get registered strictly *below* the `UNNEST` (there are none in this
query, but there could be) are discarded once the walk climbs back up past
it, and any group that existed above the `UNNEST` before the descent is
preserved unchanged rather than silently absorbing whatever the child
subtree did. The commit that added this (`bd30bfb4b4`, *"Do not carry a
distinct group across UNNEST in join_elimination"*, 2026-09-09, fixing
`duckdb/duckdb#25213`) is direct evidence that this pass, barely more than a
year into its existence, was still having its correctness boundary actively
discovered and patched. Later fixes to this file are covered in §11.2.2 and
at the end of this section. The test file directly probes
around this fix from every angle it can construct: the same query with
`GROUP BY` instead of `DISTINCT` (same result, `20`), the same shape with a
*nested* `UNNEST` (`recursive := true`, still `20`), and — the case that
proves the guard is precise rather than a blunt "never trust anything near
an `UNNEST`" — a `DISTINCT` applied *above* an `UNNEST` still counts:

```sql
-- a DISTINCT above the unnest still makes the inner side unique
select count(*) from probe_keys
left join (select distinct k from (select k, unnest([1,2,3]) e from dup_keys) u) t
on t.k=probe_keys.idb;
----
8
```
```
physical_plan	<!REGEX>:.*JOIN.*   -- eliminated
```

Here the `UNNEST`'s row-tripling happens *first*, and only *then* does
`SELECT DISTINCT k` collapse the result back down to one row per key — the
`invalidates_distinct` guard only discards groups established *below* the
point of descent into `UNNEST`'s child, it does not (and should not) touch a
group registered by an operator sitting *above* the `UNNEST` in the tree,
which is exactly this case.

**`LIMIT`, `TOP_N`, `SAMPLE` and `WINDOW` (issue #27155):** a `DISTINCT`
group does not pass down through these four operators. The pass clears all
groups when its walk reaches one of them.

```cpp
// src/optimizer/join_elimination.cpp:121
case LogicalOperatorType::LOGICAL_LIMIT:
case LogicalOperatorType::LOGICAL_TOP_N:
case LogicalOperatorType::LOGICAL_SAMPLE:
case LogicalOperatorType::LOGICAL_WINDOW: {
	// these pick rows by cardinality, or compute values that depend on row multiplicity (window), so a
	// distinct group above them does not hold below them
	pipe_info.distinct_groups.clear();
	break;
}
```

Why is this needed? Rule 3 (§11.2.3) says: "a `DISTINCT` above the join will
remove any duplicate rows the join makes, so the duplicates do not matter".
That is only true if nothing between the `DISTINCT` and the join *counts*
rows. A `LIMIT` does count rows. If the join makes an extra copy of a row,
the `LIMIT` may keep that copy and drop a different row. Then the `DISTINCT`
sees a different set of values. `SAMPLE` and `TOP_N` also pick rows by count.
A window function such as `row_number()` gives different numbers when there
are extra copies.

Cases in `test/optimizer/join_elimination.test` show this with two
tables. `p(id, g)` has the rows
`(1,x),(2,y),(3,x),(4,y),(5,x),(6,y),(7,z)`. `k(a, b)` has two rows with
`a = 1` and two rows with `a = 6`, so the join `p LEFT JOIN k ON p.id = k.a`
makes two copies of `p` rows 1 and 6:

```sql
select distinct pg from (select p.id as pid, p.g as pg from p left join k on p.id = k.a
                         order by p.g desc, p.id limit 2 offset 3) order by pg;
----
y
```

Sorted by `g` descending and then `id`, the joined rows start
`(7,z), (2,y), (4,y), (6,y), (6,y), (1,x), (1,x), ...`. `OFFSET 3 LIMIT 2`
keeps the two copies of `(6,y)`, so the answer is just `y`. If the join were
removed, the list would be `(7,z), (2,y), (4,y), (6,y), (1,x), ...`, and the
slice would be `(6,y), (1,x)`, giving the wrong answer `{x, y}`. The
`EXPLAIN` checks for this query, for a plain `LIMIT 5`, for
`USING SAMPLE 5 ROWS` and for a `row_number() ... WHERE rn <= 2` filter all
expect the join to stay.

The guard is still precise. Two cases in the same test are still
eliminated:

```sql
-- the DISTINCT sits directly above the join, below the LIMIT
explain select distinct pg from (select p.g as pg from p left join k on p.id = k.a) order by pg limit 2;
----
physical_plan	<!REGEX>:.*JOIN.*

-- the inner side is DISTINCT on the join key, so the join never makes copies
explain select distinct pg from (select p.g as pg from p left join (select distinct a from k) s on p.id = s.a
                                 order by pg desc limit 3) order by pg;
----
physical_plan	<!REGEX>:.*JOIN.*
```

In the first query, the walk meets the `LIMIT` *before* it meets the
`DISTINCT`, so clearing the groups there removes nothing useful. In the
second query, the group that proves uniqueness (`{a}` on `s`) is found
*below* the `LIMIT`, so rule 2 still applies.

**"We cannot use distinct statistics from outer child" — a comment worth
reading literally:**

```sql
statement ok
select a.ida from (select ida from a group by ida) as a left join b on ida < idb;
----
physical_plan	<REGEX>:.*JOIN.*   -- NOT eliminated
```

Here `a` (with its registered `{ida}` group from `GROUP BY ida`) is the
**outer/preserved side**, and `b` is the inner (to-be-eliminated) side. The
join condition, `ida < idb`, is not an equality at all — rule 2 requires
`COMPARE_EQUAL` specifically, so it's refused immediately regardless of
anything else. The test's own comment flags the deeper point directly: even
setting the inequality issue aside, `a`'s own `DISTINCT`/`GROUP BY`-provable
uniqueness says nothing whatsoever about how many rows of `b` each `a` row
could match — that's a question only about the *inner* side's uniqueness,
and no amount of "the outer side happens to be unique too" changes it (this
is a different question from rule 3, which uses the outer side's uniqueness
for a different, narrower argument — see §11.2.3's explanation of what rule
3 actually proves). A mirrored `RIGHT JOIN` version confirms the same
refusal from the other side.

### 11.2.5 Edge cases and limitations, stated plainly

- **No catalog awareness of any kind.** §11.2.1 already covers this in full:
  a declared `PRIMARY KEY`/`UNIQUE`/`FOREIGN KEY` constraint on the join key
  is not consulted anywhere in this file. A query like
  `SELECT b.* FROM fact b LEFT JOIN dim d ON d.id = b.fk` where `dim.id` is a
  real, enforced `PRIMARY KEY` and nothing selects any `dim` column, is
  logically just as safe to eliminate as the `DISTINCT`-witnessed cases
  above — DuckDB's own catalog already knows `dim.id` is unique — but this
  pass will not eliminate it today, because nothing in the query text itself
  carries a `DISTINCT`/`GROUP BY` the pass's analysis can trace. The `TODO`
  comment (§11.2.1) is the honest acknowledgment that this is a known,
  unclaimed opportunity, not a deliberate design boundary.
- **Do not confuse this with `NextUniqueLeftJoin`.** DuckDB *does* have a
  real, catalog-constraint-driven optimization that reasons about declared
  `PRIMARY KEY`/`UNIQUE` metadata on a `LEFT JOIN`'s right side — but it
  lives in the *physical* join-execution layer
  (`src/execution/join_hashtable.cpp`, `src/execution/operator/join/physical_hash_join.cpp`),
  not in this logical-plan pass, and it does something categorically
  different: it doesn't remove the join, it picks a *cheaper physical probe
  strategy* for a `LEFT JOIN` whose right side is known, via a declared
  constraint, to match at most once per left row — the same uniqueness fact
  `JoinType::SINGLE` already carries structurally for decorrelated scalar
  subqueries (§11.2.3), but reached here from catalog metadata instead.
  `test/sql/join/left_outer/unique_left_join.test` exercises this directly
  (Tests 1–11, including an explicit "`FULL OUTER JOIN` should NOT use
  unique optimization" case, since that physical optimization's own internal
  match-tracking doesn't generalize to preserving *both* sides' unmatched
  rows). It is a real, PK/UNIQUE-aware optimization in this codebase — it
  just isn't `JOIN_ELIMINATION`, and conflating the two is an easy mistake
  to make precisely because they sound like they should be the same
  mechanism.
- **`SEMI`, `ANTI`, `MARK`, `INNER`, `OUTER` joins are entirely out of
  scope** for this pass's elimination (§11.2.3's `switch` statement) — only
  `LEFT`, `RIGHT`, and `SINGLE` are ever candidates.
- **A single non-conforming join condition anywhere in the list disqualifies
  rule 2 for the *entire* join**, not just that one conjunct — demonstrated
  directly by the `a1.ida = 1` and the `OR` test cases in §11.2.4.
- **A `GROUP BY`/`DISTINCT` only counts when it is traceable, through plain
  column-reference projections, back to a table index that the join
  actually reasons about** (either the inner side's own table index for
  rule 2, or the outer side's own table index for rule 3) — an aggregate
  sitting above the whole join and grouping on the *preserved* side's own
  key does not, by itself, prove anything about the *eliminated* side's
  uniqueness, even though it is tempting to assume "there's a `GROUP BY`
  right here, so obviously this is deduplicated" (§11.2.4's
  `sum(b.valueb) ... GROUP BY b.idb` case).
- **`DISTINCT ON` and multi-grouping-set `GROUP BY` (`ROLLUP`/`CUBE`/
  explicit multiple `GROUPING SETS`) are never registered at all** — the
  pass simply has no representation for "unique modulo this particular
  grouping set" and skips registering anything rather than guessing.
- **A `DISTINCT` group does not cross a `LIMIT`, `TOP_N`, `SAMPLE`,
  `WINDOW` or `UNNEST`.** The first four clear every group found above them
  (§11.2.4). `UNNEST` throws away groups found
  below it. In both cases, a join on the far side of that operator cannot
  use the group.

---

## 11.3 A full worked before/after example for each pass

### 11.3.1 `OUTER_JOIN_SIMPLIFICATION`: `LEFT JOIN` → `ANTI` join

```sql
SELECT l.i, r.i
FROM range(3) l(i)
LEFT JOIN range(2) r(i) ON l.i = r.i
WHERE r.i IS NULL
ORDER BY 1;
```

**Before** (the plan `OUTER_JOIN_SIMPLIFICATION` inherits — a `LEFT JOIN`
with an ordinary `IS NULL` filter above it):

```
LogicalOrder (l.i ASC)
  └─ LogicalProjection (l.i, r.i)
       └─ LogicalFilter (r.i IS NULL)
            └─ LogicalComparisonJoin [LEFT] (l.i = r.i)
                 ├─ LogicalGet (range(3) l)
                 └─ LogicalGet (range(2) r)
```

**After:** `r.i` is the join's own equi-condition column on the right, and
nothing above the join needs a real (non-`NULL`-testing) value from `r.i` —
so `TryConvertLeftToAntiJoin` (§11.1.1) fires. Three separate,
source-confirmed effects land in the same rewrite: the join type becomes
`ANTI`; `VisitFilter`'s cleanup (§11.1.4) erases the now-redundant
`r.i IS NULL` predicate from the filter, since an `ANTI` join's own
semantics already guarantee it; and `VisitProjection`'s cleanup replaces the
projection's `r.i` output expression with a literal, typed `NULL` constant,
since an `ANTI` join never produces the right side's columns as real data at
all:

```
LogicalOrder (l.i ASC)
  └─ LogicalProjection (l.i, CAST(NULL AS BIGINT) AS i)
       └─ LogicalComparisonJoin [ANTI] (l.i = r.i)
            ├─ LogicalGet (range(3) l)
            └─ LogicalGet (range(2) r)
```

The `LogicalFilter` node disappears entirely (it had exactly one predicate,
and that predicate was erased); the join's physical implementation in
Chapter 9 now only ever needs to build a hash table on `r.i` to test
membership, never to materialize or emit a single one of `r`'s actual output
columns — a strictly cheaper physical shape than the `LEFT` join it
replaced, on top of already being logically equivalent. Result:
`i=2 → NULL`, matching `range(3)`'s one value (`2`) that has no partner in
`range(2)` (`{0, 1}`).

### 11.3.2 `JOIN_ELIMINATION`: a `LEFT JOIN` deleted outright

```sql
SELECT b.*
FROM b
LEFT JOIN (SELECT DISTINCT ida AS ida FROM a) AS a1
  ON a1.ida = b.idb;
```

**Before** (the plan `JOIN_ELIMINATION` inherits):

```
LogicalProjection (b.idb, b.valueb)
  └─ LogicalComparisonJoin [LEFT] (a1.ida = b.idb)
       ├─ LogicalGet (b)
       └─ LogicalDistinct (ida)
            └─ LogicalProjection (a.ida AS ida)
                 └─ LogicalGet (a)
```

**After:** `SELECT DISTINCT ida FROM a` registers a `distinct_groups` entry
keyed to `a1`'s own table index, covering `{ida}`; the join's single
equality condition references exactly that column on the inner (`a1`) side;
nothing downstream of the join ever reads a column from `a1` at all
(`SELECT b.*` only). Rule 2 (§11.2.3) succeeds, and the entire join —
`a1`'s `LogicalDistinct`, its `LogicalProjection`, and `a`'s underlying
`LogicalGet`, everything on the eliminated side — is deleted, with the
join's parent wired directly onto `b`'s side instead:

```
LogicalProjection (b.idb, b.valueb)
  └─ LogicalGet (b)
```

No join operator, no hash-table build, no probe, and no scan of `a` at all
survives into the physical plan Chapter 9 builds from this tree — not merely
a cheaper join, as in §11.3.1, but zero join operators where the unoptimized
plan had one.

---

## 11.4 Summary: type change versus removal, one more time, precisely

`OUTER_JOIN_SIMPLIFICATION` (#16) and `JOIN_ELIMINATION` (#20) both open with
a bottom-line question of the shape "does this join actually need to do
everything its declared type promises, given everything else in this
specific query" — but they ask two different halves of that question, using
two entirely separate pieces of evidence, and neither one substitutes for
the other:

- `OUTER_JOIN_SIMPLIFICATION` asks **"does anything above this join still
  need the specific `NULL`-extension guarantee this join's type promises"**,
  and answers it purely from **filters, `IS [NOT] NULL` checks, and
  required-column sets accumulated top-down from everything sitting above
  the join** — it never once looks at `DISTINCT`, `GROUP BY`, or any notion
  of per-key uniqueness on either input.
- `JOIN_ELIMINATION` asks **"can this join's entire existence be proven
  unobservable — never duplicating, and (for `LEFT`/`RIGHT`) never
  filtering, the surviving side's rows"**, and answers it purely from
  **`DISTINCT`/`GROUP BY` structure traceable to one specific side, plus a
  hard requirement that nothing downstream reads the other side's columns
  at all** — it never once looks at what any filter above the join says
  about `NULL`s.

A join `OUTER_JOIN_SIMPLIFICATION` has just downgraded from `LEFT` to
`INNER` is not thereby any closer to being *eliminated* — `JOIN_ELIMINATION`
still needs its own, separate `DISTINCT`/`GROUP BY`-traced uniqueness proof
regardless of what type the join carries by the time pass #20 reaches it
(recall from §11.2.3 that `INNER` is not even in the list of join types
`JOIN_ELIMINATION` will touch — an `INNER` join arriving at pass #20 is
refused immediately by the `switch`'s `default:` branch, which means a
`LEFT` join that got downgraded to `INNER` by pass #16 has, if anything,
made itself *less* eligible for pass #20's elimination, not more). And a
join `JOIN_ELIMINATION` deletes was never touched by
`OUTER_JOIN_SIMPLIFICATION`'s logic at all — by the time pass #20 runs, four
passes after `OUTER_JOIN_SIMPLIFICATION`, whatever type-simplification was
going to happen to that join already happened or didn't; `JOIN_ELIMINATION`
either removes the join entirely, whatever type it currently carries, or
leaves it completely alone.

What both passes share, and what makes them worth reading back to back
despite the difference, is the same fixed-pipeline discipline §7.2
established for the optimizer as a whole: each is a single, one-shot,
self-contained analysis over the tree it inherits, with no re-triggering and
no fixpoint — `OUTER_JOIN_SIMPLIFICATION`'s downgrade has to have already
happened by the time `JOIN_ORDER` builds its query graph one pass later
(§11.1.7), and `JOIN_ELIMINATION`'s own uniqueness proof has to be traceable
entirely from `DISTINCT`/`GROUP BY` structure and column-usage facts that are
already fully resolved by the time pass #20 gets its one and only turn — in
this checkout, that still excludes the catalog entirely (§11.2.1), which is
this file's most concrete, single most surprising, fully source-and-git-
history-verified finding: the classic "eliminate the join to the dimension
table because its primary key guarantees at most one match" textbook story
is, at `cb5673685153`, not yet how `JOIN_ELIMINATION` actually proves
anything at all.
