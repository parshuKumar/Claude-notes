# 88 — Test coverage

## What is this?

Test coverage is a measurement of how much of your source code actually gets **executed** while your test suite runs. Tools like **Istanbul** (the engine that powers `nyc` and is built directly into Jest) instrument your code — quietly inserting counters around every statement, branch, and function — then report which ones were hit and which were never touched. Think of it like a delivery driver's GPS trail overlaid on a map of a city: it shows you exactly which streets the driver actually drove down, but it says nothing about whether the packages were delivered to the right house.

## Why does it matter for backend development?

Backend teams use coverage numbers as a **quality gate** in CI — a pull request that drops coverage below a threshold (say 80%) gets blocked automatically, forcing developers to write tests for new code before it merges. Coverage reports are also a map of blind spots: they show you the `catch` block for a database timeout that no test ever triggers, or the `else` branch of a validation function that always takes the `if` path in your tests. Used correctly, coverage catches "I forgot to test this" mistakes early. Used wrongly — chasing a percentage instead of testing behavior — it gives false confidence while real bugs slip through untested-but-"covered" code.

---

## Syntax / API

```js
// ── package.json — Jest has coverage built in, no extra library needed ──────
{
  "scripts": {
    // Runs all tests and prints/generates a coverage report
    "test:coverage": "jest --coverage"
  },
  "jest": {
    // Which report formats to generate
    "coverageReporters": ["text", "html", "lcov", "json-summary"],
    // Only measure coverage for files matching these globs
    "collectCoverageFrom": ["src/**/*.js", "!src/**/*.test.js"],
    // Where to write the HTML/lcov/json report files
    "coverageDirectory": "coverage",
    // Fail the whole test run if these minimums aren't met
    "coverageThreshold": {
      "global": {
        "branches": 75,   // % of if/else, ternary, switch paths executed
        "functions": 80,  // % of functions called at least once
        "lines": 80,      // % of executable lines run
        "statements": 80  // % of statements run (close to lines, not identical)
      }
    }
  }
}
```

```js
// ── Equivalent setup using nyc (Istanbul's CLI) for non-Jest test runners ───
// npm install --save-dev nyc mocha

// .nycrc.json — nyc's config file (used with Mocha, Tape, ava, etc.)
{
  "reporter": ["text", "html", "lcov"],   // report formats to output
  "check-coverage": true,                  // enforce thresholds, fail if not met
  "branches": 75,                          // minimum % branch coverage
  "functions": 80,                         // minimum % function coverage
  "lines": 80,                             // minimum % line coverage
  "statements": 80,                        // minimum % statement coverage
  "exclude": ["**/*.test.js", "coverage/**"] // files nyc should not instrument
}

// package.json script — nyc wraps the test command and instruments it
// "test:coverage": "nyc mocha src/**/*.test.js"
```

---

## How it works — line by line

1. **Instrumentation** — before running your tests, Istanbul rewrites your source files in memory, inserting tiny counters next to every statement, branch, and function declaration (your original files on disk are never touched).
2. **Execution** — your test suite runs normally against the instrumented code; every time a counter is passed, it increments.
3. **Collection** — after all tests finish, Istanbul reads every counter and knows exactly which lines, branches, and functions were hit at least once and which were never reached.
4. **Reporting** — the counts are turned into a report: `text` prints a table to your terminal, `html` produces a browsable `coverage/index.html` you can click through file-by-file, `lcov` produces a machine-readable format most CI dashboards (Codecov, Coveralls) consume.
5. **Threshold check** — if you configured minimums (`coverageThreshold` in Jest, `check-coverage` in nyc), the tool compares actual percentages against your minimums and **exits with a non-zero code** if any metric falls short — this is what makes CI fail the build.

The four numbers you'll see in every report:

```
% Stmts     → % of executable statements that ran at least once
% Branch    → % of if/else, ternary, &&/||, switch-case paths that ran
% Funcs     → % of declared functions that were called at least once
% Lines     → % of physical lines that ran (very close to % Stmts)
```

