# 16 — Container Lifecycle

## Section: Docker in Practice

---

## ELI5 — The Simple Analogy

Think of a container like a person and their day.

- **Created** — they're dressed and standing at the front door, but haven't stepped outside yet. Ready, not moving.
- **Running** — they're out living their day, doing work.
- **Paused** — someone hit a magic freeze button. They're frozen mid-step, still standing there, using no energy, remembering exactly what they were doing. Unfreeze and they continue the exact motion.
- **Stopped / Exited** — they came home and went to sleep. Their body (the room, their stuff) is still there, you can look through their pockets (logs, files), but they're not doing anything.
- **Dead** — something went so wrong that we couldn't even put them to bed properly. They're stuck in a broken half-state and we mostly just need to clean up.

The key insight kids miss and adults miss too: **"stopped" is not "deleted."** A stopped container is like a sleeping person whose room is untouched — you can wake them (restart) and even read everything in their pockets. Only `docker rm` throws the room away.

And the "stop" itself is polite-then-forceful: Docker first taps you on the shoulder and says "please wrap up" (SIGTERM). If you don't wrap up within a grace period, it drags you out (SIGKILL). Good apps listen for the tap and save their work.

---

## The Linux kernel feature underneath

A Docker container "state" is really just **the state of a Linux process** — specifically the process the kernel calls PID 1 inside the container's PID namespace (Topic 02). Docker's lifecycle words are a thin label on top of ordinary kernel process states and signals.

**1. Process states in the kernel.** Every process on Linux has a state, visible in `/proc/[PID]/stat` (field 3) and as a letter in `ps`:

```
R  running or runnable (on a CPU or waiting for one)
S  sleeping (interruptible — waiting for something, e.g. a socket)
D  uninterruptible sleep (usually disk I/O)
T  stopped (by a signal — this is what "paused" uses)
Z  zombie (finished, but parent hasn't collected its exit code yet)
X  dead
```

Map that to Docker:

```
Docker state     Kernel reality
────────────     ──────────────────────────────────────────────
running     →    PID 1 exists, state R/S/D, in its namespaces & cgroup
paused      →    every process in the container's cgroup is FROZEN (state D/T via freezer)
exited      →    PID 1 has terminated; the kernel process is gone, only bookkeeping remains
```

**2. The freezer cgroup — how "paused" works.** Pause is NOT "send a stop signal." Docker writes to the **freezer cgroup controller**:

```
/sys/fs/cgroup/.../<container>/cgroup.freeze
                              │
                              └─ write "1" to freeze ALL processes in the cgroup atomically,
                                 "0" to thaw. (cgroup v2; v1 used freezer.state = FROZEN/THAWED)
```

The kernel scheduler simply stops giving those processes CPU time. They can't run, can't even see they were frozen — unlike a signal, the app gets no notification. That's why pause is invisible to the app and instantly reversible.

**3. Signals — how "stop" works.** Stopping is the kernel signal mechanism. Docker sends **SIGTERM** (signal 15) to PID 1 via the `kill(2)` syscall, waits a grace period, then sends **SIGKILL** (signal 9):

```
SIGTERM (15)  "please shut down"  → the process CAN catch it and clean up (close DB pool, finish requests)
SIGKILL (9)   "die now"           → the kernel kills it; the process CANNOT catch, block, or ignore it
```

**4. PID 1 is special — and this bites Node apps.** In every PID namespace, PID 1 has two kernel-given superpowers/duties:

- **Default signal handlers are disabled for PID 1.** For a normal process, SIGTERM's default action is "terminate." For PID 1, the kernel **ignores** signals that have no explicit handler. So if your Node app is PID 1 and you never registered a SIGTERM handler, `docker stop` sends SIGTERM… and nothing happens, and 10 seconds later SIGKILL is used. (More in Common Mistakes.)
- **PID 1 must reap zombies.** When any process's child exits, it becomes a zombie until the parent calls `wait()`. If a grandchild's parent dies, the grandchild is re-parented to PID 1, which is expected to `wait()` on it. If PID 1 doesn't reap, zombies pile up. This is "reaping," covered below.

Everything in this topic is these four kernel facts wearing Docker clothing.

---

## What is this?

A container's lifecycle is the set of states it moves through from creation to removal: **created → running → (paused) → stopped/exited → dead → removed.** Each state corresponds to a concrete kernel condition — a process that exists-but-hasn't-exec'd, a running process, a frozen cgroup, a terminated process, or a broken teardown.

