# 71 — Types of JavaScript Errors

---

## 1. What is this?

A JavaScript **error** is an object that represents something that went wrong at runtime. When an error is thrown and not caught, the browser logs it to the console and stops executing the current call stack. Node.js either crashes the process or emits an `uncaughtException` event.

Every error in JavaScript is (or should be) an instance of `Error` — or a subclass of it. The language ships with **seven built-in error types**, each signalling a different category of problem.

**Analogy — HTTP status codes:**
Just as HTTP has `400 Bad Request`, `401 Unauthorized`, `404 Not Found`, `500 Internal Server Error` — each meaning something specific — JavaScript error types tell you *what kind* of thing went wrong so you can handle each case differently.

---

## 2. Why does it matter?

- Knowing the error type lets you write precise `catch` blocks that handle only the errors you expect
- Reading the error name instantly tells you whether the bug is a typo (`ReferenceError`), a logic mistake (`TypeError`), a parsing problem (`SyntaxError`), or a runtime limit (`RangeError`)
- In Node.js / Express, mapping error types to HTTP status codes is standard practice
- Custom subclasses (topic 73) extend this system with your own domain-specific error types

---

## 3. The Error Base Class

Every built-in error type extends `Error`. The `Error` object has these properties:

```js
const err = new Error("Something went wrong");

err.name;       // "Error" (overridden by subclasses)
err.message;    // "Something went wrong"
err.stack;      // full stack trace string (non-standard but universally supported)
err.cause;      // optional — set via { cause: originalError } (ES2022)
```

### Creating an error with a cause (ES2022)

```js
try {
  JSON.parse(badInput);
} catch (parseErr) {
  throw new Error("Failed to process config file", { cause: parseErr });
  // err.cause === parseErr — preserves the original error
}
```

`cause` is invaluable for error chaining in backend code — you wrap a low-level error with context without losing the original stack trace.

---

## 4. The Seven Built-in Error Types

### 1. `SyntaxError`

**What:** Code that cannot be parsed — invalid JavaScript syntax.

**When it happens:** At parse time, before any code runs. You cannot `catch` a `SyntaxError` in the same script where it occurred — but you can catch one from `eval()`, `JSON.parse()`, or `new Function()`.

```js
// ❌ This is caught at parse time — try/catch can't help here
// let x = {;  // SyntaxError: Unexpected token ';'

// ✅ You CAN catch SyntaxError from dynamic parsing
try {
  JSON.parse('{ "name": "Alice" ');  // unterminated JSON
} catch (e) {
  console.log(e instanceof SyntaxError); // true
  console.log(e.name);    // "SyntaxError"
  console.log(e.message); // "Unexpected end of JSON input"
}

try {
  eval("let x = {;");
} catch (e) {
  console.log(e instanceof SyntaxError); // true
}
```

**Most common source in practice:** `JSON.parse()` with malformed or untrusted input — very frequent in backend API work.

---

### 2. `ReferenceError`

**What:** Accessing a variable that doesn't exist in the current scope.

```js
// ❌ Variable was never declared
console.log(username); // ReferenceError: username is not defined

// ❌ Temporal Dead Zone — let/const before their declaration
console.log(x); // ReferenceError: Cannot access 'x' before initialization
let x = 5;

// ❌ Misspelled function name
processOder(data); // ReferenceError: processOder is not defined
```

**Note:** Accessing a property that doesn't exist on an object is **not** a `ReferenceError` — it returns `undefined`. A `ReferenceError` is specifically about an *unresolvable binding* (an undeclared variable name).

```js
const user = {};
user.address; // undefined — NOT a ReferenceError
user.address.city; // TypeError (reading property of undefined) — NOT a ReferenceError
```

---

### 3. `TypeError`

**What:** An operation is performed on a value of the wrong type — calling a non-function, reading properties of `null`/`undefined`, etc.

This is the **most common error** in JavaScript.

```js
// Calling something that isn't a function
null();                         // TypeError: null is not a function
undefined();                    // TypeError: undefined is not a function

const user = null;
user.name;                      // TypeError: Cannot read properties of null (reading 'name')
user?.name;                     // ✅ undefined — optional chaining prevents the throw

// Calling a method on the wrong type
(42).toUpperCase();             // TypeError: (42).toUpperCase is not a function
"hello".push("world");          // TypeError: "hello".push is not a function

// Assigning to a read-only property (strict mode)
"use strict";
const obj = Object.freeze({ x: 1 });
obj.x = 2;                      // TypeError: Cannot assign to read only property 'x'

// Wrong arguments to built-in functions
Object.create("not an object"); // TypeError: Object prototype may only be an Object or null
```

