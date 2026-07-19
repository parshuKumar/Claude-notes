# Topic 18 — The N+1 Query Problem
### SQL Mastery Curriculum — Phase 3: JOINs — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

You are a manager who needs a report: for each of your 100 employees, what department are they in?

**The slow way (N+1):**
You walk to the HR filing cabinet and pull out the master list of 100 employees. That's **1 trip**. Then, for each employee, you walk back to the cabinet, open the *departments* drawer, and look up their one department. That's **100 more trips** — one per employee. Total: **101 trips** to the filing cabinet.

Each trip is short, but the *walking* — opening the drawer, closing it, walking back to your desk — dominates. You spend the whole afternoon walking back and forth for a report you could have finished in two trips.

**The fast way (JOIN / batch):**
You pull the employee list (**1 trip**), read off all 100 department IDs, then make **one** trip to the departments drawer and grab all the department records you need in a single armful. Total: **2 trips**.

That is the entire N+1 problem. The "1" is the first query that fetches a list of N parent rows. The "+N" is the one-query-per-row you then fire to fetch each row's related data. The database can *answer* each of those N queries in under a millisecond — but the **round trip** (network hop, parse, plan, execute, return, hand back to your ORM) has fixed overhead that N multiplies into a disaster.

The database was never slow. Your code was just *walking to the cabinet* 101 times instead of 2.

---

## 2. Connection to SQL Internals

N+1 is not a *SQL* problem — it is a **client-side query-dispatch** problem. But understanding it requires knowing exactly what each of those N+1 round trips costs the engine.

Every single query — even `SELECT * FROM departments WHERE id = 42` — pays this full pipeline in PostgreSQL:

1. **Network round trip** — the wire protocol (libpq / the `pg` frontend/backend protocol) sends the query text to the server and waits. On localhost this is ~0.05–0.2ms; across an AZ it is 0.5–2ms; cross-region it can be 20–100ms. This is the dominant cost and it is paid *per query*.
2. **Parse** — the query string is tokenized and turned into a parse tree. Cached in the backend's plan cache only for prepared statements; ad-hoc text queries re-parse every time.
3. **Rewrite** — rules and view expansion.
4. **Plan/optimize** — the planner enumerates access paths. For a trivial PK lookup this is cheap but non-zero (~0.05–0.3ms).
5. **Execute** — a B-tree index probe on `departments_pkey`: 2–4 buffer-pool page reads (root → maybe internal → leaf → heap). If the pages are in `shared_buffers`, this is single-digit microseconds. This is the *only* part that is genuinely fast.
6. **Return + protocol** — row data serialized back over the wire, `pg` parses the `RowDescription` and `DataRow` messages, allocates JS objects.

Notice: the **actual index probe (step 5) is the cheapest thing in the list.** N+1 is death by fixed overhead. Doing 100 PK lookups is not "100× the work of one lookup" — it is **100× the per-query overhead** (network + parse + plan) plus 100× a nearly-free execution.

A single JOIN, by contrast, pays the pipeline **once**. The engine then uses a Hash Join or Nested Loop (Topic 11, Topic 17) to resolve all N matches inside one executor run, streaming results back in one protocol exchange. The B-tree, the buffer pool, and MVCC visibility checks all do the same *logical* work — but amortized across a single plan instead of N plans.

There is also a **connection-pool** dimension. Each of the N queries must acquire a pooled connection (Topic on pooling), and if your ORM issues them sequentially (the common case), they serialize: query 2 cannot start until query 1's promise resolves. N sequential round trips = N × latency, wall-clock. Even a "fast" 1ms round trip becomes 100ms of pure waiting for 100 rows.

---

## 3. Logical Execution Order Context

N+1 does not live inside a single query's logical execution order (FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT). It lives *between* queries, in your application code.

The fix, however, is precisely about collapsing what would be N+1 separate logical-execution-order pipelines into **one**:

```
-- The N+1 pattern executes this pipeline N+1 times:
Query 0:  FROM users                          → returns N user rows
Query 1:  FROM orders WHERE user_id = <u1>    → 1 pipeline
Query 2:  FROM orders WHERE user_id = <u2>    → 1 pipeline
...
Query N:  FROM orders WHERE user_id = <uN>    → 1 pipeline

-- The JOIN fix executes ONE pipeline:
FROM users u
LEFT JOIN orders o ON o.user_id = u.id        → single FROM/JOIN phase
```

The key insight tied to earlier topics: when you write a JOIN, the **FROM/JOIN phase** (Topic 11) resolves *all* matches in one planned operation — the planner picks Hash/Merge/Nested Loop *once*, filters in WHERE *once*, and the whole result streams back in one round trip. The N+1 anti-pattern throws away the planner's ability to batch by manually driving the "loop" from application code, where each iteration pays full pipeline overhead.

The `IN (...)` batching fix (the DataLoader pattern, section 6.6) is a middle ground: it collapses the +N into **one additional query** using `WHERE user_id = ANY($1)`, so you get **2 total** pipelines instead of N+1 — without a JOIN's row-multiplication (Topic 11, section 6.2).

---

## 4. What Is The N+1 Query Problem?

The N+1 query problem is a performance anti-pattern where an application executes **one query to fetch a list of N parent records, then N additional queries** — one per parent — to fetch each record's related child data. The total is N+1 queries where 1 or 2 would suffice. It almost always originates from an ORM's **lazy loading** of relations inside a loop.

### The Anatomy

```
Query count = 1 (the parent list)  +  N (one per parent row for the relation)
              └── the "1" ─────────┘   └──── the "+N" ────────────────────┘
```

### Annotated: the code shape that produces it

```javascript
const users = await User.findAll();          // ← Query 1: SELECT * FROM users  (returns N rows)
                                             //   │
for (const user of users) {                  //   ├── loop over N parent rows
  const orders = await user.getOrders();     //   │   └── Query i: SELECT * FROM orders WHERE user_id = <user.id>
  console.log(user.name, orders.length);     //   │       └── fires ONE query PER iteration
}                                            //   │
                                             //   └── TOTAL: 1 + N queries
```

Annotated: the SQL that lazy loading silently generates

```sql
-- Query 1 (the "1"): the parent fetch
SELECT * FROM users;
--     └── returns, say, 500 rows

-- Queries 2..501 (the "+N"): one per parent, fired lazily on property access
SELECT * FROM orders WHERE user_id = 1;    -- ┐
SELECT * FROM orders WHERE user_id = 2;    -- ├── 500 near-identical queries
SELECT * FROM orders WHERE user_id = 3;    -- │   each: 1 round trip + parse + plan + trivial index probe
-- ... 497 more ...                        -- │
SELECT * FROM orders WHERE user_id = 500;  -- ┘
--         └── differ ONLY in the literal on the right of `=`
```

The tell-tale signature in a query log: **hundreds of near-identical queries** that differ only in one bound parameter or literal. That repetition is the entire diagnosis.

### The two fixes (previewed here, detailed in section 6)

```sql
-- Fix A — JOIN (one query, may multiply rows):
SELECT u.*, o.* FROM users u LEFT JOIN orders o ON o.user_id = u.id;

-- Fix B — batched IN (two queries, no multiplication):
SELECT * FROM users;                                  -- query 1
SELECT * FROM orders WHERE user_id = ANY($1);         -- query 2, $1 = '{1,2,3,...,500}'
```

---

## 5. Why N+1 Mastery Matters in Production

1. **It is the single most common ORM performance bug in existence.** More production incidents trace back to N+1 than to missing indexes. It hides because every individual query is fast — your slow-query log (which typically flags queries over 100ms–1s) never fires, because no single query is slow. The *aggregate* is what kills you.

2. **It scales with your data, not your code.** The code that N+1s looks identical whether the parent list has 10 rows or 100,000. It passes code review, passes tests against a seed database with 5 rows, then falls over the moment a real customer has 50,000 orders. The failure is invisible until production scale.

3. **It saturates your connection pool.** N sequential queries hold a pooled connection for the entire loop duration. Under concurrency — 50 web requests each doing an N+1 over 200 rows — you are firing 10,000 queries and starving the pool. Requests queue waiting for a connection; p99 latency explodes; the database `max_connections` ceiling is hit; new requests get "sorry, too many clients." A single endpoint's N+1 can take down the whole service.

4. **It wastes the planner and the network, not the disk.** Because each query's *execution* is cheap (indexed PK/FK lookups), adding indexes does **not** fix N+1 — a common misdiagnosis. Engineers add indexes, see no improvement, and are baffled. The bottleneck is round-trip overhead, which no index touches.

5. **It is invisible without deliberate observability.** You cannot see N+1 in application code review alone (the loop looks innocent) or in per-query metrics (each is fast). You need query *counting* per request, `pg_stat_statements`, or ORM query logging. Mastery means knowing how to *make it visible* before it reaches production.

