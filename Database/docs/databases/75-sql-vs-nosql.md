# 75 — SQL vs NoSQL: The Decision
## Phase: Beyond Relational

---

## ELI5 — The Simple Analogy

Choosing a vehicle.

A car does most things adequately: shopping, commuting, moving a bookshelf, a road trip. It is not the fastest at any of them. **A Formula 1 car is dramatically better at exactly one thing and cannot go to the supermarket** — no boot, no reverse, and it needs a pit crew.

★ **The mistake is not choosing the F1 car. It is choosing it before you have measured how often you actually go to a racetrack** — and discovering, six months in, that you mostly do the shopping.

And the second mistake, which is subtler: ★ **assuming the car will *never* be fast enough.** A modern car does 200 km/h. Most people never find its limit. The people who genuinely need an F1 car know exactly why, in numbers.

★ **The honest version of this decision has three properties almost no real discussion has:** it names a *specific* alternative, it states a *measured* threshold, and it prices the *operational* cost — not just the query.

---

## Where this fits in the big picture

```
   70–74 — ★ each specialised model, on its own terms
   54 — the gates · 60 — the exhaustion checklist
   68 — the consistency vocabulary
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 75 SQL vs NoSQL ← YOU ARE HERE               │
        │ ★ the decision, with thresholds and costs    │
        └────────────────────┬─────────────────────────┘
                             ▼
              76 polyglot persistence & CDC
              → Phase 9 capstones
```

★ **This topic exists because the previous five each gave a partial answer.** Every one ended with "PostgreSQL covers most of this — here is where it stops." **This is where those stopping points are collected into a single decision procedure.**

---

## What is this?

A decision framework for **when a specialised data store is justified**, and — more often — **when it is not**.

```
 ★ FIRST, DISMANTLE THE QUESTION.
   "SQL vs NoSQL" is not a real dichotomy:

 ✗ ★ "NoSQL" IS NOT A CATEGORY.
    ⇒ Redis, MongoDB, Cassandra, Neo4j and Elasticsearch have
      ★ almost nothing in common. Grouping them is like grouping
      "vehicles that are not cars".

 ✗ ★ "SQL" IS NOT A DATA MODEL.
    ⇒ SQL is a query language. ★ PostgreSQL is a relational,
      document (jsonb), key-value (hstore), full-text, geospatial,
      time-series and vector store. ★ The model is not the language.

 ✗ ★ "NoSQL SCALES, SQL DOESN'T."
    ⇒ ★ PostgreSQL does 50,000+ writes/sec on one machine and
      10–50 TB. ★ Most systems never approach that (Topic 60).

 ✗ ★ "SCHEMALESS IS FASTER TO DEVELOP."
    ⇒ ★ the schema moved into every reader. It did not vanish
      (Topic 70). ★ You pay it later, with interest.

 ⇒ ★ THE REAL QUESTION, IN THREE PARTS:
   ① ★ WHICH ACCESS PATTERN does PostgreSQL genuinely handle
      badly, at MY measured scale?
   ② ★ WHAT SPECIFIC ALTERNATIVE handles it, and what does it
      cost operationally?
   ③ ★ WHAT DO I GIVE UP, and is that acceptable — in writing?
```

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE BOTH ERRORS ARE EXPENSIVE, AND THEY ARE ASYMMETRIC.

 ★ ERROR 1 — ADOPTING TOO EARLY (★ far more common)
   ⇒ ★ a second store to operate, back up, secure, monitor and be
     on call for
   ⇒ ★ a sync pipeline that is the real engineering cost (Topic 76)
   ⇒ ★ lost joins, lost transactions, lost constraints
   ⇒ ★ a team that now needs two sets of expertise
   ⇒ ★ AND: ★ it is very hard to undo. The data model is baked
     into every read path.

 ★ ERROR 2 — ADOPTING TOO LATE
   ⇒ ★ months of fighting a workload the database cannot serve
   ⇒ ★ but: ★ recoverable. You migrate when the evidence is clear.

 ⇒ ★ THE ASYMMETRY DECIDES THE DEFAULT:
   ★ START RELATIONAL. MOVE WHEN MEASURED. The cost of being late
   is a migration; ★ the cost of being early is a permanent tax.

 ★ AND THE ORGANISATIONAL FACT NOBODY PUTS IN THE DESIGN DOC:
   ★ every additional datastore multiplies the on-call surface,
   the backup surface, the security surface and the upgrade
   surface. ★ A three-store architecture needs roughly three times
   the operational maturity, not 1.2×.
```

---

## The physical reality

### What PostgreSQL actually does, measured

```
 ★ THE NUMBERS PEOPLE ARGUE WITHOUT KNOWING — on modern hardware
   (32–64 cores, NVMe, 256 GB–1 TB RAM):

   ★ simple writes            ★ 50,000–150,000 /sec
   ★ COPY bulk ingest         ★ 500,000–2,000,000 rows/sec
   ★ point reads (cached)     ★ 200,000+ /sec
   ★ with read replicas       ★ ~N× that
   ★ table size               ★ 10–50 TB comfortably
   ★ rows in one table        ★ billions, with partitioning
   ★ jsonb documents          ★ tens of millions
   ★ full-text corpus         ★ ~10M documents
   ★ connections              ★ 50–200 ACTIVE (Topic 65 —
                                more is SLOWER)

 ⇒ ★ AND THE POINT: ★ MOST SYSTEMS NEVER REACH ANY OF THESE.
   ★ The exhaustion checklist (Topic 60) exists because "we've
   outgrown Postgres" is usually "we have a missing index and a
   DELETE-based retention job".
```

### The five genuine limits — where PostgreSQL actually stops

```
 ★ ① WRITE THROUGHPUT BEYOND ONE MACHINE
    ⇒ ★ sustained >150k writes/sec after batching, index pruning,
      HOT tuning and partitioned retention
    ⇒ ★ AND vertical scaling exhausted (192 cores, 4 TB RAM)
    ⇒ ★ AND functional splitting already done
    ⇒ THEN: ★ Cassandra/Scylla (72), or ★ sharding (60)

 ★ ② DEEP OR VARIABLE-DEPTH GRAPH TRAVERSAL
    ⇒ ★ ≥5% of traffic at depth ≥5, or shortest-path between
      arbitrary nodes
    ⇒ ★ measured cliff: depth 4 = 41 s, depth 5 = OOM (Topic 74)
    ⇒ THEN: ★ Neo4j / a graph store

 ★ ③ SEARCH RELEVANCE, FACETS AND FUZZY AT SCALE
    ⇒ ★ >25M documents, or ★ multi-facet aggregation as a product
      requirement, or ★ measurable BM25 relevance gain
    ⇒ THEN: ★ Elasticsearch / OpenSearch (74)

 ★ ④ SUB-MILLISECOND ORDERED OPERATIONS
    ⇒ ★ rank, leaderboards, sliding-window rate limits at
      >50k ops/sec
    ⇒ ★ measured: `ZREVRANK` 0.19 ms vs 8,412 ms in SQL (71)
    ⇒ THEN: ★ Redis

 ★ ⑤ ANALYTICAL SCANS OVER BILLIONS OF ROWS
    ⇒ ★ aggregations over 10⁹+ rows, sub-second, ★ where rollups
      are insufficient because the queries are ad-hoc
    ⇒ THEN: ★ ClickHouse / DuckDB / a warehouse (73)

 ⇒ ★ NOTE WHAT IS NOT ON THIS LIST:
   ✗ "we have flexible schemas"       ⇒ ★ jsonb (70)
   ✗ "we have relationships"          ⇒ ★ joins (74)
   ✗ "we have lots of data"           ⇒ ★ partitioning (59)
   ✗ "we need to scale"               ⇒ ★ replicas (58), the
                                        exhaustion checklist (60)
   ✗ "we need high availability"      ⇒ ★ Patroni (63)
   ✗ "developers prefer it"           ⇒ ★ not an architecture
                                        decision
