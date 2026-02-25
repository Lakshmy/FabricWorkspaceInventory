# Fabric Workspace Inventory

Extracts a comprehensive object-level inventory for all Microsoft Fabric workspaces under one or more target capacities using **Power BI Admin REST APIs**. The notebook produces three outputs:

- **Inventory DataFrame** — every semantic model, dataflow, and report across the scanned workspaces, including datasource connection details, refresh schedules, refresh history, storage mode, and workspace-level user permissions.
- **Activity DataFrame** — recent user-activity events (views, edits, refreshes, etc.) filtered to the scanned workspaces.
- **Errors DataFrame** — any API errors encountered during extraction (e.g., inaccessible system datasets).

---

## Prerequisites

### 1. Runtime environment

This notebook uses the [Semantic Link](https://learn.microsoft.com/fabric/data-science/semantic-link-overview) library (`sempy.fabric`) and must be run inside a **Microsoft Fabric notebook**. Authentication is handled automatically through the signed-in user's identity.

### 2. Required roles

The signed-in user (or service principal) must hold **one** of the following Entra ID roles:

| Role | Scope |
|------|-------|
| **Fabric Administrator** | Microsoft 365 admin center → Fabric settings |
| **Power BI Service Administrator** | Equivalent to Fabric Administrator for Power BI workloads |
| **Global Administrator** | Grants all admin permissions (broader than needed) |

> These roles are required because the notebook calls **Admin API** endpoints (`/v1.0/myorg/admin/...`) that operate across the entire tenant.

### 3. Tenant settings

The following tenant settings must be enabled in the **Fabric Admin Portal** (Settings → Admin portal → Tenant settings):

| Tenant setting | Why it is needed |
|----------------|-----------------|
| **Enhance admin APIs responses with detailed metadata** | Required for the `$expand=users,datasets,dataflows,reports` parameter on the Admin Groups endpoint. Without this, expanded metadata is not returned. |
| **Enhance admin APIs responses with DAX and mashup expressions** |  enables richer dataset metadata if you extend the notebook later. |

If you are authenticating with a **service principal** instead of a user identity, also enable:

| Tenant setting | Why it is needed |
|----------------|-----------------|
| **Service principals can use Fabric APIs** | Allows the service principal to call Fabric/Power BI REST APIs. |
| **Allow service principals to use read-only admin APIs** | Grants the service principal access to the Admin API endpoints used by this notebook. |

### 4. API permissions summary

The notebook calls the following Admin API endpoints. No explicit Azure AD app registration scopes are required when running as a Fabric notebook (the user's identity is used directly).

| Endpoint | Purpose |
|----------|---------|
| `GET /v1.0/myorg/admin/groups?$expand=users,datasets,dataflows,reports` | List all workspaces with expanded item metadata and user permissions |
| `GET /v1.0/myorg/admin/datasets/{id}/datasources` | Retrieve datasource connection details for a semantic model |
| `GET /v1.0/myorg/admin/dataflows/{id}/datasources` | Retrieve datasource connection details for a dataflow |
| `GET /v1.0/myorg/admin/datasets/{id}/refreshSchedule` | Get the configured refresh schedule for a semantic model |
| `GET /v1.0/myorg/admin/datasets/{id}/refreshes?$top=N` | Get recent refresh history for a semantic model |
| `GET /v1.0/myorg/admin/activityevents?startDateTime=...&endDateTime=...` | Fetch tenant-wide user activity events for a given day |

---

## How to use

### 1. Import the notebook

Upload `fabric_inventory.ipynb` to a Fabric workspace attached to a capacity where you have notebook-run permissions.

### 2. Configure target capacities

In **Cell 3**, edit the `target_capacity_names` list to include the display names of the Fabric capacities you want to scan:

```python
target_capacity_names = [
    "lakfabuswest2"
]
```

### 3. Optionally filter to specific workspaces

In the final execution cell (**Cell 13**), you can limit the scan to specific workspaces by name, or set `workspace_names=None` to scan all workspaces under the target capacities:

```python
inventory_df, activity_df, errors_df = get_fabric_inventory(
    capacity_names=target_capacity_names,
    workspace_names=["MyTestWorkspace"],  # set to None for all workspaces
    activity_lookback_days=1,             # number of days of activity to fetch (max 30)
)
```

### 4. Run all cells

Execute the notebook top-to-bottom. Progress is printed at each step. The final cell displays:

- A summary of items by type and capacity
- The full inventory table
- Recent user activity events
- Any errors encountered

---

## Notebook structure

| Cell | Purpose |
|------|---------|
| 1 | Overview (markdown) |
| 2 | Imports (`sempy.fabric`, `pandas`, etc.) |
| 3 | Configuration — target capacity display names |
| 4 | `admin_api_get()` — generic Admin API caller with pagination and rate-limit retry |
| 5 | `resolve_capacities()` — map capacity display names to IDs |
| 6 | `fetch_workspaces()` — retrieve workspaces via Admin API with `$expand` |
| 7 | `format_permissions()` — format workspace user permission strings |
| 8 | `get_datasources()` / `format_connection_string()` — extract datasource connection details |
| 9 | `get_refresh_schedule()` / `get_refresh_history()` — refresh metadata for semantic models |
| 10 | `process_semantic_models()` / `process_dataflows()` / `process_reports()` — per-item-type processors |
| 11 | `fetch_user_activity()` — Admin activity events with multi-day support |
| 12 | `get_fabric_inventory()` — orchestrator that ties everything together |
| 13 | Execute and display results |

---

## Rate limiting

The Admin APIs enforce rate limits. The notebook automatically retries on HTTP 429 responses (up to 3 attempts) using the `Retry-After` header. If you are scanning a large tenant, expect the extraction to take several minutes.