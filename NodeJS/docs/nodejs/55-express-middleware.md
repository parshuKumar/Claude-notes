# 55 — Express middleware in depth

## What is this?

Middleware is a function that sits **between** the incoming request and the final response, given a chance to inspect, modify, or reject the request before it reaches your route handler. Express literally builds every request through a **chain of middleware functions**, each one calling `next()` to hand control to the one after it. Think of an airport security line — your bag (the request) passes through metal detector, x-ray scanner, and manual check (each a middleware) before you're allowed to board the plane (your route handler); any checkpoint can stop you right there.

## Why does it matter for backend development?

Almost everything a real backend does — parsing JSON bodies, checking auth tokens, logging requests, validating input, handling CORS, catching errors — is implemented as middleware, not scattered inside every route. This is the core architectural pattern of Express: instead of repeating "check the token" in 40 route handlers, you write it once as middleware and `app.use()` it. Understanding the middleware chain, and crucially the **order in which middleware runs**, is the difference between a backend dev who copies Stack Overflow snippets and one who can debug why `req.body` is `undefined` or why an auth check never fires.

---

## Syntax / API

```js
const express = require('express');
const app = express();

// ── Built-in middleware — ships with Express itself ─────────────────────────
app.use(express.json());               // parses JSON request bodies into req.body
app.use(express.urlencoded({ extended: true })); // parses HTML form bodies into req.body
app.use(express.static('public'));      // serves files from ./public as-is (no route needed)

// ── Third-party middleware — installed from npm ─────────────────────────────
const cors = require('cors');
const morgan = require('morgan');
app.use(cors());                        // adds CORS headers to every response
app.use(morgan('dev'));                 // logs "GET /users 200 12ms" for every request

// ── Custom middleware — a function you write yourself ───────────────────────
function requestLogger(req, res, next) {
  // req  → the incoming request object (method, url, headers, body...)
  // res  → the outgoing response object (you can end the chain here)
  // next → call this to pass control to the NEXT middleware in the chain
  console.log(`${req.method} ${req.url} at ${new Date().toISOString()}`);
  next();                               // MUST call next() or the request hangs forever
}

app.use(requestLogger);                 // applies to every route, in every order it's mounted

// ── Mounting middleware on a specific path ───────────────────────────────────
app.use('/api', requestLogger);         // only runs for URLs starting with /api

// ── Mounting middleware on a specific route ─────────────────────────────────
app.get('/profile', requestLogger, (req, res) => {
  res.json({ userId: 'user_101' });     // requestLogger runs, then this handler runs
});

// ── Error-handling middleware — has FOUR parameters (err first) ─────────────
app.use((err, req, res, next) => {
  console.error(err.stack);             // log the error for debugging
  res.status(500).json({ error: 'Something went wrong' }); // send a safe response
});
```

---

## How it works — line by line

- `app.use(fn)` registers `fn` as middleware that runs for **every** incoming request, regardless of the HTTP method or path.
- Every middleware function receives `(req, res, next)` — three arguments Express passes in automatically. `req` and `res` are the same request/response objects passed all the way down the chain; each middleware can attach new properties to `req` (like `req.userId = 42`) and later middleware or route handlers will see them.
- `next()` is a function Express gives you. Calling it says "I'm done, move to the next middleware in line." If you never call it, and never send a response, the request just hangs until the client times out.
- Middleware runs **in the exact order you register it with `app.use()` or `app.get()/app.post()`**. This is not optional — Express walks the chain top to bottom, so a middleware registered on line 10 always runs before one registered on line 20 for a matching request.
- `app.use('/api', fn)` scopes the middleware to only run when the request URL starts with `/api` — this is called "mounting" the middleware on a path.
- A route handler like `app.get('/profile', requestLogger, handler)` is really just middleware with an extra step: `requestLogger` runs first (because it calls `next()`), then Express moves to `handler`, which usually ends the chain by calling `res.json()` or `res.send()` instead of `next()`.
- Error-handling middleware is special: Express recognizes it by **counting its parameters** — exactly four (`err, req, res, next`) — and only calls it when a previous middleware calls `next(err)` or throws inside an async route wrapped properly. It must be registered **last**, after all other `app.use()` calls, or Express will never reach it for errors thrown earlier in the chain.

---

## Example 1 — basic

