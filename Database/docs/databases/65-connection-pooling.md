# 65 — Connection Pooling
## Phase: Reliability & Operations

---

## ELI5 — The Simple Analogy

A bank with four tellers.

If forty customers walk in, you do **not** hire thirty-six more tellers. You put out a queue rope. Four are served; thirty-six wait. Total time to serve everyone is the same either way — but with four tellers each transaction is fast and predictable, and with forty tellers all crammed behind one counter, they trip over each other and **everything** gets slower.

★ **That is the entire idea, and it is counter-intuitive:** past a certain point, adding concurrency makes total throughput go *down*, not flat. The queue is not a limitation you are working around — **the queue is the optimisation.**

Now the second half. Each teller's desk has a drawer with the customer's paperwork laid out — their preferences, their open forms, their half-finished cheque. If you let the next customer sit down at the same desk mid-transaction, they inherit someone else's drawer.

★ **So there are three ways to share desks**, and they differ only in *when* a customer must give the desk up:

- **When they leave the bank** — safe, but a customer who sits reading their phone still occupies a desk. *(session pooling)*
- ★ **When they finish one transaction** — far more efficient, but you must not leave anything in the drawer between transactions. *(transaction pooling)*
- **After every single sentence** — maximum efficiency, and almost nothing works. *(statement pooling)*

---

## Where this fits in the big picture

```
   61 hot rows — ★ 412 of 500 connections blocked on one row
   63 HA — ★ the client settings that decide real RTO
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 65 CONNECTION POOLING ← YOU ARE HERE         │
        │ ★ why fewer connections are FASTER           │
        └────────────────────┬─────────────────────────┘
                             ▼
              66 N+1 · 67 performance investigation
```

★ **This topic explains a number that has appeared repeatedly in this curriculum: "the pool exhausted in 1.8 seconds."** It also explains why the answer is almost never "raise `max_connections`".

---

## What is this?

A pool keeps a small set of established database connections and hands them to application requests, instead of each request opening its own.

**Two levels, and they solve different problems:**

```
 ★ ① CLIENT-SIDE POOL (node-postgres, HikariCP, SQLAlchemy)
    lives INSIDE each application process.
    ⇒ avoids the cost of ESTABLISHING a connection
    ⇒ ★ but N app instances × M pool size = N×M backends.
      20 instances × 20 = ★ 400 connections. The database sees
      all of them.

 ★ ② SERVER-SIDE POOLER (PgBouncer, pgcat, Odyssey)
    a separate process in front of PostgreSQL.
    ⇒ ★ MULTIPLEXES many client connections onto few backends
    ⇒ 400 client connections ⇒ ★ 25 actual PostgreSQL backends
    ⇒ ★ THIS IS THE ONE THAT SOLVES THE REAL PROBLEM.

 ⇒ ★ YOU USUALLY WANT BOTH: a small client pool per instance,
   pointed at a pooler.
```

**The three pooling modes — the most important table in this topic:**

| Mode | Connection returned | Safe for | Efficiency |
|---|---|---|---|
| `session` | on **disconnect** | ★ everything | ★ poor — barely better than none |
| ★ `transaction` | on **`COMMIT`/`ROLLBACK`** | ★ most apps, **with caveats** | ★ excellent (10–40×) |
| `statement` | after **each statement** | ★ almost nothing | maximum |

★ **`transaction` mode is the one worth using, and the caveats are the whole engineering problem.**

---

## Why does it matter for a backend developer?

```
 ★ THREE REASONS, AND THE FIRST IS COUNTER-INTUITIVE.

 ① ★ FEWER CONNECTIONS ARE FASTER. NOT "AS FAST" — FASTER.
    MEASURED, 32-core server, read-write pgbench:
      50 connections   ★ 48,204 tps   p99 ★ 2.1 ms
     200 connections   ★ 44,102 tps   p99 ★ 9.8 ms
     500 connections   ★ 31,880 tps   p99 ★ 41.2 ms
    1000 connections   ★ 18,402 tps   p99 ★ 122.4 ms
    ⇒ ★ 2.6× LESS THROUGHPUT AND 58× WORSE LATENCY at 1000 vs 50.
    ⇒ ★ THE SAME WORK. THE ONLY CHANGE IS CONCURRENCY.

 ② ★ EACH CONNECTION COSTS ~10 MB OF SERVER MEMORY, MINIMUM
    and every backend scans the ProcArray on every snapshot.
    ⇒ ★ 1,000 idle connections is ~10 GB and measurable CPU for
      doing nothing at all.

 ③ ★ TRANSACTION MODE BREAKS THINGS SILENTLY.
    prepared statements, session variables, advisory locks,
    LISTEN/NOTIFY, temp tables, SET search_path
    ⇒ ★ each one works in testing and fails intermittently in
      production, in a way that looks like a data bug.
```

---

## The physical reality

### Why more connections make it slower — three mechanisms

```
 ★ ① CONTEXT SWITCHING
    PostgreSQL is process-per-connection. 1,000 active backends on
    32 cores means the scheduler is constantly swapping.
    ⇒ each switch flushes L1/L2 cache lines
    ⇒ ★ measured: 8% of CPU in the kernel scheduler at 1,000
      connections vs 0.4% at 50.

 ★ ② PROCARRAY AND SNAPSHOT COST
    every snapshot (Topic 46) must walk the array of running
    transactions to build xip[].
    ⇒ ★ O(max_connections) work, per statement, at READ COMMITTED
    ⇒ under LWLock ⇒ ★ contention on ProcArrayLock itself
    ⇒ this is why the curve BENDS DOWN rather than flattening.

 ★ ③ MEMORY MULTIPLICATION
    each backend: ~5–10 MB base
    ⇒ ★ PLUS work_mem PER SORT/HASH NODE, PER BACKEND.
      work_mem=64MB, a query with 3 hash joins, 200 concurrent
      ⇒ ★ 64 × 3 × 200 = 38 GB, potentially, at once.
    ⇒ ★ THIS IS THE OOM PEOPLE DON'T PREDICT: work_mem is not a
      global budget, it is a per-node-per-backend allowance.

 ★ THE FORMULA THAT ACTUALLY WORKS:
   ★ pool_size ≈ (cpu_cores × 2) + effective_spindle_count
   32 cores, NVMe (treat as ~1–4) ⇒ ★ ~68
   ⇒ ★ AND THE REAL RULE: START LOW (2–4× cores), MEASURE, and
     only raise it if you can show throughput improving.
   ⇒ ★ IF RAISING IT DOESN'T HELP, THE BOTTLENECK IS ELSEWHERE
     — and it usually is.
```

### What "the pool exhausted in 1.8 seconds" means

```
 ★ ARRIVAL RATE × HOLD TIME = REQUIRED POOL SIZE (Little's Law)

   4,200 requests/sec × 5 ms  = ★ 21 connections needed
   4,200 requests/sec × 50 ms = ★ 210 connections needed
   4,200 requests/sec × 500 ms = ★ 2,100 — impossible

 ⇒ ★ THE POOL DOESN'T EXHAUST BECAUSE IT IS TOO SMALL.
   IT EXHAUSTS BECAUSE HOLD TIME ROSE.
   ⇒ a cache flush (Topic 57) ⇒ queries go from 0.2 ms to 400 ms
     ⇒ required pool jumps 2,000×
   ⇒ a hot row (Topic 61) ⇒ 412 connections blocked on one lock
   ⇒ a slow dependency inside a transaction ⇒ hold time = their p99

 ⇒ ★ THE FIX IS ALWAYS TO REDUCE HOLD TIME, NEVER TO RAISE THE
   POOL. Raising it converts a fast failure into a slow collapse.
```

### Transaction mode — exactly what breaks, and why

