# 12 — Environment variables

## What is this?

Environment variables are key-value pairs that live **outside your code**, in the operating system or process environment, and get read into your app at runtime. Think of them like the settings on a rental car — the car (your code) is identical for every driver, but the seat position, radio station, and fuel level (database URL, API keys, port number) are configured per-driver without touching the car's engine. In Node.js, a `.env` file plus the `dotenv` package is the standard way to load these values into `process.env` during development.

---

## Why does it matter for backend development?

Every backend app needs different settings depending on where it runs — a local database URL on your laptop, a different one in staging, and a production database with real credentials on the live server. Hardcoding any of this (especially passwords, API keys, or JWT secrets) directly into your source code is a security disaster the moment that code reaches GitHub — bots scan public repos for leaked keys within minutes. Environment variables let the exact same code run correctly and securely across every environment, with secrets injected only where they're needed and never committed to version control.

---

## Syntax / API

```js
// Install the dotenv package — reads a .env file and loads it into process.env
// npm install dotenv

// File: .env  (in project root — NEVER commit this file to git)
// PORT=5000
// DB_HOST=localhost
// DB_NAME=my_app_dev
// DB_PASSWORD=supersecret123
// JWT_SECRET=a-long-random-string-here
// NODE_ENV=development

// File: server.js — load .env as the very first line of your app
require('dotenv').config();
// After this line runs, every KEY=value pair from .env is copied into process.env

// Reading values — process.env always gives you STRINGS, never numbers/booleans
const port = process.env.PORT || 3000;          // fallback if PORT is missing
const dbHost = process.env.DB_HOST;               // "localhost"
const dbPassword = process.env.DB_PASSWORD;       // "supersecret123"
const jwtSecret = process.env.JWT_SECRET;         // used to sign/verify tokens

// Numbers must be converted manually — process.env.PORT is the STRING "5000"
const portNumber = Number(process.env.PORT) || 3000;

// Booleans must be compared as strings — there is no true/false in .env files
const isProduction = process.env.NODE_ENV === 'production';
```

---

## How it works — line by line

`require('dotenv').config()` reads the file named `.env` in your project's current working directory, parses each `KEY=value` line, and assigns it to `process.env.KEY = 'value'` — but only if that key does not already exist in `process.env`. This means real environment variables set by your OS, Docker, or a hosting platform always win over the `.env` file, which is intentional: in production you typically don't ship a `.env` file at all — you set real environment variables on the server instead.

Every value coming out of `process.env` is a **string**, because environment variables are a plain-text mechanism at the operating system level — there is no concept of a number or boolean in an environment variable. This is why `process.env.PORT` is `"5000"` (a string) and why comparing `process.env.NODE_ENV === 'production'` works, but `if (process.env.DEBUG)` is always truthy even when `DEBUG=false`, because the non-empty string `"false"` is still truthy in JavaScript.

`dotenv` only ever reads the file — it never writes to it, never talks to a network, and does nothing beyond string parsing. That's why it's safe to call at the very top of your entry file (`server.js` or `index.js`) before anything else runs: every subsequent `require()` in your app can then safely read `process.env` values that are already populated.

---

## Example 1 — basic

```js
// File: .env  (project root, listed in .gitignore, never committed)
// PORT=4000
// APP_NAME=InventoryService
// DEBUG_MODE=true

// File: index.js
require('dotenv').config();
// Loads .env into process.env — must run before anything reads process.env

const port = Number(process.env.PORT) || 3000;
// Convert the string "4000" to a number, fall back to 3000 if missing

const appName = process.env.APP_NAME || 'UnnamedService';
// Read a plain string value with a sensible fallback

const debugMode = process.env.DEBUG_MODE === 'true';
// Compare against the literal string "true" — booleans don't exist in .env

console.log(`Starting ${appName} on port ${port}`);
// → "Starting InventoryService on port 4000"

console.log('Debug mode:', debugMode);
// → "Debug mode: true"  (correctly converted to a real boolean)

console.log('Raw PORT type:', typeof process.env.PORT);
// → "string" — proves process.env values are always strings, even for numbers
```

