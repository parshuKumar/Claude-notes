# 08 — Storage Engines: B-tree vs LSM
## Phase: Storage Internals

---

## ELI5 — The Simple Analogy

Two ways to keep a class register up to date.

**The librarian (B-tree).** She keeps one master book, alphabetical. When a new student joins, she finds the right page, and if it's full she splits it into two pages and rewrites both. Looking anyone up is instant — flip to the right page. But *every single change* means finding the exact page, possibly rewriting several. Writing is careful and slow. Reading is perfect.

**The note-taker (LSM).** He never edits the master book. He keeps a small notepad on his desk and scribbles every change as it comes — fast, no searching, no page splits. When the notepad fills, he copies it into a new sealed booklet and starts a fresh notepad. Over time he has dozens of sealed booklets. Looking someone up means checking the notepad, then booklet 1, then booklet 2… so reads get slower and slower. So every night he **merges** booklets together into bigger ones. That merging is **compaction**, and it never stops.

Writing is fast for the note-taker, slow for the librarian. Reading is the reverse. That's the entire trade.

---

## Where this fits in the big picture

```
   04 Pages ──▶ 05 Tuples ──▶ 06 Row vs column ──▶ 07 Buffer pool
                                                        │
                                                        ▼
                                        08 B-TREE vs LSM  ← YOU ARE HERE
                                        (how writes reach disk)
                                                        │
                          ┌─────────────────────────────┼──────────────┐
                          ▼                             ▼              ▼
                    11 B-tree in depth          17 write            72 wide-column
                    (the read side)             amplification       (LSM in practice)
```

Topic 06 was *how bytes are arranged*. This is *how bytes get written*. Together they explain almost every difference between database engines.

---

## What is this?

A **storage engine** is the component that decides how data physically reaches disk and how it's found again. Two families dominate:

- **B-tree / update-in-place**: data lives in a sorted, balanced tree of pages. A write finds the page and modifies it. PostgreSQL, MySQL/InnoDB, Oracle, SQL Server, SQLite.
- **LSM tree (Log-Structured Merge)**: writes go into a sorted in-memory structure, which is periodically flushed to immutable sorted files on disk, which are periodically merged. RocksDB, LevelDB, Cassandra, ScyllaDB, HBase, MongoDB/WiredTiger (optional), ClickHouse (MergeTree is LSM-shaped), most modern KV stores.

---

## Why does it matter for a backend developer?

Because it explains, mechanically, why engines behave the way they do — and it kills a lot of cargo-cult advice.

- **"Cassandra is fast for writes."** True, and this is *why*: an LSM write is an append to a memory buffer plus a commit-log append. No page lookup, no page split, no random I/O. It is genuinely O(1)-ish.
- **"Cassandra is bad at reads."** Also true, and *why*: a read may have to check the memtable plus N SSTables. Bloom filters and caches hide most of it, but the tail latency is structurally worse.
- **"Why is my Cassandra cluster doing 400 MB/s of disk I/O when we only write 20 MB/s?"** Compaction. Write amplification in a levelled LSM is typically 10–30×.
- **"Why does PostgreSQL slow down on random UUID primary keys?"** B-tree page splits. Random insertion order means every insert lands in a different page, each needing a read-modify-write, and splits fragment the index. (Topic 22.)
- **And the big one:** if you're choosing between PostgreSQL and Cassandra/DynamoDB for a write-heavy workload, this topic is the actual decision, not the marketing.

---

## The physical reality

### B-tree — update in place

```
 A B+tree index on orders(id). Every node is one 8 KB PAGE.

                        ┌─────────── ROOT (page 0) ───────────┐
                        │ [•|500|•|1000|•|1500|•]             │
                        └───┬─────┬──────┬───────┬────────────┘
              ┌─────────────┘     │      │       └──────────────┐
              ▼                   ▼      ▼                      ▼
      ┌───────────────┐   ┌──────────────┐   ┌──────────────────────┐
      │ INTERNAL      │   │ INTERNAL     │   │ INTERNAL             │
      │ [•|100|•|200|•]│   │ [•|600|•|...]│   │ [•|1600|•|...]      │
      └───┬───────────┘   └──────────────┘   └──────────────────────┘
          ▼
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │ LEAF         │──▶│ LEAF         │──▶│ LEAF         │  ← sibling pointers
   │ 1→(0,1)      │   │ 101→(4,2)    │   │ 201→(9,7)    │
   │ 2→(0,2)      │   │ 102→(4,3)    │   │ 202→(9,8)    │
   │ ...(≈200)    │   │ ...          │   │ ...          │
   └──────────────┘   └──────────────┘   └──────────────┘

 WRITE PATH for INSERT id=150:
   1. Descend root → internal → leaf                    [3 page READS]
   2. Leaf has room? Insert in sorted position.          [1 page WRITE]
   3. Leaf FULL? SPLIT:
        - allocate a new page
        - move half the entries there
        - insert a separator key into the parent
        - parent full too? split it. Possibly up to the root.
        [up to 2 × tree_height page WRITES + WAL for all of them]

 ⇒ Every write is a READ-MODIFY-WRITE of a random page.
   Every write is potentially several.
```

