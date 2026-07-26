# 110 — Process management in production

## What is this?

Process management is the discipline of keeping your Node.js app **running, restarting, and observable** without a human babysitting it 24/7. In production, `node server.js` running in a terminal is not enough — if the process crashes, the server goes offline until someone notices and restarts it manually. A process manager (like **PM2**) or an OS-level supervisor (like **systemd**) watches your app, restarts it instantly on crash, rotates its logs so disk doesn't fill up, and exposes metrics so you know if it's healthy. Think of it like a building's fire alarm and sprinkler system — you don't want to be the person who has to notice the fire and run to put it out; you want automated systems that detect the problem and respond before anyone even calls you.

## Why does it matter for backend development?

Every real backend eventually crashes — an unhandled exception, a memory leak, an out-of-memory kill from the OS, a bad deploy. Without process management, that crash means downtime until a human intervenes, which is unacceptable for anything customers depend on. Process managers give you **auto-restart** (the app comes back up in milliseconds), **log rotation** (so `app.log` doesn't grow to 500GB and crash the disk), **cluster mode** (running one instance per CPU core for throughput), and **monitoring** (CPU/memory dashboards, restart counts, uptime). Every company running Node.js in production — from a two-person startup to a large engineering org — uses PM2, systemd, or a container orchestrator (Kubernetes) to manage the process lifecycle. This is one of the first things interviewers ask about when they want to know if you've actually run something in production versus only building it locally.

---

## Syntax / API

```bash
# ── PM2 — the most common Node.js-specific process manager ──────────────────

# Install PM2 globally (it manages Node processes system-wide)
npm install -g pm2

# Start an app under PM2 — it restarts automatically on crash by default
pm2 start server.js --name api-server

# Start in cluster mode — spawns one process per CPU core, load-balanced
pm2 start server.js --name api-server -i max

# List all processes PM2 is managing, with status, CPU, memory, uptime
pm2 list

# Show live logs streamed from the running process
pm2 logs api-server

# Reload with zero downtime (starts new workers, kills old ones one by one)
pm2 reload api-server

# Stop / restart / delete a managed process
pm2 stop api-server
pm2 restart api-server
pm2 delete api-server

# Persist the current process list so it restarts after a server reboot
pm2 save
pm2 startup   # generates + runs the OS command to boot PM2 on system startup

# Rotate logs automatically (install PM2's log rotation module)
pm2 install pm2-logrotate
```

```ini
# ── systemd — the Linux OS-level init system (no extra install needed) ──────
# File: /etc/systemd/system/api-server.service

[Unit]
Description=Node.js API server                 # human-readable description
After=network.target                            # start only after networking is up

[Service]
Type=simple                                     # process runs in the foreground
User=nodeapp                                     # run as a non-root system user
WorkingDirectory=/opt/api-server                 # cwd for the process
ExecStart=/usr/bin/node server.js                # the command systemd runs
Restart=always                                   # restart no matter how it exits
RestartSec=3                                     # wait 3 seconds before restarting
Environment=NODE_ENV=production                  # inject env vars directly
StandardOutput=append:/var/log/api-server/out.log  # redirect stdout to a file
StandardError=append:/var/log/api-server/err.log   # redirect stderr to a file

[Install]
WantedBy=multi-user.target                       # enable at normal system boot
```

```bash
# systemd management commands
sudo systemctl daemon-reload           # reload unit files after editing them
sudo systemctl enable api-server       # start automatically on boot
sudo systemctl start api-server        # start it now
sudo systemctl status api-server       # check running state, recent logs
sudo journalctl -u api-server -f       # follow live logs from systemd's journal
```

---

## How it works — line by line

**PM2** is itself a small Node.js process (a "master" or "God process") that stays alive and watches your app as a **child process**. When your app crashes or exits, PM2's parent process detects the exit event immediately and spawns a fresh copy — usually within milliseconds. In cluster mode (`-i max`), PM2 forks multiple copies of your app (one per CPU core) and load-balances incoming connections across them using Node's built-in `cluster` module under the hood, so a crash in one worker doesn't take down the others.

**systemd** works at a different layer — it's part of the Linux operating system itself, not Node-specific. You describe your app in a `.service` file (a plain text config), and systemd's own supervisor process (PID 1's helper, `systemd`) starts your app, watches its exit code, and restarts it per the `Restart=` policy you set. Because it's OS-level, systemd can manage *any* process (Node, Python, Go, a shell script) the same way, and it integrates with the server's boot sequence natively — no extra package needed.

**Log rotation** in both cases works the same conceptually: instead of one log file growing forever, the rotator periodically renames the current file (e.g. `out.log` → `out.log.1`), starts a fresh empty file, and deletes files older than a retention limit — keeping disk usage bounded while still preserving recent history for debugging.

**Monitoring** means continuously collecting metrics — CPU%, memory, restart count, uptime, event loop delay — and exposing them somewhere a human or alerting system can see them, so degradation is caught *before* a full crash.

---

## Example 1 — basic

```js
// File: server.js
// A minimal Express server designed to be run under PM2 or systemd —
// note it does NOT try to manage its own restarts; that's the supervisor's job.

const express = require('express');       // web framework
const app = express();                     // create the app instance

const PORT = process.env.PORT || 3000;     // read port from env, fallback to 3000

// A simple health endpoint — supervisors and load balancers can poll this
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok', pid: process.pid, uptime: process.uptime() });
});

// A route that simulates real work
app.get('/users/:userId', (req, res) => {
  const userId = req.params.userId;        // pull the userId from the URL
  res.json({ userId, name: 'Sample User' }); // respond with mock data
});

// Log which PID is serving requests — useful when running multiple instances
console.log(`Worker process ${process.pid} listening on port ${PORT}`);

app.listen(PORT);                          // start listening for connections
```

```bash
# Run it under PM2 — PM2 becomes the parent that restarts it on crash
pm2 start server.js --name api-server

# Check status — shows PID, uptime, restart count, memory usage
pm2 list

# Kill the process manually to prove auto-restart works
kill -9 $(pm2 pid api-server)
# PM2 detects the exit instantly and spawns a new process automatically

pm2 list   # → restart count is now 1, status is "online" again
```

---

## Example 2 — real world backend use case

```js
// File: ecosystem.config.js
// PM2 "ecosystem file" — the real-world way teams configure PM2 for a
// production deployment instead of typing flags on the command line.

module.exports = {
  apps: [
    {
      name: 'api-server',                    // name shown in `pm2 list`
      script: './src/server.js',             // entry point to run
      instances: 'max',                      // one worker per CPU core (cluster mode)
      exec_mode: 'cluster',                  // enables load-balanced clustering
      watch: false,                          // never auto-restart on file change in prod
      max_memory_restart: '500M',            // restart a worker if it exceeds 500MB RAM
      env: {
        NODE_ENV: 'production',              // env vars injected into the process
        PORT: 4000,
      },
      error_file: './logs/api-server-error.log',  // stderr destination
      out_file: './logs/api-server-out.log',      // stdout destination
      merge_logs: true,                       // combine logs from all cluster workers
      log_date_format: 'YYYY-MM-DD HH:mm:ss', // timestamp format in logs
      // Zero-downtime reload settings
      wait_ready: true,                       // wait for process.send('ready') before old worker dies
      listen_timeout: 10000,                  // ms to wait for the new worker to signal ready
      kill_timeout: 5000,                     // ms to allow graceful shutdown before force-kill
    },
  ],
};
```

```js
// File: src/server.js
// Cooperates with PM2's zero-downtime reload by signaling "ready" and
// handling graceful shutdown on SIGINT/SIGTERM (sent by PM2 during reload).

const express = require('express');
const app = express();

const dbConnection = require('./db');       // pretend DB connection module
const authToken = process.env.API_SECRET;   // secret loaded from environment (Topic 79)

app.get('/health', (req, res) => res.sendStatus(200));

const server = app.listen(process.env.PORT || 4000, () => {
  console.log(`api-server (pid ${process.pid}) up on port ${process.env.PORT}`);

  // Tell PM2 this worker has finished booting and is ready for traffic —
  // required for `wait_ready` zero-downtime reloads to work correctly
  if (process.send) {
    process.send('ready');
  }
});

// PM2 sends SIGINT during `pm2 reload` — stop accepting new connections,
// finish in-flight requests, then exit cleanly (full detail in Topic 64)
process.on('SIGINT', () => {
  console.log('Received SIGINT — closing server gracefully...');
  server.close(() => {                     // stop accepting new connections
    dbConnection.close();                  // close DB pool cleanly
    process.exit(0);                       // exit only after cleanup finishes
  });
});
```

```bash
# Deploy commands a backend dev actually runs

pm2 start ecosystem.config.js              # start using the config file above
pm2 install pm2-logrotate                  # auto-rotate logs, prevents disk fill-up
pm2 set pm2-logrotate:max_size 20M         # rotate each log file at 20MB
pm2 set pm2-logrotate:retain 14            # keep 14 rotated files, delete older ones
pm2 set pm2-logrotate:compress true        # gzip old rotated logs to save space

pm2 reload ecosystem.config.js             # zero-downtime redeploy after a code change
pm2 monit                                  # live terminal dashboard: CPU, memory per worker
pm2 save                                   # persist process list across server reboots
```

---

## Common mistakes

### Mistake 1 — Running Node directly in production with no supervisor

```bash
# ❌ WRONG — if this crashes (uncaught exception, OOM kill), it stays down
# until a human notices and SSHes in to restart it manually
node server.js &
```

```bash
# ✅ CORRECT — PM2 (or systemd) restarts it automatically within milliseconds
pm2 start server.js --name api-server
# or, using systemd's Restart=always policy, the OS itself relaunches it
```

### Mistake 2 — Letting log files grow forever

```js
// ❌ WRONG — appending forever with no rotation eventually fills the disk,
// which then crashes EVERYTHING on the server, not just this app
const fs = require('fs');
fs.appendFileSync('app.log', `${new Date().toISOString()} request received\n`);
// app.log grows unbounded — 6 months later it's 400GB and disk is full
```

```bash
# ✅ CORRECT — let the process manager rotate logs, or use a rotating logger
pm2 install pm2-logrotate                  # PM2 handles rotation automatically
pm2 set pm2-logrotate:max_size 20M
pm2 set pm2-logrotate:retain 14

# Or in code, use a proper logger with built-in rotation (Topic 63):
# const pino = require('pino');
# const logger = pino(pino.destination({ dest: './logs/app.log', mkdir: true }));
```

### Mistake 3 — Restarting on crash without fixing WHY it crashed (or restarting too fast)

```bash
# ❌ WRONG — infinite restart loop: app crashes on boot (bad config, missing
# env var), PM2 restarts it instantly, it crashes again, forever — burning
# CPU and flooding logs, while looking "online" for a split second each time
pm2 start server.js --name api-server
# no restart delay, no restart limit — a broken deploy spins forever
```

```js
// ✅ CORRECT — cap restarts and add backoff delay in the ecosystem config,
// AND monitor restart_count so repeated crashes trigger a real alert
module.exports = {
  apps: [{
    name: 'api-server',
    script: './server.js',
    max_restarts: 10,        // give up auto-restarting after 10 attempts
    min_uptime: '10s',       // a restart before 10s uptime counts as "unstable"
    restart_delay: 4000,     // wait 4 seconds between restart attempts
  }],
};
// Then wire pm2's restart_count metric into your alerting (Topic 111)
// so a crash-loop pages a human instead of silently spinning forever
```

---

## Practice exercises

### Exercise 1 — easy

Create a tiny Express server with one route (`GET /ping` returning `{ pong: true }`). Start it under PM2 with a custom name. Then:
1. Run `pm2 list` and note the PID and status
2. Force-kill the process with `kill -9 <pid>` from another terminal
3. Run `pm2 list` again and confirm PM2 auto-restarted it (check the restart count went up and a new PID appears)
4. Run `pm2 logs` to see the startup log line from the new process

```js
// Write your code here
```

---

### Exercise 2 — medium

Write an `ecosystem.config.js` for an app called `order-service` that:
1. Runs in cluster mode using all available CPU cores
2. Sets `NODE_ENV=production` and a custom `PORT=5000` via the `env` field
3. Restarts any worker that exceeds 300MB of memory usage
4. Writes stdout and stderr to separate log files under a `./logs` folder
5. Limits restarts to 8 attempts with a 3-second delay between them, treating any crash before 15 seconds of uptime as unstable

Then write the matching `server.js` that listens on `process.env.PORT`, exposes `/health`, and calls `process.send('ready')` once it's listening (guard for when `process.send` doesn't exist, e.g. running outside PM2).

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small **crash-loop guard** module, `crashGuard.js`, usable in any Node app (independent of PM2), that:
1. Tracks process restarts by writing a timestamped entry to a local file (`./data/restarts.log`) every time the module is required (i.e. every time the process boots)
2. On each boot, reads that file and counts how many restarts happened in the last 60 seconds
3. If there have been 5 or more restarts within that 60-second window, logs a clear `CRASH LOOP DETECTED` error to stderr and calls `process.exit(1)` immediately (so the supervisor — PM2 or systemd — stops trying and a human has to intervene, rather than looping forever silently)
4. Otherwise, lets the app continue booting normally
5. Exposes a `getRestartHistory()` function returning the array of recent restart timestamps

Require this module as the very first line of a sample `server.js` and simulate a crash loop by killing and restarting the process 5+ times quickly (a small shell loop is fine) to prove the guard trips.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
PM2 — Node-specific process manager
  pm2 start app.js --name api          → start & manage a process
  pm2 start app.js -i max              → cluster mode, one per CPU core
  pm2 list / pm2 status                → view status, PID, memory, restarts
  pm2 logs [name]                      → stream logs
  pm2 monit                            → live CPU/memory dashboard
  pm2 reload <name>                    → zero-downtime restart
  pm2 restart <name>                   → hard restart (brief downtime)
  pm2 stop / delete <name>             → stop / remove from PM2's list
  pm2 save + pm2 startup               → persist & auto-start on server reboot
  pm2 install pm2-logrotate            → adds automatic log rotation

SYSTEMD — OS-level supervisor (any language, not just Node)
  Restart=always / on-failure          → auto-restart policy
  RestartSec=N                         → delay between restart attempts
  systemctl start|stop|status <name>   → manage the service
  systemctl enable <name>              → start automatically on boot
  journalctl -u <name> -f              → follow logs

PM2 vs SYSTEMD — when to use which
  PM2      → Node-only shops, need cluster mode + zero-downtime reload out of the box
  systemd  → already using Linux init for other services, want one tool for everything
  Both     → commonly PM2 runs the Node app, and systemd keeps PM2 itself alive

LOG ROTATION RULES OF THUMB
  Rotate by size (e.g. 20MB) or by time (daily)
  Always cap retention (e.g. keep last 14 files) — never keep forever
  Compress old rotated files (gzip) to save disk
  Never let application code hand-roll unbounded appendFileSync logging

MONITORING BASICS TO TRACK
  Restart count           → spikes mean a crash loop or bad deploy
  Memory usage per worker → catches leaks before OOM kill
  CPU usage                → sustained high CPU = bottleneck or infinite loop
  Uptime since last restart → low uptime repeatedly = instability
  Event loop delay          → rising delay = the app is blocked/overloaded

GOTCHAS
  pm2 restart  → brief downtime (kills then starts)
  pm2 reload   → zero downtime (rolling restart, cluster mode only)
  max_memory_restart is a safety net, NOT a fix for a real memory leak
  A tight auto-restart loop with no max_restarts/restart_delay can mask
    a broken deploy indefinitely while looking "online"
```

---

## Connected topics

- **73 — Horizontal scaling with PM2** — goes deeper into `pm2 start -i max`, cluster mode internals, and `ecosystem.config.js` tuning for throughput.
- **64 — Graceful shutdown** — the `SIGINT`/`SIGTERM` handling shown in Example 2 is exactly what makes PM2's zero-downtime `reload` and systemd's `Restart=` policy safe to use.
- **111 — Monitoring and alerting** — takes the restart counts, memory, and CPU metrics PM2/systemd expose and wires them into Prometheus/Grafana dashboards and real alerts.
