# 20 — Multi-Container Apps

## Section: Docker in Practice

---

## ELI5 — The Simple Analogy

Imagine three friends who need to work together in a big office building:

- **Ollie** (the `orders-api`) takes phone calls from customers.
- **Priya** (Postgres) keeps the big permanent filing cabinet of everything that ever happened.
- **Reza** (Redis) has a tiny whiteboard for things that need to be grabbed *fast* but don't need to last.

For them to work together, three problems must be solved:

1. **How do they find each other?** In a huge building you don't memorize desk coordinates — you use the internal **phone directory**: "connect me to Priya." You say the *name*, the building's operator connects you. In Docker, the building's operator is an **embedded DNS server**, and the names are the **service names**.

2. **What if Ollie starts taking calls before Priya has even unlocked the filing cabinet?** He'd promise customers things and then find no cabinet to write in — chaos. So we make a rule: *Ollie doesn't start work until Priya signals "cabinet open, ready."* That signal is a **healthcheck**, and the rule is `depends_on: condition: service_healthy`.

3. **Who's allowed to talk to whom?** They share a private office (the Compose **network**) that outsiders can't walk into.

This whole topic is: three containers, finding each other by name, starting in the right order, on one private network. That's a multi-container app.

---

## The Linux kernel feature underneath

Two mechanisms make a multi-container app work. Neither is new — they're Topic 13's networking plus a DNS server the daemon runs.

### 1. Service discovery = an embedded DNS server + a Linux bridge

When Compose creates a network (Topic 18), the Docker daemon does two kernel-level things and one daemon-level thing:

```
                     Docker embedded DNS server
                     listens at 127.0.0.11:53
                     INSIDE every container on the network
                              ▲
                              │ "what IP is 'postgres'?"  → 172.18.0.3
                              │
  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
  │  orders-api   │     │   postgres    │     │    redis      │
  │ 172.18.0.4    │     │ 172.18.0.3    │     │ 172.18.0.2    │
  │  veth ────────┼─────┤  veth ────────┼─────┤  veth         │
  └───────┬───────┘     └───────┬───────┘     └───────┬───────┘
          │                     │                     │
          └─────────────────────┴─────────────────────┘
                     Linux bridge  br-<netid>  (a virtual switch)
                     iptables rules allow same-network traffic
```

- **veth pairs (kernel):** each container gets one end of a virtual ethernet cable; the other end plugs into the bridge. This is a `network namespace` (Topic 02) with its own interface.
- **Linux bridge (kernel):** `br-<netid>` is a software switch. All containers on the same Compose network are plugged into it, so packets can flow between their IPs.
- **Embedded DNS (daemon):** here's the key part. Inside each container, `/etc/resolv.conf` is set to `nameserver 127.0.0.11`. That address is a tiny DNS server the Docker daemon runs *per network*. When `orders-api` looks up the name `postgres`, the query goes to `127.0.0.11:53`; the daemon answers with Postgres's current container IP on that network. **That is why you connect to `postgres`, not `172.18.0.3` — the name is stable, the IP is not.**

Prove `127.0.0.11` is real:
```bash
docker compose exec orders-api cat /etc/resolv.conf
# nameserver 127.0.0.11
```

### 2. Startup ordering = Compose logic + the daemon's health status field

There is **no kernel feature** for "start B after A is ready." This is entirely:

- **Compose** builds a dependency graph from `depends_on` and decides the order of its create/start API calls.
- **The Docker daemon** runs each service's `healthcheck` command *inside that container's namespaces* and stores the result in an in-memory field: `.State.Health.Status` = `starting` → `healthy` / `unhealthy`.
- **Compose polls that field.** For `depends_on: { postgres: { condition: service_healthy } }`, Compose will not issue the "create orders-api" call until the daemon reports Postgres's `Health.Status == healthy`.

So the "wait for ready" magic is: a command you specify, run by the daemon, turning a status field green, that Compose watches before proceeding. Simple parts, powerful result.

---

## What is this?

A multi-container app is several containers that together form one application, wired onto a shared private network so they can find each other by **service name** (via the embedded DNS), and started in a **dependency-aware order** so services don't try to use dependencies that aren't ready yet. Our canonical example is `orders-api` (Node/Express) + `postgres` (durable store) + `redis` (cache), defined in one `docker-compose.yml`.

