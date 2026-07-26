# 112 — 12-Factor App methodology

## What is this?

The 12-Factor App is a set of 12 rules for building backend applications that are easy to deploy, scale, and hand off to a new developer without surprises. It was written by engineers at Heroku after watching hundreds of apps fail in production for the same predictable reasons — hardcoded config, logs written to disk, servers that remember who you are. Think of it like building codes for houses: nobody forces you to follow them, but every house that ignores them eventually has a problem an inspector would have caught. This document covers the four factors that matter most day-to-day in Node.js: **config**, **logs**, **stateless processes**, and **backing services**.

---

## Why does it matter for backend development?

Every backend developer eventually deploys the same app to three or four different places — a laptop, a staging server, a production cluster with 5 replicas behind a load balancer. If your app reads secrets from a hardcoded file, writes logs to a local `app.log`, or keeps user sessions in a JavaScript variable in memory, it will work perfectly on your laptop and then fail in ways that are hard to diagnose the moment it touches a real environment. The 12-Factor rules exist because these four mistakes — bad config, bad logging, statefulness, and tight coupling to backing services — account for the majority of "works on my machine" incidents. A Node.js developer who internalizes these four factors writes apps that survive being restarted, scaled to multiple instances, and moved between cloud providers without code changes.

---

## Syntax / API

```js
// ── FACTOR III: CONFIG ──────────────────────────────────────────────────────
// Rule: store config in environment variables, never in code.
// Anything that changes between dev/staging/prod is config — DB URLs, API keys, ports.

require('dotenv').config();               // loads .env into process.env (dev only)

const dbConnection = process.env.DATABASE_URL;   // no hardcoded connection string
const apiKey       = process.env.PAYMENT_API_KEY; // no hardcoded secret
const port          = process.env.PORT || 3000;    // sensible default, still overridable

// ── FACTOR XI: LOGS ─────────────────────────────────────────────────────────
// Rule: treat logs as an event stream — write to stdout/stderr, never to a file yourself.
// Let the environment (Docker, PM2, systemd, a log collector) decide where logs end up.

console.log(JSON.stringify({ level: 'info', msg: 'server started', port }));
// → written to stdout — the platform captures it, rotates it, ships it elsewhere

// ── FACTOR VI: STATELESS PROCESSES ──────────────────────────────────────────
// Rule: a process must not remember anything between requests that isn't in a
// backing store. Any two requests could hit two different server instances.

// ❌ in-memory state — lost/wrong the moment you run 2+ instances
const activeSessions = {};                  // dies with the process, invisible to instance #2

// ✅ externalized state — every instance reads/writes the same source of truth
const redisClient = require('./redisClient');
async function getSession(sessionId) {
  return redisClient.get(`session:${sessionId}`);   // shared across all instances
}

// ── FACTOR IV: BACKING SERVICES ─────────────────────────────────────────────
// Rule: treat databases, caches, queues as attached resources reached over the
// network via a URL/credential in config — swappable without code changes.

const { Pool } = require('pg');
const dbPool = new Pool({ connectionString: process.env.DATABASE_URL });
// Swapping local Postgres for a managed RDS instance = change one env var, zero code changes
```

---

## How it works — line by line

- `require('dotenv').config()` reads a local `.env` file and copies its key-value pairs into `process.env`, purely as a developer convenience — in staging/production, the platform (Docker, PM2, your cloud host) injects these variables directly, so `dotenv` often does nothing there and that is expected.
- `process.env.DATABASE_URL` pulls the database connection string out of the environment instead of a config file checked into git — the exact same line of code connects to a different database in dev, staging, and production, because only the environment variable changes.
- `process.env.PORT || 3000` reads the port the platform assigns, falling back to `3000` only when nothing is set (like on your own laptop) — this is why you never see `app.listen(3000)` hardcoded in a real backend.
- `console.log(JSON.stringify({...}))` writes a structured log line to standard output — it does not open a file, does not rotate anything, does not care where the bytes end up. The **environment** decides that: Docker captures stdout automatically, PM2 redirects it to a file, a log shipper like Fluentd tails it into Elasticsearch.
- `const activeSessions = {}` is called out as wrong because it lives in the memory of one running Node.js process. The moment you run two copies of this server behind a load balancer, a user's session exists in only one of them — request 1 might hit instance A (session saved), request 2 might hit instance B (session missing), and the user gets logged out randomly.
- `redisClient.get(...)` replaces that in-memory object with a call to an external, shared store — every instance of the app, no matter how many you run, asks the same Redis server for the same session data, so scaling to 10 instances doesn't break sessions.
- `new Pool({ connectionString: process.env.DATABASE_URL })` treats the entire database as a "backing service" — a resource attached by URL, not baked into the code. If ops needs to move the database to a new server, they change one environment variable and restart the app; nothing in the source code has to know or care.

---

## Example 1 — basic