6. **In Node.js it is especially insidious.** JavaScript's `async/await` inside a `for` loop serializes the N queries by default (each `await` blocks the next), turning N+1 into N sequential round trips — the worst case. GraphQL resolvers, which run per-field per-object, are an N+1 factory unless you deploy DataLoader (section 11).

---

## 6. Deep Technical Content

### 6.1 The Mechanics: Lazy Loading Is the Root Cause

Lazy loading is an ORM feature where a related entity is *not* fetched when the parent is loaded — it is fetched **on first access** of the relation property. This is convenient (you only pay for what you touch) and catastrophic in loops (you touch it N times).

```javascript
// Sequelize lazy loading — the relation is a METHOD, fetched on call
const user = await User.findByPk(1);   // SELECT * FROM users WHERE id = 1
const orders = await user.getOrders(); // SELECT * FROM orders WHERE user_id = 1  ← separate query, on demand
```

```python
# Django/Rails style — accessing the ATTRIBUTE triggers the query
user = User.objects.get(id=1)   # SELECT ... FROM users WHERE id = 1
orders = user.orders.all()      # SELECT ... FROM orders WHERE user_id = 1  ← lazy, on attribute access
```

The problem is not lazy loading itself — it is lazy loading **inside iteration**. One lazy load is fine. N lazy loads in a loop is N+1.

The opposite of lazy loading is **eager loading**: telling the ORM up front "when you fetch these parents, also fetch this relation, in as few queries as possible." Eager loading is the primary fix (sections 6.5, 12).

### 6.2 Why It's So Easy to Write Accidentally

The N+1 is often *invisible in the code* because the query is hidden behind a property access or a getter that doesn't look like I/O:

```javascript
// This looks like pure in-memory data access. It is not.
const posts = await Post.findAll({ limit: 20 });
const summary = posts.map(p => ({
  title: p.title,
  authorName: p.author.name,   // ← if `author` is lazy, THIS is a query. 20 posts = 20 queries.
}));
```

The developer sees `p.author.name` and thinks "field access." The ORM sees "lazy relation, not yet loaded, fire a query." Twenty times. Nothing in the code *reads* like I/O — that's the trap.

### 6.3 The Nested / Multiplied N+1

N+1 compounds. If for each user you load their orders (N queries), and for each order you load its items, you get an **N × M** explosion:

```javascript
const users = await User.findAll();               // 1 query
for (const user of users) {                        // N users
  const orders = await user.getOrders();           // N queries (one per user)
  for (const order of orders) {                     // M orders each
    const items = await order.getOrderItems();      // N × M queries (one per order)
  }
}
// Total: 1 + N + (N × M). For 100 users × 10 orders = 1 + 100 + 1000 = 1101 queries.
```

A three-level nesting (users → orders → items → product) can turn a single API request into **tens of thousands** of queries. This is how a "list orders" endpoint ends up taking 40 seconds.

### 6.4 Why Indexes Don't Fix It (The Misdiagnosis)

A tempting but wrong fix: "the child queries are slow, add an index on `orders.user_id`." An index makes each of the N queries go from, say, 0.8ms to 0.5ms. But the problem was never the 0.8ms of *execution* — it was the **fixed per-query overhead** (round trip + parse + plan ≈ 0.3–2ms *each*) multiplied by N. Even at 0.5ms execution, 500 queries × (0.5ms exec + 1ms overhead) = 750ms. The index shaved a rounding error. The fix must **reduce the query count**, not the per-query cost.

> Index the FK anyway — you need it for the *batched* fix (`WHERE user_id = ANY($1)`) and the JOIN fix to be fast. But understand: the index is necessary for the *solution* to be fast, not a solution by itself.

### 6.5 Fix 1 — Eager Loading via JOIN

Collapse N+1 into **one** query with a JOIN. The ORM fetches parents and children in a single round trip and stitches them into object graphs client-side.

```sql
-- One query replaces 1 + N
SELECT
  u.id AS user_id, u.name, u.email,
  o.id AS order_id, o.total_amount, o.status, o.created_at
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.id = ANY($1)          -- the N user ids from query "0", or a WHERE on users directly
ORDER BY u.id;
```

**Trade-off — row multiplication (Topic 11, section 6.2):** a JOIN fans out. A user with 50 orders produces 50 rows, with the user's columns (`name`, `email`) *repeated* on every row. The ORM de-duplicates in memory, but you paid to ship 50× the parent data over the wire. For wide parent rows joined to many children (or *multiple* one-to-many relations — the many-to-many explosion of Topic 11, section 6.3), the JOIN result can be *larger* than the N+1's total data. This is why ORMs sometimes prefer Fix 2.

### 6.6 Fix 2 — Batching via `IN` / `= ANY` (The DataLoader Strategy)

Collapse N+1 into **2** queries: fetch parents, collect their IDs, fetch *all* children in one query using `WHERE fk = ANY(array_of_parent_ids)`, then group children by FK in memory.

```sql
-- Query 1: parents
SELECT id, name, email FROM users WHERE /* ... */;

-- Query 2: ALL children in one shot (no per-parent query, no row multiplication of parents)
SELECT id, user_id, total_amount, status
FROM orders
WHERE user_id = ANY($1);       -- $1 = ARRAY[1,2,3,...,500]  (the parent ids)
```

Then in application code, build a `Map<user_id, Order[]>` and attach. This is exactly what a modern ORM does under `include` when it chooses the "separate queries" strategy, and it is exactly what **DataLoader** (section 11) does for GraphQL.

**Why 2 queries beats both N+1 and (sometimes) the JOIN:**
- vs **N+1**: 2 round trips instead of N+1. Overhead paid twice, not 501 times.
- vs **JOIN**: parent columns are shipped *once* (in query 1), not repeated per child. No fan-out multiplication. Better for wide parents / high fan-out.
- **Cost**: 2 round trips instead of the JOIN's 1. Negligible unless latency is huge.

`= ANY($1)` vs `IN ($1, $2, ...)`: in `pg`, pass a JavaScript array as a single parameter bound to a PostgreSQL array and use `= ANY($1)`. This keeps the query text *constant* regardless of batch size — so PostgreSQL's prepared-statement plan cache reuses one plan. Building `IN ($1,$2,...,$500)` creates a different query text for every batch size, defeating plan caching and risking the 65,535 parameter limit.

### 6.7 Detecting N+1 — The Full Toolkit

**A. ORM query logging (development).** Turn on SQL logging and *count* the queries per request. If you see 200 near-identical `SELECT ... WHERE fk = ?` lines, that's your N+1.

```javascript
// Sequelize: log every query
new Sequelize(url, { logging: (sql) => console.log(sql) });
// Prisma: enable query events
new PrismaClient({ log: ['query'] });
// Knex: knex.on('query', q => console.log(q.sql));
// TypeORM: { logging: true }
```

**B. Per-request query counter (staging/production guardrail).** Wrap the pool and increment a counter per query, attached to the request context. Assert in tests that an endpoint uses ≤ K queries. This turns N+1 into a *failing test* instead of a production incident.

```javascript
// Count queries in a test to lock in "no N+1"
let queryCount = 0;
pool.on('query', () => queryCount++);   // conceptual; pg needs a wrapper (section 11.5)
await request(app).get('/users-with-orders');
expect(queryCount).toBeLessThanOrEqual(2);  // fails if someone reintroduces N+1
```

**C. `pg_stat_statements` (production, the authoritative tool).** PostgreSQL's `pg_stat_statements` extension aggregates queries by *normalized text* (literals and parameters replaced by `$1`, `$2`). An N+1 shows up as **one normalized statement with an enormous `calls` count** and a large `total_exec_time` despite a tiny `mean_exec_time`.

```sql
-- Enable once (postgresql.conf: shared_preload_libraries = 'pg_stat_statements'; then restart)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- The N+1 smoking gun: high calls, low mean time, high total time
SELECT
  queryid,
  calls,
  round(mean_exec_time::numeric, 3)  AS mean_ms,
  round(total_exec_time::numeric, 1) AS total_ms,
  query
FROM pg_stat_statements
WHERE query ILIKE 'select%from orders where user_id%'
ORDER BY calls DESC
LIMIT 10;
```

```
 queryid  |  calls  | mean_ms | total_ms |                query
----------+---------+---------+----------+--------------------------------------------
 8837...  | 4821330 |  0.041  | 197674.5 | select * from orders where user_id = $1
```

Read it: **4.8 million calls**, each averaging **0.041ms** (blazing fast individually), totaling **197 seconds** of cumulative execution — plus network overhead *not even counted here*. A single query normalized form with millions of `calls` and a small `mean_exec_time` is the textbook N+1 fingerprint. Compare `calls` against how many times the *endpoint* was hit — if it's 500× higher, you have an N+1 with N≈500.

**D. APM / tracing.** Datadog, New Relic, OpenTelemetry span waterfalls show N+1 visually as a "picket fence" — hundreds of tiny sequential DB spans stacked inside one request span. The visual is unmistakable.

