# 19 — Compose File Deep Dive

## Section: Docker in Practice

---

## ELI5 — The Simple Analogy

Think of a `docker-compose.yml` as a **detailed order form for building a LEGO city**.

Each **service** is one building. On the order form for each building you fill in little boxes:

- Which pre-made LEGO kit to use (`image`) — or "build it from these instructions" (`build`).
- Which doors face the street so visitors can walk in (`ports`).
- A locked storage room that keeps its contents even if the building is knocked down and rebuilt (`volumes`).
- Sticky notes taped inside telling the building how to behave (`environment`, `env_file`).
- "Don't open this building until the power plant next door is running" (`depends_on`).
- A doctor who checks every few seconds whether the building is actually healthy (`healthcheck`).
- "If this building falls over, put it right back up" (`restart`).
- Which roads connect it to other buildings (`networks`).
- What the very first worker should start doing when the doors open (`command`).

In Topic 18 you learned the *shape* of the order form. This topic fills in **every box on it**, one at a time, so nothing is a mystery.

---

## The Linux kernel feature underneath

Each Compose field is a *friendly name for a kernel or daemon action* you already met. There is no new magic — Compose is a translator. Here is the mapping, field → kernel/daemon reality:

```
Compose field        Translates to `docker run` flag   Kernel / daemon mechanism
─────────────        ──────────────────────────────    ─────────────────────────
image:               (the image argument)              overlay2 layers mounted read-only + a writable upper (Topic 04)
build:               `docker build` then run           BuildKit assembles layers, then same as image
ports:               -p host:container                 iptables DNAT rule + userland proxy (Topic 13)
volumes:             -v name:/path  /  -v /host:/path  mount namespace bind/overlay mount (Topic 14)
environment:         -e KEY=VAL                        strings placed in the new process's environ[] (Topic 15)
env_file:            --env-file                        same, read from a file first
depends_on:          (no direct flag — Compose logic)  Compose orders the create/start API calls
healthcheck:         --health-cmd / HEALTHCHECK        daemon runs a command in the container's namespaces
restart:             --restart                         daemon (containerd) watches PID exit, re-invokes runc
networks:            --network                         veth pair + Linux bridge + embedded DNS (Topic 13)
command:             (the CMD override)                argv passed to execve() as PID 1 in the container
```

Two of these are worth pausing on because they are pure Compose behavior, not a kernel primitive:

- **`depends_on`** is not a kernel thing at all. Compose builds a directed graph of services and topologically sorts the *order in which it makes the daemon's create/start calls*. The kernel never hears the word "depends."
- **`healthcheck`** is executed by the **Docker daemon**, which periodically runs your check command *inside the container's namespaces* (it enters the container's PID/mount/net namespaces to run e.g. `pg_isready`). The result flips a `Health.Status` field the daemon stores in memory: `starting` → `healthy` / `unhealthy`. Compose then reads that status to satisfy `condition: service_healthy` (Topic 20).

So the deep truth: **Compose fields are a declarative front-end over `docker run` flags and a small amount of ordering logic. Everything ultimately becomes namespaces, cgroups, overlay mounts, and iptables rules.**

---

## What is this?

This topic is a field-by-field reference for the Compose file. For each important service field — `image`, `build`, `ports`, `volumes`, `environment`, `env_file`, `depends_on`, `healthcheck`, `restart`, `networks`, and `command` — you get the exact syntax, every form it can take, an annotated `│`-pointer breakdown, and the `docker run` equivalent so you always know what it *really* does.

By the end you can read any Compose file in the wild and know precisely what container it produces.

---

## Why does it matter for a backend developer?

A Compose file *looks* simple, so people copy-paste fields they don't understand — and then get bitten:

- They write `ports: 3000:3000` and can't figure out which number is the host and which is the container.
- They use a bind mount `- ./data:/var/lib/postgresql/data` and wonder why Postgres complains about permissions, when a named volume would have just worked.
- They set `environment` *and* `env_file` with the same key and can't explain which one wins.
- They add a `healthcheck` that always reports healthy because the check command's exit code is being swallowed.
- They set `restart: always` on a one-shot migration container and it loops forever.

