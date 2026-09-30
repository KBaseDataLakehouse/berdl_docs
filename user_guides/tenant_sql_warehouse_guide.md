# Tenant SQL Warehouse User's Guide

This guide explains how to configure your Spark session to write tables to either your personal SQL warehouse or a tenant's shared SQL warehouse.

## Overview

The BERDL JupyterHub environment supports two types of SQL warehouses:

1. **User SQL Warehouse**: Your personal workspace for tables (Iceberg catalog: `my`)
2. **Tenant SQL Warehouse**: Shared workspace for team/organization tables (Iceberg catalog per tenant, e.g., `kbase`)

Isolation is provided at the **catalog level** — no naming prefixes needed.

| Configuration | Catalog | Example Namespace |
|--------------|---------|-------------------|
| `create_namespace_if_not_exists(spark, "analysis")` | `my` (personal) | `my.analysis` |
| `create_namespace_if_not_exists(spark, "research", tenant_name="kbase")` | `kbase` (tenant) | `kbase.research` |

## Personal SQL Warehouse

Your personal catalog (`my`) is automatically provisioned and only accessible by you.

### Usage

```python
spark = get_spark_session("MyApp")

# Create namespace in your personal Iceberg catalog
ns = create_namespace_if_not_exists(spark, "analysis")
# Returns: "my.analysis"

# Write a table
df = spark.createDataFrame([(1, "Alice"), (2, "Bob")], ["id", "name"])
df.writeTo(f"{ns}.my_table").createOrReplace()

# Read it back
spark.sql(f"SELECT * FROM {ns}.my_table").show()
```

### Where Your Tables Go