---

## Example 2 — real world backend use case

```js
// File: config/env.js
// Centralized, validated environment loader — the pattern used in production APIs.
// Load once here, export clean typed values, and require THIS file everywhere else
// instead of reading process.env scattered across the codebase.

require('dotenv').config();

// List every environment variable this app truly needs to run
const requiredVars = ['DB_HOST', 'DB_NAME', 'DB_PASSWORD', 'JWT_SECRET'];

// Fail fast at startup if any required secret/config is missing — never fail
// silently in the middle of handling a user's request
for (const varName of requiredVars) {
  if (!process.env[varName]) {
    console.error(`[config] Missing required environment variable: ${varName}`);
    process.exit(1); // stop the app immediately — better than crashing later
  }
}

// Export a single, typed config object — the rest of the app never touches
// process.env directly, which makes testing and refactoring far easier
module.exports = {
  port: Number(process.env.PORT) || 3000,
  nodeEnv: process.env.NODE_ENV || 'development',
  isProduction: process.env.NODE_ENV === 'production',

  db: {
    host: process.env.DB_HOST,
    name: process.env.DB_NAME,
    password: process.env.DB_PASSWORD, // secret — never logged, never sent to client
  },

  auth: {
    jwtSecret: process.env.JWT_SECRET,           // used to sign tokens
    tokenExpiry: process.env.JWT_EXPIRY || '1h',  // e.g. "1h", "7d"
  },
};

// Usage in server.js:
// const config = require('./config/env');
// app.listen(config.port, () => console.log(`Running in ${config.nodeEnv} mode`));
//
// Usage in db.js:
// const config = require('./config/env');
// const dbConnection = createConnection({ host: config.db.host, password: config.db.password });
```

---

## Common mistakes

### Mistake 1 — Hardcoding secrets directly in source code

```js
// ❌ WRONG — the JWT secret and DB password are now permanently in git history,
// even if you delete this line later, since git keeps every past commit
const jwtSecret = 'my-super-secret-key-2024';
const dbConnection = createConnection({
  host: 'prod-db.company.com',
  password: 'Pr0dPassw0rd!', // leaked the moment this repo goes public
});

// ✅ CORRECT — secrets come from process.env, .env file is gitignored
require('dotenv').config();
const jwtSecret = process.env.JWT_SECRET;
const dbConnection = createConnection({
  host: process.env.DB_HOST,
  password: process.env.DB_PASSWORD,
});
```

### Mistake 2 — Committing the .env file to git

```bash
# ❌ WRONG — .env with real secrets gets pushed to GitHub for everyone to see
git add .env
git commit -m "add config"
git push

# ✅ CORRECT — add .env to .gitignore BEFORE it's ever tracked, and instead
# commit a .env.example with the same KEYS but placeholder values
```
```
# File: .gitignore
.env

# File: .env.example  (SAFE to commit — shows required keys, no real secrets)
PORT=3000
DB_HOST=localhost
DB_NAME=my_app_dev
DB_PASSWORD=changeme
JWT_SECRET=replace-with-a-long-random-string
```

### Mistake 3 — Assuming process.env values are numbers or booleans

```js
// ❌ WRONG — process.env.PORT is the STRING "5000", not a number
const port = process.env.PORT;
app.listen(port + 1); // "5000" + 1 → "50001" (string concatenation, not math!)

// ❌ WRONG — DEBUG=false is a non-empty string, which is always truthy
if (process.env.DEBUG) {
  console.log('This runs even when DEBUG=false in the .env file');
}

// ✅ CORRECT — explicitly convert types when reading from process.env
const port = Number(process.env.PORT) + 1;      // 5000 + 1 → 5001 (real math)
const debugEnabled = process.env.DEBUG === 'true'; // real boolean comparison
if (debugEnabled) {
  console.log('This only runs when DEBUG is literally the string "true"');
}
```

---

## Practice exercises

### Exercise 1 — easy