### 6.8 The `LIMIT` Interaction — A Subtle Trap

Batching with pagination has a gotcha: "fetch 20 users, and for each their **most recent 5 orders**." A naive `WHERE user_id = ANY($1) LIMIT 5` limits the *whole* result to 5 rows, not 5-per-user. The correct batched query needs a **lateral join** or a **window function** (`ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC)` then filter `rn <= 5`). This is why per-parent limits are one of the few cases where N+1 is *tempting* — but the windowed single query is still far better. (Window functions: later phase.)

```sql
-- Top 5 orders PER user, in ONE query (correct batched form with LIMIT semantics)
SELECT * FROM (
  SELECT o.*,
         ROW_NUMBER() OVER (PARTITION BY o.user_id ORDER BY o.created_at DESC) AS rn
  FROM orders o
  WHERE o.user_id = ANY($1)
) ranked
WHERE rn <= 5;
```

### 6.9 When N+1 Is Actually Fine (Nuance)

Not every per-row query is a bug. N+1 is acceptable when:
- **N is bounded and tiny** — fetching related data for a single entity's *one* parent (N=1) is not N+1.
- **The children are already cached** — DataLoader with a request cache, or an application cache, makes repeat lookups free.
- **The relation is rarely accessed** — lazy loading exists precisely so you *don't* pay for relations you don't touch. Eager-loading everything can be *worse* than N+1 if you rarely use the relation.

The rule: N+1 is a bug when **N is unbounded (grows with data) and the relation is accessed for every parent in a loop**. Otherwise, measure before "fixing."

---

## 7. EXPLAIN — The N+1 Pattern in the Plan

There is a crucial subtlety: **`EXPLAIN` cannot show you an N+1.** `EXPLAIN` analyzes *one* query. The N+1 is a *count* of queries, invisible to any single plan. This is *why* it's so hard to catch — your favorite tool doesn't see it.

### 7.1 What each individual N+1 query's plan looks like (deceptively perfect)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id = 42;
```

```
Index Scan using orders_user_id_idx on orders  (cost=0.43..8.61 rows=5 width=64)
                                                (actual time=0.018..0.024 rows=4 loops=1)
  Index Cond: (user_id = 42)
Buffers: shared hit=4
Planning Time: 0.061 ms
Execution Time: 0.039 ms
```

**This plan is flawless.** Index Scan, 4 buffer hits, 0.039ms execution. Every one of the N queries looks exactly this good. That is the trap — **each query passes every performance review; the aggregate of 500 of them is the disaster.** The `Planning Time: 0.061 ms` is *itself* overhead paid 500 times (~30ms of pure planning), and the network round trip (not shown by EXPLAIN at all) dwarfs even that.

### 7.2 What the JOIN fix's single plan looks like

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.id, u.name, o.id AS order_id, o.total_amount
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.id = ANY(ARRAY[1,2,3, /* ...500 ids... */ 500]);
```

```
Hash Right Join  (cost=18.50..289.30 rows=2500 width=52)
                 (actual time=0.42..4.18 rows=2470 loops=1)
  Hash Cond: (o.user_id = u.id)
  ->  Seq Scan on orders o  (cost=0.00..210.00 rows=12000 width=20)
                            (actual time=0.01..2.1 rows=12000 loops=1)
  ->  Hash  (cost=12.25..12.25 rows=500 width=36)
             (actual time=0.31..0.31 rows=500 loops=1)
        ->  Index Scan using users_pkey on users u  (actual rows=500 loops=1)
              Index Cond: (id = ANY('{1,2,...,500}'::integer[]))
Buffers: shared hit=402
Planning Time: 0.28 ms
Execution Time: 4.5 ms
```

**Reading it:** one plan, one `Planning Time` (0.28ms, paid *once* not 500×), 402 buffer hits total, **4.5ms** end to end — versus 500 × (0.061ms plan + 0.039ms exec + ~1ms network) ≈ **550ms+** for the N+1. The engine does the *same logical work* (probe orders by user_id) but batched into a single Hash Join instead of 500 separate Index Scans, and — critically — pays the round trip *once*.

### 7.3 What the batched `= ANY` fix's plan looks like

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, user_id, total_amount FROM orders WHERE user_id = ANY($1);
-- $1 = '{1,2,3,...,500}'
```

```
Index Scan using orders_user_id_idx on orders  (cost=0.43..1250.00 rows=2500 width=20)
                                                (actual time=0.05..3.2 rows=2470 loops=1)
  Index Cond: (user_id = ANY('{1,2,3,...,500}'::integer[]))
Buffers: shared hit=520
Planning Time: 0.15 ms
Execution Time: 3.6 ms
```

**Reading it:** a *single* Index Scan whose `Index Cond` matches all 500 ids via `= ANY`. One planning pass, one round trip, 3.6ms. No parent-column repetition (parents came from query 1). This is the DataLoader plan.

### 7.4 The detection query is the real "EXPLAIN" for N+1

Since no plan reveals N+1, the equivalent diagnostic is the `pg_stat_statements` query from section 6.7-C. Treat *that* as your "EXPLAIN for N+1": high `calls`, low `mean_exec_time`, high `total_exec_time` on a single normalized statement.

---

## 8. Query Examples

### Example 1 — Basic: The N+1 and its JOIN fix, side by side

```sql
-- ❌ THE N+1 (as issued by an ORM lazy-loading in a loop):
SELECT id, name, email FROM users LIMIT 50;   -- the "1"
SELECT * FROM orders WHERE user_id = 1;        -- ┐ the "+50"
SELECT * FROM orders WHERE user_id = 2;        -- │ (49 more, one per user)
-- ... 48 more ...                             -- ┘

-- ✅ THE FIX (one query, eager JOIN):
SELECT
  u.id AS user_id, u.name, u.email,
  o.id AS order_id, o.total_amount, o.status
FROM users u
LEFT JOIN orders o ON o.user_id = u.id      -- LEFT so users with zero orders still appear
WHERE u.id IN (SELECT id FROM users LIMIT 50)
ORDER BY u.id, o.created_at DESC;
```

### Example 2 — Intermediate: Batched `= ANY` fix (the DataLoader shape)

```sql
-- Two queries total, no row multiplication of the parent.
-- Query 1 — parents:
SELECT id, name, email
FROM users
WHERE status = 'active'
ORDER BY created_at DESC
LIMIT 100;

-- Query 2 — ALL children for those 100 parents, in ONE round trip:
SELECT id, user_id, total_amount, status, created_at
FROM orders
WHERE user_id = ANY($1)          -- $1 = ARRAY of the 100 ids from query 1
  AND status = 'completed'
ORDER BY user_id, created_at DESC;
-- Application then groups orders into Map<user_id, Order[]> and attaches.
```

### Example 3 — Production Grade: Nested relations without N+1

**Scenario:** an `/dashboard` endpoint must return the 100 most recent active users, each with their completed-order count, total spend, and their 3 most recent order line-item summaries. Tables: `users` (2M rows), `orders` (40M rows, indexed on `(user_id, created_at)`), `order_items` (180M rows, indexed on `order_id`). A naive ORM implementation would N+1 *twice* (orders per user, items per order) → 1 + 100 + up-to-300 = ~401 queries. **Target: 3 queries, < 50ms.**

```sql
-- Query 1 — the 100 parent users (single index scan on users(status, created_at)):
SELECT id, name, email
FROM users
WHERE status = 'active'
ORDER BY created_at DESC
LIMIT 100;

-- Query 2 — per-user aggregates AND the top-3 recent orders, batched.
-- Aggregates use COUNT(DISTINCT)/SUM to avoid fan-out (Topic 11 §6.8);
-- top-3 uses a window function to respect per-user LIMIT (§6.8 here).
WITH user_orders AS (
  SELECT
    o.id, o.user_id, o.total_amount, o.status, o.created_at,
    ROW_NUMBER() OVER (PARTITION BY o.user_id ORDER BY o.created_at DESC) AS rn,
    COUNT(*)      OVER (PARTITION BY o.user_id) AS order_count,
    SUM(o.total_amount) OVER (PARTITION BY o.user_id) AS total_spend
  FROM orders o
  WHERE o.user_id = ANY($1)          -- $1 = the 100 user ids from query 1
    AND o.status = 'completed'
)
SELECT id, user_id, total_amount, status, created_at, order_count, total_spend
FROM user_orders
WHERE rn <= 3                        -- 3 most recent orders per user
ORDER BY user_id, created_at DESC;

