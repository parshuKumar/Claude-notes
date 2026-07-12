# 33 — Environment & Process Management for Node.js

## ELI5 — The Simple Analogy

Think about the difference between **holding a flashlight** and **installing a ceiling light**.

When you run `node server.js` in your SSH terminal, you are *holding a flashlight*. As long as your arm is up and you're standing there, there's light. The moment you get tired, put your arm down, walk away, or the batteries die — darkness. Nobody replaces the batteries. Nobody turns it back on after a power cut. It only worked because *you* were personally standing there holding it.

A production deployment is a *ceiling light wired into the building*. It turns on by itself when the power comes back after an outage. If a bulb blows, the fixture is designed to be relit automatically. It doesn't care whether you're in the room, asleep, or on vacation. It's part of the building's infrastructure, not a thing a person is holding up.

This whole topic is about wiring your Node app into the building — so it survives you logging out, restarts itself when it crashes, comes back after a reboot, and shuts down cleanly instead of dropping everyone's requests on the floor.

---

## Where This Lives in the Linux Stack

```
Hardware
  └── KERNEL
       │   • sends SIGHUP when your terminal closes (topic 17) — kills a naive `node`
       │   • the OOM killer sends SIGKILL when RAM runs out (topic 01/18) — exit 137
       │   • cgroups enforce the container's memory limit
       │   • schedules ONE Node thread onto ONE core (topic 18)
       │
       └── System Calls (fork/execve, setrlimit for LimitNOFILE, kill for signals)
            │
            └── init system: systemd (PID 1 on the host) ◀◀◀ THIS TOPIC
            │     • supervises the service, restarts it, applies limits, captures logs
            │
            └── (in a container) Node is PID 1 itself ◀◀◀ THIS TOPIC's big gotcha
            │
            └── Reverse proxy: nginx ◀◀◀ THIS TOPIC
                  • TLS, static files, buffering, rate limiting, port 80/443 as root
                  └── Node app on :3000 as an unprivileged user (topic 32)
```

**This topic ties the whole curriculum together for one reader: the Node dev who needs their app to run like production infrastructure, not like a command they typed.**

---

## What Is This?

Production process management for Node is the set of decisions and configuration that turns "a program you can run" into "a service the machine runs for you": a supervisor that keeps it alive, an environment that configures it safely, memory limits it understands, a graceful shutdown so deploys don't drop requests, and a reverse proxy in front for TLS and protection. It is the operational half of shipping a Node app — the half that isn't in your `git` repo.

---

## Why `node server.js` in an SSH Session Is NOT a Deployment

Enumerate exactly why the flashlight fails. Every one of these is a real outage:

```
YOU TYPE:  node server.js     (in your SSH shell)

FAILURE MODE                         WHY (mechanism)                       TOPIC
──────────────────────────────────  ────────────────────────────────────  ─────
Dies when SSH drops                  terminal closes → kernel sends SIGHUP  17
                                     to the foreground process group → node
                                     gets no handler → default = terminate.
                                     Your app dies the second your laptop
                                     sleeps or the wifi blips.
Doesn't restart on crash             an uncaught exception exits the process 16/17
                                     with code 1. Nothing relaunches it.
                                     It's just... gone until you notice.
Doesn't start on reboot              the box reboots (kernel update, crash,  22
                                     cloud maintenance). Nothing runs your
                                     command again. Site down till you SSH in.
No log management                    stdout scrolls past in your terminal    23
                                     and is LOST when you disconnect. No
                                     history, no rotation, no search.
Runs as YOUR user                    the process has your uid — your keys,   32
                                     your sudo, your everything. An RCE owns
                                     your account (or root, if you sudo'd).
No resource limits                   no fd cap, no memory cap. A leak eats    18/19
                                     the box; a socket flood hits EMFILE and
                                     takes down everything, not just the app.
```

Any ONE of these is disqualifying. `node server.js` is for `localhost` development. Production needs a supervisor.

| If you don't understand this... | This will happen... |
|---|---|
| SIGHUP on terminal close | Your app dies every time your SSH session ends — "it was up when I left!" |
| No restart-on-crash | One uncaught exception and the API is down until a human notices |
| No start-on-boot | A cloud-maintenance reboot at 4am takes you offline until morning |
| Running as your user | An app RCE inherits your privileges instead of a locked-down service account |
| No memory limit / heap trap | Silent OOM kills (exit 137) you can't explain, or V8 heap-limit crashes |
| PID-1 signal handling in Docker | Every deploy hard-kills in-flight requests; graceful shutdown never runs |

---

## The Requirements of a Real Deployment — The Checklist

A production process management setup must satisfy ALL of these. Keep this list; everything below is how to tick each box.

```
[ ] SURVIVES DISCONNECT     keeps running after you close SSH
[ ] RESTARTS ON CRASH       relaunches automatically when it exits abnormally
[ ] STARTS ON BOOT          comes up on its own after a reboot
[ ] DURABLE LOGS            output goes somewhere persistent, searchable, rotated
[ ] UNPRIVILEGED USER       runs as a locked-down service account (topic 32)
[ ] RESOURCE LIMITS         fd limit (EMFILE) and memory limit are set explicitly
[ ] GRACEFUL SHUTDOWN       drains in-flight requests before exiting on deploy
[ ] OBSERVABLE              health checks, structured logs, metrics you can read
```

---

## The Options, Compared Honestly

There are four realistic ways to supervise a Node app. Be opinionated.

| | systemd | PM2 | Docker + restart policy | Kubernetes |
|---|---|---|---|---|
| Extra dependency | none (built in) | yes (npm global) | Docker | a whole cluster |
| Survives disconnect | ✓ | ✓ | ✓ | ✓ |
| Restart on crash | ✓ `Restart=` | ✓ | ✓ `--restart` | ✓ |
| Start on boot | ✓ `enable` | via generated systemd unit | ✓ (with daemon) | ✓ |
| Resource limits | ✓ cgroups | ✗ (not really) | ✓ cgroups | ✓ cgroups |
| Log capture | ✓ journald (topic 23) | ✓ own files + rotation | ✓ docker logs | ✓ cluster logging |
| Cluster mode | multiple units | ✓ built-in | N containers | N pods |
| Metrics/DX | `systemctl status` | nice dashboard | basic | rich |
| The catch | learn unit files | **it's another supervisor you must keep alive; `pm2 startup` just generates a systemd unit — now you have TWO layers** | you still need a supervisor for the daemon | massive overkill for one VM |

