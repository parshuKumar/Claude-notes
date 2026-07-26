# 28 — worker_threads module

## What is this?

`worker_threads` is a built-in Node.js module that lets you run JavaScript on **separate threads** inside the same process, each with its own V8 instance and event loop. Normally Node.js runs your code on a single thread — if you do heavy math or process a huge file synchronously, the whole server freezes for every user. Worker threads solve this by giving CPU-heavy work its own thread, like hiring a second cashier who only handles complicated returns while the first cashier keeps serving the regular checkout line without interruption.

---

## Why does it matter for backend development?

A backend server's main thread must stay free to accept requests, query databases, and respond to clients — that is its one job. The moment you run something CPU-intensive (image resizing, PDF generation, password hashing, large JSON parsing, video encoding) directly on the main thread, every other request queues up behind it and your API's response times spike. `worker_threads` moves that CPU-bound work off the main thread so requests keep flowing. It is the correct tool specifically for **CPU-bound** work — unlike `child_process`, workers share memory efficiently via `SharedArrayBuffer` and are much lighter weight to spin up, making them the modern choice for in-process parallel computation.

---

## Syntax / API

```js
// worker_threads is a Node.js core module — no install needed
const {
  Worker,           // class used to create a new worker thread
  isMainThread,     // boolean — true if this code runs on the main thread
  parentPort,       // MessagePort — used INSIDE a worker to talk back to the main thread
  workerData,       // data passed in when the worker was created
} = require('worker_threads');

// ── Creating a worker from the main thread ──────────────────────────────────
if (isMainThread) {
  // Spin up a new worker, pointing at a separate file that contains its logic
  const worker = new Worker('./workers/hashPassword.js', {
    workerData: { plainPassword: 'my-secret-123' }, // data sent to the worker on startup
  });

  // Listen for messages the worker sends back
  worker.on('message', (result) => {
    console.log('Worker returned:', result); // e.g. the hashed password
  });

  // Listen for errors thrown inside the worker
  worker.on('error', (err) => {
    console.error('Worker crashed:', err);
  });

  // Fires when the worker thread finishes and shuts down
  worker.on('exit', (exitCode) => {
    console.log('Worker exited with code:', exitCode); // 0 = clean exit
  });
}

// ── Inside the worker file (hashPassword.js) ────────────────────────────────
// const { parentPort, workerData } = require('worker_threads');
// const crypto = require('crypto');
//
// const hash = crypto.createHash('sha256').update(workerData.plainPassword).digest('hex');
// parentPort.postMessage(hash); // send the result back to the main thread
```

---

## How it works — line by line

- `require('worker_threads')` pulls in the built-in module — nothing to install.
- `isMainThread` tells you which "side" of the code is currently running. Worker files often reuse the *same file* for both the main-thread setup and the worker logic, so this check decides which branch to run.
- `new Worker('./path/to/file.js', options)` starts a brand-new thread that loads and runs that file from scratch, with its own independent V8 instance and its own event loop — it does not share variables or the call stack with the main thread.
- `workerData` is how you hand data to the worker at creation time — it gets cloned (copied) into the worker's memory, not shared by reference.
- Inside the worker file, `parentPort` is the communication channel back to whoever created it. Calling `parentPort.postMessage(data)` sends a message; the main thread receives it via `worker.on('message', ...)`.
- `worker.on('error', ...)` catches any uncaught exception inside the worker so one crashing thread doesn't crash your whole server.
- `worker.on('exit', ...)` fires once the worker's code finishes running (or is terminated) — exit code `0` means it finished cleanly.
- Communication between threads is done through **message passing** (structured clone algorithm) by default — data is copied, not shared — which is safe but has a copying cost for large payloads. `SharedArrayBuffer` is the exception: it lets multiple threads read/write the *same* block of raw memory without copying.

---

## Example 1 — basic

```js
// File: main.js
// Demonstrates the minimal worker lifecycle: create, receive, handle errors, clean up.

const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

if (isMainThread) {
  // We are on the MAIN thread — create a worker using THIS SAME FILE
  const worker = new Worker(__filename, {
    workerData: { number: 42 }, // data passed into the worker
  });

  // Fired when the worker sends data back via postMessage
  worker.on('message', (squaredResult) => {
    console.log('Result from worker:', squaredResult); // → 1764
  });

  // Fired if the worker throws an uncaught error
  worker.on('error', (err) => {
    console.error('Worker error:', err.message);
  });

  // Fired when the worker thread finishes execution
  worker.on('exit', (code) => {
    console.log('Worker finished with exit code:', code); // → 0
  });

} else {
  // We are INSIDE the worker thread — do the actual work
  const { number } = workerData;         // read the data passed in
  const squared = number * number;        // simple CPU work, stand-in for something heavy
  parentPort.postMessage(squared);        // send the result back to the main thread
}
```

