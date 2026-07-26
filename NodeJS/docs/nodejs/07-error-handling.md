# 07 — Errors and error handling in Node

## What is this?

Error handling is how a program reacts when something goes wrong — a file doesn't exist, a database connection times out, a user sends bad input. Node.js gives you several mechanisms for this: the built-in `Error` object and its subtypes, the "error-first callback" convention used throughout Node's core APIs, `try/catch` for synchronous and `async/await` code, and two special process-level events (`uncaughtException` and `unhandledRejection`) that catch anything that slips through. Think of it like a building's safety systems — smoke detectors (try/catch) handle small fires locally, but a building-wide alarm (`uncaughtException`) exists for the fire that nobody caught in time.

## Why does it matter for backend development?

A backend server runs for weeks or months without restarting, serving thousands of requests. If one request throws an error and nothing catches it, a naive server can crash the *entire process* — taking down every other user's request with it. Backend developers must handle errors at every layer: database calls fail, external APIs time out, user input is malformed, files are missing. Knowing exactly *which* mechanism catches *which* kind of error (sync vs async, callback vs promise) is the difference between a server that degrades gracefully and one that crashes at 3 AM because one user sent a malformed JSON body.

---

## Syntax / API

```js
// ── 1. Built-in Error types ─────────────────────────────────────────────────
const err1 = new Error('Something went wrong');       // generic base error
const err2 = new TypeError('Expected a string');       // wrong type used
const err3 = new RangeError('Value out of bounds');    // number/array out of range
const err4 = new SyntaxError('Invalid syntax');        // malformed code/JSON

// ── 2. Custom error classes (extend Error) ──────────────────────────────────
class ValidationError extends Error {
  constructor(message) {
    super(message);                    // call the parent Error constructor
    this.name = 'ValidationError';     // override the default "Error" name
    this.statusCode = 400;             // attach extra context (HTTP status)
  }
}

// ── 3. Error-first callback convention (Node core style) ────────────────────
function readUserFile(filePath, callback) {
  // callback(error, result) — error is null on success, populated on failure
  fs.readFile(filePath, 'utf8', (err, data) => {
    if (err) return callback(err);     // pass the error, stop here
    callback(null, data);              // null error means success
  });
}

// ── 4. try/catch with synchronous code ──────────────────────────────────────
try {
  const parsed = JSON.parse('{ invalid json');  // throws SyntaxError
} catch (err) {
  console.error('Parse failed:', err.message);
}

// ── 5. try/catch with async/await ───────────────────────────────────────────
async function fetchUser(userId) {
  try {
    const user = await dbConnection.query('SELECT * FROM users WHERE id = $1', [userId]);
    return user;
  } catch (err) {
    console.error('Query failed:', err.message);  // catches rejected promises too
    throw err;                                     // re-throw so caller knows it failed
  }
}

// ── 6. Process-level safety nets (last resort only) ─────────────────────────
process.on('uncaughtException', (err) => {
  console.error('Uncaught exception:', err);   // an error thrown with no try/catch
  process.exit(1);                             // Node docs recommend exiting after this
});

process.on('unhandledRejection', (reason) => {
  console.error('Unhandled rejection:', reason); // a promise rejected with no .catch
  process.exit(1);
});
```

---

## How it works — line by line

- `new Error('message')` creates an object with a `.message`, `.name` ("Error"), and `.stack` (the call trace showing where it happened).
- `TypeError` and `RangeError` are subclasses of `Error` — they behave the same but have a more specific `.name`, useful for `if (err instanceof TypeError)` checks.
- A custom class like `ValidationError` calls `super(message)` first (required — it sets up `.message` and `.stack`), then adds your own fields like `statusCode` so the code that catches it knows how to respond.
- The **error-first callback** pattern is a convention, not a language feature: every Node core async function calls your callback with `(error, result)` — if `error` is not `null`, something failed and `result` should be ignored.
- `try { ... } catch (err) { ... }` only catches errors thrown **synchronously** inside the `try` block. If you put an async callback inside `try`, and that callback throws later, `catch` will **not** see it — the error escapes.
- Marking a function `async` and using `await` changes this: a rejected promise inside an `async` function behaves like a thrown error, so `try/catch` **does** catch it. This is why `async/await` is preferred over raw `.then()/.catch()` chains for backend code.
- `process.on('uncaughtException', ...)` fires when an `Error` is thrown anywhere and nothing in the call stack catches it. This is Node's last line of defense — by this point the process is in an unknown state, so the standard advice is to log it and `process.exit(1)` rather than keep running.
- `process.on('unhandledRejection', ...)` fires when a `Promise` is rejected and no `.catch()` or `try/catch` ever handles it. As of modern Node, an unhandled rejection that reaches this handler with no listener will actually crash the process by default — so always attach a handler.

