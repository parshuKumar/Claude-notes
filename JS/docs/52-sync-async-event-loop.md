# 52 — Sync vs Async + The Event Loop

## What is this?

JavaScript is a **single-threaded language** — it has exactly one call stack, meaning it can only do one thing at a time. Yet somehow it can make network requests, read files, set timers, and handle user input all without freezing. The engine that makes this possible is the **event loop**. Think of it like a restaurant with one chef (the JS engine) and a team of kitchen assistants (the browser/Node.js APIs): the chef can hand off long-running tasks to the assistants and keep cooking other dishes while waiting. When an assistant finishes, they place the result on a pickup counter (the queue). The chef (event loop) checks the counter every time they finish a dish — if something's waiting, they pick it up and cook it next.

## Why does it matter?

Every single async operation you will ever write in JavaScript — `setTimeout`, `fetch`, file reads, database queries, HTTP requests in Node.js — runs through this exact system. If you don't understand the event loop, you will write code that runs in the wrong order, have mysterious bugs where things happen "too early" or "too late", not understand why some promises resolve before others, and completely misread async/await code. Understanding this system once, deeply, removes an entire class of bugs from your life permanently.

---

## The pieces of the system

Before we trace through examples, you need to know all five pieces:

```
┌─────────────────────────────────────────────────────────────────┐
│                      JS ENGINE (V8 etc.)                        │
│                                                                 │
│  ┌──────────────────┐        ┌────────────────────────────┐     │
│  │   CALL STACK     │        │        HEAP                │     │
│  │                  │        │  (memory — objects live    │     │
│  │  [main()]        │        │   here, not relevant to    │     │
│  │  [fn2()]         │        │   execution order)         │     │
│  │  [fn1()]         │        └────────────────────────────┘     │
│  └──────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              BROWSER / NODE.JS RUNTIME (outside JS engine)      │
│                                                                 │
│  ┌─────────────────────┐   ┌──────────────────────────────┐     │
│  │   WEB APIs / C++    │   │   MICROTASK QUEUE            │     │
│  │   setTimeout        │   │   (Promise callbacks,        │     │
│  │   fetch / HTTP      │   │    queueMicrotask,           │     │
│  │   fs.readFile       │   │    MutationObserver)         │     │
│  │   Event listeners   │   └──────────────────────────────┘     │
│  └─────────────────────┘                                        │
│                              ┌──────────────────────────────┐   │
│                              │   MACROTASK QUEUE (Task Q)   │   │
│                              │   (setTimeout callbacks,     │   │
│                              │    setInterval,              │   │
│                              │    I/O callbacks,            │   │
│                              │    setImmediate in Node)     │   │
│                              └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

                    ▲ Event Loop constantly checks:
                    "Is the call stack empty?
                     → drain all microtasks first
                     → then run ONE macrotask
                     → drain all microtasks again
                     → repeat"
```

---

## Piece 1: The Call Stack

The call stack is a **LIFO (Last In, First Out)** data structure that tracks which functions are currently executing. When you call a function, it gets **pushed** onto the stack. When it returns, it gets **popped** off.

```js
function multiply(a, b) {
  return a * b;              // line 2
}

function square(n) {
  return multiply(n, n);     // line 6
}

function printSquare(n) {
  const result = square(n);  // line 10
  console.log(result);       // line 11
}

printSquare(5);
```

Stack trace over time:
```
Step 1: [printSquare(5)]
Step 2: [printSquare(5)] → [square(5)]
Step 3: [printSquare(5)] → [square(5)] → [multiply(5,5)]
Step 4: multiply returns 25 → popped
        [printSquare(5)] → [square(5)]
Step 5: square returns 25 → popped
        [printSquare(5)]
Step 6: console.log(25) pushed + runs + popped
Step 7: printSquare returns → popped
        [] ← empty stack
```

When the stack is empty, the event loop can pick up the next task.

---

## Piece 2: The Heap

Where objects live in memory. Not relevant to execution order — just know that variables hold references into the heap.

---

## Piece 3: Web APIs / Node.js C++ APIs

These are features provided by the **runtime environment** (browser or Node.js), NOT by the JavaScript engine itself:

| API | What it does |
|---|---|
| `setTimeout(cb, ms)` | Schedules `cb` after `ms` milliseconds |
| `setInterval(cb, ms)` | Schedules `cb` repeatedly every `ms` ms |
| `fetch(url)` | Makes HTTP request |
| `fs.readFile(...)` | Reads file (Node.js) |
| `addEventListener` | Registers DOM event listener |
| `XMLHttpRequest` | Older HTTP request API |

When you call `setTimeout(myFn, 1000)`, the JS engine immediately hands the timer off to the browser/OS. The timer runs **outside** the JS engine, in C++ code. When the timer fires, the callback is placed in the **macrotask queue**. The JS engine never blocks waiting for it.

---

## Piece 4: The Two Queues

### Macrotask Queue (Task Queue)
Holds callbacks from:
- `setTimeout` / `setInterval`
- I/O callbacks (`fs.readFile`, HTTP responses)
- `setImmediate` (Node.js)
- UI rendering tasks (browser)

### Microtask Queue
Holds callbacks from:
- `Promise.then()` / `.catch()` / `.finally()`
- `async/await` (the continuation after each `await` is a microtask)
- `queueMicrotask(fn)`
- `MutationObserver` callbacks (browser)

**Critical rule: microtasks always run before macrotasks.**

---

## Piece 5: The Event Loop

The event loop is an infinite loop that does exactly this:

```
while (true) {
  if (callStack is empty) {
    // Step 1: drain the ENTIRE microtask queue
    while (microtaskQueue is not empty) {
      run next microtask      // this can ADD more microtasks — they all run too
    }
    // Step 2: run ONE macrotask (if any)
    if (macrotaskQueue is not empty) {
      run next macrotask      // this might add microtasks — they'll be drained next loop
    }
  }
}
```

Key point: **After every single macrotask, ALL microtasks are drained completely before the next macrotask runs.**

---

## Synchronous code — baseline

```js
console.log("A");
console.log("B");
console.log("C");

// Output: A  B  C
// Nothing async — everything runs top-to-bottom on the stack
```

---

## Introducing async: setTimeout

```js
console.log("1 — start");

setTimeout(() => {
  console.log("2 — timeout callback");
}, 0);

console.log("3 — end");

// Output:
// 1 — start
// 3 — end
// 2 — timeout callback
```

**Why?** Even with `0ms` delay, `setTimeout` hands the callback to the timer API. The timer immediately completes (0ms), and places the callback in the **macrotask queue**. But the main script is still on the call stack running `console.log("3")`. The event loop only picks up the macrotask when the call stack is **completely empty**.

---

## Microtasks vs Macrotasks — side by side

```js
console.log("1 — sync");

setTimeout(() => console.log("2 — macrotask (setTimeout)"), 0);

Promise.resolve().then(() => console.log("3 — microtask (Promise)"));

queueMicrotask(() => console.log("4 — microtask (queueMicrotask)"));

setTimeout(() => console.log("5 — macrotask (second setTimeout)"), 0);

Promise.resolve().then(() => console.log("6 — microtask (second Promise)"));

console.log("7 — sync");

// Output:
// 1 — sync
// 7 — sync
// 3 — microtask (Promise)
// 4 — microtask (queueMicrotask)
// 6 — microtask (second Promise)
// 2 — macrotask (setTimeout)
// 5 — macrotask (second setTimeout)
```

**Trace:**
1. Sync code runs: logs 1, schedules macrotask#1 (setTimeout), schedules microtask#1 (Promise), schedules microtask#2 (queueMicrotask), schedules macrotask#2 (setTimeout), schedules microtask#3 (Promise), logs 7
2. Call stack empty → event loop drains ALL microtasks: logs 3, 4, 6
3. No more microtasks → event loop picks ONE macrotask: logs 2
4. Drain microtasks: (none) → pick next macrotask: logs 5

---

## Microtasks spawning more microtasks

```js
console.log("start");

Promise.resolve()
  .then(() => {
    console.log("microtask 1");
    return Promise.resolve();   // adds another microtask to the queue
  })
  .then(() => {
    console.log("microtask 2");
    return Promise.resolve();   // adds yet another
  })
  .then(() => {
    console.log("microtask 3");
  });

setTimeout(() => console.log("macrotask"), 0);

console.log("end");

// Output:
// start
// end
// microtask 1
// microtask 2
// microtask 3
// macrotask

// The setTimeout fires LAST even though the Promises do 3 steps —
// because ALL microtasks (including newly added ones) drain before ANY macrotask runs
```