**On-disk state after a B-tree write:** the page is modified *in place*. The old value is gone (in the index; in the heap MVCC keeps versions — Topic 46). There is exactly one file, and it is always fully sorted and immediately readable.

### LSM tree — never update, only append and merge

```
 ┌───────────────────────── MEMORY ──────────────────────────┐
 │  MEMTABLE  (a skiplist or red-black tree, sorted)         │
 │    id=150 → {...}                                          │
 │    id=203 → {...}                                          │
 │    id=891 → TOMBSTONE                                      │
 │  writes go here. O(log n) in RAM. NO DISK SEEK.            │
 └────────────────────┬──────────────────────────────────────┘
                      │ (durability: every write ALSO appends to a
                      │  sequential commit log / WAL on disk)
                      │
        memtable full (e.g. 64 MB) → FLUSH, immutable
                      ▼
 ┌───────────────────────── DISK ────────────────────────────┐
 │ LEVEL 0   ┌────────┐┌────────┐┌────────┐┌────────┐        │
 │           │SSTable ││SSTable ││SSTable ││SSTable │        │
 │           │  64MB  ││  64MB  ││  64MB  ││  64MB  │        │
 │           └────────┘└────────┘└────────┘└────────┘        │
 │           ⚠ L0 files OVERLAP in key range                 │
 │                        │ compaction                        │
 │                        ▼                                   │
 │ LEVEL 1   ┌──────────────────────────────┐   ~300 MB       │
 │           │ SSTables, NON-overlapping    │                 │
 │           └──────────────────────────────┘                 │
 │                        │ compaction                        │
 │                        ▼                                   │
 │ LEVEL 2   ┌──────────────────────────────────────┐  ~3 GB  │
 │           └──────────────────────────────────────┘         │
 │                        ▼                                   │
 │ LEVEL 3 ... each level ~10× the previous                   │
 └────────────────────────────────────────────────────────────┘
```

**An SSTable (Sorted String Table) on disk:**

```
 ┌──────────────────────────────────────────────────────────────┐
 │ DATA BLOCKS  (sorted key→value, compressed, ~4–64 KB each)   │
 │   [k1:v1][k2:v2][k3:v3]...                                   │
 │   [k40:v40][k41:v41]...                                      │
 ├──────────────────────────────────────────────────────────────┤
 │ INDEX BLOCK   first key of each data block → block offset     │
 ├──────────────────────────────────────────────────────────────┤
 │ BLOOM FILTER  ~10 bits/key → "is key K definitely absent?"    │
 │               1% false-positive rate at 10 bits/key           │
 ├──────────────────────────────────────────────────────────────┤
 │ FOOTER        min key, max key, count, checksums              │
 └──────────────────────────────────────────────────────────────┘

 IMMUTABLE. Written once, sequentially. Never modified. Only deleted
 (after its contents have been merged into a lower level).
```

**How a delete works in an LSM — this surprises everyone:**

```
 DELETE key=891  →  writes a TOMBSTONE record: (891, <deleted>)

 The old value in L2 is STILL THERE. The tombstone in L0 shadows it.
 A read finds the tombstone first (newest wins) and returns "not found."
 The actual bytes are only removed when compaction merges L0 and L2 and
 sees both — and only if no snapshot still needs the old value.

 ⇒ Deleting data can INCREASE disk usage until compaction runs.
 ⇒ Cassandra's infamous `gc_grace_seconds` (default 10 days) exists
   because a tombstone must survive long enough for every replica to
   see it, or the deleted data "resurrects."
```

---

## How it works — step by step

### The three amplification factors — the vocabulary that makes this precise

```
 WRITE AMPLIFICATION (WA)
   bytes written to disk ÷ bytes written by the application

 READ AMPLIFICATION (RA)
   disk reads performed ÷ logical reads requested

 SPACE AMPLIFICATION (SA)
   bytes on disk ÷ bytes of live data
```

Every storage engine trades these three against each other. **You cannot optimise all three** — this is often called the RUM conjecture (Read, Update, Memory), and it is the single most useful frame for comparing engines.

### B-tree: one write, traced

```
 INSERT INTO orders (id, ...) VALUES (4102991, ...)
 with a PRIMARY KEY b-tree and 3 secondary indexes.

 1. Heap: find a page with room (FSM), write 88-byte tuple.
      WAL record ~120 B.  Page dirtied: 8 KB.
 2. PK index: descend 3 levels, insert into leaf.
      WAL ~80 B. Page dirtied: 8 KB.
 3. Secondary index ×3: same.
      WAL ~240 B. Pages dirtied: 24 KB.
 4. If any leaf split: +2 pages per split, + parent update.
 5. FULL PAGE IMAGE: the FIRST time each page is touched after a
    checkpoint, the WHOLE 8 KB page goes into the WAL.
      → 5 pages × 8 KB = 40 KB of WAL for an 88-byte insert.

 STEADY STATE (amortised over a checkpoint interval):
   WA ≈ 10–40× depending on index count and checkpoint frequency
   RA ≈ 1  (one tree descent, mostly cached)
   SA ≈ 1.2–2× (fillfactor + MVCC dead tuples + index bloat)
```

