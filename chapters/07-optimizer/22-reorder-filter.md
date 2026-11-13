# 7.6.1 (deep dive) — `REORDER_FILTER`: cheap-to-costly conjunct ordering, and the guard that keeps it safe

*Source: `src/optimizer/expression_heuristics.cpp`,
`src/include/duckdb/optimizer/expression_heuristics.hpp`.
`OptimizerType::REORDER_FILTER`, pass #40 in §7.4's pipeline listing
(`optimizer.cpp:530-534`, run inside a comment that says exactly what it's
for: *"apply simple expression heuristics to get an initial reordering"*).
Tests: `test/optimizer/expression_rewriter/functions_that_error_are_not_reordered.test`
(walked in full below, including two blocks about other passes), plus the historical regression tests attached to the
commits that hardened the guard (`test/sql/optimizer/test_no_pushdown_cast_into_cte.test`,
`test/sql/function/operator/test_division_by_zero.test`). Cross-referenced
throughout against Chapter 12's `AdaptiveFilterState`/`AdaptiveFilter`
material (§12.12), which this file's final section contrasts in full.*

The overview (`00-overview.md`, §7.6.1) already gave this pass a paragraph:
it sorts a filter's conjuncts cheap-to-costly using a static per-expression
cost heuristic, and it refuses to touch the order at all if any conjunct
`CanThrow()` or carries a barrier. That paragraph is correct as far as it
goes. What it doesn't do — and what this file exists to do — is show the
*actual* cost numbers the heuristic assigns (every one of them, reproduced
from source, not summarized), show exactly what `CanThrow()` means
mechanically and how conservative it really is, walk the one real test file
whose name promises to be about exactly this hazard, and draw a precise line
between this pass (which runs once, at plan time, with zero knowledge of the
data) and `AdaptiveFilter` (Chapter 12, §12.12 — which runs continuously, at
execution time, with nothing *but* knowledge of the data). The two are easy
to conflate — both "reorder conjuncts by cost" — and the git history of this
exact corner of DuckDB shows the DuckDB team itself patching what is
recognizably the *same bug* in both mechanisms, independently, three days
apart, which is about as clear a piece of evidence as you could ask for that
they are genuinely two different systems and not one system wearing two
names.

## 1. Where it sits, and why last

