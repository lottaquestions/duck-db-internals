# Chapter 7, Deep Dive — The Expression Rewriter: Strings and Regex

This file is one deep-dive part of Chapter 7 (see `00-overview.md` for the full
optimizer pipeline and §7.5 for the complete, ordered 37-rule list that makes up
pass #1, `EXPRESSION_REWRITER`). It expands on one coherent slice of that list: the
rules whose entire job is recognizing that a `LIKE`, `GLOB`, `regexp_matches`,
`regexp_replace`, `substring`/`left`/`instr`, `contains`/`list_contains`, or list
comprehension expression is doing something *cheaper* than its general syntax
admits, and rewriting it down to that cheaper thing. In list order, as pushed in
`Optimizer::Optimizer` (`src/optimizer/optimizer.cpp:73`), the rules covered here
are:

```cpp
rewriter.rules.push_back(make_uniq<LikeOptimizationRule>(rewriter));
rewriter.rules.push_back(make_uniq<StringPrefixRule>(rewriter));
rewriter.rules.push_back(make_uniq<InstrPrefixRule>(rewriter));
// ...
rewriter.rules.push_back(make_uniq<RegexOptimizationRule>(rewriter));
rewriter.rules.push_back(make_uniq<RegexpReplaceExtractRule>(rewriter));
rewriter.rules.push_back(make_uniq<EmptyNeedleRemovalRule>(rewriter));
rewriter.rules.push_back(make_uniq<NoopReplaceRemovalRule>(rewriter));
// ...
rewriter.rules.push_back(make_uniq<ListComprehensionRewriteRule>(rewriter));
rewriter.rules.push_back(make_uniq<ContainsToInClauseRule>(rewriter));
```

This is one of the most densely test-covered corners of the entire rule set —
seven `.test` files map almost directly onto seven pieces of source, and one of
those files (`regex_optimizer_coverage.test`) exists purely to shore up gaps a
sibling file left uncovered. It's also a good showcase of a theme that recurs
throughout this chapter: several of these rules don't just make one query
faster, they exist to *feed a later pass* — `LikeOptimizationRule`'s `prefix()`
output becomes a plan-level range filter the storage layer can prune row groups
with; `ContainsToInClauseRule`'s `IN`-clause output becomes the input the
plan-level `IN_CLAUSE` pass decides whether to turn into a hash join. Both of
those downstream consequences are traced to their actual source in this file.

## A quick note on `nosort <label>` blocks, before the walkthroughs start

Several test files below (`like_optimizer.test`, `needle_optimizer.test`) use a
sqllogictest idiom worth naming explicitly the first time it shows up: two (or
more) `query I nosort <label>` blocks that share the same `<label>` are not
checked against a literal expected value written in the file — they're checked
**against each other**, byte-for-byte. A file that writes

```
query I nosort prefix
EXPLAIN SELECT S LIKE 'aaa%' FROM test
----

query I nosort prefix
EXPLAIN SELECT prefix(S, 'aaa') FROM test
----
```

is asserting that these two queries produce an *identical* `EXPLAIN` tree — which
is precisely the claim "the optimizer rewrites the first query into the second,"
made executable rather than merely described. Neither block has an expected
value written under its `----`; the harness fills that in from whichever block
in the group ran first, and compares every subsequent block sharing the label
against it. This is how nearly all of `like_optimizer.test` is built, and it's
worth internalizing once here since the idiom recurs constantly in this file and
across the optimizer test suite generally.

## `LikeOptimizationRule` — the flagship rule of this batch

**Purpose, in one sentence:** rewrite a `LIKE`/`NOT LIKE`/`GLOB`/`NOT GLOB`
expression whose pattern is a compile-time constant into the cheapest primitive
that means the same thing — plain equality if the pattern has no wildcards at
all, `prefix()` if the only wildcard is a trailing run of `%`, `suffix()` for a
leading run, `contains()` for both, and no rewrite at all if the pattern needs
`LIKE`'s actual machinery (an internal `%`, or any `_`).

### The matcher

```cpp
// src/optimizer/rule/like_optimizations.cpp:14
LikeOptimizationRule::LikeOptimizationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	// match on a FunctionExpression that has a foldable ConstantExpression
	auto func = make_uniq<FunctionExpressionMatcher>();
	func->matchers.push_back(make_uniq<ExpressionMatcher>());
	func->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
	func->policy = SetMatcher::Policy::ORDERED;
	// we match on LIKE ("~~"), NOT LIKE ("!~~"), GLOB ("~~~"), NOT GLOB ("!~~~"), ILIKE ("~~*"), and NOT ILIKE ("!~~*")
	func->function =
	    make_uniq<ManyFunctionMatcher>(identifier_set_t {Identifier("!~~"), Identifier("~~"), Identifier("!~~~"),
	                                                     Identifier("~~~"), Identifier("~~*"), Identifier("!~~*")});
	root = std::move(func);
}
```

The matcher also lists `ILIKE` (`"~~*"`) and `NOT ILIKE` (`"!~~*"`). This
does not mean `ILIKE` is rewritten to `prefix()` and friends. `Apply` only
uses these two names to fold a `NULL` pattern to `NULL`, then returns
`nullptr` for any other `ILIKE` (see `Apply` below). The reason is
correctness: a `NULL` `ILIKE` pattern must give `NULL` for every row (DuckDB
issue #26433 was a bug where the plan lost that filter). The next sections
explain both halves.

Two things worth noting immediately, because they explain a lot of what this
rule can and can't reach. First, the matcher requires exactly two children in a
fixed order (`SetMatcher::Policy::ORDERED`) — an arbitrary expression, then a
constant. That means `LIKE ... ESCAPE '...'` never matches this rule at all: an
escaped `LIKE` is bound to a completely different, three-argument function named
`like_escape` (`ilike_escape` for `ILIKE`), a distinction made explicitly at
parse time —

```cpp
// src/parser/peg/transformer/transform_expression.cpp:1242
like_variation = "like_escape";
```

— rather than at optimization time. `LikeOptimizationRule`'s `ManyFunctionMatcher`
only lists `"~~"`/`"!~~"`/`"~~~"`/`"!~~~"` (plus the two `ILIKE` names);
`"like_escape"` isn't in that set, so
an escaped `LIKE` simply never has a chance to be classified as constant, prefix,
suffix, or contains — it always keeps its full generality. This is a clean
architectural answer to "does this rule handle escape characters correctly": it
doesn't handle them at all, because it never sees them.

Second, both `LIKE` and `GLOB` are handled by one rule and one `Apply` body,
distinguished only by which of the four function-name strings actually matched
— `PatternMatchType::GLOB` versus `PatternMatchType::LIKE` — which is what lets
`SELECT s GLOB 'aaa*'` and `SELECT s LIKE 'aaa%'` share every byte of rewrite
logic despite using different wildcard characters (`*`/`?` for `GLOB`, `%`/`_`
for `LIKE`).

### The four classifiers

The heart of the rule is four small, static, pattern-only functions — none of
them touch a single row of data, they only look at the literal pattern string
that was already folded to a constant by the binder:

```cpp
// src/optimizer/rule/like_optimizations.cpp:27
static bool PatternIsConstant(const string &pattern, PatternMatchType match_type) {
	for (idx_t i = 0; i < pattern.size(); i++) {
		if (match_type == PatternMatchType::GLOB) {
			if (pattern[i] == '*' || pattern[i] == '?' || pattern[i] == '[' || pattern[i] == '\\') {
				return false;
			}
		} else {
			if (pattern[i] == '%' || pattern[i] == '_') {
				return false;
			}
		}
	}
	return true;
}
```

`PatternIsConstant` is the simplest: any wildcard character anywhere in the
pattern (for `LIKE`, that's `%` *or* `_`) disqualifies it. If nothing
disqualifies it, the entire pattern is a literal string and `LIKE` degenerates
exactly to `=`.

```cpp
// src/optimizer/rule/like_optimizations.cpp:42
static bool PatternIsPrefix(const string &pattern, PatternMatchType match_type) {
	char wildcard_any = match_type == PatternMatchType::GLOB ? '*' : '%';
	idx_t i;
	for (i = pattern.size(); i > 0; i--) {
		if (pattern[i - 1] != wildcard_any) {
			break;
		}
	}
	if (i == pattern.size()) {
		// no trailing wildcard
		// cannot be a prefix
		return false;
	}
	// continue to look in the string
	// if there is a wildcard in the string (besides at the very end) this is not a prefix match
	for (; i > 0; i--) {
		if (match_type == PatternMatchType::GLOB) {
			if (pattern[i - 1] == '*' || pattern[i - 1] == '?' || pattern[i - 1] == '[' || pattern[i - 1] == '\\') {
				return false;
			}
		} else {
			if (pattern[i - 1] == '%' || pattern[i - 1] == '_') {
				return false;
			}
		}
	}
	return true;
}
```

`PatternIsPrefix` strips a trailing run of `%`/`*` first (it must find at least
one, or the pattern isn't a prefix pattern at all), then scans everything to the
left of that run for *any* wildcard character — and, critically, that scan
rejects `_` (or `?` for `GLOB`) exactly the same way it rejects `%`/`*`. The
inline comment right at the call site spells this out precisely as a
regex-flavored shorthand:

```cpp
// src/optimizer/rule/like_optimizations.cpp:170
// Prefix LIKE pattern : [^%_]*[%]+, ignoring underscore
```

Read literally, `[^%_]*` means "zero or more characters that are *neither* `%`
nor `_`" — so a single-character wildcard **anywhere** in the literal portion of
the pattern disqualifies the whole prefix/suffix/contains classification, not
just a `%`. `'ab_%'`, `'a_bc%'`, and `'_abc%'` are all left completely alone by
this rule — none of `PatternIsConstant`, `PatternIsPrefix`, `PatternIsSuffix`,
nor `PatternIsContains` will accept them, and `LikeOptimizationRule::Apply`
falls through to `return nullptr` for all of them, leaving the original `LIKE`
call untouched. This is the answer to the natural question "what about `_`":
DuckDB's `prefix()`/`suffix()`/`contains()` primitives have no notion of "any
single character" at all, so there is no cheaper primitive a pattern containing
`_` could be rewritten into, and the rule correctly declines rather than
attempting something wrong. The comment's "ignoring underscore" phrasing is a
little easy to misread on a first pass — it does not mean underscore is
tolerated; it means the *regex shorthand itself* omits a separate clause for it
because `_` is folded into the same disqualifying character class as `%`.

`PatternIsSuffix` and `PatternIsContains` are exact mirror images: strip a
*leading* wildcard run (suffix) or both a leading *and* trailing* run
(contains), then apply the identical "no wildcard character anywhere in what's
left" check to the remainder. `PatternIsContains` additionally requires *both*
ends to have a wildcard run — `'%aaa'` alone is a suffix pattern, not a contains
pattern, because `PatternIsContains`'s check `if (start == 0 || end ==
pattern.size())` explicitly bails when only one side has a wildcard.

### `Apply` and `ApplyRule`

```cpp
// src/optimizer/rule/like_optimizations.cpp:133
unique_ptr<Expression> LikeOptimizationRule::Apply(LogicalOperator &op, vector<reference<Expression>> &bindings,
                                                   bool &changes_made, bool is_root) {
	auto &root = bindings[0].get().Cast<BoundFunctionExpression>();
	auto &constant_expr = bindings[2].get().Cast<BoundConstantExpression>();
	D_ASSERT(root.GetChildren().size() == 2);

	if (constant_expr.GetValue().IsNull()) {
		return make_uniq<BoundConstantExpression>(Value(root.GetReturnType()));
	}

	bool is_ilike = root.Function().GetName() == "~~*" || root.Function().GetName() == "!~~*";
	if (is_ilike) {
		return nullptr;
	}

	// the constant_expr is a scalar expression that we have to fold
	if (!constant_expr.IsFoldable()) {
		return nullptr;
	}

	Value constant_value;
	if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), constant_expr, constant_value)) {
		return nullptr;
	}
	// ...
	if (PatternIsConstant(patt_str, match_type)) {
		// Pattern is constant
		return BoundComparisonExpression::Create(
		    is_not_like ? ExpressionType::COMPARE_NOTEQUAL : ExpressionType::COMPARE_EQUAL,
		    std::move(root.GetChildrenMutable()[0]), std::move(root.GetChildrenMutable()[1]));
	} else if (PatternIsPrefix(patt_str, match_type)) {
		return ApplyRule(root, "prefix", patt_str, is_not_like, match_type);
	} else if (PatternIsSuffix(patt_str, match_type)) {
		return ApplyRule(root, "suffix", patt_str, is_not_like, match_type);
	} else if (PatternIsContains(patt_str, match_type)) {
		return ApplyRule(root, "contains", patt_str, is_not_like, match_type);
	}
	return nullptr;
}
```

A `NULL` pattern is handled before any classification runs at all: `s LIKE
NULL` is always `NULL`, regardless of `s`, so the whole expression folds
directly to a typed `NULL` constant — no wildcard scanning needed, and no
runtime `LIKE` evaluation either.

Two more details in `Apply`:

1. **`ILIKE` with a `NULL` pattern.** The `NULL` check also folds
   `s ILIKE NULL` to `NULL`. Any other `ILIKE` stops at
   `if (is_ilike) return nullptr;`. A related guard lives in
   `FilterCombiner::TryPushdownLikeFilter` in `filter_combiner.cpp`: for a
   `NULL` pattern it returns `NO_PUSHDOWN`, and the filter stays. The test
   `test/sql/filter/test_like_null.test` guards against a bug (DuckDB issue
   #26433) where a `NULL` pattern pushed an `IS NOT NULL` table filter and
   reported the filter as fully pushed down. That dropped the real
   predicate, so every non-`NULL` row matched. The test runs the exact query
   from the issue:

   ```sql
   -- test/sql/filter/test_like_null.test
   CREATE TABLE t(c1 VARCHAR);
   INSERT INTO t VALUES ('Jd*q^c'),(''),('x');

   -- Test 1 — exact regression from issue #26433
   SELECT count(*) FROM t WHERE c1 ILIKE (CASE CAST(NULL AS BLOB) WHEN NULL THEN c1 END);
   -- 0
   ```

   The `CASE` has no matching branch and no `ELSE`, so it is `NULL`.
   `c1 ILIKE NULL` is `NULL` for every row, and `WHERE` keeps none.

2. **A pattern that cannot be evaluated.** The rule uses
   `TryEvaluateScalar`. If folding the pattern raises an error, the rule
   returns `nullptr` and leaves the error to execution. So the error is not
   raised while planning when the `LIKE` could never run.
   `test/optimizer/constant_argument_errors_like_regex.test` checks this:

   ```sql
   -- test/optimizer/constant_argument_errors_like_regex.test
   statement ok
   SELECT CASE WHEN false THEN s LIKE ''::DOUBLE::VARCHAR END FROM t;

   statement error
   SELECT s LIKE ''::DOUBLE::VARCHAR FROM t UNION ALL SELECT 'a' LIKE ''::DOUBLE::VARCHAR;
   ----
   <REGEX>:Conversion Error.*
   ```

   `''::DOUBLE` cannot be converted. In the first query that branch never
   runs, so the query succeeds. In the second, the `'a' LIKE ...` row
   really is evaluated, so the error appears at execution time.

`ApplyRule` does the actual surgery once a classification has been chosen:

```cpp
// src/optimizer/rule/like_optimizations.cpp:182
unique_ptr<Expression> LikeOptimizationRule::ApplyRule(BoundFunctionExpression &expr, const Identifier &function_name,
                                                       string pattern, bool is_not_like,
                                                       PatternMatchType match_type) const {
	// replace LIKE by an optimized function
	auto result = BindBuiltinScalarFunction(GetContext(), function_name, std::move(expr.GetChildrenMutable()));

	// removing wildcard from the pattern
	char wildcard_any = match_type == PatternMatchType::GLOB ? '*' : '%';
	pattern.erase(std::remove(pattern.begin(), pattern.end(), wildcard_any), pattern.end());

	result->GetChildrenMutable()[1] = make_uniq<BoundConstantExpression>(Value(std::move(pattern)));

	if (!is_not_like) {
		return std::move(result);
	}
	auto negation = make_uniq<BoundOperatorExpression>(ExpressionType::OPERATOR_NOT, LogicalType::BOOLEAN);
	negation->GetChildrenMutable().push_back(std::move(result));
	return std::move(negation);
}
```

`std::remove(...,wildcard_any,...)` deletes *every* occurrence of the wildcard
character from the pattern string, not just one — which is exactly what makes
`'aaa%%%%'` collapse correctly: four consecutive `%` all get stripped in one
pass, leaving the same bare `'aaa'` constant `prefix(S, 'aaa')` would use for a
single `%`. `NOT LIKE` wraps the resulting call in a plain `OPERATOR_NOT`
rather than trying to find a negated primitive — DuckDB has no `not_prefix()`
function, so `NOT LIKE 'aaa%'` becomes `NOT(prefix(S, 'aaa'))`, not some fused
negated-prefix call.

### Walking `test/optimizer/like_optimizer.test`, case by case

The file sets up one row (`'aaa'`) in a one-column table and drives every
classification through the same-label idiom introduced above. Every block
below is a genuine, runnable pair.

**No wildcards at all — degenerates to `=`:**

```sql
-- label: nosymbols
EXPLAIN SELECT S LIKE 'aaa' FROM test
EXPLAIN SELECT S='aaa' FROM test
```

**Trailing wildcard(s) — becomes `prefix()`, including the multi-`%` case:**

```sql
-- label: prefix
EXPLAIN SELECT S LIKE 'aaa%' FROM test
EXPLAIN SELECT prefix(S, 'aaa') FROM test
EXPLAIN SELECT S LIKE 'aaa%%%%' FROM test    -- same plan: %%%% strips to one prefix('aaa')
```

**Leading wildcard(s) — becomes `suffix()`:**

```sql
-- label: suffix
EXPLAIN SELECT S LIKE '%aaa' FROM test
EXPLAIN SELECT suffix(S, 'aaa') FROM test
EXPLAIN SELECT S LIKE '%%%%aaa' FROM test
```

**Both ends wildcarded — becomes `contains()`:**

```sql
-- label: contains
EXPLAIN SELECT S LIKE '%aaa%' FROM test
EXPLAIN SELECT contains(S, 'aaa') FROM test
EXPLAIN SELECT S LIKE '%%%%aaa%%' FROM test  -- asymmetric run lengths on each side, still fine
```

The file then repeats the exact same four groups for `NOT LIKE`
(`notlikenosymbols`, `notlikeprefix`, `notlikesuffix`, `notlikecontains`), each
one's second reference query written as the explicit `NOT(...)`-wrapped form —
`EXPLAIN SELECT NOT(prefix(S, 'aaa')) FROM test`, etc. — confirming the
`OPERATOR_NOT`-wrapping behavior from `ApplyRule` above holds for all three
non-trivial classifications, not just prefix. Two things the file does **not**
cover, worth flagging explicitly rather than glossing over: it never exercises
`GLOB`/`NOT GLOB` at all (despite `LikeOptimizationRule`'s matcher explicitly
listing `"~~~"`/`"!~~~"`), and it never exercises a pattern containing `_` —
both gaps this book's source reading fills in above, since the rule's own
logic makes the intended behavior unambiguous even without a `.test` file
proving it.

### Where the `prefix()` payoff actually lands: `filter_combiner.cpp`

`LikeOptimizationRule` rewrites `S LIKE 'aaa%'` into `prefix(S, 'aaa')`, but
`prefix()` itself is just an ordinary scalar function — nothing about
*evaluating* `prefix(S, 'aaa')` row-by-row is inherently cheaper than evaluating
`S LIKE 'aaa%'` row-by-row. The real payoff happens five passes later, inside
`FILTER_PUSHDOWN` (#6 in the overview's §7.4 pipeline), which specifically
recognizes a `prefix(column, 'literal')` filter and unfolds it into a genuine
range predicate the storage layer's zonemaps can act on:

```cpp
// src/optimizer/filter_combiner.cpp:490
FilterPushdownResult FilterCombiner::TryPushdownPrefixFilter(TableFilterSet &table_filters,
                                                             const vector<ColumnIndex> &column_ids, Expression &expr) {
	// ...
	if (function_name != "prefix" && function_name != "starts_with" && function_name != "^@") {
		return FilterPushdownResult::NO_PUSHDOWN;
	}
	// ...
	auto lower_bound = CreateComparisonExpression(*func.GetChildren()[0], ExpressionType::COMPARE_GREATERTHANOREQUALTO,
	                                              Value(prefix_string));
	table_filters.PushFilter(filter_idx, make_uniq<ExpressionFilter>(std::move(lower_bound)));
	if (Utf8Proc::FindNextLegalUTF8(prefix_string)) {
		auto upper_bound =
		    CreateComparisonExpression(*func.GetChildren()[0], ExpressionType::COMPARE_LESSTHAN, Value(prefix_string));
		table_filters.PushFilter(filter_idx, make_uniq<ExpressionFilter>(std::move(upper_bound)));
		return FilterPushdownResult::PUSHED_DOWN_FULLY;
	}
```

`prefix(s, 'aaa')` becomes `s >= 'aaa' AND s < 'aab'` (the "next legal UTF-8
string" after the prefix) at the table-filter level — a genuine two-sided range,
exactly the shape a row group's min/max zonemap can prune against without
reading a single value. `03`'s sibling file `string_prefix.test` (walked in the
next section) shows this concretely with `EXPLAIN ANALYZE` output that reports
`Filters: s >= '09999'` and a row-group count dropping from 82 scanned to 1.
`LikeOptimizationRule` and `TryPushdownPrefixFilter` are two entirely separate
passes, in two entirely separate source files, that only cooperate because they
agree on `prefix()` as a shared vocabulary — this is a small, concrete instance
of the "one rule's output shape exists to feed a later pass" pattern this file
returns to repeatedly.

### `test/optimizer/regex_to_like_optimizer.test` — a different direction, and a guard, not a rewrite

This file's name suggests "a regex that's secretly a `LIKE` pattern gets
simplified back down" — and that mechanism is real (it's `RegexOptimizationRule`,
covered fully in the next section), but this specific file is not actually
demonstrating a successful rewrite. Its whole content is two queries against
case-toggling regex flags:

```sql
-- test/optimizer/regex_to_like_optimizer.test
select word, regexp_matches(word, '(?i)b.*(?-i)[S]') from test;
----
BIGSmall	true
bIGSmall	true
BIGsmall	false
bigsmall	false
```

`(?i)` turns case-insensitive matching *on* partway through the pattern;
`(?-i)` turns it back *off* before the final `[S]`. This is exactly the shape
`RegexOptimizationRule`'s own literal-string helper explicitly refuses to touch:

```cpp
// src/optimizer/rule/regex_optimizations.cpp:64
if (regexp->parse_flags() & duckdb_re2::Regexp::FoldCase ||
    !(regexp->parse_flags() & duckdb_re2::Regexp::OneLine)) {
	// parse flags can turn on and off within a regex match, return no optimization
	// For now, we just don't optimize if these every turn on.
	ret.exists = false;
	return ret;
}
```

So this file is a **correctness regression test for the guard**, not a
demonstration of the optimization firing: it proves that a pattern whose
case-sensitivity changes mid-match still produces the right, case-sensitivity-
respecting answer — `'BIGsmall'` and `'bigsmall'` correctly fail to match (the
final, case-sensitive `[S]` rejects lowercase `s`) while `'BIGSmall'`/`'bIGSmall'`
correctly succeed — which is only guaranteed if the optimizer left this pattern
on the full RE2 engine rather than trying (and getting wrong) some partial
literal/`LIKE` reduction. It's worth being explicit about this distinction
because the file's name alone invites the opposite reading.

## `RegexOptimizationRule` and `RegexpReplaceExtractRule`

**Purpose, in one sentence (`RegexOptimizationRule`):** recognize that a
`regexp_matches(x, pattern)` call's compile-time-constant `pattern`, once
parsed by RE2, doesn't actually need general regex backtracking — it's either a
plain literal substring search, or a `^`/`$`-anchored/`.`/`.*`-only pattern that
`LikeOptimizationRule`'s own primitives (`prefix`/`suffix`/`contains`/`LIKE`)
already cover — and rewrite it down to whichever of those is equivalent.

**Purpose, in one sentence (`RegexpReplaceExtractRule`):** recognize that a
`regexp_replace(s, pattern, '\N')` call — replacing an *entire, whole-string*
match with nothing but one of its own capture groups — is exactly what
`regexp_extract(s, pattern, N, 'k')` already computes directly, and rewrite to
that instead of running a full replace.

### `RegexOptimizationRule`'s matcher and the `LikeString` machinery

```cpp
// src/optimizer/rule/regex_optimizations.cpp:19
RegexOptimizationRule::RegexOptimizationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	auto func = make_uniq<FunctionExpressionMatcher>();
	func->function = make_uniq<SpecificFunctionMatcher>("regexp_matches");
	func->policy = SetMatcher::Policy::SOME_ORDERED;
	func->matchers.push_back(make_uniq<ExpressionMatcher>());
	func->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
	root = std::move(func);
}
```

`SOME_ORDERED` (rather than `LikeOptimizationRule`'s plain `ORDERED`) is what
lets this matcher accept both the two-argument `regexp_matches(s, pattern)` and
the three-argument `regexp_matches(s, pattern, options)` forms without two
separate rule classes — `Apply` explicitly asserts on the children count instead
(`root.GetChildrenMutable().size() == 2 || ... == 3`).

Before any classification, `Apply` bails on collated `VARCHAR` operands:

```cpp
// src/optimizer/rule/regex_optimizations.cpp:168
for (idx_t i = 0; i < 2; i++) {
	const auto &type = root.GetChildren()[i]->GetReturnType();
	if (type.id() == LogicalTypeId::VARCHAR && !StringType::GetCollation(type).empty()) {
		return nullptr;
	}
}
```

This is worth contrasting with `LikeOptimizationRule`, which has no equivalent
check at all. The asymmetry isn't a bug in either rule: `LIKE` itself never
participates in DuckDB's collation machinery in the first place (`COLLATE`
affects ordering and the `=`/`<`/`>` comparison family, bound in
`bind_comparison_expression.cpp`'s `PushCollation`, not the `LIKE`/`GLOB`
function-call path), so rewriting a plain, uncollated `LIKE` into `prefix()`/
`suffix()`/`contains()`/`=` never changes what comparison semantics apply.
`RegexOptimizationRule`'s explicit guard reads as a deliberately conservative
choice for a case that may never actually arise given how collation is wired
up elsewhere — but it costs nothing to check, so it's there.

The classification machinery walks RE2's own parsed AST (`duckdb_re2::Regexp`),
not the pattern string directly, via a small accumulator:

```cpp
// src/optimizer/rule/regex_optimizations.cpp:29
struct LikeString {
	bool exists = true;
	bool escaped = false;
	string like_string = "";
};
```

Three regex-AST shapes are recognized, each mapping to a different `LikeString`
outcome:

1. **The whole pattern is one literal string** (RE2 op `kRegexpLiteralString`/
   `kRegexpLiteral`, i.e., no metacharacters survived parsing at all — this
   also catches a pattern that only *looks* like it has metacharacters, e.g.
   `'a\.a'`, where the backslash escapes the dot into a literal period). This
   converts **directly to `contains()`**, not `=` and not `LIKE` — because an
   unanchored `regexp_matches` call, by definition, succeeds if the pattern
   occurs *anywhere* in the string, which is exactly `contains()`'s semantics
   and is *not* the same thing as whole-string equality:

   ```cpp
   // src/optimizer/rule/regex_optimizations.cpp:202
   if (pattern.Regexp()->op() == duckdb_re2::kRegexpLiteralString ||
       pattern.Regexp()->op() == duckdb_re2::kRegexpLiteral) {
   	LikeString escaped_like_string = GetLikeStringEscaped(pattern.Regexp(), true);
   	// ...
   	auto contains = BindBuiltinScalarFunction(GetContext(), "contains", std::move(root.GetChildrenMutable()));
   ```

2. **A concatenation** (`kRegexpConcat`) of literal-string runs, `kRegexpAnyChar`
   (a single `.`), and `kRegexpStar`-of-`kRegexpAnyChar` (a `.*`), optionally
   bracketed by `kRegexpBeginText`/`kRegexpEndText` (`^`/`$`). This is handled by
   `LikeMatchFromRegex`, which walks the concatenation's children in order and
   builds up an equivalent `LIKE` pattern string piece by piece — `.` becomes
   `_`, `.*` becomes `%`, a literal run stays itself, and an *unanchored* end
   (no `^`/`$` at that side) gets an extra `%` inserted, since `regexp_matches`
   without anchors is a substring search and `LIKE` is not:

   ```cpp
   // src/optimizer/rule/regex_optimizations.cpp:97
   case duckdb_re2::kRegexpAnyChar:
   	if (cur_sub_index == 0) {
   		ret.like_string += "%";
   	}
   	ret.like_string += "_";
   	// ...
   case duckdb_re2::kRegexpStar:
   	if (subs[cur_sub_index]->nsub() == 1 && subs[cur_sub_index]->sub()[0]->op() == duckdb_re2::kRegexpAnyChar) {
   		ret.like_string += "%";
   		break;
   	}
   	ret.exists = false;
   	return ret;
   ```

   The resulting pattern string is packaged into a genuine `LIKE` ("`~~`")
   `BoundFunctionExpression` — deliberately, so that it re-enters
   classification. This is the moment the two rules chain: `RegexOptimizationRule`
   never itself decides whether the result should be `prefix`/`suffix`/`contains`
   or plain equality; it just hands `LikeOptimizationRule`'s own input shape
   back into the rewriter and lets that rule finish the job (mechanics of *why*
   this immediate re-matching happens — the bottom-up, per-node worklist inside
   `ExpressionRewriter::ApplyRules` — are covered in this chapter's sibling file
   on arithmetic/constant rules; the short version is that a node whose rewrite
   changes it is retried against the *entire* rule list before the walk moves
   on, regardless of where in the fixed 37-rule list each rule sits).

   An anchor must sit at the edge of the pattern. A `LIKE` pattern shows an
   anchor only by *not* having a `%` at that end. So it cannot express an
   anchor in the middle. If `LikeMatchFromRegex` simply skipped
   `kRegexpBeginText` and `kRegexpEndText` wherever they appeared, a pattern
   like `'a^b'` could come out as a `LIKE` pattern such as `'%ab%'`, which
   matches `'xab'`. But `a^b` can never match: `^` means "start of text",
   and it cannot come after an `a`. So the code refuses such patterns:

   ```cpp
   // src/optimizer/rule/regex_optimizations.cpp:137
   case duckdb_re2::kRegexpBeginText:
   	// anchors are represented by the absence of a leading / trailing '%', so they must be at the edges
   	if (cur_sub_index != 0) {
   		ret.exists = false;
   		return ret;
   	}
   	break;
   case duckdb_re2::kRegexpEndText:
   	if (cur_sub_index + 1 != num_subs) {
   		ret.exists = false;
   		return ret;
   	}
   	break;
   ```

   The test `test/optimizer/regex_optimizer_anchors.test` checks it:

   ```sql
   -- test/optimizer/regex_optimizer_anchors.test
   CREATE TABLE t AS SELECT * FROM (VALUES ('ab'), ('a'), ('b'), ('xab')) t(s);

   -- anchors in the middle of the pattern can never match here
   SELECT s, regexp_matches(s, 'a^b'), regexp_matches(s, 'a$b'), regexp_matches(s, 'a\Ab') FROM t ORDER BY s
   -- every row: false  false  false
   ```

   Also, `Apply` evaluates the pattern with `TryEvaluateScalar`. A pattern
   that raises an error when folded is left for execution, as in
   `LikeOptimizationRule`.

3. **Anything else** — a character class (`[A]`, `[A-Z]`), an alternation, a
   backreference, a repetition other than `*` — falls through every `case` to
   `default: ret.exists = false;` and the pattern is left as a genuine
   `regexp_matches` call, executed by RE2 at runtime exactly as written.

### Walking `test/optimizer/regex_optimizer.test`

The file is 255 lines and covers considerably more ground than the flagship
`like_optimizer.test`, because a regex pattern has far more shapes than a
`LIKE` pattern does. A representative sample, grouped by what each is actually
proving:

**Plain literal patterns become `contains`, regardless of length:**

```sql
-- label: regexconstantpattern
EXPLAIN SELECT regexp_matches(s, 'aa') FROM test
EXPLAIN SELECT contains(s, 'aa') FROM test

-- label: regexconstantsinglechar
EXPLAIN SELECT regexp_matches(s, 'a') FROM test
EXPLAIN SELECT contains(s, 'a') FROM test
```

**Anchored patterns become `prefix`/`suffix`, matching the previous section's
downstream expectation exactly:**

```sql
explain analyze SELECT regexp_matches(s, '^a'), s FROM TEST;
----
analyzed_plan	<REGEX>:.*prefix\(s, 'a'\).*

explain analyze SELECT regexp_matches(s, 'a$'), s FROM TEST;
----
analyzed_plan	<REGEX>:.*suffix\(s, 'a'\).*
```

**A fully anchored, wildcard-only pattern becomes a full `LIKE` string, which
then further collapses:**

```sql
explain analyze SELECT regexp_matches(s, '^a.*b$', 's'), s FROM TEST;
----
analyzed_plan	<REGEX>:.*"~~"\(s, 'a%b'\).*
```

(the `'s'` third argument is `regexp_matches`'s "dot matches newline" option,
discussed below — it matters here specifically because it's what makes `.*`
safe to treat as `LIKE`'s unrestricted `%`.)

**Escaped literal metacharacters survive correctly through the literal-string
path:**

```sql
explain analyze select regexp_matches(s, 'a\.a'), s from test;
----
analyzed_plan	<REGEX>:.*contains\(s, 'a.a'\).*

select s from test where regexp_matches(s, 'a\.a');
----
a.a
```

`'a\.a'` parses to the plain three-character literal `a.a` (the backslash
escapes RE2's own dot-metacharacter), so it takes the literal-string path above
and becomes `contains(s, 'a.a')` — matching the row `'a.a'` but *not* `'aXa'`,
which an *unescaped* `.` would have matched. The test's correctness assertion
(`select s from test where regexp_matches(s, 'a\.a')` returning only `'a.a'`,
not `'aXa'`-shaped rows) is really checking that the rewrite preserved the
escape rather than accidentally treating the dot as a wildcard.

**A single unescaped `.` needs the `'s'` (dot-matches-newline) option before it
can safely become `LIKE`'s `_`:**

```sql
explain analyze SELECT regexp_matches(s, 'aaa.'), s FROM TEST;
----
analyzed_plan	<REGEX>:.*regexp_matches\(s, 'aaa.'\).*     -- NOT rewritten

explain analyze SELECT regexp_matches(s, 'aaa.', 's'), s FROM TEST;
----
analyzed_plan	<REGEX>:.*"~~"\(s, '%aaa_%'\).*             -- rewritten, with the 's' option
```

This is a genuinely important edge case and worth spelling out precisely: RE2's
`.` metacharacter, *without* the dot-matches-newline option, matches any
character **except** a newline, whereas `LIKE`'s `_` wildcard in DuckDB always
matches exactly one character, including a newline. Those are not the same
predicate, and converting the first pattern into a `LIKE` string using `_`
would silently change behavior for any input string containing an embedded
`\n`. The rule's classification logic (walked through above) never has to
special-case this explicitly in `LikeMatchFromRegex`'s own switch — RE2 itself
represents `.` differently in its parsed AST depending on whether the
dot-matches-newline option was requested; without the option, RE2's `.` is not
the `kRegexpAnyChar` op this switch's `case duckdb_re2::kRegexpAnyChar:` branch
is looking for, so it falls straight to the `default: ret.exists = false;`
branch and the whole rewrite is declined for that pattern. The two `EXPLAIN`
outputs above are the directly observable, source-explainable consequence of
that AST difference, and the test file pins it down explicitly enough that a
future change accidentally treating the two `.` forms as interchangeable would
fail immediately.

**Newline-sensitivity is exercised end-to-end, not just at the plan level:**

```sql
-- inserts 'aaa\naaa'
select count(s) from test where regexp_matches(s, 'aaa.');       -- 1 (unanchored contains('aaa') still matches)
select count(s) from test where regexp_matches(s, 'aaa.');       -- wait, see below
select count(s) from test where regexp_matches(s, 'aaa.');
```

The file actually runs three related counts here — `regexp_matches(s, 'aaa')`
(1, plain contains), `regexp_matches(s, 'aaa.')` without `'s'` (0 — the `.`
can't match across the embedded newline, and there's no fifth character after
either `'aaa'` run to match `.` against on the same line), and
`regexp_matches(s, 'aaa.', 's')` (1 — with dot-matches-newline on, `.` reaches
across the `\n`). These three numbers are exactly the runtime evidence for the
plan-level distinction shown above, and confirm the rewritten `contains`/`LIKE`
forms still compute the newline-sensitive answer correctly when they do fire.

**Options are stripped once absorbed:** every rewrite above that starts from a
three-argument `regexp_matches(s, pattern, options)` call removes the third
child before rebuilding the replacement expression —

```cpp
// src/optimizer/rule/regex_optimizations.cpp:210
if (root.GetChildrenMutable().size() == 3) {
	root.GetChildrenMutable().pop_back();
}
```

— "so the new Contains Expression can be matched for other optimizers," per
the inline comment; a stray options argument sitting on a `contains()` call
would otherwise be nonsensical (`contains` has no options) and would also block
`ContainsToInClauseRule`'s and `EmptyNeedleRemovalRule`'s own matchers, both of
which expect a plain two-argument `contains(x, y)` shape.

### Walking `test/optimizer/regex_optimizer_coverage.test`

This file's name is a direct hint about its own history, and reading its
content confirms the hint: it is fifty lines entirely devoted to **character
classes** — `[A]`, `[AS]`, `[S]`, `[A][S]`, `[AAAA]`, `[A-A]`, and the same
`[A-A]` again under `(?i)`/`(?s)`/`(?m)` flag variants — none of which
correspond to any `case` in `LikeMatchFromRegex`'s switch statement at all
(character classes compile to RE2's `kRegexpCharClass` op, not
`kRegexpLiteral`/`kRegexpAnyChar`/`kRegexpStar`). Every one of these patterns
therefore falls to the `default:` branch and is left as a genuine
`regexp_matches` call — this file is not exercising a rewrite at all; it's a
**correctness hardening suite**, verifying that RE2's own execution still
produces the right answer for these AST shapes even though the optimizer
declines to touch them. Every query shares the same `correct_result` label
(the same-label idiom from the top of this file), which is a neat trick given
the specific test data (`'FLASH'`, `'TOAST'`, `'BELL'`): a single-character
range `[A-A]`, a redundant-repeat class `[AAAA]`, the bare literal `'A'`, and
`contains(word, 'AS')` all happen to partition this particular three-row table
identically (`{FLASH, TOAST}` match, `BELL` doesn't), so pinning all of them to
one shared expected output is really asserting "these syntactically very
different regex forms are semantically equivalent to each other and to a plain
contains, on this data" in one compact block, rather than writing out the same
three-row result eight separate times. Read together with the timing of its
name, this file reads as a regression suite added after `regex_optimizer.test`
had already shipped — evidence that the character-class corner of RE2's AST
either caused a real bug once, or was flagged as under-tested and hardened
preemptively, rather than something this book can attribute to a specific
historical incident with certainty.

### Confirming the `RegexOptimizationRule` → `LikeOptimizationRule` chain from an entirely different angle

A standalone C++ unit test suite, `bound_function_identity.cpp`, exists
specifically to check that an optimizer rewrite really did introduce the
function its comment claims, by name, rather than merely producing output that
happens to look similar. Its regex test case states the two-rule chain this
section has been building up to as directly as it's possible to state it:

```cpp
// test/optimizer/bound_function_identity.cpp:158
TEST_CASE("Regex rewrites keep the definition of the built-in they introduce", "[optimizer][function_identity]") {
	// a literal regex becomes contains()
	RequireIdentityFunction(PlanIdentityFunctions(con, "SELECT regexp_matches(s, 'abc') FROM strings"), "contains");
	// an anchored regex becomes a LIKE, which the LIKE rule then turns into prefix()
	RequireIdentityFunction(PlanIdentityFunctions(con, "SELECT regexp_matches(s, '^abc') FROM strings"), "prefix");
}
```

`regexp_matches(s, '^abc')` never once appears as a `LIKE`/`~~` call in the
final, optimized plan — `RegexOptimizationRule` produces the `LIKE` shape, and
in the very same tree-walk (per `ExpressionRewriter::ApplyRules`'s per-node
retry loop), `LikeOptimizationRule` immediately reclassifies it as
`PatternIsPrefix` and collapses it straight through to `prefix(s, 'abc')`
before the optimizer ever moves past pass #1. The test's own comment states
this exact mechanism in plain English, independently confirming the reading
this section built up from the two rules' source.

### `RegexpReplaceExtractRule`

```cpp
// src/optimizer/rule/regex_optimizations.cpp:267
// Rewrites `regexp_replace(s, P, '\N')` into `regexp_extract(s, P, N, 'k')` when parsed RE2 anchors
// prove that any match spans the whole input. The original pattern is preserved; any regexp_extract
// pattern simplification belongs in a separate optimization.
unique_ptr<Expression> RegexpReplaceExtractRule::Apply(...) {
	// ...
	if (!bind_data.constant_pattern || bind_data.global_replace) {
		return nullptr;
	}
	if (bind_data.options.literal()) {
		return nullptr;
	}
	// ...
	if (replace_str.size() != 2 || replace_str[0] != '\\' || replace_str[1] < '0' || replace_str[1] > '9') {
		return nullptr;
	}
	// ...
	if (!compiled.ok() || !RegexpHasWholeTextAnchors(compiled.Regexp())) {
		return nullptr;
	}
	if (group_index > compiled.NumberOfCapturingGroups()) {
		// leave reporting the invalid back-reference to regexp_replace
		return nullptr;
	}
```

Reading the guards in order tells the whole story of what shape qualifies:
the pattern must be a compile-time constant (not computed per row); the call
must not be a global (`'g'` option) replace, since `regexp_extract` only ever
returns one match, not every match; the replacement string must be *exactly* a
bare backreference like `'\1'` — `'x\1'`, `'\1x'`, and `'\1\2'` are all
rejected by the size/character checks, since none of those can be represented
by `regexp_extract`'s single-capture-group return value; and, via
`RegexpHasWholeTextAnchors`, the compiled pattern must be a concatenation whose
first and last children are literally `^` and `$` — i.e., any match of this
pattern necessarily consumes the *entire* input string, which is exactly the
condition under which "replace the whole match with capture group N" and
"extract capture group N" compute the same string.

The last guard protects an error. `'\2'` names capture group 2. If the
pattern has only one group, `regexp_replace` must fail with *"Invalid
replacement string for regexp_replace"*. A rewrite to `regexp_extract` would
hide that error and return a result instead. So the rule leaves such calls
alone, and `regexp_replace` reports the error.
`test/optimizer/regex_optimizer_anchors.test` checks it:

```sql
-- test/optimizer/regex_optimizer_anchors.test
statement error
SELECT regexp_replace('aaa', '^(a+)$', '\2')
----
Invalid replacement string for regexp_replace

query II
SELECT regexp_replace('aaa', '^(a+)$', '\1'), regexp_replace('aaa', '^(a+)$', '\0')
----
aaa	aaa
```

There is no dedicated `test/optimizer/*.test` file for this rule's plans — a
search of the whole `test/` tree for `RegexpReplaceExtractRule` and for a
`regexp_replace` optimizer test comes up empty. (`regex_optimizer_anchors.test`,
shown above, checks results for this rule's edge cases, but not its plans.)
The plan check lives in an ordinary function test instead. The comment at
line 120 of `test/sql/function/string/regex_replace.test` says *"The
optimizer rewrites these into regexp_extract with no_match_returns_input
set"*, and `test/sql/function/string/regex_replace.test:124` runs

```sql
SELECT regexp_replace(s, '^/([^/]+)/.*$', '\1') FROM (VALUES ...)
```

which satisfies every one of the guards above — a constant, non-global,
non-literal, `^...$`-anchored pattern with a bare `'\1'` replacement — so this
query is rewritten to `regexp_extract(s, '^/([^/]+)/.*$', 1, 'k')` before it
ever executes, and the test's expected output is also a regression pin for
this rule's correctness. The block after line 160 of the same file observes
the rewrite directly in the optimized plan. It also shows that the regex
options are kept: the `'i'` (case-insensitive) option of `regexp_replace`
becomes `'ik'` in the `regexp_extract` call:

```sql
-- test/sql/function/string/regex_replace.test
# Options are retained when a non-constant input is rewritten to regexp_extract.
query II
EXPLAIN SELECT regexp_replace(s, '^(foo)-(.*)$', '\2', 'i')
FROM (VALUES ('FOO-bar')) t(s);
----
logical_opt	<REGEX>:(?s).*regexp_extract.*'ik'.*
```

## `StringPrefixRule` and `InstrPrefixRule` — the same payoff from different syntax

**Purpose, in one sentence (`StringPrefixRule`):** recognize that a
string-extraction call (`left`, `substring`/`substr`, `array_slice`/`list_slice`,
or `[start:end]` slice syntax) compared for equality against a constant, when
the extraction always starts at the string's first character, is really asking
"does this string start with this constant" — and rewrite it to `prefix()`, or
to a plain `=`/`false`/`NULL` when the extraction length and the constant's
length disagree.

**Purpose, in one sentence (`InstrPrefixRule`):** recognize that
`instr`/`position`/`strpos(s, needle) = 1` — "the needle's first occurrence is
at position 1" — is exactly "the string starts with the needle," and rewrite it
directly to `prefix(s, needle)`.

### The two rules, and why they're two rules and not one

```cpp
// src/optimizer/rule/string_prefix.cpp:25
StringPrefixRule::StringPrefixRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	auto op = make_uniq<ComparisonExpressionMatcher>();
	op->expr_type = make_uniq<SpecificExpressionTypeMatcher>(ExpressionType::COMPARE_EQUAL);
	op->policy = SetMatcher::Policy::UNORDERED;

	auto func = make_uniq<FunctionExpressionMatcher>();
	func->function =
	    make_uniq<ManyFunctionMatcher>(identifier_set_t {"left", "array_slice", "list_slice", "substring", "substr"});
	func->type = make_uniq<TypeMatcherId>(LogicalTypeId::VARCHAR);
	func->matchers.push_back(make_uniq<ExpressionMatcher>());
	func->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
	func->policy = SetMatcher::Policy::SOME_ORDERED;   // allows the 2- or 3-argument forms
	op->matchers.push_back(std::move(func));

	auto constant = make_uniq<ConstantExpressionMatcher>();
	constant->type = make_uniq<TypeMatcherId>(LogicalTypeId::VARCHAR);
	op->matchers.push_back(std::move(constant));

	root = std::move(op);
}
```

`ComparisonExpressionMatcher`'s `Policy::UNORDERED` is the detail that makes
both `left(s, 5) = '09999'` and `'09999' = left(s, 5)` match identically — the
rule doesn't care which side of `=` the function call is on, which is directly
exercised in `string_prefix.test` (see below) and matches the general
`InstrPrefixRule` behavior too (`instr_zonemap_pruning.test`'s
`1 = instr(s, 'mmm')` case).

`StringPrefixRule::Apply` is the more involved of the two, because unlike a
`LIKE` pattern (which either *is* or *isn't* a prefix), an extraction function
carries an explicit **length**, and that length has to be reasoned about
against the constant's own length in three distinct ways:

```cpp
// src/optimizer/rule/string_prefix.cpp:150
const auto constant_length = Length<string_t, int64_t>(...);  // counts characters, not bytes

// The constant is longer than the extracted prefix, so the comparison can only be FALSE
if (constant_length > num_characters) {
	return ExpressionRewriter::ConstantOrNull(GetContext(), std::move(children[0]), Value::BOOLEAN(false));
}
// A prefix extraction can only return fewer than n characters if it returns the entire string.
if (constant_length < num_characters) {
	return RewriteAsEquality(std::move(children[0]), constant.GetValue());
}
// Convert to prefix comparison
vector<unique_ptr<Expression>> prefix_children;
prefix_children.emplace_back(std::move(children[0]));
prefix_children.emplace_back(make_uniq<BoundConstantExpression>(constant.GetValue()));
return BindBuiltinScalarFunction(GetContext(), PrefixFun::Name, std::move(prefix_children));
```

If the constant is *longer* than the number of characters being extracted,
`left(s, 5)` can never equal it no matter what `s` is (an unconditional
`FALSE`, wrapped in `ConstantOrNull` to preserve `s`'s own `NULL`-ness — `left(NULL,
5) = 'toolong'` still has to be `NULL`, not `FALSE`). If the constant is
*shorter*, the only way `left(s, 5) = 'abc'` (a 3-character constant against a
5-character extraction) can hold is if `s` itself is only 3 characters long —
i.e., `left` returned the *whole string*, not a genuine 5-character prefix —
so the comparison degenerates to plain string equality on `s` itself, dropping
the extraction call entirely. Only when the lengths **match exactly** does the
comparison become a genuine `prefix()` call. `Length<string_t, int64_t>` counts
Unicode characters, not bytes, which is what makes the rule correct for
multi-byte input (walked below with the `'🦆ab'` case) rather than silently
undercounting a 4-byte emoji as more than one "character" of prefix length.

Every extraction function needs its own small dance to arrive at
`num_characters`: `left(s, N)` — `N` directly, with its own bounds check
mirroring `left`'s own documented range limit; `substring(s, start, len)` —
only when `start == 1` (`substring(s, 2, 3) = 'abc'` is *not* a prefix
comparison at all, it's a mid-string comparison, and the rule correctly
declines by returning `nullptr`), with the 2-argument `substring(s, 1)` form
(no length at all) short-circuiting straight to plain equality since it always
returns the entire tail from position 1 onward; `array_slice`/`list_slice(s,
begin, end)` and the `s[begin:end]` slice sugar — `begin` must be `0` or `1`
(DuckDB's slice syntax is famously both 0- and 1-indexed depending on which
form is used), and an open-ended upper bound (`s[1:]`, encoded as a `LIST`-typed
`end` value rather than an integer) also degenerates straight to equality,
exactly like `substring(s, 1)`.

`InstrPrefixRule` needs none of this — an `instr`/`position`/`strpos` call
already *is* a needle, and asking "is its first match at position 1" is
already exactly the question `prefix()` answers, with no length arithmetic
required at all:

```cpp
// src/optimizer/rule/string_prefix.cpp:171
unique_ptr<Expression> InstrPrefixRule::Apply(...) {
	// If requested position is 1, rewrite as prefix comparison.
	if (position.GetValue().IsNull() || position.GetValue().GetValue<int64_t>() != 1) {
		return nullptr;
	}
	// prefix with a NULL needle binds to a NULL constant, not to a function
	if (needle.GetValue().IsNull()) {
		return nullptr;
	}
	// ...
	return BindBuiltinScalarFunction(GetContext(), PrefixFun::Name, std::move(prefix_children));
}
```

The `NULL`-needle guard matters because binding `prefix(x, NULL)` does not
give a `prefix` function call. It folds to a plain `NULL` constant. The rest
of the rule expects a function, so a `NULL` needle would break it. The guard
leaves that case alone, and the normal `instr` evaluation returns `NULL`. The
test `test/optimizer/instr_prefix_null_needle.test` checks both sides:

```sql
-- test/optimizer/instr_prefix_null_needle.test
query I
SELECT instr(x, CASE WHEN true THEN NULL ELSE x END) = 1 FROM (VALUES ('a'), ('b'), (NULL)) t(x);
----
NULL
NULL
NULL

# a needle that is not NULL is still rewritten to prefix
query II
EXPLAIN SELECT instr(x, 'ab') = 1 FROM s;
----
physical_plan	<REGEX>:.*prefix.*
```

The `CASE WHEN true THEN NULL ...` needle folds to `NULL` only during
optimization, so the binder could not catch it earlier.

This is the precise reason these are two separate rule classes rather than one
generalized "prefix-shaped comparison" rule: `StringPrefixRule` is fundamentally
about reconciling two independent *lengths* (the extraction's declared length
versus the constant's actual length), while `InstrPrefixRule` is about a
*position* check that only has one interesting value (`1`) and nothing to
reconcile once that value matches.

### Walking `test/optimizer/string_prefix.test`

This is a large, table-driven test (a synthetic 10-million-row table with
zero-padded 8-digit strings, so zonemap pruning has something real to bite
into) that walks every one of `StringPrefixRule`'s five target functions plus
the `[:]` slice syntax, each showing the same shape of evidence:

**The `left` case, with the reversed-equality variant and the zonemap payoff
made directly visible:**

```sql
EXPLAIN ANALYZE SELECT count(*) FROM t WHERE left(s, 5) = '09999';
----
analyzed_plan	<REGEX>:.*Filters: s >= '09999'.*Row Groups Scanned: 1 / 82.*

-- reversed equality is rewritten identically
EXPLAIN ANALYZE SELECT count(*) FROM t WHERE '09999' = left(s, 5);
----
analyzed_plan	<REGEX>:.*Filters: s >= '09999'.*Row Groups Scanned: 1 / 82.*
```

Notice this is the *same* downstream mechanism as `LikeOptimizationRule`'s
`prefix()` output — `filter_combiner.cpp`'s `TryPushdownPrefixFilter`, from the
previous section — being fed by a totally different piece of surface syntax.
`left(s, 5) = '09999'` and `s LIKE '09999%'` both end up, after their
respective expression-rewriter rules run, as the identical `prefix(s, '09999')`
call, and both get identical zonemap treatment from that point on: **one** of
82 row groups scanned instead of all 82, because the plan-level filter turned
into a genuine `s >= '09999'` (`AND s < '0999:'`, the "next legal UTF-8 string")
range the storage layer's min/max statistics can immediately rule 81 row
groups out with.

**The length-mismatch branches, exactly as the source's three-way logic
predicts:**

```sql
-- exact length match: genuine prefix comparison, zonemap-prunable
EXPLAIN ANALYZE SELECT count(*) FROM t WHERE substring(s, 1, 5) = '09999';
----
analyzed_plan	<REGEX>:.*Filters: s >= '09999'.*Row Groups Scanned: 1 / 82.*

-- constant shorter than the extraction: degenerates to whole-string equality
EXPLAIN ANALYZE SELECT count(*) FROM t WHERE substring(s, 1, 9) = '09999000';
----
analyzed_plan	<REGEX>:.*s = '09999000'.*Row Groups Scanned: 1 / 82.*

-- constant longer than any prefix of 'abc'/'abcdef'/'xabc'/'🦆ab': always false
SELECT count(*) FROM prefix_cases WHERE substring(s, 1, 2) = 'abc';
----
0
```

Note the second case still prunes to 1 row group even though it's now a plain
`=` filter, not a `prefix()` range — DuckDB's zonemap pruning handles equality
filters directly too, so the rewrite's *choice* between `prefix()` and `=` here
is really about picking the semantically correct primitive, not specifically
about unlocking pruning that wouldn't otherwise exist; the pruning payoff is
shared by both outcomes.

**Multi-byte correctness, using the book's now-familiar duck emoji:**

```sql
INSERT INTO prefix_cases VALUES ('abc'), ('abcdef'), ('xabc'), ('🦆ab'), (NULL);
-- ...
SELECT count(*) FROM prefix_cases WHERE left(s, 2) = '🦆a';
----
1
```

`left(s, 2)` extracts two *characters*: for `'🦆ab'` that's `🦆` (a
four-byte UTF-8 codepoint) followed by `a` — two characters, matching the
two-character constant `'🦆a'` exactly in length, so this takes the genuine
`prefix()` branch rather than either degenerate branch, and the `Length<
string_t, int64_t>` character-counting (not byte-counting) is exactly what
keeps this correct.

**NULL propagation, checked at every input position:**

```sql
SELECT (left(s, 2) = 'abc') FROM (VALUES (NULL)) t(s);        ----> NULL
SELECT (left(s, 3) = NULL) FROM (VALUES ('abc')) t(s);        ----> NULL
SELECT (left(s, NULL) = 'abc') FROM (VALUES ('abc')) t(s);    ----> NULL
```

**A dynamic (non-constant) length blocks the rewrite entirely, and the test
explicitly checks the original expression survives untouched:**

```sql
SELECT count(*) FROM (VALUES ('abcx', 3), ('abcx', 2)) t(s, n) WHERE substring(s, 1, n) = 'abc';
----
1
```

`n` is a column, not a foldable constant — `children[2]->GetExpressionClass()
!= ExpressionClass::BOUND_CONSTANT` catches exactly this and returns `nullptr`,
leaving the original, per-row `substring` call to execute normally. The
expected result (`1`, matching only the `n=3` row) is itself the correctness
check that the un-rewritten expression still computes the right thing.

**Errors are preserved, not silently swallowed by the rewrite:**

```sql
statement error
SELECT * FROM (VALUES ('a')) t(s) WHERE left(s, 4294967296) = 'a';
----
Out of Range Error: Substring length outside of supported range
```

The rule's own bounds check (`num_characters < 0 || num_characters >
NumericLimits<uint32_t>::Maximum()`) deliberately returns `nullptr` rather than
trying to handle an out-of-range length itself, letting the original `left`
call run and throw its own, pre-existing error — the rewrite doesn't
paper over a case that should legitimately fail.

### Walking `test/sql/optimizer/instr_zonemap_pruning.test`

This file lives outside `test/optimizer/` (under `test/sql/optimizer/`
instead) but is squarely `InstrPrefixRule`'s test, built around a table with
three literal-string bands (`'aaa' || i`, `'mmm' || i`, `'zzz' || i`) spread
across exactly three row groups:

```sql
EXPLAIN ANALYZE SELECT count(*) FROM instr_pruning WHERE instr(s, 'mmm') = 1;
----
analyzed_plan	<REGEX>:.*Row Groups Scanned: 1 / 3.*

-- Equality matching is unordered
EXPLAIN ANALYZE SELECT count(*) FROM instr_pruning WHERE 1 = instr(s, 'mmm');
----
analyzed_plan	<REGEX>:.*Row Groups Scanned: 1 / 3.*

-- A prefix outside all row-group ranges does not scan any row groups
EXPLAIN ANALYZE SELECT count(*) FROM instr_pruning WHERE instr(s, 'qqq') = 1;
----
analyzed_plan	<REGEX>:.*Row Groups Scanned: 0 / 3.*
```

The last case is worth pausing on: `'qqq'` never appears anywhere in the
table, and the rewritten `prefix(s, 'qqq')` filter's derived range (`s >=
'qqq' AND s < 'qqr'`) falls entirely between the `'mmm'`-band and `'zzz'`-band
row groups' min/max statistics — so the storage layer proves zero row groups
can possibly qualify and skips reading the table altogether, without ever
running a single comparison against real data. `test/optimizer/
bound_function_identity.cpp` independently confirms this same rule by name,
in the same style as its regex test case shown earlier:

```cpp
// test/optimizer/bound_function_identity.cpp:171
TEST_CASE("String prefix rewrites keep the definition of prefix()", "[optimizer][function_identity]") {
	RequireIdentityFunction(PlanIdentityFunctions(con, "SELECT left(s, 3) = 'abc' FROM strings"), "prefix");
	RequireIdentityFunction(PlanIdentityFunctions(con, "SELECT instr(s, 'abc') = 1 FROM strings"), "prefix");
}
```

## `EmptyNeedleRemovalRule` and `NoopReplaceRemovalRule` — degenerate search cases

**Purpose, in one sentence (`EmptyNeedleRemovalRule`):** recognize that
`prefix`/`suffix`/`contains(x, '')` and string concatenation with a foldable
empty-string operand have fixed, needle-independent answers, and replace them
with that fixed answer without ever touching `x`'s actual bytes.

**Purpose, in one sentence (`NoopReplaceRemovalRule`):** recognize that
`replace(s, needle, needle)` — replacing a substring with itself — can never
change `s`, and replace the whole call with `s` directly.

### The degenerate-case reasoning, stated precisely

What *should* `contains(s, '')` mean? Read literally against the standard
definition of substring search, the empty string occurs at every position in
every string (position 0 in an empty haystack, position 0 through length in a
non-empty one) — so `contains(s, '')` is `TRUE` for every non-`NULL` `s`,
unconditionally, and `PREFIX`/`SUFFIX` follow the identical reasoning for the
same reason (the empty string is trivially a prefix and a suffix of anything).
The only thing that can still vary is whether `s` itself is `NULL` — DuckDB's
scalar functions follow standard `NULL`-in-`NULL`-out propagation, so
`prefix(NULL, '')` is `NULL`, not `TRUE`. That's the entire content of the
rule's fixed answer, and the source states it exactly this way in its own
comment:

```cpp
// src/optimizer/rule/empty_needle_removal.cpp:81
// PREFIX('xyz', '') is TRUE
// PREFIX(NULL, '') is NULL
// so rewrite PREFIX(x, '') to TRUE_OR_NULL(x)
if (needle_string.empty()) {
	return ExpressionRewriter::ConstantOrNull(GetContext(), std::move(root.GetChildrenMutable()[0]),
	                                          Value::BOOLEAN(true));
}
```

`ConstantOrNull` (`src/optimizer/expression_rewriter.cpp:87`) builds exactly
this shape: `TRUE` if `x IS NOT NULL`, `NULL` otherwise — which means the
*value* of `x` genuinely never needs to be inspected at all, only its
null-map bit. That's the concrete performance content of this rule: an
`EmptyNeedleRemovalRule` rewrite turns a per-row string-scanning function call
into a per-row null-check, which every execution engine can do at a fraction
of the cost.

```cpp
// src/optimizer/rule/empty_needle_removal.cpp:35
EmptyNeedleRemovalRule::EmptyNeedleRemovalRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	identifier_set_t functions = {"prefix", "contains", "suffix", "||"};
	func->function = make_uniq<ManyFunctionMatcher>(functions);
	// ...
}
unique_ptr<Expression> EmptyNeedleRemovalRule::Apply(...) {
	if (root.Function().GetName() == "||") {
		return TryRemoveEmptyStringConcat(GetContext(), root);
	}
	auto &input_expr = bindings[1].get();
	if (input_expr.GetReturnType().InternalType() != PhysicalType::VARCHAR) {
		return nullptr;
	}
	// ...
}
```

The `VARCHAR` guard on the first argument matters because `contains` has a
*list* overload too: `contains(['a', 'b'], '')` asks "does the list have an
element equal to `''`?" Folding that call to `TRUE` would be wrong: the list
has no empty string. So the rule only fires when the first argument is
stored as a string. `test/sql/optimizer/expression/test_empty_needle_dead_branch.test`
guards against this bug (DuckDB issue #28562):

```sql
-- test/sql/optimizer/expression/test_empty_needle_dead_branch.test
# issue #28562: contains(list_column, '') should not be folded to TRUE by EmptyNeedleRemovalRule
query I
SELECT contains(['a', 'b'], '');
----
false

query I
SELECT contains(['a', '', 'b'], '');
----
true
```

`"||"` (string concatenation) gets a different helper entirely, because an
empty *concatenation operand* doesn't fold to a fixed boolean — it folds to
"the other operand, unchanged":

```cpp
// src/optimizer/rule/empty_needle_removal.cpp:13
static unique_ptr<Expression> TryRemoveEmptyStringConcat(ClientContext &context, BoundFunctionExpression &root) {
	// ...
	for (idx_t child_idx = 0; child_idx < children.size(); child_idx++) {
		if (!children[child_idx]->IsFoldable()) {
			continue;
		}
		Value value;
		if (!ExpressionExecutor::TryEvaluateScalar(context, *children[child_idx], value)) {
			// leave the error to execution, which may never evaluate it
			continue;
		}
		if (value.IsNull() || value.type().id() != LogicalTypeId::VARCHAR || !StringValue::Get(value).empty()) {
			continue;
		}
		return Expression::PreserveReturnType(root.GetReturnType(), std::move(children[1 - child_idx]));
	}
	return nullptr;
}
```

`s || ''` and `'' || s` both become plain `s` — `children[1 - child_idx]`
picks "whichever operand wasn't the empty one" regardless of which side it's
on.

Both helpers fold their constant with `TryEvaluateScalar`. If folding
fails, the error is left to execution, *"which may never evaluate it"*. The
same test file shows this:

```sql
-- test/sql/optimizer/expression/test_empty_needle_dead_branch.test
query I
SELECT CASE WHEN false THEN 'x' || ''::DOUBLE END;
----
NULL

statement error
SELECT 'x' || ''::DOUBLE;
----
Conversion Error
```

The first `||` is in a branch that never runs, so there is no error. The
second one runs, so the conversion error appears at execution time.

### Walking `test/optimizer/needle_optimizer.test`

The file is short and uses the same-label idiom to prove the exact rewrite:

```sql
-- both share label: no_output (identical plans)
EXPLAIN SELECT PREFIX(S, '') FROM test
EXPLAIN SELECT CONTAINS(S, '') FROM test
```

Both `PREFIX(S, '')` and `CONTAINS(S, '')` produce byte-identical optimized
plans — which is exactly what the shared `ConstantOrNull(GetContext(), s,
Value::BOOLEAN(true))` construction predicts, since neither rewrite's output
retains any trace of which original function name it came from. Worth flagging
plainly: the source's function-name set also includes `"suffix"`, but this
test file never exercises `SUFFIX(S, '')` directly — a real, if minor, test
coverage gap this book's source reading closes by inference from the shared
code path rather than a `.test` file proving it.

### `NoopReplaceRemovalRule`, and an honest gap

```cpp
// src/optimizer/rule/empty_needle_removal.cpp:101
unique_ptr<Expression> NoopReplaceRemovalRule::Apply(...) {
	// ...
	if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), needle_expr, needle) ||
	    !ExpressionExecutor::TryEvaluateScalar(GetContext(), replacement_expr, replacement)) {
		return nullptr;
	}
	if (needle.IsNull() || replacement.IsNull() || ... ) {
		return nullptr;
	}
	if (StringValue::Get(needle) != StringValue::Get(replacement)) {
		return nullptr;
	}
	return Expression::PreserveReturnType(root.GetReturnType(), std::move(root.GetChildrenMutable()[0]));
}
```

Two details worth being precise about. First, the check compares *evaluated
values*, not syntactic expressions — `replace(s, 'ab', 'a' || 'b')` matches
just as well as `replace(s, 'ab', 'ab')`, since both the needle and the
replacement only need to be independently foldable and equal once folded, not
written identically. Second, and worth stating candidly: **no `.test` file in
this checkout exercises `NoopReplaceRemovalRule` at all** — neither
`test/optimizer/needle_optimizer.test` (which, despite its name and its
neighboring position in `empty_needle_removal.cpp`, only covers the
empty-needle side) nor any other file this book's search reached mentions the
rule or an SQL query shaped to trigger it. This is stated plainly rather than
guessed around, in keeping with this book's sourcing policy: the rule's logic
is unambiguous from its source, but its test coverage, specifically, appears
to be genuinely absent in this snapshot of the tree.

## `ContainsToInClauseRule` — building the shape a later pass needs

**Purpose, in one sentence:** recognize a `contains`/`list_contains`/
`list_has`/`array_contains`/`array_has` call whose *container* argument is a
foldable constant list, and rewrite it into an explicit `x IN (v1, v2, ...)`
expression — not because `IN` is faster to *evaluate* than a function call by
itself, but because only the explicit `IN` shape is visible to the later,
plan-level `IN_CLAUSE` pass (§7.4 #9 in the overview; full mechanics in this
chapter's sibling file on the regex-range and `IN`-clause passes), which is
what actually decides whether a long enough list is worth turning into a hash
join.

### Where the input shape comes from: the parser, not the user

The single most important fact to establish before reading this rule's source
is *why* a plain SQL `IN` list would ever arrive at the optimizer already
wrapped in a call to `contains()` in the first place — a user writing `x IN
(1, 2, 3)` gets an ordinary, already-`COMPARE_IN`-shaped expression straight
out of the binder, with no rewrite needed. The answer is specific to DuckDB's
**bracket-list** `IN` syntax, `x IN [1, 2, 3]`, which the PEG parser's
transformer deliberately routes through `contains()` instead of building
`COMPARE_IN` children directly:

```cpp
// src/parser/peg/transformer/transform_expression.cpp:1181
unique_ptr<ParsedExpression>
PEGTransformerFactory::TransformInContainsExpression(PEGTransformer &transformer,
                                                     unique_ptr<ParsedExpression> other_operator_expression) {
	vector<unique_ptr<ParsedExpression>> children;
	children.push_back(std::move(other_operator_expression));
	return make_uniq<FunctionExpression>("contains", std::move(children));
}
```

and, once the left-hand side is available:

```cpp
// src/parser/peg/transformer/transform_expression.cpp:1077
} else if (between_in_like_expr->GetExpressionClass() == ExpressionClass::FUNCTION) {
	auto func_expr = unique_ptr_cast<ParsedExpression, FunctionExpression>(std::move(between_in_like_expr));
	if (func_expr->FunctionName() == "contains") {
		func_expr->GetArgumentsMutable().push_back(std::move(expr));
	}
```

So `x IN [1, 2, 3]` parses, before the binder ever sees it, directly into
`contains([1, 2, 3], x)` — the exact same overloaded function name `"contains"`
that a user could also call explicitly on a `LIST` (as `test/optimizer/
contains_to_in_clause.test`'s later cases do) or on a `VARCHAR` (the string
substring-search sense `LikeOptimizationRule`/`RegexOptimizationRule` both
produce). The reason the grammar takes this detour rather than just unpacking
`[1, 2, 3]` into three separate `COMPARE_IN` children the way `x IN (1, 2, 3)`
already does is that the bracket form's right-hand side is, syntactically, a
*single* expression — which might just as well be a `LIST`-typed *column*
(`x IN my_list_column`) rather than a literal — and a single expression can't
be exploded into per-element `COMPARE_IN` children at parse time without first
knowing whether it's even a constant. Routing both possibilities through one
runtime function call sidesteps that question entirely; `ContainsToInClauseRule`
is the piece that, later, specializes the constant case back down into the
`COMPARE_IN` shape the rest of the optimizer already knows how to reason
about.

### The rule itself

```cpp
// src/optimizer/rule/contains_to_in_clause.cpp:10
ContainsToInClauseRule::ContainsToInClauseRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	auto func = make_uniq<FunctionExpressionMatcher>();
	identifier_set_t functions = {"contains", "list_contains", "list_has", "array_contains", "array_has"};
	func->function = make_uniq<ManyFunctionMatcher>(functions);
	func->matchers.push_back(make_uniq<FoldableConstantMatcher>());
	func->matchers.push_back(make_uniq<ExpressionMatcher>());
	func->policy = SetMatcher::Policy::ORDERED;
	root = std::move(func);
}
```

```cpp
// src/optimizer/rule/contains_to_in_clause.cpp:20
unique_ptr<Expression> ContainsToInClauseRule::Apply(...) {
	// Null list: result is always NULL regardless of the probe value.
	if (list_val.IsNull()) {
		return make_uniq<BoundConstantExpression>(Value(LogicalType::BOOLEAN));
	}
	// For other types (i.e., string/map/struct) leave them alone.
	if (list_val.type().id() != LogicalTypeId::LIST) {
		return nullptr;
	}
	// ... collect non-NULL elements ...
	// No non-NULL elements: never contains any value.
	if (non_null_elements.empty()) {
		if (ListValue::GetChildren(list_val).empty()) {
			// An empty list never contains anything, not even NULL - but keep volatile/throwing probes.
			if (probe_arg->IsVolatile() || probe_arg->CanThrow()) {
				return nullptr;
			}
			changes_made = true;
			return make_uniq<BoundConstantExpression>(Value::BOOLEAN(false));
		}
		changes_made = true;
		return ExpressionRewriter::ConstantOrNull(GetContext(), probe_arg->Copy(), Value::BOOLEAN(false));
	}
	// Fully constant probe: let constant folding handle it.
	if (probe_arg->IsFoldable()) {
		return nullptr;
	}
	auto in_expr = make_uniq<BoundOperatorExpression>(ExpressionType::COMPARE_IN, LogicalType::BOOLEAN);
	in_expr->GetChildrenMutable().push_back(probe_arg->Copy());
	for (auto &v : non_null_elements) {
		in_expr->GetChildrenMutable().push_back(make_uniq<BoundConstantExpression>(std::move(v)));
	}
	return std::move(in_expr);
}
```

The `list_val.type().id() != LogicalTypeId::LIST` check right after evaluating
the first argument is what disambiguates the two totally different overloads
sharing the name `"contains"` — a string-`contains(s, needle)` call's first
argument evaluates to a `VARCHAR`, not a `LIST`, so it fails this check
immediately and the rule quietly returns `nullptr`, leaving the string version
completely untouched for `LikeOptimizationRule`/`RegexOptimizationRule`'s own
"contains" output (from the sections above) to keep working on unaffected.
`NULL`-handling gets three distinct outcomes worth naming precisely: a `NULL`
list itself (`contains(NULL::INTEGER[], x)`) makes the whole expression
unconditionally `NULL`; a list that's non-`NULL` but contains *no* non-`NULL`
elements after stripping (`[NULL]`) makes it `FALSE` when the probe
is non-`NULL` and `NULL` when the probe is `NULL` (`ConstantOrNull` again);
a truly empty list (`[]`) makes it `FALSE` even for a `NULL` probe; and
a genuinely non-empty list of real values becomes the explicit `COMPARE_IN`
this whole rule exists to produce.

The empty-list case differs from `[NULL]` on purpose. `contains([], NULL)`
gives `FALSE`: an empty list has nothing in it, so it does not "maybe"
contain `NULL`. A list holding only `NULL`s gives `NULL` for a `NULL` probe.
The rule keeps the call when the probe is volatile or can throw, so that a
`random()` call or an error in the probe is not silently removed. The test
`test/sql/function/list/list_contains.test` shows the difference:

```sql
-- test/sql/function/list/list_contains.test
# a constant list with a non-constant needle: an empty list never contains NULL, a list of only NULLs gives NULL
query TTT
SELECT x, list_contains([]::INT[], x), list_contains([NULL]::INT[], x) FROM (SELECT CASE WHEN i = 0 THEN NULL ELSE i END AS x FROM range(2) t(i)) ORDER BY x NULLS FIRST
----
NULL	false	NULL
1	false	false
```

### Walking `test/optimizer/contains_to_in_clause.test`

**The rewrite fires on the bracket-`IN` syntax exactly as the parser trace
above predicts — and what happens *next* is the other pass's decision, not
this rule's:**

```sql
-- Dense range, should use range filter not a join
EXPLAIN FROM the_table WHERE the_id in [1,2,3,...,1000];
----
logical_opt	<!REGEX>:.*COMPARISON_JOIN.*

-- Small list. Should NOT use a hash join, stays as a scan filter:
EXPLAIN FROM the_table WHERE the_id in [1,2,3]
----
logical_opt	<!REGEX>:.*COMPARISON_JOIN.*

-- Non-contiguous list. Should still use a join
EXPLAIN FROM the_table WHERE the_id in [1,42,999,12345,99999,100001]
----
logical_opt	<REGEX>:.*COMPARISON_JOIN.*
```

This is worth reading carefully, because the naive intuition ("bigger list,
more likely to become a join") is exactly backwards here: the **1000-element**
dense, contiguous list of `1..1000` does *not* become a join, while the
**6-element** sparse list `{1, 42, 999, 12345, 99999, 100001}` does. Both
queries reach the plan-level `IN_CLAUSE` pass carrying the identical
`COMPARE_IN` shape `ContainsToInClauseRule` builds — this rule's job stops
there. Which one of those two identically-shaped `IN` expressions is worth
turning into a hash join is entirely `IN_CLAUSE`'s own, separate cost
reasoning (covered in full in this chapter's sibling file on the regex-range
and `IN`-clause passes): a dense, contiguous run of integer values can be
represented as one cheap `BETWEEN`-style range the storage layer's zonemaps
prune directly against, so a join buys nothing; a sparse, scattered set of
values has no such compact range representation, so building a small hash
table and probing it really is cheaper than either a long chain of `OR`-ed
equality checks or an overly wide, mostly-empty range scan. This is exactly
the "one rule's output shape exists specifically to feed a later pass" pattern
promised at the top of this file, now traced through two genuinely separate
source files with the test evidence to back up both halves independently.

**The "Not yet implemented" comment is in the test file itself, not inferred:**

```sql
# Not yet implemented
EXPLAIN FROM the_table WHERE the_id NOT IN [1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20];
----
logical_opt	<!REGEX>:.*COMPARISON_JOIN.*
```

`NOT IN` never becomes a join, regardless of list size or density, and the
test's own comment states plainly that this is a known, currently-unaddressed
gap rather than an intentional design choice — worth preserving verbatim
rather than paraphrasing, since it's the test suite documenting its own
project's incompleteness.

**Degenerate lists collapse the entire filter to `EMPTY_RESULT`, tying
directly back to the `ConstantOrNull(..., FALSE)` branch above:**

```sql
EXPLAIN SELECT * FROM t WHERE list_contains([], a)
----
physical_plan	<REGEX>:.*EMPTY_RESULT.*

-- NULL-only list: list_contains([NULL], a) should return false, not NULL
SELECT list_contains([NULL], a) FROM (VALUES (1)) t(a);
----
false

EXPLAIN SELECT * FROM t WHERE list_contains([NULL], a)
----
physical_plan	<REGEX>:.*EMPTY_RESULT.*
```

The mechanism worth making explicit here: `ConstantOrNull(GetContext(),
probe_arg->Copy(), Value::BOOLEAN(false))` can only ever evaluate to `FALSE`
or `NULL` — it can *never* be `TRUE`, for any value the probe column takes at
any row. Once that fact is visible to `STATISTICS_PROPAGATION` (§7.4 #34, run
well after this expression-level rewrite), the entire `WHERE` filter is
provably unsatisfiable for every row regardless of the actual data in `t`, and
the whole subtree collapses to `LogicalEmptyResult` — the physical plan the
test's `EXPLAIN` is checking for. `list_contains([1, NULL, 2], a)`, by
contrast, strips the lone `NULL` element and still has two real values left
(`1`, `2`), so it becomes a genuine, satisfiable `a IN (1, 2)` and the test
confirms `EMPTY_RESULT` correctly does *not* appear:

```sql
EXPLAIN SELECT * FROM t WHERE list_contains([1, NULL, 2], a)
----
physical_plan	<!REGEX>:.*EMPTY_RESULT.*
```

## `ListComprehensionRewriteRule` — collapsing a three-stage lambda chain to two

**Purpose, in one sentence:** recognize the specific three-function lambda
chain (`list_apply` building a `{filter, result}` struct per element,
`list_filter` on the `.filter` field, `list_apply` on the `.result` field) that
DuckDB's Python-style list comprehension syntax always lowers to when it has an
`IF` clause, and collapse it into a plain two-function `list_filter` +
`list_apply` chain when doing so can't change the answer.

### What the surface syntax actually parses to, and why it needs a rewrite at all

DuckDB's list comprehension syntax, `[expr FOR var[, idx] IN list IF cond]`,
does not lower directly to a single function call — the parser itself takes a
different path depending on whether an `IF` clause is present at all:

```cpp
// src/parser/peg/transformer/transform_expression.cpp:2931
unique_ptr<ParsedExpression> PEGTransformerFactory::TransformListComprehensionExpression(...) {
	// ...
	if (!list_comprehension_filter || !*list_comprehension_filter) {
		auto lambda_expression = make_uniq<LambdaExpression>(lambda_columns, std::move(result_expr));
		// ...
		return make_uniq<FunctionExpression>("list_apply", std::move(apply_children));
	}

	// STAGE 1: list_apply(in_expr, x -> struct_pack(filter := ..., result := ...))
	auto struct_pack = make_uniq<FunctionExpression>("struct_pack", std::move(struct_children));
	auto stage1_apply = make_uniq<FunctionExpression>("list_apply", std::move(stage1_apply_args));

	// STAGE 2: list_filter(stage1, elem -> struct_extract(elem, 'filter'))
	auto stage2_filter = make_uniq<FunctionExpression>("list_filter", std::move(stage2_filter_args));

	// STAGE 3: list_apply(stage2, elem -> struct_extract(elem, 'result'))
	return make_uniq<FunctionExpression>("list_apply", std::move(stage3_apply_args));
}
```

A filter-free comprehension, `[len(x) for x in words]`, is already optimal the
moment it leaves the parser — one `list_apply(words, lambda x: len(x))`, and
`ListComprehensionRewriteRule` has nothing to do with it at all. A *filtered*
comprehension, `[x * x for x in list if x % 2 = 0]`, has no such direct
translation available at parse time (the grammar has no primitive that fuses
"filter" and "map" in one step), so it's built the only way the parser can
build it generically: compute *both* the filter condition and the result
expression together, per element, packed into a `{filter: bool, result: T}`
struct (stage 1); keep only the elements whose `.filter` field is true (stage
2); and unwrap the surviving elements' `.result` field (stage 3). This is
correct but wasteful — it materializes an intermediate list of structs the
size of the *original*, unfiltered input, evaluates the result expression for
every element including ones that will immediately be discarded, and pays for
a `struct_pack`/`struct_extract` round-trip that a hand-written `list_filter`+
`list_apply` chain would never need. Closing that gap is exactly what this
rule exists to do.

### The matcher is unusually broad, and the narrowing happens entirely inside `Apply`

```cpp
// src/optimizer/rule/list_comprehension_rewrite.cpp:312
ListComprehensionRewriteRule::ListComprehensionRewriteRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	root = make_uniq<ExpressionMatcher>(ExpressionClass::BOUND_FUNCTION);
}
```

Worth contrasting explicitly with every other rule in this file: `LikeOptimizationRule`
matches six specific function-name strings, `RegexOptimizationRule` matches
exactly `"regexp_matches"`, `ContainsToInClauseRule` matches five specific
names — each rule's matcher does real, cheap pre-filtering before `Apply` ever
runs. `ListComprehensionRewriteRule`'s matcher, by contrast, is `ExpressionMatcher
(ExpressionClass::BOUND_FUNCTION)` — it fires on *every single bound function
call in the entire query tree* and pushes the entire burden of recognizing the
specific three-stage shape into `MatchListComprehensionRewrite`, which has to
walk down through a `list_transform`-or-`list_apply` call, into its child
`list_filter`-or-`list_filter`-alias call, into *that* call's child
`list_transform`-or-`list_apply` call, and confirm a `struct_pack` with fields
named exactly `filter` and `result` sits at the very bottom — five nested
shape checks before `Apply` can even consider producing a rewrite. This is a
reasonable design choice given that the three-stage shape genuinely can't be
recognized from a function name alone (`list_apply` also appears in ordinary,
unrelated list-processing queries with no comprehension origin at all — see
the "non-comprehension nested list lambda" test case below), but it does mean
this is the one rule in the batch whose per-node matching cost is closer to
"linear scan of the whole subtree shape" than "single function-name lookup."

`IsTargetListFunction` is what lets the matcher recognize both spellings the
parser and a user's own SQL might use interchangeably:

```cpp
// src/optimizer/rule/list_comprehension_rewrite.cpp:22
bool IsTargetListFunction(ClientContext &context, const BoundFunctionExpression &expr, const string &target_name) {
	if (expr.Function().GetName() == target_name) {
		return true;
	}
	// Compare function name with catalog to recognize aliases
	auto entry = catalog.GetEntry<ScalarFunctionCatalogEntry>(...);
	if (!entry->alias_of.empty()) {
		return entry->alias_of == target_name;
	}
	return entry->name == target_name;
}
```

`list_apply` (what the parser actually emits) is a catalog-registered alias of
`list_transform` (what `IsTargetListFunction(..., "list_transform")` checks
for by name) — this is exactly the alias-resolution mechanism
`test/sql/function/list/lambdas/list_comprehension.test`'s own `foreach t
list_transform apply array_apply array_transform` / `foreach f list_filter
array_filter filter` loop is built to exercise directly, running the identical
three-stage query text under every real spelling of both functions and
confirming the rule still recognizes and rewrites all of them.

### Walking `test/sql/optimizer/expression/list_comprehension_rewrite.test`

The critical guard in the rule's matching logic is this one:

```cpp
// src/optimizer/rule/list_comprehension_rewrite.cpp:232
if (inner_bind.has_index && UsesIndexParameter(*result_expr)) {
	return nullptr;
}
```

— if the comprehension declared two lambda variables (`FOR x, i IN l`, so `i`
is the numeric index), and the **result** expression (not the filter
condition) actually references that index, the rewrite is declined entirely.
The test file's four core queries are built precisely to pin down that this
guard depends on the *result* expression and nothing else:

```sql
CREATE TABLE t(l INTEGER[]);
INSERT INTO t VALUES ([10, 9, 8, 7, 6]);

-- result uses the index (i) -> NOT rewritten: three-stage chain survives
EXPLAIN SELECT [i + 5 FOR x, i IN l IF i > 2] FROM t;
----
logical_opt	<REGEX>:.*list_apply.*list_filter.*list_apply.*

EXPLAIN SELECT [i + 5 FOR x, i IN l IF x > 8] FROM t;
----
logical_opt	<REGEX>:.*list_apply.*list_filter.*list_apply.*

-- result uses only the value (x) -> rewritten: collapses to two stages
EXPLAIN SELECT [x + 5 FOR x, i IN l IF i > 2] FROM t;
----
logical_opt	<!REGEX>:.*list_apply.*list_filter.*list_apply.*

EXPLAIN SELECT [x + 5 FOR x, i IN l IF x > 8] FROM t;
----
logical_opt	<!REGEX>:.*list_apply.*list_filter.*list_apply.*
```

Reading the regex's own logic makes the two outcomes unambiguous: the
un-rewritten, three-stage plan's printed text necessarily contains the
substring `list_apply` (the outer stage-3 call), then later `list_filter`
(stage 2, nested inside it), then later still `list_apply` again (stage 1,
nested inside *that*) — literally, `apply...filter...apply` in that left-to-
right order, which is what `.*list_apply.*list_filter.*list_apply.*` is
checking for. The rewritten, two-stage plan built by `BuildListComprehensionRewrite`
contains exactly one `list_filter` and one `list_apply` — no leading `list_apply`
sits above the `list_filter`, so the three-part regex cannot match, which is
exactly what the `<!REGEX>` (negated) assertions on the `x+5` queries confirm.
The filter condition (`i > 2` versus `x > 8`) never affects which of these two
shapes the plan ends up with — only which expression is used as the *result*
does — and the test's own pairing (both filter conditions tried against both
result expressions) is deliberately constructed to make that isolation
unambiguous.

The *reason* the guard exists at all, worked out precisely from the two plan
shapes: in the un-rewritten, three-stage form, the index `i` is computed
inside **stage 1**, against the *original*, unfiltered list — so `i` always
reflects each surviving element's true position in the input DuckDB started
with, independent of how many elements the later filter stage discards. If the
rewrite collapsed this into `list_apply(list_filter(l, cond_lambda),
result_lambda)` unconditionally, `result_lambda` would run **after** filtering
— against the *filtered* list — so any reference to the index inside it would
number surviving elements `1, 2, 3, ...` within the filtered list instead,
which is a different, wrong sequence of numbers the moment the filter actually
discards anything. The guard exists purely to protect this one specific piece
of information (what the index number *meant*) across the collapse; it costs
nothing when the result expression doesn't reference the index at all, which
is presumably why the guard is scoped as narrowly as it is rather than
disabling the whole rewrite whenever an index parameter was merely declared.

The correctness of all four shapes (not just their plan shape, but their
actual output) is checked immediately afterward:

```sql
SELECT [i + 5 FOR x, i IN l IF i > 2] FROM t;   ----  [8, 9, 10]
SELECT [i + 5 FOR x, i IN l IF x > 8] FROM t;   ----  [6, 7]
SELECT [x + 5 FOR x, i IN l IF i > 2] FROM t;   ----  [13, 12, 11]
SELECT [x + 5 FOR x, i IN l IF x > 8] FROM t;   ----  [15, 14]
```

**A guard against relocating volatile computation, matching a theme that
recurs across this whole chapter (`WINDOW_SELF_JOIN` and `COMMON_SUBPLAN`, both
in §7.4, carry the identical caution for their own rewrites):**

```cpp
// src/optimizer/rule/list_comprehension_rewrite.cpp:235
if (result_expr->IsVolatile()) {
	return nullptr;
}
```

```sql
EXPLAIN SELECT [x * random() FOR x, i IN l IF x > 2] FROM t;
----
logical_opt	<REGEX>:.*list_apply.*list_filter.*list_apply.*     -- stays three-stage
```

Collapsing the three stages into two doesn't just relocate *when* the result
expression runs relative to filtering — for a query like this it would also
change how the rewrite's own bookkeeping treats it, since `random()` is
explicitly disallowed from being duplicated or reordered anywhere in this
optimizer's architecture (a value the query correctness depends on being
computed exactly once per logical evaluation site); rather than reason through
whether this specific rewrite is actually safe for a volatile result
expression, the rule declines unconditionally the moment `IsVolatile()` is
true, which is the same conservative posture this chapter's overview file
documents for every other rewrite that risks relocating a side-effecting call.

**Captured columns from an enclosing scope are threaded through unchanged,
confirmed by an explicit test:**

```sql
CREATE TABLE capture_t(l INTEGER[], c INTEGER);
INSERT INTO capture_t VALUES ([10, 9, 8, 7, 6], 8);

SELECT [x + c FOR x IN l IF x > c] FROM capture_t;
----
[18, 17]
```

`c` (an outer-scope column reference, not one of the comprehension's own
lambda parameters) shows up as an *extra* trailing child on both the
`list_filter` and `list_transform`/`list_apply` calls the lambda functions'
own binding machinery adds to support captures; `CopyCapturedChildren` exists
specifically to copy these extra children across from the original stage-1
call into the rebuilt, collapsed filter call, so a capture reference like `c`
above keeps resolving correctly on either side of the rewrite.

**A structurally similar, but non-comprehension-originated, nested lambda call
is correctly left alone — because it doesn't match the exact `struct_pack`
with `filter`/`result` field-name shape the matcher requires:**

```sql
SELECT list_transform(
	list_filter(
		list_transform(l, lambda x: struct_pack(filter := x > 8, result := x + 5)),
		lambda s: NOT s.filter
	),
	lambda s: s.result
) FROM t;
----
[13, 12, 11]
```

The middle stage here filters on `NOT s.filter`, not the bare `s.filter`
`MatchesStructFieldProjection` requires — a hand-written query that happens to
resemble the comprehension shape closely, but negates the condition, is
correctly recognized as *not* the pattern this rule targets and is left to
execute exactly as written (still producing the right answer, just without
the collapse).

The matcher does not look through casts in general. If it peeled every cast
off the lambda bodies before checking for `s.filter` and `s.result`, then
`lambda s: s.result::BOOLEAN::INTEGER` would look the same as
`lambda s: s.result`, and the rewrite would throw the casts away. That
changes answers: `2::BOOLEAN::INTEGER` is `1`, not `2`. So
`MatchesStructFieldProjection` (`list_comprehension_rewrite.cpp:90`) accepts
at most one cast, and only in the filter stage: a single cast to `BOOLEAN`
that is not a `TRY_CAST`. The rewritten filter recreates that one cast. Any
other cast stops the rewrite. The test
`test/sql/optimizer/expression/list_comprehension_casts.test` runs each
query with the optimizer on and off and expects the same answer:

```sql
-- test/sql/optimizer/expression/list_comprehension_casts.test
# A lossy round trip must not be replaced with the original value.
query I
SELECT list_transform(
	list_filter(list_transform(l, lambda x: struct_pack(filter := true, result := x)), lambda s: s.filter),
	lambda s: s.result::BOOLEAN::INTEGER
) FROM (VALUES ([2])) t(l);
----
[1]
```

## How this batch of rules fits together

Pulling back to the level the overview file's §7.5 sits at: three genuinely
different pieces of surface syntax — `LIKE`/`GLOB`, `regexp_matches`, and
`left`/`substring`/`instr`/`position` — all converge, through three separate
rules in this file, on the same small vocabulary of storage-friendly
primitives (`prefix`, `suffix`, `contains`, plain `=`), which a fourth,
entirely separate pass in a different source file (`filter_combiner.cpp`'s
`TryPushdownPrefixFilter`, part of `FILTER_PUSHDOWN`) then knows how to turn
into genuine range predicates a row group's zonemap can prune against.
`ContainsToInClauseRule` plays the identical "produce the shape a later pass
needs" role one level up the pipeline, taking the parser's own `contains()`-
based lowering of bracket-list `IN` syntax and restoring it to the
`COMPARE_IN` shape the plan-level `IN_CLAUSE` pass (§7.4 #9) already knows how
to cost and, sometimes, turn into a join. `EmptyNeedleRemovalRule`/
`NoopReplaceRemovalRule` sit slightly apart from that theme — their payoff is
entirely local, replacing a string-scanning function call with a null-check or
a dropped no-op, with nothing downstream depending on their specific output
shape. `ListComprehensionRewriteRule` is the outlier of the batch: it isn't
feeding a storage-layer pushdown or a join decision at all, it's purely about
not paying for an intermediate struct-per-element materialization the parser's
own generic lowering strategy couldn't avoid producing — a reminder that not
every rule in this 37-rule list is chasing the same kind of win, even within
one thematically-grouped batch of six.
