# Delta Lake Retirement

Delta Lake and the Hive Metastore were retired on `YYYY-MM-DD`. Every BERDL table is now an Apache Iceberg table in an Apache Polaris catalog: your personal catalog (`my` in Spark, or your username) and one catalog per tenant (e.g. `kbase`).

## What happened to my tables

Your Delta tables were migrated to Iceberg before the retirement. The data is the same; only the name changed. Find a table's new name here:

| Old Delta/Hive name | New Iceberg name |
|---------------------|------------------|
| `u_<user>__<db>.<table>` | `my.<db>.<table>` (Spark only) or `<user>.<db>.<table>` (Spark and Trino) |
| `<tenant>_<db>.<table>` | `<tenant>.<db>.<table>` |

For example, `u_alice__analysis.results` is now `my.analysis.results` (or `alice.analysis.results`), and `kbase_genomes.contigs` is now `kbase.genomes.contigs`.

`get_databases()` lists every namespace you can reach, already in the `catalog.namespace` form.

## Trino

Trino needs catalog-qualified names: `<user>.<db>.<table>` for your own tables and `<tenant>.<db>.<table>` for tenant tables. The per-user Delta catalog and the `u_<user>__<db>` names behind it no longer exist, and `my` is not available in Trino. See the [Trino User's Guide](trino_guide.md#spark-vs-trino-naming).

## Code that no longer works

| No longer works | Use instead |
|-----------------|-------------|
| `df.write.format("delta").saveAsTable(name)` | `df.writeTo("my.<db>.<table>").createOrReplace()` (or `.append()`) |
| `spark.read.format("delta").load(path)` | `spark.table("my.<db>.<table>")`; read tables by name, not by S3 path |
| `DeltaTable` (`from delta.tables import DeltaTable`) | Spark SQL `MERGE INTO`, `UPDATE` and `DELETE` on the Iceberg table |
| `DESCRIBE HISTORY <table>` | `SELECT * FROM my.<db>.<table>.history` |
| `create_namespace_if_not_exists(spark, ns, iceberg=False)` | Drop the `iceberg` argument |
| `share_table()`, `unshare_table()`, `make_table_public()`, `make_table_private()`, `get_table_access_info()` | Write the table to a tenant catalog (see the [Data Sharing Guide](data_sharing_guide.md)) |
| `get_namespace_prefix()` | Not needed: each catalog is isolated, so namespaces carry no prefix |
| `get_spark_session(delta_lake=..., use_hive=..., tenant_name=...)` | `get_spark_session()`; write tenant tables as `<tenant>.<db>.<table>` |
| `get_trino_connection(connector=...)` | `get_trino_connection()` |

Creating, querying and maintaining Iceberg tables, including time travel and schema evolution, is covered in the [Tenant SQL Warehouse Guide](tenant_sql_warehouse_guide.md#working-with-iceberg-tables).

## Getting help

If a table is missing or its data looks different after the migration, reach out to the BERDL Platform team with the old name and the new name you tried.