-- Query 3 — line items for exactly the order ids returned by query 2, batched:
SELECT oi.order_id, oi.product_id, p.name AS product_name, oi.quantity, oi.unit_price
FROM order_items oi
INNER JOIN products p ON p.id = oi.product_id
WHERE oi.order_id = ANY($2)          -- $2 = the (≤300) order ids from query 2
ORDER BY oi.order_id;
```

**EXPLAIN for Query 2 (the interesting one):**

```
Subquery Scan on user_orders  (cost=8200..9100 rows=300 width=72)
                              (actual time=6.1..11.4 rows=290 loops=1)
  Filter: (user_orders.rn <= 3)
  Rows Removed by Filter: 4100
  ->  WindowAgg  (cost=8200..8850 rows=4400 width=80)
                 (actual time=6.0..10.2 rows=4390 loops=1)
        ->  Sort  (cost=8200..8300 rows=4400 width=48)
              Sort Key: o.user_id, o.created_at DESC
              Sort Method: quicksort  Memory: 812kB
              ->  Index Scan using orders_user_id_created_at_idx on orders o
                    (actual time=0.05..3.1 rows=4390 loops=1)
                    Index Cond: (user_id = ANY('{...100 ids...}'::int[]))
                    Filter: (status = 'completed')
Buffers: shared hit=1240
Planning Time: 0.34 ms
Execution Time: 11.9 ms
```

**Perf expectation:** 3 queries, ~5ms + ~12ms + ~8ms ≈ **25ms total**, versus ~401 round trips × ~1.2ms ≈ **480ms+** for the N+1 version — a ~20× win, and one that *does not degrade* as users accumulate more orders (the batched queries stay flat; the N+1 grows with fan-out).

---

## 9. Wrong → Right Patterns

### Wrong 1: Lazy loading a relation inside a `.map()` / loop

```javascript
// ❌ WRONG — each `post.author` access lazy-loads: 1 + 20 = 21 queries
const posts = await Post.findAll({ limit: 20 });
const view = await Promise.all(posts.map(async p => ({
  title: p.title,
  author: (await p.getAuthor()).name,   // fires SELECT * FROM users WHERE id = ? per post
})));
```

**Exact wrong behavior:** 21 queries. In a query log: one `SELECT ... FROM posts LIMIT 20` followed by 20 × `SELECT ... FROM users WHERE id = $1`. Endpoint latency scales with `limit`.

**Why it's wrong at the execution level:** each `getAuthor()` is a full query pipeline (round trip + parse + plan + probe). The 20 author lookups pay 20× fixed overhead for data that could arrive in the *same* round trip as the posts.

```javascript
// ✅ RIGHT — eager load the relation: 1 query (JOIN) or 2 (batched)
const posts = await Post.findAll({
  limit: 20,
  include: [{ model: User, as: 'author', attributes: ['name'] }],  // eager
});
const view = posts.map(p => ({ title: p.title, author: p.author.name }));
```

### Wrong 2: "Fixing" N+1 by adding an index

```sql
-- ❌ WRONG diagnosis — the child queries are "slow", so:
CREATE INDEX orders_user_id_idx ON orders(user_id);
-- N+1 was 520ms. After index: 505ms. Barely moved. Baffling — until you realize
-- each query was already ~1ms of OVERHEAD + ~0.04ms of execution. The index cut
-- the 0.04ms, not the 1ms. The count of queries never changed.
```

**Why it's wrong:** the bottleneck is round-trip/parse/plan overhead × N, not per-query execution. An index reduces execution, which was already negligible.

```javascript
// ✅ RIGHT — reduce the QUERY COUNT (keep the index too, for the batched query's speed)
const users = await User.findAll({ where: { status: 'active' } });
const ids = users.map(u => u.id);
const orders = await Order.findAll({ where: { userId: ids } });  // WHERE user_id = ANY($1) — ONE query
// group orders by userId in memory
```

### Wrong 3: `IN ($1,$2,...,$n)` with a dynamically-built parameter list

```javascript
// ❌ WRONG — query text changes with every batch size; defeats plan cache;
//            explodes past the 65,535-parameter protocol limit for large batches
const placeholders = ids.map((_, i) => `$${i + 1}`).join(',');
const sql = `SELECT * FROM orders WHERE user_id IN (${placeholders})`;
await pool.query(sql, ids);   // 50,000 ids → 50,000 params → error / cache miss storm
```

**Exact wrong behavior:** for 500 ids you get 500 distinct query *texts* over the app's lifetime (one per batch size), each parsed/planned separately — plan cache thrash. For > 65,535 ids: `bind message has N parameter formats but 0 parameters` / protocol error.

```javascript
// ✅ RIGHT — pass ONE array parameter, constant query text, plan-cache friendly
const sql = `SELECT * FROM orders WHERE user_id = ANY($1)`;
await pool.query(sql, [ids]);   // ids is a single JS array → one $1, one plan, any size
```

### Wrong 4: Eager loading *everything* (over-correction)

```javascript
// ❌ WRONG — nuking N+1 by eager-loading relations you don't even use on this endpoint
const users = await User.findAll({
  include: [
    { model: Order, include: [{ model: OrderItem, include: [Product] }] },  // huge JOIN
    { model: Session }, { model: AuditLog },                                 // unused here!
  ],
});
// A single query, yes — but it JOINs 6 tables, multiplies rows massively
// (Topic 11 §6.3 many-to-many explosion), and ships MBs to render a name list.
```

**Why it's wrong:** trading N+1 for a Cartesian-ish fan-out. If a user has 100 orders × 5 items, that's 500 rows *per user* with all parent columns repeated. You fixed query *count* and created a data-volume and row-multiplication problem.

```javascript
// ✅ RIGHT — eager load ONLY the relations this endpoint renders, and prefer
//            batched separate queries for high-fan-out one-to-many relations
const users = await User.findAll({
  attributes: ['id', 'name'],
  include: [{ model: Order, attributes: ['id', 'totalAmount'], separate: true }], // Sequelize: batched, not JOIN
});
```

### Wrong 5: GraphQL resolver with no DataLoader

```javascript
// ❌ WRONG — the `author` field resolver runs once PER post; 100 posts = 100 user queries
const resolvers = {
  Post: {
    author: (post) => db.query('SELECT * FROM users WHERE id = $1', [post.authorId]),
  },
};
// Query { posts(first: 100) { title author { name } } }  →  1 + 100 queries
```

**Why it's wrong:** GraphQL executes field resolvers per-object. Without batching, every `author` field is an independent query — the canonical GraphQL N+1.

```javascript
// ✅ RIGHT — DataLoader batches all author-id lookups in a tick into ONE query
const userLoader = new DataLoader(async (ids) => {
  const { rows } = await pool.query('SELECT * FROM users WHERE id = ANY($1)', [ids]);
  const byId = new Map(rows.map(u => [u.id, u]));
  return ids.map(id => byId.get(id) ?? null);   // MUST return in the same order as ids
});
const resolvers = {
  Post: { author: (post) => userLoader.load(post.authorId) },  // 100 .load() calls → 1 query
};
```

---

## 10. Performance Profile

### Scaling table — one endpoint, one request, relation fan-out, ~1ms round trip

| Parent rows (N) | N+1 queries | N+1 wall-clock (sequential) | JOIN / batched | Speedup |
|---|---|---|---|---|
| 10 | 11 | ~13ms | 1–2 queries, ~3ms | ~4× |
| 100 | 101 | ~120ms | 1–2 queries, ~6ms | ~20× |
| 1,000 | 1,001 | ~1.2s | 1–2 queries, ~20ms | ~60× |
| 10,000 | 10,001 | ~12s | 1–2 queries, ~120ms | ~100× |
| 100,000 | 100,001 | ~120s (timeout) | 1–2 queries, ~1s | ~120× |

The N+1 column grows **linearly with data**; the fix stays roughly **flat**. This is the whole story: N+1's cost is `N × per_query_overhead`; the fix's cost is `~constant + O(result_size)`.

### Memory

- **N+1:** trivial per query (one small result each), but holds a pooled connection for the *entire* loop — the real resource cost is **connection occupancy**, not memory.
- **JOIN fix:** may build a Hash Join hash table (O(smaller side), bounded by `work_mem`; spills to disk as `Batches > 1` if exceeded — Topic 11 §7). Result set can be large due to parent-column repetition (fan-out).
- **Batched `= ANY` fix:** two modest result sets; app builds a `Map` (O(children) memory). Lowest total data over the wire for high fan-out.

### CPU

- **N+1:** dominated by **N× planning** on the server (each `Planning Time` repaid) plus N× protocol parsing on the client. The database CPU is burned re-planning the *same* query shape thousands of times — `pg_stat_statements` shows it as huge cumulative `total_plan_time`.
- **Fixes:** one plan (JOIN) or two (batched). Planning cost amortized to near-zero per row.

### Connection-pool interaction (the production killer)

Under concurrency, each in-flight N+1 request holds one pool connection for its whole loop. With a pool of 20 and 20 concurrent N+1 requests each looping 500 times, the pool is fully occupied for the loop duration; request #21 queues. This is how one N+1 endpoint causes **pool exhaustion** and cascading timeouts across *unrelated* endpoints sharing the pool. The fix (2 queries) frees the connection ~250× sooner.

### Optimization techniques specific to N+1

1. **Batch with `= ANY($1)`** — one array param, constant query text, plan-cache friendly, no fan-out. Default choice for one-to-many.
2. **JOIN eager load** — one round trip; best for one-to-one / small fan-out. Watch row multiplication for one-to-many.
3. **DataLoader** — request-scoped batching + caching; mandatory for GraphQL; also useful in REST service layers.
4. **Bound N** — always paginate the parent list (`LIMIT`). An N+1 over an *unbounded* parent set is unbounded queries.
5. **Index the FK** — `orders(user_id)` — necessary for the batched/JOIN query to be fast (not a fix for N+1 by itself, but a prerequisite for the fix's speed).
6. **Cache hot parents** — DataLoader's per-request cache, or an app cache, eliminates repeat child fetches within/across requests.

---

## 11. Node.js Integration

Node.js deserves special attention: its `async/await` semantics make the *default* N+1 the **worst possible** N+1 — fully sequential round trips.

### 11.1 The trap: `await` inside `for` serializes the N queries

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL, max: 20 });

// ❌ N+1, AND sequential: query i+1 cannot start until query i's promise resolves.
async function badUserOrders() {
  const { rows: users } = await pool.query('SELECT id, name FROM users LIMIT 500');
  for (const u of users) {
    // each await BLOCKS the loop — 500 round trips end to end, ~1ms each = ~500ms
    const { rows: orders } = await pool.query(
      'SELECT id, total_amount FROM orders WHERE user_id = $1', [u.id]
    );
    u.orders = orders;
  }
  return users;
}
```