**Pattern:** any time you see `Cannot read properties of null/undefined`, it's a `TypeError` from a missing guard.

---

### 4. `RangeError`

**What:** A value is outside the set or range of allowed values.

```js
// Array with invalid length
new Array(-1);               // RangeError: Invalid array length
new Array(4294967296);       // RangeError: Invalid array length (> 2^32 - 1)

// Number precision out of range
(1.2345).toFixed(200);       // RangeError: toFixed() digits argument must be between 0 and 100
(123).toString(1);           // RangeError: toString() radix must be between 2 and 36

// Call stack overflow (recursion without a base case)
function infinite() { return infinite(); }
infinite();                  // RangeError: Maximum call stack size exceeded

// Invalid time values
new Date("not a date").getTime(); // NaN (not a RangeError — but related concept)
```

**Most common in practice:** "Maximum call stack size exceeded" from accidental infinite recursion.

---

### 5. `URIError`

**What:** Incorrect use of global URI functions (`decodeURI`, `decodeURIComponent`, `encodeURI`, `encodeURIComponent`).

```js
// Malformed URI sequence
try {
  decodeURIComponent("%");        // URIError: URI malformed
} catch (e) {
  console.log(e instanceof URIError); // true
}

try {
  decodeURIComponent("%GG");      // URIError: URI malformed (invalid hex)
} catch (e) {
  console.log(e.name); // "URIError"
}

// ✅ Safe decoding pattern
function safeDecode(str) {
  try {
    return decodeURIComponent(str);
  } catch {
    return str; // return original if decoding fails
  }
}
```

**Frequency:** Rare in modern code. More common when processing user-supplied URL parameters from untrusted sources.

---

### 6. `EvalError`

**What:** An error related to the `eval()` function. 

**Reality:** Modern JS engines no longer throw `EvalError` in practice — it exists for historical compatibility with older specs. You might still see it in legacy code or error hierarchies but you will almost never encounter it thrown by the engine.

```js
// Modern JS engines do NOT throw EvalError
// It exists as a class for legacy reasons
const e = new EvalError("legacy");
e instanceof EvalError; // true
e instanceof Error;     // true
```

---

### 7. `AggregateError` (ES2021)

**What:** Wraps multiple errors into a single error object. Used by `Promise.any()` when all promises reject.

```js
// Promise.any() throws AggregateError if ALL promises reject
try {
  const result = await Promise.any([
    fetch("/api/primary").then(r => r.json()),
    fetch("/api/fallback").then(r => r.json()),
    fetch("/api/backup").then(r => r.json()),
  ]);
} catch (e) {
  if (e instanceof AggregateError) {
    console.log(e.message);  // "All promises were rejected"
    console.log(e.errors);   // [Error, Error, Error] — one per rejected promise
    e.errors.forEach((err, i) => console.log(`Source ${i}:`, err.message));
  }
}

// Manual use
throw new AggregateError([
  new Error("DB connection failed"),
  new Error("Cache connection failed"),
], "All data sources unavailable");
```

---

## 5. Error Type Comparison Table

| Type | When thrown | Catchable at throw site? | Common source |
|---|---|---|---|
| `SyntaxError` | Parse time | ❌ (unless from `eval`/`JSON.parse`) | Malformed JSON, `eval()` |
| `ReferenceError` | Runtime | ✅ | Undeclared var, TDZ |
| `TypeError` | Runtime | ✅ | `null`/`undefined` property access, wrong type |
| `RangeError` | Runtime | ✅ | Invalid array length, infinite recursion |
| `URIError` | Runtime | ✅ | Malformed `decodeURIComponent` input |
| `EvalError` | Never (modern) | ✅ | Legacy — ignore |
| `AggregateError` | Runtime | ✅ | `Promise.any()` all rejected |

---

## 6. Identifying the Error Type in a `catch` Block

```js
try {
  riskyOperation();
} catch (e) {
  // Option A — instanceof (preferred, works with custom subclasses too)
  if (e instanceof TypeError)     handleTypeError(e);
  else if (e instanceof RangeError) handleRangeError(e);
  else                             throw e; // re-throw unknown errors

  // Option B — e.name string comparison (works even across iframes/realms)
  if (e.name === "TypeError")  { ... }
  if (e.name === "SyntaxError") { ... }

  // Option C — check message (fragile — message strings vary by engine)
  if (e.message.includes("is not a function")) { ... }  // ❌ avoid
}
```

