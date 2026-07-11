# 11 — Production Dockerfile for Node.js

## Section: Docker in Practice

---

## ELI5 — The Simple Analogy

Think of shipping a food truck to a big festival.

For the **kitchen build-out**, you bring welders, saws, spare metal, and a
whole workshop. But the truck that actually shows up at the festival should be
**clean, light, locked, and ready to serve** — no welders sleeping in the back,
no loose knives lying around, and the driver is a **trusted staff member**, not
someone with the master keys to everything.

A production Dockerfile for Node.js is that festival-ready food truck:

- Build it in a messy workshop stage (compilers, dev tools).
- Ship a clean, minimal truck (tiny runtime image).
- The person inside runs as a **regular worker, not the owner with all keys**
  (a non-root user).
- There's a **health inspector** who periodically checks "is the kitchen still
  serving?" (HEALTHCHECK).
- When the festival closes and staff yell "pack up NOW," the worker actually
  hears it and closes cleanly (proper signal handling with tini/dumb-init).

---

## The Linux kernel feature underneath

A production Node.js container leans on **three** kernel mechanisms. You must
understand all three to see why the template is shaped the way it is.

### 1. User namespaces & UIDs — the `USER` instruction

A container is just a Linux process with namespaced views (Topic 02). Inside
the container, a process runs as some **UID** (numeric user id). If you don't
set `USER`, that UID is **0 = root**. Root inside the container maps (by
default, without user-namespace remapping) to a powerful identity that, if the
container is breached or a bind mount is misused, can damage host files or
escalate. Running as a non-root UID (e.g. `node`, UID 1000) means even if an
attacker gets code execution, they're a limited user. This is enforced by the
kernel's normal permission checks on syscalls — the same checks that stop a
normal Linux user from writing to `/etc`.

### 2. PID 1 and signal delivery — why tini/dumb-init exist

Inside a container's **PID namespace**, the first process is **PID 1**. The
Linux kernel treats PID 1 specially:

- The kernel does **not** install default signal handlers for PID 1. So if your
  Node process is PID 1 and has no `SIGTERM` handler, the kernel **drops**
  `SIGTERM` — the process ignores `docker stop` until the 10-second timeout,
  then gets `SIGKILL`ed (data loss risk).
- PID 1 is responsible for **reaping zombie children** (calling `wait()` on
  dead child processes). Node is not designed to be an init system; orphaned
  child processes (e.g. from `child_process.spawn`) can pile up as zombies in
  the process table.

```
/proc/1/status              ← shows PID 1's state inside the container
/proc/1/cmdline             ← what actually runs as PID 1
```

`tini` (or `dumb-init`) is a tiny, correct **init process** that becomes PID 1,
forwards signals to your Node app, and reaps zombies. Docker even ships tini —
enable it with `docker run --init` or bake it in.

### 3. Shell form vs exec form — how the kernel exec()s your CMD

```
CMD node dist/server.js         ← "shell form": runs /bin/sh -c "node ..."
                                   → sh is PID 1, node is a CHILD; signals hit sh
CMD ["node","dist/server.js"]   ← "exec form": kernel execve()s node directly
                                   → node is PID 1 (or tini's direct child)
```

Shell form inserts `/bin/sh` between the init and your app, and `sh` usually
does **not** forward `SIGTERM` to its child. That's why exec form
(`["...","..."]`) is mandatory in production — covered in Topic 07. The kernel
`execve()` syscall replaces the process image; with exec form your node process
is the one that receives signals.

---

## What is this?

A production Dockerfile for Node.js is a **multi-stage** Dockerfile that
produces a **small, secure, correctly-behaving** image: it separates build from
runtime, installs only production dependencies with `npm ci`, runs as a
**non-root** user, has a **HEALTHCHECK**, and uses a proper **init** (tini) so
signals and zombies are handled. It's the battle-tested template you copy into
every Node service.

---

## Why does it matter for a backend developer?

Everything that goes wrong with Node containers in production traces back to a
missing piece of this template:

- **"Deploys take forever."** → No multi-stage build; 1.8 GB images.
- **"`docker stop` / pod termination hangs for 10 seconds and drops
  in-flight orders."** → Node is PID 1 with no signal handling; SIGTERM
  ignored; SIGKILL kills mid-request. (Topic 16, and Kubernetes Topic 46.)
- **"Security flagged our image: runs as root."** → No `USER` instruction.
- **"Kubernetes keeps routing traffic to a dead pod."** → No HEALTHCHECK /
  readiness signal (Topics 26 and 43).
- **"`npm install` gave us different versions than the lockfile."** → Used
  `npm install` instead of `npm ci`.
- **"Zombie processes filling the process table."** → No init reaping children.

Get this template right once and you stop firefighting these forever.

---

## The physical reality

After building the production `orders-api` image, here's what exists on disk and
in the process table.

```
$ docker images orders-api
REPOSITORY   TAG   IMAGE ID       SIZE
orders-api   prod  9a1f...        198MB       ← small, thanks to multi-stage

# The image's config records the non-root user, healthcheck, and entrypoint:
$ docker inspect orders-api:prod --format \
  'user={{.Config.User}} entry={{.Config.Entrypoint}} cmd={{.Config.Cmd}}'
user=node entry=[/usr/bin/tini --] cmd=[node dist/server.js]

$ docker inspect orders-api:prod --format '{{json .Config.Healthcheck}}'
{"Test":["CMD","node","healthcheck.js"],"Interval":30000000000,"Timeout":3000000000,"Retries":3,"StartPeriod":5000000000}
```

Run it, then inspect the **inside** of the container:

```
$ docker run -d --name oa orders-api:prod

# PID 1 is tini (correct init), NOT node directly:
$ docker exec oa cat /proc/1/cmdline | tr '\0' ' '
/usr/bin/tini -- node dist/server.js

# The node process runs as UID 1000 (node), not 0 (root):
$ docker exec oa ps -o pid,uid,comm
  PID  UID  COMMAND
    1    0  tini            ← init as root is fine; it drops to node for the app
    7 1000  node            ← YOUR app runs unprivileged

# Health state is tracked by the daemon:
$ docker inspect --format '{{.State.Health.Status}}' oa
healthy
```

The layers on disk under `/var/lib/docker/overlay2/` contain only alpine + prod
node_modules + your compiled `dist/` — no compilers, no devDependencies.

---

## How it works — step by step

Trace of building and running the production `orders-api` image:

1. **`deps` stage installs production node_modules.** `FROM node:20-alpine AS
   deps`, copy `package*.json`, run `npm ci --omit=dev`. This resolves the exact
   versions in `package-lock.json` and installs only runtime deps into
   `/app/node_modules`. Committed as layers, cached until the lockfile changes
   (Topic 08).

2. **`build` stage compiles the app.** `FROM node:20-alpine AS build`, copy
   `package*.json`, run `npm ci` (ALL deps, including typescript), copy source,
   run `npm run build` → `/app/dist`. This stage is heavy and thrown away.

3. **`runtime` stage assembles the final image.** `FROM node:20-alpine AS
   runtime`. Install `tini` via `apk`. Set `NODE_ENV=production`. Set
   `WORKDIR /app`.

4. **COPY order is deliberate for cache + ownership.** Copy `node_modules` from
   `deps`, copy `dist` from `build`, copy `package.json`. Use
   `--chown=node:node` so files are owned by the non-root user (otherwise
   they're root-owned and the app may fail to write where it's allowed to).

5. **Drop privileges with `USER node`.** Everything after runs as UID 1000. The
   `node` user already exists in official Node images.

6. **HEALTHCHECK registered.** Docker will exec `node healthcheck.js` every 30s.
   The script hits `http://localhost:3000/health`; exit 0 = healthy, non-zero =
   unhealthy. After `retries` failures the container is marked `unhealthy`.

7. **ENTRYPOINT = tini, CMD = node.** `ENTRYPOINT ["/usr/bin/tini","--"]` makes
   tini PID 1. `CMD ["node","dist/server.js"]` is passed as tini's arguments.
   tini execs node, forwards signals to it, and reaps zombies.

