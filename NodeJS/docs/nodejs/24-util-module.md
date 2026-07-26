# 24 — util module

## What is this?

`util` is a built-in Node.js module full of small helper functions that don't belong anywhere else — formatting strings, inspecting objects for debugging, checking types at runtime, and most importantly, converting old-style callback functions into modern promise-based ones. Think of it as the adapter drawer in a workshop: it doesn't build anything on its own, but it has exactly the right tool to make an old plug (a callback API) fit into a new socket (`async`/`await`).

## Why does it matter for backend development?

Node's core library and thousands of npm packages were written before promises were common, so they expose callback-style functions like `fn(args, (err, result) => {})`. A backend dev writing modern `async`/`await` code cannot mix callback APIs in cleanly — `util.promisify()` bridges that gap in one line, and you will reach for it constantly when working with legacy drivers, native modules, or older SDKs. `util.format()` gives you consistent, printf-style log lines instead of messy string concatenation. `util.inspect()` is what `console.log()` uses internally, and calling it yourself lets you control exactly how deep and how detailed your debug output is — essential when logging request bodies, database rows, or error objects that contain circular references. `util.types` lets you check runtime types (is this a `Promise`? a `Date`? a native `Buffer`?) more reliably than `typeof` or `instanceof`, which matters in validation layers and generic utility code.

---

## Syntax / API

```js
const util = require('util'); // built-in module — no npm install needed

// ── promisify() — turns an error-first callback function into a Promise-returning one ──
const fs = require('fs');
const readFileAsync = util.promisify(fs.readFile);
// readFileAsync(filePath, 'utf8') now returns a Promise instead of taking a callback

// ── format() — printf-style string formatting (%s string, %d number, %j JSON, %o object) ──
const logLine = util.format('User %s logged in with role %s', 'user_482', 'admin');
console.log(logLine);
// → "User user_482 logged in with role admin"

// ── inspect() — converts any value into a readable string, even deeply nested objects ──
const requestBody = { userId: 'user_482', roles: ['admin', 'editor'], meta: { ip: '10.0.0.5' } };
console.log(util.inspect(requestBody, { depth: null, colors: true }));
// → prints the full nested object as a readable string (this is what console.log uses internally)

// ── util.types — runtime type checks, more precise than typeof/instanceof ──
console.log(util.types.isPromise(readFileAsync(__filename))); // true — it's a Promise
console.log(util.types.isDate(new Date()));                    // true — it's a real Date
console.log(util.types.isRegExp(/^[a-z]+$/));                   // true — it's a RegExp

// ── callbackify() — the reverse of promisify: turns an async function back into a callback one ──
async function fetchUserById(userId) {
  return { userId, name: 'Ravi' }; // pretend this hits a database
}
const fetchUserByIdCb = util.callbackify(fetchUserById);
// fetchUserByIdCb(userId, (err, user) => { ... }) — useful when an old API expects a callback
```

---

## How it works — line by line

- `require('util')` loads the module — it ships with Node, nothing to install.
- `util.promisify(fs.readFile)` looks at `fs.readFile`, which follows the **error-first callback convention** (`fn(args..., callback)` where `callback(err, result)` is the last argument), and returns a **new function** that instead returns a `Promise`. Calling that new function with the same arguments (minus the callback) resolves the promise with `result`, or rejects it with `err`.
- `util.format(template, ...args)` scans the template string for placeholders (`%s` for strings, `%d` for numbers, `%j` for JSON, `%o`/`%O` for objects) and substitutes the arguments in order, producing one clean string — no manual `+` concatenation needed.
- `util.inspect(value, options)` walks the given value recursively and builds a human-readable string representation. The `depth` option controls how many levels of nested objects/arrays it will unpack (`null` means unlimited), and `colors` adds ANSI color codes for terminal readability. This is literally the function `console.log()` calls under the hood whenever you pass it a non-string value.
- `util.types.isPromise(value)`, `isDate(value)`, `isRegExp(value)`, etc. each perform an internal, engine-level check on the value's type — more reliable than `typeof` (which only distinguishes primitives) or `instanceof` (which breaks across different realms/contexts, e.g. values from a `vm` sandbox or a different `iframe`).
- `util.callbackify(asyncFn)` does the opposite of `promisify` — it takes an `async` function (or anything returning a promise) and returns a callback-style function, useful when you must hand your modern code to an older API that only accepts callbacks.

---

## Example 1 — basic