**Prefer `instanceof`** when code stays in one realm (standard browser/Node page). Use `e.name` when dealing with errors from different windows, iframes, or worker threads where `instanceof` may fail (different `Error` constructor references).

---

## 7. The Error Stack Trace

```js
function third() { throw new TypeError("bad value"); }
function second() { third(); }
function first()  { second(); }

try {
  first();
} catch (e) {
  console.log(e.stack);
}

// Output (Node.js):
// TypeError: bad value
//     at third (app.js:1:22)
//     at second (app.js:2:18)
//     at first (app.js:3:18)
//     at Object.<anonymous> (app.js:6:3)
//     ...
```

The stack trace reads **bottom to top** for cause, **top to bottom** for execution order. The top frame is where the error was thrown. The bottom frames are the entry points (event handler, HTTP request handler, etc.).

### Preserving stack traces when rethrowing

```js
// ❌ Loses original stack — creates a new Error at this line
try {
  riskyOp();
} catch (e) {
  throw new Error("Operation failed"); // original stack gone
}

// ✅ Use cause to chain errors (preserves original stack)
try {
  riskyOp();
} catch (e) {
  throw new Error("Operation failed", { cause: e }); // e.cause has original stack
}

// ✅ Or just re-throw as-is if you're not adding context
try {
  riskyOp();
} catch (e) {
  if (e instanceof TypeError) handleIt(e);
  else throw e; // re-throw unchanged
}
```

---

## 8. Errors in Async Code

### In Promises

```js
// Synchronous throw inside a Promise executor = rejected Promise
const p = new Promise((resolve, reject) => {
  throw new TypeError("bad input"); // becomes a rejection, not an unhandled throw
});

p.catch(e => console.log(e instanceof TypeError)); // true

// reject() with a non-Error is legal but bad practice
reject("something went wrong"); // string, not Error — loses stack trace
reject(new Error("something went wrong")); // ✅ always use Error objects
```

### In async/await

```js
async function fetchUser(id) {
  // Network errors throw TypeError ("Failed to fetch")
  const res = await fetch(`/api/users/${id}`);

  if (!res.ok) {
    // HTTP errors don't auto-throw — you throw manually
    const body = await res.json().catch(() => ({}));
    throw new Error(body.message || `HTTP ${res.status}`, { cause: res });
  }

  // JSON parse errors throw SyntaxError
  return res.json();
}

try {
  const user = await fetchUser(42);
} catch (e) {
  if (e instanceof SyntaxError)  console.error("Bad JSON from server:", e);
  else if (e instanceof TypeError) console.error("Network error:", e);
  else                             console.error("HTTP error:", e);
}
```

### `unhandledrejection` — global catch for missed promise rejections

```js
// Browser
window.addEventListener("unhandledrejection", (event) => {
  console.error("Unhandled promise rejection:", event.reason);
  event.preventDefault(); // suppress console error in some browsers
});

// Node.js
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled rejection:", reason);
  // In production: log to monitoring service, then optionally crash:
  // process.exit(1);
});
```

---

## 9. Real-World Patterns

### Pattern 1 — Express error handler with type mapping

```js
// Express central error handler (last middleware)
app.use((err, req, res, next) => {
  console.error(err.stack);

  // Map error types to HTTP status codes
  if (err instanceof SyntaxError && err.status === 400) {
    // express.json() throws this for malformed request body
    return res.status(400).json({ error: "Invalid JSON in request body" });
  }

  if (err.name === "ValidationError") {          // mongoose / custom
    return res.status(422).json({ error: err.message, fields: err.fields });
  }

  if (err.name === "UnauthorizedError") {        // express-jwt
    return res.status(401).json({ error: "Invalid or missing token" });
  }

  if (err.name === "CastError") {                // mongoose invalid ObjectId
    return res.status(400).json({ error: "Invalid ID format" });
  }

  // Default — 500 for everything else
  res.status(err.statusCode || 500).json({
    error: process.env.NODE_ENV === "production"
      ? "Internal server error"
      : err.message
  });
});
```

### Pattern 2 — Safe JSON parsing utility

