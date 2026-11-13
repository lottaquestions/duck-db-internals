# Chapter 12 — Expression Evaluation

## 12.1 The problem this chapter solves

By the time execution (Chapter 11) hands a `PhysicalOperator` its input, most of the
hard thinking is over. The binder (Chapter 4) has already turned every name in the
query into a `ColumnBinding`, resolved every function call to a concrete catalog
entry, and produced a tree of `Expression` nodes with a fixed, known type at every
node. What's left is arithmetic: given a bound expression tree and a batch of rows,
produce the values that expression evaluates to, for every row in the batch, as
fast as possible — and then do it again, for the next batch, and the next, possibly
millions of times over the lifetime of one query.

Query A gives three concrete instances of this problem, each a little different:

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

- `l.shipdate` on its own is the simplest possible expression: a `BoundReferenceExpression`
  pointing at a specific column slot in the input `DataChunk` (Chapter 10). "Evaluating"
  it means handing back a reference to a `Vector` that's already sitting there — no
  computation at all.
- `l.shipdate < DATE '1998-01-01'` is a comparison: two children (a column reference
  and a folded constant) combined by an operator, producing a fresh boolean-typed
  `Vector`, one entry per input row. This is the expression this chapter follows all
  the way down to the machine-code level, because it's the cleanest worked example of
  the dispatch mechanism every other expression kind uses too.
- `SUM(l.extendedprice)` is not evaluated by the machinery in this chapter at all —
  aggregates have a separate evaluation path (accumulator state carried across many
  chunks, not a fresh output value per chunk) covered in Chapter 14. It's mentioned here
  only to draw the boundary precisely: §12.13 comes back to it.

The class responsible for the middle case — and for `BoundReferenceExpression`,
`BoundCaseExpression`, `BoundConjunctionExpression`, `BoundFunctionExpression`,
and every other bound expression kind that isn't an aggregate or a window function —
is `ExpressionExecutor`. This chapter is about how it works, all the way down to the
assembly-level reasons its inner loops look the way they do.

## 12.2 The `ExpressionExecutor` class

Its job, stated in its own header comment, is narrow and precise: run a set of
expressions against an input chunk and store the results in an output chunk.

```cpp
//! ExpressionExecutor is responsible for executing a set of expressions and storing the result in a data chunk
class ExpressionExecutor {
    friend class BoundIndex;
public:
    DUCKDB_API explicit ExpressionExecutor(ClientContext &context);
    DUCKDB_API ExpressionExecutor(ClientContext &context, const Expression *expression);
    DUCKDB_API ExpressionExecutor(ClientContext &context, const Expression &expression);
    DUCKDB_API ExpressionExecutor(ClientContext &context, const vector<unique_ptr<Expression>> &expressions);

    //! The expressions of the executor
    vector<const Expression *> expressions;
    //! The data chunk of the current physical operator, used to resolve column references and
    //! determines the output cardinality
    DataChunk *chunk = nullptr;
public:
    // ...
    //! Execute the set of expressions with the given input chunk and store the result in the output chunk
    DUCKDB_API void Execute(DataChunk *input, DataChunk &result);
};
```