Every one of those is a field-semantics misunderstanding. Knowing each field cold is the difference between Compose being a reliable tool and a source of "why is it doing that?" evenings.

---

## The physical reality

When Compose applies these fields, here's what each one leaves on your machine — inspectable, right now:

- **`image`/`build`** → a read-only image in `/var/lib/docker/overlay2/` (Topic 04). `docker image inspect orders-api:latest` shows its layer digests.
- **`ports`** → a rule you can see in the host's NAT table: `sudo iptables -t nat -L DOCKER -n` shows `DNAT tcp dpt:3000 to:172.18.0.4:3000`. Also a `docker-proxy` userland process listening on the host port.
- **`volumes` (named)** → a directory at `/var/lib/docker/volumes/<project>_<name>/_data`. **(bind)** → a direct mount of a host path into the container's mount namespace; `cat /proc/<pid>/mountinfo` shows it.
- **`environment`/`env_file`** → the actual strings live in the container process's environment. `docker exec <c> env` prints them; `cat /proc/<pid>/environ | tr '\0' '\n'` shows them raw from the kernel's view.
- **`healthcheck`** → `docker inspect <c> --format '{{json .State.Health}}'` shows `Status`, `FailingStreak`, and the log of the last few probe runs (with exit codes and output).
- **`restart`** → `docker inspect <c> --format '{{.HostConfig.RestartPolicy.Name}}'` and `{{.RestartCount}}` show the policy and how many times the daemon has restarted it.
- **`networks`** → `docker network inspect <net>` lists connected containers, their IPs, and aliases; on the host `ip link` shows the `veth*` interfaces.
- **`command`** → `docker inspect <c> --format '{{json .Config.Cmd}}'` shows the argv PID 1 was started with.

None of it is hidden. Every field is observable state on disk, in the kernel, or in the daemon.

---

## How it works — step by step

Take one fully-loaded service and trace how Compose turns its fields into a running container.

```yaml
services:
  orders-api:
    build: .
    command: ["node", "dist/server.js"]
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
    env_file:
      - .env
    volumes:
      - ./logs:/app/logs
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 10s
      timeout: 3s
      retries: 3
    restart: unless-stopped
    networks:
      - backend
```

1. **`build: .`** → Compose invokes BuildKit on `./Dockerfile`, produces an image, tags it `<project>-orders-api`.
2. **`env_file: [.env]` then `environment:`** → Compose reads `.env` into a key/value map, then overlays the inline `environment` map on top (inline **wins** on conflicts). The merged map is the container's env.
3. **`depends_on: postgres: condition: service_healthy`** → Compose will not *create/start* this container until the `postgres` service's health status is `healthy` (Topic 20). It waits.
4. **create call** → Compose calls the daemon's container-create API with: the built image, `Cmd = ["node","dist/server.js"]` (from `command`), the merged env, the port binding `3000→3000`, a bind mount of `./logs` → `/app/logs`, attachment to network `backend`, restart policy `unless-stopped`, and the healthcheck spec.
5. **daemon → containerd → runc** (Topic 03) → `runc` calls `clone()` with namespace flags, sets up the overlay mount, applies the bind mount into the mount namespace, wires a veth pair into the `backend` bridge, then `execve("node", ["node","dist/server.js"])` as PID 1.
6. **healthcheck loop starts** → the daemon waits (no `start_period` here, so immediately) then every `interval` (10s) runs `wget -qO- http://localhost:3000/health` *inside* the container. Exit 0 → healthy; non-zero → after `retries` consecutive failures → unhealthy.
7. **restart watch** → containerd watches PID 1. If it exits and the policy (`unless-stopped`) says so, the daemon restarts it and bumps `RestartCount`.

That's the whole pipeline: fields → merged config → one create call → runc → live container + daemon-side health/restart supervision.

---

## Exact syntax breakdown

Now, field by field. Each shows every common form and the `docker run` equivalent.

### `image`

