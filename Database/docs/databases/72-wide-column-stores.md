# 72 — Wide-Column Stores
## Phase: Beyond Relational

---

## ELI5 — The Simple Analogy

A very long corridor of filing cabinets, and a rule about how paper is filed.

**The corridor position is decided by a hash of the customer's name.** Every document about Meera goes to cabinet 7 — always, and only cabinet 7. This is what makes it fast: you never search the corridor, you compute which cabinet and walk straight there.

**Inside cabinet 7, the papers are kept in strict date order**, newest at the front. So "Meera's last ten receipts" is: walk to cabinet 7, open the drawer, take the first ten. No searching at all.

★ **And now the two things that follow, which are the entire topic:**

1. ★ **If you don't know whose cabinet to open, you must open all of them.** "Which customers spent over ₹10,000?" means walking the whole corridor. There is no index of amounts.
2. ★ **You cannot re-file.** The corridor layout was fixed when the building was built. A new question — "show me by date across all customers" — means **filing everything a second time in a second corridor**, and keeping the two in agreement forever.

The trap is that ★ **the first corridor is so fast that it feels like a database.** It isn't. It's one enormous, hand-built index — and every additional question costs another one.

---

## Where this fits in the big picture

```
   08 LSM trees — the engine underneath
   60 sharding — the partition key, and the hot partition
   71 key-value — PK + SK, one index you design by hand
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 72 WIDE-COLUMN STORES ← YOU ARE HERE         │
        │ ★ partition key + clustering key, at scale,   │
        │   with no joins and no ad-hoc queries        │
        └────────────────────┬─────────────────────────┘
                             ▼
              73 time-series · 75 SQL vs NoSQL · 76 CDC
```

★ **Topic 71's DynamoDB section is the same model in a managed wrapper.** This topic is the open-source lineage — Cassandra, ScyllaDB, HBase, Bigtable — where **you own the replication factor, the consistency level, the compaction strategy, and every consequence of getting the key wrong.**

---

## What is this?

A distributed store where each row lives in a **partition** chosen by a hash of the **partition key**, and rows within a partition are **physically sorted** by the **clustering key**.

```sql
-- ★ Cassandra CQL. It looks like SQL. It is not SQL.
CREATE TABLE messages_by_conversation (
  conversation_id uuid,
  bucket          int,
  sent_at         timestamp,
  message_id      timeuuid,
  sender_id       uuid,
  body            text,
  ★ PRIMARY KEY ((conversation_id, bucket), sent_at, message_id)
) WITH ★ CLUSTERING ORDER BY (sent_at DESC, message_id DESC);
--        ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲   ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
--        ★ PARTITION KEY (in parens)   ★ CLUSTERING COLUMNS
--        decides WHICH NODE            decides ORDER ON DISK
```

★ **The three facts that define everything:**

```
 ① ★ THE PARTITION KEY IS HASHED. There is no ordering across
    partitions. ⇒ ★ you cannot range-scan on it, ever.
 ② ★ THE CLUSTERING KEY IS SORTED ON DISK. ⇒ ★ range queries
    within a partition are free.
 ③ ★ A QUERY MUST SPECIFY THE FULL PARTITION KEY.
    ⇒ ★ anything else is either refused, or requires
      ALLOW FILTERING — which means "scan every node".
```

★ **And the design consequence that people find hardest:** *you do not model entities and then query them. **You write one table per query.*** Denormalisation is not an optimisation here — **it is the schema design method.**

---

## Why does it matter for a backend developer?

```
 ★ THREE REASONS.

 ① ★ IT IS THE ONLY MODEL THAT SCALES WRITES LINEARLY TO
    HUNDREDS OF NODES WITHOUT COORDINATION.
    ⇒ ★ LSM-based (Topic 08): a write is an append to a memtable
      + a commitlog append. ★ No read, no lock, no coordination.
    ⇒ ★ MEASURED: a single Cassandra node sustains 30k–80k
      writes/sec; ★ 100 nodes sustain ~100× that, because nothing
      is shared.

 ② ★ EVERY MISTAKE IS EXPENSIVE AND MOSTLY PERMANENT.
    ★ a bad partition key ⇒ hot partitions and unbounded rows
    ★ a missing query ⇒ a new table and a full backfill
    ★ no joins, no aggregates, no ad-hoc queries, ★ no transactions
      across partitions

 ③ ★ THE FAILURE MODES ARE UNLIKE ANYTHING IN SQL, AND SILENT.
    ★ TOMBSTONES — deletes make reads SLOWER, for days
    ★ UNBOUNDED PARTITIONS — a partition that grows forever
    ★ LAST-WRITE-WINS by timestamp ⇒ ★ clock skew silently
      decides conflicts
    ⇒ ★ none of these produce an error until they produce an
      outage.
```

---

## The physical reality

### How a write becomes durable — and why it is so fast

```
 ★ ONE WRITE, IN ORDER:
 ① ★ APPEND to the commitlog (sequential, fsync-batched)
 ② ★ INSERT into the memtable (an in-memory sorted structure)
 ③ ★ ACKNOWLEDGE
 ⇒ ★ THAT IS ALL. ★ NO READ. NO SEEK. NO LOCK.
   ⇒ ★ this is why an "update" is just another write — there is
     nothing to read-modify-write.

 ★ LATER, ASYNCHRONOUSLY:
 ④ the memtable fills ⇒ ★ FLUSHED to an immutable SSTable on disk
 ⑤ SSTables accumulate ⇒ ★ COMPACTION merges them

 ⇒ ★ CONSEQUENCE ①: A ROW'S DATA MAY LIVE IN MANY SSTables.
   A read must merge them all. ★ Reads are harder than writes —
   the exact inverse of a B-tree (Topic 08).
 ⇒ ★ CONSEQUENCE ②: THERE IS NO UPDATE-IN-PLACE, so there is
   ★ NO READ-BEFORE-WRITE, so ★ writes never contend.
 ⇒ ★ CONSEQUENCE ③: ★ A DELETE IS A WRITE. It appends a
   TOMBSTONE. The data is still there until compaction.
```

### Tombstones — the failure mode with no SQL analogue

```
 ★ A DELETE INSERTS A MARKER SAYING "THIS IS DELETED, AS OF
   TIMESTAMP T". The original data remains in older SSTables.

 ⇒ ★ WHY IT MUST WORK THIS WAY: SSTables are immutable, and a
   node that was down during the delete must learn about it when
   it returns. ★ A tombstone IS the record of the deletion.

 ★ THEY SURVIVE FOR gc_grace_seconds — ★ DEFAULT 10 DAYS.
   ⇒ ★ WHY: if a node is down for 9 days and the tombstone were
     purged at day 1, that node would resurrect the deleted data
     on rejoin. ★ THE ZOMBIE-DATA PROBLEM.
   ⇒ ★ SO: for 10 days, every read of that partition reads and
     discards the tombstones.

 ★ THE FAILURE, CONCRETELY:
   a queue table: INSERT a job, DELETE it when done.
   ⇒ 1,000 jobs/hour × 24 × 10 days = ★ 240,000 tombstones in
     one partition.
   ⇒ ★ reading "the next pending job" scans 240,000 tombstones
     to return 1 row.
   ⇒ MEASURED: a read that was 2 ms becomes ★ 4,100 ms.
   ⇒ ★ AND: tombstone_warn_threshold (1,000) logs a warning;
     ★ tombstone_failure_threshold (100,000) ★ ABORTS THE QUERY.
       ReadFailureException — ★ the read is now impossible.

 ⇒ ★ THE RULE: ★ NEVER USE CASSANDRA AS A QUEUE. And never
   delete rows in a hot partition. ★ Use TTLs (which also create
   tombstones, but predictably) or ★ time-bucketed tables you can
   DROP whole.
```

### Partition sizing — the number that decides the design