### LSM: one write, traced

```
 PUT key=4102991, value={...}

 1. Append to the commit log (sequential, no seek).     ~120 B
 2. Insert into the memtable (RAM skiplist).            0 disk I/O
 3. Return to client.  ← DONE. Latency: microseconds.

 ...later, asynchronously:
 4. Memtable full (64 MB) → flush as an L0 SSTable.     64 MB sequential write
 5. Compaction L0→L1: read 4 L0 files + overlapping L1 files,
    merge-sort, write new L1 files, delete old.
 6. Compaction L1→L2, L2→L3 ...

 LEVELLED COMPACTION amplification:
   Each level is ~10× the previous, so each key is rewritten ~10× per level.
   With 5 levels:   WA ≈ 10 × 5 = 50×  (worst case)
                    typical measured: 10–30×
   RA ≈ number of levels + L0 files ≈ 5–15 SSTable lookups per read
        (mostly eliminated by bloom filters — a bloom filter says
         "definitely not here" in ~1 memory probe)
   SA ≈ 1.1×  ← BEST of any engine. Levelled compaction keeps at most
                ~10% obsolete data.

 SIZE-TIERED COMPACTION (Cassandra's default) trades differently:
   WA ≈ 4–10×   ← BETTER writes
   RA worse (more overlapping files)
   SA ≈ 2×+     ← WORSE space; needs 50% free disk headroom for a
                  major compaction. This is a real operational gotcha.
```

### The read path, side by side

```
 B-TREE READ: get key 4102991
   root page      → cached, ~100 ns
   internal page  → cached, ~100 ns
   leaf page      → maybe cached, else 100 µs
   heap fetch     → maybe cached, else 100 µs
   ─────────────────────────────────────────
   1–2 disk reads worst case. PREDICTABLE.

 LSM READ: get key 4102991
   memtable                 → RAM check
   L0 SSTable 1 → bloom     → "no"   (1 RAM probe)
   L0 SSTable 2 → bloom     → "no"
   L0 SSTable 3 → bloom     → "MAYBE" → index block → data block  [1 disk read]
                            → key not actually there (1% false positive)
   L1 → bloom → "no"
   L2 → bloom → "MAYBE" → index → data                            [1 disk read]
                            → FOUND
   ─────────────────────────────────────────
   0–N disk reads. Bloom filters make the average ~1.
   But p99.9 is structurally worse: a read that misses every bloom
   filter still touched every level.
```

---

## Concept breakdown

```
B-TREE / UPDATE-IN-PLACE
│
├── SORTED, BALANCED    all data always in one sorted structure
├── IN-PLACE MUTATION   find the page, modify it
├── RANDOM WRITE I/O    each write touches a different page
├── PAGE SPLITS         the expensive event; caused by insertion order
├── STRENGTH            predictable reads, perfect range scans, easy MVCC
└── WEAKNESS            write throughput ceiling; random-key inserts are brutal

LSM TREE
│
├── MEMTABLE            in-RAM sorted structure absorbing all writes
├── COMMIT LOG / WAL    sequential append for durability of the memtable
├── SSTABLE             immutable, sorted, compressed file with a bloom filter
├── COMPACTION          background merge of SSTables; the engine's heartbeat
│    ├── LEVELLED       low space amp, high write amp (RocksDB default)
│    ├── SIZE-TIERED    low write amp, high space amp (Cassandra default)
│    └── TIME-WINDOW    for time-series; compact only within a time bucket
├── TOMBSTONE           a delete marker; the data lives until compaction
├── STRENGTH            write throughput, sequential I/O, compression, space amp
└── WEAKNESS            read amplification, compaction I/O storms, tombstone
                        management, unpredictable tail latency

THE RUM CONJECTURE — you get two of three
│
│         READ optimised
│              ▲
│              │
│      B-tree ●│
│              │
│              └──────────▶ UPDATE (write) optimised
│             ╱                       ● LSM
│            ╱
│           ▼
│      MEMORY/SPACE optimised
│
└── Every engine picks a point on this surface. There is no free lunch.
    Understanding WHERE an engine sits tells you what it will be bad at
    before you benchmark it.

BLOOM FILTER  — why LSM reads aren't disastrous
│
├── A bit array + k hash functions
├── "Is key K in this SSTable?"  →  "definitely NO" or "probably yes"
├── 10 bits/key → ~1% false positive; 15 bits/key → ~0.1%
├── Kept in RAM. One probe, no disk I/O.
└── ⚠ Only answers point queries. RANGE scans cannot use bloom filters —
     they must open every SSTable whose min/max overlaps the range.
     This is why LSM range scans are much weaker than LSM point reads.
```

---

## Diagrams

**Diagram 1 — big picture: where writes go**

