# 04 — Images and Layers
## Section: Docker Foundations

---

## ELI5 — The Simple Analogy

Imagine you draw a cartoon on **glass sheets** stacked on top of each other.

- Bottom sheet: you draw a plain background (the ground, the sky).
- Next sheet: you draw a house on top.
- Next sheet: you draw a person standing in the door.

You never redraw the background. You just add a new **transparent sheet** and draw only the *new stuff* on it. When you look down through the whole stack, you see one complete picture. But physically, it is many thin sheets, each holding only its own change.

A Docker **image** is exactly this: a stack of read-only sheets. Each sheet is called a **layer**. Each layer only stores the *difference* from the layer below it — the files that were added, changed, or deleted. When Docker "flattens" the stack for you, you see one normal-looking filesystem: `/app`, `/usr`, `/bin`, everything. But on disk it is stored as separate thin diffs.

And here is the clever money-saving trick: if two different cartoons both use the *same background sheet*, you only store that sheet **once** and let both point to it. That is why pulling your 3rd Node.js image is fast — you already have the `node:20-alpine` sheets.

---

## The Linux kernel feature underneath

The kernel feature that makes layers possible is the **overlay filesystem** — specifically the `overlay` filesystem driver, which Docker uses through its `overlay2` storage driver.

`overlayfs` is a **union mount** filesystem built into the Linux kernel (`fs/overlayfs/` in the kernel source, config option `CONFIG_OVERLAY_FS`). A union mount takes several directories and presents them as **one merged directory**. You give it:

```
lowerdir  → one or more read-only directories (the image layers, stacked)
upperdir  → one writable directory (where new writes go)
workdir   → a scratch/staging directory overlayfs needs internally
merged    → the single combined view your process actually sees
```

The kernel merges them with two rules:

1. **Top wins.** If the same file path exists in `upperdir` and in a `lowerdir`, you see the `upperdir` version. Higher layers shadow lower layers.
2. **Copy-up on write.** The lower directories are read-only. The moment a process *writes* to a file that lives in a lower layer, the kernel **copies that whole file up** into `upperdir` first, then modifies the copy. The original in the lower layer is untouched. This is called **copy-up**.

There is also a special trick for **deletes**. You cannot actually delete a file from a read-only lower layer. So overlayfs writes a **whiteout** — a special character device file with major:minor `0:0` in the upper layer. When the kernel sees a whiteout for a path, it hides that path from the merged view, even though it still physically exists in the lower layer.

So an "image" is nothing magical. It is a set of directories fed to the kernel as `lowerdir`, and a running container adds one `upperdir` on top. Everything else is bookkeeping around that.

> Remember from Topic 02 — namespaces isolate *what a process can see* (PIDs, network, mounts). Overlayfs is the third pillar: it isolates *what filesystem a process sees*, cheaply, by sharing read-only layers between containers.

---

## What is this?

A Docker **image** is a read-only, layered, content-addressed bundle of a filesystem plus metadata that says how to run it. Each **layer** is a tarball of filesystem changes (a diff) identified by the SHA-256 hash of its contents. An image is defined by a **manifest** — a small JSON file that lists which layers, in which order, plus a **config** blob describing the default command, env vars, and working directory.

You never run an image directly. Docker stacks its layers with overlayfs, adds a writable top layer, and *that* running combination is a **container** (Topic 05).

---

## Why does it matter for a backend developer?

Because almost every real Docker problem you will hit is a layer problem in disguise:

- **"Why is my `orders-api` image 1.2 GB?"** — You copied `node_modules` and build tools into a layer that never gets cleaned up. Deleting files in a *later* layer does not shrink the image; the bytes are still in the lower layer (see Common Mistakes).
- **"Why does every build re-download all npm packages?"** — You put `COPY . .` before `npm install`, so any code change busts the cache for the install layer (deep-dived in Topic 08).
- **"Why did the deploy pull 900 MB when I only changed one line?"** — Your changed line landed in a low layer, invalidating every layer above it.
- **"Why is disk full on the CI box?"** — Old dangling layers under `/var/lib/docker/overlay2` never got pruned.
- **"Two teammates get different behavior from the `same` tag."** — Tags are mutable pointers; digests are not. If you do not pin by digest, `node:20-alpine` today ≠ `node:20-alpine` last month.

