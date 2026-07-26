# 10 — package.json in depth

## What is this?

`package.json` is the identity card and instruction manual for a Node.js project — a single JSON file that tells Node, npm, and every developer who touches the project what the project is called, what version it's at, what commands can be run, and what other packages it depends on to work. Think of it like a restaurant's business license plus its recipe book combined: the license part identifies the business (name, version, owner), and the recipe book part lists every ingredient (dependency) needed and every action the kitchen can perform (scripts like `start`, `test`, `build`).

## Why does it matter for backend development?

Every backend Node.js project — an Express API, a CLI tool, a microservice — starts with `package.json`. It is how you declare "this project needs `express@^4.18.0` and `pg@^8.11.0`" so anyone who clones your repo can run `npm install` and get the exact working environment. It is also how you define the `start` script your production server runs, the `test` script your CI pipeline runs, and the `type` field that decides whether your files use `require()` or `import`. Get `package.json` wrong — a missing dependency, a bad version range, a wrong `main` field — and deployments fail, teammates get "module not found" errors, or a minor dependency update silently breaks production.

---

## Syntax / API

```json
{
  "name": "order-service",
  // "name" — lowercase, no spaces, used if this package is ever published or imported by name

  "version": "1.4.2",
  // "version" — follows semver: MAJOR.MINOR.PATCH — required for publishing, useful even privately

  "description": "REST API for managing customer orders",
  // "description" — one-line summary, shown on npm registry pages and in search results

  "main": "src/server.js",
  // "main" — the entry file loaded when something does require('order-service')

  "type": "commonjs",
  // "type" — "commonjs" (default) uses require/module.exports; "module" makes .js files use import/export

  "engines": {
    "node": ">=20.0.0"
  },
  // "engines" — the Node.js version range this app is built for; npm/CI can warn or block on mismatch

  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "test": "jest",
    "build": "tsc -p ."
  },
  // "scripts" — named shell commands runnable via "npm run <name>" (start/test have shortcuts)

  "dependencies": {
    "express": "^4.18.2",
    "pg": "~8.11.3"
  },
  // "dependencies" — packages required at RUNTIME in production, installed with "npm install <pkg>"

  "devDependencies": {
    "jest": "^29.7.0",
    "nodemon": "^3.0.1"
  },
  // "devDependencies" — packages only needed while DEVELOPING (testing, hot-reload), not in production

  "private": true
  // "private" — prevents this package from ever being accidentally published to the npm registry
}
```

---

## How it works — line by line

`package.json` is plain JSON — no comments allowed in the real file, no trailing commas, and every key is a string in double quotes. Node and npm read specific fields for specific purposes:

- `name` and `version` together form the package's unique identity. If you publish to npm, these must be unique on the registry.
- `main` tells Node which file to load when another module does `require('your-package-name')`. If omitted, Node defaults to `index.js` in the project root.
- `type` controls how Node interprets every plain `.js` file in the project: `"commonjs"` (or omitted) means `require()`/`module.exports`; `"module"` means `import`/`export` syntax is expected instead.
- `engines` does not enforce anything by itself — it's a declaration. Tools like `npm install --engine-strict` or hosting platforms (Heroku, many CI systems) read it to pick the right Node runtime or warn on mismatch.
- `scripts` is a lookup table of shell commands. `npm run dev` looks up the `"dev"` key and runs its string value in a subshell that also has `node_modules/.bin` on its `PATH` — that's why you can write `"test": "jest"` instead of `"test": "./node_modules/.bin/jest"`.
- `dependencies` vs `devDependencies` is purely organizational for humans and tools — both install the same way, but `npm install --production` (or `npm ci --omit=dev`) skips `devDependencies`, which keeps production Docker images and deployments smaller.
- Version strings like `^4.18.2` are not exact — they are ranges. npm reads the caret/tilde symbol and decides which range of versions satisfies it when resolving what to actually download.

---

## Example 1 — basic

```js
// File: package.json
// A minimal, valid package.json for a small backend project
{
  "name": "todo-api",                     // lowercase project identity
  "version": "1.0.0",                     // starting version, follows semver
  "description": "A simple todo list REST API",  // short summary
  "main": "index.js",                     // entry point for require('todo-api')
  "type": "commonjs",                     // use require/module.exports (default, but explicit is clearer)
  "scripts": {
    "start": "node index.js"              // run with: npm start
  },
  "dependencies": {
    "express": "^4.18.2"                  // runtime dependency — needed to actually run the server
  },
  "engines": {
    "node": ">=18.0.0"                    // requires Node 18 or newer
  }
}
```

