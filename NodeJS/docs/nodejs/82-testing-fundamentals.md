# 82 — Testing fundamentals

## What is this?

Testing is writing small pieces of code whose only job is to run your real code and check that it behaves the way you expect — automatically, without a human clicking through the app. Think of it like a car factory: a **unit test** checks one bolt is the right size in isolation, an **integration test** checks the engine and transmission actually connect and turn together, and an **end-to-end (e2e) test** takes the finished car for a real test drive on a real road. Each level catches different kinds of mistakes, and no single level catches everything.

## Why does it matter for backend development?

Backend code runs unattended, often at 3 AM, handling other people's money, data, and trust — you cannot manually click through every API endpoint before every deploy. Tests are what let you change code with confidence: if you refactor a function and every test still passes, you know you didn't silently break checkout, login, or payments. Companies reject candidates who "code and hope" — knowing which type of test to write for which situation (and not writing only e2e tests, which are slow and brittle) is a core signal of a professional backend engineer.

---

## Syntax / API

```js
// Node.js ships a built-in test runner since v18 — no extra package required
const test = require('node:test');            // the test() function registers a test case
const assert = require('node:assert/strict');  // strict equality/assertion helpers

// A single, standalone test — a name string + a function containing the check
test('adds two numbers correctly', () => {
  const result = 2 + 2;                        // the value your code actually produced
  assert.strictEqual(result, 4);               // throws if result !== 4, passing the test fails
});

// describe/it group related tests under a readable label — same test() engine underneath
const { describe, it, beforeEach } = require('node:test');

describe('calculateDiscount()', () => {        // groups every test below under this name
  let orderTotal;                               // shared setup variable for this group

  beforeEach(() => {                            // runs fresh before EVERY test in this group
    orderTotal = 150;                           // reset state so tests never leak into each other
  });

  it('applies 10% discount for orders over $100', () => {
    const discounted = orderTotal * 0.9;        // apply the rule under test
    assert.strictEqual(discounted, 135);        // check the exact expected number
  });

  it('applies no discount for orders under $100', () => {
    orderTotal = 50;                            // override the shared setup for this one case
    assert.strictEqual(orderTotal, 50);         // no discount rule triggered
  });
});

// Run all tests in the project:
//   node --test
// Node auto-discovers files matching *.test.js, *-test.js, or anything inside test/ folders
```

---

## How it works — line by line

`node:test` is a built-in module — `require('node:test')` gives you `test()`, plus `describe`/`it` as friendlier aliases for grouping. Each call to `test()` or `it()` registers one test case with a name and a function; nothing runs immediately — the test runner collects every registered test first, then executes them and prints a pass/fail report.

Inside a test function, `assert.strictEqual(actual, expected)` is the actual check: if `actual` does not match `expected`, it throws an `AssertionError`, and the test runner catches that thrown error and marks the test as **failed**. If the function returns without throwing, the test **passed**. This is the entire mechanism behind every test framework you will ever use (Jest, Mocha, Vitest) — register a function, run it, catch thrown errors, report results.

`describe()` is purely organizational — it groups related `it()` blocks under a shared label in the output and lets you scope shared setup (`beforeEach`) to just that group. `beforeEach` re-runs before every single test inside its `describe` block, which guarantees each test starts from a known, fresh state instead of leaking data from the previous test.

### The test pyramid

The pyramid shape: a wide base of unit tests, a smaller middle layer of integration tests, and a thin tip of e2e tests — describing both how many of each you should write and how expensive each one is.

| Level | Tests | Speed | Scope | Real dependencies? |
|---|---|---|---|---|
| **Unit** | One function or class | Milliseconds | Single unit, isolated | No — mocked/faked |
| **Integration** | Multiple units together | Fast–medium | 2+ modules interacting | Some — e.g. a real test DB |
| **E2E** | The whole running app | Slow (seconds) | Full stack, real requests | Yes — real HTTP, real DB |

