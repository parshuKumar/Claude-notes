# 78 — Capstone II: Scale It Under Pressure
## Phase: Capstones

---

## ELI5 — The Simple Analogy

A restaurant that works beautifully at forty covers a night.

Then a review lands and it's four hundred. **Nothing about the food changed.** But the kitchen door becomes a bottleneck, the one dishwasher can't keep up, the till queue reaches the street, and the chef who could plate ten dishes at once now can't remember which order is which.

★ **Here is the thing nobody expects: the parts that break are not the parts that were slowest.** The slow oven was always slow, and at 40 covers it was fine, and at 400 it is *still* the same speed — it just runs continuously now. **What breaks is the narrow doorway that nobody had ever noticed, because at 40 covers two people never wanted to pass through it at the same time.**

★ **Scaling is not "make everything faster." It is discovering, in order, which single thing is currently the constraint — fixing exactly that — and then discovering what the *next* one is.** The order is not predictable from first principles, and it changes every time you fix something.

And the failure that ruins the exercise: ★ **fixing three things at once, so you never learn which one mattered.**

---

## Where this fits in the big picture

```
   77 — ★ the schema, designed and defended
   67 — ★ the investigation method
   54, 57–61, 65–66 — ★ every individual lever
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 78 CAPSTONE II: SCALE IT ← YOU ARE HERE      │
        │ ★ the same system at 1×, 10×, 100×, 1000×    │
        └────────────────────┬─────────────────────────┘
                             ▼
              79 the principal-engineer review
```

★ **Topic 67 gave you the method for one incident.** This topic runs that method **repeatedly, at increasing load, on the same system** — so you can see **the order in which things break**, which is the thing experience actually teaches and a curriculum usually cannot.

---

## What is this?

Taking one system and increasing load by orders of magnitude, **finding the single binding constraint at each stage**, and fixing exactly that.

```
 ★ THE METHOD — AND STEP ④ IS THE ONE PEOPLE SKIP:

 ① ★ ESTABLISH A BASELINE — ★ measured, not assumed
 ② ★ INCREASE LOAD until something bends
 ③ ★ FIND THE SINGLE BINDING CONSTRAINT (Topic 67)
 ④ ★ FIX EXACTLY ONE THING
 ⑤ ★ RE-MEASURE — ★ confirm the constraint MOVED
 ⑥ ★ REPEAT

 ⇒ ★ THE DISCIPLINE IS ④ AND ⑤.
   ★ Fix three things and you have learned nothing, you cannot
   revert safely, and ★ you will apply the wrong lesson next time.
```

★ **And the observation that organises the whole topic:** *the constraint moves through **layers**, in a rough but real order — **plan → index → I/O → contention → connections → single-machine limits → distribution**. Each fix pushes it one layer down. **The layers are not skippable**: you cannot fix a contention problem with more hardware, and you cannot fix a plan problem with more connections.*

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE INSTINCT AT EVERY STAGE IS WRONG IN THE SAME WAY.

 ★ THE UNIVERSAL FIRST INSTINCT: ★ "add capacity."
   ⇒ ★ it works for exactly ONE of the seven layers (I/O), and
     ★ makes two of them WORSE (contention, queueing).

 ★ THE SECOND INSTINCT: ★ "add a cache."
   ⇒ ★ it hides the constraint rather than moving it (57), ★ and
     creates a load-bearing component that turns a 90-second
     incident into a 41-minute one.

 ★ THE THIRD: ★ "shard it."
   ⇒ ★ Topic 60's checklist exists because this is almost always
     four layers too early.

 ⇒ ★ AND THE ASYMMETRY THAT MATTERS MOST:
   ★ EACH LAYER'S FIX IS 10–1000× CHEAPER THAN THE NEXT LAYER'S.
   ★ An index is an afternoon. Sharding is two quarters.
   ⇒ ★ SO THE ORDER IS NOT A PREFERENCE — ★ IT IS ECONOMICS.
```

---

## The physical reality

### The seven layers, and how to recognise each

```
 ★ LAYER 1 — THE PLAN
   ★ SYMPTOM: one query slow; ★ EXPLAIN shows a huge estimate
     error or a full scan
   ★ SIGNAL: ★ `Rows Removed by Filter` in the millions;
     ★ estimated vs actual off by >100×
   ★ FIX: ★ an index · `ANALYZE` · `CREATE STATISTICS` · a rewrite
   ★ COST: ★ hours.  ★ TYPICAL GAIN: ★ 100–10,000×

 ★ LAYER 2 — REDUNDANT WORK
   ★ SYMPTOM: the database is idle; the endpoint is slow
   ★ SIGNAL: ★ `pg_stat_statements` sorted by `calls`;
     ★ queries-per-request > 25 (66)
   ★ FIX: ★ batch the N+1 · `COPY` · pipeline
   ★ COST: ★ days.  ★ GAIN: ★ 10–100×

 ★ LAYER 3 — I/O
   ★ SYMPTOM: high `iowait`; `wait_event = DataFileRead`
   ★ SIGNAL: ★ buffer `read` ≫ `hit`; cache hit ratio < 99%
   ★ FIX: ★ indexes (read less) · more RAM · faster disk ·
     ★ partitioning to prune
   ★ COST: ★ hours to days.  ★ GAIN: ★ 5–50×
   ⇒ ★ THE ONLY LAYER WHERE "MORE HARDWARE" IS THE RIGHT ANSWER.

 ★ LAYER 4 — CONTENTION
   ★ SYMPTOM: ★ LOW CPU, ★ failing requests
   ★ SIGNAL: ★ `wait_event = transactionid` (row lock) or
     ★ `BufferContent` (index leaf page)
   ★ FIX: ★ atomic statements · shard the hot row (61) ·
     ★ hot row LAST in the transaction · hash-partition
   ★ COST: ★ days.  ★ GAIN: ★ 5–100×
   ⇒ ★ ADDING CAPACITY DOES NOTHING HERE. ★ Adding concurrency
     makes it worse.

 ★ LAYER 5 — QUEUEING / CONNECTIONS
   ★ SYMPTOM: ★ throughput FLAT, ★ latency LINEAR in concurrency
   ★ SIGNAL: ★ `idle in transaction` count; ★ pool-wait time;
     ★ `avg_xact_time ≫ avg_query_time`
   ★ FIX: ★ reduce HOLD TIME · a pooler in transaction mode (65)
   ★ COST: ★ days.  ★ GAIN: ★ 3–20×
   ⇒ ★ MORE CONNECTIONS MAKE THIS MUCH WORSE.

 ★ LAYER 6 — SINGLE-MACHINE LIMITS
   ★ SYMPTOM: ★ CPU 90%+ doing REAL work; WAL > 100 MB/s;
     replicas cannot keep up
   ★ SIGNAL: ★ vertical scaling exhausted; ★ replay is
     single-threaded (58)
   ★ FIX: ★ read replicas · ★ functional split · ★ rollups (56) ·
     ★ retention via partitioning (59)
   ★ COST: ★ weeks.  ★ GAIN: ★ 2–10×

 ★ LAYER 7 — DISTRIBUTION
   ★ SYMPTOM: ★ genuinely beyond one machine, after all of the above
   ★ FIX: ★ sharding (60) or a specialised store (75)
   ★ COST: ★ quarters.  ★ GAIN: ★ linear, ★ at the cost of joins,
     transactions and constraints

 ⇒ ★ AND THE RULE: ★ YOU CANNOT SKIP A LAYER.
   ★ A layer-4 contention problem is not solved by layer-6
   hardware or layer-7 sharding — ★ the hot key lands on one
   shard and you have the same problem, ★ having spent a quarter.
```

### The load-testing discipline

```
 ★ A LOAD TEST THAT DOES NOT RESEMBLE PRODUCTION TEACHES YOU
   NOTHING, AND USUALLY TEACHES YOU SOMETHING FALSE.

 ★ THE FIVE THINGS MOST LOAD TESTS GET WRONG:
 ① ★ UNIFORM KEY DISTRIBUTION
    ⇒ ★ real keys are POWER-LAW. ★ A uniform test never finds the
      hot row, ★ which is layer 4 — often the real constraint.
    ⇒ ★ FIX: replay real key frequencies, or use a Zipf
      distribution.
 ② ★ A COLD OR AN OVER-WARM CACHE
    ⇒ ★ a test that runs 60 seconds on 1 GB of data has a 100%
      cache hit rate and ★ never finds layer 3.
    ⇒ ★ FIX: a dataset larger than RAM, ★ and a warm-up phase.
 ③ ★ NO WRITE MIX
    ⇒ ★ read-only tests never find contention, bloat or WAL limits.
    ⇒ ★ FIX: the real read/write ratio, ★ measured.
 ④ ★ CLOSED-LOOP LOAD (a fixed client count)
    ⇒ ★ when the system slows, the test slows too ⇒ ★ it hides
      queue collapse.
    ⇒ ★ FIX: ★ OPEN-LOOP — a fixed ARRIVAL RATE, regardless of
      response time. ★ This is how production behaves.
 ⑤ ★ TOO SHORT
    ⇒ ★ bloat, checkpoint spikes, autovacuum, replication lag and
      cache eviction all need ★ 30+ minutes to appear.

 ⇒ ★ AND MEASURE THE RIGHT THING:
   ★ NOT average latency. ★ p50, p99, p999, ★ and the ERROR RATE.
   ★ A system at 99% success with p99 of 40 ms is healthier than
   one at 100% with p99 of 4 s — ★ the second is queueing and is
   about to collapse.
