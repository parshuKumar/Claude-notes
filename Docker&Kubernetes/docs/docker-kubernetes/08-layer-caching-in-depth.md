# 08 — Layer Caching in Depth
## Section: Docker in Practice

## ELI5 — The Simple Analogy

Imagine you bake a **wedding cake** in fixed steps: bake the sponge, add the filling, ice it, write the couple's names on top. Baking the sponge takes an hour; writing the names takes a minute.

Now the couple changes their names the day before. Do you re-bake the sponge? Of course not — the sponge, filling, and icing are all the same. You only redo the last step: the names.

Docker does exactly this. It keeps the finished result of each step. When you rebuild, it walks the steps from the top and asks at each one: *"Is this step exactly the same as last time, including its inputs?"* As long as the answer is yes, it reuses the saved result instantly. The **first step that differs** — and every step after it — must be redone.

The whole art of layer caching is: **put the slow, rarely-changing steps early, and the fast, frequently-changing steps late.** Bake the sponge before you write the names.

## The Linux kernel feature underneath

Caching leans on the same two mechanisms from Topic 04 and Topic 06:

1. **Content-addressable layers.** Every layer is identified by a SHA-256 digest of its contents. Two builds that produce byte-identical layers produce the **same digest** and can therefore share/reuse the layer on disk under `/var/lib/docker/overlay2/`.

2. **The build cache is a chain of `(parent, instruction, inputs) → resulting layer` records.** The daemon (or BuildKit) keeps, for each cached step, a **cache key** and the layer it produced. On rebuild it recomputes each step's cache key and looks for a stored match.

The crucial rule that makes ordering matter: **the cache key of every step includes the cache key of the step before it (the parent).** So the chain is like a linked hash — change any early link and every later link's key changes too, even if those later instructions are textually identical. This is the same "Merkle chain" idea behind git commits and blockchains: a change anywhere downstream invalidates everything after it.

```
key(step_n) = hash( key(step_{n-1})  +  instruction_text  +  input_hash )
              │                          │                    │
              │                          │                    └─ for COPY/ADD: hash of the files
              │                          └─────────────────────  the literal instruction string
              └────────────────────────────────────────────────  parent's key (the chain!)
```

## What is this?

**Layer caching** is Docker's reuse of previously-built layers when nothing that affects a step has changed. Each Dockerfile instruction is a step with a computed **cache key**; if a layer with that key already exists, Docker reuses it (`CACHED`) instead of executing the instruction. The first step whose key changed is a **cache miss**, and it plus all following steps are rebuilt.

## Why does it matter for a backend developer?

Because it is the difference between a build that takes **10 seconds** and one that takes **6 minutes**, on *every single commit*, for *every developer*, in *every CI run*.

Concretely for `orders-api`:

- A one-line change to `server.js` should NOT reinstall 400 npm packages. With correct ordering it doesn't; with wrong ordering it does — every time.
- CI cost and feedback speed depend on it. Slow builds mean slow PRs, expensive runners, and developers context-switching while they wait.
- It affects **image push/pull size**. Unchanged layers are already in the registry and on nodes, so only changed layers transfer. Bad caching means re-pushing/re-pulling hundreds of MB (Topic 24).

Layer caching is the highest-leverage Dockerfile skill: understanding it once speeds up thousands of future builds.

## The physical reality

The cache lives in the same store as your images:

```
/var/lib/docker/
├── overlay2/<cache-id>/diff/      ← the actual cached layer files
├── image/overlay2/layerdb/        ← layer metadata + digests
└── (with BuildKit) buildkit/       ← BuildKit's own cache records & keys
```

You can list what's cached and reclaimable:

```bash
docker system df -v          # shows images, layers, and reclaimable build cache
docker buildx du             # BuildKit cache usage, entry by entry
```

Every build prints the cache verdict per step:

```
 => [2/5] WORKDIR /app                                  0.0s
 => CACHED [3/5] COPY package.json package-lock.json ./ 0.0s   ← reused
 => CACHED [4/5] RUN npm ci --omit=dev                  0.0s   ← reused (the big win)
 => [5/5] COPY . .                                      0.3s   ← miss (code changed)
```

`CACHED` means "identical key found, zero work." No `CACHED` prefix means the step ran.

## How it works — step by step

Trace of a rebuild of `orders-api` after you edited one line in `server.js`:

1. **Send context.** CLI tars the build context (minus `.dockerignore`, Topic 10) and streams it to the builder.

2. **Step `FROM node:20-alpine`.** Key = digest of the base image reference. Base already pulled → **CACHED**.

