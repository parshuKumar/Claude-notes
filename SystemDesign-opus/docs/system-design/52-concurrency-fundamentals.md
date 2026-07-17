# 52 — Concurrency Fundamentals — Thread Pool, Mutex, Locks, Event Loop
## Category: LLD Patterns

---

## What is this?

Concurrency is the art of making progress on **many tasks at once** without necessarily doing them at the same instant. Think of one barista taking your order, starting your espresso, taking the next person's order while the machine runs, then steaming milk for a third — one person, many orders in flight, nobody waiting idle.

**Parallelism** is different: it's hiring four baristas so four drinks are genuinely being made at the same physical moment.

This doc is about what happens when tasks overlap: how they corrupt each other's data (race conditions), how you protect the dangerous bits (mutexes, semaphores, critical sections), how they can freeze forever (deadlock), and how Node.js — the "single-threaded" runtime — actually runs your code (the event loop, libuv's thread pool, worker threads, cluster).

---

## Why does it matter?

If you don't understand this, you will write a bug that only shows up in production, only under load, and only sometimes. Those are the worst bugs there are — they don't reproduce on your laptop, they can't be found by a unit test, and they cost money.

**The real-work angle.** The single most common shape of this bug in a Node backend is the *lost update*: two HTTP requests read the same row, both compute a new value from the stale copy, both write it back. One write silently wins. Balances go wrong, inventory oversells, coupon codes get redeemed twice, idempotency keys get double-processed. Every payments engineer has a scar from this.

**The interview angle.** "Node is single-threaded, so I don't need locks" is a very common — and wrong — answer, and interviewers love it because the follow-up separates people who have read a blog post from people who have shipped a server. Expect: *"Explain the event loop."* *"What is `UV_THREADPOOL_SIZE`?"* *"Why does one CPU-heavy request slow down every other user?"* *"How would you limit outbound API calls to 5 at a time?"* *"When would you use `worker_threads` instead of `cluster`?"*

You also need this vocabulary to *read* other people's designs. Database isolation levels, distributed locks, optimistic concurrency, message-queue consumers — all of it is the same set of ideas wearing different hats.

---

## The core idea — explained simply

### The One-Barista Coffee Shop Analogy

Picture a coffee shop with **one barista** (your Node process) and a **queue of customers** (incoming requests).

A naive barista serves people strictly one at a time: take order, grind, pull the shot, steam milk, hand over, next. If a customer orders a slow pour-over that takes 3 minutes, everyone else stands there watching. That's **blocking, sequential** code.

A good barista is **concurrent**. She takes your order, starts the espresso machine, and while the machine is running she turns around and takes the next order. She is never doing two things in the same instant — she has two hands and one brain — but she's *juggling* many orders, working on whichever one is ready to be worked on. The espresso machine running is your `await db.query(...)`: the work is happening *somewhere else* (kernel, disk, network), and the barista is free.

**Parallelism** is a second, third, fourth barista behind the counter. Now two shots really are being pulled at the same microsecond. In Node, that's `cluster` (four whole coffee shops) or `worker_threads` (four baristas sharing one counter).

Now the dangerous part. There's a whiteboard behind the counter with the **stock count of oat milk: 1 carton left**. Barista starts order A, reads the board ("1 left"), and turns away to steam milk. Meanwhile she starts order B, reads the board ("1 left" — still, nobody updated it), promises oat milk to B as well. Two customers promised the last carton. That's a **race condition**, and note carefully: it happened with *one barista*. You did not need two.

That's the whole point of this doc. **The bug isn't caused by parallelism. It's caused by interleaving.**

| In the coffee shop | In your Node server |
|---|---|
| The barista | The single JS thread / event loop |
| Customers in line | Pending requests / queued callbacks |
| Espresso machine running | `await` on IO (DB, HTTP, disk) |
| Barista turns away mid-order | The `await` yields — another task runs |
| The stock whiteboard | Shared mutable state (DB row, in-memory Map) |
| Promising the last carton twice | Race condition / lost update |
| "Don't touch the board while I'm mid-count" | Critical section protected by a mutex |
| Hiring more baristas | `cluster` / `worker_threads` (real parallelism) |
| The back-room staff (4 people) doing dishes | libuv's thread pool (`UV_THREADPOOL_SIZE`, default 4) |
| A single 3-minute pour-over blocking the counter | CPU-heavy sync loop blocking the event loop |

---

## Key concepts inside this topic

### 1. Concurrency vs parallelism — say it precisely

- **Concurrency** = *structuring* a program so multiple tasks are **in progress** over the same period. They interleave. One CPU is enough.
- **Parallelism** = *executing* multiple tasks at the **same instant**. Needs multiple cores.

Concurrency is about **dealing with** many things at once. Parallelism is about **doing** many things at once. You can have concurrency without parallelism (classic Node), parallelism without much concurrency (a batch matrix multiply across cores), or both (a clustered Node server).

```
CONCURRENT, NOT PARALLEL (one core, interleaved)
Core 1: [A..][B..][A..][C..][B..][A done][C..]
        Tasks take turns. Total wall time is not reduced by CPU work,
        but IO waits overlap, so throughput goes way up.

PARALLEL (two cores, truly simultaneous)
Core 1: [A....................A done]
Core 2: [B....................B done]
        Actual CPU work happens at the same time.
```

Practical consequence: concurrency wins for **IO-bound** work (waiting on DB/network), parallelism wins for **CPU-bound** work (hashing, image resize, JSON of 50MB).

