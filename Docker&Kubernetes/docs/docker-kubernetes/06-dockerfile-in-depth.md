# 06 — Dockerfile in Depth
## Section: Docker in Practice

## ELI5 — The Simple Analogy

Imagine you are writing a **recipe card** for a robot chef. The robot has never cooked before and remembers nothing. Every line of the card is one instruction: "start from a clean kitchen", "put flour on the counter", "set the oven to 200°C", "when a customer arrives, bake this cake".

A **Dockerfile** is exactly that recipe card. Each instruction tells Docker how to build your app's environment, one step at a time. And here is the clever part: the robot takes a **photo of the kitchen after every important step**. If you change only the last instruction tomorrow, the robot reuses all the earlier photos instead of re-doing the work. Those photos are called **layers**.

So a Dockerfile is a recipe, and building it produces a stack of photos (layers) that together form an **image**.

## The Linux kernel feature underneath

A Dockerfile itself is just a text file — it is not a kernel feature. But the *thing it builds* — an image — is stored using two real kernel-level technologies you met in Topic 02 and Topic 04:

1. **The overlay filesystem (`overlay2` driver).** Each layer is a directory of files on your host disk. The kernel's `overlayfs` module stacks these directories on top of each other so the container sees a single merged filesystem. Layers below are read-only; the top layer is writable.

2. **Content-addressable storage.** Each layer's identity is the SHA-256 hash of its contents. Change one byte, and the hash changes, and it becomes a brand-new layer. This is why caching works (Topic 08).

When you run `docker build`, the daemon (Topic 03: CLI → daemon → containerd → runc) executes each instruction. Instructions that change the filesystem create a **new overlay layer** — a new directory under `/var/lib/docker/overlay2/`. Instructions that only change **metadata** (like `ENV` or `CMD`) do NOT create a filesystem layer; they only write into the image's JSON config.

Here is the physical picture:

```
Dockerfile (text)                 Image on disk
─────────────────                 ─────────────────────────────────────
FROM node:20-alpine   ──build──▶  layer sha256:aaa...  (overlay2 dir)
WORKDIR /app                      (metadata only — no new dir)
COPY package.json .   ──────────▶ layer sha256:bbb...  (overlay2 dir)
RUN npm ci            ──────────▶ layer sha256:ccc...  (overlay2 dir)
COPY . .              ──────────▶ layer sha256:ddd...  (overlay2 dir)
CMD ["node","server.js"]          (metadata only — no new dir)
                                  + config.json (ENV, CMD, USER, ports…)
```

## What is this?

A **Dockerfile** is a plain text file containing a list of instructions. Running `docker build` reads it top-to-bottom and produces an **image**: a read-only, layered filesystem plus a JSON config that says how to start a container from it. Each instruction is a keyword (usually UPPERCASE) followed by arguments.

## Why does it matter for a backend developer?

Because the Dockerfile is *the contract* between your code and every environment it runs in — your laptop, CI, staging, production, a teammate's machine. Get it wrong and you hit classic pain:

- **"Works on my machine"** — you forgot to pin the Node version in `FROM`, so CI built with Node 22 and your app broke.
- **Slow builds** — you put `COPY . .` before `npm ci`, so every code change re-installs all dependencies (Topic 08 fixes this).
- **Huge images** — you used `ADD` with a URL and left build tools in the final image, shipping 1.2 GB instead of 150 MB.
- **Container won't stop gracefully** — you used the shell form of `CMD`, so your Node process runs as PID 1's *child* and never receives `SIGTERM` (Topic 07 goes deep here).
- **Security review fails** — you never added `USER`, so your `orders-api` runs as root inside the container.

Understanding every instruction — and specifically *which ones create layers* — is the difference between a 30-second build and a 6-minute build, and between a 150 MB image and a 1 GB one.

## The physical reality

Let's build a real image and look at what appears on disk. Our running example is **`orders-api`**: a Node.js Express service backed by Postgres and Redis.

