# Chapter 4 — Binding & Name Resolution

## 4.1 What "binding" means here, precisely

The word "binding" gets used loosely across compilers and databases, so it's
worth pinning down what it means in this specific spot of DuckDB's pipeline,
because Chapter 3 spent an entire chapter establishing what the parser
deliberately does *not* do, and this chapter is the direct payoff of that
setup.

The parser (Chapter 3) is purely syntactic. Given `SELECT o.custkey FROM
orders o`, it produces a `ColumnRefExpression` holding the strings `"o"` and
`"custkey"`, and a `BaseTableRef` holding the string `"orders"` and the alias
`"o"`. At no point does the parser open the catalog, check whether a table
named `orders` exists, check whether it has a column named `custkey`, or
check whether `custkey` is a number or a string. It can't — the PEG grammar in
`src/parser/peg/` has no `Catalog&` in scope at all, by design, so that
parsing stays a pure, catalog-independent, embarrassingly-parallelizable
function of query text alone.

Binding is the stage that closes that gap. Concretely, binding is three
things happening together, all driven by walking the parsed AST:

1. **Name resolution.** Every free-floating name in the parsed tree — a table
   name, a column name, a function name — gets looked up and turned into a
   reference to something concrete: a `TableCatalogEntry`, a specific column
   of a specific table in scope, a specific overload of a specific function in
   the catalog. "Does `orders` exist?" and "which of the two tables in scope
   has a column called `custkey`?" are both answered here, for the first
   time.
2. **Type resolution and inference.** Every expression gets a `LogicalType`.
   A column reference gets the declared type of the catalog column it
   resolved to; a function call gets the return type its resolved overload
   promises; a `CASE` expression gets the common supertype of its branches.
   None of this exists in the parser's output — a `ColumnRefExpression` simply
   has no `LogicalType` field to put it in.
3. **Implicit cast insertion.** Where two expressions need to interoperate
   (the two sides of a comparison, the arguments of a function call, the
   branches of a `UNION`) but arrive with different types, the binder inserts
   an explicit `BoundCastExpression` node to make them agree, using rules
   about which casts are allowed to happen "for free" versus which require an
   explicit `CAST(...)` from the user.

None of this is a separate "semantic analysis" pass bolted onto a mostly-built
plan, either — as section 4.5 covers in depth, DuckDB's binder builds the
logical plan directly, as a byproduct of doing all three of the above things.
By the time this chapter ends, "binding" and "constructing the initial
`LogicalOperator` tree" will turn out to be the same activity described from
two different angles.

## 4.2 Scoping: one `Binder` per lexical scope

SQL scoping is recursive: a subquery in the `WHERE` clause has its own `FROM`
clause, and names inside it should resolve against *that* `FROM` clause first,
falling back to the enclosing query's scope only for a correlated reference.
A CTE has its own scope. A macro body has its own scope, populated with the
macro's parameters. DuckDB's answer to "who tracks what's in scope" is the
`Binder` class itself, and the rule is simple: **there is exactly one
`Binder` per lexical scope**, and nested scopes get their own child `Binder`,
linked back to the scope that contains them.

> `src/include/duckdb/planner/binder.hpp` (header doc comment, condensed):
> "The binder is responsible for binding tables and columns to actual
> physical tables and columns in the catalog. In the process, it also
> resolves types of all expressions." — one `Binder` object handles exactly
> one such pass, over exactly one scope's worth of AST.

New binders come from one factory method:

```cpp
// src/include/duckdb/planner/binder.hpp:228-229
DUCKDB_API static shared_ptr<Binder> CreateBinder(ClientContext &context, optional_ptr<Binder> parent = nullptr,
                                                  BinderType binder_type = BinderType::REGULAR_BINDER);
```

and the parent link it establishes is a real member, not just a convention:

```cpp
// src/include/duckdb/planner/binder.hpp:430-431
//! The parent binder (if any)
shared_ptr<Binder> parent;
```

Every place in the source that needs a fresh scope calls `CreateBinder` with
the enclosing `Binder` passed as `parent`. The clearest example is scalar
subquery binding, which is also the exact mechanism Query B (§4.7) will run
through:

```cpp
// src/planner/binder/expression/bind_subquery_expression.cpp:66-73
	// bind the subquery in its own binder
	auto subquery_binder = Binder::CreateBinder(context, binder);
	subquery_binder->SetCanContainNulls(true);
	subquery_binder->SetInsideSubquery();

	subquery_binder->BeginSubqueryBind(binder, *this);
	auto bound_node = subquery_binder->BindNode(*expr.Subquery()->node);
	subquery_binder->FinishSubqueryBind();
```

`binder` here is `this` `ExpressionBinder`'s enclosing `Binder` — the scope
that contains the subquery expression. `subquery_binder` is a brand new
`Binder`, scoped to exactly the subquery's own query node, with `binder` as
its `parent`. The same pattern recurs, with the same `CreateBinder(context,
binder)` shape, everywhere DuckDB needs a nested scope: CTEs
(`CTEBindState::query_binder` in `table_binding.hpp:188`), table macros, and
lateral joins.

Here's the shape of it as a diagram, using Query B from the front matter
(`SELECT c.name, (SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey)
FROM customer c`) as the concrete case:

```
┌─────────────────────────────────────────────────────────────┐
│ Binder (outer query)                                          │
│  bind_context:                                                 │
│    "c" → customer(name, custkey, ...)                          │
│  parent: none                                                  │
│                                                                  │
│   ┌───────────────────────────────────────────────────────┐   │
│   │ Binder (subquery, created via CreateBinder(ctx, outer)) │   │
│   │  bind_context:                                          │   │
│   │    "o" → orders(orderkey, custkey, ...)                  │   │
│   │  parent: outer Binder ───────────────────────────────┐  │   │
│   └────────────────────────────────────────────────────┼──┘   │
└──────────────────────────────────────────────────────────┼──────┘
                                                            │
                                          resolves c.custkey by walking
                                          outward through this parent link
```

Why does a nested scope need its *own* `Binder`, rather than the outer
`Binder` just recursing into the subquery's AST with the same `BindContext`?
Two reasons, both visible in the code above:

- **Shadowing and independence.** The subquery's `FROM orders o` must not
  leak into the outer query's namespace, and if `customer` also had an alias
  `o` somewhere, the subquery's `o` must shadow it inside the subquery without
  corrupting the outer scope's own idea of what `o` means. A fresh
  `BindContext` per `Binder` (`bind_context` is a member of `Binder`, not a
  free-floating singleton — `binder.hpp:234`) gives each scope an
  independent namespace.
- **Correlated references need a controlled escape hatch outward, not an
  unrestricted merge.** `o.custkey = c.custkey` inside the subquery needs `c`
  to resolve, and `c` is not in the subquery's own `bind_context` at all. The
  mechanism for that (§4.7 walks it in full) is the child binder explicitly
  reaching into its `parent` chain when its own scope draws a blank — it is
  not that the two scopes share one flat namespace. `ExpressionBinder`'s scope
  chain — "this binder followed by its enclosing scopes" (comment at
  `expression_binder.hpp:175-176`) — is walked one level at a time via
  `ScopeAt(idx_t depth)`, and each `depth` a name resolves at gets recorded on
  the *bound expression itself* (`BoundColumnRefExpression::Depth()`), which
  is exactly the signal that later triggers `LogicalDependentJoin`
  construction.

## 4.3 `BindContext`: where names actually get resolved

If `Binder` is "one object per scope," `BindContext` is the object *inside*
that scope doing the actual bookkeeping. Its header doc comment states its
job plainly:

> `src/include/duckdb/planner/bind_context.hpp:41-42` — "The BindContext
> object keeps track of all the tables and columns that are encountered
> during the binding process."