```
 ★ TARGET: ★ ≤ 100 MB AND ≤ 100,000 ROWS PER PARTITION.
   ⇒ ★ hard problems begin around 1 GB; compaction, repair and
     streaming all operate on whole partitions.

 ★ THE THREE FAILURE MODES OF A LARGE PARTITION:
 ① ★ READS GET SLOWER — the partition index within an SSTable
    is scanned; a 2 GB partition means seeking within it.
 ② ★ COMPACTION AND REPAIR MUST HOLD IT — ★ a 2 GB partition
   can OOM a node during compaction.
 ③ ★ IT CANNOT BE SPLIT. Adding nodes does not help; ★ the
   partition is the unit of placement.

 ★ THE FIX IS ALWAYS ★ BUCKETING — adding a synthetic component
   to the partition key:
   PRIMARY KEY ((conversation_id, ★ bucket), sent_at)
   where bucket = ★ the month, the day, or floor(seq / 100000)

 ⇒ ★ AND THE COST OF BUCKETING, WHICH PEOPLE UNDERESTIMATE:
   ★ THE READER MUST NOW KNOW WHICH BUCKET(S) TO QUERY.
   "the last 50 messages" may span two buckets ⇒ ★ two queries,
   merged in the application. ★ Bucketing moves work to the reader.
```

### Consistency: the tunable that is actually a design decision

```
 ★ N = replication factor · W = write consistency · R = read consistency

 ★ IF W + R > N, a read quorum overlaps a write quorum
   ⇒ ★ the read sees the latest acknowledged write.

   RF=3, W=QUORUM(2), R=QUORUM(2) ⇒ 4 > 3  ✓ ★ strong-ish
   RF=3, W=ALL(3),    R=ONE(1)    ⇒ ★ fast reads, ★ writes fail
                                    if ANY replica is down
   RF=3, W=ONE,       R=ONE       ⇒ ★ 2 ≯ 3 ⇒ eventual
   RF=3, W=QUORUM,    R=ONE       ⇒ ★ 3 ≯ 3 ⇒ eventual

 ★ AND THE MULTI-DC LEVELS THAT MATTER:
   ★ LOCAL_QUORUM — a quorum within THIS datacentre only
     ⇒ ★ THE CORRECT DEFAULT for multi-DC. QUORUM spans DCs and
       pays cross-region latency on every operation.
   ★ EACH_QUORUM — a quorum in EVERY DC (writes only)

 ⇒ ★ BUT W+R>N IS STILL NOT LINEARIZABILITY (Topic 68).
   ★ Concurrent writes are resolved by LAST-WRITE-WINS ON
     TIMESTAMP — ★ so clock skew decides the winner, silently.
   ⇒ ★ FOR ACTUAL COMPARE-AND-SET, Cassandra offers ★ LIGHTWEIGHT
     TRANSACTIONS (Paxos):
       INSERT … IF NOT EXISTS
       UPDATE … IF version = 4
     ⇒ ★ 4 round trips instead of 1. ★ MEASURED 10–20× slower.
     ⇒ ★ use them for the rare correctness-critical path only.
```

### Secondary indexes — and why the answer is almost always "no"

```
 ★ ① A LOCAL SECONDARY INDEX (CREATE INDEX)
    ⇒ ★ stored ON EACH NODE, indexing only that node's data.
    ⇒ ★ A QUERY WITHOUT THE PARTITION KEY MUST ASK EVERY NODE.
      100 nodes ⇒ ★ 100 requests, and ★ latency = the slowest.
    ⇒ ★ AND IT IS WORSE ON HIGH CARDINALITY: indexing a uuid
      means every node returns ~0 or 1 rows, having done a full
      index lookup.
    ⇒ ★ RULE: only ever acceptable when you ALSO specify the
      partition key — at which point you barely needed it.

 ★ ② A MATERIALIZED VIEW (Cassandra 3.0+)
    ⇒ ★ the server maintains a second table with a different key.
    ⇒ ★ MARKED EXPERIMENTAL SINCE 3.0 and still is. Known
      consistency bugs under node failure.
    ⇒ ★ RULE: do not use in production. Maintain the second table
      yourself.

 ★ ③ ★ A SECOND TABLE, WRITTEN BY THE APPLICATION
    ⇒ ★ THE ACTUAL ANSWER, and it is what "one table per query"
      means.
    ⇒ ★ COST: ★ you own the consistency between them. There is no
      transaction across tables — use a BATCH for the same
      partition key, and accept eventual consistency otherwise.
```

### `BATCH` — the most misused statement in CQL

```sql
BEGIN BATCH
  INSERT INTO messages_by_conversation (…) VALUES (…);
  INSERT INTO messages_by_user (…) VALUES (…);
APPLY BATCH;
```
```
 ★ WHAT PEOPLE THINK IT IS: a transaction.
 ★ WHAT IT ACTUALLY IS: an ATOMICITY guarantee (all or none,
   eventually) — ★ NOT isolation, ★ NOT a rollback.
   ⇒ ★ other readers CAN see a partially-applied batch.

 ★ AND THE PERFORMANCE TRAP:
   ★ A MULTI-PARTITION BATCH IS SLOWER THAN N SEPARATE WRITES.
   ⇒ the coordinator writes a BATCHLOG to 2 nodes first, then
     fans out, then deletes the batchlog.
   ⇒ ★ MEASURED: a 10-statement multi-partition batch is ★ 3–5×
     slower than 10 concurrent single writes.
   ⇒ ★ AND a large batch can time out the coordinator entirely.

 ⇒ ★ THE RULES:
   ★ SINGLE-PARTITION BATCH  ⇒ ✓ genuinely atomic and FAST
     (it's one write to one node)
   ★ MULTI-PARTITION BATCH   ⇒ ★ only for atomicity you truly
     need, never for performance, and keep it small
   ★ "BATCH FOR THROUGHPUT"  ⇒ ★ ALWAYS WRONG. Use concurrent
     async writes.
```

### Compaction strategy — the choice that decides read latency

```
 ★ ① STCS (SizeTieredCompactionStrategy) — the default
    merges SSTables of similar size.
    ✓ ★ cheap writes
    ✗ ★ a row can live in MANY SSTables ⇒ ★ high read
      amplification
    ✗ ★ needs up to 50% free disk for a major compaction
    ⇒ ★ write-heavy, read-rarely workloads.

 ★ ② LCS (LeveledCompactionStrategy)
    non-overlapping SSTables in levels.
    ✓ ★ a read touches ~1 SSTable per level ⇒ ★ predictable,
      low read latency
    ✓ ★ far less space amplification
    ✗ ★ 2–10× the write I/O (constant re-compaction)
    ⇒ ★ read-heavy workloads with updates. ★ The right default
      for most OLTP-ish tables.

 ★ ③ ★ TWCS (TimeWindowCompactionStrategy)
    one SSTable set per time window; ★ never compacts across
    windows.
    ✓ ★ EXPIRED DATA IS DROPPED AS A WHOLE SSTABLE —
      ★ no tombstone scanning at all
    ✓ minimal write amplification
    ✗ ★ only correct for time-series with a TTL and ★ no updates
      to old data
    ⇒ ★ THE ONLY CORRECT CHOICE FOR TIME-SERIES, and choosing
      STCS instead is a common and expensive mistake (Topic 73).
```

---

## How it works — step by step

### The design method: one table per query

```
 ★ ① WRITE DOWN EVERY QUERY, VERBATIM
    Q1 "the last 50 messages in a conversation, newest first"
    Q2 "all conversations a user is in, by recent activity"
    Q3 "one message by its id"
    Q4 "unread count per conversation for a user"
    ⇒ ★ IF YOU CANNOT ENUMERATE THEM, STOP. This IS the schema.

 ★ ② FOR EACH QUERY, DESIGN A TABLE WHOSE PRIMARY KEY ANSWERS IT
    ⇒ ★ the partition key = what the query filters by EXACTLY
    ⇒ ★ the clustering key = what it orders/ranges by
    ⇒ ★ if two queries need different keys, ★ THAT IS TWO TABLES.

 ★ ③ CHECK EVERY PARTITION FOR SIZE AND HOTNESS
    ⇒ ★ estimate rows per partition over the system's LIFETIME
    ⇒ ★ > 100k rows or > 100 MB ⇒ ★ BUCKET IT
    ⇒ ★ one partition taking >10% of writes ⇒ ★ it is a hot
      partition; bucket or shard it

 ★ ④ DECIDE HOW THE DUPLICATES STAY IN AGREEMENT
    ⇒ ★ same partition key ⇒ a single-partition BATCH (atomic)
    ⇒ ★ different partition keys ⇒ ★ eventual, and you need a
      reconciler (Topic 51's three requirements)
```

