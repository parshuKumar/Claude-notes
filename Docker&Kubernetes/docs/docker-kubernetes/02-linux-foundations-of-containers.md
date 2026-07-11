# 02 — Linux Foundations of Containers
## Section: Docker Foundations

---

## ELI5 — The Simple Analogy

Go back to the apartment building from Topic 01. What actually makes an apartment an apartment? Three things:

1. **Walls and a locked door** — you can only *see and reach* your own rooms. You can't wander into the neighbor's bedroom. That's **namespaces**: they control *what a process can see*.

2. **A utility meter** — the building says "apartment 4B gets at most 20 amps and 200 litres of water an hour." Use more and the breaker trips. That's **cgroups**: they control *how much a process can use*.

3. **The furniture and layout** — your apartment came with a base layout (built by the landlord, shared as a template across all units), and *on top of that* you added your own furniture. If you move a chair, only your apartment changes; the shared template is untouched. That's the **overlay filesystem**: a shared read-only base plus your own thin writable layer on top.

Walls (namespaces) + meter (cgroups) + layered furniture (overlayfs). Master these three and you understand *exactly* what Docker is doing. Everything else is convenience on top.

---

## The Linux kernel feature underneath

This whole topic **is** the kernel feature — three of them, the "kernel trinity":

| Trinity member | Kernel subsystem | Question it answers | You configure it via |
|---|---|---|---|
| **Namespaces** | `CLONE_NEW*` flags to `clone()`/`unshare()` | "What can this process **see**?" | `/proc/<PID>/ns/` |
| **Cgroups** | cgroup v2 controllers | "How much can this process **use**?" | `/sys/fs/cgroup/` |
| **Overlay FS** | `overlay` union mount | "What is this process's **root `/`**?" | `/var/lib/docker/overlay2/` |

Every container runtime — Docker, containerd, Podman, CRI-O, Kubernetes — is ultimately just a program that arranges these three kernel features around your process. There is no secret fourth ingredient. Learn these and you can read any container's behavior straight from the kernel.

---

## What is this?

The Linux kernel provides three independent features — **namespaces** (isolation of visibility), **cgroups** (limits on resource usage), and **union filesystems** like **overlayfs** (a layered root filesystem) — that together let one kernel run many mutually-invisible, resource-capped processes each with its own filesystem. A container is the combination of all three applied to a process.

These features were **not** built for Docker. Namespaces landed in the kernel between 2002 and 2013, cgroups in 2007, overlayfs in 2014. Docker (2013) was the tool that stitched them into an easy package.

---

## Why does it matter for a backend developer?

Every strange container behavior you'll ever debug traces back to one of these three:

- `orders-api` **can't reach Postgres** → the containers are in different **network namespaces** with no route between them (Topic 13).
- `orders-api` **got OOMKilled** → it hit its **cgroup** `memory.max` (Topic 25).
- A file you wrote **vanished after restart** → it lived in the **overlay** writable layer, which was thrown away (Topic 14).
- `ps aux` inside the container **shows only your app** → the **PID namespace** hides everything else.
- Two containers **both bind port 3000** with no conflict → each has its own **network namespace** with its own port space.

If you only learn "Docker commands," these will feel like magic and you'll cargo-cult fixes off Stack Overflow. If you learn the trinity, you'll *reason* about them from first principles.

---

## The physical reality

Everything below is a real file or directory you can read on a Linux host with a running container.

**Namespaces — symlinks under `/proc/<PID>/ns/`:**

```
$ sudo ls -l /proc/84213/ns/
lrwxrwxrwx pid  -> 'pid:[4026532568]'    ← 6 isolatable namespaces here (plus cgroup, time)
lrwxrwxrwx net  -> 'net:[4026532570]'
lrwxrwxrwx mnt  -> 'mnt:[4026532565]'
lrwxrwxrwx uts  -> 'uts:[4026532566]'
lrwxrwxrwx ipc  -> 'ipc:[4026532567]'
lrwxrwxrwx user -> 'user:[4026531837]'
```

Each `foo:[NUMBER]` is a namespace instance. Two processes with the **same** number share that namespace; **different** numbers mean isolation.

