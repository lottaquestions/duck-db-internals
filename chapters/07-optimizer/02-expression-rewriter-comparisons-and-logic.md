# 7.5 (deep dive) — The Expression Rewriter, Part 2: Comparisons and Logic

Chapter 7's overview (`00-overview.md`, §7.5) reproduced the full, fixed
37-rule list the `EXPRESSION_REWRITER` pass (#1 in the pipeline) pushes into
`ExpressionRewriter::rules`, and walked a handful of them — `ConstantFoldingRule`,
`ComparisonSimplificationRule`, `CaseSimplificationRule`, `LikeOptimizationRule`,
`InClauseSimplificationRule`, `DistinctAggregateOptimizer`, `RegexOptimizationRule`
— at the level of "here is what it does and one code excerpt." This file is
one of several deep dives that take a subset of that same 37-rule list and go
all the way down to real test cases. A companion file in this same directory
(covering `ConstantFoldingRule`, `ArithmeticSimplificationRule`,
`MoveConstantsRule`, and the other purely arithmetic/constant-shape rules)
handles the arithmetic half of the list; this file handles the ten rules that
are specifically about **comparisons and Boolean logic** — the rules where
getting SQL's three-valued (`TRUE`/`FALSE`/`NULL`) logic exactly right is not
optional polish but the entire correctness argument for the rewrite existing
at all.

That framing is worth stating bluntly up front, because it's the throughline
of everything below: **every rule in this file has to prove it doesn't change
the answer for rows where a comparison would otherwise be `NULL`.** A
simplification rule that only got the two-valued case right — ignore `NULL`,
pretend every comparison resolves to `TRUE` or `FALSE` — would be wrong for
real SQL, and DuckDB's own test suite is written specifically to catch that
class of bug: nearly every `.test` file below has at least one query built
around a `NULL` input, and several exist for no other reason than to pin down
`NULL` behavior precisely.

## A note on reading these tests: the same-label idiom

Several of the `.test` files below use a pattern worth explaining once,
carefully, since it recurs constantly in `test/optimizer/*.test`: two (or
more) `query I nosort <label>` blocks that share the same `<label>` are not
each checked against literal text written in the file — they are checked
**against each other**, byte-for-byte. sqllogictest's runner records the first
block's output under that label and then asserts every subsequent block
sharing the label produces an identical result. When the query in both blocks
is an `EXPLAIN`, this becomes a direct, mechanical proof that two
*syntactically different* SQL statements compile down to the *exact same
optimized plan* — which is precisely what you want as evidence that a
rewrite rule fired and produced the expected result, without the test author
having to hand-write out what `EXPLAIN`'s multi-line, formatting-sensitive
output looks like and hope it doesn't drift as DuckDB's `EXPLAIN` renderer
changes cosmetically over time. `test/optimizer/comparison_simplification.test`
opens with exactly this pattern:

```
# name: test/optimizer/comparison_simplification.test:12-24
query I nosort compnorm1
EXPLAIN SELECT NULL::BOOLEAN FROM test
----

query I nosort compnorm1
EXPLAIN SELECT X=NULL FROM test
----
```

Both blocks are labeled `compnorm1`, and both have an empty explicit result
section (`----` followed by nothing) — sqllogictest's convention for "I'm not
going to write out the plan text; just confirm this block's output equals the
first block's output for this label." The claim under test is that
`EXPLAIN SELECT X=NULL FROM test` produces the identical optimized plan as
`EXPLAIN SELECT NULL::BOOLEAN FROM test` — i.e., that `ComparisonSimplificationRule`
has rewritten the entire `X=NULL` expression down to a bare, untyped `NULL`
constant, indistinguishable in the final plan from having written `NULL` in
the first place. This idiom is used throughout the rest of this file without
re-explaining it every time.

---

## 1. `ComparisonSimplificationRule` — folding comparisons that don't need a row

**Purpose, in one sentence:** when one side of a comparison is a
compile-time constant, fold whatever of the comparison can be proven true,
false, or `NULL` without inspecting a single row, and — when the constant
side is under an invertible cast — shift the cast off the column and onto
the constant instead, so the column can still be compared in its native type.

The rule's matcher is intentionally broad — it fires on *any* comparison
where *some* side folds to a constant:

```cpp
// src/optimizer/rule/comparison_simplification.cpp:145
ComparisonSimplificationRule::ComparisonSimplificationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	// match on a ComparisonExpression that has a ConstantExpression as a check
	auto op = make_uniq<ComparisonExpressionMatcher>();
	op->matchers.push_back(make_uniq<FoldableConstantMatcher>());
	op->policy = SetMatcher::Policy::SOME;
	root = std::move(op);
}
```

One architectural aside worth flagging here, because it explains why every
snippet in this file casts a "comparison" to `BoundFunctionExpression`
rather than to some dedicated comparison-expression class: DuckDB's bound
tree does not have a distinct expression node type for comparisons. A
comparison like `x = 1` is represented, at the bound-expression level, as a
`BoundFunctionExpression` calling an internal comparison function — the same
node kind a call to `substring()` or `+` would use. `BoundComparisonExpression`
still exists as a name in this codebase, but it is now a small collection of
static helper functions (`::Left`, `::LeftMutable`, `::Create`, `::SetType`,
`::IsComparison`) that operate *on* a `BoundFunctionExpression`, not a
distinct class of its own — the git history (`3b0d794f1c "Convert
BoundComparisonExpression into a BoundFunctionExpression"`) confirms this was
a deliberate, later unification, not an accident of naming. Every rule in
this file that inspects "the comparison" does so through this same
`Cast<BoundFunctionExpression>()` pattern.

### Case 1 — comparisons with `NULL` fold to `NULL`, not to `FALSE`

This is the rule's most important correctness property, and it is exactly
what the `compnorm1`-labeled block above proves. Reading the `Apply` method's
relevant branch:

```cpp
// src/optimizer/rule/comparison_simplification.cpp:221
if (constant_value.IsNull() && !(expr.GetExpressionType() == ExpressionType::COMPARE_NOT_DISTINCT_FROM ||
                                 expr.GetExpressionType() == ExpressionType::COMPARE_DISTINCT_FROM)) {
	// comparison with constant NULL, return NULL
	return make_uniq<BoundConstantExpression>(Value(LogicalType::BOOLEAN));
}
```

`Value(LogicalType::BOOLEAN)` with no value argument constructs a `NULL`
`BOOLEAN`, not `Value::BOOLEAN(false)`. `X=NULL`, `X>NULL`, and `NULL>X` all
fold to this same untyped `NULL` — proven, via the same-label idiom, to
produce a plan identical to `SELECT NULL::BOOLEAN FROM test`:

```sql
-- test/optimizer/comparison_simplification.test:18-24  (label: compnorm1)
EXPLAIN SELECT X>NULL FROM test
EXPLAIN SELECT NULL>X FROM test
```

Both match `compnorm1`. Get this exactly right in your own head before going
further: **the rule does not, and must not, simplify `X = NULL` to `FALSE`.**
`NULL` in SQL is "unknown," and `X = <unknown>` is itself unknown for every
row, regardless of what `X` holds — a row where `X` happens to equal some
specific value that the `NULL` might have represented is still `NULL`, not
`TRUE`, because SQL's comparison operators never treat `NULL` as a
placeholder for "any value" that a match could be checked against; they treat
it as "the answer to whether these are equal is itself unavailable." The
`IS DISTINCT FROM` / `IS NOT DISTINCT FROM` exclusion in the `if` condition
above is exactly the boundary case that proves the rule's authors were
tracking this precisely: those two operators are *defined* to be NULL-safe
(they always return `TRUE` or `FALSE`, never `NULL`, even when one side is
`NULL`), so a constant `NULL` on one side of `IS [NOT] DISTINCT FROM` *does*
fold to a definite boolean elsewhere in the function — just not through this
particular branch, which exists only for the operators that are supposed to
propagate `NULL`.

### Case 2 — a `NULL`-valued `WHERE` clause behaves like `FALSE`, without becoming `FALSE`

The second labeled group in the same test file makes a distinct but related
point:

```sql
-- test/optimizer/comparison_simplification.test:27-33  (label: compnorm2)
EXPLAIN SELECT * FROM test WHERE 1=0
EXPLAIN SELECT * FROM test WHERE X=NULL
```

`1=0` is fully constant (no column reference at all), so it's actually
`ConstantFoldingRule` (rule #2 in the list, running before this one) that
folds it straight to `Value::BOOLEAN(false)` via a plain
`ExpressionExecutor` evaluation — not a comparison-specific rule at all.
`X=NULL`, by contrast, goes through the branch just shown and becomes a
`NULL` boolean constant, *not* `FALSE`. And yet the test asserts both
`EXPLAIN`s produce an identical optimized plan. The reconciling fact is
purely about `WHERE`'s own semantics, not about this rule: a `WHERE` clause
keeps a row only when its condition evaluates to `TRUE` — both `FALSE` and
`NULL`/unknown are rejected, identically, by the filter machinery downstream
of this rule. So a constant-`FALSE` predicate and a constant-`NULL` predicate
reach the same observable fate (the entire scan is proven to produce no
rows), even though the two are not the same *value* and this rule is careful
never to conflate them internally. This is the single cleanest illustration
in this whole file of the difference between "these two expressions denote
the same truth value" (they don't — `FALSE ≠ NULL`) and "these two
expressions are consumed identically by the specific operator sitting above
them" (they are, for `WHERE` specifically, but would not be for, say, a
`SELECT` list, where `NULL` and `FALSE` remain observably distinct: `SELECT
X=NULL` prints `NULL` for every row, `SELECT 1=0` prints `false`).

### Case 3 — shifting an invertible cast off the column and onto the constant

The other half of `Apply` handles a completely different situation: the
constant side is fine, but the *column* side is hidden inside a cast, which
would otherwise defeat a zonemap or index lookup on the column's native type.
The `constantshift`-labeled group proves this rewrite is genuinely a rewrite,
not just a coincidence of two equivalent SQL spellings:

```sql
-- test/optimizer/comparison_simplification.test:36-48  (label: constantshift)
EXPLAIN SELECT X=1::BIGINT FROM test          -- X is INTEGER; comparison forces X to BIGINT
EXPLAIN SELECT X::BIGINT=1::INTEGER FROM test -- written with the cast already explicit
EXPLAIN SELECT X='1' FROM test                -- '1' as a VARCHAR literal, implicitly cast
EXPLAIN SELECT X::BIGINT='1' FROM test
```

All four share the `constantshift` label — one optimized plan for all of
them. `X=1::BIGINT` binds with an implicit `CAST(X AS BIGINT) = 1::BIGINT`
(the binder promotes `X`, an `INTEGER` column, to the wider type so the two
sides of `=` share a type). Both sides here are integral, and the code takes
a direct shortcut for that case rather than routing through the general
invertibility check:

```cpp
// src/optimizer/rule/comparison_simplification.cpp:108-117
static bool ConstantCastIsInvertible(...) {
	if (cast_constant.IsNull() || BoundCastExpression::CastIsInvertible(cast_expression.GetReturnType(), target_type)) {
		return true;
	}
	if (cast_expression.GetReturnType().IsIntegral() && target_type.IsIntegral()) {
		return true;
	}
	// ...
```

Once invertibility is established, the rewrite casts the *constant* down to
`X`'s native `INTEGER` type (`1::BIGINT` → `1::INTEGER`, a lossless
round-trip since `1` fits) and replaces `CAST(X AS BIGINT) = 1::BIGINT` with
plain `X = 1` — the cast is gone from the column entirely, evaluated zero
times per row instead of once per row, and the comparison is now on `X`'s own
storage type where a zonemap can actually help. The final result is not the
final observable plan for `X::BIGINT=1::INTEGER` (which starts out with the
cast already explicit in the SQL) — it's the *same* plan, confirming the
rewrite is a genuine normalization, not an artifact of one particular SQL
spelling. The `'1'`-vs-`X::BIGINT='1'` half of the label does the identical
thing one syntactic layer up: the string literal `'1'` is bound with an
implicit cast to match `X`'s type, folds/round-trips the same way, and lands
on the same plan.

`ConstantCastIsInvertible` also has a case for `DECIMAL`
(`comparison_simplification.cpp:124-129`):

```cpp
if (target_type.id() == LogicalTypeId::DECIMAL && cast_expression.GetReturnType().id() == LogicalTypeId::DECIMAL) {
	// a narrowing decimal cast of the constant is fine if the constant round-trips exactly
	string error_message;
	auto roundtrip = cast_constant.TryCastAs(context, cast_expression.GetReturnType(), &error_message, true);
	return roundtrip && *roundtrip == constant_value;
}
```

Example: a column `d DECIMAL(4,1)` compared with `1.50::DECIMAL(9,2)`. The
binder widens the column to `DECIMAL(9,2)`. To move the cast onto the
constant, the constant must be cast *down* to `DECIMAL(4,1)`, which gives
`1.5`. Cast back up, that is `1.50` again, the same value. So the rule may
compare `d = 1.5` directly. A constant like `1.55` would become `1.6` and
not round-trip, so the rule leaves that comparison alone. (This example is
built from the code; no test file covers this case.)

### Case 4 — `DATE` columns compared to non-midnight `TIMESTAMP` constants

This is the case the overview (§7.5) already introduced briefly; the full
test file earns a proper walk-through here because it is the richest,
most `NULL`-careful test in this rule's coverage. A `DATE` value is always
exactly midnight when reinterpreted as an instant; a `TIMESTAMP` constant
with a nonzero time-of-day can therefore never equal a `DATE` column's value,
and every ordering comparison against it needs a day-boundary adjustment
depending on which operator and which side the column sits on. The dedicated
helper spells this out operator by operator:

```cpp
// src/optimizer/rule/comparison_simplification.cpp:15-50
static bool DateTimestampComparisonIsInvertible(...) {
	auto timestamp = constant_value.GetValue<timestamp_t>();
	if (!timestamp.IsFinite()) {
		return false;
	}
	if (Timestamp::GetTime(timestamp) == dtime_t(0)) {
		return true; // it's midnight: no replacement needed
	}
	auto op = expr.GetExpressionType();
	// Non-midnight TIMESTAMPs cannot equal DATE values.
	switch (op) {
	case ExpressionType::COMPARE_EQUAL:
		// d =  T   -> false, preserving NULL
		replacement = ExpressionRewriter::ConstantOrNull(
		    context, std::move(BoundCastExpression::ChildMutable(cast_expression)), Value::BOOLEAN(false));
		return true;
	case ExpressionType::COMPARE_NOTEQUAL:
		// d != T   -> true, preserving NULL
		replacement = ExpressionRewriter::ConstantOrNull(
		    context, std::move(BoundCastExpression::ChildMutable(cast_expression)), Value::BOOLEAN(true));
		return true;
	case ExpressionType::COMPARE_DISTINCT_FROM:
		// d IS DISTINCT FROM T     -> true
		replacement = make_uniq<BoundConstantExpression>(Value::BOOLEAN(true));
		return true;
	case ExpressionType::COMPARE_NOT_DISTINCT_FROM:
		// d IS NOT DISTINCT FROM T -> false
		replacement = make_uniq<BoundConstantExpression>(Value::BOOLEAN(false));
		return true;
	default:
		break;
	}
```

Notice the split, precisely, between `=`/`!=` (which call
`ExpressionRewriter::ConstantOrNull`) and `IS [NOT] DISTINCT FROM` (which
build a bare `BoundConstantExpression` directly). `ConstantOrNull`
(`src/optimizer/expression_rewriter.cpp:94`) binds a call to DuckDB's
built-in `constant_or_null` scalar function, which evaluates to its constant
argument when its child argument is non-`NULL`, and to `NULL` when the child
is `NULL` — it is the general mechanism this rule (and `MonotonePreimageRule`,
below) uses whenever a comparison folds to a fixed truth value *for every
non-`NULL` row*, but must still let an actual `NULL` in the column propagate
through as `NULL` rather than silently becoming that fixed value. `d = T`
(non-midnight) is unconditionally `FALSE` for every row where `d` is
non-`NULL` — but if `d` itself is `NULL`, `d = T` must still evaluate to
`NULL`, per ordinary `NULL`-propagation, exactly as it would if the rewrite
had never happened. `constant_or_null(d, FALSE)` gets both cases right in
one expression; a bare `BoundConstantExpression(false)` would get the
`NULL`-row case *wrong* — which is exactly why `IS DISTINCT FROM`/`IS NOT
DISTINCT FROM` are the only two operators allowed to skip `ConstantOrNull`
and use a bare constant: those two operators are *defined* to never return
`NULL` regardless of either side's nullability, so a bare constant is
correct for them and `ConstantOrNull` would be needless indirection.

`ConstantOrNull` binds the call with `BindBuiltinScalarExpression`, not
`BindBuiltinScalarFunction` (`expression_rewriter.cpp:97-98`). The comment
there says why: *"constant_or_null has the type of the leading constant, also
when a typed NULL child folds the call"*. If the child is already a `NULL`
constant, binding can fold the whole call to `NULL` at once, and that `NULL`
keeps the right type (here `BOOLEAN`).

The test file proves every piece of this, including the boundary-adjustment
arithmetic, with real data:

```sql
-- test/optimizer/comparison_simplification.test
CREATE TABLE date_test(d DATE);
INSERT INTO date_test VALUES ('2024-01-01'), ('2024-06-15'), ('2025-01-01');

-- midnight: cast eliminated entirely, direct DATE comparison, all six operators
EXPLAIN SELECT * FROM date_test WHERE d < timestamp '2024-06-15 00:00:00';
----
physical_plan	<REGEX>:.*d < '2024-06-15'::DATE.*
```

and, for the non-midnight case, the exact day-bump table this book's
overview only asserted the existence of:

```sql
-- non-midnight, < : floor + 1 day
EXPLAIN SELECT * FROM date_test WHERE d < timestamp '2024-06-15 12:00:00';
----
physical_plan	<REGEX>:.*d < '2024-06-16'::DATE.*

-- non-midnight, <= : floor, no adjustment
EXPLAIN SELECT * FROM date_test WHERE d <= timestamp '2024-06-15 12:00:00';
----
physical_plan	<REGEX>:.*d <= '2024-06-15'::DATE.*
```

`d < '2024-06-15 12:00:00'` needs the bump because a `DATE` value of
`2024-06-15` (midnight) genuinely *is* less than noon on the 15th — so `d <
'2024-06-16'` (the next midnight) is the correct, tight `DATE`-only
equivalent, whereas `d <= '2024-06-15 12:00:00'` doesn't need a bump because
midnight on the 15th is already `<=` noon on the 15th without any
adjustment. The rule's `Apply` mirrors exactly this reasoning, keyed on
which side the column sits and which of the four ordering operators is in
play (`comparison_simplification.cpp:54-73`).

Then, critically, the test doesn't stop at `EXPLAIN` — it re-runs every one
of these as an actual `SELECT`, including a `NULL` row, to nail down the
`NULL`-preservation claim with real values rather than just plan-shape
assertions:

```sql
INSERT INTO date_test VALUES (NULL);

query II
SELECT d, d = timestamp '2024-06-15 12:00:00' AS r FROM date_test ORDER BY d;
----
2024-01-01	false
2024-06-15	false
2025-01-01	false
NULL	NULL

query II
SELECT d, d IS DISTINCT FROM timestamp '2024-06-15 12:00:00' AS r FROM date_test ORDER BY d;
----
2024-01-01	true
2024-06-15	true
2025-01-01	true
NULL	true

query II
SELECT d, d IS NOT DISTINCT FROM timestamp '2024-06-15 12:00:00' AS r FROM date_test ORDER BY d;
----
2024-01-01	false
2024-06-15	false
2025-01-01	false
NULL	false
```

Every non-`NULL` `d` gets `FALSE` for `=` — consistent with "a `DATE` can
never equal a non-midnight `TIMESTAMP`" — but the `NULL` row gets `NULL` for
`=`, exactly as `ConstantOrNull` promises, while it gets a definite `TRUE`
for `IS DISTINCT FROM` and a definite `FALSE` for `IS NOT DISTINCT FROM`,
exactly as the NULL-safe semantics of those two operators require. This
triple — `FALSE`/`NULL`, `TRUE`/`TRUE`, `FALSE`/`FALSE` — is the entire
three-valued-logic argument for this rule's design laid out in six rows of
real query output, and it's about as clean a worked example of "get the
`NULL` case right" as this codebase offers.

The helper gives up on an *infinite* timestamp (the first three lines of the
excerpt above). `TIMESTAMP 'infinity'` has no finite date part and no
time-of-day, so the day-bump logic cannot be applied to it. The comparison
keeps its cast and runs as written. A block in the same test file checks the
results:

```sql
-- test/optimizer/comparison_simplification.test
CREATE TABLE date_infinity_test (id INTEGER, d DATE);
INSERT INTO date_infinity_test VALUES
    (1, DATE '-infinity'), (2, DATE '2000-01-01'), (3, DATE 'infinity'), (4, NULL);

SELECT id FROM date_infinity_test WHERE d < TIMESTAMP 'infinity' ORDER BY id;
-- 1, 2
SELECT id FROM date_infinity_test WHERE d >= TIMESTAMP 'infinity' ORDER BY id;
-- 3
SELECT id FROM date_infinity_test WHERE TIMESTAMP '-infinity' < d ORDER BY id;
-- 2, 3
SELECT id FROM date_infinity_test WHERE d = TIMESTAMP '-infinity' ORDER BY id;
-- 1
```

A `DATE 'infinity'` cast to `TIMESTAMP` is `TIMESTAMP 'infinity'`, so row 3
is not `< TIMESTAMP 'infinity'`, and row 1 equals `TIMESTAMP '-infinity'`.

**Limitation worth flagging:** the invertibility check
(`ConstantCastIsInvertible`) is deliberately conservative. It handles
integral-to-integral casts unconditionally, a narrow explicit
integral-to-floating case checked by round-tripping the *constant* through
the cast and comparing (`IsLosslessIntegralToFloatingCast`), a
`DECIMAL`-to-`DECIMAL` case checked the same way (see Case 3), and the `DATE`↔`TIMESTAMP` case by name — nothing
else. A comparison hidden behind a
cast this function doesn't recognize as invertible is simply left alone by
this branch (the function returns `false`, `Apply` returns `nullptr`, and
the cast stays exactly where it was written). That's a correctness-first
default: silently assuming an unrecognized cast is invertible would risk
mis-collapsing a filter, so the rule only ever acts on cases it can prove
safe.

---

## 2. `RowComparisonSimplificationRule` — decomposing tuple equality, safely

**Purpose, in one sentence:** rewrite a row-value equality like `(a, b) = (c,
d)`, when it sits as an entire filter predicate and every one of its
elements is a bare column reference of matching, non-nested type, into a
plain conjunction of per-column `IS NOT DISTINCT FROM` comparisons — a form
the rest of the optimizer (pushdown, zonemap pruning) can reason about
column-by-column instead of as one opaque call.

This rule lives in the same source file as `ComparisonSimplificationRule`
(`comparison_simplification.cpp`) and shares its list position (#12 in
§7.5's ordering, immediately after `ComparisonSimplificationRule`). Its
matcher is narrower and doesn't require a constant anywhere — it needs an
`=` between two function calls:

```cpp
// src/optimizer/rule/comparison_simplification.cpp:153
RowComparisonSimplificationRule::RowComparisonSimplificationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	auto comparison = make_uniq<ComparisonExpressionMatcher>();
	comparison->expr_type = make_uniq<SpecificExpressionTypeMatcher>(ExpressionType::COMPARE_EQUAL);
	comparison->matchers.push_back(make_uniq<ExpressionMatcher>(ExpressionClass::BOUND_FUNCTION));
	comparison->matchers.push_back(make_uniq<ExpressionMatcher>(ExpressionClass::BOUND_FUNCTION));
	root = std::move(comparison);
}
```

DuckDB's row-value syntax, `(a, b) = (c, d)`, binds to a comparison between
two calls to an internal `row` function (the same construct `ROW(a, b)`
produces explicitly) — so this matcher is really asking "is this an equality
between two struct-like constructors." `Apply` then does the real
gatekeeping: it only fires when this comparison is a top-level conjunct of a
`LogicalFilter` (`is_root && op.type == LOGICAL_FILTER`), both function calls
are actually named `"row"`, both have the same nonzero arity, and every
paired element is a bare `BoundColumnRefExpression` of the same, non-nested
return type:

```cpp
// src/optimizer/rule/comparison_simplification.cpp:161-193
if (!is_root || op.type != LogicalOperatorType::LOGICAL_FILTER) {
	return nullptr;
}
// ... left.Function().GetName() != "row" || right.Function().GetName() != "row" -> bail
// ... left_children.empty() || sizes differ -> bail
for (idx_t child_idx = 0; child_idx < left_children.size(); child_idx++) {
	// ... not BOUND_COLUMN_REF, or nested type, or mismatched type -> bail
}
auto result = make_uniq<BoundConjunctionExpression>(ExpressionType::CONJUNCTION_AND);
for (idx_t child_idx = 0; child_idx < left_children.size(); child_idx++) {
	result->GetChildrenMutable().push_back(BoundComparisonExpression::Create(
	    ExpressionType::COMPARE_NOT_DISTINCT_FROM, std::move(left_children[child_idx]),
	    std::move(right_children[child_idx])));
}
return std::move(result);
```

`test/sql/optimizer/expression/test_row_comparison_simplification.test`
walks this with data specifically shaped to hit every combination of
matching/mismatched `NULL`s in the two rows being compared:

```sql
CREATE TABLE rows(a INTEGER, b INTEGER, c INTEGER, d INTEGER);
INSERT INTO rows VALUES
	(1, 2, 1, 2),
	(1, 2, 1, 3),
	(NULL, 2, NULL, 2),
	(NULL, 2, NULL, 3),
	(NULL, 2, 1, 3),
	(1, 2, NULL, 3),
	(NULL, 2, 1, 2),
	(1, NULL, 1, NULL);

query I rowsort
SELECT (a, b) = (c, d) FROM rows;
----
0
0
0
0
0
1
1
1
```

This result is the single most surprising, and most important, `NULL`-safety
fact in this rule's test coverage: **there is no `NULL` anywhere in that
output.** DuckDB's row-value equality (`(a,b) = (c,d)`) is not standard
scalar three-valued equality applied recursively — it's already defined to
be `NULL`-safe per element, exactly the semantics `COMPARE_NOT_DISTINCT_FROM`
gives you, which is exactly why the rewrite target is `IS NOT DISTINCT FROM`
and not plain `=`. Working through row 3, `(NULL, 2, NULL, 2)` — meaning
`a=NULL, b=2, c=NULL, d=2` — the rewritten form evaluates `a IS NOT DISTINCT
FROM c` as `NULL IS NOT DISTINCT FROM NULL`, which is `TRUE` (two `NULL`s are
"not distinct" from each other), `AND`ed with `b IS NOT DISTINCT FROM d` =
`2 IS NOT DISTINCT FROM 2` = `TRUE`, giving an overall `TRUE` for that row.
Row 5, `(NULL, 2, 1, 3)` — `a=NULL` vs `c=1` — gives `NULL IS NOT DISTINCT
FROM 1` = `FALSE` (a `NULL` and a non-`NULL` value *are* distinct), so the
whole row is `FALSE` regardless of the second column. Working through all
eight rows this way reproduces the observed 5-`FALSE`/3-`TRUE` split exactly.

Two things worth being precise about here, because they're easy to get
backwards:

**This `NULL`-safe behavior is not something the rewrite rule introduces —
it's the pre-existing semantics of row-value equality, which the rewrite
merely re-expresses.** The test proves this directly by disabling the whole
`EXPRESSION_REWRITER` pass and re-running the identical query:

```sql
statement ok
SET disabled_optimizers='expression_rewriter';

query I rowsort
SELECT (a, b) = (c, d) FROM rows;
----
0
0
0
0
0
1
1
1
```

Same eight values, same 5/3 split. So the rule is a pure **performance**
rewrite, not a semantics-changing one: whatever built-in "row equality"
function DuckDB evaluates `(a,b)=(c,d)` with, unrewritten, already implements
this NULL-safe, per-column semantics. What the rewrite buys is that the
expression is no longer an opaque call the rest of the optimizer can't see
into — it's now a plain `AND` of two ordinary column-to-column comparisons,
which pushdown and per-column statistics can reason about individually.

**The rule only fires when the row comparison is the entire predicate, at
the top of a `LogicalFilter`.** This is the `is_root && op.type ==
LOGICAL_FILTER` gate. The test's own `count(*)` query makes the point that
in that specific position, the distinction between `NULL` and `FALSE`
genuinely doesn't matter for the *observable result* (though it still
matters for the rule's own internal correctness argument, since it had to
choose `IS NOT DISTINCT FROM`, not plain `=`, to get the right per-row
answers in the first place):

