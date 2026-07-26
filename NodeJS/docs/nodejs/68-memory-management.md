# 68 — Memory management

## What is this?

Memory management is how Node.js allocates space for the objects, strings, and buffers your code creates, and how it reclaims that space once nothing needs it anymore — a process called **garbage collection (GC)**. Every value you create (an object, an array, a closure) lives in a region called the **heap**, and V8 (the engine Node runs on) periodically scans the heap to find and free memory that is no longer reachable from your running code. Think of the heap like a shared office desk — you keep piling papers (objects) on it as you work, and a cleaning crew (the garbage collector) periodically clears away any papers nobody is referencing anymore. If you keep a paper pinned under a stapler that never lets go (a lingering reference), the cleaning crew can never remove it — that is a **memory leak**.

---

## Why does it matter for backend development?

A backend server runs for weeks or months without restarting, handling thousands of requests. If every request leaves behind a tiny bit of memory that never gets freed — a forgotten event listener, a growing cache, a global array that only grows — that memory adds up until the process runs out of heap and crashes with `JavaScript heap out of memory`, or gets OOM-killed by the OS/orchestrator (Kubernetes, PM2). Unlike a CLI script that runs once and exits, a server's memory bugs are invisible in development and only show up under real, sustained traffic — often days after deployment. Understanding the heap, GC, and how to take heap snapshots is what lets you diagnose "why does memory keep climbing in production" instead of just restarting the pod and hoping.

---

## Syntax / API

```js
// process.memoryUsage() — the single most useful memory API for backend devs
const memoryUsage = process.memoryUsage();

console.log(memoryUsage);
// {
//   rss: 52428800,        // Resident Set Size — total memory OS allocated to this process (heap + code + stack)
//   heapTotal: 20000000,  // total size of the V8 heap currently allocated
//   heapUsed: 15000000,   // portion of heapTotal actually in use right now
//   external: 1000000,    // memory used by C++ objects bound to JS (e.g. Buffers)
//   arrayBuffers: 500000, // memory used by ArrayBuffers and Buffers specifically
// }

// Convert bytes to MB for readable logs — you'll do this constantly
function toMB(bytes) {
  return `${(bytes / 1024 / 1024).toFixed(2)} MB`; // human-readable size
}

console.log('Heap used:', toMB(memoryUsage.heapUsed)); // e.g. "Heap used: 14.31 MB"

// ── Forcing garbage collection manually (debugging only) ───────────────────
// Requires starting node with: node --expose-gc server.js
if (global.gc) {
  global.gc(); // forces a GC pass — never do this in production code, debugging only
}

// ── v8 module — inspect heap statistics ─────────────────────────────────────
const v8 = require('v8'); // built-in module for V8 engine internals

const heapStats = v8.getHeapStatistics();
console.log(heapStats.heap_size_limit); // the max heap size V8 will allow before crashing

// ── Writing a heap snapshot to disk for offline analysis ───────────────────
const inspector = require('inspector'); // built-in Node debugging/profiling module
const fs = require('fs');

function takeHeapSnapshot(filePath) {
  const session = new inspector.Session();  // create a debugging session
  session.connect();                         // attach to the running process

  const fileStream = fs.createWriteStream(filePath); // where the snapshot chunks are written

  session.on('HeapProfiler.addHeapSnapshotChunk', (message) => {
    fileStream.write(message.params.chunk); // V8 streams the snapshot in chunks
  });

  session.post('HeapProfiler.takeHeapSnapshot', null, () => {
    session.disconnect(); // done — snapshot is fully written to filePath
  });
}
```

---

## How it works — line by line

Node.js doesn't manage memory itself — it delegates to **V8**, the JavaScript engine also used in Chrome. V8 organizes memory into a few main areas:

- **Heap** — where objects, closures, arrays, and strings live. This is the area GC cleans up.
- **Stack** — where function call frames and primitive local variables live; cleaned up automatically when a function returns.
- **External memory** — memory outside the JS heap but tracked by V8, mainly `Buffer` and `ArrayBuffer` data (used heavily by streams and file/network I/O).