```
        B-TREE ENGINE                         LSM ENGINE
   ┌─────────────────────┐              ┌─────────────────────┐
   │   INSERT/UPDATE     │              │   INSERT/UPDATE     │
   └──────────┬──────────┘              └──────────┬──────────┘
              │                                    │
              ▼                                    ▼
      ┌───────────────┐                   ┌────────────────┐
      │ find the page │  ← RANDOM READ    │ append to log  │ ← SEQUENTIAL
      └───────┬───────┘                   └────────┬───────┘
              ▼                                    ▼
      ┌───────────────┐                   ┌────────────────┐
      │ modify page   │                   │ insert into    │ ← RAM ONLY
      │ (maybe split) │                   │ memtable       │
      └───────┬───────┘                   └────────┬───────┘
              ▼                                    │ DONE ✓ (µs)
      ┌───────────────┐                            ▼
      │ write WAL     │                   ┌────────────────┐
      │ dirty buffer  │                   │ flush → SSTable│ ← SEQUENTIAL,
      └───────┬───────┘                   └────────┬───────┘   ASYNC
              ▼                                    ▼
      ┌───────────────┐                   ┌────────────────┐
      │ checkpointer  │                   │  COMPACTION    │ ← SEQUENTIAL,
      │ RANDOM WRITE  │                   │  (forever)     │   CONTINUOUS
      └───────────────┘                   └────────────────┘
```

**Diagram 2 — data flow: the read path**

```
 B-TREE                              LSM
 ──────────────────────────          ──────────────────────────────────
      root (cached)                       memtable (RAM)
        │                                     │ miss
        ▼                                     ▼
    internal (cached)                    ┌─ bloom L0-1 ─┐ "no"
        │                                ├─ bloom L0-2 ─┤ "no"
        ▼                                ├─ bloom L0-3 ─┤ "maybe" → read block
      leaf                               ├─ bloom L1   ─┤ "no"
        │                                ├─ bloom L2   ─┤ "maybe" → read block ✓
        ▼                                └──────────────┘
     heap tuple
        │                                 up to N block reads,
        ▼                                 avg ~1 with good blooms,
     RESULT                               p99.9 much worse
   1–2 reads. Flat, predictable.
```

**Diagram 3 — before/after: what compaction actually does**

```
 BEFORE COMPACTION (L0 has 4 overlapping files)
   L0:  [a..z]  [a..z]  [a..z]  [a..z]     ← key ranges OVERLAP
        k=5:v1  k=5:v2  k=5:TOMB k=5:v3    ← 4 versions of key 5!
   L1:  [a..m] [n..z]
        k=5:v0

   Reading k=5 → check all 4 L0 files + L1. Return v3 (newest).
   Disk: 5 copies of key 5.  SA is bad.

 AFTER COMPACTION (merge L0 + overlapping L1 → new L1)
   L0:  (empty)
   L1:  [a..m] [n..z]
        k=5:v3          ← one version; v0,v1,v2 and the tombstone gone

   Reading k=5 → check L1. One file.
   Disk: 1 copy.

 THE COST: to reclaim that, the engine read 5 files and wrote 2 new ones.
 Bytes moved ≫ bytes changed. That is write amplification, made visible.
 And it happens CONTINUOUSLY, competing with your foreground traffic
 for exactly the same disk bandwidth.
```

---

## Example 1 — basic

You can observe both behaviours without leaving PostgreSQL, because PostgreSQL's *heap* is append-ish while its *indexes* are strict B-trees.

**Demonstrate the B-tree's fatal weakness: random insertion order.**

```sql
-- SEQUENTIAL keys (bigserial / UUIDv7 / ULID behaviour)
CREATE TABLE seq_keys (id bigint PRIMARY KEY, payload text);
\timing on
INSERT INTO seq_keys SELECT i, md5(i::text) FROM generate_series(1, 3000000) i;
```
```
Time: 8412.203 ms
```
```sql
-- RANDOM keys (UUIDv4 behaviour)
CREATE TABLE rnd_keys (id uuid PRIMARY KEY, payload text);
INSERT INTO rnd_keys SELECT gen_random_uuid(), md5(i::text) FROM generate_series(1, 3000000) i;
```
```
Time: 41208.771 ms          ← 4.9× SLOWER for identical row count
```
```sql
SELECT relname, pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_class WHERE relname IN ('seq_keys_pkey','rnd_keys_pkey');
```
```
    relname     |  size
----------------+---------
 seq_keys_pkey  | 64 MB
 rnd_keys_pkey  | 130 MB    ← 2× larger from page splits and fragmentation
```

**Why:** sequential keys always insert into the *rightmost* leaf page, which is always in cache, and PostgreSQL has a fast-path for it. Random keys insert into a random leaf every time — a random read, a modify, and frequent splits that leave every page ~50–70% full instead of ~90%.

This is the entire argument for UUIDv7/ULID over UUIDv4 (Topic 22), and it is *purely* a B-tree property. **In an LSM engine, random keys cost nothing extra** — writes go to a memtable that's sorted in RAM either way.

