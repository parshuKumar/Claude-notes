# Topic 76 — System Design SQL Questions

### SQL Mastery Curriculum — Phase 11: SQL for Backend Developer Interviews

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you're an architect and someone walks up and says: "Design me a house."

A junior architect immediately starts drawing walls. Where does the bathroom go? What color are the tiles? They jump to details before asking a single question.

A senior architect first asks: "How many people live here? Do you host guests? Do you work from home? What's your budget? What's the plot size? Will you extend it later?" Only *then* do they draw — and every wall they draw has a reason behind it.

SQL system-design interview questions are exactly this. When the interviewer says "design the schema for a notifications system," they are not testing whether you can type `CREATE TABLE`. They are testing whether you ask about **access patterns** before you draw tables. Who reads this data? How often? How much of it? What has to be fast? What can be slow? What's allowed to be stale?

There are really only three shapes of question you'll ever get:

1. **"Design the schema for X."** — You produce entities, keys, indexes, constraints, and you *justify every tradeoff*.
2. **"How would you query Y efficiently at scale?"** — You start from the access pattern, then reach for indexes, denormalisation, caching, and partitioning.
3. **"This query is slow — diagnose it."** — You read `EXPLAIN`, find the villain (Seq Scan, spilling sort, bad join order, missing index), and fix it.

The whole trick is: **never start drawing walls before you've asked who lives in the house.** The candidate who says "It depends on the read pattern — let me ask a few questions" has already scored higher than the one who blurts out three tables.

---

## 2. Connection to SQL Internals

System-design SQL sits *above* the individual features you've learned in Topics 01–75, but every design decision cashes out into a concrete engine behaviour. The reason this topic works as a capstone is that a good design answer names the internal that makes it fast:

- **B-tree indexes** (Topic 30) — every "make this query fast" answer eventually names a B-tree on the right column set in the right order. Composite index column order maps directly to which access patterns get an index-only scan versus a filter.
- **Heap pages and the visibility map** — "covering index" and "index-only scan" answers depend on the visibility map being set (which depends on `VACUUM`). A design that ignores dead-tuple bloat gets slower over time even with perfect indexes.
- **MVCC** (Topic 06) — an append-heavy audit log or notifications table generates dead tuples on every `UPDATE`. Understanding that `UPDATE` is `DELETE`+`INSERT` under MVCC is why "mark as read" at scale is a design smell.
- **The planner and statistics** — join order, algorithm choice (Nested Loop / Hash Join / Merge Join, Topic 17), and row estimates all come from `pg_statistic`. "The query got slow after a data-shape change" is almost always stale stats or a plan flip.
- **The buffer pool (`shared_buffers`) and OS page cache** — "will this be fast?" depends on whether the working set fits in RAM. A 2 TB audit table scanned by a report will never be cache-resident; a hot 5 GB notifications window will be.
- **WAL and write amplification** — every index you add multiplies write cost. A design with eight indexes on a write-heavy table is a design that will fall over under insert load. WAL is why "just add an index" is never free.
- **`work_mem`, sorts, and hash tables** — feed queries (ORDER BY on huge sets), aggregations, and hash joins spill to disk when they exceed `work_mem`. Pagination strategy (keyset vs OFFSET) is fundamentally about not sorting 10M rows to return 20.
- **Partition pruning** — range/list partitioning (Topic 52) lets the planner skip whole tables. "Audit log at scale" is a partitioning question in disguise.

The senior signal is connecting a design choice to the internal. Not "I'd add an index" but "I'd add a composite B-tree on `(user_id, created_at DESC)` so the feed query is an index range scan that returns pre-sorted rows and never touches a Sort node."

---

## 3. Logical Execution Order Context

System-design questions don't live at one clause — they span the whole pipeline. But the framework for answering them mirrors the logical execution order, and knowing where a cost lands tells you which lever to pull:

```
FROM / JOIN      ← schema & partitioning decisions live here
                   (how many tables, how big, partition pruning)
WHERE            ← index selectivity decisions live here
                   (which composite index, which leading column)
GROUP BY         ← pre-aggregation / rollup table decisions live here
                   (materialise counts instead of aggregating live)
HAVING           ← rarely the bottleneck; runs on grouped rows
SELECT / window  ← covering-index (index-only scan) decisions live here
DISTINCT         ← a smell in feeds; usually a fan-out bug from JOIN
ORDER BY         ← keyset pagination vs Sort node decision lives here
LIMIT            ← "top N" — the reason keyset pagination beats OFFSET
```

When you diagnose a slow query, you walk this pipeline top-down and ask "where is the row count exploding, and where is it collapsing?" The villain is almost always a stage that processes far more rows than it returns:

- A `Seq Scan` in **FROM** on a 100M-row table when **WHERE** only needs 50 rows → missing/unusable index.
- A **Sort** feeding **LIMIT 20** that sorts 10M rows → OFFSET pagination or a missing index on the ORDER BY key.
- A **GROUP BY** over the full table on every page load → should be a pre-aggregated rollup.

The design framing is the same pipeline read backwards: start from what **LIMIT/SELECT** must return fast, and design **FROM/WHERE** (schema + indexes) so the early stages never touch more rows than necessary.

---

## 4. What Is a SQL System-Design Question?

A SQL system-design question asks you to reason about **data at scale under real access patterns**, not to recall syntax. It comes in three canonical forms, and the entire skill is recognising which form you're in and applying its framework.

```
SQL SYSTEM-DESIGN QUESTION
│
├── TYPE A: "Design the schema for X"
│   └── Deliverable: entities, keys, indexes, constraints, tradeoffs
│       │
│       ├── Entities & relationships   │ what are the nouns? 1:1, 1:N, M:N?
│       ├── Primary keys               │ natural vs surrogate; bigint vs uuid
│       ├── Foreign keys & constraints │ referential integrity, CHECK, NOT NULL
│       ├── Indexes                    │ one per access pattern, composite order
│       ├── Normalisation level        │ 3NF baseline, denormalise deliberately
│       └── Tradeoffs stated out loud  │ "I chose X over Y because <access pattern>"
│
├── TYPE B: "Query Y efficiently at scale"
│   └── Deliverable: access-pattern-driven plan
│       │
│       ├── Enumerate access patterns  │ reads/writes, frequency, latency budget
│       ├── Index strategy             │ composite, covering, partial
│       ├── Denormalisation            │ precompute counters, cache columns
│       ├── Caching layer              │ app cache / Redis / materialised view
│       └── Partitioning & sharding    │ range/list partitions, then shard
│
└── TYPE C: "This query is slow — diagnose it"
    └── Deliverable: EXPLAIN-driven diagnosis
        │
        ├── Read EXPLAIN (ANALYZE, BUFFERS) │ find actual vs estimated gap
        ├── Identify the villain            │ Seq Scan, Sort spill, bad join order
        ├── Check statistics                │ stale stats → wrong plan
        ├── Propose the fix                 │ index, rewrite, config, schema change
        └── State the tradeoff of the fix   │ write cost, storage, staleness
```

