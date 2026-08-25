# 66 — The N+1 Problem
## Phase: Reliability & Operations

---

## ELI5 — The Simple Analogy

You need the phone numbers of thirty people in a directory that lives in another building.

**The N+1 way:** walk over, ask for the list of thirty names, walk back. Then walk over again and ask *"what's Meera's number?"*, walk back. Then walk over and ask for Arjun's. Thirty-one round trips across the car park.

**The right way:** walk over once and say *"here are thirty names — give me all thirty numbers."* One trip.

★ **The person in the other building is not slow.** They answer each question in a fraction of a second. **The walking is the cost**, and you did it thirty-one times instead of once.

Three things follow, and they are why this is the most common performance bug in backend engineering:

1. ★ **It is invisible in development.** With three people in the directory and the building next door, four trips feel instant. With thirty people and the building across town, it's a minute.
2. ★ **Every individual trip looks fine.** Nobody's query is slow. The slow-query log shows nothing. The database's CPU is idle.
3. ★ **It is usually written by a tool, not a person.** An ORM, doing exactly what you asked, one row at a time.

---

## Where this fits in the big picture

```
   54 gate 2 — "★ fix the N+1, don't denormalise"
   65 pooling — ★ N+1 is the most common cause of long hold times
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 66 THE N+1 PROBLEM ← YOU ARE HERE            │
        │ ★ latency × count, not work × count          │
        └────────────────────┬─────────────────────────┘
                             ▼
              67 performance investigation
```

★ **This is the most frequently misdiagnosed problem in the curriculum.** It gets blamed on "slow joins" (Topic 53), triggers denormalisation proposals (Topic 54), and exhausts connection pools (Topic 65) — and the actual fix is usually one query rewrite.

---

## What is this?

Fetching a list of N parents, then issuing **one additional query per parent** to fetch its children — producing **N + 1** round trips where one or two would do.

```js
// ★ THE CANONICAL SHAPE
const orders = await db.query('SELECT * FROM orders LIMIT 30');   // ← 1
for (const o of orders.rows) {
  o.customer = await db.query(                                     // ← N
    'SELECT * FROM customers WHERE id = $1', [o.customer_id]);
}
// ⇒ ★ 31 round trips
```

**The cost is not database work. It is *latency*, multiplied:**

```
 ★ ONE ROUND TRIP = network RTT + parse + plan + execute + return
   same host        ★ ~0.15 ms   (mostly syscall + IPC)
   same datacentre  ★ ~0.5 ms
   cross-AZ         ★ ~1.2 ms
   cross-region     ★ ~40 ms

 ★ THE ACTUAL QUERY, on an indexed primary key: ★ 0.02 ms.

 ⇒ ★ 88% OF A SAME-HOST ROUND TRIP IS NOT THE QUERY.
   ⇒ cross-AZ: ★ 98%.
   ⇒ cross-region: ★ 99.95%.
 ⇒ ★ THIS IS WHY N+1 CANNOT BE FIXED BY MAKING QUERIES FASTER,
   BY ADDING INDEXES, OR BY DENORMALISING. The query was never
   the cost.
```

**Four shapes, and people only recognise the first:**

| Shape | Example |
|---|---|
| ★ **Classic** | list parents, then one query per parent |
| ★ **Lazy-loading in serialisation** | `JSON.stringify` triggers a getter that queries |
| ★ **Nested / N×M** | for each order, for each item, fetch the product |
| ★ **Write-side** | insert 500 rows in a loop |

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE EVERY SIGNAL SAYS THE DATABASE IS HEALTHY.

   slow query log       ★ EMPTY — no query exceeds 1 ms
   database CPU         ★ 6% — it is idle
   pg_stat_statements   ★ mean_exec_time 0.04 ms
   EXPLAIN on the query ★ perfect: Index Scan, 4 buffers
   the endpoint's p99   ★ 4,200 ms

 ⇒ ★ AND THE ONLY PLACE IT IS VISIBLE:
   pg_stat_statements.CALLS — the column nobody sorts by.

   SELECT calls, mean_exec_time, calls * mean_exec_time AS total_ms
     FROM pg_stat_statements ORDER BY ★ calls DESC;

 ★ THE DIAGNOSTIC RATIO THAT NAMES IT INSTANTLY:
   queries_per_request = total_queries / total_http_requests
   ⇒ ★ healthy: 3–15
   ⇒ ★ N+1 present: 50–500+
```

And the second-order damage, which is usually larger than the latency:

```
 ★ N+1 EXHAUSTS CONNECTION POOLS (Topic 65).
   Little's Law: pool = arrival_rate × hold_time
   ⇒ 200 rps × 31 round trips × 0.5 ms = ★ 3.1 s of connection-time
     per second per request-stream
   ⇒ ★ 200 rps needs 3,100 connection-milliseconds/sec = 3.1
     connections at 1 rps, but at 200 rps ⇒ ★ 620 connections.
 ⇒ ★ THE N+1 IS OFTEN DIAGNOSED AS "WE NEED A BIGGER POOL".
```

---

## The physical reality

### Where a round trip actually goes

```
 ★ ONE `SELECT * FROM customers WHERE id = $1` ON LOCALHOST:

   application: serialise the query + params        ★ 0.008 ms
   syscall: write() to the socket                   ★ 0.012 ms
   kernel: scheduling, socket buffers               ★ 0.020 ms
   postgres backend: wake up, read                  ★ 0.015 ms
   parse + rewrite                                  ★ 0.010 ms
   plan (or fetch a cached plan)                    ★ 0.018 ms
   ★ EXECUTE (index scan, 4 buffer hits)            ★ 0.020 ms
   serialise the result                             ★ 0.011 ms
   syscall: write() back                            ★ 0.012 ms
   kernel + application deserialise                 ★ 0.024 ms
   ─────────────────────────────────────────────────────────
   TOTAL                                            ★ ~0.15 ms
   ★ OF WHICH THE ACTUAL DATABASE WORK IS 0.02 ms — ★ 13%.

 ⇒ ★ 30 ROUND TRIPS = 4.5 ms OF WHICH 0.6 ms IS QUERY EXECUTION.
   ⇒ across an AZ boundary: 30 × 1.2 ms = ★ 36 ms, of which
     0.6 ms is work. ★ 98.3% OVERHEAD.

 ★ AND IT IS SERIAL. Each round trip must complete before the
   next begins, because the loop awaits.
   ⇒ ★ you cannot parallelise your way out with a bigger database.
```

### Why the ORM does this

```
 ★ LAZY LOADING IS THE DEFAULT IN EVERY MAJOR ORM, AND IT IS
   CORRECT IN ISOLATION AND CATASTROPHIC IN A LOOP.

 // Sequelize / TypeORM / ActiveRecord / Hibernate / Django
 const orders = await Order.findAll({ limit: 30 });
 orders.map(o => o.customer.name);
 //              ▲▲▲▲▲▲▲▲▲▲
 //   ★ each access issues a query. The ORM cannot know you are
 //     in a loop. It is doing exactly what the API promises.

 ★ THE THREE PLACES IT HIDES:
 ① ★ IN SERIALISATION
    res.json(orders)  ⇒ the serialiser walks every property
    ⇒ ★ touching a lazy relation issues a query
    ⇒ ★ the N+1 is in a library you did not write, triggered by a
      line that looks like it does no I/O.
 ② ★ IN A TEMPLATE
    {% for order in orders %}{{ order.customer.name }}{% endfor %}
 ③ ★ IN A GETTER / COMPUTED PROPERTY
    get total() { return this.items.reduce(...) }  ⇒ ★ loads items

 ★ AND THE ONE THAT SURVIVES EVERY FIX:
   `eager loading` configured for the LIST endpoint, but a
   different code path (a background job, an export, a webhook)
   uses the same model without it.
   ⇒ ★ THE FIX MUST BE ENFORCED, NOT CONFIGURED. (See below.)