Every table, CTE, or subquery that appears in a `FROM` clause gets registered
as a `Binding` object (`src/include/duckdb/planner/table_binding.hpp`), keyed
by a `BindingAlias` (a possibly-qualified name — catalog, schema path, and
alias: `src/include/duckdb/planner/binding_alias.hpp:45-49`). `BindContext`
holds these in `bindings_list`, in the order they were registered
(`bind_context.hpp:181-182`), and exposes the lookup API a column reference
actually calls into:

```cpp
// src/include/duckdb/planner/bind_context.hpp:48-57
//! Given a column name, find the matching table it belongs to. Throws an
//! exception if no table has a column of the given name.
optional_ptr<Binding> GetMatchingBinding(const Identifier &column_name,
                                         QueryErrorContext context = QueryErrorContext());
//! Like GetMatchingBinding, but instead of throwing an error if multiple tables have the same binding it will
//! return a list of all the matching ones
vector<reference<Binding>> GetMatchingBindings(const Identifier &column_name);
//! Like GetMatchingBindings, but returns the top 3 most similar bindings (in levenshtein distance) instead of the
//! matching ones
vector<Identifier> GetSimilarBindings(const Identifier &column_name);
```

`GetMatchingBinding` (singular) is the "resolve an *unqualified* column
reference" entry point: given just `custkey`, scan every registered `Binding`
in the scope, and if exactly one of them has a matching column, return it —
if more than one does, throw an ambiguity error naming both candidates:

```cpp
// src/planner/bind_context.cpp:49-72
optional_ptr<Binding> BindContext::GetMatchingBinding(const Identifier &column_name, QueryErrorContext context) {
	optional_ptr<Binding> result;
	for (auto &binding_ptr : bindings_list) {
		auto &binding = *binding_ptr;
		auto is_using_binding = GetUsingBinding(column_name, binding.GetBindingAlias());
		if (is_using_binding) {
			continue;
		}
		if (binding.HasMatchingBinding(column_name)) {
			if (result) {
				throw BinderException(context,
				                      "Ambiguous reference to column name %s (use: '%s.%s' "
				                      "or '%s.%s')",
				                      column_name,
				                      MinimumUniqueAlias(result->GetBindingAlias(), binding.GetBindingAlias()),
				                      SQLIdentifier(column_name),
				                      MinimumUniqueAlias(binding.GetBindingAlias(), result->GetBindingAlias()),
				                      SQLIdentifier(column_name));
			}
			result = &binding;
		}
	}
	return result;
}
```

`GetMatchingBindings` (plural) is the building block for USING-join handling
(§4.4), and `GetSimilarBindings` is the "did you mean" mechanism, covered on
its own in §4.6.

### `ColumnBinding`: the currency of the entire logical-plan layer

Once a name resolves to a `Binding`, that `Binding` doesn't hand back a
string — it hands back a `ColumnBinding`:

```cpp
// src/include/duckdb/planner/column_binding.hpp:22-28
struct ColumnBinding {
	ColumnBinding();
	ColumnBinding(TableIndex table, ProjectionIndex column);

	TableIndex table_index;
	ProjectionIndex column_index;
	...
```

This is not an incidental detail — it is the single most important fact
about how the rest of the plan is wired together. **`ColumnBinding` is the
currency of the entire logical-plan layer.** Every `LogicalOperator`'s
`GetColumnBindings()` returns a `vector<ColumnBinding>`, and every expression
below a scan that refers to a column upstream does so purely by this
`{table_index, column_index}` pair — never again by name. A `LogicalGet` for
`orders` is assigned a `TableIndex` when it's created; each of its columns
gets a `column_index` within that table's projection; and a
`BoundColumnRefExpression` anywhere downstream (in a `LogicalFilter`'s
predicate, a `LogicalJoin`'s condition, a `LogicalAggregate`'s group key) that
wants `o.custkey` carries exactly `ColumnBinding{table_index_of_o,
column_index_of_custkey}` — a pair of integers — and nothing else. The
concrete resolution site, `bind_columnref_expression.cpp`, ends by producing
exactly this:

```cpp
// src/planner/table_binding.cpp:319-320 (TableBinding::Bind, final line)
	ColumnBinding binding = GetColumnBinding(column_index);
	return BindResult(make_uniq<BoundColumnRefExpression>(Identifier(colref.GetName()), col_type, binding, depth));
```

This is why the optimizer (Chapter 7) and physical planner (Chapter 9) never
need a symbol table again, and it's why column pruning, projection pushdown,
and expression rewriting downstream of the binder can all be done as pure
integer-pair bookkeeping rather than string comparison. It also means a
renamed alias or a repeated table name (`orders o1 JOIN orders o2`) causes
zero ambiguity below the binder — `o1` and `o2` get distinct `TableIndex`
values the moment their `LogicalGet`s are created, and everything downstream
only ever sees those two distinct indices.

### USING / NATURAL join: `BindContext` earning its keep

`GetMatchingBinding`'s ambiguity check is the *simple* case. A harder one is
`t1 a JOIN t2 b USING (id)`: here `id` is expected to resolve to *both*
tables' `id` column simultaneously for the join condition, but afterwards, an
unqualified reference to `id` in the rest of the query should resolve to a
single, merged, coalesced column — not throw the ambiguity error above. This
needs its own piece of bookkeeping, `UsingColumnSet`:

```cpp
// src/include/duckdb/planner/bind_context.hpp:34-37
struct UsingColumnSet {
	BindingAlias primary_binding;
	vector<BindingAlias> bindings;
};
```

`bind_joinref.cpp` builds one of these per `USING`/`NATURAL` column while
binding a `JoinRef`, and constructs the actual bound comparison directly
(not via the ordinary parsed-expression path, because the two sides come
from two different bindings that need to be bound against each in their own
right):

```cpp
// src/planner/binder/tableref/bind_joinref.cpp:30-45
//! Build the bound comparison of a USING/NATURAL column. Each side is bound against its own binder,
//! which is why the condition is built bound rather than as a parsed expression.
static unique_ptr<Expression> AddCondition(ClientContext &context, Binder &left_binder, Binder &right_binder,
                                           const BindingAlias &left_alias, const BindingAlias &right_alias,
                                           const Identifier &column_name, ExpressionType type) {
	ExpressionBinder expr_binder(left_binder, context);
	auto left = BindColumn(left_binder, context, left_alias, column_name);
	auto right = BindColumn(right_binder, context, right_alias, column_name);
	ErrorData error;
	auto condition = expr_binder.CreateBoundComparison(type, std::move(left), std::move(right), error);
	if (!condition) {
		error.Throw();
	}
	return condition;
}
```

The resulting `UsingColumnSet` is registered on `BindContext` via
`AddUsingBinding`/`AddUsingBindingSet` (`bind_context.hpp:133-135`), keyed by
column name in a private `using_columns` map (`bind_context.hpp:184`).
Afterwards, `GetMatchingBinding`'s loop explicitly *skips* any binding that
participates in a `UsingColumnSet` for the name being resolved
(`is_using_binding` check in the excerpt above) — that's how an unqualified
`id` after `USING (id)` avoids the "ambiguous, appears on both sides" error
that an ordinary two-`id`-columns join would trigger: `BindContext` has
already recorded that these two bindings' `id` columns are meant to be one
logical column, not two competing ones, and routes resolution through the
`UsingColumnSet`'s `primary_binding` instead.

## 4.4 Type resolution and implicit casts

Column and table resolution answers *what something refers to*. A second,
parallel job the binder does is answer *what type it has*, and — where two
already-typed things need to agree — *insert whatever cast makes them agree*.
This job is split across a base class and a family of subclasses, because the
rules genuinely differ by SQL clause.

### `ExpressionBinder`: the base class, and why clauses need their own rules