```
    image: postgres:16
    │      │        └── TAG. Version selector. Omit → defaults to `latest` (do not rely on it).
    │      └─────────── REPOSITORY. `postgres` = library/postgres on Docker Hub.
    └────────────────── field: run this service FROM this pre-built image.
```
Full form with a registry: `image: ghcr.io/acme/orders-api:1.4.2` → `registry/namespace/repo:tag`. Equivalent to the image argument of `docker run`.

### `build`

```
    build: .
    │      └── CONTEXT: directory sent to the daemon as build context; Dockerfile expected here.
    └───────── field: build the image locally instead of pulling one.
```
Long form, when you need more control:
```
    build:
      context: ./api        # ← folder to build from (the build context, Topic 10)
      dockerfile: Dockerfile.prod   # ← non-default Dockerfile name
      args:                 # ← build-time ARG values (Topic 06)
        NODE_VERSION: "20"
      target: runtime       # ← which multi-stage stage to stop at (Topic 09)
```
If you set **both** `build:` and `image:`, Compose builds the image and *tags it with the `image:` name* — handy for building then pushing (Topic 24).

### `ports`

```
    ports:
      - "3000:3000"
      │  │    └── CONTAINER port: what the app listens on inside.
      │  └─────── HOST port: what you hit from your laptop. LEFT = host, RIGHT = container.
      └────────── quoted string. Quote it! Unquoted "5432:5432" is fine but "22:22" can be
                  misread by YAML as a base-60 number. Always quote to be safe.
```
Other valid forms:
```
      - "127.0.0.1:5432:5432"   # bind only to loopback — DB reachable from host tools, not the network
      - "8080:80"               # different host/container numbers
      - "3000"                  # only container port → host port chosen RANDOMLY (rarely what you want)
      - "9000-9005:9000-9005"   # a port range
```
Equivalent to `docker run -p`. **Remember: `HOST:CONTAINER`.** The one you type in your browser is the left one.

### `volumes`

```
    volumes:
      - pgdata:/var/lib/postgresql/data
      │  │      └── mount point INSIDE the container.
      │  └───────── NAMED VOLUME (no leading / or .). Managed by Docker, lives under /var/lib/docker/volumes.
      └──────────── list item.
```
The three forms, each with a different physical meaning (Topic 14):
```
      - pgdata:/var/lib/postgresql/data     # NAMED volume  → Docker-managed, survives, correct perms
      - ./logs:/app/logs                    # BIND mount    → a host path (starts with . or /), live-synced
      - /app/node_modules                   # ANONYMOUS vol → hides a host path behind a fresh volume
```
Long form with options:
```
      - type: volume
        source: pgdata
        target: /var/lib/postgresql/data
        read_only: false
```
Equivalent to `docker run -v` / `--mount`. Rule of thumb: **named volume for databases, bind mount for source code you're editing (Topic 21).**

### `environment`

```
    environment:
      NODE_ENV: production
      │         └── VALUE.
      └──────────── KEY. Injected into the process's environ[]. Read in Node via process.env.NODE_ENV.
```
Two equivalent syntaxes — map form (above) and list form:
```
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://orders_user:orders_pass@postgres:5432/orders
      - REDIS_URL          # ← no value → PASS THROUGH the value from YOUR shell/.env
```
That last form (`- REDIS_URL` with no `=value`) means "take whatever `REDIS_URL` is in the host environment." Equivalent to `docker run -e`.

### `env_file`

```
    env_file:
      - .env
      │  └── path to a file of KEY=VALUE lines (one per line, # for comments).
      └───── list of files, applied top to bottom.
```
Example `.env` contents:
```
DATABASE_URL=postgres://orders_user:orders_pass@postgres:5432/orders
REDIS_URL=redis://redis:6379
```
**Precedence (memorize this):** for the same key, `environment:` **overrides** `env_file:`. And Compose's own `${VAR}` interpolation in the YAML uses the *directory's `.env` file* (a separate, special use of `.env`) — don't confuse the two. Equivalent to `docker run --env-file`.

### `depends_on`

