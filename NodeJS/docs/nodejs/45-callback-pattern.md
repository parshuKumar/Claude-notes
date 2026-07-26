# 45 — Callback pattern deep dive

## What is this?

A callback is simply a function you pass into another function so it can be called back later, once some work finishes. In Node.js, the convention is the **error-first callback**: the callback's first argument is always reserved for an error (or `null` if nothing went wrong), and the second argument is the actual result. Think of it like ordering food at a restaurant — you don't stand at the counter waiting; you leave your phone number (the callback), and the kitchen "calls you back" with either "your order is ready" or "sorry, we're out of that dish."

---

## Why does it matter for backend development?

Node's entire core library — `fs`, `dns`, `crypto`, the original `http` request handling, database drivers before promises existed — was built on this exact pattern before promises and `async/await` existed in JavaScript. Even today, many older npm packages, low-level APIs, and event-based systems still expose callback-style functions. A backend developer needs to recognize error-first callbacks on sight, know why "callback hell" (the pyramid of doom) happens, know how to flatten it, and know how to wrap a legacy callback API into a promise so it plays nicely with modern `async/await` code. Understanding callbacks properly is also what makes promises make sense — promises were invented specifically to fix the pain points of callbacks.

---

## Syntax / API

```js
// The universal Node.js convention: error-first callback
// function(error, result) { ... }

const fs = require('fs');

// fs.readFile takes: path, encoding, and a callback
fs.readFile('/etc/app-config.json', 'utf8', function callback(err, data) {
  // ── Rule #1 of error-first callbacks: ALWAYS check err first ──
  if (err) {
    // Something went wrong — handle it and STOP here (return early)
    console.error('Failed to read config:', err.message);
    return;
  }

  // ── Only reached if err is null/undefined — safe to use data ──
  console.log('Config loaded:', data);
});

// A callback you write yourself follows the same shape
function fetchUserById(userId, callback) {
  // Simulate an async lookup (e.g. a database call)
  setTimeout(() => {
    if (!userId) {
      // Something is wrong with the input — call back with an error
      return callback(new Error('userId is required'));
    }
    // Success — call back with (null, result)
    callback(null, { userId, name: 'Ravi Kumar' });
  }, 100);
}

// Consuming your own callback-based function
fetchUserById('user_101', (err, user) => {
  if (err) return console.error(err.message); // handle error path first
  console.log('User found:', user);            // then the success path
});
```

---

## How it works — line by line

- `fs.readFile(path, encoding, callback)` starts reading a file **without blocking** the rest of the program. Node hands the actual disk read off to its internal thread pool (libuv) and moves on immediately.
- When the read finishes (success or failure), Node schedules your `callback` function to run with the result — this is what "callback" means: a function that gets called back later, not right now.
- Inside the callback, `err` is checked first. This is the **error-first** convention — if the operation failed, `err` holds an `Error` object; if it succeeded, `err` is `null`.
- `return` after handling the error is critical — it stops execution so the success-path code below never runs on a failed call.
- `data` is only trustworthy once you've confirmed `err` is falsy — reading `data` before checking `err` is how bugs sneak into production.
- In `fetchUserById`, the same pattern is written by hand: `setTimeout` simulates an async operation, and the function calls `callback(new Error(...))` on failure or `callback(null, result)` on success — matching Node's own core convention exactly.
- The consumer of `fetchUserById` checks `err` first, exactly the same way it would for any Node core API — this consistency is the entire point of the convention: once you know the pattern, you can use *any* callback-based API the same way.

---

## Example 1 — basic

```js
// File: src/examples/read-user-file.js
// Demonstrates a single error-first callback in isolation

const fs = require('fs');
const path = require('path');

// Build a safe absolute path to a user data file
const filePath = path.join(__dirname, 'data', 'user_42.json');

// Read the file asynchronously — callback runs once the OS responds
fs.readFile(filePath, 'utf8', (err, fileContents) => {
  // Step 1: always check the error first
  if (err) {
    // err.code tells us WHY it failed (e.g. 'ENOENT' = file not found)
    if (err.code === 'ENOENT') {
      console.error(`No user file found at ${filePath}`);
    } else {
      console.error('Unexpected error reading file:', err.message);
    }
    return; // stop here — do not try to use fileContents
  }

  // Step 2: only parse JSON once we know the read succeeded
  const userRecord = JSON.parse(fileContents);
  console.log('Loaded user:', userRecord.userId, userRecord.name);
});

// This line runs BEFORE the callback above — proves it's non-blocking
console.log('Reading file... (this logs first)');
```

---

