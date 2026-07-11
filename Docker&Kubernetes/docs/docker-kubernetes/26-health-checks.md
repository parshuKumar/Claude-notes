# 26 — Health Checks

## Section: Production Patterns

---

## ELI5 — The Simple Analogy

Imagine you run a shipping company. Every delivery van has a driver. From the outside, a parked van *looks* fine — engine on, lights on, doors closed. But "the engine is running" does not mean "the driver is ready to deliver packages." Maybe the driver fell asleep. Maybe the GPS is frozen. Maybe the van is running but stuck in neutral.

So you install a little device: every 30 seconds it asks the driver one question — *"Can you deliver a package right now? Yes or no?"* If the driver answers "yes" three times in a row, you paint a green sticker on the van: **healthy**. If the driver fails to answer three times in a row, you paint a red sticker: **unhealthy**, and you tell dispatch to stop sending new packages to that van.

A Docker **health check** is exactly that little device. Docker already knows if your container *process* is running (the engine is on). But a running process is NOT the same as a *working* app. Your Node.js server can be alive but unable to reach Postgres, or stuck in an infinite loop, or out of memory but not yet crashed. The health check is the question Docker asks your container on a schedule: *"Are you actually able to do your job right now?"*

---

## The Linux kernel feature underneath

There is no single magic kernel primitive called "health check." A health check is built out of two ordinary kernel mechanisms you already met in earlier topics:

**1. `fork()` + `execve()` inside the container's namespaces.**
When Docker runs your health-check command, the daemon asks `containerd` → `runc` to spawn a *new process* **inside the exact same namespaces** as your main container (remember the namespace trinity from Topic 02: PID, network, mount, etc.). This is the same mechanism as `docker exec`. So when your health check runs `curl http://localhost:3000/healthz`, that `localhost` is the container's *own* network namespace — it can see your app's port even though the host cannot.

```
Host kernel
 └─ containerd-shim (keeps the container alive)
     ├─ PID 1: node server.js          ← your app (the "engine")
     └─ PID 47: /bin/sh -c "curl ..."  ← the health probe, SAME namespaces
                                          spawned every --interval seconds
```

**2. The process exit code (`wait4()` syscall).**
The kernel already tracks the exit status of every child process. When the health-check process finishes, the kernel stores its exit code in the process table, and the parent (the shim/daemon) reaps it with `wait4()`. Docker reads that number:

- exit code `0` → **success** (the app is healthy this round)
- exit code `1` (or anything non-zero) → **failure**

That is the whole contract. A health check is "spawn a process in the container's namespaces on a timer, read its exit code." Everything else — states, retries, timers — is bookkeeping the Docker daemon does in userspace on top of these two kernel facts.

The timer itself is just the daemon's own event loop scheduling the next probe; there is no kernel timer bound to the container.

---

## What is this?

A **health check** is a command Docker runs *inside* your container on a repeating schedule to decide whether the app is truly able to serve traffic — not just whether its process exists. You define it with the `HEALTHCHECK` instruction in a Dockerfile, or with `--health-*` flags on `docker run`. Docker turns the results into a **health status** (`starting`, `healthy`, or `unhealthy`) that you can read, react to, and depend on.

---

## Why does it matter for a backend developer?

Because "the process is running" is a lie your monitoring tells you.

Picture `orders-api`. It boots, connects to Postgres and Redis, starts listening on port 3000. Node's process is alive. `docker ps` shows `Up 2 hours`. Everyone relaxes.

Now Postgres has a network hiccup and every query hangs. Your `/orders` endpoint returns 500s or times out. But **the Node process is still running** — it did not crash, so `docker ps` still cheerfully says `Up 2 hours`. Your load balancer keeps sending customer traffic to a broken container. Orders fail. Nobody gets paged, because from Docker's naive view, nothing is wrong.

Without health checks:
- **`depends_on` in Compose is a lie.** Topic 20 covered this — `depends_on` only waits for a container to *start*, not to be *ready*. Your API starts before Postgres can accept connections, crashes, and you blame Compose.
- **Rolling deploys route traffic into cold containers.** A container that is still warming up (running migrations, filling caches) gets live traffic and errors out.
- **Zombie-but-alive containers stay in rotation.** The exact scenario above: alive process, dead app, no alert.
- **Orchestrators can't self-heal.** Kubernetes (Topic 43) and Swarm restart *unhealthy* containers automatically — but only if you told them how to measure "unhealthy." A health check is the source of that truth.