Project on disk:

```
orders-api/
├── Dockerfile
├── package.json
├── package-lock.json
├── server.js
└── src/
    └── routes.js
```

After `docker build -t orders-api:1.0 .`, the image metadata lives here:

```
/var/lib/docker/
├── image/overlay2/
│   ├── imagedb/content/sha256/<image-id>   ← the image config JSON
│   └── layerdb/sha256/<layer-id>/           ← per-layer metadata
└── overlay2/
    ├── <cache-id-1>/diff/                    ← files added by layer 1
    ├── <cache-id-2>/diff/                    ← files added by layer 2
    └── <cache-id-3>/diff/                    ← files added by layer 3
```

The image config JSON (metadata, no filesystem) records the results of `ENV`, `CMD`, `ENTRYPOINT`, `WORKDIR`, `USER`, `EXPOSE`, `LABEL`, `VOLUME`, `HEALTHCHECK`. You can see it directly:

```bash
docker inspect orders-api:1.0 --format '{{json .Config}}' | jq
```

The `.diff/` directories under `overlay2/` hold the *actual files* each `RUN`/`COPY`/`ADD` layer added. That is the split you must hold in your head: **filesystem layers on disk vs metadata in the config**.

## How it works — step by step

Here is the full trace of `docker build -t orders-api:1.0 .`:

1. **CLI packs the build context.** The `docker` CLI tars up everything in `.` (the build context — the directory you pointed at), minus whatever `.dockerignore` excludes (Topic 10), and streams it to the daemon over `/var/run/docker.sock`.

2. **Daemon parses the Dockerfile** into an ordered list of instructions.

3. **For each instruction, the daemon computes a cache key** (Topic 08). The key is derived from the instruction text plus, for `COPY`/`ADD`, a hash of the files being copied. If a matching layer already exists, it is reused ("CACHED") and no work is done.

4. **On a cache miss, the daemon executes the instruction:**
   - **Filesystem instructions** (`RUN`, `COPY`, `ADD`) start a temporary container from the previous layer, perform the change, then `commit` the container's writable layer as a new read-only layer — a new dir under `overlay2/`.
   - **Metadata instructions** (`ENV`, `ARG`, `WORKDIR`, `EXPOSE`, `CMD`, `ENTRYPOINT`, `USER`, `VOLUME`, `LABEL`, `HEALTHCHECK`) do NOT run a container. They just update the in-memory image config and produce an "empty" layer (0 bytes) recorded in history.

5. **`RUN` specifically:** the daemon creates a container, runs the command through `/bin/sh -c` (shell form) or directly (exec form), waits for exit code 0, then commits the resulting filesystem diff.

6. **After the last instruction**, the daemon writes the final image config, assigns the image ID (SHA-256 of the config), and tags it `orders-api:1.0`.

7. **Layers are deduplicated.** If two images share the `node:20-alpine` base, that base's layers exist **once** on disk and are referenced by both.

## Exact syntax breakdown

Below is every instruction from Topic 06, annotated. A ✅ **LAYER** tag means it creates a filesystem layer on disk. A ⚙️ **META** tag means it only changes the image config (no filesystem layer).

### FROM — ✅ base layers

```
FROM node:20-alpine AS build
│    │              │  │
│    │              │  └─ stage name (optional) — referenced later by multi-stage builds (Topic 09)
│    │              └──── the "AS" keyword introduces a named build stage
│    └───────────────── image reference: repository:tag  (here node, tag 20-alpine)
└────────────────────── every Dockerfile MUST start with FROM (after optional ARGs)
```

`FROM` pulls the base image. Its layers become the bottom of your stack. `node:20-alpine` pins Node 20 on Alpine Linux (~50 MB) — small and predictable. Prefer a **digest** for reproducibility: `FROM node:20-alpine@sha256:...`.

### RUN — ✅ LAYER (runs a command, commits the result)

