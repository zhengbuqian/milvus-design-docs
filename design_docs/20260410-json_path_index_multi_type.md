# MEP: Support Sort/Bitmap/Hybrid Index Types for JSON Path Index

- **Created:** 2026-04-10
- **Author(s):** @zhengbuqian
- **Status:** Draft
- **Component:** Index | QueryNode | Proxy | DataCoord

## Summary

Extend JSON Path Index to support Sort, Bitmap, and Hybrid index types in
addition to the existing Inverted (Tantivy) index. AUTOINDEX on a JSON path
routes to HYBRID, which dynamically selects Bitmap or Sort based on data
cardinality — matching the behavior of regular scalar columns.

## Motivation

JSON Path Index currently only supports the Inverted (Tantivy) index type.
Different query patterns benefit from different index structures:

- **Sort Index** — efficient range queries (`>`, `<`, `>=`, `<=`, `BETWEEN`)
  on numeric or varchar JSON keys. Binary search with O(log n) lookup.
- **Bitmap Index** — efficient equality and `IN` queries on low-cardinality
  JSON keys (status codes, enums). Roaring bitmaps give compact storage and
  fast set operations.
- **Hybrid Index** — selects Bitmap or Sort based on cardinality at build
  time. The best default when the user doesn't know the data distribution.

Regular scalar columns already support AUTOINDEX → HYBRID; JSON Path Index
now has the same capability.

## Public Interfaces

### CreateIndex API

No proto changes. The existing `index_params` map already supports
`index_type`, `json_path`, and `json_cast_type`.

```python
# Sort index on a numeric JSON key
index_params = {
    "index_type": "STL_SORT",
    "json_path": "metadata['price']",
    "json_cast_type": "DOUBLE",
}

# Bitmap index on a low-cardinality string JSON key
index_params = {
    "index_type": "BITMAP",
    "json_path": "metadata['status']",
    "json_cast_type": "VARCHAR",
}
```

### Compatibility Matrix: `json_cast_type` × `index_type`

| json_cast_type | INVERTED | STL_SORT | BITMAP | HYBRID |
|---|---|---|---|---|
| BOOL          | Y | N | Y | Y |
| DOUBLE        | Y | Y | N | Y |
| VARCHAR       | Y | Y | Y | Y |
| ARRAY_BOOL    | Y | N | N | N |
| ARRAY_DOUBLE  | Y | N | N | N |
| ARRAY_VARCHAR | Y | N | N | N |

Rationale for exclusions:
- **Sort + BOOL**: boolean has only 2 values, sort is meaningless.
- **Bitmap + DOUBLE**: floating point has high cardinality; bitmap degrades.
- **ARRAY_\***: the underlying scalar array types do not support Sort or
  Bitmap indexes today. HYBRID on ARRAY_\* is also excluded to avoid a
  breaking-change upgrade path when Array support is added later.

### AUTOINDEX Routing

`scalarAutoIndex.params.build` default for JSON routes to **HYBRID**:

```
{"json": "HYBRID", ...}
```

### Single Index Per Path

Milvus allows at most one index per `(field_id, json_path)` pair. Creating a
second index on the same path — with either a different cast type or a
different index type — is rejected at CreateIndex time. This eliminates the
need for query-time index selection logic.

## Design Details

### 1. C++ Architecture

All four JSON Path Index variants share a single wrapper class that
pre-processes raw JSON data into typed `FieldData`, then delegates the
actual indexing to an existing scalar index. This reuses the full Build,
Range, In, and serialization logic of `ScalarIndexSort`, `BitmapIndex`, and
`InvertedIndexTantivy` with no modifications to those base classes.

```
JsonScalarIndexWrapper<T, BaseIndex>
  ├── BaseIndex = ScalarIndexSort<double>      → STL_SORT on DOUBLE
  ├── BaseIndex = StringIndexSort              → STL_SORT on VARCHAR
  ├── BaseIndex = BitmapIndex<bool/string>     → BITMAP
  └── BaseIndex = InvertedIndexTantivy<T>      → INVERTED
                                                 (aliased as JsonInvertedIndex<T>)

JsonHybridScalarIndex<T>  inherits  HybridScalarIndex<T>  → HYBRID
```

