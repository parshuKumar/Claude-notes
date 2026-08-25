# 67 — Performance Investigation
## Phase: Reliability & Operations

---

## ELI5 — The Simple Analogy

A doctor with a patient who says *"I feel terrible."*

A bad doctor guesses: *"probably a virus, take these."* Sometimes right, often wrong, and when it's wrong they guess again.

A good doctor follows a **sequence that narrows the space**: temperature, pulse, blood pressure, blood test. Each one **rules out whole categories** before the next. They don't start with an MRI, because an MRI of the wrong organ tells you nothing.

★ **Database performance investigation is exactly this, and almost everyone does it backwards.** They start with `EXPLAIN` on the query they suspect — which is the MRI. It is the *last* step, not the first, because it only helps once you already know *which* query and *what kind* of problem it is.

And the diagnostic insight that organises everything:

★ **A system that is slow because it is busy looks completely different from a system that is slow because it is waiting.** High CPU with high throughput is saturation. **Low CPU with high latency is contention or queueing** — and those have opposite fixes. Adding capacity fixes one and makes the other worse.

---

## Where this fits in the big picture

```
   Every topic so far has been ONE possible answer.
   ★ THIS TOPIC IS HOW YOU FIND OUT WHICH.
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 67 PERFORMANCE INVESTIGATION ← YOU ARE HERE  │
        │ ★ the method, not the fixes                  │
        └────────────────────┬─────────────────────────┘
                             ▼
              68 CAP · 69 security
              → Phase 8, Phase 9 capstones
```

★ **This is the topic that makes the previous 66 usable.** Indexes, denormalisation, caching, replicas, partitioning, pooling, N+1 — each is correct for exactly one class of problem and harmful for the others. The method is what tells you which one you have.

---

## What is this?

A **fixed sequence** that narrows from "something is slow" to a specific cause, where each step eliminates whole categories.

```
 ★ THE FIVE LEVELS — ALWAYS IN THIS ORDER

 ① ★ IS IT THE DATABASE AT ALL?
    ⇒ compare application-side and database-side time
    ⇒ ★ eliminates: the network, the app, the ORM, GC

 ② ★ SATURATED, CONTENDED, OR QUEUEING?
    ⇒ CPU + wait events + throughput shape
    ⇒ ★ these three have OPPOSITE fixes

 ③ ★ WHICH QUERY?
    ⇒ pg_stat_statements, ★ sorted three different ways

 ④ ★ WHY IS THAT QUERY SLOW?
    ⇒ EXPLAIN (ANALYZE, BUFFERS)

 ⑤ ★ WHAT IS THE CHEAPEST SUFFICIENT FIX?
    ⇒ Topic 54's gates

 ★ THE RULE: NEVER SKIP TO ④. It is the most detailed tool and
   the least useful one when you don't yet know the category.
```

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE WRONG DIAGNOSIS PRODUCES A FIX THAT MAKES IT WORSE,
   AND THE THREE CATEGORIES ARE EASY TO CONFUSE.

 ★ SATURATION  — the machine is genuinely busy
   ⇒ FIX: less work (indexes), or more machine
   ⇒ ★ adding concurrency: helps slightly, then hurts

 ★ CONTENTION  — everyone is waiting for one thing
   ⇒ FIX: remove the shared thing (Topics 45, 48, 61)
   ⇒ ★ adding capacity: NO EFFECT
   ⇒ ★ adding concurrency: ★ MAKES IT WORSE

 ★ QUEUEING    — arrival rate × hold time exceeds capacity
   ⇒ FIX: reduce hold time (Topics 65, 66)
   ⇒ ★ adding concurrency: ★ MAKES IT MUCH WORSE

 ⇒ ★ AND THE SIGNATURES ARE UNAMBIGUOUS:
   saturation: ★ high CPU, high throughput, latency rises with load
   contention: ★ LOW CPU, low throughput, ★ latency rises, CPU FLAT
   queueing:   ★ throughput FLAT, ★ latency LINEAR in concurrency
```

★ And the practical reason: **most incidents are diagnosed by pattern-matching against the last incident**, which is why the same wrong fix gets applied repeatedly. A written method is what stops that.

---

## The physical reality

### Level 1 — is it the database at all?

```
 ★ THE SINGLE MOST USEFUL COMPARISON IN THIS ENTIRE TOPIC:

   application-measured time for the request      ★ 4,200 ms
   sum of database time for that request          ★ 12 ms
   ⇒ ★ 99.7% OF THE TIME IS NOT THE DATABASE.

 ★ WHERE THE OTHER 4,188 ms WENT — five candidates:
   ① ★ N+1 round trips (Topic 66) — DB time is tiny, count is huge
   ② ★ connection-pool wait (Topic 65) — waiting to get a
      connection at all, before any query runs
   ③ an external HTTP call in the handler
   ④ ★ application CPU: JSON serialisation, GC pauses
   ⑤ the network between app and database

 ★ THE INSTRUMENTATION THAT ANSWERS IT — three numbers per request:
   • query COUNT
   • total time INSIDE queries
   • ★ time spent WAITING FOR A POOL CONNECTION
   ⇒ ★ the third is the one nobody measures, and it distinguishes
     ② from everything else.
```

### Level 2 — the three signatures

```
 ★ SATURATION — the machine is doing real work and there is too
   much of it
   CPU              ★ 85–100%, mostly USER time
   throughput       ★ high, and it RISES with offered load until
                      it plateaus
   latency          rises smoothly with load
   wait events      ★ mostly NULL (backends are running)
   ⇒ ★ THE TELL: user CPU is high AND throughput is high.
   ⇒ FIX: do less work per query (an index), or buy more machine.

 ★ CONTENTION — everyone is waiting for one shared object
   CPU              ★ LOW (5–30%), and ★ system time may be high
   throughput       ★ LOW and ★ FLAT — it does not rise with load
   latency          ★ rises while CPU stays flat
   wait events      ★ 'Lock' / 'transactionid' / 'BufferContent' /
                      'LWLock'
   ⇒ ★ THE TELL: ★ IDLE MACHINE, FAILING REQUESTS.
   ⇒ FIX: remove the shared object (Topics 45, 48, 61).
   ⇒ ★ ADDING CAPACITY DOES NOTHING. Adding concurrency is worse.

 ★ QUEUEING — capacity is fine; too many things are holding it
   CPU              ★ moderate, and ★ NOT the limit
   throughput       ★ FLAT as concurrency rises
   latency          ★ LINEAR in concurrency (the giveaway)
   wait events      ★ 'ClientRead' with many 'idle in transaction',
                      or pool waits with no DB-side evidence at all
   ⇒ ★ THE TELL: ★ throughput flat, latency linear.
   ⇒ FIX: reduce hold time (Topics 65, 66).

 ★ THE FOURTH, WHICH LOOKS LIKE SATURATION:
   I/O BOUND
   CPU              ★ low USER, ★ high IOWAIT
   wait events      ★ 'IO' / 'DataFileRead'
   ⇒ FIX: indexes (read less), more RAM (cache more), faster disk.
```

### The one query that classifies the incident

```sql
-- ★ RUN THIS FIRST, EVERY TIME. IT ANSWERS LEVEL 2.
SELECT
  coalesce(wait_event_type, '★ RUNNING (on CPU)') AS wait_type,
  coalesce(wait_event, '-')                       AS wait_event,
  state,
  count(*)
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY 1, 2, 3
ORDER BY 4 DESC;
```
```
 ★ HOW TO READ IT — four patterns:

 ① SATURATION
    ★ RUNNING (on CPU) | -            | active | ★ 28
    ⇒ backends are executing. Check CPU: if ~100%, saturated.

 ② CONTENTION
    Lock               | ★ transactionid | active | ★ 187
    ⇒ ★ ROW LOCK CONTENTION. Topic 61 (hot row) or 48 (deadlock).
    LWLock             | ★ BufferContent | active | ★ 88
    ⇒ ★ shared index leaf page — a different hot spot.
    LWLock             | ★ ProcArray     | active | ★ 22
    ⇒ ★ too many connections (Topic 65).

 ③ QUEUEING
    Client             | ClientRead    | ★ idle in transaction | ★ 188
    ⇒ ★ the APPLICATION is holding transactions open.
      Topic 65's hold-time problem.

 ④ I/O BOUND
    IO                 | ★ DataFileRead | active | ★ 41
    ⇒ reading from disk. Topic 07 / an index.

 ★ AND THE FIFTH, WHICH MEANS "NOT THE DATABASE":
    Client             | ClientRead    | idle   | 412
    (and nothing else)
    ⇒ ★ backends are idle and waiting for the application to send
      something. The problem is upstream.
