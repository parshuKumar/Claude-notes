# 29 — cluster module

## What is this?

The `cluster` module lets a single Node.js application spawn multiple copies of itself — one per CPU core — so it can use the entire machine instead of just one core. Node.js is single-threaded per process, so out of the box an 8-core server runs your app on only 1 core while 7 sit idle. `cluster` fixes this by forking worker processes that all listen on the same port, with the operating system (or a built-in scheduler) distributing incoming connections between them. Think of it like a restaurant with one cashier (default Node) versus opening 8 identical checkout counters (cluster) that all serve customers from the same queue — same menu, same building, far more throughput.

## Why does it matter for backend development?

A production API server is judged by requests-per-second and how it survives a crash. Without clustering, your entire multi-core cloud instance (say a 4 vCPU box you're paying for) only ever exercises one vCPU — you're paying for capacity you never use. Clustering is the cheapest, most direct way to multiply throughput: forking 4 workers on a 4-core box can roughly 4x your handled requests per second with almost no code change. It's also a resilience mechanism — if one worker crashes on an unhandled exception, the master process detects the death and can immediately fork a replacement, so the service as a whole stays up instead of going fully down. This is the exact mechanism tools like PM2's cluster mode (Topic 73) automate for you in production.

---

## Syntax / API

```js
// cluster module — built into Node.js, no install needed
const cluster = require('cluster');

// os module — used to find how many CPU cores the machine has
const os = require('os');

// http module — used to build the actual server each worker runs
const http = require('http');

// cluster.isPrimary is true only in the master/parent process
// (older Node versions used cluster.isMaster — isPrimary is the modern name)
if (cluster.isPrimary) {
  // Number of CPU cores available on this machine
  const numCPUs = os.cpus().length;

  // Fork one worker process per CPU core
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork(); // spawns a new Node.js process running this same file
  }

  // Fires whenever a worker process dies (crash, killed, etc.)
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (code: ${code})`);
    cluster.fork(); // replace the dead worker to keep capacity constant
  });

} else {
  // This branch runs inside EVERY worker process, not the master
  http.createServer((req, res) => {
    res.end(`Handled by worker PID ${process.pid}\n`); // prove which worker served this
  }).listen(3000); // all workers bind to the SAME port 3000 — cluster handles routing
}
```

---

## How it works — line by line

`require('cluster')` loads the module. `require('os')` gives access to `os.cpus()`, which returns an array with one entry per logical CPU core — `.length` tells us how many workers make sense to create.

`cluster.isPrimary` is `true` only in the original process you started with `node server.js`. Everything inside that `if` block only ever runs once, in the master process — it never runs the actual server logic.

The `for` loop calls `cluster.fork()` once per CPU core. Each call to `fork()` creates a brand-new operating system process that re-runs the **entire file from the top** — but in that new process, `cluster.isPrimary` is `false`, so it falls into the `else` branch instead.

`cluster.on('exit', ...)` registers a listener on the master process only. Whenever any worker process exits — crash, manual kill, out-of-memory — this callback fires with the dead worker's info. Calling `cluster.fork()` again inside it births a replacement, so the pool of workers is kept at a constant size automatically.

The `else` branch is what actually runs in each worker: it creates a normal `http.createServer()` and calls `.listen(3000)`. Here is the key trick — every worker calls `.listen()` on the exact same port number, and normally that would throw `EADDRINUSE`. Cluster intercepts this: the master process holds the real listening socket, and workers just receive connections handed to them. On Linux, the kernel typically also participates directly via `SO_REUSEPORT`-style load balancing depending on the scheduling policy in use.

### How connections are distributed

```
Client requests  ──►  Master process (holds the socket, port 3000)
                          │
              round-robin (default on Linux/macOS)
                          │
        ┌─────────┬───────┴───────┬─────────┐
        ▼         ▼               ▼         ▼
    Worker 1   Worker 2        Worker 3   Worker 4
   (own event  (own event     (own event  (own event
    loop, own   loop, own      loop, own   loop, own
    memory)     memory)        memory)     memory)
