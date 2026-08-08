# 13 — Types of Indexes
## Phase: Indexes

---

## ELI5 — The Simple Analogy

One library, several kinds of index card drawer.

**The main catalogue** lists every book by title. Complete, general-purpose, big. That's an **ordinary index**.

**The "currently on loan" drawer** holds cards only for the ~200 books that are out, not all 50,000. It's tiny, and it answers the only question anyone asks about loans. That's a **partial index** — and it's the most under-used tool in the entire toolbox.

**The "by author's surname" drawer** is sorted by a value that isn't printed on the card as a single field — the librarian computed it from "Rabindranath Tagore." That's an **expression index**.

**The "ISBN" drawer** has a rule: *no two cards may share an ISBN*. Filing a duplicate is refused at the drawer. That's a **unique index** — it isn't just fast lookup, it's an enforced law.

**And one drawer has the book's shelf location written on the card itself**, so for "where is it?" you never walk to the shelf. That's a **covering index**.

Same underlying card system. Five completely different tools, and choosing right often changes a query by 100× — or removes an entire class of bug.

---

## Where this fits in the big picture

```
   10 what an index is
   11 B-tree structure
   12 lookup end to end (Index Cond vs Filter, INCLUDE)
              │
              ▼
   ┌──────────────────────────────┐
   │ 13 TYPES OF INDEXES          │ ← YOU ARE HERE
   │ the B-tree variants          │
   └──────────────┬───────────────┘
                  │
     ┌────────────┼─────────────┬───────────────┐
     ▼            ▼             ▼               ▼
 14 composite  15 selectivity  16 non-btree   24 constraints
 column order  (which to use)  (GIN/GiST/BRIN) (UNIQUE as a law)
```

Everything here is still a **B-tree** underneath (Topic 11). These are *modifiers* on how it's built and what it stores. Topic 16 covers genuinely different structures.

---

## What is this?

Five orthogonal ways to modify a B-tree index:

| Variant | Syntax | What changes |
|---|---|---|
| **Ordinary** | `ON t (col)` | baseline |
| **Multi-column** | `ON t (a, b, c)` | key is a tuple; order matters enormously (Topic 14) |
| **Unique** | `CREATE UNIQUE INDEX` | also **enforces a constraint** |
| **Partial** | `ON t (col) WHERE pred` | only rows matching `pred` are indexed |
| **Expression** | `ON t (f(col))` | indexes a computed value |
| **Covering** | `ON t (a) INCLUDE (b, c)` | extra payload in leaves for index-only scans |

They **compose**. A single index can be unique, partial, on an expression, and covering all at once — and that combination solves problems nothing else can.

---

## Why does it matter for a backend developer?

Because the default (ordinary index on a column) is the *wrong* choice more often than people realise, and the alternatives are dramatically cheaper.

Real numbers from the examples below:

```
 A job queue: 200M rows, ~3,000 of them 'pending'.
   ordinary index on (status)                → 4.2 GB, planner won't use it
   partial index WHERE status='pending'      → 112 KB, 0.08 ms lookups
                                                ─────────────────────────
                                                37,500× smaller

 Case-insensitive email lookup:
   index on (email) + WHERE lower(email)=$1  → Seq Scan, 840 ms
   expression index on (lower(email))        → Index Scan, 0.09 ms

 "One active subscription per user":
   application check                          → race condition, real bug
   partial unique index WHERE state='active'  → structurally impossible
```

And the last one matters beyond performance: **a unique index is the only race-free way to enforce uniqueness** (Topic 24, and case studies 01–03 all depend on it).

---

## The physical reality

### Partial index — what's actually missing from the file

```
 TABLE jobs: 200,000,000 rows
   status: 'done' (199,996,800) · 'pending' (2,800) · 'failed' (400)

 ORDINARY INDEX ON (status)
 ┌──────────────────────────────────────────────────────────────┐
 │ LEAF pages: 'done'→TID  'done'→TID  'done'→TID ... ×200M     │
 │ with PG13+ deduplication: ~6 bytes/row                        │
 │ SIZE: ~4.2 GB · tree_level 3 · 512,000 leaf pages            │
 │ USED BY THE PLANNER: never — 'done' is 99.998% of the table  │
 └──────────────────────────────────────────────────────────────┘

 PARTIAL INDEX ON (created_at) WHERE status = 'pending'
 ┌──────────────────────────────────────────────────────────────┐
 │ LEAF pages: 2,800 entries total                              │
 │ SIZE: 112 KB · tree_level 0 (the root IS the leaf)           │
 │ ★ PERMANENTLY CACHED — it's smaller than one buffer batch     │
 └──────────────────────────────────────────────────────────────┘

 AND THE WRITE PATH — this is the bigger win:
   INSERT with status='done'  → ordinary index: 1 B-tree insert
                              → partial index:  ZERO WORK.
                                The predicate is evaluated, fails,
                                and the index is never touched.
   UPDATE status 'pending'→'done'
                              → partial index: DELETE the entry.
                                The index SHRINKS as jobs complete.
```

**The index's size is bounded by the number of `pending` rows, not by table size.** A 200M-row table with a 112 KB index that never grows. That is the single most valuable property of partial indexes.

### Expression index — what's stored

```
 CREATE INDEX idx_users_lower_email ON users (lower(email));

 LEAF ENTRY:
 ┌────────────────────────────────────────────────────────┐
 │ IndexTupleData (8 B) │ 'arjun@shop.in' (computed value)│
 │   → TID (1204, 3)                                       │
 └────────────────────────────────────────────────────────┘
        ↑ the LOWERCASED string is stored, not the original

 ⚠ CONSEQUENCES:
   1. The function runs on every INSERT and every UPDATE of that column.
      It must be IMMUTABLE — same input, same output, forever. PostgreSQL
      refuses non-immutable functions:
        ERROR: functions in index expression must be marked IMMUTABLE
      (This is why `now()`, `current_date`, and unqualified `::timestamp`
       casts are rejected — the index would become silently wrong.)
   2. The planner matches the expression SYNTACTICALLY (after normalisation).
      Index on `lower(email)` serves `WHERE lower(email) = $1`.
      It does NOT serve `WHERE upper(email) = $1` or `WHERE email = $1`.
   3. It also gives the planner STATISTICS on the expression — often as
      valuable as the index itself for row estimation. (Topic 15.)
```

