# 23 — Image Security

## Section: Production Patterns

---

## ELI5 — The Simple Analogy

Imagine you order a shipping container full of furniture for your new house. Before you let it into your home, three questions matter:

1. **What's inside?** Are there rats, mold, or broken glass hidden in the packing material? (vulnerability scanning)
2. **Who's allowed to open it?** Does the delivery guy get to walk around your whole house with a master key, or just drop the box in the garage? (non-root user + capabilities)
3. **Can he rearrange your furniture while he's here?** Or is everything bolted to the floor so he can only look, not touch? (read-only filesystem)

A Docker image is that shipping container. By default Docker opens it with a **master key** (root), lets the contents **rewrite anything** (writable filesystem), and never checks **what's packed inside** (no scan). Image security is the practice of turning all of that off: pack less, open with a limited key, and bolt the furniture down.

A "smaller box with fewer things inside" is also safer — there is simply less that can go wrong. That is why we reach for `alpine` and `distroless` base images.

---

## The Linux kernel feature underneath

Everything in this topic maps to real kernel primitives. This is not Docker magic — it is the Linux kernel enforcing rules on a process.

**1. User namespaces + UIDs.** Remember from Topic 02 that a container is just a normal Linux process wrapped in namespaces and cgroups. When a process runs, the kernel tags it with a **UID** (user ID) stored in the process's credentials (`struct cred` inside the kernel). UID `0` is root. The kernel grants root special treatment in hundreds of permission checks. When you set `USER node` in a Dockerfile, the container's main process gets a non-zero UID, so the kernel refuses root-only operations for it.

You can see the UID a process runs as:

```
/proc/[PID]/status   →  contains a line "Uid: 1000 1000 1000 1000"
```

**2. Linux capabilities.** Historically "root can do anything" was one giant on/off switch. Since Linux 2.2 the kernel split root's power into ~40 discrete **capabilities**, each a bit in a bitmask attached to the process:

```
CAP_NET_BIND_SERVICE  → bind to ports below 1024
CAP_CHOWN             → change file ownership
CAP_SETUID            → change process UID
CAP_SYS_ADMIN         → the "god mode" catch-all
CAP_NET_RAW           → craft raw packets (ping, some exploits)
```

The kernel stores these bitmasks per process and checks them in functions like `capable(CAP_NET_BIND_SERVICE)`. Docker runs containers with a **restricted default set** and lets you drop more with `--cap-drop`.

**3. Read-only bind mounts / mount flags.** When Docker mounts your container's root filesystem, it can pass the `MS_RDONLY` flag to the `mount()` syscall. The kernel's VFS (virtual filesystem) layer then rejects any `write()` to that mount with `EROFS` (read-only filesystem). This is the same flag behind `mount -o ro`.

**4. The `no_new_privs` bit.** A single kernel flag (`PR_SET_NO_NEW_PRIVS`) that, once set on a process, prevents it from ever gaining more privileges via setuid binaries. Docker sets it when you drop capabilities and add `--security-opt=no-new-privileges`.

So "image security" is really: *choose the UID, choose the capability bitmask, choose the mount flags, and put fewer files on disk for an attacker to abuse.*

---

## What is this?

Image security is the set of build-time and run-time choices that shrink the "attack surface" of a container: scanning the image for known-vulnerable packages, running the process as a non-root UID, mounting the root filesystem read-only, dropping Linux capabilities the app never needs, using a tiny base image with almost nothing installed, and keeping secrets out of image layers.

None of these are exotic. Each is one flag or one Dockerfile line. Together they turn a container that can be trivially exploited into one where even a code-execution bug gives the attacker almost nothing to work with.

---

## Why does it matter for a backend developer?

Because your `orders-api` container will eventually have a bug — a dependency with a CVE, an SSRF, a path-traversal. The question is not *if* an attacker gets code execution inside the container, but *what they can do once they are there.*