```

Each worker is a **fully separate OS process** — its own V8 instance, own heap, own event loop. They share nothing in memory (no shared variables, no shared in-memory cache) unless you explicitly wire up something like Redis. This is why the diagram matters: scaling with cluster buys you CPU parallelism, not shared state — that tradeoff is the single most important thing to internalize about this module.

---

## Example 1 — basic

```js
// File: src/basic-cluster.js

const cluster = require('cluster'); // core module for forking worker processes
const os      = require('os');      // core module to detect CPU count
const http    = require('http');    // core module to build the HTTP server

if (cluster.isPrimary) {
  // process.pid → the process ID of the master, useful for logging
  console.log(`Master process started, PID: ${process.pid}`);

  const numCPUs = os.cpus().length; // e.g. 8 on an 8-core machine
  console.log(`Forking ${numCPUs} workers...`);

  // Create one worker per CPU core
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork(); // each call spawns a separate Node.js process
  }

  // Log every time a worker successfully comes online
  cluster.on('online', (worker) => {
    console.log(`Worker ${worker.process.pid} is online`);
  });

  // Replace any worker that dies, so capacity never drops
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} exited (signal: ${signal})`);
    cluster.fork(); // spin up a fresh replacement worker
  });

} else {
  // Worker code — this block runs once inside EACH forked process
  http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' }); // standard response header
    res.end(`Response from worker PID ${process.pid}\n`);  // identify which worker answered
  }).listen(3000, () => {
    console.log(`Worker ${process.pid} listening on port 3000`);
  });
}

// Run: node src/basic-cluster.js
// Then curl http://localhost:3000 repeatedly — the PID in the response
// cycles between different worker processes (round-robin scheduling).
```

---

## Example 2 — real world backend use case

```js
// File: src/server-cluster.js
// A production-style pattern: cluster an Express-like API server,
// track worker health, and shut down gracefully on SIGTERM.

const cluster = require('cluster');
const os      = require('os');
const http    = require('http');

// Reserve one core for the OS/other processes on small machines;
// use all cores in production containers with dedicated CPU limits.
const WORKER_COUNT = process.env.WEB_CONCURRENCY
  ? parseInt(process.env.WEB_CONCURRENCY, 10) // let ops override via env var
  : os.cpus().length;

if (cluster.isPrimary) {
  console.log(`[master] PID ${process.pid} starting ${WORKER_COUNT} workers`);

  const workers = new Map(); // track worker PID → worker object for visibility

  function spawnWorker() {
    const worker = cluster.fork(); // start a new worker process
    workers.set(worker.process.pid, worker);
    return worker;
  }

  for (let i = 0; i < WORKER_COUNT; i++) {
    spawnWorker(); // start with the full pool of workers
  }

  // If a worker dies unexpectedly (not a deliberate shutdown), replace it
  cluster.on('exit', (worker, code, signal) => {
    workers.delete(worker.process.pid); // remove the dead entry from tracking

    if (!worker.exitedAfterDisconnect) {
      // exitedAfterDisconnect is false → this was a CRASH, not a graceful exit
      console.error(`[master] Worker ${worker.process.pid} crashed (code ${code}), respawning`);
      spawnWorker(); // keep total worker count constant
    } else {
      console.log(`[master] Worker ${worker.process.pid} shut down gracefully`);
    }
  });

  // Graceful shutdown of the WHOLE cluster on deploy/restart signals
  process.on('SIGTERM', () => {
    console.log('[master] SIGTERM received, shutting down all workers...');
    for (const worker of workers.values()) {
      worker.disconnect(); // ask worker to stop accepting new connections and exit
    }
  });

} else {
  // ── Worker process: the actual API server ──────────────────────────────
  const dbConnection = { query: async (sql) => ({ rows: [] }) }; // stub DB client

  const server = http.createServer(async (req, res) => {
    if (req.url === '/health') {
      // Health check endpoint — used by load balancers (see Topic 65)
      res.writeHead(200, { 'Content-Type': 'application/json' });
      return res.end(JSON.stringify({ status: 'ok', pid: process.pid }));
    }

    if (req.url === '/users' && req.method === 'GET') {
      const { rows } = await dbConnection.query('SELECT * FROM users'); // simulate DB call
      res.writeHead(200, { 'Content-Type': 'application/json' });
      return res.end(JSON.stringify({ users: rows, servedBy: process.pid }));
    }

    res.writeHead(404); // unknown route
    res.end('Not found');
  });

  server.listen(3000, () => {
    console.log(`[worker ${process.pid}] listening on port 3000`);
  });

  // Each worker also listens for its OWN disconnect signal from the master
  process.on('disconnect', () => {
    console.log(`[worker ${process.pid}] disconnected, closing server`);
    server.close(() => process.exit(0)); // stop accepting new requests, finish in-flight ones
  });
}

// Run: node src/server-cluster.js
// Load test with autocannon (Topic 72) and compare throughput against
// running the same server WITHOUT cluster — the difference scales with core count.
```