---

## Why does it matter for a backend developer?

Because *every* real backend is a multi-container app. An API alone does nothing — it needs a database, usually a cache, often a queue. The two things that break constantly when people wire these up are:

1. **"It can't find the database."** Someone hardcodes `localhost:5432` or an IP address. Inside a container, `localhost` is *that container itself* — Postgres isn't there. And IPs change every restart. If you don't understand service-name DNS, you'll fight this repeatedly.

2. **"It crashes on startup, then works after I restart it."** The API booted before Postgres was accepting connections, threw `ECONNREFUSED`, and died. The "restart fixes it" is just Postgres having finished initializing by the second try. If you don't understand `healthcheck` + `condition: service_healthy`, your stack is a coin flip at boot and your CI is flaky.

Master these two — name-based discovery and health-gated startup — and multi-container apps go from fragile to boring-reliable. Boring is the goal.

---

## The physical reality

With the `orders-api` stack up, here's what exists and where to see it:

- **One network** at the daemon level: `docker network inspect orders-api_backend` lists all three containers, each with an IPv4 address and DNS aliases (the service name, plus the container name).
- **`/etc/resolv.conf` inside each container** points at `127.0.0.11` — the embedded DNS.
- **A bridge interface on the host**: `ip -br link | grep br-` shows `br-<netid>`; `ip -br addr` shows its gateway IP (e.g. `172.18.0.1`), which is also the containers' default gateway.
- **veth interfaces on the host**: `ip -br link | grep veth` shows one veth per container; its peer lives in the container's netns.
- **Health status in the daemon**: `docker inspect orders-api-postgres-1 --format '{{json .State.Health}}'` shows `Status`, `FailingStreak`, and a `Log` array of the last few probe runs with their exit codes and stdout.
- **A named volume** for Postgres data at `/var/lib/docker/volumes/orders-api_pgdata/_data`, so the database survives recreation.

Everything is inspectable. There is no hidden coordination service — just a bridge, a per-network DNS resolver, and a status field.

---

## How it works — step by step

Trace `docker compose up` for the full three-service stack, from command to serving traffic.

1. **Parse + graph.** Compose reads the YAML, builds the dependency graph: `postgres` and `redis` have no deps; `orders-api depends_on postgres (service_healthy)` and `redis (service_started)`.
2. **Create the network.** Compose creates `orders-api_backend` (a bridge). The daemon allocates a subnet (e.g. `172.18.0.0/16`), sets up `br-<netid>`, and starts the embedded DNS for this network.
3. **Create the volume.** `orders-api_pgdata` is created if absent (data preserved if it already exists).
4. **Start the leaf dependencies first.** Compose creates and starts `postgres` and `redis` (they have no deps). Each gets a veth into the bridge, an IP, and DNS registration under its service name.
5. **Postgres begins initializing.** Its entrypoint creates the data dir, runs init, and eventually starts accepting connections on 5432. Meanwhile the daemon runs its `healthcheck` (`pg_isready`) every `interval`. Early probes fail (still starting) but fall within `start_period`, so they don't count. Once `pg_isready` exits 0, the daemon flips `postgres` `Health.Status` to `healthy`.
6. **Compose waits.** It's been polling Postgres's health. `redis` reached `service_started` quickly. Once Postgres is `healthy`, both `depends_on` conditions for `orders-api` are satisfied.
7. **Create and start `orders-api`.** It gets a veth/IP/DNS name on `backend`. Its env has `DATABASE_URL=postgres://…@postgres:5432/orders` and `REDIS_URL=redis://redis:6379`.
8. **The app resolves names.** Node's pg driver connects to host `postgres`. The lookup hits `127.0.0.11`, which returns Postgres's IP. TCP connects over the bridge. Same for `redis`. **No IPs anywhere in your config.**
9. **The API's own healthcheck** starts; once `/health` returns 200, `orders-api` is `healthy` too. The stack is fully up and serving on `localhost:3000`.

If Postgres restarts later and gets a *new* IP, DNS re-registration means the name `postgres` now resolves to the new IP automatically — as long as your app reconnects (which is why apps should retry connections, not just rely on startup order).

