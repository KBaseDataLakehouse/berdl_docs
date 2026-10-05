# Trino User's Guide

This guide explains how to query your data lake tables with Trino, the interactive SQL engine available in every BERDL notebook.

## Overview

Trino provides fast, interactive, **read-only** SQL over the same tables you create with Spark — no Spark cluster or session required. It is a good fit for:

- Quick exploratory queries and aggregations
- Joining tables across catalogs (e.g., your personal tables with tenant tables)
- Lightweight dashboards or scripts that only need to read data

Writes always go through Spark. Trino enforces read-only access: `INSERT`, `CREATE`, `DROP`, and other write statements are rejected. To create or modify tables, see the [Tenant SQL Warehouse Guide](tenant_sql_warehouse_guide.md).

## Getting Connected

The `get_trino_connection()` helper is pre-imported in every BERDL notebook. It returns a standard [Python DB-API](https://github.com/trinodb/trino-python-client) connection configured with your credentials:

```python
conn = get_trino_connection()
cur = conn.cursor()

cur.execute("SHOW CATALOGS")
print(cur.fetchall())
```

Behind the scenes, the helper:

1. Fetches your MinIO (S3) credentials from the governance API
2. Creates your personal Iceberg catalog (`{username}`) backed by Polaris and makes it the connection's default catalog
3. Passes your KBase token to Trino so tenant membership can be resolved for access control

No manual credential handling is needed.

If you open a connection yourself with `trino.dbapi.connect(...)` instead, pass your KBase token the same way, or every tenant catalog fails with `Access Denied: Cannot access catalog`:

```python
import os, trino
conn = trino.dbapi.connect(
    host=os.environ["TRINO_HOST"], port=int(os.environ["TRINO_PORT"]), user=os.environ["USER"],
    extra_credential=[("kbase_auth_token", os.environ["KBASE_AUTH_TOKEN"])],
)
```

### Results as a pandas DataFrame

```python
import pandas as pd

cur = conn.cursor()
cur.execute("SELECT * FROM alice.analysis.my_table LIMIT 100")
df = pd.DataFrame(cur.fetchall(), columns=[d[0] for d in cur.description])
```

## Catalog Layout

Trino exposes your tables through two kinds of catalogs:

| Catalog | Contents | Example table path |
|---------|----------|--------------------|
| `{username}` | Your personal Iceberg (Polaris) warehouse | `alice.analysis.my_table` |
| `{tenant}` | Shared tenant Iceberg warehouses (members only) | `kbase.research.shared_dataset` |

Always use the fully qualified `catalog.namespace.table` form (`alice.analysis.my_table`, `kbase.research.shared_dataset`).

### Spark vs. Trino Naming

Iceberg table names are portable between Spark and Trino, with one exception: the `my` convenience alias is **Spark-only**. In Trino, use your username instead.

| Table | Spark | Trino |
|-------|-------|-------|
| Personal Iceberg | `my.analysis.t` or `alice.analysis.t` | `alice.analysis.t` |
| Tenant Iceberg | `kbase.research.t` | `kbase.research.t` |

### Names and Case

Trino treats identifiers as case-insensitive and shows them in lower case. A table created in Spark as `Genome_ANI` is listed by Trino as `genome_ani`, and `kbase.research.genome_ani`, `kbase.research.Genome_ANI` and `kbase.research."Genome_ANI"` all read it. Result column names come back in lower case too (`BiGG_Reaction` becomes `bigg_reaction`), so build DataFrames from `cursor.description` rather than from the names you expect.

## Example Queries

### Discovery

```python
cur.execute("SHOW CATALOGS")                      # catalogs you can access
cur.execute("SHOW SCHEMAS FROM alice")            # namespaces in your personal catalog
cur.execute("SHOW TABLES FROM alice.analysis")    # tables in a namespace
```

**Column names and types:** `SHOW COLUMNS`, `DESCRIBE` and `information_schema.columns` currently return nothing for Iceberg tables on BERDL's Trino (a known gap between Trino's Iceberg REST connector and Polaris). Read the columns from an empty result instead:

```python
cur.execute("SELECT * FROM alice.analysis.my_table WHERE 1 = 0")
cur.fetchall()
columns = [(d[0], d[1]) for d in cur.description]   # [('id', 'bigint'), ('name', 'varchar'), ...]
```

The `information_schema` of each catalog is available for schemas and tables (only its `columns` view is empty):

```python
cur.execute("""
    SELECT table_schema, table_name
    FROM alice.information_schema.tables
    WHERE table_schema != 'information_schema'
""")
```

### Aggregation

```python
cur.execute("""
    SELECT dept, count(*) AS employees, sum(salary) AS total_salary
    FROM alice.analysis.employees
    GROUP BY dept
    ORDER BY dept
""")
```

### Cross-Catalog Joins

A single Trino query can join tables across catalogs — for example, your personal tables with tenant tables:

```python
cur.execute("""
    SELECT e.name, e.dept, d.location
    FROM alice.analysis.employees e
    LEFT JOIN kbase.research.departments d
      ON e.dept = d.dept_name
""")
```