Without image security:
- The process runs as **root**, so a bug lets the attacker read/modify every file, install tools, and probe the host.
- The filesystem is **writable**, so they drop a crypto-miner or a reverse shell into `/tmp` or overwrite your app code.
- The image ships with `curl`, `wget`, `bash`, a package manager, and 200 OS libraries — a full toolkit handed to the attacker.
- Nobody scanned it, so a critical OpenSSL CVE has been sitting in production for six months.
- Someone `COPY`-ed the `.env` file, so `DATABASE_URL` with the Postgres password is baked into a layer that anyone who pulls the image can extract.

With image security, that same code-execution bug lands the attacker in a read-only, near-empty filesystem, as an unprivileged user who cannot bind low ports, cannot chown files, cannot install anything, and cannot even find a shell to run. That is the difference between an incident and a non-event.

---

## The physical reality

When your `orders-api` container is running, here is what actually exists and where.

**The image layers on disk** (host):

```
/var/lib/docker/overlay2/<layer-id>/diff/     ← each layer's files (Topic 04)
/var/lib/docker/image/overlay2/imagedb/content/sha256/<image-id>
        └── the image config JSON, which contains:
              "config": {
                 "User": "1000",              ← the USER set in the Dockerfile
                 "Env": [...],                ← baked-in env vars (secrets leak here!)
                 "Cmd": [...]
              }
```

**The running process's identity** (inside the container's PID, viewed from host):

```
/proc/<PID>/status
    Uid:  1000  1000  1000  1000            ← real, effective, saved, filesystem UID
    CapEff: 0000000000000000                 ← effective capability bitmask (0 = none!)
    CapBnd: 00000000a80425fb                 ← bounding set (max caps possible)
    NoNewPrivs: 1                            ← the PR_SET_NO_NEW_PRIVS bit
```

`CapEff: 0000000000000000` is the goal — a process that dropped all capabilities. Decode any of these with:

```
capsh --decode=00000000a80425fb
```

**The read-only root mount** (from inside the container):

```
/proc/mounts
    overlay / overlay ro,relatime,lowerdir=...   ← note the "ro"
```

**A leaked secret in a layer** (how an attacker extracts it):

```
docker save orders-api:1.4.0 -o img.tar
tar xf img.tar
# each layer.tar contains the raw files — grep for the secret:
grep -r "postgres://" .
```

That last block is the whole reason "no secrets in layers" matters: **layers are just tarballs**. Anyone with the image can unpack them.

---

## How it works — step by step

Let's trace what happens end-to-end for a hardened `orders-api`, from build to a running, locked-down container.

1. **Build with a minimal base.** `FROM node:20-alpine` pulls a ~50 MB base instead of `node:20` (~1 GB). Fewer OS packages = fewer CVEs. Alpine uses `musl` libc and `apk`; distroless has no package manager or shell at all.

2. **Scan the built image.** `docker scout cves orders-api:1.4.0` (or `trivy image orders-api:1.4.0`) reads the image config, lists every installed package and its version, and cross-references a **vulnerability database** (CVE feed). It prints each package with a known CVE, the severity, and the fixed version.

3. **Create and switch to a non-root user in the Dockerfile.** The `node:*-alpine` image already ships a user named `node` with UID 1000. `USER node` writes `"User": "1000"` into the image config JSON. runc reads this at start and calls `setuid(1000)` before `execve`-ing your process.

4. **Container start — runc applies the security context.** When `docker run` starts the container, containerd → runc (Topic 03) does, in order:
   - Sets up namespaces and cgroups.
   - Mounts the overlay root filesystem; if `--read-only`, passes `MS_RDONLY`.
   - Computes the capability bitmask: starts from Docker's default set, then removes anything in `--cap-drop`.
   - Calls `setgid`/`setuid` to the configured UID.
   - Sets `no_new_privs` if requested.
   - `execve`s your `node server.js`.

5. **The kernel now enforces everything.** From this instant, every syscall your Node process makes is checked against its UID, its capability mask, and the mount flags. A `write()` to `/app` returns `EROFS`. A `bind()` to port 80 returns `EACCES` (no `CAP_NET_BIND_SERVICE`). A `chown()` returns `EPERM`.