```

### Little's Law as the organising equation

```
 ★ L = λ × W        ★ (concurrency = arrival rate × service time)

 ⇒ ★ REARRANGED, THIS IS THE WHOLE TOPIC:
   ★ THROUGHPUT = CONCURRENCY ÷ SERVICE TIME

 ★ SO THERE ARE ONLY TWO WAYS TO INCREASE THROUGHPUT:
   ★ ① REDUCE SERVICE TIME  (layers 1–4)
   ★ ② INCREASE CONCURRENCY (layers 5–7)
 ⇒ ★ AND ② HAS A CEILING — ★ past the throughput peak, more
   concurrency REDUCES throughput (65).

 ★ WORKED, ON ONE QUERY:
   service time 400 ms, 200 connections ⇒ ★ 500 rps
   ★ fix the index ⇒ 0.4 ms ⇒ ★ 500,000 rps (theoretical)
   ★ vs. double the connections ⇒ ★ 1,000 rps, ★ and worse p99
 ⇒ ★ 1,000× vs 2×. ★ THAT IS WHY THE ORDER MATTERS.
```

### What breaks that is *not* the database

```
 ★ AT EACH ORDER OF MAGNITUDE, SOMETHING NON-DATABASE BREAKS.
   ★ Budget for finding it.

   ★ 10×   ⇒ connection pool sizing · ★ JSON serialisation CPU ·
            ★ a missing index that was fine at 1×
   ★ 100×  ⇒ ★ the load balancer's connection limits ·
            ★ log volume (★ often the first disk to fill) ·
            ★ metric cardinality (73) · GC pauses
   ★ 1000× ⇒ ★ DNS lookups · ★ TLS handshake CPU · ★ ephemeral
            port exhaustion · ★ the CI pipeline that deploys it

 ⇒ ★ AND THE ONE THAT SURPRISES EVERYONE:
   ★ AT 100×, YOUR OBSERVABILITY BECOMES THE BOTTLENECK.
   ★ Logging every request at 400,000 rps writes more than the
   database does. ★ Sample it, or it will take the system down
   while you are trying to diagnose the system.
```

---

## How it works — step by step

### The system under test

```
 ★ THE WHOLESALE MARKETPLACE FROM TOPIC 77.
   ★ THE HOT PATH: ★ W1 — place an order.
     ★ idempotency claim + stock decrement + order + items
       + totals + outbox, ★ in one transaction.

 ★ THE PLAN: ★ take it from 30 orders/sec to 30,000, ★ finding the
   constraint at each stage and fixing exactly one thing.
```

### The harness

```javascript
// ★ OPEN-LOOP: a fixed ARRIVAL RATE, not a fixed client count.
//   ★ This is the single most important property of the harness.
import http from 'k6/http';
import { Trend, Rate } from 'k6/metrics';

const latency = new Trend('order_latency');
const errors  = new Rate('order_errors');

export const options = {
  scenarios: {
    ramp: {
      executor: '★ constant-arrival-rate',
      rate: __ENV.RATE, timeUnit: '1s',
      duration: '★ 30m',                 // ★ long enough for bloat
      preAllocatedVUs: 200, maxVUs: 4000,
    },
  },
  thresholds: {
    '★ order_latency{p:99}': ['p(99)<500'],
    '★ order_errors': ['rate<0.001'],
  },
};

// ★ ZIPF-DISTRIBUTED SELLERS — ★ real key skew, not uniform
const sellers = buildZipfTable(5000, 1.1);   // ★ top 3 ≈ 30% of volume
const skus    = buildZipfTable(200000, 1.2);

export default function () {
  const body = JSON.stringify({
    idempotency_key: uuidv4(),
    seller_org_id: zipfPick(sellers),
    lines: [{ product_id: zipfPick(skus), qty: 1 }],
  });
  const t0 = Date.now();
  const r = http.post(`${__ENV.BASE}/orders`, body, {
    headers: { 'Content-Type': 'application/json' } });
  latency.add(Date.now() - t0);
  errors.add(r.status !== 201);
}
```

```bash
# ★ AND THE OBSERVATION SCRIPT — run alongside EVERY test
watch -n2 '
psql -tAc "SELECT coalesce(wait_event_type,'"'"'RUNNING'"'"'),
                  coalesce(wait_event,'"'"'-'"'"'), state, count(*)
             FROM pg_stat_activity WHERE backend_type='"'"'client backend'"'"'
            GROUP BY 1,2,3 ORDER BY 4 DESC LIMIT 6"
mpstat 1 1 | tail -1
psql -tAc "SELECT count(*) FROM pg_stat_activity
            WHERE state='"'"'idle in transaction'"'"'"
'
```

### Stage 1 — 30 → 300 orders/sec

```
 ★ BASELINE, MEASURED FIRST:
   30/sec · ★ p50 8 ms · ★ p99 22 ms · ★ errors 0% · ★ CPU 4%
```
```bash
k6 run -e RATE=300 load.js
```
```
 ★ p50 ★ 412 ms · ★ p99 ★ 4,102 ms · ★ errors ★ 3.2%
 ★ CPU ★ 11%
```
```
 ★ LAYER 2 CHECK FIRST (67 level 1): ★ is it even the database?
   ★ db_ms p99 ★ 88 ms · ★ total p99 4,102 ms
   ⇒ ★ 2% of the time is in queries.
   ★ pool_wait p99 ★ 3,980 ms
   ⇒ ★ LAYER 5. ★ NOT the database at all.
```
```sql
SELECT state, wait_event, count(*) FROM pg_stat_activity
 WHERE backend_type='client backend' GROUP BY 1,2 ORDER BY 3 DESC;
```
```
        state         | wait_event | count
----------------------+------------+-------
 ★ idle in transaction| ClientRead |  ★ 178
 idle                 | ClientRead |    14
 active               |            |     8
 ⇒ ★ 178 BACKENDS HOLDING TRANSACTIONS OPEN.
```
```js
// ★ THE CAUSE, FOUND IN THE HANDLER:
await withTenant(orgId, userId, async (tx) => {
  await claimIdempotency(tx, key);
  await decrementStock(tx, lines);
  const order = await insertOrder(tx, ...);
  ★ await auditService.record(order);      // ★ AN HTTP CALL. 120 ms p50.
  await insertOutbox(tx, order);
});
```
```
 ★ LITTLE'S LAW: ★ 300/sec × 0.15 s hold = ★ 45 connections needed,
   ★ but the p99 of the audit call is 2.8 s ⇒ ★ 840 at p99.
 ⇒ ★ FIX EXACTLY ONE THING: ★ move the audit call out.
   ★ It becomes an outbox event (52).
```
```bash
k6 run -e RATE=300 load.js
```
```
 ★ p50 ★ 11 ms · ★ p99 ★ 41 ms · ★ errors ★ 0% · ★ CPU 14%
 ★ idle in transaction: ★ 2
 ⇒ ★ CONSTRAINT MOVED. ★ 100× on p99, from deleting one line.
```

### Stage 2 — 300 → 3,000 orders/sec

```bash
k6 run -e RATE=3000 load.js
```
```
 ★ p50 ★ 84 ms · ★ p99 ★ 8,204 ms · ★ errors ★ 12.4%
 ★ CPU ★ 22%
```
```
 ★ LOW CPU, HIGH ERRORS ⇒ ★ LAYER 4, CONTENTION (67).
```
```sql
SELECT wait_event_type, wait_event, count(*) FROM pg_stat_activity
 WHERE state='active' GROUP BY 1,2 ORDER BY 3 DESC;
```
```
 wait_event_type |   wait_event    | count
-----------------+-----------------+-------
 Lock            | ★ transactionid |  ★ 412
 LWLock          | ★ BufferContent |   ★ 88
```
```
 ★ TWO DISTINCT CONTENTION POINTS, ★ and they need different fixes.
```
```sql
-- ★ ① find the hot row
SELECT product_id, count(*) FROM order_items
 WHERE created_at > now() - interval '1 minute'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 3;
```
```
 product_id | count
------------+-------
   ★ 88412  | ★ 41,204     -- ★ 23% of all decrements
```
```
 ★ THE ZIPF DISTRIBUTION IN THE HARNESS FOUND THIS.
   ★ A uniform load test would have shown nothing.
```
```js
// ★ FIX ONE THING: ★ move the stock decrement to LAST in the
//   transaction (61). ★ The lock is held until COMMIT either way,
//   ★ but everything before it no longer holds it.
await withTenant(orgId, userId, async (tx) => {
  await claimIdempotency(tx, key);
  const order = await insertOrder(tx, ...);      // ★ moved up
  await insertItems(tx, order, lines);           // ★ moved up
  await computeTotals(tx, order);                // ★ moved up
  await insertOutbox(tx, order);                 // ★ moved up
  ★ await decrementStock(tx, sortedLines);       // ★ LAST
});
```
```bash
k6 run -e RATE=3000 load.js
```
```
 ★ p50 ★ 18 ms · ★ p99 ★ 340 ms · ★ errors ★ 0.4%
 ⇒ ★ 24× on p99, ★ from reordering statements.
 ⇒ ★ AND `transactionid` waits: ★ 412 → 41.
