# 7.27 — Compressed Materialization

## 7.27.0 Where this sits, and why it doesn't have a number in §7.4's list

Every other pass in this chapter's deep-dive series corresponds to one line in
§7.4's numbered pipeline — a named `OptimizerType`, wired up through its own
`Optimizer::RunOptimizer(OptimizerType::..., ...)` call site, with its own
profiler timer. Compressed materialization is not that. The overview chapter
is explicit about this, and it's worth quoting rather than paraphrasing,
because it's easy to go looking for a `RunOptimizer(OptimizerType::COMPRESSED_MATERIALIZATION, ...)`
call and conclude — wrongly — that the pass doesn't really exist:

> `DISTINCT_AGGREGATE_REWRITE` and `COMPRESSED_MATERIALIZATION`, by contrast,
> *are* real, live toggles — they're just not wrapped in their own
> `RunOptimizer(...)` call sites, and so don't get their own line in the
> numbered list or their own profiler timer. ... `COMPRESSED_MATERIALIZATION`
> gates a check inside `STATISTICS_PROPAGATION` itself
> (`statistics_propagator.cpp:138`) that decides whether to apply a
> compressed intermediate representation for certain materialized results.
> ... they're just implemented as inline policy checks inside a *different*
> pass's callback rather than as passes of their own.
> — `00-overview.md`, §7.4, "What's conspicuously absent from this numbered
> list, and why"

The overview also lists `compressed_materialization.cpp` among the files
backing pass #34, `STATISTICS_PROPAGATION` — alongside `relation_statistics/`
and `filter_statistics.cpp` — precisely because it is not its own
`OptimizerType` pass but a helper this one invokes. Reading the actual call
site confirms the mechanism exactly:

```cpp
// src/optimizer/statistics_propagator.cpp:137-144
if (mode == StatisticsPropagationMode::FULL &&
    !optimizer.OptimizerDisabled(OptimizerType::COMPRESSED_MATERIALIZATION)) {
	// compress data based on statistics for materializing operators
	CompressedMaterialization compressed_materialization(optimizer, *root, propagation_path, statistics_map);
	compressed_materialization.Compress(node_ptr);
}

propagation_path.pop_back();
```

This sits at the tail of `StatisticsPropagator::PropagateStatistics`
(`statistics_propagator.cpp:72-146`), the function that recursively visits
every node in the plan **bottom-up**: the big `switch` on `node.type` first
dispatches to a per-operator-type overload that recurses into children,
computes that node's own output statistics from its children's, and only
*after* that returns does this block run. So the actual invocation shape is:
for every single node in the tree, the instant its own statistics have been
computed, immediately ask "is this node a materializing operator we know how
to shrink, given what we now know about its columns' value ranges?" — once,
gated behind two conditions: `mode == FULL` (a lighter, non-`FULL`
propagation mode exists too, used elsewhere for cheaper cardinality-only
estimation, and compressed materialization never runs during it), and the
toggle not being disabled. `disabled_optimizers = 'compressed_materialization'`
is therefore a completely real, working knob — `duckdb_optimizers()` lists
it, `SET disabled_optimizers` accepts it — it just isn't its own step in
§7.4's pipeline; it's a conditional *inside* step #34.