### 2. Race conditions in JavaScript — yes, really

Here is the belief to kill: *"JS is single-threaded, so my code is atomic and I'm safe."*

Half true. You are safe from **thread-level races** — no two threads write the same variable at the same nanosecond, no torn reads, no half-written 64-bit numbers. Memory is never corrupted.

You are **not** safe from **interleaving races**. And here's the rule that makes it click:

> **Every `await` is a yield point. Between the `await` and the line after it, the event loop is free to run any other pending task — including another copy of the same function.**

Your function is atomic only *between* awaits. It is not atomic *across* them.

Now the classic bug. Two withdrawal requests hit your Express server at the same moment.

```js
// bank-broken.js — DO NOT SHIP THIS
// A fake DB with realistic latency so the race is visible.
const db = {
  _balance: 100,
  async getBalance() {
    await sleep(20);           // network round-trip to Postgres
    return this._balance;
  },
  async setBalance(v) {
    await sleep(20);
    this._balance = v;
  },
};
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function withdraw(amount, who) {
  const balance = await db.getBalance();      // <-- YIELD POINT
  if (balance < amount) {
    console.log(`${who}: declined, insufficient funds`);
    return false;
  }
  const next = balance - amount;              // pure CPU, atomic
  await db.setBalance(next);                  // <-- YIELD POINT
  console.log(`${who}: withdrew ${amount}, balance now ${next}`);
  return true;
}

// Two HTTP handlers running "at the same time"
await Promise.all([withdraw(50, 'requestA'), withdraw(50, 'requestB')]);
console.log('FINAL BALANCE:', db._balance);
```

Output:

```
requestA: withdrew 50, balance now 50
requestB: withdrew 50, balance now 50
FINAL BALANCE: 50
```

**We handed out £100 and the balance only dropped by £50.** The bank just lost money.

Walk the interleaving. This is the part to be able to draw on a whiteboard:

```
time →
A: getBalance() ──await──▶ returns 100
B:      getBalance() ──await──▶ returns 100      ← B read BEFORE A wrote
A: computes 100-50 = 50, setBalance(50) ──await──▶ done
B: computes 100-50 = 50, setBalance(50) ──await──▶ done   ← overwrites with the same 50
```

Nothing was corrupted. Every individual operation was correct. The *sequence* was wrong. This is called a **lost update** or **read-modify-write race**, and `Promise.all`, a busy Express server, a Kafka consumer with `concurrency: 10`, and a `for` loop that forgets to `await` all produce it.

The same bug in an in-memory cache, no DB at all:

```js
const seen = new Map();
async function processOnce(id, job) {
  if (seen.has(id)) return;        // check
  await job();                     // <-- YIELD: another call gets here with seen still empty
  seen.set(id, true);              // act — too late
}
```
Check-then-act across an `await` is *always* suspect. Train your eyes to spot it.

### 3. Critical sections

A **critical section** is a stretch of code that touches shared state and must not be interleaved with another execution of itself. In `withdraw`, the critical section is:

```
read balance ─── decide ─── write balance
```

All three steps must happen as one indivisible unit. Three ways to enforce that:

1. **Make the DB do it.** `UPDATE accounts SET balance = balance - 50 WHERE id = $1 AND balance >= 50` — a single atomic statement, no read-modify-write in JS at all. Or `SELECT ... FOR UPDATE` in a transaction, or an optimistic-concurrency `WHERE version = $v`. **In a real multi-process backend this is the correct answer**, because an in-process lock protects one Node process and you run eight of them.
2. **Use an in-process lock (mutex)** when the shared state genuinely lives in this process: an in-memory cache, a rate-limiter counter, a file handle, a serialized write to a local file.
3. **Use a distributed lock** (Redis `SET key val NX PX 30000`, or a lease in etcd/Zookeeper) when the state spans machines. Fragile; prefer option 1 whenever you can.

We'll build option 2 now, because building it teaches the mechanics.

### 4. Implementing an async Mutex in JavaScript

A **mutex** (mutual exclusion) is a lock exactly one holder can hold at a time. Everyone else waits in a queue. In JS there's no thread to block, so "waiting" means *your promise doesn't resolve yet*.

The trick: keep an internal queue of **pending `resolve` functions**. `acquire()` either grants the lock immediately or parks a resolver in the queue. `release()` pops the next resolver and calls it — which wakes up exactly one waiter.

```js
// mutex.js
export class Mutex {
  #locked = false;
  #waiters = [];          // queued resolve() fns — FIFO, so it's fair (no starvation)

  /** Resolves with a release() function. Always release in a finally block. */
  acquire() {
    if (!this.#locked) {
      this.#locked = true;
      return Promise.resolve(this.#makeRelease());
    }
    return new Promise((resolve) => {
      this.#waiters.push(() => resolve(this.#makeRelease()));
    });
  }

  #makeRelease() {
    let released = false;                 // guard: double-release would corrupt the queue
    return () => {
      if (released) return;
      released = true;
      const next = this.#waiters.shift();
      if (next) next();                   // hand the lock straight to the next waiter
      else this.#locked = false;          // nobody waiting — lock is free
    };
  }

  /** Sugar: runs fn while holding the lock, releases even if fn throws. */
  async runExclusive(fn) {
    const release = await this.acquire();
    try {
      return await fn();
    } finally {
      release();                          // MUST be in finally, or a throw deadlocks everyone
    }
  }
}
```