6. **Writable scratch space where needed.** Because the root FS is read-only, anything that legitimately needs to write (a temp dir, a cache) gets an explicit `tmpfs` mount: `--tmpfs /tmp`. This is RAM-backed and disappears on stop.

---

## Exact syntax breakdown

### Scanning with docker scout

```
docker scout cves orders-api:1.4.0
       │      │    │
       │      │    └─ image:tag to scan (must exist locally or in a registry)
       │      └────── subcommand: list CVEs (others: quickview, recommendations)
       └───────────── Docker's built-in supply-chain / vuln scanner
```

### Scanning with trivy

```
trivy image --severity HIGH,CRITICAL --exit-code 1 orders-api:1.4.0
  │     │        │                      │              │
  │     │        │                      │              └─ target image
  │     │        │                      └─ return non-zero if anything found → fails CI
  │     │        └─ only report HIGH and CRITICAL (ignore LOW/MEDIUM noise)
  │     └─ scan target type: a container image (also: fs, repo, config)
  └─ the trivy binary (Aqua Security's open-source scanner)
```

### USER instruction in a Dockerfile

```
USER node:node
 │    │    │
 │    │    └─ group name or GID (optional; defaults to the user's primary group)
 │    └────── user name or UID — must already exist in /etc/passwd of the image
 └─────────── every instruction AFTER this line, and the container's main
              process, runs as this user (not root)
```

### Running with a read-only root filesystem + writable tmp

```
docker run --read-only --tmpfs /tmp:rw,size=64m orders-api:1.4.0
              │          │      │   │      │
              │          │      │   │      └─ cap this tmpfs at 64 MB (prevents RAM abuse)
              │          │      │   └─ mount options: read-write
              │          │      └─ path inside container to make writable
              │          └─ mount a fresh RAM-backed filesystem here
              └─ mount the container's root filesystem with MS_RDONLY (EROFS on write)
```

### Dropping capabilities

```
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE orders-api:1.4.0
              │        │     │        │
              │        │     │        └─ add back ONE capability (bind ports <1024)
              │        │     └─ add a specific capability after dropping all
              │        └─ drop EVERYTHING first (empties the bitmask)
              └─ modify the container's Linux capability set
```

The pattern is always **drop ALL, then add back only what you proved you need.** Most Node apps listening on port 3000 need *zero* capabilities.

### The no-new-privileges hardening flag

```
docker run --security-opt=no-new-privileges:true orders-api:1.4.0
              │            │                   │
              │            │                   └─ enable it
              │            └─ the PR_SET_NO_NEW_PRIVS kernel bit — a setuid
              │               binary inside can never escalate to root
              └─ pass a raw security option to runc
```

---

## Example 1 — basic

A hardened Dockerfile for `orders-api`, every line explained.

```dockerfile
# Small base: alpine variant is ~50 MB vs ~1 GB for node:20.
# Pin the digest in real life; tag shown here for readability.
FROM node:20-alpine

# Create the app directory. It will be OWNED by root by default...
WORKDIR /app

# Copy dependency manifests first (Topic 08 — layer caching).
COPY package*.json ./

# Install production deps only. Runs as root (needed to write into /app).
RUN npm ci --omit=dev

# Copy the rest of the source.
COPY . .

# The node:*-alpine image already ships a "node" user with UID 1000.
# Give it ownership of the app so the app can read its own files.
RUN chown -R node:node /app

# From here on, EVERYTHING runs as UID 1000, not root.
USER node

# Document the port. This does NOT need CAP_NET_BIND_SERVICE because 3000 > 1024.
EXPOSE 3000

# Exec form so node is PID 1 and receives signals directly (Topic 07).
CMD ["node", "server.js"]
```

Run it locked down:

```bash
docker run -d \
  --name orders-api \
  --read-only \                         # root FS is read-only
  --tmpfs /tmp:rw,size=32m \            # but /tmp is writable RAM
  --cap-drop ALL \                     # no Linux capabilities at all
  --security-opt=no-new-privileges:true \
  -p 3000:3000 \
  orders-api:1.4.0
```

