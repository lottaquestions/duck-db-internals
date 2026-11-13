# Chapter 7, Deep Dive 4 — Enums, Dates, and Aggregate-Shape Rules in the Expression Rewriter

## 4.1 Scope, and a note on the test format

Chapter 7's overview (`00-overview.md`, §7.5) established that pass #1 of the ~45-pass
optimizer pipeline, `EXPRESSION_REWRITER`, is itself a fixed, ordered list of 37
small, independent rule classes under `src/optimizer/rule/*.cpp`, each built around
a matcher (what expression shape to look for) and an `Apply` method (what to do once
found). This file goes deep on eight of those rules, grouped by what they actually
share — not "they're all in the rule list" (all 35 share that), but a real mechanical
kinship:

- **§4.2** `InClauseSimplificationRule` — a narrow, cast-pushing rewrite for `IN`
  lists, plus (as a directly adjacent but *distinct* mechanism) the real story of
  how `x NOT IN (..., NULL, ...)` gets proven empty.
- **§4.3** `InEnumSimplificationRule`, `EnumCompareSimplificationRule`, and
  `EnumComparisonRule` — three rules, split across two source files in a way that
  is easy to get wrong, all about replacing string comparison against an `ENUM`'s
  dictionary with integer comparison against its internal code.
- **§4.4** `DatePartSimplificationRule` and `DateTruncSimplificationRule` — constant
  folding for calendar functions, one trivial, one genuinely intricate.
- **§4.5** `OrderedAggregateOptimizer` — when an aggregate's own `ORDER BY` clause is
  provably redundant, including a real, once-shipped bug where "provably redundant"
  was computed wrong.
- **§4.6** `DistinctAggregateOptimizer` and `DistinctWindowedOptimizer` — a much
  narrower rule than its name suggests, and a look at the very different, much
  larger mechanism its test-suite neighbors actually belong to.

All eight rules share one architectural fact worth restating up front because it
shapes everything below: they are *expression*-local. None of them changes the
*shape* of the operator tree — no new `LogicalFilter`, no new join, no new
`LogicalAggregate`. Each one either replaces one bound expression with a smaller,
cheaper, semantically identical one, or clears a flag (`DISTINCT`, an `ORDER BY`
modifier) on an existing `BoundAggregateExpression`/`BoundWindowExpression` in
place. That narrowness is precisely why each rule earns exactly one paragraph of
inline comment in `optimizer.cpp` and one small `.cpp` file of its own — and why
getting each one's edge cases (NULL propagation above all) exactly right matters so
much: a plan-shape rewrite that's slightly too aggressive usually just produces a
*slower* plan; an expression-level rewrite that's slightly too aggressive produces
a *wrong answer*, silently, on exactly the queries a query author is least likely to
notice (a comparison to `NULL`, a `NOT IN` list with a stray `NULL` element, a
`ROLLUP` total row).

**On the sqllogictest same-label idiom.** Several test files below (starting with
`test/optimizer/ordered_aggregate.test` in §4.5) use a format worth naming once,
since this is the first deep-dive file in this directory to lean on it: two or more
`query I nosort <label>` blocks that share the same `<label>` are checked for
**byte-identical output against each other**, not against a literal expected value
written in the file. The file's own `----` sections are left empty on purpose. This
is DuckDB's own idiom for "prove a rewrite happened": if `EXPLAIN SELECT
FIRST(i ORDER BY i) FROM integers` (one query, tagged `order_by_agg`) produces
output byte-for-byte identical to `EXPLAIN SELECT FIRST(i) FROM integers` (a second,
differently-written query, same tag), then whatever rewrote the first query's plan
into the second query's plan shape actually ran — the test doesn't need to spell
out what the optimized `EXPLAIN` text looks like at all, it only needs to assert
that two *different pieces of SQL* converged onto *the same plan*. This is more
robust than pinning an exact `EXPLAIN` string (which would break on every harmless
formatting change to the pretty-printer) while still being a strict, executable
proof that a specific rewrite fired.

---

## 4.2 `InClauseSimplificationRule` — and the mechanism it is not

**Purpose, in one sentence:** when the left-hand side of an `IN`/`NOT IN` list is
a lossless, invertible cast of a plain column reference, move the cast off the
column and onto the (few) list constants instead, so the column can still be
compared in its own native type — and therefore still be seen directly by a
zonemap or index — rather than being hidden behind a per-row cast the storage layer
can't see through.

### 4.2.1 The source, read carefully

```cpp
// src/optimizer/rule/in_clause_simplification_rule.cpp:13
InClauseSimplificationRule::InClauseSimplificationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	// match on InClauseExpression that has a ConstantExpression as a check
	auto op = make_uniq<InClauseExpressionMatcher>();
	op->policy = SetMatcher::Policy::SOME;
	root = std::move(op);
}