```
RUN npm ci --omit=dev
│   │
│   └─ the command executed inside a temporary container built from the previous layer
└───── RUN executes at BUILD time, then commits the filesystem changes as a new layer
```

Two forms:

```
RUN npm ci                 ← shell form:  runs as  /bin/sh -c "npm ci"
RUN ["npm","ci"]           ← exec form:   runs npm directly, no shell (no $VAR expansion, no &&)
```

Shell form lets you chain with `&&` and use variables. Exec form skips the shell (useful in images with no shell, like distroless). Every `RUN` = one layer, so chain related commands with `&&` to keep layer count down.

### COPY — ✅ LAYER (copies from build context into the image)

```
COPY package.json package-lock.json ./
│    │                              │
│    │                              └─ destination inside the image (relative to WORKDIR)
│    └──────────────────────────────── one or more sources from the build context
└──────────────────────────────────── COPY only sees files in the build context (Topic 10)
```

`COPY --chown` sets ownership as it copies (avoids a separate `RUN chown` layer):

```
COPY --chown=node:node . .
│    │       │    │
│    │       │    └─ group
│    │       └────── user
│    └────────────── set ownership of copied files to node:node
└───────────────── copy everything in context to WORKDIR
```

`COPY --from=build /app/dist ./dist` copies from an earlier stage (Topic 09) instead of the build context.

### ADD — ✅ LAYER (like COPY, but with two extra powers)

```
ADD https://example.com/tool.tar.gz /tmp/
│   │                               │
│   │                               └─ destination
│   └─────────────────────────────── source can be a URL, OR a local tar archive
└─────────────────────────────────── ADD auto-extracts local .tar/.tar.gz and can fetch URLs
```

**Rule of thumb: use `COPY`, not `ADD`.** `ADD`'s auto-extraction and URL fetching are surprising and hard to cache correctly. Only use `ADD` when you specifically want to unpack a local tarball into the image. For downloading files, `RUN curl` is clearer and lets you verify checksums.

### ENV — ⚙️ META (persists into the running container)

```
ENV NODE_ENV=production PORT=3000
│   │                   │
│   │                   └─ second variable (space-separated pairs allowed)
│   └───────────────────── KEY=VALUE
└───────────────────────── sets environment variables for BUILD and RUNTIME
```

`ENV` values are visible during later build steps AND inside the final running container (`process.env.NODE_ENV`). They are baked into the image config — **never put secrets here** (they show up in `docker history`, Topic 15).

### ARG — ⚙️ META (build-time only, NOT in the running container)

```
ARG NODE_VERSION=20
│   │           │
│   │           └─ default value (optional)
│   └───────────── variable name, usable only DURING the build
└───────────────── ARG defines a build-time variable, passed via --build-arg
```

Key difference from `ENV`: `ARG` disappears after the build. It does NOT exist in the running container. Use it for build customization:

```
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine    ← ARG before FROM can parameterize the base image
```

Build with `docker build --build-arg NODE_VERSION=22 .`. **Never pass secrets via `ARG`** — they are recorded in image history. Use BuildKit secrets instead (Topic 15).

### WORKDIR — ⚙️ META (sets the working directory)

```
WORKDIR /app
│       │
│       └─ absolute path; created if it doesn't exist
└───────── sets the cwd for all following RUN/CMD/ENTRYPOINT/COPY/ADD
```

`WORKDIR /app` means subsequent `COPY . .` lands in `/app`, and `CMD ["node","server.js"]` runs from `/app`. Always use `WORKDIR` instead of `RUN cd /app` — `cd` in a `RUN` does not persist to the next instruction (each `RUN` is a fresh shell).

### EXPOSE — ⚙️ META (documentation + metadata only)

```
EXPOSE 3000
│      │
│      └─ port number (optionally /tcp or /udp)
└──────── documents which port the app listens on — does NOT publish it
```

`EXPOSE` is **pure documentation**. It does NOT open the port to the host. You still need `docker run -p 3000:3000` (Topic 12) to actually publish. It helps humans and enables `docker run -P` (publish all exposed ports to random host ports).

