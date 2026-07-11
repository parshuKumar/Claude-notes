# 22 — Compose Commands
## Section: Docker in Practice

---

## ELI5 — The Simple Analogy

Imagine you're the stage manager of a small theatre. Your `docker-compose.yml` is the **script and cast list**: it says which actors (services) are in the play, what costumes they wear (images), and where they stand (networks, ports). But a script sitting on a shelf does nothing. You need a set of **commands you shout from the wings** to make the show actually happen:

- "**Places, everyone!**" → `up` — get all actors on stage and start the show.
- "**Curtain down, everyone go home.**" → `down` — end the show, clear the stage.
- "**Sew the costumes.**" → `build` — prepare the outfits before anyone performs.
- "**What's everyone saying right now?**" → `logs` — listen to the actors' lines live.
- "**Actor number 3, do this real quick.**" → `exec` — walk up to one actor on stage and hand them a task.
- "**Who's on stage and are they okay?**" → `ps` — a roll call of who's up and their health.
- "**Actor 2, exit and re-enter.**" → `restart` — one actor resets without ending the show.
- "**Read me the final script after all the edits.**" → `config` — show the merged, final version of the script.

Each command is a different sentence you shout to the same cast. This topic teaches you exactly what each one does, and — just as important — *which flags matter*, because a flag like `-v` on `down` is the difference between "clear the stage" and "burn the theatre's furniture" (delete your database).

---

## The Linux kernel feature underneath

Docker Compose has **no special kernel powers of its own**. This is worth internalizing: every Compose command is a *thin orchestration layer* that translates your intent into ordinary Docker Engine API calls, which then use the same kernel features you already know — namespaces, cgroups, overlayfs (**Topic 02**).

Here's the real chain when you type `docker compose up`:

```
docker compose up
   │
   ├─ 1. Compose reads & merges YAML → an in-memory desired state
   │
   ├─ 2. Compose diffs desired vs actual (what containers/networks/volumes exist?)
   │
   └─ 3. For each difference, Compose calls the Docker Engine REST API over the
          UNIX socket /var/run/docker.sock, e.g.:
             POST /networks/create
             POST /volumes/create
             POST /containers/create      ← body has namespaces, mounts, env
             POST /containers/{id}/start
                     │
                     └─ dockerd → containerd → runc → clone()/unshare() + cgroups
                        (the exact 8-step kernel sequence from Topic 01)
```

