# 09 — ES Modules in Node

## What is this?

ES Modules (ESM) are the official, standardized module system for JavaScript — the same `import`/`export` syntax you already write in the browser and in React/Vue apps. Node.js supports ESM natively alongside its original CommonJS (CJS) system. Think of CJS and ESM as two different electrical plug standards for the same house — both deliver power (code), but they use different sockets (`require()` vs `import`), and you generally shouldn't mix them in one file without an adapter.

## Why does it matter for backend development?

Modern backend tooling — TypeScript with `"module": "esnext"`, Vite-based SSR frameworks, and most new npm packages — ships as ESM-first or ESM-only. Some popular packages (like `node-fetch` v3, `chalk` v5, `nanoid`) dropped CJS support entirely, so if your project is CJS-only, you simply cannot `require()` them without workarounds. Knowing how to configure and write ESM lets you use modern packages, share code between frontend and backend, and use native top-level `await`. Most real backend codebases today are still CJS (this whole doc series defaults to it), but every Node backend developer needs to recognize ESM, know when a project needs it, and understand the differences well enough to debug `ERR_REQUIRE_ESM` errors.

---

## Syntax / API

```js
// ── math.mjs — an ES Module file (note the .mjs extension) ─────────────────

// Named export — exports a specific named binding
export function add(a, b) {
  return a + b;                          // simple addition, returned to caller
}

// Named export of a constant
export const TAX_RATE = 0.18;            // any value can be a named export

// Default export — one per file, the "main" thing this module provides
export default function calculateTotal(amount) {
  return amount + amount * TAX_RATE;     // uses the named export above
}

// ── server.mjs — importing from the module above ───────────────────────────

// Named imports — must match the exported names exactly, wrapped in { }
import { add, TAX_RATE } from './math.mjs';

// Default import — any name you like, no curly braces
import calculateTotal from './math.mjs';

// Import everything as one namespace object
import * as mathUtils from './math.mjs';

console.log(add(2, 3));                  // → 5
console.log(TAX_RATE);                   // → 0.18
console.log(calculateTotal(100));        // → 118
console.log(mathUtils.add(1, 1));        // → 2 (same function, via namespace)
```

---

## How it works — line by line

`export` marks a value inside a module as visible to other files — without it, everything declared in a file stays private to that file. There are two kinds: **named exports** (`export const`, `export function`) let you export many things from one file, each imported by its exact name; **default export** (`export default`) marks one value as "the main thing" this file provides, importable under any name you choose.

`import` pulls those exported values into another file. Named imports use curly braces and must match the exported name (`import { add }`), default imports use no braces and any local name (`import calc from ...`), and `import * as name` grabs everything as one object.

Unlike CommonJS's `require()`, which runs synchronously and can happen anywhere in a file (even inside an `if` block), ESM `import` statements are **hoisted to the top** of the file and resolved before any code runs — this is what lets tools statically analyze your dependency graph without executing code, and it's why top-level `await` is safe (Node waits for the whole module graph to settle before your app finishes starting).

### How Node decides "is this file CJS or ESM?"

Node picks the module system per-file using this priority:

```
1. File ends in .mjs                     → ALWAYS ES Module, no exceptions
2. File ends in .cjs                     → ALWAYS CommonJS, no exceptions
3. File ends in .js:
     - nearest package.json has "type": "module"  → ES Module
     - nearest package.json has "type": "commonjs" (or no "type" field) → CommonJS
```

"Nearest" means Node walks up from the file's folder looking for a `package.json` — so you can have CJS and ESM files coexist in the same project by placing a small `package.json` with `{"type":"module"}` inside just the folder that needs it.

---

## Example 1 — basic

```js
// File: greetings.mjs
// A simple ESM module exporting two named functions and one default.

// Named export — a reusable helper
export function formatGreeting(userName) {
  return `Hello, ${userName}!`;          // template literal builds the greeting
}

// Named export — another helper
export function formatFarewell(userName) {
  return `Goodbye, ${userName}.`;        // similar shape, different message
}

// Default export — the "primary" export of this file
export default function greet(userName) {
  return formatGreeting(userName);       // delegates to the named export
}

// File: app.mjs
// Imports and uses the module above.

import greet, { formatFarewell } from './greetings.mjs';
// ^ default import (any name)   ^ named import (must match exactly)

const userName = 'Ravi';                 // the user we're greeting

console.log(greet(userName));            // → "Hello, Ravi!"
console.log(formatFarewell(userName));   // → "Goodbye, Ravi."

// Top-level await works directly in ESM — no async wrapper function needed
const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
await delay(100);                        // pause 100ms before continuing
console.log('Done waiting.');            // runs after the delay
```

