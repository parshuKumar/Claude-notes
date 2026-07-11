# 39 — Database Connections over the Network

> **Phase 6 — Topic 5 of 5**
> Prerequisites: `01-how-the-internet-works.md`, `08-tcp-in-depth.md`, `28-load-balancers.md`

---

## ELI5 — The Simple Analogy

Imagine a busy restaurant kitchen with a phone line to a warehouse that supplies ingredients.

**The naive way:** Every time a cook needs one onion, they dial the warehouse from scratch, wait for it to ring, exchange greetings, verify who they are, place the order, hang up. For *one onion*. The dialing and greeting take longer than getting the onion. If 50 cooks do this at once, the warehouse's 10 phone operators are all tied up just saying hello, and cook #51 gets a busy signal.

**The smart way:** The restaurant keeps a small bank of phone lines *permanently open* to the warehouse — say 10 lines that never hang up. A cook who needs onions grabs a free line, places the order, and hands the line back to the next cook. No re-dialing, no re-greeting. Ten open lines serve hundreds of cooks because each call lasts a fraction of a second.

That bank of always-open lines is a **connection pool**. The re-dialing-and-greeting cost is the **TCP + TLS + authentication handshake**. And the "warehouse only has 10 operators" limit is Postgres's `max_connections`.

Now scale it up: you have 50 *restaurants* (app pods), each keeping 20 open lines. That's 1,000 lines into a warehouse with 100 operators. Everything jams. That is the single most common database outage in production, and this topic is about understanding and preventing it.

---

## Where This Lives in the Network Stack