**Measure the write amplification of indexes:**

```sql
CREATE TABLE amp (id bigserial PRIMARY KEY, a bigint, b bigint, c bigint, d text);
CREATE INDEX ON amp(a); CREATE INDEX ON amp(b); CREATE INDEX ON amp(c);

SELECT pg_current_wal_lsn() AS before \gset
INSERT INTO amp (a,b,c,d) SELECT i,i,i,md5(i::text) FROM generate_series(1,200000) i;
SELECT pg_size_pretty(pg_current_wal_lsn() - :'before'::pg_lsn) AS wal_generated;
```
```
 wal_generated
---------------
 71 MB              ← for ~200,000 × 60 bytes = 12 MB of logical data
```
**WA ≈ 6×** — and that's with only 3 secondary indexes and no full-page images from a recent checkpoint. Drop the indexes and repeat: WAL drops to ~19 MB.

---

## Example 2 — production scenario

**The situation.** You are building the ingest path for an IoT platform: 400,000 sensor readings per second, each ~120 bytes, retained 90 days. Reads are: (a) "last 24 h for device X" (range scan), 2,000/s; (b) "aggregate across all devices for last hour," 50/s.

Someone proposes PostgreSQL because "we already run it." Someone else proposes Cassandra because "it's for time series." Do the arithmetic instead of arguing.

**PostgreSQL (B-tree) at 400k writes/sec:**

```
 Per row:
   heap tuple                     ~145 B (120 data + 24 header + line ptr)
   PK index entry (device_id, ts) ~40 B
   WAL for both                   ~250 B
   full-page images amortised     ~150 B
   ────────────────────────────────────
   ~585 B of disk write per 120 B row  →  WA ≈ 4.9×

 At 400,000/s:  400,000 × 585 B = 234 MB/s sustained write
                plus checkpoint I/O spikes writing dirty pages
                plus autovacuum reading and rewriting

 THE HARD PART is not bandwidth — it's the B-TREE INSERT RATE.
 The PK is (device_id, recorded_at). device_id is effectively random
 across 200,000 devices, so every insert lands in a random leaf page.
 The index is ~40 GB. Random 8 KB read-modify-write at 400k/s
 = 400,000 IOPS against an index that does not fit in RAM.

 ⇒ NOT ACHIEVABLE on a single node. Not close.
```

**LSM (Cassandra/ScyllaDB) at 400k writes/sec:**

```
 Per row:
   commit log append              ~130 B  (sequential)
   memtable insert                 0 disk
   eventual SSTable + compaction  ~120 B × WA(8) = 960 B  (all SEQUENTIAL)

 At 400,000/s:  ~440 MB/s of disk write — but ALL SEQUENTIAL.
 Sequential write bandwidth on NVMe: 2–7 GB/s. Comfortable.
 No random page lookup on the write path AT ALL.

 ⇒ 400k/s across a 6-node cluster is routine for Cassandra/Scylla.
```

**But now check the reads.** Query (a) — "last 24 h for device X" — is a range scan within one partition. In Cassandra with `PRIMARY KEY ((device_id), recorded_at)`, this is one partition, physically contiguous, already sorted. Excellent. Query (b) — "aggregate across all devices" — has no partition key. In Cassandra this is a full cluster scan. Terrible.

**The actual answer — and this is the point of the case:**

```
 ┌─────────────────────────────────────────────────────────────┐
 │ NEITHER pure choice is right. The workload has two shapes.  │
 └─────────────────────────────────────────────────────────────┘

 OPTION A — PostgreSQL + TimescaleDB (still a B-tree, but restructured)
   • Partition by time (hypertable chunks): each chunk's index is small
     and the ACTIVE chunk fits in RAM. Random device_id inserts now land
     in a 2 GB index, not a 40 GB one. The B-tree problem largely vanishes.
   • BRIN on recorded_at, compression on closed chunks (10–20×).
   • Retention = DROP CHUNK.
   • Realistic ceiling: ~100–200k rows/s on good hardware.
   ⇒ If 400k/s is a peak and 80k/s is the average, this WORKS and you
     keep SQL, joins, and transactions.

 OPTION B — Kafka → ClickHouse (LSM-shaped MergeTree, columnar)
   • Batched inserts (100k rows per batch) — LSM engines want batches,
     never single rows.
   • Handles both query shapes: partition-key range scans AND
     cross-device aggregates (it's columnar).
   • Retention = DROP PARTITION.
   ⇒ If 400k/s is genuinely sustained, this is the answer.

 OPTION C — Cassandra
   ⇒ Only if query (b) doesn't exist, or is served by a separate
     pre-aggregation pipeline. Cassandra is superb at (a) and hopeless
     at (b), and you WILL be asked for (b) eventually.
```

**The engine-level reasoning that produced this:** the write rate ruled out an unpartitioned B-tree (random index inserts at 400k/s), *not* PostgreSQL itself. Partitioning fixes the B-tree problem by shrinking the active index. And the read shape ruled out Cassandra, because bloom filters and partition keys give you nothing for a cross-partition aggregate.

