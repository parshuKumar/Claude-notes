# 18 — Docker Compose in Depth

## Section: Docker in Practice

---

## ELI5 — The Simple Analogy

Imagine you are setting up a puppet show. You have three puppets: a **storyteller** (your API), a **memory keeper** (Postgres, who remembers everything), and a **fast helper** (Redis, who remembers things quickly but temporarily).

Up to now (Topics 12–17) you have been holding each puppet up by hand, one at a time, shouting all the instructions yourself: "You stand here! You wear this! You talk to that puppet over there!" That is `docker run` — one long command per puppet, typed by hand, every single time.

Docker Compose is a **script for the whole show written on one sheet of paper**. You write down, once: which puppets exist, what they wear, where they stand, and who is allowed to talk to whom. Then you say one word — "**up**" — and the entire show sets itself up exactly the same way, every time, on any stage.

The sheet of paper is a file called `docker-compose.yml`. That is the whole idea.

---

## The Linux kernel feature underneath

Here is the honest truth that most tutorials hide: **Docker Compose introduces no new kernel feature at all.**

Everything Compose does is built out of the *exact same kernel primitives* you already met:

- **Namespaces** (Topic 02) — each service still gets its own PID, network, mount, UTS, IPC, and user namespace. Compose just calls the daemon to create them.
- **cgroups** (Topic 02) — resource limits in Compose (`mem_limit`, `cpus`) become the same cgroup writes you saw in Topic 12.
- **overlay2** (Topic 04) — every service's filesystem is still layers stacked by overlayfs under `/var/lib/docker/overlay2/`.
- **veth pairs + Linux bridge + iptables** (Topic 13) — the network Compose creates is a normal Docker bridge network with veth pairs and `iptables` NAT rules.

So what *is* Compose, at the mechanical level?

```
        You                     Compose is a CLIENT of the daemon
         │
         │ writes
         ▼
  docker-compose.yml  ─────►  docker compose up
                                     │
                                     │ Compose reads the YAML, then makes the
                                     │ SAME REST API calls to the Docker daemon
                                     │ that `docker run`, `docker network create`,
                                     │ and `docker volume create` would make.
                                     ▼
                              Docker daemon (dockerd)
                                     │  (Topic 03: dockerd → containerd → runc)
                                     ▼
                              containerd → runc → clone() with namespace flags
                                     ▼
                              Linux kernel: namespaces, cgroups, overlayfs
```

**Compose is a specification + an orchestration client.** It reads a declarative YAML file, works out what containers/networks/volumes *should* exist, compares that to what *does* exist, and makes the daemon API calls to close the gap. That "declare desired state, let the tool reconcile" idea is the seed that grows into Kubernetes (Topic 30 — the Reconciliation Loop). Learn it here, small, and Kubernetes will feel familiar later.

---

## What is this?

Docker Compose is a tool for defining and running **multi-container applications** with a single YAML file and a single command. You describe your services (containers), the networks that connect them, and the volumes that persist their data — declaratively — in `docker-compose.yml`. Then `docker compose up` creates all of it, wired together, in dependency order.

It is the answer to a simple pain: real apps are never one container. Your `orders-api` needs Postgres and Redis to do anything useful, and typing three long `docker run` commands in the right order, with the right network, every time, is error-prone misery.

---

## Why does it matter for a backend developer?

Without Compose, starting your dev stack looks like this — and you must remember it perfectly, in order, every morning:

```bash
docker network create orders-net
docker volume create pgdata
docker run -d --name postgres --network orders-net \
  -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=orders \
  -v pgdata:/var/lib/postgresql/data postgres:16
docker run -d --name redis --network orders-net redis:7
docker run -d --name orders-api --network orders-net -p 3000:3000 \
  -e DATABASE_URL=postgres://postgres:secret@postgres:5432/orders \
  -e REDIS_URL=redis://redis:6379 orders-api:latest
```

Six commands. Get the order wrong, or forget the network flag, and `orders-api` can't reach the database. A new teammate joins and you paste them a wall of shell. Nobody knows if the "real" setup matches what's in your head.

With Compose, that entire block becomes **one file checked into git** plus:

```bash
docker compose up
```

What this unlocks for you specifically:

