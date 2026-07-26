# 89 — TDD workflow

## What is this?

Test-Driven Development (TDD) is a workflow where you write a **failing test first**, write the **minimum code** to make it pass, then **refactor** the code while keeping the test green — repeating this cycle for every new piece of behavior. It is often summarized as **Red → Green → Refactor**. Think of it like a locksmith who first builds the exact lock (the test — "this key must open this door") before cutting a key to fit it, instead of cutting a random key and hoping it happens to work.

## Why does it matter for backend development?

Backend endpoints have precise contracts — a `POST /orders` route must return exactly 201 with an `orderId` on success, 400 on bad input, 401 without auth. TDD forces you to write down that contract as a test *before* the implementation exists, so the code you write is driven by requirements instead of guesswork, and every requirement automatically has a regression test protecting it forever. Teams that practice TDD on REST APIs ship endpoints with fewer edge-case bugs, because "what should happen when the body is empty?" gets answered in a test before it becomes a support ticket in production.

---

## Syntax / API

```js
// The TDD cycle has exactly three repeating steps, applied to ONE small behavior at a time.

// STEP 1 — RED: write a test for behavior that does not exist yet. It MUST fail.
test('GET /health returns 200 with status ok', async () => {
  const response = await request(app).get('/health'); // app doesn't even have this route yet
  expect(response.status).toBe(200);                  // this assertion will fail — that's correct
  expect(response.body.status).toBe('ok');
});
// Run the test suite now → RED (fails, because the route does not exist)

// STEP 2 — GREEN: write the SMALLEST amount of code that makes the test pass.
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' }); // just enough to satisfy the assertion, nothing extra
});
// Run the test suite now → GREEN (passes)

// STEP 3 — REFACTOR: clean up the code (or the test) without changing behavior.
app.get('/health', (req, res) => {
  const uptimeSeconds = Math.floor(process.uptime()); // improve the implementation
  res.status(200).json({ status: 'ok', uptimeSeconds }); // still passes the original assertions
});
// Run the test suite now → still GREEN (refactor did not break anything)
```

---

## How it works — line by line

- **Red** means you run your test suite and it fails — usually because the function, route, or module you are testing does not exist yet, or exists but does not yet do what the test expects. A failing test at this stage is not a bug; it is proof the test is actually checking something real (a test that always passes, even before code exists, is a useless test).
- Writing the test first forces you to think about the **API shape** before the implementation: what inputs come in, what status code and body come out, what errors look like. This is design work, not just verification work.
- **Green** means you write the least code possible to satisfy that one test — not the most elegant, complete, production-ready version, just enough to turn the red assertion into a passing one. Resisting the urge to "also handle five other cases while I'm here" keeps each cycle small and each failure easy to diagnose.
- Once green, you run the **entire** test suite, not just the new test — this confirms the new code didn't silently break something older.
- **Refactor** means improving the internal code (naming, structure, removing duplication, extracting a helper function) while the tests stay green throughout. If a refactor ever turns a test red, you either broke something or the refactor accidentally changed behavior — both are worth catching immediately, before more code is built on top.
- The cycle then repeats for the *next* smallest behavior — e.g. after the happy path, write a red test for the 400 validation case, make it green, refactor, then move to the 401 case, and so on — building the endpoint one contract line at a time instead of all at once.

---

## Example 1 — basic