3. **Step `WORKDIR /app`.** Key = parent key + `"WORKDIR /app"`. Unchanged → **CACHED**.

4. **Step `COPY package.json package-lock.json ./`.** Key = parent key + instruction + **hash of those two files**. You didn't touch them → same hash → **CACHED**.

5. **Step `RUN npm ci --omit=dev`.** Key = parent key + the literal string `"RUN npm ci --omit=dev"`. Note: `RUN` does NOT look at the actual filesystem result — it only hashes the *command text* and the parent key. Parent (step 4) was cached and the command text is unchanged → **CACHED**. This is the 45-second step you just skipped.

6. **Step `COPY . .`.** Key = parent key + instruction + **hash of all context files**. You changed `server.js`, so the aggregate file hash changed → key changed → **CACHE MISS**. This layer is rebuilt.

7. **Every step after step 6** now has a changed parent key → all rebuilt (here, just `CMD`, which is metadata and instant).

The lesson is baked into steps 4–6: because `RUN npm ci` sits **above** `COPY . .`, editing code never reaches the install step. Flip their order and step 5's parent changes on every code edit — reinstalling every time.

## Exact syntax breakdown

### The cache-key math, annotated

```
key(RUN npm ci)  =  SHA256(  key(COPY package.json)  +  "RUN npm ci --omit=dev"  )
                    │         │                          │
                    │         │                          └─ literal instruction text (NOT the result)
                    │         └───────────────────────────  parent step's key (the chain link)
                    └─────────────────────────────────────  any change here ⇒ new key ⇒ rebuild
```

```
key(COPY . .)    =  SHA256(  key(RUN npm ci)  +  "COPY . ."  +  hash(all copied files)  )
                                                                │
                                                                └─ COPY/ADD ALSO hash file CONTENT+metadata
```

Two rules fall out of this:

- **`RUN` is cached on its command text only** — not on whether the world changed. If `RUN npm ci` should re-run because `package-lock.json` changed, the invalidation must come *through the parent* (the preceding `COPY package-lock.json`). That's precisely why you copy the lockfile first.
- **`COPY`/`ADD` are cached on file content** — change a copied file's bytes (or mode/mtime in some cases) and the step misses.

### The "COPY package.json first" pattern, annotated

```
COPY package.json package-lock.json ./   ← copy ONLY the manifests first
│                                            (small, changes rarely)
RUN  npm ci --omit=dev                    ← install; cached unless a manifest changed
│                                            (slow, ~45s — protect it!)
COPY . .                                  ← copy the rest of the source LAST
                                             (large, changes constantly)
```

Why it works: the expensive `RUN npm ci` sits above the volatile `COPY . .`. Editing `server.js` only busts `COPY . .` and below. `npm ci` re-runs **only** when `package.json`/`package-lock.json` change — exactly when dependencies actually changed.

### Intentional cache busting

Sometimes you *want* to force a step to re-run. A `--build-arg` that changes flows into every later key:

```
ARG CACHEBUST=0
│   │        │
│   │        └─ default; override at build time to force a rebuild from here down
│   └────────── a build arg referenced right before the step you want to bust
└────────────── declaring the ARG
RUN git clone https://... /repo     ← re-runs whenever CACHEBUST changes
```

```bash
docker build --build-arg CACHEBUST=$(date +%s) -t orders-api .   # forces the clone to re-run
```

Or nuke all cache for a build:

```bash
docker build --no-cache -t orders-api:1.0 .    # ignore cache entirely
```

## Example 1 — basic

Two Dockerfiles, same result, wildly different cache behavior.

**❌ Bad ordering — code change reinstalls everything:**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .                  # copies code AND manifests together
RUN npm ci --omit=dev     # its parent (COPY . .) changes on EVERY code edit → always re-runs
CMD ["node", "server.js"]
```

**✅ Good ordering — code change skips install:**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./   # manifests only → parent of install is stable
RUN npm ci --omit=dev                    # cached unless manifests change
COPY . .                                 # volatile code copied last
CMD ["node", "server.js"]
```

Timeline with the good version after editing one line of `server.js`:

```
 => CACHED [2/5] WORKDIR /app
 => CACHED [3/5] COPY package.json package-lock.json ./
 => CACHED [4/5] RUN npm ci --omit=dev      ← 45s SAVED
 => [5/5] COPY . .                          0.3s
build finished in ~2s
```

## Example 2 — production scenario

**The scenario:** `orders-api` CI builds take 6 minutes on every push. Devs complain PRs are slow and the CI bill is high. You investigate.