```sql
-- In predicate context FALSE and NULL are both rejected, so scalarization is equivalent.
query I
SELECT count(*) FROM rows WHERE (a, b) = (c, d);
----
3
```

Three rows keep — exactly the three that evaluated `TRUE` above (rows 1, 3,
and 8). Finally, the test is explicit that this rule deliberately narrows
its scope rather than trying to handle every shape of row comparison:

```sql
-- Empty and expression-valued rows retain the original comparison implementation.
query II
SELECT row() = row(), row(a + 1, b) = row(c + 1, d) FROM rows WHERE a = 1 AND b = 2 AND d = 2;
----
1	1
```

`row()` (zero elements — `left_children.empty()` bails) and
`row(a + 1, b)` (an element that's an arithmetic expression, not a bare
`BOUND_COLUMN_REF` — the per-element type-class check bails) both fall
through untouched, and both still compute the right answer via the
original, unrewritten "row equality" function — confirming, once again, that
this rule is strictly an optimization of a *subset* of cases the general
mechanism already handles correctly, not a correctness fix.

---

## 3. `NotComparisonSimplificationRule` — pushing `NOT` through a comparison

**Purpose, in one sentence:** rewrite `NOT (a > b)` into `a <= b` (and the
analogous flip for every other comparison operator), so that a negated
comparison becomes a plain comparison the filter-combiner and row-group
pruner can reason about directly, instead of an opaque `NOT` wrapper they
can't see through.

```cpp
// src/optimizer/rule/not_comparison_simplification.cpp:9-35
NotComparisonSimplificationRule::NotComparisonSimplificationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	auto op = make_uniq<ExpressionMatcher>(ExpressionClass::BOUND_OPERATOR);
	op->expr_type = make_uniq<SpecificExpressionTypeMatcher>(ExpressionType::OPERATOR_NOT);
	root = std::move(op);
}

unique_ptr<Expression> NotComparisonSimplificationRule::Apply(...) {
	auto &not_expr = bindings[0].get().Cast<BoundOperatorExpression>();
	auto &child = not_expr.GetChildrenMutable()[0];
	if (!BoundComparisonExpression::IsComparison(*child)) {
		return nullptr;
	}
	auto &comparison = child->Cast<BoundFunctionExpression>();
	auto negated_type = NegateComparisonExpression(comparison.GetExpressionType());
	BoundComparisonExpression::SetType(comparison, negated_type);
	changes_made = true;
	return std::move(child);
}
```

This is a small, purely syntactic rewrite — `NegateComparisonExpression`
maps each comparison type to its logical negation (`>` ↔ `<=`, `>=` ↔ `<`,
`=` ↔ `!=`, `IS DISTINCT FROM` ↔ `IS NOT DISTINCT FROM`) — and the code
itself never touches whether either operand is `NULL`. That's exactly why
it's correct: **flipping a comparison operator's type doesn't change how it
propagates `NULL`.** `a > b` and `a <= b` are both "plain" (non-`NULL`-safe)
comparisons — each one independently evaluates `NULL` whenever either
operand is `NULL` — so `NOT(a > b)` and `a <= b` agree not only on every
row where both are non-`NULL`, but also on the `NULL` row itself: `NOT(a >
b)` is `NOT(NULL)`, which three-valued logic defines as `NULL`, and `a <= b`
independently evaluates to `NULL` too, for the same reason (one operand is
`NULL`). Nothing about this identity depends on the two-valued
approximation of `NOT`; it holds exactly, row by row, including on `NULL`.

`test/sql/optimizer/test_not_comparison_simplification.test` (added in the
commit that introduced this rule, to fix a specific filter-pruning gap —
`NOT(a > 1) AND a > 1` wasn't being recognized as unsatisfiable until this
rule normalized the `NOT` away) walks all eight comparison operators plus
the crucial `NULL`-behavior check:

```sql
CREATE TABLE t AS SELECT * FROM (VALUES (1),(2),(3),(NULL)) v(a);

query II
EXPLAIN SELECT * FROM t WHERE NOT (a > 2);
----
logical_opt	<REGEX>:.*a <= 2.*

query II
EXPLAIN SELECT * FROM t WHERE NOT (a = 2);
----
logical_opt	<REGEX>:.*a != 2.*

query II
EXPLAIN SELECT * FROM t WHERE NOT (a IS DISTINCT FROM 2);
----
logical_opt	<REGEX>:.*a IS NOT DISTINCT FROM 2.*

query I rowsort
SELECT * FROM t WHERE NOT (a = 1) ORDER BY a
----
2
3
```

That final query is the exact test of the point above, stated in the task
brief and worth being pedantic about: `t` has rows `1, 2, 3, NULL`. `WHERE
NOT (a = 1)` returns `2` and `3` — **and not the `NULL` row.** If the rule
(or a careless reader) treated `NOT(a = 1)` as "definitely `a != 1`, which is
`TRUE` for `NULL`" — reasoning by analogy to "a `NULL` value certainly isn't
equal to `1`, so surely it's *not equal*" — you'd expect the `NULL` row to
pass the filter. It does not, because that reasoning is wrong: `a = 1` for
the `NULL` row is `NULL` (unknown, not `FALSE`), so `NOT(a = 1)` is
`NOT(NULL)` = `NULL`, and a `WHERE` clause rejects `NULL` results exactly as
it rejects `FALSE` ones. The rewritten form, `a != 1`, gives the identical
answer for the identical reason: `NULL != 1` is *also* `NULL`, not `TRUE` —
so both the original and the rewritten expression exclude the `NULL` row,
for the same underlying three-valued-logic reason, and the rewrite changes
nothing observable. This is precisely the distinction the task brief calls
out by name: `NOT (x = NULL)` does not become "`x != NULL` evaluates to
`TRUE`" — both sides of that identity are `NULL`, full stop, and this rule's
entire correctness rests on that being true for every comparison operator it
touches.

**Guard worth noting:** the matcher only requires `OPERATOR_NOT` at the
root; `Apply` itself is what actually restricts the rewrite to comparisons
(`BoundComparisonExpression::IsComparison(*child)`), returning `nullptr` for
`NOT` wrapping anything else (a column reference, a conjunction, a `CASE`).
`NOT`-wrapped conjunctions are a different rule entirely — the next one.

---

## 4. `ConjunctionSimplificationRule` and `NotConjunctionSimplificationRule`

