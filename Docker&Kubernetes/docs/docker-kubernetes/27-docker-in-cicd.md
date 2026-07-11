# 27 — Docker in CI/CD

## Section: Production Patterns

---

## ELI5 — The Simple Analogy

Imagine a bakery that ships identical birthday cakes to shops all over the country. You do NOT want each shop baking its own cake from a scrambled memory of the recipe — one shop uses too much sugar, another forgets the eggs, and customers get wildly different cakes.

So you build **one central kitchen**. Every time someone improves the recipe, that central kitchen bakes **one** perfect cake, takes a photograph of it, and stores the photo in a **catalog** with a label ("Recipe #a3f9, baked July 12"). Every shop then just orders "the cake in photo #a3f9." They all get the exact same cake, because it's literally the same bake, copied.

**CI/CD with Docker is that central kitchen.**
- The **recipe change** = a git push / pull request.
- The **central kitchen** = GitHub Actions (a clean, throwaway computer in the cloud).
- **Baking the cake** = `docker build` — turning your code into an image.
- **The photograph** = the image, frozen and immutable.
- **The catalog** = a container **registry** (Topic 24) like Docker Hub or GitHub Container Registry.
- **The label** = the image **tag** (we'll tag by the git commit SHA, so every cake is traceable to the exact recipe change that made it).

The whole point: **build the image ONCE, in a clean room, label it with exactly which code made it, store it, and let every environment run that identical frozen artifact.** No "works on my machine." No shop improvising.

---

## The Linux kernel feature underneath

CI/CD itself is a workflow, not a kernel feature — but the thing that makes fast, reliable image builds in CI possible *is* deeply mechanical, and it's worth seeing the reality.

**1. The build happens inside a container, on a fresh kernel namespace set.**
A GitHub Actions "runner" is a virtual machine. When it runs `docker build`, BuildKit (Docker's modern build engine) executes each build step inside its own throwaway container — the same namespace + cgroup isolation from Topic 02. The runner boots **empty every single time**: fresh disk, no Docker cache, nothing. This is the core problem CI caching solves — a clean room is reproducible but *forgetful*.

**2. Layers are content-addressed blobs (recap from Topic 04).**
Every image layer is a tarball of filesystem changes, identified by the **SHA-256 hash of its contents**:

```
layer digest = sha256(<the tar of this layer's file changes>)
```

Because the identity of a layer *is* its content hash, two builds that produce byte-identical layers produce the **same digest**. That's what makes caching possible across machines: if the runner can prove "I already have a layer with digest `sha256:abc…`", it can skip rebuilding it and just reuse the blob. The cache key is math, not a filename.

**3. BuildKit's content-addressable cache + the registry as a cache store.**
This is the crucial trick for CI. Because the fresh runner has no local cache, BuildKit can **export the cache to an external place** (a registry, or GitHub's own cache service) and **import it on the next run**. Physically:

```
Local Docker daemon cache:   /var/lib/docker/buildkit/   ← wiped when the runner dies
Exported cache (cache-to):   pushed to a registry as extra manifest/blobs
                             OR stored in GitHub Actions Cache (the "gha" backend)
Next run (cache-from):       BuildKit fetches those cached layer blobs by digest
                             and reuses any whose inputs are unchanged
```

So "layer caching in CI" is literally: *serialize BuildKit's content-addressed cache to a network store, then pull it back into the next clean-room build and match by digest.* The kernel gives you isolated, reproducible builds; content-addressing gives you the ability to carry a cache between otherwise-amnesiac machines.

---

## What is this?

**Docker in CI/CD** means automating the build → tag → push of your container image using a CI system (here, GitHub Actions), so that every code change produces a versioned, immutable image in a registry without any human running `docker build` by hand. CI (Continuous Integration) tests and builds on every push; CD (Continuous Delivery/Deployment) publishes and (optionally) deploys the result.

---

## Why does it matter for a backend developer?

Because the alternative — a human building and pushing images from a laptop — is a slow, unreliable, insecure disaster, and you will feel every one of these pains:

- **"Works on my machine" ships to production.** Your laptop has Node 20.3, a leftover `node_modules`, an env var you forgot you set. You `docker build` locally and push. Prod behaves differently. A clean CI runner builds from *only what's committed to git* — if it's not in the repo, it's not in the image. This alone eliminates a huge class of bugs.

- **No traceability.** Someone pushed `orders-api:latest` three weeks ago. Which commit is running in prod? Nobody knows. If you tag every image by its **git SHA**, prod is running `orders-api:a3f9c21` and you can `git checkout a3f9c21` to see the exact code — forever.

- **Secrets on laptops.** Manual pushes mean registry credentials live on developer machines. CI uses short-lived, scoped tokens injected at runtime and never written to disk.

- **Slow feedback.** Builds without caching in CI can take 5–10 minutes (reinstalling every npm dependency from scratch, every time). Proper layer caching (the heart of this topic) cuts that to under a minute, so developers actually get fast feedback.

- **Deploy is a scary manual ritual.** With CI/CD, merging a PR *is* the deploy trigger. It's boring, repeatable, and reversible (roll back = deploy the previous SHA's image, which still exists in the registry).

For `orders-api`, this is the bridge between "I wrote code" and "the exact bytes I wrote are running in production, traceably." Everything in the Kubernetes half of this course (especially Topic 52) assumes images arrive in a registry this way.

---

## The physical reality

Let's make CI concrete — it's not magic, it's a Linux VM running scripts.

**The runner (the clean room):**
```
GitHub-hosted runner = an ephemeral Ubuntu VM
 ├─ Docker + BuildKit preinstalled
 ├─ Fresh disk every run (no cache carried over unless YOU export/import it)
 ├─ A working directory with your repo checked out: /home/runner/work/<repo>/<repo>
 └─ Destroyed the instant the job finishes — nothing persists
```

**What a build produces and where it goes:**
```
docker build ──▶ image in the runner's local /var/lib/docker/  (temporary!)
     │
     └─ docker push ──▶ registry (GHCR/Docker Hub), stored as:
                        manifests/  (the recipe: which layers, in order)
                        blobs/      (the actual layer tarballs, by sha256)
```

**Where the cache physically lives between runs (with `cache-to type=gha`):**
```
GitHub Actions Cache service (managed blob storage tied to your repo)
 └─ scope: keyed per branch/ref, ~10 GB total per repo (LRU-evicted)
    holds BuildKit's exported layer blobs so the NEXT run imports them
```

**The image reference you end up with:**
```
ghcr.io/acme/orders-api:a3f9c21
│       │    │          │
│       │    │          └─ tag = the git short SHA (7 chars) → immutable, traceable
│       │    └─ image name (repository)
│       └─ org / user namespace
└─ registry hostname (GitHub Container Registry)
```

That `ghcr.io/acme/orders-api:a3f9c21` string is the whole payoff: an immutable coordinate pointing at exact bytes, produced by exact code, stored forever, runnable anywhere.

---

## How it works — step by step

Full trace of what happens when a developer opens a PR and then merges it. Assume the workflow file below is in the repo.

1. **Developer pushes a branch / opens a PR.** GitHub receives the push, looks in `.github/workflows/` for any workflow whose `on:` trigger matches (`pull_request`), and queues a job.

2. **GitHub allocates a fresh runner VM.** Empty Ubuntu, Docker + BuildKit ready, nothing else. A clean room.

3. **`actions/checkout` clones your repo** into `/home/runner/work/orders-api/orders-api` at the exact commit. Now the runner has your code — and *only* your code, nothing from anyone's laptop.

4. **`docker/setup-buildx-action` starts a BuildKit builder.** The default Docker builder has limited cache export; Buildx gives you the full BuildKit engine that supports `cache-to`/`cache-from`. This is what makes CI caching work at all.

5. **(CI stage) Build + test.** BuildKit runs the Dockerfile. Because we imported cache with `cache-from`, unchanged layers (base image, `npm ci`) are **restored from the GitHub Actions cache** and skipped — only layers after your changed files rebuild. Then tests run (either `RUN npm test` as a build stage, or a separate step). If tests fail, the job stops here and **nothing is pushed**. This is the "gate."

6. **Decision: is this a merge to `main`, or just a PR?**
   - **PR (not merged):** build and test only — prove the image *can* be built and passes. Usually **do not push** (you don't want half-baked PR images cluttering the registry, though some teams push PR preview images).
   - **Merge to `main`:** proceed to push.

7. **(CD stage) Log in to the registry.** `docker/login-action` uses a short-lived token (`GITHUB_TOKEN` for GHCR, or a stored secret for Docker Hub) to authenticate. The token is injected into the runner's memory, never committed.

8. **Tag the image.** The `docker/metadata-action` computes tags from git context: the **short SHA** (`a3f9c21`), often also a branch tag and `latest`. Multiple tags can point at the same image digest.

9. **Push to the registry.** BuildKit uploads any layer blobs the registry doesn't already have (deduped by digest — recap Topic 24), then the manifest. Now `ghcr.io/acme/orders-api:a3f9c21` exists permanently.

10. **Export the cache (`cache-to`).** BuildKit serializes its layer cache to the GitHub Actions cache so the *next* run starts warm.

11. **(Optional CD) Trigger a deploy.** A later step (or a separate workflow / GitOps tool — Topic 52) tells the target environment "run image `…:a3f9c21`." Kubernetes pulls that exact digest and rolls it out.

12. **Runner is destroyed.** Everything local vanishes. The only durable outputs are: the image in the registry, and the cache in GitHub's cache service. Perfectly reproducible next time.

---

## Exact syntax breakdown

### The `docker/build-push-action` step (the heart of it)

```yaml
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: ghcr.io/acme/orders-api:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

```
    uses: docker/build-push-action@v6
    │     └─ the official action that wraps `docker buildx build --push` for you
    └─ run a prebuilt, versioned action (v6 = major version pin)

    with:
      context: .
      │        └─ the build context (Topic 10): the dir sent to BuildKit; . = repo root
      │
      push: true
      │     └─ true = run `docker push` after building; false = build only (use for PRs)
      │
      tags: ghcr.io/acme/orders-api:${{ github.sha }}
      │     │                        └─ github.sha = the FULL 40-char commit SHA of this run
      │     └─ registry/namespace/name:tag  → the immutable coordinate
      │
      cache-from: type=gha
      │           └─ IMPORT cache from GitHub Actions Cache before building (warm start)
      │              other backends: type=registry,ref=…  |  type=local,src=…
      │
      cache-to: type=gha,mode=max
      │         │        └─ mode=max = cache EVERY layer incl. intermediate build-stage layers
      │         │           mode=min (default) = only cache layers of the FINAL image
      │         └─ EXPORT the cache to GitHub Actions Cache after building
      └─ so the NEXT run's cache-from finds these layers
```

**`mode=min` vs `mode=max` — this matters for multi-stage builds (Topic 09):**
```
mode=min : caches only layers that end up in the final image.
           A multi-stage build's `builder` stage (npm ci, compile) is NOT cached
           → those expensive steps re-run every time. Fast to store, slow to build.

mode=max : caches ALL layers, including the throwaway builder stage.
           → `npm ci` in the builder stage is restored from cache. Bigger cache,
             MUCH faster rebuilds. Almost always what you want in CI for Node apps.
```

### Cache backend types (the `type=` values)

```
cache-to: type=gha,mode=max
          │
          ├─ type=gha       → GitHub Actions Cache service. Zero setup on GH-hosted
          │                    runners, ~10GB/repo, LRU-evicted. Best default for GH Actions.
          │
          ├─ type=registry,ref=ghcr.io/acme/orders-api:buildcache,mode=max
          │                 → store the cache AS AN IMAGE in your registry. Portable
          │                    across any CI system; you manage its lifecycle. Good when
          │                    not on GitHub, or when you want cache shared org-wide.
          │
          ├─ type=local,dest=/tmp/.buildx-cache
          │                 → write cache to a directory (pair with actions/cache).
          │                    Older pattern; type=gha superseded it.
          │
          └─ type=inline    → embed cache metadata INTO the pushed image itself.
                               Simple, but only mode=min; no separate cache artifact.
```

### The trigger and permissions block

```yaml
on:
  push:
    branches: [main]        # run the full build+push when commits land on main
  pull_request:
    branches: [main]        # run build+test (no push) on PRs targeting main

permissions:
  contents: read            # read the repo code
  packages: write           # allowed to PUSH to GHCR (ghcr.io) — required!
```

```
on:
  push:
    branches: [main]
    │         └─ only trigger on pushes to main (i.e., merges). Not every branch.
    └─ the "merge to main → build & push" trigger
  pull_request:
    branches: [main]
    └─ the "someone opened a PR into main → build & test only" trigger

permissions:
  packages: write
  │         └─ grants the auto-provided GITHUB_TOKEN permission to push to GHCR.
  │            WITHOUT this line, docker push to ghcr.io → 403 denied.
  └─ scoped permissions for THIS workflow's GITHUB_TOKEN (least privilege)
```

### `docker/metadata-action` — computing good tags automatically

```yaml
- id: meta
  uses: docker/metadata-action@v5
  with:
    images: ghcr.io/acme/orders-api
    tags: |
      type=sha,prefix=,format=short     # → a3f9c21  (7-char short SHA)
      type=ref,event=branch             # → main    (the branch name)
      type=raw,value=latest,enable={{is_default_branch}}  # → latest, only on main
```

```
    tags: |
      type=sha,prefix=,format=short
      │        │        └─ format=short = 7-char SHA (default is the full 40 chars)
      │        └─ prefix= (empty) = don't prepend "sha-"; give a bare a3f9c21
      └─ generate a tag from the commit SHA → the immutable, traceable tag
      type=raw,value=latest,enable={{is_default_branch}}
      │        │            └─ only add :latest when building the default branch (main)
      │        └─ a literal tag value
      └─ a hand-specified tag
```

**Why tag by SHA and not just `latest`?**
```
:latest        → a MOVING pointer. Yesterday's latest ≠ today's latest.
                 You can never say "which code is in prod?" reliably. Rollback is guesswork.
:a3f9c21       → an IMMUTABLE pointer to one exact commit's build. Forever.
                 Prod runs :a3f9c21 → git checkout a3f9c21 shows the exact source.
                 Rollback = redeploy :<previous-sha>, which still exists in the registry.
```
Best practice: push **both** — `:a3f9c21` for traceability/rollback, and `:latest` (only on `main`) for convenience. Deploy by SHA, never by `latest`.

---

## Example 1 — basic

The smallest workflow that builds `orders-api` and pushes it tagged by SHA on every push to `main`. Every line commented.

**`.github/workflows/docker.yml`**:

```yaml
name: build-orders-api                 # shows up in the GitHub "Actions" tab

on:
  push:
    branches: [main]                   # trigger only when commits land on main

permissions:
  contents: read                       # read repo source
  packages: write                      # allow pushing to ghcr.io

jobs:
  build:
    runs-on: ubuntu-latest             # the clean-room runner VM
    steps:
      # 1. Get the code onto the runner
      - uses: actions/checkout@v4

      # 2. Start a full BuildKit builder (needed for cache-to/from)
      - uses: docker/setup-buildx-action@v3

      # 3. Log in to GitHub Container Registry using the auto-provided token
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}       # the user/bot that triggered the run
          password: ${{ secrets.GITHUB_TOKEN }}  # auto-injected, short-lived, no setup

      # 4. Build the image and push it, tagged by the commit SHA
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository_owner }}/orders-api:${{ github.sha }}
          cache-from: type=gha              # warm start from last run's cache
          cache-to: type=gha,mode=max       # save cache for next run