```js
// JSON.parse throws SyntaxError on bad input
function safeJsonParse(str, fallback = null) {
  try {
    return JSON.parse(str);
  } catch (e) {
    if (e instanceof SyntaxError) return fallback;
    throw e; // re-throw unexpected errors
  }
}

const config = safeJsonParse(process.env.APP_CONFIG, {});
const data   = safeJsonParse(req.body, null);
```

### Pattern 3 — Retry only on specific error types

```js
async function withRetry(fn, { retries = 3, onError } = {}) {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      return await fn();
    } catch (e) {
      // Only retry on network errors (TypeError) — not on 4xx HTTP errors
      const isRetryable = e instanceof TypeError
        || e.message?.includes("ECONNRESET")
        || e.message?.includes("ETIMEDOUT");

      if (!isRetryable || attempt === retries) throw e;

      onError?.(e, attempt);
      await new Promise(r => setTimeout(r, 2 ** attempt * 100)); // exponential backoff
    }
  }
}

// Usage
const data = await withRetry(() => fetch("/api/data").then(r => r.json()), {
  retries: 3,
  onError: (e, attempt) => console.warn(`Attempt ${attempt} failed:`, e.message)
});
```

### Pattern 4 — Aggregating validation errors

```js
function validateUser(data) {
  const errors = [];

  if (!data.username) errors.push(new Error("Username is required"));
  if (!data.email)    errors.push(new Error("Email is required"));
  if (!data.password || data.password.length < 8)
    errors.push(new Error("Password must be at least 8 characters"));

  if (errors.length > 0) {
    throw new AggregateError(errors, "Validation failed");
  }
}

try {
  validateUser({});
} catch (e) {
  if (e instanceof AggregateError) {
    e.errors.forEach(err => console.log("-", err.message));
    // - Username is required
    // - Email is required
    // - Password must be at least 8 characters
  }
}
```

---

## 10. Tricky Things

### 1. `typeof undeclaredVar` does NOT throw a ReferenceError

```js
console.log(typeof undeclaredVar);  // "undefined" — no ReferenceError
console.log(undeclaredVar);         // ❌ ReferenceError

// This is intentional — typeof is the safe way to check existence of global vars
if (typeof window !== "undefined") {
  // We're in a browser, not Node.js
}
```

### 2. `instanceof` fails across iframes and worker threads

```js
// In a cross-origin iframe, the Error constructor is different
iframe.contentWindow.Error !== window.Error  // true — different realms

// instanceof check fails
iframeError instanceof Error // false!

// ✅ Use e.name or e.constructor.name for cross-realm checks
e.name === "TypeError"        // works everywhere
e.constructor.name === "TypeError" // also works
```

### 3. Errors are objects — properties can be added

```js
const err = new TypeError("Invalid input");
err.field  = "email";         // custom property
err.code   = "INVALID_EMAIL"; // custom code

throw err;

// In catch:
catch (e) {
  if (e.code === "INVALID_EMAIL") handleEmailError(e.field);
}
```

### 4. `catch` catches everything — including `null`, strings, numbers

```js
throw "something bad"; // legal but terrible — string, not Error

try {
  throw 42; // legal but terrible
} catch (e) {
  console.log(e);         // 42
  console.log(e.stack);   // undefined — no stack on a number
  console.log(e instanceof Error); // false
}

// ✅ Always throw Error instances
throw new Error("something bad"); // has .message, .stack, .name
```

### 5. Stack trace depth varies by environment

Node.js default is 10 frames. You can increase it:

```js
// At the start of your Node.js app
Error.stackTraceLimit = 50; // or Infinity for debugging
```

---

## 11. Common Mistakes

### Mistake 1 — Swallowing errors silently

```js
// ❌ Error disappears — impossible to debug
try {
  processData(input);
} catch (e) {
  // nothing
}

// ✅ Always at minimum log, ideally handle or rethrow
try {
  processData(input);
} catch (e) {
  console.error("processData failed:", e);
  throw e; // or handle specifically
}
```

### Mistake 2 — Catching too broadly and hiding bugs

```js
// ❌ Catches ALL errors including bugs you didn't intend to handle
try {
  const data = JSON.parse(input);
  processData(data);       // if this has a bug, it's silently caught
  renderUI(data);
} catch (e) {
  showError("Invalid input");
}

// ✅ Separate what you intend to handle from general processing
let data;
try {
  data = JSON.parse(input); // only catches SyntaxError from JSON.parse
} catch (e) {
  if (e instanceof SyntaxError) return showError("Invalid JSON");
  throw e;
}
processData(data); // bugs here are not caught — they surface properly
renderUI(data);
```

