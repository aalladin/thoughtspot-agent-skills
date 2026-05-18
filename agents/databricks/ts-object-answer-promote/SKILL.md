---
name: ts-object-answer-promote
description: Promote formulas and parameters from a saved ThoughtSpot Answer into a Model, making them available to all users who search against it. Databricks runtime — uses dbutils.secrets for auth and Python requests for API calls.
---

# ThoughtSpot: Promote Answer Objects to Model (Databricks)

Move formulas and parameters from a saved ThoughtSpot Answer into a Model definition.
Formulas in an Answer are private; once promoted to the Model, they appear in the
search bar for everyone.

This is the Databricks port of the Claude Code `ts-object-answer-promote` skill.
It replaces `ts` CLI calls with direct REST API calls via `requests` and uses
`dbutils.secrets` for authentication.

---

## References

| File | Purpose |
|---|---|
| [../../shared/schemas/thoughtspot-answer-tml.md](../../shared/schemas/thoughtspot-answer-tml.md) | Answer TML structure (formulas, parameters, sets, data source) |
| [../../shared/schemas/thoughtspot-model-tml.md](../../shared/schemas/thoughtspot-model-tml.md) | Model TML structure, formula placement, self-validation checklist |
| [../../shared/schemas/thoughtspot-formula-patterns.md](../../shared/schemas/thoughtspot-formula-patterns.md) | Formula syntax, column references, YAML encoding |
| [../../shared/schemas/thoughtspot-tml.md](../../shared/schemas/thoughtspot-tml.md) | TML parsing: non-printable chars, PyYAML quirks |
| [../references/direct-api-auth.md](../references/direct-api-auth.md) | Databricks Secrets auth pattern |

---

## Prerequisites

- ThoughtSpot secrets configured: `dbutils.secrets.get("thoughtspot", "api-token")`
- Python packages: `requests`, `pyyaml` (both pre-installed on Databricks)
- ThoughtSpot user must have **MODIFY** or **FULL** access on the target Model
- Compute: Serverless (CPU) or any Python cluster

---

## Workflow Overview

```
1.  Authenticate (dbutils.secrets) ......................... auto
2.  Find the Answer (metadata/search) ..................... user input
3.  Export Answer TML (metadata/tml/export) ............... auto
4.  Select formulas to promote ............................ user input
5.  Find target Model (auto-detect from Answer tables[0].fqn) .. auto/user
6.  Check edit permissions ................................ auto
7.  Export Model TML (metadata/tml/export with associated) . auto
8.  Detect duplicate formula/parameter names .............. auto
9.  Map formula column references to Model ................ auto (may ask)
10. Build updated Model TML ............................... auto
11. Checkpoint (review changes) ........................... user confirms
12. Import updated Model TML (metadata/tml/import) ........ auto
13. Verify and report ..................................... auto
```

---

## API Endpoints Used

| Step | Endpoint | Method |
|---|---|---|
| 1 | `/api/rest/2.0/auth/session/user` | GET |
| 2 | `/api/rest/2.0/metadata/search` | POST |
| 3 | `/api/rest/2.0/metadata/tml/export` | POST |
| 5 | `/api/rest/2.0/metadata/search` | POST |
| 7 | `/api/rest/2.0/metadata/tml/export` | POST |
| 12 | `/api/rest/2.0/metadata/tml/import` | POST |

---

## Implementation

### Step 1 — Authenticate

```python
import requests

token = dbutils.secrets.get(scope="thoughtspot", key="api-token")
base_url = "https://your-cluster.thoughtspot.cloud"
headers = {
    "Authorization": f"Bearer {token}",
    "Content-Type": "application/json",
    "Accept": "application/json"
}

# Verify connection
user = requests.get(f"{base_url}/api/rest/2.0/auth/session/user", headers=headers).json()
current_user_id = user["id"]
print(f"Authenticated as: {user['name']}")
```

---

### Step 2 — Find the Answer

