# 83 — Jest fundamentals

## What is this?

Jest is a JavaScript testing framework — it gives you the tools to write, organize, and run automated checks that verify your code behaves the way you expect. Think of it like a strict quality-control inspector on a factory line: instead of a human manually checking every product, the inspector runs the same checklist automatically, every single time, and immediately flags anything that doesn't match spec. `describe`, `it`/`test`, and `expect` are the vocabulary Jest gives you to write that checklist in plain, readable code.

---

## Why does it matter for backend development?

Backend code runs unattended, in production, handling other people's money, data, and requests — you cannot manually click through every code path before every deploy. Jest lets you encode "this function must behave correctly" as a script that runs in seconds, every time you push code, in CI/CD pipelines. A backend dev uses Jest to verify business logic (price calculations, permission checks, data transformations), catch regressions before they reach production, and safely refactor code without fear of silently breaking something. It is the single most common test runner in the Node.js ecosystem, and nearly every Express/Fastify job posting expects it.

---

## Syntax / API

```js
// Import Jest globals explicitly is NOT required — Jest injects them automatically
// (describe, test, it, expect, beforeEach, afterEach, beforeAll, afterAll)
// but this doc shows them for clarity of where each comes from.

// describe() groups related tests into a labeled block — purely organizational
describe('calculateOrderTotal', () => {

  // beforeAll runs ONCE before all tests in this describe block
  beforeAll(() => {
    console.log('Setting up test suite for order calculations');
  });

  // beforeEach runs before EVERY single test in this describe block
  beforeEach(() => {
    // fresh state for each test — prevents one test's leftovers affecting another
    global.taxRate = 0.08;
  });

  // afterEach runs after EVERY single test — good for cleanup (closing DB connections, clearing mocks)
  afterEach(() => {
    jest.clearAllMocks(); // reset any mock call history between tests
  });

  // afterAll runs ONCE after all tests in this block finish
  afterAll(() => {
    console.log('Order calculation suite finished');
  });

  // test() / it() are aliases — both define a single test case. "it" reads like a sentence.
  it('adds tax to the subtotal correctly', () => {
    const subtotal = 100; // input value for this test case

    const total = subtotal + subtotal * global.taxRate; // function under test (inlined here)

    // expect() wraps the actual value; a matcher method checks it against the expected value
    expect(total).toBe(108); // toBe checks exact equality (===) — best for primitives
  });

  test('returns 0 for an empty cart', () => {
    const subtotal = 0; // edge case: no items
    const total = subtotal + subtotal * global.taxRate;
    expect(total).toBe(0); // exact match expected
  });
});
```

---

## How it works — line by line

- `describe('name', fn)` does not run a test itself — it just groups tests under a shared label so failures are easy to locate in the output (e.g. "calculateOrderTotal › adds tax correctly").
- `it('description', fn)` and `test('description', fn)` are the actual test cases. Jest runs the function inside; if it throws (or an `expect` fails), the test is marked failed. If it completes without throwing, the test passes.
- `expect(value)` wraps the actual result you got from your code, returning an object with matcher methods attached to it.
- A matcher like `.toBe(108)` is called on that wrapped value — it compares the actual value to the expected value and throws a descriptive error if they don't match.
- `beforeEach` / `afterEach` are lifecycle hooks that Jest runs automatically around every test in the same `describe` block (or file, if placed outside any `describe`) — this is where you set up fresh test data and tear down side effects (open connections, timers, mocks).
- `beforeAll` / `afterAll` run only once, wrapping the entire block — useful for expensive setup like opening a single database connection shared by many tests.
- Jest discovers test files automatically by naming convention — any file ending in `.test.js` or `.spec.js`, or any file inside a `__tests__` folder, is treated as a test file and run when you execute `npx jest`.
- Each test file runs in its own isolated environment by default, so global state set in one file never leaks into another.

---

## Example 1 — basic

```js
// File: src/utils/discount.js
// A plain function we want to test — no framework dependency

function applyDiscount(price, discountPercent) {
  // guard against invalid discount values before calculating
  if (discountPercent < 0 || discountPercent > 100) {
    throw new Error('discountPercent must be between 0 and 100');
  }
  return price - price * (discountPercent / 100); // apply the percentage discount
}

module.exports = { applyDiscount }; // export for the test file to require
```

