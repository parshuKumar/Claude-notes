# 00 — Production Database Design Case Studies
## Real systems · Real traffic · Real failure modes

---

## WHAT THIS FOLDER IS

The `/docs/databases/` curriculum teaches you **the machinery**: pages, B-trees, MVCC, isolation levels, sharding.

This folder teaches you **the judgement**. Each case study is a complete, realistic system design where you:

1. Get a **requirements brief** with actual traffic numbers (not "it should scale")
2. See the **naive schema** that a competent developer would write first
3. Watch it **break at a specific scale**, with the exact query, the exact `EXPLAIN`, the exact lock trace, and the exact metric that shows it failing
4. Work through the **redesign**, decision by decision, with the trade-off named each time
5. See the **final production schema** with every index, constraint, partition, and denormalisation justified
6. Face the **hard questions** — what breaks next, what you'd monitor, what you'd do at 10× this load

These are written so you can *do this yourself* on a system nobody has written a case study for. That's the actual goal: not memorising these 20 designs, but internalising the method that produced them.

---

## HOW TO USE THIS FOLDER

**Read them in order the first time.** Case study 01 establishes the method and the vocabulary; later ones reference it.

**Then use them as a drill.** For each one:

```
1. Read ONLY the "Requirements brief" section.
2. Close the file. Design the schema yourself. Write the DDL. Pick the keys,
   the indexes, the partition strategy, the isolation level.
3. Write down the three queries you think will be slowest and why.
4. NOW read the rest.
5. Score yourself: what did you miss? Which failure mode did you not see coming?
```

The gap between your design and the final one is your actual curriculum.

**The pass/fail bar for each case study** is at the bottom of every file: seven questions you must be able to answer without looking. If you can answer all seven for all twenty, you can design a database for almost anything.

---

## THE METHOD (used identically in every case study)

Every design in this folder follows the same eight steps. Learn the steps, not the answers.

```
STEP 1 — ENUMERATE THE ACCESS PATTERNS BEFORE THE SCHEMA
   Write down every query the system will run, with:
     • calls per second (peak, not average)
     • rows read per call
     • rows written per call
     • latency budget (p99)
     • consistency requirement (can it be stale? by how long?)
   ⚠ If you cannot fill this table in, you are not ready to design the schema.

STEP 2 — IDENTIFY THE INVARIANTS
   What must NEVER be false, even for a microsecond?
     "stock >= 0"   "sum(ledger entries) = 0"   "one seat, one booking"
   These decide your isolation level, your constraints, and your transaction
   boundaries. Everything else is negotiable; these are not.

STEP 3 — FIND THE HOT PATH AND THE HOT ROW
   Which single query runs most? Which single ROW is contended most?
   The hot row is where every high-traffic system dies. Find it early.

STEP 4 — DESIGN NORMALISED FIRST
   3NF/BCNF. Every fact in one place. This is your correctness baseline
   and your source of truth. You will denormalise FROM it, deliberately,
   with evidence — never instead of it.

STEP 5 — CHOOSE KEYS AND INDEXES FROM STEP 1
   Every index must map to a line in your access-pattern table.
   An index with no query behind it is pure write tax.

STEP 6 — FIND WHERE IT BREAKS
   For each access pattern, compute pages read, lock hold time, and write
   amplification. The one that exceeds its budget first is your bottleneck.
   Fix that one. Then recompute. Repeat.

STEP 7 — SCALE OUT IN THE CORRECT ORDER
   index → query rewrite → cache → read replica → denormalise → partition
   → shard.  Each step is ~10× cheaper than the next. Never skip ahead.
   Sharding is the LAST resort, not the first idea.

STEP 8 — DESIGN THE FAILURE MODES
   What happens when the cache is cold? The replica lags 8 seconds? A
   partition is dropped mid-query? The queue redelivers a message twice?
   A design that hasn't answered these isn't finished.
```

---

## THE CASE STUDIES

### Tier 1 — Contention and correctness under extreme concurrency
*The hardest problems in database design: many writers, one row, money on the line.*