```

### Stage 3 — the second contention point

```
 ★ `BufferContent` IS STILL 88. ★ A DIFFERENT PROBLEM.
```
```sql
-- ★ which index?
SELECT c.relname, count(*) FROM pg_stat_activity a
  JOIN pg_locks l ON l.pid = a.pid
  JOIN pg_class c ON c.oid = l.relation
 WHERE a.wait_event = 'BufferContent' GROUP BY 1 ORDER BY 2 DESC;
```
```
      relname       | count
--------------------+-------
 ★ orders_pkey      |  ★ 71
 ⇒ ★ EVERY INSERT LANDS ON THE SAME RIGHTMOST LEAF PAGE.
   ★ bigserial ⇒ monotonically increasing ⇒ ★ one hot page (61, 73).
```
```
 ★ THE OPTIONS:
 ⒜ ★ UUIDv7 for the PK ⇒ ★ still time-ordered ⇒ ★ SAME PROBLEM
 ⒝ ★ UUIDv4 ⇒ ★ spreads inserts, ★ but destroys B-tree locality
   and cache hit rates (22) ⇒ ★ trades layer 4 for layer 3
 ⒞ ★ HASH-PARTITION `orders` by id ⇒ ★ N rightmost pages
   ⇒ ★ but Topic 77 rejected partitioning `orders` because it
     breaks the `tracking_number` unique constraint (R4)
 ⒟ ★ RAISE fillfactor? ⇒ ★ no; the problem is page CONTENTION,
   not page fullness
```
```
 ★ THE DECISION: ★ ⒞, ★ BUT WITH THE R4 PROBLEM SOLVED SEPARATELY.
   ⇒ ★ `tracking_number` moves to its own small unpartitioned
     lookup table: ★ (tracking_number PK, order_id, placed_at)
   ⇒ ★ R4 becomes a 2-step lookup: ★ 0.08 ms → 0.14 ms
   ⇒ ★ AND `orders` can now be hash-partitioned 16 ways.
 ★ THIS IS THE SHAPE OF REAL SCALING WORK: ★ the fix for one
   constraint requires undoing an earlier decision, ★ and the
   design record (77) is what makes that safe to reason about.
```
```bash
k6 run -e RATE=3000 load.js
```
```
 ★ p50 ★ 12 ms · ★ p99 ★ 88 ms · ★ errors ★ 0%
 ★ BufferContent: ★ 88 → 4
```

### Stage 4 — 3,000 → 10,000 orders/sec

```bash
k6 run -e RATE=10000 load.js
```
```
 ★ p50 ★ 41 ms · ★ p99 ★ 1,204 ms · ★ errors ★ 2.1%
 ★ CPU ★ 88%        ← ★ FINALLY, REAL WORK
```
```
 ★ LAYER 1 OR 3? ★ CHECK THE BUFFERS.
```
```sql
SELECT sum(heap_blks_hit) h, sum(heap_blks_read) r,
       round(100.0*sum(heap_blks_hit)/
             nullif(sum(heap_blks_hit)+sum(heap_blks_read),0),2) AS pct
  FROM pg_statio_user_tables;
```
```
     h      |    r     |  pct
------------+----------+-------
 4102884201 | ★ 88420118 | ★ 97.89
 ⇒ ★ 2.1% MISS RATE ⇒ ★ LAYER 3, partially.
```
```sql
SELECT calls, mean_exec_time::numeric(10,3) AS ms,
       (calls*mean_exec_time/1000)::numeric(10,1) AS total_s,
       substring(query,1,60) FROM pg_stat_statements
 ORDER BY calls*mean_exec_time DESC LIMIT 3;
```
```
  calls   |   ms   | total_s |                substring
----------+--------+---------+----------------------------------
 ★ 18402118|  ★ 0.41 | ★ 7,545 | INSERT INTO order_items (…) SELECT
   8842119|  0.18  |  1,591  | UPDATE stock SET on_hand = …
   8842119|  0.09  |    795  | INSERT INTO orders (…)
```
```
 ★ THE ITEM INSERT DOMINATES. ★ WHY 0.41 ms FOR ONE ROW?
```
```sql
EXPLAIN (ANALYZE, BUFFERS)
INSERT INTO order_items (order_id, product_id, sku_at_order, name_at_order,
                         unit_price_charged_minor, gst_rate_charged, qty)
SELECT 1, p.id, p.sku, p.name, p.list_price_minor, p.gst_rate, 1
  FROM products p WHERE p.id = 88412;
```
```
 Insert on order_items  (actual time=0.402..0.402 rows=0)
   Buffers: shared hit=24 ★ read=6
   ->  Index Scan using products_pkey on products
         Buffers: shared hit=4 ★ read=2
 ⇒ ★ THE JOIN TO `products` FOR THE SNAPSHOT VALUES IS READING
   FROM DISK. ★ 200,000 products × 4 indexes ≫ the buffer pool's
   share.
```
```
 ★ THE FIX, ★ AND THE ONE THING CHANGED:
   ★ `shared_buffers` 8 GB → 48 GB (of 128 GB RAM)
   ⇒ ★ this is layer 3, ★ and it is the ONE layer where "more
     hardware" is correct.
```
```bash
k6 run -e RATE=10000 load.js
```
```
 ★ p50 ★ 9 ms · ★ p99 ★ 84 ms · ★ errors ★ 0%
 ★ cache hit ★ 99.94% · ★ CPU 71%
```

### Stage 5 — 10,000 → 30,000 orders/sec

```bash
k6 run -e RATE=30000 load.js
```
```
 ★ p50 ★ 22 ms · ★ p99 ★ 410 ms · ★ errors ★ 0.8%
 ★ CPU ★ 96% · ★ WAL ★ 184 MB/s · ★ replica lag ★ 41 s
```
```
 ★ THREE THINGS AT ONCE ⇒ ★ LAYER 6.
   ★ AND THE REPLICA LAG IS THE LOUDEST SIGNAL: ★ replay is
   SINGLE-THREADED at ~100 MB/s (42, 58). ★ 184 MB/s of WAL
   cannot be replayed. ★ Lag grows without bound, forever.
```
```sql
-- ★ WHERE IS THE WAL COMING FROM?
SELECT relname, n_tup_ins, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
  FROM pg_stat_user_tables ORDER BY n_tup_ins + n_tup_upd DESC LIMIT 4;
```
```
   relname    | n_tup_ins  | n_tup_upd  | n_tup_hot_upd | hot_pct
--------------+------------+------------+---------------+---------
 order_items  | ★ 88402118 |          0 |             0 |
 ★ stock      |          0 | ★ 44201059 |     ★ 442,010 |  ★ 1.0
 orders       |   44201059 |   44201059 |    ★ 8,840,211|  ★ 20.0
 outbox       |   44201059 |   44201059 |    ★ 441,000  |  ★ 1.0
```
```
 ★ THREE FINDINGS:
 ① ★ `stock` HOT ratio is ★ 1% ⇒ ★ an index on a column that
    changes (46). ★ Which one?
 ② `orders` HOT ratio 20% ⇒ ★ the status update after insert
 ③ `outbox` HOT ratio 1% ⇒ ★ the `published_at` update, ★ and
    the partial index on it
```
```sql
SELECT indexrelname, idx_scan, pg_get_indexdef(indexrelid)
  FROM pg_stat_user_indexes WHERE relname='stock';
```
```
   indexrelname    | idx_scan |            pg_get_indexdef
-------------------+----------+----------------------------------------
 stock_pkey        | 44201059 | … (product_id)
 ★ stock_updated_idx|      ★ 0 | ★ … (updated_at)     ← ★ NEVER SCANNED
 ⇒ ★ AN INDEX ON `updated_at`, ★ which every stock update
   changes, ★ used ZERO times. ★ It destroys HOT updates (46).
```
```sql
DROP INDEX CONCURRENTLY stock_updated_idx;
ALTER SYSTEM SET ★ wal_compression = 'lz4';     -- (41)
SELECT pg_reload_conf();
```
```bash
k6 run -e RATE=30000 load.js
```
```
 ★ stock HOT ratio ★ 1% → ★ 97.4%
 ★ WAL ★ 184 MB/s → ★ 41 MB/s        ★ 4.5×
 ★ replica lag ★ 41 s → ★ 1.2 s
 ★ p99 ★ 410 ms → ★ 94 ms · ★ errors 0%
 ⇒ ★ DROPPING ONE UNUSED INDEX WAS THE LARGEST SINGLE WIN IN
   THE ENTIRE EXERCISE.
```

### Stage 6 — what breaks that is not the database

```bash
k6 run -e RATE=30000 load.js --duration 60m
```
```
 ★ MINUTE 0–20:  p99 94 ms · errors 0%
 ★ MINUTE 24:    ★ p99 2,840 ms · ★ errors 4.1%
 ★ MINUTE 41:    ★ errors 18%
```
```
 ★ THE DATABASE IS FINE. ★ CPU 68%, ★ no lock waits, ★ lag 1.2 s.
 ⇒ ★ SOMETHING ELSE.