**Cgroups — directories under `/sys/fs/cgroup/` (cgroup v2):**

```
$ ls /sys/fs/cgroup/system.slice/docker-<id>.scope/
cpu.max          ← CPU quota, e.g. "50000 100000" = 0.5 CPU
cpu.stat         ← how much CPU used, how many times throttled
memory.max       ← hard memory limit in bytes
memory.current   ← current memory usage in bytes
memory.events    ← counters: oom, oom_kill, max hits
io.max           ← disk I/O limits per block device
cgroup.procs     ← PIDs that belong to this cgroup
```

**Overlay filesystem — directories under `/var/lib/docker/overlay2/`:**

```
/var/lib/docker/overlay2/
├── <layerA>/diff/     ← lowerdir: read-only image layer (e.g. alpine base)
├── <layerB>/diff/     ← lowerdir: read-only image layer (e.g. node install)
├── <container>/diff/  ← upperdir: the container's WRITABLE layer
├── <container>/work/  ← overlayfs internal bookkeeping
└── <container>/merged/← the combined view = the container's actual /
```

And the live mount that ties them together:

```
$ mount | grep overlay
overlay on /var/lib/docker/overlay2/<container>/merged type overlay
  (rw,lowerdir=.../layerB/diff:.../layerA/diff,upperdir=.../container/diff,workdir=.../container/work)
```

---

## How it works — step by step

### Part A — Namespaces (isolation of *what you see*)

There are **6 namespace types you'll care about** (Linux has more, like `cgroup` and `time`, but these six are the container core):

1. **`pid` — Process ID namespace.** The container's first process becomes **PID 1**. It can only see itself and its descendants. On the host it has a *different, real* PID. This is why `ps aux` inside `orders-api` shows 2 processes, not the host's 300. Created with `CLONE_NEWPID`.

2. **`net` — Network namespace.** A fresh, private network stack: its own interfaces, own routing table, own iptables, own port space. A new net namespace starts with **only a loopback** (`lo`), down. Docker then creates a `veth` pair to connect it to the `docker0` bridge. This is why two containers can both listen on port 3000. Created with `CLONE_NEWNET`.

3. **`mnt` — Mount namespace.** Its own list of filesystem mounts. Combined with `pivot_root`, this is what gives the container its own `/`. Mounting something inside the container doesn't appear on the host. Created with `CLONE_NEWNS` (the oldest namespace, 2002 — the flag name predates the naming convention).

4. **`uts` — UTS namespace.** ("UNIX Time-sharing System" — a historical name.) Isolates the **hostname** and **domain name**. This is why `orders-api` can call itself `orders-api` while the host is `prod-node-7`. Created with `CLONE_NEWUTS`.

5. **`ipc` — IPC namespace.** Isolates System V IPC and POSIX message queues and **shared memory** segments. Two containers can't accidentally read each other's shared memory. (Relevant: Postgres uses shared memory heavily; each Postgres container gets its own.) Created with `CLONE_NEWIPC`.

6. **`user` — User namespace.** Maps user IDs inside the container to *different* user IDs on the host. This lets a process be **root (UID 0) inside** the container while being an **unprivileged UID (e.g. 100000) on the host** — a huge security win, because container-root can't act as host-root. Created with `CLONE_NEWUSER`. (Docker doesn't enable this by default; you turn it on — Topic 23.)

**The mechanism:** `runc` calls `clone()` (or `unshare()`) with the OR of the flags it wants:

```
clone(child_fn, stack, CLONE_NEWPID|CLONE_NEWNET|CLONE_NEWNS|CLONE_NEWUTS|CLONE_NEWIPC|SIGCHLD, arg)
```

The instant this returns, the child lives in six brand-new namespaces. A namespace stays alive as long as at least one process is in it (or something holds its `/proc/.../ns/` file open); when the last process exits, the kernel destroys it.

### Part B — Cgroups (limits on *how much you use*)

Cgroups (control groups) are a tree of directories under `/sys/fs/cgroup/`. You put a process into a group by writing its PID into `cgroup.procs`, then the kernel enforces that group's limits on it. The three controllers you'll use constantly:

