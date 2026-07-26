# 62 — Centralized error handling

## What is this?

Centralized error handling means every error in your Express app — from a database failure, a bad request, or a bug — flows through **one single place** that decides how to log it and what to send back to the client. Instead of writing `try/catch` and custom `res.status(...).json(...)` in every single route, you throw an error and let a dedicated error-handling middleware catch it at the end of the pipeline. Think of it like a hospital's emergency room: no matter what kind of injury walks in (broken bone, burn, allergic reaction), everyone goes through the same intake desk first, which triages the case and routes it correctly — you don't build a separate entrance for every injury type.

## Why does it matter for backend development?

Without centralization, error handling in a real API becomes duplicated, inconsistent, and easy to get wrong — one route returns `{ error: "not found" }`, another returns `{ message: "Not Found" }`, another leaks a raw stack trace to the client in production. A centralized error handler guarantees every error response has the **same shape**, so frontend developers and API consumers can rely on one consistent contract (`status`, `message`, `code`). It also gives you one place to log every error, hide sensitive details in production, and distinguish "expected" errors (invalid input, missing resource) from real bugs. Every production-grade Express API — Stripe, GitHub, your own company's backend — uses this pattern.

---

## Syntax / API

```js
// Express recognizes an error-handling middleware by its FOUR parameters: (err, req, res, next)
// It must be registered LAST, after all routes and other middleware

app.use((err, req, res, next) => {
  // Pull a status code off the error, default to 500 (server error) if none set
  const statusCode = err.statusCode || 500;

  // Send a consistent JSON shape to every client, for every kind of error
  res.status(statusCode).json({
    success: false,               // always false when this handler runs
    message: err.message,         // human-readable message
    code: err.code || 'INTERNAL_ERROR', // machine-readable error code for the frontend
  });
});

// Inside any route, calling next(err) hands the error to the handler above
app.get('/users/:userId', (req, res, next) => {
  const user = findUser(req.params.userId); // pretend lookup
  if (!user) {
    // next(err) skips all remaining route handlers and jumps straight to the error middleware
    return next(new Error('User not found'));
  }
  res.json(user);
});
```

---

## How it works — line by line

- Express treats any middleware function with **exactly four parameters** — `(err, req, res, next)` — as an error handler. This is not a convention, it is how Express literally detects it; three-parameter functions are treated as normal middleware.
- When any route or middleware calls `next(err)` (passing something into `next`), Express stops running normal middleware and jumps directly to the nearest error-handling middleware, skipping everything else in between.
- Because it is registered **last** (after `app.use(routes)`), it acts as a final safety net — no matter which route or controller the error came from, it always lands here.
- Inside the handler, you read a `statusCode` off the error object (if your custom error class sets one) and fall back to `500` for anything unexpected — because an unhandled bug should never leak as a `200` or silently crash the process.
- The response body always has the same three fields (`success`, `message`, `code`) so every API consumer parses errors the exact same way, whether it was a validation failure or a database timeout.
- Async route handlers need special care: Express 4 does **not** automatically catch a rejected promise thrown inside an `async` function — you must call `next(err)` yourself (or wrap the route, shown in Example 2) so the error reaches this handler instead of crashing the process with an unhandled rejection.

---

## Example 1 — basic

```js
// File: src/server.js

const express = require('express');
const app = express();

app.use(express.json()); // parse incoming JSON request bodies

// A normal route that might fail
app.get('/products/:productId', (req, res, next) => {
  const productId = req.params.productId; // e.g. "prod_42"

  // Simulate "not found" — a real app would check a database here
  if (productId !== 'prod_42') {
    const notFoundError = new Error(`Product ${productId} not found`); // build a plain Error
    notFoundError.statusCode = 404;      // attach an HTTP status to the error object
    notFoundError.code = 'PRODUCT_NOT_FOUND'; // attach a machine-readable code
    return next(notFoundError);          // hand it off to the error middleware below
  }

  res.json({ productId, name: 'Wireless Mouse', price: 799 }); // success path
});

// Catch-all for routes that don't exist at all
app.use((req, res, next) => {
  const err = new Error(`Route ${req.originalUrl} not found`); // build a 404 error
  err.statusCode = 404;
  err.code = 'ROUTE_NOT_FOUND';
  next(err); // forward to the centralized handler
});

// The ONE centralized error handler — must have 4 params and be registered LAST
app.use((err, req, res, next) => {
  console.error(err.stack); // always log the full error for debugging (server-side only)

  res.status(err.statusCode || 500).json({
    success: false,
    message: err.message,
    code: err.code || 'INTERNAL_ERROR',
  });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## Example 2 — real world backend use case

```js
// File: src/utils/AppError.js
// A custom error class so every intentional error carries structured data

