# 73 — Horizontal scaling with PM2

## What is this?

PM2 is a production process manager for Node.js apps. It starts your app, keeps it alive if it crashes, and can run **multiple copies of your app across all your CPU cores** at once — this is called cluster mode. Think of a single-lane toll booth (one Node process handling every car/request on one core) versus opening every booth at the station (one process per CPU core, all sharing the incoming traffic) — same building, same road, but far more cars processed per minute.

## Why does it matter for backend development?

Node.js runs your JavaScript on a **single thread**, so a single `node server.js` process can only ever use one CPU core, no matter how many cores the machine has. On an 8-core production server, running one bare process wastes 7 cores. PM2's cluster mode forks your app once per core, load-balances incoming connections across them, and restarts any process that crashes — all without you writing a single line of cluster or load-balancing code yourself. Every backend developer deploying a real Node service to a VM (not a serverless platform) reaches for PM2 to get multi-core throughput, auto-restart on crash, log management, and zero-downtime deploys.

---

## Syntax / API

```bash
# Install PM2 globally so the `pm2` command is available anywhere
npm install -g pm2

# Start a single instance of an app — PM2 keeps it alive and names it "auth-api"
pm2 start server.js --name auth-api

# Start in CLUSTER MODE across all available CPU cores
# "-i max" tells PM2 to detect the CPU count and fork that many instances
pm2 start server.js --name auth-api -i max

# Start with an explicit instance count instead of "max"
pm2 start server.js --name auth-api -i 4

# List every process PM2 is managing, with status, CPU, memory, restarts
pm2 list

# Live dashboard — CPU%, memory%, logs, per-instance stats
pm2 monit

# Stream logs from all managed processes in real time
pm2 logs

# Reload with ZERO DOWNTIME — restarts instances one at a time,
# so there is always at least one instance serving traffic
pm2 reload auth-api

# Stop / restart / delete a process from PM2's process list
pm2 stop auth-api
pm2 restart auth-api
pm2 delete auth-api

# Persist the current process list so it survives a server reboot
pm2 save
pm2 startup   # generates + prints the OS command to auto-launch PM2 on boot
```

```js
// ecosystem.config.js — the recommended way to configure PM2 (instead of CLI flags)
// Place this file at the project root and run: pm2 start ecosystem.config.js

module.exports = {
  apps: [
    {
      name: 'auth-api',              // process name shown in `pm2 list`
      script: './src/server.js',     // entry file PM2 will run with node
      instances: 'max',              // fork one worker per CPU core
      exec_mode: 'cluster',          // enables cluster mode load balancing
      watch: false,                  // don't restart on file change (prod default)
      max_memory_restart: '400M',    // auto-restart an instance if it exceeds 400MB RAM
      env: {
        NODE_ENV: 'production',      // env vars injected into every instance
        PORT: 4000,
      },
      error_file: './logs/auth-api-error.log',   // stderr goes here
      out_file: './logs/auth-api-out.log',       // stdout goes here
      merge_logs: true,              // combine logs from all instances into one file
    },
  ],
};
```

---

## How it works — line by line

- `pm2 start server.js --name auth-api` tells PM2 to launch `server.js` as a background process, keep watching it, and label it `auth-api` so you can refer to it later instead of remembering a process ID.
- `-i max` (or `-i 4`) switches PM2 into **cluster mode**: instead of one process, PM2 forks several *identical* copies of your app — one per CPU core (or however many you specify) — and puts a built-in load balancer in front of them.
- PM2's load balancer sits on the same port your app listens on and hands each new incoming connection to one of the worker processes, round-robin style, so all cores stay busy under load.
- Because every worker is a separate OS process, a crash in one worker (unhandled exception, out-of-memory) does not take down the others — PM2 detects the dead worker and immediately forks a fresh replacement.
- `ecosystem.config.js` is just a plain JavaScript file exporting a config object — `module.exports` works exactly like any other CommonJS module, so you can use real code (env checks, dynamic instance counts) if needed.
- `pm2 reload <name>` performs a **zero-downtime reload**: PM2 starts one new worker with the updated code, waits until it is accepting connections, then kills one old worker — repeating this one-at-a-time swap across every instance so incoming requests are always served by *some* worker during the whole deploy.
- `pm2 monit` and `pm2 logs` give you visibility into a cluster the same way you'd watch one process — PM2 aggregates stats and logs across every worker automatically.
- `pm2 save` + `pm2 startup` together make sure that if the server reboots (power loss, maintenance), PM2 relaunches every saved process exactly as it was configured, with no manual intervention.

---

## Example 1 — basic

```js
// File: server.js
// A minimal HTTP server we will scale horizontally with PM2.

const http = require('http');       // core module — no framework needed for this demo

const PORT = process.env.PORT || 3000;   // read port from env, fall back to 3000

// Create a server that reports which process (worker) handled the request
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });

  // process.pid is different for every PM2 cluster worker —
  // this is how we PROVE requests are being spread across cores
  res.end(JSON.stringify({
    message: 'Request handled',
    workerPid: process.pid,
    path: req.url,
  }));
});

server.listen(PORT, () => {
  // Each cluster worker logs its own startup line with its own pid
  console.log(`Worker ${process.pid} listening on port ${PORT}`);
});
```