The current Dockerfile:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .                           # ❌ 1: code + manifests together
RUN apk add --no-cache python3 make g++   # ❌ 2: build deps re-run because parent busts
RUN npm ci                         # ❌ 3: full install every push
RUN npm run build                  # build step
CMD ["node", "dist/server.js"]
```

Every push edits code, which busts `COPY . .`, which busts *all four* following steps — including the `apk add` and `npm ci` — even though dependencies rarely change.

The fixed Dockerfile:

```dockerfile
FROM node:20-alpine
WORKDIR /app

# Rarely-changing system build deps FIRST — now cached across code pushes.
RUN apk add --no-cache python3 make g++

# Manifests next — install caches unless deps actually change.
COPY package.json package-lock.json ./
RUN npm ci

# Volatile source LAST.
COPY . .
RUN npm run build

CMD ["node", "dist/server.js"]
```

Plus, in CI you must **persist the cache between runs** (fresh runners start empty). With BuildKit/buildx and a registry cache:

```bash
docker buildx build \
  --cache-from type=registry,ref=ghcr.io/acme/orders-api:buildcache \
  --cache-to   type=registry,ref=ghcr.io/acme/orders-api:buildcache,mode=max \
  --tag ghcr.io/acme/orders-api:sha-$GIT_SHA \
  --push .
#   --cache-from : import previous cache so CACHED works on a fresh runner
#   --cache-to   : export this build's cache for the next run (mode=max = all layers)
```

Result: on a code-only push, `apk add` and `npm ci` are `CACHED` (imported from the registry), and CI drops from ~6 minutes to ~40 seconds. Dependency-changing pushes correctly rebuild the install layer. (Topic 27 covers CI caching end-to-end.)

## Common mistakes

**1. `COPY . .` before `npm ci`**

Symptom: `npm ci` runs on every build even for a one-character code change.

```
 => [4/5] RUN npm ci    47.1s     ← never shows CACHED
```

Root cause: `COPY . .` is the parent of `RUN npm ci`; any code edit changes the parent's key, invalidating install. Fix: copy `package*.json` and install *before* `COPY . .`.

**2. Expecting a fresh CI runner to have cache**

Symptom: builds are fast locally, always slow in CI.

Root cause: CI runners are ephemeral — `/var/lib/docker` starts empty, so there is nothing to reuse. The Dockerfile ordering is fine; the *cache store* is gone. Fix: export/import cache with `--cache-to`/`--cache-from` (BuildKit) or the CI provider's cache action (Topic 27).

**3. `RUN apt-get update` in its own layer (stale cache trap)**

```dockerfile
RUN apt-get update
RUN apt-get install -y curl      # ❌ can use a STALE package index from a cached `update`
```

Symptom: `Package 'curl' has no installation candidate` or an old version installs.

Root cause: the cached `apt-get update` layer freezes the package index; the later `install` uses that stale index. Fix: combine them in one `RUN`:

```dockerfile
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

**4. A changing value early in the file (e.g., `ARG BUILD_DATE` near the top)**

Symptom: nothing is ever cached.

Root cause: an instruction whose value changes every build (a timestamp, a random SHA) sits early, so its key changes, cascading to every later step. Fix: move volatile args as **late** as possible, right before the step that truly needs them.

**5. Assuming `RUN` notices that a copied file changed when the file is copied *after* it**

```dockerfile
RUN npm ci
COPY package.json ./     # ❌ install ran BEFORE the manifest existed/changed
```

Root cause: `RUN` only hashes its command text + parent; it can't "see" a file copied later. The manifest must be copied *before* the install. Fix: `COPY package.json ./` then `RUN npm ci`.

## Hands-on proof

```bash
# Setup a project with a "heavy" install step we can watch cache.
mkdir cache-demo && cd cache-demo
printf '{"name":"orders-api","version":"1.0.0","dependencies":{}}\n' > package.json
printf '{}\n' > package-lock.json
printf 'console.log("v1");\n' > server.js

cat > Dockerfile <<'EOF'
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN echo "pretend-heavy-install" && sleep 3 && touch installed
COPY . .
CMD ["node","server.js"]
EOF

# First build — everything runs (watch the ~3s sleep).
docker build -t cache-demo .

# Edit ONLY code, rebuild — the install step must show CACHED (no 3s wait).
printf 'console.log("v2");\n' > server.js
docker build -t cache-demo .
#   => CACHED [4/5] RUN echo pretend-heavy-install ...   ← proof: no sleep
#   => [5/5] COPY . .

# Now change a dependency manifest — install MUST re-run.
printf '{"name":"orders-api","version":"1.0.0","dependencies":{"x":"1.0.0"}}\n' > package.json
docker build -t cache-demo .
#   => [4/5] RUN echo pretend-heavy-install ...  3.x s   ← re-ran (sleep is back)

# Inspect the cache store.
docker system df -v | head -n 20
docker history cache-demo
```

