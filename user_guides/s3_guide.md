# Object Storage (S3) Guide

BERDL stores data in S3-compatible object storage. Inside a BERDL notebook, your
S3 credentials and endpoint are **configured automatically** and kept current —
the AWS CLI and `boto3` work out of the box, with no keys or endpoint to set by
hand.

## How access is configured

When your notebook server starts, BERDL writes two standard AWS files for you:

| File | Contents |
|------|----------|
| `~/.aws/credentials` | Your S3 access key and secret key (the `[default]` profile). |
| `~/.aws/config` | The S3 endpoint (`endpoint_url`) and region for the `[default]` profile. |

Because these are the standard locations the AWS SDKs and CLI read, the `aws`
CLI, `boto3` and `s3fs`/`pandas` find your credentials and the right endpoint
with no configuration.

> **Managed files.** BERDL rewrites the `[default]` profile of
> `~/.aws/credentials` on startup and on every credential rotation, and the
> `[default]` profile of `~/.aws/config` on startup. Edits to `[default]` will be
> lost; other profiles you add are kept.

> **Public AWS buckets:** the endpoint is also set in environment variables, so
> every S3 call goes to BERDL storage, even one for a public AWS bucket. To read
> a public AWS bucket, give the AWS endpoint explicitly:
> `aws s3 ls s3://<public-bucket>/ --no-sign-request --endpoint-url https://s3.amazonaws.com`.

## Using the AWS CLI

The `aws` CLI is pre-installed and pre-configured. Just use it:

```bash
# List your buckets
aws s3 ls

# List your own files
aws s3 ls s3://cdm-lake/users-general-warehouse/<user>/

# Upload a file
aws s3 cp ./local_file.parquet s3://cdm-lake/users-general-warehouse/<user>/path/to/local_file.parquet

# Download a file
aws s3 cp s3://cdm-lake/users-general-warehouse/<user>/path/to/file.parquet ./file.parquet

# Sync a directory (recursive)
aws s3 sync ./local_dir/ s3://cdm-lake/users-general-warehouse/<user>/path/to/dir/
```