Understanding layers turns all of these from mysteries into obvious cause-and-effect.

---

## The physical reality

Everything Docker stores lives under `/var/lib/docker/` on the host (on Docker Desktop for Mac/Windows, this is inside a small Linux VM, not your Mac filesystem — more on that in Hands-on proof). The parts that matter for images:

```
/var/lib/docker/
├── image/
│   └── overlay2/
│       ├── imagedb/
│       │   └── content/sha256/<image-config-digest>   ← image CONFIG json (per image)
│       ├── layerdb/
│       │   └── sha256/<chainID>/                       ← per-layer metadata
│       │       ├── diff          ← the diffID (uncompressed layer digest)
│       │       ├── cache-id      ← points to the overlay2 dir holding real files
│       │       ├── parent        ← chainID of the layer below
│       │       └── size          ← layer size in bytes
│       └── repositories.json     ← tag → image-digest mapping (e.g. node:20-alpine → sha256:…)
│
└── overlay2/
    ├── <cache-id-1>/
    │   ├── diff/                 ← the ACTUAL files of this layer live here
    │   └── link                  ← short symlink name (kept in ../l/)
    ├── <cache-id-2>/
    │   ├── diff/
    │   ├── lower                 ← text file: chain of lower layers, e.g. "l/ABC:l/DEF"
    │   ├── work/                 ← overlayfs workdir
    │   └── link
    └── l/                        ← short symlink names → real cache-id dirs (keeps mount cmd short)
```

Two separate concepts of "digest" exist, and mixing them up causes real confusion:

```
digest type   where it comes from                what it identifies
───────────   ────────────────────────────────   ─────────────────────────────
diffID        sha256 of the UNCOMPRESSED tar      one layer's raw content, locally
              (layerdb/.../diff)

chainID       sha256(parentChainID + " " + diffID)  a layer AND its full history
              (the layerdb directory name)         (so identical files stacked on
                                                    different parents differ)

distribution  sha256 of the COMPRESSED (gzip) tar  a layer as it travels over the
digest        blob as stored in a registry         network / stored in registry
```

The **content-addressable** part: the name of the thing *is* the hash of the thing. If you know a layer's digest, you can verify you got the exact bytes by re-hashing them. Change one byte anywhere in the layer and the digest changes completely. This is how Docker guarantees integrity and deduplicates automatically — two images referencing digest `sha256:abc…` are guaranteed to be byte-for-byte the same layer, so it is stored once.

---

## How it works — step by step

Let's trace what actually happens on disk for a tiny `orders-api` image built from:

```dockerfile
FROM node:20-alpine        # layers A,B,... (base OS + node)
WORKDIR /app               # metadata-only, no new files layer
COPY package.json .        # layer C
RUN npm ci                 # layer D (node_modules)
COPY . .                   # layer E (your source)
CMD ["node","server.js"]   # metadata-only
```

**Build side — how layers get created:**

1. Docker reads `FROM node:20-alpine`. It resolves the tag to an image digest via `repositories.json`, then reads that image's manifest to get the ordered list of layer digests. Those base layers already exist under `overlay2/` (pulled once, shared forever).
2. `WORKDIR /app` changes *config metadata* only (a field in the config JSON). No filesystem diff, so **no new layer** — it is a "zero-byte" layer conceptually.
3. `COPY package.json .` — Docker mounts the base layers as `lowerdir`, mounts a fresh `upperdir`, copies `package.json` into `/app`. It then **snapshots** the `upperdir` as a new read-only layer C, computes its diffID (sha256 of its tar), computes its chainID, and stores it under `layerdb/` + `overlay2/<cache-id>/diff/`.
4. `RUN npm ci` — Docker starts a throwaway container with layers A,B,C as lower and a new upper. `npm` writes `/app/node_modules/**`. On exit, Docker snapshots the upper (all of `node_modules`) as layer D.
5. `COPY . .` — snapshots your source as layer E.
6. The `CMD` is written into the final **config JSON**. Docker writes the **image manifest** listing layers [A,B,C,D,E] and pointing at the config. The tag `orders-api:latest` is written into `repositories.json`.