| # | File | System | Peak load | The core problem |
|---|---|---|---|---|
| 01 | `01-flash-sale-inventory.md` | E-commerce flash sale | 500k concurrent users, 80k req/s, 10k units | **Overselling.** Every user hits the same inventory row. Row-lock convoy, lost updates, and why `SELECT ... FOR UPDATE` alone collapses at this scale |
| 02 | `02-ticket-booking-seats.md` | Movie/train seat booking | 200k concurrent at open, 5M seats | **Seat holds.** Reserve-then-pay with expiry, no double-booking, no permanently locked seats when a user abandons checkout |
| 03 | `03-payment-ledger.md` | Payments + wallet ledger | 12k txn/s, zero tolerance | **Double-entry integrity.** Balances that can never drift, idempotent retries, exactly-once against an external gateway that lies |
| 04 | `04-subscription-billing.md` | SaaS metered billing | 40M usage events/day | **Proration and idempotency.** Usage aggregation, invoice immutability, currency, and recomputing history without rewriting it |

### Tier 2 — Write-heavy and fan-out systems
*Where the write rate, not the read rate, is what kills you.*

| # | File | System | Peak load | The core problem |
|---|---|---|---|---|
| 05 | `05-social-feed-fanout.md` | Social timeline | 80M users, 300k reads/s, 20k writes/s | **Fan-out on write vs read.** The celebrity problem: one post → 40M timeline rows. Hybrid fan-out and how to pick the threshold |
| 06 | `06-chat-messaging.md` | Team chat / DM | 2M concurrent sockets, 100k msg/s | **Unbounded growth + per-user read state.** Message ordering, `last_read` per user per channel, and why the read-receipt table is bigger than the messages table |
| 07 | `07-iot-telemetry.md` | Device telemetry | 400k writes/s, 5 TB/month | **Ingest rate vs query rate.** Time-series partitioning, compression, downsampling, and retention as `DROP TABLE` |
| 08 | `08-audit-log-compliance.md` | Immutable audit trail | 60k events/s, 7-year retention | **Append-only at scale with legal retention.** Tamper evidence, partition lifecycle, and querying 7 years without scanning 7 years |
| 09 | `09-notification-delivery.md` | Push/email/SMS pipeline | 5M notifications/hour | **The database-as-a-queue trap.** Job claiming without lock convoys, retries, dedup, and when to stop using PostgreSQL for this |

### Tier 3 — Read-heavy and latency-critical systems
*Where the working set, the cache, and the query plan decide everything.*

| # | File | System | Peak load | The core problem |
|---|---|---|---|---|
| 10 | `10-product-catalog-search.md` | Catalogue + search + facets | 8M SKUs, 120k req/s | **Variable attributes and faceted filtering.** EAV vs JSONB vs wide table, and where the search engine boundary sits |
| 11 | `11-ride-hailing-dispatch.md` | Ride matching | 200k drivers, 50k rides/min | **Geospatial + real-time state.** "Nearest available driver," location updates at 1 Hz per driver, and why you don't store live location in PostgreSQL |
| 12 | `12-food-delivery-orders.md` | Order lifecycle + tracking | 3M orders/day, 15 state transitions | **State machines at scale.** Status updates as write amplification, order history, and multi-party visibility (customer/restaurant/rider) |
| 13 | `13-gaming-leaderboard.md` | Real-time leaderboards | 10M players, 200k score writes/s | **Ranked reads on constantly-changing data.** Why `ORDER BY score LIMIT 100` is fatal, sorted sets, and periodic materialisation |
| 14 | `14-ad-serving-rtb.md` | Ad serving + budget pacing | 800k req/s, 40ms budget | **Budget enforcement at microsecond scale.** Distributed counters, over-delivery tolerance, and eventual consistency where money is involved but bounded |

### Tier 4 — Structural and organisational scale
*Where the problem is the shape of the data, the tenancy model, or the org.*

| # | File | System | Peak load | The core problem |
|---|---|---|---|---|
| 15 | `15-multi-tenant-saas.md` | B2B SaaS platform | 40k tenants, 1 whale = 40% of data | **Tenancy isolation.** Shared table vs schema-per-tenant vs DB-per-tenant, noisy neighbours, and per-tenant backup/restore |
| 16 | `16-warehouse-inventory-oms.md` | Multi-warehouse inventory | 200 warehouses, 4M SKUs | **Distributed stock truth.** Available vs reserved vs in-transit, allocation across locations, and reconciliation with physical reality |
| 17 | `17-healthcare-ehr.md` | Patient records | 20M patients, strict audit | **Temporal data + access control.** Bitemporal records (what we knew vs when it was true), row-level security, and never deleting anything |
| 18 | `18-logistics-tracking.md` | Parcel tracking | 40M parcels in flight | **High-cardinality state + event sourcing.** Scan events, current-state projection, and answering "where is it" in 10ms over 8B events |
| 19 | `19-video-platform-metadata.md` | Video platform | 500M videos, 2B views/day | **View counts and the hottest rows on the internet.** Counter sharding, approximate counts, and separating metadata from engagement |
| 20 | `20-analytics-warehouse.md` | Product analytics | 12B events, sub-second dashboards | **OLTP/OLAP separation done properly.** Star schema, CDC pipeline, pre-aggregation, and why the answer is never "add an index" |

