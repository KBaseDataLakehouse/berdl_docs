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

Because these are the standard locations the AWS SDKs and CLI read, any tool that
speaks S3 — the `aws` CLI, `boto3`, `s3fs`/`pandas`, `duckdb` — finds your
credentials and the right endpoint with no configuration.

> **Managed files — do not edit.** BERDL owns `~/.aws/credentials` and
> `~/.aws/config` and overwrites them on startup and on every credential
> rotation. Hand edits will be lost.

## Using the AWS CLI

The `aws` CLI is pre-installed and pre-configured. Just use it:

```bash
# List your buckets
aws s3 ls

# List the contents of a bucket
aws s3 ls s3://cdm-lake/

# Upload a file
aws s3 cp ./local_file.parquet s3://<bucket>/path/to/local_file.parquet

# Download a file
aws s3 cp s3://<bucket>/path/to/file.parquet ./file.parquet

# Sync a directory (recursive)
aws s3 sync ./local_dir/ s3://<bucket>/path/to/dir/
```

No `--endpoint-url` and no access keys are needed — they come from the managed
`~/.aws` files.

## Using Python (boto3)

Create a client with **no arguments** — `boto3` reads your credentials from
`~/.aws/credentials` and the endpoint from `~/.aws/config`:

```python
import boto3

s3 = boto3.client("s3")

# List buckets
for bucket in s3.list_buckets()["Buckets"]:
    print(bucket["Name"])

# Upload / download
s3.upload_file("local_file.parquet", "my-bucket", "path/to/local_file.parquet")
s3.download_file("my-bucket", "path/to/file.parquet", "file.parquet")
```

`pandas` (via `s3fs`) works the same way — it picks up the same credentials, so
`pd.read_parquet("s3://my-bucket/path/to/file.parquet")` just works.

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
```

Paths that return `AccessDenied` by design:

```bash
aws s3 cp f.tsv s3://cdm-lake/users-sql-warehouse/<user>/f.tsv   # a table root, not a file drop (see below)
aws s3 ls s3://cdm-lake/tenant-general-warehouse/                # spans every tenant
aws s3 ls s3://cdm-spark-job-logs/                               # Spark event logs (see below)
```

A few things that commonly trip people up:

- **You have two personal prefixes, and only one of them takes files.**
  `users-general-warehouse/<user>/` is yours to write: put CSVs, TSVs, raw
  exports, staged inputs, and anything else that is not a table there.
  `users-sql-warehouse/<user>/` is where your tables live and is managed by the
  catalog: you can list it, but a `PutObject` directly under it is denied. Your
  policy only allows writes inside its governed children (`iceberg/`, which
  Polaris manages for your `my` catalog, and `u_<user>__*` legacy Delta database
  directories). Create namespaces with `create_namespace_if_not_exists()` and
  write tables through Spark; never build a table path by hand from
  `get_my_sql_warehouse()`. Tenants work the same way:
  `tenant-general-warehouse/<tenant>/` takes files, and
  `tenant-sql-warehouse/<tenant>/` is the catalog-managed table root.
- **You can't list a whole bucket, or a parent prefix that spans other
  users/tenants** — only the specific prefixes your policy grants (your
  `users-general-warehouse/<user>/` and `users-sql-warehouse/<user>/`, and the
  `tenant-general-warehouse/<tenant>/` and `tenant-sql-warehouse/<tenant>/` of
  tenants you belong to). This is deliberate isolation between users and tenants.
- **`aws s3 ls` prefix matching has no implicit trailing slash.**
  `aws s3 ls s3://cdm-lake/tenant-general-warehouse/kbase` matches every prefix
  *starting with* `kbase`, so it lists both `kbase/` and `kbaseincubator/`. Add
  the trailing slash (`…/kbase/`) to list inside just that one.
- **`cdm-spark-job-logs` is effectively write-only for you** — it holds the Spark
  event logs the platform writes on your behalf, and your policy doesn't grant you
  `ListBucket` there, so `aws s3 ls` on it is denied even for your own prefix.

To see exactly which prefixes you're entitled to, run in a notebook cell:

```python
get_my_accessible_paths()   # the prefixes you can list/read
get_my_sql_warehouse()      # your SQL-warehouse root (list-only; tables land here via Spark)
get_my_policies()           # the raw IAM policy, including its s3:prefix conditions
```

## Credentials are kept current automatically

Your S3 credentials rotate from time to time. You do **not** need to reconfigure
anything when they do:

- `~/.aws/credentials` is rewritten with the new secret, and the AWS CLI and
  `boto3` re-read that file on every call — so they always use current
  credentials, even immediately after a rotation.
- New terminals and new notebook kernels automatically pick up the latest
  credentials.

You can also trigger a rotation yourself from any notebook cell:

```python
refresh_spark_environment()
```

This rotates your S3 (and Polaris) credentials with the platform, refreshes your
Spark session, and updates `~/.aws/credentials` in the same step. After it runs,
`aws s3 ls` and `boto3` continue to work with no further action.

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

**1. Get your keys** from a notebook: `aws configure get aws_access_key_id` and
`aws configure get aws_secret_access_key`.

**2. Open an SSH tunnel** to KBase (a SOCKS5 proxy on port 1338):

```bash
ssh -f -D 1338 -N <your_kbase_username>@login.kbase.us
```

- `-f` runs it in the background, `-D 1338` opens the SOCKS proxy, `-N` forwards
  ports only. Use a different local port if 1338 is taken.
- Verify with `ps aux | grep "ssh -f -D 1338"`; stop it later with `kill <PID>`.

**3. Configure a local AWS profile** for your environment's S3 endpoint and route
it through the proxy:

| Environment | S3 endpoint |
|-------------|-------------|
| Development | `https://minio.dev.berdl.kbase.us` |
| Staging | `https://minio.stage.berdl.kbase.us` |
| Production | `https://minio.berdl.kbase.us` |

> **⚠️ Endpoints may change.** These are the **legacy MinIO** URLs. BERDL is
> migrating object storage to Ceph, so the endpoint for your environment may be
> different — confirm the current S3 endpoint with a BERDL admin.

```bash
# Use the endpoint for your environment (production shown)
aws configure set aws_access_key_id     <your_access_key>            --profile berdl
aws configure set aws_secret_access_key <your_secret_key>            --profile berdl
aws configure set endpoint_url          https://minio.berdl.kbase.us --profile berdl
aws configure set region                us-east-1                    --profile berdl

# Route S3 traffic through the SOCKS proxy for this command
HTTPS_PROXY=socks5://127.0.0.1:1338 aws s3 ls --profile berdl
```

> **⚠️ Proxy scope:** exporting `HTTPS_PROXY`/`HTTP_PROXY` routes *all* traffic in
> that shell through the tunnel. Set it inline (as above) or use a dedicated
> terminal, and unset it when you're done.

## Additional resources

- [AWS CLI S3 reference](https://docs.aws.amazon.com/cli/latest/reference/s3/)
- [Boto3 documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/quickstart.html)
