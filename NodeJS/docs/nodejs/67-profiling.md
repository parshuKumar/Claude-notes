# 67 — Profiling Node.js

## What is this?

Profiling is the process of measuring *where* your Node.js app actually spends its CPU time and memory, instead of guessing. Node ships with built-in tools (`--inspect`, `--prof`) that record what every function call costs, and third-party tools like `clinic.js` turn that raw data into readable flame graphs. Think of it like a mechanic hooking a diagnostic scanner to a car engine — instead of guessing why the car is slow, the scanner tells you exactly which part is overheating.

## Why does it matter for backend development?

"The API feels slow" is not actionable — you need to know *which* function, *which* database call, or *which* loop is eating the CPU before you can fix it. Backend developers profile production-like traffic to find CPU hotspots (a JSON.stringify on a huge object, a synchronous regex, an O(n²) loop) that block the single-threaded event loop and slow down every concurrent request, not just one. This is also a common system-design/performance interview topic — "how would you find why this endpoint is slow?" — and the honest answer is always "I'd profile it," not "I'd guess."

---

## Syntax / API

```js
// ── 1. Chrome DevTools inspector (best for live, interactive debugging) ─────
// Run your app with the --inspect flag:
//   node --inspect server.js
// Node prints a ws:// URL and starts listening on port 9229 for a debugger.
// Open chrome://inspect in Chrome, click "inspect" under Remote Target.

// --inspect-brk pauses execution on the very first line — useful to attach
// the debugger before any startup code runs:
//   node --inspect-brk server.js

// ── 2. node --prof (built-in V8 sampling profiler, best for CPU hotspots) ──
// Run your app normally, generate real traffic, then stop it (Ctrl+C):
//   node --prof server.js
// This writes a binary log file: isolate-0x.....-v8.log in the cwd

// Convert the raw log into a human-readable summary:
//   node --prof-process isolate-0x1234-v8.log > processed.txt
// processed.txt now shows a "Summary" and "Bottom up (heavy) profile"
// ranking every function by % of total ticks (CPU samples) it consumed

// ── 3. clinic.js (third-party, best for flame graphs + auto diagnosis) ─────
// Install globally or as a devDependency:
//   npm install -g clinic
// clinic doctor: overall health check (event loop delay, CPU, memory)
//   clinic doctor -- node server.js
// clinic flame: generates an interactive flame graph of CPU hotspots
//   clinic flame -- node server.js
// Both open an HTML report in the browser after you stop the process (Ctrl+C)
```

---

## How it works — line by line

`node --inspect` opens a debugger port and pauses execution when you set breakpoints — the same DevTools UI you already know from browser JavaScript, but attached to your server process. You step through code, inspect variables, and take live CPU/heap snapshots while the process runs.

`node --prof` turns on V8's built-in sampling profiler. Every few milliseconds, V8 "samples" what function is currently executing on the call stack and writes that sample to a `.log` file. After running real traffic through the app, you feed that log through `--prof-process`, which counts how many samples landed in each function — the functions with the most samples are your CPU hotspots.

`clinic.js` wraps Node's built-in tools (and adds its own instrumentation) to produce visual reports automatically. `clinic flame` builds a **flame graph** — a stacked bar chart where the width of each bar represents how much time was spent in that function and its children. Wide bars at the top of the stack are exactly where your CPU time is going.

The general workflow for all three tools is the same: **reproduce load → capture data → read the report → fix the hotspot → re-measure**. Profiling without generating realistic load (using a tool like `autocannon`, covered in Topic 72) usually shows you nothing interesting, because an idle server has no hotspots.

---

## Example 1 — basic

```js
// File: src/cpu-heavy.js
// A deliberately CPU-heavy function to profile — simulates a slow endpoint.

function isPrime(n) {
  // Naive primality check — intentionally slow for large n (our "hotspot")
  if (n < 2) return false;
  for (let i = 2; i <= Math.sqrt(n); i++) {
    if (n % i === 0) return false;   // not prime — divisible by i
  }
  return true;                       // no divisors found — it's prime
}

function countPrimesUpTo(limit) {
  let count = 0;                     // running total of primes found
  for (let n = 2; n <= limit; n++) {
    if (isPrime(n)) count++;         // check every number up to the limit
  }
  return count;
}

// Run this with:  node --prof src/cpu-heavy.js
const result = countPrimesUpTo(2_000_000);   // heavy synchronous CPU work
console.log(`Primes found: ${result}`);       // print the result when done

// After it finishes, a file like isolate-0x...-v8.log appears in the cwd.
// Process it with:
//   node --prof-process isolate-0x*-v8.log > processed.txt
// Open processed.txt and look at the "Bottom up (heavy) profile" section —
// isPrime() will show up near the top consuming most of the ticks.
```