(`src/include/duckdb/execution/expression_executor.hpp`; reproduced from the
research notes' condensed excerpt.) The real header at that path today carries a
larger public surface than this excerpt shows — `ExecuteExpression` (single-expression,
single-output-vector), `SelectExpression` (produce a `SelectionVector` of the rows for
which a single boolean expression is true, rather than a boolean `Vector`), and
`EvaluateScalar`/`TryEvaluateScalar` (fold a whole expression down to one `Value`,
used for e.g. constant-folding a `DATE '1998-01-01'` literal once instead of once
per chunk) — but the shape is the same: an `ExpressionExecutor` is a small,
short-lived object built around one or more `Expression` trees and a pointer to
"whatever chunk it's currently looking at." Nothing about it is query-specific in
the sense Chapter 13 cares about — it holds no generated code, only a pointer into
the same `Expression` tree the binder produced back in Chapter 4.

Every physical operator that has to touch a bound expression owns one of these.
`PhysicalFilter` owns one for the pushed-down residual predicate, `PhysicalProjection`
owns one per output column, `PhysicalHashJoin` owns one for its join condition. Each
operator calls `Execute` (or `ExecuteExpression`/`SelectExpression`) once per
`DataChunk` that flows through its pipeline (Chapter 11).

## 12.3 One file per expression kind

`ExpressionExecutor`'s implementation is split across one `.cpp` file per bound
expression kind, all living under `src/execution/expression_executor/`:

| File | Handles |
|---|---|
| `execute_reference.cpp` | `BOUND_REF` — a raw column reference; the "expression" is just an index into the input chunk |
| `execute_constant.cpp` | `BOUND_CONSTANT` — a literal value, materialized as a `CONSTANT_VECTOR` |
| `execute_conjunction.cpp` | `BOUND_CONJUNCTION` — `AND`/`OR` over child expressions |
| `execute_case.cpp` | `BOUND_CASE` — `CASE WHEN ... THEN ... ELSE ...` |
| `execute_comparison.cpp` | machinery shared by comparison operators — see §12.5, its role has moved since earlier DuckDB versions |
| `execute_operator.cpp` | `BOUND_OPERATOR` — `IS NULL`, `IS NOT NULL`, and similar unary/n-ary operators that aren't ordinary function calls; also a direct `Select` for `IS NULL`/`IS NOT NULL` (§12.4.1) |
| `execute_function.cpp` | `BOUND_FUNCTION` — scalar function calls, and (as §12.5 shows) also comparisons in the current source tree |
| `execute_lambda.cpp` | `BOUND_LAMBDA` — lambda bodies used by `list_transform` and friends |
| `execute_parameter.cpp` | `BOUND_PARAMETER` — prepared-statement `$1`-style placeholders |

This file list matches, one for one, the switch in `ExpressionExecutor::Execute`
(§12.4) — every `ExpressionClass` the switch dispatches on has exactly one file
implementing its `Execute(const BoundXExpression &, ...)` overload, and vice versa.
That's a useful thing to notice on its own: DuckDB's source layout mirrors the
dispatch structure directly enough that "which file handles this expression kind"
is never a search problem.

## 12.4 The confirmed dispatch mechanism, and how to characterize it precisely

The research notes for this book confirmed the mechanism by reading
`expression_executor.cpp` directly: a plain `switch (expr.GetExpressionClass())`,
appearing at least three times in the file (`Execute`'s state-initialization pass,
its main execution pass, and a `Select`-optimized pass), with each `case` calling a
dedicated overload:

```cpp
switch (expr.GetExpressionClass()) {
case ExpressionClass::BOUND_REF:      ...
case ExpressionClass::BOUND_CASE:     ...
case ExpressionClass::BOUND_CONJUNCTION: ...
case ExpressionClass::BOUND_CONSTANT: ...
case ExpressionClass::BOUND_FUNCTION: ...
case ExpressionClass::BOUND_LAMBDA:   ...
case ExpressionClass::BOUND_OPERATOR: ...
case ExpressionClass::BOUND_PARAMETER: ...
// ...
}
```

That's accurate as a summary, and it's worth seeing filled in against the actual
source tree examined for this book, because the full version pins down exactly
what "..." was standing in for:

```cpp
// src/execution/expression_executor.cpp:279-306
switch (expr.GetExpressionClass()) {
case ExpressionClass::BOUND_REF:
    Execute(expr.Cast<BoundReferenceExpression>(), state, sel, count, result);
    break;
case ExpressionClass::BOUND_CASE:
    Execute(expr.Cast<BoundCaseExpression>(), state, sel, count, result);
    break;
case ExpressionClass::BOUND_CONJUNCTION:
    Execute(expr.Cast<BoundConjunctionExpression>(), state, sel, count, result);
    break;
case ExpressionClass::BOUND_CONSTANT:
    Execute(expr.Cast<BoundConstantExpression>(), state, sel, count, result);
    break;
case ExpressionClass::BOUND_FUNCTION:
    Execute(expr.Cast<BoundFunctionExpression>(), state, sel, count, result);
    break;
case ExpressionClass::BOUND_LAMBDA:
    Execute(expr.Cast<BoundLambdaExpression>(), state, sel, count, result);
    break;
case ExpressionClass::BOUND_OPERATOR:
    Execute(expr.Cast<BoundOperatorExpression>(), state, sel, count, result);
    break;
case ExpressionClass::BOUND_PARAMETER:
    Execute(expr.Cast<BoundParameterExpression>(), state, sel, count, result);
    break;
default:
    throw InternalException("Attempting to execute expression of unknown type!");
}
```

Every branch does the same two things: cast the type-erased `Expression &` to its
concrete bound subtype (a `static_cast` under the hood — `GetExpressionClass()`
already told the switch which subtype it is, so this is free at runtime), then call
an overload of `Execute` that receives that concrete type. Each of those overloads
— one per row of the table in §12.3 — operates on a **whole `Vector` (or the whole
input `DataChunk`) at once**, producing one output `Vector` per call, rather than
being compiled into one fused function body that walks the expression tree row by
row for this specific query.

That is precisely a **textbook tree-walking interpreter with per-vector
(batch-at-a-time) evaluation, not per-row evaluation and not compiled/codegen
evaluation.** Every word of that sentence is load-bearing and worth restating
separately, because Chapter 13 leans on this exact framing:

- **Tree-walking interpreter** — there is a real, in-memory tree of `Expression`
  objects (built once, in Chapter 4), and evaluating it means recursively visiting
  that tree at runtime, the same tree, for every chunk. Nothing about the tree
  changes between chunks or between queries once bound.
- **Per-vector, not per-row** — the "interpreter loop" doesn't visit the tree once
  per row. It visits the tree once per `DataChunk` (up to 2,048 rows), and at each
  node it processes every row in the chunk in one call before moving to the parent
  node. This is what makes it a *vectorized* interpreter rather than the classical
  Volcano-style per-row `next()` interpreter most database textbooks describe first.
- **Not compiled/codegen** — the switch above is choosing between functions that
  already exist as machine code inside the running `duckdb` binary. Nothing is
  generated, assembled, or JIT-compiled for this specific query, this specific
  expression tree, or this specific comparison. §12.7 makes this precise; Chapter 13
  is devoted to proving it exhaustively across the whole engine, not just this one
  class.

### 12.4.1 The `Select` switch, and its case for `IS NULL`

The third switch, in `ExpressionExecutor::Select`, is smaller. `Select` does not
write a boolean `Vector`. It sorts the input rows into two lists of row numbers
(selection vectors): the rows where the predicate is true (`true_sel`) and the
rows where it is false (`false_sel`). Only a few expression kinds have their own
`Select`. All others go through `DefaultSelect`, which first computes a boolean
`Vector` and then turns it into the two lists.

```cpp
// src/execution/expression_executor.cpp:322-332
	switch (expr.GetExpressionClass()) {
	case ExpressionClass::BOUND_CONJUNCTION:
		return Select(expr.Cast<BoundConjunctionExpression>(), state, sel, count, true_sel, false_sel);
	case ExpressionClass::BOUND_FUNCTION:
		return Select(expr.Cast<BoundFunctionExpression>(), state, sel, count, true_sel,
		              false_sel); // NOLINT: c-style cast
	case ExpressionClass::BOUND_OPERATOR:
		return Select(expr.Cast<BoundOperatorExpression>(), *state, sel, count, true_sel, false_sel);
	default:
		return DefaultSelect(expr, state, sel, count, true_sel, false_sel);
	}
```

The `BOUND_OPERATOR` case handles `IS NULL` and `IS NOT NULL` directly; every
other operator falls back to `DefaultSelect`
(`src/execution/expression_executor/execute_operator.cpp:66-84`). It evaluates
the child once, then reads the child's *validity mask* (the bit per row that
says "this value is NULL or not") straight into the two lists. No boolean
`Vector` is built in between. The inner loop uses the same branch-free trick
that §12.10 explains:

```cpp
// src/execution/expression_executor/execute_operator.cpp:25-42
template <bool HAS_VALID_SEL, bool HAS_NULL_SEL>
static idx_t SelectNullLoop(const VectorValidityIterator &entries, const SelectionVector &sel, idx_t count,
                            optional_ptr<SelectionVector> valid_sel, optional_ptr<SelectionVector> null_sel) {
	idx_t valid_count = 0, null_count = 0;
	for (idx_t i = 0; i < count; i++) {
		const auto row_idx = sel.get_index(i);
		const auto valid = entries.IsValid(i);
		if (HAS_VALID_SEL) {
			valid_sel->set_index(valid_count, row_idx);
			valid_count += valid;
		}
		if (HAS_NULL_SEL) {
			null_sel->set_index(null_count, row_idx);
			null_count += !valid;
		}
	}
	return HAS_VALID_SEL ? valid_count : count - null_count;
}
```

If the child cannot hold any NULL, or is a single constant, there is not even a
loop over validity: all rows go into one list. `test/sql/filter/null_selection.test`
checks queries such as `WHERE i%3=1 AND n IS NULL` and
`WHERE (i%3=1 AND n IS NULL) OR (i%5=2 AND n IS NOT NULL)` over columns of
many types.

## 12.5 A source discrepancy worth stopping on: comparisons are `BOUND_FUNCTION` now

Notice what's *missing* from the switch above: there is no
`case ExpressionClass::BOUND_COMPARISON:`. That's not an oversight in the excerpt —
it's the whole switch, and it genuinely has no comparison case. This is exactly the
kind of thing the front matter warned about: DiDi's course material [9] (built against
DuckDB 1.4) walks a chain that starts at a distinct `BoundComparisonExpression`
node with its own `ComparisonExecutor::Execute<OP>` handler. That class and that
node no longer exist in `v2.0-cyanoptera`, and it's worth seeing exactly how they
were retired, because the mechanism that replaced them is more general, not less.

`src/include/duckdb/planner/expression/bound_comparison_expression.hpp` today reads:

```cpp
namespace duckdb {
//! The BoundComparisonExpression has been moved to a function expression
//! This class exists only as a set of helper methods
struct BoundComparisonExpression {
    static bool IsComparison(ExpressionType type);
    static bool IsComparison(const Expression &expr);
    // ...
    static unique_ptr<Expression> Create(ExpressionType type, unique_ptr<Expression> left,
                                         unique_ptr<Expression> right);
    //! Create a comparison, casting both sides to a common type - throws if they cannot be compared
    static unique_ptr<Expression> Bind(ClientContext &context, ExpressionType type, unique_ptr<Expression> left,
                                       unique_ptr<Expression> right);
    // ...
};
} // namespace duckdb
```
(`src/include/duckdb/planner/expression/bound_comparison_expression.hpp:13-29`, excerpted)