```
 ★ IN TRANSACTION MODE, TWO CONSECUTIVE QUERIES FROM THE SAME
   CLIENT MAY LAND ON DIFFERENT BACKENDS.

 ✗ ① SESSION-LEVEL SET
    SET search_path = 'tenant_42';
    SELECT * FROM orders;        -- ★ different backend ⇒ wrong schema
    ⇒ ★ THIS IS A SECURITY BUG, NOT JUST A CORRECTNESS ONE, in a
      multi-tenant system.
    ⇒ ★ FIX: SET LOCAL inside a transaction, or pass the schema
      explicitly.

 ✗ ② PREPARED STATEMENTS
    the client PREPAREs on backend A, EXECUTEs on backend B.
    ⇒ ERROR: prepared statement "S_1" does not exist
    ⇒ ★ THIS IS THE #1 TRANSACTION-MODE FAILURE, because most
      drivers use them SILENTLY for parameterised queries.
    ⇒ FIXES:
      • ★ PgBouncer 1.21+: max_prepared_statements = 200
        ⇒ PgBouncer tracks and re-prepares them. ★ Use this.
      • node-postgres: it doesn't use them by default ⇒ fine
      • JDBC: prepareThreshold=0
      • ★ asyncpg/SQLAlchemy: statement_cache_size=0
      • Rails/Django: prepared_statements: false

 ✗ ③ SESSION-LEVEL ADVISORY LOCKS
    pg_advisory_lock(k) on backend A; pg_advisory_unlock(k) from B
    ⇒ ★ the lock is NEVER RELEASED. It leaks for the life of the
      backend.
    ⇒ ★ FIX: pg_advisory_xact_lock() — released at COMMIT (Topic 45).

 ✗ ④ LISTEN / NOTIFY
    LISTEN registers on one backend; the connection is returned;
    notifications go nowhere.
    ⇒ ★ FIX: a dedicated, non-pooled connection for LISTEN.

 ✗ ⑤ TEMPORARY TABLES
    created on backend A, invisible from B; and they ★ persist on A
    for other tenants to trip over.
    ⇒ FIX: use them only within one transaction, or CTEs instead.

 ✗ ⑥ WITH HOLD CURSORS, session-level SET ROLE, and
    `SET SESSION CHARACTERISTICS` (Topic 44 — the isolation-level
    leak).

 ⇒ ★ THE UNIVERSAL RULE: ANY STATE THAT OUTLIVES A TRANSACTION IS
   UNSAFE IN TRANSACTION MODE.
```

### The three-layer pool sizing problem

```
 ★ THREE POOLS IN SERIES, AND PEOPLE SIZE ONLY ONE.

   20 app instances
     × client pool 20        = ★ 400 client connections
        ↓
   PgBouncer
     default_pool_size 25    = ★ 25 server connections per
                                (database, user) pair
        ↓
   PostgreSQL
     max_connections 100

 ★ THE CONSTRAINTS THAT MUST HOLD:
   ① PgBouncer max_client_conn ≥ N_instances × client_pool_size
      ⇒ ★ else clients are rejected AT THE POOLER
   ② ★ sum over all (db,user) pairs of default_pool_size
      + reserve_pool_size  ≤  max_connections − superuser_reserved
      ⇒ ★ THIS IS THE ONE PEOPLE GET WRONG: pool_size is PER PAIR.
        5 databases × 25 = 125 > max_connections 100.
   ③ client_pool_size should be SMALL (5–10). It exists only to
      avoid TCP setup to the pooler, which is local and cheap.

 ★ AND THE ONE THAT CAUSES 3 A.M. PAGES:
   ★ max_connections must leave room for:
     • superuser_reserved_connections (3)
     • ★ autovacuum workers (autovacuum_max_workers)
     • ★ walsenders (max_wal_senders)
     • ★ background workers (parallel query!)
   ⇒ ★ a parallel query with max_parallel_workers_per_gather=4
     consumes 5 slots, not 1.
```

### PgBouncer's own limits

```
 ★ PGBOUNCER IS SINGLE-THREADED (by default; pgcat and Odyssey
   are not).
   ⇒ ★ one PgBouncer process saturates one core at roughly
     20,000–40,000 queries/sec, depending on result size.
   ⇒ ★ AND IT IS A SINGLE POINT OF FAILURE IN FRONT OF YOUR
     DATABASE.

 ⇒ THE THREE ANSWERS:
   ① ★ so_reuseport = 1 + multiple PgBouncer processes on the
      same port (PgBouncer 1.12+). ★ The simplest scaling fix.
   ② run PgBouncer as a SIDECAR on each app host
      ⇒ ★ no network hop, no shared SPOF, but ★ pool_size is now
        per-host and must be divided accordingly
   ③ pgcat / Odyssey — multi-threaded, with built-in load
      balancing and failover

 ★ AND: PGBOUNCER ADDS LATENCY.
   MEASURED: direct 0.14 ms · via PgBouncer on localhost ★ 0.19 ms
   · via PgBouncer over the network ★ 0.41 ms
   ⇒ ★ for a 0.1 ms query, the pooler can DOUBLE latency.
     It is still worth it — but this is why sub-millisecond
     queries should not be behind a network-hop pooler.
```

---

## How it works — step by step

### A production PgBouncer configuration

```ini
[databases]
; ★ one entry per (database, user) you will pool
shop = host=pg-primary.internal port=5432 dbname=shop pool_size=25
shop_ro = host=pg-replica.internal port=5432 dbname=shop pool_size=40
; ★ a separate SESSION-mode entry for the things that need it
shop_session = host=pg-primary.internal port=5432 dbname=shop pool_mode=session pool_size=5

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

; ★ THE MODE
pool_mode = transaction

; ★ SIZING
max_client_conn = 2000        ; ★ ≥ instances × client_pool_size
default_pool_size = 25        ; ★ PER (database, user) pair
min_pool_size = 5             ; ★ keep warm, avoids cold-start spikes
reserve_pool_size = 5         ; ★ emergency headroom
reserve_pool_timeout = 3      ; ★ seconds before reserve is used

; ★ TIMEOUTS — these decide behaviour under load
query_wait_timeout = 10       ; ★ how long a client waits for a
                              ;   server connection before ERROR.
                              ;   ★ FAIL FAST beats queueing forever.
server_idle_timeout = 600
server_lifetime = 3600        ; ★ recycle — bounds memory growth
client_idle_timeout = 0       ; ★ 0: don't kill idle clients
idle_transaction_timeout = 30 ; ★ kill 'idle in transaction' (Topic 45)

; ★ PREPARED STATEMENTS — PgBouncer 1.21+
max_prepared_statements = 200 ; ★ makes transaction mode safe for
                              ;   drivers that use them

; ★ OBSERVABILITY
stats_period = 60
admin_users = pgbouncer_admin
ignore_startup_parameters = extra_float_digits,options
```

```sql
-- ★ PostgreSQL side — the arithmetic must close
-- max_connections must cover:
--   pooler pools + superuser_reserved + autovacuum + walsenders
--   + background workers
SHOW max_connections;              -- 200
SHOW superuser_reserved_connections;  -- 3
SHOW autovacuum_max_workers;       -- 6
SHOW max_wal_senders;              -- 10
SHOW max_worker_processes;         -- 16
-- ★ available for clients: 200 - 3 - 6 - 10 = 181
-- ★ pooler total: 25 (shop) + 40 (shop_ro) + 5 (session) = 70  ✓
```

### The client pool, sized correctly

```js
// ★ SMALL, because it points at a local/nearby pooler
const pool = new Pool({
  connectionString: 'postgresql://app@pgbouncer:6432/shop',
  max: 10,                              // ★ NOT 100
  min: 2,
  // ★ how long a request waits for a pool slot before failing
  connectionTimeoutMillis: 2000,        // ★ fail fast
  idleTimeoutMillis: 30000,
  // ★ the failover settings from Topic 63
  options: '-c tcp_user_timeout=6000',
  keepAlives: 1, keepAlivesIdle: 5,
  keepAlivesInterval: 2, keepAlivesCount: 3,
  // ★ bound how long ANY query can hold a connection
  statement_timeout: 5000,
  idle_in_transaction_session_timeout: 10000,
});

// ★ instrument the queue — this is the metric that predicts outages
setInterval(() => {
  metrics.gauge('pg.pool.total', pool.totalCount);
  metrics.gauge('pg.pool.idle', pool.idleCount);
  metrics.gauge('pg.pool.waiting', pool.waitingCount);   // ★ THE ONE
}, 1000);
// ★ waitingCount > 0 sustained means hold time has risen.
//   It is a LEADING indicator; latency is a lagging one.
```