```python
def search_answers(name_pattern: str) -> list:
    resp = requests.post(f"{base_url}/api/rest/2.0/metadata/search",
        headers=headers,
        json={"metadata": [{"type": "ANSWER", "name_pattern": name_pattern}]}
    )
    return resp.json()

results = search_answers("Q4 Sales")
for r in results:
    print(f"  {r['metadata_name']} ({r['metadata_id'][:8]}...)")

answer_guid = results[0]["metadata_id"]
answer_name = results[0]["metadata_name"]
```

---

### Step 3 — Export Answer TML

```python
import yaml

resp = requests.post(f"{base_url}/api/rest/2.0/metadata/tml/export",
    headers=headers,
    json={"metadata": [{"identifier": answer_guid}], "export_fqn": True}
)
export_data = resp.json()

# Parse the TML YAML string
edoc = export_data[0]["edoc"]
# Strip non-printable characters before parsing
import re
edoc_clean = re.sub(r'[^\x09\x0A\x0D\x20-\x7E\x85\xA0-\uFFFD]', '', edoc)
tml = yaml.safe_load(edoc_clean)

# Extract key structures
answer = tml.get("answer", {})
formulas = answer.get("formulas", [])
parameters = answer.get("parameters", [])
cohorts = answer.get("cohorts", [])       # sets (not promotable)
tables_section = answer.get("tables", [])

# Data source GUID
data_source_guid = tables_section[0]["fqn"] if tables_section else None
data_source_name = tables_section[0].get("name") if tables_section else None

print(f"Formulas found: {len(formulas)}")
print(f"Parameters found: {len(parameters)}")
print(f"Data source: {data_source_name} ({data_source_guid})")
```

---

### Step 4 — Select Formulas

```python
print(f"Formulas in \"{answer_name}\":")
for i, f in enumerate(formulas, 1):
    auto = " [auto]" if f.get("was_auto_generated") else ""
    print(f"  {i}  {f['name']}{auto}  →  {f['expr']}")

# User selects (e.g., indices)
selected_indices = [0, 1]  # 0-indexed
selected_formulas = [formulas[i] for i in selected_indices]
```

**Check for parameter references:**

```python
def extract_refs(expr):
    return re.findall(r'\[([^\]]+)\]', expr)

params_by_name = {p["name"]: p for p in parameters}
params_to_promote = []

for f in selected_formulas:
    for ref in extract_refs(f["expr"]):
        if ref in params_by_name and ref not in [p["name"] for p in params_to_promote]:
            params_to_promote.append(params_by_name[ref])
```

**Check formula inter-dependencies:**

```python
answer_formula_ids = {f["id"] for f in formulas}
selected_ids = {f["id"] for f in selected_formulas}

for f in selected_formulas:
    for ref in extract_refs(f["expr"]):
        if ref in answer_formula_ids and ref not in selected_ids:
            # Auto-include dependency
            dep = next(x for x in formulas if x["id"] == ref)
            selected_formulas.append(dep)
            selected_ids.add(ref)
            print(f"  Auto-including dependency: {dep['name']}")
```

---

### Step 5 — Find Target Model

```python
# Auto-detect from Answer's data source
resp = requests.post(f"{base_url}/api/rest/2.0/metadata/search",
    headers=headers,
    json={"metadata": [{"type": "LOGICAL_TABLE", "identifier": data_source_guid}]}
)
model_results = resp.json()

if model_results:
    model_name = model_results[0]["metadata_name"]
    model_guid = model_results[0]["metadata_id"]
    print(f"Target Model: {model_name} ({model_guid[:8]}...)")
```

---

### Step 6 — Check Permissions

```python
model_author = model_results[0].get("metadata_header", {}).get("author", "")
is_owner = (model_author == current_user_id)
if not is_owner:
    print(f"Warning: You are not the owner. Import may fail if you lack MODIFY access.")
```

---

### Step 7 — Export Model TML