The shape is a **pyramid** on purpose: write many cheap unit tests, fewer integration tests, and only a handful of e2e tests for the critical user journeys (signup, checkout, login). Flip it upside down — mostly e2e tests — and your test suite becomes slow, flaky, and painful to maintain; that anti-pattern is called an "ice cream cone."

---

## Example 1 — basic

```js
// File: src/utils/pricing.js
// A pure function — no I/O, no database, no network. Perfect unit test target.

function calculateOrderTotal(items, taxRate) {
  // items: [{ price: 10, quantity: 2 }, ...]
  const subtotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const tax = subtotal * taxRate;               // compute tax on the subtotal
  return Math.round((subtotal + tax) * 100) / 100; // round to 2 decimal places
}

module.exports = { calculateOrderTotal };


// File: test/pricing.test.js
const test = require('node:test');              // built-in test runner
const assert = require('node:assert/strict');   // built-in assertion library
const { calculateOrderTotal } = require('../src/utils/pricing'); // the code under test

test('calculates total with tax for a single item', () => {
  const items = [{ price: 100, quantity: 1 }];  // one $100 item
  const total = calculateOrderTotal(items, 0.08); // 8% tax
  assert.strictEqual(total, 108);                // 100 + 8 tax = 108
});

test('calculates total across multiple items', () => {
  const items = [                                // two different line items
    { price: 50, quantity: 2 },                  // 100 subtotal
    { price: 25, quantity: 1 },                  // + 25 subtotal
  ];
  const total = calculateOrderTotal(items, 0.10); // 10% tax
  assert.strictEqual(total, 137.5);               // 125 subtotal + 12.5 tax
});

test('returns 0 for an empty cart', () => {
  const total = calculateOrderTotal([], 0.08);    // no items at all
  assert.strictEqual(total, 0);                   // nothing to charge
});

// Run with: node --test test/pricing.test.js
```

---

## Example 2 — real world backend use case

```js
// A user-registration feature tested at all three pyramid levels.
// This is the pattern real backend teams follow: one feature, three kinds of confidence.

// ── 1) UNIT TEST — pure validation logic, zero dependencies ────────────────
// File: src/validators/userValidator.js
function validateRegistration(requestBody) {
  const errors = [];                              // collect every problem found
  if (!requestBody.email || !requestBody.email.includes('@')) {
    errors.push('Invalid email');                 // basic shape check, no regex needed here
  }
  if (!requestBody.password || requestBody.password.length < 8) {
    errors.push('Password must be at least 8 characters');
  }
  return { valid: errors.length === 0, errors };   // caller decides what to do with this
}
module.exports = { validateRegistration };

// File: test/unit/userValidator.test.js
const test = require('node:test');
const assert = require('node:assert/strict');
const { validateRegistration } = require('../../src/validators/userValidator');

test('unit: rejects a password shorter than 8 characters', () => {
  const result = validateRegistration({ email: 'user@example.com', password: 'short' });
  assert.strictEqual(result.valid, false);          // should fail validation
  assert.ok(result.errors.includes('Password must be at least 8 characters'));
});


// ── 2) INTEGRATION TEST — service + a real (in-memory) repository ─────────
// File: src/services/userService.js
async function registerUser(requestBody, userRepository) {
  const existing = await userRepository.findByEmail(requestBody.email); // real DB-style call
  if (existing) throw new Error('Email already registered');            // business rule
  const userId = await userRepository.create(requestBody);              // real insert
  return { userId, email: requestBody.email };
}
module.exports = { registerUser };

// File: test/integration/userService.test.js
const test = require('node:test');
const assert = require('node:assert/strict');
const { registerUser } = require('../../src/services/userService');

// An in-memory repository — same interface as the real DB repo, no mocks/spies needed
function createInMemoryUserRepository() {
  const rows = new Map();                          // stand-in for a real users table
  return {
    findByEmail: async (email) => rows.get(email) || null,
    create: async (requestBody) => {
      const userId = `user_${rows.size + 1}`;      // fake auto-increment id
      rows.set(requestBody.email, { userId, ...requestBody });
      return userId;
    },
  };
}

test('integration: registers a new user end-to-end through the service layer', async () => {
  const userRepository = createInMemoryUserRepository(); // fresh repo per test
  const result = await registerUser(
    { email: 'newuser@example.com', password: 'secretpass123' },
    userRepository
  );
  assert.ok(result.userId);                         // an id was actually generated
  assert.strictEqual(result.email, 'newuser@example.com');
});

test('integration: rejects a duplicate email across two calls', async () => {
  const userRepository = createInMemoryUserRepository();
  await registerUser({ email: 'dup@example.com', password: 'secretpass123' }, userRepository);
  await assert.rejects(                              // asserts the promise REJECTS
    () => registerUser({ email: 'dup@example.com', password: 'anotherpass1' }, userRepository),
    /Email already registered/                       // error message must match
  );
});


// ── 3) E2E TEST — real HTTP server, real network request, nothing faked ───
// File: test/e2e/registration.e2e.test.js
const test = require('node:test');
const assert = require('node:assert/strict');
const http = require('node:http');
const app = require('../../src/app');                // your real Express/http app

test('e2e: POST /api/users creates an account over real HTTP', async () => {
  const server = http.createServer(app).listen(0);    // port 0 = OS picks a free port
  const { port } = server.address();

  const response = await fetch(`http://localhost:${port}/api/users`, {
    method: 'POST',                                   // real HTTP verb over the wire
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email: 'e2e@example.com', password: 'realpassword1' }),
  });

  assert.strictEqual(response.status, 201);           // real status code from the real server
  const requestBody = await response.json();
  assert.ok(requestBody.userId);                       // real response shape

  server.close();                                      // always tear down the real server
});
```

---

## Common mistakes

### Mistake 1 — Writing mostly e2e tests (the "ice cream cone")

```js
// ❌ WRONG — testing every business rule by spinning up the full server and DB
// Slow (each test takes seconds), flaky (network/DB timing), and painful to maintain
test('rejects password under 8 chars', async () => {
  const server = startRealServer();                    // real server + real DB every time
  const response = await fetch(`${server.url}/api/users`, { method: 'POST', body: '...' });
  assert.strictEqual(response.status, 400);
  // Dozens of these for every validation rule = minutes-long test suite
});