`BoundComparisonExpression` is no longer an `Expression` subclass at all — it's a
plain `struct` of static helpers, and `Create` is where the real change shows up
(`src/function/scalar/comparison/comparison.cpp:181-190`):

```cpp
unique_ptr<Expression> BoundComparisonExpression::Create(ExpressionType type, unique_ptr<Expression> left,
                                                         unique_ptr<Expression> right) {
    vector<unique_ptr<Expression>> children;
    children.push_back(std::move(left));
    children.push_back(std::move(right));

    auto result = make_uniq<BoundFunctionExpression>(BoundScalarFunction(GetComparisonFunction(type)),
                                                     std::move(children), nullptr, true);
    return std::move(result);
}
```

`Create` builds the node but does not check or cast the argument types. The
binder uses its sibling, `Bind`, which does both
(`src/function/scalar/comparison/comparison.cpp:192-201`):

```cpp
unique_ptr<Expression> BoundComparisonExpression::Bind(ClientContext &context, ExpressionType type,
                                                       unique_ptr<Expression> left, unique_ptr<Expression> right) {
	vector<unique_ptr<Expression>> children;
	children.push_back(std::move(left));
	children.push_back(std::move(right));

	BoundScalarFunction function(GetComparisonFunction(type));
	BindScalarFunctionInput input(context, function, children);
	function.GetBindCallback()(input);
	return make_uniq<BoundFunctionExpression>(std::move(function), std::move(children), nullptr, true);
}
```

The type work lives in the comparison function's own *bind callback*,
`ComparisonBind<TYPE>` (`comparison.cpp:77-94`). It finds a common type for
both sides, or throws "Cannot compare values of type ... - an explicit cast is
required". Then it adds a cast to that type on each side, and adds a collation
if the type has one:

```cpp
	for (auto &argument : arguments) {
		argument = BoundCastExpression::AddCastToType(context, std::move(argument), input_type,
		                                              input_type.id() == LogicalTypeId::ENUM);
		ExpressionBinder::PushCollation(context, argument, input_type);
	}
```

`l.shipdate < DATE '1998-01-01'` is bound by `ExpressionBinder::BindExpression`
(`src/planner/binder/expression/bind_comparison_expression.cpp:171-187`), which calls
`CreateBoundComparison`, which calls `Bind` — meaning the node that
ends up in the plan is a `BoundFunctionExpression` wrapping the catalog's `<` scalar
function, with `ExpressionClass::BOUND_FUNCTION`.

Because the casts live in the bind callback, a comparison called *by name*,
like `"="(1, 1.0)` or `"<"(1, 2::BIGINT)`, gets the same casts as `1 = 1.0`.
`test/sql/function/operator/test_comparison_function_call_types.test` shows
this, including `SELECT "<"(MAP{1:2}, [1])` failing with "Cannot compare values
of type MAP(INTEGER, INTEGER) and type INTEGER[]". The function also has its own
serialize and deserialize callbacks. Their comment says the arguments of a
deserialized comparison are already cast, so it should not be bound again.

The old comparison node survives
only as `LegacyBoundComparisonExpression` (`ExpressionClass::LEGACY_BOUND_COMPARISON`,
`src/planner/expression/legacy_bound_comparison_expression.cpp`), kept around purely
so that old serialized plans and SQL-export code (`bound_expression_sql_exporter.cpp`,
`serialize_expression.cpp`) can still be read — it is never produced by the live
binder path and is never seen by `ExpressionExecutor::Execute`'s switch. That's why
the switch in §12.4 has no comparison case: as far as expression evaluation is
concerned, **a comparison is just a scalar function call now**, indistinguishable
in kind from `sqrt(x)` or `lower(s)`.

`execute_comparison.cpp` still exists at the path DiDi's slides cite, but its role
changed along with everything else: today it holds only the *selection-vector*
overloads of `VectorOperations::Equals/NotEquals/GreaterThan/...` — the ones that
take a `SelectionVector` and return `true_sel`/`false_sel` partitions rather than
writing a boolean `Vector`. Those are used when a comparison is being evaluated as
a *filter* (§12.10–§12.12), not as an ordinary value-producing expression. The
value-producing overloads — the ones a comparison used *as an expression* goes
through — live in `src/common/vector_operations/comparison_operators.cpp`, the
same path DiDi's slides cite; that file didn't move.

## 12.6 The full dispatch chain, verified end to end: int64 `Equals`

With that discrepancy on the table, here is the real chain, walked from the top,
for the case DiDi's slides use as their worked example: two `BIGINT` (`int64`)
columns compared with `=`, appearing as an ordinary boolean-valued expression (in a
`SELECT` list, inside a `CASE`, or wherever else a comparison is used as a value
rather than pushed down as a table filter — the filter path is its own, related
story, picked up in §12.10–§12.12).

```
ExpressionExecutor::Execute(expr, state, sel, count, result)
   │  expression_executor.cpp:279   — switch on expr.GetExpressionClass()
   ▼
case BOUND_FUNCTION:
   Execute(expr.Cast<BoundFunctionExpression>(), ...)
   │  execute_function.cpp:232     — evaluate children into an argument DataChunk,
   │                                 then call the bound function's callback
   ▼
expr.Function().Execute(arguments, *state, result)
   │  comparison.cpp:13  — ComparisonFunction<ExpressionType::COMPARE_EQUAL>
   │  `if constexpr` picks the Equals branch at DuckDB-build time
   ▼
VectorOperations::Equals(left, right, result)
   │  comparison_operators.cpp:223-227
   ▼
TryPrimitiveComparisonExecute<duckdb::Equals>(left, right, result)
   │  comparison_operators.cpp:131-187
   │  switch (left.GetType().InternalType())        ← DECISION 1: value type
   │  case PhysicalType::INT64:
   ▼
BinaryExecutor::Execute<int64_t, int64_t, bool, duckdb::Equals>(left, right, result)
   │  binary_executor.hpp:203-208, 165-175, 155-162
   │  wraps OP in a BinaryScalarAdapter, calls the shared executor
   ▼
ScalarExecutor::Execute<ExecutePolicy, bool, Adapter, int64_t, int64_t>(inputs, result, count, adapter)
   │  scalar_executor.hpp:852-857 → ExecuteInternal, scalar_executor.hpp:379-467
   │  GetInputProfile(inputs) inspects VectorType at RUNTIME
   │  switch (profile.constant_mask)                ← DECISION 2: vector representation
   ▼
ExecuteFlat<PRESERVE_VALIDITY, CONSTANT_MASK, bool, Adapter, int64_t, int64_t>(...)
   │  scalar_executor.hpp:136-191
   │  the branch-free hot loop — see below
```

**Decision 1 — value type — is made by `TryPrimitiveComparisonExecute`'s switch**
(`comparison_operators.cpp:135-186`), which the executor reaches after the scalar
function dispatch has already told it "this is an Equals comparison" and before any
per-row work happens:

```cpp
template <class OP>
static bool TryPrimitiveComparisonExecute(const Vector &left, const Vector &right, Vector &result) {
    D_ASSERT(left.GetType().InternalType() == right.GetType().InternalType());
    switch (left.GetType().InternalType()) {
    case PhysicalType::BOOL:
    case PhysicalType::INT8:
        BinaryExecutor::Execute<int8_t, int8_t, bool, OP>(left, right, result);
        return true;
    // ...
    case PhysicalType::INT64:
        BinaryExecutor::Execute<int64_t, int64_t, bool, OP>(left, right, result);
        return true;
    // ...
    case PhysicalType::VARCHAR:
        BinaryExecutor::Execute<string_t, string_t, bool, OP>(left, right, result);
        return true;
    default:
        return false;
    }
}
```
(`src/common/vector_operations/comparison_operators.cpp:131-187`)

Once `PhysicalType::INT64` is chosen, `BinaryExecutor::Execute<int64_t, int64_t,
bool, OP>` is a fully concrete function — `LEFT_TYPE`, `RIGHT_TYPE`, `RESULT_TYPE`,
and `OP` are all fixed as C++ template parameters, not runtime values, from this
line onward all the way down to the innermost loop.

**Decision 2 — vector representation — is made by `ScalarExecutor::ExecuteInternal`**
(`scalar_executor.hpp:379-467`), and this is the part that has changed shape the
most since DiDi's slides, while keeping the same underlying idea. Where DuckDB 1.4's
`BinaryExecutor` had separate named functions for "both flat," "left constant," and
"right constant" (`ExecuteGeneric`/`ExecuteGenericLoop` and friends), the current
`ScalarExecutor` generalizes this to `N`-ary functions (unary, binary, ternary) with
a runtime *profile* computed once per call:

```cpp
template <size_t N, size_t... Is>
static inline InputProfile GetInputProfile(const std::array<VectorRef, N> &inputs, std::index_sequence<Is...>) {
    InputProfile result;
    auto classify = [&](auto input_index) {
        constexpr idx_t INPUT_INDEX = decltype(input_index)::value;
        auto vector_type = inputs[INPUT_INDEX].get().GetVectorType();
        if (vector_type == VectorType::CONSTANT_VECTOR) {
            if constexpr (INPUT_INDEX < 64) {
                result.constant_mask |= uint64_t(1) << INPUT_INDEX;
            }
            result.any_constant_null |= ConstantVector::IsNull(inputs[INPUT_INDEX].get());
        } else {
            result.all_constant = false;
            result.all_flat_or_constant &= vector_type == VectorType::FLAT_VECTOR;
        }
    };
    (classify(std::integral_constant<idx_t, Is> {}), ...);
    return result;
}
```
(`src/include/duckdb/common/vector_operations/scalar_executor.hpp:42-61`)

`profile.constant_mask` is a bitmask — bit `i` set means "input `i` is a
`CONSTANT_VECTOR`." `ExecuteInternal` then does exactly what DiDi's slides describe,
just over a runtime bitmask instead of a fixed set of named cases:

```cpp
if constexpr (POLICY::SPECIALIZE_FLAT && N <= 3) {
    if (profile.all_flat_or_constant) {
        // ...
        switch (profile.constant_mask) {
        case 0:
            ExecuteFlat<POLICY::PRESERVE_RESULT_VALIDITY, 0, RESULT_TYPE, ADAPTER, ARGS...>(
                inputs, result, count, adapter, indices);
            return;
        case 1:
            /* ... ExecuteFlat<..., 1, ...> — left input constant ... */
        case 2:
            /* ... ExecuteFlat<..., 2, ...> — right input constant ... */
        // ...
        }
    }
}
ExecuteGeneric<...>(inputs, result, count, adapter, indices); // fallback: dictionary / sequence / other representations
```
(`src/include/duckdb/common/vector_operations/scalar_executor.hpp:402-466`, elided
for the cases in between)

For `l.shipdate < DATE '1998-01-01'`, `l.shipdate` is a flat column vector and the
folded date literal is a `CONSTANT_VECTOR`, so `profile.constant_mask == 2`
(bit 1 set — the right-hand input is constant) and the runtime `switch` lands on
`ExecuteFlat<..., 2, ...>` — a template instantiated specifically for "left flat,
right constant." Each `case` in that switch is calling a **different, already-compiled
function**; the switch does not decide what code to run in some abstract sense, it
picks which one of several concrete machine-code routines — one per representation
combination — DuckDB already has sitting in its binary.

Just before that switch, `ExecuteInternal` handles the case where *every*
input is constant (`profile.all_constant`). Then the answer is one value, and
the result is a `CONSTANT_VECTOR`. How that one value is computed depends on a
build option:

```cpp
// src/include/duckdb/common/vector_operations/scalar_executor.hpp:385-401
		if (profile.all_constant) {
#if !DUCKDB_SMALLER_BINARY(scalar_executor_constant)
			ExecuteConstant<RESULT_TYPE, ADAPTER, ARGS...>(inputs, result, count, profile.any_constant_null, adapter,
			                                               indices);
#else
			// the generic path computes the one row, which then becomes the constant
			PrepareConstantResult(result);
			ExecuteGeneric<POLICY::SPECIALIZE_NULLABLE_GENERIC_SELECTIONS,
			               POLICY::SPECIALIZE_NON_NULL_GENERIC_SELECTIONS, POLICY::PRESERVE_RESULT_VALIDITY,
			               RESULT_TYPE, ADAPTER, ARGS...>(inputs, result, 1, adapter, indices);
			result.SetVectorType(VectorType::CONSTANT_VECTOR);
			if (result.size() != count) {
				FlatVector::SetSize(result, count);
			}
#endif
			return;
		}
```

`DUCKDB_SMALLER_BINARY(feature)` is a switch that is set when DuckDB itself is
compiled (`src/include/duckdb/common/smaller_binary.hpp`). In a normal build it
is 0, and the special `ExecuteConstant` template is kept. In a build made to be
small (CMake option `SMALLER_BINARY`, `CMakeLists.txt:704`),
`scalar_executor_constant` can be turned on. Then the `ExecuteConstant` specialization is not compiled at all, and the
generic path computes the single row instead. The result is the same; the
binary is smaller and this case is a little slower. This is the same trade
§12.7 describes: each template specialization costs binary size, so DuckDB
lets a builder drop some of them. Chapter 13 comes back to this.

**The hot loop itself**, `ExecuteFlat`, is where all of this pays off:

```cpp
template <bool PRESERVE_RESULT_VALIDITY, uint64_t CONSTANT_MASK, class RESULT_TYPE, class ADAPTER, class... ARGS,
          size_t... Is>
static void ExecuteFlat(const std::array<VectorRef, sizeof...(ARGS)> &inputs, Vector &result, idx_t count,
                        ADAPTER &adapter, std::index_sequence<Is...> indices) {
    // ...
    auto result_data = FlatVector::GetDataMutable<RESULT_TYPE>(result);
    auto input_data = std::make_tuple(FlatVector::GetData<ARGS>(inputs[Is].get())...);
    auto &result_validity = PrepareFlatResultValidity<...>(inputs, result, count, indices);
    // ...
    if (result_validity.CanHaveNull()) {
        // ... a validity-entry-at-a-time loop, elided ...
    } else {
        for (idx_t row = 0; row < count; row++) {
            result_data[row] = adapter.Operation(result_validity, row,
                                                 std::get<Is>(input_data)[InputIndex<Is, CONSTANT_MASK>(row)]...);
        }
    }
}
```
(`src/include/duckdb/common/vector_operations/scalar_executor.hpp:136-191`, the
no-null branch shown in full)

Look at `InputIndex<Is, CONSTANT_MASK>(row)`:

```cpp
template <idx_t INPUT_INDEX, uint64_t CONSTANT_MASK>
static inline idx_t InputIndex(idx_t row) {
    if constexpr (CONSTANT_MASK & (uint64_t(1) << INPUT_INDEX)) {
        return 0;
    } else {
        return row;
    }
}
```
(`scalar_executor.hpp:63-70`)

`if constexpr` is evaluated by the compiler, not at runtime — `CONSTANT_MASK` is a
template parameter, fixed by which specialization got selected in §12.6's decision
2. For the `l.shipdate < DATE '...'` instantiation (`CONSTANT_MASK == 2`, so
`INPUT_INDEX == 1` is the constant side), the generated machine code for the
constant argument *always reads index `0`* — there is no per-row branch checking
"is this argument constant this time," because the compiler already deleted that
branch when it compiled this specific specialization. The loop that actually runs is,
after the compiler inlines everything, indistinguishable from hand-written code that
assumed from the start "left is a flat `int64_t*` (or `int32_t*` for `DATE`), right
is a single constant, write a `bool` per row" — with zero tests inside the loop body
for type, nullability (in the no-null branch), or representation. That is the entire
point of deciding decisions 1 and 2 *before* the loop: the loop itself has nothing
left to decide.

## 12.7 The template-specialization point, precisely

It is worth being pedantic about *when* the specialization in §12.6 happens,
because getting this exactly right is the single most important thing this chapter
sets up for Chapter 13.

`BinaryExecutor::Execute<int64_t, int64_t, bool, duckdb::Equals>` and
`ScalarExecutor::ExecuteFlat<..., 2, bool, Adapter, int64_t, int64_t>` are C++
function templates. Every concrete combination actually needed anywhere in DuckDB
— every `(operator, value type, vector-representation pattern)` triple that some
call site somewhere instantiates — gets turned into its own, separate, fully
optimized machine-code routine **by clang or gcc, once, when the `duckdb` binary
itself is compiled.** This happens on the machine that builds DuckDB (a developer's
laptop, or a CI runner producing a release artifact), it happens exactly once no
matter how many times DuckDB is later invoked, and it happens with *zero knowledge*
of any specific SQL query — the compiler doesn't know, and doesn't need to know,
that `l.shipdate < DATE '1998-01-01'` will ever be typed by anyone. It only knows
that *somewhere* in the `duckdb` source, `TryPrimitiveComparisonExecute<Equals>`'s
switch contains a line saying `BinaryExecutor::Execute<int64_t, int64_t, bool,
Equals>(...)`, and that's enough for the compiler to manufacture that one
specialization as object code and link it into the binary.