```

Push a commit to `main`, open the **Actions** tab, and watch it build. When it's green:

```bash
# The image now exists in the registry, tagged by SHA:
docker pull ghcr.io/acme/orders-api:$(git rev-parse HEAD)
docker run -p 3000:3000 ghcr.io/acme/orders-api:$(git rev-parse HEAD)
```

That `$(git rev-parse HEAD)` prints the full SHA — the exact tag CI used.

---

## Example 2 — production scenario

**The situation:** The `orders-api` team has grown to 8 engineers. Problems piling up:
- CI takes **7 minutes** per run because `npm ci` reinstalls 900 packages from scratch every time. Engineers context-switch waiting for green.
- A bad deploy went out and nobody could tell which commit was live, so rollback took 40 minutes of guessing.
- PRs sometimes merge even though the image doesn't actually build, breaking `main`.

You build a **complete, production-grade workflow** that: caches aggressively, runs tests as a gate on PRs, and on merge to `main` pushes an image tagged by SHA + `latest`, then verifies the image's health check before declaring success.

**`.github/workflows/orders-api.yml`** — the complete file:

```yaml
name: orders-api CI/CD

on:
  push:
    branches: [main]          # merges → build, test, push, deploy-trigger
  pull_request:
    branches: [main]          # PRs → build + test only (the gate)