```js
// File: src/utils/priceCalculator.js
// A plain function built with TDD — no Express involved yet, to show the cycle in isolation.

// ── CYCLE 1 ──────────────────────────────────────────────────────────────────
// RED: src/utils/priceCalculator.test.js
// test('calculateTotal returns 0 for an empty cart', () => {
//   expect(calculateTotal([])).toBe(0);          // fails: calculateTotal doesn't exist yet
// });

// GREEN: smallest implementation that satisfies the one test above
function calculateTotal(cartItems) {
  return 0; // hardcoded — but it IS enough to pass the current test, that's the point of TDD
}

// ── CYCLE 2 ──────────────────────────────────────────────────────────────────
// RED (new test): test('calculateTotal sums price * quantity for each item', () => {
//   const cartItems = [{ price: 10, quantity: 2 }, { price: 5, quantity: 3 }];
//   expect(calculateTotal(cartItems)).toBe(35);  // fails: hardcoded 0 cannot pass this
// });

// GREEN: now the hardcoded version must be replaced with real logic
function calculateTotalReal(cartItems) {
  return cartItems.reduce((sum, item) => sum + item.price * item.quantity, 0); // real sum
}

// REFACTOR: rename and export the real version, delete the throwaway hardcoded one
module.exports = { calculateTotal: calculateTotalReal }; // both old tests still pass after this
```

```js
// File: src/utils/priceCalculator.test.js
// The final test file that drove the two cycles above.

const { calculateTotal } = require('./priceCalculator'); // function under test

test('calculateTotal returns 0 for an empty cart', () => {
  expect(calculateTotal([])).toBe(0);                     // Cycle 1's test — still green
});

test('calculateTotal sums price * quantity for each item', () => {
  const cartItems = [
    { price: 10, quantity: 2 },  // 20
    { price: 5, quantity: 3 },   // 15
  ];
  expect(calculateTotal(cartItems)).toBe(35);              // Cycle 2's test — the driver of real logic
});
```

---

## Example 2 — real world backend use case

```js
// File: src/routes/users.routes.js
// A POST /users endpoint built entirely test-first, cycle by cycle, the way a
// backend developer would actually build it in a real pull request.

const express = require('express');
const router  = express.Router();

const users = []; // in-memory store for this example — a real app would use a database
let nextUserId = 1;

// ── CYCLE 1 (already green) — validation: missing email rejected with 400 ──
// ── CYCLE 2 (already green) — happy path: valid body creates a user, 201 ──
// ── CYCLE 3 (currently being built below) — duplicate email rejected with 409 ──

router.post('/', (req, res) => {
  const { email, name } = req.body;                       // pull fields out of the request body

  if (!email || !name) {                                  // Cycle 1's requirement
    return res.status(400).json({ error: 'email and name are required' });
  }

  const duplicate = users.find((u) => u.email === email);  // Cycle 3's requirement (new)
  if (duplicate) {
    return res.status(409).json({ error: 'email already registered' }); // 409 Conflict
  }

  const newUser = { userId: nextUserId++, email, name };   // Cycle 2's requirement
  users.push(newUser);
  res.status(201).json(newUser);                           // 201 Created with the new user
});

module.exports = { router, users }; // export `users` too, so tests can reset state between runs
```

```js
// File: src/routes/users.routes.test.js
// Each `test()` below is the RED test that was written BEFORE its matching
// line of implementation above existed — this is the actual TDD paper trail.

const request = require('supertest');
const express = require('express');
const { router, users } = require('./users.routes');

const app = express();
app.use(express.json());
app.use('/users', router);

beforeEach(() => {
  users.length = 0;     // reset in-memory state before every test — no order dependence
  // (nextUserId is not reset here on purpose — ids keep incrementing, which is realistic)
});

// This was RED first: route didn't validate, then GREEN after the `if (!email || !name)` check
test('returns 400 when email or name is missing', async () => {
  const response = await request(app).post('/users').send({ email: 'a@example.com' }); // no name
  expect(response.status).toBe(400);
  expect(response.body.error).toBe('email and name are required');
});

// This was RED first: route always crashed, then GREEN after the newUser + 201 lines were added
test('returns 201 and the created user for a valid body', async () => {
  const requestBody = { email: 'asha@example.com', name: 'Asha Verma' };
  const response = await request(app).post('/users').send(requestBody);

  expect(response.status).toBe(201);
  expect(response.body.userId).toBeDefined();
  expect(response.body.email).toBe(requestBody.email);
});

// This is the CURRENT cycle's RED test that justified adding the duplicate-check block
test('returns 409 when the email is already registered', async () => {
  const requestBody = { email: 'dupe@example.com', name: 'First User' };
  await request(app).post('/users').send(requestBody);            // first registration succeeds

  const response = await request(app).post('/users').send(requestBody); // same email again
  expect(response.status).toBe(409);
  expect(response.body.error).toBe('email already registered');
});
```