```
```bash
df -h /var/log
```
```
 Filesystem  Size  Used Avail Use%
 /dev/nvme1  200G  ★ 198G  1.2G ★ 99%
 ⇒ ★ APPLICATION LOGS. ★ 30,000 requests/sec × ~2 KB of
   structured log = ★ 60 MB/s = ★ 216 GB/hour.
 ⇒ ★ THE OBSERVABILITY BECAME THE BOTTLENECK.
```
```js
// ★ THE FIX: ★ sample, and log the tail rather than everything
const shouldLog = (req, durationMs, status) =>
  ★ status >= 400              ||    // ★ always log errors
  ★ durationMs > 200           ||    // ★ always log the slow tail
  ★ Math.random() < 0.001;           // ★ 0.1% of the rest
```
```
 ★ log volume ★ 60 MB/s → ★ 0.4 MB/s        ★ 150×
 ★ AND THE INFORMATION LOST: ★ almost none — ★ errors and the
   slow tail are exactly what you look at.
```

### The final state

```
 ★ 30,000 orders/sec sustained · ★ p50 9 ms · ★ p99 94 ms ·
 ★ errors 0.01% · ★ CPU 68% · ★ replica lag 1.2 s
 ★ ON ONE MACHINE. ★ 1,000× the baseline.
 ⇒ ★ LAYER 7 (sharding) was never reached.
```

---

## Concept breakdown

```
★ THE METHOD — ★ AND STEP ④ IS THE DISCIPLINE
   baseline → increase load → ★ find the SINGLE constraint →
   ★ FIX EXACTLY ONE THING → ★ re-measure → repeat
   ⇒ ★ fix three things and you learn nothing and cannot revert

★ SEVEN LAYERS, ★ IN ORDER, ★ NOT SKIPPABLE
   ① ★ PLAN        Rows Removed by Filter · estimate error
                   ⇒ index/ANALYZE ⇒ ★ hours ⇒ 100–10,000×
   ② ★ REDUNDANT   calls ≫ · queries-per-request > 25
                   ⇒ batch/COPY ⇒ ★ days ⇒ 10–100×
   ③ ★ I/O         iowait · read ≫ hit
                   ⇒ index/RAM/disk ⇒ ★ THE ONLY LAYER WHERE
                     MORE HARDWARE IS THE ANSWER ⇒ 5–50×
   ④ ★ CONTENTION  ★ LOW CPU + transactionid/BufferContent
                   ⇒ atomic · shard the row · hot row LAST
                   ⇒ ★ capacity does NOTHING ⇒ 5–100×
   ⑤ ★ QUEUEING    ★ flat throughput, linear latency ·
                     idle-in-transaction · pool wait
                   ⇒ ★ REDUCE HOLD TIME ⇒ ★ more connections
                     make it WORSE ⇒ 3–20×
   ⑥ ★ ONE MACHINE CPU 90% real · WAL > 100 MB/s · replica lag
                   ⇒ replicas · functional split · rollups ·
                     retention ⇒ ★ weeks ⇒ 2–10×
   ⑦ ★ DISTRIBUTION ⇒ sharding (60) ⇒ ★ quarters ⇒ linear,
                     ★ at the cost of joins and transactions

★ ★ EACH LAYER'S FIX IS 10–1000× CHEAPER THAN THE NEXT.
  ★ The order is economics, not preference.

★ LITTLE'S LAW ORGANISES IT
   ★ throughput = concurrency ÷ service time
   ⇒ ★ reduce service time (1–4) or increase concurrency (5–7)
   ⇒ ★ and concurrency has a CEILING (65)
   ⇒ ★ 400 ms → 0.4 ms is 1000×; ★ doubling connections is 2×

★ THE LOAD TEST MUST RESEMBLE PRODUCTION
   ★ ZIPF keys (★ a uniform test never finds layer 4) ·
   ★ a dataset > RAM · ★ the real write mix ·
   ★ OPEN-LOOP arrival rate · ★ 30+ minutes
   ⇒ ★ measure p50/p99/p999 AND the ERROR RATE, ★ never the mean

★ AT EACH ORDER OF MAGNITUDE, SOMETHING NON-DATABASE BREAKS
   10× pool sizing · 100× ★ LOG VOLUME, metric cardinality ·
   1000× ports, TLS, DNS
   ⇒ ★ at 100×, OBSERVABILITY BECOMES THE BOTTLENECK
```

---

## Diagrams

**Diagram 1 — big picture: the constraint moving down the layers**

```
  ★ THE SAME SYSTEM, SIX TIMES. ★ EACH FIX MOVES THE CONSTRAINT.

  30/s    ★ baseline                       p99 ★ 22 ms   ✓
    │
    ▼
  300/s   ★ LAYER 5 — an HTTP call in the transaction
          ★ 178 idle-in-transaction · db_ms 2% of total
          ★ FIX: move it to the outbox        p99 4,102 → ★ 41 ms
    │                                                  ★ 100×
    ▼
  3,000/s ★ LAYER 4a — a hot stock row (Zipf: 23% on one SKU)
          ★ 412 × transactionid · CPU 22%
          ★ FIX: hot row LAST in the txn      p99 8,204 → ★ 340 ms
    │                                                  ★ 24×
    ▼
  3,000/s ★ LAYER 4b — the rightmost index leaf page
          ★ 88 × BufferContent on orders_pkey
          ★ FIX: hash-partition (★ + move tracking_number out)
    │                                          p99 340 → ★ 88 ms
    ▼
  10,000/s ★ LAYER 3 — the product join reading from disk
          ★ cache hit 97.9% · CPU 88%
          ★ FIX: shared_buffers 8 → 48 GB     p99 1,204 → ★ 84 ms
    │
    ▼
  30,000/s ★ LAYER 6 — WAL 184 MB/s, ★ replay is single-threaded
          ★ stock HOT ratio ★ 1%
          ★ FIX: ★ drop ONE unused index + lz4
    │                                  WAL ★ 184 → 41 MB/s
    ▼                                  lag ★ 41 s → 1.2 s
  30,000/s ★ NOT THE DATABASE — ★ 216 GB/hour of logs
          ★ FIX: sample                       ★ 60 → 0.4 MB/s
    │
    ▼
  ★ 30,000/s SUSTAINED · p99 94 ms · ★ ONE MACHINE
  ⇒ ★ LAYER 7 NEVER REACHED.

 ★ NOTE THE PATTERN: ★ the constraint was NEVER where the
   previous one was, and ★ it was never predictable in advance.
```

**Diagram 2 — data flow: the three signatures, and what NOT to do**

```
  ★ THROUGHPUT vs OFFERED LOAD — ★ three completely different shapes

  ★ LAYER 3 (I/O)              ★ LAYER 4 (CONTENTION)
   tps│        ●●●●●            tps│ ●●●●●●●●●●●●●●●
      │    ●●                      │ ★ FLAT and LOW
      │ ●●                         │
      └────────────── load         └────────────── load
   ★ CPU 20%, ★ iowait 60%      ★ CPU ★ 8%, ★ Lock waits
   ★ FIX: ★ MORE RAM / DISK     ★ FIX: ★ REMOVE THE SHARED OBJECT
   ★ ✓ capacity helps            ★ ✗ CAPACITY DOES NOTHING
                                 ★ ✗ more concurrency = WORSE

  ★ LAYER 5 (QUEUEING)          ★ LAYER 6 (ONE MACHINE)
   tps│ ●●●●●●●●●●●●●●●         tps│         ●●●●●●
      │ ★ FLAT                     │     ●●
   p99│           ●                │  ●●
      │       ●   ★ LINEAR         └────────────── load
      │   ●                        ★ CPU ★ 96% REAL WORK
      └────────────── concurrency  ★ FIX: ★ replicas / split /
   ★ FIX: ★ REDUCE HOLD TIME             rollups / retention
   ★ ✗ MORE CONNECTIONS = MUCH WORSE

 ★ THE ONE-LINE DISCRIMINATOR:
   ★ HIGH CPU + high throughput  ⇒ layer 3 or 6
   ★ LOW CPU + Lock waits        ⇒ ★ layer 4
   ★ FLAT tps + LINEAR latency   ⇒ ★ layer 5
   ⇒ ★ AND THE WRONG FIX FOR EACH IS THE INSTINCTIVE ONE.