---

## Example 2 — real world backend use case

```js
// File: workers/pdfReportWorker.js
// A worker dedicated to generating a heavy PDF report without blocking the API's event loop.

const { parentPort, workerData } = require('worker_threads');

function buildReportRows(orders) {
  // Simulate CPU-heavy formatting/aggregation work over many orders
  return orders.map((order) => ({
    orderId: order.orderId,
    total: order.items.reduce((sum, item) => sum + item.price * item.qty, 0),
  }));
}

const { orders } = workerData;               // the raw order data handed in by the API route
const rows = buildReportRows(orders);         // do the expensive number crunching here, off the main thread

// Send the finished, lightweight result back — not the raw heavy dataset
parentPort.postMessage({ rows, generatedAt: Date.now() });
```

```js
// File: routes/reports.js
// Express route that offloads report generation to a worker so the API stays responsive.

const { Worker } = require('worker_threads');
const path = require('path');

function generateReportAsync(orders) {
  // Wrap the worker in a Promise so route handlers can simply `await` it
  return new Promise((resolve, reject) => {
    const worker = new Worker(path.join(__dirname, '..', 'workers', 'pdfReportWorker.js'), {
      workerData: { orders }, // pass the orders fetched from the database
    });

    worker.on('message', (report) => resolve(report));   // success path
    worker.on('error', (err) => reject(err));             // failure path
    worker.on('exit', (exitCode) => {
      if (exitCode !== 0) {
        // Non-zero exit with no message already sent means something went wrong silently
        reject(new Error(`Report worker stopped with exit code ${exitCode}`));
      }
    });
  });
}

// Express handler
app.get('/api/orders/:userId/report', async (req, res) => {
  const { userId } = req.params;

  try {
    const orders = await orderService.findAllByUserId(userId); // fetch from DB — cheap, async
    const report = await generateReportAsync(orders);           // CPU-heavy — runs off-thread
    res.json({ userId, report });
  } catch (err) {
    console.error('Report generation failed:', err);
    res.status(500).json({ error: 'Could not generate report' });
  }
});
```

---

## Common mistakes

### Mistake 1 — Doing CPU-heavy work directly on the main thread

```js
// ❌ WRONG — a synchronous, expensive loop blocks EVERY incoming request
app.get('/api/reports/:userId', (req, res) => {
  let total = 0;
  for (let i = 0; i < 5_000_000_000; i++) {
    total += i; // this locks up the entire event loop until it finishes
  }
  res.json({ total });
});

// ✅ CORRECT — offload the heavy computation to a worker thread
app.get('/api/reports/:userId', async (req, res) => {
  const total = await runInWorker('./workers/sumWorker.js', { limit: 5_000_000_000 });
  res.json({ total }); // main thread stayed free to serve other requests meanwhile
});
```

### Mistake 2 — Creating a new worker for every single request

```js
// ❌ WRONG — spinning up a fresh thread per request is expensive and doesn't scale
app.post('/api/images/resize', async (req, res) => {
  const worker = new Worker('./workers/resizeImage.js', { workerData: req.body });
  // creating/destroying threads has real overhead under high traffic
});

// ✅ CORRECT — reuse a pool of long-lived workers (e.g. via the "piscina" package
// or a hand-rolled pool) and hand each request to an idle worker
const { resizeImagePool } = require('./workerPool'); // pre-created pool of workers

app.post('/api/images/resize', async (req, res) => {
  const resized = await resizeImagePool.run(req.body); // reuses an existing thread
  res.json({ resized });
});
```

### Mistake 3 — Passing huge data with postMessage instead of SharedArrayBuffer

```js
// ❌ WRONG — postMessage CLONES the entire array into the worker's memory,
// which is slow and doubles memory usage for large datasets
const worker = new Worker('./workers/analyze.js', {
  workerData: { pixels: hugeUint8Array }, // gets fully copied, not shared
});

// ✅ CORRECT — use SharedArrayBuffer so both threads read/write the SAME memory,
// no copying required
const sharedBuffer = new SharedArrayBuffer(hugeUint8Array.length);
const sharedView    = new Uint8Array(sharedBuffer);
sharedView.set(hugeUint8Array); // fill the shared memory once

const worker = new Worker('./workers/analyze.js', {
  workerData: { sharedBuffer }, // only a reference is passed, memory is shared
});
// Inside the worker: const view = new Uint8Array(workerData.sharedBuffer);
// Both threads now see the exact same underlying bytes
```

