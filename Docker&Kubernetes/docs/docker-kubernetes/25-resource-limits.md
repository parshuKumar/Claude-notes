# 25 — Resource Limits

## Section: Production Patterns

---

## ELI5 — The Simple Analogy

Imagine an apartment building where every tenant shares one water tank and one electricity meter. Everyone gets along until one tenant leaves every tap running and every appliance on — now the whole building has no water and the power trips. One greedy tenant ruins it for everybody.

The landlord fixes this with two things:

- **A water meter with a cap.** "Apartment 4, you get at most 200 litres an hour. Try to use more and the tap just runs slower." That's a **CPU limit** — you don't get killed for wanting more CPU, you're just *throttled* (slowed down).
- **A hard rule on how much stuff you can store.** "Your apartment holds 512 boxes. Box number 513 does not fit — and if you keep forcing boxes in, we evict you." That's a **memory limit** — cross it and you get **evicted**: the kernel kills your process (OOMKilled).

The building is your Docker host. The tenants are containers. Resource limits are how the landlord (the Linux kernel, via cgroups) stops one noisy `orders-api` from starving Postgres and Redis on the same box.

The key asymmetry to burn into memory: **too much CPU → you get slowed. Too much memory → you get killed.** CPU is squishy; memory is a hard wall.

---

## The Linux kernel feature underneath

This entire topic is **cgroups** (control groups), the kernel subsystem from Topic 02. A cgroup is a set of processes the kernel accounts and limits *together*. Docker creates one cgroup per container and writes your limits into special files the kernel reads.

Almost every modern distro uses **cgroup v2**, a single unified hierarchy mounted at `/sys/fs/cgroup`. Each container gets a directory there full of control files. You literally read and write plain text to tune the kernel.

**Memory control (the memory controller):**

```
/sys/fs/cgroup/.../memory.max        ← the hard limit in bytes. Cross it → OOM.
/sys/fs/cgroup/.../memory.current    ← how many bytes the cgroup is using RIGHT NOW
/sys/fs/cgroup/.../memory.events      ← counters: how many times "oom" / "oom_kill" fired
/sys/fs/cgroup/.../memory.high        ← soft limit: reclaim hard + throttle allocations
```

When a process in the cgroup tries to allocate memory that would push `memory.current` above `memory.max`, the kernel first tries to reclaim (drop caches, etc.). If it can't, it invokes the **OOM killer** — `oom_kill_process()` in `mm/oom_kill.c` — which sends `SIGKILL` (signal 9) to a process in that cgroup. The container's PID 1 dying with SIGKILL is why you see **exit code 137** (`128 + 9`).

**CPU control (the cpu controller), via the CFS bandwidth mechanism:**

```
/sys/fs/cgroup/.../cpu.max     ← "QUOTA PERIOD" e.g. "50000 100000"
                                  = 50000µs of CPU time every 100000µs window = 0.5 CPU
/sys/fs/cgroup/.../cpu.stat    ← nr_throttled, throttled_usec — proof of throttling
/sys/fs/cgroup/.../cpu.weight  ← relative share when CPU is contended (from --cpu-shares)
```

The CFS (Completely Fair Scheduler) **bandwidth controller** enforces `cpu.max`: within each 100 ms period, once the cgroup has burned its quota, every runnable task in it is **taken off the CPU until the next period**. Nobody is killed — they just wait. That waiting *is* CPU throttling.

So: `--memory` writes `memory.max`, `--cpus` writes `cpu.max`. Everything else in this topic is watching what the kernel does with those two numbers.

---

## What is this?

Resource limits cap how much memory and CPU a container may use, enforced by the Linux kernel's cgroup controllers. `--memory` sets a hard memory ceiling; exceed it and the kernel's OOM killer terminates the process (OOMKilled, exit 137). `--cpus` sets a CPU-time quota; exceed it and the CFS scheduler throttles the container (slows it, never kills it). Reservations (`--memory-reservation`, `--cpu-shares`) are softer hints used when the host is under contention.

---

## Why does it matter for a backend developer?

Because your `orders-api`, Postgres, and Redis usually share a host, and Node.js is *especially* dangerous here. Two concrete failure modes:

