# 46 — MVCC in Depth
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

A shared document where **nobody is ever allowed to erase anything**.

When Meera changes the price from ₹499 to ₹599, she does not cross out the old line. She writes a **new line** at the bottom, and adds two notes:
- on the old line: *"valid until Meera's edit"*
- on the new line: *"valid from Meera's edit"*

Now when you walk up to read, you carry a slip of paper that says **"I arrived at 10:04, and edits #88 and #91 were still in progress when I arrived."** You read every line, check its notes, and keep the one line that was valid *for you*.

Three things fall out of this immediately:

1. **You never wait.** The line you need is already written down.
2. **Meera never waits for you.** She just appends.
3. **The document gets enormous.** Every old line is still there. ⇒ Somebody has to come along and erase the lines that *nobody's slip of paper* could ever want. That somebody is `VACUUM` (Topic 47), and **it is not optional — it is the bill for never waiting.**

---

## Where this fits in the big picture

```
   05 tuple layout (t_xmin, t_xmax)  ·  43 anomalies  ·  44 isolation levels
                          │
          ┌───────────────┴────────────────┐
          ▼                                ▼
 ┌──────────────────────┐   ┌──────────────────────────────┐
 │ 45 locks             │   │ 46 MVCC ← YOU ARE HERE       │
 │ writer vs writer     │   │ how READS never block        │
 └──────────┬───────────┘   └──────────────┬───────────────┘
            └───────────────┬───────────────┘
                            ▼
        47 VACUUM & bloat (★ the bill)  ·  50 SSI  ·  62 replication
```

Topic 44 said "REPEATABLE READ takes one snapshot." **This topic is what a snapshot actually *is*, and how a tuple is judged against it** — the single mechanism underneath every isolation level.

---

## What is this?

**Multi-Version Concurrency Control**: an `UPDATE` does not modify a row. It writes a **new version** of the row and marks the old one as expired.

Every reader carries a **snapshot** — a precise definition of "which transactions had committed when I started." Every tuple carries two transaction IDs — **when it came into existence** and **when it stopped existing**. Comparing the two answers "can I see this version?" with no locks and no waiting.

| | Locking systems | MVCC |
|---|---|---|
| Reader meets an uncommitted write | **waits** | reads the previous version |
| Writer meets an ongoing read | **waits** | proceeds |
| Cost of a read | a shared lock | **zero** |
| Cost of the design | contention | ★ **garbage**: old versions must be reclaimed |

PostgreSQL, Oracle, MySQL/InnoDB, and SQL Server (with `READ_COMMITTED_SNAPSHOT`) all use MVCC. **They differ enormously in where the old versions go** — and that difference explains almost everything about their operational character.

---

## Why does it matter for a backend developer?

```
 ★ MVCC EXPLAINS FIVE THINGS THAT OTHERWISE LOOK UNRELATED:

 ① "Why is my table 60 GB when the data is 12 GB?"
    ⇒ dead tuples. MVCC's garbage. (Topic 47.)

 ② "Why did an UPDATE of one column rewrite the whole row?"
    ⇒ ★ PostgreSQL has no in-place update. Every UPDATE is a
      DELETE + INSERT at the storage level. Wide rows cost more
      to update than narrow ones, always.

 ③ "Why did DELETE not free any disk space?"
    ⇒ DELETE only sets t_xmax. The bytes stay until VACUUM.

 ④ "Why does a long-running SELECT on the replica break replication?"
    ⇒ the replica's snapshot needs tuples the primary vacuumed away.
      (hot_standby_feedback / max_standby_streaming_delay, Topic 62.)

 ⑤ "Why is SELECT count(*) slow?"
    ⇒ ★ visibility is per-tuple. There is no single place that
      knows the row count — every tuple must be checked against
      your snapshot. (Unless the visibility map lets it skip pages.)

 ⇒ ALL FIVE ARE THE SAME FACT: rows are versioned, versions
   accumulate, and visibility is decided per tuple, per reader.
```

---

## The physical reality

### The four header fields that implement everything

Every heap tuple carries these (Topic 05):

```
 ┌────────────────────────────────────────────────────────────────┐
 │ HeapTupleHeaderData — 23 bytes, then padded to 24              │
 ├──────────────┬─────────────────────────────────────────────────┤
 │ t_xmin   (4) │ the XID that INSERTED this version              │
 │              │ ⇒ "this version came into existence at t_xmin"  │
 │ t_xmax   (4) │ the XID that DELETED or UPDATED it (0 = alive)  │
 │              │ ⇒ "this version stopped existing at t_xmax"     │
 │ t_cid    (4) │ command ID within the transaction               │
 │              │ ⇒ ★ so statement #2 doesn't see statement #1's  │
 │              │   own uncommitted changes when it shouldn't     │
 │ t_ctid   (6) │ (block, offset) — points to THIS tuple, or to   │
 │              │ ★ the NEXT version if this one was updated      │
 │ t_infomask(2)│ HEAP_XMIN_COMMITTED, HEAP_XMAX_COMMITTED,       │
 │              │ HEAP_XMIN_FROZEN, HEAP_XMAX_LOCK_ONLY, …        │
 └──────────────┴─────────────────────────────────────────────────┘

 ★ 24 BYTES OF OVERHEAD PER ROW VERSION. On a table of narrow rows
   (say, 16 bytes of data), the MVCC header is 60% of the storage.
```

### What an UPDATE physically does

```
 BEFORE:  UPDATE products SET price_minor = 59900 WHERE id = 7;

  page 42
  ┌──────────────────────────────────────────────────────────┐
  │ lp[1] → tuple: xmin=100, xmax=0,   ctid=(42,1)           │
  │                id=7, name='Kurta', price_minor=49900     │
  └──────────────────────────────────────────────────────────┘

 AFTER:
  page 42
  ┌──────────────────────────────────────────────────────────┐
  │ lp[1] → tuple: xmin=100, ★xmax=205, ★ctid=(42,2)         │  ← DEAD
  │                id=7, name='Kurta', price_minor=49900     │    (once 205
  │                                                           │     commits)
  │ lp[2] → tuple: ★xmin=205, xmax=0,  ctid=(42,2)           │  ← LIVE
  │                id=7, name='Kurta', price_minor=59900     │
  └──────────────────────────────────────────────────────────┘

 ★ THREE FACTS THAT SURPRISE PEOPLE:
   ① THE ENTIRE ROW IS COPIED — every column, even unchanged ones.
      Updating one boolean on a row with a 2 KB text column copies
      2 KB. ⇒ narrow, frequently-updated tables; wide, rarely-
      updated ones. Split them if you must (Topic 27).
   ② THE OLD TUPLE'S ctid NOW POINTS FORWARD. This chain is how a
      concurrent writer finds the newest version (EvalPlanQual,
      Topic 44).
   ③ ★ THE TABLE GREW. An UPDATE is an INSERT as far as disk space
      is concerned. Only VACUUM shrinks it back.
```