Every expression gets bound by walking through `ExpressionBinder::
BindExpression`, a single large dispatch over `ExpressionClass`:

```cpp
// src/planner/expression_binder.cpp:50-107 (excerpt)
BindResult ExpressionBinder::BindExpression(unique_ptr<ParsedExpression> &expr, idx_t depth, bool root_expression) {
	auto stack_checker = StackCheck(*expr);

	if (binder.HasEnclosingScope()) {
		// an expression an enclosing query groups by is a reference to that group, e.g. `(SELECT x::DATE)`
		// inside a query that groups by `x::DATE`
		auto owner = ResolveOuterGroupOwner(*expr);
		if (owner.IsValid()) {
			return BindGroupInEnclosingScope(expr, owner.GetIndex(), depth);
		}
	}
	auto &expr_ref = *expr;
	switch (expr_ref.GetExpressionClass()) {
	case ExpressionClass::BETWEEN:
		return BindExpression(expr_ref.Cast<BetweenExpression>(), depth);
	case ExpressionClass::CASE:
		return BindExpression(expr_ref.Cast<CaseExpression>(), depth);
	...
	case ExpressionClass::COLUMN_REF:
		return BindExpression(expr_ref.Cast<ColumnRefExpression>(), depth, root_expression, expr);
	...
	case ExpressionClass::FUNCTION: {
		auto &function = expr_ref.Cast<FunctionExpression>();
		if (IsUnnestFunction(function.FunctionName())) {
			// special case, not in catalog
			return BindUnnest(function, depth, root_expression);
		}
		// binding a function expression requires an extra parameter for macros
		return BindExpression(function, depth, expr);
	}
	...
	case ExpressionClass::WINDOW:
		// a binder that does not handle windows itself reports that they are unsupported here
		return BindUnsupportedExpression(expr_ref, depth, UnsupportedWindowMessage());
	...
```

The
dispatch starts with one special check, the `if (binder.HasEnclosingScope())`
block above. It only runs inside a subquery. It asks: "does an enclosing
query `GROUP BY` this exact expression?" If yes, the expression is bound as
a reference to that outer group. §4.8.1 shows what this allows.

But *which* expression classes are even legal, and how names that don't
resolve locally get treated, depends entirely on which clause is doing the
binding — a `WHERE` clause has different rules from an `ORDER BY` clause, even
though both are ultimately binding a `ParsedExpression` tree using the same
machinery. DuckDB's answer is a family of clause-specific subclasses
(declared under `src/include/duckdb/planner/expression_binder/`):

```
alter_binder.hpp        group_binder.hpp        qualify_binder.hpp
base_select_binder.hpp  having_binder.hpp        relation_binder.hpp
check_binder.hpp        index_binder.hpp         returning_binder.hpp
column_alias_binder.hpp insert_binder.hpp        select_bind_state.hpp
constant_binder.hpp     lateral_binder.hpp       select_binder.hpp
order_binder.hpp        projection_binder.hpp    table_function_binder.hpp
                        update_binder.hpp        where_binder.hpp
```

Each overrides a handful of `virtual` hooks `ExpressionBinder` exposes for
exactly this purpose — `BindExpression` itself (to reject whole expression
classes outright), `UnsupportedAggregateMessage()`/`UnsupportedWindowMessage()`
(to customize the error text), and `TryResolveAliasReference` /
`ClaimsAlias` (to opt into or out of resolving a bare name against a
`SELECT`-list alias). This is where the concrete difference the reader has
probably experienced first-hand — "why can't I write a bare aggregate in
`WHERE`, but `ORDER BY` lets me reference a `SELECT`-list alias?" — actually
lives in the source.

### Concrete case: `WhereBinder` rejects aggregates, `OrderBinder` allows aliases

`WhereBinder` is a thin subclass. It rejects two expression classes outright
and supplies its own aggregate-error text:

```cpp
// src/planner/expression_binder/where_binder.cpp:7-26
WhereBinder::WhereBinder(Binder &binder, ClientContext &context, optional_ptr<ColumnAliasBinder> column_alias_binder)
    : ExpressionBinder(binder, context), column_alias_binder(column_alias_binder) {
	target_type = LogicalType(LogicalTypeId::BOOLEAN);
}

BindResult WhereBinder::BindExpression(unique_ptr<ParsedExpression> &expr_ptr, idx_t depth, bool root_expression) {
	auto &expr = *expr_ptr;
	switch (expr.GetExpressionClass()) {
	case ExpressionClass::DEFAULT:
		return BindUnsupportedExpression(expr, depth, "WHERE clause cannot contain DEFAULT clause");
	case ExpressionClass::WINDOW:
		return BindUnsupportedExpression(expr, depth, "WHERE clause cannot contain window functions!");
	default:
		return ExpressionBinder::BindExpression(expr_ptr, depth);
	}
}

string WhereBinder::UnsupportedAggregateMessage() {
	return "WHERE clause cannot contain aggregates!";
}
```

Note what's *not* here: there's no `case ExpressionClass::FUNCTION` special
case for aggregates in `WhereBinder::BindExpression`. The rejection happens
one layer down — the base `ExpressionBinder::BindExpression(FunctionExpression
&...)` path resolves the function against the catalog exactly as it would
anywhere else (an aggregate is still a perfectly real, resolvable catalog
entry), and only the base class's generic "this binder doesn't accept
aggregates here" logic calls `UnsupportedAggregateMessage()` to produce
`WhereBinder`'s specific wording when it discovers what it just resolved was
an aggregate. That is: `SUM(x)` bare in `WHERE` is not a *parse* error, and
not even a *name resolution* error — it resolves against the aggregate
catalog completely successfully — it is caught one step later, purely because
`WhereBinder` is the clause-specific policy object saying "not here."

`OrderBinder` is a different animal entirely, and its difference is
structural, not just a swapped error string. Its whole `Bind()` method,
before doing any type resolution at all, tries to treat the `ORDER BY`
expression as a *reference back into the already-bound `SELECT` list* —
either by alias, or by integer position:

```cpp
// src/planner/expression_binder/order_binder.cpp:140-160 (excerpt)
unique_ptr<Expression> OrderBinder::Bind(unique_ptr<ParsedExpression> expr) {
	// in the ORDER BY clause we do not bind children
	// we bind ONLY to the select list
	// if there is no matching entry in the SELECT list already, we add the expression to the SELECT list and refer the
	// new expression the new entry will then be bound later during the binding of the SELECT list we also don't do type
	// resolution here: this only happens after the SELECT list has been bound
	switch (expr->GetExpressionClass()) {
	case ExpressionClass::CONSTANT: {
		// ORDER BY constant
		// is the ORDER BY expression a constant integer? (e.g. ORDER BY 1)
		return BindConstant(*expr);
	}
	case ExpressionClass::COLUMN_REF:
	case ExpressionClass::POSITIONAL_REFERENCE: {
		// COLUMN REF expression
		// check if we can bind it to an alias in the select list
		auto index = TryGetProjectionReference(*expr);
		if (index.IsValid()) {
			return CreateProjectionReference(*expr, index.GetIndex());
		}
		break;
	}
	...
```

and the alias/position lookup itself:

```cpp
// src/planner/expression_binder/order_binder.cpp:54-89 (excerpt)
optional_idx OrderBinder::TryGetProjectionReference(ParsedExpression &expr) const {
	switch (expr.GetExpressionClass()) {
	case ExpressionClass::CONSTANT: {
		auto &constant = expr.Cast<ConstantExpression>();
		// ORDER BY a constant
		int64_t order_value;
		if (!constant.GetLiteral().TryGetInt64(order_value)) {
			...
		}
		// INTEGER constant: we use the integer as an index into the select list (e.g. ORDER BY 1)
		return static_cast<idx_t>(order_value <= 0 ? NumericLimits<int64_t>::Maximum() : order_value - 1);
	}
	case ExpressionClass::COLUMN_REF: {
		auto &colref = expr.Cast<ColumnRefExpression>();
		if (!ExpressionBinder::IsPotentialAlias(colref)) {
			break;
		}

		auto &alias_name = colref.ColumnNames().back();
		// check the alias list
		auto entry = bind_state.alias_map.find(alias_name);
		if (entry != bind_state.alias_map.end()) {
			// this is an alias - return the index
			return entry->second;
		}
		...
```

So `ORDER BY total DESC` in Query A works precisely because `OrderBinder`
looks `total` up in `bind_state.alias_map` — a map of `SELECT`-list aliases
built while binding the projection — *before* it ever tries to resolve
`total` as an ordinary column reference against any table's `BindContext`.
`ORDER BY 1` works the same way, through the sibling `CONSTANT` case, treating
the integer as a 1-based index into the already-bound select list. Neither
alias resolution nor positional resolution is something `WhereBinder` offers
at all — a `WHERE total > 10` referring to a `SELECT`-list alias is invalid in
standard SQL specifically because filtering conceptually happens *before* the
projection has produced anything called `total`, and DuckDB's binder encodes
that standard-SQL rule as "these are two different subclasses of
`ExpressionBinder`, and only one of them consults `alias_map`" rather than as
an ad hoc special case checked inline somewhere.

### `FunctionBinder`: overload resolution and implicit casts

Resolving `sum(l.extendedprice)` or `o.orderkey = l.orderkey` requires
picking a concrete function/operator overload given the already-bound
argument types, and — where the closest overload doesn't match exactly —
deciding what implicit cast to splice in. This is `FunctionBinder`'s job
(`src/include/duckdb/function/function_binder.hpp`), and it's invoked from
the ordinary function-binding path the moment a name resolves to a callable
catalog entry:

```cpp
// src/planner/binder/expression/bind_function_expression.cpp:285-334 (excerpt)
BindResult ExpressionBinder::BindExpression(FunctionExpression &function, idx_t depth,
                                            unique_ptr<ParsedExpression> &expr_ptr) {
	auto &func = BindFunction(function);
	...
	switch (func.type) {
	case CatalogType::SCALAR_FUNCTION_ENTRY: {
		auto child = function.IsLambdaFunction();
		if (child) {
			auto syntax_type = child->Cast<LambdaExpression>().GetLambdaSyntaxType();
			return TryBindLambdaOrJson(function, depth, func, syntax_type);
		}
		return BindFunction(function, func.Cast<ScalarFunctionCatalogEntry>(), depth);
	}
	case CatalogType::MACRO_ENTRY:
		// macro function - a prepared statement must be re-bound when the macro changes
		if (!func.ParentCatalog().IsSystemCatalog()) {
			binder.GetStatementProperties().RegisterDBRead(func.ParentCatalog(), context);
		}
		return BindMacro(function, func.Cast<ScalarMacroCatalogEntry>(), depth, expr_ptr);
	case CatalogType::AGGREGATE_FUNCTION_ENTRY: {
		// an aggregate belongs to the innermost query level in which one of its arguments resolves a
		// column of that level, so that the level collecting it is the one the arguments refer to
		if (!binder.GetEnclosingScopes().empty()) {
			auto owner = ResolveAggregateOwner(function, 0);
			if (owner.IsValid() && owner.GetIndex() != 0) {
				return BindAggregateInEnclosingScope(function, owner.GetIndex(), depth, expr_ptr);
			}
		}
		return BindAggregate(function, func.Cast<AggregateFunctionCatalogEntry>(), depth);
	}
	case CatalogType::WINDOW_FUNCTION_ENTRY:
		// window function
		return BindWindow(function, func.Cast<WindowFunctionCatalogEntry>(), depth);
	default:
		throw InvalidInputException("Unsupported catalog type when binding function");
	}
}
```

The
`MACRO_ENTRY` case also records that the statement reads the macro's catalog
(`RegisterDBRead`). A prepared statement that calls a user macro is
therefore bound again if the macro changes, instead of running a stale
macro body. Views and table macros get the same treatment. In
`test/sql/prepared/prepared_rebind_macro_view.test`, `PREPARE q AS SELECT
m(10)` returns `11` with `m(x) AS x + 1`, and `1000` after
`CREATE OR REPLACE MACRO m(x) AS x * 100`.

Note the dispatch on `func.type`: `BindFunction` first resolves the *name*
`sum` against the catalog (via `ColumnQualifier::QualifyFunction`, which can
find scalar functions, macros, aggregates, or window functions all sharing
one namespace), and only afterward does the binder decide which kind of
catalog entry it got and hand off to the matching bind path. `AGGREGATE_
FUNCTION_ENTRY` goes to `BindAggregate`, which is where DuckDB decides which
concrete `sum` overload applies — DuckDB ships several `sum` overloads keyed
by input type (`sum(DOUBLE)`, `sum(BIGINT)`, `sum(HUGEINT)`, `sum(DECIMAL)`,
...) — and it is `FunctionBinder::ResolveFunction` that picks among them:

```cpp
// src/include/duckdb/function/function_binder.hpp:275-277 and 297-299 (selected overload-resolution signatures)
pair<BoundScalarFunction, unique_ptr<FunctionData>>
ResolveFunction(shared_ptr<const ScalarFunction> function, vector<unique_ptr<Expression>> &children,
                vector<pair<Identifier, unique_ptr<Expression>>> &keyword_args);
...
pair<BoundAggregateFunction, unique_ptr<FunctionData>>
ResolveFunction(shared_ptr<const AggregateFunction> function, vector<unique_ptr<Expression>> &children,
                vector<pair<Identifier, unique_ptr<Expression>>> &keyword_args);
```

`ResolveFunction` is given the already-bound argument expressions (so their
types are already known — a `LogicalType` was assigned to
`l.extendedprice` back when *its* column reference resolved), scores each
candidate overload by how well its declared parameter types match the
argument types (exact match, then implicit-castable match, then no match),
picks the best-scoring overload, and — this is the "implicit cast insertion"
half of the job — wraps any argument that didn't match its chosen overload's
parameter type exactly in a `BoundCastExpression`
(`bind_cast_expression.cpp`) so that, by the time `BindFunction` returns, the
argument list's types line up exactly with the chosen overload's signature.
`l.extendedprice`, a `DECIMAL` column in the TPC-H schema, resolving against
`sum`'s `DECIMAL` overload needs no cast at all in Query A; had it been an
`INTEGER` column, `ResolveFunction` would have selected `sum(BIGINT)` (or
promoted it) and spliced an implicit numeric widening cast around the
argument before returning.

Binding a
scalar function has two clear steps. Step one resolves the
types. Step two calls the function's `bind` callback. Step one has its own
helper, `ResolveScalarFunctionTypes`, and functions can register a
separate `resolve_types` callback (`scalar_function.hpp:196`) for the
type part of their work:

```cpp
// src/function/function_binder.cpp:1548-1580 (excerpt)
ResolvedScalarFunctionTypes
FunctionBinder::ResolveScalarFunctionTypes(shared_ptr<const ScalarFunction> function_p,
                                           vector<unique_ptr<Expression>> &arguments,
                                           vector<pair<Identifier, unique_ptr<Expression>>> &named_arguments) {
	...
	// Reorder named args and expand the variadic arguments
	result.argument_names = ResolveArguments(function, bound_function, arguments, named_arguments);

	// Attempt to resolve template types, before we call the "resolve_types" callback.
	ResolveTemplateTypes(bound_function, arguments);
	...
	if (bound_function.HasResolveTypesCallback()) {
		ResolveScalarFunctionTypesInput input(context, bound_function, arguments, result.argument_names);
		bound_function.GetResolveTypesCallback()(input);
		...
	}
	...
	result.pushed_collation =
	    PushArgumentCollations(context, bound_function, bound_function.GetProperties(), arguments, is_index_key);
	CastToFunctionArguments(bound_function, arguments);
	return result;
}
```