At query time, none of that compilation happens again. What happens at query time
is exactly two runtime `switch` statements (§12.6's decisions 1 and 2) — ordinary
branches over ordinary runtime values (a `PhysicalType` enum, a `constant_mask`
bitmask) — each of which *selects among the finitely many already-compiled
specializations* and jumps to one. No new code is written, assembled, or JIT-compiled;
the "decision" a query makes is a pointer-sized jump to a function address, nothing
more.

Contrast this with what a **compiling query engine** does instead. DiDi's course
material draws the comparison explicitly, citing Umbra [14] as the example:

> A different breed of DBMS—for example Umbra—generates and compiles C++/IR/machine
> code at query translation time. These query-specific code fragments are then
> linked with the DBMS and invoked at runtime.
>
> — [9, fn. 3]

A system built this way, faced with `l.shipdate < DATE '1998-01-01'`, does not
switch among a fixed menu of pre-existing functions. It generates brand-new machine
code (or an intermediate representation that a JIT then turns into machine code)
*specifically for this expression, in this query*, at the moment the query is
compiled — every time that query is compiled, even if it's the exact same SQL text
run a second later. That new code typically doesn't need `InputIndex<Is,
CONSTANT_MASK>` at all, because the compiler generating it already knows, at
codegen time, exactly which operand is constant for this query — it can bake in a
literal `1998-01-01` directly rather than reading it from a `Vector` at all.

Both designs end up with "specialized, branch-free code in the hot loop" — that
half of the sentence is true of both architectures, and it's the half people
usually remember, which is exactly why it's easy to conflate them. The difference
that actually matters is *when* the specialization work happens and *how* the
specific specialization gets selected:

```
DuckDB:   specialize at DuckDB-BUILD time (templates, compiled once)
          select   at  QUERY time        (runtime switch/jump to existing code)

Umbra:    specialize at QUERY-COMPILE time (fresh codegen, every query)
          select   —   there is nothing to select; the generated code IS the query
```

`switch (profile.constant_mask) { case 2: ExecuteFlat<..., 2, ...>(...); }` and "emit
a new machine-code function for this WHERE clause" are not two names for the same
mechanism — they are different mechanisms that happen to produce code with a
similar shape once you're standing inside the hot loop looking at the assembly.
Chapter 13 returns to this distinction with exhaustive, engine-wide evidence
(profiling a running query for JIT symbols, checking for an LLVM/IR dependency,
tracing what actually gets called across an entire `EXPLAIN ANALYZE`); this
chapter's job was just to make sure the distinction itself is airtight before you
see it applied more broadly.

## 12.8 Low-level tight-loop engineering: from `-O2` to SIMD

Specialization only helps if what it produces is actually fast once you're inside
the loop. DiDi's course [9] spends real time on this at the assembly level, using a
small vector-subtract routine as the running example — not DuckDB source, but the
kind of code that ends up inside a `BinaryExecutor`/`ScalarExecutor` specialization
once the type and representation decisions are made:

```cpp
// illustrative — not DuckDB source; a minimal microbenchmark in DiDi's style
void SubtractLoop(const uint8_t *__restrict a, const uint8_t *__restrict b,
                   uint8_t *__restrict out, size_t n) {
    for (size_t i = 0; i < n; i++) {
        out[i] = a[i] - b[i];
    }
}
```

Compiled at `-O2` with no further hints, this becomes a straightforward loop: load,
subtract, store, increment, compare, branch back — one iteration of real work per
one iteration of loop overhead (the increment/compare/branch). DiDi walks through
raising this in three further steps, each addressing that ratio of useful work to
loop-control overhead:

1. **Manual 4-way unrolling.** Instead of one `out[i] = a[i] - b[i]` per iteration,
   the loop body computes four (`out[i]`, `out[i+1]`, `out[i+2]`, `out[i+3]`) and
   advances `i` by 4. The loop-control instructions (compare, branch) now amortize
   over four units of real work instead of one, and — just as importantly — the
   four subtractions are independent of each other, which exposes instruction-level
   parallelism the CPU's out-of-order execution can actually exploit (four
   independent loads can be in flight at once, rather than each waiting on the
   previous iteration's store to retire).
2. **Compiler auto-unrolling.** Modern optimizing compilers will often perform
   exactly this transformation on their own at higher optimization levels, without
   being asked — the manual version exists pedagogically, to show what the compiler
   is doing, not because a DBMS author needs to write it by hand.
3. **SIMD vectorization.** Given the `__restrict__` qualifiers, the compiler can go
   further than unrolling scalar work — it can pack multiple elements into one
   vector register and operate on all of them with a single instruction. For
   byte-sized elements (as in the `uint8_t` example above) a 128-bit SSE register
   holds 16 of them at once, so one `movdqu` (unaligned 128-bit load) per operand
   plus one `psubb`/`psubd`-family instruction plus one `movdqu` store processes 16
   elements of real work per loop iteration — the same loop-control overhead as the
   scalar version, applied to sixteen times the data.

The reason `__restrict__` is not optional here is aliasing analysis, and it's worth
being precise about why. Without it, the compiler must assume `a`, `b`, and `out`
might overlap in memory — for instance, that `out` and `a` are the same buffer, or
that `out` starts one element into `a`. If that were true, vectorizing the loop
(computing sixteen outputs "at once" before any of them are written back) could
produce a different answer than the scalar version: a later scalar iteration might
have read a value that an earlier iteration hadn't overwritten yet, and the
vectorized version breaks that ordering guarantee. The compiler has no way to prove
the pointers don't overlap just by looking at the function signature, so at `-O2`
without `__restrict__` it conservatively refuses to vectorize (or inserts a runtime
overlap check and falls back to a scalar loop when the check fails, which defeats
the purpose). `__restrict__` is the programmer telling the compiler, as a binding
promise, "these three pointers never alias" — which is exactly the kind of fact a
DBMS author *does* know (a `BinaryExecutor` call is handed freshly allocated
`Vector` buffers, never overlapping input and output) but a general-purpose
compiler, looking only at the function's type signature, cannot infer on its own.
`ExecuteFlat`'s own debug-build assertion in §12.6 —
`ASSERT_RESTRICT(data, data + count, result_data, result_data + count)` — exists
for exactly this reason: to catch, in a debug build, any call site that violates the
no-aliasing promise the release build's vectorized code depends on.

## 12.9 Memory access patterns and prefetching

A vectorized inner loop is only as fast as the memory system feeding it. DuckDB's
columnar, chunk-at-a-time layout (Chapter 10) is deliberately friendly to the two
access patterns modern CPUs handle best on their own:

- **Forward or backward sequential scans** — reading `l.shipdate[0]`,
  `l.shipdate[1]`, `l.shipdate[2]`, ... in order — are *predictable*. The CPU's
  hardware prefetcher detects the stride and starts pulling upcoming cache lines
  into L1/L2 before the load instruction that needs them even executes, largely
  hiding the ~500-cycle latency of a round trip to main memory. Every one of the
  loops in §12.6–§12.8 is exactly this pattern.
- **Irregular, jumping access** — following a chain of pointers, or indexing
  through a permutation/selection vector whose next index can't be predicted from
  the current one — defeats the hardware prefetcher, because there's no fixed
  stride for it to detect. This is precisely the pattern a hash-table probe (hash
  join, hash aggregation) produces: the next bucket to visit depends on a hash of
  the current row's key, which looks random from the memory system's point of
  view.

For code that has to walk that second, unpredictable pattern, DuckDB's execution
code sometimes issues **explicit software prefetch instructions** — a
`prefetcht1`-class instruction that asks the CPU to start pulling a specific cache
line into cache *before* the code that will need it actually runs, giving the ~500
cycles of RAM latency time to overlap with other work instead of stalling the
pipeline waiting for a load. This only helps when the address to prefetch can be
computed a little ahead of when it's needed (e.g., a hash join can compute the next
row's target bucket before it's finished processing the current row) — it's a
targeted tool for the specific case where the access pattern is irregular but not
entirely unknown in advance, not a general substitute for keeping loops
sequential wherever possible.

## 12.10 Branch prediction and the branchless filter

The `WHERE l.shipdate < DATE '1998-01-01'` predicate, evaluated over a chunk,
eventually has to turn a per-row `true`/`false` result into a decision: keep this
row, or drop it. The naive way to write that is an `if`:

```cpp
// illustrative — the "obvious" branching filter
idx_t o = 0;
for (idx_t i = 0; i < count; i++) {
    if (col[i] < val) {
        res[o] = i;
        o++;
    }
}
```

Every iteration of this loop contains a data-dependent branch — one whose outcome
depends on the actual data, not on the loop structure — and modern CPUs are deeply
pipelined (roughly five stages, simplifying): fetching and starting to execute the
*next* instruction begins before the current branch's condition is even known. The
CPU's branch predictor guesses which way the branch will go and speculatively keeps
executing down that path; static heuristics (absent better information) tend to
assume backward branches — like a loop's own back-edge — are taken, and forward
branches — like an `if` skipping over a block — are not. When the guess is wrong,
everything speculatively executed past that branch has to be discarded and the
pipeline refilled from the correct address: a flush costing on the order of 15
cycles. For a predicate like `l.shipdate < DATE '1998-01-01'` against real data, the
branch's outcome is essentially a coin flip per row whenever selectivity is near
50% — the worst case for a predictor that can only track recent history, since
there's no stable pattern to learn.

The **branchless** alternative removes the conditional control flow entirely:

```cpp
// illustrative — the branchless equivalent
idx_t o = 0;
for (idx_t i = 0; i < count; i++) {
    res[o] = i;
    o += (col[i] < val);
}
```

This is a small enough trick that it's worth checking carefully, because its
correctness isn't obvious on first read. `res[o] = i` executes on *every*
iteration, unconditionally — including iterations where the predicate is false.
That looks wasteful, or even wrong: surely a failing row shouldn't be written into
the result at all? The resolution is that `o` — the write position — only advances
when the predicate held: `col[i] < val` evaluates to `0` or `1` (as an integer),
and `o += (...)` adds that `0` or `1` to `o`. When the predicate is false, `o`
doesn't move, so the *next* iteration's `res[o] = i` simply overwrites the same slot
`res[o]` was just (uselessly) written to, with the next candidate index. The write
for a failing row is never actually observed — it's always clobbered by whatever
gets written to that same slot next, whether that's the next passing row or another
failing row's now-discarded write. Only the final value of `o` after the loop
matters, and it correctly counts (and `res[0..o)` correctly lists) exactly the rows
for which the predicate held. No branch, no misprediction, no pipeline flush — the
loop's runtime per row is now constant, independent of how many rows pass.

DiDi's course [9] backs this up with a runtime-vs-selectivity chart: the classic
`if`-based filter's per-row cost rises as selectivity approaches 50% (where the
branch is least predictable) and falls back off toward either extreme (where the
branch becomes easy to predict — almost-always-taken or almost-never-taken), producing
a hump-shaped curve peaking around 50% selectivity. The branchless version's curve
is flat across the entire selectivity range — its cost genuinely doesn't depend on
how many rows pass, only on how many rows there are.

This exact idiom shows up, essentially verbatim, in the selection-vector machinery
`ScalarExecutor::Select` builds on. Its `StaticSelectionSink` — used, among other
places, by the comparison functions' `Select` callback (§12.5, §12.12) when it can
prove at compile time whether a true-selection and/or false-selection output is
wanted — is:

```cpp
inline void Append(bool comparison_result, idx_t result_idx) {
    if constexpr (HAS_TRUE_SELECTION) {
        true_selection[true_count] = UnsafeNumericCast<sel_t>(result_idx);
        true_count += comparison_result;
    }
    if constexpr (HAS_FALSE_SELECTION) {
        false_selection[false_count] = UnsafeNumericCast<sel_t>(result_idx);
        false_count += !comparison_result;
    }
}
```
(`src/include/duckdb/common/vector_operations/scalar_executor.hpp:480-489`)

`true_selection[true_count] = result_idx;` unconditionally, then
`true_count += comparison_result;` — the identical pattern, for the identical
reason, running inside DuckDB's real filter-evaluation path today, not just in a
pedagogical example.

## 12.11 Mixed-mode selection for compound predicates

A `WHERE` clause is often a conjunction of several conditions, not one — and the
individual conjuncts are rarely equally cheap or equally selective. A natural
refinement, once you have both a branching and a branchless implementation
available, is to mix them: evaluate the cheaper, more selective conjunct first
using an ordinary branch (since a highly selective predicate is easy for the branch
predictor to learn — it's almost always false, say), and evaluate the remaining,
less selective or more expensive conjuncts using the branchless accumulation
pattern from §12.10, since their outcome is closer to a coin flip and a branch on
them would mispredict often.

DiDi's material is explicit that this particular refinement is a better fit for a
different kind of engine than DuckDB:

> This approach particularly fits DBMSs that generate code from SQL
> queries—which DuckDB does not do.
>
> — [9, "Mixed-Mode Selection"]

The reason is direct: deciding *which* conjunct should get the branch treatment and
which should get the branchless treatment, and generating a loop body with that
specific mix baked in, is naturally a per-query decision — it depends on which
specific conjuncts this specific query has and (ideally) their actual observed
selectivity for this specific predicate. A system that generates fresh code per
query can bake that decision directly into the generated loop. DuckDB, evaluating a
fixed, pre-compiled tree-walking interpreter (§12.4), can't rewrite its own loop
bodies per query — but as the next section shows, it isn't stuck evaluating
conjuncts in a fixed order either.

## 12.12 `AdaptiveFilter`: runtime conjunct reordering, still fully interpreted

What DuckDB *can* do without generating code is decide, at runtime and separately
for each query, in **what order** to evaluate a set of conjuncts — and revise that
order as it observes how selective each one actually turns out to be on real data.
That's `AdaptiveFilter` (`src/execution/adaptive_filter.cpp`,
`src/include/duckdb/execution/adaptive_filter.hpp`).

```cpp
struct AdaptiveFilterState {
    TimePoint monotonic_start;
};

class AdaptiveFilter {
public:
    explicit AdaptiveFilter(const Expression &expr);
    explicit AdaptiveFilter(const TableFilterSet &table_filters, vector<idx_t> filter_global_pos = {});

public:
    void AdaptRuntimeStatistics(double duration);
    bool Remap(const TableFilterSet &new_filters, vector<idx_t> new_ids);

    AdaptiveFilterState BeginFilter() const;
    void EndFilter(AdaptiveFilterState state);

    const vector<idx_t> &GetPermutation() const {
        return permutation;
    }
    // ...
private:
    vector<idx_t> permutation;
    vector<idx_t> swap_likeliness;
    // ...
};
```
(`src/include/duckdb/execution/adaptive_filter.hpp`)

It owns a `permutation` — the order in which conjuncts should be tried this
chunk — and a `swap_likeliness` per adjacent pair of positions in that order.
`AdaptRuntimeStatistics`, called after timing a batch of chunks, does the actual
adaptation:

```cpp
void AdaptiveFilter::AdaptRuntimeStatistics(double duration) {
    iteration_count++;
    runtime_sum += duration;
    // ...
    if (!warmup) {
        if (observe && iteration_count == observe_interval) {
            // keep swap if runtime decreased, else reverse swap
            auto trial_mean = runtime_sum / static_cast<double>(iteration_count);
            if (prev_mean - trial_mean <= 0) {
                // reverse swap because runtime didn't decrease
                std::swap(permutation[swap_idx], permutation[swap_idx + 1]);
                if (swap_likeliness[swap_idx] > 1) {
                    swap_likeliness[swap_idx] /= 2;
                }
            } else {
                // keep swap because runtime decreased, reset likeliness
                swap_likeliness[swap_idx] = 100;
            }
            observe = false;
            iteration_count = 0;
            runtime_sum = 0.0;
        } else if (!observe && iteration_count == execute_interval) {
            // save old mean to evaluate swap
            prev_mean = runtime_sum / static_cast<double>(iteration_count);
            // pick a random adjacent pair to (maybe) swap, weighted by swap_likeliness
            auto random_number = generator.NextRandomInteger(1, NumericCast<uint32_t>(right_random_border));
            swap_idx = random_number / 100;
            idx_t likeliness = random_number - 100 * swap_idx;
            if (swap_likeliness[swap_idx] > likeliness) {
                std::swap(permutation[swap_idx], permutation[swap_idx + 1]);
                observe = true;
            }
            iteration_count = 0;
            runtime_sum = 0.0;
        }
    }
    // ...
}
```
(`src/execution/adaptive_filter.cpp:116-187`, elided for the warmup branch)

The scheme is a small randomized hill-climb: periodically try swapping two
adjacent conjuncts in the trial order, measure whether the chunks that follow run
faster or slower on average, and keep the swap if it helped, revert it (with reduced
future likelihood) if it didn't. Over enough chunks, conjuncts that turn out to be
cheap and highly selective — ones that reject most rows on their own — migrate
toward the front of `permutation`, so later, more expensive conjuncts run over
fewer surviving rows. This is used both for a `BoundConjunctionExpression`'s own
children and, via the second constructor, for the conjuncts inside a `TableFilterSet`
(pushed-down scan filters, including the `l.shipdate < DATE '1998-01-01'` predicate
after `filter_pushdown` moves it into `lineitem`'s scan — Chapter 7).

It's worth being explicit about what this is *not*, because it's easy to
over-read: `AdaptiveFilter` never generates, rewrites, or recompiles any code. It
only permutes the *order* in which a fixed set of already-compiled, already-chosen
`Execute`/`Select` calls (§12.4–§12.6) get invoked. Each individual conjunct is
still evaluated by exactly the same tree-walking, per-vector `ExpressionExecutor`
machinery this whole chapter has been describing — the interpreter is unchanged;
only the *schedule* the interpreter follows is adapted at runtime, per query, based
on that query's own observed data. It's a genuinely adaptive optimization, and it's
genuinely still interpretation — those two facts sit comfortably together, and
Chapter 13 cites `AdaptiveFilter` only as one more piece of evidence for "DuckDB
gets its adaptivity from runtime bookkeeping around a fixed interpreter, not from
generating new code," without re-explaining the mechanism itself.

### 12.12.1 `AND` and `OR`: two smaller speed-ups

Two more details touch conjunctions. Neither changes the interpreter model.
Both are about doing less work per row.

**`OR` as a filter: merge, don't sort.** When `OR` filters rows, `Select`
tries each child in the order `AdaptiveFilter` picked. Each child only sees
the rows that no earlier child accepted. So each child returns its passing
rows as a separate, sorted run. Row numbers in the final `true_sel` must be in
ascending order. Instead of appending each run and sorting the whole list at the
end, the code merges each new run into the list right away:

```cpp
// src/execution/expression_executor/execute_conjunction.cpp:60-81
//! Merges the first "source_count" entries of "source" into the sorted first "target_count" entries of "target"
static void MergeSortedSelection(SelectionVector &target, idx_t target_count, SelectionVector &source,
                                 idx_t source_count) {
	for (idx_t i = 1; i < source_count; i++) {
		if (source.get_index(i - 1) > source.get_index(i)) {
			source.Sort(source_count);
			break;
		}
	}
	// merge from the back so the merge can be done in-place
	idx_t target_idx = target_count;
	idx_t source_idx = source_count;
	idx_t result_idx = target_count + source_count;
	while (source_idx > 0) {
		if (target_idx > 0 && target.get_index(target_idx - 1) > source.get_index(source_idx - 1)) {
			target.set_index(--result_idx, target.get_index(--target_idx));
		} else {
			target.set_index(--result_idx, source.get_index(--source_idx));
		}
	}
}
```

A small example. Say `v < 250` accepts rows 2 and 9, and then `v >= 750`
accepts rows 4 and 7 out of the rest. The list is `[2, 9]`. Merging `[4, 7]`
from the back gives `[2, 4, 7, 9]`, with no sort. Merging two sorted lists
takes one pass; sorting takes more. `test/sql/conjunction/or_select_order.test`
checks that `WHERE v < 250 OR v >= 750` returns rows in order, for example
`SELECT list(i) = list(i ORDER BY i) FROM ints WHERE v < 250 OR v >= 750`
must be `true`.

**`AND`/`OR` as a value: 64 rows of NULL logic at once.** When `AND` or `OR`
produces a value (in a `SELECT` list, an aggregate argument, or an `IN` list,
which DuckDB evaluates as an `OR` of comparisons), SQL's three-valued logic
applies. `NULL AND FALSE` is `FALSE`, but `NULL AND TRUE` is `NULL`. Working
this out one row at a time takes a few branches per row. So for two flat
inputs, DuckDB does it for 64 rows with one bitwise formula over the validity bits
(`src/common/vector_operations/boolean_operators.cpp:16-60`). The rule for
`AND`:

```cpp
// src/common/vector_operations/boolean_operators.cpp:136-139
	//! The result is valid if both sides are valid, or if either side is a valid FALSE
	static validity_t ResultValidity(validity_t left, validity_t right, validity_t left_valid, validity_t right_valid) {
		return (left_valid & right_valid) | (left_valid & ~left) | (right_valid & ~right);
	}
```

Each argument is a 64-bit word: one bit per row. `left` holds the values
(1 = `TRUE`), and `left_valid` holds "not NULL". `OR` uses the same formula
with `left` and `right` in place of `~left` and `~right` (a valid `TRUE`
decides an `OR`). The NULL mask of the result is only written for a block of
64 rows that really contains a NULL. Inputs with no NULLs still use a plain
loop. `test/sql/function/operator/test_conjunction_nulls.test` compares `AND`,
`OR`, three-way chains and `BETWEEN` against the same logic written with
`CASE`. On 50 million rows with one thread, an `IN` list of 8 constants takes
0.940 s this way, against 1.725 s with the row-at-a-time loop.

`CASE` uses a similar word-at-a-time trick: when a branch's result is a
NULL constant, `FillNulls` (`src/execution/expression_executor/execute_case.cpp:96-115`)
clears the validity bits of a whole 64-row word at once, instead of one
`SetInvalid` call per row.

## 12.13 Query A, traced: `WHERE` and projection through this machinery

Putting the chapter's pieces together against Query A's own expressions:

**`l.shipdate < DATE '1998-01-01'`.** The binder (§12.5) turns this into a
`BoundFunctionExpression` around the `<` scalar function, with children a
`BoundReferenceExpression` (`l.shipdate`) and a `BoundConstantExpression`
(the folded date literal). The optimizer's `filter_pushdown` pass (Chapter 7) moves
it from a standalone `LogicalFilter` down into `lineitem`'s `LogicalGet`, where the
physical plan generator (Chapter 9) turns it into a `TableFilter` evaluated during
the scan itself — using the comparison function's **`Select` callback**
(`ComparisonSelect<ExpressionType::COMPARE_LESSTHAN>`, registered via
`comparison_fun.SetSelectCallback(...)` in `comparison.cpp`), which calls
`VectorOperations::LessThan(left, right, sel, count, true_sel, false_sel)` — the
selection-vector overload from §12.5's `execute_comparison.cpp`, feeding
`BinaryExecutor::Select` → `ScalarExecutor::Select` → the branchless
`StaticSelectionSink`/`RuntimeSelectionSink` machinery from §12.10, with
`AdaptiveFilter` (§12.12) deciding what order to try it in relative to any other
pushed-down conjuncts on `lineitem`. If, instead, this predicate had stayed above
the scan as an ordinary boolean-valued residual — say, folded into a `CASE`
expression, or otherwise unable to be pushed into a `TableFilter` — it would
instead go through the **value-producing** path traced in full in §12.6:
`ExpressionExecutor::Execute` → `BOUND_FUNCTION` → `ComparisonFunction<COMPARE_LESSTHAN>`
→ `VectorOperations::LessThan` → `TryPrimitiveComparisonExecute<GreaterThan>`
(operands flipped, per the "NOLINTNEXT: flip right/left" comment in
`comparison_operators.cpp:248-253`) → `BinaryExecutor::Execute<int32_t, int32_t,
bool, GreaterThan>` (DATE's physical type is `INT32`, not `INT64` — confirmed at
`src/common/types.cpp:58-61`) → `ScalarExecutor::ExecuteFlat<..., CONSTANT_MASK=2,
...>` — the same dispatch skeleton §12.6 walked for `int64` `Equals`, with a
different operator and a different physical width chosen by the same two
compile-time/runtime decisions.

**`o.custkey` and `sum(l.extendedprice)` in the `SELECT` list.** `o.custkey` is the
simplest case in §12.1: a `BoundReferenceExpression`, handled by
`execute_reference.cpp`, which does no arithmetic at all — it hands back a
reference to the already-existing input `Vector`. `SUM(l.extendedprice)` is where
this chapter's boundary is: it is *not* dispatched through
`ExpressionExecutor::Execute`'s switch at all, because `BOUND_AGGREGATE` isn't one
of that switch's cases (§12.4's table lists every case the switch actually has, and
aggregates aren't among them). Aggregates carry their own per-group accumulator
state that has to survive across every chunk of `lineitem` that `PhysicalHashAggregate`
sees, not just the current one — a fundamentally different evaluation shape than
"read a chunk, produce a same-sized output vector, forget everything." Chapter 14
covers that machinery; this chapter's `ExpressionExecutor` plays no role in it
beyond evaluating the aggregate's *argument* expression (`l.extendedprice` itself,
here just a `BoundReferenceExpression`) before handing the resulting vector to the
aggregate's own update function.

## 12.14 Bridge to Chapter 13 and Chapter 14

Every mechanism this chapter described — the `switch` in §12.4, the type-then-representation
decision pair in §12.6, the compile-time template instantiation in §12.7, even the
runtime reordering in `AdaptiveFilter` (§12.12) — is doing the same kind of work: get
specialized, branch-free code into the hot loop, without ever generating new machine
code for a specific query. §12.7 already stated the core distinction against a
compiling engine like Umbra as precisely as this chapter can state it on its own.
Chapter 13 takes that same claim and tries to break it — searching the whole engine,
not just `ExpressionExecutor`, for any sign of an LLVM dependency, a JIT, or
per-query codegen — and comes back with an exhaustive, source-cited case that the
claim holds everywhere, not just in the one class this chapter focused on. Chapter 14
picks up the thread §12.13 left open: what `SUM(l.extendedprice)` actually does,
once you stop treating aggregates as "out of scope for this chapter" and start
asking how a hash aggregate accumulates a running sum per `custkey` across every
chunk of `lineitem` that flows past it.