### Unique index — the enforcement mechanism

```
 CREATE UNIQUE INDEX uq_users_email ON users (email);

 ON INSERT, inside the B-tree insert itself:
   1. descend to the target leaf
   2. take an exclusive buffer lock on that page
   3. scan for an existing entry with the same key
   4. found one? Check whether its heap tuple is VISIBLE or from an
      IN-PROGRESS transaction:
        • visible & live       → ERROR: duplicate key value
        • in-progress          → WAIT on that transaction, then re-check
        • dead (deleted/aborted) → proceed
   5. insert
   6. release the lock

 ★ THERE IS NO WINDOW. The check and the insert happen under the same
   page lock. Compare:

     SELECT 1 FROM users WHERE email=$1;   ← txn A: none found
                                             txn B: none found
     INSERT INTO users ...                 ← txn A: inserts
                                             txn B: inserts. TWO ROWS.

   The application check has a gap between the two statements. The unique
   index does not. This is why "we validate in the app" is not equivalent.
```

### Partial unique — the combination that solves real problems

```
 CREATE UNIQUE INDEX uq_one_active_sub
   ON subscriptions (user_id) WHERE state = 'active';

 ┌─────────────────────────────────────────────────────────────┐
 │ user_id │ state      │ in the index?                        │
 ├─────────────────────────────────────────────────────────────┤
 │    7    │ active     │ YES  ← occupies the slot for user 7   │
 │    7    │ cancelled  │ no                                    │
 │    7    │ expired    │ no                                    │
 │    7    │ cancelled  │ no                                    │
 │    7    │ active     │ ✗ REJECTED — duplicate key            │
 └─────────────────────────────────────────────────────────────┘

 ⇒ "At most one ACTIVE subscription per user, unlimited historical ones."
   A plain UNIQUE(user_id) would forbid history. A CHECK constraint
   cannot see other rows. A trigger races. Only this works.
```

---

## How it works — step by step

### When the planner will and won't use a partial index

```
 INDEX: ON jobs (created_at) WHERE status = 'pending' AND attempts < 5

 The planner uses it ONLY IF it can PROVE the query's WHERE clause
 IMPLIES the index predicate. It does this with a theorem prover over
 the clauses — not string matching.

 QUERY                                            USED?  WHY
 ────────────────────────────────────────────────────────────────────
 WHERE status='pending' AND attempts<5            ✓  exact match
 WHERE status='pending' AND attempts=0            ✓  0<5 is implied
 WHERE status='pending' AND attempts<3            ✓  <3 implies <5
 WHERE status='pending' AND attempts<5 AND x=1    ✓  extra clauses fine
 WHERE status='pending'                           ✗  attempts not bounded
 WHERE status='pending' AND attempts<10           ✗  <10 does NOT imply <5
 WHERE status IN ('pending','failed')             ✗  can't prove
 WHERE status=$1                                  ✗  ⚠⚠ PARAMETER!
                                                     Cannot prove at plan
                                                     time with a generic
                                                     plan.

 ★★★ THE PARAMETER TRAP ★★★
 This catches everyone. `WHERE status = $1` with $1='pending' at runtime
 will NOT use the partial index if the planner built a generic plan
 (Topic 09/18). The predicate must be a CONSTANT in the query text.

 FIX in Node.js — inline the constant, parameterise the rest:
   ✗ 'SELECT ... WHERE status = $1 AND created_at < $2'
   ✓ "SELECT ... WHERE status = 'pending' AND created_at < $1"
 The literal is not user input, so there is no injection risk.
```

### Insert path, per index type

```
 INSERT INTO jobs (status, created_at, payload) VALUES ('done', now(), '...');

 ORDINARY (status)         → descend 3 levels, insert entry.  [3 reads, 1 write]
 PARTIAL WHERE status='pending'
                           → evaluate predicate: 'done' ≠ 'pending'
                           → SKIP ENTIRELY.                   [0 reads, 0 writes]
 EXPRESSION (lower(email)) → call lower(), then descend+insert [3 reads, 1 write
                                                                + function call]
 UNIQUE (email)            → descend, LOCK page, scan for dup,
                             possibly WAIT on another txn,
                             insert.                          [3 reads, 1 write,
                                                                + lock hold]
 COVERING (a) INCLUDE (b,c)→ descend, insert a WIDER entry     [3 reads, 1 write,
                                                                bigger WAL]
```

### The `UNIQUE INDEX` vs `UNIQUE CONSTRAINT` distinction

```
 CREATE UNIQUE INDEX uq_a ON t (email);          -- an INDEX
 ALTER TABLE t ADD CONSTRAINT uq_b UNIQUE(email);-- a CONSTRAINT
                                                    (creates an index too)

 SAME enforcement. DIFFERENT capabilities:

                                  INDEX    CONSTRAINT
 partial (WHERE ...)                ✓          ✗
 on an expression                   ✓          ✗
 INCLUDE columns                    ✓        ✓ (PG11+)
 NULLS NOT DISTINCT                 ✓          ✓
 referenced by a FOREIGN KEY        ✗          ✓   ★
 shows in \d as a constraint        ✗          ✓
 CREATE/DROP CONCURRENTLY           ✓          ✗   ★

 ⇒ Use a CONSTRAINT when a foreign key must reference it.
 ⇒ Use an INDEX when you need partial/expression, or when you must
   build it on a live table without blocking writes.
```

### NULLs — the rule that surprises everyone

