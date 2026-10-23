# JupyterHub Admin: User & Tenant Management Guide

This guide covers administrative operations for managing users, groups (tenants), and their access in BERDL. These operations require **JupyterHub Admin** privileges.

---

## Prerequisites

### Required Role: `CDM_JUPYTERHUB_ADMIN`

To perform user and tenant management operations, you must have the `CDM_JUPYTERHUB_ADMIN` role assigned via the **KBase Auth Server**.

> [!IMPORTANT]
> The `CDM_JUPYTERHUB_ADMIN` role must be granted by a KBase System Administrator. Contact them in the [#sysadmin Slack channel](https://kbase.slack.com/archives/C02MHNMCQ) to request this role.

Once you have the admin role, you can access management operations from within a BERDL JupyterHub notebook.

---

## Available Management Operations

Import the management functions:

```python
from berdl_notebook_utils.governance import (
    # List operations
    list_users,
    list_groups,
    # Group member management
    add_group_member,
    remove_group_member,
    # Tenant creation
    create_tenant_and_assign_users,
)
```

---

## User Management

### List All Users

Retrieve all users and their group memberships:

```python
from berdl_notebook_utils.governance import list_users

users = list_users()

# Print users nicely (username + groups)
print(f"{'Username':<20} {'Groups'}")
print("-" * 60)
for user in users.users:
    groups_str = ", ".join(user.groups) if user.groups else "None"
    print(f"{user.username:<20} {groups_str}")
print(f"\nTotal Jupyterhub users: {users.total_count}")
```

`list_users()` returns up to 500 users per page (`list_users(page=2)` for more) and is slow, because it loads each user's policies. For names only, use `list_user_names()` (same import).

---

## Group (Tenant) Management

### List All Groups

View all existing groups and their members:

```python
from berdl_notebook_utils.governance import list_groups

groups = list_groups()

# Print groups nicely (group_name + members)
print(f"{'Group Name':<20} {'Members'}")
print("-" * 60)
for group in groups['groups']:
    members_str = ", ".join(group['members']) if group['members'] else "None"
    print(f"{group['group_name']:<20} {members_str}")
print(f"\nTotal groups: {groups['total_count']}")
```

### Create a New Tenant

Create a new tenant (group) with optional initial members:

```python
from berdl_notebook_utils.governance import create_tenant_and_assign_users

# Create tenant (you, the creator, are its only member)
result = create_tenant_and_assign_users("kbase")

# Create tenant with initial members
result = create_tenant_and_assign_users(
    tenant_name="kbase",
    usernames=["alice", "bob", "charlie"]
)

# Check results
print(result["create_tenant"])  # Tenant creation status
print(result["add_members"])    # List of (username, status) tuples
```

Tenant names must be 2–64 lowercase letters and digits, start with a letter, and not end in `ro` (that suffix is reserved: creating `kbase` also creates the read-only group `kbasero`). Some names are reserved (for example `admin`, `public`, `default`). You, the creator, are added to both groups; remove yourself afterwards with `remove_tenant_member()` if you shouldn't be a member.

### Add Users to a Group

Add users to an existing group. By default, users are added to the read/write group. Use `read_only=True` to add users to the read-only variant instead.

```python
from berdl_notebook_utils.governance import add_group_member
from governance_client.models import GroupManagementResponse

# Add users to read/write group (default)
results = add_group_member(
    group_name="kbase",
    usernames=["alice", "bob"]
)

for username, response in results:
    if isinstance(response, GroupManagementResponse):
        print(f"Successfully added {username} to read/write group")
    else:
        print(f"Error adding {username}: {response}")

# Add users to read-only group
results = add_group_member(
    group_name="kbase",
    usernames=["charlie", "david"],
    read_only=True
)

for username, response in results:
    if isinstance(response, GroupManagementResponse):
        print(f"Successfully added {username} to read-only group")
    else:
        print(f"Error adding {username}: {response}")
```

### Remove Users from a Group

Remove users from a group. By default, users are removed from the read/write group. Use `read_only=True` to remove users from the read-only variant instead.

```python
from berdl_notebook_utils.governance import remove_group_member
from governance_client.models import GroupManagementResponse

# Remove users from read/write group (default)
results = remove_group_member(
    group_name="kbase",
    usernames=["alice", "bob"]
)

for username, response in results:
    if isinstance(response, GroupManagementResponse):
        print(f"Successfully removed {username} from read/write group")
    else:
        print(f"Error removing {username}: {response}")

# Remove users from read-only group
results = remove_group_member(
    group_name="kbase",
    usernames=["charlie"],
    read_only=True
)

for username, response in results:
    if isinstance(response, GroupManagementResponse):
        print(f"Successfully removed {username} from read-only group")
    else:
        print(f"Error removing {username}: {response}")
```

> [!WARNING]
> `add_group_member` / `remove_group_member` change **one group only**. Adding someone to `kbase` who is already in `kbasero` leaves them in both, and removing someone from `kbase` leaves their read-only access and any steward role. To move a member between read-only and read-write, or to remove them completely, use the tenant functions below.

### Tenant Members and Stewards

These functions are auto-imported in every notebook and keep a tenant's two groups consistent:

```python
# Add a member, or move an existing member between read-write and read-only
add_tenant_member("kbase", "alice", permission="read_only")   # default: "read_write"

# Remove a member from both groups (this also drops their steward role)
remove_tenant_member("kbase", "alice")

# Stewards (admin only): a steward can add and remove members and edit the
# tenant's description with update_tenant_metadata()
assign_steward("kbase", "bob")     # bob must already be a member
remove_steward("kbase", "bob")     # bob stays a member
```

---

## Operations Summary

| Operation | Function | Description |
|-----------|----------|-------------|
| List users | `list_users()` | Get all system users |
| List groups | `list_groups()` | Get all groups with members |
| Create tenant | `create_tenant_and_assign_users(name, users)` | Create new group with optional members |
| Add members (R/W) | `add_group_member(group, usernames)` | Add users to read/write group |
| Add members (RO) | `add_group_member(group, usernames, read_only=True)` | Add users to read-only group |
| Remove members (R/W) | `remove_group_member(group, usernames)` | Remove users from read/write group |
| Remove members (RO) | `remove_group_member(group, usernames, read_only=True)` | Remove users from read-only group |
| Add or move a tenant member | `add_tenant_member(tenant, username, permission=...)` | Add a user, or switch them between read-write and read-only |
| Remove a tenant member | `remove_tenant_member(tenant, username)` | Remove a user from both groups and their steward role |
| Assign / remove a steward | `assign_steward(tenant, username)`, `remove_steward(tenant, username)` | Let a member manage the tenant's members (admin only) |

---

## Requesting Admin Access

To obtain the `CDM_JUPYTERHUB_ADMIN` role:

1. Contact the KBase System Administrators in the [#sysadmin Slack channel](https://kbase.slack.com/archives/C02MHNMCQ)
2. Request the `CDM_JUPYTERHUB_ADMIN` role assignment
3. The admin will assign the role via the **KBase Auth Server**
4. Log out and log back into JupyterHub for changes to take effect

> [!NOTE]
> Without the role, these calls return an error response (HTTP 403, "Administrator privileges required for this operation") instead of data; print the returned value to see it.