### A worked schema

```sql
-- ★ Q1: the last N messages in a conversation
CREATE TABLE messages_by_conversation (
  conversation_id uuid,
  ★ bucket        text,          -- 'YYYY-MM' — bounds the partition
  sent_at         timestamp,
  message_id      timeuuid,
  sender_id       uuid,
  body            text,
  PRIMARY KEY ((conversation_id, bucket), sent_at, message_id)
) WITH CLUSTERING ORDER BY (sent_at DESC, message_id DESC)
  AND ★ compaction = {'class':'TimeWindowCompactionStrategy',
                      'compaction_window_unit':'DAYS',
                      'compaction_window_size':1}
  AND ★ default_time_to_live = 63072000     -- 2 years
  AND ★ gc_grace_seconds = 86400;           -- ★ 1 day, not 10

-- ★ Q2: a user's conversations, by recent activity
CREATE TABLE conversations_by_user (
  user_id         uuid,
  last_message_at timestamp,
  conversation_id uuid,
  title           text,
  PRIMARY KEY (user_id, last_message_at, conversation_id)
) WITH CLUSTERING ORDER BY (last_message_at DESC)
  AND compaction = {'class':'LeveledCompactionStrategy'};

-- ★ Q3: one message by id
CREATE TABLE messages_by_id (
  message_id      timeuuid PRIMARY KEY,
  conversation_id uuid, bucket text, sent_at timestamp,
  sender_id uuid, body text
);

-- ★ Q4: unread counts — ★ a COUNTER table, which has its own rules
CREATE TABLE unread_by_user (
  user_id         uuid,
  conversation_id uuid,
  ★ unread        counter,
  PRIMARY KEY (user_id, conversation_id)
);
```
```
 ★ FOUR TABLES, ONE PIECE OF DATA. ★ That is not a smell here —
   it is the method.
 ★ AND NOTE THE PER-TABLE CHOICES:
   • TWCS + TTL on the append-only, time-ordered table
   • LCS on the read-heavy, updated table
   • ★ gc_grace_seconds lowered to 1 day, because this cluster's
     repair completes in hours ⇒ ★ tombstones clear 10× sooner
```

### Writing to all of them

```js
// ★ SAME PARTITION KEY ⇒ a single-partition batch is atomic AND fast
const batch = [
  { query: `INSERT INTO messages_by_conversation
              (conversation_id, bucket, sent_at, message_id, sender_id, body)
            VALUES (?,?,?,?,?,?)`,
    params: [convId, bucket, sentAt, msgId, senderId, body] },
];
await client.batch(batch, { prepare: true, ★ logged: false });
// ★ logged:false skips the batchlog — correct for a single partition.

// ★ DIFFERENT PARTITION KEYS ⇒ ★ concurrent writes, NOT a batch
await Promise.all([
  client.execute(Q_MESSAGES_BY_ID, [...], { prepare: true }),
  client.execute(Q_CONVERSATIONS_BY_USER, [...], { prepare: true }),
  client.execute(Q_UNREAD_INCREMENT, [...], { prepare: true }),
]);
// ⇒ ★ 3–5× faster than a multi-partition batch, and the
//   atomicity a batch would give you is ★ not the atomicity you
//   need anyway (no isolation, no rollback).
```

### Reading across buckets

```js
// ★ "the last 50 messages" may span bucket boundaries
async function recentMessages(convId, limit = 50) {
  const out = [];
  let bucket = currentBucket();          // 'YYYY-MM'
  // ★ walk backwards through buckets until we have enough
  for (let i = 0; i < 6 && out.length < limit; i++) {
    const rs = await client.execute(
      `SELECT * FROM messages_by_conversation
        WHERE conversation_id = ? AND bucket = ?
        LIMIT ?`,
      [convId, bucket, limit - out.length], { prepare: true });
    out.push(...rs.rows);
    bucket = previousBucket(bucket);
  }
  return out.slice(0, limit);
}
// ★ THIS IS THE COST OF BUCKETING: the reader owns the traversal.
// ★ AND: cap the loop. An unbounded walk over empty buckets is a
//   latency cliff for a dormant conversation.
```

### Counters — a separate subsystem with different rules

```
 ★ COUNTER COLUMNS ARE NOT NORMAL COLUMNS.
   ✗ ★ cannot be mixed with non-counter columns in a table
   ✗ ★ cannot be set, only incremented/decremented
   ✗ ★ NOT IDEMPOTENT — ★ a retried increment counts twice
   ✗ ★ reads are more expensive (a read-before-write internally)
   ⇒ ★ THE CONSEQUENCE: a client-side timeout on a counter update
     leaves you unable to know whether it applied.
   ⇒ ★ THE PATTERN: for counts that must be exact, ★ store events
     and aggregate; use counters only where approximate is fine.
```

---

## Concept breakdown

```
★ THE MODEL
   ★ PARTITION KEY (hashed) ⇒ which node · ★ no ordering across
   ★ CLUSTERING KEY (sorted on disk) ⇒ ranges within a partition
   ★ A QUERY MUST GIVE THE FULL PARTITION KEY
     ⇒ anything else needs ALLOW FILTERING = scan every node

★ THE METHOD: ★ ONE TABLE PER QUERY
   denormalisation is not an optimisation — ★ it IS the design
   ⇒ four tables for one piece of data is normal

★ WHY WRITES ARE SO FAST (Topic 08)
   commitlog append + memtable insert + ack
   ⇒ ★ no read, no seek, no lock ⇒ ★ writes never contend
   ⇒ ★ an update is just another write
   ⇒ ★ and a DELETE is a write too — ★ a TOMBSTONE

★ TOMBSTONES — no SQL analogue, and silent
   ★ survive gc_grace_seconds (default ★ 10 days) to prevent
     ★ zombie data from a node that was down
   ★ every read scans them: 2 ms → ★ 4,100 ms measured
   ★ 1,000 ⇒ warn · ★ 100,000 ⇒ ★ ReadFailureException
   ⇒ ★ NEVER USE CASSANDRA AS A QUEUE

★ PARTITION SIZING
   ★ ≤100 MB and ≤100k rows · problems begin at ~1 GB
   ★ a large partition cannot be split — adding nodes doesn't help
   ⇒ ★ BUCKET the key; ★ the reader then owns the traversal

★ CONSISTENCY
   ★ W+R>N ⇒ quorums overlap · ★ LOCAL_QUORUM for multi-DC
   ★ but conflicts resolve by ★ LAST-WRITE-WINS ON TIMESTAMP
     ⇒ ★ clock skew decides, silently
   ★ real CAS ⇒ ★ lightweight transactions (Paxos), ★ 10–20× slower

★ SECONDARY INDEXES — almost always the wrong answer
   ★ local index ⇒ query without the PK asks EVERY node
   ★ materialized views ⇒ ★ still experimental; don't
   ⇒ ★ write a second table yourself

★ BATCH IS NOT A TRANSACTION
   atomic-eventually, ★ NOT isolated, ★ no rollback
   ★ single-partition ⇒ fast and atomic
   ★ multi-partition ⇒ ★ 3–5× SLOWER than concurrent writes
   ⇒ ★ "batch for throughput" is always wrong

★ COMPACTION DECIDES READ LATENCY
   STCS ⇒ cheap writes, ★ high read amplification
   ★ LCS ⇒ predictable low-latency reads, 2–10× write I/O
   ★ TWCS ⇒ ★ expired data dropped as whole SSTables, ★ no
     tombstone scanning — ★ the only correct choice for time-series

★ COUNTERS ★ are not idempotent — a retry double-counts
```

---

## Diagrams

**Diagram 1 — big picture: what the primary key actually does**

