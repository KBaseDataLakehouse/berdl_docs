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
2. Creates your personal Iceberg catalog (`{username}`) backed by Polaris
3. Passes your KBase token to Trino so tenant membership can be resolved for access control

No manual credential handling is needed.

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

## Example Queries

### Discovery

```python
cur.execute("SHOW CATALOGS")                      # catalogs you can access
cur.execute("SHOW SCHEMAS FROM alice")            # namespaces in your personal catalog
cur.execute("SHOW TABLES FROM alice.analysis")    # tables in a namespace
cur.execute("SHOW COLUMNS FROM alice.analysis.my_table")
```

The `information_schema` of each catalog is also available:

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

**Queries fail with a credential or authorization error** (e.g., `Access Denied`, S3 403, `unauthorized_client`):

```python
refresh_spark_environment()          # rotates credentials, refreshes Spark + Trino catalogs
conn = get_trino_connection()        # recreate the connection
```

Recreating the connection with `get_trino_connection()` also refreshes your personal Iceberg catalog, which resolves most stale-credential issues on its own.

**Your KBase login expired and you logged in again:** recreate your connection — once — and you are fully back:

```python
conn = get_trino_connection()        # picks up your fresh token and credentials automatically
```

A connection object created **before** the expiry does not heal itself: it keeps sending your old token with every query. Within a few minutes it silently loses access to **tenant** catalogs (queries start failing with access errors) while queries against your **personal** catalog may still work — which makes a stale connection easy to mistake for a permissions problem. There is no way to refresh an existing connection; discard it and call `get_trino_connection()` again. The same rule applies after any credential refresh or rotation.

**A tenant catalog is missing from `SHOW CATALOGS`:** tenant catalogs are provisioned by platform automation, not by your notebook session. Confirm you are a member of the tenant; if you are and the catalog still does not appear, contact an administrator.

**`Catalog 'my' not found`:** the `my` alias only exists in Spark. In Trino, use your username as the catalog name.

**A table created in Spark does not appear:** make sure you query the right catalog — tables written to `my.<namespace>` in Spark appear under `{username}.<namespace>` in Trino.

## Tips

- **Read with Trino, write with Spark**: Trino is ideal for interactive reads; use your Spark session for creating tables and heavy ETL.
- **Reuse the connection**: create one connection per notebook session and open cursors from it as needed — but recreate it after a KBase re-login or credential refresh (see Troubleshooting).
- **Standard SQL**: Trino uses ANSI SQL — some functions differ from Spark SQL (see the [Trino functions reference](https://trino.io/docs/current/functions.html)).
- **Iceberg everywhere**: the same Iceberg table names (aside from `my`) work in both engines, so SQL can be moved between Spark and Trino with minimal changes.
