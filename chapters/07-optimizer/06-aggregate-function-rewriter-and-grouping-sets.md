# Chapter 7, Part 6 — `AGGREGATE_FUNCTION_REWRITER` and `GROUPING_SETS`

This part is a deep dive on two of the ~45 plan-level passes cataloged in
§7.4 of the chapter overview: **`AGGREGATE_FUNCTION_REWRITER`** (pass #4,
`src/optimizer/aggregate_function_rewriter.cpp`) and **`GROUPING_SETS`**
(pass #11, `src/optimizer/grouping_sets_optimizer.cpp`). Both operate on
`LogicalAggregate`, and both turned out — on direct inspection of the
source — to be doing noticeably more, and something noticeably more
specific, than their one-line pipeline descriptions suggest. This part
supersedes §7.6.5's short treatment of "the `sum_rewriter` question" with
the complete picture: not two rules but **three**, a real and easy-to-miss
scope restriction on the `SUM` rule that has nothing to do with types, and a
clean disambiguation of a name (`sum_no_overflow`) that belongs to a
*different* pass entirely but shows up inside this file's own test suite.
It then does the same for grouping sets: the source answers, precisely,
whether `GROUPING_SETS` is doing syntax expansion or post-hoc
simplification, and the answer is neither of the two options the question
as posed assumes.

All code is quoted verbatim from the DuckDB repository (https://github.com/duckdb/duckdb) at
commit `cb5673685153` (`v2.0.0-1041-gcb567368515`) on branch
`v2.0-cyanoptera`, cited `file:line`.

## 7.6.5′ `AGGREGATE_FUNCTION_REWRITER`, in full

### The architecture: one driver, three independent rules, run as separate passes

`aggregate_function_rewriter.cpp` does not implement one rewrite. It
implements a small internal rule engine — structurally similar to the
matcher/rule split covered in §7.5 for the 37 scalar-expression rules, but
purpose-built for patterns that live *inside a `LogicalAggregate`'s
expression list* rather than inside an arbitrary expression tree. It uses
exactly three rule objects. They are built inside `Optimize`, each time the
pass runs (shown in full below):

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:504
	vector<unique_ptr<AggregateRewriteRule>> rules;
	rules.push_back(make_uniq<AvgRewriteRule>(optimizer, *op));
	rules.push_back(make_uniq<SumRewriteRule>(optimizer));
	rules.push_back(make_uniq<ListRewriteRule>(optimizer));
```

The constructor is empty (`aggregate_function_rewriter.cpp:496`), and `rules`
is a local variable of `Optimize`. The reason is that `AvgRewriteRule` needs
the plan's root (`*op`), to ask whether an expression can be `NULL`. That root
is only known when `Optimize` is called. See `ShouldRewrite` in §7.6.5′.1.

`AvgRewriteRule` and `SumRewriteRule` are exactly the two rewrites named in
`optimizer.cpp`'s own inline comment (*"Rewrites `AVG(x)` -> `SUM(x)/COUNT(x)`
and `SUM(x+C)` -> `SUM(x) + C*COUNT(x)`"*), and they're what DiDi's course
material calls the `sum_rewriter` [10]. `ListRewriteRule` is not mentioned
anywhere in `optimizer.cpp`'s comments or in the DiDi slide vocabulary this
chapter has been checking claims against — it only turns up by reading the
`.cpp` file itself, which is exactly the kind of thing "read this file
thoroughly, it likely contains several distinct rewrite rules bundled under
one pass" was asking to surface. It rewrites `list(x ORDER BY x)` into
`list_sort(list(x), ...)` — covered in full in §7.6.5′.3 below.

Each rule is driven through the same generic machinery,
`AggregateFunctionRewriterInternal`, but — unlike the 37 expression rules
in §7.5, which all share one tree walk — each of these three rules gets its
**own, separate, complete walk of the entire plan tree**:

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:502
void AggregateFunctionRewriter::Optimize(unique_ptr<LogicalOperator> &op) {
	RewriteAggregateCallbacks(optimizer, op);
	vector<unique_ptr<AggregateRewriteRule>> rules;
	rules.push_back(make_uniq<AvgRewriteRule>(optimizer, *op));
	rules.push_back(make_uniq<SumRewriteRule>(optimizer));
	rules.push_back(make_uniq<ListRewriteRule>(optimizer));
	// Run each rule as an independent pass so that transformations by an earlier rule affect later rules
	for (auto &rule : rules) {
		AggregateFunctionRewriterInternal rewriter(optimizer, *rule);
		rewriter.Optimize(op);
	}
}
```

The comment states the reason directly: running `AvgRewriteRule` to
completion over the whole tree *before* `SumRewriteRule` even starts means
that if the `AVG` rewrite happens to expose a `SUM(x + C)` shape somewhere
(it doesn't in the three registered rules today, but the architecture is
built to allow it), the later rule gets to see it. This is the one place in
this file where DuckDB's usual "no re-triggering, one shot per pass"
discipline (§7.2) is deliberately relaxed — not across the ~45 plan-level
passes, but *within* this one `OptimizerType::AGGREGATE_FUNCTION_REWRITER`
pass, at the granularity of "one rule, one more full tree walk."

Before any rule-specific matching happens at all, a separate helper handles
a related but distinct concern — **direct** aggregate rewrites that some
aggregate *functions* register on themselves (via `Function().HasRewriteCallback()`),
independent of the three pattern-matching rules above:

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:23
static void RewriteAggregateCallbacks(Optimizer &optimizer, unique_ptr<LogicalOperator> &op) {
	if (op->type == LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY) {
		auto &aggregate_op = op->Cast<LogicalAggregate>();
		for (auto &expr : aggregate_op.expressions) {
			auto &aggregate = expr->Cast<BoundAggregateExpression>();
			AggregateRewriteInput input(optimizer, aggregate_op, aggregate);
			auto rewrite = TryDirectAggregateRewrite(input);
			if (!rewrite) {
				continue;
			}
			expr = std::move(rewrite);
		}
	}
	for (auto &child : op->children) {
		RewriteAggregateCallbacks(optimizer, child);
	}
}
```

This is the **same** `AggregateRewritePolicy`/`RewriteCallback` framework
that §7.6.5′'s grouping-sets discussion below has to reckon with for
`histogram()`, `mode()`, and `entropy()` — a *third*, function-level
rewriting mechanism, distinct from both this file's three matcher/rule
classes and from the unrelated `MultiStageAggregateRewriter`/`AggregateRewritePolicy`
machinery that pass #11 also uses for `DISTINCT` aggregates. Three
different "aggregate rewriting" frameworks share this corner of the
optimizer; this section is about the matcher/rule one specifically
(`AvgRewriteRule`/`SumRewriteRule`/`ListRewriteRule`), and calls out the
other two only where they interact directly with what's being described.

### The shared rewrite mechanics: a projection materializes the algebra

All three rules share one execution shape, worth understanding once before
looking at each rule's specific matcher. `AggregateFunctionRewriterInternal::RewriteAggregates`
(`aggregate_function_rewriter.cpp:392`) walks every expression in a
`LogicalAggregate`'s `expressions` list, and for each one that the active
rule's matcher accepts (and whose `ShouldRewrite` hook returns `true`, see
§7.6.5′.1):

1. Calls the rule's `Rewrite()`, which mutates the aggregate expression *in
   place* inside the `LogicalAggregate` (turning `SUM(x + C)` into `SUM(x)`,
   for instance) and returns whatever extra argument expression the
   rewrite needs a `COUNT` computed over (`x`, for `SUM`; the original
   `AVG` argument, for `AVG`).
2. Appends a fresh `COUNT(...)` aggregate to the *same* `LogicalAggregate`'s
   expression list — `count(x)` if a count argument was returned,
   `count_star()` otherwise (`aggregate_function_rewriter.cpp:413-427`).
   This is why the `AvgRewriteRule` test in §7.6.5′.2's walkthrough (below)
   shows `count_star()` for `COUNT(i)` specifically: the rewrite reuses
   DuckDB's existing `count(*)` vs `count(x)` split, it doesn't invent a new
   aggregate.
3. Once every rewritable expression in that one `LogicalAggregate` has been
   processed, pushes a brand-new `LogicalProjection` directly on top of it
   (`aggregate_function_rewriter.cpp:482-487`) that recomputes each
   *original* result expression from the now-decomposed pieces —
   `SUM(x) + C * COUNT(x)` for the `SumRewriteRule` case, `SUM(x) / COUNT(x)`
   for `AvgRewriteRule` — via each rule's own `CreateProjectionExpression`.
   Anything the aggregate produces that *wasn't* rewritten (a plain,
   untouched `SUM(y)` sitting in the same query) is passed straight through
   this projection unchanged.
4. Registers every old-to-new column binding in an `aggregate_map`, so that
   any operator *above* this projection that referenced the original
   `LogicalAggregate`'s output columns gets rebound transparently
   (`VisitReplace` at line 384).

So the plan-shape effect of every one of these three rules, when it fires,
is identical: **the `LogicalAggregate` gets a wider `expressions` list (the
original aggregate calls, decomposed and/or accompanied by helper `COUNT`s)
and a new `LogicalProjection` appears immediately above it**, which is the
concrete plan-shape signature to look for in `EXPLAIN` output — and exactly
what the `test_aggregate_function_rewrite.test` `EXPLAIN` assertions below
are checking for when they grep the optimized plan text for a substring.

### 7.6.5′.1 `AvgRewriteRule` — `AVG(x)` → `SUM(x) / COUNT(x)`

**Purpose, in one sentence:** decompose `AVG` into a `SUM`/`COUNT` pair so
that every other pass downstream that already knows how to optimize `SUM`
and `COUNT` (statistics propagation's overflow specialization, §7.6.5′'s own
`SumRewriteRule`, `PARTIAL_AGGREGATE_PUSHDOWN`'s star-schema pre-aggregation)
gets to work on `AVG` too, for free, without needing its own bespoke
`AVG`-aware logic.

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:71
class AvgRewriteRule : public AggregateRewriteRule {
public:
	AvgRewriteRule(Optimizer &optimizer, LogicalOperator &root)
	    : AggregateRewriteRule(optimizer), nullability(optimizer.context, root) {
		auto op = make_uniq<AggregateExpressionMatcher>();
		op->function = make_uniq<SpecificFunctionMatcher>("avg");
		op->type = make_uniq<NumericTypeMatcher>();
		op->policy = SetMatcher::Policy::ORDERED;
		auto child_matcher = make_uniq<StableExpressionMatcher>();
		child_matcher->type = make_uniq<NumericTypeMatcher>();
		op->matchers.push_back(std::move(child_matcher));
		matcher = std::move(op);

		auto count = make_uniq<AggregateExpressionMatcher>();
		count->function = make_uniq<ManyFunctionMatcher>(identifier_set_t {"count", "count_star"});
		count->policy = SetMatcher::Policy::SOME;
		count_matcher = std::move(count);
	}
```

The second matcher, `count_matcher`, and the `nullability` helper are used
only by `ShouldRewrite`, below.

The matcher is deliberately permissive on types: `NumericTypeMatcher::Match`
is `type.IsNumeric()` (`src/include/duckdb/optimizer/matcher/type_matcher.hpp:57-62`)
— `DECIMAL`, `DOUBLE`, every integer width, everything numeric qualifies,
both for `AVG`'s own return type and for its single argument (which must
also be a `StableExpressionMatcher`, i.e. non-volatile — see §7.6.5′'s
`SumRewriteRule` section for what "stable" excludes). There is no
integer-only restriction here, and there doesn't need to be: `SUM(x)/COUNT(x)`
is computed once, as a single division, not by re-associating a chain of
additions — there is no reordering of floating-point operations for this
identity to go wrong on, unlike `SumRewriteRule`'s case below.

`ShouldSkip` encodes a real, non-obvious constraint:

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:91
	bool ShouldSkip(const LogicalAggregate &aggr) const override {
		return !aggr.grouping_functions.empty();
	}
```

(`AVG -> SUM/COUNT` is correct under any grouping, and only `GROUPING()`
blocks it.)

So `ShouldSkip` lets `AvgRewriteRule` through under a plain `GROUP BY`, under
`ROLLUP`/`CUBE`/explicit `GROUPING SETS` — anywhere *except* a query that also calls
`GROUPING()`/`GROUPING_ID()` on the same aggregate's column, because that
bitmask-producing function (see §7.6.5′'s `GetGroupingValue` discussion
below) is bound directly against the *original* `AVG` column reference, and
the rewrite has no mechanism to redirect that reference once `AVG` has been
replaced by two separate aggregates. This is a genuinely narrow,
correctly-scoped restriction — not a blanket "no `ROLLUP`" rule, just "no
`GROUPING()` on the same column."

**`ShouldRewrite` — is the rewrite worth it?**

A *grouped* `AVG` is rewritten only when the resulting `SUM` or `COUNT` can
share work with another aggregate in the same `GROUP BY`. An ungrouped `AVG`
is always rewritten.

Why does this matter? In a hash aggregate, every aggregate in the list keeps
its own state per group. `AVG(x)` keeps one state (a sum and a count together).
`SUM(x)` plus `COUNT(x)` keeps two states. So splitting a lone `AVG` makes a
grouped query *slower*, not faster. The split only pays off if one of the two
resulting pieces already exists. Then `COMMON_AGGREGATE` (pass #37) merges the
duplicates, and the `AVG` costs almost nothing extra.

A virtual hook on the rule base class, `ShouldRewrite`, makes this
decision. `RewriteAggregates` calls it after the matcher accepts an
expression (`aggregate_function_rewriter.cpp:406`):

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:95
	bool ShouldRewrite(const LogicalAggregate &aggr, Expression &expr) override {
		if (aggr.groups.empty()) {
			return true;
		}
		if (optimizer.OptimizerDisabled(OptimizerType::COMMON_AGGREGATE)) {
			return false;
		}
		auto &child = *expr.Cast<BoundAggregateExpression>().GetChildren()[0];
		FunctionBinder binder(optimizer.context);
		vector<unique_ptr<Expression>> sum_args;
		sum_args.push_back(child.Copy());
		auto sum = binder.BindAggregateFunction(
		    GetBuiltinAggregateFunction(optimizer.context, "sum", {child.GetReturnType()}), std::move(sum_args));
		vector<unique_ptr<Expression>> count_args;
		count_args.push_back(child.Copy());
		auto count = binder.BindAggregateFunction(
		    GetBuiltinAggregateFunction(optimizer.context, "count", {child.GetReturnType()}), std::move(count_args));

		D_ASSERT(aggr.children.size() == 1);
		auto is_not_null = [&](const Expression &input) {
			if (input.GetExpressionClass() == ExpressionClass::BOUND_CONSTANT) {
				return !input.Cast<BoundConstantExpression>().GetValue().IsNull();
			}
			return nullability.IsNotNull(*aggr.children[0], input);
		};
		const auto can_share_count =
		    !optimizer.OptimizerDisabled(OptimizerType::STATISTICS_PROPAGATION) && is_not_null(child);
		for (auto &other : aggr.expressions) {
			// Identical AVGs already share their native state.
			if (expr.Equals(*other)) {
				continue;
			}
			if (sum->Equals(*other) || count->Equals(*other)) {
				return true;
			}
			if (!can_share_count || other->IsVolatile()) {
				continue;
			}
			vector<reference<Expression>> bindings;
			if (!count_matcher->Match(*other, bindings) && !matcher->Match(*other, bindings)) {
				continue;
			}
			auto &children = other->Cast<BoundAggregateExpression>().GetChildren();
			if (children.empty() || (children.size() == 1 && is_not_null(*children[0]))) {
				return true;
			}
		}
		return false;
	}
```

In plain words, for an `AVG(x)` under a `GROUP BY`:

1. If `COMMON_AGGREGATE` is disabled, nothing can be merged, so: no rewrite.
2. If the same aggregate list already holds `SUM(x)` or `COUNT(x)` (an exact
   match, so no `FILTER`, `DISTINCT`, `ORDER BY` or `EXPORT_STATE`): rewrite.
3. If `x` is provably never `NULL`, then `COUNT(x)` equals `COUNT(*)`. So any
   `COUNT(*)`, any `COUNT(y)` with a non-`NULL` `y`, or another `AVG(y)` with a
   non-`NULL` `y` can share the count: rewrite. This step needs
   `STATISTICS_PROPAGATION` to be enabled, and it skips volatile aggregates.
4. Otherwise: keep `AVG(x)` as it is.

`NotNullExpressionAnalyzer` (`src/planner/expression_nullability.cpp`) answers
"is this never `NULL`?" from the plan below the aggregate, for example a
`WHERE x IS NOT NULL` filter or a `NOT NULL` column.

**Test walkthrough — `test/optimizer/avg_rewrite_profitability.test`.** The
table `avg_inputs(g, x, y, d)` has `NULL`s in `x`, `y` and `d`:

```sql
-- test/optimizer/avg_rewrite_profitability.test
EXPLAIN SELECT g, avg(x) AS a0 FROM avg_inputs GROUP BY g;
-- logical_opt   <REGEX>:.*avg\(x\).*      -- lone AVG: kept
EXPLAIN SELECT g, avg(x) AS a0, stddev_samp(x) FROM avg_inputs GROUP BY g;
-- logical_opt   <REGEX>:.*avg\(x\).*      -- nothing to share with: kept
EXPLAIN SELECT g, avg(x) AS a0, sum(x) FROM avg_inputs GROUP BY g;
-- logical_opt   <!REGEX>:.*avg\(x\).*     -- SUM(x) is shared: rewritten
EXPLAIN SELECT g, count(x), avg(x) AS a0 FROM avg_inputs GROUP BY g;
-- logical_opt   <!REGEX>:.*avg\(x\).*     -- COUNT(x) is shared: rewritten
EXPLAIN SELECT g, avg(x) AS a0, count(y) FROM avg_inputs GROUP BY g;
-- logical_opt   <REGEX>:.*avg\(x\).*      -- x and y have different NULLs: kept
EXPLAIN SELECT g, avg(x) AS a0, sum(x) FILTER (WHERE y=100), count(DISTINCT x) FROM avg_inputs GROUP BY g;
-- logical_opt   <REGEX>:.*avg\(x\).*      -- FILTER/DISTINCT states differ: kept
EXPLAIN SELECT g, avg(random()) AS a, count(*) FROM avg_inputs GROUP BY g;
-- logical_opt   <REGEX>:.*avg\(.*          -- volatile argument: kept

-- avg_nonnull has no NULLs at all
EXPLAIN SELECT g, avg(x) AS a0, count(*) FROM avg_nonnull GROUP BY g;
-- logical_opt   <!REGEX>:.*avg\(x\).*     -- x is never NULL, so COUNT(*) is shared
EXPLAIN SELECT g, avg(x) AS a0, avg(y) AS a1 FROM avg_nonnull GROUP BY g;
-- logical_opt   <!REGEX>:.*avg\(.*         -- two AVGs share one count
```

The same file checks the switches near its end. With
`SET disabled_optimizers='statistics_propagation'`, `avg(x)` next to `count(*)`
is kept (step 3 is off), but `avg(x)` next to `sum(x)` is still rewritten
(step 2). With `'common_aggregate'` disabled instead, even that one is kept
(step 1). With `'aggregate_function_rewriter'` disabled, the query results are
the same as with the rewrite on.

`Rewrite()` does the actual surgery: pull `AVG`'s one child out, look up the
`sum` overload for that child's type, bind a fresh `SUM(child)` in place of
the original `AVG(child)` expression, and hand back a *copy* of the child
as the value `RewriteAggregates` should compute `COUNT(...)` over:

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:145
unique_ptr<Expression> Rewrite(unique_ptr<Expression> &expr, vector<reference<Expression>> &bindings,
                               vector<unique_ptr<Expression>> &additional_expressions) override {
	FunctionBinder function_binder(optimizer.context);
	auto avg_child = std::move(bindings[0].get().Cast<BoundAggregateExpression>().GetChildrenMutable()[0]);
	auto sum_fun = GetBuiltinAggregateFunction(optimizer.context, "sum", {avg_child->GetReturnType()});
	vector<unique_ptr<Expression>> args;
	args.push_back(std::move(avg_child));
	auto count_arg = args.back()->Copy();
	expr = function_binder.BindAggregateFunction(std::move(sum_fun), std::move(args));
	return count_arg;
}
```

...and `CreateProjectionExpression` is the division that goes into the
synthesized `LogicalProjection`:

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:162
unique_ptr<Expression> CreateProjectionExpression(const LogicalType &, unique_ptr<Expression> aggr_ref,
                                                  vector<unique_ptr<Expression>> additional_expressions) override {
	// SUM(x) / COUNT(x): additional_expressions[0] is the COUNT(x) ref
	return optimizer.BindScalarFunction("/", std::move(aggr_ref), std::move(additional_expressions[0]));
}
```

**Test walkthrough — `test/optimizer/test_aggregate_function_rewrite.test`,
`### AvgRewriteRule ###` section (lines 99–139):** the test builds a small
synthetic `lineitem` table (100 rows, TPC-H-shaped columns) and runs a
TPC-H-Q1-flavored query with `PRAGMA explain_output = OPTIMIZED_ONLY`:

```sql
EXPLAIN SELECT
    l_returnflag, l_linestatus,
    sum(l_quantity) AS sum_qty,
    sum(l_extendedprice) AS sum_base_price,
    sum(l_extendedprice * (1 - l_discount)) AS sum_disc_price,
    sum(l_extendedprice * (1 - l_discount) * (1 + l_tax)) AS sum_charge,
    avg(l_quantity) AS avg_qty,
    avg(l_extendedprice) AS avg_price,
    avg(l_discount) AS avg_disc,
    count(*) AS count_order
FROM lineitem
WHERE l_shipdate <= CAST('1998-09-02' AS date)
GROUP BY l_returnflag, l_linestatus
ORDER BY l_returnflag, l_linestatus;
----
logical_opt	<!REGEX>:.*avg\([^)]*\).*
```

The assertion is a **negative** regex: the optimized plan must contain *no*
substring matching `avg(...)` at all. That's the whole test — it doesn't
check the rewritten shape's exact text, just that every one of the three
`avg(...)` calls (`avg_qty`, `avg_price`, `avg_disc`) has vanished from the
optimized tree, replaced (per the mechanics above) by `sum(...)`/`count(...)`
pairs feeding a division in the projection above the aggregate. Note this
query has a plain `GROUP BY l_returnflag, l_linestatus` — no `ROLLUP`, no
`GROUPING()` — so `ShouldSkip`'s `grouping_functions.empty()` guard is
trivially satisfied and every `avg` call rewrites.

This query also has to pass `ShouldRewrite`, because it has a `GROUP BY`. It
does. `avg(l_quantity)` and `avg(l_extendedprice)` can
share `sum(l_quantity)` and `sum(l_extendedprice)`, which the query already
computes. `avg(l_discount)` has no matching `SUM`, but `l_discount` has no
`NULL`s, so its count is the same as the query's `count(*)`. That is why the
negative regex holds.

### 7.6.5′.2 `SumRewriteRule` — `SUM(x + C)` → `SUM(x) + C * COUNT(x)`, and why it's narrower than it looks

**Purpose, in one sentence:** turn a sum of `(column + constant)` into a
sum of the bare column plus a cheap closed-form correction term, so the
addition doesn't have to be re-evaluated once per row when the constant
part of the answer can be computed once from the row count.

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:173
//! SUM(x + C) -> SUM(x) + C * COUNT(x)
class SumRewriteRule : public AggregateRewriteRule {
public:
	explicit SumRewriteRule(Optimizer &optimizer) : AggregateRewriteRule(optimizer) {
		// Match SUM(x + C) or SUM(C + x) on integer types
		auto op = make_uniq<AggregateExpressionMatcher>();
		op->function = make_uniq<SpecificFunctionMatcher>("sum");
		op->policy = SetMatcher::Policy::UNORDERED;

		auto arithmetic = make_uniq<FunctionExpressionMatcher>();
		arithmetic->function = make_uniq<SpecificFunctionMatcher>("+");
		arithmetic->type = make_uniq<IntegerTypeMatcher>();
		auto child_constant_matcher = make_uniq<ConstantExpressionMatcher>();
		auto child_expression_matcher = make_uniq<StableExpressionMatcher>();
		child_constant_matcher->type = GetSmallIntegerTypesMatcher();
		child_expression_matcher->type = GetSmallIntegerTypesMatcher();
		arithmetic->matchers.push_back(std::move(child_constant_matcher));
		arithmetic->matchers.push_back(std::move(child_expression_matcher));
		arithmetic->policy = SetMatcher::Policy::SOME;
		op->matchers.push_back(std::move(arithmetic));

		matcher = std::move(op);
	}
```

**The type restriction, exactly, in two independent layers.** This is
worth being precise about, because the two layers do different jobs and
together they're stricter than either one alone:

*Layer 1 — the addition's own return type.* `arithmetic->type =
IntegerTypeMatcher()`, and `IntegerTypeMatcher::Match` is exactly
`type.IsIntegral()` (`type_matcher.hpp:65-70`), which resolves to:

```cpp
// src/common/types.cpp:720
bool LogicalType::IsIntegral() const {
	switch (id()) {
	case LogicalTypeId::TINYINT:
	case LogicalTypeId::SMALLINT:
	case LogicalTypeId::INTEGER:
	case LogicalTypeId::BIGINT:
	case LogicalTypeId::UTINYINT:
	case LogicalTypeId::USMALLINT:
	case LogicalTypeId::UINTEGER:
	case LogicalTypeId::UBIGINT:
	case LogicalTypeId::HUGEINT:
	case LogicalTypeId::UHUGEINT:
		return true;
	default:
		return false;
	}
}
```

This already excludes every floating-point type (`FLOAT`, `DOUBLE`) and
`DECIMAL` outright — `DOUBLE + DOUBLE` never even reaches the second layer,
because the whole `+` expression fails `IsIntegral()` at the top-level
match before either operand's type is inspected. This is the entire
enforcement mechanism for "float addition is not associative, don't do
this rewrite for floats" — there is no separate float-specific check
anywhere in this file; the general `IsIntegral()` filter is doing that job
implicitly, by construction.

*Layer 2 — each individual operand's type.* Both matcher branches under
`arithmetic` — `child_constant_matcher` (the `C`) and `child_expression_matcher`
(the `x`) — are independently restricted to `GetSmallIntegerTypesMatcher()`:

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:63
static unique_ptr<SetTypesMatcher> GetSmallIntegerTypesMatcher() {
	vector<LogicalType> types {LogicalTypeId::TINYINT,  LogicalTypeId::SMALLINT, LogicalTypeId::INTEGER,
	                           LogicalTypeId::BIGINT,   LogicalTypeId::UTINYINT, LogicalTypeId::USMALLINT,
	                           LogicalTypeId::UINTEGER, LogicalTypeId::UBIGINT};
	return make_uniq<SetTypesMatcher>(std::move(types));
}
```

Notice what's *missing* from this list relative to Layer 1's set:
**`HUGEINT` and `UHUGEINT`.** `IsIntegral()` accepts them; this narrower,
per-operand list does not. Because the match requires *both* the whole
addition and each of its two operands to succeed, and DuckDB's `+` between
two operands of the same width doesn't silently widen the result to a
bigger type, the practical effect is that `SumRewriteRule` can only ever
fire on additions where every type involved — the sum, the column, and the
constant — is one of exactly these eight ≤64-bit exact integer types. A
`SUM(a + C)` where `a` is `HUGEINT` structurally cannot match this rule, no
matter what `C` is, because `child_expression_matcher`'s type set stops at
`UBIGINT`.

This is not an oversight — it's a second, independent correctness boundary,
and it's exactly what the `test_aggregate_function_rewrite.test` file's
`HUGEINT` case (walked through below) is checking for: `HUGEINT` addition
is exact (no rounding, unlike floats), but it's also the *widest* integer
type DuckDB has, meaning a `HUGEINT` sum has essentially no headroom left
before 128-bit wraparound — summing the column directly and summing
`(column + constant)` directly can hit that wraparound boundary at
different points in the computation, so the two forms are not guaranteed
to produce the same (wrapped) answer even though ordinary, non-overflowing
integer addition genuinely is associative. Restricting the rewrite to
types at least one width narrower than the accumulator DuckDB actually
sums into (recall from `extension/core_functions/aggregate/distributive/sum.cpp`
that even a `BIGINT` `SUM` accumulates into a `hugeint_t` internal state,
`SumToHugeintOperation`, `sum.cpp:198-204`) keeps the accumulator's headroom
large relative to any single row's contribution, which is precisely why
`(U)BIGINT` and narrower are safe here and `HUGEINT`/`UHUGEINT` are not.

**`ShouldSkip` — the restriction that has nothing to do with types at
all.** This is the detail easiest to miss on a first read, and it's the
main thing this deep dive adds on top of §7.6.5's earlier, shorter
treatment:

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:198
bool ShouldSkip(const LogicalAggregate &aggr) const override {
	// This can probably be relaxed to be the same check as AvgRewriteRule,
	// but behaviour (for now) is the same as the old SumRewriterOptimizer
	return !aggr.groups.empty();
}
```

`SumRewriteRule` skips **every `LogicalAggregate` that has any `GROUP BY`
key at all** — `aggr.groups.empty()` is only true for a bare, global
aggregate with no `GROUP BY` clause whatsoever. Contrast this directly with
`AvgRewriteRule`'s `ShouldSkip`, which only excludes queries using
`GROUPING()` and is otherwise happy to fire under any grouping, including
`ROLLUP`/`CUBE`. The comment says outright that this is a historical
carry-over from "the old `SumRewriterOptimizer`" (a predecessor,
single-purpose pass this functionality was folded into) rather than a
fundamental limitation — the rewrite's *correctness* argument (integer
addition is associative and NULL-handling is identical between the two
forms) does not actually depend on whether a `GROUP BY` is present. But as
checked into this exact commit, **`SUM(i + C)` only gets algebraically
rewritten in a query with no `GROUP BY` clause at all**; the identical
expression under a `GROUP BY` is left to evaluate `i + C` per row, computed
correctly, just not via the cheaper closed form.

**Why this restriction can't silently produce wrong answers even though
it's overly conservative:** the naive, un-rewritten evaluation path (add
the constant to each row, then sum) is always available and always
correct — `ShouldSkip` returning early just means a query with `GROUP BY`
takes the slower of two equally-correct paths, not that anything breaks.
The test file confirms this directly (see the `GROUP BY grp` case below):
correctness is unaffected, only the optimization opportunity is missed.

**The float-associativity argument, made concrete.** The class comment and
DiDi's course material [10] both assert that this rewrite is unsafe for
floating-point types "since IEEE-754 float addition isn't associative."
The type-matcher analysis above already shows *how* DuckDB enforces this
(floats fail `IsIntegral()` before the rule can even try); it's worth
seeing *why* the restriction is necessary at all, with a real, reproducible
example — DuckDB's own type-matchers never let this run on `DOUBLE`, so
this is a constructed counterfactual (what *would* go wrong if the
`IntegerTypeMatcher` guard were removed), verified against real IEEE-754
double arithmetic (Python's `float` is an IEEE-754 double, so this is not
hand-waved arithmetic):

```
rows:  x = [1e16, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0]     (9 rows)
C    = 1.0
n    = 9    (row count)
```

Direct evaluation, `SUM(x + C)`, accumulated row by row (mirroring how a
vectorized hash-aggregate state actually folds in one value at a time):

```
row 0: x=1e+16      x+C=1e+16      running_total=1e+16              (1e16 + 1.0 rounds back down to 1e16 — the
                                                                       ULP at this magnitude is 2, so +1 is an exact
                                                                       tie, and round-to-even keeps the even value)
row 1: x=1.0        x+C=2.0        running_total=1.0000000000000002e+16
row 2: x=1.0        x+C=2.0        running_total=1.0000000000000004e+16
row 3: x=1.0        x+C=2.0        running_total=1.0000000000000006e+16
row 4: x=1.0        x+C=2.0        running_total=1.0000000000000008e+16
row 5: x=1.0        x+C=2.0        running_total=1.000000000000001e+16
row 6: x=1.0        x+C=2.0        running_total=1.0000000000000012e+16
row 7: x=1.0        x+C=2.0        running_total=1.0000000000000014e+16
row 8: x=1.0        x+C=2.0        running_total=1.0000000000000016e+16
```

`direct = SUM(x + C) = 1.0000000000000016e+16`  (i.e. `1e16 + 16`)

Rewritten evaluation, `SUM(x) + n*C`, computed by summing the bare column
first and adding the correction term once at the very end:

```
row 0: x=1e+16      running_sum=1e+16
row 1: x=1.0        running_sum=1e+16   (adding 1.0 to 1e16 is again an exact tie at ULP=2;
row 2: x=1.0        running_sum=1e+16    round-to-even keeps snapping back to the same even value,
row 3: x=1.0        running_sum=1e+16    every single time — all eight +1.0 increments are individually
row 4: x=1.0        running_sum=1e+16    "lost" to rounding because none of them, alone, is large enough
row 5: x=1.0        running_sum=1e+16    to clear half an ULP)
row 6: x=1.0        running_sum=1e+16
row 7: x=1.0        running_sum=1e+16
row 8: x=1.0        running_sum=1e+16
```

`SUM(x) = 1e+16`, then `+ n*C = 1e16 + 9*1.0 = 1e16 + 9`, which again must
round to the nearest representable value at ULP=2 — another exact tie,
resolved to `1e16 + 8`:

`rewritten = SUM(x) + n*C = 1.0000000000000008e+16`  (i.e. `1e16 + 8`)

```
direct    - rewritten = 8.0     (four ULPs apart at this magnitude)
```

The two expressions, mathematically identical over the reals, land on
**two different IEEE-754 double values** — not because either evaluation
order is "wrong" (neither is; the true, infinite-precision answer, `1e16 +
17`, isn't itself representable as a double at this magnitude anyway, so
both computed answers are *approximations*, just different ones), but
because floating-point addition rounds at every step, and *which* additions
happen to land on a rounding boundary depends entirely on the order and
grouping in which they're performed. `SumRewriteRule`'s rewrite changes
that order — it moves every row's `+ C` out of the per-row loop and
replaces it with one multiplication done after the fact — and for
floating-point inputs that reordering is observable in the result's low
bits. For exact integer arithmetic (no rounding at any step, as long as no
operation overflows) there is no such boundary to land on differently:
`(x_1 + C) + (x_2 + C) + ... + (x_n + C)` and `(x_1 + x_2 + ... + x_n) + n
\cdot C` are *the same integer*, computed two different ways, which is
exactly why `IntegerTypeMatcher` is both necessary and sufficient here (modulo
the separate `HUGEINT`/overflow boundary discussed above).

**Test walkthrough — `test/optimizer/test_aggregate_function_rewrite.test`,
`### SumRewriteRule ###` section (lines 5–97):**

```sql
CREATE TABLE integers AS FROM range(10000) t(i);

query IIII
SELECT SUM(i + 1), SUM(i + 2), SUM(i + 3), SUM(i + 4) FROM integers
----
50005000	50015000	50025000	50035000
```

Four independent `SUM(column + constant)` calls in one bare (no `GROUP
BY`) query — `ShouldSkip` passes (`aggr.groups.empty()`), every one of the
four matches `SumRewriteRule`'s matcher (`i` is `BIGINT`-typed from
`range()`, well within `GetSmallIntegerTypesMatcher()`'s set), and each
gets rewritten independently into its own `SUM(i) + C*COUNT(i)` pair in the
synthesized projection. `SUM(i)` for `i` in `[0, 10000)` is `49995000`;
`COUNT(i)` is `10000`; so `SUM(i+1) = 49995000 + 1*10000 = 50005000`,
matching row 1 of the expected output, and the same arithmetic for `+2`,
`+3`, `+4` produces the remaining three columns.

```sql
query IIII
SELECT SUM(i + 1), SUM(2 + i), SUM(i + 3), SUM(4 + i) FROM integers
----
50005000	50015000	50025000	50035000
```

Identical answer with the constant on the *left* for two of the four calls
(`SUM(2 + i)`, `SUM(4 + i)`) — this is exactly what `SetMatcher::Policy::SOME`
on the `arithmetic` matcher's two children buys: `SOME` (as opposed to
`ORDERED`) matches its sub-matchers against the operands in *either* order
(`set_matcher.hpp:100-113` — `SOME` falls through to the same
try-every-unmatched-entry recursive search `UNORDERED` uses, it just also
tolerates more matchers than a strict count match would), so `C + x` and
`x + C` both satisfy `child_constant_matcher`/`child_expression_matcher`
regardless of which operand is which.

```sql
query IIIII
SELECT i//100000 grp, SUM(i + 1), SUM(2 + i), SUM(i + 3), SUM(4 + i) FROM integers GROUP BY grp
----
0	50005000	50015000	50025000	50035000
```

Same query, same expected numeric answer — but now with an explicit `GROUP
BY grp` (there's exactly one group here, since `i // 100000` is `0` for
every row in `range(10000)`). Per `ShouldSkip`'s `!aggr.groups.empty()`
check above, `SumRewriteRule` does **not** fire on this `LogicalAggregate`
at all — every `SUM(i + C)` is evaluated the naive way, adding the constant
to `i` once per row before summing. The test is not exercising the rewrite
here; it's a correctness check that the un-rewritten path (which is all
that runs once a `GROUP BY` is present) still produces the right answer,
serving as the practical demonstration that the `ShouldSkip` restriction
costs performance, not correctness.