---

## Practice exercises

### Exercise 1 — easy

Create a worker that computes the sum of numbers from `1` to `N` (a large number like `1,000,000,000`), where `N` is passed in via `workerData`. The main thread should:
1. Create the worker and pass in `N`
2. Log the result when the worker sends it back
3. Log a message if the worker errors out
4. Log the exit code when the worker finishes

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a small helper function `runInWorker(workerFilePath, data)` that:
1. Returns a Promise
2. Creates a `Worker` pointing at `workerFilePath`, passing `data` as `workerData`
3. Resolves the Promise with whatever the worker sends via `postMessage`
4. Rejects the Promise if the worker emits an `'error'` event
5. Rejects the Promise if the worker exits with a non-zero code and no message was ever received

Then create a worker file `workers/fibonacci.js` that computes the Nth Fibonacci number recursively (intentionally slow, to simulate CPU-heavy work), and call your helper like:
```js
const result = await runInWorker('./workers/fibonacci.js', { n: 35 });
console.log(result);
```

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a simple **worker pool** class `WorkerPool` that:
1. Takes a `workerFilePath` and a `poolSize` (number of workers to keep alive) in its constructor
2. Creates `poolSize` workers up front and keeps them alive (does not create a new worker per task)
3. Has a `run(taskData)` method that returns a Promise — it should hand the task to whichever worker is currently idle, and if all workers are busy, queue the task until one becomes free
4. Correctly routes each worker's response back to the Promise that requested it (careful: multiple tasks may be in flight on different workers at once — do not mix up results)
5. Has a `shutdown()` method that terminates all workers cleanly using `worker.terminate()`

Test it by running 20 CPU-heavy tasks (e.g. computing Fibonacci numbers) through a pool of only 4 workers, and confirm all 20 results come back correctly and no more than 4 workers ever run at the same time.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE PIECES
  const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

  isMainThread   → true on the main thread, false inside a worker
  new Worker(path, opts) → starts a new thread running that file
  workerData     → data cloned into the worker at startup (opts.workerData)
  parentPort     → (inside worker) channel to message the creator
  worker.postMessage(data)      → (main thread) send data TO the worker
  parentPort.postMessage(data)  → (inside worker) send data BACK to main thread

WORKER EVENTS (on the `worker` object, main-thread side)
  'message'  → worker called parentPort.postMessage()
  'error'    → uncaught exception inside the worker
  'exit'     → worker finished; exitCode 0 = clean

WHEN TO USE worker_threads
  ✔ CPU-bound work: hashing, image/video processing, big math, parsing huge files
  ✔ Need to share large binary memory efficiently → SharedArrayBuffer
  ✘ NOT for I/O waiting (DB calls, HTTP requests) — async/await already handles that
    without needing a separate thread at all

worker_threads vs child_process
  worker_threads                         child_process
  ─────────────────────────────────────  ─────────────────────────────────────
  Same process, separate thread          Separate OS process entirely
  Lightweight to spawn                   Heavier to spawn (new process overhead)
  Can share memory (SharedArrayBuffer)   Cannot share memory — IPC only (pipes)
  Crash isolation: partial (same proc)   Crash isolation: full (separate proc)
  Best for: CPU-bound JS computation     Best for: running other programs/binaries,
                                          shell commands, or isolating untrusted code

SharedArrayBuffer
  Raw memory block shared BY REFERENCE across threads — no copying
  Must wrap in a typed array to read/write: new Uint8Array(sharedArrayBuffer)
  Use Atomics.* for safe concurrent read/write without race conditions

GOTCHAS
  - Spawning a worker per request does not scale — use a worker pool
  - workerData is CLONED (structured clone), not shared — use SharedArrayBuffer for
    large shared memory instead
  - Always attach 'error' and 'exit' listeners — an unhandled worker error can
    otherwise go unnoticed
  - Call worker.terminate() to force-stop a worker; do not just let it hang around
```

---

## Connected topics

- **27 — child_process module** — the sibling API for running separate OS processes; know when to reach for each
- **29 — cluster module** — scales across CPU cores at the *process* level for handling more incoming connections, a different problem than offloading CPU work
- **67 — Profiling Node.js** — how to actually find the CPU-heavy hotspots in your code that are good candidates for moving into a worker thread
