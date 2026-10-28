# Data Sharing Guide: BERDL JupyterHub

## Overview

The BERDL (BER Data Lake) JupyterHub platform lets you share datasets with other users through **tenants**: groups whose members share an Iceberg catalog for tables (e.g. `kbase`) and a storage prefix for files.

The everyday data governance functions are **✨ automatically imported** in every notebook - no manual imports needed (admin-only group management needs an import; see the [Admin guide](admin_user_management_guide.md)). This guide shows you how to use these pre-loaded functions for data sharing and collaboration.

## Key Concepts

### Data Organization

All user data in BERDL is organized under personal namespaces:

- **General Data Storage**: `s3a://cdm-lake/users-general-warehouse/{username}/` — free-form files you write yourself (CSV, TSV, staged inputs)
- **SQL Warehouse**: `s3a://cdm-lake/users-sql-warehouse/{username}/` — your tables, written by Spark and managed by the catalog. The root is list-only, so don't put files here directly (see the [S3 guide](s3_guide.md#what-you-can-access-and-why-you-get-accessdenied))

Each tenant has the same pair, shared by its members:

- **Tenant tables**: the tenant's Iceberg catalog (e.g. `kbase.research.my_table`), stored under `s3a://cdm-lake/tenant-sql-warehouse/{tenant}/`
- **Tenant files**: `s3a://cdm-lake/tenant-general-warehouse/{tenant}/` — free-form files shared with the tenant

## Getting Started

### Auto-Imported Functions

Every BERDL notebook imports these data governance functions when its kernel starts, so they're ready to use without any imports:

**Auto-Imported Utility Functions:**

*Core Information:*
- `check_governance_health()` - Check service status
- `get_credentials()` - Get your S3 and Polaris credentials (sets environment variables). Assign the result (`creds = get_credentials()`) so the secrets are not printed into your notebook
- `get_my_sql_warehouse()` - Get your SQL warehouse root (list-only at the top; tables go under it via `create_namespace_if_not_exists()`, files under `users-general-warehouse/<user>/`)
- `get_my_workspace()` - Get comprehensive workspace information
- `get_my_groups()` - Get list of groups you belong to
- `get_my_policies()` - Get detailed policy information
- `get_my_accessible_paths()` - Get every S3 prefix you can read; `.paths` marks each one `read` or `write`

*Tenant Functions:*
- `list_tenants()` - Every tenant, with member counts and whether you are a member or steward
- `show_my_tenants()` - Print the tenants you belong to, with description, stewards and storage paths (`show_databases=True` adds namespaces and tables)
- `get_tenant_detail(tenant_name)` - One tenant's description, stewards and storage paths
- `get_group_sql_warehouse(group_name)` - Get the table root of a tenant you belong to
- `list_available_groups()`, `request_tenant_access(tenant_name, permission, justification)` - Ask to join a tenant
- `add_tenant_member(tenant_name, username, permission="read_write")`, `remove_tenant_member(tenant_name, username)` - Add or remove members (tenant stewards and admins)

*Admin Functions:*
- `create_tenant_and_assign_users(tenant_name, usernames)` - Create a tenant and add users (admin only)

**Pre-Initialized Client:**
- `governance` - The low-level governance API client, created when the kernel starts. Prefer the functions above; for a raw API call, get a current client with `get_governance_client()`

**Other Auto-Imported Functions:**
- `get_spark_session()` - Create Spark sessions with your Iceberg catalogs
- `create_namespace_if_not_exists()` - Create namespaces in your personal catalog or, with `tenant_name=`, a tenant catalog
- Plus many other utility functions for data operations