Verify the identity from inside:

```bash
docker exec orders-api id
# uid=1000(node) gid=1000(node) groups=1000(node)   ← not root!

docker exec orders-api sh -c 'echo test > /app/hack.txt'
# sh: can't create /app/hack.txt: Read-only file system   ← EROFS, exactly as designed
```

---

## Example 2 — production scenario

**The situation.** Your team runs `orders-api` in production. A security researcher reports that a transitive npm dependency (`node-tar`) has a path-traversal CVE, and separately your on-call notices that the running container has `curl`, `wget`, and `bash` — because someone based the image on the full `node:20`, and a leaked `.env` was `COPY . .`-ed in.

You do a full hardening pass.

**Step 1 — switch to distroless for the runtime stage** (multi-stage, from Topic 09). Distroless has *no shell, no package manager, no curl* — nothing for an attacker to use.

```dockerfile
# ---- build stage: has npm, needs to compile/install ----
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .

# ---- runtime stage: distroless, ships ONLY node + your app ----
FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
# Copy just the built artifacts from the build stage.
COPY --from=build /app /app
# distroless "nonroot" variant runs as UID 65532 by default; be explicit:
USER 1000
EXPOSE 3000
# distroless has no shell, so CMD MUST be exec form. Entrypoint is already "node".
CMD ["server.js"]
```

**Step 2 — keep the secret OUT of the image.** The `.env` leak happened because `COPY . .` swept it in. Fix it two ways (belt and suspenders):

```
# .dockerignore  (Topic 10) — never let it into the build context
.env
.env.*
*.pem
```

And inject config at **run time** instead (Topic 15), so it lives only in the container's env, not any layer:

```bash
docker run --env-file /etc/secrets/orders-api.env orders-api:1.5.0
```

**Step 3 — gate deploys on a scan.** Add trivy to CI so a HIGH/CRITICAL CVE (like the `node-tar` one) blocks the merge:

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 orders-api:1.5.0
# Exit code 1 → the GitHub Actions job fails → the PR can't merge until you bump node-tar.
```

**Step 4 — prove the secret is gone** from the image history:

```bash
docker history --no-trunc orders-api:1.5.0 | grep -i env    # nothing baked in
docker save orders-api:1.5.0 -o img.tar && tar xf img.tar && grep -r "postgres://" .
# (no matches) — the DB URL exists only at runtime now
```

The result: a ~120 MB image with no shell, no package manager, running as UID 1000 on a read-only filesystem with zero capabilities, and a CI gate that refuses to ship known-critical CVEs. The same SSRF bug that used to mean "attacker owns the host" now means "attacker is stuck in an empty read-only jail."

---

## Common mistakes

### Mistake 1 — `USER` before the files exist / wrong ownership

```dockerfile
USER node
COPY . .          # copied files are owned by root; node can't write logs/cache
RUN npm ci        # ✗ EACCES: permission denied, mkdir '/app/node_modules'
```

**Error you see:**

```
npm error code EACCES
npm error syscall mkdir
npm error path /app/node_modules
npm error errno -13
```

**Root cause:** After `USER node`, the process is UID 1000. `/app` is owned by root (UID 0). The kernel's inode permission check (`inode_permission()`) sees the process lacks write permission on a root-owned dir and returns `EACCES`. **Fix:** do all installs as root, `chown -R node:node /app`, and put `USER node` *last*.

### Mistake 2 — read-only filesystem with no writable temp

```bash
docker run --read-only orders-api:1.4.0
# App crashes: EROFS: read-only file system, open '/tmp/upload-xyz'
```

**Root cause:** Your app (or a library like `multer`, or the npm cache) writes to `/tmp`. With `--read-only`, the overlay mount has `MS_RDONLY`, so the kernel returns `EROFS`. **Fix:** add `--tmpfs /tmp`. Find every write path with `strace -f -e trace=openat` and mount a tmpfs for each.

### Mistake 3 — binding to port 80 as a non-root user

```
Error: listen EACCES: permission denied 0.0.0.0:80
```

**Root cause:** Ports below 1024 are "privileged." The kernel's `inet_bind()` checks for `CAP_NET_BIND_SERVICE`; a non-root process with dropped caps doesn't have it, so `bind()` returns `EACCES`. **Fix:** listen on 3000 inside the container and map with `-p 80:3000`, OR add `--cap-add NET_BIND_SERVICE`. Prefer the high port.

### Mistake 4 — secrets baked into a layer

```dockerfile
ARG DB_PASSWORD
RUN echo "DATABASE_URL=postgres://app:$DB_PASSWORD@db/orders" > /app/.env
```

**Why it's broken:** even if you later `RUN rm /app/.env`, the earlier layer still contains it. Layers are immutable diffs (Topic 04); deleting a file in a *later* layer just adds a whiteout — the secret is still in the tarball. Anyone runs `docker save` + `tar xf` and greps it out. **Fix:** never write secrets to disk during build. Use runtime `--env-file`, Docker BuildKit secrets (`RUN --mount=type=secret`), or an orchestrator's secret store.

### Mistake 5 — trusting `latest` and never scanning

```dockerfile
FROM node:latest      # yesterday's "latest" had 3 critical CVEs; you'll never know
```

**Root cause:** `latest` is a moving target and unscanned images rot — a base that was clean at build time accumulates CVEs as new ones are disclosed. **Fix:** pin a specific tag/digest, and **re-scan images on a schedule**, not just at build. `docker scout cves` against your running image tag catches newly disclosed CVEs in an image you built months ago.

---

## Hands-on proof

Run these right now to *see* the kernel enforcing each control.

```bash
# 1. Build the hardened image from Example 1 (save the Dockerfile first).
docker build -t orders-api:1.4.0 .

