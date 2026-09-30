# Spark Notebook Base

> The foundational Docker image containing PySpark, Java dependencies, and Python packages for all notebook-based services.

| | |
|---|---|
| **Docker Image** | `ghcr.io/kbasedatalakehouse/spark_notebook_base:main` |
| **GitHub Repo** | [spark_notebook_base](https://github.com/KBaseDataLakehouse/spark_notebook_base) |
| **Python** | 3.13 |
| **Package Manager** | uv |

## Overview

This image provides the core runtime environment for `spark_notebook` and dynamic Spark cluster components. It bundles:
- **PySpark 4.1.3** with Spark Connect (based on `quay.io/jupyter/pyspark-notebook`).
- **Apache Iceberg 1.11.0** — Iceberg Spark runtime for reading and writing tables in the Apache Polaris REST catalogs.
- **Apache Sedona 1.9.1** — Geospatial SQL functions and types for Spark.
- **PyIceberg 0.12.0** — Apache Iceberg table support with S3.
- **Trino 0.340** — Trino Python client for direct query access.
- **Java Dependencies**: JARs downloaded via Gradle (Iceberg Spark runtime, Sedona, `hadoop-aws` and the AWS SDK bundle) plus the custom KBase auth and namespace-validation Spark Connect interceptors.
- **Python Dependencies**: Managed via `uv` and `pyproject.toml`.
- **System Utilities**: `mc` (MinIO Client), `redis-tools`, `gettext`.

## Key Features

- **Centralized Dependencies**: All client libraries (MCP, MinIO Manager, Spark Manager, Task Service) and JupyterLab extensions (CoreUI, Task Browser, Tenant Data Browser, Access Request) are pre-installed here.
- **Multi-stage Build**: Uses Gradle to download JARs in a builder stage, keeping the final image lean.
- **Jupyter AI**: Pre-installed with CBorg, OpenAI, Anthropic, and Ollama providers.
- **Data Science Stack**: pandas, scikit-learn, scikit-bio, scipy, biopython, matplotlib, seaborn, scanpy.
- **`uv` Package Management**: Fast, reliable Python dependency resolution. The Python dependencies are bundled in `pyproject.toml` as the package `berdl-notebook-python-base`. Other repositories can consume these common dependencies by installing `git+https://github.com/KBaseDataLakehouse/spark_notebook_base.git`.

## Build Process

1.  **Builder Stage**: Uses `gradle:9.1.0-jdk24-ubi-minimal` to download Java dependencies.
2.  **Runtime Stage**: Copies JARs to `/usr/local/spark/jars/` and installs Python packages.

## Usage by Other Services

This image is the `FROM` base for:
- `spark_notebook`
- `kube_spark_manager_image` (Dynamic Spark Cluster pods)
