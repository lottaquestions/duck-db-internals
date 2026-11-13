# Part 8 — Two Cheap, Post-Pushdown Predicate Rewrites: `REGEX_RANGE` and `IN_CLAUSE`

## 8.1 Where these sit, and why this file pairs them

`00-overview.md`'s §7.4 pipeline listing puts these two passes back to back, at
positions #8 and #9, immediately after `FILTER_PUSHDOWN` (#6) and
`CTE_FILTER_PUSHER` (#7):

```
6. FILTER_PUSHDOWN         — src/optimizer/filter_pushdown.cpp
7. CTE_FILTER_PUSHER       — src/optimizer/cte_filter_pusher.cpp
8. REGEX_RANGE             — src/optimizer/regex_range_filter.cpp
9. IN_CLAUSE               — src/optimizer/in_clause_rewriter.cpp
```

They belong together for a reason beyond alphabetical or numerical
convenience: both are small, single-purpose classes (`RegexRangeFilter`,
`InClauseRewriter`) that walk the *already-pushed-down* tree looking for one
narrow expression shape — a `regexp_full_match` call in one case, a
constant-list `IN`/`NOT IN` in the other — and rewrite just that shape,
leaving everything else in the tree untouched. Neither reorders operators,
neither reasons about cost, and neither is a rule inside the pass-#1
expression rewriter's 37-rule list (§7.5) even though both have same-named or
similarly-named cousins living there. That last point turns out to be the
single most important thing to get right about each of them, and this file
spends real space nailing down the exact boundary in both cases, because — as
the task that produced this chapter suspected — the two are genuinely easy to
conflate with their expression-level cousins, and the test suite's own file
names do not help: `test/sql/function/string/regex_filter_pushdown.test`
sounds like it must be about `REGEX_RANGE`, and it isn't.

One more fact worth stating up front because it shapes both discussions
below: **both passes run after `FILTER_PUSHDOWN`, not before it.** That
ordering fact turns out to matter concretely for what each pass's rewrite can
still accomplish, in ways that are easy to miss if you only read the two
source files in isolation from the fixed pipeline they sit in.

## 8.2 `REGEX_RANGE` — deriving a range pre-filter from a full-match regex

**Purpose in one sentence:** for a `regexp_full_match(col, pattern)` filter
whose constant pattern implies every possible match must fall lexicographically
between two byte strings, `REGEX_RANGE` inserts a second, cheap comparison
filter — `col >= min AND col <= max` — directly below the original regex
filter, so most rows can be rejected by two vectorized byte comparisons before
ever reaching RE2.

### 8.2.1 Source walkthrough

The whole pass is 61 lines. Reproduced in full because every line matters to
getting the mechanics right:

```cpp
// src/optimizer/regex_range_filter.cpp:18
unique_ptr<LogicalOperator> RegexRangeFilter::Rewrite(unique_ptr<LogicalOperator> op) {
	for (idx_t child_idx = 0; child_idx < op->children.size(); child_idx++) {
		op->children[child_idx] = Rewrite(std::move(op->children[child_idx]));
	}

	if (op->type != LogicalOperatorType::LOGICAL_FILTER) {
		return op;
	}

	auto new_filter = make_uniq<LogicalFilter>();

	for (auto &expr : op->expressions) {
		if (expr->GetExpressionType() == ExpressionType::BOUND_FUNCTION) {
			auto &func = expr->Cast<BoundFunctionExpression>();
			if (func.Function().GetName() != "regexp_full_match" || func.GetChildren().size() != 2) {
				continue;
			}
			auto &info = func.BindInfo()->Cast<RegexpMatchesBindData>();
			if (!info.range_success) {
				continue;
			}
			auto filter_left = BoundComparisonExpression::Create(
			    ExpressionType::COMPARE_GREATERTHANOREQUALTO, func.GetChildren()[0]->Copy(),
			    make_uniq<BoundConstantExpression>(Value::BLOB_RAW(info.range_min)));
			auto filter_right = BoundComparisonExpression::Create(
			    ExpressionType::COMPARE_LESSTHANOREQUALTO, func.GetChildren()[0]->Copy(),
			    make_uniq<BoundConstantExpression>(Value::BLOB_RAW(info.range_max)));
			auto filter_expr = make_uniq<BoundConjunctionExpression>(ExpressionType::CONJUNCTION_AND,
			                                                         std::move(filter_left), std::move(filter_right));

			new_filter->expressions.push_back(std::move(filter_expr));
		}
	}

	if (!new_filter->expressions.empty()) {
		new_filter->children = std::move(op->children);
		op->children.clear();
		op->children.push_back(std::move(new_filter));
	}

	return op;
}
```

Point by point, several things here are worth being precise about, because
the task's own framing of this pass needed correcting on more than one of
them once checked against the source:

**It only ever looks at `regexp_full_match`, never `regexp_matches`.** Line
32's check (`func.Function().GetName() != "regexp_full_match"`) is not
incidental — it is the single most important gate in this file, and it is
*not* a "for now" restriction the code apologizes for; it is a correctness
requirement. RE2's `PossibleMatchRange` (the routine that actually computes
`range_min`/`range_max`, discussed below) documents its own guarantee this
way:

```
// Assuming PossibleMatchRange(&min, &max, N) returns successfully, any
// string s that is an anchored match for this regexp satisfies
//   min <= s && s <= max.
```
(`third_party/re2/re2/re2.h:513`, as bundled in this checkout)

"Anchored match" here means anchored at the *start* of the string — exactly
the guarantee `regexp_full_match` provides (RE2's `RegexFullMatch`, which
DuckDB's `regexp_full_match` calls, requires the whole string to match, so it
is implicitly anchored at both ends). `regexp_matches` — DuckDB's
*unanchored*, "appears anywhere in the string" variant — gives no such
guarantee: a string can satisfy `regexp_matches(x, pattern)` by containing a
match starting well past the string's first byte, in which case `x` itself
can fall completely outside `[min, max]` while still being a match. Using the
derived range as a *pre-filter* (AND-ed alongside the original predicate,
never in place of it — see below) for `regexp_matches` would therefore be an
outright correctness bug: it would discard some rows that should have
survived the `WHERE` clause. The source's own git history confirms this was
learned the hard way rather than designed in from day one — a commit in
DuckDB's history is titled, verbatim, *"The regex range filter can
only be applied for full match regex matches; it is incorrect for partial
regex matches"* — and the current `regexp_full_match`-only gate is that
lesson, permanently encoded as a string comparison at line 32.

One consequence worth spelling out because it is easy to miss: **both
`regexp_full_match` and `regexp_matches` share the exact same bind data type**
(`RegexpMatchesBindData`, `src/include/duckdb/function/scalar/regexp.hpp:84`,
bound by the same `RegexpMatchesBind` for both, per
`src/function/scalar/string/regexp.cpp:453-475`), and that bind data computes
`range_min`/`range_max`/`range_success` identically for both functions:

```cpp
// src/function/scalar/string/regexp.cpp:76
RegexpMatchesBindData::RegexpMatchesBindData(duckdb_re2::RE2::Options options, string constant_string_p,
                                             bool constant_pattern)
    : RegexpBaseBindData(options, std::move(constant_string_p), constant_pattern) {
	if (constant_pattern) {
		// an invalid pattern is reported when the function state is initialized
		auto pattern = make_uniq<RE2>(constant_string, options);
		range_success = pattern->ok() && pattern->PossibleMatchRange(&range_min, &range_max, 1000);
	} else {
		range_success = false;
	}
}
```