```js
// File: index.js — this is what "main" and "scripts.start" point to
const express = require('express');   // loaded from node_modules, listed in dependencies above

const app = express();                // create the Express application
const port = process.env.PORT || 3000; // read port from env, fall back to 3000

app.get('/', (req, res) => {
  res.send('todo-api is running');    // simple health response
});

app.listen(port, () => {
  console.log(`Server listening on port ${port}`); // confirm startup in logs
});
```

---

## Example 2 — real world backend use case

```js
// File: package.json
// A realistic Express + PostgreSQL backend service, ready for a team and CI/CD
{
  "name": "user-auth-service",
  "version": "2.3.0",
  "description": "Authentication microservice — login, signup, JWT issuance",
  "main": "src/server.js",
  "type": "commonjs",
  "private": true,                        // never accidentally publish an internal service to npm

  "engines": {
    "node": ">=20.0.0"                    // matches the Node version used in Docker/CI
  },

  "scripts": {
    "start": "node src/server.js",         // production start command — used by Docker CMD / PM2
    "dev": "nodemon src/server.js",        // local dev — auto-restarts on file changes
    "test": "jest --coverage",             // CI runs "npm test", generates coverage report
    "lint": "eslint src/",                 // catches style/quality issues before merge
    "migrate": "node-pg-migrate up"        // applies pending DB migrations
  },

  "dependencies": {
    "express": "^4.18.2",                 // web framework — patch/minor updates auto-accepted
    "pg": "~8.11.3",                       // Postgres driver — pinned tighter, only patch updates
    "jsonwebtoken": "^9.0.2",             // signs and verifies JWT auth tokens
    "bcrypt": "9.0.0",                    // password hashing — exact version, security-sensitive
    "dotenv": "^16.3.1"                    // loads .env into process.env at startup
  },

  "devDependencies": {
    "jest": "^29.7.0",                    // test runner — never shipped to production
    "supertest": "^6.3.3",                // HTTP assertions for testing Express routes
    "nodemon": "^3.0.1",                  // dev-only auto-restart tool
    "eslint": "^8.53.0"                    // dev-only linting tool
  }
}
```

```bash
# How this file gets used day to day:

npm install               # reads dependencies + devDependencies, installs everything into node_modules
npm ci                    # CI/deploy: installs EXACT versions from package-lock.json, faster & reproducible
npm install --omit=dev    # production install — skips jest/nodemon/eslint entirely
npm start                 # runs "node src/server.js" — what Docker CMD or PM2 actually executes
npm run dev                # local development with auto-restart on save
npm test                  # runs jest with coverage — what your CI pipeline calls automatically
```

---

## Common mistakes

### Mistake 1 — Putting runtime packages in devDependencies (or vice versa)

```js
// ❌ WRONG — express is required to RUN the server, but it's listed as a dev-only dependency
{
  "devDependencies": {
    "express": "^4.18.2"   // if production install uses --omit=dev, the server crashes: "Cannot find module 'express'"
  }
}

// ✅ CORRECT — runtime packages go in dependencies, testing/tooling packages go in devDependencies
{
  "dependencies": {
    "express": "^4.18.2"   // needed in production
  },
  "devDependencies": {
    "jest": "^29.7.0"      // only needed while developing/testing, safe to skip in prod
  }
}
```

### Mistake 2 — Misunderstanding semver ranges and getting surprise breaking changes

```js
// ❌ WRONG — using "*" or no caret/tilde means npm can install ANY future major version,
// including one with breaking changes, the next time someone runs npm install
{
  "dependencies": {
    "express": "*"          // could jump from 4.x to 5.x unexpectedly — code may break
  }
}

// ✅ CORRECT — use ^ to allow safe minor/patch updates within the same major version
{
  "dependencies": {
    "express": "^4.18.2"    // allows 4.18.3, 4.19.0, up to (but not including) 5.0.0
    // "~4.18.2" would allow only 4.18.x (patch-only, more conservative)
    // "4.18.2"  (exact) allows no updates at all — most conservative, requires manual bumps
  }
}
```

### Mistake 3 — Forgetting the "type" field and mixing module syntax

```js
// ❌ WRONG — package.json has no "type" field (defaults to commonjs),
// but the code uses ES Module import/export syntax
// package.json:
{
  "name": "payment-service",
  "main": "src/server.js"
  // no "type" field → Node treats .js files as CommonJS
}

// src/server.js:
import express from 'express';   // SyntaxError: Cannot use import statement outside a module
const app = express();

// ✅ CORRECT — either declare "type": "module" to match the import syntax...
{
  "name": "payment-service",
  "main": "src/server.js",
  "type": "module"          // now .js files support import/export
}

// ...or keep "commonjs" and use require() instead, matching the rest of this course:
const express = require('express');   // works with default "commonjs" type, no config change needed
```