So the order is: place named arguments, resolve template types, run
`resolve_types`, push collations, and only then insert the implicit casts
(`CastToFunctionArguments`). `ResolveFunction` calls this helper first and
then runs the `bind` callback on arguments that already have their final
types.

Also, **constant arguments are folded while binding.** A
constant argument is one that does not read any column, like
`'a' || 'b'`. The binder evaluates such arguments anyway, to check whether
they are `NULL`. `FoldConstantArgument` (`function_binder.cpp:864-889`)
keeps that result and replaces the argument with a
`BoundConstantExpression`. Each enclosing function call then sees a
constant and does not evaluate the subtree again. Without this, binding is
very slow for long chains: the commit that added it reports that a chain of
1000 `'a' || 'a' || ...` took 20 seconds to bind without folding and under
100 milliseconds with it. Lambda
arguments are never folded, because a lambda has no value of its own.

`CollationBinding` (`src/planner/collation_binding.cpp`) rides alongside this
for one specific, easy-to-miss type-resolution wrinkle: string comparisons
and `ORDER BY` on strings are collation-sensitive (case-insensitive or
locale-aware comparison, for instance), and `PushCollation`
(`expression_binder.hpp:171-172`) is what threads the correct collation
function through a bound comparison or sort key once the operand types are
known to be `VARCHAR`.

## 4.5 `BoundStatement`: the actual "Bind" object — and why it's also a plan

Every path above — column resolution, table resolution, function overload
resolution, cast insertion — ultimately feeds into producing one struct per
statement, `BoundStatement`:

```cpp
// src/include/duckdb/planner/bound_statement.hpp
struct ExtraBoundInfo {
    SetOperationType setop_type = SetOperationType::NONE;
    vector<shared_ptr<Binder>> child_binders;
    vector<BoundStatement> bound_children;
    vector<unique_ptr<ParsedExpression>> original_expressions;
};

struct BoundStatement {
    unique_ptr<LogicalOperator> plan;
    vector<LogicalType> types;
    vector<Identifier> names;
    ExtraBoundInfo extra_info;
};
```

Field by field:

- **`plan`** — a fully-formed `unique_ptr<LogicalOperator>` tree. This is the
  field that makes the "fused" architectural claim literal rather than
  figurative: there is no earlier, not-yet-a-plan representation this field
  gets converted *from*. It is built up directly, node by node, while the
  rest of binding happens (§4.5.1 below shows exactly where).
- **`types`** — the `LogicalType` of each output column of the statement, in
  order. For `SELECT o.custkey, SUM(l.extendedprice) AS total FROM ...`, this
  is `[custkey's declared type, the resolved return type of the sum overload
  chosen above]`.
- **`names`** — the output column names, in the same order (`custkey`,
  `total`) — this is purely presentational, for reporting a result set's
  column headers back to the client; nothing below the binder uses these
  strings for resolution, per the `ColumnBinding` point in §4.3.
- **`extra_info`** — the overflow struct for anything a `BoundStatement`
  needs to carry that isn't "the plan and its output schema." Its most
  important field for this chapter is `child_binders`: the list of child
  `Binder`s (§4.2) created while binding this statement — a `UNION` or CTE's
  branches, for instance, each get their own child `Binder`, and
  `bound_children` holds each branch's own already-produced `BoundStatement`.
  This is the same parent/child `Binder` relationship from §4.2, reified as
  data on the struct that gets handed up to the caller.

`BoundStatement` is produced by `Binder::Bind(SQLStatement&)`, which
dispatches by statement kind into one of the `bind_select.cpp`,
`bind_insert.cpp`, `bind_create.cpp`, ... files under
`src/planner/binder/statement/`, and it is what the rest of the system —
`Optimizer::Optimize`, then `PhysicalPlanGenerator::Plan` — consumes from
here on. Chapter 5 is dedicated to the *shape* of the `plan` tree this
struct carries; this chapter is about how that tree gets built.

The caller of `Binder::Bind` is `Planner::CreatePlan`. Extensions can run
code just before and just after it, through a `PlannerExtension`:

```cpp
// src/planner/planner.cpp:209-216 (excerpt)
		auto binding_timer = profiler.StartTimer<MetricPlannerBindingTime>();
		binder->SetParameters(bound_parameters);
		binder->SetStatementParameters(statement.named_param_map);
		RunPreBindExtensions(context, *binder, statement);
		auto bound_statement = binder->Bind(statement);
		binding_timer.EndTimer();

		RunPostBindExtensions(context, *binder, bound_statement);
```

The
`pre_bind_function` hook
(`src/include/duckdb/planner/planner_extension.hpp:45-48`) receives the
parsed `SQLStatement` right before binding starts. Its doc comment gives
the use case: "allowing e.g. a catalog to start loading the entries the
statement references". For example, a remote catalog can start fetching
table metadata before the binder asks for it. The `post_bind_function` hook
can change the finished `BoundStatement` in place.

### 4.5.1 The evidence: `plan_*.cpp` builds `LogicalOperator`s, not an intermediate IR

The clearest way to see that binding and logical-plan construction are one
activity, not two, is to watch the actual `make_uniq<Logical...>()` call
sites and notice which files they live in. They are not in some later
"planning" module reading a bound-but-unplanned tree — they are in the same
`src/planner/binder/` subtree as everything else in this chapter, in files
whose names literally start with `plan_`:

```cpp
// src/planner/binder/query_node/plan_select_node.cpp:18-23 — WHERE becomes LogicalFilter
unique_ptr<LogicalOperator> Binder::PlanFilter(unique_ptr<Expression> condition, unique_ptr<LogicalOperator> root) {
	PlanSubqueries(condition, root);
	auto filter = make_uniq<LogicalFilter>(std::move(condition));
	filter->AddChild(std::move(root));
	return std::move(filter);
}
```

```cpp
// src/planner/binder/tableref/plan_joinref.cpp:337-343 — JOIN ... ON becomes LogicalComparisonJoin
	// Case 3: Has comparison conditions - Use comparison join
	auto comp_join = make_uniq<LogicalComparisonJoin>(type, LogicalOperatorType::LOGICAL_COMPARISON_JOIN);
	comp_join->conditions = std::move(all_conditions);
	comp_join->children.push_back(std::move(left_child));
	comp_join->children.push_back(std::move(right_child));

	return std::move(comp_join);
```

```cpp
// src/planner/binder/tableref/bind_basetableref.cpp:289-291 — FROM orders becomes LogicalGet
	auto logical_get =
	    make_uniq<LogicalGet>(table_index, BoundTableFunction(scan_function), std::move(bind_data),
	                          std::move(return_types), std::move(return_names), std::move(virtual_columns));
```

`LogicalAggregate` (`plan_select_node.cpp:174`) and `LogicalProjection`
(`plan_select_node.cpp:224`) for the `GROUP BY`/`SUM` and the final
`SELECT`-list are constructed the same way, in the same file, alongside the
rest of `Binder::PlanQueryNode` / `Binder::PlanSelectNode`'s logic. There is
no point at which any of this constructs some other, `LogicalOperator`-free
representation first. The `plan_*.cpp` naming convention inside
`binder/query_node/` and `binder/tableref/`
(`plan_select_node.cpp`, `plan_query_node.cpp`, `plan_joinref.cpp`,
`plan_subquery.cpp`, `plan_setop.cpp`) exists precisely because these files
are where *binding a name* and *emitting the plan node that depends on that
binding* happen as one inseparable step — resolving `orders` to a
`TableCatalogEntry` and its columns' `LogicalType`s (binding) is what
supplies the arguments `LogicalGet`'s constructor needs (planning); there
would be nothing left over to plan afterward if binding didn't already
produce the plan.