`% Branch` is the one developers forget about most — a function can show 100% line coverage while its `else` branch was never tested, because both branches share the same physical line in a ternary or short-circuit expression.

---

## Example 1 — basic

```js
// File: src/utils/passwordValidator.js
// A small function with multiple branches — a good candidate to demonstrate coverage gaps

function isPasswordValid(password) {
  // Branch 1: reject if password is missing or not a string
  if (typeof password !== 'string') {
    return false;
  }

  // Branch 2: reject if too short
  if (password.length < 8) {
    return false;
  }

  // Branch 3: reject if it has no digit
  const hasDigit = /\d/.test(password);
  if (!hasDigit) {
    return false;
  }

  // If all checks pass, the password is valid
  return true;
}

module.exports = { isPasswordValid };
```

```js
// File: src/utils/passwordValidator.test.js
// Jest test file — note we only test the "valid" and "too short" paths on purpose,
// to demonstrate what an incomplete coverage report looks like

const { isPasswordValid } = require('./passwordValidator');

describe('isPasswordValid', () => {
  test('returns true for a valid password', () => {
    expect(isPasswordValid('secret123')).toBe(true); // hits the "return true" line
  });

  test('returns false when password is too short', () => {
    expect(isPasswordValid('abc1')).toBe(false); // hits Branch 2's "return false"
  });

  // Branch 1 (non-string input) and Branch 3 (no digit) are NEVER tested here —
  // running `npx jest --coverage` will show this file below 100% branch coverage
});
```

```
# Run: npx jest --coverage src/utils/passwordValidator.test.js

--------------------|---------|----------|---------|---------|-------------------
File                | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
--------------------|---------|----------|---------|---------|-------------------
passwordValidator.js|   88.88 |       50 |     100 |   88.88 | 6,15
--------------------|---------|----------|---------|---------|-------------------
# Lines 6 and 15 (the non-string check and the no-digit check) never ran
```

---

## Example 2 — real world backend use case

```js
// File: src/services/orderService.js
// A backend service function with several real branches — payment status handling,
// stock checks, and a discount calculation. This is the kind of code coverage
// gates exist to protect in production.

function calculateOrderTotal(items, discountCode) {
  // Guard: reject empty carts before any math happens
  if (!Array.isArray(items) || items.length === 0) {
    throw new Error('Cannot calculate total for an empty cart');
  }

  // Sum up item price * quantity for every line item
  const subtotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0);

  // Apply a 10% discount only if a valid promo code was passed
  const discount = discountCode === 'SAVE10' ? subtotal * 0.1 : 0;

  // Final total after discount, rounded to 2 decimal places for currency safety
  return Math.round((subtotal - discount) * 100) / 100;
}

module.exports = { calculateOrderTotal };
```

```js
// File: src/services/orderService.test.js
// A thorough test suite — written specifically to hit EVERY branch, not just
// to make the "happy path" pass. This is what a healthy coverage-driven test
// suite looks like for backend logic.

const { calculateOrderTotal } = require('./orderService');

describe('calculateOrderTotal', () => {
  test('throws on an empty cart', () => {
    // Covers the guard-clause branch (items.length === 0)
    expect(() => calculateOrderTotal([], null)).toThrow('Cannot calculate total for an empty cart');
  });

  test('throws when items is not an array', () => {
    // Covers the !Array.isArray(items) branch specifically
    expect(() => calculateOrderTotal(null, null)).toThrow();
  });

  test('calculates subtotal with no discount code', () => {
    const items = [{ price: 100, quantity: 2 }]; // 2 units at 100 each
    // Covers the ternary's FALSE branch — no discount applied
    expect(calculateOrderTotal(items, null)).toBe(200);
  });

  test('applies 10% discount with a valid promo code', () => {
    const items = [{ price: 100, quantity: 2 }]; // subtotal = 200
    // Covers the ternary's TRUE branch — discount applied
    expect(calculateOrderTotal(items, 'SAVE10')).toBe(180);
  });
});
```

