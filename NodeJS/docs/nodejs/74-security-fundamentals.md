# 74 — Security fundamentals in Node

## What is this?

Security fundamentals are the baseline mindset and habits that keep a Node.js backend from being an easy target: knowing your **attack surface** (every point where untrusted input or code can enter your system), applying the **principle of least privilege** (every process, user, and piece of code gets only the access it strictly needs), and practicing **dependency security** (treating every npm package you install as code you now have to trust and maintain). Think of your server like a house — the attack surface is every door and window, least privilege is not giving the delivery guy a key to the safe, and dependency security is checking that the locksmith you hired isn't secretly selling copies of your keys.

## Why does it matter for backend development?

A backend is the part of the system attackers can actually reach — it accepts input from the internet, talks to a database, holds secrets, and runs with real permissions on a real machine. A single unvalidated field, an over-privileged database user, or one compromised npm package can turn into a full data breach. Backend developers apply security fundamentals from day one of a project — not bolted on later — because retrofitting security into a live production system with real user data is far more expensive and risky than designing it in from the start. This topic is the foundation the rest of Phase 10 (input validation, injection prevention, XSS/CSRF, secrets, TLS, dependency auditing) builds on.

---

## Syntax / API

There is no single "security API" — this topic is about applying a checklist of Node-specific practices. Here is the shape of it in code:

```js
// ── 1. Reduce attack surface: only expose what you must ────────────────────
const express = require('express');
const app = express();

// Turn off the header that advertises the framework — attackers use this
// to look up known Express CVEs for the version you're running.
app.disable('x-powered-by');

// Only parse JSON bodies up to a sane size — an unbounded body is an
// attack surface for memory-exhaustion (DoS) attacks.
app.use(express.json({ limit: '100kb' }));

// ── 2. Principle of least privilege: scoped credentials ─────────────────────
// The API server's DB user should NOT be able to DROP TABLE or CREATE USER.
// That permission belongs only to the migration user, run manually by an admin.
const dbConnection = require('./db'); // connects with a read/write-only DB role

// ── 3. Dependency security: know what you ship ─────────────────────────────
// package.json — pin exact or caret ranges you understand, avoid "*"
// {
//   "dependencies": {
//     "express": "^4.19.2"   // never "*", never "latest"
//   }
// }

// Run this before every deploy / in CI — fails the build on known vulnerabilities
// $ npm audit --audit-level=high

module.exports = app;
```

---

## How it works — line by line

- `app.disable('x-powered-by')` removes the `X-Powered-By: Express` response header. Every unnecessary detail your server reveals (framework, version, error stack traces) is one more clue that shrinks an attacker's search space — this is reducing attack surface.
- `express.json({ limit: '100kb' })` caps how large an incoming request body can be. Without a limit, anyone can send a 5 GB JSON body and crash your process by exhausting memory — that's an unguarded door in your attack surface.
- `dbConnection` connecting with a restricted database role means that even if an attacker finds a SQL injection bug, the damage is capped — they can read/write rows but cannot drop tables or create new database users. That is least privilege in action: the process only has the permissions its job requires, nothing more.
- Pinning dependency versions in `package.json` (instead of `*` or `latest`) means you control exactly what code runs in production — you decide when to upgrade, after reviewing the changelog, instead of silently pulling in a compromised or breaking update.
- `npm audit --audit-level=high` scans your installed packages against a public vulnerability database and fails (non-zero exit code) if a high-or-worse severity issue is found — this is how dependency security becomes an automated gate instead of a manual chore nobody remembers to do.

---

## Example 1 — basic

```js
// File: src/security/basics.js
// Three small, concrete security habits applied to a tiny Express app.

const express = require('express');
const app = express();

// 1. Hide the framework fingerprint from response headers.
app.disable('x-powered-by');

// 2. Cap request body size — prevents memory-exhaustion DoS from huge payloads.
app.use(express.json({ limit: '50kb' }));

// 3. Never leak internal error details to the client — this is attack-surface
//    reduction: stack traces reveal file paths, package versions, and logic.
app.use((err, req, res, next) => {
  console.error(err); // full detail goes to your OWN logs, not the response
  res.status(500).json({ error: 'Internal server error' }); // generic, safe message
});

// A route that only returns what the client needs — nothing more.
app.get('/api/users/:userId', (req, res) => {
  const userId = req.params.userId; // untrusted input — always treat as attack surface

  // NOTE: real validation of userId happens in Topic 75 (input validation)
  res.json({ userId, status: 'active' }); // no password hash, no internal flags
});

app.listen(3000, () => {
  console.log('Server listening on port 3000');
});
```

