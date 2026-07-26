# 08 — CommonJS modules

## What is this?

CommonJS is the original module system in Node.js — the mechanism that lets you split code across multiple files and share values between them using `require()` and `module.exports`. Every `.js` file in Node (unless marked otherwise) is treated as its own private module: variables and functions declared in it are invisible to other files unless explicitly exported. Think of each file as a sealed room — nothing leaks out unless you hand it through the door (`module.exports`), and other rooms can only get it by asking for it by name (`require()`).

---

## Why does it matter for backend development?

A real backend is never one giant file — it is dozens or hundreds of files: routes, controllers, models, middleware, config, utilities. CommonJS is how they all talk to each other. Understanding `require()` caching means you know why a database connection created in one file is reused everywhere instead of reconnecting. Understanding path resolution means you know why `require('./db')` works but `require('db')` looks somewhere entirely different. And understanding circular dependencies means you can diagnose the "why is this export `undefined`" bug that trips up almost every Node developer at least once. This is the module system powering the majority of existing Node.js codebases, npm packages, and legacy Express apps you will work on.

---

## Syntax / API

```js
// File: src/utils/logger.js

// module.exports is the object THIS file hands out to whoever requires it.
// Anything not attached to it stays private to this file.
function logInfo(message) {
  // Simple timestamped console logger
  console.log(`[INFO] ${new Date().toISOString()} - ${message}`);
}

function logError(message) {
  console.error(`[ERROR] ${new Date().toISOString()} - ${message}`);
}

// Replace the entire exports object with our own object of functions
module.exports = { logInfo, logError };

// ── An alternative way to export: attach properties to `exports` ───────────
// `exports` starts out as a SHORTCUT that points to the same object as module.exports
exports.logInfo  = logInfo;   // works, because exports === module.exports (initially)
exports.logError = logError;  // adds a second property the same way

// ⚠️ exports = { logInfo, logError };  → this BREAKS the link (explained below)

// ── Consuming the module from another file ──────────────────────────────────
// File: src/server.js
const path        = require('path');               // core module — resolved by name
const { logInfo } = require('./utils/logger');      // local file — resolved by relative path
const express     = require('express');             // node_modules package — resolved by name lookup

logInfo('Server starting up...');                    // uses the exported function
```

---

## How it works — line by line

- `require('./utils/logger')` tells Node: "go find this file, run it once, and give me back whatever it put in `module.exports`."
- Node figures out **what kind of path** you gave it: a string starting with `./` or `../` means "a file relative to me"; a bare name like `'express'` or `'path'` means "look in `node_modules`, or check if it's a built-in core module first."
- Once Node locates the exact file, it wraps the file's entire code in a hidden function before running it — the **module wrapper**:
  ```js
  (function (exports, require, module, __filename, __dirname) {
    // your file's code lives here
  });
  ```
  This is why `require`, `module`, `exports`, `__filename`, and `__dirname` are available in every CommonJS file without importing them — they are just parameters Node passes in.
- `module` is an object that represents **this specific file**. `module.exports` is the property Node reads after your file finishes running — whatever value it holds at the end is what gets handed back to anyone who `require()`s this file.
- `exports` is a plain variable that Node sets to `module.exports` at the very start, purely as a shorthand so you can write `exports.foo = ...` instead of `module.exports.foo = ...`. They point to the same object at first — but if you reassign `exports` to something new (`exports = {...}`), it stops pointing at `module.exports`, and Node ignores it because it still reads `module.exports` at the end.
- The first time a file is required, Node executes it top to bottom and caches the resulting `module.exports` value in memory, keyed by the resolved absolute file path.
- Every subsequent `require()` of that same file — from anywhere in the app — returns the cached value instantly without re-running the file. This is why a database connection opened in `db.js` is shared, not recreated, across every file that requires it.

---

## Example 1 — basic