Create a `.env` file in a small project with these keys: `APP_NAME`, `PORT`, `MAINTENANCE_MODE` (set to `true` or `false` as a string). Write a script that:
1. Loads the `.env` file using `dotenv`
2. Reads all three values from `process.env`
3. Converts `PORT` to a real number and `MAINTENANCE_MODE` to a real boolean
4. Logs a message like `"AppName is running on port 3000 (maintenance: false)"` using the converted values

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a `loadEnvConfig()` function that:
1. Loads environment variables using `dotenv`
2. Defines an array of required variable names: `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`
3. Checks that every required variable is present and non-empty in `process.env`
4. If any are missing, logs which specific ones are missing and calls `process.exit(1)`
5. If all are present, returns a config object shaped like:
   ```
   { db: { host, name, user, password }, port: <number>, isProduction: <boolean> }
   ```

Test it by removing one required key from your `.env` file and confirming the function reports exactly which key is missing before exiting.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small CLI-runnable module `envValidator.js` that acts like a lightweight version of libraries such as `envalid`. It should export a function `validateEnv(schema)` where `schema` is an object like:

```js
{
  PORT: { type: 'number', required: true, default: 3000 },
  DB_HOST: { type: 'string', required: true },
  JWT_SECRET: { type: 'string', required: true },
  ENABLE_CACHE: { type: 'boolean', required: false, default: false },
}
```

Requirements:
1. For each key in the schema, read the raw value from `process.env`
2. If missing and `required: true` with no `default`, collect an error message (don't throw immediately — gather ALL errors first)
3. If missing but a `default` is provided, use the default
4. Convert the value according to `type` (`'number'` → `Number()`, `'boolean'` → compare to `'true'`, `'string'` → leave as-is)
5. If any errors were collected, print all of them together and call `process.exit(1)`
6. If everything is valid, return a fully typed config object matching the schema keys

Test it against a schema with at least 4 keys, including one missing required variable, to confirm all errors are reported together rather than stopping at the first one.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
SETUP
  npm install dotenv
  require('dotenv').config();     ← must run BEFORE anything reads process.env
  Put this line at the very top of your entry file (server.js / index.js)

FILES
  .env             → real values, LOCAL ONLY, NEVER committed (add to .gitignore)
  .env.example     → same keys, placeholder values, SAFE to commit
  .gitignore       → must contain: .env

READING VALUES
  process.env.KEY                  → always a STRING, or undefined if missing
  process.env.PORT || 3000         → string fallback
  Number(process.env.PORT)         → convert to number
  process.env.FLAG === 'true'      → convert to boolean (no real booleans exist)

PRECEDENCE
  Real OS/shell environment variables ALWAYS win over .env file values
  In production: set real env vars on the server/host, skip .env entirely

CROSS-PLATFORM SCRIPTS (cross-env)
  npm install --save-dev cross-env
  "scripts": { "start": "cross-env NODE_ENV=production node server.js" }
  Needed because `NODE_ENV=production node app.js` fails on native Windows cmd
  but works fine on Mac/Linux — cross-env normalizes it everywhere

NEVER DO
  Hardcode passwords/API keys/secrets directly in .js files
  Commit .env to git
  Assume process.env.PORT is a number or process.env.FLAG is a boolean
  Log process.env values that contain secrets (passwords, tokens, keys)

GOOD PATTERNS
  Validate required vars at startup, process.exit(1) if any are missing
  Centralize all process.env reads into one config.js module
  Keep .env.example always in sync with every key your app actually reads
```

---

## Connected topics

- **05 — The process object** — `process.env` is a property of the global `process` object; this topic covers `process.exit()` and `process.argv` used alongside it
- **79 — Secrets management** — goes deeper into never committing secrets, secret rotation, and managing them beyond local `.env` files (AWS Secrets Manager, Vault)
- **107 — Environment management** — covers dev/test/staging/prod configs, `NODE_ENV` conventions, and libraries like `dotenv-flow` and `convict` built on top of this topic
