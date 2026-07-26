# 53 — Express.js fundamentals

## What is this?

Express.js is a minimal, unopinionated web framework built on top of Node's raw `http` module. It gives you a clean, declarative way to define routes, handle requests, and process responses — instead of writing giant `if/else` chains to check `req.url` and `req.method` yourself. Think of Express as a restaurant's ordering system: instead of every waiter shouting orders across the kitchen (raw `http`), Express gives every request a standard ticket that passes through a defined line of stations (middleware) before reaching the right chef (your route handler).

## Why does it matter for backend development?

Almost every Node.js backend job posting mentions Express, because it is still the most widely deployed Node web framework in production, and its middleware pattern is copied by nearly every other framework (Koa, NestJS, Fastify). A backend developer uses Express to build REST APIs, serve static files, plug in authentication, handle errors centrally, and structure large applications into readable, testable pieces. Once you understand `app.use()`, the middleware chain, and `req/res/next`, you can read the source code of almost any Node backend, because this pattern is the backbone of the ecosystem.

---

## Syntax / API

```js
// Import Express — it's the default export of the package
const express = require('express');

// Create an application instance — this is your entire server
const app = express();

// ── Built-in middleware ──────────────────────────────────────────────────────
app.use(express.json());              // parses JSON request bodies into req.body
app.use(express.urlencoded({ extended: true })); // parses form-encoded bodies

// ── Custom middleware — runs for EVERY request, in the order it's registered ──
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`); // log every incoming request
  next();                                  // MUST call next() or the request hangs forever
});

// ── Route handlers — method + path + handler function ────────────────────────
app.get('/users/:userId', (req, res) => {
  const { userId } = req.params;           // read the :userId segment from the URL
  res.json({ userId, name: 'Ada Lovelace' }); // send a JSON response
});

app.post('/users', (req, res) => {
  const requestBody = req.body;            // parsed JSON body (thanks to express.json())
  res.status(201).json({ created: true, ...requestBody }); // 201 = Created
});

// ── Router — a mini, mountable Express app for grouping related routes ───────
const router = express.Router();
router.get('/ping', (req, res) => res.send('pong'));
app.use('/api', router);                   // all router routes are prefixed with /api

// ── Error-handling middleware — MUST have exactly 4 parameters (err, req, res, next) ──
app.use((err, req, res, next) => {
  console.error(err.stack);                // log the error for debugging
  res.status(500).json({ error: 'Something went wrong' }); // never leak err.stack to clients
});

// ── Start listening for connections ───────────────────────────────────────────
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

---

## How it works — line by line

`express()` creates an application object. That object is really a request-handling function, plus a set of methods (`get`, `post`, `use`, `listen`, etc.) attached to it. When Node's HTTP server receives a request, it hands it to this Express app, and Express decides what to do with it.

Every request travels down a **pipeline** of functions, in the exact order you registered them with `app.use()` or a route method like `app.get()`. Each function in that pipeline gets three things: `req` (the incoming request — method, URL, headers, body), `res` (the object you use to send a response back), and `next` (a function you call to hand control to the *next* function in the pipeline).

A "middleware" is simply any function with the shape `(req, res, next)`. It can read or modify `req`, send a response with `res` and stop the chain, or call `next()` to pass control forward. If a middleware forgets to call `next()` and never sends a response, the request just hangs — the client waits forever.

`express.Router()` creates a smaller, self-contained version of an Express app — it has its own `get`/`post`/`use` methods but isn't tied to a port. You mount it onto the main app with `app.use('/prefix', router)`, and every route inside it automatically gets that prefix. This is how large apps split routes across files instead of dumping everything into one `server.js`.

Error-handling middleware is special-cased by Express purely by **counting function parameters** — if a function you pass to `app.use()` has exactly four parameters `(err, req, res, next)`, Express treats it as an error handler and only calls it when an error is passed to `next(err)` or thrown inside an async-unaware handler. It must be registered **last**, after all other routes and middleware, because Express only reaches it when something upstream signals an error.

---

## Example 1 — basic

```js
// File: src/server.js
// A minimal Express server showing app, middleware, and req/res/next together.

const express = require('express');       // load the framework
const app = express();                    // create the application

// Middleware #1 — runs on every request, logs method and path
app.use((req, res, next) => {
  console.log(`Incoming: ${req.method} ${req.originalUrl}`); // log request line
  req.requestTime = Date.now();            // attach custom data onto req for later use
  next();                                  // pass control to the next middleware/route
});

// Middleware #2 — built-in body parser, needed before reading req.body
app.use(express.json());                  // parses "Content-Type: application/json" bodies

// Route — GET /health, a simple readiness check
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok', checkedAt: req.requestTime }); // send JSON response
});

// Route — POST /echo, sends back whatever JSON body was sent
app.post('/echo', (req, res) => {
  res.status(200).json({ youSent: req.body }); // req.body was parsed by express.json()
});

// Fallback for unmatched routes — registered AFTER all real routes
app.use((req, res) => {
  res.status(404).json({ error: `No route for ${req.method} ${req.originalUrl}` }); // 404 handler
});

// Start the server on port 3000
app.listen(3000, () => {
  console.log('Server listening on http://localhost:3000'); // confirms server started
});
```