```

### Level 3 — three different sorts of `pg_stat_statements`

```sql
-- ★ SORT 1: TOTAL TIME — "what consumes the most database capacity?"
SELECT (total_exec_time/1000)::numeric(12,1) AS total_s,
       calls, (mean_exec_time)::numeric(10,3) AS mean_ms,
       substring(query,1,60) AS q
  FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;

-- ★ SORT 2: CALLS — "is there an N+1?"  (Topic 66)
SELECT calls, mean_exec_time::numeric(10,4) AS mean_ms,
       substring(query,1,60) AS q
  FROM pg_stat_statements ORDER BY ★ calls DESC LIMIT 10;

-- ★ SORT 3: VARIANCE — "what is unstable?"  ← ★ THE ONE PEOPLE MISS
SELECT calls, mean_exec_time::numeric(10,2) AS mean_ms,
       stddev_exec_time::numeric(10,2) AS stddev_ms,
       max_exec_time::numeric(10,2) AS max_ms,
       (stddev_exec_time / nullif(mean_exec_time,0))::numeric(6,2) AS cv,
       substring(query,1,50) AS q
  FROM pg_stat_statements
 WHERE calls > 100
 ORDER BY ★ stddev_exec_time DESC LIMIT 10;
```
```
 ★ WHY VARIANCE MATTERS MOST FOR p99 INVESTIGATIONS:
   a query with mean 2 ms and stddev 400 ms is ★ NOT a slow query.
   It is a query that is USUALLY fast and OCCASIONALLY catastrophic.
   ⇒ ★ CV (stddev/mean) > 2 means: a plan flip, lock waits, a
     cache miss cliff, or parameter-dependent selectivity.
   ⇒ ★ MEAN-BASED SORTING HIDES EVERY p99 PROBLEM.

 ★ AND THE BUFFERS COLUMNS TELL YOU IF IT'S I/O:
   shared_blks_hit vs shared_blks_read
   ⇒ read ≫ hit means this query is going to disk.
```

### Level 4 — reading `EXPLAIN (ANALYZE, BUFFERS)`

```
 ★ ALWAYS USE ALL THREE: ANALYZE, BUFFERS, and (PG16+) SETTINGS.
   EXPLAIN (ANALYZE, BUFFERS, SETTINGS, VERBOSE) …

 ★ THE FIVE THINGS TO LOOK FOR, IN ORDER:

 ① ★ ESTIMATED vs ACTUAL ROWS
    (cost=… rows=★ 12) (actual rows=★ 412088)
    ⇒ ★ a 34,000× misestimate. THE PLANNER IS BLIND.
    ⇒ causes: stale statistics · correlated columns (Topic 15) ·
      a function the planner can't estimate · a JOIN of estimates
    ⇒ ★ FIX: ANALYZE · raise statistics target ·
      CREATE STATISTICS for correlated columns

 ② ★ ROWS REMOVED BY FILTER
    Seq Scan … ★ Rows Removed by Filter: 8,204,118
    ⇒ ★ THE MOST COMMON FINDING. A missing or unusable index.

 ③ ★ BUFFERS: hit vs read vs dirtied
    Buffers: shared hit=41,204 ★ read=182,004
    ⇒ read ≫ hit ⇒ I/O bound ⇒ index or RAM
    ⇒ ★ dirtied on a SELECT ⇒ hint bits (Topic 46)

 ④ ★ SORT METHOD AND HASH BATCHES
    ★ Sort Method: external merge  Disk: 412,880kB
    ★ Buckets: 1024  Batches: ★ 64   ⇒ the hash spilled
    ⇒ ★ raise work_mem for this query (SET LOCAL), or reduce rows

 ⑤ ★ LOOPS
    Index Scan … (★ loops=41,204)
    ⇒ ★ actual time is PER LOOP. Multiply it.
      "actual time=0.002..0.003 rows=1 loops=41204"
      = ★ 41,204 × 0.003 = 124 ms, not 0.003 ms.
    ⇒ ★ THIS IS THE MOST MISREAD NUMBER IN EXPLAIN OUTPUT.

 ★ AND FOR A PARAMETERISED QUERY, CHECK THE GENERIC PLAN:
   PG16+: EXPLAIN (GENERIC_PLAN) SELECT … WHERE id = $1;
   ⇒ ★ a query can be fast with your literal and slow with the
     generic plan the server actually uses after 5 executions.
```

### The `auto_explain` extension — for what you cannot reproduce

```ini
# ★ THE MOST UNDERUSED DIAGNOSTIC IN POSTGRESQL
shared_preload_libraries = 'auto_explain,pg_stat_statements'
auto_explain.log_min_duration = '500ms'   # ★ only slow ones
auto_explain.log_analyze = on             # ★ real timings
auto_explain.log_buffers = on
auto_explain.log_nested_statements = on   # ★ inside functions
auto_explain.log_timing = off             # ★ off: much lower overhead
auto_explain.sample_rate = 0.05           # ★ 5% — bounds the cost
```
```
 ★ WHY IT MATTERS: the p99 query you cannot reproduce is the one
   that ran with a different parameter, a different plan, or a cold
   cache. auto_explain captures the plan AS IT ACTUALLY RAN.
 ⇒ ★ log_timing = off removes most of the overhead while still
   giving you row counts and the plan shape — usually enough.
```

### The time-series view — because "slow" is usually "slower than before"

```sql
-- ★ pg_stat_statements is CUMULATIVE. A snapshot tells you about
--   all of history, not about right now.
-- ⇒ ★ SNAPSHOT IT PERIODICALLY AND DIFF.

CREATE TABLE pgss_snapshots AS
  SELECT now() AS taken_at, * FROM pg_stat_statements WITH NO DATA;

-- every 5 minutes:
INSERT INTO pgss_snapshots SELECT now(), * FROM pg_stat_statements;

-- ★ then: what got slower in the last hour vs the same hour
--   yesterday?
WITH now_w AS (
  SELECT queryid, sum(calls) c, sum(total_exec_time) t
    FROM pgss_snapshots WHERE taken_at > now() - interval '1 hour'
   GROUP BY 1),
prev AS (
  SELECT queryid, sum(calls) c, sum(total_exec_time) t
    FROM pgss_snapshots
   WHERE taken_at BETWEEN now() - interval '25 hours'
                      AND now() - interval '24 hours'
   GROUP BY 1)
SELECT n.queryid,
       (n.t/nullif(n.c,0))::numeric(10,2) AS mean_now_ms,
       (p.t/nullif(p.c,0))::numeric(10,2) AS mean_yesterday_ms,
       ((n.t/nullif(n.c,0)) / nullif(p.t/nullif(p.c,0),0))::numeric(8,2)
         AS ★ regression_factor
  FROM now_w n JOIN prev p USING (queryid)
 WHERE n.c > 100
 ORDER BY ★ regression_factor DESC NULLS LAST LIMIT 10;
```
```
 ★ THIS QUERY ANSWERS THE QUESTION PEOPLE ACTUALLY HAVE:
   "what changed?" — which a snapshot can never answer.
```

---

## How it works — step by step

### The investigation runbook

```
 ═══ LEVEL 1 — IS IT THE DATABASE? ═══════════════════════════
 □ app-measured request time
 □ ★ sum of database time for that request
 □ ★ query COUNT for that request        ⇒ >25 ⇒ N+1 (Topic 66)
 □ ★ pool-wait time for that request     ⇒ >0  ⇒ pooling (65)
 ⇒ ★ if DB time ≪ request time, STOP. The database is not the
   problem, and every database-side change will be wasted.

 ═══ LEVEL 2 — CLASSIFY ═════════════════════════════════════
 □ the wait-event query (above)
 □ CPU: user vs system vs iowait
 □ ★ throughput vs concurrency SHAPE
 ⇒ high CPU + high throughput      ⇒ ★ SATURATION
 ⇒ ★ low CPU + Lock waits          ⇒ ★ CONTENTION
 ⇒ ★ flat throughput + linear p99  ⇒ ★ QUEUEING
 ⇒ high iowait + IO waits          ⇒ ★ I/O BOUND

 ═══ LEVEL 3 — WHICH QUERY? ═════════════════════════════════
 □ sort by total_exec_time  ⇒ capacity consumers
 □ ★ sort by calls          ⇒ N+1
 □ ★ sort by stddev         ⇒ p99 / instability
 □ ★ diff against yesterday ⇒ what regressed
 □ check the slow-query log and auto_explain

 ═══ LEVEL 4 — WHY IS IT SLOW? ══════════════════════════════
 □ EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
 □ ★ estimated vs actual rows
 □ ★ Rows Removed by Filter
 □ ★ buffers hit vs read
 □ ★ sort/hash spilling to disk
 □ ★ loops — multiply the per-loop time
 □ ★ the GENERIC plan, if parameterised

 ═══ LEVEL 5 — THE CHEAPEST SUFFICIENT FIX ══════════════════
 ⇒ Topic 54's gates: index → query rewrite → N+1 → work_mem
   → matview → cache → denormalise → replicas → partition → shard
