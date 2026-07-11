# 24 — Docker Registry

## Section: Production Patterns

---

## ELI5 — The Simple Analogy

Think of a public library. You (a developer) write a book (build an image). You don't hand a photocopy to every friend who wants to read it. Instead you **donate it to the library** (push to a registry). Now anyone with a library card can **borrow a copy** (pull) whenever they want, from anywhere.

The library has some rules that make it work:

- **Every book has a call number** — a precise code so two people asking for "the blue one" get the *exact same book*. That precise code is the image **digest** (`sha256:...`).
- **Books also have friendly titles** on the spine, like "Orders API, 2nd Edition." That's the **tag** (`orders-api:1.4.0`). Titles can be re-glued to a different book; call numbers cannot.
- **Some libraries are public** (Docker Hub — anyone can borrow). **Some are private** (your company's — you need a badge to enter). GHCR and ECR are private libraries.

A Docker registry is that library for container images. Pushing donates, pulling borrows, tags are the friendly spine labels, and digests are the unforgeable call numbers.

---

## The Linux kernel feature underneath

A registry is not a kernel feature — it's an **HTTP service** (a web server) that speaks the **OCI Distribution Spec** (formerly the Docker Registry HTTP API v2). But the *content model* underneath is pure content-addressable storage, the same idea the kernel/filesystem uses for deduplication, and the same model from Topic 04.

Here is the real machinery:

**1. Content-addressable storage (CAS).** Every blob (a layer tarball, or the config JSON) is stored under the SHA-256 hash of its own bytes. The address *is* the content's fingerprint. Change one byte → different hash → different address. This is why a **digest is immutable**: `sha256:9b2a...` can only ever refer to those exact bytes. It's the same principle as git commit hashes.

**2. The registry HTTP API.** `docker push` and `docker pull` are just a sequence of HTTP requests to endpoints defined by the OCI spec:

```
GET  /v2/                                   → check the registry supports v2 + auth
HEAD /v2/<name>/blobs/<digest>              → "do you already have this layer?"
POST /v2/<name>/blobs/uploads/              → start uploading a layer
PUT  /v2/<name>/manifests/<tag-or-digest>   → publish the manifest (the index)
GET  /v2/<name>/manifests/<reference>       → pull the manifest
```

**3. The manifest — a JSON index, not the data.** When you pull, the *first* thing you get is a small JSON **manifest** listing the digests of the config and each layer. Docker then fetches each layer blob by digest. The manifest is the "table of contents"; layers are the "chapters."

**4. Authentication via bearer tokens.** `docker login` exchanges your username/password (or a token) for a short-lived **bearer token** stored on disk. Every push/pull sends `Authorization: Bearer <token>`. Kernel-wise this is just TLS sockets and file reads — the security is in the protocol, not the kernel.

So when someone says "the registry," picture a plain web server storing hash-named files, and Docker speaking a well-defined HTTP dialect to it.

---

## What is this?

A Docker registry is a server that stores and distributes container images. `docker push` uploads an image (its manifest plus layer blobs) to the registry; `docker pull` downloads it. Docker Hub is the default public registry; GHCR (GitHub) and ECR (AWS) are common private ones. Images are named `registry/namespace/repo:tag`, and each build also has an immutable content **digest** you can pin instead of a mutable tag.

---

## Why does it matter for a backend developer?

Because your laptop is not production. You build `orders-api` on your machine, but Kubernetes on a cluster somewhere needs to *run* it — and it can only run images it can pull from a registry. The registry is the bridge between "it works on my machine" and "it's running in prod."

Without understanding registries, these things bite you:

- Kubernetes shows `ImagePullBackOff` and you don't know it's an auth or tag problem.
- You deploy `orders-api:latest`, a teammate pushes a new `latest`, and now three nodes are running three *different* builds all called "latest" — an un-debuggable ghost.
- A rollback is impossible because you overwrote the old tag.
- You leak your app to the world by pushing to a public Docker Hub repo instead of a private one.
- You can't answer "exactly which commit is running in prod right now?" because your tags aren't tied to git.

Getting tagging, digests, and immutability right is what makes deploys **reproducible** and rollbacks **instant**.

---

## The physical reality

**Your login credentials on disk** (host):

```
~/.docker/config.json
{
  "auths": {
    "ghcr.io": { "auth": "base64(username:token)" },     ← how docker knows who you are
    "https://index.docker.io/v1/": { "auth": "..." }
  }
}
```

Note: `auth` is **base64, not encrypted** — it's an encoding, anyone reading the file sees your token. On macOS/Windows Docker Desktop stores it in the OS keychain instead (`"credsStore": "desktop"`).

**A local image's tags and digest** (host):

```
docker images --digests
REPOSITORY   TAG      DIGEST                          IMAGE ID
orders-api   1.4.0    sha256:9b2a4c...  ← the manifest digest (what the registry serves)
orders-api   latest   sha256:9b2a4c...  ← SAME digest: two tags, one image
```

**What actually travels to the registry** — the manifest:

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "digest": "sha256:1f3c...",   ← the image config blob (env, cmd, USER)
    "size": 7143
  },
  "layers": [
    { "digest": "sha256:aa11...", "size": 3123456 },   ← layer 1 (base OS)
    { "digest": "sha256:bb22...", "size": 145678  },   ← layer 2 (node_modules)
    { "digest": "sha256:cc33...", "size": 4096    }    ← layer 3 (your app code)
  ]
}
```

**On the registry server's disk** (e.g. a self-hosted registry):

```
/var/lib/registry/docker/registry/v2/
├── blobs/sha256/aa/aa11.../data      ← the raw layer tarballs, named by hash
├── blobs/sha256/1f/1f3c.../data      ← the config blob
└── repositories/orders-api/
    ├── _manifests/tags/1.4.0/current/link   → points to a manifest digest
    └── _manifests/tags/latest/current/link  → can point to the SAME or a DIFFERENT digest