8. **At `docker stop` / pod deletion:** the daemon sends `SIGTERM` to PID 1
   (tini) → tini forwards `SIGTERM` to node → your `process.on('SIGTERM', ...)`
   handler runs: stop accepting new connections, finish in-flight orders, close
   the Postgres pool and Redis client, then `process.exit(0)`. Clean shutdown,
   no dropped orders.

---

## Exact syntax breakdown

### npm ci (not npm install)

```
RUN npm ci --omit=dev
    │   │  │
    │   │  └─ skip devDependencies (typescript, eslint, jest) — prod only
    │   └──── "clean install": deletes node_modules, installs EXACT lockfile versions
    └──────── run at build time in a layer
```
`npm ci` **requires** a `package-lock.json`, **fails** if it's out of sync with
`package.json`, and is **reproducible** — perfect for images. `npm install` can
silently update versions and mutate the lockfile.

### COPY with ownership

```
COPY --chown=node:node --from=build /app/dist ./dist
     │                 │            │         │
     │                 │            │         └─ destination in runtime stage
     │                 │            └─ source path in the "build" stage
     │                 └─ pull from another stage (Topic 09)
     └─ set owner:group of copied files to the non-root user
```
Without `--chown`, copied files are owned by root. Then `USER node` can read but
often can't write (e.g. can't create a cache dir), causing `EACCES`.

### USER

```
USER node
│    │
│    └─ username (UID 1000 in official node images) OR a raw UID like "1000"
└────── all subsequent RUN/CMD/ENTRYPOINT run as this user
```
Prefer a raw UID (`USER 1000`) for Kubernetes `runAsNonRoot` checks, which
verify the numeric UID is non-zero.

### HEALTHCHECK

```
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
            CMD node healthcheck.js
│           │            │              │                  │          │
│           │            │              │                  │          └─ command; exit 0=healthy, 1=unhealthy
│           │            │              │                  └─ consecutive fails before "unhealthy"
│           │            │              └─ grace period at startup; fails here don't count
│           │            └─ kill the check if it runs longer than this
│           └─ how often to run the check
└─ declares a container health probe the daemon runs
```

### ENTRYPOINT + CMD with tini

```
ENTRYPOINT ["/usr/bin/tini", "--"]
│          │                 │
│          │                 └─ tini's "--" = "everything after is the command to run"
│          └─ exec form (JSON array) — no shell wrapper
└─ the FIXED init that becomes PID 1

CMD ["node", "dist/server.js"]
│   │
│   └─ passed as ARGUMENTS to the ENTRYPOINT → tini runs: node dist/server.js
└─ the default, overridable command
```

---

## Example 1 — basic

A clean, complete production Dockerfile for `orders-api`. Every line commented.
This is the template to memorize.

```dockerfile
# syntax=docker/dockerfile:1

# ============ STAGE 1: deps (production node_modules only) ============
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev              # exact, reproducible, prod-only install

# ============ STAGE 2: build (compile the app) ============
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci                         # ALL deps incl. typescript for building
COPY . .
RUN npm run build                  # tsc/esbuild -> /app/dist

# ============ STAGE 3: runtime (small, secure, correct) ============
FROM node:20-alpine AS runtime

# tini = tiny init for correct PID 1 signal handling + zombie reaping
RUN apk add --no-cache tini

ENV NODE_ENV=production
WORKDIR /app

# Bring ONLY what's needed, owned by the non-root user
COPY --chown=node:node --from=deps  /app/node_modules ./node_modules
COPY --chown=node:node --from=build /app/dist         ./dist
COPY --chown=node:node package.json ./
COPY --chown=node:node healthcheck.js ./

# Drop root: run the app as UID 1000 (node)
USER node

EXPOSE 3000                        # documents the port; does not publish it

# Health probe the daemon runs; feeds Kubernetes/Compose readiness
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# tini becomes PID 1 and forwards signals to node
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "dist/server.js"]
```

The tiny `healthcheck.js` (no curl needed — keeps the image minimal):

