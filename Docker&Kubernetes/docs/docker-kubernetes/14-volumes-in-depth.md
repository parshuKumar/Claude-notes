# 14 — Volumes in Depth
## Section: Docker in Practice

## ELI5 — The Simple Analogy

A container's own filesystem is like a **hotel room**. You can move furniture around and scribble on the whiteboard, but the moment you check out (the container is deleted), housekeeping wipes everything. Anything you leave in the room is gone.

If you want to *keep* your stuff, you need one of three options:

- **Named volume** = a **storage locker in the hotel basement** that the hotel manages for you. You put your Postgres data there. Check out, check into a brand-new room next week, ask for the same locker — all your data is still there. The hotel decides exactly where the locker physically sits; you just know its name (`pgdata`).
- **Bind mount** = a **tunnel from your room straight into your own house**. Whatever is in that folder on your house (the host) appears live in the room, and edits go both ways instantly. Great for "I'm editing code at home and want the room to see my changes immediately."
- **tmpfs mount** = a **magic whiteboard that only exists while you're in the room** and is stored in the room's *air* (RAM), never on paper. Super fast, wiped the instant you leave, never touches disk. Good for secrets and scratch data you never want written down.

## The Linux kernel feature underneath

All three volume types are ultimately the kernel's **`mount()` syscall** placing something at a path inside the container's **mount namespace** (remember from Topic 02 — each container has its own private view of the filesystem tree).

```
                    mount() syscall  +  mount namespace (CLONE_NEWNS)
                              │
        ┌─────────────────────┼─────────────────────────┐
        ▼                     ▼                          ▼
  BIND MOUNT             NAMED VOLUME                TMPFS MOUNT
  mount(src, dst,        mount(volpath, dst,         mount("tmpfs", dst,
        MS_BIND)               MS_BIND)  where             "tmpfs", ...)
        │                  volpath lives under            │
  a host directory        /var/lib/docker/volumes/   RAM-backed filesystem
  grafted into the        <name>/_data               (no disk file at all)
  container's tree
```

The heart of it:

- **Bind mount** — `mount(source, target, MS_BIND)` grafts an *existing host directory* onto a path in the container's mount namespace. Same inodes, same data, two names. Edits are literally the same bytes on disk.
- **Named volume** — Docker first creates a directory at `/var/lib/docker/volumes/<name>/_data`, then bind-mounts *that* into the container. The difference from a plain bind mount is **ownership**: Docker manages that directory's lifecycle, and it's isolated from the rest of your host filesystem.
- **tmpfs** — `mount("tmpfs", target, "tmpfs", ...)` creates a filesystem that lives entirely in **RAM** (and swap). There is no backing file on disk; when the mount goes away, the pages are freed and the data is gone.

Because all three land in the container's *mount namespace*, they are invisible to other containers unless explicitly shared, and they overlay whatever was at that path in the image.

## What is this?

Volumes are the mechanism for storing data **outside** a container's ephemeral writable layer, so it survives the container being stopped, deleted, or replaced. Docker offers three kinds: **bind mounts** (map a host path in), **named volumes** (Docker-managed storage under `/var/lib/docker/volumes/`), and **tmpfs** (RAM-only, never persisted). Choosing correctly is the difference between durable data and data loss.

## Why does it matter for a backend developer?

Because **containers are disposable, but your data is not.** Remember from Topic 04: a container's writable layer is a thin overlay `diff/` directory that Docker **deletes** when the container is removed. So:

- Run Postgres with no volume → `docker rm db` → **every order in your database is gone.** Forever.
- Develop `orders-api` with no bind mount → edit code → have to rebuild the image and recreate the container for *every single change*. Painful.
- Store secrets or session scratch on the normal filesystem → they get written to disk in the overlay, potentially captured in `docker commit` or forensics. tmpfs keeps them in RAM only.

Volumes are also *the* topic that maps directly to Kubernetes PersistentVolumes (Topic 39). Understanding Docker volumes is the on-ramp to stateful workloads in K8s.

## The physical reality

Say we run:

```bash
docker volume create pgdata
docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres:16-alpine
docker run -d --name api -v /home/dev/orders-api:/app --tmpfs /tmp orders-api:1.0
```

On disk and in the kernel:

```
/var/lib/docker/volumes/
├── pgdata/
│   ├── _data/                        ← the ACTUAL Postgres data directory lives here
│   │   ├── base/
│   │   ├── pg_wal/
│   │   ├── postgresql.conf
│   │   └── ...                        ← survives `docker rm db` completely
│   └── (metadata tracked in Docker's volume store)
└── metadata.db                        ← Docker's registry of volumes

/home/dev/orders-api/                  ← YOUR host source dir (bind mount source)
├── server.js                          ← edit this, container sees it instantly
├── package.json
└── ...

INSIDE the api container's mount namespace (cat /proc/<PID>/mountinfo):
/app          ← bind-mounted from /home/dev/orders-api (rw, same inodes)
/tmp          ← tmpfs, backed by RAM only, size-limited, wiped on stop
/var/lib/postgresql/data (in db)  ← bind of /var/lib/docker/volumes/pgdata/_data
```

You can prove the named volume is a real host directory:

```bash
sudo ls /var/lib/docker/volumes/pgdata/_data
# base  global  pg_wal  postgresql.conf  ...  ← Postgres's files, on the host, right there
```

And you can prove tmpfs is RAM-backed:

```bash
docker exec api df -h /tmp
# Filesystem  Size  Used  Avail  Use%  Mounted on
# tmpfs        64M     0    64M    0%   /tmp     ← type tmpfs, not overlay
```

## How it works — step by step

Trace of `docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres:16-alpine`:

1. **CLI sends create request** with a mount spec: `{Type: volume, Source: pgdata, Target: /var/lib/postgresql/data}`.
2. **Daemon checks the volume store.** Is there a volume named `pgdata`? If not, it **creates** `/var/lib/docker/volumes/pgdata/_data` (an empty directory) using the `local` volume driver.
3. **Daemon assembles the container's overlay rootfs** (Topic 04) — image layers as lower, fresh writable `diff/` as upper, `merged/` as `/`.
4. **runc sets up the mount namespace.** It first mounts `merged/` as `/`.
5. **First-run data seeding (important!):** because the target `/var/lib/postgresql/data` inside the *image* is **empty** and the named volume is **also empty**, Docker **copies the image's contents at that path into the volume** on first use. (This special "populate on first mount" behavior applies to *named volumes only*, not bind mounts.) For Postgres the image path is empty, but for images that ship default content at the mount point, this seeding matters.
6. **runc performs `mount(/var/lib/docker/volumes/pgdata/_data, /var/lib/postgresql/data, MS_BIND)`** inside the container's mount namespace. Now that container path *is* the host volume directory.
7. **Postgres starts**, runs `initdb`, and writes `base/`, `pg_wal/`, etc. — those writes land in `/var/lib/docker/volumes/pgdata/_data` on the host, **not** in the container's throwaway overlay layer.
8. **You `docker rm -f db`.** Docker deletes `/var/lib/docker/containers/<id>/` and the overlay `diff/`. But the volume `pgdata` is **left untouched** — it is not owned by the container.
9. **You `docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres:16-alpine` again.** Step 2 finds the existing `pgdata`, step 5 sees it's *not* empty and skips seeding, step 6 mounts it. Postgres starts and finds all its old data. **Zero data loss.**

Bind mount differs at step 2/5: no volume is created, no seeding ever happens, and the host source directory **shadows** whatever the image had at the target (its contents are hidden, not merged).

## Exact syntax breakdown

### `-v` short syntax — three shapes

```
-v pgdata:/var/lib/postgresql/data
   │      │
   │      └─ target path INSIDE container
   └─ NO leading slash → NAMED VOLUME (Docker-managed under /var/lib/docker/volumes/)

-v /home/dev/orders-api:/app
   │                    │
   │                    └─ target path inside container
   └─ leading slash (absolute host path) → BIND MOUNT

-v /var/lib/postgresql/data
   │
   └─ only ONE path (a container path) → ANONYMOUS VOLUME (random name, easy to lose)
```

Add options after a third colon:

```
-v /home/dev/orders-api:/app:ro
   │                        │
   │                        └─ mount read-only (container cannot write to /app)
   └─ ...
```

### `--mount` long syntax (explicit, preferred for scripts)

```
--mount type=volume,source=pgdata,target=/var/lib/postgresql/data
        │           │             │
        │           │             └─ path inside container
        │           └─ volume name (or src= for a bind path)
        └─ volume | bind | tmpfs
```

```
--mount type=bind,source=/home/dev/orders-api,target=/app,readonly
        │         │                            │           │
        │         │                            │           └─ read-only flag
        │         │                            └─ container path
        │         └─ absolute host path (must already exist by default)
        └─ bind mount
```

```
--mount type=tmpfs,target=/tmp,tmpfs-size=67108864,tmpfs-mode=1770
        │          │           │                    │
        │          │           │                    └─ permission bits on the mount
        │          │           └─ max size in bytes (64 MiB here)
        │          └─ container path (RAM-backed)
        └─ tmpfs mount
```