```js
// File: src/utils/discount.test.js
// Jest automatically finds this file because it ends in .test.js

const { applyDiscount } = require('./discount'); // import the function under test

describe('applyDiscount', () => {
  // basic case: normal discount applied correctly
  it('reduces price by the given percentage', () => {
    const result = applyDiscount(200, 10); // 200 with a 10% discount
    expect(result).toBe(180); // 200 - 20 = 180
  });

  // edge case: 0% discount should return the original price unchanged
  it('returns original price when discount is 0', () => {
    const result = applyDiscount(150, 0);
    expect(result).toBe(150);
  });

  // edge case: 100% discount should make the item free
  it('returns 0 when discount is 100', () => {
    const result = applyDiscount(150, 100);
    expect(result).toBe(0);
  });

  // error case: invalid input should throw, not silently produce a wrong number
  it('throws for a discount above 100', () => {
    // toThrow checks that calling the function raises an error
    expect(() => applyDiscount(150, 150)).toThrow('discountPercent must be between 0 and 100');
  });
});

// Run with: npx jest src/utils/discount.test.js
// Or run the whole suite with: npx jest
```

---

## Example 2 — real world backend use case

```js
// File: src/services/userService.js
// Service layer function that validates a signup payload before saving to the DB
// (in a real app this would also call a repository/DB layer — mocked out for now)

function validateSignupPayload(requestBody) {
  const errors = []; // collect all validation errors instead of failing on the first one

  if (!requestBody.email || !requestBody.email.includes('@')) {
    errors.push('A valid email is required'); // basic email shape check
  }

  if (!requestBody.password || requestBody.password.length < 8) {
    errors.push('Password must be at least 8 characters'); // minimum password strength
  }

  if (!requestBody.username || requestBody.username.trim() === '') {
    errors.push('Username is required'); // required field check
  }

  return {
    isValid: errors.length === 0, // true only if no errors were collected
    errors,                        // array of human-readable error messages
  };
}

module.exports = { validateSignupPayload };
```

```js
// File: src/services/userService.test.js
// Realistic test suite for a signup validation function — the kind a backend dev writes daily

const { validateSignupPayload } = require('./userService');

describe('validateSignupPayload', () => {
  // shared valid payload reused across tests, mutated per test case
  let requestBody;

  // fresh valid payload before every test — avoids one test's mutation leaking into another
  beforeEach(() => {
    requestBody = {
      email: 'dev@example.com',
      password: 'supersecret123',
      username: 'backend_dev',
    };
  });

  it('accepts a fully valid signup payload', () => {
    const result = validateSignupPayload(requestBody);
    expect(result.isValid).toBe(true);   // no errors expected
    expect(result.errors).toHaveLength(0); // toHaveLength checks array/string length
  });

  it('rejects a payload with a malformed email', () => {
    requestBody.email = 'not-an-email'; // corrupt only the email field
    const result = validateSignupPayload(requestBody);
    expect(result.isValid).toBe(false);
    // toContain checks the array includes this exact string
    expect(result.errors).toContain('A valid email is required');
  });

  it('rejects a payload with a short password', () => {
    requestBody.password = '123'; // too short
    const result = validateSignupPayload(requestBody);
    expect(result.errors).toContain('Password must be at least 8 characters');
  });

  it('collects multiple errors when several fields are invalid', () => {
    requestBody.email = 'bad-email';
    requestBody.password = 'short';
    requestBody.username = '';
    const result = validateSignupPayload(requestBody);
    expect(result.isValid).toBe(false);
    expect(result.errors).toHaveLength(3); // all three checks should fail together
  });

  it('matches the full expected error object shape', () => {
    requestBody.username = ''; // only username is invalid
    const result = validateSignupPayload(requestBody);
    // toEqual does a deep comparison of objects/arrays (unlike toBe, which checks reference identity)
    expect(result).toEqual({
      isValid: false,
      errors: ['Username is required'],
    });
  });
});
```

---

## Common mistakes

### Mistake 1 — Using `toBe` to compare objects or arrays

```js
// ❌ WRONG — toBe uses Object.is() (like ===), which checks reference identity
// Two different objects with identical contents are NOT the same reference
const userProfile = { userId: 42, role: 'admin' };
expect(userProfile).toBe({ userId: 42, role: 'admin' }); // fails! different object references

// ✅ CORRECT — toEqual does a recursive deep-value comparison
expect(userProfile).toEqual({ userId: 42, role: 'admin' }); // passes — values match
```

### Mistake 2 — Leaking state between tests by not resetting it

```js
// ❌ WRONG — a shared mutable array is modified by one test and read by the next,
// so test order accidentally affects results (a huge source of flaky tests)
const sessionQueue = [];

it('adds a session to the queue', () => {
  sessionQueue.push('sessionId_1');
  expect(sessionQueue).toHaveLength(1); // passes in isolation
});

it('queue starts empty for a new request', () => {
  expect(sessionQueue).toHaveLength(0); // FAILS — still holds sessionId_1 from the previous test
});

// ✅ CORRECT — reset shared state in beforeEach so every test starts clean
let sessionQueue2;

beforeEach(() => {
  sessionQueue2 = []; // fresh array before every single test
});

it('adds a session to the queue', () => {
  sessionQueue2.push('sessionId_1');
  expect(sessionQueue2).toHaveLength(1);
});

it('queue starts empty for a new request', () => {
  expect(sessionQueue2).toHaveLength(0); // passes — clean slate every time
});
```

