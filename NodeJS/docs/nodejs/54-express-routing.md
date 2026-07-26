# 54 — Express routing in depth

## What is this?

Routing is how Express decides **which piece of code should run** for a given HTTP method + URL combination. Beyond the basic `app.get()` / `app.post()` you saw in Topic 53, Express lets you capture dynamic pieces of the URL as **route params** (`/users/:userId`), read extra filters from the **query string** (`?page=2&limit=10`), and organize related routes into their own mini-applications called **routers**, which you then **mount** onto a path prefix. Think of a large Express app like a office building directory: `express.Router()` is a separate floor with its own room numbers, and mounting it with `app.use('/api/users', userRouter)` is putting up the sign in the lobby that says "Users department — 3rd floor."

## Why does it matter for backend development?

Real APIs have dozens or hundreds of endpoints — `/api/users`, `/api/orders`, `/api/products`, each with their own GET/POST/PUT/DELETE variations, nested resources, and filters. Without proper routing patterns, every one of those would need its own giant `if/else` chain checking `req.url` manually (the pain you saw in Topic 52's raw `http` router). `express.Router()` lets you split routes into separate files per resource (`routes/users.js`, `routes/orders.js`), keeping each file focused and testable. Route params let a single route handler serve infinite dynamic values (`/users/42`, `/users/99`) instead of writing one handler per ID. Query strings let clients filter, sort, and paginate (`/products?category=shoes&sort=price`) without needing a new route for every combination. Every production Express codebase you will ever touch is organized this way.

---

## Syntax / API

```js
const express = require('express');
const app = express();

// ── 1. Route params — capture dynamic URL segments ──────────────────────────
// ':userId' becomes a named parameter, available on req.params
app.get('/users/:userId', (req, res) => {
  const { userId } = req.params;        // e.g. '/users/42' → userId = '42' (always a string)
  res.json({ userId });
});

// Multiple params in one route
app.get('/users/:userId/orders/:orderId', (req, res) => {
  const { userId, orderId } = req.params; // both captured from the URL
  res.json({ userId, orderId });
});

// ── 2. Query strings — optional filters after the '?' ───────────────────────
app.get('/products', (req, res) => {
  const { category, sort, page = 1 } = req.query; // req.query is always an object (never undefined)
  res.json({ category, sort, page });               // e.g. ?category=shoes&sort=price&page=2
});

// ── 3. express.Router() — a mini, mountable Express app ─────────────────────
const userRouter = express.Router();               // create an isolated router instance

userRouter.get('/', (req, res) => res.json({ msg: 'list users' }));       // GET /  (relative to mount point)
userRouter.get('/:userId', (req, res) => res.json({ userId: req.params.userId })); // GET /:userId
userRouter.post('/', (req, res) => res.json({ msg: 'create user' }));     // POST /

// ── 4. Mounting the sub-router onto a path prefix ────────────────────────────
app.use('/api/users', userRouter);
// Every path defined on userRouter is now prefixed with /api/users:
//   userRouter's '/'        → GET  /api/users
//   userRouter's '/:userId' → GET  /api/users/:userId
//   userRouter's '/' (POST) → POST /api/users

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## How it works — line by line

- `app.get('/users/:userId', handler)` tells Express: "match any GET request whose URL looks like `/users/` followed by one path segment, and store that segment under `req.params.userId`."
- Route params are always defined with a colon (`:name`) in the path string. Express turns that into a matching rule internally — it does not do this with regex you write yourself, it builds the regex for you.
- `req.params` is always a plain object holding the captured segments as **strings** — even `/users/42` gives you `userId: '42'`, not the number `42`. You must convert it yourself (`Number(userId)`) if you need a number.
- `req.query` reads everything after the `?` in the URL and turns it into an object automatically. `?category=shoes&sort=price` becomes `{ category: 'shoes', sort: 'price' }`. If a key never appears in the URL, it simply won't be a key on the object — it does not error.
- `express.Router()` creates a brand-new, isolated instance of Express's routing system. It behaves exactly like `app` for defining routes (`.get()`, `.post()`, `.use()`) but it is not attached to anything yet — it is just a collection of route definitions sitting in memory.
- `app.use('/api/users', userRouter)` is the **mounting** step. It tells the main app: "any incoming request whose URL starts with `/api/users`, strip that prefix off and hand the rest of the URL to `userRouter` to match." This is why the router's own routes are written *relative* to the mount point (`'/'` inside the router becomes `/api/users` in the real app).
- Because mounting is just `app.use()` with a path, you can mount as many routers as you want, each at a different prefix (`/api/users`, `/api/orders`, `/api/products`), keeping each resource's routes in its own file.

---

## Example 1 — basic

```js
// File: server.js
const express = require('express');
const app = express();

app.use(express.json()); // parses JSON request bodies into req.body (Topic 53)

// Route param example — capture a single dynamic segment
app.get('/users/:userId', (req, res) => {
  const { userId } = req.params;              // extract userId from the URL
  res.json({ message: `Fetched profile for user ${userId}` });
});

// Query string example — optional filters with defaults
app.get('/search', (req, res) => {
  // destructure req.query and provide sensible defaults for missing values
  const { keyword = '', page = '1', limit = '10' } = req.query;

  res.json({
    keyword,
    page: Number(page),     // convert string '1' → number 1
    limit: Number(limit),   // convert string '10' → number 10
  });
});

// Combined — route param AND query string on the same request
app.get('/users/:userId/posts', (req, res) => {
  const { userId } = req.params;              // from the URL path
  const { sort = 'newest' } = req.query;       // from the query string

  res.json({ userId, sort, posts: [] });       // placeholder response
});

app.listen(3000, () => {
  console.log('Server listening on http://localhost:3000');
  // Try: GET /users/42
  // Try: GET /search?keyword=node&page=2
  // Try: GET /users/42/posts?sort=oldest
});
```

---

## Example 2 — real world backend use case

```js
// File: routes/users.js
// A self-contained "users" resource router — this is the standard pattern
// for organizing routes in any medium-to-large Express codebase.

const express = require('express');
const userRouter = express.Router();

// Simulated data-access layer (in a real app this would call a database — Phase 12)
const usersDb = new Map([
  ['1', { userId: '1', name: 'Ravi Kumar', email: 'ravi@example.com' }],
  ['2', { userId: '2', name: 'Priya Singh', email: 'priya@example.com' }],
]);

// GET /api/users?limit=10  — list users, with optional pagination via query string
userRouter.get('/', (req, res) => {
  const { limit = '20' } = req.query;                 // read limit from query string, default 20
  const allUsers = Array.from(usersDb.values());      // convert Map values to an array
  res.json(allUsers.slice(0, Number(limit)));          // return only up to `limit` users
});

// GET /api/users/:userId  — fetch one user by ID (route param)
userRouter.get('/:userId', (req, res) => {
  const { userId } = req.params;                       // extract the ID from the URL
  const user = usersDb.get(userId);                    // look up the user

  if (!user) {
    return res.status(404).json({ error: `User ${userId} not found` }); // 404 if missing
  }
  res.json(user);                                      // 200 with the user object
});

// POST /api/users  — create a new user from the request body
userRouter.post('/', (req, res) => {
  const { name, email } = req.body;                    // req.body needs express.json() mounted upstream
  const newUserId = String(usersDb.size + 1);          // fake ID generation for this example

  const newUser = { userId: newUserId, name, email };
  usersDb.set(newUserId, newUser);                      // persist in the in-memory store

  res.status(201).json(newUser);                        // 201 Created with the new resource
});

// GET /api/users/:userId/orders  — nested resource, route param reused for a sub-collection
userRouter.get('/:userId/orders', (req, res) => {
  const { userId } = req.params;                        // which user's orders are we fetching?
  const { status } = req.query;                          // optional status filter, e.g. ?status=shipped

  res.json({ userId, status: status || 'all', orders: [] }); // placeholder for a real DB query
});

module.exports = userRouter;                             // export the router, not the whole app

// ────────────────────────────────────────────────────────────────────────────
// File: server.js
// Mounting several resource routers under a common /api/v1 prefix (Topic 60 goes deeper on this)

const express = require('express');
const userRouter = require('./routes/users');
// const orderRouter = require('./routes/orders');     // each resource gets its own router file

const app = express();
app.use(express.json());

app.use('/api/v1/users', userRouter);                    // all userRouter paths now live under /api/v1/users
// app.use('/api/v1/orders', orderRouter);

app.listen(3000, () => console.log('API running on port 3000'));
```

---

## Common mistakes

### Mistake 1 — Reading route params as numbers without converting

```js
// ❌ WRONG — req.params values are ALWAYS strings, even if they look numeric
app.get('/users/:userId', (req, res) => {
  const { userId } = req.params;
  if (userId === 42) {           // this NEVER matches — '42' !== 42
    res.json({ isAdmin: true });
  }
});

// ✅ CORRECT — explicitly convert before comparing or using as a number
app.get('/users/:userId', (req, res) => {
  const userId = Number(req.params.userId);   // '42' → 42
  if (Number.isNaN(userId)) {
    return res.status(400).json({ error: 'userId must be numeric' }); // guard bad input
  }
  if (userId === 42) {
    res.json({ isAdmin: true });
  }
});
```

### Mistake 2 — Mounting a router with the wrong path or forgetting the prefix inside route definitions

```js
// ❌ WRONG — repeating the mount prefix inside the router's own routes
const userRouter = express.Router();
userRouter.get('/api/users/:userId', (req, res) => { /* ... */ }); // duplicated prefix

app.use('/api/users', userRouter);
// Real path becomes /api/users/api/users/:userId — almost certainly not what you wanted

// ✅ CORRECT — router routes are relative to the mount point, never repeat the prefix
const userRouterFixed = express.Router();
userRouterFixed.get('/:userId', (req, res) => { /* ... */ }); // relative path only

app.use('/api/users', userRouterFixed);
// Real path is /api/users/:userId — exactly right
```

### Mistake 3 — Defining a specific route AFTER a param route that shadows it

```js
// ❌ WRONG — '/users/:userId' matches everything, including '/users/me',
// so the more specific route below is UNREACHABLE
app.get('/users/:userId', (req, res) => {
  res.json({ userId: req.params.userId });     // this fires for '/users/me' too!
});
app.get('/users/me', (req, res) => {
  res.json({ message: 'current logged-in user' }); // never reached — dead code
});

// ✅ CORRECT — declare specific/static routes BEFORE dynamic param routes
app.get('/users/me', (req, res) => {
  res.json({ message: 'current logged-in user' });  // matched first, exact string wins
});
app.get('/users/:userId', (req, res) => {
  res.json({ userId: req.params.userId });           // falls through here for everything else
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a small Express server with a single route `GET /products/:productId` that:
1. Reads `productId` from the route params
2. Reads an optional `currency` query param, defaulting to `'USD'` if not provided
3. Responds with JSON containing both values, e.g. `{ productId: '7', currency: 'USD' }`

Test it by visiting `/products/7` and `/products/7?currency=INR` and confirming the response changes correctly.

```js
// Write your code here
```

---

### Exercise 2 — medium

Create a `express.Router()` for a "notes" resource in its own module and mount it under `/api/notes`. The router must support:
1. `GET /` — list all notes, with an optional `?limit=` query string to cap how many are returned
2. `GET /:noteId` — return a single note by ID, or a `404` JSON error if it does not exist
3. `POST /` — create a new note from `req.body` (assume `{ title, content }`), returning `201` with the created note
4. `DELETE /:noteId` — remove a note by ID, returning `204` on success or `404` if the ID does not exist

Use an in-memory array or `Map` to store the notes (no real database needed). Export the router with `module.exports` and mount it in a separate `server.js`.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small multi-resource API with **two** separate router files — `routes/users.js` and `routes/orders.js` — wired together like this:

1. `GET /api/users/:userId` — returns a user object
2. `GET /api/users/:userId/orders` — returns all orders belonging to that `userId`, supporting an optional `?status=` query filter (e.g. `pending`, `shipped`, `delivered`)
3. `GET /api/orders/:orderId` — returns a single order directly by its own ID (independent of any user), or `404` if not found
4. Both routers must be mounted under a shared `/api/v1` prefix in `server.js` (so the final paths are `/api/v1/users/...` and `/api/v1/orders/...`)
5. Add a catch-all `404` handler at the very end of `server.js` that returns `{ error: 'Route not found' }` in JSON for any unmatched path — think carefully about where this must be placed relative to your routers.

Use in-memory data (arrays/Maps) for users and orders. Verify with requests like `/api/v1/users/1/orders?status=shipped` and `/api/v1/orders/9?` for a nonexistent order.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
ROUTE PARAMS (dynamic URL segments)
  app.get('/users/:userId', handler)
  req.params.userId          → always a STRING, even for numeric-looking IDs
  Multiple params:            '/users/:userId/orders/:orderId'
  Convert when needed:        Number(req.params.userId)

QUERY STRINGS (after the '?')
  '/products?category=shoes&sort=price'
  req.query                  → { category: 'shoes', sort: 'price' }
  req.query is ALWAYS an object — missing keys are simply absent, never undefined-object errors
  Provide defaults:           const { page = '1' } = req.query;

EXPRESS.ROUTER()
  const router = express.Router();     → isolated, mountable mini-app
  router.get('/', handler)             → matches the MOUNT POINT itself
  router.get('/:id', handler)          → matches mountPoint + '/:id'
  module.exports = router;             → export it, mount elsewhere

MOUNTING
  app.use('/api/users', userRouter)    → prefixes every userRouter path with /api/users
  Router paths are RELATIVE to the mount point — never repeat the prefix inside the router
  Multiple routers can be mounted at different prefixes in the same app

ROUTE ORDER MATTERS
  Static/specific routes BEFORE dynamic param routes:
    app.get('/users/me', ...)      ← must come first
    app.get('/users/:userId', ...) ← otherwise this shadows '/users/me'

COMMON PATTERNS
  Nested resource:        router.get('/:userId/orders', ...)
  Pagination via query:   ?page=2&limit=20
  Filtering via query:    ?status=shipped&sort=newest
  Resource CRUD on one router:
    GET    /            list
    GET    /:id         read one
    POST   /            create
    PUT    /:id         update
    DELETE /:id         delete

GOTCHAS
  req.params values      → strings, always convert for numeric comparisons
  req.query values       → strings or arrays of strings, never numbers/booleans natively
  Router mount path      → do NOT duplicate it inside router.get() paths
  404 handler placement  → must be defined AFTER all routers/routes, or it fires too early
```

---

## Connected topics

- **53 — Express.js fundamentals** — the `app`, `req`/`res`/`next`, and middleware basics that routing builds on top of
- **55 — Express middleware in depth** — how middleware chains interact with individual routes and routers, and why order matters
- **56 — Request validation** — validating `req.params`, `req.query`, and `req.body` with Joi/Zod before your route logic ever runs