**That is how you use this topic at work: it tells you which constraint is actually binding.**

---

## Common mistakes

**1. "LSM is faster than B-tree."**
- *Reality:* faster at **writes**, specifically at *random-key* writes. B-trees match or beat LSM on point reads, range scans, and sequential-key inserts.
- *Engine-level why:* the LSM write path avoids the random read-modify-write. That's the entire advantage, and it disappears if your keys are sequential.
- *Fix:* state which operation you mean. Benchmark your actual key distribution.

**2. Single-row inserts into an LSM/columnar engine.**
- *Symptom:* ClickHouse falls over at 2,000 inserts/sec; "too many parts" errors.
- *Engine-level why:* each insert creates a new part/SSTable. Thousands of tiny parts means catastrophic read amplification and a compaction backlog that never drains.
- *Diagnose:* `SELECT count() FROM system.parts WHERE active` — if it's in the thousands, you're inserting wrong.
- *Fix:* batch. 10k–100k rows per insert, via Kafka, a buffer table, or async inserts.

**3. Running an LSM store at >70% disk usage.**
- *Symptom:* compaction stalls, writes back up, cluster becomes unwritable.
- *Engine-level why:* size-tiered compaction must read N files and write their merged output *before* deleting the inputs. A major compaction can transiently need as much free space as the data itself.
- *Fix:* keep 50% headroom on size-tiered; ~20% on levelled. This is a capacity-planning rule, not a nice-to-have.

**4. Deleting a lot of data in an LSM and expecting space back.**
- *Symptom:* deleted 40% of rows; disk usage went **up**.
- *Engine-level why:* every delete writes a tombstone. Nothing is reclaimed until compaction merges the tombstone with the data — which for size-tiered may be a long time, and in Cassandra not before `gc_grace_seconds`.
- *Fix:* design for `DROP` (partition/table/time-window), not `DELETE`. Use TTLs where supported. This is the same rule as PostgreSQL partitioning (Topic 59) — it's universal.

**5. UUIDv4 primary keys in a B-tree engine at scale.**
- *Symptom:* insert throughput degrades as the table grows; index is 2× the expected size.
- *Engine-level why:* random insertion → a random leaf page per insert → cache miss + read-modify-write + frequent splits leaving pages half full.
- *Diagnose:* compare `pg_relation_size(index)` to `n_live_tup × key_width`; check `pg_stat_user_indexes` and `pgstatindex(...)` `avg_leaf_density`.
- *Fix:* UUIDv7 or ULID — time-ordered, so inserts go to the rightmost page like a bigserial (Topic 22).

**6. Ignoring compaction as a capacity dimension.**
- *Symptom:* the cluster is sized for 200 MB/s of application writes and dies at 100 MB/s.
- *Engine-level why:* your *disk* sees `app_writes × WA`. At WA=10, 100 MB/s of application traffic is 1 GB/s of disk traffic, competing with reads.
- *Fix:* size disks for `write_rate × write_amplification`, and monitor compaction backlog as a first-class metric.

---

## Hands-on proof

**PROVE IT #1 — sequential vs random B-tree insert (Example 1, condensed).**
```sql
\timing on
CREATE TABLE s (id bigint PRIMARY KEY);
INSERT INTO s SELECT generate_series(1,2000000);            -- fast
CREATE TABLE r (id uuid PRIMARY KEY);
INSERT INTO r SELECT gen_random_uuid() FROM generate_series(1,2000000);  -- much slower
SELECT relname, pg_size_pretty(pg_relation_size(oid)) FROM pg_class
WHERE relname IN ('s_pkey','r_pkey');
```

**PROVE IT #2 — B-tree leaf density (the split damage).**
```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT 's' AS t, * FROM pgstatindex('s_pkey')
UNION ALL SELECT 'r', * FROM pgstatindex('r_pkey');
```
```
 t | version | tree_level | index_size | ... | avg_leaf_density | leaf_fragmentation
---+---------+------------+------------+-----+------------------+--------------------
 s |       4 |          2 |   44957696 | ... |            90.11 |               0.00
 r |       4 |          3 |   98254848 | ... |            66.42 |              48.31
```
`avg_leaf_density 66%` and `leaf_fragmentation 48%` — that's the physical cost of random keys, measured.

**PROVE IT #3 — write amplification via WAL.**
```sql
SELECT pg_current_wal_lsn() AS b \gset
INSERT INTO amp (a,b,c,d) SELECT i,i,i,md5(i::text) FROM generate_series(1,100000) i;
SELECT pg_size_pretty(pg_current_wal_lsn() - :'b'::pg_lsn);
-- drop the three secondary indexes, TRUNCATE, repeat. Compare.
```

