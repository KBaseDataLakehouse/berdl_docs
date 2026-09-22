# KIND*AI Integration Guide

KIND*AI is the graphical, natural-language front door to the BERDL lakehouse:
it skins and steers a KOROS (Claude Code) research session behind a web UI, and
launches the data portal apps (Lakehouse Explorer, Diaspora, ENIGMA Strata,
Function Junction, Fungal Jungle, GenePool, genKnown, Plant Terra). This image
bakes the whole KIND runtime in and starts it automatically for opted-in pods,
so role-holding users land on KIND with zero installation.

This guide covers both how to use it and how the integration works.

## For users

### What you'll experience

If you hold a KIND role (`KBASE_STAFF` or `BERDL_KIND_USER`), logging into the
hub lands you on the KIND UI at `https://hub.berdl.kbase.us/user/<you>/proxy/8770/`
instead of JupyterLab. Your pod auto-starts the KIND server at boot and
pre-warms the app portals.

- **JupyterLab is still there**: click the **JupyterLab icon** (the Jupyter
  logo, tooltip "open JupyterLab") in KIND's top bar and your notebook
  environment opens in a new tab — same server, same files, same running
  kernels; nothing is restarted, and the KIND tab stays open so you can switch
  between the two. You can also open `https://hub.berdl.kbase.us/user/<you>/lab`
  directly, or bookmark it. Explicit `/lab` URLs always go to Lab — only the
  bare server root (`https://hub.berdl.kbase.us/user/<you>/`) redirects to KIND.
- **Driving sessions needs an LLM credential** — BERDL does not provide one.
  Either run `claude` login in a hub terminal, or put a CBORG key in `~/.env`
  (`CBORG_API_KEY=...`). The doctor view under **⚙ Settings** in KIND's top
  bar tells you what's missing.
- **Your research state lives in your home** (`~/koros/runs`, KIND state under
  `~/.local/share/king/runs`) and survives pod restarts and culls.
