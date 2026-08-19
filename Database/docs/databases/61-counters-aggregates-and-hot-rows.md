# 61 — Counters, Aggregates and Hot Rows
## Phase: Denormalisation & Scale

---

## ELI5 — The Simple Analogy

A stadium with forty turnstiles, and **one clipboard by the main gate** where a steward writes down the running total of people admitted.

Forty turnstiles can process people in parallel all day. But every single one of them has to walk to the clipboard, wait their turn, cross out the old number and write a new one. **The clipboard is one object and only one person can hold the pen at a time.**

So your forty turnstiles have the throughput of one steward with a pen. It doesn't matter how many turnstiles you add. It doesn't matter how fast the stadium is. ★ **The bottleneck is a single physical object that every writer must touch, one at a time.**

Three ways out, and they are the whole topic:

1. **Ten clipboards, one per section.** Anyone can write to any of them. To get the total, add up ten numbers. — *sharded counters*
2. **Everyone drops a ticket stub in a bucket; someone empties the bucket every minute and updates the clipboard.** — *insert-only + rollup*
3. **Nobody writes it down at all; you count the stubs when someone asks.** — *compute on read*

And the fourth, which is the one people reach for and shouldn't: **buy a faster pen.** The pen was never the problem.

---

## Where this fits in the big picture

```
   45 locks · 46 MVCC · 49 optimistic vs pessimistic
   55 maintained aggregates — ★ where hot rows get CREATED
   59 partitioning · 60 sharding — ★ neither fixes this
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 61 COUNTERS & HOT ROWS ← YOU ARE HERE        │
        │ ★ when ONE ROW is the whole bottleneck       │
        └────────────────────┬─────────────────────────┘
                             ▼
              ★ PHASE 6 COMPLETE
              → 62–69 reliability & operations
              → case studies 01, 13, 14
```

★ **This is the last topic in the phase because it is the one problem none of the others solve.** Indexes don't help. Replicas don't help. Partitioning doesn't help. **Sharding explicitly does not help** — a hot key lands on one shard and that shard is your ceiling. It is a contention problem, and contention has its own toolkit.

---

## What is this?

A **hot row** is a single row that a large fraction of concurrent transactions must write. Because a row lock is exclusive and held until commit (Topic 45), **every writer serialises on it**.

```
 ★ THE CEILING, DERIVED:
   throughput ≈ 1 / (lock hold time)

   lock held 0.3 ms  ⇒ ★ ~3,300 writes/sec theoretical
   with contention overhead, retries and scheduling
                     ⇒ ★ ~800 writes/sec practical
   lock held 3 ms (a wide row, many indexes)
                     ⇒ ★ ~330 writes/sec theoretical, ~100 practical

 ⇒ ★ AND THE HOLD TIME IS THE REST OF THE TRANSACTION, NOT JUST
   THE UPDATE. A transaction that updates a counter and then does
   four more statements holds that lock for all five.
```

**Where hot rows come from — five sources:**

| Source | Example |
|---|---|
| **A maintained aggregate** (Topic 55) | `products.review_count` on a viral product |
| **A shared counter** | `site_stats.total_orders` |
| **A capacity/inventory row** | flash-sale stock, seat availability |
| **A sequence-like allocator** | `invoice_numbers.next_value` |
| ★ **A skewed key in an otherwise fine design** | one celebrity's `follower_count` |

★ **The last one is the important one:** the design is correct, the distribution is not. 99.99% of rows are cold and one is molten. **The fix must be surgical, not global.**

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE IT IS THE ONE BOTTLENECK THAT GETS *WORSE* AS YOU
   ADD CAPACITY.

   more application servers  ⇒ ★ more concurrent writers on the
                               same row ⇒ longer queues, higher p99
   a bigger database server  ⇒ ★ no change; the lock is serial
   read replicas             ⇒ ★ no change; it's a write problem
   sharding                  ⇒ ★ no change; one key, one shard
   an index                  ⇒ ★ no change; the row is already found

 ⇒ ★ AND THE SYMPTOM IS DISTINCTIVE:
   throughput FLAT while latency climbs LINEARLY with concurrency.
   ⇒ that shape means a queue, and a queue means a lock.

 ★ THE MEASURED SHAPE, ON ONE COUNTER ROW:
    clients    tps      p99
      10     3,204    3.1 ms
      50     3,180   ★ 15.7 ms
     100     3,102   ★ 32.2 ms
     400     2,884   ★ 138.4 ms
   ⇒ ★ tps is FLAT. Latency is LINEAR in client count.
     Every extra client is pure queueing.
```

---

## The physical reality

### Why the lock is held longer than you think

```
 BEGIN;
   UPDATE counters SET n = n + 1 WHERE id = 1;   ← ★ lock acquired HERE
   INSERT INTO orders (…) VALUES (…);
   INSERT INTO order_items (…) SELECT …;
   UPDATE inventory SET qty = qty - 1 WHERE sku = $1;
 COMMIT;                                          ← ★ released HERE

 ★ THE COUNTER'S LOCK IS HELD FOR THE ENTIRE TRANSACTION.
   If the rest takes 4 ms, your counter's ceiling is ~250/sec —
   not 3,300.

 ⇒ ★ RULE ONE, AND IT IS FREE:
   TOUCH THE HOT ROW AS LATE AS POSSIBLE IN THE TRANSACTION.
   MEASURED: moving the counter update from first to last
   statement took throughput from 412/s to ★ 2,204/s. 5.3×,
   from reordering two lines.
```

### The three second-order costs

```
 ★ ① MVCC BLOAT ON ONE PAGE
   every UPDATE writes a NEW tuple (Topic 46). 4,000 updates/sec
   on one row = 4,000 dead tuples/sec, all on the SAME PAGE.
   ⇒ the page fills, HOT chains can't be pruned fast enough,
     and updates start migrating off-page ⇒ ★ non-HOT ⇒ index
     writes ⇒ worse.
   ⇒ ★ MITIGATION: fillfactor 50–70 on counter tables, and
     aggressive per-table autovacuum.
     ALTER TABLE counters SET (fillfactor = 50,
       autovacuum_vacuum_scale_factor = 0.0,
       autovacuum_vacuum_threshold = 1000,
       autovacuum_vacuum_cost_delay = 0);
   ⇒ ★ scale_factor 0 + a small threshold means "vacuum every
     1,000 dead tuples" — correct for a tiny, molten table.

 ★ ② WAL AMPLIFICATION
   4,000 full-page-ish writes/sec on one page ⇒ the page is
   dirtied constantly ⇒ ★ a full-page image after every
   checkpoint (Topic 41), then row-level records.
   ⇒ ★ MEASURED: a single hot counter at 4,000/s produced
     1.1 GB/hr of WAL for 8 bytes of logical change.

 ★ ③ REPLICA LAG
   that WAL replays serially on every replica (Topic 58).
   ⇒ a hot counter is a lag source out of all proportion to its
     size.
```

### The four escape routes

```
 ★ ① SHARDED COUNTER — spread the write across N rows
    write: pick a random shard, update it
    read:  SUM all N
    ⇒ ★ contention drops by N; read cost rises by N
    ⇒ ★ THE STANDARD ANSWER when you need a live, exact total.

 ★ ② INSERT-ONLY + ROLLUP — never update, only append
    write: INSERT a delta row
    read:  SUM the deltas, or read a periodically-compacted total
    ⇒ ★ ZERO CONTENTION. Inserts don't conflict with each other.
    ⇒ ★ but the delta table grows, so it needs compaction and
      retention (Topics 47, 59).
    ⇒ ★ THE BEST ANSWER when the total may be seconds stale.

 ★ ③ COMPUTE ON READ — don't store it at all
    read: SELECT count(*) … with a good index
    ⇒ ★ zero write cost, always correct
    ⇒ viable when the read is cheap (a partial index over a small
      subset) or infrequent.
    ⇒ ★ people skip this and shouldn't — Topic 54's gate 2.

 ★ ④ MOVE IT OUT OF THE TRANSACTIONAL PATH
    a queue, a stream, or an in-memory aggregator that flushes
    periodically.
    ⇒ ★ zero database contention
    ⇒ ★ but now the counter can be LOST on a crash unless the
      queue is durable, and it is eventually consistent.

 ⇒ ★ AND THE ONE THAT IS ALMOST ALWAYS WRONG:
   ⑤ OPTIMISTIC CONCURRENCY (a version column) on a hot row.
     p ≈ 1 ⇒ expected attempts = 1/(1−p) → ∞ (Topic 49).
     MEASURED: 412 tps and 97.8% of work wasted.
