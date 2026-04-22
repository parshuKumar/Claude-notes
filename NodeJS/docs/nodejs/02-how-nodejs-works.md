# 02 — How Node.js Works Internally

## What is this?

Node.js runs on a **single thread** — meaning it has only one call stack and executes one piece of JavaScript at a time. Yet it can handle thousands of simultaneous connections without freezing. It achieves this through **event-driven, non-blocking I/O**: instead of waiting for slow operations (file reads, database queries, network calls) to finish, Node registers a callback and immediately moves on to the next task. When the slow operation completes, its callback is queued and executed. Think of it like a restaurant waiter who takes your order, hands it to the kitchen, and goes to serve other tables — rather than standing at your table waiting for the food to cook.

## Why does it matter for backend development?

Most backend work is I/O-bound — waiting for a database, waiting for an API, waiting for a file. In a traditional blocking model (Java/Python with threads), each waiting connection holds a thread that consumes memory. Node's non-blocking model means one thread can juggle thousands of waiting connections with very low memory overhead. This is why Node excels at APIs, real-time apps, and microservices. However, it is **bad at CPU-heavy work** (image processing, heavy computation) because that does block the single thread — you'll learn how to handle that with worker threads in Topic 28.

---

## Syntax / API

```js
// This example demonstrates the core difference: blocking vs non-blocking

const fs = require('fs');

// ---- BLOCKING (synchronous) ----
// Node STOPS here and waits until the file is fully read before moving on
const fileContents = fs.readFileSync('./data.txt', 'utf8');  // blocks the thread
console.log(fileContents);
console.log('This line runs AFTER the file is read');        // waits for above

// ---- NON-BLOCKING (asynchronous) ----
// Node REGISTERS the callback, then immediately continues — does NOT wait
fs.readFile('./data.txt', 'utf8', (err, fileContents) => {  // callback registered
  if (err) throw err;
  console.log(fileContents);                                 // runs LATER, when done
});
console.log('This line runs IMMEDIATELY, before the file is read'); // runs first
```

---

## How it works — line by line

Here is the full picture of what happens when Node.js runs:

```
Your JS Code
     │
     ▼
  [ V8 Engine ]          ← parses and executes JavaScript
     │
     │  hits an async operation (readFile, http request, setTimeout...)
     ▼
  [ Node.js Bindings ]   ← C++ layer connecting JS to the OS
     │
     ▼
  [ libuv ]              ← the real engine behind the scenes
  ┌──────────────────────────────────────┐
  │  Event Loop  ←──────────────────┐   │
  │      │                          │   │
  │      │ no more sync work        │   │
  │      ▼                          │   │
  │  [ I/O Poll ]                   │   │
  │      │                          │   │
  │      ▼                          │   │
  │  Thread Pool (4 threads by default) │
  │  [file read] [dns] [crypto] [zlib]  │
  │      │                          │   │
  │      │ operation complete       │   │
  │      └──── callback queued ─────┘   │
  └──────────────────────────────────────┘
     │
     ▼
  Callback executes in V8 (back on the single thread)
```

**Step by step:**

1. Your JS runs on V8 — synchronous code executes top to bottom on the single thread.
2. When Node hits `fs.readFile(...)`, it does NOT block. It hands the task to libuv.
3. libuv either uses the OS's async I/O (for networking) or its own thread pool (for file system, DNS, crypto).
4. The single JS thread is now FREE — it continues executing the next lines of code.
5. When the file read finishes, libuv pushes the callback into the **event queue**.
6. The **event loop** picks it up (when the call stack is empty) and runs the callback on the single thread.

This is why Node can handle 10,000 simultaneous HTTP connections — they are all sitting in libuv waiting for I/O, not blocking the JS thread.

---

## Example 1 — basic