---

## Common mistakes

### Mistake 1 — Writing the implementation first, then writing a test to match it

```js
// ❌ WRONG — this is "test-after", not TDD: the test is shaped by whatever the code
// already does, so it can't catch requirements the code forgot to handle
function calculateDiscount(price, percent) {
  return price - price * (percent / 100); // written first, with no test driving its shape
}
test('calculateDiscount works', () => {
  expect(calculateDiscount(100, 10)).toBe(90); // written after, just confirms existing behavior
});

// ✅ CORRECT — write the failing test FIRST, based on the requirement, before any implementation
test('calculateDiscount reduces price by the given percent', () => {
  expect(calculateDiscount(100, 10)).toBe(90); // this fails first — function doesn't exist yet
});
// Only now write the function to make the test above pass
function calculateDiscount(price, percent) {
  return price - price * (percent / 100);
}
```

### Mistake 2 — Trying to make a huge feature green in one giant leap

```js
// ❌ WRONG — one enormous test covering auth + validation + business rules + persistence,
// with no intermediate green states — when it fails, you don't know which part is broken
test('POST /orders handles everything correctly', async () => {
  // ...50 lines asserting auth, validation, stock rules, DB writes, response shape all at once
});

// ✅ CORRECT — break it into small red/green/refactor cycles, one behavior at a time
test('rejects requests with no Authorization header (401)', async () => { /* ... */ });
test('rejects requests with an empty items array (400)', async () => { /* ... */ });
test('rejects an order where any item quantity exceeds stock (422)', async () => { /* ... */ });
test('creates the order and returns 201 on a fully valid request', async () => { /* ... */ });
// Each one gets its own red → green → refactor pass, building the endpoint incrementally
```

### Mistake 3 — Skipping the refactor step, leaving duplicated or messy code behind

```js
// ❌ WRONG — three routes all pass their tests, but each repeats the same validation
// logic inline; the tests are green, so the mess never gets cleaned up
router.post('/users', (req, res) => {
  if (!req.body.email || !req.body.name) return res.status(400).json({ error: 'invalid' });
  // ...
});
router.post('/admins', (req, res) => {
  if (!req.body.email || !req.body.name) return res.status(400).json({ error: 'invalid' }); // dup
  // ...
});

// ✅ CORRECT — with tests already green, safely extract the shared logic (the refactor step)
function requireEmailAndName(req, res, next) {
  if (!req.body.email || !req.body.name) {
    return res.status(400).json({ error: 'invalid' });
  }
  next();
}
router.post('/users', requireEmailAndName, (req, res) => { /* ... */ });
router.post('/admins', requireEmailAndName, (req, res) => { /* ... */ });
// Re-run the full suite after the refactor — all tests must still be green, unchanged
```

---

## Practice exercises

### Exercise 1 — easy

Using strict red-green-refactor, build a function `isValidPassword(password)` that returns `true`/`false`. Do the cycles in this exact order, running the test suite after each step and confirming red then green:
1. Cycle 1 — write a failing test asserting `isValidPassword('short')` returns `false` (too short, under 8 characters). Then write the smallest code to pass it.
2. Cycle 2 — write a failing test asserting `isValidPassword('longenoughpassword')` returns `true` (8+ characters, this is currently your only rule). Make it pass, refactoring if needed.
3. Cycle 3 — write a failing test asserting `isValidPassword('alllowercase123')` returns `false` because it has no uppercase letter. Add that rule.
4. After all three cycles, run the full test file once and confirm every test is green.