The precise definition: a SQL system-design question evaluates whether you can **translate a business requirement into a data model and a query strategy that stays correct and fast as data volume, concurrency, and access frequency grow** — and whether you can *articulate the tradeoffs* of each decision. The tradeoff articulation is the whole game. Any bootcamp graduate can produce three tables. Only a senior says "I denormalised the unread count onto `users` because the alternative — `COUNT(*)` on a 500M-row notifications table on every page load — would not survive production, and I accept the write-time cost of keeping it consistent."

---

## 5. Why System-Design SQL Mastery Matters in Production

This is not an academic exercise. Every schema decision you make in an interview is a decision you'll make (and live with for years) in production. The cost of getting it wrong is real:

1. **Schema is the hardest thing to change later.** You can rewrite a query in an afternoon. Migrating a 500M-row table's primary key from `int` to `bigint` after it overflowed at 2.1 billion rows is a multi-week, downtime-risking project. The Instagram/Reddit-style "we ran out of 32-bit IDs" incident is a design failure that shipped years earlier. Getting the schema right up front is the highest-leverage decision in the system.

2. **The wrong index strategy silently degrades.** A query that runs in 5 ms at 10K rows and 5 seconds at 50M rows passes every test in development and dies in production six months after launch. Nobody notices until the table crosses the threshold where the Seq Scan stops fitting in cache. Designing for the target scale, not the launch scale, is what separates systems that survive growth from ones that need an emergency rewrite.

3. **`COUNT(*)` on the read path is a time bomb.** "Show the unread notification badge" implemented as `SELECT COUNT(*) FROM notifications WHERE user_id = $1 AND read = false` runs on *every page load for every user*. At 10M daily active users that's 10M+ counts per day against a growing table. The denormalised-counter design costs a few writes; the naive design costs an outage.

4. **Missing partitioning turns retention into an outage.** An audit log with no partitioning that needs "delete everything older than 90 days" becomes a `DELETE` of 200M rows that locks the table, bloats it with dead tuples, and takes hours. With range partitioning by month, the same operation is `DROP TABLE audit_2026_01` — instant, no bloat.

5. **Interviews gate senior roles on exactly this.** Junior interviews test whether you can write a JOIN. Staff and principal interviews test whether you can design the notifications backend for 100M users and defend every tradeoff under follow-up pressure. This topic *is* the differentiator between an L4 and an L6 loop.

---

## 6. Deep Technical Content

### 6.1 The Discovery Phase — Questions Before Tables

Before writing a single `CREATE TABLE`, a senior candidate spends 2–4 minutes on discovery. Skipping this is the number-one reason strong engineers bomb design interviews. The questions to ask, in order:

**Scale questions:**
- How many users? (10K vs 100M changes everything)
- How many of the primary entity? (notifications/day, orders/day, audit events/sec)
- Read/write ratio? (a feed is read-heavy; an audit log is write-heavy)
- Growth rate and retention? (keep forever vs 90-day window)

**Access-pattern questions:**
- What are the exact read queries? ("show me the last 20 notifications for a user, newest first")
- What's the latency budget? (feed must be <100 ms; a report can take 10 s)
- What must be strongly consistent vs eventually consistent? (unread badge can lag 1 s; a payment cannot)
- What's the query concurrency? (100 QPS vs 100K QPS)

**Correctness questions:**
- What are the uniqueness constraints? (one payment per order? idempotency keys?)
- What can be NULL and what cannot?
- Are there soft-delete or audit requirements?

You do not need answers to *all* of these. The signal is that you *ask*, then state your assumptions explicitly: "I'll assume 100M users, 50 notifications per user per month retained for 6 months, feed reads at 50K QPS, unread-count reads on every page load. That points me toward denormalisation and partitioning — let me design for that."

### 6.2 Type A Framework — Designing a Schema

Once you have the access patterns, the schema design proceeds in a fixed order. Doing it in this order prevents the classic mistake of indexing before you know the queries.

**Step 1 — Identify entities and relationships.** The nouns become tables. The relationships determine foreign keys and whether you need a join table:

- **1:1** — usually fold into one table unless there's a size or access-frequency reason to split (e.g. a rarely-read `user_profiles` blob split from a hot `users` table).
- **1:N** — foreign key on the "many" side (`orders.customer_id → users.id`).
- **M:N** — a join table (`order_items` between `orders` and `products`, carrying its own columns like `quantity`, `unit_price`).

**Step 2 — Choose primary keys.** The default for a high-write table is a surrogate `bigint GENERATED ALWAYS AS IDENTITY`. Reasons:

- `bigint` not `int` — `int` overflows at ~2.1 billion. On a table doing 10M inserts/day that's ~210 days. Always `bigint` for anything that grows.
- Sequential `bigint` over random `uuid v4` for the *clustered access* case — random UUIDs scatter inserts across the B-tree causing page splits and poor cache locality (index bloat, more WAL). If you need client-generated or non-guessable IDs, prefer `uuid v7` (time-ordered) or ULID, which preserve insert locality.
- Natural keys (email, SKU) as PK are usually a mistake — they change, they're wide, and they propagate into every foreign key. Use a surrogate PK and put a `UNIQUE` constraint on the natural key.

**Step 3 — Add constraints for correctness.** Constraints are free correctness the database enforces so your application code doesn't have to:

- `NOT NULL` on everything that logically can't be null.
- `FOREIGN KEY` with an explicit `ON DELETE` action (`CASCADE`, `RESTRICT`, `SET NULL`) — decide it deliberately.
- `UNIQUE` for business uniqueness (one payment per order: `UNIQUE (order_id)`).
- `CHECK` for domain rules (`CHECK (amount > 0)`, `CHECK (status IN (...))`).
- Idempotency: a `UNIQUE` on a client-supplied `idempotency_key` turns "did this request already run?" into a database guarantee.

**Step 4 — Design indexes, one per access pattern.** This is where most of the interview signal lives. The rule: **you cannot design indexes until you've written down the queries.** For each read query, ask "what's the WHERE, what's the ORDER BY?" and build a composite index that serves both.

**Step 5 — State the tradeoffs.** For every non-obvious choice, say the alternative and why you rejected it. This is the difference between an answer and a *senior* answer.

### 6.3 Index Design — The Heart of the Answer

Indexes are where design theory meets engine reality. The rules that matter in an interview:

**Composite index column order follows the "equality, then range, then sort" rule.** For a query like:

```sql
SELECT * FROM notifications
WHERE user_id = $1 AND read = false
ORDER BY created_at DESC
LIMIT 20;
```

The optimal index is `(user_id, read, created_at DESC)`:
- `user_id` first — it's an equality predicate, most selective, leading column.
- `read` second — also equality.
- `created_at DESC` last — it's the sort key; putting it last (in matching direction) means the index range scan returns rows *already sorted*, so no Sort node, and `LIMIT 20` stops after 20 index entries.

Get the order wrong — say `(created_at, user_id, read)` — and the index is nearly useless for this query because the leading column isn't in the WHERE.

**Covering indexes enable index-only scans.** If the query only needs a few columns, `INCLUDE` them so the query never touches the heap:

```sql
CREATE INDEX idx_notif_feed ON notifications (user_id, created_at DESC)
  INCLUDE (title, body_preview);
```

