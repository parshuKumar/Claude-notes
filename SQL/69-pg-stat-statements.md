# Topic 69 — pg_stat_statements
### SQL Mastery Curriculum — Phase 10: PostgreSQL Specific Power Features

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a busy restaurant kitchen. Hundreds of orders come in every hour. At the end of the night you want to know: *which dishes are eating up my kitchen's time?*

You could stand behind the pass with a stopwatch and time every single plate that goes out. That's `EXPLAIN ANALYZE` — precise, but you can only watch one dish at a time, and you have to be standing there when it happens.

`pg_stat_statements` is different. It's like a little accountant sitting in the corner of the kitchen who, for **every** order all night long, quietly writes down: what dish it was, how long it took, how many times it was ordered, and how much of the pantry (ingredients) it consumed. He doesn't care whether the burger was for table 4 or table 9 — a burger is a burger. He lumps all the burgers together into one line: "Burger — ordered 412 times — 38 minutes of total grill time — avg 5.5 seconds each."

At the end of the night you read his ledger and a truth jumps out that no stopwatch ever could:

> The lobster thermidor takes 12 minutes each — everyone *assumes* it's the problem. But you only sold 3 of them. Meanwhile the "simple" side salad takes 4 seconds each, and you sold 2,000 of them. The salad consumed **more total kitchen time than the lobster.**

That single insight — *the true cost of a query is time-per-call multiplied by number-of-calls, not time-per-call alone* — is the entire reason `pg_stat_statements` exists. The slow query everyone complains about is often not the one hurting your database. The one hurting your database is the fast-looking query that runs ten million times a day.

The accountant is the extension. The ledger is the `pg_stat_statements` view. And "a burger is a burger" — collapsing `WHERE id = 4` and `WHERE id = 9` into the same line — is called **query normalisation**, the single most important concept in this topic.

---

## 2. Connection to SQL Internals

`pg_stat_statements` is not a query you write — it is a C extension that hooks into the executor itself. Understanding *where* it plugs in tells you exactly what it can and cannot measure.

**The hook points.** PostgreSQL exposes a set of function-pointer hooks that extensions can wrap. `pg_stat_statements` installs itself into three of them:

1. **`post_parse_analyze_hook`** — fires after a statement is parsed and semantically analysed, but before planning. This is where the extension computes the **query ID**: a hash (a "fingerprint") of the parse tree with constants stripped out. Two statements that differ only in literal values produce the same query ID here.
2. **`ExecutorStart` / `ExecutorRun` / `ExecutorFinish` / `ExecutorEnd` hooks** — these wrap execution. The extension records the wall-clock start, lets the real executor run, then on `ExecutorEnd` computes elapsed time, rows returned, and reads the block-accounting counters.
3. **`ProcessUtility_hook`** — for utility statements (`CREATE`, `VACUUM`, etc.) that don't go through the normal executor.

**Where the numbers come from.** The buffer counters — `shared_blks_hit`, `shared_blks_read`, `shared_blks_dirtied`, `shared_blks_written` — are read straight from the backend's `BufferUsage` struct, the same counters `EXPLAIN (BUFFERS)` reports. A "block" is one 8 KB page. `shared_blks_hit` means the page was already in the **shared buffer pool** (RAM); `shared_blks_read` means it had to be fetched from the OS/disk (which itself may be an OS page-cache hit, so "read" is not always a physical disk seek). This ties directly back to Topic 68 — the numbers you saw in a single `EXPLAIN (ANALYZE, BUFFERS)` are being silently accumulated for **every** execution and summed per fingerprint.

**Where it lives.** The statistics table is **not** a real heap table. It lives in shared memory — a fixed-size hash table allocated at server start, sized by `pg_stat_statements.max` (default 5000 entries). The `pg_stat_statements` "view" is a set-returning function that reads that shared-memory hash and presents it as rows. Because it's shared memory, all backends contribute to the same counters, and the numbers survive individual disconnections. Because it's fixed-size, when you exceed `max` distinct fingerprints, the least-executed entries are **evicted** — a fact with real consequences we'll return to.

**Why it must be preloaded.** Installing hooks into the executor requires the shared library to be loaded into every backend process's address space *before* any query runs. Postgres forks a new backend per connection from the postmaster; shared-preloaded libraries are inherited by every fork. That is why `shared_preload_libraries` (a server-start-only setting) is mandatory — you cannot hook an executor that has already started.

**MVCC note.** The counters are updated with lightweight spinlocks, not through MVCC. There is no transactional rollback of statistics: if your transaction aborts, the time it spent is still counted. The stats reflect *work the engine actually did*, not *work that logically committed*.

---

## 3. Logical Execution Order Context

`pg_stat_statements` does not sit inside the `FROM → WHERE → GROUP BY → SELECT → ORDER BY → LIMIT` pipeline of any single query. It sits *around* every query. But there are two distinct execution orders that matter here.

**Order A — when your monitored queries run.** For each query the database processes, the timeline is:

```
Parse ─→ Analyze ─→ [query ID computed here] ─→ Plan ─→ Execute ─→ [counters accumulated here]
                          ▲                                              ▲
                 normalisation fingerprint                   time / rows / blocks summed
```

The fingerprint is decided *before* planning. The cost numbers are summed *after* execution. So `pg_stat_statements` captures **total** time including planning (if `pg_stat_statements.track_planning` is on) and execution — but it captures them per fingerprint, not per literal-value variant.

**Order B — when you query the view itself.** When *you* run `SELECT ... FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20`, that is an ordinary query with an ordinary logical execution order:

```
FROM pg_stat_statements()   ← set-returning function reads shared memory
WHERE ...                    ← filter (e.g. exclude your own monitoring queries)
GROUP BY ...                 ← optional: aggregate variants that share queryid
ORDER BY total_exec_time DESC← THE most important line: sort by TOTAL, not mean
LIMIT 20                     ← top offenders only
```

The `ORDER BY total_exec_time DESC` is where the entire discipline of this topic lives. Ordering by `mean_exec_time` shows you the *slowest single queries*; ordering by `total_exec_time` shows you the queries that *consume the most database time in aggregate*. These are almost never the same list, and the second list is the one that matters for capacity and latency. We will hammer this point repeatedly.

One subtlety: your own monitoring query is itself tracked by `pg_stat_statements`. The `SELECT ... FROM pg_stat_statements` you just ran appears as its own row on the next read. Analysts routinely filter it out (`WHERE query NOT LIKE '%pg_stat_statements%'`).

---

## 4. What Is pg_stat_statements?

`pg_stat_statements` is a PostgreSQL extension that records aggregated execution statistics — call count, timing, rows, and buffer I/O — for every *normalised* SQL statement the server executes. It collapses statements that differ only in literal constants into a single tracked entry (a "fingerprint"), so you can see the true cumulative cost of each query *shape* across your whole workload.

Enabling it has two parts: a server config line and a one-time `CREATE EXTENSION`.

```ini
# postgresql.conf  — requires a server RESTART, not just a reload
shared_preload_libraries = 'pg_stat_statements'
   │                        └── the C library must be loaded into every backend at fork time;
   │                            this is why a restart (not `pg_reload_conf()`) is required
   └── comma-separated list; append, don't overwrite, if other libs are present
       e.g. 'pg_stat_statements, auto_explain, pg_cron'

pg_stat_statements.max = 5000
   │                       └── max distinct normalised statements held in shared memory;
   │                           exceeding it evicts the least-executed entries (LRU-ish)
   └── raising it costs a little shared memory; 10000–20000 is common on busy servers

pg_stat_statements.track = top
   │                         └── which statements to track:
   │                             'top'  = only top-level statements (default)
   │                             'all'  = also statements inside functions/procedures
   │                             'none' = disable tracking (keeps view readable)
   └── 'all' is invaluable when heavy logic lives in PL/pgSQL

pg_stat_statements.track_utility = on
   │                                └── whether to track utility commands (VACUUM, CREATE,
   │                                    COPY, etc.); default on
   └── turn off to focus purely on DML/SELECT workload

pg_stat_statements.track_planning = off
   │                                 └── whether to also accumulate PLANNING time & counts
   │                                     (total_plan_time, mean_plan_time, plans);
   │                                     default OFF because it adds a small overhead and
   │                                     can increase lock contention on the hash table
   └── turn on temporarily when you suspect plan-time (not exec-time) is the problem

pg_stat_statements.save = on
   │                       └── persist stats across server restarts by dumping them to
   │                           $PGDATA/pg_stat/ on shutdown and reloading on startup
   └── on by default; turn off if you want a clean slate every restart
```