# Least-privilege token for THIS workflow
permissions:
  contents: read
  packages: write             # push to ghcr.io

# Cancel an in-progress run if a newer commit arrives on the same ref (saves minutes)
concurrency:
  group: orders-api-${{ github.ref }}
  cancel-in-progress: true

env:
  IMAGE: ghcr.io/${{ github.repository_owner }}/orders-api

jobs:
  build-test-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up BuildKit builder
        uses: docker/setup-buildx-action@v3

      # ---- CI: build a TEST image and run the test suite as a gate ----
      # We build the "test" stage of a multi-stage Dockerfile (see note below),
      # load it locally (not pushed), and run the tests inside it.
      - name: Build test image
        uses: docker/build-push-action@v6
        with:
          context: .
          target: test          # build only up to the `test` stage (Topic 09)
          load: true            # load into the runner's local Docker, don't push
          tags: orders-api:test
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Run tests (the gate)
        run: docker run --rm orders-api:test npm test
        # If tests fail, this step fails, the job fails, and on a PR nothing gets pushed.

      # ---- CD: only on push to main, log in + push the RUNTIME image ----
      - name: Log in to GHCR
        if: github.event_name == 'push'        # skip login/push on PRs
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Compute tags
        if: github.event_name == 'push'
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=sha,prefix=,format=short       # a3f9c21  → traceable, deploy by this
            type=raw,value=latest,enable={{is_default_branch}}  # latest only on main

      - name: Build and push runtime image
        if: github.event_name == 'push'
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          target: runtime        # the final, minimal runtime stage
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # ---- Post-push smoke test: does the pushed image actually run & report healthy? ----
      - name: Smoke test the pushed image
        if: github.event_name == 'push'
        run: |
          SHA_TAG=$(git rev-parse --short HEAD)
          docker run -d --name smoke -p 3000:3000 "$IMAGE:$SHA_TAG"
          # give it time, then hit the /healthz we built in Topic 26
          for i in $(seq 1 15); do
            if docker exec smoke curl -fs http://localhost:3000/healthz; then
              echo "healthy"; docker rm -f smoke; exit 0
            fi
            sleep 2
          done
          echo "image never became healthy"; docker logs smoke; docker rm -f smoke; exit 1

  # Separate deploy job — runs ONLY after a successful push-to-main build
  deploy:
    needs: build-test-push        # wait for build/test/push to succeed
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - name: Trigger deployment
        run: |
          echo "Deploying $IMAGE:$(echo ${{ github.sha }} | cut -c1-7)"
          # In reality: kubectl set image / call ArgoCD / render Helm — see Topic 52.
          # Deploy by SHA, NEVER by :latest, so the running version is unambiguous.