`JsonHybridScalarIndex` is a separate subclass because `HybridScalarIndex`
builds an internal index (Bitmap or Sort) selected by cardinality at build
time, which requires a custom Build flow.

#### 1.1 JSON data extraction — `ConvertJsonToTypedFieldData<T>`

A single utility in `JsonIndexBuilder.cpp` extracts the JSON path value and
runs type coercion, producing a nullable typed `FieldData<T>` plus a
separate `non_exist_offsets` vector. Rows are classified as:

| JSON value                         | typed-data validity | `non_exist_offsets` |
|------------------------------------|---------------------|---------------------|
| present, castable (e.g. `10.5`)    | valid               | —                   |
| present, not castable (e.g. `"abc"` on DOUBLE) | invalid     | —                   |
| present but JSON null              | invalid             | row included        |
| path missing                       | invalid             | row included        |

`non_exist_offsets` is a strict subset of the invalid rows — it omits
cast-failure rows. This distinction drives EXISTS semantics (see §3).

#### 1.2 Schema context for the base index

Base indexes dispatch on `schema.data_type()`, which for a JSON path index
would naturally be `JSON`. `MakeJsonCastContext()` produces a modified
`FileManagerContext` whose `field_schema.data_type` equals the cast type
(BOOL/DOUBLE/VARCHAR) and whose `nullable` flag is set to `true`. The
wrapper's base sub-object is constructed with this modified context, so the
base index sees a normal nullable typed column and its switch statements
hit the right branch.

To still read the raw JSON binlog (which is required for Build), the
wrapper also keeps a second `MemFileManagerImpl` constructed with the
*original* JSON-typed context.

#### 1.3 EXISTS semantics — `non_exist_offsets`

`valid_bitset_` (or `null_offset_` in `InvertedIndexTantivy`) tracks which
rows can be used for typed comparisons. It *includes* cast-failure rows, so
`IsNotNull()` returns false for them.

`Exists()` uses `non_exist_offsets` instead, which *excludes* cast-failure
rows. Thus `EXISTS json['price']` on `{"price": "abc"}` with a DOUBLE path
index returns true: the path exists, only the cast failed.

At the parser, `json['price'] IS NOT NULL` is rewritten to
`EXISTS json['price']` (and `IS NULL` to `NOT EXISTS`), so users never
observe the `IsNotNull()` behavior directly for JSON paths.

#### 1.4 NotEqual semantics — NotIn override

Brute-force `NotEqual` on JSON treats extraction errors (path missing,
cast failure, JSON null) as **true** — matching "a row that doesn't have a
comparable value is not equal to the probe value". The base scalar
index's `NotIn` instead masks invalid rows to false (correct for regular
nullable columns).

The wrapper overrides `NotIn` to OR the invalid rows back:

```cpp
const TargetBitmap NotIn(size_t n, const T* values) override {
    auto result = BaseIndex::NotIn(n, values);
    result |= BaseIndex::IsNull();
    return result;
}
```

`JsonHybridScalarIndex` applies the same override on top of
`HybridScalarIndex`.

#### 1.5 Exists bitmap caching

`Exists()` returns the full-row bitmap derived from `non_exist_offsets`.
The bitmap is built eagerly — at the end of `BuildWithFieldData`, `Build`,
`LoadEntries` (v3), and `Load` (v2). Because `InvertedIndexTantivy::Count()`
requires the tantivy reader to be ready, the v2 path cannot build the
bitmap from inside `LoadIndexMetas`; the wrapper instead overrides the
`Load(TraceContext, Config)` entry point and computes the bitmap after the
base `Load` finishes. With the bitmap populated at the end of every
ingestion path, `Exists()` itself is a const read and is thread-safe.

`Exists()` returns `TargetBitmap` by value (clone of the cached bitmap).
Callers wrap the result in a `shared_ptr<TargetBitmap>` for their own use,
so a borrowed reference would force a caller-side clone anyway.

#### 1.6 v2 and v3 serialization

Both on-disk formats are supported:

- **v3** (`WriteEntries` / `LoadEntries`): the wrapper's overrides call
  through to the base, then add/read a single additional entry —
  `INDEX_NON_EXIST_OFFSET_FILE_NAME` — for `non_exist_offsets`.
- **v2** (`Serialize` / `Load` → `LoadIndexMetas`): only meaningful for
  the InvertedIndexTantivy base. The wrapper's `Serialize` reimplements
  the base behavior (`null_offset` + tantivy files) and additionally
  serializes `non_exist_offsets`. `LoadIndexMetas` loads the file from
  either the non-sliced or sliced file layout, and falls back to
  `null_offset_` for legacy v2.5.x indexes that predate the separate
  `non_exist_offsets` file. `BuildTantivyMeta` and `RetainTantivyIndexFiles`
  are likewise overridden to keep the tantivy meta in sync.

All v2-specific code is guarded by
`if constexpr (std::is_base_of_v<InvertedIndexTantivy<T>, BaseIndex>)` so
the same wrapper compiles for Sort/Bitmap bases without dead references.

### 2. IndexFactory

`IndexFactory::CreateJsonIndex()` dispatches by `(index_type, cast_type)`:

```
STL_SORT + DOUBLE   → JsonScalarIndexWrapper<double, ScalarIndexSort<double>>
STL_SORT + VARCHAR  → JsonScalarIndexWrapper<std::string, StringIndexSort>
BITMAP   + BOOL     → JsonScalarIndexWrapper<bool, BitmapIndex<bool>>
BITMAP   + VARCHAR  → JsonScalarIndexWrapper<std::string, BitmapIndex<std::string>>
HYBRID   + BOOL|DOUBLE|VARCHAR  → JsonHybridScalarIndex<T>
INVERTED + BOOL|DOUBLE|VARCHAR  → JsonScalarIndexWrapper<T, InvertedIndexTantivy<T>>
                                  (typedef JsonInvertedIndex<T>)
```

### 3. Query Execution

No changes are needed to the expression executor for Range/In/NotIn — the
wrapper implements `ScalarIndex<T>` and the existing dispatch in
`UnaryExpr` / `BinaryRangeExpr` picks it up transparently. `IndexBase`
exposes a virtual `Exists()` that `ExistsExpr` calls uniformly across all
JSON index types, removing the former per-cast-type `dynamic_cast` switch.

### 4. Go Parameter Validation

Each checker gains JSON support:

- `stl_sort_checker.go` — accepts JSON if `json_cast_type ∈ {DOUBLE, VARCHAR}`
  and `json_path` is present.
- `bitmap_index_checker.go` — accepts JSON if `json_cast_type ∈ {BOOL, VARCHAR}`.
- `hybrid_index_checker.go` — accepts JSON if
  `json_cast_type ∈ {BOOL, DOUBLE, VARCHAR}`.

HYBRID remains blocked for direct user specification in `task_index.go` and
is only reachable via AUTOINDEX routing, consistent with regular scalar
columns.

AUTOINDEX default for JSON fields is changed to `HYBRID` in
`autoindex_param.go`.

### 5. Rolling Upgrade Gate

Each QueryNode registers a scalar-index engine version in its session.
DataCoord's `ResolveScalarIndexVersion()` returns the minimum across live
QueryNodes — the highest version *all* QueryNodes can load.

`CurrentScalarIndexEngineVersion` is bumped to **3** for nodes that
support JSON path Sort/Bitmap/Hybrid. In a mixed cluster, DataCoord
`CreateIndex` rejects a JSON-field request with `STL_SORT`, `BITMAP`, or
`HYBRID` if the resolved cluster version is below 3:

```go
if isJsonField && isNewJsonIndexType(indexType) {
    v := indexEngineVersionManager.ResolveScalarIndexVersion()
    if v < 3 {
        return merr.WrapErrParameterInvalidMsg(
            "index type %s on JSON path requires scalar index version >= 3, "+
            "cluster resolved version is %d", indexType, v)
    }
}
```

AUTOINDEX → HYBRID is therefore rejected during rolling upgrade. Users who
need a JSON Path Index during upgrade can explicitly request `INVERTED`.

