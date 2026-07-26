# 60 — API versioning and structure

## What is this?

API versioning means putting a version marker (usually `/api/v1/...`) in your routes so you can change how your API behaves without breaking every client that already depends on it. Project structure is how you organize the files behind that API — splitting "receive the request" (router), "handle the request" (controller), and "do the actual work" (service) into separate layers instead of one giant file. Think of a restaurant: the menu number (v1, v2) is the version customers order from, the waiter is the router/controller taking the order, and the kitchen is the service layer actually cooking — customers never see the kitchen, and the kitchen can change its recipe without reprinting the menu.

## Why does it matter for backend development?

The moment your API has real clients — a mobile app, a frontend team, a third-party integration — you cannot freely change response shapes or remove fields without breaking someone. Versioning (`/api/v1/users` vs `/api/v2/users`) lets you introduce breaking changes safely: old clients keep hitting `v1` while new clients move to `v2`, and you retire `v1` on your own schedule. Structure matters just as much: a controller/service/router split means routes only deal with HTTP (req/res), controllers only translate HTTP into business calls, and services only contain business logic with zero knowledge of Express. This makes code testable (you can unit-test a service without spinning up a server), reusable (the same service can back a REST endpoint and a cron job), and maintainable as the team and codebase grow past a handful of files.

---

## Syntax / API

```js
// ── Folder structure convention ─────────────────────────────────────────────
// src/
//   routes/
//     v1/
//       user.routes.js       ← defines URL paths, wires them to controllers
//     v2/
//       user.routes.js
//   controllers/
//     user.controller.js     ← reads req, calls service, sends res
//   services/
//     user.service.js        ← business logic, talks to DB/models
//   app.js                   ← mounts versioned routers

// ── app.js — mounting versioned routers ─────────────────────────────────────
const express = require('express');
const app = express();

const userRoutesV1 = require('./routes/v1/user.routes');
const userRoutesV2 = require('./routes/v2/user.routes');

app.use(express.json()); // parse JSON request bodies

// Mount each version under its own base path
app.use('/api/v1/users', userRoutesV1); // old clients keep working
app.use('/api/v2/users', userRoutesV2); // new clients get the new shape

module.exports = app;

// ── routes/v1/user.routes.js — router (URL → controller) ───────────────────
const express = require('express');
const router = express.Router();
const userController = require('../../controllers/user.controller');

router.get('/', userController.listUsers);        // GET  /api/v1/users
router.get('/:userId', userController.getUser);    // GET  /api/v1/users/:userId
router.post('/', userController.createUser);       // POST /api/v1/users

module.exports = router;

// ── controllers/user.controller.js — controller (HTTP → service) ──────────
const userService = require('../services/user.service');

async function listUsers(req, res, next) {
  try {
    const users = await userService.findAll();       // delegate to service
    res.status(200).json({ data: users });           // shape the HTTP response
  } catch (err) {
    next(err);                                        // pass to error middleware
  }
}

module.exports = { listUsers };

// ── services/user.service.js — service (pure business logic) ───────────────
async function findAll() {
  // No req/res here — this could be called from a route, a script, or a test
  return dbConnection.query('SELECT id, name, email FROM users');
}

module.exports = { findAll };
```

---

## How it works — line by line

The version number lives in the URL path (`/api/v1/`), not scattered inside individual route files — that way one line in `app.js` decides which set of routes a version points to. `app.use('/api/v1/users', userRoutesV1)` means every request starting with `/api/v1/users` gets handed to the `userRoutesV1` router; Express strips that prefix before the router sees the rest of the path, so the router itself just defines `/`, `/:userId`, etc. without repeating `/api/v1` everywhere.

Inside the router, each route (`GET /`, `POST /`) is wired to a controller function, not to logic directly — the router's only job is "this HTTP verb + path maps to this function." The controller function receives `req` and `res`, pulls out what it needs (`req.params.userId`, `req.body`), calls a service function to do the real work, and then decides how to shape the HTTP response (status code, JSON body). It never contains SQL queries or business rules itself.

The service function is where the actual logic lives — database queries, calculations, calling other services. It takes plain JavaScript values in and returns plain JavaScript values out, with no knowledge that HTTP even exists. This is what makes it reusable and independently testable: you can call `userService.findAll()` from a unit test, a CLI script, or a cron job, and it behaves identically because it never touches `req` or `res`.

---

## Example 1 — basic