Short (list) form — orders *start* only:
```
    depends_on:
      - postgres          # start postgres before this service (does NOT wait for readiness)
      - redis
```
Long (map) form — orders start AND waits for a condition:
```
    depends_on:
      postgres:
        condition: service_healthy   # wait until postgres's healthcheck passes (Topic 20)
      redis:
        condition: service_started    # default: just wait for the container to start
```
Conditions: `service_started` (started), `service_healthy` (passing healthcheck), `service_completed_successfully` (exited 0 — great for migration/init containers). There is **no `docker run` equivalent** — this is pure Compose ordering logic.

### `healthcheck`

```
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "orders_user"]
      │      │      └── the command + args. Exit 0 = healthy, non-zero = unhealthy.
      │      └───────── "CMD" = run directly (no shell). "CMD-SHELL" = run via /bin/sh -c "…".
      └──────────────── the probe command.
      interval: 10s     # ← how often to run the check (after start_period)
      timeout: 5s       # ← kill the check + count as fail if it runs longer than this
      retries: 5        # ← consecutive failures needed to flip status to "unhealthy"
      start_period: 30s # ← grace window at boot: failures during it don't count against retries
```
`CMD` vs `CMD-SHELL`:
```
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]        # exec form, no shell
      test: ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]  # shell form, pipes/|| work
```
Equivalent to `docker run --health-cmd/--health-interval/...` or a `HEALTHCHECK` in the Dockerfile (Topic 26). The daemon runs the probe *inside the container's namespaces*.

### `restart`

```
    restart: unless-stopped
    │        └── POLICY. What the daemon does when PID 1 exits.
    └─────────── field.
```
The four policies:
```
      no              # (default) never restart automatically
      always          # always restart; also starts on daemon/host boot
      on-failure      # restart only if exit code != 0; add :N to cap attempts, e.g. on-failure:5
      unless-stopped  # like always, but if YOU stopped it, stay stopped across reboots
```
Equivalent to `docker run --restart`. For a long-running API/DB use `unless-stopped`. For a one-shot migration job use `no` (or `on-failure` with a cap) — never `always`, or it loops forever.

### `networks`

```
    networks:
      - backend
      │  └── name of a network declared in the top-level `networks:` block.
      └───── attach this service to that network. Omit → attaches to <project>_default.
```
Long form with a fixed alias / static IP:
```
    networks:
      backend:
        aliases:
          - orders            # extra DNS name: other services can also reach this as "orders"
        ipv4_address: 172.28.1.10   # static IP (requires the network to define a subnet)
```
Equivalent to `docker run --network` (+ `--network-alias`). A service can join multiple networks — useful to isolate a database on a `backend` net while a proxy sits on both `backend` and an `edge` net.

### `command`

```
    command: ["node", "dist/server.js"]
    │         └── EXEC form (JSON array): argv passed straight to execve, no shell. PREFERRED.
    └──────────── overrides the image's default CMD (Topic 07).
```
Shell form (runs via `/bin/sh -c`, so pipes/`&&`/`$VARS` work but signals behave worse):
```
    command: node dist/server.js && echo started   # shell form
```
Related field `entrypoint:` overrides the image's ENTRYPOINT the same way (Topic 07). `command:` = the `CMD` you'd append to `docker run <image> <command>`.

---

## Example 1 — basic

A small but *complete* service showing the everyday fields together, each line commented.

```yaml
# docker-compose.yml
services:
  orders-api:
    image: orders-api:1.0.0          # run a specific, tagged image (not :latest)
    command: ["node", "server.js"]   # override CMD: start the server explicitly
    ports:
      - "3000:3000"                  # host 3000 → container 3000; open http://localhost:3000
    environment:
      NODE_ENV: development          # inline env var (highest precedence)
    env_file:
      - .env                         # bulk-load DATABASE_URL, REDIS_URL, etc. from a file
    restart: on-failure              # if the process crashes (exit != 0), restart it
```

What a reader learns at a glance: it runs a pinned image, starts `node server.js`, is reachable on 3000, gets `NODE_ENV=development` plus whatever's in `.env` (with the inline value winning any clash), and self-heals on crashes but not on clean exits.

---

## Example 2 — production scenario