1. **The noisy neighbor.** A memory leak in `orders-api` (a growing cache, an unbounded array of pending jobs) slowly eats all host RAM. With no limits, the host's *global* OOM killer wakes up and may kill **Postgres** — the innocent bystander — because it happened to be the biggest process. Your API's leak just corrupted your database's availability. With a per-container `--memory` limit, only `orders-api` dies, Postgres is untouched, and it restarts cleanly.

2. **The invisible slowdown.** You set `--cpus 0.5` and your API's p99 latency mysteriously triples under load. There's no crash, no error log — the kernel is *throttling* you every 100 ms and you have no idea unless you know to look at `cpu.stat`. Understanding throttling is the difference between "randomly slow" and "oh, we're CPU-limited, bump the quota."

There's also a Node-specific trap: the V8 heap and the libuv thread pool don't automatically read the cgroup limit, so Node can happily try to use more than `--memory` allows and get OOMKilled at a size *smaller* than you'd expect. You have to tell Node about the limit.

---

## The physical reality

For a container run with `--memory 512m --cpus 0.5`, here's what exists in the kernel, viewable on the host.

**Find the cgroup path:**

```bash
CID=$(docker inspect -f '{{.Id}}' orders-api)
ls /sys/fs/cgroup/system.slice/docker-$CID.scope/
#   memory.max  memory.current  memory.events  cpu.max  cpu.stat  ...
```

**The memory ceiling and current usage:**

```
/sys/fs/cgroup/system.slice/docker-<CID>.scope/memory.max
    536870912                    ← 512 * 1024 * 1024 = exactly your --memory in bytes

/sys/fs/cgroup/system.slice/docker-<CID>.scope/memory.current
    319012864                    ← ~304 MB used right now

/sys/fs/cgroup/system.slice/docker-<CID>.scope/memory.events
    low 0
    high 0
    max 42                       ← hit the ceiling 42 times (reclaim attempts)
    oom 3
    oom_kill 3                   ← the OOM killer fired 3 times ← your crashes
```

**The CPU quota and throttling evidence:**

```
/sys/fs/cgroup/system.slice/docker-<CID>.scope/cpu.max
    50000 100000                 ← 50 ms quota per 100 ms period = 0.5 CPU

/sys/fs/cgroup/system.slice/docker-<CID>.scope/cpu.stat
    usage_usec 1820000
    nr_periods 4200
    nr_throttled 1170            ← throttled in 1170 of 4200 periods (~28%!)
    throttled_usec 3450000       ← 3.45 seconds of forced waiting — THIS is your latency
```

**The kernel's record of the kill** (host dmesg):

```
$ dmesg | tail
[12345.678] Memory cgroup out of memory: Killed process 40871 (node)
            total-vm:1284møB, anon-rss:498møB, ... oom_score_adj:0
[12345.679] Memory cgroup stats for /system.slice/docker-<CID>.scope: ...
```

**Docker's own record:**

```
$ docker inspect -f '{{.State.OOMKilled}} {{.State.ExitCode}}' orders-api
true 137                         ← the smoking gun: OOMKilled=true, exit 137
```

---

## How it works — step by step

**Memory limit → OOMKill, traced:**

1. You run `docker run --memory 512m orders-api`. Docker (via runc) creates the container's cgroup and writes `536870912` into `memory.max`.
2. Your Node process allocates memory normally. Each allocation increments `memory.current` as pages are actually touched (memory is charged on first write, not on `malloc`).
3. `orders-api` leaks — say it buffers every incoming order in an array that never drains. `memory.current` climbs toward `memory.max`.
4. An allocation would push usage over 512 MB. The kernel first tries **reclaim**: dropping clean page cache, swapping if allowed. (`high` counter in `memory.events` increments.)
5. Reclaim can't free enough (anonymous heap memory can't just be dropped). The kernel invokes `out_of_memory()` scoped to *this cgroup*.
6. The OOM killer picks a victim in the cgroup (usually the biggest — your Node process) and sends **SIGKILL**. SIGKILL cannot be caught or ignored — no graceful shutdown, no `finally` block runs.
7. PID 1 in the container dies from signal 9. The container exits with **137** (`128 + 9`). `docker inspect` records `OOMKilled: true`.
8. If `--restart` is set (Topic 12), Docker restarts the container — and if the leak persists, you get a **crash loop**: OOMKilled → restart → leak → OOMKilled...

