# Spark Notebook

> The user's personal JupyterLab workspace, pre-configured with Spark and data lake access.

| | |
|---|---|
| **Docker Image** | `ghcr.io/kbasedatalakehouse/spark_notebook:main` |
| **GitHub Repo** | [spark_notebook](https://github.com/KBaseDataLakehouse/spark_notebook) |
| **Base Image** | [spark_notebook_base](./spark_notebook_base.md) |
| **Python** | 3.13 (inherited from base) |

## Overview

The Spark Notebook is the primary user interface for the BERDL platform. It provides a JupyterLab environment pre-configured with Spark, Python, and the necessary libraries to interact with the data lake.

## Key Features

- **JupyterLab**: Full-featured notebook environment.
- **Pre-configured Spark**: Ready-to-use Spark session with credentials auto-injected based on KBase/MinIO manager.
- **AI Integration**: Includes `jupyter-ai` with CBorg, OpenAI, Anthropic, and Ollama providers, plus MCP tool discovery via `langchain-mcp-tools`.
- **Tenant Access UI**: Built-in [Access Request Extension](./berdl-access-request-extension.md) for requesting data access.
- **Trino Integration**: Direct Trino query support via `trino` Python client.
- **Apache Iceberg**: Spark reads and writes Iceberg tables through the Apache Polaris REST catalogs (`my` for personal tables, `<tenant>` for tenant tables); PyIceberg 0.12.0 is included for direct Iceberg table operations.
- **Persistent Shell Customization**: Users can configure their shell environments persistently via `~/.custom_profile`.

## Lifecycle

1. **Spawning**: When a user logs into JupyterHub, a dedicated container is spawned via KubeSpawner.
2. **Cluster Creation**: JupyterHub triggers the Spark Cluster Manager to create a dynamic Spark cluster for the user.
3. **Credential Injection**: JupyterHub calls the MinIO Manager Service to initialize user policies and inject S3 credentials.
4. **Initialization**:
    - `entrypoint.sh` sets up user permissions and environment.
    - `setup_spark_session.py` (via `notebook_utils`) initializes Spark with the injected credentials.
5. **Ready**: User can now run Spark jobs against their personal dynamic cluster with access to their data in MinIO.

## Architecture

```mermaid
graph TD
    JH[JupyterHub] -->|Spawns| NB[Spark Notebook Container]
    NB -->|Uses Base| BASE[Spark Notebook Base]
    
    NB -->|Spark Connect| MCP[Datalake MCP Server]
    NB -->|S3| MIN[MinIO]
    NB -->|Iceberg REST| POL[Apache Polaris]
```

## Internal Components

- **`notebook_utils`**: Python package for Spark session management.
- **`configs/`**: Spark defaults, Jupyter extensions, and hooks.