Replace `<user>` with your KBase username (in a terminal, `$NB_USER` holds it).
No `--endpoint-url` and no access keys are needed — they come from the managed
`~/.aws` files. You can't list a bucket itself (`aws s3 ls s3://cdm-lake/` is
denied), only the folders you have access to; see
[What you can access](#what-you-can-access-and-why-you-get-accessdenied).

## Using Python (boto3)

Create a client with **no arguments** — `boto3` finds your credentials and the
endpoint on its own:

```python
import boto3

s3 = boto3.client("s3")

# List buckets
for bucket in s3.list_buckets()["Buckets"]:
    print(bucket["Name"])

# Upload / download (replace <user> with your KBase username)
s3.upload_file("local_file.parquet", "cdm-lake", "users-general-warehouse/<user>/path/to/local_file.parquet")
s3.download_file("cdm-lake", "users-general-warehouse/<user>/path/to/file.parquet", "file.parquet")
```

`pandas` (via `s3fs`) works the same way — it picks up the same credentials, so
`pd.read_parquet("s3://cdm-lake/users-general-warehouse/<user>/path/to/file.parquet")`
just works.

## The `lakehouse_minio` folder in the file browser

The JupyterLab file browser shows your object storage as a folder named
`lakehouse_minio` (most accounts also have it under **FAVORITES → Lakehouse
Objectstore**; to add it, right-click the folder and choose **Add Favorite**).
You can browse, open, upload and download files there with the mouse.

It is a view of S3, not a folder on disk, so it exists **only in the file
browser**. A terminal (`ls ~`), Python's `open()` or `os.listdir()`, and shell
scripts won't find it. From a terminal or from code, use the S3 path behind each
folder with the AWS CLI, `boto3` or `pandas`:

| File browser folder | S3 path | What it holds |
|---|---|---|
| `lakehouse_minio/my-files` | `s3://cdm-lake/users-general-warehouse/<user>/` | Your files (read/write) |
| `lakehouse_minio/my-sql` | `s3://cdm-lake/users-sql-warehouse/<user>/` | Your tables, managed by the catalog (see below) |
| `lakehouse_minio/<tenant>-files` | `s3://cdm-lake/tenant-general-warehouse/<tenant>/` | A tenant's files (read/write; in [some tenants](#per-user-write-areas) you write only under `users/<user>/`) |
| `lakehouse_minio/<tenant>-sql` | `s3://cdm-lake/tenant-sql-warehouse/<tenant>/` | A tenant's tables |
| `lakehouse_minio/<tenant>-files-ro`, `<tenant>-sql-ro` | The same two tenant paths | Tenants you have read-only access to |

For example, to list in a terminal what the file browser shows as
`lakehouse_minio/my-files`:

```bash
aws s3 ls s3://cdm-lake/users-general-warehouse/<user>/
```

A few notes:

- **Large files:** upload multi-GB files with `aws s3 cp`: it is faster than the
  file browser and doesn't depend on your browser tab staying open.
- **New tenants:** after you're added to a tenant, its folders appear here within
  about a minute; see [Requesting Tenant Access](requesting-tenant-access.md).

## What you can access (and why you get `AccessDenied`)

Your credentials authenticate you, but a MinIO IAM policy decides which paths you
can list and read. Access is scoped to **your** data, so an `AccessDenied` on a
path you aren't entitled to is expected isolation — not a broken setup or a bad key.

Paths you can list/read:

```bash
aws s3 ls                                                     # your buckets
aws s3 ls s3://cdm-lake/users-general-warehouse/<user>/       # your files (read/write)
aws s3 ls s3://cdm-lake/users-sql-warehouse/<user>/           # your tables (list-only root, see below)
aws s3 ls s3://cdm-lake/tenant-general-warehouse/<tenant>/    # a tenant's files (if you belong to it)
aws s3 ls s3://cdm-spark-job-logs/spark-job-logs/<user>/      # your Spark event logs
```

Paths that return `AccessDenied` by design:

```bash
aws s3 ls s3://cdm-lake/                                         # a whole bucket
aws s3 cp f.tsv s3://cdm-lake/users-sql-warehouse/<user>/f.tsv   # a table root, not a file drop (see below)
aws s3 ls s3://cdm-lake/tenant-general-warehouse/                # spans every tenant
aws s3 ls s3://cdm-spark-job-logs/                               # every user's Spark event logs
```

A few things that commonly trip people up:

- **You have two personal prefixes, and only one of them takes files.**
  `users-general-warehouse/<user>/` is yours to write: put CSVs, TSVs, raw
  exports, staged inputs, and anything else that is not a table there.
  `users-sql-warehouse/<user>/` is where your tables live and is managed by the
  catalog: you can list it, but a `PutObject` directly under it is denied. Your
  policy only allows writes inside its governed `iceberg/` child, which Polaris
  manages for your `my` catalog. Create namespaces with
  `create_namespace_if_not_exists()` and write tables through Spark; never build
  a table path by hand from `get_my_sql_warehouse()`. You may also see
  `u_<user>__…` folders under `users-sql-warehouse/<user>/`: they hold your old
  Delta tables (see the [Iceberg Migration Guide](iceberg_migration_guide.md)),
  are read-only, and will be deleted. Tenants work the same way:
  `tenant-general-warehouse/<tenant>/` takes files (with read-write membership),
  and `tenant-sql-warehouse/<tenant>/` is the catalog-managed table root.
- <a id="per-user-write-areas"></a>**Some tenants give each member a personal
  folder to write in.** In these tenants (for example `globalusers`), everyone
  in the tenant can read all of `tenant-general-warehouse/<tenant>/`, but you can
  write only under `tenant-general-warehouse/<tenant>/users/<user>/`; the
  tenant's stewards can also write under `root/`. If an upload to the tenant
  folder is denied, put the file in your own `users/<user>/` folder there.
- **You can't list a whole bucket, or a parent prefix that spans other
  users/tenants** — only the specific prefixes your policy grants (your
  `users-general-warehouse/<user>/` and `users-sql-warehouse/<user>/`, and the
  `tenant-general-warehouse/<tenant>/` and `tenant-sql-warehouse/<tenant>/` of
  tenants you belong to). This is deliberate isolation between users and tenants.
- **`aws s3 ls` prefix matching has no implicit trailing slash.**
  `aws s3 ls s3://cdm-lake/tenant-general-warehouse/kbase` matches every prefix
  *starting with* `kbase`, so it lists both `kbase/` and `kbaseincubator/`. Add
  the trailing slash (`…/kbase/`) to list inside just that one.
- **`cdm-spark-job-logs` holds the Spark event logs the platform writes for
  you.** You can list and read only your own folder,
  `aws s3 ls s3://cdm-spark-job-logs/spark-job-logs/<user>/`; listing the bucket
  itself is denied.

To see exactly which prefixes you're entitled to, run in a notebook cell:

```python
get_my_accessible_paths()   # the prefixes you can list/read
get_my_sql_warehouse()      # your SQL-warehouse root (list-only; tables land here via Spark)
get_my_policies()           # the raw IAM policy, including its s3:prefix conditions
```

## Credentials are kept current automatically

Your S3 credentials change only when they are rotated (by you, or by an
administrator). Most tools follow a rotation on their own:

- **Terminals:** the `aws` CLI and scripts you start in a terminal read
  `~/.aws/credentials`, which is rewritten with the new secret. A terminal that
  is already open picks up the change at its next prompt.
- **New notebook kernels** fetch your current credentials when they start, and
  the notebook you rotated in is updated in the same step.
- **Notebooks that were already open** keep the old secret for `boto3`, `pandas`
  and `!aws`. In each one, run `refresh_spark_environment()` (without
  `rotate=True`) or restart the kernel.

You can also trigger a rotation yourself from any notebook cell:

```python
refresh_spark_environment(rotate=True)
```

This rotates your S3 (and Polaris) credentials with the platform, refreshes your
Spark session, and updates `~/.aws/credentials` in the same step. After it runs,
`aws s3 ls` in a terminal and `boto3` in this notebook keep working; other open
notebooks need the step above. Rotate only
when you need new credentials (for example, after a secret was exposed): the old
ones stop working at once in every other notebook and script. To fix S3 or Spark
errors, work through [Fixing Spark Problems](user_guide.md#fixing-spark-problems)
first; rotating is its last step.

> **Long-running jobs:** a `boto3` client or open file handle created *before* a
> rotation keeps using the old credentials for the life of that object. Create a
> fresh client after rotating if a long job outlives a rotation.

## Finding your credentials and endpoint

Usually you never need the raw values, but if you do:

```bash
# From a terminal
aws configure get aws_access_key_id
aws configure get aws_secret_access_key
aws configure get endpoint_url
```

```python
# From a notebook — the access/secret key are also exposed as env vars
import os
print(os.environ["S3_ACCESS_KEY"])   # access key
# Keep your secret key private; avoid printing it in shared notebooks.
```

## Accessing storage from your laptop

The steps above cover the notebook environment. To reach the object store from
your own machine, you need the KBase SSH tunnel and your S3 keys.

**1. Get your keys** in a notebook terminal: `aws configure get aws_access_key_id`
and `aws configure get aws_secret_access_key`. Copy them again after a credential
rotation.

**2. Open an SSH tunnel** to KBase (a SOCKS5 proxy on port 1338):

```bash
ssh -f -D 1338 -N <ac.anl_username>@login.kbase.us
```

- Log in with your ANL CELS account (for example `ac.jsmith`), not your KBase
  username. If you do not have access to `login.kbase.us`, contact the KBase
  System Admin team.
- `-f` runs it in the background, `-D 1338` opens the SOCKS proxy, `-N` forwards
  ports only. Use a different local port if 1338 is taken.
- Verify with `ps aux | grep "ssh -f -D 1338"`; stop it later with `kill <PID>`.

**3. Configure a local AWS profile** with the same S3 endpoint your notebook
uses:

| Environment | S3 endpoint |
|-------------|-------------|
| Development | `https://minio.dev.berdl.kbase.us:31276` |
| Staging | `https://minio.stage.berdl.kbase.us` |
| Production | `https://minio.berdl.kbase.us` |

If an endpoint changes, `aws configure get endpoint_url` in a notebook terminal
always shows the current one.

```bash
# Use the endpoint for your environment (production shown)
aws configure set aws_access_key_id     <your_access_key>            --profile berdl
aws configure set aws_secret_access_key <your_secret_key>            --profile berdl
aws configure set endpoint_url          https://minio.berdl.kbase.us --profile berdl
aws configure set region                us-east-1                    --profile berdl
```

**4. Route the AWS CLI through the tunnel.** The AWS CLI and `boto3` can't use a
SOCKS proxy directly, so run a small local HTTP proxy that forwards into the
tunnel, and point the CLI at it:

```bash
# Terminal 1: an HTTP-to-SOCKS bridge (needs uv, https://docs.astral.sh/uv/;
# any HTTP-to-SOCKS proxy works)
uvx pproxy -l http://127.0.0.1:3128 -r socks5://127.0.0.1:1338

# Terminal 2
HTTPS_PROXY=http://127.0.0.1:3128 aws s3 ls s3://cdm-lake/users-general-warehouse/<user>/ --profile berdl
```

Laptop `boto3` scripts need the same `HTTPS_PROXY=http://127.0.0.1:3128` and
`boto3.Session(profile_name="berdl").client("s3")`. The MinIO client `mc`
accepts the SOCKS proxy directly; see the [MinIO Guide](minio_guide.md).

> **⚠️ Proxy scope:** exporting `HTTPS_PROXY`/`HTTP_PROXY` routes *all* traffic in
> that shell through the tunnel. Set it inline (as above) or use a dedicated
> terminal, and unset it when you're done.

## Additional resources

- [AWS CLI S3 reference](https://docs.aws.amazon.com/cli/latest/reference/s3/)
- [Boto3 documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/quickstart.html)