These two rules are adjacent in the rule list (#8 and #37 respectively) but
solve genuinely different problems: one folds a conjunction when one of its
own children is a foldable constant; the other pushes a `NOT` through an
entire conjunction via De Morgan's laws. Both have to be exactly right about
`NULL`, and — this is the more interesting fact — De Morgan's laws
themselves turn out to hold *exactly*, not just approximately, under SQL's
three-valued logic. It's the second rule's *side-effect* handling, not its
Boolean-value correctness, that needs a careful guard.

### 4a. `ConjunctionSimplificationRule` — dropping `TRUE`/`FALSE` operands

**Purpose:** simplify an `AND` or `OR` whose expression list contains a
compile-time-constant operand — drop a `TRUE` from an `AND`, collapse the
whole `AND` to `FALSE` if any operand is `FALSE`; drop a `FALSE` from an
`OR`, collapse the whole `OR` to `TRUE` if any operand is `TRUE`.

```cpp
// src/optimizer/rule/conjunction_simplification.cpp:34-75
unique_ptr<Expression> ConjunctionSimplificationRule::Apply(...) {
	auto &conjunction = bindings[0].get().Cast<BoundConjunctionExpression>();
	auto &constant_expr = bindings[1].get();
	Value constant_value;
	if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), constant_expr, constant_value)) {
		return nullptr;
	}
	constant_value = constant_value.DefaultCastAs(LogicalType::BOOLEAN);
	if (constant_value.IsNull()) {
		// FALSE and NULL have the same effect in filters and joins
		if (conjunction.GetExpressionType() != ExpressionType::CONJUNCTION_OR || !is_root ||
		    (op.type != LogicalOperatorType::LOGICAL_FILTER && op.type != LogicalOperatorType::LOGICAL_ANY_JOIN)) {
			return nullptr;
		}
		changes_made = true;
		return RemoveExpression(conjunction, constant_expr);
	}
	if (conjunction.GetExpressionType() == ExpressionType::CONJUNCTION_AND) {
		if (!BooleanValue::Get(constant_value)) {
			return make_uniq<BoundConstantExpression>(Value::BOOLEAN(false));
		} else {
			return RemoveExpression(conjunction, constant_expr);
		}
	} else {
		D_ASSERT(conjunction.GetExpressionType() == ExpressionType::CONJUNCTION_OR);
		if (!BooleanValue::Get(constant_value)) {
			return RemoveExpression(conjunction, constant_expr);
		} else {
			return make_uniq<BoundConstantExpression>(Value::BOOLEAN(true));
		}
	}
}
```

This has exactly the same `WHERE`-clause-specific carve-out as
`ComparisonSimplificationRule`'s `compnorm2` case, spelled out explicitly in
the comment: *"FALSE and NULL have the same effect in filters and joins."*
The condition it guards is narrow on purpose — it only drops a constant
`NULL` operand out of an `OR` (never an `AND`, and never outside a filter or
join-predicate root), because that's the one situation where dropping it is
provably safe: an `OR` with a `NULL` operand and at least one other operand
is only ever going to be kept by the surrounding `WHERE`/`JOIN ... ON` if
some *other* operand makes the whole `OR` `TRUE` (a `NULL` and any number of
`FALSE`s is still `NULL`/rejected either way), so a `NULL` operand can be
dropped from an `OR` used as a filter root without changing which rows
survive. The same substitution would *not* be safe inside a `SELECT` list —
`(x OR NULL)` and `x` are not the same *value* when `x` is `FALSE` (`FALSE OR
NULL` is `NULL`; plain `x` there would print `false`) — which is exactly why
the guard checks `is_root` and the specific operator type before touching
anything.

The dedicated test, `test/optimizer/conjunction_simplification.test`, is
compact but exercises the constant-`TRUE`/`FALSE` half cleanly with the
same-label idiom:

```sql
CREATE TABLE test(X BOOLEAN);

-- X AND TRUE => X
query I nosort conjnorm1
EXPLAIN SELECT X AND TRUE FROM test
----

query I nosort conjnorm1
EXPLAIN SELECT X FROM test
----

-- X OR FALSE => X
query I nosort conjnorm1
EXPLAIN SELECT X OR FALSE FROM test
----

query I nosort conjnorm1
EXPLAIN SELECT X FROM test
----
```

All four share `conjnorm1` — `X AND TRUE`, `X OR FALSE`, and bare `X` all
compile to the identical plan. This is worth being careful about too: `X AND
TRUE` and `X` agree for every value of `X` including `NULL` (`NULL AND TRUE =
NULL`), and `X OR FALSE` and `X` likewise agree including for `NULL` (`NULL
OR FALSE = NULL`) — so this simplification, unlike the `OR`-with-`NULL`-operand
case above, is a genuine value-level identity that holds everywhere, not
just under a filter-root carve-out; the code path that actually executes for
`X AND TRUE` is `RemoveExpression`'s generic "erase the constant operand,
and if only one operand remains, return it directly" logic, which applies
uniformly regardless of context.

The broader `NULL`-behavior of `AND`/`OR` themselves — not this rule's
rewriting of them, but the operators' own three-valued truth tables — is
exercised more directly by `test/sql/optimizer/expression/test_conjunction_optimization.test`,
which is really evidence for the sibling `PredicateFactoringRule` (§7) but
is worth citing here for its `NULL`-focused assertions:

```sql
INSERT INTO integers VALUES (1), (2), (3), (NULL)

query I
SELECT i FROM integers WHERE (i IS NULL AND i=1) OR (i IS NULL AND i<10) ORDER BY i
----
```

Zero rows — even though `i IS NULL` is `TRUE` for the `NULL` row, `i=1` and
`i<10` are each `NULL` for that same row (comparing `NULL` to a constant), so
`TRUE AND NULL` is `NULL` for both disjuncts, and `NULL OR NULL` is `NULL` —
rejected by the `WHERE` clause. Change the inner comparisons to `IS [NOT]
NULL` tests instead, and the row does show up:

```sql
query I
SELECT i FROM integers WHERE (i IS NULL AND (i+1) IS NULL) OR (i IS NULL AND (i+2) IS NULL) ORDER BY i
----
NULL
```

Because `(i+1) IS NULL` and `(i+2) IS NULL` (unlike `i=1` and `i<10`) are
themselves `NULL`-safe — `IS NULL` never returns `NULL` — so `TRUE AND TRUE`
genuinely evaluates to `TRUE` for the `NULL` row this time, and the row
passes.

### 4b. `NotConjunctionSimplificationRule` — De Morgan, and why volatility still matters

**Purpose:** rewrite `NOT (a AND b)` into `(NOT a) OR (NOT b)`, and `NOT (a
OR b)` into `(NOT a) AND (NOT b)` — De Morgan's laws — so a negated
conjunction (which the pushdown/pruning machinery can't see into) becomes a
disjunction or conjunction of individually-negated, individually-sargable
predicates.

```cpp
// src/optimizer/rule/not_conjunction_simplification.cpp:14-46
unique_ptr<Expression> NotConjunctionSimplificationRule::Apply(...) {
	auto &not_expr = bindings[0].get().Cast<BoundOperatorExpression>();
	auto &child = not_expr.GetChildrenMutable()[0];
	if (child->GetExpressionClass() != ExpressionClass::BOUND_CONJUNCTION) {
		return nullptr;
	}
	auto &conjunction = child->Cast<BoundConjunctionExpression>();
	if (conjunction.IsVolatile() || conjunction.CanThrow()) {
		return nullptr;
	}
	auto negated_type = conjunction.GetExpressionType() == ExpressionType::CONJUNCTION_AND
	                        ? ExpressionType::CONJUNCTION_OR
	                        : ExpressionType::CONJUNCTION_AND;
	auto result = make_uniq<BoundConjunctionExpression>(negated_type);
	for (auto &conj_child : conjunction.GetChildrenMutable()) {
		auto negated_child = make_uniq<BoundOperatorExpression>(ExpressionType::OPERATOR_NOT, LogicalType::BOOLEAN);
		negated_child->GetChildrenMutable().push_back(std::move(conj_child));
		result->GetChildrenMutable().push_back(std::move(negated_child));
	}
	changes_made = true;
	return std::move(result);
}
```

First, the good news on three-valued logic: **De Morgan's laws hold exactly
under SQL's Kleene-style three-valued logic, with no caveats needed for
`NULL` at all.** This is worth actually checking rather than assuming, since
so much else in this file has turned out to need a `NULL`-specific carve-out.
Take `NOT(a AND b)` with `a = NULL`, `b = FALSE`: `NULL AND FALSE` evaluates
to `FALSE` under three-valued `AND` (a `FALSE` operand always wins,
regardless of the other operand's unknown-ness), so `NOT(FALSE) = TRUE`. The
right-hand side, `(NOT a) OR (NOT b)`, is `NOT(NULL) OR NOT(FALSE)` = `NULL
OR TRUE`, and a `TRUE` operand always wins in three-valued `OR` regardless of
the other operand — so this is `TRUE` too. They agree. Try `a = NULL, b =
TRUE`: `NULL AND TRUE = NULL` (no operand is `FALSE`, so the unknown
propagates), `NOT(NULL) = NULL`. Right-hand side: `NOT(NULL) OR NOT(TRUE) =
NULL OR FALSE = NULL` (no operand is `TRUE`, so the unknown propagates
again). They still agree. `test/sql/optimizer/not_conjunction_simplification.test`
verifies exactly this with a truth table over all four `NULL`-inclusive
combinations, run directly (not through `EXPLAIN`, so this is checking
actual computed values, not just that a rewrite fired):

```sql
query II
SELECT NOT (a AND b), (NOT a) OR (NOT b)
FROM (VALUES (1, NULL::BOOLEAN, TRUE),
             (2, NULL::BOOLEAN, FALSE),
             (3, TRUE, NULL::BOOLEAN),
             (4, FALSE, NULL::BOOLEAN)) v(id, a, b)
ORDER BY id;
----
NULL	NULL
true	true
NULL	NULL
true	true
```

Every row matches between the two columns, for all four `NULL`-involving
combinations tried. So the *value*-level identity behind this rule needs no
special-casing at all — which makes the `IsVolatile() || CanThrow()` guard
in the code above look, at first glance, unmotivated: if the rewrite is a
value-preserving identity regardless of `NULL`, why refuse to apply it when
a child happens to call `random()` or might throw?

The answer isn't about the *value* the expression computes — it's about
*how many times, and under what circumstances, its children actually get
evaluated*, and that turns out to depend on exactly where in the tree the
conjunction sits, because of how DuckDB's vectorized expression executor
evaluates conjunctions. `ExpressionExecutor::Execute(const
BoundConjunctionExpression&, ...)` (`src/execution/expression_executor/execute_conjunction.cpp:33`)
— used whenever a conjunction is evaluated as an ordinary sub-expression,
e.g. as the direct child of a `NOT` — evaluates **every child, for every
row, unconditionally**, then combines the resulting boolean vectors with
`VectorOperations::And`/`Or`. But `ExpressionExecutor::Select(const
BoundConjunctionExpression&, ...)` (same file, line 83) — used when a
conjunction is itself the *direct* predicate a filter operator is applying —
is a genuinely different code path: it maintains a shrinking selection
vector across the conjunction's children (in an order chosen adaptively at
runtime by an `AdaptiveFilter`, based on which children have proven cheapest
and most selective in earlier batches — the same idea as the plan-level
`REORDER_FILTER` pass from §7.6.1, but re-decided per batch at execution
time rather than fixed once at plan time), and it **stops evaluating later
children for any row already decided** by an earlier one — for `AND`, a row
that already failed an earlier conjunct is dropped from `current_count`
before the next conjunct's `Select` is even called on it; for `OR`,
symmetrically, a row that's already succeeded is moved out before the next
conjunct runs.

This is exactly what the test's own two side-effect cases are checking:

```sql
CREATE SEQUENCE s;
CREATE TABLE volatile_t(a BOOLEAN);
INSERT INTO volatile_t VALUES (false), (true);

query I
SELECT count(*) FROM volatile_t WHERE NOT (a AND nextval('s') > 0);
----
1

query I
SELECT currval('s');
----
2
```

The filter's root expression here is `NOT(...)`, not the `AND` itself — so
it's evaluated via `Execute`, which runs `nextval('s')` for *both* rows of
`volatile_t` unconditionally (hence `currval('s')` reporting `2` afterward:
the sequence advanced exactly once per row, twice total). Had this rule
fired and rewritten the filter to `(NOT a) OR (NOT (nextval('s') > 0))`, the
new root expression would itself be a `BoundConjunctionExpression` (an
`OR`), which — sitting directly as the filter's predicate — would be
evaluated via `Select`, not `Execute`: for the `a = false` row, `NOT a =
TRUE` already decides that row's result, and `Select`'s short-circuiting
would skip evaluating `NOT(nextval('s') > 0)` for it entirely. The sequence
would then only advance once (for the `a = true` row), and `currval('s')`
would report `1`, not `2` — a different number of side effects than the
un-rewritten query produces. The guard exists precisely to prevent this
divergence.

The error-visibility test makes the same point even more sharply:

```sql
CREATE TABLE throwing_t(a BOOLEAN);
INSERT INTO throwing_t VALUES (false);

statement error
SELECT count(*) FROM throwing_t WHERE NOT (a AND error('expected'));
----
expected
```

Un-rewritten, `NOT(...)` is the filter root, evaluated via `Execute`, which
runs *both* children of the inner `AND` unconditionally for the one row —
`error('expected')` fires regardless of `a`'s value, and the query throws,
which is exactly the observed, asserted behavior. If the rewrite had fired,
the new root `(NOT a) OR (NOT error('expected'))` would be a top-level `OR`
evaluated via `Select`: for the single row, `NOT a = NOT FALSE = TRUE`
already decides the `OR`'s result, and `Select`'s short-circuiting would
never call `Select` on the second child at all — `error()` would never run,
and the query would silently return `count(*) = 1` instead of throwing. The
`IsVolatile() || CanThrow()` guard is what stands between "this query throws,
as the user should be able to rely on" and "this query silently changes
behavior depending on which unrelated optimizer pass happened to run" — and
it's checked on the *whole* conjunction (recursively, since `IsVolatile`/
`CanThrow` walk the entire subtree), so a volatile or throwing expression
buried arbitrarily deep inside either conjunct still blocks the rewrite at
the top.

The test file also demonstrates the actual payoff this rewrite exists for —
turning an opaque `NOT`-wrapped range or disjunction into something the
storage layer's row-group zonemaps can prune against:

```sql
CREATE TABLE t AS SELECT range AS i FROM range(1000000);
CHECKPOINT;

-- NOT BETWEEN is bound to a NOT-wrapped conjunction
query II
EXPLAIN ANALYZE SELECT count() FROM t WHERE i NOT BETWEEN 10 AND 999989;
----
analyzed_plan	<REGEX>:.*Row Groups Scanned: [12] / 9.*

-- NOT(OR) becomes two prunable predicates
query II
EXPLAIN ANALYZE SELECT count() FROM t WHERE NOT (i < 400000 OR i > 600000);
----
analyzed_plan	<REGEX>:.*Row Groups Scanned: [1-8] / 9.*
```

`i NOT BETWEEN 10 AND 999989` binds to `NOT(i >= 10 AND i <= 999989)` — a
conjunction of two plain, non-volatile, non-throwing range comparisons, so
the guard doesn't block it, and De Morgan turns it into `i < 10 OR i >
999989`, a disjunction of two anti-range predicates each of which is now
directly sargable against `i`'s zonemap — only 1–2 of the table's 9 row
groups need to be scanned at all, instead of all 9 behind an opaque `NOT`.

---

## 5. `CaseSimplificationRule` — folding `CASE` branches with foldable conditions

**Purpose:** fold any `WHEN` branch of a `CASE` expression whose condition is
itself a compile-time-constant expression, discarding branches that are
proven unreachable and short-circuiting to a branch proven always taken.

```cpp
// src/optimizer/rule/case_simplification.cpp:14-50
unique_ptr<Expression> CaseSimplificationRule::Apply(...) {
	auto &root = bindings[0].get().Cast<BoundCaseExpression>();
	for (idx_t i = 0; i < root.CaseChecksMutable().size(); i++) {
		auto &case_check = root.CaseChecksMutable()[i];
		if (case_check.when_expr->IsFoldable()) {
			// ...
			Value condition;
			if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), *case_check.when_expr, condition) ||
			    !condition.DefaultTryCastAs(LogicalType::BOOLEAN)) {
				// the condition cannot be evaluated: leave it to execution
				continue;
			}

			// fold based on the constant condition
			if (condition.IsNull() || !BooleanValue::Get(condition)) {
				// the condition is always false: remove this case check
				root.CaseChecksMutable().erase_at(i);
				i--;
			} else {
				// the condition is always true
				root.ElseMutable() = std::move(case_check.then_expr);
				root.CaseChecksMutable().erase(root.CaseChecksMutable().begin() + NumericCast<int64_t>(i),
				                               root.CaseChecksMutable().end());
				break;
			}
		}
	}
	if (root.CaseChecksMutable().empty()) {
		return Expression::PreserveReturnType(root.GetReturnType(), std::move(root.ElseMutable()));
	}
	return nullptr;
}
```

The line that matters most for `NULL`-correctness is short and easy to
misread: `if (condition.IsNull() || !BooleanValue::Get(condition))`. A
`WHEN` condition that folds to `NULL` is treated **exactly like** a `WHEN`
condition that folds to `FALSE` — both fall into the "always false: remove
this branch" arm. That's exactly right for `CASE`'s own semantics: SQL's
`CASE WHEN cond THEN ... END` takes a branch only when `cond` is `TRUE`; a
`cond` that's `NULL` or `FALSE` both cause that branch to be skipped in
favor of the next `WHEN` (or the final `ELSE`), identically. So folding
a `NULL` `WHEN` condition away is not a NULL-to-FALSE conflation bug the way
it would be inside a comparison rewrite — it's the correct behavior for
`CASE`'s own branch-selection rule, which was never three-valued to begin
with in the sense that matters here (it's "does this branch fire, yes or
no," and `NULL` answers "no" just as `FALSE` does).

The rule uses `TryEvaluateScalar` and `DefaultTryCastAs`, which do not
throw. If a foldable `WHEN` condition raises an error, or its value cannot be
cast to `BOOLEAN`, the rule skips that branch (`continue`) and leaves it for
execution time. Why this matters: if the rule threw, the error would be
raised while *planning*, even for a query that never evaluates the branch.
The test `test/optimizer/constant_argument_errors_case.test` shows both
sides:

```sql
-- test/optimizer/constant_argument_errors_case.test
CREATE TABLE t(s VARCHAR);

