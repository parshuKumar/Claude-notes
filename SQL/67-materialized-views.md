# Topic 67 — Materialized Views
### SQL Mastery Curriculum — Phase 9: Advanced SQL Patterns

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a busy pizza shop. Every evening a manager asks you: "How many pizzas of each type did we sell today, broken down by hour?"

You have two ways to answer.

**The regular view way**: Every time the manager asks, you pull out the giant stack of order tickets — thousands of them — and you re-count everything from scratch, right there on the spot. The answer is always perfectly up to date (it counts the pizza that was sold ten seconds ago), but it takes you twenty minutes of frantic counting every single time. If ten managers ask, you count ten times.

**The materialized view way**: Once, at a quiet moment, you count everything carefully and write the totals on a whiteboard on the wall. Now when a manager asks, you just glance at the whiteboard and read the number. Instant. Ten managers? Ten glances, all instant. The catch: the whiteboard shows the totals *as of the last time you counted*. If you counted at 2pm and it's now 4pm, the whiteboard is two hours stale. Whenever you want it fresh again, you do the twenty-minute count once and overwrite the whiteboard.

A **materialized view** is that whiteboard. It is a query whose *result* has been computed once and physically stored on disk, like a real table. Reading it is as cheap as reading a table. The cost of the expensive query is paid once, at "refresh" time, not every time someone reads.

The whole art of materialized views is one decision: **how often do you re-count and overwrite the whiteboard?** Too rarely and the numbers are wrong. Too often and you're back to counting all the time. That tension — staleness versus refresh cost — is the entire topic.

A regular view (Topic 65, if you covered it) is the "re-count every time" manager. A materialized view is the whiteboard. Knowing which one to reach for, and how to keep the whiteboard fresh without making everyone wait while you erase it, is what this topic teaches.

---

## 2. Connection to SQL Internals

A materialized view is not a clever query trick — it is a **real heap relation** on disk, almost indistinguishable from an ordinary table at the storage layer. Understanding what Postgres physically does when you create, read, and refresh one is what separates using it well from cargo-culting it.

**It has heap pages, a `pg_class` entry, and a relfilenode.** When you `CREATE MATERIALIZED VIEW`, Postgres runs the defining query once and writes the resulting rows into 8KB heap pages, exactly the same page format a table uses. `pg_class.relkind` is `'m'` (versus `'r'` for an ordinary table, `'v'` for a view). Because it is a genuine heap, it participates in the **buffer pool** — reading a matview pulls its pages into `shared_buffers` and they stay hot like any table's pages.

**It obeys MVCC, so it is visible to concurrent transactions and to WAL.** Rows in a matview carry `xmin`/`xmax` visibility headers. Writing it is WAL-logged (unless the matview is `UNLOGGED`), so it survives crash recovery and is replicated to standbys. This is a crucial difference from a temp table or a session cache — a matview is durable, shared, and replicated infrastructure.

**A plain `REFRESH` truncates and rewrites.** `REFRESH MATERIALIZED VIEW foo` re-executes the defining query and completely replaces the contents. Internally this is close to a `TRUNCATE` + `INSERT ... SELECT` into a fresh relfilenode, then an atomic catalog swap. Because it takes an **`ACCESS EXCLUSIVE` lock**, every reader is blocked for the whole refresh — the same lock class that `ALTER TABLE` and `DROP` take. On a large matview this can be a multi-minute stall for all readers.

**`REFRESH ... CONCURRENTLY` uses a diff-and-merge via a temp table.** Instead of swapping the whole relation, Postgres builds the new result into a transient table, then computes the *difference* against the current contents and applies only `INSERT`/`UPDATE`/`DELETE` of the changed rows using a `FULL OUTER JOIN` on a **required unique index**. It takes only an `EXCLUSIVE` lock (not `ACCESS EXCLUSIVE`), so `SELECT` readers are never blocked. The trade-off is real work: it still executes the full query, plus the diffing join, so it is *more* total CPU/IO than a plain refresh — it buys reader availability, not speed.

**You can index it, and the planner treats those indexes exactly like table indexes.** Because it is a heap, `CREATE INDEX ON matview (...)` builds a normal B-tree (or GIN, GiST, BRIN) pointing at the matview's heap tuples. Queries against the matview go through the same planner and cost model as any table scan — Seq Scan, Index Scan, Bitmap Heap Scan all apply. This is the payoff: an expensive aggregation becomes a cheap indexed lookup.