---

## Exact syntax breakdown

### The full stack file — annotated

```yaml
# docker-compose.yml — orders-api + postgres + redis
services:
  orders-api:
    build: .                                   # build image from ./Dockerfile (Topics 09, 11)
    ports:
      - "3000:3000"                            # only the API is reachable from the host
    environment:
      DATABASE_URL: postgres://orders_user:orders_pass@postgres:5432/orders
      #                                                  └── HOSTNAME = the service name "postgres"
      REDIS_URL: redis://redis:6379            #          └── HOSTNAME = the service name "redis"
      NODE_ENV: production
    depends_on:
      postgres:
        condition: service_healthy             # wait until Postgres passes pg_isready
      redis:
        condition: service_started             # Redis boots instantly; just wait for start
    networks:
      - backend
    restart: unless-stopped

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: orders_user
      POSTGRES_PASSWORD: orders_pass
      POSTGRES_DB: orders
    volumes:
      - pgdata:/var/lib/postgresql/data        # durable storage (Topic 14)
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orders_user -d orders"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s                        # grace window while Postgres initializes
    networks:
      - backend
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]       # exit 0 when Redis answers PONG
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - backend
    restart: unless-stopped

networks:
  backend:
    driver: bridge

volumes:
  pgdata:
```

### The connection string — where the service name lives

```
DATABASE_URL: postgres://orders_user:orders_pass@postgres:5432/orders
              │          │           │           │        │    └── database name
              │          │           │           │        └─────── PORT (Postgres default)
              │          │           │           └──────────────── HOST = service name → resolved by
              │          │           │                              embedded DNS to Postgres's IP.
              │          │           └──────────────────────────── password
              │          └──────────────────────────────────────── username
              └─────────────────────────────────────────────────── driver/scheme
```
The single most important character-span here is `@postgres:` — the host is the *service name*, not `localhost`, not an IP. That's what makes discovery work.

### `depends_on` conditions — the three you'll use

```
    depends_on:
      postgres:
        condition: service_healthy
        │          └── wait until the daemon reports postgres Health.Status == healthy
        └───────────── which service this condition applies to
      redis:
        condition: service_started            # wait only for the container to start (default)
      migrate:
        condition: service_completed_successfully   # wait until 'migrate' EXITS 0 (init jobs)
```

### The Postgres healthcheck that gates startup

```
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orders_user -d orders"]
      │      │             └── pg_isready: Postgres's own "are you accepting connections?" tool.
      │      └──────────────── CMD-SHELL: run via /bin/sh -c so we could add shell logic if needed.
      └─────────────────────── the probe.
      interval: 5s       # run every 5s
      timeout: 3s        # a probe taking >3s counts as a failure
      retries: 10        # need 10 consecutive fails (after start_period) to become "unhealthy"
      start_period: 10s  # for the first 10s, failing probes DON'T count — Postgres is booting
```

---

## Example 1 — basic

Prove name-based service discovery with the two smallest possible services — no app code needed.

```yaml
# docker-compose.yml
services:
  api:
    image: alpine:3.20               # a tiny box we can run shell commands in
    command: ["sleep", "3600"]       # stay alive an hour so we can exec into it
    networks: [appnet]
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: pw
    networks: [appnet]
networks:
  appnet:
```

```bash
docker compose up -d

# Resolve the service name "db" from inside "api" — embedded DNS answers with db's IP:
docker compose exec api getent hosts db
# 172.18.0.3   db

# Show the resolver that answered — 127.0.0.11 (the embedded DNS):
docker compose exec api cat /etc/resolv.conf
# nameserver 127.0.0.11

# Actually connect from api → db by NAME (install a tiny client, then reach db:5432):
docker compose exec api sh -c 'apk add --no-cache postgresql-client >/dev/null && \
  PGPASSWORD=pw psql -h db -U postgres -c "select 1 as ok;"'
#  ok
# ----
#   1
```

Line-by-line: both services are on `appnet`, so both are registered in that network's DNS under their service names (`api`, `db`). From inside `api`, the hostname `db` resolves — no IP, no `links:`, no manual config. That `-h db` is the whole lesson: **the service name is a hostname.**

---

## Example 2 — production scenario

