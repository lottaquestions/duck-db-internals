# Chapter 5 — The Logical Plan

Chapter 4 covered the binder as a *name-resolution* machine: how `orders`
becomes a `TableCatalogEntry`, how `o.custkey` becomes a `BoundColumnRefExpression`
carrying a `ColumnBinding`, how an ambiguous column reference gets a
Levenshtein-distance-ranked "did you mean" suggestion. This chapter covers the
other thing the binder produces as the literal same byproduct of that
name-resolution work (recall Difference #1 from Chapter 1: DuckDB does not have
a separate "bound but unplanned" intermediate form) — the **logical plan**
itself: the tree of `LogicalOperator` nodes that is `BoundStatement::plan`.

## 5.1 What a logical plan is, and what it deliberately is not

If you've seen relational algebra before — a query written as
`π_{custkey,total}(γ_{custkey; SUM(extendedprice)}(σ_{shipdate<date}(orders ⋈ lineitem)))`,
read right to left — a DuckDB logical plan will look extremely familiar,
because it's built out of the same conceptual pieces: a *scan* over a base
relation, a *selection* (`σ`, "keep rows matching a predicate" — DuckDB calls
this `LogicalFilter`), a *projection* (`π`, "compute this list of output
columns" — `LogicalProjection`), a *join* (`⋈`), and a *grouping/aggregation*
(`γ` — `LogicalAggregate`). The tree shape is the same idea too: each operator
is a node with zero or more child operators, each child operator's output rows
flow upward into its parent, and the root of the tree is the whole query's
result.

The one word worth dwelling on before anything else is *declarative*. A
`LogicalComparisonJoin` node says "the rows of my left child and my right child
should be combined wherever `o.orderkey = l.orderkey` holds" — full stop. It
does not say *how* that combination should be computed. It does not choose
between a hash join (build a hash table on one side, probe with the other), a
sort-merge join, a nested-loop join, or an index join. That decision — which
concrete *algorithm* implements the abstract relational operation — is
deliberately deferred, first to the optimizer (Chapter 7, specifically the
join-order and join-implementation passes covered in depth in Chapter 8), and
finally made concrete by the physical plan generator (Chapter 9), which is the
step that actually manufactures a `PhysicalHashJoin` (or `PhysicalPiecewiseMergeJoin`,
or `PhysicalIEJoin`, or `PhysicalNestedLoopJoin`) *object*. Every `LogicalOperator`
subclass in this chapter has exactly this property: it names *what* is
computed, never *how*. That is the entire reason a logical-plan chapter and a
physical-plan chapter (9) are different chapters at all — if the logical layer
already committed to an algorithm, there would be nothing left for the
optimizer to choose between.

One place this shows up concretely, and is worth internalizing early: DuckDB's
logical-operator vocabulary is considerably larger than the five or six
classical relational-algebra operators. Section 5.3 below lists 67 operator
kinds (not counting `LOGICAL_INVALID`). That's not `LogicalOperator` failing to stay "purely relational" —
it's because `BoundStatement::plan` isn't only asked to represent `SELECT`
queries. The same tree type represents `INSERT`, `CREATE TABLE`, `ALTER TABLE`,
`ATTACH`, window functions, recursive CTEs, and `PRAGMA` statements, none of
which relational algebra was ever meant to describe. `LogicalOperator` is DuckDB's
single "everything a bound SQL statement can be" representation, of which
classical relational algebra is the well-behaved `SELECT`-only core.

## 5.2 The base class: `LogicalOperator`

Every one of those operator classes derives from one class. Its full public
interface, read directly from the header:

```cpp
// src/include/duckdb/planner/logical_operator.hpp:33-106
//! LogicalOperator is the base class of the logical operators present in the
//! logical query tree
class LogicalOperator {
	friend class LogicalPlanVerifier;

public:
	explicit LogicalOperator(LogicalOperatorType type);
	LogicalOperator(LogicalOperatorType type, vector<unique_ptr<Expression>> expressions);
	virtual ~LogicalOperator();

	//! The type of the logical operator
	LogicalOperatorType type;
	//! The set of children of the operator
	vector<unique_ptr<LogicalOperator>> children;
	//! The set of expressions contained within the operator, if any
	vector<unique_ptr<Expression>> expressions;
	//! The types returned by this logical operator. Set by calling LogicalOperator::ResolveTypes.
	vector<LogicalType> types;
	//! Estimated Cardinality
	idx_t estimated_cardinality;
	bool has_estimated_cardinality;

public:
	virtual LogicalPlanSQLExportResult ToSQL(LogicalPlanSQLExportContext &context,
	                                         const LogicalPlanVerificationPath &path);
	virtual vector<ColumnBinding> GetColumnBindings();
	virtual TableIndex GetRootIndex();
	static string ColumnBindingsToString(const vector<ColumnBinding> &bindings);
	void PrintColumnBindings();
	static vector<ColumnBinding> GenerateColumnBindings(TableIndex table_idx, idx_t column_count);
	static vector<LogicalType> MapTypes(const vector<LogicalType> &types,
	                                    const vector<ProjectionIndex> &projection_map);
	static vector<ColumnBinding> MapBindings(const vector<ColumnBinding> &types,
	                                         const vector<ProjectionIndex> &projection_map);

	//! Resolve the types of the logical operator and its children
	void ResolveOperatorTypes();

	//! Returns true if this operator or any of its descendants has side effects
	//! Used to prevent inlining or elimination of side-effecting CTEs.
	bool HasSideEffects() const;
	//! Returns true if this operator or any of its descendants contains a volatile expression
	bool HasVolatileExpressions() const;

	virtual string GetName() const;
	virtual InsertionOrderPreservingMap<string> ParamsToString() const;
	virtual string ToString(optional_ptr<ClientContext> context = nullptr,
	                        const ProfilerPrintFormat &format = ProfilerPrintFormat::Default()) const;
	DUCKDB_API void Print();
	//! Debug method: verify that the integrity of expressions & child nodes are maintained
	virtual void Verify(ClientContext &context);

	void AddChild(unique_ptr<LogicalOperator> child);
	virtual idx_t EstimateCardinality(ClientContext &context);
	void SetEstimatedCardinality(idx_t _estimated_cardinality);
	void SetParamsEstimatedCardinality(InsertionOrderPreservingMap<string> &result) const;

	virtual void Serialize(Serializer &serializer) const;
	static unique_ptr<LogicalOperator> Deserialize(Deserializer &deserializer);

	virtual unique_ptr<LogicalOperator> Copy(ClientContext &context) const;

	virtual bool RequireOptimizer() const {
		return true;
	}

	//! Allows LogicalOperators to opt out of serialization
	virtual bool SupportSerialization() const {
		return true;
	};

	virtual bool HasProjectionMap() const {
		return false;
	}
	// ...
};
```

That's a small class — five data members, and roughly twenty methods, most of
them virtual. (The first method, `ToSQL`, is covered in §5.5.4.) Worth walking through each piece, because every subclass
described later in this chapter is defined entirely by *which of these it
overrides* and *what it puts in these five fields*.

### 5.2.1 The tree-shape fields: `type`, `children`, `expressions`

`type` is the `LogicalOperatorType` enum value (§5.3) — the tag that lets code
recover, from a bare `LogicalOperator&`, which concrete subclass it's actually
looking at, without a `dynamic_cast`. `children` is what makes this a *tree*
rather than a flat list: `vector<unique_ptr<LogicalOperator>>`, almost always
of size 0 (leaf nodes — scans), 1 (unary operators — filter, projection,
aggregate, order, limit), or 2 (binary operators — joins, set operations).
`expressions` is the payload: *what this operator computes*, expressed as a
list of bound `Expression` trees. What "expressions" means varies by operator
— for `LogicalFilter` it's the (one, post-`SplitPredicates` possibly many)
predicate(s) to test; for `LogicalProjection` it's the output column list; for
`LogicalAggregate` it holds the aggregate function calls; some operators
(`LogicalGet`, `LogicalCrossProduct`) legitimately have an empty `expressions`
vector because their entire behavior lives in other, subclass-specific fields
instead.

### 5.2.2 `types` and `ResolveOperatorTypes`

`types` is the resolved output schema of this node — one `LogicalType` per
output column, in the same order `GetColumnBindings()` returns them. It starts
out empty and is filled in by calling `ResolveOperatorTypes()`, whose
implementation is a small, clean piece of recursion:

```cpp
// src/planner/logical_operator.cpp:133-141
void LogicalOperator::ResolveOperatorTypes() {
	types.clear();
	// first resolve child types
	for (auto &child : children) {
		child->ResolveOperatorTypes();
	}
	// now resolve the types for this operator
	ResolveTypes();
	D_ASSERT(types.size() == GetColumnBindings().size());
}
```

Bottom-up: children resolve their own types first, then the parent's
(protected, subclass-overridden) `ResolveTypes()` looks at its children's
already-resolved `types` to compute its own — a `LogicalFilter`'s output types
are just its child's output types unchanged (a filter drops rows, not
columns), while a `LogicalProjection`'s output types are whatever its
`expressions` list evaluates to. The assertion at the end —
`types.size() == GetColumnBindings().size()` — is the load-bearing invariant of
the whole logical layer: every operator must report exactly as many resolved
types as it reports column bindings, in the same order, because downstream
consumers zip the two together positionally.