### Tier 5 — Capstone
| # | File | What it is |
|---|---|---|
| 21 | `21-capstone-design-your-own.md` | Six unseen briefs with no solutions provided — plus a self-scoring rubric and the reference answers in a separate collapsed section |
| 22 | `22-debugging-drills.md` | Fifteen production incidents: you get the symptom, the metrics, and the schema. You diagnose. Answers separated |
| 23 | `23-schema-review-checklist.md` | The checklist to run against any schema in 20 minutes — the condensed output of all 20 case studies |

---

## THE ANATOMY OF EACH CASE STUDY

Every file follows this structure exactly:

```
1.  THE BRIEF                  requirements + hard traffic numbers
2.  ACCESS PATTERN TABLE       every query, qps, rows, latency budget, staleness
3.  THE INVARIANTS             what must never be false
4.  ATTEMPT 1 — THE NAIVE SCHEMA
      the DDL a good developer writes first — genuinely reasonable
5.  WHERE IT BREAKS
      the exact query, the exact EXPLAIN, the lock trace, the metric,
      the load number at which it fails
6.  ATTEMPT 2 — THE FIX THAT ISN'T ENOUGH
      the obvious fix, why it helps, and the NEW wall it hits
7.  THE PRODUCTION DESIGN
      full DDL: keys, indexes, constraints, partitions, isolation level
      + a justification line for EVERY object
8.  THE APPLICATION CODE THAT GOES WITH IT
      Node.js — transaction boundaries, retry logic, the exact SQL
9.  THE NUMBERS
      before/after: pages read, lock hold ms, p99, storage, write amplification
10. FAILURE MODES AND WHAT YOU MONITOR
      cold cache · replica lag · partition drop · duplicate delivery
      · the 3 alerts that would page you
11. WHAT BREAKS AT 10×
      the next wall, and the design you'd move to
12. THE SEVEN QUESTIONS
      pass/fail — answer without looking
13. TRANSFERABLE LESSONS
      the 3–5 patterns here that apply to systems that look nothing like this
```

---

## PREREQUISITE MAP

You can read the case studies in parallel with the curriculum, but each one leans on specific topics. If a case study loses you, this table tells you which doc to go back to.

| Case study | Leans hardest on |
|---|---|
| 01 Flash sale | 45 locks · 46 MVCC · 48 deadlocks · 49 optimistic/pessimistic · 61 hot rows |
| 02 Ticket booking | 44 isolation · 50 SSI · 24 constraints (EXCLUDE) · 39 transactions |
| 03 Payment ledger | 40 ACID · 52 idempotency · 24 constraints · 51 2PC |
| 04 Subscription billing | 26 time & money · 25 data types · 56 materialised views |
| 05 Social feed | 55 denormalisation patterns · 57 caching · 60 sharding · 70 documents |
| 06 Chat | 22 primary keys (ULID) · 59 partitioning · 61 hot rows |
| 07 IoT telemetry | 73 time-series · 59 partitioning · 06 columnar · 16 BRIN |
| 08 Audit log | 59 partitioning · 69 security · 08 LSM |
| 09 Notifications | 45 locks (SKIP LOCKED) · 52 outbox · 03 data models |
| 10 Catalogue | 27 antipatterns (EAV) · 16 GIN/JSONB · 74 search |
| 11 Ride hailing | 16 GiST · 03 data models · 57 caching |
| 12 Food delivery | 05 tuple layout · 17 write amplification · 47 VACUUM |
| 13 Leaderboard | 61 hot rows · 71 Redis sorted sets · 56 matviews |
| 14 Ad serving | 61 counters · 68 CAP · 57 caching |
| 15 Multi-tenant | 27 patterns · 69 RLS · 59 partitioning · 64 backups |
| 16 Inventory | 44 isolation · 24 constraints · 51 distributed txn |
| 17 Healthcare | 26 temporal · 69 RLS · 36 5NF |
| 18 Logistics | 55 denormalisation · 76 CDC · 59 partitioning |
| 19 Video platform | 61 hot rows · 57 caching · 58 replicas |
| 20 Analytics | 06 columnar · 20-21 ER · 76 CDC · 56 matviews |

