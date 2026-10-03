# Shared Spark Compute: Design and Operations

How BERDL schedules Spark for notebook users from Phase 2 of the elastic compute plan onward: the cluster shape, why it is cut the way it is, what it means for a user, and how to grow or shrink it for a workshop or a demo.

This document describes the target design settled on 2026-10-03. It supersedes section 5 (Shared Spark Cluster) of [cpu_ram_capacity_analysis.md](cpu_ram_capacity_analysis.md) once the re-cut below is applied; the per-user cluster sections of that document stay valid for users who are not yet on the shared cluster.

---

## 1. Why the change

Today every notebook user gets a dedicated Spark cluster: a Medium profile is 3 worker pods of 3 cores / 22 GB each, a Large profile 10 worker pods of 6 cores / 43 GB, all held for the life of the notebook server. Measured over three days in prod (2026-09-30 to 2026-10-03, driver gauges written once a minute by the notebook watchdog):

| what | measured |
|---|---|
| dedicated clusters present | 25, holding 684 cores at 100 % allocation whether or not anything ran |
| time a dynamic-allocation server actually held executors | 16–34 % of minutes for 13 of 17 users |
| time it had work pending | 1–7 % of minutes |
| executors a burst needs | 1 for most users (p95), 3 for the heavier ones, 10 once (one Large user) |
| users holding executors at the same moment | p50 6, p95 11, max 13 |
| executors held fleet-wide at the same moment | p50 7, p95 21, max 25 |
| the existing shared cluster (320 cores) | 0 % used for the whole window |

Dynamic allocation (shipped in the notebook image on 2026-09-30) already lets a server acquire executors when a job starts and release them 60 s after it ends. On a dedicated cluster that saves nothing: the worker pods are still reserved. On a shared cluster it means the pool only has to hold what is in use at the same moment, which is a fifth to a tenth of what the dedicated clusters reserve.

## 2. The design

### 2.1 Users keep Medium and Large

There is no new profile. A user picks Medium or Large exactly as before. For users whose KBase role is listed in the hub's `SHARED_SPARK_ROLES` (staff first, everyone in Phase 3), the hub does not ask the Spark cluster manager for a dedicated cluster; it points the user's in-pod Spark Connect server at the shared master instead and sets `BERDL_SPARK_MODE=shared`. The profile's sizing variables (`SPARK_WORKER_COUNT`, `SPARK_WORKER_CORES`, `SPARK_WORKER_MEMORY`, `SPARK_MASTER_*`) are passed through unchanged, so the notebook writes the same executor size it writes today and `SPARK_WORKER_COUNT` becomes the executor cap. Taking a role out of the list sends its holders back to dedicated clusters at their next spawn.

### 2.2 Executor shapes are unchanged, and that is the point

| profile | executor | cap | driver |
|---|---|---|---|
| Medium | 3 cores / 18 GiB heap | 3 executors (9 cores) | 1 core / 4 GiB (was 2 GiB, which is the heap that runs out) |
| Large | 6 cores / 36 GiB heap | 10 executors (60 cores) | 2 cores / 8 GiB |

The 18 and 36 GiB heaps are what today's Medium and Large workers give after the cluster manager's 10 % and the notebook's 10 % overheads. Keeping them means no task that fits today runs out of memory on the shared cluster. Both shapes are about 6.7 GiB per core, which is also the ratio of the nodes themselves (1,133 GiB / 168 CPUs = 6.74), so they pack a node evenly.

### 2.3 Shared worker: 12 cores / 84 GiB, 13 per node

The shared cluster is a set of identical Spark standalone worker pods. Each is cut so that the two executor shapes pack it with nothing left over:

- **12 cores** is a multiple of both 3 and 6: a worker holds 4 Medium executors, 2 Large, or 2 Medium + 1 Large. The previous 8-core cut strands 2 cores under every Medium-shaped executor.
- **84 GiB container, `SPARK_WORKER_MEMORY=72g`**: 72 GiB is four 18 GiB heaps (the amount the Spark worker hands out); the remaining 12 GiB covers the executors' off-heap overhead (10 % of each heap) and the worker's own JVM, so a full worker stays under its limit. The previous cut set the Spark memory equal to the container limit, which is where executor kills at the limit came from.
- **13 workers per node** use 156 of 168 cores and 1,092 of 1,133 GiB and leave the rest for the node's own daemons and the monitoring agent, which the previous 20-per-node cut did not.