Real scenario: your `orders-api` kept starting *before* Postgres finished initializing, throwing `ECONNREFUSED` and getting restart-looped in a way that spammed logs and occasionally corrupted a half-run migration. The fix is to use `healthcheck` on Postgres + `depends_on: condition: service_healthy` on the API, plus sensible restart and network isolation. Here is the battle-tested service definition using every field from this topic.

```yaml
# docker-compose.yml — production-shaped orders-api service
services:
  orders-api:
    build:
      context: .
      dockerfile: Dockerfile           # multi-stage prod Dockerfile (Topic 11)
      target: runtime                  # stop at the slim runtime stage (Topic 09)
    image: ghcr.io/acme/orders-api:1.4.2   # build AND tag with this name (for later push)
    command: ["node", "dist/server.js"]    # explicit start command
    ports:
      - "127.0.0.1:3000:3000"          # only expose on loopback; a real LB fronts it in prod
    environment:
      NODE_ENV: production             # inline; overrides any NODE_ENV in env_file
    env_file:
      - .env.production                # DATABASE_URL, REDIS_URL, secrets injected here
    volumes:
      - ./logs:/app/logs               # bind mount so log files land on the host for shipping
    depends_on:
      postgres:
        condition: service_healthy     # wait until Postgres passes its healthcheck
      redis:
        condition: service_started     # Redis is fast; just wait for it to start
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]  # app's own health route
      interval: 15s
      timeout: 3s
      retries: 3
      start_period: 20s                # give Node time to boot before counting failures
    restart: unless-stopped            # keep it up, but respect a manual stop across reboots
    networks:
      - backend                        # same private net as postgres + redis

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: orders_user
      POSTGRES_PASSWORD: orders_pass
      POSTGRES_DB: orders
    volumes:
      - pgdata:/var/lib/postgresql/data   # named volume: data survives recreation
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orders_user -d orders"]  # is Postgres accepting queries?
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s
    restart: unless-stopped
    networks:
      - backend

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    networks:
      - backend

networks:
  backend:
    driver: bridge

volumes:
  pgdata:
```

Why the field choices matter here:

- **`depends_on` + `postgres.healthcheck`** — this is the whole reason the restart loop stops. `condition: service_healthy` blocks `orders-api` creation until `pg_isready` succeeds. Topic 20 explains this pairing in full.
- **`start_period`** on both — without it, the *first* few probes (while the app/DB is still booting) would count as failures and prematurely flip status to unhealthy. The grace window prevents false alarms.
- **`CMD-SHELL` for `pg_isready`** — chosen so we could add `|| exit 1` style logic if needed; `CMD` (exec form) is used for the API's `wget` because no shell features are needed and exec form is lighter.
- **`restart: unless-stopped`** for all three long-lived services; if any crashes, it comes back, but `docker compose stop` still means stop.
- **`127.0.0.1:3000:3000`** — in production the API sits behind a load balancer / reverse proxy on the host; binding to loopback stops the container port from being exposed on all host interfaces.

---

## Common mistakes

### Mistake 1 — Getting `ports` backwards (`container:host`)

```yaml
    ports:
      - "3000:8080"    # you MEANT host 8080 → container 3000, but wrote it reversed
```
**Symptom:** the app listens on 3000 inside, but you published host 3000 → container 8080, where nothing listens. `curl localhost:3000` hangs/refuses.
**Root cause:** the mapping is `HOST:CONTAINER`. You mapped host 3000 to a dead container port.
**Right:** `"8080:3000"` — host port on the left, the port your app actually listens on inside the container on the right.

### Mistake 2 — `environment` vs `env_file` precedence surprise

```yaml
    env_file:
      - .env             # contains NODE_ENV=production
    environment:
      NODE_ENV: development
```
**Symptom:** you set `production` in `.env` but the app runs in `development`.
**Root cause:** inline `environment:` **overrides** `env_file:` for the same key. That's the defined precedence, not a bug.
**Right:** decide which layer owns a variable. Put shared bulk config in `env_file`, put per-environment overrides in `environment`, and don't set the same key in both unless you *want* the inline one to win.

### Mistake 3 — A healthcheck that's always "healthy"