```js
// File: src/config/index.js
// A minimal config module that centralizes every 12-factor "config" value in one place.

require('dotenv').config();          // load .env in dev; no-op in prod where env is injected

// Collect every environment-dependent value here — nowhere else in the codebase
// should you see process.env.SOMETHING scattered around.
const config = {
  nodeEnv:      process.env.NODE_ENV || 'development',    // dev / staging / production
  port:         Number(process.env.PORT) || 3000,          // platform assigns this in prod
  databaseUrl:  process.env.DATABASE_URL,                  // e.g. postgres://user:pass@host/db
  redisUrl:     process.env.REDIS_URL,                     // e.g. redis://host:6379
  jwtSecret:    process.env.JWT_SECRET,                    // signing key for auth tokens
  logLevel:     process.env.LOG_LEVEL || 'info',           // debug / info / warn / error
};

// Fail fast at startup if a required secret is missing — better than a confusing
// crash three requests later when the app finally tries to use it.
const required = ['databaseUrl', 'jwtSecret'];
for (const key of required) {
  if (!config[key]) {
    // process.stderr write + exit — covered in Topic 05 (the process object)
    process.stderr.write(`[config] Missing required env var for "${key}"\n`);
    process.exit(1);
  }
}

module.exports = config;

// Usage elsewhere in the app:
// const config = require('./config');
// app.listen(config.port);
```

---

## Example 2 — real world backend use case

```js
// File: src/server.js
// A small Express-style server built around all four factors at once:
// config from env, logs to stdout, no in-process state, backing services via config.

const config      = require('./config');            // Factor III — centralized config
const dbPool      = require('./db/pool');            // Factor IV — Postgres as backing service
const redisClient = require('./cache/redisClient');  // Factor IV — Redis as backing service
const express     = require('express');

const app = express();
app.use(express.json());

// Structured request logging to stdout — Factor XI. No fs.appendFile, no log file path.
app.use((req, res, next) => {
  const startTime = Date.now();
  res.on('finish', () => {
    console.log(JSON.stringify({
      level:    'info',
      method:   req.method,
      path:     req.originalUrl,
      status:   res.statusCode,
      durationMs: Date.now() - startTime,
    }));
  });
  next();
});

// Login handler — Factor VI: nothing about the logged-in user lives in this process.
// The session is written to Redis, not to a local variable, so it survives restarts
// and is visible to every other instance behind the load balancer.
app.post('/login', async (req, res) => {
  const { userId, password } = req.body;

  const userRow = await dbPool.query(               // Factor IV — DB reached via config URL
    'SELECT id, password_hash FROM users WHERE id = $1',
    [userId]
  );

  if (!userRow.rows.length) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  const sessionId = require('crypto').randomUUID();  // random session identifier
  await redisClient.set(                              // externalized state, not in-memory
    `session:${sessionId}`,
    JSON.stringify({ userId, createdAt: Date.now() }),
    'EX', 3600                                        // expires in 1 hour
  );

  res.json({ sessionId });                            // client stores this, sends it back later
});

// Port comes from config (Factor III), which in turn comes from the environment.
app.listen(config.port, () => {
  console.log(JSON.stringify({ level: 'info', msg: `listening on ${config.port}` }));
});
```

---

## Common mistakes

### Mistake 1 — Hardcoding config values instead of reading them from the environment

```js
// ❌ WRONG — connection string and secret committed straight into source code
const dbConnection = 'postgres://admin:P@ssw0rd@prod-db.internal:5432/app';
const jwtSecret     = 'my-super-secret-key-123';
// Now this exact string lives in git history forever, and switching environments
// means editing and redeploying source code just to point at a different database.

// ✅ CORRECT — config comes from the environment, code never changes between environments
const dbConnection = process.env.DATABASE_URL;
const jwtSecret     = process.env.JWT_SECRET;
// Same code, same deploy artifact — dev, staging, and prod just supply different env vars
```

### Mistake 2 — Writing logs directly to a file inside the app

```js
// ❌ WRONG — the app manages its own log file, path, and rotation
const fs = require('fs');
fs.appendFileSync('/var/log/myapp/server.log', `${new Date().toISOString()} request handled\n`);
// Breaks in containers (the file disappears when the container dies), fills disks
// silently, and every instance writes to a different file with no unified view.

// ✅ CORRECT — write to stdout, let the platform capture/collect/rotate/ship logs
console.log(JSON.stringify({
  level: 'info',
  msg: 'request handled',
  timestamp: new Date().toISOString(),
}));
// Docker, PM2, Kubernetes, and log shippers (Fluentd, Datadog agent) all know how
// to capture stdout — you never have to think about file paths or rotation again
```

### Mistake 3 — Keeping request-scoped or user state in process memory

```js
// ❌ WRONG — an in-memory object used as a "session store" or rate-limit counter
const requestCounts = {};   // { [userId]: number }

function checkRateLimit(userId) {
  requestCounts[userId] = (requestCounts[userId] || 0) + 1;
  return requestCounts[userId] <= 100;
}
// Works fine with 1 server instance. The moment you run 2+ instances (or restart
// the process, or use PM2 cluster mode), each process has its own separate counter —
// a user can send 100 requests to instance A and another 100 to instance B undetected.

// ✅ CORRECT — externalize the counter to a shared backing service (Redis)
const redisClient = require('./cache/redisClient');

async function checkRateLimit(userId) {
  const count = await redisClient.incr(`ratelimit:${userId}`);  // shared across all instances
  if (count === 1) await redisClient.expire(`ratelimit:${userId}`, 60);  // reset window
  return count <= 100;
}
// Now every instance of the app agrees on the same count, no matter how many run
```