Then, once, in the database you want to observe:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
--     │                        └── creates the view + accessor function in this database's
--     │                            catalog; the shared-memory data is server-wide, but the
--     │                            VIEW must be created per-database to read it
--     └── idempotent guard so re-running your migration is safe
```

**The core view.** Reading it is just a `SELECT`:

```sql
SELECT
  queryid,                    -- stable hash of the normalised statement (the "fingerprint")
  query,                      -- the normalised text, with constants shown as $1, $2, ...
  calls,                      -- how many times this fingerprint executed
  total_exec_time,           -- cumulative execution time (ms) across all calls
  mean_exec_time,            -- total_exec_time / calls (ms)
  stddev_exec_time,          -- standard deviation of per-call exec time (ms)
  rows,                       -- cumulative rows returned/affected across all calls
  shared_blks_hit,           -- pages served from the buffer cache (RAM)
  shared_blks_read           -- pages fetched from outside the cache (OS/disk)
FROM pg_stat_statements
ORDER BY total_exec_time DESC   -- TOTAL time — the true-offender ordering
LIMIT 20;
```

Note: column names changed in **PostgreSQL 13**. Older versions had a single `total_time` / `mean_time`; from 13 onward these split into `total_exec_time` / `total_plan_time` (and `mean_exec_time` / `mean_plan_time`). This doc uses the modern (13+) names throughout.

---

## 5. Why pg_stat_statements Mastery Matters in Production

1. **It finds the real bottleneck, not the loud one.** Without it, teams optimise the query that *looks* scary in a log (a 4-second report) while a 3-millisecond query running 50,000 times a minute quietly consumes 60% of the database's CPU. `pg_stat_statements` ranks by cumulative cost and surfaces the true offender in one query. This is the number-one reason it exists.

2. **It is the only always-on, whole-workload profiler.** `EXPLAIN ANALYZE` requires you to *already know which query to inspect* and to *run it yourself*. `pg_stat_statements` is passive, continuous, and covers 100% of production traffic — including the queries your ORM generates that you've never seen. You cannot `EXPLAIN` a query you don't know exists.

3. **Normalisation reveals ORM chaos.** Prisma, Sequelize, and TypeORM emit thousands of slightly-different SQL strings. Because the extension collapses them by fingerprint, you instantly see that "these 40,000 log lines are actually the same three queries" — and which of the three is expensive.

4. **Cache-hit ratio per query.** `shared_blks_hit` vs `shared_blks_read` per fingerprint tells you *which specific query* is thrashing the buffer cache. A query with a low hit ratio is doing disk I/O every call — a prime index candidate. Aggregate database-wide hit ratios hide this; per-query ratios pinpoint it.

5. **It quantifies the win before you touch code.** Before adding an index, you can say "this query is 42% of total database time." After deploying, you `pg_stat_statements_reset()` and measure again. It turns "I think that's faster" into "total time for that fingerprint dropped from 8 minutes/hour to 12 seconds/hour." That is how you justify optimisation work to a team.

6. **`stddev_exec_time` catches the intermittent killer.** A query with `mean_exec_time = 5ms` but `stddev_exec_time = 400ms` is *usually* fast but occasionally catastrophic — often due to plan flips, lock waits, or cold cache. Mean alone hides this; the standard deviation exposes it. These variable queries cause the p99 latency spikes users actually notice.

Without this extension you are optimising in the dark — guessing, or reacting to whichever query someone happened to complain about. With it, you optimise by evidence.

---

## 6. Deep Technical Content

### 6.1 Query Normalisation — The Fingerprint

Normalisation is the mechanism that makes the extension useful. When a statement is analysed, `pg_stat_statements` walks the parse tree and replaces every **constant** with a placeholder, then hashes the result. All statements with the same shape hash to the same `queryid` and share one row.

```sql
-- These three physical statements...
SELECT * FROM orders WHERE customer_id = 42;
SELECT * FROM orders WHERE customer_id = 8137;
SELECT * FROM orders WHERE customer_id = 99;

-- ...all normalise to ONE tracked entry:
SELECT * FROM orders WHERE customer_id = $1
```

The stored `query` text shows the constants as `$1`, `$2`, … regardless of whether the client used literals or actual bind parameters. The `queryid` is a 64-bit hash computed from the parse-tree node types plus the identity of tables and columns — **not** from the literal values.

**What gets normalised (constants):**
- Numeric, string, boolean literals in `WHERE`, `VALUES`, `LIMIT`, etc.
- Multiple constants each become `$1`, `$2`, …

**What does NOT get merged (structural differences produce distinct fingerprints):**
- Different column lists: `SELECT id` vs `SELECT id, name` → two entries.
- Different table or column identifiers.
- Different operators: `WHERE x = 1` vs `WHERE x > 1` → two entries.
- Different number of items in an `IN (...)` list — **this is a classic footgun**:

```sql
WHERE id IN (1, 2)        -- normalises to  WHERE id IN ($1, $2)
WHERE id IN (1, 2, 3)     -- normalises to  WHERE id IN ($1, $2, $3)   ← DIFFERENT queryid!
```

Every distinct `IN`-list length is a separate fingerprint. An ORM that builds `IN` lists of varying length can blow up your entry count and evict useful stats. (PostgreSQL 14+ mitigates this: with the parameter list normalisation improvements and the `query_id` machinery, some cases collapse, but the general rule stands — assume different lengths differ. PG 16+ can merge constant lists via `pg_stat_statements` handling of `ArrayExpr`, but do not rely on it across versions.)

**Whitespace and comments:** normalisation is based on the parse tree, so reformatting whitespace or adding/removing comments does **not** create a new entry — the tree is identical. (One exception: a leading comment used as an *application tag* is part of the text but not the tree, so it won't split fingerprints either.)

**`utility` statements** like `VACUUM orders` are tracked by their text (not fully normalised) when `track_utility = on`.

### 6.2 The Full Column Catalogue

The view has many columns; these are the ones that earn their place in real analysis.

| Column | Meaning | Why you read it |
|--------|---------|-----------------|
| `queryid` | 64-bit fingerprint hash | Stable key to track one query across resets/versions; join to logs |
| `query` | Normalised statement text | Human identification of the fingerprint |
| `calls` | Number of executions | The multiplier in "true cost = mean × calls" |
| `total_exec_time` | Cumulative exec time (ms) | **The primary offender ranking** |
| `mean_exec_time` | `total_exec_time / calls` | Per-call latency; the "how slow is one" number |
| `min_exec_time` / `max_exec_time` | Fastest / slowest single call | Range of behaviour; `max` catches worst case |
| `stddev_exec_time` | Std dev of per-call time | Consistency; high value = intermittent spikes |
| `rows` | Cumulative rows returned/affected | `rows/calls` = avg rows per call; huge = fan-out |
| `shared_blks_hit` | Buffer-cache page hits | Numerator of per-query cache-hit ratio |
| `shared_blks_read` | Pages read from outside cache | Disk pressure; high = index candidate |
| `shared_blks_dirtied` | Pages modified | Write workload footprint |
| `shared_blks_written` | Pages evicted to disk by this stmt | Backend doing writeback — memory pressure |
| `temp_blks_read` / `temp_blks_written` | Temp-file I/O | Sorts/hashes spilling `work_mem` to disk |
| `blk_read_time` / `blk_write_time` | Time spent in block I/O (ms) | Only populated if `track_io_timing = on` |
| `total_plan_time` / `mean_plan_time` / `plans` | Planning cost | Only if `track_planning = on` |
| `wal_records` / `wal_bytes` | WAL generated | Write amplification per statement (PG 13+) |

**Derived metrics you compute yourself:**

```sql
-- Average rows per call (fan-out detector)
rows::numeric / NULLIF(calls, 0)

-- Per-query cache hit ratio (0..1); low = disk-bound
shared_blks_hit::numeric / NULLIF(shared_blks_hit + shared_blks_read, 0)

-- Share of total database time consumed by this fingerprint
100.0 * total_exec_time / SUM(total_exec_time) OVER ()

-- Coefficient of variation: how unstable is per-call time?
stddev_exec_time / NULLIF(mean_exec_time, 0)
```

### 6.3 Total Time vs Mean Time — The Central Discipline

This deserves its own worked example because it is the crux of the entire topic.

```sql
-- Two queries in the view:
queryid | query                                   | calls   | mean_exec_time | total_exec_time
--------+-----------------------------------------+---------+----------------+----------------
   A    | SELECT ... big monthly report ...       |      12 |      4200.0 ms |     50,400 ms
   B    | SELECT * FROM sessions WHERE token = $1 | 8,400,000 |        0.9 ms |  7,560,000 ms