```js
// File: src/utils/currency.js

// A small module with two independent exported functions.
const TAX_RATE = 0.18;   // private constant — NOT exported, invisible outside this file

// Converts a raw amount (in paise) to a display string in rupees
function formatAmount(amountInPaise) {
  const rupees = amountInPaise / 100;               // convert paise → rupees
  return `₹${rupees.toFixed(2)}`;                    // format with 2 decimal places
}

// Adds tax on top of a base amount
function addTax(baseAmount) {
  return baseAmount + baseAmount * TAX_RATE;          // apply the private TAX_RATE
}

// Export both functions as named properties on module.exports
module.exports = { formatAmount, addTax };


// File: src/index.js

// Destructure exactly the two functions we need from the module
const { formatAmount, addTax } = require('./utils/currency');

const basePrice   = 50000;                  // price in paise, before tax
const finalPrice  = addTax(basePrice);      // apply tax using the imported function
console.log(formatAmount(finalPrice));      // → "₹590.00"

// console.log(TAX_RATE); // ReferenceError — TAX_RATE was never exported, stays private
```

---

## Example 2 — real world backend use case

```js
// File: src/db/connection.js
// A shared database connection module — the pattern every Node backend uses.
// Because require() CACHES this file, every part of the app gets the SAME connection.

const { Pool } = require('pg');   // 3rd-party package — resolved from node_modules

// This runs ONLY ONCE, no matter how many files require this module
const dbConnection = new Pool({
  host: process.env.DB_HOST,               // covered fully in Topic 12 — env variables
  port: process.env.DB_PORT,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  max: 10,                                  // max simultaneous connections in the pool
});

console.log('[db] Connection pool created');   // proves this file ran exactly once

// Export the single shared pool instance — not a factory, the instance itself
module.exports = dbConnection;


// File: src/repositories/userRepository.js
const dbConnection = require('../db/connection');   // gets the SAME cached pool instance

async function findUserById(userId) {
  // Parameterized query — prevents SQL injection (Topic 76)
  const result = await dbConnection.query(
    'SELECT id, email, name FROM users WHERE id = $1',
    [userId]
  );
  return result.rows[0] || null;    // null if no matching user
}

module.exports = { findUserById };


// File: src/services/authService.js
const dbConnection = require('../db/connection');   // ALSO the same cached pool — no reconnect

async function isSessionValid(sessionId) {
  const result = await dbConnection.query(
    'SELECT expires_at FROM sessions WHERE id = $1',
    [sessionId]
  );
  const session = result.rows[0];
  return Boolean(session) && new Date(session.expires_at) > new Date();
}

module.exports = { isSessionValid };

// Result: "[db] Connection pool created" prints only ONCE, even though both
// userRepository.js and authService.js require connection.js — proving the cache works.
```

---

## Common mistakes

### Mistake 1 — Reassigning `exports` instead of `module.exports`

```js
// ❌ WRONG — reassigning `exports` breaks its link to module.exports
// Node still returns module.exports (an empty {}) to whoever requires this file
exports = {
  generateAuthToken() { return 'token123'; },
};
// require('./this-file') → {} — your function is silently missing!

// ✅ CORRECT — either mutate the existing exports object...
exports.generateAuthToken = function () { return 'token123'; };

// ✅ ...or reassign module.exports directly, never plain `exports`
module.exports = {
  generateAuthToken() { return 'token123'; },
};
```

### Mistake 2 — Forgetting the cache means shared, mutable state

```js
// ❌ WRONG — treating an exported object as if a fresh copy is made each time
// File: config.js
module.exports = { requestCount: 0 };

// File: middlewareA.js
const config = require('./config');
config.requestCount++;   // mutates the ONE cached object

// File: middlewareB.js
const config = require('./config');
console.log(config.requestCount);   // → 1, not 0 — surprised? It's the SAME cached object

// ✅ CORRECT — if you need a fresh instance per use, export a FACTORY function
// File: config.js
module.exports = function createConfig() {
  return { requestCount: 0 };   // new object every call
};

// File: middlewareA.js
const createConfig = require('./config');
const config = createConfig();   // this file's own independent instance
```

### Mistake 3 — Circular requires returning an incomplete/partial export

```js
// ❌ WRONG — two files that require each other at the top
// File: userService.js
const orderService = require('./orderService');
function getUser(userId) { /* ... */ }
module.exports = { getUser };

// File: orderService.js
const userService = require('./userService');   // userService.js isn't finished loading yet!
console.log(userService);   // → {} — partial, INCOMPLETE export (getUser missing)
function getOrder(orderId) { /* ... */ }
module.exports = { getOrder };
// Loading order.js first would flip which file gets the incomplete object.

// ✅ CORRECT — break the cycle: require INSIDE the function, not at the top
// File: orderService.js
function getOrder(orderId) {
  const userService = require('./userService');   // resolved lazily, module.exports is complete by now
  return userService.getUser(orderId.userId);
}
module.exports = { getOrder };
// Even better long-term fix: extract shared logic into a third module both depend on.
```