---

## Example 1 — basic

```js
// File: src/utils/errorBasics.js

// ── Custom error class with extra context ───────────────────────────────────
class NotFoundError extends Error {
  constructor(resource) {
    super(`${resource} not found`);   // build a readable message
    this.name = 'NotFoundError';       // identify the error type
    this.statusCode = 404;             // attach an HTTP status for later use
  }
}

// A function that may throw a custom error
function findUserById(userId, users) {
  const user = users.find((u) => u.id === userId);  // search the array
  if (!user) {
    throw new NotFoundError(`User with id ${userId}`);  // throw if missing
  }
  return user;   // return the user if found
}

const users = [{ id: 1, name: 'Aditi' }];  // sample data

// ── try/catch handles the throw synchronously ───────────────────────────────
try {
  const user = findUserById(2, users);   // this id doesn't exist → throws
  console.log('Found user:', user);      // never reached
} catch (err) {
  console.log('Error name  :', err.name);        // 'NotFoundError'
  console.log('Error message:', err.message);    // 'User with id 2 not found'
  console.log('Status code :', err.statusCode);  // 404
}

// ── instanceof lets you branch on error type ────────────────────────────────
try {
  findUserById(2, users);
} catch (err) {
  if (err instanceof NotFoundError) {
    console.log('Handled as a 404 case');   // specific handling for this type
  } else {
    console.log('Unknown error, rethrowing');
    throw err;                              // let unexpected errors propagate
  }
}
```

---

## Example 2 — real world backend use case

```js
// File: src/services/userService.js
// A realistic service function showing error-first callbacks, custom errors,
// and async/await error handling working together.

const fs = require('fs/promises');   // promise-based fs API

// ── Custom error hierarchy for the whole app ────────────────────────────────
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.name = 'AppError';
    this.statusCode = statusCode;
    this.isOperational = true;   // marks this as an "expected" error, not a bug
  }
}

class AuthError extends AppError {
  constructor(message = 'Invalid or expired token') {
    super(message, 401);          // 401 Unauthorized
    this.name = 'AuthError';
  }
}

// ── Legacy-style error-first callback (mimicking a third-party library) ────
function verifyAuthToken(authToken, callback) {
  if (!authToken) {
    return callback(new AuthError('No token provided'));  // error, no result
  }
  const isValid = authToken.startsWith('valid_');           // fake check
  if (!isValid) {
    return callback(new AuthError());                       // default message
  }
  callback(null, { userId: 'user_42' });                     // success, no error
}

// ── async/await wrapper around the callback-based function ─────────────────
function verifyAuthTokenAsync(authToken) {
  return new Promise((resolve, reject) => {
    verifyAuthToken(authToken, (err, session) => {
      if (err) return reject(err);   // reject the promise on error
      resolve(session);              // resolve with the result
    });
  });
}

// ── Main service function used by a route handler ───────────────────────────
async function loadUserProfile(authToken, filePath) {
  try {
    // Step 1: verify the token (converted callback → promise → awaited)
    const session = await verifyAuthTokenAsync(authToken);

    // Step 2: read the user's profile file from disk
    const raw = await fs.readFile(filePath, 'utf8');   // rejects if file missing
    const profile = JSON.parse(raw);                    // throws SyntaxError if malformed

    return { ...profile, userId: session.userId };      // merge and return
  } catch (err) {
    // Any failure above (auth, missing file, bad JSON) lands here
    if (err instanceof AppError) {
      throw err;                       // known, operational error — pass through
    }
    // Unknown error (e.g. corrupt JSON) — wrap it so callers get a consistent shape
    throw new AppError(`Failed to load profile: ${err.message}`, 500);
  }
}

module.exports = { AppError, AuthError, loadUserProfile };

// Usage in an Express route (conceptual — Express covered in Topic 53):
// app.get('/profile', async (req, res) => {
//   try {
//     const profile = await loadUserProfile(req.headers.authorization, filePath);
//     res.json(profile);
//   } catch (err) {
//     res.status(err.statusCode || 500).json({ error: err.message });
//   }
// });
```