---

## Common mistakes

### Mistake 1 — Storing state in memory and expecting workers to share it

```js
// ❌ WRONG — each worker has its own separate memory; this cache is NOT shared
const sessionCache = new Map(); // exists independently inside EACH worker

http.createServer((req, res) => {
  const sessionId = req.headers['x-session-id'];
  sessionCache.set(sessionId, { userId: 'user_42' }); // only visible to THIS worker
  res.end('ok');
}).listen(3000);
// A later request routed to a different worker will NOT find this session — random bugs.

// ✅ CORRECT — use an external shared store (Redis) that all workers can reach
const redisClient = require('./redisClient'); // shared connection, see Topic 92

http.createServer(async (req, res) => {
  const sessionId = req.headers['x-session-id'];
  await redisClient.set(sessionId, JSON.stringify({ userId: 'user_42' })); // shared across all workers
  res.end('ok');
}).listen(3000);
```

### Mistake 2 — Forking more workers than the machine has cores

```js
// ❌ WRONG — hardcoding a huge worker count wastes memory and causes
// excessive context switching; more workers than cores does NOT mean more speed
for (let i = 0; i < 32; i++) {
  cluster.fork(); // 32 processes fighting over 4 cores — slower, not faster
}

// ✅ CORRECT — base the worker count on the actual CPU cores available
const os = require('os');
const numCPUs = os.cpus().length; // matches real parallel capacity

for (let i = 0; i < numCPUs; i++) {
  cluster.fork(); // one worker per core — maximizes throughput without overhead
}
```

### Mistake 3 — Forgetting to respawn a crashed worker (silent capacity loss)

