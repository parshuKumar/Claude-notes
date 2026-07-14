# 29 — Singleton Pattern

## Category: LLD Patterns

---

## What is this?

The **Singleton** pattern guarantees that a class has **exactly one instance** for the whole lifetime of your program, and gives everyone a **single, well-known way to reach it**.

Think of the **President of a country**. There is exactly one at any moment. You don't say `new President()` — you say "the President," and everyone knows who you mean. If two people could create two Presidents, the country would have two conflicting sets of orders. Singleton is that same idea in code: one object, one global access point.

---

## Why does it matter?

Some things in a program are *inherently* one thing:

- **A database connection pool.** Opening 200 TCP connections to Postgres is expensive. If every module created its own pool, you'd blow past your database's `max_connections` limit and the server would start rejecting you.
- **The app's configuration.** Config is read once from env vars/files. Two copies could drift out of sync.
- **A logger.** All logs should funnel through one writer to one file handle, or your log lines interleave and corrupt each other.

**The interview angle:** Singleton is the single most-asked creational pattern, *and* the most common trap. Interviewers ask it not because they want you to write one, but to see whether you know **why it's widely considered an anti-pattern**. A candidate who says "Singleton is great for global state" scores badly. A candidate who says "here's how I'd implement it, here's why it hurts testability, and here's the dependency-injection alternative I'd actually use" scores very well.

**The real-work angle:** In Node you use Singletons every day without noticing — every `require()`/`import` of a module gives you the *same* cached instance. Understanding that is the difference between shipping a working DB pool and shipping a memory leak.

---

## The core idea — explained simply

### The Office Printer Analogy

Your office has **one** printer. It sits in the corner. Every employee prints to it.

Now imagine the rule was: "whenever you need to print, buy a new printer." You'd have 40 printers, 40 ink budgets, 40 paper trays that are individually half-empty, and jobs scattered across all of them. Absurd.

Instead there's a rule: **you cannot buy a printer.** You call `Office.getPrinter()`. The first time anyone calls it, the office buys one and puts it in the corner. Every call after that hands you the same printer.

That's a Singleton. Three moving parts:

1. **You are forbidden from creating one yourself** (a private/blocked constructor).
2. **The class holds the one instance internally** (a static field).
3. **There is one public door to reach it** (`getInstance()`).

But now the downside, which is the whole point of this doc:

Someone in Accounting hardcodes "print the report to `Office.getPrinter()`" deep inside their code. Now you want to *test* the Accounting code without physically printing 500 pages. You can't. The dependency on the printer is **hidden inside the function body**, not declared in its signature. You can't hand it a fake printer, because it never asked for one — it went and grabbed the global.

That is exactly what Singleton does to your unit tests.

| Analogy piece | Technical concept |
|---|---|
| The one office printer | The single instance |
| "You may not buy a printer" | Blocked / private constructor |
| `Office.getPrinter()` | `getInstance()` — the global access point |
| Buying it the first time someone asks | Lazy initialization |
| Accounting code reaching for the global printer | Hidden dependency (the anti-pattern) |
| Handing Accounting a printer at the door | Dependency Injection (the fix) |
| Each branch office has its own printer | Each Node process/worker has its OWN singleton |

That last row is the bug that bites people in production. Hold onto it.

---

## Key concepts inside this topic

### 1. The problem it solves — painful code without the pattern

Here is a perfectly reasonable-looking Node app that is quietly on fire.

```javascript
// ---------- BAD: everyone makes their own pool ----------
// db.js
import { Pool } from 'pg';

export function makePool() {
  // Each call opens up to 20 real TCP sockets to Postgres.
  return new Pool({ connectionString: process.env.DATABASE_URL, max: 20 });
}

// userRepo.js
import { makePool } from './db.js';
const pool = makePool();                 // pool #1  → 20 sockets
export const findUser = (id) => pool.query('SELECT * FROM users WHERE id=$1', [id]);

// orderRepo.js
import { makePool } from './db.js';
const pool = makePool();                 // pool #2  → 20 more sockets
export const findOrder = (id) => pool.query('SELECT * FROM orders WHERE id=$1', [id]);

// auditRepo.js  → pool #3 ... and so on for 15 repositories.
```