```yaml
    healthcheck:
      test: ["CMD-SHELL", "curl http://localhost:3000/health"]   # no -f !
```
**Symptom:** `docker inspect` shows `healthy` even when the endpoint returns HTTP 500.
**Root cause:** plain `curl` exits 0 for *any* completed HTTP request, including 4xx/5xx. Only the exit code matters to Docker, and it's 0. So the check never fails.
**Right:** add `-f`/`--fail` so curl exits non-zero on HTTP errors: `curl -f http://localhost:3000/health`. (Or `wget -qO- --spider` / a script that checks the status code.) Verify with `docker inspect --format '{{json .State.Health}}'`.

### Mistake 4 — `restart: always` on a one-shot task

```yaml
  migrate:
    image: orders-api:1.0.0
    command: ["node", "migrate.js"]
    restart: always                # ← WRONG for a task that's meant to run once and exit
```
**Symptom:** the migration runs, exits 0, and Docker immediately restarts it — forever. `docker ps` shows it constantly recreating; migrations re-run in a loop.
**Root cause:** `always` restarts on *any* exit, including a successful `exit 0`.
**Right:** use `restart: "no"` (the default) for one-shot jobs. If other services must wait for it, use `depends_on: { migrate: { condition: service_completed_successfully } }`.

### Mistake 5 — Bind-mounting over a directory the image needs (the `node_modules` trap)

```yaml
    volumes:
      - ./:/app             # mounts your host project over /app, HIDING the image's /app/node_modules
```
**Error:**
```
Error: Cannot find module 'express'
```
**Root cause:** the bind mount replaces `/app` inside the container with your host folder, which has no `node_modules` (they were installed into the image at build time, now hidden).
**Right:** add an anonymous volume to "mask" node_modules so the image's copy stays visible:
```yaml
    volumes:
      - ./:/app
      - /app/node_modules   # keeps the image's node_modules, not the host's
```
Full dev hot-reload setup is Topic 21.

---

## Hands-on proof

See each field become real state on your machine.

```bash
mkdir -p /tmp/compose-fields && cd /tmp/compose-fields
cat > .env <<'EOF'
GREETING=from-env-file
EOF
cat > docker-compose.yml <<'EOF'
services:
  web:
    image: nginx:alpine
    ports:
      - "127.0.0.1:8088:80"
    environment:
      GREETING: from-inline           # will OVERRIDE the env_file value
    env_file:
      - .env
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:80/"]
      interval: 5s
      timeout: 2s
      retries: 3
    restart: unless-stopped
EOF

docker compose up -d

# PROOF: environment overrides env_file (should print "from-inline", NOT "from-env-file")
docker compose exec web printenv GREETING

# PROOF: the port binding exists in the kernel's NAT table
sudo iptables -t nat -L DOCKER -n | grep 8088 || echo "(rootless/desktop: check 'docker compose port web 80')"
docker compose port web 80        # → 127.0.0.1:8088

# PROOF: the daemon is running the healthcheck; watch status go starting → healthy
docker inspect compose-fields-web-1 --format '{{.State.Health.Status}}'
sleep 8
docker inspect compose-fields-web-1 --format '{{json .State.Health}}' | tr ',' '\n' | head

# PROOF: the restart policy is recorded on the container
docker inspect compose-fields-web-1 --format 'policy={{.HostConfig.RestartPolicy.Name}} count={{.RestartCount}}'

docker compose down
```

If `printenv GREETING` prints `from-inline`, you've proven the `environment` > `env_file` precedence rule with your own eyes.

---

## Practice exercises

### Exercise 1 — easy

Write a single-service Compose file for `redis:7-alpine` named `cache`. Publish it on host `6390` → container `6379`, add `restart: unless-stopped`, and add a `healthcheck` that runs `["CMD", "redis-cli", "ping"]` every 5 seconds. Bring it up and confirm with `docker inspect` that `.State.Health.Status` becomes `healthy`.

### Exercise 2 — medium

Create a service `orders-api` (use `nginx:alpine` as a stand-in) that reads a variable `APP_MODE` from **both** an `env_file` (set it to `file-value`) and an inline `environment` (set it to `inline-value`). Bring it up, run `docker compose exec orders-api printenv APP_MODE`, and confirm which value wins. Then delete the inline `environment` line, run `docker compose up -d` again, and confirm the value changed — explaining why recreating the container was necessary.