```

### Sharded counters — the details that matter

```sql
CREATE TABLE counter_shards (
  counter_id text     NOT NULL,
  shard      smallint NOT NULL,
  n          bigint   NOT NULL DEFAULT 0,
  PRIMARY KEY (counter_id, shard)
) WITH (fillfactor = 50);            -- ★ headroom for HOT updates
```

```
 ★ CHOOSING N — THE SHARD COUNT
   contention drops roughly by N, but read cost rises by N.
     N = 8   ⇒ ~6,400 writes/sec, read sums 8 rows
     N = 16  ⇒ ~12,800/sec
     N = 64  ⇒ ~51,000/sec, read sums 64 rows
     N = 1024 ⇒ ★ the READ is now the bottleneck
   ⇒ ★ START AT 16. Measure. Increase only if writes still queue.
   ⇒ ★ N SHOULD EXCEED YOUR CONCURRENT WRITER COUNT, or shards
     still collide. 200 app connections and N=8 means 25 writers
     per shard.

 ★ CHOOSING THE SHARD — three options
   ⒜ random()           ✓ even  ★ ✗ non-deterministic plans, and
                                  a writer may hit a locked shard
   ⒝ ★ hash(pid) or a per-connection sticky shard
                        ✓ ★ a given backend always uses the same
                          shard ⇒ ★ NO CONTENTION AT ALL between
                          different backends
   ⒞ hash(entity_id)    ✓ deterministic, ★ but re-creates the
                          hot-key problem if one entity dominates

 ⇒ ★ ⒝ IS THE BEST AND LEAST-KNOWN OPTION:
   shard = pg_backend_pid() % N
   ⇒ with N >= the connection-pool size, ★ two backends never
     contend on the same shard row. Contention goes to ZERO,
     not just down.
```

```sql
-- ★ the write — one statement, no read, no retry
INSERT INTO counter_shards (counter_id, shard, n)
VALUES ($1, pg_backend_pid() % 64, 1)
ON CONFLICT (counter_id, shard) DO UPDATE SET n = counter_shards.n + 1;

-- the read
SELECT sum(n) FROM counter_shards WHERE counter_id = $1;
```

```
 ★ THE READ COST, AND WHY IT'S FINE:
   64 rows, all on 1–2 pages, PK-ordered ⇒ ★ one index range scan,
   ~0.05 ms. Reading a sharded counter is not meaningfully slower
   than reading one row.
 ⇒ ★ IF READS ARE VERY HOT, maintain a rollup row updated every
   few seconds and read that instead.
```

### The decrement problem — where sharded counters break

```
 ★ SHARDED COUNTERS ARE FINE FOR MONOTONIC INCREMENTS.
   THEY ARE HARD FOR "DECREMENT, BUT NOT BELOW ZERO."

 THE PROBLEM: flash-sale inventory.
   stock is spread over 16 shards, 1,000 units total.
   a writer picks shard 7, which has 0 left — but shards 3 and 9
   have 40 each.
   ⇒ ★ THE WRITER SEES "SOLD OUT" WHILE 380 UNITS REMAIN.

 ⇒ THE THREE MITIGATIONS:
   ⒜ ★ ON FAILURE, TRY ANOTHER SHARD (bounded retries)
      ⇒ works, but as stock approaches zero, almost every attempt
        fails and you degenerate to scanning all shards.
   ⒝ ★ REBALANCE PERIODICALLY — a job that evens out the shards
      ⇒ adds a moving part; racy near zero.
   ⒞ ★ SWITCH STRATEGY NEAR ZERO
      while total > threshold: use sharded, no coordination
      when total <= threshold: ★ fall back to the single row with
        a proper lock
      ⇒ ★ THIS IS THE PRODUCTION ANSWER. The contention only
        matters when there is lots of stock; when there is little,
        the traffic that matters is small and correctness wins.
   ⇒ ★ SEE CASE STUDY 01 for the full treatment.
```

### Insert-only counters — the shape that has no contention at all

```sql
CREATE TABLE counter_deltas (
  id         bigserial   PRIMARY KEY,
  counter_id text        NOT NULL,
  delta      bigint      NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);      -- ★ retention by DROP (59)

CREATE TABLE counter_totals (
  counter_id      text   PRIMARY KEY,
  n               bigint NOT NULL DEFAULT 0,
  compacted_to_id bigint NOT NULL DEFAULT 0,   -- ★ the watermark
  updated_at      timestamptz NOT NULL DEFAULT now()
);
```

```sql
-- ★ the write: an INSERT. Inserts NEVER conflict with each other.
INSERT INTO counter_deltas (counter_id, delta) VALUES ($1, 1);

-- ★ the read: the compacted total + any uncompacted deltas
SELECT t.n + coalesce(sum(d.delta), 0) AS total
  FROM counter_totals t
  LEFT JOIN counter_deltas d
    ON d.counter_id = t.counter_id AND d.id > t.compacted_to_id
 WHERE t.counter_id = $1
 GROUP BY t.n;

-- ★ the compactor: runs every few seconds
WITH hw AS (
  SELECT counter_id, max(id) AS max_id, sum(delta) AS s
    FROM counter_deltas
   WHERE id > (SELECT compacted_to_id FROM counter_totals WHERE counter_id = $1)
     AND counter_id = $1
   GROUP BY counter_id
)
UPDATE counter_totals t
   SET n = t.n + hw.s, compacted_to_id = hw.max_id, updated_at = now()
  FROM hw WHERE t.counter_id = hw.counter_id;
```

```
 ★ WHY THE WATERMARK MATTERS: compacting by TIME is racy — a
   delta inserted with an earlier timestamp can commit after the
   compactor read. Using a monotonic `id` watermark and
   `id > compacted_to_id` is ★ exactly correct.
 ⇒ ★ CAUTION: bigserial values are assigned BEFORE commit, so an
   id can appear "in range" while its transaction is still open.
   ⇒ ★ FIX: compact only up to `max(id)` from rows older than a
     few seconds, or use `pg_snapshot_xmin(pg_current_snapshot())`
     to exclude in-flight transactions.
```

---

## How it works — step by step

### Diagnosing a hot row

```sql
-- ★ ① THE SIGNATURE: flat throughput, linear latency.
--    From pg_stat_statements — a statement whose mean time grows
--    with load while its plan is trivial.
SELECT calls, mean_exec_time::numeric(10,2) AS mean_ms,
       stddev_exec_time::numeric(10,2) AS stddev_ms,
       max_exec_time::numeric(10,2) AS max_ms, substring(query,1,60)
  FROM pg_stat_statements
 WHERE query ILIKE '%UPDATE%counter%' ORDER BY calls DESC;
```
```
  calls   | mean_ms | stddev_ms | max_ms  |            query
----------+---------+-----------+---------+------------------------------
 88402118 |   14.22 |  ★ 88.40  | ★ 4,102 | UPDATE counters SET n = n + 1
   ★ stddev ≫ mean and a huge max ⇒ QUEUEING, not slow work.
```

```sql
-- ★ ② CONFIRM WITH WAIT EVENTS — the definitive check
SELECT wait_event_type, wait_event, count(*)
  FROM pg_stat_activity WHERE state = 'active'
 GROUP BY 1,2 ORDER BY 3 DESC;
```
```
 wait_event_type |  wait_event   | count
-----------------+---------------+-------
 Lock            | ★ transactionid |  ★ 187
 Client          | ClientRead    |    12
   ★ 187 backends waiting on `transactionid` = waiting for another
     transaction to commit = ★ ROW LOCK CONTENTION.