Fifteen repositories × 20 connections = **300 connections**. Postgres defaults to `max_connections = 100`. Your app boots fine in dev (only 2 repos get loaded), then dies in production at 3am with:

```
error: sorry, too many clients already
```

You wanted **one** pool. Nothing in the code says so, so nothing enforces it. That's the gap Singleton fills.

### 2. The structure — who plays which role

Singleton has the fewest participants of any GoF pattern — really just one class doing double duty.

```
┌─────────────────────────────────────────────┐
│              «Singleton»                    │
│                Logger                       │
├─────────────────────────────────────────────┤
│ - static #instance : Logger    ← holds it   │
│ - #stream          : WriteStream            │
├─────────────────────────────────────────────┤
│ - constructor()      ← blocked to outsiders │
│ + static getInstance() : Logger  ← the door │
│ + log(level, msg)    : void                 │
└─────────────────────────────────────────────┘
        ▲            ▲             ▲
        │            │             │
   ┌────┴────┐  ┌────┴─────┐  ┌────┴─────┐
   │UserSvc  │  │OrderSvc  │  │PaymentSvc│   ← Clients: all get the
   └─────────┘  └──────────┘  └──────────┘     SAME object back
```

- **Singleton (Logger):** owns the instance, blocks direct construction, exposes `getInstance()`.
- **Clients:** never `new` it; they call the access point.

### 3. Three JavaScript implementations (and which one you should actually use)

**(a) The module-level instance — the idiomatic Node way.**

Node's module system caches every module by its resolved path. The file body runs **once**, no matter how many files import it. So a module *is already a singleton*, for free.

```javascript
// logger.js
class Logger {
  #history = [];

  log(level, message) {
    const line = `[${new Date().toISOString()}] ${level.toUpperCase()}: ${message}`;
    this.#history.push(line);
    console.log(line);
  }
  info(msg)  { this.log('info', msg); }
  error(msg) { this.log('error', msg); }
  get history() { return [...this.#history]; }   // copy — don't leak internals
}

// This line runs exactly ONCE per process, no matter how many files import it.
export default new Logger();
```

```javascript
// a.js and b.js both do:
import logger from './logger.js';
logger.info('hello');
// a.js's logger === b.js's logger. Same object. Guaranteed by the module cache.
```

This is what `module.exports = new Logger()` (CommonJS) or `export default new Logger()` (ESM) buys you. **90% of the time in Node, this is the right answer** — no ceremony, no static fields, no null checks.

**(b) The classic `getInstance()` — the one interviewers want to see.**

```javascript
// config.js
class Config {
  static #instance = null;
  static #allowConstruction = false;   // guard: fake "private constructor"

  #values;

  constructor() {
    if (!Config.#allowConstruction) {
      throw new Error('Config is a singleton — use Config.getInstance()');
    }
    this.#values = Object.freeze({
      port:     Number(process.env.PORT ?? 3000),
      dbUrl:    process.env.DATABASE_URL ?? 'postgres://localhost/dev',
      logLevel: process.env.LOG_LEVEL ?? 'info',
    });
  }

  static getInstance() {
    if (Config.#instance === null) {
      Config.#allowConstruction = true;
      Config.#instance = new Config();
      Config.#allowConstruction = false;
    }
    return Config.#instance;
  }

  get(key) { return this.#values[key]; }
}

export default Config;
```

JavaScript has no `private constructor` keyword, so the `#allowConstruction` flag is how you emulate one. `new Config()` from outside now throws.

**(c) The lazy-init variant — build it only when first needed.**

`getInstance()` above is *already* lazy: the object isn't created until the first call. That matters when construction is expensive (opening sockets, reading files). The eager alternative builds it at import time:

```javascript
// EAGER: pays the cost even if nobody ever uses it
const instance = new ExpensiveThing();      // runs at import
export default instance;

// LAZY: pays the cost on first use only
let instance = null;
export function getExpensiveThing() {
  if (!instance) instance = new ExpensiveThing();   // built once, on demand
  return instance;
}
```

