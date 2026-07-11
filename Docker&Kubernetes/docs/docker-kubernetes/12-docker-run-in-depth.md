# 12 — docker run in Depth
## Section: Docker in Practice

## ELI5 — The Simple Analogy

Imagine you have a blueprint for a tiny apartment (that's your **image**). `docker run` is the moment you actually *build and move into* a real apartment from that blueprint.

But before you move in, you get to fill out a move-in form:

- Do you want a **name** on the mailbox? (`--name`)
- Should the front door have a **phone line** connected to the street so visitors can reach you? (`-p`)
- Do you want a **shared storage closet** that survives even if you demolish the apartment? (`-v`)
- What's the **maximum electricity and water** you're allowed to use? (`--memory`, `--cpus`)
- If the apartment catches fire, should it **rebuild itself automatically**? (`--restart`)
- Do you want to move in **quietly in the background** (`-d`) or **stand inside talking to it live** (`-it`)?

`docker run` is one command, but each flag is a knob that changes something *real* in the Linux kernel — a namespace, a cgroup, an iptables rule, a mount. This topic is about what each knob physically does.

## The Linux kernel feature underneath

`docker run` is not one kernel feature — it is an **orchestration of several**. When you type `docker run`, the daemon (remember from Topic 03, the CLI just sends an HTTP request to `dockerd`) eventually asks `runc` to make syscalls that assemble a container out of these kernel primitives:

```
docker run  ──►  dockerd  ──►  containerd  ──►  runc  ──►  KERNEL SYSCALLS
                                                            │
   ┌────────────────────────────────────────────────────────┘
   ▼
 clone(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWUTS | ...)  ← namespaces
 write to /sys/fs/cgroup/.../memory.max, cpu.max                        ← cgroups (limits)
 mount("overlay", ...)                                                  ← overlayfs (rootfs)
 mount(source, target, MS_BIND)                                         ← bind mounts (-v)
 iptables -t nat -A DOCKER ...                                          ← NAT (-p)
 setns / veth pair creation                                             ← network (--network)
```

Every flag you pass to `docker run` maps to **one or more of these syscalls**. That is the whole secret of this topic: a flag is just a human-friendly name for a kernel operation.

- `--memory` → writes a number into a cgroup file.
- `-p` → inserts an iptables NAT rule.
- `-v` → performs a `mount()` syscall.
- `-e` → sets a string in the new process's environment (`environ`).
- `--name` → just a label in the daemon's database, not a kernel thing at all.

Keep that mental split in mind: some flags touch the kernel deeply, some are pure Docker bookkeeping.

## What is this?

`docker run` creates **and** starts a new container from an image in a single command. It is actually two operations glued together: `docker create` (build the container's config and filesystem) followed by `docker start` (actually launch the process inside its namespaces and cgroups). The dozens of flags let you control isolation, networking, storage, resource limits, and lifecycle.

## Why does it matter for a backend developer?

Because `docker run` is where your `orders-api` meets reality. Get the flags wrong and you get the classic production incidents:

- Forgot `-p` → your API runs fine but **nobody can reach it**.
- Forgot `-v` → your Postgres data **vanishes** the moment the container is removed.
- Forgot `--memory` → one memory leak in Node.js **eats the whole host** and takes down Redis and Postgres with it.
- Used `--network host` carelessly → your container's port **collides** with the host and leaks isolation.
- Forgot `--restart` → the daemon reboots, and your API **never comes back up**.

Understanding each flag at the kernel level means you can predict *exactly* what will happen before you hit Enter, and debug it when it goes wrong.

## The physical reality

When you run a container, real things appear on disk and in the kernel. Let's say we run:

```bash
docker run -d --name orders-api -p 8080:3000 orders-api:1.0
```

Here is what physically exists afterward:

```
/var/lib/docker/
├── containers/
│   └── 3f9a...e21/                         ← one dir per container (full 64-char ID)
│       ├── config.v2.json                  ← the container's full spec (name, env, ports...)
│       ├── hostconfig.json                 ← host-side config (port bindings, restart policy, limits)
│       ├── 3f9a...e21-json.log             ← stdout/stderr captured here (Topic 17)
│       ├── hostname
│       ├── hosts                           ← bind-mounted into container as /etc/hosts
│       └── resolv.conf                     ← bind-mounted into container as /etc/resolv.conf
├── overlay2/
│   └── <layer-id>/                         ← the writable + lower layers (Topic 04)
│       ├── merged/                         ← the container's live root filesystem
│       ├── diff/                           ← the writable upper layer
│       └── work/
└── image/overlay2/imagedb/...              ← image metadata

/sys/fs/cgroup/system.slice/docker-3f9a...e21.scope/   ← cgroup v2 dir for THIS container
├── memory.max                              ← from --memory
├── memory.current                          ← live memory usage
├── cpu.max                                 ← from --cpus
├── cpu.stat                                ← throttling counters
└── pids.max

/proc/<PID>/ns/                             ← the container process's namespace handles
├── pid     → pid:[4026532210]
├── net     → net:[4026532213]
├── mnt     → mnt:[4026532208]
├── uts     → uts:[4026532209]
└── ...
```

You can literally `cat` these files. The container is not magic — it is a normal Linux process (`PID` visible in `ps` on the host) that has been placed inside private namespaces and a cgroup, reading a filesystem stitched together by overlayfs.

## How it works — step by step

Full trace of `docker run -d --name orders-api -p 8080:3000 -e NODE_ENV=production --memory 512m --cpus 1.5 orders-api:1.0`:

1. **CLI parses flags** and builds a JSON body. It POSTs to `unix:///var/run/docker.sock` → `POST /containers/create`.
2. **Daemon validates the image** `orders-api:1.0` exists locally in `/var/lib/docker/image/overlay2/`. If not, it would pull it (Topic 04).
3. **Daemon writes `config.v2.json` and `hostconfig.json`** under `/var/lib/docker/containers/<id>/`. At this point the container exists in state `Created` but no process runs yet.
4. **Overlay filesystem is assembled.** The daemon mounts the image's read-only layers as `lowerdir` and a fresh writable `diff/` as `upperdir`, producing `merged/` — the container's future `/`.
5. **CLI sends `POST /containers/<id>/start`.**
6. **Daemon calls containerd**, which calls **runc**. runc reads an OCI `config.json` (the runtime spec) describing namespaces, cgroups, mounts, and the command to run.
7. **runc creates the cgroup** at `/sys/fs/cgroup/system.slice/docker-<id>.scope/` and writes:
   - `memory.max` ← `536870912` (512 MiB in bytes)
   - `cpu.max` ← `150000 100000` (1.5 CPUs: 150000µs of runtime per 100000µs period)
8. **runc calls `clone()`** with namespace flags, creating a new process in fresh PID, NET, MNT, UTS, IPC, and (optionally) USER namespaces.
9. **Network is wired up.** For the default bridge, the daemon creates a **veth pair**: one end (`vethXXXX`) attached to the `docker0` bridge on the host, the other end moved into the container's net namespace and renamed `eth0` (full detail in Topic 13).
10. **Port publishing (`-p 8080:3000`)** inserts an iptables DNAT rule so host traffic to `:8080` is redirected to the container's `:3000` (full detail in Topic 13).
11. **runc mounts the overlay `merged/` as `/`**, plus the daemon-generated `/etc/hosts`, `/etc/resolv.conf`, `/etc/hostname` as bind mounts.
12. **runc sets the environment** — `NODE_ENV=production` is placed in the new process's `environ`.
13. **runc `execve()`s the image's ENTRYPOINT/CMD** (e.g. `node server.js`). PID 1 inside the container is now your Node process.
14. Because we used `-d`, the CLI **detaches** and returns the container ID. The Node process keeps running, its stdout piped to `<id>-json.log`.

## Exact syntax breakdown

The general shape:

```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
│          │         │      │         │
│          │         │      │         └─ args passed to the command
│          │         │      └─ overrides the image's CMD (Topic 07)
│          │         └─ which image to instantiate (name:tag or sha256:...)
│          └─ all the flags below (order among themselves doesn't matter)
└─ create + start in one shot
```

### `-d` / `--detach`

```
docker run -d orders-api:1.0
           │
           └─ run the container in the BACKGROUND, print the ID, return to your shell
```

- **Kernel implication:** none directly. `-d` only changes whether the CLI *attaches* to the container's stdio streams. The process runs identically either way. Without `-d`, the CLI attaches to the container's stdout/stderr and blocks your terminal. With `-d`, stdout/stderr still go to the json-file log; you just aren't watching them live.

### `-it` (`-i` + `-t`)

```
docker run -it node:20-alpine sh
           ││
           │└─ -t : allocate a pseudo-TTY (a fake terminal) inside the container
           └── -i : keep STDIN open even if not attached (interactive)
```

- **`-i` (`--interactive`):** keeps the container's STDIN connected to your keyboard. Without it, typing does nothing.
- **`-t` (`--tty`):** allocates a **pseudo-terminal** (`/dev/pts/N`). This is what gives you a shell prompt, line editing, colors, and Ctrl-C handling.
- **Kernel implication:** `-t` causes a PTY pair to be created; the container's shell sees a terminal device instead of a plain pipe. This is why `ls` prints in columns with `-t` but one-per-line without it (programs detect "am I on a TTY?").
- **Rule of thumb:** `-it` for interactive shells and REPLs. `-d` for services. Never combine `-d` and `-t` expecting to type into it — use `docker exec -it` instead.

### `--name`

```
docker run --name orders-api orders-api:1.0
           │      │
           │      └─ human-friendly name (must be unique on this host)
           └─ without it, Docker assigns a random name like "boring_tesla"
```

- **Kernel implication:** **none.** The name is pure Docker bookkeeping stored in `config.v2.json`. However it matters hugely because it becomes the container's **DNS hostname on custom networks** (Topic 13) — other containers reach it as `http://orders-api:3000`.

### `-p` / `--publish`

```
docker run -p 8080:3000 orders-api:1.0
           │  │    │
           │  │    └─ CONTAINER port (where your Node app listens: app.listen(3000))
           │  └─ HOST port (what the outside world connects to)
           └─ publish a port from container to host
```

Full form with host IP:

```
-p 127.0.0.1:8080:3000/tcp
   │         │    │    │
   │         │    │    └─ protocol (tcp default, or udp)
   │         │    └─ container port
   │         └─ host port
   └─ bind only to loopback (not reachable from other machines)
```

- **Kernel implication:** inserts an **iptables DNAT rule** in the `nat` table's `DOCKER` chain. Traffic hitting the host on `8080` gets its destination rewritten to `<container-ip>:3000`. A `docker-proxy` userspace helper may also listen on `8080` as a fallback. This is the single most important flag for a backend service — without it your API is unreachable from outside its Docker network.

### `-e` / `--env` and `--env-file`

```
docker run -e NODE_ENV=production -e "DATABASE_URL=postgres://db:5432/orders" orders-api:1.0
           │  │        │
           │  │        └─ value
           │  └─ variable name
           └─ set an environment variable inside the container
```

- **Kernel implication:** the string is placed in the new process's `environ` (visible at `/proc/<PID>/environ`). Node reads it via `process.env.NODE_ENV`.
- **`--env-file config.env`** reads `KEY=VALUE` lines from a file — cleaner for many vars. (Deep dive and secret-handling in Topic 15.)

### `-v` / `--volume` (and `--mount`)

```
docker run -v pgdata:/var/lib/postgresql/data postgres:16
           │  │      │
           │  │      └─ mount point INSIDE the container
           │  └─ named volume "pgdata" (Docker-managed, lives under /var/lib/docker/volumes/)
           └─ mount storage into the container
```

Bind-mount form (host path → container path):

```
-v /home/dev/orders-api:/app
   │                    │
   │                    └─ path inside container
   └─ absolute path on the HOST (must start with / to be a bind mount)
```

- **Kernel implication:** a `mount()` syscall with `MS_BIND` (for bind mounts) or an overlay/volume mount. The mount lands in the container's **mount namespace** so it only affects that container. This is how Postgres data survives container deletion. Full lifecycle and propagation detail in Topic 14.

### `--network`

```
docker run --network orders-net orders-api:1.0
           │        │
           │        └─ the network to join (bridge | host | none | <custom name>)
           └─ choose the network namespace / connectivity mode
```

- **`bridge`** (default): container gets a private IP on `docker0`, isolated, needs `-p` to be reachable.
- **`host`:** container **shares the host's network namespace** — no veth, no NAT, `app.listen(3000)` binds directly to the host's `:3000`. Fast, but zero network isolation.
- **`none`:** container gets only a loopback interface — fully network-isolated.
- **custom (`orders-net`):** a user-defined bridge with **automatic DNS**, so `orders-api` can reach `redis` and `db` by name.
- **Kernel implication:** determines which **network namespace** the container joins and whether a veth pair + iptables rules are created. Full detail in Topic 13.

### `--restart`

```
docker run --restart unless-stopped orders-api:1.0
           │        │
           │        └─ policy: no | on-failure[:max] | always | unless-stopped
           └─ what the daemon does when the container exits
```

- **`no`** (default): never restart.
- **`on-failure:5`:** restart only on non-zero exit, up to 5 times.
- **`always`:** always restart, even after a daemon/host reboot.
- **`unless-stopped`:** like `always`, but if *you* manually stopped it, don't bring it back after reboot.
- **Kernel implication:** none at the kernel level — this is the **daemon** watching the process's exit and re-invoking `start`. Stored in `hostconfig.json`. Critical for production so `orders-api` survives crashes and host reboots.

### `--memory` / `-m`

```
docker run --memory 512m orders-api:1.0
           │       │
           │       └─ hard limit (b, k, m, g suffixes). 512m = 536870912 bytes
           └─ maximum RAM this container may use
```

- **Kernel implication:** writes to the cgroup file `memory.max`. If the container's processes try to exceed it and can't reclaim memory, the kernel's **OOM killer** kills a process inside the container (usually PID 1 → container dies with exit code 137). This protects the host: a Node memory leak can't eat all RAM and starve Postgres/Redis. Pair with `--memory-swap` to also cap swap.

### `--cpus`

```
docker run --cpus 1.5 orders-api:1.0
           │     │
           │     └─ fractional CPU cores allowed
           └─ CPU quota
```

- **Kernel implication:** writes `cpu.max` in cgroup v2 as `quota period`. `--cpus 1.5` → `150000 100000`, meaning the container may consume 150ms of CPU time every 100ms window (i.e. 1.5 cores). Exceed it and the kernel **throttles** (pauses) the process until the next window — you'll see this as latency spikes, counted in `cpu.stat`'s `nr_throttled`. Prevents one container from starving the others.

### `--rm`

```
docker run --rm -it node:20-alpine sh
           │
           └─ automatically DELETE the container (and its anonymous volumes) when it exits
```

- **Kernel implication:** none at runtime. On exit, the daemon removes `/var/lib/docker/containers/<id>/` and the writable overlay layer. Perfect for throwaway commands and one-off scripts so you don't accumulate dead containers. **Never** use `--rm` on a stateful container whose logs or debug state you might need after a crash.

## Example 1 — basic

Run a throwaway Node REPL to test something, cleaned up automatically:

```bash
docker run \
  --rm \                # delete the container when I exit
  -it \                 # interactive + TTY so I get a real prompt
  --name scratch \      # nameable so I can exec into it from another terminal
  node:20-alpine \      # base image (Topic 04)
  node                  # override CMD: start the Node REPL instead of default
```

Inside the REPL:

```
> process.memoryUsage().rss
14followed by numbers...
> .exit          # exit → container stops → --rm deletes it. Nothing left behind.
```

Verify nothing was left over:

```bash
docker ps -a --filter name=scratch   # empty — --rm cleaned it up
```

## Example 2 — production scenario

**Scenario:** Your team runs `orders-api` on a single VM alongside Postgres and Redis. Last week a bug caused a memory leak; the API ballooned to 6 GB, the kernel OOM-killer started killing **Postgres** (the biggest target), and the whole app went down. Postmortem action item: *every container must have resource limits, restart policies, and persistent data*.

Here is the corrected production `docker run` for the API:

```bash
docker run \
  -d \                                   # background service, not interactive
  --name orders-api \                    # stable name → DNS + easy management
  --network orders-net \                 # custom net → reach "db" and "redis" by name
  -p 127.0.0.1:8080:3000 \               # publish only on loopback (nginx proxies in front)
  -e NODE_ENV=production \               # 12-factor config
  --env-file /etc/orders-api/prod.env \  # DB_URL, REDIS_URL, etc. (Topic 15)
  --memory 512m \                        # HARD cap: a leak kills only THIS container (exit 137)
  --memory-swap 512m \                   # no swap headroom → fail fast, don't thrash disk
  --cpus 1.0 \                           # at most 1 core → can't starve Postgres
  --restart unless-stopped \             # survive crashes + host reboot, respect manual stop
  --read-only \                          # rootfs read-only → smaller attack surface (Topic 23)
  --tmpfs /tmp \                         # but /tmp needs to be writable → tmpfs in RAM (Topic 14)
  orders-api:1.4.2                       # pinned tag, never :latest in prod (Topic 24)
```

Now when the leak recurs, the kernel OOM-killer kills only `orders-api` (which restarts automatically thanks to `--restart`), Postgres and Redis are untouched, and the incident is a 2-second blip instead of a full outage.

Supporting containers:

```bash
# Postgres with a NAMED VOLUME so data survives container replacement
docker run -d --name db --network orders-net \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/pg_pw \
  --memory 1g --cpus 1.0 --restart unless-stopped \
  postgres:16-alpine

# Redis with a memory cap matching its own maxmemory setting
docker run -d --name redis --network orders-net \
  --memory 256m --cpus 0.5 --restart unless-stopped \
  redis:7-alpine redis-server --maxmemory 200mb --maxmemory-policy allkeys-lru
```

## Common mistakes

### Mistake 1 — Forgetting `-p`, then wondering why curl fails

```bash
docker run -d --name orders-api orders-api:1.0
curl http://localhost:8080/health
```
```
curl: (7) Failed to connect to localhost port 8080: Connection refused
```
- **Root cause:** the container listens on `3000` inside its **own network namespace** on the `docker0` bridge (e.g. `172.17.0.2:3000`). No `-p` means no iptables DNAT rule maps the host's `8080` to it. The port simply isn't published.
- **Wrong:** `docker run -d --name orders-api orders-api:1.0`
- **Right:** `docker run -d --name orders-api -p 8080:3000 orders-api:1.0`

### Mistake 2 — Reversing the `-p` mapping

```bash
docker run -d -p 3000:8080 orders-api:1.0   # app actually listens on 3000!
curl http://localhost:3000/health
```
```
curl: (52) Empty reply from server
```
- **Root cause:** `-p HOST:CONTAINER`. Here Docker forwards host `3000` → container `8080`, but your Node app listens on `3000` inside. The DNAT points at a dead port.
- **Right:** `-p 3000:3000` (or `-p 8080:3000` if you want a different host port). Container port must match `app.listen(...)`.

### Mistake 3 — No `--memory`, one leak takes down everything

```bash
docker run -d --name orders-api orders-api:1.0   # no limit
# leak grows to 6GB, host has 4GB RAM...
dmesg | tail
```
```
Out of memory: Killed process 8123 (postgres) total-vm:...
```
- **Root cause:** with no `memory.max`, the container competes with the whole host for RAM. When RAM runs out, the kernel OOM-killer picks a victim by score — often a big innocent process like Postgres, not the guilty container.
- **Right:** always set `--memory`. The limit confines the OOM kill to the offending cgroup (exit 137), sparing the rest.

### Mistake 4 — Anonymous volume surprise with `--rm`

```bash
docker run --rm -v /var/lib/postgresql/data postgres:16   # no name before the colon!
```
- **Root cause:** `-v /path-inside-only` creates an **anonymous volume**. With `--rm`, that volume is **deleted on exit** — your database is gone. Anonymous volumes also silently pile up if you don't use `--rm`.
- **Wrong:** `-v /var/lib/postgresql/data` (anonymous)
- **Right:** `-v pgdata:/var/lib/postgresql/data` (named, survives). Never combine `--rm` with data you care about.

### Mistake 5 — `--network host` on macOS/Windows expecting it to work like Linux

```bash
docker run --network host -p 8080:3000 orders-api:1.0
```
```
WARNING: Published ports are discarded when using host network mode
```
- **Root cause:** `--network host` only truly shares the host stack on **Linux**. On Docker Desktop (Mac/Windows) the "host" is a Linux VM, not your laptop, so `host` networking and `-p` behave differently. Also, `-p` is meaningless with `host` mode — the container already binds host ports directly.
- **Right:** on Linux prod, use `host` only when you deliberately want zero isolation. Otherwise use bridge + `-p`.

## Hands-on proof

Run these right now to *see* each flag touch the kernel:

```bash
# 1. Start a limited container and grab its PID on the host
docker run -d --name proof --memory 256m --cpus 0.5 -p 8080:3000 nginx:alpine
PID=$(docker inspect -f '{{.State.Pid}}' proof)
echo "Host PID = $PID"

# 2. See the cgroup limit --memory actually wrote (cgroup v2)
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' proof).scope/memory.max
#   → 268435456   (256 MiB in bytes — proof --memory hit the kernel)

# 3. See the CPU quota --cpus wrote
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' proof).scope/cpu.max
#   → 50000 100000   (0.5 core)

# 4. See the container's private namespaces vs the host's
ls -l /proc/$PID/ns/net /proc/1/ns/net
#   → different inode numbers = different network namespaces = real isolation

# 5. See the iptables DNAT rule that -p created
sudo iptables -t nat -L DOCKER -n | grep 8080
#   → DNAT ... tcp dpt:8080 to:172.17.0.2:3000

# 6. See the env vars in the process
sudo cat /proc/$PID/environ | tr '\0' '\n'

# 7. Clean up
docker rm -f proof
```

Every flag you passed left a fingerprint you can read with `cat` and `ls`. That is the entire point: containers are ordinary Linux, configured by flags.

## Practice exercises

### Exercise 1 — easy
Run `nginx:alpine` detached, named `web`, publishing container port 80 on host port 8888. Then `curl http://localhost:8888` and confirm you get the nginx welcome page. Remove it with `docker rm -f web`.

### Exercise 2 — medium
Run `node:20-alpine` with `--memory 50m` and inside it run a Node one-liner that allocates a huge array until it crashes:
```bash
docker run --rm --memory 50m node:20-alpine \
  node -e 'const a=[]; while(true){ a.push(Buffer.alloc(1024*1024)); }'
```
Observe the exit code (`echo $?` → 137). Explain in one sentence *which kernel subsystem* killed it and *why the exit code is 137*.

### Exercise 3 — hard (production simulation)
Recreate the Example 2 setup on your machine: create a custom network `orders-net`, run Postgres (named volume `pgdata`, `--memory 512m`), run Redis (`--memory 128m`), and run a small container that connects to `db` by name. Then:
1. Kill the Postgres container (`docker rm -f db`) and recreate it with the same `-v pgdata:...`. Prove the data survived.
2. Set Postgres's `--restart unless-stopped`, `docker kill db`, and prove the daemon brings it back.
3. Inspect `cpu.stat`'s `nr_throttled` after hammering the API with load and explain what throttling you see.

## Mental model checkpoint

Answer from memory:

1. `docker run` is really which two lower-level commands combined?
2. Which flags touch the kernel deeply, and which are pure Docker bookkeeping?
3. In `-p 8080:3000`, which number is the host and which is the container? What kernel table gets modified?
4. What exact cgroup file does `--memory 512m` write, and what value?
5. What does the kernel do when a container exceeds its `--memory` limit? What exit code results?
6. What is the difference in kernel effect between `--network host` and the default `bridge`?
7. Why is combining `--rm` with an important named volume dangerous — and when is it actually safe?

## Quick reference card

| Flag | What it does | Key kernel/behavior detail |
|------|--------------|----------------------------|
| `-d` | Run in background | No kernel effect; only detaches CLI from stdio |
| `-it` | Interactive + TTY | Allocates a pseudo-terminal (`/dev/pts/N`) |
| `--name` | Human name | Pure bookkeeping; becomes DNS name on custom nets |
| `-p H:C` | Publish port | Inserts iptables DNAT rule (host H → container C) |
| `-e K=V` | Set env var | Writes to process `environ` (`/proc/PID/environ`) |
| `--env-file` | Bulk env vars | Reads `KEY=VALUE` lines from a file |
| `-v src:dst` | Mount storage | `mount()` syscall into the mount namespace |
| `--network` | Choose connectivity | Selects net namespace; may create veth + NAT |
| `--restart` | Auto-restart policy | Daemon-level watch, not kernel; in `hostconfig.json` |
| `--memory` / `-m` | RAM hard cap | Writes cgroup `memory.max`; breach → OOM kill (137) |
| `--cpus` | CPU quota | Writes cgroup `cpu.max` (quota period); breach → throttle |
| `--rm` | Delete on exit | Removes container dir + writable layer on exit |
| `--read-only` | Read-only rootfs | Overlay mounted RO; pair with `--tmpfs` for writable paths |

## When would I use this at work?

1. **Local dev of `orders-api`:** `docker run --rm -it -p 3000:3000 -v $(pwd):/app orders-api:dev` to run the API with your source bind-mounted for hot reload and auto-cleanup when you Ctrl-C.
2. **Debugging a flaky prod container:** run a one-off `docker run --rm -it --network orders-net node:20-alpine sh` on the same network to `curl db:5432` / `redis:6379` and reproduce connectivity issues from *inside* the network.
3. **Reproducing an OOM incident:** intentionally run with a tight `--memory` to confirm your app's memory ceiling and validate that limits confine the OOM kill to the right cgroup before you set them in production or Kubernetes (Topics 25, 44).

## Connected topics

- **Study before:** Topic 04 (Images and Layers) — you run *from* an image; Topic 05 (Your First Container) — the basic `run` trace; Topic 07 (CMD vs ENTRYPOINT) — what `[COMMAND]` overrides.
- **Study after:** Topic 13 (Networking in Docker) — the full story behind `-p` and `--network`; Topic 14 (Volumes in Depth) — the full story behind `-v`; Topic 15 (Environment Variables and Secrets) — `-e`/`--env-file` done safely; Topic 25 (Resource Limits) — `--memory`/`--cpus` and cgroups in production depth.