```
 CREATE UNIQUE INDEX uq_email ON users (email);
 INSERT INTO users (email) VALUES (NULL);   -- ok
 INSERT INTO users (email) VALUES (NULL);   -- ok!
 INSERT INTO users (email) VALUES (NULL);   -- ok!!

 In SQL, NULL ≠ NULL, so unique indexes permit unlimited NULLs.

 PostgreSQL 15+ fixes this when you want it:
   CREATE UNIQUE INDEX uq_email ON users (email) NULLS NOT DISTINCT;
   → now only ONE NULL is allowed.

 Before PG15, the workaround:
   CREATE UNIQUE INDEX ON users (coalesce(email, '<<null>>'));
   -- an expression index doing the job

 ⚠ AND: a partial index EXCLUDES NULL rows if the predicate can't be
   satisfied by NULL:
     ON t (a) WHERE b IS NOT NULL   → NULL-b rows aren't indexed at all
```

---

## Concept breakdown

```
ORDINARY INDEX
└── every row gets an entry. The baseline. Often the wrong default.

PARTIAL INDEX — ON t (col) WHERE predicate
│
├── Only rows satisfying `predicate` are indexed
├── SIZE bounded by matching rows, not table size
├── WRITE COST zero for non-matching rows        ★ the underrated win
├── Planner must PROVE query-predicate ⟹ index-predicate
├── ⚠ Constants only — `WHERE status = $1` defeats it
└── USE FOR: status queues, soft deletes, "active" subsets, hot recent data

EXPRESSION INDEX — ON t (f(col))
│
├── Stores the computed value
├── f() MUST be IMMUTABLE (not `now()`, not unqualified timezone casts)
├── Planner matches the expression syntactically
├── Also provides STATISTICS on the expression                ★ often
│   underrated: fixes bad row estimates even when the index isn't used
└── USE FOR: lower()/upper(), (jsonb->>'key'), date_trunc(),
             md5() for long text, computed derived values

UNIQUE INDEX — CREATE UNIQUE INDEX
│
├── Enforcement happens INSIDE the B-tree insert, under a page lock
├── ★ NO RACE WINDOW. This is the only correct way to enforce uniqueness
├── NULLs are all distinct by default (NULLS NOT DISTINCT in PG15+)
├── Can be PARTIAL — "unique among active rows"                ★★
└── Also the mechanism behind PRIMARY KEY and ON CONFLICT

COVERING INDEX — ON t (a) INCLUDE (b, c)
│
├── b, c stored ONLY in leaves; not searchable, not sortable
├── Keeps internal nodes narrow → higher fanout → shallower tree
├── Enables INDEX ONLY SCAN (Topic 12)
├── ⚠ depends on VACUUM keeping the visibility map current
└── USE FOR: hot queries needing a few extra output columns

MULTI-COLUMN — ON t (a, b, c)
└── Full treatment in Topic 14. Column ORDER is the whole story.

★ THEY COMPOSE. The most powerful indexes in real systems are
  combinations:
    CREATE UNIQUE INDEX uq_active_email
      ON users (lower(email)) WHERE deleted_at IS NULL;
    -- unique + expression + partial: "case-insensitively unique email
    --  among non-deleted users, with deleted users free to reuse it"
```

---

## Diagrams

**Diagram 1 — big picture: the same table, five indexes**

```
                        TABLE jobs (200M rows)
                                │
   ┌──────────┬─────────────┬───┴────────┬──────────────┬────────────┐
   ▼          ▼             ▼            ▼              ▼            ▼
 ORDINARY   PARTIAL     EXPRESSION    UNIQUE        COVERING     COMPOSITE
 (status)   (created_at (lower(       (idem_key)    (user_id)    (user_id,
            ) WHERE      email))                    INCLUDE       created_at)
             status=                                (total,
             'pending'                               status)
   │          │             │            │              │            │
 4.2 GB    112 KB        280 MB       3.1 GB        1.8 GB       2.4 GB
 unused    ★ hot         case-        enforces      no heap      filter+
           path          insensitive  a LAW         fetch        sort
```

**Diagram 2 — data flow: what a partial index skips**

```
  INSERT status='done'                    INSERT status='pending'
  ─────────────────────────               ────────────────────────
   heap write                              heap write
      │                                       │
      ├─▶ ordinary idx: descend+insert        ├─▶ ordinary idx: descend+insert
      │                                       │
      └─▶ partial idx:                        └─▶ partial idx:
           evaluate 'done'='pending'?              evaluate 'pending'='pending'?
           FALSE → ★ DO NOTHING                    TRUE → descend+insert
           0 page reads, 0 writes, 0 WAL           (into a 112 KB index that
                                                    is entirely in RAM)

  ⇒ 199,996,800 of 200,000,000 inserts pay ZERO index cost.
```

**Diagram 3 — before/after: the queue index**

```
 BEFORE — ordinary index on (status, created_at)
 ┌──────────────────────────────────────────────────────────────────┐
 │ 200,000,000 entries · 6.8 GB · tree_level 3                      │
 │ every insert: +1 B-tree entry, +80 B WAL                         │
 │ every status update: delete + insert = 2 entries                 │
 │ the claim query: Bitmap scan over 200M-entry index → 340 ms      │
 │ buffer pool consumed: 6.8 GB competing with the table            │
 └──────────────────────────────────────────────────────────────────┘

 AFTER — partial index on (created_at) WHERE status='pending'
 ┌──────────────────────────────────────────────────────────────────┐
 │ 2,800 entries · 112 KB · tree_level 0                            │
 │ 99.998% of inserts: ZERO index work                              │
 │ status 'pending'→'done': one DELETE, index SHRINKS                │
 │ the claim query: single-page index scan → 0.08 ms                │
 │ buffer pool consumed: 112 KB, permanently resident                │
 └──────────────────────────────────────────────────────────────────┘
              4,250× smaller · 4,250× faster · ~zero write cost
```

---

## Example 1 — basic