### CMD — ⚙️ META (default command, overridable) — full detail in Topic 07

```
CMD ["node", "server.js"]
│   │
│   └─ exec form: a JSON array — runs node directly as PID 1 (preferred)
└───── the default command; overridden by anything after `docker run image ...`
```

`CMD` sets the default process. `docker run orders-api:1.0 npm test` replaces it entirely. Only the **last** `CMD` in a Dockerfile takes effect.

### ENTRYPOINT — ⚙️ META (the fixed executable) — full detail in Topic 07

```
ENTRYPOINT ["node", "server.js"]
│          │
│          └─ exec form: the process that ALWAYS runs; run-time args are appended, not replaced
└──────────── makes the image behave like a single executable
```

`ENTRYPOINT` is not overridden by `docker run image args` (those args are *appended* instead). You override it only with `--entrypoint`. Topic 07 covers the CMD+ENTRYPOINT combination in depth.

### USER — ⚙️ META (which UID/GID the process runs as)

```
USER node
│    │
│    └─ username (must exist in the image) or numeric UID, e.g. USER 1000
└────── all following RUN/CMD/ENTRYPOINT run as this user, not root
```

The `node:20-alpine` image already ships a `node` user (UID 1000). Adding `USER node` means your `orders-api` process is **not root** inside the container — a critical security control. Put `USER` *after* the `RUN npm ci` steps that may need root to install, and use `COPY --chown=node:node` so the app files are owned correctly.

### VOLUME — ⚙️ META (declares a mount point)

```
VOLUME ["/var/lib/postgresql/data"]
│      │
│      └─ path inside the container that should be backed by a volume
└──────── marks this path as external storage; Docker creates an anonymous volume if none given
```

`VOLUME` tells Docker "data written here should live outside the container's writable layer." At runtime Docker creates an **anonymous volume** for that path unless you mount your own. Use it sparingly in app images — it can surprise you by creating orphaned anonymous volumes. It matters most for database images (Topic 14).

### LABEL — ⚙️ META (arbitrary metadata key/value)

```
LABEL org.opencontainers.image.source="https://github.com/acme/orders-api"
│     │                                 │
│     │                                 └─ value
│     └───────────────────────────────── key (use reverse-DNS / OCI conventions)
└───────────────────────────────────── attaches metadata queryable via docker inspect
```

Labels are searchable metadata: maintainer, git commit, source repo, version. They cost nothing at runtime and help tooling (`docker images --filter "label=..."`).

### HEALTHCHECK — ⚙️ META (how Docker tests if the container is healthy) — full detail in Topic 26

```
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
│           │              │            │
│           │              │            └─ mark unhealthy after 3 consecutive failures
│           │              └─────────────── each check must return within 3s
│           └──────────────────────────── run the check every 30s
  CMD wget -qO- http://localhost:3000/health || exit 1
  │   │
  │   └─ exit 0 = healthy, exit 1 = unhealthy
  └───── the command Docker runs INSIDE the container to test health
```

`HEALTHCHECK` makes Docker periodically run a command inside the container. The container's status becomes `healthy` / `unhealthy`, visible in `docker ps`. Orchestrators and Compose `depends_on: condition: service_healthy` use it (Topic 20, 26).

## Example 1 — basic

A minimal, correct Dockerfile for `orders-api`. Every line commented.

```dockerfile
# Start from a pinned, small base image (Node 20 on Alpine Linux). ✅ base layers
FROM node:20-alpine

# Set the working directory. All later COPY/RUN happen here. ⚙️ metadata only
WORKDIR /app

# Copy ONLY the dependency manifests first, so the npm-install layer
# can be cached even when app code changes (Topic 08). ✅ layer
COPY package.json package-lock.json ./

# Install exactly the locked production dependencies. ✅ layer
RUN npm ci --omit=dev

# Now copy the rest of the source. Changing code busts only THIS layer. ✅ layer
COPY . .

# Document the listening port (does not publish it). ⚙️ metadata only
EXPOSE 3000

# Set production mode. ⚙️ metadata only
ENV NODE_ENV=production

# Drop from root to the unprivileged built-in node user. ⚙️ metadata only
USER node

# Default process, exec form so node is PID 1 and receives signals (Topic 07). ⚙️ metadata only
CMD ["node", "server.js"]
```