The propagator remembers the path it took to reach each node. The first
line of `PropagateStatistics` is `propagation_path.emplace_back(node_ptr);`
(`statistics_propagator.cpp:74`), and the last step before it returns is
`propagation_path.pop_back();` (line 144). So while a node is being
handled, `propagation_path` holds the chain of operators from the plan
root down to that node. The propagator passes this chain to
`CompressedMaterialization` as a constructor argument, `ancestors`
(`compressed_materialization.hpp:180-181`: *"The chain of operators from
the root down to the operator being compressed"*). §7.27.3.3 explains
what the pass does with it.

## 7.27.1 The one-sentence purpose

**For a value that is about to be copied into a materializing operator's own
internal storage — a hash-aggregate's grouping key, a hash-join's build-side
hash table, a sort's key/payload buffer, a `DISTINCT`'s deduplication
structure — compressed materialization asks whether the *propagated
statistics* prove that value's actual range or shape is narrower than its
declared logical type, and if so, wraps the operator in a pair of
projections that shrink it going in and restore it coming out, so the
operator's own internal materialized copy is physically smaller for exactly
as long as that operator needs it and not one instruction longer.**

Everything else in this section is detail on top of that sentence.

## 7.27.2 Two kinds of "compression" that rhyme, and are not the same mechanism

Chapter 17's §17.4 documents an entirely different, and much more familiar,
kind of compression: the storage engine's **on-disk column compression**
(bitpacking, dictionary encoding, FSST, Chimp/Patas, ALP, Roaring, Zstd).
That family of schemes exists to shrink the bytes a `ColumnSegment` occupies
*at rest*, decided by an analyze/compress/scan framework
(`src/storage/compression/`, `src/include/duckdb/function/compression_function.hpp:131-141`)
that scans a segment's **actual, full set of values** once, at write time
(insert or checkpoint), scores several candidate schemes against each other,
and picks a winner that then persists — potentially across many future
queries, until the segment is next rewritten.

Compressed materialization is a completely different piece of machinery
that happens to rhyme with it conceptually:

| | Storage compression (Ch. 17 §17.4) | Compressed materialization (this section) |
|---|---|---|
| **When decided** | At write/checkpoint time, per `ColumnSegment` | At query-plan-optimization time, per query |
| **Decided from** | Scanning the segment's actual values (analyze phase) | *Propagated statistics* (min/max, max string length, VARIANT shredding info) — never a value scan |
| **Lifetime** | Persists on disk until the segment is rewritten | Exists only inside one query's plan, between one compress-projection and one decompress-projection |
| **What it compresses** | Whole stored columns, any column, unconditionally | Only the specific columns feeding one of four materializing operator types, and only if a narrower *safe* representation is provable |
| **Mechanism** | A `CompressionFunction`'s bespoke bit-level encoding per scheme | A handful of scalar functions (offset-subtract-and-narrow, byte-reversed prefix packing) or, best case, an ordinary `CAST` |
| **Selection** | Best-of-several-candidates scoring (`final_analyze` score) | A fixed, per-type decision tree — no competing candidates, no scoring |

The two are worth holding apart precisely *because* they rhyme: both are
instances of the general idea "the declared logical type is wider than the
data actually needs, so represent it more compactly for as long as that's
safe," and Chapter 16's ART key encoding is a third instance of the same
idea again — ART maps arbitrary values into compact, **byte-comparable**
keys so a *persistent* index structure can compare them cheaply. Compressed
materialization's string- and geometry-packing functions
(§7.27.4) do something structurally identical — pack a value's meaningful
bytes into a fixed-width integer, in reversed byte order, specifically so
that an ordinary unsigned-integer comparison of the packed form agrees with
a byte-wise (`memcmp`) comparison of the original — but apply it to a
**transient plan intermediate** feeding one operator's hash table or sort
buffer, not to a durable index. Three different subsystems, one recurring
idea: shrink the physical representation without changing what a comparison,
a hash, or a group-by would observe.

## 7.27.3 The full mechanism, read from the source

### 7.27.3.1 Entry point: `Compress`, and the four operators it targets

`CompressedMaterialization::Compress` (`src/optimizer/compressed_materialization.cpp:443-476`)
is the single dispatch point every visited node passes through:

```cpp
// src/optimizer/compressed_materialization.cpp:443-456
void CompressedMaterialization::Compress(unique_ptr<LogicalOperator> &op) {
	if (TopN::CanOptimize(*op)) { // Let's not mess with the TopN optimizer
		return;
	}

	switch (op->type) {
	case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
	case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
	case LogicalOperatorType::LOGICAL_DISTINCT:
	case LogicalOperatorType::LOGICAL_ORDER_BY:
		break;
	default:
		return;
	}
	// ...
```

Two things worth calling out immediately:

1. **The eligible operator set is exactly four, and only four**: hash
   aggregation (`GROUP BY`), comparison joins (equi-/range-joins — the
   join family that builds a hash table), `DISTINCT`, and `ORDER BY`. This
   answers the "which operators does this target" question directly from
   source rather than from inference: it is *not* every operator with some
   notion of a materializing sink — window functions, `UNNEST`, set
   operations, and plain projections never go through `Compress` at all,
   because they never reach this `switch`'s matching arms. The class
   header's own doc comment agrees: *"The `CompressedMaterialization`
   optimizer compresse[s] columns using projections, based on available
   statistics, but only if the data enters a materializing operator"*
   (`src/include/duckdb/optimizer/compressed_materialization.hpp:89-90`) —
   and in this codebase, "a materializing operator" means precisely these
   four.
2. **It defers to `TOP_N`.** If the node in front of it is one `TopN::CanOptimize`
   would fuse into a `LogicalTopN` (an `ORDER BY` immediately followed by a
   constant `LIMIT`/`OFFSET` — see §7.6.4 in the overview chapter), compressed
   materialization backs off entirely and leaves that `LogicalOrder` alone.
   `TOP_N` (pass #32) already runs *earlier* in the fixed pipeline than
   `STATISTICS_PROPAGATION` (pass #34), so by the time this check runs, an
   eligible `ORDER BY + LIMIT` has normally already been fused into a
   `LogicalTopN` node, which isn't one of the four cases in the `switch`
   anyway and would already fall through the `default: return;` branch
   harmlessly. The explicit `TopN::CanOptimize` guard at the very top exists
   as a second, defense-in-depth check for a `LogicalOrder` that is still
   `TopN`-eligible for whatever reason at this later point in the pipeline
   (for instance, a `LogicalOrder` reintroduced or left over from an
   intervening rewrite) — so this pass never second-guesses or interferes
   with a plan shape `TOP_N` either has already claimed, or is still entitled
   to claim, for its own simpler, bounded-heap execution strategy.

### 7.27.3.2 File organization: one file per operator, plus one shared core

The task-relevant file layout answers the "aggregation-and-joins, or
uniformly?" question with a concrete, checkable fact — `ls` on
`src/optimizer/compressed_materialization/` shows exactly four `.cpp`
files, one per targeted operator type, not one per "materializing sink in
general":

```
src/optimizer/compressed_materialization.cpp        (shared core: eligibility,
                                                       type-specific compress/
                                                       decompress-expression
                                                       builders, projection
                                                       wiring)
src/optimizer/compressed_materialization/
    compress_aggregate.cpp         (LOGICAL_AGGREGATE_AND_GROUP_BY)
    compress_comparison_join.cpp   (LOGICAL_COMPARISON_JOIN)
    compress_distinct.cpp          (LOGICAL_DISTINCT)
    compress_order.cpp             (LOGICAL_ORDER_BY)
```

Each of the four files implements exactly one method declared in the shared
header (`CompressAggregate`, `CompressComparisonJoin`, `CompressDistinct`,
`CompressOrder`), whose job is to work out, for its own operator's specific
shape, *which columns are candidates* and *which bindings must be excluded*
— then hand off to the shared machinery in `compressed_materialization.cpp`
(`CreateProjections`, `TryCompressChild`, `CreateCompressProjection`,
`CreateDecompressProjection`) to actually build the wrapping projections.
This is the general "one thin per-operator front end feeding one shared
back end" shape a few other multi-file passes in this chapter also use
(filter pushdown's thirteen per-operator handlers in `src/optimizer/pushdown/`
are the largest example) — here scaled down to four.

### 7.27.3.3 The generic compress/decompress-projection machinery

The shape every one of the four front ends produces is the same, and it's
worth drawing once, generically, before working through what makes each
operator's front end different:

```
Before compressed materialization:

        ┌───────────────────────────┐
        │  materializing op          │   (aggregate / join / distinct / order)
        │  reads column X : WIDE_T   │
        └──────────────┬─────────────┘
                        │
                  child operator
                  (produces X : WIDE_T)

After (only if TryCompressChild finds something worth shrinking):

        ┌───────────────────────────┐
        │  LogicalProjection          │   "decompress projection"
        │  X := restore(X') : WIDE_T  │   (CreateDecompressProjection)
        └──────────────┬─────────────┘
        ┌──────────────▼─────────────┐
        │  materializing op           │   now reads X' : NARROW_T internally
        │  (bindings replaced,        │
        │   ColumnBindingReplacer     │
        │   fixes up every reference  │
        │   above this point)         │
        └──────────────┬─────────────┘
        ┌──────────────▼─────────────┐
        │  LogicalProjection           │   "compress projection"
        │  X' := shrink(X) : NARROW_T  │   (CreateCompressProjection)
        └──────────────┬─────────────┘
                        │
                  child operator
                  (still produces X : WIDE_T)
```

`CreateProjections` (`compressed_materialization.cpp:478-497`) is the
orchestrator: for every child of the materializing operator (aggregates and
`DISTINCT`/`ORDER BY` have one child; comparison joins have two, indices
`{0, 1}`), it calls `TryCompressChild` to attempt compression of that
child's exposed bindings; if *any* binding on *any* child actually got
compressed, it calls `CreateDecompressProjection` exactly once, above the
whole operator, to restore everything that needs restoring. `TryCompressChild`
(499-528) walks every one of that child's output bindings, skips any binding
marked `can_compress[i] == false` (populated by `CMChildInfo`'s constructor,
`compressed_materialization.cpp:29-38`, from the `referenced_bindings` set
each front end computes — bindings used inside a non-column-reference
expression, e.g. a computed join predicate or a computed `ORDER BY` key, are
excluded, because compressing the underlying column out from under an
expression that still needs its *original* value would be wrong), and for
every remaining candidate binding calls `GetCompressExpression` — the type
dispatcher covered next.

#### Rebinding after a projection is inserted: only the ancestors

When a compress or decompress projection is added, its output columns get
new bindings. (A *binding* is the `(table_index, column_index)` pair that
other operators use to name a column.) Every expression that used the old
binding must be pointed at the new one. A `ColumnBindingReplacer` does this.

Running the replacer over the whole plan (`replacer.VisitOperator(*root)`)
would visit every operator, once for every projection added. A plan with
many aggregates would pay this cost many times: with 150 `count(DISTINCT)`
aggregates, `EXPLAIN` takes 9.8 s that way instead of 0.73 s. So both
`CreateCompressProjection` and `CreateDecompressProjection` call
`RebindAncestors(replacer)` (`compressed_materialization.cpp:570` and
`676`). The idea is simple. A
table index belongs to exactly one operator. So an expression can only
refer to a binding of the compressed operator if it sits *above* that
operator. Those operators are exactly the `ancestors` chain that the
statistics propagator passes in (§7.27.0):

```cpp
// src/optimizer/compressed_materialization.cpp:365-383
void CompressedMaterialization::RebindAncestors(ColumnBindingReplacer &replacer) {
	// Only the materializing operator consumes compressed values. Its ancestors see the
	// restored output and must retain their expression types, including explicit collations.
	ColumnBindingReplacer parent_replacer;
	for (auto &replacement : replacer.replacement_bindings) {
		parent_replacer.replacement_bindings.emplace_back(replacement.old_binding, replacement.new_binding);
	}
	for (auto &ancestor : ancestors) {
		auto &ancestor_op = ancestor.get();
		if (replacer.stop_operator && replacer.stop_operator.get() == ancestor_op.get()) {
			continue;
		}
		if (ancestor_op.get() == ancestors.back().get().get()) {
			replacer.VisitOperatorBindings(*ancestor_op);
		} else {
			parent_replacer.VisitOperatorBindings(*ancestor_op);
		}
	}
}
```

Read it in three steps:

1. `ancestors.back()` is the materializing operator itself (the last node
   pushed on the path). Only this operator reads the *compressed* values.
   So only this operator gets the full `replacer`, which changes both the
   binding and the type (for example `BIGINT` to `UTINYINT`).
2. Every operator higher up gets `parent_replacer`. It is a copy that keeps
   only the old and new bindings, with no type. Those operators see the
   *restored* output of the decompress projection, so their expression
   types must stay as they were. This matters for types that carry extra
   information, such as a `VARCHAR` with `COLLATE NOCASE`. The test below
   describes the risk: the rebinding must *"preserve collations above the
   restored output boundary"*. If the collation were lost, a later
   `EXCEPT` or `INTERSECT` could compare the strings without `NOCASE`.
3. The `stop_operator` (the newly inserted projection) is skipped.

`CreateCompressProjection` follows the same rule. It asks for a type change
only when the type really changed:

```cpp
// src/optimizer/compressed_materialization.cpp:556-560
if (new_type == child_info.types[col_idx]) {
	replacement_bindings.emplace_back(old_binding, new_binding);
} else {
	replacement_bindings.emplace_back(old_binding, new_binding, new_type);
}
```

In the same spirit, `Compress` resolves the types of the operator being
compressed
(`op->ResolveOperatorTypes();`, line 458) instead of the types of the whole
plan from the root.

Two tests guard this behavior:

- `test/sql/setops/compression_expression_collation.test` runs set
  operations over an ordered, collated input, once with the pass on and
  once with `SET disabled_optimizers='compressed_materialization'`. Both
  runs must agree:

  ```sql
  SELECT count(*) FROM ((SELECT s COLLATE NOCASE FROM (SELECT s,rowid FROM l ORDER BY rowid))
                        INTERSECT (SELECT s FROM r));
  ----
  3
  ```

  Here `l` holds `'A','A','B',NULL,NULL` and `r` holds `'a','b',NULL`. With
  `NOCASE`, `'A'` matches `'a'` and `'B'` matches `'b'`, so three distinct
  values survive (`a`, `b` and `NULL`). The file also checks with
  `EXPLAIN` that the `ORDER BY` input is still compressed
  (`__internal_compress_string` ... `__internal_decompress_string`).
- `test/optimizer/compressed_materialization_many_aggregates.test`
  (described in §7.27.3.6) has a query whose comment says *"references
  above the aggregate must survive the rebinding around (de)compress
  projections"*.

### 7.27.3.4 Per-type eligibility: the exact decision logic

`GetCompressExpression` (`compressed_materialization.cpp:691-720`) is the
single dispatcher every compressible binding passes through:

```cpp
// src/optimizer/compressed_materialization.cpp:691-720 (elided)
unique_ptr<CompressExpression> CompressedMaterialization::GetCompressExpression(
    unique_ptr<Expression> input, const BaseStatistics &stats) {
	const auto &type = input->GetReturnType();
	if (type.IsAggregateState()) {
		return nullptr;
	}
	if (stats.GetType().id() == LogicalTypeId::VARIANT && /* wrapped-VARIANT-comparator case */) {
		return GetVariantCompress(...);
	}
	if (type != stats.GetType()) {
		return nullptr;
	}
	if (type.IsIntegral()) {
		return GetIntegralCompress(std::move(input), stats);
	}
	switch (type.id()) {
	case LogicalTypeId::VARCHAR:  return GetStringCompress(std::move(input), stats);
	case LogicalTypeId::GEOMETRY: return GetGeometryCompress(std::move(input), stats);
	case LogicalTypeId::VARIANT:  return GetVariantCompress(std::move(input), stats);
	default:                      return nullptr;
	}
}
```

Four families are handled, and everything else — every other logical type
in the system, including every nested type (`STRUCT`, `LIST`, `MAP`,
`ARRAY`, `UNION`), `BLOB`, `BOOLEAN`, floating point, `DECIMAL`, every
temporal type, `ENUM`, `UUID` — falls through the `default:` and returns
`nullptr`, meaning "leave this column exactly as it is." This is the answer
to "does it apply uniformly, or only to specific value shapes": it is a
closed, explicit allow-list of four representations, checked precisely, not
a general-purpose "make anything smaller" mechanism.

**Integral types — `GetIntegralCompress` (722-752).** The workhorse case,
and the one with the most machinery behind it:

```cpp
// src/optimizer/compressed_materialization.cpp:722-752
unique_ptr<CompressExpression> CompressedMaterialization::GetIntegralCompress(
    unique_ptr<Expression> input, const BaseStatistics &stats) {
	const auto &type = input->GetReturnType();
	if (GetTypeIdSize(type.InternalType()) == 1) {
		return nullptr;                          // already 1 byte, nothing to shrink
	}

	LogicalType cast_type;
	Value range_value;
	Value min;
	if (!CMHelper::GetIntegralOffsetCompressInfo(context, type, stats, cast_type, min, range_value)) {
		return nullptr;                          // not enough statistics
	}

	if (GetTypeIdSize(cast_type.InternalType()) == GetTypeIdSize(type.InternalType())) {
		return nullptr;                          // the "smaller" type isn't actually smaller
	}

	const auto value_preserving_cast_type = CMHelper::GetIntegralCastType(type, cast_type, stats);
	if (value_preserving_cast_type.IsValid()) {
		auto result = CMHelper::CreateIntegralCastCompress(context, input, value_preserving_cast_type, stats);
		if (result) {
			return result;
		}
	}

	return CMHelper::CreateIntegralFunctionCompress(std::move(input), type, cast_type, min, range_value, stats);
}
```

Unpacking the helpers it calls, in order:

- **`GetIntegralOffsetCompressInfo`** (`compressed_materialization.cpp:225-249`,
  declared as a `CMHelper` static) first handles the degenerate "column is
  entirely `NULL`" case by short-circuiting to `UTINYINT` with a zero range
  (lines 228-234) — this is exactly what powers the all-`NULL` Parquet
  column test walked through in §7.27.6. Otherwise it requires
  `NumericStats::HasMinMax(stats)` (235-237: no min/max known → give up,
  return `false`), computes `range = max - min` via
  `GetIntegralRangeValue` (190-209 — evaluated by actually *running* the
  bound `SubtractFunction` through the `ExpressionExecutor`, not by naive
  arithmetic — more on why this matters in §7.27.7), and then **tries to
  cast that range down to `UBIGINT`** (241-245): `range_value.DefaultTryCastAs(LogicalType::UBIGINT)`.
  If that try-cast fails — the range itself doesn't fit in 64 unsigned bits
  — the whole function returns `false` and no compression is attempted for
  that column. Only once a `UBIGINT`-sized range is in hand does
  `GetIntegralOffsetType` (211-223) pick the narrowest unsigned type whose
  maximum still covers that range: `UTINYINT` for range ≤ 255,
  `USMALLINT` for ≤ 65535, `UINTEGER` for ≤ 2^32−1, else `UBIGINT`.
- **`GetIntegralCastType`** (`284-311`) is the "can we get this for free"
  check: given the source type and the narrow `offset_type`
  `GetIntegralOffsetType` picked, it asks whether a *plain, value-preserving
  cast* — no runtime offset-subtraction needed at all — from the source
  type directly to a same-width signed or unsigned integer type would
  losslessly represent every value the column's statistics say can occur.
  `ValuePreservingCastFits` (266-282) checks this concretely: cast the
  value to the candidate type and back, and compare for equality — a true
  roundtrip check, not a range heuristic. If **both** the statistics' `min`
  and `max` roundtrip cleanly through the narrower type, that type is
  returned; DuckDB tries the source type's own signedness first, then the
  opposite signedness, before giving up.
- If a value-preserving cast type was found, `CreateIntegralCastCompress`
  (313-328) builds a plain `BoundCastExpression` (via
  `BoundCastExpression::AddDefaultCastToType`, then `CMUtils::MarkCast` —
  §7.27.4 explains why it's tagged) and classifies the result as
  `CompressedMaterializationType::CAST`. If no lossless direct cast exists
  — the *range* fits the narrow type but the *raw values themselves*
  don't (the canonical case: values cluster around 9000–9099, a range of
  99 that fits `UTINYINT`, but 9000 itself doesn't fit in 0–255) —
  `CreateIntegralFunctionCompress` (330-353) builds an offset-subtraction
  expression instead, calling one of the internal
  `__internal_compress_integral_*` scalar functions, and classifies the
  result as `CompressedMaterializationType::FUNCTION`.

  The header documents the practical difference between these two tags
  directly:

  ```cpp
  // src/include/duckdb/optimizer/compressed_materialization.hpp:26-32
  enum class CompressedMaterializationType : uint8_t {
  	INVALID = 0,
  	//! Use the opaque internal (de-)compress functions
  	FUNCTION = 1,
  	//! Use down- and upcasts that are transparent to subsequent optimizers
  	CAST = 2
  };
  ```

  `CAST` is "free" in a second sense too: it produces an ordinary
  `BoundCastExpression`, which every other optimizer pass (constant
  folding, statistics propagation itself, cast-aware pushdown) already
  knows how to reason about — nothing downstream needs to be taught about
  compressed materialization specifically. `FUNCTION` calls an opaque,
  purpose-built internal scalar function that no other pass understands —
  it works, but it's a black box to everything else in the pipeline.

**String types — `GetStringCompress`/`GetStringCompressInfo` (789-836).**
Requires `StringStats::HasMaxStringLength(stats)` (797-799 — no length
statistic, no compression) — with the same all-`NULL` short-circuit as the
integral case (791-795: an all-`NULL` `VARCHAR` column compresses to
`UTINYINT` unconditionally, `max_string_length = 0`). Given a real
max-length statistic, it picks the smallest of `CMUtils::StringTypes()`
(`UTINYINT, USMALLINT, UINTEGER, UBIGINT, UHUGEINT` —
`compressed_materialization_utils.cpp:77-80`) whose byte width strictly
exceeds `max_string_length` (802-807: `max_string_length < GetTypeIdSize(...)`
— one byte of every candidate width is reserved for an explicit length
prefix, so a type only qualifies if it has room for the length byte *plus*
every actual byte of the longest string that can occur). If not even
`UHUGEINT` (16 bytes, 15 usable payload bytes) has room, no compression is
attempted — a genuinely long-`VARCHAR` column is left alone.

**Geometry — `GetGeometryCompress`/`GeometryIsAllPointXY` (838-873).**
The narrowest of the four families by design: it only ever compresses to
one target, `UHUGEINT`, and only when `GeometryIsAllPointXY` (838-851)
confirms every non-`NULL` value in the column is a `POINT` with exactly an
`XY` vertex type (`GeometryStats::GetTypes(stats).HasOnly(GeometryType::POINT, VertexType::XY)`)
— which, per the internal WKB layout, guarantees every value is *exactly*
21 bytes (1 byte-order byte + 4 type-tag bytes + 2 `double`s). A `LINESTRING`, a `POLYGON`, a
three-dimensional `POINT Z`, anything with a different vertex layout — all
fail `GeometryIsAllPointXY` and the column is left untouched.

The packing function does not just trust the statistics. It checks every
value, in release builds too. The reason is in its comment: the statistics
can come from a file, and a file can be wrong.

```cpp
// src/function/scalar/compressed_materialization/compress_geometry.cpp:18-25
uhugeint_t GeometryPointCompress(const string_t &input) {
	// The statistics that lead to this compression can come from a file, so verify that this is a POINT-XY
	const auto data = const_data_ptr_cast(input.GetData());
	if (input.GetSize() != POINT_XY_BLOB_SIZE || Load<uint8_t>(data) != 1 ||
	    Load<uint32_t>(data + sizeof(uint8_t)) != 1) {
		throw InvalidInputException("Expected only 2D POINT geometries based on the column statistics, but "
		                            "encountered a different geometry - the statistics might be corrupt");
	}
```

The value must be 21 bytes long, its byte-order byte must be `1` (little
endian), and its type tag must be `1` (a 2D `POINT`). Because the function
can throw, its error mode is `FunctionErrors::CAN_THROW_RUNTIME_ERROR`
(lines 42 and 120), not `CANNOT_ERROR`. Other
passes use the error mode to decide whether an expression may be moved,
so this is not just a label.

`test/sql/copy/parquet/parquet_wrong_geometry_types.test` reads a broken
GeoParquet file. Its `geospatial_types` statistics say `POINT`, but the
column really holds `GEOMETRYCOLLECTION EMPTY` and `MULTIPOINT EMPTY`
values. Sorting that column makes compressed materialization pack it, so
the query fails with a clear message instead of producing wrong values:

```sql
statement error
SELECT st_astext(g) s, count(*) FROM (SELECT g, i FROM
  'data/parquet-testing/broken/geoparquet_wrong_geometry_types.parquet'
  ORDER BY i DESC) GROUP BY ALL ORDER BY 1;
----
Expected only 2D POINT geometries based on the column statistics
```

The same file, read without the `ORDER BY`, works, and a table of real
points is compressed and sorted correctly.

**`VARIANT` — `GetVariantCompress`/`GetVariantCompressInfo` (875-983).**
The most conditionally-gated of the four, and the one most worth reading
carefully because it's a compounding decision — "can this be compressed at
all" *and*, if so, "can the resulting narrower type be compressed
*again*." `GetVariantCompressInfo` requires, in order:

1. `VariantStats::IsShredded(stats)` — the storage layer has already
   determined this `VARIANT` column's values predominantly share one
   concrete structured type (Chapter 17's *shredding*, in `variant_stats.hpp`'s
   own words: *"we find the most common type among all these values ... and
   store them separately from the rest of the values, in a structured
   way"*).
2. The shredded type must not be nested — `structured_type.IsNested()`
   disqualifies a `VARIANT` shredded onto a `LIST` or `STRUCT` (883-887).
3. `VariantShreddedStats::IsFullyShredded(shredded_stats)` — **every**
   value must fit the shredded type; if even one value needed the
   fallback "untyped" side-channel (a *partially* shredded column), the
   compression is declined outright, because a compressed representation
   built only from the shredded/typed side would have nowhere to put that
   value's real content (889-892).
4. The shredded type itself must be on an explicit allow-list of ~20
   primitive types (893-923) that specifically **excludes** `FLOAT`,
   `DOUBLE`, and `INTERVAL`, with the reasoning spelled out in the source
   comment: *"we require that equal values of the shredded type always
   have identical VARIANT binary representations, e.g., FLOAT/DOUBLE
   ('-0.0' == '0.0') and INTERVAL ('1 month' == '30 days') do not
   qualify."* This is the sharpest, most concrete instance in the whole
   pass of "decline when the compact form's equality wouldn't track the
   original type's actual equality" — `-0.0` and `0.0` compare equal as
   `DOUBLE`s but need not have byte-identical `VARIANT` encodings, so
   compressing (and later re-comparing) the shredded `DOUBLE` value could
   silently produce a different grouping/ordering than comparing the
   original `VARIANT` values directly would.

Once eligible, `GetVariantCompress` (938-983) first casts down to the
shredded type itself — already a real space win, since a `VARIANT` is a
much larger, self-describing encoding than any primitive it shreds — and
then, worth noting as a compounding step, **tries to compress that
shredded type a second time**, recursively reusing the *same* integral-
offset or string-prefix machinery just described (958-977), using the
`VARIANT`'s own typed statistics. A `VARIANT` shredded on `INTEGER` with a
narrow observed range doesn't just become an `INTEGER` — it can become a
`UTINYINT` if the shredded values' range is narrow enough, chaining two
compression decisions into one plan rewrite.

### 7.27.3.5 Join-specific gating: three defensive thresholds

Comparison joins get an extra layer of gating the other three operators
don't, declared directly in the class's private section:

```cpp
// src/include/duckdb/optimizer/compressed_materialization.hpp:99-105
//! Somewhat defensive constants that try to limit when compressed materialization is triggered for joins
//! We only consider compressed materialization for joins when the build cardinality is greater than this
static constexpr idx_t JOIN_BUILD_CARDINALITY_THRESHOLD = 1048576;
//! If the cardinality > threshold, we always perform compressed materialization if there are this many columns
static constexpr idx_t JOIN_BUILD_COLUMN_COUNT_THRESHOLD = 20;
//! If not, we perform compressed materialization if join result cardinality > build cardinality * this threshold
static constexpr double JOIN_CARDINALITY_RATIO_THRESHOLD = 8;
```

and applied in `CompressComparisonJoin` (`compress_comparison_join.cpp:127-147`)
behind a `#ifndef DEBUG` guard — worth quoting because the comment states
the testing philosophy directly:

```cpp
// src/optimizer/compressed_materialization/compress_comparison_join.cpp:127-147
#ifndef DEBUG
	// In debug mode, we always apply compressed materialization to joins regardless of cardinalities,
	// so that it is well-tested. In release mode, we use the thresholds defined in the header
	// If any of the inputs has VARIANT, we skip these checks: compressing is assumed to always be better
	if (!HasVariantType(left_child) && !HasVariantType(right_child)) {
		const auto build_cardinality = right_child.has_estimated_cardinality ? right_child.estimated_cardinality
		                                                                     : right_child.EstimateCardinality(context);
		if (build_cardinality < JOIN_BUILD_CARDINALITY_THRESHOLD) {
			return;
		}
		if (right_child.types.size() < JOIN_BUILD_COLUMN_COUNT_THRESHOLD) {
			const auto join_cardinality = /* ... */;
			const double ratio = static_cast<double>(join_cardinality) / static_cast<double>(build_cardinality);
			if (ratio > JOIN_CARDINALITY_RATIO_THRESHOLD) {
				return;
			}
		}
	}
#endif
```

The reasoning worth spelling out: compressing a join's build-side columns
isn't free — it adds a projection operator and a per-row scalar-function
call on *both* the build and probe paths. For a small build side, that
per-row overhead can outweigh the memory saved. So in release builds the
pass only bothers once the build side (`right_child` — DuckDB's comparison
join always treats its *second* child as the build side here) is estimated
at over 2^20 (1,048,576) rows, **or**, for a build side with fewer than 20
columns, once the join's output-to-build-side cardinality ratio exceeds 8×
(a proxy for "this join is fanning out a lot, so keeping the build side's
hash-table entries small matters even more per probe"). In **debug**
builds, none of this gating applies — compression is always attempted
regardless of estimated size, specifically so the mechanism itself gets
exercised by every ordinary correctness test in the suite, not just by
tests that deliberately construct million-row inputs. This is also why the
comment calls the thresholds "somewhat defensive": they are a coarse,
hand-picked cost heuristic, not a costed alternative-plan search the way
join *ordering* (Chapter 8) is.

Two further, unconditional restrictions on join-key compression, both
worth reading directly rather than summarizing away:

- **`MARK` joins are skipped entirely**, with the reason given inline:
  *"Tricky to get bindings right. RHS binding stays the same even though it
  changes type. Skip for now"* (`compress_comparison_join.cpp:119-122`) — a
  plainly stated, honest limitation rather than a silently-absent case.
- **Join-key compression is only attempted generically when there is
  exactly one join condition** (`compress_comparison_join.cpp:166-167,
  176-178`: *"We only try to compress the join condition cols if there's
  one join condition. Else it gets messy with the stats if one column shows
  up in multiple conditions."*) — with more than one condition, the columns
  are still eligible for compression as ordinary *payload* (non-key)
  columns via the generic per-binding path, just not via the special
  matched-pair path that merges both sides' statistics into one shared
  compressed representation (`compress_comparison_join.cpp:184-206`, using
  `BaseStatistics::Merge` so both sides of the comparison get compressed to
  the *same* narrow type — essential, since comparing two independently
  narrowed representations of what were originally the same wide type could
  silently misorder values compressed against different mins).
- **Only the build side's non-key columns are eligible at all** — every
  binding from the probe side (`left_child` — DuckDB's convention here
  again: the *first* child is probed, the second is built and hashed) that
  isn't one of the merged join-key bindings is explicitly added to
  `referenced_bindings` and therefore excluded (`compress_comparison_join.cpp:221-228`,
  comment: *"We don't want to compress these because they are streamed"*)
  — a direct statement of *why* compressed materialization targets these
  four operators and not, say, a plain streaming projection: only a column
  that is actually going to sit inside a **materialized** structure (a
  build-side hash table, a sort buffer, a group-by hash table, a dedup set)
  benefits from being smaller; a column that merely streams through row by
  row gains nothing from a narrower representation and only pays the
  compress/decompress function-call overhead for no benefit.

### 7.27.3.6 Aggregate- and grouping-set-specific behavior

`CompressAggregate` (`compress_aggregate.cpp:8-133`) differs from the
other three front ends in two ways worth calling out:

- **It tries harder on non-column-reference group expressions.** Where
  `CompressOrder` and `CompressDistinct` only ever compress a bare
  `BoundColumnRefExpression` generically and otherwise just *exclude* the
  referenced binding, `CompressAggregate`'s `try_compress_group` lambda
  (lines 38-51) attempts to compress a **computed** `GROUP BY` expression
  directly, using either that expression's own propagated `group_stats`
  entry or — a special case worth noting — the statistics behind a
  `VARIANT` "comparator wrapper" expression (`GetVariantWrapperStats`,
  line 64, described in §7.27.3.4's `VARIANT` discussion: the binder wraps
  `VARIANT` values compared for grouping/ordering in a normalizing
  function, and this lets compressed materialization see straight through
  that wrapper to the underlying shredded statistics). This asymmetry is a
  real, stated implementation choice, not an oversight the source hides —
  it simply means an `ORDER BY <expression>` involving a `VARIANT` column
  wrapped this way is *not* compressed at the expression level the way an
  equivalent `GROUP BY` would be.
- **Duplicate `GROUP BY` columns disable compression for the whole
  aggregate outright:**

  ```cpp
  // src/optimizer/compressed_materialization/compress_aggregate.cpp:12-19
  for (const auto &group : groups) {
  	if (group->GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
  		continue;
  	}
  	auto &colref = group->Cast<BoundColumnRefExpression>();
  	if (group_binding_set.find(colref.Binding()) != group_binding_set.end()) {
  		return; // Duplicate group - don't compress
  	}
  	group_binding_set.insert(colref.Binding());
  }
  ```

  A defensive, whole-operator bail-out rather than an attempt to handle the
  duplicate carefully.
- **Grouping sets get their own binding-info path**
  (`compress_aggregate.cpp:115-117`: `if (!aggregate.grouping_sets.empty() && group_stats[group_idx]) { binding_info.stats = group_stats[group_idx]->ToUnique(); }`)
  — when an aggregate has more than one grouping set (`ROLLUP`/`CUBE`/
  explicit `GROUPING SETS`, lowered by the mandatory rewrite and the
  `GROUPING_SETS` pass described in §7.4's pipeline before compressed
  materialization ever sees the tree), a single output column position can
  be `NULL` in some grouping sets and a real grouped value in others.
  Compressed materialization still compresses it — the grouping-sets test
  in §7.27.6.3 confirms this concretely — but the statistics attached to that
  binding have to come from the aggregate's own per-group `group_stats`
  entry directly, rather than the generic "look this binding up in the
  global `statistics_map`" path every other case uses, because the
  binding's *meaning* (one physical column position standing in for several
  different grouping-set roles) is specific to how `LogicalAggregate`
  represents grouping sets internally.
- **Columns the aggregate never reads are left alone.**

  Some columns of the aggregate's child are not used by any group or
  aggregate. No operator above the aggregate can see them, because an
  aggregate only outputs its groups and aggregates. Compressing them would
  cost planning time and make the plan bigger, for no gain. So the front end
  collects the bindings that the groups and aggregates
  really use, and marks every other child column as not compressible:

  ```cpp
  // src/optimizer/compressed_materialization/compress_aggregate.cpp:94-107
  // Columns that the aggregate does not read cannot be referenced above it, so leave them uncompressed
  column_binding_set_t used_bindings;
  for (const auto &group : groups) {
  	GetReferencedBindings(*group, used_bindings);
  }
  for (const auto &expr : aggregate.expressions) {
  	GetReferencedBindings(*expr, used_bindings);
  }
  auto &child_info = info.child_info[0];
  for (idx_t binding_idx = 0; binding_idx < child_info.bindings_before.size(); binding_idx++) {
  	if (used_bindings.find(child_info.bindings_before[binding_idx]) == used_bindings.end()) {
  		child_info.can_compress[binding_idx] = false;
  	}
  }
  ```

  The case that shows why is issue #26865: one `count(DISTINCT)` per
  column. The distinct-aggregate rewrite turns each one into its own
  branch over the same input. Without this rule, each branch's aggregate
  would compress all the input columns, not just its own. The regression test is
  `test/optimizer/compressed_materialization_many_aggregates.test`. It
  builds a 100-row table `t` with 16 short `VARCHAR` columns `c0`..`c15`
  (`'v' || (range % n)`) and a group column `g`, then checks the results
  of three query shapes:

  ```sql
  -- ungrouped: one rewrite branch per distinct aggregate, all over the same compressed input
  SELECT count(DISTINCT c0), count(DISTINCT c1), ..., count(DISTINCT c15) FROM t;
  ----
  2  3  4  5  6  7  8  9  10  11  12  13  14  15  16  17

  -- grouped: branches are joined back on the (compressed) group key
  SELECT g, count(DISTINCT c0), ..., count(DISTINCT c15) FROM t GROUP BY g ORDER BY g;

  -- references above the aggregate must survive the rebinding around (de)compress projections
  SELECT g, cd FROM (SELECT g, count(DISTINCT c15) AS cd FROM t GROUP BY g) sub
  WHERE cd > 0 ORDER BY g DESC LIMIT 2;
  ----
  g3  17
  g2  17
  ```

  The test checks results, not timing. It guards the two changes together:
  skipping unread columns here, and rebinding only the ancestors
  (§7.27.3.3).

## 7.27.4 The runtime compress/decompress functions themselves

Every `FUNCTION`-classified compression resolves, at the physical-execution
level, to one of a small family of scalar functions registered in the
catalog under names like `__internal_compress_integral_utinyint` and
`__internal_decompress_string`, implemented in
`src/function/scalar/compressed_materialization/{compress_integral,compress_string,compress_geometry}.cpp`.
Three mechanisms, each worth seeing at the byte level:

**Integral offset-encoding** (`compress_integral.cpp`). Compress is
`value - min`, narrowed to the target width:

```cpp
// src/function/scalar/compressed_materialization/compress_integral.cpp:17-23
template <class INPUT_TYPE, class RESULT_TYPE>
struct TemplatedIntegralCompress {
	static inline RESULT_TYPE Operation(const INPUT_TYPE &input, const INPUT_TYPE &min_val) {
		D_ASSERT(min_val <= input);
		return UnsafeNumericCast<RESULT_TYPE>(input - min_val);
	}
};
```

with dedicated specializations for `hugeint_t`/`uhugeint_t` inputs that
only keep the subtraction's low 64 bits (`(input - min_val).lower`) —
correct precisely *because* `GetIntegralOffsetCompressInfo` already proved
the range fits in a `UBIGINT` before this function is ever generated for
that column (§7.27.3.4). Decompress reverses it, `min + widen(value)`:

```cpp
// src/function/scalar/compressed_materialization/compress_integral.cpp:109-114, 123-128
template <class INPUT_TYPE, class RESULT_TYPE>
struct TemplatedIntegralDecompress {
	static inline RESULT_TYPE Operation(const INPUT_TYPE &input, const RESULT_TYPE &min_val) {
		return min_val + UnsafeNumericCast<RESULT_TYPE, INPUT_TYPE>(input);
	}
};
// ...
template <class INPUT_TYPE>
struct TemplatedIntegralDecompress<INPUT_TYPE, uhugeint_t> {
	static inline uhugeint_t Operation(const INPUT_TYPE &input, const uhugeint_t &min_val) {
		return min_val + uhugeint_t(0, input);
	}
};
```

— the `hugeint_t`/`uhugeint_t` specializations explicitly construct a
128-bit value with the upper half zeroed (`hugeint_t(0, input)`) before
adding, rather than relying on an implicit widening conversion. §7.27.7
explains exactly why that explicit-construction specialization exists: it's
literally the fix for one of this section's two fuzzer-discovered bugs.

**String prefix-packing** (`compress_string.cpp`). For a string known (from
`max_string_length`) to fit within `sizeof(RESULT_TYPE) - 1` bytes, the
compress function reverses the string's bytes into the integer's storage,
byte-swaps for endianness, and stores the original length in the byte that
ends up most significant:

```cpp
// src/function/scalar/compressed_materialization/compress_string.cpp:30-48 (elided)
template <class RESULT_TYPE>
inline RESULT_TYPE StringCompressInternal(const string_t &input) {
	RESULT_TYPE result;
	// ... reversed memcpy of up to sizeof(RESULT_TYPE) bytes of the string's content ...
	result_ptr[0] = UnsafeNumericCast<data_t>(input.GetSize());
	return BSwapIfBE(result);
}
```

The byte reversal is not an incidental implementation detail — it's the
entire point, and it is the same trick Chapter 16 documents for ART's key
encoding: reversing the string's bytes into a fixed-width integer means an
ordinary **unsigned-integer comparison** of the packed value agrees with a
byte-wise (`memcmp`) comparison of the original string prefix, so sorting,
grouping, and hashing on the compressed form give the same answer sorting
on the real string would, without ever touching the original bytes again
inside the materializing operator. A dedicated 1-byte (`UTINYINT`) "mini"
path (`MiniStringCompress`, lines 56-73) handles the case where even the
length-prefix byte alone has no room for any content byte — it packs
`length + first_byte` into that single byte, which only works, and is only
ever generated, when the column's statistics have already proven every
string is 0 or 1 bytes long.

**Geometry point-packing** (`compress_geometry.cpp`). For the narrow
POINT-XY case, the compress function reaches past the WKB blob's fixed
5-byte header directly to the 16 coordinate bytes (two IEEE-754 `double`s)
and reverses them into a `UHUGEINT`, with the same comment explaining the
same reasoning as the string case:

```cpp
// src/function/scalar/compressed_materialization/compress_geometry.cpp:30-31 (elided)
// Pack the coordinates into a UHUGEINT, reversing the byte order so that an unsigned integer
// comparison of the result matches the lexicographic (memcmp) byte comparison.
```

**None of these can be called from SQL directly.** Every one of them is
registered with `CMUtils::Bind` as its `BindScalarFunctionInput` callback,
and that callback's entire body is:

```cpp
// src/function/scalar/compressed_materialization_utils.cpp:82-86
// LCOV_EXCL_START
unique_ptr<FunctionData> CMUtils::Bind(BindScalarFunctionInput &input) {
	throw BinderException("Compressed materialization functions are for internal use only!");
}
// LCOV_EXCL_STOP
```

This is exactly what `test/optimizer/compressed_materialization.test_slow`'s
opening `statement error` block demonstrates (§7.27.6.1): the functions exist
as ordinary catalog entries (needed so serialization/deserialization,
`EXPLAIN`, and `duckdb_functions()`-style introspection all work uniformly),
but a user query invoking one always fails to *bind*. The optimizer itself
never goes through binding for these calls at all — `CMHelper::CreateIntegralFunctionCompress`
and its string/geometry counterparts construct the `BoundFunctionExpression`
directly, bypassing the catalog lookup and bind step entirely, which is why
`CMUtils::Bind` unconditionally throwing causes no problem for the optimizer
itself.

The `CAST`-classified path, by contrast, doesn't call a bespoke internal
function at all — it reuses DuckDB's own ordinary default-cast machinery
(`BoundCastExpression::AddDefaultCastToType`), and `CMUtils::MarkCast`
(`compressed_materialization_utils.cpp:16-24`) only *relabels* the
resulting cast expression's function definition and bind callback (to the
same always-throwing `CMUtils::Bind`, and to a distinguishing name,
`__internal_compressed_materialization_cast`) so that later code —
specifically `CMUtils::GetExpressionType`, used when reconstructing a
serialized plan or classifying an expression during a later optimizer pass
— can still recognize "this particular cast was inserted by compressed
materialization" without needing an entirely separate expression kind. This
is the literal mechanism behind the header's "transparent to subsequent
optimizers" comment (§7.27.3.4): it *is* a real `BoundCastExpression` in
every way that matters to any other pass; only the marking is special.

Other code needs to recognise compress expressions too, so `CMUtils` has
two small helpers (`compressed_materialization_utils.hpp:26-29`):

- `GetWrappedInput(expression)` returns the input that a compress,
  decompress or marked-cast expression wraps, or `nullptr` for any other
  expression. The `EXPLAIN (SQL)` exporter in `src/planner/sql_export/`
  uses it to look through these internal calls when it writes a plan back
  out as SQL.
- `CompressesInput(expression)` returns `true` for a compress function,
  and for a marked cast only when the cast makes the type *smaller*
  (`compressed_materialization_utils.cpp:56-71`).

Filter pushdown uses `CompressesInput` in
`src/optimizer/pushdown/pushdown_projection.cpp:25-35`. The comment there
states the rule: *"A compress expression only holds where it was placed,
so a filter must not inline it into a position below"*. For example,
`status_code - 9000` fits in a `UTINYINT` only for the rows the
statistics describe. If a filter is pushed below the place where those
statistics hold, the same expression could see rows with other values.
So a filter that reads a compressed column stays on top of the
compress projection (`stays_on_top`, line 173-176).
`test/optimizer/pushdown/filter_pushdown_compressed_projection.test`
builds this case: a filter derived from one join is pushed toward
another join whose compressed `g` column only holds above its own filter
on `g`. The query must return `95495000 95495000` twice: once with
`cte_filter_pusher` and `build_side_probe_side` disabled (this forces the
risky plan shape) and once with all optimizers on.

## 7.27.5 A full worked example

Take a plausible `orders` table where a `status_code` column is declared
`BIGINT` (8 bytes) for schema-evolution headroom, but in this particular
database's data, statistics show it only ever takes values in a tight band,
say 9000–9099 (100 distinct codes):

```sql
CREATE TABLE orders(order_id BIGINT, status_code BIGINT, amount DOUBLE);
-- status_code's propagated min/max: 9000 / 9099

SELECT status_code, SUM(amount)
FROM orders
GROUP BY status_code;
```

Before compressed materialization, the hash-aggregate's grouping-key column
is materialized at its declared width — 8 bytes per row, for every row that
enters the hash table, regardless of how many distinct groups actually
exist:

```
┌───────────────────────────────────────┐
│ LogicalAggregate                        │
│   GROUP BY status_code : BIGINT (8B)    │   <- 8 bytes/row in the grouping key
│   aggr: SUM(amount)                     │
└──────────────────┬──────────────────────┘
┌──────────────────▼──────────────────────┐
│ LogicalGet orders                        │
│   status_code stats: min=9000, max=9099  │
└──────────────────────────────────────────┘
```

When `STATISTICS_PROPAGATION` reaches this `LogicalAggregate` node
(bottom-up, so `orders`'s own scan statistics are already known),
`CompressAggregate` walks `status_code`'s binding and calls
`GetIntegralCompress`. `GetIntegralOffsetCompressInfo` computes
`range = 9099 - 9000 = 99`, which fits `UBIGINT` trivially, so
`GetIntegralOffsetType(99)` picks `UTINYINT` (99 ≤ 255). But
`GetIntegralCastType` then checks whether a **plain cast** would work —
`ValuePreservingCastFits(9000, BIGINT, UTINYINT)` fails immediately (9000
doesn't fit in `0..255` at all, roundtrip or not) — so no free cast is
available, and `CreateIntegralFunctionCompress` builds the offset-subtract
path instead: `FUNCTION`-classified, calling `__internal_compress_integral_utinyint(status_code, 9000)`
going in, `__internal_decompress_integral_bigint(status_code', 9000)` coming
back out. (This is exactly the `t_offset` case in
`compressed_materialization.test_slow`, walked through in full in
§7.27.6.1 — the same source-level decision, applied to a plausible
business column instead of a synthetic `range()` expression.)

After compressed materialization:

```
┌────────────────────────────────────────────────────┐
│ LogicalProjection  (decompress)                      │
│   status_code := 9000 + widen(status_code')          │  BIGINT again, for
└──────────────────────┬────────────────────────────────┘  everything above
┌──────────────────────▼────────────────────────────────┐
│ LogicalAggregate                                        │
│   GROUP BY status_code' : UTINYINT (1B)                 │   <- 1 byte/row now
│   aggr: SUM(amount)                                      │
└──────────────────────┬────────────────────────────────┘
┌──────────────────────▼────────────────────────────────┐
│ LogicalProjection  (compress)                            │
│   status_code' := status_code - 9000  (as UTINYINT)      │
└──────────────────────┬────────────────────────────────┘
┌──────────────────────▼────────────────────────────────┐
│ LogicalGet orders                                        │
└──────────────────────────────────────────────────────────┘
```

The hash-aggregate's own grouping-key storage — the thing actually
materialized inside the operator's hash table, the subject of Chapter 14's
discussion of aggregation internals — shrinks from 8 bytes to 1 byte per
row: an 8× reduction in exactly the data structure this pass targets, for
exactly as long as the aggregate needs it, invisible the moment the
decompress projection restores `status_code` to `BIGINT` for the final
output. `EXPLAIN` on the un-optimized-vs-optimized tree is the only place
this rewrite is ever visible — the query's result set is bit-for-bit
identical either way, which is exactly what the paired
`disabled_optimizers = 'compressed_materialization'` blocks in the real
test suite exist to prove (§7.27.6.1).

## 7.27.6 Test walkthroughs

### 7.27.6.1 `test/optimizer/compressed_materialization.test_slow` — the main suite

This file is organized as a sequence of increasingly specific checks, and
it's worth walking in the order it appears:

**The functions cannot be called directly.** The very first check —

```sql
statement error
select __internal_compress_string_utinyint('L')
----
Binder Error: Compressed materialization functions are for internal use only!
```

— is the direct SQL-level confirmation of `CMUtils::Bind` unconditionally
throwing (§7.27.4). It's placed first deliberately: before anything else is
tested, the file establishes that these internal functions are a genuinely
private implementation detail, not part of the surface a user could
(accidentally or deliberately) invoke.

**"internal issue 1576" — a `row_number()` + filter + `ORDER BY` +
`LIMIT` correctness check over 500,000 rows** (`t0`, columns `a`/`b` both
`range % 400000`). This isn't a plan-shape test at all — no `EXPLAIN`, no
regex — it's a pure result-correctness regression, checking that whatever
compression compressed materialization applies to `a`/`b` on their way into
the `ORDER BY` doesn't perturb `row_number()`'s numbering or the outer
filter's result. The comment format ("internal issue 1576") marks this as
a fix-then-regression-test pair, in the same spirit as the two fuzzer tests
covered in §7.27.7 — a bug was found, and this query is what exposed it.

**The `test_issue_6136` correlated-subquery tests, run single-threaded.**
Two scalar-subquery queries (`r`/`b` tables) that group, order, and `LIMIT 1`
inside a correlated subquery, explicitly run with `set threads=1` because,
per the file's own comment, "they are order dependent." These exist to
confirm that compressing a `GROUP BY`/`ORDER BY` key inside a correlated
subquery doesn't change which row a tie-breaking `LIMIT 1` happens to pick
— exactly the kind of subtle interaction a narrower internal representation
could, in principle, disturb if the compression weren't strictly
order-preserving.

**The `CAST`-vs-`FUNCTION` counting tests.** These are the cleanest,
most directly-traceable-to-source cases in the file, and they use the
sqllogictest regex-counting idiom (a `<REGEX>:(.*CAST.*){N}` pattern counts
occurrences rather than checking exact text) precisely because the exact
column names and table indices a compressed plan produces aren't worth
pinning down byte-for-byte — only the *count* of compress/decompress
occurrences matters for proving the mechanism fired the expected number of
times.

- `t1 := range i FROM range(10)` (values 0–9), `EXPLAIN SELECT i FROM t1 ORDER BY 10-i`,
  expecting exactly **3** `CAST` occurrences. Both `i` itself (as the
  selected payload column) and the computed order key `10-i` (range 1–10)
  have raw values that fit directly inside a `UTINYINT`'s domain
  (`ValuePreservingCastFits` succeeds trivially for values 0–10), so both
  compress via the free `CAST` path rather than the `FUNCTION` path — no
  `__internal_*` call appears anywhere in the plan, only plain casts,
  matching the comment "we should see compress twice (in the ORDER BY
  expression and payload) and decompress once (just the payload)."
- `t_offset := (range + 300)::INTEGER i FROM range(10)` (values 300–309),
  `EXPLAIN SELECT i FROM t_offset ORDER BY i`, expecting the regex
  `.*__internal_decompress.*__internal_compress.*` — i.e., **at least one**
  genuine internal function call, not just casts. This is precisely the
  `GetIntegralCastType` failure case worked through in full in §7.27.5:
  the range (9) fits `UTINYINT` trivially, but the raw values (300–309)
  do not fit `UTINYINT`'s domain at all, so no direct cast can represent
  them — only the offset-subtract `FUNCTION` path can, and the test's own
  comment states the point directly: *"offset compression still uses the
  internal functions when values do not fit a same-width direct cast."*
- The `test` table (`(range+7)%4 i`, `(range+7)%11 j`, i∈[0,3], j∈[0,10])
  drives four further counting tests, whose comments are worth reading as
  a set: a query with two nested `ORDER BY ... OFFSET 1` clauses over the
  same `i`/`j` expects exactly 4 `CAST`s ("we can only compress once, then
  do both ORDER BYs, then decompress" — i.e., compressing `i` and `j` once
  each and reusing that single compressed representation across both
  nested sort operators, rather than compressing/decompressing at each
  `ORDER BY` boundary independently); a `GROUP BY i ORDER BY i` query
  expects exactly 3 (compressing only the group key `i`, never the
  aggregated value `j`, per the comment "we can only compress the group
  (i), not the value being summed (j)"); and a query that projects the
  *same* column `j` twice under two aliases (`j AS j1, j AS j2`) before
  sorting by both expects **6** `CAST`s instead of an ideal 4 or fewer,
  with the comment stating the limitation plainly: *"We can't deal with
  duplicate projections (yet) so this should see 3 compresses instead of
  1."* This last one is the cleanest fully-traceable case of the four:
  compressed materialization operates strictly per `ColumnBinding`, with no
  cross-column common-subexpression awareness (that's a different pass's
  job — `COMMON_SUBEXPRESSIONS`, Chapter 7's §7.16, or `DUPLICATE_GROUPS`
  — neither of which this pass consults), so when the *same* underlying
  value is exposed twice under two distinct output bindings, it gets
  compressed twice, independently, instead of once, shared. Reconstructing
  the exact intermediate plan shape behind the first two counting tests
  precisely enough to justify the counts down to the instruction goes
  beyond what the four compressed-materialization files alone show — it
  also depends on how the binder itself represents a computed `ORDER BY`
  key as a hidden projected column (Chapter 5's territory) — so this book
  states the counts and the mechanism they're demonstrating (compress once,
  reuse across chained materializing operators reading the same
  already-compressed binding) at the confidence the source directly
  supports, rather than fabricating a byte-level derivation beyond it.
- `SELECT DISTINCT i, j FROM test ORDER BY i, j` expects exactly 4 —
  `DISTINCT` compressing both `i` and `j` generically (2 compresses) and
  one decompress projection restoring both above it (2 decompresses).

**The all-NULL-Parquet-column test.** Two Parquet files, each with one
real (hashed) column and two entirely-`NULL` columns (one `VARCHAR`, one
`BIGINT`), read together with `union_by_name=true`. The query confirms
(via `stats(j)`, `stats(k)`) that both all-`NULL` columns really do report
`has_null=true, has_no_null=false` — i.e., real, propagatable statistics —
and then checks that ordering by the real column produces a plan matching
`.*__internal_decompress.*CAST.*__internal_compress.*CAST.*`: **both**
all-`NULL` columns get compressed, one via the `FUNCTION` path (the
all-`NULL` `VARCHAR` short-circuits to `UTINYINT` per
`GetStringCompressInfo`'s `CanHaveNoNull()` check) and one via the `CAST`
path (the all-`NULL` `BIGINT` short-circuits to `UTINYINT` per
`GetIntegralOffsetCompressInfo`'s equivalent check, and `0` trivially casts
losslessly both ways). This directly foreshadows
`test/parquet/parquet_null_compressed_materialization.test_slow` (§7.27.6.5),
which does the same thing at a scale meant to matter for memory rather
than just plan shape.

**TPC-H.** `SF=0.01`, confirming Q1 still gets a `PERFECT_HASH_GROUP_BY`
physical operator (an orthogonal, physical-plan-level optimization for
small-domain grouping keys, not this pass), and that ordering `lineitem`
by `l_shipdate` produces a compressed plan
(`.*(CAST|__internal_decompress).*(CAST|__internal_compress).*`). The
file's final block is the paired-result check every test file in this
series ends with — described in general terms in `03-expression-rewriter-strings-and-regex.md`'s
note on the `nosort <label>` idiom: two `query ... nosort q0` blocks with
no literal expected output, checked byte-for-byte **against each other**.
Here, the first runs with compressed materialization enabled (the default)
and the second runs after `set disabled_optimizers to 'compressed_materialization'`
— proving, for real `lineitem` data, that the compressed and uncompressed
plans produce **identical** query results, which is the whole-file
correctness anchor every plan-shape regex check in this file is allowed to
assume.

### 7.27.6.2 `compressed_materialization_geometry.test`

Five tables, each engineered to land on one side or the other of
`GeometryIsAllPointXY`'s exact boundary:

- **`pts`** — all-`POINT`-`XY` values with some `NULL`s mixed in →
  compresses. The test confirms both directions of the roundtrip appear
  (`__internal_compress_geometry_point` in one `EXPLAIN`, `__internal_decompress_geometry_point`
  in a second, separately-run `EXPLAIN` against the same query — not a
  paired `nosort` check here, just two independent regex assertions).
- **`mixed`** — a `LINESTRING` sneaks in among otherwise-`POINT`-`XY`
  values → **must not** compress (`<!REGEX>`). This is `GeometryIsAllPointXY`'s
  `HasOnly(GeometryType::POINT, VertexType::XY)` check failing the moment a
  single non-`POINT` geometry type appears anywhere in the column's
  statistics.
- **`xyz`** — `POINT Z` (three-dimensional points) → **must not** compress,
  because the vertex type is `XYZ`, not `XY` — the fixed-21-byte assumption
  the packing function checks (with a runtime error, see §7.27.3.4)
  simply doesn't hold for a `POINT` with an extra coordinate.
- **`emptymix`** — `POINT EMPTY` values mixed in with ordinary `POINT`-`XY`
  values → **still compresses**, because (per the test's own comment) an
  empty point is stored as a single `XY` vertex with `NaN` coordinates, so
  its WKB blob is still exactly 21 bytes and its geometry/vertex-type
  classification is still `POINT`/`XY` — `GeometryIsAllPointXY` has no
  reason to exclude it.
- The **order-preservation** checks at the end of the file
  (`SELECT g::VARCHAR FROM pts WHERE g IS NOT NULL ORDER BY g, payload`,
  repeated once compressed and once with
  `SET disabled_optimizers = 'compressed_materialization'`) are the
  file's own version of the `nosort` paired-check idiom, confirming
  directly that sorting on the *compressed* `UHUGEINT` representation of a
  `POINT`'s coordinates produces the exact same row order as sorting on the
  original `GEOMETRY` values — i.e., that the byte-reversal trick described
  in §7.27.4 really does preserve comparison order, including for the
  `NaN`-coordinate empty-point case, which sorts consistently (after every
  finite point) either way.

### 7.27.6.3 `compressed_materialization_grouping_sets.test`

A three-row table (`a INTEGER NOT NULL, b VARCHAR NOT NULL, v INTEGER`)
grouped by `ROLLUP(a, b)` — the smallest possible grouping-sets shape (one
plain group, one partial rollup, one grand total). The `EXPLAIN` check
confirms the regex `.*CAST.*__internal_decompress.*CAST.*__internal_compress.*`
appears — both `a` and `b` get compressed (an `INTEGER` narrows via offset
subtraction or a direct cast depending on its actual range; `b`, a
low-cardinality `VARCHAR`, narrows via string prefix-packing), directly
confirming the `compress_aggregate.cpp:115-117` grouping-sets-specific
binding-info path from §7.27.3.6 actually fires rather than being dead
code. The second half of the file — a `ROLLUP(k)` aggregate feeding a
window function (`COUNT(k) OVER (ORDER BY k NULLS FIRST ROWS UNBOUNDED
PRECEDING)`) — carries its own explicit warning as an inline comment:
*"Windows that need an order should specify one; compressed materialization
can change the hash aggregate iteration order."* This is a limitation
worth taking at face value rather than glossing over: compressing a
grouping key can change the *physical* row order a hash aggregate happens
to produce its groups in (a narrower key can hash or bucket differently
than the original), so any query that implicitly depends on a hash
aggregate's *undefined* iteration order — rather than an explicit `ORDER
BY` — is not guaranteed stable across this optimization, and the test
exists specifically to make sure the *window function* in this query has
its own explicit `ORDER BY` rather than relying on incidental ordering.

### 7.27.6.4 `compressed_materialization_variant.test`

Four tables, each shredded on a different primitive type (or, for one,
deliberately *not* fully shredded), read back after a `CHECKPOINT` +
`restart` specifically so the `VARIANT` shredding statistics are the real,
persisted ones the storage layer computed — not statistics reconstructed
in-memory, which matters because `VariantStats::IsShredded`/`IsFullyShredded`
are genuinely storage-computed facts about the checkpointed data, not
something a fresh in-memory table would necessarily carry the same way.

- **`tv`** (shredded on `INTEGER`, with `NULL`s) — compresses via
  `__internal_compress_integral_*` (the shredded `INTEGER`'s narrow range
  compresses further, chaining exactly as §7.27.3.4 describes), and its
  decompress path is confirmed to cast back up through `AS VARIANT`
  explicitly in a *separate* `EXPLAIN` regex check for the same query — the
  file checks compress and decompress as two independent assertions here,
  not as a `nosort`-paired byte-identical check.
- **`ts`** (shredded on short `VARCHAR`s) — chains into string-prefix
  compression (`__internal_compress_string_*`), directly demonstrating that
  the `VARIANT` compression path's "try to compress the shredded type
  further" step (§7.27.3.4) applies the string family exactly the same way
  it would for a plain `VARCHAR` column.
- **`td`** (shredded on `DATE`) — compresses via a **plain cast** (`AS DATE`
  appears in the regex, no `__internal_*` call) — `DATE` is a 4-byte
  integral-backed type already narrow enough that shredding down to it from
  `VARIANT` is itself the whole win; no further integral-offset narrowing
  applies (a `DATE`'s internal representation is already `INTEGER`-sized,
  and this specific data's date range evidently doesn't clear the "would a
  narrower-than-`INTEGER` type actually help" bar `GetIntegralCompress`
  checks).
- **`tm`** (deliberately mixed types — some rows `VARCHAR`, some `INTEGER`)
  — **must not** compress at all (`<!REGEX>:.*(__internal_compress|AS
  VARIANT).*`), because this column is only *partially* shredded:
  `VariantShreddedStats::IsFullyShredded` fails, and
  `GetVariantCompressInfo` declines outright, per §7.27.3.4's point 3. The
  query still returns correct results (a mix of integers and the literal
  string `'str'`), just via the ordinary, uncompressed `VARIANT` path.

Two further checks in this file are worth flagging as precise, deliberate
confirmations of the exact invariant §7.27.3.4's `VARIANT` allow-list
exists to protect: `SELECT v FROM tv WHERE v IS NOT NULL ORDER BY v, payload`
sorts by the raw `VARIANT` column *itself* (not a payload column) and the
comment states plainly *"the variant binary representation does not order
like the shredded type"* — i.e., ordering by the actual `VARIANT` value
must use `VARIANT`'s own (uncompressed) comparison semantics, never a
shortcut through the shredded type's ordering, and this query is not one
compressed materialization touches at all for that reason (the `VARIANT`
column here is the sort *key* itself, not a payload being carried through
a sort keyed on something else). And `variant_typeof(v)` surviving the
round trip (`INT32` / `VARIANT_NULL` counts) confirms the compress/decompress
cycle doesn't lose the distinction between a genuine SQL `NULL` and a
`VARIANT`-encoded null value — both collapse to the same shredded-type
`NULL` on the way in (per the source comment at
`compressed_materialization.cpp:929-932`: *"Both SQL NULL and the VARIANT
null value become SQL NULL when casting to the shredded type ... and they
are indistinguishable at the top level of a VARIANT column, so this is
lossless"*) and the final decompress step reconstructs the correct
`VARIANT_NULL` tag rather than merely a generic SQL `NULL`.

### 7.27.6.5 `test/parquet/parquet_null_compressed_materialization.test_slow`

This is the all-`NULL`-column test from §7.27.6.1, scaled up until the
memory savings are the actual point of the test rather than an incidental
plan-shape detail. Two 600,000-row Parquet files are written, each with one
real `pk` column and **400** entirely-`NULL` `VARCHAR` columns; the test
then sets `memory_limit='1.5GB'` and joins two copies of this file (as the
build side) against three copies (as the probe side) with
`ANY_VALUE(COLUMNS(*))`. The comment does the arithmetic directly: at full
`VARCHAR` width, the (roughly 1.2M-row) build side's 400 columns would need
on the order of 8 GB just for the null-mask/pointer overhead of 400
all-`NULL` `VARCHAR` columns; compressed down to `UTINYINT` (the
`GetStringCompressInfo` all-`NULL` short-circuit from §7.27.3.4, applied
400 times), the same data needs on the order of 0.5 GB — comfortably inside
the 1.5 GB budget. The `EXPLAIN ANALYZE` regex
(`(.*internal_compress_string.*){10,}`) only asserts "at least 10"
occurrences rather than exactly 400, because the profiler's rendered plan
output truncates a 400-column operator's expression list — but the test's
real assertion is implicit and behavioral: **the query has to complete at
all** under a memory limit that would be impossible to satisfy without this
compression, which is a much stronger check than any plan-shape regex could
be on its own.

## 7.27.7 The two fuzzer-discovered regressions, in depth

Both of these tests live under `test/fuzzer/duckfuzz/`, and both trace back
to real fixes from March 2024 — small, surgical patches that are worth
reading directly (this checkout's history still contains them) because
each one exposes a genuinely different flavor of "how would a mechanical,
type-driven eligibility check get a complex-type edge case wrong."

### 7.27.7.1 `compressed_materialization_hugeint.test` — a 128-bit widening bug in decompression, not eligibility

```sql
create table all_types as select * exclude(small_enum, medium_enum, large_enum) from test_all_types();

SELECT DISTINCT TRY_CAST(COLUMNS(*) AS HUGEINT) FROM all_types AS t51(...) ORDER BY c14 DESC NULLS FIRST
SELECT DISTINCT TRY_CAST(COLUMNS(*) AS UHUGEINT) FROM all_types AS t51(...) ORDER BY c14 DESC NULLS FIRST
```

`test_all_types()` is DuckDB's standing fixture table specifically
constructed so that, per column, its rows include that column's type's own
minimum and maximum representable value — and it has genuine `hugeint`
and `uhugeint` columns
(`src/function/table/system/test_all_types.cpp:31-32`) whose min/max rows
really are `NumericLimits<hugeint_t>::Minimum()`/`Maximum()` and their
unsigned equivalents. `TRY_CAST(COLUMNS(*) AS HUGEINT)` forces **every**
column of the fixture — around fifty of them — into `HUGEINT`, so the
resulting `all_types` projection includes columns whose propagated min/max
genuinely spans the entire 128-bit range, alongside the `DISTINCT` (a
dedup) and `ORDER BY` (a sort) operators this pass targets directly.

The bug this test was written to catch, per its own `git log` history
(`dcdd65c30d`, "fix hugeint compressed materialization"), was not in the
*eligibility* logic at all — `GetIntegralOffsetCompressInfo`'s try-cast of
the computed range down to `UBIGINT` (§7.27.3.4) already correctly declines
to compress a `HUGEINT` column whose *range* (not just its min/max
individually) exceeds 64 unsigned bits, so a column genuinely spanning the
full `HUGEINT` domain is never wrongly judged "safe to compress" in the
first place. The actual bug was in **decompression**, for the columns
*narrower* than the full range that legitimately did get compressed. Before
the fix, `TemplatedIntegralDecompress` was a single, generic function
template computing `UnsafeNumericCast<RESULT_TYPE>(min_val + input)` for
every `(INPUT_TYPE, RESULT_TYPE)` pair, including `RESULT_TYPE = hugeint_t`
or `uhugeint_t` — meaning the narrow compressed offset (say a `uint32_t`)
had to be implicitly converted up to a 128-bit type *as part of the
addition itself*, before the result was ever explicitly widened by
anything the compressed-materialization code controlled. The fix (visible
directly in this checkout at `compress_integral.cpp:116-128`) replaces the
single generic template with type-specific specializations for
`hugeint_t`/`uhugeint_t` results that construct the 128-bit value
*explicitly*, upper half zeroed, before adding:

```cpp
template <class INPUT_TYPE>
struct TemplatedIntegralDecompress<INPUT_TYPE, uhugeint_t> {
	static inline uhugeint_t Operation(const INPUT_TYPE &input, const uhugeint_t &min_val) {
		return min_val + uhugeint_t(0, input);
	}
};
```

The lesson this test enshrines: the *decision* logic (should we compress
this column, and to what width) was already careful about 128-bit ranges
from the start — the try-cast-to-`UBIGINT` guard exists precisely because
128-bit ranges can't always be represented in the 64-bit offset types this
pass narrows down to. What the fuzzer actually found was a separate,
downstream hazard: even *correctly-decided* compression can still be
undone incorrectly if the code path that reconstructs the wide value from
the narrow one leaves any part of that reconstruction to an implicit,
generic numeric-widening path rather than an explicit, type-specific one —
128-bit integers are exactly the type family where "the compiler's
generic implicit conversion did the right thing" is the least safe
assumption to make silently. This test running `DISTINCT` and `ORDER BY`
together, over every column of `test_all_types()` simultaneously, is
precisely the kind of broad, structurally-simple-but-combinatorially-wide
query a fuzzer generates and a human is unlikely to write by hand — which
is exactly the class of case this pass's narrow, per-type decision tree is
hardest to hand-verify exhaustively for.

### 7.27.7.2 `compressed_materialization_json.test` — losing a logical-type alias across decompression, not a nested-type problem

```sql
require json
query I
select distinct json_array();
----
[]
```

The test's own description states the point directly: *"Compressed
Materialization optimizer should respect extension types."* It's worth
being precise about what kind of bug this actually was, because it's easy
to assume — reasonably, given how central "does the mechanism handle
nested/complex types safely" is to this whole pass — that a `JSON` value
tripped up compressed materialization because JSON is variable-shaped and
potentially deeply nested. That is *not* what happened here, and the fix
(`62859dd61b`, "fix fuzzer issue 1406") shows exactly why: DuckDB's `JSON`
type is, physically, a `VARCHAR` carrying a logical-type alias — the
eligibility check in `GetCompressExpression` (§7.27.3.4) dispatches purely
on `LogicalTypeId`, and `JSON`'s type ID *is* `VARCHAR`, so it is — entirely
correctly, from the eligibility check's point of view — judged compressible
by the exact same string-prefix-packing path any other short `VARCHAR`
column would use. `json_array()` (an empty array, serialized as the
2-character string `"[]"`) is short enough to hit the string-compression
path (potentially even the 1-byte "mini" path for very short strings,
§7.27.4) without any trouble at all.

The actual bug was entirely in **restoring the type after decompression**.
Before the fix, `GetStringDecompress` built the restored expression using
`decompress_function.return_type` — the *generic* scalar function's own
declared return type, which is plain `VARCHAR` — rather than the specific
logical type that particular binding actually needed to be, `JSON`
(`VARCHAR` plus the `JSON` alias). The fix threads the caller's actual
`result_type` through explicitly instead:

```cpp
// the fix, as it appears in this checkout, compressed_materialization.cpp:1043-1056
unique_ptr<Expression> CompressedMaterialization::GetStringDecompress(unique_ptr<Expression> input,
                                                                      const LogicalType &result_type,
                                                                      const BaseStatistics &stats) {
	// ...
	bound_function.SetReturnType(result_type);   // the caller's real type, not the generic function's own
	// ...
}
```

So the honest framing of what this test proves is narrower, and more
interesting, than "JSON is nested and that's hard": **the eligibility
decision for a value never asks what that value's type is *aliased* or
*presented* as — only what it physically *is* — which is correct and by
design (§7.27.3.4's dispatcher matches on `LogicalTypeId`, the physical
identity, not on any alias). The corresponding risk that decision creates
is downstream: once a value is compressed and later decompressed, the
*restored* expression has to be given back the exact original logical
type — alias and all — rather than whatever generic type the internal
decompression function happens to declare for itself. Get that one detail
wrong, and a `JSON` column silently becomes a plain `VARCHAR` column
somewhere in the middle of the plan — invisible to any test that doesn't
specifically check `typeof()`/output formatting, exactly the kind of defect
a value-level fuzzer (which does check exact output formatting) is well
suited to catch and a human, skimming an `EXPLAIN`, might not be. This is a
genuinely different category of edge case from the `HUGEINT` bug: that one
was an arithmetic-widening hazard inside the compress/decompress *value*
computation; this one was a metadata-propagation hazard in the
*expression-tree bookkeeping* around it — two different ways "the type is
more than its physical representation suggests" can bite a pass whose
entire premise is reasoning about physical representations.

## 7.27.8 Edge cases and limitations, gathered

Pulling together what the source and the test suite both establish as
explicit, acknowledged boundaries of this pass rather than incidental gaps:

- **The eligible operator set is closed and small**: `GROUP BY`,
  comparison joins' build sides, `DISTINCT`, `ORDER BY` — nothing else,
  and `TopN`-eligible `ORDER BY + LIMIT` shapes are deliberately skipped so
  as not to interfere with the `TOP_N` pass.
- **The eligible value-type set is closed and small**: integral types
  (above 1 byte), `VARCHAR`, `GEOMETRY` restricted to non-empty-or-empty
  `POINT`-`XY` values only, and `VARIANT` restricted to *fully* shredded,
  non-nested, primitive-typed columns whose shredded type is itself on a
  further allow-list that excludes `FLOAT`/`DOUBLE`/`INTERVAL` specifically
  because those types can represent the same logical value with different
  binary encodings. Every other type — every nested type outright, `BLOB`,
  booleans, floating point directly, `DECIMAL`, temporal types other than
  through `VARIANT` shredding, `ENUM`, `UUID` — is left completely
  untouched; this is a closed allow-list, not a general mechanism that
  merely happens to decline unsupported cases gracefully.
- **No cross-column duplicate awareness**: the same underlying value
  exposed under two different output bindings (`SELECT j AS j1, j AS j2`)
  gets compressed twice, independently, rather than once and shared — an
  explicitly acknowledged "(yet)" limitation in the test suite itself.
- **`MARK` joins are skipped entirely**, and multi-condition comparison
  joins only compress their join-key columns as ordinary payload, not via
  the paired, statistics-merged key-compression path — both stated as
  deliberate scope-narrowing in inline comments rather than silently
  handled edge cases.
- **A hash aggregate's iteration order is not something this pass
  preserves** — compressing a grouping key can change a hash aggregate's
  incidental row-production order, so any query relying on that
  order without an explicit `ORDER BY` (most concretely, a window function
  computed directly over a `GROUP BY`'s output) needs its own explicit
  ordering, independent of this pass.
- **Join-key compression only merges statistics across exactly one
  comparison condition** — additional conditions fall back to per-column,
  unmerged compression (or no compression) to avoid, in the source's own
  words, statistics getting "messy" across more than one shared column.
- **Release-build joins are gated by size heuristics** (build cardinality
  ≥ 2^20, or a build-side-narrower-than-20-columns cardinality-ratio check)
  precisely because the compression itself has real per-row cost (an extra
  projection, a scalar function call on both build and probe paths) that
  only pays for itself once the build side is large; debug builds
  deliberately disable this gating so the mechanism is exercised
  unconditionally by the ordinary test suite.
- **Complex/nested types don't fail loudly — they simply never enter the
  dispatcher's matching arms**, falling through to `default: return
  nullptr;` in `GetCompressExpression`, and the `VARIANT` case additionally
  demonstrates that even a nominally-eligible *container* type only
  qualifies when the storage layer has already proven (via *full*
  shredding) that every value inside it genuinely reduces to one uniform,
  order/equality-safe primitive representation — a partially-shredded or
  structurally-nested `VARIANT` is treated exactly like any other
  ineligible type, left completely alone rather than partially or
  unsafely compressed.