```

That last block is the whole story of tags vs digests: a **tag is a symlink you can repoint**; a **digest is the file itself.**

---

## How it works — step by step

Trace a full `docker push orders-api:1.4.0` to GHCR:

1. **Resolve the name.** `ghcr.io/acme/orders-api:1.4.0` → registry host `ghcr.io`, repo `acme/orders-api`, tag `1.4.0`. A bare name like `orders-api:1.4.0` defaults the host to `docker.io`.

2. **Authenticate.** Docker reads `~/.docker/config.json`, gets the bearer token for `ghcr.io`, and hits `GET /v2/` to confirm auth. If missing/expired → `denied` / `unauthorized`.

3. **Check which layers already exist.** For each layer digest, Docker sends `HEAD /v2/acme/orders-api/blobs/sha256:aa11...`. If the registry already has that blob (from a previous push), it returns `200` and Docker **skips uploading it**. This is why pushing a one-line code change only uploads the tiny top layer — dedup by digest.

4. **Upload missing blobs.** For each absent layer: `POST` to start an upload session, `PATCH`/`PUT` the bytes, and the registry verifies the received bytes hash to the claimed digest. If they don't match, it rejects them — you cannot lie about a digest.

5. **Upload the config blob** the same way (it's just another content-addressed blob).

6. **Publish the manifest.** `PUT /v2/acme/orders-api/manifests/1.4.0` with the manifest JSON. The registry stores the manifest under its own digest AND creates/updates the `1.4.0` tag pointer to that digest. **This is the moment the tag becomes visible.**

7. **Pull is the mirror image.** `docker pull ghcr.io/acme/orders-api:1.4.0` → `GET .../manifests/1.4.0` returns the manifest → Docker `HEAD`s each layer against the local cache and `GET`s only the ones it's missing → reassembles the overlay filesystem (Topic 04).

8. **Digest pinning.** If you instead pull `orders-api@sha256:9b2a...`, step 6's tag lookup is skipped entirely — Docker asks for that exact manifest digest. There is no ambiguity and nothing anyone can repoint. That's immutability in action.

---

## Exact syntax breakdown

### docker login

```
docker login ghcr.io -u parshuram --password-stdin
       │      │        │  │         │
       │      │        │  │         └─ read the password/token from stdin (safer than -p:
       │      │        │  │            -p leaves the token in your shell history!)
       │      │        │  └─ your username
       │      │        └─ username flag
       │      └─ the registry host (omit → defaults to Docker Hub)
       └─ store a bearer token in ~/.docker/config.json for this host
```

### Tagging an image

```
docker tag orders-api:1.4.0 ghcr.io/acme/orders-api:1.4.0
       │    │                │       │    │          │
       │    │                │       │    │          └─ new tag
       │    │                │       │    └─ repository name
       │    │                │       └─ namespace / org / account
       │    │                │       (host) registry to push to
       │    │                └─ ─────────────
       │    └─ the SOURCE image already on your machine
       └─ create a NEW name pointing at the SAME image ID (no data copied — just a label)