```

### The three correct fixes, and when each applies

```
 ★ ① ONE QUERY WITH A JOIN
    SELECT o.*, c.name FROM orders o JOIN customers c ON c.id=o.customer_id;
    ✓ one round trip
    ✗ ★ ROW MULTIPLICATION on one-to-many: 30 orders × 8 items
      = 240 rows, with the order columns repeated 8 times
      ⇒ ★ measured: 30 orders with a 4 KB jsonb column and 8 items
        each transfers ★ 960 KB instead of 120 KB.
    ⇒ ★ correct for MANY-TO-ONE (parent per child). Dangerous for
      ONE-TO-MANY with wide parents.

 ★ ② TWO QUERIES + AN IN-MEMORY JOIN  ← ★ THE DEFAULT ANSWER
    const orders = await q('SELECT * FROM orders WHERE …');
    const ids = [...new Set(orders.map(o => o.customer_id))];
    const customers = await q('SELECT * FROM customers WHERE id = ANY($1)',
                              [ids]);
    const byId = new Map(customers.map(c => [c.id, c]));
    orders.forEach(o => o.customer = byId.get(o.customer_id));
    ✓ ★ 2 round trips regardless of N
    ✓ ★ no row multiplication
    ✓ ★ works across service boundaries where a JOIN is impossible
    ⇒ ★ THIS IS WHAT EVERY ORM'S "eager loading" ACTUALLY DOES.

 ★ ③ ONE QUERY WITH AGGREGATION — for nested structures
    SELECT o.*, coalesce(json_agg(i.*) FILTER (WHERE i.id IS NOT NULL),
                         '[]') AS items
      FROM orders o LEFT JOIN order_items i ON i.order_id = o.id
     GROUP BY o.id;
    ✓ ★ one round trip, ★ no row multiplication, nested shape
    ✗ the aggregation costs CPU on the database
    ⇒ ★ best when you need a tree and the parent rows are wide.

 ★ AND THE ONE FOR ARBITRARY DEPTH:
    a LATERAL join with a per-parent LIMIT — the only way to get
    "the 5 most recent items per order" in one query:
    SELECT o.*, i.* FROM orders o
      LEFT JOIN LATERAL (
        SELECT * FROM order_items WHERE order_id = o.id
         ORDER BY created_at DESC LIMIT 5) i ON true;
```

### The write-side N+1 — usually worse

```js
// ★ THE SHAPE
for (const row of rows) {                       // 500 rows
  await db.query('INSERT INTO events (a,b,c) VALUES ($1,$2,$3)', [...]);
}
```
```
 ★ 500 round trips AND 500 transactions (autocommit)
   ⇒ ★ 500 COMMITs = 500 fsyncs (Topic 41)
   ⇒ MEASURED: ★ 4,882 ms

 ★ FIX ①: ONE TRANSACTION — 500 round trips, ★ 1 fsync
   BEGIN; …500 inserts…; COMMIT;
   ⇒ MEASURED: ★ 412 ms.  ★ 11.8×, from two lines.

 ★ FIX ②: MULTI-ROW INSERT — 1 round trip, 1 fsync
   INSERT INTO events (a,b,c)
   SELECT * FROM unnest($1::int[], $2::text[], $3::int[]);
   ⇒ MEASURED: ★ 18 ms.  ★ 271×.

 ★ FIX ③: COPY — for thousands of rows
   COPY events (a,b,c) FROM STDIN WITH (FORMAT csv)
   ⇒ MEASURED, 500,000 rows: ★ 2.1 s vs 41 s for multi-row INSERT.

 ⇒ ★ AND THE UPDATE EQUIVALENT, which people miss:
   UPDATE t SET x = v.x FROM unnest($1::int[], $2::int[]) AS v(id, x)
    WHERE t.id = v.id;
   ⇒ one statement, N rows.
```

### When N+1 is actually fine

```
 ★ IT IS NOT ALWAYS A BUG. THREE CASES:

 ① ★ N IS BOUNDED AND SMALL, AND THE CONNECTION IS LOCAL
    N ≤ 5, same host ⇒ 0.75 ms of overhead.
    ⇒ ★ not worth the code complexity.

 ② ★ THE CHILDREN ARE CACHED
    a per-request or per-process cache means the "N" queries are
    mostly memory lookups.
    ⇒ ★ this is what a DataLoader does, and it is the right answer
      in GraphQL where the query shape is not known in advance.

 ③ ★ THE ALTERNATIVE IS WORSE
    a JOIN that multiplies 30 wide parent rows by 200 children
    transfers more bytes than 30 extra round trips cost.
    ⇒ ★ MEASURE BOTH. The join is not automatically better.

 ⇒ ★ THE TEST: N × round_trip_latency, versus the bytes and CPU
   of the alternative. Both are measurable in five minutes.
```

---

## How it works — step by step

### Detecting it

```sql
-- ★ ① SORT BY CALLS, NOT BY TIME. This is the whole trick.
SELECT calls,
       mean_exec_time::numeric(10,4) AS mean_ms,
       (calls * mean_exec_time / 1000)::numeric(12,1) AS total_seconds,
       substring(query from 1 for 70) AS query
  FROM pg_stat_statements
 ORDER BY ★ calls DESC LIMIT 10;
```
```
   calls    | mean_ms | total_seconds |                query
------------+---------+---------------+----------------------------------------
 ★ 88402118 |  0.0412 |       ★ 3,642 | SELECT * FROM customers WHERE id = $1
 ★ 41204882 |  0.0388 |       ★ 1,598 | SELECT * FROM products WHERE id = $1
      88420 |  8.2040 |           725 | SELECT … FROM orders WHERE …
   ★ 88 MILLION calls of a 0.04 ms query. Nothing is slow.
     Everything is repeated.
   ★ AND: the ratio of the top query's calls to the third's
     (88,402,118 / 88,420 = ★ 1,000) IS the N.
```

```sql
-- ★ ② THE RATIO THAT NAMES IT
SELECT sum(calls) AS total_queries FROM pg_stat_statements;
-- ÷ your application's total request count for the same window
-- ★ healthy 3–15 · ★ N+1 present 50–500+
```

```js
// ★ ③ PER-REQUEST INSTRUMENTATION — the definitive answer
const als = new AsyncLocalStorage();

app.use((req, res, next) => als.run({ queries: 0, ms: 0 }, next));

const origQuery = pool.query.bind(pool);
pool.query = async (...args) => {
  const t0 = process.hrtime.bigint();
  try { return await origQuery(...args); }
  finally {
    const ctx = als.getStore();
    if (ctx) {
      ctx.queries++;
      ctx.ms += Number(process.hrtime.bigint() - t0) / 1e6;
    }
  }
};