### 5.2.3 `estimated_cardinality`

`estimated_cardinality` (plus its `has_estimated_cardinality` guard flag) is a
single `idx_t`: this node's estimated output row count. It is not filled in by
the binder — at the moment a `LogicalOperator` is first constructed, this
field is meaningless. It becomes meaningful once the optimizer's
statistics-propagation pass runs (Chapter 7) and, more importantly, once the
join-order optimizer (Chapter 8) starts using these numbers as its cost model:
the entire reason DuckDB can decide "join `lineitem` to `orders` before
joining either to `customer`" is that every candidate join order is scored
using exactly this field, propagated bottom-up. The default implementation,
absent anything smarter, is refreshingly modest:

```cpp
// src/planner/logical_operator.cpp:261-272
idx_t LogicalOperator::EstimateCardinality(ClientContext &context) {
	// simple estimator, just take the max of the children
	if (has_estimated_cardinality) {
		return estimated_cardinality;
	}
	idx_t max_cardinality = 0;
	for (auto &child : children) {
		max_cardinality = MaxValue(child->EstimateCardinality(context), max_cardinality);
	}
	has_estimated_cardinality = true;
	estimated_cardinality = max_cardinality;
	return estimated_cardinality;
}
```

"Take the max over my children's estimates" is a conservative fallback used by
operators that don't know how to do better (a `LogicalFilter`, absent
statistics, doesn't know its own selectivity). `LogicalGet` overrides this with
an estimate from the table's actual row count or the table function's own
cardinality hint; `LogicalAggregate` overrides it with a number derived from
`GROUP BY` column distinct-value estimates (also `EstimateCardinality`
override — see `src/include/duckdb/planner/operator/logical_aggregate.hpp`).
The memoization (`has_estimated_cardinality`) matters because this method gets
called repeatedly by different optimizer passes that all want the same answer
without repeating the (sometimes expensive) statistics work.

### 5.2.4 The virtual methods

Six of these deserve individual comment, because they're each the entry point
for a different downstream consumer of the logical plan:

- **`GetColumnBindings()`** returns this operator's output columns as
  `ColumnBinding{table_index, column_index}` pairs — the exact currency
  Chapter 4 introduced. This is the method every other piece of plan-tree
  machinery calls to answer "what columns does this subtree actually produce,
  and by what identity can a parent refer to them?" The default
  implementation is deliberately a degenerate placeholder —
  `{ColumnBinding(TableIndex(0), ProjectionIndex(0))}`, i.e. "I produce one
  column, called column 0 of table 0" (`src/planner/logical_operator.cpp:33-35`)
  — every operator that produces a real, non-trivial output schema overrides
  it. `TableIndex` and `ProjectionIndex` are themselves not raw `idx_t`s but
  small newtype wrapper structs (`src/include/duckdb/common/table_index.hpp`,
  `src/include/duckdb/common/projection_index.hpp`) — a deliberate,
  cheap-at-compile-time guard against silently mixing up "which table" and
  "which column" integers, both of which are `idx_t` underneath.
- **`ToString`/`ParamsToString`** are what `EXPLAIN` actually calls.
  `ToString` produces the operator's node label (e.g. `"FILTER"`,
  `"HASH_JOIN"`); `ParamsToString` (an `InsertionOrderPreservingMap<string>` —
  ordered, so the same plan always prints identically) supplies the indented
  detail lines underneath each node — the actual predicate text, the join
  condition, the `GROUP BY` expression list. Every subclass that has anything
  interesting to show overrides `ParamsToString`; `LogicalComparisonJoin`,
  `LogicalAggregate`, `LogicalOrder`, `LogicalAnyJoin`, and
  `LogicalRecursiveCTE` all do (confirmed directly in their headers under
  `src/include/duckdb/planner/operator/`).
- **`Serialize`/`Deserialize`** are the plan (de)serialization pair used for
  prepared statements (a prepared statement's plan is serialized once, then
  deserialized and re-executed on each `EXECUTE`) and for `EXPLAIN (FORMAT
  json)`. This is precisely the mechanism that motivates the deliberate
  numeric gaps in `LogicalOperatorType` discussed in §5.3 — a serialized plan
  embeds the numeric enum tag, and that tag has to keep meaning the same thing
  release over release.
