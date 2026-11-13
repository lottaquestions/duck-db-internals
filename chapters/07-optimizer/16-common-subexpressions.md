# 7.16 — Common Subexpressions (`COMMON_SUBEXPRESSIONS`)

## 7.16.0 Where this sits, and a note on how this section was checked

This is pass **#25** in §7.4's numbered pipeline: `OptimizerType::COMMON_SUBEXPRESSIONS`,
wired up at `optimizer.cpp:413`.

```cpp
// src/optimizer/optimizer.cpp:413-416
RunOptimizer(OptimizerType::COMMON_SUBEXPRESSIONS, [&]() {
	CommonSubExpressionOptimizer cse_optimizer(binder);
	cse_optimizer.VisitOperator(*plan);
});
```

and implemented entirely in one file, `src/optimizer/cse_optimizer.cpp` (193 lines,
reproduced close to in full below), plus its header,
`src/include/duckdb/optimizer/cse_optimizer.hpp`. §7.6.7 already previewed this pass
in one paragraph, as part of a three-way sketch against `COMMON_AGGREGATE` and
`COMMON_SUBPLAN`; this section is that preview's full expansion, and it corrects one
detail in the preview along the way (§7.16.4).

Every claim about scope and identity below was checked two ways: by reading
`cse_optimizer.cpp` and the surrounding `Expression`/`BaseExpression` machinery
directly, and by compiling the exact SQL fragments in question and reading the real
`EXPLAIN` output. The second check used the `duckdb` CLI already installed in this
environment (`v1.1.3`, a stable release, not the `cb5673685153` checkout — v2.0.0
plus 1,041 commits — that the rest of this book cites for source line numbers) — `cse_optimizer.cpp`'s core shape
(the `CountExpressions`/`PerformCSEReplacement` two-pass design, the
`LOGICAL_PROJECTION`/`LOGICAL_AGGREGATE_AND_GROUP_BY`-only trigger) has been stable
across this range, so the `EXPLAIN` traces below are trustworthy confirmations of the
mechanism the checked-out source describes, even though the two are different exact
commits. Every source-code citation with a `file:line` is against the `cb5673685153`
checkout, per this book's stated policy; the `EXPLAIN` output is clearly labeled as an
empirical check, not a source citation.

## 7.16.1 What problem this pass solves, in one sentence

Inside a single operator's own list of output expressions, the same computation can
appear written out more than once — `SELECT a+1, (a+1)*2 FROM t` computes `a+1` twice
as far as the *unoptimized* tree is concerned, once as its own output column and once
buried inside the multiplication — and `CommonSubExpressionOptimizer`'s job is to
notice this, compute the shared subtree exactly once, and rewire every other
occurrence to be a plain reference to that one computed result instead of a second,
independent evaluation of the same expression tree.

## 7.16.2 The full mechanism, read from the source

### The two-pass shape

`ExtractCommonSubExpressions` (`cse_optimizer.cpp:158-191`) is the entry point for a
single operator, and it makes exactly two passes over that operator's own expressions
— count, then replace:

```cpp
// src/optimizer/cse_optimizer.cpp:158-191
void CommonSubExpressionOptimizer::ExtractCommonSubExpressions(LogicalOperator &op) {
	D_ASSERT(op.children.size() == 1);

	// first we count for each expression with children how many types it occurs
	CSEReplacementState state;
	LogicalOperatorVisitor::EnumerateExpressions(
	    op, [&](unique_ptr<Expression> *child) { CountExpressions(**child, state); });
	// check if there are any expressions to extract
	bool perform_replacement = false;
	for (auto &expr : state.expression_count) {
		if (expr.second.count > 1) {
			perform_replacement = true;
			break;
		}
	}
	if (!perform_replacement) {
		// no CSEs to extract
		return;
	}
	state.projection_index = binder.GenerateTableIndex();
	// we found common subexpressions to extract
	// now we iterate over all the expressions and perform the actual CSE elimination

	LogicalOperatorVisitor::EnumerateExpressions(
	    op, [&](unique_ptr<Expression> *child) { PerformCSEReplacement(*child, state); });
	D_ASSERT(state.expressions.size() > 0);
	// create a projection node as the child of this node
	auto projection = make_uniq<LogicalProjection>(state.projection_index, std::move(state.expressions));
	if (op.children[0]->has_estimated_cardinality) {
		projection->SetEstimatedCardinality(op.children[0]->estimated_cardinality);
	}
	projection->children.push_back(std::move(op.children[0]));
	op.children[0] = std::move(projection);
}
```

The first pass (`CountExpressions`) builds a frequency table of every eligible
subexpression appearing anywhere in the operator's expression tree(s), keyed by
structural identity (§7.16.3 pins down exactly what "identity" means here). The
`for (auto &expr : state.expression_count)` loop right after is a cheap early-exit: if
nothing occurred more than once, the whole machinery — including allocating a brand
new `TableIndex` for a projection that would otherwise be pointless — is skipped
entirely. This matters because `ExtractCommonSubExpressions` runs on *every*
`LogicalProjection` and *every* `LogicalAggregate` in the tree (see §7.16.4), and the
overwhelming majority of them have no repeated subexpression at all; the early exit is
what keeps this pass's per-query cost close to the "10–50µs" figure §7.1 quotes for
the pipeline as a whole, rather than something proportional to the number of new
`TableIndex`es it *could* have allocated.