- **Reproducibility** — the stack is identical on your laptop, your teammate's laptop, and CI.
- **Documentation-as-code** — the `docker-compose.yml` *is* the answer to "how do I run this locally?"
- **Correct wiring** — Compose builds a private network and gives every service a DNS name automatically (more below), so `orders-api` reaches Postgres at the hostname `postgres` with zero manual networking.
- **Lifecycle in one place** — `up`, `down`, `logs`, `ps` operate on the whole stack at once (Topic 22).

This is the daily-driver tool for local development and small deployments. You will use it constantly before you ever touch Kubernetes.

---

## The physical reality

When you run `docker compose up` in a directory, what actually gets created on your machine?

**1. A project name.** Compose derives a *project name* — by default, the lowercased name of the directory (e.g. folder `orders-api` → project `orders-api`). Everything it creates is prefixed and labeled with this name so multiple projects don't collide.

**2. Labels on every object.** Inspect any container Compose made:

```bash
docker inspect orders-api-orders-api-1 --format '{{json .Config.Labels}}' | tr ',' '\n'
```

You will see labels like:

```
"com.docker.compose.project":"orders-api"
"com.docker.compose.service":"orders-api"
"com.docker.compose.container-number":"1"
"com.docker.compose.config-hash":"9f2c..."
```

These labels are how Compose *finds its own stuff* later. `docker compose down` doesn't remember anything in a database — it queries the daemon: "give me all containers with label `com.docker.compose.project=orders-api`" and removes them. This is the physical reality: **Compose is stateless; the state lives as labels on Docker objects.**

**3. A default network** (unless you override it):

```bash
docker network ls
# NETWORK ID     NAME                  DRIVER    SCOPE
# a1b2c3d4e5f6   orders-api_default    bridge    local
```

Named `<project>_default`, driver `bridge`. On disk this is a real Linux bridge; run `ip link` and `brctl show` (or `bridge link`) on the host and you'll see a `br-<id>` interface with veth pairs attached — exactly the mechanics from Topic 13.

**4. Named volumes**, prefixed with the project:

```bash
docker volume ls
# DRIVER    VOLUME NAME
# local     orders-api_pgdata
```

Physically at `/var/lib/docker/volumes/orders-api_pgdata/_data/` on the host — the same location any named volume lives (Topic 14).

**5. Containers**, named `<project>_<service>_<index>` (or `<project>-<service>-<index>` in newer versions):

```bash
docker ps
# NAMES
# orders-api-orders-api-1
# orders-api-postgres-1
# orders-api-redis-1
```

The `-1` index exists because a service can be scaled to multiple replicas (`docker compose up --scale redis=3` → `redis-1`, `redis-2`, `redis-3`).

There is no magic daemon, no Compose server process running in the background. After `up -d` returns, Compose exits. The containers keep running because the Docker daemon runs them, not Compose.

---

## How it works — step by step

Let's trace exactly what happens when you type `docker compose up` in a folder containing a `docker-compose.yml`.

1. **Find the file.** Compose looks in the current directory for `compose.yaml`, then `compose.yml`, then `docker-compose.yaml`, then `docker-compose.yml` (in that preference order). It also auto-merges `docker-compose.override.yml` if present (Topic 21).

2. **Parse and validate the YAML.** Compose reads the file into memory, checks it against the Compose Specification schema, expands `${VARIABLE}` references from your shell environment and any `.env` file in the directory, and resolves `extends`/`merge` rules. If a field name is misspelled or a value has the wrong type, it fails here — before touching Docker.

3. **Compute the project name.** From `-p`, or the `name:` top-level key, or `COMPOSE_PROJECT_NAME`, or (fallback) the directory name.

4. **Resolve the desired state into a dependency graph.** Compose reads `depends_on` edges and topologically sorts the services. In our case: `postgres` and `redis` have no dependencies; `orders-api` depends on both. So the start order is `{postgres, redis}` first, then `orders-api`.

5. **Reconcile networks.** For each network in the (implicit or explicit) `networks:` section, Compose asks the daemon "does a network with my project label and this name exist?" If not, it issues the equivalent of `docker network create`. The default network `<project>_default` is created here.

6. **Reconcile volumes.** Same reconcile for each named volume: exists? no → `docker volume create <project>_<name>`. **Named volumes are never deleted by `up`** — data survives restarts.

7. **Pull or build images.** For services with `image:`, Compose pulls if the image isn't present locally. For services with `build:`, Compose builds the image from the Dockerfile (Topics 06–11), tags it, and uses it.