### The snapshot — four fields

```
 A SNAPSHOT IS NOT A COPY OF THE DATA. It is four numbers:

   xmin      the lowest XID still running when the snapshot was taken
             ⇒ everything BELOW this is definitely finished
   xmax      the first XID not yet assigned
             ⇒ everything AT OR ABOVE this had not started
   xip[]     the list of XIDs that were IN PROGRESS in between
   (+ subxip[] for subtransactions)

 ⇒ ★ SO A SNAPSHOT IS TINY — a few dozen bytes — and taking one is
   nearly free. This is why READ COMMITTED can afford a NEW snapshot
   for every statement (Topic 44).

 SEE IT:
   SELECT pg_current_snapshot();
   -- 8842100:8842119:8842105,8842111
   --   xmin:xmax:xip[]
   ⇒ "everything below 8842100 is done; 8842119+ hasn't started;
      8842105 and 8842111 were running."
```

### The visibility rule — the entire algorithm

```
 ★ CAN I SEE THIS TUPLE?  (HeapTupleSatisfiesMVCC, simplified)

 ① IS t_xmin VISIBLE TO ME?
      t_xmin == my own XID?              → yes, if t_cid < my command id
      t_xmin >= snapshot.xmax?           → ✗ NO (started after me)
      t_xmin IN snapshot.xip[]?          → ✗ NO (was in flight)
      t_xmin aborted?  (check pg_xact)   → ✗ NO
      otherwise (committed before me)    → ✓ yes
      ⇒ if NO: ★ INVISIBLE. Stop.

 ② IS t_xmax VISIBLE TO ME?
      t_xmax == 0?                       → ✓ VISIBLE (never deleted)
      t_xmax is LOCK_ONLY?               → ✓ VISIBLE (a lock, not a delete)
      t_xmax >= snapshot.xmax?           → ✓ VISIBLE (deleted after me)
      t_xmax IN snapshot.xip[]?          → ✓ VISIBLE (deleter in flight)
      t_xmax aborted?                    → ✓ VISIBLE (the delete failed)
      otherwise (deleter committed
                 before me)              → ✗ INVISIBLE

 ⇒ ★ RUN FOR EVERY TUPLE THE SCAN TOUCHES. This is why dead tuples
   cost you even though you never see them — you still pay to
   evaluate and reject each one.

 ⇒ ★ THE HINT-BIT OPTIMISATION: step ① and ② need pg_xact lookups,
   which are expensive. So the FIRST reader to check a tuple writes
   the answer back into t_infomask (HEAP_XMIN_COMMITTED etc.).
   ⇒ CONSEQUENCE: ★ A PLAIN SELECT CAN DIRTY PAGES. The first scan
     after a bulk load is slower AND produces write I/O. This is
     the real answer to "why was the second run of my query so much
     faster even with a warm cache?"
```

### HOT updates — the optimisation that avoids most of the cost

```
 ★ HEAP-ONLY TUPLE: an update where
   ① NO INDEXED COLUMN CHANGED, and
   ② the new version fits ON THE SAME PAGE

 ⇒ then PostgreSQL can skip updating the indexes entirely.
   The index entry still points at lp[1]; lp[1]'s ctid points to
   lp[2]; the scan follows the chain within the page.

  ┌──────────── page 42 ────────────────────────────────────┐
  │ lp[1] ──ctid──► lp[2] ──ctid──► lp[3]  (the HOT chain)  │
  │  dead            dead            LIVE                   │
  └─────────────────────────────────────────────────────────┘
       ▲
   the index entry still points here, and always will

 ⇒ WHY IT MATTERS ENORMOUSLY:
   • a non-HOT update on a table with 6 indexes writes 7 places
   • a HOT update writes 1
   • ★ HOT chain pruning can reclaim space WITHOUT VACUUM — any
     page access can prune a dead HOT chain

 ⇒ HOW TO GET MORE HOT UPDATES:
   ① ★ don't index columns you update frequently
      (an index on `updated_at` or `last_seen_at` kills HOT for
       every single update)
   ② lower fillfactor so there's room on the page:
        ALTER TABLE sessions SET (fillfactor = 80);
      ⇒ 20% of each page reserved for future versions

 MEASURE IT:
   SELECT relname, n_tup_upd, n_tup_hot_upd,
          round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
     FROM pg_stat_user_tables ORDER BY n_tup_upd DESC;
   ⇒ ★ hot_pct below ~50% on a hot table is a design smell.
```

### Where the old versions live — the fork in the road

```
 ★ THIS ONE CHOICE EXPLAINS MOST OF THE DIFFERENCE BETWEEN ENGINES.

 POSTGRESQL — old versions live IN THE TABLE
   ✓ rollback is INSTANT (just mark the XID aborted in pg_xact —
     Topic 42's "no undo phase")
   ✓ no undo-log contention
   ✗ ★ the table BLOATS. VACUUM is mandatory, forever.
   ✗ ★ every UPDATE must update EVERY index (unless HOT applies)
   ✗ ★ transaction ID wraparound is a real operational hazard

 ORACLE / MYSQL-INNODB — old versions live in an UNDO LOG
   ✓ the table stays compact; the row is updated in place
   ✓ indexes point at a stable row identifier ⇒ no index write per
     update
   ✗ ★ ROLLBACK IS EXPENSIVE — the undo must be applied
   ✗ ★ "ORA-01555: snapshot too old" / undo tablespace pressure —
     a long reader can exhaust the undo the engine kept for it
   ✗ reading an old version means walking the undo chain

 ⇒ ★ NEITHER IS BETTER. THEY TRADE THE SAME COST DIFFERENTLY:
   PostgreSQL pays at CLEANUP time (VACUUM, bloat, wraparound).
   InnoDB pays at ROLLBACK and LONG-READ time (undo growth,
   snapshot-too-old).
 ⇒ if you have operated both, this is why PostgreSQL "needs
   babysitting with autovacuum" and MySQL "randomly kills long
   analytical queries." Same design problem, different bill.
```

---

## How it works — step by step

### A complete read, traced

```sql
-- txn 205 is mid-UPDATE and uncommitted. You run:
SELECT price_minor FROM products WHERE id = 7;
```

```
 ① your statement takes a snapshot (READ COMMITTED ⇒ a fresh one)
      xmin=8842100  xmax=8842120  xip=[205]

 ② the planner picks an Index Scan on products_pkey

 ③ the index returns TID (42,1)

 ④ read page 42 into shared buffers

 ⑤ ★ VISIBILITY CHECK on lp[1]:
      t_xmin = 100  → committed long ago, below snapshot.xmin  ✓ visible
      t_xmax = 205  → ★ 205 IS IN xip[] (in flight)
                      ⇒ the delete has NOT committed for me
                      ⇒ ✓ THE TUPLE IS STILL VISIBLE
      ⇒ RETURN price_minor = 49900

 ⑥ ★ lp[2] (xmin=205) is never even considered — the index scan was
   satisfied. But had this been a Seq Scan, lp[2] would be checked:
      t_xmin = 205 → in xip[] → ✗ invisible. Correctly skipped.

 ⇒ ★ AT NO POINT DID ANYTHING WAIT. Txn 205 holds a row lock on
   lp[1] (t_xmax=205) and it did not matter, because you are a
   reader.
```