```

### Setting it up before you need it

```sql
-- ★ ① pg_stat_statements — non-negotiable
shared_preload_libraries = 'pg_stat_statements,auto_explain'
pg_stat_statements.max = 10000
pg_stat_statements.track = all          -- ★ include nested statements
pg_stat_statements.track_utility = off

-- ★ ② logging that finds problems without drowning you
log_min_duration_statement = '500ms'
★ log_lock_waits = on                   -- ★ free; Topics 45, 48
deadlock_timeout = '1s'
log_temp_files = 0                      -- ★ ANY spill to disk
log_autovacuum_min_duration = '1s'
log_checkpoints = on
log_connections = off                   -- noisy
★ log_line_prefix = '%m [%p] %q%u@%d app=%a '   -- ★ app name!

-- ★ ③ track I/O timing (small overhead, large payoff)
track_io_timing = on
track_functions = 'pl'
```

```js
// ★ ④ the three per-request numbers
const als = new AsyncLocalStorage();
app.use((req, res, next) =>
  als.run({ q: 0, dbMs: 0, poolWaitMs: 0 }, next));

const origConnect = pool.connect.bind(pool);
pool.connect = async () => {
  const t0 = performance.now();
  const c = await origConnect();
  const ctx = als.getStore();
  if (ctx) ctx.poolWaitMs += performance.now() - t0;   // ★ the missing one
  return c;
};

app.use((req, res, next) => {
  const t0 = performance.now();
  res.on('finish', () => {
    const { q, dbMs, poolWaitMs } = als.getStore() ?? {};
    const total = performance.now() - t0;
    metrics.histogram('req.total_ms', total, { route: req.route?.path });
    metrics.histogram('req.db_ms', dbMs, { route: req.route?.path });
    metrics.histogram('req.pool_wait_ms', poolWaitMs, { route: req.route?.path });
    metrics.histogram('req.query_count', q, { route: req.route?.path });
    // ★ THE RATIO THAT CLASSIFIES INSTANTLY
    metrics.histogram('req.db_fraction', dbMs / total, { route: req.route?.path });
  });
  next();
});
```

### Reproducing a p99 problem

```
 ★ THE HARD PART OF PERFORMANCE WORK IS THAT p99 PROBLEMS DO NOT
   REPRODUCE ON DEMAND. FIVE TECHNIQUES:

 ① ★ auto_explain WITH SAMPLING — capture the real plan when it
    actually happens.
 ② ★ FIND THE PARAMETER. A query fast for one customer and slow
    for another is a selectivity problem.
    SELECT customer_id, count(*) FROM orders GROUP BY 1
     ORDER BY 2 DESC LIMIT 5;
    ⇒ ★ test with the WORST parameter, not a typical one.
 ③ ★ COLD CACHE. Restart, or use pg_prewarm to control what's
    resident, then measure.
 ④ ★ CONCURRENCY. Many p99 problems only exist under load —
    reproduce with pgbench running your actual query mix.
 ⑤ ★ THE GENERIC PLAN. EXPLAIN (GENERIC_PLAN) — a prepared
    statement switches to it after 5 executions and may pick a
    completely different plan.
```

---

## Concept breakdown

```
★ FIVE LEVELS, ALWAYS IN ORDER
   ① is it the DB at all?  ② ★ classify  ③ which query?
   ④ why?  ⑤ cheapest fix
   ⇒ ★ NEVER SKIP TO EXPLAIN. It's the MRI — useless until you
     know which organ.

★ THE THREE SIGNATURES — OPPOSITE FIXES
   SATURATION  ★ high CPU + high throughput ⇒ less work / more machine
   CONTENTION  ★ LOW CPU + Lock waits + flat throughput
               ⇒ ★ remove the shared object; capacity does NOTHING
   QUEUEING    ★ flat throughput + LINEAR latency
               ⇒ ★ reduce hold time; concurrency makes it WORSE
   (+ I/O BOUND ★ high iowait + IO waits ⇒ index / RAM)

★ THE CLASSIFYING QUERY — pg_stat_activity by wait_event
   'transactionid' ⇒ row lock (61/48) · 'BufferContent' ⇒ index leaf
   'ProcArray' ⇒ too many connections (65)
   'ClientRead'+idle in transaction ⇒ application hold time (65)
   'DataFileRead' ⇒ I/O · ★ NULL + active ⇒ actually running

★ pg_stat_statements — SORT IT THREE WAYS
   total_exec_time ⇒ capacity consumers
   ★ calls         ⇒ N+1 (Topic 66)
   ★ stddev        ⇒ ★ p99 / instability — the sort people miss
     ★ CV > 2 = usually fast, occasionally catastrophic
   ★ + DIFF SNAPSHOTS ⇒ "what changed?" — a snapshot can't answer it

★ EXPLAIN — five things, in order
   ① ★ estimated vs actual rows (a misestimate blinds the planner)
   ② ★ Rows Removed by Filter (the most common finding)
   ③ ★ buffers hit vs read
   ④ ★ external merge / hash batches (work_mem)
   ⑤ ★ LOOPS — per-loop time × loops. ★ The most misread number.
   + ★ GENERIC_PLAN for parameterised queries

★ auto_explain — ★ the most underused tool
   captures the plan of the p99 query you cannot reproduce
   ★ log_timing = off + sample_rate bound the overhead

★ REPRODUCING p99
   ★ find the worst PARAMETER · cold cache · under CONCURRENCY ·
   ★ check the GENERIC plan
```

---

## Diagrams

**Diagram 1 — big picture: the three signatures, plotted**

```
        SATURATION                CONTENTION              QUEUEING
   ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
tps│      ●●●●●●●     │ tps│                  │ tps│                  │
   │    ●●            │    │ ●●●●●●●●●●●●●●●● │    │ ●●●●●●●●●●●●●●●● │
   │  ●●              │    │ ★ FLAT and LOW   │    │ ★ FLAT           │
   │●●                │    │                  │    │                  │
   └──────────────────┘    └──────────────────┘    └──────────────────┘
      offered load            offered load            concurrency

   │              ●   │    │              ●   │    │            ●     │
p99│          ●       │ p99│        ●         │ p99│        ●     ★ LINEAR
   │      ●           │    │   ●              │    │    ●             │
   │●●●●              │    │●●                │    │●                 │
   └──────────────────┘    └──────────────────┘    └──────────────────┘

CPU│████████████ ★ 92%│ CPU│██ ★ 8%           │ CPU│█████ ★ 34%       │
   └──────────────────┘    └──────────────────┘    └──────────────────┘

waits  ★ mostly NULL         ★ Lock/LWLock          ★ ClientRead +
       (running)             transactionid           idle in transaction

FIX  ★ less work per query  ★ remove the shared    ★ reduce HOLD TIME
     (index) or a bigger      object (61, 48, 45)    (65, 66)
     machine
BAD  more concurrency:      ★ more capacity:       ★ more concurrency:
FIX  slight gain then loss    NO EFFECT              MUCH WORSE

 ★ THE ONE-LINE DISCRIMINATOR:
   ★ HIGH CPU ⇒ saturation.  ★ LOW CPU + Lock waits ⇒ contention.
   ★ FLAT THROUGHPUT + LINEAR LATENCY ⇒ queueing.
```

**Diagram 2 — data flow: the five levels, and what each eliminates**

```
                    "the endpoint is slow"
                             │
  ┌──────────────────────────▼──────────────────────────────────┐
  │ ★ LEVEL 1 — db_ms / total_ms ?                              │
  │   0.3%  ⇒ ★ NOT THE DATABASE                                │
  │          ⇒ query_count > 25 ⇒ ★ N+1 (66)                    │
  │          ⇒ pool_wait > 0    ⇒ ★ POOLING (65)                │
  │          ⇒ neither          ⇒ ★ app CPU / external call     │
  │   85%   ⇒ continue                                          │
  └──────────────────────────┬──────────────────────────────────┘
      ★ ELIMINATES: the app, the ORM, the network, GC
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ ★ LEVEL 2 — wait events + CPU + throughput shape            │
  │   Lock/transactionid, CPU 8%   ⇒ ★ CONTENTION → 61, 48      │
  │   NULL waits, CPU 92%          ⇒ ★ SATURATION → 3, 4        │
  │   ClientRead + idle-in-txn     ⇒ ★ QUEUEING → 65            │
  │   DataFileRead, iowait 40%     ⇒ ★ I/O → 7, index           │
  └──────────────────────────┬──────────────────────────────────┘
      ★ ELIMINATES: three of the four categories, and their fixes
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ ★ LEVEL 3 — pg_stat_statements, THREE SORTS + a DIFF        │
  │   by total_time ⇒ capacity · by calls ⇒ N+1                 │
  │   ★ by stddev   ⇒ p99      · ★ vs yesterday ⇒ what changed  │
  └──────────────────────────┬──────────────────────────────────┘
      ★ ELIMINATES: every query but one
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ ★ LEVEL 4 — EXPLAIN (ANALYZE, BUFFERS, SETTINGS)            │
  │   estimated vs actual · Rows Removed by Filter · buffers    │
  │   · external merge · ★ LOOPS · GENERIC_PLAN                 │
  └──────────────────────────┬──────────────────────────────────┘
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ ★ LEVEL 5 — Topic 54's gates: the CHEAPEST sufficient fix   │
  └─────────────────────────────────────────────────────────────┘

 ★ NOTE HOW MUCH IS ELIMINATED BEFORE EXPLAIN IS EVEN OPENED.
   ★ Starting at level 4 means running EXPLAIN on a query that
     may not be the problem, in a category where EXPLAIN cannot
     help at all.
