# Databricks Runtime — Agent Skills

Skills for working with ThoughtSpot from Databricks notebooks. Uses Unity Catalog
metadata, Databricks Secrets for auth, and LangChain for agent orchestration.

## Runtime differences from Claude Code / CoCo

| Aspect | Claude Code | CoCo (Snowflake) | Databricks |
|---|---|---|---|
| Auth storage | macOS Keychain | Snowflake secrets | `dbutils.secrets` |
| Compute | Local Python | Snowflake Warehouse | Serverless (CPU) or cluster |
| Metadata source | ThoughtSpot API export | `GET_DDL` / `DESCRIBE` | `DESCRIBE TABLE EXTENDED` |
| Semantic layer | ThoughtSpot Model TML | Snowflake Semantic View | UC Metric View |
| Formula language | ThoughtSpot formulas | SQL (Snowflake dialect) | SQL (Spark/Databricks) |
| Agent framework | Claude Code slash commands | CoCo stored procs | LangChain + ChatDatabricks |
| Token caching | `/tmp/ts_token_*.txt` | Session variable | Databricks Secrets (persistent) |

## Skill inventory

| Skill | Description |
|---|---|
| `ts-generate-tml` | Generate Table TMLs from Unity Catalog `DESCRIBE TABLE` metadata |
| `ts-convert-from-databricks-mv` | Convert a UC Metric View into a ThoughtSpot Model TML |
| `ts-search-data` | Query ThoughtSpot using natural language bracket syntax |
| `ts-import-tml` | Import TML YAML files to ThoughtSpot via REST API |

## Shared references used

- `../shared/schemas/thoughtspot-table-tml.md` — Table TML construction
- `../shared/schemas/thoughtspot-model-tml.md` — Model TML construction
- `../shared/schemas/thoughtspot-formula-patterns.md` — Formula syntax
- `../shared/mappings/ts-databricks/ts-databricks-type-mapping.md` — UC → TS type mapping
- `../shared/mappings/ts-databricks/ts-databricks-column-rules.md` — Column classification rules