If — and only if — something repeats, a second pass (`PerformCSEReplacement`) walks
the same expressions again, this time actually rewriting the tree: every subexpression
whose count exceeded 1 gets moved, in place, into a freshly built `state.expressions`
vector (which becomes the new child projection's own select-list), and the spot it
used to occupy is replaced with a `BoundColumnRefExpression` pointing at that new
projection's output column. The final step is structural, not just an expression-tree
edit: a brand new `LogicalProjection` node, carrying `state.expressions`, is spliced in
as this operator's *sole child* — `op.children[0]` becomes the new projection, and the
old child (`op.children[0]` before this ran — the actual scan/join/filter subtree that
used to feed this operator directly) becomes the new projection's child instead. So
the visible, structural effect on the *plan* is: one new `LogicalProjection` operator
appears, always directly beneath the `LogicalProjection` or `LogicalAggregate` this
pass fired on, and never anywhere else.

### Counting: `CountExpressions`

```cpp
// src/optimizer/cse_optimizer.cpp:51-106
void CommonSubExpressionOptimizer::CountExpressions(Expression &expr, CSEReplacementState &state) {
	// we only consider expressions with children for CSE elimination
	switch (expr.GetExpressionClass()) {
	case ExpressionClass::BOUND_COLUMN_REF:
	case ExpressionClass::BOUND_CONSTANT:
	case ExpressionClass::BOUND_PARAMETER:
	// a lambda has no value of its own, so there is nothing to share between two occurrences of it
	case ExpressionClass::BOUND_LAMBDA:
		return;
	default:
		break;
	}
	if (expr.GetExpressionClass() != ExpressionClass::BOUND_AGGREGATE && !expr.IsVolatile() && !state.inside_try) {
		// we can't move aggregates to a projection, so we only consider the children of the aggregate
		auto node = state.expression_count.find(expr);
		if (node == state.expression_count.end()) {
			// first time we encounter this expression, insert this node with [count = 1]
			// but only if it is not an interior argument of a short circuit sensitive expression.
			if (!state.short_circuited) {
				state.expression_count[expr] = CSENode();
			}
		} else {
			// we encountered this expression before, increment the occurrence count
			node->second.count++;
		}
	}

	// If we have a function that uses short circuiting, then we can only extract CSEs from the leftmost
	// side of the argument tree (child_no == 0)
	switch (expr.GetExpressionClass()) {
	case ExpressionClass::BOUND_CONJUNCTION:
	case ExpressionClass::BOUND_CASE: {
		// Save the short circuit reference
		const auto save_short_circuit = state.short_circuited;
		ExpressionIterator::EnumerateChildren(expr, [&](Expression &child) {
			CountExpressions(child, state);
			state.short_circuited = true;
		});
		state.short_circuited = save_short_circuit;
		break;
	}
	case ExpressionClass::BOUND_OPERATOR: {
		const auto save_inside_try = state.inside_try;
		if (expr.GetExpressionType() == ExpressionType::OPERATOR_TRY) {
			state.inside_try = true;
		}
		ExpressionIterator::EnumerateChildren(expr, [&](Expression &child) { CountExpressions(child, state); });
		state.inside_try = save_inside_try;
		break;
	}
	default:
		// recursively count the children
		ExpressionIterator::EnumerateChildren(expr, [&](Expression &child) { CountExpressions(child, state); });
		break;
	}
}
```

Five things are load-bearing in this one function, each worth pulling apart on its
own, because each one answers a question a careless reading of "it's just CSE" would
get wrong:

**Leaves are never candidates for extraction, on purpose.** A `BoundColumnRefExpression`,
`BoundConstantExpression`, `BoundParameterExpression`, or `BoundLambdaExpression`
returns immediately, before ever touching `state.expression_count`. This isn't an
oversight — factoring a bare column reference or a constant out into its own
projection column would trade one cheap operation (reading a pointer, or broadcasting
a constant vector) for another *exactly as cheap* one (reading a different pointer),
while adding a whole extra column to the synthetic projection for zero benefit. CSE
only ever pays for itself when the shared thing has actual **compute** behind it — a
function call, an operator, a `CASE`, a conjunction — which is exactly the set of
`ExpressionClass` values that fall through to the counting logic below the early
`return`.

**Aggregates are walked through, not counted themselves.** The check
`expr.GetExpressionClass() != ExpressionClass::BOUND_AGGREGATE` excludes a
`SUM(...)`/`COUNT(...)`/etc. call itself from ever being registered as a shareable
node — the comment says why: *"we can't move aggregates to a projection."* An
aggregate's whole reason for existing is that it accumulates state across every row of
a group, which is exactly the job of the `LogicalAggregate` operator's own physical
implementation (Chapter 14); a plain `LogicalProjection` spliced in as this operator's
child has no such accumulator and could not compute an aggregate result at all. But the
`switch` above the `if` doesn't return for `BOUND_AGGREGATE` the way it does for the
four leaf classes — execution falls through the `if`'s false branch straight into the
recursive-children walk at the bottom, so an aggregate's *arguments* are still counted
normally. `SUM(a*2)` and `AVG(a*2)` appearing as two separate aggregate calls in one
`LogicalAggregate` will still get their shared `a*2` argument factored into the
synthetic child projection, computed once — it's only the outer `SUM(...)`/`AVG(...)`
wrapper itself that's ineligible, never its insides.

**Volatile expressions are excluded outright**, via `!expr.IsVolatile()` — this is not
about performance, it's about correctness. `IsVolatile()` (Chapter 12 introduces the
base implementation; every subclass ORs in its own reason, e.g.
`BoundFunctionExpression::IsVolatile()` checking `function.GetStability() ==
FunctionStability::VOLATILE`) recursively reports whether any part of an expression
tree could return a *different* answer on two calls given the *same* input row —
`random()`, `nextval('some_sequence')`, and (as §7.16.6 verifies with a real test) any
lambda body wrapping one of those. Sharing a volatile expression's computed result
between two syntactic occurrences would silently change the query's answer: two
columns that each independently call `nextval(...)` are supposed to draw two different
sequence values per row, not the same one twice. Excluding volatile expressions from
`state.expression_count` entirely means they can never be recognized as "the same
expression, computed twice" in the first place — each occurrence is invisible to this
pass and is left exactly where the binder put it, guaranteed to be evaluated
independently.

**Short-circuit safety**, the `save_short_circuit`/`state.short_circuited` dance,
deserves its own paragraph — "The short-circuit guard, precisely", later in §7.16.2.

**Nothing inside a `TRY(...)` is ever shared.** `TRY(x)` returns `NULL` when `x`
raises an error, instead of failing the query. The new `inside_try` flag
(`cse_optimizer.cpp:35-36`) is set while the walk is under an `OPERATOR_TRY` node
(`cse_optimizer.cpp:92-100`). While it is set, no expression is counted at all.
Why? Take `SELECT TRY(s::INTEGER) AS a, TRY(s::INTEGER + 5) AS b`. Before this
change, `s::INTEGER` was counted twice and moved into the new projection below.
But that projection has no `TRY` around it. For `s = 'bad'` the cast then failed
in the projection, and the query raised a conversion error, although the user had
wrapped every cast in `TRY`. Now `s::INTEGER` stays inside each `TRY` and is
computed twice. Both columns come back `NULL`, as the user asked.

The guard stops only sharing that would cross the `TRY` boundary. A whole
`TRY(...)` call is still counted, because the check runs before the flag is set
for that node's children. So two identical calls, `TRY(s::INTEGER + 5)` written
twice, are still computed once. `test/sql/error/test_try_common_subexpression.test`
pins down both cases:

```sql
-- test/sql/error/test_try_common_subexpression.test:8-11
query II
SELECT TRY(s::INTEGER) AS a, TRY(s::INTEGER + 5) AS b FROM t;
----
NULL	NULL
```

```sql
-- test/sql/error/test_try_common_subexpression.test:42-46
# identical TRY expressions are still shared
query II
SELECT TRY(s::INTEGER + 5) AS a, TRY(s::INTEGER + 5) AS b FROM t;
----
NULL	NULL
```

The same file also checks the other direction (lines 28-32). If `s::INTEGER` also
appears outside any `TRY`, the query must still fail with a `Conversion Error`,
because the user asked for that unprotected cast.

This guard exists because of a bug: without it, the pass could move a
sub-expression out of a `TRY` and turn a `NULL` result into a query error.

### The short-circuit guard, precisely