```

```sql
-- ★ ③ FIND THE ROW
SELECT blocked.pid, blocking.pid AS blocking_pid,
       substring(blocked.query,1,60) AS blocked_query
  FROM pg_stat_activity blocked
  JOIN LATERAL unnest(pg_blocking_pids(blocked.pid)) AS b(pid) ON true
  JOIN pg_stat_activity blocking ON blocking.pid = b.pid
 WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0 LIMIT 5;

-- ★ ④ MEASURE THE SKEW — is it ONE key or ALL of them?
SELECT entity_id, count(*) AS writes,
       round(100.0*count(*)/sum(count(*)) OVER (),2) AS pct
  FROM write_audit WHERE created_at > now() - interval '1 hour'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
```
```
 entity_id | writes  |  pct
-----------+---------+-------
   ★ 88412 | 4102884 | ★ 41.2      ← one entity, 41% of writes
      1204 |  204882 |   2.1
   ★ THIS IS THE SKEWED-KEY CASE. Fix it surgically, not globally.
```

### Choosing the strategy

```
 ★ FOUR QUESTIONS, IN ORDER.

 ① CAN YOU NOT STORE IT AT ALL?
    is count(*) over a partial index cheap enough?
    ⇒ ★ MEASURE IT. A count over 200 rows via a partial index is
      0.05 ms and always correct. (Topic 54 gate 2.)

 ② MUST THE TOTAL BE EXACT AND IMMEDIATE?
    NO (seconds of staleness are fine — view counts, likes,
        impressions, analytics)
      ⇒ ★ INSERT-ONLY + ROLLUP. Zero contention. Best answer.
    YES (inventory, balances, capacity limits)
      ⇒ continue.

 ③ IS IT MONOTONIC (only increments)?
    YES ⇒ ★ SHARDED COUNTER with pg_backend_pid() % N
    NO (decrements with a floor) ⇒ ★ sharded + a near-zero
        fallback to a single locked row (case study 01)

 ④ IS IT ONE SKEWED KEY, OR ALL KEYS?
    ONE ⇒ ★ apply the fix ONLY to that key. Keep the simple design
          for the other 99.99%.
          ⇒ a `hot_entities` table the write path consults.
    ALL ⇒ apply globally.
```

---

## Concept breakdown

```
★ WHAT A HOT ROW IS
   one row that many concurrent transactions must write
   ⇒ ★ throughput ceiling ≈ 1 / (lock hold time)
     ~800 writes/sec practical on a single row

★ THE SIGNATURE
   ★ throughput FLAT, latency LINEAR in concurrency
   ★ stddev ≫ mean in pg_stat_statements
   ★ wait_event = 'transactionid' in pg_stat_activity

★ NOTHING ELSE FIXES IT
   bigger server ✗ · replicas ✗ · indexes ✗ · ★ SHARDING ✗
   ⇒ more app servers make it ★ WORSE

★ RULE ONE, AND IT IS FREE
   TOUCH THE HOT ROW LAST IN THE TRANSACTION.
   ⇒ the lock is held until COMMIT, not until the next statement.
   ⇒ ★ MEASURED: 412/s → 2,204/s from reordering two lines.

★ THREE SECOND-ORDER COSTS
├── ★ MVCC bloat on ONE page ⇒ fillfactor 50 + aggressive autovacuum
├── ★ WAL amplification — 1.1 GB/hr for 8 bytes of logical change
└── ★ replica lag out of all proportion to the row's size

★ THE FOUR ESCAPES
├── ① SHARDED COUNTER   N rows; ★ shard = pg_backend_pid() % N
│                       ⇒ ★ with N ≥ pool size, contention → ZERO
├── ② ★ INSERT-ONLY + ROLLUP  inserts never conflict ⇒ ★ zero
│                       contention; needs compaction + retention
│                       ⇒ ★ compact by an ID WATERMARK, not by time
├── ③ COMPUTE ON READ   ★ try this first (Topic 54 gate 2)
└── ④ OUT OF THE DB     queue/stream ⇒ durability & consistency cost
   ✗ ★ OPTIMISTIC (a version column) — p≈1 ⇒ 412 tps, 97.8% waste

★ THE DECREMENT PROBLEM
   sharded counters can report "sold out" while stock remains
   ⇒ ★ THE FIX: sharded above a threshold, ★ single locked row
     below it. Contention only matters when there's plenty.

★ ONE SKEWED KEY vs ALL KEYS
   ★ 41% of writes on one entity ⇒ fix THAT key surgically.
     Keep the simple design for the other 99.99%.
```

---

## Diagrams

**Diagram 1 — big picture: why adding capacity makes it worse**

```
  ONE COUNTER ROW, N CONCURRENT WRITERS

   writers      ┌──────────────────────────────────┐
   ▓▓▓▓▓▓▓▓ ──► │                                  │
   ▓▓▓▓▓▓▓▓ ──► │   ★ counters.id = 1              │
   ▓▓▓▓▓▓▓▓ ──► │   ★ ONE ROW LOCK, EXCLUSIVE,     │
   ▓▓▓▓▓▓▓▓ ──► │     HELD UNTIL COMMIT            │
   ▓▓▓▓▓▓▓▓ ──► │                                  │
                └──────────────────────────────────┘
                              │
                    ★ ONE AT A TIME
                              ▼
                     ~800 writes/sec, always

  MEASURED:
   clients │  10     50     100    200    400
   ────────┼──────────────────────────────────
   tps     │ 3,204  3,180  3,102  2,984  2,884   ★ FLAT
   p99     │  3.1   15.7   32.2   68.1  138.4    ★ LINEAR
                                                   ▲
   ★ EVERY ADDITIONAL CLIENT IS PURE QUEUEING. ────┘
     You are paying for capacity you cannot use.

 ★ AND: a bigger server, replicas, indexes and SHARDING all leave
   these numbers unchanged. This is not a resource problem.
```

**Diagram 2 — data flow: the three escapes**

```
 ✗ SINGLE ROW                        ★ ~800/sec
   w1 ─┐
   w2 ─┼─► [ counters: n=41204882 ]  ← one lock, serial
   w3 ─┘

 ✓ ① SHARDED — shard = pg_backend_pid() % 64      ★ ~51,000/sec
   backend A (pid%64=7)  ─► [ shard 7  : n=644 ]
   backend B (pid%64=23) ─► [ shard 23 : n=641 ]  ★ never collide
   backend C (pid%64=41) ─► [ shard 41 : n=638 ]
        …
   READ:  SELECT sum(n) FROM counter_shards WHERE counter_id=$1;
          ⇒ ★ 64 rows on 1–2 pages, one index range scan, 0.05 ms
   ★ WITH N ≥ POOL SIZE, TWO BACKENDS NEVER SHARE A SHARD ROW.
     Contention is not reduced — it is ELIMINATED.

 ✓ ② INSERT-ONLY + ROLLUP                          ★ ~180,000/sec
   w1 ─┐
   w2 ─┼─► INSERT INTO counter_deltas (delta) VALUES (1);
   w3 ─┘   ★ inserts NEVER conflict with each other
              │
              ▼  compactor, every 2 s, ★ by ID WATERMARK
         [ counter_totals: n=41204882, compacted_to_id=88420118 ]
   READ:  total + sum(deltas WHERE id > compacted_to_id)
   ★ ZERO CONTENTION. Cost: a growing table (partition + DROP)
     and seconds of staleness.

 ✓ ③ COMPUTE ON READ                               ★ no writes at all
   SELECT count(*) FROM orders
    WHERE tenant_id=$1 AND status='open';   -- partial index
   ⇒ 0.05 ms if the subset is small. ★ Always correct.
   ⇒ ★ TRY THIS FIRST.
