# 09 — Multi-Stage Builds

## Section: Docker in Practice

---

## ELI5 — The Simple Analogy

Imagine you're baking a cake to give to a friend.

To bake it, you need a **messy kitchen**: flour bags, mixing bowls, an
electric beater, egg shells, dirty spoons, the oven. That's a lot of stuff.

But when you hand the cake to your friend, you don't hand them the whole
kitchen. You put the finished cake on a clean plate and give them **only the
cake**. The flour, the bowls, the egg shells — all of that stays behind.

A **multi-stage build** is exactly this. You use one "messy kitchen" stage to
build your app (compilers, dev tools, the full toolbox). Then you copy **only
the finished cake** — the built app — onto a small, clean plate (a tiny final
image). Everything messy is thrown away. Your friend (the production server)
gets a light, clean plate with just the cake on it.

---

## The Linux kernel feature underneath

Multi-stage builds are not a special kernel feature — they are a **build-time
trick built on top of the same layer/overlay filesystem** you learned about in
Topic 04 (Images and Layers) and Topic 08 (Layer Caching).

Here is the kernel-level reality you must hold in your head:

- Every Docker image is a **stack of read-only layers** unioned together by
  the **overlay2** storage driver (a Linux kernel filesystem called
  `overlayfs`). Each layer lives on disk at
  `/var/lib/docker/overlay2/<layer-id>/diff/`.
- When you `RUN`, `COPY`, or `ADD` in a Dockerfile, the builder creates a new
  layer = a **filesystem diff** (what changed vs the layer below).
- The **final image** is the union of ALL those diffs. If a build tool wrote
  500 MB into an early layer, that 500 MB is baked into the image **forever**,
  even if a later layer deletes the files. Deleting in a later layer only adds
  a "whiteout" marker on top — the bytes still sit in the lower layer and still
  ship in the image.

That last point is the whole reason multi-stage builds exist. In a
single-stage build you **cannot** shrink the image by deleting build tools at
the end — the tools are already frozen into lower layers.

A multi-stage build sidesteps this entirely. Each `FROM` starts a **brand new,
independent layer stack**. `COPY --from=<stage>` reaches into another stack and
copies **just the files you name** into your final stack as a fresh layer. The
build tools were in a *different* stack that never becomes part of the final
image. The kernel's overlayfs only unions the layers of the **last** stage
(unless you target another one — more on that below).

```
Single-stage image (overlay2 stacks unioned into ONE image):

  layer 4  whiteout: "rm -rf /root/.cache"   ← delete marker only
  layer 3  RUN npm run build        (+120 MB build output)
  layer 2  RUN npm ci               (+900 MB node_modules + toolchain)
  layer 1  FROM node:20             (~1.1 GB base)
  ─────────────────────────────────────────────
  FINAL IMAGE = layer1+2+3+4  → ~2 GB, the 900 MB is STILL inside


Multi-stage (two independent stacks; only stack B ships):

  Stack A "build"                Stack B "runtime"  ← this is the image
  ┌───────────────────┐          ┌──────────────────────────┐
  │ RUN npm run build │          │ COPY --from=build /app/dist│  (+12 MB)
  │ RUN npm ci  (900M)│  ──dist─▶│ RUN npm ci --omit=dev (90M)│
  │ FROM node:20      │          │ FROM node:20-alpine (130M) │
  └───────────────────┘          └──────────────────────────┘
   thrown away after build        FINAL IMAGE ≈ 230 MB
```

---

## What is this?

A **multi-stage build** is a single Dockerfile that contains **more than one
`FROM` instruction**. Each `FROM` begins a new "stage" — an isolated build
environment. You do heavy work (installing compilers, building, bundling) in
early stages, then copy only the finished artifacts into a final, minimal stage
using `COPY --from=`. Only the last stage becomes your shipped image.

---

## Why does it matter for a backend developer?

Without multi-stage builds, your `orders-api` image carries dead weight that
hurts you every single day:

- **Huge images.** A naive `node:20` build of `orders-api` is easily
  **1.5–2 GB**. A multi-stage version is **150–250 MB**. That's 8–10× smaller.
- **Slow deploys.** Kubernetes pulls your image on every new node, every scale
  up, every rolling update. A 2 GB image over a slow registry link can add
  **minutes** to each pod start. Multiply by 50 pods and you feel it.
- **Bigger attack surface.** Compilers (`gcc`, `make`, `python`), package
  managers, and shells are **tools an attacker can use** if they break into
  your container. Shipping them in production is handing a burglar a toolbox.
  We cover this in Topic 23 (Image Security).