```js
// Write your code here
```

---

### Exercise 2 — medium

TDD a small Express `GET /products/:productId` endpoint backed by an in-memory array of products (`{ productId, name, price }`). Follow the cycle strictly — for each numbered behavior below, write the Supertest test FIRST (confirm it fails for the right reason), then write only enough route code to pass it, then move to the next:
1. Returns 200 and the correct product when `productId` matches an existing product.
2. Returns 404 with `{ error: 'Product not found' }` when `productId` does not match anything.
3. Returns 400 with `{ error: 'Invalid productId' }` when `productId` is not a valid number (e.g. `/products/abc`) — this behavior must be added as its own cycle, not folded into behavior 1 or 2.

Keep a short comment above each test noting whether it was the RED test that drove new code, so the TDD trail is visible.

```js
// Write your code here
```

---

### Exercise 3 — hard

TDD a full `POST /transfers` endpoint that transfers funds between two in-memory accounts (`{ accountId, balance }`). Build it strictly cycle by cycle — one red test, one green implementation step, one refactor pause, repeated for each behavior, in this order:
1. Rejects with 400 if `fromAccountId`, `toAccountId`, or `amount` is missing from the request body.
2. Rejects with 400 if `amount` is not a positive number.
3. Rejects with 404 if either account id does not exist in the in-memory store.
4. Rejects with 422 (`{ error: 'Insufficient funds' }`) if the `fromAccountId` account's balance is less than `amount`.
5. On success: decreases the `from` account's balance, increases the `to` account's balance, and responds 200 with `{ fromAccountId, toAccountId, amount, fromBalance, toBalance }`.
6. Refactor once all five behaviors are green: extract the account-lookup logic into a small helper function, then re-run the full suite to confirm nothing broke.

Reset the in-memory accounts array in a `beforeEach` so tests never depend on execution order.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
THE CYCLE
  RED       → write a test for behavior that doesn't exist yet; run it; it MUST fail
  GREEN     → write the SMALLEST code that makes that one test pass; run the FULL suite
  REFACTOR  → clean up code/tests while everything stays green; run the FULL suite again
  Repeat for the next small behavior

RULES OF THUMB
  - One behavior per cycle — never write two new tests before making the first one pass
  - "Smallest code to pass" can mean hardcoding a value temporarily — that's expected and fine
  - Always re-run the WHOLE suite after green and after refactor, not just the new test
  - A test that passes before any code exists is broken — verify it fails first (true RED)
  - Refactor is not optional — skipping it is how technical debt piles up even with 100% tests

TDD FOR A REST ENDPOINT — TYPICAL ORDER OF CYCLES
  1. Happy path with valid input        → 200/201 + correct body
  2. Missing/invalid required fields    → 400
  3. Auth missing/invalid               → 401
  4. Resource not found                 → 404
  5. Business-rule violation            → 409 / 422
  6. Refactor shared logic into middleware or helper functions once all above are green

WHY IT WORKS
  - Tests describe the CONTRACT before the code exists → code is driven by requirements
  - Every requirement automatically gets a regression test — nothing "forgotten" later
  - Small cycles → small diffs → easy to pinpoint exactly what broke when something goes red

COMMON PITFALLS
  - Test-after (writing code first, test second) — loses the design benefit of TDD entirely
  - One giant test for a whole feature — hard to debug, hard to know what's actually covered
  - Skipping refactor — tests stay green but the codebase quality never improves
  - Testing implementation details instead of behavior — makes refactor step break tests needlessly
```

---

## Connected topics

- **83 — Jest fundamentals** — `describe`, `test`, `expect`, and matchers are the exact tools used to write every red test in the TDD cycle.
- **86 — Testing Express routes** — Supertest is how the red/green cycles in this doc are actually run against a real Express endpoint.
- **88 — Test coverage** — coverage reports tell you which branches TDD's incremental cycles may have missed, and why 100% coverage still doesn't guarantee correctness.