---

## THE STANDING RULES USED IN EVERY DESIGN

These appear so often they're stated once here instead of in every file:

```
1.  MONEY IS bigint IN THE SMALLEST UNIT.
    ₹2,499.00 → 249900 paise. Never float. Never numeric on a hot path.

2.  TIMESTAMPS ARE timestamptz, STORED IN UTC.
    Never `timestamp`. The application converts at the edge.

3.  THE SOURCE OF TRUTH IS A TRANSACTIONAL DATABASE.
    Everything else — cache, search index, analytics store, materialised
    view — is DERIVED and REBUILDABLE. If it isn't rebuildable, it's a
    second source of truth and you now have a distributed-transaction problem.

4.  EVERY INDEX MAPS TO A LINE IN THE ACCESS-PATTERN TABLE.
    No exceptions. Unused indexes are write tax + bloat + planner confusion.

5.  NEVER HOLD A TRANSACTION OPEN ACROSS A NETWORK CALL YOU DON'T OWN.
    Not a payment gateway, not an S3 upload, not an HTTP webhook.

6.  IDEMPOTENCY KEYS ON EVERY EXTERNALLY-TRIGGERED WRITE.
    Enforced by a UNIQUE constraint, not by a SELECT-then-INSERT.

7.  RETENTION IS `DROP PARTITION`, NOT `DELETE`.
    Any table with a time-based retention policy is partitioned by time
    from day one.

8.  THE HOT ROW IS THE ENEMY.
    Any design where thousands of concurrent transactions update ONE row
    will serialise. Find it in the design phase, not in the incident.

9.  ISOLATION LEVEL IS A DESIGN DECISION, WRITTEN DOWN.
    Not a default you inherited. Each case study states its level and why.

10. NAMES ARE REALISTIC.
    users, orders, order_items, products, payments, sessions, inventory.
    Never foo/bar. You should be able to paste this into a real project.
```

---

## PROGRESS TRACKER

### Tier 1 — Contention and correctness
- [x] 01 — Flash sale inventory
- [x] 02 — Ticket booking / seat reservation
- [x] 03 — Payment ledger
- [x] 04 — Subscription billing  ← **TIER 1 COMPLETE**

### Tier 2 — Write-heavy and fan-out
- [x] 05 — Social feed fan-out
- [x] 06 — Chat / messaging
- [ ] 07 — IoT telemetry
- [ ] 08 — Audit log
- [ ] 09 — Notification delivery

### Tier 3 — Read-heavy and latency-critical
- [ ] 10 — Product catalogue and search
- [ ] 11 — Ride-hailing dispatch
- [ ] 12 — Food delivery orders
- [ ] 13 — Gaming leaderboard
- [ ] 14 — Ad serving / RTB

### Tier 4 — Structural scale
- [ ] 15 — Multi-tenant SaaS
- [ ] 16 — Warehouse inventory / OMS
- [ ] 17 — Healthcare EHR
- [ ] 18 — Logistics tracking
- [ ] 19 — Video platform metadata
- [ ] 20 — Analytics warehouse

### Tier 5 — Capstone
- [ ] 21 — Design your own (6 unseen briefs)
- [ ] 22 — Debugging drills (15 incidents)
- [ ] 23 — Schema review checklist

**Completed: 6 / 23**

---

## THE ENVIRONMENT

Every case study is reproducible on a laptop. Load generation uses `pgbench` with custom scripts so you can actually *watch* the failure modes happen rather than take my word for them.

```bash
docker run -d --name pg-lab \
  -e POSTGRES_PASSWORD=lab -p 5432:5432 \
  -c shared_buffers=1GB -c max_connections=200 \
  postgres:16

docker exec -it pg-lab psql -U postgres

# The load generator used throughout:
docker exec -it pg-lab pgbench -U postgres -f /scripts/flash_sale.sql \
  -c 200 -j 8 -T 60 -P 5 shop
```

Extensions used across the case studies:
`pageinspect` · `pgstattuple` · `pg_stat_statements` · `pg_buffercache` · `btree_gist` · `postgis` (case 11) · `pg_partman` (cases 07, 08, 18)

---

*Index version 1.0 — created 2026-08-07*