Two nodes (kworker-11 and kworker-18) give **26 workers, 312 cores, 2,184 GiB: 104 Medium executor slots, or 52 Large, or any mix.** One Large user at full burst takes 20 of the 104 slots; the measured fleet-wide peak was 21 executors. Each additional node adds 13 workers and 52 Medium slots (section 4).

### 2.4 Settings carried over from dynamic allocation

| setting | value | why |
|---|---|---|
| `spark.dynamicAllocation.executorIdleTimeout` | 60 s | a server releases its executors a minute after its last task; 30 s is the lever if slots get tight, since the idle tail is most of the measured hold time |
| `shuffleTracking.timeout` / `cachedExecutorIdleTimeout` | 10 min / 15 min | executors holding shuffle or cached data linger a little longer; neither limited anything in the measurement |
| Spark Connect session timeout | 10 h | unchanged; lowering it breaks open notebooks after a break |
| scheduler | FAIR | several queries from one user share the user's executors evenly |

Known and accepted: a server whose executors hold cached data stays allocated for as long as some client keeps running small jobs on it (KIND's periodic poll, or a user's own script). Those users hold their cap continuously and are counted at cap in capacity planning.

### 2.5 Node layout

| node(s) | role | notes |
|---|---|---|
| kworker-11, kworker-18 | shared Spark cluster | tainted `environments=prodsparkworker:NoSchedule`, so only the shared workers and the master land there |
| kworker-03 | all notebook pods | unchanged; the Spark Connect server, which is the driver, runs inside the notebook pod |
| kworker-05 to 10, 14 to 17 | dedicated per-user clusters | emptied progressively as roles move to the shared cluster; each sits at about a third of its requests today |
| kworker-01, 04, 13, 19, 20 | idle apart from a monitoring agent | the first candidates for growing the shared cluster |
| kworker-12 | Trino | coordinator plus 4 workers of 40 cores / 240 GiB |
| kworker-02 | platform services | MMS, MCP, Polaris, Hive metastore, event processor, PgBouncer, Postgres |

### 2.6 What a user notices

- No dedicated cluster is created at spawn, so the server is ready sooner.
- The first query after an idle period waits a few seconds for executors to come up; cached DataFrames and shuffle files are dropped after the idle timeouts and recomputed on the next use.
- Executor sizes and caps are the ones they had. When the pool is busy, executors arrive late and the job waits rather than fails; another user's heavy job can slow theirs.
- The shared master UI lists every user's application name; it is not exposed to users.

## 3. Reading the cluster

- **Master UI / JSON**: `sharedsparkclustermaster.prod:8090/json/` (port-forward the Service; see the Kubernetes testing runbook). `workers` should list 26 (13 per node) with 12 cores / 72 GiB each; `activeapps` carries one application per connected server named `Spark Connect server (<user>)` with its current cores; `WAITING` with 0 cores is the idle state of a dynamic-allocation server, not a fault.
- **`spark_cluster_usage`** (SCM reporter, VictoriaLogs): one record per minute per cluster, including `shared`, with cores allocated and `waiting_apps`. A rising count of WAITING applications *with pending executor requests* is the signal that the pool is full.
- **`spark_da_usage`** (notebook watchdog): per-server executors held and tasks pending, once a minute; the source of the duty-cycle numbers above.

## 4. Growing the cluster for a workshop or a demo

Plan for the peak: slots needed ≈ (users expected to run at the same time) × (executors per burst, 1–3 for Medium, up to 10 for Large) + pinned users at their cap. Each node adds 13 workers = 52 Medium slots or 26 Large.