```js
// File: src/routes/v1/health.routes.js
// A minimal versioned route showing router → controller → service in one small slice

const express = require('express');
const router = express.Router();

// Controller inline for this trivial example (normally in its own file)
function getHealthStatus(req, res) {
  const uptimeSeconds = process.uptime();          // how long the process has run
  res.status(200).json({
    status: 'ok',                                  // simple health flag
    version: 'v1',                                  // which API version answered
    uptimeSeconds: Math.floor(uptimeSeconds),        // rounded for a clean response
  });
}

router.get('/', getHealthStatus);                  // GET /api/v1/health

module.exports = router;

// File: src/app.js
const express = require('express');
const app = express();

const healthRoutesV1 = require('./routes/v1/health.routes');

app.use('/api/v1/health', healthRoutesV1);          // mount under the versioned prefix

app.listen(3000, () => {
  console.log('Server listening on http://localhost:3000/api/v1/health');
});
```

---

## Example 2 — real world backend use case

```js
// File: src/services/order.service.js
// Business logic only — no Express, no req/res, fully unit-testable

const dbConnection = require('../db/connection');   // shared DB pool

async function createOrder(userId, items) {
  if (!items || items.length === 0) {
    // Throw a plain Error — controller/error middleware decides the HTTP status
    const err = new Error('Order must contain at least one item');
    err.statusCode = 400;
    throw err;
  }

  const totalAmount = items.reduce((sum, item) => sum + item.price * item.qty, 0);

  const result = await dbConnection.query(
    'INSERT INTO orders (user_id, total_amount) VALUES ($1, $2) RETURNING id',
    [userId, totalAmount]
  );

  return { orderId: result.rows[0].id, totalAmount };
}

module.exports = { createOrder };

// File: src/controllers/order.controller.js
// Translates HTTP into a service call and shapes the response

const orderService = require('../services/order.service');

async function createOrder(req, res, next) {
  try {
    const userId = req.user.id;                      // set earlier by auth middleware
    const { items } = req.body;                       // parsed JSON body

    const order = await orderService.createOrder(userId, items);

    res.status(201).json({ data: order });            // 201 Created for a new resource
  } catch (err) {
    next(err);                                          // centralized error handler responds
  }
}

module.exports = { createOrder };

// File: src/routes/v1/order.routes.js
// Pure routing — no logic, just wiring

const express = require('express');
const router = express.Router();
const authMiddleware = require('../../middleware/auth.middleware');
const orderController = require('../../controllers/order.controller');

router.post('/', authMiddleware, orderController.createOrder); // POST /api/v1/orders

module.exports = router;

// File: src/routes/v2/order.routes.js
// v2 adds a required "currency" field — a breaking change isolated to its own version
const express = require('express');
const router = express.Router();
const authMiddleware = require('../../middleware/auth.middleware');
const orderControllerV2 = require('../../controllers/orderV2.controller'); // separate v2 controller

router.post('/', authMiddleware, orderControllerV2.createOrder); // POST /api/v2/orders

module.exports = router;
```

---

## Common mistakes

### Mistake 1 — Putting business logic directly in the route file

```js
// ❌ WRONG — router does SQL queries, calculations, and error handling all at once
// Impossible to reuse or unit-test without an HTTP request
router.post('/orders', async (req, res) => {
  const total = req.body.items.reduce((sum, i) => sum + i.price * i.qty, 0);
  const result = await dbConnection.query(
    'INSERT INTO orders (user_id, total_amount) VALUES ($1, $2) RETURNING id',
    [req.user.id, total]
  );
  res.json({ orderId: result.rows[0].id });
});

// ✅ CORRECT — router delegates to controller, controller delegates to service
router.post('/orders', authMiddleware, orderController.createOrder);
// orderController calls orderService.createOrder(userId, items)
// orderService contains the SQL and the total calculation, reusable and testable alone
```

### Mistake 2 — No versioning at all, breaking every client on change

```js
// ❌ WRONG — a single unversioned route; renaming a field breaks every existing client instantly
app.get('/users/:userId', (req, res) => {
  res.json({ id: req.params.userId, fullName: 'renamed field breaks old mobile app' });
});

// ✅ CORRECT — old shape stays on v1, new shape ships on v2, both run side by side
app.use('/api/v1/users', userRoutesV1); // keeps returning { id, name } — old clients unaffected
app.use('/api/v2/users', userRoutesV2); // returns { id, fullName } — new clients opt in
// v1 gets deprecated and removed later, on a schedule communicated to clients
```

### Mistake 3 — Duplicating the entire codebase for every new version

```js
// ❌ WRONG — copy-pasting services and controllers for v2 even when nothing changed there
// src/controllers/v1/user.controller.js   (identical to v2 file, just copy-pasted)
// src/controllers/v2/user.controller.js   (99% duplicate code, maintenance nightmare)
// src/services/v1/user.service.js
// src/services/v2/user.service.js

// ✅ CORRECT — version only the layer that actually changed; share the rest
// src/routes/v1/user.routes.js   → uses the SAME userController as v2
// src/routes/v2/user.routes.js   → uses the SAME userController as v1
// src/controllers/user.controller.js   (shared, unversioned — no changes needed)
// src/services/user.service.js         (shared, unversioned — business logic didn't change)
// Only version a controller/service if its actual behavior differs between versions:
// src/controllers/orderV2.controller.js  (only this one differs — currency field added)
```