This topic explains what each state means at the kernel level, how the graceful SIGTERM→SIGKILL stop sequence works, how restart policies decide whether a dead container comes back, and how zombie reaping works so your container doesn't accumulate defunct processes.

---

## Why does it matter for a backend developer?

Your `orders-api` is not a script that runs once — it's a long-lived server that gets deployed, redeployed, scaled, and killed constantly. If you don't understand the lifecycle, real production pain follows:

- **Dropped requests on every deploy.** If your app doesn't handle SIGTERM, `docker stop` kills it mid-request. Users see 502s every time you ship. With graceful shutdown, in-flight orders finish and the load balancer drains cleanly.
- **A crash loop you can't see.** Your container keeps exiting and restarting. Without knowing exit codes and restart policies, you can't tell "crashed with error 1" from "OOMKilled (137)" from "cleanly stopped." Each has a totally different fix.
- **Zombie pileups.** Your app shells out to run image-processing subprocesses. If it's PID 1 and doesn't reap, defunct processes accumulate until you hit the PID limit and the container can't fork anymore.
- **"Why is my container gone?"** You didn't use `--rm` or `--restart`, your app exited, and you can't understand why `docker ps` is empty but `docker ps -a` shows it Exited. Knowing the states tells you exactly where your app is and how to inspect it.

Mastering this is what makes your deploys zero-downtime and your incidents debuggable.

---

## The physical reality

When a container exists in any non-removed state, this lives on the host:

```
/var/lib/docker/containers/<container-id>/
├── config.v2.json      ← the container's config AND its current state:
│                          "State": {"Status":"running","Running":true,"Paused":false,
│                                    "Pid":48213,"ExitCode":0,"OOMKilled":false,
│                                    "StartedAt":"...","FinishedAt":"..."}
├── hostconfig.json     ← includes "RestartPolicy": {"Name":"on-failure","MaximumRetryCount":3}
├── <container-id>-json.log   ← stdout/stderr captured (Topic 17) — survives after exit
└── mounts/             ← volume mount metadata
```

When the container is **running**, additionally:

```
/proc/48213/                  ← the host-side PID of the container's PID 1 (here 48213)
├── status                    ← State: R/S/D..., and per-signal masks (SigCgt = caught signals)
├── ns/pid, ns/net, ...       ← the namespaces this container lives in
└── environ, cmdline, fd/     ← its environment, command, open files

/sys/fs/cgroup/system.slice/docker-<id>.scope/
├── cgroup.freeze             ← "0" normally; Docker writes "1" for `docker pause`
├── cgroup.events             ← "frozen 0/1", "populated 0/1"
└── memory.current, ...       ← live resource usage (Topic 25)
```

Key physical facts:

- The **container ID and its `-json.log` survive** in `/var/lib/docker/containers/` after exit. That's why `docker logs` works on a stopped container and why a stopped container isn't "gone."
- When **exited**, there is NO `/proc/<pid>/` for the container's PID 1 — the kernel process is gone. Only the on-disk bookkeeping remains.
- `docker rm` deletes the `/var/lib/docker/containers/<id>/` directory. *That's* what actually removes a container.

Note the two PIDs: inside the container your app sees itself as **PID 1** (its own PID namespace), but on the host it has a **different real PID** (48213 above). `docker inspect --format '{{.State.Pid}}'` gives you the host PID.

---

## How it works — step by step

### The full lifecycle trace

```
docker create → [CREATED] → docker start → [RUNNING] ⇄ docker pause/unpause → [PAUSED]
                                              │
                                              ├─ docker stop / app exits → [EXITED]
                                              │        │
                                              │        ├─ restart policy fires → [RUNNING] (loop)
                                              │        └─ docker start → [RUNNING]
                                              │
                                              └─ teardown fails → [DEAD] → docker rm → (removed)
```

### 1. Created

```
docker create --name orders orders-api:1.0
```

- Daemon → containerd → runc builds the container bundle: namespaces are prepared, the writable overlay2 layer is mounted at `/var/lib/docker/overlay2/<id>/merged`, cgroups are created, the env array is assembled (Topic 15).
- **But `execve` has NOT been called.** No PID 1 yet. The container has an ID, a filesystem, and a config, and does nothing. `config.v2.json` says `"Status":"created"`.
- This is why `docker create` + `docker start` exists separately from `docker run` (which is create+start in one).