**`-v` vs `--mount`:** they mostly do the same thing. Key differences: `-v` will **auto-create** a missing bind-source directory on the host; `--mount` **errors** if the bind source doesn't exist (safer — catches typos). `--mount` is more readable and is required for some advanced options.

### `--tmpfs` shorthand

```
--tmpfs /tmp:size=64m,mode=1777
        │    │         │
        │    │         └─ permissions (1777 = world-writable + sticky, like real /tmp)
        │    └─ options after the colon
        └─ container path, backed by RAM only
```

### Managing volumes

```
docker volume create pgdata      → create a named volume explicitly
docker volume ls                 → list all volumes
docker volume inspect pgdata     → show Mountpoint (the real /var/lib/docker/... path)
docker volume rm pgdata          → delete a volume (fails if in use)
docker volume prune              → delete ALL unused volumes (dangerous! removes data)
```

## Example 1 — basic

Prove that a named volume persists data across container deletion:

```bash
# 1. Create a container that writes a file into a named volume
docker run --rm -v mydata:/data alpine \
  sh -c 'echo "order #1001 persisted" > /data/orders.txt'   # container exits & is removed

# 2. The container is GONE, but run a brand-new one on the SAME volume
docker run --rm -v mydata:/data alpine cat /data/orders.txt
# → order #1001 persisted     ← data survived a full container lifecycle!

# 3. Show where it physically lives on the host
docker volume inspect mydata --format '{{ .Mountpoint }}'
# → /var/lib/docker/volumes/mydata/_data

# 4. (On a Linux host) read it directly from disk — no container needed
sudo cat /var/lib/docker/volumes/mydata/_data/orders.txt
# → order #1001 persisted

# Cleanup
docker volume rm mydata
```

Two separate, deleted containers shared one durable volume. That is exactly how Postgres keeps your orders safe across redeploys.

## Example 2 — production scenario

**Scenario:** Your team runs `orders-api` (Node), Postgres, and Redis. Three requirements collided last quarter:

1. A redeploy of Postgres **wiped the production database** (no volume). Never again.
2. Developers were rebuilding the image on **every code change** — 40-second feedback loops killing productivity.
3. A security review flagged that the API wrote **JWT signing keys** to `/tmp` on the container's disk layer, where they could be recovered.

Here is the volume strategy that solves all three:

```bash
# --- Postgres: NAMED VOLUME for durable data ---
docker volume create pgdata
docker run -d --name db --network orders-net \
  -v pgdata:/var/lib/postgresql/data \        # data survives ANY container replacement
  -e POSTGRES_PASSWORD=secret \
  --memory 1g --restart unless-stopped \
  postgres:16-alpine

# --- Dev API: BIND MOUNT source for instant hot reload ---
docker run -d --name api-dev --network orders-net -p 3000:3000 \
  -v /home/dev/orders-api/src:/app/src \      # edit host files → container sees them live
  -v /app/node_modules \                      # anonymous vol shields node_modules from host
  orders-api:dev \
  npx nodemon src/server.js                   # nodemon restarts on file change — no rebuild

# --- Prod API: TMPFS for secrets, read-only rootfs ---
docker run -d --name api-prod --network orders-net -p 8080:3000 \
  --read-only \                               # entire rootfs read-only (Topic 23)
  --tmpfs /tmp:size=16m,mode=1777 \           # JWT keys/scratch live in RAM, never on disk
  --restart unless-stopped \
  orders-api:1.4.2
```

Why each choice:

- **Postgres → named volume:** Docker manages it, it's isolated, it survives `docker rm`, and it's the pattern that maps cleanly to a Kubernetes PVC later (Topic 39).
- **Dev API → bind mount:** the host directory is grafted live into the container, so `nodemon` sees edits instantly — no rebuild, sub-second feedback. The extra anonymous volume on `/app/node_modules` is a classic trick: it prevents your host's (possibly empty or platform-mismatched) `node_modules` from shadowing the ones baked into the image.
- **Prod API → tmpfs:** signing keys written to `/tmp` live in RAM only. When the container stops, the pages are freed — nothing recoverable from disk, nothing in the overlay layer, nothing in a `docker commit`.

## Common mistakes

### Mistake 1 — Running a database with no volume