```

**Diagram 3 — before/after: fixing one thing vs fixing everything**

```
 ✗ THE "FIX EVERYTHING" APPROACH
 ┌───────────────────────────────────────────────────────────────┐
 │ ★ p99 is 4,102 ms at 300/sec. ★ The team ships, in one PR:     │
 │   ① move the HTTP call out of the transaction                  │
 │   ② add 6 indexes "that looked useful"                         │
 │   ③ raise max_connections 200 → 1000                           │
 │   ④ add a Redis cache in front of the product lookup           │
 │   ⑤ double the instance size                                   │
 │                                                                │
 │ ★ RESULT: ★ p99 4,102 → 180 ms. ★ Everyone is happy.           │
 │                                                                │
 │ ★ WHAT WAS ACTUALLY LEARNED: ★ NOTHING.                        │
 │ ★ AND WHAT WAS ACTUALLY INTRODUCED:                            │
 │   ★ ② 6 indexes ⇒ ★ HOT updates destroyed ⇒ 4.5× the WAL       │
 │      ⇒ ★ replica lag, ★ discovered at 30,000/sec               │
 │   ★ ③ 1000 connections ⇒ ★ PAST THE THROUGHPUT PEAK (65)       │
 │      ⇒ ★ 2.6× LESS throughput at high load                     │
 │   ★ ④ a cache ⇒ ★ a load-bearing component (57)                │
 │      ⇒ ★ a 90-second Redis blip is now a 41-minute outage      │
 │   ★ ⑤ 2× cost, ★ for a problem that was not resource-bound     │
 │                                                                │
 │ ★ ⇒ THE FIX WAS ① ALONE. ★ ②–⑤ made the system worse and       │
 │   ★ nobody can tell, because they shipped together.            │
 └───────────────────────────────────────────────────────────────┘

 ✓ ONE THING AT A TIME
 ┌───────────────────────────────────────────────────────────────┐
 │ ★ measure → ★ ONE fix → ★ re-measure → ★ record the delta      │
 │                                                                │
 │ ① move the HTTP call out       p99 4,102 → ★ 41 ms  ★ 100×    │
 │ ② hot row last                 p99 8,204 → ★ 340 ms ★ 24×     │
 │ ③ hash-partition               p99   340 → ★ 88 ms  ★ 3.9×    │
 │ ④ shared_buffers 8 → 48 GB     p99 1,204 → ★ 84 ms  ★ 14×     │
 │ ⑤ ★ DROP one unused index      WAL 184 → ★ 41 MB/s  ★ 4.5×    │
 │ ⑥ sample the logs              logs 60 → ★ 0.4 MB/s ★ 150×    │
 │                                                                │
 │ ★ SIX FIXES. ★ SIX MEASURED DELTAS. ★ EVERY ONE REVERTIBLE.    │
 │ ★ AND ⑤ — ★ DROPPING an index — was the largest single win.    │
 │   ★ It would never have been found by adding things.           │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Reproduce the layer signatures deliberately, on one table.**

```sql
CREATE TABLE t (id bigserial PRIMARY KEY, k int NOT NULL,
                v text NOT NULL, n bigint NOT NULL DEFAULT 0,
                updated_at timestamptz NOT NULL DEFAULT now());
INSERT INTO t (k, v) SELECT (random()*100000)::int, repeat('x',200)
  FROM generate_series(1,20000000);
VACUUM ANALYZE t;
```

**Layer 1 — the plan.**
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM t WHERE k = 500;
```
```
 ->  ★ Seq Scan on t  (actual rows=201)
       ★ Rows Removed by Filter: 19,999,799
 Execution Time: ★ 1,884 ms
```
```sql
CREATE INDEX ON t (k);
EXPLAIN (ANALYZE) SELECT count(*) FROM t WHERE k = 500;
```
```
 Execution Time: ★ 0.412 ms        ★ 4,573×
 ⇒ ★ LAYER 1. ★ Hours of work. ★ The largest single multiplier
   available anywhere.
```

**Layer 3 — I/O.**
```sql
ALTER SYSTEM SET shared_buffers = '128MB'; -- restart
SELECT pg_stat_reset();
```
```bash
pgbench -f /tmp/random_read.sql -c 32 -j 8 -T 60 db
psql -tAc "SELECT round(100.0*sum(heap_blks_hit)/
             nullif(sum(heap_blks_hit)+sum(heap_blks_read),0),2)
             FROM pg_statio_user_tables"
iostat -x 1 3 | tail -3
```
```
 ★ hit ratio ★ 61.4%
 ★ %iowait ★ 71.2
 ★ tps = 4,102
```
```sql
ALTER SYSTEM SET shared_buffers = '8GB'; -- restart, re-run
```
```
 ★ hit ratio ★ 99.91%
 ★ %iowait ★ 0.4
 ★ tps = ★ 88,204        ★ 21×
 ⇒ ★ LAYER 3. ★ THE ONLY LAYER WHERE MORE HARDWARE IS THE FIX.
```

**Layer 4 — contention, and the Zipf point.**
```bash
# ★ UNIFORM keys — ★ what a naive load test does
cat > /tmp/uniform.sql <<'EOF'
\set k random(1, 100000)
UPDATE t SET n = n + 1 WHERE k = :k;
EOF
pgbench -f /tmp/uniform.sql -c 200 -j 8 -T 30 db | grep tps
```
```
 tps = ★ 41,204        — ★ looks perfect
```
```bash
# ★ ZIPF keys — ★ what production actually does
cat > /tmp/zipf.sql <<'EOF'
\set k ★ random_zipfian(1, 100000, 1.2)
UPDATE t SET n = n + 1 WHERE k = :k;
EOF
pgbench -f /tmp/zipf.sql -c 200 -j 8 -T 30 db | grep tps
```
```
 tps = ★ 2,884        ★ 14× WORSE
```
```sql
-- ★ during the zipf run
SELECT wait_event_type, wait_event, count(*) FROM pg_stat_activity
 WHERE state='active' GROUP BY 1,2 ORDER BY 3 DESC LIMIT 2;
```
```
 wait_event_type |   wait_event    | count
-----------------+-----------------+-------
 Lock            | ★ transactionid |  ★ 178
 ⇒ ★ THE UNIFORM TEST WOULD HAVE SHIPPED THIS TO PRODUCTION.
```

**Layer 5 — queueing, and the exact linearity.**
```bash
cat > /tmp/hold.sql <<'EOF'
BEGIN;
SELECT n FROM t WHERE id = 1;
SELECT pg_sleep(0.05);
COMMIT;
EOF
for c in 25 50 100 200; do
  echo -n "c=$c  "
  pgbench -f /tmp/hold.sql -c $c -j 8 -T 20 db \
    | grep -E 'tps|latency average' | tr '\n' ' '; echo
done
```
```
 c=25   tps = ★ 494   latency = ★ 50.6 ms
 c=50   tps = ★ 496   latency = ★ 100.8 ms
 c=100  tps = ★ 495   latency = ★ 201.7 ms
 c=200  tps = ★ 494   latency = ★ 404.1 ms
 ⇒ ★ THROUGHPUT FLAT. ★ LATENCY EXACTLY LINEAR.
   ★ tps = pool_size ÷ hold_time. ★ Little's Law, visible.
```

**Layer 6 — WAL and the single-threaded replay ceiling.**
```sql
SELECT pg_current_wal_lsn() AS a \gset
-- run 60 seconds of the write workload
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'a')/60) AS wal_per_sec;
```
```
 wal_per_sec
-------------
 ★ 184 MB
 ⇒ ★ REPLAY IS ~100 MB/s, SINGLE-THREADED (42, 58).
   ⇒ ★ 184 MB/s CANNOT BE REPLAYED. ★ Lag grows forever.
```
```sql
-- ★ where is it coming from?
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
  FROM pg_stat_user_tables WHERE n_tup_upd > 0;
```
```
 relname | n_tup_upd | n_tup_hot_upd | hot_pct
---------+-----------+---------------+---------
 t       |  44201059 |       ★ 442010|   ★ 1.0
 ⇒ ★ 1% HOT ⇒ ★ an index on a column being updated (46).
```
```sql
SELECT indexrelname, idx_scan FROM pg_stat_user_indexes WHERE relname='t';
```
```
 indexrelname     | idx_scan
------------------+----------
 t_pkey           | 44201059
 t_k_idx          |  8842119
 ★ t_updated_idx  |      ★ 0
```
```sql
DROP INDEX CONCURRENTLY t_updated_idx;
ALTER SYSTEM SET wal_compression='lz4'; SELECT pg_reload_conf();
-- re-run
```
```
 ★ hot_pct ★ 1.0 → ★ 97.6
 ★ wal_per_sec ★ 184 MB → ★ 39 MB        ★ 4.7×
```

---

## Example 2 — production scenario

**The situation.** A payments platform. Traffic has grown 40× in eight months. The team has been "scaling" continuously and has spent ₹18 lakh/month on infrastructure. p99 is still 3,400 ms.

```
 ★ WHAT HAS BEEN TRIED, IN ORDER, OVER EIGHT MONTHS:
   ★ instance size          ★ 8 → 16 → 32 → 64 → 96 vCPU
   ★ max_connections        200 → 500 → ★ 1,200
   ★ read replicas          ★ 0 → 5
   ★ Redis                  ★ added, 94% hit rate
   ★ indexes added          ★ 41
   ★ a sharding project     ★ started, 3 months in

 ★ RESULT: ★ p99 3,400 ms · ★ errors 2.8% · ★ CPU 34%
 ⇒ ★ ₹18 LAKH/MONTH AND CPU IS 34%.
   ★ THAT NUMBER ALONE SAYS THE PROBLEM IS NOT RESOURCES.
```

**Step 1 — stop, and measure properly.**

```
 ★ THE FIRST ACTION: ★ FREEZE ALL SCALING WORK, ★ including the
   sharding project.
 ⇒ ★ THE ARGUMENT THAT WON: ★ "we have added capacity five times
   and CPU is 34%. ★ Whatever the constraint is, ★ it is not
   capacity — ★ and sharding is layer 7."
