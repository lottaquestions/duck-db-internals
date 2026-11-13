# Chapter 7, Deep Dive 1 — The Expression Rewriter I: Arithmetic and Constants

## 7.9 Scope, and how a `.test` file proves a rewrite

`00-overview.md` established the architecture this file assumes throughout:
pass #1 of the 45-call-site pipeline, `OptimizerType::EXPRESSION_REWRITER`
(`src/optimizer/optimizer.cpp:253`), walks the whole logical plan tree once
and, at every expression node it visits, tries a fixed, ordered list of 37
independent `Rule` subclasses (§7.5 of the overview reproduced the full list
verbatim from the `Optimizer` constructor). That section also showed a
handful of these rules at a summary level — one code snippet and one
sentence each, enough to establish that the rule/matcher split exists. This
file is the first of several that go back to that same 37-rule list and stay
on each rule far longer: full source, full test walkthroughs, and — where
the source reveals something the one-line gloss can't — corrections to
whatever the rule's name or a casual description might suggest.

This file covers seven of the 37 rules, in the order requested rather than
their order in the `Optimizer` constructor (that construction order matters,
and is discussed explicitly for every rule below, but grouping this file by
"arithmetic and constants" as a theme reads better than jumping around the
full list):

1. `ConstantFoldingRule`
2. `ConstantOrderNormalizationRule`
3. `ArithmeticSimplificationRule`
4. `MoveConstantsRule` and `MoveUnaryMinusRule`
5. `DistributivityRule`
6. `StructExtractStructPackFoldingRule`
7. `LeastGreatestSimplificationRule`

Later files in this directory work through the remaining 28 rules
(`CaseSimplificationRule` and the other `*SimplificationRule`s,
`LikeOptimizationRule` and its string-prefix siblings, the aggregate-shaped
rules, and so on).

### The evidence mechanism: same-label `EXPLAIN` pairs

Nearly every worked example below is lifted directly from a real
`test/optimizer/*.test` file, and most of those tests use an idiom worth
explaining once, carefully, because it is the primary way this file (and the
rest of this directory) proves a rewrite happened rather than merely
asserting it. Here is the first pair in `test/optimizer/constant_folding.test`
(`test/optimizer/constant_folding.test:12-18`):

```sql
query I nosort cfold1
EXPLAIN SELECT 1+1
----

query I nosort cfold1
EXPLAIN SELECT 2
----
```

Two things make this different from an ordinary sqllogictest block. First,
sqllogictest's `query` directive normally checks the query's output against
literal text written after the `----` line; here, both blocks have an empty
expected-result section. Second, both blocks share the same label,
`cfold1`. DuckDB's test runner treats two (or more) `query` blocks that share
a label as a group whose *actual outputs must be byte-identical to each
other* — not to any literal text in the file at all. So this pair does not
assert "`EXPLAIN SELECT 1+1` prints such-and-such plan"; it asserts
"whatever `EXPLAIN SELECT 1+1` prints, `EXPLAIN SELECT 2` prints exactly the
same thing." Since `EXPLAIN`'s output *is* a textual rendering of the
optimized logical (and physical) plan tree, two queries producing identical
`EXPLAIN` output have — as far as this test can tell — identical plans. If
`SELECT 1+1` reached execution as literally "compute `1+1`," its plan would
contain a `+` function-call expression wrapping two constant `1`s; `SELECT 2`
was never anything but a single constant `2`. The only way these two plans
can come out byte-identical is if some pass between parsing and this
`EXPLAIN` rewrote `1+1` into the literal constant `2`. That pass is
`ConstantFoldingRule`, and this is why the same-label idiom counts as rigorous
evidence rather than a suggestive similarity: the test doesn't just show two
outputs that *look* alike, it structurally forces the harness to compare
them and fail if they diverge in any way, including things a human skimming
two `EXPLAIN` printouts might overlook (an extra cast node, a different
column-binding index, a stray alias).

Every test file in `test/optimizer/` also opens with
`PRAGMA explain_output = OPTIMIZED_ONLY;` (`constant_folding.test:9`, and
the same line in every other file this chapter cites) — this trims `EXPLAIN`'s
default output down to just the optimized logical plan, dropping the
unoptimized tree and the physical plan that would otherwise also appear and
make an unrelated difference (say, in operator choice) look like a
rewriter-caused difference. With that pragma set, a same-label pair's
byte-identical requirement is specifically a claim about the *optimized
logical plan* — which is exactly the layer `EXPRESSION_REWRITER` operates on.

This file uses the same-label idiom as its primary evidence mechanism from
here on: wherever the source text below shows a same-label pair copied from
an actual `.test` file, treat it as a rigorous, harness-checked proof of the
stated rewrite, not an illustrative gloss. Wherever no such test exists for a
rule (this happens twice in this file — §7.16 and part of §7.17), that is
stated explicitly, and a hand-constructed example is clearly marked as
"illustrative" rather than dressed up as something it isn't.

## 7.10 Shared machinery: `Rule`, matchers, and the bottom-up rewrite loop

All seven rules below are subclasses of the same abstract base,
`Rule` (`src/include/duckdb/optimizer/rule.hpp:17`):

```cpp
// src/include/duckdb/optimizer/rule.hpp:17
class Rule {
public:
	explicit Rule(ExpressionRewriter &rewriter) : rewriter(rewriter) {
	}
	virtual ~Rule() {
	}

	//! The expression rewriter this rule belongs to
	ExpressionRewriter &rewriter;
	//! The expression matcher of the rule
	unique_ptr<ExpressionMatcher> root;

	ClientContext &GetContext() const;
	virtual unique_ptr<Expression> Apply(LogicalOperator &op, vector<reference<Expression>> &bindings,
	                                     bool &fixed_point, bool is_root) = 0;
};
```