```bash
docker run -d --name db -e POSTGRES_PASSWORD=secret postgres:16-alpine
# ... weeks of real orders ...
docker rm -f db     # redeploy
```
- **Symptom:** the new `db` starts with an **empty** database. Every order is gone.
- **Root cause:** with no `-v`, Postgres wrote to the container's **overlay writable layer**, which Docker deletes on `docker rm`. That layer is ephemeral by design (Topic 04).
- **Wrong:** `docker run -d --name db postgres:16-alpine`
- **Right:** `docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres:16-alpine`

### Mistake 2 — Bind mount shadows the image's files

```bash
# You bind-mount your host source over /app, but forgot host has no node_modules
docker run -v /home/dev/orders-api:/app orders-api:1.0
# → Error: Cannot find module 'express'
```
- **Root cause:** a bind mount **completely replaces** whatever the image had at `/app`. The image's baked-in `/app/node_modules` is now hidden behind your host directory, which lacks them.
- **Right:** add an anonymous/named volume for the modules path so it isn't shadowed: `-v /home/dev/orders-api:/app -v /app/node_modules`. The more specific mount wins for that subpath.

### Mistake 3 — `docker volume prune` deletes production data

```bash
docker volume prune
# WARNING! This will remove all local volumes not used by at least one container.
```
- **Root cause:** `prune` removes every volume **not currently attached to a container**. If your `db` container is temporarily stopped/removed during a maintenance window, `pgdata` counts as "unused" and gets wiped.
- **Right:** never `prune` volumes blindly in an environment with real data. Use targeted `docker volume rm <name>` and keep backups. Label important volumes and script carefully.

### Mistake 4 — Expecting tmpfs data to persist

```bash
docker run -d --name api --tmpfs /cache orders-api:1.0
docker restart api
# /cache is empty again — where did my cached data go?
```
- **Root cause:** tmpfs lives in **RAM**. Stopping the container frees the pages; the data does not survive a restart, let alone removal. That is the entire point of tmpfs.
- **Right:** use tmpfs only for scratch/secrets you *want* gone. For a cache that should survive restart, use a named volume.

### Mistake 5 — Permission denied on a bind mount (UID mismatch)