So `docker compose up` is a **reconciliation loop in miniature** (a concept you'll meet again at cluster scale in **Topic 30**): it looks at what you *declared* in YAML, compares it to what's *actually running*, and makes the minimal set of API calls to close the gap. If a container already matches the spec, Compose leaves it alone. If its config changed, Compose recreates it. That's why running `docker compose up` twice doesn't create duplicate containers — the second run finds reality already matches desired state and does nothing.

Every command in this topic is a variation on that theme: read intent → compute a diff → make Docker Engine API calls → the Engine drives the kernel. Compose is the conductor; the kernel is the orchestra. Understanding this tells you *why* `down` can delete things `stop` can't, and why `exec` runs a **new** process while `restart` re-runs the **same** entrypoint.

---

## What is this?

The **Compose commands** are the CLI verbs (`up`, `down`, `build`, `logs`, `exec`, `ps`, `restart`, `config`, and more) you use to operate a multi-service application defined in a Compose file. Each verb maps to a Docker Engine action applied across *all* the services at once, respecting their dependencies and shared network.

Where `docker run` operates on **one** container, `docker compose` operates on the **whole application** — your `orders-api` + `postgres` + `redis` together — as a single named unit called a **project**.

---

## Why does it matter for a backend developer?

Because these commands *are* your daily interface to your local stack. You'll type `docker compose up`, `logs`, and `exec` dozens of times a day. Knowing them cold is the difference between fluidly debugging `orders-api` and fumbling.

But the real reason is **safety and speed**. Two commands look almost identical and one of them can destroy data:

```bash
docker compose down        # stop & remove containers + network — DATA SAFE
docker compose down -v      # ALSO deletes named volumes — your Postgres DB is GONE
```

If you don't understand the flags, you *will* eventually run `down -v` on the wrong project and lose your local database, or wonder for an hour why your code change isn't showing up (answer: you needed `--build`).

Without mastering these commands you will:
- Run `docker compose up` and not see your code change, because you didn't pass `--build` after editing the Dockerfile.
- Delete your local Postgres data with a careless `-v` and not understand why.
- Try to debug a service by SSH-ing into a VM when `docker compose exec api sh` would have dropped you inside instantly.
- Blindly edit YAML and get surprised, instead of running `docker compose config` to see the *actual* merged result first.
- Scale the wrong service, or scale a service that has a fixed host port and get a bind conflict.

---

## The physical reality

When you run any Compose command, several concrete things exist on your machine. Take `orders-api` started with `docker compose up -d` from `/home/you/orders-api`:

**1. A "project" — a namespace grouping — derived from the directory name.** Compose prefixes everything with the project name (default: the folder name, `orders-api`):

```
$ docker ps --format '{{.Names}}'
orders-api-api-1          ← <project>-<service>-<replica#>
orders-api-postgres-1
orders-api-redis-1
```

Every container, network, and volume carries **labels** recording its project, so Compose knows what belongs to this app:

```
$ docker inspect orders-api-api-1 --format '{{json .Config.Labels}}'
{
  "com.docker.compose.project": "orders-api",     ← how Compose finds "its" containers
  "com.docker.compose.service": "api",
  "com.docker.compose.container-number": "1"
}
```

This label is *how* `docker compose ps`, `logs`, and `down` know which containers to act on — they filter the whole Docker Engine by `com.docker.compose.project=orders-api`.

**2. A dedicated bridge network** (Topic 13) so services resolve each other by name:

```
$ docker network ls | grep orders
abc123  orders-api_default  bridge  local
```

**3. Named volumes** that survive `down` but not `down -v`:

```
$ docker volume ls | grep orders
orders-api_pgdata     ← Postgres data lives here, at /var/lib/docker/volumes/orders-api_pgdata/_data
```

**4. Log files** on disk that `docker compose logs` reads (default `json-file` driver, Topic 17):

```
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

So a running Compose project is physically: **a set of labelled containers + one bridge network + some named volumes + per-container log files**, all tied together by that `com.docker.compose.project` label. Every command in this topic filters or acts on things carrying that label.

---

## How it works — step by step

Let's trace the two most consequential commands.

### `docker compose up -d --build`

1. **Locate & merge files.** Compose reads `docker-compose.yml` and auto-merges `docker-compose.override.yml` (Topic 21) into one desired-state document.

2. **Resolve the project name.** From the directory name (or `-p`/`COMPOSE_PROJECT_NAME`). This becomes the label everything is tagged with.

3. **Build images (because `--build`).** For every service with a `build:` key, Compose runs `docker build` (Topic 06), producing fresh images. Without `--build`, Compose reuses existing images and only builds if the image is missing entirely.

4. **Create shared infrastructure.** Compose ensures the project network and any named volumes exist. If they already exist, it reuses them (that's why your Postgres data persists across `up`s).

5. **Compute the container diff.** For each service, Compose asks: does a container with this exact config already exist and run? If yes → leave it. If it exists but config changed → recreate it. If missing → create it.

6. **Start in dependency order.** Compose reads `depends_on` and healthchecks (Topic 20) and starts services in order: `postgres` and `redis` first (waiting for healthchecks if configured), then `api`.

7. **Detach (because `-d`).** With `-d`, Compose returns your shell immediately, leaving containers running in the background. Without `-d`, it *attaches* to all services' logs and streams them until you Ctrl-C (which stops the stack).

### `docker compose down -v`

1. **Find the project's objects** by the `com.docker.compose.project` label.
2. **Stop each container** — sends `SIGTERM`, waits `--timeout` seconds (default 10), then `SIGKILL` (Topic 16).
3. **Remove the containers** — deletes their writable layers and metadata.
4. **Remove the project network.**
5. **Because `-v`: remove named volumes too** — this is the step that deletes `orders-api_pgdata` and *permanently destroys your Postgres data*. Without `-v`, volumes are kept and re-attached on the next `up`.

The asymmetry to remember: **`up` is additive and safe to repeat; `down` is destructive, and `-v` makes it destroy persistent data.**

---

## Exact syntax breakdown

### `up` — the everyday "start everything" command

```
docker compose up -d --build --scale api=3 --no-deps
│              │  │  │       │             │
│              │  │  │       │             └─ --no-deps: start ONLY named services,
│              │  │  │       │                skip their depends_on. Used to restart
│              │  │  │       │                one service without touching Postgres/Redis
│              │  │  │       │
│              │  │  │       └─ --scale api=3: run 3 replica containers of "api"
│              │  │  │          (api-1, api-2, api-3). Fails if api has a fixed host port
│              │  │  │
│              │  │  └─ --build: rebuild images before starting (after Dockerfile/code
│              │  │     changes for a built service). Default: only build if image missing
│              │  │
│              │  └─ -d: detached — run in background, return the shell.
│              │     Omit it to attach and stream all logs until Ctrl-C
│              │
│              └─ up: reconcile desired state — create network/volumes, (re)create
│                 & start containers in dependency order
└─ docker compose: the CLI, operating on the whole project (all services)
```

### `down` — stop and remove; mind the flags

```
docker compose down -v --rmi local --remove-orphans -t 30
│                   │  │           │                 │
│                   │  │           │                 └─ -t 30: wait 30s for graceful
│                   │  │           │                    SIGTERM before SIGKILL (Topic 16)
│                   │  │           │
│                   │  │           └─ --remove-orphans: also remove containers for services
│                   │  │              no longer in the compose file
│                   │  │
│                   │  └─ --rmi local: also delete images built by this project
│                   │     (local = only images without a custom tag)
│                   │
│                   └─ -v / --volumes: ALSO delete named volumes → DESTROYS DB DATA.
│                      Omit -v to keep volumes and your data across restarts
└─ down: stop containers, remove containers + the project network
```

### `logs` — read what your services are printing

```
docker compose logs -f --tail 100 --timestamps api
│                   │  │           │            │
│                   │  │           │            └─ service name(s) to show. Omit → all services
│                   │  │           │
│                   │  │           └─ --timestamps: prefix each line with an ISO timestamp
│                   │  │
│                   │  └─ --tail 100: show only the last 100 lines (then follow, with -f)
│                   │
│                   └─ -f / --follow: stream new log lines live (like tail -f). Ctrl-C to stop
└─ logs: read the json-file logs (Topic 17) of the project's containers
```

### `exec` — run a command INSIDE an already-running container

```
docker compose exec -it -e DEBUG=1 -u root api sh
│                   │   │           │       │   │
│                   │   │           │       │   └─ command to run: "sh" → an interactive shell
│                   │   │           │       └─ service to enter (its running container)
│                   │   │           │
│                   │   │           └─ -u root: run as the root user (default: image's USER)
│                   │   │
│                   │   └─ -e DEBUG=1: set an env var just for this exec'd process
│                   │
│                   └─ -i (stdin) -t (TTY): make it interactive. Usually written -it
└─ exec: start a NEW process inside the EXISTING container (does not restart the app)
```

### `ps`, `restart`, `build`, `config` — the rest

```
docker compose ps -a                docker compose restart api
│              │  │                  │              │       │
│              │  └─ -a: include     │              │       └─ service(s); omit → all
│              │     stopped/exited  │              └─ restart: SIGTERM then start SAME
│              └─ ps: list this         │                 container (config unchanged)
│                 project's containers  └─ (config is NOT re-read; use up to apply YAML edits)
│                 + status + ports

docker compose build --no-cache api        docker compose config --services
│              │     │          │           │              │      │
│              │     │          └─ service  │              │      └─ --services: list service
│              │     └─ --no-cache: ignore  │              │         names only
│              │        layer cache (Topic 8)│              └─ config: print the FINAL merged,
│              └─ build: build images without│                 variable-substituted YAML
│                 starting anything          └─ (great for debugging overrides, Topic 21)
```

---

## Example 1 — basic

The core daily loop with `orders-api` + `postgres` + `redis`. Every command annotated.

```bash
# Start the whole stack in the background, building images first.
docker compose up -d --build
#              │  │  │
#              │  │  └─ build images (first run, or after Dockerfile change)
#              │  └─ detached: get your shell back
#              └─ create network + volumes, start postgres, redis, then api

# See what's running and whether it's healthy.
docker compose ps
# NAME                 SERVICE    STATUS                 PORTS
# orders-api-api-1     api        Up 12s                 0.0.0.0:3000->3000/tcp
# orders-api-postgres-1 postgres  Up 12s (healthy)       5432/tcp
# orders-api-redis-1   redis      Up 12s                 6379/tcp

# Follow the API's logs live while you hit an endpoint.
docker compose logs -f api
# api  | orders-api listening on :3000
# (Ctrl-C to stop following — the container keeps running)

# Jump inside the api container to poke around.
docker compose exec api sh
# /app # node -e "console.log(process.env.DATABASE_URL)"
# postgres://orders:secret@postgres:5432/orders
# /app # exit

# Run a one-off command against Postgres (using the postgres container's psql).
docker compose exec postgres psql -U orders -c '\dt'
#   → lists tables in the orders database

# Restart just the API (e.g. after changing an env var in the running container).
docker compose restart api

# Stop and remove everything BUT keep the database volume.
docker compose down
#   → containers + network removed; orders-api_pgdata volume KEPT (data safe)
```

The loop you'll live in: `up -d --build` once, then `logs -f` and `exec` all day, `down` at the end. Your Postgres data survives because you didn't pass `-v`.

---

## Example 2 — production scenario

**The situation.** It's 4pm. A teammate reports that `orders-api` is returning `500`s locally after they pulled the latest code. You need to diagnose fast without nuking anyone's database. Here's the exact command sequence a fluent engineer runs, and *why each one*.

```bash
# 1. Roll call: what's up, and is anything unhealthy or restarting?
docker compose ps -a
# NAME                  SERVICE   STATUS
# orders-api-api-1      api       Restarting (1) 3 seconds ago   ← the smoking gun
# orders-api-postgres-1 postgres  Up (healthy)
# orders-api-redis-1    redis     Up
#   → api is crash-looping. Postgres is healthy, so it's not the DB being down.
```

```bash
# 2. Read the crash reason from the logs — last 50 lines, no need to follow.
docker compose logs --tail 50 api
# api | Error: connect ECONNREFUSED 127.0.0.1:6379
# api |   at TCPConnectWrap.afterConnect
#   → It's trying Redis on 127.0.0.1 instead of the "redis" service name.
#     A teammate hardcoded localhost. In Compose, services talk by SERVICE NAME (Topic 13,20).
```

```bash
# 3. Confirm the config Compose actually computed (did an override sneak in?).
docker compose config | grep -A3 'REDIS_URL'
#   environment:
#     REDIS_URL: redis://127.0.0.1:6379     ← wrong, confirmed. Should be redis://redis:6379
```

```bash
# 4. Fix the .env / compose, then apply ONLY to api without disturbing Postgres/Redis.
#    up recreates api with the new config; --no-deps means don't touch its dependencies.
docker compose up -d --no-deps api
#   → only api-1 is recreated. Postgres keeps its connections and data untouched.
```

```bash
# 5. Verify the fix live.
docker compose logs -f api
# api | Connected to Redis at redis:6379
# api | orders-api listening on :3000     ← fixed, no more restart loop
```

Contrast with what a *panicked* engineer does: `docker compose down -v && docker compose up --build`. That "works" too — but `-v` **deleted the Postgres volume**, wiping the local orders data and every migration, turning a 2-minute config fix into a 20-minute re-seed. The fluent engineer used `ps` → `logs` → `config` → targeted `up --no-deps` and never risked the data.

**Now, a scaling scenario.** Suppose you want to load-test `orders-api` with three instances behind whatever you have locally:

```bash
# WRONG: api has "ports: 3000:3000" (a FIXED host port) → scaling collides.
docker compose up -d --scale api=3
# Error: driver failed programming external connectivity ... Bind for 0.0.0.0:3000 failed: port is already allocated
#   Root cause: 3 containers can't all claim host port 3000.

# RIGHT: remove the fixed host port (or use a range), let each replica get an ephemeral port.
#   In compose, change  ports: ["3000:3000"]  →  ports: ["3000"]  (host port auto-assigned)
docker compose up -d --scale api=3
docker compose ps
# orders-api-api-1  ... 0.0.0.0:32768->3000/tcp
# orders-api-api-2  ... 0.0.0.0:32769->3000/tcp
# orders-api-api-3  ... 0.0.0.0:32770->3000/tcp
#   → three replicas, each on its own host port. (Real load balancing across them
#     is a Kubernetes Service job — Topic 35 — but this proves the scaling mechanics.)
```

---

## Profiles — turning services on and off per use-case

Sometimes you don't want *every* service every time. You want Postgres and Redis always, but a `pgadmin` GUI or a `seed` job only occasionally. **Profiles** let a service opt out of the default `up` unless its profile is activated.

```yaml
services:
  api:
    build: .
    # no profiles → always started

  postgres:
    image: postgres:16-alpine
    # no profiles → always started

  pgadmin:
    image: dpage/pgadmin4
    profiles: ["tools"]        # only starts when the "tools" profile is active
    ports: ["5050:80"]

  seed:
    build: .
    command: node scripts/seed.js
    profiles: ["seed"]         # a one-shot job, only when you ask for it
```

```
docker compose --profile tools up -d
│              │         │     │
│              │         │     └─ up: start default services + those in "tools"
│              │         └─ "tools": the profile to activate
│              └─ --profile: enable a profile (repeatable: --profile tools --profile seed)
└─ (services with NO profiles always run; profiled ones run only when activated)
```

Behavior to remember:
- A service with **no `profiles:`** always starts on a plain `docker compose up`.
- A service **with** `profiles:` starts **only** when that profile is activated (via `--profile name` or `COMPOSE_PROFILES=name`).
- You can also target a profiled service by name directly: `docker compose run seed` starts it even without the flag, because you named it explicitly.

Use it so `docker compose up` gives everyone the lean core stack, while `--profile tools` adds optional extras only when needed.

---

## Common mistakes

**Mistake 1 — Expecting `up` to pick up a code/Dockerfile change without `--build`.**

You edit the Dockerfile (or code that gets baked into the image), run `docker compose up -d`, and your change isn't there.

```
# You added a dependency install to the Dockerfile, but:
$ docker compose exec api node -e "require('axios')"
Error: Cannot find module 'axios'
```

Root cause: `up` **reuses the existing image** if one already exists; it only builds when the image is *missing*. Your edit changed the Dockerfile, but the old image is still cached. Wrong: `docker compose up -d`. Right:
```bash
docker compose up -d --build      # force a rebuild, then start
```

**Mistake 2 — `down -v` on the wrong project (data loss).**

You meant to free some space and reflexively ran:

```
$ docker compose down -v
[+] Removing volume orders-api_pgdata      ← your entire local Postgres just vanished
```

Root cause: `-v` deletes **named volumes**, where Postgres stores its data (Topic 14). There is no undo. Wrong (when you want to keep data): `down -v`. Right: `docker compose down` (no `-v`) to stop the app but keep volumes; use `down -v` *only* when you deliberately want a clean database.

**Mistake 3 — Thinking `restart` re-reads the YAML.**

You change an environment variable in `docker-compose.yml`, run `docker compose restart api`, and the old value is still there.

```
$ docker compose restart api
$ docker compose exec api printenv LOG_LEVEL
info                       ← still the OLD value; your YAML edit was ignored
```

Root cause: `restart` **stops and starts the *same* container** with its *existing* config — it does **not** re-read the Compose file or recreate the container. Only `up` recomputes desired state and recreates containers when config changes. Wrong: `restart`. Right:
```bash
docker compose up -d api    # recreates api with the new YAML, applying the change
```
Mnemonic: **`restart` = same container, same config; `up` = apply the file.**

**Mistake 4 — Confusing `exec` with `run`.**

You want a shell in your running `api` and type:

```
$ docker compose run api sh
```

This *works* but it starts a **brand-new, extra** container (with a fresh, empty state), not the one that's already serving traffic. You end up debugging a different container than the one with the bug.

Root cause: `exec` runs a process **inside an already-running** container; `run` **creates a new one-off** container from the service definition. Wrong (to inspect the live container): `run`. Right:
```bash
docker compose exec api sh    # enter the ACTUAL running container
```
Use `run` for one-off tasks (migrations, a REPL) where a throwaway container is fine; use `exec` to inspect the live one.

**Mistake 5 — `--scale` on a service with a fixed host port.**

```
$ docker compose up -d --scale api=3
Error: driver failed programming external connectivity on endpoint orders-api-api-2:
  Bind for 0.0.0.0:3000 failed: port is already allocated
```

Root cause: `ports: ["3000:3000"]` pins the host side to 3000, and two containers can't both own host port 3000. Wrong: fixed host port + scale. Right: publish only the container port so Docker assigns a random host port per replica:
```yaml
ports:
  - "3000"        # host port auto-assigned (e.g. 32768, 32769, ...) → scaling works
```

---

## Hands-on proof

Run these **right now** to feel every command and see the safe-vs-destructive difference with your own data.

```bash
# --- Setup: a stack with a "database" volume that holds a marker file ---
mkdir -p /tmp/compose-cmds && cd /tmp/compose-cmds
cat > docker-compose.yml <<'EOF'
services:
  api:
    image: alpine
    command: sh -c "echo api up; sleep 600"
  db:
    image: alpine
    command: sh -c "sleep 600"
    volumes:
      - dbdata:/data
  tool:
    image: alpine
    command: sh -c "sleep 600"
    profiles: ["tools"]
volumes:
  dbdata:
EOF

# 1. See the FINAL merged config before running anything.
docker compose config

# 2. Start the default services (note: "tool" is profiled, should NOT start).
docker compose up -d
docker compose ps                 # you should see api + db, but NOT tool

# 3. Prove profiles: bring up the tools profile, now "tool" appears.
docker compose --profile tools up -d
docker compose ps                 # now api + db + tool

# 4. Write a marker into the db volume, then use exec to read it back.
docker compose exec db sh -c 'echo IMPORTANT_DATA > /data/marker.txt'
docker compose exec db cat /data/marker.txt      # → IMPORTANT_DATA

# 5. SAFE teardown: down WITHOUT -v keeps the volume.
docker compose down
docker compose up -d db
docker compose exec db cat /data/marker.txt      # → IMPORTANT_DATA  (survived!)

# 6. DESTRUCTIVE teardown: down WITH -v deletes the volume.
docker compose down -v
docker compose up -d db
docker compose exec db cat /data/marker.txt 2>&1 # → No such file  (data GONE)

# 7. logs and restart:
docker compose logs api                          # shows "api up"
docker compose restart api                       # same container, restarted
docker compose ps

# 8. Clean up everything.
docker compose down -v
cd / && rm -rf /tmp/compose-cmds
```

What you verified: `config` shows merged YAML (step 1); profiled services stay off until activated (steps 2–3); `exec` runs inside the live container (step 4); `down` keeps volumes but `down -v` destroys them (steps 5–6) — the single most important safety lesson in this topic.

---

## Practice exercises

### Exercise 1 — easy
With any two-service Compose file, run `docker compose up -d`, then `docker compose ps` and `docker compose ps -a`. Add a service to the file that immediately exits (`command: sh -c "exit 0"`), `up -d` again, and note which `ps` variant shows the exited container. Then run `docker compose logs` for both services and describe how output from multiple services is labelled.

### Exercise 2 — medium
Create a service `api` (image `node:20-alpine`, `command: sleep 600`) with `environment: LOG_LEVEL: info`. Bring it up. Now change `LOG_LEVEL` to `debug` in the YAML and run `docker compose restart api` — check `docker compose exec api printenv LOG_LEVEL` and explain why it's unchanged. Then run `docker compose up -d api` and check again. Write one sentence stating the exact difference between `restart` and `up`.

### Exercise 3 — hard (production simulation)
Model the "don't lose the database" incident. Build `api` + `postgres` (with a named volume for `/var/lib/postgresql/data`) + `redis`. Seed a table in Postgres via `docker compose exec postgres psql`. Now simulate a bad config: give `api` a wrong `REDIS_URL`, observe the crash loop with `docker compose ps` and `docker compose logs --tail 30 api`. Fix the URL in YAML and apply it to **only** `api` using `docker compose up -d --no-deps api`. Prove the Postgres table still exists afterward. Then explain what would have been lost if you'd instead run `docker compose down -v && docker compose up`.

---

## Mental model checkpoint

Answer these from memory:

1. Compose has no kernel powers of its own — so what does `docker compose up` actually *do* under the hood to start a container?
2. What is the exact difference between `docker compose down` and `docker compose down -v`, and which one can lose your database?
3. Why doesn't `docker compose up` (without `--build`) pick up your Dockerfile change?
4. `restart` vs `up` for one service: which one re-reads the YAML and recreates the container?
5. `exec` vs `run`: which one enters the already-running container, and which creates a new throwaway one?
6. Why does `--scale api=3` fail when `api` has `ports: ["3000:3000"]`, and how do you fix it?
7. A service has `profiles: ["tools"]`. Does a plain `docker compose up` start it? How do you start it?

---

## Quick reference card

| Command | What it does | Key detail / flag |
|---|---|---|
| `docker compose up` | Create network/volumes, (re)create & start all services in dep order | `-d` detach · `--build` rebuild · `--no-deps` skip dependencies · `--scale svc=N` |
| `docker compose down` | Stop & remove containers + project network | `-v` **also deletes named volumes (data loss)** · `--rmi local` · `--remove-orphans` |
| `docker compose build` | Build images without starting | `--no-cache` ignore layer cache (Topic 8) · name a service to build just it |
| `docker compose logs` | Show/stream service logs (json-file, Topic 17) | `-f` follow · `--tail N` · `--timestamps` · name a service to filter |
| `docker compose exec` | Run a command **inside a running** container | `-it` interactive · `-u` user · does **not** restart the app |
| `docker compose run` | Start a **new one-off** container from a service | For migrations/REPL; `--rm` to auto-remove |
| `docker compose ps` | List this project's containers + status + ports | `-a` include stopped/exited |
| `docker compose restart` | SIGTERM then start the **same** container | Does **not** re-read YAML — use `up` to apply edits |
| `docker compose config` | Print the final merged, substituted YAML | Best debug for overrides/env (Topic 21); `--services` lists names |
| `--profile NAME` | Activate optional profiled services | Profiled services stay off on a plain `up` |

---

## When would I use this at work?

1. **Daily inner loop on `orders-api`.** `docker compose up -d --build` in the morning, `docker compose logs -f api` in a side terminal while you code, `docker compose exec api sh` to run a migration or poke at env vars, `docker compose down` at night. This is your whole local workflow.

2. **Diagnosing a crash-looping service without data loss.** When a bad config makes `api` restart forever, you reach for `ps` (spot the loop) → `logs --tail` (read the error) → `config` (confirm the merged value) → `up -d --no-deps api` (fix just that service). You never touch `down -v`, so the local Postgres data is safe.

3. **Optional tooling and one-off jobs.** Keep `pgadmin`, `mailhog`, or a `seed` job behind `profiles:` so the default `up` is lean, and bring them in with `--profile tools` only when a task needs them — or fire a one-shot migration with `docker compose run --rm migrate`.

---

## Connected topics

- **Study before:**
  - **Topic 16 — Container Lifecycle**: what `stop`/`start`/`restart` mean at the SIGTERM→SIGKILL level, which `down`/`restart` build on.
  - **Topic 17 — Logging**: the `json-file` logs that `docker compose logs` reads.
  - **Topic 18 — Docker Compose in Depth** and **Topic 19 — Compose File Deep Dive**: the file these commands operate on.
  - **Topic 21 — Compose for Development**: the override-merge behavior that `config` reveals and `up` applies.
- **Study after:**
  - **Topic 20 — Multi-Container Apps**: `depends_on`/healthcheck ordering that `up` obeys (revisit with the command lens).
  - **Topic 30 — The Reconciliation Loop**: `up` is a tiny reconciliation loop; this is the same idea at cluster scale.
  - **Topic 31 — kubectl Basics**: the Kubernetes equivalents (`apply`, `get`, `logs`, `exec`) of these very commands.
  - **Topic 35 — Services**: real load balancing across replicas, which `--scale` only hints at locally.
