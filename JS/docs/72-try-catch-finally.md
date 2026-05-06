# 72 — try / catch / finally in Depth

---

## 1. What is this?

`try / catch / finally` is JavaScript's mechanism for **handling runtime errors gracefully** instead of crashing. You wrap risky code in a `try` block. If anything throws, execution jumps to `catch`. The `finally` block runs no matter what — error or not, return or not.

**Analogy — database transaction:**
```
BEGIN TRANSACTION        ← try
  run your queries
ROLLBACK on error        ← catch
COMMIT / CLEANUP always  ← finally
```

In Node.js/Express you use this exact pattern: try the DB call, catch and return an error response, finally release the connection back to the pool.

---

## 2. Why does it matter?

- Prevents unhandled errors from crashing the entire page or Node process
- Lets you distinguish *expected* failures (network down, invalid input) from *unexpected bugs*
- `finally` guarantees cleanup code (releasing locks, closing files, re-enabling buttons) always runs — even if an early `return` is hit inside `try` or `catch`
- Properly re-throwing errors keeps bugs visible instead of silently swallowed

---

## 3. Syntax

```js
try {
  // code that might throw
} catch (err) {
  // runs only if try threw something
  // err is the thrown value (usually an Error object)
} finally {
  // runs ALWAYS — with or without an error, even after a return
}

// All combinations are valid:
try { } catch (e) { }           // try + catch
try { } finally { }             // try + finally (no catch — error still propagates)
try { } catch (e) { } finally { } // all three
```

---

## 4. How it works — execution flow

```js
function example(input) {
  try {
    console.log("1 — try starts");
    if (input === "bad") throw new Error("bad input");
    console.log("2 — try ends (no error)");
    return "success";
  } catch (err) {
    console.log("3 — catch runs:", err.message);
    return "recovered";
  } finally {
    console.log("4 — finally ALWAYS runs");
  }
}

example("good");
// 1 — try starts
// 2 — try ends (no error)
// 4 — finally ALWAYS runs
// return value: "success"

example("bad");
// 1 — try starts
// 3 — catch runs: bad input
// 4 — finally ALWAYS runs
// return value: "recovered"
```

### `finally` overrides return values

```js
function tricky() {
  try {
    return "from try";
  } finally {
    return "from finally"; // ⚠️ overrides the try's return
  }
}
tricky(); // "from finally"
```

This is a subtle footgun. **Never use `return` inside `finally`** unless you explicitly want to override — it also suppresses any exception that was thrown in `try` or `catch`.

```js
function dangerous() {
  try {
    throw new Error("real error");
  } finally {
    return "fine"; // ❌ silently swallows the Error — no exception propagates
  }
}
dangerous(); // "fine" — the Error is gone
```

---

## 5. The `catch` Block in Detail

### Optional binding (ES2019)

```js
// You don't have to name the error if you don't need it
try {
  JSON.parse(input);
} catch {
  // no (err) parameter needed
  return defaultValue;
}
```

### Re-throwing selectively

The most important pattern — only handle what you expect, re-throw everything else:

```js
try {
  const data = JSON.parse(rawInput);
  return processData(data);
} catch (e) {
  if (e instanceof SyntaxError) {
    // Expected: user sent bad JSON
    throw new Error("Invalid request body", { cause: e });
  }
  // Unexpected: bug in processData — re-throw so it surfaces
  throw e;
}
```

### `catch` catches everything — even non-Error throws

```js
try {
  throw "a plain string"; // legal but bad practice
} catch (e) {
  console.log(typeof e); // "string"
  console.log(e.stack);  // undefined — no stack on a string
}

try {
  throw 42;
} catch (e) {
  console.log(e); // 42
}

// ✅ Always check before using Error properties
catch (e) {
  const message = e instanceof Error ? e.message : String(e);
}
```

---

## 6. The `finally` Block in Depth

`finally` runs in all four possible exit scenarios from `try/catch`:

| Scenario | `finally` runs? |
|---|---|
| `try` completes normally | ✅ yes |
| `try` throws, `catch` handles it | ✅ yes |
| `try` throws, no `catch` | ✅ yes (then re-throws) |
| `try` or `catch` does `return` | ✅ yes (before return completes) |

### The primary use case — cleanup

```js
async function readFile(path) {
  let fileHandle;
  try {
    fileHandle = await fs.promises.open(path, "r");
    const content = await fileHandle.readFile("utf8");
    return content;
  } catch (e) {
    throw new Error(`Failed to read ${path}`, { cause: e });
  } finally {
    await fileHandle?.close(); // always close, even if readFile threw
  }
}
```