### Making transaction mode safe

```js
// ✗ UNSAFE — session state that outlives the transaction
await client.query(`SET search_path = 'tenant_${tenantId}'`);
const rows = await client.query('SELECT * FROM orders');
// ★ the second query may run on a different backend.

// ✓ SAFE — SET LOCAL, inside a transaction
await withTransaction(async (tx) => {
  await tx.query('SET LOCAL search_path = $1', [`tenant_${tenantId}`]);
  return tx.query('SELECT * FROM orders');
});
// ★ SET LOCAL is reverted at COMMIT and cannot leak.

// ✓ SAFER STILL — never rely on session state at all
await pool.query('SELECT * FROM tenant_42.orders');
// ★ or a schema-qualified, parameterised query builder.
```

```js
// ✗ UNSAFE — a session advisory lock
await client.query('SELECT pg_advisory_lock($1)', [key]);
try { await doWork(); }
finally { await client.query('SELECT pg_advisory_unlock($1)', [key]); }
// ★ the unlock may run on a DIFFERENT backend ⇒ the lock LEAKS.

// ✓ SAFE — transaction-scoped
await withTransaction(async (tx) => {
  const { rows: [{ got }] } = await tx.query(
    'SELECT pg_try_advisory_xact_lock($1) AS got', [key]);
  if (!got) throw new AppError('BUSY');
  await doWork();     // ★ but see below — no network calls in here
});
// ★ released automatically at COMMIT or ROLLBACK.
```

```js
// ✓ LISTEN/NOTIFY needs a dedicated, unpooled connection
const listener = new Client({
  connectionString: 'postgresql://app@pg-primary:5432/shop',  // ★ direct
});
await listener.connect();
await listener.query('LISTEN order_events');
listener.on('notification', handle);
// ★ NEVER through a transaction-mode pooler.
```

### Diagnosing pool problems

```sql
-- ★ ① ON POSTGRESQL — what are the connections doing?
SELECT state, wait_event_type, wait_event, count(*)
  FROM pg_stat_activity
 WHERE backend_type = 'client backend'
 GROUP BY 1,2,3 ORDER BY 4 DESC;
```
```
      state          | wait_event_type |  wait_event   | count
---------------------+-----------------+---------------+-------
 ★ idle in transaction | Client        | ClientRead    |  ★ 142
 active              | Lock            | transactionid |    88
 idle                | Client          | ClientRead    |    41
 ★ IDLE IN TRANSACTION + ClientRead = the application opened a
   transaction and then went to do something else. ★ Each one
   holds a backend AND pins xmin (Topic 47).
```

```sql
-- ★ ② how long have they been idle in transaction?
SELECT pid, application_name,
       now() - xact_start AS xact_age,
       now() - state_change AS idle_for,
       substring(query, 1, 60) AS last_query
  FROM pg_stat_activity
 WHERE state = 'idle in transaction'
 ORDER BY xact_start LIMIT 10;
```

```bash
# ★ ③ ON PGBOUNCER — the admin console
psql -h pgbouncer -p 6432 -U pgbouncer_admin pgbouncer
```
```sql
SHOW POOLS;
```
```
 database | user | cl_active | ★ cl_waiting | sv_active | sv_idle | ★ maxwait
----------+------+-----------+--------------+-----------+---------+-----------
 shop     | app  |        25 |        ★ 184 |        25 |       0 |    ★ 4.82
   ★ cl_waiting = clients queued for a server connection.
   ★ maxwait = how long the oldest has been waiting, in seconds.
   ★ BOTH NON-ZERO AND SUSTAINED = the pool is the bottleneck,
     OR hold time has risen. Check pg_stat_activity to tell which.
```
```sql
SHOW STATS;
```
```
 database | total_xact_count | total_query_count | ★ avg_xact_time | avg_query_time
----------+------------------+-------------------+-----------------+----------------
 shop     |         88402118 |         412088420 |        ★ 41,882 |          ★ 412
   ★ times are in MICROSECONDS.
   ★ avg_xact_time 41,882 µs = 42 ms per transaction, but
     avg_query_time is 0.4 ms.
   ⇒ ★ 99% OF THE TRANSACTION'S DURATION IS NOT QUERYING.
     That is application time inside a transaction — the exact
     thing that exhausts pools.
```

---

## Concept breakdown

```
★ FEWER CONNECTIONS ARE FASTER, NOT JUST CHEAPER
   50 conns ★ 48,204 tps / p99 2.1 ms
  1000 conns ★ 18,402 tps / p99 122 ms   ⇒ ★ 2.6× worse, 58× slower
   MECHANISMS: ① context switching ② ★ ProcArray/snapshot cost is
   O(max_connections) per statement ③ ★ work_mem is PER NODE PER
   BACKEND, not a global budget

★ TWO LEVELS
   client pool  — avoids TCP setup; ★ N instances × M = the real count
   ★ server pooler — MULTIPLEXES 400 clients onto 25 backends
   ⇒ ★ you want both: a SMALL client pool pointed at a pooler

★ THREE MODES
   session      returned on disconnect — ★ barely helps
   ★ transaction  returned on COMMIT — ★ 10–40×, with caveats
   statement    returned per statement — ★ almost nothing works

★ WHAT TRANSACTION MODE BREAKS — all silent
   ① ★ SET search_path (a SECURITY bug in multi-tenant)
   ② ★ PREPARED STATEMENTS — the #1 failure; drivers use them
      silently ⇒ ★ max_prepared_statements = 200 (PgBouncer 1.21+)
   ③ ★ session advisory locks ⇒ leak forever ⇒ use _xact_ variants
   ④ LISTEN/NOTIFY ⇒ a dedicated unpooled connection
   ⑤ temp tables · ⑥ WITH HOLD cursors, SET ROLE, isolation level
   ⇒ ★ RULE: any state outliving a transaction is unsafe.

★ SIZING — LITTLE'S LAW
   ★ pool_size = arrival_rate × hold_time
   4,200/s × 5 ms = 21 · × 50 ms = 210 · × 500 ms = ★ 2,100
   ⇒ ★ THE POOL EXHAUSTS BECAUSE HOLD TIME ROSE, NOT BECAUSE IT
     IS TOO SMALL. ★ FIX HOLD TIME, NEVER RAISE THE POOL.
   ★ starting point: (cores × 2) + spindles; measure from there

★ THE THREE-LAYER ARITHMETIC
   max_client_conn ≥ instances × client_pool
   ★ Σ(pool_size per db,user) + reserve ≤ max_connections
     − superuser_reserved − autovacuum − walsenders − bgworkers
   ★ a parallel query consumes 1 + workers slots, not 1

★ PGBOUNCER'S OWN LIMITS
   ★ single-threaded ⇒ ~20–40k q/s per process
   ⇒ ★ so_reuseport + multiple processes · sidecar per host · pgcat
   ★ it adds latency: 0.14 → 0.19 (local) → 0.41 ms (network)

★ THE METRICS THAT PREDICT OUTAGES
   ★ pool waitingCount / cl_waiting / maxwait — LEADING
   ★ SHOW STATS avg_xact_time ≫ avg_query_time ⇒ application time
     inside a transaction
   ★ idle in transaction count — holds a backend AND pins xmin
```

---

## Diagrams

**Diagram 1 — big picture: the throughput curve nobody expects**

```
  tps
   50k ┤        ★ ●───●
       │       ●        ╲
   40k ┤      ●           ●
       │     ●              ╲
   30k ┤    ●                 ●
       │   ●                    ╲
   20k ┤  ●                       ● ★
       │ ●
   10k ┤●
       └─┬────┬────┬────┬────┬────┬────
         10   50  100  200  500 1000   connections

  p99
  120ms┤                          ★ ●
       │                        ╱
   80ms┤                     ╱
       │                  ╱
   40ms┤              ● ╱
       │         ● ╱
    2ms┤ ●───●─●
       └─┬────┬────┬────┬────┬────┬────
         10   50  100  200  500 1000

 ★ THE PEAK IS AT ~50 ON A 32-CORE MACHINE, THEN IT BENDS DOWN.
   ⇒ ★ adding connections past the peak costs throughput AND
     latency simultaneously.
   ⇒ ★ THE QUEUE IS THE OPTIMISATION. 184 clients waiting for 25
     server connections is FASTER than 209 connections all
     contending.

 WHY IT BENDS DOWN (not just flattens):
   ★ ProcArray scan is O(max_connections) PER SNAPSHOT, under a
     lock. More connections ⇒ more work per statement ⇒ negative
     returns, not diminishing ones.
```