```python
resp = requests.post(f"{base_url}/api/rest/2.0/metadata/tml/export",
    headers=headers,
    json={
        "metadata": [{"identifier": model_guid}],
        "export_fqn": True,
        "export_associated": True
    }
)
model_export = resp.json()

model_tml = None
model_guid_from_export = None
table_tmls = {}

for item in model_export:
    edoc_clean = re.sub(r'[^\x09\x0A\x0D\x20-\x7E\x85\xA0-\uFFFD]', '', item["edoc"])
    parsed = yaml.safe_load(edoc_clean)
    if "model" in parsed:
        model_tml = parsed
        model_guid_from_export = parsed.get("guid", model_guid)
    elif "table" in parsed:
        table_tmls[parsed["table"]["name"]] = parsed

model = model_tml["model"]
existing_formulas = model.get("formulas", [])
existing_columns = model.get("columns", [])
model_tables = model.get("model_tables", [])

# Build lookups
col_by_name = {c["name"]: c for c in existing_columns}
formula_names_in_model = {f["name"] for f in existing_formulas}
valid_table_names = {t["name"] for t in model_tables}
```

---

### Step 8 — Detect Duplicates

```python
duplicates = [f for f in selected_formulas if f["name"] in formula_names_in_model]
formulas_to_overwrite = []
formulas_to_add = []

for f in selected_formulas:
    if f["name"] in formula_names_in_model:
        # Overwrite existing
        formulas_to_overwrite.append(f)
    formulas_to_add.append(f)

# Parameter duplicates
param_names_in_model = {p["name"] for p in model.get("parameters", [])}
params_to_overwrite = [p for p in params_to_promote if p["name"] in param_names_in_model]
```

---

### Step 9 — Map Column References

```python
def map_references(expr, col_by_name, formula_names_in_model, selected_names):
    """Validate and optionally rewrite formula column references."""
    rewrites = {}
    for ref in extract_refs(expr):
        # Skip if it's a formula reference
        if ref in formula_names_in_model or ref in selected_names:
            continue
        # Skip if TABLE::col format
        if "::" in ref:
            continue
        # Try to resolve bare name to column_id
        match = col_by_name.get(ref)
        if match and "column_id" in match:
            rewrites[f"[{ref}]"] = f"[{match['column_id']}]"
    
    rewritten = expr
    for old, new in rewrites.items():
        rewritten = rewritten.replace(old, new)
    return rewritten

selected_names = {f["name"] for f in selected_formulas}
for f in formulas_to_add:
    f["rewritten_expr"] = map_references(
        f["expr"], col_by_name, formula_names_in_model, selected_names
    )
```

---

### Step 10 — Build Updated Model TML

```python
import copy, re as re_mod

# Infer column_type from expression
AGGREGATE_FUNCS = re_mod.compile(
    r'\b(sum|count|count_distinct|average|min|max|median|stddev|variance|'
    r'cumulative_\w+|moving_\w+|group_aggregate|rank)\s*\(',
    re_mod.IGNORECASE,
)

def make_formula_id(name):
    return f"formula_{name}"

def infer_column_type(expr):
    return "MEASURE" if AGGREGATE_FUNCS.search(expr) else "ATTRIBUTE"

# Build entries
new_formula_entries = []
new_column_entries = []

for f in formulas_to_add:
    fid = make_formula_id(f["name"])
    col_type = infer_column_type(f["rewritten_expr"])
    
    new_formula_entries.append({
        "id": fid,
        "name": f["name"],
        "expr": f["rewritten_expr"]
    })
    
    props = {"column_type": col_type}
    if col_type == "MEASURE":
        props["aggregation"] = "SUM"
    new_column_entries.append({
        "name": f["name"],
        "formula_id": fid,
        "properties": props
    })

# Merge into Model TML
updated = copy.deepcopy(model_tml)
m = updated["model"]

# Remove overwrites
overwrite_names = {f["name"] for f in formulas_to_overwrite}
if overwrite_names:
    m["formulas"] = [x for x in m.get("formulas", []) if x.get("name") not in overwrite_names]
    m["columns"] = [x for x in m.get("columns", []) if x.get("name") not in overwrite_names]

# Append
m.setdefault("formulas", []).extend(new_formula_entries)
m.setdefault("columns", []).extend(new_column_entries)

# Merge parameters
if params_to_promote:
    overwrite_param_names = {p["name"] for p in params_to_overwrite}
    if overwrite_param_names:
        m["parameters"] = [x for x in m.get("parameters", []) if x.get("name") not in overwrite_param_names]
    for p in params_to_promote:
        entry = {"name": p["name"], "data_type": p["data_type"]}
        if "default_value" in p:
            entry["default_value"] = p["default_value"]
        m.setdefault("parameters", []).append(entry)

# Serialize
def _str_representer(dumper, data):
    if '\n' in data or ('{' in data and '}' in data):
        return dumper.represent_scalar('tag:yaml.org,2002:str', data, style='>')  
    return dumper.represent_scalar('tag:yaml.org,2002:str', data)

yaml.add_representer(str, _str_representer)
updated_yaml = yaml.dump(updated, allow_unicode=True, default_flow_style=False)

# Ensure guid is at document root
if not updated_yaml.strip().startswith("guid:"):
    updated_yaml = f"guid: {model_guid_from_export}\n" + updated_yaml
```