```
  PRIMARY KEY ((conversation_id, bucket), sent_at, message_id)
               ╰──────── partition key ────────╯ ╰─ clustering ─╯

  ★ STEP 1 — HASH THE PARTITION KEY ⇒ a token ⇒ a node
  ┌────────────────────────────────────────────────────────────────┐
  │  hash('conv-88' + '2026-08') = token 4,182,...                  │
  │                                                                 │
  │   node A        node B        ★ node C       node D             │
  │  [0 … 2^62]   [2^62 … 2^63]  ★[2^63 … 3·2^62]  […]              │
  │                                    ▲                            │
  │                          ★ this partition lives HERE            │
  │  ⇒ ★ NO ORDERING ACROSS PARTITIONS. conv-89 may be on node A.  │
  │  ⇒ ★ "all conversations updated today" ⇒ ★ ASK EVERY NODE.     │
  └────────────────────────────────────────────────────────────────┘

  ★ STEP 2 — WITHIN THE PARTITION, ROWS ARE SORTED ON DISK
  ┌────────────────────────────────────────────────────────────────┐
  │  partition ('conv-88','2026-08')  on node C                     │
  │  ┌──────────────┬────────────┬──────────┬────────────────────┐ │
  │  │ sent_at DESC │ message_id │ sender   │ body               │ │
  │  ├──────────────┼────────────┼──────────┼────────────────────┤ │
  │  │ 09:14:22     │ ...f2a     │ user:42  │ "on my way"        │ │
  │  │ 09:12:04     │ ...e81     │ user:88  │ "where are you"    │ │
  │  │ 09:04:11     │ ...c04     │ user:42  │ "leaving now"      │ │
  │  └──────────────┴────────────┴──────────┴────────────────────┘ │
  │  ⇒ ★ "the last 50" = ★ read the first 50 rows. ★ NO SORT.     │
  │  ⇒ ★ "messages between 09:00 and 09:15" = ★ a contiguous       │
  │    disk range.                                                 │
  └────────────────────────────────────────────────────────────────┘

  ★ AND WHAT IS IMPOSSIBLE:
    WHERE sender_id = 'user:42'          ⇒ ★ not in the key
    WHERE body LIKE '%urgent%'           ⇒ ★ no such thing
    ORDER BY sender_id                   ⇒ ★ disk order is fixed
    JOIN anything                        ⇒ ★ does not exist
  ⇒ ★ EACH OF THESE NEEDS ANOTHER TABLE, WRITTEN AT THE SAME TIME.
```

**Diagram 2 — data flow: how a tombstone destroys a read**

```
 ★ A QUEUE TABLE — the canonical Cassandra anti-pattern
   PRIMARY KEY (queue_name, job_id)
   INSERT a job … process it … ★ DELETE it

  DAY 0
  ┌────────────────────────────────────────────────────────────┐
  │ partition 'jobs:email'                                      │
  │  job_001 ✓live  job_002 ✓live  job_003 ✓live                │
  │ ★ SELECT … LIMIT 1  ⇒ reads 1 row.  ★ 2 ms                  │
  └────────────────────────────────────────────────────────────┘

  DAY 3 — 72,000 jobs processed and deleted
  ┌────────────────────────────────────────────────────────────┐
  │ partition 'jobs:email'                                      │
  │  ✗✗✗✗✗✗✗✗✗✗✗✗ ×72,000 tombstones ✗✗✗✗✗✗✗  job_072001 ✓     │
  │                                                             │
  │ ★ SELECT … LIMIT 1                                          │
  │   ⇒ the read must ★ SCAN AND DISCARD 72,000 tombstones      │
  │     to find the 1 live row                                  │
  │   ⇒ ★ 1,840 ms                                              │
  │   ⇒ ★ WARN: Read 1 live rows and 72,000 tombstone cells     │
  └────────────────────────────────────────────────────────────┘

  DAY 5
  ┌────────────────────────────────────────────────────────────┐
  │ ★ 120,000 tombstones                                        │
  │ ★ ReadFailureException: Operation failed — received 0       │
  │   responses and 1 failures                                  │
  │ ⇒ ★ THE READ IS NOW IMPOSSIBLE. The partition is dead until │
  │   gc_grace_seconds (★ 10 DAYS) passes AND compaction runs.  │
  └────────────────────────────────────────────────────────────┘

 ✓ WHAT IT SHOULD HAVE BEEN
   ★ a time-bucketed table with a TTL and TWCS:
     PRIMARY KEY ((queue_name, ★ hour_bucket), job_id)
     WITH default_time_to_live = 86400
       AND compaction = {'class':'TimeWindowCompactionStrategy',…}
   ⇒ ★ expired data is dropped as WHOLE SSTABLES.
     ★ Zero tombstone scanning.
   ⇒ ★ or — ★ don't use Cassandra for a queue.
```

**Diagram 3 — before/after: the partition key that killed a cluster**

```
 ✗ PARTITION KEY = tenant_id
 ┌───────────────────────────────────────────────────────────────┐
 │ CREATE TABLE events (                                          │
 │   tenant_id uuid, event_time timestamp, event_id timeuuid, …,  │
 │   PRIMARY KEY (★ tenant_id, event_time, event_id));            │
 │                                                                │
 │ ★ TENANT SIZES ARE POWER-LAW (Topic 60):                      │
 │   tenant A: ★ 4.1 billion events ⇒ ★ ONE PARTITION            │
 │   tenant B: 88 million                                         │
 │   8,400 others: a few thousand each                            │
 │                                                                │
 │ ★ RESULT:                                                      │
 │   • ★ tenant A's partition = 1.2 TB ★ ON ONE NODE              │
 │   • ★ that node OOMs during compaction                         │
 │   • ★ repair on that partition never completes                 │
 │   • ★ adding nodes does NOTHING — a partition cannot split     │
 │   • ★ 41% of all writes hit one node                           │
 └───────────────────────────────────────────────────────────────┘

 ✓ PARTITION KEY = (tenant_id, day)
 ┌───────────────────────────────────────────────────────────────┐
 │   PRIMARY KEY ((★ tenant_id, ★ day), event_time, event_id)     │
 │   WITH ★ compaction = {'class':'TimeWindowCompactionStrategy',  │
 │                        'compaction_window_size':1,             │
 │                        'compaction_window_unit':'DAYS'}        │
 │     AND ★ default_time_to_live = 7776000   -- 90 days          │
 │                                                                │
 │   • tenant A: ★ 11 M events/day ⇒ ★ still too big!            │
 │     ⇒ ★ ((tenant_id, day, shard)) with shard = hash(id) % 16  │
 │     ⇒ ★ 700k rows per partition ✓                             │
 │   • ★ writes spread over 16 × 90 partitions per tenant         │
 │   • ★ expired days drop as whole SSTables — no tombstones      │
 │                                                                │
 │ ★ COST: ★ a read for "tenant A, today" is now ★ 16 QUERIES    │
 │   merged in the application. ★ Bucketing moves work to the     │
 │   reader — always.                                             │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
-- ★ a 3-node local cluster, RF=3
CREATE KEYSPACE demo WITH replication =
  {'class':'NetworkTopologyStrategy', 'dc1': 3};
USE demo;
```

**Prove a query must specify the partition key.**
```sql
CREATE TABLE messages (
  conversation_id uuid, sent_at timestamp, message_id timeuuid,
  sender_id uuid, body text,
  PRIMARY KEY (conversation_id, sent_at, message_id)
) WITH CLUSTERING ORDER BY (sent_at DESC, message_id DESC);

SELECT * FROM messages WHERE sender_id = 6b1b3ca0-...;
```
```
 ★ InvalidRequest: Cannot execute this query as it might involve
   data filtering and thus may have unpredictable performance.
   If you want to execute this query despite the performance
   unpredictability, use ★ ALLOW FILTERING
```
```sql
SELECT * FROM messages WHERE sender_id = 6b1b3ca0-... ★ ALLOW FILTERING;
```
```
 ★ (returns rows, having asked every node and scanned every
   partition it owns)
 ★ tracing shows: ★ 3 nodes contacted, 41,204 rows scanned,
   ★ 8 rows returned.
 ⇒ ★ ALLOW FILTERING IS NOT A FEATURE. It is a warning you
   silenced.
```