```js
// File: src/server.js

const express = require('express');
const app = express();

// Middleware 1 — runs for every request, logs the method and URL
app.use((req, res, next) => {
  console.log(`[LOG] ${req.method} ${req.url}`);
  next(); // pass control forward — without this the request never reaches the route
});

// Middleware 2 — attaches a timestamp to every request object
app.use((req, res, next) => {
  req.requestTime = new Date().toISOString(); // stash data on req for later middleware/routes to use
  next(); // move on to the next middleware or route handler
});

// Route handler — the final stop in the chain, sends the response
app.get('/', (req, res) => {
  // req.requestTime was set two middlewares ago — it's still on the same req object
  res.send(`Hello! Request received at ${req.requestTime}`);
});

app.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});

// Visiting http://localhost:3000 in the terminal prints:
// [LOG] GET /
// Then the browser shows: "Hello! Request received at 2026-07-27T10:15:00.000Z"
```

---

## Example 2 — real world backend use case

```js
// File: src/server.js
// A realistic Express app: parsing bodies, auth checking, per-route middleware,
// and a final error handler — the standard shape of a production API.

const express = require('express');
const app = express();

// ── Global middleware (runs for EVERY request, registered first) ────────────
app.use(express.json());                // parse JSON bodies → populates req.body

app.use((req, res, next) => {
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.originalUrl}`);
  next();                               // request logger — always runs, always calls next
});

// ── Custom auth middleware — reusable, applied only where needed ────────────
function requireAuth(req, res, next) {
  const authHeader = req.headers['authorization']; // e.g. "Bearer abc123token"
  const authToken = authHeader && authHeader.split(' ')[1];

  if (!authToken) {
    // No token at all — stop the chain here, never reach the route handler
    return res.status(401).json({ error: 'Missing auth token' });
  }

  // In a real app you'd verify with jwt.verify() — Topic 57 covers this in depth
  if (authToken !== 'valid-demo-token') {
    return res.status(403).json({ error: 'Invalid auth token' });
  }

  req.userId = 'user_101';              // attach the authenticated user id for downstream use
  next();                               // token is valid — continue to the route handler
}

// ── Public route — no auth middleware, reachable by anyone ──────────────────
app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

// ── Protected route — requireAuth runs BEFORE the handler ───────────────────
app.get('/api/profile', requireAuth, (req, res) => {
  // req.userId was set by requireAuth — safe to use here
  res.json({ userId: req.userId, message: 'Welcome to your profile' });
});

// ── Protected route that also needs the parsed body ──────────────────────────
app.post('/api/orders', requireAuth, (req, res) => {
  const orderPayload = req.body;        // parsed by express.json() earlier in the chain
  console.log(`Order from ${req.userId}:`, orderPayload);
  res.status(201).json({ message: 'Order created', userId: req.userId });
});

// ── 404 handler — catches any request that matched no route above ──────────
app.use((req, res) => {
  res.status(404).json({ error: `No route for ${req.method} ${req.originalUrl}` });
});

// ── Error-handling middleware — MUST be last, has 4 params ──────────────────
app.use((err, req, res, next) => {
  console.error('[ERROR]', err.stack);
  res.status(500).json({ error: 'Internal server error' });
});

app.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

---

## Common mistakes

### Mistake 1 — Forgetting to call next()

```js
// ❌ WRONG — middleware never calls next(), request hangs forever, client times out
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  // no next() call — Express has no idea what to do next, response never sent
});

app.get('/orders', (req, res) => {
  res.json({ orders: [] });   // this handler is NEVER reached
});

// ✅ CORRECT — always call next() when the middleware doesn't itself send a response
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();                     // hands control to the next middleware/route handler
});

app.get('/orders', (req, res) => {
  res.json({ orders: [] });   // now this runs correctly
});
```

### Mistake 2 — Registering middleware in the wrong order

```js
// ❌ WRONG — route reads req.body BEFORE express.json() has parsed it
const express = require('express');
const app = express();

app.post('/api/login', (req, res) => {
  console.log(req.body);      // undefined! JSON parsing middleware hasn't run yet
  res.json({ received: req.body });
});

app.use(express.json());      // registered AFTER the route — too late, never applies to it

// ✅ CORRECT — body-parsing middleware must be registered BEFORE the routes that need it
const app2 = express();

app2.use(express.json());     // parses req.body FIRST, for every route below this line

app2.post('/api/login', (req, res) => {
  console.log(req.body);      // now correctly populated: { email: '...', password: '...' }
  res.json({ received: req.body });
});
```

### Mistake 3 — Putting the error handler before other middleware/routes