---

### Step 12 — Import

```python
resp = requests.post(f"{base_url}/api/rest/2.0/metadata/tml/import",
    headers=headers,
    json={
        "metadata_tmls": [updated_yaml],
        "import_policy": "ALL_OR_NONE",
        "create_new": False
    },
    timeout=120
)
result = resp.json()

status = result[0]["response"]["status"]["status_code"]
if status == "OK":
    print(f"\u2713 Formulas promoted to {model_name}")
    for f in formulas_to_add:
        print(f"    + {f['name']}")
else:
    error = result[0]["response"]["status"].get("error_message", "Unknown")
    print(f"\u2717 Import failed: {error}")
```

---

### Step 13 — Verify

```python
print(f"\n\u2713 Formula promotion complete.")
print(f"  Model: {base_url}/#/data/tables/{model_guid}")
print(f"  Formulas added: {[f['name'] for f in formulas_to_add]}")
if params_to_promote:
    print(f"  Parameters added: {[p['name'] for p in params_to_promote]}")
```

---

## LangChain Tool Wrapper

```python
from langchain_core.tools import tool

@tool
def thoughtspot_promote_formula(answer_name: str, formula_names: str, model_name: str = "") -> str:
    """Promote formulas from a ThoughtSpot Answer to a Model.
    
    Args:
        answer_name: Name of the saved Answer containing formulas
        formula_names: Comma-separated formula names to promote (or 'all')
        model_name: Target Model name (auto-detected if empty)
    """
    ts = ThoughtSpotClient()
    result = ts.promote_answer_formulas(answer_name, formula_names.split(","), model_name or None)
    return result
```

---

## ThoughtSpotClient Method