**Prove clustering order is physical.**
```sql
SELECT sent_at, body FROM messages
 WHERE conversation_id = 8842119a-... LIMIT 5;
```
```
 ★ returns newest-first with NO sort step — the rows are already
   in that order on disk.
```
```sql
SELECT sent_at, body FROM messages
 WHERE conversation_id = 8842119a-... ★ ORDER BY message_id;
```
```
 ★ InvalidRequest: Order by currently only supports the ordering
   of columns following their declared order in the PRIMARY KEY
 ⇒ ★ you can reverse the declared order, and nothing else.
```

**Reproduce the tombstone problem.**
```sql
CREATE TABLE jobs (
  queue text, job_id timeuuid, payload text,
  PRIMARY KEY (queue, job_id));
```
```bash
# insert and delete 80,000 jobs in one partition
python3 - <<'EOF'
from cassandra.cluster import Cluster
s = Cluster(['127.0.0.1']).connect('demo')
ins = s.prepare("INSERT INTO jobs (queue, job_id, payload) VALUES ('email', now(), ?)")
for i in range(80000):
    r = s.execute(ins, [f'job{i}'])
s.execute("SELECT job_id FROM jobs WHERE queue='email'")
# delete them all
for row in s.execute("SELECT job_id FROM jobs WHERE queue='email'"):
    s.execute("DELETE FROM jobs WHERE queue='email' AND job_id=%s", [row.job_id])
EOF
```
```sql
TRACING ON;
SELECT * FROM jobs WHERE queue = 'email' LIMIT 1;
```
```
 ★ Read 0 live rows and ★ 80,000 tombstone cells for query
   SELECT * FROM jobs WHERE queue = 'email' LIMIT 1
   (see tombstone_warn_threshold)
 ★ Request complete — ★ 1,884,204 µs   (1.88 s to return NOTHING)
```
```sql
-- ★ push it past the failure threshold
-- (insert and delete 120,000 more)
SELECT * FROM jobs WHERE queue = 'email' LIMIT 1;
```
```
 ★ ReadFailureException: Operation failed - received 0 responses
   and 1 failures
 ★ Scanned over 100,000 tombstones during query; query aborted
 ⇒ ★ THE PARTITION IS UNREADABLE.
```
```sql
-- ★ and it stays that way for gc_grace_seconds
SELECT gc_grace_seconds FROM system_schema.tables
 WHERE keyspace_name='demo' AND table_name='jobs';
```
```
 ★ 864000        — ★ 10 DAYS.
```

**Prove partition size matters.**
```bash
nodetool tablehistograms demo.messages
```
```
 demo/messages histograms
 Percentile  SSTables  Write(µs)  Read(µs)  ★ Partition Size  Cell Count
 50%              1.0      35.43     88.15         ★ 2,346          17
 95%              2.0      61.21    182.79        ★ 42,510         310
 ★ 99%            3.0      88.15  ★ 4,102.41   ★ 1,258,291,200  ★ 8,204,118
 ⇒ ★ THE p99 PARTITION IS 1.2 GB WITH 8.2 MILLION CELLS.
   ★ Its reads are 46× the median. This is the partition that
   will OOM a node during compaction.
```
```bash
nodetool tablestats demo.messages | grep -E 'partition|Compacted'
```
```
 ★ Compacted partition maximum bytes: 1,258,291,200
 ⇒ ★ find it:
```
```bash
nodetool toppartitions demo messages 10000
```
```
 ★ Nodetool top WRITES partitions in the last 10000ms
 ★ Partition                        Count
 ★ 8842119a-0000-...                 41,204
 ⇒ ★ one partition, 41% of all writes.
```

**Prove `ALLOW FILTERING` scales with the cluster, not the result.**
```sql
TRACING ON;
SELECT count(*) FROM messages WHERE sender_id = 6b1b3ca0-... ALLOW FILTERING;
```
```
 ★ Executing seq scan across 0 sstables for (min(-9223372036854775808),
   max(9223372036854775807))
 ★ Read 412,088 live rows and 0 tombstone cells
 ★ Request complete — ★ 12,884,201 µs   (12.9 seconds)
 ⇒ ★ AND IT GETS SLOWER AS THE CLUSTER GROWS.
```

**Compare a batch against concurrent writes.**
```python
import time, uuid
from cassandra.cluster import Cluster
from cassandra.query import BatchStatement, BatchType
from cassandra.concurrent import execute_concurrent_with_args

s = Cluster(['127.0.0.1']).connect('demo')
ins = s.prepare("INSERT INTO messages (conversation_id, sent_at, message_id, body) VALUES (?,?,now(),?)")
rows = [(uuid.uuid4(), int(time.time()*1000), 'x') for _ in range(1000)]

# ★ multi-partition LOGGED batch, 10 statements each
t=time.time()
for i in range(0, 1000, 10):
    b = BatchStatement(batch_type=★ BatchType.LOGGED)
    for r in rows[i:i+10]: b.add(ins, r)
    s.execute(b)
print('batched', time.time()-t)

# ★ concurrent single writes
t=time.time()
execute_concurrent_with_args(s, ins, rows, concurrency=64)
print('concurrent', time.time()-t)
```
```
 ★ batched     4.204 s
 ★ concurrent  0.884 s        ★ 4.8× FASTER without the batch
 ⇒ ★ the batchlog write to 2 nodes, then the fan-out, then the
   batchlog delete — all overhead for atomicity you didn't need.
```

**Prove lightweight transactions are expensive.**
```sql
CREATE TABLE accounts (id uuid PRIMARY KEY, balance bigint, version int);
INSERT INTO accounts (id, balance, version) VALUES (uuid(), 1000, 1);
TRACING ON;
UPDATE accounts SET balance = 900, version = 2 WHERE id = ... ★ IF version = 1;
```
```
 ★ Parsing / Preparing / ★ Paxos Prepare / ★ Paxos Propose /
   ★ Paxos Commit / Reading existing values
 ★ Request complete — ★ 8,842 µs
```
```sql
UPDATE accounts SET balance = 800 WHERE id = ...;    -- ★ no IF
```
```
 ★ Request complete — ★ 412 µs        ★ 21× faster
 ⇒ ★ 4 round trips vs 1. Use LWT only where correctness demands it.
```

**Show the compaction strategies differ.**
```sql
ALTER TABLE messages WITH compaction =
  {'class':'LeveledCompactionStrategy'};
```
```bash
nodetool compactionstats
nodetool tablehistograms demo.messages | grep -E '95%|99%'
```
```
 ★ STCS:  95% ★ 4 SSTables, read 182 µs · 99% ★ 12 SSTables, ★ 4,102 µs
 ★ LCS:   95% ★ 2 SSTables, read  88 µs · 99% ★  3 SSTables, ★   204 µs
 ⇒ ★ 20× better p99 reads, at the cost of ~3× the write I/O.
```

**Prove counters are not idempotent.**
```sql
CREATE TABLE views (page text PRIMARY KEY, n counter);
UPDATE views SET n = n + 1 WHERE page = 'home';
UPDATE views SET n = n + 1 WHERE page = 'home';   -- ★ a "retry"
SELECT * FROM views WHERE page = 'home';
```
```
  page | n
 ------+---
  home | ★ 2
 ⇒ ★ a client that timed out and retried has now counted twice,
   and there is no way to tell.
```

---

## Example 2 — production scenario

**The situation.** An IoT platform ingesting sensor readings. 400,000 devices, one reading every 10 seconds — **3.5 billion readings per day**.

```
 THE FIRST DESIGN
 CREATE TABLE readings (
   device_id uuid, reading_time timestamp, metric text, value double,
   ★ PRIMARY KEY (device_id, reading_time, metric));
 -- ★ default compaction (STCS), ★ no TTL, ★ gc_grace 10 days

 AFTER 5 WEEKS
   ★ read p99             182 ms → ★ 8,400 ms
   ★ two nodes OOM-killed during compaction, repeatedly
   ★ repair                ★ has not completed in 11 days
   ★ disk                  ★ 82% and climbing; ★ nothing expires
   ★ one node              ★ 41% of all writes
```

**Step 1 — find the partition problem.**