### Why `SELECT count(*)` is slow — and how the visibility map rescues it

```
 THE PROBLEM: there is no authoritative row count anywhere. Whether
 a row "exists" depends on WHO IS ASKING. ⇒ count(*) must visit
 every tuple and run the visibility check.

 THE MITIGATION — THE VISIBILITY MAP (Topic 04):
   a separate fork, 2 bits per heap page:
     ALL_VISIBLE  — every tuple on this page is visible to EVERY
                    current and future snapshot
     ALL_FROZEN   — every tuple is frozen (wraparound-safe)

 ⇒ if a page is ALL_VISIBLE, an INDEX-ONLY SCAN can answer from the
   index alone and never touch the heap.

   VACUUM ANALYZE products;   -- sets the visibility map
   EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM products;
```
```
 Aggregate  (cost=… rows=1)
   ->  Index Only Scan using products_pkey on products
         Heap Fetches: 0                    ★ the heap was not read
         Buffers: shared hit=274
 Execution Time: 18.402 ms
```
```
 vs. the same query when the VM is stale (after many updates):
         Heap Fetches: 1284000              ★ every row hit the heap
 Execution Time: 3,914.221 ms               ★ 213× slower
 ⇒ "Heap Fetches" in an Index Only Scan is a VACUUM signal.
```

### Transaction ID wraparound — the hazard MVCC creates

```
 XIDs are 32 BITS. ~4.2 billion. They WRAP AROUND.
 ⇒ comparison is MODULAR: "2 billion ahead" and "2 billion behind"
   are the same bit pattern.
 ⇒ if a tuple's t_xmin were left alone, after 2^31 transactions it
   would appear to be IN THE FUTURE ⇒ ★ the row would vanish.

 THE FIX: FREEZING. VACUUM rewrites old-enough tuples with
 HEAP_XMIN_FROZEN in t_infomask, meaning "visible to everyone,
 forever, regardless of XID comparison."

 THE ESCALATION LADDER:
   age > autovacuum_freeze_max_age (200M)
     ⇒ ★ an ANTI-WRAPAROUND autovacuum starts and CANNOT BE
       cancelled by normal means; it holds SHARE UPDATE EXCLUSIVE
       and it will run to completion
   age > 10M remaining
     ⇒ WARNING: database "shop" must be vacuumed within 10000000
       transactions
   age exhausted
     ⇒ ★ THE DATABASE REFUSES ALL WRITES. Single-user mode required.
       This is a multi-hour outage and it has happened to well-known
       companies.

 MONITOR IT — this belongs in every production dashboard:
   SELECT datname, age(datfrozenxid) AS xid_age,
          2^31 - age(datfrozenxid) AS remaining
     FROM pg_database ORDER BY xid_age DESC;
   ⇒ alert at 500 million. Page at 1 billion.

 ★ THE #1 CAUSE OF WRAPAROUND EMERGENCIES: something holding an
   old xmin so VACUUM can't freeze — a long transaction, an
   abandoned replication slot, or a stale prepared transaction.
   Check all three (Topic 47).
```

---

## Concept breakdown

```
THE CORE IDEA
└── an UPDATE writes a NEW VERSION; nothing is modified in place
     ⇒ readers never block writers, writers never block readers
     ⇒ ★ and old versions accumulate as garbage

THE FOUR TUPLE FIELDS
├── t_xmin      the XID that created this version
├── t_xmax      the XID that expired it (0 = alive; may be a lock)
├── t_cid       command id — self-visibility within a transaction
└── t_ctid      points forward to the next version (the update chain)
     ★ 24 bytes of MVCC overhead on EVERY row version

THE SNAPSHOT — four numbers, not a copy
├── xmin   everything below is finished
├── xmax   everything at/above hadn't started
└── xip[]  what was in flight in between
     ★ tiny and nearly free ⇒ READ COMMITTED can take one per statement

THE VISIBILITY RULE
└── xmin committed-for-me AND xmax not-committed-for-me ⇒ visible
     ★ evaluated PER TUPLE, PER READER
     ★ hint bits cache the answer ⇒ a plain SELECT can dirty pages

HOT UPDATES — the optimisation that decides your write cost
├── requires: no indexed column changed AND room on the same page
├── skips ALL index updates
├── can be pruned without VACUUM
└── ★ killed by indexing frequently-updated columns (updated_at!)
     ⇒ measure n_tup_hot_upd / n_tup_upd; below 50% is a smell
     ⇒ tune with fillfactor

THE FIVE CONSEQUENCES YOU WILL MEET
├── bloat — dead tuples occupy space until VACUUM      (Topic 47)
├── every UPDATE rewrites the WHOLE row (all columns)
├── DELETE frees nothing immediately
├── count(*) is O(rows) unless the visibility map is fresh
└── ★ XID WRAPAROUND — a real, total outage if ignored

WHERE OLD VERSIONS LIVE — the fork
├── PostgreSQL: in the table   ⇒ instant rollback, ★ bloat + VACUUM
└── InnoDB/Oracle: undo log    ⇒ compact table, ★ costly rollback,
                                  "snapshot too old"
     ★ same problem, different bill
```

---

## Diagrams

**Diagram 1 — big picture: one row, three versions, three readers**

```
                    HEAP PAGE 42 (products, id=7)
  ┌───────────────────────────────────────────────────────────────┐
  │ lp[1] xmin=100 xmax=205 ctid→(42,2)   price=49900   ← v1      │
  │ lp[2] xmin=205 xmax=311 ctid→(42,3)   price=59900   ← v2      │
  │ lp[3] xmin=311 xmax=0   ctid→(42,3)   price=54900   ← v3 LIVE │
  └───────────────────────────────────────────────────────────────┘
       ▲                    ▲                    ▲
       │                    │                    │
  ┌────┴──────────┐  ┌──────┴────────┐  ┌────────┴──────┐
  │ READER A      │  │ READER B      │  │ READER C      │
  │ snapshot      │  │ snapshot      │  │ snapshot      │
  │ xmax=205      │  │ xmax=311      │  │ xmax=400      │
  │               │  │               │  │               │
  │ v1: xmin 100 ✓│  │ v1: xmax 205  │  │ v1: xmax 205  │
  │     xmax 205  │  │     committed✗│  │     committed✗│
  │     not yet ✓ │  │ v2: xmin 205 ✓│  │ v2: xmax 311 ✗│
  │ ⇒ SEES 49900  │  │     xmax 311  │  │ v3: xmin 311 ✓│
  │               │  │     not yet ✓ │  │     xmax 0   ✓│
  │               │  │ ⇒ SEES 59900  │  │ ⇒ SEES 54900  │
  └───────────────┘  └───────────────┘  └───────────────┘

 ★ THREE READERS, THREE ANSWERS, ZERO LOCKS, ZERO WAITING.
 ★ AND: v1 and v2 cannot be reclaimed until READER A finishes.
   ⇒ THIS IS EXACTLY WHY LONG TRANSACTIONS CAUSE BLOAT.
```

