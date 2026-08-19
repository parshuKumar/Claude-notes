# 60 — Sharding
## Phase: Denormalisation & Scale

---

## ELI5 — The Simple Analogy

One post office serving a whole city. The queue is out the door. You cannot make the building bigger — you have already knocked through every wall.

So you open four post offices, one per district, and **every resident is permanently assigned to exactly one of them** by their address.

Four things change, and they are the entire topic:

1. ★ **You must be able to work out which office someone belongs to, from the address alone, before you go.** If you don't know the district, you have to visit all four. Every time.

2. ★ **A parcel from one district to another is now genuinely hard.** Two offices, two ledgers, no shared counter. What used to be one clerk moving a slip between two drawers is now a coordination problem between two buildings (Topic 51).

3. ★ **"How many parcels did the city handle today?" means asking all four and adding up.** Every question that isn't about one district costs four times as much and is only as fast as the slowest office.

4. ★ **When you open a fifth office, everybody's district assignment changes.** Unless you were careful about how you assigned them in the first place — and being careful about that is most of the design work.

★ **And the part that decides everything: you chose to split by address. If most of your queries are "find this parcel by tracking number," you chose wrong, and you cannot change your mind cheaply.**

---

## Where this fits in the big picture

```
   57 caching · 58 replicas — ★ read scaling
   59 partitioning — ★ one server, many files, full transactions
   51 2PC — ★ why cross-node atomicity is expensive
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 60 SHARDING ← YOU ARE HERE                   │
        │ ★ MANY SERVERS. The only way to scale WRITES │
        │   — and a distributed system, with all that  │
        │   implies.                                    │
        └────────────────────┬─────────────────────────┘
                             ▼
              61 hot rows · 68 CAP · 76 polyglot persistence
```

★ **Sharding is the last resort in this phase, and deliberately so.** Everything before it — indexes, denormalisation, matviews, caching, replicas, partitioning — keeps you on one machine with full transactional semantics. Sharding gives that up permanently.

---

## What is this?

Splitting data **across independent database servers**, each holding a disjoint subset, chosen by a **shard key**.

```
              application
                   │
          ┌────────┼────────┬────────┐
          ▼        ▼        ▼        ▼
      ┌───────┐┌───────┐┌───────┐┌───────┐
      │shard 0││shard 1││shard 2││shard 3│
      │ 0–25% ││25–50% ││50–75% ││75–100%│
      └───────┘└───────┘└───────┘└───────┘
      ★ each is a COMPLETE, INDEPENDENT PostgreSQL server
```

**What it gives you, and what it costs:**

| | Gained | Lost |
|---|---|---|
| Write throughput | ★ ~N× | — |
| Storage | ★ ~N× | — |
| Working set in RAM | ★ ~N× | — |
| Cross-shard joins | — | ★ **gone** |
| Cross-shard transactions | — | ★ **gone (or 2PC — Topic 51)** |
| Global unique constraints | — | ★ **gone** |
| `AUTO_INCREMENT` / sequences | — | ★ **gone** |
| Aggregate queries | — | ★ scatter-gather, slowest-shard-bound |
| Operational complexity | — | ★ **N× everything** |

★ **The one sentence that matters:** *sharding is the only technique that scales writes, and it is the only one that converts your database into a distributed system.* Every other tool in Phase 6 is reversible. This one is not.

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE DECISION IS MADE ONCE, EARLY, AND CANNOT BE UNDONE.

 ★ THE SHARD KEY IS THE MOST CONSEQUENTIAL SCHEMA DECISION YOU
   WILL EVER MAKE.
   • it determines which queries are fast and which are impossible
   • it determines whether your data is evenly distributed
   • ★ changing it later means rewriting every row on every shard
     AND rewriting the application AND doing both online
   • ⇒ ★ measured in quarters, not sprints

 ★ AND THE MOST COMMON OUTCOME IS THAT IT WAS NEVER NEEDED:
   the "we need to shard" conversation almost always precedes:
   ① a missing index                  (Topic 54 gate 2)
   ② an N+1                            (Topic 66)
   ③ a hot row                         (Topic 61)
   ④ ★ retention done with DELETE      (Topic 59)
   ⑤ a cache that was a pessimisation  (Topic 57)
 ⇒ ★ FIX ALL FIVE BEFORE YOU SHARD. Most teams find they no
   longer need to.
```

---

## The physical reality

### The exhaustion checklist — what "one machine isn't enough" actually means

```
 ★ A SINGLE POSTGRESQL SERVER IN 2026 CAN COMFORTABLY DO:
   • 64–128 cores, 1–4 TB RAM
   • ★ 50,000–200,000 simple writes/sec on NVMe
   • ★ 10–50 TB of data
   • hundreds of thousands of reads/sec with replicas

 ⇒ ★ BEFORE CONCLUDING "ONE MACHINE ISN'T ENOUGH", PROVE EACH:

 ① ★ WRITE THROUGHPUT — is it really the ceiling?
    SELECT sum(xact_commit + xact_rollback) FROM pg_stat_database;
    ⇒ ★ measure WAL bytes/sec too — often the real limit
    ⇒ have you: batched writes? removed redundant indexes?
      enabled HOT updates? set wal_compression? used COPY?

 ② ★ DATA SIZE — is it real data, or bloat and retention failure?
    ⇒ ★ Topic 47: bloat. Topic 59: DELETE-based retention.
    ⇒ MEASURED IN PRACTICE: a 412 GB table with 11 GB of live data.
      "We have 40 TB" is very often "we have 4 TB and no retention."

 ③ ★ WORKING SET — does the hot data fit in RAM?
    SELECT round(100.0*sum(heap_blks_hit)/
      nullif(sum(heap_blks_hit)+sum(heap_blks_read),0),2)
      FROM pg_statio_user_tables;
    ⇒ > 99% ⇒ ★ your working set fits. Sharding for RAM is
      not the answer; a bigger box or better indexes is.

 ④ ★ CONNECTIONS — is it a pooling problem?
    ⇒ 5,000 connections to one Postgres is a PgBouncer problem,
      not a sharding problem. (Topic 65.)

 ⑤ ★ ONE HOT ROW / ONE HOT KEY
    ⇒ ★ SHARDING DOES NOT FIX THIS. If one key takes 40% of
      writes, it lands on one shard and that shard is your
      bottleneck. (Topic 61.)

 ⇒ ★ IF ALL FIVE ARE GENUINELY EXHAUSTED, SHARD. Otherwise you
   are about to take on a distributed system to avoid writing
   an index.
```

### Choosing the shard key — the decision you cannot undo

```
 ★ THE KEY MUST SATISFY FOUR PROPERTIES. Losing any one is fatal.

 ① ★ PRESENT IN NEARLY EVERY QUERY
    if a query doesn't include the key, it must SCATTER to all
    shards. Scatter-gather is bounded by the SLOWEST shard and
    consumes N× the resources.
    ⇒ ★ TARGET: > 95% of queries are single-shard.

 ② ★ EVEN DISTRIBUTION
    ⇒ ★ real-world entity sizes are POWER-LAW distributed.
      Sharding by tenant_id in a B2B SaaS gives you one shard
      holding your largest customer and 15 nearly empty ones.
    ⇒ MEASURE IT BEFORE COMMITTING:
      SELECT tenant_id, count(*), 
             100.0*count(*)/sum(count(*)) OVER () AS pct
        FROM orders GROUP BY 1 ORDER BY 2 DESC LIMIT 10;
      ⇒ ★ if the top tenant is > 5% of rows, tenant_id alone is
        not a viable shard key.

 ③ ★ IMMUTABLE
    if the key can change, the row must MOVE BETWEEN SHARDS —
    which is a cross-shard transaction, i.e. the thing you were
    avoiding.
    ⇒ ★ user_id ✓  ·  account_status ✗  ·  region ✗ (users move)

 ④ ★ CO-LOCATES RELATED DATA
    a customer's orders, payments and addresses should live on
    the SAME shard, or every read becomes cross-shard.
    ⇒ ★ THIS IS WHY tenant_id / customer_id ARE THE MOST COMMON
      KEYS: they form a natural "everything about X" boundary.

 ★ THE THREE CANDIDATE SHAPES:
   ⑴ tenant/customer id  ✓①✓③✓④  ★ ✗② (power law) unless mitigated
   ⑵ entity id (order id) ✓②✓③  ★ ✗① (queries are by customer)
                                ★ ✗④ (nothing co-locates)
   ⑶ ★ a COMPOSITE — hash(tenant_id) with large tenants split out
      ⇒ the practical answer in most real systems