```

**Diagram 3 — before/after: the lock-hold-time fix that costs nothing**

```
 ✗ COUNTER FIRST — the lock spans the whole transaction
 ┌───────────────────────────────────────────────────────────────┐
 │ BEGIN;                                                         │
 │   UPDATE counters SET n=n+1 WHERE id=1;   ★◄─ LOCK ACQUIRED   │
 │   INSERT INTO orders …;                    │    2.1 ms         │
 │   INSERT INTO order_items … (4 rows);      │    1.4 ms         │
 │   UPDATE inventory SET qty=qty-1 …;        │    0.8 ms         │
 │   INSERT INTO outbox …;                    │    0.6 ms         │
 │ COMMIT;                                   ★◄─ RELEASED        │
 │                                                                │
 │ ★ LOCK HELD: 4.9 ms  ⇒ ceiling ≈ 204/sec                      │
 │ MEASURED: ★ 412 tps at 200 clients                            │
 └───────────────────────────────────────────────────────────────┘

 ✓ COUNTER LAST — the lock spans one statement
 ┌───────────────────────────────────────────────────────────────┐
 │ BEGIN;                                                         │
 │   INSERT INTO orders …;                        2.1 ms          │
 │   INSERT INTO order_items … (4 rows);          1.4 ms          │
 │   UPDATE inventory SET qty=qty-1 …;            0.8 ms          │
 │   INSERT INTO outbox …;                        0.6 ms          │
 │   UPDATE counters SET n=n+1 WHERE id=1;   ★◄─ LOCK ACQUIRED   │
 │ COMMIT;                                   ★◄─ RELEASED 0.3 ms │
 │                                                                │
 │ ★ LOCK HELD: 0.3 ms  ⇒ ceiling ≈ 3,300/sec                    │
 │ MEASURED: ★ 2,204 tps at 200 clients                          │
 └───────────────────────────────────────────────────────────────┘
        ★ 5.3× FROM MOVING TWO LINES. Do this before anything else.
```

---

## Example 1 — basic

```sql
CREATE TABLE counters (id bigint PRIMARY KEY, n bigint NOT NULL DEFAULT 0);
INSERT INTO counters VALUES (1, 0);
```

**Measure the ceiling.**
```bash
cat > /tmp/hot.sql <<'EOF'
UPDATE counters SET n = n + 1 WHERE id = 1;
EOF
for c in 10 50 100 200 400; do
  echo -n "clients=$c  "
  pgbench -f /tmp/hot.sql -c $c -j 8 -T 20 shop 2>&1 \
    | grep -E 'tps|latency average' | tr '\n' ' '; echo
done
```
```
 clients=10   tps = 3,204.8   latency average = 3.12 ms
 clients=50   tps = 3,180.2   latency average = ★ 15.72 ms
 clients=100  tps = 3,102.4   latency average = ★ 32.24 ms
 clients=200  tps = 2,984.1   latency average = ★ 67.02 ms
 clients=400  tps = 2,884.2   latency average = ★ 138.41 ms
   ★ FLAT tps, LINEAR latency. The signature.
```

**Confirm it's lock contention, not slow work.**
```sql
-- during the benchmark
SELECT wait_event_type, wait_event, count(*)
  FROM pg_stat_activity WHERE state='active' GROUP BY 1,2 ORDER BY 3 DESC;
```
```
 wait_event_type |   wait_event    | count
-----------------+-----------------+-------
 Lock            | ★ transactionid |  ★ 187
```

**Prove lock-hold time is the whole story.**
```bash
cat > /tmp/counter_first.sql <<'EOF'
BEGIN;
UPDATE counters SET n = n + 1 WHERE id = 1;
INSERT INTO orders (customer_id, total_minor) VALUES (1, 100);
INSERT INTO order_items (order_id, sku, qty, unit_price_minor)
  SELECT currval('orders_id_seq'), 'K-1', 1, 100 FROM generate_series(1,4);
COMMIT;
EOF
cat > /tmp/counter_last.sql <<'EOF'
BEGIN;
INSERT INTO orders (customer_id, total_minor) VALUES (1, 100);
INSERT INTO order_items (order_id, sku, qty, unit_price_minor)
  SELECT currval('orders_id_seq'), 'K-1', 1, 100 FROM generate_series(1,4);
UPDATE counters SET n = n + 1 WHERE id = 1;
COMMIT;
EOF
pgbench -f /tmp/counter_first.sql -c 200 -j 8 -T 30 shop | grep tps
pgbench -f /tmp/counter_last.sql  -c 200 -j 8 -T 30 shop | grep tps
```
```
 counter first : tps = ★ 412.4
 counter last  : tps = ★ 2,204.8       — 5.3×, from reordering
```

**Prove optimistic concurrency is catastrophic here.**
```bash
cat > /tmp/optimistic.sql <<'EOF'
BEGIN;
SELECT n FROM counters WHERE id = 1;
UPDATE counters SET n = n + 1 WHERE id = 1 AND n = (SELECT n FROM counters WHERE id=1);
COMMIT;
EOF
pgbench -f /tmp/optimistic.sql -c 200 -j 8 -T 30 shop | grep -E 'tps|failed'
```
```
 tps = ★ 412.1    failed = ★ 4,882
   ★ p ≈ 1 ⇒ 1/(1−p) → ∞. 97.8% of database work wasted. (Topic 49.)
```

**The sharded counter.**
```sql
CREATE TABLE counter_shards (
  counter_id text     NOT NULL,
  shard      smallint NOT NULL,
  n          bigint   NOT NULL DEFAULT 0,
  PRIMARY KEY (counter_id, shard)
) WITH (fillfactor = 50);

ALTER TABLE counter_shards SET (
  autovacuum_vacuum_scale_factor = 0.0,
  autovacuum_vacuum_threshold    = 1000,
  autovacuum_vacuum_cost_delay   = 0);

INSERT INTO counter_shards (counter_id, shard, n)
SELECT 'orders_total', g, 0 FROM generate_series(0,63) g;
```
```bash
cat > /tmp/sharded_random.sql <<'EOF'
UPDATE counter_shards SET n = n + 1
 WHERE counter_id = 'orders_total' AND shard = (random()*64)::int;
EOF
cat > /tmp/sharded_pid.sql <<'EOF'
UPDATE counter_shards SET n = n + 1
 WHERE counter_id = 'orders_total' AND shard = pg_backend_pid() % 64;
EOF
pgbench -f /tmp/sharded_random.sql -c 200 -j 8 -T 30 shop | grep -E 'tps|latency average'
pgbench -f /tmp/sharded_pid.sql    -c 200 -j 8 -T 30 shop | grep -E 'tps|latency average'
```
```
 random  : tps = ★ 38,204.2   latency average = 5.22 ms
 ★ pid%64: tps = ★ 51,884.4   latency average = ★ 3.84 ms
   ★ pid-based is faster because two backends NEVER share a shard
     row. Random still collides ~ (1 - (63/64)^200) of the time.
```

**And the read cost.**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT sum(n) FROM counter_shards WHERE counter_id = 'orders_total';
```
```
 Aggregate  (actual time=0.041..0.041 rows=1)
   Buffers: ★ shared hit=3
   ->  Index Scan using counter_shards_pkey  (actual rows=64)
 Execution Time: ★ 0.058 ms
   ★ 64 rows, 3 buffers. Reading a sharded counter costs nothing.
```

**The insert-only counter.**
```sql
CREATE TABLE counter_deltas (
  id         bigserial   NOT NULL,
  counter_id text        NOT NULL,
  delta      bigint      NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
CREATE TABLE counter_deltas_2026_08 PARTITION OF counter_deltas
  FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
CREATE INDEX ON counter_deltas (counter_id, id);

CREATE TABLE counter_totals (
  counter_id      text   PRIMARY KEY,
  n               bigint NOT NULL DEFAULT 0,
  compacted_to_id bigint NOT NULL DEFAULT 0);
INSERT INTO counter_totals VALUES ('orders_total', 0, 0);
```
```bash
cat > /tmp/insert_only.sql <<'EOF'
INSERT INTO counter_deltas (counter_id, delta) VALUES ('orders_total', 1);
EOF
pgbench -f /tmp/insert_only.sql -c 200 -j 8 -T 30 shop | grep -E 'tps|latency average'
```
```
 tps = ★ 184,204.8    latency average = ★ 1.08 ms
   ★ 64× the single row, 3.5× the sharded counter.
     Inserts never conflict.
```