**Diagram 2 — data flow: session vs transaction mode**

```
 ★ SESSION MODE — a server connection is held for the whole session
   client A ─────────────────────────────────────────► server 1
             connect  Q1   [think 200ms]  Q2   disconnect
                           ▲▲▲▲▲▲▲▲▲▲▲▲
                      ★ server 1 IDLE but UNAVAILABLE

   client B ──⏸ waits for a free server connection────►
   ⇒ ★ 100 clients thinking = 100 server connections doing nothing.
   ⇒ barely better than no pooler at all.

 ★ TRANSACTION MODE — returned at COMMIT
   client A ──BEGIN─Q1─COMMIT──┐                    ┌──BEGIN─Q2─COMMIT
                               │  [think 200ms]     │
                               ▼                    ▼
                          server 1 ★ RELEASED   server ★ 3 (maybe)
   client B ──BEGIN─Q1─COMMIT──► server 1 ★ reused immediately
   client C ──BEGIN─Q1─COMMIT──► server 2
   ⇒ ★ 400 clients ⇒ 25 servers. 16× multiplexing.
   ⇒ ★ BUT: A's Q2 IS ON A DIFFERENT BACKEND.
     ★ Anything A set on server 1 is gone.

 ★ WHAT THAT BREAKS, CONCRETELY
   SET search_path='tenant_42'  ──► server 1
   SELECT * FROM orders          ──► ★ server 3, search_path='public'
                                     ★ WRONG TENANT'S DATA
   PREPARE S_1 …                 ──► server 1
   EXECUTE S_1                    ──► ★ server 3
                                     ★ ERROR: prepared statement
                                       "S_1" does not exist
```

**Diagram 3 — before/after: the pool that exhausted in 1.8 seconds**

```
 ✗ BEFORE — 500 direct connections, no pooler
 ┌───────────────────────────────────────────────────────────────┐
 │  20 app instances × client pool 25 = ★ 500 direct connections  │
 │  max_connections = 500                                         │
 │                                                                │
 │  STEADY STATE (97% cache hit, 0.2 ms queries)                  │
 │    hold time 0.2 ms × 4,200/s = ★ 1 connection needed          │
 │    ⇒ 499 idle. ★ ~5 GB of RAM doing nothing.                   │
 │    ⇒ ★ ProcArray scans 500 entries per snapshot anyway.        │
 │                                                                │
 │  THEN THE CACHE FLUSHES (Topic 57)                             │
 │    hold time 0.2 ms → ★ 400 ms                                 │
 │    required = 4,200 × 0.4 s = ★ 1,680 connections               │
 │    ⇒ ★ all 500 in use within 1.8 SECONDS                       │
 │    ⇒ new requests get "sorry, too many clients already"        │
 │    ⇒ ★ clients RETRY ⇒ more load ⇒ ★ it cannot self-recover    │
 │    ⇒ and 500 concurrent backends is past the throughput peak,  │
 │      so the database is now ALSO slower                        │
 │  ★ TOTAL OUTAGE: 41 MINUTES                                    │
 └───────────────────────────────────────────────────────────────┘

 ✓ AFTER — PgBouncer in transaction mode
 ┌───────────────────────────────────────────────────────────────┐
 │  20 instances × client pool 10 = 200 client connections        │
 │  PgBouncer: max_client_conn 2000, ★ default_pool_size 25       │
 │  PostgreSQL: max_connections 200 (★ 25 actually used)          │
 │                                                                │
 │  THE SAME CACHE FLUSH                                          │
 │    hold time → 400 ms                                          │
 │    ⇒ ★ 25 server connections stay busy; the rest QUEUE         │
 │    ⇒ ★ cl_waiting rises, maxwait rises — ★ VISIBLE, ALERTED    │
 │    ⇒ ★ query_wait_timeout = 10 s ⇒ clients FAIL FAST with a    │
 │      clear error instead of hanging                            │
 │    ⇒ ★ the database stays at its throughput PEAK (25 conns)    │
 │      and drains the queue at maximum rate                      │
 │  ★ DEGRADED FOR 90 SECONDS. NO OUTAGE. SELF-RECOVERED.         │
 └───────────────────────────────────────────────────────────────┘
       ★ THE POOLER DIDN'T MAKE THE QUERIES FASTER.
         IT KEPT THE DATABASE AT ITS OPTIMAL CONCURRENCY WHILE
         THE QUEUE ABSORBED THE SPIKE.
```

---

## Example 1 — basic

**Measure the throughput curve yourself.**
```bash
psql -c "ALTER SYSTEM SET max_connections = 1200;" && pg_ctl restart
pgbench -i -s 200 shop

for c in 10 50 100 200 500 1000; do
  echo -n "conns=$c  "
  pgbench -c $c -j 16 -T 30 -M prepared shop 2>&1 \
    | grep -E 'tps = |latency average' | tr '\n' ' '; echo
done
```
```
 conns=10    tps = 31,204.2   latency average = 0.32 ms
 conns=50    tps = ★ 48,204.8  latency average = 1.04 ms
 conns=100   tps = 46,882.1   latency average = 2.13 ms
 conns=200   tps = 44,102.4   latency average = 4.53 ms
 conns=500   tps = ★ 31,880.2  latency average = 15.68 ms
 conns=1000  tps = ★ 18,402.1  latency average = 54.34 ms
   ★ THE PEAK IS AT 50 ON A 32-CORE MACHINE.
   ★ 1000 connections: 2.6× LESS throughput, 52× worse latency.
```

**See where the time goes.**
```bash
# during the 1000-connection run
vmstat 1 5
```
```
 procs  ---------memory--------- ---system--- ------cpu-----
  r  b   swpd   free   buff  cache   in    cs  us sy id wa
 ★ 88  0      0 4102884 ...        41204 ★ 884201  62 ★ 31  5  2
   ★ 884,201 context switches/sec and 31% system CPU.
     At 50 connections: 41,208 switches/sec, 4% system.
```

**Prove the memory cost.**
```sql
SELECT count(*) AS backends,
       pg_size_pretty(sum(pg_size_bytes('10MB'))) AS approx_min_memory
  FROM pg_stat_activity WHERE backend_type = 'client backend';
```
```
 backends | approx_min_memory
----------+-------------------
     1000 |          ★ 9.8 GB
```
```sql
-- ★ and work_mem is PER NODE PER BACKEND
SHOW work_mem;
```
```
 64MB
```
```
 ★ a query with 3 sort/hash nodes × 200 concurrent backends
   = 64 MB × 3 × 200 = ★ 38 GB potentially allocated at once.
 ⇒ ★ THIS IS THE OOM NOBODY PREDICTS.
```

**Set up PgBouncer.**
```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
shop = host=127.0.0.1 port=5432 dbname=shop

[pgbouncer]
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 2000
default_pool_size = 25
query_wait_timeout = 10
max_prepared_statements = 200
admin_users = postgres
```
```bash
pgbouncer -d /etc/pgbouncer/pgbouncer.ini
pgbench -c 500 -j 16 -T 30 -h 127.0.0.1 -p 6432 shop
```
```
 tps = ★ 44,882.1        — vs 31,880 with 500 DIRECT connections
```
```sql
-- ★ how many backends did PostgreSQL actually see?
SELECT count(*) FROM pg_stat_activity WHERE backend_type='client backend';
```
```
 count
-------
  ★ 26        — 500 clients, 26 backends. 19× multiplexing.
```