`Promise.all(users.map(...))` would at least *parallelize* the 500 queries — but that floods the pool with 500 concurrent queries (only `max: 20` run at once; the rest queue) and still fires 500 queries. Parallelizing an N+1 is not fixing it; it's making it a denser flood.

### 11.2 The batched fix with `pg` (the idiomatic Node solution)

```javascript
// ✅ TWO queries total. Pass the id array as ONE parameter → `= ANY($1)`.
async function goodUserOrders() {
  const { rows: users } = await pool.query('SELECT id, name FROM users LIMIT 500');
  if (users.length === 0) return users;

  const ids = users.map(u => u.id);
  const { rows: orders } = await pool.query(
    `SELECT id, user_id, total_amount, status
     FROM orders
     WHERE user_id = ANY($1)`,          // constant query text, any batch size
    [ids]                               // $1 bound to a PostgreSQL integer[]
  );

  // group children by FK in memory — O(children)
  const byUser = new Map(users.map(u => [u.id, { ...u, orders: [] }]));
  for (const o of orders) byUser.get(o.user_id)?.orders.push(o);
  return [...byUser.values()];
}
```

### 11.3 The single-JOIN fix with `pg` + client-side stitching

```javascript
// ✅ ONE query. Accept row multiplication; de-dupe parents in memory.
async function joinUserOrders() {
  const { rows } = await pool.query(
    `SELECT u.id AS user_id, u.name,
            o.id AS order_id, o.total_amount, o.status
     FROM users u
     LEFT JOIN orders o ON o.user_id = u.id
     WHERE u.id IN (SELECT id FROM users LIMIT 500)
     ORDER BY u.id`
  );
  const byUser = new Map();
  for (const r of rows) {
    if (!byUser.has(r.user_id)) byUser.set(r.user_id, { id: r.user_id, name: r.name, orders: [] });
    if (r.order_id !== null) {               // LEFT JOIN → NULL order columns for users w/ no orders
      byUser.get(r.user_id).orders.push({ id: r.order_id, total: r.total_amount, status: r.status });
    }
  }
  return [...byUser.values()];
}
```

### 11.4 DataLoader — batching for REST service layers and GraphQL