**The opinionated recommendation:**

- **On a single VM: use systemd.** It's already PID 1, already supervising everything else, gives you cgroup limits and journald for free, and adds zero dependencies. PM2's headline features (keep-alive, boot start) are things systemd already does — and `pm2 startup` literally *generates a systemd unit* to keep PM2 alive, so you end up with systemd supervising PM2 supervising node: two layers where one would do. Use PM2 only if you specifically want its cluster/reload DX and accept the extra moving part.
- **In containers: let the orchestrator supervise. Do NOT run PM2 or nodemon inside a container.** One process per container. The orchestrator (Docker's restart policy, Kubernetes' kubelet) *is* your supervisor — that's its entire job. Running a supervisor *inside* the container defeats the platform's health checks and restart logic: the container looks "healthy" because PM2 is alive even when your app is wedged, and PM2 restarting the app internally hides crashes from Kubernetes, which should be recreating the pod. **One process, exec-form CMD, let the platform watch it.**

---

## A Complete, Production-Grade systemd Unit

This builds on topic 22 — it does not merely repeat it. Read every directive's comment.

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Node.js API
After=network-online.target postgresql.service   # start after network + DB are up
Wants=network-online.target

[Service]
Type=simple                       # the process we exec IS the service (no forking)
User=nodeapp                      # run as the unprivileged service user (topic 32)
Group=nodeapp
WorkingDirectory=/srv/app

# ── ABSOLUTE PATH to node — the NVM trap (topic 14/21/22) ─────────────────
# systemd does NOT source your shell profile. Your interactive `node` comes from
# ~/.nvm/.../bin/node via a line in .bashrc that systemd never reads. A bare
# `node` here fails with status=203/EXEC "No such file or directory".
# Use the absolute path (a system-wide install, e.g. /usr/bin/node):
ExecStart=/usr/bin/node /srv/app/dist/server.js

# ── Environment / config (topic 14) ──────────────────────────────────────
Environment=NODE_ENV=production   # inline vars for non-secrets
EnvironmentFile=/etc/myapp/env    # secrets file, 640 root:nodeapp (topic 32)

# ── Restart policy ────────────────────────────────────────────────────────
Restart=always                    # relaunch on ANY exit (crash, clean, killed)
RestartSec=5                      # wait 5s between restarts (don't hammer the CPU/DB)

# ── Crash-loop wall ───────────────────────────────────────────────────────
# If it fails 5 times within 60s, systemd STOPS trying and marks it 'failed'
# (so a broken deploy doesn't spin forever). Clear the counter with:
#   systemctl reset-failed myapp
StartLimitBurst=5
StartLimitIntervalSec=60

# ── Resource limits ───────────────────────────────────────────────────────
LimitNOFILE=65535                 # max open file descriptors — the EMFILE fix (topic 19).
                                  # Default 1024 is far too low for a busy server: every
                                  # socket is an fd; you hit "EMFILE: too many open files"
                                  # under load and the app stops accepting connections.
MemoryMax=1G                      # cgroup HARD cap: exceed it → the CGROUP OOM killer
                                  # SIGKILLs the process (exit 137). See the memory section:
                                  # this must be coordinated with V8's --max-old-space-size.

# ── Graceful shutdown ─────────────────────────────────────────────────────
KillSignal=SIGTERM                # send SIGTERM first (your handler runs) — the default
TimeoutStopSec=30                 # give the app 30s to drain; then systemd sends SIGKILL.
                                  # Must be >= your in-flight request timeout. Pair with
                                  # the graceful-shutdown code below.

# ── Logging (topic 23) ────────────────────────────────────────────────────
StandardOutput=journal            # stdout → journald (journalctl -u myapp)
StandardError=journal             # stderr → journald too

# ── Security hardening (topic 32 continues here) ──────────────────────────
NoNewPrivileges=true              # process can NEVER gain privileges via setuid binaries
PrivateTmp=true                   # private /tmp — can't see or be seen in the real /tmp
ProtectSystem=strict              # the ENTIRE filesystem is read-only to this service...
ReadWritePaths=/srv/app/uploads   # ...EXCEPT these paths. App can write ONLY uploads.
ProtectHome=true                  # /home is invisible to the service
UMask=0027                        # new files not world-readable (topic 05/32)

[Install]
WantedBy=multi-user.target        # `systemctl enable` → starts on boot at this target
```

```bash
sudo systemctl daemon-reload          # re-read unit files after editing
sudo systemctl enable --now myapp     # start now AND on every boot
systemctl status myapp                 # is it running? recent logs? memory used?
journalctl -u myapp -f                 # follow the logs (topic 23)
```

That single file ticks: survives disconnect (systemd owns it), restarts on crash (`Restart=always`), starts on boot (`enable`), durable logs (journald), unprivileged (`User=nodeapp`), fd + memory limits, and graceful shutdown (SIGTERM + TimeoutStopSec). Eight boxes, one file.

---

## Environment & Config (topic 14 callback)

The 12-factor rule: **store config in the environment, not in the code**. The same built artifact runs in staging and prod; only the environment differs. Never hard-code the DB URL or a feature flag.

### NODE_ENV=production — what it ACTUALLY changes

`NODE_ENV` is just an environment variable, but a lot of the ecosystem branches on it:

```
NODE_ENV=production actually changes:
  • Express: skips per-request view template re-compilation (caches views),
    and does NOT send stack traces in error responses (dev leaks them to users).
  • npm ci --omit=dev / npm install --production: skips devDependencies
    (smaller install, fewer attack-surface packages in the image).
  • React and MANY libraries: strip dev-only warnings/invariants, enable fast paths.
  • Countless `if (process.env.NODE_ENV !== 'production')` dev-only code paths turn off.