```js
// File: non-blocking-demo.js
// Demonstrates the order of execution in Node's async model

const fs = require('fs');

console.log('1 — Script starts');           // runs first — synchronous

// Register an async operation — Node does NOT wait here
fs.readFile('./package.json', 'utf8', (err, data) => {
  // This callback runs LATER, after the event loop picks it up
  if (err) {
    console.log('Error reading file:', err.message);
    return;
  }
  console.log('3 — File read complete, size:', data.length, 'chars');  // runs last
});

console.log('2 — After readFile call');     // runs second — synchronous, before callback

// OUTPUT ORDER:
// 1 — Script starts
// 2 — After readFile call
// 3 — File read complete, size: XXXX chars
```

**The key insight:** Line `console.log('2...')` runs *before* the file is finished reading — even though the `readFile` call came first. The callback only runs when the event loop gets to it.

---

## Example 2 — real world backend use case

```js
// File: concurrent-requests-demo.js
// Simulates what happens when an HTTP server receives 3 requests at the same time,
// each needing to read a file and query a database (simulated with setTimeout).
// This is what Node handles every second in production.

const fs = require('fs');

// Simulate 3 incoming HTTP requests arriving at the same time
function handleRequest(requestId) {
  console.log(`[${requestId}] Request received — starting processing`);

  // Simulate: read a config file (async I/O — goes to libuv)
  fs.readFile('./package.json', 'utf8', (err, configData) => {
    if (err) return console.error(`[${requestId}] File error:`, err.message);
    console.log(`[${requestId}] Config loaded (${configData.length} chars)`);

    // Simulate: database query (async — goes to libuv/thread pool)
    setTimeout(() => {
      const userId = `user_${requestId}_data`;   // simulated DB result
      console.log(`[${requestId}] DB query complete — userId: ${userId}`);
      console.log(`[${requestId}] Sending response`);
    }, Math.random() * 100);   // random delay to simulate varying DB response times
  });

  console.log(`[${requestId}] Request handler returned — thread is FREE`);
}

// All 3 requests are kicked off synchronously
// Node does NOT wait for request 1 to finish before starting request 2
handleRequest('REQ-001');
handleRequest('REQ-002');
handleRequest('REQ-003');

// OUTPUT shows all 3 requests are being processed CONCURRENTLY
// even though Node is single-threaded — I/O is all in-flight at once
```

**Why this matters:** A multi-threaded Java server would spin up 3 threads for 3 requests. Node handles all 3 on one thread — the file reads and DB queries run in libuv's thread pool, the JS thread processes callbacks as they complete. At scale (1000 requests), Node still uses one JS thread.

---

## Common mistakes

### Mistake 1 — Using synchronous methods in a server

```js
// ❌ WRONG — readFileSync blocks the ENTIRE server while it reads
// Every other request has to wait — this kills concurrency
app.get('/config', (req, res) => {
  const config = fs.readFileSync('./config.json', 'utf8');  // BLOCKS the thread
  res.send(config);
});

// ✅ CORRECT — async version; the thread is free while the file is being read
app.get('/config', (req, res) => {
  fs.readFile('./config.json', 'utf8', (err, config) => {
    if (err) return res.status(500).send('Error');
    res.send(config);
  });
});
// (Or even better: use fs.promises with async/await — covered in Topic 16)
```

### Mistake 2 — Doing heavy CPU work on the main thread

```js
// ❌ WRONG — this loop runs on the single JS thread and BLOCKS everything
// All requests freeze until this loop finishes
app.get('/compute', (req, res) => {
  let result = 0;
  for (let i = 0; i < 10_000_000_000; i++) {   // 10 billion iterations — blocks for seconds
    result += i;
  }
  res.send({ result });
});

// ✅ CORRECT — offload CPU work to a worker thread (Topic 28)
const { Worker } = require('worker_threads');
app.get('/compute', (req, res) => {
  const worker = new Worker('./compute-worker.js');   // runs in a separate thread
  worker.on('message', (result) => res.send({ result }));
});
```

### Mistake 3 — Assuming callbacks always run immediately