// ✅ CORRECT — put validation logic in a pure function, unit-test it directly
// Keep e2e tests only for the handful of critical full-journey checks
test('unit: rejects password under 8 chars', () => {
  const result = validateRegistration({ email: 'a@b.com', password: 'short' });
  assert.strictEqual(result.valid, false);             // milliseconds, no server needed
});
```

### Mistake 2 — Testing implementation details instead of behavior

```js
// ❌ WRONG — asserting on HOW the function works internally
// Breaks the moment you refactor the internals, even if behavior is unchanged
test('uses a for loop internally', () => {
  const spy = jest.spyOn(Array.prototype, 'forEach'); // tying the test to implementation
  calculateOrderTotal([{ price: 10, quantity: 1 }], 0.1);
  assert.strictEqual(spy.mock.calls.length, 1);        // fragile — irrelevant to correctness
});

// ✅ CORRECT — assert on the OUTPUT for a given INPUT, never the internal mechanism
test('calculates the correct total regardless of implementation', () => {
  const total = calculateOrderTotal([{ price: 10, quantity: 1 }], 0.1);
  assert.strictEqual(total, 11);                       // survives any internal refactor
});
```

### Mistake 3 — Tests that depend on shared, leaking state

```js
// ❌ WRONG — a module-level variable persists across tests, causing order-dependent failures
let requestCount = 0;                                  // lives outside any test, never reset

test('first request increments count', () => {
  requestCount++;
  assert.strictEqual(requestCount, 1);                 // passes alone, fails if run after other tests
});

test('second request increments count', () => {
  requestCount++;
  assert.strictEqual(requestCount, 1);                 // FAILS — count is now 2, leaked from test 1
});

// ✅ CORRECT — reset shared state fresh before every single test
const { describe, it, beforeEach } = require('node:test');