**Run side — how the stack becomes a live filesystem (preview of Topic 05):**

7. `docker run orders-api` — Docker (via containerd/snapshotter) creates a **new writable upperdir** and asks the kernel to mount overlayfs:

```
mount -t overlay overlay \
  -o lowerdir=l/E:l/D:l/C:l/B:l/A,upperdir=<new>/diff,workdir=<new>/work \
  /var/lib/docker/overlay2/<container-id>/merged
```

8. The container's PID 1 sees `merged/` as its root filesystem `/`. Reading `/app/server.js` resolves down the stack to layer E. Reading `/usr/bin/node` resolves down to a base layer. Writing a new file `/app/tmp.log` goes into the **upperdir only** — the image layers stay pristine.
9. When the container is deleted, only the upperdir is thrown away. The image layers are untouched and reused by the next container.

**Pull side — how `docker pull node:20-alpine` works layer by layer:**

1. `docker pull node:20-alpine` → the CLI asks the daemon; the daemon contacts the registry (default `registry-1.docker.io`) and first does an **auth handshake** (gets a bearer token for `repository:library/node:pull`).
2. Daemon requests the **manifest**: `GET /v2/library/node/manifests/20-alpine` with header `Accept: application/vnd.oci.image.index.v1+json` (and the older Docker manifest-list type). The registry may return a **manifest list / image index** — a list of manifests, one per platform (`linux/amd64`, `linux/arm64`, …).
3. Daemon picks the manifest matching your CPU/OS and fetches *that* manifest by digest. The manifest lists the **config blob digest** and the ordered **layer blob digests** (compressed, gzip).
4. Daemon downloads the **config** blob, then downloads each **layer blob** — **in parallel**, up to 3 concurrent by default. You see the familiar:

```
20-alpine: Pulling from library/node
a1b2c3d4: Pulling fs layer
a1b2c3d4: Downloading  [====>       ]  2.1MB/8.4MB
e5f6g7h8: Already exists          ← this layer's digest is already on disk → skipped!
a1b2c3d4: Extracting  [======>     ]
a1b2c3d4: Pull complete
```

5. For each layer: if its digest already exists locally, Docker prints **`Already exists`** and downloads nothing (content-addressable dedup in action). Otherwise it downloads the gzip blob, **verifies** the sha256 matches the digest (integrity check), **decompresses** it, and unpacks the tar into a new `overlay2/<cache-id>/diff/`.
6. Once all layers are present and verified, the image is assembled and the tag recorded in `repositories.json`. Now `docker images` shows it.

The key insight: **pull is per-layer and deduplicated**. Change one line of app code and rebuild — only the top layer's digest changes, so a `docker push`/`pull` moves only that one small layer. Everyone already has the base.

---

## Exact syntax breakdown

**A layer digest as printed by `docker pull`:**

```
sha256:e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6
│      │
│      └─ 64 hex chars = 256 bits = the SHA-256 hash of the blob's bytes
└─ algorithm prefix. Always "sha256:" for current Docker/OCI images.
   The name IS the hash → content-addressable. Re-hash the bytes to verify.
```

**`docker image inspect` layer list (the `RootFS.Layers` field — these are diffIDs):**