```

Query A is the one everyone *feels* — it takes 4.2 seconds and someone stares at a spinner. Query B is invisible; nobody notices 0.9 ms.

But `total_exec_time` tells the truth: **Query B consumes 7,560,000 ms — 150× more total database time than A's 50,400 ms.** Query B is the reason your CPU is pinned and your connection pool is saturated. Optimising A from 4.2 s to 2 s saves 25 seconds/period. Optimising B from 0.9 ms to 0.3 ms (e.g. a covering index on `token`) saves ~5,000,000 ms/period.

**Rule: rank by `total_exec_time DESC` to find what to fix. Use `mean_exec_time` only to understand *how* to fix a single one.** A high-total/low-mean query is fixed by cutting call count (caching, batching) or shaving per-call cost with an index. A high-mean/low-total query may not be worth fixing at all.

### 6.4 Reading Buffer Counters — hit vs read

`shared_blks_hit + shared_blks_read` is the total pages this fingerprint touched over all calls. The ratio tells you where the pages came from.

```sql
SELECT
  query,
  calls,
  shared_blks_hit,
  shared_blks_read,
  round(100.0 * shared_blks_hit
        / NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS hit_pct,
  round((shared_blks_hit + shared_blks_read)::numeric
        / NULLIF(calls, 0), 1) AS blks_per_call
FROM pg_stat_statements
WHERE calls > 100
ORDER BY shared_blks_read DESC   -- queries doing the most out-of-cache reads
LIMIT 20;
```

Interpretation:
- **High `blks_per_call` + high `hit_pct`**: query reads many pages but they're cached. Often a big in-memory scan — an index may cut the page count, or the query genuinely needs the data.
- **Low `hit_pct` (e.g. < 90%)**: this query repeatedly misses the cache — its working set doesn't fit in `shared_buffers`, or it's scanning cold data. Strong index candidate, or a candidate for a smaller/more-selective query.
- **`shared_blks_read` dominating the whole server**: this single fingerprint is your disk-I/O hot spot.

`shared_blks_read` counts pages fetched from *outside the buffer pool* — which may still be served by the OS page cache, so it is not identical to physical disk reads. To measure actual I/O *time*, enable `track_io_timing` and read `blk_read_time`.

### 6.5 Resetting Statistics

Statistics are cumulative from the last reset (or server start). To measure a change, you reset, let traffic flow, then read a clean window.

```sql
-- Reset EVERYTHING (all statements, all users, all databases):
SELECT pg_stat_statements_reset();

-- Targeted reset (PostgreSQL 12+): reset one user / db / query
SELECT pg_stat_statements_reset(
  userid  => 0,       -- 0 = all users
  dbid    => 0,       -- 0 = all databases
  queryid => 0        -- 0 = all queries; pass a specific queryid to reset just one
);

-- Reset only one fingerprint you're actively optimising:
SELECT pg_stat_statements_reset(0, 0, -6543210987654321::bigint);
```

**Reset discipline for benchmarking:**
1. `SELECT pg_stat_statements_reset();`
2. Run representative traffic for a fixed window (e.g. 15 minutes of production load, or a load-test).
3. Read the top-by-total-time snapshot.
4. Deploy your change.
5. Reset again, run the same window, compare.

**Warning:** a reset is global and destroys history. On a monitored production system, prefer a **delta approach** instead of resetting: snapshot the view into a table on a schedule and subtract successive snapshots. Resetting blinds any other tool (Datadog, pgwatch, PMM) that is also reading the view. Many teams *never* reset production and always work in deltas.

### 6.6 The Delta / Snapshot Pattern

Because counters only ever increase, you compute "what happened in the last N minutes" by differencing two snapshots — no reset needed.

```sql
-- Persist a snapshot table once:
CREATE TABLE pgss_snapshots (
  captured_at  timestamptz NOT NULL DEFAULT now(),
  queryid      bigint,
  query        text,
  calls        bigint,
  total_exec_time double precision,
  rows         bigint,
  shared_blks_hit  bigint,
  shared_blks_read bigint
);

-- Capture (run on a schedule via pg_cron or an external job):
INSERT INTO pgss_snapshots
  (queryid, query, calls, total_exec_time, rows, shared_blks_hit, shared_blks_read)
SELECT queryid, query, calls, total_exec_time, rows, shared_blks_hit, shared_blks_read
FROM pg_stat_statements;

-- Delta between the two most recent snapshots:
WITH latest AS (
  SELECT * FROM pgss_snapshots
  WHERE captured_at = (SELECT max(captured_at) FROM pgss_snapshots)
),
prev AS (
  SELECT * FROM pgss_snapshots
  WHERE captured_at = (SELECT max(captured_at) FROM pgss_snapshots
                       WHERE captured_at < (SELECT max(captured_at) FROM pgss_snapshots))
)
SELECT
  l.query,
  l.calls           - p.calls            AS calls_delta,
  l.total_exec_time - p.total_exec_time  AS exec_ms_delta,
  l.rows            - p.rows             AS rows_delta
FROM latest l
JOIN prev   p USING (queryid)
WHERE l.total_exec_time - p.total_exec_time > 0
ORDER BY exec_ms_delta DESC
LIMIT 20;
```

Beware: if the entry was **evicted** and re-created between snapshots, its counters reset, and the delta goes negative. Filter out negative deltas or handle them explicitly.

### 6.7 Entry Eviction and `max`

The shared-memory hash holds at most `pg_stat_statements.max` entries. When a new fingerprint arrives and the table is full, the entry with the **fewest calls** (weighted, effectively an LRU on usage) is deallocated. Symptoms of thrashing:

- The `pg_stat_statements_info` view (PG 14+) exposes a `dealloc` counter — how many times eviction has run. A steadily climbing `dealloc` means `max` is too small for your query diversity.

```sql
-- PG 14+: is the table thrashing?
SELECT dealloc, stats_reset FROM pg_stat_statements_info;
```

- You see fingerprints you *know* run frequently missing from the view, or their `calls` mysteriously resetting.

Fix: raise `pg_stat_statements.max` (requires restart), or fix the *cause* — usually unparameterised queries or variable-length `IN` lists generating fingerprint explosions. Also watch for a special catch-all: when the extension cannot store the query text (text file full), `query` may appear as `<insufficient privilege>` or the entry may show a null query.

### 6.8 Pairing with EXPLAIN

`pg_stat_statements` tells you *which* query to fix and *how much* it costs. It does **not** tell you *why* it's slow — for that you go to `EXPLAIN` (Topic 68). The workflow:

1. Find the top offender by `total_exec_time`.
2. Copy its normalised `query` text.
3. Substitute realistic literals for `$1, $2, …` (use *representative* values — a value that hits many rows, not `id = 1`).
4. Run `EXPLAIN (ANALYZE, BUFFERS)` on it.
5. Read the plan for the actual problem (Seq Scan, bad estimate, spill).

```sql
-- The extension gives you this normalised text:
--   SELECT * FROM orders WHERE customer_id = $1 AND status = $2
-- Substitute representative constants and explain:
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 8137 AND status = 'pending';
```

The `shared_blks_read` in `pg_stat_statements` and the `Buffers: shared read=` in `EXPLAIN` are the *same counter* — one summed across all calls, one for this single run. That continuity lets you confirm you're explaining the right thing.

### 6.9 Related Views — The Observability Family

`pg_stat_statements` is the historical/aggregate lens. It has siblings for the *live* and *table-level* view.

- **`pg_stat_activity`** — one row per connection, showing *what is running right now*: `state`, `query`, `wait_event`, `query_start`, `backend_start`. This is the "who is on the pass **this second**" view. Use it to catch a query mid-flight, find blockers, and kill runaway sessions (`pg_cancel_backend`, `pg_terminate_backend`). Note: its `query` column is the *raw* text (with real literals), unlike `pg_stat_statements`' normalised text.

```sql
-- Live long-running queries (over 30s), newest first:
SELECT pid, now() - query_start AS duration, state, wait_event, query
FROM pg_stat_activity
WHERE state <> 'idle'
  AND now() - query_start > interval '30 seconds'
ORDER BY duration DESC;
```

- **`pg_stat_user_tables`** — one row per table, with `seq_scan`, `seq_tup_read`, `idx_scan`, `idx_tup_fetch`, `n_live_tup`, `n_dead_tup`, `last_autovacuum`. Where `pg_stat_statements` says "this query is slow," `pg_stat_user_tables` says "this table is being sequentially scanned 4 million times — it needs an index."

```sql
-- Tables suffering many sequential scans relative to index scans:
SELECT relname, seq_scan, idx_scan, n_live_tup,
       round(100.0 * seq_scan / NULLIF(seq_scan + idx_scan, 0), 1) AS seq_pct
FROM pg_stat_user_tables
WHERE seq_scan + idx_scan > 0
ORDER BY seq_scan DESC
LIMIT 20;
```

- **`pg_stat_user_indexes`** — `idx_scan` per index; a value of 0 over a long window flags an **unused index** you can drop (write-path savings).
- **`pg_statio_user_tables`** — heap/index block hit/read at the table level, complementing the per-query buffer view.

The mental model: **`pg_stat_activity` = now, `pg_stat_statements` = history-by-query, `pg_stat_user_tables` = history-by-table.** A complete diagnosis usually reads all three.

### 6.10 Permissions and Security

Reading full query text and stats of *other users* requires privilege. By default:
- A superuser (or member of `pg_read_all_stats`, PG 10+) sees all queries and all text.
- A regular user sees their own queries' full text; other users' rows show `query = '<insufficient privilege>'` and their stats may be redacted depending on `pg_stat_statements.track` and role grants.

Grant read access to a monitoring role without superuser:

```sql
GRANT pg_read_all_stats TO monitoring_role;
```

This matters because query text can contain schema details; the extension deliberately gates it.

### 6.11 Overhead

The extension is cheap but not free. Each executed statement takes a spinlock to update its hash entry. On extremely high-throughput, short-query workloads this contention is measurable (typically low single-digit percent). `track_planning = on` roughly doubles the update points and increases contention — keep it off unless investigating plan time. `track = all` adds entries for nested statements. On most systems the overhead is negligible and the visibility is worth it; on a pathological 200k-QPS workload, benchmark before assuming.

---

## 7. EXPLAIN — pg_stat_statements in the Plan

`pg_stat_statements` itself is a set-returning function, so querying it produces a `Function Scan`. More importantly, this section shows the **round trip**: read the view, then `EXPLAIN` the offender it names.

### 7.1 EXPLAIN of the monitoring query

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT queryid, query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

```
Limit  (cost=248.00..248.05 rows=20 width=72)
       (actual time=3.10..3.12 rows=20 loops=1)
  ->  Sort  (cost=248.00..260.50 rows=5000 width=72)
            (actual time=3.09..3.10 rows=20 loops=1)
        Sort Key: total_exec_time DESC
        Sort Method: top-N heapsort  Memory: 29kB
        ->  Function Scan on pg_stat_statements
              (cost=0.00..100.00 rows=5000 width=72)
              (actual time=1.80..2.40 rows=4871 loops=1)
Planning Time: 0.10 ms
Execution Time: 3.20 ms
```

**Reading it:**
- `Function Scan on pg_stat_statements` — the view is a C set-returning function reading shared memory, not a heap table; there is no index and no `Seq Scan`. The `rows=5000` is a hard-coded estimate (functions default to 1000, but this one reports `max`); the actual `rows=4871` is how many live fingerprints exist right now.
- `top-N heapsort` — because of `LIMIT 20`, Postgres keeps only the top 20 in a heap rather than fully sorting all 4,871 rows. Cheap.
- No `Buffers:` line appears because the data is shared memory, not the buffer pool — reading the view does **not** consume `shared_blks`. This is why the monitoring query is lightweight.

### 7.2 EXPLAIN of the offender it surfaced

Suppose the view named this normalised fingerprint as the top total-time consumer:

```
query           | SELECT * FROM sessions WHERE token = $1
calls           | 8,412,006
total_exec_time | 7,559,900 ms
mean_exec_time  | 0.898 ms
shared_blks_hit | 41,900,004
shared_blks_read| 22,410,331     ← notice: huge out-of-cache reads
```

The low-ish cache hit ratio (`~65%`) is the smoking gun. Explain it with a representative token:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM sessions WHERE token = 'a3f9c1e0-...-b7';
```

```
Seq Scan on sessions  (cost=0.00..48210.00 rows=1 width=220)
                       (actual time=0.30..0.88 rows=1 loops=1)
  Filter: (token = 'a3f9c1e0-...-b7'::text)
  Rows Removed by Filter: 1999999
  Buffers: shared hit=3 read=18210
Planning Time: 0.09 ms
Execution Time: 0.89 ms
```

**Reading it:**
- `Seq Scan` with `Rows Removed by Filter: 1999999` — every call scans the whole 2M-row `sessions` table to find one token. There is **no index on `token`**.
- Per call this is only 0.89 ms because the table is mostly cached — but multiplied by 8.4M calls it is 7,560 seconds of total time. This is *exactly* the total-vs-mean lesson made physical: a fast-looking Seq Scan is the biggest cost on the server.
- `Buffers: shared read=18210` per call maps directly to the extension's summed `shared_blks_read`.

The fix — add the index — and re-measure:

```sql
CREATE INDEX CONCURRENTLY idx_sessions_token ON sessions(token);
SELECT pg_stat_statements_reset(0, 0, <that_queryid>);  -- reset just this fingerprint
```

After the index, re-explaining shows an `Index Scan` at ~0.02 ms/call, and the next window's `total_exec_time` for that fingerprint collapses from millions of ms to thousands. That before/after, proven by the extension, is the entire point.

---

## 8. Query Examples

### Example 1 — Basic: Top offenders by total time

```sql
-- The single most useful pg_stat_statements query.
-- Ranks fingerprints by CUMULATIVE execution time — the true cost.
SELECT
  queryid,
  substring(query, 1, 80)            AS query_snippet,   -- truncate for readability
  calls,
  round(total_exec_time::numeric, 0) AS total_ms,        -- the ranking metric
  round(mean_exec_time::numeric, 3)  AS mean_ms,         -- per-call latency
  rows
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'              -- exclude our own monitoring
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Example 2 — Intermediate: Offenders with cache-hit ratio and % of total time

```sql
-- Adds two derived metrics: each query's share of total DB time,
-- and its personal buffer-cache hit ratio.
SELECT
  substring(query, 1, 60) AS query_snippet,
  calls,
  round(total_exec_time::numeric, 0)                        AS total_ms,
  round(mean_exec_time::numeric, 3)                         AS mean_ms,
  round(100.0 * total_exec_time
        / SUM(total_exec_time) OVER (), 1)                  AS pct_of_total,  -- window over all rows
  round(rows::numeric / NULLIF(calls, 0), 1)                AS avg_rows,       -- fan-out signal
  round(100.0 * shared_blks_hit
        / NULLIF(shared_blks_hit + shared_blks_read, 0), 1) AS cache_hit_pct  -- lower = disk-bound
FROM pg_stat_statements
WHERE calls > 50                                            -- ignore rarely-run noise
  AND query NOT LIKE '%pg_stat_statements%'
ORDER BY total_exec_time DESC
LIMIT 25;
```

### Example 3 — Production Grade: Weekly optimisation triage report

```sql
-- Context:
--   • Server processes ~40M statements/day; pg_stat_statements.max = 20000.
--   • track_io_timing = on, track_planning = off.
--   • This report is run every Monday to pick the week's optimisation targets.
--   • It flags queries that are (a) expensive in total, AND (b) either disk-bound,
--     high-variance, or high fan-out — i.e. actionable, not just big.
WITH stats AS (
  SELECT
    queryid,
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    stddev_exec_time,
    rows,
    shared_blks_hit,
    shared_blks_read,
    blk_read_time                                            -- needs track_io_timing = on
  FROM pg_stat_statements
  WHERE query NOT LIKE '%pg_stat_statements%'
    AND calls > 100                                          -- statistical significance
),
ranked AS (
  SELECT
    *,
    100.0 * total_exec_time / SUM(total_exec_time) OVER ()   AS pct_total_time,
    rows::numeric / NULLIF(calls, 0)                         AS avg_rows,
    100.0 * shared_blks_hit
      / NULLIF(shared_blks_hit + shared_blks_read, 0)        AS cache_hit_pct,
    stddev_exec_time / NULLIF(mean_exec_time, 0)             AS cv_time            -- coefficient of variation
  FROM stats
)
SELECT
  substring(query, 1, 70)               AS query_snippet,
  calls,
  round(total_exec_time::numeric, 0)    AS total_ms,
  round(pct_total_time::numeric, 1)     AS pct_total,
  round(mean_exec_time::numeric, 3)     AS mean_ms,
  round(cv_time::numeric, 2)            AS cv,               -- >1.0 = highly variable
  round(avg_rows::numeric, 1)           AS avg_rows,
  round(cache_hit_pct::numeric, 1)      AS hit_pct,
  round(blk_read_time::numeric, 0)      AS io_ms,
  -- A crude "why look at this" tag:
  CASE
    WHEN cache_hit_pct < 90                THEN 'DISK-BOUND: index candidate'
    WHEN cv_time > 1.0                     THEN 'UNSTABLE: plan flip / lock wait'
    WHEN avg_rows > 10000                  THEN 'FAN-OUT: returns too much'
    ELSE 'HIGH-VOLUME: cache or batch'
  END                                    AS action_hint
FROM ranked
WHERE pct_total_time > 1.0                                   -- only queries worth >1% of DB time
ORDER BY total_exec_time DESC
LIMIT 30;
```

```
-- Realistic EXPLAIN of this report query itself:
Limit  (cost=520.00..520.08 rows=30 width=180)
       (actual time=6.90..6.94 rows=27 loops=1)
  ->  Sort  (cost=520.00..530.00 rows=4000 width=180)
            (actual time=6.89..6.91 rows=27 loops=1)
        Sort Key: ranked.total_exec_time DESC
        Sort Method: top-N heapsort  Memory: 40kB
        ->  Subquery Scan on ranked  (cost=200.00..400.00 rows=4000 width=180)
              (actual time=4.10..6.40 rows=27 loops=1)
              Filter: (ranked.pct_total_time > 1.0)
              Rows Removed by Filter: 3844
              ->  WindowAgg  (cost=200.00..320.00 rows=4000 width=210)
                    (actual time=4.08..5.90 rows=3871 loops=1)
                    ->  Function Scan on pg_stat_statements
                          (cost=0.00..100.00 rows=4000 width=160)
                          (actual time=2.00..2.90 rows=3871 loops=1)
Planning Time: 0.20 ms
Execution Time: 7.10 ms
```

Performance expectation: even on a busy server with ~20k fingerprints, this report runs in single-digit milliseconds because the entire dataset is in shared memory and the window aggregate + top-N sort are trivial at that row count. It is safe to run on production during peak.

---

## 9. Wrong → Right Patterns

### Wrong 1: Ranking by `mean_exec_time` to find what to optimise

```sql
-- WRONG: "show me the slowest queries"
SELECT query, mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

**What you get:** the query at the top is a nightly reporting job that runs 4 times a month at 12 seconds each — total impact 48 seconds/month. You spend a day optimising it. Meanwhile the query actually saturating your CPU (0.5 ms × 20M calls = 10,000 s) is nowhere on this list. The execution-level reason: `mean_exec_time` ignores `calls` entirely, so it structurally cannot see cumulative cost. You optimised the loud query, not the expensive one.

```sql
-- RIGHT: rank by cumulative time; mean is a secondary detail
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC   -- total = mean × calls; the real cost
LIMIT 10;
```

### Wrong 2: Expecting the extension to work after only editing postgresql.conf

```sql
-- WRONG: added the line, reloaded, ran CREATE EXTENSION — then got:
--   ERROR:  pg_stat_statements must be loaded via "shared_preload_libraries"
-- (or the view exists but calls/rows never populate)
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';
SELECT pg_reload_conf();   -- ← reload does NOT load a preload library
CREATE EXTENSION pg_stat_statements;
```

**Why it's wrong at the execution level:** `shared_preload_libraries` is a *postmaster-start-time* setting. The library must be mapped into the postmaster's address space **before** it forks any backends, because the extension installs executor hooks at load time. `pg_reload_conf()` (SIGHUP) only re-reads runtime-changeable settings; it cannot inject a library into already-running processes. The setting is marked `PGC_POSTMASTER` — a full restart is mandatory.

```sql
-- RIGHT:
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';
-- Then, at the OS level, RESTART the server:
--   sudo systemctl restart postgresql        (or pg_ctl restart)
-- After restart, in the target database:
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
-- Verify it's active:
SELECT count(*) FROM pg_stat_statements;   -- should return a growing number
```

### Wrong 3: Treating different `IN`-list lengths as one query

```sql
-- WRONG assumption: "my batch loader always runs the same query"
-- The loader emits IN-lists of whatever length the batch happens to be:
SELECT * FROM products WHERE id IN (1,2,3);
SELECT * FROM products WHERE id IN (1,2,3,4,5,6,7);
-- These are TWO different fingerprints. You search pg_stat_statements for
-- "the products IN query" and find its stats split across hundreds of rows,
-- each with a different IN-list length — none of which looks significant alone.
```

**Why it's wrong:** normalisation is structural. A 3-element `IN` list and a 7-element `IN` list have different parse trees (`IN ($1,$2,$3)` vs `IN ($1,$2,$3,$4,$5,$6,$7)`), hence different `queryid`s. The real cumulative cost is fragmented and invisible.

```sql
-- RIGHT: use a single-parameter array form so all batch sizes share one fingerprint
SELECT * FROM products WHERE id = ANY($1);   -- $1 is an int[] of any length
-- Now every batch, regardless of size, is ONE fingerprint in pg_stat_statements.
-- (Bonus: this also stabilises the query plan and cuts parse overhead.)
```

### Wrong 4: Resetting production stats to "get a clean number"

```sql
-- WRONG: on a live system also monitored by Datadog/pgwatch:
SELECT pg_stat_statements_reset();   -- global wipe of ALL history
-- Consequence: every external monitoring tool that computes deltas now sees a
-- massive negative jump (counters dropped to zero), producing false alerts and a
-- gap in your dashboards. You also destroyed the baseline you'd need to compare against.
```

**Why it's wrong:** the reset is server-global and irreversible; there is exactly one set of counters and every reader shares it. You cannot "reset just for me."

```sql
-- RIGHT (option A): reset only the ONE fingerprint you're benchmarking (PG 12+)
SELECT pg_stat_statements_reset(0, 0, <queryid_of_interest>);

-- RIGHT (option B): don't reset at all — snapshot and diff (see §6.6)
INSERT INTO pgss_snapshots SELECT now(), queryid, query, calls, total_exec_time, rows,
       shared_blks_hit, shared_blks_read FROM pg_stat_statements;
-- ...later, subtract two snapshots to get the window you care about.
```

### Wrong 5: Reading raw literals from pg_stat_statements

```sql
-- WRONG expectation: "let me find the exact customer_id that ran slowly"
SELECT query FROM pg_stat_statements WHERE query LIKE '%customer_id = 8137%';
-- Returns NOTHING. The stored text is normalised:  customer_id = $1
-- The literal 8137 was stripped at fingerprinting time and is gone forever.
```

**Why it's wrong:** `pg_stat_statements` deliberately discards constants — that's the whole normalisation mechanism. It aggregates *shapes*, not *instances*. Individual literal values are not retained anywhere in the view.

```sql
-- RIGHT: for the specific offending value, use the LIVE view or logs.
-- pg_stat_activity shows real literals of currently-running queries:
SELECT pid, query FROM pg_stat_activity WHERE query LIKE '%customer_id = 8137%';
-- Or enable log_min_duration_statement to capture slow individual executions
-- WITH their literal values in the server log.
```

---

## 10. Performance Profile

### 10.1 Overhead of the extension itself

| Aspect | Cost | Notes |
|--------|------|-------|
| Per-statement update | 1 spinlock acquire + hash update | ~sub-microsecond; contention rises with QPS |
| Memory | `max` × ~entry size (few KB each) + a text file for query strings | 5000 entries ≈ small MB range |
| `track_planning = on` | Roughly doubles update sites | Extra contention; keep off unless investigating plan time |
| `track = all` | Adds nested-statement entries | More fingerprints, more `max` pressure |
| Reading the view | `Function Scan` over shared memory | No buffer I/O; single-digit ms even at 20k entries |

### 10.2 Scaling the *number of fingerprints* (not rows)

Unlike a normal table, `pg_stat_statements` does not scale with your data volume — a 100M-row `orders` table produces the *same* number of fingerprints as a 1k-row one. It scales with **query diversity**.

| Distinct fingerprints | Behaviour | Action |
|-----------------------|-----------|--------|
| < 1,000 | Comfortably under default `max=5000` | Nothing needed |
| ~5,000 | At default ceiling; eviction begins | Watch `pg_stat_statements_info.dealloc` |
| 10k–50k | Common on ORM-heavy apps with `IN`-list explosions | Raise `max`; fix `IN` → `= ANY($1)` |
| > `max`, thrashing | Frequent eviction; stats for real queries vanish | Raise `max` AND stop generating fingerprints |

The dominant scaling risk is **fingerprint explosion** from unparameterised SQL: string-concatenated literals and variable-length `IN` lists. Each variant is a new entry; the table fills; useful stats get evicted. The fix is parameterisation, not a bigger table.

### 10.3 Interaction with the queries it monitors

The extension changes nothing about how your queries plan or execute — it only observes. But it changes *your* behaviour: it lets you target the queries whose `total_exec_time` dominates. The typical distribution is Pareto — the top 5–10 fingerprints account for 80–95% of total database time. Optimising those (via indexes surfaced by pairing with `EXPLAIN`, or by cutting call counts) yields nearly all the available win.

### 10.4 `track_io_timing` — turning on I/O attribution

By default `blk_read_time`/`blk_write_time` are zero. Enabling `track_io_timing = on` (a runtime setting, `SET`/reload) makes the extension attribute *actual I/O wait time* per query — invaluable for separating "slow because CPU" from "slow because disk." The overhead is a `gettimeofday()` per block I/O; on systems with a fast clock source (`tsc`) it's negligible. Verify clock cost first:

```sql
-- Check that reading the clock is cheap on this host before enabling globally:
-- (pg_test_timing is a command-line tool shipped with PostgreSQL)
-- $ pg_test_timing
```

### 10.5 Optimisation techniques specific to this topic

1. **Parameterise to consolidate fingerprints** — `= ANY($1)` over `IN (...)`; bind parameters over string concatenation. Fewer, denser entries = clearer signal and no eviction.
2. **Set `max` to comfortably exceed steady-state fingerprint count** — check `SELECT count(*) FROM pg_stat_statements` and size `max` ~2× above it.
3. **Use `track = all` when logic is in PL/pgSQL** — otherwise a function that internally runs the real work shows up as a single opaque `SELECT my_func(...)` and hides the expensive inner statements.
4. **Combine with `pg_stat_user_tables`** — once a query is flagged, confirm the *table's* `seq_scan` count to validate an index will help.
5. **Snapshot deltas beat resets** — for continuous production monitoring, never reset; diff snapshots so you can look at arbitrary time windows retroactively.

---

## 11. Node.js Integration

### 11.1 Enabling and reading the top-offenders report

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// One-time setup (runs in a migration; requires the library to already be
// in shared_preload_libraries and the server restarted).
async function ensurePgStatStatements() {
  await pool.query('CREATE EXTENSION IF NOT EXISTS pg_stat_statements');
}

// The core report: top offenders by TOTAL time.
async function topOffenders(limit = 20) {
  const { rows } = await pool.query(
    `SELECT
       queryid::text                         AS query_id,   -- bigint → text to avoid JS precision loss
       substring(query, 1, 120)              AS query,
       calls,
       round(total_exec_time::numeric, 0)    AS total_ms,
       round(mean_exec_time::numeric, 3)     AS mean_ms,
       rows,
       round(100.0 * shared_blks_hit
             / NULLIF(shared_blks_hit + shared_blks_read, 0), 1) AS cache_hit_pct
     FROM pg_stat_statements
     WHERE query NOT LIKE '%pg_stat_statements%'
     ORDER BY total_exec_time DESC
     LIMIT $1`,
    [limit]
  );
  return rows;
}
```

**Critical gotcha:** `queryid` is a 64-bit `bigint`. JavaScript's `number` cannot represent it precisely (safe integers stop at 2^53). The `pg` driver returns `bigint` columns as **strings** by default — good — but if you cast or aggregate, always `::text` the `queryid` (as above) so you never truncate it. Never `parseInt` a queryid.

### 11.2 Snapshot-and-diff scheduled capture

```javascript
// Capture a snapshot on a schedule (e.g. every 5 min via node-cron).
// Never resets — computes windows by differencing later.
async function captureSnapshot() {
  await pool.query(
    `INSERT INTO pgss_snapshots
       (queryid, query, calls, total_exec_time, rows, shared_blks_hit, shared_blks_read)
     SELECT queryid, query, calls, total_exec_time, rows, shared_blks_hit, shared_blks_read
     FROM pg_stat_statements`
  );
}

// Compute the delta between the two most recent snapshots.
async function windowDelta(minTotalMs = 100) {
  const { rows } = await pool.query(
    `WITH latest AS (
       SELECT * FROM pgss_snapshots
       WHERE captured_at = (SELECT max(captured_at) FROM pgss_snapshots)
     ),
     prev AS (
       SELECT * FROM pgss_snapshots
       WHERE captured_at = (
         SELECT max(captured_at) FROM pgss_snapshots
         WHERE captured_at < (SELECT max(captured_at) FROM pgss_snapshots))
     )
     SELECT
       substring(l.query, 1, 120)             AS query,
       l.calls           - p.calls            AS calls_delta,
       round((l.total_exec_time - p.total_exec_time)::numeric, 0) AS exec_ms_delta
     FROM latest l
     JOIN prev   p USING (queryid)
     WHERE l.total_exec_time - p.total_exec_time > $1   -- ignore trivial / negative (evicted) deltas
     ORDER BY exec_ms_delta DESC
     LIMIT 20`,
    [minTotalMs]
  );
  return rows;
}
```

### 11.3 Targeted reset for a benchmark

```javascript
// Reset ONLY the fingerprint under test (PG 12+), run load, re-read.
async function resetOneFingerprint(queryId /* string */) {
  await pool.query(
    'SELECT pg_stat_statements_reset(0, 0, $1::bigint)',
    [queryId]                      // pass as string; ::bigint cast preserves precision
  );
}

// Full global reset — guard it; dangerous on shared/monitored systems.
async function resetAll() {
  if (process.env.NODE_ENV === 'production') {
    throw new Error('Refusing global pg_stat_statements_reset() in production');
  }
  await pool.query('SELECT pg_stat_statements_reset()');
}
```

### 11.4 Live view via pg_stat_activity

```javascript
// Complement the historical view with what's running RIGHT NOW.
async function longRunningQueries(seconds = 30) {
  const { rows } = await pool.query(
    `SELECT pid,
            (now() - query_start)      AS duration,
            state,
            wait_event_type,
            wait_event,
            query
     FROM pg_stat_activity
     WHERE state <> 'idle'
       AND now() - query_start > make_interval(secs => $1)
     ORDER BY duration DESC`,
    [seconds]
  );
  return rows;
}
```

**Does an ORM handle any of this?** No. `pg_stat_statements`, `pg_stat_activity`, and `pg_stat_user_tables` are observability catalog objects; every ORM reaches them only through raw SQL. There is no `prisma.pgStatStatements.findMany()`. This is purely a raw-query domain — which is fine, because it's read-only introspection you run from a script or an internal admin endpoint, not application code.

---

## 12. ORM Comparison

Unlike most topics, `pg_stat_statements` is not something an ORM *emits* — it's a monitoring facility you *read*. So the comparison is really: **how well does each ORM let you read a system view, and — just as important — does the ORM's own SQL parameterise cleanly so it produces tidy fingerprints?**

### Prisma

**Can Prisma read it?** Only via `$queryRaw`. There is no model for a system view.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

const offenders = await prisma.$queryRaw<Array<{
  query_id: string; query: string; calls: bigint; total_ms: number;
}>>`
  SELECT queryid::text AS query_id, substring(query,1,120) AS query,
         calls, round(total_exec_time::numeric, 0) AS total_ms
  FROM pg_stat_statements
  WHERE query NOT LIKE '%pg_stat_statements%'
  ORDER BY total_exec_time DESC
  LIMIT 20`;
```

**Fingerprint quality:** Prisma parameterises all user values (`$1, $2`), so its generated SQL normalises cleanly — good. But Prisma's relation loading can emit different SQL shapes per query variant, and its `IN` clauses (for `in: [...]` filters) produce variable-length lists that fragment fingerprints. **Verdict:** read via `$queryRaw`; watch for `IN`-list fingerprint spread from `where: { id: { in: [...] } }`.

### Drizzle

**Can Drizzle read it?** Via the `sql` template / `db.execute`.

```typescript
import { sql } from 'drizzle-orm';
const offenders = await db.execute(sql`
  SELECT queryid::text AS query_id, substring(query,1,120) AS query,
         calls, round(total_exec_time::numeric,0) AS total_ms
  FROM pg_stat_statements
  ORDER BY total_exec_time DESC
  LIMIT 20`);
```

**Fingerprint quality:** Drizzle emits tight parameterised SQL close to what you'd write by hand, so fingerprints are clean and predictable. For `IN`, prefer `inArray` — but note it still expands to a positional list, so the length-explosion caveat applies; use a raw `= ANY(${array})` when you want a single fingerprint. **Verdict:** best of the group for both reading and producing clean fingerprints.

### Sequelize

**Can Sequelize read it?** Via `sequelize.query` with `QueryTypes.SELECT`.

```javascript
const { QueryTypes } = require('sequelize');
const offenders = await sequelize.query(
  `SELECT queryid::text AS query_id, substring(query,1,120) AS query,
          calls, round(total_exec_time::numeric,0) AS total_ms
   FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20`,
  { type: QueryTypes.SELECT }
);
```

**Fingerprint quality:** Sequelize parameterises values but is notorious for verbose generated SQL and for `IN` lists that vary in length, spreading fingerprints. Its eager-loading (`include`) can also generate large distinct query shapes. **Verdict:** readable via raw query; expect noisier `pg_stat_statements` output — filter and group aggressively.

### TypeORM

**Can TypeORM read it?** Via `dataSource.query()`.

```typescript
const offenders = await dataSource.query(
  `SELECT queryid::text AS query_id, substring(query,1,120) AS query,
          calls, round(total_exec_time::numeric,0) AS total_ms
   FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20`
);
```

**Fingerprint quality:** TypeORM parameterises via `$1` and generally produces stable shapes; `whereInIds` and `In([...])` produce variable-length `IN` lists (fingerprint spread) unless you use `= ANY(:...ids)` via raw. **Verdict:** fine for reading; standard `IN`-list caveat.

### Knex

**Can Knex read it?** Directly — it's the most transparent.

```javascript
const offenders = await knex
  .select(knex.raw('queryid::text AS query_id'),
          knex.raw('substring(query,1,120) AS query'),
          'calls',
          knex.raw('round(total_exec_time::numeric,0) AS total_ms'))
  .from('pg_stat_statements')
  .whereRaw("query NOT LIKE '%pg_stat_statements%'")
  .orderBy('total_exec_time', 'desc')
  .limit(20);
```

**Fingerprint quality:** Knex builds parameterised SQL and its `.whereIn` expands to positional placeholders (length-varying → fingerprint spread); use `.whereRaw('id = ANY(?)', [arr])` for a single fingerprint. **Verdict:** cleanest to read a system view with a query builder; same `IN`-list note.

### ORM Summary Table

| ORM | Reads pg_stat_statements via | Own SQL parameterised? | `IN`-list fingerprint spread | Verdict |
|-----|------------------------------|------------------------|------------------------------|---------|
| Prisma | `$queryRaw` | Yes | Yes (`in: [...]`) | Read raw; watch IN spread |
| Drizzle | `sql` / `db.execute` | Yes | Yes via `inArray`; avoid with `= ANY` | Cleanest fingerprints |
| Sequelize | `sequelize.query` | Yes | Yes, plus verbose SQL | Noisiest output |
| TypeORM | `dataSource.query()` | Yes | Yes via `In([...])` | Fine; IN caveat |
| Knex | `.from('pg_stat_statements')` | Yes | Yes via `.whereIn` | Most transparent reader |

**Cross-cutting lesson:** the single biggest thing any ORM does to `pg_stat_statements` health is generate variable-length `IN` lists. Prefer array parameters (`= ANY($1)`) wherever the ORM allows a raw escape hatch, and your fingerprint count stays small and your stats stay meaningful.

---

## 13. Practice Exercises

### Exercise 1 — Easy

`pg_stat_statements` is installed and populated. Write a single query that returns the **top 10 queries by cumulative execution time**, showing the truncated query text (first 80 chars), `calls`, `total_exec_time` rounded to whole ms, and `mean_exec_time` rounded to 3 decimals. Exclude any row whose query text mentions `pg_stat_statements` itself.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines prior topics)

