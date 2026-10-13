# KBERDL CPU/RAM Capacity Analysis

> **Updated 2026-10-13.** Notebook Spark runs on the shared cluster for every user since 2026-10-11 (section 5, designed in [shared_spark_compute_design.md](shared_spark_compute_design.md)). Sections 1.3, 3 and 4 describe the per-user cluster model, which remains only as the rollback path; the dedicated pool nodes are being retired into the shared cluster.

---

## 1. Current State

### 1.1 Cluster Node Inventory

| Node | Cores | RAM | Disk | Current Assignment |
|------|------:|----:|-----:|-------------------|
| kworker-01 | 168 | 1024 GB | 100 TB | Dev/Staging Services |
| kworker-02 | 168 | 1024 GB | 100 TB | Prod Services |
| kworker-03 | 168 | 1024 GB | 100 TB | All user notebook pods |
| kworker-05 – 10 | 168 each | 1024 GB each | 100 TB each | Per-user Spark clusters (retiring: no new cluster since 2026-10-11; nodes move to the shared cluster in pairs) |
| kworker-11 | 168 | 1024 GB | 100 TB | Shared Spark cluster (master + 13 workers) |
| kworker-12 | 168 | 1024 GB | 100 TB | Trino cluster (coordinator + 4 workers) |
| kworker-13 | 168 | 1024 GB | 100 TB | Other workloads |
| kworker-14 – 17 | 168 each | 1024 GB each | 100 TB each | Per-user Spark clusters (retiring, as above) |
| kworker-18 | 168 | 1024 GB | 100 TB | Shared Spark cluster (13 workers) |
| kworker-19 | 168 | 1024 GB | 100 TB | Other workloads |
| kworker-20 | 168 | 1024 GB | 100 TB | Other workloads |

**Dedicated Spark pool: 10 nodes** (kworker-05–10, kworker-14–17) — **1,680 cores, 10,240 GB RAM total**, being retired: four clusters started before 2026-10-11 remain, and the nodes join the shared cluster a pair at a time once they are empty.

### 1.2 User Workload Components