## Architecture Summary

```
   CreateIndex(index_type=AUTOINDEX, json_path="/price", json_cast_type=DOUBLE)
                                    │
                                    ▼
   ┌────────────── Go Proxy ──────────────────────────────────────────┐
   │ 1. AUTOINDEX → HYBRID via scalarAutoIndex config                 │
   │ 2. {STL_SORT, BITMAP, HYBRID}Checker.CheckTrain validates inputs │
   └──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
   ┌────────────── Go DataCoord ──────────────────────────────────────┐
   │ 1. Version gate: ResolveScalarIndexVersion() >= 3                │
   │ 2. ParseAndVerifyNestedPath: json['price'] → /price              │
   └──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
   ┌────────────── C++ IndexFactory::CreateJsonIndex() ───────────────┐
   │  STL_SORT  → JsonScalarIndexWrapper<T, ScalarIndexSort / StringIndexSort>
   │  BITMAP    → JsonScalarIndexWrapper<T, BitmapIndex<T>>           │
   │  HYBRID    → JsonHybridScalarIndex<T>                            │
   │  INVERTED  → JsonScalarIndexWrapper<T, InvertedIndexTantivy<T>>  │
   │              (alias JsonInvertedIndex<T>)                        │
   │  NGRAM     → NgramInvertedIndex (existing)                       │
   └──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
   ┌────────────── Build Phase ──────────────────────────────────────┐
   │ 1. ConvertJsonToTypedFieldData<T>()                             │
   │    - path present + castable            → valid value           │
   │    - cast failure                       → invalid, not in       │
   │                                           non_exist_offsets     │
   │    - path missing / JSON null / row NULL→ invalid, in            │
   │                                           non_exist_offsets     │
   │ 2. BaseIndex::BuildWithFieldData(typed_data)                    │
   │    - Sort: sorted array + valid_bitset_                         │
   │    - Bitmap: per-value roaring bitmaps + valid_bitset_          │
   │    - Inverted: tantivy index + null_offset_                     │
   │    - Hybrid: cardinality-aware choice between Bitmap and Sort   │
   │ 3. BuildExistsBitset(total_rows) from non_exist_offsets         │
   └─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
   ┌────────────── Query Phase ──────────────────────────────────────┐
   │ Range / In             → ScalarIndex<T> interface               │
   │ NotIn (NotEqual)       → wrapper override: base.NotIn           │
   │                          | base.IsNull (matches brute-force)   │
   │ EXISTS / IS NOT NULL   → IndexBase::Exists() → non_exist_offsets│
   │ IS NULL / NOT EXISTS   → ¬ Exists (parser rewrites)             │
   └─────────────────────────────────────────────────────────────────┘
```

## File Change Summary

| File | Change |
|---|---|
| `internal/core/src/index/JsonScalarIndexWrapper.h` | **New**: generic wrapper for Sort/Bitmap/Inverted bases; defines `JsonInvertedIndex<T>` alias |
| `internal/core/src/index/JsonHybridScalarIndex.h`  | **New**: HYBRID subclass with validity-aware cardinality counting |
| `internal/core/src/index/JsonIndexBuilder.h/.cpp`  | Add `ConvertJsonToTypedFieldData<T>` and `json::IsDataTypeSupported` |
| `internal/core/src/index/JsonInvertedIndex.h/.cpp` | **Deleted**: now a type alias in `JsonScalarIndexWrapper.h` |
| `internal/core/src/index/Index.h`                  | Virtual `TargetBitmap Exists()` (returns-by-value) |
| `internal/core/src/index/IndexFactory.cpp`         | `CreateJsonIndex` routes STL_SORT / BITMAP / HYBRID |
| `internal/core/src/index/Meta.h`                   | Add `INDEX_NON_EXIST_OFFSET_FILE_NAME` |
| `internal/core/src/exec/expression/ExistsExpr.cpp` | Uniform `index->Exists()` dispatch |
| `internal/util/indexparamcheck/stl_sort_checker.go` | Accept JSON for {DOUBLE, VARCHAR} |
| `internal/util/indexparamcheck/bitmap_index_checker.go` | Accept JSON for {BOOL, VARCHAR} |
| `internal/util/indexparamcheck/hybrid_index_checker.go` | Accept JSON for {BOOL, DOUBLE, VARCHAR} |
| `pkg/util/paramtable/autoindex_param.go`           | JSON default: INVERTED → HYBRID |
| `pkg/common/common.go`                             | Bump `Current/MaximumScalarIndexEngineVersion` to 3 |
| `internal/datacoord/index_service.go`              | Version-3 gate on JSON path Sort/Bitmap/Hybrid |