```

The **multi-stage Dockerfile** this workflow expects (ties together Topics 09, 11, 26):

```dockerfile
# ---------- deps: install everything, incl. dev deps for testing ----------
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci                    # ALL deps (this expensive layer gets cached via mode=max)

# ---------- test: the gate stage ----------
FROM deps AS test
COPY . .
# `docker run orders-api:test npm test` runs the suite; build target=test stops here.

# ---------- runtime: minimal production image ----------
FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev         # prod deps only → smaller image
COPY . .
RUN apk add --no-cache curl   # for the health check
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:3000/healthz || exit 1     # from Topic 26
USER node
CMD ["node", "server.js"]
```

**What this buys the team:**
- **CI drops from 7 min to ~1 min** on unchanged dependencies, because `npm ci` is restored from the `type=gha` cache (`mode=max` caches the `deps` stage). Change only `server.js` and the `npm ci` layer is a cache hit.
- **`main` can't break:** the `test` stage runs on every PR *before* merge; a failing test = failing check = can't merge.
- **Total traceability:** every prod image is `orders-api:<sha>`. "What's running?" → read the tag → `git checkout` it. Rollback = redeploy the previous SHA.
- **No dead images ship:** the smoke test proves the pushed image boots and its Topic-26 health check goes green before deploy runs.

---

## Common mistakes

### Mistake 1 — no `cache-from`/`cache-to`, so every build is cold

Your workflow just does `docker/build-push-action` with no cache lines. Each run starts on a **fresh runner with an empty `/var/lib/docker`**, so `npm ci` downloads all 900 packages every single time. 7-minute builds.

**Symptom:** the `npm ci` / `RUN` step takes minutes on *every* run, even when you only changed a comment.

**Root cause:** the runner is amnesiac (see The physical reality). Without exporting/importing BuildKit's content-addressed cache, there is nothing to reuse.

**Right:**
```yaml
cache-from: type=gha
cache-to: type=gha,mode=max      # mode=max is essential for multi-stage Node builds
```

---

### Mistake 2 — `mode=min` on a multi-stage build (cache "works" but is useless)

You added caching but rebuilds are still slow. You used the default `mode=min`.

**Symptom:** the *final* image layers are cached, but the expensive `builder`/`deps`-stage `npm ci` re-runs every time.

**Root cause:** `mode=min` only caches layers that survive into the final image. In a multi-stage build the `npm ci` happens in an intermediate stage that gets discarded, so `min` never caches it.

**Wrong:** `cache-to: type=gha` (defaults to min)
**Right:** `cache-to: type=gha,mode=max`

---

### Mistake 3 — forgetting `permissions: packages: write` → 403 on push

```
ERROR: failed to push ghcr.io/acme/orders-api:a3f9c21:
denied: installation not allowed to Write organization package
```
Your build succeeds but the push fails with `denied` / `403`.

**Root cause:** the default `GITHUB_TOKEN` is read-only for packages unless you explicitly grant write. It's a least-privilege default.

**Right:** add to the workflow (or job):
```yaml
permissions:
  contents: read
  packages: write