```

**The classic bug: deploying with `NODE_ENV` unset or `=development`.** It's silent — the app *works* — but you pay for it twice: measurably worse performance (Express recompiles views every request, dev code paths run), and a *security leak* — verbose error pages send full stack traces (file paths, library versions, sometimes query fragments) straight to end users and attackers. Always set it explicitly in the unit: `Environment=NODE_ENV=production`. Verify it in the running process:

```bash
# PROVE IT: what NODE_ENV does the LIVE process actually see?
sudo cat /proc/$(pgrep -f 'node .*server.js'|head -1)/environ | tr '\0' '\n' | grep NODE_ENV
# NODE_ENV=production   ← if this is empty or 'development', you have the silent bug
```

`PORT` (which port to bind — commonly 3000), `DATABASE_URL`, `REDIS_URL`, etc. all come from the environment the same way.

### Secrets — where they must NOT go

```
SECRETS (DB password, JWT secret, API keys) must NOT be:

  ✗ in the git repo            → history is forever; every clone has them.
  ✗ baked into a Docker image  → `ENV SECRET=...` or `COPY .env` writes them into a
                                 LAYER. Layers are immutable and shipped. ANYONE who
                                 pulls the image runs `docker history --no-trunc IMG`
                                 and reads the secret out of the layer metadata. Forever.
  ✗ on the command line        → `node server.js --db-pass=hunter2` shows up in
                                 `ps aux` (visible to every user) AND in
                                 /proc/PID/cmdline. Same for putting them in argv.

SECRETS SHOULD be:
  ✓ an EnvironmentFile, chmod 640 root:nodeapp (topic 32) — systemd injects them,
    they never touch disk in the repo/image, only root writes them.
  ✓ a secrets manager (Vault, AWS Secrets Manager, SOPS-encrypted file) fetched at
    boot into the environment.
  ✓ .env + dotenv → DEV ONLY. Never commit .env; add it to .gitignore.
```

Even the *environment* is semi-visible: `/proc/PID/environ` is readable by the process owner and root (topic 14/16). That's fine when the app runs as a locked-down `nodeapp` and root is trusted — but it's why command-line args (world-visible via `ps`) are strictly worse than env vars.

---

## Memory — A Node-Specific Deep Dive You NEED

This is where Node ops most often goes wrong. There are **two completely different "out of memory" failures**, with **opposite fixes**. Confusing them will make an outage worse.

### Failure A — V8 heap limit (Node hitting ITS OWN cap)

V8 (Node's JS engine) caps the "old space" heap *independently of the machine's RAM* — historically ~1.5–2 GB on 64-bit unless you raise it. Hit that cap and you get:

```
<--- Last few GCs --->
FATAL ERROR: Ineffective mark-compacts near heap limit allocation failed -
             JavaScript heap out of memory
 1: 0x... node::Abort() ...
```

This is **V8 giving up**, not the OS. The process aborts *itself*. The machine may have plenty of free RAM.

Fix: either your app has a **memory leak** (find and fix it — see Observability), or it legitimately needs a bigger heap:

```bash
# Raise V8's old-space cap to 3 GB (via NODE_OPTIONS so it applies everywhere):
Environment=NODE_OPTIONS=--max-old-space-size=3072
# (3072 MB. Set it BELOW the machine/container RAM AND below MemoryMax.)
```

### Failure B — kernel/cgroup OOM kill (out of REAL memory)

The kernel's OOM killer (topic 01) or the cgroup memory limiter SIGKILLs your process because *actual* memory ran out. The signature is different:

```
$ journalctl -u myapp | tail
myapp.service: A process of this unit has been killed by the OOM killer.
myapp.service: Main process exited, code=killed, status=9/KILL
myapp.service: Failed with result 'oom-kill'.

$ dmesg | grep -i 'killed process'
Out of memory: Killed process 4127 (node) total-vm:... anon-rss:...
```

Exit **137** (128 + signal 9 = SIGKILL). No V8 "FATAL ERROR" — the process is killed from *outside*, mid-execution, no chance to log a graceful message.

Fix is the **OPPOSITE** of Failure A: you are out of *real* memory. Raising `--max-old-space-size` makes it **WORSE** (you let V8 grab even more RAM before the kernel kills it). The right moves: lower memory pressure (fix leak, reduce concurrency, add RAM, raise `MemoryMax`), not raise the heap.

```
                       "Out of memory"
                    ┌────────┴────────┐
      "FATAL ERROR: ...JS heap        exit 137 / status=9/KILL
       out of memory" in YOUR logs    "killed by OOM killer" in journal/dmesg
                    │                          │
              V8 hit ITS cap            KERNEL ran out of REAL RAM
                    │                          │
       FIX: fix leak OR raise           FIX: fix leak / add RAM / raise MemoryMax.
            --max-old-space-size        RAISING the heap makes it WORSE.
```

Learn to read which one you have *before* you touch a flag.

### The container trap (topic 18 callback) — the one that bites everyone

**V8 and libuv size themselves from the HOST's CPU/RAM as seen in `/proc`, not from the cgroup limit.** So in a 512 MB container, older/unconfigured Node reads the *host's* `/proc/meminfo` (say 64 GB), thinks it has 64 GB, sets a multi-GB heap target, and **GCs lazily** — it sees no reason to collect aggressively. Then it sails past 512 MB and the **cgroup OOM killer SIGKILLs it (exit 137)** while V8 still thinks everything's fine.

```
Container MemoryLimit = 512 MB
   │
   ├── V8 reads HOST /proc/meminfo → "I have 64 GB" → heap target ~2 GB, lazy GC
   │
   └── process grows past 512 MB → cgroup OOM kill → exit 137
       (V8 never warned you — it thought it had 63 GB to spare)