```python
def promote_answer_formulas(
    self,
    answer_name: str,
    formula_names: List[str],
    target_model: Optional[str] = None
) -> str:
    """Full pipeline: find answer, export, promote formulas to model."""
    # 1. Find answer
    answers = self._post("/api/rest/2.0/metadata/search",
        {"metadata": [{"type": "ANSWER", "name_pattern": answer_name}]})
    if not answers:
        return f"Answer '{answer_name}' not found"
    answer_guid = answers[0]["metadata_id"]

    # 2. Export answer TML
    export = self._post("/api/rest/2.0/metadata/tml/export",
        {"metadata": [{"identifier": answer_guid}], "export_fqn": True})
    import yaml, re
    edoc = re.sub(r'[^\x09\x0A\x0D\x20-\x7E\x85\xA0-\uFFFD]', '', export[0]["edoc"])
    tml = yaml.safe_load(edoc)
    answer = tml["answer"]
    formulas = answer.get("formulas", [])

    # 3. Select formulas
    if formula_names == ["all"]:
        selected = formulas
    else:
        selected = [f for f in formulas if f["name"] in formula_names]
    if not selected:
        return "No matching formulas found"

    # 4. Find target model
    model_guid = answer["tables"][0]["fqn"] if not target_model else None
    if target_model:
        models = self._post("/api/rest/2.0/metadata/search",
            {"metadata": [{"type": "LOGICAL_TABLE", "name_pattern": target_model}]})
        model_guid = models[0]["metadata_id"]

    # 5. Export model TML
    model_export = self._post("/api/rest/2.0/metadata/tml/export",
        {"metadata": [{"identifier": model_guid}], "export_fqn": True, "export_associated": True})
    
    for item in model_export:
        edoc = re.sub(r'[^\x09\x0A\x0D\x20-\x7E\x85\xA0-\uFFFD]', '', item["edoc"])
        parsed = yaml.safe_load(edoc)
        if "model" in parsed:
            model_tml = parsed
            break

    # 6. Build and import
    import copy
    updated = copy.deepcopy(model_tml)
    m = updated["model"]
    
    for f in selected:
        fid = f"formula_{f['name']}"
        m.setdefault("formulas", []).append({"id": fid, "name": f["name"], "expr": f["expr"]})
        col_type = "MEASURE" if re.search(r'\b(sum|count|average|min|max)\s*\(', f["expr"], re.I) else "ATTRIBUTE"
        props = {"column_type": col_type}
        if col_type == "MEASURE":
            props["aggregation"] = "SUM"
        m.setdefault("columns", []).append({"name": f["name"], "formula_id": fid, "properties": props})

    updated_yaml = yaml.dump(updated, allow_unicode=True, default_flow_style=False)
    if not updated_yaml.startswith("guid:"):
        updated_yaml = f"guid: {model_tml.get('guid', model_guid)}\n" + updated_yaml

    result = self._post("/api/rest/2.0/metadata/tml/import",
        {"metadata_tmls": [updated_yaml], "import_policy": "ALL_OR_NONE", "create_new": False})
    
    status = result[0]["response"]["status"]["status_code"]
    if status == "OK":
        return f"Promoted {len(selected)} formula(s) to model"
    else:
        return f"Failed: {result[0]['response']['status'].get('error_message', 'Unknown')}"
```

---

## Differences from Claude Code Version

| Claude Code | Databricks |
|---|---|
| `ts tml export {guid} --fqn --parse` | `POST /metadata/tml/export` + `yaml.safe_load()` |
| `ts metadata search --type ANSWER` | `POST /metadata/search` with `type: ANSWER` |
| `ts tml import --no-create-new` | `POST /metadata/tml/import` with `create_new: False` |
| Profile in `~/.claude/thoughtspot-profiles.json` | `TSConfig` + `dbutils.secrets` |
| macOS Keychain | Databricks Secrets scope |
| Interactive slash-command UX | Programmatic method + LangChain tool |
| PyYAML via pip | Pre-installed on Databricks |

---

## Error Handling

| Error | Cause | Fix |
|---|---|---|
| 401 Unauthorized | Token expired | Refresh via `/api/rest/2.0/auth/token/full` |
| No formulas in Answer TML | Answer uses only Model columns | Search a different Answer |
| 403 on import | Lacks MODIFY access | Contact Model owner |
| `FORMULA is not a valid aggregation type` | `aggregation:` in `formulas[]` entry | Move to `columns[]` |
| `duplicate column name` | Formula name conflicts with existing | Rename or skip |
| `column_id not found` | Unresolved column ref | Re-map in Step 9 |
| Import creates duplicate | Missing `guid:` or `create_new: True` | Ensure `guid:` at root + `create_new: False` |

---

## Changelog

| Version | Date | Summary |
|---|---|---|
| 1.0.0 | 2026-05-18 | Initial Databricks port from Claude Code v1.2.0 |