- **Catalog**: `my` (dedicated Polaris catalog)
- **Namespace**: User-defined (e.g., `analysis`, `experiments`)
- **Full table path**: `my.{namespace}.{table_name}` (`{username}.{namespace}.{table_name}` in Trino — see [Querying from Trino](#querying-from-trino))
- **S3 location**: Managed automatically by Polaris under `s3a://cdm-lake/users-sql-warehouse/{username}/`. Do not write files there yourself: that prefix is list-only for you, and only Spark writes into it. Non-table files (CSV, TSV, staged inputs) go in `s3a://cdm-lake/users-general-warehouse/{username}/` — see the [S3 guide](s3_guide.md#what-you-can-access-and-why-you-get-accessdenied).

## Tenant SQL Warehouse

Tenant catalogs are shared across all members of a tenant/group.

### Usage

```python
spark = get_spark_session("TeamApp")

# Create namespace in tenant Iceberg catalog
ns = create_namespace_if_not_exists(
    spark, "research", tenant_name="kbase"
)
# Returns: "kbase.research"

# Write a shared table
df.writeTo(f"{ns}.shared_dataset").createOrReplace()

# All kbase members can read this table
spark.sql(f"SELECT * FROM {ns}.shared_dataset").show()
```

### Where Your Tables Go

- **Catalog**: Tenant name (e.g., `kbase`)
- **Namespace**: User-defined (e.g., `research`, `shared_data`)
- **Full table path**: `{tenant}.{namespace}.{table_name}` (same name in Trino)
- **S3 location**: Managed automatically by Polaris under `s3a://cdm-lake/tenant-sql-warehouse/{tenant}/`. As with your personal warehouse, that prefix is list-only; non-table files shared with the tenant go in `s3a://cdm-lake/tenant-general-warehouse/{tenant}/`.

### Requirements

- You must be a member of the specified tenant/group
- The system validates your membership before allowing access

## Spark Connect Architecture

BERDL uses **Spark Connect**, which provides a client-server architecture:
- **Connection Protocol**: Uses gRPC for efficient communication with remote Spark clusters
- **Spark Connect Server**: Runs locally in your notebook pod as a proxy
- **Spark Connect URL**: `sc://localhost:15002`
- **Driver and Executors**: Runs locally in your notebook pod

### Automatic Credential Management

Your credentials are automatically configured when your notebook server starts:

1. **JupyterHub** calls the governance API to create/retrieve your S3 and Polaris credentials
2. **Environment Variables** are set in your notebook (S3 keys + Polaris OAuth2 tokens)
3. **Spark Session** is pre-configured with access to your personal catalog and tenant catalogs
4. **Notebooks** use credentials from environment (no API call needed)

This ensures secure, per-user access without exposing credentials in code.

## Cross-Catalog Queries

You can query across personal and tenant catalogs in a single query:

```python
spark.sql("""
    SELECT u.name, d.dept_name
    FROM my.analysis.users u
    JOIN kbase.shared.depts d
    ON u.dept_id = d.id
""")
```

## Working with Iceberg Tables

In the examples below, `my` is your personal catalog; a tenant catalog (`kbase.research.my_table`) works the same way.

### Write a Table

```python
# Create the table, or replace it if it exists
df.writeTo("my.analysis.my_table").createOrReplace()

# Append rows to an existing table
df.writeTo("my.analysis.my_table").append()
```

`df.write.format("iceberg").saveAsTable(...)` also works; `writeTo` is recommended because it supports `createOrReplace()` and `append()` natively.

> **Bulk ingestion?** For config-driven loading from CSV/TSV/JSON/XML/Parquet sources, the auto-imported `ingest(config)` function from `data_lakehouse_ingest` handles namespace creation, schema enforcement, and writes Iceberg Silver tables in one call. See its docstring for the config schema.

### List Namespaces and Tables

```python
# Every namespace you can access, across your personal and tenant catalogs,
# as "my.analysis", "alice.analysis", "kbase.genomes", ... (usable directly in queries)
get_databases()

# Tables in one namespace; "my" and your username are aliases for the same catalog
get_tables("kbase.genomes")
get_tables("my.analysis")
get_tables("alice.analysis")
```

### Drop a Table

```python
# PURGE removes both the catalog entry and the underlying S3 data files
spark.sql("DROP TABLE IF EXISTS my.analysis.my_table PURGE")
```

Plain `DROP TABLE` (no `PURGE`) removes only the catalog entry and leaves the data files in S3.

### Drop a Whole Namespace (and its S3 data)

Polaris does not support `DROP NAMESPACE ... CASCADE`: a namespace can only be dropped once it is empty. To remove a namespace and its data, `DROP TABLE ... PURGE` every table first, then drop the empty namespace:

```python
ns = "my.analysis"

# return_json=False gives a Python list to iterate
for table in get_tables(ns, return_json=False):
    spark.sql(f"DROP TABLE IF EXISTS {ns}.{table} PURGE")

spark.sql(f"DROP NAMESPACE IF EXISTS {ns}")
```

`get_tables(ns)` returns a JSON string by default, hence `return_json=False`. Skipping `PURGE` leaves orphaned files in S3 even after the namespace is gone.

### Time Travel

```python
# Snapshot history
spark.sql("SELECT * FROM my.analysis.my_table.snapshots")

# The table as it was at a snapshot
spark.sql("SELECT * FROM my.analysis.my_table VERSION AS OF 1234567890")

# The table as it was at a point in time
spark.sql("SELECT * FROM my.analysis.my_table TIMESTAMP AS OF '2026-03-01 12:00:00'")
```

### Schema Evolution

Change a table's schema without rewriting its data:

```python
spark.sql("ALTER TABLE my.analysis.my_table ADD COLUMN email STRING")
spark.sql("ALTER TABLE my.analysis.my_table RENAME COLUMN name TO full_name")
spark.sql("ALTER TABLE my.analysis.my_table DROP COLUMN temp_field")
```

### Inspect Snapshots, Files and History

```python
display_df(spark.sql("SELECT * FROM my.analysis.my_table.snapshots"))
display_df(spark.sql("SELECT * FROM my.analysis.my_table.files"))
display_df(spark.sql("SELECT * FROM my.analysis.my_table.history"))
```

## Querying from Trino

Tables you write here are also readable from Trino (read-only). Catalog names are the same in both engines with one exception: `my` is a **Spark-only** alias. In Trino your personal catalog appears under your username, so `my.analysis.my_table` in Spark is `{username}.analysis.my_table` in Trino.

Your username also works in Spark, so it is the portable form. Use it whenever you store a table reference somewhere durable — a registry, a dashboard config, a notebook you hand to someone else — and keep `my` for interactive work. See the [Trino User's Guide](trino_guide.md#spark-vs-trino-naming) for the full catalog layout and example queries.

## Tips

- **Always use the returned namespace**: `create_namespace_if_not_exists()` returns the fully qualified namespace (e.g., `my.analysis`). Always use this value when creating or querying tables.
- **Tenant membership required**: Attempting to access a tenant warehouse without membership will fail.
- **Credentials are automatic**: S3 and Polaris credentials are set by JupyterHub — you don't need to call any API to get them.
- **Spark Connect is default**: All sessions use Spark Connect for better stability and resource isolation.
- **Iceberg features**: Your tables support time travel, schema evolution, and snapshot management. See [Working with Iceberg Tables](#working-with-iceberg-tables).
- **Old `u_<user>__<db>` or `<tenant>_<db>` names**: Delta Lake is retired; see [Delta Lake Retirement](iceberg_migration_guide.md) for the new names.
- **Persist the portable name**: `my` does not exist in Trino. Store `{username}.{namespace}.{table}` when a reference has to outlive the session or be used outside Spark (see [Querying from Trino](#querying-from-trino)).