Using `pg_stat_statements`, produce a report of queries that are **disk-bound**: for every fingerprint with more than 500 `calls`, compute the buffer-cache hit ratio (`shared_blks_hit / (shared_blks_hit + shared_blks_read)`), the average rows per call, and each query's percentage share of total execution time (use a window function — recall window functions from earlier topics). Return only queries with a cache-hit ratio below 95%, ordered by `total_exec_time` descending. Handle the divide-by-zero cases correctly.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

Your API's p99 latency spikes intermittently, but the team can't reproduce it — the "slow" query looks fast on average. You suspect a fingerprint that is *usually* fast but *occasionally* catastrophic.

1. Write a query against `pg_stat_statements` that surfaces high-**variance** queries: those where `stddev_exec_time` is large relative to `mean_exec_time` (coefficient of variation > 1) **and** `max_exec_time` is at least 100× `mean_exec_time`, restricted to fingerprints with more than 1,000 calls.
2. Explain why ranking by `mean_exec_time` (the naive approach) would completely miss these queries.
3. Once you've identified a suspect fingerprint, describe the exact next step to diagnose *why* it occasionally spikes (name the view and the concept).

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

You join a team whose PostgreSQL server is at 90% CPU. `pg_stat_statements` is **not** installed. Walk through, as runnable SQL and commands where applicable:

1. The exact steps (config line, restart requirement, `CREATE EXTENSION`) to enable it safely on a production server, and why a `pg_reload_conf()` is insufficient.
2. The one query you'd run first, and precisely why you order it the way you do.
3. Given the top fingerprint it returns, the two-view follow-up (which view confirms the table-level cause, which command produces the plan) that turns "this query is expensive" into "here is the index to add."
4. How you'd measure that your fix worked *without* globally resetting the stats and breaking the team's existing dashboards.

```sql
-- Write your answer/queries here
```

---

## 14. Interview Questions

### Junior Level

**Q: What does `pg_stat_statements` do, and how do you turn it on?**

*Junior answer:* "It records statistics about queries. You run `CREATE EXTENSION pg_stat_statements`."

*Principal answer:* "It's a shared-library extension that hooks the executor to accumulate per-*normalised-query* statistics — call count, timing, rows, and buffer I/O — in a fixed-size shared-memory hash. 'Normalised' is key: it strips literal constants so `WHERE id = 4` and `WHERE id = 9` collapse into one tracked fingerprint (`WHERE id = $1`), letting you see a query shape's cumulative cost across the whole workload. Enabling it is two steps: add `pg_stat_statements` to `shared_preload_libraries` in postgresql.conf and **restart** the server — a reload won't do, because the library must be loaded before backends fork so it can install executor hooks — then run `CREATE EXTENSION pg_stat_statements` in each database you want to read it from."

