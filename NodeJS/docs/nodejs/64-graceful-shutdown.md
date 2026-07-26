# 64 — Graceful shutdown

## What is this?

Graceful shutdown is the practice of letting a running server finish its current work — in-flight HTTP requests, open database queries, queued jobs — before the process actually exits, instead of dying instantly and cutting everything off mid-air. Node.js gives you a hook into the operating system signals (`SIGTERM`, `SIGINT`) that ask a process to stop, so you can intercept that request, stop accepting new work, drain what's already running, clean up resources, and only then exit. Think of it like a restaurant closing for the night — the manager doesn't throw customers out mid-meal when closing time hits; they stop seating new customers, let existing tables finish eating, then lock the doors.

## Why does it matter for backend development?

Every production deployment, container restart, or auto-scaling event sends your Node process a shutdown signal — Kubernetes does it on every pod rollout, PM2 does it on every reload, Docker does it on every `docker stop`. If your app doesn't handle that signal, requests that were mid-flight get their TCP connection killed instantly, database transactions get abandoned half-committed, and users see failed requests during every single deploy. A backend developer who implements graceful shutdown correctly gets **zero-downtime deploys** — the load balancer stops sending new traffic, the server finishes what it has, and it exits cleanly with no dropped requests. This is one of the most common things interviewers ask about and one of the most common things missing from junior-built APIs.

---

## Syntax / API

```js
// Node.js listens for OS signals via process.on() — covered in Topic 05
const server = require('http').createServer((req, res) => {
  res.end('ok');
});

server.listen(3000);

// SIGTERM — the "polite" signal sent by Docker, Kubernetes, PM2 on shutdown/restart
process.on('SIGTERM', () => {
  console.log('SIGTERM received — starting graceful shutdown');
  shutdown();
});

// SIGINT — sent when you press Ctrl+C in the terminal during local development
process.on('SIGINT', () => {
  console.log('SIGINT received — starting graceful shutdown');
  shutdown();
});

function shutdown() {
  // server.close() stops accepting NEW connections but lets existing ones finish
  server.close((err) => {
    if (err) {
      console.error('Error during shutdown:', err);
      process.exit(1);   // exit code 1 = shutdown failed
    }
    console.log('All connections drained — exiting cleanly');
    process.exit(0);      // exit code 0 = shutdown succeeded
  });

  // Safety net — if connections never drain (e.g. a stuck request), force-kill
  // after a timeout so the process doesn't hang forever
  setTimeout(() => {
    console.error('Forced shutdown — connections did not drain in time');
    process.exit(1);
  }, 10000).unref();   // unref() lets Node exit early if shutdown finishes first
}
```

---

## How it works — line by line

1. `require('http').createServer(...)` — creates the HTTP server that answers incoming requests, same as in any Node app.
2. `server.listen(3000)` — starts accepting connections on port 3000.
3. `process.on('SIGTERM', callback)` — registers a listener for the `SIGTERM` signal. This is the signal container orchestrators (Kubernetes, Docker, PM2) send when they want your app to stop — it means "please shut down, you have some time before we force-kill you."
4. `process.on('SIGINT', callback)` — registers a listener for `SIGINT`, the signal sent when a human presses `Ctrl+C` in a terminal. Handling both means the same shutdown logic works locally and in production.
5. Inside `shutdown()`, `server.close(callback)` tells the HTTP server: stop accepting brand new incoming connections, but keep serving any request that is already in progress. The callback only fires once every existing connection has finished and closed on its own.
6. Inside that callback, `process.exit(0)` ends the process with exit code `0`, meaning "everything shut down successfully." Exit codes are covered in Topic 05 (`process.exit()`).
7. The `setTimeout(..., 10000)` is a safety net — if some connection never finishes (a hung request, a stuck stream), the server would wait forever. This timer force-exits after 10 seconds no matter what.
8. `.unref()` on that timer tells Node "don't let this timer alone keep the process alive" — if shutdown finishes cleanly before the 10 seconds are up, the process can still exit immediately instead of waiting for the timer.

---

## Example 1 — basic

```js
// File: src/basic-shutdown.js
const http = require('http');

// A minimal server that simulates a slow request with setTimeout
const server = http.createServer((req, res) => {
  // Simulate work that takes 3 seconds — like a slow DB query
  setTimeout(() => {
    res.end('Request handled\n');
  }, 3000);
});

server.listen(3000, () => {
  console.log('Server listening on port 3000');
});

// Track whether we are already shutting down to avoid running shutdown twice
let isShuttingDown = false;

function gracefulShutdown(signal) {
  // Guard against Ctrl+C being pressed twice, or both signals firing
  if (isShuttingDown) return;
  isShuttingDown = true;

  console.log(`\n${signal} received — no longer accepting new connections`);

  // Stop accepting new connections; existing ones (like the 3s request) keep running
  server.close(() => {
    console.log('All in-flight requests finished — exiting now');
    process.exit(0);
  });
}

// Wire both signals to the same handler
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

---

## Example 2 — real world backend use case

```js
// File: src/server.js
// A realistic Express-style API server with a database pool, an in-flight
// request counter, and a health check endpoint that reports shutdown state —
// exactly the pattern used before deploying behind a load balancer.