---

## Example 2 — real world backend use case

```js
// File: src/db/connection.mjs
// A database connection module written in ESM, using top-level await —
// a common pattern in modern Node backends that need the DB ready
// before the rest of the app boots.

import { MongoClient } from 'mongodb';               // official MongoDB driver (ESM-compatible)

const connectionString = process.env.DB_CONNECTION_STRING;  // read from environment
const client = new MongoClient(connectionString);            // create the client instance

// Top-level await — pauses THIS module's execution until connected
// (only legal in ESM; CJS has no equivalent without an async IIFE)
await client.connect();
console.log('[db] Connected to MongoDB');

const dbConnection = client.db('production_app');    // grab the target database

// Named export — other files import this ready-to-use connection
export { dbConnection, client };

// File: src/routes/users.mjs
// A route handler that imports the already-connected database.

import { dbConnection } from '../db/connection.mjs';  // import the live connection

export async function getUserById(userId) {
  // find one document matching the given userId
  const user = await dbConnection.collection('users').findOne({ userId });

  if (!user) {
    throw new Error(`User not found: ${userId}`);      // fail loudly if missing
  }

  return user;                                          // return the found document
}

// File: server.mjs
// Entry point — package.json here has "type": "module"

import { getUserById } from './src/routes/users.mjs';  // import handler
import http from 'node:http';                            // core module, works fine in ESM

const server = http.createServer(async (req, res) => {
  const userId = new URL(req.url, 'http://localhost').searchParams.get('userId');
  const user = await getUserById(userId);                 // reuse the imported handler
  res.setHeader('Content-Type', 'application/json');
  res.end(JSON.stringify(user));                          // send the user back as JSON
});

server.listen(3000, () => console.log('Server on :3000'));
```

---

## Common mistakes

### Mistake 1 — Mixing require() into a file Node treats as ESM

```js
// ❌ WRONG — this file is server.mjs (or package.json has "type":"module")
// require is not defined in ES Modules — ReferenceError: require is not defined
const path = require('path');
const configPath = path.join(__dirname, 'config.json');   // also crashes: no __dirname either

// ✅ CORRECT — use import, and reconstruct __dirname if you need it (see Topic 06)
import path from 'node:path';
import { fileURLToPath } from 'node:url';

const __filename = fileURLToPath(import.meta.url);
const __dirname  = path.dirname(__filename);
const configPath = path.join(__dirname, 'config.json');    // works correctly
```

### Mistake 2 — Forgetting the file extension in a relative import

```js
// ❌ WRONG — ESM requires explicit extensions for relative imports (unlike CJS)
// Error: Cannot find module '/app/src/utils' imported from '/app/src/server.mjs'
import { hashPassword } from './utils';

// ✅ CORRECT — always include the extension in ESM relative imports
import { hashPassword } from './utils.mjs';
// (CommonJS's require('./utils') can omit the extension — ESM cannot)
```

### Mistake 3 — Assuming "type":"module" only affects a few files

```json
// ❌ WRONG — setting "type": "module" in package.json silently breaks
// every plain .js file in the project that used require()/module.exports
{
  "name": "payment-service",
  "type": "module",
  "main": "index.js"
}
```

```js
// index.js now throws: exports is not defined in ES module scope
const express = require('express');
module.exports = app;
```

```json
// ✅ CORRECT — either convert ALL .js files to import/export syntax,
// OR rename the CJS files you want to keep to .cjs, OR omit "type" entirely
{
  "name": "payment-service",
  "type": "module",
  "main": "index.js"
}
```

```js
// index.js — converted to match "type": "module"
import express from 'express';
export default app;
```

---

## Practice exercises

### Exercise 1 — easy

