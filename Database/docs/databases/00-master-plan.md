# 00 — Master Plan
## Databases — Deep Mastery Curriculum
### Design · Internals · Transactions · Normalisation · Denormalisation
### Zero → Principal Engineer Level

---

## THE CONTRACT

This file is the contract between us. It defines:

1. **The complete curriculum** — every topic, numbered, with a one-line description of what it teaches
2. **The complete file list** — the exact filename for every doc
3. **The progress tracker** — a checklist to tick off as we go
4. **The phase overview** — where you are in the journey at any moment

Nothing gets taught that is not on this plan. Nothing on this plan gets skipped.

**Target reader:** a Node.js backend developer using PostgreSQL and MongoDB who wants to
understand what actually happens inside the engine — not which command to type.

**Explicitly out of scope:** SQL syntax tutorials, ORM API docs, product comparisons,
vendor marketing. This is concepts, internals, design, and theory.

**The bar:** after each topic you should be able to explain it from memory, draw it on a
whiteboard, debug it in production, and defend the design decision under pressure.

---

## PHASE OVERVIEW

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  THE JOURNEY — 9 phases, 79 topics                                             │
└────────────────────────────────────────────────────────────────────────────────┘

  PHASE 1 ── FOUNDATIONS & PHYSICAL STORAGE ─────────────────── 9 topics (01–09)
  │   "A database is a set of files. Here is what is in them."
  │   Pages, tuples, heap files, buffer pool, engine architecture, life of a query
  ↓
  PHASE 2 ── INDEXES: HOW DATABASES FIND DATA FAST ──────────  10 topics (10–19)
  │   "Why 40ms instead of 4 seconds — and why sometimes it's still 4 seconds."
  │   B-trees, index scans, selectivity, planner, EXPLAIN, join algorithms
  ↓
  PHASE 3 ── DATABASE DESIGN ────────────────────────────────── 9 topics (20–28)
  │   "Turning a problem description into a schema that survives 3 years."
  │   ER modelling, keys, foreign keys, constraints, types, patterns, migrations
  ↓
  PHASE 4 ── NORMALISATION ────────────────────────────────── 10 topics (29–38)
  │   "One fact, one place. The maths behind why your data goes wrong."
  │   Anomalies, functional dependencies, 1NF→5NF, worked example, when to stop
  ↓
  PHASE 5 ── TRANSACTIONS & CONCURRENCY ───────────────────── 14 topics (39–52)
  │   "The hardest part of databases. Where most production bugs actually live."
  │   ACID, WAL, recovery, isolation, locks, MVCC, VACUUM, deadlocks, 2PC, outbox
  ↓
  PHASE 6 ── DENORMALISATION & PERFORMANCE ─────────────────── 9 topics (53–61)
  │   "Deliberately breaking the rules from Phase 4 — and paying for it correctly."
  │   Denorm patterns, materialised views, caching, replicas, partitioning, sharding
  ↓
  PHASE 7 ── RELIABILITY & SCALE ───────────────────────────── 8 topics (62–69)
  │   "Keeping it alive at 3am when the primary dies."
  │   Replication, failover, backups/PITR, pooling, N+1, perf method, CAP, security
  ↓
  PHASE 8 ── NoSQL DATA MODELLING ──────────────────────────── 7 topics (70–76)
  │   "Different engines, different rules. Model for your queries, not your entities."
  │   Document, key-value, wide-column, time-series, graph/search, SQL-vs-NoSQL, CDC
  ↓
  PHASE 9 ── CAPSTONE & SYNTHESIS ──────────────────────────── 3 topics (77–79)
      "Put all of it together on one real system, twice, then review it like a principal."
