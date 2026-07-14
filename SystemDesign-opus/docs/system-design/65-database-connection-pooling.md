# 65 — Database Connection Pooling
## Category: HLD Components

---

## What is this?

A **connection pool** is a small, fixed set of already-open database connections that your application borrows from, uses for a few milliseconds, and gives back — instead of opening a brand-new connection every time it needs to run a query.

Think of a **library with 20 reading desks**. Anyone can walk in and read, but there are only 20 desks. You take a free desk, read your book, and leave — the next person takes the desk. Nobody builds a new desk for each visitor, and nobody hauls their desk home. The desks are the connections; the pool is the librarian handing them out.

---

## Why does it matter?

Because a database connection is **shockingly expensive**, and almost nobody realises it until production melts.

Opening one costs a TCP handshake, a TLS handshake, an authentication round-trip, and server-side session setup — **tens of milliseconds**. And on PostgreSQL, every connection is a **whole operating-system process** that eats roughly **5–10 MB of RAM**. Two hundred connections is 1–2 GB of RAM burned before you run a single query.

**What breaks without a pool:**
- Every request pays a 30–50 ms latency tax before the first byte of SQL leaves your app.
- Under load, your app opens connections faster than it closes them. The DB hits `max_connections`, starts refusing *everyone* — including your healthy requests, including your monitoring, including the DBA trying to log in and fix it. This is a **connection storm**, and it turns a slow-day into a full outage.

**Interview angle:** "You have 20 app servers and a Postgres box. How many connections should the pool have?" is a real question, and the naive answer ("as many as possible") is the wrong one. Interviewers use it to see if you understand that **more concurrency past a point makes a database slower, not faster**.

**Work angle:** Pool exhaustion is one of the most common production incidents in Node backends. The single most common cause is a code path that borrows a connection and forgets to give it back. You will debug this at some point in your career. Better to know it now.

---

## The core idea — explained simply

### The Restaurant Kitchen Pass-Through Analogy

A restaurant has a kitchen (the database) and a dining room (your app). Waiters (requests) need to send orders to the kitchen.

**Without a pool:** Every time a waiter has an order, they *hire a new cook*. Interview them, check their ID, teach them where the pans are, let them cook one dish, then fire them. The hiring takes 40 minutes; the dish takes 30 seconds. And after 200 hires the kitchen is so crowded that the cooks can't move.

**With a pool:** The restaurant employs **exactly 10 cooks**, permanently. A waiter walks up to the pass-through window, hands an order to a free cook, waits for the dish, and walks away. If all 10 cooks are busy, the waiter **stands in a queue** at the window. If they've been queuing for 5 seconds and still no cook is free, they give up and tell the customer "sorry, we're overloaded" — which is far better than the whole restaurant grinding to a halt.

And here's the counter-intuitive bit that every engineer gets wrong: **hiring 100 cooks does not make the kitchen 10× faster.** There are only 4 stoves. 100 cooks in a kitchen with 4 stoves means 96 cooks elbowing each other, fighting over pans, and blocking the walkways. Throughput goes *down*.

**Mapping it back:**

| Restaurant | Database system |
|---|---|
| Kitchen | The database server |
| Cook | An open DB connection (a Postgres backend process) |
| Hiring a cook | TCP + TLS + auth + session setup (~30–50 ms) |
| Waiter | An incoming HTTP request / a Node async task |
| Pass-through window | The connection pool |
| Waiter standing in line | The pool's **wait queue** |
| Waiter giving up after 5s | **Acquire timeout** |
| Stoves | CPU cores + disk spindles — the real, finite resource |
| 100 cooks, 4 stoves | Context switching + lock contention — throughput collapse |
| Cook gets sick, is replaced | Connection **health check** and **recycling** |
| Waiter walks off holding a cook hostage | A **connection leak** (forgot to release) |

The pool's job is not to give you *more* database capacity. It's to give you a **controlled, bounded, reusable** slice of the capacity you already have — and to make the excess wait politely instead of killing the DB.

---

## Key concepts inside this topic

### 1. Why one connection is so expensive

Let's actually count what happens when `pg` opens a connection to a Postgres server in another AZ:

```
1. DNS lookup (usually cached)                          ~0–2 ms
2. TCP three-way handshake (SYN, SYN-ACK, ACK)          ~1 RTT   ≈ 1–2 ms
3. TLS handshake (certs, key exchange)                  ~2 RTT   ≈ 5–20 ms
4. Postgres startup packet + auth (SCRAM-SHA-256)       ~2 RTT   ≈ 2–5 ms
5. Server FORKS a new backend PROCESS for you           ~1–5 ms  + 5–10 MB RAM
6. Session setup: search_path, timezone, prepared stmts ~1–5 ms
                                                        ─────────────────────
                                                  TOTAL ≈ 15–50 ms, 5–10 MB
```

Now compare that with the query you actually wanted to run:

```
SELECT * FROM users WHERE id = 42;   -- indexed primary key lookup:  ~0.3 ms
```

**You spent 40 ms of setup to do 0.3 ms of work.** That's a 130× overhead. And you paid it *per request*.

The RAM point deserves emphasis because it's Postgres-specific and it's what actually kills you. MySQL uses a **thread** per connection (cheaper, ~256 KB). Postgres forks a **process** per connection. 500 Postgres connections ≈ **2.5–5 GB of RAM** consumed by processes that are mostly sitting idle. That is memory your DB is not using for its page cache.

### 2. What "no pool" actually looks like in production

Here is the failure, step by step. It is always the same story.

```
Normal day:  100 req/s, each opens+closes a connection.
             Connections live ~50 ms. Concurrent connections ≈ 100 × 0.05 = 5.  Fine.

Traffic spike / a slow query appears. Queries now take 500 ms instead of 50 ms.
             Concurrent connections ≈ 100 × 0.5 = 50.  Still OK.

Requests pile up because responses are slow. Clients retry. Now 400 req/s arriving.
             Concurrent connections ≈ 400 × 0.5 = 200.  Getting hot.

DB is now slower (200 processes fighting for 8 cores) → queries take 2 s.
             Concurrent connections ≈ 400 × 2 = 800.

max_connections = 500.  ──▶  "FATAL: sorry, too many clients already"
             Every new connection is REFUSED. Including healthy requests.
             Including your health-check endpoint. Including psql from your laptop.
```

This is a **connection storm**, and notice the vicious cycle: more connections → slower DB → connections held longer → more connections. It is a positive feedback loop, so it goes from "fine" to "outage" in seconds. A pool **caps** the loop: the pool never opens more than `max`, and the excess load waits in a queue in your app, where you can time it out cheaply.

### 3. How a pool actually works — the lifecycle

```
Pool configuration knobs:
  min              — connections kept open even when idle (warm, ready to go)
  max              — hard ceiling. Never more than this. THE important one.
  idleTimeoutMillis— close an idle connection after this long (shrink back to min)
  connectionTimeoutMillis — how long a caller waits in the queue before giving up
  maxLifetimeSeconds — retire a connection after N seconds even if healthy (recycling)
```

**Acquire path:**
1. Caller asks the pool for a connection.
2. Is there an **idle** connection? → validate it (a cheap `SELECT 1`, or trust a recent heartbeat) → hand it over.
3. No idle connection, but `total < max`? → **open a new one** (pay the 40 ms), hand it over.
4. No idle connection and `total == max`? → put the caller in the **wait queue**.
5. Caller waits. If a connection is released, the queue head gets it.
6. If `connectionTimeoutMillis` elapses first → throw an error. **This is a feature.** Failing fast under overload is what stops the storm.

**Release path:** caller finishes → returns the connection to the idle set → next queued caller (or nobody) takes it.

**Validation / health check:** connections die silently. A load balancer with a 350-second idle timeout, a DB failover, a network blip — and now your pool holds a socket that looks open but is dead. The pool must detect this. Two strategies:
- **Test on borrow:** run `SELECT 1` before handing it out. Safe, costs ~0.2 ms per acquire.
- **Background reaper:** periodically ping idle connections. Cheaper on the hot path, slightly riskier.

**Recycling (`maxLifetime`):** even a healthy connection gets retired after, say, 30 minutes. Why? It sheds server-side memory bloat, and it lets connections gracefully migrate after a DB failover instead of all reconnecting at once. Always set `maxLifetime` **shorter** than any network/DB idle timeout in front of the database.

### 4. Pool sizing — why bigger is NOT better

This is the section people get wrong, so let's kill the myth with a picture.