```js
// Express route — release DB connection no matter what
app.get("/users/:id", async (req, res, next) => {
  const client = await pool.connect();
  try {
    const result = await client.query("SELECT * FROM users WHERE id = $1", [req.params.id]);
    res.json(result.rows[0]);
  } catch (e) {
    next(e); // pass to error middleware
  } finally {
    client.release(); // ← always returns connection to pool
  }
});
```

```js
// UI — re-enable a button no matter what
async function handleSubmit(e) {
  e.preventDefault();
  submitBtn.disabled = true;
  try {
    await submitForm(new FormData(form));
    showSuccess();
  } catch (e) {
    showError(e.message);
  } finally {
    submitBtn.disabled = false; // ← always re-enabled
  }
}
```

### `try` without `catch` — just cleanup

```js
// Error still propagates — we just need to ensure cleanup
async function withLock(lockKey, fn) {
  await acquireLock(lockKey);
  try {
    return await fn();
  } finally {
    await releaseLock(lockKey); // always release, even if fn() throws
    // The error from fn() still propagates to the caller
  }
}
```

---

## 7. try / catch / finally in Async Code

### async/await — same syntax, async errors

```js
async function fetchUser(id) {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } catch (e) {
    // Catches both: network errors (TypeError) and our thrown HTTP error
    console.error("fetchUser failed:", e);
    throw e; // re-throw so the caller can also handle it
  } finally {
    console.log("fetchUser done"); // runs regardless of success/failure
  }
}
```

### Await inside finally

```js
async function processJob(jobId) {
  try {
    await markJobRunning(jobId);
    await doHeavyWork(jobId);
    await markJobDone(jobId);
  } catch (e) {
    await markJobFailed(jobId, e.message); // async in catch ✅
    throw e;
  } finally {
    await releaseJobResources(jobId);      // async in finally ✅
  }
}
```

### Unhandled rejection vs caught rejection

```js
// ❌ This rejection is unhandled — crashes Node / logs warning in browser
async function bad() {
  throw new Error("oops");
}
bad(); // no await, no .catch()

// ✅ Always await or .catch()
await bad().catch(e => console.error(e));

// ✅ Or use try/catch with await
try {
  await bad();
} catch (e) {
  console.error(e);
}
```

### Promise chain `.catch()` vs try/catch — same thing

```js
// These are equivalent:

// Promise chain style
fetch("/api/data")
  .then(r => r.json())
  .then(processData)
  .catch(e => handleError(e))
  .finally(() => hideSpinner());

// async/await style
async function load() {
  try {
    const r    = await fetch("/api/data");
    const data = await r.json();
    processData(data);
  } catch (e) {
    handleError(e);
  } finally {
    hideSpinner();
  }
}
```

---

## 8. Error Propagation — the Full Picture

```
function c() { throw new Error("boom"); }
function b() { c(); }             // no try/catch — error propagates up
function a() {
  try {
    b();
  } catch (e) {
    console.log("caught in a:", e.message); // "boom"
  }
}
a();
```

The call stack unwinds until a `catch` is found. If none is found: browser logs to console + stops the current event handler; Node.js emits `uncaughtException`.

### Propagation through async boundaries

```js
async function c() { throw new Error("async boom"); }
async function b() { await c(); }   // no try/catch — rejection propagates
async function a() {
  try {
    await b();
  } catch (e) {
    console.log("caught:", e.message); // "async boom"
  }
}
```

The `await` keyword unwraps rejected promises into thrown errors — so try/catch works identically across sync and async.

---

## 9. Real-World Patterns

### Pattern 1 — Layered error handling (backend style)

```js
// Data layer — wraps DB errors with context
async function getUserById(id) {
  try {
    const row = await db.query("SELECT * FROM users WHERE id = $1", [id]);
    if (!row) throw Object.assign(new Error("User not found"), { code: "NOT_FOUND" });
    return row;
  } catch (e) {
    if (e.code === "NOT_FOUND") throw e; // known — pass through
    throw new Error(`DB error fetching user ${id}`, { cause: e }); // wrap unknown
  }
}

// Service layer — adds business logic errors
async function getProfile(userId) {
  try {
    const user    = await getUserById(userId);
    const profile = await getProfileData(user.profileId);
    return { user, profile };
  } catch (e) {
    if (e.code === "NOT_FOUND") throw e;
    throw new Error("Failed to build profile", { cause: e });
  }
}

// Route handler — maps to HTTP responses
app.get("/profile/:id", async (req, res, next) => {
  try {
    const profile = await getProfile(req.params.id);
    res.json(profile);
  } catch (e) {
    if (e.code === "NOT_FOUND") return res.status(404).json({ error: e.message });
    next(e); // unknown error → 500 via central error handler
  }
});
```

### Pattern 2 — `withRetry` with granular error handling