Now fix the bank:

```js
// bank-fixed.js
import { Mutex } from './mutex.js';

const locks = new Map();                     // one mutex PER ACCOUNT, not one global lock
const lockFor = (id) => {
  if (!locks.has(id)) locks.set(id, new Mutex());
  return locks.get(id);
};

async function withdraw(accountId, amount, who) {
  return lockFor(accountId).runExclusive(async () => {
    const balance = await db.getBalance();   // critical section starts
    if (balance < amount) {
      console.log(`${who}: declined, insufficient funds`);
      return false;
    }
    await db.setBalance(balance - amount);
    console.log(`${who}: withdrew ${amount}, balance now ${balance - amount}`);
    return true;                             // critical section ends; release() in finally
  });
}

await Promise.all([
  withdraw('acct-1', 50, 'requestA'),
  withdraw('acct-1', 50, 'requestB'),
]);
console.log('FINAL BALANCE:', db._balance);
```

Fixed output:

```
requestA: withdrew 50, balance now 50
requestB: declined, insufficient funds
FINAL BALANCE: 50
```

Now the balance and the outcomes agree: one withdrawal succeeded, one was correctly declined. Note the *per-account* mutex — a single global lock would serialize every user in your system behind every other user, turning a concurrent server into a sequential one. **Lock the narrowest thing that is correct.**

### 5. Semaphore — a concurrency limiter you'll actually use

A **semaphore** is a mutex that allows **N** holders instead of 1. A mutex is just a semaphore with N = 1.

The killer use case: you have 5,000 users to enrich via a third-party API that rate-limits you at 10 concurrent requests. `Promise.all(users.map(fetchUser))` fires 5,000 sockets and gets you banned. A `for...of` loop with `await` does them one at a time and takes an hour. You want *exactly 10 in flight, always*.

```js
// semaphore.js
export class Semaphore {
  #permits;
  #waiters = [];

  constructor(permits) {
    if (permits < 1) throw new RangeError('permits must be >= 1');
    this.#permits = permits;
  }

  async acquire() {
    if (this.#permits > 0) {
      this.#permits -= 1;
      return this.#makeRelease();
    }
    return new Promise((resolve) => {
      this.#waiters.push(() => resolve(this.#makeRelease()));
    });
  }

  #makeRelease() {
    let released = false;
    return () => {
      if (released) return;
      released = true;
      const next = this.#waiters.shift();
      if (next) next();          // pass the permit directly to a waiter
      else this.#permits += 1;   // no waiters — return the permit to the pool
    };
  }

  async run(fn) {
    const release = await this.acquire();
    try { return await fn(); } finally { release(); }
  }
}

/** Map over items with at most `limit` operations in flight. Order of results preserved. */
export async function mapLimit(items, limit, fn) {
  const sem = new Semaphore(limit);
  return Promise.all(items.map((item, i) => sem.run(() => fn(item, i))));
}
```

Usage — 5,000 users, never more than 10 sockets open:

```js
const enriched = await mapLimit(users, 10, async (user) => {
  const res = await fetch(`https://api.example.com/profile/${user.id}`);
  return res.json();
});
```

This is genuinely one of the most useful 30 lines of code in a Node backend. (The `p-limit` package is the same idea; now you know what's inside it.)

### 6. Deadlock, livelock, starvation

**Deadlock** = two or more tasks each waiting for a lock the other holds. Nobody ever proceeds. It requires all **four Coffman conditions** simultaneously:

| Condition | Meaning | Break it by |
|---|---|---|
| **Mutual exclusion** | The resource can't be shared | Use immutable data / copy-on-write / atomic DB ops |
| **Hold and wait** | You hold lock 1 while requesting lock 2 | Acquire all locks up front, or hold none while waiting |
| **No preemption** | A lock can't be forcibly taken back | Add **lock timeouts** — give up and retry |
| **Circular wait** | A → waits for B → waits for A | **Global lock ordering** (the practical fix) |

Break **any one** and deadlock is impossible.

A worked deadlock — a money transfer that locks both accounts:

```js
// DEADLOCK — transfer A→B and B→A at the same time
async function transferBroken(from, to, amount) {
  const r1 = await lockFor(from).acquire();   // T1 grabs 'alice'; T2 grabs 'bob'
  await sleep(10);                            // any await widens the window
  const r2 = await lockFor(to).acquire();     // T1 waits for 'bob' (held by T2)
  try { /* move money */ } finally { r2(); r1(); }  // T2 waits for 'alice' (held by T1)
}