```js
// ❌ WRONG — error handler registered first, so Express never routes errors to it;
// any error thrown later in the chain crashes the process or hangs the request
const express = require('express');
const app = express();

app.use((err, req, res, next) => {          // registered too early
  res.status(500).json({ error: 'Something broke' });
});

app.get('/api/orders', (req, res) => {
  throw new Error('DB connection failed');  // this error has nowhere to go
});

// ✅ CORRECT — error-handling middleware must be the LAST app.use() call,
// after every route and every other middleware
const app2 = express();

app2.get('/api/orders', (req, res, next) => {
  try {
    throw new Error('DB connection failed');
  } catch (err) {
    next(err);                              // pass the error into the chain explicitly
  }
});

app2.use((err, req, res, next) => {          // registered LAST — now Express finds it
  console.error(err.message);
  res.status(500).json({ error: 'Something broke' });
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a small Express app with three pieces of custom middleware, all applied globally with `app.use()`:
1. One that logs the HTTP method and URL of every request to the console.
2. One that attaches `req.startTime = Date.now()` so later code can measure how long a request took.
3. One route, `GET /ping`, that responds with `{ elapsedMs: <number> }` computed from `Date.now() - req.startTime`.

Verify the middleware order matters by moving the logging middleware after the route and observing that it no longer logs anything for `/ping`.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an Express app with:
1. A custom `requireApiKey` middleware that reads an `x-api-key` header (`req.headers['x-api-key']`) and rejects the request with `401` and a JSON error if it doesn't match a hardcoded value like `'secret-key-123'`.
2. Two routes: `GET /api/public/status` (no `requireApiKey`, always works) and `GET /api/private/data` (protected by `requireApiKey`).
3. A `404` handler for any unmatched route, returning JSON like `{ error: 'Not found' }`.
4. An error-handling middleware (4 params) registered last, that catches anything thrown in a route and returns `500` with `{ error: 'Internal server error' }`.

Test all four cases: valid key, missing key, wrong key, and an unmatched route.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small "middleware pipeline" system that mimics how Express itself chains middleware, without using Express at all:

1. Write a class `MiddlewarePipeline` with a `use(fn)` method that stores middleware functions in an array, where each `fn` has the signature `(context, next)`.
2. Write a `run(context)` method that executes the stored middleware **in registration order**, where calling `next()` inside one middleware moves to the next one, and NOT calling `next()` stops the chain right there.
3. Add support for **error propagation**: if a middleware throws, or calls `next(error)`, the pipeline should skip straight to a special error-handling middleware (one registered via a separate `useError(fn)` method) instead of continuing the normal chain.
4. Demonstrate it with three normal middlewares (e.g. logging, attaching a fake `userId` to context, validating something on context) and one error middleware, run against a `context` object like `{ requestId: 'req_1', body: { amount: -50 } }` where the validation middleware throws for a negative amount.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT MIDDLEWARE IS
  A function (req, res, next) => {} that runs BEFORE a route handler
  Can inspect/modify req and res, or stop the request early

TYPES
  Built-in     → express.json(), express.urlencoded(), express.static()
  Third-party  → cors, morgan, helmet, compression (installed from npm)
  Custom       → any (req, res, next) function you write yourself
  Error-handling → (err, req, res, next) — exactly 4 params, registered LAST

REGISTERING MIDDLEWARE
  app.use(fn)                → runs for every request, any method/path
  app.use('/api', fn)        → runs only for URLs starting with /api
  app.get('/x', fn, handler) → runs only for GET /x, before handler
  Multiple in one call: app.get('/x', mw1, mw2, handler)

THE GOLDEN RULE — ORDER MATTERS
  Express runs middleware in the EXACT order you register it
  Body parsers (express.json()) must come BEFORE routes that read req.body
  Auth middleware must come BEFORE the protected route
  404 handler must come AFTER all real routes
  Error handler (4 params) must be registered LAST of all

next() RULES
  Call next()        → passes control to the next middleware/route
  Call next(err)      → skips to the nearest error-handling middleware
  Don't call next()   → chain stops here; you MUST send a response yourself
  Calling next() twice → causes "Cannot set headers after sent" bugs

COMMON PATTERNS
  Logging:      app.use((req,res,next) => { console.log(req.method, req.url); next(); })
  Auth guard:   app.get('/private', requireAuth, handler)
  Attach data:  req.userId = decodedToken.sub;  next();
  Reusable:     module.exports = function requireAuth(req,res,next) {...}

GOTCHAS
  express.json() only parses Content-Type: application/json — form data needs express.urlencoded()
  Error middleware needs all 4 params or Express treats it as regular middleware (ignored for errors)
  Async route errors are NOT auto-caught — wrap in try/catch and call next(err), or use express-async-errors
```

---

## Connected topics

- **53 — Express.js fundamentals** — introduces `app`, `req`/`res`/`next`, and `app.use()` for the first time; this topic goes deep on how the chain actually executes.
- **54 — Express routing in depth** — routes and middleware share the same `(req, res, next)` signature and mounting rules covered here (`express.Router`, path scoping).
- **62 — Centralized error handling** — builds directly on the error-handling middleware pattern shown here into a full `AppError` class and consistent error response format.