Growing the cluster is a change to one Deployment, `sharedsparkclusterworker` in the `prod` namespace: add the node's hostname to the worker pod's node affinity and raise `replicas` by 13 per node. Nothing changes on the master, the new workers register themselves within a minute, and running applications pick them up for their next executor request. Before that:

1. **Pick a node with the room.** Each node needs 13 × 12 cores and 13 × 84 GiB free of requests. The idle nodes (kworker-01, 04, 13, 19, 20) have it today; a per-user pool node does not until its dedicated clusters are gone.
2. **Keep other workloads off it.** Remove the node from the Spark cluster manager's `MASTER_NODE_SELECTOR_VALUES` / `WORKER_NODE_SELECTOR_VALUES` (its ConfigMap) and restart the manager, so no new dedicated cluster lands there; if it is a pool node, let its existing clusters drain (users respawn) and check with `kubectl get pods -o wide`. Taint it like the other shared nodes (`environments=prodsparkworker:NoSchedule`) so nothing else is scheduled onto it; the worker pods already carry the matching toleration.
3. **Edit the Deployment** (through the Rancher kubectl described in the Kubernetes testing runbook):

   ```bash
   # add the node to the affinity list and raise the replica count by 13 per node added
   kubectl -n prod patch deploy sharedsparkclusterworker --type=json -p '[
     {"op":"add","path":"/spec/template/spec/affinity/nodeAffinity/requiredDuringSchedulingIgnoredDuringExecution/nodeSelectorTerms/0/matchExpressions/0/values/-","value":"kworker-19"}
   ]'
   kubectl -n prod scale deploy sharedsparkclusterworker --replicas=39   # 26 + 13
   ```

   The scheduler places the 13 new pods on the only node in the list with room. Confirm the placement with `kubectl -n prod get pods -l app.kubernetes.io/name=sharedsparkclusterworker -o wide` and the registration on the master JSON (`workers` count, all `ALIVE`).
4. **Mirror the change into the manifests.** The Deployment is declared in the GitLab `manifests` repo (`definitions/sharedsparkcluster/overlays/prod`). A direct edit works, but the next `kubectl apply -k` of that overlay reverts it unless the overlay carries the same hostnames and replica count. Edit the overlay in the same change window.
5. **During the event**, watch the master JSON for WAITING applications with pending requests and the `spark_cluster_usage` records; if they appear, add another node the same way.

### 4.1 Shrinking afterwards

Do it when the cluster is quiet (no active applications with cores on the master): executors on a removed worker are killed and their tasks retried elsewhere, which a user sees as a slower query, not a failure, but a long job will repeat work.

1. Lower `replicas` by 13 per node you are removing. Kubernetes picks which pods to remove, so some may remain on the node you want to free.
2. Remove the hostname from the affinity list with the matching `remove` patch. Pods still on that node are now outside the allowed set only for *new* scheduling; delete them (`kubectl -n prod delete pod <name>`) and the Deployment recreates them on the remaining nodes if `replicas` still calls for them.
3. Confirm the worker count on the master and mirror the overlay.

### 4.2 Converting a per-user pool node permanently (Phase 3)

Same as growing, with the drain in step 2 done deliberately: remove the node from the Spark cluster manager's selector lists, wait for its dedicated clusters to disappear as users respawn (coordinate a Stop/Start for stragglers), then taint it and add it to the shared cluster. Node pairs move one at a time, with a week of WAITING-app observation before the next.

## 5. Related

- [cpu_ram_capacity_analysis.md](cpu_ram_capacity_analysis.md): node inventory and the per-user cluster capacity model; its shared-cluster section describes the pre-re-cut 40 × 8-core layout.
- Notebook image: [spark_notebook](https://github.com/KBaseDataLakehouse/spark_notebook) (Connect server, dynamic allocation, watchdog); hub: [BERDL_JupyterHub](https://github.com/KBaseDataLakehouse/BERDL_JupyterHub) (profiles, role gates); cluster manager: [spark_cluster_manager](https://github.com/KBaseDataLakehouse/spark_cluster_manager) (dedicated clusters, usage reporter); worker image: [kube_spark_manager_image](https://github.com/KBaseDataLakehouse/kube_spark_manager_image).