### 2. Running

```
docker start orders          # or docker run does create+start together
```

- runc calls `execve` on your `CMD`/`ENTRYPOINT`. Your Node process is born as PID 1 in the new PID namespace.
- `"Status":"running"`, `"Pid":48213` recorded. `/proc/48213` now exists on the host.
- The process runs until it exits on its own OR receives a fatal signal.

### 3. Paused

```
docker pause orders
```

- Daemon writes `1` to `/sys/fs/cgroup/.../docker-<id>.scope/cgroup.freeze`.
- Kernel freezer stops scheduling every process in the cgroup. They keep all their memory, open sockets, and register state — they simply get no CPU.
- `"Status":"paused"`, `"Paused":true`. `docker unpause` writes `0` and they resume mid-instruction with no awareness of the gap.
- Use case: freeze a container to take a consistent snapshot, or to instantly relieve CPU pressure without losing state.

### 4. Stopping (the graceful sequence)

```
docker stop orders           # default grace period: 10 seconds
```

This is the most important sequence in the topic:

```
1. Daemon sends SIGTERM (15) to PID 1.
      kill(48213, SIGTERM)
2. Daemon starts a timer (default 10s, override with -t / --time).
3a. If PID 1 exits before the timer → clean stop. ExitCode is whatever the app returned.
3b. If the timer expires and PID 1 is still alive:
      Daemon sends SIGKILL (9) to PID 1.
      kill(48213, SIGKILL)
      The kernel destroys the process immediately. ExitCode becomes 137.
4. Container transitions to EXITED. FinishedAt is recorded.
```

The number **137** is worth memorizing: `128 + 9` = terminated by signal 9 (SIGKILL). `143` = `128 + 15` (SIGTERM took it down). `130` = `128 + 2` (SIGINT, i.e. Ctrl-C).

Your app's job during step 1→3a: catch SIGTERM, stop accepting new work, finish in-flight requests, close the Postgres pool and Redis connection, then exit 0 — all within the grace window.

### 5. Exited

- PID 1 is gone. No `/proc/<pid>`. The overlay2 writable layer, the config, and the JSON log **remain** on disk.
- `docker ps` (running only) does NOT show it; `docker ps -a` (all) does, as `Exited (0) 3 minutes ago` or `Exited (137) ...`.
- You can still `docker logs orders`, `docker inspect orders`, `docker cp` files out, or `docker start orders` to bring it back.

### 6. Dead

- A rare, bad state. It means the daemon tried to remove/tear down the container but **couldn't fully** — e.g. a filesystem/mount was busy and unmount failed, or the daemon crashed mid-operation.
- The container is non-functional and can't be started. `"Status":"dead"`.
- Fix: `docker rm -f <id>`; occasionally you must manually unmount a stuck mount under `/var/lib/docker/overlay2/` or restart the daemon.

### 7. Removed

```
docker rm orders             # container must be stopped, or use -f to force
```

- Deletes `/var/lib/docker/containers/<id>/` including the JSON log, and drops the writable overlay2 layer. Now it's truly gone. `docker ps -a` no longer lists it.

### Restart policies — who brings a container back

When PID 1 exits, the daemon consults `hostconfig.json`'s `RestartPolicy`:

```
no             → never restart (default). Exited stays exited.
on-failure[:N] → restart ONLY if exit code ≠ 0, up to N times (backoff grows each try)
always         → always restart if it stops, EXCEPT it stays stopped if you `docker stop` it
                 manually; comes back when the daemon (re)starts
unless-stopped → like always, but if you manually stopped it, it stays stopped even across
                 daemon restarts
```

The daemon uses **exponential backoff** (100ms, then doubling) between restarts so a fast-crashing container doesn't spin the CPU. A container repeatedly failing shows status `Restarting`.

### Reaping — cleaning up zombies

When a process exits, it stays a **zombie** (state Z) holding just its exit code until its parent calls `wait()`. Inside a container, orphaned processes get re-parented to **PID 1** (your app). If PID 1 doesn't `wait()`, zombies accumulate in the process table forever. This matters when your app spawns subprocesses. Two fixes:

- Run a tiny **init** as PID 1 that reaps for you: `docker run --init ...` injects `tini` as PID 1, which forwards signals to your app AND reaps zombies.
- Or handle it in-app (Node's `child_process` cleans up children you explicitly `wait`/listen for, but re-parented grandchildren still need an init).

---

## Exact syntax breakdown

### `docker stop` with grace period

```
docker stop -t 30 orders
            │  │  │
            │  │  └─ the container name or ID
            │  └─ seconds to wait after SIGTERM before sending SIGKILL
            └─ -t / --time: the grace period (default 10)
```

### `docker kill` — send a specific signal now (no grace)

```
docker kill --signal=SIGTERM orders
            │       │
            │       └─ which signal to send (default SIGKILL)
            └─ --signal / -s: send exactly this signal immediately, no waiting
```

`docker kill` skips the polite phase entirely. `docker kill -s SIGTERM` sends only SIGTERM (useful to test your handler); plain `docker kill` sends SIGKILL right away.

### `docker pause` / `docker unpause`

```
docker pause orders      # freeze via cgroup freezer — app gets NO signal, keeps all state
docker unpause orders    # thaw — resumes mid-instruction
```

### `--restart` policy on run

```
docker run --restart on-failure:3 orders-api:1.0
           │         │          │
           │         │          └─ max retry count (only valid with on-failure)
           │         └─ the policy: no | on-failure | always | unless-stopped
           └─ set what happens when PID 1 exits
```

### `--stop-signal` — change which signal means "stop"

```
docker run --stop-signal=SIGINT orders-api:1.0
           │             │
           │             └─ the signal docker stop will send first (default SIGTERM)
           └─ some apps shut down cleanly on SIGINT/SIGQUIT instead of SIGTERM
```

(You can also set this in the Dockerfile: `STOPSIGNAL SIGINT`.)

### `--init` — inject an init/reaper as PID 1

```
docker run --init orders-api:1.0
           │
           └─ makes tini PID 1; it forwards signals to your app and reaps zombies
```

### Inspecting state

```
docker inspect -f '{{.State.Status}} exit={{.State.ExitCode}} oom={{.State.OOMKilled}}' orders
               │  │      │                  │                    │
               │  │      │                  │                    └─ true if the kernel OOM-killer hit it
               │  │      │                  └─ the exit code (137 = SIGKILL, 143 = SIGTERM, 0 = clean)
               │  │      └─ current lifecycle state
               │  └─ the Go-template format string
               └─ -f / --format: extract specific fields
```

---

## Example 1 — basic

Watch a container move through the states, and see exit codes.

```
# CREATED (exists, not started)
docker create --name demo alpine sleep 30
docker inspect -f '{{.State.Status}}' demo        # created

# RUNNING
docker start demo
docker inspect -f '{{.State.Status}}' demo        # running
docker ps                                          # shows demo

# PAUSED (frozen via cgroup freezer)
docker pause demo
docker inspect -f '{{.State.Status}} {{.State.Paused}}' demo   # paused true
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' demo).scope/cgroup.freeze 2>/dev/null
# → 1   (on cgroup v2 Linux hosts)
docker unpause demo                                # running again

# EXITED (sleep finishes on its own → exit 0)
sleep 30
docker inspect -f '{{.State.Status}} exit={{.State.ExitCode}}' demo   # exited exit=0
docker ps                                          # empty (not running)
docker ps -a                                        # shows demo, "Exited (0)"

# The log and config survive — container is NOT gone:
docker logs demo                                    # (empty here, but the mechanism works)

# REMOVED
docker rm demo
docker ps -a                                         # demo no longer listed
```

Now see a non-zero exit and SIGKILL code:

```
docker run --name crash alpine sh -c 'exit 42'
docker inspect -f 'exit={{.State.ExitCode}}' crash    # exit=42   (your app's own code)
docker rm crash

docker run -d --name stubborn alpine sh -c 'trap "" TERM; while true; do sleep 1; done'
#   trap "" TERM  → this shell IGNORES SIGTERM on purpose
time docker stop stubborn         # takes ~10s: SIGTERM ignored, then SIGKILL
docker inspect -f 'exit={{.State.ExitCode}}' stubborn  # exit=137  (128 + 9 = SIGKILL)
docker rm stubborn
```

The 10-second wait you just felt IS the grace period. The 137 IS SIGKILL.

---

## Example 2 — production scenario

**The situation:** Your `orders-api` drops ~30 requests every deploy. Users occasionally see "order failed" right when you ship. The logs show requests being cut off mid-flight. Root cause: the app doesn't handle SIGTERM, so `docker stop` waits 10s then SIGKILLs it, severing in-flight connections and leaving the Postgres pool open.

**The broken app (no graceful shutdown):**

```javascript
// server.js — RUNNING but NOT lifecycle-aware
const express = require('express');
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const app = express();

app.post('/orders', async (req, res) => {
  const r = await pool.query('INSERT INTO orders(status) VALUES($1) RETURNING id', ['new']);
  res.json({ id: r.rows[0].id });
});

app.listen(3000, () => console.log('orders-api up'));
// docker stop → SIGTERM arrives → NOBODY listening → 10s later SIGKILL → in-flight INSERT cut off
```

**The fix — handle SIGTERM, drain, close pools, exit clean:**

```javascript
// server.js — lifecycle-aware graceful shutdown
const express = require('express');
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const app = express();

app.post('/orders', async (req, res) => {
  const r = await pool.query('INSERT INTO orders(status) VALUES($1) RETURNING id', ['new']);
  res.json({ id: r.rows[0].id });
});

const server = app.listen(3000, () => console.log('orders-api up'));

async function shutdown(signal) {
  console.log(`${signal} received — draining`);
  // 1) Stop accepting new connections; let in-flight requests finish.
  server.close(async () => {
    // 2) Now that HTTP is drained, close the DB pool cleanly.
    await pool.end();
    console.log('drained, exiting 0');
    process.exit(0);           // clean exit within the grace window
  });
  // 3) Safety net: if draining hangs, force-exit before Docker's SIGKILL.
  setTimeout(() => { console.error('drain timeout, forcing'); process.exit(1); }, 25000);
}

// Catch BOTH — Docker sends SIGTERM; local Ctrl-C sends SIGINT.
process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

**Deploy config to match:**

```
docker run -d --name orders \
  --restart unless-stopped \       # auto-recover from crashes, but respect manual stops
  --stop-timeout 30 \              # give the app up to 30s to drain (matches the 25s safety net)
  --init \                         # tini reaps any zombies from image-processing subprocesses
  -e DATABASE_URL=postgres://orders@postgres:5432/orders \
  orders-api:2.0
```

**The critical PID-1 gotcha in this scenario:** the above works only if Node is PID 1 *and* actually receives the signal. If your Dockerfile uses **shell form** `CMD node server.js`, then `/bin/sh` is PID 1 and Node is its child — `sh` does NOT forward SIGTERM to Node, so your handler never fires and you're back to the 10s-then-SIGKILL problem. Use **exec form** `CMD ["node", "server.js"]` so Node is PID 1 directly. (This is the lifecycle payoff of Topic 07, CMD vs ENTRYPOINT.)

**Verify the fix:**

```
docker stop orders     # returns in ~1s now, not 10s
docker inspect -f 'exit={{.State.ExitCode}}' orders   # exit=0, not 137
docker logs orders | tail -3
#   SIGTERM received — drained
#   drained, exiting 0
```

Zero-downtime deploys, no dropped orders.

---

## Common mistakes

### Mistake 1 — Node as a child of `sh`, so SIGTERM never reaches it

**What you did:** `CMD node server.js` (shell form).

**Symptom:** `docker stop` always takes the full grace period; exit code 137; your `process.on('SIGTERM')` handler never logs anything.

**Prove it:**

```
docker exec orders ps -o pid,comm
# PID  COMMAND
#   1  sh          ← sh is PID 1
#   7  node        ← node is a CHILD; sh doesn't forward SIGTERM
```

**Root cause:** shell form runs `/bin/sh -c "node server.js"`. `sh` becomes PID 1, `node` is a child. `sh` doesn't propagate signals. **Right:** exec form `CMD ["node", "server.js"]` — now `ps` shows node as PID 1.

### Mistake 2 — App is PID 1 but has no SIGTERM handler

**What you did:** exec form correctly, but never registered `process.on('SIGTERM', ...)`.

**Symptom:** `docker stop` takes the full 10s; exit 137.

**Root cause:** the kernel disables default signal handlers **for PID 1**. A normal process would default-terminate on SIGTERM; PID 1 with no handler **ignores** it. So SIGTERM does nothing and SIGKILL is needed. **Right:** always register a SIGTERM handler in any process that will be PID 1.

### Mistake 3 — Reading 137 as "my app crashed"

**What you saw:**

```
docker inspect -f '{{.State.ExitCode}} oom={{.State.OOMKilled}}' orders
# 137 oom=true
```

**Root cause:** 137 = 128 + 9 (SIGKILL). It can come from `docker stop` timing out **or** from the kernel OOM-killer. The `OOMKilled: true` field disambiguates: here the kernel killed it for exceeding its memory cgroup limit (Topic 25), not a deploy. **Right:** check `OOMKilled` — if true, raise `--memory` or fix the leak; if false, it's a stop-timeout/graceful-shutdown problem.

### Mistake 4 — Zombie pileup from an un-reaped PID 1

**Symptom:**

```
docker exec orders ps -o pid,stat,comm | grep Z
#  1234 Z    convert <defunct>
#  1235 Z    convert <defunct>     ← growing over time; eventually "fork: retry: Resource temporarily unavailable"
```

**Root cause:** your app spawns `convert`/`ffmpeg` subprocesses; their children get re-parented to PID 1, which never calls `wait()`. Zombies fill the process table until the PID limit is hit. **Right:** `docker run --init` (tini reaps them), or ensure your app `wait()`s on every child.

### Mistake 5 — Assuming `docker stop` deletes the container

**What you did:** `docker stop orders`, then `docker run --name orders ...` again → `Conflict. The container name "/orders" is already in use`.

**Root cause:** `stop` transitions to **exited**, it does NOT remove. The name and `/var/lib/docker/containers/<id>/` still exist. **Right:** `docker rm orders` (or run with `--rm` so it auto-removes on exit, or `docker start orders` to reuse it).

---

## Hands-on proof

```
# 1) See the graceful sequence and the 10s grace with your own stopwatch
docker run -d --name grace alpine sh -c 'trap "echo caught TERM; exit 0" TERM; while true; do sleep 1; done'
docker stop grace                 # returns fast — the trap catches TERM and exits 0
docker inspect -f 'exit={{.State.ExitCode}}' grace   # exit=0
docker rm grace

docker run -d --name nograce alpine sh -c 'trap "" TERM; while true; do sleep 1; done'
time docker stop nograce          # ~10s — TERM ignored, SIGKILL used
docker inspect -f 'exit={{.State.ExitCode}}' nograce # exit=137
docker rm nograce

# 2) See PAUSED freeze a process via the cgroup (Linux host)
docker run -d --name p alpine sh -c 'while true; do date +%s >> /t; sleep 1; done'
docker pause p
sleep 3
docker unpause p
docker exec p sh -c 'tail -5 /t'  # notice the ~3s GAP in timestamps: it was frozen, not sleeping
docker rm -f p

# 3) See restart policy + backoff on a crash loop
docker run -d --name loop --restart on-failure:3 alpine sh -c 'exit 1'
sleep 3
docker inspect -f '{{.State.Status}} restarts={{.RestartCount}}' loop  # eventually exited restarts=3
docker rm -f loop

# 4) See zombies WITHOUT init, and their absence WITH --init
docker run -d --name noinit alpine sh -c 'while true; do (sleep 0.1 &); sleep 1; done'
sleep 3; docker exec noinit sh -c 'ps -o stat,comm | grep -c Z'   # nonzero: zombies pile up
docker rm -f noinit
docker run -d --init --name withinit alpine sh -c 'while true; do (sleep 0.1 &); sleep 1; done'
sleep 3; docker exec withinit sh -c 'ps -o pid,comm | head'       # PID 1 is tini; zombies reaped
docker rm -f withinit
```

Each proves a kernel fact: (1) SIGTERM→SIGKILL and exit codes; (2) freezer cgroup pause; (3) restart policy + count; (4) reaping.

---

## Practice exercises

### Exercise 1 — easy

Run `alpine sleep 60` detached. Using only `docker inspect -f`, print its `Status`, then `docker pause` it and print `Status` and `Paused`, then `docker unpause` and confirm it's running again. Finally let it exit and show `docker ps` is empty while `docker ps -a` still lists it. Explain in one line why the container still appears in `ps -a`.

### Exercise 2 — medium

Write two tiny apps: A) exec-form `CMD ["node","app.js"]` where `app.js` registers a SIGTERM handler that logs and exits 0; B) shell-form `CMD node app.js` with the same code. Deploy both and `time docker stop` each. Record the two durations and the two exit codes, then use `docker exec ... ps` to explain *why* B took ~10s (hint: what's PID 1?).

### Exercise 3 — hard (production simulation)

Simulate a crash loop and an OOM kill and tell them apart. (a) Run a container with `--restart on-failure:5` whose command sleeps 2s then `exit 1`; observe `RestartCount` climb and the exponential backoff between restarts (watch `docker events`). (b) Run a container with `--memory=32m` running a Node script that allocates an ever-growing array; capture the exit code and `OOMKilled` field. (c) Write a short runbook paragraph: given a container in `Exited (137)`, list the exact commands and fields you'd check to decide whether it was a deploy stop-timeout, a manual kill, or an OOM — and the fix for each.

---

## Mental model checkpoint

1. What kernel condition corresponds to each of: created, running, paused, exited?
2. What exactly does `docker pause` write, and why does the app get no notification?
3. Describe the SIGTERM→SIGKILL sequence and name the default grace period.
4. What do exit codes 0, 137, and 143 each mean?
5. Why does a Node app that IS PID 1 still ignore SIGTERM if it has no handler?
6. Why does shell-form `CMD` break graceful shutdown even with a handler?
7. What is a zombie, why does PID 1 have to reap, and what does `--init` do about it?
8. Why doesn't `docker stop` free up the container name, and what does?

---

## Quick reference card

| Command / flag / field | What it does | Key detail |
|---|---|---|
| `docker create` | Make a container without starting | State = created; no PID 1 yet |
| `docker start` | Run PID 1 via `execve` | Reuses a created/exited container |
| `docker pause` / `unpause` | Freeze/thaw via cgroup freezer | App gets no signal; resumes mid-instruction |
| `docker stop [-t N]` | SIGTERM, wait N s, then SIGKILL | Default grace 10s; sets exit code |
| `docker kill [-s SIG]` | Send a signal immediately | Default SIGKILL; no grace period |
| `docker rm` | Delete container dir + writable layer | This is what actually removes it |
| `--restart no\|on-failure\|always\|unless-stopped` | Auto-restart policy | Exponential backoff between retries |
| `--stop-signal` / `STOPSIGNAL` | Change the "stop" signal | Default SIGTERM |
| `--stop-timeout` / `-t` | Grace period before SIGKILL | Match your app's drain time |
| `--init` | tini as PID 1 | Forwards signals AND reaps zombies |
| exit 137 / 143 / 130 | 128 + signal (9 / 15 / 2) | 137 = SIGKILL, 143 = SIGTERM, 130 = Ctrl-C |
| `.State.OOMKilled` | Kernel OOM-killer flag | Disambiguates a 137 |

---

## When would I use this at work?

1. **Zero-downtime deploys.** Every rolling deploy sends SIGTERM to old containers. Implementing graceful shutdown (drain HTTP, close Postgres/Redis pools, exit 0 within the grace window) is what makes deploys invisible to users instead of a burst of 502s.
2. **Diagnosing a crash loop at 2am.** A container is flapping. You read `.State.ExitCode`, `.State.OOMKilled`, and `RestartCount` to instantly classify it: app error (exit 1), memory limit (137 + OOMKilled true), or stop-timeout (137 + OOMKilled false) — each with a different fix.
3. **Batch/worker containers that shell out.** Your image-processing or video worker spawns subprocesses. You add `--init` so re-parented children are reaped, preventing zombie pileups that would eventually exhaust PIDs and stall the worker.

---

## Connected topics

- **Study before:** Topic 02 (Linux Foundations — PID namespaces, cgroups, the freezer), Topic 05 (Your First Container — first look at lifecycle states), Topic 07 (CMD vs ENTRYPOINT — exec vs shell form decides whether your app is PID 1), Topic 12 (`docker run` in Depth — `--restart`, `--init`, `--stop-timeout`).
- **Study after:** Topic 17 (Logging — logs persist across exit and are how you read a dead container), Topic 25 (Resource Limits — OOMKilled and 137 in depth), Topic 26 (Health Checks — how Docker decides a running container is unhealthy), Topic 33 (Pods in Depth — Kubernetes pod lifecycle phases build directly on this), Topic 43 (Health Checks in Kubernetes — liveness probes and the same graceful-shutdown story at pod scale).