*Interviewer follow-up:* "Why a restart and not a reload?" → Because `shared_preload_libraries` is a `PGC_POSTMASTER` setting; the shared object has to be mapped into the postmaster before it forks any backend, and executor hooks can only be installed at load time.

---

**Q: You have two queries — one takes 4 seconds, another takes 0.5 ms. Which do you optimise first?**

*Junior answer:* "The 4-second one, obviously — it's way slower."

*Principal answer:* "It depends entirely on `calls`. The metric that matters is `total_exec_time = mean × calls`. If the 4-second query runs 10 times a day (40 s total) and the 0.5 ms query runs 20 million times a day (10,000 s total), the 'fast' query consumes 250× more database time and is the real bottleneck. I'd rank `pg_stat_statements` by `total_exec_time DESC`, not `mean_exec_time`. The slow query is what users *notice*; the high-total query is what's *saturating the server*. They're rarely the same, and confusing them is the classic optimisation mistake."

*Interviewer follow-up:* "How would you actually cut the cost of the high-volume one?" → Two levers: reduce per-call cost (an index — pair the fingerprint with `EXPLAIN` to find the Seq Scan), or reduce call count (application caching, batching, or an `= ANY($1)` to consolidate).

---

**Q: In `pg_stat_statements`, the `query` column shows `$1` where I expected a real value. Why, and how do I find the real value?**