## 4.6 A concrete illustration of `BindContext` earning its keep: "did you mean"

One more piece of `BindContext`'s job, briefly: when a column reference
doesn't resolve at all, `BindContext` doesn't just say "not found" — it
offers candidates, ranked by string similarity. `GetSimilarBindings` scans
every column name registered across every `Binding` in scope and scores it
against the misspelled name by Levenshtein-style similarity:

```cpp
// src/planner/bind_context.cpp:74-90
vector<Identifier> BindContext::GetSimilarBindings(const Identifier &column_name) {
	vector<pair<string, double>> scores;
	for (auto &binding_ptr : bindings_list) {
		auto &binding = *binding_ptr;
		for (auto &name : binding.GetColumnNames()) {
			double distance = StringUtil::SimilarityRating(name.GetIdentifierName(), column_name.GetIdentifierName());
			// check if we need to qualify the column
			auto matching_bindings = GetMatchingBindings(name);
			if (matching_bindings.size() > 1) {
				scores.emplace_back(binding.GetAlias() + "." + name.GetIdentifierName(), distance);
			} else {
				scores.emplace_back(name, distance);
			}
		}
	}
	return StringsToIdentifiers(StringUtil::TopNStrings(scores));
}
```

and the caller that reaches for it is exactly where you'd expect: the point
where ordinary column-name qualification has just failed and is about to
give up and report an error:

```cpp
// src/planner/column_qualifier.cpp:184-192
	// bind as a regular column
	if (table_binding) {
		return binder.bind_context.CreateColumnReference(table_binding->GetBindingAlias(), column_name);
	}

	// it's not, find candidates and error
	auto similar_bindings = binder.bind_context.GetSimilarBindings(column_name);
	error = ErrorData(BinderException::ColumnNotFound(column_name, similar_bindings));
	return nullptr;
```

`BinderException::ColumnNotFound` then formats whatever candidates came back
into the message text:

```cpp
// src/common/exception/binder_exception.cpp:15-24
BinderException BinderException::ColumnNotFound(const Identifier &name, const vector<Identifier> &similar_bindings,
                                                QueryErrorContext context) {
	auto extra_info = Exception::InitializeExtraInfo("COLUMN_NOT_FOUND", context.query_location);
	string candidate_str = StringUtil::CandidatesMessage(IdentifiersToStrings(similar_bindings), "Candidate bindings");
	extra_info["name"] = name.GetIdentifierName();
	if (!similar_bindings.empty()) {
		extra_info["candidates"] = StringUtil::Join(similar_bindings, ",");
		return BinderException(extra_info, StringUtil::Format("Referenced column \"%s\" not found in FROM clause!%s",
		                                                      name.GetIdentifierName(), candidate_str));
	} else {
		...
```

To make this concrete: take Query A and misspell `l.shipdate` as
`l.shipdat` in the `WHERE` clause —

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdat < DATE '1998-01-01'   -- typo: shipdat, not shipdate
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

`WhereBinder` dispatches `l.shipdat` down through
`BindExpression(ColumnRefExpression&, ...)` → `QualifyColumnName` →
`ColumnQualifier`, which finds a `Binding` for alias `l` (the `lineitem`
scan) but no column on it named `shipdat`. The path above then walks every
column name registered on every `Binding` currently in `BindContext` —
`orders`'s `orderkey`, `custkey`, ..., and `lineitem`'s `orderkey`,
`extendedprice`, `shipdate`, ... — scores each against `shipdat` by string
similarity, and `shipdate` wins by a wide margin over everything else in
scope. The resulting error is DuckDB's familiar `Referenced column
"shipdat" not found in FROM clause!\nCandidate bindings: "shipdate", ...` —
a "did you mean" suggestion produced entirely out of the same `BindContext`
bookkeeping this section has been describing, not a separate spell-checking
subsystem. (This walkthrough is constructed to show the mechanism precisely;
`test/sql/catalog/did_you_mean.test` demonstrates the sibling case — the same
`StringUtil::CandidatesMessage`-style suggestion machinery applied to
*table*-name resolution instead of column-name resolution, e.g. `SELECT *
FROM helloo` suggesting `Did you mean "hello"`.)

## 4.7 Query A, traced through the binder

Time to walk Query A through everything above, end to end:

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

Binding a `SelectNode` proceeds, in order, roughly: bind the `FROM` clause,
bind the `WHERE` clause against the resulting scope, bind the `SELECT` list
(and `GROUP BY`), then bind `ORDER BY` and fold in `LIMIT` — and *plan* each
piece (build its `LogicalOperator`) as each piece finishes binding, per §4.5.

1. **`FROM orders o JOIN lineitem l ON o.orderkey = l.orderkey`.**
   `bind_basetableref.cpp` resolves `orders` against the catalog, obtains its
   `TableCatalogEntry`, assigns it a fresh `TableIndex`, and registers it in
   `BindContext` under alias `o` — every column of `orders` (`orderkey`,
   `custkey`, ...) is now a resolvable `Binding` entry. The same happens for
   `lineitem` under alias `l`, with its own, distinct `TableIndex`. Each
   `bind_basetableref.cpp` call directly constructs a `LogicalGet`
   (`bind_basetableref.cpp:289-290`). Then `bind_joinref.cpp` binds
   `o.orderkey = l.orderkey`: `o.orderkey` resolves via `GetMatchingBinding`
   restricted to alias `o` (qualified references skip the ambiguity-scan path
   entirely and look the alias up directly) to `ColumnBinding{table_index_o,
   0}` (say), `l.orderkey` resolves to `ColumnBinding{table_index_l, 0}`, and
   `plan_joinref.cpp` wraps the resulting `BoundComparisonExpression` into a
   `LogicalComparisonJoin` (`plan_joinref.cpp:338-343`) with the two
   `LogicalGet`s as its two children.

2. **`WHERE l.shipdate < DATE '1998-01-01'`.** This binds using a
   `WhereBinder` constructed over the join's combined `BindContext` (both `o`
   and `l` are now in scope). `l.shipdate` resolves to
   `ColumnBinding{table_index_l, k}` for whichever column index `shipdate` is
   on `lineitem`; the date literal binds to a `BoundConstantExpression` of
   type `DATE`; the comparison operator resolves through `FunctionBinder`
   exactly as any comparison would, needing no cast since both sides are
   already `DATE`. `PlanFilter` (§4.5.1) wraps this into a `LogicalFilter`
   sitting on top of the `LogicalComparisonJoin`.

3. **`SELECT o.custkey, SUM(l.extendedprice) AS total`.** `o.custkey`
   resolves the same qualified way as `o.orderkey` did, to
   `ColumnBinding{table_index_o, j}`. `SUM(l.extendedprice)` goes through the
   dispatch in §4.4: `BindFunction` resolves the name `sum` against the
   catalog, finds an `AggregateFunctionCatalogEntry`, and hands off to
   `BindAggregate`, which — via `FunctionBinder::ResolveFunction` — picks the
   `sum` overload matching `l.extendedprice`'s already-bound `DECIMAL` type
   (no cast needed here; a `BIGINT` column would have required one, per
   §4.4). This whole `SELECT`-list binding, together with the `GROUP BY
   o.custkey` it's checked against, drives `plan_select_node.cpp` to emit a
   `LogicalAggregate` (`plan_select_node.cpp:174`) sitting on top of the
   `LogicalFilter`, followed by a `LogicalProjection`
   (`plan_select_node.cpp:224`) selecting `custkey` and the finished
   aggregate out as `total`.

4. **`ORDER BY total DESC`.** As §4.4 traced in detail, `OrderBinder` matches
   `total` against the alias map built while binding step 3's `SELECT` list,
   and resolves it as a projection reference rather than re-resolving it as
   an independent expression. This becomes a `LogicalOrder`.

5. **`LIMIT 10`.** Folded into a `LogicalLimit` on top of the `LogicalOrder`.

Every column reference in the query — `o.custkey`, `l.extendedprice`,
`l.shipdate`, `l.orderkey`, `o.orderkey` — has, by the end of this, become a
`ColumnBinding{table_index, column_index}` pair; the strings `"o"`,
`"l"`, `"custkey"`, and so on appear nowhere in the resulting tree except as
cosmetic `names` on the outer `BoundStatement` (§4.5) and as display labels
`LogicalOperator::ToString()` can look up for `EXPLAIN` output. The resulting
unoptimized tree — the same shape Chapter 1 previewed, reproduced here with
the provenance now fully explained — is:

```
LogicalLimit (10)
  └─ LogicalOrder (total DESC)
       └─ LogicalAggregate (GROUP BY o.custkey; SUM(l.extendedprice))
            └─ LogicalFilter (l.shipdate < DATE '1998-01-01')
                 └─ LogicalComparisonJoin (o.orderkey = l.orderkey)
                      ├─ LogicalGet (orders)      TableIndex = table_index_o
                      └─ LogicalGet (lineitem)    TableIndex = table_index_l
