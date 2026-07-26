# 107 — Environment management

## What is this?

Environment management is the practice of running the **same codebase** with **different configuration** depending on where it's deployed — your laptop (development), the automated test runner (test), a pre-production server (staging), or the live server real users hit (production). Instead of hardcoding a database URL or API key, the code asks "which environment am I in right now?" and loads the matching settings. Think of it like a TV with different channel presets — the remote (your code) stays identical, but pressing "prod" tunes in completely different values than pressing "dev."

## Why does it matter for backend development?

A backend server needs different values depending on where it runs: a local Postgres URL in dev, a throwaway test database in CI, a staging API key for a payment gateway's sandbox, and the real production database and secrets in prod. If you hardcode any of these, you either leak secrets into git or crash the app the moment it runs somewhere else. Every real backend — from a solo side project to a company's payment system — needs a clean, predictable way to say "load dev settings here, prod settings there" without ever touching the source code. Getting this wrong is one of the most common causes of "it worked on my machine but broke in production."

---

## Syntax / API

```js
// ── The core signal: NODE_ENV ───────────────────────────────────────────────
// NODE_ENV is a convention (not enforced by Node itself) that tells your app
// and many libraries (Express, React, logging tools) which environment is active
console.log(process.env.NODE_ENV);
// → "development" | "test" | "staging" | "production" | undefined

// ── dotenv: load a single .env file into process.env ────────────────────────
require('dotenv').config();
// Reads a file named ".env" in the project root and copies each KEY=VALUE
// line into process.env — only if that key isn't already set

// ── dotenv-flow: load DIFFERENT .env files based on NODE_ENV ────────────────
require('dotenv-flow').config();
// If NODE_ENV=development → loads .env, .env.development, .env.local (in order)
// If NODE_ENV=production  → loads .env, .env.production
// Later files override earlier ones — lets you share common values + override per env

// ── convict: schema-validated, typed configuration ──────────────────────────
const convict = require('convict');

const config = convict({
  port: {
    doc: 'The port the server listens on',   // description shown in errors/docs
    format: 'port',                           // convict validates this is 1-65535
    default: 3000,                             // fallback if nothing else is set
    env: 'PORT',                               // reads process.env.PORT
  },
  dbUrl: {
    doc: 'Database connection string',
    format: String,                            // must be a string
    default: null,                              // no safe default — must be provided
    env: 'DATABASE_URL',
  },
});

config.validate({ allowed: 'strict' });        // throws if required values are missing/wrong type
module.exports = config;                        // export the validated, typed config object
```

---

## How it works — line by line

`process.env.NODE_ENV` is just a string sitting in the process's environment variables — Node itself does nothing special with it, but the ecosystem treats it as a convention. Express logs verbose errors when it's `"development"` and hides stack traces when it's `"production"`. You set it before starting the app: `NODE_ENV=production node server.js`.

`dotenv` solves the problem of typing long `export KEY=value` commands every time. It reads a `.env` text file line by line, splits each line on the first `=`, and assigns the result to `process.env`. It never overwrites a variable that's already set in the real shell environment — so a value set by your hosting platform (Heroku, Docker, Kubernetes) always wins over the file.

`dotenv-flow` extends this idea across environments. Instead of one `.env` file, you keep several: `.env` (shared defaults for every environment), `.env.development`, `.env.test`, `.env.staging`, `.env.production`, and an optional `.env.local` (personal overrides, never committed to git). It picks which files to load based on the current `NODE_ENV`, loading the shared file first and then layering the environment-specific file on top, so specific values override shared ones.

`convict` goes one step further than either — it doesn't just load values, it **defines a schema**. Each config key declares its expected type (`format`), a `default`, and which environment variable feeds it (`env`). Calling `.validate()` at startup crashes the app immediately with a clear error if something required is missing or the wrong type — far better than discovering a missing `DATABASE_URL` three requests into production traffic.

---

## Example 1 — basic

```js
// File: src/config.js
// Minimal environment-aware config using plain process.env + dotenv

require('dotenv').config();   // load .env file into process.env (dev convenience)

// NODE_ENV tells us which "mode" we're running in
const nodeEnv = process.env.NODE_ENV || 'development';   // default to dev if unset

// Simple flags derived from NODE_ENV — used all over the codebase
const isProduction  = nodeEnv === 'production';
const isDevelopment = nodeEnv === 'development';
const isTest        = nodeEnv === 'test';

// Read individual values, with safe fallbacks for local development only
const port      = process.env.PORT || 3000;               // server listening port
const dbUrl     = process.env.DATABASE_URL || 'postgres://localhost:5432/dev_db';
const logLevel  = isProduction ? 'warn' : 'debug';         // quieter logs in prod

console.log(`Running in "${nodeEnv}" mode on port ${port}`);

// Export everything the rest of the app needs
module.exports = { nodeEnv, isProduction, isDevelopment, isTest, port, dbUrl, logLevel };
```