---

## The event loop in Node.js — slightly different

Node.js has the same concept but a more explicit phase-based event loop powered by **libuv**:

```
   ┌───────────────────────────┐
┌─►│  timers                   │  ← setTimeout, setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │  pending callbacks        │  ← I/O error callbacks from previous tick
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │  idle, prepare            │  ← internal Node use only
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │  poll                     │  ← retrieve new I/O events (fs, network)
│  │                           │    block here if nothing else to do
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │  check                    │  ← setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
└──│  close callbacks          │  ← socket.on("close"), etc.
   └───────────────────────────┘

   Between EVERY phase: drain the microtask queue (Promises, process.nextTick)
   process.nextTick runs BEFORE other microtasks (it has the highest priority)
```

### Node.js specific: `process.nextTick` vs `setImmediate` vs `setTimeout(fn,0)`

```js
console.log("sync");

setImmediate(() => console.log("setImmediate"));

setTimeout(() => console.log("setTimeout 0"), 0);

Promise.resolve().then(() => console.log("Promise microtask"));

process.nextTick(() => console.log("process.nextTick"));

console.log("sync end");

// Output (Node.js):
// sync
// sync end
// process.nextTick       ← highest priority microtask in Node
// Promise microtask      ← regular microtask
// setTimeout 0           ← macrotask (timers phase)
// setImmediate           ← check phase (after poll, after timers)

// Note: setTimeout(fn,0) vs setImmediate order can vary depending on
// where in the event loop phases the code was called from.
// When called from I/O callbacks, setImmediate ALWAYS fires before setTimeout.
```

---

## Stack overflow — the call stack has a limit

```js
function infinite() {
  return infinite();   // calls itself forever — no base case
}

infinite();
// RangeError: Maximum call stack size exceeded
```

Each recursive call adds a frame to the stack. The stack has a finite size (typically ~10,000–15,000 frames). Async recursion via `setTimeout` avoids this because each timeout callback runs in a fresh stack:

```js
// This runs "forever" without stack overflow — each iteration is a fresh stack frame
function loop(count) {
  if (count > 100_000) return;
  setTimeout(() => loop(count + 1), 0);
}
loop(0);
```

---

## Starvation — microtask queue can block the macrotask queue

Because the event loop drains ALL microtasks before running any macrotask, an infinite microtask loop will starve all macrotasks (including I/O, timers, rendering):

```js
// DANGER — infinite microtask loop starves everything
function scheduleForever() {
  Promise.resolve().then(scheduleForever);
}
scheduleForever();

setTimeout(() => console.log("I'll never run"), 100);
// The setTimeout callback NEVER fires — microtask queue never empties
```

In practice this happens with poorly written recursive Promise chains. It's rare but catastrophic — the browser tab freezes or Node.js becomes completely unresponsive.

---

## Blocking the event loop — the #1 Node.js performance mistake

Because JS is single-threaded, any synchronous code that takes a long time blocks EVERYTHING — no incoming requests can be handled, no timers fire, no responses go out:

```js
// In a Node.js server — this BLOCKS the entire server for ~5 seconds
app.get("/slow", (req, res) => {
  // WRONG — synchronous CPU work blocks all other requests
  const start = Date.now();
  while (Date.now() - start < 5000) {}  // busy-wait for 5 seconds
  res.send("done");
});

// While this handler runs, NO other request can be served.
// 1000 concurrent users all wait 5 seconds. Server appears hung.
```

The fix is to **never block the event loop** — use async I/O, worker threads for CPU work, or break up large loops with `setImmediate`:

```js
// RIGHT — CPU-intensive work broken into async chunks
function processLargeArray(array, callback) {
  let index = 0;

  function processChunk() {
    const end = Math.min(index + 1000, array.length);   // process 1000 items at a time
    while (index < end) {
      callback(array[index]);
      index++;
    }
    if (index < array.length) {
      setImmediate(processChunk);   // yield to event loop, then continue
    }
  }

  processChunk();
}
```

---

## Real execution traces — test yourself

### Trace 1

