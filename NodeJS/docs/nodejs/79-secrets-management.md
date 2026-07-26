# 79 — Secrets management

## What is this?

Secrets management is the practice of storing sensitive values — database passwords, API keys, JWT signing keys, third-party tokens — outside your source code and outside your git history, then loading them safely into your app at runtime. Think of it like a hotel safe in your room: you never leave your passport and cash lying on the desk (in the code) or photograph them into the hotel's guestbook (git history) — you lock them in the safe (a secrets store) and only you have the combination (access credentials) to open it when needed.

## Why does it matter for backend development?

A backend server almost always talks to a database, a payment provider, an email service, and issues signed tokens — every one of those requires a secret. If a secret is hardcoded and the repo becomes public, gets forked, or a laptop is stolen, an attacker gets instant access to production systems. Backend developers are the ones who choose *how* secrets travel from "a value that exists" to "a value the running process can read" — via `.env` files locally, and via a managed secrets store (AWS Secrets Manager, Vault, GCP Secret Manager) in production — and who set up rotation so a leaked secret has a short shelf life.

---

## Syntax / API

```js
// ── Local development: dotenv ────────────────────────────────────────────────
// npm install dotenv
require('dotenv').config();          // reads .env file, populates process.env

const dbPassword = process.env.DB_PASSWORD;   // read a secret from the environment
const apiKey     = process.env.PAYMENT_API_KEY;

// ── .env file (NEVER committed to git) ───────────────────────────────────────
// DB_PASSWORD=s3cr3t_local_only
// PAYMENT_API_KEY=sk_test_xxx

// ── .env.example file (COMMITTED to git — documents what's needed, no values) ─
// DB_PASSWORD=
// PAYMENT_API_KEY=

// ── .gitignore (MUST include this line) ──────────────────────────────────────
// .env

// ── Production: AWS Secrets Manager (fetch at startup, not per-request) ─────
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

const client = new SecretsManagerClient({ region: 'ap-south-1' });   // AWS region

async function loadSecret(secretName) {
  const command  = new GetSecretValueCommand({ SecretId: secretName }); // build request
  const response = await client.send(command);                        // fetch from AWS
  return JSON.parse(response.SecretString);                           // parse JSON secret
}
```

---

## How it works — line by line

`require('dotenv').config()` opens a file named `.env` in your project root, reads it line by line, and for every `KEY=VALUE` pair it finds, it sets `process.env.KEY = 'VALUE'` — but only if that key isn't already set. This means real environment variables set by your hosting platform (Docker, Kubernetes, AWS) always win over the `.env` file, which is exactly what you want: `.env` is a local development convenience, not how production gets its secrets.

`.env.example` is a template file with the same key names but empty or placeholder values. It gets committed to git so any teammate who clones the repo knows exactly which environment variables they need to create locally — it documents the shape of the secrets without exposing any actual value.

`.gitignore` containing `.env` tells git to never track that file. Even if you `git add .` by accident, git skips it. This is the single most important line standing between "secret stays local" and "secret ends up on GitHub forever."

In production, instead of a `.env` file sitting on disk (which can be read by anyone with server access, or accidentally baked into a Docker image), a secrets manager like AWS Secrets Manager stores the value encrypted at rest, control who/what can fetch it via IAM permissions, and lets you rotate it without redeploying code — the app just fetches the latest version the next time it starts (or on a scheduled refresh).

---

## Example 1 — basic

```js
// File: src/config/env.js
// Loads secrets locally with dotenv, with a safety check that fails loudly
// if a required secret is missing — better to crash at startup than at 3am in prod.

require('dotenv').config();   // populate process.env from .env file (dev only)

// List every secret this app absolutely needs to run
const requiredSecrets = ['DB_PASSWORD', 'JWT_SECRET', 'PAYMENT_API_KEY'];

// Check each one exists — fail fast instead of crashing later mid-request
for (const key of requiredSecrets) {
  if (!process.env[key]) {
    // Never log the actual secret — only log which key is missing
    console.error(`[env] Missing required secret: ${key}`);
    process.exit(1);   // stop the process — don't run half-configured
  }
}

// Export a clean config object — the rest of the app never touches process.env directly
module.exports = {
  dbPassword:   process.env.DB_PASSWORD,
  jwtSecret:    process.env.JWT_SECRET,
  paymentApiKey: process.env.PAYMENT_API_KEY,
  nodeEnv:      process.env.NODE_ENV || 'development',
};
```

---

## Example 2 — real world backend use case