statement ok
SELECT CASE WHEN ''::DOUBLE > 1 THEN 1 ELSE 2 END FROM t;

statement error
SELECT CASE WHEN ''::DOUBLE > 1 THEN 1 ELSE 2 END;
----
<REGEX>:Conversion Error.*
```

Table `t` is empty, so the `CASE` is never evaluated, and the first query
succeeds. The second query has no table, so it produces one row; the `CASE`
runs, and the cast error is raised at execution time.

`test/optimizer/case_simplification.test` demonstrates all three outcomes —
condition always true, always false, and always `NULL` — with the same-label
idiom, and the third case is exactly the `NULL`-condition scenario:

```sql
CREATE TABLE test(X INTEGER);

query I nosort casenorm1
EXPLAIN SELECT CASE WHEN 1=1 THEN X+1 ELSE X+2 END FROM test
----

query I nosort casenorm1
EXPLAIN SELECT X+1 FROM test
----

query I nosort casenorm2
EXPLAIN SELECT CASE WHEN 1=0 THEN X+1 ELSE X+2 END FROM test
----

query I nosort casenorm2
EXPLAIN SELECT X+2 FROM test
----

query I nosort casenorm3
EXPLAIN SELECT CASE WHEN NULL>3 THEN X+1 ELSE X+2 END FROM test
----

query I nosort casenorm3
EXPLAIN SELECT X+2 FROM test
----
```

`NULL>3` folds (via `ComparisonSimplificationRule`, upstream in the same
rule list) to a constant `NULL`; `CaseSimplificationRule` then treats that
`NULL` `WHEN` condition exactly like the `1=0` case in `casenorm2` — both
compile to the bare `ELSE` branch, `X+2`. This is a good illustration of how
these 37 rules cooperate within one pass: `ComparisonSimplificationRule`
(rule #11) folds `NULL>3` to a bare `NULL` constant first; `CaseSimplificationRule`
(rule #7, earlier in the *list* but — per the overview's §7.5 closing note —
the inner expression-rewrite loop re-visits newly-produced subexpressions,
so ordering within the fixed list doesn't prevent a later rule's output from
being picked up by an earlier one at the same tree position on a later
visit) then folds the whole `CASE` down to its `ELSE`.

**Limitation worth noting:** the rule only folds branches whose `WHEN`
condition `IsFoldable()` — a condition that depends on any column is left
completely untouched, and the rule makes no attempt to prove anything about
*which* branches are reachable when the condition is data-dependent (that's
a job, if it happens at all, for `STATISTICS_PROPAGATION`, not this rule).

---

## 6. `EqualOrNullSimplification` — recognizing a hand-written `IS NOT DISTINCT FROM`

**Purpose:** recognize the specific pattern `a = b OR (a IS NULL AND b IS
NULL)` — a common, verbose, hand-written way to express "these are equal, or
both `NULL`" — and rewrite it directly to `a IS NOT DISTINCT FROM b`, DuckDB's
built-in `NULL`-safe equality operator.

```cpp
// src/optimizer/rule/equal_or_null_simplification.cpp:23-51
EqualOrNullSimplification::EqualOrNullSimplification(ExpressionRewriter &rewriter) : Rule(rewriter) {
	// match on OR conjunction
	auto op = make_uniq<ConjunctionExpressionMatcher>();
	op->expr_type = make_uniq<SpecificExpressionTypeMatcher>(ExpressionType::CONJUNCTION_OR);
	op->policy = SetMatcher::Policy::SOME;

	// equi comparison on one side
	auto equal_child = make_uniq<ComparisonExpressionMatcher>();
	equal_child->expr_type = make_uniq<SpecificExpressionTypeMatcher>(ExpressionType::COMPARE_EQUAL);
	// ...
	// AND conjunction on the other, containing two IS NULL tests
	auto and_child = make_uniq<ConjunctionExpressionMatcher>();
	and_child->expr_type = make_uniq<SpecificExpressionTypeMatcher>(ExpressionType::CONJUNCTION_AND);
	// ...
}
```

`Apply` (via the `TryRewriteEqualOrIsNull` helper) checks both orderings of
the `OR`'s two children, requires the `AND` side to have exactly two `IS
NULL` children, and requires each `IS NULL` test's operand to `Expression::Equals`
one side of the `=` comparison — i.e., it has to be checking `IS NULL` on
*exactly* the same expressions the equality compares, not merely
similar-looking ones. It also gates on operator position, exactly like
`RowComparisonSimplificationRule` and `PredicateFactoringRule` below:

```cpp
// src/optimizer/rule/equal_or_null_simplification.cpp:10-21
static bool IsPredicateRoot(LogicalOperator &op, bool is_root) {
	if (!is_root) {
		return false;
	}
	switch (op.type) {
	case LogicalOperatorType::LOGICAL_FILTER:
	case LogicalOperatorType::LOGICAL_ANY_JOIN:
		return true;
	default:
		return false;
	}
}
```

This one is worth being precise about, because — unlike
`RowComparisonSimplificationRule`, where the pre-rewrite semantics were
*already* `NULL`-safe — here the rewrite changes what happens for rows
outside the pattern's own logic, but only in ways that are provably
equivalent. `a = b OR (a IS NULL AND b IS NULL)` and `a IS NOT DISTINCT FROM
b` agree for every combination of `a`/`b` (both `NULL`: original is `FALSE
OR (TRUE AND TRUE)` = `TRUE`; rewritten is `TRUE`; one `NULL`, other not:
original is `NULL OR (TRUE AND FALSE)` = `NULL OR FALSE` = `NULL`, and — this
is the detail worth sitting with — `IS NOT DISTINCT FROM` for one-`NULL`,
other-not is `FALSE`, not `NULL`. `NULL` is rejected by a `WHERE` clause
exactly as `FALSE` is, so the two forms agree observably wherever this rule
is actually allowed to fire (root of a filter/join predicate) — but they are
not the same *value* everywhere, which is exactly why the rule restricts
itself to filter/join-predicate roots via `IsPredicateRoot`, the same
discipline every context-sensitive rule in this file follows.

`test/sql/optimizer/expression/test_equal_or_null_optimization.test` proves
this restriction is load-bearing with a table built specifically to surface
the one-`NULL`/other-not case:

```sql
CREATE TABLE test(i INTEGER, j INTEGER, k integer);
INSERT INTO test VALUES (1,1,3), (2,2,4), (NULL,NULL,NULL);

query I rowsort
SELECT i FROM test WHERE (i=j) OR (i IS NULL AND j IS NULL);
----
1
2
NULL

query II nosort distinctrewrite1
EXPLAIN SELECT i IS NOT DISTINCT FROM j FROM test;
----
physical_plan	<REGEX>:.*DISTINCT.*
```

And, critically, the projection-context counter-examples, which show the
rewrite is correctly *not* value-preserving outside a predicate root — and
that the rule correctly declines to fire there:

```sql
CREATE TABLE projection_test(a INTEGER, b INTEGER);
INSERT INTO projection_test VALUES (0, NULL);

query I
SELECT a = b OR (a IS NULL AND b IS NULL) AS v FROM projection_test;
----
NULL
```

`a=0, b=NULL`: `a=b` is `NULL` (comparing to `NULL`), `a IS NULL` is `FALSE`,
so the `AND` is `FALSE`, and `NULL OR FALSE` is `NULL` — printed as `NULL`
in a `SELECT` list, which is a genuinely different displayed value than the
`FALSE` that `a IS NOT DISTINCT FROM b` would print for this row. Had the
rule mistakenly fired here (it doesn't, precisely because this is a
`SELECT`-list context, not a filter/join-predicate root), the query would
silently print `false` instead of `NULL` — a real, observable behavior
change, not merely an internal-consistency nicety. The test suite also
checks that the rule correctly identifies when the pattern *doesn't* apply
— e.g., `(i=k) OR (i IS NULL AND j IS NULL)` (mismatched columns between the
equality and the null-checks) is explicitly left as an unrewritten `OR`:

```sql
query II nosort nodistinctrewrite1
EXPLAIN SELECT i FROM test WHERE (i=k) OR (i IS NULL AND j IS NULL);
----
physical_plan	<REGEX>:.*OR.*
```

— a good reminder that this rule is a syntactic pattern match on exact
`Expression::Equals` structural identity, not a semantic prover; a
semantically-equivalent but differently-shaped predicate simply won't
match, and is left alone rather than mishandled.

---

## 7. `PredicateFactoringRule` — deriving a pushable predicate from an OR of ANDs

**Purpose:** given a disjunction whose branches are conjunctions, find any
single column that every branch constrains (even via *different*
predicates on that column in each branch), and `AND` in a new, derived,
weaker predicate — the disjunction of just those per-column constraints —
alongside the original, unmodified disjunction, so the rest of the pipeline
(pushdown, zonemaps) has something sargable to grab onto even though the
original `OR` itself can't be pushed down as a unit.

This rule's relationship to the sibling `DistributivityRule` (covered in
this chapter's companion arithmetic/constants file) is worth pinning down
precisely, since both operate on an `OR` of `AND`s and the names invite
confusion. `DistributivityRule` requires an **exact, identical subexpression**
to appear in every branch of the `OR`, and it *removes* that shared
subexpression from each branch, replacing the disjunction with a strictly
equivalent, smaller one: `(x AND a) OR (x AND b)` becomes `x AND (a OR b)` —
a genuine algebraic factoring that changes nothing about which rows match.
`PredicateFactoringRule` works at a coarser grain — "the same *column
binding* is constrained in every branch, possibly by different predicates
on that column in each branch" — and it does not remove or replace anything:
it *adds* a new, logically weaker, `AND`-ed conjunct next to the untouched
original `OR`. The new conjunct is a **necessary but not sufficient**
consequence of the original — sound to add because if every branch of the
`OR` requires *something* to be true of column `c`, then the whole `OR`
requires the disjunction of those something's to be true of `c`, regardless
of which branch a given row ultimately satisfies — but adding it changes
nothing semantically, since the original predicate is still there,
unweakened, doing the real filtering.

```cpp
// src/optimizer/rule/predicate_factoring.cpp:129-186 (abbreviated)
unique_ptr<Expression> PredicateFactoringRule::Apply(...) {
	if ((op.type != LogicalOperatorType::LOGICAL_FILTER && op.type != LogicalOperatorType::LOGICAL_ANY_JOIN) ||
	    !is_root) {
		return nullptr;
	}
	if (!ExpressionIsDisjunction(bindings[0])) {
		return nullptr;
	}
	// ... GetDisjunctedPredicateMap groups each branch's AND'd predicates by single-column binding,
	// then intersects those groups across all branches ...
	auto result = make_uniq<BoundConjunctionExpression>(ExpressionType::CONJUNCTION_AND);
	result->GetChildrenMutable().push_back(bindings[0].get().Copy());   // original OR, untouched
	result->GetChildrenMutable().push_back(std::move(derived_filter));  // new AND-ed-in derived predicate
	return std::move(result);
}
```

`test/sql/optimizer/predicate_factoring.test` walks this with a table
purpose-built for it and, notably, checks `NULL` handling directly:

```sql
CREATE TABLE t (a INTEGER, b INTEGER, c INTEGER);
INSERT INTO t VALUES (1, 5, 3), (1, 2, 3), (1, 5, 11), (2, 5, 3), (NULL, 5, 3);

-- Basic predicate factoring: a=1 is common to both disjuncts and should be extracted
query III rowsort
SELECT * FROM t WHERE (a=1 AND b>3) OR (a=1 AND c<5)
----
1	2	3
1	5	11
1	5	3

-- Verify a=1 is pushed down into the scan as a required filter
query II
EXPLAIN SELECT * FROM t WHERE (a=1 AND b>3) OR (a=1 AND c<5)
----
physical_plan	<REGEX>:.*Filters:.*a = 1.*
```

Both branches of the `OR` constrain `a` (both to `a=1`, the same predicate
in this case), so `PredicateFactoringRule` derives `a=1` on its own and
`AND`s it into the filter — visible in the physical plan as a scan-level
`Filters: ... a = 1 ...` clause, which the storage layer can use for
row-group pruning even though the full `(a=1 AND b>3) OR (a=1 AND c<5)`
predicate itself can't be pushed into the scan as a single unit (it needs
`b` and `c` too). The `NULL` row, `(NULL, 5, 3)`, correctly does not appear
in the result — `NULL` never satisfies `a=1` in either branch, and the
derived `a=1` conjunct rejects it independently and consistently with the
original predicate, so nothing is lost or double-counted by adding it.

The test's two-common-predicate case shows the rule can derive *more than
one* column's constraint from the same `OR`, `AND`ing both in:

```sql
query III rowsort
SELECT * FROM t WHERE (a=1 AND b=5 AND c>2) OR (a=1 AND b=5 AND c<6)
----
1	5	11
1	5	3