```
```
 ★ THE THREE PER-REQUEST NUMBERS (67 level 1):
   total_ms p99      ★ 3,400 ms
   db_ms p99         ★ 41 ms        ⇒ ★ 1.2%
   ★ pool_wait p99   ★ 3,180 ms     ⇒ ★ 93.5%
   query_count p99   ★ 8
 ⇒ ★ LAYER 5. ★ AND IT HAS BEEN LAYER 5 THE ENTIRE TIME.
   ★ Every fix applied was for layers 3, 6 and 7.
```
```sql
SELECT state, wait_event, count(*) FROM pg_stat_activity
 WHERE backend_type='client backend' GROUP BY 1,2 ORDER BY 3 DESC LIMIT 4;
```
```
        state          | wait_event  | count
-----------------------+-------------+-------
 idle                  | ClientRead  | ★ 884
 ★ idle in transaction | ClientRead  | ★ 291
 active                | -           |  ★ 12
 LWLock                | ★ ProcArray |  ★ 41
 ⇒ ★ 1,175 CONNECTIONS. ★ 12 DOING WORK.
 ⇒ ★ AND 41 CONTENDING ON ProcArray — ★ the snapshot cost is
   O(max_connections) PER STATEMENT (65). ★ The connections
   themselves are now a cost.
```

**Step 2 — fix layer 5. One thing.**

```sql
SELECT application_name, count(*), max(now()-xact_start) AS oldest,
       substring(mode() WITHIN GROUP (ORDER BY query),1,60) AS typical
  FROM pg_stat_activity WHERE state='idle in transaction'
 GROUP BY 1 ORDER BY 2 DESC;
```
```
 application_name | count |   oldest    |               typical
------------------+-------+-------------+--------------------------------------
 ★ payments-api   | ★ 241 | ★ 00:00:06.2| SELECT * FROM accounts WHERE id = $1
 ★ settlement-job |  ★ 50 | ★ 00:04:11  | UPDATE settlements SET …
```
```js
// ★ THE CAUSE
await withTransaction(async (tx) => {
  const acct = await tx.query('SELECT * FROM accounts WHERE id=$1', [id]);
  ★ const risk = await riskService.evaluate(acct);   // ★ p99 5.8 s
  ★ const auth = await gateway.authorise(acct, amt); // ★ p99 4.1 s
  await tx.query('INSERT INTO payments (…) VALUES (…)', [...]);
});
// ★ HOLD TIME = ★ the sum of two external p99s = ★ ~10 s.
// ★ LITTLE'S LAW: ★ 400 rps × 10 s = ★ 4,000 connections needed.
// ⇒ ★ NO POOL SIZE SURVIVES THIS. ★ Raising max_connections to
//   1,200 was treating the symptom, ★ and past the throughput
//   peak it made things worse.
```
```js
// ★ THE FIX — ★ and ONLY this fix
const acct = await pool.query('SELECT * FROM accounts WHERE id=$1', [id]);
const risk = await riskService.evaluate(acct);           // ★ no txn held
const auth = await gateway.authorise(acct, amt, { idempotencyKey: key });
await withTransaction(async (tx) => {                     // ★ ~6 ms
  await tx.query(
    `INSERT INTO payments (…, idempotency_key) VALUES (…)
     ★ ON CONFLICT (idempotency_key) DO NOTHING`, [...]);
  await tx.query('INSERT INTO outbox (…) VALUES (…)', [...]);
});
// ★ plus a sweeper for payments stuck in 'pending' (52)
```
```
 ★ MEASURED, ★ CHANGING NOTHING ELSE:
   ★ hold time ★ 10,000 ms → ★ 6 ms
   ★ idle in transaction ★ 291 → ★ 3
   ★ p99 ★ 3,400 ms → ★ 180 ms        ★ 19×
   ★ errors ★ 2.8% → ★ 0.02%
 ⇒ ★ NO INFRASTRUCTURE CHANGE. ★ NO NEW INDEX. ★ ONE HANDLER.
```

**Step 3 — now undo the damage the previous "scaling" caused.**

```sql
-- ★ ① the 41 indexes added over 8 months
SELECT relname, indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
  FROM pg_stat_user_indexes
 WHERE idx_scan < 100 AND pg_relation_size(indexrelid) > 100*1024*1024
 ORDER BY pg_relation_size(indexrelid) DESC;
```
```
     relname   |       indexrelname        | idx_scan |  size
---------------+---------------------------+----------+---------
 ★ payments    | idx_payments_updated_at   |     ★ 0  | ★ 41 GB
 ★ payments    | idx_payments_metadata_gin |     ★ 12 | ★ 88 GB
 ★ accounts    | idx_accounts_last_seen    |     ★ 0  | ★ 12 GB
 ★ settlements | idx_settlements_notes_trgm|     ★ 3  | ★ 18 GB
 ⇒ ★ 159 GB OF INDEXES SCANNED 15 TIMES IN EIGHT MONTHS.
```
```sql
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
  FROM pg_stat_user_tables WHERE n_tup_upd > 1000000 ORDER BY hot_pct;
```
```
   relname   | n_tup_upd  | n_tup_hot_upd | hot_pct
-------------+------------+---------------+---------
 ★ payments  | ★ 884201188|    ★ 8,842,011|  ★ 1.0
 ★ accounts  | ★ 412088420|   ★ 20,604,421|  ★ 5.0
 ⇒ ★ THE `updated_at` AND `last_seen` INDEXES ARE ON THE EXACT
   COLUMNS EVERY UPDATE CHANGES (46).
```
```sql
DROP INDEX CONCURRENTLY idx_payments_updated_at;
DROP INDEX CONCURRENTLY idx_payments_metadata_gin;
DROP INDEX CONCURRENTLY idx_accounts_last_seen;
DROP INDEX CONCURRENTLY idx_settlements_notes_trgm;
```
```
 ★ hot_pct: ★ payments 1% → ★ 96.8% · ★ accounts 5% → ★ 94.2%
 ★ WAL ★ 148 MB/s → ★ 22 MB/s        ★ 6.7×
 ★ replica lag ★ 22 min → ★ 0.8 s
 ★ disk ★ 4.1 TB → ★ 1.9 TB
 ⇒ ★ DROPPING FOUR INDEXES WAS THE SECOND-LARGEST WIN.
```

```sql
-- ★ ② the connection count, now that hold time is 6 ms
--    ★ Little's Law: 400 rps × 0.006 s = ★ 2.4 connections
ALTER SYSTEM SET max_connections = ★ 300;   -- was 1,200
ALTER SYSTEM SET work_mem = '32MB';         -- ★ safe now (65)
```
```ini
# ★ PgBouncer, transaction mode
pool_mode = transaction
default_pool_size = 64
max_client_conn = 3000
query_wait_timeout = 10
max_prepared_statements = 200
```
```
 ★ backends ★ 1,175 → ★ 68
 ★ ProcArray waits ★ 41 → ★ 0
 ★ memory ★ 78 GB → ★ 21 GB
 ★ p99 ★ 180 ms → ★ 94 ms        ★ 1.9× MORE, from FEWER connections
```

```js
// ★ ③ the Redis cache — ★ measure whether it is still needed
// ★ hit rate 94% sounds great. ★ What is the underlying query?
```
```sql
SELECT calls, mean_exec_time FROM pg_stat_statements
 WHERE query LIKE '%FROM accounts WHERE id%';
```
```
 calls  | mean_exec_time
--------+----------------
 412088 |      ★ 0.04
 ⇒ ★ 0.04 ms. ★ A Redis GET is 0.2 ms (57).
 ⇒ ★ THE CACHE WAS MAKING IT 5× SLOWER, ★ and made a Redis
   failure a total outage.
```
```
 ★ CACHE REMOVED FROM THAT PATH.
   ★ p50 ★ 0.24 ms → ★ 0.09 ms · ★ 180 lines of invalidation code
   deleted · ★ blast radius eliminated.
```

**Step 4 — cancel the sharding project, with the numbers.**

```
 ★ THE EXHAUSTION CHECKLIST (60), RUN PROPERLY:
   ✓ ★ indexes           — ★ 4 DROPPED (not added)
   ✓ ★ N+1               — none found (8 queries/request)
   ✓ ★ retention         — ★ payments unpartitioned, 1.9 TB
                            ⇒ ★ NEXT, not sharding
   ✓ ★ caching           — ★ removed; it was a pessimisation
   ✓ ★ replicas          — ★ 5 exist; ★ now actually usable
                            (lag 0.8 s vs 22 min)
   ✓ ★ hot rows          — ★ checked: max 41 writes/sec on one
                            account. ★ Not hot.
   ✓ ★ pooling           — ★ fixed
   ✓ ★ vertical scaling  — ★ 96 vCPU at ★ 31% ⇒ ★ NOT exhausted
   ✓ ★ functional split  — ★ not yet attempted

 ⇒ ★ SHARDING WAS FOUR LAYERS TOO EARLY.
 ★ PROJECT CANCELLED. ★ Three months of work stopped.
 ★ AND THE TRIGGER CONDITION RECORDED: ★ "revisit when sustained
   writes exceed 100k/sec after partitioning and a functional
   split, with vertical scaling exhausted."