class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);              // pass the message up to the native Error class
    this.statusCode = statusCode; // HTTP status to send (400, 401, 404, 409, etc.)
    this.code = code;             // machine-readable code, e.g. 'INVALID_CREDENTIALS'
    this.isOperational = true;    // marks this as a KNOWN, expected error (see Topic 61)
    Error.captureStackTrace(this, this.constructor); // keeps stack trace clean, hides this constructor frame
  }
}

module.exports = AppError;
```

```js
// File: src/utils/catchAsync.js
// Wraps async route handlers so rejected promises are forwarded to next() automatically

const catchAsync = (fn) => {
  // returns a new function Express will call as the route handler
  return (req, res, next) => {
    // fn(...).catch(next) forwards any thrown/rejected error straight to next(err)
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};

module.exports = catchAsync;
```

```js
// File: src/routes/authRoutes.js

const express = require('express');
const router = express.Router();
const AppError = require('../utils/AppError');
const catchAsync = require('../utils/catchAsync');
const { findUserByEmail, comparePassword } = require('../services/authService');

// catchAsync wraps this handler so we never need a manual try/catch here
router.post('/login', catchAsync(async (req, res, next) => {
  const { email, password } = req.body; // pull credentials out of the request body

  const user = await findUserByEmail(email); // await a DB lookup

  if (!user) {
    // Throwing an AppError inside catchAsync's promise is automatically caught and forwarded
    throw new AppError('No account found with that email', 404, 'USER_NOT_FOUND');
  }

  const isValid = await comparePassword(password, user.passwordHash); // check password
  if (!isValid) {
    throw new AppError('Incorrect password', 401, 'INVALID_CREDENTIALS');
  }

  res.json({ success: true, userId: user.id, message: 'Login successful' });
}));

module.exports = router;
```

```js
// File: src/middleware/errorHandler.js
// The single centralized error handler, mounted last in server.js

function errorHandler(err, req, res, next) {
  const statusCode = err.statusCode || 500;  // known errors set this; unknown bugs default to 500
  const isOperational = err.isOperational || false; // was this a "known" AppError?

  // Log full details server-side always — including stack trace for debugging
  console.error(`[${new Date().toISOString()}] ${req.method} ${req.originalUrl} -> ${err.message}`);
  if (!isOperational) console.error(err.stack); // unexpected bugs get their full stack logged

  // Never leak internal details (stack traces, DB errors) to the client in production
  const clientMessage = isOperational
    ? err.message                                // safe to show — it's an expected, crafted message
    : 'Something went wrong. Please try again.'; // generic message for unknown/programmer errors

  res.status(statusCode).json({
    success: false,
    message: clientMessage,
    code: err.code || 'INTERNAL_ERROR',
  });
}

module.exports = errorHandler;
```

```js
// File: src/server.js

const express = require('express');
const authRoutes = require('./routes/authRoutes');
const errorHandler = require('./middleware/errorHandler');

const app = express();
app.use(express.json());
app.use('/auth', authRoutes);

app.use(errorHandler); // registered LAST — the single funnel for every error in the app

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## Common mistakes

### Mistake 1 — Registering the error handler before the routes

```js
// ❌ WRONG — error middleware registered BEFORE routes never gets called for their errors
app.use((err, req, res, next) => {
  res.status(500).json({ message: err.message });
});
app.use('/users', userRoutes); // errors thrown here never reach the handler above

// ✅ CORRECT — error handler must come AFTER all routes/middleware, at the very bottom
app.use('/users', userRoutes);
app.use((err, req, res, next) => {
  res.status(err.statusCode || 500).json({ message: err.message });
});
```

### Mistake 2 — Forgetting to forward async errors with next()

```js
// ❌ WRONG — a rejected promise inside async route handler is NOT caught by Express automatically
// This crashes with an unhandled rejection and the client request hangs forever
app.get('/orders/:orderId', async (req, res) => {
  const order = await findOrder(req.params.orderId); // if this rejects, nothing catches it
  res.json(order);
});

// ✅ CORRECT — either wrap with catchAsync or use a manual try/catch that calls next(err)
app.get('/orders/:orderId', async (req, res, next) => {
  try {
    const order = await findOrder(req.params.orderId);
    res.json(order);
  } catch (err) {
    next(err); // forwards the error to the centralized handler instead of crashing
  }
});
```

### Mistake 3 — Leaking internal error details to the client in production

```js
// ❌ WRONG — sending the raw error object exposes stack traces, file paths, and DB internals
app.use((err, req, res, next) => {
  res.status(500).json(err); // leaks err.stack, connection strings, internal file paths
});

// ✅ CORRECT — send a safe, generic message for unknown errors; only expose known/operational messages
app.use((err, req, res, next) => {
  const isOperational = err.isOperational || false;
  res.status(err.statusCode || 500).json({
    success: false,
    message: isOperational ? err.message : 'Internal server error', // hide details for real bugs
    code: err.code || 'INTERNAL_ERROR',
  });
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a small Express app with one route `GET /items/:itemId` that only recognizes `itemId === 'item_1'` as valid. For any other id, create a plain `Error`, attach a `.statusCode = 404` and `.code = 'ITEM_NOT_FOUND'` to it, and call `next(err)`. Add a centralized error-handling middleware (4 parameters) registered last that returns a JSON response with `success`, `message`, and `code` fields, using `err.statusCode` (default `500`) as the HTTP status.

```js
// Write your code here
```

---

### Exercise 2 — medium

Create an `AppError` class (extending `Error`) that takes `(message, statusCode, code)` in its constructor and sets `this.isOperational = true`. Build two routes:
- `POST /signup` — throws `new AppError('Email already registered', 409, 'EMAIL_TAKEN')` if `req.body.email === 'taken@example.com'`, otherwise responds with success.
- `GET /reports/:reportId` — throws `new AppError('Report not found', 404, 'REPORT_NOT_FOUND')` unless `reportId === 'rep_1'`.

Write a centralized error handler that checks `err.isOperational`: if true, send `err.message` to the client; if false (or missing), send a generic `"Something went wrong"` message instead, but always log the real error server-side with `console.error`.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small "wallet" API with the following requirements:
1. A `catchAsync(fn)` helper that wraps async route handlers and forwards rejected promises to `next()`.
2. An `AppError` class as described above, plus a static helper `AppError.notFound(resource)` that returns `new AppError(\`${resource} not found\`, 404, 'NOT_FOUND')`.
3. Routes: `POST /wallets/:walletId/withdraw` that reads `amount` from `req.body`, looks up a wallet (fake an in-memory object keyed by id), throws `AppError.notFound('Wallet')` if the wallet doesn't exist, throws a `400 INSUFFICIENT_FUNDS` `AppError` if `amount` exceeds the wallet balance, and otherwise deducts the balance and responds with the new balance.
4. A centralized error handler that logs every error with a timestamp and route, distinguishes operational vs programmer errors (hide details for the latter), and always returns the same `{ success, message, code }` shape.
5. A catch-all 404 middleware (before the error handler) for unmatched routes, using `AppError.notFound('Route')`.

Test it by calling withdraw on a wallet that doesn't exist, one with insufficient funds, and one that succeeds — verify each returns the correct status code and body shape.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
ERROR MIDDLEWARE SIGNATURE
  (err, req, res, next) => { ... }   → 4 params = Express treats it as an error handler
  Must be registered LAST, after all routes/middleware

TRIGGERING THE HANDLER
  next(err)                          → skips remaining routes, jumps to error handler
  throw err  (inside catchAsync)     → caught by .catch(next), forwarded automatically
  throw err  (inside plain async fn) → NOT auto-caught by Express — will crash/hang, wrap it

CUSTOM ERROR CLASS PATTERN
  class AppError extends Error {
    constructor(message, statusCode, code) {
      super(message);
      this.statusCode = statusCode;
      this.code = code;
      this.isOperational = true;   // known/expected error, safe to show message
    }
  }

CONSISTENT RESPONSE SHAPE
  {
    success: false,
    message: "...",     // safe, human-readable
    code: "SOME_CODE"    // machine-readable, frontend can switch on this
  }

RULES
  - Default to statusCode 500 for anything without one (unexpected bugs)
  - Only show err.message to the client if err.isOperational is true
  - Always console.error (or log) the full error server-side, always
  - Never send err directly (res.json(err)) — leaks stack traces and internals
  - 404 "route not found" handler goes BEFORE the error handler, AFTER all real routes

CATCHASYNC WRAPPER
  const catchAsync = (fn) => (req, res, next) =>
    Promise.resolve(fn(req, res, next)).catch(next);
```

---

## Connected topics

- **61 — Operational vs programmer errors** — the classification (`isOperational`) that this centralized handler relies on to decide what to show the client.
- **53 — Express.js fundamentals** — middleware, `app.use()`, and the `(req, res, next)` signature that error middleware builds on.
- **63 — Logging in Node** — replacing the `console.error` calls inside the error handler with structured logging (winston/pino) in production.