```js
async function withRetry(fn, { retries = 3, delay = 200 } = {}) {
  let lastError;
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (e) {
      lastError = e;
      const retryable = e instanceof TypeError            // network error
        || (e.statusCode >= 500 && e.statusCode < 600);  // 5xx server error
      if (!retryable) throw e;                           // 4xx — don't retry
      if (i < retries - 1) await new Promise(r => setTimeout(r, delay * 2 ** i));
    }
  }
  throw lastError;
}
```

### Pattern 3 — `withTimeout` using finally for cleanup

```js
function withTimeout(promise, ms) {
  let timerId;
  const timeout = new Promise((_, reject) => {
    timerId = setTimeout(
      () => reject(new Error(`Operation timed out after ${ms}ms`)),
      ms
    );
  });

  return Promise.race([promise, timeout]).finally(() => {
    clearTimeout(timerId); // ← always cancel the timer, win or lose
  });
}

// Usage
const data = await withTimeout(fetch("/api/slow"), 5000).then(r => r.json());
```

### Pattern 4 — Transaction wrapper

```js
async function withTransaction(pool, fn) {
  const client = await pool.connect();
  try {
    await client.query("BEGIN");
    const result = await fn(client);
    await client.query("COMMIT");
    return result;
  } catch (e) {
    await client.query("ROLLBACK");
    throw e; // re-throw — caller handles the business error
  } finally {
    client.release(); // always return to pool
  }
}

// Usage
const user = await withTransaction(pool, async (client) => {
  const { rows: [u] } = await client.query(
    "INSERT INTO users(name, email) VALUES($1,$2) RETURNING *",
    [name, email]
  );
  await client.query(
    "INSERT INTO audit_log(user_id, action) VALUES($1,'created')",
    [u.id]
  );
  return u;
});
```

### Pattern 5 — Centralized async route wrapper (Express)

```js
// Without this, every route needs its own try/catch
const asyncRoute = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Now routes don't need try/catch — errors auto-route to error middleware
app.get("/users/:id", asyncRoute(async (req, res) => {
  const user = await getUserById(req.params.id);
  res.json(user);
}));

app.post("/orders", asyncRoute(async (req, res) => {
  const order = await createOrder(req.body);
  res.status(201).json(order);
}));
```

---

## 10. Tricky Things

### 1. `finally` return overrides everything — including throws

```js
function sneaky() {
  try {
    throw new Error("real error");
  } catch (e) {
    throw new Error("catch error");
  } finally {
    return "oops"; // ❌ silently swallows BOTH errors
  }
}
sneaky(); // "oops" — no error thrown
```

Rule: **never `return` or `throw` from `finally`** unless you understand you're suppressing the original error.

### 2. `try` without `await` doesn't catch async errors

```js
// ❌ try/catch only covers synchronous throws inside the try block
// The Promise rejects but nothing catches it
try {
  fetchUser(1); // no await — returns a Promise, doesn't throw here
} catch (e) {
  console.log("never runs");
}

// ✅ Must await to catch async errors
try {
  await fetchUser(1);
} catch (e) {
  console.log("caught:", e.message);
}
```

### 3. `catch` variable is scoped to the catch block

```js
try {
  throw new Error("oops");
} catch (e) {
  console.log(e.message); // "oops"
}
console.log(e); // ❌ ReferenceError: e is not defined
```

### 4. Nested try/catch — inner catch swallows before outer sees it

```js
function outer() {
  try {
    inner();
  } catch (e) {
    console.log("outer caught:", e.message); // only runs if inner re-throws
  }
}

function inner() {
  try {
    throw new Error("inner error");
  } catch (e) {
    console.log("inner caught:", e.message); // "inner error"
    // not re-thrown — outer never sees it
  }
}
```

### 5. `for...of` and `forEach` differ in how errors propagate

```js
// ❌ try/catch outside forEach cannot catch throws INSIDE the callback
try {
  [1, 2, 3].forEach(n => {
    if (n === 2) throw new Error("found 2");
  });
} catch (e) {
  console.log("caught"); // actually DOES run — forEach propagates sync throws
}

// ✅ But async callbacks in forEach are NOT caught
try {
  [1, 2, 3].forEach(async (n) => {
    await Promise.reject(new Error("async error")); // unhandled rejection!
  });
} catch (e) {
  console.log("never runs"); // forEach returns before promises settle
}

// ✅ Use for...of with await for async iterations
for (const n of [1, 2, 3]) {
  await riskyAsyncOp(n); // errors caught by outer try/catch
}
```

---

## 11. Common Mistakes

### Mistake 1 — `try` around too much code