app.use((req, res, next) => {
  res.on('finish', () => {
    const { queries, ms } = als.getStore() ?? {};
    metrics.histogram('http.db_queries', queries, { route: req.route?.path });
    // ★ THE ALERT THAT CATCHES N+1 IN CI AND IN PRODUCTION
    if (queries > 25)
      log.warn({ route: req.route?.path, queries, dbMs: ms },
               '★ possible N+1');
  });
  next();
});
```

```js
// ★ ④ FAIL THE TEST SUITE — the only fix that stays fixed
test('GET /orders issues a bounded number of queries', async () => {
  const spy = countQueries();
  await request(app).get('/api/orders?limit=50');
  expect(spy.count).toBeLessThanOrEqual(4);   // ★ NOT 51
});
// ★ assert a CONSTANT, independent of the limit. Then run it
//   again with limit=200 and assert the SAME number.
test('query count does not grow with page size', async () => {
  const a = await countQueriesFor('/api/orders?limit=10');
  const b = await countQueriesFor('/api/orders?limit=200');
  expect(b).toBe(a);        // ★ THE REAL ASSERTION
});
```

### Fixing it — the batch-load pattern

```js
// ★ THE GENERAL SHAPE, worth having as a utility
async function batchLoad(rows, { key, table, into, select = '*' }) {
  const ids = [...new Set(rows.map(r => r[key]).filter(v => v != null))];
  if (ids.length === 0) return rows;
  const { rows: loaded } = await pool.query(
    `SELECT ${select} FROM ${table} WHERE id = ANY($1)`, [ids]);
  const byId = new Map(loaded.map(x => [x.id, x]));
  for (const r of rows) r[into] = byId.get(r[key]) ?? null;
  return rows;
}

// usage
const orders = (await pool.query('SELECT * FROM orders WHERE … LIMIT 50')).rows;
await batchLoad(orders, { key: 'customer_id', table: 'customers',
                          into: 'customer' });
await batchLoad(orders, { key: 'shipping_address_id', table: 'addresses',
                          into: 'address' });
// ★ 3 queries for 50 orders. Constant, regardless of N.
```

```js
// ★ ONE-TO-MANY needs a different helper
async function batchLoadMany(rows, { parentKey, table, fk, into }) {
  const ids = [...new Set(rows.map(r => r[parentKey]))];
  if (!ids.length) return rows;
  const { rows: children } = await pool.query(
    `SELECT * FROM ${table} WHERE ${fk} = ANY($1)`, [ids]);
  const grouped = new Map(ids.map(id => [id, []]));
  for (const c of children) grouped.get(c[fk])?.push(c);
  for (const r of rows) r[into] = grouped.get(r[parentKey]) ?? [];
  return rows;
}
```

### The single-query alternative with `json_agg`

```sql
-- ★ one round trip, nested shape, no row multiplication
SELECT
  o.id, o.total_minor, o.created_at,
  to_jsonb(c.*) - 'password_hash' AS customer,
  coalesce(
    (SELECT json_agg(json_build_object(
        'sku', i.sku, 'qty', i.qty, 'price_minor', i.unit_price_minor)
       ORDER BY i.id)
       FROM order_items i WHERE i.order_id = o.id),
    '[]'::json) AS items
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.customer_id = $1
ORDER BY o.created_at DESC
LIMIT 50;
```
```
 ★ NOTE THE CORRELATED SUBQUERY RATHER THAN A GROUP BY:
   • no row multiplication
   • ★ the ORDER BY inside json_agg is preserved
   • ★ and with an index on order_items(order_id, id) it is one
     index scan per order — which sounds like N+1, but it is
     ★ N index scans INSIDE ONE ROUND TRIP, which is the entire
     difference.
```

---

## Concept breakdown

```
★ WHAT IT IS
   N+1 round trips where 1–2 would do
   ⇒ ★ THE COST IS LATENCY × COUNT, NOT WORK × COUNT

★ THE NUMBERS
   round trip: ★ 0.15 ms local · 0.5 ms same-DC · 1.2 ms cross-AZ
               · ★ 40 ms cross-region
   the query itself: ★ 0.02 ms
   ⇒ ★ 87% overhead locally, ★ 98% cross-AZ, ★ 99.95% cross-region
   ⇒ ★ CANNOT BE FIXED BY INDEXES, FASTER QUERIES, OR
     DENORMALISATION

★ WHY IT'S INVISIBLE
   slow-query log ★ EMPTY · CPU ★ idle · EXPLAIN ★ perfect
   ⇒ ★ THE ONLY SIGNAL IS pg_stat_statements.CALLS —
     the column nobody sorts by
   ⇒ ★ THE RATIO: queries ÷ requests. Healthy 3–15, N+1 50–500+

★ FOUR SHAPES
   ① classic loop  ② ★ lazy-load during SERIALISATION
   ③ nested N×M    ④ ★ write-side (insert in a loop)

★ WHY THE ORM DOES IT
   lazy loading is correct in isolation, catastrophic in a loop
   ⇒ ★ hides in serialisers, templates and getters — lines that
     look like they do no I/O
   ⇒ ★ and eager loading configured on ONE path leaves every other
     path (jobs, exports, webhooks) broken

★ THE FIXES
   ① JOIN            ★ ✗ row multiplication on one-to-many
   ② ★ TWO QUERIES + Map    ← ★ the default; what ORMs do internally
   ③ json_agg / LATERAL     ★ one trip, nested, no multiplication
   ★ WRITE SIDE: one transaction (11.8×) → multi-row INSERT (271×)
     → COPY (for thousands)

★ WHEN IT'S FINE
   N small and bounded, local · children cached (DataLoader) ·
   the JOIN would transfer more bytes than the trips cost
   ⇒ ★ MEASURE BOTH

★ SECOND-ORDER DAMAGE
   ★ it exhausts CONNECTION POOLS (Topic 65) and is diagnosed as
     "we need a bigger pool"

★ THE FIX THAT STAYS FIXED
   ★ a test asserting query count is CONSTANT as page size changes
```

---

## Diagrams

**Diagram 1 — big picture: where the time actually goes**

```
  ONE ROUND TRIP (localhost), to scale
  ┌──────────────────────────────────────────────────────────────┐
  │ serialise ▓                                          0.008 ms│
  │ syscall   ▓▓                                         0.012 ms│
  │ kernel    ▓▓▓                                        0.020 ms│
  │ wake+read ▓▓                                         0.015 ms│
  │ parse     ▓▓                                         0.010 ms│
  │ plan      ▓▓▓                                        0.018 ms│
  │ ★ EXECUTE ▓▓▓                                      ★ 0.020 ms│
  │ serialise ▓▓                                         0.011 ms│
  │ syscall   ▓▓                                         0.012 ms│
  │ deserialise ▓▓▓▓                                     0.024 ms│
  └──────────────────────────────────────────────────────────────┘
                            ★ TOTAL 0.15 ms
              ★ DATABASE WORK: 0.020 ms = 13%

  ⇒ 30 ROUND TRIPS
  ┌──────────────────────────────────────────────────────────────┐
  │ ████████████████████████████████████████████████  4.5 ms     │
  │ ▓▓▓▓▓▓                                            0.6 ms work│
  └──────────────────────────────────────────────────────────────┘

  ⇒ 30 ROUND TRIPS, CROSS-AZ (1.2 ms each)
  ┌──────────────────────────────────────────────────────────────┐
  │ ██████████████████████████████████████████████… ★ 36 ms      │
  │ ▓                                                 0.6 ms work│
  └──────────────────────────────────────────────────────────────┘
       ★ 98.3% OVERHEAD. No index, no faster query, and no
         denormalisation changes this number.