Health checks are how you make the difference between **liveness** ("is the process there?") and **readiness** ("can it actually work?") into something the platform can act on.

---

## The physical reality

When a container has a health check, the daemon stores its state and history in the container's JSON config on the host. On a standard Linux Docker install:

```
/var/lib/docker/containers/<container-id>/
 └─ config.v2.json     ← contains the "Health" object
```

Inside that file (pretty-printed) you get something like:

```json
"State": {
  "Status": "running",
  "Running": true,
  "Health": {
    "Status": "healthy",          // starting | healthy | unhealthy
    "FailingStreak": 0,           // consecutive failures so far
    "Log": [                      // last 5 probe results (ring buffer)
      {
        "Start": "2026-07-12T10:15:03.11Z",
        "End":   "2026-07-12T10:15:03.20Z",
        "ExitCode": 0,
        "Output": "OK healthy\n"  // stdout+stderr, truncated to 4096 bytes
      }
    ]
  }
}
```

Key physical facts:
- **`FailingStreak`** is a counter in memory (persisted to that JSON). It increments on each failed probe and **resets to 0 on the first success**. When it reaches `--retries`, the status flips to `unhealthy`.
- **`Log`** is a ring buffer of the **last 5** probe results only. Output is captured but **truncated to 4096 bytes** — do not print megabytes of debug in your health command; Docker throws away the rest.
- The probe runs as a **real child process** you can catch with `docker top <container>` or `ps` on the host if your timing is lucky — you'll see the `/bin/sh -c` of your health command appear and vanish every interval.
- There is **no separate health-check daemon**. The main `dockerd` process schedules and runs the probes as part of managing that container.

You can read the live object without digging through JSON on disk:

```bash
docker inspect --format '{{json .State.Health}}' orders-api | jq
```

---

## How it works — step by step

Let's trace a container from launch to `unhealthy` and back, assuming:
`--interval=30s --timeout=3s --retries=3 --start-period=20s`.

1. **Container starts.** Docker sets `Health.Status = "starting"` immediately. During `starting`, the container is considered "not yet failing" — this protects slow-booting apps.

2. **The start-period clock begins (20s).** For the first 20 seconds, probe failures **do not count** toward the failing streak. A boot-time failure is expected (Postgres pool not connected yet), so Docker forgives it. Important nuance: if a probe *succeeds* during the start period, the container flips to `healthy` **immediately** — the start period does not force you to wait the full 20s.

3. **First probe fires.** The daemon waits `--interval` (30s) after container start, then spawns `/bin/sh -c "<your health-cmd>"` inside the container's namespaces (the `fork()`+`execve()` from above).

   - Actually: Docker runs the *first* probe after `interval` from start by default. (Newer Docker also supports `--start-interval` to probe more frequently during the start period — covered in the syntax section.)

4. **Timeout guard (3s).** The daemon starts a 3-second timer. If the probe process has not exited within 3s, Docker **kills it** (sends signals to that process) and records the result as a **failure** with exit code `-1` and output `"Health check exceeded timeout (3s)"`. This is critical: a hung health check (e.g. `curl` blocked on a dead socket) must not hang forever.

5. **Read the exit code (`wait4()`).** Probe process exits.
   - Exit `0` → success. `FailingStreak` resets to `0`. If we were `starting` or `unhealthy`, status may move to `healthy`.
   - Exit non-zero (or timeout kill) → failure. `FailingStreak += 1`.

6. **State machine decision.**
   - If `Status == starting` **and** we are past the start period **and** `FailingStreak >= retries` → set `Status = unhealthy`.
   - If `Status == healthy` and `FailingStreak >= retries` → set `Status = unhealthy`.
   - Any success → `FailingStreak = 0`, `Status = healthy`.

7. **Schedule the next probe.** The daemon schedules the next run for `interval` (30s) after the *previous probe finished* (interval counts from completion, not from start of the previous probe). Loop back to step 3.

8. **Emit an event.** Each transition emits a Docker event you can subscribe to:
   ```
   docker events --filter event=health_status
   → health_status: healthy
   → health_status: unhealthy
   ```
   This is the hook Compose, Swarm, and monitoring tools listen on.

**Worked timeline** — orders-api loses Postgres at t=120s:

```
t=0    Status: starting   (start-period active until t=20)
t=30   probe OK           Status: healthy   FailingStreak=0
t=60   probe OK           Status: healthy   FailingStreak=0
t=90   probe OK           Status: healthy   FailingStreak=0
       --- Postgres dies at t=120, /healthz now returns 503 ---
t=120  probe FAIL (503)   Status: healthy   FailingStreak=1
t=150  probe FAIL (503)   Status: healthy   FailingStreak=2
t=180  probe FAIL (503)   Status: unhealthy FailingStreak=3   ← flips here
       docker event: health_status: unhealthy
       --- Postgres recovers at t=200 ---
t=210  probe OK           Status: healthy   FailingStreak=0   ← recovers on FIRST success
       docker event: health_status: healthy
```

Notice it took **90 seconds** (3 × 30s interval) to notice the failure. That lag is entirely determined by `interval × retries`. Tune accordingly.

---

## Exact syntax breakdown

### The Dockerfile `HEALTHCHECK` instruction

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -f http://localhost:3000/healthz || exit 1
```

```
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 CMD curl -f http://localhost:3000/healthz || exit 1
│           │               │            │                  │           │   │  │  │                                  │
│           │               │            │                  │           │   │  │  │                                  └─ force a failure exit code if curl fails
│           │               │            │                  │           │   │  │  └─ the URL: localhost = container's OWN net namespace
│           │               │            │                  │           │   │  └─ -f = fail (non-zero exit) on HTTP 4xx/5xx; without -f, curl exits 0 even on 500!
│           │               │            │                  │           │   └─ the program to run: curl
│           │               │            │                  │           └─ CMD = the actual probe command follows (shell form here → wrapped in /bin/sh -c)
│           │               │            │                  └─ --retries=3 : consecutive failures needed to become "unhealthy"
│           │               │            └─ --start-period=20s : grace window at boot where failures don't count
│           │               └─ --timeout=3s : kill the probe if it runs longer than this; counts as a failure
│           └─ --interval=30s : wait this long between the END of one probe and the START of the next
└─ HEALTHCHECK : the Dockerfile instruction (only the LAST HEALTHCHECK in a Dockerfile wins)
```

Two forms of the `CMD`:

```dockerfile
# Shell form — wrapped in /bin/sh -c, so ||, &&, $VARS work. Needs a shell in the image.
HEALTHCHECK CMD curl -f http://localhost:3000/healthz || exit 1

# Exec form — no shell, args passed directly to execve(). Works in distroless/scratch.
HEALTHCHECK CMD ["curl", "-f", "http://localhost:3000/healthz"]
```
```
HEALTHCHECK CMD ["curl", "-f", "http://localhost:3000/healthz"]
│               │  │       │     │
│               │  │       │     └─ arg: the URL
│               │  │       └─ arg: -f flag (fail on HTTP errors)
│               │  └─ argv[0]: the executable, run directly — NO /bin/sh involved
│               └─ JSON array = exec form. Use this for distroless images (Topic 23) that have no shell.
└─ instruction
```

**Disabling an inherited health check** (a base image set one you don't want):

```dockerfile
HEALTHCHECK NONE
```
```
HEALTHCHECK NONE
│           └─ NONE = turn OFF any HEALTHCHECK inherited from a FROM base image
└─ instruction
```

### The `docker run` flags (same knobs, at runtime)

```bash
docker run -d --name orders-api \
  --health-cmd='curl -f http://localhost:3000/healthz || exit 1' \
  --health-interval=30s \
  --health-timeout=3s \
  --health-start-period=20s \
  --health-retries=3 \
  orders-api:latest
```

```
--health-cmd='curl -f http://localhost:3000/healthz || exit 1'
│             └─ the probe command (a string → run via /bin/sh -c inside the container)
└─ runtime equivalent of the HEALTHCHECK ... CMD part

--health-interval=30s      → same as --interval
--health-timeout=3s        → same as --timeout
--health-start-period=20s  → same as --start-period
--health-retries=3         → same as --retries
--no-healthcheck           → runtime equivalent of HEALTHCHECK NONE (disable entirely)
```

### The Compose `healthcheck` block (what you'll use most)

```yaml
services:
  orders-api:
    image: orders-api:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/healthz"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 20s
```

```
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/healthz"]
      │      │       └─ exec form: run curl directly, no shell
      │      └─ "CMD"      = exec form (array of args)
      │         "CMD-SHELL"= shell form → test: ["CMD-SHELL", "curl -f ... || exit 1"]
      │         "NONE"     = disable an inherited healthcheck
      └─ test : the probe. In Compose, note it's start_period (underscore), not --start-period
