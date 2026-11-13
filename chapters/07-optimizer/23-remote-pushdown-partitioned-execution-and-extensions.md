# 7.23 — Optimizer's Interfaces to the Outside World: `REMOTE_PUSHDOWN`, `PARTITIONED_EXECUTION`, and `EXTENSION`

*Sources: `src/optimizer/remote_pushdown_optimizer.cpp` +
`src/include/duckdb/optimizer/remote_pushdown_optimizer.hpp`;
`src/optimizer/partitioned_execution.cpp` +
`src/include/duckdb/optimizer/partitioned_execution.hpp`;
`src/include/duckdb/optimizer/optimizer_extension.hpp`. Registered
`OptimizerType`s: `REMOTE_PUSHDOWN` (invoked from `Optimizer::OptimizeStatement`,
`optimizer.cpp:217-227` — *not* one of the numbered passes in §7.4's list, see below),
`PARTITIONED_EXECUTION` (pass #41, `optimizer.cpp:537`), `EXTENSION` (two hook
points bracketing the whole pipeline, `optimizer.cpp:594` and `:606`). Tests:
`test/sql/catalog/comment_on_column_remote_catalog.test` (needs an out-of-tree
extension) and two C++ tests with a mock remote
catalog, `test/api/test_remote_pushdown_macro_arguments.cpp` and
`test/api/test_remote_pushdown_merge.cpp` (all assessed in §7.23.5); `test/optimizer/partitioned_execution_coverage.test`
and `test/optimizer/partitioned_execution.test_slow`; `test/extension/test_remote_optimizer.cpp`
and `test/extension/loadable_extension_optimizer_demo.cpp` for `EXTENSION`.*

## 7.23.1 What ties these three together, and why they share one file

Every other pass in this chapter takes a `LogicalOperator` tree that DuckDB
itself produced, from a query DuckDB itself is going to execute, and makes it
cheaper. These three passes are different in the same specific way: each one
is a point where the optimizer's authority stops being total. `REMOTE_PUSHDOWN`
decides whether a whole statement should stop being DuckDB's problem at all
and become some *other* system's problem. `PARTITIONED_EXECUTION` decides
whether a plan shape that would ordinarily need cross-thread coordination
(one shared hash table, one shared sorted run) can instead be sliced into
pieces that owe each other nothing. `EXTENSION` is the literal, documented
seam where code DuckDB's own developers didn't write gets to see the plan and
change it. Chapter 00's overview already flagged the first of these as an
architectural oddity worth a closer look — `REMOTE_PUSHDOWN` runs from
`Optimizer::OptimizeStatement`, a method separate from the ~45-pass
`RunBuiltInOptimizers()` sequence this chapter otherwise walks pass by pass —
and this file starts from that finding rather than re-deriving it, then goes
one level deeper into *why* the separation is structurally forced, not just
observed.

## 7.23.2 `REMOTE_PUSHDOWN`: what "remote" means here, precisely

One sentence: **`RemotePushdownOptimizer` decides whether an entire parsed
SQL statement refers to exactly one attached remote catalog and nothing
local, and if so, replaces the statement with a single opaque call into that
catalog's own query engine instead of letting DuckDB plan and execute it
locally.**

"Remote" here is a specific, narrow thing: a `Catalog` subclass — attached via
an ordinary `ATTACH '<uri>' AS <name>` — that opts in to
`RemoteCapability::EXECUTE_QUERY_NODE` and/or `EXECUTE_STATEMENT`
(`src/include/duckdb/catalog/catalog.hpp:89-105`):

```cpp
// src/include/duckdb/catalog/catalog.hpp:100-105
enum class RemoteCapability : uint8_t {
	IS_REMOTE,
	EXECUTE_QUERY_NODE,
	EXECUTE_STATEMENT,
	CONNECT,
};
```

The doc comment directly above it (lines 89-99) spells out what each value
unlocks: `IS_REMOTE` is a generic marker other engine code checks (e.g.
`DatabaseManager::GetRemoteCatalogCount()`, the very gate that decides whether
`RemotePushdownOptimizer` runs at all); `EXECUTE_QUERY_NODE` means the catalog
implements `RemoteExecute(ClientContext&, unique_ptr<QueryNode>)` and is
therefore eligible for `RemotePushdownOptimizer` to hand it a `SELECT`/DML
query body; `EXECUTE_STATEMENT` is the DDL analogue
(`RemoteExecute(ClientContext&, unique_ptr<SQLStatement>)`); `CONNECT` is a
sibling capability used by a completely different code path (`PhysicalConnect`,
the `CONNECT`/`DISCONNECT` chokepoint — see the aside at the end of §7.23.5)
that this optimizer pass does not touch.

This is a categorically different notion of "pushdown" from the two passes
that close out §7.4's numbered list, `TYPE_PUSHDOWN` (#44) and
`SCALAR_FN_PUSHDOWN` (#45). Those push a *cast* or a *scalar function call*
into a **file-format reader** (Parquet, CSV) so the reader can do slightly
more work per row it already has to touch — the query still runs entirely
inside this DuckDB process, against a file DuckDB itself is scanning.
`REMOTE_PUSHDOWN` pushes the **entire query** — the whole `QueryNode`, or the
whole `SQLStatement` for DDL — across a catalog boundary to a *different
query engine's* planner and executor, which might be another DuckDB process,
might be Postgres, might be anything a `Catalog` subclass wants it to be.
Nothing about predicates or projections specifically is pushed; what's pushed
is the statement, wholesale, once the optimizer has proven it's safe to ship
as one unit.

## 7.23.3 Why a separate method, not a step in `RunBuiltInOptimizers()`

The overview already established the mechanical fact: `OptimizeStatement`
takes `unique_ptr<SQLStatement>&`, not `unique_ptr<LogicalOperator>&`. The
reason isn't stylistic — it's that **at the point this pass has to run, no
`LogicalOperator` tree exists yet.** `Planner::CreatePlan`
(`src/planner/planner.cpp:301-304`) makes the ordering explicit:

```cpp
// src/planner/planner.cpp:301-304
void Planner::CreatePlan(unique_ptr<SQLStatement> statement) {
	D_ASSERT(statement);
	Optimizer optimizer(*binder, context);
	optimizer.OptimizeStatement(statement);
	// ... binder->Bind(...) happens in the switch below this call
```

`OptimizeStatement` runs *before* `CreatePlan`'s subsequent dispatch into
`binder->Bind(*statement)` for whichever statement type it is — before name
resolution, before type checking, before any `LogicalOperator` exists to hand
to `RunBuiltInOptimizers()`. A pass that needs to inspect the *parsed AST* —
`BaseTableRef`s with their raw, possibly catalog-qualified names;
`ColumnRefExpression`s that haven't been resolved to a `ColumnBinding` yet —
has no logical plan to operate on at this point in the pipeline, full stop.
It is not "could have been" a `LogicalOperator` pass and wasn't; it
*structurally cannot be* one, because the object that pass would need doesn't
exist until later. This is also why `RemotePushdownOptimizer::Rewrite`
recurses over `QueryNode`, `TableRef`, and `ParsedExpression` — the parser's
own tree types (`src/include/duckdb/optimizer/remote_pushdown_optimizer.hpp:18-43` forward-
declares `Binder`, `JoinRef`, `SubqueryRef`, `TableFunctionRef`, `QueryNode`,
never a single planner/binder type) — rather than `BoundColumnRefExpression`
or `LogicalGet`.

There is a second hook between this pass and the binder. The
`switch` that follows `OptimizeStatement` calls
`Planner::CreatePlan(SQLStatement &)`, and that method runs
*pre-bind planner extensions* just before `binder->Bind`:

```cpp
// src/planner/planner.cpp:182-189
static void RunPreBindExtensions(ClientContext &context, Binder &binder, SQLStatement &statement) {
	for (auto &planner_extension : PlannerExtension::Iterate(context)) {
		if (planner_extension.pre_bind_function) {
			PlannerExtensionInput input {context, binder, planner_extension.planner_info.get()};
			planner_extension.pre_bind_function(input, statement);
		}
	}
}
```

```cpp
// src/planner/planner.cpp:209-213
		auto binding_timer = profiler.StartTimer<MetricPlannerBindingTime>();
		binder->SetParameters(bound_parameters);
		binder->SetStatementParameters(statement.named_param_map);
		RunPreBindExtensions(context, *binder, statement);
		auto bound_statement = binder->Bind(statement);
```

The hook is a field on `PlannerExtension`
(`src/include/duckdb/planner/planner_extension.hpp:36`, `:45-48`):

```cpp
// src/include/duckdb/planner/planner_extension.hpp:45-48
	//! The pre-bind function of the planner extension.
	//! Takes the statement that is about to be bound. This runs right before the binder binds the statement,
	//! allowing e.g. a catalog to start loading the entries the statement references.
	pre_bind_function_t pre_bind_function = nullptr;
```

So for one statement the order is:

```
parsed SQLStatement
   │
   ├─ Optimizer::OptimizeStatement      REMOTE_PUSHDOWN (may replace the whole
   │                                    statement with SELECT * FROM <remote ref>)
   ├─ RunPreBindExtensions              extension sees the (possibly rewritten)
   │                                    parsed statement; may start loading entries
   ├─ binder->Bind                      first LogicalOperator tree appears
   ├─ RunPostBindExtensions
   └─ Optimizer::Optimize               EXTENSION (pre) → built-in passes → EXTENSION (post)
```

Why a pre-bind hook? Binding looks tables up one at a time. For a remote
catalog, each lookup is a network round trip. With the hook, a remote
catalog can walk the parsed statement, collect the table names, and start
loading them all together before binding begins. Core DuckDB does not
collect the table names itself; an extension that wants to prefetch walks
the parsed statement on its own. No code in this checkout's `src/` or
`test/` sets `pre_bind_function`; its users are out-of-tree remote-catalog
extensions.

The gate itself, right at the top of `OptimizeStatement`
(`optimizer.cpp:217-227`):

```cpp
// src/optimizer/optimizer.cpp:217-227
void Optimizer::OptimizeStatement(unique_ptr<SQLStatement> &statement) {
	if (!Settings::Get<EnableOptimizerSetting>(context)) {
		return;
	}
	if (DatabaseManager::Get(context).GetRemoteCatalogCount() > 0) {
		// if we have any remote catalogs attached then pushdown into remote
		RunOptimizer(OptimizerType::REMOTE_PUSHDOWN, [&]() {
			RemotePushdownOptimizer optimizer(binder);
			optimizer.Rewrite(statement);
		});
	}
}
```

Two things worth noting precisely. First, this still respects
`PRAGMA disable_optimizer` (the same `EnableOptimizerSetting` check every
other pass respects), so turning the optimizer off entirely also turns this
one off, even though it lives outside `RunBuiltInOptimizers()`. Second, it's
wrapped in the same `RunOptimizer(OptimizerType::REMOTE_PUSHDOWN, ...)` call
every built-in pass uses, so it gets the same profiler timer and the same
`disabled_optimizers` toggle — `REMOTE_PUSHDOWN` is a real, listed
`OptimizerType` in `src/common/enum_util.cpp:4424`, so
`SET disabled_optimizers = 'remote_pushdown'` works exactly like disabling any
of the ~45 passes this chapter otherwise numbers. It's outside the pipeline's
*sequence*, not outside its *toggle/timer infrastructure*.

`RunOptimizer` itself is short. It is the wrapper every pass goes through,
this one included:

```cpp
// src/optimizer/optimizer.cpp:137-151
void Optimizer::RunOptimizer(OptimizerType type, const std::function<void()> &callback) {
	context.InterruptCheck();

	if (OptimizerDisabled(type)) {
		// optimizer is marked as disabled: skip
		return;
	}
	auto &profiler = QueryProfiler::Get(context);
	{
		auto optimizer_timer = profiler.StartTimerInternal("optimizer." + StringUtil::Lower(EnumUtil::ToString(type)));
		callback();
	}
	if (plan) {
		Verify(*plan);
	}
}
```

Two details of this wrapper are worth a note. First, the interrupt check
at the top is `context.InterruptCheck()`. The binder makes the same check
while it expands macros and views. So a `max_execution_time` limit can stop
a query that is still being planned.
`test/sql/settings/max_execution_time_nested_expansion.test` uses a macro
nested 24 deep and expects "Query exceeded maximum execution time".
Second, `OptimizerDisabled` does not read `disabled_optimizers` directly.
It calls `DBConfig::IsOptimizerDisabled` (`src/main/config.cpp:1016-1019`),
which takes a mutex, `disabled_optimizers_lock`. The reason: one connection
can run `SET GLOBAL disabled_optimizers = ...` while other connections are
planning queries, and reading a `std::set` while another thread changes it
is a data race. `test/sql/settings/concurrent_set_disabled_optimizers.test`
runs six concurrent loops that switch the setting on and off while they
plan correlated subqueries. For `REMOTE_PUSHDOWN` this means the toggle is
safe to flip from another connection.

There's a cheap, always-paid cost worth being honest about: the
`GetRemoteCatalogCount() > 0` check runs on *every* statement in a session
that has *any* remote catalog attached — not just ones that look like they
reference it. If you `ATTACH 'quack:host' AS qk` and then run a query that
never mentions `qk`, `RemotePushdownOptimizer::Rewrite` still walks the whole
statement to prove that (Case 2 in `Rewrite(BaseTableRef&)`, §7.23.4, is
precisely the "no catalog named, is there exactly one remote one in the
search path" check). This is why the gate is `GetRemoteCatalogCount() > 0`
rather than something statement-shape-specific: cheaply skip the whole
mechanism for the overwhelmingly common case (no remote catalog attached at
all), then pay the walk once a remote catalog exists in the session.

## 7.23.4 The rewrite: a three-valued lattice over the whole parsed tree

The core data structure is `CatalogPushdownResult`
(`remote_pushdown_optimizer.hpp:44-58`), which classifies any parsed
construct — an expression, a table reference, a whole query node — into one
of three states (`CatalogReferenceType`, same file, line 44):

- `NO_CATALOG_REFERENCED` — this construct touches no table or catalog object
  at all (a literal, a scalar function of literals, an unqualified CTE that
  turned out to resolve to nothing catalog-backed).
- `SINGLE_REMOTE_CATALOG` — every table/entry this construct touches lives in
  exactly one specific remote catalog.
- `UNKNOWN_CATALOG_REFERENCE` — this construct touches something local, or
  something whose remote-eligibility can't be proven; pushdown is vetoed for
  anything this taints.

`Merge` (`remote_pushdown_optimizer.cpp:165-215`) is the join operator over
this lattice, and its rules are worth stating exactly because they're not
symmetric in the way "AND-like" combinators usually are:

- `NO_CATALOG` ⊔ `NO_CATALOG` = `NO_CATALOG` (union the tracked reference
  lists — a literal joined with another literal is still catalog-free).
  The union appends `b`'s lists onto `a` in place and returns `a`. `Merge`
  and all its callers move results instead of copying them. This keeps
  folding a long list linear: a `VALUES` list with 50,000 rows merges 50,000
  times, and copying everything gathered so far at each merge would make it
  quadratic.
- `NO_CATALOG` ⊔ `SINGLE_REMOTE` = `SINGLE_REMOTE`, **conditionally**: the
  remote catalog gets asked, via `Catalog::SupportsPushdown(...)` overloads
  (`catalog.hpp:427-430`, default implementation returns `true` for all four —
  `catalog.cpp:383-397`), whether it can actually accept every no-catalog
  construct that was mixed in (an expression, a table construct, a query
  node). If it says no for even one, the whole merge collapses to `UNKNOWN`.
  This is the hook a real remote-catalog implementation uses to veto pushing
  down, say, a DuckDB-specific function the remote engine doesn't understand,
  without the optimizer needing to know anything function-specific itself.
- `SINGLE_REMOTE` ⊔ `SINGLE_REMOTE` = `SINGLE_REMOTE` only if both sides name
  the *same* catalog object; two different remote catalogs in one statement
  merge to `UNKNOWN` (there is no cross-remote-catalog federation here — one
  statement pushes to at most one remote).
- Anything touching `UNKNOWN` merges to `UNKNOWN`. This is the veto that
  propagates: one local table reference anywhere in a query poisons the whole
  merge chain back up to the statement root.

`Rewrite(BaseTableRef&)` (`remote_pushdown_optimizer.cpp:999-1054`) is where
a concrete table reference gets classified, and it has three cases worth
walking because they show the tri-state lattice isn't just theoretical
plumbing — it's answering real ambiguity in SQL syntax. **Case 0**: an
unqualified name might be a CTE reference, checked first via `RefersToCTE`,
because a bare `foo` in `FROM foo` could shadow a remote catalog's real table
named `foo`. **Case 1**: an explicitly catalog-qualified name (`qk.foo`) is
resolved straight to that catalog and checked against
`RemoteCapability::EXECUTE_QUERY_NODE`; if the catalog doesn't support it, or
the table doesn't actually exist there, the reference is tracked as local
(`TrackLocalTable`) and the result is `UNKNOWN` — not `NO_CATALOG` — precisely
so `Merge(SINGLE_REMOTE, UNKNOWN)` collapses correctly instead of treating a
failed remote lookup as catalog-neutral. **Case 2**: an unqualified name with
no matching CTE is checked against every catalog in the current search path;
pushdown is only attempted if *exactly one* remote catalog sits in that
search path (`pushdown_state.remote_catalogs_in_search_path.size() != 1`
blocks it) and the name doesn't also resolve in any local catalog on the
search path (`EntryExistsInLocalCatalog`) — an unqualified name that could
plausibly mean either the local or the remote table is deliberately
conservative and blocks pushdown rather than guessing.

Once a whole statement (or, recursively, a whole `QueryNode` — a materialized
CTE's own body gets analyzed by a child `RemotePushdownOptimizer`, line
`remote_pushdown_optimizer.cpp:263`) resolves to `SINGLE_REMOTE_CATALOG`,
`FinishPushdown` (`remote_pushdown_optimizer.cpp:1847-1868`) does the actual
surgery:

```cpp
// src/optimizer/remote_pushdown_optimizer.cpp:1847-1857
void RemotePushdownOptimizer::FinishPushdown(unique_ptr<SQLStatement> &statement, CatalogPushdownResult result) {
	if (result.reference_type != CatalogReferenceType::SINGLE_REMOTE_CATALOG) {
		return;
	}
	// Strip the catalog name so the remote server doesn't recursively re-push
	StripCatalogName(*statement, result.catalog->GetName());
	auto node = GetNodeFromStatement(*statement);
	if (node) {
		statement = WrapRemoteRef(CreateRemoteFunctionRef(result, std::move(node)));
		return;
	}
```

Three moves, in order: (1) every explicit `qk.`-style qualifier naming this
same catalog is stripped from the tree (`StripCatalogName`, walking every
`TableRef`/`ParsedExpression`/`CreateInfo`/`AlterInfo` variant — the
`comment_on_column_remote_catalog.test` name literally documents that this
walk has to touch `COMMENT ON COLUMN` targets too) — the remote side has its
own `main` schema and would otherwise try to re-resolve (and potentially
re-push) a catalog name that, from its point of view, refers to itself; (2)
`CreateRemoteFunctionRef` calls the one virtual hook a remote catalog
implementation actually has to provide,
`result.catalog->RemoteExecute(context, std::move(node))`
(`remote_pushdown_optimizer.cpp:1298-1301`), handing over the *entire*
(catalog-name-stripped) `QueryNode` and getting back an opaque `TableRef` —
in practice, a table function call the remote catalog registered, which will
stream the remote engine's own result set back as ordinary DuckDB rows; (3)
`WrapRemoteRef` (`remote_pushdown_optimizer.cpp:1838-1845`) wraps that
opaque ref in the smallest possible local statement, `SELECT * FROM <ref>`,
which is what DuckDB's own binder, optimizer, and executor then plan and run
completely normally — from that point on, the rest of the pipeline sees an
ordinary table-function scan and has no idea the rows behind it came from a
socket instead of a Parquet file.

The default `Catalog::RemoteExecute` overloads all `throw
NotImplementedException` (`catalog.cpp:371-381`) — there is no built-in
remote catalog in this checkout's core `src/`; every catalog you can attach
without a third-party extension answers `Supports(RemoteCapability::...)`
with the base class's `false` (`catalog.hpp:420-422`) and is therefore never
eligible for this pass to touch at all.

### Constant folding before shipping, and the macro-argument bug

One more step happens during the walk. When an expression uses no catalog
at all (for example `1 + 1`, or `list_value(1, 2)`), the pass evaluates it
locally and puts the result in the tree as a constant. The remote engine
then gets a plain literal and does not need to know DuckDB's functions.
`TryConstantFold` does this. After it folds, it gives the new constant a
name:

```cpp
// src/optimizer/remote_pushdown_optimizer.cpp:1187-1189
	auto folded = ConstantExpression::FromValue(fold_result);
	// preserve the name DuckDB would generate for the original expression
	folded->SetAlias(expr->GetAlias().empty() ? Identifier(expr->ToString()) : expr->GetAlias());
```

The name is there so that `SELECT [1, 2]` still shows a column called
`list_value(1, 2)`, the same name it has without a remote catalog. But in some places DuckDB reads an
alias on an argument as the *name* of a named argument. A table function
argument is one such place, and the code already guarded it: only an alias
the user wrote may survive (`RewriteTableFunctionArgument`,
`remote_pushdown_optimizer.cpp:921-930`).

Macro calls were the other place. Take this built-in macro call, with any
remote-capable catalog attached:

```sql
SELECT generate_subscripts([1, 2], 1);
```

The pass folds `[1, 2]` and names the constant `list_value(1, 2)`. A macro
binder that treated *any* argument with a non-empty alias as a named
argument would see a named argument `list_value(1, 2)` followed by a
positional argument `1`, and fail with "positional argument following named
argument". This bug was reported against the Postgres extension
(duckdb/duckdb-postgres#630), which attaches a catalog with `IS_REMOTE`.

The fix lives in the binder, not in the optimizer.
`MacroFunction::BindMacroFunction` asks the argument whether the *user* gave
it a name (`arg.HasName()`), instead of looking at the expression's alias:

```cpp
// src/function/macro_function.cpp:84-91
		if (arg.HasName()) {
			// Default argument
			if (named_arguments.find(arg.GetName()) != named_arguments.end()) {
				return MacroBindResult(
				    StringUtil::Format("Macro %s() has named argument repeated %s", name, arg.GetName()));
			}
			named_arg_types.insert(arg.GetName(), std::move(arg_type));
			named_arguments[arg.GetName()] = std::move(arg.GetExpressionMutable());
```

So `remote_pushdown_optimizer.cpp` can keep the generated alias on folded
constants. `test/api/test_remote_pushdown_macro_arguments.cpp` checks this
(see §7.23.5).

## 7.23.5 Test coverage: thin, in three files

Three tests cover this pass: one `.test` file, described further below,
and two Catch2 tests under `test/api/` that need no out-of-tree extension.
The two Catch2 tests use
the same trick: a `StorageExtension` whose `attach` callback returns a
subclass of `DuckCatalog` (the normal local catalog) that *claims* remote
capabilities. That is enough to switch `RemotePushdownOptimizer` on.

- `test/api/test_remote_pushdown_macro_arguments.cpp` returns only
  `IS_REMOTE` from `Supports`. The pass runs (the gate counts `IS_REMOTE`
  catalogs) but can never push anything. The test checks that folded macro
  arguments stay positional: `generate_subscripts([1, 2], 1)` returns
  `1, 2`; a user macro `add_default(a, b := 5)` called as
  `add_default(1 + 1, 2 * 3)` returns `8`; and
  `DESCRIBE SELECT [1, 2]` still names the column `list_value(1, 2)`.
- `test/api/test_remote_pushdown_merge.cpp` returns `IS_REMOTE` and
  `EXECUTE_QUERY_NODE`. Its `RemoteExecute` records the pushed node and
  answers with a single row `'pushed'`. Its `SupportsPushdown(expression)`
  records each expression it is asked about and rejects any that contains
  `map`. It is the in-tree test of the `Merge` lattice from
  §7.23.4:

  ```sql
  -- test_remote_pushdown_merge.cpp, first TEST_CASE
  SELECT count(*) FROM s.tbl, (VALUES (1, NULL), (2, NULL), (3, NULL), (4, MAP {1: 2})) t(x, y)
  -- a MAP cell vetoes pushdown: runs locally, returns 20
  SELECT count(*) FROM s.tbl, (VALUES (1, NULL), (2, NULL), (3, NULL), (4, 5)) t(x, y)
  -- pushed: returns 'pushed'; each cell 1..5 was offered to
  -- SupportsPushdown exactly once, NULL three times
  ```

  It then checks the same rule for a select list (`SELECT 1, 2, MAP {1: 2},
  (SELECT count(*) FROM s.tbl)` stays local), a multi-row
  `INSERT INTO s.tbl VALUES ...` (pushed), and that after `DETACH` nothing
  is pushed. A second `TEST_CASE`, hidden by the `[.]` tag so it does not
  run by default because it measures time, plans a 50,000-row `VALUES`
  query and requires planning with the fake remote catalog attached to take
  less than `3 × local + 1` seconds. That guards the linear `Merge` from
  §7.23.4.

The rest of this section describes the `.test` file.

Searching this checkout's `test/` tree for "remote pushdown" by name or
content finds the `.test` file
`test/sql/catalog/comment_on_column_remote_catalog.test`, the macro test
above, and `test/api/CMakeLists.txt`. The `.test` file's own
description line says what it's actually testing: *"`COMMENT ON COLUMN`
while a remote catalog is attached (the remote pushdown optimizer inspects
every statement)."* It attaches a real remote catalog —

```sql
-- test/sql/catalog/comment_on_column_remote_catalog.test
require quack
require httpfs

statement ok
CALL quack_serve(token='test_token');

statement ok
ATTACH 'quack:localhost' AS qk (TOKEN 'test_token');
```

— then runs a sequence of purely **local** `COMMENT ON COLUMN` statements
(unqualified, `memory.main`-qualified, and a nonexistent-table error case) and
checks they still work correctly. It is not, in any real sense, a test of the
pushdown *rewrite itself*: it never attaches a table that lives in `qk` and
checks that a query against it got shipped there; it only proves that having
a remote catalog attached (and therefore `RemotePushdownOptimizer` running on
every statement, per §7.23.3's gate) doesn't break an unrelated, entirely
local DDL statement. Outside the two `test/api/` files, a search of
`test/` finds no `test/optimizer/remote_pushdown*.test`. There is no test
with two remote catalogs in one query, and none where a correlated
subquery blocks pushdown through `TrackLocalTable`. The merge test does
cover a positive pushdown (a `SELECT` and an `INSERT` that get shipped) and
a `NO_CATALOG` expression vetoed by `SupportsPushdown`. So the honest
summary: **coverage is shallow, but it exercises the mechanism itself, not
only an interaction bug.**

Part of why is structural, not neglect: `quack`, the extension both this test
and `test/sql/connect/connect_quack.test` `require`, is not vendored in this
checkout at all — `.github/config/extensions/quack.cmake` pulls it from a
*separate* GitHub repository (`duckdb/duckdb-quack`) at a pinned commit,
exactly the way `.github/config/extensions/*.cmake` pulls in any other
out-of-tree extension. The catalog implementation that actually overrides
`Supports(RemoteCapability::EXECUTE_QUERY_NODE)` and `RemoteExecute` —
the concrete thing `RemotePushdownOptimizer` was built to drive — lives in
that separate repository and wasn't fetched or inspected as part of writing
this book (per its scope statement in Chapter 0: "the specific
extension-loading mechanism ... only the specific optimizer/parser
extension hooks that matter to the core pipeline" — this is exactly the line
where the book's own scope stops). What's confirmed from this checkout alone
is the mechanism's shape and its default (`NotImplementedException`-throwing)
behavior, not `quack`'s specific remote-execution semantics.

One adjacent, easy-to-conflate mechanism worth a one-line disambiguation:
`test/sql/connect/connect_quack.test` exercises `RemoteCapability::CONNECT`,
which routes an entire *session* (via `CONNECT qk;` / `DISCONNECT;`) through
a completely different chokepoint — `PhysicalConnect`
(`src/execution/operator/helper/physical_connect.cpp`) — rewriting every
subsequent statement as `SELECT * FROM <fn>('qk', '<raw SQL text>')`. That is
a session-level routing decision, not an optimizer pass, and it is not
`RemotePushdownOptimizer`'s code path even though it serves a related goal
(get a query to a remote catalog's own engine). The two mechanisms share a
`Catalog` and a `RemoteCapability` enum, not a code path.

## 7.23.6 `PARTITIONED_EXECUTION`: purpose, and a correction to the overview's own guess

One sentence: **for a `GROUP BY`, `ORDER BY`, or `PARTITION BY`-only window
operator sitting directly (through simple projections/filters) above a base
table scan, `PartitionedExecution` uses the scan's own column statistics to
carve the operator's input into disjoint value ranges, physically duplicates
the operator (and everything beneath it) once per range with a range filter
injected, and unions the duplicates back together — turning one operator
that would need shared cross-thread state into several that need none.**

§7.4's numbered list, written from `00-overview.md`, ventured a specific
guess about what this pass is *for*: "this is where, for example, the
logical groundwork for a `PhysicalPartitionedAggregate` ... gets laid down."
Reading `partitioned_execution.cpp` directly shows this guess is not quite
right, and it's worth being precise about exactly how, because Chapter 14
already documents the *actual* `PhysicalPartitionedAggregate` eligibility
check in detail (`chapters/14-memory-and-aggregation.md`, §14.9), and the two
mechanisms turn out to be genuine siblings — same family of table-function
capability, different callback, different decision point, different
question being answered.

Chapter 14 §14.9 traces `CanUsePartitionedAggregate` /
`PhysicalPlanGenerator::HasSingleValuePartitions`
(`src/execution/physical_plan/plan_aggregate.cpp:32-46, 94-122`), which asks
the scan's table function for `TablePartitionInfo` via
`table_function.get_partition_info` — a single enum answer
(`NOT_PARTITIONED` / `SINGLE_VALUE_PARTITIONS` / `OVERLAPPING_PARTITIONS` /
`DISJOINT_PARTITIONS`, `src/include/duckdb/function/partition_stats.hpp:20-25`)
about whether the data *already on disk* is laid out so that each grouping
key's rows arrive contiguously (the canonical case: Hive-partitioned Parquet,
one directory per key value). That check runs in the **physical plan
generator** — after this chapter's entire optimizer pipeline has already
finished — and it decides between `PhysicalHashAggregate` and
`PhysicalPartitionedAggregate` for a `LogicalAggregate` the optimizer produced
unchanged.

`partitioned_execution.cpp` calls a *different* table-function callback,
`get_partition_stats` (`table_function_get_partition_stats_t`,
`table_function.hpp:415-416`), which returns a `vector<PartitionStatistics>`
— per-partition row counts and, optionally, a `PartitionRowGroup` handle for
fetching that partition's own column min/max
(`partition_stats.hpp:29-48`). This pass doesn't ask "is the data already
partitioned by the grouping key" — it asks "given whatever row groups or
files this scan is already divided into, and their column statistics, can I
*invent* a small number of disjoint value ranges over some column this
operator cares about, wide enough to be worth splitting on?" It runs
**inside the optimizer**, as `OptimizerType::PARTITIONED_EXECUTION`
(`optimizer.cpp:537`), operates on the `LogicalOperator` tree directly, and
never produces or references a `PhysicalPartitionedAggregate` — its output is
an ordinary `LogicalAggregate`/`LogicalOrder`/`LogicalWindow`, just N copies
of it unioned together, each with a narrower input. Whether the physical
plan generator subsequently picks `PhysicalHashAggregate` or
`PhysicalPartitionedAggregate` *for each of those N copies* is a wholly
separate, later decision Chapter 14 already covers and this pass doesn't
influence one way or the other.

So: related infrastructure (both live off a scan's ability to describe its
own partitioning), but **not** the mechanism that decides
`PhysicalPartitionedAggregate` eligibility, and this chapter's own earlier
speculation about it is corrected here rather than repeated. What
`PartitionedExecution` actually is: a **logical-plan-level, statistics-driven
manual range-partitioning rewrite** — the optimizer manufacturing
parallelism-friendly structure that may not have existed in the source data
at all, rather than detecting structure that already did.

## 7.23.7 Eligibility mechanics: which operators, which columns, which stats

`PartitionedExecution::Optimize` (`partitioned_execution.cpp:536-580`) walks
the tree depth-first and, at every operator, runs a chain of increasingly
expensive checks that bail out the instant one fails:

```cpp
// src/optimizer/partitioned_execution.cpp:536-559
void PartitionedExecution::Optimize(unique_ptr<LogicalOperator> &op) {
	for (auto &child : op->children) {
		Optimize(child); // Depth-first
	}
	if (op->EstimateCardinality(optimizer.context) < PartitionedExecutionConfig::MINIMUM_RESULT_CARDINALITY) {
		return; // Too small
	}
	vector<PartitionedExecutionColumn> columns;
	if (!PartitionedExecutionGetColumns(*op, columns) || columns.empty()) {
		return; // Unable to get partition columns from this operator
	}
	optional_ptr<LogicalGet> get = PartitionedExecutionTraceColumns(*op, columns);
	if (!get || columns.empty()) {
		return; // Unable to trace any binding down to scan
	}
```

**Gate 1 — cardinality, and a subtlety worth naming precisely.**
`MINIMUM_RESULT_CARDINALITY = 4194304` (2²²) is compared against
`op->EstimateCardinality(...)` — the estimated **output** row count of the
`GROUP BY`/`ORDER BY`/window operator itself, not the row count of the table
underneath it. An `ORDER BY` never reduces row count, so a 5-million-row
scan's `LogicalOrder` clears the gate easily. A `GROUP BY` over a
low-cardinality key (say, ten distinct values) does *not* clear it even over
a 10-million-row table, because the *aggregate's own output* is estimated at
ten rows — splitting a ten-row result across partitions has nothing to gain.
This is exactly what `partitioned_execution.test_slow` documents in its own
comment: *"no tests for single-column GROUP BY here because the optimization
doesn't trigger due to low estimated cardinality"* — and it's why the
coverage test's multi-column case (`GROUP BY i, recurring`, where `i` alone
has ~5 million distinct values) is the one that actually clears the gate.

**Gate 2 — is this operator shape even eligible, and which of its columns
qualify.** `PartitionedExecutionGetColumns`
(`partitioned_execution.cpp:47-88`) accepts exactly three operator types,
each with its own narrower rule:

- `LOGICAL_AGGREGATE_AND_GROUP_BY`: rejected outright if there's more than
  one grouping set, any grouping function (`GROUPING(...)`), or no groups at
  all (`agg.grouping_sets.size() > 1 || !agg.grouping_functions.empty() ||
  agg.groups.empty()`) — a plain `GROUP BY` is fine, `ROLLUP`/`CUBE`/multiple
  `GROUPING SETS` are not. Every group expression that's a bare
  `BOUND_COLUMN_REF` becomes a candidate column; anything else (an
  expression like `GROUP BY x + 1`) is simply skipped, not disqualifying.
- `LOGICAL_ORDER_BY`: candidate columns are collected **only as a prefix run**
  of bare column refs — the loop `break`s at the first `ORDER BY` entry that
  isn't a plain column reference, because order matters here and a
  non-colref entry can't be range-split at all, let alone anything after it.
- `LOGICAL_WINDOW`: every window expression in the operator must share
  *equivalent* partitions (`BoundWindowExpression::PartitionsAreEquivalent`)
  — `ROW_NUMBER() OVER (PARTITION BY i)` alongside `RANK() OVER (PARTITION BY
  i)` in the same projection is fine (they partition identically and can
  split together); mixed `PARTITION BY` clauses across window expressions in
  one operator are not. Only the (single, shared) partition list's bare
  column refs become candidates — an `ORDER BY`/frame inside the window is
  irrelevant to this check because this pass only ever targets the
  partition boundary, never the ordering within a partition.

**Gate 3 — can a candidate column's binding actually be traced down to a
scan.** `PartitionedExecutionTraceColumns`
(`partitioned_execution.cpp:105-158`) walks straight down from the target
operator's single child, through any chain of `LOGICAL_PROJECTION` (rewriting
each candidate's `ColumnBinding` to the projection's underlying expression,
and dropping the candidate if that expression isn't itself a bare colref —
`PartitionedExecutionHandleColumnRemoval`, lines 90-103, which for
`ORDER BY` drops everything from that point *and everything after it* in the
prefix, since a dropped middle column invalidates the ordering of what
follows it) and `LOGICAL_FILTER` (bindings pass through filters unchanged),
bailing out entirely (returning `nullptr`) at anything else — a join, a
second child, table in/out functions. It stops only at a bare
`LOGICAL_GET` with no children of its own, and even then only survives if
that scan's table function exposes both a statistics callback
(`get.function.statistics` or `.statistics_extended`) **and**
`get.function.get_partition_stats` — a scan that can't report either is a dead
end regardless of how promising the operator above it looked. `rowid`, the
virtual row-id "column," specifically fails here: `TryGetStorageIndex`
returns false for it because it has no on-disk storage index to fetch
statistics for (the coverage test's `PARTITION BY rowid` case, line 96,
exercises exactly this).

**Gate 4 — global statistics, before spending anything on per-partition
detail.** `PartitionedExecutionHandleGlobalStats`
(`partitioned_execution.cpp:202-220`) fetches whole-column stats once per
surviving candidate and applies `PartitionedExecutionCanUseStats`
(lines 160-181): no stats at all, or a column that `CanHaveNull()`, is
disqualifying outright (the coverage test's `nullable_i` case) — this pass
never has to reason about NULL placement relative to a range boundary
because it simply refuses any column where NULLs are possible. Only
`NUMERIC_STATS` and non-`BLOB` `STRING_STATS` are supported (the coverage
test's `constant_blob` case is rejected purely for being a `BLOB`, with a
comment admitting the omission is deliberate: *"Let's not mess with BLOB for
now"*); and for `VARCHAR`, the min/max bytes themselves have to be valid
UTF-8 (`Value::StringIsValid`) — a truncated multi-byte sequence at a
row-group boundary would otherwise produce an invalid string as a filter
bound, which `partitioned_execution.test_slow`'s final case
(`test_non_utf8`, lines 306-321) specifically constructs to guard against,
mixing plain zero-padded numeric-string keys with keys prefixed by
multi-byte Japanese characters. Separately, `PartitionedExecutionMinMaxEqual`
drops any column whose global min equals its global max — a truly constant
column (the coverage test's `constant_col`/`constant_varchar` cases) can
never produce more than one range, so it's removed before any per-partition
work is spent on it at all.

## 7.23.8 Computing ranges, and rewriting the plan

Surviving candidates go into `PartitionedExecutionComputeRanges`
(`partitioned_execution.cpp:324-434`), capped at
`PartitionedExecutionConfig::MAXIMUM_COLUMNS = 3` columns so a wide
multi-column `GROUP BY`/`PARTITION BY` doesn't force fetching statistics for
an unbounded number of columns. For each candidate, every partition's
min/max becomes a pair of "sweep line" events (`PartitionedExecutionStatsNode`
— a value plus a signed count delta, `+count` at the partition's start,
`-count` at its end, `partitioned_execution.cpp:222-257`), sorted by value.
Walking that sorted event list left to right and accumulating the running
sum turns "how many row groups overlap at this point" into a single running
counter (`current_overlap`), which is exactly what lets
`PartitionedExecutionGrowPartition` (lines 297-315) build up a candidate
partition boundary by three successive growth rules, in order: grow until
the partition has accumulated at least
`num_threads * MIN_ROW_GROUPS_PER_THREAD_PER_PARTITION` (32) row groups'
worth of rows; then keep growing while the overlap-to-partition-size ratio
exceeds `MAX_OVERLAP_RATIO` (0.1) — because a boundary drawn through a region
where many row groups' value ranges overlap would force each generated
partition's filter to still touch most of the table's row groups, defeating
the purpose; then a bounded lookahead (at most half of one more
`min_partition_count`'s worth) to see whether growing slightly further finds
a strictly lower-overlap cut point nearby, and finally a small top-up so the
very last partition in the sweep isn't left as an oddly tiny remainder. The
column whose sweep produces the **most** ranges is the one actually used to
split (`max_col_idx`, line 423) — more ranges means finer-grained, more
independent parallel work — and for a `DESC`-ordered `ORDER BY` candidate,
the chosen ranges are reversed (line 429-431) so the partitions, once
unioned, come out in the query's actual requested order.

`PartitionedExecutionSplitPipeline` (`partitioned_execution.cpp:452-534`) then
does the actual rewrite, once per computed range: **deep-copy** the entire
operator (`LogicalOperatorDeepCopy::DeepCopy`), bailing out of the *whole*
optimization for this operator if any single operator in the subtree refuses
to be deep-copied (`catch (NotImplementedException&) { return; }` —
correctness trumps opportunism here); build a `LogicalFilter` with a
`>=`-lower-bound and `<`-upper-bound comparison against the range's min/max
(only the bounds that actually exist — the first and last partition are
open-ended); splice that filter directly under the copy; and then **reuse
`FilterPushdown`** (pass #6, §7.4) on the copy so the freshly injected range
filter gets pushed all the way down to the scan exactly the way any ordinary
user-written `WHERE` clause would be. If pushdown proves the injected range
is incompatible with filters already present on that copy (the scan
collapses to `LOGICAL_EMPTY_RESULT`), that copy is simply dropped — a range
that can never match anything just doesn't get a union arm. Each surviving
copy's scan gets its estimated cardinality set directly from the computed
partition's row count (`SetEstimatedCardinality`), feeding the cost model
that runs *after* this pass (`STATISTICS_PROPAGATION`, `PROJECTION_PLACEMENT`
already ran earlier at #34/#35, but downstream physical-plan costing still
consults these cardinalities). If fewer than two copies survive, the whole
rewrite is abandoned (`children.size() < 2`) — there's no point unioning one
partition with itself.

The union itself is built with a detail easy to miss but load-bearing for
correctness: `setop_all = true` (these ranges are disjoint by construction,
so no deduplication is wanted or needed), and `allow_out_of_order =
(op->type != LOGICAL_ORDER_BY)`
(`src/include/duckdb/planner/operator/logical_set_operation.hpp:16-17`,
call site `partitioned_execution.cpp:522-524`). For a `GROUP BY` or window
split, the union's arms may be executed and interleaved in any order — the
physical union operator is free to run them however parallelism dictates.
For an `ORDER BY` split specifically, `allow_out_of_order` is `false`,
forcing the physical union to emit each child's rows **in the order the
children were constructed**, which is exactly the range order
`PartitionedExecutionComputeRanges` already produced (ascending, or reversed
for `DESC`). That single boolean is the entire reason a range-partitioned
`ORDER BY` still comes out globally sorted: each partition is independently
sorted over a disjoint, already-ordered value range, and the union simply
concatenates them in the right sequence instead of merging or interleaving.
Finally, `ColumnBindingReplacer` patches every reference to the original
operator's column bindings, tree-wide above the rewrite point, onto the new
`LogicalSetOperation`'s bindings — the same binding-patching idiom
Chapter 14's unused-columns/column-lifetime passes use elsewhere in this
pipeline.

## 7.23.9 `partitioned_execution_coverage.test`: the fast eligibility matrix

This file (`test/optimizer/partitioned_execution_coverage.test`, 122 lines)
is explicitly a **gate-by-gate** test, not a correctness test — it sets
`threads = 1` up front specifically because, per its own comment, *"the
optimizer's split size depends on the number of active threads"*, and
pinning threads to one keeps the required row count (5 million, per the
table it builds) small enough to run quickly while still comfortably
clearing `MINIMUM_RESULT_CARDINALITY`. Every case is a single `EXPLAIN`
checked against a `<REGEX>:.*UNION.*` (rewrite fired) or
`<!REGEX>:.*UNION.*` (it didn't), and each maps onto one specific gate from
§7.23.7 traced above:

- `ORDER BY i ASC` / `DESC` and `PARTITION BY i` window: baseline positive
  cases — one clean high-cardinality integer column, rewrite fires both
  ways.
- `GROUP BY i, recurring`: multi-column grouping keeps the estimated output
  cardinality above the gate (§7.23.7's Gate 1 subtlety) even though
  `recurring` alone wouldn't.
- `GROUPING SETS ((i, recurring))` with `grouping(i)`: rejected — a
  grouping function disqualifies the operator outright at
  `PartitionedExecutionGetColumns`'s first check.
- `row_number() ... rank() ... PARTITION BY i`: two window expressions,
  identical partition list — `PartitionsAreEquivalent` accepts them and both
  ride the same split.
- `PARTITION BY nullable_i`: rejected — `CanHaveNull()` disqualifies at
  Gate 4.
- `PARTITION BY constant_col`: rejected — global min == max, dropped before
  ranges are even attempted.
- `PARTITION BY constant_varchar`, `recurring_varchar`, `constant_blob`:
  three distinct rejection reasons exercised back to back — a genuinely
  constant `VARCHAR` (min==max), a `BLOB` (unsupported stats type
  categorically), and (implicitly, via the *ranges* stage rather than the
  global-stats stage) a low-cardinality recurring `VARCHAR` whose row-group
  ranges overlap too heavily to produce a useful split.
- `PARTITION BY rowid`: rejected — no storage index for a virtual column
  (Gate 3).
- `PARTITION BY recurring`: rejected — only ten distinct values recurring
  throughout the table means every row group's min/max range overlaps every
  other's almost entirely; `MAX_OVERLAP_RATIO` is never satisfied and no
  partition boundary is found.
- `PARTITION BY recurring, i`: accepted — multi-column candidate list, `i`
  (the higher-cardinality column) is the one whose sweep produces more
  ranges and gets selected as `max_col_idx`.
- `ORDER BY i, recurring`: accepted, but only `i` is actually eligible as a
  split column — this exercises the "prefix run of bare colrefs" rule for
  `ORDER BY`, since both entries here *are* bare colrefs so the whole prefix
  survives to Gate 3/4, but only `i` clears the statistics gates.
- `ORDER BY i + 1`: rejected — the sort key isn't a bare column reference,
  so `PartitionedExecutionGetColumns`'s `LOGICAL_ORDER_BY` branch `break`s
  immediately and no candidate columns exist at all.

## 7.23.10 `partitioned_execution.test_slow`: proving the rewrite doesn't lose or reorder a single row

Where the coverage test asks "did the rewrite fire," this file
(`test/optimizer/partitioned_execution.test_slow`, 321 lines, over a
10-million-row synthetic table with five differently-shaped columns —
low/high cardinality, ordered/recurring, and one constant) asks "is the
rewritten plan's *result* actually identical to what an unpartitioned
execution would have produced." It leans on the paired-label
`query I nosort <label>` idiom this book's sibling file
(`03-expression-rewriter-strings-and-regex.md`, "A quick note on `nosort
<label>` blocks") already introduces: two blocks sharing a label are checked
byte-for-byte against **each other**, not against a literal value written in
the file. Here that idiom proves something slightly different from a
same-shape-EXPLAIN rewrite check — it proves a **row-level invariant across
the whole rewrite**, computed once early in the file and re-checked after
every subsequent variant of the query:

- **`h`** (line 22): `bit_xor(hash(*COLUMNS('^t_')))` over the whole table —
  an order-*agnostic* reduction (XOR is commutative/associative), computed
  once before any partitioned rewrite runs anywhere near this query, then
  recomputed after every `ORDER BY`/`GROUP BY`/window variant later in the
  file that triggers the rewrite. If the range split ever caused a row to be
  scanned twice (an off-by-one in a boundary comparison) or dropped
  entirely (a range computed with a gap), this XOR would change — it can't
  be fooled by the union simply emitting rows in a different order, which is
  exactly the failure mode range-partitioning could introduce and ordinary
  row-count assertions wouldn't catch.
- **`asc_h` / `desc_h`** (lines 28-36): `hash(list(test ORDER BY ... ASC))`
  and the `DESC` counterpart — an order-*preserving* reduction (`list(...)`
  over an explicit order, then hashed as a single value), which specifically
  proves the `allow_out_of_order = false` mechanism from §7.23.8 actually
  holds: that unioning several independently range-filtered, independently
  sorted partitions back together in the constructed range order reproduces
  the exact same total order a single unpartitioned sort would have
  produced, not merely the same *set* of rows.

The `foreach c ...` / `foreach order_type ASC DESC` loop (lines 48-72)
exercises three different column shapes individually — low-cardinality
ordered, high-cardinality ordered, high-cardinality recurring — each
compared against the same `h` label both ways, then the file moves to
genuinely multi-column cases the coverage test only gestures at: `ORDER BY
t_const, t_high_cardinality_recurring, t_low_cardinality_recurring` (three
columns, only the middle one is actually usable as the split column — the
constant one is dropped, the low-cardinality-recurring one would overlap too
much on its own, but the file's comment admits the gap explicitly: *"we can
split by `t_high_cardinality_recurring` in the following tests, but not by
the other columns — we can still do the optimization!"*), a `GROUP BY ALL`
followed by `unnest`-ing the grouped lists back out (proving the rewrite
survives being embedded inside a CTE chain, not just a bare top-level query),
and — the most exacting correctness check in the file (lines 189-260) — a
`lag()` window function applied *after* the `ORDER BY` rewrite, asserting
`count(*)` is exactly `0` for every row where the previous row's sort key
came out greater than the current row's: a direct, row-by-row proof that
global order survived the range-split-and-reunion, phrased as "no row is
ever preceded by something that sorts after it," in both `ASC` and `DESC`
form. The file closes with the truncated-multi-byte-UTF-8 regression
(`test_non_utf8`, already described in §7.23.7's Gate 4 discussion) — a
targeted crash/incorrect-bound regression test rather than a general
correctness check, added specifically because a `VARCHAR` min/max computed
naively from truncated byte ranges could otherwise produce a filter bound
that isn't valid UTF-8 at all.

## 7.23.11 `EXTENSION`: the `OptimizerExtension` mechanism, precisely

One sentence: **`OptimizerExtension` is DuckDB's registered hook for
third-party C++ code — typically a loadable extension — to install its own
function that sees (and may rewrite in place) the entire logical plan, at
one of exactly two fixed points relative to the ~45 built-in passes.**

The interface (`src/include/duckdb/optimizer/optimizer_extension.hpp:20-53`)
is deliberately minimal:

```cpp
// src/include/duckdb/optimizer/optimizer_extension.hpp:26-53
struct OptimizerExtensionInput {
	ClientContext &context;
	Optimizer &optimizer;
	optional_ptr<OptimizerExtensionInfo> info;
};

typedef void (*optimize_function_t)(OptimizerExtensionInput &input, unique_ptr<LogicalOperator> &plan);
typedef void (*pre_optimize_function_t)(OptimizerExtensionInput &input, unique_ptr<LogicalOperator> &plan);

class OptimizerExtension {
public:
	optimize_function_t optimize_function = nullptr;
	pre_optimize_function_t pre_optimize_function = nullptr;
	shared_ptr<OptimizerExtensionInfo> optimizer_info;
	static void Register(DBConfig &config, OptimizerExtension extension);
	// ...
};
```

An extension registers with `OptimizerExtension::Register(config, ...)` at
load time and supplies either or both function pointers; `optimizer_info` is
an arbitrary, extension-owned payload (subclassing the empty
`OptimizerExtensionInfo` base) threaded through to every invocation, so an
extension's hook function can close over its own state without any global or
thread-local variable. Both hooks receive the **same** signature — a mutable
reference to the plan's root `unique_ptr<LogicalOperator>`, which the hook
is free to replace wholesale (not just mutate the tree it points at).

The two call sites bracket `RunBuiltInOptimizers()` exactly as
`00-overview.md` already found (`optimizer.cpp:594-612`):

```cpp
// src/optimizer/optimizer.cpp:594-612
for (auto &pre_optimizer_extension : OptimizerExtension::Iterate(context)) {
	RunOptimizer(OptimizerType::EXTENSION, [&]() {
		OptimizerExtensionInput input {GetContext(), *this, pre_optimizer_extension.optimizer_info.get()};
		if (pre_optimizer_extension.pre_optimize_function) {
			CTEFilterPusher::ClearDependencies(*plan);
			pre_optimizer_extension.pre_optimize_function(input, plan);
		}
	});
}

RunBuiltInOptimizers();

for (auto &optimizer_extension : OptimizerExtension::Iterate(context)) {
	RunOptimizer(OptimizerType::EXTENSION, [&]() {
		OptimizerExtensionInput input {GetContext(), *this, optimizer_extension.optimizer_info.get()};
		if (optimizer_extension.optimize_function) {
			optimizer_extension.optimize_function(input, plan);
		}
	});
}
```

What's actually available to reason about at each point follows directly
from where it sits: a **pre-optimize** hook sees the plan right after
`LowerMandatoryAggregateRewrites` (§7.2's mandatory `ROLLUP`/`CUBE` lowering)
but before a single one of the ~45 heuristic passes has touched it — still
shaped essentially the way the binder produced it (Chapter 5), still
carrying whatever redundant filters, un-pushed-down predicates, and
unpruned columns the binder never tries to remove. A pre-optimize extension
is the right place for a rewrite that wants to inject or restructure
*before* DuckDB's own passes get a chance to make assumptions about the
shape it's handed — for instance, wrapping part of the plan in a marker
operator so a later built-in pass treats it specially, or performing a
domain-specific rewrite that's easier to express before column pruning has
already renumbered every binding. A **post-optimize** hook, by contrast, sees
the plan in its *cheapest form DuckDB itself could produce* — every filter
already pushed as far down as it can go, every join already reordered, every
unused column already pruned, every runtime join filter already wired up.
That is exactly the right (and, in a sense, the *only* sensible) point for
a decision like "is this plan, in its final shape, worth handing to a
completely different execution engine instead" — which is precisely what the
one real in-tree example does (§7.23.12).

Two structural details worth flagging precisely, both visible only by
reading the two loops side by side. First, both hook points share the
*same* `OptimizerType::EXTENSION` value for their `RunOptimizer(...)` call —
there is no separate `PRE_EXTENSION`/`POST_EXTENSION` type. This means
`SET disabled_optimizers = 'extension'` and the profiler's
`optimizer.extension` timer bucket cannot distinguish pre- from
post-optimize extension time or toggle one independently of the other; a
`disabled_optimizers` entry for `extension` turns off *every* registered
extension's *both* hooks at once, for every extension currently loaded, not
just one. Second, the pre-optimize loop calls
`CTEFilterPusher::ClearDependencies(*plan)`
(`optimizer.cpp:598`) immediately before invoking each extension's
`pre_optimize_function` — defensive cleanup of CTE-filter-pushdown
dependency bookkeeping specifically because an extension is about to be
allowed to restructure the tree in ways the built-in `CTE_FILTER_PUSHER` pass
(#7, which hasn't run yet at this point) can't anticipate — and the
post-optimize loop has no equivalent call, because by that point
`CTE_FILTER_PUSHER` has already run and finished its own bookkeeping.

## 7.23.12 A real example: the "Waggle" demo extension and its round-trip test

There is no production, shipped DuckDB extension in this checkout that
registers an `OptimizerExtension` — a repo-wide search for
`OptimizerExtension` outside the core headers turns up exactly two files,
both under `test/extension/`. That's not nothing, though: one of them is a
complete, working demonstration of exactly what a post-optimize hook can do,
and the other is a genuine (if unusually theatrical) integration test that
exercises it end to end over a real TCP socket and a forked child process.

`test/extension/loadable_extension_optimizer_demo.cpp` defines
`WaggleExtension : public OptimizerExtension`
(`loadable_extension_optimizer_demo.cpp:30-147`), registering only an
`optimize_function` (no `pre_optimize_function`) — a deliberate choice, given
what it does. `WaggleOptimizeFunction` first checks, recursively, whether the
**fully-optimized** plan contains a `parquet_scan`
(`HasParquetScan`, lines 36-47); if not, it returns immediately and changes
nothing. If it does, it opens a raw TCP socket to a host/port pulled from two
custom extension settings the demo registers for itself
(`waggle_location_host`, `waggle_location_port`, via
`config.AddExtensionOption(...)`, lines 161-162), **serializes the entire
optimized `LogicalOperator` tree** with `BinarySerializer` and ships it
across the socket, reads back a stream of serialized `DataChunk`s, and
replaces the local plan's root outright with a `LogicalColumnDataGet` wrapping
the results it just received (lines 103-146). From that point, DuckDB's own
physical-plan generator and executor see nothing but a pre-materialized
result set — the entire rest of the query, as far as this DuckDB process is
concerned, already happened somewhere else.

`test/extension/test_remote_optimizer.cpp` is the Catch2 integration test
that actually exercises this, and it's worth describing concretely because
it makes the mechanism's real-world shape unmistakable: it `fork()`s a child
process that opens a listening socket, loads the Parquet extension into its
own fresh in-memory `DuckDB` instance, and enters a loop that deserializes
whatever `LogicalOperator` tree arrives, wraps it in a
`LogicalPlanStatement`, executes it as an ordinary query against its own
database, and serializes the resulting `DataChunk`s back over the same
socket (lines 80-116). The parent process loads `parquet`, loads the
`loadable_extension_optimizer_demo` extension, sets
`waggle_location_host='127.0.0.1'` / `waggle_location_port=4242`, and runs
one query:

```sql
-- test/extension/test_remote_optimizer.cpp:147-148
SELECT first_name FROM PARQUET_SCAN('data/parquet-testing/userdata1.parquet') GROUP BY first_name
```

By the time `WaggleOptimizeFunction` sees this plan (post-optimize — every
built-in pass has already run), it's a fully-optimized `LogicalAggregate`
over a `LogicalGet` with a `parquet_scan` table function, `HasParquetScan`
fires, the whole tree gets shipped to the child process's own DuckDB
instance, executed there, and the results come back to replace the parent's
plan. The parent process never runs the aggregate itself — it runs
`LogicalColumnDataGet` over data another engine (in this test, coincidentally
also a `DuckDB`, but architecturally interchangeable with anything able to
deserialize a `LogicalOperator` and execute it) already computed. The
Catch2 test above is the only place where the hook actually rewrites a
plan, and it does so over a real process boundary and a real socket.

There is also a SQL logic test for the same extension,
`test/extension/test_loadable_optimizer.test`. It requires only
`notwindows` (the demo extension is not built on Windows). It is a smoke
test: it `LOAD`s the demo extension and runs `select 42`. That plan has no
`parquet_scan`, so `WaggleOptimizeFunction` returns without touching it.
The test proves the extension loads and registers its hook, not that the
hook ships a plan. (The demo's `HasParquetScan` reads the function name
with `get.function.GetName()`.)

This one example demonstrates the whole point of the two-hook-point design
from §7.23.11 concretely: the demo registers **only** `optimize_function`
(post), specifically because its decision — "is this plan worth shipping to
another engine" — depends on seeing the plan already reduced to its cheapest
DuckDB-producible form (fewer columns, pushed-down filters, a settled join
order) before deciding it's worth the round-trip cost of shipping it
somewhere else entirely.

## 7.23.13 Why all three live in one file

Every other pass in this chapter answers "how do I make DuckDB's own plan,
for DuckDB's own execution engine, cheaper." These three answer a different
question each: *should this query even run here* (`REMOTE_PUSHDOWN`),
*can this operator's cross-thread coordination be avoided entirely by
manufacturing structure the optimizer itself invents*
(`PARTITIONED_EXECUTION`), and *does code DuckDB's own authors didn't write
get a say at all* (`EXTENSION`). None of the three is a heuristic rewrite of
the kind the other ~45 passes are — each is a **seam**: a place where the
optimizer's authority over the plan is deliberately handed off, to a remote
catalog's own engine, to a manufactured set of independent execution
partitions, or to arbitrary third-party code. That's the throughline that
justifies grouping them, and it's also exactly why each one's test coverage
looks the way it does: `REMOTE_PUSHDOWN` and `EXTENSION` are thin in this
checkout precisely because exercising them fully requires an actual external
system (a real remote catalog extension, a real second process) that lives
outside `test/`'s ordinary SQL logic test harness — `PARTITIONED_EXECUTION`,
which needs nothing external, is correspondingly the one of the three with a
real, exacting test suite of its own.