await Promise.all([
  transferBroken('alice', 'bob', 50),
  transferBroken('bob', 'alice', 30),
]);   // hangs forever — event loop is alive, but both tasks are parked
```

Notice this is a **hang, not a crash**. Your process stays up, health checks pass, and requests pile up until the box falls over. Deadlocks are quiet.

The fix — **consistent global lock ordering**. Always take locks in the same order (sorted by ID), so a cycle is impossible:

```js
async function transfer(from, to, amount) {
  const [first, second] = [from, to].sort();   // ALWAYS alphabetical: alice before bob
  const r1 = await lockFor(first).acquire();
  try {
    const r2 = await lockFor(second).acquire();
    try {
      const balance = await db.getBalance(from);
      if (balance < amount) throw new Error('insufficient funds');
      await db.setBalance(from, balance - amount);
      await db.setBalance(to, (await db.getBalance(to)) + amount);
    } finally { r2(); }
  } finally { r1(); }
}
```

Both tasks now want `alice` first. One gets it, the other waits, the first finishes, the second proceeds. No cycle, no deadlock.

Three more defences worth naming in an interview:
- **Lock timeouts.** `acquireWithTimeout(5000)` — reject rather than hang, then retry with jitter. Turns a permanent hang into a transient error you can alert on.
- **Never hold a lock across an `await` you don't control.** Awaiting a third-party HTTP call while holding a lock means a slow vendor can stall your whole queue. Do the IO first, hold the lock only for the short critical section.
- **Don't hold two locks at all** if a single atomic DB transaction can do the job.

**Livelock**: tasks are *actively running* but making no progress — two people stepping side-to-side in a corridor. In code: two retry loops that both back off and retry in perfect lockstep, forever. Fix with **randomized (jittered) exponential backoff**.

**Starvation**: a task never gets the resource because others keep jumping the queue — e.g. a priority queue where high-priority work never stops arriving, so a low-priority job waits forever. Fix with **FIFO fairness** (our mutex's `waiters` array is FIFO, so it's starvation-free) or **ageing** (raise a job's priority the longer it waits).

### 7. Thread pools — including Node's hidden one

**Why thread pools exist:** creating an OS thread costs ~0.5–1 MB of stack plus a syscall — roughly 50–100 µs. Do that per request at 10,000 req/s and you've spent all your time creating threads. Worse, 10,000 live threads means the OS scheduler spends its life **context switching** (each switch ~1–5 µs, plus cache pollution) instead of doing work. That's **thrashing**.

A thread pool creates a fixed set of workers once and feeds them tasks from a queue. Bounded memory, bounded scheduler pressure, no per-task creation cost.

**Sizing a pool:**

| Workload | Size | Why |
|---|---|---|
| **CPU-bound** (hashing, compression, image resize) | ≈ number of cores (`os.cpus().length`) | Extra threads can't create extra CPU; they only add context switches |
| **IO-bound** (DB, HTTP, disk) | Much higher — threads are asleep most of the time | Classic formula: `cores × (1 + waitTime/computeTime)`. If a task waits 90 ms and computes 10 ms, that's `cores × 10` |
| **Mixed** | Two pools, one per class | Never let a slow IO task occupy a CPU worker |

**Node's own thread pool — the surprising production bottleneck.** Node is not really single-threaded. libuv keeps a thread pool, **4 threads by default**, and it is used by:
- `fs.*` (all filesystem calls except `fs.FSWatcher` and a couple of sync paths)
- `crypto.pbkdf2`, `crypto.randomBytes`, `crypto.scrypt`, and **`bcrypt`** (native addon)
- `zlib` (gzip/brotli)
- `dns.lookup()` (but *not* `dns.resolve*`, which uses the network directly)

Network sockets do **not** use it — those are handled by the OS event notification system (epoll/kqueue), which is why Node handles 10k idle sockets happily.

Consequence: **only 4 `bcrypt` hashes can run at once.** bcrypt at cost factor 12 takes ~250 ms. Ten concurrent logins:

```
Requests 1–4:  run immediately          → ~250 ms
Requests 5–8:  queue behind them        → ~500 ms
Requests 9–10: queue again              → ~750 ms
```

The 10th user waits three quarters of a second for a login, and your CPU is barely warm. The graph looks like a mystery: latency climbing, CPU low. This is the thread pool queue.

```js
// Raise it — must be set BEFORE any lib touches the pool. Ideally in the shell:
//   UV_THREADPOOL_SIZE=8 node server.js
process.env.UV_THREADPOOL_SIZE = '8';   // works only if set before the first pool use
```

Rule: set `UV_THREADPOOL_SIZE` to roughly your core count if you do heavy `crypto`/`zlib`/`fs`. Max is 1024, but going far past your cores just moves the queue — bcrypt is CPU-bound, so more threads than cores won't make it faster. If password hashing is a real load, move it to `worker_threads` or a dedicated service.

### 8. The Node event loop, properly

The **call stack** runs your synchronous JS. When it's empty, the **event loop** picks up the next callback. libuv runs the loop in phases, and each turn ("tick") goes through them in order:

| Phase | What runs here |
|---|---|
| **timers** | `setTimeout` / `setInterval` callbacks whose time has come |
| **pending callbacks** | Deferred system callbacks (e.g. some TCP errors) |
| **idle, prepare** | Internal to libuv |
| **poll** | **The big one.** Retrieve new IO events; run their callbacks (socket data, file reads). Blocks here if there's nothing else to do |
| **check** | `setImmediate` callbacks |
| **close callbacks** | `socket.on('close')`, etc. |

**Macrotasks vs microtasks.** The phases above hold **macrotasks**. Separately there are two **microtask** queues:
1. `process.nextTick()` queue — highest priority
2. Promise queue — `.then` / `await` continuations

After **every single macrotask** (and between each phase), Node drains the *entire* nextTick queue, then the *entire* promise queue, before moving on. That's why microtasks "jump ahead" of timers:

```js
setTimeout(() => console.log('1: timeout (macrotask, timers phase)'), 0);
setImmediate(() => console.log('2: immediate (macrotask, check phase)'));
Promise.resolve().then(() => console.log('3: promise (microtask)'));
process.nextTick(() => console.log('4: nextTick (microtask, top priority)'));
console.log('5: synchronous');