### Exercise 3 — hard (production simulation)

Reproduce the Example 2 stack (`orders-api` + `postgres` + `redis`) but *deliberately break* the Postgres healthcheck by using `pg_isready -U wrong_user`. Bring it up and observe: (a) Postgres reports `unhealthy`, (b) `orders-api` never starts because its `depends_on: condition: service_healthy` is never satisfied — confirm with `docker compose ps` that `orders-api` is stuck in `Created`/waiting. Then fix the user, `up` again, and confirm `orders-api` now starts once Postgres is healthy. Write two sentences on how the daemon's stored `Health.Status` drives Compose's start decision.

---

## Mental model checkpoint

1. In `ports: "8080:3000"`, which number do you type in the browser, and which one does the app listen on inside?
2. If a key is set in both `env_file` and `environment`, which value wins?
3. What's the difference between `depends_on` short (list) form and long (map) form?
4. Why can a `healthcheck` using plain `curl` (no `-f`) report healthy on an HTTP 500?
5. Which `restart` policy is correct for a long-running API, and which for a one-shot migration?
6. What are the three forms of a `volumes:` entry, and how do you tell a named volume from a bind mount by looking at it?
7. What does the daemon actually do to run a `healthcheck`, and where is the result stored?

---

## Quick reference card

| Field | What it does | Key detail |
|---|---|---|
| `image` | Run the service from a pre-built image | `repo:tag`; pin the tag, avoid `latest` |
| `build` | Build the image locally from a Dockerfile | `context`, `dockerfile`, `target`, `args`; can combine with `image` to tag |
| `ports` | Publish container port to the host | `"HOST:CONTAINER"`; quote it; prefix `127.0.0.1:` to limit exposure |
| `volumes` | Mount data into the container | named (persist) vs bind (`./` or `/`) vs anonymous |
| `environment` | Inline env vars | **Overrides** `env_file` on key clashes |
| `env_file` | Load env vars from a file | KEY=VALUE lines; applied before `environment` |
| `depends_on` | Start ordering / readiness gating | list = start only; map = wait for `condition` |
| `healthcheck` | Daemon probes container health | exit 0 = healthy; `interval/timeout/retries/start_period` |
| `restart` | Auto-restart policy on exit | `no`/`always`/`on-failure[:N]`/`unless-stopped` |
| `networks` | Which networks to join | omit → `<project>_default`; supports `aliases` |
| `command` | Override the image's CMD | exec form `["a","b"]` preferred over shell form |

---

## When would I use this at work?

1. **Reading an unfamiliar repo's `docker-compose.yml`.** You join a team, open their Compose file, and immediately understand the whole stack: which image, which ports are exposed, where data persists, what the startup order and health gates are. No archaeology.

2. **Debugging "works on my machine."** A teammate's container behaves differently. You compare `environment`/`env_file` precedence, check whether they used a bind mount vs named volume, and inspect the `healthcheck`/`restart` fields — the field semantics you know here are exactly the knobs that cause such divergence.

3. **Hardening a dev Compose file toward production.** You pin `image` tags, switch DB storage to a named volume, add `healthcheck` + `depends_on: service_healthy`, set `restart: unless-stopped`, and bind ports to loopback. Every one of those is a field from this topic, applied deliberately.

---

## Connected topics

- **Study before:** Topic 18 (Docker Compose in Depth — the file's shape and the three top-level keys), Topic 12 (docker run flags each field maps to), Topic 14 (Volumes), Topic 15 (Environment Variables), Topic 07 (CMD vs ENTRYPOINT — `command`/`entrypoint`).
- **Study after:** Topic 20 (Multi-Container Apps — `depends_on` + `healthcheck` in action, service discovery), Topic 21 (Compose for Development — bind mounts and overrides), Topic 22 (Compose Commands), Topic 26 (Health Checks — the daemon side in depth). Kubernetes later mirrors many of these fields: probes (Topic 43), env/config (Topic 36), restart/QoS (Topic 44).