```

### The costs that never appear in the proposal

```
 ★ THE QUERY LANGUAGE IS THE CHEAPEST PART. THESE ARE THE REST:

 ★ ① THE SYNC PIPELINE (Topic 76)
    ⇒ ★ CDC or an outbox, an idempotent consumer, a reconciler,
      a lag alert, a TESTED full-rebuild procedure
    ⇒ ★ MEASURED: 4–8 weeks of engineering, and it is ★ ongoing
      maintenance, not a one-off.

 ★ ② LOST JOINS ⇒ DENORMALISATION ⇒ MORE SYNC
    ⇒ ★ Topic 74's finding: 88% of searches also filtered
      ⇒ ★ four more fields to denormalise and keep in agreement.

 ★ ③ LOST TRANSACTIONS
    ⇒ ★ every invariant that spanned tables now spans systems
    ⇒ ★ sagas and compensations (52), or accepted inconsistency

 ★ ④ LOST CONSTRAINTS
    ⇒ ★ FKs, CHECKs and UNIQUEs become application code —
      ★ and application code has code paths that forget (69)

 ★ ⑤ OPERATIONAL SURFACE ×N
    ⇒ ★ backups AND a tested restore · monitoring · alerting ·
      upgrades · security patching · capacity planning ·
      ★ on-call rotation · ★ runbooks
    ⇒ ★ AND: ★ N independent backup timelines with ★ no
      cross-store consistent restore point (60, 64)

 ★ ⑥ EXPERTISE
    ⇒ ★ every store has failure modes that only experience
      teaches: tombstones (72), cardinality (73), supernodes (74),
      `allkeys-lru` (71).
    ⇒ ★ a team of five cannot be expert in four datastores.

 ⇒ ★ THE HONEST MULTIPLIER: ★ a second store costs 2–4× the
   proposal's estimate, and ★ most of it is recurring.
```

### The migration cost asymmetry

```
 ★ ADDING A STORE IS EASY. REMOVING ONE IS NOT.

   adding      ⇒ ★ a sprint (a new client, a sync job, some reads)
   removing    ⇒ ★ every read path rewritten, every denormalised
                  field re-derived, ★ months
   ⇒ ★ AND THE DATA MODEL IS THE HARD PART:
     ★ a Cassandra partition key cannot be altered (72)
     ★ a DynamoDB key schema cannot be altered (71)
     ★ an embedded document shape is baked into every reader (70)

 ★ WHEREAS GOING THE OTHER WAY IS TRACTABLE:
   ★ PostgreSQL → Elasticsearch is a CDC pipeline and a reindex.
   ★ PostgreSQL → Cassandra is a dual-write and a backfill.
   ⇒ ★ BOTH ARE WELL-TRODDEN. The reverse is not.

 ⇒ ★ THEREFORE: ★ WHEN UNCERTAIN, CHOOSE THE ONE THAT IS EASIER
   TO LEAVE. ★ That is almost always PostgreSQL.
```

### The one-table summary

```
 ★ WHAT EACH STORE IS GENUINELY BEST AT — one line each:

 ★ PostgreSQL   ★ everything, adequately; ★ transactions, joins
                and constraints, ★ excellently
 ★ Redis        ★ sub-ms ordered/atomic operations on small values
 ★ Cassandra    ★ linear write scaling with a known access pattern
 ★ DynamoDB     ★ the same, managed, with predictable cost
 ★ MongoDB      ★ document-shaped data with a flexible schema
                ★ (and jsonb covers most of this)
 ★ Elasticsearch ★ relevance, facets and fuzzy matching at scale
 ★ Neo4j        ★ deep and variable-depth traversal
 ★ ClickHouse   ★ analytical scans over billions of rows
 ★ TimescaleDB  ★ time-series, ★ inside PostgreSQL
 ★ Prometheus   ★ operational metrics (★ never business data)
 ★ S3           ★ large immutable blobs — ★ and the one most often
                forgotten: ★ don't put a 400 KB PDF in any of the
                above.