```json
// package.json — enforcing this specific file at a stricter threshold than the rest
// of the codebase, since order totals directly affect billing correctness
{
  "jest": {
    "coverageThreshold": {
      "global": { "branches": 75, "functions": 80, "lines": 80, "statements": 80 },
      "./src/services/orderService.js": {
        "branches": 100,   // billing logic must have every branch tested
        "functions": 100,
        "lines": 100,
        "statements": 100
      }
    }
  }
}
```

---

## Common mistakes

### Mistake 1 — Chasing 100% coverage by testing implementation instead of behavior

```js
// ❌ WRONG — this test only exists to "touch" every line for the coverage report;
// it doesn't verify anything meaningful about correct behavior
test('calls the reduce function', () => {
  const spy = jest.spyOn(Array.prototype, 'reduce');
  calculateOrderTotal([{ price: 10, quantity: 1 }], null);
  expect(spy).toHaveBeenCalled(); // coverage goes up, but nothing about correctness is proven
});

// ✅ CORRECT — test the actual output for a given input; coverage rises as a
// side effect of testing real behavior, not as the goal itself
test('returns 10 for a single item priced at 10', () => {
  expect(calculateOrderTotal([{ price: 10, quantity: 1 }], null)).toBe(10);
});
```

### Mistake 2 — Ignoring branch coverage and only watching the line/statement percentage

```js
// ❌ WRONG — this single test makes "% Lines" look great (both branches share
// one physical line) while the ELSE path of the ternary is never verified
function getShippingFee(orderTotal) {
  return orderTotal > 50 ? 0 : 5.99; // one line, but TWO branches
}

test('free shipping over 50', () => {
  expect(getShippingFee(100)).toBe(0); // only exercises the TRUE branch
});
// Report shows 100% lines, but only 50% branches — easy to miss if you only skim "% Lines"

// ✅ CORRECT — test both branches explicitly, and configure Jest/nyc to fail
// the build on low BRANCH coverage, not just line coverage
test('charges 5.99 for orders under 50', () => {
  expect(getShippingFee(30)).toBe(5.99); // exercises the FALSE branch too
});
// coverageThreshold: { branches: 80, ... } catches this gap automatically in CI
```

### Mistake 3 — Treating "100% covered" as proof the code is correct