```

**Step 5 — and then the next constraint, which was layer 6.**

```bash
k6 run -e RATE=4000 load.js --duration 45m
```
```
 ★ p99 stable at 94 ms for 30 minutes, ★ then ★ 1,840 ms
```
```sql
SELECT relname, n_dead_tup, last_autovacuum,
       pg_size_pretty(pg_total_relation_size(relid)) AS size
  FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 2;
```
```
  relname  | n_dead_tup |    last_autovacuum     |  size
-----------+------------+------------------------+---------
 ★ payments| ★ 412088420|  ★ 2026-08-19 03:11:02 | ★ 1.9 TB
 ⇒ ★ AUTOVACUUM LAST RAN 8 DAYS AGO on a table with 412 MILLION
   dead tuples.
```
```sql
SELECT 'backend' AS src, pid::text, (now()-xact_start)::text AS age
  FROM pg_stat_activity WHERE backend_xmin IS NOT NULL
UNION ALL SELECT 'slot', slot_name, active::text FROM pg_replication_slots
UNION ALL SELECT '2pc', gid, (now()-prepared)::text FROM pg_prepared_xacts
ORDER BY 3;
```
```
   src    |     pid      |   age
----------+--------------+----------
 ★ slot   | ★ debezium_v1| ★ false
 ⇒ ★ AN INACTIVE REPLICATION SLOT from a CDC connector
   decommissioned in June. ★ It has pinned xmin for 8 weeks (47, 62).
```
```sql
SELECT pg_drop_replication_slot('debezium_v1');
ALTER SYSTEM SET max_slot_wal_keep_size = '128GB';   -- ★ the guard
ALTER TABLE payments SET (autovacuum_vacuum_scale_factor = 0.01,
                          autovacuum_vacuum_cost_delay = 0);
```
```
 ★ dead tuples ★ 412M → ★ 8M within 4 hours
 ★ p99 stable at ★ 88 ms over 45 minutes
 ⇒ ★ THE SAME OBJECT AS TOPICS 41, 47, 62 AND 76. ★ Fifth time.
```

**Step 6 — results.**

| | 8 months of "scaling" | 3 weeks of measurement |
|---|---|---|
| p99 | ★ **3,400 ms** | **88 ms** (**39×**) |
| Error rate | ★ 2.8% | **0.01%** |
| CPU | ★ 34% (waiting) | 61% (**working**) |
| Backends | ★ 1,175 | **68** |
| `max_connections` | ★ 1,200 | **300** |
| Indexes | ★ +41 added | ★ **−4 dropped** |
| WAL | ★ 148 MB/s | **22 MB/s** (**6.7×**) |
| Replica lag | ★ 22 min | **0.8 s** |
| Disk | ★ 4.1 TB | **1.9 TB** |
| Redis | ★ load-bearing | ★ **removed from the hot path** |
| Sharding project | ★ 3 months in | ★ **cancelled** |
| Infrastructure | ★ ₹18 L/month | **₹6.2 L/month** |

```
 ★ SEVEN LESSONS:
 ① ★ 34% CPU AT ₹18 LAKH/MONTH WAS THE WHOLE DIAGNOSIS.
   ★ Five capacity increases against a problem that was never
   capacity.
 ② ★ IT HAD BEEN LAYER 5 FOR EIGHT MONTHS, and every fix
   applied was for layers 3, 6 and 7.
 ③ ★ THE FIX WAS ONE HANDLER. ★ Two external calls moved out of
   a transaction: ★ 19× on p99, ★ zero infrastructure change.
 ④ ★ DROPPING FOUR INDEXES WAS THE SECOND-LARGEST WIN. ★ Eight
   months of adding indexes had destroyed HOT updates and made
   the replicas unusable.
 ⑤ ★ FEWER CONNECTIONS MADE IT FASTER. ★ 1,175 → 68 gave
   ★ 1.9× more throughput (65).
 ⑥ ★ THE 94%-HIT-RATE CACHE WAS A PESSIMISATION. ★ 0.04 ms
   query behind a 0.2 ms cache (57).
 ⑦ ★ THE SHARDING PROJECT WAS FOUR LAYERS TOO EARLY, ★ and the
   checklist that cancelled it took an afternoon.
```

---

## Common mistakes

**1. Fixing more than one thing at a time.**
- *Symptom:* p99 improves, nobody knows why, and two of the five changes made the system worse in ways discovered months later.
- *Fix:* one change, re-measure, record the delta. Every fix must be independently revertible.

**2. Adding capacity for a contention or queueing problem.**
- *Symptom:* five instance upgrades and CPU still at 34%.
- *Fix:* low CPU with failing requests is layers 4 or 5. Capacity is the answer for layer 3 only.

**3. Uniform key distribution in load tests.**
- *Symptom:* the test passes at 41,000 tps and production collapses at 2,884.
- *Fix:* Zipf-distributed keys, or replay real key frequencies. `pgbench` has `random_zipfian`.

**4. Closed-loop load testing.**
- *Symptom:* the test slows down when the system does, hiding queue collapse entirely.
- *Fix:* open-loop — a fixed arrival rate regardless of response time.

**5. Load tests that are too short or too small.**
- *Symptom:* a 100% cache hit rate and no bloat, checkpoint spikes, autovacuum pressure or replication lag.
- *Fix:* a dataset larger than RAM, 30+ minutes.

**6. Measuring the mean.**
- *Symptom:* "average latency is 40 ms" while the p99 is 4 seconds and users are leaving.
- *Fix:* p50, p99, p999 **and the error rate**.

**7. Adding indexes as a scaling strategy.**
- *Symptom:* 41 indexes, 159 GB, scanned 15 times in eight months — and HOT updates destroyed.
- *Fix:* check `idx_scan` before keeping any index. Dropping is often the bigger win.

**8. Raising `max_connections` when the pool exhausts.**
- *Symptom:* past the throughput peak, more connections mean *less* throughput and `ProcArray` contention.
- *Fix:* reduce hold time. `pool_size = arrival_rate × hold_time`.

**9. External calls inside a transaction.**
- *Symptom:* hold time equals the sum of third-party p99s; no pool size survives it.
- *Fix:* read, call outside, write with an idempotency key, plus a sweeper.

**10. Skipping a layer.**
- *Symptom:* three months of sharding work against a problem an afternoon of pooling analysis would have fixed.
- *Fix:* the exhaustion checklist. Layers are not skippable.

**11. Forgetting that observability scales too.**
- *Symptom:* 216 GB/hour of logs filling the disk while you try to diagnose the system.
- *Fix:* sample. Always log errors and the slow tail; sample the rest at 0.1%.

**12. Not re-checking earlier layers after a fix.**
- *Symptom:* fixing layer 5 exposes a layer-4 problem that was previously masked.
- *Fix:* re-run the full classification after every change. The constraint *moves*.

---

## Hands-on proof

**PROVE IT #1–#5 — Example 1** (layer 1 at 4,573× from one index, layer 3 at 21× from `shared_buffers`, layer 4 showing uniform 41,204 tps versus Zipf 2,884 tps on identical code, layer 5's exactly-linear latency with flat throughput, and layer 6's 184 MB/s of WAL traced to one unused index).

**PROVE IT #6 — open-loop vs closed-loop hides the truth.**
```bash
# ★ closed loop: 200 fixed clients
pgbench -c 200 -j 8 -T 60 db | grep -E 'tps|latency average'
```
```
 tps = ★ 494   latency average = ★ 404 ms
 ⇒ ★ "the system does 494 tps." ★ Looks stable.
```
```bash
# ★ open loop: a fixed arrival rate of 2,000/sec
k6 run --vus 2000 -e RATE=2000 open_loop.js
```
```
 ★ p99 ★ 41,204 ms · ★ errors ★ 71%
 ⇒ ★ THE SAME SYSTEM. ★ The closed-loop test could not show
   collapse, because it slowed down with the system.
```

**PROVE IT #7 — the constraint moves after every fix.**
```bash
for stage in baseline fix1 fix2 fix3; do
  git checkout $stage && deploy
  k6 run -e RATE=3000 load.js > /tmp/$stage.txt
  psql -tAc "SELECT wait_event, count(*) FROM pg_stat_activity
              WHERE state='active' GROUP BY 1 ORDER BY 2 DESC LIMIT 1"
done
```
```
 ★ baseline: ClientRead (idle in transaction)   ⇒ ★ layer 5
 ★ fix1:     transactionid                       ⇒ ★ layer 4
 ★ fix2:     BufferContent                       ⇒ ★ layer 4b
 ★ fix3:     DataFileRead                        ⇒ ★ layer 3
 ⇒ ★ FOUR DIFFERENT CONSTRAINTS. ★ None predictable in advance.
```

**PROVE IT #8 — dropping an index increases write throughput.**
```bash
psql -c "SELECT indexrelname, idx_scan FROM pg_stat_user_indexes
          WHERE relname='t' AND idx_scan = 0"
pgbench -f /tmp/update.sql -c 64 -j 8 -T 30 db | grep tps
psql -c "DROP INDEX CONCURRENTLY t_updated_idx"
pgbench -f /tmp/update.sql -c 64 -j 8 -T 30 db | grep tps
```
```
 ★ before: tps = ★ 12,204
 ★ after:  tps = ★ 41,882        ★ 3.4×, ★ by REMOVING something