*Junior answer:* "That's a bug or a bind parameter."

*Principal answer:* "That's normalisation working as designed. The extension replaces every constant with a placeholder at fingerprinting time and hashes the resulting parse tree — so all executions of the same shape aggregate into one row, regardless of the literals used. The literal values are discarded and not stored anywhere in the view. To find a specific offending value you go elsewhere: `pg_stat_activity` shows real literals of currently-running queries, and `log_min_duration_statement` captures slow individual executions with their literals in the server log. `pg_stat_statements` answers 'which query shape is expensive'; the log and the activity view answer 'which specific execution was slow'."

*Interviewer follow-up:* "What creates *separate* fingerprints even though they look similar?" → Structural differences: different columns, operators, or — the classic — different `IN`-list lengths, each of which is a distinct parse tree.

---

### Principal Level

**Q: Your `pg_stat_statements` view seems to be missing stats for queries you know run constantly, and their `calls` counts occasionally reset. What's happening and how do you fix it?**

*Principal answer:* "That's entry eviction. The shared-memory hash holds at most `pg_stat_statements.max` fingerprints (default 5000). When it's full and a new fingerprint arrives, the least-used entry is deallocated — so a frequently-run query can be evicted if there's a flood of one-off fingerprints crowding it out. On PG 14+ I'd confirm by reading `pg_stat_statements_info.dealloc`; a steadily climbing value means chronic thrashing. The root cause is almost always **fingerprint explosion** — unparameterised SQL (string-concatenated literals) or variable-length `IN` lists, where every literal or every list length is a distinct entry. The fix is twofold: raise `max` (restart) to comfortably exceed steady-state fingerprint count, *and* eliminate the explosion by parameterising — replace `IN (...)` with `= ANY($1)` so all batch sizes share one fingerprint. Raising `max` alone just delays the problem if the app keeps minting new shapes."

*Interviewer follow-up:* "How do you size `max`?" → Measure `SELECT count(*) FROM pg_stat_statements` at steady state and set `max` to roughly 2× that, after fixing the parameterisation so the count is stable.

---

**Q: Walk me through going from '`pg_stat_statements` says query X is 40% of total time' to a deployed fix, and how you prove the fix worked.**