```js
// ❌ WRONG — this mock makes the database-error branch "run" during the test,
// so coverage marks it green, but the test never proves the ACTUAL database
// driver produces an error shaped like this, or that the response is correct
async function getUserById(userId, dbConnection) {
  try {
    return await dbConnection.query('SELECT * FROM users WHERE id = $1', [userId]);
  } catch (error) {
    return { error: 'Database unavailable' }; // marked "covered" below
  }
}

test('handles db errors', async () => {
  const fakeDb = { query: () => { throw new Error('anything'); } }; // fabricated error
  const result = await getUserById(1, fakeDb);
  expect(result.error).toBe('Database unavailable');
  // 100% coverage on the catch block — but this never proves real pg driver
  // timeouts, connection resets, or constraint violations are handled correctly
});

// ✅ CORRECT — treat coverage as "this code ran", and pair it with integration
// tests (Topic 87) against a real test database to prove behavior is actually correct
// coverage tells you WHERE to look; integration/e2e tests tell you WHAT is true
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `getDiscountTier(totalSpent)` that returns `'gold'` if `totalSpent >= 1000`, `'silver'` if `totalSpent >= 500`, or `'none'` otherwise. Write a Jest test file that achieves **100% branch coverage** for this function (you will need at least 3 test cases — one per branch). Run `npx jest --coverage` and confirm the terminal report shows 100% for branches, lines, and functions on this file.

```js
// Write your code here
```

---

### Exercise 2 — medium

You are given this existing (untested) module:

```js
function applyLateFee(invoiceAmount, daysOverdue) {
  if (daysOverdue <= 0) return invoiceAmount;
  if (daysOverdue <= 30) return invoiceAmount * 1.05;
  return invoiceAmount * 1.15;
}
module.exports = { applyLateFee };
```

1. Set up a Jest `coverageThreshold` in `package.json` requiring 100% branches, functions, lines, and statements for this specific file (use the per-file threshold syntax shown in Example 2).
2. Write only enough tests to hit two of the three branches, run `npx jest --coverage`, and confirm the build **fails** because the threshold isn't met.
3. Add the missing test case, re-run, and confirm the build now **passes**.
4. Generate an HTML report (`coverageReporters: ["html"]`) and note which uncovered line number appeared before your final fix.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a Node.js script `checkCoverageGate.js` that enforces **different coverage thresholds for different folders** by reading Istanbul's `coverage/coverage-summary.json` output directly (this is the pattern real CI pipelines use when Jest's built-in per-file threshold syntax isn't flexible enough). Requirements:

1. Run `jest --coverage --coverageReporters=json-summary` first so `coverage/coverage-summary.json` exists.
2. Your script reads that JSON file with `fs.readFileSync` and `JSON.parse`.
3. Define per-directory rules, e.g.: files under `src/controllers/` must have `lines >= 90`; files under `src/utils/` must have `lines >= 100`; everything else must have `lines >= 75`.
4. For every file in the summary, check its `lines.pct` against the matching rule based on its path.
5. Print a clear pass/fail table to the console listing each file, its actual percentage, its required minimum, and PASS/FAIL.
6. Call `process.exit(1)` if any file fails its rule, `process.exit(0)` if everything passes — so it can be wired into a CI pipeline (Topic 108) as a required check.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT COVERAGE MEASURES (Istanbul / nyc / Jest's built-in coverage)
  % Stmts   → executable statements that ran at least once
  % Branch  → if/else, ternary, &&/||, switch paths that ran
  % Funcs   → declared functions called at least once
  % Lines   → physical source lines executed (close to % Stmts)

RUNNING COVERAGE
  jest --coverage                          → Jest, built in, no extra install
  nyc mocha src/**/*.test.js               → nyc wraps any test runner
  jest --coverage --coverageReporters=json-summary  → machine-readable output

REPORT FORMATS
  text   → printed straight to terminal
  html   → coverage/index.html, click through file by file
  lcov   → coverage/lcov.info, consumed by Codecov/Coveralls
  json-summary → coverage/coverage-summary.json, parse in scripts/CI gates

ENFORCING THRESHOLDS
  Jest:  "coverageThreshold": { "global": {...} }         → fails build if below
  Jest:  "coverageThreshold": { "./src/file.js": {...} }  → per-file override
  nyc:   "check-coverage": true + branches/lines/etc keys → same idea

IGNORING LINES INTENTIONALLY (use sparingly, always explain why)
  /* istanbul ignore next */         → skip the next block/statement
  /* istanbul ignore else */         → skip only the else branch
  /* istanbul ignore file */         → skip an entire file (e.g. generated code)

WHAT COVERAGE DOES TELL YOU
  - Which lines/branches your tests never executed at all
  - Dead code and forgotten error-handling paths
  - A useful CI gate to stop obviously untested code merging

WHAT COVERAGE DOES NOT TELL YOU
  - Whether your assertions actually check the right thing
  - Whether mocked dependencies behave like the real ones
  - Whether the code is logically correct — only that it RAN
  - Whether concurrent/race conditions or real network failures are handled
  100% coverage ≠ 0 bugs. It only means every line executed at least once.

COMMON THRESHOLD STARTING POINT
  New projects:        70-80% global, ratchet up over time
  Billing/auth/payments: 100% on those specific files, always
```

---

## Connected topics

- **83 — Jest fundamentals** — coverage is built directly into Jest via the `--coverage` flag; you need `describe`/`it`/`expect` fluency before coverage numbers mean anything.
- **86 — Testing Express routes** — supertest-driven route tests are exactly what fills in the branch coverage gaps for controllers and middleware in a real API.
- **108 — CI/CD basics with GitHub Actions** — coverage thresholds are almost always wired into a CI workflow so a pull request that drops coverage fails the build automatically.
