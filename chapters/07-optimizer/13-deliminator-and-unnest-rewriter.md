# 7.13 — `DELIMINATOR` and `UNNEST_REWRITER`: cleaning up after decorrelation

## 13.1 Scope, and what this section deliberately does not re-explain

Chapter 6 covered `FlattenDependentJoins` — the planner-level algorithm, running
in `Planner::CreatePlan` right after binding and before `Optimizer::Optimize`
is ever called, that turns a
correlated subquery's `LogicalDependentJoin` into an ordinary join tree. That
chapter also gave a first pass over the two optimizer passes this section goes
deep on: §6.6 introduced `Deliminator::RemoveJoinWithDelimGet` and its
`IS NOT NULL` compensation filter as one of two independent implementations of
the "decoupling" optimization from Neumann and Kemper's paper
[12], and §6.7 sketched `UnnestRewriter`'s
job as removing the `LOGICAL_DELIM_JOIN`/`LogicalWindow` scaffolding that
`FlattenDependentJoins` leaves behind for a correlated subquery whose body is
`UNNEST`.

This section does not re-derive the D ⋈ᵈ Q algebra, does not re-explain what
"DELIM" stands for, and does not re-walk `FlattenDependentJoins`'s three
pushdown rules — Chapter 6, §§6.2–6.5 and §6.9, already did that with full
rigor, and nothing here supersedes it. What follows is instead a much deeper
mechanical account of the two **optimizer-level, post-decorrelation cleanup**
passes themselves: the exact pattern each one matches, the exact conditions
under which each rewrite is safe, every test case in the two primary test
files, and — for each pass — a real regression-test case study showing a
specific way the pass once got the correctness boundary wrong.

## 13.2 Pipeline position, resolved

Chapter 6 flagged, and explicitly deferred, a numbering discrepancy between two
ways of counting pipeline position. §7.4 of the overview chapter is the
authoritative list, and its numbering is unambiguous: **`DELIMINATOR` is pass
#10** (`src/optimizer/optimizer.cpp:316`, `src/optimizer/deliminator.cpp`) and
**`UNNEST_REWRITER` is pass #22** (`src/optimizer/optimizer.cpp:395`,
`src/optimizer/unnest_rewriter.cpp`), out of the ~45 passes `RunBuiltInOptimizers`
invokes. Both call sites carry their own one-line inline comments, which are
worth quoting verbatim because they are the most compact correct description
of each pass's job that exists anywhere in the source:

```cpp
// src/optimizer/optimizer.cpp:315-319
// removes any redundant DelimGets/DelimJoins
RunOptimizer(OptimizerType::DELIMINATOR, [&]() {
	Deliminator deliminator(context);
	plan = deliminator.Optimize(std::move(plan));
});
```

```cpp
// src/optimizer/optimizer.cpp:394-398
// rewrites UNNESTs in DelimJoins by moving them to the projection
RunOptimizer(OptimizerType::UNNEST_REWRITER, [&]() {
	UnnestRewriter unnest_rewriter(context);
	plan = unnest_rewriter.Optimize(std::move(plan));
});
```

Eleven passes separate them (`GROUPING_SETS`, a second `CTE_INLINING`,
`EMPTY_RESULT_PULLUP`, `WINDOW_SELF_JOIN`, `PROJECTION_PULLUP`,
`OUTER_JOIN_SIMPLIFICATION`, `JOIN_ORDER`, `AGGREGATE_REUSE`,
`PARTIAL_AGGREGATE_PUSHDOWN`, `JOIN_ELIMINATION`, `REDUNDANT_ORDER_KEYS` — see
§7.4's numbered list for what each of those does), none of which touch
`LOGICAL_DELIM_JOIN` machinery directly.

`UnnestRewriter` takes the `ClientContext`. It needs the context to add an
`ARRAY`-to-`LIST` cast (§13.6.5).

`Deliminator` runs early, right after `IN_CLAUSE`'s mark-join
rewrite and before join ordering ever sees the tree; `UnnestRewriter` runs
considerably later, after join ordering has already had its say. Both are
fixed, one-shot, no-fixpoint passes exactly in the sense §7.2 describes for
the pipeline as a whole: each gets exactly one traversal of the tree it
inherits, and if a later pass exposes a new opportunity for either of them,
neither gets a second turn in this run of the query.

## 13.3 `DELIMINATOR`, in full

**Purpose in one sentence, building on Chapter 6's framing:** where
`FlattenDependentJoins` guarantees the tree is *executable* (no
`LOGICAL_DEPENDENT_JOIN` survives), `Deliminator` asks a narrower, purely
local question about the executable tree it's handed — does *this specific*
join, against *this specific* `DelimGet`, actually need to exist, or does the
surrounding tree already carry the same information some other way — and
either deletes the join (compensating for what it silently relied on), or
demotes the DelimJoin's uniqueness guarantee when it turns out to be stronger
than necessary.

**When this pass actually sees a DelimJoin.** With default settings, it does
not. The setting `delim_join_as_cte` defaults to `true`, so the planner turns
every DelimJoin into plain joins over materialized CTEs right after
flattening, before the optimizer starts (§13.3.6, Chapter 6 §6.4.1). The same
clean-up then happens inside that CTE step. Everything in §13.3.1–§13.3.5 and
§13.5 describes the DelimJoin form, which you get with
`SET delim_join_as_cte = false`. The ideas carry over to the CTE form one for
one.

> **Note:** DuckDB switched `delim_join_as_cte` on by default in June 2026
> (commit `a7b37725088`, "Flip the delim_join_as_cte switch"). The setting is
> also marked deprecated.

### 13.3.1 The two data structures `Deliminator::Optimize` builds

`Deliminator::Optimize` (`src/optimizer/deliminator.cpp:56-101`) is short —
sixteen lines of actual control flow — precisely because almost everything it
does is delegated to two small structs built by a single tree walk:

```cpp
// src/optimizer/deliminator.cpp:23-41
struct JoinWithDelimGet {
	JoinWithDelimGet(unique_ptr<LogicalOperator> &join_p, idx_t depth_p) : join(join_p), depth(depth_p) {
	}
	reference<unique_ptr<LogicalOperator>> join;
	idx_t depth;
};

struct DelimCandidate {
public:
	explicit DelimCandidate(unique_ptr<LogicalOperator> &op, LogicalComparisonJoin &delim_join)
	    : op(op), delim_join(delim_join), delim_get_count(0) {
	}
public:
	unique_ptr<LogicalOperator> &op;
	LogicalComparisonJoin &delim_join;
	vector<JoinWithDelimGet> joins;
	idx_t delim_get_count;
};
```

A `DelimCandidate` is built once per `LOGICAL_DELIM_JOIN` found anywhere in the
tree (`FindCandidates`, `deliminator.cpp:103-117`, a plain bottom-up recursion
over `op->children` that appends a candidate whenever `op->type ==
LOGICAL_DELIM_JOIN`). Its `joins` vector is then populated by
`FindJoinWithDelimGet` (`deliminator.cpp:160-175`), which recurses into the
DelimJoin's **RHS only** (`op->children[1]`, the subquery side —
`FindCandidates` calls it as `FindJoinWithDelimGet(op->children[1], candidate)`
at line 116) and records two things as it goes:

```cpp
// src/optimizer/deliminator.cpp:160-175
void Deliminator::FindJoinWithDelimGet(unique_ptr<LogicalOperator> &op, DelimCandidate &candidate, idx_t depth) {
	if (op->type == LogicalOperatorType::LOGICAL_DELIM_JOIN) {
		FindJoinWithDelimGet(op->children[0], candidate, depth + 1);
	} else if (op->type == LogicalOperatorType::LOGICAL_DELIM_GET) {
		candidate.delim_get_count++;
	} else {
		for (auto &child : op->children) {
			FindJoinWithDelimGet(child, candidate, depth + 1);
		}
	}
	if (op->type == LogicalOperatorType::LOGICAL_COMPARISON_JOIN &&
	    (OperatorIsDelimGet(*op->children[0]) || OperatorIsDelimGet(*op->children[1]))) {
		candidate.joins.emplace_back(op, depth);
	}
}
```

Two things worth being exact about here, because they are the whole shape of
the pattern this pass recognizes:

1. **`delim_get_count`** is simply *how many bare `LogicalDelimGet` leaves
   exist* anywhere under the RHS — regardless of whether each one is wrapped
   in a join. A `LogicalDelimGet` reached through a nested `LOGICAL_DELIM_JOIN`
   is also counted (the first branch recurses one level deeper, tracking
   `depth`, for exactly this nested-decorrelation case), which matters later
   for the "is everything gone" check.
2. **`candidate.joins`** is populated only for a `LOGICAL_COMPARISON_JOIN`
   (an *ordinary*, already-untagged join — never a nested `LOGICAL_DELIM_JOIN`
   itself) with a `LogicalDelimGet` directly on one side, optionally wrapped in
   exactly one `LogicalFilter` (`OperatorIsDelimGet`,
   `deliminator.cpp:149-158`, checks both the bare and filter-wrapped shape).
   This is precisely the "join-with-DelimGet" pattern: somewhere inside the
   already-pushed-down subquery body, a correlated base table gets joined,
   on equality or inequality, against a leaf that simply re-exposes the
   outer domain's duplicate-eliminated columns. This is exactly the shape
   `FlattenDependentJoins`'s filter/aggregate pushdown rules from Chapter 6
   produce whenever the correlation predicate resolves against a base scan
   that isn't the DelimJoin's own immediate RHS child — the general case,
   not a special one. §13.5 below walks a real, test-verified example of
   this shape end to end.

### 13.3.2 `Deliminator::Optimize`'s per-candidate decision sequence

```cpp
// src/optimizer/deliminator.cpp:65-98
for (auto &candidate : candidates) {
	auto &delim_join = candidate.delim_join;

	// Sort these so the deepest are first
	std::sort(candidate.joins.begin(), candidate.joins.end(),
	          [](const JoinWithDelimGet &lhs, const JoinWithDelimGet &rhs) { return lhs.depth > rhs.depth; });

	bool all_removed = true;
	if (!candidate.joins.empty() && HasSelection(delim_join)) {
		// Keep the deepest join with DelimGet in these cases,
		// as the selection can greatly reduce the cost of the RHS child of the DelimJoin
		candidate.joins.erase(candidate.joins.begin());
		all_removed = false;
	}

	bool all_equality_conditions = true;
	for (auto &join : candidate.joins) {
		all_removed = RemoveJoinWithDelimGet(delim_join, candidate.delim_get_count, join.join.get(),
		                                     all_equality_conditions) &&
		              all_removed;
	}

	// Change type if there are no more duplicate-eliminated columns
	if (candidate.joins.size() == candidate.delim_get_count && all_removed) {
		delim_join.type = LogicalOperatorType::LOGICAL_COMPARISON_JOIN;
		delim_join.duplicate_eliminated_columns.clear();
	}

	// Only DelimJoins are ever created as SINGLE joins,
	// and we can switch from SINGLE to LEFT if the RHS is de-duplicated by an aggr
	if (delim_join.join_type == JoinType::SINGLE) {
		TrySwitchSingleToLeft(delim_join);
	}
}
```

Four separate decisions live in these 34 lines, in order:

**(1) Sort deepest-first, then possibly spare the deepest.** `joins` is
sorted so the join *farthest from the DelimJoin itself* is processed first.
Then `HasSelection(delim_join)` (`deliminator.cpp:119-147`) walks the *entire*
DelimJoin subtree — both its LHS (the domain-producing side) and its RHS —
looking for any `LogicalFilter`, or any `LogicalGet` carrying a table filter
that isn't a plain `IS NOT NULL`, anywhere at all:

```cpp
// src/optimizer/deliminator.cpp:119-147
bool Deliminator::HasSelection(const LogicalOperator &op) {
	// TODO once we implement selectivity estimation using samples we need to use that here
	switch (op.type) {
	case LogicalOperatorType::LOGICAL_GET: {
		auto &get = op.Cast<LogicalGet>();
		for (const auto &entry : get.table_filters) {
			auto &expr_filter = ExpressionFilter::GetExpressionFilter(entry.Filter(), "Deliminator::HasSelection");
			auto &expr = *expr_filter.expr;
			if (expr.GetExpressionClass() != ExpressionClass::BOUND_OPERATOR ||
			    expr.GetExpressionType() != ExpressionType::OPERATOR_IS_NOT_NULL) {
				return true;
			}
		}
		break;
	}
	case LogicalOperatorType::LOGICAL_FILTER:
		return true;
	default:
		break;
	}

	for (auto &child : op.children) {
		if (HasSelection(*child)) {
			return true;
		}
	}

	return false;
}
```

If it finds one, the code **removes the deepest join from the to-be-processed
list** (`candidate.joins.erase(candidate.joins.begin())`) and sets
`all_removed = false` up front — meaning that join with its DelimGet is kept,
untouched, for the rest of this pass's run. The comment states the reasoning
plainly: *"the selection can greatly reduce the cost of the RHS child of the
DelimJoin."* This is a cost heuristic, not a correctness rule — the deepest
join-with-DelimGet is the one physically closest to the base table the
selective filter narrows, and removing it would mean re-deriving the domain
restriction some other way (or losing it), when keeping the join lets the
selective filter continue to prune the correlated subquery's own input before
the physical `PhysicalHashAggregate`/DISTINCT step even runs. This is the
mechanism behind the TPC-H Q17 filtered-variant case walked in §13.5.

**(2) Try to remove every remaining join-with-DelimGet.** For every entry
still in `candidate.joins` (after the possible one-item removal above),
`RemoveJoinWithDelimGet` is attempted (§13.3.3). It returns `false` on its own
whenever the join isn't actually redundant or doesn't match the shape it can
handle — `all_removed` accumulates the conjunction of every attempt, so a
*single* join that can't be removed makes `all_removed` false for the whole
candidate.