A database connection is not some special "database thing." It is an ordinary **TCP connection** (Topic 08) carrying a database-specific application protocol on top.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application  (Postgres wire protocol, MySQL protocol) │  ← queries, results, auth
│  Layer 6 — Presentation (TLS — optional but common for DBs)     │  ← encrypt the session
│  Layer 5 — Session      (the DB "session" lives conceptually     │  ← stateful: txn, temp tables,
│                          here — but it's app-managed state)      │     prepared stmts, SET vars
│  Layer 4 — Transport    (TCP — port 5432 Postgres / 3306 MySQL) │  ← the actual connection
│  Layer 3 — Network      (IP — usually a PRIVATE address)        │  ← DB in a private subnet
│  Layer 2 — Data Link    (Ethernet inside the VPC/data center)   │
│  Layer 1 — Physical     (NIC, fiber inside the DC)              │
└─────────────────────────────────────────────────────────────────┘
```

Everything you learned about TCP handshakes, connection states (`ESTABLISHED`, `TIME_WAIT`), ports, load balancers, and latency comes together here. This is the capstone: the database connection is where **TCP cost**, **port limits**, **connection state**, **pooling**, and **latency** all collide in the most business-critical place in your stack.

---

## What Is This?

When your application "connects to Postgres," this physically happens:

1. Your app opens a **TCP socket** to the database host on port **5432** (Postgres) or **3306** (MySQL) — see Topic 05 for ports.
2. TCP does its **3-way handshake** (SYN → SYN-ACK → ACK) — one round trip.
3. The database's **wire protocol** does a startup handshake: the client says "I'm user `app`, database `orders`, protocol version 3.0."
4. Optionally, **TLS is negotiated** to encrypt the session — one or two more round trips.
5. **Authentication** happens (password / SCRAM-SHA-256 / certificate) — one or more round trips.
6. The connection is now an **established, stateful session**. The app sends queries and reads results over the *same* socket, reusing it for thousands of queries.

The key insight: **a database connection is expensive to create but cheap to reuse.** Establishing one costs multiple network round trips plus server-side setup (Postgres literally forks an OS process). Reusing one costs nothing but the query itself. Therefore you never open a connection per query — you keep a **pool** of warm connections and hand them out.

A **connection pool** is a set of already-established database connections that your app keeps open and lends to incoming requests. **PgBouncer** is an external pooler that takes this further: it multiplexes thousands of cheap client connections onto a handful of real Postgres connections.

---

## Why Does It Matter for a Backend Developer?

This is the topic that separates "my code works on my laptop" from "my service survives Black Friday." Concretely, without understanding it you cannot:

- Explain **why your API's p99 latency doubled** the moment traffic rose (pool exhaustion — requests queueing for a free connection).
- Diagnose the **`FATAL: sorry, too many clients already`** outage that took down every service pointed at one database.
- Understand why **autoscaling your app pods can crash your database** — more app instances silently multiply the total connection count.
- Choose between an **application-side pool** and an **external pooler like PgBouncer**, and know why "transaction mode" quietly breaks prepared statements and `SET` commands.
- Size `max`, `min`, `idleTimeout`, and `maxLifetime` on your pool so connections don't go stale after a **database failover**.
- Reason about the **network placement** of the database (private subnet, read replicas, cross-region round trips) and why an **L4 load balancer** in front of your DB can wreck pooling.

Databases are the number-one shared, finite, stateful resource in most backends. Every request path eventually touches one. Connection management is where the network layer and the database layer meet — and it's where most real outages start.

---

## The Packet/Protocol Anatomy

Here's what actually crosses the wire when a Postgres client connects and runs one query. Postgres uses a **message-based** protocol (protocol version 3.0) on top of TCP.

### The full lifecycle on one connection

```
CLIENT (app / pool)                              SERVER (Postgres :5432)
      │                                                  │
      │  ── TCP SYN ─────────────────────────────────►   │   Layer 4:
      │  ◄──── TCP SYN-ACK ───────────────────────────   │   3-way handshake
      │  ── TCP ACK ─────────────────────────────────►   │   (1 RTT)
      │                                                  │
      │  ── SSLRequest (optional) ───────────────────►   │
      │  ◄──── 'S' (SSL supported) ───────────────────   │   TLS negotiation
      │  ══════ TLS handshake (1–2 RTT) ═════════════    │   (Topic 09)
      │                                                  │
      │  ── StartupMessage ──────────────────────────►   │   "user=app
      │     {user, database, protocol=3.0}               │    database=orders"
      │  ◄──── AuthenticationSASL (SCRAM-SHA-256) ────   │
      │  ── SASLInitialResponse ─────────────────────►   │   Auth exchange
      │  ◄──── SASLContinue ──────────────────────────   │   (1–2 RTT)
      │  ── SASLResponse ────────────────────────────►   │
      │  ◄──── AuthenticationOk ──────────────────────   │
      │  ◄──── ParameterStatus (server_version, etc) ─   │
      │  ◄──── BackendKeyData (PID + secret) ─────────   │   ← a real OS PROCESS
      │  ◄──── ReadyForQuery ('I' = idle) ────────────   │      now exists for you
      │                                                  │
      │  ════════ CONNECTION IS NOW REUSED ═════════     │
      │                                                  │
      │  ── Query "SELECT * FROM users WHERE id=1" ───►  │   Simple query protocol
      │  ◄──── RowDescription ────────────────────────   │
      │  ◄──── DataRow ───────────────────────────────   │
      │  ◄──── CommandComplete ("SELECT 1") ──────────   │
      │  ◄──── ReadyForQuery ('I') ───────────────────   │
      │                                                  │
      │  ...(reused for thousands more queries)...       │
      │                                                  │
      │  ── Terminate ('X') ─────────────────────────►   │   Graceful close
      │  ── TCP FIN ─────────────────────────────────►   │   backend process exits
```

### Simple vs. Extended query protocol

Postgres has **two** ways to run a query over that established connection:

```
SIMPLE QUERY (one round trip, no parameters, no plan reuse):
  Client → Query("SELECT * FROM users WHERE id = 1")
  Server → RowDescription, DataRow..., CommandComplete, ReadyForQuery

EXTENDED QUERY (prepared statements — parse once, execute many):
  Client → Parse   ("SELECT * FROM users WHERE id = $1")   name="stmt1"
  Client → Bind    (stmt1, params=[1])                     portal="p1"
  Client → Execute (p1)
  Client → Sync
  Server → ParseComplete, BindComplete, DataRow..., CommandComplete, ReadyForQuery

  Later, reuse the parsed plan — skip Parse:
  Client → Bind (stmt1, params=[42])
  Client → Execute, Sync
```

The extended protocol (**Parse / Bind / Execute**) is how prepared statements and parameterized queries work. It prevents SQL injection (params never mix with SQL text) and lets the server cache the query plan. **This matters enormously for pooling:** a prepared statement named `stmt1` lives inside *one specific connection's* session. If a pooler hands you a *different* backend connection next time, `stmt1` doesn't exist there. Hold that thought — it's why PgBouncer's transaction mode breaks prepared statements.

### A connection is a stateful session

Everything below is bound to a *single physical connection* and is invisible to any other:

```
Per-connection session state (lost if the connection is swapped):
  • Current transaction    (BEGIN ... the txn lives on THIS connection)
  • Prepared statements     (PREPARE / the extended protocol's parsed plans)
  • Temporary tables        (CREATE TEMP TABLE ... visible only here)
  • Session variables       (SET search_path, SET timezone, SET statement_timeout)
  • Advisory locks           (pg_advisory_lock — held by THIS session)
  • Server-side cursors      (DECLARE cursor)
```

This is the deep reason connection management is subtle: a connection is not a stateless pipe. It carries a **session**, and swapping which physical connection serves a client mid-session corrupts that state.

---

## How It Works — Step by Step

### The cost of a single connection (why you pool)

```
Establishing ONE new Postgres connection, cross-AZ, with TLS + SCRAM auth:

  TCP handshake        1 RTT   ≈  0.5–1 ms   (same region, cross-AZ)
  TLS 1.3 handshake    1 RTT   ≈  0.5–1 ms
  SCRAM auth exchange  1–2 RTT ≈  1–2 ms
  Server fork + setup          ≈  1–3 ms     (Postgres forks a NEW OS PROCESS)
  ─────────────────────────────────────────
  Total setup          ≈ 3–7 ms  BEFORE your first query runs

  Cross-REGION (e.g. app in us-east, DB in eu-west, ~80ms RTT):
  Same handshakes, but each RTT ≈ 80 ms → 300+ ms just to connect.
```

If you open a fresh connection per query and your query itself takes 2 ms, you're spending **~4 ms of overhead per 2 ms of work** — the handshake dominates. Reuse the connection and the overhead is amortized to essentially zero.

### Why 10,000 raw connections kill Postgres — the process-per-connection model

This is the single most important mechanical fact in this entire topic.

```
Postgres uses a PROCESS-PER-CONNECTION architecture.
Every client connection = one dedicated OS process (a "backend").

        postmaster (parent process, listens on :5432)
             │  fork() on each new connection
   ┌─────────┼─────────┬─────────┬─────────┐
   ▼         ▼         ▼         ▼         ▼
 backend   backend   backend   backend   backend    ← each is a full OS process
 PID 4101  PID 4102  PID 4103  PID 4104  PID 4105
   │         │         │         │         │
 ~5-10 MB  ~5-10 MB  ~5-10 MB  ~5-10 MB  ~5-10 MB    ← baseline RAM per backend
  +work_mem +work_mem ...                            ← MORE per sort/hash/query
```

Consequences of thousands of these processes, **even if most are idle**:

```
1. MEMORY:  10,000 backends × ~10 MB baseline = ~100 GB of RAM
            ...just to hold idle connections. Plus work_mem PER OPERATION
            (a single query can use work_mem multiple times).

2. CONTEXT SWITCHING:  The OS scheduler must juggle 10,000 processes.
            CPU spends time switching between processes instead of doing work.

3. LOCK / LATCH CONTENTION:  Postgres has shared structures (the lock manager,
            the buffer mapping, the proc array). Every backend contends for
            these. More backends = more contention = each query gets slower.
            Throughput can DROP as you add connections past a point.

4. max_connections IS A HARD CAP:
            postgresql.conf:  max_connections = 100   (default, often 100–500)
            Connection #101 gets: FATAL: sorry, too many clients already
            (Some slots are reserved: superuser_reserved_connections.)
```

The counter-intuitive result: **more connections often means *less* total throughput.** A well-tuned Postgres is happiest with a *small* number of active connections — often the low dozens, roughly proportional to CPU cores. Idle connections aren't free either; each still costs a process and RAM. This is exactly why an external pooler that keeps the real connection count tiny is such a big win.

### The math trap in distributed apps — THE classic outage

```
The formula every backend engineer must burn into memory:

    TOTAL DB CONNECTIONS  =  (number of app instances)  ×  (pool size per instance)

Looks harmless in a config file. It's a landmine under autoscaling.

  ┌──────────┐  pool=20   ┐
  │ app pod 1│────────────┤
  ├──────────┤  pool=20   │
  │ app pod 2│────────────┤
  ├──────────┤  pool=20   │      50 pods × 20 = 1,000 connections
  │   ...    │    ...     ├───────────────────────────────►  ┌────────────┐
  ├──────────┤  pool=20   │                                   │  Postgres  │
  │ app pod50│────────────┤                                   │ max_conn   │
  └──────────┘            ┘                                   │  = 100     │
                                                              └────────────┘
                                                         900 connections REFUSED
```

The insidious part: **each pod's config looks reasonable.** "Pool of 20? Sure, that's small." But nobody multiplies by the pod count, and the pod count *changes automatically* under autoscaling. A traffic spike triggers the autoscaler → more pods → more pools → the DB silently blows past `max_connections` → new connections are refused → every service sharing that database starts erroring → cascading outage. It's a favorite interview question and a favorite 3 a.m. page precisely because the config that caused it looks fine in isolation.

### Application-side connection pooling mechanics

A pool sits inside your app process (or is shared across the process) and manages a set of connections:

```
                    APPLICATION-SIDE POOL (e.g. node-postgres, HikariCP)

  incoming request ──┐
  incoming request ──┼──►  ┌──────────────── POOL ────────────────┐
  incoming request ──┘     │                                       │
                           │  [conn1: IDLE ]  ◄── handed to req    │──► Postgres
                           │  [conn2: IN-USE]                      │──► Postgres
                           │  [conn3: IN-USE]                      │──► Postgres
                           │  [conn4: IDLE ]                       │──► Postgres
                           │  waiting queue: [req, req, req...]    │
                           └───────────────────────────────────────┘
                              max = 4  →  extra requests QUEUE here

  Lifecycle of one request:
    1. acquire()  — take an idle conn, or WAIT (up to acquireTimeout)
    2. run query  — over that connection
    3. release()  — return conn to pool (NOT closed — stays warm)
```

Key pool parameters and what they mean:

| Parameter | Meaning | Failure if wrong |
|-----------|---------|------------------|
| `max` (pool size) | Max simultaneous connections this pool opens | Too high → overwhelms DB; too low → requests starve |
| `min` (idle) | Connections kept warm even when idle | Too low → cold-start handshake on spikes |
| `acquireTimeout` / `connectionTimeoutMillis` | How long a request waits for a free conn before erroring | Too low → spurious timeouts under load; too high → requests pile up |
| `idleTimeout` | Close a connection idle for this long | Too high → stale idle conns waste DB slots |
| `maxLifetime` | Force-recycle a conn after this age | Prevents stale conns surviving a DB failover / DNS change |
| validation (`SELECT 1`) | Test a conn before handing it out | Catches dead conns after network blips |

**Pool exhaustion** is the classic symptom: all `max` connections are in use, so new requests sit in the waiting queue. They don't error immediately — they *queue*, which shows up as **latency spikes** (p99 climbs) and then, once `acquireTimeout` fires, as **timeout errors**. The pool is a queue in disguise: when it's too small, your slow endpoint isn't the DB — it's the wait for a connection.

How to *see* it: `pg_stat_activity` on the server side, `ss` / `netstat` at the TCP level (Topic 24), and your pool's own metrics (active/idle/waiting counts). If you're churning connections instead of reusing them — opening and closing per request — you'll also see a pile of `TIME_WAIT` sockets (Topic 24), the tell-tale sign of short-lived connections exhausting ephemeral ports.

### PgBouncer at the TCP level — multiplexing many clients onto few servers

An application-side pool bounds connections *per app instance*. But the M×P math trap means per-instance pools still explode when M grows. The fix is an **external pooler** that ALL app instances share, so the *total* real connection count to Postgres is capped centrally.

```
                            PgBouncer (the multiplexer)

  5,000 app clients                                          Postgres
  (cheap, lightweight              PgBouncer               (real, expensive
   TCP connections)          ┌───────────────────┐         backends)
                             │                    │
  client 1 ────────────────► │  client-side       │
  client 2 ────────────────► │  connections:      │  server-side
  client 3 ────────────────► │  up to             │  connections:
     ...                     │  max_client_conn   │  default_pool_size
  client 5000 ─────────────► │  = 10,000          │  = 20            │──► backend 1
                             │                    │──────────────────│──► backend 2
                             │  multiplexes  ─────┼──────────────────│──► ...
                             │  MANY clients      │                  │──► backend 20
                             │  onto FEW servers  │
                             └────────────────────┘
                                                         Postgres sees only 20
                                                         real connections. Ever.
```

Why this works: PgBouncer is a tiny, single-purpose C program. A client connection to PgBouncer is *cheap* — no fork, no per-process RAM, just a socket in an event loop. So 5,000 clients connecting to PgBouncer costs almost nothing, while PgBouncer keeps a *small* pool of genuine Postgres backends and lends them out. The expensive process-per-connection cost is paid only 20 times, not 5,000 times.

**Pooling modes** — this is the critical trade-off:

```
┌──────────────┬──────────────────────────────────────────────────────────┐
│ SESSION mode │ A server connection is assigned to a client for the WHOLE  │
│  (default)   │ duration of the client's connection (1:1 while connected). │
│              │ Safest — full session semantics work. LEAST reuse.         │
│              │ Reuse only happens when a client disconnects.              │
├──────────────┼──────────────────────────────────────────────────────────┤
│ TRANSACTION  │ A server connection is assigned to a client only for the   │
│  mode        │ duration of ONE transaction, then returned to the pool.    │
│  (most used) │ HIGHEST reuse — a server conn serves many clients rapidly. │
│              │ BUT: breaks anything that spans transactions or relies on  │
│              │ session state (see below).                                 │
├──────────────┼──────────────────────────────────────────────────────────┤
│ STATEMENT    │ Server conn returned after each STATEMENT. Even multi-     │
│  mode        │ statement transactions are disallowed. Rarely used.        │
└──────────────┴──────────────────────────────────────────────────────────┘
```

**Transaction mode gives the best connection reuse and is the most common production choice — but it silently breaks session-scoped features**, because between two transactions your client may be handed a *different* backend:

```
BREAKS in transaction mode (each relies on staying on the SAME backend):
  • Prepared statements  — PREPARE on backend A, EXECUTE lands on backend B
                           → "prepared statement does not exist"
  • SET / session vars   — SET search_path on backend A, next txn on backend B
                           → your setting silently vanished
  • Advisory locks        — pg_advisory_lock held on A; you can't unlock from B
  • LISTEN / NOTIFY       — the listening session isn't the one you get back
  • Temp tables            — created on A, gone when you land on B
  • Server-side cursors WITHOUT HOLD — don't survive the transaction boundary
```

Mitigations: disable client-side prepared statements (or use a driver mode that avoids named server-side prepared statements — newer PgBouncer versions support prepared statements in transaction mode with configuration), avoid session-level `SET` (set at connect-time via server defaults instead), and don't rely on advisory locks through a transaction-mode pooler. Many ORMs need explicit configuration to run cleanly behind transaction-mode PgBouncer.

**Where PgBouncer runs:**

```
  Sidecar (one PgBouncer per app pod):
    + simple, no shared bottleneck, low added latency
    - each sidecar has its own pool → M×default_pool_size again (smaller trap)

  Central (a few shared PgBouncer instances):
    + true global cap on Postgres connections (its whole point)
    - a network hop + a component that must be HA (don't make it a SPOF)
```

### Related network concerns

- **DB in a private subnet (Topic 31):** Databases should never have a public IP. They live in a private subnet inside the VPC, reachable only from app subnets via security groups (Topic 32). Your app reaches them over the private network — low, stable latency, no internet exposure.
- **Read replicas (Topics 33/35):** To spread read load and cut cross-region round trips, you route reads to a replica physically closer to the app. A read in the same region might be 1 ms; the same read to a primary across an ocean is 80+ ms. But replicas add **replication lag** — a read right after a write may not see it.
- **Why an L4 load balancer in front of a DB is dangerous:** An L4 LB (Topic 28) balances *TCP connections*, not queries, and can silently break things. If it re-balances or resets long-lived pooled connections, it destroys the session state and the whole point of pooling. If it round-robins each new connection to a *different* replica, you lose read-your-writes consistency and session affinity. Postgres connections are long-lived, stateful, and session-bound — the opposite of the stateless short requests L4 LBs are designed for. Use a purpose-built pooler/proxy (PgBouncer, RDS Proxy, pgpool) that understands the protocol, not a naive TCP load balancer.

---

## Exact Syntax Breakdown

### Node.js `pg` Pool config (application-side pool)

```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: 'orders-db.internal',   // private DNS name, resolves to a private IP
  port: 5432,                   // Postgres well-known port (Topic 05)
  database: 'orders',
  user: 'app',
  //
  max: 20,                      // ← MAX connections THIS pool opens.
                                //   THE number in the M×P math. 50 pods × 20 = 1000.
  min: 2,                       // keep 2 warm so a spike doesn't pay cold handshakes
  //
  connectionTimeoutMillis: 3000,// ← how long acquire() waits for a free conn
                                //   before throwing. Exhaustion surfaces HERE.
  idleTimeoutMillis: 30000,     // close a conn idle >30s → frees DB slots
  maxLifetimeSeconds: 1800,     // recycle conns after 30 min → survive failovers,
                                //   avoid stale conns after a DB IP change
  //
  ssl: { rejectUnauthorized: true }, // TLS to the DB (Topic 09)
});

// Correct usage — ALWAYS release, even on error:
const client = await pool.connect();   // acquire from pool (may queue/timeout)
try {
  const res = await client.query('SELECT * FROM users WHERE id = $1', [id]);
  //                                                          ^^ extended protocol
  //                                                          (Parse/Bind/Execute)
} finally {
  client.release();                    // ← return to pool. NOT client.end()!
}
```

```
max                        the single most important number — multiply by pod count
connectionTimeoutMillis    exhaustion shows up as timeouts here
idleTimeoutMillis          reclaim idle conns → don't hoard DB slots
maxLifetimeSeconds         forced recycle → resilience to failover / DNS change
client.release()           returns conn to pool (reuse); client.end() CLOSES it (don't)
```

### PgBouncer config (`pgbouncer.ini`) — external pooler

```ini
[databases]
orders = host=orders-db.internal port=5432 dbname=orders

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432                 ; app connects HERE, not to 5432 directly

pool_mode = transaction            ; ← session | transaction | statement
                                   ;   transaction = max reuse, breaks session state

max_client_conn = 10000            ; how many APP connections PgBouncer accepts (cheap)
default_pool_size = 20             ; ← real Postgres backends PER (user,db) pair.
                                   ;   THIS is what Postgres actually sees.
min_pool_size = 5                  ; keep a few backends warm
reserve_pool_size = 5              ; extra backends for bursts
server_idle_timeout = 600          ; close idle server conns after 10 min

auth_type = scram-sha-256
```

```
listen_port 6432       app points its connection string at PgBouncer, not Postgres
pool_mode              transaction = best reuse (but session-state caveats)
max_client_conn        the "many" side — thousands of cheap client conns
default_pool_size      the "few" side — the real cap on Postgres backends
```

### Observing connections — server side and TCP side

```sql
-- Server side: how many backends, and what are they doing?
SELECT count(*) FROM pg_stat_activity;                    -- total backends

SELECT state, count(*)                                    -- breakdown by state
FROM pg_stat_activity
GROUP BY state;
-- active            = running a query right now
-- idle              = connected, doing nothing (still costs a process!)
-- idle in transaction = holds a txn open, doing nothing (DANGEROUS — holds locks)

SHOW max_connections;                                     -- the hard cap
```

```bash
# TCP side (Topic 24): count established connections to Postgres from this host
ss -tanp state established '( dport = :5432 )'
#  -t tcp  -a all  -n numeric (no name lookup)  -p process  dport = :5432

# Count them:
ss -tan state established '( dport = :5432 )' | wc -l

# On the DB host, count who is connected TO port 5432:
ss -tan state established '( sport = :5432 )' | wc -l

# Watch TIME_WAIT pile up (sign of churn — NOT pooling, Topic 24):
ss -tan state time-wait '( dport = :5432 )' | wc -l
```

---

## Example 1 — Basic

**Goal:** See the difference between opening a connection per query vs. using a pool, and prove that pooling reuses the same backend.

### Without a pool — a fresh connection per query (the anti-pattern)

```javascript
const { Client } = require('pg');

async function getUserBad(id) {
  const client = new Client({ host: 'localhost', database: 'orders', user: 'app' });
  await client.connect();                    // ← full TCP + auth handshake EVERY call
  const res = await client.query('SELECT pg_backend_pid()');  // which backend am I?
  await client.end();                        // ← close it. TIME_WAIT socket left behind.
  return res.rows[0];
}

// Call it 3 times:
console.log(await getUserBad(1));  // { pg_backend_pid: 4101 }  ← new process
console.log(await getUserBad(2));  // { pg_backend_pid: 4102 }  ← ANOTHER new process
console.log(await getUserBad(3));  // { pg_backend_pid: 4103 }  ← ANOTHER new process
```

Each call spawns a **different backend PID** — Postgres forked a brand-new OS process, paid the handshake, and threw it away. Watch the churn:

```bash
$ ss -tan state time-wait '( dport = :5432 )' | wc -l
3        # three sockets in TIME_WAIT after three queries — pure waste
```

### With a pool — connections reused

```javascript
const { Pool } = require('pg');
const pool = new Pool({ host: 'localhost', database: 'orders', user: 'app', max: 2 });

async function getUserGood() {
  const res = await pool.query('SELECT pg_backend_pid()');   // acquire, run, release
  return res.rows[0];
}

console.log(await getUserGood());  // { pg_backend_pid: 4110 }
console.log(await getUserGood());  // { pg_backend_pid: 4110 }  ← SAME backend reused
console.log(await getUserGood());  // { pg_backend_pid: 4110 }  ← still the same one
```

Same PID every time — the connection (and its OS process) is reused. Confirm on the server:

```sql
SELECT count(*), state FROM pg_stat_activity
WHERE usename = 'app' GROUP BY state;

 count |   state
-------+-----------
     2 | idle          ← the pool's 2 warm connections, sitting ready
```

Two idle connections wait in the pool between requests instead of being torn down and rebuilt. **That is the entire value of pooling in one screen.**

---

## Example 2 — Production Scenario

### Case A — The autoscaling connection-exhaustion outage

**The situation:** It's a product launch. Your Node.js order service normally runs 8 pods, each with a `pg` pool of `max: 20`. That's 160 connections into a Postgres with `max_connections = 200`. Comfortable. Traffic surges. The Kubernetes HPA scales the deployment to **50 pods**. Suddenly every service that shares this database starts throwing errors and pages fire everywhere.

**The error, in every app log:**

```
error: sorry, too many clients already
    at Connection.parseE (/app/node_modules/pg/lib/connection.js:...)
  code: '53300'                         ← Postgres SQLSTATE: too_many_connections
```

Some pods also log:

```
FATAL:  remaining connection slots are reserved for
        non-replication superuser connections
```

**The diagnosis — walk the layers:**

Step 1 — On the DB, how many backends and what's the cap?

```sql
SELECT count(*) FROM pg_stat_activity;
 count
-------
   200                          ← pegged at the ceiling

SHOW max_connections;
 max_connections
-----------------
 200
```

Step 2 — Are they even doing work, or just idle?

```sql
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
      state       | count
------------------+-------
 active           |    12          ← only 12 actually running queries
 idle             |   181          ← 181 connections doing NOTHING but hogging a slot
 idle in transaction |    7
```

The smoking gun: **181 of 200 connections are idle.** The database isn't busy — it's *saturated with idle connections*. The process-per-connection model means those 181 idle backends still cost RAM and a process each, and they've consumed every slot so no *new* connection can be made.

Step 3 — Confirm the multiplication at the TCP level from a DB host:

```bash
$ ss -tan state established '( sport = :5432 )' | wc -l
200

# Group by source IP to see it's spread across many pods:
$ ss -tan state established '( sport = :5432 )' | awk '{print $4}' | \
    cut -d: -f1 | sort | uniq -c | sort -rn | head
   4  10.0.3.51     # each pod IP holds ~4 established conns on this DB node
   4  10.0.3.52     # (pool spread across replicas / partial ramp)
   4  10.0.3.53
   ...              # ~50 distinct pod IPs → the M×P explosion, made visible
```

Step 4 — Do the math out loud: **50 pods × 20 pool =  1,000 connections wanted, but `max_connections = 200`.** 800 connection attempts refused. The config `max: 20` never changed and looked fine — the *pod count* changed under autoscaling and multiplied it.

**The fix — put a central PgBouncer in transaction mode between the app and Postgres:**

```
BEFORE:  50 pods × pool 20  ──────────────────────────────►  Postgres (max 200)  ✗
                                                              1,000 wanted, 200 cap

AFTER:   50 pods × pool 20  ──►  PgBouncer  ──►  Postgres (max 200)               ✓
         (cheap client conns)   default_pool_size=40    only 40 real backends used
         max_client_conn=5000
```

```ini
# pgbouncer.ini
[pgbouncer]
pool_mode = transaction        ; return server conn after each txn → huge reuse
max_client_conn = 5000         ; absorb all 1,000 app conns (and headroom)
default_pool_size = 40         ; Postgres NEVER sees more than 40 backends
```

Point every pod's connection string at PgBouncer's `:6432` instead of Postgres `:5432`. Now 1,000 app-side connections multiplex onto 40 real backends. Postgres stays comfortable regardless of pod count. Also: **cap the total** so autoscaling can't outrun the pooler, and **right-size per-pod pools** down (each pod rarely needs 20 concurrent DB queries — many were idle). Because you switched to transaction mode, audit for prepared statements, `SET`, and advisory locks and configure the driver/ORM accordingly.

### Case B — Latency spikes because the pool is too SMALL

**The situation:** Different service, no exhaustion errors at all — but under moderate load, p99 latency jumps from 15 ms to 900 ms. The database's own query stats say every query runs in <5 ms. The DB is bored. So where do 900 ms come from?

**The diagnosis:** The pool is `max: 5`, but the endpoint handles ~40 concurrent requests, each needing a DB round trip. Only 5 can hold a connection at once; the other 35 **queue** inside the pool waiting for `acquire()`.

Instrument the pool:

```javascript
setInterval(() => {
  console.log({
    total: pool.totalCount,     // 5   ← maxed out
    idle: pool.idleCount,       // 0   ← none free
    waiting: pool.waitingCount, // 35  ← 35 requests STUCK in the acquire queue
  });
}, 1000);
// { total: 5, idle: 0, waiting: 35 }
```

`waiting: 35` with `idle: 0` is the signature of pool exhaustion by *undersizing*. The latency isn't in the DB or the network — it's time spent **waiting for a connection to free up**. On the server side, `pg_stat_activity` shows only 5 backends, all healthy. The bottleneck is entirely the app-side queue.

**The fix:** Raise `max` — but carefully, respecting the M×P math and `max_connections`. If the DB has headroom, bump the pool from 5 to, say, 20 and the queue drains, p99 drops back to ~15 ms. If raising it would breach `max_connections` across all pods, the real fix is a pooler (Case A) plus enough real backends. 

**The lesson tying both cases together:** the pool size is a **Goldilocks** number. Too big → you recreate Case A and overwhelm Postgres. Too small → you recreate Case B and starve your own requests into a queue. The right size is bounded above by `max_connections ÷ pod_count` and below by your endpoint's real concurrency — and when those two constraints conflict, that conflict *is* the reason PgBouncer exists.

---

## Common Mistakes

### Mistake 1: Opening a new connection per query/request (no pool)

```javascript
// WRONG — full TCP + TLS + auth handshake on EVERY request
app.get('/user/:id', async (req, res) => {
  const client = new Client(config);
  await client.connect();                        // 3–7ms handshake, forks a process
  const r = await client.query('SELECT ...', [req.params.id]);
  await client.end();                            // tears it down, leaves TIME_WAIT
  res.json(r.rows);
});
```
**Why it's wrong:** You pay the entire handshake + fork cost per request, and Postgres churns processes. Under load you pile up `TIME_WAIT` sockets (Topic 24) and can exhaust ephemeral ports.
```javascript
// RIGHT — one shared pool, acquire/release per request
const pool = new Pool({ ...config, max: 20 });
app.get('/user/:id', async (req, res) => {
  const r = await pool.query('SELECT ...', [req.params.id]); // reuses a warm conn
  res.json(r.rows);
});
```

---

### Mistake 2: Per-instance pools that multiply past `max_connections` when scaling

```
WRONG mental model: "My pool is max: 25, that's small and safe."
Reality under autoscaling:
    pool max 25  ×  40 pods  =  1,000 connections  >  max_connections 200  →  OUTAGE
```
**Why it's wrong:** You reasoned about *one* pod. The database sees *all* pods summed. Autoscaling changes the multiplier without changing your config.
```
RIGHT: Always compute TOTAL = pods × pool_max and keep it under max_connections
       (minus reserved slots and other services sharing the DB). When pods scale
       freely, put a central pooler (PgBouncer / RDS Proxy) in front so the DB's
       real connection count is capped centrally, not per-pod.
```

---

### Mistake 3: Pool too big (overwhelms DB) or too small (requests starve)

```
Too BIG:   max: 200 per pod × 10 pods = 2,000 backends → 20+ GB RAM, lock
           contention, throughput COLLAPSES even though "we gave it more".

Too SMALL: max: 3 for an endpoint doing 40 concurrent DB calls → 37 requests
           queue on acquire() → p99 latency explodes, DB sits idle.
```
**Why it's wrong:** Both extremes hurt. Bigger is NOT safer — past a point, more connections lower throughput. Smaller is NOT cheaper — it converts DB headroom into request-queue latency.
```
RIGHT: Size to actual concurrency, not "as high as it'll go." A common starting
       point: total ACTIVE connections ≈ a small multiple of DB CPU cores. Measure
       pool waiting/idle counts and pg_stat_activity 'active' vs 'idle', then tune.
```

---

### Mistake 4: Using transaction-mode pooling with session-dependent features

```javascript
// WRONG behind PgBouncer pool_mode = transaction:
await pool.query('SET search_path TO tenant_42');   // runs on backend A
await pool.query('SELECT * FROM orders');           // may run on backend B!
//   → search_path is back to default → queries the WRONG schema, silently

await pool.query('PREPARE q AS SELECT ...');        // prepared on backend A
await pool.query('EXECUTE q');                       // lands on B → "does not exist"

await pool.query('SELECT pg_advisory_lock(1)');     // locked on A
await pool.query('SELECT pg_advisory_unlock(1)');   // on B → can't unlock; lock leaks
```
**Why it's wrong:** Transaction mode may hand each transaction a *different* backend. Anything scoped to a session — `SET`, prepared statements, advisory locks, temp tables, `LISTEN/NOTIFY` — assumes it stays on one backend.
```
RIGHT: Behind transaction-mode PgBouncer, avoid session-scoped state. Set
       search_path/timezone via server-side role defaults (ALTER ROLE ... SET),
       disable client-side named prepared statements or use a driver mode that
       works with the pooler, and don't rely on advisory locks through the pooler.
       If you truly need session features, use SESSION mode (less reuse) for that path.
```

---

### Mistake 5: Ignoring idle/lifetime timeouts — stale connections after a failover

```
Managed DB fails over: primary IP changes, or the old backend is gone.
Your pool still holds "established" TCP connections to a socket that's dead
or points at the demoted node. Next query on a stale conn:
    Error: Connection terminated unexpectedly    (or a long hang, then error)
```
**Why it's wrong:** Without `maxLifetime` and connection validation, pooled connections can outlive the server they point to. After a failover you get a burst of errors from conns that "look" alive.
```
RIGHT: Set maxLifetime (e.g. 30 min) so connections are recycled and re-resolve
       DNS periodically. Enable validation (test query / TCP keepalive) so dead
       conns are detected and replaced before a request uses them.
```

---

### Mistake 6: Putting a naive L4 load balancer in front of the database

```
       app pool ──► L4 TCP load balancer ──► ┌ Postgres primary
       (long-lived,   (balances CONNECTIONS,  └ Postgres replica
        stateful       resets/rebalances them,
        conns)         no idea about sessions)

  Symptoms: pooled connections silently reset; reads scattered across
  replicas break read-your-writes; session state destroyed mid-flight.
```
**Why it's wrong:** L4 LBs (Topic 28) are built for many short, stateless requests. DB connections are few, long-lived, and stateful. Balancing/resetting them destroys the session and defeats pooling.
```
RIGHT: Use a protocol-aware pooler/proxy (PgBouncer, RDS Proxy, pgpool) that
       understands transactions and sessions — not a dumb TCP LB. Route reads to
       replicas explicitly in the app or via a proxy that respects consistency.
```

---

### Mistake 7 (bonus): Ignoring cross-region handshake/TLS RTT cost

```
App in us-east-1, DB in eu-west-1 (~80ms RTT). Opening a fresh connection:
   TCP handshake  = 80ms
 + TLS handshake  = 80–160ms
 + auth exchange  = 80–160ms
 = 240–400ms  BEFORE the first query — per connection, if you don't pool.
```
**Why it's wrong:** Cross-region multiplies every handshake RTT (Topic 33). Per-query connections become catastrophic across an ocean.
```
RIGHT: Keep the DB (or a read replica) in the app's region; pool aggressively so
       the handshake is paid once and amortized; use maxLifetime to keep warm
       connections alive rather than reconnecting across the WAN repeatedly.
```

---

## Hands-On Proof

```bash
# --- Server side: what does Postgres actually see? ---

# 1. Total backends and the hard cap
psql -c "SELECT count(*) AS backends FROM pg_stat_activity;"
psql -c "SHOW max_connections;"

# 2. Breakdown by state — how many are just idle?
psql -c "SELECT state, count(*) FROM pg_stat_activity GROUP BY state ORDER BY 2 DESC;"

# 3. Who is connected, from where, running what?
psql -c "SELECT pid, usename, client_addr, state,
                left(query,40) AS query
         FROM pg_stat_activity WHERE backend_type = 'client backend';"

# --- TCP side (Topic 24): connections at the socket level ---

# 4. Established connections to Postgres from THIS host
ss -tanp state established '( dport = :5432 )'

# 5. Count them
ss -tan state established '( dport = :5432 )' | wc -l

# 6. On the DB host: who's connected TO 5432, grouped by client IP (see M×P)
ss -tan state established '( sport = :5432 )' \
  | awk 'NR>1 {print $4}' | sed 's/:[0-9]*$//' | sort | uniq -c | sort -rn

# 7. Watch TIME_WAIT — pile-up means you're CHURNING conns, not pooling
watch -n1 "ss -tan state time-wait '( dport = :5432 )' | wc -l"

# --- Prove reuse vs. churn ---

# 8. Same backend PID across pooled queries = reuse working
psql -c "SELECT pg_backend_pid();"   # run twice via a pooled app → same PID

# --- Measure raw connect cost ---

# 9. How long does just CONNECTING take? (open + auth, then quit)
time psql "host=localhost dbname=orders user=app" -c "SELECT 1;"
#   Compare local vs a cross-AZ / cross-region host — watch the connect time grow.

# --- PgBouncer's own admin console (if using PgBouncer) ---

# 10. Connect to PgBouncer's admin DB and inspect the pools
psql -p 6432 pgbouncer -c "SHOW POOLS;"    # cl_active, cl_waiting, sv_active, sv_idle
psql -p 6432 pgbouncer -c "SHOW STATS;"    # per-database request/transaction rates
```

---

## Practice Exercises

### Exercise 1 — Easy: Prove pooling reuses connections

```bash
# 1. Start a Postgres locally (docker is fine):
docker run --rm -e POSTGRES_PASSWORD=pw -e POSTGRES_DB=orders -p 5432:5432 postgres:16

# 2. In another terminal, watch the backend count live:
watch -n1 "psql 'host=localhost user=postgres dbname=orders' \
  -c \"SELECT count(*), state FROM pg_stat_activity \
       WHERE usename='postgres' GROUP BY state;\""

# 3. Write a tiny Node (or Python) script that creates a Pool with max: 3
#    and fires 100 queries of: SELECT pg_backend_pid()
#
# Answer:
#   1. How many DISTINCT pg_backend_pid values appear across 100 queries?
#   2. What is the max backend count you see in the watch window? Why not 100?
#   3. Now rewrite it WITHOUT a pool (new Client per query). How many backends
#      appear now, and how many TIME_WAIT sockets does `ss` show afterward?
```

### Exercise 2 — Medium: Trigger and diagnose pool exhaustion

```bash
# Set up a pool with max: 2. Write an endpoint whose handler does:
#     await pool.query('SELECT pg_sleep(2)');   -- holds the conn 2 seconds
#
# Fire 10 concurrent requests at it (e.g. with `hey` or `ab -c 10 -n 10`).
#
# Instrument the pool: log pool.totalCount, pool.idleCount, pool.waitingCount
# every 500ms during the test.
#
# Answer:
#   1. What is the maximum waitingCount you observe?
#   2. How long does the 10th request take end-to-end, and why?
#   3. On the server, does pg_stat_activity ever show more than 2 'active'? Why not?
#   4. Where is the latency living — in the DB, the network, or the acquire queue?
#   5. What single config change fixes it, and what constraint bounds how far you
#      can push that change?
```

### Exercise 3 — Hard: Reproduce the M×P outage and fix it with PgBouncer

```bash
# 1. Start Postgres with a LOW cap so you can hit it:
docker run --rm -e POSTGRES_PASSWORD=pw -e POSTGRES_DB=orders \
  -p 5432:5432 postgres:16 -c max_connections=25

# 2. Simulate "many pods": launch 5 processes, each opening a pool of max: 10
#    (5 × 10 = 50 wanted > 25 cap). Have each process keep its connections open
#    and periodically query.
#
# 3. Observe the failure:
psql 'host=localhost user=postgres' -c "SELECT count(*) FROM pg_stat_activity;"
#    Watch some processes get: FATAL: sorry, too many clients already
ss -tan state established '( dport = :5432 )' | wc -l   # capped near 25

# 4. Now introduce PgBouncer in transaction mode in front (listen on 6432):
#      pool_mode = transaction
#      max_client_conn = 500
#      default_pool_size = 15
#    Repoint all 5 processes at PgBouncer's port 6432 instead of 5432.
#
# 5. Re-run and answer:
#   1. How many REAL backends does Postgres now see, regardless of the 50 clients?
#   2. Use `SHOW POOLS;` on PgBouncer — what are cl_active vs sv_active?
#   3. Add a `SET search_path` + later query in one of the clients. Does it behave
#      correctly in transaction mode? Explain what you observe.
#   4. What did you have to give up (session features) to gain the connection cap?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the linked section.

1. **A database connection IS a ____ connection.** Name the port for Postgres and for MySQL, and list the round-trip costs paid before the first query can run.

2. **Why does opening one connection per query destroy performance?** Name both the client/network cost and the server-side cost specific to Postgres's architecture.

3. **Explain in one sentence why 10,000 idle connections hurt Postgres even though they're idle.** What is the name of the architecture that causes this, and what hard config setting caps it?

4. **Write the M×P formula.** You have 40 app pods, each with pool `max: 30`, hitting a Postgres with `max_connections = 500`. Do you have a problem? What happens when the autoscaler doubles the pods?

5. **What is pool exhaustion, and what are its TWO different symptoms** — one when the pool is too small, one when it's too big? How would you tell which you're facing using `pg_stat_activity` and pool metrics?

6. **PgBouncer transaction mode gives the highest connection reuse. Name three session-scoped features it can break, and explain the underlying reason** (one sentence about which backend serves each transaction).

7. **Why is putting a naive L4 TCP load balancer directly in front of Postgres a bad idea?** Contrast the nature of a DB connection with the kind of traffic an L4 LB is designed for.

---

## Quick Reference Card

| Concept | What it is | Key number / fact |
|---------|-----------|-------------------|
| DB connection | A TCP connection + protocol handshake + auth | Postgres :5432, MySQL :3306 |
| Connection cost | TCP + TLS + auth RTTs + server setup | ~3–7 ms same-region; 240–400 ms cross-region |
| Process-per-connection | Postgres forks 1 OS process per backend | ~5–10 MB RAM each, even when idle |
| `max_connections` | Hard cap on Postgres backends | Default 100; often 100–500 |
| M × P trap | total = pods × pool_size_each | 50 pods × 20 = 1,000 → outage |
| Pool | App-side set of warm, reused connections | Amortizes handshake to ~0 |
| Pool exhaustion | All conns busy → requests queue | Symptom: p99 latency spike, then timeouts |
| PgBouncer | External pooler multiplexing many→few | 5,000 clients → 20 real backends |
| `pool_mode = transaction` | Server conn returned after each txn | Highest reuse; breaks session state |
| Session state | Txn, prepared stmts, temp tables, SET, advisory locks | Bound to ONE backend — don't swap mid-session |
| `TIME_WAIT` pile-up | Sign of churn, not pooling | Short-lived conns exhaust ephemeral ports (Topic 24) |

**Pool sizing rule of thumb:**
```
Lower bound: endpoint's real concurrent DB usage (else requests queue)
Upper bound: max_connections ÷ pod_count  (minus reserved + other services)
When those conflict → that's exactly why you deploy PgBouncer.
Active connections best kept to a small multiple of DB CPU cores.
```

**Diagnosis cheat block:**
```bash
# Server side
psql -c "SELECT count(*) FROM pg_stat_activity;"          # total backends
psql -c "SELECT state, count(*) FROM pg_stat_activity GROUP BY state;"  # idle vs active
psql -c "SHOW max_connections;"                            # the cap

# TCP side (Topic 24)
ss -tan state established '( dport = :5432 )' | wc -l      # conns to Postgres
ss -tan state time-wait   '( dport = :5432 )' | wc -l      # churn indicator

# App side (node-postgres)
pool.totalCount / pool.idleCount / pool.waitingCount       # waiting>0 = exhaustion

# PgBouncer
psql -p 6432 pgbouncer -c "SHOW POOLS;"                     # cl_waiting, sv_active
```

**The two failure signatures:**
```
too many clients / 53300  →  connections EXHAUSTED at the DB (pool too big × pods)
p99 spike, DB idle        →  connections STARVED at the app (pool too small)
```

---

## When Would I Use This at Work?

### Scenario 1: "We autoscaled during a sale and the whole platform went down"
Every service sharing the database started erroring with `too many clients already`. You now immediately compute pods × pool_size, check `pg_stat_activity` for a wall of `idle` connections, and confirm the M×P explosion with `ss` grouped by client IP. The durable fix is a central PgBouncer in transaction mode capping the real backend count, plus right-sized per-pod pools — not "just raise `max_connections`," which only delays the next crash.

### Scenario 2: "This endpoint's p99 is 900 ms but the DB says every query is 4 ms"
You know the latency is probably not in the DB or the wire — it's in the pool's acquire queue. You log `pool.waitingCount`, see requests stacking up against a too-small `max`, and either raise the pool (within the M×P budget) or add capacity. You've learned that a pool is a queue in disguise: undersize it and your fast database looks slow.

### Scenario 3: "After we put PgBouncer in transaction mode, we get random 'prepared statement does not exist' errors"
You recognize this instantly: transaction mode hands each transaction a potentially different backend, so named prepared statements, `SET`, temp tables, and advisory locks silently break. You reconfigure the ORM/driver to avoid session-scoped state (server-side role defaults for `search_path`, disable client prepared statements or use a compatible mode), or route the few session-dependent paths through a session-mode pool.

### Scenario 4: "Reads are slow for our European users hitting the US primary"
You connect the dots to latency (Topic 33) and read replicas (Topic 35): each cross-Atlantic round trip is ~80 ms, and a fresh connection pays that several times over in handshakes. You add a read replica in the EU region, route reads there, pool aggressively so the handshake is amortized, and account for replication lag in read-your-writes paths. You also make sure there's no naive L4 LB scattering connections across replicas and breaking session affinity.

---

## Connected Topics

**Study before this:**
- `08-tcp-in-depth.md` — the handshake, connection states, and teardown that a DB connection is built on
- `05-ports.md` — why 5432/3306 matter and how ephemeral ports get exhausted by churn
- `28-load-balancers.md` — L4 vs L7, and why L4 in front of a DB breaks pooling

**Study alongside / referenced here:**
- `24-netstat-and-ss.md` — reading `ESTABLISHED` and `TIME_WAIT`, spotting connection churn
- `31-vpns-and-private-networks.md` — why the DB lives in a private subnet, reached over the VPC
- `33-network-latency.md` — the RTT cost of every handshake, especially cross-region
- `35-network-in-system-design.md` — latency budgets, read replicas, and reasoning about DB placement in a design

**This is the capstone (39 of 39).** It ties together TCP (08), ports (05), connection states (24), load balancing (28), private networks (31), and latency (33) into the single most common real-world backend bottleneck: how your application talks to its database over the network. Master this, and you can walk into any "the database fell over" incident and reason from the socket up.

---

*Doc saved: `/docs/networking/39-database-connections-over-network.md`*