Build and run:

```bash
docker build -t orders-api:1.0 .
docker run -d -p 3000:3000 --name orders orders-api:1.0
curl localhost:3000/health
```

## Example 2 — production scenario

**The scenario:** Your team's `orders-api` image is 1.1 GB, builds take 5 minutes on every push, security scanning flags it running as root, and it downloads a CLI tool from a URL with `ADD`. You are asked to fix all of it in one Dockerfile review.

Here is the "before" — the problematic Dockerfile:

```dockerfile
FROM node:20                       # ❌ full image (~1 GB), not alpine
ADD https://get.tool.io/cli /usr/local/bin/cli   # ❌ ADD URL, no checksum, no chmod, cache-unfriendly
COPY . .                           # ❌ copies everything BEFORE install → cache always busts
RUN npm install                    # ❌ not `npm ci`, installs dev deps too
RUN chmod +x /usr/local/bin/cli    # ❌ extra layer
CMD npm start                      # ❌ shell form → node runs as child of sh, no SIGTERM (Topic 07)
```

The "after" — reviewed and corrected:

```dockerfile
# Small, pinned base. Cuts ~850 MB immediately.
FROM node:20-alpine

WORKDIR /app

# Fetch the tool with RUN so we can verify a checksum and control the layer.
RUN wget -qO /usr/local/bin/cli https://get.tool.io/cli \
 && echo "<sha256>  /usr/local/bin/cli" | sha256sum -c - \
 && chmod +x /usr/local/bin/cli

# Dependency manifests first → install layer stays cached across code edits.
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# App code last.
COPY --chown=node:node . .

ENV NODE_ENV=production
EXPOSE 3000

# Health endpoint so Compose/K8s know when we're ready (Topic 26/43).
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

# Run unprivileged.
USER node

# Exec form → node is PID 1 → receives SIGTERM for graceful shutdown.
CMD ["node", "server.js"]
```

Result: image drops from ~1.1 GB to ~180 MB, builds cache the install layer (5 min → ~15 s on code-only changes), scanner passes (non-root), and the container shuts down cleanly on `docker stop`.

## Common mistakes

**1. `COPY . .` before installing dependencies**

```dockerfile
COPY . .
RUN npm ci
```

Symptom: every code change triggers a full re-install.

```
=> [3/4] COPY . .          0.4s
=> [4/4] RUN npm ci        47.2s   ← re-runs on EVERY code edit
```

Root cause: the `COPY . .` layer's cache key includes a hash of *all* files. Change one line of `server.js` and the hash changes, busting that layer and everything after it — including `npm ci`. Fix: copy `package*.json` first, install, then copy the rest (Topic 08).

**2. Using shell form for `CMD`**

```dockerfile
CMD node server.js
```

Symptom: `docker stop orders` hangs for 10 seconds, then the container is killed hard.

```
$ docker stop orders
orders          ← returns only after the 10s SIGKILL timeout
```

Root cause: shell form runs `/bin/sh -c "node server.js"`. `sh` becomes PID 1; `node` is its child. `docker stop` sends `SIGTERM` to PID 1 (`sh`), which does not forward it to `node`. Node never runs its shutdown handler. Fix: exec form `CMD ["node","server.js"]` (Topic 07).

**3. Baking a secret into `ENV` or `ARG`**

```dockerfile
ENV DATABASE_PASSWORD=supersecret
```

Symptom: anyone can read it:

```
$ docker history --no-trunc orders-api:1.0
... ENV DATABASE_PASSWORD=supersecret
```