**Diagram 2 — data flow: HOT vs non-HOT update**

```
 ✗ NON-HOT UPDATE — an indexed column changed, or the page is full
 ┌────────────────────────────────────────────────────────────────┐
 │  UPDATE products SET sku = 'K-902' WHERE id = 7;               │
 │                        ↑ sku IS INDEXED                        │
 │                                                                 │
 │   heap page 42:  lp[1] dead ──► lp[2] LIVE                     │
 │                                                                 │
 │   ★ AND EVERY INDEX MUST BE UPDATED:                           │
 │     products_pkey        + 1 entry → (42,2)                    │
 │     idx_products_sku     + 1 entry → (42,2)                    │
 │     idx_products_cat     + 1 entry → (42,2)                    │
 │     idx_products_brand   + 1 entry → (42,2)                    │
 │     idx_products_price   + 1 entry → (42,2)                    │
 │     idx_products_created + 1 entry → (42,2)                    │
 │   ⇒ ★ 7 PAGES DIRTIED, 7 WAL RECORDS, for one column change    │
 └────────────────────────────────────────────────────────────────┘

 ✓ HOT UPDATE — no indexed column changed, room on the page
 ┌────────────────────────────────────────────────────────────────┐
 │  UPDATE products SET view_count = view_count + 1 WHERE id = 7; │
 │                        ↑ NOT indexed                           │
 │                                                                 │
 │   heap page 42:  lp[1] dead ──ctid──► lp[2] LIVE               │
 │      ▲                                                          │
 │      └── the index entry STILL POINTS HERE and is UNTOUCHED    │
 │          the scan follows the ctid chain within the page       │
 │                                                                 │
 │   ⇒ ★ 1 PAGE DIRTIED, 1 WAL RECORD                             │
 │   ⇒ ★ and lp[1] can be pruned by ANY later page access,        │
 │     without waiting for VACUUM                                 │
 └────────────────────────────────────────────────────────────────┘
        ★ 7× the write amplification, decided entirely by
          "is the column you're updating indexed?"
```

**Diagram 3 — before/after: an index that destroyed HOT updates**

```
 ✗ BEFORE — someone added an index on last_seen_at
 ┌───────────────────────────────────────────────────────────────┐
 │ CREATE INDEX idx_sessions_last_seen ON sessions (last_seen_at);│
 │                                                                │
 │ every request:  UPDATE sessions SET last_seen_at = now() …    │
 │                 ↑ ★ THE INDEXED COLUMN IS THE ONE THAT CHANGES │
 │                                                                │
 │ MEASURED:                                                      │
 │   n_tup_upd        412,000,000                                 │
 │   n_tup_hot_upd      3,100,000    ★ hot_pct = 0.8%             │
 │   table size          14 GB → 96 GB in 9 days                  │
 │   index size          31 GB (bloated, always behind)           │
 │   WAL generated       88 GB/hour                               │
 │   autovacuum          running continuously, never catching up  │
 └───────────────────────────────────────────────────────────────┘

 ✓ AFTER — drop the index, lower fillfactor
 ┌───────────────────────────────────────────────────────────────┐
 │ DROP INDEX idx_sessions_last_seen;                            │
 │ ALTER TABLE sessions SET (fillfactor = 70);                   │
 │ VACUUM FULL sessions;   -- one-off, in a window               │
 │                                                                │
 │ MEASURED:                                                      │
 │   n_tup_hot_upd / n_tup_upd    ★ 97.4%                        │
 │   table size                   14 GB, stable                  │
 │   WAL generated                6 GB/hour     ★ 14.6× less     │
 │   autovacuum                   idle most of the time          │
 │   the query that "needed" the index: replaced by a partial    │
 │     index on (last_seen_at) WHERE status='active' — 400 MB    │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE TABLE products (
  id bigint PRIMARY KEY,
  name text NOT NULL,
  price_minor bigint NOT NULL,
  view_count bigint NOT NULL DEFAULT 0
);
INSERT INTO products VALUES (7, 'Cotton Kurta', 49900, 0);
```

**See the MVCC fields directly.**
```sql
SELECT ctid, xmin, xmax, id, price_minor FROM products;
```
```
 ctid  |  xmin   | xmax | id | price_minor
-------+---------+------+----+-------------
 (0,1) | 8842101 |    0 |  7 |       49900
```

**Watch an UPDATE create a second version.**
```sql
UPDATE products SET price_minor = 59900 WHERE id = 7;
SELECT lp, lp_off, t_xmin, t_xmax, t_ctid
  FROM heap_page_items(get_raw_page('products', 0));
```
```
 lp | lp_off  | t_xmin  | t_xmax  | t_ctid
----+---------+---------+---------+--------
  1 |    8152 | 8842101 | 8842102 | (0,2)     ★ old version, points forward
  2 |    8104 | 8842102 |       0 | (0,2)     ★ new version, live
```
```sql
SELECT pg_relation_size('products') AS bytes;
```
```
 bytes
-------
  8192        ★ one page still, but it now holds TWO versions
```

**Prove a reader is never blocked.**
```sql
-- session 1                          -- session 2
BEGIN;
UPDATE products SET price_minor=1
  WHERE id=7;
-- do NOT commit
                                      SELECT price_minor, xmin, xmax
                                        FROM products WHERE id=7;
```
```
 price_minor |  xmin   |  xmax
-------------+---------+---------
       59900 | 8842102 | 8842103
   ★ returned instantly. xmax is set (session 1's XID) but that
     XID is in session 2's xip[] ⇒ the delete hasn't happened
     for session 2.
```
```sql
-- session 1
ROLLBACK;
-- ★ instant. Nothing is undone — pg_xact simply records "aborted",
--   and the new version becomes invisible to everyone. (Topic 42.)
```

**See a snapshot.**
```sql
SELECT pg_current_snapshot();
```
```
       pg_current_snapshot
---------------------------------
 8842104:8842104:                ★ xmin:xmax:xip[] — nothing in flight
```
```sql
-- with session 1 holding an open transaction:
SELECT pg_current_snapshot();
```
```
 8842105:8842107:8842105,8842106    ★ two XIDs in flight
```