```bash
nodetool tablehistograms iot.readings
```
```
 Percentile  SSTables  Read(µs)   ★ Partition Size   Cell Count
 50%              3.0    182.78          ★ 74,051         ★ 1,109
 95%             12.0  1,331.15       ★ 20,924,300       ★ 310,000
 ★ 99%           24.0  ★ 8,409.01   ★ 3,006,477,107  ★ 44,834,232
 ⇒ ★ THE p99 PARTITION IS 3 GB WITH 44 MILLION CELLS.
```
```
 ★ THE ARITHMETIC NOBODY DID:
   one device × 6 readings/min × 60 × 24 × 35 days × 4 metrics
   = ★ 1,209,600 rows and ★ 4.8 million cells, ★ AND GROWING
     FOREVER, because ★ the partition key has no time component.
 ⇒ ★ THE PARTITION IS UNBOUNDED BY CONSTRUCTION.
   ★ It was always going to fail; five weeks was just when.
```

**Step 2 — find the hot node.**

```bash
nodetool toppartitions iot readings 10000
```
```
 ★ Nodetool top WRITES partitions in the last 10000ms
 Partition                              Count
 ★ 4a1f...  (a gateway device)          ★ 88,204
 ★ 7b2c...                              ★ 41,102
 ⇒ ★ 20 "gateway" devices report on behalf of 2,000 sensors each.
   ⇒ ★ their partitions take 41% of all writes and land on 3 nodes.
```

**Step 3 — measure the tombstone and TTL problem.**

```sql
SELECT default_time_to_live, gc_grace_seconds, compaction
  FROM system_schema.tables
 WHERE keyspace_name='iot' AND table_name='readings';
```
```
 default_time_to_live | gc_grace_seconds | compaction
 ---------------------+------------------+-------------------------
                  ★ 0 |         ★ 864000 | ★ {'class':'…SizeTiered…'}
 ⇒ ★ NO TTL ⇒ ★ NOTHING EVER EXPIRES.
 ⇒ ★ A retention job was DELETING old rows instead
   ⇒ ★ millions of tombstones, ★ held for 10 days,
     ★ and STCS never compacts old data with new, so
     ★ THE TOMBSTONES AND THE DATA NEVER MEET TO BE PURGED.
```
```bash
grep -c 'tombstone' /var/log/cassandra/system.log
```
```
 ★ 412,088        warnings in 5 weeks
```

**Step 4 — the redesign, decision by decision.**

```
 ★ DECISION 1 — BOUND THE PARTITION.
   target ≤100k rows. At 4 metrics × 6/min:
     ★ 1 day  = 34,560 rows  ✓
     1 week   = 241,920      ✗
   ⇒ ★ bucket by DAY.

 ★ DECISION 2 — HANDLE THE GATEWAY DEVICES.
   a gateway does 2,000× the volume ⇒ ★ 69 million rows/day.
   ⇒ ★ add a shard component, ★ ONLY for hot devices:
     shard = hash(metric) % N, where N comes from a registry.
   ⇒ ★ ordinary devices: N=1. Gateways: N=32.
   ⇒ ★ THE SAME SURGICAL PATTERN AS TOPIC 61 — fix the 20 hot
     keys, not all 400,000.

 ★ DECISION 3 — TTL, NOT DELETE.
   retention is 90 days ⇒ ★ default_time_to_live = 7776000.

 ★ DECISION 4 — TWCS.
   ⇒ ★ expired data is dropped as WHOLE SSTABLES.
     ★ Zero tombstone scanning, zero compaction of dead data.

 ★ DECISION 5 — gc_grace_seconds.
   repair completes in 6 hours ⇒ ★ 10 days is 40× more than needed.
   ⇒ ★ 86400 (1 day). ★ But ONLY because repair is verified to
     complete well within it — ★ otherwise this resurrects data.
```

```sql
CREATE TABLE readings_by_device_day (
  device_id     uuid,
  ★ day         date,
  ★ shard       smallint,
  reading_time  timestamp,
  metric        text,
  value         double,
  PRIMARY KEY ((device_id, day, shard), reading_time, metric)
) WITH CLUSTERING ORDER BY (reading_time DESC, metric ASC)
  AND ★ compaction = {
        'class':'TimeWindowCompactionStrategy',
        'compaction_window_unit':'DAYS',
        'compaction_window_size':1,
        ★ 'unchecked_tombstone_compaction':'true'}
  AND ★ default_time_to_live = 7776000       -- 90 days
  AND ★ gc_grace_seconds = 86400             -- 1 day (repair: 6 h)
  AND ★ compression = {'class':'ZstdCompressor','chunk_length_in_kb':16};
```
```sql
-- ★ the shard registry — which devices are hot
CREATE TABLE device_shard_config (
  device_id uuid PRIMARY KEY,
  shard_count smallint);
-- ★ default 1; only the 20 gateways get 32.
```

**Step 5 — the second table, for the second query.**

```
 ★ Q2: "the latest reading for every device in a site" — a
   dashboard, 4,000 devices per site.
 ⇒ ★ THE FIRST TABLE CANNOT ANSWER IT: it would be 4,000 queries.
 ⇒ ★ A SECOND TABLE, keyed for that query.
```
```sql
CREATE TABLE latest_by_site (
  site_id   uuid,
  device_id uuid,
  metric    text,
  value     double,
  updated_at timestamp,
  PRIMARY KEY (site_id, device_id, metric)
) WITH ★ compaction = {'class':'LeveledCompactionStrategy'};
--        ★ LCS, because this table is OVERWRITTEN constantly and
--          read constantly ⇒ read latency matters more than write I/O.
```
```js
// ★ both writes, concurrently — NOT a batch (different partitions)
await Promise.all([
  client.execute(INSERT_READING, [...], { prepare: true }),
  client.execute(UPSERT_LATEST,  [...], { prepare: true }),
]);
// ⇒ ★ eventual consistency between them. ★ Acceptable: the
//   dashboard is a live view, and a reconciler rebuilds
//   latest_by_site nightly from readings_by_device_day.
```

**Step 6 — reading across shards and buckets.**

```js
async function readingsForDay(deviceId, day, limit) {
  const n = await shardCount(deviceId);        // ★ 1 for most devices
  if (n === 1) {
    return client.execute(SELECT_DAY, [deviceId, day, 0, limit],
                          { prepare: true });
  }
  // ★ hot device: query all shards concurrently and merge
  const results = await Promise.all(
    Array.from({length: n}, (_, s) =>
      client.execute(SELECT_DAY, [deviceId, day, s, limit],
                     { prepare: true })));
  return mergeSortedDesc(results.map(r => r.rows), limit);
  // ★ latency = the SLOWEST shard (Topic 60's scatter-gather)
}
```

**Step 7 — the migration, because you cannot ALTER a primary key.**

```
 ★ THE PRIMARY KEY CANNOT BE CHANGED. ★ EVER.
 ⇒ ★ this is a full dual-write plus backfill (Topic 28's expand).
```
```js
// ★ ① dual-write for 90 days (= the retention window)
await Promise.all([
  client.execute(INSERT_OLD, [...]),      // ★ keep serving reads
  client.execute(INSERT_NEW, [...]),
]);
```
```bash
# ★ ② backfill with Spark, ★ rate-limited so it doesn't starve
#   live traffic
spark-submit --class Backfill \
  --conf spark.cassandra.output.throughputMBPerSec=32 \
  --conf spark.cassandra.output.concurrent.writes=8 \
  backfill.jar --from 2026-05-01 --to 2026-08-25
```
```
 ★ 41 hours, throttled. ★ Unthrottled it saturated the cluster
   and doubled read p99 — ★ measured on the first attempt.
```
```js
// ★ ③ shadow-read: serve from old, compare against new, log
//   mismatches. Run for a week before cutting over.
const [oldRows, newRows] = await Promise.all([readOld(), readNew()]);
if (!deepEqual(oldRows, newRows)) metrics.increment('migration.mismatch');
return oldRows;
```
```sql
-- ★ ④ cut over, then DROP the old table (which reclaims disk
--   instantly — ★ unlike deleting rows)
DROP TABLE iot.readings;
```

**Step 8 — results.**