## Example 2 — real world backend use case

```js
// File: src/services/registerUser.js
// A realistic registration flow using ONLY callback-style APIs —
// this is the "pyramid of doom" a backend dev must recognize and later fix.

const crypto = require('crypto');
const fs = require('fs');
const path = require('path');

// Simulated callback-based "database" lookup
function findUserByEmail(email, callback) {
  setTimeout(() => callback(null, null), 50); // null = no existing user found
}

// Simulated callback-based password hashing (crypto's real API IS callback-based)
function hashPassword(plainPassword, callback) {
  const salt = crypto.randomBytes(16).toString('hex');
  // scrypt is Node's built-in async, callback-based hashing function
  crypto.scrypt(plainPassword, salt, 64, (err, derivedKey) => {
    if (err) return callback(err);
    callback(null, `${salt}:${derivedKey.toString('hex')}`);
  });
}

// Simulated callback-based "save to database"
function saveUserRecord(userRecord, callback) {
  const filePath = path.join(__dirname, '..', 'data', `${userRecord.userId}.json`);
  fs.writeFile(filePath, JSON.stringify(userRecord), 'utf8', (err) => {
    if (err) return callback(err);
    callback(null, userRecord);
  });
}

// THE PYRAMID OF DOOM — each async step nests one level deeper than the last
function registerUser(requestBody, finalCallback) {
  const { email, password } = requestBody;

  findUserByEmail(email, (err, existingUser) => {          // level 1
    if (err) return finalCallback(err);
    if (existingUser) return finalCallback(new Error('Email already registered'));

    hashPassword(password, (err, passwordHash) => {          // level 2
      if (err) return finalCallback(err);

      const userId = crypto.randomUUID();
      const userRecord = { userId, email, passwordHash };

      saveUserRecord(userRecord, (err, savedUser) => {        // level 3
        if (err) return finalCallback(err);

        finalCallback(null, { userId: savedUser.userId, email: savedUser.email });
      });
    });
  });
}

// Usage — notice how deep the indentation goes even with just 3 steps
registerUser({ email: 'ravi@example.com', password: 'S3cure!Pass' }, (err, newUser) => {
  if (err) return console.error('Registration failed:', err.message);
  console.log('User registered:', newUser);
});

module.exports = { registerUser };
```

---

## Common mistakes

### Mistake 1 — Forgetting to check the error before using the result

```js
// ❌ WRONG — data is used even when err exists, causing a crash on failure
fs.readFile(filePath, 'utf8', (err, data) => {
  const config = JSON.parse(data); // data is undefined if err was set — crashes here
  console.log(config);
});

// ✅ CORRECT — always guard on err first and return early
fs.readFile(filePath, 'utf8', (err, data) => {
  if (err) {
    console.error('Could not read config file:', err.message);
    return; // stop execution — never touch `data` after an error
  }
  const config = JSON.parse(data);
  console.log(config);
});
```

### Mistake 2 — Calling the callback more than once

```js
// ❌ WRONG — callback fires twice if the timeout AND validation both trigger,
// which can cause double DB writes, double responses, or crashes ("headers already sent")
function chargeCard(amount, callback) {
  if (amount <= 0) {
    callback(new Error('Invalid amount')); // fires once here...
  }
  setTimeout(() => {
    callback(null, { charged: amount }); // ...but ALSO fires here, always
  }, 100);
}

// ✅ CORRECT — return immediately after calling back to guarantee a single call
function chargeCard(amount, callback) {
  if (amount <= 0) {
    return callback(new Error('Invalid amount')); // return stops further execution
  }
  setTimeout(() => {
    callback(null, { charged: amount });
  }, 100);
}
```

### Mistake 3 — Nesting callbacks instead of flattening or converting to async/await