**(3) Downgrade the tag once nothing is left that needs it.** `candidate.joins.size()
== candidate.delim_get_count && all_removed` is the exact condition for
"every `LogicalDelimGet` this candidate found has now been eliminated" — note
this is a *count* comparison, not a structural re-scan; it works because
`RemoveJoinWithDelimGet` only ever succeeds against a join that owns exactly
one `LogicalDelimGet` (checked inside it, §13.3.3), so removing every entry in
`candidate.joins` removes exactly `candidate.joins.size()` DelimGets, and if
that equals the total `delim_get_count` found earlier, none survive anywhere
in the subtree. When this holds, the `LOGICAL_DELIM_JOIN` tag is stripped back
to a plain `LOGICAL_COMPARISON_JOIN` and `duplicate_eliminated_columns` is
cleared — the join no longer needs the physical plan generator to build a
duplicate-eliminating `PhysicalHashAggregate` at all (§6.4's physical-level
"decoupling" mechanism, `plan_delim_join.cpp:36-41`, would in fact have caught
this too, but `Deliminator` gets there first, at the logical level, which
matters for every other pass between here and physical plan generation that
inspects `duplicate_eliminated_columns` or the `LOGICAL_DELIM_JOIN` tag —
Chapter 8's join-order optimizer being the most consequential downstream
reader, per §6.10's discussion of `RelationManager::ExtractJoinRelation`).

**(4) Try to weaken `SINGLE` to `LEFT`.** Independently of whether any
DelimGet was removed, if `delim_join.join_type == JoinType::SINGLE`,
`TrySwitchSingleToLeft` is always attempted (§13.3.5).

### 13.3.3 `RemoveJoinWithDelimGet` — the exact removal condition

This is the function §6.6 previewed. Read in full, its structure is: figure
out which side of the join is the DelimGet, collect (and copy out) any filter
sitting directly on that side, verify the join is actually redundant, then —
only if it's redundant — perform the splice.

```cpp
// src/optimizer/deliminator.cpp:187-259
bool Deliminator::RemoveJoinWithDelimGet(LogicalComparisonJoin &delim_join, const idx_t delim_get_count,
                                         unique_ptr<LogicalOperator> &join, bool &all_equality_conditions) {
	auto &comparison_join = join->Cast<LogicalComparisonJoin>();
	if (!ChildJoinTypeCanBeDeliminated(comparison_join.join_type)) {
		return false;
	}

	// Get the index (left or right) of the DelimGet side of the join
	const idx_t delim_idx = OperatorIsDelimGet(*join->children[0]) ? 0 : 1;

	// Get the filter (if any)
	optional_ptr<LogicalFilter> filter;
	vector<unique_ptr<Expression>> filter_expressions;
	if (join->children[delim_idx]->type == LogicalOperatorType::LOGICAL_FILTER) {
		filter = &join->children[delim_idx]->Cast<LogicalFilter>();
		for (auto &expr : filter->expressions) {
			filter_expressions.emplace_back(expr->Copy());
		}
	}

	auto &delim_get = (filter ? filter->children[0] : join->children[delim_idx])->Cast<LogicalDelimGet>();
	if (comparison_join.conditions.size() != delim_get.chunk_types.size()) {
		return false; // Joining with DelimGet adds new information
	}

	// Check if joining with the DelimGet is redundant, and collect relevant column information
	ColumnBindingReplacer replacer;
	auto &replacement_bindings = replacer.replacement_bindings;
	for (auto &cond : comparison_join.conditions) {
		if (!cond.IsComparison()) {
			return false;
		}
		all_equality_conditions = all_equality_conditions && IsEqualityJoinCondition(cond);
		auto &delim_side = delim_idx == 0 ? cond.GetLHS() : cond.GetRHS();
		auto &other_side = delim_idx == 0 ? cond.GetRHS() : cond.GetLHS();
		if (delim_side.GetExpressionType() != ExpressionType::BOUND_COLUMN_REF ||
		    other_side.GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
			return false;
		}
		auto &delim_colref = delim_side.Cast<BoundColumnRefExpression>();
		auto &other_colref = other_side.Cast<BoundColumnRefExpression>();
		replacement_bindings.emplace_back(delim_colref.Binding(), other_colref.Binding());

		// Only add IS NOT NULL filter for regular equality/inequality comparisons
		// Do NOT add for DISTINCT FROM variants, as they handle NULL correctly
		if (cond.GetComparisonType() != ExpressionType::COMPARE_NOT_DISTINCT_FROM &&
		    cond.GetComparisonType() != ExpressionType::COMPARE_DISTINCT_FROM) {
			auto is_not_null_expr =
			    make_uniq<BoundOperatorExpression>(ExpressionType::OPERATOR_IS_NOT_NULL, LogicalType::BOOLEAN);
			is_not_null_expr->GetChildrenMutable().push_back(other_side.Copy());
			filter_expressions.push_back(std::move(is_not_null_expr));
		}
	}

	if (!all_equality_conditions &&
	    !RemoveInequalityJoinWithDelimGet(delim_join, delim_get_count, join, replacement_bindings)) {
		return false;
	}

	unique_ptr<LogicalOperator> replacement_op = std::move(comparison_join.children[1 - delim_idx]);
	if (!filter_expressions.empty()) { // Create filter if necessary
		auto new_filter = make_uniq<LogicalFilter>();
		new_filter->expressions = std::move(filter_expressions);
		new_filter->children.emplace_back(std::move(replacement_op));
		replacement_op = std::move(new_filter);
	}

	join = std::move(replacement_op);

	// TODO: Maybe go from delim join instead to save work
	replacer.VisitOperator(*root);
	return true;
}
```

Every guard here is a distinct correctness requirement, and it's worth being
precise about exactly which ones there are, in order:

- **`ChildJoinTypeCanBeDeliminated`** (`deliminator.cpp:177-185`) restricts
  this whole function to `JoinType::INNER` or `JoinType::SEMI` joins only —
  every other join type (`LEFT`, `MARK`, `ANTI`, `SINGLE`, `RIGHT_SEMI`, …)
  is handled, if at all, only through the separate inequality path
  (§13.3.4) with its own, narrower set of eligible types.
- **`comparison_join.conditions.size() != delim_get.chunk_types.size()`**
  is the single most important structural check in the whole function: it
  requires the join to compare *every single column* the DelimGet exposes,
  one condition per column, no more and no fewer. The comment is exact:
  *"Joining with DelimGet adds new information"* if this doesn't hold — a
  DelimGet exposing three duplicate-eliminated columns but only two of them
  actually compared in the join condition means the join is *filtering* on
  the third column's implicit relationship to something else, information
  that would be lost by deleting the join outright.
- **Every condition must be a plain `BOUND_COLUMN_REF` on both sides.** A
  join-with-DelimGet whose condition compares an *expression* of the DelimGet
  column (a cast, a function call) rather than the bare column itself is left
  alone — the function has no way to "undo" an expression when splicing the
  DelimGet's producer back in.
- **The redundancy claim itself is never separately proven — it's a
  structural invariant of what `FlattenDependentJoins` already guarantees.**
  Nothing in this function checks a foreign key or a uniqueness constraint;
  the whole reason this join is provably redundant is that a `LogicalDelimGet`
  by construction re-exposes exactly the duplicate-eliminated correlation
  columns of the DelimJoin's own domain side — the join comparing them
  against `other_side` is therefore *always* an equality (or, in the general
  inequality case, a comparison) between something the DelimJoin's domain
  already carries and something the correlated base table already carries,
  and the domain's own presence at the top of the tree makes the *filtering
  effect* of that comparison redundant once the values are threaded through
  directly — the "decoupling" idea Chapter 6 named. What this function does
  is not *prove* the optimization is applicable in some cost-based sense; it
  recognizes the syntactic shape that `FlattenDependentJoins` is known to
  produce whenever the optimization *is* applicable, and performs the splice.

**The `IS NOT NULL` compensation filter — exactly when, and exactly why.**
This is the mechanism §6.6 flagged for deeper treatment. The condition for
adding it is a single `if`, evaluated once **per join condition** (not once
per join): add `IS NOT NULL(other_side)` unless the condition's comparison
type is `COMPARE_NOT_DISTINCT_FROM` or `COMPARE_DISTINCT_FROM`. The reasoning
is exactly about what an **equi-join throws away implicitly** versus what a
raw column reference does not:

An ordinary `INNER`/`SEMI` join with condition `delim_side = other_side` (an
ordinary `COMPARE_EQUAL`, or its `NOTEQUAL`-family cousins under the
inequality path) only ever produces an output row when *both* sides are
non-NULL — SQL's three-valued comparison logic means `NULL = anything` is
`NULL`, which a join treats as "does not match," silently dropping any row of
the correlated base table where `other_side` is NULL. Once the join is
deleted and `other_side` (previously reached only by successfully passing
through that filtering join) becomes directly reachable, **that implicit
NULL-filtering effect must be reproduced explicitly**, or a row that used to
be dropped (because its `other_side` value was NULL, so it could never equal
anything, DelimGet-supplied or otherwise) would now survive. Hence: one
`OPERATOR_IS_NOT_NULL(other_side)` filter expression per removed equality
condition, folded into the same `LogicalFilter` the removed join's own
`filter` (if any) already contributed expressions to.

**Why `DISTINCT FROM`/`NOT DISTINCT FROM` are exempted.** These comparison
types are, by definition, exactly the null-safe variants of equality/
inequality — `NULL IS NOT DISTINCT FROM NULL` evaluates to `TRUE`, not `NULL`,
so a join on this condition type does **not** implicitly drop NULL-valued
rows the way `COMPARE_EQUAL` does. There is nothing to compensate for,
because there was never any implicit NULL-filtering effect to begin with —
adding an `IS NOT NULL` filter here would be an outright correctness bug (it
would incorrectly discard rows the original, null-safe join was specifically
designed to keep). The comment in the source states this directly: *"Only add
IS NOT NULL filter for regular equality/inequality comparisons. Do NOT add for
DISTINCT FROM variants, as they handle NULL correctly."*

**Where the compensation filter goes.** It is appended to `filter_expressions`
— the same vector that may already hold copies of an existing
`LogicalFilter`'s own expressions from directly atop the DelimGet — and if
that combined vector is non-empty, a *new* `LogicalFilter` is built wrapping
`replacement_op` (the surviving, non-DelimGet side of the removed join).
`join` — the `unique_ptr` slot the removed join used to occupy in the tree —
is then reassigned to this filter (or, if there was nothing to compensate for
at all, directly to `replacement_op` with no filter wrapper). Finally,
`replacer.VisitOperator(*root)` propagates every `(delim_colref.Binding() →
other_colref.Binding())` pair recorded during the condition walk across the
**entire plan**, not just the local subtree — every reference anywhere in the
tree to a column the deleted DelimGet used to expose (most commonly, the
correlated aggregate's own `GROUP BY` expression, per §13.5's worked example)
is rewritten to point at `other_side`'s binding instead.

### 13.3.4 `RemoveInequalityJoinWithDelimGet` — the wider join-type path

Equality conditions are handled entirely by the logic above. When
`all_equality_conditions` is false — some condition in the join uses
`<`, `<=`, `<>`, or a `DISTINCT FROM` variant — a second, considerably more
involved function is consulted before the removal is allowed to proceed at
all:

```cpp
// src/optimizer/deliminator.cpp:261-264
static bool InequalityDelimJoinCanBeEliminated(JoinType &join_type) {
	return join_type == JoinType::ANTI || join_type == JoinType::MARK || join_type == JoinType::SEMI ||
	       join_type == JoinType::SINGLE;
}
```

`RemoveInequalityJoinWithDelimGet` (`deliminator.cpp:287-424`) requires: the
DelimJoin has **exactly one** `LogicalDelimGet` total (`delim_get_count == 1`
— multi-column inequality domains are not handled at all), the DelimJoin's
own `join_type` is one of the four listed above, and the number of
conditions on the *outer* DelimJoin matches the number of conditions on the
*inner* join-with-DelimGet being considered for removal. For `MARK`/`SINGLE`
DelimJoins specifically, it additionally insists at least one of the inner
join's conditions is a plain equality (`has_one_equality`) — the comment
explains why: *"our pure inequality joins don't implement `JoinType::SINGLE`
yet, and `JoinType::MARK` is a special case"* — i.e., this is a physical
execution-operator limitation, not a semantic one: DuckDB's physical
single-side and mark-join operators are only implemented for the mixed
equality-plus-inequality shape, not pure inequality, so the rewrite declines
outright rather than produce a join shape nothing downstream can execute.

Its core job, once those gates pass, is to **rewrite the outer DelimJoin's
own conditions** to reference the correlated base table's column directly
instead of the DelimGet's column — tracing bindings down through any chain of
`LogicalProjection`/`LogicalFilter` operators between the DelimJoin's RHS and
the inner join (`FindAndReplaceBindings`, `deliminator.cpp:266-285`, bailing
out entirely, returning `false`, the moment it meets anything other than a
plain column-reference projection or a filter) and then, per condition,
**flipping the comparison operator** to account for which side of the
original condition the traced binding actually was on
(`FlipComparisonExpression`), while separately reconciling `DISTINCT FROM`/
`NOT DISTINCT FROM` semantics against plain `NOTEQUAL`/`EQUAL` — because the
DelimJoin's own condition was written against the *domain*'s column (always
implicitly null-rejecting for a `SINGLE`/non-`MARK` join, per how
`FlattenDependentJoins` builds it), while the inner join's condition was
written against the *DelimGet*'s re-exposed column, and the two null-handling
conventions do not automatically agree — the code comment even links a
separate historical bug directly: *"join condition was a not equal and
filtered out all NULLs. DELIM JOIN need to do that for not DELIM_GET side.
Easiest way is to change the comparison expression type. See
duckdb/duckdb#16803."*

The one gate that matters most for correctness, though, is the one §13.4
below is entirely devoted to, because it is exactly what a whole regression
test file exists to pin down: for a `MARK` join whose original comparison was
`COMPARE_NOTEQUAL`, the rewrite is refused outright unless a
`NotNullExpressionAnalyzer` can *prove*, by actually walking the plan (not by
trusting a declared schema constraint in isolation), that the relevant
expression cannot be NULL at that point in the tree:

```cpp
// src/optimizer/deliminator.cpp:370-375
if (delim_join.join_type == JoinType::MARK &&
    original_join_comparison == ExpressionType::COMPARE_NOTEQUAL) {
	if (!nullability.IsNotNull(*delim_join.children[0], delim_condition.GetLHS())) {
		return false;
	}
}
```

### 13.3.5 `TrySwitchSingleToLeft` — downgrading a uniqueness guarantee

Recall from Chapter 6 that DuckDB only ever *constructs* a DelimJoin as
`JoinType::SINGLE` (used for scalar, at-most-one-row-per-outer-row correlated
subqueries). `TrySwitchSingleToLeft` (`deliminator.cpp:426-479`) checks
whether that guarantee, which the physical `SINGLE`-join operator has to pay
for at execution time, is actually *needed*:

```cpp
// src/optimizer/deliminator.cpp:426-479 (structure, elided)
void Deliminator::TrySwitchSingleToLeft(LogicalComparisonJoin &delim_join) {
	// collect the RHS binding of every equality condition
	// walk delim_join.children[1] down through PROJECTION/FILTER only
	// until it finds a LOGICAL_AGGREGATE_AND_GROUP_BY
	// if aggr.grouping_functions is non-empty, bail (GROUPING SETS complicate this)
	// if aggr.grouping_sets.size() > 1, bail
	// if every one of the join's own RHS-side bindings is among aggr.groups:
	delim_join.join_type = JoinType::LEFT;
}
```

The logic: if the columns the outer join actually compares against are all
already part of the aggregate's own `GROUP BY` list, that aggregate is *by
construction* one row per distinct combination of those columns — the
`SINGLE`-join's own uniqueness enforcement is therefore provably redundant,
and the cheaper `LEFT` join (still preserving every outer row, still
producing `NULL` on the RHS side for outer rows with no match — exactly the
semantics `COUNT`-style empty-group defaulting from Chapter 6, §6.5 needs) can
replace it outright. Any non-column-reference expression anywhere along the
traced path (a computed group-by key, for instance) causes the function to
bail conservatively (`return;` with no change) rather than risk a false
positive.

There is one more bail-out. "The groups cover the join keys" only proves
one row per key when the aggregate has a single grouping set. With
`GROUPING SETS ((), ())`, the same aggregate emits one row *per set*, so a key
can get two rows. The check sits right after the `grouping_functions` test:

```cpp
// src/optimizer/deliminator.cpp:466-469
if (aggr.grouping_sets.size() > 1) {
	// multiple grouping sets emit one row per set, so the groups do not determine a unique row
	return;
}
```

Keeping `SINGLE` matters here. A scalar subquery must fail with "More than one
row returned by a subquery" when it returns two rows. The physical `SINGLE`
join raises that error; a `LEFT` join would silently return both rows.
`test/sql/subquery/lateral/lateral_empty_grouping_sets.test` sets
`delim_join_as_cte = false` at line 507, so the classic `Deliminator` path
runs, and then checks that the error still fires (lines 542–563):

```sql
SELECT o.k, (
    SELECT sum(v)
    FROM (VALUES (1, 42)) AS i(k, v)
    WHERE i.k = o.k
    GROUP BY GROUPING SETS ((), ())
) AS s
FROM (VALUES (1)) AS o(k);
-- error: More than one row returned by a subquery
```

The CTE-domain path (§13.3.6) has the same guard, in
`SingleJoinRHSIsDeduplicated` in `delim_join_cte_rewriter.cpp`.

### 13.3.6 A parallel implementation worth knowing about: the CTE-domain path

`test/optimizer/deliminator.test` mentions a second on/off switch,
`SET delim_join_as_cte=true`. That switch is already on by default, so in
practice the whole file tests what happens under it. This is worth being
precise about, because it is easy to conflate with `Deliminator` itself: when
`delim_join_as_cte` is on (the default), `DelimJoinCTERewriter::Rewrite`
(`src/planner/subquery/delim_join_cte_rewriter.cpp`) runs from
**`FlattenDependentJoins::DecorrelateIndependent`**
(`src/planner/subquery/flatten_dependent_join.cpp:286-288`) — in the planner,
right after decorrelation itself, and always before
`Optimizer::Optimize` is invoked at all:

```cpp
// src/planner/subquery/flatten_dependent_join.cpp:281-292
unique_ptr<LogicalOperator> FlattenDependentJoins::DecorrelateIndependent(Binder &binder,
                                                                          unique_ptr<LogicalOperator> plan) {
	CorrelatedColumns correlated;
	FlattenDependentJoins flatten(binder, correlated);
	flatten.DecorrelateSubtree(plan, true, {});
	if (Settings::Get<DelimJoinAsCteSetting>(binder.context)) {
		DelimJoinCTERewriter::Rewrite(binder, plan);
	}
	// ANY joins must retain their general shape until enclosing decorrelation and binding rewrites are complete.
	FinalizeAnyJoins(plan);
	return plan;
}
```

This rewriter converts every fully-decorrelated `LOGICAL_DELIM_JOIN` into an
ordinary join against a **materialized CTE** (named `__duckdb_delim` in
`EXPLAIN` output) instead of the classic `LogicalDelimGet` scaffolding. At the
end it checks with `VerifyNoDelim` (`delim_join_cte_rewriter.cpp:33-45`) that
no `LOGICAL_DELIM_JOIN` or `LOGICAL_DELIM_GET` is left. Inside that same
conversion it also runs its own redundant-domain-join eliminator,
`GeneratedDedupRefEliminator` (`delim_join_cte_rewriter.cpp:954` onward),
which implements the *identical* "decoupling" idea against the CTE
representation: it too builds an explicit `OPERATOR_IS_NOT_NULL` compensation
filter when it deletes a redundant join
(`delim_join_cte_rewriter.cpp:1632-1635`, byte-for-byte the same construction
as `deliminator.cpp:234-237`). Crucially, this eliminator gates itself on
**the exact same setting `Deliminator` itself checks**:

```cpp
// src/planner/subquery/delim_join_cte_rewriter.cpp:2641-2645
DelimJoinCTERewriter::DelimJoinCTERewriter(Binder &binder) : binder(binder) {
	auto &config = DBConfig::GetConfig(binder.context);
	cte_deliminator_enabled = Settings::Get<EnableOptimizerSetting>(binder.context) &&
	                          !config.IsOptimizerDisabled(OptimizerType::DELIMINATOR);
}
```

and profiles itself under the **literal same profiler key** `Deliminator`
uses, even though it is never invoked through `Optimizer::RunOptimizer` at
all:

```cpp
// src/planner/subquery/delim_join_cte_rewriter.cpp:31
static constexpr const char *CTE_DELIMINATOR_PROFILER_KEY = "optimizer.deliminator";
```

So the accurate statement is: **`SET disabled_optimizers = 'deliminator'`
turns off two separate pieces of code, in two separate files, in two separate
compilation directories (`src/optimizer/` and `src/planner/subquery/`), one of
which runs from the fixed optimizer pipeline this chapter otherwise
describes, and one of which runs from the planner before that pipeline even
starts.** With default settings, only the second one has any work to do. They exist because they clean up two different *representations* of
the exact same underlying construct — classic `LogicalDelimGet` scaffolding
versus a materialized `__duckdb_delim` CTE — but they are, conceptually, the
same optimization applied twice, by design, to whichever representation the
`delim_join_as_cte` setting produced. This chapter's remaining depth is spent
on the `src/optimizer/deliminator.cpp` half specifically, since that is the
piece that actually lives inside the fixed pass sequence Chapter 7 as a whole
documents. But note what that means: a reader who runs a default query from
`deliminator.test` under a debugger lands in `delim_join_cte_rewriter.cpp`,
not in `Deliminator::Optimize`. To reach `Deliminator::Optimize`, first run
`SET delim_join_as_cte = false`.

## 13.4 Every case in `test/optimizer/deliminator.test`

The file runs on TPC-H at `sf=0.0001` (`CALL dbgen(sf=0.0001)`) plus several
purpose-built small tables, with `PRAGMA explain_output = OPTIMIZED_ONLY` so
every `EXPLAIN` shown is the tree *after* all ~45 passes have run, not just
after `Deliminator`. Every case is grouped below by the specific behavior it
demonstrates.

One thing to keep in mind while reading this section: the file never sets
`delim_join_as_cte = false`, so with the default setting every query in it
runs in the CTE form (§13.3.6). That is why its patterns accept either form,
for example `(DELIM_GET|CTE_SCAN)` and `(.*DELIM_JOIN.*)|(.*CTE Name:.*__duckdb_delim.*)`.
The explanations below use `Deliminator`'s DelimJoin vocabulary, because
that is the form the patterns were first written for. In a default run, the
CTE rewriter's own eliminators do the matching work.

**TPC-H Q2 and Q20 — `SINGLE` → `LEFT` downgrade, at real query scale.** The
first and last two blocks (lines 20–23, 50–54 of the test file) are full TPC-H
Q2 and Q20 — both genuinely complex queries with several joins and nested
`IN`/scalar-subquery correlation — and both assert nothing more specific than
`<!REGEX>:.*SINGLE.*`: no `JoinType::SINGLE` survives *anywhere* in the final
optimized plan. This is `TrySwitchSingleToLeft` (§13.3.5) firing on every
DelimJoin these two queries produce, confirming the mechanism holds at
realistic multi-join query complexity, not just on toy single-table examples.

**TPC-H Q17 — the deepest single test block, exercising almost every
mechanism at once.** Four consecutive blocks (lines 26–47) use the same base
query (§13.5 below reproduces and traces this one in full):

```sql
SELECT sum(l_extendedprice) / 7.0 AS avg_yearly
FROM lineitem, part
WHERE p_partkey = l_partkey AND p_brand = 'Brand#23' AND p_container = 'MED BOX'
  AND l_quantity < (SELECT 0.2 * avg(l_quantity) FROM lineitem WHERE l_partkey = p_partkey);
```

- Line 26–29 again asserts no `SINGLE` survives (the scalar `AVG` subquery's
  DelimJoin gets downgraded to `LEFT`, exactly as in §13.3.5).
- Line 32–35 asserts the join-with-DelimGet is **retained**: `Groups:
  p_partkey ... COMPARISON_JOIN ... (l_partkey = p_partkey|p_partkey =
  l_partkey) ... (DELIM_GET|CTE_SCAN)`. This is the `HasSelection`-driven
  retention from §13.3.2 step (1): the `p_brand = 'Brand#23' AND p_container
  = 'MED BOX'` filters sit on the DelimJoin's own domain side, so
  `HasSelection` returns true, the deepest (and, here, only) join-with-DelimGet
  is spared removal, and the aggregate keeps restricting its own input by the
  narrowed domain rather than aggregating over every distinct `p_partkey` in
  all of `part`.
- Line 38–41 asserts `l_orderkey` (a column of `lineitem` genuinely unused by
  this subquery) does not survive into the generated domain — this is
  `UNUSED_COLUMNS`/column-lifetime pruning (§7.4, passes #23/#26) acting on
  the CTE materialization, not `Deliminator`'s own doing, but the test file
  places it here because it's only observable in combination with the
  DelimJoin's own domain materialization.
- Line 44–47, with the `p_brand`/`p_container` filters *removed* from the
  query, asserts the **whole** DelimJoin structure vanishes: `<!REGEX>:
  (.*DELIM_JOIN.*)|(.*CTE Name:.*__duckdb_delim.*)`. With no selective filter
  left on the domain side, `HasSelection` returns false, `RemoveJoinWithDelimGet`
  is allowed to fire on the (now sole) join-with-DelimGet, and — since
  `candidate.joins.size() == candidate.delim_get_count` — the outer DelimJoin
  itself gets downgraded straight to an ordinary `LOGICAL_COMPARISON_JOIN` per
  §13.3.2 step (3). §13.5 traces this exact collapse operator by operator.

**TPC-H Q20 — the two-key domain, and a false-tautology guard.** Lines 52–66
use Q20's nested `IN`/scalar-subquery shape, where the correlated subquery
inside the `IN` filters on *two* correlated columns at once (`l_partkey =
ps_partkey AND l_suppkey = ps_suppkey`). Line 57–60 asserts both keys still
restrict `lineitem` before the aggregate (`Groups: ps_suppkey, ps_partkey ...
Join Type: SEMI ... l_partkey = ps_partkey ... l_suppkey = ps_suppkey ...
(DELIM_GET|CTE_SCAN)`) — the multi-column analogue of Q17's single-key case,
and a direct confirmation that `RemoveJoinWithDelimGet`'s `conditions.size()
!= delim_get.chunk_types.size()` guard (§13.3.3) is doing its job: a
two-column DelimGet compared by exactly two conditions is eligible; nothing
here would collapse if only one of the two were compared. Line 63–66 is a
narrower regression-style assertion: after the redundant domain join *is*
eliminated elsewhere in the plan, no tautological, self-referential filter
like `l_partkey IS NOT DISTINCT FROM l_partkey` is left behind — i.e. the
`IS NOT NULL(other_side)` compensation filter (§13.3.3) must be built from
the *correlated table's own* column, never accidentally from a
self-comparison of the same expression against itself.

**The "storm" CTE query — retaining `MARK` when SEMI can't be proven safe.**
Lines 68–105 build a synthetic multi-CTE query (`ranked_posts`, `top_posts`)
whose innermost correlated predicate is genuinely **pair-dependent**: an
`EXISTS` subquery referencing `rp.post_id`, where `rp` is itself a
`rank()`-windowed, already-joined CTE. The assertion,
`Join Type: (MARK|SEMI|RIGHT_SEMI) ... post_id ... post_id ...
(user_id = #N|#N = user_id)`, is described in the test's own comment as
testing that "column liveness cannot prove that conversion to SEMI is safe" —
i.e., this is a case where the
mark-to-semi simplification a *different* pass
(`pushdown_mark_join.cpp`'s `SimplifyNullSafeSemiJoinConditions`, briefly
mentioned in §13.6 below) would otherwise perform is correctly declined,
because the correlation column's non-nullability cannot be proven through the
window/CTE indirection — the query is included in `deliminator.test`
specifically because it interacts with `Deliminator`'s own MARK-preserving
behavior on the way there.

> **Note:** the assertion also accepts `RIGHT_SEMI`, and it does not expect
> the join key at a fixed position such as `#0`. A comment at line 71 explains
> why: verification projections can change the numbers of unnamed join keys. The plan shape the
> test protects is the same.

**The `integers`/`SUM` block — scalar-aggregate NULL edge cases, without a
plan-shape assertion.** Lines 107–249 are a long, purely result-checking
sequence (`query II`, no `EXPLAIN`) exercising a single-table correlated
scalar aggregate (`(SELECT SUM(i) ... FROM integers WHERE j = i1.j)`) wrapped
in every combination of `IS NULL`/`IS NOT NULL`/`AND`/`OR`/`IS [NOT] DISTINCT
FROM`/`CONCAT`/list-literal/struct-literal the test authors could think of,
against a table containing an explicit `(NULL, 1)` row. These don't assert
anything about `Deliminator`'s plan shape directly — they assert that
whatever `Deliminator` (and `FlattenDependentJoins` before it) did to the plan
still produces the textbook SQL three-valued-logic answer for every one of
these wrapper shapes. This is the same kind of belt-and-suspenders,
result-level correctness check §13.4.1 below treats with more rigor for the
nullability file specifically — here it exists as breadth (many wrapper
shapes, one join shape) rather than depth (few wrapper shapes, many
NULL-introducing join shapes).

**The `delim_join_as_cte=true` block — proving the CTE-domain path removes
the same redundancy, on harder shapes.** From line 254 onward (through line
~797), the file sets `delim_join_as_cte=true` explicitly (it is already the
default, so this changes nothing) and runs a large family
of `EXISTS`/`NOT EXISTS`/mixed-boolean-marker queries against purpose-built
`outer_fact`/`evidence_a..d`/`selected_dim` tables. As §13.3.6 established,
this exercises `GeneratedDedupRefEliminator`, not `Deliminator` itself, but
the *shape* of what's being proven is the direct CTE-domain analogue of
everything §13.3.3–§13.3.5 established:

- Simple `EXISTS` predicates on a selective, non-nullable domain collapse
  their redundant domain join to `SEMI`/`RIGHT_SEMI` (lines 256–259,
  316–325) — the CTE-domain equivalent of `RemoveJoinWithDelimGet`.
- Filters that select the domain *from an evidence side* rather than the
  outer relation, or through a `LEFT`/`RIGHT` join that could reintroduce
  NULLs into the domain column, correctly **retain** the `__duckdb_delim` CTE
  rather than collapse it (lines 327–411) — this is the CTE-domain path's own
  version of the nullability-boundary reasoning §13.4.1 walks in full for the
  classic path.
- A projected (not merely filtered) marker expression cannot use the simple
  filter-to-SEMI conversion, and a marker referencing *two* independent
  equality keys against a single-key domain cannot reuse that domain either
  (lines 449–479, 464–479) — both are shape guards analogous to
  `RemoveJoinWithDelimGet`'s `conditions.size() != chunk_types.size()` check.
- Marker provenance survives being renamed through nested projection aliases
  (`coalesce(..., false)` wrapped two levels deep, lines 549–590) — the
  CTE-domain path's own binding-propagation mechanism, playing the same role
  `ColumnBindingReplacer::VisitOperator(*root)` plays for the classic path
  (§13.3.3's closing step).

**Disabling `deliminator` and re-running everything — proving cleanup is
optional, not load-bearing.** Line 672 sets `disabled_optimizers='deliminator'`
(while `delim_join_as_cte` is still `true` from the earlier block) and
re-executes essentially every correctness-checking query from earlier in the
file, expecting byte-identical answers. This is the single most important
test in the file from an architectural standpoint, because it is a direct,
executable demonstration of §6.7's claim: disabling this pass changes *only*
which redundant scaffolding survives in the plan, never what the query
computes. Line 797 resets `disabled_optimizers=''` before the file's final
two blocks (TPC-H Q22 and Q21, lines 799–815), which close out the file by
confirming the same domain-restriction and MARK-retention behavior once more
at full TPC-H query complexity.

## 13.4.1 Case study: `deliminator_nullability.test` and issue #23658

**The underlying correctness issue.** `RemoveInequalityJoinWithDelimGet`'s
`MARK`+`COMPARE_NOTEQUAL` rewrite (§13.3.4) needs to know that a particular
expression — the DelimJoin's own domain-side column — **cannot be NULL** at
the point the rewrite reasons about it, because the flip it performs is only
sound when the original `!=` comparison's implicit NULL-rejection is either
preserved or provably unnecessary. This checkout's `git log` for this test
file shows the fix commit directly:

```
ae6227827f  fix nullability issue mentioned in #23658
```

Before that commit, this specific gate did not exist in `Deliminator` at all
— the flip ran unconditionally. What the fix added is a **new, general,
plan-aware nullability prover**, `NotNullExpressionAnalyzer`
(`src/planner/expression_nullability.{hpp,cpp}`, new files in that same
commit), and it is worth being precise about *why* a general prover was
necessary rather than a simple schema check. A narrower nullability helper
already existed at the time, living inside a completely different pass
(`ExpressionIsNotNull` in `src/optimizer/pushdown/pushdown_mark_join.cpp`,
used by `SimplifyNullSafeSemiJoinConditions` to decide whether a null-safe
`SEMI` join condition can be simplified to plain equality). That helper
handled exactly three operator types on its way down from an expression to
its source: `LOGICAL_PROJECTION` (trace through), `LOGICAL_FILTER` (check for
an explicit rejecting predicate), and `LOGICAL_GET` (check the scan's own
table filters, then fall back to the base table's catalog-declared `NOT NULL`
constraint via column statistics). Anything else — critically, a
`LOGICAL_COMPARISON_JOIN` or a `LOGICAL_CTE_REF` sitting between the
expression and its ultimate source — was **not handled**, and hit a
conservative `default: return false`.

That was safe wherever it was the *only* caller, because "cannot prove"
simply meant "skip the simplification." But `Deliminator`'s new MARK+`!=`
call site needed to reason starting from `delim_join.children[0]` —
**the DelimJoin's own domain-producing subtree** — and for a query like:

```sql
SELECT ids.id, nn.wh,
       EXISTS (SELECT 1 FROM ev WHERE ev.id = ids.id AND nn.wh <> ev.wh)
FROM ids LEFT JOIN nn ON ids.id = nn.k
ORDER BY ids.id;
```

that domain-producing subtree **is itself** `ids LEFT JOIN nn` — because the
`EXISTS`'s free reference `nn.wh` is resolved against the outer query's own
join output, not against a bare base table. `nn.wh` is declared `wh INTEGER
NOT NULL` in the schema (`CREATE TABLE nn(k INTEGER, wh INTEGER NOT NULL)`),
so a check that only ever consulted the catalog constraint — without asking
whether an enclosing operator could still null-pad the column at the point
being reasoned about — would have concluded "not null" unconditionally. But
`nn` sits on the *nullable* side of a `LEFT JOIN`: for `ids.id = 2` (no
matching row in `nn`), `nn.wh` is genuinely `NULL` in the join's actual
output, `NOT NULL` column constraint notwithstanding. Trusting the declared
constraint in isolation here would let the MARK+`!=` flip proceed on a false
premise, and the query's correct answer — `false` for `ids.id = 2`, because
`nn.wh <> ev.wh` is `NULL` (unknown) when `nn.wh` is `NULL`, and `EXISTS` over
an all-`NULL`/no-match result is `false` — could silently become wrong.

**The fix's shape.** `NotNullExpressionAnalyzer::IsNotNull`
(`expression_nullability.cpp:127-244`) keeps the same `PROJECTION`/`FILTER`/
`GET` cases essentially unchanged, but adds exactly the two the narrow
pushdown-only helper never needed:

```cpp
// src/planner/expression_nullability.cpp:86-105
static bool JoinOutputPreservesChildNonNullability(JoinType join_type, idx_t child_idx) {
	switch (join_type) {
	case JoinType::INNER:
		return child_idx < 2;
	case JoinType::LEFT:
	case JoinType::SEMI:
	case JoinType::ANTI:
	case JoinType::MARK:
	case JoinType::SINGLE:
		return child_idx == 0;
	case JoinType::RIGHT:
	case JoinType::RIGHT_SEMI:
	case JoinType::RIGHT_ANTI:
		return child_idx == 1;
	case JoinType::OUTER:
	case JoinType::INVALID:
		return false;
	}
	return false;
}
```

— i.e., a `LOGICAL_COMPARISON_JOIN` case that traces which single child
produced the binding in question, and only recurses into proving that
child's non-nullability if the join type *provably* cannot null-pad that
particular side (`INNER` preserves both; `LEFT`/`SEMI`/`ANTI`/`MARK`/`SINGLE`
preserve only the LHS; `RIGHT`/`RIGHT_SEMI`/`RIGHT_ANTI` preserve only the
RHS; a full `OUTER` join preserves neither) — plus a `LOGICAL_CTE_REF` case
that recurses into the CTE's own defining subtree, with a `seen_ctes` guard
against infinite recursion for exactly the recursive-CTE case the test file
covers. For `ids LEFT JOIN nn`, `nn.wh` is produced by child index 1 of a
`LEFT` join, and `JoinOutputPreservesChildNonNullability(LEFT, 1)` returns
`false` — the analyzer now correctly refuses to prove non-nullability there,
`RemoveInequalityJoinWithDelimGet`'s gate returns `false`, and the unsafe
rewrite is declined. (The same commit also generalized the *original*
`pushdown_mark_join.cpp` call site to reuse this shared analyzer instead of
its own ~127-line copy — collapsing duplicated logic into two call sites
sharing one implementation, and giving that older, narrower call site the
same join/CTE awareness for free, as a byproduct rather than the point of the
fix.)

**Walking the test file's cases against this mechanism.** Every scenario in
`deliminator_nullability.test` is a different way of making a
catalog-`NOT NULL` column observably `NULL` at the point `Deliminator`
reasons about it, run under `foreach cte_mode true false` so both the classic
and CTE-domain representations are exercised identically:

- **LEFT join, nullable side** (lines 49–63) — the case just walked. `id=2`
  correctly returns `false`, not a wrong answer from an incorrectly-applied
  rewrite.
- **RIGHT join, same effect from the other side** (lines 65–79) — swapping
  which side of the join is written first (`FROM nn RIGHT JOIN ids`) still
  puts `nn` on the side a `RIGHT` join can null-pad; the analyzer's `RIGHT`
  case (`child_idx == 1` required, but here `nn` is child 0) correctly
  refuses again.
- **FULL join, both sides nullable** (lines 81–102) — `JoinOutputPreservesChildNonNullability(OUTER,
  ...)` returns `false` unconditionally for both children, matching that a
  full outer join can null-pad either side.
- **Scalar subquery via `LEFT JOIN LATERAL`, a `SINGLE` join with no match**
  (lines 104–120) — this is the case that ties directly back to §13.3.5's own
  subject matter: a `SINGLE` join's RHS output is `NULL` whenever no matching
  row exists, which is exactly why `JoinOutputPreservesChildNonNullability`
  lists `SINGLE` alongside `LEFT`/`SEMI`/`ANTI`/`MARK` as "only child 0
  (LHS) is safe" — the RHS of a `SINGLE` join is never provably non-NULL from
  the join alone, matched-row-or-not.
- **Materialized CTE, then recursive CTE** (lines 122–164) — the two cases the
  new `LOGICAL_CTE_REF` handling exists for specifically. The plain
  materialized-CTE case recurses one level into the CTE's own definition
  (finds the same `LEFT JOIN` underneath, same correct refusal). The
  recursive CTE case is the more interesting one: the test's own comment says
  its result is "conservatively nullable even when its seed reads a NOT NULL
  base column" — a `LOGICAL_RECURSIVE_CTE`/working-table reference isn't a
  plain `LOGICAL_CTE_REF` the analyzer's `FindCTE`/single-definition-recursion
  logic is built to trace, so it falls through to the `default: return false`
  case unconditionally, refusing to prove anything about recursive-CTE output
  — correct (if maximally conservative) either way, and the test asserts only
  the *correct answer*, not a specific plan shape, for exactly this reason.
- **Explicit null-rejection re-enables the optimization** (lines 166–195) —
  adding `WHERE nn.wh IS NOT NULL` above the same `LEFT JOIN` is asserted, via
  `EXPLAIN`, to *restore* the MARK-condition rewrite (`Join Type: MARK ... wh
  IS DISTINCT FROM wh`) — this is the analyzer's `LOGICAL_FILTER` case
  (`FilterRejectsNull`) doing exactly its job: an explicit `IS NOT NULL`
  filter sitting above the join *does* prove the expression non-NULL at that
  point, regardless of what the join underneath could otherwise produce, and
  the rewrite correctly re-engages. The plain `INNER JOIN` variant
  immediately below (lines 183–195) gets the same rewrite for a structurally
  simpler reason: `INNER` preserves non-nullability from either side
  unconditionally.
- **Nested `STRUCT`/`LIST` equality — PostgreSQL child-NULL semantics** (lines
  197–248) — a materially different correctness axis from everything above:
  DuckDB's nested-value equality (`STRUCT`/`LIST`) follows PostgreSQL's
  convention that two composite values compare equal if their corresponding
  fields do, **even when an individual field is `NULL`** — `{'x': NULL, 'y':
  1} = {'x': NULL, 'y': 1}` is `TRUE`, not `NULL`, under this convention,
  because the composite values themselves are not NULL, only one of their
  members is. `Deliminator`'s equality/binding-replacement machinery
  (`IsEqualityJoinCondition`, the `IS NOT NULL` compensation logic) has to
  agree with this convention rather than the scalar convention, or an
  eliminated join's compensation filter would incorrectly reject a
  legitimately-matching nested value merely because one of its members
  happens to be `NULL`. The test's own git history names the fix commit that
  hardened this specifically: `f93318fab7` ("Preserve nested comparison
  semantics in deliminator"), followed by `2952b17aca` ("Share
  IsNestedComparison helper and cover second-pass nested elimination") and
  `c373ad8861` ("Update nested tests for PostgreSQL semantics") — a small
  chain of follow-up commits refining exactly this one boundary, which is
  itself a useful reminder that "preserve NULL semantics through a deleted
  join" is not a single fix-once problem; it recurs every time a new value
  category (nested structs, then lists) is checked against the same
  compensation-filter machinery.
- **Nested correlation columns through a generated CTE domain, plus a final
  self-equality check** (lines 249–267) — the closing cases confirm the same
  nested-equality correctness holds when the domain is threaded through
  `RemoveGeneratedDedupJoin`'s CTE-based sibling mechanism (§13.3.6), and end
  with `b = b` on a nested `STRUCT` correctly returning only the *non-NULL*
  top-level rows (`k=3`, whose `a` is itself `NULL` at the top level, is
  excluded) — a direct, minimal confirmation that composite-value `NULL`-at-
  the-top-level (as opposed to `NULL` in a member field) still behaves like
  ordinary scalar `NULL` for equality purposes.

## 13.5 `DELIMINATOR` worked example: TPC-H Q17, three real states

This trace shows the DelimJoin form, which you get with
`SET delim_join_as_cte = false`. With the default setting, State 1 is
already in the CTE form of Chapter 6 §6.4.1 when the optimizer starts, and the
CTE rewriter's own eliminator does the work described here.

This traces the **unfiltered** variant of the Q17 query from §13.4 (`p_brand`/
`p_container` predicates removed, so the domain carries no selective filter
and the whole DelimJoin is provably eliminable — the "we can remove the whole
DELIM join" case the test file itself calls out):

```sql
SELECT sum(l_extendedprice) / 7.0 AS avg_yearly
FROM lineitem, part
WHERE p_partkey = l_partkey
  AND l_quantity < (SELECT 0.2 * avg(l_quantity) FROM lineitem WHERE l_partkey = p_partkey);
```

**State 1 — immediately after `FlattenDependentJoins` (end of Chapter 6's
machinery, start of the optimizer pipeline).** The scalar `AVG` subquery
becomes a `JoinType::SINGLE` DelimJoin. Aggregation-pushdown (Chapter 6, §6.5)
has already added the correlation column to the subquery's own `GROUP BY`;
filter-pushdown has already threaded the correlation predicate down to where
`lineitem` (the *correlated*, inner instance — a separate scan from the
`lineitem` the outer `FROM` clause itself joins against) is reached, and —
this is the detail this chapter adds on top of Chapter 6's pedagogical
Query-B diagram — the correlation reference resolves there via an explicit
`LogicalDelimGet`, joined ordinarily against that `lineitem` instance, rather
than folded invisibly into some enclosing node's own condition list:

```
LogicalProjection        (sum(l_extendedprice) / 7.0 AS avg_yearly)
 └─ LogicalAggregate       (Γ: sum(l_extendedprice))
      └─ LogicalFilter      (l_quantity < <subquery result>)
           └─ LogicalComparisonJoin      (LOGICAL_DELIM_JOIN, JoinType::SINGLE,
                                          duplicate_eliminated_columns = [p_partkey])
                ├─ LogicalComparisonJoin   (p_partkey = l_partkey)        — outer FROM-clause join
                │    ├─ LogicalGet (part)
                │    └─ LogicalGet (lineitem)                             — outer instance
                └─ LogicalProjection        (0.2 * avg(l_quantity))
                     └─ LogicalAggregate     (Γ: avg(l_quantity)  groups: p_partkey)
                          └─ LogicalComparisonJoin   (l_partkey = p_partkey)
                               ├─ LogicalGet (lineitem)                    — correlated instance
                               └─ LogicalDelimGet  (chunk_types: [p_partkey])
```

**State 2 — after `DELIMINATOR`.** `FindCandidates` finds one
`LOGICAL_DELIM_JOIN`; `FindJoinWithDelimGet` walks its RHS and finds exactly
one join-with-DelimGet (`lineitem ⋈ DelimGet`, condition `l_partkey =
p_partkey`) and one bare `LogicalDelimGet` (`delim_get_count = 1`).
`HasSelection` walks the whole DelimJoin subtree and finds nothing (no
`p_brand`/`p_container` filter left anywhere) — the deepest-join protection
from §13.3.2 does **not** trigger this time, so `RemoveJoinWithDelimGet` is
attempted and succeeds: the condition is a plain equality between two bound
column references (`p_partkey` from the DelimGet, `l_partkey` from
`lineitem`), so an `IS NOT NULL(l_partkey)` compensation filter is added, the
join is replaced by `LogicalFilter(l_partkey IS NOT NULL) → LogicalGet(lineitem)`,
and every reference to the DelimGet's `p_partkey` binding — most importantly,
the aggregate's own `GROUP BY` expression — is globally rebound to
`lineitem.l_partkey` via `ColumnBindingReplacer`. Since `candidate.joins.size()
== candidate.delim_get_count` (1 == 1) and the removal succeeded,
`delim_join.type` is downgraded from `LOGICAL_DELIM_JOIN` to plain
`LOGICAL_COMPARISON_JOIN` and `duplicate_eliminated_columns` is cleared.
Independently, `TrySwitchSingleToLeft` also fires (the aggregate's own
`GROUP BY` binding, unaffected by which *source* expression feeds it, is
still exactly the column the outer join's own condition compares against),
downgrading `join_type` from `SINGLE` to `LEFT`:

```
LogicalProjection        (sum(l_extendedprice) / 7.0 AS avg_yearly)
 └─ LogicalAggregate       (Γ: sum(l_extendedprice))
      └─ LogicalFilter      (l_quantity < <subquery result>)
           └─ LogicalComparisonJoin      (ordinary join now — no LOGICAL_DELIM_JOIN
                                          tag, no duplicate_eliminated_columns;
                                          join_type downgraded SINGLE -> LEFT)
                ├─ LogicalComparisonJoin   (p_partkey = l_partkey)        — outer FROM-clause join
                │    ├─ LogicalGet (part)
                │    └─ LogicalGet (lineitem)                             — outer instance
                └─ LogicalProjection        (0.2 * avg(l_quantity))
                     └─ LogicalAggregate     (Γ: avg(l_quantity)  groups: l_partkey)
                          └─ LogicalFilter    (l_partkey IS NOT NULL)
                               └─ LogicalGet (lineitem)                    — correlated instance
```

Both the join-with-DelimGet removal and the `SINGLE`→`LEFT` downgrade are on
display in one query — exactly matching the test file's own two separate
assertions for this shape (§13.4's Q17 discussion).

**State 3 — after `UNNEST_REWRITER`.** By the time this pass runs (twelve
passes later), the tree above no longer contains a single `LOGICAL_DELIM_JOIN`
node anywhere — `Deliminator` already retagged the operator as an ordinary
`LOGICAL_COMPARISON_JOIN` in State 2. `UnnestRewriter::FindCandidates`'s very
first structural check (`op->children[0]->type != LOGICAL_DELIM_JOIN`,
`unnest_rewriter.cpp:133-135`) fails immediately at every point in the tree.
State 3 is therefore **byte-identical to State 2** — a clean, confirmable
no-op, for the same underlying reason §7.7 gives for why `DELIMINATOR`/
`UNNEST_REWRITER` are no-ops on Query A: there is nothing here for this pass's
specific pattern to match.

## 13.6 `UNNEST_REWRITER`, in full

**Purpose in one sentence, building on Chapter 6's framing:** where a
correlated subquery whose body is `UNNEST` gets decorrelated via the
`row_number()`-window scaffolding Chapter 6, §6.7 already introduced (rather
than a full duplicate-eliminating aggregate), `UnnestRewriter` recognizes that
this scaffolding was only ever needed to give the `UNNEST` a per-row key to
join back on, and — once it can prove the `UNNEST`'s own input is reachable
directly — deletes the DelimJoin and the window operator entirely, splicing
the `UNNEST` to read straight from the original per-row input instead.

With default settings the DelimJoin has already become a CTE when this pass
runs (§13.3.6), so the CTE-domain variants in §13.6.4 are the ones a default
query takes. §13.6.1–§13.6.3 and §13.7 describe the DelimJoin form
(`SET delim_join_as_cte = false`); the matching logic is the same.

### 13.6.1 The exact pattern: two variants of "RHS bottoms out in an UNNEST fed by a DelimGet"

`UnnestRewriter::FindCandidates` (`unnest_rewriter.cpp:122-240`) walks the
tree bottom-up (children first, exactly like `Deliminator::FindCandidates`)
and, for each operator with **exactly one child** whose type is
`LOGICAL_DELIM_JOIN`, checks a fixed structural shape before accepting it as
a candidate:

```cpp
// src/optimizer/unnest_rewriter.cpp:129-153
// search for operator that has a LOGICAL_DELIM_JOIN as its child
if (op->children.size() != 1) {
	return;
}
if (op->children[0]->type != LogicalOperatorType::LOGICAL_DELIM_JOIN) {
	return;
}

// found a delim join
auto &delim_join = op->children[0]->Cast<LogicalComparisonJoin>();
// only support INNER delim joins
if (delim_join.join_type != JoinType::INNER) {
	return;
}
// INNER delim join must have exactly one condition
if (delim_join.conditions.size() != 1) {
	return;
}

// LHS child is a window
idx_t delim_idx = delim_join.delim_flipped ? 1 : 0;
idx_t other_idx = 1 - delim_idx;
if (delim_join.children[delim_idx]->type != LogicalOperatorType::LOGICAL_WINDOW) {
	return;
}
```

So the DelimJoin must be a single-condition **`INNER`** join whose LHS is a
`LogicalWindow` — exactly the `row_number()`-scaffolded LHS Chapter 6's
`CreateRowNumberWindow` (`flatten_dependent_join.cpp:406-422`) produces for
the `!perform_delim` UNNEST path. From there, two distinct RHS shapes are
recognized:

**Pattern 1 — a `LogicalUnnest` reading directly from a `LogicalDelimGet`,**
possibly through a chain of projections:

```cpp
// src/optimizer/unnest_rewriter.cpp:155-169
// RHS child must be projection(s) followed by an UNNEST
auto curr_op = &delim_join.children[other_idx];
while (curr_op->get()->type == LogicalOperatorType::LOGICAL_PROJECTION) {
	if (curr_op->get()->children.size() != 1) {
		break;
	}
	curr_op = &curr_op->get()->children[0];
}

// pattern1: delim_get -> unnest-> projection
if (curr_op->get()->type == LogicalOperatorType::LOGICAL_UNNEST &&
    curr_op->get()->children[0]->type == LogicalOperatorType::LOGICAL_DELIM_GET) {
	candidates.push_back(op);
	return;
}
```

This is the pattern §13.7's worked example uses: `SELECT UNNEST(i)` written
directly as a correlated scalar subquery, where `i` is the outer row's array
column, threaded through the DelimGet as one of the duplicate-eliminated
columns.

**Pattern 2 — a `LOGICAL_GET`-shaped "table-in-out" UNNEST** (a table
function, not the `LogicalUnnest` operator — the distinction matters, and is
exactly what the case study in §13.8 turns on), reached via
`ExpressionBinder::IsUnnestFunction(get.function.GetName())`, whose own single
child is a `LogicalProjection` sitting directly atop the `LogicalDelimGet`:

```cpp
// src/optimizer/unnest_rewriter.cpp:171-240 (structure)
curr_op = &delim_join.children[other_idx];
if (curr_op->get()->type == LOGICAL_GET) {
	auto &get = curr_op->get()->Cast<LogicalGet>();
	if (!ExpressionBinder::IsUnnestFunction(get.function.GetName())) return;
	if (get.Cast<LogicalGet>().ordinality_idx.IsValid()) return; // WITH ORDINALITY: cannot rewrite
	curr_op = &curr_op->get()->children[0];
	if (curr_op->get()->type == LOGICAL_PROJECTION &&
	    curr_op->get()->children[0]->type == LOGICAL_DELIM_GET) {
		// verify every unnested column is a plain BOUND_COLUMN_REF in the projection...
		// ...then perform an in-place conversion of pattern 2 into pattern 1,
		// building a real LogicalUnnest node to replace the table-function GET.
	}
}
```

This second pattern exists because DuckDB has two independent ways a plan can
represent `UNNEST`: the dedicated `LogicalUnnest` operator, and a table
function (`get.function.GetName()` matching `IsUnnestFunction`) used when `UNNEST`
appears in table-function position (`FROM UNNEST(...)`, or, as §13.8's case
study shows, inside a macro's `TABLE(...)` body). Before this code can treat
the two uniformly, it defensively converts pattern 2 into pattern 1 in place
— but only after checking, **for every one of the DelimGet's columns that the
unnest reads**, that the projection expression exposing it is a plain
`BOUND_COLUMN_REF`:

```cpp
// src/optimizer/unnest_rewriter.cpp:194-206
auto &proj = curr_op->get()->Cast<LogicalProjection>();
for (idx_t i = 0; i < unnest_get_column.size(); i++) {
	auto &col_bind = unnest_get_column[i];
	if (col_bind.table_index != unnest_get_index) {
		// not part of the unnest
		continue;
	}
	auto &expr = proj.GetExpression(col_bind.column_index);
	if (expr.GetExpressionClass() != ExpressionClass::BOUND_COLUMN_REF) {
		// unnest reference is not a projection reference - bail
		return;
	}
}
```

This is exactly the defensive guard §13.8 traces back to a real, named
regression.

### 13.6.2 `RewriteCandidate` — the actual splice

Once a candidate is accepted, `RewriteCandidate`
(`unnest_rewriter.cpp:637-683`) does the rewrite in five steps:

```cpp
// src/optimizer/unnest_rewriter.cpp:637-683 (structure, elided)
bool UnnestRewriter::RewriteCandidate(unique_ptr<LogicalOperator> &candidate) {
	auto &topmost_op = *candidate;
	if (!IsSupportedUnnestTop(topmost_op.type)) return false;
	auto &delim_join = topmost_op.children[0]->Cast<LogicalComparisonJoin>();
	GetDelimColumns(delim_join);                       // (1)

	idx_t delim_idx = delim_join.delim_flipped ? 1 : 0;
	auto &window = *delim_join.children[delim_idx];
	auto &lhs_op = window.children[0];
	GetLHSExpressions(*lhs_op);                        // (2)

	// walk down through any LOGICAL_PROJECTIONs on the RHS to the LogicalUnnest
	auto &unnest = /* ... */;
	unnest.children[0] = std::move(lhs_op);             // (3)

	topmost_op.children[0] = std::move(path_to_unnest.front().get());  // (4)
	return true;
}
```

1. **`GetDelimColumns`** (`unnest_rewriter.cpp:858-867`) simply reads
   `delim_join.duplicate_eliminated_columns` — the set of columns the
   DelimGet re-exposes (in the worked example below, `[row_number, i]`).
2. **`GetLHSExpressions`** (`unnest_rewriter.cpp:869-890`) captures the
   *window's own child*'s output columns and, when that child is itself a
   plain `LogicalProjection`, their aliases — this is what the outer relation
   actually exposes (`t.i` in the worked example), independent of the
   synthetic `row_number()` column the window operator added on top.
3. **The splice's core move**: `unnest.children[0] = std::move(lhs_op)` —
   the `LogicalUnnest` node's child, which used to be the `LogicalDelimGet`,
   becomes the window's *own original input* directly. The DelimGet
   disappears from the tree entirely (it is simply never referenced by
   anything after this assignment).
4. **The DelimJoin (and the window sitting above the now-redirected LHS) are
   dropped wholesale**: `topmost_op.children[0]` — the slot that used to hold
   the whole `LOGICAL_DELIM_JOIN` subtree — is reassigned to
   `path_to_unnest.front()`, the topmost node of whatever chain of
   projections sat between the DelimJoin's RHS and the `LogicalUnnest` (or,
   if pattern 2 was converted to pattern 1 first, the newly-built
   `LogicalUnnest` itself). Neither the `LOGICAL_DELIM_JOIN` node nor the
   `LogicalWindow` node above `lhs_op` is referenced by anything in the tree
   after this line — both are simply dropped along with their now-orphaned
   `unique_ptr`s.

### 13.6.3 Why the binding bookkeeping is the hard part

Everything in §13.6.2 changes *which operator* produces which value, but the
surrounding tree still holds `ColumnBinding`s that pointed at the *old*
table indices — the DelimGet's own synthetic index, and the window operator's
projected `row_number` column index. `UnnestRewriter::Optimize`
(`unnest_rewriter.cpp:98-120`) always runs three further steps after a
successful `RewriteCandidate`, in this order:

1. **`UpdateBoundUnnestBindings`** (`unnest_rewriter.cpp:816-856`) fixes the
   `BOUND_UNNEST` expression's own argument — it used to read a DelimGet
   column; it must now read the corresponding column of whatever `lhs_op`
   actually is.
2. **`UpdateRHSBindings`** (`unnest_rewriter.cpp:685-814`) is the largest of
   the three: it walks every remaining `LogicalProjection` on the path from
   the new top down to the `LogicalUnnest`, strips the trailing
   duplicate-eliminated columns those projections used to carry purely to
   thread the DelimGet's values through, and **prepends** the LHS's own
   exposed columns (captured by `GetLHSExpressions` in step (2) above) to
   every one of those projections instead — because anything above the old
   DelimJoin that used to read `t.i` via the DelimJoin's own LHS output now
   needs to read it from wherever the RHS chain re-exposes it post-splice.
3. A tree-wide `ColumnBindingReplacer` pass (used internally by both of the
   above, and separately in the pattern-2-to-pattern-1 conversion,
   `unnest_rewriter.cpp:209-237`) propagates every one of these binding
   changes to every remaining reference anywhere in the plan — the same
   mechanism, and the same underlying reason, `Deliminator`'s own
   `replacer.VisitOperator(*root)` call exists (§13.3.3's closing step):
   deleting or repositioning an operator that used to own a `ColumnBinding`
   is only safe once every *other* reference to that binding, anywhere in
   the tree, has been retargeted.

**Chains of more than one projection.** Step 2 has a subtle case. Each
projection on the path carries the delimiter columns as its *last* columns
(the "trailing slots"). When there is only one projection, each trailing slot
reads the DelimGet (or a column of the LHS) directly, so it is easy to find
which LHS column it stands for. When there are two or more projections, an
upper projection's trailing slot reads the trailing slot of the projection
*below* it, not the DelimGet. Code that only handled the direct case would
record no replacement for the upper projection, so references above it would
keep pointing at slot numbers that had shifted to other columns, often of
another type.

So `UpdateRHSBindings` works in two passes. The first pass walks the
projections **bottom-up** and remembers, for each trailing slot it has
resolved, which LHS column it maps to. An upper slot that reads a lower slot
looks up that memory:

```cpp
// src/optimizer/unnest_rewriter.cpp:701-738
column_binding_map_t<idx_t> trailing_slot_to_lhs;
for (idx_t path_idx = path_to_unnest.size(); path_idx > 0; path_idx--) {
	auto &proj = path_to_unnest[path_idx - 1].get()->Cast<LogicalProjection>();
	D_ASSERT(proj.expressions.size() > distinct_unnest_count);
	auto payload_count = proj.expressions.size() - distinct_unnest_count;
	for (idx_t i = payload_count; i < proj.expressions.size(); i++) {
		auto &expr = proj.expressions[i];
		if (expr->GetExpressionClass() != ExpressionClass::BOUND_COLUMN_REF) {
			continue;
		}
		auto &colref = expr->Cast<BoundColumnRefExpression>();
		auto removed_binding = colref.Binding();
		optional_idx found_lhs_idx;
		auto chained = trailing_slot_to_lhs.find(removed_binding);
		if (chained != trailing_slot_to_lhs.end()) {
			found_lhs_idx = chained->second;
		} else {
			if (removed_binding.table_index == overwritten_tbl_idx &&
			    removed_binding.column_index.GetIndex() < delim_columns.size()) {
				// CTE-based rewrites can still reference the deduplicated input here; translate it back to the
				// corresponding domain producer column before finding the new LHS projection slot.
				removed_binding = delim_columns[removed_binding.column_index.GetIndex()];
			}
			for (idx_t lhs_idx = 0; lhs_idx < lhs_bindings.size(); lhs_idx++) {
				if (removed_binding == lhs_bindings[lhs_idx].binding) {
					found_lhs_idx = lhs_idx;
					break;
				}
			}
		}
		if (found_lhs_idx.IsValid()) {
			ColumnBinding source_binding(proj.table_index, ProjectionIndex(i));
			ColumnBinding target_binding(proj.table_index, ProjectionIndex(found_lhs_idx.GetIndex()));
			updater.replace_bindings.emplace_back(source_binding, target_binding);
			trailing_slot_to_lhs[source_binding] = found_lhs_idx.GetIndex();
		}
	}
}
```

The second pass (`unnest_rewriter.cpp:740-750`) then drops the trailing slots
from every projection and records the shift of the remaining payload columns.

The test for this is
`test/sql/subquery/lateral/lateral_unnest_intermediate_projection.test`. Its
first query puts one projection (`r`) between the correlated reference `xs[j]`
and the `unnest`:

```sql
SELECT * FROM (SELECT [1,2,3]::INTEGER[] AS xs, 1 AS i) s
CROSS JOIN (SELECT xs[j] AS a FROM (SELECT elem AS j FROM
  (SELECT unnest(range(i+1, 4)) AS elem) b) r) e
ORDER BY a;
-- [1, 2, 3]  1  2
-- [1, 2, 3]  1  3
```

Later cases add a second intermediate projection, a `SELECT *` wrapper, a
multi-row outer relation, and (from line 178) the same queries with
`cte_inlining` disabled, so the explicit dedup-CTE path runs too.

### 13.6.4 CTE-domain variants — the same idea, for materialized `__duckdb_delim` CTEs

Just as `Deliminator` has a CTE-domain sibling (§13.3.6),
`UnnestRewriter::RewriteCTECandidate` and
`RewriteInlineCTEDedupCandidate` (`unnest_rewriter.cpp:382-635`) recognize the
identical UNNEST-fed-by-domain shape when `delim_join_as_cte` has already
converted the DelimJoin into a `LogicalMaterializedCTE`/`LogicalCTERef` pair
rather than a `LogicalDelimGet`. The pattern-matching is structurally the
same three-part check (LHS window/domain producer, a single-condition inner
join, an `UNNEST` reading the domain's own columns through a chain of
projections) — `RewriteInlineCTEDedupCandidate` additionally has to trace the
domain's dedup columns back through an *already-inlined* `LogicalAggregate`
(when an earlier `CTE_INLINING` pass has removed the explicit deduplicating
CTE the classic path would otherwise still show), via `GetInlineDedupColumns`
(`unnest_rewriter.cpp:310-380`) — a direct illustration of exactly the kind
of "a later pass changed the shape an earlier pass's pattern-matcher expects"
interaction §7.2 warns is structurally possible in a fixed, one-shot pipeline:
here, rather than missing the optimization outright, `UnnestRewriter` carries
the extra tracing logic needed to still recognize its own pattern through
that shape change.

### 13.6.5 Fixed-size `ARRAY` input: the cast the rewrite must rebuild

`UNNEST` works on `LIST` values. A fixed-size `ARRAY` column (for example
`INT[2]`) is a different type. When the binder sees `UNNEST(m.fixed)`, it
wraps the argument in a cast from `ARRAY` to `LIST`.

The pattern-2 conversion in §13.6.1 builds a new `BoundUnnestExpression` from
the projected column. If it copied the bare column, it would lose the
binder's cast. The `UNNEST` would then run on a vector that is not a list.
Builds with assertions would fail one in `list_vector.cpp`; release builds
would crash (this guards against DuckDB issue #26435). So the rewriter adds
the cast back with the same helper the binder uses:

```cpp
// src/optimizer/unnest_rewriter.cpp:222-224
auto unnest_expr = make_uniq<BoundUnnestExpression>(unnest_get->types[i]);
unnest_expr->ChildMutable() =
    BoundCastExpression::AddArrayCastToList(context, proj.expressions[col_bind.column_index]->Copy());
```

The same cast is added in `ConvertCTETableInOutUnnest`
(`unnest_rewriter.cpp:291-293`), which does the same job for the CTE-domain
path of §13.6.4. `AddArrayCastToList` needs a `ClientContext`, which is why
`UnnestRewriter`'s constructor takes one (`unnest_rewriter.hpp`, and the
call site in §13.2). The test is
`test/sql/types/nested/array/array_lateral_unnest.test`:

```sql
CREATE TABLE m(id INT, fixed INT[2]);
INSERT INTO m VALUES (1, [1, 2]), (2, [3, 4]), (3, NULL);
SELECT * FROM m, UNNEST(m.fixed) AS s(v) ORDER BY ALL;
-- 1  [1, 2]  1
-- 1  [1, 2]  2
-- 2  [3, 4]  3
-- 2  [3, 4]  4
```

The file also covers a filter on `v`, a `LEFT JOIN ... ON true` (where the
`NULL` array gives one `NULL` row), and a `MATERIALIZED` CTE, which takes the
CTE-domain path. A plain `SELECT UNNEST(fixed) FROM m` is not at risk,
because it does not go through this rewriter.

## 13.7 `UNNEST_REWRITER` worked example: three real states

Using the first, simplest query in `unnest_rewriter.test_slow`
(line 10 of that file):

```sql
SELECT * FROM (VALUES ([1, 2, 3]), ([4, 5])) t(i), (SELECT UNNEST(i)) t2(j)
ORDER BY 1, 2;
```

**State 1 — immediately after `FlattenDependentJoins`.** `(SELECT UNNEST(i))`
is a scalar correlated subquery whose free reference is `t.i`. Per Chapter 6,
§6.7, its `LogicalDependentJoin` takes the `!perform_delim` path: the LHS gets
a synthetic `row_number()` window column, and the RHS's `UNNEST` reads its
argument through a `LogicalDelimGet` that re-exposes both the window's
`row_number` (the per-row key) and `t`'s own array column `i` (the value
actually being unnested) — `duplicate_eliminated_columns` therefore holds
*two* entries, matching `RewriteCandidate`'s own assertion
(`D_ASSERT(delim_get.chunk_types.size() > 1)`, `unnest_rewriter.cpp:676`) that
this shape's DelimGet always carries more than one column:

```
LogicalOrder                (t.i, j)
 └─ LogicalProjection          (t.i, j)                       <- UnnestRewriter's "candidate"
      └─ LogicalComparisonJoin    (LOGICAL_DELIM_JOIN, JoinType::INNER,
                                   duplicate_eliminated_columns = [row_number, i])
           ├─ LogicalWindow         (row_number() OVER () AS rn)
           │    └─ LogicalGet (t — VALUES ([1,2,3]), ([4,5]))
           └─ LogicalProjection      (j)
                └─ LogicalUnnest      (j := UNNEST(i))
                     └─ LogicalDelimGet  (chunk_types: [rn, i])
```

**State 2 — after `DELIMINATOR`.** `FindJoinWithDelimGet` walks the RHS
(`LogicalProjection → LogicalUnnest → LogicalDelimGet`) and increments
`delim_get_count` to 1 the moment it reaches the `LogicalDelimGet` leaf — but
it never finds a `LOGICAL_COMPARISON_JOIN` wrapping that DelimGet anywhere
along the way, because nothing here joins against it; the `LogicalUnnest`
reads from it directly as an ordinary operator child, not as one side of a
join. `candidate.joins` therefore stays empty, `candidate.joins.size() ==
candidate.delim_get_count` is `0 == 1` (false), and the type-downgrade branch
correctly does not fire — the DelimGet is genuinely still load-bearing at
this point, and `Deliminator` recognizes that it has no applicable pattern
here. `join_type` is `INNER`, not `SINGLE`, so `TrySwitchSingleToLeft` isn't
even attempted. **State 2 is byte-identical to State 1** — this is exactly
the "these solve different problems entirely" point Chapter 6, §6.7 already
made in prose, now shown as a concrete, traceable no-op rather than an
assertion.

**State 3 — after `UNNEST_REWRITER`.** The candidate (the outer
`LogicalProjection(t.i, j)`) matches `IsSupportedUnnestTop`; its single child
is the `LOGICAL_DELIM_JOIN`; that join is `INNER` with exactly one condition;
its LHS is a `LogicalWindow`; its RHS, walked through the one intervening
projection, bottoms out in a `LogicalUnnest` whose own child is a
`LogicalDelimGet` — pattern 1 (§13.6.1) matches exactly. `RewriteCandidate`
splices `LogicalGet(t)` (the window's own child) in as the `LogicalUnnest`'s
new child, replacing the DelimGet; the DelimJoin and the window operator are
dropped; and `UpdateRHSBindings` prepends `t`'s own exposed column (`i`) onto
the one surviving RHS projection, so the query's `SELECT *` still sees both
`t.i` and `j`:

```
LogicalOrder            (t.i, j)
 └─ LogicalProjection      (t.i, j)
      └─ LogicalUnnest       (j := UNNEST(t.i))
           └─ LogicalGet (t — VALUES ([1,2,3]), ([4,5]))
```

This is precisely what `unnest_rewriter.test_slow` asserts at line 75
(`<!REGEX>:(.*DELIM_JOIN.*)|(.*CTE Name:.*__duckdb_delim.*)`) — no trace of
either DelimJoin representation survives — and it is, semantically, exactly
the plan you would get from writing the same query directly as an ordinary
lateral cross join (`FROM t, UNNEST(t.i) AS t2(j)`) rather than through the
indirection of a correlated scalar subquery: `UnnestRewriter`'s whole job, in
this one query, was recovering that simpler shape from the more roundabout
one decorrelation is forced to produce in the general case.

## 13.8 Every case in `test/optimizer/unnest_rewriter.test_slow`

The file is organized in three deliberate movements, and is worth reading in
that order rather than line by line:

**Movement 1 (lines 9–66) — correctness first, with no `EXPLAIN` at all.**
Six queries of escalating shape, each checked only for the right *answer*:
a simple two-array cross join (line 10, walked in full in §13.7); the same
shape with extra, untouched LHS columns riding along (line 18 — this is
exactly what `GetLHSExpressions`/`UpdateRHSBindings`'s "prepend the LHS's own
columns" mechanism (§13.6.3) has to get right, now with more than one
column and a string literal payload column mixed in); a **doubly-nested**
`UNNEST` (line 27, `UNNEST(j)` over an already-unnested `j` produced by a
first `UNNEST(i)`) — this exercises `UnnestRewriter::FindCandidates`'s
bottom-up recursion directly, since the *inner* DelimJoin candidate has to be
rewritten before the outer one is even reachable in its final shape; a
three-stage CTE pipeline ending in `UNNEST(stage3.sub) → UNNEST(s1.sub.sub)`
(lines 36–53) — a considerably larger, more realistic query that stresses the
same nested-candidate recursion at real complexity, checked only by its
`count(*) = 1000` result; and a Parquet-backed, doubly-nested-`UNNEST`
analytics query over `hits`/`access.product` (lines 55–65) that is, in
substance, a compact version of a genuinely common "explode two levels of a
nested JSON/Parquet column, then aggregate" pattern.

**Movement 2 (lines 69–129) — the same six queries, now checked by `EXPLAIN`
for the DelimJoin's absence,** plus two negative controls that prove the
rewrite is *not* unconditional: line 122 (`SELECT (SELECT UNNEST(i)) FROM
(VALUES ([])) tbl(i)`) asserts the DelimJoin (or its CTE-domain
equivalent) is **retained**, with `SINGLE` still present — an empty-array
literal input still needs the general, uniqueness-guaranteeing machinery,
since nothing about `UnnestRewriter`'s pattern-matching depends on the
runtime cardinality of what gets unnested, only on the plan's static shape,
and this query's *shape* (a correlated scalar subquery around an `UNNEST`) is
identical whether the array is empty or not — meaning this negative control
is really confirming something else: that this specific query's LHS is not,
in fact, a `LogicalWindow` at all (a bare `VALUES` literal with no further
correlation-consuming operator above it doesn't need the row-number
scaffolding the way a genuinely tabular outer relation does), so
`FindCandidates`'s very first structural gate declines the match. Line 127
(`select * from (select [42,43,44]) t(a), (select unnest(t.a)) t2(b)`) is a
positive control confirming the rewrite *does* still apply to a
single-row-VALUES-producing subquery wrapped in an extra layer of
projection, contrasting directly with line 122's empty-list case.

**Movement 3 (lines 131–249) — five named historical GitHub issues, each
its own small regression.** Issue #7444 (lines 133–150): two independent
`UNNEST`s of two different array columns (`arr` and a derived
`generate_series`) inside the *same* subquery, checked for `NULL`-padding
correctness when the two arrays have different lengths (DuckDB's parallel-array
`UNNEST` semantics pad the shorter one with `NULL`) — this is a case where
`UnnestRewriter` has to correctly rewrite **two** `BOUND_UNNEST` expressions
inside the same `LogicalUnnest` node, not just one. Issue #7733 (lines
154–201): `LATERAL` joins, both with and without an explicit correlated
reference inside the `LATERAL` body, over two arrays of *mismatched* lengths
per row — the `NULL`-padding again has to survive the rewrite intact,
including for rows where one array is longer than the other. Issue #7792
(lines 205–215): `UNNEST(..., recursive := true)` over a list of structs,
checked for correct field-by-field expansion. The final two blocks
(lines 217–249) are a plain `CROSS JOIN UNNEST(arr)` and its `WITH
ORDINALITY` counterpart — the latter is asserted to **retain** the DelimJoin
(`<REGEX>:(.*DELIM_JOIN.*)|(.*CTE Name:.*__duckdb_delim.*)`), which is exactly
the `unnest_get_ref.ordinality_idx.IsValid()` guard from §13.6.1's pattern-2
walkthrough: an ordinality column (the row's position within its own array)
is itself sourced from the DelimGet's own machinery in a way the simple
column-forwarding rewrite cannot safely reproduce, so the pass declines
outright rather than risk losing or mis-numbering the ordinality values.

## 13.8.1 Case study: `unnest_rewriter_issue_14542.test` and the missing type check

The regression test itself is deceptively small — three `statement ok` blocks,
no result checking at all, just proving three related queries don't crash:

```sql
CREATE OR REPLACE MACRO json_each(input) AS
TABLE (
  SELECT
      CASE json_type(val::json)
      WHEN 'ARRAY' THEN unnest(RANGE(json_array_length(val::json)::bigint))::varchar
      ELSE unnest(json_keys(val::json))
      END AS key,
      json_extract(val::json, key) AS value
  FROM (SELECT input AS val)
);

FROM (SELECT '[1,2,3]' AS message) CROSS JOIN json_each(message);
FROM (SELECT '[1,2,3]'::json AS message) CROSS JOIN json_each(message);
FROM (SELECT '[1,2,3]'::json[] AS message) CROSS JOIN json_each(message);
```

**The bug, found via `git log`.** This checkout's history shows the exact
fix commit, `c962046f5d` ("fix #14542 and memory safety for UnnestRewriter"),
merged via `c31c46a875` ("Fix #14542 (#14610)"). The diff isolates the
precise correctness defect to a single condition inside `FindCandidates`.
Before the fix, the check that decides whether an `UNNEST` at the bottom of
the RHS chain is a genuine candidate was:

```cpp
// pre-fix
if (curr_op->get()->type == LogicalOperatorType::LOGICAL_UNNEST) {
	candidates.push_back(op_ptr);
}
```

— accepting **any** `LogicalUnnest` node found at that position, with no
check on what its own child actually was. The fix added exactly the check
this chapter's §13.6.1 already quoted as current behavior:

```cpp
// post-fix (current source, unnest_rewriter.cpp:165-169)
if (curr_op->get()->type == LogicalOperatorType::LOGICAL_UNNEST &&
    curr_op->get()->children[0]->type == LogicalOperatorType::LOGICAL_DELIM_GET) {
	candidates.push_back(op);
	return;
}
```

**Why this mattered.** `RewriteCandidate`'s own body, both before and after
the fix, unconditionally treats the `LogicalUnnest`'s child as a
`LogicalDelimGet` — reading `delim_get.chunk_types`, asserting its size, and
splicing the window's own LHS input directly into `unnest.children[0]` in its
place. Before the fix, `FindCandidates` could hand `RewriteCandidate` a
`LogicalUnnest` whose child was **not** a DelimGet at all — which, for the
`json_each` macro above, is exactly what happens: the macro's `TABLE(...)`
body is planned with its own internal `UNNEST` calls (`unnest(RANGE(...))`
inside one `CASE` branch, `unnest(json_keys(...))` inside the other), and
when this macro is cross-joined against an outer row, the plan can contain a
`LogicalUnnest` sitting in a position that superficially matches
`FindCandidates`'s walk (reached by descending through the RHS side of some
enclosing `LOGICAL_DELIM_JOIN` produced elsewhere in the plan for a different
reason) whose actual child is not the DelimGet the rewrite's own body
assumes — a plain scan, a projection over the macro's own internal
`CASE`-selected value, or some other operator entirely. `RewriteCandidate`'s
unconditional `.Cast<LogicalDelimGet>()` (a debug-checked cast in DuckDB) then
fails, surfacing as exactly the symptom the issue title names: an internal
error, not a wrong answer — the pass crashed rather than silently
mis-compiling, but it crashed on an entirely valid, syntactically ordinary
query that had no business reaching the optimizer's internals at all.

**The fix's shape, and the lesson.** The one-line type check the diff adds is
the whole fix: `FindCandidates` must verify the exact shape it is about to
hand to `RewriteCandidate`, not merely the shape of the node three levels
up the tree that first triggered the walk. This is a small, specific instance
of a general lesson worth stating plainly for a pattern-matching pass like
this one: a multi-step structural match (parent → DelimJoin → window-or-not →
chain-of-projections → target operator) is only as safe as its **last**
check, because that is the one `RewriteCandidate` actually depends on for
every field access it performs unconditionally afterward — every earlier
check in the chain narrows the search, but only the final one licenses the
unconditional cast the rewrite's body relies on. The same commit also
happened to fix a related, but conceptually separate, memory-safety issue in
the same function: `FindCandidates`'s and `RewriteCandidate`'s signatures
were changed from raw `unique_ptr<LogicalOperator> *` pointers to
`reference<unique_ptr<LogicalOperator>>` wrappers throughout, removing a
latent class of dangling-pointer risk if an earlier sibling rewrite ever
reallocated a parent's `children` vector out from under a raw pointer a
later step in the same pass still held — bundled into the same commit
because both defects surfaced from the same code review, not because they
were the same bug.

## 13.9 Summary

`DELIMINATOR` and `UNNEST_REWRITER` are both, in the sense §7.2 uses the word,
narrow specialist passes: each recognizes exactly one syntactic shape
`FlattenDependentJoins` is known to leave behind, and does nothing at all
when a plan doesn't contain that shape — as concretely demonstrated by
§13.5's third snapshot (`UNNEST_REWRITER` finding no `LOGICAL_DELIM_JOIN` left
to act on, because `DELIMINATOR` already retagged it) and §13.7's second
snapshot (`DELIMINATOR` finding no join-with-DelimGet to remove, because the
DelimGet feeds an `UNNEST` directly rather than through a join). Both passes'
correctness rests on the same underlying discipline: never delete a node
without either compensating for what it implicitly filtered
(`Deliminator`'s `IS NOT NULL` filter) or fully rewiring every remaining
reference to what it used to expose (`UnnestRewriter`'s three-step binding
update, `Deliminator`'s own `ColumnBindingReplacer` pass) — and both passes'
real regression history (`#23658`'s missing join-type-aware nullability
proof; `#14542`'s missing child-type check) is a story about exactly one of
those two disciplines being incompletely applied at the specific boundary a
test file now exists to hold in place. Chapter 8 picks up immediately
downstream of both passes: by the time `JOIN_ORDER` runs, whatever
`LOGICAL_DELIM_JOIN` structure survived `DELIMINATOR`'s cleanup is treated,
per §6.10's already-established distinction, as a single opaque relation in
the enclosing join hypergraph — cleanup this chapter's two passes performed
being exactly what determines how much (or how little) of that structure is
still there for Chapter 8's search to have to work around.