**PROVE IT #4 — full-page images spike WAL right after a checkpoint.**
```sql
CHECKPOINT;
SELECT pg_current_wal_lsn() AS b \gset
UPDATE amp SET a = a + 1 WHERE id < 50000;
SELECT pg_size_pretty(pg_current_wal_lsn() - :'b'::pg_lsn) AS first_touch;   -- large

SELECT pg_current_wal_lsn() AS b2 \gset
UPDATE amp SET a = a + 1 WHERE id < 50000;
SELECT pg_size_pretty(pg_current_wal_lsn() - :'b2'::pg_lsn) AS second_touch; -- much smaller
```
```
 first_touch | second_touch
-------------+--------------
 402 MB      | 11 MB
```
The first update after a checkpoint writes whole 8 KB pages into WAL. This is a B-tree/update-in-place engine paying for torn-write protection — an LSM never needs this, because it never overwrites a page.

**PROVE IT #5 — an LSM engine, on your laptop.**
```bash
docker run -d --name ch -p 8123:8123 clickhouse/clickhouse-server
docker exec -it ch clickhouse-client
```
```sql
CREATE TABLE readings (device_id UInt64, recorded_at DateTime, value Float64)
ENGINE = MergeTree ORDER BY (device_id, recorded_at);

-- ONE ROW AT A TIME — the wrong way
INSERT INTO readings VALUES (1, now(), 1.0);   -- repeat 1000 times → 1000 parts
SELECT count() FROM system.parts WHERE table='readings' AND active;

-- BATCHED — the right way
INSERT INTO readings SELECT number % 200000, now() - number, rand()/1e9
FROM numbers(10000000);
SELECT count() FROM system.parts WHERE table='readings' AND active;  -- a handful
SELECT formatReadableSize(sum(bytes)) FROM system.parts WHERE table='readings' AND active;
```
Watch compaction happen:
```sql
SELECT event_time, event_type, rows, formatReadableSize(size_in_bytes)
FROM system.part_log WHERE table='readings' ORDER BY event_time DESC LIMIT 10;
```

---

## The design decision framework