Create two files: `mathUtils.mjs` and `main.mjs`.
1. In `mathUtils.mjs`, create and export (as named exports) two functions: `celsiusToFahrenheit(celsius)` and `fahrenheitToCelsius(fahrenheit)`.
2. Also add a default export function `roundToTwoDecimals(value)` that rounds a number to 2 decimal places.
3. In `main.mjs`, import all three functions and use them to convert 100°C to Fahrenheit and 32°F to Celsius, rounding both results.
4. Log the final rounded values with descriptive labels.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a tiny ESM-based configuration loader.
1. Create a folder `config/` containing `default.mjs` which exports a default object with keys `port`, `dbName`, and `logLevel`.
2. Create `config/production.mjs` and `config/development.mjs`, each exporting a default object that overrides only some of those keys (e.g. production sets a different `port`).
3. In `loadConfig.mjs`, write an async function `loadConfig(env)` that uses **dynamic `import()`** (not a static import) to load `./config/${env}.mjs`, merges it on top of `default.mjs` using object spread, and returns the merged config.
4. In `main.mjs`, call `loadConfig(process.env.NODE_ENV || 'development')` with top-level await and log the resolved config.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small CJS/ESM interop demo for a real backend scenario.
1. Create a CommonJS file `logger.cjs` that exports (via `module.exports`) an object with `info(message)` and `error(message)` methods that prefix output with `[INFO]` / `[ERROR]` and a timestamp.
2. Create an ESM file `paymentService.mjs` (in a project where `package.json` has `"type": "module"`) that needs to use this logger. Since `logger.cjs` uses `module.exports`, import it correctly from ESM (research and use the correct import syntax for a CJS module with a single `module.exports` object — no named exports available).
3. In `paymentService.mjs`, write an async function `processPayment(userId, amount)` that logs an info message before "processing", simulates an async delay, then logs a success or error message depending on whether `amount` is positive.
4. Add a `.cjs` file `legacyReport.cjs` that needs to call the ESM `processPayment` function — since CJS cannot statically `import` an ESM file, use dynamic `import()` (which returns a Promise) inside an async IIFE to load `paymentService.mjs` and call `processPayment`.
5. Run and verify all three files interoperate correctly.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
FILE TYPE RULES (priority order)
  .mjs                                  → ALWAYS ES Module
  .cjs                                  → ALWAYS CommonJS
  .js + package.json "type":"module"    → ES Module
  .js + package.json "type":"commonjs"  → CommonJS
  .js + no "type" field at all          → CommonJS (default)

EXPORT SYNTAX
  export const x = 1;              → named export
  export function fn() {}          → named export
  export default function() {}     → default export (one per file)

IMPORT SYNTAX
  import { x } from './file.mjs';       → named import, name must match
  import x from './file.mjs';           → default import, any local name
  import * as ns from './file.mjs';     → namespace import (all as object)
  import './file.mjs';                  → side-effect only import (no bindings)
  const mod = await import('./file.mjs'); → dynamic import, returns a Promise

KEY DIFFERENCES FROM COMMONJS
  CJS: require() — synchronous, can be conditional/anywhere in file
  ESM: import — hoisted, static, must be top-level (except dynamic import())
  CJS: module.exports / exports        ESM: export / export default
  CJS: __dirname, __filename built in  ESM: must derive from import.meta.url
  CJS: extension-less require ok       ESM: relative imports need extension
  CJS: require() is cached, sync       ESM: modules load async, also cached

INTEROP
  ESM importing CJS   → import cjsModule from './legacy.cjs';  (default only)
  CJS importing ESM   → const mod = await import('./modern.mjs'); (async only)
  CJS CANNOT require() an ESM file synchronously — throws ERR_REQUIRE_ESM

TOP-LEVEL AWAIT
  Only works in ESM (.mjs or "type":"module")
  Lets a module pause its own evaluation until a Promise resolves
  Common for DB connections, config loading before server starts

WHEN TO USE WHICH
  Use CJS   → most existing Node backend codebases, broad ecosystem compat
  Use ESM   → new projects, needing top-level await, ESM-only dependencies,
              sharing code with frontend/browser bundles, using native TS ESM output
```

---

## Connected topics

- **06 — \_\_dirname and \_\_filename** — ESM has no `__dirname`/`__filename`; you must reconstruct them from `import.meta.url`, covered in full there.
- **08 — CommonJS modules** — the module system ESM is most often compared against; understanding `require()`/`module.exports` makes the differences in this doc concrete.
- **10 — package.json in depth** — the `"type": "module"` field is set here, and it is what ultimately decides whether your `.js` files are parsed as CJS or ESM.