---

## Example 2 — real world backend use case

```js
// File: src/server.js
// A realistic setup: JSON parsing, a mounted user router, auth middleware,
// and a centralized error handler — the pattern used in real Express backends.

const express = require('express');
const app = express();

app.use(express.json());                  // parse JSON bodies for every route

// ── Custom auth middleware — checks a Bearer token before letting requests through ──
function requireAuth(req, res, next) {
  const authHeader = req.headers.authorization;      // e.g. "Bearer abc123"
  const authToken  = authHeader && authHeader.split(' ')[1]; // extract the token part

  if (!authToken) {
    return res.status(401).json({ error: 'Missing auth token' }); // stop here, no next()
  }

  // In a real app you'd verify a JWT (see Topic 57) — here we fake it for the example
  if (authToken !== 'valid-session-token') {
    return res.status(403).json({ error: 'Invalid auth token' }); // stop here too
  }

  req.userId = 'user_42';                  // attach the authenticated user's id to req
  next();                                  // token is valid — continue to the route
}

// ── A Router dedicated to /api/users routes ───────────────────────────────────
const userRouter = express.Router();

// GET /api/users/:userId — protected by requireAuth, applied only to this router
userRouter.get('/:userId', requireAuth, (req, res) => {
  const { userId } = req.params;           // the id requested in the URL
  if (userId !== req.userId) {
    return res.status(403).json({ error: 'Cannot access another user\'s data' });
  }
  res.json({ userId, email: 'ada@example.com', role: 'admin' }); // pretend DB lookup result
});

// POST /api/users/:userId/sessions — creates a new login session
userRouter.post('/:userId/sessions', (req, res, next) => {
  const { userId } = req.params;
  const { apiKey } = req.body;              // read apiKey from the parsed JSON body

  if (!apiKey) {
    const missingKeyError = new Error('apiKey is required in request body');
    missingKeyError.statusCode = 400;        // attach a status code for the error handler
    return next(missingKeyError);            // forward the error, skipping remaining routes
  }

  const sessionId = `sess_${Date.now()}`;    // pretend session generation
  res.status(201).json({ sessionId, userId });
});

// Mount the router — every route inside is prefixed with /api/users
app.use('/api/users', userRouter);

// ── 404 handler — catches anything that didn't match a route above ────────────
app.use((req, res) => {
  res.status(404).json({ error: 'Route not found' });
});

// ── Centralized error-handling middleware — always LAST, always 4 params ──────
app.use((err, req, res, next) => {
  console.error('[error]', err.message);     // log server-side for debugging
  const statusCode = err.statusCode || 500;  // use attached status or default to 500
  res.status(statusCode).json({ error: err.message }); // never send err.stack to the client
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`API running on port ${PORT}`));
```

---

## Common mistakes

### Mistake 1 — Forgetting `express.json()`, so `req.body` is `undefined`

```js
// ❌ WRONG — no body parser registered, req.body is undefined for JSON requests
const express = require('express');
const app = express();

app.post('/users', (req, res) => {
  const { userId } = req.body;   // TypeError-adjacent bug: req.body is undefined
  res.json({ userId });          // sends { userId: undefined }
});

// ✅ CORRECT — register express.json() BEFORE routes that read req.body
const express = require('express');
const app = express();

app.use(express.json());         // now req.body is populated from JSON payloads

app.post('/users', (req, res) => {
  const { userId } = req.body;   // works correctly now
  res.json({ userId });
});
```

### Mistake 2 — Forgetting to call `next()`, so the request hangs forever

```js
// ❌ WRONG — middleware does work but never calls next() and never sends a response
app.use((req, res, next) => {
  const requestStart = Date.now();
  req.requestStart = requestStart;
  // missing next() — every request to every route now hangs until it times out
});

app.get('/ping', (req, res) => {
  res.send('pong'); // this line is NEVER reached — the chain stopped above
});

// ✅ CORRECT — always call next() unless you deliberately end the response here
app.use((req, res, next) => {
  req.requestStart = Date.now();
  next();           // hand control to the next middleware/route
});

app.get('/ping', (req, res) => {
  res.send('pong'); // now this actually runs
});
```

### Mistake 3 — Registering the error handler before other routes (or with wrong arity)

