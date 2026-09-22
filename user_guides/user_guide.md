# User Guide: Accessing BERDL JupyterHub

## Prerequisites

Before you begin, ensure you have:
* **KBase Account** - Register at https://narrative.kbase.us/ if you don't have one
* **ORCID Account linked with MFA enabled** - Required for authentication
* **BERDL_USER group membership** - Contact KBase System Admin team for access

## Step-by-Step Guide

### 1. Access BERDL JupyterHub

Navigate to BERDL JupyterHub:

```
https://hub.berdl.kbase.us/
```

> **💡 Accessing Other Environments:**
>
> Development and staging environments require SSH tunnel setup:
> - Development: `https://hub.dev.berdl.kbase.us/`
> - Staging: `https://hub.stage.berdl.kbase.us/`
>
> For SSH tunnel setup instructions, see the [Object Storage (S3) Guide - Accessing storage from your laptop](s3_guide.md#accessing-storage-from-your-laptop) section.

This will open the JupyterHub login interface.

<img src="screen_shots/login.png" alt="login" style="max-width: 800px;">

### 2. Log In with Your KBase Credentials

Log in using your KBase credentials (the same 3rd party identity provider, username and password you use for KBase services).

> **⚠️ Important Authentication Requirements:**
> - If you don't have a KBase account yet, you'll need to register at https://narrative.kbase.us/ before proceeding.
> - **Your KBase account must be linked to ORCID**
>   - To link your ORCID account: At https://narrative.kbase.us/ → Click on your username (top right) → My Account → Linked Providers → Link with ORCID
>
>      <img src="screen_shots/link_ORCID.png" alt="link_ORCID" style="max-width: 600px;">
>
> - **Multi-Factor Authentication (MFA) must be enabled on your ORCID account for security**
>   - To enable MFA: Go to https://orcid.org/account → Click on your username → Account settings → Two-factor authentication
>
>      <img src="screen_shots/ORCID_MFA.png" alt="ORCID_MFA" style="max-width: 600px;">
>
> - **Your account must be added to the BERDL_USER group**
>   - If you see an error message like the one below, please [contact the KBase Platform team](https://docs.kbase.us/troubleshooting/support) to have your KBase account added to the appropriate user group.
>
>      <img src="screen_shots/require_role.png" alt="require_role" style="max-width: 400px;">

### 3. Data Access
By default, you have **read/write access** to:
- Your personal Iceberg catalog (`my`) — create namespaces and tables here
- Any tenant Iceberg catalogs you belong to (e.g., `kbase`) — shared team data
- Your personal file storage (`s3a://cdm-lake/users-general-warehouse/{username}/`) — free-form files such as CSVs, TSVs, raw exports, and staged inputs

Your tables live under `s3a://cdm-lake/users-sql-warehouse/{username}/`. That prefix is managed by the catalog: you can list it, but you cannot write files into it directly. Write tables through Spark (see the [Tenant SQL Warehouse Guide](tenant_sql_warehouse_guide.md)) and put everything else in your file storage. The [Object Storage (S3) Guide](s3_guide.md#what-you-can-access-and-why-you-get-accessdenied) explains which prefixes you can write to.

For questions about data access or permissions, please reach out to the BERDL Platform team.

### 4. Access the Workspace

#### 4.1 Home Directory
After logging in, click on `username` (your KBase username) under the `FAVORITES` section to access your personal home directory.
This directory is exclusive to your account and is where you can store your notebooks and files.

#### 4.2 Shared Directory
To access shared resources and example notebooks, click on the `Global Share` folder icon. This directory contains shared content available to all users.

### 5. Using Pre-loaded Functions

BERDL JupyterHub automatically loads helper functions and utilities when you start a notebook. These are imported from [berdl_notebook_utils](../notebook_utils/berdl_notebook_utils) and are immediately available without any imports needed!

#### 5.1 Creating a Spark Session

Use the `get_spark_session()` function to create or get a Spark session with all necessary configurations:

```python
spark = get_spark_session()
```

This automatically configures:
- Spark Connect server connection
- Apache Iceberg catalogs (personal `my` + tenant catalogs via Polaris) — `spark.sql("SHOW CATALOGS")` or `list_catalogs()` lists them
- S3 object storage access
- Delta Lake support (legacy, for backward compatibility)

> **Note:** BERDL has migrated from Delta Lake to Apache Iceberg. See the [Iceberg Migration Guide](iceberg_migration_guide.md) for details on the new catalog structure and Iceberg-specific features like time travel and schema evolution.

#### 5.2 Refreshing Spark Credentials and Catalog Access

Use `refresh_spark_environment()` when your BERDL credentials or catalog access changes while a notebook is already running. This is commonly needed after access is granted or revoked for an Iceberg namespace, or when you are instructed to refresh Spark before using a newly available catalog or namespace.

```python
# Refresh credentials, catalog metadata, Spark Connect, and Trino catalogs
refresh_result = refresh_spark_environment()
refresh_result

# Re-create the Spark session after the refresh stops any active session
spark = get_spark_session()
```

The function returns a status dictionary showing which refresh steps succeeded, were skipped, or failed. It stops the active Spark session and restarts Spark Connect, so run it between operations and reassign `spark` before continuing with Spark queries.

#### 5.3 Displaying DataFrames

Use the standard `show()` method to display Spark DataFrames:

```python
# Display DataFrame using show()
df = spark.sql("SELECT * FROM my_namespace.my_table")
df.show()
```

**For formatted, interactive display with CSV-like formatting:**

If you would like to display your DataFrame in a formatted table with sorting, filtering, and pagination capabilities, use `display_df()`:

```python
# Display DataFrame in interactive table format
display_df(df)
```

<img src="screen_shots/display_func.png" alt="display_df" style="max-width: 800px;">

> **⚠️ Warning:** `display_df()` converts Spark DataFrames to pandas DataFrames and loads the entire DataFrame into memory. Only use this for reasonably-sized datasets that can fit in memory. For large datasets, use `show()` or `limit()` the results first:
> ```python
> # Safe for large datasets - limit rows first
> display_df(df.limit(100))
> ```

### 6. Accessing Data

BERDL provides prebuilt functions to explore and query your data. All functions are automatically loaded - no imports needed!

#### 6.1 Listing Databases (Namespaces)

Use `get_databases()` to list all namespaces you have access to:

```python
# Get list of all databases/namespaces
databases = get_databases()
print(f"Available databases: {databases}")
```

#### 6.2 Listing Tables in a Database

Use `get_tables()` to list all tables in a specific namespace:

```python
# List all tables in a namespace
tables = get_tables("my_namespace")
print(f"Tables in my_namespace: {tables}")
```

#### 6.3 Getting Table Schema

Use `get_table_schema()` to view the structure of a table:

```python
# Get schema information for a table
schema = get_table_schema("my_namespace", "my_table")
display_df(schema)
```

#### 6.4 Getting Complete Database Structure

Use `get_db_structure()` to see all tables and their schemas in a namespace:

```python
# Get complete structure of a database
db_structure = get_db_structure("my_namespace")
for table_info in db_structure:
    print(f"Table: {table_info['table']}")
    print(f"Columns: {table_info['columns']}")
```

## Installing your own Python packages

Install extra packages into a **virtual environment or conda environment**, not with
`pip install --user`. Packages under `~/.local` are invisible to the notebook server,
terminals and KIND on purpose (`PYTHONNOUSERSITE=1`): a `--user` package shadows the
image's copy, survives every image upgrade, and can stop your server from starting.
Notebook kernels still see `~/.local` for compatibility, but new `--user` installs from a
terminal are refused.

```bash
python -m venv ~/envs/myproj && source ~/envs/myproj/bin/activate
pip install <package>
python -m ipykernel install --user --name myproj --display-name "Python (myproj)"   # optional: use it as a kernel
```

## Troubleshooting

### Restarting Your Server

If you encounter issues with your notebook environment (such as kernel errors, connection problems, or need to apply environment updates), you can restart your JupyterHub server:

**Step 1:** In JupyterLab, click **File** → **Hub Control Panel**

**Step 2:** Click the **Stop My Server** button

**Step 3:** Wait for the "Stop My Server" button to disappear (this may take 10-30 seconds)

**Step 4:** Once the button disappears, click **Start My Server** to start a fresh server instance

> **⚠️ Note:** Stopping your server will close all running notebooks and kernels. Make sure to save your work before restarting.

> **💡 Tip:** Your files in your home directory are persistent and will not be deleted when you restart your server.

### Restarting Your Spark Connect Server (Preferred First Step)

**Preferred fix for most Spark issues — the non-rotating restart.** This re-fetches your current credentials **without rotating them**, so it never invalidates credentials in use elsewhere (other kernels, running scripts, remote connections). Try it before `refresh_spark_environment()`.

Common symptoms it fixes:

- `spark.sql("SHOW CATALOGS")` returns only `spark_catalog` — your Iceberg catalogs are missing
- Queries fail with `SCHEMA_NOT_FOUND` for schemas you know exist
- `get_spark_session()` fails with `RETRIES_EXCEEDED` or hangs
- Your Spark session is unresponsive

**The fix:** in any notebook cell, run

```python
get_credentials()
start_spark_connect_server(force_restart=True)
```

then get a fresh session:

```python
spark = get_spark_session()
```

> **💡 Tip:** If your Iceberg catalogs are still missing afterwards, restart your kernel (**Kernel → Restart Kernel**) and run the same commands again — a kernel restart re-runs the catalog discovery your Spark configuration depends on.

If the problem persists (especially `403 AccessDenied` errors), use `refresh_spark_environment()` below.

### Refreshing Your Spark Environment

Use this when your **storage credentials have drifted out of sync** with the platform — even though you're logged in, every data-touching operation silently fails. Common symptoms:

- The **Tenant Browser** spins forever and never renders any folders
- **Folder favorites** under `FAVORITES` show up empty
- The notebook UI feels unusually slow (typing in a cell lags, cells take a long time to start)
- `get_spark_session()` hangs or fails to start the Spark Connect server
- Spark Connect server logs (in `~/.spark/connect-server-logs/`) show:
  ```
  java.nio.file.AccessDeniedException: s3a://cdm-spark-job-logs/spark-job-logs/<username>:
    getFileStatus ... Status Code: 403
  ```
- Reads or writes to your personal/tenant warehouses return `403 AccessDenied` even though you have permission

**The fix:** in any notebook cell, run

```python
refresh_spark_environment()
```

This single call:
1. Rotates your S3 credentials with the platform
2. Rotates your Polaris (Iceberg) credentials
3. Re-fetches your Polaris catalog list
4. Stops the existing Spark session
5. Restarts the Spark Connect server with the new credentials
6. Refreshes your Trino dynamic catalogs

You should see output similar to:

```python
{
  'credentials':      {'status': 'ok', 'username': '<your-username>'},
  'polaris_catalog':  {'status': 'ok', 'personal_catalog': 'user_<your-username>', 'tenant_catalogs': [...]},
  'spark_connect':    {'pid': '...', ...},
  'trino_catalogs':   {'status': 'ok'},
}
```

**Step 1:** Run `refresh_spark_environment()` in any cell. Wait for it to complete (~15-30 seconds).

**Step 2:** Refresh your browser tab. The Tenant Browser and favorites should now render normally.

**Step 3:** (Only if step 2 doesn't resolve it) Restart your server using the steps in [Restarting Your Server](#restarting-your-server) above.

> **⚠️ Note:** If you still see errors after `refresh_spark_environment()` plus a server restart plus a browser refresh, please reach out to the BERDL Platform team with a screenshot of the function's output and any visible error message — there are a handful of rarer failure modes that need platform-side intervention.