7. **`cpu` controller.** `cpu.max` holds `"QUOTA PERIOD"` in microseconds. `"50000 100000"` means "50 ms of CPU per 100 ms window" = **half a core**. Exceed it and the kernel **throttles** the process (pauses it until the next window) — it does *not* kill it. `cpu.stat`'s `nr_throttled` and `throttled_usec` tell you how badly you're being throttled. This is why `orders-api` can go slow under load without crashing.

8. **`memory` controller.** `memory.max` is a hard byte limit. Every page the process touches is accounted in `memory.current`. When `memory.current` would exceed `memory.max`, the kernel's **OOM killer** fires *inside that cgroup* and kills a process (usually the biggest — often your app). `memory.events` increments `oom_kill`. Docker reports this as **`OOMKilled: true`**. Crucially: this is *local* to the container — the host can have gigabytes free and your container still dies at its own limit.

9. **`io` controller.** `io.max` limits read/write bytes-per-second and IOPS *per block device*. Stops one container's noisy disk logging from starving Postgres's disk. Format: `"MAJOR:MINOR rbps=... wbps=... riops=... wiops=..."`.

**cgroup v1 vs v2:** Modern systems use **cgroup v2**, a single unified hierarchy (one tree, all controllers). Older systems used v1 (a separate tree per controller: `/sys/fs/cgroup/memory/`, `/sys/fs/cgroup/cpu/`, ...). Check with `stat -fc %T /sys/fs/cgroup/` → `cgroup2fs` means v2. This doc shows v2 paths.

### Part C — Overlay filesystem (your own *root `/`*)

An image (Topic 04) is a stack of **read-only layers**. But a running process needs to *write* (temp files, logs, `npm` caches). Overlayfs solves this by **union-mounting** layers:

10. **lowerdir(s)** — the read-only image layers, stacked. Upper layers shadow lower ones. If `layerB` and `layerA` both have `/etc/hostname`, the container sees `layerB`'s.

11. **upperdir** — a single **writable** layer, unique per container. All writes land here.

12. **workdir** — scratch space overlayfs needs to perform atomic operations. You never touch it.

13. **merged** — the unified view. This becomes the container's `/` (via `pivot_root`). Reads fall through the stack top-to-bottom until a file is found; writes go to `upperdir`.

**Copy-on-write (COW):** when the container *modifies* a file that exists only in a read-only lower layer, overlayfs first **copies it up** into `upperdir`, then edits the copy. The original image layer is never touched. This is why 50 `orders-api` containers can share **one** copy of the 300 MB `node:20-alpine` image on disk — they only diverge in their tiny upper layers. Deleting a lower-layer file writes a special **whiteout** marker in upperdir so the file "disappears" from the merged view without touching the real layer.

---

## Exact syntax breakdown

Manually creating a namespace (no Docker) with `unshare`:

```
sudo unshare --pid --net --mount --uts --ipc --fork --mount-proc bash
│    │        │     │     │       │     │     │      │            │
│    │        │     │     │       │     │     │      │            └─ program to run in the new namespaces
│    │        │     │     │       │     │     │      └─ remount /proc so `ps` reflects the new PID namespace
│    │        │     │     │       │     │     └─ fork a child (needed for PID ns to take effect)
│    │        │     │     │       │     └─ --ipc: new IPC namespace
│    │        │     │     │       └─ --uts: new UTS (hostname) namespace
│    │        │     │     └─ --mount: new mount namespace
│    │        │     └─ --net: new (empty) network namespace
│    │        └─ --pid: new PID namespace (this bash becomes PID 1)
│    └─ unshare: detach the current process into NEW namespaces (a lighter cousin of clone)
└─ sudo: creating namespaces (except user ns) needs privilege
```

Reading a container's cgroup memory limit:

```
cat /sys/fs/cgroup/system.slice/docker-<ID>.scope/memory.max
│   │            │            │                 │
│   │            │            │                 └─ memory.max: the hard limit in bytes ("max" = unlimited)
│   │            │            └─ per-container scope dir; <ID> is the full 64-char container ID
│   │            └─ system.slice: systemd's slice where docker places container cgroups
│   └─ /sys/fs/cgroup: the cgroup v2 unified hierarchy mount point
└─ cat: just read the file — cgroup config IS files
```