```js
// ❌ WRONG — error handler registered FIRST, and only has 3 parameters
const express = require('express');
const app = express();

app.use((err, req, res) => {              // only 3 params — Express does NOT treat this as
  res.status(500).json({ error: err.message }); // an error handler, it's a normal middleware
});                                        // and it runs on EVERY request, crashing on `err.message`

app.get('/orders/:orderId', (req, res) => {
  res.json({ orderId: req.params.orderId });
});

// ✅ CORRECT — error handler has 4 params and is registered LAST, after all routes
const express = require('express');
const app = express();

app.get('/orders/:orderId', (req, res) => {
  res.json({ orderId: req.params.orderId });
});

app.use((err, req, res, next) => {        // exactly 4 params, registered last
  console.error(err.stack);
  res.status(err.statusCode || 500).json({ error: err.message });
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a small Express server with:
1. A `GET /status` route that responds with `{ status: 'running', uptime: process.uptime() }`
2. A logging middleware (registered with `app.use()`) that prints the HTTP method and URL of every incoming request to the console before the route runs
3. A `GET /greet/:name` route that responds with `{ message: 'Hello, <name>!' }` using the `name` route parameter
4. Start the server on port `4000` and confirm both routes work

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an Express app that manages an in-memory list of "products" (a plain array is fine, no database):
1. Use `express.json()` so you can accept JSON bodies
2. Create an `express.Router()` mounted at `/api/products` with:
   - `GET /` — returns the full array of products
   - `GET /:productId` — returns a single product by id, or a `404` with `{ error: 'Product not found' }` if it doesn't exist
   - `POST /` — reads `{ name, price }` from `req.body`, generates a new id, pushes it into the array, and responds with `201` and the created product
3. Add a custom middleware (only on the `POST /` route) that rejects the request with `400` if `name` or `price` is missing, using `next(err)` with a custom `Error` that has a `statusCode` property
4. Add a centralized error-handling middleware at the very end that reads `err.statusCode` and sends a JSON error response

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small "mini API gateway" using Express that simulates a realistic multi-router backend:
1. Create two separate `express.Router()` instances: `userRouter` (mounted at `/api/users`) and `orderRouter` (mounted at `/api/orders`)
2. Write a reusable `requireApiKey` middleware that checks for a `x-api-key` header equal to a hardcoded `'secret-key-123'`; apply it only to the `orderRouter` (not `userRouter`)
3. In `userRouter`, add `GET /:userId` (returns a fake user object) and `POST /` (creates a fake user, validates that `email` exists in the body, otherwise calls `next()` with a `400` error)
4. In `orderRouter`, add `GET /:orderId` (returns a fake order) and `POST /` (creates a fake order tied to a `userId` from the body)
5. Add a request-timing middleware at the very top of the app (before both routers) that records `Date.now()` on `req`, and after the response finishes (hint: listen to the `res.on('finish', ...)` event), logs how many milliseconds the request took
6. Add a `404` handler for unmatched routes and a final 4-parameter error handler that distinguishes between validation errors (`400`) and unexpected errors (`500`)
7. Test manually: a request to an order route WITHOUT the `x-api-key` header should return `401`, and WITH the correct header should succeed

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE PIECES
  const app = express();              → creates the application
  app.use(fn)                         → registers middleware for ALL routes/methods
  app.use('/prefix', router)          → mounts a Router under a path prefix
  app.get/post/put/delete/patch(...)  → registers a route for that HTTP method
  app.listen(port, cb)                → starts the HTTP server

MIDDLEWARE SIGNATURE
  (req, res, next)          → normal middleware / route handler
  (err, req, res, next)     → error-handling middleware (EXACTLY 4 params, order matters)

req / res QUICK FACTS
  req.params      → route params, e.g. /users/:userId → req.params.userId
  req.query       → query string, e.g. ?page=2 → req.query.page
  req.body        → parsed body (needs express.json() / express.urlencoded() first)
  req.headers     → all request headers, lowercase keys
  res.status(code)  → set HTTP status code (chainable)
  res.json(obj)     → send JSON, sets Content-Type automatically
  res.send(str)     → send plain text/html
  next()            → pass control to next middleware
  next(err)         → skip to the nearest error-handling middleware

ROUTER
  const router = express.Router();
  router.get('/ping', handler);
  app.use('/api', router);   → routes become /api/ping etc.

MIDDLEWARE ORDER MATTERS
  1. Body parsers (express.json(), express.urlencoded())
  2. Logging / auth / custom middleware
  3. Routes (app.get, router mounted with app.use)
  4. 404 handler (catch-all app.use at the end)
  5. Error handler (4-param function, ALWAYS LAST)

COMMON GOTCHAS
  Forgetting next()              → request hangs forever, no response ever sent
  Forgetting express.json()      → req.body is undefined
  Error handler not 4 params     → Express treats it as normal middleware, not an error handler
  Error handler registered early → it never gets reached for errors that occur after it
  Throwing inside async handler  → NOT auto-caught in older Express; wrap in try/catch + next(err)
                                    (Express 5 auto-forwards rejected promises from async handlers)

STATUS CODE CONVENTIONS
  200 OK          → successful GET/PUT
  201 Created     → successful POST that created a resource
  400 Bad Request → client sent invalid data
  401 Unauthorized→ missing/invalid auth credentials
  403 Forbidden   → authenticated but not allowed to do this
  404 Not Found   → route or resource doesn't exist
  500 Server Error→ unexpected server-side failure
```

---

## Connected topics

- **52 — Building a REST API with raw http** — shows what Express automates: manual routing and JSON parsing you'd otherwise write by hand with the raw `http` module.
- **54 — Express routing in depth** — expands on `req.params`, `req.query`, route groups, and `express.Router()` mounting patterns introduced here.
- **62 — Centralized error handling** — builds on the 4-parameter error middleware shown here into a full `AppError` class and consistent error-response strategy for production apps.
