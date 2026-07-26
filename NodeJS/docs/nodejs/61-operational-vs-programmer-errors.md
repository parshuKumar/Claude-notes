# 61 — Operational vs programmer errors

## What is this?

Not every error in a Node.js app deserves the same response. An **operational error** is a known, expected failure that happens during normal running of a correct program — a user submits bad data, a database connection times out, a file doesn't exist. A **programmer error** is a bug — a typo, calling a function with the wrong arguments, forgetting to `await` a promise. Think of it like a restaurant: a customer sending back a dish because it's too spicy is operational (you apologize, remake it, move on) — the chef accidentally cutting his own hand is a programmer error (you stop, treat the wound, figure out how it happened before continuing to cook).

## Why does it matter for backend development?

This distinction decides what your error-handling code should actually do. Operational errors should be **caught and handled** — send the user a 400 response, retry the DB call, log it, keep the server running. Programmer errors mean your app is in an **unknown, possibly corrupted state** — the correct move is to log everything you can, crash the process, and let a process manager (PM2, Kubernetes, systemd) restart it clean. Backend developers who don't make this distinction end up doing one of two dangerous things: crashing the whole server on a routine "invalid email" validation error, or — worse — swallowing a real bug with a generic `try/catch` and letting the server limp along with corrupted state, silently serving wrong data to users.

---

## Syntax / API

```js
// A custom error class that tags whether an error is "operational" (expected) or not (a bug)
class AppError extends Error {
  constructor(message, statusCode, isOperational = true) {
    super(message);                          // set the standard Error.message
    this.statusCode = statusCode;            // HTTP status to send back (400, 404, 503...)
    this.isOperational = isOperational;      // true = expected/known, false = a real bug
    this.name = this.constructor.name;       // "AppError" instead of generic "Error"
    Error.captureStackTrace(this, this.constructor); // clean stack trace, hides constructor frame
  }
}

// ── Creating an operational error (expected failure) ────────────────────────
throw new AppError('User not found', 404);
// isOperational defaults to true — this is a normal, anticipated situation

// ── A programmer error is just a normal bug — usually NOT thrown on purpose ─
// TypeError: Cannot read properties of undefined (reading 'id')
// ReferenceError: userId is not defined
// These come from JS itself when code is wrong, not from your business logic

// ── The classification check used in a central error handler ───────────────
function isTrustedError(error) {
  if (error instanceof AppError) {
    return error.isOperational;              // ask the error itself what kind it is
  }
  return false;                               // unknown error types are treated as bugs
}
```

---

## How it works — line by line

The core idea is a **flag on the error object** that answers one question: "did I, the developer, anticipate this could happen?"