**CPU limit → throttling, traced:**

1. You run `docker run --cpus 0.5 orders-api`. runc writes `50000 100000` into `cpu.max` (50 ms allowed per 100 ms period).
2. Under load, your Node event loop + libuv threads burn CPU. The kernel accounts every microsecond against the period's quota.
3. Within a given 100 ms period, the cgroup consumes its full 50 ms of CPU time.
4. The CFS bandwidth controller marks the cgroup **throttled**. Every runnable task in it is **dequeued** (removed from the run queue) — it will not be scheduled again until the next period begins.
5. `nr_throttled` and `throttled_usec` in `cpu.stat` increment. Your requests sit waiting; latency spikes.
6. At the next 100 ms boundary the quota refills and tasks resume. Nobody died — the container is just *periodically frozen*, which shows up as tail-latency, not as errors.

---

## Exact syntax breakdown

### Memory limit

```
docker run --memory 512m --memory-swap 512m orders-api:1.4.0
              │      │      │           │
              │      │      │           └─ total memory + swap. Setting it EQUAL to --memory
              │      │      │              disables swap → hard RAM cap (recommended in prod)
              │      │      └─ the swap-inclusive limit flag
              │      └─ 512 mebibytes. Suffixes: b, k, m, g. Written to memory.max.
              └─ set the container's hard memory ceiling
```

### Memory reservation (soft limit)

```
docker run --memory 512m --memory-reservation 256m orders-api:1.4.0
              │                  │              │
              │                  │              └─ soft target. Under host memory pressure,
              │                  │                 kernel tries to push this container back
              │                  │                 toward 256m, but won't kill for it.
              │                  └─ the soft-limit flag (maps to memory.low/high)
              └─ the hard limit still applies on top
```

### CPU limit

```
docker run --cpus 1.5 orders-api:1.4.0
              │    │
              │    └─ 1.5 CPU cores' worth of time. Written to cpu.max as "150000 100000"
              │       (150 ms quota per 100 ms period → can use 1.5 cores).
              └─ the CPU quota flag (CFS bandwidth). This is a HARD cap → throttling.
```

### CPU shares (relative weight, only matters under contention)

```
docker run --cpu-shares 512 orders-api:1.4.0
              │          │
              │          └─ relative weight (default 1024). A container with 512 gets HALF
              │             the CPU of a 1024 one — but ONLY when the CPU is contended.
              │             Idle CPU is free for anyone. Maps to cpu.weight.
              └─ soft CPU priority — NOT a cap (that's --cpus)
```

### Pinning to specific cores

```
docker run --cpuset-cpus 0,1 orders-api:1.4.0
              │           │
              │           └─ run only on CPU cores 0 and 1 (maps to cpuset.cpus)
              └─ restrict WHICH cores, not HOW MUCH — useful for isolating noisy workloads
```

### The Node.js-specific flag (critical!)

```
docker run --memory 512m -e NODE_OPTIONS="--max-old-space-size=384" orders-api:1.4.0
                                          │                   │
                                          │                   └─ cap V8's old heap at 384 MB,
                                          │                      safely UNDER the 512 MB cgroup
                                          │                      limit (leave room for buffers,
                                          │                      stack, C++ addons, code)
                                          └─ tell V8 the heap ceiling — it does NOT read
                                             the cgroup automatically!
```

---

## Example 1 — basic

See a memory kill happen on purpose.

```bash
# Run a container capped at 128 MB, then deliberately allocate ~300 MB.
docker run --rm --memory 128m --memory-swap 128m node:20-alpine \
  node -e '
    const chunks = [];
    setInterval(() => {
      chunks.push(Buffer.alloc(20 * 1024 * 1024)); // grab 20 MB each tick
      console.log("allocated", chunks.length * 20, "MB");
    }, 200);
  '
# allocated 20 MB
# allocated 40 MB
# ...
# allocated 120 MB
# Killed                          ← SIGKILL from the cgroup OOM killer

echo $?
# 137                             ← 128 + 9 (SIGKILL). The OOMKilled signature.
```

