# 86 — Testing Express routes

## What is this?

Testing Express routes means writing automated tests that make real HTTP requests to your Express app — without starting an actual server on a real port — and asserting on the response: status code, headers, and body. The tool that makes this possible is **Supertest**, a library that wraps your Express `app` object and lets you write `request(app).get('/users').expect(200)` style assertions. Think of it like a robot QA tester that fires requests at your API and checks the receipt every single time you run `npm test`, instead of you manually opening Postman after every code change.

## Why does it matter for backend development?

Every backend developer eventually breaks a route while "just fixing something small" in a different file — a middleware order change, a validation tweak, a typo in a status code. Route tests catch that instantly, in seconds, before it reaches production. Supertest is the industry-standard way to test Express (and Fastify, Koa, etc.) APIs because it needs no real network port, runs fast in CI pipelines, and lets you test the full request/response cycle including middleware, validation, auth guards, and error handlers — not just isolated functions. Any backend job that touches Node.js will expect you to write these tests as part of a pull request.

---

## Syntax / API

```js
// Install: npm install --save-dev supertest jest

const request = require('supertest');   // supertest — makes HTTP requests against an app
const app     = require('../app');      // your Express app (NOT app.listen() — just the app)

// Basic GET request test
test('GET /users returns 200 and a list', async () => {
  const response = await request(app)   // pass the app directly, no server needed
    .get('/users')                      // simulate an HTTP GET to /users
    .set('Authorization', 'Bearer token123'); // set a request header

  expect(response.status).toBe(200);           // assert on status code
  expect(response.body).toBeInstanceOf(Array); // assert on parsed JSON body
});

// POST request with a JSON body
test('POST /users creates a new user', async () => {
  const requestBody = { name: 'Asha Verma', email: 'asha@example.com' };

  const response = await request(app)
    .post('/users')                     // simulate an HTTP POST
    .send(requestBody)                  // attach JSON body (sets Content-Type automatically)
    .expect('Content-Type', /json/)     // supertest's built-in header assertion
    .expect(201);                       // supertest's built-in status assertion (throws if wrong)

  expect(response.body.userId).toBeDefined(); // server should return the new user's id
});
```

---

## How it works — line by line

- `require('supertest')` loads a small library built on top of Node's `http` module. It knows how to spin up a temporary, ephemeral server behind the scenes just for the duration of one request.
- `require('../app')` imports your actual Express `app` — the same object your real server uses — but this file must export `app` **without** calling `app.listen(PORT)`. Listening on a real port is what the real server file (`server.js`) does; the test file only needs the app definition.
- `request(app)` hands that app to Supertest. Supertest binds it to a random free port internally, sends the request, waits for the response, then tears the temporary server down — all automatically, all in memory-fast time.
- `.get('/users')`, `.post('/users')`, `.put(...)`, `.delete(...)` pick the HTTP method and the path, exactly like a real client would.
- `.set(header, value)` attaches a request header — used for auth tokens, content negotiation, custom headers.
- `.send(requestBody)` attaches a body to the request. Supertest auto-detects it's an object and serializes it as JSON with the right `Content-Type` header.
- `.expect(status)` or `.expect(header, value)` are **built-in assertions** — if they fail, Supertest throws immediately with a clear message, separate from your test framework's own `expect()`.
- `await` is required because everything Supertest does is asynchronous — the request goes out, the app processes it through its middleware chain, and the response comes back on a future tick.
- `response.body` is the already-parsed JSON response body; `response.status` and `response.headers` give you the rest of what a normal HTTP client would see.

---

## Example 1 — basic

```js
// File: src/app.js
// The Express app itself — no app.listen() here, so it is safely importable in tests.

const express = require('express');
const app = express();

app.use(express.json());   // parse incoming JSON request bodies into req.body

// Simple in-memory "database" just for this example
const users = [{ userId: 1, name: 'Ravi Kumar' }];

// GET /users — return the list of users
app.get('/users', (req, res) => {
  res.status(200).json(users);          // 200 OK with the users array as JSON
});

// GET /users/:userId — return a single user or 404
app.get('/users/:userId', (req, res) => {
  const userId = Number(req.params.userId);      // route params are always strings, convert
  const foundUser = users.find((u) => u.userId === userId);

  if (!foundUser) {
    return res.status(404).json({ error: 'User not found' }); // 404 when missing
  }
  res.status(200).json(foundUser);       // 200 OK with the found user
});

module.exports = app;   // export the app object itself, ready to be imported by tests
```