```bash
# Start 4 instances of this server in cluster mode
pm2 start server.js --name demo-server -i 4

# Hit the endpoint several times — notice "workerPid" changes between requests,
# proving PM2 is load-balancing across the 4 worker processes
curl http://localhost:3000/ping
curl http://localhost:3000/ping
curl http://localhost:3000/ping

# See all 4 workers listed with their own pid, CPU, and memory usage
pm2 list
```

---

## Example 2 — real world backend use case

```js
// File: ecosystem.config.js
// Production config for a real Express-based API with two separate services:
// one CPU-light API cluster, and one CPU-heavy worker running as a single instance.

module.exports = {
  apps: [
    {
      // ── Public-facing REST API — benefits hugely from cluster mode ──────────
      name: 'orders-api',
      script: './src/server.js',
      instances: 'max',                 // use every core for request throughput
      exec_mode: 'cluster',
      max_memory_restart: '500M',       // guard against slow memory leaks
      env: {
        NODE_ENV: 'production',
        PORT: 5000,
        DB_CONNECTION_STRING: process.env.DB_CONNECTION_STRING,  // pulled from real env, never hardcoded
      },
      error_file: './logs/orders-api-error.log',
      out_file: './logs/orders-api-out.log',
      // Wait up to 5s for the app to signal it's ready before considering reload successful
      listen_timeout: 5000,
      kill_timeout: 5000,               // grace period for in-flight requests during shutdown
    },
    {
      // ── Background email/report worker — must NOT run multiple copies ──────
      // Running this in cluster mode would send duplicate emails / reports,
      // so it stays as a single "fork mode" instance.
      name: 'report-worker',
      script: './src/workers/reportWorker.js',
      instances: 1,
      exec_mode: 'fork',                // single process, no load balancing needed
      autorestart: true,                // still restart automatically if it crashes
      cron_restart: '0 3 * * *',         // also force a clean restart daily at 3 AM
      env: {
        NODE_ENV: 'production',
      },
    },
  ],
};
```

```js
// File: src/server.js — the app referenced above
// Shows graceful shutdown so PM2's zero-downtime reload works correctly.

const express = require('express');
const app = express();

app.get('/orders/:orderId', (req, res) => {
  const orderId = req.params.orderId;         // real request param, not foo/bar
  res.json({ orderId, status: 'confirmed' }); // stub response for the example
});

const PORT = process.env.PORT || 5000;
const server = app.listen(PORT, () => {
  console.log(`orders-api worker ${process.pid} up on port ${PORT}`);
});

// PM2 sends SIGINT during `pm2 reload` — handling it lets in-flight
// requests finish before this worker exits, avoiding dropped connections
process.on('SIGINT', () => {
  console.log(`Worker ${process.pid} received SIGINT, closing gracefully...`);
  server.close(() => {
    process.exit(0);   // exit only after all open connections are drained
  });
});
```

```bash
# Deploy this config
pm2 start ecosystem.config.js

# Ship a code change, then reload with ZERO downtime —
# each orders-api worker is swapped one at a time; report-worker restarts normally
pm2 reload ecosystem.config.js

# Persist across server reboots
pm2 save
```

---

## Common mistakes

### Mistake 1 — Using `pm2 restart` instead of `pm2 reload` for deploys

```bash
# ❌ WRONG — restart kills ALL instances at once, then starts them again
# There is a window where the app is completely down — real downtime for real users
pm2 restart auth-api

# ✅ CORRECT — reload replaces instances one at a time in cluster mode
# At least one instance is always serving traffic during the swap
pm2 reload auth-api
```

### Mistake 2 — Running a stateful/singleton job in cluster mode

```js
// ❌ WRONG — a daily report/email job started with instances: 'max'
// means EVERY worker runs the cron logic, sending the same report N times
module.exports = {
  apps: [{
    name: 'report-worker',
    script: './src/workers/reportWorker.js',
    instances: 'max',       // 8 cores = report sent 8 times!
    exec_mode: 'cluster',
  }],
};

// ✅ CORRECT — singleton/cron-style jobs must run as exactly one instance
module.exports = {
  apps: [{
    name: 'report-worker',
    script: './src/workers/reportWorker.js',
    instances: 1,            // only one process ever runs this job
    exec_mode: 'fork',
  }],
};
```

### Mistake 3 — Forgetting graceful shutdown handling, breaking zero-downtime reload