DataLoader (Facebook's `dataloader` package) collects all `.load(key)` calls made within a single event-loop tick, then invokes your **batch function once** with the array of keys. It also caches per instance (create one per request to avoid cross-request data leaks).

```javascript
import DataLoader from 'dataloader';

// Create PER REQUEST (in your context factory), not globally — cache must not outlive a request.
function createLoaders(pool) {
  return {
    // Batch-load users by id
    userById: new DataLoader(async (ids) => {
      const { rows } = await pool.query(
        'SELECT * FROM users WHERE id = ANY($1)', [ids]
      );
      const byId = new Map(rows.map(u => [u.id, u]));
      // CONTRACT: return an array the SAME LENGTH and ORDER as `ids`; misses → null/Error
      return ids.map(id => byId.get(id) ?? null);
    }),

    // Batch-load a one-to-many: orders grouped by user_id
    ordersByUserId: new DataLoader(async (userIds) => {
      const { rows } = await pool.query(
        'SELECT * FROM orders WHERE user_id = ANY($1) ORDER BY created_at DESC', [userIds]
      );
      const byUser = new Map(userIds.map(id => [id, []]));
      for (const o of rows) byUser.get(o.user_id)?.push(o);
      return userIds.map(id => byUser.get(id) ?? []);   // one array element PER requested key
    }),
  };
}
```

```javascript
// Usage — 100 independent .load() calls in a tick collapse to ONE query each loader:
const loaders = createLoaders(pool);
const posts = await pool.query('SELECT id, title, author_id FROM posts LIMIT 100');
const enriched = await Promise.all(
  posts.rows.map(async p => ({
    ...p,
    author: await loaders.userById.load(p.author_id),   // 100 loads → 1 batched query
  }))
);
```

**The DataLoader batch-function contract (memorize this):**
1. Input is `readonly Key[]`; output must be `Value[] | Error[]` of the **exact same length**.
2. Output **order must match input order** — element `i` is the result for `keys[i]`. Build a `Map` and re-map by key; never rely on the DB returning rows in the input order.
3. Missing keys must still occupy their slot (return `null` or an `Error`), or every subsequent index is misaligned.
4. One DataLoader instance **per request** — its cache is a request-scoped memoization, not a global cache.

### 11.5 A per-request query counter to catch N+1 in tests/CI

```javascript
// Wrap the pool so every query increments a counter — assert query budgets in tests.
function instrumentPool(pool) {
  const orig = pool.query.bind(pool);
  pool.queryCount = 0;
  pool.query = (...args) => { pool.queryCount++; return orig(...args); };
  return pool;
}

// In a test:
instrumentPool(pool);
pool.queryCount = 0;
await getDashboard(pool);                 // the endpoint under test
expect(pool.queryCount).toBeLessThanOrEqual(3);   // FAILS if someone reintroduces N+1
```

### 11.6 GraphQL: DataLoader is not optional

In GraphQL, resolvers run **per field, per object**. A query for `posts { author { name } }` over 100 posts runs the `author` resolver 100 times. Without DataLoader that is 100 queries; with it, 1. Wire loaders into the request context:

```javascript
// Apollo Server context factory — fresh loaders each request
const server = new ApolloServer({ typeDefs, resolvers });
await startStandaloneServer(server, {
  context: async () => ({ loaders: createLoaders(pool) }),
});

const resolvers = {
  Post: { author: (post, _args, { loaders }) => loaders.userById.load(post.authorId) },
};
```

**Do ORMs handle N+1 for you?** Partially, and only if you *ask*. No ORM eager-loads by default — lazy loading is the default precisely because eager-loading everything is wasteful. You must explicitly opt in (`include`, `select`, `with`, `relations`, `.innerJoin`) per query. The ORM will then choose a JOIN or a batched-`IN` strategy. GraphQL specifically needs DataLoader on top, because the ORM can't see across independent resolver invocations.

---

## 12. ORM Comparison

The N+1 story is fundamentally an ORM story: every ORM *can* cause it (via lazy loading) and every ORM *can* fix it (via eager loading) — they differ in the default, the syntax, and the strategy (JOIN vs batched-IN) they pick.

### Prisma

**Can it cause N+1?** Yes — if you loop and query per item. **Can it fix it?** Yes, and its default `include`/`select` strategy is *batched separate queries* (like DataLoader), not a JOIN.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient({ log: ['query'] });   // ← log to SEE the query count

// ❌ N+1: querying inside a loop
const users = await prisma.user.findMany({ take: 100 });
for (const u of users) {
  u.orders = await prisma.order.findMany({ where: { userId: u.id } }); // 100 queries
}

// ✅ FIX: `include` — Prisma issues ~2 queries (users, then orders WHERE userId IN (...))
const usersWithOrders = await prisma.user.findMany({
  take: 100,
  include: { orders: { where: { status: 'completed' }, take: 3, orderBy: { createdAt: 'desc' } } },
});
// Prisma logs: SELECT ... FROM users ...  then  SELECT ... FROM orders WHERE userId IN ($1,$2,...)
```

**Where it breaks:** Prisma's `include` uses `IN ($1,$2,...)` with expanded parameters (not `= ANY`), so very large parent sets can approach parameter limits; and its default relation-load is separate queries — great for avoiding fan-out, but that's 1 query *per included relation level*, not a single JOIN. For a single-round-trip JOIN, use `relationLoadStrategy: 'join'` (newer Prisma) or `$queryRaw`. `findMany` with nested `include` handles per-relation `take` correctly (no LIMIT trap) because it batches per parent internally.

**Verdict:** Strong N+1 defaults *when you use `include`*. Enable query logging in dev to verify the count. It will never eager-load unless you ask.

---

### Drizzle ORM

**Can it cause N+1?** Yes (manual loops). **Can it fix it?** Yes — two idioms: `db.query...with` (relational, batched) and explicit `.leftJoin` (single JOIN).

```typescript
import { db } from './db';
import { users, orders } from './schema';
import { eq, inArray } from 'drizzle-orm';

// ✅ FIX A — relational query API, `with` (Drizzle batches related rows):
const result = await db.query.users.findMany({
  limit: 100,
  with: { orders: { where: (o, { eq }) => eq(o.status, 'completed'), limit: 3 } },
});

// ✅ FIX B — explicit single JOIN, then stitch in code:
const rows = await db
  .select({ userId: users.id, name: users.name, orderId: orders.id, total: orders.totalAmount })
  .from(users)
  .leftJoin(orders, eq(orders.userId, users.id))
  .limit(500);

// ✅ FIX C — manual batched IN (the DataLoader shape, fully typed):
const parents = await db.select().from(users).limit(500);
const kids = await db.select().from(orders).where(inArray(orders.userId, parents.map(u => u.id)));
```

**Where it breaks:** the explicit `.leftJoin` returns flat rows — *you* de-duplicate parents (row multiplication is yours to manage, Topic 11 §6.2). The relational `with` API handles stitching but you must trust its batching strategy (verify via logging).

**Verdict:** Excellent — gives you the *choice* of JOIN vs batched explicitly, with full type safety. `inArray(col, ids)` compiles to a clean parameterized `IN`.

---

### Sequelize

**Can it cause N+1?** Very easily — lazy loading via `instance.getX()` in a loop is the classic Sequelize N+1. **Can it fix it?** Yes — `include` (eager). Crucially, `include` defaults to a **JOIN**, but `separate: true` switches a one-to-many to a **batched separate query** (avoiding fan-out).

```javascript
const { Op } = require('sequelize');

// ❌ N+1: lazy getter in a loop
const users = await User.findAll({ limit: 100 });
for (const u of users) { u.orders = await u.getOrders(); }   // 100 queries

// ✅ FIX A — eager JOIN (one query, but fan-out multiplies user rows):
const usersJoin = await User.findAll({
  limit: 100,
  include: [{ model: Order, where: { status: 'completed' }, required: false }],
});

// ✅ FIX B — `separate: true` → Sequelize issues a SECOND batched query (WHERE user_id IN (...)),
//            avoids fan-out AND is the ONLY way to honor a per-parent `limit` correctly:
const usersBatched = await User.findAll({
  limit: 100,
  include: [{ model: Order, separate: true, limit: 3, order: [['createdAt', 'DESC']] }],
});
```

**Where it breaks:** the classic trap is `include` with a `limit` on the *parent* combined with a one-to-many JOIN — the JOIN fan-out makes the parent `limit` behave unexpectedly, and Sequelize sometimes wraps in a subquery. `separate: true` is essential for per-parent `limit` and for avoiding fan-out, but it costs an extra query per included relation. Default lazy getters (`getOrders`) are an N+1 waiting to happen.

**Verdict:** Powerful but foot-gun-heavy. Prefer `include` for eager loading; reach for `separate: true` on high-fan-out one-to-many and whenever you need a per-parent `limit`. Log SQL in dev.

---

### TypeORM

**Can it cause N+1?** Yes — lazy relations (`Promise`-typed relations) or `find` with per-item follow-ups. **Can it fix it?** Yes — `relations` option (may JOIN or batch depending on version) or explicit `leftJoinAndSelect` in QueryBuilder.

```typescript
import { DataSource } from 'typeorm';

// ❌ N+1: loading a lazy relation per entity in a loop
const users = await repo.find({ take: 100 });
for (const u of users) { u.orders = await u.orders; }   // lazy Promise relation → 100 queries

// ✅ FIX A — `relations` (eager fetch of the relation):
const usersEager = await repo.find({
  take: 100,
  relations: { orders: true },
});

// ✅ FIX B — QueryBuilder single JOIN (explicit, one query):
const usersJoin = await repo
  .createQueryBuilder('u')
  .leftJoinAndSelect('u.orders', 'o', 'o.status = :s', { s: 'completed' })
  .take(100)
  .getMany();
```

**Where it breaks:** `leftJoinAndSelect` + `.take()` has the classic pagination-with-JOIN problem — `take` limits *rows*, and fan-out means you get fewer than N parents. TypeORM mitigates with an internal id-subquery in `getMany()`, but it's subtle; verify. Eager-marked relations (`eager: true` on the entity) fire on *every* find — an over-eager N+1-avoidance that becomes its own over-fetch problem. Lazy relations (`Promise<Order[]>` typed) are silent N+1 sources.

**Verdict:** Capable via QueryBuilder; the `relations` option is convenient. Watch the `take` + JOIN interaction and avoid entity-level `eager: true` as a blanket setting.

---

### Knex.js

**Can it cause N+1?** Only if *you* write the loop — Knex is a query builder, not an ORM, so it has no lazy loading and thus no *automatic* N+1. **Can it fix it?** Yes — you write the JOIN or the batched `whereIn` explicitly. This transparency is Knex's strength here.

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

// ❌ N+1 only if you hand-roll it:
const users = await knex('users').limit(100);
for (const u of users) {
  u.orders = await knex('orders').where('user_id', u.id);   // 100 queries — YOUR loop
}

// ✅ FIX A — single JOIN:
const rows = await knex('users as u')
  .leftJoin('orders as o', 'o.user_id', 'u.id')
  .select('u.id as user_id', 'u.name', 'o.id as order_id', 'o.total_amount')
  .limit(500);

// ✅ FIX B — batched whereIn (the DataLoader shape; whereIn compiles to `= ANY`-style IN):
const parents = await knex('users').select('id', 'name').limit(500);
const kids = await knex('orders')
  .whereIn('user_id', parents.map(u => u.id))   // one query for all children
  .select('id', 'user_id', 'total_amount');
// group kids by user_id in memory
```

**Where it breaks:** Knex gives you no stitching — you assemble the object graph yourself for both JOIN (de-dupe parents) and batched (group children). `whereIn` with a very large array builds a long `IN (...)` list; for huge batches prefer `.whereRaw('user_id = ANY(?)', [ids])` to keep it a single array param.

**Verdict:** No hidden N+1 (no lazy loading), full control, but you own all the stitching. The most *transparent* option — what you write is what runs.

---

### ORM Summary Table

| ORM | Lazy default? | Eager-load API | Default eager strategy | Per-parent `LIMIT` safe? | N+1 risk |
|---|---|---|---|---|---|
| Prisma | Yes | `include` / `select` | Batched separate queries (`IN`) | Yes (nested `take`) | Low if you use `include` |
| Drizzle | Yes (manual) | `with` / `.leftJoin` | `with` batches; JOIN explicit | Yes (`with … limit`) | Low; you choose strategy |
| Sequelize | Yes (`getX()`) | `include` (+`separate`) | JOIN (or batched via `separate:true`) | Only with `separate:true` | High — lazy getters, fan-out+limit trap |
| TypeORM | Yes (lazy relations) | `relations` / `leftJoinAndSelect` | JOIN | Tricky (`take`+JOIN) | Medium — lazy relations, `eager:true` |
| Knex | No (no ORM lazy) | write JOIN / `whereIn` yourself | N/A (explicit) | You control it | None automatic; only if you loop |

**Cross-cutting rule:** *No ORM eager-loads by default.* Every one requires an explicit opt-in per query. The safest workflow is identical across all of them: **enable SQL query logging in development, hit the endpoint, count the queries.** If the count grows with your seed-data size, you have an N+1.

---

## 13. Practice Exercises

### Exercise 1 — Basic

You have:
- `users(id, name, email, status)`
- `orders(id, user_id, total_amount, status, created_at)` with an index on `orders(user_id)`

A teammate wrote this Node.js code and reports the `/customers` endpoint is slow:

```javascript
const users = await pool.query('SELECT id, name FROM users WHERE status = $1', ['active']);
for (const u of users.rows) {
  const orders = await pool.query('SELECT * FROM orders WHERE user_id = $1', [u.id]);
  u.orderCount = orders.rows.length;
}
```

1. How many queries does this issue if there are 300 active users?
2. Rewrite it to use exactly **2** queries with `= ANY($1)`, computing `orderCount` per user.
3. Then rewrite it again to use exactly **1** query (hint: `GROUP BY`, `COUNT`, `LEFT JOIN`).

```sql
-- Write your query here (parts 2 and 3)
```

---

### Exercise 2 — Intermediate (combines Topic 11 fan-out + N+1)

You have `users`, `orders`, and `order_items(id, order_id, product_id, quantity, unit_price)`.

A GraphQL endpoint returns, for the 50 most recent users: each user's name, their number of **distinct completed orders**, and their **total spend**. A developer's resolver lazy-loads `user.orders`, then for each order lazy-loads `order.items`, then sums.

1. How many queries does this nested-lazy resolver issue for 50 users averaging 8 orders each? Show the arithmetic (`1 + N + N×M`).
2. Write a **single** SQL query that returns `user_id, name, distinct_completed_orders, total_spend`, correctly avoiding the fan-out count bug (recall Topic 11 §6.8 — why is `COUNT(o.id)` wrong here?).
3. Explain why `total_spend` must use `SUM(DISTINCT ...)` carefully or be pre-aggregated — what goes wrong if you `SUM(o.total_amount)` after joining `order_items`?

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; naive answer is wrong)