Inspecting the overlay mount:

```
mount -t overlay
│     │  │
│     │  └─ overlay: only show filesystems of type "overlay"
│     └─ -t: filter by filesystem type
└─ mount: list active mounts; each container's merged/ shows lowerdir/upperdir/workdir
```

---

## Example 1 — basic

Build a container **by hand**, using only kernel tools, to prove Docker isn't magic. (Linux host, run as root.)

```bash
# --- NAMESPACES: create an isolated world ---
sudo unshare --pid --uts --mount --fork --mount-proc bash
#     └─ we are now in new pid/uts/mount namespaces, running bash as PID 1

# Inside the new namespace:
hostname isolated-box        # change hostname — only affects THIS uts namespace
hostname                     # -> isolated-box
ps aux                       # -> only bash + ps. The host's processes are INVISIBLE (pid ns)
echo $$                      # -> 1   ... this bash is PID 1 in its pid namespace
exit                         # leave; the namespaces are destroyed when the last process exits
```

```bash
# --- CGROUPS: cap memory by hand ---
sudo mkdir /sys/fs/cgroup/demo                 # create a cgroup (just a dir!)
echo 50000000 | sudo tee /sys/fs/cgroup/demo/memory.max   # cap at ~50 MB
echo $$ | sudo tee /sys/fs/cgroup/demo/cgroup.procs        # put THIS shell into it
# now any child of this shell is capped at 50 MB; allocate more -> kernel OOM-kills it
```

```bash
# --- OVERLAY FS: stack a read-only base + a writable layer by hand ---
mkdir -p /tmp/overlay/{base,upper,work,merged}
echo "from base (read-only)" > /tmp/overlay/base/hello.txt
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/overlay/base,upperdir=/tmp/overlay/upper,workdir=/tmp/overlay/work \
  /tmp/overlay/merged
cat /tmp/overlay/merged/hello.txt      # -> "from base (read-only)"  (read fell through to lowerdir)
echo "new file" > /tmp/overlay/merged/new.txt   # write goes to UPPER, not base
ls /tmp/overlay/base                   # -> hello.txt  (base UNTOUCHED)
ls /tmp/overlay/upper                  # -> new.txt    (write landed here)  <- copy-on-write proof
```

You just built the three pillars of a container with nothing but `unshare`, a cgroup directory, and `mount -t overlay`. Docker automates exactly this.

---

## Example 2 — production scenario

**The situation.** It's 2 a.m. Your pager fires: `orders-api` in the `orders` environment is flapping — restarting every 40 seconds. Redis and Postgres are fine. You need to know *why*, using the kernel trinity as your diagnostic map.

**Step 1 — Is it a cgroup (memory) problem?** Get the container's host PID and read its memory cgroup:

```bash
PID=$(docker inspect --format '{{.State.Pid}}' orders-api)
SCOPE=$(cat /proc/$PID/cgroup | awk -F: '{print $NF}')   # e.g. /system.slice/docker-<id>.scope
cat /sys/fs/cgroup${SCOPE}/memory.max        # -> 536870912  (512 MiB limit)
cat /sys/fs/cgroup${SCOPE}/memory.events     # -> oom_kill 6   <-- SMOKING GUN
```

`oom_kill 6` means the kernel OOM killer fired **6 times** inside this cgroup. The container isn't buggy — it's **memory-starved**. A recent feature added an in-memory cache that pushed steady-state RAM past 512 MiB. Every time it crosses `memory.max`, the kernel kills it, Docker restarts it, it climbs again, dies again — the 40-second flap.

**Step 2 — Confirm CPU isn't the cause.** Rule out throttling:

```bash
cat /sys/fs/cgroup${SCOPE}/cpu.stat
# nr_throttled 3         <- minor throttling, not the killer
# throttled_usec 210000  <- 0.21s total, negligible
```

Low throttling → CPU is not the problem. Good, memory is confirmed as the cause.

**Step 3 — Confirm isolation is intact (not a shared-resource bug).** Check `orders-api` is in its own net namespace so this isn't a port/network clash:

```bash
sudo ls -l /proc/$PID/ns/net           # note the net:[NUMBER]
sudo ls -l /proc/1/ns/net              # host's net namespace — DIFFERENT number => isolated, fine
```