---

## Example 2 — real world backend use case

```js
// File: src/server.js
// A more realistic Express bootstrap showing least privilege and reduced
// attack surface applied together, the way you'd actually set up a new
// backend service on day one.

const express = require('express');
const app = express();

// ── Reduce attack surface ───────────────────────────────────────────────────
app.disable('x-powered-by');                 // don't advertise the framework
app.use(express.json({ limit: '100kb' }));    // bound request body size

// Only allow the HTTP methods this API actually supports — reject the rest
// early instead of letting them fall through to unexpected route handlers.
const allowedMethods = new Set(['GET', 'POST', 'PATCH', 'DELETE']);
app.use((req, res, next) => {
  if (!allowedMethods.has(req.method)) {
    return res.status(405).json({ error: 'Method not allowed' }); // reject early
  }
  next(); // continue only if the method is expected
});

// ── Principle of least privilege — per-role DB connections ─────────────────
// The reporting endpoint only ever needs to READ data, so it uses a
// read-only DB role. Even a bug in this route can't corrupt data.
const readOnlyDb = require('./db/readOnlyConnection'); // role: SELECT only

// The account-update endpoint needs write access, but ONLY to the `users`
// table — the same role cannot touch `payments` or `audit_logs`.
const usersWriteDb = require('./db/usersWriteConnection'); // role: users table only

app.get('/api/reports/summary', async (req, res, next) => {
  try {
    // Even if this query had a bug, the read-only role prevents any write.
    const result = await readOnlyDb.query('SELECT COUNT(*) FROM orders');
    res.json({ totalOrders: result.rows[0].count });
  } catch (err) {
    next(err); // handled by the centralized error handler below
  }
});

app.patch('/api/users/:userId', async (req, res, next) => {
  const userId = req.params.userId;        // untrusted — validate in real code
  const { displayName } = req.body;        // untrusted — validate in real code

  try {
    // usersWriteDb literally cannot run "DROP TABLE payments" — the DB
    // grants for this role were scoped to the users table at creation time.
    await usersWriteDb.query(
      'UPDATE users SET display_name = $1 WHERE id = $2',
      [displayName, userId]
    );
    res.json({ userId, displayName });
  } catch (err) {
    next(err);
  }
});

// ── Centralized error handler: never leak internals ─────────────────────────
app.use((err, req, res, next) => {
  console.error('[unhandled]', err); // full stack trace stays server-side only
  res.status(500).json({ error: 'Something went wrong' }); // client sees nothing sensitive
});

app.listen(process.env.PORT || 3000, () => {
  console.log('API server running with least-privilege DB roles');
});
```

---

## Common mistakes

### Mistake 1 — Running the app (and its DB user) with full/root privileges

```js
// ❌ WRONG — API connects with the DB superuser, and the container runs as root.
// A single SQL injection bug now means total database compromise.
const dbConnection = require('pg').Client({
  user: 'postgres',        // superuser — can drop databases, create roles, read all tables
  password: process.env.DB_ROOT_PASSWORD,
});
// Dockerfile also has no USER instruction, so the container runs as root by default.

// ✅ CORRECT — API connects with a scoped role; container runs as an unprivileged user.
const dbConnection = require('pg').Client({
  user: 'api_service',     // role granted SELECT/INSERT/UPDATE on specific tables only
  password: process.env.DB_API_PASSWORD,
});
// Dockerfile:
//   RUN adduser --system --uid 1001 appuser
//   USER appuser   // process cannot write outside its own permissions
```

### Mistake 2 — Trusting `npm install` blindly, never auditing dependencies

```js
// ❌ WRONG — dependencies added over months, never checked, never updated.
// package.json has "lodash": "*" and 40 transitive packages nobody reviewed.
// A known-vulnerable version silently gets pulled in on every fresh install.

// ✅ CORRECT — pin sane version ranges and audit automatically before every deploy.
// package.json:
// "dependencies": { "lodash": "^4.17.21" }   // specific, known-good minimum version
//
// package-lock.json committed to git — guarantees the EXACT same tree every install.
//
// CI pipeline step:
// $ npm ci                          // installs exactly what's in the lockfile
// $ npm audit --audit-level=high    // fails the build if a high/critical CVE is found
```

### Mistake 3 — Exposing internal errors and stack traces to the client

