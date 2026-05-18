# ThoughtSpot API Auth from Databricks

How to authenticate to the ThoughtSpot REST API from Databricks compute.

## Auth Flow

```
dbutils.secrets.get("thoughtspot", "api-token")
       │
       └─→ Bearer token ─→ Authorization header ─→ ThoughtSpot API
```

## Headers

```python
headers = {
    "Authorization": f"Bearer {token}",
    "Content-Type": "application/json",
    "Accept": "application/json"
}
```

## Token Refresh

Tokens expire after ~24h. Refresh via:

```python
import requests

resp = requests.post(
    f"{base_url}/api/rest/2.0/auth/token/full",
    json={
        "username": dbutils.secrets.get("thoughtspot", "username"),
        "password": dbutils.secrets.get("thoughtspot", "password"),
        "validity_time_in_sec": 86400
    }
)
new_token = resp.json()["token"]

# Update secret
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
w.secrets.put_secret(scope="thoughtspot", key="api-token", string_value=new_token)
```

## Automated Refresh

Schedule as a Databricks Job (every 12h):
```python
# scripts/token_refresh.py
import requests
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()
base_url = "https://your-cluster.thoughtspot.cloud"

resp = requests.post(f"{base_url}/api/rest/2.0/auth/token/full", json={
    "username": w.dbutils.secrets.get("thoughtspot", "username"),
    "password": w.dbutils.secrets.get("thoughtspot", "password"),
    "validity_time_in_sec": 86400
})
w.secrets.put_secret(scope="thoughtspot", key="api-token", string_value=resp.json()["token"])
print(f"Token refreshed: {resp.json()['token'][:20]}...")
```