- **Secret leakage.** Build-time things — private npm tokens, `.git` history,
  SSH keys — can accidentally get baked into a single-stage image. A separate
  build stage that you throw away contains the blast radius.

If you have ever waited 4 minutes for a pod to pull an image during an incident
rollback, you already know why this matters.

---

## The physical reality

Let's see what actually exists on disk during and after a multi-stage build of
`orders-api`.

**During the build**, the builder materializes every stage. With the legacy
builder each stage's layers live under:

```
/var/lib/docker/overlay2/<layer-id>/diff/    ← the actual files of each layer
/var/lib/docker/image/overlay2/layerdb/      ← layer metadata & chain IDs
```

With **BuildKit** (the modern default builder since Docker 23), intermediate
stages live in the **build cache**, not as normal images. You can see them:

```
$ docker buildx du
ID         RECLAIMABLE  SIZE      LAST ACCESSED
abc123...  true         912.4MB   2 minutes ago   ← the "build" stage node_modules
def456...  true         12.1MB    2 minutes ago   ← the built dist/
...
```

**After the build finishes**, only the final stage is registered as a real
image. The build-stage layers are cache — reclaimable, not part of the image:

```
$ docker images orders-api
REPOSITORY   TAG      IMAGE ID       SIZE
orders-api   latest   7f2c9a1b3d4e   214MB     ← ONLY the runtime stage

$ docker history orders-api:latest
IMAGE          CREATED BY                                      SIZE
7f2c9a1b3d4e   CMD ["node" "dist/server.js"]                  0B
<missing>      COPY /app/dist /app/dist  # buildkit           12.1MB   ← from build stage
<missing>      RUN npm ci --omit=dev                          89.3MB
<missing>      COPY package*.json ./                          4.1kB
<missing>      /bin/sh -c #(nop)  WORKDIR /app                0B
<missing>      FROM node:20-alpine                            113MB
```

Notice the `history` shows **only** layers from the runtime stage. The 900 MB
of build tooling from the `build` stage is **nowhere in the image** — it exists
only as reclaimable build cache that `docker builder prune` will delete.

---

## How it works — step by step

Full trace of `docker build -t orders-api .` on this Dockerfile:

```dockerfile
# ---------- Stage 0: build ----------
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build          # produces /app/dist

# ---------- Stage 1: runtime ----------
FROM node:20-alpine AS runtime
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
CMD ["node", "dist/server.js"]
```

1. **CLI reads the Dockerfile and parses stages.** The builder scans for every
   `FROM` and builds a dependency graph. It sees two stages: `build` (index 0)
   and `runtime` (index 1). It notices `runtime` depends on `build` because of
   `COPY --from=build`.

2. **BuildKit computes what's actually needed.** Because only `runtime` is the
   final target, BuildKit works backward. It only executes stages that the
   final target depends on. (If a stage is never referenced, BuildKit skips it
   entirely — a real speedup.)

3. **Stage `build` executes.** `FROM node:20` pulls the full base (~1.1 GB) into
   `/var/lib/docker/overlay2/`. `WORKDIR /app` sets the working dir. `COPY
   package*.json ./` writes a small layer. `RUN npm ci` spawns a container from
   the current layer, runs npm (installing ~900 MB into `/app/node_modules`),
   and commits the result as a new layer diff. `COPY . .` copies your source.
   `RUN npm run build` runs your bundler (tsc/esbuild/webpack), producing
   `/app/dist`, committed as another layer.

4. **Stage `runtime` executes.** `FROM node:20-alpine` starts a **fresh, empty
   layer stack** (~113 MB alpine base). This stack knows nothing about stage
   `build`. `COPY package*.json ./`, then `RUN npm ci --omit=dev` installs
   **only production dependencies** (~90 MB, no devDependencies like
   typescript/eslint).

5. **The magic step — `COPY --from=build /app/dist ./dist`.** BuildKit mounts
   the final filesystem of stage `build` as a read-only source, reads the bytes
   at `/app/dist`, and writes them into the `runtime` stack at `/app/dist` as a
   new layer diff (~12 MB). **Nothing else** from `build` crosses over — not
   node_modules, not the source, not the toolchain.

6. **`CMD` records metadata.** No filesystem change, just image config saying
   "when you run me, exec `node dist/server.js`".

7. **Final image assembled.** Docker registers **only the `runtime` stack** as
   `orders-api:latest`. The `build` stack becomes reclaimable build cache. Final
   size ≈ 214 MB instead of ≈ 2 GB.