```

### The full image reference anatomy

```
ghcr.io / acme / orders-api : 1.4.0
   │       │        │           │
   │       │        │           └─ TAG (mutable pointer) — or @sha256:... for a digest
   │       │        └─ repository
   │       └─ namespace (org or user)
   └─ registry host  (missing → docker.io ;  "library/" namespace if none → official images)
```

### Pushing and pulling

```
docker push ghcr.io/acme/orders-api:1.4.0      # upload manifest + missing layers
docker pull ghcr.io/acme/orders-api:1.4.0      # download by TAG (mutable)
docker pull ghcr.io/acme/orders-api@sha256:9b2a4c...   # download by DIGEST (immutable)
                                    │
                                    └─ "@" instead of ":" means pin the exact content
```

### Inspecting the digest without pulling

```
docker buildx imagetools inspect ghcr.io/acme/orders-api:1.4.0
       │       │         │
       │       │         └─ show the manifest, digest, platforms, and layer digests
       │       └─ the multi-platform image tool
       └─ resolves the tag → digest over the network (no full pull)
```

---

## Example 1 — basic

Push `orders-api` to GitHub Container Registry (GHCR), every step commented.

```bash
# 1. Log in. Use a GitHub Personal Access Token with write:packages scope,
#    piped via stdin so it never lands in shell history.
echo "$GHCR_TOKEN" | docker login ghcr.io -u parshuram --password-stdin
#   Login Succeeded

# 2. Build the image with a LOCAL name.
docker build -t orders-api:1.4.0 .

# 3. Tag it with the FULL registry reference (this copies no data — just adds a name).
docker tag orders-api:1.4.0 ghcr.io/acme/orders-api:1.4.0

# 4. Push. Watch it upload each layer by digest.
docker push ghcr.io/acme/orders-api:1.4.0
#   The push refers to repository [ghcr.io/acme/orders-api]
#   aa11...: Pushed          ← base layer
#   bb22...: Pushed          ← node_modules
#   cc33...: Pushed          ← app code
#   1.4.0: digest: sha256:9b2a4c... size: 1156     ← the manifest digest, REMEMBER THIS

# 5. From any other machine (or a CI runner), pull it back.
docker pull ghcr.io/acme/orders-api:1.4.0

# 6. Pull the EXACT same bytes by digest — immune to anyone repointing the tag.
docker pull ghcr.io/acme/orders-api@sha256:9b2a4c...
```

Push a second time after a one-line code change and watch the dedup:

```bash
docker build -t orders-api:1.4.1 . && docker tag orders-api:1.4.1 ghcr.io/acme/orders-api:1.4.1
docker push ghcr.io/acme/orders-api:1.4.1
#   aa11...: Layer already exists     ← base skipped (HEAD returned 200)
#   bb22...: Layer already exists     ← node_modules skipped
#   dd44...: Pushed                   ← ONLY the changed app-code layer uploaded
```

---

## Example 2 — production scenario

**The situation.** Your team ships `orders-api` several times a day. Early on, everyone deployed `orders-api:latest`. Last week, prod started behaving inconsistently: three Kubernetes nodes were serving three *different* builds, all tagged `latest`, because nodes pulled at different times and `latest` had moved. A customer-facing bug appeared and disappeared depending on which node handled the request. Nobody could reproduce it. Rollback was impossible because the previous `latest` had been overwritten.

**The fix: a real tagging strategy.** Every image gets three kinds of tags at build time in CI:

```bash
GIT_SHA=$(git rev-parse --short HEAD)      # e.g. a1b2c3d
VERSION=$(node -p "require('./package.json').version")   # e.g. 1.4.0
IMG=ghcr.io/acme/orders-api

docker build -t "$IMG:$GIT_SHA" .          # 1. IMMUTABLE per-commit tag — the source of truth
docker tag "$IMG:$GIT_SHA" "$IMG:$VERSION" # 2. HUMAN semver tag for releases
docker tag "$IMG:$GIT_SHA" "$IMG:latest"   # 3. convenience only — NEVER deploy this

docker push "$IMG:$GIT_SHA"
docker push "$IMG:$VERSION"
docker push "$IMG:latest"
```

**The critical rule for deploys — pin the digest, not the tag.** In the Kubernetes manifest (Topic 32), reference the image by digest so every node runs *provably identical bytes*:

```yaml
containers:
  - name: orders-api
    image: ghcr.io/acme/orders-api@sha256:9b2a4c...   # ← digest, not :latest
    imagePullPolicy: IfNotPresent
