
# MinIO Guide

This guide covers two MinIO tools for BERDL object storage: the **MinIO UI** (a web
console) and the **MinIO Client** (`mc`). For the AWS CLI, `boto3`, the S3 paths
you can access and how credentials are kept current, see the
[Object Storage (S3) Guide](s3_guide.md).

## Your credentials

MinIO uses your BERDL S3 keys. In a notebook terminal:

```bash
aws configure get aws_access_key_id       # access key (MinIO UI username)
aws configure get aws_secret_access_key   # secret key (MinIO UI password)
```

These always show your current keys. After a credential rotation (see
[Credentials are kept current automatically](s3_guide.md#credentials-are-kept-current-automatically)),
log in to the UI again and re-run `mc alias set` with the new secret.

## Using `mc` in a notebook

`mc` is already installed in BERDL notebooks and needs no tunnel there. In a
notebook terminal:

```bash
mc alias set berdl "${AWS_ENDPOINT_URL_S3%/}" "$S3_ACCESS_KEY" "$S3_SECRET_KEY"
mc ls berdl/cdm-lake/users-general-warehouse/$NB_USER/
```

`mc` then works with the paths your policy grants; see [Basic commands](#basic-commands).

## Using MinIO from your laptop

The MinIO UI and the S3 endpoint are reachable only through the KBase SSH tunnel.

### Setting Up SSH Tunnel

**Step 1:** Ensure you have SSH access to the remote server (`login.kbase.us`)
   * If you do not have access, please contact the KBase System Admin team.

**Step 2:** Open a terminal on your machine and run the following command:

```bash
ssh -f -D 1338 -N <ac.anl_username>@login.kbase.us
```

**What this command does:**
* `-f`: Runs SSH in the background
* `-D 1338`: Creates a SOCKS5 proxy on port 1338
* `<ac.anl_username>`: Replace with your ANL CELS account (for example `ac.jsmith`), not your KBase username
* `-N`: No remote command is executed — useful for pure port-forwarding connections

> **⚠️ Note:** If port `1338` is already in use on your system, you can replace it with another free port (e.g., 1080).

**Step 3:** If you haven't set up SSH keys, you'll be prompted to enter your password.

**Step 4:** Verify the tunnel is running by checking for the background process:
```bash
ps aux | grep "ssh -f -D 1338"
```

> **💡 Tip:** To close the tunnel later, find the process ID (PID) and terminate it with `kill <PID>`.

### Using the MinIO UI

Point your browser at the tunnel: send `*.berdl.kbase.us` through the SOCKS5
proxy `127.0.0.1:1338`, for example with a proxy-switcher extension or Firefox's
**Settings → Network Settings → Manual proxy configuration** (SOCKS v5, with
**Proxy DNS when using SOCKS v5** ticked). Then open:

| Environment | MinIO UI |
|-------------|----------|
| Staging | `https://minio-ui.stage.berdl.kbase.us` |
| Production | `https://minio-ui.berdl.kbase.us` |

Log in with your access key as the username and your secret key as the password.
The UI is a web console only; S3 tools use the S3 endpoint, not this address.
In development, use `mc` or the AWS CLI instead.

### Installing `mc`

MinIO no longer publishes `mc` on its download site. Download it from its GitHub
release page,
[RELEASE.2025-08-13T08-35-41Z](https://github.com/minio/mc/releases/tag/RELEASE.2025-08-13T08-35-41Z)
(the final release, and the version BERDL notebooks use). Pick the
`mc.<os>-<arch>.RELEASE…` file for your system, rename it to `mc`, run
`chmod +x mc`, and put it on your `PATH`.

### Configuring `mc`

**⚠️ IMPORTANT**: The proxy settings below route ALL HTTP/HTTPS traffic in this terminal through the SOCKS proxy. Use a **dedicated terminal window** for MinIO operations, or unset them when you are done (step 4).

1. **Set up proxy environment variables** (`mc` accepts the SOCKS proxy directly):

    ```bash
    unset ALL_PROXY

    export HTTP_PROXY="socks5://127.0.0.1:1338"
    export HTTPS_PROXY="socks5://127.0.0.1:1338"
    ```

2. Add the S3 endpoint for your environment (the table in
   [Accessing storage from your laptop](s3_guide.md#accessing-storage-from-your-laptop);
   production shown):

    ```bash
    mc alias set berdl https://minio.berdl.kbase.us
    ```

    It will prompt you to enter the access key and secret key (see [Your credentials](#your-credentials)).

3. Verify the configuration and connection:

    ```bash
    mc ls berdl/cdm-lake/users-general-warehouse/<username>/
    ```

4. **Clean up proxy settings** when you're done with MinIO operations:

    ```bash
    unset HTTP_PROXY
    unset HTTPS_PROXY
    ```

    These settings only affect the current terminal session. Opening a new terminal will start fresh without them.

## Basic commands

You can't list or upload at the top of a bucket, only inside the folders your
policy grants: your own `cdm-lake/users-general-warehouse/<username>/`, and
`cdm-lake/tenant-general-warehouse/<tenant>/` for tenants you belong to. See
[What you can access](s3_guide.md#what-you-can-access-and-why-you-get-accessdenied).

* List your files:

    ```bash
    mc ls berdl/cdm-lake/users-general-warehouse/<username>/
    ```

* Upload a file:

    ```bash
    mc cp <local_file_path> berdl/cdm-lake/users-general-warehouse/<username>/
    ```

* Upload a directory:

    ```bash
    mc cp --recursive <local_directory_path> berdl/cdm-lake/users-general-warehouse/<username>/
    ```

* Download a file:

    ```bash
    mc cp berdl/cdm-lake/users-general-warehouse/<username>/<file_name> <local_file_path>
    ```

* Download a directory:

    ```bash
    mc cp --recursive berdl/cdm-lake/users-general-warehouse/<username>/<folder>/ <local_directory_path>
    ```

## Additional Resources
* [MinIO Client (mc) on GitHub](https://github.com/minio/mc)
* [Object Storage (S3) Guide](s3_guide.md) for the AWS CLI and `boto3`