```js
// ❌ WRONG — the raw error (with file paths, SQL, and stack trace) goes straight
// to the response. This hands an attacker a map of your internals for free.
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message, stack: err.stack }); // leaks everything
});

// ✅ CORRECT — log full detail privately, send a generic message publicly.
app.use((err, req, res, next) => {
  console.error('[server error]', err); // full detail goes to your logging system only
  res.status(500).json({ error: 'Internal server error' }); // attacker learns nothing
});
```

---

## Practice exercises

### Exercise 1 — easy

Create a tiny Express app with a single `GET /api/status` route. Apply three baseline hardening steps to it:
1. Disable the `X-Powered-By` header.
2. Limit incoming JSON body size to `10kb`.
3. Add a centralized error handler that logs the full error to the console but only ever sends `{ error: 'Internal server error' }` to the client (status 500).

Verify by temporarily throwing an error inside the route and confirming the client response contains no stack trace.

```js
// Write your code here
```

---

### Exercise 2 — medium

Design (in code comments/pseudocode is fine, plus real Express routing code) a small API with two roles of database access:
1. A `readOnlyDb` connection object used only by a `GET /api/products` route.
2. A `writeDb` connection object used only by a `POST /api/products` route.

Write a middleware function `enforceMethodWhitelist(allowedMethods)` that takes an array like `['GET', 'POST']` and rejects any other HTTP method on that router with a `405` response before the request reaches your route handlers. Apply it to a router that only supports `GET` and `POST`. Explain (as a code comment) why using two separate connection objects, even to the same physical database, is an application of least privilege.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small "security audit" CLI script (`securityCheck.js`, run with plain Node, no framework needed) that inspects a target Node.js project folder and reports on:
1. Whether `package.json` contains any dependency version pinned as `"*"` or `"latest"` (attack surface: unpredictable dependency updates).
2. Whether a `package-lock.json` file exists (missing lockfile = unreproducible installs, a supply-chain risk).
3. Whether any `.js` file in the project contains the literal string `x-powered-by: false` disabled somewhere OR whether `app.disable('x-powered-by')` appears anywhere in the source (best-effort text search is fine).
4. Whether any file matches common secret-leak patterns, e.g. a line containing `AKIA` (AWS key prefix) or the literal string `password =` followed by a quoted string (a naive but useful heuristic — full secret scanning is out of scope here).

Print a clear pass/fail report to the console for each of the four checks, with the specific file and line number when something fails.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE IDEAS
  Attack surface           → every point where untrusted input/code enters your system
                             (routes, request bodies, headers, files, dependencies, env vars)
  Principle of least       → every process, DB user, API key, and container gets ONLY the
    privilege                permissions it strictly needs — nothing "just in case"
  Dependency security      → every npm package is code you now trust; audit and pin it

REDUCE ATTACK SURFACE — DAY-ONE CHECKLIST
  app.disable('x-powered-by')        → hide framework fingerprint
  express.json({ limit: '100kb' })   → bound request body size
  Reject unexpected HTTP methods early
  Never send stack traces / raw errors to clients
  Validate & sanitize ALL external input (see Topic 75)
  Keep NODE_ENV=production in prod (disables verbose Express error pages)

LEAST PRIVILEGE — CHECKLIST
  DB user for the API      → SELECT/INSERT/UPDATE only, never DROP/CREATE ROLE
  Separate read-only vs write DB roles where possible
  Container/process runs as a non-root user (Docker: USER instruction)
  API keys scoped to only the resources/actions they need
  File system access limited to specific directories (never root/home)

DEPENDENCY SECURITY — CHECKLIST
  Never use "*" or "latest" in package.json — pin sane ranges (e.g. ^4.19.2)
  Always commit package-lock.json — guarantees reproducible installs
  npm ci                              → installs exactly what's locked (CI/production)
  npm audit --audit-level=high        → fail builds on known high/critical CVEs
  Remove unused dependencies regularly (fewer packages = smaller attack surface)
  Review new dependencies before adding — stars/maintenance/last-publish date

GOTCHAS
  "It's just an internal tool" is not a security exemption — internal tools get breached too
  A vulnerability scanner passing today doesn't mean tomorrow's new CVE won't affect you
  Security is layered — no single control (helmet, validation, TLS) is sufficient alone
```

---

## Connected topics

- **75 — Input validation and sanitization** — the next concrete defense against the largest single category of attack surface: untrusted user input.
- **79 — Secrets management** — least privilege applied specifically to API keys, database passwords, and tokens — never in code, never in git.
- **81 — Dependency auditing** — the deep dive on `npm audit`, Snyk, and CVE tracking that this topic only introduced at a high level.