### Mistake 3 — Throwing non-Error values

```js
// ❌ No stack trace, no .message, no type info
throw "user not found";
throw { code: 404, msg: "not found" };

// ✅ Always throw Error instances (or subclasses)
throw new Error("User not found");
throw new RangeError("Page number must be positive");
```

### Mistake 4 — Re-wrapping without cause

```js
// ❌ Original error and its stack trace are lost
try {
  await db.query(sql);
} catch (e) {
  throw new Error("Database query failed"); // loses original DB error
}

// ✅ Preserve with cause
try {
  await db.query(sql);
} catch (e) {
  throw new Error("Database query failed", { cause: e });
}
```

### Mistake 5 — Not checking `instanceof` before accessing custom properties

```js
// ❌ If e is a generic Error, e.fields is undefined
catch (e) {
  showFieldErrors(e.fields); // TypeError if e.fields is undefined
}

// ✅ Check type first
catch (e) {
  if (e instanceof ValidationError) showFieldErrors(e.fields);
  else showGenericError(e.message);
}
```

---

## 12. Practice Exercises

### Easy — Error type detector

Write a function `identifyError(fn)` that calls `fn()` and returns a string describing the error type:
- `"SyntaxError"`, `"TypeError"`, `"ReferenceError"`, etc.
- `"no error"` if `fn()` runs successfully
- `"unknown error"` if the thrown value isn't an Error instance

Test it with at least four different functions that throw different error types.

---

### Medium — Safe data pipeline

Write a `safePipeline(input)` function that:
1. Parses `input` as JSON (catches `SyntaxError`)
2. Validates the result is an object with a `users` array (throws `TypeError` if not)
3. Validates each user has a `name` string and positive `age` number (throws `RangeError` if age ≤ 0)
4. Returns the validated users array

Wrap each step in its own try/catch with specific error messages. Test with valid, malformed JSON, missing fields, and invalid age values.

---

### Hard — Express-style error dispatcher

Build a `handleError(err, req)` function that:
- Takes any error and a mock `req` object (`{ path, method }`)
- Returns `{ statusCode, body }` based on the error type:
  - `SyntaxError` → 400
  - `TypeError` with message containing "not found" → 404
  - `RangeError` → 422
  - `AggregateError` → 422 with `{ errors: [...messages] }`
  - Anything with `.statusCode` property → use that status code
  - Default → 500
- In `NODE_ENV=production`, never expose internal error messages in the body for 5xx
- Log all errors with `console.error` including the full stack

Write unit-style tests (plain `if` assertions) for each branch.

---

## 13. Quick Reference

```
BUILT-IN ERROR TYPES
  SyntaxError        invalid syntax — caught from eval/JSON.parse
  ReferenceError     undeclared variable or TDZ
  TypeError          wrong type, null/undefined property access — MOST COMMON
  RangeError         value out of range, stack overflow
  URIError           malformed decodeURIComponent input
  EvalError          legacy — never thrown by modern engines
  AggregateError     multiple errors — from Promise.any()

CREATING ERRORS
  new Error("msg")
  new TypeError("msg")
  new Error("msg", { cause: originalError })    // ES2022 — chains errors

ERROR PROPERTIES
  .name        "TypeError", "RangeError", etc.
  .message     description string
  .stack       full stack trace
  .cause       original error (if set via { cause })

IDENTIFYING IN CATCH
  e instanceof TypeError    ← preferred (works with subclasses)
  e.name === "TypeError"    ← cross-realm safe

ASYNC ERRORS
  throw inside Promise executor  → rejected promise
  await throws  →  caught by try/catch
  unhandledrejection  → window / process.on("unhandledRejection")
  
NEVER DO
  catch (e) {}              // silent swallow
  throw "string"            // no stack trace
  throw new Error(e.message) // loses original stack — use { cause: e }
```

---

## 14. Connected Topics

- **72 — try/catch/finally in depth** — the mechanics of catching and re-throwing all these types
- **73 — Custom error classes** — extending `Error` to add `statusCode`, `fields`, `code` properties
- **52 — Async JS** — how errors propagate through promises and async/await
- **58 — Fetch API** — `TypeError` ("Failed to fetch") vs HTTP errors (not auto-thrown)
- **Node.js** — `process.on("uncaughtException")`, `process.on("unhandledRejection")`, Express error middleware