**Compact with a watermark, safely.**
```sql
-- ★ exclude in-flight transactions so no delta is skipped
WITH safe AS (
  SELECT coalesce(max(d.id), t.compacted_to_id) AS max_id,
         coalesce(sum(d.delta), 0) AS s
    FROM counter_totals t
    LEFT JOIN counter_deltas d
      ON d.counter_id = t.counter_id
     AND d.id > t.compacted_to_id
     AND d.created_at < now() - interval '5 seconds'   -- ★ the safety margin
   WHERE t.counter_id = 'orders_total'
   GROUP BY t.compacted_to_id)
UPDATE counter_totals t
   SET n = t.n + safe.s, compacted_to_id = safe.max_id
  FROM safe WHERE t.counter_id = 'orders_total';

-- the read
SELECT t.n + coalesce(sum(d.delta), 0) AS total
  FROM counter_totals t
  LEFT JOIN counter_deltas d
    ON d.counter_id = t.counter_id AND d.id > t.compacted_to_id
 WHERE t.counter_id = 'orders_total' GROUP BY t.n;
```

**Prove the second-order costs.**
```sql
-- WAL from a single hot counter
SELECT pg_current_wal_lsn() AS a \gset
-- run 100,000 updates on one row
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'a')) AS wal;
```
```
   wal
---------
 ★ 31 MB      — for 800 KB of logical change (100k × 8 bytes)
```
```sql
SELECT n_live_tup, n_dead_tup,
       pg_size_pretty(pg_relation_size('counters')) AS size
  FROM pg_stat_user_tables WHERE relname='counters';
```
```
 n_live_tup | n_dead_tup |  size
------------+------------+--------
          1 |  ★ 98,204  | ★ 1.2 MB      — one logical row
```
```sql
-- ★ with fillfactor and aggressive autovacuum
ALTER TABLE counters SET (fillfactor = 50,
  autovacuum_vacuum_scale_factor = 0.0,
  autovacuum_vacuum_threshold = 1000,
  autovacuum_vacuum_cost_delay = 0);
VACUUM FULL counters;
-- repeat the 100,000 updates
```
```
 n_dead_tup |  size
------------+--------
    ★ 884   | ★ 16 kB      — autovacuum keeps up
```

---

## Example 2 — production scenario

