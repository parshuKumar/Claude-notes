# 114 — Building npm packages

## What is this?

Building an npm package means turning a folder of reusable code into something anyone can install with `npm install your-package` and `require()` or `import` into their own project. It is the same skill you use every day when you run `npm install express` — except now you are the one publishing the box, not just opening it. Think of it like packaging a product for a store shelf: the code inside is the same code you'd write in any project, but you now also need a label (`package.json`), instructions for both "readers" (CommonJS and ES Modules consumers), and a version number so people know what they're getting.

---

## Why does it matter for backend development?

Backend teams constantly extract shared logic — a custom logger, a validation helper, a database retry wrapper, an internal auth-token utility — into its own package so multiple services can reuse it instead of copy-pasting code across repos. Once you publish that logic as a proper npm package (privately to your org's registry, or publicly to npmjs.com), every microservice just does `npm install @yourorg/auth-kit` and gets updates via version bumps instead of manual file copying. Getting the `exports` field and dual CJS/ESM support right matters because your package will be consumed by projects using `require()`, projects using `import`, and bundlers like Vite or esbuild — if you get it wrong, half your consumers get confusing errors on install day one.

---

## Syntax / API

```js
// ── package.json — the label on your package ────────────────────────────────
{
  "name": "@yourorg/api-response-utils",   // scoped name — avoids collisions on npm registry
  "version": "1.0.0",                       // semver: MAJOR.MINOR.PATCH — bump on every publish
  "description": "Shared response helpers for internal REST APIs",
  "main": "dist/index.cjs",                 // fallback entry for old tools that ignore "exports"
  "module": "dist/index.mjs",               // hint for some bundlers (not standard, but widely honored)
  "types": "dist/index.d.ts",               // TypeScript type declarations entry point
  "exports": {                              // modern entry map — Node reads THIS first
    ".": {
      "require": "./dist/index.cjs",        // what require('@yourorg/api-response-utils') resolves to
      "import": "./dist/index.mjs",         // what import '@yourorg/api-response-utils' resolves to
      "types": "./dist/index.d.ts"          // what TypeScript resolves to
    },
    "./package.json": "./package.json"      // explicitly allow consumers to read your package.json
  },
  "files": ["dist"],                        // ONLY these paths get published — keeps the tarball small
  "engines": { "node": ">=18" },            // minimum Node version your package supports
  "scripts": {
    "build": "node build.js",               // compiles src/ into dist/ (both CJS and ESM)
    "prepublishOnly": "npm run build"        // runs automatically before every `npm publish`
  }
}
```

```bash
# ── Versioning commands (semver: MAJOR.MINOR.PATCH) ─────────────────────────
npm version patch      # 1.0.0 -> 1.0.1  bug fixes, no API changes
npm version minor      # 1.0.1 -> 1.1.0  new features, backward compatible
npm version major       # 1.1.0 -> 2.0.0  breaking changes

# ── Publishing commands ──────────────────────────────────────────────────────
npm login                          # authenticate with the npm registry
npm publish --access public        # required flag for scoped packages (@yourorg/...) to be public
npm publish                        # unscoped packages default to public already
npm unpublish pkg@1.0.0 --force    # remove a bad version (only allowed within 72 hours, use sparingly)
```

---

## How it works — line by line

When another project runs `require('@yourorg/api-response-utils')` or `import '@yourorg/api-response-utils'`, Node.js does not guess how to load your code — it reads your `package.json`'s `exports` field like a routing table.

- Node sees the consumer used `require(...)` → it looks under `exports["."].require` → loads `dist/index.cjs`
- Node sees the consumer used `import ... from`  → it looks under `exports["."].import` → loads `dist/index.mjs`
- A bundler resolving types for autocomplete → follows `exports["."].types` → loads `dist/index.d.ts`

This is why you ship **two separate built files** (`.cjs` and `.mjs`) instead of one — CommonJS and ES Modules have different syntax at the top level (`module.exports` vs `export`), so one file cannot serve both worlds. The `files` field in `package.json` is a whitelist — when you run `npm publish`, npm packs only `dist/` (plus `package.json`, `README.md`, `LICENSE` which are always included) into the tarball, keeping your source, tests, and config out of consumers' `node_modules`. The `prepublishOnly` script guarantees you never publish stale or missing `dist/` files — npm runs it automatically right before uploading.

---

## Example 1 — basic

```js
// File: my-first-package/package.json
{
  "name": "sleep-ms",                 // simple unscoped package name
  "version": "1.0.0",                 // starting version — always 1.0.0 for a first public release
  "description": "A tiny promise-based delay helper",
  "main": "index.js",                 // single CommonJS entry — no dual build needed for a package this small
  "files": ["index.js"],              // only ship the one file that matters
  "license": "MIT"                    // required for most public packages to be usable at companies
}
```

```js
// File: my-first-package/index.js
// A CommonJS module — the simplest possible npm package, no build step required

// Returns a Promise that resolves after `ms` milliseconds — useful for retries, rate limiting, tests
function sleepMs(ms) {
  return new Promise((resolve) => {
    setTimeout(resolve, ms);   // resolve() with no value — caller just awaits the delay
  });
}

module.exports = sleepMs;       // export a single function — consumers do: const sleepMs = require('sleep-ms')
```

```js
// File: consumer-project/server.js
// How another project would use the package above after `npm install sleep-ms`

const sleepMs = require('sleep-ms');   // loads index.js via package.json "main"

async function retryFetch(url) {
  for (let attempt = 1; attempt <= 3; attempt++) {
    try {
      return await fetch(url);          // try the request
    } catch (err) {
      console.log(`Attempt ${attempt} failed, retrying...`);
      await sleepMs(500 * attempt);      // wait longer each retry — 500ms, 1000ms, 1500ms
    }
  }
  throw new Error('All retry attempts failed');
}
```

---

## Example 2 — real world backend use case

```js
// File: db-retry-kit/src/index.js
// Source file — written once, compiled into BOTH dist/index.cjs and dist/index.mjs by the build script

// Retries an async database operation with exponential backoff — used across multiple internal services
async function withRetry(dbOperation, { retries = 3, baseDelayMs = 200 } = {}) {
  let lastError;

  for (let attempt = 0; attempt < retries; attempt++) {
    try {
      return await dbOperation();               // run the caller's DB query function
    } catch (err) {
      lastError = err;
      const delay = baseDelayMs * 2 ** attempt;  // 200ms, 400ms, 800ms — exponential backoff
      await new Promise((r) => setTimeout(r, delay));
    }
  }

  throw new Error(`DB operation failed after ${retries} attempts: ${lastError.message}`);
}

module.exports = { withRetry };   // CommonJS export — this file itself stays CJS for the build script to read
```

```js
// File: db-retry-kit/build.js
// A tiny build script using esbuild — compiles src/index.js into both output formats

const esbuild = require('esbuild');   // dev dependency, not a runtime dependency for consumers

// Build 1 — CommonJS bundle for require() consumers
esbuild.buildSync({
  entryPoints: ['src/index.js'],
  outfile: 'dist/index.cjs',
  format: 'cjs',                       // outputs module.exports style code
  platform: 'node',
  target: 'node18',
});

// Build 2 — ES Module bundle for import consumers
esbuild.buildSync({
  entryPoints: ['src/index.js'],
  outfile: 'dist/index.mjs',
  format: 'esm',                        // outputs export {} style code
  platform: 'node',
  target: 'node18',
});

console.log('Build complete: dist/index.cjs and dist/index.mjs');
```

```json
// File: db-retry-kit/package.json
{
  "name": "@yourorg/db-retry-kit",
  "version": "1.2.0",
  "description": "Retry wrapper with exponential backoff for database calls",
  "main": "dist/index.cjs",
  "exports": {
    ".": {
      "require": "./dist/index.cjs",
      "import": "./dist/index.mjs"
    }
  },
  "files": ["dist"],
  "engines": { "node": ">=18" },
  "scripts": {
    "build": "node build.js",
    "prepublishOnly": "npm run build"
  },
  "devDependencies": {
    "esbuild": "^0.21.0"
  }
}
```

```js
// File: order-service/src/repositories/orderRepository.js
// How a real backend service consumes the published package

const { withRetry } = require('@yourorg/db-retry-kit');   // installed via npm install @yourorg/db-retry-kit
const dbConnection = require('../db');                     // this service's own pg/mongoose connection

async function findOrderById(orderId) {
  // Wrap the flaky/transient-failure-prone query in the shared retry helper
  return withRetry(() => dbConnection.query('SELECT * FROM orders WHERE id = $1', [orderId]), {
    retries: 3,
    baseDelayMs: 300,
  });
}

module.exports = { findOrderById };
```

---

## Common mistakes

### Mistake 1 — Forgetting the `exports` field, breaking ESM consumers

```js
// ❌ WRONG — only "main" is set; ESM consumers get a confusing resolution error
// package.json
{
  "name": "@yourorg/api-response-utils",
  "main": "dist/index.cjs"
}
// A project using `import { formatResponse } from '@yourorg/api-response-utils'`
// may fail or accidentally load the CJS file wrapped in a way that breaks named imports

// ✅ CORRECT — explicit exports map tells Node exactly which file to load per module system
{
  "name": "@yourorg/api-response-utils",
  "main": "dist/index.cjs",
  "exports": {
    ".": {
      "require": "./dist/index.cjs",   // require() consumers get this
      "import": "./dist/index.mjs"     // import consumers get this
    }
  }
}
```

### Mistake 2 — Publishing your entire source folder, including secrets and tests

```json
// ❌ WRONG — no "files" field means npm publishes almost everything, including
// .env.test, node_modules artifacts left over locally, test fixtures, and src/
{
  "name": "@yourorg/auth-kit",
  "version": "1.0.0",
  "main": "dist/index.cjs"
}
// Run `npm pack --dry-run` and you'd see test/, .env.example, coverage/ all bundled in

// ✅ CORRECT — "files" whitelists exactly what ships in the tarball
{
  "name": "@yourorg/auth-kit",
  "version": "1.0.0",
  "main": "dist/index.cjs",
  "files": ["dist", "README.md"]
}
// A .npmignore file works too, but "files" is safer — it's an allowlist, not a denylist
```

### Mistake 3 — Bumping the version manually and forgetting to rebuild first

```bash
# ❌ WRONG — edits package.json version by hand, then publishes stale dist/ files
# (developer changed src/index.js but forgot to run the build script)
npm publish
# Consumers now install v1.3.0 but get v1.2.0's actual behavior — silent bug

# ✅ CORRECT — let npm bump the version AND rely on prepublishOnly to force a fresh build
npm version patch     # bumps package.json version + creates a git tag automatically
npm publish           # prepublishOnly script runs `npm run build` before uploading — always fresh
```

---

## Practice exercises

### Exercise 1 — easy

Create a small standalone npm package folder called `slugify-lite` (you do not need to publish it — just build it locally).

1. Write `index.js` with a CommonJS-exported function `slugify(text)` that lowercases a string, trims it, and replaces spaces with hyphens (e.g. `"Hello World"` → `"hello-world"`)
2. Write a `package.json` with `name`, `version`, `main`, `description`, and a `files` array containing only `index.js`
3. In a separate test file in the same folder, `require()` your package by its **relative path** (`require('./index.js')`) and log the result of `slugify('  My First Backend Project  ')`

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a package called `http-status-helper` that provides response-shaping utilities for Express-style backends.

1. Create `src/index.js` exporting two functions: `successResponse(data)` which returns `{ ok: true, data }`, and `errorResponse(message, code = 500)` which returns `{ ok: false, error: message, code }`
2. Write a `package.json` with `name`, `version`, `main: "dist/index.cjs"`, an `exports` map pointing `require` at `dist/index.cjs` and `import` at `dist/index.mjs`, and a `files` array limited to `dist`
3. Manually write both `dist/index.cjs` (using `module.exports`) and `dist/index.mjs` (using `export`) by hand — no bundler needed for this exercise — so both point to logically identical code
4. Write a small consumer script that does `const { successResponse } = require('./dist/index.cjs')` and logs the result for `successResponse({ userId: 'user_204' })`

```js
// Write your code here
```

---

### Exercise 3 — hard

Design and build a complete dual-format package called `@practice/rate-limiter-kit` meant to be shared across multiple internal Express services.

1. `src/index.js` should export a function `createRateLimiter({ windowMs, maxRequests })` that returns middleware-style function `(req, res, next)` tracking request counts per `req.ip` in an in-memory `Map`, resetting counts after `windowMs`, and calling `next()` if under the limit or responding with a 429 status and `{ error: 'Too many requests' }` if over
2. Write a `build.js` script (using `esbuild`, or hand-write the two output files if you don't want the dependency) that produces both `dist/index.cjs` and `dist/index.mjs` from `src/index.js`
3. Write a `package.json` with a full `exports` map (`require`, `import`), a `files` field restricted to `dist`, an `engines` field requiring Node 18+, a `build` script, and a `prepublishOnly` script that runs the build
4. Bump the version using `npm version minor` and confirm in `package.json` that the version changed and a git tag was created (skip actually running `npm publish` — just verify the package would be ready)
5. Write a short consumer example showing an Express app using `createRateLimiter({ windowMs: 60000, maxRequests: 100 })` as middleware on a `/api/orders` route

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
PACKAGE.JSON FIELDS THAT MATTER
  name         → unscoped ("my-pkg") or scoped ("@org/my-pkg") — scoped avoids name collisions
  version      → semver MAJOR.MINOR.PATCH — bump every publish, never reuse a version
  main         → fallback entry point for tools that don't read "exports"
  exports      → modern routing table — Node reads this FIRST, overrides "main"
  types        → path to .d.ts file for TypeScript consumers
  files        → allowlist of what gets published (folders/files only)
  engines      → { "node": ">=18" } — documents/enforces minimum Node version

EXPORTS FIELD SHAPE
  "exports": {
    ".": {
      "require": "./dist/index.cjs",   // require() resolves here
      "import":  "./dist/index.mjs",   // import resolves here
      "types":   "./dist/index.d.ts"   // TS tooling resolves here
    }
  }

SEMVER RULES (MAJOR.MINOR.PATCH)
  npm version patch   → 1.0.0 -> 1.0.1   bug fix, no API change
  npm version minor   → 1.0.1 -> 1.1.0   new feature, backward compatible
  npm version major   → 1.1.0 -> 2.0.0   breaking change
  Each command updates package.json AND creates a git commit + tag automatically

PUBLISHING WORKFLOW
  npm login                        → authenticate once per machine
  npm run build                    → produce fresh dist/ files
  npm version patch|minor|major    → bump version, tag commit
  npm publish --access public      → required for scoped packages to be non-private
  npm publish                      → unscoped packages are public by default

DUAL CJS/ESM CHECKLIST
  [ ] src/ has ONE source of truth (write once)
  [ ] build script outputs dist/index.cjs (format: cjs)
  [ ] build script outputs dist/index.mjs (format: esm)
  [ ] exports field maps require -> .cjs, import -> .mjs
  [ ] prepublishOnly script runs the build automatically before every publish

NEVER DO
  Ship src/ or tests/ in the tarball        → bloats install size, leaks internals
  Hand-edit version without npm version     → forgets git tag, easy to desync
  Publish without running build first        → stale dist/, silent bugs for consumers
  Skip "files" field                         → publishes .env, node_modules debris, coverage/
```

---

## Connected topics

- **10 — package.json in depth** — every field used here (`main`, `exports`, `engines`, semver ranges) is defined in full there
- **09 — ES Modules in Node** — understanding `import`/`export` syntax is required before you can produce a correct `.mjs` build
- **113 — Building a CLI tool** — CLI tools are npm packages too; this topic's publishing and versioning steps apply directly to shipping a CLI with `npx`