```
CHOOSE A B-TREE ENGINE (PostgreSQL, MySQL, …) WHEN:
  ✓ Writes are under ~20–50k/sec on one node
  ✓ Reads are point lookups or range scans with predictable latency needs
  ✓ You need real transactions, foreign keys, joins, and ad-hoc queries
  ✓ Keys are sequential or you can make them sequential (UUIDv7/ULID)
  ✓ Read latency consistency matters more than peak write throughput
  → i.e. almost every OLTP application ever built

CHOOSE AN LSM ENGINE (Cassandra, RocksDB, ClickHouse, …) WHEN:
  ✓ Write rate genuinely exceeds a single B-tree node (>100k/sec sustained)
  ✓ Keys are random/high-cardinality and CANNOT be made sequential
  ✓ Data is append-mostly; updates and deletes are rare
  ✓ You can batch writes (never single-row inserts)
  ✓ You accept worse and less predictable read tail latency
  ✓ Space amplification matters (levelled LSM is the best-in-class here)

FIX THE B-TREE INSTEAD OF SWITCHING ENGINES WHEN:  ← usually this
  ✓ Time-ordered keys (UUIDv7/ULID) — removes random-insert pain entirely
  ✓ PARTITION by time — shrinks the active index so it fits in RAM.
    This single change often takes a "we need Cassandra" workload and
    makes PostgreSQL comfortable.
  ✓ Drop unused indexes — each one multiplies write amplification
  ✓ Batch inserts (COPY / multi-row INSERT) — amortises WAL and page locks
  ✓ Raise checkpoint_timeout — fewer full-page images
  → Measure again. Most "we've outgrown PostgreSQL" claims die here.

COMPACTION STRATEGY (if you are on an LSM):
  LEVELLED       read-heavy, space-constrained · WA 10–30× · SA ~1.1×
  SIZE-TIERED    write-heavy, disk is cheap    · WA 4–10×  · SA 2×+
  TIME-WINDOW    time-series with TTL          · compacts within buckets only
                 → almost always correct for time-series; avoids
                   re-compacting old immutable data forever

THE SIGNAL TO LOOK FOR:
  Compute your write amplification, then ask what dominates it.
      WA = (bytes written to disk) / (bytes of application data)
  In PostgreSQL: measure WAL bytes per row inserted (PROVE IT #3).
  • WA driven by INDEX COUNT      → drop indexes; you have too many
  • WA driven by FULL-PAGE IMAGES → raise checkpoint_timeout
  • WA driven by RANDOM KEY ORDER → switch to time-ordered keys  ★ biggest win
  • Random index inserts at high rate into an index bigger than RAM
                                  → PARTITION, or you genuinely need an LSM
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Run PROVE IT #1 and #2. Report insert time, index size, `avg_leaf_density`, and `leaf_fragmentation` for both key types. Then explain, in terms of page splits and the buffer pool, why the random case is slower *and* larger. Predict what would happen with UUIDv7 and justify your prediction.

### Exercise 2 — medium (apply it)
Measure write amplification in PostgreSQL for a table with 0, 1, 3, and 6 secondary indexes (100k inserts each, using the `pg_current_wal_lsn()` technique). Plot WA against index count. Then run the same test immediately after `CHECKPOINT` versus 10 minutes after, and explain the difference. Finally: state the rule you'd give a teammate about how many indexes a write-heavy table may have.

### Exercise 3 — hard (production simulation)
You run a fleet-tracking system: 800,000 vehicles, one GPS ping every 10 seconds = 80,000 writes/sec. Current design is PostgreSQL, `PRIMARY KEY (vehicle_id, recorded_at)`, one 3 TB table, no partitioning. Insert latency p99 has climbed from 4 ms to 900 ms over six months. Disk shows 45,000 IOPS, 95% random writes.

(a) Explain precisely why this degraded *over time* rather than being slow from day one. Your answer must reference index size, RAM, and page splits.
(b) Compute the approximate write amplification and show your working.
(c) Design a fix that keeps PostgreSQL. Specify the partitioning scheme, the retention policy, what happens to the primary key, and what the active index size becomes. Estimate the new random-IOPS requirement.
(d) State the exact traffic level at which your PostgreSQL design stops working, and what you'd move to.
(e) A colleague proposes Cassandra with `PRIMARY KEY ((vehicle_id), recorded_at)`. Name the one query the business will definitely ask within a year that this cannot serve, and how you'd handle it.

---

## Mental model checkpoint

1. Draw the LSM write path from `PUT` to a compacted L2 SSTable. Which steps are synchronous with the client?
2. Define write, read, and space amplification. Give typical numbers for a B-tree, levelled LSM, and size-tiered LSM.
3. Why does a bloom filter help LSM point reads but not range scans?
4. Why does deleting data in Cassandra sometimes *increase* disk usage?
5. Why is a UUIDv4 primary key expensive in PostgreSQL but free in Cassandra? Be specific about the mechanism.
6. What is a full-page image in the WAL, why does it exist, and why does an LSM engine never need one?
7. You have a workload doing 150k random-key writes/sec. Give two ways to keep it on PostgreSQL, and state the one condition under which neither works.

---

## Quick reference card

| | B-tree | LSM (levelled) | LSM (size-tiered) |
|---|---|---|---|
| Write path | random read-modify-write | sequential append | sequential append |
| Write amp | 10–40× (index-dependent) | 10–30× | 4–10× |
| Read amp | ~1 | 1–2 (with blooms) | 2–5 |
| Space amp | 1.2–2× | ~1.1× | 2×+ |
| Point read | excellent, predictable | good, worse p99.9 | fair |
| Range scan | **excellent** | good | fair |
| Random-key inserts | **poor** | excellent | excellent |
| Deletes | immediate (logically) | tombstone + compaction | tombstone + compaction |
| Disk headroom | ~20% | ~20% | **~50%** |
| Examples | PostgreSQL, InnoDB, Oracle | RocksDB, LevelDB | Cassandra (default) |

**Key mechanisms**

| Term | Meaning |
|---|---|
| Memtable | In-RAM sorted write buffer |
| SSTable | Immutable sorted file on disk with index + bloom filter |
| Compaction | Background merge of SSTables; reclaims space, reduces read amp |
| Tombstone | Delete marker; data survives until compaction |
| Bloom filter | RAM bit-array: "definitely absent" or "maybe present" |
| Page split | B-tree leaf overflow → two half-full pages |
| Full-page image | Whole 8 KB page in WAL on first touch after checkpoint |

**The three rules**
1. Random keys are free in an LSM and expensive in a B-tree. Use time-ordered IDs.
2. LSM engines want **batches**. Single-row inserts are an antipattern.
3. Before switching engines, partition. It fixes the B-tree's actual problem (index bigger than RAM) far more often than people expect.

---

## When would I use this at work?

1. **The "we've outgrown PostgreSQL" meeting.** Someone wants to migrate to Cassandra because writes are slow. You measure write amplification, find it's 6 indexes plus UUIDv4 keys plus an unpartitioned 3 TB index, and demonstrate that fixing those three gives 8× headroom for two weeks of work instead of a six-month migration.

2. **Choosing a primary key.** In a design review you can explain, with `avg_leaf_density` numbers, why UUIDv7 over UUIDv4 is worth the small effort — and why the same argument would be irrelevant if the system were on DynamoDB.

3. **Capacity planning a Cassandra/ClickHouse cluster.** You size disk for `write_rate × write_amplification` and insist on 50% headroom for size-tiered compaction, instead of discovering it during an incident when compaction can't run and the cluster goes read-only.

---

## Connected topics

**Understand before this:** 04 (pages), 05 (tuples), 07 (buffer pool — why random I/O is the expensive kind).

**This unlocks:**
- **11** — B-trees in full detail: splits, fanout, the read path
- **17** — write amplification and when indexes hurt
- **22** — primary keys: the UUIDv4/v7/ULID decision rests entirely on this topic
- **41** — WAL and full-page images
- **59** — partitioning: the fix that makes B-trees survive high write rates
- **72, 73** — wide-column and time-series stores, which are LSM in practice