**The planner never rewrites a query to use a matview automatically.** Unlike some commercial databases (Oracle's query rewrite, or "automatic materialized views"), Postgres does **not** transparently substitute a matview for a query that happens to match its definition. You must reference the matview by name. This is a frequent interview trap.

---

## 3. Logical Execution Order Context

A materialized view interacts with logical execution order at two distinct moments, and conflating them is a common source of confusion.

**At refresh time**, the *defining query* runs through the full logical pipeline exactly like any SELECT:

```
FROM / JOIN        ← the base tables are read and joined
WHERE              ← rows filtered
GROUP BY           ← aggregation (the usual reason a matview exists)
HAVING             ← post-aggregate filter
SELECT             ← projection, expressions, window functions
DISTINCT
ORDER BY           ← usually pointless in a matview definition — see below
LIMIT
```

The entire result of this pipeline is frozen onto disk. Whatever the query computed at that instant is what the matview now contains until the next refresh.

**At query time**, when *you* SELECT from the matview, it behaves as a base relation — it enters your query at the `FROM` stage:

```
FROM my_matview          ← the matview is just a table here
JOIN other_table ON ...  ← you can join a matview to anything
WHERE ...                ← filters apply to the stored rows
GROUP BY / HAVING ...    ← you can even re-aggregate a matview
ORDER BY / LIMIT
```

Two consequences fall out of this split:

1. **`ORDER BY` inside a matview definition is almost always wasted.** The rows are stored in whatever physical order the refresh produced, but the planner is free to return them in any order when you query — so a downstream query must supply its own `ORDER BY`. Worse, a plain (non-concurrent) refresh may re-run the sort every time for no benefit. Put ordering in the *reading* query, not the definition. (The one exception: a definition `ORDER BY` can improve physical clustering, which a BRIN index can exploit — but that is a deliberate storage decision, not a correctness one.)

2. **The matview's freshness is decoupled from your query's execution.** Your SELECT is evaluated against whatever the last refresh stored. Logical execution order tells you nothing about *when* those rows were computed — that is governed entirely by your refresh schedule, which is external to SQL's evaluation model. This is the mental shift: with a regular view the "when" is *now*; with a matview the "when" is "last refresh."

---

## 4. What Is a Materialized View?

A materialized view is a database object that **stores the result of a query physically on disk** and lets you read it like a table. The stored result is a snapshot as of the last refresh; it does not update automatically when the underlying tables change. You explicitly re-run the query with `REFRESH MATERIALIZED VIEW` to bring it current.

```sql
CREATE MATERIALIZED VIEW daily_product_sales AS
SELECT
    DATE_TRUNC('day', o.created_at) AS sale_day,
    oi.product_id,
    COUNT(DISTINCT o.id)            AS order_count,
    SUM(oi.quantity)               AS units_sold,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
INNER JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'completed'
GROUP BY DATE_TRUNC('day', o.created_at), oi.product_id
WITH DATA;
```

Annotated syntax breakdown:

```
CREATE MATERIALIZED VIEW daily_product_sales
│      │                 └── name of the stored relation (becomes a pg_class entry, relkind='m')
│      └── keyword pair: this is a matview, not a plain VIEW (which stores no data)
└── DDL — takes ACCESS EXCLUSIVE lock on the new object; runs in a transaction

AS
│
└── separates the object declaration from its defining query

SELECT ... FROM orders o
│              └── base tables; their current contents are read ONCE, now
│                  (later changes to orders do NOT flow through automatically)
INNER JOIN order_items oi ON oi.order_id = o.id
│          └── any join is allowed; the join cost is paid at refresh, not at read
WHERE o.status = 'completed'
│     └── filters baked into the stored snapshot
GROUP BY DATE_TRUNC('day', o.created_at), oi.product_id
│        └── aggregation — the single most common reason a matview exists:
│            it pre-computes an expensive GROUP BY so reads are cheap
└── the full result set of this query is written to heap pages

WITH DATA
│    └── populate immediately by running the query now (this is the DEFAULT)
└── alternative: WITH NO DATA creates an empty, UNSCANNABLE shell —
    querying it errors until the first REFRESH
```

The two population modes:

```
WITH DATA      → query runs now, matview is populated and queryable immediately
WITH NO DATA   → matview created empty; SELECT from it raises:
                 ERROR: materialized view "..." has not been populated
                 HINT:  Use the REFRESH MATERIALIZED VIEW command.
```

`WITH NO DATA` is useful when you want to create the object and its indexes first (a fast metadata-only operation) and defer the expensive population to a scheduled job.

To refresh:

```
REFRESH MATERIALIZED VIEW daily_product_sales;
│                                              └── plain refresh: re-runs the query,
│                                                  replaces ALL contents,
│                                                  takes ACCESS EXCLUSIVE (blocks readers)
│
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_product_sales;
│                         └── diff-and-merge refresh: readers keep working,
│                             REQUIRES a UNIQUE index on the matview
```

---

## 5. Why Materialized View Mastery Matters in Production

1. **They turn a 30-second dashboard into a 5-millisecond one.** The canonical use case is an analytics or reporting query that joins and aggregates millions of rows. Run live, it hammers the primary and takes seconds. Materialized, it is a single indexed table scan. The entire "our dashboard is slow" class of tickets is often solved by one matview plus a refresh job.

2. **A plain `REFRESH` can take down your read traffic.** Because `REFRESH MATERIALIZED VIEW` (without `CONCURRENTLY`) holds an `ACCESS EXCLUSIVE` lock for its full duration, every `SELECT` against the matview blocks until it finishes. Engineers who don't know this schedule a naive refresh, and at 2am the refresh of a 40M-row matview locks the dashboard for four minutes. Knowing to use `CONCURRENTLY` — and that it requires a unique index — is the difference between a transparent refresh and an outage.

3. **The refresh strategy is a real distributed-systems decision.** Choosing scheduled vs event-driven refresh, and how stale is acceptable, is an architecture question with correctness implications. Bill a customer off a stale matview and you've overcharged them. Show a stale "in stock" count and you oversell. Mastery means reasoning explicitly about the staleness window and matching it to the business tolerance.

4. **`CONCURRENTLY` is not free and can silently degrade.** It runs the full query *plus* a diffing join, so it costs more total resources than a plain refresh. On a matview whose rows churn heavily, the diff step can dominate. Teams that switch to `CONCURRENTLY` for availability sometimes find refresh duration doubled — knowing the trade-off lets you decide consciously.

5. **Choosing the wrong tool wastes weeks.** Matview vs regular view vs a hand-maintained summary table vs a rollup updated by triggers — each fits a different freshness/write-pattern profile. Reaching for a matview when you needed real-time (should have been a summary table with incremental updates) or hand-rolling triggers when a nightly matview would do are both expensive mistakes that a clear mental model prevents.

---

## 6. Deep Technical Content

### 6.1 Creating a Materialized View: `WITH DATA` vs `WITH NO DATA`

The default is `WITH DATA` — the defining query executes immediately and the result is stored. `WITH NO DATA` creates the catalog entry and allocates the relation but leaves it empty and **unscannable**:

```sql
CREATE MATERIALIZED VIEW mv_report AS
SELECT ... FROM huge_table ...
WITH NO DATA;

SELECT * FROM mv_report;
-- ERROR:  materialized view "mv_report" has not been populated
-- HINT:   Use the REFRESH MATERIALIZED VIEW command.
```

The state is tracked in `pg_class.relispopulated`. A matview created `WITH NO DATA`, or that has never been refreshed, is `relispopulated = false` and any scan raises the error above. The first `REFRESH` flips it to `true`.

Why defer? You can create the object and build its indexes as a fast, near-instant metadata operation during a deploy, then run the heavy population from a background job — keeping the deploy itself quick and non-blocking.

### 6.2 The Plain Refresh: What Actually Happens

```sql
REFRESH MATERIALIZED VIEW mv_report;
```

Mechanically, Postgres:

1. Takes an **`ACCESS EXCLUSIVE`** lock on `mv_report`.
2. Creates a new, empty relfilenode.
3. Re-executes the defining query and writes every result row into the new relfilenode (heap-inserts, WAL-logged unless `UNLOGGED`).
4. Atomically swaps the new relfilenode in for the old one in the catalog.
5. Drops the old relfilenode and releases the lock.

Consequences:

- **All readers block** from step 1 to step 5. A `SELECT` issued during the refresh waits on the lock; it does not see old data and it does not error — it simply stalls until the refresh commits, then sees the new data.
- **It is all-or-nothing.** Because it runs in a transaction and swaps atomically, readers either see the entirely-old snapshot (before commit) or the entirely-new one (after). There is no partial state.
- **It re-does all the work.** Even if only one base-table row changed, the full query re-runs over all base rows. Postgres has no incremental refresh natively (see 6.9).

### 6.3 The Concurrent Refresh: Diff-and-Merge

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_report;
```

This exists to solve the "readers are blocked" problem. Instead of swapping the whole relation, Postgres:

1. Takes only an **`EXCLUSIVE`** lock (which blocks writes to the matview and other refreshes, but **not** `SELECT` readers).
2. Runs the defining query into a **transient temp table**.
3. Performs a `FULL OUTER JOIN` between the temp table and the current matview contents **on the unique index columns**, computing which rows are new (INSERT), gone (DELETE), or changed (UPDATE).
4. Applies only those deltas to the live matview via ordinary DML.
5. Releases the lock.

Because only `INSERT`/`UPDATE`/`DELETE` of *changed* rows touch the live relation, and those are MVCC operations, concurrent `SELECT`s keep reading the old rows until each delta commits — no reader ever blocks.

**The hard requirement:** `CONCURRENTLY` needs a **UNIQUE index** covering rows unambiguously, so the diff join has a key to match old and new rows on:

```sql
-- Without a unique index:
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_report;
-- ERROR:  cannot refresh materialized view "mv_report" concurrently
-- HINT:   Create a unique index with no WHERE clause on one or more
--         columns of the materialized view.

-- The fix — a unique index on the matview's logical key:
CREATE UNIQUE INDEX mv_report_pk
    ON mv_report (sale_day, product_id);
```

The unique index must have **no `WHERE` clause** (it cannot be partial) and its columns must be unique across all rows of the matview. Choosing this key is exactly like choosing a primary key: it must be the set of columns that identifies one output row (typically the `GROUP BY` columns).

**Cost trade-off:** `CONCURRENTLY` does *strictly more* work than a plain refresh — it runs the same query, then additionally builds the temp table and performs the diffing `FULL OUTER JOIN` and per-row DML. It optimizes for **reader availability, never for speed**. On a matview where most rows change every refresh, the diff overhead is close to pure waste and a plain refresh (accepting the lock) may be faster overall.

### 6.4 Indexing a Materialized View

Because a matview is a real heap, you index it exactly like a table, and those indexes are the whole reason reads are fast:

```sql
-- Unique index: required for CONCURRENTLY, doubles as the logical PK
CREATE UNIQUE INDEX mv_daily_sales_pk
    ON daily_product_sales (sale_day, product_id);

-- Secondary index for a common filter/sort on reads
CREATE INDEX mv_daily_sales_product
    ON daily_product_sales (product_id, sale_day DESC);

-- Partial index if reads target a hot subset
CREATE INDEX mv_daily_sales_recent
    ON daily_product_sales (sale_day)
    WHERE sale_day >= DATE '2026-01-01';

-- GIN index if the matview stores a tsvector or jsonb column
CREATE INDEX mv_docs_search ON mv_docs USING GIN (search_vector);
```

Key facts:

- **Indexes survive a refresh.** A plain refresh swaps the relfilenode but Postgres rebuilds the indexes as part of the operation, so index definitions persist. A concurrent refresh keeps them live and maintains them via the delta DML.
- **A refresh must rebuild every index**, which adds to refresh cost. Every extra index you add to a matview slows the refresh. Index the matview for its *read* patterns, but be aware each index is a refresh tax.
- **You need the unique index before the first `CONCURRENTLY`.** Create it right after `CREATE MATERIALIZED VIEW ... WITH NO DATA`, then do the first populating refresh concurrently or plainly.

### 6.5 The Refresh-Strategy Problem: Staleness vs Cost

This is the intellectual core of the topic. A matview is always stale to some degree; the design question is *how stale is acceptable, and what does keeping it fresher cost?*

**Axis 1 — Scheduled (time-driven) refresh.** A cron job / `pg_cron` / application scheduler refreshes on a fixed cadence: every 5 minutes, hourly, nightly.

- *Pros:* dead simple; predictable load; easy to reason about staleness window (max staleness = the interval).
- *Cons:* refreshes even when nothing changed (wasted work); data can be up to one full interval stale; a fixed cadence can't distinguish quiet from busy periods.

**Axis 2 — Event-driven refresh.** A trigger, a change-data-capture consumer, or application logic refreshes when the underlying data actually changes (e.g., after a batch import completes).

- *Pros:* matview is fresh right after the data that feeds it changes; no wasted refreshes during quiet periods.
- *Cons:* far more complex; a naive "refresh after every insert" trigger is catastrophic — a full refresh per row turns an O(1) insert into an O(N) query. Almost always you must **debounce**: coalesce many changes into one refresh (e.g., "at most one refresh per minute" or "refresh once the import job signals done").

**The staleness/cost frontier:**

| Refresh cadence | Max staleness | Refresh load | Fits |
|---|---|---|---|
| On every write (trigger) | ~0 | Catastrophic | Almost never — use a summary table instead |
| Every 1 min (pg_cron) | 1 min | High | Near-real-time dashboards on moderate data |
| Every 15 min | 15 min | Moderate | Ops dashboards, internal analytics |
| Hourly | 1 hr | Low | Business reporting |
| Nightly | up to 24 hr | Very low | Daily rollups, warehouse-style reports |
| Event-driven (post-batch) | seconds after batch | Low, bursty | ETL pipelines where data lands in batches |

The correct answer is always: **find the business tolerance for staleness, then pick the cheapest cadence that stays inside it.** A financial "revenue yesterday" report tolerates nightly. A "live orders in the last hour" widget needs minutes. Never refresh more often than the data's freshness is actually consumed.

**A common hybrid:** schedule a frequent `CONCURRENTLY` refresh (so reads never block) at the fastest cadence the refresh duration allows. If the refresh itself takes 90 seconds, you cannot meaningfully schedule it every 30 seconds — refreshes would pile up. The floor on your cadence is the refresh duration itself (plus headroom).

### 6.6 Matview vs Regular View vs Summary Table

Three tools solve overlapping problems with very different freshness/cost profiles.

**Regular view** (`CREATE VIEW`): stores *no data* — it is a saved query, macro-expanded into every query that references it. Always perfectly fresh (it runs live), zero storage, zero refresh, but **every read pays the full query cost**. Right when the underlying query is cheap, or when you need absolute freshness and can afford the per-read cost.

**Materialized view**: stores the *result*. Reads are cheap; freshness is "as of last refresh"; the full query re-runs on each (non-incremental) refresh. Right when the query is expensive, reads are frequent, and bounded staleness is acceptable.

**Summary / rollup table** (a real table you maintain yourself): stores an aggregate that you update **incrementally** — e.g., a trigger or application code that does `UPDATE daily_totals SET revenue = revenue + NEW.amount` on each order. Reads are cheap *and* it can be near-real-time, but **you** own the correctness of the incremental maintenance logic (the hardest part — every insert, update, delete, and out-of-order event must be handled).

| | Regular View | Materialized View | Summary Table |
|---|---|---|---|
| Stores data? | No | Yes (full result) | Yes (aggregate) |
| Freshness | Always live | As of last refresh | Near-real-time (if maintained well) |
| Read cost | Full query every time | Cheap (indexed scan) | Cheap (indexed scan) |
| Write/maintenance cost | None | Full re-query per refresh | Incremental per base-row change |
| Maintenance complexity | None | Low (one REFRESH command) | High (you write correct delta logic) |
| Handles arbitrary query? | Yes | Yes | Only what your update logic covers |
| Best for | Cheap queries, need freshness | Expensive queries, tolerate staleness | Need both cheap reads AND freshness |

The decision tree: *Is the query cheap?* → regular view. *Is it expensive but can you tolerate staleness?* → materialized view. *Is it expensive AND you need near-real-time?* → summary table with incremental maintenance (accept the complexity), or a stream/rollup system outside Postgres.

### 6.7 When a Matview Is the Right Answer (and When It Isn't)

**Right when:**
- The query is expensive (big joins, heavy aggregation, window functions over large sets).
- It is read far more often than the underlying data changes (read-heavy).
- Bounded staleness is acceptable — minutes or hours, not zero.
- The result set is small enough to store and refresh in reasonable time.

**Wrong when:**
- You need strict real-time correctness (billing, inventory-at-checkout, balances). Stale numbers here are *bugs*, not lag.
- The base data changes on nearly every read — the refresh can't keep up and you're paying full re-query cost constantly anyway.
- The defining query is already cheap — a regular view or direct query is simpler with no staleness.
- The result is huge and changes broadly each refresh — refresh cost dominates and you never amortize it over enough reads.
- You need row-level security or per-user filtering baked in — a matview stores one shared snapshot; per-user variation belongs in the reading query or a view.

### 6.8 Dependencies, Ownership, and Dropping

A matview depends on its base tables. This has sharp edges:

- **You cannot `DROP` or incompatibly `ALTER` a base table/column** the matview references without first dropping the matview (or using `CASCADE`). `DROP TABLE orders;` errors if a matview selects from `orders`, listing the matview as a dependent.
- **Matviews do not chain-refresh.** If matview B is built on matview A, refreshing A does **not** refresh B. You must refresh A then B, in dependency order, yourself. There is no built-in dependency-ordered refresh.
- **Permissions:** reading a matview requires `SELECT` on the matview itself; the reader does *not* need access to the base tables. This makes matviews a way to expose a pre-computed, pre-filtered slice to a role that shouldn't see raw tables.
- **Schema changes to the definition require a drop/recreate** (or `CREATE OR REPLACE` is *not* supported for matviews the way it is for views — you `DROP MATERIALIZED VIEW` and recreate, or use a migration that swaps a new one in).

### 6.9 No Native Incremental Refresh (and the Workarounds)

Postgres (through current versions) has **no built-in incremental/automatic materialized view maintenance** — every non-trivial refresh re-runs the whole query. This is the single biggest limitation. Workarounds:

- **Partition the matview conceptually by time** and only refresh recent partitions. Postgres can't partition a matview directly, but you can build **per-period matviews** (`mv_sales_2026_07`) and only refresh the current one, treating older ones as frozen.
- **Roll your own summary table** with triggers for true incremental maintenance (6.6) when the workload demands it.
- **Use an extension** — e.g., `pg_ivm` (Incremental View Maintenance) provides trigger-based incremental matviews for a subset of query shapes. Not core Postgres; evaluate carefully.
- **Refresh only what's stale** at the application layer: track a "last changed" watermark on base data and skip the refresh entirely if nothing changed since the last one.

### 6.10 `UNLOGGED` Materialized Views

A matview can be `UNLOGGED`:

```sql
CREATE UNLOGGED MATERIALIZED VIEW mv_scratch AS SELECT ...;
```

Its contents are **not WAL-logged**, so refreshes are faster and generate no replication traffic — but the matview is **truncated on crash recovery** and **does not exist on replicas**. Right for a derived cache that a scheduled job will rebuild anyway and that no replica needs to serve. Wrong if read replicas must serve it or if a post-crash empty matview would break the app before the next refresh.

---

## 7. EXPLAIN — Materialized View Reads and Refreshes in the Plan

### 7.1 Reading a matview: an indexed scan, like any table

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT sale_day, units_sold, revenue
FROM daily_product_sales
WHERE product_id = 4823
ORDER BY sale_day DESC
LIMIT 30;
```

```
Limit  (cost=0.42..8.61 rows=30 width=28)
       (actual time=0.028..0.061 rows=30 loops=1)
  ->  Index Scan Backward using mv_daily_sales_product on daily_product_sales
        (cost=0.42..123.4 rows=452 width=28)
        (actual time=0.026..0.052 rows=30 loops=1)
        Index Cond: (product_id = 4823)
  Buffers: shared hit=6
Planning Time: 0.11 ms
Execution Time: 0.084 ms
```

**Reading it:**
- The matview is scanned via its own index `mv_daily_sales_product` — the planner treats it as an ordinary indexed relation.
- `Index Scan Backward` + `Limit` satisfies `ORDER BY sale_day DESC LIMIT 30` without a Sort node — the index `(product_id, sale_day DESC)` already provides the order.
- `Buffers: shared hit=6` — six page hits, all from cache. This is the payoff: 0.08ms versus the seconds the underlying aggregation would take live.

Contrast with running the **defining query live** (what a regular view would do on every read):

```
GroupAggregate  (cost=1892340..2013400 rows=... )
                (actual time=4120..8830 rows=... )
  ->  Sort (... spills to disk, Batches: 14 ...)
        ->  Hash Join  (orders ⋈ order_items)
              ->  Seq Scan on order_items  (actual rows=41,203,551 ...)
              ->  Hash  ->  Seq Scan on orders (actual rows=8,402,110 ...)
Execution Time: 8912 ms
```

8.9 **seconds** live versus 0.08 **milliseconds** from the matview — a ~100,000× read speedup. That gap is the entire value proposition.

### 7.2 Watching a plain REFRESH (via `EXPLAIN ANALYZE` of the defining query)

You cannot `EXPLAIN` the `REFRESH` command directly, but you profile it by explaining the defining query — that is exactly the work a refresh does:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT DATE_TRUNC('day', o.created_at) AS sale_day,
       oi.product_id,
       COUNT(DISTINCT o.id) AS order_count,
       SUM(oi.quantity) AS units_sold,
       SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
INNER JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'completed'
GROUP BY DATE_TRUNC('day', o.created_at), oi.product_id;
```

```
GroupAggregate  (cost=6231044..6698120 rows=182400 width=44)
                (actual time=18240..24310 rows=176980 loops=1)
  Group Key: (date_trunc('day', o.created_at)), oi.product_id
  ->  Sort  (cost=6231044..6333260 rows=40886... width=20)
            (actual time=18240..21120 rows=39104882 loops=1)
        Sort Key: (date_trunc('day', o.created_at)), oi.product_id
        Sort Method: external merge  Disk: 1352104kB
        ->  Hash Join  (cost=402180..1889340 rows=40886400 width=20)
                       (actual time=1820..12040 rows=39104882 loops=1)
              Hash Cond: (oi.order_id = o.id)
              ->  Seq Scan on order_items oi
                    (actual time=0.02..4210 rows=41203551 loops=1)
              ->  Hash  (actual time=1790..1790 rows=8104233 loops=1)
                    Buckets: 131072  Batches: 128  Memory Usage: ...
                    ->  Seq Scan on orders o
                          Filter: (status = 'completed')
                          Rows Removed by Filter: 297877
                          (actual rows=8104233 loops=1)
Buffers: shared hit=201 read=642233, temp read=169013 written=169013
Execution Time: 24980 ms
```

**Reading it — this is why you'd never run it live per request:**
- The `Sort` spills to disk: `Sort Method: external merge  Disk: 1352104kB` — 1.3GB of temp files because the 39M intermediate rows exceed `work_mem`.
- The `Hash` spills too: `Batches: 128` (not 1) — the hash table for `orders` doesn't fit in `work_mem`, so it's partitioned into 128 disk batches.
- `temp read=169013 written=169013` blocks — heavy temp IO, the signature of an under-provisioned `work_mem` for this refresh.
- ~25 seconds. A plain `REFRESH` holds `ACCESS EXCLUSIVE` for this whole time — 25 seconds of blocked readers. This is precisely the case for `CONCURRENTLY` (or a bigger `work_mem` set locally for the refresh session).

### 7.3 What a concurrent refresh adds

A `CONCURRENTLY` refresh runs the plan above **plus** a diffing step. Conceptually the extra work is:

```
-- (internal, not user-visible) after populating a temp table `newdata`:
... FULL OUTER JOIN diff ...
  ->  Merge/Hash Join between newdata and daily_product_sales
        Join Cond: (newdata.sale_day = mv.sale_day
                    AND newdata.product_id = mv.product_id)   ← the UNIQUE index key
  ->  then INSERT new-only rows, DELETE gone-only rows,
      UPDATE rows whose measures changed
```

So a concurrent refresh's total time ≈ (defining-query time) + (temp table build) + (diff join over the unique key) + (delta DML). It is always slower end-to-end than the plain refresh of the same matview — it trades that extra time for never blocking readers.

---

## 8. Query Examples

### Example 1 — Basic: create, read, refresh

```sql
-- A simple pre-aggregated matview of completed revenue per day
CREATE MATERIALIZED VIEW mv_daily_revenue AS
SELECT
    DATE_TRUNC('day', created_at) AS day,
    COUNT(*)                      AS orders,
    SUM(total_amount)             AS revenue
FROM orders
WHERE status = 'completed'
GROUP BY DATE_TRUNC('day', created_at)
WITH DATA;                         -- populate now (the default)

-- Read it like a table — cheap indexed/seq scan, not the live aggregation
SELECT * FROM mv_daily_revenue
ORDER BY day DESC
LIMIT 7;

-- Bring it up to date (plain refresh: blocks readers while it runs)
REFRESH MATERIALIZED VIEW mv_daily_revenue;
```

### Example 2 — Intermediate: unique index + concurrent refresh

```sql
-- Create empty first so we can build the unique index before populating
CREATE MATERIALIZED VIEW mv_product_month AS
SELECT
    oi.product_id,
    DATE_TRUNC('month', o.created_at) AS month,
    SUM(oi.quantity)                  AS units,
    SUM(oi.quantity * oi.unit_price)  AS revenue
FROM orders o
INNER JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'completed'
GROUP BY oi.product_id, DATE_TRUNC('month', o.created_at)
WITH NO DATA;                        -- shell only; not yet queryable

-- The UNIQUE index is mandatory for CONCURRENTLY; here it doubles as the PK.
-- Its columns must uniquely identify one output row = the GROUP BY columns.
CREATE UNIQUE INDEX mv_product_month_pk
    ON mv_product_month (product_id, month);

-- Secondary index for the common read pattern (one product over time)
CREATE INDEX mv_product_month_by_product
    ON mv_product_month (product_id, month DESC);

-- First population. The very first refresh of a WITH NO DATA matview can be
-- plain OR concurrent; here plain is fine since nobody reads it yet.
REFRESH MATERIALIZED VIEW mv_product_month;

-- Subsequent scheduled refreshes use CONCURRENTLY so dashboards never block:
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_month;
```

### Example 3 — Production-grade: scheduled concurrent refresh with staleness tracking

**Scenario.** `orders` holds ~8M rows, `order_items` ~41M rows. A customer-facing analytics dashboard reads a per-product 90-day rollup thousands of times an hour. The live aggregation takes ~25s (see §7.2); business tolerates up to 10 minutes of staleness. Target: reads in single-digit ms, refresh every 5 minutes without ever blocking a reader.

```sql
-- The rollup, restricted to the last 90 days to bound refresh cost
CREATE MATERIALIZED VIEW mv_product_sales_90d AS
SELECT
    oi.product_id,
    p.name                              AS product_name,
    p.category_id,
    COUNT(DISTINCT o.id)                AS order_count,
    SUM(oi.quantity)                    AS units_sold,
    SUM(oi.quantity * oi.unit_price)    AS revenue,
    MAX(o.created_at)                   AS last_sold_at,
    NOW()                               AS refreshed_at   -- staleness watermark
FROM orders o
INNER JOIN order_items oi ON oi.order_id = o.id
INNER JOIN products     p  ON p.id = oi.product_id
WHERE o.status = 'completed'
  AND o.created_at >= NOW() - INTERVAL '90 days'
GROUP BY oi.product_id, p.name, p.category_id
WITH NO DATA;

-- Unique index (required for CONCURRENTLY) on the logical grain
CREATE UNIQUE INDEX mv_psales90_pk ON mv_product_sales_90d (product_id);
-- Read-path indexes
CREATE INDEX mv_psales90_cat     ON mv_product_sales_90d (category_id, revenue DESC);
CREATE INDEX mv_psales90_revenue ON mv_product_sales_90d (revenue DESC);

-- First populate
REFRESH MATERIALIZED VIEW mv_product_sales_90d;

-- Schedule a non-blocking refresh every 5 minutes with pg_cron.
-- 5-minute cadence > the ~30-90s CONCURRENTLY refresh duration, so no pile-up.
SELECT cron.schedule(
    'refresh_psales90',
    '*/5 * * * *',
    $$ REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales_90d $$
);
```

Read path — and how the app surfaces staleness to the user:

```sql
SELECT product_id, product_name, units_sold, revenue,
       refreshed_at,
       NOW() - refreshed_at AS data_age          -- show "updated 3 min ago"
FROM mv_product_sales_90d
WHERE category_id = 17
ORDER BY revenue DESC
LIMIT 25;
```

Expected read plan and performance:

```
Limit  (cost=0.42..41.0 rows=25 width=72) (actual time=0.03..0.14 rows=25 loops=1)
  ->  Index Scan using mv_psales90_cat on mv_product_sales_90d
        Index Cond: (category_id = 17)
        (actual rows=25 loops=1)
  Buffers: shared hit=9
Execution Time: 0.17 ms
```

~0.17ms per read versus ~25s live, refreshed every 5 minutes without blocking a single reader, with `refreshed_at` letting the UI honestly show data age. That is the production shape of a materialized view.

---

## 9. Wrong → Right Patterns

### Wrong 1: Plain REFRESH on a hot matview — locks out all readers

```sql
-- WRONG: scheduled every 5 minutes on a 40M-row matview the dashboard reads live
REFRESH MATERIALIZED VIEW mv_product_sales_90d;
```

**What actually happens:** the refresh takes an `ACCESS EXCLUSIVE` lock for its full ~30–90s runtime. Every dashboard `SELECT` issued during that window blocks on the lock and hangs. Under load, connections pile up waiting, the connection pool exhausts, and requests that touch *other* tables start failing too because there are no free connections. A "5-minute refresh" becomes a recurring 60-second partial outage.

**Why at the execution level:** `ACCESS EXCLUSIVE` conflicts with `ACCESS SHARE` — the lock every `SELECT` takes. There is no MVCC escape: the reader cannot see the old snapshot because the plain refresh swaps the whole relfilenode atomically and holds the strongest lock until commit.

```sql
-- RIGHT: concurrent refresh — readers keep the old rows until each delta commits
-- (requires a UNIQUE index on the matview, created once)
CREATE UNIQUE INDEX IF NOT EXISTS mv_psales90_pk
    ON mv_product_sales_90d (product_id);

REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales_90d;
-- Readers never block; the cost is a slower refresh (query + diff), which is fine
-- because it runs in the background.
```

### Wrong 2: `CONCURRENTLY` without the unique index

```sql
-- WRONG: matview has no unique index
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;
-- ERROR:  cannot refresh materialized view "mv_daily_revenue" concurrently
-- HINT:   Create a unique index with no WHERE clause on one or more
--         columns of the materialized view.
```

**Why:** the concurrent path diffs new vs old rows with a `FULL OUTER JOIN` and needs a key to match them. No unique key → no way to identify "the same row, changed" → the command refuses rather than guess. A **partial** unique index (`... WHERE ...`) also fails — the key must cover every row.

```sql
-- RIGHT: unique index on the true grain of the matview (its GROUP BY key)
CREATE UNIQUE INDEX mv_daily_revenue_pk ON mv_daily_revenue (day);

REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;  -- now works
```

If no single column is unique, index the full set of grouping columns: `CREATE UNIQUE INDEX ... (col_a, col_b, col_c)`.

### Wrong 3: Expecting the matview to auto-update

```sql
-- Developer inserts an order, then reads the matview and expects to see it:
INSERT INTO orders (customer_id, total_amount, status, created_at)
VALUES (42, 99.00, 'completed', NOW());

SELECT revenue FROM mv_daily_revenue WHERE day = CURRENT_DATE;
-- WRONG expectation: revenue reflects the new order.
-- REALITY: it shows the value as of the last REFRESH. The new order is invisible
-- until a refresh runs. No error — just silently stale data.
```

**Why:** a matview is a frozen snapshot; base-table DML does not propagate to it. Postgres has no automatic/incremental maintenance. This is the single most common misconception.

```sql
-- RIGHT: either refresh after the write completes (batch/ETL pattern)...
INSERT INTO orders (...) VALUES (...);
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;

-- ...or, if you need real-time, DON'T use a matview — use a summary table
-- maintained incrementally, or query the base tables live for that number.
```

### Wrong 4: Refresh-per-row trigger

```sql
-- WRONG: "keep it fresh" trigger that refreshes on every insert
CREATE OR REPLACE FUNCTION refresh_sales() RETURNS trigger AS $$
BEGIN
    REFRESH MATERIALIZED VIEW mv_product_sales_90d;  -- full 25s re-query!
    RETURN NEW;
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER t_refresh AFTER INSERT ON orders
FOR EACH ROW EXECUTE FUNCTION refresh_sales();
-- CATASTROPHE: every single order insert now triggers a full 25-second
-- aggregation over 41M rows. Inserts go from <1ms to >25s. Under any real
-- write rate the system melts. (Also: you can't even run a plain REFRESH
-- inside the transaction that holds a lock on orders — deadlock/lock errors.)
```

**Why:** you turned an O(1) write into an O(N) full re-query, per row. Event-driven refresh must be **debounced** — coalesce many changes into at most one refresh per interval.

```sql
-- RIGHT: decouple. Writes stay fast; a debounced job refreshes at most once/min.
-- e.g. with pg_cron (time-debounced), guarded to skip if base data is unchanged:
SELECT cron.schedule('refresh_sales_1min', '* * * * *', $$
    REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales_90d
$$);
-- Or trigger sets a "dirty" flag; a single worker refreshes when dirty, then clears it.
```

### Wrong 5: `ORDER BY`/`LIMIT` semantics baked into the definition

```sql
-- WRONG: "top 100 products" frozen into the matview
CREATE MATERIALIZED VIEW mv_top_products AS
SELECT product_id, SUM(revenue) AS rev
FROM order_items GROUP BY product_id
ORDER BY rev DESC LIMIT 100;
-- Problem 1: the ORDER BY does not guarantee your reads come back sorted —
--   you must ORDER BY again when you SELECT.
-- Problem 2: LIMIT 100 permanently discards rows 101+. If tomorrow's #40 was
--   today's #150, it isn't in the matview and can never surface until refresh —
--   and even then the cutoff is arbitrary and hides data.
```

**Why:** the matview stores exactly the query's result — including the truncation. Storing a `LIMIT`ed, pre-sorted slice bakes a presentation decision into stored data and loses information.

```sql
-- RIGHT: store the full aggregate; sort and limit at READ time
CREATE MATERIALIZED VIEW mv_product_revenue AS
SELECT product_id, SUM(revenue) AS rev
FROM order_items GROUP BY product_id;

CREATE UNIQUE INDEX ON mv_product_revenue (product_id);
CREATE INDEX ON mv_product_revenue (rev DESC);   -- supports the top-N read

-- The reading query decides ordering and how many rows:
SELECT * FROM mv_product_revenue ORDER BY rev DESC LIMIT 100;
```

---

## 10. Performance Profile

### 10.1 Two totally separate cost regimes

A matview has two independent performance stories, and mixing them up is the classic analysis error:

- **Read cost** — cheap and roughly constant: an indexed or sequential scan of the *stored* result. This is what your application latency depends on.
- **Refresh cost** — expensive and proportional to the *defining query* plus (for `CONCURRENTLY`) the diff. This is background load, not user-facing latency, but it consumes CPU, IO, and WAL, and holds locks.

You optimize these separately. Reads: index the matview for its query patterns. Refreshes: bound the query (time windows, pre-filters), raise `work_mem` for the refresh session, and pick the cheapest cadence that meets the staleness SLA.

### 10.2 Read scaling with matview size

| Matview rows | Indexed point/range read | Full scan (no index) |
|---|---|---|
| 10K | < 0.1 ms | ~2 ms |
| 100K | < 0.2 ms | ~15 ms |
| 1M | < 0.5 ms | ~120 ms |
| 10M | < 1 ms (indexed) | ~1.2 s |
| 100M | 1–3 ms (indexed) | ~12 s |

The lesson: **reads stay fast as long as you index for them.** An unindexed 100M-row matview is as slow to scan as any 100M-row table — the win comes from the index, not from being a matview.

### 10.3 Refresh scaling and the cadence floor

Refresh time is dominated by the defining query. Rough numbers for the §7.2-style aggregation:

| Base data scanned | Plain REFRESH | CONCURRENTLY (query + diff) |
|---|---|---|
| 1M rows | ~1–3 s | ~2–5 s |
| 10M rows | ~8–20 s | ~15–40 s |
| 41M rows (example) | ~25 s | ~45–90 s |
| 100M+ rows | minutes | several minutes |

**The cadence floor:** you cannot schedule refreshes closer together than one refresh takes to run (with headroom), or they pile up / overlap and starve the box. If `CONCURRENTLY` takes 90s, a 1-minute schedule is impossible — refreshes overlap and the `EXCLUSIVE` lock serializes them into a permanent backlog. This directly bounds achievable freshness: **minimum staleness ≈ refresh duration.** To refresh more often, you must make the refresh cheaper (bound the window, partition into per-period matviews and refresh only the current one).

### 10.4 Optimization techniques specific to matviews

- **Bound the defining query with a time window** (`WHERE created_at >= NOW() - INTERVAL '90 days'`) so refresh cost is proportional to recent data, not all history. Freeze older periods into separate, never-refreshed matviews.
- **Raise `work_mem` for the refresh session only** (`SET LOCAL work_mem = '512MB'` in the refresh transaction) to keep the big Sort/Hash in memory and avoid the disk-spill seen in §7.2 (`external merge Disk: 1.3GB`, `Batches: 128`).
- **Minimize indexes to what reads need** — every index is rebuilt/maintained on refresh, taxing every refresh.
- **Prefer `CONCURRENTLY` for availability, plain for a quiet window.** A nightly matview nobody reads at 3am can use a plain refresh (faster, no diff); a 24/7 dashboard matview needs `CONCURRENTLY`.
- **Consider `UNLOGGED`** for a purely derived cache to cut WAL and speed refresh — accepting it's empty after crash and absent on replicas (§6.10).
- **Refresh on a replica? You can't usefully** — a matview refresh is a write; it must run on the primary and replicate. Read replicas serve the matview but cannot refresh it.

### 10.5 Memory, CPU, WAL

- **Memory:** reads use `shared_buffers` (the matview's hot pages) like any table. Refreshes use `work_mem` for the Sort/Hash/Aggregate and, for `CONCURRENTLY`, additional temp space for the new-data table and the diff join.
- **CPU:** reads are cheap. Refreshes are CPU-heavy (the aggregation) — schedule them off-peak where possible.
- **WAL:** a plain refresh WAL-logs the entire new contents (a full rewrite). `CONCURRENTLY` WAL-logs only the delta rows (often far less WAL if few rows changed — a hidden bonus of `CONCURRENTLY` on low-churn matviews). `UNLOGGED` writes no WAL at all. On a busy primary, refresh WAL volume affects replication lag — size it in.

---

## 11. Node.js Integration

### 11.1 Reading a matview — just a table to `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Reads are ordinary SELECTs — the app doesn't know or care it's a matview.
async function getCategoryTopProducts(categoryId, limit = 25) {
  const { rows } = await pool.query(
    `SELECT product_id, product_name, units_sold, revenue,
            refreshed_at,
            EXTRACT(EPOCH FROM (NOW() - refreshed_at)) AS data_age_seconds
     FROM mv_product_sales_90d
     WHERE category_id = $1
     ORDER BY revenue DESC
     LIMIT $2`,
    [categoryId, limit]
  );
  return rows;
}
```

### 11.2 Triggering a refresh from application code

```javascript
// Called by a background worker / job runner — NOT in a request handler.
// CONCURRENTLY is essential so live dashboard reads never block.
async function refreshSalesMatview() {
  // A dedicated client with a long statement_timeout for the heavy refresh.
  const client = await pool.connect();
  try {
    await client.query(`SET statement_timeout = '10min'`);
    await client.query(
      `REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales_90d`
    );
  } finally {
    client.release();
  }
}
```

Notes that bite people:
- **Never refresh inside an HTTP request handler.** The refresh takes tens of seconds; the request will hang or time out and you'll tie up a pool connection. Do it in a scheduled worker.
- **`REFRESH` cannot run inside a larger transaction with `CONCURRENTLY`** in some contexts, and it takes locks — keep it in its own statement/connection, not bundled with other writes.
- **Set a generous `statement_timeout`** for the refresh connection; the app's default (often a few seconds) will abort the refresh.

### 11.3 A guarded, debounced refresh (skip if nothing changed)

```javascript
// Avoid wasted refreshes: only refresh if base data changed since last time.
async function refreshIfStale() {
  const client = await pool.connect();
  try {
    const { rows } = await client.query(
      `SELECT MAX(created_at) AS latest FROM orders WHERE status = 'completed'`
    );
    const latest = rows[0].latest;
    const { rows: mvRows } = await client.query(
      `SELECT MAX(last_sold_at) AS mv_latest FROM mv_product_sales_90d`
    );
    if (latest && (!mvRows[0].mv_latest || latest > mvRows[0].mv_latest)) {
      await client.query(`SET statement_timeout = '10min'`);
      await client.query(
        `REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales_90d`
      );
      return { refreshed: true };
    }
    return { refreshed: false, reason: 'no new data' };
  } finally {
    client.release();
  }
}
```

### 11.4 Do ORMs handle matviews?

Mostly they treat a matview as a **read-only table** you map to and `SELECT` from — that part works everywhere. What no ORM does natively is the *lifecycle*: creating the matview, its unique index, and issuing `REFRESH ... CONCURRENTLY` almost always drop to a raw SQL migration and a raw `REFRESH` call. Details per ORM in the next section.

---

## 12. ORM Comparison

The consistent story: every ORM can **read** a matview (map it as a read-only entity/table), but **none** manage the matview's creation, unique index, or refresh through their high-level API. Those are always raw SQL.

### Prisma

**Can Prisma do it?** — Read: yes. Create/refresh: no, raw SQL only.

Prisma has no `materializedView` concept in its schema. You create the matview in a migration's raw SQL, then map a read-only model to it.

```prisma
/// Mapped to a materialized view. Prisma treats it as a table for reads.
model ProductSales90d {
  productId   Int     @id @map("product_id")
  productName String  @map("product_name")
  categoryId  Int     @map("category_id")
  unitsSold   Int     @map("units_sold")
  revenue     Decimal
  refreshedAt DateTime @map("refreshed_at")

  @@map("mv_product_sales_90d")
}
```

```typescript
// Reads work exactly like any model:
const top = await prisma.productSales90d.findMany({
  where: { categoryId: 17 },
  orderBy: { revenue: 'desc' },
  take: 25,
});

// Create + refresh are raw SQL:
await prisma.$executeRawUnsafe(`
  CREATE MATERIALIZED VIEW mv_product_sales_90d AS SELECT ... WITH NO DATA;
`);
await prisma.$executeRawUnsafe(`
  CREATE UNIQUE INDEX mv_psales90_pk ON mv_product_sales_90d (product_id);
`);
await prisma.$executeRawUnsafe(
  `REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales_90d`
);
```

**Where it breaks:** Prisma migrations don't model matviews, so `prisma migrate` won't diff them — you hand-write the SQL and Prisma may warn about drift. It also assumes it can write to mapped models; keep the app from ever `create`/`update`-ing this model.

**Verdict:** Fine as a typed read layer; use `$executeRaw` for all lifecycle. Mark the model conceptually read-only.

### Drizzle ORM

**Can Drizzle do it?** — Best-in-class: it has a first-class `pgMaterializedView` builder.

```typescript
import { pgMaterializedView, integer, numeric, timestamp } from 'drizzle-orm/pg-core';

export const productSales90d = pgMaterializedView('mv_product_sales_90d').as((qb) =>
  qb.select({ /* ... */ }).from(/* ... */)
);

// Reads:
const rows = await db.select().from(productSales90d)
  .where(eq(productSales90d.categoryId, 17));

// Drizzle even exposes refresh helpers:
await db.refreshMaterializedView(productSales90d).concurrently();
// Plain: await db.refreshMaterializedView(productSales90d);
```

**Where it breaks:** the unique index (required for `.concurrently()`) is still a normal index declaration you must remember to add; Drizzle won't infer it from the refresh call. Complex defining queries sometimes still need `sql` templates.

**Verdict:** The only mainstream ORM with genuine matview support, including a `.concurrently()` refresh. Preferred when matviews are central to the app.

### Sequelize

**Can Sequelize do it?** — Read: yes via a model with `{ freezeTableName: true }` pointed at the matview name. Create/refresh: raw `sequelize.query`.

```javascript
const ProductSales90d = sequelize.define('ProductSales90d', {
  productId: { type: DataTypes.INTEGER, primaryKey: true, field: 'product_id' },
  revenue:   DataTypes.DECIMAL,
}, { tableName: 'mv_product_sales_90d', timestamps: false });

// Read:
await ProductSales90d.findAll({ where: { categoryId: 17 },
  order: [['revenue', 'DESC']], limit: 25 });

// Lifecycle — raw:
await sequelize.query(
  'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales_90d'
);
```

**Where it breaks:** Sequelize thinks it's a writable table; guard against `.create()`/`.update()`. Migrations (umzug/sequelize-cli) hold the `CREATE MATERIALIZED VIEW` as raw SQL.

**Verdict:** Works as a read model; everything else is `sequelize.query`.

### TypeORM

**Can TypeORM do it?** — Yes, notably: it has a `@ViewEntity({ materialized: true })` decorator.

```typescript
@ViewEntity({
  materialized: true,
  name: 'mv_product_sales_90d',
  expression: `SELECT ... GROUP BY oi.product_id`,
})
export class ProductSales90d {
  @ViewColumn() productId: number;
  @ViewColumn() revenue: string;
}

// Read via repository:
await dataSource.getRepository(ProductSales90d).find({
  where: { categoryId: 17 }, order: { revenue: 'DESC' }, take: 25,
});

// Refresh — raw query:
await dataSource.query(
  'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales_90d'
);
```

**Where it breaks:** TypeORM will `CREATE MATERIALIZED VIEW` from the entity on synchronize/migration, but it does **not** manage the unique index needed for `CONCURRENTLY` (add it in a migration) and has no refresh abstraction — always raw. Changing the `expression` triggers a drop/recreate migration.

**Verdict:** Decent — `@ViewEntity({ materialized: true })` handles creation and reads; you still hand-write the unique index and the refresh.

### Knex.js

**Can Knex do it?** — Via `knex.schema` there is no matview helper, but `knex.raw` covers everything cleanly and it's the most transparent option.

```javascript
// Migration:
exports.up = async (knex) => {
  await knex.raw(`CREATE MATERIALIZED VIEW mv_product_sales_90d AS SELECT ... WITH NO DATA`);
  await knex.raw(`CREATE UNIQUE INDEX mv_psales90_pk ON mv_product_sales_90d (product_id)`);
  await knex.raw(`REFRESH MATERIALIZED VIEW mv_product_sales_90d`);
};
exports.down = (knex) => knex.raw(`DROP MATERIALIZED VIEW mv_product_sales_90d`);

// Read with the query builder (it's just a relation):
const rows = await knex('mv_product_sales_90d')
  .where({ category_id: 17 }).orderBy('revenue', 'desc').limit(25);

// Refresh:
await knex.raw('REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_sales_90d');
```

**Where it breaks:** nothing surprising — Knex never pretends to abstract matviews, so there's no leaky abstraction. You own the SQL, which for matviews is the right level.

**Verdict:** Clean. Query builder for reads, `knex.raw` for lifecycle. Most transparent.

### ORM Summary Table

| ORM | Matview creation | Read support | Refresh API | Unique-index-aware | Verdict |
|---|---|---|---|---|---|
| Prisma | Raw SQL (`$executeRaw`) | Yes (mapped model) | Raw SQL | No | Typed reads, raw lifecycle |
| Drizzle | `pgMaterializedView` | Yes (typed) | `.refreshMaterializedView().concurrently()` | Manual index | Best support |
| Sequelize | Raw SQL | Yes (model → tableName) | Raw SQL | No | Read model + raw |
| TypeORM | `@ViewEntity({materialized:true})` | Yes (view entity) | Raw SQL | No (add index by hand) | Good; index+refresh raw |
| Knex | `knex.raw` | Yes (query builder) | `knex.raw` | Manual index | Most transparent |

Universal rule: **the `REFRESH` (especially `CONCURRENTLY`) and the required unique index are your responsibility in every ORM.** Never assume the ORM keeps a matview fresh.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `orders(id, customer_id, total_amount, status, created_at)`:

1. Create a materialized view `mv_monthly_revenue` holding, per calendar month, the number of completed orders and total completed revenue.
2. Add whatever index is required to allow refreshing it `CONCURRENTLY`.
3. Write the read query that returns the last 12 months, newest first.
4. Write the command that brings it up to date without blocking readers.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topic 11 JOINs and Topic 20 GROUP BY)

Given `orders`, `order_items(id, order_id, product_id, quantity, unit_price)`, `products(id, name, category_id)`, and `categories(id, name)`:

Build a materialized view `mv_category_daily` with one row per (category, day) for completed orders, containing: distinct order count, units sold, gross revenue, and the timestamp the row was computed. Then add the unique index for concurrent refresh and a secondary index that makes "one category over time, newest first" reads use an index scan with no Sort node. Explain why the unique-index columns must be exactly `(category_id, day)`.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A team has a matview `mv_product_sales_90d` refreshed nightly by a plain `REFRESH MATERIALIZED VIEW`. Users complain the customer-facing product dashboard "freezes for about a minute every night at 2am." A junior proposes "run the refresh more often so it's smaller each time."

1. Explain why running a *plain* refresh more often does **not** fix the freeze and in fact makes it worse.
2. Give the actual fix, including the exact prerequisite object that must exist first and why.
3. The matview scans all 90 days of a 41M-row `order_items` table every refresh and now takes 90 seconds even concurrently, so it can't be scheduled more often than ~2 minutes. Propose a redesign that lets the "today" numbers refresh every 30 seconds while older days refresh rarely. What Postgres limitation are you working around?

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

You're asked in a system-design interview: "We have a dashboard that runs a 20-second aggregation query on every page load and it's killing the database. Walk me through how you'd fix it, and what could go wrong with your fix."

Write out the full answer you'd give: the tool you'd choose and why, the refresh strategy and how you'd pick the cadence, the exact locking pitfall and its prerequisite, and one scenario where a matview would be the *wrong* choice.

```sql
-- Write your query here (plus prose)
```

---

## 14. Interview Questions

### Q1 — "What's the difference between a view and a materialized view?"

**Junior answer:** "A view is virtual and a materialized view is stored."

**Principal answer:** A regular view stores no data — it's a saved query that Postgres macro-expands into every statement that references it, so it always reflects live data but pays the full query cost on every read. A materialized view stores the query's *result* as a real heap relation on disk (`pg_class.relkind = 'm'`), so reads are a cheap indexed scan, but the data is a snapshot as of the last `REFRESH` and does not auto-update. The trade is: view = always fresh, expensive per read; matview = cheap per read, bounded-stale. You choose a matview when the query is expensive, reads dominate writes, and some staleness is tolerable.

**Interviewer follow-up:** "Can the planner automatically use a matview for a query that matches its definition?" → No. Postgres does not do automatic query rewrite (unlike Oracle). You must reference the matview by name explicitly.

### Q2 — "You refresh a materialized view every 5 minutes and users report the dashboard hangs during the refresh. Why, and how do you fix it?"

**Junior answer:** "Make the refresh faster or run it less often."

**Principal answer:** A plain `REFRESH MATERIALIZED VIEW` takes an `ACCESS EXCLUSIVE` lock for its entire duration, which conflicts with the `ACCESS SHARE` lock every `SELECT` takes — so all readers block until the refresh commits. Under load, blocked connections exhaust the pool and the impact spreads. The fix is `REFRESH MATERIALIZED VIEW CONCURRENTLY`, which takes only an `EXCLUSIVE` lock (doesn't conflict with reads) by diffing new vs old rows and applying only deltas via MVCC DML. Its prerequisite is a **unique index** on the matview with no `WHERE` clause, because the diff needs a key to match rows on. `CONCURRENTLY` costs more total work (it runs the query *plus* a diffing full-outer-join), but it never blocks readers — the right trade for a 24/7 dashboard.

**Interviewer follow-up:** "What if `CONCURRENTLY` now takes 3 minutes and you want data fresher than that?" → You've hit the cadence floor (min staleness ≈ refresh duration). Make the refresh cheaper: bound the query to a recent time window, or split into per-period matviews and only refresh the current period, since Postgres has no native incremental refresh.

### Q3 — "How would you decide between a materialized view and a summary table maintained by triggers?"

**Junior answer:** "A matview is easier so I'd use that."

**Principal answer:** It's a freshness-vs-complexity trade. A matview is simple — one `REFRESH` command re-computes everything — but it's non-incremental (full re-query each refresh) and can only be as fresh as the cadence the refresh duration allows. A summary table you maintain incrementally (`UPDATE totals SET revenue = revenue + NEW.amount` on write) can be near-real-time and cheap per read, but *you* own the correctness of every delta path: inserts, updates, deletes, out-of-order and concurrent writes, and backfills. If bounded staleness (minutes/hours) is acceptable and the query is a periodic rollup, a matview wins on simplicity. If you genuinely need real-time aggregates *and* cheap reads — e.g., a live account balance — a summary table (or a stream processor) is the right tool despite the complexity, because a matview simply cannot be fresh enough without refreshing constantly.

**Interviewer follow-up:** "Where does a plain regular view fit?" → When the underlying query is cheap, or when you need absolute freshness and can afford the per-read cost — no storage, no refresh, always live.

### Q4 — "A developer added an AFTER INSERT trigger that refreshes the matview so it's always current. What happens?"

**Junior answer:** "The data stays fresh, seems fine."

**Principal answer:** It's catastrophic. Each insert now triggers a full re-query of the entire matview — turning an O(1), sub-millisecond insert into an O(N) operation that can take tens of seconds over millions of base rows. Under any real write rate the inserts serialize and the system collapses; it also invites lock conflicts and errors because the refresh contends with the very table the trigger fired on. Event-driven refresh must be **debounced** — coalesce many changes into at most one refresh per interval (a periodic job, or a "dirty flag" that a single worker acts on), and use `CONCURRENTLY` so those refreshes don't block reads. Real-time freshness with per-write correctness is a summary-table job, not a matview job.

**Interviewer follow-up:** "How would you implement the debounce?" → A `dirty` boolean/timestamp set by a lightweight trigger, plus a worker (pg_cron or app scheduler) that refreshes only when dirty and clears the flag — bounding refreshes to one per interval regardless of write volume.

---

## 15. Mental Model Checkpoint

1. A matview was refreshed at 14:00. At 14:30 someone inserts 500 matching orders and immediately queries the matview. What do they see, and why is there no error?

2. You run `REFRESH MATERIALIZED VIEW CONCURRENTLY mv` and get an error about needing a unique index. Explain *mechanically* why the concurrent path requires a unique index but the plain path does not.

3. A matview's plain refresh takes 40 seconds. Two dashboard users run `SELECT` at second 10 of the refresh. Describe exactly what happens to their queries and when they return.

4. Why is putting `ORDER BY revenue DESC LIMIT 100` inside a matview's *definition* usually a mistake? Give both the correctness reason and the "the sort doesn't stick" reason.

5. You have a 100M-row matview and reads are slow. Is "it's a matview" the reason it's slow? What's the actual fix, and how is it identical to fixing a slow table?

6. Your `CONCURRENTLY` refresh takes 90 seconds, and the business wants data no more than 60 seconds stale. Explain why you can't simply schedule the refresh every 60 seconds, and outline a redesign that could meet the SLA.

7. Refreshing matview A does not refresh matview B built on top of A. What does this tell you about how Postgres tracks matview dependencies for refresh, and what must you do?

---

## 16. Quick Reference Card

```sql
-- CREATE (WITH DATA is the default; WITH NO DATA makes an empty, unscannable shell)
CREATE MATERIALIZED VIEW mv AS SELECT ... GROUP BY ...  WITH DATA;
CREATE MATERIALIZED VIEW mv AS SELECT ...               WITH NO DATA;

-- REQUIRED for CONCURRENTLY: a UNIQUE index, NO WHERE clause, on the row grain
CREATE UNIQUE INDEX mv_pk ON mv (group_col_a, group_col_b);

-- REFRESH
REFRESH MATERIALIZED VIEW mv;               -- ACCESS EXCLUSIVE: BLOCKS all readers
REFRESH MATERIALIZED VIEW CONCURRENTLY mv;  -- EXCLUSIVE: readers OK; needs unique idx

-- Populate a WITH NO DATA matview (first refresh cannot be CONCURRENTLY if empty
-- and unpopulated in some setups — do a plain first refresh, then concurrent after)
REFRESH MATERIALIZED VIEW mv;

-- Index it like any table (each index is a refresh tax)
CREATE INDEX mv_read ON mv (category_id, revenue DESC);

-- Drop (base tables can't be dropped while a matview depends on them)
DROP MATERIALIZED VIEW mv;

-- Inspect
SELECT relname, relispopulated FROM pg_class WHERE relkind = 'm';
\d+ mv        -- psql: shows it's a matview + its indexes
```

**Perf rules of thumb**
- Read cost = cheap indexed scan (index it for reads). Refresh cost = full defining query (+ diff for CONCURRENTLY).
- Minimum achievable staleness ≈ refresh duration. To go fresher, make the refresh cheaper (bound the window / per-period matviews).
- Plain refresh blocks readers (`ACCESS EXCLUSIVE`); `CONCURRENTLY` never does but does more total work and needs a unique index.
- Postgres has **no** native incremental refresh — every refresh re-runs the whole query.
- The planner never auto-substitutes a matview; reference it by name.

**Interview one-liners**
- "View = saved query, always fresh, pays per read. Matview = saved *result*, cheap per read, stale until refresh."
- "`CONCURRENTLY` trades refresh speed for reader availability, and its price of admission is a unique index."
- "The whole design is staleness vs refresh cost: pick the cheapest cadence that stays inside the business tolerance."
- "Never refresh-per-row in a trigger — debounce, or use a summary table for real-time."
- "Right for expensive, read-heavy, staleness-tolerant queries; wrong for billing, balances, and inventory-at-checkout."

---

## Connected Topics

- **Topic 65 — Views (Regular)**: The always-fresh, zero-storage counterpart; understand the view before the matview — the matview is a view whose result is frozen to disk.
- **Topic 66 — Full-Text Search** *(previous)*: A prime matview use case — materialize a `tsvector` column and its GIN index so search reads don't recompute vectors live.
- **Topic 68 — EXPLAIN Options Deep Dive** *(next)*: The tool for profiling both the matview read plan and the defining query that a refresh runs (Sort spills, Hash batches, buffers).
- **Topic 11 — INNER JOIN in Depth**: Matview defining queries are usually big joins; the fan-out and aggregation rules there govern what you store.
- **Topic 20 — GROUP BY Fundamentals**: Pre-computing an expensive `GROUP BY` is the number-one reason matviews exist; the grouping key becomes the matview's unique index.
- **Internals — MVCC & Locking**: Why plain refresh (`ACCESS EXCLUSIVE`) blocks readers while `CONCURRENTLY` (`EXCLUSIVE` + delta DML) does not.
- **Internals — Heap Storage, WAL & the Buffer Pool**: A matview is a real heap — indexed, WAL-logged (unless `UNLOGGED`), cached in `shared_buffers` — which is exactly why reads are table-fast.
- **Internals — B-tree & GIN Indexes**: Indexing a matview is identical to indexing a table; the required unique index for `CONCURRENTLY` is an ordinary B-tree.