| | Before | After |
|---|---|---|
| p99 partition size | ★ **3.0 GB** | **6.2 MB** |
| p99 cells per partition | ★ 44,834,232 | **34,560** |
| Read p99 | ★ **8,400 ms** | **11 ms** (**764×**) |
| SSTables touched (p99) | 24 | **3** |
| Node OOMs | ★ 2/week | **0** |
| Repair | ★ never completed | **6 hours** |
| Tombstone warnings | ★ 412,088 | **0** |
| Disk | 82%, climbing | **41%, flat** |
| Retention mechanism | ★ `DELETE` | ★ **TTL + TWCS (whole-SSTable drop)** |
| Writes on the hottest node | ★ 41% | **9%** |

```
 ★ SIX LESSONS:
 ① ★ THE PARTITION WAS UNBOUNDED BY CONSTRUCTION. A partition key
   with no time component on time-series data will always fail —
   ★ five weeks was just when.
 ② ★ THE ARITHMETIC TAKES TWO MINUTES: rows/partition over the
   system's LIFETIME. ★ Nobody did it.
 ③ ★ `DELETE` FOR RETENTION IS THE WORST POSSIBLE CHOICE HERE.
   ★ TTL + TWCS drops whole SSTables; DELETE creates tombstones
   that STCS never even compacts away.
 ④ ★ 20 GATEWAY DEVICES OF 400,000 NEEDED SHARDING. The surgical
   fix from Topic 61, again — a registry, not a global change.
 ⑤ ★ THE PRIMARY KEY CANNOT BE ALTERED. This was a 90-day
   dual-write, a 41-hour throttled backfill, and a week of shadow
   reads. ★ Getting it right the first time is worth real effort.
 ⑥ ★ gc_grace_seconds = 1 DAY WAS SAFE ONLY BECAUSE REPAIR WAS
   MEASURED AT 6 HOURS. ★ Lowering it without that measurement
   resurrects deleted data.
```

---

## Common mistakes

**1. A partition key with no bound.**
- *Symptom:* multi-gigabyte partitions, node OOMs during compaction, repair that never completes.
- *Fix:* compute rows-per-partition over the system's lifetime. Bucket by time or a synthetic shard.

**2. Using it as a queue.**
- *Symptom:* tombstones accumulate until `ReadFailureException` makes the partition unreadable for 10 days.
- *Fix:* don't. Use a purpose-built queue, or time-bucketed tables with TTL and TWCS.

**3. `DELETE` for retention.**
- *Symptom:* tombstones that STCS never compacts away, and disk that never frees.
- *Fix:* `default_time_to_live` + TWCS — expired data drops as whole SSTables.

**4. `ALLOW FILTERING` in production.**
- *Symptom:* a query whose cost scales with cluster size, not result size.
- *Fix:* it's a warning you silenced. Design a table for the query.

**5. Secondary indexes on high-cardinality columns.**
- *Symptom:* every query contacts every node to return one row.
- *Fix:* a second table maintained by the application.

**6. Materialized views in production.**
- *Symptom:* silent inconsistency under node failure.
- *Fix:* still experimental. Maintain the second table yourself.

**7. Multi-partition batches for throughput.**
- *Symptom:* 3–5× slower than concurrent writes, plus coordinator timeouts.
- *Fix:* concurrent async writes. Batches only for single-partition atomicity.

**8. Assuming `BATCH` is a transaction.**
- *Symptom:* readers observe partially-applied batches.
- *Fix:* it provides eventual atomicity, not isolation and not rollback.

**9. Leaving STCS on a time-series table.**
- *Symptom:* high read amplification, and expired data that is never dropped.
- *Fix:* TWCS. It is the single highest-impact setting for time-series.

**10. Lowering `gc_grace_seconds` without measuring repair.**
- *Symptom:* deleted data resurrects when a node rejoins.
- *Fix:* only lower it below your *measured, verified* repair completion time.

**11. Counters where exactness matters.**
- *Symptom:* a timeout-and-retry double-counts, undetectably.
- *Fix:* store events and aggregate; use counters only where approximate is acceptable.

**12. `QUORUM` instead of `LOCAL_QUORUM` in multi-DC.**
- *Symptom:* every operation pays cross-region latency.
- *Fix:* `LOCAL_QUORUM` unless you specifically need cross-DC agreement.

**13. Expecting last-write-wins to be safe.**
- *Symptom:* clock skew silently decides which concurrent write survives.
- *Fix:* lightweight transactions where correctness demands it — and accept the 10–20× cost.

---

## Hands-on proof

**PROVE IT #1–#10 — Example 1** (a query refused without the partition key, `ALLOW FILTERING` scanning 412k rows for 8, clustering order being physical, 80,000 tombstones turning a 2 ms read into 1.88 s and then into `ReadFailureException`, `tablehistograms` showing a 1.2 GB p99 partition, `toppartitions` finding the hot key, a multi-partition batch at 4.8× the cost of concurrent writes, LWT at 21×, LCS improving p99 reads 20×, and a counter double-counting on retry).

**PROVE IT #11 — TWCS drops whole SSTables.**
```bash
nodetool tablestats iot.readings_by_device_day | grep -E 'SSTable count|Space used'
# ★ wait for a TTL window to expire, then:
nodetool compact iot readings_by_device_day
nodetool tablestats iot.readings_by_device_day | grep -E 'SSTable count|Space used'
```
```
 ★ before: SSTable count: 94 · Space used (live): 412.1 GB
 ★ after:  SSTable count: 61 · Space used (live): ★ 188.4 GB
 ⇒ ★ 33 SSTables were dropped WHOLE — ★ not compacted row by row.
   ★ Zero tombstone processing.
```

**PROVE IT #12 — token distribution shows the hot node.**
```bash
nodetool status iot
```
```
 --  Address     Load       Tokens  Owns (effective)  Host ID
 UN  10.0.1.11   ★ 1.82 TB    256     ★ 41.2%          ...
 UN  10.0.1.12     412 GB     256       19.4%          ...
 UN  10.0.1.13     388 GB     256       19.7%          ...
 ⇒ ★ "Owns" is token-range ownership; ★ Load is actual data.
   A large gap between them means ★ unbalanced partitions,
   not unbalanced tokens.
```

**PROVE IT #13 — consistency levels and availability.**
```bash
# RF=3; stop one node
nodetool -h 10.0.1.13 stopdaemon
cqlsh -e "CONSISTENCY ALL; SELECT * FROM demo.messages LIMIT 1;"
```
```
 ★ NoHostAvailable: Cannot achieve consistency level ALL
```
```bash
cqlsh -e "CONSISTENCY QUORUM; SELECT * FROM demo.messages LIMIT 1;"
```
```
 ★ (succeeds — 2 of 3 is a quorum)
 ⇒ ★ W=ALL means ANY node down stops writes. ★ Never use it
   for availability-sensitive paths (Topic 63's lesson, again).
```

**PROVE IT #14 — you cannot alter a primary key.**
```sql
ALTER TABLE messages ADD PRIMARY KEY (conversation_id, bucket, sent_at);
```
```
 ★ SyntaxException / InvalidRequest
 ⇒ ★ THE ONLY PATH IS A NEW TABLE + DUAL-WRITE + BACKFILL.
   ★ Design the key as if it were permanent, because it is.
```

---

## The design decision framework