```
Throughput (queries/sec)
   ▲
   │            ┌──────●───────────┐
   │          ┌─┘  peak            └─┐
   │        ┌─┘                       └──┐
   │      ┌─┘                             └───┐
   │    ┌─┘                                   └────┐
   │  ┌─┘                                          └─────  collapse
   │┌─┘
   └┴────┬──────┬──────┬──────┬──────┬──────┬──────┬───▶ pool size
         5     10     20     50    100    200    500
              ▲                    ▲
        sweet spot          "we added more connections
        (~2× cores)          and it got SLOWER"
```

Why does it collapse? A database with 8 CPU cores can **truly execute only 8 things at once**. Everything beyond that isn't parallelism — it's queuing, with costs:

- **Context switching.** The OS scheduler thrashes between 200 runnable backend processes. Each switch flushes CPU cache lines. Real work per second drops.
- **Lock contention.** More concurrent transactions → more of them fighting over the same rows, the same index pages, the same buffer-pool latches. Waiting on a lock is 100% wasted time.
- **Memory pressure.** Each connection takes `work_mem` for sorts/hashes. 200 connections × 4 MB = 800 MB, which is 800 MB not spent on caching your data. Cache hit rate falls → more disk reads → slower.

So the queue has to exist *somewhere*. The question is only **where**. In the pool's wait queue (cheap, controllable, in your app's memory) or inside the database (expensive, uncontrollable, takes the whole DB down). **Always choose the pool.**

**The classic heuristic** (from the HikariCP project, widely quoted):

```
pool_size = (core_count × 2) + effective_spindle_count
```

- `core_count` = the **database server's** CPU cores (not your app server's!).
- `effective_spindle_count` = roughly how many concurrent disk I/O operations the storage can serve. For spinning disks, the number of disks. For SSD/NVMe/cloud volumes, this is fuzzy — use a small number, or just skip it.

The `× 2` exists because a connection isn't running on CPU all the time — it also blocks on disk I/O, so one core can usefully interleave ~2 connections.

**Worked example.** A `db.m6g.2xlarge` Postgres: 8 vCPU, gp3 SSD.

```
pool_size = (8 × 2) + ~1  ≈ 17   →  round to 20 per... wait. Per WHAT?
```

That's the trap. Read on.

### 5. The N × M multiplication trap

Your pool config says `max: 50`. Feels modest. But you run **20 app server instances** behind a load balancer.

```
20 app servers  ×  50 connections each  =  1000 connections to ONE database.

Postgres max_connections is typically 100 (default!) or a few hundred.
Even if you raise it: 1000 × 8 MB = 8 GB of RAM just for backend processes.
And 1000 concurrent backends on 8 cores = throughput collapse.

Your database dies. The pool didn't protect you — you multiplied it by 20.
```

The pool `max` is **per process**, and it multiplies by every process you run. And in Node, if you run 4 workers per box via `cluster` or PM2, multiply by 4 again. `20 boxes × 4 workers × 50 = 4000`.

**The right way to size it:** work **backwards from the database**.

```
Step 1. Total connections the DB can healthily serve  →  ~20   (from (8×2)+spindles)
Step 2. Leave headroom for migrations, psql, monitoring, replication  →  reserve 10
Step 3. Budget available to the app                   →  ~20 usable
Step 4. Divide by number of app processes:  20 processes → max: 1 each  ←── absurd!
```

Which is exactly why **you need an external pooler**.

**PgBouncer** (or Amazon RDS Proxy, or pgcat) sits *between* your app servers and Postgres. Your 20 app servers each open 50 connections **to PgBouncer** — which is cheap, because PgBouncer is a single lightweight process that multiplexes them onto a handful of *real* Postgres connections.

```
                             ┌──────────────┐
  20 app servers  ──1000──▶  │  PgBouncer   │  ──20──▶  ┌────────────┐
  (cheap client conns)       │ (multiplexer)│            │ PostgreSQL │
                             └──────────────┘            │  8 cores   │
                                                         └────────────┘
```

**PgBouncer's three pooling modes** — this is a guaranteed interview follow-up:

| Mode | A real DB connection is held for... | Concurrency you get | What breaks |
|---|---|---|---|
| **session** | the whole client session (until the client disconnects) | Almost none — same as no pooler | Nothing breaks, but it barely helps |
| **transaction** | one transaction only, then returned to the pool | **Huge** — this is the mode you want | Session state: `SET`, advisory locks, `LISTEN/NOTIFY`, server-side prepared statements, temp tables. `WITH HOLD` cursors break. |
| **statement** | one single statement | Maximum | **Multi-statement transactions are impossible.** Only for very special workloads. |