- **`Copy`** produces a deep copy of the subtree. Its default implementation is
  a small piece of practical cleverness: rather than hand-writing a
  field-by-field deep-copy for every subclass, it just round-trips
  the operator through its own `Serialize`/`Deserialize` machinery via an
  in-memory `MemoryStream` (`src/planner/logical_operator.cpp`, `Copy`'s body
  — "serialize this node into memory, then deserialize a fresh copy back out
  of it"). The reason `Copy` needs to exist at all: a CTE's body can be
  planned into more than one place in the final tree (once per reference to
  the CTE, in the non-materialized case — see §5.4.12), and each reference
  needs its own independent operator objects, not a tree with the same
  `LogicalOperator` pointer aliased in twice.
- **`RequireOptimizer()`** defaults to `true`. `Planner::Optimize` skips the
  optimizer pipeline when it returns `false`. Only two operators override it.
  `LogicalPrepare` (`logical_prepare.hpp:49-54`) returns `false` when the
  prepared statement still has unbound parameters or must be re-bound on
  every run, because optimizing now would be wasted work. `LogicalExecute`
  (`logical_execute.hpp:34-37`) returns `false` when it has no child plan:
  then the prepared plan is run as it is, and it was already optimized when
  it was prepared. This means each `EXECUTE` of a prepared statement does
  not run the full optimizer again over an already-optimized plan. Utility
  operators like `LOGICAL_PRAGMA` and `LOGICAL_SET` do not override it; they
  have no relational structure, so the passes simply find nothing to do.

## 5.3 The enum: `LogicalOperatorType`

Every `LogicalOperator::type` field holds one value from this enum:

```cpp
// src/include/duckdb/common/enums/logical_operator_type.hpp:18-118
enum class LogicalOperatorType : uint8_t {
	LOGICAL_INVALID = 0,
	LOGICAL_PROJECTION = 1,
	LOGICAL_FILTER = 2,
	LOGICAL_AGGREGATE_AND_GROUP_BY = 3,
	LOGICAL_WINDOW = 4,
	LOGICAL_UNNEST = 5,
	LOGICAL_LIMIT = 6,
	LOGICAL_ORDER_BY = 7,
	LOGICAL_TOP_N = 8,
	LOGICAL_COPY_TO_FILE = 10,
	LOGICAL_DISTINCT = 11,
	LOGICAL_SAMPLE = 12,
	LOGICAL_PIVOT = 14,
	LOGICAL_COPY_DATABASE = 15,
	LOGICAL_SECURE_VIEW = 16,

	// -----------------------------
	// Data sources
	// -----------------------------
	LOGICAL_GET = 25,
	LOGICAL_CHUNK_GET = 26,
	LOGICAL_DELIM_GET = 27,
	LOGICAL_EXPRESSION_GET = 28,
	LOGICAL_DUMMY_SCAN = 29,
	LOGICAL_EMPTY_RESULT = 30,
	LOGICAL_CTE_REF = 31,
	// -----------------------------
	// Joins
	// -----------------------------
	LOGICAL_JOIN = 50,
	LOGICAL_DELIM_JOIN = 51,
	LOGICAL_COMPARISON_JOIN = 52,
	LOGICAL_ANY_JOIN = 53,
	LOGICAL_CROSS_PRODUCT = 54,
	LOGICAL_POSITIONAL_JOIN = 55,
	LOGICAL_ASOF_JOIN = 56,
	LOGICAL_DEPENDENT_JOIN = 57,
	// -----------------------------
	// SetOps
	// -----------------------------
	LOGICAL_UNION = 75,
	LOGICAL_EXCEPT = 76,
	LOGICAL_INTERSECT = 77,
	LOGICAL_RECURSIVE_CTE = 78,
	LOGICAL_MATERIALIZED_CTE = 79,

	// -----------------------------
	// Updates
	// -----------------------------
	LOGICAL_INSERT = 100,
	LOGICAL_DELETE = 101,
	LOGICAL_UPDATE = 102,
	LOGICAL_MERGE_INTO = 103,
	LOGICAL_TRIGGER = 104,

	// -----------------------------
	// Schema
	// -----------------------------
	LOGICAL_ALTER = 125,
	LOGICAL_CREATE_TABLE = 126,
	LOGICAL_CREATE_INDEX = 127,
	LOGICAL_CREATE_SEQUENCE = 128,
	LOGICAL_CREATE_VIEW = 129,
	LOGICAL_CREATE_SCHEMA = 130,
	LOGICAL_CREATE_MACRO = 131,
	LOGICAL_DROP = 132,
	LOGICAL_PRAGMA = 133,
	LOGICAL_TRANSACTION = 134,
	LOGICAL_CREATE_TYPE = 135,
	LOGICAL_ATTACH = 136,
	LOGICAL_DETACH = 137,
	LOGICAL_CREATE_TRIGGER = 138,

	// -----------------------------
	// Explain
	// -----------------------------
	LOGICAL_EXPLAIN = 150,

	// -----------------------------
	// Helpers
	// -----------------------------
	LOGICAL_PREPARE = 175,
	LOGICAL_EXECUTE = 176,
	LOGICAL_EXPORT = 177,
	LOGICAL_VACUUM = 178,
	LOGICAL_SET = 179,
	LOGICAL_LOAD = 180,
	LOGICAL_RESET = 181,
	LOGICAL_UPDATE_EXTENSIONS = 182,
	LOGICAL_CONNECT = 183,
	LOGICAL_DISCONNECT = 184,
	LOGICAL_EXTERNAL_RESOURCE = 185,

	// -----------------------------
	// Secrets
	// -----------------------------
	LOGICAL_CREATE_SECRET = 190,

	LOGICAL_EXTENSION_OPERATOR = 255
};
```

Two design facts about this list are worth calling out explicitly.

**It's `uint8_t`.** DuckDB caps itself at 256 possible logical operator kinds,
total, forever (barring a wire-format-breaking change) — which is a small,
concrete piece of evidence for how disciplined the project is about not
growing this particular vocabulary casually. `LOGICAL_EXTENSION_OPERATOR = 255`
sitting at the very top of the `uint8_t` range is not an accident either: it's
the deliberately-reserved escape hatch (§5.4.15) for anything a third-party
extension needs that the built-in values don't cover, placed as far as possible
from the built-in numbering so it never collides with a future addition.

**The numeric gaps are deliberate, and they exist for exactly one reason:
plan-serialization stability.** Look at the "general" category:
`LOGICAL_PROJECTION = 1` through `LOGICAL_SECURE_VIEW = 16`, but the values
actually used are 1–12, 14, 15, 16 — gaps at 9, 13. Every category does this:
data sources jump straight to 25 rather than continuing at 17; joins start at
50, not 32; set-ops at 75; updates at 100; schema at 125; explain at 150;
helpers at 175; secrets at 190. This is the same instinct that motivates
leaving gaps in a database's own primary-key sequence, applied to a wire
format: `Serialize`/`Deserialize` (§5.2.4) embed this raw numeric tag directly
into a prepared-statement's serialized bytes and into `EXPLAIN (FORMAT json)`
output. If a future DuckDB release wants to add, say, a new kind of set
operation, it can slot the new value in at `80` — a gap left inside the
"SetOps" category — without renumbering `LOGICAL_INSERT` and every value after
it. Renumbering an existing, already-shipped value would silently corrupt any
plan serialized by an older binary and deserialized by a newer one (or vice
versa); leaving room to insert *within* a category means new operators almost
never need to touch existing numbers at all.

## 5.4 Concrete operators, one by one

`src/planner/operator/` holds 57 `.cpp` files (roughly one per enum value).
Their headers live in the mirrored `src/include/duckdb/planner/operator/`
directory, which holds 61 `.hpp` files. The rest of this section walks through the ones that
matter for ordinary query processing in real depth; the DML/DDL/utility
operators get one paragraph each, since their job — carry the bound
`CreateInfo`/`AlterInfo`/etc. struct the binder already built, do essentially
no relational computation — is much less conceptually rich than the
`SELECT`-side operators.

### 5.4.1 `LogicalGet` — every data source, not just base tables

```
SELECT * FROM orders;                              -- LogicalGet over a table
SELECT * FROM read_csv('orders.csv');                -- LogicalGet over a table FUNCTION
SELECT * FROM read_parquet('lineitem.parquet');       -- also LogicalGet
SELECT * FROM range(10);                              -- also LogicalGet
```

`LogicalGet` (`src/include/duckdb/planner/operator/logical_get.hpp`,
`src/planner/operator/logical_get.cpp`) is worth stating plainly because it's
a common point of confusion: **`LogicalGet` is not "the base-table scan
operator" — it is DuckDB's one and only "data source" node, and a base-table
scan is simply the single most common special case of a much more general
mechanism, "call a `TableFunction`."** Confirmed directly in the binder: both
`bind_basetableref.cpp` (an ordinary `FROM orders` reference) and
`bind_table_function.cpp` (`FROM read_csv(...)`, `FROM range(...)`, and every
other table-function call) construct the *same* `LogicalGet`:

```cpp
// src/planner/binder/tableref/bind_basetableref.cpp:290-291 (base table)
make_uniq<LogicalGet>(table_index, BoundTableFunction(scan_function), std::move(bind_data),
                      std::move(return_types), std::move(return_names), std::move(virtual_columns));

// src/planner/binder/tableref/bind_table_function.cpp:346-347 (table function call)
auto get = make_uniq<LogicalGet>(bind_index, table_function, std::move(bind_data), return_types, return_names,
                                 virtual_columns);
```

A base table scan is, mechanically, just a call to a built-in `TableFunction`
that happens to read from the catalog's stored row groups instead of a CSV
parser or an HTTP range request. This is *why* `read_csv` and `read_parquet`
compose so cleanly with everything else in the planner/optimizer/execution
pipeline (filter pushdown, projection pushdown, statistics, parallelism): none
of that machinery has a special case for "is this a real table or a table
function" — it all operates on `LogicalGet`'s fields uniformly. Its key
fields: `function` (a `BoundTableFunction`: the `TableFunction` together with
the argument types this call was bound with), `bind_data` (whatever that
function's own bind stage produced — a `DuckTableEntry` reference for a real
table, parsed CSV options for `read_csv`), `table_filters` (a `TableFilterSet`
— this is the destination the optimizer's filter-pushdown pass, Chapter 7,
pushes predicates *into*, so the scan itself — or the storage layer's
zonemaps, Chapter 16 — can skip data without a separate `LogicalFilter` node
ever running), and `projection_ids` (which of the table's/function's
`returned_types` columns are actually needed downstream — the destination of
projection pushdown).

The
`function` field is a `BoundTableFunction`, not a plain `TableFunction`
(`logical_get.hpp`: "The function that is called, together with the
argument types this call was bound with"). `LogicalGet` also has an
`at_clause` field. It keeps the `AT (...)` time-travel clause the table was
looked up with, so that `EXPLAIN (SQL)` (§5.5.4) can write it out again.

### 5.4.2 `LogicalFilter`

```sql
... WHERE l.shipdate < DATE '1998-01-01' ...
```

`LogicalFilter` (`logical_filter.hpp`) is the relational-algebra `σ`,
unmodified: one child, `expressions` holds the predicate(s) to test, one row
in equals zero-or-one rows out. It has a `SplitPredicates()` helper that
breaks a single `AND`-conjunction expression into separate list entries —
worth mentioning here because it's the reason the optimizer can push
`WHERE a AND b` down as two independently-relocatable filters `a` and `b`
rather than one atomic blob, each of which might belong at a different point
in the tree once the join order is decided (Chapter 7). Like `LogicalOrder`
below, it also carries an optional `projection_map` (its
`HasProjectionMap()` returns `!projection_map.empty()`) letting a filter also
reorder/subset its child's columns without needing a separate
`LogicalProjection` wrapped around it.

### 5.4.3 `LogicalProjection`

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total ...
```

`LogicalProjection` (`logical_projection.hpp`) is `π`: it owns a `TableIndex`
of its own (every projection introduces a *new* table index — its output
columns are addressed as `ColumnBinding{this_projection's_table_index, i}`
from here on, regardless of what table indices its input expressions actually
read from) and an `expressions` list that is literally the output column
list. `GetExpression(ColumnBinding)` lets later code map "the column this
downstream operator wants" straight back to "the bound `Expression` tree that
computes it" — needed, for instance, when the optimizer wants to push an
expression like `total > 1000` down past a projection and has to first check
what `total` actually *is* underneath.

### 5.4.4 The `LogicalJoin` family

All join operators share one base, `LogicalJoin`
(`src/include/duckdb/planner/operator/logical_join.hpp`), which itself
carries no join-condition representation at all — only what every join needs
regardless of how its condition is expressed: `join_type` (INNER, LEFT,
MARK, SEMI, ...), `mark_index` (for MARK joins — the synthetic boolean/NULL
column a `MARK` join adds), and `left_projection_map`/`right_projection_map`.
The condition representation is what differs between the concrete subclasses,
and that difference *is* the reason they're different classes rather than one
class with a variant field:

- **`LogicalComparisonJoin`** (`logical_comparison_join.hpp`) — the
  overwhelmingly common case: a join whose condition is a list of
  `JoinCondition { left; right; comparison }` pairs, each comparing one
  expression from the left child against one from the right using a single
  `ExpressionType` (`=`, `<`, `>=`, ...). `o.orderkey = l.orderkey` is exactly
  one `JoinCondition` with `comparison = COMPARE_EQUAL`. This structured
  representation (rather than a bare boolean expression tree) is precisely
  what lets the physical planner recognize "this is hash-joinable" (an
  equality condition) versus "this needs a merge join or a range-join
  operator" (an inequality condition) without re-parsing an expression tree —
  Chapter 9 covers exactly how that dispatch works.
- **`LogicalAnyJoin`** (`logical_any_join.hpp`) — the fallback for a join
  condition that *isn't* representable as a clean list of per-column
  comparisons: an arbitrary boolean `Expression`, e.g.
  `ON a.x + a.y > b.z` or `ON levenshtein(a.name, b.name) < 3`. It even has a
  `TrySpecialize()` static method whose job is exactly the inverse of "any
  join is the fallback" — given an `ANY` join whose expression turns out, on
  inspection, to actually decompose into ordinary comparisons after all,
  rewrite it in place into the more specialized `LogicalComparisonJoin` so the
  rest of the optimizer gets the more structured representation to work with.
- **`LogicalCrossProduct`** — no condition at all (`FROM a, b` with no `ON`/
  `WHERE` linking them, or a deliberate `CROSS JOIN`); every row of the left
  paired with every row of the right.
- **`LogicalPositionalJoin`** — DuckDB's `POSITIONAL JOIN`: pairs row *i* of
  the left with row *i* of the right by ordinal position, not by any column
  value at all (both `LogicalCrossProduct` and `LogicalPositionalJoin` derive
  from a shared `LogicalUnconditionalJoin` base, reflecting that neither has a
  `JoinCondition` list to speak of).
- **`LOGICAL_ASOF_JOIN`** is notable for having *no dedicated subclass file* —
  confirmed by grepping the planner source for its construction site
  (`src/planner/binder/tableref/plan_joinref.cpp:317`:
  `make_uniq<LogicalComparisonJoin>(type, LogicalOperatorType::LOGICAL_ASOF_JOIN)`).
  An `ASOF JOIN` is represented as an ordinary `LogicalComparisonJoin` object,
  just tagged with the `LOGICAL_ASOF_JOIN` enum value instead of
  `LOGICAL_COMPARISON_JOIN` — the class hierarchy doesn't need a new C++ type
  because an asof join's condition genuinely *is* a `JoinCondition` list (an
  equality on the "match key" plus one inequality on the "as-of" column), it's
  only the *execution strategy* implied by that tag that's special. For
  readers who haven't met it: `ASOF JOIN` is a DuckDB (and ClickHouse)
  extension to standard SQL for joining a table of events to the
  *most-recent-at-or-before* row of a second table by some ordered column —
  the canonical use case is joining trades to the most recent quoted price at
  or before each trade's timestamp, a pattern that in standard SQL requires an
  awkward correlated subquery or window function and in DuckDB is one join
  clause: `... ASOF JOIN quotes ON trades.symbol = quotes.symbol AND trades.ts >= quotes.ts`.

### 5.4.5 `LogicalDependentJoin` — the not-yet-decorrelated marker

```sql
SELECT c.name, (SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey) AS n_orders
FROM customer c;
```

```cpp
// src/include/duckdb/planner/operator/logical_dependent_join.hpp:6-38 (elided: #include lines)
// logical_dependent_join represents a logical operator for lateral joins that
// is planned but not yet flattened
//
// This construct only exists during planning and should not exist in the plan
// once flattening is complete. Although the same information can be kept in the
// join itself, creating a new construct makes the code cleaner and easier to
// understand.

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

This is the node that represents "the right-hand side of this join needs to
be *re-evaluated once per row* of the left-hand side, because it references
columns (`c.custkey`) that only exist on the left" — the thing no physical
join operator in DuckDB can actually execute directly (there is no
`PhysicalCorrelatedNestedLoopJoin`). Its own header comment is unusually
blunt about its lifetime: it is a deliberate, self-described *temporary*
node — "should not exist in the plan once flattening is complete." Query B
above is precisely the smallest query that forces the binder to build one, and
§5.7 below shows the actual tree shape right after binding produces it. What
happens to it next — the `FlattenDependentJoins` algorithm
(`src/planner/subquery/flatten_dependent_join.cpp`) that rewrites it away into
an ordinary join tree (a `LogicalComparisonJoin` tagged `LOGICAL_DELIM_JOIN`,
which by default is then rewritten into joins over materialized CTEs), *before* the
optimizer's first pass ever runs — is Chapter 6's entire subject. This chapter
only needs you to recognize the shape when you see it; decorrelating it is
somebody else's job.

### 5.4.6 `LogicalAggregate` — `GROUP BY` and grouping sets, one node

```sql
GROUP BY o.custkey                    -- ordinary GROUP BY
GROUP BY GROUPING SETS ((a), (b), ())  -- grouping sets / ROLLUP / CUBE
```

```cpp
// src/include/duckdb/planner/operator/logical_aggregate.hpp:21-64
class LogicalAggregate : public LogicalOperator {
	// ...
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
	// ...
};
```

One node handles both a plain `GROUP BY` (a single `GroupingSet` covering all
the `groups` expressions) and `GROUPING SETS`/`ROLLUP`/`CUBE` (multiple
entries in `grouping_sets`, each naming a different subset of `groups` to
group by, plus `grouping_functions` for the `GROUPING(col)` meta-function that
tells a caller which grouping-set produced a given output row). Notice it
carries *three* separate table indices, not one — `groups`, aggregate results,
and `GROUPING()` calls each get addressed as columns of a logically distinct
"table," even though they're computed by the same physical operator; this is
purely a naming/addressing convenience for the binder, not a hint about
execution (Chapter 9's `PhysicalHashAggregate`/`PhysicalUngroupedAggregate`
computes all three together).

### 5.4.7 `LogicalWindow`

```sql
SELECT custkey, RANK() OVER (PARTITION BY custkey ORDER BY orderdate) FROM orders;
```

`LogicalWindow` (`logical_window.hpp`) is a one-field wrapper —
`window_index`, a `TableIndex` for addressing its outputs — around an
`expressions` list of `BoundWindowExpression`s, each holding its own
`PARTITION BY`/`ORDER BY`/frame clause. Unlike `LogicalAggregate`, a window
function doesn't collapse rows (`GROUP BY` produces one row per group; a
window function produces one *extra column* per input row), which is exactly
why they're different node types even though both compute "an aggregate-like
function over a set of related rows."

### 5.4.8 `LogicalUnnest`

```sql
SELECT UNNEST([1, 2, 3]);        -- list -> rows
```

`LogicalUnnest` (`logical_unnest.hpp`) — same one-field shape as
`LogicalWindow` (`unnest_index`) — expands each `LIST`/`ARRAY`/`STRUCT`-typed
expression in `expressions` into multiple output rows (or, for a struct,
multiple output columns), the relational-algebra "unnest" operation that has
no classical single-letter symbol but is standard in any system supporting
nested/semi-structured types.

### 5.4.9 `LogicalOrder` vs. `LogicalTopN`

```sql
... ORDER BY total DESC;                -- LogicalOrder alone
... ORDER BY total DESC LIMIT 10;        -- LogicalOrder + LogicalLimit,
                                          -- *before* optimization
```

`LogicalOrder` (`logical_order.hpp`) holds a `vector<BoundOrderByNode> orders`
— each entry an expression plus ascending/descending and
nulls-first/nulls-last. On its own, right after binding, `ORDER BY ... LIMIT
n` is *two* separate nodes stacked — `LogicalLimit` on top of `LogicalOrder`
— exactly as shown in Chapter 1's preview of Query A. `LogicalTopN`
(`logical_top_n.hpp`) is a *different* concrete class the binder never
produces directly:

```cpp
// src/include/duckdb/planner/operator/logical_top_n.hpp:17-53
//! LogicalTopN represents a comibination of ORDER BY and LIMIT clause, using Min/Max Heap
class LogicalTopN : public LogicalOperator {
	// ...
	vector<BoundOrderByNode> orders;
	//! The maximum amount of elements to emit
	idx_t limit;
	//! The offset from the start to begin emitting elements
	idx_t offset;
	// ...
};
```

**`LogicalTopN` is created entirely by the optimizer, not the binder** — it's
the output of the `top_n` optimizer pass (Chapter 7) recognizing the pattern
"an `ORDER BY` immediately followed by a `LIMIT`" and fusing the two logical
nodes into this one. The reason that fusion is worth an entire dedicated node
type rather than just leaving `LogicalOrder`+`LogicalLimit` stacked: computing
a full sort of every row just to keep the top 10 wastes both memory and work
proportional to the *entire* input, when a bounded min/max-heap of size 10
(the doc comment says exactly this: "using Min/Max Heap") only ever holds 10
elements no matter how large the input is. This chapter only needs you to know
the *shape* difference — `LogicalOrder`+`LogicalLimit` is what binding
produces; `LogicalTopN` is what the optimizer's pattern-matching produces from
it — Chapter 7 covers the pass itself.

### 5.4.10 `LogicalDistinct`

```sql
SELECT DISTINCT custkey FROM orders;
SELECT DISTINCT ON (custkey) * FROM orders ORDER BY orderdate DESC;
```

`LogicalDistinct` (`logical_distinct.hpp`) carries a `DistinctType` —
`DISTINCT` (deduplicate on every output column) or `DISTINCT_ON` (deduplicate
on an explicit `distinct_targets` expression list, keeping one row per group
according to an accompanying `order_by`, DuckDB's implementation of Postgres's
`DISTINCT ON` extension).

### 5.4.11 `LogicalSetOperation`

```sql
SELECT custkey FROM orders UNION SELECT custkey FROM returned_orders;
... EXCEPT ...
... INTERSECT ...
```

`LogicalSetOperation` (`logical_set_operation.hpp`) is the one class behind
three enum values (`LOGICAL_UNION`, `LOGICAL_EXCEPT`, `LOGICAL_INTERSECT`) —
which one applies is carried in the base `LogicalOperator::type` field, not a
separate enum inside the class, because all three share the identical
`{table_index, column_count, setop_all, allow_out_of_order}` shape and differ
only in the row-combination semantics a physical set-operation implementation
applies at execution time. `setop_all` distinguishes `UNION` from `UNION ALL`
(same for the other two); `allow_out_of_order` records whether the rows from
each side may be interleaved in any order (true for `UNION ALL`) or must
preserve a side's relative order (false where that matters).

### 5.4.12 The CTE trio

```sql
WITH regional_totals AS (SELECT ... ) SELECT * FROM regional_totals r1, regional_totals r2 ...;

WITH RECURSIVE t(n) AS (
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM t WHERE n < 100
) SELECT * FROM t;
```

Three distinct node types share the job of representing a `WITH` clause,
because a CTE's *materialization strategy* and its *recursiveness* are both
things the relational tree needs to represent explicitly, not just an
inlining convenience:

- **`LogicalMaterializedCTE`** (`logical_materialized_cte.hpp`) — a binary
  node: child 0 computes the CTE body once, child 1 is the rest of the query
  that references it (possibly more than once) via `LogicalCTERef` nodes that
  all read from the *same* materialized result rather than each re-running
  the CTE body. Its `materialize` field (`CTEMaterialize`) records whether
  materialization is forced, forbidden, or left to the optimizer's judgment.
- **`LogicalRecursiveCTE`** (`logical_recursive_cte.hpp`) — the `WITH
  RECURSIVE` case: `top` computes the base case, `bottom` computes the
  recursive step, and the node itself owns the fixed-point iteration logic
  (`union_all` distinguishing `UNION` from `UNION ALL` recursion semantics,
  `key_targets` naming the columns used to detect duplicate rows for
  non-`ALL` recursion).
- **`LogicalCTERef`** (`logical_cteref.hpp`) — a leaf, structurally similar to
  `LogicalGet` but reading from an already-planned CTE result (identified by
  `cte_index`) instead of a table or table function.

### 5.4.13 DML: `LogicalInsert` / `LogicalUpdate` / `LogicalDelete` / `LogicalMergeInto`

```sql
INSERT INTO orders VALUES (...);
UPDATE orders SET status = 'shipped' WHERE orderkey = 42;
DELETE FROM orders WHERE orderkey = 42;
MERGE INTO orders USING staging ON ... WHEN MATCHED THEN UPDATE ...;
```

These four represent DuckDB's write path at the logical level. Each wraps a
single child subtree that computes the rows to be affected (an `INSERT ...
SELECT`'s source query, an `UPDATE`/`DELETE`'s `WHERE`-filtered scan of the
target table, a `MERGE INTO`'s join between target and source) plus the
catalog target and, for `UPDATE`/`MERGE`, the bound expressions computing new
column values. They are unapologetically less "relational" than the `SELECT`
side of the vocabulary — there's no algebraic operator for "write these rows
to a table" — but they still need `GetColumnBindings()`/`types` like anything
else, because `INSERT ... RETURNING` and similar constructs mean a write
operator can still have a meaningful output schema of its own.

### 5.4.14 DDL: `LogicalCreateTable` / `LogicalCreateIndex` / `LogicalAlter` / `LogicalDrop`

```sql
CREATE TABLE orders (orderkey BIGINT, custkey BIGINT, ...);
CREATE INDEX idx_orderkey ON orders(orderkey);
ALTER TABLE orders ADD COLUMN notes VARCHAR;
DROP TABLE orders;
```

These carry the already-bound `CreateInfo`/`AlterInfo`/etc. structs the
binder's `bind_create.cpp`/`bind_alter.cpp`/`bind_drop.cpp` produced, plus (for
`LogicalCreateTable` with a `CREATE TABLE ... AS SELECT`) a child subtree
computing the rows to populate the new table with. There is essentially no
relational computation happening in these nodes themselves — they are
thin carriers that let the rest of the pipeline (optimizer skip, via
`RequireOptimizer()`; physical plan generator) treat schema changes uniformly
alongside everything else that flows through `unique_ptr<LogicalOperator>`.

### 5.4.15 `LogicalExtensionOperator` — the third-party hook

```cpp
// src/include/duckdb/planner/operator/logical_extension_operator.hpp:18-44
struct LogicalExtensionOperator : public LogicalOperator {
public:
	LogicalPlanSQLExportResult ToSQL(LogicalPlanSQLExportContext &context,
	                                 const LogicalPlanVerificationPath &path) override;

	static constexpr const LogicalOperatorType TYPE = LogicalOperatorType::LOGICAL_EXTENSION_OPERATOR;

public:
	LogicalExtensionOperator() : LogicalOperator(LogicalOperatorType::LOGICAL_EXTENSION_OPERATOR) {
	}
	explicit LogicalExtensionOperator(vector<unique_ptr<Expression>> expressions)
	    : LogicalOperator(LogicalOperatorType::LOGICAL_EXTENSION_OPERATOR, std::move(expressions)) {
	}

	void Serialize(Serializer &serializer) const override;
	static unique_ptr<LogicalOperator> Deserialize(Deserializer &deserializer);

	virtual PhysicalOperator &CreatePlan(ClientContext &context, PhysicalPlanGenerator &planner) = 0;

	//! Return a non-empty stable identity to opt into standard traversal and exact output binding/type verification
	virtual optional_ptr<const string> GetTypeBindingVerificationIdentifier() const noexcept {
		return nullptr;
	}

	virtual void ResolveColumnBindings(ColumnBindingResolver &res, vector<ColumnBinding> &bindings);
	virtual string GetExtensionName() const;
};
```

Every enum value examined so far belongs to DuckDB itself. This is the one
value (`LOGICAL_EXTENSION_OPERATOR = 255`, deliberately parked at the top of
the `uint8_t` range as noted in §5.3) reserved for extensions that need to
introduce genuinely new relational operators the core engine has no built-in
concept of. A concrete subclass overrides the pure-virtual `CreatePlan` to
supply its own physical operator directly (bypassing the ordinary
`LogicalOperatorType`-to-`PhysicalOperator` dispatch table Chapter 9
describes) and can override `ResolveColumnBindings` if its column-binding
behavior doesn't match `ColumnBindingResolver`'s default traversal (§5.5.3).

## 5.5 Supporting visitor infrastructure

Three pieces of machinery operate *over* the `LogicalOperator` tree rather
than being nodes *in* it. All three matter for what comes next in this book.
A fourth turns the tree back into SQL (§5.5.4).

### 5.5.1 `LogicalOperatorVisitor`

```cpp
// src/include/duckdb/planner/logical_operator_visitor.hpp:21-74 (elided: doc comments retained inline)
//! The LogicalOperatorVisitor is an abstract base class that implements the
//! Visitor pattern on LogicalOperator.
class LogicalOperatorVisitor {
public:
	virtual ~LogicalOperatorVisitor() {
	}

	virtual void VisitOperator(unique_ptr<LogicalOperator> &op);
	virtual void VisitOperator(LogicalOperator &op);
	virtual void VisitExpression(unique_ptr<Expression> *expression);

	static void EnumerateExpressions(LogicalOperator &op,
	                                 const std::function<void(unique_ptr<Expression> *child)> &callback);
	// ...

protected:
	//! Automatically calls the Visit method for LogicalOperator children of the current operator.
	void VisitOperatorChildren(LogicalOperator &op);
	//! Automatically calls the Visit method for Expression children of the current operator.
	void VisitOperatorExpressions(LogicalOperator &op);
	// ...

	virtual void VisitExpressionChildren(Expression &expression);

	virtual unique_ptr<Expression> VisitReplace(BoundAggregateExpression &expr, unique_ptr<Expression> *expr_ptr);
	virtual unique_ptr<Expression> VisitReplace(BoundCaseExpression &expr, unique_ptr<Expression> *expr_ptr);
	virtual unique_ptr<Expression> VisitReplace(BoundColumnRefExpression &expr, unique_ptr<Expression> *expr_ptr);
	// ... one VisitReplace overload per BoundExpression subtype
	virtual unique_ptr<Expression> VisitReplace(BoundWindowExpression &expr, unique_ptr<Expression> *expr_ptr);
	virtual unique_ptr<Expression> VisitReplace(BoundUnnestExpression &expr, unique_ptr<Expression> *expr_ptr);
};
```

This is the base tree-walker class the overwhelming majority of DuckDB's
optimizer passes derive from — recognize this shape now, because Chapter 7 is
largely a tour of concrete `LogicalOperatorVisitor` subclasses, each
overriding a small subset of `VisitOperator`/`VisitReplace` to implement one
rewrite (filter pushdown walks the tree looking for `LogicalFilter` nodes to
relocate; a constant-folding pass overrides the relevant `VisitReplace`
overloads to fold foldable expressions in place). The one-`VisitReplace`-
overload-per-bound-expression-type design lets a pass that only cares about,
say, `BoundColumnRefExpression`s (the `ColumnBindingResolver` below is exactly
this) override a single method and inherit correct default (identity)
behavior for every other expression kind, rather than writing an exhaustive
`switch` by hand.

### 5.5.2 `LogicalPlanVerifier`

```cpp
// src/include/duckdb/planner/logical_plan_verifier.hpp:18-40
class LogicalPlanVerifier : private ColumnBindingResolver {
public:
	static void Verify(ClientContext &context, LogicalOperator &op);
	DUCKDB_API static LogicalPlanVerificationResult<LogicalPlanVerificationSuccess> VerifyAlways(LogicalOperator &op);
	//! Check bindings against an already resolved plan without resolving or changing its output metadata.
	//! The caller supplies the resolved types; this does not infer or repair stale operator types.
	DUCKDB_API static LogicalPlanVerificationResult<LogicalPlanVerificationSuccess>
	VerifyResolvedBindings(LogicalOperator &op);

private:
	enum class TypeResolution { RESOLVE, EXISTING };
	explicit LogicalPlanVerifier(LogicalPlanVerificationState &verification_state);

	unique_ptr<Expression> VisitReplace(BoundColumnRefExpression &expr, unique_ptr<Expression> *expr_ptr) override;
	static bool ResolveOperatorTypes(LogicalOperator &op, LogicalPlanVerificationState &verification_state,
	                                 TypeResolution resolution = TypeResolution::RESOLVE);
	static void VerifyColumnBindings(LogicalOperator &op, LogicalPlanVerificationState &verification_state);
	// ...
};
```

Two things worth noting here. First, this is genuinely a sanity-check pass run
*after every single optimizer pass* (when verification is enabled, which it
is in debug builds and the test suite) — confirmed directly in
`Optimizer::RunOptimizer`:

```cpp
// src/optimizer/optimizer.cpp:137-156
void Optimizer::RunOptimizer(OptimizerType type, const std::function<void()> &callback) {
	// ...
	{
		auto optimizer_timer = profiler.StartTimerInternal("optimizer." + StringUtil::Lower(EnumUtil::ToString(type)));
		callback();
	}
	if (plan) {
		Verify(*plan);
	}
}

void Optimizer::Verify(LogicalOperator &op) {
	LogicalPlanVerifier::Verify(context, op);
}
```

Every one of the optimizer's ~45 passes (Chapter 7) is wrapped in this same
`RunOptimizer` call, so if any pass leaves the tree in a state where a column
binding doesn't resolve, or a resolved type doesn't match what an expression
claims to produce, the very next line of code catches it — rather than
letting a subtly-broken plan silently reach the physical planner or (worse)
execution, where the failure mode would be a much harder-to-diagnose crash or
wrong answer. Second, notice the inheritance: `LogicalPlanVerifier` *privately
derives from* `ColumnBindingResolver` (§5.5.3) — it reuses that exact same
binding-resolution traversal, just in a `verify_only` mode that checks every
`BoundColumnRefExpression` actually resolves to something and every claimed
type matches, without mutating the tree into the physical, flat-index form.
Verification and physical-index resolution are, mechanically, the *same
walk*, run twice: once early and repeatedly (as a checker), once at the very
end and for real (as a transformation).

### 5.5.3 `ColumnBindingResolver` — the bridge into Chapter 9

```cpp
// src/include/duckdb/execution/column_binding_resolver.hpp:17-33
//! The ColumnBindingResolver resolves ColumnBindings into base tables
//! (table_index, column_index) into physical indices into the DataChunks that
//! are used within the execution engine
class ColumnBindingResolver : public LogicalOperatorVisitor {
public:
	explicit ColumnBindingResolver(bool verify_only = false);
	DUCKDB_API ColumnBindingResolver(vector<ColumnBinding> bindings, vector<LogicalType> types);

	void VisitOperator(LogicalOperator &op) override;

protected:
	vector<ColumnBinding> bindings;
	vector<LogicalType> types;
	bool verify_only;

	unique_ptr<Expression> VisitReplace(BoundColumnRefExpression &expr, unique_ptr<Expression> *expr_ptr) override;
};
```

This is the step that runs *after* the optimizer has finished (not during it
— confirmed by its being called from the physical-planning path, not from
inside `Optimizer::Optimize`) and *right before* the physical plan generator
takes over (Chapter 9). Its job, stated precisely, is a coordinate-system
change: convert every `BoundColumnRefExpression`'s `ColumnBinding{table_index,
column_index}` — the addressing scheme the entire logical layer has used
since Chapter 4 — into a flat, positional `BoundReferenceExpression{index}`,
where `index` is nothing more than "how far into the incoming `DataChunk`'s
column list is this value." The conversion itself:

```cpp
// src/execution/column_binding_resolver.cpp:243-270
unique_ptr<Expression> ColumnBindingResolver::VisitReplace(BoundColumnRefExpression &expr,
                                                           unique_ptr<Expression> *expr_ptr) {
	D_ASSERT(expr.Depth() == 0);
	// check the current set of column bindings to see which index corresponds to the column reference
	for (idx_t i = 0; i < bindings.size(); i++) {
		if (expr.Binding() == bindings[i]) {
			// ... (type-consistency assertions elided)
			if (verify_only) {
				// in verification mode
				return nullptr;
			}
			return make_uniq<BoundReferenceExpression>(expr.GetAlias(), expr.GetReturnType(), i);
		}
	}
	// ...
}
```

The `bindings` member is walked to find the position `i` at which the
currently-in-scope column list contains this exact `ColumnBinding` — that
position `i` is exactly the value stamped into the replacement
`BoundReferenceExpression`, whose own header states its purpose in one line:
`//! A BoundReferenceExpression represents a physical index into a DataChunk`
(`src/include/duckdb/planner/expression/bound_reference_expression.hpp:15`).

*Why* this conversion has to happen at all is worth being explicit about,
because it's the cleanest possible bridge into Chapter 9's subject matter:
**physical operators do not carry table indices.** A `PhysicalHashJoin` or
`PhysicalFilter`, at execution time, is handed a `DataChunk` — an anonymous,
flat vector of columns with no memory of "table 3, column 1" anywhere in it;
it only knows "evaluate my expression against column *N* of whatever chunk
arrived." The entire `{table_index, column_index}` addressing scheme that
made Chapter 4's binder and this chapter's logical operators legible — where
every column can be traced back to which logical node produced it — is
*planning-time* infrastructure with no counterpart at runtime, and it would be
actively wasteful to carry it forward: resolving a table/column pair to an
actual value at every single row, on every single expression evaluation,
would mean a hash lookup or linear scan per reference. `ColumnBindingResolver`
does that resolution work exactly *once per expression*, at the plan/physical
boundary, so that by the time execution starts, every column reference is
already a bare integer offset — an array index, not a lookup.

### 5.5.4 `ToSQL` — turning a logical plan back into SQL

DuckDB can walk a logical plan and write it out as SQL text that DuckDB
can run again. The user-facing entry point is a third kind of `EXPLAIN`,
`ExplainType::EXPLAIN_SQL` (Chapter 2 §2.5.1):

```sql
-- test/sql/sql_export/explain_sql_scopes.test
EXPLAIN (SQL) SELECT NULL::INTEGER AS "null value", 2::BIGINT AS "two";
----
sql	SELECT CAST(NULL AS INTEGER) AS "null value", CAST(2 AS BIGINT) AS two
```

The result is one row. The key is `sql` and the value is the generated query.
It is *not* the original query text. It is rebuilt from the plan *after* the
optimizer has run, so it shows the optimizer's rewrites: casts the binder
added become explicit `CAST`s, and folded constants show up as literals. The
design note in `src/planner/sql_export/README.md` gives more examples: a join
input known to be empty can become `WHERE false`, and `now()` can become a
fixed timestamp literal.

Each logical operator does its own part of the work through the virtual
method `ToSQL` on the base class (see the excerpt in §5.2). The base-class
version simply reports that the operator is not supported:

```cpp
// src/planner/sql_export/logical_plan_sql_exporter.cpp:322-330
LogicalPlanVerificationResult<LogicalPlanSQLExportRelation>
LogicalOperator::ToSQL(LogicalPlanSQLExportContext &, const LogicalPlanVerificationPath &path) {
	if (type == LogicalOperatorType::LOGICAL_DELIM_GET) {
		return LogicalPlanSQLExportResult::Failure({LogicalPlanSQLExportHelpers::UnsupportedSource(
		    path, LogicalPlanSQLExportHelpers::LogicalSourceIdentity(), "delim_get")});
	}
	D_ASSERT(type != LogicalOperatorType::LOGICAL_INVALID);
	return LogicalPlanSQLExportResult::Failure({UnsupportedOperator(path, type)});
}
```

28 operator headers under `src/include/duckdb/planner/operator/` override
it, including `LogicalFilter`, `LogicalProjection`, `LogicalAggregate`, the
joins, the CTE operators, and `LogicalExtensionOperator` (§5.4.15). For
example, `LogicalFilter::ToSQL` (`src/planner/sql_export/sql_export_relations.cpp:110`)
first exports its child, then turns each predicate in `expressions` back
into a parsed expression for a `WHERE` clause. The implementation lives in
the directory `src/planner/sql_export/` (16 `.cpp` files, grouped by
sources, joins, CTEs, `LIMIT`, `PIVOT`, windows and DML).

When does it run? `Planner::Optimize` runs the optimizer, then calls
`RenderSQLExplain` (`src/planner/planner.cpp:37-48`, called at line 77). If
the root is a `LogicalExplain` with `ExplainType::EXPLAIN_SQL`, that
function replaces it with the one-row result built by
`LogicalExplain::CreateSQLResult` (`src/planner/operator/logical_explain.cpp:16`).
If some operator in the plan cannot be exported, the user gets an error
that starts with `EXPLAIN (SQL) cannot render this query:`. DuckDB never
falls back to returning the original query text.

`EXPLAIN (SQL)` works for `SELECT`, `VALUES`, `WITH`, `INSERT`, `UPDATE`,
`DELETE` and `MERGE INTO`. It does not combine with `ANALYZE` or `FORMAT`,
and it does not support DDL. The same machinery also backs a test-suite
mode, `debug_verify_statement='explain_sql'`: every test query is exported to
SQL, the SQL is run, and its result must match. This makes every logical
operator's `ToSQL` a second, independent check on the plan.

## 5.6 Query A, in full: the unoptimized logical plan

Chapter 1 previewed this tree with just operator names and one-line
annotations. Here is the same tree, right after binding, with each node's
actual `LogicalOperatorType`, its `expressions`, and its resolved `types`
filled in. (Table indices below are illustrative — `Binder::GenerateTableIndex()`
is a simple process-wide incrementing counter, so the *exact* numbers a real
run assigns depend on binding order; what matters, and *is* guaranteed, is
that every operator introducing a new addressable column set gets a distinct
index.)

```sql
SELECT o.custkey, SUM(l.extendedprice) AS total
FROM orders o
JOIN lineitem l ON o.orderkey = l.orderkey
WHERE l.shipdate < DATE '1998-01-01'
GROUP BY o.custkey
ORDER BY total DESC
LIMIT 10;
```

```
┌──────────────────────────────────────────────────────────────────────────┐
│ LOGICAL_LIMIT  (LogicalLimit)                                             │
│   limit = 10, offset = 0                                                  │
│   types  = [BIGINT, DOUBLE]           (unchanged from child)              │
└──────────────────────────────────┬───────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼───────────────────────────────────────┐
│ LOGICAL_ORDER_BY  (LogicalOrder)                                          │
│   orders = [ { expr: ColumnBinding{2,1} ("total"), ASCENDING: false } ]    │
│   types  = [BIGINT, DOUBLE]                                               │
└──────────────────────────────────┬───────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼───────────────────────────────────────┐
│ LOGICAL_AGGREGATE_AND_GROUP_BY  (LogicalAggregate)                        │
│   group_index = 2, aggregate_index = 2 (same table index; two            │
│                     column groups addressed as {2,0} and {2,1})           │
│   groups      = [ ColumnBinding{0,1} ]        -- o.custkey                │
│   expressions = [ SUM(ColumnBinding{1,5}) ]    -- SUM(l.extendedprice)    │
│   types       = [BIGINT, DOUBLE]              -- custkey, total           │
│   GetColumnBindings() = [ {2,0}, {2,1} ]                                   │
└──────────────────────────────────┬───────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼───────────────────────────────────────┐
│ LOGICAL_FILTER  (LogicalFilter)                                           │
│   expressions = [ ColumnBinding{1,10} < DATE '1998-01-01' ]  -- l.shipdate│
│   types       = (unchanged from child — filter drops rows, not columns)  │
└──────────────────────────────────┬───────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼───────────────────────────────────────┐
│ LOGICAL_COMPARISON_JOIN  (LogicalComparisonJoin, join_type = INNER)       │
│   conditions = [ JoinCondition{ left:  ColumnBinding{0,0} ("orderkey"),   │
│                                 right: ColumnBinding{1,0} ("orderkey"),   │
│                                 comparison: COMPARE_EQUAL } ]              │
│   types = types(orders) ++ types(lineitem)   (all columns of both sides, │
│                                                pre-projection)             │
└──────────────┬─────────────────────────────────────┬─────────────────────┘
               │                                       │
┌──────────────▼───────────────┐         ┌────────────▼───────────────────┐
│ LOGICAL_GET  (LogicalGet)     │         │ LOGICAL_GET  (LogicalGet)      │
│   table_index = 0             │         │   table_index = 1              │
│   function    = seq_scan       │         │   function    = seq_scan        │
│     (built-in TableFunction    │         │     (built-in TableFunction     │
│      bound against the         │         │      bound against the          │
│      "orders" catalog entry)   │         │      "lineitem" catalog entry)  │
│   table_filters = {}  (none    │         │   table_filters = {}  (empty    │
│     pushed yet — this is       │         │     pre-optimization: shipdate  │
│     pre-optimization)          │         │     filter still lives in the   │
│                                 │         │     LogicalFilter node above)   │
│   types = types(orders)         │         │   types = types(lineitem)       │
└─────────────────────────────────┘         └──────────────────────────────┘
```

A few things this annotated version makes visible that the Chapter 1 preview
glossed over: the `LogicalAggregate`'s `groups`/aggregate-`expressions` split
means `o.custkey` and `SUM(l.extendedprice)` are stored in genuinely different
fields of the same node, not one undifferentiated list; the `LogicalFilter`'s
predicate still references `lineitem`'s *un-pushed* `ColumnBinding` — nothing
about "push this into the scan" has happened yet, because that's an optimizer
pass (Chapter 7), not something binding does; and both `LogicalGet` nodes have
an empty `table_filters` for the identical reason. This is, precisely, the
tree `Optimizer::Optimize` receives as its input argument.

## 5.7 Query B: a `LogicalDependentJoin`, caught in the act

```sql
SELECT c.name,
       (SELECT COUNT(*) FROM orders o WHERE o.custkey = c.custkey) AS n_orders
FROM customer c;
```

Binding the scalar subquery in the `SELECT` list is what forces the marker
node from §5.4.5 into existence. The subquery's own `WHERE o.custkey =
c.custkey` references `c.custkey` — a column that belongs to the *outer*
query's `FROM customer c`, not to anything the subquery itself scans — which
is exactly the definition of a correlated reference, and exactly what
`LogicalDependentJoin` exists to represent explicitly rather than silently:

```
┌───────────────────────────────────────────────────────────────────────────┐
│ LOGICAL_PROJECTION  (LogicalProjection)                                    │
│   table_index = 2                                                         │
│   expressions = [ ColumnBinding{0,1} ("c.name"),                          │
│                    ColumnBinding{1,0} ("n_orders" — the subquery's COUNT)] │
│   types = [VARCHAR, BIGINT]                                                │
└──────────────────────────────────────┬────────────────────────────────────┘
                                        │
┌───────────────────────────────────────▼───────────────────────────────────┐
│ LOGICAL_DEPENDENT_JOIN  (LogicalDependentJoin, join_type = INNER)          │
│   condition           = nullptr   (this join's "condition" IS the         │
│                          correlation itself, not a JoinCondition list)     │
│   correlated_columns  = [ ColumnBinding{0,0} ]   -- c.custkey, referenced  │
│                          from inside the right child                       │
│   perform_delim           = true                                            │
│   propagate_null_values   = true                                            │
│                                                                             │
│   ── this node's entire reason for existing: the right child below        │
│      cannot be evaluated once for the whole query — conceptually, it       │
│      must be re-evaluated once per row the left child produces, with       │
│      c.custkey bound to that row's value each time. No physical           │
│      operator implements this directly (Chapter 6 rewrites it away         │
│      before the optimizer's first pass ever runs). ──                     │
└──────────────┬───────────────────────────────────────┬────────────────────┘
               │ (left: outer query)                    │ (right: subquery body)
┌──────────────▼───────────────┐         ┌──────────────▼──────────────────┐
│ LOGICAL_GET  (LogicalGet)     │         │ LOGICAL_AGGREGATE_AND_GROUP_BY   │
│   table_index = 0              │         │ (LogicalAggregate — COUNT(*),   │
│   function    = seq_scan        │         │  no GROUP BY: one row out)      │
│   ("customer")                  │         │   aggregate_index = 1            │
│                                  │         │   groups = []                    │
│                                  │         │   expressions = [ COUNT(*) ]      │
│                                  │         │   types = [BIGINT]                │
│                                  │         └──────────────┬───────────────────┘
│                                  │                          │
│                                  │         ┌────────────────▼───────────────┐
│                                  │         │ LOGICAL_FILTER                  │
│                                  │         │   expressions =                 │
│                                  │         │    [ ColumnBinding{3,0}          │
│                                  │         │      ("o.custkey")  =            │
│                                  │         │      ColumnBinding{0,0}          │
│                                  │         │      ("c.custkey" — CORRELATED,  │
│                                  │         │       reaches outside this       │
│                                  │         │       subtree entirely) ]        │
│                                  │         └────────────────┬───────────────┘
│                                  │                            │
│                                  │         ┌──────────────────▼─────────────┐
│                                  │         │ LOGICAL_GET  (table_index = 3) │
│                                  │         │   ("orders")                    │
│                                  │         └────────────────────────────────┘
└─────────────────────────────────┘
```

The tell-tale detail is the `LogicalFilter` two levels down on the right: its
predicate mixes a `ColumnBinding` local to that subtree (`{3,0}`, `orders`'
own `custkey`) with a `ColumnBinding` (`{0,0}`) belonging to a `LogicalGet`
that isn't even one of its ancestors in this subtree — it's the *sibling*
subtree's scan, reachable only by climbing back up through the
`LogicalDependentJoin` node itself. That crossing-the-subtree-boundary
reference is, precisely, what `LogicalJoin`'s `LogicalJoin::GetTableReferences`
helper (§5.4.4) and the `CorrelatedColumns correlated_columns` field on
`LogicalDependentJoin` exist to track explicitly, rather than leaving buried
inside an expression tree where nothing else in the planner would notice it.
This is exactly the shape the binder hands off; nothing here has been
decorrelated yet, and nothing in this chapter's machinery is capable of doing
that. `FlattenDependentJoins` (`src/planner/subquery/flatten_dependent_join.cpp`)
is what turns this into an ordinary, executable join tree. It runs in
`Planner::CreatePlan` right after binding returns, still before the
optimizer's first pass — Chapter 6, in full.

## 5.8 Handoff to the optimizer

Whatever shape binding produced — Query A's clean join/filter/aggregate/order/
limit stack, or Query B's already-flattened-by-the-time-you'd-see-it-in-
practice dependent join — it all arrives at the optimizer through one
function, taking and returning the same raw tree type:

```cpp
unique_ptr<LogicalOperator> Optimizer::Optimize(unique_ptr<LogicalOperator> plan_p);
```
(`src/include/duckdb/optimizer/optimizer.hpp:28`, implementation
`src/optimizer/optimizer.cpp:577` — this
book's Chapter 7 is the full account of what happens between that call being
made and it returning.)

Nothing about the node *vocabulary* changes across that call — the optimizer
does not introduce new kinds of `LogicalOperator` wholesale (with the single,
already-noted exception of `LogicalTopN`, fused from `LogicalOrder` +
`LogicalLimit` by one specific pass). What changes is the tree's *shape* and
the *contents* of individual nodes: filters relocate closer to scans,
`table_filters` gets populated, join order gets chosen, cardinalities get
estimated and attached via `estimated_cardinality`. Every one of those
rewrites is still expressed in the same operator vocabulary this
chapter just walked through, checked after every single pass by the same
`LogicalPlanVerifier` this chapter introduced, and only converted out of its
`ColumnBinding`-addressed form into flat physical indices once, right at the
very end, by `ColumnBindingResolver` — at which point, and only at which
point, it stops being a logical plan at all and becomes raw material for
Chapter 9's physical plan generator.