---

## Practice exercises

### Exercise 1 — easy

Create a new folder for a project called `notes-api`. Write a `package.json` file by hand (do not use `npm init`) that includes:
1. A `name` and `version` field
2. A `main` field pointing to `index.js`
3. A `scripts` section with a `start` script that runs `node index.js`
4. A `dependencies` section listing `express` with a caret (`^`) version range
5. An `engines` field requiring Node 18 or higher

Then create the matching `index.js` that starts an Express server on port 4000 and logs a confirmation message.

```js
// Write your code here
```

---

### Exercise 2 — medium

You are given a `package.json` for a project called `inventory-service` that currently has ALL its packages (`express`, `pg`, `jest`, `nodemon`, `eslint`) dumped into a single `dependencies` object. Rewrite the file so that:
1. Only packages actually needed to run the server in production stay in `dependencies`
2. Testing and dev-tooling packages move to `devDependencies`
3. Add a `scripts` section with `start`, `dev` (using nodemon), `test` (using jest), and `lint` (using eslint) entries
4. Add a `type` field explicitly set to `"commonjs"`
5. Add a `private: true` field so it's never accidentally published

Write out the corrected `package.json` and briefly note (as a comment) why each package landed where it did.

```js
// Write your code here
```

---

### Exercise 3 — hard

Design a `package.json` and matching folder structure for a real multi-service backend called `billing-platform` that must satisfy ALL of these constraints:
1. It uses Express, a database driver (`pg`), JWT auth (`jsonwebtoken`), and environment config (`dotenv`) as runtime dependencies
2. It uses Jest, Supertest, and Nodemon as dev-only dependencies
3. It has FIVE scripts: `start` (production), `dev` (nodemon), `test` (jest with coverage), `migrate` (running a migration command of your choice), and `lint` (eslint)
4. Version ranges must reflect intent: `express` should allow minor+patch updates, `pg` should allow ONLY patch updates, and `bcrypt` (a security-sensitive package) should be pinned to an EXACT version with no range at all
5. It declares `"engines"` requiring Node 20+
6. It sets `"private": true` and explains in a comment why that matters for an internal company service
7. Write a short comment block explaining what would happen (which packages get installed, which get skipped) if someone ran `npm install --omit=dev` in this project versus a plain `npm install`

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE IDENTITY FIELDS
  "name"          → lowercase, no spaces, project/package identity
  "version"       → semver: MAJOR.MINOR.PATCH (e.g. 2.3.0)
  "description"   → one-line summary
  "private": true → blocks accidental "npm publish"

ENTRY & MODULE TYPE
  "main"          → file loaded by require('this-package'); defaults to index.js
  "type": "commonjs" → require()/module.exports (default if omitted)
  "type": "module"   → import/export syntax for all .js files

RUNTIME REQUIREMENT
  "engines": { "node": ">=20.0.0" }   → declares required Node version (advisory, not enforced by default)

SCRIPTS
  "scripts": { "start": "...", "dev": "...", "test": "..." }
  Run with: npm run <name>      (start/test/restart/stop have shortcuts: npm start, npm test)
  Has node_modules/.bin on PATH automatically inside scripts

DEPENDENCIES VS DEVDEPENDENCIES
  dependencies     → needed at RUNTIME (express, pg, jsonwebtoken)
  devDependencies  → needed only while DEVELOPING (jest, nodemon, eslint)
  npm install                  → installs both
  npm install --omit=dev       → installs dependencies only (production/CI)
  npm ci                       → clean install from package-lock.json (exact versions, faster, reproducible)

SEMVER RANGES
  ^4.18.2   → allow MINOR and PATCH updates (4.18.3, 4.19.0) but NOT 5.0.0     [most common]
  ~4.18.2   → allow PATCH updates only (4.18.3, 4.18.9) but NOT 4.19.0          [conservative]
  4.18.2    → EXACT version, no updates at all                                  [security-critical pkgs]
  *  or "latest" → any version — AVOID, causes unpredictable breaking updates

GOTCHAS
  No comments allowed in real package.json (this doc uses // for teaching only)
  No trailing commas — invalid JSON crashes npm install
  package-lock.json records EXACT resolved versions — commit it to git, always
  Changing "type" affects EVERY plain .js file in the project, not just one file
```

---

## Connected topics

- **11 — npm in depth** — covers `npm install`, `npm init`, `package-lock.json`, and `node_modules` resolution that all operate on top of this file
- **09 — ES Modules in Node** — the `"type": "module"` field decides whether your `.js` files use `import`/`export` or `require`/`module.exports`
- **12 — Environment variables** — `dotenv` is typically listed as a dependency here and loaded at startup to keep secrets out of the codebase