**Use `transaction` mode.** Just be aware it forbids session-scoped state — which is fine for 95% of web apps, because a normal request is "run a couple of queries, maybe in one transaction, done."

### 6. Serverless makes all of this worse

A Lambda / Cloud Function is a **process per concurrent invocation**. There is no shared pool, because there is no shared process.

```
Traffic spike → AWS spins up 500 concurrent Lambdas
              → each Lambda cold-starts, each opens its OWN connection
              → 500 connections to a DB that can handle 20
              → "FATAL: sorry, too many clients already"
              → your API returns 500s, which triggers retries, which spawns more Lambdas
```

You cannot pool your way out of this *inside* the function. `max: 1` per Lambda still gives you 500. Fixes, in order of preference:

1. **Put a pooler in front:** RDS Proxy / PgBouncer. It absorbs the 500 client connections onto ~20 real ones. This is the standard answer.
2. **Use an HTTP-based data API** (Neon serverless driver, PlanetScale HTTP, Aurora Data API) — no persistent TCP connection at all; each query is an HTTPS call to a service that owns the pool.
3. **Cap Lambda reserved concurrency** so the fan-out can't exceed what the DB can take. Crude but effective.
4. **Reuse the connection across warm invocations** — declare the pool *outside* the handler so it survives between invocations on the same container. This helps, but does nothing about the *number* of containers.

### 7. Full Node.js implementation with `pg`

```javascript
// db.js — ONE pool for the whole process. Created at module load, never per-request.
import pg from 'pg';
const { Pool } = pg;

export const pool = new Pool({
  host: process.env.DB_HOST,
  database: 'app',
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,

  // Sized backwards from the DB, then divided by process count.
  // DB has 8 cores → ~20 healthy connections total → 4 app processes → 5 each.
  max: 5,
  min: 1,                        // keep one warm so the first request isn't cold

  idleTimeoutMillis: 30_000,     // shrink back to min after 30s of quiet
  connectionTimeoutMillis: 5_000,// FAIL FAST if the queue is jammed. Do not set to 0.
  maxLifetimeSeconds: 1_800,     // recycle every 30 min — must be < the LB idle timeout
  statement_timeout: 10_000,     // a runaway query can't hold a connection forever
});

// A dead connection in the idle set emits 'error'. If you don't handle it,
// Node crashes the whole process with an unhandled error event.
pool.on('error', (err) => {
  console.error('[pool] idle client error', err.message);
});
```

**The simple case — let the pool manage acquire/release for you:**

```javascript
// pool.query() acquires, runs, and releases automatically. It CANNOT leak.
// Use this for every single-statement query. It is the safe default.
export async function getUser(id) {
  const { rows } = await pool.query('SELECT * FROM users WHERE id = $1', [id]);
  return rows[0] ?? null;
}
```

**The transaction case — you must borrow and return manually:**

```javascript
// A transaction needs the SAME connection for BEGIN...COMMIT, so you must
// check one out explicitly. This is where leaks are born.
export async function transferMoney(fromId, toId, amountCents) {
  const client = await pool.connect();          // ── ACQUIRE (may wait in the queue)
  try {
    await client.query('BEGIN');

    const { rows } = await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1 RETURNING balance',
      [amountCents, fromId]
    );
    if (rows.length === 0) throw new Error('INSUFFICIENT_FUNDS');

    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amountCents, toId]
    );

    await client.query('COMMIT');
    return rows[0].balance;
  } catch (err) {
    await client.query('ROLLBACK').catch(() => {}); // rollback may itself fail; swallow
    throw err;
  } finally {
    client.release();                            // ── RELEASE. ALWAYS. NO EXCEPTIONS.
  }
}
```

**The leak that will page you at 3 a.m.:**