// Output:
// 5: synchronous        ← call stack first, always
// 4: nextTick           ← nextTick queue drains first
// 3: promise            ← then the promise queue
// 1: timeout            ← then macrotasks
// 2: immediate
```

(`setTimeout` vs `setImmediate` order at top level is actually non-deterministic — it depends on how long the process took to boot. Inside an IO callback, `setImmediate` always wins, because you're already past the timers phase.)

**Starving the loop with microtasks** is possible — a `process.nextTick` that schedules another `process.nextTick` forever means the loop never reaches the poll phase and your server stops answering. Microtask priority is a foot-gun, not just a feature.

**Why a CPU-heavy sync loop blocks everything:**

```js
app.get('/report', (req, res) => {
  let total = 0;
  for (let i = 0; i < 5e9; i++) total += i;   // ~5 seconds of pure CPU
  res.json({ total });                        // meanwhile: NOTHING else runs
});
```

For those 5 seconds the call stack never empties. No timers fire, no sockets are read, no health check is answered, no other user is served. **One request took the whole server hostage.** This is the single biggest difference between Node and a thread-per-request runtime, and the reason `JSON.parse` of a 200 MB body, a synchronous `bcrypt.hashSync`, or a regex with catastrophic backtracking is an availability incident, not a slow endpoint.

Fixes: offload to `worker_threads`, chunk the work with `setImmediate` between batches, or move it to a separate service/queue.

### 9. Real parallelism in Node: worker_threads vs cluster

**`worker_threads`** — multiple JS threads *inside one process*. Each has its own V8 isolate and event loop, but they share the process's memory space and can share raw memory via `SharedArrayBuffer`. Communication via `postMessage` (structured clone) or shared buffers.

```js
// worker.js
import { parentPort, workerData } from 'node:worker_threads';
const hash = expensiveHash(workerData.password);   // CPU-bound, off the main loop
parentPort.postMessage(hash);
```

```js
// main.js
import { Worker } from 'node:worker_threads';
const runInWorker = (password) =>
  new Promise((resolve, reject) => {
    const w = new Worker(new URL('./worker.js', import.meta.url), { workerData: { password } });
    w.on('message', resolve);
    w.on('error', reject);
    w.on('exit', (code) => code !== 0 && reject(new Error(`worker exit ${code}`)));
  });
// In production: keep a POOL of workers (spawning one costs ~10-30ms) — see piscina.
```

**`cluster`** — multiple *processes*, each a full Node runtime, sharing one listening socket so the OS load-balances connections across them.

```js
// server.js
import cluster from 'node:cluster';
import os from 'node:os';
import http from 'node:http';

if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();
  cluster.on('exit', (worker) => {
    console.log(`worker ${worker.process.pid} died — restarting`);
    cluster.fork();                          // free self-healing
  });
} else {
  http.createServer((req, res) => res.end(`served by pid ${process.pid}\n`)).listen(3000);
}
```

**THE CLASSIC BUG.** With `cluster`, each worker is a **separate process with its own heap**. So:

```js
// rate-limit.js — BROKEN under cluster
const hits = new Map();                      // ← one Map PER PROCESS
export function allow(ip) {
  const n = (hits.get(ip) ?? 0) + 1;
  hits.set(ip, n);
  return n <= 100;                           // "100 req/min"
}
```
With 8 workers, a user gets **800** requests/min, not 100 — the counter is duplicated eight times. The same trap applies to an in-memory cache (8 independent caches, 8× the memory, inconsistent hits), a **Singleton** (you have eight of them), an in-memory session store (users get logged out when they land on a different worker), and `setInterval`-based cron (your "nightly email" job runs eight times).

Fix: move shared state **out of the process** — Redis for counters/sessions/caches, a proper scheduler (or a leader-elected single worker) for cron. Anything that must be global across workers cannot live in a JS variable.

| | `worker_threads` | `cluster` / child processes |
|---|---|---|
| Unit | Thread | Process |
| Memory | Shared heap possible (`SharedArrayBuffer`) | **Fully isolated** |
| Startup cost | ~10–30 ms | ~50–100 ms+ |
| A crash kills | Possibly the whole process | Only that worker |
| Message passing | `postMessage`, fast, structured clone | IPC over a socket, slower |
| Best for | **CPU-bound work in-process** — hashing, image/video, parsing, ML inference | **Scaling an HTTP server across cores**; fault isolation |
| Watch out for | Shared-memory races (real ones! `Atomics` needed) | Duplicated in-memory state |

**Rule:** `cluster` (or just N containers behind a load balancer) to use all your cores for request throughput. `worker_threads` (behind a pool like `piscina`) to keep one CPU-heavy task from freezing the event loop.

---

## Visual / Diagram description

**Diagram 1 — the Node event loop.** Draw this from memory in an interview and you're immediately in the top decile.

```
        ┌──────────────────────────────┐
        │      YOUR JS CALL STACK      │  ← synchronous code runs here.
        │   (one thread, runs to       │     While it's busy, the loop is FROZEN.
        │    completion, never         │
        │    interrupted)              │
        └──────────────┬───────────────┘
                       │ stack empties
                       ▼
   ┌───────────── DRAIN MICROTASKS ─────────────┐
   │  1. process.nextTick queue  (drain ALL)    │ ← runs after EVERY macrotask
   │  2. Promise / await queue   (drain ALL)    │    and between EVERY phase
   └──────────────────────┬─────────────────────┘
                          ▼
   ┌──────────────── THE EVENT LOOP (libuv) ─────────────────┐
   │                                                          │
   │   ┌──────────┐   setTimeout / setInterval callbacks      │
   │   │  timers  │                                           │
   │   └────┬─────┘                                           │
   │        ▼                                                 │
   │   ┌──────────────────┐  deferred system callbacks        │
   │   │ pending callbacks│                                   │
   │   └────┬─────────────┘                                   │
   │        ▼                                                 │
   │   ┌──────────┐   ★ THE BIG ONE ★                         │
   │   │   poll   │   new IO events: socket data, fs results  │
   │   └────┬─────┘   blocks here when there's nothing to do  │
   │        ▼                                                 │
   │   ┌──────────┐   setImmediate callbacks                  │
   │   │  check   │                                           │
   │   └────┬─────┘                                           │
   │        ▼                                                 │
   │   ┌──────────────┐  socket.on('close'), etc.             │
   │   │ close cbs    │                                       │
   │   └────┬─────────┘                                       │
   │        └──────────── loop back to timers ────────────┐   │
   └───────────────────────▲──────────────────────────────┼───┘
                           │ completed IO pushed back     │
   ┌───────────────────────┴──────────────────────────────┴───┐
   │  KERNEL (epoll/kqueue)   │   libuv THREAD POOL (4!)      │
   │  sockets, TCP, DNS       │   fs.*, crypto, zlib, bcrypt  │
   │  — no threads needed     │   UV_THREADPOOL_SIZE          │
   └──────────────────────────┴───────────────────────────────┘