---

## Exact syntax breakdown

### Naming a stage

```
FROM node:20 AS build
│    │        │  │
│    │        │  └─ stage NAME (your choice) — reference it later with --from=build
│    │        └──── the AS keyword: "give this stage a name"
│    └───────────── base image + tag for THIS stage (the messy kitchen)
└────────────────── starts a NEW, independent layer stack
```

### Copying from another stage

```
COPY --from=build /app/dist ./dist
│    │            │         │
│    │            │         └─ DESTINATION inside the CURRENT stage
│    │            └─────────── SOURCE path inside the "build" stage's filesystem
│    └──────────────────────── --from=<stage>: read from that stage, not build context
└───────────────────────────── normal COPY, but source is another stage
```

You can also copy **from an external image** you didn't build:

```
COPY --from=nginx:1.27 /etc/nginx/nginx.conf ./nginx.conf
│    │      │           │
│    │      │           └─ path inside that image's filesystem
│    │      └───────────── an image reference (pulled if not present)
│    └──────────────────── --from can name an IMAGE, not just a stage
└───────────────────────── grab a config/binary from a published image
```

### Copying from a stage by index (when unnamed)

```
COPY --from=0 /app/dist ./dist
│           │
│           └─ stage INDEX (0 = first FROM, 1 = second FROM, ...)
└───────────── works even if the stage has no AS name
```
Prefer names (`AS build`) over numbers — reordering stages breaks index refs.

### Building only up to a specific stage — `--target`

```
docker build --target build -t orders-api:builder .
│            │       │      │                     │
│            │       │      │                     └─ build context (this dir)
│            │       │      └─ tag for the produced image
│            │       └──────── name of the stage to STOP at
│            └──────────────── --target: build only up to this stage, ignore later ones
└───────────────────────────── the build command
```

`--target build` produces an image from the `build` stage — useful for running
tests, debugging, or a dev container that *needs* the toolchain.

---

## Example 1 — basic

A minimal two-stage build for `orders-api`. Every line commented.

```dockerfile
# ===== STAGE 1: build =====
FROM node:20 AS build          # full Node image: has npm, python, gcc for native modules
WORKDIR /app                   # all following paths are relative to /app
COPY package*.json ./          # copy ONLY manifests first (cache-friendly, see Topic 08)
RUN npm ci                     # install ALL deps incl. devDeps (typescript, etc.)
COPY . .                       # now copy the rest of the source
RUN npm run build              # compile TS -> JS into /app/dist

# ===== STAGE 2: runtime =====
FROM node:20-alpine AS runtime # tiny alpine base: ~113 MB vs ~1.1 GB
WORKDIR /app
COPY package*.json ./          # manifests again (this stage has its own filesystem)
RUN npm ci --omit=dev          # install ONLY production deps — no typescript/eslint
COPY --from=build /app/dist ./dist  # bring over ONLY the compiled output
CMD ["node", "dist/server.js"] # start the API
```

Build and compare:

```
$ docker build -t orders-api:multi .
$ docker images | grep orders-api
orders-api   multi   214MB     ← vs ~1.9 GB for a single-stage node:20 build
```

---

## Example 2 — production scenario

**The situation:** Your team's `orders-api` uses a native module,
`bcrypt`, which must be **compiled** against the OS during `npm ci` (it has C++
bindings). On day one someone shipped a single-stage `node:20` image. It's
1.8 GB. During a Friday-night incident you tried to roll back, and each of your
40 pods took ~90 seconds just to pull the image over a congested link. The
rollback that should have taken 30 seconds took over 4 minutes while orders
piled up. Post-incident action item: **shrink the image**.

Here is the production multi-stage Dockerfile that fixes it. The trick with
native modules: you must **compile in the build stage** where the toolchain
exists, then copy the **already-compiled** `node_modules` into runtime.

```dockerfile
# ===== deps: install & COMPILE production node_modules =====
FROM node:20 AS deps            # full image HAS python3+make+g++ to build bcrypt
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev           # compiles bcrypt's native .node binary against glibc

# ===== build: compile the TypeScript app =====
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci                      # ALL deps, incl. typescript, for the build
COPY . .
RUN npm run build               # -> /app/dist

# ===== runtime: minimal final image =====
FROM node:20-slim AS runtime    # slim (glibc-based) so the compiled bcrypt still loads
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps  /app/node_modules ./node_modules   # PRE-COMPILED prod deps
COPY --from=build /app/dist         ./dist           # compiled app
COPY package.json ./                                  # for `npm start`/metadata
USER node                       # drop root (see Topic 11)
CMD ["node", "dist/server.js"]
```

