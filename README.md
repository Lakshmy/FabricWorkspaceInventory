# Fabric Workspace Inventory

Extracts a comprehensive object-level inventory for all Microsoft Fabric workspaces under one or more target capacities using **Power BI Admin REST APIs**. The notebook produces three outputs:

- **Inventory DataFrame** — every semantic model, dataflow, and report across the scanned workspaces, including datasource type, connection details, refresh schedules, refresh history, storage mode, and workspace-level user permissions.
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

**Additional endpoints used by `fabric_inventory_scan.ipynb`:**

| Endpoint | Purpose |
|----------|---------|
| `POST /v1.0/myorg/admin/workspaces/getInfo?datasourceDetails=True&datasetSchema=True&datasetExpressions=True&getArtifactUsers=True` | Initiate an async workspace scan |
| `GET /v1.0/myorg/admin/workspaces/scanStatus/{scanId}` | Poll scan completion status |
| `GET /v1.0/myorg/admin/workspaces/scanResult/{scanId}` | Retrieve scan results with `datasourceInstances` array |

---

## Choosing a notebook

This repository contains two notebooks with different approaches:

| Notebook | Approach | Best for |
|----------|----------|----------|
| `fabric_inventory.ipynb` | Per-item Admin API calls (`/admin/datasets/{id}/datasources`) | Quick runs, simpler setup, fewer API permissions |
| `fabric_inventory_scan.ipynb` | Batch Workspace Scan API (`getInfo` → `scanResult`) | Complete datasource coverage (ODBC, JDBC, gateway-bound), fewer total API calls at scale |

The **scan-based notebook** uses the Admin `getInfo` / `scanResult` flow which returns a global `datasourceInstances` array. This captures connection details for **all** datasource types — including ODBC, JDBC, OData, and gateway-bound sources — that the per-item `/datasources` endpoint may miss.

### Detailed comparison

#### Architecture & API approach

| Aspect | `fabric_inventory.ipynb` | `fabric_inventory_scan.ipynb` |
|--------|--------------------------|-------------------------------|
| **API strategy** | **Per-item REST calls** — fetches workspaces with `$expand=users,datasets,dataflows,reports`, then calls individual endpoints per dataset/dataflow for datasources, refresh schedule, and history | **Batch Scan API** — uses the async `getInfo` → `scanStatus` → `scanResult` flow to retrieve all metadata in bulk |
| **Datasource resolution** | Calls `/admin/datasets/{id}/datasources` and `/admin/dataflows/{id}/datasources` **per item**, plus `Default.GetBoundGatewayDatasources` for gateway-bound ODBC/JDBC sources | Relies on the **global `datasourceInstances` array** returned by the scan result, linked to items via `datasourceUsages` (requires `lineage=True`) |
| **API call volume** | **High** — O(n) calls where n = total datasets + dataflows (one datasource call per item) | **Low** — a few batch calls regardless of item count (batches of 100 workspaces) |
| **Tenant setting** | Standard Admin API permissions + *Enhance admin APIs responses with detailed metadata* | Same, plus the scan API also requires *Enhance admin APIs responses with detailed metadata* for `datasourceInstances` to be populated |

#### Key code differences

| Feature | `fabric_inventory.ipynb` | `fabric_inventory_scan.ipynb` |
|---------|--------------------------|-------------------------------|
| **Workspace fetch** | `$expand=users,datasets,dataflows,reports` — returns full item metadata inline | Fetches only workspace IDs, then submits them to the scan API |
| **POST helper** | Not needed | Has `admin_api_post()` for the `getInfo` call |
| **Scan polling** | N/A | Polls `scanStatus` with configurable interval/timeout, processes in batches of 100 |
| **Datasource logic** | `get_datasources()` + `get_gateway_datasources()` — direct API calls per item | `build_datasource_lookup()` + `resolve_datasources()` — dictionary lookup from the scan's global `datasourceInstances` array |
| **Gateway sources** | Explicitly fetches gateway-bound datasources via `GetBoundGatewayDatasources` | Gateway sources are included automatically in `datasourceInstances` when `datasourceDetails=True` |
| **Item processing** | Separate functions: `process_semantic_models()`, `process_dataflows()`, `process_reports()` | Single function: `process_scan_workspaces()` handles all item types in one loop |
| **Orchestrator steps** | 4 steps (capacities → workspaces → inventory → activity) | 5 steps (capacities → workspace IDs → scan → inventory → activity) |
| **Config constants** | None beyond capacity names | `SCAN_POLL_INTERVAL_SECONDS`, `SCAN_MAX_WAIT_SECONDS`, `SCAN_BATCH_SIZE` |

#### When to use which

