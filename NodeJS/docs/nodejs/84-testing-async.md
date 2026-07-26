# 84 — Testing async code

## What is this?

Testing async code means writing test cases that correctly wait for promises, `async/await` calls, and callbacks to finish before checking the result — and also verifying that failures (rejections, thrown errors) behave the way you expect. It matters because most backend code is asynchronous: database queries, API calls, file reads. Think of it like tasting a soup that's still cooking — if you taste it (assert) before it finishes simmering (the promise resolves), you get the wrong verdict, not because the soup is bad, but because you checked too early.

## Why does it matter for backend development?

Almost every function a backend developer writes touches something async: `dbConnection.query()`, `fetch()` calls to a payment API, `fs.promises.readFile()` for a template. If your test doesn't properly `await` these operations, the test finishes and reports "passed" before the actual work even ran — a false-positive green checkmark that hides real bugs. Backend developers rely on correctly written async tests to catch broken database queries, unhandled promise rejections, and API timeout bugs before they reach production, especially since these bugs are often silent (no crash, just a wrong or missing result).

---

## Syntax / API

```js
// Jest test file — testing async code has 3 valid patterns

// ── Pattern 1: async/await (preferred — cleanest, most readable) ───────────
test('fetches user by id', async () => {                 // mark the test function 'async'
  const user = await getUserById('user_101');             // await the promise inside the test
  expect(user.id).toBe('user_101');                        // assertion runs only after promise resolves
});

// ── Pattern 2: returning a promise (older style, still valid) ──────────────
test('fetches user by id (returned promise)', () => {
  return getUserById('user_101').then((user) => {          // Jest waits for the RETURNED promise
    expect(user.id).toBe('user_101');                       // assertion inside .then()
  });
});

// ── Pattern 3: testing rejections with .rejects ─────────────────────────────
test('rejects when user is not found', async () => {
  await expect(getUserById('missing_id')).rejects.toThrow('User not found');
  // .rejects unwraps the rejected promise, then .toThrow checks the error message
});

// ── Testing resolved values with .resolves ──────────────────────────────────
test('resolves with the correct email', async () => {
  await expect(getUserById('user_101')).resolves.toHaveProperty('email');
  // .resolves unwraps the resolved value so you can assert on it directly
});
```

---

## How it works — line by line