```

### Defaults (if you omit a flag)

| Flag | Default | Meaning |
|------|---------|---------|
| `--interval` | `30s` | time between probes |
| `--timeout` | `30s` | max probe duration before it's killed |
| `--retries` | `3` | failures in a row to become unhealthy |
| `--start-period` | `0s` | no grace window unless you set one |
| `--start-interval` | `5s` | (newer Docker) probe frequency *during* the start period |

The default `--timeout` of 30s is dangerously long for most apps — always set it lower (2–5s).

---

## Example 1 — basic

A minimal `orders-api` with a real `/healthz` endpoint and a Dockerfile health check. Every line commented.

**`server.js`** — the app plus its health endpoint:

```javascript
const express = require('express');          // web framework
const { Pool } = require('pg');              // Postgres client
const Redis = require('ioredis');            // Redis client

const app = express();
const pg = new Pool({ connectionString: process.env.DATABASE_URL });
const redis = new Redis(process.env.REDIS_URL);

// Normal business endpoint
app.get('/orders', async (req, res) => {
  const { rows } = await pg.query('SELECT id, total FROM orders LIMIT 10');
  res.json(rows);
});

// The health endpoint Docker will probe.
// It must be CHEAP and answer the question "can I do my job RIGHT NOW?"
app.get('/healthz', async (req, res) => {
  try {
    await pg.query('SELECT 1');              // can I reach Postgres? (cheap, no table scan)
    await redis.ping();                      // can I reach Redis?
    res.status(200).send('OK healthy');      // exit-code-0 path for curl -f
  } catch (err) {
    // A dependency is down → tell Docker we are NOT healthy.
    res.status(503).send('DEP DOWN: ' + err.message);  // 503 makes curl -f exit non-zero
  }
});

app.listen(3000, () => console.log('orders-api on :3000'));
```

**`Dockerfile`**:

```dockerfile
FROM node:20-alpine                          # small base (Topic 23)
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev                        # install prod deps only
COPY . .

# alpine ships wget but NOT curl by default — install it so our probe works:
RUN apk add --no-cache curl

EXPOSE 3000

# The health check: hit our own /healthz every 30s.
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:3000/healthz || exit 1

USER node                                    # non-root (Topic 23)
CMD ["node", "server.js"]
```

Build and run:

```bash
docker build -t orders-api:latest .
docker run -d --name orders-api -p 3000:3000 \
  -e DATABASE_URL=postgres://app:pw@postgres:5432/orders \
  -e REDIS_URL=redis://redis:6379 \
  orders-api:latest

docker ps
# STATUS column now shows the health inline:
# Up 25 seconds (health: starting)
# ...then...
# Up 2 minutes (healthy)
```

That `(healthy)` / `(health: starting)` text in `docker ps` **only appears when a health check is defined**. If you never see it, you don't have one.

---

## Example 2 — production scenario

**The situation:** Your team runs `orders-api` behind a load balancer, deployed via Docker Compose on a single VM (pre-Kubernetes). Twice this month, customers reported "orders won't load" while your dashboard showed all containers `Up`. Post-mortem: Redis connection pool silently exhausted after a traffic spike; the Node process stayed alive but every request hung waiting for a Redis connection that never came. `docker ps` said `Up 6 hours` the whole time.

You add a **meaningful** health check and let Compose gate startup and restarts on it.

**`docker-compose.yml`**:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: pw
      POSTGRES_DB: orders
    healthcheck:
      # pg_ships with pg_isready — the canonical "is Postgres accepting connections?" probe
      test: ["CMD-SHELL", "pg_isready -U app -d orders"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 10s

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]     # returns PONG + exit 0 when healthy
      interval: 10s
      timeout: 3s
      retries: 5

  orders-api:
    build: .
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgres://app:pw@postgres:5432/orders
      REDIS_URL: redis://redis:6379
    # Do NOT start the API until BOTH deps report healthy — fixes the depends_on lie (Topic 20):
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/healthz"]
      interval: 15s
      timeout: 3s
      retries: 3
      start_period: 20s     # migrations run at boot; give them room
    restart: unless-stopped  # if it dies, bring it back
```

**Making the health check *catch the real failure*.** A `/healthz` that only returns `200 OK` without touching anything would NOT have caught the Redis pool exhaustion. So the endpoint must actually exercise the dependency the way real requests do:

```javascript
app.get('/healthz', async (req, res) => {
  try {
    // Race each check against a short timeout so a HUNG dependency
    // fails the probe fast instead of hanging until Docker's --timeout kills us.
    const withTimeout = (p, ms) =>
      Promise.race([p, new Promise((_, rej) => setTimeout(() => rej(new Error('timeout')), ms))]);

    await withTimeout(pg.query('SELECT 1'), 1000);
    await withTimeout(redis.ping(), 1000);       // THIS is what catches pool exhaustion

    res.status(200).send('OK healthy');
  } catch (err) {
    res.status(503).send('unhealthy: ' + err.message);
  }
});
```

Now, when Redis stalls: after 3 × 15s = 45s the container flips to `unhealthy`, emits a `health_status: unhealthy` event, and your monitoring (which subscribes to `docker events`) pages someone. Combined with an orchestrator, the container gets restarted automatically. The silent-outage class of bug is gone.

**Important production caveat — what plain Docker does NOT do:** Marking a container `unhealthy` does **not**, by itself, restart it. `restart: unless-stopped` reacts to the process *exiting*, not to *health*. Read the next section carefully.

---

## What Docker actually does with an unhealthy container

This is the single most misunderstood part of health checks. Be precise:

| Runtime | On `unhealthy` | Restarts it? |
|---------|----------------|--------------|
| **Plain `docker run` / Compose** | Sets status to `unhealthy`, emits an event, shows it in `docker ps`. **That's all.** | **No.** The container keeps running, broken, forever. |
| **Docker Swarm** | The swarm scheduler sees `unhealthy`, kills the task, and starts a replacement to satisfy the desired replica count. | **Yes.** |
| **Kubernetes** | Docker's own `HEALTHCHECK` is **ignored**. K8s uses its *own* liveness/readiness probes instead (Topic 43). | **Yes**, via liveness probe. |

So in plain Docker, a health check is an **information signal**, not an **action**. To actually *do* something, you need one of:

1. **An orchestrator** (Swarm/K8s) that acts on health — the real answer for production.
2. **`--restart` does NOT help** — restart policy triggers on *exit code*, not health status. A running-but-unhealthy container never exits, so a restart policy never fires. This trips up everyone once.
3. **An external watcher** like [`willfarrell/autoheal`](https://github.com/willfarrell/autoheal) — a small container that subscribes to `health_status: unhealthy` events and calls `docker restart` for you. Useful glue on a single-VM Compose setup before you move to Kubernetes.

The design lesson: a Docker health check exists so that **some higher-level system** can make a decision. Its job is to *report truth*, not to *fix* anything.

---

## Common mistakes

### Mistake 1 — `curl` without `-f`, so every probe "passes"

```dockerfile
HEALTHCHECK CMD curl http://localhost:3000/healthz    # ← no -f
```
Your `/healthz` returns `503`. `curl` prints the error body and exits `0` anyway (it only cares that it *reached* the server). Docker reads exit `0` → **healthy forever**, even when the app is down.

**Symptom:** `docker ps` says `(healthy)` during a real outage.

**Root cause:** exit code `0` = success. Without `-f`/`--fail`, curl's exit code reflects "did I complete the HTTP transaction," not "was the status 2xx." The health-check contract is *only* the exit code (the `wait4()` value from the kernel).

**Right:**
```dockerfile
HEALTHCHECK CMD curl -f http://localhost:3000/healthz || exit 1
```
`-f` makes curl exit `22` on HTTP ≥ 400; the `|| exit 1` is belt-and-suspenders.

---

### Mistake 2 — using `curl` in an image that doesn't have it

```
OCI runtime exec failed: exec: "curl": executable file not found in $PATH: unknown
```
`node:20-alpine` and `distroless` do **not** ship `curl`. Docker tries to run your probe, the exec fails, exit code is non-zero → the container is **permanently `unhealthy`** even though the app is perfectly fine.

**Symptom:** container is `unhealthy` immediately, `docker inspect` health log shows `"executable file not found"`.

**Right — three options:**
```dockerfile
RUN apk add --no-cache curl                       # 1. install curl on alpine
# or use wget (alpine's busybox has it built in):
HEALTHCHECK CMD wget -qO- http://localhost:3000/healthz || exit 1
# 2. or probe from inside Node with no external binary (works in distroless):
HEALTHCHECK CMD ["node", "-e", "require('http').get('http://localhost:3000/healthz',r=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))"]
```
Option 2 is the cleanest for distroless: `node` is already in the image, no shell or extra binary needed.

---

### Mistake 3 — health check does an expensive query, hammering your DB

```javascript
app.get('/healthz', async (req, res) => {
  const { rows } = await pg.query('SELECT count(*) FROM orders');  // full table scan!
});
```
With `--interval=10s` across 20 replicas, that's a heavy `count(*)` on the orders table **120 times a minute**, forever. Your health check becomes a load source.

**Root cause:** the probe runs on a tight loop; anything expensive multiplies by (replicas × 60 / interval).

**Right:** use the cheapest possible liveness signal — `SELECT 1`, `pg_isready`, `redis-cli ping`. Never scan a table in a health check.

---

### Mistake 4 — no `start_period`, so slow boots get killed

Your API runs DB migrations on startup (takes 15s). With `--retries=3 --interval=5s` and **no start period**, three probes fail during migration (t=5, 10, 15), the container is declared `unhealthy` at t=15, and your orchestrator kills it — right before it would have finished booting. It never comes up. **CrashLoopBackoff** in K8s terms.

**Root cause:** failures during boot count against you when there's no grace window.

**Right:** set `--start-period` generously to cover worst-case boot (migrations, cache warm, JIT). Failures inside it are free; a success inside it flips you healthy early.

---

### Mistake 5 — thinking `restart: always` reacts to health

```yaml
orders-api:
  healthcheck: { test: [...], ... }
  restart: always      # ← does NOT restart on unhealthy
```
The container goes `unhealthy` and sits there broken. The team assumes `restart: always` will recycle it. It won't — restart policies fire on process **exit**, not on health **status** (see the table above).

**Right:** run an orchestrator (Swarm/K8s), or add an `autoheal` sidecar that watches `health_status` events and restarts on your behalf.

---

## Hands-on proof

Run these right now to *see* every state transition. No Postgres needed — we'll fake the health command.

```bash
# 1. A container that is healthy for 15s, then permanently fails.
#    (touch a file at boot; delete it after 15s; probe checks the file exists)
docker run -d --name htest \
  --health-cmd='test -f /tmp/ok' \
  --health-interval=3s --health-retries=2 --health-timeout=2s \
  alpine sh -c 'touch /tmp/ok; sleep 15; rm /tmp/ok; sleep 600'

# 2. Watch the STATUS column flip in real time:
watch -n1 'docker ps --format "table {{.Names}}\t{{.Status}}"'
#   Up 2 seconds (health: starting)
#   Up 6 seconds (healthy)
#   ... after ~15s the file is gone ...
#   Up 20 seconds (unhealthy)      ← 2 failed probes × 3s interval

# 3. Read the raw health object + last 5 probe results:
docker inspect --format '{{json .State.Health}}' htest | jq
#   watch FailingStreak climb 0 → 1 → 2, then Status: "unhealthy"

# 4. Subscribe to the health events stream (open in another terminal FIRST):
docker events --filter event=health_status
#   2026-07-12T...  container health_status: healthy   htest
#   2026-07-12T...  container health_status: unhealthy htest

# 5. Prove that restart policy does NOT react to unhealthy:
#    (the container above has no --restart; it stays Up + unhealthy forever)
docker ps   # still "Up ... (unhealthy)" — nobody restarts it

# cleanup
docker rm -f htest
```

What you just verified with your own eyes:
- `starting` → `healthy` on the first success, before the interval math would suggest.
- `FailingStreak` counts up and the flip to `unhealthy` happens exactly at `retries`.
- Docker emits `health_status` events you can build automation on.
- An unhealthy container is **not** restarted by Docker alone.

---

## Practice exercises

### Exercise 1 — easy
Add a `HEALTHCHECK` to the Example 1 Dockerfile that probes `/healthz` every 10 seconds with a 2-second timeout and 3 retries. Build it, run it, and confirm `docker ps` shows `(health: starting)` and then `(healthy)`. Then run `docker inspect --format '{{.State.Health.Status}}'` and confirm it prints `healthy`.

### Exercise 2 — medium
Write a `/healthz` endpoint for `orders-api` that returns `503` if Postgres is unreachable. Start the API with Compose, confirm it's `healthy`, then run `docker stop postgres`. Time how long it takes for `orders-api` to become `unhealthy`. Explain the number using `interval × retries`. Then bring Postgres back with `docker start postgres` and confirm the API returns to `healthy` on the **first** successful probe.

### Exercise 3 — hard (production simulation)
On a single VM with Compose, make `orders-api` **self-heal** without Kubernetes:
1. Add the `willfarrell/autoheal` container to your Compose file (label your API with `autoheal=true`).
2. Give `orders-api` a health check whose `/healthz` returns `503` when Redis is down.
3. Simulate the Example 2 outage: `docker stop redis`, watch the API go `unhealthy`, and confirm autoheal restarts it.
4. Now explain: why does the restart alone NOT actually fix anything until Redis is back? What does this teach you about the difference between "restart the app" and "fix the dependency"? Write down when a restart *does* help (in-process leak) vs when it's useless (external dep down).

---

## Mental model checkpoint

Answer from memory:
1. What are the three health states, and what exactly causes each transition?
2. Why is `curl` without `-f` a silent bug in a health check?
3. What does `--start-period` protect against, and what happens if a probe *succeeds* during it?
4. If a container is `unhealthy`, does plain Docker restart it? What actually will?
5. What two kernel mechanisms does a health check reduce to?
6. How long (in terms of the flags) does it take to detect a failure with `--interval=15s --retries=4`?
7. Why should a health check never run `SELECT count(*)`?

---

## Quick reference card

| Instruction / flag / field | What it does | Key detail |
|---|---|---|
| `HEALTHCHECK ... CMD <cmd>` | Dockerfile: define the probe | Only the **last** HEALTHCHECK in the file applies |
| `HEALTHCHECK NONE` | Disable an inherited probe | Use when a base image sets one you don't want |
| `--interval` / `interval` | Time between probes | Counts from **end** of previous probe; default 30s |
| `--timeout` / `timeout` | Max probe runtime | Exceeding = failure; default 30s (set it lower!) |
| `--retries` / `retries` | Consecutive fails → unhealthy | Detection lag ≈ `interval × retries` |
| `--start-period` / `start_period` | Boot grace window | Failures inside don't count; a success ends it early |
| `--start-interval` | Probe frequency during start period | Newer Docker only; default 5s |
| `--health-cmd` (run) / `test:` (compose) | The probe command | `CMD` = exec form, `CMD-SHELL` = shell form |
| `--no-healthcheck` | Runtime disable | Same as `HEALTHCHECK NONE` |
| `docker inspect --format '{{json .State.Health}}'` | Read live health + last 5 logs | `FailingStreak`, `Log[]` ring buffer (4096-byte cap) |
| `docker events --filter event=health_status` | Stream transitions | The hook for autoheal / monitoring |
| Exit `0` | Probe success | Resets FailingStreak to 0 |
| Exit non-zero / timeout | Probe failure | Increments FailingStreak |

---

## When would I use this at work?

1. **Gating startup order in Compose (dev + single-VM prod).** Your `orders-api` must not boot before Postgres accepts connections. `depends_on: { postgres: { condition: service_healthy } }` uses Postgres's health check to hold the API back until the DB is truly ready — fixing the classic "API crashes on first deploy because DB wasn't up yet" (Topic 20).

2. **Catching silent, alive-but-broken outages.** The Redis-pool-exhaustion scenario: process alive, app dead. A `/healthz` that actually pings Redis turns an invisible outage into a `health_status: unhealthy` event that pages someone within `interval × retries` seconds.

3. **Feeding your orchestrator's self-healing.** When you move `orders-api` to Kubernetes (Topic 43), the *concept* you built here — a cheap endpoint that answers "can I work right now?" — becomes your liveness and readiness probes. The Dockerfile `HEALTHCHECK` is where you first design that endpoint; K8s then acts on it far more aggressively than plain Docker ever will.

---

## Connected topics

- **Study before:**
  - Topic 02 — Linux Foundations (namespaces: why the probe sees the container's `localhost`).
  - Topic 06 — Dockerfile in Depth (where `HEALTHCHECK` lives among the instructions).
  - Topic 20 — Multi-Container Apps (the `depends_on` + `service_healthy` startup gating).
  - Topic 25 — Resource Limits (OOMKilled is another way "alive" becomes "dead").
- **Study after:**
  - Topic 27 — Docker in CI/CD (verify a built image passes its health check before you ship it).
  - Topic 43 — Health Checks in Kubernetes (liveness vs readiness vs startup probes — the same endpoint, but the orchestrator finally *acts*).