---

## Common mistakes

### Mistake 1 — Using try/catch around a callback-based async call

```js
// ❌ WRONG — try/catch cannot catch errors from a callback fired later
const fs = require('fs');

try {
  fs.readFile('./config.json', 'utf8', (err, data) => {
    if (err) throw err;   // this throw happens INSIDE the event loop callback,
                           // long after try/catch has already finished running
  });
} catch (err) {
  console.log('This will NEVER run for the error above');
}

// ✅ CORRECT — handle the error inside the callback itself
fs.readFile('./config.json', 'utf8', (err, data) => {
  if (err) {
    console.error('Failed to read config:', err.message);  // handled right here
    return;
  }
  console.log('Config loaded:', data);
});
```

### Mistake 2 — Forgetting to catch a rejected promise (silent failure or crash)

```js
// ❌ WRONG — no .catch() and no try/catch, the rejection is "unhandled"
async function chargeCustomer(sessionId, amount) {
  const result = await paymentGateway.charge(sessionId, amount);  // may reject
  return result;
}
chargeCustomer('sess_123', 4999);   // rejection here crashes the process (Node 15+)

// ✅ CORRECT — always attach a .catch() or wrap the call in try/catch
async function chargeCustomer(sessionId, amount) {
  try {
    return await paymentGateway.charge(sessionId, amount);
  } catch (err) {
    console.error('Payment failed:', err.message);   // handled locally
    throw err;                                        // let the caller decide what's next
  }
}
chargeCustomer('sess_123', 4999).catch((err) => {
  console.error('Charge attempt failed:', err.message);  // final safety net
});
```

### Mistake 3 — Relying on uncaughtException/unhandledRejection as normal error handling