```

**Diagram 2 — data flow: N+1 versus the two-query fix**

```
 ✗ N+1 — 31 SERIAL round trips
   app                                  database
    │── SELECT * FROM orders LIMIT 30 ──►│
    │◄─────────────── 30 rows ───────────│
    │── SELECT … customers WHERE id=1 ──►│   ★ each one must
    │◄──────────────── 1 row ────────────│     COMPLETE before
    │── SELECT … customers WHERE id=2 ──►│     the next STARTS
    │◄──────────────── 1 row ────────────│
    │            … × 30 …                │
    ⇒ ★ 31 × 0.5 ms = 15.5 ms of pure walking
    ⇒ ★ AND the pool connection is held for all of it (Topic 65)

 ✓ TWO QUERIES + AN IN-MEMORY MAP
   app                                  database
    │── SELECT * FROM orders LIMIT 30 ──►│
    │◄─────────────── 30 rows ───────────│
    │   ids = [...new Set(...)]  ★ dedup │
    │── SELECT … WHERE id = ANY($1) ────►│
    │◄────────── ≤30 rows ───────────────│
    │   new Map(...) + assign            │
    ⇒ ★ 2 × 0.5 ms = 1.0 ms.  ★ 15×
    ⇒ ★ AND IT IS CONSTANT: 300 orders is still 2 round trips.

 ✓ ONE QUERY WITH json_agg
    │── SELECT o.*, (SELECT json_agg(...)) FROM orders … ─►│
    │◄──────────── 30 nested rows ─────────────────────────│
    ⇒ ★ 1 × 0.5 ms + aggregation CPU
    ⇒ ★ N index scans happen INSIDE ONE ROUND TRIP —
      which is the entire difference.
```

**Diagram 3 — before/after: the write-side N+1**

```
 ✗ 500 INSERTS IN A LOOP, AUTOCOMMIT
 ┌───────────────────────────────────────────────────────────────┐
 │ for (const r of rows) await db.query('INSERT …', [...]);      │
 │                                                                │
 │ ★ 500 round trips        500 × 0.15 ms  =   75 ms             │
 │ ★ 500 TRANSACTIONS       500 × COMMIT                          │
 │ ★ 500 fsyncs             500 × ~9 ms    = ★ 4,500 ms          │
 │ ★ 500 WAL flushes                                              │
 │                                                                │
 │ MEASURED: ★ 4,882 ms                                           │
 └───────────────────────────────────────────────────────────────┘

 ✓ STEP 1 — ONE TRANSACTION (two lines of code)
 ┌───────────────────────────────────────────────────────────────┐
 │ BEGIN;  …500 inserts…  COMMIT;                                 │
 │ ★ 500 round trips, ★ 1 fsync                                   │
 │ MEASURED: ★ 412 ms      ★ 11.8×                                │
 └───────────────────────────────────────────────────────────────┘

 ✓ STEP 2 — MULTI-ROW INSERT (one statement)
 ┌───────────────────────────────────────────────────────────────┐
 │ INSERT INTO events (a,b,c)                                     │
 │ SELECT * FROM unnest($1::int[], $2::text[], $3::int[]);        │
 │ ★ 1 round trip, ★ 1 fsync                                      │
 │ MEASURED: ★ 18 ms       ★ 271×                                 │
 └───────────────────────────────────────────────────────────────┘

 ✓ STEP 3 — COPY (thousands of rows)
 ┌───────────────────────────────────────────────────────────────┐
 │ COPY events (a,b,c) FROM STDIN WITH (FORMAT csv)               │
 │ MEASURED, 500,000 rows: ★ 2.1 s   (multi-row INSERT: 41 s)     │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
CREATE TABLE customers (id bigserial PRIMARY KEY, name text NOT NULL,
                        email text NOT NULL);
CREATE TABLE orders (id bigserial PRIMARY KEY,
                     customer_id bigint NOT NULL REFERENCES customers(id),
                     total_minor bigint NOT NULL,
                     created_at timestamptz NOT NULL DEFAULT now());
CREATE TABLE order_items (id bigserial PRIMARY KEY,
                          order_id bigint NOT NULL REFERENCES orders(id),
                          sku text NOT NULL, qty int NOT NULL,
                          unit_price_minor bigint NOT NULL);
INSERT INTO customers (name,email)
  SELECT 'Customer '||g, 'c'||g||'@example.com' FROM generate_series(1,50000) g;
INSERT INTO orders (customer_id,total_minor)
  SELECT (random()*49999+1)::bigint, (random()*500000)::bigint
    FROM generate_series(1,500000);
INSERT INTO order_items (order_id,sku,qty,unit_price_minor)
  SELECT (random()*499999+1)::bigint, 'SKU-'||(random()*1000)::int,
         (random()*5+1)::int, (random()*50000)::bigint
    FROM generate_series(1,3000000);
CREATE INDEX ON orders (customer_id, created_at DESC);
CREATE INDEX ON order_items (order_id, id);
VACUUM ANALYZE;
```

**Measure the N+1.**
```js
console.time('n+1');
const orders = (await pool.query(
  'SELECT * FROM orders ORDER BY created_at DESC LIMIT 30')).rows;
for (const o of orders) {
  const { rows } = await pool.query(
    'SELECT * FROM customers WHERE id = $1', [o.customer_id]);
  o.customer = rows[0];
}
console.timeEnd('n+1');
```
```
 ★ n+1: 5.204 ms        (31 round trips on localhost)
```

**And the fix.**
```js
console.time('batched');
const orders2 = (await pool.query(
  'SELECT * FROM orders ORDER BY created_at DESC LIMIT 30')).rows;
const ids = [...new Set(orders2.map(o => o.customer_id))];
const { rows: custs } = await pool.query(
  'SELECT * FROM customers WHERE id = ANY($1)', [ids]);
const byId = new Map(custs.map(c => [c.id, c]));
orders2.forEach(o => (o.customer = byId.get(o.customer_id)));
console.timeEnd('batched');
```
```
 ★ batched: 0.884 ms        ★ 5.9× on localhost
```

**Prove it is latency, not work.**
```sql
SELECT calls, mean_exec_time::numeric(10,4) AS mean_ms,
       (calls*mean_exec_time)::numeric(10,2) AS total_ms
  FROM pg_stat_statements
 WHERE query LIKE '%customers WHERE id = $1%';
```
```
 calls | mean_ms | total_ms
-------+---------+----------
    30 |  0.0388 |  ★ 1.164
   ★ THE DATABASE DID 1.16 ms OF WORK. The request took 5.2 ms.
   ⇒ ★ 78% of the time was not the database.
```

**Simulate a cross-AZ connection.**
```bash
# ★ add 1 ms of latency to the loopback
sudo tc qdisc add dev lo root netem delay 1ms
```
```
 ★ n+1:     36.882 ms      (31 × ~1.2 ms)
 ★ batched:  2.412 ms      ★ 15.3×
```
```bash
sudo tc qdisc del dev lo root netem
```
```
 ★ THE SAME CODE. THE SAME DATABASE. ★ 15× INSTEAD OF 5.9×,
   PURELY BECAUSE THE WALK GOT LONGER.
```

**The nested N×M case.**
```js
// ✗ 1 + 30 + (30 × ~6) = ★ 211 round trips
const orders = (await pool.query('SELECT * FROM orders LIMIT 30')).rows;
for (const o of orders) {
  o.items = (await pool.query(
    'SELECT * FROM order_items WHERE order_id=$1', [o.id])).rows;
  for (const i of o.items) {
    i.product = (await pool.query(
      'SELECT * FROM products WHERE sku=$1', [i.sku])).rows[0];
  }
}
```
```
 ★ 34.204 ms localhost · ★ 253 ms with 1 ms latency
```
```js
// ✓ 3 round trips, regardless of N or M
const orders = (await pool.query('SELECT * FROM orders LIMIT 30')).rows;
const items = (await pool.query(
  'SELECT * FROM order_items WHERE order_id = ANY($1)',
  [orders.map(o => o.id)])).rows;