---

## Practice exercises

### Exercise 1 — easy

Create a file `mathUtils.js` that exports three functions: `add(a, b)`, `subtract(a, b)`, and `multiply(a, b)`. Export them using `module.exports` as an object. Then create a second file `app.js` that requires `mathUtils.js`, destructures all three functions, and logs the results of each one called with two sample numbers.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a module `sessionStore.js` that behaves like a simple in-memory cache:
1. Keeps a private object (not exported directly) holding session data, keyed by `sessionId`.
2. Exports a function `createSession(sessionId, userId)` that stores `{ userId, createdAt: Date.now() }` under that `sessionId`.
3. Exports a function `getSession(sessionId)` that returns the stored session or `null` if it doesn't exist.
4. Exports a function `destroySession(sessionId)` that removes it.
5. Exports a function `countActiveSessions()` that returns how many sessions currently exist.

Then, from a separate file, require the module twice under two different variable names and prove that both requires return the exact same cached object (sessions created via one reference should be visible through the other).

```js
// Write your code here
```

---

### Exercise 3 — hard

Reproduce and then fix a circular dependency bug:
1. Create `moduleA.js` that requires `moduleB.js` at the top of the file, exports a function `describeA()` that calls a function from `moduleB`, and logs something on load.
2. Create `moduleB.js` that requires `moduleA.js` at the top of the file, exports a function `describeB()` that calls a function from `moduleA`, and logs something on load.
3. Create `main.js` that requires `moduleA.js` first, then calls `describeA()`, and observe/print which export came back incomplete (`undefined` function call, or an error).
4. Fix the circular dependency using ONE of these two strategies (your choice): moving the `require()` calls inside the functions that need them (lazy require), or extracting the shared function both files need into a third `sharedUtils.js` module that neither A nor B depends on circularly.
5. Prove the fix works by running `main.js` again and confirming `describeA()` now completes without error.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE PIECES
  require(path)      → loads a module and returns its module.exports value
  module.exports      → the object THIS file exports (source of truth)
  exports              → shortcut pointing to module.exports (initially, same object)

PATH RESOLUTION RULES
  './file'  or '../file'  → relative to the CURRENT file (not cwd) — always a file path
  'express' / 'fs'         → bare name → check core modules first, then node_modules
  No extension given       → Node tries .js, .json, .node in that order
  Path is a folder         → Node looks for package.json "main", else index.js

CACHING
  First require()  → file executes top to bottom, result is cached by absolute path
  Every require() after → returns the SAME cached object instantly, file does NOT re-run
  Cache key = resolved absolute file path (not the string you typed)
  Exported OBJECTS are shared and mutable — mutating them in one file affects all requirers

EXPORTS — SAFE PATTERNS
  module.exports = { fnOne, fnTwo };     ✅ full replace, always safe
  exports.fnOne = fnOne;                  ✅ safe — mutates the existing shared object
  exports = { fnOne };                    ❌ BREAKS the link — Node still returns old module.exports

CIRCULAR DEPENDENCIES
  A requires B, B requires A at the TOP of each file → one of them gets an INCOMPLETE object
  Fix 1: require() lazily INSIDE the function body, not at the top
  Fix 2: extract shared logic into a third module both depend on (no cycle)
  Fix 3: restructure so dependencies form a one-way chain, not a loop

MENTAL MODEL
  Every CJS file is wrapped in:
  (function(exports, require, module, __filename, __dirname) { ...your code... });
  require/module/exports are injected PARAMETERS, not real globals
```

---

## Connected topics

- **09 — ES Modules in Node** — the modern `import`/`export` alternative to CommonJS; no caching quirks around `exports` reassignment, but different circular-dependency behavior
- **13 — Module resolution algorithm** — the exact step-by-step lookup Node performs to turn a `require()` string into a file path, including the `node_modules` search chain
- **06 — \_\_dirname and \_\_filename** — the module wrapper function that injects these two variables is the same wrapper that provides `require`, `module`, and `exports` in every CommonJS file