```javascript
// healthcheck.js — exit 0 if /health responds 200, else exit 1
const http = require("http");
const req = http.get(
  { host: "127.0.0.1", port: 3000, path: "/health", timeout: 2000 },
  (res) => process.exit(res.statusCode === 200 ? 0 : 1)
);
req.on("error", () => process.exit(1));
req.on("timeout", () => { req.destroy(); process.exit(1); });
```

And the graceful-shutdown handler your `server.js` MUST have for tini's
forwarded SIGTERM to matter:

```javascript
// server.js (excerpt) — clean shutdown so no order is dropped on deploy
const server = app.listen(3000, () => console.log("orders-api on :3000"));

async function shutdown(signal) {
  console.log(`${signal} received, draining...`);
  server.close(async () => {            // stop accepting new connections
    await pgPool.end();                 // close Postgres pool
    await redis.quit();                 // close Redis connection
    process.exit(0);
  });
  // safety net: force-exit if drain hangs
  setTimeout(() => process.exit(1), 10000).unref();
}
process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT",  () => shutdown("SIGINT"));
```

---

## Example 2 — production scenario

**The situation:** `orders-api` runs in Kubernetes. During every deploy, the
on-call sees a spike of failed order requests, and customers get 502s for a few
seconds. Investigation shows two bugs in the old Dockerfile:

```dockerfile
FROM node:20                       # 1.1 GB base
WORKDIR /app
COPY . .                           # copies node_modules, .git, .env too
RUN npm install                    # not npm ci; not prod-only
CMD npm start                      # shell form + npm wrapper = signals swallowed
```

Two failures:
1. `CMD npm start` → runs `/bin/sh -c "npm start"` → `sh` is PID 1, `npm` is a
   child, `node` is a grandchild. On pod deletion Kubernetes sends `SIGTERM` to
   PID 1 (`sh`), which does **not** forward it. Node never drains; after the
   30s `terminationGracePeriod` Kubernetes `SIGKILL`s it mid-request → dropped
   orders and 502s.
2. Huge image → slow pulls → rolling update is slow → longer window of mixed
   old/new pods.

**The distroless production fix** (smallest, no shell, correct signals):

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---- distroless runtime: no shell, no apk, no package manager ----
FROM gcr.io/distroless/nodejs20-debian12 AS runtime
ENV NODE_ENV=production
WORKDIR /app
# distroless includes a built-in "nonroot" user (UID 65532)
COPY --chown=nonroot:nonroot --from=deps  /app/node_modules ./node_modules
COPY --chown=nonroot:nonroot --from=build /app/dist         ./dist
COPY --chown=nonroot:nonroot package.json healthcheck.js ./
USER nonroot
EXPOSE 3000
# distroless has no shell, so HEALTHCHECK must use exec form with the node binary.
# In Kubernetes you'd use an httpGet readiness probe instead (Topic 43).
ENTRYPOINT ["/nodejs/bin/node", "dist/server.js"]
```

**About signals on distroless:** distroless has **no tini**. Two correct
options:
- In **Kubernetes**, the container runtime + `terminationGracePeriodSeconds`
  sends `SIGTERM` directly to your node process (which IS PID 1 here because we
  used exec form), and your `process.on('SIGTERM')` handler drains cleanly. Node
  as PID 1 works *as long as* you install signal handlers — which we did.
- If you still want a real init, run with `docker run --init` (Docker injects
  tini) or keep the alpine+tini variant from Example 1.

**Result:** image dropped to ~140 MB (distroless nodejs base is tiny and has no
shell/attack surface), and because node receives `SIGTERM` directly and drains,
the deploy-time 502s disappeared.

**When to pick which runtime base:**

| Base | Size | Shell? | Best for |
|---|---|---|---|
| `node:20-alpine` + tini | ~140–200 MB | yes (sh) | most apps; easy debugging; native musl deps |
| `node:20-slim` + tini | ~200–260 MB | yes (bash) | glibc native modules (bcrypt, sharp) |
| `distroless/nodejs20` | ~120–160 MB | **no** | max security; K8s with httpGet probes |

---

## Common mistakes

### Mistake 1 — Running as root

```
$ docker inspect orders-api --format '{{.Config.User}}'
                                          ← empty = root (UID 0)