const skus = [...new Set(items.map(i => i.sku))];
const products = (await pool.query(
  'SELECT * FROM products WHERE sku = ANY($1)', [skus])).rows;

const bySku = new Map(products.map(p => [p.sku, p]));
const byOrder = new Map(orders.map(o => [o.id, (o.items = [])]));
for (const i of items) { i.product = bySku.get(i.sku); byOrder.get(i.order_id).push(i); }
```
```
 ★ 1.104 ms localhost · ★ 3.4 ms with 1 ms latency
   ⇒ ★ 74× at cross-AZ latency.
```

**The single-query version.**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.total_minor, to_jsonb(c.*) AS customer,
       coalesce((SELECT json_agg(json_build_object('sku',i.sku,'qty',i.qty)
                                 ORDER BY i.id)
                   FROM order_items i WHERE i.order_id = o.id), '[]') AS items
  FROM orders o JOIN customers c ON c.id = o.customer_id
 ORDER BY o.created_at DESC LIMIT 30;
```
```
 Limit  (actual time=0.284..1.882 rows=30)
   ->  Nested Loop  (actual rows=30)
         ->  Index Scan Backward using orders_created_at_idx
         ->  Index Scan using customers_pkey (loops=30)
         SubPlan 1
           ->  Aggregate (loops=30)
                 ->  Index Scan using order_items_order_id_id_idx (loops=30)
 Execution Time: ★ 1.942 ms
   ★ 30 subplan executions — but ★ ALL INSIDE ONE ROUND TRIP.
     That is the whole point.
```

**Prove row multiplication with a naive join.**
```sql
SELECT count(*) FROM (
  SELECT o.*, i.* FROM orders o JOIN order_items i ON i.order_id = o.id
   ORDER BY o.created_at DESC LIMIT 30) x;
-- ★ LIMIT 30 now limits ROWS, not ORDERS — a subtle correctness bug too
```
```sql
-- ★ and the byte cost, on a wide parent
ALTER TABLE orders ADD COLUMN metadata jsonb;
UPDATE orders SET metadata = jsonb_build_object('pad', repeat('x', 4000))
 WHERE id IN (SELECT id FROM orders ORDER BY created_at DESC LIMIT 30);

EXPLAIN (ANALYZE)
SELECT o.*, i.* FROM orders o JOIN order_items i ON i.order_id = o.id
 WHERE o.id IN (SELECT id FROM orders ORDER BY created_at DESC LIMIT 30);
```
```
 (actual rows=★ 187)
   ★ 30 orders × ~6 items = 187 rows, each carrying the 4 KB
     metadata column ⇒ ★ ~750 KB transferred instead of ~120 KB.
   ⇒ ★ THIS IS WHY "just use a JOIN" IS NOT ALWAYS RIGHT.
```

**The write-side N+1.**
```js
const rows = Array.from({length: 500}, (_, i) => [i, `sku-${i}`, i]);

console.time('loop');
for (const r of rows)
  await pool.query('INSERT INTO events (a,b,c) VALUES ($1,$2,$3)', r);
console.timeEnd('loop');
```
```
 ★ loop: 4,882.4 ms
```
```js
console.time('txn');
await withTransaction(async (tx) => {
  for (const r of rows)
    await tx.query('INSERT INTO events (a,b,c) VALUES ($1,$2,$3)', r);
});
console.timeEnd('txn');
```
```
 ★ txn: 412.1 ms        ★ 11.8× — from BEGIN/COMMIT alone
```
```js
console.time('unnest');
await pool.query(
  `INSERT INTO events (a,b,c)
   SELECT * FROM unnest($1::int[], $2::text[], $3::int[])`,
  [rows.map(r => r[0]), rows.map(r => r[1]), rows.map(r => r[2])]);
console.timeEnd('unnest');
```
```
 ★ unnest: 18.2 ms       ★ 271×
```

**And `COPY` at volume.**
```js
const { from as copyFrom } = require('pg-copy-streams');
console.time('copy');
const client = await pool.connect();
const stream = client.query(copyFrom('COPY events (a,b,c) FROM STDIN WITH (FORMAT csv)'));
for (const r of bigRows) stream.write(`${r[0]},${r[1]},${r[2]}\n`);
stream.end();
console.timeEnd('copy');
```
```
 ★ 500,000 rows — copy: 2,104 ms · multi-row INSERT: ★ 41,882 ms
```

**Instrument per-request query counts.**
```js
app.get('/api/orders', async (req, res) => { /* … */ });
// with the AsyncLocalStorage middleware from above:
```
```
 ★ { route: '/api/orders', queries: 31, dbMs: 1.2 } possible N+1
 ★ after the fix: { route: '/api/orders', queries: 3, dbMs: 1.1 }
   ⇒ ★ dbMs barely changed. The QUERY COUNT is the signal.
```

---

## Example 2 — production scenario

**The situation.** A B2B invoicing platform. The invoice-detail endpoint has degraded steadily for six months.

```
 GET /api/invoices/:id
   p50                     ★ 180 ms
   p99                     ★ 4,100 ms
   rps                     ★ 340
   database CPU            ★ 6%
   slow query log (>100ms) ★ EMPTY
   the team's conclusion   ★ "the database can't keep up —
                             we need read replicas"
```

**Step 1 — the ratio names it in thirty seconds.**

```sql
SELECT sum(calls) AS total_queries FROM pg_stat_statements;
```
```
 total_queries
---------------
 ★ 4,102,884,201
```
```
 ★ over the same window the load balancer recorded ★ 8,842,119
   HTTP requests.
 ⇒ ★ 4,102,884,201 / 8,842,119 = ★ 464 QUERIES PER REQUEST.
 ⇒ ★ HEALTHY IS 3–15. THIS IS NOT A CAPACITY PROBLEM.
```

**Step 2 — sort by `calls`, not by time.**

```sql
SELECT calls, mean_exec_time::numeric(10,4) AS mean_ms,
       (calls*mean_exec_time/1000/3600)::numeric(10,1) AS hours,
       substring(query from 1 for 60) AS q
  FROM pg_stat_statements ORDER BY calls DESC LIMIT 6;
```
```
    calls     | mean_ms | hours |                      q
--------------+---------+-------+----------------------------------------
 ★ 1,884,201,188 |  0.031 | ★ 16.2 | SELECT * FROM tax_rates WHERE id = $1
 ★   884,201,188 |  0.028 |  ★ 6.9 | SELECT * FROM products WHERE id = $1
 ★   412,088,420 |  0.034 |  ★ 3.9 | SELECT * FROM users WHERE id = $1
      88,421,188 |  0.041 |    1.0 | SELECT * FROM invoice_lines WHERE …
       8,842,119 |  2.104 |    5.2 | SELECT * FROM invoices WHERE id = $1
```
```
 ★ THE STRUCTURE IS VISIBLE IN THE RATIOS:
   8,842,119 invoices           = 1 per request
   88,421,188 invoice_lines     = ★ 10 per request
   884,201,188 products         = ★ 100 per request  (10 lines × ?)
   1,884,201,188 tax_rates      = ★ 213 per request
 ⇒ ★ A THREE-LEVEL NESTED N+1, and the deepest level is the worst.
```

**Step 3 — find it. It is not in the controller.**