```javascript
// ✗ BAD — the release is on the happy path only.
async function badTransfer(fromId, toId, amount) {
  const client = await pool.connect();
  await client.query('BEGIN');
  const { rows } = await client.query('SELECT balance FROM accounts WHERE id = $1', [fromId]);
  if (rows[0].balance < amount) {
    throw new Error('INSUFFICIENT_FUNDS');   // ◀── THROWS. release() never runs.
  }                                          //      Connection is gone FOREVER.
  // ... more queries ...
  await client.query('COMMIT');
  client.release();                          // only reached when nothing goes wrong
}

// Every failed transfer permanently burns one connection.
// max: 5  →  after 5 insufficient-funds errors, your app is DEAD.
// Symptom: every request hangs for exactly connectionTimeoutMillis, then errors.
// It looks like "the database is slow." It is not. You leaked.
```

The `finally` block is not a style preference. It is the entire safety mechanism. If you take one thing from this doc: **`pool.connect()` and `client.release()` must be paired by a `try/finally`, always.**

A helper that makes leaks structurally impossible:

```javascript
// Callers can no longer forget to release, because they never hold the client.
export async function withTransaction(fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await fn(client);     // your logic runs here
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK').catch(() => {});
    throw err;
  } finally {
    client.release();
  }
}

// Usage — clean, and impossible to leak:
await withTransaction(async (tx) => {
  await tx.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [500, 'A']);
  await tx.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [500, 'B']);
});
```

**Monitoring — the three numbers that tell you everything:**

```javascript
// pg exposes these as live counters. Export them to Prometheus/Datadog every 10s.
setInterval(() => {
  metrics.gauge('db.pool.total',    pool.totalCount);   // open connections (idle + busy)
  metrics.gauge('db.pool.idle',     pool.idleCount);    // sitting free, ready
  metrics.gauge('db.pool.waiting',  pool.waitingCount); // callers stuck in the QUEUE
}, 10_000);
```

How to read them:

| Signal | What it means | What to do |
|---|---|---|
| `waiting` is consistently > 0 | Demand exceeds pool capacity | Either the queries are too slow, or the pool is too small. **Check query latency first** — 9 times out of 10 the fix is an index, not a bigger pool. |
| `total == max` and `idle == 0` forever | Saturated, or **leaking** | If `waiting` grows unboundedly and never drains even when traffic stops → it's a leak. Grep for `pool.connect()` without a matching `finally`. |
| `total` stuck at `max` at 3 a.m. with zero traffic | **Definitely a leak** | Connections should have decayed to `min`. |
| Acquire-timeout errors spiking | Pool exhausted | Alert on this. It is the earliest reliable signal of a DB problem. |

Also track **acquire wait time** (a histogram: how long `pool.connect()` took). p99 acquire time should be near zero. If it's 500 ms, your app is spending half a second per request just waiting for a desk in the library.

---

## Visual / Diagram description

### Diagram 1: Anatomy of a connection pool

```
                       YOUR NODE.JS PROCESS
   ┌───────────────────────────────────────────────────────────────────┐
   │                                                                   │
   │  req A ─┐                                                         │
   │  req B ─┤                                                         │
   │  req C ─┼──▶ ┌──────────────────────────────────────────┐         │
   │  req D ─┤    │            CONNECTION POOL               │         │
   │  req E ─┘    │                                          │         │
   │              │  WAIT QUEUE (FIFO)                       │         │
   │              │  ┌────┬────┐   acquire timeout = 5000ms  │         │
   │              │  │ D  │ E  │   ──▶ exceeded? THROW       │         │
   │              │  └────┴────┘                             │         │
   │              │                                          │         │
   │              │  IN USE (3)          IDLE (0)            │         │
   │              │  ┌────┐┌────┐┌────┐                      │         │
   │              │  │ c1 ││ c2 ││ c3 │   max = 3            │         │
   │              │  └─┬──┘└─┬──┘└─┬──┘   min = 1            │         │
   │              └────┼─────┼─────┼──────────────────────────┘        │
   └───────────────────┼─────┼─────┼───────────────────────────────────┘
                       │     │     │   3 long-lived TCP+TLS sockets
                       ▼     ▼     ▼   (handshake paid ONCE, at startup)
                  ┌──────────────────────┐
                  │     POSTGRESQL       │
                  │  ┌────┐┌────┐┌────┐  │  ◀── each conn = one OS PROCESS
                  │  │proc││proc││proc│  │      ~5-10 MB RAM each
                  │  └────┘└────┘└────┘  │
                  │   8 CPU cores        │  ◀── the REAL concurrency limit
                  │   max_connections=100│
                  └──────────────────────┘
```