---

## Practice exercises

### Exercise 1 — easy

Create a `src/config.js` file that:
1. Loads environment variables using `dotenv` (assume a `.env` file with `PORT`, `NODE_ENV`, and `API_KEY` already exists)
2. Exports an object with `port` (number, default `3000`), `nodeEnv` (default `'development'`), and `apiKey`
3. If `apiKey` is missing, prints an error to `process.stderr` and exits the process with code `1`

Log the final config object to verify it loads correctly.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a small Express-style logging middleware function `requestLogger(req, res, next)` that:
1. Records the start time when the request comes in
2. Listens for the response `'finish'` event
3. When it fires, logs a single JSON string to stdout via `console.log(JSON.stringify(...))` containing: `method`, `path`, `statusCode`, and `durationMs`
4. Never writes to a file — stdout only (Factor XI)

Then write a second function `getOrCreateVisitorId(req, redisClient)` that:
1. Reads a `visitorId` from the request's cookies (assume `req.cookies.visitorId` exists)
2. If missing, generates one with `crypto.randomUUID()`
3. Stores it in `redisClient` (not in a local JS object — Factor VI) with a 1-day expiry
4. Returns the visitor ID

You may stub `redisClient` as a simple object with `set(key, value, ...args)` for this exercise.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small `ConfigLoader` module that fully embodies Factor III and fails fast on misconfiguration:

1. A function `loadConfig(schema)` where `schema` is an object like:
   ```js
   {
     PORT:          { type: 'number', default: 3000 },
     DATABASE_URL:  { type: 'string', required: true },
     REDIS_URL:     { type: 'string', required: true },
     LOG_LEVEL:     { type: 'string', default: 'info' },
     ENABLE_CACHE:  { type: 'boolean', default: false },
   }
   ```
2. For each key in the schema, read `process.env[KEY]`
3. If `required: true` and the variable is missing, collect an error message (don't exit immediately — gather ALL missing vars first)
4. If a value is present, coerce it to the declared `type` (`'number'` → `Number(...)`, `'boolean'` → `value === 'true'`, `'string'` → as-is)
5. If any required variables were missing, print all the collected error messages to `process.stderr` (one per line) and call `process.exit(1)`
6. Otherwise return the fully resolved, correctly-typed config object

Test it by simulating two scenarios: one where `DATABASE_URL` and `REDIS_URL` are both set (should return a config object), and one where both are missing (should print two error lines and exit).

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
THE FOUR FACTORS COVERED HERE

  III — CONFIG
    Store config in the environment (process.env), never in code.
    Config = anything that varies between dev/staging/prod: DB URLs, API keys, ports, secrets.
    Use dotenv locally; the platform injects env vars in staging/prod.

  XI — LOGS
    Treat logs as an event stream. Write to stdout/stderr only.
    Never open/manage log files yourself — let Docker/PM2/Kubernetes/log-shippers do it.
    Prefer structured logs: console.log(JSON.stringify({ level, msg, ...fields }))

  VI — STATELESS PROCESSES
    A process must not remember anything between requests beyond a single request's lifetime.
    Sessions, rate-limit counters, uploaded file temp state → externalize to Redis/DB/disk-service.
    Any request can be served by any instance — that's what makes horizontal scaling possible.

  IV — BACKING SERVICES
    Treat DBs, caches, queues, third-party APIs as attached resources, reached via a URL in config.
    Swapping a local Postgres for managed RDS = change one env var, zero code changes.

WHY IT MATTERS
  Config in code       → can't redeploy to a new env without a code change/rebuild
  Logs to local files   → lost when the container dies, unreadable across instances
  In-memory state       → breaks the instant you run 2+ instances or restart the process
  Hardcoded service URLs→ can't swap providers or scale services independently

QUICK RULES OF THUMB
  If it changes between dev/staging/prod           → it's config, put it in process.env
  If you need to see it after the process restarts → it doesn't belong in memory
  If two server instances need to agree on it       → it belongs in Redis/DB, not a JS variable
  If you're calling fs.appendFile for logs           → stop, use console.log/console.error instead

TWELVE FACTORS (for context — this doc covers III, IV, VI, XI)
  I. Codebase  II. Dependencies  III. Config  IV. Backing services  V. Build/release/run
  VI. Processes  VII. Port binding  VIII. Concurrency  IX. Disposability  X. Dev/prod parity
  XI. Logs  XII. Admin processes
```

---

## Connected topics

- **12 — Environment variables** — `.env` files, `process.env`, and `dotenv` are the direct implementation of Factor III (Config) covered in this doc.
- **63 — Logging in Node** — goes deep into `winston`/`pino`, log levels, and structured JSON logging that put Factor XI (Logs) into production practice.
- **107 — Environment management** — dev/test/staging/prod config strategies and libraries like `convict`/`dotenv-flow` that build directly on top of the config principles here.