**Prove DELETE frees nothing.**
```sql
INSERT INTO products
  SELECT g, 'item '||g, g*100, 0 FROM generate_series(100, 200000) g;
SELECT pg_size_pretty(pg_relation_size('products')) AS size;
```
```
  size
--------
 11 MB
```
```sql
DELETE FROM products WHERE id > 100;
SELECT pg_size_pretty(pg_relation_size('products')) AS size;
```
```
  size
--------
 11 MB        ★ UNCHANGED. DELETE only set t_xmax.
```
```sql
VACUUM products;
SELECT pg_size_pretty(pg_relation_size('products')) AS size;
```
```
  size
--------
 11 MB        ★ STILL unchanged! VACUUM marks the space REUSABLE,
              it does not return it to the OS (unless the free space
              is at the very end of the file). Topic 47.
```
```sql
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname='products';
```
```
 n_live_tup | n_dead_tup
------------+------------
          1 |          0     ★ the dead tuples were reclaimed for REUSE
```

**Prove HOT updates depend entirely on whether the column is indexed.**
```sql
-- ① no index on view_count
UPDATE products SET view_count = view_count + 1 WHERE id = 7;
SELECT n_tup_upd, n_tup_hot_upd FROM pg_stat_user_tables WHERE relname='products';
```
```
 n_tup_upd | n_tup_hot_upd
-----------+---------------
         1 |             1     ★ HOT
```
```sql
-- ② now index it
CREATE INDEX idx_products_views ON products (view_count);
UPDATE products SET view_count = view_count + 1 WHERE id = 7;
SELECT n_tup_upd, n_tup_hot_upd FROM pg_stat_user_tables WHERE relname='products';
```
```
 n_tup_upd | n_tup_hot_upd
-----------+---------------
         2 |             1     ★ NOT HOT. One index turned it off.
```

**Prove a plain `SELECT` can dirty pages (hint bits).**
```sql
CHECKPOINT;
SELECT pg_stat_reset_shared('bgwriter');
INSERT INTO products SELECT g, 'x', g, 0 FROM generate_series(300000, 400000) g;
CHECKPOINT;

EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM products;
```
```
 Buffers: shared hit=32 read=610 dirtied=610      ★ A SELECT DIRTIED 610 PAGES
 Execution Time: 41.882 ms
```
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM products;
```
```
 Buffers: shared hit=642    dirtied=0             ★ hint bits are now set
 Execution Time: 12.104 ms                        ★ 3.5× faster
```

**Prove the visibility map makes `count(*)` fast.**
```sql
VACUUM ANALYZE products;
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM products;
```
```
 Aggregate
   ->  Index Only Scan using products_pkey on products
         Heap Fetches: 0            ★ zero heap access
         Buffers: shared hit=277
 Execution Time: 8.221 ms
```

**Check wraparound age.**
```sql
SELECT datname, age(datfrozenxid) AS xid_age,
       (2^31)::bigint - age(datfrozenxid) AS remaining
  FROM pg_database ORDER BY xid_age DESC LIMIT 3;
```
```
  datname  | xid_age  |  remaining
-----------+----------+-------------
 shop      | 48291042 |  2099192606
 postgres  |    18240 |  2147465408
   ★ alert at 500,000,000. Page at 1,000,000,000.
```

---

## Example 2 — production scenario

**The situation.** A session-tracking table on an e-commerce platform. 40 million active sessions, every API request updates `last_seen_at`. Over nine days:

```
 sessions table       14 GB → 96 GB
 sessions indexes     4 GB  → 31 GB
 WAL generated        6 GB/hr → 88 GB/hr     ★ 14.6×
 replica lag          2 s → 22 minutes
 p99 write latency    8 ms → 340 ms
 autovacuum           running on `sessions` continuously, 24/7
 disk                 88% full, projected exhausted in 4 days
```

**Step 1 — is it bloat, and how much?**

```sql
SELECT
  relname,
  n_live_tup, n_dead_tup,
  round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
  pg_size_pretty(pg_total_relation_size(relid)) AS total,
  last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 3;
```
```
 relname  | n_live_tup | n_dead_tup | dead_pct |  total  |    last_autovacuum
----------+------------+------------+----------+---------+------------------------
 sessions |   40118204 |  318440129 |  ★ 88.8 | 127 GB  | 2026-08-16 11:04:22+05:30
 orders   |   18402118 |     104882 |      0.6 |  22 GB  | 2026-08-16 03:12:08+05:30
```

```
 ★ 88.8% of the table is dead tuples, and autovacuum ran 3 minutes
   ago. So autovacuum is RUNNING — it just cannot keep up.
   ⇒ this is a PRODUCTION RATE problem, not a "vacuum isn't running"
     problem. The two have completely different fixes.
```

**Step 2 — the decisive metric: HOT update ratio.**

```sql
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0 * n_tup_hot_upd / nullif(n_tup_upd, 0), 2) AS hot_pct
  FROM pg_stat_user_tables
 WHERE n_tup_upd > 1000000
 ORDER BY n_tup_upd DESC;
```
```
 relname  | n_tup_upd  | n_tup_hot_upd | hot_pct
----------+------------+---------------+---------
 sessions |  412884019 |       3104882 | ★ 0.75
 orders   |   18220114 |      16884221 |   92.67
```

```
 ★ 0.75%. On the busiest table in the database.
   99.25% of updates are writing a new tuple AND updating every index.
```

**Step 3 — why? Which index is on a column that changes?**

```sql
SELECT
  i.relname AS index_name,
  pg_size_pretty(pg_relation_size(i.oid)) AS size,
  pg_get_indexdef(i.oid) AS definition,
  s.idx_scan
FROM pg_class t
JOIN pg_index ix ON ix.indrelid = t.oid
JOIN pg_class i ON i.oid = ix.indexrelid
JOIN pg_stat_user_indexes s ON s.indexrelid = i.oid
WHERE t.relname = 'sessions';
```
```
       index_name        |  size   |                  definition                      | idx_scan
-------------------------+---------+--------------------------------------------------+----------
 sessions_pkey           | 2.1 GB  | … USING btree (id)                               | 88402118
 idx_sessions_user       | 1.8 GB  | … USING btree (user_id)                          |  4028841
 idx_sessions_last_seen  | ★ 21 GB | … USING btree (last_seen_at)                     |   ★ 1204
 idx_sessions_token      | 6.2 GB  | … USING btree (token)                            | 41028804
