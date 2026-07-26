# 65 — Health checks and readiness probes

## What is this?

A health check is a small HTTP endpoint your server exposes (like `/health`) that answers one question: "are you okay?" Orchestrators like Kubernetes, load balancers, and monitoring tools call this endpoint every few seconds and decide whether to keep sending traffic to your app, restart it, or wait before adding it to the pool. Think of it like a bouncer checking a restaurant kitchen every few minutes — if the kitchen is on fire (dead), stop sending in orders and call the fire department (restart the pod); if the kitchen is just prepping and not ready for new orders yet (starting up), hold new customers at the door but don't shut the place down.

## Why does it matter for backend development?

In production, your app rarely runs as one process on one server — it runs as multiple replicas behind a load balancer or inside Kubernetes pods. Without health checks, the orchestrator has no way to know a pod has hung, lost its database connection, or is still warming up its cache. It would keep routing user requests to a broken instance, causing timeouts and errors for real users. Health checks let the platform automatically stop routing to sick instances, restart truly dead ones, and hold traffic back from instances that are still booting — all without a human watching a dashboard at 3 AM.

---

## Syntax / API

```js
// Minimal health endpoints using plain Node's http module (no framework needed)
const http = require('http');

// dbConnection would normally be a real database client (pg, mongoose, etc.)
let dbConnection = { isConnected: true };

const server = http.createServer((req, res) => {
  // /live — "is the process alive at all?" — almost never fails once the process is up
  if (req.url === '/live') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok' }));
    return;
  }

  // /ready — "can I actually serve traffic right now?" — checks real dependencies
  if (req.url === '/ready') {
    const isReady = dbConnection.isConnected; // check DB, cache, queue, etc.
    res.writeHead(isReady ? 200 : 503, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: isReady ? 'ready' : 'not ready' }));
    return;
  }

  // /health — combined, human-friendly summary (often used by simple setups, not K8s)
  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok', uptime: process.uptime() }));
    return;
  }

  res.writeHead(404);
  res.end();
});

server.listen(3000); // app listens for both real traffic and probe requests
```

---

## How it works — line by line

Kubernetes (and similar orchestrators) send periodic HTTP requests to your pod, not because a person clicks anything, but on a fixed timer. There are two distinct questions being asked, and they map to two distinct probes:

- **Liveness probe → hits `/live`.** This asks "is the process stuck or deadlocked?" If this fails repeatedly, Kubernetes concludes the process is broken beyond repair and **kills and restarts the container**. This check should be extremely cheap — just "the event loop is responding" — never check the database here, because a slow database would cause Kubernetes to restart a perfectly healthy app.
- **Readiness probe → hits `/ready`.** This asks "should I send you user traffic right now?" If this fails, Kubernetes **stops routing traffic to this pod** but does NOT restart it — it just waits and keeps checking. This is what you use during startup (still connecting to the DB) or during temporary trouble (DB connection dropped, cache warming up).
- **`/health`** is a convention from before Kubernetes' two-probe model existed — many simple apps, uptime monitors (like UptimeRobot or Pingdom), and load balancers still just want one combined endpoint. It's kept for compatibility and human debugging (`curl /health` to eyeball status).

The status code is what actually matters, not just the JSON body — Kubernetes only looks at whether the HTTP response was in the 200-299 range or not. A `503 Service Unavailable` is the conventional "not ready" signal.

---

## Example 1 — basic

```js
// File: src/health.js
// A basic health module you can plug into any Node app, framework or not.

const os = require('os'); // used to report basic host info, useful for debugging

// Tracks whether the app has finished its startup sequence
let appIsReady = false;

// Call this once your DB pool, cache client, etc. have all connected successfully
function markAppReady() {
  appIsReady = true; // flip the flag — readiness probe will now succeed
}

// Liveness check — deliberately dumb, only proves the event loop is responsive
function getLivenessStatus() {
  return { status: 'ok' }; // if this function even runs, the process is alive
}

// Readiness check — reflects real startup/dependency state
function getReadinessStatus() {
  return {
    status: appIsReady ? 'ready' : 'starting',
    uptimeSeconds: Math.round(process.uptime()), // how long the process has run
    hostname: os.hostname(),                      // which pod/container answered
  };
}

module.exports = { markAppReady, getLivenessStatus, getReadinessStatus };
```

```js
// File: src/server.js
const http = require('http');
const { markAppReady, getLivenessStatus, getReadinessStatus } = require('./health');

const server = http.createServer((req, res) => {
  if (req.url === '/live') {
    const body = getLivenessStatus();                 // always { status: 'ok' } if we got here
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(body));
    return;
  }

  if (req.url === '/ready') {
    const body = getReadinessStatus();                // reflects real app state
    const code = body.status === 'ready' ? 200 : 503;  // 503 tells K8s "hold traffic"
    res.writeHead(code, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(body));
    return;
  }

  res.writeHead(404);
  res.end();
});

server.listen(3000, () => {
  console.log('Server listening on port 3000');

  // Simulate a slow startup task (e.g. connecting to a database)
  setTimeout(() => {
    markAppReady(); // only now does /ready start returning 200
    console.log('App marked as ready');
  }, 3000);
});
```