> **Note:** the bind data does not throw for an invalid constant pattern. It
> sets `range_success = false`, and the error is raised later, when the
> function's state is initialized at execution. That lets `TRY(regexp_matches(x, '('))` return `NULL` instead of failing while the
> query is planned (`test/sql/error/test_try_constant_regex.test`). For
> `REGEX_RANGE` the effect is simple: an invalid pattern just has no range,
> so the pass skips it.

So a `regexp_matches(x, '^abc')` call *does* have a perfectly valid
`range_min`/`range_max` sitting in its bind data at optimization time — RE2
computed it regardless of which function called it — and `REGEX_RANGE`
simply never looks at it, because `func.Function().GetName()` for that call
is `"regexp_matches"`, not `"regexp_full_match"`, and the very first `if` in
the loop `continue`s past it. This is a real, source-confirmed asymmetry, not
speculation: an explicitly `^`-anchored `regexp_matches` pattern, where the
derived range genuinely *would* be safe to use as a pre-filter (since `^`
already forces the match to start at position 0, making the "anchored match"
guarantee hold even under `regexp_matches`'s otherwise-unanchored semantics),
still gets none of this optimization, purely because the gate is a function
name check, not a pattern-anchoring check. §8.2.5 returns to this as a named
limitation.

**`maxlen=1000`** is the third argument to `PossibleMatchRange` above, and it
caps how many bytes RE2 will walk its compiled automaton before giving up —
per its own header comment, "Note that `PossibleMatchRange()` will only
consider the first copy of an infinitely repeated element (i.e., any regexp
element followed by a `*` or `+` operator)." For a pattern like `abc.*`, RE2
does not need to walk 1000 bytes deep to determine the range is exactly
`["abc", "abd")`-ish (see the exact min/max semantics below) — the `.*` tail
does not narrow the range further once the fixed prefix is exhausted, and
`range_success` comes back `true` either way. `range_success` only comes back
`false` for patterns RE2's automaton walk cannot bound at all within the
byte budget, or (per the constructor above) whenever the pattern itself is
*not* a compile-time constant (`constant_pattern == false` skips the call
entirely and hard-sets `range_success = false`).

**It is an addition, not a replacement — the task's own guess was right
about this part, and the source confirms it precisely.** Walking the code
again: `new_filter` starts empty and only ever *accumulates* derived range
conjuncts; nothing in the loop ever removes or rewrites `op->expressions`
itself. Then:

```cpp
if (!new_filter->expressions.empty()) {
	new_filter->children = std::move(op->children);
	op->children.clear();
	op->children.push_back(std::move(new_filter));
}
```

`new_filter` is spliced in as a **new child of `op`**, inheriting `op`'s old
children, and `op` — the *original* `LogicalFilter`, still holding the
original, untouched `regexp_full_match` expression in its own `expressions`
list — becomes `new_filter`'s new parent. In diagram form, for
`WHERE regexp_full_match(name, 'Smi.*')` sitting directly above a scan:

```
before:
┌───────────────────────────────────────┐
│ LogicalFilter                          │
│  regexp_full_match(name, 'Smi.*')      │
└──────────────────┬──────────────────────┘
                    │
            ┌───────▼────────┐
            │ LogicalGet      │
            └────────────────┘

after REGEX_RANGE:
┌───────────────────────────────────────┐
│ LogicalFilter (unchanged, original op) │
│  regexp_full_match(name, 'Smi.*')      │
└──────────────────┬──────────────────────┘
                    │
┌───────────────────▼───────────────────┐
│ LogicalFilter (new, derived)            │
│  name >= 'Smi' AND name <= 'Smj'        │
│  (BLOB_RAW comparison, not zonemap-linked│
│   in this same pipeline run — see 8.2.2)│
└──────────────────┬──────────────────────┘
                    │
            ┌───────▼────────┐
            │ LogicalGet      │
            └────────────────┘
```

Both filters survive to execution. The reason the original regex predicate
*must* survive is exactly the task's own framing: the derived range is a
*necessary* but not *sufficient* condition for a full match. `name >= 'Smi'
AND name <= 'Smj'` is true for `"Smith"`, `"Smile"`, and also for `"Smix"` —
none of which necessarily satisfies the specific pattern `Smi.*` in full (in
this particular example every string starting with `Smi` does happen to
satisfy `Smi.*`, but that is a property of this specific pattern, not of the
mechanism in general — a pattern like `Smi[aeiou]th` has the same derived
range yet rejects `"Smixth"`). Replacing the regex check with the range check
alone would silently change the query's answer; the pass is careful never to
do that.

### 8.2.2 What "alongside, not replacing" actually gets you — and, precisely, what it doesn't

Here is where checking this pass against its exact position in the fixed
pipeline (§7.2, §7.4) corrects a plausible but wrong assumption that a
first read of the source, in isolation, would encourage.

The natural assumption — the one embedded in the task's own framing — is
that the new `col >= min AND col <= max` filter is "usable by zonemaps/an ART
index range scan," the same way `LikeOptimizationRule`'s `LIKE 'foo%'` →
prefix-range rewrite is described doing in §7.5. That would be true **if**
this newly-created comparison filter ended up inside `LogicalGet.table_filters`
— the field the storage layer's row-group zonemaps actually consult to decide
whether an entire row group can be skipped without reading it. But the
*only* code in this whole codebase that ever populates
`LogicalGet.table_filters` is `FilterCombiner::GenerateTableScanFilters`,
called from inside `FilterPushdown::PushdownGet`
(`src/optimizer/pushdown/pushdown_get.cpp:165`) — and that is `FILTER_PUSHDOWN`,
pass #6. `REGEX_RANGE` is pass #8, called at `optimizer.cpp:305`, strictly
**after** `FILTER_PUSHDOWN` at `optimizer.cpp:286` has already run and
finished populating `table_filters` for every `LogicalGet` in the tree:

```cpp
// src/optimizer/optimizer.cpp:286 / :305 (exact call sites, in this order)
RunOptimizer(OptimizerType::FILTER_PUSHDOWN, [&]() { ... });
// ... CTE_FILTER_PUSHER runs in between ...
RunOptimizer(OptimizerType::REGEX_RANGE, [&]() {
	RegexRangeFilter regex_opt;
	plan = regex_opt.Rewrite(std::move(plan));
});
```

Per §7.2's own stated architecture — a fixed, one-shot sequence with no pass
ever getting a second turn — the brand-new `LogicalFilter` that `REGEX_RANGE`
inserts at pass #8 is created *after* the one pass capable of converting a
residual `LogicalFilter`'s comparison into a `TableFilter` has already come
and gone. Nothing downstream in the ~45-pass sequence re-runs
`FilterCombiner::GenerateTableScanFilters` on it. `plan_get.cpp`, which is
what actually builds the physical `PhysicalTableScan` and hands it its
`TableFilterSet`, only ever reads whatever is *already* sitting in
`op.table_filters` (`src/execution/physical_plan/plan_get.cpp:71,103` —
`if (op.table_filters.HasFilters() ...) table_filters = MoveTableFilters(op.table_filters);`)
— it never inspects the operators sitting above the `LogicalGet` in the tree.

The practical upshot: in this specific fixed pipeline, `REGEX_RANGE`'s
derived range filter becomes an ordinary `PhysicalFilter` operator sitting
between the scan and the (also-surviving) regex-matching `PhysicalFilter` —
a cheap, vectorized `>=`/`<=` byte-comparison pass that every row must still
pass through in memory, not a row-group-skipping zonemap predicate the
storage layer can use to avoid reading data at all. It is still a genuine,
real speedup — two vectorized comparisons against a constant, executed on
every batch, are enormously cheaper than invoking RE2's DFA per row, so rows
that fail the range check never pay for a regex match attempt — but it is a
narrower win than "usable by zonemaps" implies, and the source's own fixed
ordering is the reason why. This is exactly the kind of thing this book's
established practice (see §7.6.2's correction to Chapter 1's shallow preview
of `STATISTICS_PROPAGATION`) exists to catch: the plausible-sounding claim and
the source-verified claim diverge, and the divergence traces to one very
specific, citable fact (`optimizer.cpp:286` running before `optimizer.cpp:305`)
rather than to any ambiguity in what the code does.

*(A genuinely honest caveat: this book's evidence for this claim is a static
read of `optimizer.cpp`'s pass ordering plus `plan_get.cpp`'s exclusive
reliance on `table_filters`, not an `EXPLAIN ANALYZE` captured against a live
`v2.0-cyanoptera` build inside this exercise — no such build was produced for
this file specifically. §7.3's own advice — disable `filter_pushdown` or
`regex_range` individually via `disabled_optimizers` and diff the physical
plan — is the direct way to confirm this on a real checkout, and is
recommended over taking this chapter's word for it.)*

### 8.2.3 The boundary with the expression-level `RegexOptimizationRule`

`00-overview.md`'s §7.5 already covers `RegexOptimizationRule`
(`src/optimizer/rule/regex_optimizations.cpp`) — the pass-#1,
expression-level rule that recognizes a metachar-free or simply-structured
constant regex pattern and rewrites the *whole call* into a cheaper
equivalent (`contains`, `~~` / `LIKE`, `prefix`, `suffix`). It is worth
stating the boundary between that rule and `REGEX_RANGE` as precisely as the
source allows, because the two are the single easiest thing in this whole
area to conflate, and the boundary turns out to be a clean, almost
accidental-looking partition by function name:

```cpp
// src/optimizer/rule/regex_optimizations.cpp:19
RegexOptimizationRule::RegexOptimizationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	auto func = make_uniq<FunctionExpressionMatcher>();
	func->function = make_uniq<SpecificFunctionMatcher>("regexp_matches");
	// ...
```

`RegexOptimizationRule`'s matcher is hard-wired to `"regexp_matches"` and
`"regexp_matches"` alone — it never even looks at `regexp_full_match` calls.
`REGEX_RANGE`'s gate, symmetrically, only ever looks at `"regexp_full_match"`
(§8.2.1). The two passes therefore partition regex predicates by which of
DuckDB's two match functions was called, with essentially no overlap:

| Function called | Pass-#1 `RegexOptimizationRule` (expression-level) | Pass-#8 `REGEX_RANGE` (plan-level) |
|---|---|---|
| `regexp_matches(x, p)` (unanchored/partial match) | Can rewrite the *whole call* to `contains`/`~~`/`prefix`/`suffix` if `p` is metachar-free or a simple shape | Never touches it — the function-name gate excludes it (and doing so for an unanchored match would be a correctness bug, §8.2.1) |
| `regexp_full_match(x, p)` (anchored/full match) | Never touches it — the rule's matcher requires the name `"regexp_matches"` | Can add a derived range pre-filter if `p`'s `PossibleMatchRange` succeeds |

A practical consequence: a `regexp_full_match` predicate is *never*
simplified into `contains`/`LIKE`/`prefix`/`suffix` by anything in this
pipeline, no matter how simple its pattern is — the call always survives, at
minimum, as a genuine `regexp_full_match` invocation at execution time (RE2
still runs on every row that passes whatever pre-filter exists), with
`REGEX_RANGE` at best adding a cheap pre-filter beside it. `SIMILAR TO`
(§8.2.4 below explains why that syntax always compiles to
`regexp_full_match`) therefore never gets the "turn into `contains`"
treatment that an equivalent `x ~ 'pattern'` (which, under the *default*
`regex_match_operator_semantics` setting, compiles to `regexp_matches`,
`src/include/duckdb/parser/parser_options.hpp:36`) would.

### 8.2.4 Real test evidence — and an honest accounting of how thin it is for this specific pass

The task that produced this chapter asked for a direct check of
`test/optimizer/regex_optimizer_coverage.test` and
`test/optimizer/regex_to_like_optimizer.test` to determine which pass each
one actually exercises, rather than assuming from the filename. Having read
both in full, plus a third file in the same family
(`test/optimizer/regex_optimizer.test`) and one more that its name alone
makes tempting to misattribute
(`test/sql/function/string/regex_filter_pushdown.test`), the honest
conclusion is: **none of the four exercise `REGEX_RANGE`.**

- **`test/optimizer/regex_optimizer.test`** uses `PRAGMA explain_output =
  OPTIMIZED_ONLY` and a long run of `explain analyze select regexp_matches(s,
  pattern), s from test` queries, checking the `analyzed_plan` output against
  regexes like `<REGEX>:.*contains\(s, 'aa'\).*`,
  `<REGEX>:.*"~~"\(s, '%a_a%'\).*`, `<REGEX>:.*prefix\(s, 'a'\).*`,
  `<REGEX>:.*suffix\(s, 'a'\).*`. Every one of these is a
  `regexp_matches` call being rewritten into a *different function entirely*
  — the unmistakable signature of the expression-level `RegexOptimizationRule`
  from §8.2.3, not of a range filter being spliced in alongside an unchanged
  original predicate. A pair of these blocks even use the exact
  two-blocks-sharing-a-label pattern this book's test-format conventions call
  out elsewhere in this chapter's sibling files (`query I nosort
  regexconstantpattern` appears twice — once wrapping
  `EXPLAIN SELECT regexp_matches(s, 'aa') FROM test`, once wrapping
  `EXPLAIN SELECT contains(s, 'aa') FROM test` — proving the two `EXPLAIN`
  outputs are byte-identical, i.e. that the rewrite really did happen). This
  is the expression-rewriter's test, through and through.
- **`test/optimizer/regex_to_like_optimizer.test`** and
  **`test/optimizer/regex_optimizer_coverage.test`** are both, likewise,
  entirely about `regexp_matches` (case-insensitive inline flags, character
  classes, `(?i)`/`(?s)`/`(?m)` mode groups) with `contains`/comparison-shape
  simplification as the thing under test — same rule, same pass, same
  function name. Neither file contains the string `regexp_full_match`.
- **`test/sql/function/string/regex_filter_pushdown.test`** — the one whose
  filename most directly suggests it belongs to this section — is four small
  `query T` blocks (`SELECT s FROM regex WHERE REGEXP_MATCHES(s,
  'as(c|d|e)f')`, its `NOT` form, and two `AND`-combined forms), checking only
  final row output, with no `EXPLAIN` at all. It uses `REGEXP_MATCHES`
  throughout — again the excluded function for `REGEX_RANGE` — and its actual
  subject, per its own description comment ("regex filter push test"), is
  simply that `FILTER_PUSHDOWN` (the *general* pass, not `REGEX_RANGE`) is
  willing to push an opaque, non-decomposable `regexp_matches(...)` predicate
  down through a projection to sit directly above the scan, and that `AND`-ing
  two such predicates together still produces the correct rows. It is a
  `FILTER_PUSHDOWN` correctness test wearing a name that, at a glance, looks
  like it should be about `REGEX_RANGE`.

The one test in the whole suite that genuinely does exercise `REGEX_RANGE`'s
code path is a regression test, not a demonstration test:

```
# test/fuzzer/public/regex_range_filter_mismatch.test
statement ok
CREATE TABLE v00 ( c01 INTEGER ) ;

query I
SELECT * FROM v00 WHERE concat(c01 IS NOT NULL, 'string') SIMILAR TO 'string';
----
```

`SIMILAR TO` is the one piece of SQL syntax that *always*, unconditionally,
compiles to `regexp_full_match` regardless of the
`regex_match_operator_semantics` setting — confirmed directly in the PEG
transformer:

```cpp
// src/parser/peg/transformer/transform_expression.cpp:1276
string PEGTransformerFactory::TransformSimilarToToken(PEGTransformer &transformer) {
	return "regexp_full_match";
}
```

(contrast with the `~` operator's token transform, which consults
`RegexMatchOperatorFunctionName` and returns `"regexp_matches"` under the
*default* `PARTIAL` semantics — `src/parser/peg/transformer/transform_expression.cpp:929-934`
— meaning `~` only reaches `REGEX_RANGE` at all when a session has explicitly
set `regex_match_operator_semantics='full'`). Because `RegexOptimizationRule`'s
matcher (§8.2.3) only ever fires on the literal name `"regexp_matches"`, a
`SIMILAR TO` expression is *never* touched by pass #1 — it survives, entirely
unmodified, all the way to `REGEX_RANGE` at pass #8. This test's WHERE clause
— `concat(c01 IS NOT NULL, 'string') SIMILAR TO 'string'` — is exactly a
`regexp_full_match` call whose left-hand side is a non-column expression
(`concat` of a boolean and a string literal, implicitly cast to `VARCHAR`),
which `REGEX_RANGE` happily tries to build a derived range comparison for
(the pattern `'string'` has no regex metacharacters at all, so
`PossibleMatchRange` degenerates to the single point `min = max = "string"`).

The bug this regression test guards against is a real, previously-crashing
one, and its fix is directly attributable via `git log --follow`:

```cpp
// src/common/vector_operations/comparison_operators.cpp (fixed in commit ddcb3e7410,
// "Fix #15465: fix assertion in comparison executor to compare physical type instead")
- D_ASSERT(left.GetType() == right.GetType() && result.GetType() == LogicalType::BOOLEAN);
+ D_ASSERT(left.GetType().InternalType() == right.GetType().InternalType() &&
+          result.GetType() == LogicalType::BOOLEAN);
```

`REGEX_RANGE` compares the (`VARCHAR`-typed, post-implicit-cast) left-hand
expression against a `Value::BLOB_RAW(range_min)` constant — `VARCHAR` and
`BLOB` are different *logical* types that share the exact same *physical*
representation (`string_t`), and DuckDB's comparison executor is written to
operate on physical types, so the comparison itself is entirely valid; the
`D_ASSERT` that used to guard it, checking exact logical-type equality, was
simply stricter than the actual invariant the executor relies on. The fix
loosened the assertion to compare `InternalType()` instead, and the fuzzer
test that had found the debug-build crash was kept as a permanent regression
guard. It is a real, if narrow, piece of evidence that `REGEX_RANGE`'s
`VARCHAR`-vs-`BLOB` comparison construction is exercised by at least one
CI-run test on this exact code path — but it is testing "does not crash," not
"produces the plan shape §8.2.1 describes." No test in this suite asserts,
via `EXPLAIN`, that the derived range filter actually got spliced in as a
second `LogicalFilter` node the way §8.2.1's diagram shows. Coverage for this
*specific* plan-level pass, isolated from its expression-level cousin and
from generic filter-pushdown machinery, is genuinely thin, and it is worth
saying so plainly rather than stretching the fuzzer regression test into more
than it actually demonstrates.

### 8.2.5 Edge cases and honest limits

- **`regexp_matches` with an explicit `^` anchor gets nothing**, even though
  the anchoring alone would make the derived range legitimate under RE2's own
  "anchored match" guarantee (§8.2.1) — the gate is a function-name check,
  not a pattern-anchoring check, so `WHERE regexp_matches(x, '^abc')` never
  reaches this pass at all. This is a real, source-confirmed missed
  optimization opportunity, not a hypothetical one.
- **A non-constant pattern never qualifies.** `RegexpMatchesBindData`'s
  constructor only calls `PossibleMatchRange` when `constant_pattern` is
  true; for `regexp_full_match(x, some_column)`, `range_success` is
  hard-set `false` at bind time (`regexp.cpp:84`), and `REGEX_RANGE`'s own
  `if (!info.range_success) continue;` filters it out before ever building
  a comparison.
- **Only calls with exactly two arguments qualify** —
  `func.GetChildren().size() != 2` is checked explicitly (line 32) — so the
  three-argument overload of `regexp_full_match` that takes an explicit
  options string (`regexp_full_match(x, pattern, 'i')`) is excluded outright,
  even when its pattern is otherwise a perfectly good constant. This makes
  sense given how `PossibleMatchRange` is computed (it is unclear from this
  file alone whether the 1000-byte walk correctly accounts for
  case-insensitive or other flag-modified matching; the pass simply declines
  to find out, rather than risk it).
- **`maxlen=1000`** in `PossibleMatchRange`'s call means an extremely long or
  pathologically unbounded constant pattern can, in principle, fail to
  produce a usable range (`range_success = false`) even though it is
  perfectly well-anchored — this is a soft cap, not a hard architectural
  limit, and is unlikely to matter for realistic patterns.

## 8.3 `IN_CLAUSE` — turning a long constant list into a `MARK` join

**Purpose in one sentence:** for an `IN`/`NOT IN` predicate whose value list
is entirely made of compile-time constants and long enough to be worth it,
`IN_CLAUSE` replaces the naive per-row linear scan-and-compare with a `MARK`
join against a small, in-memory column of those constants — turning an
O(n·m) comparison chain into an O(n+m) hash build-and-probe.

### 8.3.1 Source walkthrough and the exact threshold

The threshold constant, cited precisely:

```cpp
// src/include/duckdb/optimizer/in_clause_rewriter.hpp:31
// Minimum number of values to use a hash join
static constexpr idx_t IN_CLAUSE_REWRITE_THRESHOLD = 6;
```

The number `6` needs one careful correction before it is usable: it is
compared against `expr.GetChildrenMutable().size()`, and for a bound
`COMPARE_IN`/`COMPARE_NOT_IN` expression, `GetChildren()[0]` is the left-hand
side being tested and `GetChildren()[1..]` are the values —
confirmed directly in the binder's own type-resolution code:

```cpp
// src/planner/binder/expression/bind_operator_expression.cpp:74
case ExpressionType::COMPARE_IN:
case ExpressionType::COMPARE_NOT_IN:
	ResolveCoalesceType(op, children);   // children[0] is the LHS; children[1..] are the value list
	return LogicalType::BOOLEAN;
```

So `children.size() >= 6` really means **the left-hand side plus at least
five values** — the practical minimum number of *literal values* needed to
trigger the mark-join rewrite is **5, not 6**. This is an easy off-by-one to
get wrong from the constant's name alone, and the test suite, discussed next,
happens to only ever demonstrate the pass at exactly 6 values and at 3
values — it never pins down the boundary at the true minimum of 5, which is
itself worth flagging as a real, if narrow, gap in the test evidence rather
than glossing over.

### 8.3.2 The three-way branch, in full

`00-overview.md`'s §7.4/§7.6.6 describes `IN_CLAUSE` as rewriting "a long,
all-foldable `IN (...)` list into a `MARK`-join." Reading
`InClauseRewriter::VisitReplace` in full shows this to be true but
incomplete: the pass is actually the **unconditional canonicalization point
for every constant-list `IN`/`NOT IN` expression in the plan**, of which "long
list → `MARK` join" is only the largest of three branches. `VisitReplace`
fires — via the generic `LogicalOperatorVisitor` walk `VisitOperatorExpressions`
invokes for every `LOGICAL_FILTER`/`LOGICAL_PROJECTION` operator's expression
tree — on *every* `COMPARE_IN`/`COMPARE_NOT_IN` node it finds, not only ones
that pass `HasRewritableInClause`:

```cpp
// src/optimizer/in_clause_rewriter.cpp:88
unique_ptr<Expression> InClauseRewriter::VisitReplace(BoundOperatorExpression &expr, unique_ptr<Expression> *expr_ptr) {
	if (expr.GetExpressionType() != ExpressionType::COMPARE_IN &&
	    expr.GetExpressionType() != ExpressionType::COMPARE_NOT_IN) {
		return nullptr;
	}
	// ...
	const bool rewrite_to_join = expr.GetChildrenMutable().size() >= IN_CLAUSE_REWRITE_THRESHOLD && all_scalar;
	// ...
	if (expr.GetChildrenMutable().size() == 2) {
		// only one child
		// IN: turn into X = 1
		// NOT IN: turn into X <> 1
		return BoundComparisonExpression::Create(
		    is_regular_in ? ExpressionType::COMPARE_EQUAL : ExpressionType::COMPARE_NOTEQUAL,
		    std::move(expr.GetChildrenMutable()[0]), std::move(expr.GetChildrenMutable()[1]));
	}
	if (!rewrite_to_join) {
		if (expr.GetChildrenMutable()[0]->IsVolatile()) {
			return nullptr;
		}
		// low amount of children or not all scalar
		// IN: turn into (X = 1 OR X = 2 OR X = 3...)
		// NOT IN: turn into (X <> 1 AND X <> 2 AND X <> 3 ...)
		// ...
	}
	// IN clause with many constant children
	// generate a mark join that replaces this IN expression
	// ...
```

So the real shape is a **three-way branch**, driven by children count and by
whether every value is `IsFoldable()`:

1. **Exactly one value** (`children.size() == 2`) → collapses directly into a
   plain `=`/`<>` comparison. Regardless of the threshold.
2. **More than one value, but below the threshold, or containing a
   non-scalar value** → expands into an explicit `OR`-chain of `=`
   comparisons (`IN`) or an `AND`-chain of `<>` comparisons (`NOT IN`).
   If the left-hand side is volatile, the `IN` is left as it is
   (`return nullptr`). The `OR`-chain copies the left-hand side once per
   value, so `nextval('s') IN (1, 2)` would call `nextval` twice.
3. **At or above `IN_CLAUSE_REWRITE_THRESHOLD` children *and* every value
   scalar/foldable** → builds a `MARK` join.

Only branch 3 is what the overview's one-line pipeline summary describes;
branches 1 and 2 are just as real a part of what `OptimizerType::IN_CLAUSE`
does, and — as §8.3.6 shows in full — branch 1 is exactly the code path a
seemingly unrelated NULL-handling test turns out to exercise.

For branch 3, the mark-join construction, in full:

```cpp
// src/optimizer/in_clause_rewriter.cpp:133
// IN clause with many constant children
// generate a mark join that replaces this IN expression
// first generate a ColumnDataCollection from the set of expressions
vector<LogicalType> types = {in_type};
auto collection = make_uniq<ColumnDataCollection>(context, types);
// ... appends every value into `collection`, STANDARD_VECTOR_SIZE rows per chunk ...

// now generate a ChunkGet that scans this collection
auto chunk_index = optimizer.binder.GenerateTableIndex();
auto chunk_scan = make_uniq<LogicalColumnDataGet>(chunk_index, types, std::move(collection));

// then we generate the MARK join with the chunk scan on the RHS
auto mark_index = optimizer.binder.GenerateTableIndex();
auto join = make_uniq<LogicalComparisonJoin>(JoinType::MARK);
join->mark_index = mark_index;
join->AddChild(std::move(root));
join->AddChild(std::move(chunk_scan));
// ... one JoinCondition: LHS = the chunk's single column, COMPARE_EQUAL ...
```

`LogicalColumnDataGet` (not a real table scan — an in-memory,
already-materialized columnar chunk holding just the constant values,
appended `STANDARD_VECTOR_SIZE` rows at a time) becomes the join's
build-friendly right-hand side; the query's own rows (`root`, whatever the
`LogicalFilter`/`LogicalProjection`'s child subtree was) become the left. The
join type, confirmed directly from the enum it is drawn from:

```cpp
// src/include/duckdb/common/enums/join_type.hpp:26
MARK = 7,    // MARK join returns marker indicating whether or not there is a join partner (true), there is no join
             // partner (false)
```

— i.e. exactly the join family whose entire purpose is answering "does this
row have *any* match on the other side," materialized as a boolean column
(`mark_index`) rather than as a duplicated or filtered row stream. The
rewritten `IN (...)` expression itself becomes a
`BoundColumnRefExpression` pointing at that mark column (inverted with an
`OPERATOR_NOT` wrapper for `NOT IN`, when this branch is reached via a
genuine `COMPARE_NOT_IN` node — see §8.3.6 for why ordinary `NOT IN` SQL
usually does *not*, in fact, reach this branch that way). This is the DiDi
"IN-list becomes a hash join" mechanic [10] exactly as the task's framing
expected: `n` constant values plus `m` query rows, built once into a hash
table and probed `m` times, replacing what would otherwise be up to `n`
comparisons per row.

### 8.3.3 Keeping IN clauses visible to the rewriter — `FILTER_PUSHDOWN`'s cooperation

Because `IN_CLAUSE` (pass #9) only ever looks inside `LOGICAL_FILTER` and
`LOGICAL_PROJECTION` operators' own `expressions` (`Rewrite`'s switch,
`in_clause_rewriter.cpp:43-55`, has no case for `LogicalGet` or
`LogicalComparisonJoin`), a rewritable `IN` clause that `FILTER_PUSHDOWN`
(pass #6, running *before* it) had already folded into a `LogicalGet`'s
`table_filters`, or merged directly into a join's `conditions` list, would be
invisible to it — those are different storage locations this pass's
`Rewrite` never visits. `FilterCombiner` and the cross-product pushdown
handler both know this, and both explicitly special-case a rewritable `IN`
clause to keep it sitting in a plain expression position instead:

```cpp
// src/optimizer/pushdown/pushdown_get.cpp:186
// Keep expressions owned by InClauseRewriter in the logical plan so they can become hash joins.
// Also skip throwing IN expressions: scan pushdown loses short-circuit evaluation semantics.
if (expr.GetExpressionType() == ExpressionType::COMPARE_IN &&
    (expr.CanThrow() || InClauseRewriter::HasRewritableInClause(expr))) {
	continue;
}
```

```cpp
// src/optimizer/pushdown/pushdown_cross_product.cpp:92
} else {
	D_ASSERT(side == JoinSide::BOTH || side == JoinSide::NONE);
	if (InClauseRewriter::HasRewritableInClause(*f->filter)) {
		deferred_filters.push_back(std::move(f->filter));
	} else {
		// bindings match both: turn into join condition
		join_expressions.push_back(std::move(f->filter));
	}
}
```

The second of these is the exact mechanism behind §8.3.4's first test case.
When a filter's column bindings straddle both sides of a cross product (so it
cannot be pushed purely left or purely right), the *default* behavior is to
fold it directly into the resulting join's `ON` condition list
(`join_expressions`) — a `LogicalComparisonJoin`'s `conditions`, which
`IN_CLAUSE`'s `Rewrite` switch never inspects. `HasRewritableInClause` is the
guard that prevents that default from swallowing a large `IN` clause: if the
straddling predicate contains one, it is instead pushed into
`deferred_filters`, which get wrapped in a brand-new `LogicalFilter` sitting
*above* the resulting join —

```cpp
// src/optimizer/pushdown/pushdown_cross_product.cpp:138
auto filter = make_uniq<LogicalFilter>();
filter->expressions = std::move(deferred_filters);
filter->children.push_back(std::move(result));
```

— which is precisely a `LOGICAL_FILTER` node, exactly the shape `IN_CLAUSE`
knows how to walk. Without this special case, a large `IN` clause combined
into a cross-join's residual predicate would be silently merged into the
join's own condition list and would never become a `MARK` join at all — not
a correctness bug, just a permanently missed optimization, invisible unless
you went looking for exactly this interaction.

`HasRewritableInClause` itself is written to survive being buried arbitrarily
deep inside other expression shapes, not just to check the top-level node:

```cpp
// src/optimizer/in_clause_rewriter.cpp:16
bool InClauseRewriter::HasRewritableInClause(const Expression &expr) {
	if (/* expr itself qualifies */) {
		return true;
	}
	bool result = false;
	ExpressionIterator::EnumerateChildren(expr, [&](const Expression &child) {
		if (!result && HasRewritableInClause(child)) {
			result = true;
		}
	});
	return result;
}
```

It always recurses into every child regardless of whether the outer
expression itself qualified — which is exactly what §8.3.4's third test case
depends on.

### 8.3.4 Test walkthrough: `test/optimizer/in_clause_rewriter_join.test`

Three schema rows across two tables, then three `EXPLAIN` cases and one
correctness-under-toggle check:

```sql
CREATE TABLE in_left(i INTEGER, j INTEGER);
CREATE TABLE in_right(i INTEGER);
INSERT INTO in_left VALUES (1, 1), (2, 8), (11, 8);
INSERT INTO in_right VALUES (1), (2), (11);
```

**Case 1 — a large `IN` clause nested inside a cross-table `OR`:**

```sql
EXPLAIN SELECT *
FROM in_left, in_right
WHERE in_left.i = in_right.i
  AND (in_left.j IN (1, 2, 3, 4, 5, 6) OR in_right.i > 10);
----
logical_opt	<REGEX>:.*MARK.*
```

The `FROM in_left, in_right` implicit join binds to a `LogicalCrossProduct`.
`FILTER_PUSHDOWN` splits the `WHERE` clause's two top-level `AND` conjuncts:
`in_left.i = in_right.i` references both sides purely as an equality and
becomes the resulting `LogicalComparisonJoin`'s condition; the second
conjunct, `(in_left.j IN (1,2,3,4,5,6) OR in_right.i > 10)`, *also* references
both sides (through the `OR`), so it would normally go the same route — except
`HasRewritableInClause` finds the nested six-element `IN` list (six values,
`children.size() == 7 >= IN_CLAUSE_REWRITE_THRESHOLD`, all foldable) and
diverts it into `deferred_filters` instead (§8.3.3), producing a
`LogicalFilter` above the join. `IN_CLAUSE` (pass #9) then finds that
`LOGICAL_FILTER`, recurses into the `OR`, finds the nested `COMPARE_IN`, and
rewrites it into a `MARK` join — which is exactly what `logical_opt`'s
`<REGEX>:.*MARK.*` check confirms.

(Case 1 still rewrites under the rule in §8.3.5 below: the `IN` is the
*first* child of the `OR`, and the first child of a conjunction always runs.)

**Case 2 — the same shape, three values instead of six:**

```sql
EXPLAIN SELECT *
FROM in_left, in_right
WHERE in_left.i = in_right.i
  AND (in_left.j IN (1, 2, 3) OR in_right.i > 10);
----
logical_opt	<!REGEX>:.*MARK.*
```

Three values means `children.size() == 4`, below the threshold of 6 —
`rewrite_to_join` is false, so `IN_CLAUSE`'s branch 2 fires instead
(§8.3.2): the `IN` list expands into `(in_left.j = 1 OR in_left.j = 2 OR
in_left.j = 3)`, folded into the surrounding `OR`. No `MARK` join anywhere in
the plan — confirmed by the `<!REGEX>` negative match.

**Case 3 — a non-rewritable outer `IN` does not hide a rewritable inner
one:**

```sql
EXPLAIN SELECT *
FROM in_left, in_right
WHERE (CASE WHEN in_left.j IN (1, 2, 3, 4, 5, 6)
	THEN in_left.i ELSE 0 END)
	IN (in_right.i, 10, 20, 30, 40, 50);
----
logical_opt	<REGEX>:.*MARK.*
```

The *outer* `IN`'s value list is `(in_right.i, 10, 20, 30, 40, 50)` — six
children counting the `CASE`, but `in_right.i` is a `BoundColumnRefExpression`,
not foldable, so `all_scalar` is false for the outer node and it is never a
mark-join candidate itself (branch 2 applies to it: an `OR`-chain, one of
whose disjuncts compares against a column rather than a constant). But the
`CASE`'s own `WHEN` condition contains a second, wholly independent `IN`:
`in_left.j IN (1, 2, 3, 4, 5, 6)` — six *foldable* values, squarely over
threshold. `HasRewritableInClause`'s unconditional recursion (§8.3.3) finds
this nested one regardless of the outer node's own disqualification, and
`VisitReplace`'s generic expression-tree walk (which visits every matching
node, not just the first one found) rewrites *that* one into a `MARK` join,
while the outer `IN` still expands into its `OR`-chain. The `MARK` string
`logical_opt` matches on is the *inner* rewrite; the outer `IN` never becomes
one.

The file closes with a direct correctness check that the rewrite changes
nothing about the *answer*, only how it's computed — run twice, once with
`IN_CLAUSE` enabled and once with it explicitly disabled via
`disabled_optimizers`:

```sql
foreach do '' 'in_clause'
statement ok
SET disabled_optimizers = {do};

query III
SELECT *
FROM in_left, in_right
WHERE (CASE WHEN in_left.j IN (1, 2, 3, 4, 5, 6)
	THEN in_left.i ELSE 0 END)
	IN (in_right.i, 10, 20, 30, 40, 50)
ORDER BY ALL;
----
1	1	1
endloop
```

Working it by hand confirms the expected single row: for `in_left` row
`(1,1)`, the `CASE` evaluates to `1` (since `j=1` is in `(1..6)`); for
`(2,8)` and `(11,8)` it evaluates to `0` (since `j=8` is not). The outer `IN`
then only matches when the case result equals one of `in_right.i, 10, 20,
30, 40, 50` — which only happens for `in_left.i=1` paired against
`in_right.i=1` (`1 IN (1, 10, 20, ...)` is true). Both optimizer states
(`IN_CLAUSE` on, `IN_CLAUSE` off) must produce exactly `(1, 1, 1)` — and per
the test, do.

The file ends with a check of the volatile rule from §8.3.2 branch 2:

```sql
-- test/optimizer/in_clause_rewriter_join.test
CREATE SEQUENCE in_seq START 2;
SELECT nextval('in_seq') IN (1, 2);      -- true
SELECT currval('in_seq');                -- 2   (nextval ran once, not twice)
SELECT nextval('in_seq') NOT IN (1, 2);  -- true
SELECT currval('in_seq');                -- 3
```

### 8.3.5 The mark join must not move a probe out of `TRY` or a guarded arm

The pass does not visit every `IN` expression. Some are skipped on purpose.

**The problem.** The mark-join rewrite moves the `IN`'s left-hand side, the
*probe*, out of the expression and into a join *below* the filter or
projection. The join then evaluates the probe for *every* row. That is fine
for a plain `IN`. It is wrong when the `IN` sits somewhere that is only
evaluated for some rows, or where errors are caught:

```sql
CREATE TABLE t AS SELECT * FROM (VALUES (0::BIGINT), (100::BIGINT)) v(b);
SELECT b, TRY((100 // b) IN (1, 3, 7, 11, 13)) FROM t;
-- with a mark join: Invalid Input Error: Division by zero
-- expected:         NULL for b = 0, true for b = 100
```

`TRY(...)` should turn the division error into `NULL`. But a mark join would
run `100 // b` outside the `TRY`, so the error would escape. The same holds for
an `IN` inside a `CASE` arm that is only taken for some rows, or after the
first operand of `AND`/`OR`/`COALESCE`.

**The guard.** `InClauseRewriter` overrides `VisitExpression`, so it decides
which subexpressions to walk into:

```cpp
// src/optimizer/in_clause_rewriter.cpp:63
void InClauseRewriter::VisitExpression(unique_ptr<Expression> *expression) {
	auto &expr = **expression;
	const auto expression_type = expr.GetExpressionType();
	if (expression_type == ExpressionType::OPERATOR_TRY) {
		// an IN below an error capture must stay an expression, the mark join would raise outside the capture
		return;
	}
	// in conjunctions, CASE, and COALESCE only the first child evaluates unconditionally, the mark
	// join would evaluate a guarded arm's probe for every row (the Select paths run later children
	// on a remaining-row selection, so untaken arms never raise)
	if (expr.GetExpressionClass() == ExpressionClass::BOUND_CONJUNCTION ||
	    expr.GetExpressionClass() == ExpressionClass::BOUND_CASE ||
	    expression_type == ExpressionType::OPERATOR_COALESCE) {
		bool first = true;
		ExpressionIterator::EnumerateChildren(expr, [&](unique_ptr<Expression> &child) {
			if (first) {
				VisitExpression(&child);
			}
			first = false;
		});
		return;
	}
	LogicalOperatorVisitor::VisitExpression(expression);
}
```

In words: never look inside `TRY`. Inside `AND`, `OR`, `CASE` and `COALESCE`,
only look at the *first* child, because only the first child is always
evaluated. Everything else is walked as usual. An `IN` that is skipped stays a
normal `COMPARE_IN` expression, which the executor can run directly (just
without the hash-join speed-up).

**The test.** `test/sql/error/test_try_in_clause.test` checks both the guarded
cases and the cases that must still rewrite:

```sql
-- test/sql/error/test_try_in_clause.test
SELECT b, TRY((100 // b) IN (1, 3, 7, 11, 13)) FROM t ORDER BY b;
-- 0    NULL
-- 100  true
SELECT b, CASE WHEN b = 0 THEN false ELSE (100 // b) IN (1, 3, 7, 11, 13) END FROM t ORDER BY b;
-- 0    false
-- 100  true
EXPLAIN SELECT b, TRY((100 // b) IN (1, 3, 7, 11, 13)) FROM t;
-- physical_plan   <!REGEX>:.*MARK.*     (below TRY: no mark join)
EXPLAIN SELECT (b + 1) IN (1, 3, 7, 11, 13, 17, 19, 23) AND b <> 0 FROM t;
-- physical_plan   <REGEX>:.*MARK.*      (first conjunct: still rewritten)
EXPLAIN SELECT NOT ((b + 1) IN (1, 3, 7, 11, 13, 17, 19, 23)) FROM t;
-- physical_plan   <REGEX>:.*MARK.*      (NOT always evaluates its child)
SELECT (100 // b) IN (1, 3, 7, 11, 13) FROM t ORDER BY b;
-- error: Division by zero               (no guard, so the error is correct)
```

The file also covers `list_contains([1, 3, 7, 11, 13], 100 // b)` under `TRY`.
`ContainsToInClauseRule` (a pass-#1 rule) turns that call into an `IN`, so it
needs the same guard.

One trade-off remains: `HasRewritableInClause` (§8.3.3) does not apply this
rule. So filter pushdown still keeps a filter that holds a large `IN` out of
a join condition, even when that `IN` sits under `TRY` and will not be
rewritten.

### 8.3.6 `NOT IN` and `NULL`: a three-valued-logic deep dive (`test/optimizer/not_in_null.test`)

```sql
CREATE TABLE integers(i INTEGER);
INSERT INTO integers VALUES (1), (2), (NULL);

EXPLAIN SELECT count(*) FROM integers WHERE i NOT IN (NULL);
----
physical_plan	<REGEX>:.*Empty Result.*

SELECT count(*) FROM integers WHERE i NOT IN (NULL);
----
0

EXPLAIN SELECT count(*) FROM integers WHERE i NOT IN (NULL) AND i > 0;
----
physical_plan	<REGEX>:.*Empty Result.*

SELECT count(*) FROM integers WHERE i NOT IN (NULL) AND i > 0;
----
0
```

The semantic answer is textbook three-valued logic, and it holds regardless
of which pass proves it: `x NOT IN (v1, ..., vn)` is defined as `x <> v1 AND
x <> v2 AND ... AND x <> vn`. With the list containing only `NULL`, that
reduces to the single comparison `x <> NULL`, which — per SQL's `NULL`
propagation rule that *any* comparison against `NULL` yields `NULL`, never
`TRUE` or `FALSE` — evaluates to `NULL` for *every* row, including rows where
`i` is itself non-`NULL` (this is the classic "`NOT IN` with a `NULL` in the
list poisons the whole predicate" gotcha many SQL engines share). A `WHERE`
clause only keeps rows where the predicate is `TRUE`; `NULL` is treated the
same as `FALSE`. So `count(*)` must be `0` for every row in the table,
unconditionally — the physical plan collapsing all the way to a
`PhysicalEmptyResult` node (not merely "zero rows survive a normal filter at
runtime," but the *plan itself* proving, before execution starts, that no
row ever could survive) is a stronger, plan-time version of the same fact.

Tracing exactly *which* mechanism proves this, at exactly which point in the
pipeline, is worth doing in full, because it turns out to intersect directly
with this book's own stated theme about the very recent parser rewrite
(front matter, "What version of DuckDB, precisely"), in a way this
investigation did not expect going in.

**Step 1 — how `NOT IN` actually parses.** The PEG transformer that handles
`BETWEEN`/`IN`/`LIKE` with an optional leading `NOT` keyword does not build a
distinct "negated IN" node type at all; for the `IN` case specifically
(`ExpressionClass::OPERATOR`), it builds the ordinary `COMPARE_IN` operator
expression first, and only afterward, if a `NOT` keyword was present, wraps
the *whole thing* in a separate `OPERATOR_NOT`:

```cpp
// src/parser/peg/transformer/transform_expression.cpp:1102
} else if (between_in_like_expr->GetExpressionClass() == ExpressionClass::OPERATOR) {
	auto &operator_expr = between_in_like_expr->Cast<OperatorExpression>();
	operator_expr.GetChildrenMutable().insert(operator_expr.GetChildrenMutable().begin(), std::move(expr));
	if (has_not) {
		expr = make_uniq<OperatorExpression>(ExpressionType::OPERATOR_NOT, std::move(between_in_like_expr));
	} else {
		expr = std::move(between_in_like_expr);
	}
}
```

So `i NOT IN (NULL)` parses, at the parse-tree level, as literally
`NOT(i IN (NULL))` — `OPERATOR_NOT` wrapping `COMPARE_IN` — not as some
single, pre-negated node.

**Step 2 — the binder does not fold this back together.** Reading
`ExpressionBinder::BindExpression(OperatorExpression&, idx_t)`
(`src/planner/binder/expression/bind_operator_expression.cpp:158` onward) in
full turns up no special-case logic anywhere that recognizes "`OPERATOR_NOT`
whose single child is a bound `COMPARE_IN`" and collapses it into a single
`COMPARE_NOT_IN` node. Nor do either of the two expression-rewriter rules
whose names suggest they might (`NotComparisonSimplificationRule`,
`NotConjunctionSimplificationRule`, both pass-#1 rules, §7.5) — the first
only matches a `NOT` wrapping something `BoundComparisonExpression::IsComparison`
recognizes (`=`,`<>`,`<`,`>`,`<=`,`>=`,`IS [NOT] DISTINCT FROM` — `COMPARE_IN`
is not in that list, being a `BoundOperatorExpression`, not a comparison
*function*, in this codebase's representation), and the second only matches
a `NOT` wrapping a `BOUND_CONJUNCTION` (`AND`/`OR`), not a `BOUND_OPERATOR`.
Neither rule's matcher fires on `NOT(COMPARE_IN(...))`. The bound tree
genuinely keeps the `OPERATOR_NOT`-wrapping-`COMPARE_IN` shape all the way
through pass #1.

**Step 3 — `FILTER_PUSHDOWN` (pass #6) captures exactly this shape into the
table filter.** `i NOT IN (NULL)` references a single column, so the whole
`OPERATOR_NOT(COMPARE_IN(i, NULL))` expression is eligible to be pushed all
the way down and wrapped in an `ExpressionFilter` on `integers`'s
`LogicalGet.table_filters` — the general, arbitrary-expression fallback path
(`FilterCombiner::TryPushdownGenericExpression`,
`src/optimizer/filter_combiner.cpp:447`), since `COMPARE_NOT_IN`-shaped or
`NOT`-wrapped predicates have no dedicated `TryPushdownXFilter` handler of
their own. This is the copy of the predicate that matters for what follows —
and it is a *separate* copy from whatever the residual `LogicalFilter`
expression tree (if any residual filter remains at all) goes on to become.

**Step 4 — `IN_CLAUSE` (pass #9) only ever rewrites the inner `COMPARE_IN`
node, not the outer `NOT`.** `VisitReplace`'s type check
(`expr.GetExpressionType() != COMPARE_IN && != COMPARE_NOT_IN`) means the
outer `OPERATOR_NOT` node itself is not something this pass's replacement
logic ever fires on directly — but the generic expression-tree walk still
descends into its child and finds the genuine `COMPARE_IN(i, NULL)` node
sitting inside it. That inner node has exactly two children (`i`, the
constant `NULL`) — branch 1 of §8.3.2's three-way split — and because it is
a true `COMPARE_IN` at this point (not `COMPARE_NOT_IN`), `is_regular_in` is
`true`, so it becomes `i = NULL`, **not** `i <> NULL`. The outer `NOT`
wrapper, untouched by this pass, remains in place around it. So whatever
residual expression tree survives past `IN_CLAUSE`, if any, is `NOT(i =
NULL)` — a subtly different shape than the single direct comparison a first
reading of §8.3.2's branch-1 comment ("NOT IN: turn into X <> 1") would
suggest, precisely because that comment describes what happens when
`VisitReplace` is handed a genuine `COMPARE_NOT_IN` node directly, and — per
Steps 1–2 above — ordinary `x NOT IN (single_value)` SQL, under this PEG
parser, never actually constructs one. *(This checkout's `in_clause_rewriter.cpp`
still carries full, explicit handling for a directly-typed `COMPARE_NOT_IN`
node throughout `VisitReplace` — this investigation could not find a live
SQL syntax path in this exact checkout that constructs one directly rather
than as `NOT(COMPARE_IN(...))`; the most likely explanation, though not
independently confirmed here, is that this branch exists for expression
trees built through some non-parser path — an extension, the C API, or a
deserialized plan from before the front matter's stated parser rewrite —
rather than for any plain SQL text a user would type today.)*

**Step 5 — the table filter, unaffected by Step 4, still holds the
pre-`IN_CLAUSE` shape, and that is exactly what a dedicated, recently-added
check recognizes.** `git log --follow` on the test file turns up the exact
commit that added this behavior: `a703ef1a3c`, *"Expression filter handle NOT
IN NULL"*, touching only `src/planner/filter/expression_filter.cpp` and this
test. The function it added:

```cpp
// src/planner/filter/expression_filter.cpp (CheckNotOperatorStatistics, current form)
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
```

— matching exactly `OPERATOR_NOT` wrapping a two-child `COMPARE_IN` whose
single value is a literal `NULL`: exactly the shape Steps 1–3 traced the
table filter as holding. This is dispatched from
`CheckOperatorStatistics`'s `case ExpressionType::OPERATOR_NOT:` and, in
turn, from `StatisticsPropagator::PropagateTableFilter`
(`src/optimizer/statistics/operator/propagate_get.cpp:41`,
`auto check_result = filter.Cast<ExpressionFilter>().CheckStatistics(stats);`)
— i.e. it runs as part of `STATISTICS_PROPAGATION`, pass #34, using
whatever table-level column statistics are available for `i` (notably: this
particular check does not even need real statistics — it returns
`FILTER_FALSE_OR_NULL` from the expression's own syntactic shape alone,
independent of what `stats` says about `i`'s actual min/max or null
fraction). A `FILTER_ALWAYS_FALSE`/`FILTER_FALSE_OR_NULL` result on a
`LogicalGet`'s filter is exactly the trigger for `ReplaceWithEmptyResult`
(§7.6.2), which is what produces the `Empty Result` node the test's `EXPLAIN`
checks for.

**Why a dedicated special case was needed at all — the three-valued-logic
subtlety.** It would be natural to assume the *general* `NOT`-inversion logic
sitting right below `CheckNotOperatorStatistics`'s new special case should
already have handled this:

```cpp
// src/planner/filter/expression_filter.cpp, immediately following the new check above
auto child_result = ExpressionFilter::CheckExpressionStatistics(context_p, child, input_stats);
if (child_result == FilterPropagateResult::FILTER_ALWAYS_TRUE) {
	return FilterPropagateResult::FILTER_ALWAYS_FALSE;
}
if (child_result == FilterPropagateResult::FILTER_TRUE_OR_NULL) {
	return FilterPropagateResult::FILTER_FALSE_OR_NULL;
}
return FilterPropagateResult::NO_PRUNING_POSSIBLE;
```

It does not, and the reason is a genuine, instructive three-valued-logic
trap. The *existing*, older `CheckInOperatorStatistics` — the handler for a
bare `COMPARE_IN` — already special-cases an all-`NULL` value list:

```cpp
// src/planner/filter/expression_filter.cpp:763 (CheckInOperatorStatistics)
if (values.empty()) {
	return FilterPropagateResult::FILTER_ALWAYS_FALSE;
}
```

(`values` only accumulates non-`NULL` constants; a list containing only
`NULL` leaves it empty.) That classification is *correct for a plain filter*
— `FILTER_ALWAYS_FALSE` and "always evaluates to `NULL`" are operationally
identical from the point of view of "does this row survive `WHERE`," so
collapsing the true answer (always `NULL`) down to the coarser
`FILTER_ALWAYS_FALSE` loses nothing *for that purpose*. But it loses
something crucial the moment you then try to **invert** it: `NOT(FALSE) =
TRUE`, while `NOT(NULL) = NULL` — two very different answers — and the
general inversion code above only knows how to safely invert
`FILTER_ALWAYS_TRUE` (→ `FILTER_ALWAYS_FALSE`) and `FILTER_TRUE_OR_NULL` (→
`FILTER_FALSE_OR_NULL`); it has no branch for a child result of
`FILTER_ALWAYS_FALSE` at all, precisely because a `FILTER_ALWAYS_FALSE`
coming back from `CheckInOperatorStatistics` is ambiguous about whether the
*true* underlying truth value was really always-`FALSE` or was really
always-`NULL`-conflated-into-`FALSE`-for-filter-purposes — and only one of
those two safely inverts to `TRUE`. Rather than trying to recover which case
it was from the conflated classification after the fact, `a703ef1a3c` adds a
direct, syntactic re-derivation: look at the child's *shape* directly (a
`COMPARE_IN` whose only value is a literal `NULL`), which is unambiguous, and
skip the lossy round-trip through `CheckInOperatorStatistics`'s classification
entirely. It is a small, precise fix, and the two-line commit that follows it
(`ae3435f4fd`, "regression test for AND") exists specifically to confirm the
new check composes correctly through `CheckConjunctionStatistics`'s `AND`
handling (`FILTER_FALSE_OR_NULL` on one conjunct short-circuits the whole
`AND` to `FILTER_FALSE_OR_NULL`, matching the test's second `EXPLAIN` block
with `i NOT IN (NULL) AND i > 0`).

*(Method note: this whole five-step chain was cross-checked empirically
against a locally `pip`-installed stable DuckDB `1.5.5` — not the
`v2.0-cyanoptera` snapshot this book otherwise sources from, since no build
of that exact commit was produced for this file. Under `1.5.5`'s
libpg_query-derived parser — the pre-rewrite architecture that the main
line replaced with the PEG parser in April 2026 (libpg_query removed
2026-04-20, PEG parser merged 2026-04-25, PR #22194; the v1.5.x releases
kept libpg_query, v2.0.0 is the first release with the PEG parser; see the
front matter) — `EXPLAIN`
with `filter_pushdown` disabled shows the residual filter printed as
`(i != NULL)`, i.e. a direct `<>` comparison, not `NOT(i = NULL)` — consistent
with an older parser binding `NOT IN` directly to `COMPARE_NOT_IN` rather
than wrapping `COMPARE_IN` in `OPERATOR_NOT`. `1.5.5` does not produce an
`Empty Result` plan for this query at all — the dedicated
`CheckNotOperatorStatistics` machinery this section describes is `v2.0`-only.
This is offered as corroborating, not authoritative, evidence — the actual
citations above are what this chapter's claims rest on.)*

### 8.3.7 The boundary with the expression-level `InClauseSimplificationRule`

`00-overview.md`'s §7.5 already draws this line, and it is worth restating
briefly rather than re-deriving: `InClauseSimplificationRule`
(`src/optimizer/rule/in_clause_simplification_rule.cpp`, one of pass #1's 37
rules) solves an entirely different problem — a *cast* sitting on the
column side of an `IN` list (`CAST(x AS BIGINT) IN (1, 2, 3)`), pushed onto
the constants instead so the column itself stays comparable in its native
type against a zonemap or index. It never changes how many items are in the
list or whether the list becomes a join; `IN_CLAUSE` (this file's subject)
never touches casts at all and only cares about list length and
foldability. The two rules share a name by English-vocabulary coincidence
only, one operating at pass #1 on cast placement, the other at pass #9 on
join-worthiness.

### 8.3.8 Edge cases and honest limits

- **The exact 5-value boundary is untested.** §8.3.1 already flagged this:
  the test suite demonstrates 6 values (rewrites) and 3 values (does not),
  but never the true minimum of 5 values (`children.size() == 6`, which
  should also rewrite per the `>=` comparison in the source) or 4 values
  (`children.size() == 5`, which should not). A reasonable, source-grounded
  inference, not a directly tested fact.
- **All values must be foldable, not merely constant-looking.** A value that
  is itself a subquery result, a correlated column reference, or a volatile
  expression disqualifies the *entire* list from branch 3 even if every
  other value is a plain literal and the list is long — `all_scalar` is an
  all-or-nothing gate (§8.3.2), demonstrated directly by case 3's outer `IN`
  in §8.3.4 (one non-foldable value among six total children is enough to
  keep the whole node out of the mark-join branch).
- **Whether a `MARK` join then survives, unconverted, as a genuine physical
  `MARK`-type join**, or is later simplified by a downstream pass into a
  `SEMI`/`ANTI` join when nothing besides the boolean result is ever read, is
  a physical-plan and later-optimizer-pass question (`CheckMarkToSemi`,
  referenced in `optimizer.cpp`'s own `FILTER_PUSHDOWN` call site) this file
  did not chase further — it belongs to Chapter 8/9's join-family discussion,
  not to `IN_CLAUSE` itself, which only ever constructs the `JoinType::MARK`
  shape and stops there.

## 8.4 Summary

Both passes in this file take the same basic shape — recognize one narrow,
syntactically-identifiable expression pattern in an already-pushed-down tree,
and rewrite just that pattern — but land at very different points on the
"how thoroughly is this tested" and "how much does the fixed pipeline's exact
order matter" axes. `REGEX_RANGE` is a small, cleanly-scoped correctness-first
pass (its entire design center of gravity is "never let the derived range
substitute for the real regex check"), genuinely under-tested in isolation
from its expression-level cousin, and its practical payoff is smaller than a
first read suggests once its exact position *after* `FILTER_PUSHDOWN` is
taken into account — a real, source-verified correction to the natural
"usable by zonemaps" assumption this chapter's own task started from.
`IN_CLAUSE` is richer, better-tested, and careful about where it may act: it
never pulls an `IN` probe out of a `TRY` or a guarded `CASE`/`AND`/`OR`/
`COALESCE` arm (§8.3.5), and it never copies a volatile left-hand side. It also
turned out to be an unexpectedly
good lens on this book's own stated theme about the very recent parser
rewrite: the exact shape a `NOT IN` predicate takes by the time it reaches
`STATISTICS_PROPAGATION`'s table-filter statistics check depends on which
parser produced it, and the machinery that proves `NOT IN (NULL)` always
unsatisfiable had to be added, quite recently, specifically because the shape
changed.

Both passes hand off to the same next stage as every other pass in this
chapter: whatever tree they leave behind is what `DELIMINATOR` (#10) and
`GROUPING_SETS` (#11) inherit next, with no mechanism for either pass to be
revisited later in this same query's optimization run, per §7.2's
architecture. Where either pass's output could, in principle, unlock a
rewrite an earlier pass already walked past — the way `REGEX_RANGE`'s
derived comparison filter, had it existed one pass earlier, could have
become a genuine zonemap-usable `TableFilter` — that opportunity is simply
gone for this run, which is the recurring, load-bearing fact this whole
chapter keeps landing back on.