const http = require('http');

// Simulated database connection pool — in a real app this is `pg.Pool` (Topic 90)
const dbConnection = {
  activeQueries: 0,
  async query(sql) {
    this.activeQueries++;
    await new Promise((resolve) => setTimeout(resolve, 200)); // simulate query time
    this.activeQueries--;
    return { rows: [] };
  },
  async close() {
    console.log('Closing database connection pool...');
    await new Promise((resolve) => setTimeout(resolve, 100)); // simulate cleanup
    console.log('Database connection pool closed');
  },
};

// Track how many requests are currently in flight
let activeRequests = 0;
let isShuttingDown = false;

const server = http.createServer(async (req, res) => {
  // Reject new work immediately once shutdown has begun
  if (isShuttingDown) {
    res.writeHead(503, { 'Connection': 'close' });   // 503 = Service Unavailable
    res.end(JSON.stringify({ error: 'Server is shutting down, try again shortly' }));
    return;
  }

  // Health check endpoint — flips to unhealthy during shutdown so a load
  // balancer or Kubernetes readiness probe stops routing traffic here (Topic 65)
  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok', activeRequests }));
    return;
  }

  activeRequests++;
  try {
    // Simulate a real handler that reads a userId and hits the database
    const userId = 'user_42';
    await dbConnection.query(`SELECT * FROM users WHERE id = '${userId}'`);

    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ userId, status: 'fetched' }));
  } finally {
    activeRequests--;
  }
});

server.listen(3000, () => {
  console.log(`Server running with PID ${process.pid}`);
});

async function shutdown(signal) {
  if (isShuttingDown) return;
  isShuttingDown = true;
  console.log(`\n${signal} received — draining ${activeRequests} in-flight request(s)`);

  // Stop accepting new connections
  server.close(async () => {
    console.log('HTTP server closed — no more sockets open');

    // Clean up resources AFTER the server is done serving requests
    await dbConnection.close();

    console.log('Cleanup complete — exiting');
    process.exit(0);
  });

  // Force-exit if draining takes longer than 15 seconds — prevents a stuck
  // deploy pipeline from hanging on an unresponsive process forever
  setTimeout(() => {
    console.error('Shutdown timed out — forcing exit');
    process.exit(1);
  }, 15000).unref();
}

process.on('SIGTERM', () => shutdown('SIGTERM'));   // sent by Docker/Kubernetes/PM2
process.on('SIGINT', () => shutdown('SIGINT'));     // sent by Ctrl+C locally
```

---

## Common mistakes

### Mistake 1 — Only handling SIGINT, ignoring SIGTERM

```js
// ❌ WRONG — works when you Ctrl+C locally, but Kubernetes/Docker/PM2 send
// SIGTERM in production, not SIGINT — so this code never runs in prod
process.on('SIGINT', () => {
  server.close(() => process.exit(0));
});
// In production the process gets killed with zero cleanup, every single deploy

// ✅ CORRECT — handle both, since each environment sends a different signal
process.on('SIGTERM', () => shutdown('SIGTERM'));   // production (Docker/K8s/PM2)
process.on('SIGINT', () => shutdown('SIGINT'));     // local development (Ctrl+C)
```

### Mistake 2 — Calling process.exit() immediately instead of draining

```js
// ❌ WRONG — kills the process instantly, severing any request currently
// mid-response and abandoning any in-progress database write
process.on('SIGTERM', () => {
  console.log('Shutting down');
  process.exit(0);   // in-flight requests get their socket ripped away, users see errors
});

// ✅ CORRECT — server.close() waits for existing connections to finish first
process.on('SIGTERM', () => {
  server.close(() => {
    console.log('All requests finished');
    process.exit(0);
  });
});
```

### Mistake 3 — No forced-timeout safety net

```js
// ❌ WRONG — if even one connection never closes (a hung request, a websocket
// that's still open, a stream that never ends), server.close()'s callback
// never fires and the process hangs forever, blocking your deploy pipeline
process.on('SIGTERM', () => {
  server.close(() => process.exit(0));   // may wait forever
});