```

Now:
- **No drift.** Every node resolves the same digest to the same layers. Impossible to run "three different latests."
- **Instant rollback.** The old commit's tag (`orders-api:a1b2c3d`) and its digest still exist in the registry, untouched. Roll back by pointing at the previous digest — the image is right there.
- **Traceability.** `orders-api:a1b2c3d` tells you *exactly* which git commit is in prod. `kubectl describe pod` shows the digest; `docker buildx imagetools inspect` maps it back.

**Enforce immutability at the registry.** On GHCR/ECR, enable "immutable tags" so a tag, once pushed, cannot be overwritten — a push to an existing `1.4.0` is rejected. This makes the `latest`-drift class of bug structurally impossible for versioned tags.

```
# ECR example: create the repo with immutability on
aws ecr create-repository --repository-name orders-api --image-tag-mutability IMMUTABLE
                                                          │
                                                          └─ re-pushing an existing tag → error
```

---

## Common mistakes

### Mistake 1 — deploying `:latest` (the anti-pattern)

```yaml
image: ghcr.io/acme/orders-api:latest
```

**Symptom:** nodes run different builds; rollbacks impossible; "works on node A, broken on node B." **Root cause:** `latest` is just a mutable tag pointer — it has no special meaning to Docker, it's the default when you omit a tag. Two pushes move it. Combined with `imagePullPolicy: Always`, different nodes pull at different times and land on different digests. **Fix:** deploy an immutable per-commit or semver tag, and ideally pin the digest.

### Mistake 2 — forgetting to tag with the registry host before push

```
docker push orders-api:1.4.0
# denied: requested access to the resource is denied
```

**Root cause:** `orders-api:1.4.0` has no registry host, so Docker defaults to `docker.io/library/orders-api` — a namespace you don't own. **Fix:** `docker tag orders-api:1.4.0 ghcr.io/acme/orders-api:1.4.0` first, then push the fully-qualified name.

### Mistake 3 — not logged in / expired token

```
Error response from daemon: Head "https://ghcr.io/v2/acme/orders-api/blobs/...":
denied: denied
```

or in Kubernetes:

```
Failed to pull image: rpc error: code = Unknown desc = failed to authorize: ... 401 Unauthorized
  → ImagePullBackOff
```

**Root cause:** no bearer token for that host in `~/.docker/config.json`, or it expired, or (in K8s) no `imagePullSecret` on the pod. **Fix:** `docker login ghcr.io` locally; in Kubernetes create a `docker-registry` secret and reference it via `imagePullSecrets`.

### Mistake 4 — leaking a token via `-p` on the command line

```
docker login ghcr.io -u parshuram -p ghp_realTokenHere
# WARNING! Using --password via the CLI is insecure. Use --password-stdin.
```

**Root cause:** the token is now in your shell history (`~/.zsh_history`) and in the process list (`ps aux`) for anyone on the box to read. **Fix:** always `--password-stdin` and pipe from an env var or secret store.

### Mistake 5 — assuming the same tag is the same image

You pulled `orders-api:1.4.0` on Monday and again on Friday and assume they're identical. If tags are mutable and someone re-pushed, they're not. **Root cause:** a tag is a repointable pointer. **Fix:** pin by digest (`@sha256:...`) whenever "the same bytes" actually matters, and enable immutable tags on the registry.

---

## Hands-on proof

```bash
# 1. See tags and digests side by side — note two tags can share ONE digest.
docker build -t orders-api:1.4.0 . && docker tag orders-api:1.4.0 orders-api:latest
docker images --digests orders-api
#   1.4.0 and latest show the SAME IMAGE ID (and digest once pushed)

# 2. Run a LOCAL registry to push/pull without any cloud account.
docker run -d -p 5000:5000 --name registry registry:2
docker tag orders-api:1.4.0 localhost:5000/orders-api:1.4.0
docker push localhost:5000/orders-api:1.4.0
#   watch each layer upload by its sha256 digest

# 3. Look at the registry's raw storage — blobs named by hash.
docker exec registry ls /var/lib/registry/docker/registry/v2/blobs/sha256
#   directories named by the first 2 hex chars of each blob's digest

# 4. Fetch the manifest directly over HTTP (this is all `docker pull` does first).
curl -s localhost:5000/v2/orders-api/manifests/1.4.0 \
  -H "Accept: application/vnd.oci.image.manifest.v1+json" | head -30
