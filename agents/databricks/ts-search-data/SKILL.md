---
name: ts-search-data
description: Query ThoughtSpot worksheets using bracket syntax and return results as pandas DataFrames.
---

# ThoughtSpot Search Data

Query ThoughtSpot using natural language bracket syntax. Returns pandas DataFrames.

## API Endpoint

`POST /api/rest/2.0/searchdata`

## Query Syntax

```
[Measure] by [Dimension]
[Measure1] [Measure2] by [Dimension] [Date].monthly
[Measure] by [Dimension] sort by [Measure] descending top 10
```

### Examples

| Intent | Query |
|---|---|
| Revenue by region | `[Revenue] by [Region]` |
| Monthly trend | `[Net Pay] by [Pay Date].monthly` |
| Top 5 | `[Gross Pay] by [Department Name] sort by [Gross Pay] descending top 5` |
| Filtered | `[Revenue] by [Customer Name] [Region] = 'EMEA'` |

## Payload

```json
{
  "query_string": "[Net Pay] by [Department Name]",
  "logical_table_identifier": "<worksheet_guid>",
  "data_format": "COMPACT",
  "record_offset": 0,
  "record_size": 100
}
```

## Response Parsing Gotcha

`column_names` may be list of strings OR list of dicts:
```python
if isinstance(col_names_raw[0], dict):
    columns = [c["name"] for c in col_names_raw]
else:
    columns = col_names_raw
```

## Worksheet Resolution

Resolve name → GUID via `/api/rest/2.0/metadata/search`:
```python
resp = self._post("/api/rest/2.0/metadata/search", {
    "metadata": [{"type": "LOGICAL_TABLE", "name_pattern": name}],
    "record_size": 5
})
```

## LangChain Wrapper

```python
@tool
def thoughtspot_search(query: str, worksheet: str) -> str:
    """Search ThoughtSpot using bracket syntax like [Measure] by [Dimension]."""
    df = ts.search_data(query, worksheet)
    return df.to_markdown(index=False)
```