```js
console.log("A");

setTimeout(() => console.log("B"), 1000);
setTimeout(() => console.log("C"), 0);

Promise.resolve()
  .then(() => console.log("D"))
  .then(() => console.log("E"));

console.log("F");

// Answer: A  F  D  E  C  B
//
// Reasoning:
// Sync: A, schedule macro(B,1000ms), schedule macro(C,0ms), schedule micro(D), F
// Stack empty → drain micros: D → .then schedules E → drain E
// Macros: C fires (0ms elapsed) → B fires (1000ms elapsed)
```

### Trace 2

```js
async function fetchData() {
  console.log("1 — inside async fn, before await");
  const result = await Promise.resolve("data");
  console.log("3 — after await, result:", result);
}

console.log("0 — before call");
fetchData();
console.log("2 — after call, sync continues");

// Answer: 0  1  2  3
//
// Reasoning:
// "0" — sync
// fetchData() called — "1" runs synchronously UNTIL the first 'await'
// await suspends fetchData, schedules continuation as microtask
// control returns to caller — "2" runs
// stack empty → drain microtask: fetchData resumes → "3"
```

### Trace 3 (Node.js)

```js
setTimeout(() => console.log("timeout"), 0);

setImmediate(() => console.log("immediate"));

Promise.resolve().then(() => console.log("promise"));

process.nextTick(() => console.log("nextTick"));

// Output (from inside I/O callback or top-level):
// nextTick    ← process.nextTick queue (drained before all microtasks)
// promise     ← microtask queue
// timeout     ← timers phase macrotask
// immediate   ← check phase (setImmediate)
```

---

## Why async I/O doesn't block — the real mechanism

When Node.js calls `fs.readFile("file.txt", callback)`:

1. JS calls `fs.readFile` → immediately returns (non-blocking)
2. libuv (Node's async I/O library written in C++) sends the read request to the OS
3. The OS kernel performs the actual disk I/O asynchronously (using OS-level async APIs like epoll/kqueue/IOCP)
4. Meanwhile, JS continues running other code
5. When the OS signals the read is done, libuv places the callback in the I/O callbacks phase of the event loop
6. When the event loop reaches that phase, the callback runs with the file data

```
JS engine          libuv (C++)          OS Kernel
    │                   │                    │
    │──fs.readFile()───►│                    │
    │  (returns immed.) │──async read req───►│
    │                   │                    │  (disk I/O happens here)
    │ (JS keeps running)│                    │
    │                   │◄──"done, here's data"
    │◄──callback queued─│                    │
    │  (on next event   │                    │
    │   loop iteration) │                    │
```

This is why Node.js can handle thousands of concurrent connections with a single thread — it never waits for I/O. It just queues work and moves on.

---

## Practical mental model for backend developers

When writing backend code in Node.js, think of your code in two categories:

**Category 1 — Synchronous (runs immediately, blocks)**
- Any arithmetic, string manipulation, object creation
- `JSON.parse()` / `JSON.stringify()`
- `crypto.createHash()` (sync version)
- Array `.map()`, `.filter()`, `.reduce()`
- `fs.readFileSync()` — blocking file read (avoid in servers)

**Category 2 — Asynchronous (defers to runtime, non-blocking)**
- `fs.readFile()` — async file read
- `fetch()` / `http.request()` — HTTP calls
- `setTimeout` / `setInterval`
- Database queries (mongoose, pg, prisma all return Promises)
- `crypto.randomBytes()` (async version)

**Rule of thumb for Node.js servers:**
- Never use `*Sync` methods in request handlers — they block all other requests
- Keep CPU-bound work under ~1ms per event loop tick
- Move heavy CPU computation to worker threads or a separate process

---

## Common mistakes

### Mistake 1 — Reading async results synchronously

```js
// WRONG — trying to use async result before it's ready
let userData;
fetch("/api/user")
  .then(res => res.json())
  .then(data => { userData = data; });

console.log(userData);   // undefined — fetch hasn't completed yet
                          // the .then() callbacks run LATER as microtasks

// RIGHT — use the result inside the .then() or with async/await
fetch("/api/user")
  .then(res => res.json())
  .then(data => {
    console.log(data);    // correct — data is available here
    processUser(data);
  });
```

### Mistake 2 — Thinking setTimeout(fn, 0) runs immediately

```js
// WRONG assumption
setTimeout(() => console.log("runs first"), 0);
console.log("runs second?");

// WRONG — actual output:
// "runs second?"
// "runs first"

// setTimeout(fn, 0) means "run this when the call stack is next empty"
// NOT "run this right now"
```

### Mistake 3 — Blocking the event loop with synchronous loops in servers

```js
// WRONG in a Node.js server handler
app.get("/report", (req, res) => {
  const result = hugeArray.reduce((acc, item) => heavyComputation(acc, item), 0);
  // If this takes 100ms, your server is frozen for 100ms for ALL users
  res.json({ result });
});

// RIGHT — offload to a worker thread, or use streaming/pagination
const { Worker } = require("worker_threads");
app.get("/report", (req, res) => {
  const worker = new Worker("./heavy-computation.js", { workerData: hugeArray });
  worker.on("message", result => res.json({ result }));
});
```

### Mistake 4 — Ignoring that Promise chains are microtasks

```js
// This surprises people: the .then() doesn't run immediately after resolve
let resolved = false;
Promise.resolve().then(() => { resolved = true; });
console.log(resolved);   // false — the .then() callback hasn't run yet
                          // it's queued as a microtask, not run synchronously
```

---

## Practice exercises

### Exercise 1 — easy

Predict the output of the following code **before running it**. Then run it and verify:

```js
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
setTimeout(() => console.log("4"), 100);
Promise.resolve().then(() => console.log("5"));
console.log("6");

// Write your predicted output order here as a comment, then run to verify
```

### Exercise 2 — medium

Write a function `delay(ms)` that returns a Promise which resolves after `ms` milliseconds. Then write an async function `runSequence()` that:
1. Logs `"Step 1 — starting"`
2. Waits 1 second using `delay(1000)`
3. Logs `"Step 2 — after 1s"`
4. Waits 500ms
5. Logs `"Step 3 — after 0.5s more"`
6. Returns `"done"`

Also log `"After calling runSequence"` immediately after calling it to prove async suspension works.

```js
// Write your code here
```

### Exercise 3 — hard

Build a function `asyncQueue(tasks, concurrency)` where:
- `tasks` is an array of functions that each return a Promise
- `concurrency` is the max number of tasks to run at the same time

The function returns a Promise that resolves with an array of all results in the original order.

Requirements:
- Never run more than `concurrency` tasks simultaneously
- As soon as one task finishes, start the next pending one (always keep the queue full up to the limit)
- Preserve result order regardless of which task finishes first
- Handle task rejections — if any task rejects, the overall Promise rejects

Test with 6 tasks that each log their start/end and take different amounts of time (use your `delay` function), with a concurrency of 2. Verify that only 2 run at a time.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Detail |
|---|---|
| Call stack | LIFO — tracks current execution; one at a time |
| Heap | Memory storage for objects |
| Web API / libuv | Runtime handles async work outside the JS engine |
| Macrotask queue | `setTimeout`, `setInterval`, I/O callbacks, `setImmediate` |
| Microtask queue | `Promise.then/catch/finally`, `queueMicrotask`, `process.nextTick` |
| Event loop rule | After each macrotask: drain ALL microtasks first |
| Microtask priority | Always before macrotasks; newly added microtasks also drain before macrotask |
| `process.nextTick` | Node.js only — runs before even regular microtasks |
| `setTimeout(fn, 0)` | Does NOT run "right now" — runs when call stack is empty AND timers phase |
| `setImmediate` | Node.js — runs in check phase, after I/O, before close events |
| Blocking the loop | Any long sync operation freezes JS entirely — avoid in servers |
| Stack overflow | Too many synchronous recursive calls — use async recursion to avoid |
| Microtask starvation | Infinite `.then()` chains block macrotasks forever |
| `async/await` | Each `await` suspension is a microtask — code after `await` runs as microtask |
| Non-blocking I/O | OS handles disk/network; callback queued when done; JS never waits |

---

## Connected topics

- **53 — Callbacks and callback hell** — The original async pattern before Promises; directly uses the macrotask queue.
- **54 — Promises** — Built on microtasks; `.then()` callbacks are always microtasks even if the Promise is already resolved.
- **56 — async/await** — Syntactic sugar over Promises; every `await` schedules the continuation as a microtask.
- **58 — fetch API** — A real-world async operation; understanding this topic tells you exactly when each `.then()` callback fires relative to other code.