### Mistake 3 — Forgetting that a test with no assertions always "passes"

```js
// ❌ WRONG — this test calls the function but never checks the result with expect()
// Jest reports it as PASSED even though nothing was actually verified
it('generates an auth token', () => {
  const authToken = generateAuthToken({ userId: 42 }); // called, but result ignored
  // no expect() call — this test can never fail, giving false confidence
});

// ✅ CORRECT — always assert on the actual output you care about
it('generates an auth token', () => {
  const authToken = generateAuthToken({ userId: 42 });
  expect(typeof authToken).toBe('string'); // check the shape/type
  expect(authToken.length).toBeGreaterThan(10); // check it's not empty/trivial
});
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `isValidApiKey(apiKey)` that returns `true` if the key is a non-empty string starting with the prefix `"sk_"` and at least 20 characters long, otherwise `false`. Save it in a file, then write a Jest test file for it with a `describe` block containing at least four `it` cases: a valid key, a key missing the `"sk_"` prefix, a key that's too short, and an empty string.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `paginate(items, pageNumber, pageSize)` that returns an object `{ data, currentPage, totalPages, hasNextPage }` representing one "page" of an array of items (simulating how a backend API paginates a list of database rows). Then write a Jest test suite using `beforeEach` to set up a shared array of 25 fake `userId` items, and test: the first page returns the correct slice, the last page returns fewer items and `hasNextPage: false`, requesting a page number beyond the data returns an empty `data` array, and `totalPages` is calculated correctly for different `pageSize` values.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small in-memory `RateLimiter` class with a constructor `(maxRequests, windowMs)` and a method `allow(userId)` that returns `true` if the given user is still under their request limit within the current time window, and `false` once they exceed it (tracking requests per `userId` internally, resetting counts after `windowMs` elapses). Write a full Jest test suite covering: a user under the limit is allowed, a user who exceeds `maxRequests` is denied, different `userId`s are tracked independently of each other, and — using `beforeEach`/`afterEach` with `jest.useFakeTimers()` and `jest.advanceTimersByTime()` — a denied user is allowed again after the time window resets.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE FUNCTIONS
  describe('label', fn)   → groups related tests, purely organizational, no assertions
  it('label', fn)         → a single test case (alias: test)
  expect(value)           → wraps the actual value so a matcher can check it

COMMON MATCHERS
  .toBe(x)                → exact equality (===) — use for numbers, strings, booleans
  .toEqual(x)              → deep value equality — use for objects and arrays
  .toStrictEqual(x)        → like toEqual but also checks undefined props and object types
  .toBeTruthy() / .toBeFalsy()   → checks truthy/falsy coercion
  .toBeNull() / .toBeUndefined()  → checks exact null / undefined
  .toContain(x)            → array/string includes x
  .toHaveLength(n)          → array/string has length n
  .toThrow(msg?)            → wrapped function call throws an error (optionally matching msg)
  .toBeGreaterThan(n) / .toBeLessThan(n) → numeric comparisons
  .not.toBe(x)              → negate any matcher by chaining .not before it

LIFECYCLE HOOKS (scoped to nearest describe, or whole file if outside one)
  beforeAll(fn)   → runs once, before all tests in the block
  beforeEach(fn)  → runs before every test in the block
  afterEach(fn)   → runs after every test — good for cleanup/reset
  afterAll(fn)    → runs once, after all tests in the block

TEST FILE DISCOVERY
  *.test.js  or  *.spec.js         → auto-discovered by Jest
  __tests__/ folder                → any file inside is auto-discovered
  Run all:        npx jest
  Run one file:   npx jest path/to/file.test.js
  Watch mode:     npx jest --watch
  Coverage:       npx jest --coverage

GOTCHAS
  toBe on objects/arrays        → compares reference, not value — use toEqual
  no expect() in a test          → test always "passes" silently, verify nothing
  shared mutable state           → reset it in beforeEach or tests leak into each other
  async test without await/done  → Jest may finish before your assertion even runs (see Topic 84)
```

---

## Connected topics

- **82 — Testing fundamentals** — the unit/integration/e2e concepts and test pyramid that Jest is used to implement
- **84 — Testing async code** — how to correctly test promises, `async`/`await`, and callbacks with Jest, avoiding the "test finishes before assertion runs" trap
- **85 — Mocking and stubbing** — `jest.mock()` and `jest.spyOn()` build directly on the `describe`/`it`/`expect` structure covered here to isolate units from their dependencies