Real incident: your `orders-api` deploy was flaky. About one boot in three, the API logged `ECONNREFUSED 172.18.0.3:5432` and crashed; `restart` brought it back and the second attempt "worked." CI was worse — the integration tests ran the instant containers started and failed ~40% of the time, so people started re-running CI on red, hiding real failures. Root cause: **the API connected to Postgres before Postgres finished initializing.** `depends_on` (list form) only waited for the container to *start*, not to be *ready*.

The fix is the full stack file above (health-gated startup), plus one more production-grade service: a database migration that must run *after* Postgres is healthy but *before* the API starts. Here's the addition:

```yaml
services:
  # ... postgres and redis as above (postgres has the healthcheck) ...

  migrate:
    build: .
    command: ["node", "migrate.js"]         # run schema migrations, then EXIT
    environment:
      DATABASE_URL: postgres://orders_user:orders_pass@postgres:5432/orders
    depends_on:
      postgres:
        condition: service_healthy          # don't migrate until DB is ready
    networks:
      - backend
    restart: "no"                           # one-shot: never auto-restart

  orders-api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://orders_user:orders_pass@postgres:5432/orders
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
      migrate:
        condition: service_completed_successfully   # start ONLY after migrations succeed
    networks:
      - backend
    restart: unless-stopped
```

Why this is the right shape:

- **Deterministic order:** `postgres` becomes healthy → `migrate` runs → `migrate` exits 0 → `orders-api` starts. Three conditions, three different needs, one predictable sequence. CI flakiness caused by ordering disappears.
- **`migrate` uses `restart: "no"`** — a one-shot task must not loop (Topic 19, Mistake 4).
- **`service_completed_successfully`** is the condition made for init/migration jobs: it gates the API on the migration's *successful exit*, so the API never starts against an un-migrated schema.
- **Belt and suspenders:** even with perfect ordering, your app code should still retry the initial DB connection with backoff. Containers can restart mid-life (a Postgres crash, an OOM kill), and when they do, the API must reconnect on its own. `depends_on` only governs *initial* startup, not the whole lifetime.

Node reconnect sketch (the application half of reliability):

```js
// db.js — retry the initial connection instead of crashing on the first refusal
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function connectWithRetry(attempt = 1) {
  try {
    await pool.query('select 1');           // probe the connection
    console.log('DB ready');
  } catch (err) {
    if (attempt > 10) throw err;            // give up after 10 tries
    const delay = Math.min(1000 * attempt, 5000);
    console.warn(`DB not ready (attempt ${attempt}): ${err.code}; retrying in ${delay}ms`);
    await new Promise(r => setTimeout(r, delay));
    return connectWithRetry(attempt + 1);
  }
}
module.exports = { pool, connectWithRetry };
```

---

## Common mistakes

### Mistake 1 — Using `localhost` (or `127.0.0.1`) to reach another container

```
DATABASE_URL: postgres://orders_user:orders_pass@localhost:5432/orders
```
**Error:**
```
Error: connect ECONNREFUSED 127.0.0.1:5432
```
**Root cause:** inside a container, `localhost` is *that container's own* network namespace (Topic 02). Postgres runs in a *different* namespace/container. There's nothing on 127.0.0.1:5432 inside `orders-api`.
**Right:** use the *service name* as the host: `@postgres:5432`. The embedded DNS resolves it to Postgres's container IP on the shared network.

### Mistake 2 — Hardcoding a container IP address

```
DATABASE_URL: postgres://...@172.18.0.3:5432/orders
```
**Symptom:** works today, breaks tomorrow with `ECONNREFUSED` after a `docker compose down && up`.
**Root cause:** container IPs are assigned by the daemon's IPAM and change across recreation. `172.18.0.3` might be Redis next time.
**Right:** never use IPs. Use the service name; DNS always points at the current IP.

### Mistake 3 — Believing `depends_on` (list form) waits for readiness

```yaml
  orders-api:
    depends_on:
      - postgres        # ← only waits for the CONTAINER to start, not for Postgres to be ready
```
**Error:** intermittent `ECONNREFUSED` at boot, "fixed" by a restart.
**Root cause:** the plain list form gates on `service_started`. The container's PID exists, but Postgres inside it is still running init scripts and isn't listening yet.
**Right:** use the map form with `condition: service_healthy` and give Postgres a `healthcheck`. And retry in app code for lifetime resilience.