Use lazy when the singleton opens real resources (a DB pool, a file handle) and some code paths — like a CLI subcommand that only prints `--version` — never need it.

### 4. "Thread safety" in Node — mostly moot, until it isn't

In Java or C++, `getInstance()` is dangerous: two threads can both see `instance === null` at the same instant and both construct one. You need `synchronized` blocks or double-checked locking.

**In Node, your JavaScript runs on a single thread with an event loop.** Nothing can interrupt you between the `if (!instance)` check and the `instance = new Thing()` assignment, because there's no other thread to interrupt you. Classic Singleton race conditions **do not exist** for synchronous constructors. If an interviewer asks "how do you make this thread-safe in JS?", the correct answer is *"you don't need to — the event loop is single-threaded"*, and that answer alone impresses.

**But there are two real gotchas.**

**Gotcha A — async construction still races.** If building the instance is `async`, another request *can* slip in during the `await`:

```javascript
// BROKEN: two concurrent callers both start connecting
let pool = null;
export async function getPool() {
  if (!pool) {
    pool = await connectToDatabase();   // ← await yields! caller #2 sees pool === null
  }
  return pool;
}
```

The fix is to **cache the promise, not the value** — the promise exists immediately, so the second caller finds it:

```javascript
// CORRECT: the promise is stored synchronously, so there's no window to race in
let poolPromise = null;
export function getPool() {
  if (!poolPromise) poolPromise = connectToDatabase();  // assigned before any await
  return poolPromise;                                   // both callers await the SAME promise
}
```

**Gotcha B — one singleton PER PROCESS. This is the classic production bug.**

Node scales by running multiple processes (`cluster`, PM2, Kubernetes replicas) or `worker_threads`. Each one has its **own** module cache, its own heap, its own singleton.

```javascript
// rateLimiter.js  — looks fine, is broken at scale
class RateLimiter {
  #hits = new Map();
  allow(userId, limit = 100) {
    const n = (this.#hits.get(userId) ?? 0) + 1;
    this.#hits.set(userId, n);
    return n <= limit;
  }
}
export default new RateLimiter();   // in-memory state!
```

Run this under `cluster` with 4 workers. A user's requests get load-balanced across all 4. Each worker has its own `#hits` Map, each counting to 100. Your "100 requests/min" limit is actually **400 requests/min**. The bug is invisible in dev (1 process) and only appears in production.

**Rule:** a Singleton guarantees *one instance per process* — never *one instance per system*. If the state must be shared across processes, it belongs in **Redis or the database**, not in a Singleton's memory. In-process singletons should hold only things that are naturally per-process: a connection pool, a logger's file handle, parsed config.

### 5. A real Node.js ecosystem example

You already depend on singletons:

- **`process`, `console`, `require.cache`** — Node hands you one of each per process. `process` is a Singleton with a global access point that isn't even a method call.
- **Mongoose.** `import mongoose from 'mongoose'` returns a singleton `Mongoose` instance holding the connection. That's why you call `mongoose.connect()` once at boot and every model file magically shares the connection — they all imported the same object.
- **Express's `app`.** Conventionally created once in `app.js` and exported; every router imports the same one.
- **Winston / Pino default loggers.** `import pino from 'pino'` lets you make many, but the common pattern is exactly `export default pino()` from a `logger.js` — module-level singleton.

### 6. When NOT to use it / how it's abused

Singleton is on almost every "patterns considered harmful" list. Here is precisely why — with the pain, not just the slogan.

**(a) It hides dependencies.** Look at this signature:

```javascript
async function placeOrder(cart) { ... }
```

What does it need? From the signature: a cart. From the *body*: a DB, a logger, a payment gateway, and the config — all yanked out of globals. The signature lies. Nobody reading the call site can tell what this function touches.

**(b) It destroys testability.** Watch the test pain:

```javascript
// ---------- BAD: singleton reached from inside ----------
// orderService.js
import db from './db.js';            // singleton
import logger from './logger.js';    // singleton
import paymentGateway from './payments.js';  // singleton — talks to real Stripe!

export async function placeOrder(cart) {
  logger.info(`placing order for ${cart.userId}`);
  const charge = await paymentGateway.charge(cart.userId, cart.total);
  await db.query('INSERT INTO orders(user_id, total, charge_id) VALUES($1,$2,$3)',
                 [cart.userId, cart.total, charge.id]);
  return charge.id;
}
```

To unit-test `placeOrder` you must:
- run a real Postgres (or monkey-patch the module registry with `jest.mock`, which is global mutation with extra steps),
- avoid actually charging a credit card,
- and reset the singleton's internal state between tests — except **you can't**, because the module cache holds it and tests in the same process share it. Test A pollutes Test B. Tests pass alone and fail in suite. Everyone has debugged this for an afternoon.

**(c) It violates the Dependency Inversion Principle.** Recall from [18 — SOLID: Dependency Inversion] that high-level modules should depend on *abstractions*, not concrete classes. `import db from './db.js'` is a hard, compile-time dependency on one concrete class. You cannot substitute an in-memory fake, a read-replica client, or a different vendor without editing `orderService.js`.

**The fix — Dependency Injection.** Keep the singleton (one pool is still correct!) but **stop reaching for it from inside**. Pass it in.

```javascript
// ---------- GOOD: dependencies are declared, not smuggled ----------
// orderService.js — knows NOTHING about where its collaborators come from
export class OrderService {
  constructor({ db, logger, paymentGateway }) {
    this.db = db;
    this.logger = logger;
    this.paymentGateway = paymentGateway;
  }

  async placeOrder(cart) {
    this.logger.info(`placing order for ${cart.userId}`);
    const charge = await this.paymentGateway.charge(cart.userId, cart.total);
    await this.db.query(
      'INSERT INTO orders(user_id, total, charge_id) VALUES($1,$2,$3)',
      [cart.userId, cart.total, charge.id],
    );
    return charge.id;
  }
}

// main.js — the COMPOSITION ROOT: the one place that knows about singletons
import db from './db.js';
import logger from './logger.js';
import paymentGateway from './payments.js';
import { OrderService } from './orderService.js';

export const orderService = new OrderService({ db, logger, paymentGateway });
```

Now the test is trivial, fast, and hermetic:

```javascript
// orderService.test.js
import { OrderService } from './orderService.js';

test('places an order and records the charge', async () => {
  const queries = [];
  const fakeDb      = { query: (sql, params) => { queries.push([sql, params]); } };
  const fakeLogger  = { info: () => {} };
  const fakeGateway = { charge: async () => ({ id: 'ch_123' }) };

  const svc = new OrderService({ db: fakeDb, logger: fakeLogger, paymentGateway: fakeGateway });
  const chargeId = await svc.placeOrder({ userId: 'u1', total: 4200 });

  expect(chargeId).toBe('ch_123');
  expect(queries).toHaveLength(1);
  expect(queries[0][1]).toEqual(['u1', 4200, 'ch_123']);
});
```

No database. No Stripe. No shared state between tests. **The object is still a singleton — there's still one pool in production — but nothing *depends* on its singleton-ness.** That's the whole lesson: *single instance* is a lifecycle decision, made once in `main.js`. *Global access* is a coupling decision, and it's the one that hurts.

### 7. Related patterns and how they differ

| Pattern | Core intent | vs Singleton |
|---|---|---|
| **Monostate** | Many instances, but all share the *same static state* | Singleton controls the *instance*; Monostate controls the *state*. Callers can `new` freely and still see one shared state. |
| **Factory Method** ([30](./30-pattern-factory-method.md)) | Defer *which class* to instantiate | Factory usually returns a **new** object each call; Singleton always returns the **same** one. A factory *can* return a cached singleton — they compose. |
| **Flyweight** ([40](./40-pattern-flyweight.md)) | Share many small immutable objects to save memory | Flyweight has a *pool* of shared objects keyed by value; Singleton has exactly one. |
| **Facade** ([36](./36-pattern-facade.md)) | Simplify a complex subsystem behind one interface | Facades are *often implemented* as singletons, but that's incidental — a Facade could have many instances. |
| **DI Container** | Manages object lifetimes (`singleton` / `transient` / `scoped`) | This is the modern replacement: you get one-instance semantics **without** global access. |