```

Read it like this: your JS runs on the stack. When the stack empties, Node drains nextTick then promises, then walks the phases in order. IO you kicked off (a DB query, a file read) is executing *outside* JS entirely — in the kernel for sockets, in the 4-thread pool for `fs`/`crypto` — and when it finishes, its callback is queued and picked up in the **poll** phase, which resumes your `await`. The one thing that can break everything is the top box staying busy: a long synchronous loop means the stack never empties and nothing below ever runs.

**Diagram 2 — the race condition, and the mutex that fixes it.**

```
WITHOUT A LOCK — lost update
                                         balance in DB
  A: ── read(100) ──await──┐                  100
  B: ──── read(100) ──await┼─┐                100   ← B read stale value
  A: ──────── write(50) ───┘ │                 50
  B: ──────────── write(50) ─┘                 50   ← A's write vanished
                                 £100 paid out, balance dropped £50

WITH A MUTEX — serialized critical section
  A: [acquire]── read(100) ── write(50) ──[release]
  B:  (parked in waiters[] — its promise has not resolved) ─┐
  B:                                          [acquire]─────┘── read(50) ── DECLINED ──[release]
                                 £50 paid out, balance dropped £50  ✓
```

**Diagram 3 — deadlock, the circular wait.**

```
      holds 'alice'                    holds 'bob'
    ┌───────────────┐  wants 'bob'   ┌──────────────┐
    │  Transfer T1  │ ─────────────▶ │ Transfer T2  │
    │  (alice→bob)  │ ◀───────────── │ (bob→alice)  │
    └───────────────┘  wants 'alice' └──────────────┘
              ▲                              │
              └──────── CYCLE ───────────────┘
                  Neither can ever proceed.

  FIX: sort lock names. Both must take 'alice' first, then 'bob'.
       No cycle can form → deadlock is structurally impossible.