Requests A, B, C hold the three connections. D and E are **not talking to the database at all** — they are parked in the wait queue inside your Node process, costing nothing but a promise. When C finishes and releases, D is handed C's connection instantly (no handshake). If D waits more than 5 seconds, it gets an error and the request fails fast — which is *good*, because a fast failure sheds load instead of amplifying it.

### Diagram 2: The N × M trap and the PgBouncer fix

```
   ── WITHOUT A POOLER ───────────────────────────────────────────────
                                                     1000 connections
   ┌───────┐ ┌───────┐ ┌───────┐        ┌───────┐    ~8 GB RAM
   │ app 1 │ │ app 2 │ │ app 3 │  ...   │ app20 │    1000 procs / 8 cores
   │ max50 │ │ max50 │ │ max50 │        │ max50 │           │
   └───┬───┘ └───┬───┘ └───┬───┘        └───┬───┘           ▼
       └─────────┴─────────┴────────────────┘        ┌─────────────┐
                    50 × 20 = 1000 ───────────────▶  │  POSTGRES   │
                                                     │  💀 DEAD    │
                                                     └─────────────┘

   ── WITH PGBOUNCER (transaction mode) ──────────────────────────────

   ┌───────┐ ┌───────┐ ┌───────┐        ┌───────┐
   │ app 1 │ │ app 2 │ │ app 3 │  ...   │ app20 │
   │ max50 │ │ max50 │ │ max50 │        │ max50 │
   └───┬───┘ └───┬───┘ └───┬───┘        └───┬───┘
       └─────────┴─────────┴────────────────┘
                    1000 CHEAP client conns
                            │
                            ▼
                   ┌──────────────────┐
                   │    PGBOUNCER     │  single lightweight process,
                   │  pool_mode=      │  ~2 KB per client connection
                   │    transaction   │
                   │  default_pool_   │  holds a real conn only for the
                   │    size = 20     │  duration of ONE TRANSACTION
                   └────────┬─────────┘
                            │ 20 real server conns
                            ▼
                   ┌──────────────────┐
                   │    POSTGRES      │
                   │  20 procs        │  ~160 MB, 8 cores, happy
                   │  ✓ HEALTHY       │
                   └──────────────────┘
```

The insight: **client connections are cheap; server connections are not.** PgBouncer's whole job is to be the place where the cheap ones become the expensive ones — under a hard cap you control.

---

## Real world examples

### 1. HikariCP and the "about pool sizing" study

HikariCP is the default JDBC connection pool in Spring Boot, and its documentation contains the most-cited pool-sizing analysis in the industry. It references a benchmark where a single Oracle instance was tested with a pool of **2048 connections** (~7,900 TPS, with a ~100 ms wait time per request) and then with a pool of **just 96 connections** — which produced **the same throughput with roughly 1/50th the wait time**. The formula `connections = (core_count × 2) + effective_spindle_count` comes from that work, and it's why HikariCP's default `maximumPoolSize` is a deliberately small **10**. The lesson the industry took from it: *the pool is a throttle, not an accelerator.*

### 2. Amazon RDS Proxy — pooling as a managed AWS service

AWS shipped RDS Proxy specifically because Lambda + RDS was such a reliable way to kill a database. It sits between your functions and RDS/Aurora, holding a warm pool of real DB connections and multiplexing thousands of transient client connections onto them. It also **survives failovers**: when the primary fails over, the proxy re-points its server connections at the new primary and your app's client connections stay open — reducing failover-visible downtime substantially compared with every app instance independently reconnecting at once (which is itself a connection storm).

### 3. Heroku / Postgres plan limits — pooling forced by the pricing model

Heroku's Postgres plans have explicitly enforced connection limits (small plans historically capped at 20 connections; larger plans in the hundreds). This meant every Rails/Node app on Heroku hit the N × M trap almost immediately: a handful of dynos × a default pool size instantly exceeded 20. Heroku's documented answer was to run **PgBouncer as a buildpack** inside each dyno, and their guidance is essentially the one in this doc — size the pool from the database's limit, divide by dyno count, and use transaction-mode pooling. It's a good example of a platform constraint teaching an entire community the right architecture.

---

## Trade-offs