## Access Control

- **Personal catalog**: only you can access `{username}`. Other users' catalogs are blocked.
- **Tenant catalogs**: visible only to tenant members. Membership is resolved from the governance API using your KBase token. To join a tenant, see [Requesting Tenant Access](requesting-tenant-access.md).
- **Read-only**: `INSERT`, `CREATE SCHEMA`, `DROP TABLE`, and all other write operations are denied on every catalog.

## Troubleshooting

**Queries fail with a credential or authorization error** (e.g., `Access Denied`, S3 403, `unauthorized_client`): recreate the connection first. It refreshes your personal Iceberg catalog with your current credentials, which resolves most stale-credential issues on its own:

```python
conn = get_trino_connection()
```

Only if that does not help, rotate your credentials and connect again:

```python
refresh_spark_environment()          # rotates credentials, restarts Spark, refreshes the Trino catalog
conn = get_trino_connection()
```

`refresh_spark_environment()` issues **new** S3 and Polaris secrets and revokes the old ones immediately, so every other kernel, script or app still holding the old ones (Spark sessions, boto3/fsspec clients, Trino connections) starts failing until it reconnects. Run it by hand when something is actually broken, never in a loop, a retry handler or several processes at once. To recover a dead Spark session without rotating, restart Spark Connect with your current credentials instead:

```python
from berdl_notebook_utils.spark.connect_server import start_spark_connect_server
start_spark_connect_server(force_restart=True)
spark = get_spark_session()
```

**Your KBase login expired and you logged in again:** recreate your connection — once — and you are fully back:

```python
conn = get_trino_connection()        # picks up your fresh token and credentials automatically
```

A connection object created **before** the expiry does not heal itself: it keeps sending your old token with every query. Within a few minutes it silently loses access to **tenant** catalogs (queries start failing with access errors) while queries against your **personal** catalog may still work — which makes a stale connection easy to mistake for a permissions problem. There is no way to refresh an existing connection; discard it and call `get_trino_connection()` again. The same rule applies after any credential refresh or rotation.

**`ICEBERG_CATALOG_ERROR: Cannot obtain metadata` on your personal catalog** (from `get_trino_connection()` or from queries on `{username}`): Trino could not load your personal catalog because Polaris rejected the credential it was created with, usually because your credentials were rotated in another process at the same moment. Call `get_trino_connection()` again, which re-creates the catalog. If the error keeps coming back, contact an administrator: on older Trino versions such a catalog can only be cleared by a Trino restart. Tenant catalogs are not affected.

**A tenant catalog is missing from `SHOW CATALOGS`:** tenant catalogs are provisioned by platform automation, not by your notebook session. Confirm you are a member of the tenant; if you are and the catalog still does not appear, contact an administrator.

**`Catalog 'my' not found`:** the `my` alias only exists in Spark. In Trino, use your username as the catalog name.

**`SHOW COLUMNS` or `DESCRIBE` returns no rows:** expected for now; use the empty `SELECT ... WHERE 1 = 0` pattern under [Discovery](#discovery) to read column names and types.

**A table created in Spark does not appear:** make sure you query the right catalog — tables written to `my.<namespace>` in Spark appear under `{username}.<namespace>` in Trino.

**`GENERIC_INTERNAL_ERROR` with `Namespace does not exist: <name>`:** the schema in your query does not exist in that catalog. Trino reports a missing Iceberg namespace as an internal error instead of "schema not found", but nothing is broken. Check the spelling with `SHOW SCHEMAS FROM <catalog>`, and remember that a two-part name (`schema.table`) is looked up in your personal catalog. Old Delta-style names that join the tenant and namespace with `_` (for example `kbase_research.genome`) no longer exist; use the three-part name (`kbase.research.genome`).

**A view fails with `Cannot read unsupported dialect 'spark' for view ...`** (`ICEBERG_UNSUPPORTED_VIEW_DIALECT`): a view created with `CREATE VIEW` in Spark stores its definition in Spark SQL only, and Trino cannot read it, although it still appears in `SHOW TABLES`. Read the view from your Spark session, or query its underlying tables in Trino (`SHOW CREATE TABLE <view>` in Spark prints its SQL). If you need the result in Trino, save it as a table from Spark (`CREATE TABLE ... AS SELECT ...`).

## Tips

- **Read with Trino, write with Spark**: Trino is ideal for interactive reads; use your Spark session for creating tables and heavy ETL.
- **Reuse the connection**: create one connection per notebook session and open cursors from it as needed — but recreate it after a KBase re-login or credential refresh (see Troubleshooting). Every `get_trino_connection()` call checks, and may re-create, your personal catalog on the Trino coordinator (several statements before your first query), so do not call it per query or inside a polling loop.
- **Standard SQL**: Trino uses ANSI SQL — some functions differ from Spark SQL (see the [Trino functions reference](https://trino.io/docs/current/functions.html)).
- **Iceberg everywhere**: the same Iceberg table names (aside from `my`) work in both engines, so SQL can be moved between Spark and Trino with minimal changes. Views are the exception: a view created in Spark can only be read from Spark (see Troubleshooting).
