# 05 — Your First Container
## Section: Docker Foundations

---

## ELI5 — The Simple Analogy

Think of a **theater stage** being set up for one actor.

Before the actor walks on, stagehands rush around doing invisible work:
- They build the **walls** so the actor can only see their own set, not the whole theater (that's the *namespaces* — the actor thinks their little set is the entire world).
- They decide **how much food and water** the actor is allowed, so one greedy actor can't eat the whole building's supplies (that's *cgroups* — resource limits).
- They lay down the **floor** using those stacked glass sheets from Topic 04, plus one fresh blank sheet the actor can scribble on (that's the *overlay mount*).
- They run a **phone line** from the actor's set to the front desk so messages can go in and out (that's the *virtual network cable* — the veth pair — and the switchboard rules, the iptables).
- Finally they push **one actor** onto the stage and say "you are now person number 1 here" (that's *PID 1*).

`docker run` is you shouting "action!" All those stagehands are the kernel doing setup in a precise order. When people say "a container started instantly," they mean all of this happened in a few hundred milliseconds. This topic watches the stagehands, step by step.

---

## The Linux kernel feature underneath

A container is **not one kernel feature** — it is several combined at the moment of `run`. There is no `create_container()` syscall in Linux. Docker assembles a container out of primitives you met in Topic 02:

```
namespaces  → what the process can SEE     (clone() / unshare() / setns() with CLONE_NEW* flags)
cgroups     → what the process can USE      (write limits into /sys/fs/cgroup/…)
overlayfs   → what filesystem it gets       (mount -t overlay …, from Topic 04)
capabilities→ what privileged ops it can do (drop most of root's powers)
seccomp     → which syscalls it may call    (a filter blocking ~44 dangerous syscalls)
veth + iptables → how it talks on the network
```

The core kernel call is **`clone(2)`** (used by runc, via Go's runtime). `clone()` is like `fork()` but takes flags to say "give the new process brand-new namespaces":

```
CLONE_NEWPID   → new PID namespace  → the process becomes PID 1 inside
CLONE_NEWNS    → new mount namespace→ its own view of mounts (the overlay root)
CLONE_NEWNET   → new network namespace → its own interfaces, routes, iptables
CLONE_NEWUTS   → new hostname
CLONE_NEWIPC   → new shared-memory / semaphores
CLONE_NEWUSER  → new user-id mapping (optional; rootless mode uses it)
```

After `clone()`, the new process still needs its filesystem swapped. runc calls **`pivot_root(2)`** (not `chroot` — `pivot_root` is stronger and harder to escape) to make the overlay `merged/` directory the new `/`. Then it `execve()`s your actual program (`node server.js`), which *becomes* PID 1.

> Remember from Topic 03 — you type `docker run`, the CLI sends an HTTP call to the **daemon (dockerd)**, which asks **containerd**, which spawns a **containerd-shim**, which execs **runc**. It is *runc* that actually makes the `clone()`/`pivot_root()`/`execve()` syscalls above and then exits, leaving the shim as the container's babysitter. This topic is the "what runc does" zoom-in.

---

## What is this?

A **container** is a normal Linux process (or process tree) that the kernel has wrapped in namespaces (isolated view), cgroups (capped resources), and an overlay root filesystem. `docker run` is the command that builds an image's layers into that live, isolated process. Nothing is virtualized — it is your host kernel running one more process that simply cannot see or exceed its box.

---

## Why does it matter for a backend developer?

Because "the container started but immediately exited" and "the container is running but I can't reach it" are daily realities, and both are explained by the trace below:

- **PID 1 confusion.** Your `orders-api` ignores `Ctrl-C` / takes 10s to stop on deploy. That's because your app runs as **PID 1**, and PID 1 has special signal rules (see step 9 and Common Mistakes). Knowing this is the difference between graceful shutdown and killed-in-flight requests.
- **"Exited (0)" immediately.** You ran a container detached and it vanished. Understanding that a container lives exactly as long as its PID 1 explains why.
- **Detached vs interactive.** Knowing what `-d`, `-it`, `attach`, `logs`, and `exec` actually connect to (which file descriptors, which namespace) stops you from "hanging" your terminal or losing logs.
- **Networking.** "Container runs but `curl localhost:3000` fails" is a veth/iptables/`-p` story, and you'll see exactly where the packet goes.
- **Debugging prod.** `docker exec` drops you *into* the container's namespaces — the single most useful debugging move you have.

---

## The physical reality

The instant `docker run -d -p 8080:3000 --name orders-api orders-api:latest` is live, these concrete things exist on the host:

```
PROCESS TREE (host view — `ps`):
  systemd(1)
   └─ dockerd
       └─ containerd
           └─ containerd-shim-runc-v2   ← the babysitter, re-parents to PID 1 if daemon restarts
               └─ node server.js         ← host PID e.g. 34512
                                            (runc already exited after setup)

NAMESPACES (one set per container):
  /proc/34512/ns/pid   → pid:[4026532abc]     ← inside here, node is PID 1
  /proc/34512/ns/net   → net:[4026532def]     ← its own eth0, lo, routing table
  /proc/34512/ns/mnt   → mnt:[4026532ghi]     ← its own / (the overlay)
  /proc/34512/ns/uts, ipc, cgroup, user

CGROUP (limits + accounting; cgroup v2 unified hierarchy):
  /sys/fs/cgroup/system.slice/docker-<CONTAINER_ID>.scope/
    ├── memory.max        ← e.g. "536870912" if --memory 512m, else "max"
    ├── memory.current    ← live bytes used
    ├── cpu.max           ← e.g. "50000 100000" if --cpus 0.5
    ├── pids.current      ← how many processes inside
    └── cgroup.procs      ← host PIDs of every process in the container

FILESYSTEM (overlay, from Topic 04):
  /var/lib/docker/overlay2/<id>/merged   ← this IS the container's "/"
    lowerdir = image layers (read-only, shared)
    upperdir = writable layer (this container only)

NETWORK (default bridge):
  host: docker0 bridge (172.17.0.1) + vethXXXX@ifYY  ← one end of the virtual cable
  container netns: eth0 (172.17.0.2) + lo             ← the other end
  iptables nat table: DNAT rule mapping host :8080 → 172.17.0.2:3000

METADATA:
  /var/lib/docker/containers/<CONTAINER_ID>/
    ├── config.v2.json     ← full container config (env, cmd, mounts)
    ├── hostconfig.json    ← ports, restart policy, resource limits
    ├── hostname, hosts, resolv.conf   ← bind-mounted into the container as /etc/*
    └── <CONTAINER_ID>-json.log        ← where `docker logs` reads from (json-file driver)
```

Nothing here is a virtual machine. It is one `node` process, a few directories, some kernel bookkeeping, and two network interfaces.

---

## How it works — step by step

Full trace of `docker run -d -p 8080:3000 --name orders-api orders-api:latest`:

1. **CLI → daemon.** `docker` CLI serializes your flags into a JSON request and POSTs to the daemon over the Unix socket `/var/run/docker.sock` (`POST /containers/create`, then `/start`). (Topic 03.)
2. **Image resolve.** dockerd finds `orders-api:latest` in `repositories.json`, reads its manifest and config (Topic 04): default `CMD ["node","server.js"]`, env, workdir, the ordered layer list.
3. **Create container record.** dockerd writes `/var/lib/docker/containers/<id>/` with `config.v2.json`, generates `/etc/hostname`, `/etc/hosts`, `/etc/resolv.conf` to bind-mount in. State becomes **Created** (not running yet).
4. **Snapshot the rootfs.** containerd asks the overlay2 snapshotter for a fresh **writable upperdir**, and the kernel mounts overlay at `…/merged` (the mount command from Topic 04). This directory is now the container's future `/`.
5. **cgroup setup.** dockerd/runc creates `/sys/fs/cgroup/system.slice/docker-<id>.scope/` and writes any limits: `memory.max`, `cpu.max`, `pids.max`. Even with no `--memory`, the cgroup exists for *accounting*.
6. **Spawn shim + runc.** containerd starts a `containerd-shim-runc-v2`, which execs **runc** with an OCI `config.json` (the runtime spec: namespaces to create, mounts, the process to exec).
7. **Create namespaces.** runc calls `clone()`/`unshare()` with the `CLONE_NEW*` flags. A new process is born inside fresh PID/net/mnt/uts/ipc namespaces. It is **PID 1** in its PID namespace, even though the host sees it as PID 34512.
8. **Wire the network (CNI/libnetwork).**
   a. Create a **veth pair** — two ends of a virtual cable: `vethXXXX` stays on the host, `eth0` is moved *into* the container's net namespace (`ip link set eth0 netns <pid>`).
   b. Attach the host end to the `docker0` bridge.
   c. Assign the container `172.17.0.2/16`, set default route via `172.17.0.1` (docker0).
   d. Because of `-p 8080:3000`, install an **iptables DNAT** rule so packets hitting the host's port 8080 get rewritten to `172.17.0.2:3000`, plus a MASQUERADE rule for return traffic:
   ```
   -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:3000
   ```
9. **Pivot root + drop privileges.** runc `pivot_root()`s into `…/merged` (now `/`), mounts `/proc`, `/sys`, `/dev` fresh inside, applies the **seccomp** profile (blocks dangerous syscalls), drops Linux **capabilities** (keeps ~14 of ~40), sets the user (e.g. `USER node`), then `execve("node", ["node","server.js"])`. **`node` is now PID 1.** runc exits. The shim stays as the parent/babysitter.
10. **Running.** dockerd marks state **Running** with the host PID. Your app's stdout/stderr are piped by the shim to `…/<id>-json.log`. `docker run -d` prints the full container ID and returns your shell. The container is alive exactly as long as PID 1 (`node`) keeps running.
11. **Someone requests `curl localhost:8080`.** Packet hits host port 8080 → iptables DNAT rewrites dest to `172.17.0.2:3000` → routed across `docker0` → into the veth → out `eth0` inside the container → `node` accepts on `:3000` → replies, MASQUERADE rewrites the source on the way back. You see `orders ok`.
12. **Stop (`docker stop orders-api`).** dockerd sends **SIGTERM** to PID 1. It waits **10 seconds** (default grace). If PID 1 is still alive, it sends **SIGKILL**. State → **Exited (code)**. Overlay upperdir and namespaces are torn down on `docker rm`; the metadata dir is removed too.

---

## Exact syntax breakdown

**The launch command:**

```
docker run -d -p 8080:3000 -e NODE_ENV=production --name orders-api orders-api:latest
       │    │  │            │                      │              │
       │    │  │            │                      │              └ IMAGE[:tag] to run (Topic 04)
       │    │  │            │                      └ human name (else Docker random-generates one)
       │    │  │            └ set an env var inside the container (repeatable)
       │    │  └ PUBLISH port: hostPort:containerPort → installs the iptables DNAT rule (step 8d)
       │    └ DETACHED: run in background, print the ID, return your shell. PID 1's stdio → json log.
       └ subcommand: create + start a container from an image
```

`-p` deep detail:

```
-p 8080:3000
   │    │
   │    └ containerPort — the port your node app listens on INSIDE (server.listen(3000))
   └ hostPort — the port on the HOST that gets DNAT'd to it. `curl host:8080` reaches the app.

-p 127.0.0.1:8080:3000   ← bind only to localhost on the host (not 0.0.0.0) — safer
-p 3000                  ← publish to a RANDOM host port (see it with `docker port`)
```

**Interactive/TTY flags:**

```
docker run -it --rm alpine sh
           ││  │
           ││  └ --rm: auto-delete the container the moment it exits (no leftover Exited junk)
           │└ -t: allocate a pseudo-TTY (so prompts, colors, line editing work)
           └ -i: keep STDIN open & connected (so what you type reaches the process)
   → -i without -t: piping works but no pretty terminal.  -t without -i: looks like a terminal but
     you can't send input. You almost always want them together as `-it` for a shell.
```

**Lifecycle commands:**

```
docker logs -f --tail 50 orders-api        docker exec -it orders-api sh
            │  │                                       │  │           │
            │  └ show only last 50 lines               │  │           └ command to run INSIDE
            └ -f: follow (stream new lines live)       │  └ TTY (interactive shell)
                                                       └ -i: keep STDIN open
   `logs` reads the json-file on disk.        `exec` joins the RUNNING container's namespaces
   `attach` connects to PID 1's live stdio.   and starts a NEW process (a 2nd PID) inside.
```

`attach` vs `exec` — the trap:

```
docker attach orders-api   → connects your terminal to PID 1's stdin/stdout.
                             Ctrl-C here sends SIGINT to PID 1 → may KILL your app!
                             Detach safely with the escape sequence:  Ctrl-P Ctrl-Q
docker exec -it orders-api sh → starts a brand-new shell (a 2nd process) inside.
                             Exiting it does NOT stop the container. This is the safe debug tool.
```

---

## Example 1 — basic

Watch every lifecycle state of a real container.

```bash
# 1) Detached: starts in background, prints the full 64-char ID, returns your shell.
docker run -d -p 8080:3000 --name orders-api orders-api:latest
# 3f9a1c...   ← that's the container ID

# 2) Confirm it's Running and see the port mapping (the iptables DNAT rule):
docker ps
# CONTAINER ID   IMAGE               STATUS         PORTS                    NAMES
# 3f9a1c8b2d4e   orders-api:latest   Up 4 seconds   0.0.0.0:8080->3000/tcp   orders-api
#                                                   │
#                                                   └ host 8080 → container 3000 (step 8d)

# 3) Hit it — the packet path from step 11:
curl localhost:8080          # → orders ok

# 4) See the app's logs (reads the json-file on disk, step 10):
docker logs orders-api
docker logs -f --tail 20 orders-api   # stream live; Ctrl-C stops FOLLOWING, not the container

# 5) Get a shell INSIDE (joins the container's namespaces — step: a 2nd process):
docker exec -it orders-api sh
  / # ps aux          # ← node is PID 1 here! runc/shim are invisible (different PID namespace)
  / # cat /proc/1/cmdline | tr '\0' ' '   # node server.js
  / # hostname        # the container ID (own UTS namespace)
  / # exit            # leaving the shell does NOT stop the container

# 6) Pause / unpause (freezes all processes via the cgroup freezer):
docker pause orders-api      # state: Paused — processes frozen, still in memory
docker unpause orders-api

# 7) Stop gracefully (SIGTERM, wait 10s, then SIGKILL — step 12):
docker stop orders-api       # state → Exited (0) if it handled SIGTERM
docker ps -a                 # -a shows stopped containers too
# STATUS: Exited (0) 3 seconds ago

# 8) Start it again (reuses the same writable layer + config), then remove:
docker start orders-api
docker rm -f orders-api      # -f = stop then remove; frees the upperdir + metadata dir
```

---

## Example 2 — production scenario

**"We deploy `orders-api`, and every rollout drops ~200 in-flight requests. Shutdown takes the full 10 seconds and logs show `SIGKILL`."**

Your `server.js`:

```js
const http = require('http');
const server = http.createServer(handler);
server.listen(3000);
// ...no SIGTERM handler!
```

Your Dockerfile:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci --omit=dev
CMD npm start        # ← shell form! (the real bug)
```

Two PID-1 bugs stacked:

**Bug A — `CMD npm start` uses the shell form.** The shell form runs your process as `/bin/sh -c "npm start"`. So **PID 1 is `sh`**, and `node` is PID 2, a *child* of sh. When `docker stop` sends SIGTERM, it goes to PID 1 = `sh`. `sh` does not forward signals to children. `node` never hears SIGTERM. After 10s it gets SIGKILLed mid-request. Prove it:

```bash
docker exec orders-api ps aux
# PID   COMMAND
#   1   /bin/sh -c npm start     ← SIGTERM lands HERE and stops
#   7   node server.js           ← never receives it
```

**Bug B — even as PID 1, `node` has no SIGTERM handler.** Node's default behavior for SIGTERM as PID 1 is... nothing special, and it won't drain connections.

The fix — make `node` PID 1 with the **exec form**, and handle the signal:

```dockerfile
CMD ["node", "server.js"]   # exec form → node IS PID 1, receives SIGTERM directly
```

```js
const server = http.createServer(handler);
server.listen(3000);

process.on('SIGTERM', () => {
  console.log('SIGTERM received, draining connections…');
  server.close(() => {           // stop accepting new; finish in-flight
    console.log('drained, exiting cleanly');
    process.exit(0);
  });
  // safety net so we never hit the 10s SIGKILL:
  setTimeout(() => process.exit(1), 9000).unref();
});
```

Now `docker stop` → SIGTERM reaches `node` directly → `server.close()` finishes the ~200 in-flight requests → clean `Exited (0)` in well under a second. Zero dropped requests on rollout.

Bonus for images that *must* spawn children (e.g. a wrapper): add an init with `docker run --init …`, which inserts a tiny `tini` as PID 1 to reap zombies and forward signals.

---

## Common mistakes

**Mistake 1 — App runs as PID 1 via shell form and ignores signals.**
```dockerfile
CMD npm start     # WRONG
```
*Symptom:* `docker stop` always takes 10s; container dies with `Exited (137)` (137 = 128 + 9 = SIGKILL).
*Root cause:* PID 1 is `sh`, which doesn't forward SIGTERM; your app never gets it.
*Right:* `CMD ["node","server.js"]` (exec form) and handle SIGTERM. Or `docker run --init`.

**Mistake 2 — Container "won't stay running" / `Exited (0)` immediately.**
```bash
docker run -d --name w orders-api sh -c 'echo starting'
docker ps       # empty!  docker ps -a → Exited (0)
```
*Symptom:* nothing in `docker ps`.
*Root cause:* a container lives exactly as long as its **PID 1**. `echo` finished, so PID 1 exited, so the container exited. There is no such thing as a container "running in the background with nothing to do."
*Right:* PID 1 must be a long-lived foreground process (`node server.js`, `nginx -g 'daemon off;'`). Never background your main process inside the container.

**Mistake 3 — Using `docker attach` to check on the app, then Ctrl-C kills it.**
```bash
docker attach orders-api
^C     # oops — SIGINT went to PID 1 = node → app stopped in production
```
*Symptom:* checking logs accidentally takes down the service.
*Root cause:* `attach` wires your terminal to PID 1's stdio; Ctrl-C = SIGINT to PID 1.
*Right:* use `docker logs -f` to watch output, and `docker exec -it … sh` to poke around. If you must attach, detach with **Ctrl-P Ctrl-Q**, never Ctrl-C.

**Mistake 4 — `port is already allocated`.**
```
docker: Error response from daemon: driver failed programming external connectivity
on endpoint orders-api: Bind for 0.0.0.0:8080 failed: port is already allocated.
```
*Symptom:* run fails.
*Root cause:* another process/container already holds host port 8080; Docker can't install the DNAT/bind for it.
*Right:* pick another host port (`-p 8081:3000`), or `docker ps` to find and stop the holder, or `lsof -i :8080`.

**Mistake 5 — Publishing the port but the app binds to `127.0.0.1` inside.**
```js
server.listen(3000, '127.0.0.1');   // WRONG for containers
```
*Symptom:* `docker ps` shows the mapping, but `curl localhost:8080` gives `connection reset`.
*Root cause:* inside the container, `127.0.0.1` is the container's own loopback. The DNAT'd packet arrives on `eth0` (172.17.0.2), not loopback, so nothing is listening where it lands.
*Right:* bind to all interfaces: `server.listen(3000, '0.0.0.0')` (or just `server.listen(3000)`).

---

## Hands-on proof

See the namespaces, cgroup, veth, and PID 1 for a live container.

> On Docker Desktop, prefix host-level inspection by entering the VM:
> `docker run -it --rm --privileged --pid=host justincormack/nsenter1`. On Linux, use `sudo`.

```bash
# Start the running example:
docker run -d -p 8080:3000 --name orders-api orders-api:latest

# 1) node is PID 1 INSIDE, but a normal PID on the host:
docker exec orders-api sh -c 'echo inside-pid=$$; ps -o pid,comm'   # inside: node is PID 1
HOSTPID=$(docker inspect orders-api --format '{{.State.Pid}}')
echo "host sees node as PID $HOSTPID"
ps -o pid,ppid,comm -p $HOSTPID       # parent is containerd-shim, NOT dockerd

# 2) The container's namespaces (compare inode numbers to the host's — they differ):
ls -l /proc/$HOSTPID/ns/               # pid, net, mnt, uts, ipc, cgroup, user
ls -l /proc/1/ns/net                   # host's net namespace — different inode = truly isolated

# 3) The cgroup — resource accounting/limits (cgroup v2):
CID=$(docker inspect orders-api --format '{{.Id}}')
cat /sys/fs/cgroup/system.slice/docker-$CID.scope/memory.current   # live memory bytes
cat /sys/fs/cgroup/system.slice/docker-$CID.scope/pids.current     # process count
cat /sys/fs/cgroup/system.slice/docker-$CID.scope/cgroup.procs     # host PIDs inside

# 4) The overlay root filesystem (from Topic 04):
docker inspect orders-api --format '{{.GraphDriver.Data.MergedDir}}'   # this is the container's /

# 5) The network wiring — veth pair + bridge + DNAT rule:
docker exec orders-api ip addr show eth0     # 172.17.0.2 inside
ip link | grep veth                          # the host end of the cable
sudo iptables -t nat -L DOCKER -n | grep 8080   # the DNAT rule for -p 8080:3000

# 6) The log file `docker logs` actually reads:
docker inspect orders-api --format '{{.LogPath}}'   # …/<id>-json.log
docker logs orders-api

# 7) Watch the SIGTERM → SIGKILL grace window live:
time docker stop orders-api    # ~0s if app handles SIGTERM; ~10s if it ignores it
docker inspect orders-api --format '{{.State.Status}} exit={{.State.ExitCode}}'

# cleanup
docker rm -f orders-api
```

You have now seen with your own eyes: your app is PID 1 in an isolated PID namespace, capped by a cgroup, rooted in an overlay mount, and wired to the host through a veth pair and an iptables DNAT rule.

---

## Practice exercises

### Exercise 1 — easy
Run `docker run -it --rm node:20-alpine sh`. Inside, run `ps aux`, `hostname`, `cat /etc/hostname`, and `ip addr`. Note that you are PID 1 (well, `sh` is) and the hostname is the container ID. Exit and confirm `--rm` left no container behind with `docker ps -a`.

### Exercise 2 — medium
Start `orders-api` detached with `-p 8080:3000`. Using only `docker logs -f` in one terminal and `docker exec -it … sh` in another, generate some requests with `curl` and watch them appear in the logs — without ever using `docker attach`. Then find the container's host PID via `docker inspect … .State.Pid` and confirm the same process appears under `/sys/fs/cgroup/.../cgroup.procs`.

### Exercise 3 — hard (production simulation)
Reproduce the dropped-requests bug: write `server.js` with a slow 3-second request handler and **no** SIGTERM handler, and a Dockerfile using `CMD npm start` (shell form). Start it, fire a slow `curl` in the background, then `docker stop` it and observe `Exited (137)` and the killed request. Now fix it (exec-form CMD + `server.close()` on SIGTERM), repeat, and prove with `time docker stop` that shutdown is fast and the in-flight request completes with `Exited (0)`. Explain, referencing PID 1 and signal forwarding, exactly why each version behaves as it does.

---

## Mental model checkpoint

Answer from memory:

1. There is no `create_container()` syscall. Which kernel primitives does runc combine to make a container, and which single call creates the namespaces?
2. A container lives exactly as long as *what*? Why does `docker run ... echo hi` exit immediately?
3. Inside the container your app is PID 1. What is special about PID 1's handling of SIGTERM, and why does `CMD npm start` (shell form) break graceful shutdown?
4. Trace a packet from `curl localhost:8080` to your `node` app and back. Where does the iptables DNAT rule come from?
5. What is the difference between `docker attach` and `docker exec`, and which one can accidentally kill your app with Ctrl-C?
6. Where does `docker logs` physically read from?
7. What happens during the 10-second window of `docker stop`, and what exit code means "SIGKILLed"?

---

## Quick reference card

| Command / flag | What it does | Key detail |
|---|---|---|
| `docker run -d` | Create + start, detached | Prints ID, returns shell; stdio → json log |
| `docker run -it` | Interactive + TTY | `-i` keeps STDIN, `-t` allocates a pseudo-TTY |
| `docker run --rm` | Auto-remove on exit | No leftover `Exited` container |
| `-p host:container` | Publish port | Installs iptables DNAT rule |
| `--name` | Human name | Else Docker random-generates |
| `--init` | Insert tini as PID 1 | Reaps zombies, forwards signals |
| `docker ps` / `ps -a` | List running / all | `-a` shows Exited too |
| `docker logs -f` | Stream stdout/stderr | Reads `<id>-json.log` on disk |
| `docker exec -it … sh` | New process inside | Safe debug; joins namespaces |
| `docker attach` | Connect to PID 1 stdio | Ctrl-C = SIGINT to app! Detach: Ctrl-P Ctrl-Q |
| `docker stop` | SIGTERM, wait 10s, SIGKILL | `Exited (137)` = SIGKILLed |
| `docker pause` | Freeze via cgroup freezer | Processes stay in memory |
| `docker start` | Restart a stopped one | Reuses upperdir + config |
| Lifecycle states | Created→Running→Paused→Exited→(Dead) | Alive = PID 1 alive |

---

## When would I use this at work?

1. **Achieving zero-downtime deploys.** Rollouts drop requests because containers get SIGKILLed. You diagnose PID 1 / signal forwarding (shell vs exec form), add a SIGTERM drain handler, and rollouts become clean — directly enabling Kubernetes rolling updates later (Topic 46).
2. **Debugging a misbehaving container in prod.** A pod is "running" but returning errors. You `docker exec -it` (or `kubectl exec`) into its namespaces to inspect env vars, hit `localhost:3000` from inside, and read `/proc/1/cmdline` — pinpointing config vs code vs network issues without redeploying.
3. **Fixing "can't reach the service."** `docker ps` shows the port mapped but curl fails. You trace it: app bound to `127.0.0.1` instead of `0.0.0.0`, or a missing `-p`, or a port already allocated — using the exact packet-path model from this topic.

---

## Connected topics

- **Study before:** Topic 02 — Linux Foundations (namespaces, cgroups, overlayfs — the primitives). Topic 03 — Docker Architecture (CLI → daemon → containerd → shim → runc). Topic 04 — Images and Layers (what gets mounted as the root filesystem).
- **Study after:** Topic 07 — CMD vs ENTRYPOINT (exec vs shell form, the PID 1 story in full). Topic 12 — docker run in Depth (every flag). Topic 13 — Networking in Docker (bridge, veth, iptables in detail). Topic 16 — Container Lifecycle (every state at the kernel level). Topic 17 — Logging (log drivers beyond json-file).