```
"RootFS": {
  "Type": "layers",
  "Layers": [
    "sha256:aaa…",   ← layer A (base, bottom of stack)
    "sha256:bbb…",   ← layer B
    "sha256:ccc…"    ← layer C (top, applied last, wins conflicts)
  ]
}
│                    │
│                    └─ ORDER MATTERS. Bottom applied first, top applied last.
└─ These are diffIDs (uncompressed digests), not the compressed registry digests.
```

**The overlay mount options (what the kernel is actually told):**

```
mount -t overlay overlay -o lowerdir=L3:L2:L1,upperdir=U,workdir=W /merged
       │          │       │  │                │        │        │
       │          │       │  │                │        │        └ where the merged view appears
       │          │       │  │                │        └ overlayfs scratch dir (same fs as upper)
       │          │       │  │                └ the single writable layer (container writes land here)
       │          │       │  └ read-only layers, LEFT = TOP of stack, RIGHT = BOTTOM
       │          │       └ mount options
       │          └ the "device" name (arbitrary string "overlay")
       └ filesystem type = overlay (the kernel union mount)
```

Note the ordering gotcha: in the *mount command* `lowerdir` is written **top-first** (`L3:L2:L1`), but in `docker inspect` the `Layers` array is **bottom-first**. Same stack, opposite reading direction.

**Referencing an image by digest vs by tag (in a Dockerfile or `docker run`):**

```
node:20-alpine
│    │
│    └─ TAG. A MUTABLE, human-friendly pointer. Can change under you over time.
└─ repository name

node@sha256:abc123…
│   │
│   └─ DIGEST. IMMUTABLE. Always the exact same bytes. Use in production for reproducibility.
└─ repository name
```

---

## Example 1 — basic

See the layers of a real Node.js base image and prove layers are shared.

```bash
# 1) Pull the base once. Watch each layer download separately.
docker pull node:20-alpine

# 2) See the layer stack (these are diffIDs, bottom → top):
docker image inspect node:20-alpine --format '{{json .RootFS.Layers}}' | tr ',' '\n'
# "sha256:aaa…"     ← alpine base rootfs
# "sha256:bbb…"     ← node binaries
# ...

# 3) See how each Dockerfile step maps to a layer + its size:
docker history node:20-alpine
# IMAGE          CREATED       CREATED BY                             SIZE
# a1b2c3        2 weeks ago   CMD ["node"]                           0B      ← metadata-only, 0B
# <missing>     2 weeks ago   ENV NODE_VERSION=20.11.0               0B      ← metadata-only
# <missing>     2 weeks ago   RUN addgroup ... && apk add ...        45MB    ← real files → real layer
# <missing>     2 weeks ago   /bin/sh -c #(nop) ADD file:… in /      7.4MB   ← alpine rootfs
#   │
#   └─ "<missing>" just means "this layer has no standalone image ID of its own",
#      NOT that anything is broken. Layers below the top rarely have their own tag.
```

Now build a tiny image on top and watch the base layers get reused:

```bash
mkdir orders-api && cd orders-api
cat > server.js <<'EOF'
require('http').createServer((_, res) => res.end('orders ok\n')).listen(3000);
EOF
cat > Dockerfile <<'EOF'
FROM node:20-alpine
WORKDIR /app
COPY server.js .
CMD ["node","server.js"]
EOF

docker build -t orders-api:latest .
# Notice: it does NOT re-download node:20-alpine. Those layers already exist on disk.

docker history orders-api:latest
# Only your COPY layer is new & small (a few hundred bytes). Everything below is shared.
```

---

## Example 2 — production scenario

**"Our `orders-api` image is 1.4 GB and CI is slow. We only ship a 3 MB app."**

Your teammate wrote this Dockerfile:

```dockerfile
FROM node:20                      # ← full Debian-based node, ~1.1 GB!
WORKDIR /app
COPY . .                          # ← copies node_modules, .git, tests, everything
RUN npm install                   # ← installs prod + dev deps (webpack, jest, typescript…)
RUN npm run build                 # ← creates dist/
RUN rm -rf node_modules && npm ci --omit=dev   # ← "cleanup" that does NOT shrink the image
CMD ["node","dist/server.js"]
```