Now the same idea but for **CPU throttling** — no crash, just slowness:

```bash
# Cap at 0.25 CPU, run a busy loop, and watch the throttling counters climb.
docker run -d --name burn --cpus 0.25 node:20-alpine \
  node -e 'while (true) { Math.sqrt(Math.random()); }'

# Peek at the CPU cgroup stats (path uses the container id).
CID=$(docker inspect -f '{{.Id}}' burn)
cat /sys/fs/cgroup/system.slice/docker-$CID.scope/cpu.stat
#   nr_throttled 830          ← throttled hundreds of times
#   throttled_usec 41000000   ← 41 seconds of forced waiting and climbing
docker rm -f burn
```

The busy loop *wants* a full core but only gets 25% — the other 75% of every period it sits frozen. That's throttling, live.

---

## Example 2 — production scenario

**The situation.** Your `orders-api` runs alongside Postgres and Redis on a 4 GB, 2-CPU host. During a flash sale, the host froze: Postgres got OOM-killed by the *host* kernel, taking the database down, even though the actual leak was in the API. Post-mortem: no container had limits, `orders-api` leaked to ~3 GB, and the global OOM killer picked Postgres because it had the largest resident set at that instant.

**Fix step 1 — give every container a hard memory limit** so a leak is contained to its own cgroup:

```bash
docker run -d --name orders-api \
  --memory 512m --memory-swap 512m \                 # hard 512 MB, no swap
  -e NODE_OPTIONS="--max-old-space-size=384" \       # V8 heap safely under the cap
  --cpus 1 \                                          # 1 core max
  --restart on-failure:3 \                            # restart a few times, then give up
  orders-api:1.4.0

docker run -d --name postgres \
  --memory 1g --memory-reservation 768m \            # protect the DB with its own budget
  --cpus 1 \
  postgres:16-alpine

docker run -d --name redis --memory 256m --cpus 0.5 redis:7-alpine
```

Now when `orders-api` leaks, **only `orders-api`'s cgroup OOMs**. Postgres has its own guaranteed 1 GB and is never a victim of the API's bug. The blast radius is one container.

**Fix step 2 — make the kill observable** so you find out *before* customers do:

```bash
# Alert on this in monitoring; check it after any crash:
docker inspect -f '{{.State.OOMKilled}} {{.RestartCount}}' orders-api
#   true 3          ← it OOM-looped 3 times → there's a real leak to fix

# Confirm at the kernel level:
CID=$(docker inspect -f '{{.Id}}' orders-api)
grep oom_kill /sys/fs/cgroup/system.slice/docker-$CID.scope/memory.events
#   oom_kill 3
```

**Fix step 3 — diagnose the mysterious latency.** After adding `--cpus 1`, someone reports p99 latency doubled during peak. Check throttling before blaming the code:

```bash
cat /sys/fs/cgroup/system.slice/docker-$CID.scope/cpu.stat
#   nr_periods 60000
#   nr_throttled 22000        ← throttled in 37% of periods → the limit is too tight
#   throttled_usec 9.8e9      ← ~9.8s of cumulative waiting
```

37% throttling means the CPU cap is your latency, not your code. You raise `--cpus` to 1.5, redeploy, and `nr_throttled` drops to near zero — latency recovers. Without knowing about `cpu.stat` you'd have spent a day profiling JavaScript for nothing.

**The mental split that saved this system:** memory limits **contain crashes** (protect neighbors), CPU limits **shape latency** (and can silently cause it). Set both, and *watch* both.

---

## Common mistakes

### Mistake 1 — no memory limit, so the host OOM-kills the wrong container

**Symptom:** Postgres dies with exit 137 even though the bug is in your API. `dmesg` shows the host-level OOM killer, not a cgroup one. **Root cause:** with no per-container `memory.max`, the kernel's *global* OOM killer chooses a victim across the whole host by `oom_score` — often the biggest process, which may be an innocent bystander. **Fix:** give every container `--memory`. Now each leak is confined to its own cgroup and can only kill itself.

### Mistake 2 — Node OOMs "too early" because V8 doesn't know the limit