```js
// File: src/app.test.js
// Basic Supertest test for the two routes above.

const request = require('supertest');   // HTTP-assertion helper
const app     = require('./app');       // the app under test (no server listening)

describe('GET /users', () => {
  test('responds with 200 and an array of users', async () => {
    const response = await request(app).get('/users'); // fire a GET request

    expect(response.status).toBe(200);              // status must be 200 OK
    expect(Array.isArray(response.body)).toBe(true); // body must be an array
    expect(response.body[0].name).toBe('Ravi Kumar'); // spot-check the data
  });
});

describe('GET /users/:userId', () => {
  test('responds with 200 for an existing user', async () => {
    const response = await request(app).get('/users/1'); // request an existing id

    expect(response.status).toBe(200);
    expect(response.body.userId).toBe(1);
  });

  test('responds with 404 for a missing user', async () => {
    const response = await request(app).get('/users/999'); // id that does not exist

    expect(response.status).toBe(404);              // should be Not Found
    expect(response.body.error).toBe('User not found'); // matches the app's error message
  });
});
```

---

## Example 2 — real world backend use case

```js
// File: src/routes/auth.routes.js
// A login route protected by validation middleware, auth middleware, and a
// centralized error handler — the realistic shape of a production Express app.

const express = require('express');
const jwt     = require('jsonwebtoken');
const router  = express.Router();

const JWT_SECRET = process.env.JWT_SECRET || 'test-secret'; // fallback only for local/test runs

// Middleware: validates that email and password are present
function validateLoginBody(req, res, next) {
  const { email, password } = req.body;
  if (!email || !password) {
    // Delegate to the centralized error handler instead of responding directly
    const validationError = new Error('Email and password are required');
    validationError.statusCode = 400;
    return next(validationError);
  }
  next(); // valid — move on to the route handler
}

// POST /auth/login
router.post('/login', validateLoginBody, (req, res, next) => {
  const { email, password } = req.body;

  // Fake credential check — a real app hits the database here
  if (email !== 'user@example.com' || password !== 'correct-password') {
    const authError = new Error('Invalid credentials');
    authError.statusCode = 401;
    return next(authError);           // hand off to the error-handling middleware
  }

  const authToken = jwt.sign({ email }, JWT_SECRET, { expiresIn: '1h' }); // sign a JWT
  res.status(200).json({ authToken });                                   // return it
});

module.exports = router;
```

```js
// File: src/app.js
// Wires the router and a centralized error-handling middleware into the app.

const express     = require('express');
const authRouter  = require('./routes/auth.routes');
const app = express();

app.use(express.json());
app.use('/auth', authRouter);

// Centralized error handler — MUST have 4 params so Express recognizes it as one
app.use((err, req, res, next) => {
  const statusCode = err.statusCode || 500;      // default to 500 if not set
  res.status(statusCode).json({ error: err.message }); // consistent error shape
});

module.exports = app;
```

```js
// File: src/routes/auth.routes.test.js
// Tests covering the happy path, validation middleware, and the error handler.

const request = require('supertest');
const app     = require('../app');

describe('POST /auth/login', () => {
  test('returns 400 when the request body is missing fields (middleware test)', async () => {
    const response = await request(app)
      .post('/auth/login')
      .send({});                                   // empty body — should fail validation

    expect(response.status).toBe(400);              // validateLoginBody kicked in
    expect(response.body.error).toBe('Email and password are required');
  });

  test('returns 401 for wrong credentials (error handler test)', async () => {
    const requestBody = { email: 'user@example.com', password: 'wrong-password' };

    const response = await request(app)
      .post('/auth/login')
      .send(requestBody);

    expect(response.status).toBe(401);               // handled by the error middleware
    expect(response.body.error).toBe('Invalid credentials');
  });

  test('returns 200 and an authToken for correct credentials (happy path)', async () => {
    const requestBody = { email: 'user@example.com', password: 'correct-password' };

    const response = await request(app)
      .post('/auth/login')
      .send(requestBody)
      .expect('Content-Type', /json/);

    expect(response.status).toBe(200);
    expect(typeof response.body.authToken).toBe('string'); // a JWT string was returned
  });
});
```

---

## Common mistakes

### Mistake 1 — Testing against a real running server instead of the app object

```js
// ❌ WRONG — starts a real server, ties up a port, and needs manual cleanup
const app = require('../app');
app.listen(3000);                       // now a real server is running during tests
const response = await request('http://localhost:3000').get('/users');
// Flaky in CI: port conflicts, leaked processes, slower tests

// ✅ CORRECT — pass the app object directly, Supertest handles the ephemeral server
const request = require('supertest');
const app = require('../app');          // app.js never calls app.listen() itself
const response = await request(app).get('/users'); // no real port used, no cleanup needed
```

### Mistake 2 — Forgetting to await the request, so the assertion runs before the response arrives

```js
// ❌ WRONG — missing await means the test finishes before the request completes
test('GET /users works', () => {
  const response = request(app).get('/users'); // this is a Promise, not the response!
  expect(response.status).toBe(200);            // fails: status is undefined on a Promise
});

// ✅ CORRECT — mark the test async and await the request
test('GET /users works', async () => {
  const response = await request(app).get('/users'); // wait for the real response
  expect(response.status).toBe(200);                  // now response.status actually exists
});
```