Two separate bugs, both layer bugs:

**Bug 1 — the "cleanup" line is a lie about size.** `RUN rm -rf node_modules` runs in a *new layer* on top. Overlayfs records a **whiteout** to hide `node_modules`, but the original bloated `node_modules` still physically lives in the layer below. The image ships *both* the fat directory and the whiteout. You can prove it:

```bash
docker history orders-api:fat
# ...
# <missing>  RUN npm install              620MB   ← the fat deps are STILL here, forever
# <missing>  RUN rm -rf node_modules …      0B    ← whiteout adds nothing back but removes nothing
```

**Bug 2 — `FROM node:20` (Debian) vs `node:20-alpine`.** The base alone is ~1.1 GB vs ~130 MB.

The fix uses **multi-stage builds** (full treatment in Topic 09) so the fat build layers never enter the final image at all:

```dockerfile
# ---- build stage: throwaway, its layers are NOT in the final image ----
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./            # copy manifests first → better cache (Topic 08)
RUN npm ci                       # all deps, incl dev
COPY . .
RUN npm run build

# ---- runtime stage: only THIS stage's layers ship ----
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev            # prod deps only, in a FRESH layer with no dev cruft below it
COPY --from=build /app/dist ./dist   # copy ONLY the built output across stages
USER node
CMD ["node","dist/server.js"]
```

Result: the final image contains only alpine + node + prod `node_modules` + `dist`. From 1.4 GB → ~150 MB. And because `package*.json` is copied before the source, editing a route file busts only the tiny `COPY . .` layer, not the multi-hundred-MB `npm ci` layer — CI pull/push drops from minutes to seconds.

---

## Common mistakes

**Mistake 1 — Deleting files in a later layer to "save space."**

```dockerfile
RUN wget https://example.com/big.tar.gz && tar xzf big.tar.gz && rm big.tar.gz
# vs
RUN wget https://example.com/big.tar.gz \
 && tar xzf big.tar.gz
RUN rm big.tar.gz     # ← WRONG: separate RUN = separate layer, big.tar.gz still in lower layer
```

*Symptom:* image far bigger than expected; `docker history` shows a big layer that a later `0B` whiteout "removed."
*Root cause:* overlayfs whiteouts hide but never reclaim bytes from lower read-only layers.
*Right:* download, extract, and delete in **one** `RUN` so the temp file never becomes part of any committed layer.

**Mistake 2 — Assuming a tag is immutable.**

```bash
docker pull node:20-alpine    # today: sha256:aaa
# ...six weeks later, same command, DIFFERENT bytes:
docker pull node:20-alpine    # now:   sha256:zzz (upstream rebuilt it)
```

*Symptom:* "works on my machine," reproducibility failures, a CVE reappears or a patch silently changes behavior.
*Root cause:* tags are mutable pointers in the registry; only digests are immutable.
*Right:* pin production bases by digest — `FROM node:20-alpine@sha256:…` — and let Dependabot/Renovate bump the digest via PR.

**Mistake 3 — Disk fills up: `no space left on device`.**

```
write /var/lib/docker/overlay2/abc/diff/…: no space left on device
```

*Symptom:* builds and pulls fail; `df -h` shows `/var/lib/docker` full.
*Root cause:* every rebuild can leave **dangling** layers (old cache-ids no longer referenced by any tag). They accumulate under `overlay2/`.
*Right:* `docker system df` to see usage, then `docker image prune` (dangling only) or `docker system prune -a` (careful — removes all unused images).

**Mistake 4 — Confusing the compressed (registry) digest with the local diffID.**

*Symptom:* the digest in `docker pull` output does not match the digest in `docker image inspect .RootFS.Layers`, and you think Docker is lying.
*Root cause:* the registry stores the **gzip-compressed** blob (distribution digest); locally Docker tracks the **uncompressed** tar (diffID). Same layer, two hashes.
*Right:* use `docker manifest inspect node:20-alpine` for registry (compressed) digests; use `docker image inspect` for local diffIDs. They are *supposed* to differ.