```

```
 ★ THERE IT IS.
   idx_sessions_last_seen indexes THE EXACT COLUMN THAT EVERY
   REQUEST UPDATES.
   ⇒ every UPDATE must remove the old index entry and add a new one
     ⇒ HOT is impossible
     ⇒ the index itself bloats faster than anything
     ⇒ 21 GB of index, used ★ 1,204 times in nine days
       (vs. sessions_token's 41 million)
```

**Step 4 — what was the index for?**

```sql
SELECT calls, mean_exec_time::numeric(10,1) AS mean_ms,
       substring(query from 1 for 90) AS query
  FROM pg_stat_statements
 WHERE query ILIKE '%last_seen_at%' AND query ILIKE '%sessions%'
 ORDER BY calls DESC LIMIT 3;
```
```
 calls | mean_ms |                     query
-------+---------+------------------------------------------------------------------
  1204 |  2840.1 | DELETE FROM sessions WHERE last_seen_at < $1 AND status = 'active'
```

```
 ★ A CLEANUP JOB. Runs every 10 minutes. 1,204 calls in nine days.
   A 21 GB index, killing HOT on 412 million updates, exists to
   serve a job that runs 134 times a day.
```

**Step 5 — the fix, in three parts.**

```sql
-- ① replace the index with a partial one that matches the ONE query
DROP INDEX CONCURRENTLY idx_sessions_last_seen;

CREATE INDEX CONCURRENTLY idx_sessions_stale
  ON sessions (last_seen_at)
  WHERE status = 'active';
-- ⇒ ★ WAIT. This still indexes last_seen_at, so HOT is still dead
--   for active sessions. Think harder.
```

```
 ★ THE REAL QUESTION: does the cleanup job need an index at all?
   It runs every 10 minutes over a 40M-row table looking for rows
   older than 30 days. There are ~200,000 such rows at any time.
   ⇒ OPTION A: keep an index on last_seen_at ⇒ HOT stays dead
   ⇒ OPTION B: ★ PARTITION BY RANGE on last_seen_at, and DROP the
     old partition instead of DELETEing rows.
     ⇒ no index on last_seen_at needed at all
     ⇒ cleanup becomes DROP PARTITION: instant, no bloat, no WAL
     ⇒ ★ this is the standing rule from the case-study index:
       "retention is DROP PARTITION, not DELETE."  (Topic 59.)
```

```sql
-- ★ THE ACTUAL FIX
-- ① drop the offending index entirely
DROP INDEX CONCURRENTLY idx_sessions_last_seen;

-- ② give the page room for new versions so HOT can apply
ALTER TABLE sessions SET (fillfactor = 70);
-- ⇒ 30% of each page reserved. New versions land on the same page.

-- ③ make autovacuum aggressive on THIS table only
ALTER TABLE sessions SET (
  autovacuum_vacuum_scale_factor = 0.01,   -- default 0.2 ⇒ 8M dead tuples
  autovacuum_vacuum_cost_delay   = 0,      -- don't throttle
  autovacuum_vacuum_cost_limit   = 10000,
  autovacuum_naptime             = 10
);
-- ⇒ with 40M rows, the default 0.2 scale factor means autovacuum
--   waits for 8 MILLION dead tuples before starting. On this table
--   that is 20 seconds of traffic. (Topic 47.)

-- ④ one-off: reclaim the 96 GB
--    pg_repack rebuilds without an ACCESS EXCLUSIVE lock
--    (VACUUM FULL would lock the table for ~40 minutes)
--    $ pg_repack -d shop -t sessions --no-superuser-check
```

```sql
-- ⑤ and the retention job, done right (Topic 59)
--    convert sessions to a range-partitioned table on last_seen_at,
--    then:
DROP TABLE sessions_2026_07;    -- ★ instant, no dead tuples, ~0 WAL
```

**Step 6 — the second-order finding.**

```sql
-- while investigating, check what else was holding xmin back
SELECT pid, state, now() - xact_start AS age, application_name
  FROM pg_stat_activity WHERE xact_start IS NOT NULL ORDER BY xact_start LIMIT 3;

SELECT slot_name, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
  FROM pg_replication_slots;
```
```
   slot_name    | active |  restart_lsn  | retained
----------------+--------+---------------+-----------
 replica_mumbai | t      | 4A/8C001220   | 1.2 GB
 replica_old    | ★ f    | 12/AB0044F0   | ★ 812 GB
```
```
 ★ AN ABANDONED REPLICATION SLOT from a replica decommissioned in
   May. It pinned xmin AND retained 812 GB of WAL.
   ⇒ VACUUM could not remove ANY tuple newer than that slot's xmin,
     no matter how aggressively it ran.
   ⇒ this is the SECOND reason autovacuum wasn't keeping up, and it
     would have made every other fix ineffective.
```
```sql
SELECT pg_drop_replication_slot('replica_old');
ALTER SYSTEM SET max_slot_wal_keep_size = '64GB';   -- ★ the guard (Topic 41)
```

**Step 7 — results, 24 hours later.**

| Metric | Before | After | Change |
|---|---|---|---|
| HOT update ratio | **0.75%** | **97.4%** | **130×** |
| `sessions` total size | 127 GB | 16 GB | **8×** |
| Dead tuple % | 88.8% | 3.1% | — |
| WAL generated | 88 GB/hr | 6 GB/hr | **14.6×** |
| Replica lag | 22 min | 1.4 s | **940×** |
| p99 write latency | 340 ms | 7 ms | **48×** |
| Autovacuum on `sessions` | continuous | ~3 min/hour | — |
| Retained WAL | 812 GB | 1.2 GB | — |

```
 ★ THE ROOT CAUSE, STATED PRECISELY:
   ONE INDEX on ONE column that ONE cleanup job queried 1,204 times,
   which made 412 MILLION updates non-HOT, which multiplied write
   amplification by 7×, which outpaced autovacuum, which was ALSO
   blocked by an abandoned replication slot.
 ⇒ NONE OF THIS IS VISIBLE WITHOUT UNDERSTANDING MVCC. The symptom
   was "disk filling up." The dashboard said "autovacuum is running."
   The fix was to delete an index.
```

---

## Common mistakes

**1. Indexing a frequently-updated column.**
- *Symptom:* HOT ratio near zero; the table and every index bloat; WAL volume multiplies.
- *Engine-level why:* any change to an indexed column forces a non-HOT update, which writes a new entry in *every* index.
- *Fix:* check `n_tup_hot_upd / n_tup_upd`. Drop the index or restructure (partition for retention rather than indexing a timestamp you update).

**2. Expecting `DELETE` to free disk space.**
- *Symptom:* "I deleted 90% of the table and nothing changed."
- *Engine-level why:* `DELETE` sets `t_xmax`. `VACUUM` marks the space reusable but does not return it to the OS unless it's at the end of the file.
- *Fix:* `pg_repack` (online) or `VACUUM FULL` (ACCESS EXCLUSIVE) to actually shrink; better, partition and `DROP` (Topic 59).

**3. Long-running transactions.**
- *Symptom:* `n_dead_tup` in the hundreds of millions while autovacuum runs constantly.
- *Engine-level why:* VACUUM cannot remove a tuple newer than the oldest snapshot's xmin. One 40-minute transaction pins everything.
- *Fix:* `idle_in_transaction_session_timeout`; alert on `max(now() - xact_start)`.

**4. Abandoned replication slots.**
- *Symptom:* the same as #3 with no long transaction visible anywhere.
- *Engine-level why:* an inactive slot pins `xmin` *and* retains WAL indefinitely.
- *Fix:* audit `pg_replication_slots WHERE NOT active`; set `max_slot_wal_keep_size` (Topic 41).

**5. Leaving `autovacuum_vacuum_scale_factor` at the default on a huge table.**
- *Symptom:* autovacuum on a 40M-row table waits for 8M dead tuples.
- *Fix:* set it per table (`0.01` or lower) on high-churn tables.

**6. Ignoring XID age.**
- *Symptom:* an anti-wraparound autovacuum you can't cancel, then a database that refuses all writes.
- *Fix:* dashboard `age(datfrozenxid)`; alert at 500M.

**7. Assuming an `UPDATE` only writes the changed column.**
- *Symptom:* a table with a 4 KB `jsonb` column and a hot counter generates enormous WAL.
- *Engine-level why:* the whole row is copied, every time.
- *Fix:* split hot, narrow columns into their own table (Topic 27).

**8. Being surprised that a `SELECT` produced write I/O.**
- *Engine-level why:* hint bits. The first reader of a tuple caches the commit status into `t_infomask` and dirties the page.
- *Fix:* nothing to fix — but `VACUUM` after a bulk load pays for itself on the first scan.

**9. `SELECT count(*)` with a stale visibility map.**
- *Symptom:* an index-only scan showing millions of `Heap Fetches`.
- *Fix:* `VACUUM ANALYZE`. `Heap Fetches > 0` is a vacuum signal.

---

## Hands-on proof

**PROVE IT #1–#9 — all in Example 1** (MVCC fields, version chains, no reader blocking, snapshots, `DELETE` freeing nothing, HOT toggled by one index, hint bits dirtying pages, the visibility map, XID age).

**PROVE IT #10 — measure the write amplification of an index directly.**
```sql
CREATE TABLE counters (id bigint PRIMARY KEY, n bigint NOT NULL DEFAULT 0, pad text);
INSERT INTO counters SELECT g, 0, repeat('x', 200) FROM generate_series(1,100000) g;
VACUUM ANALYZE counters;

SELECT pg_current_wal_lsn() AS a \gset
UPDATE counters SET n = n + 1;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'a')) AS wal_no_index;

