---
name: ts-generate-tml
description: Generate ThoughtSpot Table TML files from Unity Catalog table metadata using DESCRIBE TABLE EXTENDED.
---

# Unity Catalog → ThoughtSpot Table TML

Generates valid ThoughtSpot Table TML YAML from Databricks Unity Catalog metadata.

## References

| File | Purpose |
|---|---|
| [../../shared/schemas/thoughtspot-table-tml.md](../../shared/schemas/thoughtspot-table-tml.md) | Table TML structure |
| [../../shared/mappings/ts-databricks/ts-databricks-type-mapping.md](../../shared/mappings/ts-databricks/ts-databricks-type-mapping.md) | Type mapping |
| [../../shared/mappings/ts-databricks/ts-databricks-column-rules.md](../../shared/mappings/ts-databricks/ts-databricks-column-rules.md) | Column classification |

## Workflow

1. Run `DESCRIBE TABLE EXTENDED {catalog}.{schema}.{table}`
2. Parse column names and Databricks SQL types
3. Map types using `ts-databricks-type-mapping.md`
4. Classify columns using `ts-databricks-column-rules.md`
5. Build TML YAML per `thoughtspot-table-tml.md`
6. Write to `{output_dir}/{TABLE_NAME}.table.tml`

## TML Template

```yaml
table:
  name: {TABLE_NAME_UPPER}
  db: {catalog_lower}
  schema: {schema_lower}
  db_table: {table_lower}
  connection:
    name: "{connection_name}"
  columns:
  - name: {COLUMN_NAME_UPPER}
    db_column_name: {column_name_lower}
    properties:
      column_type: {ATTRIBUTE|MEASURE}
      aggregation: {SUM}              # MEASURE only; omit for ATTRIBUTE
      index_type: DONT_INDEX          # measures, dates, FK columns
    db_column_properties:
      data_type: {mapped_ts_type}
  properties:
    spotter_config:
      is_spotter_enabled: false
```

## Critical Rules

1. **Lowercase**: `db`, `schema`, `db_table`, `db_column_name` MUST be lowercase
2. **BOOL not VARCHAR**: BOOLEAN → `data_type: BOOL` (validated against live metadata)
3. **No `aggregation` for ATTRIBUTEs**: API rejects `aggregation: NONE`
4. **Always include `db_column_name`**: Even when equal to `name`
5. **No `joins_with: []`**: Omit the key entirely for standalone tables