query II
EXPLAIN SELECT * FROM t WHERE (a=1 AND b=5 AND c>2) OR (a=1 AND b=5 AND c<6)
----
physical_plan	<REGEX>:.*Filters:.*a = 1.*b = 5.*
```

— and the test is equally careful to demonstrate the negative case, where
no column is constrained in *every* branch, so nothing is derived:

```sql
-- No common predicates: no factoring should occur
query III rowsort
SELECT * FROM t WHERE (a=1 AND b>3) OR (a=2 AND c<5)
----
1	5	11
1	5	3
2	5	3
```

Note what this test does and doesn't check: it only asserts the *result
rows* are unchanged, not the plan shape — there is no accompanying `EXPLAIN`
for this specific query the way there is for every other case in this file.
That matters, because reading `GetDisjunctedPredicateMap`'s own logic
(`predicate_factoring.cpp:86-127`) closely, `a` is still a column referenced
in *every* branch here (`a=1` in the first, `a=2` in the second) — the
intersection the function computes is keyed on column **binding**, not on
the branches sharing an *identical* predicate, and the accumulation loop
explicitly collects each branch's own expression for that binding into a
list (`it->second.push_back(new_predicate.get())`) rather than requiring
them to be equal. Whether that means this specific query still derives and
pushes a harmless, logically-implied `a=1 OR a=2` alongside the original
predicate isn't something this test actually settles — it settles only that
if such a filter is derived, it's derived *correctly* (the row set is
unaffected either way, exactly as `PredicateFactoringRule`'s design
requires). The test's own comment ("no factoring should occur") is a claim
about the SQL-level intuition the test's author had in mind, not something
this specific block of assertions independently verifies at the plan level —
worth flagging honestly rather than repeating as settled fact.

Finally, the rule's context-sensitivity carve-out — identical in spirit to
`RowComparisonSimplificationRule`'s and `EqualOrNullSimplification`'s — is
tested directly:

```sql
-- Rule only applies to FILTER context, not SELECT clause
query I rowsort
SELECT (a=1 AND b>3) OR (a=1 AND c<5) FROM t
----
0
1
1
1
NULL
```

The `NULL` row prints `NULL` here (a projected Boolean expression, not a
filtered-out row) — exactly what you'd expect from evaluating the
*unmodified* three-valued expression directly, confirming that outside a
`FILTER`/`ANY_JOIN` root the rule does nothing at all, leaving the original
expression's ordinary `NULL`-propagating evaluation completely untouched.

---

## 8. `MonotonePreimageRule` — turning `f(x) OP const` into a range on `x`

**Purpose, in one sentence:** when `f` is a function DuckDB has declared
monotonic in its one argument, rewrite a comparison `f(x) OP const` (for any
of `=`, `<`, `<=`, `>`, `>=`) into an equivalent comparison directly on `x` —
computing, via bisection over `x`'s own domain, the smallest range of `x`
values whose image under `f` satisfies the original comparison — so the
optimizer never has to evaluate `f` at execution time at all for this
predicate, and the resulting range on `x` is directly usable by `x`'s own
zonemap.

This is conceptually the richest rule in this batch, and it's worth
building the intuition before the code. "Monotonic" here means: as `x`
increases, `f(x)` never decreases (non-decreasing) or never increases
(non-increasing) — it *preserves order*, even if it's not strictly
one-to-one (`year()` is a textbook example: many different dates map to the
same year, but no date with a smaller year value ever produces a *larger*
`year()` output than a date with a larger year value). If you know `f` is
order-preserving, then a comparison on `f(x)`'s *output* — say, `f(x) =
2020` — corresponds to some *contiguous range* of `x` values (the
**preimage** of `2020` under `f`) — every date in the year 2020, all
contiguous once dates are ordered — and, crucially, you don't need a closed-
form inverse of `f` to find that range: because `f` is monotonic, you can
**binary-search** for the range's boundaries by just calling `f` at
candidate `x` values and checking whether the result crosses the threshold.
That's exactly what `MonotonePreimageRule` does — it treats the monotonic
function as a black box and bisects its way to the boundary, rather than
needing per-function inverse-function code.

Why this matters operationally: `year(shipdate) = 2020` evaluated the naive
way means calling `year()` on every single row's `shipdate`, then comparing
each result to `2020` — a full-table, per-row function-call cost, and one
that a zonemap on `shipdate` (which only knows `shipdate`'s own min/max, not
`year(shipdate)`'s) can't use at all to skip row groups. `shipdate >=
DATE '2020-01-01' AND shipdate <= DATE '2020-12-31'`, by contrast, is a plain
range predicate directly on the *stored* column — exactly what a zonemap is
built to answer, letting entire row groups be skipped without decompressing
or evaluating anything, and letting an index (if one exists on `shipdate`)
be used directly. The rule's entire value proposition is turning the first
form into the second automatically, for any function DuckDB has declared
monotonic, without hand-writing a rewrite per function.

### The matcher and the eligibility gate

```cpp
// src/optimizer/rule/monotone_preimage.cpp:18-28
MonotonePreimageRule::MonotonePreimageRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	// match a comparison between a foldable constant and a unary function of some expression
	auto op = make_uniq<ComparisonExpressionMatcher>();
	op->policy = SetMatcher::Policy::UNORDERED;
	op->matchers.push_back(make_uniq<FoldableConstantMatcher>());
	auto func = make_uniq<FunctionExpressionMatcher>();
	func->matchers.push_back(make_uniq<ExpressionMatcher>());
	func->policy = SetMatcher::Policy::ORDERED;
	op->matchers.push_back(std::move(func));
	root = std::move(op);
}
```

`Apply` then layers on the actual eligibility checks, each one a distinct
safety condition:

```cpp
// src/optimizer/rule/monotone_preimage.cpp:171-194
if (func.GetChildren().size() != 1 || !func.Function().HasArgProperties() ||
    func.Function().GetStability() == FunctionStability::VOLATILE) {
	return nullptr;
}
auto m = func.Function().GetArgProperties(0).monotonicity;
if (!IsKnownMonotonic(m) || m == Monotonicity::CONSTANT) {
	return nullptr;
}
if (func.GetChildren()[0]->GetExpressionClass() != ExpressionClass::BOUND_COLUMN_REF) {
	return nullptr;
}
auto &col = func.GetChildren()[0]->Cast<BoundColumnRefExpression>();
auto &col_type = col.GetReturnType();

int64_t dom_lo, dom_hi;
bool has_infinity;
if (!GetFiniteDomain(col_type, dom_lo, dom_hi, has_infinity)) {
	return nullptr;
}
```

Four gates, each load-bearing: (1) `f` must take **exactly one argument** —
this rule does not attempt to reason about multi-argument functions like
`date_trunc(unit, ts)` at all, even though `date_trunc`'s *timestamp*
argument does carry monotonicity metadata elsewhere in the source (more on
this below); (2) `f` must not be `FunctionStability::VOLATILE` — a
volatile function has no fixed preimage to compute, by definition; (3) `f`'s
one argument must itself be a **bare column reference**, not an arbitrary
expression — `year(shipdate + 1)` doesn't qualify, only `year(shipdate)`
does; and (4) the column's type must be one this rule knows how to bisect
over exactly — currently `TINYINT`/`SMALLINT`/`INTEGER`, `DATE`,
`TIMESTAMP`, and `TIMESTAMP_TZ` (`GetFiniteDomain`,
`monotone_preimage.cpp:31-67`) — with an explicit comment noting `BIGINT`'s
full `int64` domain, `HUGEINT`, other timestamp resolutions, and floating
types are all deliberately left to "a later extension," rather than handled
speculatively.

Gate (2) rejects only `VOLATILE` functions. So a function marked
`CONSISTENT_WITHIN_QUERY` (same answer for the same input during one query,
but maybe not across queries) can be rewritten too. Inside one query, such a
function is still monotone and fixed, which is all the bisection needs.

`GetFiniteDomain` deserves its own beat, because it's where the rule's
`±infinity`-awareness lives — and DuckDB's `DATE`/`TIMESTAMP` types both have
explicit `±infinity` sentinel values, which is a real correctness trap for
any bisection-based scheme:

```cpp
// src/optimizer/rule/monotone_preimage.cpp:46-60
case LogicalTypeId::DATE:
	// exclude the ±infinity sentinels so bisection stays inside the finite, totally-ordered range
	lo = int64_t(date_t::ninfinity().days) + 1;
	hi = int64_t(date_t::infinity().days) - 1;
	has_infinity = true;
	return true;
case LogicalTypeId::TIMESTAMP:
case LogicalTypeId::TIMESTAMP_TZ:
	// every non-sentinel int64 is a valid, comparable timestamp (make_timestamp accepts all of them). Search
	// between the ±infinity sentinels so that hi+1 and lo-1 stay representable. Apply handles the finite
	// INT64_MIN below -infinity.
	lo = timestamp_t::ninfinity().value + 1;
	hi = timestamp_t::infinity().value - 1;
	has_infinity = true;
	return true;
```

The bisection domain deliberately *excludes* the `±infinity` sentinel
values themselves — the search only ever probes `f` at genuinely finite
dates/timestamps, never at the sentinels. Combined with a specific bail-out
rule for infinity-carrying types (`if (has_infinity && (!is_root || op.type
!= LOGICAL_FILTER)) return nullptr;`), this is the rule's whole answer to a
subtle hazard: `year(±infinity)` is itself `NULL` (a date at positive or
negative infinity has no meaningful year), so a row whose `shipdate` is
`'infinity'` would, unrewritten, produce a `NULL` from `year(shipdate)` and
be correctly excluded by any comparison against it — but a naive `shipdate
>= X AND shipdate <= Y` rewrite, if it accidentally admitted `'infinity'`
into the range (`'infinity'` is, after all, greater than any finite date),
would wrongly *include* that row, since the rewritten range predicate has no
way to independently know that `f` of that particular boundary value is
`NULL`. Restricting the domain to the finite range, and requiring the
rewrite to only ever happen at a literal filter root (where the "both `FALSE`
and `NULL` are rejected identically" argument used throughout this file
applies), is how the rule avoids this trap without having to special-case
`±infinity` in the bisection logic itself.

### The bisection, and the two flavors of "no valid range"

`FirstTrueSuffix`/`LastTruePrefix` (`monotone_preimage.cpp:88-147`) are
textbook binary searches over the finite integer domain for the boundary
where a monotone boolean predicate (`f(x) >= c` or `f(x) > c`) flips from
`false` to `true` (or vice-versa for a decreasing function), using an
overflow-safe midpoint (`SafeMidpoint`, computed via unsigned subtraction so
it never overflows even at the edges of the domain). Once the boundary (or
boundaries, for `=`) is found, the rule constructs a `>=`/`<=` bound directly
on the column and, if both a lower and upper bound apply, `AND`s them:

```cpp
// src/optimizer/rule/monotone_preimage.cpp:315-365
a = MaxValue<int64_t>(a, dom_lo);
b = MinValue<int64_t>(b, dom_hi);

if (a > b) {
	// empty preimage. On types with ±infinity, f(±inf) is NULL (not false), so returning a plain
	// constant would mishandle those rows - bail instead.
	if (has_infinity) {
		return nullptr;
	}
	return ExpressionRewriter::ConstantOrNull(GetContext(), col.Copy(), Value::BOOLEAN(false));
}
// ... emit lower/upper BoundComparisonExpressions, AND them if both needed
```

This is the same `ConstantOrNull` mechanism seen twice already in this
file, doing the same job: if the bisection proves *no* value of `x` in the
column's domain can satisfy the original comparison (`a > b`, an empty
preimage), the rewrite has to fold to `FALSE` — but only for non-`NULL` rows
of the column; a `NULL` row must still evaluate `NULL`, exactly as `f(NULL)
OP const` would (function calls propagate `NULL` inputs to `NULL` outputs,
by DuckDB's default). And, symmetrically to the `±infinity` gate above, when
the column's type *does* carry `±infinity`, an empty preimage is
deliberately left completely unrewritten (`return nullptr`) rather than even
attempting the `ConstantOrNull` fallback — because on those types, `f`
applied to the sentinel values themselves is `NULL`, not merely "no
matching finite value," and getting that distinction right inside a
generated `ConstantOrNull` expression would require encoding logic this
rule's authors evidently decided wasn't worth the complexity for what should
be a rare case (an unsatisfiable predicate on a real column, discovered
only via exhaustive bisection).

Bounds at the *edges* of the `TIMESTAMP` domain need care. Two facts drive
it:

- The `-infinity` sentinel is not the smallest `int64`. It is
  `INT64_MIN + 1`. So `INT64_MIN` itself is a real, finite timestamp that
  sorts *below* `-infinity`.
- The search domain stops one step inside each sentinel.

The helper `MakeColBound` (`monotone_preimage.cpp:149-163`) checks for a
bound that sits exactly on the domain edge. It writes that bound as a strict
comparison with the sentinel next to it: `col > '-infinity'` or
`col < 'infinity'`. Nothing lies between the edge and the sentinel, so this
means the same thing as a non-strict bound on the edge, and it avoids
printing an extreme timestamp.

`Apply` also checks the finite value below `-infinity`
(`monotone_preimage.cpp:334-348`). When the lower bound is the domain edge,
it evaluates `f` at `dom_lo - 2` (that is, `INT64_MIN`). If that value also
satisfies the comparison, the lower bound becomes `col != '-infinity'`,
which keeps `INT64_MIN` in. The test file has a block for this:

```sql
-- test/sql/optimizer/monotone_preimage.test
EXPLAIN SELECT count(*) FROM ets WHERE epoch(t) > -9223372036854775807;
-- logical_opt  <REGEX>:.*t != '-infinity'.*

EXPLAIN SELECT count(*) FROM ets WHERE epoch(t) < 9223372036854775807;
-- logical_opt  <REGEX>:.*t != '-infinity'.*t < 'infinity'.*