### Mistake 4 — A dependency and its dependent are on different networks

```yaml
  orders-api:
    networks: [frontend]
  postgres:
    networks: [backend]      # ← different network; DNS name "postgres" won't resolve for the API
```
**Error:**
```
Error: getaddrinfo ENOTFOUND postgres
```
**Root cause:** the embedded DNS is *per network*. A service is only registered in the DNS of networks it's attached to. If they share no network, the name doesn't resolve (and even if it did, no route exists).
**Right:** put services that must talk on at least one shared network. Here, both need `backend`.

### Mistake 5 — A healthcheck tool that isn't installed in the image

```yaml
  orders-api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]   # curl not in node:20-alpine!
```
**Symptom:** the service is stuck `unhealthy` (or `health: starting` forever), and anything depending on it never starts.
```
docker inspect ... State.Health.Log → "exec: \"curl\": executable file not found in $PATH"
```
**Root cause:** `alpine`/distroless images are minimal and often lack `curl`. The probe command can't run, so it "fails."
**Right:** use a tool that exists (`wget` is in alpine; Node images have `node`), or install curl in the Dockerfile, or write a tiny Node probe: `["CMD", "node", "-e", "fetch('http://localhost:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]`.

---

## Hands-on proof

Bring up the real three-service stack and watch discovery *and* health-gated ordering happen.

```bash
mkdir -p /tmp/multi-app && cd /tmp/multi-app
cat > docker-compose.yml <<'EOF'
services:
  orders-api:
    image: alpine:3.20
    command: ["sleep", "3600"]           # stand-in for the API so no build is needed
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    networks: [backend]
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: orders_user
      POSTGRES_PASSWORD: orders_pass
      POSTGRES_DB: orders
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orders_user -d orders"]
      interval: 3s
      timeout: 3s
      retries: 10
      start_period: 5s
    networks: [backend]
  redis:
    image: redis:7-alpine
    networks: [backend]
networks:
  backend:
volumes:
  pgdata:
EOF

# 1. Bring it up and WATCH the ordering: postgres goes healthy BEFORE orders-api starts
docker compose up -d
docker compose ps

# 2. PROOF of health gating: postgres health transitions starting → healthy
watch -n1 "docker inspect multi-app-postgres-1 --format '{{.State.Health.Status}}'"
# (Ctrl-C once it says 'healthy'. orders-api only appears 'running' after this.)

# 3. PROOF of service discovery: from orders-api, resolve BOTH names by DNS
docker compose exec orders-api getent hosts postgres
docker compose exec orders-api getent hosts redis
docker compose exec orders-api cat /etc/resolv.conf     # nameserver 127.0.0.11

# 4. PROOF of connectivity by name (not IP):
docker compose exec orders-api sh -c 'nc -zv postgres 5432; nc -zv redis 6379'

# 5. PROOF the IP is not stable but the NAME is: recreate postgres, re-resolve
OLD=$(docker compose exec orders-api getent hosts postgres | awk '{print $1}')
docker compose up -d --force-recreate postgres
sleep 6
NEW=$(docker compose exec orders-api getent hosts postgres | awk '{print $1}')
echo "postgres IP was $OLD, now $NEW — the NAME 'postgres' still resolves either way"

# 6. Inspect the network: all three containers, their IPs and aliases
docker network inspect multi-app_backend --format '{{json .Containers}}' | tr ',' '\n'

docker compose down -v
```

Step 5 is the money shot: recreate Postgres, its IP changes, but the name `postgres` keeps resolving — proving why you must use names, never IPs.

---

## Practice exercises

### Exercise 1 — easy

Write a two-service Compose file: `web` (`nginx:alpine`) and `cache` (`redis:7-alpine`) on a shared network. From inside `web`, run `getent hosts cache` and confirm it resolves to an IP, and confirm `/etc/resolv.conf` shows `nameserver 127.0.0.11`. In one sentence, say what would happen to the name resolution if you put them on *different* networks.

### Exercise 2 — medium