You proved all three rules: code-only edits keep `RUN` `CACHED`, a manifest change busts it (because its parent `COPY package.json` changed), and the ordering is what protects the slow step.

## Practice exercises

### Exercise 1 — easy
Build a Dockerfile with `COPY . .` before `RUN npm ci`. Change one line in `server.js`, rebuild, and confirm `npm ci` re-runs (no `CACHED`). Then reorder to copy `package*.json` first, repeat, and confirm `npm ci` now shows `CACHED`. Paste both build outputs.

### Exercise 2 — medium
You have `RUN apt-get update` and `RUN apt-get install -y curl` on separate lines. Demonstrate the stale-index problem conceptually, then combine them into a single `RUN ... && ... && rm -rf /var/lib/apt/lists/*`. Explain, using the cache-key formula, why the two-line version can serve a stale index.

### Exercise 3 — hard (production simulation)
Simulate CI: build `orders-api` once, then delete the local build cache with `docker builder prune -af` (mimicking a fresh runner) and rebuild — observe that nothing is `CACHED`. Now redo it using `docker buildx build --cache-to type=local,dest=/tmp/dc --cache-from type=local,src=/tmp/dc`, prune the daemon cache again, and rebuild importing from `/tmp/dc`. Prove that the `npm ci` layer is `CACHED` even though the daemon's own cache was wiped. Explain why the import made the difference.

## Mental model checkpoint

1. Write the cache-key formula. Why does it include the parent's key?
2. Why must `RUN npm ci` be invalidated *through* a preceding `COPY package.json`, rather than by the filesystem?
3. What exactly does `COPY`/`ADD` hash to compute its key, and how does that differ from `RUN`?
4. Why is a build fast locally but slow on a fresh CI runner, even with a perfect Dockerfile?
5. Where should volatile values (timestamps, build SHAs) go in the file, and why?
6. Name two ways to intentionally bust the cache.
7. Why does combining `apt-get update && install` in one `RUN` avoid a stale index?

## Quick reference card

| Concept | What it means | Key detail |
|---------|---------------|------------|
| Cache key | `hash(parent_key + instruction + inputs)` | Parent link means a change cascades downward |
| `CACHED` in output | Layer reused, zero work | First non-CACHED step = first miss |
| `RUN` cache | Keyed on command **text** + parent | Does NOT inspect the resulting filesystem |
| `COPY`/`ADD` cache | Keyed on **file content** + parent | Change a copied file → miss |
| COPY-manifests-first | Protects the slow install layer | Put `package*.json` + install above `COPY . .` |
| `--no-cache` | Ignore all cache | Full clean rebuild |
| `ARG CACHEBUST=...` | Force a rebuild from a point | Change the arg value to bust downstream |
| `--cache-from/--cache-to` | Persist cache across CI runs | Needed because runners are ephemeral |
| Order rule | Stable/slow early, volatile/fast late | Bake the sponge before writing the names |

## When would I use this at work?

1. **Slashing CI time.** A PR complains builds take 6 minutes. You reorder the Dockerfile (manifests + install before `COPY . .`) and add registry cache import/export — CI drops to under a minute for code-only changes (Topic 27).
2. **Fixing flaky package installs.** A build intermittently installs an old `curl` version. You spot a cached `apt-get update` on its own line and merge it with the install, eliminating the stale-index bug.
3. **Speeding up local dev loops.** Developers rebuild `orders-api` dozens of times a day. Correct layer ordering means each rebuild after a code tweak finishes in a couple of seconds instead of reinstalling dependencies every time.

## Connected topics

- **Study before:** Topic 04 (Images and Layers — content-addressable storage), Topic 06 (Dockerfile in Depth — which instructions create layers), Topic 07 (CMD vs ENTRYPOINT).
- **Study after:** Topic 09 (Multi-Stage Builds — caching across stages), Topic 10 (.dockerignore — smaller context = better COPY caching), Topic 11 (Production Dockerfile for Node.js — the fully-ordered template), Topic 24 (Docker Registry — layer reuse on push/pull), Topic 27 (Docker in CI/CD — persisting cache across runners).