---

## Practice exercises

### Exercise 1 — easy

Build an Express app with a single resource, `products`, structured across three files: `src/routes/v1/product.routes.js`, `src/controllers/product.controller.js`, and `src/services/product.service.js`. The service should hold an in-memory array of at least 3 products (`{ id, name, price }`) and expose a `findAll()` function. The controller should call the service and return the products as JSON. The router should expose `GET /` and be mounted in `app.js` at `/api/v1/products`. Test it by starting the server and hitting `GET /api/v1/products`.

```js
// Write your code here
```

---

### Exercise 2 — medium

Extend the `products` API from Exercise 1 to support two versions at once:
1. `v1`: `GET /api/v1/products/:productId` returns `{ id, name, price }`
2. `v2`: `GET /api/v2/products/:productId` returns `{ id, name, priceInCents }` — same data, different shape (convert `price` dollars to `priceInCents` by multiplying by 100)

Keep a single shared `product.service.js` with one `findById(productId)` function. Create two separate controllers (`product.controller.js` for v1, `productV2.controller.js` for v2) that each call the same service but shape the response differently. Mount both versions in `app.js` and verify both endpoints return correct, differently-shaped data for the same product ID.

```js
// Write your code here
```

---

### Exercise 3 — hard

Design a small multi-resource API (`users` and `orders`) with a full folder structure: `routes/v1/`, `controllers/`, `services/`, and a `middleware/` folder containing a fake `authMiddleware` (it should just check for a header `x-api-key` and call `next()` if present, or respond `401` if missing). Requirements:
1. `orderService.createOrder(userId, items)` should throw an `Error` with a `.statusCode = 400` if `items` is empty, and calculate `totalAmount` from `price * qty` for each item.
2. `orderController.createOrder` must catch service errors with try/catch and call `next(err)`.
3. Add a centralized error-handling middleware in `app.js` (an Express middleware with 4 params: `err, req, res, next`) that reads `err.statusCode` (defaulting to 500) and returns `{ error: err.message }` with that status.
4. Mount everything under `/api/v1/orders`, protect the `POST /` route with `authMiddleware`, and manually test: a request with no `x-api-key` header gets `401`, a request with empty `items` gets `400` with a clear message, and a valid request gets `201` with the created order.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
LAYERS (request flows top to bottom)
  Router      → maps URL + HTTP verb to a controller function, no logic
  Controller  → reads req, calls service, shapes res, no business logic
  Service     → pure business logic / DB access, no req/res, fully testable

FOLDER STRUCTURE
  src/
    routes/
      v1/resource.routes.js
      v2/resource.routes.js
    controllers/
      resource.controller.js       ← shared unless behavior truly differs per version
    services/
      resource.service.js          ← shared, versioned only if logic itself changes
    middleware/
      auth.middleware.js
    app.js                         ← mounts each versioned router at its base path

VERSIONING RULE OF THUMB
  Only fork the layer that actually changed between versions.
  Router  → almost always forked (different path prefix)
  Controller → fork ONLY if response shape/validation differs
  Service → fork ONLY if business logic itself differs

MOUNTING VERSIONS
  app.use('/api/v1/users', userRoutesV1);
  app.use('/api/v2/users', userRoutesV2);
  // Express strips the mount prefix — router paths stay simple ('/', '/:id')

WHY VERSION AT ALL
  - Old clients keep working on v1 while you evolve v2
  - Deprecate and remove v1 on your own schedule, not the client's
  - Never silently change a field name/type on a live, unversioned endpoint

CONTROLLER RULES
  - try/catch around every async call, always next(err) on failure
  - Never contains SQL, calculations, or direct DB/model access
  - Only decides status code + response shape

SERVICE RULES
  - Never imports express, never touches req/res
  - Throws plain Error objects (optionally with .statusCode) on failure
  - Callable from routes, tests, scripts, or cron jobs identically

NEVER DO
  - Business logic inline in the route callback
  - One unversioned endpoint serving all clients forever
  - Duplicating an entire controller/service for a version when nothing changed
```

---

## Connected topics

- **53 — Express.js fundamentals** — the app/Router/middleware primitives this topic's folder structure is built on
- **54 — Express routing in depth** — `express.Router()`, mounting sub-routers, and route params used to build versioned routers
- **62 — Centralized error handling** — the `AppError` class and global error middleware pattern that catches errors thrown by services in this structure