- `AppError` extends the built-in `Error` class so it still works with `instanceof Error`, `try/catch`, and `.stack`.
- `statusCode` lets a central handler map the error straight to an HTTP response (covered fully in Topic 62) without guessing.
- `isOperational` is the classification switch: `true` means "this is a normal part of the business — a missing record, bad input, a third-party API being down." `false` (or any error that *isn't* an `AppError` at all, like a raw `TypeError`) means "something is broken that I didn't plan for."
- `Error.captureStackTrace(this, this.constructor)` removes the `AppError` constructor itself from the stack trace, so the trace points at where the error was actually thrown in your code — makes debugging faster.
- `isTrustedError()` is the decision function: operational errors get handled gracefully (send a response, log, continue). Anything else — a genuine bug — should propagate up to a process-level handler that logs it and exits, because the process may be in a state nobody can trust anymore (a partially updated in-memory cache, a corrupted variable, an unclosed lock).

The rule of thumb: **operational errors are recoverable per-request; programmer errors are not recoverable at all — they mean restart the process.**

---

## Example 1 — basic

```js
// File: src/errors/AppError.js

// Custom error class — the foundation for classifying every error in the app
class AppError extends Error {
  constructor(message, statusCode, isOperational = true) {
    super(message);                            // pass message up to Error
    this.statusCode = statusCode;              // e.g. 400, 404, 503
    this.isOperational = isOperational;        // true = expected, false = a bug
    this.name = this.constructor.name;         // 'AppError'
    Error.captureStackTrace(this, this.constructor); // clean trace
  }
}

// A function that simulates looking up a user — throws an operational error if missing
function findUserById(userId) {
  const users = { user_1: { userId: 'user_1', name: 'Aditi' } }; // fake in-memory "DB"

  const user = users[userId];                  // may or may not exist
  if (!user) {
    // Expected situation — a bad/unknown userId is normal, NOT a bug in our code
    throw new AppError(`User ${userId} not found`, 404);
  }
  return user;
}

// Using it — try/catch distinguishes operational vs programmer errors
try {
  const user = findUserById('user_99');        // this userId does not exist
  console.log(user);
} catch (error) {
  if (error instanceof AppError && error.isOperational) {
    // Known, safe to handle — log it calmly and move on
    console.log(`Handled operational error: ${error.message} (status ${error.statusCode})`);
  } else {
    // Unknown error shape — treat as a bug, rethrow so it isn't silently hidden
    console.error('Unexpected programmer error:', error);
    throw error;
  }
}

module.exports = AppError;
```

---

## Example 2 — real world backend use case

```js
// File: src/middleware/errorHandler.js
// A centralized Express error handler that reacts differently based on error classification.
// (Full centralized-handler pattern is Topic 62 — this shows the classification decision itself.)

const AppError = require('../errors/AppError');

function isOperationalError(error) {
  // Only errors we explicitly created as AppError with isOperational=true are "safe"
  return error instanceof AppError && error.isOperational === true;
}

// Express error-handling middleware — note the 4 arguments, that's what makes it an error handler
function errorHandler(err, req, res, next) {
  if (isOperationalError(err)) {
    // ── OPERATIONAL: expected failure — respond normally, keep serving traffic ──
    console.warn(`[operational] ${req.method} ${req.originalUrl} → ${err.message}`);

    return res.status(err.statusCode).json({
      success: false,
      message: err.message,                     // safe to show the user
    });
  }

  // ── PROGRAMMER ERROR (or truly unknown error): do NOT trust app state ──────
  console.error('[FATAL] Unclassified error — crashing process for a clean restart:', err);

  res.status(500).json({
    success: false,
    message: 'Internal server error',           // never leak stack traces to clients
  });

  // Let the process manager (PM2 / Kubernetes) restart a fresh, known-good process
  // instead of continuing to serve requests from a possibly-corrupted state.
  process.exit(1);
}

// ── Example route that produces BOTH kinds of errors ────────────────────────
function getOrderHandler(req, res, next) {
  const orderId = req.params.orderId;           // e.g. '/orders/ord_501'

  const order = database.findOrder(orderId);    // fake lookup

  if (!order) {
    // Operational — a missing order is an expected, everyday occurrence
    return next(new AppError(`Order ${orderId} not found`, 404));
  }

  // If `order.items` were undefined due to a bug in database.findOrder(),
  // this next line throws a raw TypeError — a PROGRAMMER error, not operational.
  const total = order.items.reduce((sum, item) => sum + item.price, 0);

  res.json({ success: true, order, total });
}

module.exports = { errorHandler, getOrderHandler };
```

---

## Common mistakes

### Mistake 1 — Treating every caught error as safe to recover from

```js
// ❌ WRONG — catches EVERYTHING and just logs, including real bugs, then keeps serving requests
app.get('/orders/:orderId', async (req, res) => {
  try {
    const order = await getOrder(req.params.orderId);
    res.json(order);
  } catch (error) {
    console.log('Something went wrong:', error.message); // swallowed! bug hidden forever
    res.status(500).json({ message: 'Error' });
    // server keeps running with whatever corrupted state caused this
  }
});

// ✅ CORRECT — classify first, only recover from operational errors
app.get('/orders/:orderId', async (req, res, next) => {
  try {
    const order = await getOrder(req.params.orderId);
    res.json(order);
  } catch (error) {
    next(error); // pass to centralized handler, which decides recover vs crash
  }
});
```

### Mistake 2 — Crashing the whole process on routine validation failures

```js
// ❌ WRONG — treating an operational error (bad input) like a fatal bug
app.post('/users', (req, res) => {
  if (!req.body.email) {
    console.error('Missing email!');
    process.exit(1); // takes down the ENTIRE server for one bad request — never do this
  }
});

// ✅ CORRECT — bad input is expected and recoverable; just respond with 400
const AppError = require('../errors/AppError');

app.post('/users', (req, res, next) => {
  if (!req.body.email) {
    return next(new AppError('Email is required', 400)); // operational, isOperational=true
  }
  // ... create user
});
```

### Mistake 3 — Not marking third-party/library errors as unknown by default

```js
// ❌ WRONG — assuming any error thrown by a library is operational without checking
try {
  const data = JSON.parse(requestBody); // could throw SyntaxError on malformed JSON
} catch (error) {
  res.status(400).json({ message: 'Bad request' }); // OK here, but what if it wasn't JSON.parse?
  // Danger: this pattern gets copy-pasted onto code where the error IS a real bug
}

// ✅ CORRECT — wrap known-risky operations in an explicit AppError, don't guess
const AppError = require('../errors/AppError');

function safeParseBody(requestBody) {
  try {
    return JSON.parse(requestBody);
  } catch (parseError) {
    // We KNOW malformed JSON from a client is operational — say so explicitly
    throw new AppError('Invalid JSON in request body', 400, true);
  }
}
// Any other error type reaching the handler is NOT wrapped as AppError,
// so it correctly falls through to the "unknown/programmer error" path.
```

---

## Practice exercises

### Exercise 1 — easy

Create an `AppError` class (extending `Error`) with a constructor that accepts `message`, `statusCode`, and `isOperational` (default `true`). Then write three functions that simulate backend scenarios and throw an `AppError` for each:
1. `validateAge(age)` — throws a 400 `AppError` if `age` is negative or not a number
2. `checkFileExists(filePath)` — throws a 404 `AppError` if a hardcoded list of "existing files" does not include `filePath`
3. `checkApiKey(apiKey)` — throws a 401 `AppError` if `apiKey` does not equal `'secret-key-123'`

Call each function with both a valid and invalid input, using `try/catch` to log whether the error is operational.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a function `classifyError(error)` that returns one of three strings: `'OPERATIONAL'`, `'PROGRAMMER'`, or `'UNKNOWN'`, using these rules:
1. If `error` is an instance of your `AppError` class **and** `isOperational` is `true` → `'OPERATIONAL'`
2. If `error` is an instance of `TypeError` or `ReferenceError` → `'PROGRAMMER'`
3. Anything else (a plain `Error`, a string thrown, etc.) → `'UNKNOWN'`

Then write a small test harness that throws/catches at least 5 different error scenarios (including a deliberate bug like calling `.toUpperCase()` on `undefined`) and prints the classification for each using `classifyError`.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a mini request-handling simulator (no real HTTP server needed) that models a production-style decision pipeline:
1. A function `handleRequest(requestBody, dbConnection)` that:
   - Throws an `AppError('Missing userId', 400)` if `requestBody.userId` is missing (operational)
   - Throws an `AppError('User not found', 404)` if `dbConnection.users` does not contain that `userId` (operational)
   - Deliberately contains ONE line of buggy code that throws a raw `TypeError` when a certain crafted input is passed (e.g. accessing `.profile.settings.theme` on a user object that has no `profile` field) — this represents a programmer error
2. A function `processRequest(requestBody, dbConnection)` that calls `handleRequest` inside a `try/catch`, uses your `classifyError` logic (from Exercise 2) to decide what to do:
   - `'OPERATIONAL'` → return `{ success: false, statusCode: error.statusCode, message: error.message }`
   - anything else → log `'[FATAL] would restart process here'` and return `{ success: false, statusCode: 500, message: 'Internal server error' }` — but must NOT actually call `process.exit()` in this simulation
3. Run `processRequest` with three different inputs: one that succeeds, one that triggers each of the two operational errors, and one crafted specifically to trigger your programmer-error bug. Print all four results.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
OPERATIONAL ERRORS (expected — RECOVER)
  - Invalid user input / failed validation
  - Resource not found (user, file, record)
  - Failed DB connection / query timeout
  - Third-party API down or slow
  - Network request failed
  - Auth token expired / invalid credentials
  → Action: catch it, log it, send a proper HTTP response, KEEP RUNNING

PROGRAMMER ERRORS (bugs — DO NOT RECOVER)
  - TypeError: Cannot read properties of undefined
  - ReferenceError: x is not defined
  - Calling a function with wrong arguments
  - Forgetting to await a promise, causing bad state
  - Memory leaks, infinite loops from logic errors
  → Action: log everything, CRASH the process, let PM2/K8s restart clean

THE CLASSIFICATION FLAG
  class AppError extends Error {
    constructor(message, statusCode, isOperational = true) { ... }
  }
  error instanceof AppError && error.isOperational === true  → safe to handle

WHY CRASH ON PROGRAMMER ERRORS
  A bug means the process state is UNKNOWN and possibly corrupted.
  Continuing to serve requests risks silently returning wrong data.
  A supervised restart (PM2, Docker restart policy, K8s) gives a clean slate.

GOLDEN RULE
  Known and expected  → handle gracefully, respond, move on
  Unknown or a bug     → log loudly, exit process, let it restart

NEVER DO
  process.exit() on validation errors (kills server for routine input)
  Swallowing every error with a generic try/catch + console.log
  Sending raw error.stack or error.message from bugs to the client
```

---

## Connected topics

- **07 — Errors and error handling in Node** — error types, error-first callbacks, `uncaughtException`/`unhandledRejection` — the foundation this topic builds classification rules on top of
- **62 — Centralized error handling** — takes this classification (`isOperational`) and wires it into a single Express error-handling middleware with the `AppError` class
- **64 — Graceful shutdown** — what actually happens when a programmer error triggers a crash: draining connections and exiting cleanly instead of just calling `process.exit()` abruptly
