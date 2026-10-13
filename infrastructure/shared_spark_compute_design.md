# Shared Spark Compute: Design and Operations

How BERDL schedules Spark for notebook users on the shared cluster: the cluster shape, why it is cut the way it is, what it means for a user, and how to grow or shrink it for a workshop or a demo.

This document describes the cluster as deployed: the worker re-cut of 2026-10-03, a staff cohort from 2026-10-05, and every notebook user since 2026-10-11 15:33 UTC. The capacity figures live in section 3 of [cpu_ram_capacity_analysis.md](cpu_ram_capacity_analysis.md); that document keeps the per-user cluster model (its section 6) only because it is the rollback path.

Updated 2026-10-13.

---

## 1. Why the change

Until 2026-10-11 every notebook user got a dedicated Spark cluster: a Medium profile was 3 worker pods of 3 cores / 22 GB each, a Large profile 10 worker pods of 6 cores / 43 GB, all held for the life of the notebook server. Measured over three days in prod before the change (driver gauges written once a minute by the notebook watchdog):

| what | measured |
|---|---|
| dedicated clusters present | 25, holding 684 cores at 100 % allocation whether or not anything ran |
| time a dynamic-allocation server actually held executors | 16–34 % of minutes for 13 of 17 users |
| time it had work pending | 1–7 % of minutes |
| executors a burst needs | 1 for most users (p95), 3 for the heavier ones, 10 once (one Large user) |
| users holding executors at the same moment | p50 6, p95 11, max 13 |
| executors held fleet-wide at the same moment | p50 7, p95 21, max 25 |
| the existing shared cluster (320 cores) | 0 % used for the whole window |

Dynamic allocation in the notebook image already lets a server acquire executors when a job starts and release them 60 s after it ends. On a dedicated cluster that saves nothing: the worker pods are still reserved. On a shared cluster it means the pool only has to hold what is in use at the same moment, which is a fifth to a tenth of what the dedicated clusters reserve.

**What the first days of the shared cluster showed** (prod, every user, 2026-10-11 to 2026-10-13; the staff cohort before that from 2026-10-05):

| what | measured |
|---|---|
| notebooks on the shared cluster | 19 of 23 running; every new server since the change |
| peak of the pool | 84 of 312 cores, 19 applications at once |
| executors held at once (all users) | p50 6, p95 12, max 18 |
| time a user wanted more executors than it held | at most 18 minutes in two days, always one executor short |
| wait for the first executor after idle | p50 2.7 s, p95 2.8 s, worst 8.9 s (5,299 cold starts over six days) |
| executor endings on the shared workers | 3,497 normal releases, none crashed, none hit the 84 GiB worker limit |
| master and workers | 0 restarts since the re-cut |

## 2. The design

### 2.1 Users keep Medium and Large

There is no new profile. A user picks Medium or Large exactly as before. The hub's `DYNAMIC_ALLOCATION_ROLES` names `BERDL_USER`, every notebook user, since 2026-10-11 (a staff cohort from 2026-10-05), and its `SHARED_SPARK_MASTER_URL` points at the shared master. With both set the hub does not ask the Spark cluster manager for a dedicated cluster: it points the user's in-pod Spark Connect server at the shared master, passes the profile's executor shape and cap unchanged, and the server takes executors from the pool on demand. Nobody gets a dedicated cluster any more (decision 2026-10-11); the cluster-manager path remains only as the rollback: narrow the roles or unset the URL, and the next spawn builds a dedicated cluster again. A server that started before the change keeps its dedicated cluster until the user restarts it.

In shared mode the notebook helpers refuse a classic (non-Connect) session and the create-my-cluster helpers; `get_spark_session()` is the way in. The MCP server's standalone applications and the event processor submit their own applications to the same master.

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
| `spark.io.encryption.enabled` | true (since 2026-10-11) | every application's executors run under one uid on a shared worker, so one user's code could read another's shuffle, spill and cached-block files; with this on they are ciphertext under a per-application key held only in the driver's and executors' memory. Event logs, the UI and executor logs are unchanged; no measurable cost on a 40-million-row shuffle |
| `spark.authenticate` | off (decision 2026-10-03) | on a standalone cluster it is one secret held by every user's driver and executors, so it separates nobody |
| master port 7077 | open to the cluster (decision 2026-10-11) | anyone who can reach the master may start a Spark session; the per-application cap and `spark.deploy.defaultCores=8` (the bound on sessions that set no limit) are the only limits |
| master | 2 CPU / 8 GiB, 4 GiB daemon heap, `FILESYSTEM` recovery on a host path | a master restart re-adopts running applications; tested on prod on 2026-10-03 |