```bash
docker run -v /home/dev/data:/data --user 1000 postgres:16-alpine
# → could not create directory "/data/...": Permission denied
```
- **Root cause:** bind mounts keep the **host's ownership/UIDs**. The container process (a specific UID) may not own the host directory. There's no UID remapping unless you use user namespaces.
- **Right:** align ownership — `sudo chown -R 1000:1000 /home/dev/data` on the host, or run the container as a matching user, or use a **named volume** (Docker initializes it with the right ownership for the image's default user on first mount).

## Hands-on proof

```bash
# 1. Named volume persists across container deletion
docker run --rm -v proof:/d alpine sh -c 'echo hello > /d/f.txt'
docker run --rm -v proof:/d alpine cat /d/f.txt          # → hello (container was deleted between runs)

# 2. Find its real location on the host
docker volume inspect proof --format '{{.Mountpoint}}'   # → /var/lib/docker/volumes/proof/_data
sudo ls -l /var/lib/docker/volumes/proof/_data           # → f.txt sitting on the host disk

# 3. Bind mount is two-way and shares inodes
mkdir -p /tmp/bindtest && echo "from host" > /tmp/bindtest/x.txt
docker run --rm -v /tmp/bindtest:/m alpine sh -c 'cat /m/x.txt; echo "from container" >> /m/x.txt'
cat /tmp/bindtest/x.txt                                  # → both lines: host + container edits

# 4. tmpfs is RAM-backed and wiped on stop
docker run -d --name tf --tmpfs /ram:size=8m alpine sleep 300
docker exec tf sh -c 'echo secret > /ram/s.txt; df -h /ram'   # → Filesystem type tmpfs
docker restart tf
docker exec tf cat /ram/s.txt                            # → No such file — gone after restart

# 5. See the actual mounts inside the container's mount namespace
docker exec tf cat /proc/1/mountinfo | grep -E 'ram|overlay'

# Cleanup
docker rm -f tf; docker volume rm proof; rm -rf /tmp/bindtest
```

## Mount propagation (the advanced bit)

Mount propagation controls what happens when a **new mount is created inside a directory that is itself a bind mount**. It matters when a container mounts something (e.g. a filesystem, a device) and you need the host — or another container — to see it, or vice versa.

```
-v /host/path:/container/path:rshared
                              │
                              └─ propagation mode
```

Modes (from the Linux kernel's shared-subtree feature):

| Mode | Meaning |
|------|---------|
| `rprivate` (default) | Mounts made on either side stay **private** to that side |
| `rshared` | Mounts made inside propagate **both ways** (host ↔ container) |
| `rslave` | Mounts made on the **host** propagate **into** the container, but not the reverse |
| `shared`/`slave`/`private` | Same but non-recursive (single mount point) |

Real-world need: a monitoring/backup agent container that must see filesystems the host mounts *after* the container started (e.g. a newly attached disk) uses `rslave` so host mounts appear inside. This is exactly the mechanism Kubernetes uses for its `mountPropagation` field on volumes (relevant later in Topic 39). For most app containers, the default `rprivate` is correct and you never touch this.

## Practice exercises

### Exercise 1 — easy
Create a named volume `notes`. Write a file into it from one `--rm` container, then read it back from a *different* `--rm` container. Use `docker volume inspect` to print the real `/var/lib/docker/volumes/...` path, and (on Linux) `sudo cat` the file directly from the host.

### Exercise 2 — medium
Set up a dev-style bind mount: create `/tmp/app` on the host with an `index.js` that prints "v1". Run `node:20-alpine` with `-v /tmp/app:/app -w /app` running `node index.js`... then edit the host file to print "v2" and run again *without rebuilding anything*. Confirm the container sees "v2". Explain why (inode-level) the change appeared instantly.

### Exercise 3 — hard (production simulation)
Simulate a Postgres redeploy with zero data loss:
1. Run Postgres with `-v pgdata:/var/lib/postgresql/data`, create a table `orders`, insert 3 rows.
2. `docker rm -f db` (simulate a redeploy).
3. Run a *new* Postgres container with the same volume and prove the 3 rows are still there.
4. Now do a **volume backup**: use `docker run --rm -v pgdata:/data -v $(pwd):/backup alpine tar czf /backup/pg.tar.gz -C /data .` and explain, step by step, how this "sidecar" container reads one volume and writes the archive to a bind-mounted host directory.

## Mental model checkpoint

1. What are the three volume types, and what is the one-line use case for each?
2. Which kernel syscall underlies all three, and into which namespace does the mount land?
3. Where on the host does a named volume physically live?
4. Why does a database with no volume lose all data on `docker rm`?
5. What does a bind mount do to the image's original contents at the target path?
6. Why is tmpfs a good place for secrets, and what happens to that data on container stop?
7. What does the "populate on first mount" behavior do, and does it apply to bind mounts?
8. What does `docker volume prune` remove, and why is it dangerous in production?

## Quick reference card

| Syntax / command | What it does | Key detail |
|------------------|--------------|------------|
| `-v name:/path` | Named volume | Under `/var/lib/docker/volumes/name/_data`; Docker-managed |
| `-v /host:/path` | Bind mount | Host dir grafted in; shadows image's path; two-way |
| `-v /path` | Anonymous volume | Random name; easy to lose; deleted with `--rm` |
| `-v ...:ro` | Read-only mount | Container can't write |
| `--tmpfs /path` | tmpfs mount | RAM-only; wiped on stop; good for secrets |
| `--mount type=...` | Explicit long form | Errors if bind source missing (safer than `-v`) |
| `docker volume create X` | Make named volume | Empty `_data` dir on host |
| `docker volume inspect X` | Show real path | `.Mountpoint` = host location |
| `docker volume rm X` | Delete a volume | Fails if attached to a container |
| `docker volume prune` | Delete unused volumes | DANGER: removes data of stopped containers |
| `:rshared` / `:rslave` | Mount propagation | Controls mount visibility host↔container |

## When would I use this at work?

1. **Persisting the `orders` database:** every stateful service (Postgres, Redis with AOF, uploaded files) gets a named volume so redeploys and image upgrades never touch the data — the single most important production habit.
2. **Fast local development:** bind-mount your `orders-api` source into the container with `nodemon` so code edits reload in under a second, with no image rebuild — while an anonymous volume protects `node_modules`.
3. **Handling secrets and scratch data:** mount `/tmp` or a secrets path as tmpfs so signing keys, session scratch, and decrypted config never hit disk, satisfying security reviews and reducing forensic exposure.

## Connected topics

- **Study before:** Topic 02 (Linux Foundations) — mount namespaces; Topic 04 (Images and Layers) — the ephemeral writable overlay layer volumes exist to escape; Topic 12 (docker run in Depth) — where `-v` and `--tmpfs` are introduced.
- **Study after:** Topic 15 (Env Vars and Secrets) — file-based secrets often ride on tmpfs mounts; Topic 20 (Multi-Container Apps) — Compose `volumes:` for Postgres/Redis; Topic 39 (Persistent Volumes and Claims) — the Kubernetes evolution of named volumes, including `mountPropagation` and dynamic provisioning.