*Principal answer:* "Five steps. (1) Take the normalised text of X from the view. (2) Substitute *representative* literals for the `$n` placeholders — a value that touches many rows, not `id = 1` — and run `EXPLAIN (ANALYZE, BUFFERS)`. The buffer numbers there are the same counters `pg_stat_statements` summed, so I know I'm explaining the right thing. (3) Read the plan for the real cause — typically a `Seq Scan` with high `Rows Removed by Filter`, or a bad row estimate, or a `work_mem` spill. (4) Deploy the fix — usually an index (`CREATE INDEX CONCURRENTLY` to avoid locking), sometimes a query rewrite. (5) Prove it: rather than a global `pg_stat_statements_reset()` — which would break every dashboard reading the same counters — I reset only that fingerprint with `pg_stat_statements_reset(0, 0, queryid)`, or better, diff two snapshots across the deploy. I want to show `total_exec_time` for X collapsing and its share of total time dropping, in a specific time window. That converts 'I think it's faster' into 'this fingerprint went from 40% of DB time to 2%.'"

*Interviewer follow-up:* "Why representative literals and not any value?" → Because the plan is data-dependent; `id = 1` might return one row via an existing index and hide the Seq Scan that a high-cardinality value would trigger. The extension aggregates over *all* values, so you must explain a value typical of the expensive ones.

---

**Q: A query has `mean_exec_time = 3 ms` but users report occasional multi-second stalls on that endpoint. `mean` looks fine. How do you use `pg_stat_statements` to investigate?**

*Principal answer:* "Mean hides tail behaviour. I'd look at `stddev_exec_time` and `max_exec_time` for that fingerprint. A `mean` of 3 ms with a `stddev` of 500 ms and a `max` of 4 s means the query is usually fast but occasionally catastrophic — a coefficient of variation (`stddev/mean`) well above 1 flags it. `pg_stat_statements` tells me *that* it's unstable but not *why* — it aggregates, so it can't show me the one bad execution. The causes are usually: a plan flip (the planner sometimes picks a Seq Scan when statistics are stale or a parameter value is atypical), lock/buffer waits, or cold-cache reads. Next steps: enable `track_io_timing` to see if `blk_read_time` spikes (disk), check `pg_stat_activity` and `pg_locks` during a stall for lock waits, consider `auto_explain` with `log_min_duration` to capture the plan of the *slow* executions specifically, and review whether a generic plan vs custom plan flip is happening for a prepared statement. The extension pointed me at the unstable fingerprint; the diagnosis of *why* lives in the live views, I/O timing, and `auto_explain`."

*Interviewer follow-up:* "Why can't `mean_exec_time` ever reveal this?" → Because it's `total/calls` — a single scalar that mathematically averages away the tail. You need `stddev`, `max`, or per-execution logging to see variance.

---

## 15. Mental Model Checkpoint

1. Two fingerprints: A has `mean_exec_time = 900 ms, calls = 20`; B has `mean_exec_time = 0.4 ms, calls = 50,000,000`. Which appears first when you `ORDER BY total_exec_time DESC`, and which is the real production problem? Compute both totals.

2. You run `SELECT * FROM users WHERE email = 'a@x.com'` and `SELECT * FROM users WHERE email = 'b@y.com'`. How many rows do these create in `pg_stat_statements`, and what does the `query` column show for them?

3. You added `pg_stat_statements` to `shared_preload_libraries` and ran `SELECT pg_reload_conf()`. The extension still doesn't populate. Why, and what did you actually need to do?

4. A batch job runs `... WHERE id IN (1,2)` sometimes and `... WHERE id IN (1,2,3,4)` other times. Why might its cost look insignificant in `pg_stat_statements` even though it's your heaviest job? What single rewrite fixes both the stats *and* the plan stability?

5. You want to prove an index helped, but three external dashboards read `pg_stat_statements` and compute deltas. Why is `SELECT pg_stat_statements_reset()` the wrong move, and what are two alternatives?

6. `pg_stat_statements` shows a fingerprint at `$1, $2` — you need the exact literal value of a slow real execution to reproduce a bug. Which view or log gives you the literal, and why doesn't `pg_stat_statements` have it?

7. A fingerprint has `shared_blks_hit = 100, shared_blks_read = 900` over its lifetime. What's its cache-hit ratio, what does that imply, and which *other* view would you consult to confirm the table needs an index?

---

## 16. Quick Reference Card

```sql
-- ── ENABLE (two steps + a RESTART) ─────────────────────────────
-- 1) postgresql.conf (PGC_POSTMASTER — needs full restart, NOT reload):
--    shared_preload_libraries = 'pg_stat_statements'
--    pg_stat_statements.max = 10000        -- distinct fingerprints held
--    pg_stat_statements.track = top        -- top | all | none
--    pg_stat_statements.track_planning = off
-- 2) Per database:
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- ── THE ONE QUERY: top offenders by TOTAL time (not mean!) ─────
SELECT substring(query,1,80) AS q, calls,
       round(total_exec_time::numeric,0) AS total_ms,
       round(mean_exec_time::numeric,3)  AS mean_ms, rows
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
ORDER BY total_exec_time DESC   -- total = mean × calls = the real cost
LIMIT 20;

-- ── DERIVED METRICS ───────────────────────────────────────────
rows::numeric / NULLIF(calls,0)                              -- avg rows/call (fan-out)
shared_blks_hit::numeric/NULLIF(shared_blks_hit+shared_blks_read,0)  -- cache hit ratio
100.0*total_exec_time/SUM(total_exec_time) OVER ()          -- % of total DB time
stddev_exec_time/NULLIF(mean_exec_time,0)                   -- CV: >1 = unstable

-- ── RESET (careful!) ──────────────────────────────────────────
SELECT pg_stat_statements_reset();                 -- GLOBAL wipe (breaks dashboards)
SELECT pg_stat_statements_reset(0,0,<queryid>);    -- just ONE fingerprint (PG12+)
SELECT dealloc FROM pg_stat_statements_info;       -- eviction counter (PG14+)

-- ── KEY COLUMNS ───────────────────────────────────────────────
-- calls, total_exec_time, mean_exec_time, stddev_exec_time, min/max_exec_time,
-- rows, shared_blks_hit, shared_blks_read, temp_blks_*, blk_read_time (track_io_timing),
-- wal_bytes, queryid (::text in JS to avoid bigint precision loss)

-- ── PAIR WITH EXPLAIN (the round trip) ────────────────────────
-- 1. Find offender via total_exec_time.  2. Sub REPRESENTATIVE literals for $n.
EXPLAIN (ANALYZE, BUFFERS) SELECT ... WHERE token = 'real-value';  -- find the Seq Scan

-- ── RELATED VIEWS ─────────────────────────────────────────────
-- pg_stat_activity     → what's running NOW (raw literals, wait_event, kill runaways)
-- pg_stat_user_tables  → per-TABLE seq_scan vs idx_scan (confirms index need)
-- pg_stat_user_indexes → idx_scan=0 → unused index, drop it

-- ── PERF RULES OF THUMB ───────────────────────────────────────
-- • Rank by TOTAL time to choose targets; use MEAN only to understand one.
-- • IN (...) of varying length = many fingerprints → use = ANY($1).
-- • Don't reset production; snapshot & diff instead.
-- • track_planning off by default; track_io_timing on if clock is cheap.
-- • Size max ~2× steady-state fingerprint count; watch dealloc.

-- ── INTERVIEW ONE-LINERS ──────────────────────────────────────
-- "The true cost of a query is mean × calls, not mean."
-- "Normalisation collapses literals into $1 — it aggregates shapes, not instances."
-- "shared_preload_libraries needs a RESTART; it installs executor hooks before fork."
-- "pg_stat_statements says WHICH and HOW MUCH; EXPLAIN says WHY."
```

---

## Connected Topics

- **Topic 68 — EXPLAIN Options**: The other half of the diagnosis loop. `pg_stat_statements` finds *which* query and *how much* it costs; `EXPLAIN (ANALYZE, BUFFERS)` reveals *why* — the same buffer counters (`shared hit/read`) appear in both, summed in one and per-run in the other.
- **Topic 70 — Table Partitioning**: A frequent *fix* for the disk-bound, high-total-time offenders this extension surfaces — partition pruning cuts the `shared_blks_read` that showed up in the per-query buffer counters.
- **Buffer pool / shared_buffers internals**: `shared_blks_hit` vs `shared_blks_read` is a direct window into the buffer manager — the same cache whose hit ratio governs whether a query is CPU- or I/O-bound.
- **Executor hooks & MVCC**: The extension lives in `ExecutorRun`/`ExecutorEnd` hooks; understanding the executor pipeline explains what it can measure (work done) and can't (logical commit).
- **Topic 07 — NULL in Depth**: `NULLIF` guards every ratio you compute against `pg_stat_statements` (divide-by-zero on `calls` or `hit+read`), a direct application of NULL semantics.
- **Window functions**: `SUM(total_exec_time) OVER ()` to compute each query's share of total database time is the idiomatic reporting pattern, reusing window mechanics from the aggregation phase.
- **Prepared statements & generic vs custom plans**: A common cause of the high-`stddev` fingerprints this extension flags — plan flips between generic and custom plans produce the intermittent latency spikes mean-time hides.