```

This *is* `BoundStatement::plan` for Query A — not a translation of some
prior representation, but the literal output of the walk just described.
There is no correlated subquery anywhere in this query, so nothing in §4.8
below — `LogicalDependentJoin`, `FlattenDependentJoins` — ever gets
constructed for it; every `Binder` involved in binding Query A is the single
top-level `Binder` for the statement, with no child `Binder`s at all.

## 4.8 Query B: where a child scope stops resolving locally

Query B is the smallest query in this book's running pair that forces a
second `Binder` into existence, and that's exactly the point of it:

```sql
SELECT c.name,
       (SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey) AS n_orders
FROM customer c;
```

Binding the outer `SELECT` proceeds as in Query A: `FROM customer c` gives
one top-level `Binder` a `BindContext` with a single `Binding` for alias `c`,
covering `customer`'s columns (`custkey`, `name`, ...). Binding the outer
`SELECT`-list's second expression, the scalar subquery, is where things
diverge: `ExpressionBinder::BindExpression(SubqueryExpression&, idx_t)`
(§4.2) creates a *child* `Binder` — `Binder::CreateBinder(context, binder)`,
with the outer `Binder` passed as `parent` — and binds the subquery's own
`SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey` inside it.

That child `Binder` gets its own, independent `BindContext`, populated from
*its own* `FROM orders o` — containing exactly one `Binding`, for alias `o`,
covering `orders`'s columns. `o.custkey` inside the subquery resolves
against this local scope exactly the way `o.orderkey` did in Query A: an
ordinary qualified lookup, `ColumnBinding{table_index_o_inner, custkey_idx}`.

`c.custkey`, though, does not. The child `Binder`'s own `BindContext` has no
`Binding` for alias `c` — `c` was never registered here, because this
scope's own `FROM` clause never mentioned it. This is precisely the failure
path traced in §4.3's `ColumnQualifier::QualifyColumnName`: `table_binding`
comes back null for alias `c` inside this scope. Rather than immediately
falling through to the "not found, suggest a correction" path from §4.6,
`ExpressionBinder::BindExpression(ColumnRefExpression&, ...)` takes the other
branch already shown in §4.3's code excerpt:

```cpp
// src/planner/binder/expression/bind_columnref_expression.cpp:90-106 (recap)
	auto expr = QualifyColumnName(col_ref_p, error);
	if (!expr) {
		// column wasn't found
		if (ExpressionBinder::IsPotentialAlias(col_ref_p)) {
			...
		}
		// the name does not resolve in this scope: look for it in the enclosing ones
		return BindInEnclosingScope(col_ref_p, depth, expr_ptr, std::move(error));
	}
```

`BindInEnclosingScope` (`expression_binder.cpp:125-160`) is the mechanism
that walks *outward* along the `ScopeAt(depth)` chain — ultimately following
the child `Binder`'s `parent` link back to the outer `Binder` — resolving
`c.custkey` against the *outer* query's `BindContext`, where alias `c`
*does* have a `Binding`. Resolution succeeds there, at scope depth 1 instead
of depth 0, and the resulting `BoundColumnRefExpression` carries that depth
on itself. The moment a bound expression comes back with `Depth() > 0`,
`ExpressionBinder::ExtractCorrelatedExpressions` (`expression_binder.cpp:
176-185`) notices and records it:

```cpp
// src/planner/expression_binder.cpp:176-185
void ExpressionBinder::ExtractCorrelatedExpressions(Binder &binder, Expression &expr) {
	if (expr.GetExpressionType() == ExpressionType::BOUND_COLUMN_REF) {
		auto &bound_colref = expr.Cast<BoundColumnRefExpression>();
		if (bound_colref.Depth() > 0) {
			binder.AddCorrelatedColumn(CorrelatedColumnInfo(bound_colref));
		}
	}
	ExpressionIterator::EnumerateChildren(expr,
	                                      [&](Expression &child) { ExtractCorrelatedExpressions(binder, child); });
}
```

— registering `c.custkey`'s `ColumnBinding`, type, name, and depth on the
**child** `Binder`'s own `correlated_columns` list
(`CorrelatedColumns correlated_columns;` — `binder.hpp:237`). Back at the
call site that created the child `Binder` in the first place,
`bind_subquery_expression.cpp` inspects exactly this list once the subquery
finishes binding:

```cpp
// src/planner/binder/expression/bind_subquery_expression.cpp:75-84
	// check the correlated columns of the subquery for correlated columns with depth > 1
	for (idx_t i = 0; i < subquery_binder->correlated_columns.size(); i++) {
		CorrelatedColumnInfo corr = subquery_binder->correlated_columns[i];
		if (corr.depth > 1) {
			// depth > 1, the column references the query ABOVE the current one
			// add to the set of correlated columns for THIS query
			corr.depth -= 1;
			binder.AddCorrelatedColumn(corr);
		}
	}
```

The subquery's `correlated_columns` list being non-empty at this point is
the single fact that distinguishes Query B from Query A at bind time: the
resulting `BoundSubqueryExpression::IsCorrelated()` is true, and that boolean
is exactly what later decides which of two completely different planning
paths gets taken:

```cpp
// src/planner/binder/query_node/plan_subquery.cpp:808-825
unique_ptr<Expression> Binder::PlanSubquery(BoundSubqueryExpression &expr, unique_ptr<LogicalOperator> &root) {
	D_ASSERT(root);
	// first we translate the QueryNode of the subquery into a logical plan
	auto subquery_root = std::move(expr.SubqueryMutable().plan);
	D_ASSERT(subquery_root);

	// now we actually flatten the subquery
	auto plan = std::move(subquery_root);

	unique_ptr<Expression> result_expression;
	if (!expr.IsCorrelated()) {
		result_expression = PlanUncorrelatedSubquery(*this, expr, root, std::move(plan));
	} else {
		result_expression = PlanCorrelatedSubquery(*this, expr, root, std::move(plan));
	}
	IncreaseDepth();
	return result_expression;
}
```

For Query B, `PlanCorrelatedSubquery` is the path taken, and it is what
constructs the special logical node this book has been previewing since
Chapter 1: a `LogicalDependentJoin`
(`src/include/duckdb/planner/operator/logical_dependent_join.hpp:23-38`),
built via `CreateDuplicateEliminatedJoin`
(`plan_subquery.cpp:422`), carrying `correlated_columns` (the `c.custkey`
reference the child `Binder` recorded) as data on the node itself:

```cpp
// src/include/duckdb/planner/operator/logical_dependent_join.hpp:23-38
class LogicalDependentJoin : public LogicalJoin {
public:
	static constexpr const LogicalOperatorType TYPE = LogicalOperatorType::LOGICAL_DEPENDENT_JOIN;

public:
	explicit LogicalDependentJoin(JoinType type);