```

Fixes:
1. **Set `--max-old-space-size` explicitly to ~75% of the container limit.** For a 512 MB container: `NODE_OPTIONS=--max-old-space-size=384`. This tells V8 the real ceiling so it GCs before the cgroup kills it.
2. **Use a cgroup-aware Node** (recent versions detect the cgroup limit better) and/or set the container's memory limit and let Node's newer heuristics read it — but don't rely on this; setting the flag explicitly is the reliable move.

### UV_THREADPOOL_SIZE — the hidden 4-thread bottleneck

Node's event loop is single-threaded, but libuv has a **thread pool (default size 4)** for work that can't be done async at the OS level: **filesystem I/O, `dns.lookup()` (the default resolver!), and `crypto.pbkdf2`/`bcrypt`-style CPU work** all queue through it. A DNS-heavy or crypto-heavy app can be *silently bottlenecked on 4 threads* — requests queue behind each other for no obvious reason, CPU looks idle, latency spikes.

```bash
# Raise the pool (rule of thumb: number of cores, or higher for crypto/fs-heavy apps):
Environment=UV_THREADPOOL_SIZE=16
```

---

## Graceful Shutdown — The Real, Correct Code

This is one of the most valuable things in this whole curriculum for a Node dev, so do it right. When you deploy, systemd sends `SIGTERM` (topic 17); `docker stop` sends `SIGTERM`; Kubernetes sends `SIGTERM` then waits `terminationGracePeriodSeconds`. If you don't handle it, the process is hard-killed **mid-request** and every in-flight client gets a dropped connection. Graceful shutdown drains cleanly.

```js
// server.js — graceful shutdown done correctly
const http = require('http');
const server = http.createServer(app);
server.listen(process.env.PORT || 3000);

let shuttingDown = false;

async function shutdown(signal) {
  if (shuttingDown) return;      // ignore a second SIGTERM
  shuttingDown = true;
  console.log(JSON.stringify({ level: 'info', msg: 'shutdown start', signal }));

  // 1) STOP ACCEPTING NEW CONNECTIONS. server.close() stops taking new ones and
  //    fires its callback once all EXISTING connections have finished.
  server.close(async () => {
    try {
      // 3) After in-flight requests drain, release downstream resources:
      await db.end();               // close the DB pool (return connections)
      await queue.disconnect();     // stop consuming from the message queue
      console.log(JSON.stringify({ level: 'info', msg: 'shutdown clean' }));
      process.exit(0);              // 4) clean exit
    } catch (err) {
      console.error(JSON.stringify({ level: 'error', msg: 'shutdown error', err: String(err) }));
      process.exit(1);
    }
  });

  // 2b) THE KEEP-ALIVE GOTCHA: idle HTTP keep-alive sockets are "open connections",
  //     so server.close() will NEVER fire its callback while they sit there idle.
  //     Actively close idle sockets (Node 18.2+) so close() can complete:
  server.closeIdleConnections();
  // (also set server.keepAliveTimeout / server.headersTimeout so slow/idle
  //  clients can't hold connections open indefinitely.)

  // SAFETY NET: if something (a stuck socket, a hung DB call) prevents a clean
  //   exit, force-quit before systemd's TimeoutStopSec / K8s grace period SIGKILLs
  //   us (which would be an unclean, unlogged death). .unref() so this timer
  //   itself doesn't keep the process alive.
  setTimeout(() => {
    console.error(JSON.stringify({ level: 'error', msg: 'forced exit after timeout' }));
    process.exit(1);
  }, 25_000).unref();               // < TimeoutStopSec=30, so WE decide the exit
}

process.on('SIGTERM', () => shutdown('SIGTERM'));   // systemd / docker / k8s
process.on('SIGINT', () => shutdown('SIGINT'));     // Ctrl-C in dev
```

The ordered steps, and what ties to what:

```
SIGTERM arrives (systemd stop / docker stop / k8s pod delete)
  │
  1. server.close()            → stop accepting NEW connections
  2. server.closeIdleConnections() → release idle keep-alive sockets so close() CAN finish
  3. wait for in-flight requests to complete (server.close callback)
  4. close DB pool / MQ consumers  → no half-open transactions
  5. process.exit(0)           → clean exit BEFORE the supervisor's timeout
  │
  SAFETY NET: setTimeout(...).unref() force-exits if step 3/4 hangs
  │
  timers must fit inside: systemd TimeoutStopSec=30 / docker stop -t / k8s
  terminationGracePeriodSeconds — or the supervisor SIGKILLs you mid-drain.
```

### The PID-1-in-Docker problem (topic 16 callback) — THE big one

This is subtle and it silently breaks every deploy. In Linux, **PID 1 is special**: it does *not* get the kernel's default signal dispositions. A normal process with no SIGTERM handler is terminated by default; **PID 1 with no handler ignores the signal entirely.** And Node running as PID 1 in a container has no default handler for SIGTERM — so `docker stop` sends SIGTERM, nothing happens, Docker waits its grace period, then SIGKILLs. Your beautiful graceful handler *never runs*.

Worse is the common mistake of wrapping node in a shell/npm script:

```dockerfile
# WRONG — npm (or sh) becomes PID 1 and does NOT forward SIGTERM to node:
CMD npm start
# → SIGTERM goes to npm (PID 1). npm doesn't forward it. node never hears it.
#   Every `docker stop` / deploy HARD-KILLS in-flight requests.

# WRONG — shell form ALSO wraps in `/bin/sh -c`, so sh is PID 1:
CMD node server.js

# RIGHT — exec form makes node ITSELF PID 1 (and it DOES have your handler):
CMD ["node", "server.js"]
```

Even with exec form, PID 1 has quirks (it must reap zombie children — topic 16). The robust fix is a tiny init as PID 1 that forwards signals and reaps zombies:

```dockerfile
# docker run --init  (adds tini as PID 1), OR bake tini in:
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "server.js"]
```

Rules to never break:
1. **Exec-form `CMD ["node", "server.js"]`** — never shell form, never `npm start`, as the container entrypoint.
2. **Use `--init` / tini** so PID 1 forwards signals and reaps zombies.
3. **Never wrap node in npm scripts** for the container entrypoint — npm swallows SIGTERM.

Do this and `docker stop` / a Kubernetes rollout delivers SIGTERM straight to node, your handler runs, requests drain. Skip it and *every deploy drops live traffic.*

---

## Clustering — Using All Your Cores

Node runs your JS on **one thread** (topic 18's "one core pegged at 100%, the box shows 12% total" — a 8-core box with one busy core is ~12%). To use all cores you must run **multiple processes**:

```
Options to use N cores:
  • cluster module        → master forks N workers sharing the listen socket.
  • PM2 cluster mode      → `pm2 start server.js -i max` — same idea, managed.
  • N systemd instances   → myapp@1..myapp@N template units behind nginx.
  • N containers/pods      → run N replicas, let the load balancer spread traffic.
                            THIS is the container-native answer: one process per
                            container, scale by replica count, LB does the fan-out.