```

### Range vs hash vs directory

```
 ★ ① RANGE (shard 0 = ids 0–1M, shard 1 = 1M–2M, …)
    ✓ range queries work
    ✓ ★ adding a shard is easy — just extend the range
    ✗ ★ HOTSPOTS: with sequential ids, ALL new writes go to the
      LAST shard. The others are idle.
    ⇒ ★ almost always wrong for a write-scaling motive.

 ★ ② HASH (shard = hash(key) mod N)
    ✓ ★ even distribution, by construction
    ✗ range queries scatter
    ✗ ★ CHANGING N REMAPS ALMOST EVERYTHING.
      4 → 5 shards moves ★ ~80% of all rows.

 ★ ③ CONSISTENT HASHING / VIRTUAL BUCKETS — the practical answer
    hash the key into a FIXED, LARGE number of buckets (e.g. 4096),
    then MAP buckets → physical shards.
      bucket  = hash(key) % 4096          ★ never changes
      shard   = bucket_map[bucket]        ★ a small lookup table
    ⇒ ★ adding a shard moves only 1/N of the BUCKETS, not a
      rehash of everything.
    ⇒ ★ 4 → 5 shards moves ~20% of rows, and you choose WHICH.
    ⇒ ★ THIS IS WHAT VITESS, CITUS AND EVERY MATURE SYSTEM DO.
      ★ IF YOU BUILD SHARDING YOURSELF, BUILD THIS ONE.

 ★ ④ DIRECTORY (a lookup table: key → shard)
    ✓ ★ total flexibility — move any tenant anywhere
    ✓ ★ handles power-law distribution: give the big tenant its
      own shard
    ✗ ★ the directory is a new single point of failure and must
      be cached everywhere
    ⇒ ★ THE RIGHT ANSWER FOR B2B MULTI-TENANT SYSTEMS, where
      tenant sizes differ by 1000× and you need per-tenant control.