| Decision | Pros | Cons |
|---|---|---|
| **No pool** (connect per request) | Trivially simple; no leaks possible; no shared state | 30–50 ms latency tax per request; unbounded connections; guaranteed outage under load. Essentially never correct for a server. |
| **In-process pool** (`pg.Pool`) | Removes the handshake cost; bounds connections *per process*; zero extra infrastructure | Does **not** bound connections *globally* — multiplies by process count; useless for serverless |
| **External pooler** (PgBouncer / RDS Proxy) | Globally caps DB connections; enables serverless; smooths failover; lets each app be generous locally | One more component to run and monitor; transaction mode forbids session state (`SET`, `LISTEN`, temp tables, server-side prepared statements); adds ~0.1–1 ms hop |
| **HTTP data API** (Neon/PlanetScale/Data API) | No TCP connection at all; perfect for edge/serverless | Higher per-query latency; feature limits; vendor lock-in |

| Pool size | Pros | Cons |
|---|---|---|
| **Too small** (e.g. 2) | DB is never overloaded; excellent tail latency at the DB | Acquire queue grows; app-side latency spikes; you leave DB capacity unused |
| **"Right"** (≈ 2× DB cores, globally) | Peak throughput; DB stays responsive; queuing happens in the app where it's cheap | Requires you to actually count your processes |
| **Too big** (e.g. 500) | Feels reassuring | Context switching + lock contention + memory pressure → **lower** throughput; risks hitting `max_connections` and taking down the whole DB for everyone |

**Rule of thumb:** Size the pool from the **database's** cores, not your app's traffic. Start at `(db_cores × 2)` **total across all app processes**, divide by process count, set `connectionTimeoutMillis` to a few seconds so you fail fast, wrap every `connect()` in `try/finally`, and the moment you have more than a handful of app processes (or any serverless), put **PgBouncer in transaction mode** in front. If `waitingCount` is high, look at your slow queries before you touch `max`.

---

## Common interview questions on this topic

### Q1: "Why is opening a database connection expensive?"
**Hint:** Name the layers: TCP handshake (1 RTT), TLS handshake (~2 RTT), authentication (~2 RTT), then server-side session setup — tens of milliseconds total. Then land the killer detail: **on Postgres, each connection forks a full OS process consuming 5–10 MB of RAM**, so connections cost memory too, not just latency. Contrast with MySQL, which uses a thread per connection (much cheaper, but still not free).

### Q2: "Your DB has 8 cores. Should the pool be 200 connections so you can serve 200 concurrent users?"
**Hint:** No — and explain *why* the intuition fails. 8 cores can genuinely execute only 8 things at once; the other 192 aren't parallel, they're queued, and queuing *inside the DB* costs context switches, lock contention, and `work_mem`. Throughput goes **down** past the peak. Cite `(cores × 2) + spindles` ≈ 17. The 200 users still get served — they just queue in the pool's wait queue, which is free. The queue has to exist somewhere; put it where it's cheap.

### Q3: "You have 20 app servers each with a pool of max 50. What's wrong?"
**Hint:** 20 × 50 = **1000 connections** to one DB. That likely exceeds `max_connections` and definitely exceeds what the hardware can serve. Pool limits are *per process*, and they multiply. The fix: size backwards from the DB's total budget and divide by process count — or, more practically, put **PgBouncer in transaction mode** between them so 1000 cheap client connections multiplex onto ~20 real ones. Bonus points for mentioning Node `cluster`/PM2 workers multiply it again.

### Q4: "Requests are timing out. `pool.totalCount` is pinned at max and `waitingCount` keeps climbing — even after traffic stopped. What is it?"
**Hint:** A **connection leak**. The tell is that it doesn't recover when traffic drops — a genuine overload drains. Someone called `pool.connect()` on a path that can throw before `client.release()`. Look for `release()` outside a `finally` block. The fix: `try/finally`, or better, a `withTransaction(fn)` helper so callers never hold the client and *cannot* forget.

### Q5: "Why does PgBouncer's transaction mode break `SET` statements and prepared statements?"
**Hint:** In transaction mode, a real server connection is bound to your client only for the duration of **one transaction**, then it goes back into the pool and may be handed to a *different* client. So anything you stashed in the *session* — `SET timezone`, server-side prepared statements, advisory locks, temp tables, `LISTEN/NOTIFY` — is either lost or, worse, leaks to another tenant. Anything session-scoped needs `session` mode (which mostly defeats the purpose) or must be avoided.

---