```js
// ❌ WRONG — deeply nested "pyramid of doom", hard to read, hard to add error
// handling consistently, and impossible to reuse individual steps
function processOrder(orderId, callback) {
  findOrder(orderId, (err, order) => {
    if (err) return callback(err);
    chargeCustomer(order, (err, payment) => {
      if (err) return callback(err);
      updateInventory(order, (err, inventory) => {
        if (err) return callback(err);
        sendConfirmationEmail(order, (err) => {
          if (err) return callback(err);
          callback(null, { orderId, status: 'complete' });
        });
      });
    });
  });
}

// ✅ CORRECT — wrap each callback-based step with util.promisify and use async/await
// (flat, sequential, readable — see Topic 24 and Topic 47 for full details)
const util = require('util');
const findOrderAsync = util.promisify(findOrder);
const chargeCustomerAsync = util.promisify(chargeCustomer);
const updateInventoryAsync = util.promisify(updateInventory);
const sendConfirmationEmailAsync = util.promisify(sendConfirmationEmail);

async function processOrder(orderId) {
  const order = await findOrderAsync(orderId);
  const payment = await chargeCustomerAsync(order);
  await updateInventoryAsync(order);
  await sendConfirmationEmailAsync(order);
  return { orderId, status: 'complete' };
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `validateAge(age, callback)` that:
1. Calls `callback(new Error('...'))` if `age` is not a number, or is negative, or is over 150
2. Otherwise calls `callback(null, { age, isAdult: age >= 18 })`
3. Follows the error-first callback convention exactly (error first, result second, never both)

Test it by calling it three times: once with a valid adult age, once with a valid minor age, and once with an invalid value like `-5`. Log the outcome of each call.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a small callback-based "cache" module with two functions:
1. `setCache(key, value, callback)` — stores the value in an in-memory object keyed by `key`, then calls `callback(null, true)`
2. `getCache(key, callback)` — looks up the key; if found calls `callback(null, value)`, if not found calls `callback(new Error('Cache miss for key: ' + key))`

Then write a `getOrFetch(key, fetchFn, callback)` function that:
1. Tries `getCache(key, ...)` first
2. If it's a cache hit, calls `callback(null, cachedValue)` immediately
3. If it's a cache miss, calls `fetchFn(key, ...)` (a callback-based "database" simulator you write yourself using `setTimeout`) to get fresh data, stores it with `setCache`, then calls the original `callback(null, freshValue)`

This is intentionally nested 2–3 levels deep — that nesting is the point of the exercise.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a callback-based `TaskRunner` that processes an array of async tasks **one at a time, in order** (not in parallel), without using promises or async/await anywhere:
1. `runTasksInSeries(tasks, finalCallback)` where `tasks` is an array of functions, each shaped like `function(callback)` that eventually calls `callback(err, result)`
2. It must run `tasks[0]`, wait for its callback, THEN run `tasks[1]`, and so on
3. It collects every successful result into an array, in order
4. If ANY task calls back with an error, it must immediately stop running further tasks and call `finalCallback(err)` — no result array in that case
5. If all tasks succeed, it calls `finalCallback(null, resultsArray)`

Test it with three fake async tasks (using `setTimeout` to simulate delay) where the third one is deliberately made to fail, and confirm `runTasksInSeries` stops before running a fourth task you add after it.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
THE ERROR-FIRST CONVENTION
  function(err, result) { ... }
  err === null/undefined  → success, use `result`
  err is an Error object  → failure, do NOT use `result`

GOLDEN RULES
  1. Always check `err` first, before touching the result
  2. Always `return` after handling an error (prevents double execution)
  3. Never call the callback more than once
  4. Never call the callback synchronously AND asynchronously in the same
     function — pick one (always sync or always async) to avoid Zalgo bugs

CALLBACK HELL / PYRAMID OF DOOM
  Symptom: code drifts right with every nested async step
  Cause:   each step's logic lives INSIDE the previous step's callback
  Fixes:   - name functions instead of using anonymous inline callbacks
           - extract steps into standalone functions
           - util.promisify() + async/await (Topics 24, 47)
           - use Promise chaining (Topic 46)

WHERE NODE CORE STILL USES CALLBACKS
  fs.readFile / fs.writeFile / fs.readdir   (fs/promises exists too — Topic 16)
  crypto.scrypt / crypto.pbkdf2
  dns.lookup / dns.resolve
  EventEmitter listeners (not exactly the same pattern, but callback-shaped)
  Many older npm packages (redis clients, some ORMs, legacy SDKs)

CONVERTING A CALLBACK API TO A PROMISE
  const util = require('util');
  const readFileAsync = util.promisify(fs.readFile);
  const data = await readFileAsync(filePath, 'utf8');

WRITING YOUR OWN CALLBACK-BASED FUNCTION
  function doSomething(input, callback) {
    if (invalid(input)) return callback(new Error('bad input'));
    asyncWork(input, (err, result) => {
      if (err) return callback(err);
      callback(null, result);
    });
  }
```

---

## Connected topics

- **46 — Promises in Node context** — the direct successor to callbacks; shows how `.then()/.catch()` chains fix the pyramid of doom
- **24 — util module** — `util.promisify()` is the standard way to convert any error-first callback function into a promise-returning one
- **07 — Errors and error handling in Node** — deeper dive into the `Error` object itself, custom error types, and how error-first callbacks fit into Node's broader error strategy