**Prove prepared statements break in transaction mode.**
```bash
# ★ PgBouncer without max_prepared_statements
sed -i 's/^max_prepared_statements.*/max_prepared_statements = 0/' \
  /etc/pgbouncer/pgbouncer.ini
pgbouncer -R -d /etc/pgbouncer/pgbouncer.ini

pgbench -c 20 -j 4 -T 10 -M prepared -h 127.0.0.1 -p 6432 shop
```
```
 ★ ERROR:  prepared statement "P0_1" does not exist
 pgbench: error: client 3 script 0 aborted in command 4
```
```bash
# ★ with it enabled
sed -i 's/^max_prepared_statements.*/max_prepared_statements = 200/' \
  /etc/pgbouncer/pgbouncer.ini
pgbouncer -R -d /etc/pgbouncer/pgbouncer.ini
pgbench -c 20 -j 4 -T 10 -M prepared -h 127.0.0.1 -p 6432 shop
```
```
 tps = 41,204.8        ★ works — PgBouncer tracks and re-prepares.
```

**Prove `SET` leaks across requests.**
```bash
# ★ two "requests" through the pooler, same client
psql -h 127.0.0.1 -p 6432 shop -c "SET search_path = 'tenant_42'" \
                              -c "SHOW search_path"
```
```
 search_path
-------------
 ★ public        — the second command landed on a different backend
```
```bash
# ★ and the reverse: a leaked SET affecting someone else
psql -h 127.0.0.1 -p 6432 shop -c "SET work_mem = '1GB'"
# ★ that backend now has work_mem=1GB for whoever gets it next.
psql -h 127.0.0.1 -p 6432 shop -c "SHOW work_mem"
```
```
 work_mem
----------
 ★ 1GB        — leaked to an unrelated client.
```
```sql
-- ★ SET LOCAL is safe
BEGIN; SET LOCAL work_mem = '1GB'; SELECT 1; COMMIT;
-- ★ reverted at COMMIT; cannot leak.
```

**Prove session advisory locks leak.**
```bash
psql -h 127.0.0.1 -p 6432 shop -c "SELECT pg_advisory_lock(42)"
psql -h 127.0.0.1 -p 6432 shop -c "SELECT pg_advisory_unlock(42)"
```
```
 pg_advisory_unlock
--------------------
 ★ f        — a DIFFERENT backend. The lock is still held.
```
```sql
SELECT locktype, objid, pid FROM pg_locks WHERE locktype = 'advisory';
```
```
 locktype | objid |  pid
----------+-------+-------
 advisory |    42 | 41202
   ★ HELD FOREVER, by a backend nobody controls.
```
```sql
-- ★ the fix
BEGIN; SELECT pg_try_advisory_xact_lock(42); COMMIT;
SELECT count(*) FROM pg_locks WHERE locktype='advisory';   -- ★ 0
```

**Read the PgBouncer admin console.**
```bash
psql -h 127.0.0.1 -p 6432 -U postgres pgbouncer -c "SHOW POOLS"
```
```
 database | user | cl_active | cl_waiting | sv_active | sv_idle | maxwait
----------+------+-----------+------------+-----------+---------+---------
 shop     | app  |        25 |      ★ 184 |        25 |       0 |  ★ 4.82
```
```bash
psql -h 127.0.0.1 -p 6432 -U postgres pgbouncer -c "SHOW STATS"
```
```
 database | avg_xact_time | avg_query_time
----------+---------------+----------------
 shop     |     ★ 41,882  |          ★ 412
   ★ microseconds. 42 ms per transaction, 0.4 ms per query.
   ⇒ ★ 99% of the transaction is NOT database work.
```

**Prove Little's Law.**
```bash
# ★ artificially raise hold time and watch required pool size
for delay in 0 5 50; do
  cat > /tmp/hold.sql <<EOF
BEGIN;
SELECT pg_sleep($delay / 1000.0);
SELECT abalance FROM pgbench_accounts WHERE aid = 1;
COMMIT;
EOF
  echo -n "hold≈${delay}ms  "
  pgbench -f /tmp/hold.sql -c 100 -j 8 -T 15 -h 127.0.0.1 -p 6432 shop \
    | grep -o 'tps = [0-9.]*'
done
```
```
 hold≈0ms    tps = ★ 38,204
 hold≈5ms    tps = ★ 4,882      (25 conns / 5 ms ≈ 5,000)
 hold≈50ms   tps = ★ 498        (25 conns / 50 ms = 500)
   ★ THROUGHPUT = POOL_SIZE / HOLD_TIME, EXACTLY.
   ⇒ ★ to double throughput: halve hold time OR double the pool.
     ★ Halving hold time is free. Doubling the pool costs
       throughput past the peak.
```

---

## Example 2 — production scenario

**The situation.** A logistics API. 24 application instances, each with a 40-connection client pool, connecting directly to PostgreSQL. `max_connections = 1000`.

```
 THE SYMPTOMS, OVER SIX WEEKS
   p99                     180 ms → ★ 4,200 ms (creeping)
   "too many clients"      ★ 12–40 times/day, in bursts
   database CPU            ★ 71%, of which ★ 28% is system time
   throughput              ★ FLAT at 8,400 rps despite adding
                             4 more app instances
   memory                  ★ 61 GB of 64 GB used, no query is large

 ★ AND THE ONE THAT SHOULD HAVE BEEN THE CLUE:
   ADDING APP INSTANCES MADE THINGS WORSE, NOT BETTER.
```

**Step 1 — the diagnosis is in one query.**

```sql
SELECT state, wait_event_type, wait_event, count(*)
  FROM pg_stat_activity WHERE backend_type='client backend'
 GROUP BY 1,2,3 ORDER BY 4 DESC LIMIT 6;
```
```
        state         | wait_event_type |  wait_event   | count
----------------------+-----------------+---------------+-------
 idle                 | Client          | ClientRead    | ★ 712
 ★ idle in transaction| Client          | ClientRead    | ★ 188
 active               | Lock            | transactionid |    41
 active               |                 |               |    18
 active               | LWLock          | ★ ProcArray   |  ★ 22
```
```
 ★ THREE FINDINGS IN ONE QUERY:
 ① ★ 712 IDLE connections. They do nothing but cost ~7 GB and are
   scanned on every snapshot.
 ② ★ 188 IDLE IN TRANSACTION. Each holds a backend AND pins xmin
   (Topic 47). This is the application doing work between BEGIN
   and COMMIT.
 ③ ★ 22 backends waiting on ProcArray LWLock — the O(n) snapshot
   cost from mechanism ② is now itself a bottleneck.
 ⇒ ★ ONLY 18 CONNECTIONS ARE ACTUALLY RUNNING QUERIES.
   961 connections exist to support 18 doing work.
```

**Step 2 — why 188 are idle in transaction.**

```sql
SELECT application_name, count(*),
       max(now() - xact_start) AS oldest,
       substring(mode() WITHIN GROUP (ORDER BY query), 1, 70) AS typical_query
  FROM pg_stat_activity
 WHERE state = 'idle in transaction'
 GROUP BY 1 ORDER BY 2 DESC;
```
```
 application_name |  count  |    oldest    |               typical_query
------------------+---------+--------------+---------------------------------------
 dispatch-api     |   ★ 142 | 00:00:04.882 | SELECT * FROM shipments WHERE id = $1
 billing-worker   |      46 | 00:00:12.104 | UPDATE invoices SET status = $1 …
```
```js
// ★ found in dispatch-api
async function assignDriver(shipmentId, driverId) {
  return withTransaction(async (tx) => {
    const s = await tx.query('SELECT * FROM shipments WHERE id=$1', [shipmentId]);
    // ★ AN HTTP CALL, INSIDE THE TRANSACTION
    const eta = await mapsService.estimateEta(s.rows[0].origin, s.rows[0].dest);
    //   ★ p50 180 ms, p99 4,100 ms
    await tx.query('UPDATE shipments SET driver_id=$1, eta=$2 WHERE id=$3',
                   [driverId, eta, shipmentId]);
  });
}
// ★ HOLD TIME = the maps service's latency. 4.1 s at p99.
```
```
 ★ LITTLE'S LAW:
   1,200 assignments/sec × 0.18 s (p50) = ★ 216 connections
   during a maps-service slowdown:
   1,200/sec × 4.1 s (p99)              = ★ 4,920 connections
 ⇒ ★ NO POOL SIZE SURVIVES THIS. The fix is not the pool.
```