```

---

## How it works — step by step

### The decision procedure

```
 ★ SEVEN GATES. ALL MUST PASS BEFORE ADOPTING ANYTHING.

 ★ GATE 1 — IS THERE A MEASURED PROBLEM?
   □ a named query or operation, with p50/p99/p999
   □ a stated SLO it misses
   □ ★ Topic 67's levels 1–2 completed: ★ is it even the database?
   ⇒ ★ "we might need to scale" fails here.

 ★ GATE 2 — HAS THE EXHAUSTION CHECKLIST BEEN RUN? (Topic 60)
   □ indexes (54) □ N+1 (66) □ ★ retention via partitioning (59)
   □ caching/CDN (57) □ replicas (58) □ ★ hot rows (61)
   □ ★ connection pooling (65) □ ★ vertical scaling
   □ ★ FUNCTIONAL SPLIT (a separate database per workload)
   ⇒ ★ in Topic 60's example this bought 18 months in 3 weeks.

 ★ GATE 3 — IS IT ONE OF THE FIVE GENUINE LIMITS?
   write throughput · deep traversal · search relevance ·
   sub-ms ordered ops · analytical scans
   ⇒ ★ if not, ★ STOP. The answer is a PostgreSQL feature.

 ★ GATE 4 — NAME THE SPECIFIC ALTERNATIVE AND ITS THRESHOLD
   □ ★ which store, which version, which managed offering
   □ ★ the number that triggers it (>25M docs, depth ≥5, …)
   □ ★ a prototype with YOUR data and YOUR queries
   ⇒ ★ "NoSQL" is not an answer.

 ★ GATE 5 — PRICE THE FULL COST
   □ the sync pipeline (★ 4–8 weeks + ongoing)
   □ ★ denormalisation required by lost joins
   □ ★ invariants that lose transactional enforcement
   □ ★ backup, restore-testing, monitoring, on-call, upgrades
   □ ★ expertise and the failure modes you don't know yet
   ⇒ ★ multiply the proposal's estimate by 2–4×.

 ★ GATE 6 — WRITE DOWN WHAT YOU GIVE UP
   □ ★ consistency model per operation (Topic 68's table)
   □ ★ which constraints become application code
   □ ★ which queries become impossible
   □ ★ signed off by someone who will own the consequences
   ⇒ ★ "we'll be eventually consistent" is not a design.

 ★ GATE 7 — CAN YOU LEAVE?
   □ ★ is the data model reversible?
   □ ★ is there a documented exit path?
   ⇒ ★ prefer the store that is easier to leave.
```

### The five limits, with concrete thresholds

```
 ★ ① WRITE THROUGHPUT
   TRIGGER: ★ >150k writes/sec sustained, after
     ✓ batching / COPY (66) ✓ index pruning ✓ HOT updates (46)
     ✓ partitioned retention (59) ✓ 192-core hardware
     ✓ functional split
   ⇒ ★ Cassandra/Scylla (72) or application sharding (60)
   ★ GIVE UP: joins · transactions · ad-hoc queries · secondary
     indexes · ★ the ability to change the partition key

 ★ ② GRAPH TRAVERSAL
   TRIGGER: ★ ≥5% of traffic at depth ≥5, or shortest-path /
     centrality between arbitrary nodes
   ⇒ ★ MEASURE THE DEPTH DISTRIBUTION FIRST (74) — ★ 99.3% at
     depth ≤2 in the worked example
   ⇒ ★ AND check for supernodes: ★ the timeout is usually 14 nodes,
     not the depth
   ⇒ Neo4j / a graph store
   ★ GIVE UP: SQL · a second sync pipeline · a different
     operational model

 ★ ③ SEARCH
   TRIGGER: ★ >25M documents, ★ or multi-facet aggregation as a
     product requirement, ★ or a measured relevance gain (A/B on
     top-3 CTR)
   ⇒ ★ FIRST: generated tsvector + setweight + ts_rank_cd +
     websearch_to_tsquery + pg_trgm (74)
   ⇒ ★ AND count how many searches also FILTER — that is what you
     will denormalise
   ⇒ Elasticsearch / OpenSearch
   ★ GIVE UP: transactional freshness · joins · ★ a tested
     reindex procedure is now mandatory

 ★ ④ SUB-MS ORDERED OPERATIONS
   TRIGGER: ★ rank/leaderboard/rate-limit at >50k ops/sec
   ⇒ ★ `ZREVRANK` 0.19 ms vs 8,412 ms (71) — ★ the clearest
     single justification in this phase
   ⇒ Redis
   ★ GIVE UP: durability (★ RPO ~1 s at best) ⇒ ★ Redis is the
     INDEX, PostgreSQL is the LEDGER

 ★ ⑤ ANALYTICAL SCANS
   TRIGGER: ★ ad-hoc aggregation over 10⁹+ rows, sub-second,
     ★ where rollups don't help because the queries are unknown
   ⇒ ★ FIRST: rollup tables (56) and partitioning (59) — ★ they
     cover known queries entirely
   ⇒ ClickHouse / DuckDB / a warehouse
   ★ GIVE UP: point updates · transactional consistency with the
     OLTP store
```

### Writing the decision down

```markdown
## Datastore decision: search   (2026-08-26)

★ PROBLEM
  `GET /api/search` p99 8,412 ms. SLO: 200 ms.
  Corpus: 8.2M listings, 11 GB of text.

★ EXHAUSTION CHECKLIST (Topic 60)
  ✓ indexes: ILIKE cannot be indexed ⇒ replaced with GIN tsvector
  ✓ N+1: none
  ✓ pooling: fine · replicas: read-only path already there

★ RESULT AFTER POSTGRESQL FULL-TEXT
  p99 8,412 ms → ★ 6.2 ms. ★ SLO met with 32× headroom.

★ REMAINING GAP
  multi-facet aggregation: ★ 413 ms for one facet, ~1.6 s for four.
  ⇒ ★ PRODUCT DECISION: one facet, counts capped at 1,000 ("500+")
    ⇒ ★ 68 ms. ★ Gap closed.

★ DECISION: ★ do not adopt Elasticsearch.

★ TRIGGER CONDITIONS TO REVISIT — any one:
  ① corpus > 25M documents
  ② search p99 > 200 ms after tuning
  ③ >2 uncapped facets becomes a product requirement
  ④ A/B shows BM25 beats ts_rank_cd on top-3 CTR by >5%

★ IF ADOPTED, WE ACCEPT:
  • ★ CDC-based sync (never dual-write), a reconciler, a lag alert
  • ★ eventual consistency: "published but not searchable ~2 s"
    becomes documented product behaviour
  • ★ denormalising tenant/category/price/status (88% of searches
    filter on them)
  • ★ a tested full-reindex procedure, timed
  • ★ 4–8 weeks of engineering, plus ongoing maintenance

★ Signed: platform lead, product owner.
```

---

## Concept breakdown

```
★ DISMANTLE THE QUESTION FIRST
   ✗ "NoSQL" is not a category — ★ Redis and Neo4j share nothing
   ✗ "SQL" is not a data model — ★ Postgres is relational,
     document, key-value, search, geospatial, time-series, vector
   ✗ "NoSQL scales" — ★ Postgres: 150k writes/sec, 10–50 TB
   ✗ "schemaless is faster" — ★ the schema moved into every reader

★ THE ASYMMETRY THAT SETS THE DEFAULT
   ★ adopting too early ⇒ a PERMANENT tax, ★ hard to undo
   ★ adopting too late  ⇒ a migration, ★ recoverable
   ⇒ ★ START RELATIONAL. MOVE WHEN MEASURED.

★ WHAT POSTGRESQL ACTUALLY DOES
   ★ 50–150k writes/s · 10–50 TB · billions of rows partitioned ·
   ★ ~10M-doc full-text · tens of millions of jsonb docs
   ⇒ ★ most systems never reach any of these

★ THE FIVE GENUINE LIMITS
   ① ★ >150k writes/s after the checklist  ⇒ Cassandra / sharding
   ② ★ depth ≥5 traversal, ≥5% of traffic  ⇒ a graph store
   ③ ★ >25M docs, facets, BM25 relevance   ⇒ Elasticsearch
   ④ ★ sub-ms ordered ops >50k/s           ⇒ Redis
   ⑤ ★ ad-hoc scans over 10⁹+ rows         ⇒ ClickHouse
   ⇒ ★ NOT on the list: flexible schema · relationships ·
     "lots of data" · HA · developer preference

★ THE COSTS THAT NEVER APPEAR IN THE PROPOSAL
   ★ the sync pipeline (4–8 weeks + ongoing) · lost joins ⇒ more
   denormalisation ⇒ more sync · lost transactions ⇒ sagas ·
   lost constraints ⇒ application code · ★ operational surface ×N
   · ★ N backup timelines with no consistent restore point ·
   ★ expertise in failure modes you haven't met
   ⇒ ★ multiply the estimate by 2–4×

★ SEVEN GATES
   ① measured problem  ② ★ exhaustion checklist  ③ one of the five
   ④ ★ a NAMED alternative + a THRESHOLD + a prototype
   ⑤ ★ the full cost  ⑥ ★ what you give up, in writing, signed
   ⑦ ★ can you leave?

★ AND THE TIE-BREAKER
   ★ prefer the store that is EASIER TO LEAVE.
   ★ Postgres → X is well-trodden. ★ X → Postgres usually is not.
```

---

## Diagrams

**Diagram 1 — big picture: where each store's advantage begins**

```
  ★ THE X-AXIS IS THE MEASUREMENT THAT MATTERS FOR EACH SHAPE.

  WRITE THROUGHPUT
   PostgreSQL  ████████████████████████░░░░  ★ 150k/s
   Cassandra   ████████████████████████████  ★ linear, N nodes
                                        ▲
                            ★ THE CROSSOVER — and almost nobody
                              is to the right of it

  GRAPH TRAVERSAL DEPTH
   PostgreSQL  ████████████░░░░░░░░░░░░░░░░  ★ depth 4 = 41 s
   Neo4j       ████████████████████████████  ★ depth 4 = 12 ms
                         ▲
              ★ THE CROSSOVER IS AT DEPTH 4–5, ★ not depth 2

  SEARCH CORPUS
   PostgreSQL  ██████████████████░░░░░░░░░░  ★ ~10–25M docs
   Elastic     ████████████████████████████  ★ sharded
                              ▲
              ★ AND the crossover for RELEVANCE is earlier than
                for SPEED — ★ Postgres is fast at 8M docs, but
                ts_rank_cd is not BM25

  ORDERED OPS / SECOND
   PostgreSQL  ███░░░░░░░░░░░░░░░░░░░░░░░░░  ★ rank = 8,412 ms
   Redis       ████████████████████████████  ★ rank = 0.19 ms
                ▲
   ★ THE CROSSOVER IS ALMOST IMMEDIATE — ★ this is the clearest
     specialised-store justification in the whole phase

  ANALYTICAL SCAN ROWS
   PostgreSQL  ██████████████░░░░░░░░░░░░░░  ★ 10⁸ with rollups
   ClickHouse  ████████████████████████████  ★ 10¹⁰+
                            ▲
              ★ but rollups (56) move the crossover a long way right

 ⇒ ★ FOUR OF THE FIVE CROSSOVERS ARE FURTHER RIGHT THAN PEOPLE
   ASSUME. ★ The exception is Redis's ordered operations, which
   is genuinely immediate.
```

**Diagram 2 — data flow: what the proposal says vs what it costs**

```
 ★ WHAT THE PROPOSAL CONTAINS
 ┌───────────────────────────────────────────────────────────────┐
 │  "Add Elasticsearch for search."                               │
 │   • ★ better relevance                                         │
 │   • ★ faster queries                                           │
 │   • ★ 2 weeks                                                  │
 └───────────────────────────────────────────────────────────────┘

 ★ WHAT IT ACTUALLY REQUIRES
 ┌───────────────────────────────────────────────────────────────┐
 │  ① ★ THE SYNC PIPELINE                          ★ 4–8 weeks   │
 │     CDC from the WAL (76) · an idempotent indexer ·            │
 │     a reconciler · a lag alert · ★ a TESTED reindex            │
 │  ② ★ DENORMALISATION                            ★ 1–2 weeks   │
 │     88% of searches filter ⇒ tenant, category, price, status   │
 │     into every document ⇒ ★ 4 more fields to keep in sync      │
 │  ③ ★ CONSISTENCY DECISIONS                      ★ ongoing      │
 │     "published but not searchable for 2 s" ⇒ ★ a documented    │
 │     product behaviour, ★ signed off                            │
 │  ④ ★ OPERATIONS                                 ★ ongoing      │
 │     backups + ★ a TESTED restore · monitoring · alerting ·     │
 │     upgrades · ★ on-call · ★ runbooks · capacity planning      │
 │  ⑤ ★ EXPERTISE                                  ★ ongoing      │
 │     shard sizing · mapping explosions · heap tuning ·          │
 │     ★ the failure modes you meet at 3 a.m.                     │
 │  ⑥ ★ NO CONSISTENT CROSS-STORE RESTORE POINT    ★ permanent    │
 │     two independent backup timelines (60, 64)                  │
 │                                                                │
 │  ★ TOTAL: ★ 8–14 weeks + permanent operational load           │
 │  ★ vs THE ALTERNATIVE MEASURED IN TOPIC 74: ★ 2 weeks,        │
 │    ★ 6.2 ms p99, ★ zero new systems.                          │
 └───────────────────────────────────────────────────────────────┘
```

**Diagram 3 — before/after: the same system, two architectures**

```
 ✗ "MODERN" POLYGLOT — adopted by preference, not measurement
 ┌───────────────────────────────────────────────────────────────┐
 │  ★ MongoDB       — the primary store ("flexible schema")       │
 │  ★ Elasticsearch — search                                      │
 │  ★ Redis         — caching                                     │
 │  ★ Neo4j         — "recommendations"                           │
 │  ★ PostgreSQL    — billing (★ "because it needs transactions") │
 │                                                                │
 │  ★ 5 stores · ★ 4 sync pipelines · ★ 5 backup strategies       │
 │  ★ 5 monitoring setups · ★ 5 upgrade paths · ★ 5 on-call docs  │
 │  ★ NO cross-store consistent restore point                     │
 │  ★ NO transaction spans more than one of them                  │
 │  ★ every invariant crossing two stores is a saga (52)          │
 │  ★ 3 of 5 chosen without a measurement                         │
 │                                                                │
 │  ★ TEAM SIZE: 6.                                               │
 └───────────────────────────────────────────────────────────────┘

 ✓ MEASURED — the same product
 ┌───────────────────────────────────────────────────────────────┐
 │  ★ PostgreSQL    — ★ the system of record                      │
 │     • ★ jsonb for the flexible parts (70)                      │
 │     • ★ tsvector + pg_trgm for search (74)                     │
 │     • ★ recursive CTEs, depth ≤4, for recommendations (74)     │
 │     • ★ partitioning for retention (59)                        │
 │     • ★ transactions and FKs across ALL of it                  │
 │  ★ Redis        — ★ leaderboards and rate limits ONLY,         │
 │                   ★ with PostgreSQL as the ledger (71)         │
 │                                                                │
 │  ★ 2 stores · ★ 1 sync pipeline (★ and it's a cache, so a      │
 │    rebuild is trivial) · ★ 2 backup strategies                 │
 │  ★ every business invariant is a database constraint           │
 │  ★ every adoption has a written trigger condition              │
 │                                                                │
 │  ★ AND THE EXIT PATHS ARE DOCUMENTED: if search outgrows       │
 │    Postgres, CDC → Elasticsearch is a known 6-week project.    │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Establish what PostgreSQL actually does, on your hardware.**
```bash
# ★ write throughput
pgbench -i -s 500 bench
pgbench -c 64 -j 16 -T 60 -M prepared bench | grep tps
```
```
 tps = ★ 88,204.2        (simple update/insert mix, 32 cores)
```
```bash
# ★ bulk ingest
time psql -c "COPY events FROM '/tmp/5m_rows.csv' WITH (FORMAT csv)"
```
```
 ★ real 4.1s        — ★ 1,219,512 rows/sec
```
```bash
# ★ point reads
pgbench -c 64 -j 16 -T 60 -S -M prepared bench | grep tps
```
```
 tps = ★ 412,088.4
```
```
 ★ WRITE THESE DOWN. ★ Every "we've outgrown Postgres"
   conversation should start with your own numbers, not a blog
   post's.
```

**Prove the crossover for each of the five limits.**
```sql
-- ★ ① ORDERED OPERATIONS — the clearest case
SELECT count(*) + 1 AS rank FROM scores WHERE game_id=7 AND score > 88420;
```
```
 Execution Time: ★ 8,412 ms        (12.8M index entries counted)
```
```bash
redis-cli ZREVRANK leaderboard player:4201
```
```
 ★ 0.19 ms        ★ 44,000×
 ⇒ ★ THIS CROSSOVER IS IMMEDIATE. No amount of PostgreSQL tuning
   changes it, because the operation is O(n) in a B-tree and
   O(log n) in a skip list.
```

```sql
-- ★ ② GRAPH DEPTH
-- (the recursive CTE from Topic 74, depths 1–5)
```
```
 depth 1: ★ 0.4 ms · 2: ★ 12 ms · 3: ★ 1,840 ms
 depth 4: ★ 41,204 ms · 5: ★ OOM
 ⇒ ★ THE CROSSOVER IS AT DEPTH 4. ★ Not depth 2.
```

```sql
-- ★ ③ SEARCH — speed vs relevance are different crossovers
EXPLAIN (ANALYZE) SELECT id FROM articles
  WHERE search @@ websearch_to_tsquery('english','refund policy')
  ORDER BY ts_rank_cd(search, websearch_to_tsquery('english','refund policy')) DESC
  LIMIT 20;
```
```
 Execution Time: ★ 2.1 ms        (4M documents)
 ⇒ ★ SPEED IS NOT THE PROBLEM AT THIS SCALE.
```
```sql
-- ★ but faceting is
SELECT category_id, count(*) FROM articles
  WHERE search @@ websearch_to_tsquery('english','refund policy')
  GROUP BY 1 ORDER BY 2 DESC LIMIT 10;
```
```
 Execution Time: ★ 412 ms        ⇒ ★ 4 facets ≈ 1.6 s
 ⇒ ★ THE CROSSOVER FOR FACETS ARRIVES LONG BEFORE THE ONE FOR
   SPEED.
```

```sql
-- ★ ④ ANALYTICAL SCANS — and what rollups do to the crossover
SELECT date_trunc('hour', ts), avg(value) FROM samples
 WHERE ts >= now() - interval '30 days' GROUP BY 1;
```
```
 Execution Time: ★ 41,204 ms       (260M rows)
```
```sql
SELECT bucket, sum(s)/sum(n) FROM samples_1h
 WHERE bucket >= now() - interval '30 days' GROUP BY 1;
```
```
 Execution Time: ★ 88 ms        ★ 468×
 ⇒ ★ ROLLUPS MOVE THE ANALYTICAL CROSSOVER A LONG WAY RIGHT.
   ★ ClickHouse is for the queries you CANNOT precompute.
```

**Prove PostgreSQL covers the "flexible schema" case.**
```sql
CREATE TABLE things (
  id bigserial PRIMARY KEY,
  tenant_id bigint NOT NULL REFERENCES tenants(id),   -- ★ an FK
  status text NOT NULL CHECK (status IN ('a','b')),   -- ★ a CHECK
  attrs jsonb NOT NULL DEFAULT '{}',                  -- ★ flexible
  hot int GENERATED ALWAYS AS ((attrs->>'hot')::int) STORED);
CREATE INDEX ON things USING gin (attrs jsonb_path_ops);
CREATE INDEX ON things (hot) WHERE hot IS NOT NULL;
```
```sql
BEGIN;
  INSERT INTO things (tenant_id, status, attrs) VALUES (1,'a','{"x":1}');
  UPDATE tenants SET thing_count = thing_count + 1 WHERE id = 1;
COMMIT;
-- ★ a flexible document AND a transactional invariant, together.
--   ★ MongoDB gives you the first; ★ this gives you both.
```

**Prove the operational multiplication is real.**
```bash
# ★ count what a single store requires
ls /etc/postgresql/*/main/*.conf | wc -l          # config
ls /etc/prometheus/rules/postgres*.yml | wc -l    # alert rules
ls docs/runbooks/postgres*.md | wc -l             # runbooks
```
```
 ★ 4 config files · ★ 18 alert rules · ★ 11 runbooks
 ⇒ ★ EACH ADDITIONAL STORE NEEDS ITS OWN VERSION OF ALL THREE,
   plus a backup strategy, a tested restore, an upgrade path and
   an on-call rotation.
 ⇒ ★ THIS IS THE COST THAT NEVER APPEARS IN THE PROPOSAL.
```

**Prove there is no cross-store consistent restore point.**
```bash
psql -tAc "SELECT pg_current_wal_lsn()"
redis-cli LASTSAVE
curl -s localhost:9200/_stats | jq '.._all.primaries.docs.count'
```
```
 ★ 4A/8C001220
 ★ 1756201422
 ★ 8842119
 ⇒ ★ THREE INDEPENDENT TIMELINES. ★ There is no single point in
   time all three can be restored to. (Topics 60, 64.)
```

---

## Example 2 — production scenario

**The situation.** A 6-person team, 18 months into building a B2B logistics platform. The architecture, as it grew:

```
 ★ THE CURRENT STACK
   ★ MongoDB       — shipments, the "primary" store
   ★ Elasticsearch — shipment search
   ★ Redis         — caching + some counters
   ★ Neo4j         — "route optimisation"
   ★ PostgreSQL    — billing only ("because money needs
                     transactions")

 ★ THE SYMPTOMS
   ★ p99 on the main dashboard    ★ 4,200 ms
   ★ data inconsistencies         ★ 41 support tickets/month
   ★ "shipment shows delivered in
      search but in-transit in the
      app"                        ★ ~weekly
   ★ on-call load                 ★ 2 pages/night average
   ★ ★ time spent on data plumbing ★ ~40% of engineering
   ★ ★ nobody can restore the
      system to a point in time    ★ 5 independent timelines
```

**Step 1 — audit why each store was adopted.**

```
 ★ THE QUESTION ASKED OF EACH: ★ "what measurement led to this?"

 ★ MongoDB
   ⇒ ★ "shipment attributes vary by carrier"
   ⇒ ★ MEASUREMENT: ★ none. Chosen at the prototype stage.
   ⇒ ★ ACTUAL VARIABILITY: 14 carriers, ★ 9 attributes, ★ a
     stable set for 14 months.
   ⇒ ★ jsonb covers this ENTIRELY (Topic 70).

 ★ ELASTICSEARCH
   ⇒ ★ "users search shipments"
   ⇒ ★ MEASUREMENT: none.
   ⇒ ★ ACTUAL CORPUS: ★ 4.1M shipments.
   ⇒ ★ ACTUAL QUERIES: ★ 96% are `tracking_number = ?` —
     ★ AN EXACT LOOKUP, NOT A SEARCH.
   ⇒ ★ the remaining 4% are prefix matches on address.

 ★ NEO4J
   ⇒ ★ "route optimisation"
   ⇒ ★ MEASUREMENT: none.
   ⇒ ★ ACTUAL USE: ★ "which hubs connect A to B" — ★ depth ≤3,
     over ★ 412 hubs and 8,842 routes.
   ⇒ ★ 412 NODES. ★ The entire graph fits in a CTE in memory.

 ★ REDIS
   ⇒ ★ caching + counters
   ⇒ ★ MEASUREMENT: ★ some. The cached queries were genuinely
     slow — ★ because of a missing index (Topic 57's finding).

 ★ POSTGRESQL
   ⇒ ★ "money needs transactions"
   ⇒ ★ CORRECT — ★ and the implicit admission that ★ the other
     four stores cannot provide them.

 ⇒ ★ FOUR OF FIVE STORES WERE ADOPTED WITHOUT A MEASUREMENT.
```

**Step 2 — measure what each is actually being asked to do.**

```sql
-- ★ Elasticsearch query shapes, from its slow log
```
```
 query_type              | count/day |  pct
------------------------+-----------+-------
 ★ term (tracking_number)| ★ 8,842,119| ★ 96.1
 prefix (address)        |   ★ 288,402|   3.1
 match (free text)       |    ★ 71,204|   0.8
 ⇒ ★ 96% IS AN EXACT LOOKUP. ★ That is a B-tree index.
```
```
 ★ Neo4j: 412 nodes, 8,842 edges, max depth used ★ 3
 ★ MongoDB: 4.1M documents, ★ 9 distinct attribute keys,
   ★ zero schema changes in 14 months
 ★ Redis: hit rate 94%, ★ and 71% of cached queries became
   sub-millisecond after one index was added
```

**Step 3 — the consolidation, store by store.**

```sql
-- ★ ① MongoDB → PostgreSQL jsonb (Topic 70)
CREATE TABLE shipments (
  id              bigserial   PRIMARY KEY,
  tracking_number text        NOT NULL UNIQUE,        -- ★ 96% of queries
  tenant_id       bigint      NOT NULL REFERENCES tenants(id),
  carrier_id      bigint      NOT NULL REFERENCES carriers(id),
  origin_hub_id   bigint      NOT NULL REFERENCES hubs(id),
  dest_hub_id     bigint      NOT NULL REFERENCES hubs(id),
  status          text        NOT NULL CHECK (status IN
                    ('created','in_transit','out_for_delivery',
                     'delivered','exception')),
  created_at      timestamptz NOT NULL DEFAULT now(),
  delivered_at    timestamptz,
  -- ★ the genuinely variable part
  carrier_attrs   jsonb       NOT NULL DEFAULT '{}',
  ★ CHECK (delivered_at IS NULL OR status = 'delivered')  -- ★ an invariant
) PARTITION BY RANGE (created_at);

CREATE UNIQUE INDEX ON shipments (tracking_number);      -- ★ the 96%
CREATE INDEX ON shipments (tenant_id, created_at DESC);
CREATE INDEX ON shipments USING gin (carrier_attrs jsonb_path_ops);
```
```
 ★ NOTE WHAT BECAME POSSIBLE:
   ★ FKs to tenants, carriers and hubs — ★ previously application
     code, ★ and the source of the "shipment references a deleted
     hub" tickets
   ★ a CHECK enforcing delivered_at ⇔ status='delivered' —
     ★ previously the source of the "delivered in search,
     in-transit in the app" bug
   ★ partitioning for retention (59)
```

```sql
-- ★ ② Elasticsearch → a B-tree + tsvector + pg_trgm (Topic 74)
-- 96%: an exact lookup
SELECT * FROM shipments WHERE tracking_number = $1;      -- ★ 0.08 ms

-- 3.1%: an address prefix
CREATE INDEX ON shipments USING gin (
  (carrier_attrs->>'delivery_address') gin_trgm_ops);
SELECT * FROM shipments
 WHERE carrier_attrs->>'delivery_address' ILIKE $1 || '%'
   AND tenant_id = $2 LIMIT 20;                           -- ★ 4.2 ms

-- 0.8%: free text
ALTER TABLE shipments ADD COLUMN search tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('simple', tracking_number), 'A') ||
    setweight(to_tsvector('english',
      coalesce(carrier_attrs->>'notes','')), 'C')) STORED;
CREATE INDEX ON shipments USING gin (search);             -- ★ 6.1 ms
```

```sql
-- ★ ③ Neo4j → a recursive CTE. ★ 412 nodes.
WITH RECURSIVE route AS (
  SELECT dest_hub_id AS hub, 1 AS hops, ARRAY[$1, dest_hub_id] AS path
    FROM hub_routes WHERE origin_hub_id = $1
  UNION ALL
  SELECT r.dest_hub_id, route.hops + 1, route.path || r.dest_hub_id
    FROM route JOIN hub_routes r ON r.origin_hub_id = route.hub
   WHERE route.hops < 4
     AND NOT (r.dest_hub_id = ANY(route.path)))            -- ★ cycle guard
SELECT hub, min(hops) FROM route WHERE hub = $2 GROUP BY 1;
```
```
 ★ Execution Time: ★ 1.8 ms        (Neo4j: 4 ms + a network hop)
 ⇒ ★ FASTER, because there is no second system to call.
```

```sql
-- ★ ④ Redis: keep it — but for what it is actually good at
--    ✓ rate limiting (sliding window ZSET)      — ★ genuinely Redis
--    ✓ session storage with a TTL                — ★ genuinely Redis
--    ✗ caching slow queries ⇒ ★ FIX THE QUERIES (Topic 57)
CREATE INDEX CONCURRENTLY ON shipments (tenant_id, status, created_at DESC)
  WHERE status IN ('in_transit','out_for_delivery');
-- ⇒ ★ 71% of cached queries became sub-millisecond ⇒ ★ cache removed
```

**Step 4 — the migration, over 14 weeks.**

```
 ★ THE ORDER MATTERED, AND WAS CHOSEN TO REDUCE RISK FIRST:

 ★ WEEK 1–2   ★ Neo4j → CTE.  ★ Smallest blast radius, 412 nodes,
              ★ proves the pattern to a sceptical team.
 ★ WEEK 3–5   ★ Elasticsearch → Postgres. ★ 96% was already an
              exact lookup; ★ the risk was the 4%.
 ★ WEEK 6–11  ★ MongoDB → Postgres. ★ The big one: dual-write,
              backfill, ★ shadow reads for a week (Topic 62's
              cutover pattern).
 ★ WEEK 12    ★ Redis caching removed after the index landed.
 ★ WEEK 13–14 ★ decommission, and ★ delete the sync pipelines.

 ★ AND THE DECISION THAT MADE IT SAFE:
   ★ EACH STEP WAS INDEPENDENTLY REVERSIBLE, and ★ shadow reads
   compared old and new for a week before each cutover.
```

```js
// ★ the shadow-read harness used at every step
async function shadowRead(key, oldFn, newFn) {
  const [oldR, newR] = await Promise.allSettled([oldFn(), newFn()]);
  if (oldR.status === 'fulfilled' && newR.status === 'fulfilled') {
    if (!deepEqual(normalise(oldR.value), normalise(newR.value))) {
      metrics.increment('migration.mismatch', { key });
      log.warn({ key, old: oldR.value, new: newR.value }, 'shadow mismatch');
    }
  }
  return oldR.status === 'fulfilled' ? oldR.value : Promise.reject(oldR.reason);
}
// ★ MEASURED: 412 mismatches in week 1 of the MongoDB migration,
//   ★ all traced to 3 documents with a legacy attribute shape.
//   ⇒ ★ found BEFORE the cutover, not after.
```

**Step 5 — what was kept, and why.**

```markdown
## Datastore inventory   (post-consolidation, 2026-08-26)

★ PostgreSQL 17 — ★ the system of record
  • relational core with FKs and CHECKs
  • ★ jsonb for carrier-specific attributes (9 keys, stable)
  • ★ tsvector + pg_trgm for the 4% of searches that need it
  • ★ recursive CTEs for hub routing (412 nodes, depth ≤4)
  • ★ partitioned by month, retention by DROP
  • ★ Patroni for HA, ★ wal-g for PITR, ★ a nightly restore test

★ Redis 7 — ★ rate limiting and sessions ONLY
  • ★ sliding-window ZSET rate limiter
  • ★ session storage with a TTL
  • ★ NOT the system of record for anything
  • ★ appendonly yes, ★ maxmemory-policy volatile-lru
  • ★ a flush is survivable: sessions re-authenticate,
    rate limits reset. ★ Tested quarterly.

★ TRIGGER CONDITIONS TO REVISIT — ★ written, with numbers:
  ① search corpus > 25M shipments, or search p99 > 200 ms
  ② hub graph > 50,000 nodes, or traversal depth ≥5 needed
  ③ write throughput > 100k/sec sustained after the checklist
  ④ multi-facet search becomes a product requirement

★ Signed: engineering lead, CTO.
```

**Step 6 — results.**

| | Before (5 stores) | After (2 stores) |
|---|---|---|
| Datastores | ★ **5** | **2** |
| Sync pipelines | ★ **4** | **0** |
| Dashboard p99 | ★ 4,200 ms | **41 ms** (**102×**) |
| Tracking lookup | 12 ms (ES) | **0.08 ms** |
| Route query | 4 ms + hop | **1.8 ms** |
| Data-inconsistency tickets | ★ **41/month** | **0** |
| Pages per night | ★ 2 | **0.2** |
| Engineering time on plumbing | ★ **~40%** | **~5%** |
| Backup timelines | ★ 5, no consistent point | ★ **1 + a survivable cache** |
| Invariants enforced by the DB | ★ 0 cross-store | ★ **all of them** |
| Infrastructure cost | ★ ₹4.1L/month | **₹1.2L/month** |

```
 ★ SEVEN LESSONS:
 ① ★ FOUR OF FIVE STORES WERE ADOPTED WITHOUT A MEASUREMENT.
   That is the finding, and it is the usual one.
 ② ★ 96% OF "SEARCH" WAS AN EXACT LOOKUP. ★ Elasticsearch was
   serving a B-tree query, over a network, with a sync pipeline.
 ③ ★ THE "GRAPH" HAD 412 NODES. It fits in memory in a CTE, and
   the CTE is faster because there is no network hop.
 ④ ★ THE CACHE EXISTED BECAUSE OF A MISSING INDEX (Topic 57's
   exact finding, again).
 ⑤ ★ THE INCONSISTENCY TICKETS VANISHED BECAUSE THE INVARIANTS
   BECAME CONSTRAINTS. "Delivered in search but in-transit in the
   app" is impossible when one CHECK constraint enforces it.
 ⑥ ★ 40% OF ENGINEERING TIME WAS DATA PLUMBING. ★ That is the
   real cost of polyglot persistence, and it never appears in the
   proposal.
 ⑦ ★ THE TRIGGER CONDITIONS ARE WRITTEN DOWN. ★ "Not yet" is a
   defensible engineering position only when you state, in
   numbers, what would change it.
```

---

## Common mistakes

**1. Treating "NoSQL" as a category.**
- *Symptom:* a discussion comparing "SQL vs NoSQL" as if Redis and Neo4j were interchangeable.
- *Fix:* name the specific store and the specific access pattern it serves.

**2. Confusing the query language with the data model.**
- *Symptom:* "we need documents, so not PostgreSQL."
- *Fix:* PostgreSQL is a document store, a key-value store, a search engine and a time-series store. The model is not the language.

**3. Adopting before running the exhaustion checklist.**
- *Symptom:* a distributed system built to avoid writing an index.
- *Fix:* Topic 60's checklist. In its worked example, three weeks of fixes bought eighteen months.

**4. Pricing only the query, not the operations.**
- *Symptom:* a two-week estimate for an eight-to-fourteen-week project with permanent recurring load.
- *Fix:* price the sync pipeline, the denormalisation, the lost constraints, the backups, the on-call and the expertise. Multiply by 2–4×.

**5. Ignoring that lost joins mean more sync.**
- *Symptom:* four more fields denormalised into every document because most queries filter on them.
- *Fix:* count how many queries filter before adopting. It is usually most of them.

**6. Not writing down what you give up.**
- *Symptom:* "we're eventually consistent" as the entire consistency design.
- *Fix:* Topic 68's table — per operation, model, bound, partition behaviour — signed off.

**7. Choosing the store that is harder to leave.**
- *Symptom:* a partition key or embedded document shape that cannot be changed without rewriting every read path.
- *Fix:* when uncertain, choose the reversible option.

**8. Adopting for developer preference.**
- *Symptom:* a datastore chosen at prototype stage and never revisited.
- *Fix:* preference is a real input to hiring and velocity, but it is not an architecture measurement.

**9. Underestimating the operational multiplier.**
- *Symptom:* a six-person team operating five datastores, spending 40% of its time on plumbing.
- *Fix:* each store needs config, alerts, runbooks, backups, a tested restore, an upgrade path and on-call. That is roughly linear, not marginal.

**10. Forgetting there is no cross-store restore point.**
- *Symptom:* discovering during an incident that five independent timelines cannot be restored to a consistent moment.
- *Fix:* know this before adopting the second store, and decide whether it matters.

**11. Keeping a cache that exists because of a missing index.**
- *Symptom:* a caching layer whose hit rate is high and whose underlying queries are 0.08 ms once indexed.
- *Fix:* Topic 57 — measure the underlying query before caching it.

**12. Never revisiting the decision.**
- *Symptom:* a store adopted at prototype scale still in place at 100× the scale, for reasons nobody remembers.
- *Fix:* written trigger conditions, reviewed annually.

---

## Hands-on proof

**PROVE IT #1–#6 — Example 1** (measuring your own write/read/ingest throughput, the immediate Redis rank crossover at 44,000×, the graph cliff at depth 4, search speed being fine at 4M documents while faceting is not, rollups moving the analytical crossover 468×, and jsonb giving flexibility *plus* transactions).

**PROVE IT #7 — the operational surface, counted.**
```bash
for store in postgres redis elasticsearch mongodb neo4j; do
  echo -n "$store: "
  echo -n "$(ls /etc/prometheus/rules/${store}*.yml 2>/dev/null | wc -l) alerts, "
  echo -n "$(ls docs/runbooks/${store}*.md 2>/dev/null | wc -l) runbooks, "
  echo "$(ls ops/backup/${store}* 2>/dev/null | wc -l) backup scripts"
done
```
```
 ★ postgres: 18 alerts, 11 runbooks, 3 backup scripts
 ★ redis: 9 alerts, 4 runbooks, 1 backup script
 ★ elasticsearch: 14 alerts, 7 runbooks, 2 backup scripts
 ★ mongodb: 12 alerts, 6 runbooks, 2 backup scripts
 ★ neo4j: 6 alerts, 2 runbooks, 1 backup script
 ⇒ ★ 59 alerts, 30 runbooks, 9 backup procedures — ★ for a
   six-person team.
```

**PROVE IT #8 — measure what your "search" actually is.**
```sql
SELECT query_type, count(*),
       round(100.0*count(*)/sum(count(*)) OVER (), 1) AS pct
  FROM search_log WHERE at > now() - interval '30 days'
 GROUP BY 1 ORDER BY 2 DESC;
```
```
 ★ if `term`/exact-match dominates, ★ you have an index problem,
   not a search problem.
```

**PROVE IT #9 — measure your actual graph size and depth.**
```sql
SELECT (SELECT count(*) FROM hubs) AS nodes,
       (SELECT count(*) FROM hub_routes) AS edges,
       (SELECT max(depth) FROM traversal_log
         WHERE at > now() - interval '30 days') AS max_depth_used;
```
```
 nodes | edges | max_depth_used
-------+-------+----------------
 ★ 412 | 8,842 |            ★ 3
 ⇒ ★ 412 nodes is not a graph database problem.
```

**PROVE IT #10 — the cost of a mismatch, before cutover.**
```js
// ★ run shadow reads for a week before every migration cutover
const mismatches = await metrics.query('sum(migration_mismatch_total)');
console.log({ mismatches });
```
```
 ★ { mismatches: 412 }        — ★ traced to 3 legacy document
   shapes. ★ Found before the cutover, at zero user impact.
```

---

## The design decision framework

```
★★★ START RELATIONAL. MOVE WHEN MEASURED.
    ★ THE COST OF BEING LATE IS A MIGRATION;
    ★ THE COST OF BEING EARLY IS A PERMANENT TAX. ★★★

 ★ GATE 1 — A MEASURED PROBLEM
   □ a named operation with p50/p99/p999 and a missed SLO
   □ ★ Topic 67 levels 1–2: ★ is it even the database?
   ⇒ ★ "we might need to scale" fails here.

 ★ GATE 2 — THE EXHAUSTION CHECKLIST (Topic 60)
   □ indexes □ N+1 □ ★ partitioned retention □ caching/CDN
   □ replicas □ ★ hot rows □ ★ pooling □ ★ vertical scaling
   □ ★ a functional split
   ⇒ ★ measured: 3 weeks of this bought 18 months.

 ★ GATE 3 — IS IT ONE OF THE FIVE GENUINE LIMITS?
   ① >150k writes/s   ② depth ≥5 traversal   ③ >25M docs /
   facets / BM25   ④ sub-ms ordered ops   ⑤ ad-hoc 10⁹-row scans
   ⇒ ★ NOT: flexible schema · relationships · "lots of data" ·
     HA · developer preference

 ★ GATE 4 — NAME IT, THRESHOLD IT, PROTOTYPE IT
   □ ★ the specific store and version
   □ ★ the number that triggers adoption
   □ ★ a prototype with YOUR data and YOUR queries

 ★ GATE 5 — PRICE THE FULL COST (★ multiply by 2–4×)
   □ ★ the sync pipeline: 4–8 weeks + ongoing
   □ ★ denormalisation forced by lost joins
   □ ★ invariants losing transactional enforcement
   □ ★ backups, tested restores, monitoring, on-call, upgrades
   □ ★ expertise in failure modes you haven't met
   □ ★ N backup timelines, no consistent restore point

 ★ GATE 6 — WRITE DOWN WHAT YOU GIVE UP, AND SIGN IT
   □ ★ Topic 68's table: operation | model | bound | partition
     behaviour
   □ ★ which constraints become application code
   □ ★ which queries become impossible

 ★ GATE 7 — CAN YOU LEAVE?
   □ ★ is the data model reversible?
   □ ★ Postgres → X is well-trodden; ★ X → Postgres often is not
   ⇒ ★ when uncertain, choose the one that is easier to leave.

 ★ AND AFTERWARDS
   ✓ ★ record the trigger conditions IN WRITING, with numbers
   ✓ ★ review annually — ★ a decision made at prototype scale is
     not a decision made at 100× scale
   ✓ ★ shadow-read before every cutover, in both directions
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Measure your own PostgreSQL: write throughput (`pgbench`), bulk ingest (`COPY`), point reads, and the largest table you have. Write the four numbers down. Then find one blog post claiming "PostgreSQL doesn't scale" and identify which of your four numbers it contradicts.

### Exercise 2 — medium (apply it)
For each of the five genuine limits, construct the measurement that would tell you whether you have crossed it: the query, the metric, and the threshold. Then run all five against your own system and record the results.

### Exercise 3 — hard (production simulation)
A six-person team runs five datastores. Dashboard p99 is 4,200 ms, there are 41 data-inconsistency tickets a month, two pages a night, and roughly 40% of engineering time goes on data plumbing.

(a) For each store, write the audit question and predict the answer.
(b) 96% of Elasticsearch queries are `tracking_number = ?`. What does that tell you, and what replaces it?
(c) The Neo4j graph has 412 nodes and a maximum used depth of 3. Write the replacement and explain why it is *faster*.
(d) The Redis cache has a 94% hit rate. Explain why that is not evidence it is needed, and what you would measure instead.
(e) Design the consolidated PostgreSQL schema. Identify three invariants that become database constraints and name the bug each one eliminates.
(f) Order the migration to reduce risk first. Justify the ordering.
(g) Design the shadow-read harness. What did it find, and why does finding it before cutover matter so much?
(h) Write the post-consolidation inventory with trigger conditions, in numbers.
(i) Infrastructure cost fell 3.4×, but that was not the main benefit. Argue what was.

---

## Mental model checkpoint

1. Why is "SQL vs NoSQL" a false dichotomy? Give two reasons.
2. State the asymmetry between adopting too early and too late, and what default it implies.
3. Give PostgreSQL's approximate limits on writes/sec, table size, full-text corpus and active connections.
4. Name the five genuine limits, with a threshold for each.
5. Name five reasons that are *not* on that list, and what handles each instead.
6. Name six costs that never appear in an adoption proposal.
7. Why is the migration cost asymmetric, and what does that imply when uncertain?
8. Why does "88% of searches also filter" matter so much?
9. What are the seven gates?
10. Why must trigger conditions be written down in numbers?
11. Why is a 94% cache hit rate not evidence that the cache is needed?
12. What does "no cross-store consistent restore point" mean in practice?

---

## Quick reference card

**★ The five genuine limits**

| Limit | Threshold | Store | Give up |
|---|---|---|---|
| write throughput | ★ >150k/s after the checklist | Cassandra / sharding | joins, transactions, ★ the key schema |
| graph depth | ★ ≥5% at depth ≥5 | Neo4j | SQL, ★ a sync pipeline |
| search | ★ >25M docs / facets / BM25 | Elasticsearch | ★ freshness, joins |
| ordered ops | ★ >50k/s rank ops | Redis | ★ durability (RPO ~1 s) |
| ad-hoc scans | ★ >10⁹ rows, sub-second | ClickHouse | point updates |

**★ Not on the list:** flexible schema (→ `jsonb`) · relationships (→ joins) · lots of data (→ partitioning) · HA (→ Patroni) · developer preference.

**★ Seven gates:** measured problem → ★ exhaustion checklist → one of the five → ★ named alternative + threshold + prototype → ★ full cost (**×2–4**) → ★ what you give up, signed → ★ can you leave?

**★ Costs never in the proposal:** the sync pipeline (4–8 wk + ongoing) · denormalisation from lost joins · lost transactions → sagas · lost constraints → app code · ★ operational surface ×N · ★ N backup timelines, no consistent restore point · expertise.

**★ PostgreSQL, measured:** 50–150k writes/s · 500k–2M rows/s via `COPY` · 200k+ point reads/s · 10–50 TB · ~10M-doc full-text · ★ 50–200 *active* connections.

**★ The default:** start relational, move when measured, prefer the store that is easier to leave, and **write the trigger conditions down in numbers.**

---

## When would I use this at work?

1. **Every "should we adopt X?" conversation.** The seven gates turn a preference debate into a measurement exercise. In the worked example, four of five stores had been adopted without a single measurement — which is the usual finding, not an unusual one.

2. **When inheriting a polyglot architecture.** Ask of each store: *what measurement led to this?* The answers are often "none" or "a prototype decision from three years ago", and consolidation is frequently the highest-leverage work available.

3. **When someone says "PostgreSQL doesn't scale".** Have your own four numbers — writes/sec, ingest rate, point reads, largest table. Most such claims contradict at least one of them, and the conversation improves immediately.

4. **Writing an architecture decision record.** The valuable part is not the decision; it is the **trigger conditions**. "Not yet, and here are the four numbers that would change it" is a defensible engineering position that ends the quarterly debate — and it makes the future migration a planned project rather than a crisis.

---

## Connected topics

**Understand before this:** 70–74 (each specialised model on its own terms), 60 (the exhaustion checklist — gate 2), 54 (the denormalisation gates — the same discipline), 68 (the consistency vocabulary — gate 6), 67 (is it even the database? — gate 1).

**This unlocks:**
- **76** — polyglot persistence and CDC: how to run more than one store honestly
- **77–79** — the capstones, where this decision is part of the deliverable