```

| Phase | Name | Topics | Files | Why it comes here |
|-------|------|--------|-------|-------------------|
| 1 | Foundations & Physical Storage | 9 | 01–09 | You cannot reason about anything above without knowing data is bytes in 8KB pages |
| 2 | Indexes | 10 | 10–19 | Indexes are just another file layout — Phase 1 makes them obvious instead of magic |
| 3 | Database Design | 9 | 20–28 | Now that you know the cost of a row and an index, you can design tables honestly |
| 4 | Normalisation | 10 | 29–38 | The theory that says which of your Phase 3 designs is correct |
| 5 | Transactions & Concurrency | 14 | 39–52 | The engine machinery that makes concurrent access on those tables safe |
| 6 | Denormalisation & Performance | 9 | 53–61 | Breaking Phase 4 on purpose, now that you can price the risk from Phase 5 |
| 7 | Reliability & Scale | 8 | 62–69 | Operating everything above under load, failure, and attack |
| 8 | NoSQL Data Modelling | 7 | 70–76 | The same physics with different trade-offs — meaningful only after Phases 1–7 |
| 9 | Capstone | 3 | 77–79 | Proof you can actually do it end to end |

**Reordering note (deliberate changes from the original outline):**
- `VACUUM & bloat` moved from Phase 7 → Phase 5 (topic 47). It is a *direct consequence* of
  MVCC and makes no sense taught 20 topics away from it.
- `Join algorithms` added to Phase 2 (topic 19). EXPLAIN output is unreadable without it.
- `Row-store vs column-store` and `B-tree vs LSM storage engines` added to Phase 1 (06, 08).
  These are the two storage decisions that explain almost every engine difference later.
- `Crash recovery / checkpoints` split out of WAL (topic 42) — WAL is the *format*, recovery
  is the *algorithm*; conflating them is why most people can only half-explain durability.
- `Transactional messaging / outbox` added (topic 52) — the #1 real transaction bug in
  Node.js backends that call a queue and a database in the same handler.

---

## THE COMPLETE CURRICULUM

---

### PHASE 1 — FOUNDATIONS & PHYSICAL STORAGE (9 topics)

| # | File | What this file teaches |
|---|------|------------------------|
| 01 | `01-what-is-a-database.md` | Why a folder of JSON files fails at concurrency, crash recovery, querying and integrity — and what a DBMS adds to make those four problems disappear. |
| 02 | `02-database-engine-architecture.md` | The component map of an engine — parser, planner, executor, buffer pool, storage manager, transaction manager, lock manager, WAL writer, background workers — and which component owns which guarantee. |
| 03 | `03-types-of-databases-and-data-models.md` | The six data models (relational, document, key-value, wide-column, graph, time-series/search), the access pattern each is physically optimised for, and how to match a problem to a model without naming a product. |
| 04 | `04-how-data-is-stored-on-disk.md` | The data directory, heap files, 8KB pages, extents and segments, the page header / item pointer / free space / tuple layout, and why 8KB is the number. |
| 05 | `05-row-storage-format-and-tuple-layout.md` | How one row becomes bytes: tuple header, NULL bitmap, alignment padding, variable-length fields, and TOAST for oversized values — plus why column order changes your table size. |
| 06 | `06-row-store-vs-column-store.md` | Why OLTP engines store rows together and analytics engines store columns together, what compression does to each, and how to recognise which workload you actually have. |
| 07 | `07-the-buffer-pool.md` | Why pages live in RAM, clock-sweep/LRU eviction, dirty pages, pinning, the OS page cache underneath, and cache hit ratio translated into real millisecond numbers. |
| 08 | `08-storage-engines-btree-vs-lsm.md` | Update-in-place B-tree engines vs log-structured merge-tree engines — memtables, SSTables, compaction, write amplification vs read amplification, and which workload each wins. |
| 09 | `09-the-life-of-a-query.md` | One SELECT and one INSERT traced end to end through every component from Phase 1 — the trace you will reuse in every later topic. |

---

### PHASE 2 — INDEXES: HOW DATABASES FIND DATA FAST (10 topics)

| # | File | What this file teaches |
|---|------|------------------------|
| 10 | `10-what-an-index-is.md` | The full-table-scan problem in page counts, what an index physically is (a separate file with its own pages), and the core trade: read speed bought with write cost and disk. |
| 11 | `11-btree-indexes-in-depth.md` | The B+tree on disk — root/internal/leaf pages, high keys, sibling pointers, fanout maths, node splits and balance on insert — and why not a binary tree, not a hash table. |
| 12 | `12-index-lookup-end-to-end.md` | The complete lookup trace: descend the tree, read the leaf, fetch the heap tuple, check visibility — plus index-only scans, the visibility map, covering indexes, and bitmap heap scans. |
| 13 | `13-types-of-indexes.md` | Single-column, unique, partial, expression/functional, and multi-column indexes — what each one physically stores and the exact query shape each one serves. |
| 14 | `14-composite-index-column-order.md` | The leftmost-prefix rule, why `(a,b)` and `(b,a)` are different data structures, equality-then-range ordering, and how to design one index that serves five queries. |
| 15 | `15-selectivity-cardinality-and-statistics.md` | Selectivity vs cardinality, the histogram and most-common-values statistics the planner keeps, why an index on a boolean is usually dead weight, and how stale stats produce catastrophic plans. |
| 16 | `16-specialised-index-types.md` | Hash, GIN (inverted, for arrays/JSONB/full-text), GiST (geometric/range), BRIN (block-range for huge append-only tables) — the structure of each and the exact query it exists for. |
| 17 | `17-when-indexes-hurt.md` | Write amplification per index, index bloat, HOT updates, the cost of ten indexes on a hot table, and the cases where a sequential scan legitimately beats an index scan. |
| 18 | `18-the-query-planner-and-explain.md` | How cost is estimated (page reads, CPU tuple cost, row estimates), EXPLAIN and EXPLAIN ANALYZE read field by field, estimated-vs-actual row skew as the master diagnostic. |
| 19 | `19-join-algorithms-and-execution-strategies.md` | Nested loop, hash join, merge join, and the sort/aggregate/materialise nodes around them — the cost model of each and what the planner's choice tells you about your schema. |

---

### PHASE 3 — DATABASE DESIGN (9 topics)

| # | File | What this file teaches |
|---|------|------------------------|
| 20 | `20-entity-relationship-modelling.md` | Going from a paragraph of requirements to entities, attributes, relationships, cardinality (1:1, 1:N, M:N) and participation — with a repeatable interrogation method, not intuition. |
| 21 | `21-translating-er-to-relational-schema.md` | The mechanical rules: entity→table, 1:N→foreign key on the many side, M:N→junction table, 1:1→which side owns the key, weak entities, and how to model inheritance. |
| 22 | `22-primary-keys-in-depth.md` | Natural vs surrogate keys; bigserial vs UUIDv4 vs UUIDv7/ULID — index fragmentation, page splits, enumeration attacks, sortability, distributed generation — and a hard recommendation per case. |
| 23 | `23-foreign-keys-and-referential-integrity.md` | What a FK constraint actually does at execution time (the implied lookup, the trigger, the required index on the child), CASCADE/SET NULL/RESTRICT semantics, and the real write cost. |
| 24 | `24-constraints-in-depth.md` | NOT NULL, UNIQUE, CHECK, DEFAULT, EXCLUSION — how each is physically enforced, why a unique constraint is an index, and the honest argument for DB-level vs application-level validation. |
| 25 | `25-data-types-and-physical-storage.md` | The byte cost and behaviour of int/bigint/numeric/float, varchar/text, timestamp vs timestamptz, JSONB vs JSON vs text, arrays and enums — and what breaks when you pick wrong at 100M rows. |
| 26 | `26-modelling-time-money-and-identity.md` | Timezones done correctly, why money is never a float, intervals and recurrence, audit columns, soft deletes, and versioned/temporal rows — the four domains that ruin schemas most often. |
| 27 | `27-schema-patterns-and-antipatterns.md` | EAV, polymorphic foreign keys, single-table inheritance, status enums vs lookup tables, multi-tenant strategies, nullable-column sprawl — what each costs and the safer alternative. |
| 28 | `28-schema-versioning-and-zero-downtime-migrations.md` | Migrations as code, which DDL takes which lock, the expand→backfill→contract pattern, adding NOT NULL and indexes without downtime, and rolling out a rename across a deployed Node.js fleet. |

---

### PHASE 4 — NORMALISATION (10 topics)

| # | File | What this file teaches |
|---|------|------------------------|
| 29 | `29-what-normalisation-is.md` | The three anomalies (insert, update, delete) demonstrated on a real flat table, the "one fact in one place" principle, and the cost paid in join count. |
| 30 | `30-functional-dependencies-and-keys.md` | Reading `X → Y` off a schema, full vs partial vs transitive dependencies, closure, candidate keys, superkeys, prime attributes — the maths every normal form is defined in terms of. |
| 31 | `31-first-normal-form.md` | Atomicity, repeating groups, comma-separated columns and `phone1/phone2/phone3` — how to detect and fix each, and the honest modern position on arrays and JSONB vs 1NF. |
| 32 | `32-second-normal-form.md` | Partial dependency on a composite key, why it only appears with composite keys, detection procedure and the decomposition that fixes it. |
| 33 | `33-third-normal-form.md` | Transitive dependency (non-key → non-key), the "depends on the key, the whole key, and nothing but the key" formulation, and the decomposition with before/after schemas. |
| 34 | `34-boyce-codd-normal-form.md` | Where 3NF is not enough, overlapping candidate keys, the BCNF rule, and the dependency-preservation trade-off that sometimes makes 3NF the correct final answer. |
| 35 | `35-fourth-normal-form.md` | Multi-valued dependencies, the combinatorial row explosion they cause, how to spot one in a real schema, and the decomposition. |
| 36 | `36-fifth-normal-form-and-beyond.md` | Join dependencies, lossless decomposition, 5NF/PJNF and domain-key normal form — what they mean and the rare practical cases where you actually meet them. |
| 37 | `37-normalisation-worked-example.md` | One messy spreadsheet-style orders table taken through 1NF→2NF→3NF→BCNF step by step, every dependency named, every intermediate schema printed. |
| 38 | `38-when-to-stop-normalising.md` | The cost curve of over-normalisation, join count vs correctness, why most production schemas target 3NF/BCNF, and the decision framework for stopping. |

---

### PHASE 5 — TRANSACTIONS & CONCURRENCY (14 topics)

| # | File | What this file teaches |
|---|------|------------------------|
| 39 | `39-what-a-transaction-is.md` | The unit of work, BEGIN/COMMIT/ROLLBACK at the engine level, what a transaction ID is, savepoints, and why partial writes are catastrophic. |
| 40 | `40-acid-in-depth.md` | Atomicity, Consistency, Isolation, Durability defined precisely, what violates each, and exactly which engine component enforces each one. |
| 41 | `41-the-write-ahead-log.md` | Why logging beats flushing, the WAL record format, LSNs, the log buffer, fsync and commit, group commit, synchronous_commit levels, and WAL's second life as the replication stream. |
| 42 | `42-crash-recovery-and-checkpoints.md` | What happens on restart after a crash: checkpoints, the redo phase, the undo problem, ARIES concepts, and why recovery time is a configuration decision you make in advance. |
| 43 | `43-concurrency-anomalies.md` | Dirty read, non-repeatable read, phantom read, lost update, read skew, write skew — each with an exact two-transaction timeline and the exact business damage it causes. |
| 44 | `44-transaction-isolation-levels.md` | READ UNCOMMITTED → SERIALIZABLE, which anomaly each level permits, what the SQL standard says vs what PostgreSQL and MySQL actually do, and why READ COMMITTED is the default. |
| 45 | `45-locks-in-depth.md` | Shared vs exclusive, row/page/table/advisory levels, the lock compatibility matrix, lock queues (why one blocked writer blocks all readers behind it), and lock timeouts. |
| 46 | `46-mvcc-in-depth.md` | xmin/xmax on every tuple, snapshots, visibility rules, why readers never block writers, and the physical truth that an UPDATE is an INSERT plus a tombstone. |
| 47 | `47-vacuum-dead-tuples-and-bloat.md` | Where dead tuples come from, what VACUUM and VACUUM FULL do, autovacuum tuning, transaction ID wraparound, table and index bloat measurement, and CLUSTER/REINDEX. |
| 48 | `48-deadlocks.md` | The classic two-transaction deadlock traced line by line, the wait-for graph detector, victim selection, and the ordering discipline that makes deadlocks structurally impossible. |
| 49 | `49-optimistic-vs-pessimistic-concurrency.md` | SELECT FOR UPDATE vs version-column compare-and-swap, contention maths, retry loops, and both patterns implemented properly in a Node.js service layer. |
| 50 | `50-serializable-snapshot-isolation.md` | How true SERIALIZABLE is implemented without locking everything — dangerous structures, predicate locks, serialization failures, and designing an app that retries them correctly. |
| 51 | `51-distributed-transactions-and-2pc.md` | The prepare and commit phases, the coordinator, every failure point and its outcome, why 2PC blocks, and the saga pattern as the usual alternative. |
| 52 | `52-idempotency-and-transactional-messaging.md` | The dual-write problem (DB commit + queue publish), the transactional outbox, idempotency keys, exactly-once as an illusion, and the correct Node.js handler shape. |

---

### PHASE 6 — DENORMALISATION & PERFORMANCE (9 topics)

| # | File | What this file teaches |
|---|------|------------------------|
| 53 | `53-what-denormalisation-is.md` | Deliberate redundancy, the read/write asymmetry that justifies it, and the consistency debt you take on the moment you copy a fact. |
| 54 | `54-when-to-denormalise.md` | The measured signals that justify it, the cheaper fixes to exhaust first (index, query rewrite, cache), and a decision framework with a single clear go/no-go signal. |
| 55 | `55-denormalisation-patterns.md` | Duplicated columns, precomputed aggregates, embedded/JSONB blobs, summary and rollup tables, star-schema-style modelling — each with its refresh strategy and failure mode. |
| 56 | `56-materialised-views.md` | Views vs materialised views physically, full vs concurrent vs incremental refresh, staleness windows, indexing a matview, and when it replaces denormalisation entirely. |
| 57 | `57-caching-as-an-alternative.md` | Cache-aside, write-through, write-behind, TTL vs event invalidation, stampedes and negative caching, and the honest limits of putting Redis in front of a slow query. |
| 58 | `58-read-replicas.md` | Read/write splitting, replication lag measured, read-your-own-writes breakage, routing rules per query class, and the sticky-primary pattern in a Node.js data layer. |
| 59 | `59-partitioning-in-depth.md` | Vertical vs horizontal partitioning; range, list and hash partitioning; partition pruning in EXPLAIN; local vs global indexes; detach-to-archive; and when partitioning makes things slower. |
| 60 | `60-sharding-in-depth.md` | Sharding vs partitioning, shard key selection as the highest-stakes decision, hotspots, cross-shard queries and joins, rebalancing/resharding pain, and how to know you truly need it. |
| 61 | `61-counters-aggregates-and-hot-rows.md` | Why one row updated 5,000 times a second serialises your whole API, and the fixes: sharded counters, batched increments, append-and-fold, queue-based aggregation. |

---

### PHASE 7 — RELIABILITY & SCALE (8 topics)

| # | File | What this file teaches |
|---|------|------------------------|
| 62 | `62-replication-in-depth.md` | Physical (WAL streaming) vs logical replication, synchronous vs asynchronous commit, replication slots, cascading replicas, and the durability-vs-latency dial. |
| 63 | `63-high-availability-and-failover.md` | Detecting a dead primary, promotion, split-brain and fencing, quorum and consensus (Raft in plain terms), and what your application must do during the 30 seconds of failover. |
| 64 | `64-backup-recovery-and-pitr.md` | Logical vs physical backups, base backup + WAL archive = point-in-time recovery, RPO/RTO defined and measured, retention design, and how to actually test a restore. |
| 65 | `65-connection-pooling.md` | Why a PostgreSQL connection is a process, what 10,000 connections does to a server, session vs transaction vs statement pooling, the sizing formula, and pool-exhaustion symptoms. |
| 66 | `66-the-n-plus-1-query-problem.md` | How ORMs generate 1,001 queries, detecting it in logs and traces, the four fixes (join, eager load, IN batch, DataLoader), and the latency arithmetic that makes it fatal. |
| 67 | `67-performance-investigation-methodology.md` | A repeatable procedure: symptom → pg_stat_statements → EXPLAIN ANALYZE → classify (I/O bound / plan / lock / bloat / app) → fix → measure — with the exact query at each step. |
| 68 | `68-cap-theorem-and-consistency-models.md` | CAP stated correctly (and PACELC), linearizable vs sequential vs causal vs eventual consistency, what each choice means for your Node.js code, and reading a vendor's claims critically. |
| 69 | `69-security-at-the-data-layer.md` | Least-privilege roles, row-level security, encryption in transit and at rest, why parameterised queries defeat injection at the parser, PII handling, and audit logging. |

---

### PHASE 8 — NoSQL DATA MODELLING (7 topics)

| # | File | What this file teaches |
|---|------|------------------------|
| 70 | `70-document-database-modelling.md` | Embed vs reference as the central decision, the one-to-few / one-to-many / one-to-squillions rule, document growth and the 16MB limit, and MongoDB's index and write-concern realities. |
| 71 | `71-key-value-store-design.md` | Key namespace design, TTL patterns, Redis strings/hashes/lists/sets/sorted sets/streams and the exact problem each solves, plus persistence and eviction implications. |
| 72 | `72-wide-column-design.md` | Partition key and clustering key, the physical layout of a wide row, query-first schema design, tunable consistency, and why you duplicate a table per query pattern. |
| 73 | `73-time-series-data-modelling.md` | The insert-heavy/range-read/downsample access pattern, why B-tree OLTP schemas struggle, hypertables and chunking, retention and continuous aggregates. |
| 74 | `74-graph-and-search-data-modelling.md` | When relationships are the data (traversal depth as the signal), property-graph modelling, and inverted-index/search modelling — analysers, relevance, and why search is not a database. |
| 75 | `75-choosing-between-sql-and-nosql.md` | A genuine decision framework: access patterns, consistency needs, schema volatility, scale shape, operational cost and team familiarity — with the failure mode of each wrong choice. |
| 76 | `76-polyglot-persistence-and-cdc.md` | Running several stores without corrupting them: the dual-write problem, change data capture from the WAL, event-carried state transfer, and where the source of truth lives. |

---

### PHASE 9 — CAPSTONE & SYNTHESIS (3 topics)

| # | File | What this file teaches |
|---|------|------------------------|
| 77 | `77-capstone-design-the-schema.md` | Full design of a multi-tenant SaaS commerce schema from requirements to normalised DDL — every key, constraint, index and type decision justified against Phases 1–5. |
| 78 | `78-capstone-scale-it.md` | The same system at 10M users: profile it, index it, denormalise where measured, partition, replicate, cache, pool — with the reasoning trail for every change. |
| 79 | `79-principal-engineer-review-checklist.md` | The condensed review checklist and the interview-grade answer bank — how to audit any schema or database incident in 20 minutes using everything in this curriculum. |

---

## PROGRESS TRACKER

Tick each topic when the doc is written **and** all three exercises pass.

### Phase 1 — Foundations & Physical Storage
- [x] 01 — What is a database
- [x] 02 — Database engine architecture
- [x] 03 — Types of databases and data models
- [x] 04 — How data is stored on disk
- [x] 05 — Row storage format and tuple layout
- [x] 06 — Row store vs column store
- [x] 07 — The buffer pool
- [x] 08 — Storage engines: B-tree vs LSM
- [x] 09 — The life of a query  ← **PHASE 1 COMPLETE**

### Phase 2 — Indexes
- [x] 10 — What an index is
- [x] 11 — B-tree indexes in depth
- [x] 12 — Index lookup end to end
- [x] 13 — Types of indexes
- [x] 14 — Composite index column order
- [x] 15 — Selectivity, cardinality and statistics
- [x] 16 — Specialised index types (Hash, GIN, GiST, BRIN)
- [x] 17 — When indexes hurt
- [x] 18 — The query planner and EXPLAIN
- [x] 19 — Join algorithms and execution strategies  ← **PHASE 2 COMPLETE**

### Phase 3 — Database Design
- [x] 20 — Entity-relationship modelling
- [x] 21 — Translating ER to relational schema
- [x] 22 — Primary keys in depth
- [x] 23 — Foreign keys and referential integrity
- [x] 24 — Constraints in depth
- [x] 25 — Data types and physical storage
- [x] 26 — Modelling time, money and identity
- [x] 27 — Schema patterns and antipatterns
- [x] 28 — Schema versioning and zero-downtime migrations  ← **PHASE 3 COMPLETE**

### Phase 4 — Normalisation
- [x] 29 — What normalisation is
- [x] 30 — Functional dependencies and keys
- [x] 31 — First normal form
- [x] 32 — Second normal form
- [x] 33 — Third normal form
- [x] 34 — Boyce-Codd normal form
- [x] 35 — Fourth normal form
- [x] 36 — Fifth normal form and beyond
- [x] 37 — Normalisation worked example
- [x] 38 — When to stop normalising  ← **PHASE 4 COMPLETE**

### Phase 5 — Transactions & Concurrency
- [x] 39 — What a transaction is
- [x] 40 — ACID in depth
- [x] 41 — The write-ahead log
- [x] 42 — Crash recovery and checkpoints
- [x] 43 — Concurrency anomalies
- [x] 44 — Transaction isolation levels
- [x] 45 — Locks in depth
- [x] 46 — MVCC in depth
- [x] 47 — VACUUM, dead tuples and bloat
- [x] 48 — Deadlocks
- [x] 49 — Optimistic vs pessimistic concurrency
- [x] 50 — Serializable snapshot isolation
- [x] 51 — Distributed transactions and 2PC
- [x] 52 — Idempotency and transactional messaging  ← **PHASE 5 COMPLETE**

### Phase 6 — Denormalisation & Performance
- [x] 53 — What denormalisation is
- [x] 54 — When to denormalise
- [x] 55 — Denormalisation patterns
- [x] 56 — Materialised views
- [x] 57 — Caching as an alternative
- [x] 58 — Read replicas
- [x] 59 — Partitioning in depth
- [x] 60 — Sharding in depth
- [x] 61 — Counters, aggregates and hot rows  ← **PHASE 6 COMPLETE**

### Phase 7 — Reliability & Scale
- [x] 62 — Replication in depth
- [x] 63 — High availability and failover
- [ ] 64 — Backup, recovery and PITR
- [ ] 65 — Connection pooling
- [ ] 66 — The N+1 query problem
- [ ] 67 — Performance investigation methodology
- [ ] 68 — CAP theorem and consistency models
- [ ] 69 — Security at the data layer

### Phase 8 — NoSQL Data Modelling
- [ ] 70 — Document database modelling
- [ ] 71 — Key-value store design
- [ ] 72 — Wide-column design
- [ ] 73 — Time-series data modelling
- [ ] 74 — Graph and search data modelling
- [ ] 75 — Choosing between SQL and NoSQL
- [ ] 76 — Polyglot persistence and CDC

### Phase 9 — Capstone
- [ ] 77 — Capstone: design the schema
- [ ] 78 — Capstone: scale it
- [ ] 79 — Principal engineer review checklist

**Completed: 63 / 79**  ·  Case studies: 7 / 23

---

## COMPANION FOLDER — PRODUCTION CASE STUDIES

`/docs/db-case-studies/` — 23 files. Where this curriculum's theory becomes judgement.

Each case study takes a real system at real traffic (flash sales at 80k req/s, ride
dispatch, payment ledgers, social feed fan-out, multi-tenant SaaS), shows the naive
schema, breaks it at a specific measured load, and rebuilds it to production standard
with every index, constraint, partition and denormalisation justified.

Read `/docs/db-case-studies/00-index.md` for the full list and the eight-step design
method used identically in every one. They can be read in parallel with this curriculum —
each case study lists which topics it leans on.

---

## HOW EACH TOPIC IS TAUGHT

Every doc follows the mandatory template, in this exact order, with no section omitted:

```
1.  ELI5 — the simple analogy               ← the mental anchor
2.  Where this fits in the big picture      ← journey diagram
3.  What is this?                           ← plain English, 2–3 sentences
4.  Why does it matter for a backend dev?   ← make the pain real
5.  The physical reality                    ← MANDATORY: bytes, pages, files
6.  How it works — step by step             ← full numbered trace, reads & writes noted
7.  Concept breakdown                       ← decompose the term itself
8.  Diagram(s)                              ← MINIMUM 2: big picture → data flow → before/after
9.  Example 1 — basic                       ← one concept, isolated
10. Example 2 — production scenario         ← real Node.js system under load
11. Common mistakes                         ← what they do → symptom → engine cause → diagnose → fix
12. Hands-on proof                          ← runnable commands + expected output
13. The design decision framework           ← USE WHEN / AVOID WHEN / THE SIGNAL
14. Practice exercises                      ← easy / medium / hard (production sim)
15. Mental model checkpoint                 ← 5–7 open questions from memory
16. Quick reference card                    ← concepts, numbers to memorise, decisions
17. When would I use this at work?          ← 3 concrete backend scenarios
18. Connected topics                        ← prerequisites in, unlocks out
```

Non-negotiables: physical storage reality in every topic, minimum two diagrams, a design
decision framework wherever there is a trade-off, realistic table names only
(`users`, `orders`, `products`, `payments`, `sessions`) — never `foo`/`bar`.

---

## SESSION COMMANDS

| You type | What happens |
|----------|--------------|
| `START` | Topic 01 is generated in full |
| *(paste exercise answers)* | Each marked PASS or NEEDS WORK; NEEDS WORK gets an engine-level explanation + one hint and a retry |
| `NEXT` | Next topic in the plan is generated (only after all 3 exercises pass) |
| `HINT` | One nudge — never the answer |
| `PROGRESS` | This checklist reprinted with completed topics ticked |
| `REDO [topic]` | That doc regenerated completely from scratch |
| `SKIP` | Move on without passing exercises (tracked as incomplete in the tracker) |

After every doc I say exactly:

> Doc saved: /docs/databases/[XX]-[topic-name].md
> Complete all 3 exercises. Paste your answers when done.

---

## ENVIRONMENT FOR THE HANDS-ON SECTIONS

The proof commands assume:

- **PostgreSQL 15+** — with `pageinspect`, `pgstattuple`, `pg_stat_statements`, `pg_buffercache`
  extensions available (all ship in `contrib`)
- **MongoDB 6+** — for the Phase 8 document-modelling topics
- **Redis 7+** — for the caching and key-value topics
- A **Node.js** project with `pg` (or Prisma/Knex) to reproduce the application-side scenarios

Docker one-liner used throughout:

```bash
docker run -d --name pg-lab -e POSTGRES_PASSWORD=lab -p 5432:5432 postgres:16
docker exec -it pg-lab psql -U postgres
```

Nothing in the curriculum requires a production database. Every proof runs on a laptop.

---

## FILE INDEX (flat list, copy-paste ready)

```
/docs/databases/00-master-plan.md
/docs/databases/01-what-is-a-database.md
/docs/databases/02-database-engine-architecture.md
/docs/databases/03-types-of-databases-and-data-models.md
/docs/databases/04-how-data-is-stored-on-disk.md
/docs/databases/05-row-storage-format-and-tuple-layout.md
/docs/databases/06-row-store-vs-column-store.md
/docs/databases/07-the-buffer-pool.md
/docs/databases/08-storage-engines-btree-vs-lsm.md
/docs/databases/09-the-life-of-a-query.md
/docs/databases/10-what-an-index-is.md
/docs/databases/11-btree-indexes-in-depth.md
/docs/databases/12-index-lookup-end-to-end.md
/docs/databases/13-types-of-indexes.md
/docs/databases/14-composite-index-column-order.md
/docs/databases/15-selectivity-cardinality-and-statistics.md
/docs/databases/16-specialised-index-types.md
/docs/databases/17-when-indexes-hurt.md
/docs/databases/18-the-query-planner-and-explain.md
/docs/databases/19-join-algorithms-and-execution-strategies.md
/docs/databases/20-entity-relationship-modelling.md
/docs/databases/21-translating-er-to-relational-schema.md
/docs/databases/22-primary-keys-in-depth.md
/docs/databases/23-foreign-keys-and-referential-integrity.md
/docs/databases/24-constraints-in-depth.md
/docs/databases/25-data-types-and-physical-storage.md
/docs/databases/26-modelling-time-money-and-identity.md
/docs/databases/27-schema-patterns-and-antipatterns.md
/docs/databases/28-schema-versioning-and-zero-downtime-migrations.md
/docs/databases/29-what-normalisation-is.md
/docs/databases/30-functional-dependencies-and-keys.md
/docs/databases/31-first-normal-form.md
/docs/databases/32-second-normal-form.md
/docs/databases/33-third-normal-form.md
/docs/databases/34-boyce-codd-normal-form.md
/docs/databases/35-fourth-normal-form.md
/docs/databases/36-fifth-normal-form-and-beyond.md
/docs/databases/37-normalisation-worked-example.md
/docs/databases/38-when-to-stop-normalising.md
/docs/databases/39-what-a-transaction-is.md
/docs/databases/40-acid-in-depth.md
/docs/databases/41-the-write-ahead-log.md
/docs/databases/42-crash-recovery-and-checkpoints.md
/docs/databases/43-concurrency-anomalies.md
/docs/databases/44-transaction-isolation-levels.md
/docs/databases/45-locks-in-depth.md
/docs/databases/46-mvcc-in-depth.md
/docs/databases/47-vacuum-dead-tuples-and-bloat.md
/docs/databases/48-deadlocks.md
/docs/databases/49-optimistic-vs-pessimistic-concurrency.md
/docs/databases/50-serializable-snapshot-isolation.md
/docs/databases/51-distributed-transactions-and-2pc.md
/docs/databases/52-idempotency-and-transactional-messaging.md
/docs/databases/53-what-denormalisation-is.md
/docs/databases/54-when-to-denormalise.md
/docs/databases/55-denormalisation-patterns.md
/docs/databases/56-materialised-views.md
/docs/databases/57-caching-as-an-alternative.md
/docs/databases/58-read-replicas.md
/docs/databases/59-partitioning-in-depth.md
/docs/databases/60-sharding-in-depth.md
/docs/databases/61-counters-aggregates-and-hot-rows.md
/docs/databases/62-replication-in-depth.md
/docs/databases/63-high-availability-and-failover.md
/docs/databases/64-backup-recovery-and-pitr.md
/docs/databases/65-connection-pooling.md
/docs/databases/66-the-n-plus-1-query-problem.md
/docs/databases/67-performance-investigation-methodology.md
/docs/databases/68-cap-theorem-and-consistency-models.md
/docs/databases/69-security-at-the-data-layer.md
/docs/databases/70-document-database-modelling.md
/docs/databases/71-key-value-store-design.md
/docs/databases/72-wide-column-design.md
/docs/databases/73-time-series-data-modelling.md
/docs/databases/74-graph-and-search-data-modelling.md
/docs/databases/75-choosing-between-sql-and-nosql.md
/docs/databases/76-polyglot-persistence-and-cdc.md
/docs/databases/77-capstone-design-the-schema.md
/docs/databases/78-capstone-scale-it.md
/docs/databases/79-principal-engineer-review-checklist.md
```

---

## THE THREAD THAT RUNS THROUGH EVERYTHING

One system is used as the running example across all 79 topics, so every concept lands on
the same schema you already have in your head:

```
An e-commerce backend, Node.js + PostgreSQL (+ MongoDB, Redis in Phase 8/6)

  users ──< orders ──< order_items >── products
    │         │                           │
    │         └──< payments               └──< inventory
    └──< sessions                              │
                                          categories

Phase 1–2:  how one `users` row and its index physically live on disk
Phase 3–4:  designing and normalising this exact schema from requirements
Phase 5:    two customers buying the last unit of the same product
Phase 6:    the product page doing 9 joins at 3,000 req/s
Phase 7:    the primary dies during a Black Friday sale
Phase 8:    moving the product catalogue to a document store — and whether you should
Phase 9:    the whole thing at 10 million users
```

You will end this curriculum having designed, broken, diagnosed, and scaled the same system
seven different ways.

---

*Plan version 1.0 — created 2026-08-07*