```
(Also make sure the package's visibility/permissions in GitHub allow the repo to push.)

---

### Mistake 4 — deploying `:latest`, then unable to roll back or know what's live

You tag and deploy only `:latest`. A bad build ships. To roll back you'd deploy... `:latest`? Which is now the bad one. And "what commit is in prod?" has no answer.

**Symptom:** frantic rollback, redeploying `latest` gives you the same broken image; no way to pin the previous good version.

**Root cause:** `:latest` is a *mutable* pointer — it always means "the newest thing," so it carries no version identity.

**Right:** always also tag by SHA and **deploy by SHA**. Rollback becomes trivial: deploy `orders-api:<previous-sha>`, which still exists immutably in the registry (Topic 24).

---

### Mistake 5 — putting secrets in build args (baking them into the image)

```dockerfile
ARG DB_PASSWORD                      # ← wrong for secrets
ENV DB_PASSWORD=$DB_PASSWORD
```
```yaml
build-args: DB_PASSWORD=${{ secrets.DB_PASSWORD }}   # ← leaks into image layers
```
The password is now baked into an image layer. Anyone who pulls the image can run `docker history` / extract layers and read it. Build args are **not** secret — they're visible in image metadata.

**Root cause:** `ARG`/`ENV` values persist in the image's layer metadata (Topic 04). CI secrets injected as build args become permanent image contents.

**Right:** never bake secrets into images. Provide config at **runtime** via env vars / mounted secrets (Topic 15), and for build-time secrets (e.g. a private npm token) use BuildKit secret mounts which are **not** persisted:
```yaml
- uses: docker/build-push-action@v6
  with:
    secrets: |
      npmtoken=${{ secrets.NPM_TOKEN }}