Take the three-service stack. Give Redis a `healthcheck` using `redis-cli ping`, and change `orders-api`'s dependency on Redis to `condition: service_healthy`. Then deliberately break the Redis healthcheck (e.g. `redis-cli -p 9999 ping`) and bring the stack up. Observe with `docker compose ps` that `orders-api` never reaches `running` because Redis never becomes healthy. Fix the check and confirm the API starts. Explain which daemon field Compose was polling.

### Exercise 3 — hard (production simulation)

Reproduce the Example 2 stack with the `migrate` one-shot service (`condition: service_completed_successfully` gating `orders-api`). Make `migrate` *fail* the first time (e.g. `command: ["sh","-c","exit 1"]`) and confirm that `orders-api` never starts, because its migration precondition failed. Then fix `migrate` to `exit 0` and confirm the full ordered chain: `postgres healthy → migrate exits 0 → orders-api starts`. Finally, kill Postgres mid-run (`docker compose kill postgres`) and describe what your *application code* must do (not Compose) so the API survives — tying it back to the retry sketch in Example 2.

---

## Mental model checkpoint

1. Inside `orders-api`, why does connecting to `localhost:5432` fail but connecting to `postgres:5432` succeed?
2. What IP address is in every container's `/etc/resolv.conf`, and what runs there?
3. Why must you never hardcode a container's IP in a connection string?
4. What's the exact difference between `depends_on` list form and `depends_on` with `condition: service_healthy`?
5. Which `depends_on` condition is designed for a one-shot migration job, and why?
6. If two services are on different networks, can they resolve each other's service names? Why or why not?
7. `depends_on` handles startup order — but what must your *app code* do to survive a dependency restarting mid-life?

---

## Quick reference card

| Thing | What it does | Key detail |
|---|---|---|
| Service name as hostname | Other services reach it by this name | Resolved by embedded DNS to current container IP |
| `127.0.0.11` | The embedded DNS server, per network | In every container's `/etc/resolv.conf` |
| Shared `networks:` | Lets services talk + be discoverable | DNS is per-network; no shared net → no resolution |
| `depends_on` (list) | Start ordering only | Waits for *started*, not *ready* |
| `condition: service_healthy` | Wait for dependency's healthcheck to pass | Requires a `healthcheck` on the dependency |
| `condition: service_started` | Wait for the container to start (default) | Fast deps like Redis |
| `condition: service_completed_successfully` | Wait for a service to exit 0 | For migration/init one-shot jobs |
| `healthcheck` + `.State.Health.Status` | Daemon probes; Compose polls the status | `starting` → `healthy`/`unhealthy` |
| App-side connect retry | Survives dependency restarts mid-life | `depends_on` covers only startup |

---

## When would I use this at work?

1. **Standing up any real service locally.** `orders-api` + Postgres + Redis (or Kafka, or another API) is the shape of nearly every backend. This topic is the daily pattern: names for discovery, health gates for order, one private network.

2. **Killing CI flakiness.** Integration tests that "randomly fail" almost always have a startup-order bug. Adding a Postgres `healthcheck` + `condition: service_healthy` (and app-side retries) is the standard fix that turns a 40%-red pipeline green.

3. **Safe schema migrations.** The `migrate` service gated by `service_healthy` (dependency ready) and gating the API by `service_completed_successfully` (migration done) is exactly how teams run migrations in dev/CI before the API touches the schema — the same idea reappears as Kubernetes init containers and Jobs (Topics 33, 48).

---

## Connected topics

- **Study before:** Topic 13 (Networking in Docker — the bridge, veth, and DNS this topic relies on), Topic 18 (Docker Compose in Depth — the default network and project model), Topic 19 (Compose File Deep Dive — `depends_on`, `healthcheck`, `networks` field semantics), Topic 15 (Environment Variables — connection strings).
- **Study after:** Topic 21 (Compose for Development — overrides and hot reload on top of this stack), Topic 22 (Compose Commands — operating the stack), Topic 26 (Health Checks — the daemon side in depth). In Kubernetes: Topic 35 (Services — the same name-based discovery via cluster DNS), Topic 43 (Health Checks — liveness/readiness probes, the K8s evolution of healthcheck + service_healthy), Topic 33 (Pods/init containers — the `migrate` pattern reimagined).