```js
// ❌ WRONG — no SIGINT/SIGTERM handler means PM2 force-kills the process
// mid-request during `pm2 reload`, dropping active user requests
const server = app.listen(PORT);
// process just gets killed — no chance to finish in-flight work

// ✅ CORRECT — listen for PM2's shutdown signal and close the server cleanly
const server = app.listen(PORT);

process.on('SIGINT', () => {
  server.close(() => {           // stop accepting new connections,
    dbConnection.close(() => {    // finish existing DB queries,
      process.exit(0);            // then exit cleanly
    });
  });
});
```

---

## Practice exercises

### Exercise 1 — easy

Create a small Express (or raw `http`) server that listens on `process.env.PORT || 3000` and responds to `GET /whoami` with a JSON body containing `process.pid` and the current timestamp. Start it with PM2 in cluster mode using 2 instances (`pm2 start server.js --name whoami-api -i 2`). Call the endpoint at least 6 times with `curl` and confirm you see at least two different `pid` values in the responses, proving requests are load-balanced across instances.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write an `ecosystem.config.js` for a project with two apps:
1. `payments-api` — an Express server on port 6000, cluster mode, using all CPU cores, with `max_memory_restart` set to `300M`, separate `error_file` and `out_file` paths under a `./logs` folder, and `NODE_ENV: 'production'` in its env.
2. `queue-worker` — a background job processor (`./src/workers/queueWorker.js`) that must run as exactly one instance in fork mode, with `autorestart: true`.

Start both with `pm2 start ecosystem.config.js`, verify with `pm2 list` that `payments-api` shows multiple instances and `queue-worker` shows exactly one, then use `pm2 logs payments-api` to confirm logs are flowing.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `sessions-api` server that:
1. Stores an in-memory counter of requests handled per worker (`let requestCount = 0`, incremented on every request) and exposes it at `GET /stats` as `{ workerPid, requestCount }`.
2. Implements a graceful shutdown handler on `SIGINT` that logs `"Worker <pid> shutting down after handling <requestCount> requests"`, closes the HTTP server, and exits only once all in-flight requests finish.
3. Is configured in `ecosystem.config.js` with `instances: 'max'`, `exec_mode: 'cluster'`, and `kill_timeout: 4000` (so PM2 gives 4 seconds for graceful shutdown before force-killing).

Deploy it with `pm2 start ecosystem.config.js`, generate some load against `/stats` from multiple terminals with `curl` in a loop, then run `pm2 reload sessions-api` while the load is running and confirm (via `pm2 logs`) that each old worker logs its graceful shutdown message with a nonzero `requestCount` before the new worker takes over — with zero failed requests during the reload.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
INSTALL
  npm install -g pm2

CORE COMMANDS
  pm2 start server.js --name <name>        → start single instance
  pm2 start server.js --name <name> -i max → cluster mode, one worker per CPU core
  pm2 start server.js --name <name> -i 4   → cluster mode, exactly 4 workers
  pm2 list                                 → show all managed processes + status
  pm2 monit                                → live CPU/memory dashboard
  pm2 logs [name]                          → stream stdout/stderr
  pm2 stop <name>                          → stop but keep in PM2's list
  pm2 restart <name>                       → kill + restart ALL instances (brief downtime)
  pm2 reload <name>                        → ZERO-DOWNTIME rolling restart (cluster mode only)
  pm2 delete <name>                        → remove from PM2's managed list
  pm2 save                                 → persist current process list
  pm2 startup                              → generate OS command to relaunch PM2 on reboot

ecosystem.config.js KEY FIELDS
  name               → process label
  script             → entry file
  instances          → number or 'max' (all CPU cores)
  exec_mode          → 'cluster' (load-balanced) | 'fork' (single/independent process)
  env                → env vars injected into the process
  max_memory_restart → auto-restart if RAM usage exceeds this
  error_file/out_file→ log file paths
  kill_timeout       → ms PM2 waits for graceful shutdown before SIGKILL
  watch              → restart on file change (dev only — false in prod)

RESTART vs RELOAD
  restart → stops then starts ALL instances together      → brief full downtime
  reload  → swaps instances ONE AT A TIME                 → zero downtime (cluster mode only)

CLUSTER MODE RULES
  Stateless HTTP APIs        → instances: 'max', exec_mode: 'cluster'  ✓ safe to multiply
  Cron jobs / singleton work → instances: 1,     exec_mode: 'fork'     ✓ never duplicate
  In-memory state per worker → NOT shared across workers — use Redis/DB instead

GRACEFUL SHUTDOWN IS MANDATORY FOR ZERO-DOWNTIME RELOAD
  process.on('SIGINT', () => server.close(() => process.exit(0)))
```

---

## Connected topics

- **29 — cluster module** — PM2's cluster mode is a managed wrapper around Node's built-in `cluster` module; understanding `cluster` explains what PM2 is doing under the hood.
- **64 — Graceful shutdown** — zero-downtime `pm2 reload` only works correctly if your app handles `SIGINT`/`SIGTERM` and drains in-flight requests before exiting.
- **110 — Process management in production** — compares PM2 against systemd and covers auto-restart, log rotation, and monitoring in more depth than this topic alone.