-- both ends of the finite domain, the first midnight of the range, ±infinity and NULL
CREATE TABLE edge(t TIMESTAMP);
INSERT INTO edge VALUES
  (make_timestamp(-9223372036854775806)), ('290309-12-22 (BC) 00:00:00'), ('2000-01-01'),
  (make_timestamp(9223372036854775806)), ('-infinity'), ('infinity'), (NULL);

SELECT count(*) FROM edge WHERE epoch(t) > -9223372036854775807;
-- 4
```

The answer `4` counts the four finite rows. The two infinity rows have a
`NULL` epoch, and the `NULL` row is `NULL`, so all three fail the filter,
just as they would without the rewrite.

### Real test cases

`test/sql/optimizer/monotone_preimage.test` is built around `year()`, which
carries exactly the monotonicity annotation this rule needs —
`Monotonicity::NON_DECREASING`, set where the function itself is defined:

```cpp
// extension/core_functions/scalar/date/date_part.cpp:2642-2646
ScalarFunctionSet YearFun::GetFunctions() {
	auto set = GetCachedDatepartFunction<DatePart::YearOperator>();
	set.SetUnaryArgProperties(ArgProperties().NonDecreasing());
	return set;
}
```

(`NonDecreasing`, not `StrictlyIncreasing` — correctly conservative, since
many dates share the same year; the rule's `IsMonotonicIncreasing` check
accepts both, but the distinction matters for how tight a preimage the
bisection can prove.) The equivalent registration for `month()`
(`date_part.cpp:2648-2650`) carries **no** `ArgProperties` at all — `month()`
cycles `1..12` and is not monotonic in any sense the rule could exploit —
and the test explicitly checks that the rule correctly declines to touch it:

```sql
CREATE TABLE d AS SELECT (DATE '1970-01-01' + INTERVAL (range) DAY)::DATE AS dt FROM range(500000);

-- year() is strictly increasing on finite dates: year(dt) = 2020 becomes a direct date range,
-- and year() is dropped from the plan entirely (full replacement).
query II
EXPLAIN SELECT count(*) FROM d WHERE year(dt) = 2020;
----
logical_opt	<REGEX>:.*dt >= '2020-01-01'.*dt <= '2020-12-31'.*

query II
EXPLAIN SELECT count(*) FROM d WHERE year(dt) = 2020;
----
logical_opt	<!REGEX>:.*year.*

-- non-monotone functions (month cycles 1..12) must NOT be rewritten
query II
EXPLAIN SELECT count(*) FROM d WHERE month(dt) = 6;
----
logical_opt	<REGEX>:.*month.*

-- the rewritten range is sargable: it prunes row groups on the base column zonemap
query II
EXPLAIN ANALYZE SELECT count(*) FROM d WHERE year(dt) = 2020;
----
analyzed_plan	<REGEX>:.*Row Groups Scanned: 1 / 5.*
```

`year(dt) = 2020`, over 500,000 sequential dates split into 5 row groups,
becomes `dt >= '2020-01-01' AND dt <= '2020-12-31'` — a plain `DATE` range,
with `year()` gone from the plan entirely, not merely simplified — and the
row-group pruning payoff is immediate and measurable: only 1 of 5 row groups
is scanned. `month(dt) = 6`, correctly, stays as `month(dt) = 6` in the plan
— the rule found no monotonicity metadata to exploit and left it alone.

The `±infinity`/`NULL` correctness argument gets its own dedicated block,
run as real values rather than `EXPLAIN` shapes, over a table built
specifically to include both sentinels and an actual `NULL`:

```sql
CREATE TABLE e(dt DATE);
INSERT INTO e VALUES ('2020-06-15'), ('2019-01-01'), ('2021-12-31'), ('infinity'), ('-infinity'), (NULL);

query I
SELECT count(*) FROM e WHERE year(dt) = 2020;
----
1

-- > must exclude +infinity AND -infinity (both have year = NULL)
query I
SELECT count(*) FROM e WHERE year(dt) > 2020;
----
1

-- Infinity-domain rewrites are only safe in root filter predicates: projections and nested boolean
-- contexts must preserve the NULL result of year(±infinity).
query I
SELECT year(dt) = 2020 FROM e ORDER BY dt NULLS LAST;
----
NULL
false
true
false
NULL
NULL
```

`year(dt) > 2020` returning `1` (just `2021-12-31`) is the sharpest check
here: naively, you might expect `'infinity'`'s date to satisfy `> 2020` too
(infinity is "bigger" than any year), but `year('infinity')` is actually
`NULL` — an undefined year, not an enormous one — so it must be excluded
from a `> 2020` filter exactly as any other `NULL`-producing row would be,
and the test confirms exactly one row (not two) passes. The final projected
query is the cleanest possible confirmation that the rewrite's `NULL`/`FALSE`
distinction is preserved outside a filter root: `year(dt) = 2020` printed
directly (not filtered) shows `NULL` for both the `-infinity`/`infinity`
rows *and* the genuine `NULL` row, `false` for the two other real,
non-matching dates, and `true` for the matching one — six distinct,
correctly-`NULL`-aware values for six input rows, computed by the
*unrewritten* expression, because a `SELECT`-list position is exactly where
this rule's `is_root && LOGICAL_FILTER` gate declines to touch anything on
an infinity-carrying domain.

The test suite goes further than most in this file by cross-checking the
rewrite differentially against a deliberately-inserted pushdown barrier
(`OFFSET 0`, which blocks filter pushdown into the scan, forcing a different
physical evaluation path for the same logical predicate) using `EXCEPT ALL`
to assert zero rows differ between the two paths, on a 200,000+-row table
mixing ordinary dates, both infinities, and `NULL`:

```sql
query I
SELECT count(*) FROM (
  SELECT 1 FROM big WHERE year(dt) > 2000
  EXCEPT ALL SELECT 1 FROM (SELECT dt FROM big OFFSET 0) WHERE year(dt) > 2000
);
----
0
```

— and repeats the entire exercise on `TIMESTAMP` columns (an `int64`-valued
domain, exercising the overflow-safe `SafeMidpoint` bisection specifically,
since a `TIMESTAMP`'s microsecond-resolution range is wide enough that a
naive `(lo+hi)/2` midpoint calculation would overflow `int64_t` near the
domain's edges).

### A sibling mechanism, not to be confused with this one

The task of exploiting a monotonic function to prune row groups also exists
completely independently of this rule, inside
`src/planner/filter/expression_filter.cpp`'s `CanDeriveExpressionStatistics`
/ `PropagateMonotoneBounds` machinery (`expression_filter.cpp:311-333,
388-429`), and it's worth being precise about the difference so the two
don't get conflated. That mechanism runs at **scan-filter statistics
evaluation time** (deriving output-value bounds for an arbitrary expression
tree so a `LogicalGet`'s table filters can prune row groups directly,
without rewriting the plan's expression tree at all), and it is
**multi-argument aware** — it checks `func.Function().GetArgProperties(child_idx)`
for *each* child index independently, skipping any child that's foldable —
which is exactly what lets `date_trunc(unit, ts)`'s monotonicity annotation
on its *second* argument (`extension/core_functions/scalar/date/date_trunc.cpp:614`,
`func.SetArgProperties(1, ArgProperties().NonDecreasing())`) actually get
used somewhere, even though `MonotonePreimageRule` itself, requiring
`func.GetChildren().size() == 1`, could never fire on a two-argument
function like `date_trunc` at all. `test/sql/optimizer/monotone_function_zonemap_pruning.test`
demonstrates this sibling mechanism directly, with `floor()` — a genuinely
unary function, but applied through a `CAST` (`floor(x::DOUBLE)`), which
`MonotonePreimageRule`'s own gate would reject outright (its column-operand
check requires a bare `BOUND_COLUMN_REF`, and `DOUBLE` isn't one of
`GetFiniteDomain`'s supported types besides):

```sql
-- floor() carries ArgProperties (non-decreasing) but has no dedicated statistics callback, so
-- these prune purely through the monotone corner-eval wired into scan-time filter pruning.
query II
EXPLAIN ANALYZE SELECT count(*) FROM t WHERE floor(x::DOUBLE) > 495000;
----
analyzed_plan	<REGEX>:.*Row Groups Scanned: 1 / 5.*
```

So: two independent consumers of the same `ArgProperties`/`Monotonicity`
metadata, doing conceptually related but mechanically distinct jobs — one an
expression-tree *rewrite* (`MonotonePreimageRule`, unary-only, bisects a
new predicate directly on the column), the other a scan-time *statistics
derivation* (`expression_filter.cpp`, multi-argument-aware, evaluates
corner bounds without touching the plan's expressions at all). Casts, the
task brief's third suggested example alongside `date_trunc`, turn out not
to be wired through either mechanism in this checkout at all — a search of
`src/` and `extension/` for `StrictlyIncreasing()`/`NonDecreasing()`/etc.
turns up only date/time-part functions (`year`, `decade`, `century`,
`millennium`, `isoyear`, `era`, `date_trunc`'s timestamp argument, `epoch`)
and monotonic numeric functions (`ceil`, `floor`, `round`, `ln`, `log10`,
`sqrt`, and similar) — no cast. `CAST`-driven range rewrites in this
codebase go through an entirely different, cast-specific path instead:
`ComparisonSimplificationRule`'s own invertible-cast handling (§1, above)
and `TimeStampComparison` (§10, below).

`monotone_function_zonemap_pruning.test` also has a block for functions that
have their own *statistics callback* (a function that computes output
min/max from input min/max). `date_diff` and `epoch_ns` prune row groups
this way even when a row group holds more than one value:

```sql
-- test/sql/optimizer/monotone_function_zonemap_pruning.test
-- date_diff and epoch_ns prune through their statistics callback when a row group holds more than one value
EXPLAIN ANALYZE SELECT d FROM last_day_db.two WHERE date_diff('day', DATE '2025-01-01', d) > 60;
-- analyzed_plan  <REGEX>:.*Row Groups Scanned: 1 / 3.*
```

This is again the scan-time mechanism, not `MonotonePreimageRule`:
`date_diff` has three arguments, so the rule's one-argument gate rejects
it.

---

## 9. `JoinDependentFilterRule` — deriving a filter from an OR whose branches disagree by table

**Purpose:** given a disjunction of conjunctions sitting at a join's own
condition (or a filter directly above one), where each conjunction
constrains a different combination of tables' columns, derive — per table —
the disjunction of that table's own per-branch constraints, and `AND` it in,
exactly as `PredicateFactoringRule` does for a single-table `OR`, except
here the grouping key is "which table does this sub-predicate belong to,"
not "which column."

"Join-dependent" in this rule's name refers to the fact that the derivable
filter is only meaningful in light of which *table* each piece of the
original `OR` came from — a predicate like `(t1.i = 2 AND t2.i = 4) OR
(t1.i = 0 AND t2.i = 2)` doesn't have a single column any one table's
predicate factors over (unlike `PredicateFactoringRule`'s case), but it
*does* have a per-table structure: every branch constrains `t1.i` to one of
`{2, 0}` and `t2.i` to one of `{4, 2}` — so `t1.i IN (2, 0)` (derived as
`t1.i = 2 OR t1.i = 0`) and `t2.i IN (4, 2)` can both be safely `AND`-ed in,
each independently pushable to its own table's scan, even though neither
one alone is equivalent to the original predicate.

```cpp
// src/optimizer/rule/join_dependent_filter.cpp:52-100 (abbreviated)
unique_ptr<Expression> JoinDependentFilterRule::Apply(...) {
	if (!is_root) return nullptr;
	switch (op.type) {
	case LogicalOperatorType::LOGICAL_FILTER:
		break;
	case LogicalOperatorType::LOGICAL_ANY_JOIN:
	case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
	case LogicalOperatorType::LOGICAL_ASOF_JOIN: {
		const auto &join = op.Cast<LogicalJoin>();
		switch (join.join_type) {
		case JoinType::INNER:
		case JoinType::SEMI:
			break;
		default:
			return nullptr;
		}
		break;
	}
	default:
		return nullptr;
	}
	// ... must be an OR; at least one branch must be an AND referencing >1 table;
	// extract each branch's per-table conjuncts (skipping volatile expressions);
	// a table's constraint is only kept if that table appears in *every* branch's map ...
}
```

The `JoinType::INNER`/`JoinType::SEMI`-only restriction is a real semantic
boundary, not an arbitrary scope-narrowing: for a `LEFT`/`FULL OUTER JOIN`,
a row from the preserved side with no match on the other side still
survives with the other side's columns as `NULL` — deriving a filter on
`t2.i` from a predicate that presumes `t2` participated meaningfully in
every branch would incorrectly exclude exactly those preserved,
no-match rows, which is precisely the kind of outer-join/null-producing-side
interaction this chapter's task brief flagged as the concept behind
"join-dependent." Restricting to `INNER`/`SEMI` sidesteps that hazard
entirely by only ever deriving filters in join types where every row genuinely
did have both sides' predicates evaluated against real (non-`NULL`-padded)
values.

`test/optimizer/join_dependent_filter.test` is unusually thorough about
walking both the positive cases (a filter *is* derivable) and precise
negative cases (it structurally is not), which is exactly the right way to
pin down a rule this specific:

```sql
create table test as select range i, range j from range(10)

-- can derive two filters for this query
query I
select count(*)
from test t1, test t2
where (t1.i = 2 and t2.i = 4) or (t1.i = 0 and t2.i = 2)
----
2