---

## Example 2 — real world backend use case

```js
// File: src/server.js
// An Express-style endpoint with a hidden CPU hotspot — the kind of code
// a backend dev profiles after users report "the reports endpoint is slow".

const http = require('http');

// Simulates loading a large dataset (e.g. from a DB query result)
function loadOrders() {
  const orders = [];
  for (let i = 0; i < 50_000; i++) {
    orders.push({ orderId: i, userId: `user_${i % 500}`, total: Math.random() * 100 });
  }
  return orders;
}

// HOTSPOT: recomputes a full summary on every single request using a
// nested loop — O(n * m) instead of a single O(n) pass with a map.
function buildUserTotalsSlow(orders) {
  const uniqueUserIds = [...new Set(orders.map(order => order.userId))];
  return uniqueUserIds.map(userId => {
    // For EVERY user, we re-scan the ENTIRE orders array — this is the hotspot
    const userOrders = orders.filter(order => order.userId === userId);
    const total = userOrders.reduce((sum, order) => sum + order.total, 0);
    return { userId, total };
  });
}

const orders = loadOrders();   // pretend this came from a database query

const server = http.createServer((req, res) => {
  if (req.url === '/reports/user-totals') {
    // This line is what shows up as the wide bar in a clinic flame graph
    const report = buildUserTotalsSlow(orders);
    res.setHeader('Content-Type', 'application/json');
    res.end(JSON.stringify(report));
    return;
  }
  res.writeHead(404);
  res.end();
});

server.listen(3000, () => {
  console.log('Server running — hit it with load, then profile it:');
  console.log('  clinic flame -- node src/server.js');
  console.log('  autocannon http://localhost:3000/reports/user-totals');
});

// After running clinic flame + autocannon, the flame graph reveals
// buildUserTotalsSlow → Array.prototype.filter as the dominant hot path.
// The fix is a single-pass reduce keyed by userId (O(n) instead of O(n*m)):
//
// function buildUserTotalsFast(orders) {
//   const totalsByUser = new Map();
//   for (const order of orders) {
//     const current = totalsByUser.get(order.userId) || 0;
//     totalsByUser.set(order.userId, current + order.total);
//   }
//   return [...totalsByUser].map(([userId, total]) => ({ userId, total }));
// }
```

---

## Common mistakes

### Mistake 1 — Profiling with no real load, then trusting the report

```js
// ❌ WRONG — starting the profiler and immediately hitting Ctrl+C
// with zero requests sent means the report is empty/misleading
// $ clinic flame -- node server.js
// (never sends a request, stops after 2 seconds)
// → flame graph shows only Node startup cost, hides the actual bottleneck

// ✅ CORRECT — generate realistic concurrent traffic while profiling
// Terminal 1:
//   clinic flame -- node server.js
// Terminal 2 (while it's running):
//   npx autocannon -c 50 -d 20 http://localhost:3000/reports/user-totals
// Then stop the server (Ctrl+C in Terminal 1) — clinic opens the real report
```

### Mistake 2 — Running --prof and --inspect in production without a plan

```js
// ❌ WRONG — attaching --inspect on a public production port with no
// authentication exposes a full remote debugger to the internet
// node --inspect=0.0.0.0:9229 server.js   // DANGEROUS: anyone can attach
// and run arbitrary code inside your process via the debugger protocol

// ✅ CORRECT — bind the inspector to localhost only, and tunnel via SSH
// node --inspect=127.0.0.1:9229 server.js
// Then from your local machine:
//   ssh -L 9229:127.0.0.1:9229 user@production-host
// Now only you (through the SSH tunnel) can reach the debugger port
```

### Mistake 3 — Reading the profiler's "self time" wrong