	//! The condition of the dependent join, if any
	unique_ptr<Expression> condition;
	//! The list of columns that have correlations with the right
	CorrelatedColumns correlated_columns;

	bool perform_delim = true;
	bool any_join = false;
	bool propagate_null_values = true;
};
```

Every claim in this section has one purpose: to pin down, with the actual
call chain, *why* the binder — not the optimizer — is the stage that builds
this node. It falls out entirely from ordinary scope resolution: a child
`Binder`'s `BindContext` genuinely does not contain `c`, the parent
`Binder`'s does, `BindInEnclosingScope` is the code path that walks outward
to find it, and a non-empty `correlated_columns` list on the child `Binder`
at the end of subquery binding is the one boolean (`IsCorrelated()`) that
routes `PlanSubquery` toward building a `LogicalDependentJoin` instead of a
plain, uncorrelated plan.

**This is exactly where Chapter 6 picks up.** A `LogicalDependentJoin` is
explicitly documented as transient — its own header calls it "planned but
not yet flattened," and states plainly that "this construct only exists
during planning and should not exist in the plan once flattening is
complete" (`logical_dependent_join.hpp:6-12`). What happens to it next —
`FlattenDependentJoins` (`src/planner/subquery/flatten_dependent_join.cpp`),
the correlated-column rewriting, and how it becomes ordinary joins (a
`LogicalComparisonJoin` tagged `LOGICAL_DELIM_JOIN`, which by default is then
rewritten into joins over materialized CTEs) before the optimizer's first
pass ever runs — is the
entire subject of Chapter 6, and this chapter deliberately stops at the
handoff: by the time `Binder::Bind(SQLStatement&)` returns a
`BoundStatement` for Query B, a `LogicalDependentJoin` sits in the tree in
place of `n_orders`'s subquery, still unflattened.

### 4.8.1 A subquery can refer to an outer `GROUP BY` expression

Query B's subquery refers to a plain outer *column*, `c.custkey`. Sometimes
a subquery wants an outer *expression* instead, one that the outer query
groups by. Here is the first query of
`test/sql/subquery/scalar/correlated_grouped_expression.test`:

```sql
SELECT o.created_at::date AS day, count(*) AS orders,
       (SELECT sum(s.is_open) FROM stores s WHERE s.day = o.created_at::date) AS open_stores
FROM orders o
GROUP BY 1
ORDER BY 1 NULLS LAST;
----
2024-01-01	2	2
2024-01-02	1	0
NULL	1	NULL
```

The outer query groups by `o.created_at::date`. After grouping, the column
`o.created_at` itself no longer exists, only the grouped value does. So
the subquery may not see `o.created_at`. It may only see
`o.created_at::date` as a whole. A binder that looked at the column
`o.created_at` on its own would find it is not a group and fail with a
`must be part of an aggregate function` error. Not every system accepts this
query: a comment in
`test/sql/subquery/scalar/test_grouped_correlated_subquery.test` notes that
"this used to fail in postgres but succeeds in sqlite". DuckDB accepts it,
and that test expects four `false` rows.

What makes it work is the check at the top of `ExpressionBinder::BindExpression`
(§4.4). Before it looks at the expression's class, a binder inside a
subquery calls `ResolveOuterGroupOwner`:

```cpp
// src/planner/expression_binder/scope_resolution.cpp:305-329
optional_idx ExpressionBinder::ResolveOuterGroupOwner(ParsedExpression &expr) {
	if (!CanMatchGroup(expr)) {
		return optional_idx();
	}
	// resolving the owner walks the expression, so only do so when some enclosing scope can match at all -
	// plain column groups are already matched in `ResolveColumn`
	bool any_compound_groups = false;
	for (idx_t scope = 1; scope < ScopeCount() && !any_compound_groups; scope++) {
		any_compound_groups = ScopeAt(scope).HasCompoundGroups();
	}
	if (!any_compound_groups) {
		return optional_idx();
	}
	// the expression only refers to a group of the scope that owns all of its columns, so that e.g. a
	// column of an inner scope that shadows the grouped one keeps the expression in the inner scope
	auto owner = ResolveExpressionOwner(expr, 0);
	if (!owner.IsValid() || owner.GetIndex() == 0) {
		return optional_idx();
	}
	// the columns of a nested subquery can resolve in other scopes than the expression's own columns
	if (!ScopeAt(owner.GetIndex()).MatchesGroupWithoutSubquery(expr)) {
		return optional_idx();
	}
	return owner;
}
```

In plain steps:

1. Skip expressions that can never equal a group, such as a plain column, a
   constant, or a subquery (`CanMatchGroup`). Plain column groups are
   already handled elsewhere.
2. Skip the work if no enclosing scope groups by anything other than plain
   columns (`HasCompoundGroups`). This keeps ordinary queries fast.
3. Find the innermost scope where the expression's columns resolve
   (`ResolveExpressionOwner`). If that is the subquery itself (depth 0),
   stop. An inner column with the same name wins.
4. Check that this outer scope really groups by the expression.

If all checks pass, `BindGroupInEnclosingScope`
(`scope_resolution.cpp:331-337`) binds a qualified copy of the expression in
the outer scope. The result is a correlated reference at depth 1 or more,
like `c.custkey` in Query B. From there, the normal path takes over:
`ExtractCorrelatedExpressions` records it, and the subquery is planned as a
correlated subquery (Chapter 6).

The same test file shows the rule working for other expression shapes:
`SELECT x + 1, (SELECT x + 1) FROM t GROUP BY 1` returns `0 0`, `2 2`,
`3 3`, `NULL NULL`. An expression that is only *part* of a larger one also
works: `(SELECT x::BIGINT * 10 + 1)` under `GROUP BY 1` (where column 1 is
`x::BIGINT`) returns `-9`, `11`, `21`, `NULL`.

## 4.9 Summary: what crossed the line from Chapter 3 to Chapter 4

Chapter 3 handed this stage a tree of `ParsedExpression`/`TableRef` nodes
holding nothing but strings and syntactic structure — no catalog contact, no
types, no casts. This chapter's `Binder`/`BindContext`/`ExpressionBinder`
machinery is what turns every one of those strings into something concrete:
table names into `TableCatalogEntry`s and `TableIndex`es, column names into
`ColumnBinding{table_index, column_index}` pairs (the only vocabulary
anything downstream will ever use again), function names into a specific
resolved overload with any needed implicit casts already spliced in, and —
because DuckDB fuses binding with logical-plan construction rather than
treating them as two stages — every one of those resolutions has, as its
direct byproduct, already built the corresponding `LogicalGet` /
`LogicalComparisonJoin` / `LogicalFilter` / `LogicalAggregate` node. The
`BoundStatement` this stage hands off already *is* the unoptimized logical
plan; Chapter 5 picks it up from there to examine the shape and vocabulary of
that tree in its own right, and Chapter 6 picks up the one loose thread this
chapter deliberately left dangling — the still-unflattened
`LogicalDependentJoin` sitting in Query B's plan. (One related detail from
§4.8.1: a subquery may also refer to an outer `GROUP BY` *expression*, not
only to an outer column; the binder turns that into an ordinary correlated
reference too.)