```

### What breaks, concretely

```
 ★ ① GLOBAL UNIQUENESS IS GONE
    UNIQUE (email) can only be enforced WITHIN a shard.
    ⇒ ★ TWO USERS CAN REGISTER THE SAME EMAIL ON DIFFERENT SHARDS.
    ⇒ FIXES:
      ⒜ ★ shard BY the unique column (email → shard). Then it's
         local. Works only for one such column.
      ⒝ ★ a small unsharded "registry" table holding all emails,
         written first. ⇒ every registration touches two systems.
      ⒞ accept it and reconcile. ⇒ usually unacceptable.

 ★ ② SEQUENCES ARE GONE
    bigserial on 4 shards produces id=1 on all four.
    ⇒ FIXES (Topic 22):
      ★ Snowflake ids (timestamp | shard_id | sequence)
      ★ UUIDv7 / ULID — globally unique, time-ordered
      ⒞ per-shard sequence offsets (shard N starts at N, step 4)
        ⇒ ★ fragile: adding a shard breaks the scheme.

 ★ ③ CROSS-SHARD TRANSACTIONS ARE GONE
    "transfer money from account A to account B" where A and B
    are on different shards.
    ⇒ ★ 2PC (Topic 51 — blocking, availability multiplies) or
      ★ a SAGA with compensations (Topic 52).
    ⇒ ★ OR: design so it never happens. Co-locate accounts that
      transact with each other (e.g. shard by ledger, not account).

 ★ ④ CROSS-SHARD JOINS ARE GONE
    ⇒ FIXES:
      ★ REFERENCE TABLES — replicate small, slowly-changing tables
        (countries, currencies, product catalogue) to EVERY shard.
        ⇒ joins to them become local.
      ★ denormalise the needed column (Topic 53 — and this is one
        of its genuinely strongest cases)
      ⒞ join in the application (an N+1 across the network — ★ bad)

 ★ ⑤ AGGREGATES BECOME SCATTER-GATHER
    SELECT count(*) FROM orders;
    ⇒ query all N shards, sum the results
    ⇒ ★ latency = the SLOWEST shard, not the average
    ⇒ ★ p99 of a scatter-gather over 16 shards ≈ p99.994 of one
      shard. Tail latency compounds brutally.
    ⇒ ★ FIX: precompute aggregates (Topic 56's rollup tables) or
      push them to a separate analytics store (Topic 76).

 ★ ⑥ SCHEMA MIGRATIONS ARE N× HARDER
    ⇒ every DDL must run on every shard, and shards can diverge
      mid-migration.
    ⇒ ★ you need a migration runner that tracks per-shard state
      and can resume.
```

### Resharding — the operation you must plan for on day one

```
 ★ YOU WILL ADD SHARDS. DESIGN FOR IT BEFORE YOU NEED IT.

 THE ONLINE RESHARD, WITH VIRTUAL BUCKETS:
 ① choose which buckets move to the new shard
 ② ★ start LOGICAL REPLICATION of those buckets to the new shard
    (Topic 58 — logical, not physical, because it's selective)
 ③ wait for the copy to catch up
 ④ ★ BRIEFLY freeze writes for those buckets only
    ⇒ a short, bounded write pause on 1/N of your traffic
 ⑤ verify row counts and checksums
 ⑥ flip bucket_map[bucket] = new_shard
 ⑦ unfreeze
 ⑧ delete the moved buckets from the old shard, ★ after a soak

 ★ MEASURED, a 4→8 reshard of 2 TB:
   copy phase       ★ 14 hours (online, no impact)
   freeze window    ★ 4.2 seconds per bucket batch
   total moved      ★ 50% of rows
   ⇒ ★ AND THIS IS ONLY POSSIBLE BECAUSE OF THE BUCKET LAYER.
     With hash(key) % N, the same operation moves 80% of rows and
     has no safe incremental path.
```

---

## How it works — step by step

### Should you shard? — the honest sequence

```
 ★ WORK THROUGH ALL OF THESE FIRST. WRITE DOWN WHAT EACH BOUGHT.

 ① VERTICAL SCALING — a bigger machine
    ⇒ ★ 128 cores / 4 TB RAM is available and cheap relative to
      a distributed system. ★ Exhaust this first, always.
 ② INDEXES (Topic 54 gate 2)          — usually 100–1,000×
 ③ FIX N+1 (Topic 66)
 ④ ★ RETENTION VIA PARTITIONING (59)  — often removes 90% of data
 ⑤ CACHING / CDN (57)                 — read load
 ⑥ READ REPLICAS (58)                 — read load
 ⑦ ★ FIX HOT ROWS (61)                — sharding does NOT fix these
 ⑧ ★ SPLIT BY SERVICE (functional partitioning)
    move a distinct workload — events, analytics, sessions — to
    its own database. ★ Much simpler than sharding: no shard key,
    no cross-shard queries, and it often removes the bottleneck.
 ⑨ ★ ONLY NOW: SHARD.
```

### The application-level shard router

```js
// ★ VIRTUAL BUCKETS — the layer that makes resharding survivable
const BUCKET_COUNT = 4096;                 // ★ fixed forever

function bucketFor(shardKey) {
  // ★ a STABLE hash. Never Node's built-in hash or JSON key order.
  return murmur3(String(shardKey)) % BUCKET_COUNT;
}

// bucket_map is loaded from a config store and cached in-process,
// refreshed on a version bump
let bucketMap = [];                        // bucket → shard name
let bucketMapVersion = 0;

function shardFor(shardKey) {
  const b = bucketFor(shardKey);
  const shard = bucketMap[b];
  if (!shard) throw new Error(`no shard for bucket ${b}`);
  return shard;
}

async function withShard(shardKey, fn) {
  const shard = shardFor(shardKey);
  const pool = pools.get(shard);
  metrics.increment('shard.query', { shard });
  return fn(pool);
}

// ★ every query MUST carry the shard key. Make it impossible to forget.
async function getOrders(customerId) {
  return withShard(customerId, (pool) =>
    pool.query('SELECT * FROM orders WHERE customer_id = $1', [customerId]));
}
```

```js
// ★ scatter-gather, with the tail-latency problem made explicit
async function countAllOrders() {
  const t0 = Date.now();
  const results = await Promise.all(
    [...pools.entries()].map(async ([name, pool]) => {
      const t = Date.now();
      const r = await pool.query('SELECT count(*)::bigint AS n FROM orders');
      metrics.histogram('scatter.shard_ms', Date.now() - t, { shard: name });
      return BigInt(r.rows[0].n);
    }));
  // ★ total latency = the SLOWEST shard. Record both.
  metrics.histogram('scatter.total_ms', Date.now() - t0);
  return results.reduce((a, b) => a + b, 0n);
}
// ★ AND: a partial failure means a WRONG ANSWER, not an error.
//   Decide explicitly whether to fail or to return partial results.
```

```js
// ★ globally unique ids without a shared sequence (Topic 22)
function snowflakeId(shardId) {
  const ts = BigInt(Date.now() - EPOCH);      // 41 bits
  const seq = BigInt(nextSeq() & 0xfff);      // 12 bits
  return (ts << 22n) | (BigInt(shardId) << 12n) | seq;
  // ★ time-ordered (good B-tree locality), globally unique,
  //   and the shard is recoverable from the id itself.
}
```

### Reference tables — making cross-shard joins local

```sql
-- ★ replicated to EVERY shard, identical everywhere
CREATE TABLE ref_countries (code char(2) PRIMARY KEY, name text NOT NULL);
CREATE TABLE ref_currencies (code char(3) PRIMARY KEY, minor_units int NOT NULL);
CREATE TABLE ref_products (
  sku text PRIMARY KEY, name text NOT NULL, category_id bigint NOT NULL);

-- ★ maintained by logical replication FROM a single authoritative
--   source, or by an idempotent sync job.
-- ★ RULES:
--   • small (< ~1M rows)
--   • slowly changing
--   • ★ NEVER written on a shard — one writer only
--   • a reconciler comparing checksums across shards
SELECT md5(string_agg(sku || name, ',' ORDER BY sku)) FROM ref_products;
-- ★ must be identical on every shard.
```

---

## Concept breakdown

```
WHAT IT IS
└── data split across INDEPENDENT SERVERS by a shard key
     ★ the ONLY technique that scales WRITES
     ★ and the only one that makes your DB a distributed system

★ PARTITIONING vs SHARDING — do not confuse these
   partitioning: one server, many files, ★ full transactions,
                 full joins, full constraints
   sharding:     many servers, ★ no cross-shard joins, no
                 cross-shard transactions, no global uniqueness

★ THE EXHAUSTION CHECKLIST — prove all five before sharding
   ① write throughput really at the ceiling?
   ② data size real, or ★ bloat / DELETE-retention?
   ③ working set fits in RAM? (hit ratio > 99% ⇒ yes)
   ④ ★ connections — a PgBouncer problem, not a sharding one
   ⑤ ★ ONE HOT KEY — ★ SHARDING DOES NOT FIX THIS (Topic 61)

★ THE SHARD KEY — four required properties
   ① ★ present in > 95% of queries
   ② ★ evenly distributed (★ real entities are POWER-LAW)
   ③ ★ immutable (a changing key ⇒ cross-shard row moves)
   ④ ★ co-locates related data

★ STRATEGIES
├── range      ✓ range queries · ★ ✗ HOTSPOT on the last shard
├── hash       ✓ even · ★ ✗ 4→5 shards remaps ~80% of rows
├── ★ VIRTUAL BUCKETS  hash → 4096 buckets → shard map
│                ★ 4→5 moves ~20%, and you choose which
│                ★ BUILD THIS ONE
└── ★ directory  key → shard lookup table
                 ★ handles power-law tenants; a new SPOF

★ WHAT BREAKS
├── ★ global UNIQUE      ⇒ shard by it, or an unsharded registry
├── ★ sequences          ⇒ Snowflake / UUIDv7 (Topic 22)
├── ★ cross-shard txns   ⇒ 2PC (51) or sagas (52), or design them away
├── ★ cross-shard joins  ⇒ ★ REFERENCE TABLES replicated everywhere
├── ★ aggregates         ⇒ scatter-gather, ★ bounded by the SLOWEST
│                          shard; p99 over 16 shards ≈ p99.994 of one
└── ★ migrations         ⇒ N× harder, need per-shard state tracking

★ RESHARDING — plan it on day one
   virtual buckets ⇒ logical-replicate selected buckets → verify →
   ★ brief per-bucket write freeze → flip the map → soak → delete
   ⇒ 4→8 on 2 TB: ★ 14 h online copy, ★ 4.2 s freeze window
```

---

## Diagrams

**Diagram 1 — big picture: why the bucket layer exists**

```
 ✗ NAIVE HASH — shard = hash(key) % N
 ┌───────────────────────────────────────────────────────────────┐
 │  N = 4                          N = 5                         │
 │  key 1001 → hash%4 = 1          key 1001 → hash%5 = ★ 3       │
 │  key 1002 → hash%4 = 2          key 1002 → hash%5 = ★ 0       │
 │  key 1003 → hash%4 = 3          key 1003 → hash%5 = ★ 2       │
 │                                                                │
 │  ★ ADDING ONE SHARD REMAPS ~80% OF ALL ROWS.                  │
 │  ★ There is no incremental path. You must move almost         │
 │    everything, at once, correctly.                            │
 └───────────────────────────────────────────────────────────────┘

 ✓ VIRTUAL BUCKETS — hash → 4096 buckets → a shard MAP
 ┌───────────────────────────────────────────────────────────────┐
 │  bucket = hash(key) % 4096      ★ NEVER CHANGES               │
 │                                                                │
 │  bucket_map (a small table, 4096 rows):                        │
 │    buckets    0–1023  → shard-0                                │
 │    buckets 1024–2047  → shard-1                                │
 │    buckets 2048–3071  → shard-2                                │
 │    buckets 3072–4095  → shard-3                                │
 │                                                                │
 │  ADDING shard-4:                                               │
 │    move buckets  820–1023 → shard-4   (from shard-0)           │
 │    move buckets 1844–2047 → shard-4   (from shard-1)           │
 │    move buckets 2868–3071 → shard-4   (from shard-2)           │
 │    move buckets 3892–4095 → shard-4   (from shard-3)           │
 │                                                                │
 │  ★ ~20% OF ROWS MOVE, AND YOU CHOOSE WHICH, ONE BATCH AT A    │
 │    TIME, WITH A 4-SECOND FREEZE PER BATCH.                    │
 │  ★ THIS LAYER IS THE DIFFERENCE BETWEEN A RESHARD BEING A     │
 │    ROUTINE OPERATION AND A MULTI-WEEK PROJECT.                │
 └───────────────────────────────────────────────────────────────┘
```

**Diagram 2 — data flow: single-shard vs scatter-gather**

```
 ✓ SINGLE-SHARD — the query carries the shard key
   SELECT * FROM orders WHERE customer_id = 8842 AND status='paid';
        │
        │ bucket = hash(8842) % 4096 = 1204 → shard-1
        ▼
   ┌─────────┐
   │ shard-1 │  ★ 0.8 ms
   └─────────┘
   ⇒ ★ exactly as fast as an unsharded database. This is the
     ONLY query shape that scales.

 ✗ SCATTER-GATHER — the key is absent
   SELECT count(*) FROM orders WHERE status = 'paid';
        │
   ┌────┼────┬────┬────┬────┬────┬────┬────┐
   ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼
  s0   s1   s2   s3   s4   s5   s6   s7  … s15
 0.8  0.9  0.8  1.1  0.8  0.9  ★14.2 0.8  0.9  ms
                              ▲
                    ★ one slow shard (a vacuum, a
                      checkpoint, a noisy neighbour)
        │
        ▼  ★ TOTAL = 14.2 ms — the MAXIMUM, not the average
   ⇒ ★ AND THE TAIL COMPOUNDS:
     P(all 16 fast) = 0.99^16 = 85%
     ⇒ ★ the p99 of a 16-shard scatter ≈ the p99.994 of one shard.
   ⇒ ★ AND A PARTIAL FAILURE RETURNS A WRONG ANSWER, NOT AN ERROR.
```

**Diagram 3 — before/after: the power-law shard key**

```
 ✗ SHARD BY tenant_id, hash % 8 — a B2B SaaS
 ┌───────────────────────────────────────────────────────────────┐
 │ tenant row counts (measured before committing):                │
 │   tenant  1204 :  41,204,882 rows   ★ 38.2% of all data        │
 │   tenant  8842 :  18,402,118        17.1%                      │
 │   tenant  4120 :   8,204,201         7.6%                      │
 │   … 8,397 others: ~37% combined                                │
 │                                                                │
 │ hash(1204) % 8 = 3                                             │
 │                                                                │
 │  s0   s1   s2   ★s3   s4   s5   s6   s7                        │
 │  8%   6%   7%  ★45%   9%   8%   9%   8%     of data            │
 │  ▂    ▂    ▂   ████   ▂    ▂    ▂    ▂                         │
 │                                                                │
 │ ⇒ ★ shard-3 is 5× the others. It saturates first. You have    │
 │   8 servers and the throughput of ~2.                          │
 │ ⇒ ★ AND YOU CANNOT REBALANCE — tenant 1204 is one key.        │
 └───────────────────────────────────────────────────────────────┘

 ✓ A DIRECTORY + A COMPOSITE KEY FOR LARGE TENANTS
 ┌───────────────────────────────────────────────────────────────┐
 │ shard_directory:                                               │
 │   tenant_id | strategy      | shards                           │
 │   ----------+---------------+----------------------            │
 │   ★ 1204    | dedicated     | shard-a, shard-b   ← ★ its own,  │
 │             |               |   sub-sharded by customer_id     │
 │   ★ 8842    | dedicated     | shard-c                          │
 │   *         | hashed        | shard-0 … shard-7                │
 │                                                                │
 │ shard key for hashed tenants:  hash(tenant_id)                 │
 │ shard key for tenant 1204:     ★ hash(tenant_id, customer_id)  │
 │                                                                │
 │  sa   sb   sc   s0   s1   s2   s3   s4   s5   s6   s7          │
 │ 19%  19%  17%  6%   6%   6%   6%   6%   6%   5%   4%           │
 │ ███  ███  ███  ▃    ▃    ▃    ▃    ▃    ▃    ▃    ▃            │
 │                                                                │
 │ ⇒ ★ even enough. And when tenant 1204 doubles, you split its   │
 │   dedicated shards again without touching anyone else.         │
 │ ⇒ ★ THE DIRECTORY IS WHAT MAKES POWER-LAW TENANCY SURVIVABLE. │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Prove the distribution before choosing a key.**
```sql
SELECT customer_id, count(*) AS n,
       round(100.0*count(*)/sum(count(*)) OVER (), 2) AS pct,
       round(100.0*sum(count(*)) OVER (ORDER BY count(*) DESC)
             / sum(count(*)) OVER (), 2) AS cumulative_pct
  FROM orders GROUP BY 1 ORDER BY 2 DESC LIMIT 10;
```
```
 customer_id |    n     |  pct  | cumulative_pct
-------------+----------+-------+----------------
        1204 | 41204882 | ★38.2 |          ★38.2
        8842 | 18402118 |  17.1 |           55.3
        4120 |  8204201 |   7.6 |           62.9
   ★ THREE CUSTOMERS ARE 63% OF THE DATA.
     customer_id is NOT a viable hash shard key here.
```

**Simulate the shard distribution before committing.**
```sql
-- ★ do this BEFORE you build anything
SELECT (hashtext(customer_id::text) & 2147483647) % 8 AS shard,
       count(*) AS rows,
       round(100.0*count(*)/sum(count(*)) OVER (), 1) AS pct
  FROM orders GROUP BY 1 ORDER BY 1;
```
```
 shard |   rows   |  pct
-------+----------+-------
     0 |  8204118 |   7.6
     1 |  6402881 |   5.9
     2 |  7104882 |   6.6
     3 | ★48820114| ★ 45.2      — ★ tenant 1204 landed here
     4 |  9204118 |   8.5
     5 |  8842119 |   8.2
     6 |  9420114 |   8.7
     7 |  9902884 |   9.2
   ★ 45% on one shard. This design is dead before it is built.
```

**And with virtual buckets — the same imbalance, but now movable.**
```sql
SELECT (hashtext(customer_id::text) & 2147483647) % 4096 AS bucket,
       count(*) AS rows
  FROM orders GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
```
```
 bucket |   rows
--------+-----------
 ★ 1204 | 41204882      — one bucket holds the giant tenant
   3820 |  1204118
   0044 |  1188402
   ★ NOW YOU CAN MAP BUCKET 1204 TO ITS OWN DEDICATED SHARD.
     With hash % 8 you could not. ★ This is the bucket layer
     earning its keep.
```

**Set up four shards locally.**
```bash
for i in 0 1 2 3; do
  initdb -D "/tmp/shard$i"
  echo "port = 555$i" >> "/tmp/shard$i/postgresql.conf"
  pg_ctl -D "/tmp/shard$i" -l "/tmp/shard$i.log" start
  createdb -p "555$i" shop
  psql -p "555$i" -d shop -c "
    CREATE TABLE orders (
      id bigint PRIMARY KEY,
      customer_id bigint NOT NULL,
      total_minor bigint NOT NULL,
      created_at timestamptz NOT NULL DEFAULT now());
    CREATE INDEX ON orders (customer_id, created_at DESC);"
done
```

**Prove global uniqueness is gone.**
```bash
psql -p 5550 -d shop -c "
  CREATE TABLE users (id bigint PRIMARY KEY, email text UNIQUE NOT NULL);
  INSERT INTO users VALUES (1, 'meera@example.com');"
psql -p 5551 -d shop -c "
  CREATE TABLE users (id bigint PRIMARY KEY, email text UNIQUE NOT NULL);
  INSERT INTO users VALUES (2, 'meera@example.com');"
```
```
 INSERT 0 1
 INSERT 0 1
   ★ BOTH SUCCEEDED. The same email now exists twice.
     UNIQUE is per-shard. There is no global constraint.
```

**And the registry-table fix.**
```sql
-- ★ on an unsharded "global" database
CREATE TABLE email_registry (
  email      text   PRIMARY KEY,
  user_id    bigint NOT NULL,
  shard      text   NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now());
```
```js
// ★ reserve globally FIRST, then write to the shard.
async function registerUser(email, name) {
  const id = snowflakeId(SHARD_ID);
  const shard = shardFor(id);
  // ① claim the email globally — ★ this is the uniqueness guarantee
  const { rowCount } = await globalPool.query(
    `INSERT INTO email_registry (email, user_id, shard)
     VALUES ($1,$2,$3) ON CONFLICT (email) DO NOTHING`, [email, id, shard]);
  if (rowCount === 0) throw new AppError('EMAIL_TAKEN');
  try {
    // ② write the user to its shard
    await pools.get(shard).query(
      'INSERT INTO users (id, email, name) VALUES ($1,$2,$3)', [id, email, name]);
  } catch (e) {
    // ★ compensate — this is a saga (Topic 52), not a transaction
    await globalPool.query('DELETE FROM email_registry WHERE email=$1', [email]);
    throw e;
  }
  return id;
}
// ★ NOTE: a crash between ① and ② leaves an orphan registry row.
//   ⇒ a reconciler must sweep registry rows with no matching user.
//   ★ THIS IS THE REAL COST OF LOSING A UNIQUE CONSTRAINT.
```

**Prove sequences collide.**
```bash
for p in 5550 5551; do
  psql -p $p -d shop -c "
    CREATE TABLE t (id bigserial PRIMARY KEY, v text);
    INSERT INTO t (v) VALUES ('x') RETURNING id;"
done
```
```
 id
----
  1
 id
----
  ★ 1        — the same id on two shards
```

**Snowflake ids instead.**
```sql
-- shard-aware id generation in the database
CREATE OR REPLACE FUNCTION snowflake_id(shard_id int) RETURNS bigint AS $$
DECLARE
  epoch bigint := 1735689600000;   -- 2025-01-01
  seq   bigint;
BEGIN
  seq := nextval('id_seq') % 4096;
  RETURN ((floor(extract(epoch from clock_timestamp())*1000)::bigint - epoch) << 22)
         | (shard_id::bigint << 12) | seq;
END $$ LANGUAGE plpgsql;

CREATE SEQUENCE id_seq;
SELECT snowflake_id(0), snowflake_id(1);
```
```
   snowflake_id   |   snowflake_id
------------------+------------------
 ★ 132891038834689 | ★ 132891038838785
   ★ globally unique, time-ordered, and the shard is recoverable:
   SELECT (132891038838785 >> 12) & 1023 AS shard;  -- 1
```

**Measure scatter-gather tail latency.**
```js
// 16 shards, one deliberately slowed
const times = [0.8,0.9,0.8,1.1,0.8,0.9,14.2,0.8,0.9,1.0,0.8,0.9,1.1,0.8,0.9,1.0];
console.log('avg shard:', avg(times).toFixed(2));   // 1.67 ms
console.log('★ total  :', Math.max(...times));       // ★ 14.20 ms
// ⇒ the scatter costs the MAXIMUM, and one bad shard dominates.
```

**Prove a cross-shard transaction is not a transaction.**
```js
// ✗ this looks atomic and is not
await pools.get('shard-0').query('UPDATE accounts SET balance = balance-500 WHERE id=1');
// ★ crash here ⇒ money destroyed
await pools.get('shard-1').query('UPDATE accounts SET balance = balance+500 WHERE id=2');
```
```
 ★ THERE IS NO BEGIN THAT SPANS THESE. Your options are 2PC
   (Topic 51) or a saga with compensations (Topic 52) — or
   designing so that accounts that transact are CO-LOCATED.
```

---

## Example 2 — production scenario

**The situation.** A B2B logistics SaaS. 8,400 tenants. The team schedules a "sharding project."

```
 THE STATED PROBLEM
   primary                 ★ 96 vCPU, 768 GB RAM
   CPU                     ★ 94% sustained
   write throughput        ★ 41,000 writes/sec, "at the ceiling"
   database size           ★ 18 TB
   p99 on the main API     ★ 4,200 ms
   the proposal            ★ shard by tenant_id into 16 shards
```

**Step 1 — work the exhaustion checklist before agreeing.**

```sql
-- ① is the write throughput real?
SELECT sum(xact_commit)/extract(epoch from now()-stats_reset) AS commits_per_sec
  FROM pg_stat_database WHERE datname = current_database();
```
```
 commits_per_sec
-----------------
      ★ 41,204
```
```sql
-- ★ but WHAT are they writing?
SELECT calls, mean_exec_time::numeric(10,3) AS ms,
       (calls*mean_exec_time/1000/3600)::numeric(10,1) AS hours,
       substring(query,1,70) AS q
  FROM pg_stat_statements ORDER BY calls*mean_exec_time DESC LIMIT 5;
```
```
   calls    |   ms    | hours  |                      q
------------+---------+--------+---------------------------------------------
 1884201188 |  ★ 0.08 | ★ 41.9 | UPDATE shipments SET last_scan_at = now() …
  412088420 |  ★ 8.84 | ★ 1011 | SELECT … FROM shipments WHERE tenant_id=$1 …
   88420118 |    0.12 |    2.9 | INSERT INTO scan_events …
      41204 | ★ 14200 |  ★ 162 | DELETE FROM scan_events WHERE created_at < $1
```

```
 ★ THREE FINDINGS, NONE OF WHICH SHARDING FIXES:

 ① 1.88 BILLION UPDATEs OF last_scan_at.
    SELECT relname, n_tup_upd, n_tup_hot_upd,
           round(100.0*n_tup_hot_upd/n_tup_upd,1) AS hot_pct
      FROM pg_stat_user_tables WHERE relname='shipments';
    ⇒ hot_pct = ★ 0.9%
    ⇒ ★ an index on last_scan_at is killing HOT updates (Topic 46)
    ⇒ ★ 8 pages dirtied per update instead of 1

 ② THE 8.84 ms SELECT runs 412 MILLION times = ★ 1,011 HOURS/DAY
    EXPLAIN shows a Bitmap Heap Scan with 8.2M rows removed by
    filter ⇒ ★ A MISSING COMPOSITE INDEX.

 ③ THE NIGHTLY DELETE is 14 SECONDS × 41,204 calls — it runs in a
    loop because it never finishes. ⇒ ★ Topic 59.
```

**Step 2 — fix the five things first.**

```sql
-- ① restore HOT updates (Topic 46)
DROP INDEX CONCURRENTLY idx_shipments_last_scan;   -- ★ used 1,204 times/9 days
ALTER TABLE shipments SET (fillfactor = 70);
-- ⇒ hot_pct 0.9% → ★ 96.8%
-- ⇒ WAL 88 GB/hr → ★ 11 GB/hr

-- ② the missing index (Topic 54 gate 2)
CREATE INDEX CONCURRENTLY idx_shipments_tenant_status_created
  ON shipments (tenant_id, created_at DESC) WHERE status <> 'delivered';
-- ⇒ 8.84 ms → ★ 0.9 ms
-- ⇒ ★ 1,011 hours/day → 103 hours/day

-- ③ retention via partitioning (Topic 59)
-- scan_events range-partitioned by day, DROP instead of DELETE
-- ⇒ 14 s × 41,204 → ★ 12 ms × 1
-- ⇒ ★ AND: 18 TB → 4.1 TB, because 77% was expired data the
--   DELETE had never managed to remove.

-- ④ a hot row (Topic 61)
SELECT tenant_id, count(*) FROM shipment_counters
 GROUP BY 1 ORDER BY 2 DESC LIMIT 3;
-- ⇒ one counter row per tenant, tenant 1204 at 4,100 updates/sec
-- ⇒ ★ sharded the counter into 16 sub-rows

-- ⑤ connection pooling (Topic 65)
-- 4,200 direct connections → PgBouncer in transaction mode, 120 backends
```

**Step 3 — re-measure.**

```
 ★ AFTER FIXING THE FIVE, BEFORE ANY SHARDING:
   CPU                 94% → ★ 31%
   write throughput    41,000/s → ★ 44,000/s (higher, and idle)
   database size       18 TB → ★ 4.1 TB
   p99                 4,200 ms → ★ 84 ms
   WAL                 88 GB/hr → ★ 11 GB/hr

 ⇒ ★ THE SHARDING PROJECT WAS CANCELLED. For 18 months.
 ⇒ ★ THIS IS THE MOST COMMON OUTCOME OF AN HONEST EXHAUSTION
   CHECKLIST, AND IT IS WHY THE CHECKLIST EXISTS.
```

**Step 4 — 18 months later, the ceiling is real.**

```
 growth: 8,400 → 31,000 tenants; 4.1 TB → 22 TB
   CPU                 ★ 89% on a 192-core machine (the largest
                         instance available)
   write throughput    ★ 148,000/sec
   ★ vertical scaling exhausted — no bigger instance exists
   ★ functional split already done: events, analytics and sessions
     are separate databases
 ⇒ ★ NOW SHARD.
```

**Step 5 — choose the key, with measurement.**

```sql
SELECT tenant_id, count(*) AS shipments,
       round(100.0*count(*)/sum(count(*)) OVER (), 2) AS pct
  FROM shipments GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
```
```
 tenant_id | shipments  |  pct
-----------+------------+-------
   ★ 1204  | 1884201188 | ★ 31.4
   ★ 8842  |  884201188 | ★ 14.7
      4120 |  212048820 |   3.5
      9902 |  188402118 |   3.1
   ★ TWO TENANTS ARE 46% OF ALL DATA. Classic power law.
```

```sql
-- ★ how many queries actually carry tenant_id?
SELECT
  count(*) FILTER (WHERE query ILIKE '%tenant_id%') AS with_key,
  count(*) AS total,
  round(100.0*count(*) FILTER (WHERE query ILIKE '%tenant_id%')/count(*),1) AS pct
  FROM pg_stat_statements WHERE query ILIKE '%shipments%';
```
```
 with_key | total |  pct
----------+-------+-------
      142 |   148 | ★ 95.9
   ★ 95.9% of query shapes carry tenant_id. Property ① satisfied.
```

```
 ★ THE DESIGN THAT FOLLOWS FROM THE MEASUREMENTS:
   ① VIRTUAL BUCKETS: bucket = murmur3(tenant_id) % 4096
   ② A DIRECTORY for the two giant tenants:
      tenant 1204 → dedicated shards, sub-sharded by shipment_id
      tenant 8842 → its own dedicated shard
      everyone else → the bucket map over 12 shards
   ③ ★ tenant_id is IMMUTABLE (property ③) and co-locates
     everything about a tenant (property ④)
```

**Step 6 — build it.**

```sql
-- ★ the routing metadata, in an unsharded control database
CREATE TABLE shard_buckets (
  bucket   int  PRIMARY KEY CHECK (bucket BETWEEN 0 AND 4095),
  shard    text NOT NULL,
  state    text NOT NULL DEFAULT 'active'
             CHECK (state IN ('active','migrating','frozen')),
  version  bigint NOT NULL DEFAULT 1
);
CREATE TABLE shard_overrides (         -- ★ the directory for giants
  tenant_id bigint PRIMARY KEY,
  strategy  text   NOT NULL,           -- 'dedicated' | 'subsharded'
  shards    text[] NOT NULL
);
CREATE TABLE tenant_registry (         -- ★ global uniqueness
  tenant_slug text PRIMARY KEY,
  tenant_id   bigint NOT NULL UNIQUE
);
```

```js
// ★ the router
function shardFor(tenantId, subKey) {
  const override = overrides.get(tenantId);
  if (override) {
    if (override.strategy === 'dedicated') return override.shards[0];
    // ★ sub-sharded: a second level of hashing within the tenant
    return override.shards[murmur3(String(subKey)) % override.shards.length];
  }
  const bucket = murmur3(String(tenantId)) % 4096;
  const b = bucketMap[bucket];
  if (b.state === 'frozen')
    throw new RetryableError('BUCKET_MIGRATING');   // ★ brief, bounded
  return b.shard;
}

// ★ every data-access function takes the tenant. Enforced by lint.
async function listShipments(tenantId, filters) {
  return withShard(shardFor(tenantId), (pool) =>
    pool.query(`SELECT … FROM shipments
                 WHERE tenant_id=$1 AND created_at >= $2
                 ORDER BY created_at DESC LIMIT 50`,
               [tenantId, filters.since]));
}
```

```sql
-- ★ reference tables, replicated to every shard by logical replication
--   from the control database
CREATE PUBLICATION ref_pub FOR TABLE ref_countries, ref_currencies,
                                     ref_carriers, ref_service_levels;
-- on each shard:
CREATE SUBSCRIPTION ref_sub_shard_3
  CONNECTION 'host=control dbname=control' PUBLICATION ref_pub;

-- ★ and a checksum reconciler run hourly on every shard
SELECT md5(string_agg(code || name, ',' ORDER BY code)) FROM ref_carriers;
-- ★ must be identical everywhere; alert on divergence.
```

**Step 7 — the reshard procedure, exercised before it's needed.**

```js
// ★ move a set of buckets to a new shard, online
async function migrateBuckets(buckets, fromShard, toShard) {
  // ① mark migrating (reads still served by fromShard)
  await control.query(
    `UPDATE shard_buckets SET state='migrating' WHERE bucket = ANY($1)`, [buckets]);

  // ② logical replication of just these buckets' rows
  await startBucketReplication(fromShard, toShard, buckets);
  await waitForCatchup(toShard, { maxLagBytes: 1_000_000 });

  // ③ ★ freeze writes for THESE BUCKETS ONLY — a few seconds,
  //    affecting 1/N of traffic. Clients see a retryable error.
  await control.query(
    `UPDATE shard_buckets SET state='frozen', version=version+1
      WHERE bucket = ANY($1)`, [buckets]);
  await sleep(500);                       // let in-flight writes drain
  await waitForCatchup(toShard, { maxLagBytes: 0 });

  // ④ ★ verify before flipping — counts AND checksums
  const [a, b] = await Promise.all([
    bucketChecksum(fromShard, buckets),
    bucketChecksum(toShard, buckets)]);
  if (a !== b) {
    await control.query(
      `UPDATE shard_buckets SET state='active' WHERE bucket=ANY($1)`, [buckets]);
    throw new Error('checksum mismatch — aborted, no data moved');
  }

  // ⑤ flip and unfreeze
  await control.query(
    `UPDATE shard_buckets SET shard=$2, state='active', version=version+1
      WHERE bucket = ANY($1)`, [buckets, toShard]);
  await broadcastMapVersion();

  // ⑥ ★ soak, THEN delete from the source
  await sleep(3600_000);
  await deleteBuckets(fromShard, buckets);
}
```
```
 ★ MEASURED, 12 → 16 shards, 22 TB:
   buckets moved       1,024 of 4,096  (25%)
   online copy         ★ 41 hours, no user impact
   freeze window       ★ 3.8 seconds per 64-bucket batch
   total frozen time   ★ 16 batches × 3.8 s = 61 seconds,
                         each affecting 1.6% of tenants
   ⇒ ★ NO MAINTENANCE WINDOW. This is only possible because of
     the bucket layer.
```

**Step 8 — what the application had to change.**

```
 ★ THE PARTS PEOPLE UNDERESTIMATE:

 ① ★ 340 QUERIES AUDITED for whether they carry tenant_id.
    ★ 11 did not. Each needed either a rewrite or a documented
    scatter-gather with a latency budget.

 ② ★ EVERY MIGRATION RUNNER REWRITTEN to track per-shard state,
    resume from failure, and handle a shard being mid-reshard.

 ③ ★ CROSS-TENANT REPORTING moved out entirely — to a separate
    analytics warehouse fed by CDC (Topic 76). ★ Attempting
    scatter-gather aggregates over 16 shards was rejected after
    measuring p99 at 2,840 ms vs 84 ms on one shard.

 ④ ★ GLOBAL UNIQUENESS (tenant slugs, user emails) moved to the
    unsharded control database, with a reconciler for orphans.

 ⑤ ★ BACKUPS AND PITR became 16 independent timelines. Restoring
    to a consistent cross-shard point in time is ★ NOT POSSIBLE
    without coordinated snapshots — a genuinely hard problem that
    took a quarter to solve properly. (Topic 64.)

 ⑥ ★ ON-CALL: dashboards, alerts and runbooks × 16.
```

**Step 9 — results.**

| | Before the checklist | After the checklist | After sharding (16) |
|---|---|---|---|
| Database size | 18 TB | **4.1 TB** | 22 TB / 16 shards |
| CPU | 94% | **31%** | 38% per shard |
| Write throughput | 41,000/s | 44,000/s | **412,000/s** |
| p99 (single-tenant) | 4,200 ms | **84 ms** | 78 ms |
| p99 (cross-tenant) | 4,200 ms | 180 ms | ★ **2,840 ms** — moved to the warehouse |
| Servers to operate | 1 + 2 replicas | 1 + 2 replicas | **16 + 32 replicas + control** |
| Time to first fix | — | **3 weeks** | ★ **7 months** |

```
 ★ FIVE LESSONS:
 ① ★ THE EXHAUSTION CHECKLIST BOUGHT 18 MONTHS IN 3 WEEKS.
   Four fixes — HOT updates, one index, partitioned retention,
   a sharded counter — took CPU from 94% to 31% and the database
   from 18 TB to 4.1 TB.
 ② ★ 77% OF THE "DATA" WAS EXPIRED ROWS A DELETE HAD NEVER
   MANAGED TO REMOVE. "We have 18 TB" was not true.
 ③ ★ THE POWER LAW DECIDED THE ARCHITECTURE. Two tenants at 46%
   of data made plain hash sharding impossible; the directory +
   bucket design was forced by measurement, not preference.
 ④ ★ THE BUCKET LAYER TURNED A RESHARD FROM A PROJECT INTO A
   PROCEDURE — 61 seconds of total frozen time across a 22 TB
   redistribution.
 ⑤ ★ THE COSTS PEOPLE FORGET ARE OPERATIONAL: 16 backup
   timelines with no cross-shard consistent restore point, ×16
   dashboards, ×16 migrations, and cross-tenant reporting moved
   to an entirely separate system.
```

---

## Common mistakes

**1. Sharding before exhausting the checklist.**
- *Symptom:* a distributed system built to avoid writing an index.
- *Fix:* work all five items. In the example, three weeks of fixes bought 18 months.

**2. Choosing the shard key without measuring distribution.**
- *Symptom:* one shard holds 45% of the data and saturates first; you have 8 servers and the throughput of 2.
- *Fix:* run the hash-distribution query on real data *before* building anything.

**3. `hash(key) % N` with no bucket layer.**
- *Symptom:* adding one shard remaps ~80% of rows, with no incremental path.
- *Fix:* virtual buckets (a fixed 4096) mapped to shards. Build this from day one.

**4. A mutable shard key.**
- *Symptom:* changing a value requires moving a row between shards — a cross-shard transaction.
- *Fix:* the key must be immutable. `user_id`, `tenant_id` — not `region` or `status`.

**5. Assuming `UNIQUE` still works.**
- *Symptom:* duplicate emails on different shards.
- *Fix:* shard by the unique column, or an unsharded registry written first — plus a reconciler for orphans.

**6. Using `bigserial` after sharding.**
- *Symptom:* colliding ids.
- *Fix:* Snowflake or UUIDv7 (Topic 22).

**7. Cross-shard writes treated as transactions.**
- *Symptom:* money destroyed on a crash between two shard writes.
- *Fix:* co-locate transacting entities; otherwise a saga with compensations (Topic 52).

**8. Scatter-gather for aggregates.**
- *Symptom:* p99 bounded by the slowest shard; a 16-shard scatter's p99 ≈ one shard's p99.994.
- *Fix:* precomputed rollups, or a separate analytics store fed by CDC.

**9. Partial scatter failures returning wrong answers.**
- *Symptom:* a count that is silently short because one shard timed out.
- *Fix:* decide explicitly — fail the request, or return a flagged partial result. Never silently.

**10. Expecting sharding to fix a hot key.**
- *Symptom:* one key takes 40% of writes, lands on one shard, and that shard is still the bottleneck.
- *Fix:* Topic 61. Sharding distributes *keys*, not load within a key.

**11. Forgetting the operational multiplication.**
- *Symptom:* 16 backup timelines with no consistent cross-shard restore point, discovered during an incident.
- *Fix:* budget for backups, migrations, dashboards, alerts and runbooks × N — before committing.

**12. Not exercising the reshard before you need it.**
- *Symptom:* the first reshard happens under capacity pressure, untested.
- *Fix:* move a small bucket set in production, on purpose, early.

---

## Hands-on proof

**PROVE IT #1–#8 — Example 1** (the power-law distribution, simulating hash placement before committing and finding 45% on one shard, buckets making the giant tenant movable, duplicate emails across shards, colliding `bigserial` ids, Snowflake ids with a recoverable shard, scatter-gather bounded by the maximum, and a cross-shard "transaction" that isn't one).

**PROVE IT #9 — resharding cost with and without buckets.**
```sql
-- naive hash: how many rows change shard when N goes 4 → 5?
SELECT round(100.0*count(*) FILTER (
         WHERE (hashtext(customer_id::text)&2147483647)%4
            <> (hashtext(customer_id::text)&2147483647)%5) / count(*), 1) AS pct_moved
  FROM orders;
```
```
 pct_moved
-----------
   ★ 79.8
```
```sql
-- ★ with buckets: only the buckets you choose to move
SELECT round(100.0*count(*) FILTER (
         WHERE (hashtext(customer_id::text)&2147483647)%4096
               BETWEEN 3277 AND 4095) / count(*), 1) AS pct_moved
  FROM orders;
```
```
 pct_moved
-----------
   ★ 20.1        — and you choose which, in batches
```

**PROVE IT #10 — scatter-gather tail latency, empirically.**
```js
const N = 16, p99Single = 12;   // ms
// probability all N are under p99
console.log('P(all fast):', Math.pow(0.99, N).toFixed(3));   // 0.851
// ⇒ ★ 14.9% of scatter queries exceed a single shard's p99.
//   The scatter's p99 corresponds to ~p99.994 of one shard.
```

**PROVE IT #11 — reference-table divergence detection.**
```bash
for p in 5550 5551 5552 5553; do
  psql -p $p -d shop -tAc \
    "SELECT md5(string_agg(code||name, ',' ORDER BY code)) FROM ref_carriers"
done | sort -u | wc -l
```
```
 ★ 1        — all shards agree. Any other number is an alert.
```

**PROVE IT #12 — no cross-shard consistent restore point.**
```bash
for i in 0 1 2 3; do
  psql -p 555$i -tAc "SELECT pg_current_wal_lsn()"
done
```
```
 4A/8C001220
 2B/14002880
 7C/2A004410
 1D/88001040
   ★ FOUR INDEPENDENT TIMELINES. There is no single LSN, and no
     single point in time, that all four can be restored to.
   ⇒ ★ cross-shard PITR requires coordinated snapshots or a
     global transaction log. Budget for it. (Topic 64.)
```

---

## The design decision framework

```
★★★ SHARDING IS THE ONLY THING THAT SCALES WRITES,
    AND THE ONLY THING YOU CANNOT UNDO. ★★★

 ① ★ EXHAUST EVERYTHING ELSE — AND WRITE DOWN WHAT EACH BOUGHT
    ① vertical scaling (128 cores / 4 TB RAM is available)
    ② indexes                         ★ usually 100–1,000×
    ③ fix N+1                         (66)
    ④ ★ retention via partitioning    (59) — often removes 70–90%
    ⑤ caching / CDN                   (57)
    ⑥ read replicas                   (58)
    ⑦ ★ fix hot rows                  (61) — sharding does NOT
    ⑧ ★ functional split by service — much simpler than sharding
    ⇒ ★ IN THE EXAMPLE, THREE WEEKS OF THIS BOUGHT 18 MONTHS.

 ② PROVE THE CEILING IS REAL
    ✓ writes/sec AND WAL bytes/sec at the hardware limit
    ✓ ★ data size is REAL, not bloat or failed retention
    ✓ working-set hit ratio < 99% even with more RAM
    ✓ ★ it is not a connection-pooling problem (65)
    ✓ ★ it is not ONE hot key (61)

 ③ CHOOSE THE KEY — ALL FOUR PROPERTIES, MEASURED
    ① ★ present in > 95% of queries — count them in
       pg_stat_statements
    ② ★ evenly distributed — ★ RUN THE HASH-PLACEMENT QUERY ON
       REAL DATA. Top entity > 5% ⇒ plain hash is not viable.
    ③ ★ immutable
    ④ ★ co-locates related data
    ⇒ power-law tenancy ⇒ ★ a DIRECTORY + dedicated shards for
      giants + buckets for the rest.

 ④ ★ USE VIRTUAL BUCKETS. ALWAYS.
    bucket = hash(key) % 4096   (fixed forever)
    shard  = bucket_map[bucket] (a small, versioned table)
    ⇒ 4→5 shards moves ~20% instead of ~80%, in batches you choose
    ⇒ ★ this single decision determines whether resharding is a
      procedure or a project.

 ⑤ PLAN FOR WHAT BREAKS — BEFORE, NOT AFTER
    ✓ ★ global uniqueness → an unsharded registry + a reconciler
    ✓ ★ ids → Snowflake / UUIDv7 (22)
    ✓ ★ cross-shard writes → co-locate, or sagas (52)
    ✓ ★ cross-shard joins → REFERENCE TABLES replicated everywhere
        + a checksum reconciler
    ✓ ★ aggregates → rollups or a separate analytics store (76)
    ✓ ★ migrations → a per-shard-state runner that resumes
    ✓ ★ backups → N timelines, ★ NO cross-shard consistent restore
        point without coordinated snapshots (64)

 ⑥ AUDIT EVERY QUERY
    ★ does it carry the shard key? Count them.
    ⇒ < 95% single-shard ⇒ ★ the key is wrong. Stop and rechoose.
    ⇒ each scatter-gather query needs a documented latency budget
      and an explicit partial-failure policy.

 ⑦ EXERCISE THE RESHARD EARLY
    move a small bucket set in production, deliberately, before
    you are under pressure.
    ⇒ ★ verify with CHECKSUMS, not counts, before flipping.
    ⇒ ★ soak before deleting from the source.

 ⑧ BUDGET THE OPERATIONAL COST HONESTLY
    ★ N× dashboards, alerts, runbooks, migrations, backups,
      on-call surface — and a control plane that is itself a
      new single point of failure.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
On a real dataset: (a) measure the distribution of your candidate shard key and report the top-5 percentages; (b) simulate placement with `hash % 8` and report per-shard percentages; (c) compute what fraction of rows would move going from 4 to 5 shards with naive hashing, and with a 4096-bucket map.

### Exercise 2 — medium (apply it)
Stand up four local PostgreSQL instances as shards. Then demonstrate and fix each of: (a) duplicate values across shards where `UNIQUE` should have prevented it; (b) colliding `bigserial` ids; (c) a cross-shard "transaction" losing money on a simulated crash; (d) a scatter-gather aggregate whose latency equals the slowest shard.

For (a) build the registry-table approach including the compensation path and explain what a crash between the two writes leaves behind.

### Exercise 3 — hard (production simulation)
A B2B logistics SaaS proposes sharding by `tenant_id` into 16 shards. The primary is at 94% CPU, 41,000 writes/sec, 18 TB.

(a) Work the exhaustion checklist. From `pg_stat_statements`, identify the three findings that sharding would not fix, and quantify each.
(b) The HOT update ratio is 0.9%. Explain the mechanism and the fix, with the WAL impact.
(c) 77% of the 18 TB turns out to be expired data. Explain how, and what it means for the "we need to shard" claim.
(d) After the fixes, CPU is 31% and the database is 4.1 TB. Write the one-paragraph recommendation you'd give the team.
(e) Eighteen months later the ceiling is genuine. Measure the tenant distribution — two tenants are 46% of data. Explain precisely why plain hash sharding is now impossible.
(f) Design the directory + bucket architecture. Give the control-plane schema and the routing function, including the sub-sharding of the largest tenant.
(g) Explain why the bucket count must be fixed forever, and why 4096.
(h) Write the online reshard procedure. Explain why you verify with checksums rather than counts, and why you soak before deleting.
(i) 11 of 340 queries do not carry the shard key. For each of three plausible examples, decide: rewrite, scatter-gather with a budget, or move to the analytics store.
(j) Explain why there is no cross-shard consistent restore point, and what you would build to get one.
(k) List the operational costs that multiply by 16, and estimate the on-call impact.

---

## Mental model checkpoint

1. State the difference between partitioning and sharding in one sentence each.
2. Name the five items on the exhaustion checklist. Which one does sharding explicitly *not* fix?
3. Name the four required properties of a shard key. Which is most often violated in B2B systems?
4. Why does `hash(key) % N` make resharding nearly impossible? What fixes it?
5. Why 4096 buckets, and why must the count never change?
6. Name five things that break when you shard, and the mitigation for each.
7. Why is a 16-shard scatter-gather's p99 so much worse than one shard's p99?
8. Why is a partial scatter failure more dangerous than a total one?
9. Explain the online reshard procedure and where the write freeze occurs.
10. Why is there no cross-shard consistent point-in-time restore?
11. What is a reference table, and what rule must it obey?

---

## Quick reference card

**★ Exhaust first:** vertical scaling · indexes · N+1 · **partitioned retention** · caching · replicas · **hot rows** · **functional split**.

**Shard key — all four required**
```
① present in > 95% of queries   ② ★ evenly distributed (measure!)
③ ★ immutable                    ④ ★ co-locates related data
```
```sql
-- ★ RUN THIS BEFORE BUILDING ANYTHING
SELECT (hashtext(key::text) & 2147483647) % 8 AS shard, count(*),
       round(100.0*count(*)/sum(count(*)) OVER (),1) AS pct
  FROM t GROUP BY 1 ORDER BY 1;      -- ★ any shard > 20% ⇒ rethink
```

**★ Always use virtual buckets**
```js
bucket = murmur3(key) % 4096;   // ★ fixed forever
shard  = bucketMap[bucket];     // ★ a small versioned table
// 4→5 shards: naive hash moves ★ 80%; buckets move ★ 20%, in batches
```

**What breaks → mitigation**

| Breaks | Mitigation |
|---|---|
| global `UNIQUE` | ★ unsharded registry + reconciler |
| sequences | ★ Snowflake / UUIDv7 |
| cross-shard txns | co-locate · sagas (52) · 2PC (51) |
| cross-shard joins | ★ reference tables + checksum reconciler |
| aggregates | ★ rollups / analytics store (76) |
| migrations | per-shard state, resumable |
| backups | ★ N timelines, **no consistent cross-shard PITR** |

**★ Scatter-gather:** latency = the **slowest** shard. P(all 16 fast) = `0.99¹⁶` = **85%**. A partial failure is a **wrong answer**, not an error.

**Reshard:** mark migrating → logical-replicate buckets → **freeze those buckets (~4 s)** → **verify checksums** → flip the map → soak → delete source.

**★ Sharding does not fix a hot key** (Topic 61) · **does not fix a missing index** · **does not fix bloat**.

---

## When would I use this at work?

1. **Almost never — and that's the point.** The exhaustion checklist is the deliverable. In the example it took three weeks and bought eighteen months, and the four fixes (HOT updates, one index, partitioned retention, a sharded counter) were things the team should have done anyway.

2. **When you genuinely cannot buy a bigger machine.** 192 cores and 4 TB of RAM is the current practical ceiling. If you're there, sustained, with a real working set and no retention debt, then shard — and the measurement work above is what makes it survivable.

3. **The moment anyone proposes a shard key.** Ask for the distribution query on real data, and the count of query shapes that carry it. Both take five minutes and both routinely kill a proposal — a top tenant at 45% of data means eight servers with the throughput of two.

4. **When choosing between building it and adopting Citus/Vitess.** These exist because the bucket layer, the reshard procedure, the reference tables and the per-shard migration runner are genuinely hard to get right. Building your own is defensible; doing so without knowing what you're reimplementing is not.

---

## Connected topics

**Understand before this:** 59 (partitioning — the single-server answer to try first), 51 (2PC — why cross-shard atomicity is expensive), 52 (sagas and idempotency — the alternative), 22 (id generation — Snowflake/UUIDv7), 54 (the gates — most sharding proposals fail gate 2).

**This unlocks:**
- **61** — counters and hot rows: the problem sharding cannot solve
- **68** — CAP and consistency models: the formal framing
- **64** — backup and PITR: why N shards means N timelines
- **76** — polyglot persistence and CDC: where cross-shard analytics goes
- **Case study 15** — multi-tenant SaaS, where power-law tenancy is the central problem