```
docker run --memory 512m orders-api
# <--- Last few GCs --->
# FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory
```

or the container is killed at ~500 MB when you expected 512. **Root cause:** V8's default old-space heap and libuv threads size themselves from *host* RAM (or a hardcoded default), not the cgroup. Node tries to grow past what `memory.max` allows. **Fix:** set `NODE_OPTIONS=--max-old-space-size=<~75% of limit>`. For 512 MB, use ~384. Leave headroom for non-heap memory (buffers, native addons, stack).

### Mistake 3 — confusing `--cpus` with `--cpu-shares`

```
docker run --cpu-shares 2048 orders-api   # expecting a hard 2-core cap
```

**Symptom:** the container still uses all cores when the host is idle. **Root cause:** `--cpu-shares` is a *relative weight* that only matters when CPUs are contended — it is NOT a cap. On an idle host, a high-share container uses everything available. `--cpus` is the hard quota. **Fix:** use `--cpus N` for an absolute cap; use `--cpu-shares` only to express relative priority between contending containers.

### Mistake 4 — setting `--cpus` too low and blaming the code for latency

**Symptom:** p99 latency spikes under load, no errors, CPU graph looks "not maxed" on the host. **Root cause:** the container is throttled every 100 ms period once it burns its quota; averaged host CPU looks fine, but your requests are periodically frozen. **Fix:** read `cpu.stat` — if `nr_throttled` is a meaningful fraction of `nr_periods`, raise `--cpus`. Don't profile JS until you've ruled out throttling.

### Mistake 5 — allowing swap to mask a leak

```
docker run --memory 512m orders-api    # (no --memory-swap → swap allowed up to 2x)
```

**Symptom:** the app gets slower and slower instead of failing fast; the leak hides for hours. **Root cause:** by default `--memory-swap` is twice `--memory`, so the container swaps to disk before OOMing — trading a clean crash for miserable, thrashing performance. **Fix:** in production set `--memory-swap` equal to `--memory` to disable swap, so a leak fails fast and visibly (OOMKilled) instead of degrading silently.

---

## Hands-on proof

```bash
# 1. Set a limit and read it straight out of the kernel.
docker run -d --name lim --memory 256m --cpus 0.5 node:20-alpine sleep 999
CID=$(docker inspect -f '{{.Id}}' lim)
BASE=/sys/fs/cgroup/system.slice/docker-$CID.scope
cat $BASE/memory.max     # 268435456  = 256 * 1024 * 1024
cat $BASE/cpu.max        # 50000 100000  = 0.5 CPU

# 2. Watch live memory usage climb.
cat $BASE/memory.current # bytes in use right now

# 3. Trigger an OOM kill and read the evidence.
docker run --rm --memory 64m --memory-swap 64m node:20-alpine \
  node -e 'const a=[]; while(true) a.push(Buffer.alloc(5*1024*1024));'
echo $?                  # 137  ← OOMKilled signature

# 4. Confirm Docker's own OOM flag on a killed container.
docker run -d --name oomtest --memory 64m --memory-swap 64m node:20-alpine \
  node -e 'const a=[]; while(true) a.push(Buffer.alloc(5*1024*1024));'
sleep 2
docker inspect -f '{{.State.OOMKilled}} {{.State.ExitCode}}' oomtest   # true 137
docker rm -f oomtest lim

# 5. See CPU throttling accumulate on a busy container.
docker run -d --name burn --cpus 0.2 node:20-alpine node -e 'while(true){}'
sleep 3
CID=$(docker inspect -f '{{.Id}}' burn)
cat /sys/fs/cgroup/system.slice/docker-$CID.scope/cpu.stat   # nr_throttled climbing
docker rm -f burn

# 6. See the kernel's own log of the kill (Linux host).
dmesg | grep -i "out of memory" | tail
```

---

## Practice exercises

### Exercise 1 — easy
Run `orders-api` (or any `node:20-alpine` container) with `--memory 200m --cpus 0.5`. Read `memory.max` and `cpu.max` from its cgroup directory and confirm they equal `209715200` and `50000 100000`. Explain how each number was derived.