```js
// ❌ One big try catches everything — bugs in processData hide as "parse errors"
try {
  const data = JSON.parse(input);
  processData(data);
  updateUI(data);
  sendAnalytics(data);
} catch (e) {
  showError("Invalid JSON");
}

// ✅ Narrow try blocks — catch only what you expect
let data;
try {
  data = JSON.parse(input);
} catch (e) {
  if (e instanceof SyntaxError) return showError("Invalid JSON");
  throw e;
}
processData(data); // bugs here surface properly
updateUI(data);
sendAnalytics(data);
```

### Mistake 2 — Re-throwing as a new Error without cause

```js
// ❌ Original stack trace lost — debugging becomes harder
catch (e) {
  throw new Error("Something failed: " + e.message);
}

// ✅ Use { cause } to chain
catch (e) {
  throw new Error("Something failed", { cause: e });
}
```

### Mistake 3 — Async function inside try without await

```js
// ❌ The error is not caught here
try {
  loadUser(); // async, not awaited
} catch (e) { }

// ✅ Await it
try {
  await loadUser();
} catch (e) { }
```

### Mistake 4 — Using `finally` for conditional logic

```js
// ❌ finally is for cleanup, not branching
let success = false;
try {
  doWork();
  success = true;
} finally {
  if (success) celebrate();
  else cry();
}

// ✅ Put conditional logic in try/catch
try {
  doWork();
  celebrate();
} catch (e) {
  cry();
  throw e;
}
```

### Mistake 5 — Silent catch in production code

```js
// ❌ Error vanishes — system behaves incorrectly with no visible cause
catch (e) { /* ignored */ }

// ✅ At minimum, log it
catch (e) {
  logger.error({ err: e, context: "fetchUser" });
  throw e; // or return a safe fallback
}
```

---

## 12. Practice Exercises

### Easy — retry wrapper

Write `retry(fn, times)` that calls `fn()` up to `times` times, retrying only if it throws. Return the first successful result. If all attempts throw, re-throw the last error.

Test with a function that throws twice then succeeds on the third call.

---

### Medium — safe async pipeline

Write `safeFetch(url)` that:
1. Fetches the URL inside a try/catch
2. Catches `TypeError` (network down) and throws a custom `NetworkError`
3. Checks `res.ok` — if not, reads the error body and throws an `HttpError` with `.statusCode` and `.body`
4. Parses JSON — catches `SyntaxError` and throws a `ParseError`
5. Has a `finally` block that logs the URL and whether it succeeded or failed

---

### Hard — transaction helper

Write `withTransaction(pool, fn)`:
- Acquires a client from `pool` (mock it as a simple object with `.query()`, `.release()`)
- Runs `BEGIN`, calls `fn(client)`, runs `COMMIT`
- On any error: runs `ROLLBACK`, re-throws the error
- `finally`: always calls `client.release()`
- Write three tests:
  1. Happy path — transaction commits, result returned
  2. fn() throws — transaction rolls back, error re-thrown, client released
  3. COMMIT throws — ROLLBACK runs, client released

---

## 13. Quick Reference

```
STRUCTURE
  try { risky } catch (e) { handle } finally { cleanup }
  try { risky } finally { cleanup }          // error still propagates
  try { risky } catch (e) { handle }         // no cleanup

EXECUTION ORDER
  No error:   try → finally
  Error:      try → catch → finally
  Return:     try (return) → finally → return completes
  finally return/throw OVERRIDES try/catch return/throw — avoid!

CATCH
  catch (e) { }              // e = thrown value
  catch { }                  // ES2019 — omit binding if unused
  if (e instanceof X) { }    // handle specific types
  else throw e;              // re-throw everything else

FINALLY
  Always runs — with or without error, even after return
  Use for: close connections, release locks, re-enable UI, clear timers
  Never return/throw from finally unless intentional

ASYNC
  await inside try catches rejected promises — same syntax as sync
  No await = no catch — Promise rejects silently
  async in finally ✅ — awaited before finally exits
  forEach async callback ❌ — errors not caught by outer try
  for...of with await ✅ — errors caught normally

RE-THROW PATTERN
  try { op() } catch (e) {
    if (e instanceof Expected) handle(e);
    else throw e;   ← let unexpected errors surface
  }

CHAINING
  throw new Error("context", { cause: originalErr })
  err.cause → original error and its stack
```

---

## 14. Connected Topics

- **71 — Types of JS errors** — `SyntaxError`, `TypeError`, `RangeError` etc. — what you're catching
- **73 — Custom error classes** — subclassing `Error` to create `HttpError`, `ValidationError`, `NetworkError`
- **52 — Async JS** — promise rejection = async error; how rejected promises and try/catch interact
- **58 — Fetch API** — `TypeError` (network) vs HTTP errors (manual throw) vs `SyntaxError` (bad JSON)
- **62 — Generators** — `generator.throw(err)` injects errors into a generator's try/catch
- **Node.js** — `uncaughtException`, `unhandledRejection`, Express error middleware pattern