```js
// src/controllers/invoices.js — ★ this looks completely fine
app.get('/api/invoices/:id', async (req, res) => {
  const invoice = await Invoice.findByPk(req.params.id, {
    include: [{ model: InvoiceLine }],      // ★ eager-loaded!
  });
  res.json(invoice);                        // ★ ← THE BUG IS HERE
});
```
```js
// src/models/InvoiceLine.js
class InvoiceLine extends Model {
  toJSON() {
    return {
      ...this.get(),
      product: this.product,        // ★ LAZY — issues a query
      taxRate: this.taxRate,        // ★ LAZY — issues a query
      taxBreakdown: this.computeTax(),
    };
  }
  computeTax() {
    // ★ AND THIS LOOPS OVER JURISDICTIONS, EACH LOADING A tax_rate
    return this.jurisdictions.map(j => ({          // ★ LAZY
      code: j.code,
      rate: TaxRate.findByPk(j.taxRateId),         // ★ LAZY, IN A LOOP
    }));
  }
}
```
```
 ★ THE N+1 IS IN toJSON(). It is triggered by res.json() — a line
   that looks like it does no I/O at all.
 ⇒ ★ 10 lines × (1 product + 1 taxRate + ~21 jurisdictions)
   = ★ 230 queries, from one res.json() call.
 ⇒ ★ THE `include` IN THE CONTROLLER WAS A PREVIOUS ATTEMPT TO
   FIX THIS. It fixed one level of three.
```

**Step 4 — the second finding: it also exhausts the pool.**

```sql
SELECT state, wait_event, count(*) FROM pg_stat_activity
 WHERE backend_type='client backend' GROUP BY 1,2 ORDER BY 3 DESC;
```
```
 state  | wait_event | count
--------+------------+-------
 idle   | ClientRead | ★ 418
 active |            |    12
```
```
 ★ LITTLE'S LAW (Topic 65):
   340 rps × 464 round trips × 0.35 ms = ★ 55 seconds of
   connection-time per second
 ⇒ ★ 55 connections minimum, and far more at p99.
 ⇒ ★ THE TEAM HAD ALREADY RAISED max_connections TWICE.
```

**Step 5 — the fix, level by level.**

```js
// ★ ① STOP LAZY-LOADING IN toJSON. Serialise explicitly.
class InvoiceLine extends Model {
  toJSON() {
    return {
      id: this.id, sku: this.sku, qty: this.qty,
      unit_price_minor: this.unit_price_minor,
      // ★ only what was EXPLICITLY loaded. No property access
      //   may trigger I/O.
      product: this.dataValues.product ?? null,
      taxRate: this.dataValues.taxRate ?? null,
      taxBreakdown: this.dataValues.taxBreakdown ?? [],
    };
  }
}
```

```js
// ★ ② ONE QUERY THAT BUILDS THE WHOLE TREE
const INVOICE_DETAIL_SQL = `
SELECT
  i.id, i.number, i.issued_at, i.total_minor, i.status,
  to_jsonb(cu.*) - 'password_hash' AS customer,
  coalesce((
    SELECT json_agg(json_build_object(
      'id',           l.id,
      'sku',          l.sku,
      'qty',          l.qty,
      'unit_price_minor', l.unit_price_minor,
      'product',      to_jsonb(p.*),
      'tax_rate',     to_jsonb(tr.*),
      'jurisdictions', coalesce(jt.js, '[]'::json)
    ) ORDER BY l.id)
    FROM invoice_lines l
    LEFT JOIN products  p  ON p.id  = l.product_id
    LEFT JOIN tax_rates tr ON tr.id = l.tax_rate_id
    LEFT JOIN LATERAL (
      SELECT json_agg(json_build_object(
               'code', j.code, 'rate', to_jsonb(jr.*)) ORDER BY j.code) AS js
        FROM line_jurisdictions j
        LEFT JOIN tax_rates jr ON jr.id = j.tax_rate_id
       WHERE j.line_id = l.id
    ) jt ON true
    WHERE l.invoice_id = i.id
  ), '[]'::json) AS lines
FROM invoices i
JOIN customers cu ON cu.id = i.customer_id
WHERE i.id = $1`;

app.get('/api/invoices/:id', async (req, res) => {
  const { rows } = await pool.query(INVOICE_DETAIL_SQL, [req.params.id]);
  if (!rows.length) return res.status(404).end();
  res.json(rows[0]);
});
// ★ ONE ROUND TRIP. 464 → 1.
```

```sql
-- ★ ③ the indexes the new query needs
CREATE INDEX CONCURRENTLY ON invoice_lines (invoice_id, id);
CREATE INDEX CONCURRENTLY ON line_jurisdictions (line_id, code);
-- ★ tax_rates and products are small and cache-resident;
--   their PK indexes suffice.
```

```sql
-- ★ ④ AND THE OBSERVATION THAT SAVED MOST OF THE REMAINING WORK
SELECT count(*) FROM tax_rates;
```
```
 count
-------
 ★ 412        — 412 rows, queried 1.88 BILLION times.
```
```js
// ★ a tiny, slowly-changing reference table ⇒ load it once per
//   process and refresh on a version bump. NOT a Redis cache —
//   just a Map. (Topic 57's layer ④.)
let taxRates = new Map(), taxRatesVersion = 0;
async function refreshTaxRates() {
  const { rows } = await pool.query(
    'SELECT * FROM tax_rates ORDER BY id');
  taxRates = new Map(rows.map(r => [r.id, r]));
}
setInterval(refreshTaxRates, 60_000);
await refreshTaxRates();
// ⇒ ★ even without the SQL rewrite, this alone removes
//   1.88 billion queries.
```

**Step 6 — make it impossible to regress.**

```js
// ★ ① a test that asserts query count is CONSTANT
describe('GET /api/invoices/:id', () => {
  it('issues a bounded number of queries', async () => {
    const c = await countQueries(() => get('/api/invoices/1'));
    expect(c).toBeLessThanOrEqual(2);
  });

  // ★ THE ASSERTION THAT ACTUALLY CATCHES N+1
  it('query count does not grow with the number of lines', async () => {
    const small = await countQueries(() => get(`/api/invoices/${SMALL_ID}`));  // 2 lines
    const large = await countQueries(() => get(`/api/invoices/${LARGE_ID}`));  // 200 lines
    expect(large).toBe(small);      // ★ CONSTANT, not "small enough"
  });
});
```

```js
// ★ ② a global runtime guard — logs, and in CI, throws
const MAX_QUERIES_PER_REQUEST = 25;
app.use((req, res, next) => {
  res.on('finish', () => {
    const { queries } = als.getStore() ?? {};
    if (queries > MAX_QUERIES_PER_REQUEST) {
      metrics.increment('http.n_plus_one_suspected', { route: req.route?.path });
      if (process.env.NODE_ENV === 'test')
        throw new Error(`★ N+1: ${queries} queries on ${req.route?.path}`);
      log.warn({ route: req.route?.path, queries }, '★ possible N+1');
    }
  });
  next();
});
```

```js
// ★ ③ BAN LAZY LOADING AT THE ORM LEVEL — the structural fix
//    Sequelize: no implicit lazy getters
Model.init(attrs, { sequelize, modelName, ★ getterMethods: {} });
//    TypeORM: relations must be explicitly requested
@Entity() class InvoiceLine {
  @ManyToOne(() => Product, { ★ lazy: false, eager: false })
  product: Product;   // ★ undefined unless explicitly joined
}
//    Django: raise on deferred field access
//      from django.db import connection; assertNumQueries(2)
//    Rails: config.active_record.strict_loading_by_default = true
// ⇒ ★ THIS IS THE ONLY FIX THAT SURVIVES A NEW ENGINEER.
```