describe('request counter', () => {
  let requestCount;

  beforeEach(() => {
    requestCount = 0;                                  // guaranteed clean slate every test
  });

  it('first request increments count', () => {
    requestCount++;
    assert.strictEqual(requestCount, 1);                // always passes, in any order
  });

  it('second request increments count independently', () => {
    requestCount++;
    assert.strictEqual(requestCount, 1);                // isolated — no leftover state
  });
});
```

---

## Practice exercises

### Exercise 1 — easy

Write a pure function `isValidApiKey(apiKey)` that returns `true` only if `apiKey` is a non-empty string, starts with `"sk_"`, and is at least 20 characters long. Then write at least 4 unit tests using `node:test` and `node:assert/strict` covering: a valid key, a key with the wrong prefix, a key that's too short, and `null`/`undefined` input. Run them with `node --test`.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an in-memory `sessionStore` with two async methods: `create(userId)` (generates a `sessionId`, stores `{ sessionId, userId, createdAt }`, and returns the `sessionId`) and `get(sessionId)` (returns the stored session or `null`). Then write a `loginService.login(requestBody, sessionStore)` function that checks a hardcoded username/password (`admin` / `password123`), and on success calls `sessionStore.create()` and returns the `sessionId`; on failure throws an error. Write integration tests (using the real in-memory store, no mocking) that check: successful login returns a valid session, a failed login throws, and the session returned by `get()` matches what was created.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small Express-style app (using raw `http` or Express, your choice) with a single route `GET /api/health` that returns `{ status: 'ok', uptime: process.uptime() }` with status 200, and a route `POST /api/orders` that accepts `{ items, taxRate }` in the JSON body, computes the total using a `calculateOrderTotal` function, and returns `{ total }` with status 201 — or status 400 with `{ error: 'items is required' }` if `items` is missing or empty. Write a full test suite with three files: a unit test file for `calculateOrderTotal` alone, an integration test file that calls the route handler function directly (no real network), and an e2e test file that starts a real server on an ephemeral port and hits both routes with `fetch`, verifying status codes and response bodies for both success and failure cases.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
THE THREE LEVELS
  Unit         → one function/class, all dependencies faked, milliseconds
  Integration  → 2+ real units working together (e.g. service + in-memory/test DB)
  E2E          → full running app, real HTTP requests, real infrastructure

THE PYRAMID (bottom = most tests, top = fewest)
  Unit (many, fast, cheap)
    → Integration (some, medium speed)
      → E2E (few, slow, most realistic)
  Upside-down pyramid ("ice cream cone") = slow, flaky suite — avoid it

WHEN TO USE WHICH
  Unit         → business logic, validation, calculations, pure functions
  Integration  → service talking to repository/DB, module boundaries
  E2E          → critical user journeys only (signup, login, checkout)

NODE'S BUILT-IN TEST RUNNER (Node 18+, no install needed)
  require('node:test')              → test(), describe(), it(), before/after hooks
  require('node:assert/strict')     → strictEqual, deepStrictEqual, ok, rejects, throws
  node --test                       → auto-discovers *.test.js and test/ folders
  node --test --watch               → re-runs on file changes

GOOD TEST HABITS
  - Test behavior/output, never internal implementation details
  - Reset shared state with beforeEach — no test should depend on another
  - One logical assertion focus per test — name describes exactly what's checked
  - Fake/mock at integration boundaries only when the real thing is slow or external
  - Prefer many fast unit tests over a handful of slow e2e tests

GOTCHAS
  - Testing only the "happy path" — always cover error/edge cases too
  - Sharing a single test DB across parallel tests → race conditions, flaky failures
  - Skipping teardown (closing servers/connections) → leaked handles, hanging test runs
```

---

## Connected topics

- **83 — Jest fundamentals** — a full-featured test framework with the same `describe`/`it`/`expect` mental model, plus built-in mocking and matchers beyond what `node:test` offers
- **86 — Testing Express routes** — `supertest` builds directly on the integration/e2e ideas here to test real Express req/res cycles without manually starting a server
- **89 — TDD workflow** — the red-green-refactor cycle applies the unit-testing habits from this topic as the driving process for writing new backend code
