# Databricks Runtime — Setup Guide

---

## Prerequisites

| Component | Requirement |
|---|---|
| Databricks workspace | Unity Catalog enabled |
| Databricks Runtime | 14.3+ (Python 3.12) |
| Compute | Serverless (CPU) or general-purpose cluster |
| ThoughtSpot | Cloud instance with REST API v2 enabled |
| ThoughtSpot connection | Databricks connection configured (OAuth or PAT) |
| Python packages | `requests` (pre-installed), `langchain-databricks` (optional, for agent mode) |

---

## 1. ThoughtSpot Connection Setup

In ThoughtSpot:

1. **Data → Connections → Add Connection → Databricks**
2. Configure:
   - Connection name: `Databricks MV` (or your preferred name)
   - Host: Databricks SQL warehouse hostname
   - HTTP Path: SQL warehouse HTTP path
   - Auth: OAuth 2.0 with service principal (recommended) or PAT
3. **Advanced Settings** — set:
   ```
   catalog = your_catalog_name
   ```
   > ⚠️ Required for Unity Catalog — without it, table discovery fails.

---

## 2. Databricks Secrets (Auth)

```bash
# Create scope (one-time)
databricks secrets create-scope thoughtspot

# Store bearer token
databricks secrets put-secret thoughtspot api-token --string-value "<token>"

# (Optional) For automated token refresh
databricks secrets put-secret thoughtspot username --string-value "<client_id>"
databricks secrets put-secret thoughtspot password --string-value "<client_secret>"
```

Retrieve in code:
```python
token = dbutils.secrets.get(scope="thoughtspot", key="api-token")
```

---

## 3. Install Dependencies

```python
%pip install langchain-databricks langchain-core -q
dbutils.library.restartPython()
```

---

## 4. Configuration

```python
from dataclasses import dataclass

@dataclass
class TSConfig:
    base_url: str = "https://your-cluster.thoughtspot.cloud"
    secret_scope: str = "thoughtspot"
    secret_key: str = "api-token"
    connection_name: str = "Databricks MV"
    default_catalog: str = "your_catalog"
```

---

## 5. Running Skills

### Direct (no LLM)
```python
from thoughtspot_agent_skills import ThoughtSpotClient, TSConfig
ts = ThoughtSpotClient(TSConfig(base_url="https://your-cluster.thoughtspot.cloud"))
ts.test_connection()
ts.generate_tml("catalog", "schema", ["table1", "table2"])
ts.import_tml("/tmp/tml/TABLE1.table.tml")
df = ts.search_data("[Revenue] by [Region]", "SALES_MODEL")
```

### Agent mode (LLM picks tools)
```python
from langchain_core.messages import HumanMessage
from databricks_langchain import ChatDatabricks
llm = ChatDatabricks(endpoint="databricks-claude-sonnet-4")
llm_with_tools = llm.bind_tools(ts_tools)
response = llm_with_tools.invoke([HumanMessage(content="What are top departments by pay?")])
```

---

## Differences from Claude Code

| Claude Code | Databricks |
|---|---|
| `~/.claude/thoughtspot-profiles.json` | `TSConfig` dataclass |
| macOS Keychain | `dbutils.secrets.get(scope, key)` |
| `ts` CLI binary | `ThoughtSpotClient` Python class |
| Slash commands | LangChain `@tool` decorators |
| Local file system | `/tmp/` or Volumes |