```js
// File: src/scripts/read-config-demo.js

const util = require('util');       // load the util module
const fs   = require('fs');         // load the callback-based fs module
const path = require('path');       // for building a safe file path

// Convert fs.readFile (callback style) into a Promise-returning function
const readFileAsync = util.promisify(fs.readFile);

async function main() {
  const configPath = path.join(__dirname, '..', 'config', 'app.json'); // build path safely

  try {
    // Await the promisified function instead of nesting a callback
    const raw = await readFileAsync(configPath, 'utf8'); // reads file as a string
    const config = JSON.parse(raw);                       // parse the JSON text

    // util.format builds a clean log line instead of string concatenation
    console.log(util.format('Loaded config with %d keys from %s', Object.keys(config).length, configPath));

    // util.inspect shows the full nested object, even if console.log would truncate it
    console.log(util.inspect(config, { depth: null, colors: true }));
  } catch (err) {
    // err is a real Error object (e.g. ENOENT if the file is missing)
    console.error(util.format('Failed to load config: %s', err.message));
  }
}

main(); // run the async function
```

---

## Example 2 — real world backend use case

```js
// File: src/services/legacyPaymentGateway.js
// A third-party payment SDK only exposes callback-style methods (common in older libraries).
// This wraps it in promises so the rest of the codebase can use async/await cleanly.

const util = require('util');

// ── Pretend this is an old SDK you imported, e.g. require('legacy-payment-sdk') ──
const legacyPaymentSdk = {
  chargeCard(apiKey, amountInCents, cardToken, callback) {
    // simulates an async network call to a payment provider
    setTimeout(() => {
      if (!cardToken) {
        return callback(new Error('Missing card token')); // error-first callback
      }
      callback(null, { transactionId: 'txn_88213', amountInCents, status: 'succeeded' });
    }, 50);
  },
};

// Convert the callback-based method into a promise-based one
const chargeCardAsync = util.promisify(legacyPaymentSdk.chargeCard);

// ── Service function used by the rest of the app ──
async function processPayment({ apiKey, amountInCents, cardToken, userId }) {
  // util.types guards against passing the wrong shape of data before hitting the network
  if (!util.types.isBigInt64Array && typeof amountInCents !== 'number') {
    throw new TypeError('amountInCents must be a number');
  }

  try {
    const result = await chargeCardAsync(apiKey, amountInCents, cardToken); // await instead of callback

    // util.format for consistent, greppable log lines
    console.log(util.format('[payment] user=%s amount=%d status=%s txn=%s',
      userId, amountInCents, result.status, result.transactionId));

    return result;
  } catch (err) {
    // util.inspect gives full detail on the error object, including nested properties
    console.error(util.format('[payment] FAILED user=%s: %s', userId, util.inspect(err, { depth: 2 })));
    throw err; // re-throw so the calling route/controller can respond with an error status
  }
}

module.exports = { processPayment };

// Usage in a controller:
// const { processPayment } = require('./legacyPaymentGateway');
// await processPayment({ apiKey, amountInCents: 2500, cardToken: 'tok_abc', userId: 'user_482' });
```

---

## Common mistakes

### Mistake 1 — Promisifying a function that doesn't follow the error-first callback convention

```js
// ❌ WRONG — this function's callback signature is (result, err), reversed from the Node convention
function legacyLookup(userId, callback) {
  callback({ userId, name: 'Ravi' }, null); // result first, error second — NOT standard
}
const lookupAsync = util.promisify(legacyLookup);
const user = await lookupAsync('user_1');
// util.promisify ASSUMES (err, result) order — this silently produces wrong/broken behavior

// ✅ CORRECT — only promisify functions using true error-first callbacks: (err, result)
function properLookup(userId, callback) {
  callback(null, { userId, name: 'Ravi' }); // error first (null = no error), then result
}
const properLookupAsync = util.promisify(properLookup);
const properUser = await properLookupAsync('user_1'); // works correctly
```

### Mistake 2 — Using string concatenation instead of util.format for log messages

```js
// ❌ WRONG — manual concatenation is error-prone and inconsistent across the codebase
console.log('User ' + userId + ' failed login attempt #' + attemptCount + ' from IP ' + ipAddress);
// Easy to typo, hard to keep consistent formatting, breaks if a value is undefined/null

// ✅ CORRECT — util.format enforces a consistent template with type-aware placeholders
const util = require('util');
console.log(util.format('User %s failed login attempt #%d from IP %s', userId, attemptCount, ipAddress));
// Same info, one consistent pattern used everywhere in the codebase — easier to grep in logs
```

### Mistake 3 — Using JSON.stringify to debug-log objects that may have circular references