- **Tables and files you (or an app) create go to two different places.**
  Tables belong in your personal Iceberg catalog (`my` in Spark, `<you>` in
  Trino) and are written through Spark; the catalog manages their S3 layout
  under `users-sql-warehouse/<you>/`, which is list-only for you. Free-form
  files (TSVs, raw exports, staged inputs) go under
  `s3a://cdm-lake/users-general-warehouse/<you>/`. When a table reference has
  to outlive a session or be used from Trino, store the `<you>.namespace.table`
  form, not `my.…`. See the [Tenant SQL Warehouse](tenant_sql_warehouse_guide.md)
  and [S3](s3_guide.md#what-you-can-access-and-why-you-get-accessdenied) guides.

### If something looks wrong

| Symptom | Fix |
|---|---|
| You land on JupyterLab although you hold a KIND role | Log out of the hub, log back in, **then** stop + start your server (that order — the role is stamped into your session at login) |
| KIND shows an error page right after spawn | The server is still booting (~15 s); reload |
| KIND looks old / no Jupyter button in the top bar (you once installed KIND manually) | The built-in build always wins over a leftover `~/.local/bin` install, so a stop + start of your server is enough — unless you created `~/.kind/prefer-user-install` (see *Running your own build*). Your `~/king` checkout and research state are untouched |
| `KIND seed NOTICE: ~/koros … not refreshed` in the pod log / KOROS features missing after an image update | Your `~/koros` (or `~/semcat`) has local changes, so the boot refresh left it alone. Commit/stash them or move the directory aside (`mv ~/koros ~/koros.mine`; `runs/` can be copied back) and stop + start your server — a clean checkout is refreshed to the image's pinned version automatically, with the previous tree kept under `~/.kind/backups/` |
| "KBase token expired" although you just logged in | Stop + start your server (a fresh server picks up your current login); if it persists, log out, log in again, then restart |

### Getting the latest KIND version

A stop + start of your server (hub control panel) is the complete update: the
image carries a pinned, tested set of KIND, KOROS, semcat, narrative-connector,
Lakehouse Explorer and every app portal, and at boot your `~/koros` and
`~/semcat` checkouts are brought to the same versions (your `runs/` and the
previous tree are kept — see *How it works → Boot*). Nothing to install.

**Can't restart yet (a long-running job)?** The parts that live in your home
can be updated in place; the parts baked into the image cannot.

| Component | Lives in | Update without a server restart |
|---|---|---|
| KOROS (`~/koros`) and the semantic catalog (`~/semcat`) | your home | Yes — in a hub terminal: `gh auth login` once (the checkouts are private kbaseincubator repos; the seeded copy carries no credentials), then `git -C ~/koros fetch --tags origin && git -C ~/koros checkout v2026.08.28` (the cut you want; same for `~/semcat`). KOROS picks it up on your next session turn — no relaunch needed, and your next server restart will **not** undo it (the boot refresh never downgrades a checkout that is newer than the image's) |
| KIND server, narrative-connector, Lakehouse Explorer, the app portals | the image (`/opt/berdl/kind/venv`) | No — wait for the restart, or run a newer version as *your own build* (below) and relaunch only the KIND process |

**Relaunching only the KIND process** never touches your kernels, Spark
cluster or running jobs; the app portals keep running and KIND re-adopts them:

```bash
python3 -m berdl_notebook_utils.kind_server --restart     # in a hub terminal; then reload the KIND tab
```

The KIND command-line tools are **not** on your terminal's `PATH` (the venv is
kept separate from the notebook's Python). Call them by full path,
`/opt/berdl/kind/venv/bin/semcat doctor --spark`, or add
`export PATH=/opt/berdl/kind/venv/bin:$PATH` to `~/.custom_profile`.

### Running your own build of KIND, KOROS or an app

The image-baked build wins by default, so a forgotten leftover install can
never shadow a newer image — but developers and testers can opt out:

1. Install your build into your home with KIND's own tooling
   ([`kbaseincubator/kind` INSTALL.md](https://github.com/kbaseincubator/kind/blob/main/docs/INSTALL.md)):
   `gh repo clone kbaseincubator/kind ~/king && bash ~/king/scripts/install.sh`,
   then `kind update [@<cut>]` / `kind install <app>[@<tag>]` as you like. This
   puts `kind`, the capability CLIs and the app CLIs in `~/.local/bin`.
2. Opt in: `touch ~/.kind/prefer-user-install` (or, for one launch only, set
   `BERDL_KIND_PREFER_USER_INSTALL=true` in the environment of whatever starts
   KIND — the Jupyter server does not read `~/.env`).
3. Relaunch KIND: `python3 -m berdl_notebook_utils.kind_server --restart`.

While the marker exists **everything** in `~/.local/bin` (server and CLIs)
takes precedence over the image, so image updates no longer reach you until
you remove it (`rm ~/.kind/prefer-user-install` + relaunch). Manual
development via `serve.sh` from your own checkout also still works — the
autostart defers to any server already listening on the KIND port.

For KOROS and semcat you simply work in `~/koros` / `~/semcat`: a checkout
with local changes, or one newer than the image's pinned ref, is never
replaced at boot (a clean, older one is — with the previous tree kept under
`~/.kind/backups/`).

## How it works

### Image build (`Dockerfile`, `kind_builder` stage)

- Clones the private `kbaseincubator` repos at pinned ARGs (`KIND_REF`,
  `KOROS_REF`, `SEMCAT_REF`, `NARRATIVE_CONNECTOR_REF`, `LAKEHOUSE_EXPLORER_REF`
  plus one ARG per portal app) using the `gh_token` BuildKit secret
  (CI: the `KBASEINCUBATOR_READ_PAT` repo secret; locally:
  `export GH_TOKEN=$(gh auth token)` — compose wires it).
- Builds the self-contained `king_backend` wheel (SPA + plugins bundled — no
  node at runtime) via king's `scripts/package.sh`.
- Assembles `/opt/berdl/kind/`:
  - `venv/` — a `--system-site-packages` venv (sees the Spark/berdl runtime)
    holding king, semcat, narrative-connector, lakehouse-explorer, the portal
    apps, mmseqs2, and a `koros` CLI wrapper. The notebook's own Python
    environment is never touched.
  - `koros/`, `semcat/` — checkouts staged for seeding into user homes.
  - `REFS` — the pinned refs this image carries.
- koros is deliberately **not** pip-installed: its wheel ships flat top-level
  modules meant for pipx isolation; the wrapper dispatches to the user's
  seeded `~/koros` so code and run state stay one tree.

### Boot (opt-in via `BERDL_START_KIND=true`)

The hub injects `BERDL_START_KIND=true` and `default_url=/proxy/8770/` for
KIND-role users in `pre_spawn_start` (BERDL_JupyterHub reads KBase custom
roles into `auth_state.kind_user`; `KIND_ROLES` env on the hub names the
roles). Everything below no-ops on other pods.

1. **`configs/jupyter_docker_stacks_hooks/13-kind-seed.sh`** (root, before the
   notebook server): seeds `~/koros` + `~/semcat` if absent and **refreshes them to the image's
   pinned refs when they are git-clean and older than the image's** (the
   previous tree is kept under `~/.kind/backups/`, `runs/` — research state —
   is carried over; a checkout with local changes is never touched and is
   named in a `KIND seed NOTICE`; one the user moved ahead is left as is),
   opens the root-only `/opt/berdl/kind` for this pod, and maintains the
   `seeded-refs` drift marker. Pods that did not opt in never touch any of
   this and keep `/opt/berdl/kind` unreadable.
2. **`berdl_notebook_utils/kind_server.py`** (from
   `configs/jupyter_server_config.py`, as the user, in a daemon thread):
   idempotent launcher — skips if the KIND port is already listening (a
   developer's `serve.sh` wins), resolves the binary (baked venv first; a user
   install only with `BERDL_KIND_PREFER_USER_INSTALL=true`), runs the idempotent capability provisioning
   (`koros setup-hooks`, `narrative-connector install-skills`, `kind mrc-seed`),
   launches the server on `127.0.0.1:8770` with logs in `~/.kind/server.log`
   (deliberately not the pod log — session content stays out of the central
   log pipeline), waits for health, warms the caches + proxy route, and
   pre-launches the app fleet (`BERDL_KIND_WARM_APPS`).
3. **Idle-cull heartbeat**: emits a `terminal_command` marker
   (`source: kind_session_heartbeat`) into the activity log whenever KIND/KOROS
   run state changed since the last tick — so KIND-only users aren't reaped
   mid-arc. Activity-based, not presence-based: app access logs and health
   history are excluded, so an idle open tab cannot keep a pod alive.

### Security invariants

- KIND binds `127.0.0.1` only; the sole route in is the hub's authenticated
  `/user/<name>/proxy/8770/` (per-server OAuth). KIND itself does no
  per-request auth — **hub-proxy-only access and the loopback bind are
  load-bearing**.
- The KIND child environment carries **no KBase token**; KIND reads
  `~/.berdl_kbase_session` per request (rotation-current; `11-setup_env.sh`
  seeds it from the boot-fresh spawner token for every user), so token
  rotation reaches it without a restart.
- `/opt/berdl/kind` holds **private kbaseincubator code** (KIND, KOROS,
  semcat, the apps). The image ships it root-only (`0700`); `13-kind-seed.sh`
  opens it (`0755`) only on a pod that opted in, so a user without a KIND role
  cannot read it from their pod (pods run without sudo). The same reasoning
  requires the `spark_notebook` **container image itself to be private** in
  the registry — anyone able to pull the image can read the code.

### Configuration knobs

| Env var | Default | Meaning |
|---|---|---|
| `BERDL_START_KIND` | unset (off) | Opt the pod into seeding + autostart; hub-injected per role. Local compose defaults it on |
| `BERDL_KIND_WARM_APPS` | `all` | Apps to pre-launch after boot: `all`, a comma-separated id list, or `none` for launch-on-click |
| `KIND_PORT` / `KING_PORT` | `8770` | KIND server port |
| `BERDL_KIND_HEARTBEAT_SECONDS` | `300` | Cull-heartbeat scan interval |
| `KIND_KOROS_DIR` | `~/koros` | KOROS checkout the sessions run against |
| `BERDL_KIND_PREFER_USER_INSTALL` / `~/.kind/prefer-user-install` | unset | Prefer a user's own `~/.local/bin` install over the baked venv (env var = per process; marker file = persistent). Default: baked venv wins |

### Local development (docker compose)

The local stack enables everything by default (`BERDL_START_KIND=true`,
`BERDL_KIND_WARM_APPS=all`) and points KIND at CI auth
(`KBASE_ENDPOINT=https://ci.kbase.us/services/` — without it a CI token reads
as "expired" against KIND's prod default). Building needs
`export GH_TOKEN=$(gh auth token)`. For the KIND app portals to render inside
the UI locally, enable hub-parity paths: `BERDL_LOCAL_BASE_URL=/user/<you>/`
(put it in `.env` to make it sticky) — Lab then lives at
`localhost:8888/user/<you>/lab`, and the familiar root URLs 301-redirect there
(`configs/base_url_redirect.py`).

### Releasing a new KIND version

1. PR bumping the relevant Dockerfile ARG(s) (`KIND_REF=king-vX`, app refs as
   they tag). CI rebuilds the image.
2. Point `BERDL_NOTEBOOK_IMAGE_TAG` (hub configmap, dev first) at the new tag
   and restart the hub; users pick it up on their next server stop/start.
3. If **koros/semcat** moved: existing homes keep their seeded checkouts (by
   design) and only log a drift notice. Code-only refresh per user, research
   state preserved:
   `rsync -a --delete --exclude=runs/ --exclude=.env --exclude=.git/ /opt/berdl/kind/koros/ ~/koros/`