```js
// ❌ WRONG — no 'exit' handler means a crashed worker is gone forever;
// your cluster silently shrinks from 8 workers to 7, 6, 5... over time
if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();
  // missing cluster.on('exit', ...) — dead workers are never replaced
}

// ✅ CORRECT — always listen for 'exit' and fork a replacement
if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();

  cluster.on('exit', (worker, code, signal) => {
    console.error(`Worker ${worker.process.pid} died, respawning...`);
    cluster.fork(); // keeps the pool at full strength indefinitely
  });
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a clustered HTTP server where:
1. The master process forks exactly 4 workers (hardcode `4`, don't use `os.cpus().length` for this one).
2. Each worker runs an HTTP server on port `4000` that responds with a plain-text message containing its own `process.pid`.
3. The master logs a message every time a worker comes online (`cluster.on('online', ...)`).
4. Start the server, then make 8 requests with `curl` in a loop and observe the PIDs in the responses cycling between the 4 workers.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a clustered server with basic crash resilience:
1. Fork `os.cpus().length` workers.
2. Each worker serves `GET /crash` by deliberately throwing an uncaught error (to simulate a real bug) and everything else normally on port `5000`.
3. In the master, listen for `'exit'` and log the dead worker's PID, then immediately fork a replacement so the total worker count never drops.
4. Add a `GET /status` route (served by workers) that returns JSON with `{ pid: process.pid, uptime: process.uptime() }` so you can verify a "young" replacement worker has a small uptime after a crash.
5. Test it: hit `/crash` a few times, then hit `/status` repeatedly and confirm one of the PIDs is new with a near-zero uptime.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a production-style clustered API with graceful shutdown and shared session storage via a mocked "external store" module:

1. Create a `store.js` module that simulates Redis with an in-memory object at module scope, exposing `async get(key)` and `async set(key, value)` — this file will be `require`d fresh inside each worker (simulating what a real Redis client connection would provide, since actual Redis would truly be shared, unlike this in-process mock).
2. In the master: read the desired worker count from `process.env.WEB_CONCURRENCY`, defaulting to `os.cpus().length` if unset. Track all live workers in a `Map` keyed by PID.
3. In the master: on `SIGTERM`, call `.disconnect()` on every tracked worker and only call `process.exit(0)` once **all** workers have exited (track this by listening for `'exit'` and checking if the `Map` is empty).
4. In each worker: build an HTTP server with `POST /session` (accepts a `userId` in the request body via JSON, generates a fake `sessionId` using `crypto.randomUUID()`, stores it) and `GET /session/:id` (looks it up and returns it, or 404).
5. In each worker: listen for `process.on('disconnect', ...)` and close the HTTP server gracefully before letting the process exit.
6. Distinguish crash-exits from graceful-exits in the master's `'exit'` handler using `worker.exitedAfterDisconnect`, only respawning on real crashes.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE API
  cluster.isPrimary          → true in the master process (modern name)
  cluster.isMaster            → deprecated alias for isPrimary, still works
  cluster.fork()              → spawns a new worker process, re-runs the file
  cluster.workers              → object of all live worker instances, keyed by id
  worker.process.pid           → OS process ID of a specific worker
  worker.disconnect()          → gracefully ask a worker to stop and exit
  worker.kill()                → forcefully terminate a worker immediately
  worker.exitedAfterDisconnect → true if exit was graceful, false if it was a crash

EVENTS (on the cluster object, master process only)
  'online'   → fires when a worker starts and is ready
  'exit'     → fires when a worker process ends (crash OR graceful)
  'disconnect' → fires when a worker's IPC channel disconnects
  'message'  → fires when a worker sends an IPC message to master

WHAT IS SHARED vs NOT SHARED BETWEEN WORKERS
  Shared    → the listening PORT (all workers can bind to it via cluster)
  NOT shared → memory, variables, in-memory caches/Maps, module-level state
  Fix for shared state → Redis, a database, or another external store

WHEN TO USE
  ✔ CPU-bound-ish HTTP workloads that need more raw throughput
  ✔ Improving resilience — a crashed worker doesn't take down the whole app
  ✘ Not a substitute for horizontal scaling across multiple MACHINES
  ✘ Not for sharing large in-memory state between requests

RULE OF THUMB
  Worker count = os.cpus().length (override via env var like WEB_CONCURRENCY)
  Always add a cluster.on('exit', ...) handler to respawn crashed workers
  Never assume in-memory data set by one worker is visible to another

PRODUCTION NOTE
  Tools like PM2 (Topic 73) wrap this exact cluster module —
  `pm2 start app.js -i max` does the fork-per-core pattern for you automatically.
```

---

## Connected topics

- **18 — os module** — `os.cpus()` is how you determine the correct number of workers to fork for the current machine
- **27 — child_process module** — `cluster` is built on top of `child_process.fork()` internally; understanding fork() explains how cluster spawns workers
- **73 — Horizontal scaling with PM2** — PM2's `cluster` mode automates exactly this pattern (fork-per-core, auto-restart on crash) without writing the master/worker code by hand