- **`fabric_inventory.ipynb`** — simpler setup, works without async scan complexity, good for **smaller environments**. More API calls but no polling logic.
- **`fabric_inventory_scan.ipynb`** — better for **large tenants** (fewer API calls), captures **all datasource types** (ODBC, JDBC, OData, gateway-bound) in one shot via the global `datasourceInstances` array, but requires the additional `lineage=True` parameter and scan polling logic.

---

## How to use

### 1. Import a notebook

Upload `fabric_inventory.ipynb` or `fabric_inventory_scan.ipynb` (or both) to a Fabric workspace attached to a capacity where you have notebook-run permissions.

### 2. Configure target capacities

In **Cell 3**, edit the `target_capacity_names` list to include the display names of the Fabric capacities you want to scan:

```python
target_capacity_names = [
    "lakfabuswest2"
]
```

### 3. Optionally filter to specific workspaces

In the execution cell (**Cell 14**), you can limit the scan to specific workspaces by name, or set `workspace_names=None` to scan all workspaces under the target capacities:

```python
inventory_df, activity_df, errors_df = get_fabric_inventory(
    capacity_names=target_capacity_names,
    workspace_names=["MyTestWorkspace"],  # set to None for all workspaces
    activity_lookback_days=1,             # number of days of activity to fetch (max 30)
)
```

### 5. Lakehouse attachment

The notebook must be **attached to a lakehouse** for the CSV export to work. The `export_to_onelake()` function writes to `/lakehouse/default/Files/csv_exports`, which is the mounted Files folder of the attached lakehouse.

### 6. Run all cells

Execute the notebook top-to-bottom. Progress is printed at each step. The final cell displays:

- A summary of items by type and capacity
- The full inventory table
- Recent user activity events
- Any errors encountered
- A CSV export of the inventory to OneLake (`/lakehouse/default/Files/csv_exports/fabric_items_<timestamp>.csv`)

---

## Notebook structure — `fabric_inventory.ipynb`

| Cell | Purpose |
|------|---------|
| 1 | Overview (markdown) |
| 2 | Imports (`sempy.fabric`, `pandas`, etc.) |
| 3 | Configuration — target capacity display names |
| 4 | `admin_api_get()` — generic Admin API caller with pagination and rate-limit retry |
| 5 | `resolve_capacities()` — map capacity display names to IDs |
| 6 | `fetch_workspaces()` — retrieve workspaces via Admin API with `$expand` |
| 7 | `format_permissions()` — format workspace user permission strings |
| 8 | `get_datasources()` / `format_connection_string()` / `get_connection_info()` — extract datasource type and connection details (only for semantic models, dataflows, and reports) |
| 9 | `get_refresh_schedule()` / `get_refresh_history()` — refresh metadata for semantic models |
| 10 | `process_semantic_models()` / `process_dataflows()` / `process_reports()` — per-item-type processors |
| 11 | `fetch_user_activity()` — Admin activity events with multi-day support |
| 12 | `get_fabric_inventory()` — orchestrator that ties everything together |
| 13 | `export_to_onelake()` — export inventory DataFrame to a timestamped CSV in OneLake |
| 14 | Execute, display results, and export to OneLake |

---

## Notebook structure — `fabric_inventory_scan.ipynb`

| Cell | Purpose |
|------|---------|
| 1 | Overview (markdown) |
| 2 | Imports |
| 3 | Configuration — target capacity names and scan settings |
| 4 | `admin_api_get()` / `admin_api_post()` — Admin API helpers with retry |
| 5 | `resolve_capacities()` — map capacity display names to IDs |
| 6 | `fetch_workspace_ids()` — retrieve workspace IDs via Admin API |
| 7 | `scan_workspaces()` — run the async workspace scan (`getInfo` → `scanStatus` → `scanResult`) |
| 8 | `build_datasource_lookup()` / `resolve_datasources()` — map `datasourceUsages` to connection details |
| 9 | `format_permissions()` — format workspace user permission strings |
| 10 | `get_refresh_schedule()` / `get_refresh_history()` — refresh metadata for semantic models |
| 11 | `process_scan_workspaces()` — build inventory rows from scan results |
| 12 | `fetch_user_activity()` — Admin activity events with multi-day support |
| 13 | `get_fabric_inventory_scan()` — orchestrator |
| 14 | `export_to_onelake()` — export DataFrames to timestamped CSVs in OneLake |
| 15 | Execute, display results, and export to OneLake |

---

## Rate limiting

The Admin APIs enforce rate limits. The notebook automatically retries on HTTP 429 responses (up to 3 attempts) using the `Retry-After` header. If you are scanning a large tenant, expect the extraction to take several minutes.