```sql
CREATE TABLE jobs (
  id          bigserial PRIMARY KEY,
  status      text        NOT NULL DEFAULT 'pending',
  attempts    int         NOT NULL DEFAULT 0,
  queue       text        NOT NULL DEFAULT 'default',
  payload     jsonb       NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  completed_at timestamptz NULL
);

-- realistic distribution: almost everything is done
INSERT INTO jobs (status, attempts, payload, created_at, completed_at)
SELECT CASE WHEN i % 70000 = 0 THEN 'pending'
            WHEN i % 500000 = 0 THEN 'failed' ELSE 'done' END,
       (random()*2)::int,
       jsonb_build_object('order_id', i),
       now() - (random()*90)::int * interval '1 day',
       CASE WHEN i % 70000 = 0 THEN NULL ELSE now() END
FROM generate_series(1, 20000000) i;
VACUUM ANALYZE jobs;

SELECT status, count(*) FROM jobs GROUP BY status;
```
```
 status  |  count
---------+----------
 done    | 19999714
 pending |      286
 failed  |       40
```

**Step 1 — the ordinary index.**

```sql
CREATE INDEX idx_jobs_status_created ON jobs (status, created_at);
VACUUM ANALYZE jobs;
SELECT pg_size_pretty(pg_relation_size('idx_jobs_status_created'));
```
```
 pg_size_pretty
----------------
 601 MB
```
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, payload FROM jobs WHERE status='pending' ORDER BY created_at LIMIT 10;
```
```
Limit  (actual time=0.118..0.142 rows=10 loops=1)
  ->  Index Scan using idx_jobs_status_created on jobs
        Index Cond: (status = 'pending'::text)
        Buffers: shared hit=14
Execution Time: 0.164 ms
```
Fast! But look at the cost you're carrying: **601 MB of index, 99.998% of which is `done` entries that will never be queried**, plus a B-tree insert on every one of 20 million rows.

**Step 2 — the partial index.**

```sql
CREATE INDEX idx_jobs_pending ON jobs (created_at) WHERE status = 'pending';
SELECT pg_size_pretty(pg_relation_size('idx_jobs_pending'));
```
```
 pg_size_pretty
----------------
 16 kB              ← 601 MB → 16 KB. 38,000× smaller.
```
```sql
DROP INDEX idx_jobs_status_created;
VACUUM ANALYZE jobs;
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, payload FROM jobs WHERE status='pending' ORDER BY created_at LIMIT 10;
```
```
Limit  (actual time=0.031..0.048 rows=10 loops=1)
  ->  Index Scan using idx_jobs_pending on jobs
        Buffers: shared hit=12
Execution Time: 0.062 ms
```
**Same speed, 38,000× less disk and RAM.**

**Step 3 — measure the write saving.**

```sql
\timing on
CREATE INDEX idx_full ON jobs (status, created_at);
INSERT INTO jobs (status, payload) SELECT 'done', '{}'::jsonb FROM generate_series(1,500000);
-- Time: 4812.204 ms
DROP INDEX idx_full;
INSERT INTO jobs (status, payload) SELECT 'done', '{}'::jsonb FROM generate_series(1,500000);
-- Time: 3104.882 ms     ← 35% faster with only the partial index
```

And the WAL:
```sql
SELECT pg_current_wal_lsn() AS b \gset
INSERT INTO jobs (status, payload) SELECT 'done','{}'::jsonb FROM generate_series(1,200000);
SELECT pg_size_pretty(pg_current_wal_lsn() - :'b'::pg_lsn);
-- 34 MB with the partial index only; 51 MB with the full index too
```

**Step 4 — the parameter trap, demonstrated.**

```sql
EXPLAIN SELECT id FROM jobs WHERE status = 'pending' ORDER BY created_at LIMIT 10;
-- Index Scan using idx_jobs_pending   ✓

PREPARE p(text) AS SELECT id FROM jobs WHERE status = $1 ORDER BY created_at LIMIT 10;
EXPLAIN EXECUTE p('pending');   -- run 6+ times to force a generic plan
```
```
Limit
  ->  Sort
        Sort Key: created_at
        ->  Seq Scan on jobs
              Filter: (status = $1)          ← ⚠ 20 MILLION ROWS SCANNED
```
**The partial index is invisible to a generic plan.** Fix it in your Node.js code:

```js
// ✗ defeats the partial index
await pool.query('SELECT id FROM jobs WHERE status = $1 ORDER BY created_at LIMIT 10',
                 ['pending']);
// ✓ constant in the SQL, parameterise everything else
await pool.query(
  "SELECT id FROM jobs WHERE status = 'pending' AND queue = $1 ORDER BY created_at LIMIT 10",
  [queueName]);
```

**Step 5 — expression index.**

```sql
CREATE TABLE users (id bigserial PRIMARY KEY, email text NOT NULL, name text);
INSERT INTO users (email, name)
SELECT 'User'||i||'@Shop.IN', 'User '||i FROM generate_series(1,2000000) i;
CREATE INDEX idx_users_email ON users (email);
VACUUM ANALYZE users;

EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE lower(email) = 'user4471@shop.in';
```
```
Seq Scan on users  (actual time=0.4..812.2 rows=1 loops=1)
  Filter: (lower(email) = 'user4471@shop.in'::text)
  Rows Removed by Filter: 1999999
  Buffers: shared hit=1204 read=17888
Execution Time: 813.1 ms
```
```sql
CREATE INDEX idx_users_lower_email ON users (lower(email));
VACUUM ANALYZE users;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE lower(email) = 'user4471@shop.in';
```
```
Index Scan using idx_users_lower_email on users  (actual time=0.048..0.052 rows=1)
  Index Cond: (lower(email) = 'user4471@shop.in'::text)
  Buffers: shared hit=4
Execution Time: 0.071 ms
```
**19,092 buffers → 4. 813 ms → 0.07 ms.**

**Step 6 — the immutability rule.**

```sql
CREATE INDEX bad ON jobs (created_at::date);
-- ERROR:  functions in index expression must be marked IMMUTABLE
--   (timestamptz→date depends on TimeZone, which is a session setting)

