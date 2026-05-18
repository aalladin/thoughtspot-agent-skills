---
name: ts-convert-from-databricks-mv
description: Convert a Databricks Unity Catalog Metric View into a ThoughtSpot Model TML.
---

# UC Metric View → ThoughtSpot Model

Converts a UC Metric View (WITH METRICS LANGUAGE) into a ThoughtSpot Model TML.
Databricks equivalent of `ts-convert-from-snowflake-sv`.

## References

| File | Purpose |
|---|---|
| [../../shared/schemas/thoughtspot-model-tml.md](../../shared/schemas/thoughtspot-model-tml.md) | Model TML structure |
| [../../shared/schemas/thoughtspot-table-tml.md](../../shared/schemas/thoughtspot-table-tml.md) | Table TML (must exist first) |
| [../../shared/schemas/thoughtspot-formula-patterns.md](../../shared/schemas/thoughtspot-formula-patterns.md) | Formula syntax |
| [../../shared/mappings/ts-databricks/ts-databricks-type-mapping.md](../../shared/mappings/ts-databricks/ts-databricks-type-mapping.md) | Type mapping |

## Concept Mapping

| UC Metric View | ThoughtSpot Model |
|---|---|
| `entities:` (tables) | `model_tables[]` |
| `dimensions:` | `columns[]` with `column_type: ATTRIBUTE` |
| `measures:` (simple) | `columns[]` with `column_type: MEASURE` + aggregation |
| `metrics:` (expressions) | `formulas[]` + `columns[]` with `formula_id` |
| `relationships:` | `model_tables[].joins[]` (inline) |

## Workflow

1. `SHOW CREATE TABLE {catalog}.{schema}.{mv_name}` → get DDL
2. Parse WITH METRICS LANGUAGE YAML block
3. Extract entities → `model_tables[]`
4. Extract dimensions → ATTRIBUTE columns
5. Extract measures → MEASURE columns or formulas
6. Extract relationships → inline joins
7. Ensure Table TMLs exist (run `ts-generate-tml` if needed)
8. Build Model TML
9. Import via `ts-import-tml`

## SQL → ThoughtSpot Formula Translation

| Databricks SQL | ThoughtSpot |
|---|---|
| `SUM(col)` | `sum ( [{TABLE}::{col}] )` |
| `COUNT(DISTINCT col)` | `count_distinct ( [{TABLE}::{col}] )` |
| `AVG(col)` | `average ( [{TABLE}::{col}] )` |
| `CASE WHEN ... END` | `if ... then ... else ...` |
| `COALESCE(a, b)` | `if ( [a] = null ) then [b] else [a]` |
| `col * 100` | `[{TABLE}::{col}] * 100` |

## Critical Rules

1. Table TMLs must be imported BEFORE Model TML
2. Import order: dimensions first, then facts
3. Formula id format: `"formula_{name}"` (spaces preserved)
4. Join on uses bracket syntax: `[TABLE::column]`
5. Set `join_progressive: true` for chasm-trap safety
6. Add `synonyms` + `ai_context` for Spotter quality

## Differences from Snowflake SV

| Snowflake SV | Databricks MV |
|---|---|
| `GET_DDL('SEMANTIC VIEW')` | `SHOW CREATE TABLE` |
| `TABLES ( BASE TABLE ... )` | `entities:` in YAML |
| `DIMENSIONS` / `METRICS` | `dimensions:` / `measures:` within entities |
| Stored procs for API | Python `requests` + `dbutils.secrets` |