8. **Create and start containers in dependency order.** For each service, Compose translates the YAML into the daemon's container-create API call — the programmatic equivalent of the giant `docker run` you'd type by hand — attaches it to the network(s), mounts the volumes, injects env vars, and starts it. It waits for `depends_on` conditions (Topic 20) before starting dependents.

9. **Attach to logs** (if running in the foreground, i.e. without `-d`). Compose multiplexes the stdout/stderr of every container into your terminal, each line prefixed with the service name, and forwards `Ctrl-C` as a stop signal to all of them.

On the *next* `docker compose up`, steps 5–8 become no-ops for anything already in the desired state — Compose only creates what's missing and recreates what *changed* (e.g. you edited an env var or bumped an image tag). That diff-and-converge behavior is reconciliation in miniature.

---

## Exact syntax breakdown

### The three top-level keys you care about first

A `docker-compose.yml` is a YAML map. The top-level keys that matter most are `services`, `networks`, and `volumes`. Here is the skeleton, annotated:

```yaml
services:            # ← REQUIRED. Map of containers to run. Each key = one service.
  orders-api:        # ← a service name. Becomes a DNS hostname + container name part.
    image: node:20-alpine
  postgres:
    image: postgres:16

networks:            # ← OPTIONAL. Declare custom networks. Omit → Compose makes one default.
  backend:           # ← a network name (becomes <project>_backend on disk).
    driver: bridge

volumes:             # ← OPTIONAL. Declare named volumes for persistent data.
  pgdata:            # ← a volume name (becomes <project>_pgdata on disk).
```

Line-by-line pointer view of the `services` block:

```
services:
│        └── the top-level key. Everything nested under it is one service each.
│
  orders-api:
  │         └── SERVICE NAME. Three roles at once:
  │             1. logical name in this file (used by depends_on)
  │             2. DNS hostname other services resolve (Topic 20)
  │             3. part of the container name: <project>-orders-api-1
  │
    image: node:20-alpine
    │      │    │  └── tag: which version. Defaults to `latest` if omitted (avoid that).
    │      │    └───── image repository name.
    │      └────────── the field: which image to run this service from.
    └───────────────── indentation = "this belongs to orders-api". YAML is space-sensitive.
```

### Naming: `compose.yaml` vs `docker-compose.yml`

```
docker-compose.yml
│              │  └── extension. `.yml` and `.yaml` both work.
│              └───── the modern spec prefers `compose.yaml`, but docker-compose.yml
│                     is still fully supported and by far the most common in the wild.
└──────────────────── historical name from the standalone `docker-compose` (v1) era.
```

Both names work. This book uses `docker-compose.yml` because that's what you'll see in 90% of existing projects.

### The command itself

```
docker compose up -d
│      │       │  └── DETACHED: run containers in the background, return to shell.
│      │       │       Without it, Compose stays attached and streams all logs.
│      │       └───── SUBCOMMAND: create + start the whole stack (reconcile to "up").
│      └──────────── the Compose plugin (v2). Note: SPACE, not a hyphen.
└─────────────────── the docker CLI.
```

Note the space: modern Compose (v2) is a *plugin* invoked as `docker compose`. The old standalone binary was `docker-compose` (hyphen). They are nearly identical for our purposes; prefer `docker compose`.

---

## Example 1 — basic

The simplest useful Compose file: just `orders-api` and Redis, so you can see the shape before we add complexity.

```yaml
# docker-compose.yml — minimal two-service stack
services:
  orders-api:                          # service #1: our Node/Express API
    image: orders-api:latest           # use a locally-built image (Topics 06–11)
    ports:
      - "3000:3000"                    # host:container — reach it at localhost:3000
    environment:
      REDIS_URL: redis://redis:6379    # note the hostname is literally "redis" ↓
    depends_on:
      - redis                          # start redis before orders-api

  redis:                               # service #2: the cache
    image: redis:7-alpine              # official Redis image, small alpine variant
```

Run it:

```bash
docker compose up -d
```

Every line explained:

- `services:` — the required top-level key; its children are our two containers.
- `orders-api:` / `redis:` — service names. Because they share a network (the auto-created default), `orders-api` can reach Redis using the hostname **`redis`** — that's why `REDIS_URL` says `redis://redis:6379`. No IP addresses. This is service discovery by name; the deep dive is Topic 20.
- `ports:` — publishes container port 3000 to host port 3000 (Topic 12). Redis has no `ports:` because only `orders-api` needs to reach it — it stays private on the network.
- `depends_on: [redis]` — tells Compose to *start* redis first. (Important nuance: this waits for redis to *start*, not to be *ready* — Topic 20 fixes that with health conditions.)
- Notice there is **no `networks:` and no `volumes:` block**. Compose still creates one default network (`<project>_default`) and attaches both services to it. That default network is what makes the `redis` hostname resolve.

---

## Example 2 — production scenario

Your team's `orders-api` is Express + Postgres + Redis. A new backend hire spent a whole afternoon fighting a "connection refused" bug because they ran the three `docker run` commands but forgot `--network`, so the containers couldn't see each other. Never again — you commit a Compose file that encodes the *entire* dev stack, including the private network and persistent database volume.

```yaml
# docker-compose.yml — the full orders-api dev stack
services:
  orders-api:
    build: .                                   # build from ./Dockerfile (Topic 09)
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://orders_user:orders_pass@postgres:5432/orders
      REDIS_URL: redis://redis:6379
      NODE_ENV: development
    depends_on:
      - postgres
      - redis
    networks:
      - backend

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: orders_user
      POSTGRES_PASSWORD: orders_pass
      POSTGRES_DB: orders
    volumes:
      - pgdata:/var/lib/postgresql/data        # persist DB across restarts
    networks:
      - backend

  redis:
    image: redis:7-alpine
    networks:
      - backend

networks:
  backend:                                     # one explicit private network
    driver: bridge

volumes:
  pgdata:                                      # named volume — survives `down` (without -v)
```

Why each production choice matters:

- **`build: .` instead of `image:`** — during development you want Compose to build `orders-api` from your local source each time, not pull a stale published image. (In production you'd flip this to `image:` with a versioned tag from your registry — Topic 24.)
- **Explicit `networks: backend`** — we *could* rely on the default network, but naming it makes intent obvious and lets us later add a second network (e.g. an `edge` network for a reverse proxy that Redis and Postgres are NOT on, so they stay unreachable from the outside). Explicit beats implicit for anything a teammate will read.
- **`pgdata` named volume** — without it, every `docker compose down` + `up` would give you an empty database. With it, your seed data and migrations survive. The volume lives at `/var/lib/docker/volumes/<project>_pgdata/_data`.
- **Postgres and Redis have no `ports:`** — they are only reached *by other containers on `backend`*, via DNS names `postgres` and `redis`. Not publishing their ports means nothing on your laptop's network (or a coworker's, or an attacker's) can reach your database directly. Least exposure by default.
- **Secrets in plaintext here** are fine for *local dev only*. Real secrets come from `.env`/secret stores — Topics 15 and 50. Never commit production credentials into `docker-compose.yml`.

Bring it up and prove the wiring:

```bash
docker compose up -d
docker compose ps
docker compose exec orders-api sh -c 'nc -zv postgres 5432 && nc -zv redis 6379'
# postgres (172.x.x.x:5432) open
# redis    (172.x.x.x:6379) open
```

---

## Common mistakes

### Mistake 1 — Assuming `depends_on` waits for the database to be *ready*

**Wrong mental model:** "`depends_on: [postgres]` means my API won't start until Postgres can accept queries."

**Reality:** by default `depends_on` only waits for the container to be *started* (its process launched), not *ready* (accepting connections). Postgres takes a few seconds to initialize. Your API starts immediately and crashes:

```
Error: connect ECONNREFUSED 172.18.0.2:5432
    at TCPConnectWrap.afterConnect [as oncomplete] (node:net:1595:16)
```

**Root cause:** starting a process and being ready to serve are different moments. The kernel has created the container's namespaces and PID, but Postgres inside it is still running its init scripts. `depends_on` (in its plain list form) checks only "did the container start."

**Right:** use the long-form `depends_on` with `condition: service_healthy` plus a `healthcheck` on Postgres. Full recipe in Topic 20. In your app you should *also* retry the connection — dependencies can restart at any time, not just at boot.

### Mistake 2 — Wrong indentation (tabs, or misaligned spaces)

```yaml
services:
  orders-api:
	image: node:20-alpine    # ← a TAB was used here
```

**Error:**

```
yaml: line 3: found character that cannot start any token
```

**Root cause:** YAML forbids tabs for indentation and is strictly space-sensitive. A tab, or two-space vs three-space misalignment, changes which parent a key belongs to — or breaks parsing entirely.

**Right:** use spaces only, consistent width (2 is conventional). Configure your editor to show whitespace and convert tabs to spaces for `.yml` files.

### Mistake 3 — Publishing ports you didn't need to

```yaml
  postgres:
    image: postgres:16
    ports:
      - "5432:5432"       # ← exposes your DB to the whole host network
```

**Symptom:** works fine, feels harmless — until you realize *anything that can reach your machine* can now hit your database on 5432, and it can *collide* with a Postgres already running on your host:

```
Error starting userland proxy: listen tcp4 0.0.0.0:5432: bind: address already in use
```

**Root cause:** `ports:` creates a host-level port binding + iptables DNAT rule (Topic 13). Other containers never needed it — they reach Postgres over the private Compose network by hostname. Publishing is only for traffic coming *from the host or outside*.

**Right:** only publish ports for services the outside world must reach (your `orders-api` on 3000). Leave Postgres/Redis unpublished. If you *do* need local DB access for a GUI tool, bind to loopback only: `"127.0.0.1:5432:5432"`.

### Mistake 4 — Expecting `docker compose down` to delete your data

```bash
docker compose down          # stops and removes containers + networks
docker compose up -d         # DB is... still there. Good.
docker compose down -v       # -v ALSO removes named volumes → data GONE
```

**Root cause:** `down` removes containers and the default network but **deliberately preserves named volumes** so you don't lose your database. The `-v`/`--volumes` flag opts into deleting them.

**Right:** know that `-v` is destructive. Use plain `down` for a clean restart that keeps data; use `down -v` only when you truly want a fresh, empty database.

### Mistake 5 — Editing the YAML but not recreating the container

```bash
# you change an environment: value, then run:
docker compose start orders-api   # ← start just restarts the EXISTING container
```

**Symptom:** your new env var isn't picked up.

**Root cause:** `start` reuses the already-created container (with its old config). Environment variables are baked in at container *create* time.

**Right:** run `docker compose up -d` again. Compose diffs the desired config against the running container, sees the change, and **recreates** just that service. (`docker compose up -d --force-recreate <service>` forces it.)

---

## Hands-on proof

Run these right now to *see* Compose's physical reality, not just trust it.

```bash
# 1. Make a working directory and a minimal compose file
mkdir -p /tmp/compose-proof && cd /tmp/compose-proof
cat > docker-compose.yml <<'EOF'
services:
  orders-api:
    image: nginx:alpine          # stand-in for the API so this runs with no build
    ports:
      - "3000:80"
  redis:
    image: redis:7-alpine
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: proof
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
EOF

# 2. Bring it up
docker compose up -d

# 3. PROOF: three containers, all prefixed by the project (folder) name
docker compose ps

# 4. PROOF: Compose created a default bridge network named <project>_default
docker network ls | grep compose-proof

# 5. PROOF: every container carries Compose's identifying labels
docker inspect compose-proof-redis-1 \
  --format '{{json .Config.Labels}}' | tr ',' '\n' | grep com.docker.compose

# 6. PROOF: service discovery by name works — redis resolves postgres over DNS
docker compose exec redis getent hosts postgres

# 7. PROOF: the named volume exists on the host filesystem
docker volume ls | grep pgdata
docker volume inspect compose-proof_pgdata --format '{{ .Mountpoint }}'
# → /var/lib/docker/volumes/compose-proof_pgdata/_data

# 8. PROOF: data survives `down` but NOT `down -v`
docker compose down            # containers gone
docker volume ls | grep pgdata # volume STILL THERE
docker compose down -v         # now the volume is deleted too
docker volume ls | grep pgdata # gone
```

If step 6 prints an IP and the hostname `postgres`, you have just watched Compose's embedded DNS resolve one service to another — with zero networking config from you. That is the payoff.

---

## Practice exercises

### Exercise 1 — easy

Write a `docker-compose.yml` with a single service named `cache` running `redis:7-alpine`, published on host port `6380` mapped to container port `6379`. Bring it up, confirm with `docker compose ps` that the container name is `<yourfolder>-cache-1`, then connect with `docker compose exec cache redis-cli ping` and confirm it returns `PONG`. Tear it down with `docker compose down`.

### Exercise 2 — medium

Extend Exercise 1 into a two-service file: add a `postgres:16` service named `db` with `POSTGRES_PASSWORD` set, and give it a **named volume** for `/var/lib/postgresql/data`. Do NOT publish the DB's port. Then: (a) create a table using `docker compose exec db psql ...`, (b) run `docker compose down` (without `-v`), (c) `docker compose up -d` again, and (d) prove the table is still there. Explain in one sentence *why* it survived.

### Exercise 3 — hard (production simulation)

Build the full `orders-api` + `postgres` + `redis` stack from Example 2, but with the ports deliberately misconfigured so Postgres and Redis are NOT publicly published and only `orders-api` is on `3000`. Then simulate a security review: from inside the `orders-api` container prove it *can* reach both `postgres:5432` and `redis:6379` (use `nc -zv`), and from your host prove you *cannot* reach 5432 or 6379 directly (`nc -zv localhost 5432` should refuse). Write two sentences explaining, in terms of iptables/port-publishing (Topic 13), why the container can reach the DB but the host cannot.

---

## Mental model checkpoint

Answer these from memory before moving on:

1. Does Docker Compose add any new Linux kernel feature? If not, what is it actually doing?
2. If you delete the `docker-compose.yml` file after `up`, do the containers stop? Why or why not?
3. Where does Compose "remember" which containers belong to a project — is there a database?
4. What are the three top-level keys you'll use most, and what does each declare?
5. If you write no `networks:` block at all, can `orders-api` still reach `redis` by the hostname `redis`? Why?
6. What's the difference between `docker compose down` and `docker compose down -v`?
7. Why is `depends_on` alone not enough to guarantee your API can talk to Postgres at startup?

---

## Quick reference card

| Field / Command | What it does | Key detail |
|---|---|---|
| `services:` | Top-level key; each child is one container | Required. Service name → DNS hostname + container name |
| `networks:` | Top-level key; declares custom networks | Omit → Compose creates `<project>_default` bridge |
| `volumes:` | Top-level key; declares named volumes | Persist across `down`; deleted only by `down -v` |
| `image:` | Which image to run a service from | Pulled if absent; add a real tag, avoid `latest` |
| `build:` | Build the service's image from a Dockerfile | Use in dev; switch to `image:` + tag in prod |
| `ports:` | Publish container port to the host | `"host:container"`; only for outside-reachable services |
| `depends_on:` | Start ordering between services | Waits for *start*, not *ready* (see Topic 20) |
| `docker compose up -d` | Reconcile the whole stack to "running", detached | Creates only what's missing; recreates what changed |
| `docker compose down` | Stop + remove containers and default network | Keeps named volumes |
| `docker compose ps` | List this project's containers | Reads containers by project label |

---

## When would I use this at work?

1. **Local development stack.** You clone the `orders-api` repo, run `docker compose up`, and instantly have the API, Postgres, and Redis running and wired together — no "install Postgres on my Mac" onboarding doc. New hires are productive in minutes.

2. **Integration tests in CI.** Your CI pipeline (Topic 27) runs `docker compose up -d`, waits for health, runs the integration test suite against the real Postgres and Redis (not mocks), then `docker compose down`. Same stack as dev, so "works on my machine" stops being a lie.

3. **Small single-host deployments.** For an internal tool or a low-traffic service that doesn't justify Kubernetes yet, a `docker-compose.yml` on one VM (with `restart: unless-stopped`) is a legitimate, maintainable production deployment. When it outgrows one host, the mental model you learned here transfers directly to Kubernetes.

---

## Connected topics

- **Study before:** Topic 12 (docker run in Depth — Compose is `docker run` in YAML), Topic 13 (Networking — the default network is a bridge with veth + iptables), Topic 14 (Volumes — named volumes are what `volumes:` declares), Topic 15 (Environment Variables — `environment:`/`env_file:`).
- **Study after:** Topic 19 (Compose File Deep Dive — every field in detail), Topic 20 (Multi-Container Apps — service discovery and health-based startup order), Topic 21 (Compose for Development — hot reload and overrides), Topic 22 (Compose Commands). Further out: Topic 30 (The Reconciliation Loop — the idea Compose seeds, Kubernetes perfects).