Root cause: `ENV`/`ARG` values are stored in the image config and history in plaintext. Anyone who pulls the image sees them. Fix: pass secrets at runtime with `-e` / `--env-file` (Topic 15), or BuildKit `--secret` for build-time.

**4. `ADD` with a URL expecting caching or extraction**

```dockerfile
ADD https://example.com/data.tar.gz /data
```

Symptom: the URL is refetched even when unchanged, and `.tar.gz` from a URL is NOT auto-extracted (only *local* tarballs are).

Root cause: `ADD` treats remote URLs differently from local files — remote files are not extracted, and cache behavior on URLs is unreliable. Fix: `COPY` for local files, `RUN curl/wget` with checksum for remote.

**5. Forgetting `WORKDIR` and using `RUN cd`**

```dockerfile
RUN cd /app
COPY package.json .     # ❌ lands in / , not /app
```

Root cause: each `RUN` is a new shell; `cd` does not persist to later instructions, and `COPY` never respected the `cd` anyway. Fix: `WORKDIR /app`.

## Hands-on proof

Run these right now to *see* which instructions create layers and where they live on disk.

```bash
# 1. Create a tiny project.
mkdir orders-api && cd orders-api
printf '{"name":"orders-api","version":"1.0.0"}\n' > package.json
printf 'console.log("orders-api up");\n' > server.js

cat > Dockerfile <<'EOF'
FROM node:20-alpine
WORKDIR /app
COPY package.json ./
RUN echo "installed" > installed.txt
COPY server.js ./
ENV NODE_ENV=production
USER node
CMD ["node","server.js"]
EOF

# 2. Build.
docker build -t orders-api:1.0 .

# 3. See the history — note which steps have a SIZE (filesystem layers)
#    and which show 0B (metadata only).
docker history orders-api:1.0
#   IMAGE  ...  CREATED BY                       SIZE
#          ...  CMD ["node" "server.js"]         0B      ← META
#          ...  USER node                        0B      ← META
#          ...  ENV NODE_ENV=production          0B      ← META
#          ...  COPY server.js ./                 41B     ← LAYER (has size)
#          ...  RUN echo installed > ...          10B     ← LAYER
#          ...  COPY package.json ./              43B     ← LAYER
#          ...  WORKDIR /app                     0B      ← META
#          ...  /bin/sh -c #(nop) CMD ["node"]   ...     ← base layers below

# 4. Prove metadata lives in the config, not the filesystem.
docker inspect orders-api:1.0 --format '{{json .Config.Env}}'
docker inspect orders-api:1.0 --format '{{.Config.User}}'
docker inspect orders-api:1.0 --format '{{json .Config.Cmd}}'

# 5. See the actual layer directories on disk (Linux host / Docker Desktop VM).
docker inspect orders-api:1.0 --format '{{json .GraphDriver.Data}}' | jq
#   → LowerDir / UpperDir point into /var/lib/docker/overlay2/<id>/diff
```

You just verified the core mental model: `COPY`/`RUN` have SIZE (they are filesystem layers under `overlay2/`), while `ENV`/`USER`/`CMD`/`WORKDIR` are 0B (metadata in the config JSON).

## Practice exercises

### Exercise 1 — easy
Write a Dockerfile for a Node app that: starts from `node:20-alpine`, sets `WORKDIR /app`, copies `package.json` then runs `npm ci --omit=dev`, copies the rest, exposes port 3000, sets `NODE_ENV=production`, runs as user `node`, and starts with `node server.js` in exec form. Then run `docker history` and identify every 0B (metadata) instruction.

### Exercise 2 — medium
Take this deliberately-bad Dockerfile and fix all issues, explaining each:
```dockerfile
FROM node:20
COPY . .
RUN npm install
ENV API_KEY=abc123
ADD https://example.com/x.tar.gz /x
CMD npm start
```
Your fixed version must build a smaller image, cache the install step, keep the secret out of the image, and stop gracefully.