## Practice exercise

### The Pool Sizing and Leak Hunt

**Part A — Size it (10 min, pen and paper).**
You run a Node API on Postgres. Facts:
- DB instance: **16 vCPU**, NVMe SSD storage.
- Fleet: **12 app servers**, each running **4** Node cluster workers.
- `max_connections = 200`.
- Peak traffic: **3,000 req/s**. Average query time: **4 ms**. Each request runs **2 queries**.

Answer these, showing the arithmetic:
1. What total connection count does the `(cores × 2) + spindles` heuristic suggest?
2. If you set `max: 20` in `pg.Pool`, how many total connections will hit the DB? Is that safe?
3. Using Little's Law (`concurrency = arrival_rate × service_time`), how many connections does the traffic *actually* need at peak? (Hint: 3000 req/s × 2 queries × 4 ms.) Compare with your answer to (1) — do they agree?
4. What `max` would you set per worker process, and would you add PgBouncer? Justify.

**Part B — Find the leak (20 min, code).**
Write a small Express app with `pg`, `max: 2`, and `connectionTimeoutMillis: 2000`. Add two endpoints:
- `GET /good` — uses a `withTransaction()` helper.
- `GET /bad` — calls `pool.connect()`, then throws before `release()`.

Now:
1. Hit `/good` 50 times. Log `pool.totalCount / idleCount / waitingCount` after each. It should stay stable.
2. Hit `/bad` **twice**. Then hit `/good`. Observe: it hangs for exactly 2000 ms, then throws an acquire timeout.
3. Write down the three metric values at the moment of failure — that pattern is what you will see in Grafana on the day this happens for real.

**Produce:** your Part A arithmetic, and the metric readings from Part B step 3 with a one-sentence explanation of why `/good` broke even though `/good` has no bug.

---

## Quick reference cheat sheet

- **A connection is expensive:** TCP + TLS + auth + session setup ≈ **15–50 ms**, and on Postgres it's a full **OS process using 5–10 MB RAM**.
- **A pool is a throttle, not an accelerator.** Its job is to *bound* concurrency, not increase it.
- **The knobs:** `min`, `max`, `idleTimeout`, `connectionTimeout` (acquire), `maxLifetime` (recycle), plus health-check on borrow.
- **`max` is the only one that really matters.** Everything else is tuning.
- **Bigger is NOT better.** Past ~2× DB cores, throughput *falls* — context switching, lock contention, `work_mem` pressure.
- **Sizing heuristic:** `(core_count × 2) + effective_spindle_count`, using the **database's** cores. HikariCP's default is 10 for a reason.
- **Size backwards from the DB**, then divide by the number of app processes.
- **The N × M trap:** `20 servers × 50 max = 1000 connections` → your DB dies. Pool limits are per *process* and multiply.
- **The fix at scale: PgBouncer** (or RDS Proxy). Use **transaction mode** — but it forbids `SET`, temp tables, advisory locks, and server-side prepared statements.
- **Serverless is the worst case:** every concurrent Lambda opens its own connection. Always put a pooler (or an HTTP data API) in front.
- **`pool.query()` cannot leak.** Use it for single statements. `pool.connect()` **can** leak — it's only for transactions.
- **Always `try { ... } finally { client.release() }`.** Better: a `withTransaction(fn)` helper so callers never touch the client.
- **The leak signature:** `totalCount == max`, `idleCount == 0`, `waitingCount` climbing — and it **does not recover when traffic stops**.
- **Set an acquire timeout** (a few seconds). Failing fast under overload sheds load; waiting forever amplifies the storm.
- **Monitor three gauges:** `totalCount`, `idleCount`, `waitingCount` — plus a histogram of acquire wait time. Alert on acquire timeouts.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [64 — Database Sharding](./64-database-sharding.md) — once you shard, you need a pool *per shard*, and the N × M math gets even sharper |
| **Next** | [66 — Database Transactions and Isolation Levels](./66-database-transactions-and-isolation.md) — transactions are exactly why you must borrow a connection explicitly (and exactly where leaks happen) |
| **Related** | [63 — Database Replication](./63-database-replication.md) — read replicas usually mean a second pool, and a pooler is what makes failover survivable |
| **Related** | [06 — Latency and Throughput](./06-latency-and-throughput.md) — the pool-size curve is Little's Law and queuing theory in disguise |