// ✅ CORRECT — always cap shutdown with a hard timeout as a safety net
process.on('SIGTERM', () => {
  server.close(() => process.exit(0));

  setTimeout(() => {
    console.error('Graceful shutdown timed out — forcing exit');
    process.exit(1);
  }, 10000).unref();   // force-exit after 10s no matter what
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a simple HTTP server on port `4000` that responds `Hello` to every request. Add handlers for both `SIGINT` and `SIGTERM` that:
1. Log a message saying which signal was received
2. Call `server.close()` and log `"Server closed"` once it finishes
3. Exit the process with code `0`

Test it by running the server and pressing `Ctrl+C`.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an HTTP server that tracks the number of active in-flight requests using a counter (increment when a request starts, decrement when it ends). Add:
1. A `/health` route that returns `{ status: 'ok', activeRequests }` as JSON with status code `200`
2. A `/slow` route that waits 4 seconds (`setTimeout`) before responding, to simulate a long-running request
3. A shutdown handler for `SIGTERM`/`SIGINT` that sets an `isShuttingDown` flag, immediately makes `/health` return status `503` with `{ status: 'shutting down' }`, and only exits once `server.close()`'s callback fires

Test it: start a request to `/slow`, then immediately send `SIGINT` — the process should wait for that request to finish before exiting.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a reusable `GracefulShutdownManager` class that any server can use. It should:
1. Take an HTTP `server` instance and a `timeoutMs` value (default `10000`) in its constructor
2. Have a method `addCleanupTask(name, fn)` that registers a named async cleanup function (e.g. `'closeDatabase'`, `'flushLogs'`, `'closeRedisConnection'`) to be run during shutdown
3. Have a method `listen()` that wires up both `SIGTERM` and `SIGINT` to trigger shutdown
4. When shutdown starts: stop the HTTP server first (`server.close()`), then run every registered cleanup task **in the order they were added**, logging each task's name as it starts and finishes
5. If any cleanup task throws, log the error but continue running the remaining tasks (one failing cleanup shouldn't block the others)
6. Enforce the `timeoutMs` — if the whole shutdown sequence takes longer than that, force-exit with code `1` and log which stage it got stuck on
7. Exit with code `0` if everything completes cleanly, or `1` if any cleanup task failed or the timeout was hit

Test it by registering a fake `dbConnection.close()`, a fake `redisClient.quit()`, and a cleanup task that deliberately throws — verify all three still run and the process exits with the correct code.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
SIGNALS TO HANDLE
  SIGTERM   → sent by Docker, Kubernetes, PM2 — the "please stop" signal, has a grace period
  SIGINT    → sent by Ctrl+C in a terminal — local development
  SIGKILL   → CANNOT be caught or handled — the OS force-kills immediately (last resort)

CORE SHUTDOWN STEPS (in order)
  1. Stop accepting NEW work        → server.close(), stop consuming new queue jobs
  2. Flip health check to unhealthy → so load balancer/K8s stops routing traffic here
  3. Let IN-FLIGHT work finish      → existing requests/queries complete naturally
  4. Clean up resources             → close DB pools, Redis clients, file handles, flush logs
  5. Exit with correct code         → process.exit(0) success, process.exit(1) failure

KEY API
  server.close(callback)   → stops new connections, callback fires when all existing ones end
  process.on('SIGTERM', fn)  process.on('SIGINT', fn) → register the handlers
  process.exit(code)       → 0 = success, non-zero = failure (Topic 05)
  setTimeout(fn, ms).unref() → force-exit safety net that won't itself block early exit

GOTCHAS
  server.close() does NOT close existing keep-alive sockets by itself in older
    Node versions — track and close idle sockets manually if needed
  Forgetting a timeout safety net → one stuck connection hangs shutdown forever
  Running cleanup tasks BEFORE server.close() → risk closing a DB connection
    that an in-flight request still needs
  Only handling SIGINT → production signals (SIGTERM) never trigger cleanup
  Health check must return unhealthy status BEFORE server.close() finishes,
    not after — orchestrators poll it to decide when to stop sending traffic

KUBERNETES TIMELINE (why timing matters)
  1. Pod marked "Terminating"
  2. SIGTERM sent to your process
  3. Readiness probe should already be failing → traffic stops routing in
  4. Grace period (default 30s) for your app to exit cleanly
  5. If still running after grace period → SIGKILL (uncatchable, instant death)
```

---

## Connected topics

- **65 — Health checks and readiness probes** — the `/health` and `/ready` endpoints that graceful shutdown must flip to "unhealthy" so orchestrators stop routing traffic before the process exits
- **63 — Logging in Node** — every shutdown step should be logged so you can debug slow or failed deploys after the fact
- **20 — http module** — `server.close()` and the request/response lifecycle this topic depends on to know when connections have actually drained