**Mistake 5 — Expecting container writes to persist in the image.**

```bash
docker run -d --name o orders-api
docker exec o sh -c 'echo hi > /app/data.txt'   # writes to the WRITABLE upper layer
docker rm -f o                                   # upper layer is destroyed with the container
# data.txt is gone. The image never changed.
```

*Root cause:* the image layers are read-only; writes live only in the per-container upperdir.
*Right:* persist real data with a **volume** (Topic 14), or bake fixed files in at build time with `COPY`.

---

## Hands-on proof

Run these right now to *see* layers physically on disk.

> On **Docker Desktop (Mac/Windows)** the Docker engine runs inside a Linux VM, so `/var/lib/docker` is not on your Mac. Drop into the VM first with:
> `docker run -it --rm --privileged --pid=host justincormack/nsenter1`
> (gives you a shell in the VM). On a **Linux host**, just `sudo` the commands directly.

```bash
# 1) Build the tiny orders-api from Example 1, then find its overlay2 dirs:
docker build -t orders-api:latest .

# 2) List the layer metadata database (chainIDs = directory names):
ls /var/lib/docker/image/overlay2/layerdb/sha256/
# each dir = one layer. cat its files:
CID=$(ls /var/lib/docker/image/overlay2/layerdb/sha256/ | head -1)
cat /var/lib/docker/image/overlay2/layerdb/sha256/$CID/diff      # the diffID
cat /var/lib/docker/image/overlay2/layerdb/sha256/$CID/size      # bytes
cat /var/lib/docker/image/overlay2/layerdb/sha256/$CID/cache-id  # → points into overlay2/

# 3) Follow the cache-id to the REAL files of that layer:
CACHE=$(cat /var/lib/docker/image/overlay2/layerdb/sha256/$CID/cache-id)
ls /var/lib/docker/overlay2/$CACHE/diff/    # ← the actual filesystem diff of this layer

# 4) See the image config (default CMD, env, working dir) — content-addressed by digest:
docker image inspect orders-api:latest --format '{{.Id}}'   # sha256:… = config digest
cat /var/lib/docker/image/overlay2/imagedb/content/sha256/<that-hash-without-prefix> | head

# 5) Run a container and watch a NEW writable upperdir appear:
docker run -d --name proof orders-api:latest
docker inspect proof --format '{{.GraphDriver.Data.UpperDir}}'   # the container's writable layer
docker inspect proof --format '{{.GraphDriver.Data.LowerDir}}'   # the shared read-only image layers
docker exec proof sh -c 'echo change > /app/new.txt'
ls $(docker inspect proof --format '{{.GraphDriver.Data.UpperDir}}')/app/   # new.txt is HERE only
docker rm -f proof   # upperdir destroyed; image layers untouched

# 6) Prove dedup: pull another node:20-alpine-based image; watch "Already exists":
docker pull node:20-alpine   # every layer → "Already exists" (nothing downloaded)

# 7) Inspect a real registry manifest (compressed digests, platform list):
docker manifest inspect node:20-alpine
```

What you have verified: an image is a stack of directories on disk, each named by the hash of its contents; a running container is those directories plus one writable directory; and identical layers are stored exactly once.

---

## Practice exercises

### Exercise 1 — easy
Pull `redis:7-alpine` and `postgres:16-alpine`. Run `docker history` on each and identify (a) which steps are metadata-only `0B` layers, and (b) which single step contributes the most bytes. Then pull `node:20-alpine` and run `docker system df` to see total image disk usage.

### Exercise 2 — medium
Build the `orders-api` image from Example 1. Change one character in `server.js` and rebuild. Run `docker history` before and after and confirm only the top `COPY server.js` layer's ID changed while all base layers kept the same IDs. Then add a second file and rebuild again — prove the base layers still never rebuild.