```js
// ❌ WRONG — assuming the top function in the summary is always the bug
// Example: JSON.stringify shows up as 40% of ticks in --prof-process output.
// A dev "fixes" it by removing JSON.stringify from logging calls,
// without checking WHAT is being stringified.

// ✅ CORRECT — check "self time" vs "total time" and trace the caller chain
// node --prof-process processed.txt
// Look at the "Bottom up (heavy) profile" — it shows the CALLERS of
// JSON.stringify. Usually the real bug is upstream:
function logRequestBody(requestBody) {
  // The actual hotspot is building a huge object before stringifying it,
  // not JSON.stringify itself — fix the object construction, not the call:
  const summary = { userId: requestBody.userId, action: requestBody.action };
  console.log(JSON.stringify(summary));   // stringify a small summary, not requestBody
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a script `src/slow-sort.js` that:
1. Builds an array of 500,000 random numbers
2. Sorts a copy of it using a deliberately slow bubble sort function you write yourself
3. Logs how many milliseconds the sort took using `console.time` / `console.timeEnd`

Then run it with `node --prof src/slow-sort.js`, process the log with `node --prof-process`, and identify which function consumed the most ticks in the output. Write your findings as a comment at the bottom of the file.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a small HTTP server (`src/report-server.js`, no framework needed) with a single route `/summary` that loads an in-memory array of 100,000 `{ userId, amount }` transaction objects and computes the total amount per user using an inefficient nested-loop approach (similar to Example 2, but write your own version — don't copy it).

Then:
1. Install `clinic` (`npm install -g clinic`) and `autocannon` (`npm install -g autocannon`)
2. Run `clinic flame -- node src/report-server.js`
3. In another terminal, hammer the `/summary` route with `autocannon`
4. Stop the server and open the generated flame graph
5. Rewrite the summary logic to be efficient (single pass with a `Map`) and re-profile to confirm the hotspot shrank

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small "profiling toolkit" module `src/profileMiddleware.js` for an HTTP server that, without using any external profiling tool:
1. Exports a function `withTiming(handler)` that wraps a request handler function
2. Measures how long the wrapped handler takes using `process.hrtime.bigint()` (start and end)
3. Logs a structured line for every request: `{ url, durationMs, timestamp }`
4. Keeps an in-memory rolling record of the slowest 10 requests seen so far (sorted by `durationMs` descending)
5. Exposes a second function `getSlowestRequests()` that returns that top-10 list
6. Wire it into a real server with at least two routes — one fast, one artificially slow (e.g. a busy-wait loop) — and prove that after sending traffic to both, `getSlowestRequests()` correctly reports the slow route at the top

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
TOOLS OVERVIEW
  --inspect            → attach Chrome DevTools, live debugging + breakpoints
  --inspect-brk         → same, but pauses on the very first line
  --prof                → V8 sampling profiler, writes isolate-*.log
  --prof-process        → converts the .log into a readable text summary
  clinic doctor          → overall health report (event loop delay, CPU, mem)
  clinic flame           → interactive CPU flame graph (best for hotspots)
  clinic bubbleprof      → visualizes async operations and their delays
  clinic heapprofiler    → memory allocation profiling

WORKFLOW
  1. Reproduce production-like load (autocannon / real traffic)
  2. Start profiler BEFORE or DURING that load, not on an idle server
  3. Stop the process cleanly (Ctrl+C) to flush the report
  4. Read the report — widest bars / highest "self time" = hotspot
  5. Fix the hotspot, re-run the same load, re-profile, compare

READING node --prof-process OUTPUT
  "Summary"              → % of ticks in JS, C++, GC, etc.
  "Bottom up (heavy)"     → functions ranked by self time — start here
  ticks / total          → higher % = more CPU time spent in that function

CHROME DEVTOOLS (via --inspect)
  chrome://inspect            → open in Chrome to see remote targets
  Sources tab                 → set breakpoints, step through code
  Performance tab             → record CPU profile while clicking through app
  Memory tab                  → take heap snapshots (see Topic 68)

SECURITY
  NEVER bind --inspect to 0.0.0.0 on a public server
  Bind to 127.0.0.1 and use an SSH tunnel to reach it remotely

WHEN TO USE WHICH
  Quick "why is this one request slow?"     → --inspect + DevTools breakpoints
  "Find the CPU hotspot under load"          → clinic flame or --prof
  "Is my app healthy overall?"                → clinic doctor
  "Something async is slow, not CPU-bound"    → clinic bubbleprof
```

---

## Connected topics

- **68 — Memory management** — profiling CPU hotspots is half the story; heap snapshots and GC analysis (the natural next topic) find memory-based slowdowns
- **72 — Load testing** — tools like `autocannon` generate the realistic traffic you need before a profiling session shows anything meaningful
- **28 — worker_threads module** — once profiling identifies a genuine CPU-bound hotspot, moving that work to a worker thread is the standard fix so it stops blocking the event loop