Known and accepted: a server whose executors hold cached data stays allocated for as long as some client keeps running small jobs on it (KIND's periodic poll, or a user's own script). Those users hold their cap continuously and are counted at cap in capacity planning.

### 2.5 Node layout

| node(s) | role | notes |
|---|---|---|
| kworker-11, kworker-18 | shared Spark cluster | tainted `environments=prodsparkworker:NoSchedule`, so only the shared workers and the master land there |
| kworker-03 | all notebook pods | unchanged; the Spark Connect server, which is the driver, runs inside the notebook pod |
| kworker-05 to 10, 14 to 17 | the retiring dedicated pool | no new dedicated cluster has been created since 2026-10-11; four clusters started before the change remain until their users restart, then the nodes move to the shared cluster a pair at a time (section 4.2) |
| kworker-01, 04, 13, 19, 20 | idle apart from a monitoring agent | the first candidates for growing the shared cluster |
| kworker-12 | Trino | coordinator plus 4 workers of 40 cores / 240 GiB |
| kworker-02 | platform services | MMS, MCP, Polaris, Hive metastore, event processor, PgBouncer, Postgres |

### 2.6 What a user notices

- No dedicated cluster is created at spawn, so the server is ready sooner.
- The first query after an idle period waits for executors to come up: 2.7 s at the median and 2.8 s at the 95th percentile measured over 5,299 such starts; cached DataFrames and shuffle files are dropped after the idle timeouts and recomputed on the next use.
- Classic (non-Connect) sessions and the create-my-cluster helpers are refused; `get_spark_session()` is the entry point.
- Executor sizes and caps are the ones they had. When the pool is busy, executors arrive late and the job waits rather than fails; another user's heavy job can slow theirs.
- The shared master UI lists every user's application name; it is not exposed to users.

## 3. Reading the cluster

- **Master UI / JSON**: `sharedsparkclustermaster.prod:8090/json/`, readable without a port-forward through the API server proxy (`kubectl get --raw /api/v1/namespaces/prod/services/sharedsparkclustermaster:8090/proxy/json/`; see the Kubernetes testing runbook). `workers` should list 26 (13 per node) with 12 cores / 72 GiB each; `activeapps` carries one application per connected server named `Spark Connect server (<user>)` with its current cores; `WAITING` with 0 cores is the idle state of a dynamic-allocation server, not a fault.
- **`spark_cluster_usage`** (SCM reporter, VictoriaLogs): one record per minute per cluster, including `shared`, with cores allocated and `waiting_apps`. A rising count of WAITING applications *with pending executor requests* is the signal that the pool is full.
- **`spark_da_usage`** (notebook watchdog): per-server executors held, wanted and needed, once a minute; the source of the duty-cycle numbers above. A minute in which a server wants more executors than it holds is the queue signal; an idle server registered as `WAITING` with 0 cores is not.
- **The prod status page** lists who is on the shared cluster and what each holds ("Resources by user"), and the per-user sweep guide has the cluster's checks (master gauges, workers, disk, lifecycle logs, waiting and headroom).

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
4. **Record the change for the manifests.** The Deployment is declared in the GitLab `manifests` repo (`definitions/sharedsparkcluster/overlays/prod`), which is behind the live cluster and is being rewritten. Until that rewrite lands, do not apply the overlay (it would revert the live Deployment); note the new hostnames and replica count for the rewrite.
5. **During the event**, watch the master JSON for WAITING applications with pending requests and the `spark_cluster_usage` records; if they appear, add another node the same way.

### 4.1 Shrinking afterwards

Do it when the cluster is quiet (no active applications with cores on the master): executors on a removed worker are killed and their tasks retried elsewhere, which a user sees as a slower query, not a failure, but a long job will repeat work.

1. Lower `replicas` by 13 per node you are removing. Kubernetes picks which pods to remove, so some may remain on the node you want to free.
2. Remove the hostname from the affinity list with the matching `remove` patch. Pods still on that node are now outside the allowed set only for *new* scheduling; delete them (`kubectl -n prod delete pod <name>`) and the Deployment recreates them on the remaining nodes if `replicas` still calls for them.
3. Confirm the worker count on the master and mirror the overlay.

### 4.2 Converting a per-user pool node permanently

Same as growing, with the drain in step 2 done deliberately: remove the node from the Spark cluster manager's selector lists, wait for its dedicated clusters to disappear as users respawn (coordinate a Stop/Start for stragglers), then taint it and add it to the shared cluster. Since 2026-10-11 no new dedicated cluster is created, so the drain is only the clusters started before that date. Node pairs move one at a time, with a week of observation of the queue signal before the next.

## 5. Adding a profile (extra small, extra large)