**The fix (informed by the trinity):** either raise the limit (`--memory 768m`) if the new cache is legitimate, or fix the leak and cap the cache. You *proved* it was cgroup memory, not a code deadlock, not CPU throttling, not a network collision — by reading three kernel files. That's the payoff of understanding the foundations: you diagnose from evidence, not guesswork.

---

## Common mistakes

**Mistake 1 — Setting `--memory` too low and blaming the app.**

```
$ docker inspect orders-api --format '{{.State.OOMKilled}}'
true
```

Root cause: `memory.max` in the cgroup is below the app's real working set; the kernel OOM killer inside the cgroup killed PID 1. The host having free RAM is irrelevant — cgroup limits are *local*. Right: size `--memory` to observed `memory.current` under load, plus headroom.

**Mistake 2 — Thinking `--cpus` kills the container. It throttles.**

People expect CPU overuse to crash the container like memory does. It doesn't:

```
$ cat cpu.stat
nr_throttled 44012      ← app is being paused constantly, appears "slow/hung"
```

Root cause: the CPU controller *throttles* (pauses) rather than kills. Symptom is latency, not crashes. Right: if the app feels hung under load, check `nr_throttled` before assuming a deadlock.

**Mistake 3 — Expecting overlay upper-layer writes to persist.**

```bash
docker run --name t alpine sh -c 'echo hi > /data.txt'
docker rm t                         # upperdir deleted
# /data.txt is gone forever
```

Root cause: writes land in the per-container `upperdir` under `/var/lib/docker/overlay2/<id>/diff`, which is destroyed with the container. Right: mount a **volume** for anything that must survive (Topic 14).

**Mistake 4 — Assuming container-root == host-root is fine (no user namespace).**

By default Docker does **not** enable the `user` namespace, so UID 0 in the container is UID 0 on the host kernel. A container escape = host root. Root cause: shared kernel + no UID remapping. Right: enable user namespaces / rootless Docker, or at minimum run as non-root `USER` (Topic 23).

**Mistake 5 — Confusing "shares kernel" with "shares everything."**

New folks think containers share the host's `/etc`, processes, or network. They don't — those are namespaced. But they *do* share the kernel, `/proc/sys` tunables partially, and the clock. Root cause: incomplete mental model of *which* things are namespaced. Right: memorize the 6 namespaces — anything on that list is isolated; anything not (the kernel itself, kernel modules, the CPU/hardware) is shared.

---

## Hands-on proof

```bash
# Start a container to inspect.
docker run -d --name trinity --memory 256m --cpus 0.5 alpine sleep 900
PID=$(docker inspect --format '{{.State.Pid}}' trinity)

# --- NAMESPACES: prove isolation ---
echo "container ns:"; sudo ls -l /proc/$PID/ns/ | awk '{print $9, $11}'
echo "your ns:";      ls -l /proc/$$/ns/     | awk '{print $9, $11}'
# Compare the numbers: net/mnt/pid/uts/ipc DIFFER => isolated

# --- CGROUPS: prove the limits are real kernel files ---
SCOPE=$(awk -F: '{print $NF}' /proc/$PID/cgroup)
echo "memory.max: $(cat /sys/fs/cgroup${SCOPE}/memory.max)"   # ~268435456 (256 MiB)
echo "cpu.max:    $(cat /sys/fs/cgroup${SCOPE}/cpu.max)"      # ~50000 100000 (0.5 core)

# --- OVERLAY: prove the layered filesystem ---
docker inspect --format '{{ .GraphDriver.Data.MergedDir }}' trinity   # the merged/ = container's /
mount -t overlay | grep "$(docker inspect --format '{{.Id}}' trinity | cut -c1-12)" | head -1

# Clean up.
docker rm -f trinity
```

You verified: namespaces exist as `/proc/<PID>/ns/` links that differ from yours, cgroup limits are readable files matching your `--memory`/`--cpus`, and the container's `/` is an overlay mount with lower/upper/work dirs.

---

## Practice exercises

### Exercise 1 — easy
Using **only** `unshare` (no Docker), create a new UTS + PID namespace running `bash`, change the hostname inside to `lab`, and prove with `ps aux` that it can't see the host's processes. Then `exit` and confirm the host's hostname is unchanged.