```
Security scanners (and Kubernetes `securityContext: runAsNonRoot: true`) reject
this. In K8s you'll see the pod fail to start:
```
Error: container has runAsNonRoot and image will run as root
```
**Fix:** add `USER node` (or `USER 1000`) and `--chown=node:node` on your
COPYs.

### Mistake 2 — Shell-form CMD swallows SIGTERM

```dockerfile
CMD node dist/server.js         # shell form → sh is PID 1
```
```
$ time docker stop orders-api
# hangs ~10 seconds, then the container is SIGKILLed
orders-api
docker stop  took 10.3s
```
**Root cause:** `sh` (PID 1) doesn't forward SIGTERM; Docker waits the grace
period then SIGKILLs. **Fix:** exec form `CMD ["node","dist/server.js"]` plus
tini as ENTRYPOINT, plus a SIGTERM handler in your code.

### Mistake 3 — Using `npm install` instead of `npm ci`

```
$ docker build .
npm warn ... updated 3 packages          # lockfile silently drifted
```
Two builds of the same commit can produce different node_modules. **Fix:**
`npm ci` — it fails loudly if `package-lock.json` and `package.json` disagree,
and installs exact locked versions.

### Mistake 4 — HEALTHCHECK using curl on an image without curl

```dockerfile
HEALTHCHECK CMD curl -f http://localhost:3000/health || exit 1
```
```
$ docker inspect --format '{{.State.Health.Status}}' orders-api
unhealthy
$ docker logs orders-api        # or inspect health log:
/bin/sh: curl: not found
```
Alpine and distroless don't ship `curl`. **Fix:** use a Node-based
`healthcheck.js` (no extra binaries) as shown above.

### Mistake 5 — COPY without --chown, then EACCES at runtime

```
$ docker run orders-api
Error: EACCES: permission denied, mkdir '/app/.cache'
```
**Root cause:** files copied as root; `USER node` can't write. **Fix:**
`COPY --chown=node:node ...`, and if the app needs a writable dir, create it and
chown it before switching users, or mount a volume (Topic 14).

### Mistake 6 — Zombie processes from spawned children

If `orders-api` spawns workers (`child_process`), and node is PID 1 with no
reaping, dead children become zombies:
```
$ docker exec orders-api ps -o pid,stat,comm | grep Z
  42  Z   node <defunct>          ← zombie piling up
```
**Fix:** tini as PID 1 reaps them automatically (or `docker run --init`).

---

## Hands-on proof

```bash
# 0) Assume orders-api/ with package.json, package-lock.json, src/, and a
#    /health route. Add healthcheck.js and the Dockerfile from Example 1.

# 1) Build the production image
docker build -t orders-api:prod .

# 2) Prove it's small (multi-stage worked)
docker images orders-api:prod            # expect ~150-200 MB, not ~1 GB

# 3) Prove it runs as non-root (UID 1000)
docker run -d --name oa orders-api:prod
docker exec oa id                        # uid=1000(node) gid=1000(node)  ✓

# 4) Prove tini is PID 1 and node is its child
docker exec oa cat /proc/1/cmdline | tr '\0' ' '   # /usr/bin/tini -- node dist/server.js
docker exec oa ps -o pid,comm            # 1 tini, N node

# 5) Prove SIGTERM drains fast (graceful shutdown)
time docker stop oa                      # should be ~1s (drained), NOT ~10s

# 6) Prove HEALTHCHECK works
docker run -d --name oa2 orders-api:prod
sleep 8
docker inspect --format '{{.State.Health.Status}}' oa2   # healthy  ✓
docker inspect --format '{{json .State.Health.Log}}' oa2 # see probe history

# 7) Prove no devDependencies / compilers shipped
docker run --rm orders-api:prod sh -c \
  'ls node_modules | grep -E "typescript|eslint|jest" || echo "prod-only ✓"'