Endpoint `/feed`: return the 100 most recent active users, and for **each** user their **3 most recent completed orders** (not more, not fewer), with each order's total. Tables: `users` (2M), `orders` (40M, indexed `(user_id, created_at DESC)`).

A developer "fixes" the N+1 like this and is happy there's now one query:

```sql
SELECT u.id, u.name, o.id AS order_id, o.total_amount
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'completed'
WHERE u.status = 'active'
ORDER BY u.created_at DESC
LIMIT 100;   -- ← the developer thinks this gives 100 users with their recent orders
```

1. Explain precisely why this query is **wrong** — what does `LIMIT 100` actually limit here, given the one-to-many fan-out?
2. Write the **correct** solution. It must return at most 3 orders per user for exactly 100 users. (Hint: window function `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)`, and you may need to establish the 100 users *first*.)
3. Would you implement this as 1 query or 2 (batched)? Justify based on fan-out and payload size.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You are handed a production incident: the `/dashboard` endpoint has p99 latency of 8 seconds and is intermittently returning `remaining connection slots are reserved for non-replication superuser connections`. `pg_stat_statements` shows:

```
 calls   | mean_exec_time | total_exec_time |          query
---------+----------------+-----------------+-----------------------------------------
 9284410 |     0.038      |    352807.5     | select * from order_items where order_id = $1
```

1. Diagnose the root cause from this row alone. What is `9284410` telling you?
2. Explain the connection-slot error's link to the N+1.
3. Describe your fix, and how you'd add a regression guard so it can't come back.
4. The tech lead says "just add an index on `order_items.order_id`." Explain why that alone won't fix the latency, and what it *is* good for.

```
-- Write your explanation and the corrected query approach here
```

---

## 14. Interview Questions

### Junior Level

**Q: What is the N+1 query problem?**

**Junior answer:** It's when you run one query to get a list, then run another query for each item in the list, so you end up running way more queries than you need — 1 plus N. It usually happens with ORMs when you access a related object inside a loop.

**Principal answer:** It's a client-side query-dispatch anti-pattern: one query returns N parent rows, then N additional queries — one per parent — fetch each parent's related data, when 1 (JOIN) or 2 (batched `= ANY`) would suffice. The root cause is ORM lazy loading triggered inside iteration. The reason it's so damaging isn't the database work — each child query is a sub-millisecond indexed lookup — it's the **fixed per-query overhead** (network round trip + parse + plan) multiplied by N, plus, in Node, the sequential `await`-in-loop serialization, plus connection-pool occupancy under concurrency. It's invisible to slow-query logs because no single query is slow; you detect it via `pg_stat_statements` (one normalized statement with millions of `calls` and tiny `mean_exec_time`) or a per-request query counter.

**Interviewer follow-up:** *"If each child query is only 0.04ms, why does N+1 matter?"* → Because 0.04ms is the *execution*; the round trip and planning are ~1ms *each*, paid N times, serially. 500 × ~1ms = 500ms of pure overhead for data one JOIN delivers in 5ms.

---

**Q: How do you fix an N+1?**

**Junior answer:** Use eager loading — in the ORM, add an `include` or `with` or `join` so it fetches the related data in one query instead of one per row.

**Principal answer:** Two strategies. **(A) JOIN eager load** — one query, but it fan-out-multiplies parent rows (Topic 11 row multiplication), repeating parent columns per child; best for one-to-one / low fan-out. **(B) Batched `= ANY($1)`** — two queries: fetch parents, then fetch *all* children with `WHERE fk = ANY(array_of_ids)`, group in memory; no parent repetition, plan-cache friendly (constant query text), best for one-to-many / high fan-out — this is the DataLoader strategy. For GraphQL you *must* add DataLoader because resolvers run per-field-per-object and the ORM can't batch across independent resolver calls. Adding an index on the FK is a prerequisite for the fix to be fast, but is **not** itself a fix — it reduces per-query execution, which was already negligible; it doesn't reduce the query *count*.

**Interviewer follow-up:** *"When would you pick the JOIN over the batched approach?"* → One-to-one or small bounded fan-out, where parent-column repetition is cheap and you'd rather pay one round trip than two. For wide parents joined to many children, batched wins on payload and avoids the fan-out.

---

**Q: Why doesn't the slow-query log catch N+1?**

**Junior answer:** Because each individual query is fast — under the slow threshold — so none of them get logged even though there are thousands of them.

**Principal answer:** Slow-query logging (`log_min_duration_statement`) fires per-statement above a duration threshold. N+1's constituent queries are individually sub-millisecond indexed lookups, so none crosses the threshold. The pathology is *aggregate*: count × per-query overhead. You need count-aware tooling — `pg_stat_statements` (aggregates by normalized text, exposing `calls` and `total_exec_time`), APM span waterfalls (the "picket fence" of tiny sequential DB spans), or an application-level per-request query counter that you assert against a budget in CI.

**Interviewer follow-up:** *"Show me the `pg_stat_statements` query you'd run."* → `SELECT calls, mean_exec_time, total_exec_time, query FROM pg_stat_statements ORDER BY calls DESC LIMIT 20;` and look for a statement with huge `calls`, tiny `mean_exec_time`, large `total_exec_time`.

---

### Principal Level

**Q: A Node.js GraphQL API degrades under load; APM shows thousands of tiny sequential `SELECT ... FROM users WHERE id = $1` spans inside single requests. Walk me through the diagnosis and fix.**

**Answer:** That span shape — many tiny sequential same-shape DB spans in one request — is a textbook GraphQL N+1: a field resolver (e.g. `Post.author`) runs once per parent object, each firing an independent user lookup. The sequential arrangement suggests either `await`-in-loop or resolvers awaited one by one. Fix: introduce **DataLoader**, instantiated **per request** in the GraphQL context. The `userById` loader's batch function receives all author ids collected within an event-loop tick and issues a single `SELECT * FROM users WHERE id = ANY($1)`, returning results *re-mapped to the exact input order* (the batch-function contract — build a `Map` keyed by id, then `ids.map(id => map.get(id) ?? null)`). This collapses N author queries into 1, and DataLoader's per-request cache dedupes repeated ids for free. I'd also add a per-request query-count guardrail asserted in integration tests so the regression can't silently return. Verify with `pg_stat_statements`: the `calls` count on that normalized statement should drop from ~N×requests to ~1×requests.

**Interviewer follow-up:** *"Why per-request DataLoader instances and not one global?"* → The loader caches by key; a global instance would serve stale, cross-request, potentially cross-tenant data and never reflect writes. Per-request scoping makes the cache a safe within-request memoization.

---

**Q: Your ORM's eager loading fixed the N+1 but the endpoint is now shipping 40MB per response and is somehow slower. What happened and what do you do?**