### Exercise 2 — medium
Start `docker run -d --name m --memory 100m alpine sleep 600`. Find its cgroup scope via `/proc/<PID>/cgroup`. Write a one-liner that reads `memory.current` every second for 10 seconds. Then run a memory allocator inside it that grabs 150 MB and watch `memory.events`' `oom_kill` counter increment. Explain why the host having free RAM didn't save it.

### Exercise 3 — hard (production simulation)
Reproduce the Example 2 flap. Run `orders-api` (use any Node image or `alpine` with a memory-hog loop) with `--memory 128m` and `--restart on-failure`. Trigger repeated OOM kills. Using **only** cgroup files (`memory.events`, `cpu.stat`) and namespace files (`/proc/<PID>/ns/`), write a 5-line diagnosis proving (a) it's memory not CPU, and (b) isolation is intact. Then fix it by choosing an appropriate `--memory` value based on measured `memory.current`, and prove the flapping stops.

---

## Mental model checkpoint

1. Name all 6 namespace types and the one question each answers.
2. What is the fundamental difference between what *namespaces* do and what *cgroups* do?
3. Where do you read a container's memory limit, and what happens the moment usage would exceed it?
4. Does CPU overuse kill or throttle a container? Where do you see the evidence?
5. In overlayfs, name lowerdir, upperdir, workdir, merged — which is writable, and which becomes `/`?
6. Explain copy-on-write and why 50 containers can share one 300 MB base image on disk.
7. Why is running without a `user` namespace a security risk?

---

## Quick reference card

| Feature / path | What it does | Key detail |
|---|---|---|
| `pid` namespace | Isolates process IDs | Container's first proc = PID 1 |
| `net` namespace | Isolates interfaces, ports, routes, iptables | Starts with only `lo` |
| `mnt` namespace | Isolates mounts | Enables a private `/` via `pivot_root` |
| `uts` namespace | Isolates hostname | Lets container name itself |
| `ipc` namespace | Isolates shared memory / IPC | Postgres shm is per-container |
| `user` namespace | Maps container UIDs to host UIDs | Container-root ≠ host-root (security) |
| `/sys/fs/cgroup/.../memory.max` | Hard memory cap (bytes) | Exceed → OOM kill inside cgroup |
| `/sys/fs/cgroup/.../cpu.max` | `QUOTA PERIOD` µs | Exceed → throttle, not kill |
| `/sys/fs/cgroup/.../io.max` | Disk bandwidth/IOPS cap | Per block device |
| `overlay2` lowerdir | Read-only image layers | Shared across containers |
| `overlay2` upperdir | Per-container writable layer | Destroyed with container |
| `overlay2` merged | Unified view = container's `/` | Read falls through; write copies up |

---

## When would I use this at work?

1. **On-call triage.** A container is flapping — you go straight to `memory.events` and `cpu.stat` in its cgroup to decide "OOM vs throttle vs app bug" in under a minute, instead of guessing.

2. **Right-sizing and cost.** Knowing overlay copy-on-write shares base-image bytes across containers, you pack many `node:20-alpine`-based services on one host without duplicating hundreds of MB each, and you set `--memory`/`--cpus` from measured cgroup counters instead of superstition.

3. **Security reviews.** When a security engineer asks "can a compromised `orders-api` reach the host?", you can speak precisely about which namespaces isolate it, and that without a user namespace, container-root is host-root — and recommend rootless mode.

---

## Connected topics

- **Study before:**
  - **Topic 01 — What Is a Container**: introduced that a container is a process + these three features. This topic is the deep dive.
- **Study after:**
  - **Topic 03 — Docker Architecture**: which component (dockerd → containerd → runc) actually calls `clone()` and writes the cgroup files described here.
  - **Topic 04 — Images and Layers**: the read-only lowerdirs come from image layers; this explains where they're built.
  - **Topic 13 — Networking in Docker**: the `net` namespace, `veth` pairs, and iptables in full.
  - **Topic 25 — Resource Limits**: the complete cgroup story — `OOMKilled`, CPU throttling, QoS.