## Test Plan

### C++ unit tests

1. `ConvertJsonToTypedFieldData` — normal values, missing paths, null values,
   cast failures, mixed rows, varchar.
2. `JsonScalarIndexWrapper` with Sort base — Range, In, NotIn, Exists.
3. `JsonScalarIndexWrapper` with Bitmap base — In, Count, IsNotNull, Exists.
4. `JsonHybridScalarIndex` — low-cardinality routing to Bitmap, high
   cardinality routing to Sort, cardinality counting excludes invalid rows,
   Exists semantics.
5. `IndexFactory` dispatch — accepts supported combinations, rejects invalid
   ones.

### Go integration tests

Per-`(cast_type, index_type)` matrix test for create, query
(range/equal/not-equal/exists), before-load, dynamic-field, and
same-path-different-field cases.

### Cross-version compatibility

Index files are byte-compatible between master (pre-feature) and the
refactor branch:

1. Build v2 INVERTED on master → load on refactor branch → queries match.
2. Build v3 Sort/Bitmap/Inverted on refactor branch → restart → reload
   produces identical results.
3. Build v2 INVERTED on refactor branch (scalar version=2) → load on master
   → queries match, except that master applies its own (pre-fix) NotEqual
   semantics (invalid rows excluded) — all other queries are identical.

### Rolling upgrade

1. Mixed cluster (some v2, some v3 QN): CreateIndex JSON+HYBRID rejected
   with clear error.
2. Mixed cluster: CreateIndex JSON+INVERTED succeeds.
3. Fully upgraded cluster: CreateIndex JSON+HYBRID / AUTOINDEX succeeds.

## Rejected Alternatives

### Subclass Sort/Bitmap/Inverted directly

Creating dedicated `JsonSortIndex<T>` / `JsonBitmapIndex<T>` subclasses per
base index. Rejected — duplicates JSON extraction across classes and
produces parallel class hierarchies. The single-wrapper design compiles
against all base indexes with `if constexpr` and delivers the same result
with less code.

### `valid_bitset_` alone for EXISTS

Using the base index's `valid_bitset_` (or `null_offset_`) for EXISTS.
Rejected — cast-failure rows would be excluded by EXISTS, diverging from
brute-force and from the semantics users get with path-level
`IS NOT NULL`. Separate `non_exist_offsets` is required.

### Silently fall back HYBRID + ARRAY_\* to INVERTED

When HYBRID meets an ARRAY_\* cast type, transparently pick INVERTED.
Rejected — the moment Bitmap/Sort gain Array support, HYBRID's behavior
would silently change from INVERTED to Bitmap/Sort, a breaking change for
existing indexes. Explicit rejection today keeps the upgrade path clean.

## References

- JSON Path Index wrapper: `internal/core/src/index/JsonScalarIndexWrapper.h`
- JSON Hybrid: `internal/core/src/index/JsonHybridScalarIndex.h`
- JSON extraction: `internal/core/src/index/JsonIndexBuilder.h/.cpp`
- Scalar bases: `ScalarIndexSort.h`, `StringIndexSort.h`, `BitmapIndex.h`,
  `InvertedIndexTantivy.h`, `HybridScalarIndex.h`
- Index factory: `internal/core/src/index/IndexFactory.cpp`
- Go-side validation: `internal/util/indexparamcheck/`
- AUTOINDEX config: `pkg/util/paramtable/autoindex_param.go`
- Scalar index version: `pkg/common/common.go`,
  `internal/datacoord/index_engine_version_manager.go`