> **Note:** **Tenant catalogs** are how you share tables. Create tables in a tenant catalog (e.g., `kbase`) and all members can access them. See [Sharing Tables Through a Tenant](#sharing-tables-through-a-tenant) below.

### Quick Start

```python
# Check your workspace information
workspace = get_my_workspace()
print(f"👤 Username: {workspace.username}")
print(f"🏠 Home paths: {workspace.home_paths}")
print(f"👥 Groups: {workspace.groups}")
print(f"📁 Accessible paths: {len(workspace.accessible_paths)}")

# Check service health
health = check_governance_health()
print(f"🔍 Service status: {health.status}")
```

## Core Information Functions

### Service Health and Credentials

```python
# Check governance service status
health = check_governance_health()
print(f"Service status: {health.status}")

# Get your SQL warehouse root. This is where your tables live, not a folder
# to write into: create tables through create_namespace_if_not_exists() +
# Spark, and put free-form files under users-general-warehouse/<user>/.
sql_warehouse = get_my_sql_warehouse()
print(f"SQL warehouse root: {sql_warehouse.sql_warehouse_prefix}")

# The pre-initialized governance client is also available
print(f"Governance client ready: {governance is not None}")
```

### Workspace Information

```python
# Get comprehensive workspace information
workspace = get_my_workspace()
print(f"Username: {workspace.username}")
print(f"Home paths: {workspace.home_paths}")
print(f"Groups: {workspace.groups}")
print(f"Total accessible paths: {len(workspace.accessible_paths)}")

# Get list of groups you belong to
my_groups = get_my_groups()
print(f"Your groups: {my_groups.groups}")
print(f"Total groups: {my_groups.group_count}")

# Get detailed policy information
policies = get_my_policies()
print(f"User policies: {policies}")
```

## Tenant and Group Management

### Creating Tenants (Admin Only)

Tenants (groups) enable collaborative workspaces where multiple users can share data. **Only admin users** can create tenants.

```python
# Create a new tenant and add users to it
result = create_tenant_and_assign_users(
    tenant_name="kbase",
    usernames=["alice", "bob", "charlie"]
)

# Check creation result
print(f"Tenant creation: {result['create_tenant']}")
print(f"\nUser assignments:")
for username, response in result['add_members']:
    print(f"  {username}: {response}")
```

### Checking Your Group Memberships

```python
# Get list of all groups you belong to
my_groups = get_my_groups()
print(f"Your groups: {my_groups.groups}")
print(f"Total: {my_groups.group_count}")

# Get SQL warehouse prefix for a specific group
group_warehouse = get_group_sql_warehouse("kbase")
print(f"Group SQL warehouse: {group_warehouse.sql_warehouse_prefix}")
```

## Sharing Tables Through a Tenant

Write a table to a tenant catalog and every member of that tenant can query it. Members of the tenant's read-only group (`<tenant>ro`) can read the tables but not change them.

```python
spark = get_spark_session()

# Create a namespace in the kbase tenant catalog and write a table to it
ns = create_namespace_if_not_exists(spark, "research", tenant_name="kbase")
# Returns: "kbase.research"
df.writeTo(f"{ns}.climate_data").createOrReplace()
```

To share files that are not tables (CSV, TSV, raw exports), put them under the tenant's file prefix, `s3a://cdm-lake/tenant-general-warehouse/<tenant>/`. Writing there needs read-write membership; read-only members can browse and download. Some tenants give each member a personal folder instead (`users/<username>/`); if a write to the tenant folder is denied, write under that folder (see the [S3 guide](s3_guide.md#per-user-write-areas)).

To give someone access, add them to the tenant: a steward uses `add_tenant_member()`, and anyone can ask to join with `request_tenant_access()` (see [Requesting Tenant Access](requesting-tenant-access.md)). There is no per-table or per-path sharing; access follows tenant membership.

## Managing Access Information

### View Your Complete Workspace

```python
# Get comprehensive workspace information
workspace = get_my_workspace()

print(f"🏠 Your username: {workspace.username}")
print(f"📁 Home directories: {len(workspace.home_paths)}")
for path in workspace.home_paths:
    print(f"   - {path}")

print(f"👥 Group memberships: {len(workspace.groups)}")
for group in workspace.groups:
    print(f"   - {group}")

print(f"🔓 Total accessible paths: {len(workspace.accessible_paths)}")

# Show shared paths (paths not owned by you)
shared_paths = [
    path for path in workspace.accessible_paths 
    if not any(path.startswith(home) for home in workspace.home_paths)
]
print(f"🤝 Shared with you: {len(shared_paths)}")
for path in shared_paths[:5]:  # Show first 5
    print(f"   - {path}")
```

## Working with Shared Tables

### Accessing Shared Tables in Spark

Tables shared through a tenant catalog are queried like any other Iceberg table, by `catalog.namespace.table`:

```python
# get_spark_session is auto-imported, no need to import
spark = get_spark_session()

# A table a colleague wrote to the kbase tenant catalog
shared_df = spark.sql("SELECT * FROM kbase.research.climate_data")
print(f"📊 Loaded shared table with {shared_df.count()} rows")
shared_df.show(5)
```

If the query fails with `TABLE_OR_VIEW_NOT_FOUND` or a permission error, check `get_my_groups(force_refresh=True)`. Spark only knows the catalogs of tenants you belong to, so a tenant you are not in looks like a missing table. If the tenant is missing, request access (see [Requesting Tenant Access](requesting-tenant-access.md)); if you were just added, restart Spark Connect first (see [After Your Request Is Approved](requesting-tenant-access.md#after-your-request-is-approved)).

> **Delta Lake is retired:** tables that used to be shared as Delta paths or `u_<user>__<db>` / `<tenant>_<db>` databases are now Iceberg tables. See [Delta Lake Retirement](iceberg_migration_guide.md) for their new names.

## Troubleshooting

### Common Issues

1. **Table Not Found**: Ensure the table exists and you have access
   ```python
   # Namespaces you can see across your personal and tenant catalogs
   get_databases()
   # Tables in one namespace
   get_tables("kbase.research")
   ```
   Tenant tables are named `<tenant>.<namespace>.<table>`; if the tenant was added after your server started, restart Spark Connect first (item 5).

2. **User Not Found**: Verify usernames are correct and users exist in the system

3. **Namespace Not Found**: Confirm database/namespace names are correct
    ```python
    # Check if namespace exists
    get_databases()
    ```

4. **Permission Denied on a tenant table**: Confirm you are a member of the tenant with `get_my_groups()`. Reading needs the tenant or its read-only group; writing needs the tenant itself.

5. **New tenant access not taking effect**: After you are added to a tenant, restart Spark Connect with `start_spark_connect_server(force_restart=True)` and create a new Spark session (see [Requesting Tenant Access](requesting-tenant-access.md#after-your-request-is-approved)).


### Getting Help
❓ If you have any questions, feel free to reach out to the BERDL Platform Team.