**Why `node:20-slim` and not `-alpine` here?** Alpine uses **musl libc**;
`node:20` (used to compile bcrypt) uses **glibc**. A native `.node` binary
compiled against glibc will crash on musl with
`Error relocating ... symbol not found`. So we match the C library: compile on
glibc (`node:20`), run on glibc (`node:20-slim`). This is a real, common trap —
covered again in Common mistakes below.

**Result:** 1.8 GB → ~260 MB. Pull time per pod dropped from ~90 s to ~12 s.
The next rollback took under 40 seconds total.

---

## Common mistakes

### Mistake 1 — Trying to shrink a single-stage image by deleting at the end

```dockerfile
FROM node:20
RUN npm ci && npm run build && rm -rf node_modules   # "cleanup"
```

**What you expect:** small image.
**What you get:** still ~1.5 GB.
**Root cause (kernel level):** `npm ci` wrote node_modules into one overlay2
layer. `rm -rf` runs in the **same** `RUN`, so here it's actually in the same
layer and *does* help — but the moment you split them across two `RUN`s, the
`rm` only adds a **whiteout** in a higher layer; the bytes stay in the lower
layer and ship in the image. You cannot reliably shrink this way. Use a
multi-stage build instead.

### Mistake 2 — musl vs glibc mismatch with native modules

```
$ docker run orders-api
node: Error relocating /app/node_modules/bcrypt/.../bcrypt_lib.node:
  __snprintf_chk: symbol not found
```

**Root cause:** you compiled a native addon in a glibc stage (`node:20`) and
copied it into an alpine (musl) runtime. The dynamic linker can't resolve glibc
symbols against musl.
**Fix:** either compile *and* run on the same libc (build on `node:20`, run on
`node:20-slim`), or compile *and* run on alpine (build on `node:20-alpine` with
`apk add --no-cache python3 make g++`, run on `node:20-alpine`).

### Mistake 3 — Referencing a stage by index after reordering

```dockerfile
COPY --from=0 /app/dist ./dist
```
Later a teammate adds a new stage at the top. Now `--from=0` points at the
**wrong** stage and you get:
```
COPY failed: stat /app/dist: file does not exist
```
**Fix:** always name stages (`AS build`) and use `--from=build`.

### Mistake 4 — Copying node_modules from a stage that has the WRONG deps

```dockerfile
FROM node:20 AS build
RUN npm ci                 # installs ALL deps incl. devDependencies
...
FROM node:20-alpine
COPY --from=build /app/node_modules ./node_modules   # ← drags in typescript, eslint, jest
```
**Symptom:** image is 400 MB bigger than expected; devDependencies present in
production.
**Fix:** use a dedicated `deps` stage with `npm ci --omit=dev`, and copy
node_modules from *that* stage — not from the full build stage.

### Mistake 5 — Forgetting the runtime stage needs its own WORKDIR/files

```dockerfile
FROM node:20-alpine
COPY --from=build /app/dist ./dist
CMD ["node", "dist/server.js"]
```
```
$ docker run orders-api
Error: Cannot find module 'express'
```
**Root cause:** the runtime stage is a **fresh filesystem**. You copied `dist`
but never installed production node_modules in this stage. Each stage starts
empty — you must bring over (or install) everything the app needs.

---

## Hands-on proof

Run these right now to see multi-stage in action with `orders-api`.

```bash
# 1) Make a tiny fake orders-api
mkdir -p orders-api/src && cd orders-api
cat > package.json <<'EOF'
{ "name": "orders-api", "version": "1.0.0",
  "scripts": { "build": "mkdir -p dist && cp src/server.js dist/server.js" },
  "dependencies": { "express": "^4.19.2" },
  "devDependencies": { "typescript": "^5.5.0" } }
EOF
cat > src/server.js <<'EOF'
console.log("orders-api up on :3000");
EOF

# 2) Single-stage (the "bad" way) for comparison
cat > Dockerfile.single <<'EOF'
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
CMD ["node", "dist/server.js"]
EOF

# 3) Multi-stage (the "good" way)
cat > Dockerfile.multi <<'EOF'
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY --from=build /app/dist ./dist
CMD ["node", "dist/server.js"]
EOF

# 4) Build both and compare sizes
docker build -f Dockerfile.single -t orders-api:single .
docker build -f Dockerfile.multi  -t orders-api:multi  .
docker images | grep orders-api
#   orders-api  single  ~1.1GB
#   orders-api  multi   ~140MB   ← proof

# 5) Prove the build tools are NOT in the multi image
docker run --rm orders-api:single  which tsc   # finds it (devDeps shipped)
docker run --rm orders-api:multi   sh -c 'ls node_modules | grep typescript || echo "no typescript ✓"'

# 6) Build ONLY the build stage with --target
docker build -f Dockerfile.multi --target build -t orders-api:builder .
docker run --rm orders-api:builder ls node_modules   # toolchain present here

# 7) See intermediate stages as reclaimable cache
docker buildx du | head
docker history orders-api:multi   # ← only runtime-stage layers appear
```