### Mistake 3 — Only testing the happy path and never the error handler or middleware rejections

```js
// ❌ WRONG — only checks that a valid request works, ignoring what happens when it doesn't
test('POST /users creates a user', async () => {
  const response = await request(app).post('/users').send({ name: 'Meera Joshi' });
  expect(response.status).toBe(201);
  // No test for: missing name, duplicate email, malformed JSON, auth middleware rejecting it
});

// ✅ CORRECT — cover validation middleware and the error handler explicitly too
test('POST /users creates a user (happy path)', async () => {
  const response = await request(app).post('/users').send({ name: 'Meera Joshi' });
  expect(response.status).toBe(201);
});

test('POST /users returns 400 when name is missing (middleware test)', async () => {
  const response = await request(app).post('/users').send({});   // triggers validation
  expect(response.status).toBe(400);
});

test('POST /users returns 500 shape when the DB throws (error handler test)', async () => {
  // simulate a downstream failure, e.g. by mocking the DB layer to throw
  const response = await request(app).post('/users').send({ name: 'DUPLICATE_TRIGGER' });
  expect(response.status).toBe(500);
  expect(response.body.error).toBeDefined(); // centralized handler returned a clean error shape
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a tiny Express app in `src/health.js` with a single route `GET /health` that returns `{ status: 'ok', uptime: <number> }` with a 200 status code (export the `app`, do not call `listen`). Then write a Supertest test file that:
1. Sends a GET request to `/health`
2. Asserts the status code is 200
3. Asserts `response.body.status` equals `'ok'`
4. Asserts `response.body.uptime` is a number

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an Express app with a `GET /products/:productId` route backed by an in-memory array of at least 3 products (each with `productId`, `name`, `price`). Then write Supertest tests that:
1. Return 200 and the correct product for a valid `productId`
2. Return 404 with a JSON `{ error: '...' }` body for a `productId` that does not exist
3. Return 400 if `productId` is not a valid number (e.g. `/products/abc`) — add a small middleware that checks this before the route handler runs, and write a separate test proving that middleware runs (test it in isolation from the "not found" case)

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small Express API for a `POST /orders` endpoint that:
1. Requires an `Authorization: Bearer <authToken>` header — write auth middleware that rejects requests without a valid-looking token with 401
2. Validates the request body has `userId` (number) and `items` (non-empty array) — reject invalid bodies with 400 via a validation middleware
3. Simulates an "out of stock" business rule: if any item's `quantity` is greater than 10, call `next(error)` with a custom error carrying `statusCode = 422`
4. On success, responds 201 with `{ orderId, userId, itemCount }`
5. Has a centralized error-handling middleware that returns `{ error: message }` with the correct status code for every failure case above

Then write a Supertest suite covering all five behaviors (auth rejection, validation rejection, business-rule rejection, happy path, and confirming the error handler shape is consistent across all three failure types).

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
SETUP
  npm install --save-dev supertest jest
  app.js must export the Express `app` WITHOUT calling app.listen()
  server.js (separate file) imports app and calls app.listen(PORT)

BASIC REQUEST
  const request = require('supertest');
  const app     = require('../app');
  const response = await request(app).get('/path');

METHODS
  .get(path)  .post(path)  .put(path)  .patch(path)  .delete(path)

BUILDING THE REQUEST
  .send(requestBody)              → attach JSON/form body
  .set('Header-Name', value)      → set a request header (e.g. Authorization)
  .query({ page: 1, limit: 10 })  → attach query string params

SUPERTEST BUILT-IN ASSERTIONS (throw immediately on mismatch)
  .expect(200)                    → assert status code
  .expect('Content-Type', /json/) → assert a header (regex or string)

RESPONSE OBJECT (after await)
  response.status    → numeric status code
  response.body       → parsed JSON body
  response.headers    → response headers object
  response.text       → raw response body as string

WHAT TO TEST ON EVERY ROUTE
  - Happy path (valid input → correct status + body)
  - Validation middleware (bad input → 400)
  - Auth middleware (missing/invalid token → 401/403)
  - Not found cases (→ 404)
  - Centralized error handler (thrown/forwarded errors → correct status + { error })

GOTCHAS
  - Never call app.listen() in the file you import for tests
  - Always `await` (or return) the Supertest request — it's a Promise
  - Test error paths, not just success paths — middleware bugs hide there
  - Reset in-memory/mock state between tests (beforeEach) to avoid test order dependence
  - Error middleware needs exactly 4 params: (err, req, res, next) or Express won't treat it as one
```

---

## Connected topics

- **53 — Express.js fundamentals** — `app`, `req`/`res`/`next`, and error middleware are the exact objects Supertest exercises in every test.
- **62 — Centralized error handling** — the `AppError` / error-handler pattern tested here is the same pattern that keeps error responses consistent across a real API.
- **85 — Mocking and stubbing** — combine `jest.mock()` with Supertest to isolate route tests from real databases or external HTTP calls.