```js
// File: src/config/secrets.js
// Real pattern: use dotenv locally, AWS Secrets Manager in production.
// One config module — the rest of the codebase never knows which source was used.

const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

let cachedSecrets = null;   // in-memory cache — fetch once at startup, not per-request

async function loadSecretsFromAws(secretName) {
  const client  = new SecretsManagerClient({ region: process.env.AWS_REGION || 'ap-south-1' });
  const command = new GetSecretValueCommand({ SecretId: secretName });

  try {
    const response = await client.send(command);          // network call to AWS
    return JSON.parse(response.SecretString);              // secret stored as JSON blob
  } catch (err) {
    // Never crash-log the secret name's contents — only the operation that failed
    console.error('[secrets] Failed to fetch from AWS Secrets Manager:', err.message);
    throw err;   // let the caller decide whether to retry or exit
  }
}

async function loadSecrets() {
  if (cachedSecrets) return cachedSecrets;   // return cached values on repeat calls

  if (process.env.NODE_ENV === 'production') {
    // Production: pull the whole secret bundle from AWS in one call
    cachedSecrets = await loadSecretsFromAws('prod/backend-api/db-and-keys');
  } else {
    // Local/dev: dotenv already populated process.env from .env
    require('dotenv').config();
    cachedSecrets = {
      dbPassword:    process.env.DB_PASSWORD,
      jwtSecret:     process.env.JWT_SECRET,
      paymentApiKey: process.env.PAYMENT_API_KEY,
    };
  }

  return cachedSecrets;
}

module.exports = { loadSecrets };

// Usage in server.js — fetch once, before starting the server:
// const { loadSecrets } = require('./config/secrets');
// (async () => {
//   const secrets = await loadSecrets();
//   const dbConnection = await connectToDatabase(secrets.dbPassword);
//   app.listen(3000);
// })();
```

---

## Common mistakes

### Mistake 1 — Hardcoding secrets directly in source code

```js
// ❌ WRONG — the API key is now permanently in git history, even if you delete it later
const stripeClient = require('stripe')('sk_live_51H8xYz9KJp2Q3rEXAMPLE');

// ✅ CORRECT — read it from the environment, never write the literal value in code
const stripeApiKey  = process.env.STRIPE_SECRET_KEY;
const stripeClient  = require('stripe')(stripeApiKey);
// The actual key lives in .env locally, or AWS Secrets Manager in production
```

### Mistake 2 — Forgetting to add .env to .gitignore before the first commit

```js
// ❌ WRONG — .env gets committed on day one, secret is now in git history forever
// (even after later adding .env to .gitignore and deleting the file)
// $ git add .
// $ git commit -m "initial commit"   ← .env silently included

// ✅ CORRECT — add .gitignore BEFORE the first commit that touches .env
// File: .gitignore
// .env
// .env.local
// .env.*.local

// If a secret was ALREADY committed, adding .gitignore is not enough —
// you must rotate the leaked secret immediately (treat it as compromised)
// and use tools like git-filter-repo to scrub it from history.
```

### Mistake 3 — Fetching secrets from AWS on every single request

```js
// ❌ WRONG — every incoming HTTP request triggers a network call to AWS,
// adding latency and risking rate limits / throttling under load
app.get('/api/orders', async (req, res) => {
  const secrets = await loadSecretsFromAws('prod/backend-api/db-and-keys'); // slow, repeated
  const dbConnection = await connectToDatabase(secrets.dbPassword);
  const orders = await dbConnection.query('SELECT * FROM orders');
  res.json(orders);
});

// ✅ CORRECT — fetch once at startup (or on a scheduled interval for rotation),
// cache in memory, reuse across all requests
const { loadSecrets } = require('./config/secrets');
let secrets;

async function startServer() {
  secrets = await loadSecrets();                          // fetched ONCE
  const dbConnection = await connectToDatabase(secrets.dbPassword);

  app.get('/api/orders', async (req, res) => {
    const orders = await dbConnection.query('SELECT * FROM orders');   // no re-fetch
    res.json(orders);
  });

  app.listen(3000);
}
startServer();
```

---

## Practice exercises

### Exercise 1 — easy