# cleanup
docker rm -f oa oa2
```

---

## Practice exercises

### Exercise 1 — easy
Take a naive single-stage `orders-api` Dockerfile (`FROM node:20`, `COPY . .`,
`npm install`, `CMD npm start`). Rewrite it into the three-stage template from
Example 1. Verify: image is <250 MB, `id` inside shows uid=1000, and
`docker stop` returns in ~1 second.

### Exercise 2 — medium
Add a `/health` endpoint to `orders-api` that returns 200 only when both
Postgres and Redis are reachable, and 503 otherwise. Wire it to the
`healthcheck.js` and HEALTHCHECK. Kill the Postgres container and prove with
`docker inspect --format '{{.State.Health.Status}}'` that the API container
flips to `unhealthy` after `retries` failures.

### Exercise 3 — hard (production simulation)
Ship two runtime variants of `orders-api`: one on `node:20-alpine` + tini, one
on `gcr.io/distroless/nodejs20-debian12`. For each: (a) confirm non-root, (b)
confirm graceful SIGTERM shutdown drains an in-flight request (use a slow route
+ `docker stop` mid-request and check the client gets a completed response, not
a reset), and (c) compare image sizes. Then explain, in three sentences, why the
distroless image has no `HEALTHCHECK` with a shell command and what you'd use
instead in Kubernetes (hint: Topic 43 httpGet probe).

---

## Mental model checkpoint

Answer from memory:

1. Why must the runtime `CMD` be exec form, not shell form?
2. What does tini/dumb-init do that Node-as-PID-1 doesn't do by default?
3. What is the difference between `npm ci` and `npm install`, and why does the
   image want `ci`?
4. Why do you add `--chown=node:node` to your COPY instructions?
5. Why does `HEALTHCHECK CMD curl ...` fail on alpine/distroless, and what's the
   fix?
6. What exactly happens, signal by signal, when Kubernetes deletes your pod, if
   your Dockerfile and code are correct?
7. When would you choose `node:20-slim` over `node:20-alpine`?

---

## Quick reference card

| Instruction / practice | What it does | Key detail |
|---|---|---|
| Multi-stage (`deps`/`build`/`runtime`) | Separate build from ship | Final image tiny (Topic 09) |
| `npm ci --omit=dev` | Exact, reproducible, prod-only install | Needs lockfile; fails on drift |
| `COPY --chown=node:node` | Copy files owned by non-root user | Prevents EACCES after USER |
| `USER node` / `USER 1000` | Drop root privileges | Required for K8s runAsNonRoot |
| `HEALTHCHECK CMD node healthcheck.js` | Daemon-run liveness probe | No curl needed; Node-only |
| `ENTRYPOINT ["/usr/bin/tini","--"]` | Correct PID 1 init | Forwards signals, reaps zombies |
| `CMD ["node","dist/server.js"]` | Exec-form app start | Node receives SIGTERM directly |
| `ENV NODE_ENV=production` | Prod mode for Express/libs | Faster, less logging |
| distroless base | No shell/package manager | Smallest attack surface |

---

## When would I use this at work?

1. **Standardizing every Node service** at your company on one hardened
   Dockerfile so all images are small, non-root, and shut down cleanly during
   Kubernetes rolling updates (Topic 46) — zero dropped orders on deploy.
2. **Passing a security audit** — the scanner requires non-root, no compilers in
   the runtime, and a minimal base; this template checks all three (Topic 23).
3. **Fixing deploy-time 502s** caused by SIGTERM being swallowed, by moving to
   exec form + tini + a graceful `process.on('SIGTERM')` drain of Postgres/Redis
   connections.

---

## Connected topics

- **Study before:** Topic 07 (CMD vs ENTRYPOINT) — exec vs shell form and PID 1.
  Topic 08 (Layer Caching) — why COPY order matters. Topic 09 (Multi-Stage
  Builds) — the backbone of this template. Topic 10 (.dockerignore) — keep the
  context clean.
- **Study after:** Topic 15 (Environment Variables and Secrets) — inject config
  at runtime, never bake it in. Topic 16 (Container Lifecycle) — states and
  signals in detail. Topic 23 (Image Security) — distroless, scanning, read-only
  root. Topic 26 (Health Checks) — HEALTHCHECK deep dive. Topic 43 (Health
  Checks in Kubernetes) — liveness/readiness/startup probes that replace/augment
  HEALTHCHECK.