```
```dockerfile
RUN --mount=type=secret,id=npmtoken \
    NPM_TOKEN=$(cat /run/secrets/npmtoken) npm ci   # token used, never stored in a layer
```

---

## Hands-on proof

You can prove the *mechanics* locally with `docker buildx` — the exact engine GitHub Actions uses — without any cloud.

```bash
# 1. Create a buildx builder (same engine CI uses via setup-buildx-action)
docker buildx create --use --name proof
docker buildx inspect --bootstrap

# 2. First build, EXPORTING cache to a local dir (simulates cache-to)
docker buildx build \
  --tag orders-api:local \
  --cache-to type=local,dest=/tmp/bkcache,mode=max \
  --load .
#   → watch every layer run: npm ci downloads everything. Note the time.

# 3. Change ONLY server.js (a source file, not package.json)
echo "// touch $(date)" >> server.js

# 4. Rebuild, IMPORTING that cache (simulates cache-from on a fresh runner)
docker buildx build \
  --tag orders-api:local \
  --cache-from type=local,src=/tmp/bkcache \
  --cache-to type=local,dest=/tmp/bkcache,mode=max \
  --load .
#   → the `npm ci` layer prints "CACHED" — restored by digest, not re-run.
#     Only the COPY . . and later layers rebuild. This is the CI speedup, proven.