---

## Example 2 — real world backend use case

```js
// File: src/health/checks.js
// A production-style health check that verifies REAL dependencies before saying "ready".
// This is the pattern used behind Express apps deployed to Kubernetes.

const express = require('express');
const router = express.Router();

// These would be real clients in a real app (pg.Pool, ioredis, amqplib, etc.)
let dbConnection = null;   // set by app startup once pg pool connects
let cacheConnection = null; // set once Redis client connects
let isShuttingDown = false; // set to true during graceful shutdown (see Topic 64)

function registerDependencies({ db, cache }) {
  dbConnection = db;
  cacheConnection = cache;
}

function notifyShuttingDown() {
  isShuttingDown = true; // readiness must fail immediately once shutdown starts
}

// Liveness — cheap, no I/O, no dependency calls. Just "is the process responsive?"
router.get('/live', (req, res) => {
  res.status(200).json({ status: 'ok' });
});

// Readiness — checks every critical dependency with a short timeout each
router.get('/ready', async (req, res) => {
  if (isShuttingDown) {
    // Draining: tell the load balancer to stop sending new requests NOW
    return res.status(503).json({ status: 'shutting_down' });
  }

  const checks = {};
  let allHealthy = true;

  // Check database with a fast query and a timeout so a hung DB doesn't hang the probe
  try {
    await Promise.race([
      dbConnection.query('SELECT 1'),                       // cheap liveness query
      new Promise((_, reject) => setTimeout(() => reject(new Error('db timeout')), 2000)),
    ]);
    checks.database = 'ok';
  } catch (err) {
    checks.database = 'unreachable';
    allHealthy = false; // any failed dependency makes the whole pod "not ready"
  }

  // Check cache connection similarly
  try {
    await cacheConnection.ping();                            // Redis PING command
    checks.cache = 'ok';
  } catch (err) {
    checks.cache = 'unreachable';
    allHealthy = false;
  }

  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'ready' : 'not_ready',
    checks,                                                   // per-dependency breakdown for debugging
  });
});

// /health — combined human-friendly view, used by monitoring dashboards, not K8s
router.get('/health', (req, res) => {
  res.status(200).json({
    status: 'ok',
    uptimeSeconds: Math.round(process.uptime()),
    memoryMB: Math.round(process.memoryUsage().rss / 1024 / 1024),
    nodeVersion: process.version,
  });
});

module.exports = { router, registerDependencies, notifyShuttingDown };
```

```yaml
# Kubernetes deployment snippet showing how these endpoints get wired in
livenessProbe:
  httpGet:
    path: /live
    port: 3000
  initialDelaySeconds: 5   # wait 5s after container start before first check
  periodSeconds: 10        # check every 10 seconds
  failureThreshold: 3      # 3 consecutive failures → restart the container

readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 2
  periodSeconds: 5         # check more often — this controls traffic routing
  failureThreshold: 2      # 2 consecutive failures → remove from load balancer
```

---

## Common mistakes

### Mistake 1 — Checking the database in the liveness probe

```js
// ❌ WRONG — liveness checks a DB. A slow/overloaded DB now kills healthy pods.
router.get('/live', async (req, res) => {
  await dbConnection.query('SELECT 1'); // if the DB is slow, this probe times out
  res.status(200).json({ status: 'ok' });
  // Kubernetes sees repeated failures → restarts a perfectly fine app process,
  // which makes a DB outage MUCH worse by mass-restarting every pod at once.
});

// ✅ CORRECT — liveness only proves the process itself is responsive
router.get('/live', (req, res) => {
  res.status(200).json({ status: 'ok' }); // no I/O, no dependency calls
});
// Dependency health belongs ONLY in /ready, never in /live
```

### Mistake 2 — Returning 200 even when the app is not actually ready

```js
// ❌ WRONG — always returns 200, so Kubernetes routes traffic before the DB connects
router.get('/ready', (req, res) => {
  res.status(200).json({ status: 'ready' }); // hardcoded, ignores real state
  // Users hit the app during the first few seconds of startup and get DB errors
});

// ✅ CORRECT — reflects a real readiness flag set only after startup finishes
let appIsReady = false;
dbConnection.connect().then(() => { appIsReady = true; }); // set once truly connected

router.get('/ready', (req, res) => {
  res.status(appIsReady ? 200 : 503).json({ status: appIsReady ? 'ready' : 'starting' });
});
```

### Mistake 3 — Not failing readiness during graceful shutdown