---

## Practice exercises

### Exercise 1 — easy
Take the single-stage `Dockerfile.single` above and convert it to a two-stage
build. Confirm with `docker images` that the final image is at least 5× smaller.
Confirm with `docker history` that no `npm install` of devDependencies appears
in the final image.

### Exercise 2 — medium
Add a native dependency (`npm install bcrypt`) to `orders-api`. Write a
three-stage Dockerfile (`deps`, `build`, `runtime`) that compiles bcrypt in a
glibc stage and runs it on `node:20-slim`. Verify the container starts without
a `symbol not found` error. Then deliberately switch the runtime base to
`node:20-alpine` and observe the crash — explain why in one sentence.

### Exercise 3 — hard (production simulation)
Your `orders-api` build must also run its **tests** in CI, but tests need
devDependencies and the test image must NOT ship to production. Design a
Dockerfile with stages `deps`, `build`, `test`, and `runtime`. In CI run
`docker build --target test .` to execute tests (fail the build if they fail),
then run `docker build --target runtime .` to produce the shippable image. Prove
that the `runtime` image contains neither the test files nor jest. Bonus: make
BuildKit skip the `test` stage entirely when building `runtime`, and prove it by
watching the build output.

---

## Mental model checkpoint

Answer from memory:

1. Why can't you shrink a single-stage image by adding `rm -rf` in a later
   `RUN` instruction? (Hint: whiteouts and lower layers.)
2. What does each `FROM` instruction start?
3. What exactly does `COPY --from=build /app/dist ./dist` move, and what does it
   leave behind?
4. Which stage becomes your final image by default?
5. What does `docker build --target build` do, and when would you use it?
6. Why would copying `node_modules` from a full build stage bloat your image?
7. Why can a native module compiled on `node:20` crash on `node:20-alpine`?

---

## Quick reference card

| Instruction / flag | What it does | Key detail |
|---|---|---|
| `FROM img AS name` | Starts a new stage with a name | Each stage = independent layer stack |
| `COPY --from=name src dst` | Copy files from another stage | Only the named paths cross over |
| `COPY --from=img:tag src dst` | Copy from an external image | `--from` can name a published image |
| `COPY --from=0 ...` | Copy from stage by index | Fragile — prefer named stages |
| `--target <stage>` | Build only up to that stage | Great for test/dev/debug images |
| `docker history <img>` | Show layers in final image | Only final-stage layers appear |
| `docker buildx du` | Show build cache incl. stages | Intermediate stages are reclaimable |
| `docker builder prune` | Delete reclaimable build cache | Removes old intermediate stages |

---

## When would I use this at work?

1. **Shrinking a TypeScript `orders-api` image** from ~1.8 GB to ~200 MB so
   Kubernetes rolling updates and rollbacks pull fast during incidents.
2. **Compiling native modules** (bcrypt, sharp, node-gyp addons) in a
   toolchain-rich stage, then shipping a slim runtime that has no compilers —
   smaller and safer.
3. **Running tests inside the build** via a `test` stage and `--target test` in
   CI, while guaranteeing the production image never contains jest, test
   fixtures, or devDependencies.

---

## Connected topics

- **Study before:** Topic 04 (Images and Layers) — you must understand layers
  and overlay2 to see *why* deleting doesn't shrink. Topic 06 (Dockerfile in
  Depth) for `FROM`/`COPY` semantics. Topic 08 (Layer Caching) for ordering.
- **Study after:** Topic 10 (.dockerignore) — controls what even enters the
  build context. Topic 11 (Production Dockerfile for Node.js) — the full
  battle-tested template. Topic 23 (Image Security) — distroless and minimal
  bases that pair perfectly with multi-stage.
