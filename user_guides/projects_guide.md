# Projects Guide

> **If your notebook was already running on 24 November 2026**, restart your server once to get projects:
> stop it from the hub's control panel and start it again.

This guide explains how to keep the tables of one piece of work in a **project**, share the project with the
people you work with, and take your own copy of someone else's project.

## Overview

A project is a place for one piece of work, such as an analysis or a paper. It has:

- **its own tables**, which you write and read with Spark like any other table;
- **its own members**, whom you choose one by one.

Use a project when a few people work on the same tables. Use your personal catalog (`my`) for work that is
only yours, and a tenant for data that belongs to a whole group. See the
[Tenant SQL Warehouse guide](tenant_sql_warehouse_guide.md) for those two.

Each member has one of three roles:

| Role | Read tables | Write tables | Add members | Fork | Delete the project |
|------|-------------|--------------|-------------|------|--------------------|
| **author** (who created it) | yes | yes | yes, at either level | yes | yes |
| **read_write** | yes | yes | read-only members only | yes | no |
| **read_only** | yes | no | no | no | no |

Someone who is not a member cannot see a project at all: to them it does not exist.

The project functions are **automatically imported** in every notebook. No imports are needed.

## Create a project

```python
project = create_project("soil_microbes", "Soil microbes of the Sierra")

project["project_id"]       # 'soil_microbes_2jcq5j'
project["spark_namespace"]  # 'projects.soil_microbes_2jcq5j'
```

- The first argument is a short name for SQL: it starts with a letter and holds only letters, digits and
  underscores, up to 40 characters. Spaces and hyphens become underscores.
- The second is the project's name for people. It is optional and defaults to the short name.
- The **project ID** is the short name plus six random characters. You use it for every other call.
- You are the project's author. You can have up to 20 projects.

## Write and read tables

A project's tables live in the `projects` catalog, under the project's ID. `spark_namespace` holds the first
two parts of the name:

```python
spark = get_spark_session()
ns = project["spark_namespace"]

df = spark.createDataFrame([(1, "S-001"), (2, "S-002")], ["id", "sample"])
df.writeTo(f"{ns}.samples").createOrReplace()

spark.sql(f"SELECT * FROM {ns}.samples").show()
```

Every member uses the same name, for example `projects.soil_microbes_2jcq5j.samples`.

You do not create the namespace yourself: it exists as soon as the project does.

## See your projects

```python
for p in list_projects():
    print(p["spark_namespace"], p["role"], p["display_name"])
```

`list_projects()` returns the projects you are in, with your role in each. `get_project(project_id)` returns
one of them.

## Share a project

Add someone by their KBase username:

```python
add_project_member(project["project_id"], "alice")                  # read-only
add_project_member(project["project_id"], "bob", access="write")    # read and write
```

- The author adds members at either level and can change a member's level by adding them again.
- A read_write member can add read-only members.
- The person can use the project on their next query. Nothing has to be restarted.

List and remove members:

```python
list_project_members(project["project_id"])
remove_project_member(project["project_id"], "alice")
```

The author can remove anyone else. Any other member can remove themself to leave the project.

**A read_write member can change and drop the project's tables.** BERDL keeps no copy of a dropped table, so
give write access only to people who need it.

## Fork a project

A fork is a new project of your own that starts with a copy of another project's tables. Use it to try
something without touching the original.

```python
mine = fork_project("soil_microbes_2jcq5j", "soil_mine")

mine["fork"]["status"]      # 'complete'
spark.sql(f"SELECT count(*) FROM {mine['spark_namespace']}.samples").show()
```

- You can fork a project if you are its author or a read_write member.
- You become the new project's author. The original's members are **not** copied.
- Each table is copied as it stood when you asked, without its history. Later changes to the original do
  not reach your fork, and your changes do not reach the original.
- Only tables are copied, not views or files.
- A project with more than 5 GiB of table data cannot be forked for now.

By default `fork_project` waits until the copy ends. Always check `fork["status"]`: the new project exists
even when the copy did not finish.

| Status | Meaning | What to do |
|--------|---------|------------|
| `complete` | Every table was copied | Nothing |
| `failed` or `interrupted` | The copy stopped part-way | `retry_fork(project_id)`: it goes on from the tables already copied |
| `pending` or `copying` | Still running | `get_fork(project_id)` to look again |

For a large project, start the copy without waiting and check on it later:

```python
mine = fork_project("soil_microbes_2jcq5j", "soil_mine", wait=False)
get_fork(mine["project_id"])["status"]
```

## Delete a project

Only the author can delete a project, and only when it holds no tables. Deleting a project never deletes
data, so drop its tables first:

```python
ns = project["spark_namespace"]
for row in spark.sql(f"SHOW TABLES IN {ns}").collect():
    spark.sql(f"DROP TABLE {ns}.{row.tableName} PURGE")

delete_project(project["project_id"])
```

## Query a project from Trino and from AI assistants

- **Trino** shows a project as a schema of the `projects` catalog, for example
  `projects.soil_microbes_2jcq5j.samples`. Trino is read-only. See the [Trino guide](trino_guide.md).
- **AI assistants** (Claude Code and others that use the BERDL MCP server) list your projects as
  `projects.<project_id>` beside your other databases. See the
  [Claude Code guide](claude_code_guide.md).

Trino can take up to five minutes to follow a change: a new project or a new member may not show at once,
and a removed member can still read through Trino for up to five minutes. Spark follows a change on the
next query.

## Troubleshooting

| What you see | Why | What to do |
|--------------|-----|------------|
| `Project '…' not found` | The ID is wrong, or you are not a member | Check `list_projects()`; ask the author to add you |
| The catalog `projects` is not found | Your Spark server started before projects were available | Run `spark = get_spark_session()` again |
| Writing fails with `AccessDenied` or `Forbidden` | You are a read_only member | Ask the author for write access |
| `A read-only collaborator cannot fork a project` | Only the author and read_write members can fork | Ask the author for write access |
| `… a fork can copy at most 5.0 GiB for now` | The project is over the fork's size limit | Copy the tables you need with Spark into a project of your own |
| `delete_project` is refused | The project still holds tables | Drop the tables first |
| Trino says `Cannot access catalog projects` | Trino has not followed a recent change yet | Wait five minutes and try again |

## Function reference

| Function | What it does |
|----------|--------------|
| `create_project(slug, display_name=None, description="")` | Create a project; you are its author |
| `list_projects()` | The projects you are in, with your role |
| `get_project(project_id)` | One project you are in |
| `delete_project(project_id)` | Delete an empty project (author only) |
| `list_project_members(project_id)` | The project's members, the author first |
| `add_project_member(project_id, username, access="read")` | Add a member, or change their level; `access` is `"read"` or `"write"` |
| `remove_project_member(project_id, username)` | Remove a member, or leave the project yourself |
| `fork_project(project_id, slug, display_name=None, description="", wait=True, timeout=1800)` | Make your own project with a copy of this one's tables |
| `get_fork(project_id)` | How the copy into a fork is going |
| `retry_fork(project_id, wait=True, timeout=1800)` | Run a fork's copy again after it failed or was interrupted |