```

---

## Real world examples

### Stripe / payment systems — idempotency and the lost update

Payment APIs are the purest example of why interleaving races matter. Stripe's public API requires an **idempotency key** on write requests precisely because a client retry can race with the original request. Behind an idempotency layer sits a check-then-act problem: "has this key been used? if not, charge." If two copies of that check interleave, you charge the card twice. The industry-standard fix is not an application mutex but the database: insert the idempotency key with a **unique constraint** in the same transaction as the charge, so the second attempt fails at the storage layer where atomicity is guaranteed. Balance updates similarly use `UPDATE ... SET balance = balance - $1 WHERE balance >= $1` (atomic) or row-level locks, never a JS read-modify-write.

### Nginx and Node — the event-loop architecture

Nginx famously beat Apache on high-concurrency workloads by swapping "one thread (or process) per connection" for a small number of **event-driven worker processes**, each running an epoll loop. Node.js took the same architecture into application code: one event loop per process, plus a small libuv thread pool for the filesystem and crypto calls the OS can't do asynchronously. It's why a Node process can hold tens of thousands of mostly-idle WebSocket connections on modest memory — an idle socket costs a file descriptor and a bit of buffer, not a 1 MB thread stack. And it's why one blocking `JSON.parse` of a huge payload takes the whole box down: the architecture's strength and its weakness are the same property.

### Cloudflare Workers and the isolate model

Cloudflare Workers runs many customers' code on the same machine using V8 **isolates** rather than one process or container per tenant. Each isolate is single-threaded and memory-isolated, and the runtime multiplexes thousands of them across a handful of OS threads — concurrency without a thread per request, with far lower startup cost than a container. It's a large-scale bet on exactly the model in this doc: cheap concurrency plus process/isolate-level parallelism, rather than OS threads and locks.

---

## Trade-offs

**Event loop (concurrency) vs thread-per-request**

| | Event loop (Node) | Thread per request (classic Java/Rails) |
|---|---|---|
| Memory per connection | ~few KB | ~1 MB stack per thread |
| Idle connections | Cheap — 10k+ easily | Expensive |
| One slow CPU task | **Blocks everyone** | Blocks only that thread |
| Mental model | Must think about yield points | Feels sequential, but real memory races |
| Shared-state bugs | Interleaving races (lost update) | Thread races AND interleaving races |

**Locking strategies**

| Approach | Pros | Cons |
|---|---|---|
| **In-process mutex** | Simple, zero infra, fast | Only protects one process — useless under `cluster`/k8s replicas |
| **DB atomic statement** (`SET x = x - 1 WHERE ...`) | Correct across all processes, no extra infra | Requires you to express the logic in SQL |
| **Pessimistic DB lock** (`SELECT ... FOR UPDATE`) | Strong guarantees, arbitrary logic | Holds a row lock; contention → slow; risk of DB deadlocks |
| **Optimistic concurrency** (`WHERE version = $v`, retry) | Great when conflicts are rare; no locks held | Wasted work + retry storms when conflicts are common |
| **Distributed lock** (Redis/etcd) | Works across machines | Can be *unsafe* (clock skew, GC pauses, lost leases); needs fencing tokens |

**Parallelism strategies**

| Approach | Pros | Cons |
|---|---|---|
| **`worker_threads`** | Uses cores for CPU work; can share memory | Adds complexity; a bad worker can hurt the process; real races via `SharedArrayBuffer` |
| **`cluster` / N containers** | Uses all cores; fault isolation; trivially horizontal | Each process has **its own memory** — in-memory cache/singleton/rate-limiter break |
| **Offload to a queue + worker service** | Best isolation and scaling; retries for free | Extra infra; eventual consistency; more moving parts |

**Rule of thumb:** Use the **narrowest lock that is correct** and hold it for the **shortest time possible**. For anything that must be correct across processes, push the atomicity **into the database** — an in-process mutex is for in-process state only. Keep CPU work off the event loop, and never hold a lock across an `await` you don't control.

---

## Common interview questions on this topic

### Q1: "Node is single-threaded — can you even have a race condition?"

**Hint:** Yes, absolutely. You can't have *thread* races (no torn memory, no two writes in the same instant), but you can have *interleaving* races. Every `await` is a yield point: another task can run between your read and your write. Give the two-withdrawals-from-£100 example, show that the balance drops by 50 while 100 is paid out, and name it a lost update / read-modify-write race. Then say the real fix in a multi-process deployment is an atomic DB update or a row lock — an in-process mutex only protects one process.

### Q2: "Explain the Node event loop. Why does `process.nextTick` run before `setTimeout(fn, 0)`?"

**Hint:** Call stack runs sync code. When it empties, Node drains microtasks (nextTick queue first, then the promise queue) *completely*, then walks the libuv phases: timers → pending → poll → check → close. `setTimeout` callbacks are macrotasks in the **timers** phase; `nextTick` and promise callbacks are microtasks drained **between** macrotasks, so they always jump the queue. Bonus: mention that an infinitely self-scheduling `nextTick` can starve the loop entirely.

### Q3: "Your login endpoint gets slow under load. CPU is only at 30%. What's happening?"

**Hint:** Almost certainly the **libuv thread pool**. `bcrypt`/`scrypt`/`pbkdf2` run there, and it has **4 threads by default**. At ~250 ms per hash, the 9th concurrent login waits ~750 ms — a queue, not a CPU limit, which is why CPU looks fine. Fixes: raise `UV_THREADPOOL_SIZE` toward core count, move hashing into `worker_threads`/a pool, lower the bcrypt cost factor if security allows, or scale out with `cluster`. Mention that `fs` and `zlib` share the same 4 threads, so a chatty file-logging library competes with your logins.

### Q4: "What is a deadlock and how do you prevent it?"

**Hint:** Four Coffman conditions must hold at once — mutual exclusion, hold-and-wait, no preemption, circular wait — and breaking any one prevents it. Give the A→B / B→A transfer example. The practical fix is **consistent global lock ordering** (sort the account IDs and always lock in that order), which structurally eliminates circular wait. Add lock timeouts (breaks no-preemption) as a safety net, and mention that a deadlock in Node presents as a silent hang, not a crash — health checks still pass.

### Q5: "You need to call a third-party API for 10,000 records, but they allow 5 concurrent connections. How?"

**Hint:** Not `Promise.all` (10,000 sockets, instant ban) and not a serial `for await` loop (far too slow). Use a **semaphore / concurrency limiter**: N permits, acquire before the call, release in a `finally`, waiters queue up in FIFO order. Sketch the `mapLimit(items, 5, fn)` helper. Mention `p-limit`/`piscina` as the off-the-shelf versions, and add retry-with-jittered-backoff for 429s — plain fixed-interval retries can livelock.

### Q6: "You run 8 workers with `cluster`. Your in-memory rate limiter allows 100 req/min. What do users actually get?"

**Hint:** 800/min. Each `cluster` worker is a separate process with its own heap, so the counter `Map` exists eight times. Same trap for in-memory caches, Singletons, session stores, and `setInterval` cron jobs (which fire eight times). Fix: move shared state to Redis, and elect a single worker (or use a real scheduler) for periodic jobs. This is *the* classic cluster bug.

---

## Practice exercise

### Build a Rate-Limited, Deadlock-Free Bank

Create `bank.js` and do all five steps. Budget ~35 minutes.

1. **Reproduce the bug.** Write a fake `db` with `getBalance(id)` / `setBalance(id, v)` that each `await sleep(20)`. Start `acct-1` at 100. Write `withdraw(id, amount)` as a naive read-modify-write. Fire `Promise.all([withdraw('acct-1', 50), withdraw('acct-1', 50)])` and confirm the final balance is **50** while **100** was paid out. Print the interleaving with timestamped logs so you can *see* it.

2. **Build the `Mutex`.** Implement `acquire()` returning a `release` function, backed by a FIFO array of pending resolvers, plus a `runExclusive(fn)` helper that releases in a `finally`. Use a **per-account** mutex (a `Map<accountId, Mutex>`). Re-run step 1 and confirm one withdrawal succeeds and the other is declined.

3. **Build the `Semaphore`.** Generalize the mutex to N permits and write `mapLimit(items, limit, fn)`. Test it with 20 fake API calls at `limit: 3`, logging "start"/"end" with a running in-flight counter — assert the counter never exceeds 3.

4. **Cause a deadlock, then kill it.** Write `transfer(from, to, amount)` that locks both accounts and `await`s between the two `acquire()` calls. Run `transfer('alice','bob',50)` and `transfer('bob','alice',30)` concurrently and watch it hang forever (add a 3-second `setTimeout` that prints "DEADLOCKED" so you get a clear signal). Then fix it with **sorted lock ordering** and confirm both transfers complete.

5. **Prove the event loop blocks.** Add a `setInterval(() => console.log('heartbeat'), 100)`. Then run a synchronous `for (let i = 0; i < 5e9; i++) {}` loop. Observe the heartbeat stop dead for the duration and then fire late. Finally, move that loop into a `worker_threads` Worker and confirm the heartbeat keeps ticking.

**Deliverable:** one file that prints, in order: the broken output, the fixed output, the semaphore trace, the deadlock and its fix, and the blocked-vs-unblocked heartbeat. If you can explain each section out loud to someone, you know this topic.

---

## Quick reference cheat sheet

- **Concurrency ≠ parallelism.** Concurrency = dealing with many things at once (interleaving, one barista). Parallelism = doing many things at once (many baristas).
- **Every `await` is a yield point.** Your function is atomic *between* awaits, never *across* them. This is the whole reason JS has race conditions.
- **Single-threaded protects you from torn memory, not from lost updates.** Check-then-act across an `await` is always a bug waiting to happen.
- **Critical section** = the read-modify-write that must be indivisible. Find it first, then decide how to protect it.
- **Mutex** = 1 holder. **Semaphore** = N holders. In JS both are just a queue of pending `resolve` functions.
- **Always `release()` in a `finally`.** A throw inside a critical section without a finally deadlocks every waiter, permanently.
- **Lock the narrowest thing.** Per-account, not global — a global mutex turns your concurrent server into a sequential one.
- **In-process locks don't work across processes.** Under `cluster` or k8s replicas, push atomicity into the DB (`UPDATE ... WHERE balance >= $1`) or use Redis.
- **Deadlock = 4 Coffman conditions** (mutual exclusion, hold-and-wait, no preemption, circular wait). Break **one** and it's impossible. The practical fix is **consistent global lock ordering**.
- **A deadlock in Node is a silent hang**, not a crash. Health checks keep passing. Add lock timeouts.
- **Never hold a lock across an `await` you don't control** (a third-party HTTP call). A slow vendor becomes your outage.
- **Livelock** = busy but no progress (fix: jittered backoff). **Starvation** = never scheduled (fix: FIFO fairness or ageing).
- **Thread pool sizing:** CPU-bound ≈ number of cores; IO-bound ≈ `cores × (1 + wait/compute)`, often much higher.
- **`UV_THREADPOOL_SIZE` defaults to 4** and is used by `fs`, `crypto`/`bcrypt`, `zlib`, `dns.lookup`. Four concurrent bcrypt hashes saturate it — the 9th login waits ~750 ms while CPU sits idle. Sockets do *not* use it.
- **Event loop phases:** timers → pending → poll → check → close. Microtasks (`process.nextTick`, then promises) drain **fully** between every macrotask, which is why they beat `setTimeout(fn, 0)`.
- **A synchronous CPU loop blocks the entire server** — no timers, no sockets, no health checks. Offload to `worker_threads` or chunk with `setImmediate`.
- **`worker_threads`** = CPU work in-process, shared memory possible. **`cluster`** = scale HTTP across cores, fully isolated memory.
- **Under `cluster`, every process has its own heap** — your in-memory cache, rate-limiter, Singleton, and `setInterval` cron all exist N times. Classic bug. Move shared state to Redis.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [51 — Dependency Injection](./18-solid-dependency-inversion.md) — DI is how you inject a *shared* Mutex, Semaphore, or worker pool into the classes that need it, instead of reaching for a module-level global |
| **Next** | [53 — Idempotency](./85-idempotency.md) — the production answer to the race we saw here: make the operation safe to run twice, so retries and interleavings can't double-charge |
| **Related** | [30 — Singleton Pattern](./29-pattern-singleton.md) — the "one shared instance" assumption that quietly breaks under `cluster`, where each process gets its own |
| **Related** | [08 — Consistency Models](./08-consistency-models.md) — the same read-modify-write hazard, scaled up: lost updates across *machines* instead of across `await`s |
| **Related** | [21 — Load Balancing](./55-load-balancing.md) — how requests get spread across the `cluster` workers and containers whose isolated memory we just warned about |