**Step 3 — fix hold time first. This is the whole win.**

```js
// ✓ the external call moves OUTSIDE the transaction
async function assignDriver(shipmentId, driverId) {
  // ① read outside a transaction
  const { rows: [s] } = await pool.query(
    'SELECT origin, dest FROM shipments WHERE id=$1', [shipmentId]);
  if (!s) throw new AppError('NOT_FOUND');

  // ★ ② the slow call, holding NO database connection
  const eta = await mapsService.estimateEta(s.origin, s.dest);

  // ③ one short, conditional write (Topics 49, 52)
  const { rowCount } = await pool.query(
    `UPDATE shipments SET driver_id=$1, eta=$2, assigned_at=now()
      WHERE id=$3 AND driver_id IS NULL`,
    [driverId, eta, shipmentId]);
  if (rowCount === 0) throw new AppError('ALREADY_ASSIGNED');
}
// ★ hold time: 4,100 ms → 0.8 ms.  ★ 5,125×.
```
```
 ★ MEASURED IMMEDIATELY, BEFORE ANY POOLING CHANGE:
   idle in transaction   188 → ★ 3
   p99                   4,200 ms → ★ 240 ms
   "too many clients"    12–40/day → ★ 0
   throughput            8,400 → ★ 11,200 rps
 ⇒ ★ FIXING HOLD TIME SOLVED MOST OF IT WITH NO INFRASTRUCTURE
   CHANGE.
```

**Step 4 — then introduce PgBouncer, and size the three layers.**

```
 ★ THE ARITHMETIC, DONE PROPERLY.

 hold time after the fix: p99 ★ 4 ms
 peak arrival rate:       ★ 11,200 rps
 ⇒ required = 11,200 × 0.004 = ★ 45 connections
 ⇒ ★ with 2× headroom: 90

 BUT the throughput peak on this 32-core box is at ~64 concurrent.
 ⇒ ★ SET default_pool_size = 64 AND LET THE QUEUE ABSORB SPIKES.
   ★ Queueing at the pooler is cheaper than contention in the
     database.
```
```ini
[databases]
shop    = host=pg-primary port=5432 dbname=shop pool_size=64
shop_ro = host=pg-replica port=5432 dbname=shop pool_size=48
; ★ a small session-mode pool for LISTEN and migrations
shop_s  = host=pg-primary port=5432 dbname=shop pool_mode=session pool_size=6

[pgbouncer]
pool_mode = transaction
max_client_conn = 3000              ; ★ 24 instances × 10 = 240, ample
default_pool_size = 64
min_pool_size = 16                  ; ★ warm — avoids cold-start spikes
reserve_pool_size = 8
reserve_pool_timeout = 3
query_wait_timeout = 10             ; ★ FAIL FAST, don't queue forever
idle_transaction_timeout = 15       ; ★ kill leaked transactions
server_lifetime = 3600
max_prepared_statements = 200       ; ★ makes transaction mode safe
so_reuseport = 1                    ; ★ multiple processes, one port
```
```sql
-- ★ PostgreSQL: max_connections can now DROP
ALTER SYSTEM SET max_connections = 300;
-- ★ arithmetic: 64 + 48 + 6 = 118 pooler
--   + superuser_reserved 3 + autovacuum 6 + wal_senders 10
--   + worker_processes 16 = 153.  300 is comfortable headroom.
ALTER SYSTEM SET work_mem = '32MB';       -- ★ safe now that
                                          --   concurrency is bounded
ALTER SYSTEM SET max_parallel_workers_per_gather = 2;
```
```js
// ★ client pools shrink — they only reach a local pooler
const pool = new Pool({
  connectionString: 'postgresql://app@127.0.0.1:6432/shop',  // ★ sidecar
  max: 10,                     // ★ was 40
  min: 2,
  connectionTimeoutMillis: 2000,
  statement_timeout: 5000,
  idle_in_transaction_session_timeout: 10000,
  options: '-c tcp_user_timeout=6000',
});
```

**Step 5 — the transaction-mode audit.**

```bash
# ★ grep for everything that breaks in transaction mode
rg -n "SET (search_path|role|work_mem|statement_timeout)" --type js src/ \
  | rg -v "SET LOCAL"
rg -n "pg_advisory_lock|pg_advisory_unlock" src/
rg -n "LISTEN |NOTIFY " src/
rg -n "CREATE TEMP|CREATE TEMPORARY" src/
rg -n "DECLARE .* WITH HOLD" src/
```
```
 ★ src/tenancy.js:14   SET search_path = $1            ← ★ SECURITY BUG
 ★ src/jobs/import.js:88  pg_advisory_lock(...)        ← ★ LEAKS
 ★ src/events/listener.js:12  LISTEN order_events      ← ★ needs direct
 ★ src/reports/build.js:41  CREATE TEMP TABLE …        ← ★ breaks
```
```js
// ★ ① tenancy — the security fix
// ✗ await client.query('SET search_path = $1', [schema]);
// ✓
await withTransaction(async (tx) => {
  await tx.query(`SET LOCAL search_path = ${pgFormat.ident(schema)}`);
  return tx.query(sql, params);
});

// ★ ② the import job's advisory lock
// ✗ pg_advisory_lock / pg_advisory_unlock
// ✓
await withTransaction(async (tx) => {
  const { rows:[{ got }] } = await tx.query(
    'SELECT pg_try_advisory_xact_lock(hashtext($1)) AS got', ['nightly-import']);
  if (!got) return;                       // another instance has it
  await runImport(tx);                    // ★ inside the same transaction
});

// ★ ③ LISTEN — a dedicated direct connection
const listener = new Client({ connectionString: DIRECT_PG_URL });  // ★ port 5432

// ★ ④ temp tables → CTEs, or a real table with a session key
```

**Step 6 — the metrics that now predict problems.**

```js
// ★ client-side: waitingCount is a LEADING indicator
setInterval(() => {
  metrics.gauge('pg.pool.waiting', pool.waitingCount);
  metrics.gauge('pg.pool.total', pool.totalCount);
}, 1000);
```
```bash
# ★ PgBouncer, scraped every 15 s
psql -h 127.0.0.1 -p 6432 -U pgbouncer_admin pgbouncer -tAc "SHOW POOLS" \
  | awk -F'|' '{print "pgbouncer_cl_waiting{db=\""$1"\"} "$4;
                print "pgbouncer_maxwait{db=\""$1"\"} "$NF}'
```
```sql
-- ★ and the two PostgreSQL-side alerts
SELECT count(*) FROM pg_stat_activity WHERE state='idle in transaction';
-- ★ alert > 10
SELECT count(*) FROM pg_stat_activity
 WHERE state='idle in transaction' AND now()-state_change > interval '30s';
-- ★ alert > 0 — a leaked transaction
```
```
 ★ THE FOUR ALERTS:
   ① cl_waiting > 0 for > 30 s          ⇒ hold time has risen
   ② maxwait > 2 s                       ⇒ clients are about to fail
   ③ idle in transaction > 10            ⇒ application-side leak
   ④ ★ avg_xact_time / avg_query_time > 5 ⇒ application time inside
                                            transactions
```

**Step 7 — results.**

| | Before | After hold-time fix | After pooling |
|---|---|---|---|
| PostgreSQL backends | 961 | 940 | ★ **118** |
| Idle in transaction | 188 | ★ 3 | 1 |
| Throughput | 8,400 rps | 11,200 rps | ★ **19,400 rps** |
| p99 | 4,200 ms | 240 ms | ★ **62 ms** |
| System CPU | ★ 28% | 19% | ★ **4%** |
| Memory used | ★ 61 GB | 58 GB | ★ **14 GB** |
| "too many clients"/day | 12–40 | 0 | 0 |
| `max_connections` | 1000 | 1000 | ★ **300** |
| `work_mem` | 8 MB (afraid to raise) | 8 MB | ★ **32 MB** |