---

## Example 2 — real world backend use case

```js
// File: src/config/index.js
// Production-grade config: dotenv-flow for per-env files + convict for validation.
// This is the pattern used in real Express/Fastify backends deployed across
// dev, test, staging, and production.

// dotenv-flow picks the right .env files based on NODE_ENV, then loads them
require('dotenv-flow').config({
  node_env: process.env.NODE_ENV || 'development',   // which set of files to load
  silent: true,                                       // don't warn if a file is missing
});

const convict = require('convict');

// Define the full shape of the app's configuration up front
const schema = convict({
  env: {
    doc: 'The application environment',
    format: ['development', 'test', 'staging', 'production'],  // only these values allowed
    default: 'development',
    env: 'NODE_ENV',
  },
  server: {
    port: {
      doc: 'HTTP port to listen on',
      format: 'port',
      default: 3000,
      env: 'PORT',
    },
  },
  db: {
    connectionString: {
      doc: 'Postgres connection string',
      format: String,
      default: null,                 // no default — every env MUST set this explicitly
      env: 'DATABASE_URL',
      sensitive: true,                // convict masks this in config.toString() output
    },
    poolSize: {
      doc: 'Max connections in the pool',
      format: 'nat',                  // natural number (0, 1, 2, ...)
      default: 10,
      env: 'DB_POOL_SIZE',
    },
  },
  auth: {
    jwtSecret: {
      doc: 'Secret used to sign JWTs',
      format: String,
      default: null,                  // required in every environment — no fallback
      env: 'JWT_SECRET',
      sensitive: true,
    },
  },
});

// Validate immediately at startup — fail fast instead of failing mid-request
schema.validate({ allowed: 'strict' });   // throws AppConfigError if anything is wrong

// Freeze so no code accidentally mutates config after startup
const config = schema.getProperties();
Object.freeze(config);

module.exports = config;

// Usage elsewhere in the app:
// const config = require('./config');
// const dbConnection = createPool(config.db.connectionString, config.db.poolSize);
// app.listen(config.server.port);
```

```
Project files that make this work:

.env               → PORT=3000, LOG_LEVEL=debug              (shared, safe to commit)
.env.development    → DATABASE_URL=postgres://localhost/dev_db
.env.test           → DATABASE_URL=postgres://localhost/test_db
.env.staging        → DATABASE_URL=<staging DB, injected by CI, never committed>
.env.production     → NOT a file at all — real prod secrets come from the
                        hosting platform's environment (Docker secrets, AWS
                        Secrets Manager, Kubernetes secrets), never a checked-in file
.env.local          → personal overrides for one developer's machine (gitignored)
```

---

## Common mistakes

### Mistake 1 — Committing real secrets in a .env file

```js
// ❌ WRONG — .env.production checked into git with the real database password
// .env.production
// DATABASE_URL=postgres://admin:SuperSecret123@prod-db.company.com:5432/app
// (now visible to anyone with repo access, forever, even after deletion — it's in git history)

// ✅ CORRECT — only commit a template with placeholder values, add real files to .gitignore
// .env.example (committed — shows what variables are needed, no real values)
// DATABASE_URL=postgres://user:password@localhost:5432/dbname
// JWT_SECRET=replace-with-a-long-random-string

// .gitignore
// .env
// .env.local
// .env.*.local
// .env.production   (real production values are injected by the hosting platform instead)
```

### Mistake 2 — Trusting process.env values without validating type or presence

```js
// ❌ WRONG — assumes PORT is always set and always a valid number
const port = process.env.PORT;        // could be undefined, or the string "abc"
app.listen(port);                      // crashes with a confusing error deep in Express/net

// ✅ CORRECT — validate and coerce at startup, fail with a clear message immediately
const rawPort = process.env.PORT;
const port = Number(rawPort) || 3000;              // fallback if missing or non-numeric
if (rawPort && Number.isNaN(Number(rawPort))) {
  throw new Error(`Invalid PORT value: "${rawPort}" — must be a number`);
}
app.listen(port);
// (convict/joi/zod do this validation automatically and more thoroughly — see Example 2)
```

### Mistake 3 — Branching application logic on NODE_ENV all over the codebase