# 5. See the cache blobs physically on disk:
ls -la /tmp/bkcache
#   → index.json + blobs/  (content-addressed layer tarballs, by sha256)

# 6. Tag by "SHA" the way CI does, and inspect the immutable digest:
SHA=$(git rev-parse --short HEAD)
docker tag orders-api:local orders-api:$SHA
docker inspect --format '{{index .RepoDigests 0}}{{.Id}}' orders-api:$SHA
#   → the sha256 image ID is the content-addressed identity CI pushes.

# cleanup
docker buildx rm proof
```

What you verified:
- BuildKit caching is real content-addressing: change a source file and the `npm ci` layer is a **CACHED** hit matched by digest.
- The cache is a serializable set of blobs you can carry between machines — exactly how `type=gha` moves cache between amnesiac runners.
- The image identity is a sha256 hash — the immutable thing a SHA tag points at.

---

## Practice exercises

### Exercise 1 — easy
Write a minimal `.github/workflows/docker.yml` that, on every push to `main`, checks out the repo, sets up Buildx, logs in to GHCR, and builds+pushes `orders-api` tagged with `${{ github.sha }}`. Push it, watch it run in the Actions tab, then `docker pull` the resulting SHA-tagged image and run it locally. Confirm the tag equals your commit SHA.

### Exercise 2 — medium
Take the Exercise 1 workflow and add `cache-from: type=gha` and `cache-to: type=gha,mode=max`. Run it once (cold). Then push a commit that changes only a source file (not `package.json`) and run it again. Compare the build times and find the `CACHED` lines in the log for the dependency-install layer. In one paragraph, explain *why* changing `package.json` would have invalidated that cache but changing `server.js` did not (hint: recall layer caching, Topic 08).

### Exercise 3 — hard (production simulation)
Build the full Example 2 pipeline for `orders-api`:
1. A multi-stage Dockerfile with `deps`, `test`, and `runtime` stages plus a Topic-26 `HEALTHCHECK`.
2. A workflow that runs the `test` stage as a **gate on PRs** (no push) and, on merge to `main`, pushes an image tagged with both the short SHA and `latest`, then **smoke-tests** the pushed image by running it and curling `/healthz` until healthy.
3. Deliberately break a test in a PR and confirm the check fails and the image is NOT pushed. Fix it, merge, and confirm the SHA-tagged image appears in GHCR and passes the smoke test.
4. Finally, write the one command you would run to **roll back** to the previous commit's image, and explain why that image still exists and is safe to deploy.

---

## Mental model checkpoint

Answer from memory:
1. Why is building in a fresh CI runner *better* than building on a developer's laptop, even though it's slower without cache?
2. What is a "layer digest" and why does it make cache portable between machines?
3. What's the difference between `cache-to: mode=min` and `mode=max`, and which do you need for a multi-stage Node build?
4. Why tag by git SHA instead of (only) `:latest`? How does it make rollback trivial?
5. What single line is missing if `docker push` to GHCR returns `403 denied`?
6. Why must you never pass a secret as a Docker build arg? Where should secrets go instead?
7. On a PR (not a merge), which stages of the Example 2 pipeline run, and which are skipped?

---

## Quick reference card

| Action / field | What it does | Key detail |
|---|---|---|
| `on: push: branches: [main]` | Trigger build+push on merges | The "deploy on merge" hook |
| `on: pull_request:` | Trigger build+test on PRs | The "gate before merge" hook |
| `permissions: packages: write` | Let GITHUB_TOKEN push to GHCR | Omit → `403 denied` |
| `actions/checkout@v4` | Clone repo onto runner | Builds from committed code only |
| `docker/setup-buildx-action@v3` | Start full BuildKit builder | Required for `cache-to`/`cache-from` |
| `docker/login-action@v3` | Auth to the registry | Uses short-lived token, never on disk |
| `docker/build-push-action@v6` | Build + optionally push | `push: true/false`, `target:`, `load:` |
| `cache-from: type=gha` | Import cache before build | Warm start on an amnesiac runner |
| `cache-to: type=gha,mode=max` | Export cache after build | `mode=max` caches intermediate stages |
| `type=registry,ref=…:buildcache` | Store cache as a registry image | Portable across any CI system |
| `docker/metadata-action@v5` | Compute tags from git context | `type=sha,format=short` → `a3f9c21` |
| `tags: …:${{ github.sha }}` | Tag by commit | Immutable, traceable, deploy by this |
| `secrets:` + `--mount=type=secret` | Build-time secret, not baked in | Never use build-args for secrets |
| `concurrency: cancel-in-progress` | Kill superseded runs | Saves runner minutes |

---

## When would I use this at work?

1. **Every merge auto-ships a traceable image.** Your team merges a PR to `main`; within a minute `ghcr.io/acme/orders-api:<sha>` exists, tested and smoke-checked. No one ever runs `docker build` by hand for prod again, and "what's running?" is always answerable by reading a tag.

2. **Fast, trustworthy PR feedback.** A teammate opens a PR; CI builds the image (warm from cache in ~1 min) and runs the test suite inside it as a gate. Broken code physically cannot merge, and reviewers see a green check that means "this actually builds and passes."

3. **One-command, safe rollback.** A bad deploy goes out. Because every build is tagged by SHA and stored immutably in the registry, you roll back by deploying the previous SHA's image — no rebuild, no guessing, seconds not minutes. This is the exact pattern Topic 52 automates with GitOps in Kubernetes.

---

## Connected topics

- **Study before:**
  - Topic 04 — Images and Layers (content-addressed layers = why caching is portable).
  - Topic 08 — Layer Caching in Depth (what invalidates a layer; ordering instructions for cache hits).
  - Topic 09 — Multi-Stage Builds (the `test`/`runtime` split the pipeline relies on).
  - Topic 11 — Production Dockerfile for Node.js (the image CI actually builds).
  - Topic 24 — Docker Registry (where pushed images live; tagging strategy, digest dedup).
  - Topic 26 — Health Checks (the `/healthz` the smoke test verifies).
- **Study after:**
  - Topic 52 — CI/CD with Kubernetes (GitOps: take the SHA-tagged image this topic produces and deploy it to a cluster with ArgoCD/Flux or `kubectl` in Actions).
  - Topic 53 — RBAC (least-privilege service accounts/tokens for the CI system that deploys).