Every rule, in its constructor, builds a tree of `ExpressionMatcher` objects
and assigns it to `root` — this is the pattern the rule is looking for.
`ExpressionMatcher` itself (`src/include/duckdb/optimizer/matcher/expression_matcher.hpp:21`)
is a small base class with an `expr_class` filter (which `ExpressionClass`
the node must belong to), an optional `expr_type` filter (which
`ExpressionType`), and an optional `type` filter (a `TypeMatcher` — matching
against the expression's *return type*, not its `ExpressionClass`). Several
matcher subclasses this file's rules build on are worth having the exact
definitions for, since the prose below cites them by name repeatedly:

```cpp
// src/include/duckdb/optimizer/matcher/expression_matcher.hpp:53
class ConstantExpressionMatcher : public ExpressionMatcher {
public:
	ConstantExpressionMatcher() : ExpressionMatcher(ExpressionClass::BOUND_CONSTANT) {
	}
};
```

```cpp
// src/include/duckdb/optimizer/matcher/expression_matcher.hpp:67
class ComparisonExpressionMatcher : public ExpressionMatcher {
public:
	ComparisonExpressionMatcher()
	    : ExpressionMatcher(ExpressionClass::BOUND_FUNCTION), policy(SetMatcher::Policy::INVALID) {
	}
	vector<unique_ptr<ExpressionMatcher>> matchers;
	SetMatcher::Policy policy;

	bool Match(Expression &expr, vector<reference<Expression>> &bindings) override;
};
```

```cpp
// src/include/duckdb/optimizer/matcher/expression_matcher.hpp:126
class FunctionExpressionMatcher : public ExpressionMatcher {
public:
	FunctionExpressionMatcher() : ExpressionMatcher(ExpressionClass::BOUND_FUNCTION) {
	}
	vector<unique_ptr<ExpressionMatcher>> matchers;
	SetMatcher::Policy policy;
	unique_ptr<FunctionMatcher> function;

	bool Match(Expression &expr, vector<reference<Expression>> &bindings) override;
};
```

```cpp
// src/include/duckdb/optimizer/matcher/expression_matcher.hpp:154-162
//! The FoldableConstant matcher matches any expression that is foldable into a constant by the ExpressionExecutor
//! (i.e. scalar but not aggregate/window/parameter)
class FoldableConstantMatcher : public ExpressionMatcher {
public:
	FoldableConstantMatcher() : ExpressionMatcher(ExpressionClass::INVALID) {
	}
	bool Match(Expression &expr, vector<reference<Expression>> &bindings) override;
};
```

`FunctionExpressionMatcher` and `ComparisonExpressionMatcher` both carry a
`matchers` vector (one matcher per expected child) and a `policy` from
`SetMatcher::Policy` (`src/include/duckdb/optimizer/matcher/set_matcher.hpp:19`):

```cpp
// src/include/duckdb/optimizer/matcher/set_matcher.hpp:19
enum class Policy {
	//! All entries have to be matched, and the matches have to be ordered
	ORDERED,
	//! All entries have to be matched, but the order of the matches does not matter
	UNORDERED,
	//! Only some entries have to be matched, the order of the matches does not matter
	SOME,
	//! Only some entries have to be matched. The order of the matches does matter.
	SOME_ORDERED,
	INVALID
};
```

This `Policy` enum turns out to matter a great deal for one of the
corrections this file makes (§7.12): `UNORDERED` lets a two-child matcher
match a node's children *regardless of which literal side of the expression*
each child sits on, which is how several rules below handle "the constant
might be on the left or the right" without any separate normalization step.

Finally, every rule's `Apply` gets invoked through the same traversal, and
the exact shape of that traversal explains a property both this file and
the overview keep returning to — that the "one-shot, no-fixpoint" discipline
described for the ~45 *plan-level* passes in §7.2 is explicitly not how the
*inner* expression-rewrite loop behaves:

```cpp
// src/optimizer/expression_rewriter.cpp:56
unique_ptr<Expression> ExpressionRewriter::ApplyRules(LogicalOperator &op, const vector<reference<Rule>> &rules,
                                                      unique_ptr<Expression> expr, bool &changes_made, bool is_root) {
	vector<RewriteFrame> stack;
	vector<reference<unique_ptr<Expression>>> children;
	stack.push_back({expr, is_root, false});

	while (!stack.empty()) {
		auto &frame = stack.back();
		auto &current_expr = frame.expr.get();
		D_ASSERT(current_expr);
		if (!frame.children_visited) {
			frame.children_visited = true;
			// Rewrite the subtree bottom-up so parent rules see already-simplified children.
			CollectChildren(*current_expr, children);
			for (idx_t i = children.size(); i > 0; i--) {
				stack.push_back({children[i - 1], false, false});
			}
			continue;
		}
		bool node_made_change = false;
		current_expr = ApplyRule(op, rules, std::move(current_expr), node_made_change, frame.is_root);
		if (node_made_change) {
			changes_made = true;
			frame.children_visited = false;
			continue;
		}
		stack.pop_back();
	}
	return expr;
}
```

This is an explicit stack machine, not recursion, but the logic is
straightforward: for a given expression node, first push all of its children
(so they get fully processed — themselves bottom-up — before this node is
touched at all), then, once children are done, run `ApplyRule` (singular)
against the current node. `ApplyRule` (`expression_rewriter.cpp:19`) tries
every rule in this file's fixed list, in order, returns as soon as one rule
both matches *and* makes a change, and otherwise falls through to the next
rule. The critical line is `frame.children_visited = false;` right after a
change is detected: this doesn't pop the frame and move on — it resets the
frame so the *next* iteration re-collects and re-pushes this node's
(possibly brand-new) children, then re-applies the full rule list to this
same node again once they're done. A node keeps getting reprocessed — full
rule list, from the top, every time — until a pass through all 37 rules
produces no further change, at which point (and only then) the frame is
popped and its parent gets its turn. This is why §7.5 of the overview could
say the inner loop "does re-apply" even though the outer 45-pass pipeline
never does: expression rewriting for one plan-level pass is itself a small,
local fixpoint computation, one AST node at a time, re-entered as many times
as a single node keeps changing shape — and this is precisely what makes
`ConstantOrderNormalizationRule` and `ConstantFoldingRule`'s division of
labor work at all (§7.12 traces exactly this: rule 1 rewrites a node's shape,
the reset-and-redo logic immediately gives rule 2 a chance to fold the new
shape, all within one visit to what started as a single tree node).

With that shared machinery established, the seven rules follow, each with
its own construction-order position from `Optimizer::Optimizer`
(`src/optimizer/optimizer.cpp:73-109`, reproduced in full in the overview's
§7.5) called out explicitly, since — as promised — order among these rules
is not incidental.

## 7.11 `ConstantFoldingRule` — evaluate anything foldable, once

**Purpose, in one sentence:** any expression whose entire subtree contains
no column reference, aggregate, window function, or parameter — i.e. any
expression the `ExpressionExecutor` could evaluate right now, without a
single row of table data — gets replaced by its literal result.

**Construction-order position:** #2 in the `Optimizer` constructor
(`optimizer.cpp:75`), immediately after `ConstantOrderNormalizationRule`
(#1, `optimizer.cpp:74`) — a placement §7.12 explains is not a coincidence.

### Source

The matcher is a two-line specialization of `FoldableConstantMatcher`
(`src/optimizer/rule/constant_folding.cpp:12`):

```cpp
// src/optimizer/rule/constant_folding.cpp:11-21
//! The ConstantFoldingExpressionMatcher matches on any scalar expression (i.e. Expression::IsFoldable is true)
class ConstantFoldingExpressionMatcher : public FoldableConstantMatcher {
public:
	bool Match(Expression &expr, vector<reference<Expression>> &bindings) override {
		// we also do not match on ConstantExpressions, because we cannot fold those any further
		if (expr.GetExpressionType() == ExpressionType::VALUE_CONSTANT) {
			return false;
		}
		return FoldableConstantMatcher::Match(expr, bindings);
	}
};
```

The one line of logic here (excluding `VALUE_CONSTANT`) is a small but
important guard: without it, a constant that has *already* been folded would
itself match "any foldable expression," `Apply` would re-evaluate it (a
no-op, since it's already a literal), return a fresh `BoundConstantExpression`
holding the same value, and the bottom-up loop from §7.10 would see
`node_made_change = true` and re-enter the same node forever — an infinite
loop disguised as idempotence. Excluding already-constant nodes from the
matcher is what makes this rule's fixpoint at any given node actually
terminate.

`Apply` itself is the simplest of the seven rules in this file:

```cpp
// src/optimizer/rule/constant_folding.cpp:28-42
unique_ptr<Expression> ConstantFoldingRule::Apply(LogicalOperator &op, vector<reference<Expression>> &bindings,
                                                  bool &changes_made, bool is_root) {
	auto &root = bindings[0].get();
	// the root is a scalar expression that we have to fold
	D_ASSERT(root.IsFoldable() && root.GetExpressionType() != ExpressionType::VALUE_CONSTANT);

	// use an ExpressionExecutor to execute the expression
	Value result_value;
	if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), root, result_value)) {
		return nullptr;
	}
	D_ASSERT(result_value.type().InternalType() == root.GetReturnType().InternalType());
	// now get the value from the result vector and insert it back into the plan as a constant expression
	return make_uniq<BoundConstantExpression>(result_value);
}
```

`ExpressionExecutor::TryEvaluateScalar` runs the *entire* matched
subexpression through DuckDB's real, vectorized expression executor — the
same machinery that would evaluate `+`, `CASE`, or `IN` at actual query
runtime — just on a one-row, no-input "chunk," and captures whatever comes
out. This is a mechanically important point: `ConstantFoldingRule` does not
special-case which *function or operator* it's folding at all. It doesn't
know `+` means addition or that `IN` means list membership. Its entire
contribution is recognizing *which subtrees have no data dependency* (via
`Expression::IsFoldable`, `src/planner/expression.cpp:100` — a plain
recursive AND over `child.IsFoldable()`, with leaf classes like
`BoundColumnRefExpression`, `BoundParameterExpression`, and aggregate/window
expressions overriding it to `false`) and handing the whole subtree to the
one evaluator that already knows how to run every scalar function DuckDB
has. This is also why `TryEvaluateScalar` can fail (returning `false`,
in which case `Apply` returns `nullptr` and nothing changes) — a genuinely
foldable expression can still fail to *fold*, most commonly when evaluating
it would throw (an overflow, a cast error): the rule declines rather than
propagating an exception at optimization time for an expression that might,
in a differently-shaped query, never actually be reached at execution time.

### `test/optimizer/constant_folding.test`, in full

The file has exactly six labeled pairs (`test/optimizer/constant_folding.test:12-58`).
Each is worth walking individually, because they get progressively less
obvious.

**`cfold1`** — `EXPLAIN SELECT 1+1` ≡ `EXPLAIN SELECT 2`. Already discussed
in §7.9 as the idiom's canonical example. Mechanically: `1+1` is a
`BoundFunctionExpression` (`+`) over two `BoundConstantExpression` children.
`IsFoldable` is true (no column refs anywhere), it isn't itself a
`VALUE_CONSTANT`, so `ConstantFoldingExpressionMatcher` matches at this node
during the bottom-up walk, and `TryEvaluateScalar` runs the addition,
producing the integer `2`.

**`cfold2`** — `EXPLAIN SELECT (1+1+1)*2` ≡ `EXPLAIN SELECT 6`. Here the
bottom-up traversal actually matters for a reason worth spelling out:
`(1+1+1)*2` parses as `((1+1)+1)*2`. The innermost `1+1` is visited first
(children-before-parent), folds to `2`; that change resets the parent frame
(§7.10's `frame.children_visited = false` step) so the walk re-collects
children of `(1+1)+1` — now `2+1` — and folds *that* to `3`; the outer `*2`
node is visited last and folds `3*2` to `6`. Nothing here required three
separate optimizer passes or a worklist scheduler: it's the same rule,
matching three times against three different (successively smaller) nodes,
inside one call to `ApplyRules` for the whole expression tree, purely
because of the reset-and-redo behavior of the traversal itself.

**`cfold3`** — `EXPLAIN SELECT CASE WHEN 1 IN (1, 2, 3, 4) THEN 3 ELSE 5 END`
≡ `EXPLAIN SELECT 3`. This is the first case worth pausing on, because it is
tempting to assume `CaseSimplificationRule` (a separate rule, further down
the 37-rule list, covered in a later file in this directory) is doing the
work here. It isn't — or rather, it never gets the chance to. `IsFoldable`
is a whole-subtree property: the entire `CASE` expression has no column
reference anywhere in it (the condition is a constant `IN` check, the `THEN`
and `ELSE` branches are bare literals), so the *entire `CASE` node* is
`IsFoldable() == true` from the moment the binder produces it. When the
bottom-up walk finally reaches this node (after its already-constant
children have been individually — trivially — visited and left alone, since
they're already `VALUE_CONSTANT`), `ConstantFoldingExpressionMatcher`
matches the whole `CASE` expression directly, and `TryEvaluateScalar`
evaluates the `IN` check, the branch selection, and the result in one call
to the real expression executor — collapsing three levels of the tree
(`CASE`, `IN`, and the boolean check inside it) to the single literal `3` in
one `Apply` call, without `CaseSimplificationRule` or
`InClauseSimplificationRule` (both later in the rule list) ever matching
anything here at all. The distinction matters: those other two rules exist
specifically for the case where a `CASE` or `IN` expression is *not* fully
foldable (a real column reference is mixed in) — this test file's first
three pairs never reach that situation, so it is testing
`ConstantFoldingRule`, and only `ConstantFoldingRule`, however "CASE-shaped"
or "IN-shaped" the SQL text looks.

**`cfold4`** — `EXPLAIN SELECT CASE WHEN 1 IN (1+1, 2, 3, 4) THEN 3 ELSE 5 END`
≡ `EXPLAIN SELECT 5`. Same mechanism as `cfold3`, with one more layer: `1+1`
inside the `IN` list is itself visited (and folded to `2`) before the `IN`
expression's own turn comes, so by the time `ConstantFoldingRule` considers
the whole `CASE`, the tree it's evaluating is effectively
`CASE WHEN 1 IN (2, 2, 3, 4) THEN 3 ELSE 5 END` — and `1` is not a member of
`{2, 2, 3, 4}`, so the condition is `FALSE`, and the result is the `ELSE`
branch, `5`. (Whether the inner `1+1` gets folded to `2` as a genuinely
separate visit to a separate tree node, or gets folded as part of the same
`TryEvaluateScalar` call that folds the whole `CASE` — either is consistent
with `IsFoldable` and this rule's mechanics, and the test cannot distinguish
between them since it only checks the final `EXPLAIN` output; either way,
the *result* is unambiguous.)

**`cfold5`** — `EXPLAIN SELECT 1 IN (1+1, 2, 3, 4, NULL)` ≡
`EXPLAIN SELECT NULL::BOOLEAN`. This is the pair the task explicitly flags
as needing a careful three-valued-logic explanation, and it's worth doing
that explanation from first principles rather than just citing the result.
SQL's `IN` is defined as syntactic sugar for a disjunction of equality
checks: `x IN (v1, v2, ..., vn)` means `x = v1 OR x = v2 OR ... OR x = vn`.
After the inner `1+1` folds to `2`, the check becomes
`1 IN (2, 2, 3, 4, NULL)`, i.e.
`1=2 OR 1=2 OR 1=3 OR 1=4 OR 1=NULL`. Every one of the first four
comparisons evaluates to a definite `FALSE` — `1` genuinely, provably, does
not equal `2`, `3`, or `4`. The fifth comparison, `1 = NULL`, is not `FALSE`
and not `TRUE` under SQL's three-valued logic — it's `UNKNOWN` (rendered as
SQL `NULL`), because comparing anything to an unknown value produces an
unknown result, not a known-false one. Now the disjunction rule for
three-valued `OR`: `FALSE OR FALSE OR FALSE OR FALSE OR UNKNOWN` is
`UNKNOWN`, not `FALSE` — and the reason is exactly what a database has to
respect to stay correct: if that fifth list element (whatever value is
actually hiding behind the `NULL`) *had* turned out to equal `1`, the whole
expression would have been `TRUE`; since it's unknown whether it does, and
every other disjunct is definitely `FALSE`, the honest answer is "we don't
know" — `UNKNOWN`/`NULL` — not a confident `FALSE`. This is precisely the
same rule that makes `OR` with an `UNKNOWN` operand `UNKNOWN` unless some
*other* operand is already definitely `TRUE` (which would make the whole
`OR` `TRUE` regardless of what the unknown one turns out to be — exactly
what happens two paragraphs down in `cfold6`). `ExpressionExecutor::TryEvaluateScalar`
doesn't need a special rule to know any of this — it runs DuckDB's actual,
production `IN`/comparison/boolean-logic evaluation code (the same code a
real query with an actual `NULL` value in a real column would hit), so the
constant-folding rule inherits correct three-valued-logic behavior for free,
simply by delegating to the real executor instead of re-implementing SQL
semantics itself.

**`cfold6`** — `EXPLAIN SELECT CASE WHEN 1 IN (1+1, 2, 3, 4, NULL, 1) THEN (3+4) ELSE 2+2+2 END`
≡ `EXPLAIN SELECT 7`. Same list as `cfold5`, but with one more element: a
literal `1` appended at the end. Now the disjunction is
`FALSE OR FALSE OR FALSE OR FALSE OR UNKNOWN OR TRUE` — and because one
disjunct (`1=1`) is definitely `TRUE`, the *whole* `OR` chain is `TRUE`
regardless of the `UNKNOWN` sitting earlier in the list (an `OR` needs only
one `TRUE` operand to be `TRUE`, and no number of `UNKNOWN`s can override an
already-settled `TRUE`). So the `CASE` condition folds to a definite `TRUE`,
the `THEN` branch (`3+4`, which folds to `7`) is selected, and the `ELSE`
branch (`2+2+2`, which would have folded to `6`, and is never evaluated in
the final result) is discarded. The whole `CASE`/`IN`/nested-arithmetic tree
collapses, in one bottom-up pass, to the single literal `7`.

### Order and interactions

`ConstantFoldingRule` sits at construction-list position #2, immediately
after `ConstantOrderNormalizationRule` at #1. §7.12 traces a concrete
example (`col0 + 1 + 2`) where this exact adjacency is load-bearing: rule #1
regroups scattered constants in a `+`/`*` chain into one contiguous
sub-node, and rule #2 — thanks to §7.10's reset-and-redo traversal — gets to
fold that freshly-created sub-node on the very next visit to that same tree
position, within the same call to `ApplyRules`. Nothing about
`ConstantFoldingRule`'s own matcher or `Apply` depends on running second;
what depends on the ordering is whether a *multi-term* arithmetic chain ever
produces a fully-constant sub-node for `ConstantFoldingRule` to find in the
first place. For every test in this section's own file — none of which
involve a scattered multi-constant chain — `ConstantOrderNormalizationRule`
never actually fires (there's no `+`/`*` chain with more than zero
non-constant operands anywhere in `constant_folding.test`), so all six pairs
above are, in a real sense, evidence for `ConstantFoldingRule` alone,
uncomplicated by rule #1's interaction.

## 7.12 `ConstantOrderNormalizationRule` — regrouping, not just reordering

**Purpose, in one sentence, corrected against the source rather than the
rule's name:** for a chain of `+` or `*` operations over integer types,
gather every constant operand in the chain into one contiguous, rebuilt
sub-expression (preserving their relative order) with the non-constant
operands trailing after it — which matters because it turns a set of
constants *scattered non-adjacently* through a left-deep arithmetic tree
into a single node `ConstantFoldingRule` (position #2, run immediately
after) can actually match and fold.

That sentence is deliberately more specific than a natural first guess at
what a rule named "constant order normalization" does — and it's worth
being explicit that a natural first guess (something like "canonicalize
`5 = x` into `x = 5`, so comparisons always have their constant on a fixed
side") is *not* what this class does, once you actually read
`src/optimizer/rule/constant_order_normalization.cpp`. The matcher this rule
builds never touches a comparison at all:

```cpp
// src/optimizer/rule/constant_order_normalization.cpp:52-81
ConstantOrderNormalizationRule::ConstantOrderNormalizationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	// '+' and '*' satisfy commutative law and associative law.
	auto add_matcher = make_uniq<FunctionExpressionMatcher>();
	add_matcher->function = make_uniq<SpecificFunctionMatcher>("+");
	add_matcher->type = make_uniq<IntegerTypeMatcher>();
	auto left_expression_matcher = make_uniq<ExpressionMatcher>();
	auto right_expression_matcher = make_uniq<ExpressionMatcher>();
	left_expression_matcher->type = make_uniq<IntegerTypeMatcher>();
	right_expression_matcher->type = make_uniq<IntegerTypeMatcher>();
	add_matcher->matchers.push_back(std::move(left_expression_matcher));
	add_matcher->matchers.push_back(std::move(right_expression_matcher));
	add_matcher->policy = SetMatcher::Policy::ORDERED;

	auto multiply_matcher = make_uniq<FunctionExpressionMatcher>();
	multiply_matcher->function = make_uniq<SpecificFunctionMatcher>("*");
	// ... (identical shape, for "*")

	vector<unique_ptr<FunctionExpressionMatcher>> func_matchers;
	func_matchers.push_back(std::move(add_matcher));
	func_matchers.push_back(std::move(multiply_matcher));
	auto op = make_uniq<RecursiveFunctionExpressionMatcher>(std::move(func_matchers));
	root = std::move(op);
}
```

`RecursiveFunctionExpressionMatcher` (defined just above in the same file,
`constant_order_normalization.cpp:13-50`) is the interesting part: it isn't
a plain `ExpressionMatcher` combinator like the others in this file — it's a
small custom class whose `Match` first checks whether the expression is a
`+` or `*` call at all, and if so, *recursively flattens* through any nested
`+`/`*` calls of the same operator, walking all the way down and collecting
every non-`+`/`*` leaf operand into one flat `bindings` list, then pushing
the original (outer) expression itself as the final binding:

```cpp
// src/optimizer/rule/constant_order_normalization.cpp:36-47
void RecursiveMatch(FunctionExpressionMatcher *func_matcher, Expression &expr,
                    vector<reference<Expression>> &bindings) {
	vector<reference<Expression>> curr_bindings;
	if (func_matcher->Match(expr, curr_bindings)) {
		auto &func_expr = expr.Cast<BoundFunctionExpression>();
		for (auto &child : func_expr.GetChildren()) {
			RecursiveMatch(func_matcher, *(child.get()), bindings);
		}
	} else {
		bindings.push_back(expr);
	}
}
```

So for `col0 + 1 + 2` — which the binder builds as the left-deep tree
`(col0 + 1) + 2` — matching at the *outer* node flattens straight through
both `+` levels and produces `bindings = [col0, 1, 2, <outer node>]`, not
just the outer node's two immediate children. This single detail is the key
to everything the `Apply` method does next, and it's the reason "regrouping"
is the right word rather than "swapping": the rule doesn't see two operands
and decide which one goes where — it sees the *entire flattened chain* and
decides where every operand in it belongs.

`Apply` then partitions the flattened operand list into foldable
(`IsFoldable()`, i.e. constant or constant-shaped) and non-foldable operands,
and — the load-bearing move — rebuilds the expression with **all the
foldable operands first, in their original relative order, followed by all
the non-foldable operands**, but only if this reordering would actually
change something:

```cpp
// src/optimizer/rule/constant_order_normalization.cpp:89-120
// Put all constant expressions in front.
vector<reference<Expression>> ordered_bindings;
vector<reference<Expression>> remain_bindings;
idx_t last_constant_position = 0;
for (idx_t i = 0; i < bindings.size() - 1; ++i) {
	if (bindings[i].get().IsFoldable()) {
		ordered_bindings.push_back(bindings[i]);
		last_constant_position = i;
	} else {
		remain_bindings.push_back(bindings[i]);
	}
}

if (ordered_bindings.empty()) {
	return nullptr;
}

if (ordered_bindings.size() == 1) {
	// Multiplication trees can lose their folded constant subtree in the bottom-up rewriter
	// (e.g. 1 * 2 * x -> 2 * x). Normalize that remaining binary form to x * 2 so it
	// matches the shape produced by the other multiplication variants.
	if (!MultiplicationConstantBelongsAtEnd(root) || last_constant_position == expression_count - 1) {
		return nullptr;
	}
	remain_bindings.push_back(ordered_bindings[0]);
	ordered_bindings = std::move(remain_bindings);
} else {
	if (last_constant_position == ordered_bindings.size() - 1) {
		return nullptr;
	}
	ordered_bindings.insert(ordered_bindings.end(), remain_bindings.begin(), remain_bindings.end());
}
```

Two distinct code paths, worth telling apart:

- **Two or more constants in the chain** (the `else` branch): if the
  constants aren't *already* the first `N` entries of the flattened list
  (`last_constant_position == ordered_bindings.size() - 1` is the "already
  in canonical position, nothing to do" check), the rule rebuilds the whole
  chain with constants-then-non-constants and returns it — a genuine
  restructuring.
- **Exactly one constant in the chain** (the `ordered_bindings.size() == 1`
  branch): the rule almost always declines (`return nullptr`) here, *except*
  for one narrow multiplication-specific case the comment names directly —
  a residual two-operand `*` node left over after an earlier constant in the
  same chain has already been folded away by a previous visit, which this
  branch renormalizes to keep multiplication's single-remaining-constant
  shape consistent with what the two-or-more-constants branch produces.

After reordering, the rebuild itself is a straightforward left-fold: start
from the first entry in the reordered list, and repeatedly bind a fresh `+`
or `*` call combining the running result with the next entry
(`constant_order_normalization.cpp:122-140`), which is what actually
produces new tree nodes — including, critically, a brand-new sub-node that
contains *only* the gathered constants, glued together first because they
were placed first in the reordered list.

### Tracing `add_test` step by step

`test/optimizer/constant_order_normalization.test:12-26` is the `add_test`
group:

```sql
query I nosort add_test
EXPLAIN SELECT col0 + 1 + 2 FROM test
----

query I nosort add_test
EXPLAIN SELECT 1 + col0 + 2 FROM test
----

query I nosort add_test
EXPLAIN SELECT 1 + 2 + col0  FROM test
----

query I nosort add_test
EXPLAIN SELECT 3 + col0  FROM test
----
```

Four different-looking `SELECT` expressions, one shared label — the test
claims all four produce identical optimized plans, and that plan is the same
one `SELECT 3 + col0` produces on its own. Working through the first
variant, `col0 + 1 + 2` (parsed as `(col0 + 1) + 2`), against the mechanics
above: the outer node's flattened bindings are `[col0, 1, 2, <root>]`.
`ordered_bindings = [1, 2]` (both foldable), `remain_bindings = [col0]`,
`last_constant_position = 2` (the index of the `2`). Since
`ordered_bindings.size() == 2`, this takes the "two or more constants"
branch; `last_constant_position (2) != ordered_bindings.size() - 1 (1)`, so
the constants are *not* already in canonical position (`col0` sits between
them positionally in the original chain, at index 0, ahead of position 2 —
they aren't contiguous at the front), and the rule rebuilds:
`ordered_bindings` becomes `[1, 2, col0]`, and the left-fold rebuild produces
`(1 + 2) + col0` as a fresh replacement for the whole node. Because this
counts as a change, §7.10's traversal resets and redescends: the newly
created `(1 + 2)` sub-node is now visited on its own — `ConstantOrderNormalizationRule`
finds it already in canonical order (both constants, contiguous, nothing to
regroup) and declines, but `ConstantFoldingRule` (the very next rule in the
list) matches it directly (`IsFoldable`, not itself `VALUE_CONSTANT`) and
folds it to the literal `3`. The tree that remains is `3 + col0` — exactly
the fourth `SELECT` in the group, confirmed identical by the shared label.

The second and third variants (`1 + col0 + 2`, i.e. `(1+col0)+2`, and
`1 + 2 + col0`, i.e. `(1+2)+col0`) go through the identical mechanism with a
different starting flattened order — `[1, col0, 2]` and `[1, 2, col0]`
respectively — but since the rebuild always places every foldable operand
*first, in original relative order*, both collapse to the same
`(1 + 2) + col0` intermediate shape before `ConstantFoldingRule` reduces the
inner pair to `3`. This is the concrete payoff of "regrouping, not
reordering, in original relative order": no matter which of the three
positions the user happened to write `col0` in relative to the two literal
constants, the rule's output is the same canonical `constants-then-column`
shape, which is exactly why all four `EXPLAIN`s in this group are required
to be byte-identical.

`mult_test` (`constant_order_normalization.test:29-43`) runs the identical
mechanism over `*` instead of `+`, converging on `2 * col0` for all of
`col0*1*2`, `1*col0*2`, and `1*2*col0` — the same regroup-then-fold pipeline,
just riding the `multiply_matcher` half of the constructor instead of
`add_matcher`. `add_mult` (`constant_order_normalization.test:46-60`)
exercises both at once, in a single expression
(`col0 * 1 * 2 + col1 + 1 + 2`): the `*` sub-chain (`col0*1*2`) is a
*strict descendant* of the outer `+` chain, so the bottom-up walk fully
normalizes and folds it down to `2 * col0` before the outer `+`-flattening
matcher ever runs against the top-level node (from the `+`-matcher's
perspective, `2 * col0` is just one opaque, non-foldable leaf operand,
because it isn't itself a `+`/`*`-of-the-matching-operator node it can
recurse through further — a `*` node fails a `+`-shaped `FunctionExpressionMatcher::Match`
outright). The outer `+` chain's flattened operands, by the time it's
processed, are `[(2*col0), col1, 1, 2]`; gathering the two constants to the
front and folding them the same way as `add_test` produces the final,
fully-normalized shape `3 + 2 * col0 + col1` — exactly the fourth query in
that group.

### Why this must run first — and what "first" actually buys

`ConstantOrderNormalizationRule` occupies construction-list position #1,
strictly before `ConstantFoldingRule` at #2 (`optimizer.cpp:74-75`). The
architectural point requested for this section is a real one, and the trace
above makes it concrete rather than hand-wavy: `ConstantFoldingRule` can
only ever fold a subtree that **already exists as a single tree node** —
its matcher operates on whatever shape the tree happens to have at the
moment it's visited, and it has no way to look *across* an intervening
non-constant operand to find two constants that aren't siblings. In
`(col0 + 1) + 2`, the two literal constants `1` and `2` are not adjacent in
the original parse tree at all — `1` is nested two levels down, sharing a
node with `col0`, and `2` sits one level up, sharing a node with the whole
`col0 + 1` subexpression. There is no tree node in the *original* shape
whose two children are exactly `1` and `2`. `ConstantFoldingRule`, running
alone, would visit `col0 + 1` (not foldable — `col0` blocks it) and
`(col0+1) + 2` (also not foldable, for the same reason) and find nothing to
do anywhere in this expression. `ConstantOrderNormalizationRule`'s
contribution is manufacturing the missing tree node — a fresh `1 + 2`
sub-expression that did not exist in the parse tree — specifically so that
the very next rule in the list has a matching target. Running the two rules
in the opposite order would not merely be less efficient; it would leave
`col0 + 1 + 2` entirely unfolded, since a single subsequent pass of
`ConstantFoldingRule` alone has no mechanism to discover it.

One more correction belongs here, since the task that produced this
chapter's outline specifically named "rewriting `5 = x` into `x = 5`" as this
rule's example, and it's worth being explicit about why that specific claim
doesn't survive contact with the source. `ConstantOrderNormalizationRule`'s
matcher requires an `ExpressionClass::BOUND_FUNCTION` node whose function
name is exactly `+` or `*` (`SpecificFunctionMatcher("+")` /
`SpecificFunctionMatcher("*")`, `constant_order_normalization.cpp:55,66`) —
a `=` comparison is a different `ExpressionType` entirely
(`BoundComparisonExpression`, matched elsewhere by
`ComparisonExpressionMatcher`, not by anything in this file), and this
rule's `root` matcher simply never looks at one. The "which side does a
constant sit on in a comparison" problem is real, and several rules later in
this file (`MoveConstantsRule`, `MoveUnaryMinusRule` — §7.14) do have to
solve it, but they solve it a different way entirely: their matchers use
`ComparisonExpressionMatcher` with `SetMatcher::Policy::UNORDERED`
(`move_constants.cpp:19`, `move_constants.cpp:241`) — meaning the matcher
itself can find "one child matches `ConstantExpressionMatcher`, the other
matches the arithmetic-shape matcher" *regardless of which literal side of
the `=` each one is written on*, without needing any prior normalization
pass to have already sorted them into a canonical position. So the
architectural need this book's task description gestured at — "so that
later rules don't need to handle both orderings" — is real and correctly
placed in this file's outline, but the actual mechanism DuckDB uses to meet
that need for *comparisons* is baked into the matcher framework's
`Policy::UNORDERED` option (§7.10), not into
`ConstantOrderNormalizationRule`. This rule's real job — regrouping
scattered constants inside commutative/associative `+`/`*` chains so
`ConstantFoldingRule` has something contiguous to fold — is a related but
distinct concern, and conflating the two would misdescribe what
`constant_order_normalization.cpp` actually does.

## 7.13 `ArithmeticSimplificationRule` — algebraic identities

**Purpose, in one sentence:** replace an arithmetic expression with one
constant operand and one non-constant operand by the non-constant operand
alone (or by a constant), whenever the constant is an identity element
(`0` for `+`/`-`, `1` for `*`/`/`/`//`) or an absorbing element (`0` for
`*`, `NULL` for anything).

**Construction-order position:** #5 (`optimizer.cpp:78`), after
`ConstantOrderNormalizationRule` (#1), `ConstantFoldingRule` (#2),
`StructExtractStructPackFoldingRule` (#3), and `DistributivityRule` (#4);
before `LeastGreatestSimplificationRule` (#6).

### Source, and the full identity catalog

```cpp
// src/optimizer/rule/arithmetic_simplification.cpp:12-24
ArithmeticSimplificationRule::ArithmeticSimplificationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	// match on an OperatorExpression that has a ConstantExpression as child
	auto op = make_uniq<FunctionExpressionMatcher>();
	op->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
	op->matchers.push_back(make_uniq<ExpressionMatcher>());
	op->policy = SetMatcher::Policy::SOME;
	op->function = make_uniq<ManyFunctionMatcher>(identifier_set_t {"+", "-", "*", "/", "//"});
	// and only with numeric results
	op->type = make_uniq<NumericTypeMatcher>();
	op->matchers[0]->type = make_uniq<NumericTypeMatcher>();
	op->matchers[1]->type = make_uniq<NumericTypeMatcher>();
	root = std::move(op);
}
```

`Policy::SOME` (as opposed to `ORDERED`/`UNORDERED`, both of which require
*every* matcher in the list to find a match) means only one of the two
listed matchers (`ConstantExpressionMatcher`) needs to actually find a
matching child; the other (`ExpressionMatcher`, no constraints beyond
"numeric") matches essentially anything. Concretely: this rule requires
exactly one of the two operands to be a literal `BOUND_CONSTANT` — this is
the detail that rules out one identity the task explicitly asked to check
for. **`x - x` is not an identity this rule implements**, and it's worth
saying why precisely rather than just asserting it: `x - x` has *no*
constant child at all (both operands are the same non-constant expression),
so `ConstantExpressionMatcher` cannot match either side, and the whole rule
declines to match the node in the first place. A search of
`src/optimizer/` for anywhere else this identity might live turned up
nothing — `x - x → 0` does not appear to be an identity DuckDB's optimizer
implements anywhere, foldable-constant machinery aside (if `x` itself
happens to be foldable, `ConstantFoldingRule` would fold the whole thing
regardless, independently of this rule).

`Apply` implements the rest of the catalog:

```cpp
// src/optimizer/rule/arithmetic_simplification.cpp:26-89
unique_ptr<Expression> ArithmeticSimplificationRule::Apply(LogicalOperator &op, vector<reference<Expression>> &bindings,
                                                           bool &changes_made, bool is_root) {
	auto &root = bindings[0].get().Cast<BoundFunctionExpression>();
	auto &constant = bindings[1].get().Cast<BoundConstantExpression>();
	idx_t constant_child = root.GetChildren()[0].get() == &constant ? 0 : 1;
	// any arithmetic operator involving NULL is always NULL
	if (constant.GetValue().IsNull()) {
		return make_uniq<BoundConstantExpression>(Value(root.GetReturnType()));
	}
	auto &func_name = root.Function().GetName();
	if (func_name == "/") {
		if (constant_child == 1 && constant.GetValue() == 1 &&
		    (root.GetChildren()[0]->GetReturnType().id() == LogicalTypeId::FLOAT ||
		     root.GetChildren()[0]->GetReturnType().id() == LogicalTypeId::DOUBLE)) {
			return std::move(root.GetChildrenMutable()[0]);
		}
		return nullptr;
	}
	if (!root.GetReturnType().IsIntegral()) {
		return nullptr;
	}
	if (func_name == "+") {
		if (constant.GetValue() == 0) {
			return Expression::PreserveReturnType(root.GetReturnType(),
			                                      std::move(root.GetChildrenMutable()[1 - constant_child]));
		}
	} else if (func_name == "-") {
		if (constant_child == 1 && constant.GetValue() == 0) {
			return Expression::PreserveReturnType(root.GetReturnType(),
			                                      std::move(root.GetChildrenMutable()[1 - constant_child]));
		}
	} else if (func_name == "*") {
		if (constant.GetValue() == 1) {
			return Expression::PreserveReturnType(root.GetReturnType(),
			                                      std::move(root.GetChildrenMutable()[1 - constant_child]));
		} else if (constant.GetValue() == 0) {
			return ExpressionRewriter::ConstantOrNull(GetContext(),
			                                          std::move(root.GetChildrenMutable()[1 - constant_child]),
			                                          Value::Numeric(root.GetReturnType(), 0));
		}
	} else if (func_name == "//") {
		if (constant_child == 1) {
			if (constant.GetValue() == 1) {
				return Expression::PreserveReturnType(root.GetReturnType(),
				                                      std::move(root.GetChildrenMutable()[1 - constant_child]));
			} else if (constant.GetValue() == 0 && !Settings::Get<ErrorOnDivisionByZeroSetting>(rewriter.context)) {
				return make_uniq<BoundConstantExpression>(Value(root.GetReturnType()));
			}
		}
	} else {
		throw InternalException("Unrecognized function name in ArithmeticSimplificationRule");
	}
	return nullptr;
}
```

Reading this top to bottom gives the exact, complete identity catalog — no
paraphrase needed, the branches speak for themselves, but three structural
details are easy to miss on a first read and change what the rule actually
covers:

1. **The `NULL`-constant check runs first, unconditionally, for every one of
   the five operators.** Any of `x+NULL`, `x-NULL`, `x*NULL`, `x/NULL`,
   `x//NULL` folds straight to a typed `NULL` — reflecting that every one of
   these arithmetic operators propagates `NULL` unconditionally in SQL. This
   is a real, source-supported identity, but this file found **no direct
   test coverage for it** in `test/optimizer/arithmetic_simplification.test`
   specifically — that file's own `test(X INTEGER)` table and its query list
   never construct an arithmetic expression against a literal `NULL`
   constant. The identity is real and unconditional in the code; it simply
   isn't the thing this particular `.test` file was written to demonstrate.
2. **The `/` (true division) branch is handled entirely separately, before
   the integral-return-type gate below it**, and only fires for a divide-by-`1`
   where the *non-constant* child's return type is specifically `FLOAT` or
   `DOUBLE`. This matters for the test file at hand: `arithmetic_simplification.test`
   declares `test(X INTEGER)` and only ever tests `X//1` (floor/integer
   division), never `X/1` (true division) — and given DuckDB's `/` operator
   promotes integer operands to `DOUBLE` at bind time, the identity this
   branch implements genuinely requires a floating-point operand to exercise
   at all. **This means the `/`-by-`1` branch has no coverage in this
   specific test file** — it is real, reachable code, just not reachable by
   anything `arithmetic_simplification.test` actually runs.
3. **Every remaining branch (`+`, `-`, `*`, `//`) is gated by
   `if (!root.GetReturnType().IsIntegral()) return nullptr;`** — meaning
   `x + 0.0` for a `DOUBLE`-typed `x` is *not* simplified by this rule at
   all (only the `/`-by-`1.0` branch above handles floating-point types, and
   only for division specifically). This is a real, if slightly surprising,
   scope limit worth stating plainly: outside of `/1.0`, this rule's
   identities are integer-only.

### `test/optimizer/arithmetic_simplification.test`, walked through

The `xnorm` group (`test/optimizer/arithmetic_simplification.test:12-38`) is
one shared label spanning six different `SELECT`s, all claimed identical to
plain `SELECT X FROM test`:

```sql
query I nosort xnorm
EXPLAIN SELECT X FROM test
----

query I nosort xnorm
EXPLAIN SELECT X+0 FROM test
----

query I nosort xnorm
EXPLAIN SELECT 0+X FROM test
----

query I nosort xnorm
EXPLAIN SELECT X-0 FROM test
----

query I nosort xnorm
EXPLAIN SELECT X*1 FROM test
----

query I nosort xnorm
EXPLAIN SELECT 1*X FROM test
----

query I nosort xnorm
EXPLAIN SELECT X//1 FROM test
----
```

Each of `X+0`, `0+X`, `X*1`, `1*X`, and `X//1` is directly one of the
identity branches above (`+` with constant `0` on either side — the rule
uses `1 - constant_child` to pick "the other child" regardless of which
index the constant occupies, so `0+X` and `X+0` are both covered by the same
`if (constant.GetValue() == 0)` check; `*` with constant `1` on either side,
same mechanism; `//` with constant `1`, gated additionally to
`constant_child == 1`, i.e. specifically the `X // 1` shape, not `1 // X`).
`X-0` is the one entry gated to `constant_child == 1` for `-` specifically
(subtraction is not commutative, so `0 - X` is deliberately *not* covered by
this branch — and indeed isn't in this test group at all). Each of these
six collapses to bare `X`, matching the first block's plain
`SELECT X FROM test`.

The `xnull` group (`arithmetic_simplification.test:40-56`) covers the
division-by-zero edge case directly, and shows the rule respecting a
session setting rather than a fixed identity:

```sql
# division by zero is an error by default and is not simplified away
statement error
SELECT X//0 FROM (VALUES (1)) t(X)
----
<REGEX>:.*Division by zero.*

# with error_on_division_by_zero disabled, division by zero results in a NULL
statement ok
SET error_on_division_by_zero=false;

query I nosort xnull
EXPLAIN SELECT NULL::INTEGER FROM test
----

query I nosort xnull
EXPLAIN SELECT X//0 FROM test
----
```

`Settings::Get<ErrorOnDivisionByZeroSetting>(rewriter.context)` is checked
*inside* the `//`-by-`0` branch specifically — this is one of only a
handful of places in this file's seven rules where a rule's rewrite decision
depends on session configuration rather than purely on the expression's own
shape and type. With the default setting (errors enabled), the rule
declines the `X // 0` rewrite entirely (there's nothing to fold to — the
correct behavior is an error, and an error can't be represented as a
`BoundConstantExpression`, so the expression is left untouched and the
error is raised later, at actual execution time, by the division function
itself). With the setting disabled, `X // 0` folds directly to a typed
`NULL`, matching the second half of the test.

Finally, `test/optimizer/constant_or_null_value_arg.test` provides direct,
regex-checked confirmation of the `x * 0` absorbing-element branch's
specific output shape — worth including here because it pins down exactly
*how* NULL-propagation is preserved, which `Value::Numeric(root.GetReturnType(), 0)`
plus `ExpressionRewriter::ConstantOrNull` alone doesn't make obvious from the
source snippet:

```sql
# arithmetic_simplification rewrites x*0 -> constant_or_null(0, x). The one-arg
# ConstantOrNull(child, value) overload must NOT push `value` into children before
# delegating to the vector overload (which inserts value itself); otherwise the
# plan carries constant_or_null(0, 0, x) instead of constant_or_null(0, x).

# regression guard: the duplicated form must NOT appear
query II
EXPLAIN SELECT count(*) FROM t WHERE x * 0 = 1
----
physical_plan	<!REGEX>:.*constant_or_null\(0, 0,.*

# the non-duplicated form (value once, then child) must appear
query II
EXPLAIN SELECT count(*) FROM t WHERE x * 0 = 1
----
physical_plan	<REGEX>:.*constant_or_null\(0, x\).*
```

The `<!REGEX>` marker on the first block is a negative assertion — this
specific pair isn't a same-label proof at all, it's two independent regex
checks against one query's plan, the first confirming a bug that *used* to
exist (a duplicated `value` argument) does not appear, the second confirming
the correct, singular form does.

`constant_or_null(0, x)` reads exactly as it looks: "return `0`, unless `x`
is `NULL`, in which case return `NULL`" — i.e. `x * 0` does *not* simply
become the bare literal `0` (which would be wrong: `NULL * 0` is `NULL`, not
`0`, under standard SQL null-propagation), it becomes a call to a small
built-in helper function that reconstructs exactly that NULL-preserving
behavior while still avoiding a real multiplication at runtime. Every place
this file's rules use `ExpressionRewriter::ConstantOrNull` (this rule's
`x*0` case; `MoveConstantsRule` and `MoveUnaryMinusRule`'s out-of-range
cases in §7.14) is doing the same thing: producing a provably-correct
constant answer for every *non-null* input while still leaving room for
`NULL` to propagate through untouched.

### Order and interactions

Position #5 places `ArithmeticSimplificationRule` after
`ConstantOrderNormalizationRule` (#1) and `ConstantFoldingRule` (#2) but
before the large block of `*SimplificationRule`s (`CaseSimplificationRule`
at #7 onward) and well before `MoveConstantsRule`/`MoveUnaryMinusRule`
(#17/#18, §7.14). This ordering has one concrete, testable consequence
already visible in §7.12's `add_mult` trace: by the time
`ArithmeticSimplificationRule` gets a turn at any given node, both
constant-regrouping and constant-folding have already had their turns
(reset-and-redo per §7.10 guarantees this within a single visit), so a
chain like `col0 * 1 * 2` is already `2 * col0` — a single constant, already
folded — by the time `ArithmeticSimplificationRule` would even consider it;
since `2` isn't an identity element for `*`, this rule correctly declines,
leaving `2 * col0` as the final answer rather than, say, mistaking it for a
`*1` no-op. The one interaction worth flagging with `MoveConstantsRule`
specifically: nothing prevents `ArithmeticSimplificationRule` from firing
*inside* a comparison — e.g. `(x + 0) = 10` has its `x + 0` sub-node visited
(bottom-up) before the outer `=` node, so by the time `MoveConstantsRule`'s
own matcher would consider the comparison, `ArithmeticSimplificationRule`
has often already simplified the arithmetic child down to bare `x`,
leaving `MoveConstantsRule` nothing to do for that particular shape (there's
no more "constant to move" once the `+0` has already vanished). The two
rules are not in conflict — they overlap in *which* cases they can each
independently resolve, and DuckDB's fixed rule order simply lets whichever
one matches first (by list position, at that specific node, on that
specific visit) do the work.

## 7.14 `MoveConstantsRule` and `MoveUnaryMinusRule` — isolating the column

**Purpose, in one sentence (both rules together):** rearrange a comparison
whose non-constant side is wrapped in simple integer arithmetic (`+`, `-`,
`*` for `MoveConstantsRule`; a leading unary minus for `MoveUnaryMinusRule`)
so that the bare column reference (or other non-constant expression) ends
up alone on one side of the comparison and a single, recomputed constant
ends up alone on the other — i.e. turning `x + 5 = 10` into `x = 5`, not
just simplifying the arithmetic but *moving* it across the comparison
operator entirely.

**Construction-order position:** #17 (`MoveConstantsRule`) and #18
(`MoveUnaryMinusRule`), both at `optimizer.cpp:90-91` — well after every
rule discussed so far in this file, and after the whole run of `*SimplificationRule`s
(`CaseSimplificationRule` through `EqualOrNullSimplification`, positions
#7–#16).

### Source: `MoveConstantsRule`

```cpp
// src/optimizer/rule/move_constants.cpp:16-38
MoveConstantsRule::MoveConstantsRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	auto op = make_uniq<ComparisonExpressionMatcher>();
	op->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
	op->policy = SetMatcher::Policy::UNORDERED;

	auto arithmetic = make_uniq<FunctionExpressionMatcher>();
	// we handle multiplication, addition and subtraction because those are "easy"
	// integer division makes the division case difficult
	// e.g. [x / 2 = 3] means [x = 6 OR x = 7] because of truncation -> no clean rewrite rules
	arithmetic->function =
	    make_uniq<ManyFunctionMatcher>(identifier_set_t {Identifier("+"), Identifier("-"), Identifier("*")});
	// we match only on integral numeric types
	arithmetic->type = make_uniq<IntegerTypeMatcher>();
	auto child_constant_matcher = make_uniq<ConstantExpressionMatcher>();
	auto child_expression_matcher = make_uniq<ExpressionMatcher>();
	child_constant_matcher->type = make_uniq<IntegerTypeMatcher>();
	child_expression_matcher->type = make_uniq<IntegerTypeMatcher>();
	arithmetic->matchers.push_back(std::move(child_constant_matcher));
	arithmetic->matchers.push_back(std::move(child_expression_matcher));
	arithmetic->policy = SetMatcher::Policy::SOME;
	op->matchers.push_back(std::move(arithmetic));
	root = std::move(op);
}
```

The pattern this builds: a comparison (`Policy::UNORDERED`, so the two
things it's looking for — a bare constant, and an arithmetic expression —
can appear on either side) where one side is a plain constant and the other
is a `+`/`-`/`*` call over integer types with one constant child of its own.
The source comment explains, precisely, why `/` (true division) is
deliberately excluded: `x / 2 = 3` cannot be cleanly inverted into a single
equivalent predicate on `x` alone, because integer division truncates —
both `x = 6` and `x = 7` satisfy `x / 2 = 3` (in truncating-toward-zero
integer division), so there is no single rewritten comparison that's
equivalent, and the rule declines rather than produce something subtly
wrong. `Apply` (`move_constants.cpp:54-196`) then, per operator:

- **`+`**: `x + c1 COMP c2` (either operand order) becomes `x COMP (c2 - c1)`
  — addition commutes, so no comparison-direction flip is ever needed.
- **`-`**: direction-sensitive. `x - c1 COMP c2` becomes `x COMP (c2 + c1)`;
  but `c1 - x COMP c2` (the constant *first*, `x` subtracted from it)
  becomes `x COMP' (c1 - c2)` with the comparison **flipped**
  (`BoundComparisonExpression::FlipType(comparison)`,
  `move_constants.cpp:134`) — the source comment gives the intuition
  directly: "if we have `[4 - x < 2]` then we should have `[x > 2]`."
  Subtracting `x` from a constant reverses the direction of the underlying
  inequality, so the comparison operator has to flip along with the
  rearrangement to stay equivalent.
- **`*`**: `x * c1 COMP c2` (or `c1 * x COMP c2`) becomes `x COMP (c2 / c1)`
  — but only when `c2` is **cleanly divisible** by `c1`
  (`outer_value % inner_value != 0` is checked explicitly,
  `move_constants.cpp:150`); if it isn't, an equality or inequality
  comparison can still be resolved to a definite `TRUE`/`FALSE`/`NULL`
  (no integer `x` can make `x*c1` land on a value `c2` isn't a multiple of,
  for `=`/`<>`), but an ordered comparison (`<`, `<=`, `>`, `>=`) is left
  alone entirely rather than risk an off-by-one error from naive integer
  division. And if `c1` is negative, the comparison flips
  (`move_constants.cpp:179-182`), for the same reason multiplying an
  inequality by a negative number reverses it in ordinary algebra.

Two more source details worth citing directly, because they're easy to miss
and both matter for correctness rather than being purely cosmetic:

```cpp
// src/optimizer/rule/move_constants.cpp:62-68
if (inner_constant.GetValue().IsNull() || outer_constant.GetValue().IsNull()) {
	if (comparison.GetExpressionType() == ExpressionType::COMPARE_DISTINCT_FROM ||
	    comparison.GetExpressionType() == ExpressionType::COMPARE_NOT_DISTINCT_FROM) {
		return nullptr;
	}
	return make_uniq<BoundConstantExpression>(Value(comparison.GetReturnType()));
}
```

A `NULL` on either side of the comparison folds the whole thing to a typed
`NULL` immediately — *except* for `IS [NOT] DISTINCT FROM`, which is
specifically the SQL comparison operator designed to treat `NULL` as an
ordinary, comparable value rather than propagate it, so the rule correctly
refuses to touch those two comparison types at all here.

```cpp
// src/optimizer/rule/move_constants.cpp:142-146
if (inner_value == 0) {
	// x * 0, the result is either 0 or NULL
	// we let the arithmetic_simplification rule take care of simplifying this first
	return nullptr;
}
```

This is an explicit, in-source acknowledgment of the rule-ordering
discussion in §7.13: `MoveConstantsRule` deliberately declines `x * 0` and
defers to `ArithmeticSimplificationRule`, which the comment names by name.
This is one of the few places in the entire 37-rule list where one rule's
source comment explicitly documents its intended handoff to another rule
by name, rather than leaving the reader to infer the interaction from
construction order alone.

### Source: `MoveUnaryMinusRule`

```cpp
// src/optimizer/rule/move_constants.cpp:238-251
MoveUnaryMinusRule::MoveUnaryMinusRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	auto op = make_uniq<ComparisonExpressionMatcher>();
	op->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
	op->policy = SetMatcher::Policy::UNORDERED;

	// match a unary minus over a numeric expression, e.g. [-x = -5] or [-x < -3.14]
	auto negation = make_uniq<FunctionExpressionMatcher>();
	negation->function = make_uniq<SpecificFunctionMatcher>("-");
	negation->type = make_uniq<NumericTypeMatcher>();
	negation->matchers.push_back(make_uniq<ExpressionMatcher>());
	negation->policy = SetMatcher::Policy::ORDERED;
	op->matchers.push_back(std::move(negation));
	root = std::move(op);
}
```

This handles the unary (single-child) form of `-`, distinct from
`MoveConstantsRule`'s binary arithmetic — `-x COMP c` becomes `x COMP' -c`,
comparison flipped, the same "negating both sides of an inequality reverses
it" logic as the `-`-branch above. `Apply` (`move_constants.cpp:253-360`)
handles this across every numeric representation DuckDB has — integers
(via `hugeint_t` arithmetic and `Hugeint::TryNegate`), `DECIMAL` (negating
the unscaled integer while preserving width/scale, with a `switch` over
every backing `PhysicalType`), and floating point — and the floating-point
path is where this rule earns its most interesting edge case.

### `test/optimizer/move_constants.test`, in depth

**`add_left`/`add_right`** (`move_constants.test:12-34`): `X+1=10`,
`1+X=10`, and `10=X+1`, `10=1+X` all normalize to `X=9` — the "+ commutes,
no flip needed" case, both comparison directions (constant-on-left,
constant-on-right) handled by the matcher's `UNORDERED` policy without a
separate normalization step.

**`subtract_left_right` / `subtract_left_left`** (`move_constants.test:37-51`):
`X-1=10` → `X=11` (subtracting the constant: add it back to the RHS); but
`10-X=5` → `X=5`, and critically **without needing a comparison flip here**,
because `10-5=5` happens to be self-symmetric for this specific numeric
example — the *general* case, where a flip genuinely changes the answer, is
covered next.

**`subtract_flipped_lt` / `subtract_flipped_gte`** (`move_constants.test:54-68`):

```sql
query I nosort subtract_flipped_lt
EXPLAIN SELECT 10-X<5 FROM test
----

query I nosort subtract_flipped_lt
EXPLAIN SELECT X>5 FROM test
----

query I nosort subtract_flipped_gte
EXPLAIN SELECT 10-X>=5 FROM test
----

query I nosort subtract_flipped_gte
EXPLAIN SELECT X<=5 FROM test
----
```

`10 - X < 5` becoming `X > 5` is exactly the source comment's own example
(`[4 - x < 2]` → `[x > 2]`, restated here with different numbers): moving
`X` from the subtracted position across the inequality reverses which
direction the inequality points, and the rule's explicit
`BoundComparisonExpression::FlipType` call (`move_constants.cpp:134`) is
precisely what makes this test pass rather than silently producing a
plan equivalent to the *wrong* inequality.

**`mult_left` / `mult_left_gt`** (`move_constants.test:71-90`): `X*3=6`,
`3*X=6` → `X=2` (cleanly divisible, no flip — multiplication by a positive
constant doesn't reverse direction); `X*3>3` → `X>1` (an ordered comparison,
still cleanly divisible, still no flip since `3` is positive).

**`mult_left_negative` / `mult_left_negative_flip`** (`move_constants.test:93-107`):
`-1*X=-5` → `X=5` (equality, no flip needed — negating both sides of an
equality doesn't change which values satisfy it); but `-1*X<-5` → `X>5`
(inequality *does* flip when the multiplier is negative,
`move_constants.cpp:179-182`, exactly analogous to ordinary algebra's rule
that multiplying an inequality by a negative number reverses it).

**`negation`/`negation_flip`/`negation_flip_left`** (`move_constants.test:110-133`):
the `MoveUnaryMinusRule` cases — `-X=-5` → `X=5` (no flip, equality);
`-X<-5` → `X>5` (flip, inequality); and, showing the matcher's
`UNORDERED` policy handling the constant on the *left* this time,
`-5>-X` → `5<X` (same underlying rewrite, opposite literal side).

**Out-of-range constants** (`move_constants.test:135-161`): a `TINYINT`
column `x` against `-x = -128` — `-128` is `TINYINT`'s minimum, and its
negation, `128`, doesn't fit in `TINYINT` at all
(`TINYINT`'s range is `-128..127`). Rather than silently wrapping or
erroring, `MoveUnaryMinusRule`'s `DefaultTryCastAs` failure path
(`move_constants.cpp:333-341`) recognizes that *no* representable `TINYINT`
value can make this equality true, and folds the whole comparison to a
provably-safe constant using `ExpressionRewriter::ConstantOrNull`:

```sql
query I
SELECT count(*) FROM tiny WHERE -x = -128
----
0

query I
SELECT count(*) FROM tiny WHERE -x != -128
----
5
```

Five non-`NULL` rows in `tiny` (`-127, -1, 0, 5, 127`), all correctly
counted as satisfying `!=` and none satisfying `=` — the rule proved the
equality unconditionally false for any representable input without ever
comparing a single stored value at runtime, while still correctly excluding
the one `NULL` row from both counts (per `NULL`'s ordinary comparison
semantics, preserved by `ConstantOrNull`).

**The NaN guard** (`move_constants.test:174-292`) is the single most
interesting edge case attached to either rule, and it exists specifically
because IEEE-754 floating point has a value — `NaN` — that breaks the
"negating an inequality reverses it" assumption every integer/decimal case
above relies on unconditionally. DuckDB defines `NaN` as sorting *above*
every other value (an explicit design choice the source comment states
directly), which means `-x > c` and `-x < c` are not perfect mirror images
of `x < -c` and `x > -c` once `x` might be `NaN`: negating `NaN` is still
`NaN`, and `NaN`'s ordering behavior under negation doesn't flip the way a
literal ordinary number's would. The source spells out the exact
compensating logic in a comment worth reproducing verbatim:

```cpp
// src/optimizer/rule/move_constants.cpp:229-236
//! -NaN is still NaN, and DuckDB orders NaN above every other value, so [-x > c] and [-x >= c] are TRUE for a
//! NaN input while the flipped [x < -c] / [x <= -c] are FALSE. Those two need [OR isnan(x)]; [-x < c] and
//! [-x <= c] have the mirrored problem and need [AND NOT isnan(x)]. The comparison type passed here must
//! already be oriented with the negation on the left.
static bool NaNGuardIsDisjunctive(ExpressionType comparison_type) {
	return comparison_type == ExpressionType::COMPARE_GREATERTHAN ||
	       comparison_type == ExpressionType::COMPARE_GREATERTHANOREQUALTO;
}
```

The test confirms the guard is actually attached to the rewritten expression,
not just present in a comment:

```sql
query I nosort float_neg_lt
EXPLAIN SELECT -x < -2.5 FROM floats
----

query I nosort float_neg_lt
EXPLAIN SELECT x > 2.5 AND NOT isnan(x) FROM floats
----
```

`-x < -2.5` doesn't simplify to the bare `x > 2.5` an integer/decimal case
would produce — it becomes `x > 2.5 AND NOT isnan(x)`, because if `x` were
`NaN`, `-x` would still be `NaN`, and `NaN < -2.5` is `FALSE` under DuckDB's
ordering, while a naive `x > 2.5` (with `x = NaN`) would also evaluate the
comparison per the same ordering rules and — since `NaN` sorts *above*
everything — return `TRUE`, disagreeing with the original expression. The
explicit `AND NOT isnan(x)` patches exactly that one disagreeing input. The
test file even verifies this guard is itself conditionally elidable:
when a column is declared `NOT NULL` and statistics prove it can never
actually contain `NaN` (`floats_no_nan`, `move_constants.test:220-241`), the
guard doesn't appear in the rewritten plan at all — that particular
narrowing is `STATISTICS_PROPAGATION`'s job (pass #34 in the overview's
pipeline, not this rule), confirmed here by explicitly disabling it
(`SET disabled_optimizers='statistics_propagation'`,
`move_constants.test:244`) and observing the guard reappear.

### Why isolating a bare column reference matters downstream

The task that produced this chapter's outline asked, specifically, for this
connection to be confirmed concretely from source rather than asserted on
authority — and it is confirmable, directly, in
`src/optimizer/statistics/operator/propagate_filter.cpp`. The function
`StatisticsPropagator::UpdateFilterStatistics(const Expression &left, const Expression &right, ExpressionType comparison_type)`
is what narrows a column's known value range (min/max) whenever
`STATISTICS_PROPAGATION` (pass #34) walks past a comparison filter, and its
very first move is a structural shape check:

```cpp
// src/optimizer/statistics/operator/propagate_filter.cpp:173-199
// check if this is a comparison between a constant and a column ref
optional_ptr<const BoundConstantExpression> constant;
optional_ptr<const BoundColumnRefExpression> columnref;
if (left.GetExpressionType() == ExpressionType::VALUE_CONSTANT &&
    right.GetExpressionType() == ExpressionType::BOUND_COLUMN_REF) {
	constant = &left.Cast<BoundConstantExpression>();
	columnref = &right.Cast<BoundColumnRefExpression>();
	comparison_type = FlipComparisonExpression(comparison_type);
} else if (left.GetExpressionType() == ExpressionType::BOUND_COLUMN_REF &&
           right.GetExpressionType() == ExpressionType::VALUE_CONSTANT) {
	columnref = &left.Cast<BoundColumnRefExpression>();
	constant = &right.Cast<BoundConstantExpression>();
} else if (/* column ref vs column ref */) {
	// ...
} else {
	// unsupported filter
	return;
}
if (constant && columnref) {
	auto entry = statistics_map.find(columnref->Binding());
	if (entry == statistics_map.end()) {
		return;
	}
	UpdateFilterStatistics(*entry->second, comparison_type, constant->GetValue());
}
```

This is source-grounded, not inferred: `UpdateFilterStatistics` only knows
how to narrow a column's statistics when one side of the comparison is
*exactly* `ExpressionType::VALUE_CONSTANT` and the other is *exactly*
`ExpressionType::BOUND_COLUMN_REF` (or column-ref-vs-column-ref, a separate
branch this file doesn't need). Every other shape — including a column
wrapped in live arithmetic, like `x + 5` — falls through to the `else`
branch labeled, in the source's own words, `"unsupported filter"`, and the
function returns having narrowed nothing. So a predicate that reaches
`STATISTICS_PROPAGATION` still shaped as `x + 5 = 10` genuinely does *not*
narrow `x`'s known range at all under this code path — it would need to be
`x = 5` first. `MoveConstantsRule`, running earlier in the same
`EXPRESSION_REWRITER` pass (#1) that runs long before `STATISTICS_PROPAGATION`
(#34) ever gets a turn, is precisely what turns `x + 5 = 10` into `x = 5`
before this statistics code ever sees it — meaning the connection this
section set out to confirm is not merely plausible, it's the literal,
documented (via the `"unsupported filter"` comment) behavior of the
concrete function that would otherwise be blocked. The same shape
requirement — a bare `BOUND_COLUMN_REF` on one side — is exactly what
DuckDB's zonemap-based row-group pruning and any index lookup would also
need to recognize "this predicate is about column `x`, plainly," so while
this file's evidence is specific to `STATISTICS_PROPAGATION`'s own code
path (the one place this was concretely checked), the same underlying
reasoning — "wrapping arithmetic hides the column from anything downstream
that pattern-matches on `BOUND_COLUMN_REF` structurally" — plausibly extends
to those other consumers too, and that broader extension is this book's own
reasoning rather than something separately confirmed line-by-line in this
pass.

## 7.15 `DistributivityRule` — factoring a common conjunct out of a disjunction

**Purpose, in one sentence, direction confirmed from source rather than
assumed:** given an `OR` whose branches are (or contain) `AND`s, find every
term common to *all* branches and factor it out to the front, rewriting
`(a AND b) OR (a AND c)` into `a AND (b OR c)` — i.e., the "factor a common
term out of a disjunction" direction, not the reverse "distribute a term
into a disjunction" direction a more careless reading of the class's name
might suggest.

**Construction-order position:** #4 (`optimizer.cpp:77`), after
`ConstantOrderNormalizationRule` (#1), `ConstantFoldingRule` (#2), and
`StructExtractStructPackFoldingRule` (#3); before `ArithmeticSimplificationRule`
(#5).

### Source

The matcher is almost trivially narrow — this rule looks only for a bare
`OR` node, anywhere in the expression tree, not specifically inside a
`LogicalFilter`:

```cpp
// src/optimizer/rule/distributivity.cpp:11-15
DistributivityRule::DistributivityRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	// we match on an OR expression within a LogicalFilter node
	root = make_uniq<ExpressionMatcher>();
	root->expr_type = make_uniq<SpecificExpressionTypeMatcher>(ExpressionType::CONJUNCTION_OR);
}
```

(The constructor's own comment says "within a `LogicalFilter` node," but the
matcher itself has no such restriction — it's a bare
`ExpressionType::CONJUNCTION_OR` filter, and the bottom-up walk in §7.10
visits every expression in every operator's expression list, `LogicalFilter`
or otherwise, so this rule fires on an `OR` inside a `CASE WHEN` condition
or a `SELECT` boolean column just as readily. The comment describes the
rule's most common home, not an enforced constraint.)

`Apply`'s algorithm, worth walking through mechanically since it's the
richest single piece of logic in this file: given `initial_or`'s children
(each either a plain expression or itself an `AND` of several), build a
per-child *set* of "terms this branch contains" (an `AND`'s children
individually, or — for a non-`AND` branch — the single expression itself,
treated as a one-term set):

```cpp
// src/optimizer/rule/distributivity.cpp:17-26
void DistributivityRule::AddExpressionSet(Expression &expr, expression_set_t &set) {
	if (expr.GetExpressionType() == ExpressionType::CONJUNCTION_AND) {
		auto &and_expr = expr.Cast<BoundConjunctionExpression>();
		for (auto &child : and_expr.GetChildrenMutable()) {
			set.insert(*child);
		}
	} else {
		set.insert(expr);
	}
}
```

Then intersect these per-branch sets across every branch of the `OR`
(`distributivity.cpp:65-80`) — what survives the intersection is exactly
the set of terms present in *every single branch*, structural equality
(`expression_set_t` hashes/compares by `Expression::Equals`, not pointer
identity, so two independently-parsed copies of the same sub-expression
still count as "the same term"). If nothing survives, the rule declines
(`candidate_set.empty()` → `return nullptr`). If something does, each
surviving term is removed from every branch (`ExtractExpression`,
`distributivity.cpp:28-55` — if a branch is an `AND`, the term is erased
from that `AND`'s child list, collapsing the `AND` to its single remaining
child if only one term is left; if a branch *is* the term itself with no
surrounding `AND`, the whole branch is nulled out) and gathered into a fresh
`AND` at the front of the result, with whatever's left of the original `OR`
(if anything) appended as its second operand.

### `test/optimizer/distributivity_rule.test`, in full

**`distributivity1`** (`distributivity_rule.test:12-18`) is the richest
worked case, a genuine three-way factoring:

```sql
query I nosort distributivity1
EXPLAIN SELECT (X AND A AND B) OR (A AND X AND C) OR (X AND B AND D) FROM test
----

query I nosort distributivity1
EXPLAIN SELECT X AND ((A AND B) OR (A AND C) OR (B AND D)) FROM test
----
```

Per-branch term sets: `{X, A, B}`, `{A, X, C}`, `{X, B, D}`. `A` appears in
branches 1 and 2 but not 3; `B` appears in branches 1 and 3 but not 2 — only
`X` survives the three-way intersection. `ExtractExpression` pulls `X` out
of every branch (leaving `{A,B}`, `{A,C}`, `{B,D}` respectively), the
surviving `X` becomes the first child of a fresh `AND`, and the three
term-depleted branches are re-assembled into a fresh `OR` as the `AND`'s
second child — exactly `X AND ((A AND B) OR (A AND C) OR (B AND D))`.

**`distributivity2`** (`distributivity_rule.test:20-27`) is the textbook
two-branch case the task's own outline names directly:

```sql
# (X AND B) OR (X AND C) => X AND (B OR C)
query I nosort distributivity2
EXPLAIN SELECT (X AND B) OR (X AND C) FROM test
----

query I nosort distributivity2
EXPLAIN SELECT X AND (B OR C) FROM test
----
```

**`distributivity3`** (`distributivity_rule.test:30-40`) exercises the
degenerate case where the candidate set turns out to be the *entire*
content of the `OR` — `X OR X` (and its four-way and explicitly-nested
variants):

```sql
query I nosort distributivity3
EXPLAIN SELECT X OR X FROM test
----

query I nosort distributivity3
EXPLAIN SELECT X OR X OR X OR X FROM test
----

query I nosort distributivity3
EXPLAIN SELECT X OR (X OR (X OR X)) FROM test
----
```

All three are asserted identical to each other (one shared label spans all
three plus the earlier plain-`X` block from `distributivity1`'s second
pair's absence — actually, this label group stands alone). For `X OR X`:
each branch is a bare (non-`AND`) expression, so each contributes a
singleton set `{X}`; the intersection of `{X}` and `{X}` is `{X}`.
`ExtractExpression` on a non-`AND` branch nulls out the whole branch
(there's nothing left of it once its one term is removed), so *both*
original branches of the `OR` end up null, and `Apply`'s bookkeeping for
"we completely erased a branch" (`distributivity.cpp:101-116`) short-circuits
straight to returning the single surviving candidate term, `X`, with no
residual `OR` at all — `X OR X` becomes exactly `X`, not `X AND (TRUE)` or
any other more roundabout equivalent. The four-way and explicitly-nested
variants reach the same result via the same mechanism applied recursively:
`X OR (X OR (X OR X))`'s innermost `X OR X` collapses to `X` first (visited
first, bottom-up, per §7.10), leaving `X OR (X OR X)`, whose inner `X OR X`
then also collapses (reset-and-redo revisits it), leaving `X OR X`, which
collapses one final time to bare `X` — the same single rule, reapplied
repeatedly to progressively shallower copies of the same shape, entirely
within one visit to the outermost node, exactly the mechanism §7.10
described in the abstract.

**`distributivity4`** (`distributivity_rule.test:43-49`) is a Boolean
absorption law, `X OR (X AND A) → X`:

```sql
# X OR (X AND A) => X
query I nosort distributivity4
EXPLAIN SELECT X OR (X AND A) FROM test
----

query I nosort distributivity4
EXPLAIN SELECT X OR X FROM test
----
```

Per-branch sets: `{X}` (branch 1, not an `AND`) and `{X, A}` (branch 2, is
an `AND`). Intersection: `{X}`. Extracting `X` from branch 1 (a non-`AND`
branch) nulls that branch entirely; extracting `X` from branch 2 (an `AND`)
leaves just `{A}` behind, so branch 2 becomes bare `A` rather than being
nulled. The "did we completely erase a branch" check
(`distributivity.cpp:108-116`) finds branch 1 nulled, and since the
candidate-holding `new_root` at that point has exactly one child (`X`),
returns that child directly — discarding the leftover `A` from branch 2
entirely, which is exactly the semantic content of the absorption law: a
row satisfying bare `X` always satisfies `X OR (X AND A)` regardless of `A`,
and a row failing `X` can't satisfy either branch, so `A`'s truth value is
provably irrelevant to the whole expression's outcome. The expected
`EXPLAIN` in the test — `X OR X`, not bare `X` — is worth noting precisely:
the same-label proof here confirms the rewrite lands on a plan equivalent
to `X OR X` specifically (which, per `distributivity3` just above, is
itself further rewritten to bare `X` — the test's chosen second query is
one more application of this exact rule away from the simplest possible
form, not the simplest form itself, and the label only asserts the two
sides shown are equal to *each other*, which they are).

**`distributivity5`** (`distributivity_rule.test:52-60`) confirms the whole
mechanism works over arbitrary expressions, not just bare boolean columns —
`expression_set_t`'s equality is structural (`Expression::Equals`), so a
repeated non-trivial subexpression like `X=1` is recognized as "the same
term" across branches exactly as a bare boolean column would be:

```sql
query I nosort distributivity5
EXPLAIN SELECT (X=1 AND Y=1) OR (X=1 AND Z=1) FROM test2
----

query I nosort distributivity5
EXPLAIN SELECT X=1 AND (Y=1 OR Z=1) FROM test2
----
```

### Order and interactions

Position #4 places `DistributivityRule` after `ConstantFoldingRule` (#2) but
before `ArithmeticSimplificationRule` (#5) and every later
`*SimplificationRule`. Since this rule's matcher looks only for
`ExpressionType::CONJUNCTION_OR` and its algorithm reasons about term
equality via structural `Equals` comparison (not evaluation), running
before the later arithmetic/comparison-simplifying rules doesn't create any
correctness dependency the way §7.12's ordering did — a term like `X=1`
would be recognized as identical across two branches whether or not `1` had
already been touched by anything else, since it's still the same literal
constant either way. The one place order plausibly matters here is exactly
the opposite direction from most of this file's examples: `ConstantFoldingRule`
(#2) already ran by the time `DistributivityRule` (#4) sees an expression,
so if factoring exposed a fully-foldable term (this file found no test case
where it does, but the mechanism doesn't preclude it), a subsequent visit
triggered by this rule's own changes would still get a chance at
`ConstantFoldingRule` via the same reset-and-redo traversal — the fixed
*list* order doesn't block a rule positioned earlier in the list from
getting a second look at newly-produced content, since §7.10's algorithm
restarts the whole rule list from the top on every change to a given node,
regardless of which rule caused the change.

## 7.16 `StructExtractStructPackFoldingRule` — cancelling a pack/extract round-trip

**Purpose, in one sentence:** when a query builds a struct with
`struct_pack(...)` and then immediately extracts one named field back out
with `struct_extract(...)` (or the `[...]`/`.field` subscript syntax, which
binds to the same `array_extract` overload for structs), skip constructing
the intermediate struct entirely and substitute the original field
expression directly.

**Construction-order position:** #3 (`optimizer.cpp:76`) — remarkably early,
immediately after `ConstantFoldingRule` (#2) and before `DistributivityRule`
(#4).

### Source

```cpp
// src/optimizer/rule/struct_extract_struct_pack_folding.cpp:10-21
StructExtractStructPackFoldingRule::StructExtractStructPackFoldingRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	auto func = make_uniq<FunctionExpressionMatcher>();
	// a bracket subscript on a struct binds to the "array_extract" overload - fold both the same way
	func->function = make_uniq<ManyFunctionMatcher>(identifier_set_t {"struct_extract", "array_extract"});
	func->policy = SetMatcher::Policy::ORDERED;
	auto packed = make_uniq<FunctionExpressionMatcher>();
	packed->function = make_uniq<SpecificFunctionMatcher>("struct_pack");
	packed->policy = SetMatcher::Policy::SOME;
	func->matchers.push_back(std::move(packed));
	func->matchers.push_back(make_uniq<ExpressionMatcher>());
	root = std::move(func);
}
```

The comment is worth noting as a small but genuinely useful piece of
DuckDB-internals trivia in its own right: DuckDB's `.field` and `struct['field']`
subscript syntax on a struct-typed expression doesn't bind to a distinct
"struct subscript" function at all — it binds to the same generic
`array_extract` scalar function structs and lists both use, and this rule's
matcher explicitly covers *both* function names so that
`struct_pack(a := 1, b := 2).a` folds exactly as readily as
`struct_extract(struct_pack(a := 1, b := 2), 'a')` does.

`Apply` is a direct field lookup, not a re-evaluation:

```cpp
// src/optimizer/rule/struct_extract_struct_pack_folding.cpp:23-43
unique_ptr<Expression> StructExtractStructPackFoldingRule::Apply(LogicalOperator &op,
                                                                 vector<reference<Expression>> &bindings,
                                                                 bool &changes_made, bool is_root) {
	auto &extract = bindings[0].get().Cast<BoundFunctionExpression>();
	auto &packed = bindings[1].get().Cast<BoundFunctionExpression>();
	auto &bind_info = extract.BindInfo();
	if (!bind_info) {
		return nullptr;
	}
	auto &extract_data = bind_info->Cast<StructExtractBindData>();
	auto &packed_children = packed.GetChildrenMutable();
	if (extract_data.index >= packed_children.size()) {
		return nullptr;
	}
	// struct_extract(struct_pack(e0, .., en), ki) is equivalent to ei
	auto result = std::move(packed_children[extract_data.index]);
	auto &return_type = extract.GetReturnType();
	// AddCastToType skips the cast when only the collation differs - keep the collation of the extract
	auto cast = BoundCastExpression::AddCastToType(GetContext(), std::move(result), return_type);
	return Expression::PreserveReturnType(return_type, std::move(cast));
}
```

`struct_extract`'s bind step (elsewhere, at bind time, not shown here) has
already resolved *which* field name the call is asking for down to a plain
integer `index` into the struct's field list, stashed in its
`StructExtractBindData` — so this rule doesn't need to do any string
matching against field names at optimization time at all; it just indexes
directly into `struct_pack`'s own child-expression list at that position
and lifts that child out, bypassing struct construction entirely. Two
defensive details: the rule bails out (`return nullptr`) if there's no
`bind_info` at all, or if the resolved `index` is out of range for the
*matched* `struct_pack` call specifically — the second guard matters
because `packed->policy = SetMatcher::Policy::SOME` means the matched
`struct_pack` might not be the *only* struct-shaped input feeding the outer
extract in every possible tree shape the matcher could technically accept,
so this index-bounds check is a real safety net rather than dead code.
Finally, the extracted child is wrapped in `BoundCastExpression::AddCastToType`
targeting the *original extract call's* own return type — not because
the field value's own type is wrong, but because `struct_extract`/`array_extract`'s
bind logic may have already settled on a specific output type (accounting
for, e.g., an enclosing cast on the whole struct, as the interaction below
shows) that doesn't necessarily match the packed field's own bound type
bit-for-bit, and the fold has to preserve whatever type the rest of the
plan above it already expects.

The last two lines deal with collations. A *collation* is a rule for
comparing strings, such as `nocase`, which ignores upper and lower case. It
is stored as part of the `VARCHAR` type. `AddCastToType` adds no cast when two
types differ only in their collation. So in `({'b': s}).b COLLATE nocase`, a
plain cast would return `s` and lose the `nocase` collation.
`Expression::PreserveReturnType` (`src/planner/expression.cpp:16-21`) stamps
the extract's own return type, collation included, onto the result. The type
comes from `extract.GetReturnType()`, the type of the bound expression
itself, not from the function's declared return type.

### Evidence: no dedicated `.test` file found, an illustrative example instead, plus one genuine correctness test

A systematic search of `test/optimizer/` and `test/sql/` for a same-label
`EXPLAIN` pair proving this specific fold (searching for `struct_extract`
paired with `struct_pack` inside `EXPLAIN`-based tests, and for any file
whose name suggests this rule directly) turned up nothing — this appears to
be one of the rules in the 37-rule list without dedicated, harness-checked
`EXPLAIN`-pair coverage in this checkout. (One test does name this rule:
`test/sql/optimizer/struct_extract_struct_pack_collation.test`.
It checks results, not plans. For example,
`SELECT s FROM c WHERE ({'b': s}).b COLLATE nocase = 'A'` must return the
row `a`, and the same query without `COLLATE nocase` must return no rows.
It proves the fold keeps the collation, not that the fold happens.) Two adjacent things did turn up,
worth distinguishing carefully from what this rule itself does:

`test/common/test_cast_struct.test` confirms the *correctness* of a
`struct_extract(struct_pack(...), 'a')` expression under a cast, but as a
`query`/`statement error` value check, not an `EXPLAIN`-pair plan proof —
it's evidence the expression *evaluates* right, not direct evidence this
specific rule is what made it cheap to evaluate:

```sql
query I
SELECT struct_extract(struct_pack(a => 42)::STRUCT(a STRING), 'a');
----
42
```

`test/sql/optimizer/pushdown_extract_reapplied.test` is worth naming
explicitly as a *false lead* rather than silently ignoring it: it also pairs
`struct_extract`-shaped subscripts with structs in its test data
(`{'id': k + 1000, 'ms': {'a': k + 200, ...}}`), but its subject is an
entirely different mechanism — column-pruning re-application after
`LATE_MATERIALIZATION` (pass #33) reruns `UNUSED_COLUMNS`/`COLUMN_LIFETIME`
a second time, pushing a residual `.field` extract down onto a base-table
scan a second time. It has nothing to do with folding a `struct_pack`
literal away, and is mentioned here only to head off the confusion that its
superficially similar SQL text might otherwise cause.

Given the absence of dedicated coverage, here is a clearly **illustrative**
example, constructed directly from the source above rather than lifted from
any `.test` file — following this section's own stated convention (§7.9)
for marking constructed rather than harness-checked evidence:

```sql
-- ILLUSTRATIVE, not from a .test file: constructed directly from
-- struct_extract_struct_pack_folding.cpp's Apply logic above.
EXPLAIN SELECT struct_extract(struct_pack(a := price, b := quantity), 'b') FROM orders;
-- optimizes to a plan equivalent to:
EXPLAIN SELECT quantity FROM orders;
```

Reasoning directly from `Apply`: `struct_extract`'s bind resolves `'b'` to
index `1` in the packed field list `[a, b]`; `Apply` looks up index `1` in
`struct_pack`'s own child-expression list, finds `quantity`, and returns
`CAST(quantity AS <b's resolved type>)` in its place — no `STRUCT` value is
ever constructed at execution time for this expression, even though the SQL
text explicitly asks for one to be packed and then immediately unpacked.

### Order and interactions

Position #3 — running right after `ConstantFoldingRule` (#2) and before
`DistributivityRule` (#4) and every arithmetic/comparison rule that
follows — has no comment in the source explaining the specific choice, so
what follows is this book's own reasoning, flagged as such rather than
presented as documented fact: `struct_pack`/`struct_extract` are ordinary
scalar function calls from every other rule's point of view, and nothing
in the other 34 rules' matchers specifically targets or special-cases
either function name. Running this fold early, right behind constant
folding, means that if a query builds and immediately unpacks a struct
around a genuinely foldable field value (`struct_extract(struct_pack(a := 1+1), 'a')`),
the unpacked `1+1` is exposed as a plain top-level expression early enough
in the visit sequence for `ConstantFoldingRule` and every later rule in the
list to treat it exactly as if the struct round-trip had never been
written at all — earlier is safer here specifically because it minimizes
the number of later rules that would otherwise need their own struct-aware
special case to see through an intervening pack/extract wrapper.

## 7.17 `LeastGreatestSimplificationRule` — collapsing a duplicate-argument call

**Purpose, in one sentence:** when a two-argument `LEAST`/`GREATEST` call
has structurally identical, non-volatile arguments on both sides, replace
the whole call with either argument alone — `LEAST(x, x)` and
`GREATEST(x, x)` both just mean `x`.

**Construction-order position:** #6 (`optimizer.cpp:79`), immediately after
`ArithmeticSimplificationRule` (#5) and before `CaseSimplificationRule` (#7).

### Source

```cpp
// src/optimizer/rule/least_greatest_simplification.cpp:8-15
LeastGreatestSimplificationRule::LeastGreatestSimplificationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
	auto function = make_uniq<FunctionExpressionMatcher>();
	function->function = make_uniq<ManyFunctionMatcher>(identifier_set_t {"least", "greatest"});
	function->matchers.push_back(make_uniq<ExpressionMatcher>());
	function->matchers.push_back(make_uniq<ExpressionMatcher>());
	function->policy = SetMatcher::Policy::ORDERED;
	root = std::move(function);
}

unique_ptr<Expression> LeastGreatestSimplificationRule::Apply(LogicalOperator &op,
                                                              vector<reference<Expression>> &bindings,
                                                              bool &changes_made, bool is_root) {
	auto &root = bindings[0].get().Cast<BoundFunctionExpression>();
	auto &children = root.GetChildrenMutable();
	D_ASSERT(children.size() == 2);
	if (children[0]->IsVolatile() || !children[0]->Equals(*children[1])) {
		return nullptr;
	}
	return Expression::PreserveReturnType(root.GetReturnType(), std::move(children[0]));
}
```

The matcher's `Policy::ORDERED` with exactly two `ExpressionMatcher` entries
means this rule specifically targets the **binary** form of `LEAST`/`GREATEST`
— `D_ASSERT(children.size() == 2)` in `Apply` confirms the assumption is
load-bearing, not incidental. The guard in `Apply` is two conditions,
both necessary: `children[0]->Equals(*children[1])` checks *structural*
equality (the same kind `DistributivityRule`'s `expression_set_t` uses,
§7.15) — the two arguments have to be the literal same expression tree, not
merely provably-equal-at-runtime values — and `!children[0]->IsVolatile()`
guards against the one case where two textually identical calls are *not*
interchangeable: if the shared argument called something like `random()`,
`LEAST(random(), random())` (two *separate* textual invocations that happen
to structurally parse the same way, or literally the same sub-expression
object referenced twice) would evaluate the volatile function twice at
runtime, and there is no guarantee those two evaluations produce the same
value — collapsing the call to a single evaluation would silently change
the query's behavior, so the rule declines whenever the shared child is
volatile.

### What the found tests actually cover — and what they don't

A search for `LEAST`/`GREATEST` test coverage in `test/optimizer/` turned up
`test/optimizer/statistics/statistics_least_greatest.test`, named directly
in this chapter's outline as a research lead — and it is real, substantial
test coverage of DuckDB's `LEAST`/`GREATEST` handling, but it is worth being
precise about *what* it tests, because it is not this rule:

```sql
query II
EXPLAIN SELECT count(*) FROM integers WHERE greatest(id, 5) < 3;
----
physical_plan	<REGEX>:.*Empty Result.*
```

This is `STATISTICS_PROPAGATION` (pass #34, §7.6.2 of the overview) at
work, not `LeastGreatestSimplificationRule`: it's confirming that
`StatisticsPropagator` knows how to derive a value range for a
`GREATEST(id, 5)` expression from `id`'s own propagated statistics (the
result of `GREATEST` is bounded below by the larger of `id`'s minimum and
the constant `5`) and use that derived range to prove the whole filter
`< 3` can never be satisfied, collapsing the query to an `Empty Result`
plan. `greatest(id, 5)` has two structurally *different* arguments (a
column and a constant) — `LeastGreatestSimplificationRule`'s matcher
requires the two children to satisfy `Equals`, which `id` and `5` plainly
don't, so this rule never fires anywhere in this test file. This is worth
stating as directly as the overview's own "sum_rewriter" correction
(§7.6.5 there): **this specific `.test` file is real, valuable coverage of
`LEAST`/`GREATEST`-aware statistics propagation, and is not coverage of
`LeastGreatestSimplificationRule`.**

A second, more suggestive lead — `test/sql/optimizer/cross_column_expression_stats_pruning.test`,
which contains the literal duplicate-argument shape this rule actually
targets:

```sql
# A statistics callback can decline to produce statistics at runtime
query II
EXPLAIN ANALYZE SELECT count(*) FROM expression_t WHERE greatest(s, s) > s2;
----
analyzed_plan	<REGEX>:.*Row Groups Scanned: 3 / 3.*
```

`greatest(s, s)` is exactly the shape `LeastGreatestSimplificationRule` is
built to collapse — both arguments are the same column, `s`, and (assuming
a plain column reference is never volatile, which it isn't) this rule
should reduce it to bare `s` before the plan ever reaches
`STATISTICS_PROPAGATION` or `ROW_GROUP_PRUNER`. But this test's own comment
signals its subject is something else — a runtime statistics callback that
*declines* to produce usable statistics, checked by confirming all three
row groups were scanned (i.e., pruning did *not* happen) — and without
independently re-running this query against this exact checkout and
inspecting the optimized plan directly (this book's stated policy is not to
claim more certainty than was actually checked), this file states plainly:
this test is **suggestive, indirect corroboration** that
`LeastGreatestSimplificationRule` fires here (the query it's testing reads
naturally as "`s > s2`, once `greatest(s,s)` has already become bare `s`"),
but it was not built to prove that specific rewrite the way a same-label
`EXPLAIN` pair would, and this book declines to claim more than that.

Given the gap, here is a small, clearly **illustrative** `EXPLAIN`-pair
example, constructed directly from the `Apply` method's own logic rather
than adapted from either test above:

```sql
-- ILLUSTRATIVE, not from a .test file: constructed directly from
-- least_greatest_simplification.cpp's Apply logic above.
EXPLAIN SELECT GREATEST(price, price) FROM orders;
-- optimizes to a plan equivalent to:
EXPLAIN SELECT price FROM orders;

-- but this identity does NOT apply here, since the shared argument is volatile:
EXPLAIN SELECT LEAST(random(), random());
-- stays exactly as written -- two separate calls to random() are not
-- guaranteed to agree, so IsVolatile() blocks the fold.
```

### Order and interactions

Position #6, directly after `ArithmeticSimplificationRule` (#5) and before
`CaseSimplificationRule` (#7), places this rule squarely inside the same
early cluster of "cheap, purely-structural, no-cost-model-needed"
simplifications this file has covered throughout. Nothing about this rule's
matcher or `Apply` depends on anything `ArithmeticSimplificationRule` might
have already done to either child — `Equals` is a structural comparison
over whatever tree shape each child currently has, so if
`ArithmeticSimplificationRule` had, on an earlier visit to a sibling node,
already reduced two originally-different-looking arguments down to
*identical* simplified forms (an admittedly contrived scenario — e.g.
`LEAST(x + 0, x)` becoming `LEAST(x, x)` after the `+0` identity fires on
the first argument during the same bottom-up visit that later reaches this
`LEAST` call), the reset-and-redo traversal from §7.10 would still hand
`LeastGreatestSimplificationRule` a fair shot at the now-identical pair on
a subsequent pass over the same node, exactly the same mechanism that let
`ConstantOrderNormalizationRule` and `ConstantFoldingRule` cooperate in
§7.12 despite being two independently-authored, independently-triggered
rules.

## 7.18 What this file established, and what's next

Seven rules, all from the front of the 37-rule `EXPRESSION_REWRITER` list
(construction-list positions #1 through #6, plus #17/#18), turn out to
divide fairly cleanly into three families once their actual source is read
rather than assumed from name or one-line gloss: pure evaluation
(`ConstantFoldingRule`, which delegates entirely to the real expression
executor and knows nothing about what any specific function or operator
means); associativity/commutativity-driven tree reshaping in service of
that evaluator (`ConstantOrderNormalizationRule`, whose real job —
regrouping scattered constants, not canonicalizing comparison sides — this
file corrected against a more casual first-guess description; and
`MoveConstantsRule`/`MoveUnaryMinusRule`, which rearrange a comparison to
isolate a bare column reference specifically so that later, structurally-
strict consumers like `StatisticsPropagator::UpdateFilterStatistics` — shown
directly, source-cited, not merely asserted — can recognize and narrow it);
and pure Boolean/algebraic identity recognition with no evaluation involved
at all (`ArithmeticSimplificationRule`'s identity-element catalog,
`DistributivityRule`'s common-term factoring confirmed in the
factor-out-of-a-disjunction direction rather than assumed, and the two
narrower structural cancellations, `StructExtractStructPackFoldingRule` and
`LeastGreatestSimplificationRule`, both of which turned out to lack
dedicated same-label test coverage in this checkout — a gap this file
stated plainly rather than papering over with a test that tests something
adjacent instead).

The shared machinery §7.10 established — the `Rule`/`ExpressionMatcher`
framework, `SetMatcher::Policy`'s four matching disciplines, and, most
importantly, the bottom-up, reset-and-redo traversal in `ApplyRules` that
lets one node get revisited by the full 37-rule list as many times as it
keeps changing shape — is exactly what the remaining 28 rules in this list
build on too. The next file in this directory picks up where the
36-rule ordered list resumes: `CaseSimplificationRule`,
`ConjunctionSimplificationRule`, `DatePartSimplificationRule`,
`DateTruncSimplificationRule`, `ComparisonSimplificationRule`,
`RowComparisonSimplificationRule`, and the `IN`/`ENUM`-shaped simplification
rules that immediately follow `EqualOrNullSimplification` at construction
position #16 — each gets the same treatment this file gave its seven: real
source, `file:line` citations, and every test case in its dedicated
`.test` file walked through as a same-label proof rather than summarized
from a distance.