```

**PROVE IT #9 — logs before the database.**
```bash
# ★ at 30,000 rps with full structured logging
watch -n5 'df -h /var/log | tail -1; psql -tAc "SELECT
  pg_size_pretty(pg_database_size(current_database()))"'
```
```
 ★ /dev/nvme1  200G  ★ 198G  ★ 99%     ← ★ logs
 ★ database:   ★ 41 GB                  ← ★ the actual data
 ⇒ ★ THE LOGS ARE 4.8× THE DATABASE.
```

---

## The design decision framework

```
★★★ FIND THE ONE CONSTRAINT. FIX EXACTLY IT. RE-MEASURE. ★★★

 ① ★ ESTABLISH A BASELINE BEFORE CHANGING ANYTHING
    ★ p50/p99/p999 · ★ error rate · CPU · ★ the three per-request
    numbers (db_ms, pool_wait, query_count)
    ⇒ ★ without a baseline you cannot prove a fix worked.

 ② ★ MAKE THE LOAD TEST RESEMBLE PRODUCTION
    ✓ ★ ZIPF keys — ★ a uniform test never finds layer 4
    ✓ ★ a dataset larger than RAM
    ✓ ★ the real write mix
    ✓ ★ OPEN-LOOP arrival rate — ★ closed-loop hides collapse
    ✓ ★ 30+ minutes — bloat, checkpoints, vacuum, lag
    ⇒ ★ a test that does not do these teaches you something false.

 ③ ★ CLASSIFY BEFORE FIXING (Topic 67)
    ★ high CPU + high throughput  ⇒ layer 3 or 6
    ★ LOW CPU + Lock waits        ⇒ ★ layer 4 — ★ capacity does
                                    nothing
    ★ flat tps + linear latency   ⇒ ★ layer 5 — ★ more connections
                                    make it worse
    ★ high iowait                 ⇒ ★ layer 3 — ★ the ONLY layer
                                    where hardware is the answer
    ★ db_ms ≪ total_ms            ⇒ ★ not the database at all

 ④ ★ FIX EXACTLY ONE THING
    ⇒ ★ and it must be independently revertible.
    ⇒ ★ record the before/after delta.

 ⑤ ★ RE-MEASURE AND RE-CLASSIFY
    ⇒ ★ the constraint MOVES, and ★ it is never where the last
      one was.

 ⑥ ★ RESPECT THE LAYER ORDER — ★ IT IS ECONOMICS
    ★ plan (hours, 100–10,000×) → redundant work (days) →
    I/O (days) → contention (days) → queueing (days) →
    one machine (weeks) → ★ distribution (quarters)
    ⇒ ★ each layer's fix is 10–1000× cheaper than the next.
    ⇒ ★ AND YOU CANNOT SKIP ONE: ★ a hot key lands on one shard.

 ⑦ ★ CONSIDER REMOVING THINGS
    ★ dropping an unused index was the largest single win in one
      example and the second-largest in the other.
    ★ removing a cache made p50 2.7× faster.
    ★ reducing connections 1,175 → 68 gave 1.9× throughput.
    ⇒ ★ SCALING IS NOT ONLY ADDING.

 ⑧ ★ BUDGET FOR WHAT IS NOT THE DATABASE
    ★ 10× pool sizing · ★ 100× LOG VOLUME and metric cardinality
    · ★ 1000× ports, TLS, DNS
    ⇒ ★ at 100×, ★ observability becomes the bottleneck. ★ Sample.

 ⑨ ★ WRITE DOWN EVERY DELTA
    ★ the change · ★ the before/after · ★ the layer it addressed ·
    ★ whether it was reverted
    ⇒ ★ this is what makes the next incident an hour instead of
      eight months.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
On a 20M-row table, deliberately produce each of layers 1, 3, 4 and 5. For each, capture the wait events and CPU, apply the correct fix, and record the multiplier. Then apply the *wrong* fix (add capacity to the contention case) and show that nothing changes.

### Exercise 2 — medium (apply it)
Build an open-loop load test with Zipf-distributed keys. Run the same workload with uniform keys and report both results. Then run a closed-loop test at the same nominal load and explain why it cannot show collapse.

Finally, take a table with an unused index on a frequently-updated column, measure write throughput and WAL, drop the index, and re-measure.

### Exercise 3 — hard (production simulation)
A payments platform has spent eight months scaling: five instance upgrades, `max_connections` 200 → 1,200, five read replicas, Redis, 41 new indexes, and a three-month sharding project. p99 is 3,400 ms, errors 2.8%, CPU 34%, cost ₹18 lakh/month.

(a) What does "34% CPU at ₹18 lakh/month" tell you, before any other measurement?
(b) Take the three per-request numbers. Which layer is the constraint, and how long has it been?
(c) Find the cause in the handler. Apply Little's Law to show why no pool size would have worked.
(d) Apply exactly one fix and predict the improvement.
(e) Audit the 41 added indexes. Which four matter, and why does dropping them help *writes*?
(f) `max_connections` is 1,200 and hold time is now 6 ms. Compute the correct value and explain why fewer is faster.
(g) The Redis cache has a 94% hit rate. Measure the underlying query and decide.
(h) Run the exhaustion checklist and write the argument for cancelling the sharding project, including the revisit trigger.
(i) After 30 minutes at stable p99, latency degrades to 1,840 ms. Find the cause. Which earlier topic is this the fifth appearance of?
(j) Write the change log: every fix, its layer, its measured delta, and whether it could be reverted independently.

---

## Mental model checkpoint

1. Name the seven layers in order, with a signal and a typical cost for each.
2. Which single layer is "add more hardware" the correct answer for?
3. Why can you not skip a layer?
4. Give Little's Law and the two ways it says throughput can increase. Which has a ceiling?
5. Name five things most load tests get wrong.
6. Why must a load test be open-loop?
7. Why does a uniform key distribution hide the most common real constraint?
8. Give the one-line discriminator for saturation, contention and queueing.
9. Why is fixing one thing at a time a discipline rather than a preference?
10. Give three examples where *removing* something increased throughput.
11. What becomes the bottleneck at 100×, and what is the fix?
12. Why must you re-classify after every fix?

---

## Quick reference card

**★ The seven layers**

| # | Layer | Signal | Fix | Cost | Gain |
|---|---|---|---|---|---|
| 1 | ★ plan | `Rows Removed by Filter` | index / `ANALYZE` | hours | ★ 100–10,000× |
| 2 | redundant | ★ `calls` ≫ | batch / `COPY` | days | 10–100× |
| 3 | ★ I/O | `iowait`, `read` ≫ `hit` | ★ **RAM / disk** | days | 5–50× |
| 4 | ★ contention | ★ **low CPU** + `transactionid` | atomic / shard the row | days | 5–100× |
| 5 | ★ queueing | ★ **flat tps, linear p99** | ★ **reduce hold time** | days | 3–20× |
| 6 | one machine | CPU 90% real, WAL > 100 MB/s | replicas / split / rollups | weeks | 2–10× |
| 7 | distribution | all of the above exhausted | shard | ★ **quarters** | linear |

**★ Little's Law:** `throughput = concurrency ÷ service time`. Reduce service time (1–4) or add concurrency (5–7) — ★ **and concurrency has a ceiling**.

**★ Load-test requirements:** ★ Zipf keys · dataset > RAM · real write mix · ★ **open-loop arrival rate** · 30+ min · measure ★ **p99/p999 and error rate**, never the mean.

**★ The discriminator:** high CPU ⇒ 3/6 · ★ **low CPU + Lock waits ⇒ 4** · ★ **flat tps + linear latency ⇒ 5** · `db_ms ≪ total_ms` ⇒ not the database.

**★ Consider removing:** an unused index (★ largest win, twice) · a cache (★ 2.7× faster) · connections (★ 1.9× throughput).

**★ At 100×, observability is the bottleneck.** Log errors + the slow tail; sample the rest at 0.1%.

**★ Fix one thing. Re-measure. Record the delta.**

---

## When would I use this at work?

1. **Before any capacity increase.** "Low CPU with failing requests" means the money will not help. That single check has, in the example above, saved eight months and ₹12 lakh a month.

2. **Running a load test.** Zipf keys and open-loop arrival rate are the two properties that separate a test which finds real constraints from one that produces a reassuring number. Both are one-line changes.

3. **When a scaling effort has stalled.** Freeze it, take the three per-request numbers, and classify. In the example the constraint had been layer 5 for eight months while every fix targeted layers 3, 6 and 7.

4. **Whenever you are about to add something.** Ask what could be *removed* instead. An unused index on a hot column, a cache in front of a 0.04 ms query, and 1,100 idle connections were each worth more than anything that was added.

---

## Connected topics

**Understand before this:** 67 (the investigation method — this is it, applied repeatedly), 65 (queueing and pooling), 61 (hot rows), 46/47 (HOT updates and bloat), 58 (replica lag and single-threaded replay), 60 (the exhaustion checklist), 77 (the schema being scaled).

**This unlocks:**
- **79** — the principal-engineer review: how to find these before they happen
