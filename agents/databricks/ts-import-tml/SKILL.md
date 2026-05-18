---
name: ts-import-tml
description: Import ThoughtSpot TML YAML files via REST API with create/update modes and batch support.
---

# ThoughtSpot TML Import

Import TML files via REST API. Supports single and batch with rate limiting.

## API Endpoint

`POST /api/rest/2.0/metadata/tml/import`

## Import Modes

| Mode | `create_new` | When |
|---|---|---|
| Create | `True` | First import |
| Update | `False` | Re-import existing |

## Import Order (Critical)

1. Dimension Table TMLs (no dependencies)
2. Fact Table TMLs (may have `joins_with`)
3. Model TML (references all tables)

## Payload

```json
{
  "metadata_tmls": ["<yaml_string>"],
  "import_policy": "PARTIAL",
  "create_new": true
}
```

## Status Codes

| Code | Meaning |
|---|---|
| `OK` | Success |
| `WARNING` | Imported with warnings (usually benign name-matching) |
| `ERROR` | Failed — check `error_message` |

## Common Errors

| Error | Fix |
|---|---|
| `Tables do not exist` | Import table TMLs first |
| `DataType VARCHAR does not match CDW DataType` | Use `data_type: BOOL` for booleans |
| `Object already exists` | Use `create_new: False` |
| `aggregation: NONE` rejected | Omit `aggregation` field entirely |

## Rate Limiting

- 2-second pause between imports
- Max ~20 imports/minute
- Use `import_policy: "PARTIAL"` to avoid full-batch rollback

## LangChain Wrapper

```python
@tool
def thoughtspot_import(file_path: str, create_new: bool = True) -> str:
    """Import a TML file to ThoughtSpot."""
    result = ts.import_tml(file_path, create_new=create_new)
    return f"{result['status']}: {result['name']}"
```