A profile is three numbers for the shared cluster: executor cores, executor memory, and the cap on executors, plus the driver size. The hub passes them as `SPARK_WORKER_CORES`, `SPARK_WORKER_MEMORY` (the pod-equivalent figure; the heap is that minus the two 10 % overheads) and `SPARK_WORKER_COUNT`; the notebook needs no change for a new profile, and the same three numbers size real worker pods if the profile is also offered as a dedicated cluster. The rules that keep a new profile from breaking the pool:

1. **Keep the memory-per-core ratio.** The worker is cut at 6 GiB of executor heap per core (7 GiB of container per core). A new shape must use the same ratio: 1 core / 6 GiB, 2 / 12, 3 / 18 (Medium), 4 / 24, 6 / 36 (Large), 12 / 72. Any of these packs a 12-core / 84 GiB worker with nothing stranded, in any mix. A shape with a different ratio, for example 2 cores / 36 GiB for a task that needs a very large heap, strands either cores or memory on every worker it lands on; offer such a shape only as a dedicated cluster, not on the shared pool.
2. **Executor cores must divide 12.** 1, 2, 3, 4, 6 or 12. A 5-core or 8-core executor leaves cores idle on every worker.
3. **Memory per task does not change with the shape.** Spark runs one task per executor core and the tasks share the executor heap, so every shape above gives a task about 6 GiB. A bigger executor buys more parallelism per executor, not a bigger task. Users who need a single task to have more memory than that run fewer tasks per executor (`spark.task.cpus`) or need a dedicated cluster with a different ratio; "extra large" on the shared pool should mean a higher cap, not a fatter executor.
4. **Prefer raising the cap over inventing a shape.** An extra-large profile is the Large executor (6 cores / 36 GiB) with a cap of 15 or 20 instead of 10; an extra-small one is 1 core / 6 GiB or 2 cores / 12 GiB with a cap of 2 or 3. Fewer shapes keep the master's packing simple and the capacity arithmetic in one unit.
5. **Bound the cap by the pool.** Cap × executor cores is the most one user can take; keep it under about a quarter of the pool (78 of 312 cores on two nodes, so a Large cap of 13, or 20 once a third node is added) so one user at full burst cannot starve everyone else. Dynamic allocation makes the cap a ceiling, not a reservation, so a generous cap costs nothing while the user is idle.
6. **Size the driver with the cap.** The driver, the Spark Connect server in the notebook pod, holds the plan, the task bookkeeping and anything the user collects: 1 core / 4 GiB serves a cap of 3, 2 cores / 8 GiB a cap of 10, 4 cores / 16 GiB beyond that. The notebook pod's own limit (48 GiB since 2026-10-05, with a CPU limit of 2 cores for Medium and 8 for Large) bounds it.
7. **Check the slot budget before offering it.** Expected concurrent users of the profile × their cap, added to the existing demand, against the pool's slots (104 Medium-sized per two nodes, 52 per added node), using the measured concurrency rather than the user count. If it does not fit at the p95, add a node (section 4) before the profile goes live.

| profile | executor | cap | total at full burst | driver | notes |
|---|---|---|---|---|---|
| extra small | 1 core / 6 GiB | 2 | 2 cores / 12 GiB | 1 core / 2 GiB | teaching and light interactive use |
| Medium | 3 cores / 18 GiB | 3 | 9 cores / 54 GiB | 1 core / 4 GiB | the default |
| Large | 6 cores / 36 GiB | 10 | 60 cores / 360 GiB | 2 cores / 8 GiB | today's Large |
| extra large | 6 cores / 36 GiB | 13 (20 with a third node) | 78 cores / 468 GiB (120 / 720) | 4 cores / 16 GiB | the same executor, more of them; would be gated by a KBase role (there is no approval process for dedicated clusters: decision 2026-10-11, nobody gets one) |

The executor column repeats on purpose: the shapes are the pool's packing units, and a profile differs from the next one by how many of them it may hold, not by their size. The total column is what a user can take at full burst; with dynamic allocation it is a ceiling, and an idle user of any profile holds nothing.

## 6. Related

- [cpu_ram_capacity_analysis.md](cpu_ram_capacity_analysis.md): node inventory, the shared-cluster capacity model (section 3) and the per-user cluster model kept for the rollback path (section 6).
- Notebook image: [spark_notebook](https://github.com/KBaseDataLakehouse/spark_notebook) (Connect server, dynamic allocation, watchdog); hub: [BERDL_JupyterHub](https://github.com/KBaseDataLakehouse/BERDL_JupyterHub) (profiles, role gates); cluster manager: [spark_cluster_manager](https://github.com/KBaseDataLakehouse/spark_cluster_manager) (dedicated clusters, usage reporter); worker image: [kube_spark_manager_image](https://github.com/KBaseDataLakehouse/kube_spark_manager_image).