**The one-liner for interviews:** Singleton controls *how many* exist. Factory controls *which kind* gets built. A DI container controls *both*, from the outside, without poisoning your call sites.

---

## Visual / Diagram description

### Diagram 1 — Lifecycle of `getInstance()`

```
   First call                            Every call after
   ─────────                             ────────────────

   Client                                Client
     │  getInstance()                      │  getInstance()
     ▼                                     ▼
 ┌────────────────┐                   ┌────────────────┐
 │  #instance     │                   │  #instance     │
 │  === null ?    │──── yes ──┐       │  === null ?    │─── no ──┐
 └────────────────┘           │       └────────────────┘         │
                              ▼                                  │
                     ┌──────────────────┐                        │
                     │ new Logger()     │  ← runs ONCE           │
                     │ open file handle │                        │
                     └────────┬─────────┘                        │
                              │ store in #instance               │
                              ▼                                  ▼
                        ┌──────────────────────────────────────────┐
                        │        return the SAME Logger            │
                        └──────────────────────────────────────────┘
```

The only branch that ever constructs is the first one. Everything after is a cache hit.

### Diagram 2 — The per-process trap (draw this one on the whiteboard)

```
                         ┌───────────────────┐
   requests ────────────▶│   Load Balancer   │
                         └─────┬──────┬──────┘
                 ┌─────────────┘      └─────────────┐
                 ▼                                  ▼
   ┌──────────────────────────┐       ┌──────────────────────────┐
   │  Node process (worker 1) │       │  Node process (worker 2) │
   │  ┌────────────────────┐  │       │  ┌────────────────────┐  │
   │  │ RateLimiter #1     │  │       │  │ RateLimiter #2     │  │
   │  │  hits: {u1 → 100}  │  │       │  │  hits: {u1 → 100}  │  │
   │  └────────────────────┘  │       │  └────────────────────┘  │
   └──────────────────────────┘       └──────────────────────────┘
              ▲                                    ▲
              └──── u1 has actually made 200 ──────┘
                    requests, but each worker
                    thinks it's only 100.  BUG.

   THE FIX: move shared state out of the process.

   ┌───────────┐   ┌───────────┐   ┌───────────┐
   │ worker 1  │   │ worker 2  │   │ worker 3  │
   └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
         └───────────────┼───────────────┘
                         ▼
                 ┌───────────────┐
                 │     Redis     │  ← ONE counter, shared by all
                 │  u1 → 200     │
                 └───────────────┘
```

The singleton is still fine as a *client* to Redis (one connection per process). What must not live in it is the *shared counter*.

---

## Real world examples

### 1. Node.js core — `process` and the module cache

Node itself ships singletons. `process` is one object per process, reachable globally with no `getInstance()` at all. More interestingly, `require.cache` / the ESM module map **is the mechanism** that makes `export default new Logger()` a singleton: Node resolves the specifier to an absolute path, and returns the cached module namespace on every subsequent import. This is also why importing the *same* file via two different paths (e.g. a symlinked `node_modules` copy) can give you **two** instances — a genuinely nasty bug that has bitten React (`Invalid hook call — two copies of React`) and Mongoose users alike.

### 2. Mongoose — one connection, many models

`mongoose` exports a single `Mongoose` instance. You call `mongoose.connect(uri)` once at boot; every `mongoose.model(...)` call in every file registers against that same instance's default connection. Without this singleton you'd have to thread a connection object into every model definition. Note the trade-off Mongoose accepted: this design is exactly why testing Mongoose models in isolation is awkward, and why the library also exposes `mongoose.createConnection()` — an escape hatch back to explicit, injectable connections.

### 3. Database connection pools (`pg`, `mysql2`) — the legitimate case