```js
// ❌ WRONG — JSON.stringify throws on circular references (common with req/res, DB connections, streams)
const dbConnection = { name: 'primary' };
dbConnection.self = dbConnection; // circular reference
console.log(JSON.stringify(dbConnection));
// TypeError: Converting circular structure to JSON — crashes the log line, sometimes the whole request

// ✅ CORRECT — util.inspect handles circular references gracefully
const util = require('util');
console.log(util.inspect(dbConnection, { depth: 2 }));
// → "<ref *1> { name: 'primary', self: [Circular *1] }" — safe, readable, never throws
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Uses `util.promisify` to convert `fs.readFile` into a promise-based function.
2. Reads any existing `.json` file on your machine (or create a small one first) using `await`.
3. Uses `util.format` to log a message like `"Loaded {n} bytes from {path}"` using `%d` and `%s` placeholders.
4. Uses `util.inspect` to pretty-print the parsed JSON content with `depth: null`.

```js
// Write your code here
```

---

### Exercise 2 — medium

You are given this legacy callback-style function that simulates checking a user's session in a database:

```js
function findSessionById(sessionId, callback) {
  setTimeout(() => {
    if (sessionId === 'invalid') return callback(new Error('Session not found'));
    callback(null, { sessionId, userId: 'user_209', expiresAt: Date.now() + 60000 });
  }, 30);
}
```

Write code that:
1. Uses `util.promisify` to convert `findSessionById` into a promise-based function.
2. Writes an `async function getActiveSession(sessionId)` that calls the promisified function inside a `try/catch`.
3. On success, logs the session details using `util.format`.
4. On failure, logs the error using `util.format` and `util.inspect`, then re-throws the error.
5. Calls `getActiveSession('sess_1')` and `getActiveSession('invalid')` and observes both outcomes.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small `RequestLogger` module that:
1. Exports a function `logRequest(requestBody, meta)` where `meta` contains `{ userId, method, path, durationMs }`.
2. Uses `util.types` to validate: `meta.durationMs` must be a number (use `typeof` or `util.types` as appropriate), and `requestBody` must not be `null`/`undefined`.
3. If `requestBody` contains a circular reference (build a test object that references itself), safely logs it anyway using `util.inspect` instead of crashing.
4. Uses `util.format` to produce one single-line summary log: `method`, `path`, `userId`, `durationMs`.
5. Follows the single-line summary with a full `util.inspect` dump of `requestBody` (`depth: 3`, `colors: true`) on a separate line, but only if `durationMs > 200` (i.e. only log full detail for slow requests).
6. Also exports a `wrapLegacyLogger(legacyLogFn)` function that uses `util.promisify` to convert an error-first callback logging function (you write a fake one) into a promise-based one, so it can be `await`-ed alongside the rest of the request pipeline.

Test it with a fast request (should print only the summary line) and a slow request with a circular `requestBody` (should print both lines without crashing).

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
LOADING
  const util = require('util');   // built-in, no install needed

promisify(fn)
  - Converts an error-first callback fn(args..., cb) into a Promise-returning fn
  - REQUIRES the callback to be (err, result) — last argument, error first
  - const readFileAsync = util.promisify(fs.readFile);
  - const data = await readFileAsync(path, 'utf8');
  - Custom promisify behavior: fn[util.promisify.custom] = customImpl

callbackify(asyncFn)
  - The reverse of promisify — async fn → callback-style fn
  - Useful when an old API demands a callback but your logic is async

format(template, ...args)
  - Printf-style string building for consistent log lines
  - %s string | %d number | %i integer | %f float | %j JSON | %o/%O object | %% literal %
  - util.format('User %s did %s', userId, action)

inspect(value, options)
  - Turns any value (including circular refs) into a readable string
  - This is what console.log() uses internally for non-string values
  - options: { depth: null (unlimited) | number, colors: true, showHidden: false }
  - Handles circular references safely: [Circular *1] instead of crashing

util.types.isX(value)
  - More reliable than typeof/instanceof for runtime checks
  - isPromise, isDate, isRegExp, isMap, isSet, isAsyncFunction, isNativeError, etc.
  - Works correctly even across different realms (vm sandboxes, workers)

GOTCHAS
  - promisify assumes (err, result) callback shape — reversed-order callbacks break silently
  - Functions with MULTIPLE success values in the callback (err, a, b) only resolve
    the promise with the FIRST value (a) by default — check docs for exceptions
  - JSON.stringify throws on circular objects — util.inspect does not
  - console.log(obj) already calls util.inspect internally — calling it yourself
    is for when you need custom depth/color/formatting control

COMMON PATTERN — promisifying a legacy SDK once, reusing everywhere
  const chargeCardAsync = util.promisify(legacySdk.chargeCard);
  module.exports = { chargeCardAsync };
```

---

## Connected topics

- **45 — Callback pattern deep dive** — understanding error-first callbacks in depth is required before `util.promisify` makes sense, since it only works on that exact convention.
- **46 — Promises in Node context** — `promisify` produces standard Promises, so `Promise.all`/`Promise.allSettled` patterns apply directly to promisified functions.
- **63 — Logging in Node** — `util.format` and `util.inspect` are the foundation of readable log lines before you graduate to structured loggers like winston or pino.