#   see the config digest and the list of layer digests

# 5. Prove digest pinning ignores the tag. Grab the digest, then repoint the tag.
DIGEST=$(docker buildx imagetools inspect localhost:5000/orders-api:1.4.0 \
  --format '{{.Manifest.Digest}}')
echo "$DIGEST"
docker pull localhost:5000/orders-api@$DIGEST     # exact bytes, tag-independent

# 6. See where your login token lives (base64, NOT encrypted).
cat ~/.docker/config.json     # or: docker-credential-desktop list  (on Docker Desktop)
```

---

## Practice exercises

### Exercise 1 — easy
Spin up a local registry (`docker run -d -p 5000:5000 registry:2`). Build `orders-api`, tag it `localhost:5000/orders-api:1.0.0`, push it, then `docker rmi` the local copy and `docker pull` it back. Confirm it runs.

### Exercise 2 — medium
Push `orders-api` with three tags from one build: a git SHA (`git rev-parse --short HEAD`), a semver (`1.0.0`), and `latest`. Verify with `docker images --digests` that all three share one digest. Then rebuild after a code change, re-tag only the SHA and `latest` (not the semver), push, and observe that `latest` now points to a *different* digest than `1.0.0`. Explain what that means for anyone deploying `latest`.

### Exercise 3 — hard (production simulation)
Simulate the "latest drift" incident and its fix. (a) Push `orders-api:latest` (v1). (b) Pull it into a "node A" tag. (c) Push a changed `orders-api:latest` (v2). (d) Pull it into "node B". (e) Prove nodes A and B now differ (compare image IDs). (f) Now redo the whole thing but reference images by digest instead — capture each digest and show that pinning makes A and B provably identical no matter when they pulled. Write up why digest pinning + immutable tags eliminates this class of bug.

---

## Mental model checkpoint

1. What is the difference between a tag and a digest, at the storage level?
2. What is a manifest, and what does it contain vs. what it does NOT contain?
3. During `docker push`, how does the registry avoid re-uploading layers it already has?
4. Why is `sha256:...` immutable but `1.4.0` is not?
5. What are three problems with deploying `:latest`?
6. Where does `docker login` store your credentials, and how safe is that file?
7. Given a running prod pod, how do you find out exactly which git commit it's serving?

---

## Quick reference card

| Command / concept | What it does | Key detail |
|---|---|---|
| `docker login HOST` | Store a bearer token for a registry | Use `--password-stdin`, never `-p` |
| `docker tag SRC HOST/ns/repo:tag` | Add a fully-qualified name | Copies no data; must include host to push |
| `docker push REF` | Upload manifest + missing layers | Skips layers the registry already has |
| `docker pull REF:tag` | Download by mutable tag | Tag can be repointed later |
| `docker pull REF@sha256:...` | Download exact bytes by digest | Immutable; immune to tag changes |
| `docker images --digests` | Show local tags + digests | Two tags can share one digest |
| `imagetools inspect REF` | Resolve tag→digest without full pull | Great for pinning in manifests |
| Docker Hub | Default public registry (`docker.io`) | Public repos are world-readable |
| GHCR / ECR | Private registries (GitHub / AWS) | Need auth + `imagePullSecrets` in K8s |
| Immutable tags | Registry rejects re-pushing a tag | Kills the `latest`-drift bug class |

---

## When would I use this at work?

1. **Every deploy.** CI builds `orders-api`, tags it with the git SHA and semver, pushes to GHCR/ECR, and the deploy references the digest. This is the daily backbone of shipping.

2. **Incident rollback.** Prod is broken after a deploy. Because every commit is an immutable tag/digest in the registry, you roll back to the previous digest in seconds — the old image was never overwritten.

3. **Auditing "what's running."** Security or your manager asks "exactly what code is in prod?" You read the pod's image digest, map it to the git-SHA tag, and point at the precise commit — no guessing.

---

## Connected topics

- **Study before:** Topic 04 (Images and Layers — content-addressable storage, what a layer is), Topic 08 (Layer Caching — why layer dedup on push works), Topic 23 (Image Security — scan before you push, keep secrets out).
- **Study after:** Topic 25 (Resource Limits — the runtime side of production), Topic 27 (Docker in CI/CD — automating tag + push on merge), Topic 34 (Deployments — where the image reference actually gets consumed), Topic 36 (ConfigMaps and Secrets — `imagePullSecrets` for private registries).