A pool is the textbook-correct singleton. It owns a fixed, expensive, finite resource (TCP sockets to the DB) that must be *shared and bounded*. The pool itself is safely a per-process singleton because connection limits are usually configured per-process anyway (e.g. `max: 10` × 5 replicas = 50 connections, sized against Postgres's `max_connections`). This is a case where you want one instance — and where you should still **inject** it into your repositories rather than importing it inside them.

---

## Trade-offs

| Benefit | What it buys you |
|---|---|
| **Guaranteed single instance** | Bounded expensive resources (sockets, file handles); no accidental duplication |
| **Lazy initialization** | Cost paid only if the object is actually used |
| **Global access** | Any code, anywhere, can reach it with zero wiring |
| **Simple in Node** | A one-line module export gives it to you for free |

| Cost | What it takes away |
|---|---|
| **Hidden dependencies** | Function signatures no longer tell the truth about what a function touches |
| **Broken unit tests** | Can't substitute fakes; state leaks between tests in the same process |
| **Violates DIP** ([18](./18-solid-dependency-inversion.md)) | Hard-coded dependency on a concrete class |
| **Violates SRP** ([14](./14-solid-single-responsibility.md)) | The class does its job *and* manages its own lifecycle — two reasons to change |
| **Per-process only** | Silently wrong under `cluster` / `worker_threads` / multiple pods |
| **Hard to subclass or reconfigure** | One instance means one configuration, forever |

**The sweet spot:** Create the object **once** in a composition root (`main.js`) — that gives you the single-instance guarantee. Then **inject it** everywhere it's needed — that gives you testability. You get 100% of Singleton's benefit and 0% of its cost. Reach for a true `getInstance()` Singleton only for process-wide infrastructure with no meaningful alternative (logger, config, pool) — and never, ever put shared business state in one.

**Rule of thumb:** If you find yourself writing `SomethingSingleton.getInstance()` inside a business-logic method, stop. That's the smell. Move it to a constructor parameter.

---

## Common interview questions on this topic

### Q1: "Implement a thread-safe Singleton."
**Hint:** First clarify the language. In Java you'd need double-checked locking with a `volatile` field, or the enum trick. In **Node/JS, there is no thread-safety problem for synchronous construction** — the event loop is single-threaded, so nothing can preempt you between the null check and the assignment. Say that out loud; it shows you understand the runtime rather than reciting Java. Then raise the two real JS hazards: (a) an **async** initializer *can* race — fix by caching the *promise*, not the value; (b) `worker_threads`/`cluster` give each thread/process its **own** singleton.

### Q2: "Why is Singleton considered an anti-pattern?"
**Hint:** Three concrete reasons, in this order: it's **global mutable state** with all the aliasing bugs that implies; it makes **unit testing** nearly impossible because you can't inject a fake and can't reset state between tests; and it **violates DIP** by hard-coding a dependency on a concrete class. Then land the nuance: the problem isn't "one instance" — that's often correct. The problem is **global access**. Dependency injection gives you one instance without global access.

### Q3: "How do you test code that uses a Singleton?"
**Hint:** Honestly? You mostly fight it. Options, from worst to best: monkey-patch the module registry (`jest.mock`) — works, but it's global mutation and brittle; add a `resetForTests()` method — leaks test concerns into production code; **refactor to inject the dependency** — the real answer. Follow up by showing the constructor-injection version and pointing out that the test now needs no mocking framework at all.

### Q4: "In Node, isn't every module already a Singleton? Why would I ever write `getInstance()`?"
**Hint:** Yes — the module cache makes `export default new Logger()` a per-process singleton, and that's the idiomatic Node answer. You'd write `getInstance()` when you need (a) **lazy** construction because the constructor is expensive, (b) an explicit guard so `new Config()` throws a helpful error, or (c) you're writing in a language/framework style where the module cache doesn't apply. Also mention the module-cache caveat: two copies of a package in `node_modules` = two "singletons."

### Q5: "Give me a case where Singleton is genuinely the right call, and one where it isn't."
**Hint:** **Right:** a Postgres connection pool — one bounded, expensive, per-process resource; duplicating it exhausts `max_connections`. **Wrong:** an in-memory rate limiter or session store — that state must be shared across *processes*, and a Singleton only guarantees one instance *per process*, so it silently under-counts by a factor of N under `cluster`. That state belongs in Redis.

---

## Practice exercise

### Build It, Break It, Fix It — the ConnectionPool

Create a file `pool-exercise.js`. Work in three stages; write real, runnable Node.

**Stage 1 — Build it.** Write a `ConnectionPool` Singleton:
- A private static `#instance` field and a `static getInstance(config)` method.
- The constructor must **throw** if called directly with `new` (use the guard-flag technique from section 3b).
- It should hold `#connections` (an array of fake connection objects `{ id, inUse }`) sized by `config.max`, and expose `acquire()` and `release(conn)`.
- Add a `console.log('POOL CREATED')` inside the constructor.
- Demo: call `getInstance()` from three different functions, `acquire()` a connection in each, and prove that `'POOL CREATED'` prints **exactly once** and that all three see the same pool.

**Stage 2 — Break it.** Now write an `OrderRepository` class that calls `ConnectionPool.getInstance()` *inside* its `save()` method. Then try to write a unit test for `save()` that does **not** touch the pool. Actually attempt it. Feel the pain — write a one-paragraph comment in the file explaining exactly why you can't.

**Stage 3 — Fix it.** Refactor `OrderRepository` to take `pool` as a constructor parameter. Move the `ConnectionPool.getInstance()` call to a `main()` composition root at the bottom of the file. Now write the test with a hand-rolled fake: `{ acquire: () => ({ id: 'fake' }), release: () => {} }`. It should pass in milliseconds with no real pool.

**What to produce:** one file, three sections clearly commented `// STAGE 1`, `// STAGE 2`, `// STAGE 3`, that runs with `node pool-exercise.js`. The point isn't the code — it's that you *felt* Stage 2.

---

## Quick reference cheat sheet

- **Intent:** Ensure a class has exactly one instance and provide a global point of access to it.
- **Participants:** Just one — the Singleton class holds a static instance, blocks its constructor, and exposes `getInstance()`.
- **Idiomatic Node:** `export default new Logger()` — the module cache makes it a singleton for free. Use this by default.
- **Explicit form:** private static `#instance` + `static getInstance()` + guarded constructor. Use when construction is expensive (lazy) or you want a loud error on `new`.
- **Lazy vs eager:** lazy = build on first `getInstance()`; eager = build at import. Prefer lazy when the constructor opens real resources.
- **Thread safety in Node:** a non-issue for sync constructors — the event loop is single-threaded. Say this in interviews.
- **Async init DOES race:** cache the **promise**, not the resolved value, so concurrent callers share one in-flight construction.
- **ONE INSTANCE PER PROCESS, not per system.** `cluster`, PM2, `worker_threads`, and k8s replicas each get their own. Shared state belongs in Redis.
- **Legit uses:** config, logger, DB connection pool, metrics client — process-wide infrastructure.
- **The anti-pattern:** hidden dependencies, untestable code, DIP violation — all caused by *global access*, not by *one instance*.
- **The fix:** create it once in a composition root, then **inject** it. Same single instance, zero coupling.
- **Smell:** `X.getInstance()` appearing inside a business-logic method. Move it to a constructor parameter.
- **Interview one-liner:** "Singleton controls how many exist; Factory controls which kind gets built; a DI container does both from the outside without poisoning your call sites."

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [28 — What Are Design Patterns](./28-what-are-design-patterns.md) — the categories and why patterns exist at all |
| **Next** | [30 — Factory Method Pattern](./30-pattern-factory-method.md) — the other end of creation: not *how many*, but *which kind* |
| **Related** | [18 — SOLID: Dependency Inversion](./18-solid-dependency-inversion.md) — the principle Singleton violates, and the injection technique that fixes it |
| **Related** | [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md) — why the Node event loop makes "thread-safe Singleton" a non-question, and what `worker_threads` changes |
