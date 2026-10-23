
# MinIO Guide

This guide provides an overview of using MinIO with the `mc` (MinIO Client) tool and a sample Python script for 
interacting with MinIO.

## Using MinIO UI

BERDL provides MinIO UI access through the following URLs based on your environment:

**Development Environment:**
```
https://minio-ui.dev.berdl.kbase.us
```

**Staging Environment:**
```
https://minio-ui.stage.berdl.kbase.us
```

**Production Environment:**
```
https://minio-ui.berdl.kbase.us
```

Log in using your MinIO credentials (access key and secret key).

### Getting Your MinIO Credentials

**From within JupyterHub notebooks**, your MinIO credentials are automatically available as environment variables:

```python
import os

# Get your MinIO credentials
access_key, secret_key = os.environ['S3_ACCESS_KEY'], os.environ['S3_SECRET_KEY']

print(f"Access Key (username for MinIO UI): {access_key}")
# Note: Keep your secret key secure - don't share or print it in shared notebooks
```

These are the same credentials you use to:
- Log in to the MinIO UI (username = access key, password = secret key)
- Configure the MinIO Client (`mc`)
- Access MinIO via Python/boto3

## Prerequisites (For Local MinIO Client Access)

If you need to access MinIO from your local machine using the MinIO Client (`mc`) or MinIO UI, you'll need to set up SSH tunneling first.

### Setting Up SSH Tunnel

You need to create a secure tunnel from your local machine to the KBase server to access MinIO services.

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

Once your tunnel is configured, you can proceed with the MinIO Client setup below.

## Using MinIO Client (`mc`)

### Installation

Download and install the MinIO Client (`mc`) from the [MinIO official website](https://min.io/download?license=agpl&platform=macos).

### Configuration

**⚠️ IMPORTANT**: The proxy settings below will route ALL HTTP/HTTPS traffic through the SOCKS proxy. This may affect normal web access in the same terminal session.

**Recommendation**: Open a **dedicated terminal window** for MinIO operations, or revert the proxy settings when done (see cleanup instructions at the end of this section).

1. **Set up proxy environment variables** (required for accessing MinIO through the SOCKS proxy):

    ```bash
    unset ALL_PROXY
    
    export HTTP_PROXY="socks5://127.0.0.1:1338"
    export HTTPS_PROXY="socks5://127.0.0.1:1338"
    ```

2. Add MinIO server to the `mc` configuration based on your environment:

    **Development Environment:**
    ```bash
    mc alias set berdl-minio https://minio.dev.berdl.kbase.us
    ```

    **Staging Environment:**
    ```bash
    mc alias set berdl-minio https://minio.stage.berdl.kbase.us
    ```

    **Production Environment:**
    ```bash
    mc alias set berdl-minio https://minio.berdl.kbase.us
    ```

    It will prompt you to enter the access key and secret key. See the "Getting Your MinIO Credentials" section above for how to retrieve these values.

3. Verify the configuration and connection:

    ```bash
    mc ls berdl-minio
    ```

4. **Clean up proxy settings** (optional - do this when you're done with MinIO operations):

    ```bash
    # Unset proxy variables to restore normal web access
    unset HTTP_PROXY
    unset HTTPS_PROXY
    ```

    **Note**: These proxy settings only affect the current terminal session. Opening a new terminal will start fresh without these settings.

### Basic Commands

* List all buckets:

    ```bash
    mc ls berdl-minio
    ```

* Upload a file:

    ```bash
    mc cp <local_file_path> berdl-minio/<bucket_name>
    ```
* Upload a directory:

    ```bash
    mc cp --recursive <local_directory_path> berdl-minio/<bucket_name>
    ```

* Download a file:

    ```bash
    mc cp berdl-minio/<bucket_name>/<file_name> <local_file_path>
    ```

* Download a directory:

    ```bash
    mc cp --recursive berdl-minio/<bucket_name> <local_directory_path>
    ```
  
## Using MinIO with Python

The following Python script demonstrates how to interact with MinIO using the `boto3` library.

`boto3` is already installed in BERDL notebooks. To run the script on your own machine,
install it there first:

```bash
pip install boto3
```

### Sample Python Script

**From within JupyterHub** (credentials auto-injected):
```python
from pathlib import Path
import os
import boto3

# MinIO configuration - adjust endpoint based on your environment
# Development:   endpoint_url = 'https://minio-ui.dev.berdl.kbase.us'
# Staging:       endpoint_url = 'https://minio-ui.stage.berdl.kbase.us'
# Production:    endpoint_url = 'https://minio-ui.berdl.kbase.us'

endpoint_url = 'https://minio-ui.berdl.kbase.us'

# Get credentials from environment variables (automatically set in JupyterHub)
access_key = os.environ['S3_ACCESS_KEY']
secret_key = os.environ['S3_SECRET_KEY']

# Create S3 client
s3 = boto3.client('s3',
                  endpoint_url=endpoint_url,
                  aws_access_key_id=access_key,
                  aws_secret_access_key=secret_key)

def upload_to_s3(
        upload_file: Path,
        s3_key: str,
        s3: boto3.client,
        bucket: str):
    """
    Upload the specified file to the specified S3 bucket.

    :param upload_file: path of the file to upload
    :param s3_key: key of the file in the S3 bucket
    :param s3: boto3 client for S3
    :param bucket: name of the S3 bucket
    """
    try:
        # Skip uploading if the file already exists in the bucket
        s3.head_object(Bucket=bucket, Key=s3_key)
    except s3.exceptions.ClientError:
        s3.upload_file(str(upload_file), bucket, s3_key)
```

## Additional Resources
* [MinIO Client (mc) Official Guide](https://min.io/docs/minio/linux/reference/minio-mc.html?ref=docs)
* [MinIO Python SDK](https://docs.min.io/docs/python-client-quickstart-guide.html)
* [Boto3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/quickstart.html#using-boto3)