unique_ptr<Expression> InClauseSimplificationRule::Apply(LogicalOperator &op, vector<reference<Expression>> &bindings,
                                                         bool &changes_made, bool is_root) {
	auto &expr = bindings[0].get().Cast<BoundOperatorExpression>();
	if (!BoundCastExpression::IsCast(*expr.GetChildrenMutable()[0])) {
		return nullptr;
	}
	auto &cast_expression = expr.GetChildrenMutable()[0]->Cast<BoundFunctionExpression>();
	if (BoundCastExpression::IsTryCast(cast_expression)) {
		// a TRY_CAST turns failing values into NULL, which is lost when the cast is removed
		return nullptr;
	}
	if (BoundCastExpression::Child(cast_expression).GetExpressionClass() != ExpressionClass::BOUND_COLUMN_REF) {
		return nullptr;
	}
	//! The goal here is to remove the cast from the probe expression
	//! and apply a cast to the constant expressions. We can only do this
	//! if the semantics do not change, which only happens when BOTH casts
	//! are invertible.
	auto target_type = BoundCastExpression::SourceType(cast_expression);
	if (!BoundCastExpression::CastIsInvertible(target_type, cast_expression.GetReturnType())) {
		return nullptr;
	}
	vector<unique_ptr<BoundConstantExpression>> cast_list;
	//! First check if we can cast all children
	for (size_t i = 1; i < expr.GetChildrenMutable().size(); i++) {
		if (expr.GetChildrenMutable()[i]->GetExpressionClass() != ExpressionClass::BOUND_CONSTANT) {
			return nullptr;
		}
		D_ASSERT(expr.GetChildrenMutable()[i]->IsFoldable());
		auto constant_value = ExpressionExecutor::EvaluateScalar(GetContext(), *expr.GetChildrenMutable()[i]);
		if (!BoundCastExpression::CastIsInvertible(constant_value.type(), target_type)) {
			return nullptr;
		}
		auto new_constant = constant_value.DefaultTryCastAs(target_type);
		if (!new_constant) {
			return nullptr;
		} else {
			auto new_constant_expr = make_uniq<BoundConstantExpression>(std::move(*new_constant));
			cast_list.push_back(std::move(new_constant_expr));
		}
	}
	//! We can cast, so we move the new constant
	for (size_t i = 1; i < expr.GetChildrenMutable().size(); i++) {
		expr.GetChildrenMutable()[i] = std::move(cast_list[i - 1]);
	}
	//! We can cast the full list, so we move the column
	expr.GetChildrenMutable()[0] = std::move(BoundCastExpression::ChildMutable(cast_expression));
	return nullptr;
}
```

There are two separate invertibility checks here, and the difference between them
is exactly what a bug once slipped through:

1. **The declared-cast check** (source lines 38–41): is casting from the column's own
   type (`target_type`, obtained via `SourceType(cast_expression)`) to whatever type
   the visible `CAST` produces (`cast_expression.GetReturnType()`) invertible?
2. **The per-constant check** (inside the loop): is casting from *this specific
   constant's own type* back to the column's type invertible?

Both have to hold, because the rewrite is doing two things at once: proving the
*general shape* of the cast is safe to move, and proving each *specific literal* in
the list actually round-trips. `BoundCastExpression::CastIsInvertible` itself
(`src/planner/expression/bound_cast_expression.cpp:155`) is a purely type-level
function — it never looks at runtime data — and it has an explicit, named case for
exactly the dangerous direction this rule has to avoid:

```cpp
// src/planner/expression/bound_cast_expression.cpp:206
switch (source_type.id()) {
case LogicalTypeId::TIMESTAMP:
// ... (all TIMESTAMP variants)
	switch (target_type.id()) {
	// ...
	case LogicalTypeId::DATE:
	case LogicalTypeId::TIME:
	case LogicalTypeId::TIME_TZ:
		return false;
	// ...
```

`CastIsInvertible(TIMESTAMP, DATE)` is hard-coded `false`: a `TIMESTAMP → DATE` cast
throws away the time-of-day, so it can never be "inverted" — going `DATE → TIMESTAMP`
afterward does not recover the original value. The opposite direction,
`CastIsInvertible(DATE, TIMESTAMP)`, falls through every special case in the
function all the way to its final `return true;` (line 280): `DATE → TIMESTAMP` is a
pure widening (every date maps to exactly one timestamp, at midnight), so it round
trips perfectly.

The rule and `CastIsInvertible` refuse several other casts too:

- The rule skips a `TRY_CAST`. A `TRY_CAST` turns a value that does not fit into
  `NULL`. If the rule removed the cast, that `NULL` would be lost.
- Every `ENUM → ENUM` cast is "not invertible". Two enums can list the same
  strings in a different order, so the cast can change how values compare.
- A narrowing `DECIMAL` cast is not invertible either. The target must hold
  every digit before the decimal point. For example, `SMALLINT` (width 5) cannot
  hold `99999`.
- A nested cast (`LIST`, `ARRAY`, `MAP`, `STRUCT`) is invertible only if its shape
  stays the same and every child cast is invertible (`NestedCastIsInvertible`,
  `bound_cast_expression.cpp:115`).
- `TIME_NS → TIME` is not invertible, because it rounds away the nanoseconds.

`test/sql/types/enum/enum_cast_optimizer.test` shows the first two points. With
`e1 = ('a','b','c')` and `e2 = ('c','b','a')`, the query
`SELECT e, e::e2 >= 'c'::e2 FROM t` must return `true` for all three rows. And
`TRY_CAST(e AS ENUM('a','b')) NOT IN ('a'::ENUM('a','b'))` must return `NULL` for
the row `'c'`, because that row does not fit in the smaller enum.

### 4.2.2 A bug this rule actually shipped: swapped arguments (issue #13380)

`test/sql/optimizer/expression/test_in_clause_simplification.test` looks, at first
glance, like it should exercise the rewrite:

```sql
-- test/sql/optimizer/expression/test_in_clause_simplification.test
CREATE TABLE issue13380(c0 TIMESTAMP);
INSERT INTO issue13380(c0) VALUES ('2024-08-09 14:48:00');

SELECT c0::DATE IN ('2024-08-09') d FROM issue13380;
-- true
SELECT NOT (c0::DATE IN ('2024-08-09')) FROM issue13380;
-- false
SELECT c0::DATE NOT IN ('2024-08-09') FROM issue13380;
-- false
```

`c0` is `TIMESTAMP`. The expression is `CAST(c0 AS DATE) IN ('2024-08-09')` — cast
of a column, exactly the shape `InClauseSimplificationRule` matches. If the rule
fired, `target_type = SourceType(cast_expression) = TIMESTAMP`, and the guard would
be checking `CastIsInvertible(TIMESTAMP, DATE)`, which the switch above returns
`false` for — so, correctly, the rule declines and leaves `c0::DATE IN ('2024-08-09')`
to be evaluated exactly as written, per row. That is why the test's expected
answers (`true`, `false`, `false`) are the *un-rewritten*, semantically correct
ones: `c0`'s actual value is `2024-08-09 14:48:00`, and `c0::DATE` truncates that
down to `2024-08-09`, which does equal the literal — the answer has nothing to do
with whether `c0` itself was ever exactly midnight.

The file's own one-line description — *"Removing the TIMESTAMP => DATE cast only
works if the IN clause has TIMESTAMPs"* — is a regression comment. It describes
what the code looked like **before** it was fixed. The fix is commit `0035bdc665`
("Issue #13380: IN Invertible Casts", fixing `duckdb/duckdb#13380`):

```diff
// src/optimizer/rule/in_clause_simplification_rule.cpp (commit 0035bdc665)
-	//! Here we check if we can apply the expression on the constant side
+	//! The goal here is to remove the cast from the probe expression
+	//! and apply a cast to the constant expressions. We can only do this
+	//! if the semantics do not change, which only happens when BOTH casts
+	//! are invertible.
 	auto target_type = cast_expression.source_type();
-	if (!BoundCastExpression::CastIsInvertible(cast_expression.return_type, target_type)) {
+	if (!BoundCastExpression::CastIsInvertible(target_type, cast_expression.return_type)) {
 		return nullptr;
 	}
 	vector<unique_ptr<BoundConstantExpression>> cast_list;
 	for (size_t i = 1; i < expr.children.size(); i++) {
 		// ...
 		auto constant_value = ExpressionExecutor::EvaluateScalar(GetContext(), *expr.children[i]);
+		if (!BoundCastExpression::CastIsInvertible(constant_value.type(), target_type)) {
+			return nullptr;
+		}
 		auto new_constant = constant_value.DefaultTryCastAs(target_type);
```

Before this patch, the call was `CastIsInvertible(cast_expression.return_type,
target_type)` — the two arguments in the *wrong order*. That call is
`CastIsInvertible(DATE, TIMESTAMP)`, which is `true` (the harmless, widening
direction) — checking, in effect, "can I invert `DATE → TIMESTAMP`?" when the
question that actually needed answering was "can I invert `TIMESTAMP → DATE`?" With
the arguments swapped, the guard *always* said yes for exactly the dangerous shape
this rule exists to be careful about, and the rewrite proceeded: it would have
turned `c0::DATE IN ('2024-08-09')` into `c0 IN (TIMESTAMP '2024-08-09 00:00:00')`
by casting the `DATE` literal up to a midnight `TIMESTAMP` and dropping the cast off
`c0`. For the test's actual data — `c0 = '2024-08-09 14:48:00'`, not midnight —
that rewritten comparison evaluates to `false`, while the correct, as-written
comparison evaluates to `true`. **A silently wrong query result**, from one
transposed pair of function arguments. The fix does two things: it corrects the
argument order on the existing check, and it adds the second, per-constant
`CastIsInvertible` check the diff shows above — a defense against the case where
the *general* cast direction is safe but one specific literal in the list isn't
(for instance, a list mixing an `INTEGER_LITERAL`-typed constant with values that
don't all survive the same round trip).

This is worth sitting with for a second, because it is exactly the kind of mistake
that is easy to make and easy for a reviewer to miss: `CastIsInvertible(a, b)` and
`CastIsInvertible(b, a)` are *not* symmetric — the whole point of the function is
that widening and narrowing are different directions — and a call site that
transposes them doesn't fail to compile, doesn't crash, and doesn't fail any type
check. It just quietly answers a different, easier question than the one that
needed answering, and the wrongness only shows up on rows where the two questions'
answers happen to diverge — here, any row whose `TIMESTAMP` wasn't exactly
midnight. `test_in_clause_simplification.test` is the guard that makes sure that
mistake, once found, stays fixed.

**Edge case worth naming explicitly:** despite what a plausible-sounding name might
suggest, `InClauseSimplificationRule` does **not** do generic `IN`-list cleanup —
no deduplication of repeated list elements, no collapsing `x IN (a)` down to
`x = a`. Its `Apply` has exactly one job (move an invertible cast off a column, onto
the list's constants) and returns `nullptr` (no rewrite) the moment any part of that
one job's preconditions fails to hold. Anything resembling "simplify this `IN` list"
in the more generic sense is either handled by a completely different rule (§4.3,
for `ENUM`-typed lists specifically) or not handled at expression-rewrite time at
all.

### 4.2.3 Contrast with the plan-level `IN_CLAUSE` pass

`00-overview.md` (§7.4 item 9, §7.5, §7.6.6) already establishes this distinction in
full, so only the load-bearing summary belongs here: `InClauseSimplificationRule` is
one of the 37 rules inside pass #1 (`EXPRESSION_REWRITER`) and operates purely on
one expression's shape, regardless of list length. `InClauseRewriter`
(`src/optimizer/in_clause_rewriter.cpp`) is a completely separate, later,
*plan-level* pass (`OptimizerType::IN_CLAUSE`, #9 in the fixed pipeline) that
rewrites a **long** (`≥ IN_CLAUSE_REWRITE_THRESHOLD = 6` children,
`src/include/duckdb/optimizer/in_clause_rewriter.hpp:31`), all-foldable `IN`/`NOT IN`
list into a `MARK` join against a materialized `LogicalColumnDataGet` — turning
per-row linear list-scanning into a hash-join-style probe. The two share a name
purely by English-vocabulary coincidence; nothing about the small, expression-local
rule described above has anything to do with join strategy.

### 4.2.4 Where `NOT IN`'s `NULL` handling actually lives (it is not this rule)

`test/optimizer/not_in_null.test` is exactly the kind of file this task description
expected `InClauseSimplificationRule` to own, and it is worth showing precisely why
it does not, because the real mechanism is more interesting, and getting the
attribution right is the whole point of reading source instead of guessing from
names.

```sql
-- test/optimizer/not_in_null.test
CREATE TABLE integers(i INTEGER);
INSERT INTO integers VALUES (1), (2), (NULL);

EXPLAIN SELECT count(*) FROM integers WHERE i NOT IN (NULL);
-- physical_plan   <REGEX>:.*Empty Result.*
SELECT count(*) FROM integers WHERE i NOT IN (NULL);
-- 0

EXPLAIN SELECT count(*) FROM integers WHERE i NOT IN (NULL) AND i > 0;
-- physical_plan   <REGEX>:.*Empty Result.*
SELECT count(*) FROM integers WHERE i NOT IN (NULL) AND i > 0;
-- 0
```

**The SQL reasoning first, because it is the classic trap.** `i NOT IN (NULL)` is
*not* `i` failing to match anything in `(NULL)` and therefore passing — three-valued
logic says otherwise, and it is worth working through carefully:

- `i IN (NULL)` asks: does `i` equal any element of the list `{NULL}`? The only
  candidate comparison is `i = NULL`, and a comparison against `NULL` is *never*
  `TRUE` and *never* provably `FALSE` — it is `UNKNOWN`, for every possible value of
  `i`, including `NULL` itself. Since the list has no other element that could
  rescue the disjunction with a definite `TRUE`, `i IN (NULL)` evaluates to
  `UNKNOWN` (SQL's `NULL`) for **every row**, unconditionally — not just rows where
  `i` is itself `NULL`.
- `NOT (UNKNOWN)` is `UNKNOWN` again (SQL's `NOT` maps `TRUE↔FALSE` and leaves `NULL`
  as `NULL`). So `i NOT IN (NULL)` is `NULL` for every row, too.
- A `WHERE` clause only keeps rows where the predicate is definitely `TRUE`. A row
  whose predicate evaluates to `NULL` is filtered out exactly like a row whose
  predicate is `FALSE`. So **every row of `integers` is excluded**, regardless of
  whether that row's `i` is `1`, `2`, or `NULL` — the result is `0` rows, and it
  would be `0` rows even if `integers` held a million distinct non-null values, none
  of which is anywhere close to the list.

The naive mistake this trips people into is reading `NOT IN` as "negate the `IN`
result the way you'd negate a boolean" — `i NOT IN (1, 2)` "should" behave like `i
!= 1 AND i != 2`. That equivalence *does* hold when every list element is
non-`NULL`. It breaks the instant even one list element is `NULL`, because the
presence of that one unknown-comparison poisons the whole disjunction (and, after
`NOT`, the whole conjunction) into `UNKNOWN` rather than a definite answer — even
for rows that would have cleanly, definitely failed to match every *other* element
of the list. A three-element `NOT IN (1, 2, NULL)` against `i = 5`: is `5` distinct
from `1`? Yes. From `2`? Yes. From `NULL`? Unknown. `NOT IN` can only answer `TRUE`
if it can rule out *every* element with certainty — one unresolvable element is
enough to downgrade the whole expression to `NULL`, regardless of how many of the
other elements were cleanly ruled out.

**Now, the actual optimizer mechanism — and it is not `InClauseSimplificationRule`.**
That rule's very first check (`BoundCastExpression::IsCast(*expr.GetChildrenMutable()[0])`,
§4.2.1) requires the `IN` list's probe expression to be a cast of a column. `i` in
`i NOT IN (NULL)` is a plain `BoundColumnRefExpression` — no cast anywhere — so the
rule's matcher never even gets a foothold. Tracing where the `Empty Result` in the
`EXPLAIN` output actually comes from leads through three separate pieces of
machinery, not one rule:

1. **`FILTER_PUSHDOWN`** (pass #6, `src/optimizer/filter_combiner.cpp`) pushes
   `i NOT IN (NULL)` down onto `integers`' own `LogicalGet` as a `TableFilter`.
   It does not match any of the *specialized* `IN`-list handlers in that file (those
   require the un-negated shape `COMPARE_IN` directly, see `TryPushdownInFilter`,
   `filter_combiner.cpp:644`) — instead it falls back to the fully generic path that
   wraps the whole predicate expression as an opaque `ExpressionFilter`
   (`filter_combiner.cpp:485`).
2. **`STATISTICS_PROPAGATION`** (pass #34) walks every `LogicalGet`'s table filters
   via `StatisticsPropagator::PropagateTableFilter`
   (`src/optimizer/statistics/operator/propagate_get.cpp:29`), which calls:

   ```cpp
   // src/optimizer/statistics/operator/propagate_get.cpp:41
   auto check_result = filter.Cast<ExpressionFilter>().CheckStatistics(stats);
   ```

   `ExpressionFilter::CheckStatistics` dispatches by expression shape
   (`src/planner/filter/expression_filter.cpp:825`), and for an `OPERATOR_NOT`
   wrapping a `COMPARE_IN`, it calls `CheckNotOperatorStatistics`
   (`expression_filter.cpp:795`), which contains the exact, targeted case:

   ```cpp
   // src/planner/filter/expression_filter.cpp:795
   static FilterPropagateResult CheckNotOperatorStatistics(optional_ptr<ClientContext> context_p,
                                                           const BoundOperatorExpression &op_expr,
                                                           array_ptr<const BaseStatistics> input_stats) {
   	if (op_expr.GetChildren().size() != 1) {
   		return FilterPropagateResult::NO_PRUNING_POSSIBLE;
   	}
   	auto &child = *op_expr.GetChildren()[0];
   	if (child.GetExpressionType() == ExpressionType::COMPARE_IN) {
   		auto &children = child.Cast<BoundOperatorExpression>().GetChildren();
   		if (children.size() == 2 && children[1]->GetExpressionType() == ExpressionType::VALUE_CONSTANT &&
   		    children[1]->Cast<BoundConstantExpression>().GetValue().IsNull()) {
   			return FilterPropagateResult::FILTER_FALSE_OR_NULL;
   		}
   	}
   	// ...
   	// a child matching every row makes NOT match no row - the converse does not hold, since a child
   	// that matches no row may be NULL rather than false, and NOT NULL does not match either
   ```

   Notice this check takes **no use of the column's actual statistics at all** for
   the single-`NULL`-element case — it doesn't need to, since the reasoning above
   (`NOT IN (NULL)` is unconditionally `NULL`) holds for *any* column, regardless of
   what values it actually contains. This is a purely structural proof, not a
   data-dependent one.
3. Back in `propagate_get.cpp`'s `LogicalGet` handling, any table filter classified
   as anything other than `NO_PRUNING_POSSIBLE` collapses the whole scan (and,
   transitively, the query) to a `LogicalEmptyResult`
   (`src/optimizer/statistics/operator/propagate_get.cpp:234`) — the same
   `ReplaceWithEmptyResult` helper cited in `00-overview.md`'s §7.6.2.

The `AND i > 0` variant in the test's second query needs nothing extra: standard
conjunction short-circuiting means one conjunct classified `FILTER_FALSE_OR_NULL`
makes the whole `AND` unsatisfiable regardless of what the other conjunct says
(`CheckConjunctionStatistics`, `expression_filter.cpp:841`).

There is a small, honest terminology wrinkle worth flagging while we're here. The
sibling function for the *un-negated* case, `CheckInOperatorStatistics`
(`expression_filter.cpp:741`), classifies a `NULL`-only `IN` list as
`FILTER_ALWAYS_FALSE`, not `FILTER_FALSE_OR_NULL`:

```cpp
// src/planner/filter/expression_filter.cpp:763
if (values.empty()) {
	return FilterPropagateResult::FILTER_ALWAYS_FALSE;
}
```

Strictly, `x IN (NULL)` is `NULL` for every row (never a definite `FALSE`) — so
`FILTER_FALSE_OR_NULL` would be the more precise label. But both classifications
feed the exact same `!= NO_PRUNING_POSSIBLE` pruning check one layer up, so the
observable behavior (the whole scan collapses to zero rows) is identical either
way; the imprecision is harmless because nothing downstream ever tries to further
distinguish "provably false" from "provably false-or-null" once a filter has
already been classified as unconditionally unsatisfiable.

`CheckInOperatorStatistics` is also worth a second look for a different reason: it
independently reimplements the same "does this list cover the column's whole known
range" tautology check that §4.3's `InEnumSimplificationRule` implements for
`ENUM` domains —

```cpp
// src/planner/filter/expression_filter.cpp:771
if (result == FilterPropagateResult::NO_PRUNING_POSSIBLE &&
    NumericStats::ConstantsCoverRange(*filter_stats, constants)) {
	// an integral column whose whole [min, max] range is listed always matches
	result = FilterPropagateResult::FILTER_ALWAYS_TRUE;
}
```

— just against a numeric column's propagated `[min, max]` zonemap range instead of
a fixed dictionary. Two independent pieces of code, arrived at the same underlying
idea (a list that provably covers every value a column could hold makes an `IN`
test tautological) via two different data sources (statistics vs. a type's fixed
domain). Neither one is `InClauseSimplificationRule`.

---

## 4.3 The `ENUM` rules — three classes, two files, one shared trick

DuckDB's `ENUM` type is a dictionary-encoded fixed set of string values: creating
`CREATE TYPE mood AS ENUM ('sad', 'ok', 'happy')` stores the three strings once, in
a fixed order, and every column value of type `mood` is stored internally as a
small unsigned integer — the string's position in that dictionary — not as a
repeated copy of the string itself. Comparing two `ENUM` values of the *same*
underlying type is therefore, at the storage layer, just integer comparison.
Comparing an `ENUM` value against a string *literal* naively requires a dictionary
lookup (or worse, decoding the enum back to a string) on every row. These three
rules exist to notice, at optimization time, when a string-literal comparison can
be pre-resolved to the enum's internal integer code exactly once, so the
per-row runtime cost drops to plain integer comparison.

**A source-location note worth flagging on its own**, because it's exactly the kind
of naming trap this book's overview file warns about for other rules: despite the
header `duckdb/optimizer/rule/enum_comparison.hpp` sounding like the obvious home
for all "enum comparison" logic, only **`EnumComparisonRule`** lives there, in its
own `.cpp` file (`src/optimizer/rule/enum_comparison.cpp`). The other two —
**`InEnumSimplificationRule`** and **`EnumCompareSimplificationRule`** — are
declared in a *different* header (`duckdb/optimizer/rule/in_clause_simplification.hpp`)
and implemented in the *same* `.cpp` file as `InClauseSimplificationRule`
(`src/optimizer/rule/in_clause_simplification_rule.cpp`), grouped there because they
share private helper classes (`EnumInProbeMatcher`, `EnumInConstantMatcher`) with
that file's other rule, not because of anything to do with `enum_comparison.cpp`.
Three rule *names* about enums; two files.

### 4.3.1 `InEnumSimplificationRule` — enum values inside an `IN` list

**Purpose:** for `enum_col IN (const1, const2, ...)` (or the `enum_col::VARCHAR IN
(...)` cast form), replace each string constant with its integer `ENUM` code, drop
any string that isn't a member of this particular enum's domain (it can never
match), and, when every possible value of the enum's domain is present, collapse
the whole comparison to a `NULL`-preserving constant.

```cpp
// src/optimizer/rule/in_clause_simplification_rule.cpp:114
InEnumSimplificationRule::InEnumSimplificationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	// match enum IN constants and enum::VARCHAR IN constants
	auto op = make_uniq<InUniformExpressionMatcher>();
	op->probe_matcher = make_uniq<EnumInProbeMatcher>();
	op->child_matcher = make_uniq<EnumInConstantMatcher>();
	root = std::move(op);
}
```

The `Apply` method (`in_clause_simplification_rule.cpp:139`) builds a lookup table
once (`string_map_t<uint64_t> enum_domain`, populated from
`EnumType::GetValuesInsertOrder`/`GetSize`), then walks the `IN` list's constants:

```cpp
// src/optimizer/rule/in_clause_simplification_rule.cpp:160
for (idx_t i = 1; i < children.size(); ++i) {
	auto &child = *children[i];
	auto &v = child.Cast<BoundConstantExpression>().GetValue();
	if (v.IsNull()) {
		//	Keep NULLs
		in_children.emplace_back(make_uniq<BoundConstantExpression>(Value(enum_type)));
	} else {
		// ...
		auto s = v.ToString();
		auto it = enum_domain.find(s);
		if (it != enum_domain.end()) {
			//	The literal is in the ENUM domain, so translate it
			in_children.emplace_back(make_uniq<BoundConstantExpression>(Value::ENUM(it->second, enum_type)));
			matched_enum_values.insert(it->second);
		}
		// Not in the domain, so ignore it.
	}
}
```

Three outcomes, each with its own care taken over `NULL` semantics:

1. **`matched_enum_values.size() == EnumType::GetSize(enum_type)`** — every single
   value the enum can hold appeared somewhere in the list. For any non-`NULL` enum
   value, `x IN (<every possible value>)` is trivially `TRUE` (and `NOT IN` is
   trivially `FALSE`) — but a `NULL` probe must still produce `NULL`, not `TRUE`/`FALSE`.
   The rule collapses to exactly that, via the same `ConstantOrNull` helper used
   throughout the expression rewriter:

   ```cpp
   // src/optimizer/rule/in_clause_simplification_rule.cpp:184
   if (matched_enum_values.size() == EnumType::GetSize(enum_type)) {
   	const bool in_clause = (expr.GetExpressionType() == ExpressionType::COMPARE_IN);
   	return rewriter.ConstantOrNull(GetContext(), in_children[0]->Copy(), Value::BOOLEAN(in_clause));
   }
   ```
2. **Some (but not all) matched** — the rewritten list keeps only the in-domain
   literals, now translated to their integer codes, and the comparison stays a
   (now cheaper) `IN`/`NOT IN` over that pruned list.
3. **None matched at all** (`in_children.size() == 1`, only the probe remains) —
   the whole comparison collapses the other way: for a plain `IN`, nothing in the
   list can ever match, so the result is `FALSE`-or-`NULL`; for `NOT IN`, it's
   `TRUE`-or-`NULL`:

   ```cpp
   // src/optimizer/rule/in_clause_simplification_rule.cpp:191
   const bool not_in = (expr.GetExpressionType() == ExpressionType::COMPARE_NOT_IN);
   return rewriter.ConstantOrNull(GetContext(), in_children[0]->Copy(), Value::BOOLEAN(not_in));
   ```

**Test walkthrough.** `test/sql/optimizer/expression/test_enum_in_simplification.test`
builds a ten-value `sym_enum` (`ENUM (SELECT i::VARCHAR FROM range(10) tbl(i))`, i.e.
`'0'` through `'9'`) and drives every branch above:

```sql
-- test/sql/optimizer/expression/test_enum_in_simplification.test
-- a mixed list: some literals in-domain ('0'..'9'), some not ('10'..'12'), one NULL
EXPLAIN SELECT sym, ... FROM quote
WHERE sym IN (NULL, '0', '1', ..., '9', '10', '11', '12')
GROUP BY sym ORDER BY sym;
-- physical_plan   <!REGEX>:.*AS VARCHAR.*     -- no VARCHAR cast survives: it's all integer-coded now

-- an all-out-of-domain list: every literal is outside sym_enum's ten-value domain
EXPLAIN SELECT sym, ... FROM quote WHERE sym IN ('10', '11', '12') GROUP BY sym ORDER BY sym;
-- physical_plan   <REGEX>:.*EMPTY_RESULT.*    -- nothing can ever match; whole plan collapses

-- the full domain, every one of '0'..'9', nothing left over
EXPLAIN SELECT count(*) FROM quote WHERE sym IN ('0','1','2','3','4','5','6','7','8','9');
-- physical_plan   <!REGEX>:.*constant_or_null.*   -- collapses further still; see below
EXPLAIN SELECT count(*) FROM quote WHERE sym NOT IN ('0','1','2','3','4','5','6','7','8','9');
-- physical_plan   <REGEX>:.*EMPTY_RESULT.*
```

The full-domain `IN` case is worth one extra step: `InEnumSimplificationRule` itself
only gets as far as `constant_or_null(TRUE, sym)` (case 1 above). A *separate*
mechanism, `ConstantOrNullSimplification` (run in the same `EXPRESSION_REWRITER`
step per `00-overview.md` §7.4 item 1, `src/optimizer/constant_or_null_simplification.cpp`),
can fold `constant_or_null(c, x)` straight down to the bare constant `c` whenever it
can *prove `x` is never `NULL`* from the plan's own statistics — which the
`<!REGEX>:.*constant_or_null.*` assertion above confirms happened here (the
`quote` table, at that point in the test, has no `NULL` `sym` values yet). This
fold is explicitly guarded against side-effecting plans:

```cpp
// src/optimizer/constant_or_null_simplification.cpp:146
// Folding removes the NULL check, so disable it for plans with side effects.
// Same-statement DML can add NULLs after statistics-based nullability analysis.
const bool allow_folding = !plan_has_side_effects;
```

— and `test/sql/optimizer/expression/enum_full_domain_fold_guard.test` demonstrates
exactly why: a `WITH ins AS (INSERT ... VALUES (NULL) RETURNING *) SELECT count(*)
FROM t_enum WHERE m IN ('sad', 'ok', 'happy')` correctly returns `3` (not `4`),
because the CTE's freshly-inserted `NULL` row is visible in the same statement but
the optimizer's *pre-execution* statistics snapshot never saw it; disabling the
fold on any side-effecting plan means `constant_or_null(TRUE, m)` keeps evaluating
per row (correctly filtering the `NULL` row via ordinary `NULL` propagation)
instead of getting collapsed to a bare `TRUE` that would have let the `NULL` row
through. This fold-guard is not part of `InEnumSimplificationRule` itself — it's a
downstream safety net for a rewrite *this* rule produces — but it's directly
responsible for several of `test_enum_in_simplification.test`'s assertions, so it's
worth naming precisely rather than silently attributing to the wrong rule.

**Edge case:** a literal that matches nothing in the enum's domain isn't an error —
it's silently dropped from the list, because a value that provably can't equal
anything the column could ever hold contributes nothing to the disjunction. This is
visible in `test_enum_comparisons.test`: `SELECT person_mood IN ('beaming'::pet_mood)
FROM person_pet_den` (comparing a `mood`-typed column against a value from the
*different* `pet_mood` enum) returns `False` for every row — `'beaming'` isn't in
`mood`'s three-value domain at all, so the list is empty after pruning, and (per
branch 3 above) the whole thing becomes `constant_or_null(person_mood, FALSE)`.

### 4.3.2 `EnumCompareSimplificationRule` — the scalar-comparison sibling

**Purpose:** the same trick as `InEnumSimplificationRule`, for a plain (non-`IN`)
comparison between `enum_col::VARCHAR` and a single string constant — `=`, `<>`,
`IS DISTINCT FROM`, `IS NOT DISTINCT FROM`.

```cpp
// src/optimizer/rule/in_clause_simplification_rule.cpp:198
EnumCompareSimplificationRule::EnumCompareSimplificationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	// match enum::VARCHAR = string literal
	auto op = make_uniq<ComparisonExpressionMatcher>();
	vector<ExpressionType> types;
	types.emplace_back(ExpressionType::COMPARE_EQUAL);
	types.emplace_back(ExpressionType::COMPARE_NOTEQUAL);
	types.emplace_back(ExpressionType::COMPARE_DISTINCT_FROM);
	types.emplace_back(ExpressionType::COMPARE_NOT_DISTINCT_FROM);
	op->expr_type = make_uniq<ManyExpressionTypeMatcher>(types);
	// ...
```

If the string is found in the enum's domain, both operands are replaced with the
plain `(enum_col, Value::ENUM(i, enum_type))` pair, dropping the `::VARCHAR` cast
entirely. If the string is **not** in the domain, the rule resolves the comparison
directly, with each of the four operators handled by its own, individually correct,
3VL-aware case:

```cpp
// src/optimizer/rule/in_clause_simplification_rule.cpp:276
switch (expr.GetExpressionType()) {
case ExpressionType::COMPARE_EQUAL:
	return rewriter.ConstantOrNull(GetContext(), std::move(cmp_children[0]), Value::BOOLEAN(false));
case ExpressionType::COMPARE_NOT_DISTINCT_FROM:
	return make_uniq<BoundConstantExpression>(Value::BOOLEAN(false));
case ExpressionType::COMPARE_NOTEQUAL:
	return rewriter.ConstantOrNull(GetContext(), std::move(cmp_children[0]), Value::BOOLEAN(true));
case ExpressionType::COMPARE_DISTINCT_FROM:
	return make_uniq<BoundConstantExpression>(Value::BOOLEAN(true));
```

Note precisely which two of the four use `ConstantOrNull` and which two don't:
`=` and `<>` are `NULL`-propagating operators (an enum value that's `NULL` still
makes `enum_col = 'nonexistent-string'` evaluate to `NULL`, not `FALSE`), so those
two need the `NULL`-preserving wrapper. `IS [NOT] DISTINCT FROM` are, by
definition, `NULL`-*safe* operators — they never themselves produce `NULL` — so a
plain, unconditional `BoundConstantExpression` is correct without any wrapper at
all.

**Test walkthrough**, using the ten-value `sym_enum` again:

```sql
-- test/sql/optimizer/expression/test_enum_in_simplification.test ("Test comparisons")
EXPLAIN FROM quote WHERE sym = '0';
-- physical_plan   <!REGEX>:.*AS VARCHAR.*      -- '0' is in-domain: rewritten to integer equality
EXPLAIN FROM quote WHERE '0' = sym::VARCHAR;
-- physical_plan   <!REGEX>:.*AS VARCHAR.*      -- same, with the constant on the left
EXPLAIN FROM quote WHERE sym = '100';
-- physical_plan   <REGEX>:.*EMPTY_RESULT.*     -- '100' is not in sym_enum's domain: provably false-or-null
EXPLAIN FROM quote WHERE sym <> '100';
-- physical_plan   <REGEX>:.*IS NOT NULL.*      -- '100' not in domain: provably true-or-null -> "sym IS NOT NULL"
EXPLAIN FROM quote WHERE sym IS NOT DISTINCT FROM '100';
-- physical_plan   <REGEX>:.*EMPTY_RESULT.*     -- NULL-safe operator: unconditionally FALSE, no per-row check needed
EXPLAIN FROM quote WHERE sym IS DISTINCT FROM '0';
-- physical_plan   <!REGEX>:.*AS VARCHAR.*      -- '0' in-domain: still needs the per-row comparison
```

Every one of these lines is a direct, verifiable consequence of the four-way switch
above, plus (for the `false-or-null`/`true-or-null` cases) the same
`ConstantOrNullSimplification` fold described in §4.3.1 collapsing
`constant_or_null(...)` further once statistics prove the column is never `NULL`.

The `'0' = sym::VARCHAR` line checks that the rule also works when the constant
is on the left side. A second file,
`test/sql/optimizer/expression/enum_compare_null.test`, checks the `NULL`-safe
operators with a `NULL` constant. With a `mood` column holding `'sad'`, `'ok'` and
`NULL`, `m::VARCHAR IS NOT DISTINCT FROM NULL::VARCHAR` must count 1 row, and
`NULL::VARCHAR IS DISTINCT FROM m::VARCHAR` must count 2 rows.

**Where `IS NOT NULL` comes from.** The `sym <> '100'` line expects `IS NOT NULL` in
the plan. The rule produces `constant_or_null(TRUE, sym)`. Filter pushdown turns it
into a table filter on `sym`. Statistics propagation then sees that this filter can
only be `TRUE` or `NULL` (`FILTER_TRUE_OR_NULL`). So it swaps the filter for the
cheaper `sym IS NOT NULL` (`src/optimizer/statistics/operator/propagate_get.cpp:218`).

The swap to `IS NOT NULL` has a strict guard. The swap is correct only if the
filter is `NULL` exactly when the column is `NULL`. Filter pruning also derives
statistics for expressions inside a call, so this is not always true. For
example, turning `contains(CASE WHEN b THEN 'a' END, 'a')` into
`b IS NOT NULL` would wrongly keep the row where `b = false`. The helper
`IsNullOnlyForNullColumn` (`propagate_get.cpp:129`) allows the swap only for
these shapes: the column itself; a function with default `NULL` handling whose
arguments are the column and non-`NULL` constants; `col IN (...)` with
non-`NULL` constants; `constant_or_null` over the column; and `NOT`, `AND`,
`OR` of these. A filter with `CASE`, `COALESCE` or `nullif` is left as it is.
`test/issues/general/test_27715.test` (for DuckDB issue #27715) checks both
sides:

```sql
-- t(b BOOLEAN) holds true and false
SELECT * FROM t WHERE contains(CASE WHEN b THEN 'a' END, 'a');
-- true                     (only one row; false is not kept)
EXPLAIN SELECT * FROM t WHERE contains(CASE WHEN b THEN 'a' END, 'a');
-- physical_plan   <!REGEX>:.*IS NOT NULL.*

-- s(v VARCHAR) holds 'a', NULL, 'a'
EXPLAIN SELECT * FROM s WHERE contains(v, 'a');
-- physical_plan   <REGEX>:.*IS NOT NULL.*   (the safe shape is swapped)
```

The enum case above is the `constant_or_null` shape, so it does become
`sym IS NOT NULL`.

### 4.3.3 `EnumComparisonRule` — comparing two *different* `ENUM` types

**Purpose:** when both sides of an `=` comparison are `ENUM`-typed columns of
**different** underlying enum types (each with its own dictionary), first check
whether the two enums' domains even overlap at all — if they provably don't, the
comparison is a structural, data-independent `false-or-null`; if they might, and the
comparison sits directly at the root of a `LogicalFilter`, cast one enum value
directly into the other's type instead of routing both through `VARCHAR`.

```cpp
// src/optimizer/rule/enum_comparison.cpp:14
EnumComparisonRule::EnumComparisonRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	// match on a ComparisonExpression that is an Equality and has a VARCHAR and ENUM as its children
	auto op = make_uniq<ComparisonExpressionMatcher>();
	// Enum requires expression to be root
	op->expr_type = make_uniq<SpecificExpressionTypeMatcher>(ExpressionType::COMPARE_EQUAL);
	for (idx_t i = 0; i < 2; i++) {
		auto child = make_uniq<CastExpressionMatcher>();
		child->type = make_uniq<TypeMatcherId>(LogicalTypeId::VARCHAR);
		child->matcher = make_uniq<ExpressionMatcher>();
		child->matcher->type = make_uniq<TypeMatcherId>(LogicalTypeId::ENUM);
		op->matchers.push_back(std::move(child));
	}
	root = std::move(op);
}
```

This is the only one of the three enum rules matching **only `COMPARE_EQUAL`** —
`>` between two mismatched enums (see `test_enum_comparisons.test`'s `"String
Comparison"` case, `person.current_mood > pet.current_mood`) is deliberately left
to fall back on ordinary `VARCHAR` string comparison; only equality gets the
integer-native treatment.

`AreMatchesPossible` (`enum_comparison.cpp:29`) does the domain-overlap check —
picks the smaller of the two enums, linear-scans its members, and checks whether
*any* of them exists in the larger enum's domain via `EnumType::GetPos`:

```cpp
// src/optimizer/rule/enum_comparison.cpp:49
if (!AreMatchesPossible(BoundCastExpression::Child(left_child).GetReturnType(),
                        BoundCastExpression::Child(right_child).GetReturnType())) {
	vector<unique_ptr<Expression>> children;
	children.push_back(std::move(BoundComparisonExpression::LeftMutable(root)));
	children.push_back(std::move(BoundComparisonExpression::RightMutable(root)));
	return ExpressionRewriter::ConstantOrNull(GetContext(), std::move(children), Value::BOOLEAN(false));
}

if (!is_root || op.type != LogicalOperatorType::LOGICAL_FILTER) {
	return nullptr;
}
```

This check runs **unconditionally**, independent of position in the tree; the
second, direct-cast rewrite only fires when `is_root && op.type == LOGICAL_FILTER`
— i.e. only when the equality is the top-level predicate of a `WHERE` clause, not
when it's nested inside a projection or another expression. This is a real,
testable limitation, and `test/sql/types/enum/test_enum_comparisons.test` shows
both halves side by side:

```sql
-- test/sql/types/enum/test_enum_comparisons.test
-- mood {'sad','ok','happy'} and wealth {'poor','medium-class','rich','very super rich'}
-- share NOT ONE string -> AreMatchesPossible is false, fires everywhere, no WHERE needed:
SELECT a = b FROM test;    -- 'happy' = 'rich', 'ok' = 'poor'
-- False
-- False

-- mood and pet_mood DO overlap ('happy', 'quackity-quack') -> direct-cast rewrite,
-- but ONLY because this sits at the root of a WHERE filter:
SELECT person.name, pet.name FROM person INNER JOIN pet ON (person.current_mood = pet.current_mood)
-- (the join condition becomes a LogicalFilter predicate at bind time)

-- the SAME comparison, same two overlapping enums, in a plain SELECT list (not WHERE) -
-- is_root/LOGICAL_FILTER guard fails, so this keeps going through VARCHAR:
SELECT person_mood == pet_mood FROM person_pet_den;
-- True / True / NULL / False
```

The last query's `NULL` result (row where `pet_mood` is itself `NULL`) is the
ordinary `NULL`-propagation you'd expect from a plain `=` — nothing about this rule
changes that; it only changes *how* the (still fully `NULL`-correct) comparison
gets computed, and only in the specific tree position where it recognized the
pattern.

---

## 4.4 `DatePartSimplificationRule` and `DateTruncSimplificationRule`

### 4.4.1 `DatePartSimplificationRule` — redirect to the dedicated function

**Purpose:** when `date_part`'s specifier argument is a compile-time-constant
string, rebind the call directly to the one dedicated scalar function that
specifier corresponds to (`year(d)`, `month(d)`, `hour(d)`, ...) instead of leaving
a generic, specifier-dispatching `date_part(...)` call in the tree.

```cpp
// src/optimizer/rule/date_part_simplification.cpp:15
DatePartSimplificationRule::DatePartSimplificationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	auto func = make_uniq<FunctionExpressionMatcher>();
	func->function = make_uniq<SpecificFunctionMatcher>("date_part");
	auto specifier = make_uniq<ConstantExpressionMatcher>();
	// only the single-part overload is simplified - the struct overload takes a constant LIST of part names
	specifier->type = make_uniq<SpecificTypeMatcher>(LogicalType::VARCHAR);
	func->matchers.push_back(std::move(specifier));
	func->matchers.push_back(make_uniq<ExpressionMatcher>());
	func->policy = SetMatcher::Policy::ORDERED;
	root = std::move(func);
}
```

The rule's own comment already flags one boundary worth repeating: `date_part`
has a second overload that takes a constant *list* of part names (returning a
`STRUCT`), and this rule deliberately only matches the single-`VARCHAR`-specifier
overload — the `SpecificTypeMatcher(LogicalType::VARCHAR)` on the specifier
argument is what excludes the list form.

A `NULL` specifier folds directly to a typed `NULL` (no lookup needed); otherwise,
`GetDatePartSpecifier` maps the string to a `DatePartSpecifier` enum value, and a
17-case switch (`date_part_simplification.cpp:40`–`91`) redirects to the matching
dedicated function — `YEAR→"year"`, `QUARTER→"quarter"`, `DOW→"dayofweek"`,
`MICROSECONDS→"microsecond"`, and so on through `HOUR→"hour"`. Any specifier not in
that switch (the `default:` case) returns `nullptr` — no rewrite, the generic
`date_part(...)` call is left exactly as written.

**Which specifiers are left out, and why.** `DatePartSpecifier`
(`src/include/duckdb/common/enums/date_part_specifier.hpp:20`) has 23 non-`INVALID`
members; the rule's switch covers 17. The seven left out —
`ISOYEAR`, `ERA`, `TIMEZONE`, `TIMEZONE_HOUR`, `TIMEZONE_MINUTE`, `EPOCH`,
`JULIAN_DAY` — have no dedicated scalar function to redirect to at all (there is,
for instance, no standalone `era(...)` function registered anywhere in
`src/function/` or `src/core_functions/`), which is presumably exactly why they
were never added to the switch. `test/sql/function/date/test_date_part.test`
exercises `date_part('era', d)`, `date_part('julian', d)`, `date_part('timezone', d)`,
`date_part('timezone_hour', d)`, and `date_part('isoyear', d)` directly, purely for
functional correctness — there is no dedicated `test/optimizer/date_part_simplification.test`
verifying the rewrite at the `EXPLAIN` level the way there is for `DateTruncSimplificationRule`
(§4.4.2) — a gap this book's sourcing policy says to note rather than paper over.

**Why the rewrite earns its keep, concretely.** `test/sql/optimizer/date_part_row_group_pruning.test`
demonstrates row-group-zonemap pruning through calls like `month(d) = 5`,
`hour(ts) = 12`, and `dayofyear(d) = 130` — but every query in that file is written
using the dedicated function names directly, never `date_part('month', d)`. That's
not an oversight in this book's research — it's the whole point:
`DatePartSimplificationRule`'s rewrite is exactly what makes `date_part('month', d)`
and `month(d)` become the *same expression* in the tree, once, before any later
pass (row-group pruning included) ever has to reason about which spelling it's
looking at. A query author who happens to write `WHERE date_part('month', d) = 5`
gets the identical zonemap-pruning benefit `test/sql/optimizer/date_part_row_group_pruning.test`
demonstrates for the `month(d)` spelling — without any of the downstream passes
needing to special-case a second, generic-dispatch spelling of the same idea.

### 4.4.2 `DateTruncSimplificationRule` — turning a truncated equality into a range

**Purpose:** rewrite `date_trunc(part, column) OP constant` (for every comparison
operator: `=`, `<>`, `<`, `<=`, `>`, `>=`, `IS [NOT] DISTINCT FROM`) into a direct
comparison — or a small range of comparisons — against `column` itself, so the
`date_trunc` call never has to run per row at all, and the resulting predicate is
something a zonemap can actually use.

The core insight: `date_trunc` maps every timestamp in a period (a day, an hour, a
week, ...) down to that period's start. So `date_trunc(part, column) = constant` is
true exactly when `column` falls anywhere in `[constant, next_period_start)` —
*provided* `constant` is itself already a period start (`date_trunc(part, constant)
== constant`); if it isn't, the equality can never be satisfied by *any* value.
`DateTruncSimplificationRule::DateIsTruncated`
(`src/optimizer/rule/date_trunc_simplification.cpp:447`) is exactly this check, and
`CreateTrunc`/`CreateTruncAdd` (`date_trunc_simplification.cpp:368`, `396`) build —
and immediately constant-fold — the two boundary values the rewrite needs.

**Walkthrough 1 — a clean range operator.** `test/optimizer/date_trunc_simplification.test`
builds `test(d TIMESTAMP)` with three rows, then checks the rewrite directly via
`EXPLAIN ANALYZE`:

```sql
-- test/optimizer/date_trunc_simplification.test
EXPLAIN ANALYZE SELECT * FROM test WHERE date_trunc('day', d) < '2025-01-08';
-- analyzed_plan   <REGEX>:.*Filters[ \t\n│]*.*d < '2025.*$
```

`'2025-01-08'` (bound as midnight) is already day-truncated, so `is_truncated` is
`true`; for `<`/`>=`, the code's own comment states the rule directly: *"if
`trunc(rhs) = rhs`, then we need to just use the rhs as-is"* — `date_trunc` simply
disappears, `d < '2025-01-08'` is the entire rewrite, and the storage layer sees a
plain, zonemap-usable range predicate on `d` where it used to see an opaque
function call.

**Walkthrough 2 — equality expands into a range.** For `=` (and `IS NOT DISTINCT
FROM`), a single point isn't enough — the rewrite has to bracket the whole period:

```sql
EXPLAIN ANALYZE SELECT * FROM test WHERE date_trunc('day', d) = '2025-01-08';
-- analyzed_plan   <REGEX>:.*Filters[ \t\n│]*.*d >= '2025.*AND.*d <.*'2025.*$
```

matching the code's own comment for the `COMPARE_EQUAL` branch exactly:
`date_trunc(part, column) = constant_rhs → column >= date_trunc(part, constant_rhs)
AND column < date_trunc(part, date_add(constant_rhs, INTERVAL 1 part))`.

**Walkthrough 3 — `NULL` semantics fall out for free, with no special case needed.**
`select * from test3 where date_trunc('hour', d) = NULL;` returns **empty** — not
because the rule special-cases a `NULL` constant for plain `=`, but because it
doesn't need to: `DateIsTruncated` treats a `NULL` right-hand side as trivially
"truncated" (comment: *"the rhs is null, then the date is 'truncated' in the sense
that date_trunc(...,NULL) is also NULL"*), so the rewrite proceeds and builds
`column >= date_trunc('hour', NULL) AND column < date_trunc('hour', NULL + INTERVAL
1 HOUR)` — both boundaries fold to `NULL`, so the conjunction is `column >= NULL
AND column < NULL`, which is `NULL` for every row via completely ordinary
comparison `NULL`-propagation, and a `WHERE` clause treats a `NULL` predicate
exactly like `FALSE`. No `IsNull()` branch had to fire at all for this to come out
correct — it's an emergent consequence of building the range out of the same
`date_trunc` function the naive, per-row evaluation would have called anyway. The
one place the rule *does* special-case `NULL` explicitly is the `NULL`-safe
operators, where the generic range-building approach would be actively wrong: for
`IS NOT DISTINCT FROM NULL`, a plain `column >= NULL AND column < NULL` would
(incorrectly) exclude every row, including the one where `column IS NULL` — which
is exactly the row `IS NOT DISTINCT FROM NULL` should match. So the rule detects
this specific case up front and returns `column IS NULL` directly instead:

```sql
select * from test3 where date_trunc('hour', d) IS NOT DISTINCT FROM NULL;
-- NULL          -- only the row where d itself is NULL
select * from test3 where date_trunc('hour', d) IS DISTINCT FROM NULL;
-- 2025-01-06 03:01:00
-- 2025-01-10 05:10:06
```

**Walkthrough 4 — infinity is a fixed point, handled before any of the above.**
`date_trunc` preserves `TIMESTAMP`'s `infinity`/`-infinity` sentinels (truncating
infinity to any granularity is still infinity), so range-splitting via `+1 period`
interval arithmetic against an infinite value would be meaningless. The rule checks
for this first, for every operator uniformly, and simply drops the `date_trunc`
call with no range expansion at all:

```sql
select date_trunc('day', d) = TIMESTAMP 'infinity', ... from infinity_test order by d;
-- false  true  true  true  false  false      -- (-infinity row)
-- false  true  true  true  false  false      -- (a normal, finite row)
-- true   false false true  false true        -- (the infinity row itself)
-- NULL   NULL  NULL  NULL  NULL  NULL         -- (the NULL row)
```

**The ICU test — same rule, calendar-aware arithmetic underneath.**
`test/optimizer/date_trunc_simplification_icu.test` runs the *identical*
`DateTruncSimplificationRule` code against `TIMESTAMPTZ` values in
`America/St_Johns` (a half-hour-offset zone with a real DST transition) and against
non-whole-hour UTC offsets (`+02:10`, `-06:45`):

```sql
-- test/optimizer/date_trunc_simplification_icu.test
require icu
set TimeZone='America/St_Johns';
CREATE TABLE test2(d TIMESTAMPTZ);
INSERT INTO test2 VALUES ('2025-03-09T01:05:00'), ('2025-03-09T03:05:00'),
                         ('2025-11-02T00:30:00'), ('2025-11-02T01:30:00');

SELECT * FROM test2 WHERE date_trunc('hour', d) <= '2025-03-09T05:30:00+00'::TIMESTAMPTZ;
-- 2025-03-09 01:05:00-03:30
-- 2025-03-09 03:05:00-02:30
```

The `2025-03-09` row set straddles a DST spring-forward transition, and the second
row set (`2025-11-02`) straddles the corresponding fall-back — the ambiguous-hour
case. What's important about this test is what's conspicuously *absent* from
`DateTruncSimplificationRule` itself: there is no ICU-specific code anywhere in
`date_trunc_simplification.cpp`. `CreateTrunc` and `CreateTruncAdd` do their work by
binding and calling the ordinary scalar functions `date_trunc`, `+`, and (for
`CreateTruncAdd`) an interval-conversion function like `to_hours` — exactly the
same functions a naive, per-row, un-rewritten `date_trunc('hour', d) <= ...` would
have called. When the ICU extension is loaded, those *same* function names resolve
to ICU's calendar-aware implementations, which correctly account for the
half-hour offset and the DST transition. The optimizer rule doesn't need to know
any of that — it just needs "call whatever `date_trunc`/`+`/interval function is
bound for this type" to mean the same thing at plan time (folding the constant
boundary once) as it means at row-execution time (had the rewrite never happened).
The ICU test's entire job is to prove that delegation actually holds up once the
calendar stops being trivial — the same source file, unmodified, has to get DST and
non-hour offsets right purely because it never tried to reimplement calendar
arithmetic itself.

**Walkthrough 5 — decades, centuries and millennia before year 1.**

The rule refuses some constants for the `decade`, `century` and `millennium`
parts. The range rewrite assumes that `date_trunc` always rounds a value *down*, towards
the past. For these three parts that is not true before year 1. Truncating to a
century rounds a BC year towards zero: 150 BC becomes 100 BC, which is *later* than
150 BC. So the rule checks the constant first:

```cpp
// src/optimizer/rule/date_trunc_simplification.cpp:41
static bool TruncatesYearsTowardsZero(const BoundConstantExpression &date_part) {
	DatePartSpecifier part;
	if (!TryGetDatePartSpecifier(StringValue::Get(date_part.GetValue()), part)) {
		return true;
	}
	return part == DatePartSpecifier::DECADE || part == DatePartSpecifier::CENTURY ||
	       part == DatePartSpecifier::MILLENNIUM;
}

static bool IsAfterFirstYear(const Value &value) {
	if (value.IsNull()) {
		return true;
	}
	auto date_value = value.DefaultTryCastAs(LogicalType::DATE);
	if (!date_value || date_value->IsNull()) {
		return false;
	}
	// leave a margin for time zones
	return Date::ExtractYear(date_value->GetValue<date_t>()) > 1;
}
```

`Apply` calls both before it builds any range
(`date_trunc_simplification.cpp:100`). If the part is one of the three, and the
constant is not after year 1, the rule returns `nullptr` and `date_trunc` stays in
the plan. An unknown part name also counts as "towards zero", to be safe. The
check uses year `> 1`, not `>= 1`, to leave room for time zones.
`test/optimizer/statistics/monotonicity_annotations.test` checks it:

```sql
-- test/optimizer/statistics/monotonicity_annotations.test
CREATE TABLE b AS SELECT * FROM (VALUES (TIMESTAMP '0151-06-01 (BC)'), (TIMESTAMP '0050-01-01')) t(x);

SELECT count(*) FROM b WHERE date_trunc('century', x) >= TIMESTAMP '0101-01-01 (BC)';
-- 2
SELECT count(*) FROM b WHERE date_trunc('century', x) = TIMESTAMP '0101-01-01 (BC)';
-- 1
SELECT count(*) FROM b WHERE date_trunc('century', x::DATE) < DATE '0001-01-01';
-- 2
```

The 151 BC row truncates to 101 BC. So it matches the `=` query, even though
151 BC itself is earlier than the constant. A range rewrite (`x >= ... AND x < ...`)
would have missed it.

A separate test, `test/sql/optimizer/expression/date_trunc_non_interval_parts.test`,
covers parts that are not a plain interval, such as `'dow'`. With dates
2020-06-14, -15, -16 and `NULL`, `date_trunc('dow', d) = DATE '2020-06-15'` must
count 1 row and `date_trunc('dow', d) <= DATE '2020-06-15'` must count 2 rows.

---

## 4.5 `OrderedAggregateOptimizer` — dropping a redundant `ORDER BY`, correctly

**Purpose:** DuckDB lets an aggregate carry its own `ORDER BY` (`FIRST(i ORDER BY
i)`, `string_agg(x, ',' ORDER BY y)`, `list(x ORDER BY y)`) for aggregates whose
result depends on input order. This rule removes that `ORDER BY` in two situations
where keeping it would cost a sort for no observable benefit: when the aggregate
function itself doesn't actually care about order at all, and when the `ORDER BY`
key is provably redundant given the surrounding `GROUP BY`.

```cpp
// src/optimizer/rule/ordered_aggregate_optimizer.cpp:17
unique_ptr<Expression> OrderedAggregateOptimizer::Apply(ClientContext &context, BoundAggregateExpression &aggr,
                                                        vector<unique_ptr<Expression>> &groups,
                                                        optional_ptr<vector<GroupingSet>> grouping_sets,
                                                        bool &changes_made) {
	if (!aggr.GetOrderBys()) {
		return nullptr;
	}
	if (aggr.Function().GetOrderDependent() == AggregateOrderDependent::NOT_ORDER_DEPENDENT) {
		// not an order dependent aggregate but we have an ORDER BY clause - remove it
		aggr.GetOrderBysMutable().reset();
		changes_made = true;
		return nullptr;
	}
	// Remove unnecessary ORDER BY clauses and return if nothing remains
	if (aggr.GetOrderBysMutable()->Simplify(groups, grouping_sets)) {
		aggr.GetOrderBysMutable().reset();
		changes_made = true;
		return nullptr;
	}
	AggregateRewriteInput input(context, aggr);
	auto rewrite = TryDirectAggregateRewrite(input);
	changes_made |= rewrite != nullptr;
	return rewrite;
}
```

Two independent removal paths, plus a third mechanism worth naming briefly at the
end:

1. **The aggregate function is order-*independent*.** `MIN`, `MAX`, and `COUNT` are
   all registered with `SetOrderDependent(AggregateOrderDependent::NOT_ORDER_DEPENDENT)`
   (`src/function/aggregate/distributive/minmax.cpp:408`,
   `src/function/aggregate/distributive/count.cpp:298`/`309`) — the largest value in
   a set is the largest value regardless of the order you feed the set in, so
   `MAX(i ORDER BY i)` unconditionally drops its `ORDER BY`, no cost/group
   reasoning needed at all.
2. **The `ORDER BY` key is redundant given the grouping.** `BoundOrderModifier::Simplify`
   (`src/planner/bound_result_modifier.cpp:104`) drops any `ORDER BY` expression
   that's *also* one of the query's `GROUP BY` keys, on the reasoning stated in its
   own comment: *"`ORDER BY` on a group has no effect, because for each aggregate,
   the group is unique."* — within one output group, a `GROUP BY` key's value is,
   by construction, constant across every row that group aggregates, so ordering by
   it is a no-op. (§4.5.1 below is the story of exactly how careful this reasoning
   has to be.)

`test/optimizer/ordered_aggregate.test` exercises both, using the same-label idiom
from §4.1:

```sql
-- test/optimizer/ordered_aggregate.test
-- duplicate ORDER BY expressions are removed (this is BoundOrderModifier::Simplify's
-- de-duplication half, "we only need to ORDER BY each aggregate once")
query I nosort order_by_agg
EXPLAIN SELECT FIRST(i ORDER BY i) FROM integers
----

query I nosort order_by_agg
EXPLAIN SELECT FIRST(i ORDER BY i, i, i) FROM integers
----

-- ORDER BY on the GROUP BY key vanishes entirely
query I nosort order_by_agg_grp
EXPLAIN SELECT grp, FIRST(i ORDER BY i) FROM integers GROUP BY grp ORDER BY grp
----

query I nosort order_by_agg_grp
EXPLAIN SELECT grp, FIRST(i ORDER BY grp, i) FROM integers GROUP BY grp ORDER BY grp
----

-- order-independent aggregates drop ORDER BY unconditionally
query I nosort order_by_min
EXPLAIN SELECT MIN(i) FROM integers
----
query I nosort order_by_min
EXPLAIN SELECT MIN(i ORDER BY i) FROM integers
----
```

Each pair above asserts, via the shared label, that DuckDB really did converge two
differently-written queries onto one identical plan — `FIRST(i ORDER BY i)` and
`FIRST(i ORDER BY i, i, i)` become the same plan because the duplicate `i`
occurrences are de-duplicated by `Simplify`'s `seen_expressions` set; `MIN(i)` and
`MIN(i ORDER BY i)` become the same plan because `MIN` is order-independent.

**A third mechanism, briefly, for completeness.** When an `ORDER BY` survives both
checks above (it's genuinely needed and not redundant), `TryDirectAggregateRewrite`
(`src/optimizer/aggregate_rewrite.cpp:53`) gets one more chance: some order-sensitive
aggregates register a `DirectRewriteCallback` that can replace the whole
`agg(x ORDER BY y)` pattern with a single-pass `arg_min`/`arg_max`-family
computation instead of a full sort. `FIRST`/`LAST` are registered this way
(`SetDirectRewriteCallback(RewriteOrderedFirst<...>)`,
`src/function/aggregate/distributive/first_last_any.cpp:486`, `511`): the callback
synthesizes a `create_sort_key(y, order_modifier)` expression and rebinds the whole
thing to `arg_min`/`arg_max`/`arg_min_null`/`arg_max_null` over that key — turning
what would otherwise require materializing and sorting every group's inputs into a
single streaming min/max-tracking pass. This lives in a different file
(`aggregate_rewrite.cpp`) from the removal logic above, but it's reached from the
tail of this same rule's `Apply`, so it's worth knowing it's there even though the
full mechanism (and its sibling, the much larger `MultiStageAggregateRewriter` used
for `MODE`/frequency-style aggregates) belongs to a different deep dive.

### 4.5.1 A real regression: removing an `ORDER BY` that a `ROLLUP` still needed

`test/optimizer/expression_rewriter/ordered_aggregate_incorrectly_remove_order.test`
is exactly what its filename promises: a guard against a bug this optimizer once
shipped, where "the `ORDER BY` key matches a `GROUP BY` key, so it's redundant"
turned out to be **only sometimes true**, and the old code didn't check which
"sometimes."

```sql
-- test/optimizer/expression_rewriter/ordered_aggregate_incorrectly_remove_order.test
CREATE TABLE t1(col1 INT, col2 INT);
INSERT INTO t1 VALUES (2, 1), (1, 2);

SELECT GROUP_CONCAT(col1 ORDER BY col1) AS c1
FROM t1
GROUP BY ROLLUP (col1)
ORDER BY c1;
----
1
1,2
2
```

`GROUP BY ROLLUP (col1)` expands into **two grouping sets**: `{col1}` (one output
row per distinct `col1` value) and `{}` (one grand-total row, spanning every input
row regardless of `col1`). For the `{col1}` grouping sets, `col1 = 1` produces
exactly one row `(1, 2)`, so `GROUP_CONCAT(col1 ORDER BY col1)` over that single-row
group is trivially `'1'`; `col1 = 2` similarly gives `'2'`. But the `{}` grand-total
row aggregates **both** input rows, `(2, 1)` and `(1, 2)`, and — because `col1` is
*not* part of that particular grouping set's key at all — its value genuinely
varies across the rows this one output group is built from. The explicit
`ORDER BY col1` inside `GROUP_CONCAT` is not redundant for this row; it's the only
thing making the answer `'1,2'` (sorted) rather than whatever order the rows happen
to arrive at the hash aggregator in.

Here is exactly what `BoundOrderModifier::Simplify` looked like before the fix
(commit `232010d101`, "Fix optimizer incorrectly remove ORDER BY clause from
aggregates"):

```diff
-bool BoundOrderModifier::Simplify(vector<BoundOrderByNode> &orders, const vector<unique_ptr<Expression>> &groups) {
+bool BoundOrderModifier::Simplify(vector<BoundOrderByNode> &orders, const vector<unique_ptr<Expression>> &groups,
+                                  optional_ptr<vector<GroupingSet>> grouping_sets) {
 	// for each ORDER BY - check if it is actually necessary
 	// expressions that are in the groups do not need to be ORDERED BY
 	// `ORDER BY` on a group has no effect, because for each aggregate, the group is unique
+	expression_map_t<idx_t> group_expressions;
 	expression_set_t seen_expressions;
+	idx_t i = 0;
 	for (auto &target : groups) {
-		seen_expressions.insert(*target);
+		group_expressions.insert({*target, i++});
 	}
 	vector<BoundOrderByNode> new_order_nodes;
 	for (auto &order_node : orders) {
 		if (seen_expressions.find(*order_node.expression) != seen_expressions.end()) {
 			continue;
 		}
+		auto it = group_expressions.find(*order_node.expression);
+		bool add_to_new_order = it == group_expressions.end();
+		if (!add_to_new_order && grouping_sets) {
+			idx_t group_idx = it->second;
+			for (auto &grouping_set : *grouping_sets) {
+				if (grouping_set.find(group_idx) == grouping_set.end()) {
+					add_to_new_order = true;
+					break;
+				}
+			}
+		}
 		seen_expressions.insert(*order_node.expression);
-		new_order_nodes.push_back(std::move(order_node));
+		if (add_to_new_order) {
+			new_order_nodes.push_back(std::move(order_node));
+		}
 	}
```

The bug is exactly as blunt as it looks: the old signature didn't even *take* a
`grouping_sets` parameter. It built `seen_expressions` from every `GROUP BY` key,
unconditionally, and stripped **any** `ORDER BY` expression that textually matched
one of them — with no notion that a `ROLLUP`/`CUBE`/explicit `GROUPING SETS` query
produces multiple grouping sets, some of which don't actually include a given key at
all. For `GROUP BY ROLLUP (col1)`, `col1` is unconditionally "in the groups," so the
old code stripped `ORDER BY col1` from `GROUP_CONCAT` for *every* output row this
plan would ever produce — including the `{}` grand-total row, where the premise
("the group is unique") is false, because that row's group spans every input row
regardless of `col1`. Whatever order the hash aggregator happened to process rows
in would leak straight into the grand total's `GROUP_CONCAT` output — a real,
silent, data-dependent wrongness (unlike the `InClauseSimplificationRule` bug in
§4.2.2, this one wouldn't even be consistently wrong the same way twice, since hash
iteration order isn't a stable, predictable thing to depend on).

The fix (the version now in the source, shown in full in §4.5) makes the reasoning
grouping-set-aware: it still looks up whether an `ORDER BY` expression matches a
`GROUP BY` key (`group_expressions`), but *only* trusts "this key makes the group
unique" when a `grouping_sets` list is present **and** every one of its grouping
sets actually includes that key's index. The moment even one grouping set omits the
key (`ROLLUP (col1)`'s `{}` set omitting `col1`), `add_to_new_order` flips back to
`true` and the `ORDER BY` node survives into the final plan for every row that
query could ever produce — correctly, since the *same* `BoundAggregateExpression`
node is shared across all of a `ROLLUP`'s grouping sets; there's no per-grouping-set
copy to selectively strip the `ORDER BY` from only where it's safe.

The lesson this regression test preserves is a specific, narrow, easy-to-restate
one: "this `ORDER BY` key equals a `GROUP BY` key" is not, by itself, enough to
prove the key makes a group a singleton — that inference is only sound for the
grouping sets that actually include the key, and `ROLLUP`/`CUBE`/`GROUPING SETS`
exist specifically to produce plans where more than one grouping set applies to the
very same aggregate expression.

---

## 4.6 `DistinctAggregateOptimizer` and `DistinctWindowedOptimizer`

**Purpose, precisely:** clear the `DISTINCT` flag on an aggregate (or windowed
aggregate) function when that function's result provably does not depend on
whether its input was de-duplicated first — nothing more.

This is worth stating as narrowly as the source actually implements it, because the
name invites a much bigger guess. `agg(DISTINCT x)` genuinely is expensive in
general — conceptually, every row's `x` has to be checked against every distinct
value seen so far before it can be folded into the aggregate state — and it would
be entirely reasonable to expect a rule with this name to rewrite *how* that
deduplication happens (a separate hash-based dedup stage, a different physical
strategy, streaming fan-out). It does not. Both rules are two of the simplest in
the entire 37-rule list:

```cpp
// src/optimizer/rule/distinct_aggregate_optimizer.cpp:14
unique_ptr<Expression> DistinctAggregateOptimizer::Apply(ClientContext &context, BoundAggregateExpression &aggr,
                                                         bool &changes_made) {
	if (!aggr.IsDistinct()) {
		return nullptr;
	}
	if (aggr.Function().GetDistinctDependent() == AggregateDistinctDependent::NOT_DISTINCT_DEPENDENT) {
		// not a distinct-sensitive aggregate but we have an DISTINCT modifier - remove it
		aggr.GetAggregateTypeMutable() = AggregateType::NON_DISTINCT;
		changes_made = true;
		return nullptr;
	}
	return nullptr;
}

DistinctWindowedOptimizer::DistinctWindowedOptimizer(ExpressionRewriter &rewriter) : Rule(rewriter) {
	root = make_uniq<ExpressionMatcher>();
	root->expr_class = ExpressionClass::BOUND_WINDOW;
}

unique_ptr<Expression> DistinctWindowedOptimizer::Apply(ClientContext &context, BoundWindowExpression &wexpr,
                                                        bool &changes_made) {
	if (!wexpr.Distinct()) {
		return nullptr;
	}
	if (!wexpr.AggregateFunction()) {
		return nullptr;
	}
	if (wexpr.AggregateFunction()->GetDistinctDependent() == AggregateDistinctDependent::NOT_DISTINCT_DEPENDENT) {
		wexpr.DistinctMutable() = false;
		changes_made = true;
		return nullptr;
	}
	return nullptr;
}
```

That's the entire mechanism, twice — once for a plain `GROUP BY`/ungrouped
aggregate, once for the `OVER (...)` windowed form. `MIN`, `MAX`, and `FIRST`/`LAST`
are registered `NOT_DISTINCT_DEPENDENT`
(`src/function/aggregate/distributive/minmax.cpp:409`,
`src/function/aggregate/distributive/first_last_any.cpp:485`, `510`) precisely
because deduplicating the input first can never change their answer: the largest
value in a multiset is the largest distinct value too, and the first value a scan
encounters is, definitionally, the first *distinct* value it encounters (removing
later duplicates never changes which value arrived first). For those aggregates
specifically, `DISTINCT` is pure, avoidable overhead, and the rule removes it —
nothing about the underlying computation strategy changes for any *other*
aggregate.

`test/sql/optimizer/expression/test_indistinct_aggregates.test` is the direct,
correctly-scoped test for exactly this behavior — small, and doing exactly one
thing, twice:

```sql
-- test/sql/optimizer/expression/test_indistinct_aggregates.test
SELECT max(distinct x) from range(10) tbl(x);
-- 9

SELECT x, max(distinct x) over (order by x desc) from range(10) tbl(x);
-- 9  9
-- 8  9
-- ...
-- 0  9
```

Both queries return the answer you'd get from plain `MAX`/windowed `MAX` without
`DISTINCT` at all — which is the entire point: the rule has silently converted
`max(distinct x)` into ordinary `max(x)` (ungrouped case,
`DistinctAggregateOptimizer`) and `max(distinct x) over (...)` into ordinary
`max(x) over (...)` (windowed case, `DistinctWindowedOptimizer`) before either one
ever reaches the physical aggregate operator — no deduplication step ever runs for
either query.

### 4.6.1 What the `distinct_aggregate_rewrite*.test` files actually belong to

`test/sql/aggregate/distinct/rewrite/distinct_aggregate_rewrite.test` and
`.../distinct_aggregate_rewrite_join_order.test` are real, relevant files — but they
are **not** tests of `DistinctAggregateOptimizer`/`DistinctWindowedOptimizer`. They
belong to a much larger, separate mechanism: `MultiStageAggregateRewriter`
(`src/optimizer/multi_stage_aggregate_rewriter.cpp`), gated by the
`DISTINCT_AGGREGATE_REWRITE` toggle `00-overview.md` names in its §7.4 discussion of
pass #11 (`GROUPING_SETS`) — a toggle that, per that overview, is real and
disableable but isn't wrapped in its own `RunOptimizer` call site, so it never gets
its own numbered line in the plan-level pipeline.

The difference in scale is immediately visible from the source. Where
`distinct_aggregate_optimizer.cpp` is ~65 lines total and touches one flag,
`multi_stage_aggregate_rewriter.cpp` builds whole new plan structure — a
`DistinctAggregateSet` per group of aggregates sharing the same `DISTINCT`
argument, and a dedicated helper (`CreateDistinctBranch`,
`multi_stage_aggregate_rewriter.cpp:186`) that constructs an entire separate
aggregation branch (its own `groups`, its own dedicated dedup pass) fanned out
alongside the regular, non-distinct aggregates and reunited afterward — precisely
the kind of underlying-computation-strategy rewrite one might have guessed
`DistinctAggregateOptimizer` itself would do. The test-suite evidence for *this*
mechanism is exactly `distinct_aggregate_rewrite.test`'s own description, *"DISTINCT
aggregate logical rewrites,"* and its content confirms it: separate streaming
fan-out branches (`EXPLAIN ANALYZE ... Execution Mode: STREAMING_FANOUT ... Direct
Consumers: 2`), volatile-argument staging (`nextval(...)` evaluated exactly once per
row despite `DISTINCT`, confirmed via `currval` checks after the query), and
grouping-set-aware expansion of `DISTINCT` aggregates under `CUBE`/`ROLLUP` — all
plan-shape concerns entirely outside what a single-flag `Rule::Apply` could ever
produce.

Two `DISTINCT` aggregates share one branch only under strict rules.
`MultiStageAggregateRewriter::TryRewrite` checks three things before it puts an
aggregate into an existing `DistinctAggregateSet`
(`multi_stage_aggregate_rewriter.cpp:767`):

- **The arguments match, collation included.** `ArgumentsEqualIncludingCollation`
  (`multi_stage_aggregate_rewriter.cpp:74`) also compares the collation. A
  collation decides which strings count as duplicates. For example, under
  `NOCASE`, `'a'` and `'A'` are the same value. Plain expression equality ignores
  it.
- **The deduplication keys and the `FILTER` match.**
- **The extra `ORDER BY` keys match.** An `ORDER BY` expression that is not an
  argument becomes an extra deduplication key of the branch. If an ordered
  aggregate could join a set and add its key, that would change which rows the
  *other* aggregates in the set treat as duplicates. So `GetOrderExpressions`
  collects the keys of each aggregate, and `OrderExpressionsEqual` must say they
  are the same.

`test/sql/aggregate/distinct/rewrite/distinct_aggregate_rewrite_order_by_keys.test`
guards against that bug (DuckDB issue #27815). A `UNION(a VARCHAR, b INT)` column `u` holds `'1'` and `1`. Both
become the string `'1'`, but `ORDER BY u` still tells them apart:

```sql
SELECT g, list_sort(string_split(STRING_AGG(DISTINCT u, ','), ',')), STRING_AGG(DISTINCT u, ',' ORDER BY u)
FROM tu GROUP BY g ORDER BY g;
-- 1   [1]      1,1
-- 2   [2, x]   x,2
```

The plain `STRING_AGG(DISTINCT u, ',')` must give `[1]` for group 1. The ordered
one gives `1,1`. Each aggregate gets its own branch, so neither changes the
other.

`distinct_aggregate_rewrite.test` also checks that `sum(DISTINCT x) EXPORT_STATE`
is an error ("EXPORT_STATE is not supported for DISTINCT aggregates").

`distinct_aggregate_rewrite_join_order.test`'s own description narrows the point
further: *"DISTINCT aggregate and GROUPING SETS rewrites keep the shared input
column layout when joins below it are reordered."* — this is a test about
`MultiStageAggregateRewriter`'s interaction with `JOIN_ORDER` (Chapter 8's DPhyp
search reordering the joins feeding into a `count(DISTINCT ...)`-bearing aggregate),
not about anything expression-rewriter rules ever touch, since `JOIN_ORDER` runs
long after pass #1 in the fixed pipeline (`00-overview.md` §7.4, pass #17 versus
pass #1).

Both mechanisms are real, both are about `DISTINCT` aggregates, and both live under
the umbrella term "distinct aggregate optimization" if you're skimming file names —
but they solve genuinely different problems at genuinely different points in the
pipeline: `DistinctAggregateOptimizer`/`DistinctWindowedOptimizer` ask "does
`DISTINCT` change anything here at all?" once, cheaply, per aggregate, inside pass
#1; `MultiStageAggregateRewriter` asks "given that `DISTINCT` *does* matter here,
what's the cheapest plan shape to actually compute it?", as a much larger,
separately-toggled, later mechanism. The two `distinct_aggregate_rewrite*.test`
files are excellent, real test coverage — just not of the two rules this section is
about.

---

## 4.7 What these eight rules have in common

Step back from the individual mechanics and a single theme runs through every rule
in this file: each one notices that the SQL text asked for something more general —
and more expensive — than the specific situation actually required, and narrows it
down to the cheap special case, *without* ever letting the narrowing change the
answer. An `IN` list's column doesn't need to be cast away from its native,
zonemap-friendly type just because the SQL wrote a cast (§4.2). A dictionary-encoded
`ENUM` doesn't need string comparison just because the SQL wrote a string literal
(§4.3). `date_part`/`date_trunc` don't need a generic, specifier-dispatching call
at all once the specifier is known at compile time (§4.4). An aggregate doesn't
need its own sort, or its own deduplication pass, when the surrounding query already
guarantees the sort or the dedup was never going to change anything (§4.5, §4.6).

And in every single case, the hard part — the part that actually earned each rule
its own regression test, its own historical bug, or its own careful multi-branch
`switch` — was not finding the general shortcut. It was finding *exactly* where the
shortcut stops applying: a `TIMESTAMP` whose time-of-day isn't midnight, a `NULL`
sitting inside an `IN` list, a `ROLLUP` grouping set that doesn't include the very
key an `ORDER BY` was checked against. The eight rules in this file are, in that
sense, a fairly complete tour of what "correctness" costs a rule-based optimizer in
practice: not extra passes, not more search — just paying very close, very specific
attention to the one input shape that breaks the otherwise-obvious rewrite.