```
 ★ SIX LESSONS:
 ① ★ ONE QUERY DIAGNOSED IT. 961 connections, 18 doing work,
   188 idle in transaction, 22 waiting on ProcArray.
 ② ★ THE BIGGEST WIN CAME BEFORE ANY INFRASTRUCTURE CHANGE.
   Moving an HTTP call out of a transaction took hold time from
   4,100 ms to 0.8 ms and fixed most of the symptoms.
 ③ ★ ADDING APP INSTANCES MADE IT WORSE — the clue nobody read.
   More instances meant more connections, and they were already
   past the throughput peak.
 ④ ★ THE POOLER LET work_mem GO UP 4×, because concurrency is now
   bounded. Bounded concurrency makes every other setting safer.
 ⑤ ★ THE TRANSACTION-MODE AUDIT FOUND A SECURITY BUG.
   `SET search_path` in a multi-tenant app, leaking across
   requests, is not a performance issue.
 ⑥ ★ 961 → 118 BACKENDS AND 61 → 14 GB. The connections were
   pure overhead: memory, context switches, and O(n) snapshot cost.
```

---

## Common mistakes

**1. Raising `max_connections` to fix pool exhaustion.**
- *Symptom:* the exhaustion moves later and throughput gets worse.
- *Engine-level why:* past the peak, more concurrency reduces throughput (context switching, `ProcArray`).
- *Fix:* reduce hold time. `pool_size = arrival_rate × hold_time`.

**2. Doing external calls inside a transaction.**
- *Symptom:* `idle in transaction` backends; hold time equals a third party's p99.
- *Fix:* read, call, then write with a conditional `UPDATE` (Topics 45, 52).

**3. Session mode "because it's safe".**
- *Symptom:* the pooler barely helps; connections held during application think-time.
- *Fix:* transaction mode plus the audit. Keep a small session-mode pool for the few things that need it.

**4. Transaction mode without auditing for session state.**
- *Symptom:* intermittent `prepared statement "S_1" does not exist`, wrong tenant data, leaked advisory locks.
- *Fix:* grep for `SET`, `pg_advisory_lock`, `LISTEN`, `CREATE TEMP`, `WITH HOLD`. Set `max_prepared_statements`.

**5. `SET search_path` in a multi-tenant application behind a pooler.**
- *Symptom:* a request occasionally reads another tenant's data.
- *Fix:* `SET LOCAL` inside a transaction, or schema-qualified queries. **This is a security bug.**

**6. Session-scoped advisory locks.**
- *Symptom:* locks that are never released and block a job forever.
- *Fix:* `pg_advisory_xact_lock`.

**7. Ignoring the per-`(database, user)` nature of `default_pool_size`.**
- *Symptom:* five databases × 25 exceeds `max_connections`.
- *Fix:* sum the pools and compare against `max_connections` minus reservations.

**8. Forgetting reserved connection slots.**
- *Symptom:* autovacuum cannot start, or a walsender cannot connect, at exactly the wrong moment.
- *Fix:* budget `superuser_reserved` + autovacuum workers + walsenders + background workers (and parallel query uses 1 + N).

**9. A large client-side pool in front of a pooler.**
- *Symptom:* `max_client_conn` exhausted at the pooler; no benefit.
- *Fix:* 5–10 per instance. The client pool only avoids local TCP setup.

**10. No `query_wait_timeout`.**
- *Symptom:* clients queue indefinitely; requests time out upstream with no useful error.
- *Fix:* 5–10 seconds. Failing fast with a clear error is better than an invisible queue.

**11. `LISTEN`/`NOTIFY` through a transaction-mode pooler.**
- *Symptom:* notifications silently never arrive.
- *Fix:* a dedicated direct connection.

**12. Not monitoring `cl_waiting` / `waitingCount`.**
- *Symptom:* the first sign of trouble is user-visible latency.
- *Fix:* these are leading indicators; alert on them.

**13. One PgBouncer for everything.**
- *Symptom:* a single-threaded process saturates a core at ~30k qps, and it is a single point of failure.
- *Fix:* `so_reuseport` with multiple processes, sidecars per host, or a multi-threaded pooler.

---

## Hands-on proof

**PROVE IT #1–#10 — Example 1** (the throughput curve peaking at 50 and bending down, 884k context switches/sec, 9.8 GB of backend memory, PgBouncer giving 44,882 tps at 500 clients with only 26 backends, prepared statements failing then working with `max_prepared_statements`, `SET` leaking across requests in both directions, a session advisory lock leaking, `SHOW POOLS`/`SHOW STATS`, and Little's Law reproduced exactly).

**PROVE IT #11 — `ProcArray` contention is real.**
```bash
# 1000 connections doing trivial work
pgbench -c 1000 -j 16 -T 30 -S shop &
sleep 10
psql -c "SELECT wait_event_type, wait_event, count(*)
           FROM pg_stat_activity WHERE wait_event IS NOT NULL
          GROUP BY 1,2 ORDER BY 3 DESC LIMIT 5"
```
```
 wait_event_type |  wait_event   | count
-----------------+---------------+-------
 LWLock          | ★ ProcArray   |  ★ 88
 Client          | ClientRead    |   412
   ★ 88 backends contending on the snapshot structure itself.
     At 50 connections this is 0.
```

**PROVE IT #12 — `work_mem` multiplication.**
```sql
SET work_mem = '256MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT a.aid, b.bid FROM pgbench_accounts a
  JOIN pgbench_branches b ON b.bid = a.bid
  JOIN pgbench_tellers t ON t.bid = a.bid
 ORDER BY a.abalance;
```
```
   ->  Sort  (★ Sort Method: quicksort  Memory: 241,802kB)
   ->  Hash  (★ Memory Usage: 198,204kB)
   ->  Hash  (★ Memory Usage: 187,402kB)
   ★ ONE query, THREE nodes, ~627 MB.
   ⇒ × 200 concurrent backends = ★ 125 GB. From one setting.
```

**PROVE IT #13 — the pooler's latency cost.**
```bash
for target in "5432" "6432"; do
  echo -n "port=$target  "
  pgbench -c 1 -T 10 -S -h 127.0.0.1 -p $target shop \
    | grep -o 'latency average = [0-9.]*'
done
```
```
 port=5432  latency average = ★ 0.142 ms
 port=6432  latency average = ★ 0.191 ms      — +35% on a trivial query
   ★ worth it at scale; ★ not worth a network hop for a
     sub-millisecond workload.
```

**PROVE IT #14 — `idle_transaction_timeout` cleans up leaks.**
```bash
psql -h 127.0.0.1 -p 6432 shop -c "BEGIN; SELECT 1;" &
sleep 20
psql -c "SELECT count(*) FROM pg_stat_activity WHERE state='idle in transaction'"
```
```
 count
-------
   ★ 0        — PgBouncer killed it at idle_transaction_timeout=15.
```

---

## The design decision framework