### Exercise 3 — hard (production simulation)
You inherit this Dockerfile and the image is 1.3 GB:
```dockerfile
FROM node:20
COPY . .
RUN npm install && npm run build
RUN rm -rf node_modules && npm ci --omit=dev
CMD ["node","dist/server.js"]
```
Rewrite it as a multi-stage build on `node:20-alpine` with a non-root user, prod-only deps, and correct COPY ordering for cache. Measure the before/after size with `docker images`. Then use `docker history --no-trunc` on both to explain *exactly* which layers disappeared and why the old "cleanup" line never reduced size. Finally, pin the base image by digest and explain what reproducibility guarantee that adds.

---

## Mental model checkpoint

Answer from memory:

1. What kernel filesystem feature underlies Docker layers, and what are its four directory roles (`lowerdir`, `upperdir`, `workdir`, `merged`)?
2. When a container writes to a file that exists only in a read-only image layer, what does the kernel do? What is that mechanism called?
3. How does overlayfs represent a *deleted* file that lives in a lower layer?
4. What is the difference between a **diffID**, a **chainID**, and a **distribution (compressed) digest**?
5. Why does deleting a large file in a *later* `RUN` instruction fail to shrink the image?
6. During `docker pull`, why do some layers say `Already exists` and download nothing?
7. Why is pinning `node:20-alpine@sha256:…` more reproducible than `node:20-alpine`?

---

## Quick reference card

| Command / concept | What it does | Key detail |
|---|---|---|
| `docker history <img>` | Lists layers + sizes + the instruction that made each | `0B` = metadata-only; `<missing>` = layer has no standalone ID |
| `docker image inspect <img>` | Full JSON: config, `RootFS.Layers` (diffIDs) | `Layers` array is bottom→top |
| `docker manifest inspect <img>` | Registry manifest: compressed layer digests + platforms | Shows manifest list (multi-arch) |
| `docker system df` | Disk used by images/containers/volumes | Add `-v` for per-image breakdown |
| `docker image prune` | Delete dangling (untagged) layers | `-a` deletes all unused images |
| overlayfs `lowerdir` | Read-only stacked image layers | Written top-first in mount cmd |
| overlayfs `upperdir` | The container's single writable layer | Destroyed with the container |
| copy-up | Copy a file up before first write | Why first write to a big file is slow |
| whiteout | Hides a lower-layer file marked as deleted | Bytes are NOT reclaimed |
| diffID | sha256 of uncompressed layer tar | Local identity |
| distribution digest | sha256 of gzip layer blob | Registry/network identity |
| content-addressable | Name = hash of contents | Enables dedup + integrity |
| `img@sha256:…` | Pin exact image bytes | Immutable, use in prod |

---

## When would I use this at work?

1. **Slashing image size and CI time.** When your `orders-api` image is huge, you read `docker history`, spot the fat and mis-ordered layers, and reorder/multi-stage them — turning multi-minute pulls into seconds and shrinking the registry bill.
2. **Debugging "works on my machine."** A pod behaves differently in prod than locally. You compare `docker manifest inspect` digests and discover the `:latest`/`:20-alpine` tag moved. You pin by digest and the divergence disappears.
3. **Fixing a full CI/worker disk.** `no space left on device` during a deploy. You use `docker system df` to find gigabytes of dangling overlay2 layers from months of builds and add a `docker image prune` step to the pipeline.

---

## Connected topics

- **Study before:** Topic 02 — Linux Foundations (namespaces, cgroups, overlayfs). Topic 03 — Docker Architecture (CLI → daemon → containerd → runc; who actually pulls and mounts).
- **Study after:** Topic 05 — Your First Container (how these layers become a live, running root filesystem). Topic 08 — Layer Caching in Depth (what busts the cache and how to order instructions). Topic 09 — Multi-Stage Builds (keeping fat layers out of the final image). Topic 24 — Docker Registry (pushing/pulling and tagging strategy).