```js
// ❌ WRONG — NODE_ENV checks scattered across many files, hard to maintain and test
// File: src/services/email.js
if (process.env.NODE_ENV === 'production') {
  sendRealEmail(userEmail, message);
} else {
  console.log('Would have sent email:', message);   // logic hidden inside a random file
}

// ✅ CORRECT — read NODE_ENV once in config, expose a named FEATURE flag instead
// File: src/config/index.js
const config = {
  features: {
    sendRealEmails: process.env.NODE_ENV === 'production',
  },
};

// File: src/services/email.js
const config = require('../config');
function notifyUser(userEmail, message) {
  if (config.features.sendRealEmails) {
    return sendRealEmail(userEmail, message);
  }
  console.log('[dev] Skipped real email:', message);   // clear, centralized, testable
}
```

---

## Practice exercises

### Exercise 1 — easy

Create a `.env` file in a small project with three variables: `PORT`, `APP_NAME`, and `LOG_LEVEL`. Write a script that:
1. Loads the `.env` file using `dotenv`
2. Reads `NODE_ENV` (default to `'development'` if not set)
3. Prints a single line like: `[development] MyApp is starting on port 3000 (log level: debug)`
4. Run it twice — once normally, once with `NODE_ENV=production node yourfile.js` — and confirm the printed environment name changes.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a config loader that supports four environments: `development`, `test`, `staging`, `production`. Create four separate `.env.<environment>` files, each defining a different `DATABASE_URL` and `PORT`. Write a `loadConfig()` function that:
1. Reads `NODE_ENV` to determine which file to load (default `'development'`)
2. Loads the matching `.env.<environment>` file manually (you may use `dotenv-flow` or read the file yourself with `fs` and `dotenv.parse`)
3. Throws a clear error if `DATABASE_URL` ends up missing after loading
4. Returns a frozen object: `{ env, port, databaseUrl }`

Test it by running your script with `NODE_ENV` set to each of the four values and confirming the correct file loads each time.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a full config module using `convict` (or hand-roll equivalent validation logic if convict isn't installed) that:
1. Defines a schema covering: `env` (enum of the 4 environments), `server.port` (must be a valid port number), `db.connectionString` (required string, no default), `db.poolSize` (natural number, default 10), and `auth.jwtSecret` (required string, marked sensitive)
2. Loads environment-specific `.env.<environment>` files before validating (layer this on top of Exercise 2's loading logic)
3. Calls `.validate({ allowed: 'strict' })` and catches validation errors, re-throwing them with a message that lists exactly which required field is missing
4. Exposes a `redactedSummary()` function that returns the full config as a plain object but with `jwtSecret` and `connectionString` replaced by `"***REDACTED***"` — safe to log at startup
5. Freezes the final config object so nothing downstream can mutate it

Test it by deliberately omitting `JWT_SECRET` from one environment's file and confirming your app throws a clear, actionable error at startup instead of failing later mid-request.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
NODE_ENV VALUES (convention, not enforced by Node)
  development   → local machine, verbose logs, hot reload, fake/sample data
  test          → CI runs, automated tests, isolated throwaway database
  staging       → production-like environment for final checks before release
  production    → live server, real users, real data, minimal logging noise

READING IT
  process.env.NODE_ENV                    → raw string or undefined
  const env = process.env.NODE_ENV || 'development';   → always give a fallback

FILE LAYERING (dotenv-flow convention)
  .env                  → shared defaults, safe to commit
  .env.development       → dev-only overrides
  .env.test               → test-only overrides
  .env.staging             → staging-only overrides
  .env.production          → usually NOT a file — real values injected by platform
  .env.local                → personal machine overrides, gitignored, highest priority
  .env.example                → committed template showing required keys, no real values

LIBRARIES
  dotenv        → loads ONE .env file into process.env, zero validation
  dotenv-flow   → loads MULTIPLE files based on NODE_ENV, layers overrides
  convict       → schema + type validation + defaults + sensitive-value masking
  joi / zod     → alternative schema validators, often used for the same job

RULES
  Never commit real secrets — only .env.example with placeholders
  Always validate required env vars at startup — fail fast, not mid-request
  Never scatter `if (NODE_ENV === 'production')` across the codebase —
    read it once into a config object, expose named feature flags instead
  Real process env vars (set by the OS/platform) always beat values from a .env file
  Freeze the final config object — nothing should mutate it after startup

STARTING WITH THE RIGHT ENV
  NODE_ENV=production node server.js         (Linux/Mac)
  cross-env NODE_ENV=production node server.js   (Windows-safe, via cross-env package)
```

---

## Connected topics

- **12 — Environment variables** — the `.env` file basics and `dotenv` fundamentals this topic builds directly on top of
- **79 — Secrets management** — why production secrets (JWT keys, DB passwords) should never live in committed `.env` files at all
- **112 — 12-Factor App methodology** — "config in the environment" is Factor III, the formal principle behind everything in this topic