```sql
# empty result
query IIII
SELECT SUM(i + 1), SUM(2 + i), SUM(i + 3), SUM(4 + i) FROM integers WHERE i > 100000
----
NULL	NULL	NULL	NULL
```

`i > 100000` matches zero rows out of `range(10000)`. A bare (ungrouped)
`SUM` over zero input rows is `NULL` by SQL convention — and critically,
the rewritten form has to preserve that. `SUM(i) + C * COUNT(i)` over zero
rows: `SUM(i)` is `NULL`, `COUNT(i)` is `0`, so `C * 0 = 0` (or a typed
zero), and `NULL + 0` is still `NULL` under ordinary SQL arithmetic NULL
propagation — the rewrite's arithmetic naturally preserves the "no rows →
NULL" behavior without any special-casing, because `+` already propagates
`NULL`.

```sql
# nulls
CREATE TABLE integer_nulls AS SELECT CASE WHEN i%2=0 THEN NULL ELSE i END i FROM range(10000) t(i);

query IIII
SELECT SUM(i + 1), SUM(i + 2), SUM(i + 3), SUM(i + 4) FROM integer_nulls;
----
25005000	25010000	25015000	25020000
```

Half the rows are `NULL`. `SUM` (both forms) ignores `NULL` inputs
entirely rather than propagating them per-row, and `COUNT(x)` (as opposed
to `COUNT(*)`) also only counts non-`NULL` values of `x` — so `COUNT(i)`
here is `5000` (the odd `i`s), not `10000`, and the rewritten
`SUM(i) + C*COUNT(i)` uses the *same* non-`NULL` row count the direct
per-row evaluation implicitly would have. This is exactly why
`RewriteAggregates` (§7.6.5′'s shared-mechanics section above) adds
`COUNT(x)` — the *unary* count over the rewritten argument — rather than
`COUNT(*)`, whenever the rule hands back a non-null count argument;
`SumRewriteRule::Rewrite` always does (it returns the extracted `main_expr`
directly, `aggregate_function_rewriter.cpp:216`, never `nullptr`), so this
case is handled correctly by construction, not by a special check.

```sql
statement ok
pragma explain_output='optimized_only'

query II
EXPLAIN SELECT SUM(i + 1), SUM(i + 2), SUM(i + 3), SUM(i + 4) FROM integers
----
logical_opt	<REGEX>:.*sum_no_overflow\(i\).*
```

**This is the assertion worth being careful about**, because at first
glance it looks like it's checking `SumRewriteRule`'s own output text, and
it isn't — or rather, it's checking the *combined* effect of two
completely different, independently-triggered optimizations. `SumRewriteRule`
does fire here (no `GROUP BY`, `BIGINT` column, small integer constant),
turning `SUM(i + 1)` into a bare `SUM(i)` living inside the
`LogicalAggregate`, plus a `1 * COUNT(i)` term in the projection above it —
exactly the mechanism walked through above. But the *name*
`sum_no_overflow` never appears anywhere in `aggregate_function_rewriter.cpp`.
It's a separate aggregate-function implementation, registered in
`extension/core_functions/aggregate/distributive/sum.cpp`, that a much
later pass — `STATISTICS_PROPAGATION` (§7.4's pass #34,
`src/optimizer/statistics_propagator.cpp`) — swaps in for a plain `sum(...)`
call once it can *prove*, from propagated column statistics and the known
row-count bound, that the sum cannot exceed what fits in a 64-bit
accumulator:

```cpp
// extension/core_functions/aggregate/distributive/sum.cpp:386
// Replace with fast implementation when possible: only INT32/INT64 have a narrower no-overflow implementation to
// swap in.
const bool has_no_overflow_variant = internal_type == PhysicalType::INT32 || internal_type == PhysicalType::INT64;
const bool sum_fits_in_int64 =
    wide_negative > NumericLimits<int64_t>::Minimum() && wide_positive < NumericLimits<int64_t>::Maximum();
if (has_no_overflow_variant && sum_fits_in_int64) {
	expr.FunctionMutable().ReplaceImplementation(GetSumAggregateNoOverflow(internal_type));
}
```

This callback (`SumPropagateStats`, registered as `sum`'s statistics
callback) is a pure **physical-implementation swap for speed** — DuckDB's
default `sum` implementation for an `INT32`/`INT64` column accumulates into
a 128-bit (`hugeint_t`) state so a long-running sum can never silently
overflow (`SumToHugeintOperation`, `sum.cpp:198-204`); once the optimizer
has proven from statistics that the sum is bounded well inside 64-bit
range, it substitutes a narrower `int64_t`-accumulating implementation
(`IntegerSumOperation`/`SumState<int64_t>`, `sum.cpp:181-196`) that's
cheaper to update per row, purely because it skips the wider arithmetic —
this has nothing to do with `SumRewriteRule`'s algebraic identity. The test
comment even says as much a few lines later: *"the rewrite needs the
cardinality of the CTE definition, so it only fires when the sum provably
fits in an int64"* — confirming this is a statistics-driven decision, not
the matcher-driven `SumRewriteRule` decision. Two `EXPLAIN` checks make the
distinction concrete:

```sql
query II
EXPLAIN WITH d AS MATERIALIZED (SELECT i FROM integers) SELECT SUM(i) FROM d
----
logical_opt	<REGEX>:.*sum_no_overflow\(i\).*

query II
EXPLAIN WITH d AS MATERIALIZED (SELECT i FROM integers) SELECT COUNT(i) FROM d
----
logical_opt	<REGEX>:.*count_star\(\).*
```

The first query is a **bare `SUM(i)`** — no `+ C`, nothing for
`SumRewriteRule`'s matcher to match at all — and `sum_no_overflow(i)` still
shows up in the optimized plan, which on its own proves the two mechanisms
are independent: `sum_no_overflow` appears whether or not `SumRewriteRule`
fired. (The second query, `COUNT(i)` over a column statistically known to
be non-`NULL`, is folded to `count_star()` by a *different* rewrite inside
`STATISTICS_PROPAGATION`/its statistics-driven simplifications — not part
of this file either, included here only because it sits in the same test
block.) Two boundary tests confirm the row-count/statistics dependency
directly:

```sql
CREATE TABLE small_vals(v BIGINT);
INSERT INTO small_vals SELECT (i * 1000000000)::BIGINT FROM range(1000) s(i);

query II
EXPLAIN WITH c AS MATERIALIZED (SELECT v FROM small_vals) SELECT sum(v) FROM c
----
logical_opt	<REGEX>:.*sum_no_overflow\(v\).*

CREATE TABLE big_vals(v BIGINT);
INSERT INTO big_vals SELECT 4000000000000000000::BIGINT FROM range(5) s(i);

query II
EXPLAIN WITH c AS MATERIALIZED (SELECT v FROM big_vals) SELECT sum(v) FROM c
----
logical_opt	<!REGEX>:.*sum_no_overflow.*
```

`small_vals` has 1000 rows each up to `999 × 10^9 ≈ 10^12`; the maximum
possible sum (`~10^15`) sits comfortably inside `int64_t`'s range
(`~9.2 × 10^18`), so the statistics-driven swap fires. `big_vals` has 5
rows each at `4 × 10^18`; summing even two of them (`8 × 10^18`) already
exceeds `int64_t::Maximum()` (`~9.223 × 10^18`) — `sum_fits_in_int64` is
false, and the plan correctly keeps the safe, wide `hugeint_t`-accumulating
implementation (the negative-regex assertion on the second query is
checking for the *absence* of any `sum_no_overflow` substring at all).

```sql
# test rewrite with HUGEINT
CREATE TABLE T(a HUGEINT);
INSERT INTO T VALUES ('-40282366920938463463374607431768211455'::HUGEINT) /* × 6 */;

query I
SELECT sum(a + '40282366920938463463374607431768211455'::HUGEINT) FROM T;
----
0
```

This is the negative case for `SumRewriteRule`'s own type restriction
(distinct from the `sum_no_overflow` mechanism above). Despite the block
comment ("test rewrite with HUGEINT"), this query does **not** actually
match `SumRewriteRule`: `a` and the constant are both `HUGEINT`, and
`GetSmallIntegerTypesMatcher()` — the per-operand type set inside the `+`
matcher — stops at `UBIGINT`, so `child_expression_matcher` (which requires
`x`'s type to be in that narrower set) never matches `a`. What this case
actually verifies is that `SUM` over a column whose values are already
close to `HUGEINT`'s boundary is computed the *direct* way — evaluating
`a + '40282366920938463463374607431768211455'::HUGEINT` per row (each row's
result wraps or cancels according to `HUGEINT`'s 128-bit modular
arithmetic) and summing those six results — and that this direct
evaluation is exactly the path DuckDB takes, since the rewrite structurally
cannot apply. Had the rule's per-operand type set been loosened to include
`HUGEINT` (contradicting the deliberate restriction explained above), the
rewritten form `SUM(a) + 6 * C` would sum six copies of a near-boundary
`HUGEINT` value directly — a computation with far less remaining headroom
before its own 128-bit wraparound than the direct per-row form has, and one
which is not guaranteed to hit the *same* wraparound boundary at the *same*
point as the direct evaluation. This is the `HUGEINT` analogue of the
floating-point-associativity argument above: not rounding error, but a
different, difference-preserving reordering of *exactly which additions
saturate a finite-width accumulator's range*, which can only be relied on
to agree with the direct evaluation for types with meaningfully more
headroom than the values actually being summed — exactly the boundary
`GetSmallIntegerTypesMatcher()` draws.

### An aside: `test/optimizer/misc/test_count_and_sum.test` and the same-label idiom

**The idiom, since no sibling part in this directory has introduced it
yet:** a DuckDB `.test` file can give two separate `query <cols> nosort
<label>` blocks the *same* label. The test harness then requires those two
blocks' actual output to be **byte-identical**, in addition to whatever
each block's own `----`-delimited expected-rows section separately asserts.
This is a compact way to encode "these two SQL expressions must always
agree with each other" without hand-writing the same set of expected rows
twice: write the first query, its full expected output; write the second
query under the identical label, and leave its own expected-output section
empty (or partial) — the harness fills in the comparison from the first
block's actual result and fails the test if the second block's actual
output differs by even one byte. It's a parity check, not an
absolute-correctness check: the test doesn't assert *what* the shared
answer has to be, only that both expressions land on the same one.

`test_count_and_sum.test` uses exactly this idiom, and it's a useful
example precisely because it does *not* live inside `aggregate_function_rewriter.cpp`
at all — it's grouped under `test/optimizer/misc/`, described as an
"Arithmetic simplification test," and its two queries are:

```sql
CREATE TABLE t0 (c0 TEXT);
CREATE TABLE t1 (c1 TEXT);
CREATE VIEW v0 AS SELECT t0.c0 FROM t1 LEFT JOIN t0 ON t1.c1 = t0.c0;
INSERT INTO t1(c1) VALUES ('example_value');

query I nosort result_1
SELECT COUNT(*)
FROM v0
WHERE (CURRENT_TIMESTAMP, c0) != (CAST(NULL AS TEXT), '0');
----


query I nosort result_1
SELECT SUM(CASE WHEN (CURRENT_TIMESTAMP, c0) != (CAST(NULL AS TEXT), '0') THEN 1 ELSE 0 END)
FROM v0;
----
```

The two queries express the same intent two different ways —
`COUNT(*) WHERE <predicate>` versus `SUM(CASE WHEN <predicate> THEN 1 ELSE
0 END)`, a textbook "conditional count" rewrite a human or an ORM might
apply by hand — over a one-row view (`t1` has one row; `t0` is empty, so
the `LEFT JOIN` produces exactly one row with `c0 = NULL`), filtered by a
row-value (tuple) comparison that mixes a volatile-looking but
statement-stable `CURRENT_TIMESTAMP` against a `NULL`-typed constant in one
tuple position and the (here, `NULL`) column `c0` in the other. As checked
into this exact commit, **both `query I nosort result_1` blocks' expected
sections are entirely empty** — confirmed directly from the raw file bytes
(no expected value line, not even a blank placeholder, between each `----`
and the next directive) — meaning both forms are expected to produce the
same, zero-row result.

This is worth flagging honestly rather than asserting a root cause this
investigation couldn't fully verify against this exact commit: a bare
aggregate with no `GROUP BY` (both queries here have none) ordinarily
always emits exactly one output row under standard SQL, even when zero
input rows satisfy the aggregate's condition — `COUNT(*) FROM t WHERE
false` returns one row with value `0`, not zero rows. Checking DuckDB's row
tuple-comparison semantics against a separately available, older release
build (`v1.1.3`, *not* commit `cb5673685153` — a different codebase
snapshot, used here only to sanity-check general SQL-level tuple-comparison
behavior, not this commit's optimizer internals) shows `(CURRENT_TIMESTAMP,
c0) != (CAST(NULL AS TEXT), '0')` evaluating to a definite `true` for this
row (and the mirrored `=` form to a definite `false`) rather than the
`NULL` a naive per-field three-valued-logic reading of the tuple comparison
might predict — i.e. DuckDB's row/`STRUCT` equality does not appear to
propagate `NULL` the way per-field scalar equality would. Given that, the
literal, standard-SQL expectation for a bare, ungrouped `COUNT(*)`/`SUM(CASE...)`
pair here would be a single row (with `COUNT(*) = 1`, matching the older
release's observed behavior), not the wholly empty result this exact
commit's test file expects. Whether this checkout genuinely differs from
that older release's behavior for this specific view/tuple-comparison
combination, or whether something about `v0`'s `LEFT JOIN` construction
interacts with an optimizer pass in a way this deep dive didn't trace all
the way through, was not something this investigation could settle without
building this exact commit — so, per this book's stated policy, that
specific "why zero rows" question is left flagged rather than guessed at.
What the test file *does* prove, unambiguously, straight from its own
bytes, is the mechanical point the same-label idiom exists to make: **both
phrasings of the conditional-count idea produce identical output on this
commit, whatever that output turns out to be** — which is the entire,
narrower claim the test is actually designed to check.

### 7.6.5′.3 `ListRewriteRule` — `list(x ORDER BY x)` → `list_sort(list(x), ...)`

**Purpose, in one sentence:** when `list()`'s `ORDER BY` clause sorts by
the exact same expression it's collecting (`list(x ORDER BY x)`, not
`list(x ORDER BY y)`), skip the (more expensive, per-group) ordered
aggregation path entirely and instead collect the values unordered, then
sort the resulting list once with a plain scalar function.

This rule's matcher is structured differently from the other two — it
needs to match not just the aggregate's arguments but its `ORDER BY`
clause too, and needs the `ORDER BY` key to be *the same expression* as the
thing being collected:

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:231
//! list(expr ORDER BY expr <sense> <nulls>) => list_sort(list(expr), <sense>, <nulls>)
class BindingMatcher : public ExpressionMatcher {
public:
	explicit BindingMatcher(idx_t binding_idx) : ExpressionMatcher(ExpressionClass::INVALID), binding_idx(binding_idx) {
	}
	bool Match(Expression &expr, vector<reference<Expression>> &bindings) override {
		if (!expr.Equals(bindings[binding_idx])) {
			return false;
		}
		bindings.push_back(expr);
		return true;
	}
	const idx_t binding_idx;
};
```

`BindingMatcher` doesn't match on expression *shape* at all — it matches
only if the candidate expression is `Equals()` to whatever was already
bound at `binding_idx` (index `1`, i.e. the aggregate's already-matched
sole child argument). Wired up in `ListRewriteRule`'s constructor:

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:295
explicit ListRewriteRule(Optimizer &optimizer) : AggregateRewriteRule(optimizer) {
	auto op = make_uniq<OrderedAggregateMatcher>();
	op->function = make_uniq<SpecificFunctionMatcher>("list");
	op->policy = SetMatcher::Policy::ORDERED;
	op->matchers.push_back(make_uniq<StableExpressionMatcher>());
	op->order_bys.emplace_back(make_uniq<BindingMatcher>(idx_t(1)));
	matcher = std::move(op);
}
```

So `list(x ORDER BY x)` matches (the `ORDER BY` key equals the collected
argument), but `list(x ORDER BY y)` does not (`y` fails `BindingMatcher`'s
equality check against `x`) — the latter genuinely needs its values
collected in `y`'s order, which the rewrite's unordered-collect-then-sort
strategy can't preserve if `y` isn't derivable from the collected value
itself.

The rewrite also explicitly refuses to touch an aggregate that has already
been asked to export its raw accumulator state (`EXPORT_STATE`), and this
guard lives in `OrderedAggregateMatcher::Match` rather than `ListRewriteRule`
itself — the underlying matcher class this rule shares with unrelated
ordered-aggregate handling:

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:253
bool Match(Expression &expr_p, vector<reference<Expression>> &bindings) override {
	if (!ExpressionMatcher::Match(expr_p, bindings)) {
		return false;
	}
	auto &expr = expr_p.Cast<BoundAggregateExpression>();
	// don't rewrite state-export aggregates - list(x ORDER BY x) EXPORT_STATE would become
	// list_sort(list(x) EXPORT_STATE, ...), which cannot bind list_sort on the AGGREGATE_STATE result
	if (expr.StateExportMode() == AggregateStateExportMode::STATE_EXPORT) {
		return false;
	}
	// ...
```

The comment states the failure mode directly: `EXPORT_STATE` mode makes
the aggregate return its raw internal accumulator bytes (an
`AGGREGATE_STATE` value) rather than a finished `LIST`, and `list_sort` has
no overload that can operate on that opaque state type — attempting the
rewrite anyway would produce a plan that fails to bind.

`Rewrite()` moves the aggregate's `ORDER BY` clause out entirely (turning
`list(x ORDER BY x <sense> <nulls>)` into a plain, unordered `list(x)`) and
hands the sort direction and null-ordering back as two extra constant
string expressions:

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:318
unique_ptr<Expression> ListRewriteRule::Rewrite(unique_ptr<Expression> &expr, vector<reference<Expression>> &bindings,
                                                vector<unique_ptr<Expression>> &additional_expressions) {
	auto &aggr = bindings[0].get().Cast<BoundAggregateExpression>();
	auto &order_bys = aggr.GetOrderBysMutable();
	auto &order_by = order_bys->orders[0];
	auto sense = make_uniq<BoundConstantExpression>(EnumUtil::ToChars(order_by.type));
	auto nulls = make_uniq<BoundConstantExpression>(EnumUtil::ToChars(order_by.null_order));
	additional_expressions.emplace_back(std::move(sense));
	additional_expressions.emplace_back(std::move(nulls));
	order_bys.reset();
	return nullptr;
}
```

...and the projection above the aggregate calls `list_sort` on the
now-unordered list, passing those two constants through:

```cpp
// src/optimizer/aggregate_function_rewriter.cpp:312
unique_ptr<Expression> CreateProjectionExpression(const LogicalType &, unique_ptr<Expression> aggr_ref,
                                                  vector<unique_ptr<Expression>> addnl) override {
	return optimizer.BindScalarFunction("list_sort", std::move(aggr_ref), std::move(addnl[0]), std::move(addnl[1]));
}
```

Returning `nullptr` from `Rewrite()` (rather than a count argument) means
`RewriteAggregates`'s generic machinery (§7.6.5′'s shared-mechanics section)
adds a `COUNT(*)` — not a unary `COUNT(x)` — alongside the plain `list(x)`
in this case; that helper `COUNT` is unused by `ListRewriteRule`'s own
`CreateProjectionExpression` (which only reads `addnl[0]`/`addnl[1]`, the
sense/nulls constants), a small, harmless bit of dead weight the shared
machinery adds unconditionally rather than something specific to this
rule.

**Test evidence:** this rewrite is confirmed directly by the export-state
test suite's own inline comment (a different `.test` file from the two
this deep dive was specifically pointed at, included here because it's the
clearest first-hand confirmation of this exact rewrite's name and trigger
condition):

```sql
-- test/sql/aggregate/aggregate_state_export/ordered_aggregate_export_state.test:34
# ORDER BY the same column as the argument (this path is rewritten to list_sort(list(...)) for non-export aggregates,
# which must be skipped for EXPORT_STATE)
SELECT finalize(list(i ORDER BY i DESC) EXPORT_STATE) FROM (VALUES (1), (3), (2)) t(i);
```

This single comment independently confirms every piece of the source
analysis above: the rewrite target (`list_sort(list(...))`), the trigger
condition (`ORDER BY` the same column being collected), and the
`EXPORT_STATE` exception (`OrderedAggregateMatcher::Match`'s
`StateExportMode() == STATE_EXPORT` guard) — from a file that never even
mentions `aggregate_function_rewriter.cpp` by name.

### Summary — all three rules side by side

| Rule | Rewrite | Type restriction | `ShouldSkip` condition |
|---|---|---|---|
| `AvgRewriteRule` | `AVG(x)` → `SUM(x) / COUNT(x)` | any numeric type (`IsNumeric()`) | only skips if `GROUPING()`/`GROUPING_ID()` is called on the same expression (`!aggr.grouping_functions.empty()`). Under a `GROUP BY`, `ShouldRewrite` also requires a `SUM`/`COUNT` it can share |
| `SumRewriteRule` | `SUM(x + C)` → `SUM(x) + C·COUNT(x)` | addition's own type must be integral (`IsIntegral()` — excludes floats/`DECIMAL`); **both** operands individually restricted to `{TINYINT..UBIGINT, U-variants}` (excludes `HUGEINT`/`UHUGEINT`, which pass `IsIntegral()` but not this narrower set) | skips whenever `GROUP BY` has any key at all (`!aggr.groups.empty()`) — only fires on a bare, ungrouped aggregate |
| `ListRewriteRule` | `list(x ORDER BY x)` → `list_sort(list(x), sense, nulls)` | none (type-agnostic; matches on `ORDER BY` key equaling the collected argument) | never skips on its own; the *matcher* itself refuses `EXPORT_STATE`-mode aggregates |

## 7.6.6′ `GROUPING_SETS` — cascade, not expansion

### The question this section resolves precisely

`GROUPING SETS (...)`, `ROLLUP(...)`, and `CUBE(...)` all describe a single
`GROUP BY` clause that computes **more than one grouping level** in one
query — `ROLLUP(region, product)` means "give me `GROUP BY region, product`,
*and* `GROUP BY region`, *and* the grand total (`GROUP BY ()`)," unioned
together. The natural question — and the one this deep dive was
specifically asked to resolve carefully rather than assume an answer to —
is whether the `GROUPING_SETS` *optimizer pass* (`src/optimizer/grouping_sets_optimizer.cpp`,
`OptimizerType::GROUPING_SETS`, pass #11 in §7.4's list) is the mechanism
that turns that syntax into something the physical execution engine can
actually run, or whether it's a later, optional simplification over a plan
shape some earlier stage already made executable.

**The answer, read directly from the source: neither of those framings is
quite right, and the precise answer matters.** `ROLLUP`/`CUBE`/explicit
`GROUPING SETS` syntax is already fully expanded into DuckDB's internal
multi-grouping-set representation — a `vector<GroupingSet>` on
`LogicalAggregate` — **during binding/planning, before any optimizer pass
of any kind runs.** And DuckDB's physical hash-aggregate operator can
already **execute** that multi-grouping-set representation directly, with
no further logical-level rewriting required at all. The `GROUPING_SETS`
optimizer pass, in the overwhelmingly common case, is a **pure, optional,
disableable performance optimization** — it replaces one full-cardinality
scan-and-hash-aggregate that computes every grouping level from raw rows
with a **cascade**: compute the most detailed level once, then derive every
coarser level by re-aggregating the *already-aggregated* output of a finer
level, which is cheaper when the finer levels have already reduced the row
count substantially. Turning this optimization off does not change the
query's result and does not make the query fail to plan — it only removes
the cascade, leaving the physical operator to compute every grouping level
directly against the base data, which is exactly what
`test/optimizer/grouping_sets_optimizer.test`'s very first two assertions
demonstrate (walked through below).

### Where the multi-grouping-set representation actually comes from

`LogicalAggregate` carries the representation directly, alongside the
ordinary single-level `groups` list:

```cpp
// src/include/duckdb/planner/operator/logical_aggregate.hpp:32-42
//! The table index for the groups of the LogicalAggregate
TableIndex group_index;
//! The table index for the aggregates of the LogicalAggregate
TableIndex aggregate_index;
//! The table index for the GROUPING function calls of the LogicalAggregate
TableIndex groupings_index;
//! The set of groups (optional).
vector<unique_ptr<Expression>> groups;
//! The set of grouping sets (optional).
vector<GroupingSet> grouping_sets;
//! The list of grouping function calls (optional)
vector<unsafe_vector<ProjectionIndex>> grouping_functions;
```

`GroupingSet` itself is simply `set<ProjectionIndex>`
(`src/include/duckdb/parser/group_by_node.hpp:18`) — an ordered set of
indices into the `groups` list. `ROLLUP(region, product)`, once bound,
becomes `groups = [region, product]` and `grouping_sets =
[{0,1}, {0}, {}]` — the finest set (both columns), then progressively
coarser prefixes, ending at the empty set (the grand total). `CUBE(region,
product)` becomes the full powerset instead: `[{0,1}, {0}, {1}, {}]`.
`grouping_functions` records, per `GROUPING(...)`/`GROUPING_ID(...)` call
site, *which* group-column indices that particular call reports on — used
below to compute each call's per-grouping-set bitmask value.

This population happens in the **planner**, not the optimizer — before
`Optimizer::Optimize` is ever called:

```cpp
// src/planner/binder/query_node/plan_select_node.cpp:115
aggregate->grouping_sets = statement.groups.grouping_sets;
```

(and again at lines 127 and 178, for the different shapes a `SELECT`'s
plan can take depending on whether a `HAVING`/window clause needs its own
aggregate node). By the time `Optimizer::Optimize` receives the tree, a
`ROLLUP`/`CUBE`/`GROUPING SETS` query already has a fully-populated,
multi-entry `grouping_sets` vector on its `LogicalAggregate` — the
optimizer inherits this representation, it does not create it.

### The physical layer already knows what to do with it

`PhysicalPlanGenerator::CreatePlan(LogicalAggregate &op)` passes
`op.grouping_sets` straight into `PhysicalHashAggregate`'s constructor,
whether it holds one entry or several:

```cpp
// src/execution/physical_plan/plan_aggregate.cpp:306
auto &group_by = Make<PhysicalHashAggregate>(context, op.types, std::move(op.expressions), std::move(op.groups),
                                             std::move(op.grouping_sets), std::move(op.grouping_functions),
                                             op.estimated_cardinality, group_validity, op.distinct_validity);
```

The two faster, more specialized physical operators explicitly opt out the
moment there's more than one grouping set to handle, falling through to
the general `PhysicalHashAggregate` instead:

```cpp
// src/execution/physical_plan/plan_aggregate.cpp:34
static bool CanUsePartitionedAggregate(ClientContext &context, LogicalAggregate &op, PhysicalOperator &child,
                                       vector<column_t> &partition_columns) {
	if (op.grouping_sets.size() > 1 || !op.grouping_functions.empty()) {
		return false;
	}
	// ...
```

and the ungrouped-aggregate fast path is likewise gated on there being at
most one (trivial) grouping set:

```cpp
// src/execution/physical_plan/plan_aggregate.cpp:278
if (op.groups.empty() && op.grouping_sets.size() <= 1) {
	// no groups: use the dedicated ungrouped aggregate path
	auto &group_by = Make<PhysicalUngroupedAggregate>(op.types, std::move(op.expressions), op.estimated_cardinality,
	                                                  op.distinct_validity);
	// ...
```

`PhysicalHashAggregate`'s underlying radix-partitioned hash table is
already built to compute several grouping sets in one pass over the input
— each input row's key gets hashed once per grouping set it belongs to,
against a single shared hash table structure, rather than the plan tree
containing several separate aggregate operators. (The full mechanics of
that hash table are physical-execution material belonging to this book's
memory/aggregation chapter, not this one; the grouping-sets optimizer
file's own source comments reference it directly as `RadixPartitionedHashTable`
when explaining why `GROUPING()`'s bitmask value is a per-grouping-set
constant rather than something computed per row — see below.) This is the
concrete, source-verified basis for the claim above: DuckDB does not need
any logical-level "expand the grouping sets into separate operators" step
for the query to be *executable* at all — the physical operator already
handles the general case directly.

### So what does the `GROUPING_SETS` optimizer pass actually do?

`GroupingSetsOptimizer` is a single class used from **two different call
sites**, distinguished by one boolean constructor argument,
`mandatory_aggregate_lowering`:

```cpp
// src/include/duckdb/optimizer/grouping_sets_optimizer.hpp:17-22
//! The GroupingSetsOptimizer rewrites aggregates over multiple grouping sets (ROLLUP/CUBE/GROUPING SETS) into
//! explicit aggregate branches. Compatible aggregates use a cascade that combines exported states; other aggregates
//! use one branch per grouping set over a shared materialized input.
class GroupingSetsOptimizer : public LogicalOperatorVisitor {
public:
	explicit GroupingSetsOptimizer(Optimizer &optimizer, bool mandatory_aggregate_lowering = false);
```

**Call site 1 — the mandatory legalization pre-pass**, inside
`Optimizer::LowerMandatoryAggregateRewrites`, which runs unconditionally,
even with `PRAGMA disable_optimizer` set (§7.2):

```cpp
// src/optimizer/optimizer.cpp:567
unique_ptr<LogicalOperator> Optimizer::LowerMandatoryAggregateRewrites(unique_ptr<LogicalOperator> plan_p) {
	Verify(*plan_p);
	GroupingSetsOptimizer grouping_sets_optimizer(*this, true);
	grouping_sets_optimizer.VisitOperator(plan_p);
	MultiStageAggregateRewriter aggregate_rewriter(*this, AggregateRewritePolicy::MANDATORY);
	aggregate_rewriter.VisitOperator(plan_p);
	Verify(*plan_p);
	return plan_p;
}
```

**Call site 2 — the optional, disableable, timed pipeline pass**, #11 in
§7.4's numbered list:

```cpp
// src/optimizer/optimizer.cpp:321-325
// rewrite aggregates over multiple grouping sets (ROLLUP/CUBE/GROUPING SETS) into explicit aggregate plans
RunOptimizer(OptimizerType::GROUPING_SETS, [&]() {
	GroupingSetsOptimizer grouping_sets_optimizer(*this);
	grouping_sets_optimizer.VisitOperator(plan);
});
```

Both calls end up inside the same method, `TryRewriteGroupingSets`
(`grouping_sets_optimizer.cpp:228`), but the `mandatory_aggregate_lowering`
flag changes what it's willing to do:

```cpp
// src/optimizer/grouping_sets_optimizer.cpp:228
bool GroupingSetsOptimizer::TryRewriteGroupingSets(unique_ptr<LogicalOperator> &op) {
	if (op->type != LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY || op->children.size() != 1) {
		return false;
	}
	auto &aggr = op->Cast<LogicalAggregate>();
	if (aggr.grouping_sets.size() < 2 || aggr.expressions.empty()) {
		// the rewrite is only beneficial when multiple grouping sets are computed
		return false;
	}
	if (mandatory_aggregate_lowering) {
		for (auto &expr : aggr.expressions) {
			if (expr->GetExpressionClass() != ExpressionClass::BOUND_AGGREGATE) {
				return false;
			}
			auto &aggregate = expr->Cast<BoundAggregateExpression>();
			if (aggregate.Function().HasRewriteCallback() &&
			    aggregate.Function().GetRewritePolicy() == AggregateRewritePolicy::MANDATORY &&
			    aggregate.StateExportMode() != AggregateStateExportMode::STATE_EXPORT) {
				return TryExpandGroupingSets(op);
			}
		}
		return false;
	}
	// ... (the optional, cascade-first path — see below)
```

The very first line already answers the "is this pass ever a no-op"
question: **if a `LogicalAggregate` has fewer than two grouping sets, this
pass does nothing at all**, mandatory or optional — an ordinary `GROUP BY`
(which binds to exactly one implicit grouping set) never reaches either
branch of this logic.

For a genuine multi-grouping-set aggregate, the **mandatory** call
(`mandatory_aggregate_lowering = true`) only actually does anything if at
least one of the aggregate's expressions is a function that has registered
its own `RewriteCallback` with `AggregateRewritePolicy::MANDATORY` — a
third, distinct rewriting mechanism, this one keyed per aggregate
*function* rather than per matched expression shape (see the
`RewriteAggregateCallbacks` aside earlier in this part). Grepping the
source for real functions that set this exact policy turns up
`histogram()`, `mode()`, and `entropy()`:

```cpp
// extension/core_functions/aggregate/nested/histogram.cpp:277
function.SetRewriteCallback(RewriteHistogram, AggregateRewritePolicy::MANDATORY);
// extension/core_functions/aggregate/holistic/mode.cpp:624
function.SetRewriteCallback(RewriteMode, AggregateRewritePolicy::MANDATORY);
// extension/core_functions/aggregate/holistic/mode.cpp:770
function.SetRewriteCallback(RewriteEntropy, AggregateRewritePolicy::MANDATORY);
```

These are aggregates whose "real" implementation is itself an algebraic
rewrite into more primitive pieces (a histogram, for instance, decomposes
into a grouped count that then gets folded into a map/list structure) —
they are not primitive, distributive aggregates the way `SUM`/`COUNT`/`MIN`/`MAX`
are, and the physical multi-grouping-set hash table machinery that handles
ordinary aggregates directly cannot execute their rewritten pieces across
several grouping sets in a single shared operator the way it can for a
plain `SUM`. So: **`SELECT region, histogram(product) FROM sales GROUP BY
ROLLUP(region)`** is exactly the shape that forces `LowerMandatoryAggregateRewrites`'s
pre-pass to act — even with the entire optimizer disabled via `PRAGMA
disable_optimizer`, this query still needs its `ROLLUP` legalized into
explicit per-grouping-set branches before the plan can be executed at all,
because `histogram()`'s own mandatory rewrite can't operate correctly
inside a still-multi-grouping-set aggregate. An ordinary `ROLLUP(region,
product)` with only `SUM`/`COUNT`/`MIN`/`MAX` in its `SELECT` list, by
contrast, contains no `MANDATORY`-policy aggregate, so this loop finds
nothing and the mandatory pre-pass returns `false` — doing nothing,
leaving the `LogicalAggregate`'s multi-entry `grouping_sets` vector fully
intact for `RunBuiltInOptimizers()`'s later, **optional** pass to see.

### The optional path: cascade first, explicit branches as the fallback

When `mandatory_aggregate_lowering` is `false` (the pipeline's normal, #11
invocation), `TryRewriteGroupingSets` tries something considerably more
interesting than either "leave it alone" or "expand it":

```cpp
// src/optimizer/grouping_sets_optimizer.cpp:251
bool can_cascade = true;
for (auto &expr : aggr.expressions) {
	if (expr->GetExpressionClass() != ExpressionClass::BOUND_AGGREGATE) {
		return false;
	}
	if (!CanRewriteAggregate(expr->Cast<BoundAggregateExpression>())) {
		can_cascade = false;
	}
}
if (aggr.grouping_sets.size() > MAX_REWRITE_GROUPING_SETS) {
	// the rewritten plan nests a materialized CTE per grouping set - leave large grouping sets to the aggregate
	return false;
}
vector<GroupingSetLevel> levels;
if (!can_cascade || !FindCascade(aggr.grouping_sets, levels)) {
	// Explicit branches support arbitrary aggregates and expose single grouping sets to subsequent rewrites.
	return TryExpandGroupingSets(op);
}
```

The optional path has a size limit. If an aggregate has more than
`MAX_REWRITE_GROUPING_SETS = 1024` grouping sets
(`grouping_sets_optimizer.cpp:54`), the pass leaves it alone. Both rewrites
nest one materialized CTE per grouping set, so a huge `CUBE` would build a
very deep plan. The physical hash aggregate can still run all the sets
directly. The *parser* has a cap too: a `GROUP BY` that expands to more than
65,535 grouping sets is an error
(`src/parser/peg/transformer/transform_select.cpp:869`).
`test/sql/aggregate/grouping_sets/grouping_sets_many_sets.test` runs a
`CUBE` over 12 columns (4,096 sets, above the optimizer limit) and expects the
right totals. It also expects three 15-column `CUBE`s to fail with
`Maximum grouping set count of 65535 exceeded`.

**`CanRewriteAggregate`** rules out any aggregate that is inherently
unsafe to reconstruct from a coarser re-aggregation of a finer level's
already-computed result:

```cpp
// src/optimizer/grouping_sets_optimizer.cpp:56
static bool CanRewriteAggregate(const BoundAggregateExpression &aggregate) {
	if (aggregate.IsDistinct() || aggregate.GetOrderBys()) {
		// DISTINCT / ORDER BY aggregates cannot be computed by combining states of a finer aggregation
		return false;
	}
	if (aggregate.StateExportMode() != AggregateStateExportMode::NONE) {
		return false;
	}
	auto &function = aggregate.Function();
	if (!function.HasStateCombineCallback() || function.HasStateDestructorCallback() ||
	    !function.HasStateSizeCallback() || !function.HasStateFinalizeCallback() ||
	    !function.HasGetStateTypeCallback()) {
		return false;
	}
	return true;
}
```

`COUNT(DISTINCT x)` is the clearest intuition for why: once a finer
grouping level has already reduced its input down to `(region, product,
count-of-distinct-amounts)` rows, the individual distinct raw `amount`
values that went into computing that count are gone — recombining two
already-distinct-counted rows at a coarser grouping level cannot recover
how many *distinct* values existed across both of them combined, because
the duplicate-detection information was thrown away the moment the finer
level finished counting. A plain `SUM`/`COUNT`/`MIN`/`MAX`, by contrast,
genuinely can be re-aggregated: the sum of several groups' sums is the sum
of the whole; the max of several groups' maxes is the max of the whole.
The remaining `HasState*Callback()` checks mirror exactly what
`ExportAggregateFunction::Bind` itself requires to succeed (the comment
says so directly), since the cascade's mechanism, walked through next,
genuinely does export and re-import each aggregate's raw internal state.

**`FindCascade`** then checks whether the *set of grouping sets itself*
forms a structure amenable to this combining trick — specifically, whether
every grouping set can be derived by re-aggregating some other,
already-computed **superset** of its own group columns:

```cpp
// src/optimizer/grouping_sets_optimizer.cpp:88
static bool FindCascade(const vector<GroupingSet> &grouping_sets, vector<GroupingSetLevel> &levels) {
	// order the grouping sets by size (descending) - supersets must be computed before their subsets
	for (idx_t set_idx = 0; set_idx < grouping_sets.size(); set_idx++) {
		levels.emplace_back(set_idx);
	}
	std::stable_sort(levels.begin(), levels.end(), [&](const GroupingSetLevel &a, const GroupingSetLevel &b) {
		return grouping_sets[a.grouping_set_idx].size() > grouping_sets[b.grouping_set_idx].size();
	});
	// each level is computed from the smallest already-computed level that is a superset of it
	// for ROLLUP this forms a chain, for CUBE a lattice rooted in the complete grouping set
	for (idx_t level_idx = 1; level_idx < levels.size(); level_idx++) {
		auto &grouping_set = grouping_sets[levels[level_idx].grouping_set_idx];
		for (idx_t source_idx = level_idx; source_idx > 0; source_idx--) {
			auto &source_set = grouping_sets[levels[source_idx - 1].grouping_set_idx];
			if (std::includes(source_set.begin(), source_set.end(), grouping_set.begin(), grouping_set.end())) {
				auto source_level = source_idx - 1;
				if (source_set.size() == grouping_set.size() && levels[source_level].source_level.IsValid()) {
					// identical grouping set: share the source instead of chaining, to keep the cascade shallow
					source_level = levels[source_level].source_level.GetIndex();
				}
				levels[level_idx].source_level = source_level;
				levels[source_level].materialized = true;
				break;
			}
		}
		if (!levels[level_idx].source_level.IsValid()) {
			// no computed superset to compute this grouping set from - we cannot cascade
			return false;
		}
	}
	return true;
}
```

Every `ROLLUP(a, b, ...)` produces a chain of grouping sets each exactly
one column narrower than the last, so `FindCascade` always succeeds for a
`ROLLUP` (each level's immediate predecessor is trivially its superset).
`CUBE(a, b)` produces the full powerset, which is a *lattice* rather than a
simple chain — `{a,b}` is still the unique root everything else derives
from, but `{a}` and `{b}` both derive directly from `{a,b}` rather than
from each other, and `{}` (the grand total) can derive from either. An
**explicit `GROUPING SETS`** list, on the other hand, can name grouping
sets with no subset/superset relationship between them at all — the
`grouping_sets_optimizer.test` case for `GROUPING SETS ((region),
(product))`, walked through below, is exactly this: neither `{region}` nor
`{product}` is a superset of the other, so `FindCascade` fails, and
`TryRewriteGroupingSets` falls straight through to `TryExpandGroupingSets`
instead.

The `source_level` lines in `FindCascade` handle duplicates. A
`GROUPING SETS` list can name the *same* set many times, for example
`GROUPING SETS ((c0), (c0), (c0))`. Each copy must still produce its own rows.
If each copy were computed from the copy just before it, `n` copies would make
a chain `n` CTEs deep, and a long chain could overflow the stack. So a copy
reads from the same source as the copy before it, and the cascade stays
shallow. `test/sql/aggregate/grouping_sets/many_duplicate_grouping_sets.test`
checks that three copies of `(c0)` give three rows, and that 5,000 copies
work.

### The cascade, concretely: export states, combine, finalize

When `can_cascade` holds and `FindCascade` succeeds, the rewrite builds one
`LogicalAggregate` **per grouping-set level**, ordered from finest (most
columns) to coarsest:

- **The finest level** aggregates the *original* base input directly, but
  instead of computing each aggregate's finished value, it wraps every
  aggregate in `ExportAggregateFunction::Bind` so the `LogicalAggregate`
  produces each aggregate's raw, combinable internal **state** (an
  `AGGREGATE_STATE`-typed value) rather than a finished `SUM`/`COUNT`/etc.:

  ```cpp
  // src/optimizer/grouping_sets_optimizer.cpp:289
  for (auto &expr : aggr.expressions) {
      auto aggregate_copy = unique_ptr_cast<Expression, BoundAggregateExpression>(expr->Copy());
      auto export_aggregate = ExportAggregateFunction::Bind(std::move(aggregate_copy));
      if (!export_aggregate->GetReturnType().IsAggregateState()) {
          return false;
      }
      state_types.push_back(export_aggregate->GetReturnType());
      level_aggregates.push_back(std::move(export_aggregate));
  }
  ```

- **Every coarser level** does not touch the base input at all. It reads
  from a `LogicalCTERef` onto its immediate finer source level (which gets
  marked `materialized = true` and later wrapped in a real
  `LogicalMaterializedCTE`), re-groups by its own (narrower) set of group
  columns, and re-aggregates by calling a generic `combine(...)` aggregate
  over the *already-exported states* it reads from that CTE:

  ```cpp
  // src/optimizer/grouping_sets_optimizer.cpp:311
  for (idx_t aggr_idx = 0; aggr_idx < aggregate_count; aggr_idx++) {
      const auto state_pos = source_set.size() + aggr_idx;
      vector<unique_ptr<Expression>> arguments;
      arguments.push_back(make_uniq<BoundColumnRefExpression>(
          state_types[aggr_idx], ColumnBinding(cte_ref_index, ProjectionIndex(state_pos))));
      auto combine_function = GetBuiltinAggregateFunction(optimizer.context, CombineAggrFun::Name, {state_types[aggr_idx]});
      auto combine_aggregate = function_binder.BindAggregateFunction(std::move(combine_function), std::move(arguments));
      // ...
  ```

- **Every level's branch**, finally, finalizes its own states back into
  ordinary values (`optimizer.BindScalarFunction("finalize", ...)`,
  `grouping_sets_optimizer.cpp:393`) inside a `LogicalProjection`, filling
  in a literal `NULL` for every group column *not* part of that particular
  grouping set (`grouping_sets_optimizer.cpp:387`) — exactly the "rolled
  up to `NULL`" column behavior `ROLLUP`/`CUBE` output is defined to have.

The finished shape is a `UNION ALL` of one branch per grouping set, each
branch reading either the raw base data (finest level only) or a
materialized-CTE reference onto a finer level's exported states, wrapped
in as many nested `LogicalMaterializedCTE`s as there are re-used
intermediate levels:

```
ROLLUP(region, product) cascade shape (three levels, a simple chain):

  UNION ALL
   ├─ Projection (finalize states, NULL-fill missing groups)     ← grouping set {region, product}
   │    └─ LogicalAggregate  (groups: region, product; EXPORT_STATE aggregates)
   │         └─ <base input>                                      ← only this branch touches raw rows
   ├─ Projection (finalize states, NULL-fill missing groups)     ← grouping set {region}
   │    └─ LogicalAggregate  (groups: region; combine(state) aggregates)
   │         └─ LogicalCTERef → materialized {region, product} level
   └─ Projection (finalize states, NULL-fill missing groups)     ← grouping set {}
        └─ LogicalAggregate  (groups: none; combine(state) aggregates)
             └─ LogicalCTERef → materialized {region, product} level   (same CTE, reused)
```

Every re-aggregation past the finest level works over states, not raw
rows, and reads from a **materialized** CTE rather than recomputing the
finer level's aggregate twice — the whole point of the exercise. This
`LogicalMaterializedCTE`'s presence in the physical plan is exactly what
`grouping_sets_optimizer.test`'s first `EXPLAIN` assertion (below) is
checking a regex for.

### `TryExpandGroupingSets` — the fallback (and the mandatory path's only tool)

When cascading isn't possible — a non-cascadable aggregate
(`can_cascade = false`), an explicit `GROUPING SETS` list with no
subset/superset lattice (`FindCascade` fails), or the mandatory
legalization path, which never even attempts a cascade — `TryExpandGroupingSets`
is the fallback used instead, and it's considerably simpler: materialize
the **raw base input** once as a CTE, then build one completely
independent `LogicalAggregate` per grouping set, each reading from that
same shared CTE reference and re-grouping/re-aggregating from scratch:

```cpp
// src/optimizer/grouping_sets_optimizer.cpp:134-161 (structure, elided)
vector<unique_ptr<LogicalOperator>> branches(aggr.grouping_sets.size());
for (idx_t grouping_set_idx = 0; grouping_set_idx < aggr.grouping_sets.size(); grouping_set_idx++) {
	auto &grouping_set = aggr.grouping_sets[grouping_set_idx];
	auto branch_child = AggregateRewriteHelper::CreateCTERef(optimizer, cte_index, input_types, input_names,
	                                                         input_bindings, input_replacements);
	// ... build one independent LogicalAggregate over branch_child, grouped by only this grouping_set's columns
```

This still avoids re-*scanning* the underlying tables N times (the shared
input is scanned once and materialized), but every grouping set's
aggregation is computed independently, over the full input cardinality,
with no reuse of one level's partial work by another — strictly more work
than the cascade, but correct for arbitrary aggregate functions (including
`DISTINCT`, `histogram()`, `mode()`) and for grouping-set collections with
no lattice structure to exploit.

### `GROUPING()`/`GROUPING_ID()`: a per-grouping-set constant, not a per-row computation

Both rewrite paths compute each `GROUPING(...)` call's value the same way
— as a plain integer **bitmask constant** baked directly into that
branch's projection, not as anything computed per row:

```cpp
// src/optimizer/grouping_sets_optimizer.cpp:75
static int64_t GetGroupingValue(const GroupingSet &grouping_set,
                                const unsafe_vector<ProjectionIndex> &grouping_function) {
	int64_t grouping_value = 0;
	for (idx_t i = 0; i < grouping_function.size(); i++) {
		if (grouping_set.find(grouping_function[i]) == grouping_set.end()) {
			// we do not group on this column in this grouping set
			grouping_value += 1LL << (grouping_function.size() - (i + 1));
		}
	}
	return grouping_value;
}
```

For each column named in a particular `GROUPING(col1, col2, ...)` call,
its corresponding bit is set to `1` exactly when that column is **not**
part of the current branch's grouping set (i.e. it's been rolled up away,
replaced by `NULL`), and `0` when it is still a live grouping key — the bit
position runs most-significant-first across the call's argument list. This
matches the values in `grouping_sets_optimizer.test`'s own `GROUPING()`
test case exactly (walked through below): a bit is on only in exactly the
rows where that column shows `NULL` because it was aggregated away.

### Test walkthrough — `test/optimizer/grouping_sets_optimizer.test`

The file builds one small `sales` table used throughout:

```sql
CREATE TABLE sales (region VARCHAR, product VARCHAR, amount INTEGER);
INSERT INTO sales VALUES
	('eu', 'fish', 1), ('eu', 'bread', 2), ('us', 'fish', 3), ('us', NULL, 4),
	(NULL, 'bread', 5), ('eu', 'fish', 10), ('us', 'bread', NULL);
```

**1 — the cascade is visible in the physical plan:**

```sql
query II
EXPLAIN SELECT region, product, SUM(amount) FROM sales GROUP BY ROLLUP(region, product);
----
physical_plan	<REGEX>:.*CTE.*
```

`ROLLUP(region, product)` produces the three-set chain `{region,product},
{region}, {}`; `SUM` easily satisfies `CanRewriteAggregate`; `FindCascade`
succeeds trivially (a `ROLLUP`'s sets always form a simple chain). The
resulting plan contains a `LogicalMaterializedCTE` for the finest level,
matching the cascade diagram above — hence the positive `CTE` regex match
against the *physical* plan text.

**2 — disabling the pass removes the cascade, but the query still runs:**

```sql
statement ok
SET disabled_optimizers='grouping_sets';

query II
EXPLAIN SELECT region, product, SUM(amount) FROM sales GROUP BY ROLLUP(region, product);
----
physical_plan	<!REGEX>:.*CTE.*

statement ok
SET disabled_optimizers='';
```

With the pass off, `TryRewriteGroupingSets` never runs, `aggr.grouping_sets`
stays at its full three entries, and — per the physical-plan-generator
excerpts earlier in this section — `PhysicalHashAggregate` receives that
`vector<GroupingSet>` directly and executes all three grouping levels
itself, in one hash-table structure, with **no CTE anywhere in the plan**.
This pair of assertions is the single clearest, most direct piece of
evidence in the whole test suite for the claim this section opened with:
the multi-grouping-set representation is executable on its own; the
optimizer pass is purely about how efficiently that execution happens.

**3 — `ROLLUP` with several aggregate kinds, walked through mechanically:**

```sql
query IIIIII
SELECT region, product, SUM(amount), COUNT(*), COUNT(amount), MIN(amount) FROM sales GROUP BY ROLLUP(region, product) ORDER BY ALL;
----
eu	bread	2	1	1	2
eu	fish	11	2	2	1
eu	NULL	13	3	3	1
us	bread	NULL	1	0	NULL
us	fish	3	1	1	3
us	NULL	4	1	1	4
us	NULL	7	3	2	3
NULL	bread	5	1	1	5
NULL	NULL	5	1	1	5
NULL	NULL	25	7	6	1
```

Ten output rows for seven input rows, in three tiers: the finest
`(region, product)` pairs that actually occur in the data (`('eu','bread')`,
`('eu','fish')`, `('us','bread')`, `('us','fish')`, `('us', NULL)`,
`(NULL,'bread')` — six distinct pairs), then one coarser `(region, NULL)`
row per distinct `region` value including `NULL` itself (`'eu'`, `'us'`,
`NULL` — three rows), then exactly one grand-total `(NULL, NULL)` row.
`('us', NULL)` appears **twice** — once as its own finest-level pair (a row
where `product` is a genuine `NULL` in the source data, count `1`) and once
folded into the `('us', NULL)` *rolled-up* row (`region='us'`, `product`
rolled away, count `3` = all three `'us'` rows) — these are two distinct
grouping levels that happen to print identically in their leading two
columns; the surrounding aggregate values (`3` vs `7` for `SUM`, `1` vs `3`
for `COUNT(*)`) are what actually distinguishes them. `COUNT(amount)` vs
`COUNT(*)` diverges exactly once, on `('us','bread')`: that row's single
source row has `amount = NULL`, so `COUNT(*) = 1` but `COUNT(amount) = 0`
(non-`NULL` count), and `SUM(amount)` and `MIN(amount)` are both `NULL` for
that same group, per ordinary SQL aggregate-over-all-NULL-inputs
semantics.

**4 — `CUBE` produces the full powerset, not a chain:**

```sql
query IIII
SELECT region, product, SUM(amount), COUNT(*) FROM sales GROUP BY CUBE(region, product) ORDER BY ALL;
----
eu	bread	2	1
eu	fish	11	2
eu	NULL	13	3
us	bread	NULL	1
us	fish	3	1
us	NULL	4	1
us	NULL	7	3
NULL	bread	5	1
NULL	bread	7	3
NULL	fish	14	3
NULL	NULL	4	1
NULL	NULL	5	1
NULL	NULL	25	7
```

Thirteen rows here versus `ROLLUP`'s ten over the identical input — `CUBE`
adds the `{product}`-only grouping level (`GROUP BY product` alone, with
`region` rolled away), which `ROLLUP` never computes. `(NULL, 'bread')`
appears twice for exactly the reason `('us', NULL)` did above: once as a
genuine finest-level pair (a source row where `region` really is `NULL`,
`SUM=5`, `COUNT=1`) and once as the `{product}`-only level's aggregate over
*every* `'bread'` row regardless of region (`SUM=7`, `COUNT=3` — the two
`'eu'`/`'us'` bread rows plus the one genuinely-`NULL`-region bread row).
`(NULL, NULL)` likewise appears twice: once as the finest-level pair for
the one source row that's `NULL` in *both* columns (`region=NULL, amount=4`,
`COUNT=1`), and once as the true grand total (`SUM=25, COUNT=7`, all seven
rows). `FindCascade` handles `CUBE`'s lattice (not a simple chain) exactly
as designed: the `{product}`-only level and the `{region}`-only level both
derive independently from the same `{region,product}` finest level
(`std::includes` finds `{region,product} ⊇ {region}` and separately
`{region,product} ⊇ {product}`), and the grand total derives from whichever
of those it's checked against first in the sorted-by-size ordering.

**5 — explicit `GROUPING SETS`, including a literal duplicate set:**

```sql
query IIII
SELECT region, product, SUM(amount), COUNT(*) FROM sales GROUP BY GROUPING SETS ((region, product), (region), (region), ()) ORDER BY ALL;
----
eu	bread	2	1
eu	fish	11	2
eu	NULL	13	3
eu	NULL	13	3
us	bread	NULL	1
us	fish	3	1
us	NULL	4	1
us	NULL	7	3
us	NULL	7	3
NULL	bread	5	1
NULL	NULL	5	1
NULL	NULL	5	1
NULL	NULL	25	7
```

The `(region)` grouping set was named **twice** in the `GROUPING SETS`
list, and both occurrences show up as separate, byte-identical output rows
(`eu NULL 13 3` appears twice; `us NULL 7 3` appears twice; `NULL NULL 5 1`
appears twice). This test case establishes something specific:
`GROUPING_SETS`'s cascade/expansion is a **per-branch** rewrite — it never
deduplicates the *list* of grouping sets itself, only reasons about
subset/superset relationships between distinct entries in
`aggr.grouping_sets` (each occurrence of `{region}` is tracked as its own,
separate `GroupingSetLevel`). Removing genuinely duplicate *grouping-column
expressions within one single set* is a different pass entirely —
`DUPLICATE_GROUPS` (§7.4's pass #24, `src/optimizer/remove_duplicate_groups.cpp`)
— which operates at the granularity of one `GroupingSet`'s own column list,
not the granularity of "is this whole set repeated elsewhere in the
`GROUPING SETS (...)` list," and evidently isn't applied here (or isn't
scoped to catch this case) — the query's own literal duplication survives
straight through to the output.

`FindCascade` does not chain the second `{region}` below the first `{region}`;
both copies read from the same finer level (see the note after `FindCascade`
above). The output is the same either way.

**6 — `GROUPING()` is a per-branch constant, verified against `GetGroupingValue`'s bit math:**

```sql
query IIIIII
SELECT region, product, GROUPING(region), GROUPING(product), GROUPING(region, product), SUM(amount) FROM sales GROUP BY ROLLUP(region, product) ORDER BY ALL;
----
eu	bread	0	0	0	2
eu	fish	0	0	0	11
eu	NULL	0	1	1	13
us	bread	0	0	0	NULL
us	fish	0	0	0	3
us	NULL	0	0	0	4
us	NULL	0	1	1	7
NULL	bread	0	0	0	5
NULL	NULL	0	1	1	5
NULL	NULL	1	1	3	25
```

`GROUPING(region, product)` is a two-argument call, so per
`GetGroupingValue`'s bit-position formula (`1LL << (2 - (i+1))`), `region`
(argument index `0`) contributes bit value `2` when rolled away, and
`product` (index `1`) contributes bit value `1` when rolled away. Reading
the grand-total row: `region` and `product` are both rolled away there, so
`GROUPING(region, product) = 2 + 1 = 3` — exactly the `3` printed. The
`('eu', NULL)` row has only `product` rolled away (`region='eu'` is a live
group key at that level), so `GROUPING(region, product) = 0 + 1 = 1`,
matching `GROUPING(product) = 1` on the same row and `GROUPING(region) = 0`.
Every finest-level row (both columns live) shows all three `GROUPING`
columns as `0`, as expected — nothing has been rolled away yet at that
level.

**7 — `FILTER` on a rolled-up aggregate applies once, at the finest level:**

```sql
query III
SELECT region, product, SUM(amount) FILTER (region = 'eu') FROM sales GROUP BY ROLLUP(region, product) ORDER BY ALL;
----
eu	bread	2
eu	fish	11
eu	NULL	13
us	bread	NULL
us	fish	NULL
us	NULL	NULL
us	NULL	NULL
NULL	bread	NULL
NULL	NULL	13
NULL	NULL	NULL
```

Every `'us'`-region row (at every grouping level) comes out `NULL` — the
`FILTER (region = 'eu')` clause is a property attached directly to the
`BoundAggregateExpression` itself, and it rides along, unmodified, into
whichever level's aggregate the cascade builds (finest level: filtered
directly against raw rows; coarser levels: `combine()`d from the finer
level's already-filtered exported state, which is why the filter doesn't
need to be re-applied or even re-expressed at those levels — a row that
never contributed to the finer level's state can't un-filter its way back
in during a `combine`). The grand-total row correctly shows `13` — the same
total as the `('eu', NULL)` row — because once `region <> 'eu'` rows are
filtered out at the finest level, everything left to combine up to the
grand total *is* exclusively `'eu'` data.

**8 — expressions as `ROLLUP` keys:**

```sql
query III
SELECT UPPER(region), product || '!', SUM(amount) FROM sales GROUP BY ROLLUP(UPPER(region), product || '!') ORDER BY ALL;
----
EU	bread!	2
...
```

Confirms `ROLLUP`'s keys aren't restricted to bare column references —
arbitrary expressions work as grouping keys, each treated as its own
opaque group-by expression by the same indices-into-`groups` machinery
described above. (This is a different concern from
`grouping_expression_simplification.test`, covered next — that file is
about whether two *differently-spelled but equivalent* grouping
expressions collapse to the same bound expression before grouping even
happens, not about whether expressions are allowed as group keys at all,
which this test already answers "yes" to independently.)

**9 — `HAVING` on top of a `ROLLUP`, and 10 — an empty table:**

```sql
query III
SELECT region, product, SUM(amount) FROM sales GROUP BY ROLLUP(region, product) HAVING SUM(amount) > 5 ORDER BY ALL;
----
eu	fish	11
eu	NULL	13
us	NULL	7
NULL	NULL	25
```

An ordinary `LogicalFilter` sitting above the entire cascade/expansion
`UNION ALL` — nothing here is specific to grouping sets; `HAVING` filters
whatever rows the (already fully-resolved, single-level-per-row) union
output.

```sql
CREATE TABLE empty_sales (region VARCHAR, product VARCHAR, amount INTEGER);

query IIII
SELECT region, product, SUM(amount), COUNT(*) FROM empty_sales GROUP BY ROLLUP(region, product) ORDER BY ALL;
----
NULL	NULL	NULL	0
```

Exactly **one** output row over zero input rows — not three (one per
grouping-set level) and not zero. This matches standard SQL semantics for
grouping by the empty set specifically: `GROUP BY ()` is a *global*
aggregation, which (like a bare aggregate with no `GROUP BY` at all)
always produces exactly one row regardless of how many input rows
qualify, while `GROUP BY region` or `GROUP BY region, product` over zero
input rows produces zero groups, hence zero output rows, at those two
finer levels. The single surviving row is the grand total's `NULL, NULL,
NULL, 0` — `SUM` over zero rows is `NULL`; `COUNT(*)` over zero rows is
`0`. (The mechanism inside the physical multi-grouping-set hash table that
guarantees the empty grouping set always emits its one row even when every
other grouping set in the same query emits none is execution-layer
machinery this deep dive didn't trace in detail — it belongs with the rest
of `RadixPartitionedHashTable`'s internals, material for this book's
memory/aggregation chapter rather than the optimizer chapter.)

**11 — `DISTINCT` cannot cascade, but is still correct:**

```sql
query III
SELECT region, product, COUNT(DISTINCT amount) FROM sales GROUP BY ROLLUP(region, product) ORDER BY ALL;
----
eu	bread	1
...
NULL	NULL	6
```

Directly exercises `CanRewriteAggregate`'s `aggregate.IsDistinct()` guard:
`can_cascade` becomes `false` the moment this loop inspects `COUNT(DISTINCT
amount)`, so `TryRewriteGroupingSets` falls straight through to
`TryExpandGroupingSets` — one independent `LogicalAggregate` per grouping
set, each computing its own `COUNT(DISTINCT amount)` from scratch over the
(shared, materialized, but not partially-pre-aggregated) raw input. The
grand total, `6`, is the count of distinct `amount` values across the
whole table (`1,2,3,4,5,10` — `NULL` doesn't count, and note the value `4`
appears once); this is a value no cascade of already-summarized states
could have reconstructed, which is exactly the point `CanRewriteAggregate`
exists to enforce.

**12 — grouping sets with no common superset structure:**

```sql
query III
SELECT region, product, SUM(amount) FROM sales GROUP BY GROUPING SETS ((region), (product)) ORDER BY ALL;
----
eu	NULL	13
us	NULL	7
NULL	bread	7
NULL	fish	14
NULL	NULL	4
NULL	NULL	5
```

`{region}` and `{product}` are each other's *siblings*, not one a superset
of the other — `FindCascade`'s inner loop (searching, for `{product}`,
through already-placed levels for one that `std::includes` its columns)
finds nothing, `source_level` stays invalid, and the function returns
`false`. `TryRewriteGroupingSets` falls back to `TryExpandGroupingSets`:
two fully independent `LogicalAggregate`s (one `GROUP BY region`, one
`GROUP BY product`), each over the same materialized copy of the raw
input, unioned together — the "regular path" the test's own comment names.

**13/14 — payload-type generality, and three-column `ROLLUP`/`CUBE`:**

The remaining cases (`ROLLUP`/`CUBE` over a `measurements` table with
`DECIMAL`, `VARCHAR`, and `DOUBLE` payload columns, and three-column
`ROLLUP(grp, subgrp, grp + subgrp)`/`CUBE(grp, subgrp, grp % 2)`) don't
introduce new mechanics — they confirm the same cascade/expansion
machinery generalizes cleanly past two columns and past `SUM`/`COUNT`
specifically to `MIN`/`MAX` over `VARCHAR` and `SUM` over `DOUBLE`, and
that a computed group key (`grp + subgrp`, `grp % 2`) is treated as just
another opaque `groups` entry, exactly as case 8 already established for
`ROLLUP`.

### Test walkthrough — `test/optimizer/grouping_expression_simplification.test`

```sql
CREATE TABLE test(t timestamp);
PRAGMA explain_output = OPTIMIZED_ONLY;

query I nosort year
EXPLAIN SELECT COUNT(*) FROM test GROUP BY EXTRACT(year from t)
----

query I nosort year
EXPLAIN SELECT COUNT(*) FROM test GROUP BY YEAR(t)
----
```

This is the same-label idiom again (see the earlier aside on it), this
time asserting that two *syntactically different but semantically
identical* `GROUP BY` key expressions produce a **byte-identical optimized
plan**. It is not part of `GroupingSetsOptimizer` at all, and it is not
testing anything about multiple grouping sets — this file's own
description, *"Make sure expressions are optimized in the groups too,"*
says precisely what it checks: that the 37-rule `EXPRESSION_REWRITER` pass
(§7.5, pass #1 — the very first pass in the whole pipeline) is applied to
expressions sitting inside a `LogicalAggregate`'s `groups` list, not only
to expressions inside `LogicalFilter`/`LogicalProjection`.

`EXTRACT(year FROM t)` binds to a call to the internal `date_part`
function with a constant `'year'` specifier string; `DatePartSimplificationRule`
— one of the 37 rules from §7.5's list — recognizes exactly this shape
(a `date_part` call whose specifier argument is a constant `VARCHAR`) and
replaces it with a direct call to the dedicated, specifier-free `year(...)`
scalar function:

```cpp
// src/optimizer/rule/date_part_simplification.cpp:15
DatePartSimplificationRule::DatePartSimplificationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	auto func = make_uniq<FunctionExpressionMatcher>();
	func->function = make_uniq<SpecificFunctionMatcher>("date_part");
	auto specifier = make_uniq<ConstantExpressionMatcher>();
	specifier->type = make_uniq<SpecificTypeMatcher>(LogicalType::VARCHAR);
	// ...
```

```cpp
// src/optimizer/rule/date_part_simplification.cpp:40
switch (specifier) {
case DatePartSpecifier::YEAR:
	new_function_name = "year";
	break;
// ...
```

`YEAR(t)` written directly, meanwhile, already binds straight to that same
`year` scalar function with no `date_part` indirection at all. So after
pass #1 has run over both queries' `groups` expression lists, both `GROUP
BY EXTRACT(year FROM t)` and `GROUP BY YEAR(t)` are grouping by the
literally identical bound expression `year(t)` — which is exactly why the
same-label idiom's byte-for-byte comparison passes. This test's role in
this book's coverage is narrow and specific: it confirms the *scope* of
pass #1's tree walk includes a `LogicalAggregate`'s grouping keys, a detail
easy to overlook if you only think of "expression rewriting" as something
that happens to `WHERE`/`SELECT` expressions. It has no relationship to
`GroupingSetsOptimizer` beyond sharing a test directory and a general
subject-matter neighborhood ("things that touch `GROUP BY`").

## Recap

- **`AGGREGATE_FUNCTION_REWRITER`** bundles three independent, separately-run
  rules (`AvgRewriteRule`, `SumRewriteRule`, `ListRewriteRule`), not one.
  `SumRewriteRule`'s type restriction is enforced in two layers — `IsIntegral()`
  excludes floats/`DECIMAL` outright (the associativity argument), and a
  narrower per-operand type set excludes `HUGEINT`/`UHUGEINT` specifically
  (a headroom-before-wraparound argument, distinct from the
  floating-point one) — and its `ShouldSkip` restricts it, independently of
  types, to bare aggregates with no `GROUP BY` at all, a historical
  narrowing the source comments themselves call out as tighter than it
  needs to be. `sum_no_overflow`, which shows up in this file's own test
  suite, belongs to a completely different, later, statistics-driven pass.
- **`GROUPING_SETS`** does not expand `ROLLUP`/`CUBE`/`GROUPING SETS`
  syntax — the binder already does that, before optimization starts, into
  a `vector<GroupingSet>` the physical hash-aggregate operator can execute
  natively. The optimizer pass is an optional performance transform that
  replaces one full-cardinality multi-grouping-set scan with a cascade of
  progressively coarser re-aggregations over already-exported aggregate
  states, materialized via CTEs — falling back to independent per-set
  branches whenever an aggregate (`DISTINCT`, `ORDER BY`) or the grouping
  sets' own shape (no subset/superset lattice) makes cascading impossible.
  A *separate*, function-level `MANDATORY`-rewrite-policy mechanism
  (`histogram()`, `mode()`, `entropy()`) forces the same expansion fallback
  to run unconditionally, even with the rest of the optimizer disabled —
  for a different, narrower reason than performance.
- **Limits on both passes:** a grouped `AVG` is split into `SUM`/`COUNT`
  only when one of the two can be shared with another aggregate
  (`ShouldRewrite`). The `GROUPING_SETS` pass skips aggregates with more
  than 1,024 grouping sets, and identical grouping sets in a cascade share one
  source instead of forming a deep chain.