`REORDER_FILTER` is pass #40 of the ~45 in `RunBuiltInOptimizers()` — deep
into the "cleanup" tail of the pipeline, after `JOIN_ORDER` (#17),
`STATISTICS_PROPAGATION` (#34), and every rewrite that could still add,
remove, or restructure a conjunct has already had its turn:

```cpp
// src/optimizer/optimizer.cpp:530-534
// apply simple expression heuristics to get an initial reordering
RunOptimizer(OptimizerType::REORDER_FILTER, [&]() {
	ExpressionHeuristics expression_heuristics(*this);
	plan = expression_heuristics.Rewrite(std::move(plan));
});
```

The ordering is deliberate, in the same sense every other late-pipeline
placement in §7.4 is deliberate: `FILTER_PUSHDOWN` (#6) may have split one
`WHERE` clause's conjuncts across several different operators (some landing
in a `LogicalGet`'s `table_filters`, some staying behind as a residual
`LogicalFilter`); `FILTER_PULLUP` (#5) may have manufactured brand-new
conjuncts that didn't exist in the original SQL; `IN_CLAUSE` (#9) may have
deleted a conjunct entirely by turning it into a join. By the time pass #40
runs, every `LogicalFilter` in the tree has its *final* conjunct list — the
one the physical plan generator is about to turn into a `PhysicalFilter` — and
reordering that final list is the very last structural thing that happens to
it before physical planning. Reorder any earlier and a later pass could
still shuffle the list again for reasons that have nothing to do with cost;
reorder any later and there's no plan-level operator left to reorder (only
execution machinery, which is exactly the boundary Chapter 12 starts from).

## 2. The class, and its two entry points

`ExpressionHeuristics` (`src/include/duckdb/optimizer/expression_heuristics.hpp:18`)
is a `LogicalOperatorVisitor` with a small, three-method public surface:

```cpp
// src/include/duckdb/optimizer/expression_heuristics.hpp:18-38
class ExpressionHeuristics : public LogicalOperatorVisitor {
public:
	explicit ExpressionHeuristics(Optimizer &optimizer) : optimizer(optimizer) {
	}

	Optimizer &optimizer;
	unique_ptr<LogicalOperator> root;

public:
	//! Search for filters to be reordered
	unique_ptr<LogicalOperator> Rewrite(unique_ptr<LogicalOperator> op);
	//! Reorder the expressions of a filter
	void ReorderExpressions(vector<unique_ptr<Expression>> &expressions);
	//! Return the cost of an expression
	static idx_t Cost(const Expression &expr);

	static vector<idx_t> GetInitialOrder(const TableFilterSet &table_filters);

	unique_ptr<Expression> VisitReplace(BoundConjunctionExpression &expr, unique_ptr<Expression> *expr_ptr) override;
	//! Override this function to search for filter operators
	void VisitOperator(LogicalOperator &op) override;
	// ... (private ExpressionCost/*Cost helpers follow, covered in §3)
```

`VisitOperator` is the tree walk's hook for the obvious case — a
`LogicalFilter` with more than one expression in its `expressions` list:

```cpp
// src/optimizer/expression_heuristics.cpp:18-29
void ExpressionHeuristics::VisitOperator(LogicalOperator &op) {
	if (op.type == LogicalOperatorType::LOGICAL_FILTER) {
		// reorder all filter expressions
		if (op.expressions.size() > 1) {
			ReorderExpressions(op.expressions);
		}
	}
	VisitOperatorChildren(op);
	VisitOperatorExpressions(op);
}
```

`VisitReplace(BoundConjunctionExpression&, ...)` is the less obvious case,
and it's the reason a `WHERE a AND b` buried inside something that *isn't* a
`LogicalFilter` — a `CASE WHEN a AND b THEN ...`, an `AND` inside a scalar
subquery's own predicate, an `OR` chain nested inside a function argument —
still gets reordered:

```cpp
// src/optimizer/expression_heuristics.cpp:31-34
unique_ptr<Expression> ExpressionHeuristics::VisitReplace(BoundConjunctionExpression &expr,
                                                          unique_ptr<Expression> *expr_ptr) {
	ReorderExpressions(expr.GetChildrenMutable());
	return nullptr;
}
```

`VisitOperatorExpressions(op)` at the end of `VisitOperator` is what drives
`LogicalOperatorVisitor`'s generic expression walk into every expression in
every operator, which is what gives `VisitReplace` a chance to fire on any
`BoundConjunctionExpression` node anywhere in the tree, not just the ones
sitting directly under a `LogicalFilter`. This split — one hook for the
common case, one for the general case — wasn't there from the beginning:
`git log` on this file shows it was added six years before this book's
snapshot commit, by Mark Raasveldt, in a commit titled, plainly,
*"ExpressionHeuristics: also reorder inside AND/OR expressions"*
(`acb925884a`, Dec 20 2019) — the `VisitReplace` override and its
`BoundConjunctionExpression` overload didn't exist before that commit; only
the `LogicalFilter`-only path did. The generalization is small in code but
real in scope: it's the difference between "reorders `WHERE` clauses" and
"reorders every conjunction anywhere a conjunction can occur."

Both entry points funnel into the same `ReorderExpressions`, covered next.

## 3. The cost model, reproduced in full

`ExpressionHeuristics::Cost` (`expression_heuristics.cpp:187-227`) is a
`switch` on `Expression::GetExpressionClass()` — DuckDB's coarse expression
*kind* tag (not its SQL operator) — and each branch either returns a number
directly or delegates to one of eight small `ExpressionCost`/`*Cost` helper
overloads:

```cpp
// src/optimizer/expression_heuristics.cpp:187-227
idx_t ExpressionHeuristics::Cost(const Expression &expr) {
	switch (expr.GetExpressionClass()) {
	case ExpressionClass::BOUND_CASE: {
		auto &case_expr = expr.Cast<BoundCaseExpression>();
		return ExpressionCost(case_expr);
	}
	case ExpressionClass::BOUND_CONJUNCTION: {
		auto &conj_expr = expr.Cast<BoundConjunctionExpression>();
		return ExpressionCost(conj_expr);
	}
	case ExpressionClass::BOUND_FUNCTION: {
		auto &func_expr = expr.Cast<BoundFunctionExpression>();
		return ExpressionCost(func_expr);
	}
	case ExpressionClass::BOUND_OPERATOR: {
		auto &op_expr = expr.Cast<BoundOperatorExpression>();
		return ExpressionCost(op_expr, expr.GetExpressionType());
	}
	case ExpressionClass::BOUND_COLUMN_REF: {
		auto &col_expr = expr.Cast<BoundColumnRefExpression>();
		return ExpressionCost(col_expr.GetReturnType().InternalType(), 8);
	}
	case ExpressionClass::BOUND_CONSTANT: {
		auto &const_expr = expr.Cast<BoundConstantExpression>();
		return ExpressionCost(const_expr.GetReturnType().InternalType(), 1);
	}
	case ExpressionClass::BOUND_PARAMETER: {
		auto &const_expr = expr.Cast<BoundParameterExpression>();
		return ExpressionCost(const_expr.GetReturnType().InternalType(), 1);
	}
	case ExpressionClass::BOUND_REF: {
		auto &col_expr = expr.Cast<BoundReferenceExpression>();
		return ExpressionCost(col_expr.GetReturnType().InternalType(), 8);
	}
	default: {
		break;
	}
	}
	// return a very high value if nothing matches
	return 1000;
}
```

That `default: break;` matters as much as any of the named cases: any
expression class this switch doesn't recognize at all —
`BOUND_AGGREGATE`, `BOUND_WINDOW`, `BOUND_SUBQUERY`, `BOUND_LAMBDA`, `BOUND_UNNEST`
— falls all the way through to the final `return 1000;`. This is a genuine
"we don't know, assume the worst" default, and it lands on exactly the same
number (1000) as the *other* explicit "we don't recognize this function"
fallback inside `ExpressionCost(BoundFunctionExpression&)`, covered next —
which means, functionally, "an expression this heuristic has no specific
opinion about" and "a scalar function this heuristic has no entry for in its
lookup table" are treated identically: expensive, sort last, by default.

### 3.1 Leaves: column references, constants, parameters

```cpp
// src/optimizer/expression_heuristics.cpp:174-186
idx_t ExpressionHeuristics::ExpressionCost(PhysicalType return_type, idx_t multiplier) {
	switch (return_type) {
	case PhysicalType::VARCHAR:
		return 5 * multiplier;
	case PhysicalType::FLOAT:
	case PhysicalType::DOUBLE:
		return 2 * multiplier;
	default:
		return 1 * multiplier;
	}
}
```

Column references and physical references (`BOUND_COLUMN_REF`, `BOUND_REF`)
use `multiplier = 8`; constants and parameters (`BOUND_CONSTANT`,
`BOUND_PARAMETER`) use `multiplier = 1`. Multiplied out, that gives a small,
concrete leaf-cost table:

| leaf kind | physical type | cost |
|---|---|---|
| column/physical reference | fixed-width (`INT32`, `BOOL`, `INT64`, …) | `1 × 8 = 8` |
| column/physical reference | `FLOAT`/`DOUBLE` | `2 × 8 = 16` |
| column/physical reference | `VARCHAR` | `5 × 8 = 40` |
| constant/parameter | fixed-width | `1 × 1 = 1` |
| constant/parameter | `FLOAT`/`DOUBLE` | `2 × 1 = 2` |
| constant/parameter | `VARCHAR` | `5 × 1 = 5` |

Two things worth noticing: reading a column is always modeled as strictly
more expensive than reading a constant of the same physical type (`8` vs
`1` at the cheap end) — a constant needs no memory access at all, a column
reference does — and a `VARCHAR` value, whether column or constant, is
modeled as `5×` costlier than a fixed-width value of the same role, which is
the same "wide, variable-length values cost more to touch" reasoning Chapter
12 attaches to `PhysicalType::VARCHAR` elsewhere in the execution engine.
This heuristic isn't measuring string length at all (it can't — this runs at
plan time, over an unbound number of future rows) — it's a flat, syntactic
proxy: *this expression's static type is `VARCHAR`, therefore assume it's
more expensive than an `INTEGER` of the same shape*, nothing more refined
than that.

### 3.2 `BOUND_OPERATOR` — `IS [NOT] NULL`, `NOT`, `IN`/`NOT IN`

```cpp
// src/optimizer/expression_heuristics.cpp:154-172
idx_t ExpressionHeuristics::ExpressionCost(const BoundOperatorExpression &expr, ExpressionType expr_type) {
	idx_t sum = 0;
	for (auto &child : expr.GetChildren()) {
		sum += Cost(*child);
	}
	if (expr_type == ExpressionType::OPERATOR_IS_NULL || expr_type == ExpressionType::OPERATOR_IS_NOT_NULL) {
		return sum + 5;
	} else if (expr_type == ExpressionType::COMPARE_IN || expr_type == ExpressionType::COMPARE_NOT_IN) {
		return sum + (expr.GetChildren().size() - 1) * 100;
	} else if (expr_type == ExpressionType::OPERATOR_NOT) {
		return sum + 10; // TODO: evaluate via measured runtimes
	} else {
		return sum + 1000;
	}
}
```

`IS [NOT] NULL` is cheap (`+5`) — a validity-bitmask check, no data touched
at all. `NOT` is a little more (`+10`, and the `TODO` comment is verbatim in
the source — DuckDB's own authors flag this specific constant as a guess, not
a measurement). `IN (...)`/`NOT IN (...)` scales with the length of the
list: `+100` *per extra value after the first* — a ten-item `IN` list costs
`900` just for the list itself, on top of whatever the compared expression
costs, which is a real, structural reason a long `IN` list sorts very late
relative to almost anything else on this table (and is also exactly why the
plan-level `IN_CLAUSE` pass, #9, exists to convert long `IN` lists into a
join before reaching this point at all — see §7.6.6 of the overview). Any
other `BOUND_OPERATOR` expression type this switch doesn't name explicitly
falls to the same `+1000` default the top-level `Cost` dispatch uses for
unrecognized classes.

### 3.3 `BOUND_FUNCTION` — comparisons, casts, `BETWEEN`, and the named-function table

This is the branch that carries the most weight, because — per Chapter 12
§12.5's note that comparisons are `BOUND_FUNCTION`s — an ordinary
`x = 5` or `x < y` predicate is *not* a distinct expression class in this
checkout; it's a `BoundFunctionExpression` around the `=`/`<`/… scalar
function, dispatched through this exact branch of `Cost`'s switch. The
`ExpressionCost(const BoundFunctionExpression&)` overload checks, in order,
whether the function is a comparison, a `BETWEEN`, or a cast, before falling
back to a fixed name-keyed table:

```cpp
// src/optimizer/expression_heuristics.cpp:125-152
idx_t ExpressionHeuristics::ExpressionCost(const BoundFunctionExpression &expr) {
	if (expr.GetExpressionType() == ExpressionType::COMPARE_BETWEEN) {
		return BetweenExpressionCost(expr);
	}
	if (BoundComparisonExpression::IsComparison(expr)) {
		return ComparisonExpressionCost(expr);
	}
	if (BoundCastExpression::IsCast(expr)) {
		return CastExpressionCost(expr);
	}
	identifier_map_t<idx_t> function_costs = {{"+", 5},       {"-", 5},    {"&", 5},          {"#", 5},
	                                          {">>", 5},      {"<<", 5},   {"abs", 5},        {"*", 10},
	                                          {"%", 10},      {"/", 15},   {"date_part", 20}, {"year", 20},
	                                          {"round", 100}, {"~~", 200}, {"!~~", 200},      {"regexp_matches", 200},
	                                          {"||", 200}};

	idx_t cost_children = 0;
	for (auto &child : expr.GetChildren()) {
		cost_children += Cost(*child);
	}
	auto cost_function = function_costs.find(expr.Function().GetName());
	if (cost_function != function_costs.end()) {
		return cost_children + cost_function->second;
	} else {
		return cost_children + 1000;
	}
}
```

`BoundComparisonExpression::IsComparison`/`::Left`/`::Right` here are the
static accessor helpers Chapter 12 already documented as existing precisely
*because* comparisons migrated to `BOUND_FUNCTION` representation — this
file is a second, independent confirmation of that same fact from a
completely different corner of the codebase: the optimizer's own cost
heuristic has to explicitly special-case "is this `BoundFunctionExpression`
secretly a comparison" before it can fall through to the generic
name-lookup path, which it would not need to do if comparisons still had
their own dedicated `ExpressionClass`.

**Comparisons** (`ComparisonExpressionCost`, `expression_heuristics.cpp:108-113`):
`Cost(left) + 5 + Cost(right)` — a flat `+5` regardless of which comparison
operator it is (`=`, `<>`, `<`, `<=`, `>`, `>=` all cost the same `5`; the
heuristic makes no distinction between them at all).

**`BETWEEN`** (`BetweenExpressionCost`, `expression_heuristics.cpp:72-77`):
`Cost(input) + Cost(lower_bound) + Cost(upper_bound) + 10` — modeled as
"comparison-ish," a little pricier than one bare comparison, cheaper than two
separate ones would be (`5+5=10` for two comparisons' flat overhead alone,
before even counting that `input` would otherwise be costed twice).

**Casts** (`CastExpressionCost`, `expression_heuristics.cpp:90-106`):

```cpp
// src/optimizer/expression_heuristics.cpp:90-106
idx_t ExpressionHeuristics::CastExpressionCost(const BoundFunctionExpression &expr) {
	auto source_type = BoundCastExpression::SourceType(expr);
	idx_t cast_cost = 0;
	if (expr.GetReturnType() != source_type) {
		if (expr.GetReturnType().id() == LogicalTypeId::VARCHAR || source_type.id() == LogicalTypeId::VARCHAR ||
		    expr.GetReturnType().id() == LogicalTypeId::BLOB || source_type.id() == LogicalTypeId::BLOB) {
			cast_cost = 200;
		} else {
			cast_cost = 5;
		}
	}
	return Cost(BoundCastExpression::Child(expr)) + cast_cost;
}
```

A cast is either free (source and target type identical — the cast is a
no-op the binder inserted and folds away), cheap (`+5`, any fixed-width
numeric-to-numeric conversion), or expensive (`+200`, anything touching
`VARCHAR` or `BLOB` on either end) — the same `200` bracket the named-function
table reserves for `LIKE`/`NOT LIKE`/`regexp_matches`/string concatenation,
which is a deliberate, consistent signal throughout this file: *string
work is expensive, full stop*, whether it shows up as an explicit function
call or as an implicit cast.

**The named-function table**, reproduced verbatim above, is the heuristic's
entire opinion about scalar functions, and it's worth stating exactly which
17 names it knows and nothing more: `+`, `-`, `&`, `#`, `>>`, `<<`, `abs`
(all `5`); `*`, `%` (`10`); `/` (`15`); `date_part`, `year` (`20`); `round`
(`100`); `~~` (`LIKE`), `!~~` (`NOT LIKE`), `regexp_matches`, `||` (all
`200`). Every scalar function *not* in that table — `sqrt`, `xor`,
`list_kurtosis`, `substring`, every extension function, every function added
to DuckDB since this table was last touched — falls to `cost_children +
1000`, the same "expensive by default" bucket the unrecognized-class default
uses. Two gaps worth flagging explicitly, in the spirit of this book's
policy of naming what a naive reading would miss: the table has an entry for
`/` (true division, cost `15`) but **no entry for `//`** (integer division,
DuckDB's separate, distinctly-named operator — `arithmetic.cpp`'s
`OperatorIntegerDivideFun::GetFunctions()` registers it as a function
literally named `"//"`). A query using `//` doesn't get costed at `15`; it
falls straight through to the `+1000` unknown-function default, exactly as
if it were `sqrt` or `xor`. This is a real, checkable gap in the table (not
a guess — `function_costs.find("//")` genuinely returns `end()` on this
checkout), and it means a query using `//` gets a *much* more conservative
(higher) cost estimate for the exact same conceptual operation `/` gets
costed cheaply for.

### 3.4 `BOUND_CASE` and `BOUND_CONJUNCTION`

```cpp
// src/optimizer/expression_heuristics.cpp:79-88
idx_t ExpressionHeuristics::ExpressionCost(const BoundCaseExpression &expr) {
	idx_t case_cost = 0;
	for (auto &case_check : expr.CaseChecks()) {
		case_cost += Cost(*case_check.then_expr);
		case_cost += Cost(*case_check.when_expr);
	}
	case_cost += Cost(expr.Else());
	return case_cost;
}
```

A `CASE` expression's cost is the plain sum of every branch's `WHEN` cost
plus every branch's `THEN` cost, plus the `ELSE` cost — there's no discount
for the fact that, at runtime, most rows only ever actually evaluate a
handful of branches before one matches; the static heuristic assumes the
worst case (every branch, every row) because it has no selectivity
information to do otherwise.

```cpp
// src/optimizer/expression_heuristics.cpp:116-123
idx_t ExpressionHeuristics::ExpressionCost(const BoundConjunctionExpression &expr) {
	idx_t cost = 5;
	for (auto &child : expr.GetChildren()) {
		cost += Cost(*child);
	}
	return cost;
}
```

A nested `AND`/`OR` (one that survived as a single sub-expression rather than
being flattened into the parent's own conjunct list — see §7.5's note that
`ConjunctionSimplificationRule` and DuckDB's general "AND is associative,
flatten it" convention mean a *top-level* `WHERE a AND b AND c` is stored as
three sibling conjuncts, not as nested pairs) costs `5` plus the sum of its
own children — the same flat `+5` "there's a boolean combination happening
here" surcharge the comparison cost model uses.

### 3.5 The full ranking, cheapest to costliest

Collecting every number above into one table — the *exact* real ranking this
heuristic produces, not an approximation of it:

| cost contribution | expression shape |
|---|---|
| `1` | fixed-width constant/parameter |
| `2` | `FLOAT`/`DOUBLE` constant/parameter |
| `5` | `VARCHAR` constant/parameter |
| `8` | fixed-width column/physical reference |
| `+5` | `IS [NOT] NULL` |
| `+5` | `+`, `-`, `&`, `#`, `>>`, `<<`, `abs` |
| `+5` | any comparison (`=`,`<>`,`<`,`<=`,`>`,`>=`) — flat, added between the two operand costs |
| `+5` | a numeric-to-numeric cast where source and target types differ, neither being `VARCHAR`/`BLOB` (a true no-op cast, source type equal to target type, costs `+0`) |
| `+5` | a conjunction's own flat overhead (`AND`/`OR`, on top of its children) |
| `+10` | `NOT` |
| `+10` | `*`, `%` |
| `+10` | `BETWEEN`'s own flat overhead (on top of input+lower+upper) |
| `16` | `FLOAT`/`DOUBLE` column/physical reference |
| `+15` | `/` (true division) |
| `+20` | `date_part`, `year` |
| `+100` | `round` |
| `+100` | each item past the first in an `IN (...)`/`NOT IN (...)` list |
| `+200` | `~~` (`LIKE`), `!~~` (`NOT LIKE`), `regexp_matches`, `||` (string concat) |
| `+200` | any cast touching `VARCHAR` or `BLOB` on either side |
| `40` | `VARCHAR` column/physical reference |
| `+1000` | any scalar function not named in the table above (`sqrt`, `xor`, `//`, `list_kurtosis`, every extension function, …) |
| `+1000` | any `BOUND_OPERATOR` expression type not `IS [NOT] NULL`/`NOT`/`IN`/`NOT IN` |
| `1000` | any expression class the top-level `Cost` switch doesn't recognize at all (`BOUND_AGGREGATE`, `BOUND_WINDOW`, `BOUND_SUBQUERY`, …) |

Every `+N` line adds to whatever the expression's own children already cost,
recursively — the whole thing is a bottom-up sum over the expression tree,
computed fresh, from scratch, every single time `ReorderExpressions` runs
(there's no caching of a sub-expression's cost across sibling conjuncts that
happen to share structure).

## 4. The safety guard: exact code, exact mechanics

`ReorderExpressions` puts the cost model to work, but only after clearing
one gate first:

```cpp
// src/optimizer/expression_heuristics.cpp:37-70
void ExpressionHeuristics::ReorderExpressions(vector<unique_ptr<Expression>> &expressions) {
	struct ExpressionCosts {
		unique_ptr<Expression> expr;
		idx_t cost;

		bool operator==(const ExpressionCosts &p) const {
			return cost == p.cost;
		}
		bool operator<(const ExpressionCosts &p) const {
			return cost < p.cost;
		}
	};

	for (idx_t i = 0; i < expressions.size(); i++) {
		if (expressions[i]->CanThrow() || ExpressionBarrier::Contains(*expressions[i])) {
			// do not allow reordering if an expression can throw, or carries a barrier
			return;
		}
	}

	vector<ExpressionCosts> expression_costs;
	expression_costs.reserve(expressions.size());
	// iterate expressions, get cost for each one
	for (idx_t i = 0; i < expressions.size(); i++) {
		idx_t cost = Cost(*expressions[i]);
		expression_costs.push_back({std::move(expressions[i]), cost});
	}

	// sort by cost and put back in place
	sort(expression_costs.begin(), expression_costs.end());
	for (idx_t i = 0; i < expression_costs.size(); i++) {
		expressions[i] = std::move(expression_costs[i].expr);
	}
}
```

Read the shape of this precisely, because it's easy to over-read what it
does: the loop at lines 50-55 walks **every** conjunct in the list, and the
moment it finds **any one** that `CanThrow()` or carries a barrier, it
`return`s immediately — abandoning the whole function, before a single
`Cost()` call has even happened. This is not "sort everything, but pin the
throwing conjunct in place"; it is not "only refuse to move the throwing
conjunct past others, but still sort the rest of the list around it." It is
a **blanket, all-or-nothing veto**: one throwing (or barrier-carrying)
conjunct anywhere in a filter's conjunct list disables *all* reordering of
*that entire list*, leaving every conjunct — the throwing one and every
conjunct around it — exactly where the plan handed them to this pass. A
`WHERE` clause with five conjuncts, four of them ordinary and cheap and one
of them a division, gets *zero* reordering, not "four reordered, one pinned."

### 4.1 What `CanThrow()` actually checks, and why it's a static, not a data-dependent, property

`Expression::CanThrow()` (`src/planner/expression.cpp:68-71`) is, by
default, a pure structural recursion — no data, no session settings, no
statistics involved at all:

```cpp
// src/planner/expression.cpp:68-71
bool Expression::CanThrow() const {
	bool can_throw = false;
	ExpressionIterator::EnumerateChildren(*this, [&](const Expression &child) { can_throw |= child.CanThrow(); });
	return can_throw;
}
```

Exactly one expression class overrides it, and the override still calls the
base implementation as part of its own answer — so the recursive "or my
children" behavior is preserved everywhere, not just at the leaves:

```cpp
// src/planner/expression/bound_function_expression.cpp:75-78
bool BoundFunctionExpression::CanThrow() const {
	auto lambda_expr = GetLambdaExpression(*this);
	return function.GetErrorMode() == FunctionErrors::CAN_THROW_RUNTIME_ERROR ||
	       (lambda_expr && lambda_expr->CanThrow()) || Expression::CanThrow();
}
```

`function.GetErrorMode()` reads a `FunctionErrors` enum value
(`src/include/duckdb/function/function.hpp:598`, default
`FunctionErrors::CANNOT_ERROR`). The usual way to set it is a one-line
helper, called when the function is *registered*:

```cpp
// src/include/duckdb/function/function.hpp:583-585
auto SetFallible() -> void {
	errors = FunctionErrors::CAN_THROW_RUNTIME_ERROR;
}
```

There is a second way. A function's *bind* callback runs once per call site,
when the query is bound. It may call `SetErrorMode(FunctionErrors::CANNOT_ERROR)`
on that one bound copy of the function. It does this when it can prove that
this call, with these argument types and these settings, cannot throw. So the
rule is:

- **Registration** gives every call of the function a default: fallible or
  not.
- **Binding** may clear the flag for one call, never set it for a function
  that was registered as safe (casts are the exception: the cast binder sets
  the flag only when the chosen cast can fail).

After binding, the flag is fixed for that expression. `CanThrow()` does not
look at data or at the settings again. It only reads the flag.

So `CanThrow()` is not always fixed at registration, and it can depend on
settings. Several functions clear the flag at bind time: the DOUBLE/FLOAT
`/`; DECIMAL `+`, `-` and `*` when their result type is wide enough to never
overflow; `substring` and `repeat`; and `regexp_replace` and
`regexp_extract` with safe constant patterns.

Two operators that this chapter and the test file lean on show the two
cases side by side.

- **Division.** `arithmetic.cpp`'s `"/"` (`OperatorFloatDivideFun`) and
  `"//"` (`OperatorIntegerDivideFun`) function sets both call
  `.SetFallible()` at registration (`arithmetic.cpp:1513`, `:1530`). But
  `"/"`'s bind callback, `BindBinaryFloatingPoint`
  (`arithmetic.cpp:1435-1453`), looks at two settings. It picks the
  implementation, and it also clears the error flag when a division by
  zero cannot raise an error:

  ```cpp
  // src/function/scalar/operator/arithmetic.cpp:1435-1453
  template <class OP>
  unique_ptr<FunctionData> BindBinaryFloatingPoint(BindScalarFunctionInput &input) {
  	auto &context = input.GetClientContext();
  	auto &bound_function = input.GetBoundFunction();

  	const auto ieee_ops = Settings::Get<IeeeFloatingPointOpsSetting>(context);
  	const auto null_on_zero = !Settings::Get<ErrorOnDivisionByZeroSetting>(context);
  	if (ieee_ops) {
  		bound_function.SetFunctionCallback(GetScalarBinaryFunction<OP>(bound_function.GetReturnType().InternalType()));
  	} else {
  		bound_function.SetFunctionCallback(
  		    GetBinaryFunctionZeroCheck<OP>(bound_function.GetReturnType().InternalType(), null_on_zero));
  	}
  	if (std::is_same<OP, DivideOperator>::value && (ieee_ops || null_on_zero)) {
  		// a division by zero yields inf, nan or NULL instead of raising an error
  		bound_function.SetErrorMode(FunctionErrors::CANNOT_ERROR);
  	}
  	return nullptr;
  }
  ```

  `ieee_floating_point_ops` **defaults to `true`**
  (`src/common/settings.json:1074-1078`, *"Use IEEE 754 behavior for
  supported floating point operations, returning NAN/INF instead of
  errors/NULL"*). So under default settings, `10.0 / 0.0` returns `inf`,
  and a bound `/` reports `CanThrow() == false`. The test suite shows
  the `inf`:

  ```sql
  -- test/sql/function/operator/test_division_by_zero.test
  query III
  SELECT 7 / 0, 7.5::DOUBLE / 0, -7.5::FLOAT / 0
  ----
  inf	inf	-inf
  ```

  Note that the implementation and the flag are chosen in the same call,
  from the same two settings. They cannot disagree. If a session runs
  `SET ieee_floating_point_ops = false` before the query is bound, `/` gets
  the zero-checking implementation *and* keeps its `CAN_THROW_RUNTIME_ERROR`
  flag.

  `"//"` (integer division) and `%` (modulo) are different. Their bind
  callbacks never clear the flag, so they are always `CanThrow() == true`.
  Under default settings they also really throw on a zero divisor
  (`error_on_division_by_zero` defaults to `true`,
  `src/common/settings.json:854-859`; that setting is marked
  `"is_deprecated": true`). The same test file confirms this:

  ```sql
  foreach type TINYINT SMALLINT INTEGER BIGINT HUGEINT UTINYINT USMALLINT UINTEGER UBIGINT UHUGEINT DECIMAL(4,1) DECIMAL(9,2) DECIMAL(18,3) DECIMAL(38,4)
  statement error
  SELECT 7::{type} // 0::{type}
  ----
  <REGEX>:.*Division by zero.*
  ```

  An optimizer test that pins these flags is
  `test/optimizer/partial_aggregate_pushdown_throwing_argument.test`. It
  checks, through `EXPLAIN`, that `sum(price / qty)` is still pushed below a
  join under default settings (the division cannot throw), that it is *not*
  pushed after `SET ieee_floating_point_ops = false`, and that
  `sum(sales.id // qty)` is never pushed.

- **`sqrt`.** `extension/core_functions/scalar/math/numeric.cpp:1455-1462`
  registers `sqrt` with `.SetFallible()`. Its bind callback,
  `BindIEEEFloatingUnary` (`numeric.cpp:22-31`), picks between
  `SqrtOperator` (throws `OutOfRangeException("cannot take square root of
  a negative number")`) and `IEEESqrtOperator` (returns `NaN`) based on the
  same `IeeeFloatingPointOpsSetting`. But unlike `/`, it does **not** clear
  the error flag. So a bound `sqrt` is `CanThrow() == true` even under the
  default settings, where it can only return `NaN`.

The design consequence: **the guard in `ReorderExpressions` asks only "is
this bound expression marked as able to throw?"** It never runs the
expression and never reads the settings itself. The flag is as precise as
the function's author made it. For `/` it is exact. For `sqrt` it is
still conservative: the guard blocks reordering even though, with the
default settings, nothing could throw. Being too careful costs a possibly
worse conjunct order. Being too loose would let a query fail that should
succeed. Every function author can choose to be careful, and the guard is
correct either way, as long as the flag never says "cannot throw" for a
call that can.

### 4.2 `ExpressionBarrier` — the second half of the `||`, and why it's there even though it looks redundant

`ExpressionBarrier::Contains` (`src/function/scalar/generic/barrier.cpp:56-63`)
recurses through an expression's entire subtree checking for a call to the
internal `__internal_barrier(...)` function, which secure views (and any
future caller) wrap around a user expression specifically to mark it as
"observable from outside the query — may throw, may have a side effect, must
never be evaluated on a row some other filter would have discarded first":

```cpp
// src/function/scalar/generic/barrier.cpp:49-63
bool ExpressionBarrier::IsBarrier(const Expression &expr) {
	if (expr.GetExpressionClass() != ExpressionClass::BOUND_FUNCTION) {
		return false;
	}
	return expr.Cast<BoundFunctionExpression>().Function().GetName() == BarrierFun::Name;
}

bool ExpressionBarrier::Contains(const Expression &expr) {
	if (IsBarrier(expr)) {
		return true;
	}
	bool contains = false;
	ExpressionIterator::EnumerateChildren(expr, [&](const Expression &child) { contains |= Contains(child); });
	return contains;
}
```

The barrier function itself is *also* marked `SetFallible()`
(`barrier.cpp:39`), with a comment that states the intent directly: *"the
barrier itself never throws, but marking it fallible keeps the expression
out of every optimization that evaluates an expression on more rows than it
would otherwise see (dictionary execution, filter reordering)."* Given that,
a barrier-wrapped conjunct is *already* `CanThrow() == true` on its own
account (it's a `BOUND_FUNCTION` with `GetErrorMode() ==
CAN_THROW_RUNTIME_ERROR`), and `Expression::CanThrow()`'s default recursion
means any *ancestor* expression containing that barrier anywhere in its
subtree inherits `CanThrow() == true` too — which means, mechanically, the
`ExpressionBarrier::Contains(*expressions[i])` half of the `||` in
`ReorderExpressions`'s guard is, today, logically implied by the
`CanThrow()` half alone. The redundancy isn't an oversight to be quietly
corrected; it's exactly what you'd expect from reading the commit history:

- `596bd1d540` ("do not reorder if an expression can throw," Nov 4 2024,
  Tom Ebergen) added the unconditional-veto form of the guard, checking only
  `CanThrow()`.
- `b283b00a54` (Sep 7 2026, Mark Raasveldt — the same commit that introduced
  `__internal_barrier` and marked it fallible) extended the check to
  `CanThrow() || UnsafeBarrier::Contains(...)` (renamed to `ExpressionBarrier`
  a few hours later, same day, in `f4d061d48c`) — belt-and-suspenders,
  written by the same author who was simultaneously wiring `SetFallible()`
  onto the barrier function itself. Given that both changes landed together,
  the explicit `Contains()` check reads as deliberate defense-in-depth
  against the barrier's own fallible-marking ever changing in the future,
  rather than a gap the fallible-marking was meant to close alone.

### 4.3 A short history: from a soft penalty to a hard veto

The guard wasn't always an unconditional early-return. `git log` on
`expression_heuristics.cpp` shows the very first attempt at handling
throwing conjuncts, three days before the version reproduced above, added a
**cost penalty** instead of a veto:

```cpp
// ce465cc3dc, Nov 1 2024, Tom Ebergen — "adaptive filters should not reorder filters that can throw"
idx_t cost = Cost(*expressions[i]);
if (expressions[i]->CanThrow()) {
	cost += ExpressionHeuristics::CAN_THROW_PENALTY;   // = 10000
}
expression_costs.push_back({std::move(expressions[i]), cost});
```

This still *sorted* every conjunct — it just added a `10000` handicap to any
throwing one, on the theory that a big enough penalty would, in practice,
always push it to the back of the list. Three days later, `596bd1d540`
replaced this with the unconditional early-return form quoted in §4 above,
discarding the penalty constant entirely. The commit message doesn't
elaborate, but the shape of the fix says what the problem was: a penalty
that merely biases a *sort* isn't a guarantee, because sorting is a
comparison between *all* the costs in the list, and a sufficiently
pathological set of costs among the *other* conjuncts could, in principle,
still produce an order where the throwing conjunct isn't last relative to
the specific guard that was protecting it (a large penalty on the throwing
conjunct doesn't stop some *other* conjunct in the same list from being
penalized past it too, if that other conjunct is *also* expensive by this
heuristic's reckoning — a cost-based bias is not the same claim as a
structural guarantee). The unconditional veto is a strictly stronger, simpler
claim — *don't reorder at all* — and it's the one that survived to this
checkout. `AdaptiveFilter` went through the mirror-image version of this same
hardening at almost the same time, covered in §6 below.

## 5. Two worked examples, with the exact numbers

### 5.1 A case where cost-only sorting would have gotten it right anyway

Take the classic shape the hazard is usually described with, adjusted to use
an operator that actually throws under this checkout's default settings
(§4.1 already established plain `/` does not):

```sql
WHERE x <> 0 AND 10 // x > 1
```

Using §3's table by hand: `x <> 0` is a comparison, `Cost(x) + 5 + Cost(0)
= 8 + 5 + 1 = 14` (assuming `x` is a fixed-width integer column). `10 // x >
1` is also a comparison: `Cost(10 // x) + 5 + Cost(1)`. `10 // x` is a
`BOUND_FUNCTION` named `"//"` — and, per §3.3's gap, `"//"` is **not** in the
named-function table, so it falls to the `+1000` unknown-function default:
`Cost(10) + Cost(x) + 1000 = 1 + 8 + 1000 = 1009`. The whole comparison costs
`1009 + 5 + 1 = 1015`. `14 < 1015`, so a pure, guard-free cost sort would
already rank `x <> 0` first — the safe order — purely because `"//"`
happens to fall into this heuristic's most expensive bucket. In this
specific instance, the guard and the cost model would have agreed even
without each other.

### 5.2 A case where they don't — the actual danger

That agreement is a coincidence of these particular numbers, not something
`ReorderExpressions` checks for or relies on — and it's easy to construct a
conjunct pair, still using only real entries from §3's table, where cost-only
sorting would get it backwards. Swap the guard for a shape that's logically
equivalent but syntactically more expensive — comparing against a string,
the way a query built by loosely-typed application code sometimes ends up
doing:

```sql
WHERE CAST(x AS VARCHAR) <> '0' AND 10 // x > 1
```

`CAST(x AS VARCHAR) <> '0'`: the cast (`CastExpressionCost`) touches
`VARCHAR`, so it costs `Cost(x) + 200 = 8 + 200 = 208`. Wrapped in the outer
comparison: `208 + 5 + Cost('0')` — `'0'` is a `VARCHAR` constant, cost `5`
— total `208 + 5 + 5 = 218`. The division conjunct is unchanged from §5.1:
`1015`. Cost-only sorting would still rank the cast-guard (`218`) *before*
the division (`1015`) here too, since `218 < 1015` — the `+1000`
unknown-function bucket for `"//"` is simply large enough to swamp most
plausible guard costs on its own. To actually invert it, drop the throwing
conjunct back down to one this heuristic *does* recognize cheaply — plain
`/`, cost `15` in the table. For the danger to be real, `/` must also be
able to throw. Per §4.1, that is only the case when the session has turned
IEEE semantics off before the query is bound:

```sql
SET ieee_floating_point_ops = false;
-- ...
WHERE CAST(x AS VARCHAR) <> '0' AND 10 / x > 1
```

Now `10 / x > 1` costs `Cost(10 / x) + 5 + Cost(1)`, and `"/"` *is* in the
table at `15`: `Cost(10 / x) = 1 + 8 + 15 = 24`; the whole comparison costs
`24 + 5 + 1 = 30`. (This ignores the implicit cast of the integer `x` to
`DOUBLE` that the binder adds, because `/` only has `FLOAT`/`DOUBLE`
overloads. The cast adds `+5`, and the constants become `DOUBLE` constants
at cost `2`, giving `37`. The order below does not change.) The guard still
costs `218`. **`30 < 218`** — a pure,
guard-free cost sort would rank the division *first*, ahead of the guard
that exists specifically to keep it from seeing an `x = 0` row. This is the
textbook hazard, reconstructed with DuckDB's real, checkable cost constants:
a syntactically "simple-looking" arithmetic expression that this heuristic
happens to cost cheaply, sitting in front of a syntactically "complicated"
guard this heuristic costs expensively, purely because of what it costs
*strings*, not what it costs *risk*.

Under the *default* settings, though, the veto does not protect this
query, and it does not need to. A `/` bound with
`ieee_floating_point_ops = true` reports `CanThrow() == false` (§4.1).
Neither conjunct can throw, so `ReorderExpressions` sorts them and the
division (`30`) goes first. That is safe: for a row with `x = 0`,
`10 / x` is `inf`, `inf > 1` is true, and the guard still removes the
row. The veto described next only matters with `ieee_floating_point_ops`
set to `false` (or with an operator like `//`, which always throws).

The important conclusion isn't "therefore always check your guards' relative
cost" — it's the opposite. `ReorderExpressions`'s guard loop (§4) never
computes or compares a single one of these numbers when a throwing conjunct
is present; it returns at line 53, before `Cost()` is called even once. §5.1
and the throwing form of §5.2 differ in how the guard is spelled and in
which division they use, and the guard behaves identically in both — full veto,
list untouched — regardless of which specific numeric relationship happens
to hold between the two conjuncts' costs in this particular query. That's
exactly the point of a categorical guard over a cost-biased one: it doesn't
need the numbers in §5.1 and §5.2 to already agree; it makes the question
"do they agree" irrelevant.

## 6. `functions_that_error_are_not_reordered.test`, walked in full

The file's own name is the most direct possible description of what it's
for, and it's built as six independent scenarios (the last two are in
§6.1), each pairing a cheap
guard against a conjunct that would throw for some rows if evaluated first:

```sql
# name: test/optimizer/expression_rewriter/functions_that_error_are_not_reordered.test
# description: Functions can error
# group: [expression_rewriter]
```

**Block 1 — `CASE` guarding `sqrt`:**

```sql
statement ok
create table t1 as from range(-5000, 5000, 1) t1(a);

# case expression must happen first, otherwise sqrt will throw an error
query I
select count(*)
from t1
where
	case
	when a >= 0 then true
	When a > 5 then false
	when a > 10 then true
	when a < 0 then false
	when a > 15 then true
	else false END
and sqrt(a) >= 0;
----
5000
```

`t1.a` ranges over `[-5000, 5000)`, ten thousand rows, half negative. The
`CASE` is a slightly convoluted (deliberately so — five branches, not one) way
of computing "is `a` non-negative," and it's `AND`-ed with `sqrt(a) >= 0`. On
this checkout, under this test's own (default) settings, `sqrt` binds to
`IEEESqrtOperator` (§4.1) — negative input returns `NaN`, not a thrown error
— which is a genuinely important qualifier this book's sourcing policy
requires stating plainly: **the inline comment's claim that "sqrt will throw
an error" if reordered is not, in fact, what would observably happen under
this checkout's default `ieee_floating_point_ops=true` setting.** `NaN`
doesn't error here; it flows through the comparison, and — checked directly
against a DuckDB build — `NaN >= 0` evaluates to `true`, following the same
"NaN sorts/compares as the largest value" total-ordering convention
Postgres-lineage SQL engines commonly use for floating types, not IEEE 754's
own "any comparison against NaN is false" rule. Under that convention, even a
*fully reordered* version of this query (division first) wouldn't actually
observably diverge in its returned count under default settings, because
`sqrt(a) >= 0` would be `true` for every row regardless of `a`'s sign.

This isn't a flaw in the test, and it's the single clearest illustration in
this whole chapter of exactly why the guard is written the way §4.1
describes: `sqrt` is registered `SetFallible()` unconditionally, so
`ReorderExpressions` refuses to reorder this list under *every* settings
combination, including the one — `SET ieee_floating_point_ops=false` — under
which `sqrt(-1)` really does throw `OutOfRangeException("cannot take square
root of a negative number")` and this test's own comment becomes literally,
observably true. The guard doesn't get to assume "well, this session
probably has the default settings" — it protects the plan against the
settings the *next* execution might run under, not just the one the plan was
optimized under.

**Block 2 — `len(a) = len(b)` guarding `xor` on `BITSTRING`:**

```sql
statement ok
CREATE TABLE t2 AS
SELECT
	list_reduce([floor((random() * 2))::INT::VARCHAR for t, i in range(floor((random() * 20))::INT + 1)], lambda x, y: concat(x, y)) a,
	list_reduce([floor((random() * 2))::INT::VARCHAR for t, i in range(floor((random() * 20))::INT + 1)], lambda x, y: concat(x, y)) b
FROM range(10000);

# len(a) = len(b) must happen first
# otherwise xor function will throw an error
statement ok
select * from t2 where len(a) = len(b) and ((xor(a::BITSTRING, b::BITSTRING))::VARCHAR)[0] = '0';
```

Each row's `a`/`b` is a randomly-generated string of `'0'`/`'1'` characters
with an *independently* random length (`0` to `19` characters each) — so, by
construction, most rows have `len(a) != len(b)`. This is the one block whose
throw is genuinely unconditional, with no settings escape hatch anywhere:
`Bit::BitwiseXor` (`src/common/types/bit.cpp:480-483`) checks the two
bitstrings' lengths directly and throws unconditionally if they differ:

```cpp
// src/common/types/bit.cpp:480-483
void Bit::BitwiseXor(const bitstring_t &rhs, const bitstring_t &lhs, bitstring_t &result) {
	if (Bit::BitLength(lhs) != Bit::BitLength(rhs)) {
		throw InvalidInputException("Cannot XOR bit strings of different sizes");
	}
	// ...
}
```

There is no setting anywhere in DuckDB that relaxes this into a `NULL` or a
`NaN`-style silent value — every mismatched-length row genuinely, always
throws `InvalidInputException("Cannot XOR bit strings of different
sizes")`, in every session. `len(a) = len(b)` is the only thing standing
between this query and an error on (statistically) the overwhelming majority
of its ten thousand rows, and it is a `statement ok` for exactly that
reason — the assertion here isn't a row count, it's simply that the
statement *doesn't throw*, which it wouldn't if `xor(...)` were ever allowed
to run on a length-mismatched pair before the length-guard excluded it.

**Block 3 — `CASE` guarding an `INT` overflow in `(a * b)::INT`:**

```sql
statement ok
create table t3 (a INT, b INT);
statement ok
insert into t3 (select -2147483645, 2147483647 from range(500000));
statement ok
insert into t3 (select range, range from range(32700));

query I
select count(*) from t3
where
	case
	when (a < 0 and a > -3000) then false
	when (a > -2147483645 and a < 0) then true
	when (a >= 0 and a < 32700) then true
	when a < 3270000 then false
	when a < 2147483645 then true
	else false END
and (a * b)::INT < 2147483648;
----
32700
```

500,000 rows carry `a = -2147483645, b = 2147483647` — near the extremes of
`INT32`'s range, chosen so that `a * b` overflows `INT32` (and, per
`arithmetic.cpp`'s `MultiplyOperatorOverflowCheck`/`.SetFallible()` on the
integer multiply overload, throws an overflow error rather than wrapping)
long before the `::INT` cast at the end is even reached. 32,700 rows carry
small, mutually-safe `a = b = 0..32699` values, for which `a * b` (at most
`32699² ≈ 1.07 × 10⁹`, comfortably inside `INT32`'s ~2.1 × 10⁹ ceiling) never
overflows. The `CASE`'s five branches are a deliberately roundabout way of
computing "is `a` one of the small, safe values" — and the query's answer,
`32700`, is exactly the count of safe rows, confirming the overflow-prone
`(a * b)::INT` comparison genuinely only ever runs against the rows the
`CASE` has already let through.

**Block 4 — `a[1] < 1001` guarding an overflow inside `list_kurtosis`:**

```sql
statement ok
create table t4 as select [2e304, 2e305, 2e306, 2e307] a from range(10000);
statement ok
insert into t4 select [range, range +1, range + 2, 2e50] from range(1000);

require vector_size 2048

query I
select count(*) from t4 where a[1] < 1001 and list_kurtosis(a) is NOT NULL;
----
1000
```

`list_kurtosis` (`src/catalog/default/default_functions.cpp:191`, a default
SQL macro expanding to `list_aggr(l, 'kurtosis')`) computes kurtosis over a
list's elements, which involves raising each element's deviation from the
mean to the fourth power. The 10,000 rows built from `2e304`..`2e307` carry
magnitudes whose fourth powers are far beyond even a `DOUBLE`'s ~`1.8 ×
10³⁰⁸` representable range; the 1,000 rows built from small integers plus a
`2e50` outlier stay comfortably inside it (`(2e50)⁴ = 1.6 × 10²⁰¹`, still
finite). `a[1] < 1001` is a cheap, single-element index check that
distinguishes the two batches by their *first* list element alone — `2e304 >
1001`, so every giant-magnitude row is excluded, and only the 1,000
small-magnitude rows reach `list_kurtosis` at all. The result count, `1000`,
confirms exactly that split held. (The `require vector_size 2048` directive
and its own `# FIXME: this should not be required` comment are a harness
quirk unrelated to the reordering hazard this test exists to check — they
constrain which build configuration the test is allowed to run under, not
anything about conjunct order.)

### 6.1 Other passes must keep the order too

Blocks 5 and 6 are not about `REORDER_FILTER`'s own sort. They check two
*other* places that rebuild a conjunct list before `REORDER_FILTER` ever
runs.

The veto in §4 only stops `ReorderExpressions` from sorting. It cannot help
if an earlier pass has already put the throwing conjunct first. By the time
pass #40 runs, a reversed list looks exactly like a list the user wrote in
that order, and the veto then *keeps* the wrong order. So every pass that
rebuilds a conjunct list must keep the written order.

Both blocks use `get_bit(bitstring, index)`. It is registered with
`.SetFallible()` (`extension/core_functions/scalar/bit/bitstring.cpp:103-109`)
and throws when the index is past the end of the bit string:

```cpp
// extension/core_functions/scalar/bit/bitstring.cpp:95-96
throw OutOfRangeException("bit index %s out of valid range (0..%s)", NumericHelper::ToString(n),
                          NumericHelper::ToString(Bit::BitLength(input) - 1));
```

`'1'::BIT` has one bit, so only index `0` is valid.

**Block 5 — a guard on one side of a join, `b < 1` before `get_bit`:**

```sql
# b < 1 must happen first, also when join ordering rebuilds the filters, otherwise get_bit will throw an error
query II
SELECT * FROM (VALUES (0)) x(a), (VALUES (2)) y(b) WHERE b < 1 AND get_bit('1'::BIT, b) = 1;
----

query II
SELECT * FROM (VALUES (0)) x(a), (VALUES (0), (2)) y(b) WHERE b < 1 AND get_bit('1'::BIT, b) = 1;
----
0	0
```

With `b = 2`, `get_bit` would throw, so `b < 1` must run first. The first
query returns no rows; the second returns the one safe row. The test
guards against a bug (GitHub issue #26098) in which the order was lost. Here
is how it could go wrong, step by step:

1. `FILTER_PUSHDOWN` pushes the safe `b < 1` down. It keeps the throwing
   `get_bit(...)` filter above a projection, because a throwing filter must
   not move past operators that remove rows.
2. `PROJECTION_PULLUP` removes that projection. Now the two filters sit
   directly on top of each other: `get_bit` above, `b < 1` below.
3. `JOIN_ORDER` collects stacked filters top-down and later re-applies them
   in that order. So the lowest filter, the one evaluated first, would end
   up last.

To prevent this, `JOIN_ORDER` reverses each collected chain of stacked
filters:

```cpp
// src/optimizer/join_order/relation_manager.cpp:434-435
// stacked filters are re-applied in this order: keep the lower (earlier evaluated) filters first
std::reverse(filter_operators.begin() + NumericCast<int64_t>(filter_chain_start), filter_operators.end());
```

`filter_chain_start` (`relation_manager.cpp:421`) remembers where this
chain began, so only the filters collected in this walk are reversed. A
related guard in `FilterCombiner::GenerateEquivalentFilters`
(`src/optimizer/filter_combiner.cpp:154-156`) stops it from copying a filter
onto the same column it came from, which would duplicate filters.

**Block 6 — a guard on both sides of a join, `a + b < 1` before `get_bit`:**

```sql
# a + b < 1 must happen first when both conjuncts become a single join condition, otherwise get_bit will throw an error
query II
SELECT * FROM (VALUES (0)) x(a), (VALUES (1)) y(b) WHERE a + b < 1 AND get_bit('1'::BIT, a + b) = 1;
----

query II
SELECT * FROM (VALUES (0)) x(a) JOIN (VALUES (1)) y(b) ON a + b < 1 AND get_bit('1'::BIT, a + b) = 1;
----

query II
SELECT * FROM (VALUES (0)) x(a), (VALUES (0), (1)) y(b) WHERE a + b < 1 AND get_bit('1'::BIT, a + b) = 1;
----
0	0

# written in the opposite order, get_bit happens first and throws
statement error
SELECT * FROM (VALUES (0)) x(a), (VALUES (1)) y(b) WHERE get_bit('1'::BIT, a + b) = 1 AND a + b < 1;
----
bit index 1 out of valid range
```

Both conjuncts read `a` *and* `b`, so neither can be pushed into one side.
Both become one join condition. `JoinCondition::CreateExpression` folds a
list of such conditions into one nested `AND`. It builds
`AND(previous, new)`, so the earlier conjunct stays first. Building
`AND(new, previous)` would put the later conjunct first (GitHub issue
#28628):

```cpp
// src/planner/joinside.cpp:40-53
unique_ptr<Expression> JoinCondition::CreateExpression(vector<JoinCondition> conditions) {
	unique_ptr<Expression> result;
	for (auto &cond : conditions) {
		auto expr = CreateExpression(std::move(cond));
		if (!result) {
			result = std::move(expr);
		} else {
			auto conj = make_uniq<BoundConjunctionExpression>(ExpressionType::CONJUNCTION_AND, std::move(result),
			                                                  std::move(expr));
			result = std::move(conj);
		}
	}
	return result;
}
```

The last query in the block is the control. Written with `get_bit` first,
the query must fail with `bit index 1 out of valid range`. This shows that
DuckDB does keep the *written* order, and that the written order is what
protects (or fails to protect) the user. DuckDB does not try to find a safe
order on its own.

Taken together, the six blocks state one rule: when any conjunct can throw,
the order the user wrote is the order that runs. `REORDER_FILTER`'s veto
enforces this for its own sort, and every other pass that rebuilds a filter
or a join condition must keep it too.

## 7. What else is (and isn't) in the test suite

A broad search of `test/` for anything else naming this pass or its
mechanism (`REORDER_FILTER`, `reorder_filter`, `ExpressionHeuristics`) turns
up nothing beyond the file just walked — there is no `EXPLAIN`-based test
anywhere in the suite that asserts a specific before/after conjunct order the
way, say, `07-filter-pullup-and-pushdown.md`'s sibling deep-dive file can
point to same-label `query II nosort <label>` pairs proving a rewrite fired.
That's a real, checkable absence, not an oversight in this search: this pass
sorts an *already-flat* list of conjuncts by a scalar cost, which doesn't
produce a structurally different plan shape `EXPLAIN` would show differently
— only a different *order* within one operator's expression list, which
`EXPLAIN`'s printed output doesn't distinguish (`a AND b` and `b AND a` print
identically). The only way to observe this pass's effect at all is
indirectly, through whether a query that *should* throw does or doesn't —
which is exactly the shape every block of
`functions_that_error_are_not_reordered.test` takes, and exactly why that
file is the entirety of this pass's direct test coverage.

Two other files surfaced by a broader search are worth a one-line note each,
since both were touched by the same commits that hardened this guard, even
though neither is really *about* `REORDER_FILTER` specifically:
`test/sql/optimizer/test_no_pushdown_cast_into_cte.test` was added alongside
the softer, since-superseded `CAN_THROW_PENALTY` commit (`ce465cc3dc`), but
its own assertions are about filter-vs-projection ordering around a CTE's
cast (`FILTER_PUSHDOWN`'s concern, §7.6.3 of the overview), not about
reordering `CanThrow` conjuncts — it happened to land in the same commit,
not because it exercises this mechanism. `test/sql/function/operator/test_division_by_zero.test`
is the ground truth this file leaned on throughout §4.1 and §5 for exactly
which division operators throw under which settings, but it's a
function-correctness test, not an optimizer test — it never runs `EXPLAIN`
and says nothing about conjunct order at all.

A third file is
`test/optimizer/partial_aggregate_pushdown_throwing_argument.test` (§4.1).
It is not about conjunct order either. It is the clearest test of what
`CanThrow()` reports for `/`, `//`, casts and DECIMAL arithmetic under
different settings, and so of which conjuncts `REORDER_FILTER` will and
will not sort.

## 8. `AdaptiveFilterState`/`AdaptiveFilter` (Chapter 12, §12.12) — the same hazard, a completely different mechanism

Chapter 12 documents `AdaptiveFilter` as the runtime counterpart to this
pass: given a fixed set of conjuncts, it observes how long recent batches
took under the current trial order, and periodically proposes swapping an
adjacent pair, keeping the swap if it measurably helped and reverting it
(with reduced future likelihood) if it didn't — a small randomized
hill-climb over conjunct order, driven entirely by wall-clock timing
observed on this query's actual data. It is worth stating the contrast with
`REORDER_FILTER` in the most literal terms possible, because "reorders
conjuncts by some notion of cost" is genuinely true of both, and everything
interesting is in what differs underneath that shared description:

| | `REORDER_FILTER` (this file) | `AdaptiveFilter` (Ch. 12 §12.12) |
|---|---|---|
| **When it runs** | once, at plan-optimization time (pass #40 of ~45, before physical planning) | continuously, at execution time, re-evaluated every `execute_interval`/`observe_interval` batch of chunks, for as long as the query runs |
| **What it knows** | only the expression tree's static shape and type — no row has been read yet | only wall-clock elapsed time actually measured over this query's actual data, this run |
| **What it optimizes over** | a plan-level `vector<unique_ptr<Expression>>` — the actual expression objects, physically reordered in place | a `vector<idx_t> permutation` — an index array, layered on top of a list whose objects never move |
| **Cost source** | `ExpressionHeuristics::Cost`, a fixed table over expression/function shape (§3) | `TimePoint`-measured elapsed seconds per trial batch (`AdaptRuntimeStatistics`) |
| **Result if it's wrong** | a query that was always going to be slow in the same way, every run, on every dataset | self-correcting — a bad guess gets reverted (with reduced future likelihood) within a handful of batches |
| **Safety mechanism's shape** | a blanket early-return in `ReorderExpressions`: *any* conjunct `CanThrow()`/carries a barrier ⇒ leave the entire list untouched | a single boolean, `disable_permutations`, set `true` if *any* conjunct/table-filter `CanThrow()`/carries a barrier ⇒ `BeginFilter`/`EndFilter` skip all adaptation for the entire query |

### 8.1 `AdaptiveFilter`'s own guard, in full, on this checkout

Chapter 12 quotes `AdaptRuntimeStatistics`; it's worth showing the guard
that gates whether that function is ever allowed to touch the permutation at
all, because — read against the history in §4.3 — it is the mirror image of
`REORDER_FILTER`'s own hardening, built by the same two authors within the
same week in November 2024:

```cpp
// src/execution/adaptive_filter.cpp:17-30 (the BoundConjunctionExpression constructor)
AdaptiveFilter::AdaptiveFilter(const Expression &expr) : observe_interval(10), execute_interval(20), warmup(true) {
	auto &conj_expr = expr.Cast<BoundConjunctionExpression>();
	D_ASSERT(conj_expr.GetChildren().size() > 1);
	for (idx_t idx = 0; idx < conj_expr.GetChildren().size(); idx++) {
		permutation.push_back(idx);
		if (conj_expr.GetChildren()[idx]->CanThrow() || ExpressionBarrier::Contains(*conj_expr.GetChildren()[idx])) {
			disable_permutations = true;
		}
		// ...
	}
}
```

```cpp
// src/execution/adaptive_filter.cpp:99-114
AdaptiveFilterState AdaptiveFilter::BeginFilter() const {
	if (permutation.size() <= 1 || disable_permutations) {
		return AdaptiveFilterState();
	}
	AdaptiveFilterState state;
	state.monotonic_start = TimePoint::Tick();
	return state;
}

void AdaptiveFilter::EndFilter(AdaptiveFilterState state) {
	if (permutation.size() <= 1 || disable_permutations) {
		// nothing to permute
		return;
	}
	AdaptRuntimeStatistics(state.monotonic_start.ElapsedSeconds());
}
```

Just like `ReorderExpressions`, this is a blanket, all-or-nothing veto, not a
per-pair "pin this one conjunct" mechanism: `disable_permutations` is a
single flag checked once per filter-evaluation call, and if it's set, the
entire adaptive machinery — timing, swap proposals, the random hill-climb,
all of it — is skipped for the *whole* conjunct list, for the *entire*
query's execution, not just for swaps that would specifically involve the
throwing conjunct. `AdaptiveFilter`'s constructor for the *other* case —
seeding from a `LogicalGet`'s `TableFilterSet` rather than a
`BoundConjunctionExpression` — runs the identical check per table filter
(`adaptive_filter.cpp:36-40`), with an inline comment that states the
invariant as plainly as `ReorderExpressions`'s own: *"a filter that can throw
must keep the order the optimizer put it in."*

This is, concretely, **the same bug fixed twice, independently, in two
different mechanisms, three days apart**: `ce465cc3dc` (Nov 1 2024) first
patched `AdaptiveFilter`'s constructor with a *soft* fix — permanently
zeroing the `swap_likeliness` for any adjacent pair touching a throwing
conjunct, rather than a global flag — while the *same commit* also added
`REORDER_FILTER`'s first, equally soft, `CAN_THROW_PENALTY` cost-bias fix.
`596bd1d540` (Nov 4 2024) hardened `REORDER_FILTER`'s side into the
unconditional early-return quoted in §4. The `disable_permutations` flag
shown above is `AdaptiveFilter`'s own later hardening into the equivalent
unconditional form — both mechanisms converged, independently, on "when in
doubt, disable the whole thing," rather than either one trying to reason
precisely about *which* specific swaps are safe and which aren't. That
convergence on the same conservative shape, arrived at separately for two
architecturally unrelated systems, is strong evidence that "leave the entire
list alone the moment anything in it can throw" isn't an accident of this
particular commit's code style — it's the only shape of fix either team
trusted enough to ship, for either mechanism.

### 8.2 The bridge: how the static pass's output becomes the dynamic mechanism's starting point

The two mechanisms aren't fully independent of each other, and the exact
nature of the connection is worth being precise about, because it differs
between the two places `AdaptiveFilter` gets constructed.

**For a `LogicalFilter`'s own conjuncts** (the `BoundConjunctionExpression`
constructor, `adaptive_filter.cpp:17-30`): the initial `permutation` is
simply the identity order, `0, 1, 2, …` — no cost is computed at all at this
point. That's not laziness; it's because there's nothing left to compute.
`PhysicalFilter`'s constructor (`src/execution/operator/filter/physical_filter.cpp:7-11`)
builds its `BoundConjunctionExpression` directly from the `LogicalFilter`'s
own `select_list`, i.e. `op.expressions` — the exact vector `ReorderExpressions`
already sorted by cost back at plan-optimization time (or, if any conjunct
could throw, left in the plan's original order — §4). The constructor is
a one-liner around a shared helper:

```cpp
// src/execution/operator/filter/physical_filter.cpp:7-11
PhysicalFilter::PhysicalFilter(PhysicalPlan &physical_plan, vector<LogicalType> types,
                               vector<unique_ptr<Expression>> select_list, idx_t estimated_cardinality)
    : CachingPhysicalOperator(physical_plan, PhysicalOperatorType::FILTER, std::move(types), estimated_cardinality),
      expression(BoundConjunctionExpression::Create(ExpressionType::CONJUNCTION_AND, std::move(select_list))) {
}
```

`BoundConjunctionExpression::Create`
(`src/planner/expression/bound_conjunction_expression.cpp:18-27`) returns
the single child as-is when the list has one entry, and otherwise wraps the
list, in order, in one `AND`. By the time
`AdaptiveFilter`'s identity permutation is constructed at execution time,
`REORDER_FILTER`'s static ordering (or its refusal to reorder) is already
baked into the list's physical order; `AdaptiveFilter` starts from "the order
the optimizer already decided" and perturbs it from there.

**For a `LogicalGet`'s `table_filters`**, the relationship is structurally
different, because table filters never pass through `ReorderExpressions` at
all — that function only ever touches a `LogicalFilter`'s `expressions` list
or a `BoundConjunctionExpression`'s children, and a `TableFilterSet` is
neither of those; it's a separate, per-column filter structure that
`FILTER_PUSHDOWN` (#6) builds when a predicate lands on a scan (§7.6.3 of the
overview). Instead, `ScanFilterInfo::Initialize`
(`src/storage/table/scan_state.cpp:86-94`) constructs its `AdaptiveFilter`
using the *other* constructor, which calls
`ExpressionHeuristics::GetInitialOrder` directly:

```cpp
// src/storage/table/scan_state.cpp:91-94
if (filters.HasFilters()) {
	adaptive_filter = make_uniq<AdaptiveFilter>(filters);
	adaptive_filter->SetLogger(context.logger);
}
```

```cpp
// src/optimizer/expression_heuristics.cpp:256-284
vector<idx_t> ExpressionHeuristics::GetInitialOrder(const TableFilterSet &table_filters) {
	// ... costs each entry via Cost(const TableFilter&), sorts, returns the permutation
}
```

`Cost(const TableFilter &filter)` (`expression_heuristics.cpp:230-254`) is a
small wrapper around the same `Cost(const Expression&)` this whole chapter
has been reproducing, with four special cases worth naming because they
reach forward into machinery this book covers elsewhere: a barrier-carrying
table filter is costed at `NumericLimits<idx_t>::Maximum()` (always sorts
absolute last — the comment states the reason directly: *"a barrier filter
is always evaluated last, so that it never runs on rows another filter
removes"*); a `DynamicFilterScalarFun`-wrapped filter (the min/max runtime
range check `JOIN_FILTER_PUSHDOWN`, pass #42, arranges for a hash join's
build side to hand to the probe side's scan — §7.6.5 of the overview) is
costed at a flat `0`, sorting first, since it's a cheap range check that's
usually highly selective; an `OptionalFilterScalarFun`-wrapped filter is
likewise costed `0`; and a `SelectivityOptionalFilterScalarFun`-wrapped
filter is costed by its wrapped child (found with
`ExpressionFilter::GetOptionalFilterChild`), or `0` if it has none. So: **table-level scan filters get their own,
freshly-computed static initial order, calculated by exactly the same
`Cost()` function `REORDER_FILTER` uses for `LogicalFilter` conjuncts, but
computed separately, at execution-state construction time, not by the
`REORDER_FILTER` pass itself.** The two code paths share a cost *function*;
they don't share a single pass invocation.

### 8.3 Why DuckDB has both

Put the two halves of §8.2 together and the design answer falls out
directly. A static, syntax-only heuristic (§3's table) is cheap to compute
(it runs once, over an expression tree, with no I/O and no waiting for a
single row to be scanned) and gives every query — even a query that will
only ever execute once, on a table nobody has profiled — *some* starting
order that isn't simply "whatever order the `WHERE` clause happened to be
typed in." But it is necessarily blind to the one thing that actually
determines how much a conjunct is worth evaluating first: how often it's
actually true. A `LIKE '%x%'` predicate that rejects 99% of a table's rows
and one that rejects 1% of it cost *identically* under §3's table (both are
`~~`, flat `200`, regardless of pattern or data) — the static model has no
way to represent that difference, because representing it would require
knowing something about the data, which doesn't exist yet at plan time.
`AdaptiveFilter` exists precisely to fill that gap, and it can only fill it
*after* rows have actually been scanned and a candidate order's real
selectivity and real per-conjunct cost can be timed directly — which is
also exactly why it has to run continuously rather than once: the
selectivity that was true for the first million rows of a table isn't
guaranteed to hold for the next million (a table sorted by insertion date,
say, where an early filter's selectivity genuinely drifts partway through a
scan), and a mechanism that only adapted once, early, and then stopped would
be exposed to exactly that drift the same way a one-shot static heuristic
is exposed to data it never saw at all.

Both mechanisms needed the identical fix to the identical hazard — a
conjunct capable of throwing must never be reordered past whatever guard was
protecting it, full stop, regardless of what either mechanism's own cost
signal says — and both, independently, arrived at the same conservative
shape for that fix: an unconditional, whole-list veto rather than a
surgical, per-pair one. That the fix had to be written twice, in two
unrelated pieces of machinery, three days apart, by people working on two
different problems (one on a plan-level optimizer pass, one on an
execution-time adaptivity mechanism) is itself the clearest evidence this
chapter can offer that `REORDER_FILTER` and `AdaptiveFilter` really are two
separate systems that happen to be pointed at a similar-looking problem —
not one mechanism the source tree merely describes twice.