query II
explain select count(*)
from test t1, test t2
where (t1.i = 2 and t2.i = 4) or (t1.i = 0 and t2.i = 2)
----
physical_plan	<REGEX>:.*Filters.*Filters.*
```

Two `Filters:` clauses in the physical plan — one per table's scan — confirm
both `t1.i IN (2, 0)` and `t2.i IN (4, 2)` were derived and pushed down
independently. The rule's volatility guard mirrors `PredicateFactoringRule`'s
(`GetTableIndices`/`ExtractConjunctedExpressions` explicitly skips any
sub-expression that `IsVolatile()`), tested directly:

```sql
-- not if the constants are volatile however
query II
explain select count(*)
from test t1, test t2
where (t1.i = random() and t2.i = random()) or (t1.i = 0 and t2.i = 2)
----
physical_plan	<!REGEX>:.*FILTER.*
```

And the "must appear in *every* branch" requirement — the crux of why this
rule needs the disjunction's branches to *all* constrain a given table, not
just some of them — is tested with a minimal pair that differs by exactly
one added predicate:

```sql
-- a predicate for a column must show up on both sides,
-- so, adding a predicate for t2.i to only one side won't create a filter
query I
select count(*)
from test t1, test t2
where (t1.i = t2.i and t1.i > 7 and t2.i = 0) or (t1.i != t2.i and t1.i < 3)
----
27

query II
explain select count(*)
from test t1, test t2
where (t1.i = t2.i and t1.i > 7 and t2.i = 0) or (t1.i != t2.i and t1.i < 3)
----
physical_plan	<!REGEX>:.*FILTER.*FILTER.*

-- if we add another predicate to the other side, we get another filter
query II
explain select count(*)
from test t1, test t2
where (t1.i = t2.i and t1.i > 7 and t2.i = 0) or (t1.i != t2.i and t1.i < 3 and t2.i = 5)
----
physical_plan	<REGEX>:.*Filters.*Filters.*
```

The first query's second branch, `t1.i != t2.i and t1.i < 3`, says nothing
at all about `t2.i` on its own — so no derived filter on `t2.i` is sound
(some rows satisfying that branch could have any `t2.i` value whatsoever),
and the plan shows only the single filter derivable from `t1.i`. Adding
`t2.i = 5` to that same branch makes `t2.i` constrained in *both* branches,
and the second `Filters:` clause appears. The test file closes with two real
TPC-H queries (Q7 and Q19, both textbook examples of exactly this
"OR of ANDs, each AND naming different tables' columns" shape in their
canonical form) proving the rule derives the expected number of filters
against realistic, larger join graphs, and that the query's actual output
still matches TPC-H's reference answer set after the rewrite — the
strongest form of evidence this rule doesn't change results, only plan
shape.

---

## 10. `TimeStampComparison` — a note on the task's named test, and the rule's real coverage

**Purpose, in one sentence:** when a `TIMESTAMP` column is cast to `DATE`
and compared for equality against a foldable `DATE` constant, rewrite the
comparison into a direct range on the *un-cast* `TIMESTAMP` column — `CAST(ts
AS DATE) = D` becomes `ts >= D AND ts < D + 1 day` — so the cast (which would
otherwise have to be evaluated per row) disappears from the plan and the
comparison becomes directly sargable against `ts`'s own zonemap.

A source-fidelity note belongs here, in the same spirit as the overview's
resolution of the `sum_rewriter` naming question (§7.6.5) and its flagged
uncertainty about `MATERIALIZED_CTE` (§7.4): **the test file named in this
chapter's brief, `test/optimizer/timestamp_comparisons.test`, does not
actually exercise this rule.** Reading it in full, it's a real, valuable
test file — but its subject is the general correctness of `TIMESTAMPTZ`
casting, timezone-normalization, and cross-resolution (`TIMESTAMP_S` /
`TIMESTAMP_MS` / `TIMESTAMP_NS`) comparison behavior at the *execution*
level (e.g. `t::timestamp = timestamp '...'` where `t` is a `TIMESTAMPTZ`
column, and the surprising-but-correct fact that this can fail to match
because the cast strips the timezone-aware instant down to a naive local
wall-clock reading). None of its queries construct the pattern
`TimeStampComparison`'s own matcher requires — a cast **to `DATE`** of a
**bare `TIMESTAMP`-typed column reference**:

```cpp
// src/optimizer/rule/timestamp_comparison.cpp:19-39
TimeStampComparison::TimeStampComparison(ExpressionRewriter &rewriter) : Rule(rewriter), context(rewriter.context) {
	// match on an equality comparison between a timestamp column cast to DATE and a foldable DATE constant
	auto op = make_uniq<ComparisonExpressionMatcher>();
	op->policy = SetMatcher::Policy::UNORDERED;
	op->expr_type = make_uniq<SpecificExpressionTypeMatcher>(ExpressionType::COMPARE_EQUAL);

	auto left = make_uniq<CastExpressionMatcher>();
	left->type = make_uniq<TypeMatcherId>(LogicalTypeId::DATE);
	left->matcher = make_uniq<ExpressionMatcher>();
	left->matcher->expr_class = ExpressionClass::BOUND_COLUMN_REF;
	left->matcher->type = make_uniq<TypeMatcherId>(LogicalTypeId::TIMESTAMP);
	op->matchers.push_back(std::move(left));

	auto right = make_uniq<FoldableConstantMatcher>();
	right->type = make_uniq<TypeMatcherId>(LogicalTypeId::DATE);
	op->matchers.push_back(std::move(right));

	root = std::move(op);
}
```

Tracing the rule's own git history to its introducing commit
(`be12a75ebb "have rule matching working"`, May 2024) turns up the actual,
purpose-built test the author added alongside the rule itself:
`test/optimizer/pushdown/timestamp_to_date_pushdown.test` (plus a
follow-up, `timestamp_to_date_infinity_pushdown.test`, added later
specifically to cover `±infinity` `DATE` constants). This section walks
those two files instead, since they're the ones that actually exercise
`TimeStampComparison`'s logic — while still being explicit, per this book's
sourcing policy, that the brief's named file is real and worth reading for
its own (different, also `NULL`/precision-sensitive) subject matter.

### The core rewrite, proven both ways

```sql
-- test/optimizer/pushdown/timestamp_to_date_pushdown.test
create or replace table t1 (ts timestamp, i int);
insert into t1 select '2024-05-01 00:00:00'::timestamp ts, i from generate_series(1, 2000) g(i);
insert into t1 select '2024-05-02 00:00:00'::timestamp ts, i from generate_series(1, 1000) g(i);
insert into t1 select '2024-05-02 00:22:00'::timestamp ts, i from generate_series(1, 1000) g(i);
insert into t1 select '2024-05-03 00:00:00'::timestamp ts, i from generate_series(1, 2000) g(i);

query II
explain select * from t1 where ts::date == '2024-05-02';
----
physical_plan	<REGEX>:.*SEQ_SCAN.*Filters.*

query II
explain select * from t1 where '2024-05-02' == ts::date;
----
physical_plan	<REGEX>:.*SEQ_SCAN.*Filters.*
```

Both orderings of the comparison (`ts::date == const` and `const ==
ts::date`) produce a scan with a `Filters:` clause directly on the base
`SEQ_SCAN` — the cast has been eliminated and the comparison pushed all the
way to the scan, which is exactly what the source's
`op->policy = SetMatcher::Policy::UNORDERED` is for (matching either operand
order without needing two separate rule instances). The test then proves
this is genuinely a rewrite, not just a different-but-equivalent way of
writing the same plan, by disabling optimization entirely and re-running the
identical query, checking the *unoptimized* result against the *optimized*
one under the same result label:

```sql
statement ok
pragma disable_optimizer;

query II nosort no_opt_result
select * from t1 where ts::date == '2024-05-02' order by ts, i;
----

statement ok
pragma enable_optimizer;

query II nosort no_opt_result
select * from t1 where ts::date == '2024-05-02' order by ts, i;
----
```

Same result set both ways — the rewrite changes the plan, not the answer.
The test also checks that the pattern is still recognized when it's one
conjunct among several (not necessarily the entire filter predicate) and
gives a concrete row-count proof that the date-boundary arithmetic is
correct — `2024-05-02 00:22:00`, a timestamp with a nonzero time-of-day, is
correctly included as matching `ts::date = '2024-05-02'` (unlike the
`DATE`-column case in §1, this is asking whether the *timestamp itself*,
truncated to its date, equals a given day — which every timestamp on
2024-05-02 satisfies, regardless of its time-of-day):

```sql
query I
select count(*) from t1 where ts::date == '2024-05-02' and i <= 500;
----
1000
```

1,000, not 500 — both the midnight batch (500 of its 1,000 rows have `i <=
500`) and the `00:22:00` batch (500 of its 1,000 rows have `i <= 500`)
qualify, since both timestamps fall on 2024-05-02 regardless of
time-of-day, confirming the rewritten range genuinely spans the whole day
(`ts >= '2024-05-02 00:00:00' AND ts < '2024-05-03 00:00:00'`), not just the
exact midnight instant.

### `±infinity`, and the same `ConstantOrNull`-style discipline

`timestamp_to_date_infinity_pushdown.test` is the direct analogue of
`MonotonePreimageRule`'s own `±infinity` care (§8) — a `DATE` constant of
`'infinity'` or `'-infinity'` needs the rewrite's day-boundary-plus-one-day
arithmetic to degrade gracefully rather than overflow or silently produce a
wrong range:

```sql
CREATE TABLE ts_inf(t timestamp);
INSERT INTO ts_inf VALUES (timestamp '-infinity'), (timestamp '2025-01-01'), (timestamp 'infinity'), (NULL);

query I
select count(*) from ts_inf where cast(t as date) = date 'infinity';
----
1

query I
select count(*) from ts_inf where cast(t as date) = date '-infinity';
----
1
```

The source code's own handling of this is visible directly in
`TimeStampComparison::Apply`: rather than always computing "the constant's
date, plus one day" as the upper bound, it checks finiteness first and takes
a completely different branch for a non-finite constant, emitting a direct
equality on the *timestamp* column against the equivalent `±infinity`
timestamp value instead of a range:

```cpp
// src/optimizer/rule/timestamp_comparison.cpp:96-103
if (!original_val.IsFinite()) {
	auto column_copy = cast_columnref->Copy();
	auto eq_expr = BoundComparisonExpression::Create(ExpressionType::COMPARE_EQUAL, std::move(column_copy),
	                                                 std::move(original_val_for_comparison));
	new_expr->GetChildrenMutable().push_back(std::move(eq_expr));
	return std::move(new_expr);
}
```

For the finite case, the code explicitly avoids calling DuckDB's normal
date-arithmetic addition operator directly, specifically because that
operator throws on overflow and this rule needs to *decline* the rewrite
silently rather than propagate an exception from inside a plan-rewrite pass:

```cpp
// src/optimizer/rule/timestamp_comparison.cpp:105-115
// code is inspired by AddOperator::Operation(date_t left, int32_t right). The function wasn't used directly
// since it throws errors that I cannot catch here.
if (!TryAddOperator::Operation<date_t, int32_t, date_t>(date_t_copy, 1, date_t_result)) {
	// don't rewrite the expression and let the expression executor handle the invalid date
	return nullptr;
}
```

A `DATE` constant one day short of the type's maximum representable value
would overflow when the rule tries to compute "plus one day" for the range's
upper bound; rather than risk that, the code uses the `Try`-prefixed,
non-throwing variant and simply abandons the rewrite (falling back to the
original, unrewritten `CAST(...) = ...` expression, which the ordinary
expression executor will still evaluate correctly, just without the
zonemap-friendly range) if the addition can't be proven safe. This is the
same defensive posture as `ConstantCastIsInvertible`'s conservative
`return false` default in §1 and `MonotonePreimageRule`'s `EvalFunc`
bailing on any evaluation failure in §8: every rule in this file treats "I
can't prove this rewrite is safe" as a reason to leave the original
expression alone, never as a reason to guess.

---

## Closing: the theme across all ten rules

Read together, these ten rules are a study in one recurring engineering
discipline more than they are ten unrelated tricks: **a rewrite is only
applied when it can be shown to agree with the original expression on every
input, including every input where the original would have produced `NULL`
— and where an exact identity isn't available everywhere, the rule instead
restricts itself to the narrow syntactic position (the root of a `WHERE` or
join predicate) where `NULL` and `FALSE` are provably interchangeable to
whatever operator sits above it.** `ComparisonSimplificationRule`'s
`ConstantOrNull` wrapper, `NotComparisonSimplificationRule`'s reliance on
comparisons sharing identical `NULL`-propagation before and after negation,
`ConjunctionSimplificationRule`'s `OR`-only, filter-root-only `NULL`-dropping
carve-out, `NotConjunctionSimplificationRule`'s volatility/`CanThrow` guard
against changing *evaluation counts* even though the *Boolean values*
already agreed exactly, `CaseSimplificationRule`'s `NULL`-treated-as-`FALSE`
branch selection (correct precisely because that's what `CASE` itself does),
`EqualOrNullSimplification`'s and `PredicateFactoringRule`'s and
`RowComparisonSimplificationRule`'s and `JoinDependentFilterRule`'s shared
`is_root`-plus-operator-type gating, and `MonotonePreimageRule`'s and
`TimeStampComparison`'s both-independently-arrived-at `±infinity` carve-outs
— every one of these is a variation on the same question: *does this
rewrite still tell the truth about the rows where the original expression
would have shrugged and said "unknown"?* The test suite's own repeated
pattern — construct a table with an explicit `NULL` row (and, for the
temporal rules, explicit `±infinity` rows), run the *un-rewritten* query,
run the rewritten one, and check they agree — is the mechanical embodiment
of that same question, asked and answered file after file.