### Exercise 2 — medium
Write a tiny Node script that allocates 20 MB every second. Run it with `--memory 128m` and time how long until it's OOMKilled. Then add `-e NODE_OPTIONS="--max-old-space-size=96"` and observe how Node now throws a catchable "heap out of memory" error *before* the cgroup kills it. Explain the difference between a V8 heap error and a cgroup OOM kill.

### Exercise 3 — hard (production simulation)
Reproduce the "noisy neighbor kills the database" scenario. (a) Start two containers with NO limits: a "postgres" (`postgres:16-alpine`) and an "api" that leaks memory in a loop. (b) Watch `dmesg` and `docker ps -a` to see which one the host OOM killer takes down. (c) Now redo it with per-container `--memory` limits sized to fit the host, plus `--memory-swap` equal to `--memory` and a `NODE_OPTIONS` heap cap on the api. (d) Prove the leak now only kills the api, that Postgres survives, and that the api's `RestartCount` and `memory.events oom_kill` counter both climb. Write up why per-container limits changed the outcome.

---

## Mental model checkpoint

1. Which cgroup file does `--memory` write to, and which does `--cpus` write to?
2. What happens when a container exceeds its memory limit vs. its CPU limit — and why is that asymmetry so important?
3. Why is the exit code exactly 137? What signal, and why can't the app clean up?
4. Why does Node.js sometimes OOM at a size below `--memory`, and what flag fixes it?
5. How do you *prove* a latency problem is CPU throttling and not slow code?
6. What's the difference between `--cpus` and `--cpu-shares`? When does each actually take effect?
7. Why set `--memory-swap` equal to `--memory` in production?

---

## Quick reference card

| Flag / file | What it does | Key detail |
|---|---|---|
| `--memory 512m` | Hard memory cap → `memory.max` | Exceed → OOMKilled, exit 137 |
| `--memory-swap 512m` | Total mem+swap; = `--memory` disables swap | Prod: set equal to fail fast |
| `--memory-reservation 256m` | Soft memory target under pressure | Not a hard cap; won't kill |
| `--cpus 1.5` | Hard CPU quota → `cpu.max` | Exceed → throttled, never killed |
| `--cpu-shares 512` | Relative weight → `cpu.weight` | Only matters when CPU contended |
| `--cpuset-cpus 0,1` | Pin to specific cores → `cpuset.cpus` | Which cores, not how much |
| `NODE_OPTIONS=--max-old-space-size` | Cap V8 heap under the cgroup limit | Node doesn't read cgroup itself |
| `memory.current` | Live bytes used | Watch a leak climb |
| `memory.events` (`oom_kill`) | Count of OOM kills | Proof of a real leak |
| `cpu.stat` (`nr_throttled`) | Count of throttled periods | Proof of CPU throttling |
| `docker inspect .State.OOMKilled` | Was it OOMKilled? | `true` + exit 137 = the signature |

---

## When would I use this at work?

1. **Sizing containers before a launch.** You load-test `orders-api`, watch `memory.current` and `cpu.stat`, and pick `--memory`/`--cpus` values that fit real usage with headroom — so one service can't starve the others on a shared host.

2. **Debugging a crash loop.** A container keeps restarting. `docker inspect` shows `OOMKilled: true` and exit 137, `memory.events` shows a climbing `oom_kill` count — you now *know* it's a memory leak, not a config bug, and where to look.

3. **Explaining mysterious tail latency.** Latency spikes under load with no errors. You check `cpu.stat`, see high `nr_throttled`, raise `--cpus`, and the latency vanishes — a five-minute fix instead of a day of profiling.

---

## Connected topics

- **Study before:** Topic 02 (Linux Foundations — cgroups, the mechanism behind every limit here), Topic 12 (docker run in Depth — where `--memory`/`--cpus` live among the other flags), Topic 16 (Container Lifecycle — exit codes and restart behavior).
- **Study after:** Topic 44 (Resource Requests and Limits in Kubernetes — the same cgroups, now with requests, limits, and QoS classes), Topic 45 (Horizontal Pod Autoscaler — scaling on the CPU/memory you've learned to measure), Topic 43 (Health Checks in Kubernetes — how a crash-looping container is detected and handled).