```
★★★ THE POOL EXHAUSTS BECAUSE HOLD TIME ROSE.
    FIX HOLD TIME. NEVER RAISE THE POOL FIRST. ★★★

 ① ★ MEASURE HOLD TIME BEFORE TOUCHING ANY POOL SETTING
    SHOW STATS: ★ avg_xact_time vs avg_query_time
    ⇒ ★ a ratio > 5 means the application is doing work inside
      transactions. That is the bug.
    ⇒ pg_stat_activity: ★ 'idle in transaction' count

 ② ★ REDUCE HOLD TIME — the free wins, in order
    ✗ ★ NO NETWORK CALLS INSIDE A TRANSACTION (Topics 45, 52)
    ✗ no user think-time, no retries with backoff
    ✓ ★ hot rows LAST in the transaction (Topic 61)
    ✓ ★ fix N+1 (Topic 66) and missing indexes (Topic 54)
    ✓ statement_timeout, idle_in_transaction_session_timeout
    ⇒ ★ in the example this alone took p99 from 4,200 → 240 ms.

 ③ SIZE IT WITH LITTLE'S LAW, THEN CAP AT THE THROUGHPUT PEAK
    ★ required = arrival_rate × hold_time
    ★ start at (cores × 2) + spindles; MEASURE THE CURVE
    ⇒ ★ if raising the pool doesn't raise throughput, the
      bottleneck is elsewhere — stop raising it.
    ⇒ ★ QUEUE AT THE POOLER RATHER THAN CONTEND IN THE DATABASE.

 ④ ★ USE TRANSACTION MODE — AFTER THE AUDIT
    grep for: SET (without LOCAL) · pg_advisory_lock ·
              LISTEN/NOTIFY · CREATE TEMP · WITH HOLD ·
              SET SESSION CHARACTERISTICS
    ✓ ★ max_prepared_statements = 200 (PgBouncer 1.21+)
    ✓ ★ SET LOCAL only, inside transactions
    ✓ ★ pg_advisory_xact_lock only
    ✓ ★ a dedicated DIRECT connection for LISTEN
    ✓ ★ a small session-mode pool for migrations and anything else

 ⑤ ★ THE THREE-LAYER ARITHMETIC MUST CLOSE
    max_client_conn ≥ instances × client_pool_size
    ★ Σ(pool_size per db,user) + reserve_pool_size
      ≤ max_connections − superuser_reserved − autovacuum_workers
        − max_wal_senders − background workers
    ★ client pool 5–10 per instance (it only avoids local TCP setup)
    ★ a parallel query consumes 1 + workers slots

 ⑥ ★ TIMEOUTS THAT MAKE FAILURE FAST AND VISIBLE
    query_wait_timeout = 10        ★ fail fast, don't queue forever
    idle_transaction_timeout = 15  ★ kill leaks at the pooler
    server_lifetime = 3600         ★ bound backend memory growth
    + statement_timeout and idle_in_transaction_session_timeout
      on the PostgreSQL side

 ⑦ ★ THE FOUR ALERTS — cl_waiting IS A LEADING INDICATOR
    ✓ cl_waiting > 0 sustained 30 s
    ✓ maxwait > 2 s
    ✓ idle in transaction > 10
    ✓ ★ avg_xact_time / avg_query_time > 5

 ⑧ SCALE THE POOLER ITSELF
    ★ PgBouncer is single-threaded ⇒ ~20–40k qps per process
    ⇒ ★ so_reuseport + N processes · a sidecar per app host ·
      or pgcat/Odyssey
    ⇒ ★ and it is a SPOF — plan for it explicitly.

 ⑨ ★ BOUNDED CONCURRENCY MAKES EVERYTHING ELSE SAFER
    once concurrency is capped you can raise work_mem, enable
    parallel query, and lower max_connections — all of which were
    unsafe before.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Run `pgbench` at 10, 50, 100, 200, 500 and 1000 connections on your machine. Plot tps and p99. Identify the peak, and explain — using `vmstat` and `pg_stat_activity` wait events — why the curve bends *down* rather than flattening.

### Exercise 2 — medium (apply it)
Set up PgBouncer in transaction mode. Then demonstrate and fix each failure: (a) prepared statements failing; (b) `SET` leaking between clients in both directions; (c) a session advisory lock leaking; (d) `LISTEN` receiving nothing.

Then reproduce Little's Law: vary hold time and show throughput equals `pool_size / hold_time`.

### Exercise 3 — hard (production simulation)
A logistics API has 24 instances × 40 connections direct to PostgreSQL. p99 has crept to 4,200 ms, "too many clients" fires 12–40 times a day, system CPU is 28%, and **adding app instances made it worse**.

(a) Write the single query that diagnoses this, and interpret each of its three findings.
(b) 188 backends are idle in transaction. Find the cause and explain, using Little's Law, why no pool size would survive it.
(c) Rewrite the offending handler. Predict the change in hold time and required pool size.
(d) After that fix alone, most symptoms are gone. Explain why the biggest win required no infrastructure change.
(e) Now size the three pooling layers. Show the arithmetic including every reserved connection category.
(f) Explain why `default_pool_size` is set to the throughput peak rather than to the Little's-Law requirement.
(g) Run the transaction-mode audit. One finding is a security bug — identify it and explain the exposure.
(h) Give the four alerts and say which is leading rather than lagging.
(i) After pooling, `work_mem` was raised 4×. Explain why that became safe.
(j) PgBouncer is now a single point of failure saturating one core. Give three options and choose one, with justification.

---

## Mental model checkpoint

1. Why do more connections reduce throughput rather than merely flattening it? Name the three mechanisms.
2. State Little's Law for pool sizing. What does it imply about the correct response to pool exhaustion?
3. Name the three pooling modes and what each returns the connection on.
4. List six things that break in transaction mode. Which is a security bug and why?
5. Why do prepared statements break, and what are the two fixes?
6. Why must advisory locks be transaction-scoped behind a pooler?
7. Give the full arithmetic constraint linking `max_client_conn`, `default_pool_size` and `max_connections`.
8. Why is `work_mem` dangerous at high connection counts?
9. What does `avg_xact_time ≫ avg_query_time` tell you?
10. Which pooling metric is a leading indicator, and why?
11. Why does bounded concurrency make other settings safer?
12. When is a pooler *not* worth it?

---

## Quick reference card

**★ The curve:** 50 conns → 48,204 tps / 2.1 ms · 1000 conns → 18,402 tps / 122 ms. **The peak is ~2–4× cores.**

**★ Little's Law:** `pool_size = arrival_rate × hold_time`. **Fix hold time, never raise the pool.**

```ini
pool_mode = transaction
max_client_conn = 3000            ; ★ ≥ instances × client_pool
default_pool_size = 64            ; ★ PER (database, user)
min_pool_size = 16
reserve_pool_size = 8
query_wait_timeout = 10           ; ★ fail fast
idle_transaction_timeout = 15     ; ★ kill leaks
server_lifetime = 3600
max_prepared_statements = 200     ; ★ makes transaction mode safe
so_reuseport = 1                  ; ★ multiple processes
```
**★ Arithmetic:** `Σ pool_size + reserve ≤ max_connections − superuser_reserved − autovacuum − wal_senders − bgworkers`. **Parallel query uses 1 + N slots.**

**Transaction-mode audit**
```bash
rg "SET (search_path|role|work_mem)" | rg -v "SET LOCAL"   # ★ SET LOCAL only
rg "pg_advisory_lock"                                      # ★ use _xact_
rg "LISTEN |NOTIFY "                                       # ★ direct connection
rg "CREATE TEMP|WITH HOLD"
```

**Diagnose**
```sql
SELECT state, wait_event_type, wait_event, count(*) FROM pg_stat_activity
 WHERE backend_type='client backend' GROUP BY 1,2,3 ORDER BY 4 DESC;
```
```
SHOW POOLS;   -- ★ cl_waiting, maxwait
SHOW STATS;   -- ★ avg_xact_time vs avg_query_time (µs); ratio > 5 = bug
```

**★ Four alerts:** `cl_waiting > 0` sustained · `maxwait > 2 s` · idle-in-transaction > 10 · `avg_xact_time / avg_query_time > 5`.

**★ Never inside a transaction:** an HTTP call, user think-time, a retry with backoff.

---

## When would I use this at work?

1. **The moment anyone proposes raising `max_connections`.** It is almost always the wrong lever. One `pg_stat_activity` query shows whether the connections are idle, idle-in-transaction, or actually working — and those three findings have three completely different fixes.

2. **Before switching a pooler to transaction mode.** The audit takes twenty minutes with `rg` and prevents a class of intermittent bug that is very hard to diagnose afterwards — including, in multi-tenant systems, cross-tenant data exposure.

3. **When adding application instances stops helping.** That's the signature of being past the throughput peak. More instances means more connections, and past the peak each one costs throughput.

4. **Any incident where the pool exhausted.** Ask what changed about *hold time*, not about the pool. A cache flush, a hot row, a slow dependency inside a transaction — each multiplies required pool size by orders of magnitude, and no pool size survives that.

---

## Connected topics

**Understand before this:** 46 (MVCC — snapshots and the `ProcArray` cost), 45 (locks — `idle in transaction`), 61 (hot rows — 412 of 500 connections blocked on one row), 63 (client timeouts and failover), 57 (a cache flush multiplying hold time).

**This unlocks:**
- **66** — N+1: the most common cause of long hold times
- **67** — performance investigation: distinguishing saturation from contention from queueing
- **69** — security at the data layer: `SET search_path` leakage and role handling behind a pooler