- Jest's `test()` function accepts a callback. If that callback is declared `async`, Jest automatically waits for the returned promise (every `async` function returns a promise) before deciding pass or fail.
- Inside an `async` test, using `await` pauses the test at that line until the promise settles — either resolves (continue to the next line) or rejects (throws, which fails the test unless you're expecting it).
- If you forget `async`/`await` and forget to `return` the promise, Jest has no way to know the test is still running — it marks the test "passed" immediately and any assertions inside `.then()` that fail later are silently swallowed. This is the single most common async testing bug.
- `.rejects` and `.resolves` are special matchers built into `expect()` specifically for promises. `.rejects` unwraps a rejected promise so you can assert on the error (its message, its type). `.resolves` unwraps a resolved promise so you can assert on the value directly, without an extra `.then()`.
- Both `.rejects` and `.resolves` must themselves be `await`-ed, because they return a promise too — forgetting the outer `await` reintroduces the same silent-pass bug.
- When testing a function that is supposed to fail, always assert on *something specific* about the failure (an error message, an error class) — not just "it threw" — otherwise the test would still pass even if it failed for a completely different, unrelated reason.

---

## Example 1 — basic

```js
// File: src/services/userService.js
// A simple async function to test — simulates a database lookup with a Promise

function getUserById(userId) {                              // takes a user id
  return new Promise((resolve, reject) => {                  // returns a Promise (simulates async DB call)
    setTimeout(() => {                                        // setTimeout simulates network/DB latency
      if (userId === 'user_101') {                             // pretend this id exists in the "database"
        resolve({ id: 'user_101', name: 'Asha Rao', email: 'asha@example.com' }); // resolve with user data
      } else {
        reject(new Error('User not found'));                   // reject with an Error for unknown ids
      }
    }, 50);                                                     // 50ms fake delay
  });
}

module.exports = { getUserById };
```

```js
// File: src/services/userService.test.js
// Testing the async function above with all three patterns

const { getUserById } = require('./userService');            // import the function under test

test('resolves with correct user using async/await', async () => {
  const user = await getUserById('user_101');                 // wait for the promise to resolve
  expect(user.name).toBe('Asha Rao');                          // assertion only runs after resolution
});

test('resolves with correct user using .resolves', async () => {
  await expect(getUserById('user_101')).resolves.toEqual({    // .resolves unwraps the value
    id: 'user_101',
    name: 'Asha Rao',
    email: 'asha@example.com',
  });
});

test('rejects for an unknown user id', async () => {
  await expect(getUserById('user_999')).rejects.toThrow('User not found'); // .rejects unwraps the error
});
```

---

## Example 2 — real world backend use case

```js
// File: src/services/authService.js
// A login function — the kind of async code backend devs test constantly

const bcrypt = require('bcrypt');                             // for password comparison

async function login(dbConnection, requestBody) {
  const { email, password } = requestBody;                     // pull credentials out of the request body

  const user = await dbConnection.findUserByEmail(email);       // async DB lookup
  if (!user) {
    throw new Error('Invalid email or password');               // don't reveal which field was wrong
  }

  const passwordMatches = await bcrypt.compare(password, user.passwordHash); // async hash comparison
  if (!passwordMatches) {
    throw new Error('Invalid email or password');
  }

  return { userId: user.id, sessionId: `sess_${user.id}_${Date.now()}` };   // successful login payload
}

module.exports = { login };
```

```js
// File: src/services/authService.test.js
// Testing login() with a FAKE dbConnection — no real database needed for a unit test

const bcrypt = require('bcrypt');
const { login } = require('./authService');

// A fake dbConnection object — just enough behavior to exercise login()
const fakeDbConnection = {
  findUserByEmail: async (email) => {                          // fake async lookup
    if (email === 'asha@example.com') {
      return { id: 'user_101', passwordHash: await bcrypt.hash('correctPass123', 10) };
    }
    return null;                                                 // simulate "no user found"
  },
};

test('logs in successfully with correct credentials', async () => {
  const result = await login(fakeDbConnection, {                // await the async login()
    email: 'asha@example.com',
    password: 'correctPass123',
  });

  expect(result.userId).toBe('user_101');                        // check the returned user id
  expect(result.sessionId).toMatch(/^sess_user_101_\d+$/);        // check sessionId shape with a regex
});

test('rejects login with wrong password', async () => {
  await expect(
    login(fakeDbConnection, { email: 'asha@example.com', password: 'wrongPass' })
  ).rejects.toThrow('Invalid email or password');                 // async rejection assertion
});

test('rejects login for unknown email', async () => {
  await expect(
    login(fakeDbConnection, { email: 'unknown@example.com', password: 'anything' })
  ).rejects.toThrow('Invalid email or password');
});
```

---

## Common mistakes

### Mistake 1 — Forgetting to `await` or `return` the promise

```js
// ❌ WRONG — the promise inside is never awaited or returned
// Jest thinks the test finished instantly and marks it PASSED
// even if the assertion inside .then() actually fails
test('fetches user', () => {
  getUserById('user_101').then((user) => {
    expect(user.name).toBe('Wrong Name');   // this assertion silently never blocks the test
  });
});

// ✅ CORRECT — await the promise so Jest waits for it to settle
test('fetches user', async () => {
  const user = await getUserById('user_101');  // test pauses here until resolved
  expect(user.name).toBe('Asha Rao');           // now this assertion actually gates pass/fail
});
```

### Mistake 2 — Wrapping an async assertion in try/catch without failing on missing rejection

```js
// ❌ WRONG — if getUserById() does NOT reject (bug fixed, behavior changed),
// the catch block never runs and the test passes with ZERO assertions executed
test('rejects for unknown user', async () => {
  try {
    await getUserById('user_999');
  } catch (error) {
    expect(error.message).toBe('User not found');
  }
  // if no error was thrown, this test still reports PASSED — false positive
});

// ✅ CORRECT — use .rejects, which fails the test if the promise does NOT reject
test('rejects for unknown user', async () => {
  await expect(getUserById('user_999')).rejects.toThrow('User not found');
  // if the promise resolves instead of rejecting, this line itself fails the test
});
```

### Mistake 3 — Testing only the "happy path" and ignoring rejection handling

```js
// ❌ WRONG — only checks success; a bug that breaks error handling goes undetected
test('login works', async () => {
  const result = await login(fakeDbConnection, {
    email: 'asha@example.com',
    password: 'correctPass123',
  });
  expect(result.userId).toBe('user_101');
  // no test ever checks what happens with a wrong password or missing user
});

// ✅ CORRECT — test both success AND failure paths for async functions
test('login works with correct credentials', async () => {
  const result = await login(fakeDbConnection, {
    email: 'asha@example.com',
    password: 'correctPass123',
  });
  expect(result.userId).toBe('user_101');
});

test('login fails with wrong password', async () => {
  await expect(
    login(fakeDbConnection, { email: 'asha@example.com', password: 'wrongPass' })
  ).rejects.toThrow('Invalid email or password');   // covers the rejection path explicitly
});
```

---

## Practice exercises

### Exercise 1 — easy

Write an async function `delayedGreeting(name, delayMs)` that returns a Promise resolving with the string `Hello, ${name}!` after `delayMs` milliseconds (use `setTimeout` inside the Promise). Then write a Jest test file that:
1. Tests that `delayedGreeting('Ravi', 20)` resolves with `'Hello, Ravi!'` using `async/await`.
2. Tests the same case again but using the `.resolves` matcher instead.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write an async function `withdrawFunds(accountBalance, amount)` that:
1. Returns a Promise.
2. Rejects with `new Error('Insufficient funds')` if `amount > accountBalance`.
3. Rejects with `new Error('Amount must be positive')` if `amount <= 0`.
4. Otherwise resolves with the new balance (`accountBalance - amount`).

Then write a Jest test file covering all three branches: a successful withdrawal, an insufficient-funds rejection, and a non-positive-amount rejection. Use `.rejects.toThrow(...)` for both rejection cases and check the exact new balance for the success case.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build an async `OrderProcessor` module with a function `placeOrder(dbConnection, orderRequest)` where:
1. `orderRequest` has `{ userId, productId, quantity }`.
2. It calls `await dbConnection.getProductStock(productId)` (async) to get current stock.
3. If stock is less than `quantity`, it throws `new Error('Out of stock')`.
4. Otherwise it calls `await dbConnection.reduceStock(productId, quantity)` (async) and then `await dbConnection.createOrder({ userId, productId, quantity })` (async), returning the created order object.
5. If ANY of the three async calls rejects unexpectedly (simulate a random `dbConnection.createOrder` failure), the error should propagate out of `placeOrder` unchanged (do not swallow it).

Write a fake `dbConnection` object with jest-style fake async methods (plain `async` functions returning canned values is fine) and write tests for: a successful order, an out-of-stock rejection, and a case where `createOrder` itself rejects and the error message passes through untouched.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
THREE WAYS TO TEST ASYNC CODE IN JEST
  1. async/await     → test('...', async () => { await something(); ... })
  2. return promise   → test('...', () => { return something().then(...) })
  3. done callback    → for callback-style APIs only (legacy) — prefer promisify + await

PROMISE MATCHERS (must be awaited themselves!)
  await expect(promise).resolves.toBe(value)      → asserts on resolved value
  await expect(promise).resolves.toEqual(obj)      → deep equality on resolved value
  await expect(promise).rejects.toThrow('message')  → asserts rejection + message
  await expect(promise).rejects.toBeInstanceOf(Error) → asserts rejection type

THE #1 BUG
  Forgetting await/return → Jest marks test PASSED before promise settles
  → assertions inside unreturned .then() are never actually checked

GOLDEN RULES
  - Mark test callback `async` whenever you use `await` inside it
  - Always `await` .rejects / .resolves — they return promises too
  - Test BOTH the resolve path and the reject path for every async function
  - Assert on a SPECIFIC error message/type, not just "it threw something"
  - Prefer async/await over .then() chains in tests — easier to read top-to-bottom

TIMEOUT GOTCHA
  Jest default test timeout is 5000ms
  → jest.setTimeout(10000) or test('name', async () => {...}, 10000) for slow tests
```

---

## Connected topics

- **83 — Jest fundamentals** — `describe`, `it`/`test`, `expect`, matchers, setup/teardown; the base API this topic builds async patterns on top of
- **85 — Mocking and stubbing** — `jest.mock()`, `jest.spyOn()`, mocking HTTP calls with `nock`; used to fake the async dependencies (like `dbConnection`) shown in Example 2
- **86 — Testing Express routes** — `supertest` tests real HTTP endpoints, which are themselves async handlers — the exact rejection/resolution patterns from this topic apply directly there