**Answer:** The eager load used a **JOIN** across one or more one-to-many (or many-to-many) relations, causing row multiplication (Topic 11 §6.2–6.3): every parent column is repeated once per child, and with two independent one-to-many relations the result is a Cartesian-ish explosion (a user with 100 orders and 50 sessions → up to 5,000 rows). You traded query *count* for data *volume* and client-side de-dup cost. Fixes: (1) switch high-fan-out relations to the **batched separate-query** strategy (`separate: true` in Sequelize, Prisma's default `include`, Drizzle `with`, or a manual `= ANY` second query) so parent rows aren't repeated; (2) select only the columns the endpoint renders (stop `SELECT *`); (3) if you truly need aggregates, compute them with `COUNT(DISTINCT)`/`SUM` in SQL rather than shipping all children to count in JS; (4) paginate the children. The principle: eager-load, but pick JOIN vs batched based on fan-out, and never eager-load relations you don't render.

**Interviewer follow-up:** *"How do you decide JOIN vs batched programmatically?"* → Heuristic: one-to-one or bounded small fan-out → JOIN (one round trip, cheap repetition). One-to-many with unbounded or large fan-out, wide parent rows, or multiple sibling collections → batched `= ANY` (no repetition, avoids explosion), accepting one extra round trip.

---

**Q: Under concurrency, one N+1 endpoint is causing `too many clients already` errors that break unrelated endpoints. Explain the mechanism and the layered fix.**

**Answer:** Each in-flight N+1 request holds a pooled connection for the *entire* loop — with `await`-in-loop, that's the sum of N sequential round trips. Under concurrency, many requests each occupy a connection for that whole duration, so the pool's connections are all checked out for long stretches; new requests queue for a connection, and once the pool and the database's `max_connections` are saturated, PostgreSQL rejects new connections — affecting *every* endpoint sharing that database, not just the offender. Layered fix: **(1) eliminate the N+1** (batched `= ANY` or JOIN) so each request holds a connection for ~2 round trips instead of ~N, freeing connections ~N/2× faster — this is the real fix. **(2) Bound N** by paginating the parent list. **(3) Right-size the pool** and put a proxy pooler (PgBouncer) in front so app-side spikes don't map 1:1 to backend connections. **(4) Add per-request query budgets** in CI to prevent reintroduction, and alert on `pg_stat_statements` `calls` spikes. Note that adding an FK index does nothing for the connection pressure — the pressure is connection *hold time* × concurrency, driven by query *count*, not per-query speed.

**Interviewer follow-up:** *"If you could only ship one change tonight, which?"* → Eliminate the N+1 on the hot endpoint (batched query). It attacks the root — connection hold time — whereas pool tuning and PgBouncer only raise the ceiling before the same failure recurs.

---

## 15. Mental Model Checkpoint

1. An endpoint fetches 250 blog posts and renders each post's author name. Author is a lazy relation. How many queries fire, and what is the exact query-log signature you'd look for?

2. Each of the N child queries in an N+1 has `Execution Time: 0.03 ms` and a perfect Index Scan in `EXPLAIN`. Your teammate concludes "the queries are fine, the problem is elsewhere." Where is the cost that `EXPLAIN` isn't showing?

3. You replace an N+1 (`1 + 500` queries) with a single `LEFT JOIN`. The parent (`users`) has 30 columns; each user averages 40 orders. Roughly how many rows and how much *duplicated* parent data does the JOIN result contain, and when would the batched `= ANY` approach have been the better fix?

4. Why does passing `[ids]` as a single array parameter with `WHERE user_id = ANY($1)` play nicely with PostgreSQL's prepared-statement plan cache, while building `IN ($1,$2,...,$500)` does not?

5. In a DataLoader batch function you run `SELECT * FROM users WHERE id = ANY($1)` and simply `return rows`. Under what circumstance does this silently return wrong data to some callers, and what is the correct return contract?

6. You add an index on `orders.user_id` to "fix" an N+1 and see almost no improvement. Explain precisely which cost the index reduced and which cost — the one that actually dominates N+1 — it left untouched.

7. When is a per-row query *not* an N+1 problem — i.e., when is lazy-loading-per-item the correct, defensible choice rather than a bug to eliminate?

---

## 16. Quick Reference Card

```text
── WHAT IT IS ─────────────────────────────────────────────────────────────
N+1 = 1 query for N parents  +  N queries (one per parent) for their relation.
Root cause: ORM LAZY LOADING triggered inside a loop / per-object resolver.
Cost driver: per-query OVERHEAD (round trip + parse + plan) × N — NOT execution.
```

```sql
-- ── DETECT (pg_stat_statements) — the N+1 fingerprint ────────────────────
SELECT calls, round(mean_exec_time::numeric,3) AS mean_ms,
       round(total_exec_time::numeric,1) AS total_ms, query
FROM pg_stat_statements
ORDER BY calls DESC LIMIT 20;
-- ⇒ N+1 = one normalized query with HUGE calls, TINY mean_ms, LARGE total_ms.
```

```sql
-- ── FIX A: JOIN (1 query; multiplies/repeats parent rows — Topic 11 §6.2) ─
SELECT u.*, o.* FROM users u LEFT JOIN orders o ON o.user_id = u.id
WHERE u.id = ANY($1);
--   Best for: one-to-one / small fan-out.

-- ── FIX B: BATCHED = ANY (2 queries; no parent repetition) ───────────────
SELECT * FROM users WHERE ...;                    -- query 1: parents
SELECT * FROM orders WHERE user_id = ANY($1);     -- query 2: ALL children, $1 = id[]
--   Best for: one-to-many / high fan-out. This is the DataLoader strategy.
```

```javascript
// ── NODE.JS RULES ────────────────────────────────────────────────────────
// ❌ await INSIDE for-loop = N SEQUENTIAL round trips (worst case).
for (const u of users) await pool.query('... WHERE user_id=$1',[u.id]);
// ✅ pass the id ARRAY as ONE param → constant query text, plan-cache friendly:
await pool.query('... WHERE user_id = ANY($1)', [ids]);   // ids = [1,2,3,...]

// ── DATALOADER (GraphQL: mandatory) — per REQUEST, order-preserving ──────
new DataLoader(async (ids) => {
  const { rows } = await pool.query('SELECT * FROM users WHERE id = ANY($1)',[ids]);
  const m = new Map(rows.map(r => [r.id, r]));
  return ids.map(id => m.get(id) ?? null);   // MUST match input length & order
});
```

```text
── ORM ONE-LINERS ─────────────────────────────────────────────────────────
Prisma    : include:{orders:true}     → batched IN by default (low N+1 risk)
Drizzle   : with:{orders:true} OR .leftJoin(...)   → you choose batch vs JOIN
Sequelize : include:[{model:Order, separate:true}] → separate:true = batched, no fan-out
TypeORM   : relations:{orders:true} OR .leftJoinAndSelect(); avoid entity eager:true
Knex      : no auto N+1 (no lazy loading); write .leftJoin or .whereIn yourself
RULE: NO ORM eager-loads by default. Log SQL in dev; COUNT the queries.
```

```text
── PERF RULES OF THUMB ────────────────────────────────────────────────────
• N+1 cost grows LINEARLY with data; the fix stays ~FLAT.
• Indexing the FK does NOT fix N+1 (cuts execution, not query count/overhead).
  — but you STILL need it so the batched/JOIN fix is fast.
• Always paginate the parent list → bound N.
• Under concurrency, N+1 exhausts the connection pool → cascading failures.
• EXPLAIN cannot show N+1 (it sees ONE query). Use pg_stat_statements / query count.

── INTERVIEW ONE-LINERS ───────────────────────────────────────────────────
"N+1 is death by round-trip overhead, not by slow queries."
"Each query is fast; the aggregate is the disaster — that's why the slow log misses it."
"Two fixes: JOIN (1 query, row fan-out) or batched = ANY (2 queries, no fan-out)."
"GraphQL needs DataLoader because resolvers run per-object; the ORM can't batch across them."
"pg_stat_statements: huge calls + tiny mean_exec_time = N+1."
```

---

## Connected Topics

- **Topic 11 — INNER JOIN in Depth**: The JOIN fix for N+1 relies directly on join semantics — and on understanding row multiplication (§6.2) and the fan-out aggregation bug (§6.8) so the "fix" doesn't create a new bug.
- **Topic 12 — LEFT JOIN and RIGHT JOIN**: Eager-loading an optional relation needs LEFT JOIN so parents with zero children still appear — the N+1's lazy load would have returned an empty child list, which LEFT JOIN reproduces with NULL child columns.
- **Topic 17 — JOIN Performance Deep Dive** *(previous)*: Nested Loop / Hash / Merge selection determines whether your eager-load JOIN is fast; the batched `= ANY` fix leans on the FK index and the planner's array-probe handling.
- **Topic 19 — JOIN Alternatives** *(next)*: `EXISTS`, `IN`, and lateral joins as alternatives — the batched `= ANY` fix is itself a "JOIN alternative," and lateral joins solve the per-parent `LIMIT` trap (§6.8).
- **Topic 20 — GROUP BY Fundamentals**: Replacing an N+1 that counts children per parent with a single `GROUP BY ... COUNT(DISTINCT)` query — the aggregate fix.
- **Topic 07 — NULL in Depth**: LEFT-JOIN eager loading produces NULL child columns for childless parents; stitching code must handle them (see §11.3).
- **Internals — Connection Pooling & Round-Trip Cost**: The fixed per-query overhead (network + parse + plan) that N multiplies is the engine-level reason N+1 hurts; pool occupancy is the production failure mode.
- **Internals — `pg_stat_statements`**: The authoritative production detector — normalized-statement aggregation exposes N+1 as high `calls` + low `mean_exec_time`.