Set up a project that reads secrets safely with dotenv:
1. Create a `.env` file with keys `DB_PASSWORD`, `JWT_SECRET`, and `EMAIL_API_KEY` (use fake values)
2. Create a `.env.example` file with the same keys but empty values
3. Create a `.gitignore` file that ignores `.env` (but NOT `.env.example`)
4. Write a script `src/loadEnv.js` that uses `dotenv` to load the `.env` file, then logs each key name and whether it was successfully loaded (log only `"DB_PASSWORD: loaded"`, never the actual value)

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a module `src/config/validateEnv.js` that:
1. Exports a function `validateEnv(requiredKeys)` that accepts an array of key names (e.g. `['DB_PASSWORD', 'JWT_SECRET']`)
2. Checks that every key exists in `process.env` and is not an empty string
3. Collects ALL missing keys (don't stop at the first one) into an array
4. If any keys are missing, throws an `Error` listing every missing key name in the message (still never printing any actual secret values)
5. If all keys are present, returns an object containing only those requested keys, mapped from `process.env`

Then write a small test script that:
- Sets only `DB_PASSWORD` and `JWT_SECRET` via `process.env` (skip `EMAIL_API_KEY`)
- Calls `validateEnv(['DB_PASSWORD', 'JWT_SECRET', 'EMAIL_API_KEY'])`
- Catches the thrown error and logs its message to confirm it correctly reports `EMAIL_API_KEY` as missing

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `SecretsProvider` class that simulates a production-grade secrets loader with rotation support:
1. Constructor accepts `{ source, ttlMs }` where `source` is either `'env'` (reads from `process.env`, for local dev) or `'mock-aws'` (simulate a remote call using a hardcoded fake secrets object, with an artificial `setTimeout` delay of 200ms to mimic network latency)
2. Has an async method `getSecret(key)` that:
   - Returns the cached value if it was fetched less than `ttlMs` milliseconds ago
   - Otherwise re-fetches (from `process.env` or the mock AWS source depending on `source`), updates the cache and the cache timestamp, then returns the value
3. Has a method `forceRotate()` that clears the cache immediately, forcing the next `getSecret()` call to re-fetch (simulating what happens after a secret rotation event)
4. Never logs the actual secret value anywhere — only logs `"[SecretsProvider] fetched fresh copy of <key>"` when a real fetch happens, and `"[SecretsProvider] served <key> from cache"` when cache is used

Test it:
```js
const provider = new SecretsProvider({ source: 'mock-aws', ttlMs: 5000 });
await provider.getSecret('DB_PASSWORD');   // should log "fetched fresh copy"
await provider.getSecret('DB_PASSWORD');   // should log "served from cache"
provider.forceRotate();
await provider.getSecret('DB_PASSWORD');   // should log "fetched fresh copy" again
```

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
GOLDEN RULE
  Secrets never live in source code, and never live in git history — ever

LOCAL DEVELOPMENT
  .env             → actual secret values, MUST be in .gitignore
  .env.example     → same key names, empty values, COMMITTED to git
  require('dotenv').config()  → loads .env into process.env at startup

PRODUCTION
  Use a managed secrets store, not a .env file on disk:
    AWS Secrets Manager   → GetSecretValueCommand, IAM-controlled access
    HashiCorp Vault       → dynamic secrets, short-lived credentials
    GCP Secret Manager    → versioned secrets, IAM bindings
  Fetch ONCE at startup (or on a TTL/interval), cache in memory —
  never fetch a secret inside a per-request handler

.gitignore MUST INCLUDE
  .env
  .env.local
  .env.*.local

IF A SECRET LEAKS INTO GIT
  1. Rotate it immediately — treat it as compromised, deleting the file is NOT enough
  2. Scrub history with git-filter-repo or BFG Repo-Cleaner
  3. Force-push the cleaned history (coordinate with the team first)

SECRET ROTATION
  Rotate on a schedule (e.g. every 90 days) AND immediately after any suspected leak
  AWS Secrets Manager supports automatic rotation via a Lambda function
  App should re-fetch periodically or on a signal, not require a redeploy

NEVER DO
  const apiKey = 'sk_live_xxx';        → hardcoded secret in source
  console.log(process.env.DB_PASSWORD) → secret printed to logs
  git add .env                         → secret committed to git
  Fetching secrets inside a route handler on every request → latency + rate limits

ALWAYS DO
  process.env.SECRET_KEY_NAME          → read from environment
  Validate required secrets exist at startup, fail fast if missing
  Cache fetched secrets in memory with a TTL for rotation support
```

---

## Connected topics

- **12 — Environment variables** — `.env` files, dotenv, `process.env` fundamentals that this topic builds directly on
- **05 — The process object** — `process.env` is the actual mechanism secrets are read through at runtime
- **74 — Security fundamentals in Node** — secrets management is one pillar of the broader attack-surface and least-privilege mindset covered there