```

**Diagram 3 — before/after: the same symptom, three different causes**

```
  SYMPTOM IN ALL THREE CASES: ★ "p99 is 4,200 ms"

 ── CASE A ────────────────────────────────────────────────────
  LEVEL 1  db_ms 3,980 / total 4,200 = ★ 95%   ⇒ it IS the DB
  LEVEL 2  CPU ★ 94% user · waits ★ NULL       ⇒ ★ SATURATION
  LEVEL 3  by total_time: one query, ★ 1,011 h/day
  LEVEL 4  ★ Rows Removed by Filter: 8,204,118 ⇒ Seq Scan
  ★ FIX    ONE INDEX.  8.84 ms → 0.9 ms.
  ✗ WRONG FIXES: replicas · a bigger box · denormalisation

 ── CASE B ────────────────────────────────────────────────────
  LEVEL 1  db_ms 12 / total 4,200 = ★ 0.3%     ⇒ NOT the DB
           ★ query_count = 464
  LEVEL 2  (not reached — level 1 answered it)
  ★ FIX    N+1 (Topic 66).  464 queries → 1.
  ✗ WRONG FIXES: ★ read replicas (the team's plan), indexes,
                 a bigger pool

 ── CASE C ────────────────────────────────────────────────────
  LEVEL 1  db_ms 8 / total 4,200 = ★ 0.2%
           ★ query_count = 4  ★ pool_wait = 4,100 ms
  LEVEL 2  ★ 188 'idle in transaction' + ClientRead
                                              ⇒ ★ QUEUEING
  LEVEL 4  (not needed)
  ★ FIX    move an HTTP call out of the transaction (Topic 65).
           hold time 4,100 ms → 0.8 ms.
  ✗ WRONG FIXES: raise max_connections (★ done twice, no effect)

 ★ IDENTICAL SYMPTOM. THREE CATEGORIES. THREE FIXES.
   ★ AND EACH ONE'S FIX WOULD HAVE BEEN USELESS OR HARMFUL FOR
     THE OTHER TWO.
```

---

## Example 1 — basic

**Set up the tooling.**
```sql
-- postgresql.conf
shared_preload_libraries = 'pg_stat_statements,auto_explain'
pg_stat_statements.track = all
track_io_timing = on
log_min_duration_statement = '500ms'
log_lock_waits = on
log_temp_files = 0
auto_explain.log_min_duration = '500ms'
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_timing = off
auto_explain.sample_rate = 0.05
-- restart
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

**The classifying query.**
```sql
SELECT coalesce(wait_event_type, 'RUNNING') AS wait_type,
       coalesce(wait_event, '-') AS wait_event, state, count(*)
  FROM pg_stat_activity WHERE backend_type='client backend'
 GROUP BY 1,2,3 ORDER BY 4 DESC;
```
```
 wait_type | wait_event | state  | count
-----------+------------+--------+-------
 Client    | ClientRead | idle   |   412
 ★ RUNNING | -          | active |  ★ 28
   ★ 28 backends actually executing. Check CPU next.
```

**Produce and classify each signature. ① Saturation.**
```bash
pgbench -i -s 500 shop && psql -c "DROP INDEX IF EXISTS pgbench_accounts_pkey"
pgbench -c 32 -j 8 -T 30 -S shop & 
sleep 5
psql -c "SELECT coalesce(wait_event_type,'RUNNING') w, count(*)
           FROM pg_stat_activity WHERE state='active' GROUP BY 1"
mpstat 1 3
```
```
    w     | count
----------+-------
 ★ RUNNING|  ★ 31
 %usr  %sys  %iowait  %idle
 ★ 94.2  2.1     0.4     3.3
   ★ HIGH USER CPU + BACKENDS RUNNING = SATURATION.
```

**② Contention.**
```sql
CREATE TABLE counters (id int PRIMARY KEY, n bigint DEFAULT 0);
INSERT INTO counters VALUES (1,0);
```
```bash
echo "UPDATE counters SET n=n+1 WHERE id=1;" > /tmp/hot.sql
pgbench -f /tmp/hot.sql -c 200 -j 8 -T 30 shop &
sleep 5
psql -c "SELECT wait_event_type, wait_event, count(*) FROM pg_stat_activity
          WHERE state='active' GROUP BY 1,2 ORDER BY 3 DESC"
mpstat 1 3
```
```
 wait_event_type |   wait_event    | count
-----------------+-----------------+-------
 Lock            | ★ transactionid |  ★ 187
 %usr  %sys  %iowait  %idle
 ★ 6.8   4.1     0.2  ★ 88.9
   ★ IDLE MACHINE, 187 BACKENDS WAITING = CONTENTION.
   ⇒ ★ Topic 61. A bigger machine would change nothing.
```

**③ Queueing.**
```bash
cat > /tmp/hold.sql <<'EOF'
BEGIN;
SELECT abalance FROM pgbench_accounts WHERE aid = 1;
SELECT pg_sleep(0.05);
COMMIT;
EOF
for c in 25 50 100 200; do
  echo -n "c=$c  "
  pgbench -f /tmp/hold.sql -c $c -j 8 -T 15 shop \
    | grep -E 'tps|latency average' | tr '\n' ' '; echo
done
```
```
 c=25   tps = ★ 494.2   latency average = ★ 50.6 ms
 c=50   tps = ★ 496.1   latency average = ★ 100.8 ms
 c=100  tps = ★ 495.8   latency average = ★ 201.7 ms
 c=200  tps = ★ 494.9   latency average = ★ 404.1 ms
   ★ THROUGHPUT FLAT, LATENCY EXACTLY LINEAR. QUEUEING.
   ⇒ ★ tps = pool_size / hold_time (Topic 65). Adding clients
     adds only queue depth.
```

**④ I/O bound.**
```bash
psql -c "ALTER SYSTEM SET shared_buffers='64MB'" && pg_ctl restart
pgbench -c 16 -j 4 -T 30 -S shop &
sleep 5
psql -c "SELECT wait_event_type, wait_event, count(*) FROM pg_stat_activity
          WHERE state='active' GROUP BY 1,2 ORDER BY 3 DESC LIMIT 3"
iostat -x 1 3 | tail -5
```
```
 wait_event_type |  wait_event   | count
-----------------+---------------+-------
 IO              | ★ DataFileRead|  ★ 14
 %usr  %sys  ★ %iowait  %idle
  8.2   3.1     ★ 61.4   27.3
   ★ HIGH IOWAIT + IO WAITS = I/O BOUND.
```

**The three sorts of `pg_stat_statements`.**
```sql
SELECT pg_stat_statements_reset();
-- run a mixed workload, then:

-- ★ by total time
SELECT (total_exec_time/1000)::numeric(10,1) AS total_s, calls,
       mean_exec_time::numeric(10,3) AS mean_ms, substring(query,1,45) AS q
  FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 3;
```
```
 total_s |  calls  | mean_ms |                q
---------+---------+---------+-------------------------------
 ★ 412.8 |   88420 |   4.669 | SELECT … FROM orders o JOIN …
    88.2 | 4102884 |   0.021 | SELECT … FROM customers WHERE
```
```sql
-- ★ by calls
SELECT calls, mean_exec_time::numeric(10,4) AS mean_ms, substring(query,1,45)
  FROM pg_stat_statements ORDER BY calls DESC LIMIT 3;
```
```
   calls   | mean_ms |                substring
-----------+---------+-------------------------------
 ★ 4102884 |  0.0210 | SELECT … FROM customers WHERE
   ★ 4 MILLION calls of a 0.02 ms query ⇒ N+1 (Topic 66).
```
```sql
-- ★ by variance — the p99 sort
SELECT calls, mean_exec_time::numeric(10,2) AS mean_ms,
       stddev_exec_time::numeric(10,2) AS stddev_ms,
       max_exec_time::numeric(10,2) AS max_ms,
       (stddev_exec_time/nullif(mean_exec_time,0))::numeric(6,2) AS cv,
       substring(query,1,40) AS q
  FROM pg_stat_statements WHERE calls > 100
 ORDER BY stddev_exec_time DESC LIMIT 3;
```
```
 calls | mean_ms | stddev_ms | max_ms |  cv   |            q
-------+---------+-----------+--------+-------+--------------------------
 41208 |  ★ 2.11 |  ★ 412.88 | ★ 8842 |★195.7 | SELECT … WHERE status = $1
   ★ MEAN 2 ms, MAX 8.8 SECONDS, CV 196.
   ⇒ ★ INVISIBLE IN A MEAN-BASED SORT. This is your p99.
```

**Read `EXPLAIN` properly — the five things.**
```sql
EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
SELECT o.id, c.name FROM orders o JOIN customers c ON c.id = o.customer_id
 WHERE o.status = 'pending' ORDER BY o.created_at DESC LIMIT 50;
```
```
 Limit  (actual time=2884.1..2884.3 rows=50 loops=1)
   Buffers: shared ★ hit=41,204 read=182,004
   ->  Sort  (actual time=2884.1..2884.2 rows=50)
         ★ Sort Method: external merge  Disk: 412,880kB
         ->  Hash Join  (cost=… ★ rows=12) (★ actual rows=412,088)
               ->  Seq Scan on orders
                     Filter: (status = 'pending'::text)
                     ★ Rows Removed by Filter: 8,204,118
               ->  Hash  (actual rows=50000)
                     ★ Buckets: 4096  Batches: 16  Memory: 4,102kB
 Settings: work_mem = '4MB'
 Execution Time: ★ 2,884.9 ms
```
```
 ★ ALL FIVE SIGNALS PRESENT:
 ① ★ estimated 12, actual 412,088 — a 34,000× misestimate
 ② ★ Rows Removed by Filter: 8.2M — a missing index
 ③ ★ read=182,004 ≫ hit=41,204 — I/O bound
 ④ ★ external merge 412 MB + 16 hash batches — work_mem too small
 ⑤ loops=1 here, but check it on nested nodes
```
```sql
-- ★ fix ① and ② with one index
CREATE INDEX CONCURRENTLY ON orders (status, created_at DESC)
  WHERE status = 'pending';
ANALYZE orders;
EXPLAIN (ANALYZE, BUFFERS) /* same query */;
```
```
 Limit  (actual time=0.084..0.412 rows=50)
   Buffers: shared hit=204
   ->  Nested Loop
         ->  ★ Index Scan using orders_status_created_idx (rows=50)
         ->  Index Scan using customers_pkey (loops=50)
 Execution Time: ★ 0.482 ms       ★ 5,985×
```

**The loops trap.**
```sql
EXPLAIN (ANALYZE)
SELECT o.id, (SELECT count(*) FROM order_items i WHERE i.order_id=o.id)
  FROM orders o LIMIT 41204;
```
```
   SubPlan 1
     ->  Aggregate (actual time=★ 0.003..0.003 rows=1 ★ loops=41204)
 Execution Time: ★ 148.2 ms
   ★ "0.003 ms" LOOKS INSTANT. It is PER LOOP.
   ★ 41,204 × 0.003 = 124 ms — 84% of the total.
   ⇒ ★ ALWAYS MULTIPLY BY loops.
```

**The generic-plan trap.**
```sql
PREPARE p(text) AS SELECT * FROM orders WHERE status = $1;
EXPLAIN (ANALYZE) EXECUTE p('cancelled');   -- 1 of 8M rows
```
```
 ★ Index Scan using orders_status_idx  (actual rows=1)
 Execution Time: 0.041 ms
```
```sql
-- ★ after 5 executions the planner may switch to a generic plan
EXPLAIN (GENERIC_PLAN) SELECT * FROM orders WHERE status = $1;
```
```
 ★ Seq Scan on orders  (cost=0.00..204882.00 rows=1,640,822)
   ★ THE GENERIC PLAN IS A SEQ SCAN, because it must assume an
     average selectivity across all values.
 ⇒ ★ your literal-value EXPLAIN was fast and the server is slow.
```

**See `auto_explain` capture what you can't reproduce.**
```bash
grep -A 20 'duration:' /var/log/postgresql/postgresql.log | tail -30
```
```
 LOG:  duration: 2884.912 ms  plan:
   Query Text: SELECT … WHERE customer_id = $1 AND created_at >= $2
   Limit  (actual rows=50 loops=1)
     ->  Sort  (★ Sort Method: external merge  Disk: 88,204kB)
           ->  ★ Seq Scan on orders (★ Rows Removed by Filter: 4,102,884)
   ★ THE ACTUAL PLAN FROM THE ACTUAL SLOW EXECUTION, with the
     actual parameters. This is what you cannot get any other way.
```

**Snapshot and diff.**
```sql
CREATE TABLE pgss_snap AS SELECT now() AS t, * FROM pg_stat_statements
  WITH NO DATA;
-- cron: */5 * * * *
INSERT INTO pgss_snap SELECT now(), * FROM pg_stat_statements;

-- ★ what regressed?
WITH a AS (SELECT queryid, sum(calls) c, sum(total_exec_time) t
             FROM pgss_snap WHERE t > now()-interval '1 hour' GROUP BY 1),
     b AS (SELECT queryid, sum(calls) c, sum(total_exec_time) t
             FROM pgss_snap
            WHERE t BETWEEN now()-interval '25 hours'
                        AND now()-interval '24 hours' GROUP BY 1)
SELECT a.queryid, (a.t/a.c)::numeric(10,2) AS now_ms,
       (b.t/b.c)::numeric(10,2) AS yesterday_ms,
       ((a.t/a.c)/(b.t/b.c))::numeric(8,2) AS ★ factor
  FROM a JOIN b USING (queryid) WHERE a.c > 100
 ORDER BY factor DESC LIMIT 3;
```
```
  queryid   | now_ms | yesterday_ms | factor
------------+--------+--------------+--------
 -884201188 | ★ 412.8|         2.11 | ★ 195.6
   ★ THIS QUERY GOT 196× SLOWER SINCE YESTERDAY.
     A snapshot could never have told you that.
```

---

## Example 2 — production scenario

**The situation.** A marketplace. A Slack message at 11:04:

> *"Checkout is slow. Can someone look at the database?"*

```
 WHAT IS ACTUALLY KNOWN
   ★ nothing else.
 ★ AND THE TEAM'S IMMEDIATE INSTINCT: "it's probably the orders
   table again — let's add an index."
   ⇒ ★ THAT INSTINCT IS WHY THIS TOPIC EXISTS.
```

**Level 1 — is it the database?**

```
 ★ FROM THE APPLICATION'S OWN METRICS (the three numbers):
   route              /api/checkout
   total_ms p99       ★ 6,400 ms
   db_ms p99          ★ 84 ms
   ★ pool_wait_ms p99 ★ 6,180 ms
   query_count p99    ★ 6
```
```
 ★ 1.3% OF THE TIME IS SPENT IN QUERIES.
 ★ 96.6% IS SPENT WAITING TO GET A CONNECTION AT ALL.
 ⇒ ★ THE QUERIES ARE FINE. THERE ARE NO SPARE CONNECTIONS.
 ⇒ ★ AN INDEX ON `orders` WOULD HAVE CHANGED NOTHING.
 ⇒ ★ AND: query_count is 6, so this is NOT an N+1.
```

**Level 2 — classify. Who is holding the connections?**

```sql
SELECT coalesce(wait_event_type,'RUNNING') AS w, wait_event, state, count(*)
  FROM pg_stat_activity WHERE backend_type='client backend'
 GROUP BY 1,2,3 ORDER BY 4 DESC LIMIT 6;
```
```
     w     |  wait_event   |        state        | count
-----------+---------------+---------------------+-------
 Client    | ClientRead    | idle                |   ★ 41
 ★ RUNNING | -             | active              |  ★ 178
 Lock      | transactionid | active              |     4
```
```bash
mpstat 1 3
```
```
 %usr  %sys  %iowait  %idle
 ★ 96.1  2.2     0.3     1.4
```
```
 ★ 178 BACKENDS RUNNING, 96% USER CPU.
 ⇒ ★ SATURATION, NOT CONTENTION.
   The connections aren't blocked — they are all genuinely working.
 ⇒ ★ SO: something is consuming enormous CPU, and checkout is
   queued behind it.
 ⇒ ★ CHECKOUT IS A VICTIM, NOT THE CAUSE. This is the key
   realisation, and it inverts the investigation.
```

**Level 3 — which query is burning the CPU?**

```sql
-- ★ what is running RIGHT NOW, longest first
SELECT pid, application_name, now()-query_start AS running,
       substring(query,1,70) AS q
  FROM pg_stat_activity WHERE state='active' AND backend_type='client backend'
 ORDER BY query_start LIMIT 5;
```
```
  pid  | application_name |   running    |                    q
-------+------------------+--------------+----------------------------------
 41202 | ★ analytics-job  | ★ 00:41:18   | SELECT seller_id, date_trunc('d'…
 41288 | ★ analytics-job  | ★ 00:38:02   | SELECT seller_id, date_trunc('d'…
 41304 | ★ analytics-job  | ★ 00:37:44   | SELECT seller_id, date_trunc('d'…
   ★ THREE COPIES OF THE SAME 40-MINUTE ANALYTICS QUERY.
```
```sql
-- ★ and the diff against yesterday
WITH a AS (SELECT queryid, sum(calls) c, sum(total_exec_time) t
             FROM pgss_snap WHERE t > now()-interval '2 hours' GROUP BY 1),
     b AS (SELECT queryid, sum(calls) c, sum(total_exec_time) t
             FROM pgss_snap WHERE t BETWEEN now()-interval '26 hours'
                                        AND now()-interval '24 hours' GROUP BY 1)
SELECT (a.t/1000/60)::numeric(10,1) AS minutes_now,
       (b.t/1000/60)::numeric(10,1) AS minutes_yesterday,
       a.c AS calls_now, b.c AS calls_yesterday,
       substring(s.query,1,50) AS q
  FROM a JOIN b USING (queryid)
  JOIN pg_stat_statements s ON s.queryid = a.queryid
 ORDER BY a.t - b.t DESC LIMIT 3;
```
```
 minutes_now | minutes_yesterday | calls_now | calls_yesterday |    q
-------------+-------------------+-----------+-----------------+---------------
     ★ 118.4 |             ★ 4.2 |      ★ 47 |            ★ 12 | SELECT seller_
```
```
 ★ THE ANALYTICS QUERY WENT FROM 4 MINUTES/DAY TO 118 MINUTES IN
   TWO HOURS, AND FROM 12 CALLS TO 47.
 ⇒ ★ TWO CHANGES: it got slower AND it is running more often.
```

**Level 4 — why did it get slower?**

```bash
grep -B2 -A 25 'analytics-job' /var/log/postgresql/postgresql.log \
  | grep -A 25 'duration:' | tail -35
```
```
 LOG:  duration: 2,481,204 ms  plan:
   Query Text: SELECT seller_id, date_trunc('day', created_at), count(*),
               sum(total_minor) FROM orders WHERE created_at >= $1
               GROUP BY 1,2
   GroupAggregate  (actual rows=1,204,882 loops=1)
     ->  Sort  (actual rows=41,204,882 loops=1)
           Sort Key: seller_id, (date_trunc('day', created_at))
           ★ Sort Method: external merge  Disk: 8,204,880kB
           ->  Seq Scan on orders  (cost=… ★ rows=412,088)
                                   (★ actual rows=41,204,882)
                 Filter: (created_at >= $1)
                 ★ Rows Removed by Filter: 2,104
```
```
 ★ THREE FINDINGS:
 ① ★ ESTIMATED 412,088, ACTUAL 41,204,882 — a 100× misestimate.
    ⇒ the planner chose a sort-based GroupAggregate because it
      thought the input was small.
 ② ★ 8.2 GB SPILLED TO DISK in an external merge sort.
 ③ ★ "Rows Removed by Filter: 2,104" — the WHERE clause removes
    almost nothing. ⇒ ★ $1 IS EFFECTIVELY 'ALL TIME'.
```
```sql
-- ★ why the misestimate?
SELECT last_analyze, last_autoanalyze, n_live_tup, n_mod_since_analyze
  FROM pg_stat_user_tables WHERE relname='orders';
```
```
     last_analyze     |  last_autoanalyze   | n_live_tup | n_mod_since_analyze
----------------------+---------------------+------------+---------------------
                      | 2026-07-12 04:11:02 |  ★ 412,088 |       ★ 40,882,204
   ★ STATISTICS ARE SIX WEEKS OLD. The table has grown 100×
     since, and autoanalyze has not run.
```
```sql
-- ★ WHY hasn't autoanalyze run?
SELECT 'long txn' AS src, pid::text, (now()-xact_start)::text AS age
  FROM pg_stat_activity WHERE xact_start IS NOT NULL
UNION ALL
SELECT 'repl slot', slot_name, active::text FROM pg_replication_slots
UNION ALL
SELECT '2pc', gid, (now()-prepared)::text FROM pg_prepared_xacts
ORDER BY 3 DESC LIMIT 5;
```
```
    src    |     pid      |      age
-----------+--------------+----------------
 long txn  | ★ 41202      | ★ 00:41:18
 long txn  | ★ 41288      | ★ 00:38:02
   ★ THE ANALYTICS QUERIES THEMSELVES are the long transactions.
   ⇒ ★ A FEEDBACK LOOP: stale stats ⇒ a bad plan ⇒ a 40-minute
     query ⇒ which blocks autoanalyze ⇒ ⇒ stats stay stale.
```

**Level 5 — the fixes, in order of immediacy.**

```sql
-- ★ ① IMMEDIATE: stop the bleeding. Cancel the runaway queries.
SELECT pg_cancel_backend(pid) FROM pg_stat_activity
 WHERE application_name = 'analytics-job' AND now()-query_start > interval '5 min';
-- ⇒ ★ checkout p99 6,400 ms → 180 ms within 20 seconds.
```

```sql
-- ★ ② FIX THE STATISTICS — the root of the bad plan
ANALYZE orders;
SELECT last_analyze, n_live_tup FROM pg_stat_user_tables WHERE relname='orders';
```
```
     last_analyze      | n_live_tup
-----------------------+-------------
 2026-08-25 11:22:04+0 | ★ 41,204,882
```
```sql
-- ★ and make it not happen again on a table this large
ALTER TABLE orders SET (
  autovacuum_analyze_scale_factor = 0.02,   -- ★ was 0.1 ⇒ 4.1M rows
  autovacuum_analyze_threshold    = 50000,
  autovacuum_vacuum_cost_delay    = 0);
-- ★ default 0.1 on a 41M-row table means waiting for 4.1 MILLION
--   modifications before analysing. On a fast-growing table that
--   is far too late.
```

```sql
-- ★ ③ THE QUERY ITSELF — it should never scan 41M rows
-- (a) an index that supports the range and the grouping
CREATE INDEX CONCURRENTLY ON orders (created_at, seller_id)
  INCLUDE (total_minor);

-- (b) ★ but the real fix: it's a daily rollup (Topic 56)
CREATE TABLE seller_daily_rollup (
  seller_id bigint NOT NULL, day date NOT NULL,
  orders bigint NOT NULL, revenue_minor bigint NOT NULL,
  PRIMARY KEY (seller_id, day)
) PARTITION BY RANGE (day);
-- ⇒ ★ incremental: process only the last 2 days
-- ⇒ 2,481 s → ★ 4.2 s
```

```sql
-- ★ ④ ISOLATE THE WORKLOAD so analytics can never do this again
ALTER ROLE analytics_job SET statement_timeout = '300s';
ALTER ROLE analytics_job SET idle_in_transaction_session_timeout = '60s';
ALTER ROLE analytics_job SET default_transaction_read_only = on;
-- ★ and route it to the analytics replica (Topic 58)
```

```ini
# ★ ⑤ a separate PgBouncer pool so it cannot consume checkout's
#   connections (Topic 65)
[databases]
shop           = host=pg-primary dbname=shop pool_size=48
shop_analytics = host=pg-analytics-replica dbname=shop pool_size=6
```
```
 ★ THIS IS THE STRUCTURAL FIX: a bounded pool means an analytics
   runaway can consume at most 6 connections, not 178.
```

```js
// ★ ⑥ AND WHY WERE THERE THREE COPIES RUNNING?
// the scheduler had no overlap guard. Each 40-minute run started
// while the previous was still going.
await withTransaction(async (tx) => {
  const { rows:[{ got }] } = await tx.query(
    'SELECT pg_try_advisory_xact_lock(hashtext($1)) AS got', ['seller-rollup']);
  if (!got) { metrics.increment('job.skipped_overlap'); return; }
  await runRollup(tx);
});
// ★ Topic 45. One line, and the pile-up becomes impossible.
```

**Step 6 — the alerts that would have caught it hours earlier.**

```sql
-- ★ ① any query running longer than 5 minutes
SELECT count(*) FROM pg_stat_activity
 WHERE state='active' AND now()-query_start > interval '5 minutes';

-- ★ ② stale statistics on large tables — ★ the leading indicator
SELECT relname, n_live_tup, n_mod_since_analyze,
       round(100.0*n_mod_since_analyze/nullif(n_live_tup,0),1) AS pct_stale,
       last_autoanalyze
  FROM pg_stat_user_tables
 WHERE n_live_tup > 1000000
   AND n_mod_since_analyze > n_live_tup * 0.2;
-- ★ alert: any row

-- ★ ③ temp-file spills (log_temp_files = 0)
SELECT datname, temp_files, pg_size_pretty(temp_bytes) FROM pg_stat_database
 WHERE temp_bytes > 0;
-- ★ alert: temp_bytes growing rapidly

-- ★ ④ pool wait time, from the application
-- ★ alert: p99 pool_wait_ms > 100
```

**Step 7 — results.**

| | Before | After |
|---|---|---|
| Checkout p99 | 6,400 ms | **180 ms** |
| Checkout `db_ms` p99 | 84 ms | 78 ms *(barely changed)* |
| ★ Checkout `pool_wait` p99 | ★ **6,180 ms** | **2 ms** |
| Analytics query | 2,481 s | **4.2 s** (rollup) |
| `orders` statistics age | ★ 6 weeks | < 1 hour |
| Concurrent analytics runs | ★ 3 | **1** (advisory lock) |
| Analytics connection ceiling | ★ 178 | **6** (separate pool) |
| Time to diagnose | — | ★ **11 minutes**, by method |

```
 ★ SEVEN LESSONS:
 ① ★ THE THREE PER-REQUEST NUMBERS ANSWERED LEVEL 1 IMMEDIATELY.
   db_ms 84, pool_wait 6,180. The queries were never the problem,
   and the team's instinct (add an index to `orders`) would have
   changed nothing.
 ② ★ CHECKOUT WAS A VICTIM, NOT THE CAUSE. The wait-event query
   showed 178 backends genuinely RUNNING at 96% CPU — saturation,
   not contention. That inverted the whole investigation.
 ③ ★ THE DIFF AGAINST YESTERDAY FOUND IT. 4 minutes/day → 118
   minutes in two hours. A snapshot of pg_stat_statements could
   never have shown that.
 ④ ★ auto_explain GAVE THE PLAN OF A QUERY NOBODY COULD
   REPRODUCE — including the 8.2 GB spill and the 100× misestimate.
 ⑤ ★ IT WAS A FEEDBACK LOOP: stale statistics ⇒ a bad plan ⇒ a
   40-minute query ⇒ which blocked autoanalyze ⇒ stats stayed stale.
 ⑥ ★ THE DEFAULT autovacuum_analyze_scale_factor OF 0.1 MEANS
   WAITING FOR 4.1 MILLION MODIFICATIONS on a 41M-row table.
 ⑦ ★ THE STRUCTURAL FIX WAS A SEPARATE POOL AND AN ADVISORY LOCK,
   not an index. A bounded pool means a runaway can consume 6
   connections instead of 178.
```

---

## Common mistakes

**1. Starting with `EXPLAIN`.**
- *Symptom:* a perfect plan for a query that isn't the problem, in a category where `EXPLAIN` cannot help.
- *Fix:* levels 1 and 2 first. They eliminate most of the search space in two queries.

**2. Not measuring pool-wait time separately.**
- *Symptom:* `db_ms` looks fine and the endpoint is slow; the cause is invisible.
- *Fix:* instrument `pool.connect()`. It's the number that distinguishes queueing from everything else.

**3. Confusing contention with saturation.**
- *Symptom:* a bigger instance is provisioned and nothing changes.
- *Fix:* low CPU + `Lock` waits is contention. Capacity has no effect.

**4. Sorting `pg_stat_statements` only by total time.**
- *Symptom:* the N+1 and the high-variance p99 query are both invisible.
- *Fix:* three sorts — total time, `calls`, `stddev`.

**5. Ignoring variance.**
- *Symptom:* a query with mean 2 ms and max 8.8 s never appears in any report.
- *Fix:* sort by `stddev_exec_time`; `CV > 2` means unstable.

**6. Treating `pg_stat_statements` as a point-in-time view.**
- *Symptom:* you can see what is expensive but not what *changed*.
- *Fix:* snapshot every 5 minutes and diff against the same window yesterday.

**7. Misreading `loops`.**
- *Symptom:* "the inner node takes 0.003 ms" — with `loops=41204`, it's 124 ms.
- *Fix:* always multiply per-loop time by `loops`.

**8. `EXPLAIN`ing with a literal when the app uses a parameter.**
- *Symptom:* fast in `psql`, slow in production.
- *Fix:* `EXPLAIN (GENERIC_PLAN)`, and test with the *worst* parameter value.

**9. Not enabling `auto_explain`.**
- *Symptom:* the p99 plan can never be captured because it can't be reproduced.
- *Fix:* `log_min_duration` + `sample_rate` + `log_timing = off`.

**10. Ignoring statistics freshness.**
- *Symptom:* a 100× misestimate producing a catastrophic plan, on a table that has grown.
- *Fix:* alert on `n_mod_since_analyze > 20%` of `n_live_tup`; lower `autovacuum_analyze_scale_factor` on large tables.

**11. Fixing the symptom rather than the loop.**
- *Symptom:* cancelling the query fixes it until tomorrow.
- *Fix:* find the feedback loop — here, a long query blocking the autoanalyze that would have prevented it.

**12. No workload isolation.**
- *Symptom:* one analytics job consumes every connection and takes down checkout.
- *Fix:* separate pools, per-role `statement_timeout`, a dedicated replica.

---

## Hands-on proof

**PROVE IT #1–#12 — Example 1** (all four signatures produced deliberately and classified by wait events + CPU, queueing showing exactly-linear latency with flat throughput, the three `pg_stat_statements` sorts each revealing something the others hide, an `EXPLAIN` with all five signals, the loops trap at 41,204 × 0.003 ms, the generic-plan trap, `auto_explain` capturing an unreproducible plan, and the snapshot diff finding a 196× regression).

**PROVE IT #13 — the db-fraction ratio classifies instantly.**
```js
// three routes, same p99, three categories
//   /a: db_ms/total = 0.95  ⇒ ★ the database
//   /b: db_ms/total = 0.003, query_count = 464 ⇒ ★ N+1
//   /c: db_ms/total = 0.002, pool_wait = 4,100 ⇒ ★ queueing
```

**PROVE IT #14 — statistics staleness produces the misestimate.**
```sql
CREATE TABLE t AS SELECT g AS id, g % 100 AS k FROM generate_series(1,1000) g;
ANALYZE t;
INSERT INTO t SELECT g, g % 100 FROM generate_series(1001, 10000000) g;
-- ★ do NOT analyze
EXPLAIN SELECT * FROM t WHERE k = 5;
```
```
 Seq Scan on t  (cost=… ★ rows=10)      ← ★ estimate from 1,000 rows
```
```sql
ANALYZE t;
EXPLAIN SELECT * FROM t WHERE k = 5;
```
```
 ... (cost=… ★ rows=100,102)             ← ★ 10,000× different
   ⇒ ★ and the PLAN CHOICE changes with it.
```

**PROVE IT #15 — the autoanalyze feedback loop.**
```sql
-- session 1: a long transaction
BEGIN ISOLATION LEVEL REPEATABLE READ; SELECT 1 FROM t LIMIT 1;
-- session 2
UPDATE t SET k = k + 1;
SELECT relname, n_mod_since_analyze, last_autoanalyze
  FROM pg_stat_user_tables WHERE relname='t';
```
```
 relname | n_mod_since_analyze | last_autoanalyze
---------+---------------------+------------------
 t       |         ★ 10000000  | ★ (null)
   ★ autoanalyze cannot make progress while the snapshot is held.
```

**PROVE IT #16 — `track_io_timing` separates CPU from I/O.**
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM orders;
```
```
   Buffers: shared read=182,004
   ★ I/O Timings: read=2,104.882 ms
 Execution Time: 2,884.9 ms
   ★ 73% of the query was WAITING FOR DISK, not computing.
   ⇒ ★ without track_io_timing you cannot tell these apart.
```

---

## The design decision framework

```
★★★ CLASSIFY BEFORE YOU DIAGNOSE. DIAGNOSE BEFORE YOU FIX. ★★★

 ① ★ LEVEL 1 — IS IT THE DATABASE?  (three numbers per request)
    db_ms · ★ pool_wait_ms · query_count
    ⇒ db_ms/total < 10%     ⇒ ★ NOT THE DATABASE. Stop.
      · query_count > 25    ⇒ ★ N+1 (66)
      · pool_wait > 0       ⇒ ★ POOLING (65)
      · neither             ⇒ app CPU / an external call
    ⇒ ★ THIS ELIMINATES MOST INVESTIGATIONS IN ONE STEP.

 ② ★ LEVEL 2 — CLASSIFY. ONE QUERY + CPU.
    ★ high CPU + RUNNING waits    ⇒ SATURATION → less work / more machine
    ★ LOW CPU + Lock waits        ⇒ CONTENTION → 61, 48, 45
                                    ★ capacity has NO effect
    ★ flat throughput + linear p99 ⇒ QUEUEING → 65, 66
                                    ★ concurrency makes it WORSE
    ★ high iowait + IO waits      ⇒ I/O → index, RAM, disk
    ⇒ ★ THE THREE HAVE OPPOSITE FIXES. Getting this wrong is why
      the same wrong fix gets applied repeatedly.

 ③ ★ LEVEL 3 — WHICH QUERY? THREE SORTS AND A DIFF.
    total_exec_time ⇒ capacity consumers
    ★ calls         ⇒ N+1
    ★ stddev / CV   ⇒ ★ p99 instability — mean-based sorting hides
                      every p99 problem
    ★ snapshot diff vs yesterday ⇒ ★ "what changed?" — the question
                      people actually have, and the one a snapshot
                      cannot answer
    + the slow-query log and ★ auto_explain

 ④ ★ LEVEL 4 — WHY? EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
    ① ★ estimated vs actual rows  ⇒ stale stats / correlation (15)
    ② ★ Rows Removed by Filter    ⇒ the most common finding
    ③ ★ buffers hit vs read + I/O Timings
    ④ ★ external merge / hash batches ⇒ work_mem
    ⑤ ★ LOOPS — multiply. ★ The most misread number in EXPLAIN.
    + ★ GENERIC_PLAN, and test the WORST parameter

 ⑤ ★ LEVEL 5 — THE CHEAPEST SUFFICIENT FIX (Topic 54's gates)
    index → query rewrite → fix N+1 → work_mem → matview →
    cache → denormalise → replica → partition → shard

 ★ SET IT UP BEFORE YOU NEED IT
    pg_stat_statements (track=all) · ★ auto_explain (sampled) ·
    ★ track_io_timing · log_lock_waits · log_temp_files=0 ·
    ★ log_line_prefix with %a · ★ pgss snapshots every 5 min ·
    ★ the three per-request numbers

 ★ LOOK FOR FEEDBACK LOOPS
    a long query blocks autoanalyze ⇒ stats stay stale ⇒ the plan
    stays bad ⇒ the query stays long.
    ⇒ ★ cancelling the query fixes today. Breaking the loop fixes
      it permanently.

 ★ ISOLATE WORKLOADS SO ONE CANNOT STARVE ANOTHER
    separate pools · per-role statement_timeout · a dedicated
    analytics replica · ★ an advisory lock so jobs cannot overlap
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Deliberately produce all four signatures on your own machine: saturation (drop an index and run `pgbench -S`), contention (a single hot counter row), queueing (a `pg_sleep` inside a transaction at increasing concurrency), and I/O bound (tiny `shared_buffers`). For each, capture the wait-event query and `mpstat`, and write the one-line discriminator.

### Exercise 2 — medium (apply it)
Set up `pg_stat_statements`, `auto_explain`, and a 5-minute snapshot table. Run a mixed workload, then produce all three sorts and explain what each reveals that the others hide.

Then construct a query with `mean 2 ms, max 5 s` (hint: parameter-dependent selectivity) and show that it is invisible in a mean-based report but obvious in a variance-based one.

### Exercise 3 — hard (production simulation)
"Checkout is slow. Can someone look at the database?" is all you have. The team wants to add an index to `orders`.

(a) Give the three per-request numbers you'd ask for first, and explain what each rules in or out. Given `db_ms 84 / pool_wait 6,180 / query_count 6`, what have you eliminated?
(b) Run the classification. 178 backends `RUNNING` at 96% CPU — which category, and why does that invert the investigation?
(c) Find the cause. Show the query for "what is running now" and the diff-against-yesterday query, and interpret both.
(d) Explain why `auto_explain` was necessary here and what a fresh `EXPLAIN` would have shown instead.
(e) The plan shows `estimated 412,088 / actual 41,204,882`. Trace the cause back two steps.
(f) Identify the feedback loop and explain why cancelling the query is not a fix.
(g) Give the immediate mitigation, the root-cause fix, and the structural fix — and say which of the three prevents recurrence.
(h) `autovacuum_analyze_scale_factor` is 0.1. Compute what that means for a 41M-row table and give the correct setting.
(i) Three copies of the job were running. Give the one-line fix.
(j) Write the four alerts, and say which is the leading indicator.
(k) Write the one-paragraph explanation of why "add an index to `orders`" would have changed nothing.

---

## Mental model checkpoint

1. Name the five levels in order. Why is `EXPLAIN` last?
2. Give the three per-request numbers and what each distinguishes.
3. Give the signature of saturation, contention and queueing on CPU, throughput and latency.
4. Why does adding capacity help saturation and do nothing for contention?
5. Why does adding concurrency make queueing much worse?
6. Name the three ways to sort `pg_stat_statements` and what each finds.
7. Why does mean-based sorting hide every p99 problem?
8. Why must you snapshot and diff `pg_stat_statements`?
9. Name the five things to look for in `EXPLAIN`, in order.
10. What does `loops=41204` do to a reported per-node time?
11. Why can a query be fast in `psql` and slow in production?
12. What is `auto_explain` for, and what two settings bound its cost?
13. Describe the stale-statistics feedback loop.

---

## Quick reference card

**★ Level 1 — three numbers per request:** `db_ms` · ★ `pool_wait_ms` · `query_count`.
`db_ms/total < 10%` ⇒ **not the database**. `query_count > 25` ⇒ **N+1**. `pool_wait > 0` ⇒ **pooling**.

**★ Level 2 — the classifying query**
```sql
SELECT coalesce(wait_event_type,'RUNNING'), wait_event, state, count(*)
  FROM pg_stat_activity WHERE backend_type='client backend'
 GROUP BY 1,2,3 ORDER BY 4 DESC;
```

| | CPU | throughput | latency | waits | fix |
|---|---|---|---|---|---|
| **saturation** | ★ high | high | rises | NULL | less work / more machine |
| **contention** | ★ **low** | ★ flat, low | rises | ★ `Lock` | ★ remove the shared object |
| **queueing** | moderate | ★ **flat** | ★ **linear** | `ClientRead` + idle-in-txn | ★ reduce hold time |
| **I/O** | ★ iowait | low | rises | `DataFileRead` | index / RAM |

**★ Level 3 — three sorts + a diff**
```sql
ORDER BY total_exec_time DESC   -- capacity
ORDER BY ★ calls DESC           -- N+1
ORDER BY ★ stddev_exec_time DESC -- p99 (CV = stddev/mean > 2 = unstable)
-- ★ + snapshot every 5 min and diff vs the same window yesterday
```

**★ Level 4 — `EXPLAIN (ANALYZE, BUFFERS, SETTINGS)`**
① estimated vs actual · ② `Rows Removed by Filter` · ③ `hit` vs `read` + I/O Timings · ④ `external merge` / hash `Batches` · ⑤ ★ **multiply by `loops`** · + `GENERIC_PLAN`.

**Set up before you need it**
```ini
shared_preload_libraries = 'pg_stat_statements,auto_explain'
pg_stat_statements.track = all · ★ track_io_timing = on
auto_explain.log_min_duration='500ms' · log_analyze=on
  · ★ log_timing=off · ★ sample_rate=0.05
log_lock_waits=on · log_temp_files=0 · ★ log_line_prefix with %a
```

**★ Look for feedback loops.** A long query blocks autoanalyze ⇒ stats stay stale ⇒ the plan stays bad.

---

## When would I use this at work?

1. **Every performance incident, without exception.** The method's value is that it eliminates categories in the first two steps. In the example it took 11 minutes to go from "checkout is slow" to a specific runaway job, a stale-statistics feedback loop, and three structural fixes — and the team's instinct would have produced an index that changed nothing.

2. **Before provisioning anything.** Low CPU with failing requests is contention or queueing, and capacity fixes neither. That one discriminator prevents most wasted infrastructure spend.

3. **When "it's slow" means "slower than last week".** A `pg_stat_statements` snapshot cannot answer that; a diff against the same window yesterday can, and it usually names the regression directly.

4. **Setting up a new service.** `pg_stat_statements`, `auto_explain` with sampling, `track_io_timing`, snapshot diffs, and the three per-request numbers cost an afternoon and turn every future incident from archaeology into a checklist.

---

## Connected topics

**Understand before this:** 18 (EXPLAIN), 15 (statistics and selectivity — the misestimate), 46/47 (MVCC and autovacuum — the feedback loop), 61 (contention), 65 (queueing), 66 (N+1).

**This unlocks:**
- **68** — CAP and consistency models
- **69** — security at the data layer
- **77–79** — the Phase 9 capstones, where the method is applied end to end