CREATE INDEX idx_counters_n ON counters (n);
VACUUM ANALYZE counters;
SELECT pg_current_wal_lsn() AS b \gset
UPDATE counters SET n = n + 1;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'b')) AS wal_with_index;
```
```
 wal_no_index
--------------
 28 MB
 wal_with_index
----------------
 ★ 214 MB        — 7.6× more WAL for the same UPDATE
```

**PROVE IT #11 — fillfactor's effect on HOT.**
```sql
CREATE TABLE ff100 (id bigint PRIMARY KEY, n bigint, pad text) WITH (fillfactor=100);
CREATE TABLE ff70  (id bigint PRIMARY KEY, n bigint, pad text) WITH (fillfactor=70);
INSERT INTO ff100 SELECT g,0,repeat('x',300) FROM generate_series(1,50000) g;
INSERT INTO ff70  SELECT g,0,repeat('x',300) FROM generate_series(1,50000) g;
VACUUM ANALYZE ff100, ff70;

UPDATE ff100 SET n = n + 1;
UPDATE ff70  SET n = n + 1;
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/n_tup_upd,1) AS hot_pct,
       pg_size_pretty(pg_relation_size(relid)) AS size
  FROM pg_stat_user_tables WHERE relname IN ('ff100','ff70');
```
```
 relname | n_tup_upd | n_tup_hot_upd | hot_pct |  size
---------+-----------+---------------+---------+--------
 ff100   |     50000 |          1204 |  ★ 2.4 | 39 MB
 ff70    |     50000 |         48802 | ★ 97.6 | 27 MB
```

**PROVE IT #12 — a long transaction blocks reclamation.**
```sql
-- session 1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT 1 FROM counters LIMIT 1;

-- session 2
UPDATE counters SET n = n + 1;
VACUUM (VERBOSE) counters;
```
```
INFO:  table "counters": found 0 removable, 200000 nonremovable row versions
DETAIL:  100000 dead row versions cannot be removed yet, oldest xmin: 8842119
   ★ "cannot be removed yet" + the exact xmin holding it back
```

**PROVE IT #13 — find everything holding xmin back.**
```sql
SELECT 'long txn' AS source, pid::text AS id, now()-xact_start AS age
  FROM pg_stat_activity WHERE xact_start IS NOT NULL
UNION ALL
SELECT 'repl slot', slot_name, NULL FROM pg_replication_slots WHERE NOT active
UNION ALL
SELECT 'prepared txn', gid, now()-prepared FROM pg_prepared_xacts
ORDER BY 3 DESC NULLS LAST;
-- ★ the three causes of "VACUUM runs but reclaims nothing"
```

---

## The design decision framework

```
★★★ MVCC IS FREE READS PAID FOR WITH GARBAGE. DESIGN FOR THE GARBAGE. ★★★

 ① BEFORE ADDING AN INDEX, ASK: DOES THIS COLUMN GET UPDATED?
    ⇒ ★ THE SINGLE HIGHEST-LEVERAGE QUESTION IN THIS TOPIC.
      indexing an updated column turns every HOT update into a
      non-HOT one, multiplying write cost by (1 + index count).
    ⇒ if yes, ask what the index is FOR:
      • a retention/cleanup job?  ⇒ ★ PARTITION instead (Topic 59)
      • a rare admin query?       ⇒ let it seq-scan
      • a hot user-facing query?  ⇒ then you must pay; lower
                                     fillfactor to soften it

 ② DESIGN TABLES FOR THEIR UPDATE RATE
    a whole row is copied on EVERY update.
    ⇒ hot, narrow, frequently-updated columns → their own table
      (session_activity: user_id, last_seen_at)
    ⇒ wide, cold columns → the main table
    ⇒ ★ never put a counter next to a 4 KB jsonb column

 ③ SET FILLFACTOR ON UPDATE-HEAVY TABLES
    default 100 ⇒ no room for a new version on the page
                ⇒ HOT fails ⇒ index writes
    ⇒ 70–90 on tables where n_tup_upd is large
    ⇒ costs disk; buys HOT

 ④ TUNE AUTOVACUUM PER TABLE, NOT GLOBALLY
    the default scale_factor 0.2 means a 40M-row table waits for
    8M dead tuples.
    ⇒ ALTER TABLE hot_table SET (autovacuum_vacuum_scale_factor=0.01,
        autovacuum_vacuum_cost_delay=0);

 ⑤ RETENTION IS `DROP PARTITION`, NEVER `DELETE`
    DELETE creates one dead tuple per row, plus index churn, plus
    WAL, plus vacuum work, and frees no space.
    DROP PARTITION is O(1) and creates none of it.

 ⑥ THE THREE THINGS THAT MAKE VACUUM POWERLESS — audit all three
    ① a long-running transaction    → idle_in_transaction_session_timeout
    ② ★ an inactive replication slot → max_slot_wal_keep_size, and
                                       drop dead slots
    ③ a stale prepared transaction   → pg_prepared_xacts
    ⇒ if any is present, every other tuning change is wasted.

 ⑦ THE FOUR METRICS THAT BELONG ON YOUR DASHBOARD
    ✓ n_dead_tup / (n_live_tup + n_dead_tup)     per table
    ✓ ★ n_tup_hot_upd / n_tup_upd                per table
    ✓ ★ age(datfrozenxid)                        alert 500M
    ✓ max(now() - xact_start)                    alert 5 min

 ⑧ WHEN CHOOSING AN ENGINE, KNOW WHICH BILL YOU'RE SIGNING
    PostgreSQL : instant rollback, ★ mandatory VACUUM, wraparound
    InnoDB     : compact tables, ★ expensive rollback, undo pressure,
                 "snapshot too old" on long readers
    ⇒ neither is "better" — pick the failure mode your team can
      operate.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Using `pageinspect`: (a) show a row's `t_xmin`/`t_xmax`/`t_ctid` before and after an `UPDATE`; (b) prove `DELETE` doesn't shrink the file, and that `VACUUM` doesn't either; (c) show a reader returning the old value while an uncommitted `UPDATE` is in flight, and explain via `xip[]`. (d) Explain in one sentence why `ROLLBACK` is instant in PostgreSQL.