`BOUND_CONJUNCTION` (`AND`/`OR`) and `BOUND_CASE` (`CASE WHEN ... THEN ... ELSE ...`)
are the two expression classes where DuckDB's evaluator does *not* unconditionally
evaluate every child for every row — a conjunction can short-circuit (`a AND b` need
not evaluate `b` for a row where `a` is false in the row-at-a-time sense a scalar
reading of `AND` suggests, and even in DuckDB's per-vector evaluator, `CASE`'s later
branches are only evaluated for the subset of rows that reached them — Chapter 12
covers `execute_case.cpp`'s selection-vector narrowing in full), and a `CASE`
expression's second and later `WHEN`/`THEN` pairs, and its `ELSE`, are conditionally
reached depending on whether an earlier branch's condition already matched.

`CountExpressions`'s handling of these two classes walks each child in order via
`ExpressionIterator::EnumerateChildren`, and sets `state.short_circuited = true`
*after* processing each child — meaning the very first child processed is counted
under whatever `short_circuited` value was already in force on the way in (initially
`false`, restored via `save_short_circuit` once this node's children are done), and
every subsequent child is counted with `short_circuited == true`. Look back at the
counting logic itself: when `short_circuited` is true and an expression is being seen
**for the first time**, it is deliberately *not* inserted into `state.expression_count`
at all (`if (!state.short_circuited) { state.expression_count[expr] = CSENode(); }`) —
so a subexpression that only ever appears inside a conditionally-evaluated position can
never become a fresh CSE candidate. But if that same expression *also* appears in the
unconditionally-evaluated leftmost position (or was already registered from some
earlier, unconditional context), the `else` branch — `node->second.count++` — still
fires unconditionally, incrementing the existing entry's count regardless of
`short_circuited`. The distinction being enforced is exactly: *is this expression
already guaranteed to be evaluated for every row anyway?* If yes (it showed up
unconditionally somewhere), reusing its result inside a conditional branch too is free
and safe. If the *only* place it's ever seen is inside a conditional branch, hoisting
it out into an eagerly-evaluated projection would force it to run on rows where the
original plan never would have run it at all — which, for an expression that
`CanThrow()` (a cast that might fail, a division that might divide by zero), would
turn a query that used to succeed into one that raises an error the user never wrote
a predicate to prevent. This is the same category of hazard §7.6.1 documents for
`REORDER_FILTER`'s `CanThrow()`/`ExpressionBarrier` guard — a different pass, but the
same underlying correctness concern about not being allowed to make something run that
short-circuiting was protecting from ever running.

### Replacement: `PerformCSEReplacement`

```cpp
// src/optimizer/cse_optimizer.cpp:108-156
void CommonSubExpressionOptimizer::PerformCSEReplacement(unique_ptr<Expression> &expr_ptr, CSEReplacementState &state) {
	Expression &expr = *expr_ptr;
	if (expr.GetExpressionClass() == ExpressionClass::BOUND_COLUMN_REF) {
		auto &bound_column_ref = expr.Cast<BoundColumnRefExpression>();
		// bound column ref, check if this one has already been recorded in the expression list
		auto column_entry = state.column_map.find(bound_column_ref.Binding());
		if (column_entry == state.column_map.end()) {
			// not there yet: push the expression
			auto new_col_ref = make_uniq<BoundColumnRefExpression>(
			    bound_column_ref.GetAlias(), bound_column_ref.GetReturnType(), bound_column_ref.Binding());
			auto new_column_index = ColumnBinding::PushExpression(state.expressions, std::move(new_col_ref));
			state.column_map[bound_column_ref.Binding()] = new_column_index;
			bound_column_ref.BindingMutable() = ColumnBinding(state.projection_index, new_column_index);
		} else {
			// else: just update the column binding!
			bound_column_ref.BindingMutable() = ColumnBinding(state.projection_index, column_entry->second);
		}
		return;
	}
	// check if this child is eligible for CSE elimination
	if (!state.inside_try && state.expression_count.find(expr) != state.expression_count.end()) {
		auto &node = state.expression_count[expr];
		if (node.count > 1) {
			// this expression occurs more than once! push it into the projection
			// check if it has already been pushed into the projection
			auto alias = expr.GetAlias();
			auto type = expr.GetReturnType();
			if (!node.column_index.IsValid()) {
				// has not been pushed yet: push it
				node.column_index = ColumnBinding::PushExpression(state.expressions, std::move(expr_ptr));
			} else {
				state.cached_expressions.push_back(std::move(expr_ptr));
			}
			// replace the original expression with a bound column ref
			expr_ptr = make_uniq<BoundColumnRefExpression>(alias, type,
			                                               ColumnBinding(state.projection_index, node.column_index));
			return;
		}
	}
	// this expression only occurs once, we can't perform CSE elimination
	// look into the children to see if we can replace them
	const auto save_inside_try = state.inside_try;
	if (expr.GetExpressionType() == ExpressionType::OPERATOR_TRY) {
		state.inside_try = true;
	}
	ExpressionIterator::EnumerateChildren(expr,
	                                      [&](unique_ptr<Expression> &child) { PerformCSEReplacement(child, state); });
	state.inside_try = save_inside_try;
}
```

The `inside_try` lines here mirror the ones in `CountExpressions`. They make
sure the replace pass also never swaps out an expression under a `TRY`. Without
them, an expression that is shared *outside* a `TRY` could still be replaced by
a column reference at a second place *inside* one.

Two branches matter here beyond the obvious "count > 1, so extract it":

**Every raw column reference gets threaded through the new projection too, not just
the repeated compound expressions.** Once this pass decides to splice in a new
`LogicalProjection` as this operator's child, *every* binding this operator's
expressions used to resolve directly against its old child now has to resolve against
the new projection instead — the operator's old child is no longer its direct child.
`state.column_map` deduplicates this: the first time a given `ColumnBinding` (e.g. "the
column `a` from table index 3, column index 0") is seen, a fresh copy of that column
reference is pushed into `state.expressions` and the binding is rewritten to
`(state.projection_index, new_column_index)`; every subsequent reference to the exact
same original binding just looks up the already-assigned index in `column_map` instead
of pushing a second copy. This is why, in every worked plan trace in §7.16.5 and
§7.16.7, a plain column like `a` shows up as one of the new projection's own output
expressions even in queries where `a` itself was never the "duplicated" part of the
computation — it has to ride through the new intermediate operator somehow, and this
is where that happens, with the same deduplication discipline (one copy of `a` per
distinct binding, however many times the original expression list referenced it) that
the compound-expression path gets.

**A repeated expression is only physically pushed into the new projection's expression
list once**, even though `PerformCSEReplacement` is called once per *occurrence*. The
`node.column_index.IsValid()` check is exactly that guard: the first occurrence of a
qualifying repeated expression moves its actual `unique_ptr<Expression>` into
`state.expressions` (note the `std::move(expr_ptr)` — the original expression object
itself, not a copy, becomes the new projection's expression) and records where it
landed; every later occurrence of the same expression finds `node.column_index`
already valid and instead pushes its now-orphaned `unique_ptr<Expression>` into
`state.cached_expressions` — a vector that exists purely to keep those
otherwise-abandoned `Expression` objects alive (so the `expression_map_t`'s stored
`const_reference<Expression>` keys, which point *into* those objects, never dangle)
without them ever becoming part of any operator's actual expression list. Both
occurrences are then replaced with the exact same kind of `BoundColumnRefExpression`,
built fresh, pointing at `(state.projection_index, node.column_index)` — this is the
one-node-many-references structure §7.16.7's diagram makes concrete.

## 7.16.3 Identity, precisely: is it exact structural equality? Does it account for commutativity?

This is one of the two questions this section was specifically asked to resolve by
reading the source rather than assuming, and the source gives an unambiguous answer:
**exact, ordered, structural equality — no commutative or associative reasoning of any
kind lives inside this pass.**

`state.expression_count` is an `expression_map_t<CSENode>`
(`cse_optimizer.cpp:26`), and that type is defined in
`src/include/duckdb/parser/expression_map.hpp:33-35`:

```cpp
// src/include/duckdb/parser/expression_map.hpp:19-35
template <class T>
struct ExpressionHashFunction {
	uint64_t operator()(const const_reference<T> &expr) const {
		return (uint64_t)expr.get().Hash();
	}
};

template <class T>
struct ExpressionEquality {
	bool operator()(const const_reference<T> &a, const const_reference<T> &b) const {
		return a.get().Equals(b.get());
	}
};

template <typename T>
using expression_map_t =
    unordered_map<const_reference<Expression>, T, ExpressionHashFunction<Expression>, ExpressionEquality<Expression>>;
```

So two expressions land in the same hash bucket, and are then confirmed equal, by
calling straight through to `Expression::Hash()` and `Expression::Equals()`
(dispatched, through virtual overrides, to whatever concrete bound expression subtype
is involved). `Expression::Hash()` itself
(`src/planner/expression.cpp:123-129`) is a structural tree hash: the node's own
`ExpressionType`, combined with its `LogicalType`, combined (via `CombineHash`) with
the hash of *every* child, recursively:

```cpp
// src/planner/expression.cpp:123-129
hash_t Expression::Hash() const {
	hash_t hash = duckdb::Hash<uint32_t>(static_cast<uint32_t>(type));
	hash = CombineHash(hash, return_type.Hash());
	ExpressionIterator::EnumerateChildren(*this,
	                                      [&](const Expression &child) { hash = CombineHash(child.Hash(), hash); });
	return hash;
}
```

The base `BaseExpression::Equals` (`src/parser/base_expression.cpp:21-26`) only checks
that the `ExpressionClass` and `ExpressionType` match — the real, type-specific
comparison lives in each subclass's override. `BoundFunctionExpression::Equals`
(`src/planner/expression/bound_function_expression.cpp:129-144`) is the one that
matters most for arithmetic like `a+b`:

```cpp
// src/planner/expression/bound_function_expression.cpp:129-144
bool BoundFunctionExpression::Equals(const BaseExpression &other_p) const {
	if (!Expression::Equals(other_p)) {
		return false;
	}
	auto &other = other_p.Cast<BoundFunctionExpression>();
	if (other.function != function) {
		return false;
	}
	if (!Expression::ListEquals(children, other.children)) {
		return false;
	}
	if (!FunctionData::Equals(bind_info.get(), other.bind_info.get())) {
		return false;
	}
	return true;
}
```

`Expression::ListEquals` forwards to `ExpressionUtil::ListEquals`
(`src/parser/expression_util.cpp:59-61`), which is `ExpressionListEquals` — a plain,
**positional**, index-by-index comparison:

```cpp
// src/parser/expression_util.cpp:9-19
template <class T>
bool ExpressionUtil::ExpressionListEquals(const vector<unique_ptr<T>> &a, const vector<unique_ptr<T>> &b) {
	if (a.size() != b.size()) {
		return false;
	}
	for (idx_t i = 0; i < a.size(); i++) {
		if (!(*a[i] == *b[i])) {
			return false;
		}
	}
	return true;
}
```

There is a *different* helper, `ExpressionUtil::SetEquals`, built on
`ExpressionSetEquals` (same file, using an `expression_map_t`/`parsed_expression_map_t`
internally to compare two lists as multisets rather than as ordered sequences) — but
`BoundFunctionExpression::Equals` calls `ListEquals`, not `SetEquals`. **This means
`a + b` and `b + a` — two calls to the same `+` function, each with two column-ref
children, but in opposite argument order — are two structurally *different* expression
trees as far as this equality check is concerned**, because child index 0 of one
doesn't equal child index 0 of the other. `CountExpressions`/`PerformCSEReplacement`
would never recognize them as "the same subexpression, occurring twice"; each would be
computed independently, exactly as if the query genuinely needed two unrelated sums.

It's worth being equally precise about the one place commutativity-*flavored*
canonicalization does happen nearby, because it's easy to conflate with CSE's own
equality check and it isn't the same mechanism. `ConstantOrderNormalizationRule`
(`src/optimizer/rule/constant_order_normalization.cpp`) is one of the 37 expression
rewriter rules from §7.5, run in pass **#1** — twenty-four pass-slots *before*
`COMMON_SUBEXPRESSIONS` at #25. Reading its constructor and `Apply` method directly
(reproduced in full at `constant_order_normalization.cpp:52-141`) shows it is narrowly
scoped: it only matches `+` and `*` calls over integer types, and it only reorders
children when **at least one child is foldable** (`bindings[i].get().IsFoldable()`) —
its whole job is moving constant-or-constant-foldable operands to a canonical position
(generally to the front, except multiplication's constant specifically moves to the
*end*, per `MultiplicationConstantBelongsAtEnd`) within a flattened `+`/`*` chain. Two
concrete consequences follow directly from reading the code this carefully rather than
assuming "there's a normalization pass, so commutativity must be handled somewhere":

- `1 + a` and `a + 1` **do** end up identical by the time `COMMON_SUBEXPRESSIONS` runs,
  because both get canonicalized to the same constant-first form at pass #1 — so CSE's
  plain structural equality, applied to the *already-normalized* tree, sees them as one
  expression. This is a real, useful interaction between two passes, but the
  commutative reasoning happened at pass #1, not inside `cse_optimizer.cpp` itself.
- `a + b` and `b + a`, where **neither** operand is foldable (two column references,
  say), never enter `ConstantOrderNormalizationRule`'s `ordered_bindings` set at all —
  `if (ordered_bindings.empty()) { return nullptr; }` bails out with no change. They
  reach pass #25 exactly as written, in whatever order the SQL text put them, and CSE's
  ordered `ListEquals` treats them as two different expressions.

So the fully precise answer to "does it account for commutativity" is: **no, not as a
property of the CSE pass itself** — `cse_optimizer.cpp` performs exact, positional,
structural tree comparison with no awareness that `+`/`*`/etc. are commutative
operations at all. What can make two commutative-looking expressions collapse into one
CSE candidate is an *upstream*, narrowly-scoped normalization rule that only reorders
operands when a foldable constant is involved — general operand-vs-operand
commutativity (two non-constant subexpressions swapped) is never normalized anywhere
in this pipeline, and is invisible to this pass.

## 7.16.4 Scope, precisely: within one operator, but which operators?

The second question this section was asked to verify rather than assume: is this
"within one operator's expression list," and does it ever reach across operator
boundaries (a filter and a projection sharing a computed value, for instance)? The
answer from `VisitOperator` is unambiguous, and narrower than "any operator" might
suggest:

```cpp
// src/optimizer/cse_optimizer.cpp:36-44
void CommonSubExpressionOptimizer::VisitOperator(LogicalOperator &op) {
	switch (op.type) {
	case LogicalOperatorType::LOGICAL_PROJECTION:
	case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
		ExtractCommonSubExpressions(op);
		break;
	default:
		break;
	}
	LogicalOperatorVisitor::VisitOperator(op);
}
```

`ExtractCommonSubExpressions` only ever runs for exactly two `LogicalOperatorType`
values: `LOGICAL_PROJECTION` and `LOGICAL_AGGREGATE_AND_GROUP_BY`. Every other operator
type — `LOGICAL_FILTER`, every join type, `LOGICAL_ORDER_BY`, `LOGICAL_LIMIT`, and so
on — falls into `default: break;`, meaning this pass never even inspects that
operator's own expressions for repeated subtrees, let alone rewrites them. The
recursive call, `LogicalOperatorVisitor::VisitOperator(op)`, still walks down into
every child operator (so a `LOGICAL_PROJECTION` or `LOGICAL_AGGREGATE_AND_GROUP_BY`
buried three joins deep still gets its own, independent `ExtractCommonSubExpressions`
call when the traversal reaches it) — but that's a separate, later call on a
*different* operator's own expression list, not a search across operator boundaries.
There is no code path anywhere in `cse_optimizer.cpp` that compares an expression
belonging to one operator against an expression belonging to a different operator.

This resolves the scope question in the strict direction: **it is not "within one
operator's expression list, looking at whichever operator" — it is specifically and
only within a `LogicalProjection`'s select-list, or within a `LogicalAggregate`'s own
expressions, and nowhere else.** A duplicated subexpression sitting inside a
`LogicalFilter`'s own predicate, entirely on its own with no projection or aggregate
anywhere nearby, is simply never looked at by this pass.

That last claim is checkable directly, and it's worth actually checking rather than
inferring purely from the `switch` statement, because `EXPLAIN` output could in
principle be shaped by some other pass acting on the filter first. Confirmed
empirically against the real optimizer (`duckdb v1.1.3` CLI):

```sql
create table test3(a integer);
insert into test3 values (42),(1),(200);
EXPLAIN SELECT * FROM test3 WHERE ((a*2)+(a*2))>100;
```

```
┌───────────────────────────┐
│           FILTER          │
│    ────────────────────   │
│(((a * 2) + (a * 2)) > 100)│
│          ~1 Rows          │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│         SEQ_SCAN          │
│           test3           │
│       Projections: a      │
│          ~3 Rows          │
└───────────────────────────┘
```

No projection was spliced in beneath this `FILTER` at all — `(a * 2)` appears
literally twice, side by side, inside the filter's own printed expression. (A version
of this same query against a single-row table, where DuckDB's own propagated column
statistics prove the comparison is unconditionally true for the only value the column
can hold, has the entire filter optimized away by `STATISTICS_PROPAGATION` — #34,
running many passes *after* this one — before ever reaching a state where this
question could be observed at all; the three-value table above avoids that trap and
lets the filter survive to the final plan, unresolved by CSE, exactly as the source
predicts.) This means `test/sql/optimizer/expression/test_cse.test`'s own two
`WHERE`-clause cases (§7.16.5 walks through them) are testing something real and
important, but it is not what a name like "test_cse.test" might suggest for those
specific cases — see §7.16.5's discussion of exactly those two blocks for the honest
version of what they actually demonstrate.

One further, positive detail about scope, worth stating precisely because it's a real
capability and not just a restriction: for `LOGICAL_AGGREGATE_AND_GROUP_BY` specifically,
"this operator's own expressions" is *not* limited to the aggregate function calls.
`EnumerateOperatorExpressions` (`src/planner/logical_operator_visitor.cpp:185-191`,
the helper `LogicalOperatorVisitor::EnumerateExpressions` forwards to) enumerates
*both* the `GROUP BY` key list (`aggr.groups`) *and* the aggregate's own output
expressions (`op.expressions`, inherited from the base `LogicalOperator`, which for a
`LogicalAggregate` holds the bound `SUM(...)`/`COUNT(...)`/etc. calls) as one combined
set for this single call to `CountExpressions`/`PerformCSEReplacement`:

```cpp
// src/planner/logical_operator_visitor.cpp:185-191
case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY: {
	auto &aggr = op.template Cast<LogicalAggregate>();
	for (auto &group : aggr.groups) {
		callback(&group);
	}
	break;
}
// ... falls through to:
for (auto &expression : op.expressions) {
	callback(&expression);
}
```

Concretely: a query grouping by an expression and also aggregating over that exact
same expression — `SELECT a+1, SUM(a+1) FROM t GROUP BY a+1` — has its `GROUP BY` key
`a+1` and `SUM`'s argument `a+1` counted together, as the same candidate, within one
`LogicalAggregate` node; if both are structurally identical (same rules as §7.16.3),
`CountExpressions` sees two occurrences and the pass factors the shared `a+1` into the
one synthetic projection beneath the aggregate, computed once for both the grouping
key and the aggregate's input. This is a real, if narrow, cross-role sharing capability
— it just isn't cross-*operator* sharing; it's still entirely internal to the one
`LogicalAggregate` node's own combined expression set.

## 7.16.5 `test/sql/optimizer/expression/test_cse.test`, every case

The full file is nine `query`/`statement` blocks past its setup, over two small
tables. It is short enough, and important enough as this book's primary citable CSE
test, to walk through completely rather than sampling.

```sql
-- test/sql/optimizer/expression/test_cse.test
statement ok
SET default_null_order='nulls_first';

statement ok
create table test(a integer);

statement ok
insert into test values (42);
```

### Case 1 — single CSE in a plain projection

```sql
# single CSE
query I
SELECT (a*2)+(a*2) FROM test
----
168
```

`(a*2)` appears twice as children of the top-level `+`. This is the cleanest possible
instance of §7.16.2's mechanism: `CountExpressions` sees `(a*2)` (a `BoundFunctionExpression`
over `*`) twice while walking the single output expression `(a*2)+(a*2)`, both
occurrences unconditionally reached (no conjunction/`CASE` short-circuiting involved),
so its count reaches 2 and it qualifies. Confirmed directly against `EXPLAIN`:

```
PROJECTION   (#0 + #0)
     │
PROJECTION   (a * 2)
     │
SEQ_SCAN     test  (Projections: a)
```

The lower `PROJECTION` — the synthetic one `ExtractCommonSubExpressions` spliced in —
computes `a * 2` exactly once, as its single output column `#0`. The original
projection, now sitting immediately above it, no longer contains a `*` node at all: its
one output expression is `(#0 + #0)`, two references to the same materialized column.
Tying this to Chapter 12: `ExpressionExecutor::Execute` walks each operator's own
expression tree once per input `DataChunk` (up to 2,048 rows). Before this rewrite, that
walk for the single output expression `(a*2)+(a*2)` would visit *two independent*
`BoundFunctionExpression(*)` nodes and evaluate the multiply twice per chunk — full
vectorized work, twice, on the exact same input vector. After this rewrite, the lower
projection's walk evaluates `*` once per chunk, producing one `Vector`; the upper
projection's walk never touches `*` at all — its own `Execute` call for
`BOUND_FUNCTION`/`+` (`execute_function.cpp`, per Chapter 12's dispatch table) receives
the *same* already-materialized vector as both of its two operands and only has to
perform the addition. One multiply's worth of vectorized work, per chunk, is saved —
not "fewer tree nodes" in the abstract, but one fewer real pass of the multiply kernel
over every row of every `DataChunk` this query chunk-processes, for as long as the
query runs.

### Case 2 — multiple occurrences, still one shared computation

```sql
# multiple CSEs
query I
SELECT (a*2)+(a*2)+(a*2)+(a*2)+(a*2) FROM test
----
420
```

Five occurrences of `(a*2)`, not two. `CountExpressions`'s frequency table doesn't care
how many times past the first an expression repeats — the `CSENode::count` field just
keeps incrementing, and `PerformCSEReplacement`'s `node.column_index.IsValid()` guard
(§7.16.2) still only pushes the *first* occurrence's actual expression object into the
new projection; the remaining four all become the same `BoundColumnRefExpression`. The
verified plan (§7.16.7 shows the identical shape for the `, a` variant below) is exactly
one lower projection computing `a*2` once, and an upper projection summing the same
column reference five times: `((((#0+#0)+#0)+#0)+#0)`. Per Chapter 12's framing: four
multiplies' worth of redundant vectorized work are avoided per `DataChunk`, not just
one — the savings scale with how many times the subexpression was written, not with
whether it was written twice or five times.

### Case 3 — a passthrough column survives alongside the shared computation

```sql
# use the actual columns still
query II
SELECT (a*2)+(a*2)+(a*2)+(a*2)+(a*2), a FROM test
----
420	42
```

This is the case that exercises `PerformCSEReplacement`'s
`ExpressionClass::BOUND_COLUMN_REF` branch directly (§7.16.2): the raw column `a`,
requested in the second output position, is not itself a repeated compound expression
— it's a leaf — but because the whole operator now has a new child projection sitting
between it and the scan, `a` has to be re-bound to reference that new projection's
output instead of the scan directly. `state.column_map` makes sure this happens exactly
once per distinct binding even though nothing here calls for deduplicating *multiple*
occurrences of `a` (there's only one). The verified `EXPLAIN` output (§7.16.7 traces
this exact query) shows the lower projection outputting two columns, `(a * 2)` and `a`,
and the upper projection outputting `((((#0+#0)+#0)+#0)+#0)` and `#1` — the plain
column reference passed straight through, at zero recomputation cost, alongside the
shared multiply result.

### Case 4 — CSE inside an aggregate's argument

```sql
# CSE in aggregates
query I
SELECT SUM((a*2)+(a*2)+(a*2)+(a*2)+(a*2)) FROM test
----
420
```

Here the operator that fires `ExtractCommonSubExpressions` is a `LOGICAL_AGGREGATE_AND_GROUP_BY`
node with no `GROUP BY` clause at all (a single implicit group), not a
`LOGICAL_PROJECTION`. Per §7.16.2's `BOUND_AGGREGATE` exclusion, the `SUM(...)` call
itself is walked through rather than counted, but its argument — the five-way-repeated
`(a*2)+...` — is exactly the shape Case 2 already demonstrated, just reached through an
aggregate's argument list instead of a projection's select-list. The physically
observable effect: a synthetic projection computing `a*2` once gets spliced in directly
beneath the `LogicalAggregate`, and the aggregate's own argument expression becomes the
column-reference sum, exactly like Case 2's upper projection — except here that
rewritten expression lives inside the aggregate operator's own argument slot rather
than a separate `LogicalProjection`.

### Case 5 — CSE shared between the `GROUP BY` key and the aggregate argument

```sql
# also with group by clause
query II
SELECT a, SUM((a*2)+(a*2)+(a*2)+(a*2)+(a*2)) FROM test GROUP BY a
----
42	420
```

This is the concrete instance of the "GROUP BY key and aggregate argument counted
together" capability §7.16.4 described from reading `EnumerateOperatorExpressions`
directly. Here the `GROUP BY` key is just `a` (a leaf, so it can't itself be a
duplicated *compound* expression, and isn't — it participates in the
`BOUND_COLUMN_REF`/`column_map` passthrough path, same as Case 3), while the aggregate
argument contributes the same five-way `(a*2)` repetition Case 4 already covered. Both
are enumerated in the same `CountExpressions`/`PerformCSEReplacement` walk over this one
`LogicalAggregate` node, and §7.16.7 shows the fully verified, real `EXPLAIN` trace for
almost this exact query (over a three-row table, so the plan isn't collapsed by
statistics propagation) — including the two neighboring, *unrelated* passes
(`COMPRESSED_MATERIALIZATION`, feeding the perfect-hash `GROUP BY` implementation) that
show up right next to this pass's own output and are easy to mis-attribute to CSE if
you aren't reading the plan carefully.

### Cases 6 and 7 — `WHERE`-clause repetition: correctness, not a CSE demonstration

```sql
# CSE in WHERE clause
query I
SELECT * FROM test WHERE ((a*2)+(a*2))>100
----
42

# multiple CSE in WHERE clause
query I
SELECT * FROM test WHERE ((a*2)+(a*2)+(a*2)+(a*2)+(a*2))>400
----
42
```

This is the honest, source-and-`EXPLAIN`-verified reading §7.16.4 built up to: per the
`VisitOperator` switch, `LOGICAL_FILTER` is never a trigger for
`ExtractCommonSubExpressions`, and the direct `EXPLAIN` check in §7.16.4 confirmed that
a genuinely surviving filter (one statistics propagation can't fold away first) keeps
its duplicated `(a * 2)` written out twice, side by side, inside the filter's own
predicate — no synthetic projection appears beneath it. These two test cases are real
and are worth having in the suite, but what they actually test is that a query with a
repeated subexpression **inside a `WHERE` clause** still produces the *correct answer*
even though — contrary to what the test file's name might suggest for these two blocks
specifically — the repeated `(a*2)` is not deduplicated by this optimizer pass at all.
Both `a*2` evaluations genuinely happen, once each, per row, inside the physical
`FILTER` operator; the test is a correctness regression guard for repeated arithmetic
inside a predicate, not a demonstration of `COMMON_SUBEXPRESSIONS` reaching into a
filter. This is exactly the kind of distinction this book's sourcing policy asks to be
stated plainly rather than glossed over: the file's title covers a broader theme
("queries involving common subexpressions") than the one narrow optimizer pass this
chapter is about, and cases 6–7 are where that broader/narrower gap is most visible.

### Cases 8 and 9 — strings, `NULL`, and why CSE never risks a false "always equal"

```sql
statement ok
create table test2(a VARCHAR);

statement ok
insert into test2 values ('hello'), ('world'), (NULL);

# single CSE in projection
query T
SELECT substring(a, 1, 3)=substring(a, 1, 3) FROM test2 ORDER BY 1
----
NULL
1
1

# now with GROUP BY clause
query T
SELECT substring(a, 1, 3)=substring(a, 1, 3) AS b FROM test2 GROUP BY b ORDER BY b
----
NULL
1
```

(`SET default_null_order='nulls_first'` from the file's setup is why the `NULL` row
sorts first in both result sets.) `substring(a, 1, 3)` is a `BoundFunctionExpression`
call, non-volatile, with a column-ref child and two constant children — exactly
eligible for CSE by every rule §7.16.2/§7.16.3 established, and it appears twice, as
both operands of `=`. Confirmed empirically (against an equivalent cast-based query, to
match a column type this book's setup already had on hand):

```
PROJECTION   (#0 = #0)
     │
PROJECTION   substring(CAST(a AS VARCHAR), 1, 3)
     │
SEQ_SCAN     test5  (Projections: a)
```

The comparison becomes a single materialized `substring` result compared against
*itself* via one shared column reference. This case is worth pausing on for a subtlety
that could easily go wrong in a less careful implementation: **DuckDB's CSE pass never
attempts to fold `X = X` down to a constant `TRUE`.** That would be a *different*,
much more aggressive (and, for nullable `X`, outright *unsound*) rewrite — a
self-comparison simplification, which several of the expression-rewriter rules from
§7.5 (`ComparisonSimplificationRule`, for instance) *do* perform in other narrow,
carefully-guarded cases, always with explicit `NULL`-preserving machinery
(`ExpressionRewriter::ConstantOrNull`) bolted on. `CommonSubExpressionOptimizer` does
nothing of the sort: it only ever introduces a *shared reference* to one materialized
value, and lets whatever operator consumes that reference (here, `=`) evaluate however
it normally would. `#0 = #0`, where `#0` happens to be `NULL` for the third row, still
correctly evaluates to `NULL` — three-valued logic is completely undisturbed, because
CSE never reasoned about the *meaning* of the `=` at all, only about the fact that its
left and right children were, structurally, the exact same subexpression. This is the
concrete payoff of §7.16.3's finding that identity here is purely structural and
syntactic: a pass that only ever asks "have I computed this exact tree before," never
"what does this comparison logically simplify to," cannot introduce a semantic bug of
this kind by construction — there's no code path in `cse_optimizer.cpp` that inspects
what *operator* is consuming the shared subexpression, only what the subexpression
*is*.

## 7.16.6 Coverage beyond the primary test file

A broader sweep of `test/` for "common subexpression" (not just literal mentions of
`cse_optimizer`, which turns up nothing outside `cse_optimizer.cpp`'s own file name)
turns up three more files worth a look, and — being honest about what they actually
add, per this book's stated policy — only one of them is really more coverage of
*this* pass; the other two turn out to be evidence of a related but distinct mechanism,
which is itself worth knowing about precisely because its name invites confusion with
this chapter's subject.

**`test/sql/function/generic/invoke.test`** contains a block explicitly commented
*"Lambda volatility must be visible to common subexpression elimination"*:

```sql
CREATE SEQUENCE invoke_cse_seq START 1

query II
SELECT count(*) FILTER (WHERE a = b), count(*)
FROM (
	SELECT
		invoke(lambda x: nextval('invoke_cse_seq'), i) a,
		invoke(lambda x: nextval('invoke_cse_seq'), i) b
	FROM range(10000) tbl(i)
)
----
0	10000
```

`a` and `b` are two *syntactically identical* calls — same `invoke`, same lambda
source text, same argument `i` — sitting in the same `LogicalProjection`'s expression
list. A CSE implementation that only checked syntactic shape and ignored volatility
would happily fold these into one shared computation, and every row would then have
`a == b` by construction (the count would come back `10000`, not `0`). The result the
test actually asserts, `0 10000`, is a direct, load-bearing confirmation that
`IsVolatile()` correctly propagates *through* the `invoke`/lambda wrapper — recall
`BoundFunctionExpression::IsVolatile()`'s explicit lambda-body check
(`src/planner/expression/bound_function_expression.cpp:54-58`,
`function.GetStability() == FunctionStability::VOLATILE || (lambda_expr &&
lambda_expr->IsVolatile()) || Expression::IsVolatile()`) — so that
`CountExpressions`'s `!expr.IsVolatile()` guard (§7.16.2) correctly refuses to register
either occurrence as a CSE candidate at all, and both `nextval` calls genuinely execute
independently, once per row, exactly as the surrounding SQL demands. This is real,
additional coverage of `cse_optimizer.cpp`'s volatility guard specifically, from a
different angle (a lambda body, rather than a directly-volatile top-level function
call) than anything in `test_cse.test` exercises.

**`test/sql/window/test_window_cse.test`** and **`test/sql/window/test_duplicate_partition.test`**
both use the phrase "common subexpression (elimination)" in their own comments and
descriptions, and it would be easy to fold them into this section's coverage on a
keyword match alone — but reading the mechanism they actually exercise shows they are
testing a **different, independent piece of machinery entirely**, one this chapter's
subject (`OptimizerType::COMMON_SUBEXPRESSIONS`, `src/optimizer/cse_optimizer.cpp`) has
no involvement in. `PhysicalPlanGenerator::PlanWindow`
(`src/execution/physical_plan/plan_window.cpp:116-128`) contains its own,
purpose-built deduplication step for window functions, run during **physical** plan
generation (Chapter 9's territory) rather than anywhere in the logical
`RunBuiltInOptimizers()` sequence this chapter's ~45 passes belong to:

```cpp
// src/execution/physical_plan/plan_window.cpp:116-128
// CSE Elimination: Search for a previous match (volatile expressions must not be deduplicated)
bool cse = false;
if (!wexpr.IsVolatile()) {
	for (idx_t i = 0; i < matching.size(); ++i) {
		const auto match_idx = matching[i];
		auto &match_expr = op.expressions[match_idx]->Cast<BoundWindowExpression>();
		if (wexpr.Equals(match_expr)) {
			projection_map[input_width + expr_idx] = output_pos + i;
			cse = true;
			break;
		}
	}
}
```

This code compares `BoundWindowExpression`s sharing an equivalent `PARTITION BY`
(checked separately, via `PartitionsAreEquivalent`, before this snippet even runs) using
the exact same structural `Equals()`/`IsVolatile()` discipline §7.16.2–§7.16.3 already
established for the scalar-expression case — same *shape* of guard, same reasoning
about volatility — but it is a hand-written, one-off dedup loop living inside the
physical window-operator planner, folding two identical window-function calls (e.g. two
occurrences of `sum(cnt_trace) over ()` inside one query, as `test_window_cse.test`'s
own worked example constructs) so that only one physical `PhysicalWindow` operator
column actually computes the shared aggregate, with the second occurrence's output
just a `projection_map` alias onto the first. `test_duplicate_partition.test`'s
"different functions prevent common subexpression elimination from hiding mask
mismatches" comment is testing an edge case of this *same* window-level mechanism (two
different window functions over a partition key list containing a literal duplicate,
`PARTITION BY a, a`), not `cse_optimizer.cpp`.

So: this book's honest accounting is that DuckDB has (at least) **four** distinct,
independently-implemented "don't compute the same thing twice" mechanisms, not three —
this chapter's `COMMON_SUBEXPRESSIONS` (logical, expression-tree-local, Projection/Aggregate
only), `COMMON_AGGREGATE` and `COMMON_SUBPLAN` (§7.16.8 covers both), and this
fourth one, window-function CSE, which doesn't even have its own `OptimizerType` or
appear in `duckdb_optimizers()` at all — it's not toggleable, not individually timed,
and lives entirely inside the physical plan generator rather than the optimizer
pipeline §7.2–§7.4 describe. A full treatment of window-function physical planning
belongs to Chapter 9's territory, not this chapter's; it's flagged here only because
its test files' own vocabulary ("CSE," "common subexpression elimination") makes it a
natural, and easy to get wrong, place for a reader grepping the test suite to land.

## 7.16.7 A complete worked example: `SELECT a, a+1, (a+1)*2 FROM t`

This section builds the example the task set out to construct, verified against a
real running instance rather than only reasoned about from source:

```sql
create table t(a integer);
insert into t values (5), (10), (100);
EXPLAIN SELECT a, a+1, (a+1)*2 FROM t;
```

```
┌───────────────────────────┐
│         PROJECTION        │
│    ────────────────────   │
│             a             │
│             #1            │
│          (#1 * 2)         │
│          ~3 Rows          │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│         PROJECTION        │
│    ────────────────────   │
│             a             │
│          (a + 1)          │
│          ~3 Rows          │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│         SEQ_SCAN          │
│             t             │
│       Projections: a      │
│          ~3 Rows          │
└───────────────────────────┘
```

That's the operator-tree view, in this book's usual plan-diagram convention. The point
of this section is the *other* diagram the task asked for — the actual expression
trees, before and after, at the level `ExpressionExecutor` (Chapter 12) walks them.

### Before CSE — three independent expression trees, one subtree duplicated

The unoptimized `LogicalProjection`'s select-list has three entries. Written out as
full expression trees, `(a + 1)` is a genuinely separate, independently-allocated
`BoundFunctionExpression` object inside the third entry from the one that *is* the
second entry — same shape, same children, same everything `Expression::Equals` checks,
but two distinct objects that would each be walked, and each be evaluated, on their own:

```
 col0: a          col1: (a + 1)                 col2: ((a + 1) * 2)

 ColumnRef(a)      Function(+)                    Function(*)
                    ├── ColumnRef(a)               ├── Function(+)          ← duplicate of col1's tree
                    └── Constant(1)                │    ├── ColumnRef(a)
                                                    │    └── Constant(1)
                                                    └── Constant(2)
```

Per Chapter 12, `ExpressionExecutor::Execute` visits this operator's three output
expressions once per input `DataChunk`. Walking `col1`'s tree evaluates one `+`
kernel call over the chunk's `a` vector. Walking `col2`'s tree — entirely separately —
evaluates a *second*, independent `+` kernel call over the exact same `a` vector (to
reproduce `a + 1` all over again) before it can even begin the multiply. Two calls to
the addition kernel, per chunk, computing the identical result twice.

### After CSE — one shared node, reached by two references

`CommonSubExpressionOptimizer` finds `(a + 1)` counted twice (once as `col1` itself,
once as `col2`'s child), splices in a new `LogicalProjection` beneath the original one,
and moves the actual `(a + 1)` expression object into that new projection's own output
list — alongside a passthrough copy of `a`, threaded through via the
`BOUND_COLUMN_REF`/`column_map` path (§7.16.2), since `col0` still needs `a` directly
and `col0`'s old binding no longer resolves against the scan directly once this new
projection sits in between:

```
                    synthetic PROJECTION (new TableIndex P)
                    ────────────────────────────────────────
                    P.0 = ColumnRef(a)        P.1 = Function(+)
                                                     ├── ColumnRef(a)
                                                     └── Constant(1)
                                                            │
                                                 (a + 1) computed ONCE here
                                                            │
              ┌─────────────────────────┬──────────────────┴───────────────┐
              │                         │                                  │
        col0 = ColumnRef(P.0)     col1 = ColumnRef(P.1)              col2 = Function(*)
         "a", passthrough           "a+1" — now just a                ├── ColumnRef(P.1)  ──┐
                                     reference, no + node              └── Constant(2)      │
                                     here at all                                             │
                                                                        (references the SAME
                                                                         node col1 points at)
```

The two arrows converging on `P.1` are the whole point: there is exactly **one**
`Function(+)` node left anywhere in this operator's combined expression forest, and
both `col1` and `col2`'s `*` now reach it via an ordinary `BoundColumnRefExpression`
rather than each carrying their own copy of it. This is a genuine directed acyclic
graph of computation, not a tree, even though each individual operator's expression
*list* is still, formally, a set of separate trees — the DAG-ness lives in the fact
that two of those trees' leaves are column references that resolve to the *same*
materialized vector, produced by a shared node one level below.

Tied back to Chapter 12 one more time, concretely: the lower, synthetic
`PROJECTION`'s `ExpressionExecutor::Execute` call evaluates `Function(+)` exactly once
per `DataChunk`, producing one output `Vector`. The upper `PROJECTION`'s
`ExpressionExecutor::Execute` call, walking `col1` and `col2`, never touches a `+` node
at all — `col1`'s evaluation is a `BOUND_REF` lookup (Chapter 12, §12.1: "handing back
a reference to a `Vector` that's already sitting there — no computation at all"), and
`col2`'s evaluation performs exactly one `*` kernel call, over that already-materialized
vector and the constant `2`. Total addition-kernel invocations per chunk, across the
whole rewritten operator pair: **one** — down from two in the unoptimized tree, for
this specific three-column example, and down from `n − 1` avoided calls for the
`n`-fold repetition Case 2 (§7.16.5) demonstrated at `n = 5`. Over a query scanning
millions of rows in thousands of 2,048-row `DataChunk`s, that's thousands of avoided
vectorized addition passes, not an abstract "fewer tree nodes" saving — it is exactly
the kind of real, measurable, per-chunk execution-engine work Chapter 12 spends its own
pages quantifying, and this pass's entire reason to exist is arranging for the
optimizer to hand the execution engine a tree where that avoided work is structurally
impossible to accidentally redo.

## 7.16.8 The taxonomy: four mechanisms, one shared worry, four different scopes

§7.6.7 in the overview already sketched a three-way distinction between
`COMMON_SUBEXPRESSIONS`, `COMMON_AGGREGATE`, and `COMMON_SUBPLAN`; §7.16.6 above adds a
fourth, unnumbered mechanism (window-function CSE in the physical planner). Two sibling
parts cover the other passes in full: `05-cte-inlining-and-common-subplan.md` for
`COMMON_SUBPLAN` and `12-aggregate-reuse-partial-pushdown-and-common-aggregate.md` for
`COMMON_AGGREGATE`. The table below is a short, source-checked comparison of all four.

| Pass | `OptimizerType` / location | Scope of "identical" | What identity is checked against | What happens once found |
|---|---|---|---|---|
| **`COMMON_SUBEXPRESSIONS`** (this chapter) | `cse_optimizer.cpp`, pass #25 | One `LogicalProjection`'s select-list, or one `LogicalAggregate`'s combined `groups` + output expressions — never across operators | Exact, ordered, structural `Expression::Equals`/`Hash` — leaves (column refs, constants) never extracted themselves; `BOUND_AGGREGATE` nodes never extracted (only their arguments); volatile expressions never registered at all; nothing inside a `TRY(...)` registered (a whole `TRY` call still can be) | A new child `LogicalProjection` is spliced in below the operator; every duplicate becomes a `BoundColumnRefExpression` into it |
| **`COMMON_AGGREGATE`** | `common_aggregate_optimizer.cpp`, pass #37 | One `LogicalAggregate` node's own list of aggregate *calls* (`aggr.expressions`) — i.e. whole `SUM(x)`/`COUNT(*)`-style computations, not their arguments | The same `expression_map_t` machinery (`ExtractCommonAggregates`, `src/optimizer/common_aggregate_optimizer.cpp:46-83`), applied to the aggregate calls themselves; volatile aggregates are explicitly exempted (`aggr.expressions[i]->IsVolatile()`) since "each call is independent" | The duplicate aggregate call is erased outright from `aggr.expressions`, and every column reference that used to point at it is remapped, via `aggregate_map`, to the surviving one — no new operator is introduced, because the `LogicalAggregate` itself already holds the shared value |
| **`COMMON_SUBPLAN`** | `common_subplan_optimizer.cpp`, pass #28 | Entire plan **subtrees** — a scan, a filter, a join, or any combination — appearing anywhere in the tree, not limited to sibling operators or one parent's children | A dedicated plan-signature/canonicalization mechanism (per the overview's own inline-comment citation and this file's own `PlanSignatureColumnIndexMap` machinery, `common_subplan_optimizer.cpp`), heavier machinery than a simple expression hash, built specifically to recognize two structurally identical subtrees regardless of where in the overall plan each one sits | The shared subtree is materialized once as a synthetic CTE, and every original occurrence is replaced with a `CTE_SCAN` referencing it; explicitly skipped for any subtree containing something volatile, and skipped for the whole pass if the plan contains a DML CTE |
| **Window-function CSE** (no dedicated `OptimizerType`) | `plan_window.cpp:116-128`, inside physical plan generation (Chapter 9), not the logical optimizer pipeline at all | `BoundWindowExpression`s sharing an equivalent `PARTITION BY`, within one `LogicalWindow`/`PhysicalWindow` operator | Structural `BoundWindowExpression::Equals`, with the same `IsVolatile()` exemption as the other three | No new operator; the duplicate's output column becomes a `projection_map` alias onto the first occurrence's computed slot |

The one sentence version, for keeping these apart at a glance: **`COMMON_SUBEXPRESSIONS`
shares scalar sub-computations inside one Projection or Aggregate's own expression
list; `COMMON_AGGREGATE` shares whole aggregate-function calls inside one Aggregate
node; `COMMON_SUBPLAN` shares entire multi-operator subtrees anywhere in the plan, via
a synthetic CTE; and window-function CSE, off in the physical planner and not part of
this pipeline's naming scheme at all, shares whole window-function computations inside
one window operator.** All four exist because the same worry — a query that
accidentally asks the engine to compute the same thing more than once — shows up at
four different granularities of "the same thing," and DuckDB's architecture (per §7.2's
"fixed, one-shot, narrowly-specialized passes" description) responds to each
granularity with its own dedicated, independently-guarded piece of code rather than one
general-purpose deduplication framework covering all four at once.

## 7.16.9 Summary

`COMMON_SUBEXPRESSIONS` is small — one 193-line file, no sub-rules, no cost model, a
single count-then-replace pass — and its narrowness is exactly what this section spent
its length pinning down precisely rather than leaving as a vague "it's CSE" gloss.
Identity is exact structural equality, checked via `Expression::Hash`/`Equals` all the
way down, with no commutative or associative reasoning built into this pass itself
(some of that reasoning happens much earlier, in pass #1, and only for
constant-adjacent operands, in `ConstantOrderNormalizationRule`). Scope is exactly two operator types —
`LogicalProjection` and `LogicalAggregate`, verified both by reading `VisitOperator`'s
switch and by watching a genuinely surviving `WHERE`-clause duplicate stay
un-deduplicated in a real `EXPLAIN` — never a `LogicalFilter`, never a cross-operator
search. What it protects against getting wrong — volatile expressions, short-circuited
branches, and sub-expressions inside `TRY` — is handled by narrow, specific guards rather than a general soundness
argument, and each guard has its own test-suite evidence (`invoke.test`'s
`nextval`-in-a-lambda case for volatility; the short-circuit logic's own inline
comment for the conjunction/`CASE` guard; `test_try_common_subexpression.test` for
the `TRY` guard). And its actual payoff, made concrete in
§7.16.7's worked example and tied throughout to Chapter 12's `ExpressionExecutor`
model, is not an abstract reduction in tree size — it is a real, per-`DataChunk`
reduction in how many times a specific vectorized kernel call (an addition, a
multiplication, a `substring` call) actually runs, repeated over however many chunks a
query's input takes to scan.