```js
// ❌ WRONG — using process-level handlers instead of handling errors where they happen
process.on('uncaughtException', (err) => {
  console.log('Something broke somewhere:', err.message);
  // process keeps running in an unknown/corrupted state — very risky in production
});

app.get('/users/:userId', (req, res) => {
  const user = db.findUser(req.params.userId);   // if this throws, we rely on the global handler
  res.json(user);                                  // request never gets a proper response
});

// ✅ CORRECT — handle errors locally at the route level; use uncaughtException
// only as a last-resort safety net that logs and shuts down cleanly
app.get('/users/:userId', (req, res) => {
  try {
    const user = db.findUser(req.params.userId);
    res.json(user);                          // request completes normally
  } catch (err) {
    res.status(500).json({ error: 'Failed to fetch user' });  // proper response
  }
});

process.on('uncaughtException', (err) => {
  console.error('FATAL — uncaught exception:', err);
  process.exit(1);   // exit cleanly rather than continue in a broken state
});
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Defines a custom error class `InvalidInputError` extending `Error`, which sets `this.name = 'InvalidInputError'` and accepts a `fieldName` in its constructor, storing it as `this.fieldName`.
2. Writes a function `validateAge(age)` that throws an `InvalidInputError` if `age` is not a number, or if it is less than 0 or greater than 120.
3. Calls `validateAge` inside a `try/catch` with three different values: `25`, `-5`, and `'abc'`.
4. Logs the error's `name`, `message`, and `fieldName` for each failing case.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a small module that simulates fetching a record from a database using error-first callbacks:
1. Write `findRecordById(recordId, callback)` where `callback(error, record)` — it should call back with an error if `recordId` is not a positive integer, and otherwise call back with a fake record object like `{ id: recordId, name: 'Sample' }`.
2. Write an async wrapper `findRecordByIdAsync(recordId)` that wraps the callback function in a `Promise` and can be used with `await`.
3. Write an `async function getRecordSafely(recordId)` that calls `findRecordByIdAsync`, catches any error, and returns either `{ success: true, data: record }` or `{ success: false, error: err.message }` — it should never throw.
4. Test it by calling `getRecordSafely` with a valid id, `0`, and `-1`, logging each result.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a mini request-handling simulator that demonstrates full error handling in one flow:
1. Create a custom error hierarchy: a base `AppError` (with `statusCode` and `isOperational`), and two subclasses — `ValidationError` (400) and `DatabaseError` (500, `isOperational = false`).
2. Write an async function `processOrder(orderPayload)` that:
   - Throws a `ValidationError` if `orderPayload.userId` or `orderPayload.amount` is missing.
   - Throws a `ValidationError` if `orderPayload.amount` is not a positive number.
   - Simulates a database write with a function that randomly (using `Math.random() < 0.3`) rejects with a `DatabaseError('Connection lost')`, otherwise resolves with `{ orderId: 'order_' + Date.now() }`.
3. Write a `handleRequest(orderPayload)` function that calls `processOrder`, catches any error, and logs a response object shaped like `{ status: <statusCode>, body: { error: <message> } }` on failure, or `{ status: 201, body: <result> }` on success.
4. Add a top-level `process.on('unhandledRejection', ...)` handler that logs any promise rejection your code fails to catch, so you can verify your error handling has no gaps.
5. Call `handleRequest` multiple times with valid and invalid payloads to observe all the branches.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
BUILT-IN ERROR TYPES
  Error          → generic base error, all custom errors should extend this
  TypeError      → wrong type passed (e.g. calling a method on undefined)
  RangeError     → number/array outside allowed range
  SyntaxError    → malformed code or JSON.parse() on invalid JSON
  ReferenceError → referencing an undeclared variable

CUSTOM ERROR PATTERN
  class MyError extends Error {
    constructor(message) {
      super(message);        // required — sets .message and .stack
      this.name = 'MyError'; // override default "Error" name
      this.statusCode = 400; // attach app-specific context
    }
  }

ERROR-FIRST CALLBACK (Node core convention)
  function doThing(input, callback) {
    callback(err, result);   // err is null on success, populated on failure
  }
  ALWAYS check `if (err)` first, before touching `result`

WHAT try/catch CATCHES
  ✓ synchronous throws                    try { JSON.parse(bad) } catch {}
  ✓ rejected promises inside async/await  try { await mayReject() } catch {}
  ✗ throws inside a plain callback        fs.readFile(p, (err) => { throw err })
  ✗ errors in setTimeout/setInterval bodies

PROCESS-LEVEL SAFETY NETS (last resort only)
  process.on('uncaughtException', (err) => { log(err); process.exit(1); });
  process.on('unhandledRejection', (reason) => { log(reason); process.exit(1); });
  Rule of thumb: log and exit — do NOT try to "recover" and keep serving traffic

GOLDEN RULES
  - Handle errors as close to where they happen as possible
  - Never swallow an error silently (empty catch block)
  - Distinguish operational errors (bad input, network) from programmer bugs
  - Always attach .catch() or try/catch to every promise you create
  - uncaughtException/unhandledRejection are safety nets, not your primary strategy
```

---

## Connected topics

- **45 — Callback pattern deep dive** — goes deeper into error-first callbacks, callback hell, and how Node core APIs are built around this convention
- **61 — Operational vs programmer errors** — the classification (`isOperational`) introduced here becomes the foundation for deciding what to recover from vs what should crash the process
- **62 — Centralized error handling** — shows how the `AppError` pattern from this doc becomes a single Express error-handling middleware instead of repeated try/catch blocks in every route