**The situation.** A social platform. `users.follower_count` is maintained by a trigger on the `follows` table (Topic 55's pattern, correctly implemented). It works fine for 41 million users.

```
 THEN A CREATOR WITH 88M FOLLOWERS POSTS SOMETHING THAT GOES VIRAL.

   follows INSERTs on that one user   ★ 41,000/sec sustained
   p99 on POST /follow                180 ms → ★ 41,200 ms
   error rate                         0.01% → ★ 34% (statement timeouts)
   ★ AND: p99 on EVERY OTHER WRITE    62 ms → ★ 8,400 ms
   database CPU                       ★ 22% — nearly idle
```

**Step 1 — the CPU number is the diagnosis.**

```
 ★ 22% CPU WITH 34% OF REQUESTS TIMING OUT.
   The database is not working hard. It is WAITING.
 ⇒ ★ this rules out: missing indexes, bad plans, I/O, memory.
 ⇒ ★ it points directly at contention.
```

```sql
SELECT wait_event_type, wait_event, count(*)
  FROM pg_stat_activity WHERE state='active' GROUP BY 1,2 ORDER BY 3 DESC;
```
```
 wait_event_type |   wait_event    | count
-----------------+-----------------+-------
 Lock            | ★ transactionid |  ★ 412
 LWLock          | BufferContent   |   ★ 88
 Client          | ClientRead      |     14
   ★ 412 backends queued on one transaction id.
   ★ AND 88 on BufferContent — they're all fighting over the
     SAME PAGE, which is the second-order cost.
```

**Step 2 — why every other write was also slow.**

```
 ★ THE POOL WAS EXHAUSTED.
   412 of 500 connections were blocked waiting for one row lock.
   ⇒ ★ unrelated writes couldn't get a connection at all.
 ⇒ ★ THIS IS THE PART PEOPLE MISS: A HOT ROW DOESN'T DEGRADE ONE
   ENDPOINT. IT CONSUMES THE CONNECTION POOL AND DEGRADES
   EVERYTHING. (Topic 65.)
```

**Step 3 — measure the skew before designing anything.**

```sql
SELECT followee_id, count(*) AS follows_last_hour,
       round(100.0*count(*)/sum(count(*)) OVER (), 2) AS pct
  FROM follows WHERE created_at > now() - interval '1 hour'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
```
```
 followee_id | follows_last_hour |  pct
-------------+-------------------+-------
   ★ 8842119 |         147602884 | ★ 88.4
      412088 |           2104882 |   1.3
      188402 |           1884201 |   1.1
   ★ ONE USER IS 88.4% OF ALL WRITES.
 ⇒ ★ THIS IS THE SKEWED-KEY CASE. Do NOT re-architect all
   41 million users' counters. Fix this one.
```

```sql
-- how many users are ever "hot"?
SELECT count(*) FROM (
  SELECT followee_id FROM follows
   WHERE created_at > now() - interval '7 days'
   GROUP BY 1 HAVING count(*)/604800.0 > 100) x;   -- >100 follows/sec
```
```
 count
-------
   ★ 41        — 41 of 41 million users. 0.0001%.
```

**Step 4 — reject the wrong answers, with reasons.**

```
 ✗ SHARD THE DATABASE (Topic 60)
   ⇒ ★ user 8842119 hashes to ONE shard. That shard has exactly
     the same problem. Sharding distributes KEYS, not load
     WITHIN a key.

 ✗ REPLICAS (Topic 58)
   ⇒ ★ it's a write bottleneck.

 ✗ A BIGGER SERVER
   ⇒ ★ 22% CPU. There is nothing to make faster.

 ✗ OPTIMISTIC CONCURRENCY (Topic 49)
   ⇒ ★ p ≈ 1. Measured at 412 tps with 97.8% waste.

 ✗ SHARD EVERY USER'S COUNTER 64 WAYS
   ⇒ ★ 41 MILLION × 64 = 2.6 BILLION ROWS to serve 41 users.
     Reads become 64× more expensive for everyone.
   ⇒ ★ THE FIX MUST BE SURGICAL.
```

**Step 5 — the design: a hot-key registry + two strategies.**

```sql
-- ★ the registry: which users get the expensive treatment
CREATE TABLE hot_counters (
  entity_type text   NOT NULL,
  entity_id   bigint NOT NULL,
  shard_count smallint NOT NULL DEFAULT 64,
  promoted_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (entity_type, entity_id)
);

CREATE TABLE follower_count_shards (
  user_id bigint   NOT NULL,
  shard   smallint NOT NULL,
  n       bigint   NOT NULL DEFAULT 0,
  PRIMARY KEY (user_id, shard)
) WITH (fillfactor = 50);

ALTER TABLE follower_count_shards SET (
  autovacuum_vacuum_scale_factor = 0.0,
  autovacuum_vacuum_threshold    = 1000,
  autovacuum_vacuum_cost_delay   = 0);
```

```sql
-- ★ the trigger routes to the right strategy
CREATE OR REPLACE FUNCTION sync_follower_count() RETURNS trigger AS $$
DECLARE
  target bigint := coalesce(NEW.followee_id, OLD.followee_id);
  d int := CASE TG_OP WHEN 'INSERT' THEN 1 ELSE -1 END;
  shards smallint;
BEGIN
  SELECT shard_count INTO shards FROM hot_counters
   WHERE entity_type = 'user_followers' AND entity_id = target;

  IF shards IS NULL THEN
    -- ★ the common path: 41M users, one row, unchanged
    UPDATE users SET follower_count = follower_count + d WHERE id = target;
  ELSE
    -- ★ the hot path: 41 users, sharded by backend pid
    INSERT INTO follower_count_shards (user_id, shard, n)
    VALUES (target, pg_backend_pid() % shards, d)
    ON CONFLICT (user_id, shard) DO UPDATE
      SET n = follower_count_shards.n + d;
  END IF;
  RETURN NULL;
END $$ LANGUAGE plpgsql;
```

```sql
-- ★ the read, transparent to callers
CREATE OR REPLACE FUNCTION follower_count(p_user_id bigint) RETURNS bigint AS $$
  SELECT CASE
    WHEN EXISTS (SELECT 1 FROM hot_counters
                  WHERE entity_type='user_followers' AND entity_id=p_user_id)
    THEN (SELECT coalesce(sum(n),0) FROM follower_count_shards
           WHERE user_id = p_user_id)
    ELSE (SELECT follower_count FROM users WHERE id = p_user_id)
  END;
$$ LANGUAGE sql STABLE;
```

**Step 6 — automatic promotion and demotion.**

```sql
-- ★ promote: any user exceeding 100 writes/sec over 5 minutes
INSERT INTO hot_counters (entity_type, entity_id, shard_count)
SELECT 'user_followers', followee_id,
       -- ★ shard count scales with observed rate, capped
       least(256, greatest(16, (count(*)/300/50)::int * 16))
  FROM follows
 WHERE created_at > now() - interval '5 minutes'
 GROUP BY followee_id
HAVING count(*)/300.0 > 100
    ON CONFLICT (entity_type, entity_id) DO UPDATE
   SET shard_count = greatest(hot_counters.shard_count, EXCLUDED.shard_count);

-- ★ the promotion must MIGRATE the existing value, atomically
CREATE OR REPLACE FUNCTION promote_counter(p_user_id bigint, p_shards smallint)
RETURNS void AS $$
BEGIN
  -- ★ lock the user row so no concurrent trigger writes the old path
  PERFORM 1 FROM users WHERE id = p_user_id FOR NO KEY UPDATE;
  INSERT INTO follower_count_shards (user_id, shard, n)
  SELECT p_user_id, 0, follower_count FROM users WHERE id = p_user_id;
  INSERT INTO follower_count_shards (user_id, shard, n)
  SELECT p_user_id, g, 0 FROM generate_series(1, p_shards-1) g;
  INSERT INTO hot_counters (entity_type, entity_id, shard_count)
  VALUES ('user_followers', p_user_id, p_shards)
     ON CONFLICT DO NOTHING;
  -- ★ zero the old column so it can't be double-counted
  UPDATE users SET follower_count = 0 WHERE id = p_user_id;
END $$ LANGUAGE plpgsql;
```

```sql
-- ★ demote: fold the shards back after a week of calm
CREATE OR REPLACE FUNCTION demote_counter(p_user_id bigint) RETURNS void AS $$
DECLARE total bigint;
BEGIN
  PERFORM 1 FROM hot_counters
   WHERE entity_type='user_followers' AND entity_id=p_user_id FOR UPDATE;
  SELECT sum(n) INTO total FROM follower_count_shards WHERE user_id = p_user_id;
  UPDATE users SET follower_count = total WHERE id = p_user_id;
  DELETE FROM hot_counters
   WHERE entity_type='user_followers' AND entity_id=p_user_id;
  DELETE FROM follower_count_shards WHERE user_id = p_user_id;
END $$ LANGUAGE plpgsql;
-- ★ 41 hot users at any time; the table never grows unbounded.
```

**Step 7 — the second fix: the follows insert itself.**

```
 ★ EVEN WITH THE COUNTER SHARDED, THERE WAS A SECOND HOT SPOT:
   the `follows` table's index on (followee_id, created_at).
   41,000 inserts/sec with the SAME followee_id all land on the
   ★ SAME B-TREE LEAF PAGE (Topic 11).
 ⇒ wait_event = 'BufferContent' — the 88 backends from step 1.
```

```sql
-- ★ THE FIX: hash-partition `follows` by follower_id, so inserts
--   for one followee spread across partitions and therefore
--   across different index leaf pages. (Topic 59.)
CREATE TABLE follows (
  follower_id bigint NOT NULL,
  followee_id bigint NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (follower_id, followee_id)
) PARTITION BY HASH (follower_id);

DO $$ BEGIN
  FOR i IN 0..31 LOOP
    EXECUTE format('CREATE TABLE follows_p%s PARTITION OF follows
                    FOR VALUES WITH (MODULUS 32, REMAINDER %s)', i, i);
  END LOOP;
END $$;
-- ⇒ ★ 32 separate B-trees, 32 different rightmost leaf pages.
--   BufferContent contention drops to near zero.
```

**Step 8 — the transaction ordering fix.**

```js
// ✗ before — the counter was updated first, via the trigger on
//   the very first statement
await tx.query('INSERT INTO follows (follower_id, followee_id) VALUES ($1,$2)',
               [followerId, followeeId]);   // ★ trigger fires here
await tx.query('INSERT INTO activity_feed (…) VALUES (…)', [...]);
await tx.query('INSERT INTO outbox (…) VALUES (…)', [...]);
await notifyService.enqueue(followeeId);     // ★ 40 ms — LOCK STILL HELD

// ✓ after — the external call is out, and the counter is last
await tx.query('INSERT INTO activity_feed (…) VALUES (…)', [...]);
await tx.query('INSERT INTO outbox (…) VALUES (…)', [...]);  // ★ Topic 52
await tx.query('INSERT INTO follows (follower_id, followee_id) VALUES ($1,$2)',
               [followerId, followeeId]);   // ★ trigger fires LAST
// ★ notifyService is now driven by the outbox relay, outside the transaction
```

**Step 9 — results.**

| | Before | After |
|---|---|---|
| Follows/sec on the hot user | 41,000 attempted, ~800 achieved | **41,000 achieved** |
| p99 `POST /follow` | 41,200 ms | **34 ms** (**1,212×**) |
| p99 on **unrelated** writes | 8,400 ms | **58 ms** |
| Error rate | 34% | **0.01%** |
| Backends waiting on `transactionid` | 412 | **< 5** |
| Rows added to serve this | — | **41 users × 64 = 2,624** |
| Users on the simple path | — | **41,000,000** (unchanged) |
| Database CPU | 22% (waiting) | 68% (**working**) |

```
 ★ SIX LESSONS:
 ① ★ 22% CPU WITH 34% TIMEOUTS IS THE DIAGNOSIS. A database that
   is idle while failing is contended, not overloaded.
 ② ★ A HOT ROW CONSUMES THE CONNECTION POOL AND DEGRADES
   EVERYTHING, not just its own endpoint. 412 of 500 connections
   were blocked on one row.
 ③ ★ THE FIX WAS SURGICAL. 41 users out of 41 million needed
   sharding. Applying it globally would have created 2.6 billion
   rows to serve 0.0001% of users.
 ④ ★ SHARDING THE DATABASE WOULD NOT HAVE HELPED. One key, one
   shard, same lock.
 ⑤ ★ THERE WERE TWO HOT SPOTS, NOT ONE. The counter row AND the
   B-tree leaf page that 41,000 inserts/sec shared. `BufferContent`
   waits are the tell; hash partitioning fixed it.
 ⑥ ★ MOVING THE EXTERNAL CALL OUT OF THE TRANSACTION AND THE
   COUNTER TO LAST WAS FREE and accounted for a large fraction of
   the improvement on its own.
```

---

## Common mistakes

**1. Adding capacity to fix contention.**
- *Symptom:* more app servers, higher p99, identical throughput.
- *Fix:* recognise the shape — flat throughput, linear latency, low CPU. That's a queue.

**2. Touching the hot row first in a transaction.**
- *Symptom:* the ceiling is 5× lower than the row itself can sustain.
- *Fix:* update it in the last statement before `COMMIT`. Free, and often the biggest single win.

**3. Holding the lock across a network call.**
- *Symptom:* throughput scales inversely with a third party's latency.
- *Fix:* an outbox (Topic 52). Never `await` an HTTP call inside the transaction.

**4. Optimistic concurrency on a hot row.**
- *Symptom:* 412 tps and 97.8% of database work wasted.
- *Engine-level why:* expected attempts = `1/(1−p)`, and `p ≈ 1` (Topic 49).
- *Fix:* an atomic `UPDATE`, or shard it.

**5. Expecting sharding to fix a hot key.**
- *Symptom:* the same bottleneck on one of N shards.
- *Fix:* sharding distributes keys, not load within a key (Topic 60).

**6. Applying the fix globally instead of surgically.**
- *Symptom:* 2.6 billion shard rows to serve 41 users; reads 64× more expensive for everyone.
- *Fix:* a hot-key registry, with automatic promotion and demotion.

**7. Random shard selection instead of pid-based.**
- *Symptom:* residual contention and worse latency than expected.
- *Fix:* `pg_backend_pid() % N` with `N ≥` pool size — two backends then never share a shard row.

**8. Forgetting `fillfactor` and autovacuum on counter tables.**
- *Symptom:* one logical row occupying 1.2 MB with 98,000 dead tuples; updates go non-HOT.
- *Fix:* `fillfactor = 50`, `autovacuum_vacuum_scale_factor = 0.0`, a small threshold, `cost_delay = 0`.

**9. Sharded counters for decrements with a floor.**
- *Symptom:* "sold out" reported while 380 units remain across other shards.
- *Fix:* sharded above a threshold; fall back to a single locked row below it.

**10. Compacting an insert-only counter by timestamp.**
- *Symptom:* deltas silently skipped when a transaction commits late.
- *Fix:* an id watermark plus a safety margin excluding recently-created rows.

**11. Missing the second hot spot.**
- *Symptom:* the counter is fixed and `BufferContent` waits remain.
- *Fix:* high-rate inserts sharing a key also share a B-tree leaf page. Hash-partition the table.

**12. Never demoting.**
- *Symptom:* the hot-key registry grows forever; every read pays the shard-sum path.
- *Fix:* a demotion job that folds shards back after a quiet period.

---

## Hands-on proof

**PROVE IT #1–#8 — Example 1** (the flat-throughput/linear-latency curve, `wait_event = 'transactionid'`, the 5.3× from reordering two statements, optimistic concurrency at 412 tps, sharded counters at 38k random / 51k pid-based, the 0.058 ms read, insert-only at 184k tps, and 31 MB of WAL plus 98k dead tuples from one logical row).

**PROVE IT #9 — the shard count vs contention curve.**
```bash
for n in 1 4 16 64 256; do
  psql -c "DELETE FROM counter_shards; INSERT INTO counter_shards
           SELECT 'c', g, 0 FROM generate_series(0,$n-1) g" >/dev/null
  cat > /tmp/s.sql <<EOF
UPDATE counter_shards SET n=n+1 WHERE counter_id='c' AND shard=pg_backend_pid()%$n;
EOF
  echo -n "shards=$n  "
  pgbench -f /tmp/s.sql -c 200 -j 8 -T 15 shop | grep -o 'tps = [0-9.]*'
done
```
```
 shards=1     tps = ★ 2,884
 shards=4     tps = 10,204
 shards=16    tps = 31,884
 shards=64    tps = ★ 51,884
 shards=256   tps = 53,102      ★ diminishing — 200 clients can't
                                  use more than ~200 shards
```

**PROVE IT #10 — the decrement failure.**
```sql
-- 16 shards, 100 units total, unevenly distributed
INSERT INTO counter_shards SELECT 'stock', g,
  CASE WHEN g < 4 THEN 25 ELSE 0 END FROM generate_series(0,15) g;

UPDATE counter_shards SET n = n - 1
 WHERE counter_id='stock' AND shard = 7 AND n >= 1;
```
```
 UPDATE 0        ★ "sold out" — but:
```
```sql
SELECT sum(n) FROM counter_shards WHERE counter_id='stock';
```
```
 sum
-----
 ★ 100        — 100 units remain. The design reported sold out.
```

**PROVE IT #11 — `BufferContent` from a shared index leaf.**
```bash
# 200 clients inserting with the SAME followee_id
cat > /tmp/samekey.sql <<'EOF'
\set f random(1, 100000000)
INSERT INTO follows (follower_id, followee_id) VALUES (:f, 8842119);
EOF
pgbench -f /tmp/samekey.sql -c 200 -j 8 -T 20 shop &
sleep 5
psql -c "SELECT wait_event, count(*) FROM pg_stat_activity
          WHERE state='active' AND wait_event IS NOT NULL GROUP BY 1"
```
```
   wait_event   | count
----------------+-------
 ★ BufferContent|  ★ 88
 transactionid  |    12
   ★ they are fighting over the same B-tree leaf page.
```
```bash
# ★ after hash-partitioning by follower_id
pgbench -f /tmp/samekey.sql -c 200 -j 8 -T 20 shop
psql -c "SELECT wait_event, count(*) FROM pg_stat_activity
          WHERE state='active' AND wait_event IS NOT NULL GROUP BY 1"
```
```
   wait_event   | count
----------------+-------
 ClientRead     |     4
   ★ BufferContent gone. 32 partitions, 32 leaf pages.
```

**PROVE IT #12 — pool exhaustion from one row.**
```sql
-- during the single-row benchmark
SELECT count(*) FILTER (WHERE wait_event = 'transactionid') AS blocked,
       count(*) AS total,
       (SELECT setting::int FROM pg_settings WHERE name='max_connections') AS max
  FROM pg_stat_activity;
```
```
 blocked | total | max
---------+-------+-----
   ★ 412 |   441 | 500
   ★ 83% of the connection pool is queued on ONE ROW.
     Every other query in the system is now competing for the
     remaining 88 slots. (Topic 65.)
```

---

## The design decision framework

```
★★★ A HOT ROW IS A CONTENTION PROBLEM. NOTHING THAT ADDS
    CAPACITY WILL FIX IT. ★★★

 ① RECOGNISE IT
    ★ throughput FLAT while latency grows LINEARLY with clients
    ★ LOW CPU while requests time out
    ★ wait_event = 'transactionid' in pg_stat_activity
    ★ stddev ≫ mean in pg_stat_statements
    ⇒ ★ AND CHECK FOR THE SECOND HOT SPOT: 'BufferContent' means
      a shared index leaf page, not a shared row.

 ② DO THE FREE THINGS FIRST
    ★ ⒜ MOVE THE HOT UPDATE TO THE LAST STATEMENT before COMMIT
        ⇒ measured 5.3×, from reordering two lines
    ★ ⒝ REMOVE EVERY NETWORK CALL from the transaction (Topic 52)
    ★ ⒞ SHORTEN THE TRANSACTION generally
    ⇒ ★ contention = lock_hold_time × arrival_rate. You can only
      change one of those for free.

 ③ MEASURE THE SKEW — ONE KEY OR ALL KEYS?
    SELECT key, count(*), 100.0*count(*)/sum(count(*)) OVER ()
      FROM writes WHERE … GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
    ⇒ ★ ONE key ⇒ FIX IT SURGICALLY with a hot-key registry.
      Do NOT re-architect 41 million rows to serve 41.
    ⇒ ALL keys ⇒ change the design globally.

 ④ CHOOSE THE STRATEGY
    can you compute it on read?      ⇒ ★ DO THAT (Topic 54 gate 2)
    seconds of staleness acceptable? ⇒ ★ INSERT-ONLY + ROLLUP
                                        (zero contention, 184k/s)
    exact, immediate, monotonic?     ⇒ ★ SHARDED COUNTER
    exact, immediate, decrements?    ⇒ ★ sharded above a threshold,
                                        single locked row below
                                        (case study 01)
    ✗ ★ NEVER optimistic concurrency (p≈1 ⇒ 97.8% waste)

 ⑤ SHARDED COUNTER — GET THE DETAILS RIGHT
    ★ shard = pg_backend_pid() % N,  N ≥ connection-pool size
      ⇒ two backends NEVER share a row ⇒ contention → ZERO
    ★ fillfactor = 50
    ★ autovacuum_vacuum_scale_factor = 0.0, threshold = 1000,
      cost_delay = 0
    ★ start at N = 16, measure, raise if writes still queue
    ★ the read is one index range scan — it costs nothing

 ⑥ INSERT-ONLY — GET THE COMPACTION RIGHT
    ★ compact by an ID WATERMARK, never by timestamp
    ★ exclude rows newer than a few seconds (in-flight transactions
      have ids assigned before commit)
    ★ partition the delta table and DROP for retention (Topic 59)

 ⑦ AUTOMATE PROMOTION AND DEMOTION
    ★ promote when a key exceeds a measured write rate
    ★ migrate the existing value atomically, under a row lock
    ★ demote after a quiet period, or the registry grows forever

 ⑧ REMEMBER THE BLAST RADIUS
    ★ a hot row consumes the CONNECTION POOL. 412 of 500
      connections blocked on one row degrades EVERY endpoint.
    ⇒ ★ a statement_timeout and a concurrency limiter on the hot
      path protect the rest of the system (Topics 57, 65).
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create a single-row counter and benchmark it at 10, 50, 100, 200 and 400 clients. Plot throughput and latency. Then, during the benchmark, capture `pg_stat_activity` wait events. Explain what the two curves and the wait event together prove.

### Exercise 2 — medium (apply it)
Implement all four strategies for the same counter — single row, sharded (random *and* pid-based), insert-only with compaction, and compute-on-read — and benchmark each at 200 clients. Report tps, p99, read cost, staleness and storage for all five variants.

Then demonstrate the decrement failure with a sharded counter and implement the threshold fallback.

### Exercise 3 — hard (production simulation)
A social platform's `follower_count` trigger works fine for 41 million users. One creator goes viral: 41,000 follows/sec, `POST /follow` p99 41,200 ms, 34% errors — and p99 on *every other write* rises to 8,400 ms. Database CPU is 22%.

(a) Explain what 22% CPU with 34% timeouts tells you, and what it rules out.
(b) Why did unrelated endpoints degrade? Name the specific resource.
(c) Measure the skew. One user is 88.4% of writes and only 41 of 41M users are ever hot. What does that imply about the shape of the fix?
(d) Explain, for each of sharding the database, adding replicas, a bigger server, and optimistic concurrency, exactly why it would not help.
(e) Design the hot-key registry, the sharded table, and the trigger that routes between the two paths.
(f) Write the promotion function. Why must it lock the user row, and why must it zero the old column?
(g) Write the demotion function and explain why it's necessary.
(h) `wait_event` also showed 88 backends on `BufferContent`. What is the second hot spot, and how does hash partitioning fix it?
(i) Two changes in the transaction body were free and accounted for a large share of the improvement. Name them and explain the mechanism.
(j) Give the `fillfactor` and autovacuum settings for the shard table, with justification for each number.
(k) Write the alert that would detect the next viral user before it becomes an incident.

---

## Mental model checkpoint

1. Derive the throughput ceiling of a single hot row. What determines it?
2. What is the distinctive signature of lock contention in throughput/latency curves and in `pg_stat_activity`?
3. Why do more application servers make a hot row *worse*?
4. Why does sharding the database not fix a hot key?
5. Name the free fix that often gives the largest single improvement, and explain the mechanism.
6. Name the three second-order costs of a hot row.
7. Why is `pg_backend_pid() % N` better than `random() % N`? What must be true of N?
8. When is insert-only better than sharded, and what does it cost?
9. Why do sharded counters break for decrements with a floor, and what is the production fix?
10. Why must insert-only compaction use an id watermark rather than a timestamp, and what extra guard does it need?
11. What does `wait_event = 'BufferContent'` indicate, and how does it differ from `transactionid`?
12. Why must a hot-key registry support demotion?

---

## Quick reference card

**Diagnose**
```sql
SELECT wait_event_type, wait_event, count(*) FROM pg_stat_activity
 WHERE state='active' GROUP BY 1,2 ORDER BY 3 DESC;
-- ★ 'transactionid' = row lock contention
-- ★ 'BufferContent' = shared index leaf page — a SECOND hot spot
```
**The signature:** throughput **flat**, latency **linear**, CPU **low**, `stddev ≫ mean`.

**Ceiling:** `≈ 1 / lock_hold_time` — ~**800 writes/sec** practical on one row.

**Free fixes, first**
```sql
-- ★ hot row LAST in the transaction        ⇒ measured 5.3×
-- ★ no network calls inside the transaction (Topic 52)
-- ★ shorten the transaction
```

**The four strategies**

| | tps @200 | Staleness | Notes |
|---|---|---|---|
| single row | ★ 2,884 | none | the baseline |
| sharded, `random()` | 38,204 | none | residual collisions |
| ★ sharded, `pid % N` | ★ **51,884** | none | ★ N ≥ pool size ⇒ **zero** contention |
| ★ insert-only + rollup | ★ **184,204** | seconds | ★ inserts never conflict |
| compute on read | n/a | none | ★ try this first |
| ✗ optimistic | ★ 412 | none | ★ 97.8% waste |

```sql
-- sharded write
INSERT INTO counter_shards (counter_id, shard, n)
VALUES ($1, pg_backend_pid() % 64, 1)
ON CONFLICT (counter_id, shard) DO UPDATE SET n = counter_shards.n + 1;

-- ★ mandatory table settings
WITH (fillfactor = 50)
ALTER TABLE … SET (autovacuum_vacuum_scale_factor = 0.0,
                   autovacuum_vacuum_threshold = 1000,
                   autovacuum_vacuum_cost_delay = 0);
```

**★ Decrements with a floor:** sharded above a threshold, **single locked row below it**.
**★ Insert-only compaction:** an **id watermark**, plus exclude rows newer than a few seconds.
**★ One skewed key ⇒ a hot-key registry with promotion *and* demotion.** Don't re-architect 41M rows to serve 41.

**★ A hot row consumes the connection pool and degrades every endpoint.**

---

## When would I use this at work?

1. **Any incident where the database is idle but requests are timing out.** Low CPU plus high error rate is contention, and `pg_stat_activity`'s wait events name it in one query. This diagnosis takes thirty seconds if you know the shape and an afternoon if you don't.

2. **Before adding a maintained counter (Topic 55).** Measure writes/sec on the *hottest* key, not the average. Under ~100/sec a plain counter is fine; above ~800 it's a bottleneck waiting for a viral moment.

3. **Any "top N", "like count", "view count" or "follower count" feature.** These are power-law by nature — the design must work for the median entity *and* the outlier, and the answer is usually a registry with two paths rather than one design for both.

4. **When someone proposes sharding to fix a throughput problem.** Check for a hot key first. Sharding distributes keys, not load within a key — and if 40% of writes hit one key, sixteen shards give you the throughput of one.

---

## Connected topics

**Understand before this:** 45 (locks — why the row lock is held until commit), 46 (MVCC — the bloat on one page), 49 (optimistic vs pessimistic — why `p≈1` is fatal), 55 (maintained aggregates — where hot rows come from), 60 (sharding — what it cannot fix).

**This unlocks:**
- **65** — connection pooling: why a hot row exhausts the pool
- **67** — performance investigation: contention vs saturation
- **Case study 01** — flash-sale inventory: the decrement problem in full
- **Case study 13** — gaming leaderboards: hot counters at scale
- **Case study 14** — ad serving: impression counters at extreme rates

---

> ### ★ PHASE 6 COMPLETE — Denormalisation & Scale (53–61)
>
> **The arc:** **53** what a copy costs · **54** five gates, and most proposals die at gate 2 · **55** four patterns, four failure modes · **56** async copies the engine maintains · **57** copies outside, where nothing helps · **58** a full copy, milliseconds behind · **59** one table, many pieces, one server · **60** many servers, and everything you give up · **61** the one problem none of them solve.
>
> **The thread running through all nine:** every technique in this phase trades correctness, freshness or simplicity for speed — and every one of them has a cheaper alternative that teams skip. An index beats denormalisation. A CDN beats Redis. Partitioning beats sharding. Measuring beats assuming. The production examples in this phase are almost all the same story: **someone reached for the expensive tool without measuring, and the measurement would have found something cheaper and better.**
>
> **Next: Phase 7 — Reliability & Operations (62–69).** Everything so far assumed the database stays up. Now: replication internals, failover, backups, connection pooling, performance investigation, CAP, and security.