**Step 7 — results.**

| | Before | After |
|---|---|---|
| Queries per request | ★ **464** | **1** |
| p50 | 180 ms | **8 ms** |
| p99 | ★ **4,100 ms** | **41 ms** (**100×**) |
| Total queries/day | 4.1 billion | ★ **8.8 million** (**466×**) |
| Database CPU | 6% (idle, waiting) | 11% (**doing work**) |
| PostgreSQL backends | 430 | ★ **14** |
| `max_connections` | 600 | **200** |
| Read replicas added | *(planned: 3)* | ★ **0** |

```
 ★ SIX LESSONS:
 ① ★ THE RATIO NAMED IT IN THIRTY SECONDS. 464 queries per
   request. No profiling, no tracing — two numbers.
 ② ★ SORTING pg_stat_statements BY calls SHOWED THE STRUCTURE.
   The ratios between call counts (1 : 10 : 100 : 213) mapped
   directly onto the three levels of nesting.
 ③ ★ THE BUG WAS IN res.json(). A line that looks like it does
   no I/O triggered 230 queries through toJSON().
 ④ ★ A PREVIOUS FIX HAD MADE IT LOOK SOLVED. The controller's
   `include` eager-loaded one level of three.
 ⑤ ★ 412 ROWS WERE QUERIED 1.88 BILLION TIMES. A tiny reference
   table belongs in process memory, not in a query.
 ⑥ ★ THE TEAM WAS ABOUT TO ADD THREE READ REPLICAS to fix an
   application bug. The database CPU was 6%.
```

---

## Common mistakes

**1. Diagnosing it as a database capacity problem.**
- *Symptom:* proposals for read replicas, bigger instances, or denormalisation while CPU sits at 6%.
- *Fix:* compute queries-per-request. Healthy is 3–15.

**2. Sorting `pg_stat_statements` by total time only.**
- *Symptom:* the N+1 query is invisible because each execution is 0.03 ms.
- *Fix:* sort by `calls`. The ratios between call counts reveal the nesting structure.

**3. Assuming the slow-query log would catch it.**
- *Symptom:* an empty log and a 4-second p99.
- *Fix:* no individual query is slow. Instrument per-request query counts instead.

**4. Fixing one level of a nested N+1.**
- *Symptom:* adding `include`/`join` improves things and the problem persists.
- *Fix:* count queries per request before and after. If it's still >25, there's another level.

**5. Lazy loading in a serialiser.**
- *Symptom:* the controller looks correct; `res.json()` issues hundreds of queries.
- *Fix:* explicit serialisation that only reads already-loaded data.

**6. "Just use a JOIN" on one-to-many with wide parents.**
- *Symptom:* row multiplication transfers 6× the bytes, and `LIMIT` now limits rows rather than parents.
- *Fix:* two queries with an in-memory map, or `json_agg` with a correlated subquery.

**7. Not deduplicating ids before the batch query.**
- *Symptom:* `WHERE id = ANY($1)` with 500 ids of which 30 are distinct.
- *Fix:* `[...new Set(ids)]`.

**8. Write-side loops.**
- *Symptom:* a 500-row import takes 5 seconds.
- *Fix:* one transaction (11.8×), then `unnest` (271×), then `COPY` for thousands.

**9. Fixing it once instead of preventing it.**
- *Symptom:* it returns six months later on a different endpoint.
- *Fix:* a test asserting query count is **constant** as page size varies, plus a runtime guard, plus banning lazy loading at the ORM level.

**10. Querying a tiny reference table millions of times.**
- *Symptom:* 412 rows fetched 1.88 billion times.
- *Fix:* load it into process memory with a periodic refresh.

**11. Testing with three rows.**
- *Symptom:* everything passes locally and the endpoint is unusable in production.
- *Fix:* assert the query *count*, not the elapsed time — the count is the invariant that survives environment differences.

---

## Hands-on proof

**PROVE IT #1–#9 — Example 1** (N+1 at 5.2 ms vs batched at 0.88 ms, `pg_stat_statements` showing only 1.16 ms of actual work, `tc netem` turning 5.9× into 15.3×, the nested N×M case at 211 round trips, `json_agg` doing 30 subplans in one round trip, row multiplication transferring 750 KB instead of 120 KB, and the write-side ladder at 4,882 → 412 → 18 ms).

**PROVE IT #10 — the ratio, on your own system.**
```sql
SELECT sum(calls) AS queries FROM pg_stat_statements;
```
```bash
# ÷ requests over the same window
curl -s localhost:9090/api/v1/query \
  --data-urlencode 'query=sum(increase(http_requests_total[24h]))'
```
```
 ★ queries / requests
   3–15   healthy
   50+    ★ an N+1 exists
   400+   ★ nested N+1
```

**PROVE IT #11 — the call-count ratios reveal the structure.**
```sql
SELECT calls,
       calls / (SELECT calls FROM pg_stat_statements
                 WHERE query LIKE '%FROM invoices WHERE id%') AS per_request,
       substring(query, 1, 50)
  FROM pg_stat_statements ORDER BY calls DESC LIMIT 5;
```
```
    calls     | per_request |            substring
--------------+-------------+-------------------------------
 1,884,201,188|       ★ 213 | SELECT * FROM tax_rates WHERE
   884,201,188|       ★ 100 | SELECT * FROM products WHERE i
    88,421,188|        ★ 10 | SELECT * FROM invoice_lines WH
     8,842,119|           1 | SELECT * FROM invoices WHERE i
   ★ 1 → 10 → 100 → 213 is the shape of the nesting, read directly.
```

**PROVE IT #12 — the constant-query-count test.**
```js
const a = await countQueries(() => get('/api/orders?limit=10'));
const b = await countQueries(() => get('/api/orders?limit=200'));
console.log({ a, b });
```
```
 ★ before: { a: 11, b: 201 }     — grows with N ⇒ N+1
 ★ after:  { a: 3,  b: 3 }       — CONSTANT ⇒ fixed
```

**PROVE IT #13 — `LIMIT` with a naive join is also a correctness bug.**
```sql
SELECT count(DISTINCT o.id) FROM (
  SELECT o.id FROM orders o JOIN order_items i ON i.order_id=o.id
   ORDER BY o.created_at DESC LIMIT 30) o;
```
```
 count
-------
   ★ 6        — you asked for 30 orders and got 6.
   ⇒ ★ LIMIT applies to the joined rows, not the parents.
```

---

## The design decision framework