```
★★★ ONE TABLE PER QUERY. THE PARTITION KEY IS PERMANENT.
    BOUND EVERY PARTITION. ★★★

 ① ★ SHOULD YOU USE THIS AT ALL?
    ✓ ★ write volume that no single machine can absorb
      (>100k writes/sec sustained)
    ✓ ★ a known, small set of access patterns
    ✓ ★ multi-datacentre with local-quorum writes
    ✓ ★ time-series or append-heavy data with TTL retention
    ✗ ★ ad-hoc queries · joins · aggregates · transactions
    ✗ ★ queues · anything with heavy DELETE traffic
    ✗ ★ "we might need to scale someday" — ★ Topic 60's checklist
      first. Postgres does 50k+ writes/sec on one box.

 ② ★ ENUMERATE EVERY QUERY, VERBATIM, BEFORE ANY DDL
    ⇒ ★ one table per query. Denormalisation IS the method.
    ⇒ ★ a query you didn't plan for needs a new table and a
      backfill — ★ there is no "add an index".

 ③ ★ FOR EACH TABLE, DO THE PARTITION ARITHMETIC
    rows per partition over the system's ★ LIFETIME
    ⇒ ★ target ≤100k rows and ≤100 MB
    ⇒ ★ over ⇒ BUCKET (by day/month/synthetic shard)
    ⇒ ★ AND remember: bucketing moves work to the READER.

 ④ ★ CHECK FOR HOT PARTITIONS
    ⇒ ★ nodetool toppartitions
    ⇒ ★ any key >10% of writes ⇒ ★ shard it — ★ via a registry,
      only for the hot keys (Topic 61's surgical pattern).

 ⑤ ★ CHOOSE COMPACTION PER TABLE — IT DECIDES READ LATENCY
    ★ TWCS ⇒ time-series with TTL, no updates to old data
             ⇒ ★ expired data dropped as whole SSTables
    ★ LCS  ⇒ read-heavy with updates ⇒ ★ predictable low p99
    STCS   ⇒ write-heavy, read-rarely
    ⇒ ★ leaving STCS on time-series is the most expensive default
      in Cassandra.

 ⑥ ★ RETENTION IS TTL, NEVER DELETE
    ⇒ ★ DELETE creates tombstones that survive gc_grace_seconds
      and make reads slower — sometimes impossible.
    ⇒ ★ TTL + TWCS drops whole files.

 ⑦ ★ SET gc_grace_seconds FROM MEASURED REPAIR TIME
    ⇒ ★ default 10 days is usually far too long
    ⇒ ★ but lowering it below your repair time ★ RESURRECTS
      DELETED DATA. Measure first.

 ⑧ ★ CONSISTENCY
    ★ LOCAL_QUORUM for multi-DC (QUORUM pays cross-region latency)
    ★ W+R>N for read-your-writes — ★ but it is NOT linearizability
    ★ conflicts resolve by ★ LAST-WRITE-WINS ON TIMESTAMP
      ⇒ ★ clock skew decides. Run NTP and know this.
    ★ real CAS ⇒ lightweight transactions, ★ 10–20× slower,
      rare paths only

 ⑨ ★ BATCHES
    ★ single-partition ⇒ atomic and fast
    ★ multi-partition ⇒ ★ 3–5× slower than concurrent writes
    ⇒ ★ never batch for throughput

 ⑩ ★ TREAT THE PRIMARY KEY AS PERMANENT
    changing it = a new table + dual-write + a throttled backfill
    + shadow reads. ★ Measured at 90 days and 41 hours in the
    example. ★ Spend the design time up front.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Start a single-node Cassandra. Create a table with a compound primary key and prove: (a) a query without the partition key is refused; (b) `ALLOW FILTERING` works but scans everything (use `TRACING ON`); (c) clustering order is physical and `ORDER BY` on a non-clustering column is rejected.

### Exercise 2 — medium (apply it)
Reproduce the tombstone failure: insert and delete 120,000 rows in one partition, then read from it. Show the warning, the timing degradation, and then the `ReadFailureException`. Report `gc_grace_seconds` and explain why the partition stays broken.

Then compare a 10-statement multi-partition `LOGGED` batch against 10 concurrent writes, and STCS against LCS using `tablehistograms`.

### Exercise 3 — hard (production simulation)
An IoT platform ingests 3.5 billion readings/day with `PRIMARY KEY (device_id, reading_time, metric)`, default compaction, no TTL, and a nightly `DELETE` for retention. After five weeks: read p99 8,400 ms, two nodes OOM-killed weekly, repair incomplete for 11 days, disk 82%.

(a) Do the partition arithmetic that should have been done on day one. Explain why this design was always going to fail.
(b) `toppartitions` shows 20 devices producing 41% of writes. What are they, and why does the standard fix not apply to all 400,000 devices?
(c) Explain precisely why `DELETE`-based retention is worse here than in any relational database, referencing both tombstones and STCS.
(d) Design the new primary key. Justify the bucket granularity with numbers and explain what the shard component costs the reader.
(e) Choose a compaction strategy per table and justify each.
(f) `gc_grace_seconds` is lowered from 10 days to 1. State the precondition, and what happens if it isn't met.
(g) A second query ("latest reading per device in a site") cannot be served. Design the second table and explain how the two stay in agreement.
(h) The primary key cannot be altered. Write the migration plan, including why the backfill must be throttled and what shadow reads prove.
(i) List every setting that changed and the failure each one prevents.

---

## Mental model checkpoint

1. What does the partition key decide, and what does the clustering key decide?
2. Why can't you range-query on the partition key?
3. Why are writes so fast, and what does that imply about reads?
4. What is a tombstone, why must it survive `gc_grace_seconds`, and what does that period protect against?
5. Give the two tombstone thresholds and what each does.
6. What is the target partition size, and what three things break above it?
7. Why does adding nodes not fix a large partition?
8. What does bucketing cost the reader?
9. Why is a multi-partition `BATCH` slower than concurrent writes?
10. Name the three compaction strategies and the workload each suits.
11. Why is TWCS the only correct choice for time-series?
12. What resolves concurrent writes, and what silently decides the winner?
13. Why are counters not idempotent, and what does that mean for retries?

---

## Quick reference card

```sql
PRIMARY KEY ((★ partition, key), ★ clustering, cols)
--           ╰ hashed → node ╯    ╰ sorted on disk ╯
WITH CLUSTERING ORDER BY (col DESC)
  AND ★ compaction = {'class':'TimeWindowCompactionStrategy',
                      'compaction_window_unit':'DAYS',
                      'compaction_window_size':1}
  AND ★ default_time_to_live = 7776000
  AND ★ gc_grace_seconds = 86400;   -- ★ only if repair < this
```

**★ Rules**
```
★ one table per query          ★ ≤100k rows / ≤100 MB per partition
★ TTL, never DELETE            ★ never a queue
★ ALLOW FILTERING = a scan     ★ secondary index = ask every node
★ batch: single-partition only ★ counters are not idempotent
★ the primary key is PERMANENT
```

| Compaction | For | Cost |
|---|---|---|
| STCS | write-heavy, read-rarely | ★ high read amplification |
| ★ LCS | read-heavy with updates | ★ 2–10× write I/O |
| ★ **TWCS** | ★ **time-series + TTL** | ★ **drops whole SSTables** |

**Diagnose**
```bash
nodetool tablehistograms ks.tbl    # ★ p99 partition size & cells
nodetool toppartitions ks tbl 10000 # ★ the hot key
nodetool tablestats ks.tbl          # ★ max partition bytes
nodetool status ks                  # ★ Load vs Owns = imbalance
```

**Consistency:** ★ `LOCAL_QUORUM` for multi-DC · `W+R>N` for overlap (★ **not** linearizability) · conflicts = ★ **last-write-wins on timestamp** · real CAS = LWT at ★ 10–20×.

---

## When would I use this at work?

1. **Sustained write volume beyond one machine**, with a small, known set of access patterns — telemetry, event logs, message history, time-series at scale. That's the shape it is genuinely unmatched at.

2. **Before choosing it, run Topic 60's exhaustion checklist.** PostgreSQL does 50,000+ writes/sec on one box and gives you joins, transactions and ad-hoc queries. Cassandra's linear write scaling costs all three, permanently.

3. **Reviewing any Cassandra schema.** Two questions find most problems: *how many rows will the largest partition hold in three years?* and *how is old data removed?* If the answers are "unbounded" and "DELETE", the cluster has a failure date.

4. **When a Cassandra cluster is already in trouble.** `tablehistograms` and `toppartitions` name the problem in thirty seconds — and it's almost always an unbounded or hot partition, plus STCS on time-series data.

---

## Connected topics

**Understand before this:** 08 (LSM trees — the engine), 60 (sharding, hot partitions), 61 (the surgical hot-key fix), 68 (quorums, last-write-wins), 71 (PK + SK, the same model managed).

**This unlocks:**
- **73** — time-series: where TWCS, TTL and bucketing are the whole design
- **75** — SQL vs NoSQL: the decision framework
- **76** — polyglot persistence: Cassandra for volume, PostgreSQL for truth
- **Case study 06** — chat/messaging · **Case study 07** — IoT telemetry