Now the feed query is an index-only scan (assuming the visibility map is set via `VACUUM`) — no heap fetch per row. This is a real 2–5× win on wide tables.

**Partial indexes shrink the index to the hot subset.** If 95% of notifications are read and you only ever query unread:

```sql
CREATE INDEX idx_notif_unread ON notifications (user_id, created_at DESC)
  WHERE read = false;
```

This index is a fraction of the size, stays in cache, and is cheaper to maintain — because read notifications aren't in it at all.

**Every index has a write cost.** Say this out loud: each index is another B-tree that every `INSERT`/`UPDATE`/`DELETE` must maintain, and more WAL. On a write-heavy table, "add an index per column" is wrong. You index for the *actual read patterns*, and you accept that write-heavy tables carry fewer indexes.

### 6.4 Denormalisation — Deliberate, Not Accidental

Normalisation (3NF) is the default: no duplicated data, every fact in one place. You **denormalise deliberately** when a read pattern can't be served fast enough by joins and aggregation. The canonical cases:

**Counter columns.** Instead of `COUNT(*)` on every read, keep a maintained counter:

```sql
-- Instead of counting 500M rows on every page load:
ALTER TABLE users ADD COLUMN unread_count int NOT NULL DEFAULT 0;
-- Maintained on write (trigger or app logic):
--   on new notification:  UPDATE users SET unread_count = unread_count + 1 ...
--   on mark-read:         UPDATE users SET unread_count = unread_count - 1 ...
```

The tradeoff: the counter can drift under concurrency or bugs, so you run a periodic reconciliation job that recomputes the true count. You accept eventual correctness on the badge in exchange for O(1) reads.

**Cached/embedded columns.** Copy a slowly-changing parent attribute onto the child to avoid a join on the hot path — e.g. storing `author_name` on `posts` so the feed doesn't join `users`. Tradeoff: when the author renames, you must update denormalised copies (a background job), and you accept brief staleness.

**Precomputed rollups / summary tables.** For "revenue per category per day," don't aggregate 20M `order_items` on every dashboard load. Maintain a `daily_category_revenue` rollup table updated incrementally or via a scheduled job / materialised view.

The senior framing: **normalise for correctness, denormalise for a specific proven read pattern, and always name the consistency cost you're taking on.**

### 6.5 Partitioning and Sharding — Scaling the Table Itself

When a single table gets too big for indexes to keep it fast, or retention/deletion becomes painful, you partition.

**Range partitioning by time** is the workhorse for append-heavy, time-series-shaped data (audit logs, notifications, events):

```sql
CREATE TABLE audit_logs (
  id          bigint GENERATED ALWAYS AS IDENTITY,
  actor_id    bigint,
  action      text,
  entity_type text,
  entity_id   bigint,
  created_at  timestamptz NOT NULL,
  payload     jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_logs_2026_07 PARTITION OF audit_logs
  FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
```

Wins:
- **Partition pruning** — a query with `WHERE created_at >= '2026-07-01'` only scans the July partition; the planner skips the rest.
- **Cheap retention** — dropping old data is `DROP TABLE audit_logs_2026_01`, instant and bloat-free, versus a giant `DELETE`.
- **Smaller indexes per partition** — each partition's index stays cache-friendly.

**List partitioning** suits categorical splits (by `region`, by `tenant_id` for a small number of large tenants).

**Sharding** is the next level — splitting across *multiple database servers*, typically by `user_id` hash. It's a big step (cross-shard queries and transactions become hard), so in an interview you reach for it only after single-node partitioning, read replicas, and caching are exhausted. The right answer is usually "partition first; shard only when a single node's write throughput or storage is the wall."

### 6.6 Caching Layers — Where Reads Go

Not every read should hit the primary. The layered answer:

- **Application-level / Redis cache** for hot, tolerant-of-staleness reads (the unread badge, a user's session, a rarely-changing config). Set a TTL and accept bounded staleness.
- **Materialised views** for expensive aggregations that can be refreshed on a schedule (`REFRESH MATERIALIZED VIEW CONCURRENTLY`). The dashboard reads the view, not the base tables.
- **Read replicas** to offload read traffic from the primary. The catch: replication lag means replicas are eventually consistent — fine for feeds and reports, wrong for read-after-write flows like "show the order I just placed."

The senior signal is matching each read to the cheapest layer that meets its consistency requirement, and naming the staleness you accept.

### 6.7 Type C Framework — Diagnosing a Slow Query

When handed a slow query, follow a fixed diagnostic sequence rather than guessing:

1. **Get the real plan:** `EXPLAIN (ANALYZE, BUFFERS)` — never reason from `EXPLAIN` alone; you need *actual* rows and timing.
2. **Find the actual-vs-estimated gap.** If the planner estimated 10 rows and got 2M, statistics are stale or a correlation the planner can't see is at play → the plan choice is built on a lie. `ANALYZE` the table.
3. **Find the villain node** — the one consuming most of the time:
   - `Seq Scan` on a big table under a selective WHERE → missing or unusable index.
   - `Sort` with `Sort Method: external merge Disk` → spilling; `work_mem` too small, or you're sorting far too many rows (fix pagination).
   - Nested Loop with high `loops` and a Seq Scan inner → missing index on the inner join key.
   - `Hash Join` with `Batches > 1` → hash spilling to disk.
4. **Check for the anti-patterns** that defeat indexes: a function on the indexed column (`WHERE lower(email) = ...` with no expression index), a leading wildcard `LIKE '%foo'`, an implicit type cast, or `OR` across different columns.
5. **Propose the fix and its tradeoff.** "Add `(user_id, created_at DESC)` — turns the Seq Scan + Sort into an index range scan, at the cost of one more index to maintain on writes."

The discipline is: *read the plan, name the node, name the fix, name the cost.* Never "I'd add some indexes."

### 6.8 Common Anti-Patterns to Name Proactively

Naming these before the interviewer asks is a strong signal:

- **`OFFSET` pagination at depth** — `OFFSET 100000 LIMIT 20` still scans and discards 100K rows. Use **keyset (seek) pagination**: `WHERE (created_at, id) < ($last_created, $last_id) ORDER BY created_at DESC, id DESC LIMIT 20`.
- **`SELECT *` on wide tables** — defeats covering indexes and ships bytes you don't need.
- **Unbounded `IN` lists** and giant `OR` chains — often better as a join or `= ANY($1::bigint[])`.
- **`COUNT(*)` for existence** — use `EXISTS` (Topic 27); it stops at the first row.
- **Storing enums as free-text** without a `CHECK` or lookup table — leads to `'active'`, `'Active'`, `'ACTIVE'` drift.
- **No index on foreign keys** — makes `ON DELETE CASCADE` and reverse lookups do Seq Scans.

---

## 7. EXPLAIN — Reading Plans in a Design Interview

In a Type C question you'll be handed a plan and asked to diagnose. Here are the three plans you must be able to read cold.

### 7.1 The Villain: Seq Scan + External Sort

A feed query with no supporting index on a 50M-row table:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, created_at
FROM notifications
WHERE user_id = 40217 AND read = false
ORDER BY created_at DESC
LIMIT 20;
```

```
Limit  (cost=2451020.44..2451020.49 rows=20 width=48)
       (actual time=8123.5..8123.5 rows=20 loops=1)
  ->  Sort  (cost=2451020.44..2451132.19 rows=44700 width=48)
            (actual time=8123.5..8123.5 rows=20 loops=1)
        Sort Key: created_at DESC
        Sort Method: external merge  Disk: 41208kB
        ->  Seq Scan on notifications
              (cost=0.00..2449832.00 rows=44700 width=48)
              (actual time=0.3..8003.1 rows=43912 loops=1)
              Filter: ((NOT read) AND (user_id = 40217))
              Rows Removed by Filter: 49956088
        Buffers: shared hit=812 read=1123456
Planning Time: 0.2 ms
Execution Time: 8140.6 ms
```

**Reading it:**
- `Seq Scan` reads all 50M rows; `Rows Removed by Filter: 49956088` — 99.9% of the work is thrown away. This is the smoking gun for a missing index.
- `Sort Method: external merge  Disk: 41208kB` — the sort spilled to disk because it's sorting 44K rows that shouldn't have been read at all.
- `Buffers: ... read=1123456` — over a million pages read from disk. This table is not cache-resident.
- 8.1 seconds. In production this query would time out.

**The fix:**

```sql
CREATE INDEX idx_notif_unread_feed
  ON notifications (user_id, created_at DESC)
  WHERE read = false;
```

### 7.2 The Fixed Plan: Index Range Scan, No Sort

Re-running the same query after adding the partial index:

```
Limit  (cost=0.56..33.21 rows=20 width=48)
       (actual time=0.04..0.11 rows=20 loops=1)
  ->  Index Scan using idx_notif_unread_feed on notifications
        (cost=0.56..73012.44 rows=44700 width=48)
        (actual time=0.03..0.09 rows=20 loops=1)
        Index Cond: (user_id = 40217)
  Buffers: shared hit=5
Planning Time: 0.3 ms
Execution Time: 0.14 ms
```

**Reading it:**
- `Index Scan using idx_notif_unread_feed` — the index leads with `user_id` (equality) and stores rows in `created_at DESC` order.
- **No Sort node** — the index already yields rows in the ORDER BY order, so `LIMIT 20` stops after 20 index entries.
- `Buffers: shared hit=5` — five cached pages, zero disk reads.
- 0.14 ms versus 8140 ms — a ~58,000× improvement from one correctly-ordered partial index.

### 7.3 The Aggregation Villain: Count on the Read Path

The unread badge implemented naively:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*) FROM notifications
WHERE user_id = 40217 AND read = false;
```

```
Aggregate  (cost=73124.44..73124.45 rows=1 width=8)
           (actual time=41.8..41.8 rows=1 loops=1)
  ->  Index Only Scan using idx_notif_unread_feed on notifications
        (cost=0.56..73012.44 rows=44700 width=0)
        (actual time=0.03..38.9 rows=43912 loops=1)
        Index Cond: (user_id = 40217)
        Heap Fetches: 0
  Buffers: shared hit=289
Planning Time: 0.2 ms
Execution Time: 41.9 ms
```

**Reading it:** even with a perfect index-only scan, counting 43,912 index entries takes ~42 ms — *per page load, per user*. At 50K QPS that's a wall. The design fix isn't a better index; it's **denormalising the count onto `users`** so the read is a single-row primary-key lookup (~0.05 ms). The EXPLAIN proves that no index can make a live `COUNT` cheap enough at this scale — the schema must change.

---

## 8. Query Examples

### Example 1 — Basic: Design and Query a Sessions Table

The interviewer says "design session storage and the lookup." Access pattern: look up a session by token on *every authenticated request* (very high QPS, must be <1 ms); delete expired sessions in bulk.

```sql
CREATE TABLE sessions (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  token_hash  bytea       NOT NULL,          -- store a hash, never the raw token
  user_id     bigint      NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at  timestamptz NOT NULL DEFAULT now(),
  expires_at  timestamptz NOT NULL,
  CONSTRAINT uq_sessions_token UNIQUE (token_hash)   -- the hot lookup path
);

-- Reverse lookup: "all sessions for a user" (for logout-everywhere)
CREATE INDEX idx_sessions_user ON sessions (user_id);

-- Bulk expiry sweep runs on this:
CREATE INDEX idx_sessions_expires ON sessions (expires_at);
```

```sql
-- The hot path: one unique-index probe, sub-millisecond
SELECT user_id, expires_at
FROM sessions
WHERE token_hash = $1 AND expires_at > now();
```

The `UNIQUE (token_hash)` doubles as the index for the hot lookup — no separate index needed. Tradeoff stated: I hash the token so a database leak doesn't expose live sessions.

### Example 2 — Intermediate: A Feed with Keyset Pagination

Access pattern: infinite-scroll feed, "next 20 items older than what I've seen," must stay fast at any scroll depth.

```sql
CREATE TABLE feed_items (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id     bigint      NOT NULL,
  author_id   bigint      NOT NULL,
  author_name text        NOT NULL,        -- denormalised to avoid a join on the hot path
  body        text        NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);

-- Serves both the WHERE (user_id) and the ORDER BY (created_at, id)
CREATE INDEX idx_feed_keyset ON feed_items (user_id, created_at DESC, id DESC);
```

```sql
-- First page:
SELECT id, author_name, body, created_at
FROM feed_items
WHERE user_id = $1
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page — keyset, NOT offset. $2/$3 = last row's (created_at, id):
SELECT id, author_name, body, created_at
FROM feed_items
WHERE user_id = $1
  AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

The composite index makes every page an index range scan of exactly 20 rows regardless of depth. `OFFSET 100000` would scan 100K rows per request; keyset scans 20. `id` is the tiebreaker so pagination is stable when two items share a timestamp.

### Example 3 — Production Grade: Notifications with Denormalised Unread Count

**Scenario.** 100M users, ~500M live notifications, feed read at 50K QPS, unread badge on every page load. Retention 6 months. `notifications` is range-partitioned by month. Index availability: partial index on unread per user. Performance expectation: feed <5 ms, badge <0.5 ms.

```sql
-- Denormalised counter for the O(1) badge read
ALTER TABLE users ADD COLUMN unread_count int NOT NULL DEFAULT 0;

CREATE TABLE notifications (
  id          bigint GENERATED ALWAYS AS IDENTITY,
  user_id     bigint      NOT NULL,
  type        text        NOT NULL,
  title       text        NOT NULL,
  body        text,
  read        boolean     NOT NULL DEFAULT false,
  created_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (id, created_at)               -- PK must include the partition key
) PARTITION BY RANGE (created_at);

CREATE TABLE notifications_2026_07 PARTITION OF notifications
  FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

-- Feed: partial index, unread only, pre-sorted
CREATE INDEX idx_notif_unread ON notifications (user_id, created_at DESC)
  WHERE read = false;
```

```sql
-- Badge: single-row PK lookup, ~0.05 ms
SELECT unread_count FROM users WHERE id = $1;

-- Feed: index range scan on the current partition, ~2 ms
SELECT id, type, title, created_at
FROM notifications
WHERE user_id = $1 AND read = false
  AND created_at >= now() - interval '30 days'
ORDER BY created_at DESC
LIMIT 20;
```

```
Limit  (actual time=0.05..1.9 rows=20 loops=1)
  ->  Index Scan using idx_notif_unread_2026_07 on notifications_2026_07
        Index Cond: (user_id = $1)
        Buffers: shared hit=6
Execution Time: 2.1 ms
```

The `created_at >= now() - interval '30 days'` predicate lets the planner prune to recent partitions. The badge never counts — it reads one maintained integer. Tradeoff: `unread_count` is maintained on write and reconciled nightly to correct drift.

---

## 9. Wrong → Right Patterns

### Wrong 1: Counting on the read path

```sql
-- WRONG: runs on every page load, for every user, against a growing table
SELECT COUNT(*) AS unread
FROM notifications
WHERE user_id = $1 AND read = false;
-- At 500M rows and 50K QPS this is ~42 ms per call (see §7.3) — a wall.
-- Even a perfect index-only scan still counts tens of thousands of entries.
```

Why it's wrong at the execution level: `COUNT(*)` must visit every matching index entry — the cost is proportional to the number of unread notifications, which for active users grows without bound. No index makes an O(N) count into an O(1) read.

```sql
-- RIGHT: denormalise the counter, read one row by PK
SELECT unread_count FROM users WHERE id = $1;   -- ~0.05 ms, PK lookup
-- Maintain unread_count on insert/mark-read; reconcile nightly to fix drift.
```

### Wrong 2: OFFSET pagination on a deep feed

```sql
-- WRONG: page 5000 of an infinite scroll
SELECT * FROM feed_items
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20 OFFSET 100000;
-- The engine scans and DISCARDS 100,000 rows to return 20.
-- Latency grows linearly with scroll depth.
```

Why it's wrong: `OFFSET n` is not a seek — the executor produces the first `n` rows in order and throws them away. Deep offsets read enormous amounts and get slower the further you scroll.

```sql
-- RIGHT: keyset (seek) pagination — always reads exactly 20 rows
SELECT * FROM feed_items
WHERE user_id = $1
  AND (created_at, id) < ($last_created_at, $last_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- With index (user_id, created_at DESC, id DESC) this is O(20) at any depth.
```

### Wrong 3: `int` primary key on a high-growth table

```sql
-- WRONG: will overflow
CREATE TABLE audit_logs (
  id serial PRIMARY KEY,   -- serial = int4, max 2,147,483,647
  ...
);
-- At 10M inserts/day, this overflows in ~215 days. The failure is a
-- production INSERT that suddenly errors with "integer out of range."
```

Why it's wrong: `serial`/`int` is 32-bit. The overflow is invisible in dev (small tables) and catastrophic in prod. Migrating `int → bigint` on a live 2B-row table is a multi-week project with rewrite/downtime risk.

```sql
-- RIGHT: bigint from day one for anything that grows
CREATE TABLE audit_logs (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,   -- int8, ~9.2 quintillion
  ...
);
```

### Wrong 4: Random UUID PK on a write-heavy table

```sql
-- WRONG: random v4 UUIDs as the clustered insert key
CREATE TABLE events (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),   -- v4 = fully random
  ...
);
-- Each insert lands at a random point in the B-tree → page splits,
-- poor cache locality, index bloat, extra WAL. Insert throughput degrades.
```

Why it's wrong at the engine level: a random key destroys insert locality. Sequential keys append to the right-most leaf page (hot in cache); random keys dirty pages all over the index, causing splits and write amplification.

```sql
-- RIGHT: time-ordered ID if you need non-sequential/global uniqueness
CREATE TABLE events (
  id uuid PRIMARY KEY DEFAULT uuidv7(),   -- v7 = time-ordered, preserves locality
  ...
);
-- Or a plain bigint identity if you don't need client-side/global generation.
```

### Wrong 5: No partitioning on a table with a retention policy

```sql
-- WRONG: 90-day retention enforced by DELETE on a 200M-row flat table
DELETE FROM audit_logs WHERE created_at < now() - interval '90 days';
-- Locks contention, generates 200M dead tuples, bloats the table and indexes,
-- and the space isn't reclaimed without a VACUUM FULL (which locks the table).
```

Why it's wrong: a mass `DELETE` under MVCC marks rows dead but doesn't free space; the table bloats and every subsequent query scans the tombstones until `VACUUM` catches up.

```sql
-- RIGHT: range-partition by time; retention is a metadata operation
DROP TABLE audit_logs_2026_01;   -- instant, no bloat, no dead tuples
-- Pair with a job that pre-creates next month's partition.
```

---

## 10. Performance Profile

Design decisions have to be justified with numbers. Here's how the canonical patterns scale.

### Scaling the Feed Query

| Table size | No index (Seq Scan + Sort) | Composite index range scan |
|-----------|----------------------------|----------------------------|
| 10K rows | ~2 ms (fits in cache) | ~0.1 ms |
| 1M rows | ~180 ms | ~0.15 ms |
| 10M rows | ~2 s | ~0.2 ms |
| 100M rows | ~25 s (mostly disk) | ~0.3 ms |

The Seq Scan grows linearly with table size; the index range scan is effectively flat because `LIMIT 20` stops after 20 index entries regardless of table size. This table is the whole argument for "design for target scale, not launch scale."

### Badge Read: Live COUNT vs Denormalised Counter

| Approach | 1M notifs | 100M notifs | 500M notifs |
|----------|-----------|-------------|-------------|
| `COUNT(*)` with index-only scan | ~1 ms | ~25 ms | ~42 ms |
| Denormalised `unread_count` (PK read) | ~0.05 ms | ~0.05 ms | ~0.05 ms |

The counter is *constant* regardless of scale; the count degrades with the number of unread rows per user. At 50K QPS the difference is the difference between a healthy primary and a saturated one.

### Write Cost vs Index Count

| Indexes on table | Relative INSERT cost | WAL volume |
|------------------|----------------------|-----------|
| 1 (PK only) | 1.0× | baseline |
| 3 | ~1.8× | ~1.7× |
| 6 | ~3.2× | ~3.0× |
| 10 | ~5× | ~4.8× |

Every index is a B-tree that each write maintains, plus more WAL. On a write-heavy table (audit log, events), fewer indexes is a *feature*. This is why you index for actual read patterns, not "just in case."

### Partition Pruning Impact

| Query | Flat 200M-row table | Range-partitioned by month |
|-------|---------------------|----------------------------|
| Last 7 days | scans full index | scans 1 partition |
| Delete >90 days | 200M-row DELETE, hours | `DROP TABLE`, ms |
| Point lookup by recent time | full index height | smaller per-partition index |

### Optimization Techniques Specific to Design

- **Covering indexes (`INCLUDE`)** turn heap fetches into index-only scans — 2–5× on wide rows.
- **Partial indexes** (`WHERE read = false`) shrink the index to the hot subset, keeping it cache-resident and cheaper to maintain.
- **Keyset pagination** keeps deep pages O(page size) instead of O(offset).
- **BRIN indexes** on naturally-ordered append columns (`created_at` on an audit log) are tiny and prune huge ranges — a fraction of a B-tree's size for time-range scans.
- **Pre-aggregated rollup tables / materialised views** turn per-request aggregation into a single row read.

---

## 11. Node.js Integration

Design answers become code. Here's how the patterns look with `pg`.

### 11.1 The denormalised counter, kept consistent in a transaction

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Insert a notification AND bump the counter atomically.
async function createNotification(userId, type, title, body) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query(
      `INSERT INTO notifications (user_id, type, title, body)
       VALUES ($1, $2, $3, $4)`,
      [userId, type, title, body]
    );
    await client.query(
      `UPDATE users SET unread_count = unread_count + 1 WHERE id = $1`,
      [userId]
    );
    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

// The badge read — O(1), single-row PK lookup.
async function getUnreadCount(userId) {
  const { rows } = await pool.query(
    `SELECT unread_count FROM users WHERE id = $1`,
    [userId]
  );
  return rows[0]?.unread_count ?? 0;
}
```

### 11.2 Keyset pagination endpoint

```javascript
// cursor = { createdAt, id } from the last row of the previous page (or null)
async function getFeedPage(userId, cursor, pageSize = 20) {
  if (!cursor) {
    const { rows } = await pool.query(
      `SELECT id, author_name, body, created_at
       FROM feed_items
       WHERE user_id = $1
       ORDER BY created_at DESC, id DESC
       LIMIT $2`,
      [userId, pageSize]
    );
    return rows;
  }
  const { rows } = await pool.query(
    `SELECT id, author_name, body, created_at
     FROM feed_items
     WHERE user_id = $1
       AND (created_at, id) < ($2, $3)
     ORDER BY created_at DESC, id DESC
     LIMIT $4`,
    [userId, cursor.createdAt, cursor.id, pageSize]
  );
  return rows;
}
```

### 11.3 Idempotent write via a unique key

```javascript
// The UNIQUE(idempotency_key) constraint makes "did this already run?" a DB guarantee.
async function recordPayment(orderId, amount, idempotencyKey) {
  try {
    const { rows } = await pool.query(
      `INSERT INTO payments (order_id, amount, idempotency_key)
       VALUES ($1, $2, $3)
       RETURNING id`,
      [orderId, amount, idempotencyKey]
    );
    return { created: true, id: rows[0].id };
  } catch (err) {
    if (err.code === '23505') {         // unique_violation → already processed
      return { created: false, duplicate: true };
    }
    throw err;
  }
}
```

A note on ORMs: the *schema* (partitioning, partial indexes, `INCLUDE` columns, `CHECK` constraints) is almost always defined in raw SQL migrations even when the app uses an ORM — most ORMs can't express these. The *queries* (feed, badge) are simple enough for any ORM, but keyset pagination and counter-in-a-transaction are cleaner in raw SQL. Use the ORM for CRUD, drop to raw SQL for the design-critical paths.

---

## 12. ORM Comparison

The question for system-design isn't "can the ORM write a SELECT" — it's "can the ORM express the *schema* and the *design-critical queries* your access patterns require." Here's where each stands on the features that matter for design: partitioning, partial/covering indexes, keyset pagination, and denormalised counters.

### Prisma

**Can it?** Partially. Prisma models express tables, PKs, FKs, `@unique`, and basic indexes. It cannot express table partitioning, partial indexes (`WHERE`), `INCLUDE` covering columns, or `CHECK` constraints — those go in raw migration SQL.

```typescript
// Schema-level design lives in prisma/migrations as raw SQL, e.g.:
//   CREATE INDEX idx_notif_unread ON notifications (user_id, created_at DESC)
//     WHERE read = false;
// Prisma's schema.prisma only knows about a plain index.

// Keyset pagination — Prisma supports cursor-based pagination natively:
const page = await prisma.feedItem.findMany({
  where: { userId },
  orderBy: [{ createdAt: 'desc' }, { id: 'desc' }],
  take: 20,
  ...(cursor && { cursor: { id: cursor.id }, skip: 1 }),
});
// But Prisma's cursor uses a single-column cursor + skip:1; true
// (createdAt,id) tuple comparison needs $queryRaw for correctness under ties.

// Denormalised counter increment:
await prisma.user.update({
  where: { id: userId },
  data: { unreadCount: { increment: 1 } },
});
```

**Where it breaks:** partitioning, partial indexes, and tuple-keyset pagination all need `$executeRaw`/`$queryRaw`. **Verdict:** great for CRUD and the counter; use raw SQL for partitioning and correct keyset pagination.

### Drizzle

**Can it?** More than most. Drizzle expresses partial indexes and composite indexes in the schema builder, and its SQL-first design makes tuple comparisons natural.

```typescript
import { pgTable, bigserial, boolean, timestamp, index } from 'drizzle-orm/pg-core';
import { sql, and, eq, lt } from 'drizzle-orm';

export const notifications = pgTable('notifications', { /* ... */ }, (t) => ({
  unread: index('idx_notif_unread')
    .on(t.userId, t.createdAt.desc())
    .where(sql`read = false`),          // partial index expressible in-schema
}));

// True tuple keyset pagination:
const page = await db.select().from(feedItems)
  .where(and(eq(feedItems.userId, userId),
             sql`(${feedItems.createdAt}, ${feedItems.id}) < (${c.createdAt}, ${c.id})`))
  .orderBy(desc(feedItems.createdAt), desc(feedItems.id))
  .limit(20);
```

**Where it breaks:** table partitioning still needs raw SQL migrations. **Verdict:** best fit for design-heavy schemas — partial indexes and tuple keyset are first-class.

### Sequelize

**Can it?** Schema basics yes; partial/covering indexes and partitioning no. Keyset pagination is manual via `Op`.

```javascript
const page = await FeedItem.findAll({
  where: {
    userId,
    [Op.or]: [                      // tuple comparison must be unrolled manually
      { createdAt: { [Op.lt]: c.createdAt } },
      { createdAt: c.createdAt, id: { [Op.lt]: c.id } },
    ],
  },
  order: [['createdAt', 'DESC'], ['id', 'DESC']],
  limit: 20,
});
await User.increment('unreadCount', { by: 1, where: { id: userId } });
```

**Where it breaks:** no partial indexes, no partitioning, verbose keyset. **Verdict:** workable but drop to `sequelize.query()` for the design-critical paths.

### TypeORM

**Can it?** Indexes and constraints via decorators; partial indexes via the `where` index option; partitioning via raw migrations only.

```typescript
@Index('idx_notif_unread', ['userId', 'createdAt'], { where: '"read" = false' })
@Entity()
export class Notification { /* ... */ }

// Keyset via QueryBuilder:
const page = await repo.createQueryBuilder('f')
  .where('f.userId = :userId', { userId })
  .andWhere('(f.createdAt, f.id) < (:createdAt, :id)', cursor)
  .orderBy('f.createdAt', 'DESC').addOrderBy('f.id', 'DESC')
  .limit(20).getMany();
```

**Where it breaks:** partitioning and `INCLUDE` covering columns need raw SQL. **Verdict:** partial indexes and tuple keyset both work; partitioning is raw.

### Knex

**Can it?** It's a query builder, not an ORM — schema is explicit and it passes raw SQL through cleanly, so it expresses almost everything with `knex.raw`.

```javascript
await knex.schema.raw(`
  CREATE INDEX idx_notif_unread ON notifications (user_id, created_at DESC)
    WHERE read = false
`);

const page = await knex('feed_items')
  .where('user_id', userId)
  .andWhereRaw('(created_at, id) < (?, ?)', [c.createdAt, c.id])
  .orderBy([{ column: 'created_at', order: 'desc' }, { column: 'id', order: 'desc' }])
  .limit(20);

await knex('users').where({ id: userId }).increment('unread_count', 1);
```

**Where it breaks:** nothing structurally — but you're writing the SQL yourself. **Verdict:** most transparent; the design *is* the SQL.

### Summary Table

| ORM | Partial index | Partitioning | Covering (`INCLUDE`) | Tuple keyset | Counter increment | Verdict |
|-----|---------------|--------------|----------------------|--------------|-------------------|---------|
| Prisma | Raw migration | Raw migration | Raw migration | `$queryRaw` | Native | CRUD + counter native, raw for schema |
| Drizzle | In-schema | Raw migration | Raw migration | Native `sql` | Native | Best design fit |
| Sequelize | No | No | No | Manual `Op.or` | Native | Drop to raw for design paths |
| TypeORM | Index `where` | Raw migration | Raw migration | QueryBuilder | QueryBuilder | Partial + keyset ok |
| Knex | `knex.raw` | `knex.raw` | `knex.raw` | `whereRaw` | Native | Most transparent |

The through-line: **the schema-level design decisions (partitioning, partial/covering indexes) almost always live in raw SQL migrations regardless of ORM.** In an interview, say this — it signals you know where the ORM's abstraction ends.

---

## 13. Practice Exercises

### Exercise 1 — Easy

Design the schema for a `url_shortener`: given a short code, redirect to a long URL (very high read QPS, must be <1 ms); record each click. Specify tables, primary keys, the index that serves the redirect lookup, and one constraint that guarantees short-code uniqueness.

```sql
-- Write your schema here
```

### Exercise 2 — Medium (combines indexing + pagination from prior topics)

You have `orders(id bigint, customer_id bigint, status text, created_at timestamptz, total_amount numeric)` with 80M rows. The access pattern is an admin screen: "show this customer's orders, newest first, infinite scroll." Write (a) the composite index that serves it with no Sort node, and (b) the keyset-paginated query for the *next* page. Explain why `OFFSET` would be wrong here.

```sql
-- Write your index and query here
```

### Exercise 3 — Hard (production simulation; naive answer is wrong)

Design a "who liked my post" feature for a social app: 200M users, some posts get 5M+ likes, and you must show (a) the total like count on every post render, and (b) a paginated list of likers, and (c) whether *the current viewer* has liked the post. A junior will propose `SELECT COUNT(*) FROM likes WHERE post_id = $1` for (a) and a full scan for (c). Design the schema and access strategy so all three are fast at this scale. Name every denormalisation and its consistency cost.

```sql
-- Write your schema and the three queries here
```

### Exercise 4 — Interview Simulation

The interviewer says: *"Design an audit log for a fintech app. Every state change to any entity must be recorded immutably, kept for 7 years for compliance, and support two queries: (1) 'show me everything that happened to entity X in time order' and (2) 'show me everything user Y did last month.' It'll take ~5,000 writes/second at peak."* Walk through your discovery questions, the schema, the partitioning strategy, the indexes for both queries, and the retention mechanism. State every tradeoff.

```sql
-- Write your full design here
```

---

## 14. Interview Questions

### Q1 — "Design the schema for a notifications system."

**Junior answer:** "A `notifications` table with `id`, `user_id`, `message`, `read`, `created_at`. To show the unread badge, `SELECT COUNT(*) WHERE user_id = X AND read = false`."

**Principal answer:** "First, my access patterns: feed reads (last N unread, newest first) at high QPS, an unread badge on *every* page load, and writes when events fire. That badge-on-every-page-load is the load-bearing requirement. I'll keep a denormalised `unread_count` on `users` so the badge is an O(1) PK read instead of a `COUNT(*)` that grows without bound — I maintain it transactionally on write and reconcile nightly to correct drift. For the feed I add a *partial* index `(user_id, created_at DESC) WHERE read = false` so the query is a pre-sorted index range scan and `LIMIT 20` stops after 20 entries. At 500M rows with 6-month retention I range-partition by month so retention is `DROP TABLE`, not a mass `DELETE`, and recent-time queries prune to one partition. `id` is `bigint`. Tradeoffs: the counter trades a small write cost and eventual-correctness for a constant-time read; partitioning trades operational overhead (pre-creating partitions) for cheap retention and pruning."

**Follow-up the interviewer asks:** "The `unread_count` has drifted — a user sees 3 but has 0 unread. How did it happen and how do you prevent it?" *(A double-processed event, a non-transactional update, or a mark-read that didn't decrement. Prevent with the increment/decrement inside the same transaction as the insert/update, an idempotency key on the event, and a scheduled reconciliation job that recomputes the true count.)*

### Q2 — "This query got slow last week. Nothing about the query changed. Why?"

**Junior answer:** "Add an index? Maybe restart the database."

**Principal answer:** "A query that regressed without a code change is almost always a *plan flip*. Three usual causes: (1) **stale statistics** — the data shape changed (a bulk load, a value became common) and `pg_statistic` no longer reflects it, so the planner's row estimate is wrong and it picks a bad algorithm — fix with `ANALYZE`. (2) **data volume crossed a threshold** where a Seq Scan stopped fitting in cache, or a Sort started spilling to disk (`Sort Method: external merge`). (3) **index bloat** from heavy updates/deletes without adequate vacuuming, making the index less effective. I'd run `EXPLAIN (ANALYZE, BUFFERS)`, compare actual vs estimated rows to spot the stats problem, and check `Buffers` for a jump in disk reads. The fix follows the diagnosis — I don't add an index blindly."

**Follow-up:** "`EXPLAIN` shows the estimate is 12 rows but actual is 4 million. What now?" *(Stale stats or a correlation the planner can't model. Run `ANALYZE`; if the columns are correlated, consider extended statistics `CREATE STATISTICS`. If a specific value is skewed, increase the statistics target on that column.)*

### Q3 — "How would you paginate a feed with 100 million rows so page 5,000 is as fast as page 1?"

**Junior answer:** "`LIMIT 20 OFFSET (page * 20)`."

**Principal answer:** "`OFFSET` is the trap — `OFFSET 100000` makes the engine produce and discard 100K rows in order, so deep pages get linearly slower. I'd use **keyset (seek) pagination**: order by `(created_at DESC, id DESC)`, and each page carries the last row's `(created_at, id)` as a cursor; the next page is `WHERE (created_at, id) < ($cursor_created, $cursor_id)`. With a composite index on `(user_id, created_at DESC, id DESC)` every page is an index range scan of exactly 20 rows — O(page size), independent of depth. The `id` tiebreaker makes it stable when timestamps collide. The one thing keyset gives up is random 'jump to page N' access — but infinite-scroll feeds don't need that."

**Follow-up:** "What if the user can sort by different columns?" *(Each sort order needs its own composite index and its own cursor tuple; you can't keyset on a column you don't have an index for without a Sort. This is why arbitrary multi-column sort on huge tables is expensive — you pick a bounded set of supported sort orders.)*

### Q4 — "Design an audit log that handles 5,000 writes/second and 7-year retention."

**Junior answer:** "One big `audit_logs` table with indexes on everything I might query."

**Principal answer:** "Write-heavy and append-only, so I *minimise* indexes — every index multiplies write cost and WAL. I range-partition by month on `created_at`: retention becomes `DROP TABLE audit_logs_2019_07`, and time-range queries prune. Two access patterns: 'everything for entity X in time order' wants `(entity_type, entity_id, created_at)`; 'everything user Y did last month' wants `(actor_id, created_at)` — both benefit from partition pruning first. For the naturally time-ordered `created_at`, a BRIN index is tiny and prunes ranges cheaply. Rows are immutable so I never `UPDATE` — no MVCC bloat from updates. I'd store the changed payload as `jsonb`. Tradeoffs: partitioning adds the operational task of pre-creating partitions (automate it); keeping indexes minimal means some ad-hoc analytical queries will be slow — that's acceptable for an audit log, which is written constantly and read rarely."

**Follow-up:** "Compliance wants to prove no record was tampered with. How?" *(Append-only enforced by revoking UPDATE/DELETE; optionally a hash chain — each row stores a hash of its content plus the previous row's hash — so any tampering breaks the chain.)*

---

## 15. Mental Model Checkpoint

1. An interviewer says "design the schema for X" and you have 20 minutes. What do you do in the first three minutes, and why is skipping it the most common way strong engineers fail this question?

2. You have `SELECT COUNT(*) FROM notifications WHERE user_id = $1 AND read = false` on every page load. Even with a perfect partial index-only scan it's 40 ms at scale. Why can no index fix this, and what has to change instead?

3. Explain, in terms of what the executor actually does, why `OFFSET 100000 LIMIT 20` is slow but keyset pagination returning the same 20 rows is fast.

4. You're choosing between a sequential `bigint` and a random `uuid v4` as the primary key of a table doing 20K inserts/second. What happens in the B-tree in each case, and which do you pick? When would `uuid v7` change your answer?

5. A table has a 90-day retention policy. Compare, in terms of MVCC and locking, enforcing it with `DELETE ... WHERE created_at < ...` on a flat table versus `DROP TABLE` on a monthly partition.

6. You add a denormalised `unread_count` column. Name two distinct ways it can drift from the true count, and the mechanism that keeps it eventually correct.

7. A query that ran fine for a year suddenly got 100× slower with no code change. Give three engine-level explanations and the one command you'd run first to distinguish them.

---

## 16. Quick Reference Card

```sql
-- ── THE THREE QUESTION TYPES ──────────────────────────────────
-- TYPE A "design schema"  → entities, keys, indexes, constraints, TRADEOFFS
-- TYPE B "query at scale"  → access patterns → index/denorm/cache/partition
-- TYPE C "diagnose slow"   → EXPLAIN(ANALYZE,BUFFERS) → villain → fix → cost

-- ── DISCOVERY BEFORE DESIGN (ask, then state assumptions) ─────
--   scale?  read/write ratio?  latency budget?  retention?
--   exact read queries?  consistency needs?  concurrency/QPS?

-- ── PRIMARY KEYS ──────────────────────────────────────────────
id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY  -- NOT serial/int (overflows)
-- random uuid v4 as PK  → insert scatter, page splits (avoid on write-heavy)
-- uuid v7 / ULID        → time-ordered, preserves insert locality

-- ── INDEX ORDER: equality, then range, then sort ─────────────
-- WHERE a = $1 AND b = $2 ORDER BY c DESC  →  INDEX (a, b, c DESC)
CREATE INDEX ix ON t (user_id, created_at DESC) WHERE read = false;  -- partial
CREATE INDEX ix ON t (user_id, created_at DESC) INCLUDE (title);     -- covering

-- ── DENORMALISE DELIBERATELY (name the consistency cost) ──────
--   counter column  → O(1) badge vs COUNT(*) that grows unbounded
--   cached column   → skip a hot-path join (staleness on parent change)
--   rollup table    → precompute aggregation instead of live GROUP BY

-- ── PAGINATION: keyset, never deep OFFSET ─────────────────────
WHERE (created_at, id) < ($last_created, $last_id)
ORDER BY created_at DESC, id DESC LIMIT 20;   -- O(20) at any depth

-- ── PARTITION time-series / append-heavy tables ──────────────
PARTITION BY RANGE (created_at);   -- pruning + DROP-based retention (no DELETE)

-- ── DIAGNOSE (Type C) villains in EXPLAIN ────────────────────
--   Seq Scan + high "Rows Removed by Filter"  → missing index
--   Sort Method: external merge Disk          → spilling / paginate wrong
--   estimate 12 vs actual 4,000,000           → stale stats → ANALYZE
--   Nested Loop, high loops, Seq Scan inner   → missing index on inner key
--   Hash Join Batches > 1                      → work_mem too small
```

**Interview one-liners:**
- "Let me ask about the access patterns before I draw tables."
- "Schema is the hardest thing to change later; I design for target scale, not launch scale."
- "`COUNT(*)` on the read path is a time bomb — I denormalise the counter and reconcile it."
- "`OFFSET` scans and discards; keyset seeks. I paginate by cursor."
- "Every index has a write cost — I index for the actual read patterns, not just in case."
- "I normalise for correctness and denormalise for a specific proven read pattern, and I name the staleness I accept."
- "Partition time-series tables so retention is `DROP TABLE`, not a `DELETE` that bloats."
- "A query that got slow with no code change is a plan flip — I check statistics first."

---

## Connected Topics

**Internals this builds on:**
- **Topic 06 — MVCC and Visibility**: why `UPDATE`-heavy counters and mass `DELETE`s cause bloat; why append-only audit logs stay clean.
- **Topic 17 — JOIN Performance Deep Dive**: the Nested Loop / Hash / Merge choices you diagnose in Type C questions.
- **Topic 30 — B-tree Indexes**: the structure behind composite order, covering, and partial indexes.
- **Topic 52 — Partitioning**: range/list partitioning and pruning, the backbone of scaling large tables.

**Prior SQL topics:**
- **Topic 11 — INNER JOIN in Depth**: fan-out and `COUNT(DISTINCT)` reasoning that underlies aggregation-vs-rollup design decisions.
- **Topic 27 — EXISTS and NOT EXISTS**: the semi-join for existence checks ("has the viewer liked this?") instead of counting.
- **Topic 75 — Live Coding Patterns**: the tactical query-writing skills you apply once the design is set.

**Next SQL topic:**
- **Topic 77 — Canonical Hard SQL Problems**: the hardest standalone query problems — gaps-and-islands, running totals, top-N-per-group — that show up inside the systems you design here.