CREATE INDEX good ON jobs ((created_at AT TIME ZONE 'UTC')::date);   -- ✓ fixed zone
CREATE INDEX also_good ON jobs (date_trunc('day', created_at));      -- ✓ immutable
```

**Step 7 — partial unique: the real-world use.**

```sql
CREATE TABLE subscriptions (
  id bigserial PRIMARY KEY,
  user_id bigint NOT NULL,
  plan text NOT NULL,
  state text NOT NULL CHECK (state IN ('active','cancelled','expired')),
  started_at timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX uq_one_active_sub ON subscriptions (user_id) WHERE state = 'active';

INSERT INTO subscriptions (user_id, plan, state) VALUES (7,'pro','cancelled');   -- ok
INSERT INTO subscriptions (user_id, plan, state) VALUES (7,'pro','expired');     -- ok
INSERT INTO subscriptions (user_id, plan, state) VALUES (7,'pro','active');      -- ok
INSERT INTO subscriptions (user_id, plan, state) VALUES (7,'team','active');
```
```
ERROR:  duplicate key value violates unique constraint "uq_one_active_sub"
DETAIL:  Key (user_id)=(7) already exists.
```
**History preserved, invariant enforced, zero race window.**

**Step 8 — soft deletes done correctly.**

```sql
ALTER TABLE users ADD COLUMN deleted_at timestamptz NULL;

-- ✗ WRONG: a deleted user's email is permanently reserved
CREATE UNIQUE INDEX uq_email_wrong ON users (lower(email));

-- ✓ RIGHT: unique among live users; deleted users free the address
DROP INDEX uq_email_wrong;
CREATE UNIQUE INDEX uq_email_live ON users (lower(email)) WHERE deleted_at IS NULL;

-- ✓ AND every query on live users gets a smaller index for free:
CREATE INDEX idx_users_live_created ON users (created_at) WHERE deleted_at IS NULL;
```

---

## Example 2 — production scenario

**The situation.** An order-processing service. `orders` is 340M rows, 180 GB. Fourteen indexes, 92 GB total. Two problems:

1. The worker query — "next 50 orders needing fulfilment" — takes 4.1 s and runs every 2 seconds from 30 workers.
2. Duplicate orders appear ~40 times/day under mobile retry.

```sql
-- the worker query
SELECT id, user_id, payload FROM orders
WHERE status = 'awaiting_fulfilment' AND warehouse_id = $1 AND attempts < 5
ORDER BY priority DESC, created_at ASC LIMIT 50 FOR UPDATE SKIP LOCKED;
```

**Step 1 — the distribution.**

```sql
SELECT status, count(*), round(100.0*count(*)/sum(count(*)) OVER (), 4) AS pct
FROM orders GROUP BY status ORDER BY 2 DESC;
```
```
       status        |   count   |   pct
---------------------+-----------+---------
 delivered           | 331204882 | 97.4132
 cancelled           |   7102884 |  2.0891
 shipped             |   1204009 |  0.3541
 awaiting_fulfilment |     41208 |  0.0121   ← the ONLY rows the worker wants
 payment_pending     |      8814 |  0.0026
```

**41,208 rows out of 340 million. 0.012%.**

**Step 2 — what the current index costs.**

```sql
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS size, idx_scan
FROM pg_stat_user_indexes WHERE relname='orders' ORDER BY pg_relation_size(indexrelid) DESC;
```
```
            indexrelname             |  size   |  idx_scan
-------------------------------------+---------+------------
 idx_orders_status_wh_created        | 18 GB   |   41209884
 idx_orders_status                   |  8 GB   |          0
 idx_orders_warehouse                | 12 GB   |     220481
 ...
```

The worker's index is **18 GB** — and 99.988% of it is `delivered` entries.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, user_id, payload FROM orders
WHERE status='awaiting_fulfilment' AND warehouse_id=12 AND attempts < 5
ORDER BY priority DESC, created_at ASC LIMIT 50;
```
```
Limit  (actual time=4088.2..4088.4 rows=50 loops=1)
  ->  Sort  (actual time=4088.2..4088.3 rows=50 loops=1)
        Sort Key: priority DESC, created_at
        Sort Method: top-N heapsort  Memory: 41kB
        ->  Bitmap Heap Scan on orders  (actual time=812.1..4044.9 rows=3982)
              Recheck Cond: ((status='awaiting_fulfilment') AND (warehouse_id=12))
              Filter: (attempts < 5)
              Rows Removed by Filter: 118
              Heap Blocks: exact=3901
              Buffers: shared hit=2104 read=88214       ← 706 MB
              ->  Bitmap Index Scan on idx_orders_status_wh_created
Execution Time: 4089.1 ms
```

**Step 3 — the partial index.**

```sql
CREATE INDEX CONCURRENTLY idx_orders_fulfil
  ON orders (warehouse_id, priority DESC, created_at ASC)
  INCLUDE (id, user_id)
  WHERE status = 'awaiting_fulfilment' AND attempts < 5;

SELECT pg_size_pretty(pg_relation_size('idx_orders_fulfil'));
```
```
 pg_size_pretty
----------------
 3448 kB           ← 18 GB → 3.4 MB
```

Note the design: **the leading column is `warehouse_id`, not `status`** — because `status` is fixed by the predicate and contributes nothing to the key. Then `priority DESC, created_at ASC` exactly matches the `ORDER BY`, so the Sort disappears. `attempts < 5` is in the predicate rather than the key, so it costs nothing. (Column ordering is Topic 14.)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, user_id, payload FROM orders
WHERE status='awaiting_fulfilment' AND warehouse_id=12 AND attempts < 5
ORDER BY priority DESC, created_at ASC LIMIT 50;
```
```
Limit  (actual time=0.038..0.211 rows=50 loops=1)
  ->  Index Scan using idx_orders_fulfil on orders
        Index Cond: (warehouse_id = 12)
        Buffers: shared hit=53
Execution Time: 0.238 ms
```

**90,318 buffers → 53. 4,089 ms → 0.24 ms. 17,000×.** And the `Sort` node is gone, so `LIMIT 50` now stops early.

**Step 4 — the duplicate orders.**

```js
// the bug
const existing = await db.query(
  'SELECT id FROM orders WHERE idempotency_key=$1', [key]);
if (existing.rows.length) return existing.rows[0];
return db.query('INSERT INTO orders (...) VALUES (...) RETURNING *', [...]);
```

Two requests, both `SELECT` and find nothing, both `INSERT`. The fix is not a better check — it's making the check unnecessary:

```sql
-- ⚠ On a live 340M-row table, this can fail if duplicates already exist.
--    Clean them first, then build without blocking:
CREATE UNIQUE INDEX CONCURRENTLY uq_orders_idem
  ON orders (idempotency_key) WHERE idempotency_key IS NOT NULL;
-- partial: historical rows with NULL keys don't participate

SELECT indisvalid FROM pg_index WHERE indexrelid = 'uq_orders_idem'::regclass;
-- MUST be true. If CREATE INDEX CONCURRENTLY fails it leaves an INVALID
-- index that is maintained on writes but unusable for queries — drop and retry.
```

```js
// the fix — no window
const { rows } = await db.query(
  `INSERT INTO orders (user_id, total_paise, idempotency_key)
   VALUES ($1,$2,$3)
   ON CONFLICT (idempotency_key) WHERE idempotency_key IS NOT NULL
   DO NOTHING
   RETURNING *`, [userId, total, key]);
if (rows.length) return rows[0];
const prev = await db.query('SELECT * FROM orders WHERE idempotency_key=$1', [key]);
return prev.rows[0];      // idempotent replay
```

**Step 5 — clean up.**

```sql
DROP INDEX CONCURRENTLY idx_orders_status;              -- 8 GB, 0 scans
DROP INDEX CONCURRENTLY idx_orders_status_wh_created;   -- 18 GB, replaced
```

**Results:**

| | Before | After |
|---|---|---|
| Worker query p99 | 4,100 ms | **0.3 ms** |
| Index total | 92 GB | **66 GB** |
| Duplicate orders/day | ~40 | **0** |
| Insert throughput | 6,200/s | **9,800/s** |
| WAL per hour | 180 GB | **121 GB** |

**The general lesson:** whenever a query filters on a *small* subset of a *large* table, the ordinary index is the wrong tool. The partial index is often three to five orders of magnitude smaller, and the write saving is usually a bigger win than the read saving.

---

## Common mistakes

**1. Parameterising the partial index predicate.**
- *Symptom:* the index exists and is perfect, but `EXPLAIN` shows a Seq Scan in production while the same query is fast in psql.
- *Engine-level why:* the planner must *prove* the query implies the index predicate. With `$1` and a generic plan, it can't.
- *Diagnose:* `PREPARE` the query, `EXECUTE` it six times, then `EXPLAIN EXECUTE`.
- *Fix:* inline the constant. Parameterise everything else.

**2. A partial index whose predicate is narrower than the query.**
- *Symptom:* index unused.
- *Engine-level why:* `WHERE attempts < 10` does not imply `WHERE attempts < 5`, so the index might be missing rows the query needs.
- *Fix:* make the index predicate at least as broad as every query that should use it — or narrow the queries.

**3. Non-immutable expression indexes.**
- *Symptom:* `ERROR: functions in index expression must be marked IMMUTABLE`, or (worse) someone marks a volatile function `IMMUTABLE` to get past it.
- *Engine-level why:* the index stores the computed value. If the function's output can change, the index becomes silently wrong and returns incorrect rows — a correctness bug, not a performance bug.
- *Fix:* pin the timezone (`AT TIME ZONE 'UTC'`), use `date_trunc`, or store a generated column (`GENERATED ALWAYS AS ... STORED`) and index that.

**4. Assuming a unique index prevents duplicate NULLs.**
- *Symptom:* 40,000 rows with `NULL` in a "unique" column.
- *Engine-level why:* `NULL ≠ NULL` in SQL.
- *Fix:* `NULLS NOT DISTINCT` (PG15+), or `NOT NULL`, or an expression index on `coalesce(...)`.

**5. Plain unique index + soft deletes.**
- *Symptom:* a user deletes their account and can never sign up with that email again.
- *Fix:* `CREATE UNIQUE INDEX ... WHERE deleted_at IS NULL;`

**6. `CREATE INDEX` (not `CONCURRENTLY`) on a live table.**
- *Symptom:* the application freezes for minutes.
- *Engine-level why:* a `SHARE` lock blocks all writes for the whole build.
- *Fix:* `CONCURRENTLY` — but know the caveats: it can't run inside a transaction, it does two table passes (slower), and on failure it leaves an `INVALID` index that still costs writes. Always verify `indisvalid` afterwards.

**7. Using `SELECT ... FOR UPDATE` where a unique index would do.**
- *Symptom:* lock contention on a hot row, or deadlocks.
- *Engine-level why:* a unique index enforces the invariant during the insert with a microsecond page lock. `FOR UPDATE` holds a row lock until COMMIT.
- *Fix:* `INSERT ... ON CONFLICT DO NOTHING` on a unique index. (Case studies 01–03.)

---

## Hands-on proof

**PROVE IT #1 — partial index size.**
```sql
CREATE INDEX i_full ON jobs (status, created_at);
CREATE INDEX i_part ON jobs (created_at) WHERE status='pending';
SELECT relname, pg_size_pretty(pg_relation_size(oid)) FROM pg_class
WHERE relname IN ('i_full','i_part');
```

**PROVE IT #2 — partial indexes skip non-matching writes.**
```sql
DROP INDEX i_full;
\timing on
INSERT INTO jobs (status,payload) SELECT 'done','{}'::jsonb FROM generate_series(1,300000);
CREATE INDEX i_full ON jobs (status, created_at);
INSERT INTO jobs (status,payload) SELECT 'done','{}'::jsonb FROM generate_series(1,300000);
```

**PROVE IT #3 — the parameter trap.**
```sql
EXPLAIN SELECT id FROM jobs WHERE status='pending' LIMIT 5;     -- uses i_part
PREPARE q(text) AS SELECT id FROM jobs WHERE status=$1 LIMIT 5;
EXPLAIN EXECUTE q('pending');  -- ×6 → generic plan → Seq Scan
DEALLOCATE q;
```

**PROVE IT #4 — the planner's implication prover.**
```sql
CREATE INDEX i_impl ON jobs (created_at) WHERE attempts < 5;
EXPLAIN SELECT id FROM jobs WHERE attempts < 5 ORDER BY created_at LIMIT 5;  -- ✓
EXPLAIN SELECT id FROM jobs WHERE attempts < 3 ORDER BY created_at LIMIT 5;  -- ✓ implied
EXPLAIN SELECT id FROM jobs WHERE attempts = 0 ORDER BY created_at LIMIT 5;  -- ✓ implied
EXPLAIN SELECT id FROM jobs WHERE attempts < 9 ORDER BY created_at LIMIT 5;  -- ✗
```

**PROVE IT #5 — unique index has no race window.**
```sql
CREATE TABLE u (k int); CREATE UNIQUE INDEX ON u (k);
-- session 1
BEGIN; INSERT INTO u VALUES (1);          -- do NOT commit
-- session 2
INSERT INTO u VALUES (1);                 -- ⏸ BLOCKS, waiting on session 1
-- session 1
COMMIT;
-- session 2 immediately errors:
-- ERROR: duplicate key value violates unique constraint
```
Session 2 **waited on the in-progress transaction** rather than seeing "no duplicate." That waiting is the mechanism that closes the window.

**PROVE IT #6 — expression index statistics.**
```sql
CREATE TABLE ev (id bigserial, payload jsonb);
INSERT INTO ev (payload) SELECT jsonb_build_object('type',
  (ARRAY['click','view','purchase'])[1+(i%3)]) FROM generate_series(1,1000000) i;
ANALYZE ev;
EXPLAIN SELECT count(*) FROM ev WHERE payload->>'type' = 'purchase';
--  rows=5000     ← default 0.5% guess. WRONG by 66×.
CREATE INDEX ON ev ((payload->>'type'));
ANALYZE ev;
EXPLAIN SELECT count(*) FROM ev WHERE payload->>'type' = 'purchase';
--  rows=333333   ← correct, because the index gave ANALYZE something
--                  to collect statistics on.
```
**The index fixed the row estimate even before it was used for access.** That correction cascades into every join and aggregate decision above it (Topic 15).

**PROVE IT #7 — NULLs.**
```sql
CREATE TABLE n (e text); CREATE UNIQUE INDEX ON n (e);
INSERT INTO n VALUES (NULL),(NULL),(NULL);      -- all succeed
CREATE TABLE n2 (e text); CREATE UNIQUE INDEX ON n2 (e) NULLS NOT DISTINCT;
INSERT INTO n2 VALUES (NULL);                   -- ok
INSERT INTO n2 VALUES (NULL);                   -- ERROR: duplicate key
```

---

## The design decision framework

```
USE A PARTIAL INDEX WHEN:  ★ check this FIRST, always
  ✓ Queries always filter on a small subset (< ~10% of rows)
  ✓ That subset is defined by a CONSTANT predicate
    (status='pending' · deleted_at IS NULL · is_active · created_at > fixed date)
  ✓ The table is large and the subset is small — the bigger the ratio,
    the bigger the win
  ✓ Rows leave the subset over time (queues, active flags) → the index
    SHRINKS instead of growing
  ✗ AVOID when the predicate must be a runtime parameter

USE AN EXPRESSION INDEX WHEN:
  ✓ Queries wrap the column: lower(), date_trunc(), (jsonb->>'k'), md5()
  ✓ The function is genuinely IMMUTABLE
  ✓ You need statistics on a computed value (even without the access path)
  ✗ AVOID when you could rewrite to a sargable range instead —
    `created_at >= X AND < Y` beats an index on `created_at::date`,
    because it needs no extra index at all

USE A UNIQUE INDEX WHEN:
  ✓ ANY time uniqueness is a business rule. No exceptions.
  ✓ Especially for idempotency keys — this is the ONLY race-free option
  ✓ Add WHERE when the rule applies to a subset (active, non-deleted)
  → Prefer a CONSTRAINT if a foreign key must reference it;
    prefer an INDEX if you need partial/expression or CONCURRENTLY

USE A COVERING INDEX (INCLUDE) WHEN:
  ✓ A hot query needs a few extra output columns
  ✓ You can afford the size and can tune autovacuum (Topic 12)
  ✗ AVOID on tables with heavy updates to the included columns

COMBINE THEM — the highest-value indexes in production usually are:
  CREATE UNIQUE INDEX uq_active_email ON users (lower(email))
    WHERE deleted_at IS NULL;
  CREATE INDEX idx_queue ON jobs (queue, priority DESC, created_at)
    INCLUDE (payload) WHERE status = 'pending';

THE SIGNAL TO LOOK FOR:
      SELECT <col>, count(*), round(100.0*count(*)/sum(count(*)) OVER (),4) AS pct
      FROM <table> GROUP BY <col> ORDER BY 2 DESC;

  • One value is > 90% and you never query it        → PARTIAL INDEX on
                                                       the rare values
  • The queried value is < 1% of the table           → PARTIAL, definitely
  • Query wraps the column in a function             → EXPRESSION INDEX,
                                                       or rewrite to a range
  • Duplicates appear in production under retry      → UNIQUE INDEX. Today.
  • `Heap Fetches` dominates a hot query             → INCLUDE
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build the 20M-row `jobs` table. Create both an ordinary index on `(status, created_at)` and a partial index on `(created_at) WHERE status='pending'`. Report both sizes and the `EXPLAIN (ANALYZE, BUFFERS)` output for the worker query. Then measure insert time for 200k `'done'` rows with each index alone, and explain the difference mechanically.

### Exercise 2 — medium (apply it)
You have `users(id, email, deleted_at, tenant_id)`. Requirements: (a) email is unique **case-insensitively**, (b) only among non-deleted users, (c) only within a tenant, (d) a deleted user's email may be reused. Write the single index that enforces all four. Prove it with four `INSERT` statements — two that must succeed and two that must fail — and explain which clause of the index causes each failure.

### Exercise 3 — hard (production simulation)
A `notifications` table: 1.2 billion rows, 640 GB, 11 indexes totalling 310 GB. Distribution: 99.7% `sent`, 0.2% `failed`, 0.1% `queued`. Three hot queries:

```sql
-- Q1, 8,000/s from 40 workers
SELECT id, payload FROM notifications
 WHERE state='queued' AND channel=$1 AND scheduled_at <= now()
 ORDER BY priority DESC, scheduled_at ASC LIMIT 100 FOR UPDATE SKIP LOCKED;

-- Q2, 300/s
SELECT count(*) FROM notifications
 WHERE state='failed' AND attempts < 3 AND created_at > now() - interval '1 hour';

-- Q3, 12,000/s
SELECT id, state, sent_at FROM notifications
 WHERE user_id=$1 ORDER BY created_at DESC LIMIT 20;
```

(a) Design the minimum set of indexes covering all three. For each, justify: partial or not (and the exact predicate), the key column order, what goes in `INCLUDE`, and unique or not.
(b) Estimate each index's size and the total, and compare against the current 310 GB.
(c) Q1 uses `FOR UPDATE SKIP LOCKED`. Explain what that does to your partial index as rows are claimed, and why the index shrinks rather than bloats.
(d) `scheduled_at <= now()` cannot be in a partial index predicate. Explain why, and say where it must go instead.
(e) Q3 has no constant predicate. Explain why a partial index cannot help it, and design the right index anyway.
(f) The table receives 40k inserts/s. Compute the write amplification before and after your redesign, showing your reasoning.
(g) Give the exact, safe deployment procedure for a live 640 GB table, including how you detect a failed `CONCURRENTLY` build and how you roll back.

---

## Mental model checkpoint

1. Name the five B-tree variants and one problem each uniquely solves.
2. Why is a partial index's *write* saving often more valuable than its read saving?
3. Explain the parameter trap. Why can't the planner prove `$1 = 'pending'` implies the predicate, and what's the fix in application code?
4. Why must an expression-index function be `IMMUTABLE`? What kind of bug results if it isn't?
5. Why does a unique index have no race window, while `SELECT`-then-`INSERT` does? Describe what the second transaction actually does.
6. How many NULLs may a unique index contain, and what are the two ways to change that?
7. Write the single index for "email unique per tenant, case-insensitive, among non-deleted users." Name which clause enforces each requirement.

---

## Quick reference card

| Variant | Syntax | Solves |
|---|---|---|
| Ordinary | `ON t (c)` | general lookup |
| Multi-column | `ON t (a,b)` | combined filter + sort (T14) |
| **Partial** | `ON t (c) WHERE p` | small subset of a huge table |
| **Expression** | `ON t (f(c))` | function-wrapped predicates; statistics |
| **Unique** | `CREATE UNIQUE INDEX` | race-free uniqueness |
| **Covering** | `ON t (a) INCLUDE (b)` | index-only scans (T12) |

**Rules to memorise**

| Rule | |
|---|---|
| Partial predicate must be a **constant** | `$1` defeats it |
| Planner must **prove** query ⟹ index predicate | `<3` implies `<5`; `<9` doesn't |
| Expression functions must be `IMMUTABLE` | no `now()`, no unqualified tz casts |
| Unique indexes allow **unlimited NULLs** | unless `NULLS NOT DISTINCT` (PG15+) |
| `INCLUDE` columns are leaf-only | not searchable, not sortable |
| Only a **constraint** can be an FK target | not a bare unique index |
| `CONCURRENTLY` can leave an **INVALID** index | always check `indisvalid` |

**The four highest-value patterns**

```sql
-- 1. Queue / status subset
CREATE INDEX ON jobs (queue, priority DESC, created_at)
  INCLUDE (payload) WHERE status = 'pending';

-- 2. Soft-delete-aware uniqueness
CREATE UNIQUE INDEX ON users (tenant_id, lower(email)) WHERE deleted_at IS NULL;

-- 3. Idempotency
CREATE UNIQUE INDEX ON orders (idempotency_key) WHERE idempotency_key IS NOT NULL;

-- 4. One-active-per-parent
CREATE UNIQUE INDEX ON subscriptions (user_id) WHERE state = 'active';
```

---

## When would I use this at work?

1. **Any table with a `status` column.** Before writing `CREATE INDEX ON t (status)`, you run the distribution query. If one value is 97% of the table, you write a partial index instead — typically 1000× smaller, faster, and nearly free on writes.

2. **A duplicate-records bug report.** Instead of adding a better application check, you add a partial unique index on the idempotency key and switch to `ON CONFLICT DO NOTHING`. That closes the class of bug permanently rather than narrowing the window.

3. **Soft-delete schema review.** Someone adds `UNIQUE(email)` alongside `deleted_at`. You can point out immediately that deleted users will permanently reserve their address, and give the one-line partial-unique fix.

---

## Connected topics

**Understand before this:** 11 (B-tree structure), 12 (`Index Cond` vs `Filter`, `INCLUDE`, index-only scans).

**This unlocks:**
- **14** — composite column order, the other half of index design
- **15** — selectivity: the distribution query that tells you to go partial
- **16** — GIN/GiST/BRIN/hash, genuinely different structures
- **17** — the write cost these variants reduce
- **24** — constraints: unique indexes as enforced business rules
- **27** — soft deletes and the schema patterns that need partial uniques
- **Case studies 01–03** — every one depends on a partial unique index