```

The kernel primitive that makes multiple processes share one port cleanly is **`SO_REUSEPORT`** — several processes each `bind()` the same port and the kernel load-balances incoming connections across them (the `cluster` module and modern setups use this so there's no single accept bottleneck).

Opinion: **in containers, don't use the cluster module — run N replicas** and let the orchestrator/LB spread load. One process per container keeps health checks and restarts honest. On a single bare VM, `cluster` or `PM2 -i max` or N systemd instances are all reasonable.

---

## nginx in Front — and WHY, Concretely

Never put a bare Node process directly on the internet. Put nginx in front. Concrete reasons:

```
WHY NGINX:
  • TLS termination      — nginx does HTTPS; Node speaks plain HTTP on :3000.
  • Static files         — nginx serves /public/* from disk; don't waste the Node
                           event loop reading files it could stream directly.
  • Buffering slow clients (SLOWLORIS PROTECTION) — nginx buffers slow/partial
                           requests and only hands node a COMPLETE request. A bare
                           Node facing the internet can be held hostage by many
                           slow-trickle connections (slowloris) tying up its loop.
  • gzip                 — compress responses at the proxy.
  • Rate limiting        — limit_req zones throttle abusive clients before Node.
  • Privileged ports     — nginx binds 80/443 (ports <1024 need root/CAP_NET_BIND —
                           topic 28) while YOUR app stays on :3000 as the unprivileged
                           nodeapp user (topic 32). The root-needing part is isolated
                           to hardened, boring nginx; your code never runs as root.
```

A real reverse-proxy config:

```nginx
# /etc/nginx/sites-available/myapp
upstream myapp {
    server 127.0.0.1:3000;          # Node app, unprivileged, loopback only
    keepalive 32;                    # reuse upstream connections
}

server {
    listen 443 ssl http2;
    server_name api.example.com;
    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    client_max_body_size 25m;        # allow up to 25MB uploads (else nginx 413s them)

    # rate limit (zone defined in http{}): 10 req/s with a burst of 20
    limit_req zone=api burst=20 nodelay;

    location / {
        proxy_pass http://myapp;

        # ── the essential proxy header block ──────────────────────────────
        proxy_set_header Host              $host;               # original Host
        proxy_set_header X-Real-IP         $remote_addr;        # client IP
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;             # http vs https
        # Without these, req.ip is nginx's 127.0.0.1 and secure-cookie logic breaks.

        # ── WebSocket upgrade (socket.io, ws) ─────────────────────────────
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout 60s;       # how long to wait for the app to respond
    }

    location /public/ {               # serve static assets from disk, bypass Node
        alias /srv/app/public/;
        expires 1h;
    }
}
```

For those headers to work, tell Express to trust the proxy so `req.ip` and secure-cookie detection use `X-Forwarded-*` instead of the loopback address:

```js
app.set('trust proxy', 1);   // trust ONE proxy hop (nginx). Now req.ip = real client,
                             // and req.secure reflects X-Forwarded-Proto=https.
```

### 502 vs 504 vs 499 (topic 30 callback) — map the code to the cause

```
502 Bad Gateway     nginx reached upstream but got an INVALID/no response.
                    → your Node app CRASHED, isn't listening on :3000, or died
                      mid-response. Check `systemctl status myapp` / journalctl.
504 Gateway Timeout nginx connected but the app didn't respond within
                    proxy_read_timeout. → app is ALIVE but SLOW/HUNG (blocked event
                    loop, slow DB query, deadlock). The process is up; the request stuck.
499 (nginx-specific) the CLIENT closed the connection before nginx replied.
                    → user hit stop / mobile client gave up / a too-aggressive
                      client timeout. Not your server erroring — the caller left.
```

Reading these correctly on an incident saves you from restarting a healthy app (504 = look at slowness, not liveness) or hunting a phantom bug (499 = the client, not you).

---

## Zero-Downtime Deploys

A plain `git pull && systemctl restart` **drops requests**: it overwrites files under the running process and there's a window where the old process is dead and the new one isn't accepting yet. Do it atomically instead.

### The atomic-symlink release pattern (topic 09 callback)

```
/srv/app/
├── releases/
│   ├── 20260712T0900/        ← previous release (full build)
│   └── 20260712T1030/        ← new release (freshly built)
├── shared/                    ← persistent stuff: uploads/, .env symlink target
└── current → releases/20260712T1030   ← a SYMLINK, atomically re-pointed
```

```bash
# build the new release into releases/TIMESTAMP, then flip the symlink atomically:
ln -sfn /srv/app/releases/20260712T1030 /srv/app/current
#   -s symbolic  -f force (replace)  -n treat existing symlink as a file, don't
#   deref into it. ln -sfn does an atomic rename() under the hood (topic 09) — the
#   symlink swap is a single kernel operation; no moment where `current` is missing.
sudo systemctl reload myapp   # or restart — combined with graceful shutdown = no dropped reqs
```

Because `current` is swapped with an atomic `rename()` (topic 09's atomic-rename lesson — a rename over an existing path is atomic on the same filesystem), there is never an instant where `current` points at nothing. Combined with graceful shutdown (SIGTERM drains in-flight requests) and `Restart=always`, the changeover drops zero requests. Rolling back is just re-pointing `current` at the previous release and reloading.

**Blue/green and draining:** run two full environments (blue live, green idle); deploy to green; health-check green; flip the load balancer from blue to green; drain blue (let its in-flight requests finish behind the LB before stopping it). Same principle at the LB layer instead of the symlink layer.

---

## Health Checks — Liveness vs Readiness

Two different questions, two different endpoints. Conflating them causes restart storms.

```
/healthz  (LIVENESS)   "Is the PROCESS alive and the event loop responsive?"
                       → cheap. Return 200 if the process can answer. Do NOT check
                         the DB here. If this fails, the supervisor RESTARTS the app.

/readyz   (READINESS)  "Can this instance actually SERVE traffic right now?"
                       → check real dependencies: DB reachable, cache up, migrations done.
                         If this fails, the LOAD BALANCER stops sending traffic — but
                         does NOT restart the process.
```

**Why conflating them causes restart storms:** if your *liveness* check also pings the DB, then a brief DB blip makes liveness fail → the supervisor **kills and restarts every instance simultaneously** → the fresh instances all hammer the recovering DB on startup → the DB stays down → they all fail liveness again → **crash loop across the fleet.** Liveness must depend only on the process itself; readiness is where dependency checks belong (a DB blip should pull instances *out of the LB*, not *restart* them).

```js
app.get('/healthz', (req, res) => res.sendStatus(200));   // process is up. that's it.
app.get('/readyz', async (req, res) => {
  try { await db.query('SELECT 1'); res.sendStatus(200); }  // can we really serve?
  catch { res.sendStatus(503); }                            // pull me from the LB
});
```

Wire liveness to systemd (a watchdog or an external check that restarts) / Kubernetes `livenessProbe`; wire readiness to the LB / K8s `readinessProbe`. Post-deploy, verify with curl + retry (topic 30/13 callback):

```bash
# post-deploy smoke check with retry (topic 30/13)
for i in $(seq 1 10); do
  code=$(curl -s -o /dev/null -w '%{http_code}' https://api.example.com/readyz)
  [ "$code" = "200" ] && { echo "ready"; exit 0; }
  echo "attempt $i: got $code, retrying"; sleep 3
done
echo "NOT ready after 10 tries — rolling back"; exit 1
```

---

## Observability

You cannot operate what you cannot see.

```
STRUCTURED LOGS (topic 23)  → log JSON to STDOUT (not to a file the app manages —
                              let journald/the platform capture stdout). One line =
                              one JSON object: {level, msg, requestId, ...}. Machine-
                              parseable, greppable, ships to any log system.
REQUEST IDs                 → generate/propagate an X-Request-Id per request; log it
                              on every line so you can trace ONE request across services.
```

The diagnosis toolkit:

```
node --inspect + heap snapshot   → for a LEAK: attach Chrome DevTools, take two heap
                                   snapshots minutes apart, diff them — the growing
                                   object set is your leak.
node --cpu-prof                  → write a CPU profile (.cpuprofile) to find where the
                                   event loop spends time (a hot function pegging a core).
process.memoryUsage()            → { rss, heapTotal, heapUsed, external, arrayBuffers }
     rss        = total resident memory (what the OS/cgroup counts → the OOM boundary)
     heapTotal  = V8 heap reserved     heapUsed = live JS objects (watch it climb = leak)
     external   = C++ objects bound to JS (Buffers etc.) — leaks here don't show in heapUsed
EVENT-LOOP LAG                   → THE key Node health metric. Measure how late a
                                   setInterval fires vs schedule; rising lag = the loop
                                   is blocked. A blocked event loop looks like "the server
                                   is UP but everything TIMES OUT": one core at 100%, the
                                   accept queue / Recv-Q growing (topic 28: `ss -lnt`
                                   shows Recv-Q climbing on :3000), requests queue and 504.
```

A blocked event loop is the most Node-specific failure there is: `systemctl status` says "active (running)", the process is alive, yet every request times out because the single thread is stuck in a synchronous loop (a giant `JSON.parse`, a sync `crypto`, a `while` that won't yield). Event-loop lag is how you *see* it before users do.

---

## Exact Syntax Breakdown

```
systemctl enable --now myapp          journalctl -u myapp -f
          │      │     │                        │  │      └─ follow (like tail -f)
          │      │     └─ the unit               │  └─ this unit only
          │      └─ ALSO start it right now      └─ query the journal (topic 23)
          └─ start on boot (symlink into target)

ExecStart=/usr/bin/node /srv/app/dist/server.js
          │              └─ script (absolute path too — cwd is WorkingDirectory)
          └─ ABSOLUTE path to node (systemd won't find a bare `node`)

NODE_OPTIONS=--max-old-space-size=384     UV_THREADPOOL_SIZE=16
             │                  └─ MB cap for V8 old-space heap    └─ libuv worker threads
             └─ flags applied to every `node` invocation

ln -sfn /srv/app/releases/NEW /srv/app/current
   ││││                        └─ the symlink to (atomically) re-point
   │││└─ n: don't follow an existing symlink target — replace the link itself
   ││└─ f: force (remove existing)
   │└─ s: symbolic link
   └─ ln: link
```

---

## Example 1 — Basic: from `node server.js` to a real service

```bash
# The flashlight (DON'T ship this):
node /srv/app/dist/server.js &        # dies on logout, no restart, no logs, your uid

# The ceiling light:
sudo tee /etc/systemd/system/myapp.service >/dev/null <<'EOF'
[Unit]
Description=My Node API
After=network-online.target
[Service]
User=nodeapp
WorkingDirectory=/srv/app
ExecStart=/usr/bin/node /srv/app/dist/server.js
Environment=NODE_ENV=production
EnvironmentFile=/etc/myapp/env
Restart=always
RestartSec=5
LimitNOFILE=65535
StandardOutput=journal
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now myapp
systemctl status myapp                 # active (running), runs as nodeapp

# Prove it survives disconnect and restarts on crash:
sudo kill $(pgrep -f 'node .*server.js')   # simulate a crash
sleep 6; systemctl status myapp             # active (running) again — systemd relaunched it
```

---

## Example 2 — Production Scenario

**Situation:** 2am page. The Node API is "up" (`systemctl status` = active) but every request times out with 504 at nginx. Uploads and health checks all hang. It got worse right after the 1am deploy.

```bash
# nginx says 504 = app is ALIVE but not responding in time. So it's slowness, not a crash.
tail -3 /var/log/nginx/error.log
# upstream timed out (110: Connection timed out) while reading response, upstream: "http://127.0.0.1:3000"

# Is the process actually alive? Yes.
systemctl status myapp | head -3
# Active: active (running) since ...   ← it's up. So why 504?

# THE tell: is the event loop blocked? Check CPU per core and the accept queue.
top -1 -n1 | grep node
#  4127 nodeapp  20  0 ... 100.0  ...  node   ← ONE core pegged at 100% (topic 18)

ss -lnt 'sport = :3000'
# State   Recv-Q Send-Q  Local Address:Port
# LISTEN  231    511     127.0.0.1:3000     ← Recv-Q=231: connections PILING UP unaccepted
#                                             (topic 28) — the loop is too blocked to accept

# Confirm memory isn't the real story (rule out OOM):
journalctl -u myapp --since "1 hour ago" | grep -iE 'oom|killed|heap'
# (nothing) → not an OOM, not a V8 heap crash. It's a BLOCKED EVENT LOOP.

# Grab a CPU profile of the live process to find the hot path:
kill -SIGUSR1 4127     # if the app is wired to start the inspector on SIGUSR1
# ...or restart with --cpu-prof in a canary. The 1am deploy added a synchronous
# JSON.parse of a 40MB payload in a hot route — pegging the single thread.

# Immediate mitigation: roll back to the previous release (atomic symlink) and reload.
sudo ln -sfn /srv/app/releases/PREVIOUS /srv/app/current
sudo systemctl reload myapp
# Recv-Q drains, 504s stop. Root cause: a sync CPU-bound call blocked the event loop.
```

**What you diagnosed:** not a crash (process was `active`), not memory (no OOM/heap error) — a **blocked event loop**: one core at 100%, `Recv-Q` climbing on :3000, requests queuing until nginx 504s. The atomic-symlink rollback restored service; the fix is to move the heavy synchronous work off the event loop (worker thread / stream / chunk it).

---

## Common Mistakes

### Mistake 1 — `CMD npm start` (or shell-form CMD) in the Dockerfile

**Wrong:** `CMD npm start` → npm becomes PID 1 and does not forward SIGTERM.
**Right:** `CMD ["node", "server.js"]` (exec form) + `--init`/tini.
**Root cause:** PID 1 has no default signal disposition; npm/sh don't forward SIGTERM to the child. Your graceful handler never runs.
**Broken state:** every `docker stop` / rollout hangs the grace period then SIGKILLs mid-request — dropped connections on every deploy.
**Fix/Prevention:** exec-form CMD; tini as PID 1; never wrap node in npm as the entrypoint.

### Mistake 2 — Bare `node` in the systemd ExecStart (NVM trap)

**Wrong:** `ExecStart=node /srv/app/server.js`.
**Right:** `ExecStart=/usr/bin/node /srv/app/server.js` (absolute path).
**Root cause:** systemd doesn't source your shell profile (topic 14/22), so the nvm-managed `node` on your interactive PATH doesn't exist for it.
**Broken state:** `status=203/EXEC`, "No such file or directory" — service never starts.
**Fix/Prevention:** absolute path to a system-wide node; verify with `which node` as the *service* user, not you.

### Mistake 3 — Raising `--max-old-space-size` to "fix" an OOM kill

**Wrong:** seeing exit 137, bumping `--max-old-space-size=4096`.
**Right:** first identify WHICH OOM it is. Exit 137 / "killed by OOM killer" = out of REAL RAM; raising the heap makes it WORSE.
**Root cause:** V8 heap-limit crash ("FATAL ERROR ... JS heap out of memory") and kernel/cgroup OOM (exit 137) are opposite problems.
**Broken state:** you give V8 more heap, it grabs more RAM, the cgroup kills it sooner and harder.
**Fix/Prevention:** read journal/dmesg to classify; for real-RAM OOM fix the leak or add RAM/raise MemoryMax; for V8-heap set the flag or fix the leak.

### Mistake 4 — Deploying with NODE_ENV unset

**Wrong:** no `Environment=NODE_ENV=production`; app runs in default/development.
**Right:** set it explicitly and verify via `/proc/PID/environ`.
**Root cause:** many libs branch on NODE_ENV; unset = dev fast-paths off, verbose errors on.
**Broken state:** slower app (Express recompiles views), and stack traces leaked to end users/attackers.
**Fix/Prevention:** set it in the unit; assert it at boot (`if (process.env.NODE_ENV !== 'production') log a loud warning`).

### Mistake 5 — Running PM2/nodemon inside a container

**Wrong:** `CMD ["pm2-runtime", ...]` or `nodemon` as the container entrypoint.
**Right:** one process per container; the orchestrator supervises.
**Root cause:** the platform's restart/health logic assumes it watches your process directly; an inner supervisor hides crashes and keeps the container "healthy" while the app is wedged.
**Broken state:** Kubernetes never recreates a bad pod because PM2 keeps restarting node internally; health checks lie.
**Fix/Prevention:** exec-form single-process CMD; scale by replicas; let restart policies do their job.

---

## Hands-On Proof

```bash
# PROVE IT: the running process's actual environment (NODE_ENV, PORT)
sudo cat /proc/$(pgrep -f 'node .*server')/environ | tr '\0' '\n' | grep -E 'NODE_ENV|PORT'

# PROVE IT: the process runs as the unprivileged service user, with the fd limit set
ps -o user,pid -C node
cat /proc/$(pgrep -f 'node .*server')/limits | grep 'open files'   # LimitNOFILE took effect

# PROVE IT: systemd restarts it on crash
sudo kill $(pgrep -f 'node .*server'); sleep 6; systemctl is-active myapp   # → active

# PROVE IT: which OOM is it — read the journal for the signature
journalctl -u myapp | grep -iE 'oom-kill|status=9|heap out of memory'

# PROVE IT: a blocked event loop shows one pegged core + a growing accept queue
top -1 -n1 | grep node          # one core near 100%
ss -lnt 'sport = :3000'         # Recv-Q climbing = connections not being accepted

# PROVE IT: SIGTERM triggers your graceful handler (watch the logs while stopping)
journalctl -u myapp -f &
sudo systemctl stop myapp       # you should see "shutdown start"/"shutdown clean" logs

# PROVE IT: the atomic symlink swap never leaves 'current' missing
ls -l /srv/app/current          # → symlink to releases/TIMESTAMP
```

---

## Practice Exercises

### Exercise 1 — Easy

Write a minimal `myapp.service` for a Node app that: runs as `nodeapp`, uses an absolute node path, sets `NODE_ENV=production`, restarts on crash, and logs to journald. `daemon-reload`, `enable --now`, confirm `systemctl is-active` is `active`, then `kill` the process and prove systemd brings it back. Paste the transcript.

### Exercise 2 — Medium

Add graceful shutdown to a tiny HTTP server: on `SIGTERM`, `server.close()`, log "draining", exit 0, with a `setTimeout(...).unref()` safety net. Run it, hit it with a slow request (`curl` a route that sleeps 3s), and in another terminal `kill -TERM` the pid *during* the request. Prove the in-flight request completes and the process then exits cleanly (not mid-request). Then containerize it and show that `CMD npm start` breaks this (hard kill) while `CMD ["node","server.js"]` fixes it.

### Exercise 3 — Hard (Production Simulation)

Reproduce the container memory trap. In a 256 MB-limited container (`docker run -m 256m`), run a Node script that allocates a growing array. First run it *without* `--max-old-space-size` and capture the failure (observe exit 137 via `docker inspect --format '{{.State.ExitCode}}'` and the OOM in `dmesg`). Then run it *with* `NODE_OPTIONS=--max-old-space-size=192` and show it now hits V8's "JS heap out of memory" *inside* the limit instead of being OOM-killed by the cgroup. Explain in two lines why the second behavior is the safer failure mode.

---

## Mental Model Checkpoint

1. List four distinct reasons `node server.js` in an SSH session is not a production deployment, and the mechanism behind each.
2. On a single VM, why is systemd the default recommendation over PM2 — and what does `pm2 startup` reveal about running PM2?
3. Why must `ExecStart` use an absolute path to node, and what error do you get if you don't?
4. How do you tell a V8 heap-limit crash from a kernel/cgroup OOM kill, and why is the fix *opposite*?
5. Why does Node in a 512 MB container often get OOM-killed even though the app "isn't using much memory"?
6. Walk through the correct SIGTERM graceful-shutdown sequence, including the keep-alive gotcha and the hard-exit safety net.
7. Why does `CMD npm start` in a Dockerfile silently break graceful shutdown, and what are the two fixes?
8. What's the difference between `/healthz` and `/readyz`, and why does putting a DB check in liveness cause a fleet-wide restart storm?

---

## Quick Reference Card

| Command / Directive | What It Does | Key Detail |
|---|---|---|
| `systemctl enable --now myapp` | Start now + on boot | needs `[Install] WantedBy=` |
| `systemctl reset-failed myapp` | Clear crash-loop counter | after hitting StartLimitBurst |
| `journalctl -u myapp -f` | Follow app logs (topic 23) | `-u` unit, `-f` follow |
| `Restart=always` / `RestartSec=5` | Relaunch on exit, backoff | pair with StartLimit* |
| `LimitNOFILE=65535` | Raise fd limit (EMFILE fix) | default 1024 too low (topic 19) |
| `MemoryMax=1G` | cgroup memory hard cap | coordinate with `--max-old-space-size` |
| `TimeoutStopSec=30` | Grace before SIGKILL | pair with graceful shutdown code |
| `NODE_OPTIONS=--max-old-space-size=N` | V8 old-space heap cap (MB) | ~75% of container RAM |
| `UV_THREADPOOL_SIZE=N` | libuv worker threads | default 4; fs/dns/crypto queue here |
| `process.memoryUsage()` | rss/heapTotal/heapUsed/external | rss = the OOM boundary |
| `ln -sfn NEW current` | Atomic release swap (topic 09) | `-n` don't deref existing link |
| `ss -lnt 'sport = :3000'` | Listen socket + accept queue | Recv-Q climbing = blocked loop (topic 28) |
| `CMD ["node","server.js"]` | Exec-form entrypoint | node is PID 1 & gets SIGTERM |

---

## When Would I Use This at Work?

### Scenario 1: First deploy of a new service to a VM
You've built the app and created the `nodeapp` user (topic 32). Now you write the systemd unit — absolute node path, `NODE_ENV=production`, `EnvironmentFile`, `Restart=always`, `LimitNOFILE=65535`, hardening block — add graceful shutdown to the code, put nginx in front for TLS and buffering, and wire `/healthz` + `/readyz`. `systemctl enable --now`, smoke-test with a curl-retry loop, done. It now survives reboots, crashes, and your logout.

### Scenario 2: "The app is up but everything times out"
`systemctl status` says active, but nginx returns 504. You check per-core CPU (one core at 100%), `ss -lnt` on :3000 (Recv-Q climbing), rule out OOM in the journal, and conclude: blocked event loop. You roll back with the atomic symlink, then fix the synchronous hot path (a giant sync parse) by moving it to a worker thread.

### Scenario 3: Deploys keep dropping user requests
Every deploy shows a burst of connection errors. You discover the Dockerfile uses `CMD npm start`, so SIGTERM never reaches node and the graceful handler never runs — every rollout hard-kills in-flight requests. You switch to exec-form `CMD ["node","server.js"]` with `--init`, verify the shutdown logs appear on `docker stop`, and deploys become clean.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 17 — Process Management | SIGHUP kills a naive node; SIGTERM drives graceful shutdown; signals are the core |
| **Builds on** | 18 — System Resources | single-thread/one-core behavior, load, and the OOM/memory-pressure picture |
| **Builds on** | 19 — File Descriptors | LimitNOFILE / EMFILE — why the fd cap must be raised |
| **Builds on** | 22 — systemd | the unit file is the deployment; this topic extends it to production-grade |
| **Builds on** | 32 — Deploy User & Permissions | the unprivileged user, EnvironmentFile perms, and secrets handling |
| **Builds on** | 28 — Ports & Sockets | privileged ports (nginx on 80/443), Recv-Q/accept queue diagnosis |
| **Next** | 34 — Performance Investigation | when the app is slow/leaking, dig deeper with strace/perf/profiling |
| **Used by** | 35 — Security Basics | hardening, NODE_ENV stack-trace leaks, and the production checklist |
```