```
★★★ N+1 IS LATENCY × COUNT. NO DATABASE CHANGE FIXES IT. ★★★

 ① ★ DETECT IT WITH THE RATIO, NOT WITH PROFILING
    queries_per_request = sum(pg_stat_statements.calls) ÷ requests
    ⇒ ★ 3–15 healthy · 50+ an N+1 · 400+ nested
    ⇒ ★ then sort pg_stat_statements BY CALLS. The ratios between
      call counts give you the nesting structure directly.

 ② ★ KNOW WHY EVERY OTHER SIGNAL IS SILENT
    slow-query log empty · CPU idle · EXPLAIN perfect
    ⇒ ★ each query IS fast. There are just N of them, serially.

 ③ CHOOSE THE FIX BY RELATIONSHIP SHAPE
    MANY-TO-ONE (a parent per child)
      ⇒ a JOIN, or ★ two queries + a Map
    ONE-TO-MANY with narrow parents
      ⇒ a JOIN is fine
    ★ ONE-TO-MANY with WIDE parents
      ⇒ ★ two queries + a Map, or json_agg with a CORRELATED
        SUBQUERY — never a naive join (row multiplication, and
        ★ LIMIT breaks)
    NESTED / TREE
      ⇒ ★ json_agg + LATERAL, one round trip
    "top K per parent"
      ⇒ ★ LATERAL with a per-parent LIMIT — the only single-query
        way

 ④ ★ WRITE SIDE — THE LADDER
    loop with autocommit        ★ 4,882 ms
    → one transaction           ★ 412 ms   (11.8×, two lines)
    → multi-row / unnest        ★ 18 ms    (271×)
    → COPY (thousands of rows)  ★ 2.1 s for 500k

 ⑤ ★ SOMETIMES N+1 IS FINE — MEASURE BOTH SIDES
    N small and bounded, local connection
    children cached in process (a DataLoader / a Map)
    ★ the JOIN would transfer more bytes than the trips cost
    ⇒ ★ N × round_trip vs bytes + CPU. Five minutes to check.

 ⑥ ★ REMEMBER THE SECOND-ORDER COST
    N+1 holds a pooled connection for N round trips.
    ⇒ Little's Law (Topic 65): ★ it exhausts pools, and gets
      diagnosed as "we need a bigger pool".

 ⑦ ★ PREVENT IT STRUCTURALLY — three layers
    ① ★ a test asserting query count is CONSTANT as page size
       varies — not "small enough", CONSTANT
    ② ★ a runtime guard: > 25 queries per request ⇒ warn in
       production, THROW in tests
    ③ ★ BAN LAZY LOADING AT THE ORM LEVEL
       strict_loading_by_default · eager:false · no getterMethods
       ⇒ ★ the only fix that survives a new engineer

 ⑧ ★ AND CHECK FOR TINY TABLES QUERIED MILLIONS OF TIMES
    412 rows × 1.88 billion queries ⇒ ★ a process-local Map with
    a periodic refresh. Not Redis — a Map (Topic 57, layer ④).
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build a 30-parent / 6-children dataset. Implement and time: (a) the N+1 loop; (b) two queries with a `Map`; (c) one query with `json_agg`. Then add 1 ms of loopback latency with `tc netem` and re-run all three. Explain why the *ratio* between them changes.

### Exercise 2 — medium (apply it)
Implement the write-side ladder: 500 inserts in a loop, in one transaction, as a multi-row `INSERT`, and via `COPY`. Report all four timings and explain each improvement mechanically (round trips vs fsyncs).

Then demonstrate row multiplication: join 30 wide parents to their children and measure the bytes transferred versus the two-query approach. Also show that `LIMIT 30` on the joined query returns fewer than 30 parents.

### Exercise 3 — hard (production simulation)
An invoicing endpoint has p99 4,100 ms, database CPU 6%, and an empty slow-query log. The team is proposing three read replicas.

(a) Compute queries-per-request from `pg_stat_statements` and the load balancer. What does 464 tell you immediately?
(b) Sort by `calls` and derive the nesting structure from the ratios between call counts.
(c) The controller eager-loads its relation and still exhibits the problem. Find where the remaining queries come from and explain why `res.json()` is the trigger.
(d) A previous fix made the problem *look* solved. Explain what it fixed and what it missed.
(e) Write the single-query version producing the full nested structure, including "jurisdictions per line". Explain why a correlated subquery beats `GROUP BY` here.
(f) 412 rows were queried 1.88 billion times. Give the fix and explain why it isn't Redis.
(g) Apply Little's Law to show why this also exhausted the connection pool, and why raising `max_connections` twice didn't help.
(h) Write the three prevention layers. Explain why the constant-query-count assertion is stronger than a threshold.
(i) The team was about to add three read replicas. Write the one-paragraph explanation you'd give them.

---

## Mental model checkpoint

1. Why is N+1 a latency problem rather than a work problem? Give the percentage breakdown of a local round trip.
2. Why does no database-side change fix it?
3. Which `pg_stat_statements` column reveals it, and why is the usual sort order useless?
4. What is the queries-per-request ratio for a healthy endpoint, and for one with a nested N+1?
5. Name the four shapes. Which hides in code that looks like it does no I/O?
6. When is a `JOIN` the wrong fix, and what two problems does it cause?
7. Give the write-side ladder with the mechanism behind each improvement.
8. Name three situations where N+1 is acceptable.
9. How does N+1 exhaust a connection pool? Give the arithmetic.
10. What is the strongest possible regression test, and why is it stronger than a threshold?
11. Why does banning lazy loading at the ORM level matter more than any individual fix?

---

## Quick reference card

**★ Detect**
```sql
SELECT calls, mean_exec_time, calls*mean_exec_time/1000 AS total_s,
       substring(query,1,60)
  FROM pg_stat_statements ORDER BY ★ calls DESC LIMIT 10;
SELECT sum(calls) FROM pg_stat_statements;   -- ÷ requests
```
**★ Ratio:** 3–15 healthy · 50+ N+1 · 400+ nested. **Call-count ratios give you the nesting depth.**

**Fix by shape**

| Shape | Fix |
|---|---|
| many-to-one | JOIN, or ★ 2 queries + `Map` |
| one-to-many, wide parents | ★ 2 queries + `Map`, or `json_agg` correlated subquery |
| nested tree | ★ `json_agg` + `LATERAL`, one round trip |
| top-K per parent | ★ `LEFT JOIN LATERAL (… LIMIT k) ON true` |
| **write side** | txn (**11.8×**) → `unnest` (**271×**) → `COPY` |

```js
const ids = [...new Set(rows.map(r => r.fk))];        // ★ dedup
const loaded = await q('SELECT * FROM t WHERE id = ANY($1)', [ids]);
const byId = new Map(loaded.rows.map(x => [x.id, x]));
rows.forEach(r => r.rel = byId.get(r.fk));            // ★ 2 trips, any N
```

**★ Prevent — three layers**
```js
expect(await countQueries(get('?limit=200')))
  .toBe(await countQueries(get('?limit=10')));   // ★ CONSTANT
if (queries > 25) throw new Error('N+1');         // runtime guard
// ★ ORM: strict_loading_by_default / eager:false / no lazy getters
```

**★ Round trip:** 0.15 ms local · 0.5 same-DC · 1.2 cross-AZ · **40 ms cross-region**. **The query is 0.02 ms.**
**★ It also exhausts your connection pool** (Topic 65).

---

## When would I use this at work?

1. **Any endpoint that is slow while the database is idle.** That combination is N+1 until proven otherwise. Two numbers — total queries and total requests — settle it in thirty seconds, before anyone opens a profiler.

2. **Before agreeing to add read replicas or denormalise.** Both are expensive and permanent; both are routinely proposed to fix an application bug. Topic 54's gate 2 exists largely because of this.

3. **Reviewing any code that serialises an ORM model.** `res.json(model)` is where N+1 hides most often, because the line contains no visible I/O. Explicit serialisers that read only already-loaded data eliminate the entire class.

4. **Setting up a new service.** Banning lazy loading at the ORM level and adding the constant-query-count assertion costs an hour at the start and prevents the problem permanently. Fixing it later, endpoint by endpoint, never finishes.

---

## Connected topics

**Understand before this:** 18 (EXPLAIN — why the plan looks perfect), 53/54 (denormalisation — the wrong fix, and gate 2), 65 (pooling — the second-order damage), 41 (why 500 commits mean 500 fsyncs).

**This unlocks:**
- **67** — performance investigation: the systematic method this is one branch of
- **57** — caching: the process-local reference-table cache
- **70** — document modelling: where nesting is the storage shape, not a query pattern