Each user session spawns **three components** across two node groups (managed by [BERDL_JupyterHub](https://github.com/KBaseDataLakehouse/BERDL_JupyterHub)):

```
┌──────────────────────────────────────────────────────────────┐
│  kworker-03 (168 cores, 1024 GB RAM, 100 TB disk)            │
│                                                              │
│  ┌─────────────────┐                                         │
│  │  Notebook Pod   │  (All users scheduled here)             │
│  │  (JupyterLab +  │                                         │
│  │  Spark Connect  │                                         │
│  │  server inside) │                                         │
│  └─────────────────┘                                         │
│                                                              │
│  User home dirs stored on local disk via hostPath            │
│  Path: /mnt/state/prod/hub_homes/{username}                  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  kworker-05–10, 14–17 (per-user Spark clusters)              │
│                                                              │
│  ┌────────────────┐  ┌──────────────┐                        │
│  │  Spark Master  │  │ Spark Workers│  (per-user,            │
│  │  (per-user)    │  │  5 or 10     │   dynamically created) │
│  └────────────────┘  │  replicas)   │                        │
│                      └──────────────┘                        │
└──────────────────────────────────────────────────────────────┘
```

> Since 2026-10-11 the per-user master/workers box applies only to the clusters started before that date. A notebook's Spark Connect server, which runs inside the pod on kworker-03, submits to the shared cluster (section 5) and takes executors from it on demand.

### 1.3 Profile Definitions

Two profiles are available to users. Both profiles use **identical notebook pods** — the differentiation is entirely in the Spark cluster size.

#### Notebook Pod (both profiles) — [spark_notebook](https://github.com/KBaseDataLakehouse/spark_notebook)

| Component | CPU (guarantee) | Memory (limit) | Memory (guarantee) |
|-----------|----------------:|---------------:|-------------------:|
| Notebook pod | 1 core | 24 GB | 6 GB |

> No `cpu_limit` is set — pods can burst to use available CPU on the node.

On the shared cluster a profile is an executor shape and a cap, not a set of worker pods: Medium = up to 3 executors of 3 cores / 18 GiB heap with a 1 core / 4 GiB driver, Large = up to 10 executors of 6 cores / 36 GiB heap with a 2 core / 8 GiB driver. The driver is the Spark Connect server in the notebook pod; executors are taken from the pool when a job runs and released 60 s after it ends. The worker-pod shapes below are what a dedicated cluster of each profile was, kept for the rollback path.

#### Medium Spark Cluster

| Component | Cores | Memory |
|-----------|------:|-------:|
| Spark Master (×1) | 1 | 2 GiB |
| Spark Workers (×3) | 3 each (9 total) | 20 GiB each (60 GiB total) |
| **Per-user Spark total** | **10 cores** | **62 GiB** |

> 3 cores/worker is the sweet spot for Spark executors (balances parallelism vs GC pressure). 20 GiB/worker provides 6.7 GB/core — generous for data processing. Sized so all 160 notebook users can run a Medium cluster concurrently within the 10-node pool.

#### Large Spark Cluster

| Component | Cores | Memory |
|-----------|------:|-------:|
| Spark Master (×1) | 2 | 8 GiB |
| Spark Workers (×10) | 6 each (60 total) | 40 GiB each (400 GiB total) |
| **Per-user Spark total** | **62 cores** | **408 GiB** |

> Large is 4× the memory and ~4× the cores of Medium, designed for power-user workloads.

---

## 2. Notebook Pod Capacity (kworker-03)

### 2.1 Available Resources

| Resource | Total | System/OS Overhead (~5%) | Available for Workloads |
|----------|------:|-------------------------:|------------------------:|
| CPU | 168 cores | ~8 cores | **~160 cores** |
| RAM | 1024 GB | ~50 GB | **~974 GB** |

### 2.2 Maximum Concurrent Users

**Per-user notebook pod:** 1 core (guarantee), 24 GB RAM (limit), 6 GB RAM (guarantee), no CPU limit

> [!NOTE]
> Spark Master and Workers are scheduled on kworker-05–10 and kworker-14–17, **not** kworker-03. The capacity analysis below only considers notebook pod resources on kworker-03.

| Bottleneck | Available | Per User | Max Users |
|-----------|----------:|---------:|----------:|
| CPU (by guarantee) | 160 cores | 1 core | **160 users** |
| RAM (by guarantee) | 974 GB | 6 GB | **162 users** |
| RAM (by limit) | 974 GB | 24 GB | 40 users |

> **Result: kworker-03 can schedule up to ~160 concurrent notebook pods** (CPU guarantee is the binding constraint).
>
> RAM overcommit ratio is 4× (24 GB limit / 6 GB guarantee). At 100 users, total guaranteed RAM = 600 GB (62% of 974 GB). Total limit = 2,400 GB — typical JupyterLab + Spark Connect client pods rarely approach their memory limit, so this overcommit is acceptable.

---

## 3. Per-User Spark Cluster Capacity (kworker-05–10, 14–17) — rollback path only

> This section sizes the dedicated-cluster model that was the default until 2026-10-11. It applies again only if the hub's shared-mode gate is rolled back. The live capacity model is section 5.

Per-user Spark clusters (managed by [spark_cluster_manager](https://github.com/KBaseDataLakehouse/spark_cluster_manager)) ran on **10 dedicated nodes** (kworker-05 through kworker-10, kworker-14 through kworker-17). Each node has 168 cores and 1024 GB RAM.

**Total Spark pool: 1,680 cores, 10,240 GB RAM**

### 3.1 Medium Spark Cluster

| Component | Cores | Memory |
|-----------|------:|-------:|
| Spark Master (×1) | 1 | 2 GiB |
| Spark Workers (×3) | 3 each (9 total) | 20 GiB each (60 GiB total) |
| **Per-user total** | **10 cores** | **62 GiB** |

| Bottleneck | Total Pool | Per User | Max Users |
|-----------|----------:|---------:|----------:|
| CPU | 1,680 cores | 10 cores | **168 users** |
| RAM | 10,240 GB | 62 GB | **165 users** |

> **Result: 10 Spark nodes can serve up to ~165 concurrent Medium Spark clusters** (RAM is the bottleneck) — enough to cover all 160 notebook users.
>
> At 160 users: 9,920 GiB (96.9% RAM), 1,600 cores (95.2% CPU) — both resources near full utilization.

### 3.2 Large Spark Cluster

| Component | Cores | Memory |
|-----------|------:|-------:|
| Spark Master (×1) | 2 | 8 GiB |
| Spark Workers (×10) | 6 each (60 total) | 40 GiB each (400 GiB total) |
| **Per-user total** | **62 cores** | **408 GiB** |

| Bottleneck | Total Pool | Per User | Max Users |
|-----------|----------:|---------:|----------:|
| CPU | 1,680 cores | 62 cores | **27 users** |
| RAM | 10,240 GB | 408 GB | **25 users** |

> **Result: 10 Spark nodes can serve up to ~25 concurrent Large Spark clusters** (RAM is the bottleneck).

---

## 4. Combined Capacity Summary

| Profile | Notebook Capacity (kworker-03) | Spark Cluster Capacity (10 nodes) | Effective Limit |
|---------|-------------------------------:|----------------------------------:|----------------:|
| Medium | 160 users | 165 users | **160 users** (notebook-limited) |
| Large | 160 users | 25 users | **25 users** (Spark-limited) |

> Medium was sized so the dedicated pool covered all 160 notebook users; kworker-03 (notebook pods) is the binding constraint and is at its ceiling (160/160 cores, 960/974 GB guaranteed — no headroom).
>
> **With the shared cluster (every user since 2026-10-11)** the Spark side is no longer a per-user reservation: the pool holds what is in use at the same moment. Two nodes give 104 Medium-sized executor slots; measured over the first two days with 19 users on the pool, the p95 of executors held at once was 12 and the peak 18, so the notebook node, not Spark, remains the limit on concurrent users (section 5).

---

## 5. Shared Spark Cluster (kworker-11 + kworker-18)

The shared Spark standalone cluster carries every notebook user's Spark since 2026-10-11, plus the data API's standalone applications and the event processor. Design and operations: [shared_spark_compute_design.md](shared_spark_compute_design.md). Re-cut to the shape below on 2026-10-03.

### 5.1 Cluster Configuration

| Component | Cores | Memory | Replicas | Node |
|-----------|------:|-------:|---------:|------|
| Spark Master | 2 | 8 GiB (4 GiB daemon heap) | 1 | kworker-11 |
| Spark Workers | 12 each | 84 GiB each (`SPARK_WORKER_MEMORY=72g`) | 26 (13 per node) | kworker-11 + kworker-18 |
| **Total** | **314** | **2,192 GiB** | | **312 worker cores, 1,872 GiB of executor memory** |

> The master keeps `FILESYSTEM` recovery on a host path, so a restart re-adopts running applications, and `spark.deploy.defaultCores=8` bounds a session that sets no core limit. Workers are placed by node affinity on the two tainted nodes; 13 per node uses 156 of 168 cores and leaves memory for the node's own daemons.
>
> 12 cores / 72 GiB per worker is a multiple of both executor shapes: a worker holds 4 Medium executors (3 cores / 18 GiB), 2 Large (6 cores / 36 GiB) or 2 Medium + 1 Large, with nothing stranded. Local disk I/O (shuffle, spill, cached blocks) is encrypted per application (`spark.io.encryption.enabled`, since 2026-10-11) because every application's executors share the worker's uid.

### 5.2 Resource Utilization (requests on the two nodes)

| Resource | Allocatable (2 nodes) | Requested | Utilization |
|----------|----------------------:|----------:|------------:|
| CPU | 336 cores | 314 cores | **93.5%** |
| RAM | 2,266 GiB | 2,192 GiB | **96.7%** |

### 5.3 Capacity Model (what the pool serves)

| | Medium units (3 cores / 18 GiB) | Large units (6 cores / 36 GiB) |
|---|---:|---:|
| slots on two nodes | 104 | 52 |
| one user at full burst | 3 (Medium cap) | 10 (Large cap, 20 Medium units) |
| each added node (13 workers) | +52 | +26 |

Demand is measured, not reserved. Over the first two days with every user on the pool (19 notebooks at once at most): peak 84 of 312 cores in use, p95 of 12 executors held at once across all users, the worst user short of an executor for 18 minutes in total, first executor after idle in 2.7 s at the median and 2.8 s at the 95th percentile, 3,497 executor endings all normal. Growing the pool is one Deployment edit per node (design doc, section 4); the five idle nodes (kworker-01, 04, 13, 19, 20) and, once empty, the ten retiring pool nodes are the candidates.

### 5.4 Data API Executor Configuration

The [datalake-mcp-server](https://github.com/KBaseDataLakehouse/datalake-mcp-server) (data API) answers through Trino by default since 2026-10-01. Its Spark path uses the caller's own notebook Connect server when one is running, and otherwise a standalone application on the shared cluster:

| Setting | Value | Description |
|---------|-------|-------------|
| `SPARK_WORKER_CORES` | 2 | Cores per executor |
| `SPARK_WORKER_COUNT` | 2 | Executors per job |
| `SPARK_WORKER_MEMORY` | 16 GiB | Memory per executor |
| `STANDALONE_SPARK_POOL_SIZE` | 8 | Concurrent standalone sessions per replica |
| **Per-job total** | **4 cores, 32 GiB** | 2 executors × (2c, 16G) |

**Executor packing per worker (12c, 72G):** 6 by CPU, 4 by memory → **4 executors per worker**, 26 workers × 4 = **104 executor slots**, or **52 concurrent standalone jobs** against 8 replicas × pool size 8 = 64 sessions. In practice the standalone path serves a handful of calls a day; the notebook applications set the pool's load.

---

## 6. Trino Query Engine (kworker-12)

A shared Trino cluster on **kworker-12** provides interactive SQL query capabilities for all users. Unlike per-user Spark clusters, Trino uses a single shared cluster with fair scheduling via resource groups.

### 6.1 Cluster Configuration

| Component | Cores | Memory | Heap (-Xmx) | Replicas | Purpose |
|-----------|------:|-------:|------------:|---------:|---------|
| Coordinator | 4 | 32 GiB | 24G | 1 | Query planning, scheduling, Web UI |
| Workers | 40 each | 240 GiB each | 192G | 4 | Query execution |
| **Total** | **164** | **992 GiB** | | **5 pods** | |

> The coordinator does not execute queries (`node-scheduler.include-coordinator=false`). All query execution runs on the 4 dedicated workers.

### 6.2 Resource Utilization

| Resource | Total (kworker-12) | Used | Utilization |
|----------|-------------------:|-----:|------------:|
| CPU | 168 cores | 164 cores | **97.6%** |
| RAM | 1024 GB | 992 GiB | **96.9%** |

### 6.3 Memory Limits

| Setting | Value | Description |
|---------|-------|-------------|
| `query.max-memory` | 100 GB | Max memory a single query can use across all workers |

Trino automatically manages per-node memory allocation. The query memory pool per worker is ~80% of heap = ~154 GB.

### 6.4 Concurrency (Resource Groups)

| Setting | Value | Description |
|---------|-------|-------------|
| Global concurrent queries | 50 | Max queries running across all users |
| Global queued | 500 | Max queries waiting in queue |
| Per-user concurrent queries | 10 | Max running queries per user |
| Per-user queued | 50 | Max queued queries per user |
| Per-user soft memory limit | 20% | One user can use at most 20% of cluster memory |
| Scheduling policy | weighted_fair | Active users share capacity equally |

**Concurrent query capacity estimate:**
- At 40 GB per query max: 960 GB pool / 40 GB = 24 large queries concurrently
- At 10 GB per query avg: 960 GB pool / 10 GB = 96 typical queries concurrently
- CPU-bound: 160 worker cores / 4 cores per query avg = 40 queries concurrently

> **Result: kworker-12 can handle ~40–96 concurrent queries** depending on query complexity. The resource group limits (50 global, 10 per-user) keep this within safe bounds.