```js
// ❌ WRONG — on SIGTERM the app starts closing connections, but /ready still says 200
process.on('SIGTERM', () => {
  dbConnection.end();     // closing the DB pool
  server.close();         // closing the HTTP server
  // /ready keeps returning 200 for the few seconds it takes K8s to notice the pod
  // is terminating — in-flight requests during that window get connection errors
});

// ✅ CORRECT — flip readiness to "not ready" the INSTANT shutdown begins,
// then wait a moment before actually closing connections (see Topic 64)
let isShuttingDown = false;

process.on('SIGTERM', async () => {
  isShuttingDown = true;                          // /ready now returns 503 immediately
  await new Promise((r) => setTimeout(r, 5000));  // give K8s time to stop routing traffic
  dbConnection.end();                              // now safe to close connections
  server.close(() => process.exit(0));
});

router.get('/ready', (req, res) => {
  res.status(isShuttingDown ? 503 : 200).json({ status: isShuttingDown ? 'shutting_down' : 'ready' });
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a plain Node `http` server (no framework) with two endpoints:
1. `GET /live` — always returns HTTP 200 with JSON `{ status: 'ok' }`
2. `GET /ready` — returns HTTP 200 with `{ status: 'ready' }` only after 4 seconds have passed since the server started; before that, it should return HTTP 503 with `{ status: 'starting' }`

Test it by starting the server and curling `/ready` a few times in the first few seconds, then again after 4 seconds have passed, to confirm the status code changes.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an Express app with a `dbConnection` object that simulates a database client with a `connect()` method (resolves after a 2 second delay) and a `query(sql)` method (throws an error if not connected, otherwise resolves).

1. On server startup, call `dbConnection.connect()` and only mark the app "ready" once it resolves
2. Add a `/ready` endpoint that calls `dbConnection.query('SELECT 1')` and returns 200 if it succeeds, 503 if it throws
3. Add a `/live` endpoint that never touches `dbConnection` at all
4. Add an admin-only route `POST /simulate-db-down` that sets a flag making `dbConnection.query()` throw on demand, so you can test `/ready` flipping to 503 without restarting the server

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a reusable `HealthMonitor` class for a backend with multiple dependencies (a database, a cache, and a message queue — you can simulate all three with fake async clients that randomly fail ~10% of the time).

Requirements:
1. Constructor accepts a list of named dependency checkers: `new HealthMonitor([{ name: 'database', check: fn }, { name: 'cache', check: fn }, ...])`, where each `check` is an async function that resolves if healthy and rejects if not
2. A `checkReadiness()` method that runs all checks concurrently with a 2-second timeout per check, and returns `{ overallStatus: 'ready' | 'not_ready', checks: { database: 'ok'|'failed', cache: 'ok'|'failed', ... } }`
3. A `checkLiveness()` method that just returns `{ status: 'ok' }` — no dependency calls at all
4. Wire it into an Express app exposing `/live`, `/ready`, and `/health` (where `/health` reuses `checkReadiness()` output but always returns 200, purely informational)
5. Support a `notifyShuttingDown()` method that, once called, makes `checkReadiness()` immediately return `not_ready` with `{ shutdown: true }` regardless of the actual dependency states

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
THREE COMMON ENDPOINTS
  /live    → "is the process alive?"        — no I/O, no dependency checks, almost never fails
  /ready   → "can I serve traffic now?"     — checks DB/cache/queue, fails during startup/shutdown
  /health  → combined, human-friendly       — used by uptime monitors, dashboards, curl checks

WHAT KUBERNETES DOES WITH EACH PROBE
  livenessProbe fails N times  → RESTARTS the container (assumes it's stuck/deadlocked)
  readinessProbe fails N times → REMOVES pod from load balancer, does NOT restart, keeps polling

STATUS CODES THAT MATTER
  200-299  → healthy / ready     (any 2xx works, 200 is conventional)
  503      → not ready           (conventional "temporarily unavailable" code)
  Body content is for humans/logs — orchestrators only read the status code

RULES OF THUMB
  /live   → never call the database, cache, or any network dependency
  /ready  → DO call dependencies, but with a short timeout per check (1-2s)
  /health → safe to include extra info: uptime, memory, version — for debugging only

GRACEFUL SHUTDOWN INTEGRATION (see Topic 64)
  On SIGTERM: flip readiness to "not ready" FIRST, wait a few seconds,
  THEN close DB/server connections — this drains traffic before shutdown

TYPICAL K8s PROBE CONFIG
  initialDelaySeconds  → grace period before the first check after container start
  periodSeconds        → how often to check
  failureThreshold     → consecutive failures needed before acting
  timeoutSeconds        → how long to wait for a single probe response

COMMON PITFALLS
  Checking DB in /live           → DB hiccup restarts every pod at once
  Hardcoding /ready to 200       → traffic hits app before it's actually ready
  Ignoring shutdown state        → in-flight requests fail during pod termination
  No timeout on dependency check → one slow dependency hangs the whole probe
```

---

## Connected topics

- **64 — Graceful shutdown** — the readiness flag flipping to "not ready" on `SIGTERM` is the mechanism that makes graceful shutdown actually drain traffic safely.
- **62 — Centralized error handling** — the `AppError` classification pattern is the same mindset used to decide what makes a dependency check "failed" versus recoverable.
- **66 — Circuit breaker pattern** — circuit breakers and readiness checks solve related problems: both stop sending requests toward a dependency (or of a pod) that is currently unhealthy.