### Exercise 3 — hard (production simulation)
Your `orders-api` needs a build-time argument for the git commit SHA (to embed in `/app/version.txt`), must NOT leak any secret, must run as a non-root user, must include a `HEALTHCHECK` that hits `/health`, and must be reproducible. Write the Dockerfile using `ARG` for the commit, `LABEL org.opencontainers.image.revision` set from that ARG, and prove with `docker inspect` that the label is set and with `docker history` that no secret appears. Then rebuild changing only `server.js` and confirm the `npm ci` layer shows `CACHED`.

## Mental model checkpoint

Answer from memory:

1. Which instructions create a filesystem layer under `/var/lib/docker/overlay2/`, and which only change the image config JSON?
2. What is the difference between `ENV` and `ARG` at runtime?
3. Why does `RUN cd /app` not affect the next instruction?
4. Does `EXPOSE 3000` make the port reachable from your host? What actually does?
5. When would you legitimately choose `ADD` over `COPY`?
6. Why is exec-form `CMD` important for signal handling?
7. Where do the results of `USER`, `CMD`, and `LABEL` end up, and how do you read them back?

## Quick reference card

| Instruction | What it does | Creates a layer? | Key detail |
|-------------|--------------|------------------|------------|
| `FROM` | Sets the base image | ✅ base layers | Must be first; pin the tag/digest |
| `RUN` | Executes a command at build time | ✅ layer | Chain with `&&` to reduce layers |
| `COPY` | Copies from build context into image | ✅ layer | Prefer over ADD; supports `--chown`, `--from` |
| `ADD` | COPY + URL fetch + local tar extract | ✅ layer | Avoid unless extracting a local tarball |
| `ENV` | Sets env var (build + runtime) | ⚙️ meta | Baked into image — no secrets |
| `ARG` | Build-time-only variable | ⚙️ meta | Gone at runtime; no secrets |
| `WORKDIR` | Sets working directory | ⚙️ meta | Use instead of `RUN cd` |
| `EXPOSE` | Documents a listening port | ⚙️ meta | Does NOT publish; still need `-p` |
| `CMD` | Default (overridable) command | ⚙️ meta | Only the last one applies |
| `ENTRYPOINT` | Fixed executable | ⚙️ meta | Run args are appended, not replaced |
| `USER` | Sets UID/GID for later steps | ⚙️ meta | Use to run non-root |
| `VOLUME` | Declares an external mount point | ⚙️ meta | Creates anonymous volume if unmounted |
| `LABEL` | Attaches metadata | ⚙️ meta | Use OCI reverse-DNS keys |
| `HEALTHCHECK` | Command to test container health | ⚙️ meta | Sets healthy/unhealthy status |

## When would I use this at work?

1. **Cutting CI time.** A teammate's PR doubles build time. You open the Dockerfile, see `COPY . .` before `npm ci`, reorder it, and the `npm ci` layer starts hitting cache — CI drops from 5 minutes to 20 seconds.
2. **Passing a security review.** Compliance requires containers not run as root. You add a `USER node` line and `COPY --chown=node:node`, rebuild, and `docker inspect` now shows a non-root user — audit passes.
3. **Debugging a container that won't die.** Deploys hang on rollout because pods take the full termination grace period to stop. You find shell-form `CMD npm start`, switch to exec form `CMD ["node","server.js"]`, and graceful shutdown works — rollouts speed up (Topic 07, 46).

## Connected topics

- **Study before:** Topic 02 (Linux Foundations — namespaces, cgroups, overlayfs), Topic 04 (Images and Layers — how layers are stored), Topic 05 (Your First Container).
- **Study after:** Topic 07 (CMD vs ENTRYPOINT — the deep dive on the two you just met), Topic 08 (Layer Caching in Depth — why order matters), Topic 09 (Multi-Stage Builds), Topic 10 (.dockerignore), Topic 11 (Production Dockerfile for Node.js — the complete template).