### Exercise 2 — medium (apply it)
Create a table with 100,000 rows and one non-indexed integer column. Measure WAL generated by `UPDATE t SET n = n+1` on all rows. Then add an index on `n` and repeat. Then set `fillfactor = 70`, `VACUUM FULL`, and repeat.

Report WAL bytes, HOT ratio, and table size for all three. Explain each number mechanically. Then state the rule you'd put in a code-review checklist.

### Exercise 3 — hard (production simulation)
A `sessions` table grew 14 GB → 96 GB in nine days. WAL went 6 → 88 GB/hour, replica lag 2 s → 22 minutes, p99 write latency 8 → 340 ms. `n_dead_tup` is 88.8% and autovacuum is running continuously.

(a) Write the four diagnostic queries, in the order you'd run them, and say what each rules in or out.
(b) The HOT ratio is 0.75%. Explain mechanically why that produces 14.6× the WAL.
(c) Identify the offending index. It is 21 GB and was scanned 1,204 times in nine days. What is it for?
(d) Explain why a *partial* index on the same column would not fix the problem, and what would.
(e) A second, independent cause is present. Name the three things that pin `xmin`, write the query that finds all three, and say which one it was.
(f) Give the complete fix: index change, `fillfactor`, per-table autovacuum settings, the one-off reclamation, and the retention redesign. Justify each.
(g) Why `pg_repack` rather than `VACUUM FULL`? What lock does each take?
(h) Write the four dashboard metrics and their alert thresholds that would have caught this on day one.
(i) The same table on MySQL/InnoDB would not have bloated this way. Explain what it would have done instead, and why that is not automatically better.

---

## Mental model checkpoint

1. What does an `UPDATE` physically do to a heap page? How many bytes of MVCC overhead does each version carry?
2. What are the four fields of a snapshot, and why is taking one cheap enough to do per statement?
3. State the visibility rule for a tuple in two clauses.
4. What is a HOT update? Name the two conditions, and the one design decision that most often destroys it.
5. Why can a plain `SELECT` dirty pages and generate write I/O?
6. Why doesn't `DELETE` free disk space? Why doesn't plain `VACUUM`?
7. Name the three things that prevent VACUUM from reclaiming dead tuples.
8. What is XID wraparound, what happens at the end of the escalation ladder, and what do you monitor?
9. PostgreSQL keeps old versions in the table; InnoDB keeps them in an undo log. Give one advantage and one failure mode of each.

---

## Quick reference card

**The fields**
```
t_xmin  created by   ·  t_xmax  expired by (0 = alive)
t_cid   command id   ·  t_ctid  → next version
★ 24 bytes per row version
```

**Visibility:** visible ⟺ `xmin` committed-for-my-snapshot **and** `xmax` not-committed-for-my-snapshot.

**Snapshot:** `xmin:xmax:xip[]` — `SELECT pg_current_snapshot();`

**Inspect**
```sql
SELECT ctid, xmin, xmax, * FROM products;
SELECT lp, t_xmin, t_xmax, t_ctid, t_infomask::bit(16)
  FROM heap_page_items(get_raw_page('products', 0));   -- pageinspect
```

**The four metrics**
```sql
-- bloat
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0*n_dead_tup/nullif(n_live_tup+n_dead_tup,0),1) AS dead_pct
  FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;

-- ★ HOT ratio — the highest-signal metric in this topic
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
  FROM pg_stat_user_tables WHERE n_tup_upd > 100000 ORDER BY hot_pct;

-- ★ wraparound — alert at 500M
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;

-- oldest snapshot — alert at 5 min
SELECT max(now()-xact_start) FROM pg_stat_activity WHERE xact_start IS NOT NULL;
```

**The three xmin holders**
```sql
SELECT pid, now()-xact_start FROM pg_stat_activity WHERE xact_start IS NOT NULL;
SELECT slot_name FROM pg_replication_slots WHERE NOT active;   -- ★ the silent one
SELECT gid, prepared FROM pg_prepared_xacts;
```

**Tune**
```sql
ALTER TABLE sessions SET (fillfactor = 70);
ALTER TABLE sessions SET (autovacuum_vacuum_scale_factor = 0.01,
                          autovacuum_vacuum_cost_delay = 0);
```

**The rules:** never index a column you update on a hot path · a whole row is copied on every update · retention is `DROP PARTITION` · `Heap Fetches > 0` means vacuum · monitor XID age.

---

## When would I use this at work?

1. **Any "the disk is filling up" or "replica lag is growing" incident.** Both are usually the same root cause — write amplification from non-HOT updates, or something pinning `xmin`. Four queries separate them.

2. **Every index review.** "Does this column get updated on a hot path?" is a question almost nobody asks, and it's the difference between a 14 GB table and a 96 GB one.

3. **Schema design for high-write tables.** Splitting a hot counter away from a wide `jsonb` column is not premature optimisation — it's a direct consequence of "the whole row is copied every time."

4. **Explaining PostgreSQL's operational character to a team coming from MySQL.** "Rollback is free, cleanup is not" versus "the table stays compact, rollback and long readers are not free" is the whole comparison, and it prevents a lot of bad architectural arguments.

---

## Connected topics

**Understand before this:** 04 (heap files, forks, FSM/VM), 05 (tuple header layout), 39 (transactions), 43 (anomalies), 44 (isolation levels — snapshots defined), 45 (locks — `t_xmax` doubles as a row lock).

**This unlocks:**
- **47** — VACUUM, dead tuples and bloat: the bill, in full
- **48** — deadlocks
- **50** — SSI: how SERIALIZABLE layers conflict detection on top of snapshots
- **59** — partitioning: why `DROP PARTITION` beats `DELETE`
- **62** — replication: why a long query on a replica conflicts with the primary's vacuum
- **12/18** — index-only scans and `Heap Fetches`, now fully explained