# 2. Prove it runs as non-root.
docker run --rm orders-api:1.4.0 id
#   uid=1000(node) gid=1000(node)      ← NOT uid=0(root)

# 3. Prove the read-only filesystem rejects writes.
docker run --rm --read-only orders-api:1.4.0 sh -c 'touch /app/x'
#   touch: /app/x: Read-only file system      ← EROFS from the kernel

# 4. Inspect the capability bitmask of a cap-dropped container.
docker run --rm --cap-drop ALL alpine sh -c 'grep CapEff /proc/self/status'
#   CapEff: 0000000000000000        ← zero capabilities

# 5. Decode a capability mask to human-readable names.
docker run --rm alpine sh -c 'apk add -q libcap; grep CapEff /proc/self/status'
#   compare a default container (non-zero) to the --cap-drop ALL one above

# 6. Scan for CVEs (installs nothing if you have Docker Desktop's scout).
docker scout cves orders-api:1.4.0
#   lists packages with known CVEs, severity, and fixed versions
#   (or:  trivy image orders-api:1.4.0 )

# 7. PROVE a leaked secret is extractable from a layer.
printf 'FROM alpine\nRUN echo "SECRET=hunter2" > /s.txt\nRUN rm /s.txt\n' > Dockerfile.leak
docker build -f Dockerfile.leak -t leaky .
docker save leaky -o leaky.tar && mkdir leak && tar xf leaky.tar -C leak
grep -r "hunter2" leak/          ← FOUND, even though we "rm"-ed it
```

Step 7 is the one that changes how you think forever: the secret is still there after `rm`.

---

## Practice exercises

### Exercise 1 — easy
Take the plain `orders-api` Dockerfile (`FROM node:20`, no USER) and harden it: switch to `node:20-alpine`, add a `chown` + `USER node`, and confirm with `docker run --rm <img> id` that it no longer runs as root.

### Exercise 2 — medium
Run your hardened image with `--read-only`. It will crash on the first write. Use `docker logs` (and `strace -f -e trace=openat` if you have it) to find *every* path the app writes to, and add the minimal set of `--tmpfs` mounts so it starts cleanly. Document each write path and why it exists.

### Exercise 3 — hard (production simulation)
Build two images of `orders-api`: one `FROM node:20`, one `FROM gcr.io/distroless/nodejs20-debian12` (multi-stage). For each, (a) record the size, (b) run `trivy image` and record the HIGH/CRITICAL count, (c) try to `docker exec <c> sh` and note which one has no shell. Then write a `docker run` command for the distroless one that is fully locked down (`--read-only`, `--cap-drop ALL`, `--security-opt=no-new-privileges`, non-root, writable `/tmp`) and prove all four controls are active by inspecting `/proc/self/status` and `/proc/mounts`.

---

## Mental model checkpoint

Answer these from memory:

1. Where in the kernel is a process's UID stored, and what UID does `USER node` produce?
2. What does `--cap-drop ALL` change in `/proc/<PID>/status`, and what does a value of `0000000000000000` mean?
3. Why does a `write()` to a `--read-only` container return `EROFS` — what mount flag causes it?
4. Why can a non-root container still not bind port 80, and which capability fixes it?
5. Why does `RUN rm secret.txt` NOT remove a secret from an image? What are layers physically?
6. Name two reasons distroless is more secure than `node:20-alpine`.
7. What does `trivy image --exit-code 1` do that makes it useful in CI?

---

## Quick reference card

| Command / instruction / flag | What it does | Key detail |
|---|---|---|
| `docker scout cves IMG` | List known CVEs in an image | Built into Docker Desktop; cross-refs a CVE feed |
| `trivy image --exit-code 1 IMG` | Scan; fail on findings | Exit 1 breaks CI — the enforcement point |
| `USER node` (Dockerfile) | Run process as UID 1000, not root | Put it AFTER installs; user must exist in image |
| `--read-only` | Mount root FS read-only (`MS_RDONLY`) | Writes return `EROFS`; pair with `--tmpfs` |
| `--tmpfs /tmp:rw,size=32m` | RAM-backed writable dir | Vanishes on stop; cap the size |
| `--cap-drop ALL` | Empty the capability bitmask | Add back only what you prove you need |
| `--cap-add NET_BIND_SERVICE` | Allow binding ports <1024 | Usually unnecessary — use a high port |
| `--security-opt=no-new-privileges` | Set `PR_SET_NO_NEW_PRIVS` | Blocks setuid privilege escalation |
| `FROM ...-alpine` | ~50 MB base, `musl` + `apk` | Far fewer packages → fewer CVEs |
| `FROM gcr.io/distroless/nodejs20` | No shell, no package manager | Nothing for an attacker to run |
| `.dockerignore` with `.env` | Keep secrets out of build context | First line of defense against baked secrets |

---

## When would I use this at work?

1. **Passing a security audit / SOC 2.** Auditors ask "do your containers run as non-root?" and "do you scan images for CVEs?" This topic is the literal checklist: non-root `USER`, `--cap-drop`, read-only FS, and a scanning gate in CI.

2. **Responding to a disclosed CVE.** A critical OpenSSL or `libwebp` CVE drops. You run `docker scout cves` / `trivy` across your images to find which are affected, bump the base image, rebuild, and redeploy — without guessing.

3. **Limiting blast radius after a bug.** Your `orders-api` has an SSRF or RCE. Because the container is non-root, read-only, capability-stripped, and shell-less, the attacker can't pivot, install tools, or persist — turning a potential breach into a contained annoyance.

---

## Connected topics

- **Study before:** Topic 02 (Linux Foundations — namespaces, cgroups, UIDs, capabilities), Topic 04 (Images and Layers — why layers are immutable tarballs), Topic 09 (Multi-Stage Builds — how you get a small runtime image), Topic 15 (Environment Variables and Secrets — where secrets should live).
- **Study after:** Topic 24 (Docker Registry — signing and pulling trusted images), Topic 25 (Resource Limits — the other half of container hardening), Topic 27 (Docker in CI/CD — where the scan gate runs), Topic 53 (RBAC — least privilege carried into Kubernetes).