```js
// ❌ WRONG mental model — thinking the callback runs synchronously
let userData = null;

fs.readFile('./users.json', 'utf8', (err, data) => {
  userData = JSON.parse(data);   // this runs LATER
});

console.log(userData);   // ❌ prints null — callback hasn't run yet!

// ✅ CORRECT — any code that depends on async results must be INSIDE the callback
fs.readFile('./users.json', 'utf8', (err, data) => {
  if (err) throw err;
  const userData = JSON.parse(data);
  console.log(userData);   // ✅ runs after data is available
  processUserData(userData);   // ✅ any dependent logic goes here
});
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that demonstrates non-blocking execution order. It must:
1. Log `"Step 1 — sync"`
2. Use `fs.readFile` to read any file (e.g. `package.json`) — log its byte length inside the callback
3. Log `"Step 2 — sync"` immediately after the `readFile` call (before the callback runs)
4. Use `setTimeout` with 0ms delay — log `"Step 3 — timeout"` inside it

Predict the output order BEFORE running it, then run it and verify.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `loadUserProfile(userId, callback)` that simulates loading a user profile by:
1. First "reading a config file" — use `setTimeout(fn, 50)` to simulate async file read
2. Inside that callback, "querying the database" — use another `setTimeout(fn, 100)` to simulate a DB query
3. The final callback should receive an object: `{ userId, config: 'loaded', dbRecord: 'found' }`

Call `loadUserProfile('user_42', (result) => console.log(result))` and verify the output.

This is the classic **callback chain** — you will feel the pain of nesting, which is why Promises and async/await were invented.

```js
// Write your code here
```

---

### Exercise 3 — hard

You are debugging a slow API. Write a script that:

1. Defines a function `simulateBlockingEndpoint()` that runs a CPU loop for ~500ms (use a `for` loop with enough iterations — find a number that takes roughly 500ms on your machine using `Date.now()`)
2. Defines a function `simulateAsyncEndpoint()` that uses `setTimeout(fn, 500)` 
3. Simulates 5 concurrent "requests" to the **blocking** version — measures and logs total elapsed time for all 5 to complete
4. Simulates 5 concurrent "requests" to the **async** version — measures and logs total elapsed time for all 5 to complete
5. Prints a summary explaining WHY the times are different

The blocking version should take ~2500ms total (5 × 500ms sequential).  
The async version should take ~500ms total (all 5 run concurrently in libuv).

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
THREADING MODEL
  Single JS thread     → one call stack, one thing at a time in JavaScript
  libuv thread pool    → 4 threads by default (UV_THREADPOOL_SIZE to change)
  OS async I/O         → networking uses OS-level async (no thread pool needed)

THE FLOW
  sync code → call stack
  async I/O → libuv (thread pool or OS async) → callback queue → event loop → call stack

BLOCKING vs NON-BLOCKING
  Blocking  → fs.readFileSync, crypto.pbkdf2Sync, heavy for loops
  Non-block → fs.readFile, http requests, setTimeout, setInterval, Promises

WHAT GOES TO LIBUV THREAD POOL
  File system (fs.*), DNS lookup, crypto (pbkdf2, scrypt), zlib compression

WHAT USES OS ASYNC (no thread pool)
  TCP/UDP networking, pipes — these are truly non-blocking at the OS level

WHY NODE IS FAST FOR I/O
  Thread sits idle 0ms — it registers a callback and moves on immediately
  1 JS thread can manage thousands of concurrent in-flight I/O operations

WHY NODE IS BAD FOR CPU WORK
  A for loop with 1 billion iterations holds the thread → all requests freeze
  Solution: worker_threads (Topic 28) or child_process (Topic 27)

COMPARISON
  Java/Spring   → thread per request (blocking, high memory)
  Python/Django → GIL, often thread-per-request or gevent
  Node.js       → single thread, event loop, great for I/O concurrency
  Go            → goroutines (lightweight threads), great for both I/O and CPU
```

---

## Connected topics

- **03 — The event loop in depth** — the exact mechanism (call stack, queues, phases) that makes non-blocking work
- **27 — timers in Node** — how `setTimeout` and `setImmediate` fit into the event loop
- **28 — worker_threads** — the solution for CPU-heavy work on a single-threaded runtime