The heap itself is split into **generations**, because most objects die young (a request handler's temporary variables) while a few live forever (a database connection pool):

- **Young generation (Scavenge)** — small, fast-collecting space for newly created objects. Most garbage is collected here, cheaply and frequently.
- **Old generation (Mark-Sweep-Compact)** — objects that survive a few young-generation collections get "promoted" here. Collecting this space is slower and pauses your event loop briefly (this is why big old-gen collections can cause request latency spikes).

Garbage collection works by **reachability**, not by counting references: V8 starts from "roots" (global variables, currently executing function scopes, the call stack) and walks every reference it can find. Anything it cannot reach is garbage and gets freed. A **memory leak** in Node.js almost always means: something is still reachable from a root that you *believed* was no longer needed — a listener still attached to a long-lived `EventEmitter`, an array that only ever gets `.push()`-ed, or a `Map` used as a cache with no eviction.

`process.memoryUsage()` reports V8's own bookkeeping numbers so you can watch trends in production without special tooling. A **heap snapshot** is a full point-in-time dump of every object on the heap and what references it — loading it in Chrome DevTools lets you see exactly what is taking up space and who is holding onto it, which is how real leaks get diagnosed instead of guessed at.

---

## Example 1 — basic

```js
// File: src/scripts/memory-basics.js
// Demonstrates reading memory usage and watching the heap grow and shrink.

function toMB(bytes) {
  return `${(bytes / 1024 / 1024).toFixed(2)} MB`; // helper to format bytes as megabytes
}

function logMemory(label) {
  const usage = process.memoryUsage(); // snapshot of current memory numbers
  console.log(
    `[${label}] rss=${toMB(usage.rss)} heapUsed=${toMB(usage.heapUsed)} heapTotal=${toMB(usage.heapTotal)}`
  );
}

logMemory('start'); // baseline before allocating anything

// Allocate a large array of objects — this grows the heap
let bigArray = [];
for (let i = 0; i < 1_000_000; i++) {
  bigArray.push({ id: i, payload: `record-${i}` }); // each iteration adds a small object
}

logMemory('after allocation'); // heapUsed should be noticeably higher now

bigArray = null; // remove the only reference — the array is now unreachable garbage

// Note: memory is not freed instantly — GC runs on its own schedule.
// We can request (not force) a GC pass only when node was started with --expose-gc
if (global.gc) {
  global.gc();                      // requires: node --expose-gc memory-basics.js
  logMemory('after manual gc');     // heapUsed should drop back down close to baseline
} else {
  console.log('Run with --expose-gc to see immediate collection: node --expose-gc memory-basics.js');
}
```

---

## Example 2 — real world backend use case

```js
// File: src/middleware/memoryMonitor.js
// A middleware + background watcher that logs memory trends and warns
// before the process gets OOM-killed — a real pattern used in production Node services.

const EventEmitter = require('events'); // used to emit a warning event other code can react to

const memoryEvents = new EventEmitter(); // internal bus — server.js can listen for 'high-memory'

const HEAP_WARNING_MB = 500;  // warn once heapUsed crosses this threshold
const CHECK_INTERVAL_MS = 30_000; // check every 30 seconds — cheap enough to run forever

function toMB(bytes) {
  return Math.round(bytes / 1024 / 1024); // whole-number MB for concise logs
}

function startMemoryWatcher(logger) {
  // setInterval keeps checking memory for the lifetime of the process
  const timer = setInterval(() => {
    const usage = process.memoryUsage();
    const heapUsedMB = toMB(usage.heapUsed);
    const rssMB = toMB(usage.rss);

    logger.info(`memory heapUsed=${heapUsedMB}MB rss=${rssMB}MB`); // structured log, topic 63

    if (heapUsedMB > HEAP_WARNING_MB) {
      // Emit so the rest of the app (alerting, metrics) can react without this module knowing about them
      memoryEvents.emit('high-memory', { heapUsedMB, rssMB });
      logger.warn(`heap usage crossed ${HEAP_WARNING_MB}MB threshold`);
    }
  }, CHECK_INTERVAL_MS);

  timer.unref(); // don't let this timer keep the process alive on its own during shutdown

  return () => clearInterval(timer); // return a stop function for graceful shutdown (topic 64)
}

// Express middleware — attaches current memory numbers to every response for an ops dashboard
function memoryHeaderMiddleware(requestBody, res, next) {
  const usage = process.memoryUsage();
  res.setHeader('X-Heap-Used-MB', toMB(usage.heapUsed)); // visible in browser devtools / curl -i
  next(); // hand off to the next middleware in the chain (topic 55)
}

module.exports = { startMemoryWatcher, memoryHeaderMiddleware, memoryEvents };

// Usage in server.js:
// const { startMemoryWatcher, memoryEvents } = require('./middleware/memoryMonitor');
// memoryEvents.on('high-memory', ({ heapUsedMB }) => {
//   // send an alert to Slack/PagerDuty in a real system
//   console.error(`ALERT: heap usage is ${heapUsedMB}MB`);
// });
// const stopWatcher = startMemoryWatcher(logger);
// process.on('SIGTERM', () => stopWatcher()); // clean shutdown, topic 64
```

---

## Common mistakes

### Mistake 1 — An unbounded in-memory cache that never evicts

```js
// ❌ WRONG — cache grows forever, one entry per unique userId, never removed
const userSessionCache = {};

function cacheSession(userId, sessionData) {
  userSessionCache[userId] = sessionData; // nothing ever deletes old entries
  // After weeks of traffic, this object holds millions of stale sessions in the heap
}

// ✅ CORRECT — use a Map with a maximum size and evict the oldest entry (simple LRU-ish pattern)
const MAX_CACHE_SIZE = 10_000;
const sessionCache = new Map();

function cacheSession(sessionId, sessionData) {
  if (sessionCache.size >= MAX_CACHE_SIZE) {
    const oldestKey = sessionCache.keys().next().value; // Map preserves insertion order
    sessionCache.delete(oldestKey);                     // evict oldest before inserting new
  }
  sessionCache.set(sessionId, sessionData);
  // For production, prefer a real cache like Redis (topic 69) or the lru-cache package
}
```

### Mistake 2 — Forgetting to remove event listeners on short-lived objects

```js
// ❌ WRONG — a new listener is added to a long-lived emitter on every request,
// and none are ever removed — this is a classic Node.js memory leak
const EventEmitter = require('events');
const appEvents = new EventEmitter(); // lives for the entire process lifetime

function handleRequest(requestBody, res) {
  appEvents.on('order-placed', () => {
    res.write('order confirmed\n'); // this closure captures `res` and never releases it
  });
  // Every request adds one more listener — appEvents.listenerCount() grows forever
}

// ✅ CORRECT — use `.once()` so the listener removes itself after firing, or explicitly remove it
function handleRequest(requestBody, res) {
  const onOrderPlaced = () => {
    res.write('order confirmed\n');
  };

  appEvents.once('order-placed', onOrderPlaced); // auto-removed after first emit

  res.on('close', () => {
    appEvents.off('order-placed', onOrderPlaced); // also remove if the client disconnects early
  });
}
```

### Mistake 3 — Buffering an entire large response in memory instead of streaming it

```js
// ❌ WRONG — reads the whole file into a Buffer/string before sending it
// A 2GB export file means a 2GB spike in heap/external memory per concurrent request
const fs = require('fs');

function downloadExport(requestBody, res) {
  const fileContent = fs.readFileSync('./exports/full-report.csv', 'utf8'); // entire file in RAM
  res.end(fileContent);
}

// ✅ CORRECT — stream the file so only small chunks live in memory at any moment (topic 17, 26)
function downloadExport(requestBody, res) {
  const readStream = fs.createReadStream('./exports/full-report.csv'); // reads in small chunks
  readStream.pipe(res); // pipes chunks straight to the response, applying backpressure automatically

  readStream.on('error', (err) => {
    res.destroy(err); // never leave a broken stream hanging — release its resources
  });
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Logs `process.memoryUsage().heapUsed` in MB before doing anything
2. Creates an array of 500,000 objects, each with an `id` and a random string field
3. Logs `heapUsed` again after creating the array
4. Sets the array reference to `null`
5. If `global.gc` is available (run with `node --expose-gc yourFile.js`), calls it and logs `heapUsed` one more time
6. Prints a short summary comparing all three readings

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a `BoundedCache` class that behaves like a simple in-memory cache but never grows unbounded:
1. Constructor takes `maxSize` (a number)
2. Has a `set(key, value)` method — if the cache is already at `maxSize`, it must evict the **oldest inserted** key before adding the new one
3. Has a `get(key)` method that returns the value or `undefined`
4. Has a `has(key)` method
5. Has a `size()` method returning the current number of entries
6. Write a test loop that inserts 10,000 keys into a `BoundedCache` with `maxSize: 100`, then confirms `size()` never exceeds 100 and that only the most recent 100 keys are still present

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a memory-leak reproduction and detection tool:
1. Write a function `leakyHandler()` that simulates a leak by pushing a new object into a module-level array on every call, and never clearing it
2. Write a function `healthyHandler()` that does equivalent work but keeps no references after it returns
3. Write a `MemorySampler` class with:
   - a `start(intervalMs)` method that samples `process.memoryUsage().heapUsed` on an interval and stores each sample with a timestamp
   - a `stop()` method that stops sampling
   - a `getTrend()` method that returns `'growing'`, `'stable'`, or `'shrinking'` based on comparing the average of the first third of samples to the average of the last third of samples
4. Run a simulation: call `leakyHandler()` in a loop with the sampler running, and confirm `getTrend()` reports `'growing'`
5. Run a second simulation with `healthyHandler()` and confirm `getTrend()` reports `'stable'`

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
MEMORY REGIONS (V8)
  Heap              → objects, arrays, closures, strings — what GC manages
  Stack             → function call frames, primitives — freed on function return
  External memory   → Buffers, ArrayBuffers — tracked outside the JS heap

HEAP GENERATIONS
  Young gen (Scavenge)        → new objects, fast + frequent collection
  Old gen (Mark-Sweep-Compact) → long-lived objects, slower, can pause event loop briefly

GARBAGE COLLECTION RULE
  An object is garbage when it is NOT REACHABLE from a root
  (global scope, active call stack, currently referenced closures)
  GC does NOT use reference counting — it uses reachability

process.memoryUsage() FIELDS
  rss           → total memory OS gave the process (heap + stack + code + external)
  heapTotal     → total heap V8 has allocated
  heapUsed      → heap actually in use right now (the number to watch over time)
  external      → memory for C++ objects bound to JS (mostly Buffers)
  arrayBuffers  → memory for ArrayBuffers/Buffers specifically

COMMON CAUSES OF LEAKS
  - Unbounded caches / objects that only grow (Map, plain object, array)
  - Event listeners added repeatedly, never removed (.on() without .off()/.once())
  - Closures capturing large objects (e.g. `res`, `req`) longer than needed
  - Global arrays/variables used as "temporary" storage that never gets cleared
  - Timers (setInterval) that are never cleared, keeping their closure alive forever

DIAGNOSING LEAKS
  node --inspect server.js              → attach Chrome DevTools (chrome://inspect)
  Take 2-3 heap snapshots over time     → compare in DevTools "Comparison" view
  Growing "Retained Size" for one type  → that's your leak — check who "holds" it
  node --expose-gc file.js              → allows calling global.gc() manually for testing

USEFUL BUILT-IN APIS
  process.memoryUsage()                 → quick numeric snapshot, safe for prod logging
  require('v8').getHeapStatistics()     → heap limits and space breakdown
  require('v8').writeHeapSnapshot()     → writes a .heapsnapshot file to disk directly
  require('inspector')                  → programmatic access to Chrome DevTools protocol

NEVER DO
  Force GC in production request paths      → global.gc() is a debugging tool only
  Buffer huge files/responses fully in RAM   → stream them instead (topic 17, 26)
  Add listeners per-request to shared emitters without removing them
  Assume memory dropping to zero after `= null` happens instantly — GC runs on its own schedule
```

---

## Connected topics

- **67 — Profiling Node.js** — `--inspect` and Chrome DevTools are the same tools used to capture and read heap snapshots for memory debugging
- **26 — Stream module in depth** — streaming data instead of buffering it in memory is the primary way backend code avoids heap growth on large payloads
- **69 — Caching strategies** — unbounded in-process caches are one of the most common real-world memory leaks; this topic covers doing caching safely with TTL and eviction
