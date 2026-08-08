# 01 — What Is a Database
## Phase: Storage Internals

---

## ELI5 — The Simple Analogy

Imagine you run a school and you keep every student's record on a paper card in a shoebox.

That works — until four things happen at once. Two teachers reach into the box for the same card at the same time and both scribble on it (**concurrency**). The building loses power halfway through you rewriting a card, and now the card says "Fees paid: ₹" with no number (**crash recovery**). The principal asks "which students in class 7 haven't paid?" and you have to touch every single card in the box (**querying**). And a student's card lists a class that was closed last year, because nobody checked (**integrity**).

A shoebox is a *file*. A **database management system** is the shoebox plus a librarian who enforces rules: only one person edits a card at a time, every change is written in a logbook *before* the card is touched, the cards are pre-sorted so you never touch all of them, and no card can name a class that doesn't exist.

**The whole of this curriculum is the story of that librarian.** Everything — pages, WAL, MVCC, locks, indexes — exists to solve one of those four problems.

---

## Where this fits in the big picture

```
        YOU ARE HERE
             ↓
    ┌─────────────────┐
    │ 01 What is a DB │  ← the four problems a DBMS solves
    └────────┬────────┘
             │
             ├──→ 02 Engine architecture   (which component solves which problem)
             ├──→ 03 Data models           (different shapes of the same solution)
             └──→ 04 Storage on disk       (it really is just files)

    Problem                Solved by             Taught in
    ─────────────────────────────────────────────────────────
    Concurrency      →     locks + MVCC      →   Phase 5 (45, 46)
    Crash recovery   →     WAL + checkpoints →   Phase 5 (41, 42)
    Querying fast    →     indexes + planner →   Phase 2 (10–19)
    Integrity        →     constraints       →   Phase 3 (23, 24)
```

Every later topic in this curriculum is a detailed answer to one of those four rows.

---

## What is this?

A **database** is an organised collection of data stored on disk. A **database management system (DBMS)** is the program that owns those files and mediates all access to them, so that many users can read and write concurrently, data survives crashes, questions can be answered without scanning everything, and invalid data is rejected at the door.

You almost never touch the database directly. You talk to the DBMS, and the DBMS touches the files.

---

## Why does it matter for a backend developer?

Because every production incident you will ever debug is one of the four problems leaking through.

- **"Two users got the same order number."** → concurrency. You did read-then-write without a lock or a unique constraint.
- **"We lost 40 seconds of orders after the EC2 instance rebooted."** → durability. Someone set `synchronous_commit = off` to make writes faster.
- **"The product list page takes 8 seconds."** → querying. No index; the engine is reading all 12 million rows.
- **"There are 3,000 `order_items` rows pointing at deleted products."** → integrity. Someone thought foreign keys were "slow" and enforced it in application code that had a bug.

If you don't know which of the four you're looking at, you will fix the wrong thing. Developers who "just add a cache" to an integrity problem, or "just add a retry" to a lock problem, make outages longer.

---

## The physical reality

A database is a directory of files. Nothing more exotic than that. Here is a real PostgreSQL 16 cluster:

```
/var/lib/postgresql/data/
├── PG_VERSION                  ← literally the text "16"
├── postgresql.conf             ← config
├── base/                       ← ALL your actual table data lives here
│   ├── 1/                      ← template1 database
│   ├── 5/                      ← template0
│   └── 16384/                  ← YOUR database (directory name = database OID)
│       ├── 16385               ← heap file for `users` — the row bytes
│       ├── 16385_fsm           ← free space map: which pages have room
│       ├── 16385_vm            ← visibility map: which pages are all-visible
│       ├── 16388               ← b-tree index file for users_pkey
│       └── 16391               ← heap file for `orders`
├── pg_wal/                     ← the write-ahead log (durability lives here)
│   ├── 000000010000000000000023
│   └── 000000010000000000000024      each file is 16MB
├── pg_xact/                    ← commit status: 2 bits per transaction
├── pg_stat/                    ← statistics collector output
└── global/                     ← cluster-wide catalogs (pg_database, pg_authid)
```

Two things to internalise right now:

1. **Table names do not exist on disk.** The file is called `16385`. The mapping from `users` → `16385` lives in a table called `pg_class`, which is itself a file. The database bootstraps its own metadata using its own storage engine.
2. **A heap file is not one blob.** It is a sequence of fixed-size **8KB pages**. A 1GB table is 131,072 pages laid end to end. All I/O happens in whole pages, never in rows.

```
users heap file (16385) — 40KB table = 5 pages
┌────────┬────────┬────────┬────────┬────────┐
│ page 0 │ page 1 │ page 2 │ page 3 │ page 4 │
│  8KB   │  8KB   │  8KB   │  8KB   │  8KB   │
└────────┴────────┴────────┴────────┴────────┘
  ↑
  byte offset 0        offset 8192      offset 32768
```

---

## How it works — step by step

Trace what actually happens when your Node.js app runs one insert. Every read and write is noted.

```
OPERATION: INSERT INTO users (email, name) VALUES ('arjun@shop.in', 'Arjun');

 1. CONNECT     Your `pg` pool hands the query to an existing TCP connection.
                PostgreSQL has one OS process per connection (postgres: backend).
                READS: nothing.

 2. PARSE       Text → parse tree. Syntax errors die here.
                READS: nothing on disk.

 3. ANALYSE     Names resolved to OIDs. "users" → 16385, "email" → attnum 2.
                READS: pg_class, pg_attribute (usually already in cache).

 4. PLAN        For an INSERT the plan is trivial: one ModifyTable node.
                READS: nothing.

 5. LOCK        RowExclusiveLock acquired on the table (not on rows — this lock
                only conflicts with schema changes, not other inserts).
                WRITES: an entry in the in-memory lock table.

 6. FIND SPACE  Free Space Map consulted: "page 4 has 3,200 bytes free."
                READS: 16385_fsm.

 7. PIN PAGE    Page 4 loaded from disk into the shared buffer pool (if not
                already there). This is a real 8KB disk read.
                READS: 16385, offset 32768, 8192 bytes.

 8. WAL RECORD  A record describing the change is appended to the WAL buffer
                in memory: "insert tuple X into relation 16385 page 4."
                WRITES: WAL buffer (RAM).

 9. HEAP WRITE  The tuple bytes are written into the page — IN MEMORY ONLY.
                The buffer is marked dirty. The disk file is still unchanged.
                WRITES: shared buffer (RAM).

10. INDEX       For every index on users, an entry is inserted into that
                index's b-tree — same pattern: WAL record, then in-memory page.
                WRITES: WAL buffer + index buffers (RAM).

11. COMMIT      WAL buffer is flushed to pg_wal and fsync()'d to physical disk.
                *** THIS is the moment the write becomes durable. ***
                Commit status recorded in pg_xact.
                WRITES: pg_wal (DISK, fsync), pg_xact.

12. ACK         "INSERT 0 1" returned to your Node.js process.

13. LATER       Minutes later, the background writer / checkpointer flushes the
                dirty page 4 to the heap file on disk.
                WRITES: 16385 (DISK).
```

**The single most important sentence in this document:** at step 12, when your API returns `201 Created`, the row is **not** in the table file yet. It is only in the log. If the machine loses power at step 12.5, PostgreSQL restarts, reads the log, and replays the change into the table file. That is durability — and it's why the log is called *write-ahead*.

---

## Concept breakdown

```
DBMS — Database Management System
│      │        │          │
│      │        │          └── System: a running program with processes,
│      │        │               memory, background workers — not a library
│      │        └── Management: it OWNS the files. Nothing else may touch
│      │             them. All access goes through it, which is precisely
│      │             what makes concurrency and integrity enforceable
│      └── Database: the organised data itself (the files)
└── (Database ≠ DBMS. PostgreSQL is the DBMS. `shop_production` is a database.)


THE FOUR PROBLEMS — memorise these four words
│
├── CONCURRENCY   many clients, same data, same instant, no corruption
├── RECOVERY      the power dies mid-write; committed data must survive
├── QUERYING      answer questions without reading every byte
└── INTEGRITY     reject data that violates the rules of the domain


VOCABULARY THAT PEOPLE CONFUSE
│
├── Cluster   one running PostgreSQL instance + its data directory
│              (confusingly: NOT multiple machines)
├── Database  a namespace inside a cluster (`shop_production`)
├── Schema    a namespace inside a database (`public`, `analytics`)
│              also loosely used to mean "the table structure"
├── Table     a relation — one heap file (plus TOAST + index files)
└── Instance  the running process set serving a cluster
```

---

## Diagrams

**Diagram 1 — the big picture: why files alone fail**

```
        ┌──────────────── WITHOUT a DBMS ────────────────┐
        │                                                │
   Node process A ──write──┐                             │
                           ├──→  users.json  ←── corrupt │
   Node process B ──write──┘        │                    │
                                    │                    │
                          power cut ↓                    │
                              half-written file          │
        └────────────────────────────────────────────────┘

        ┌────────────────  WITH  a DBMS  ────────────────┐
        │                                                │
   Node process A ──SQL──┐                               │
                         ├──→ [ DBMS ] ──→ 16385 (heap)  │
   Node process B ──SQL──┘       │    └──→ pg_wal (log)  │
                                 │                       │
                     locks · WAL · constraints · planner │
        └────────────────────────────────────────────────┘
```

**Diagram 2 — the data flow through a DBMS**

```
   Your Node.js app
         │  SQL text over TCP (port 5432)
         ▼
   ┌───────────────────────────────────────────────┐
   │  BACKEND PROCESS (one per connection)         │
   │                                               │
   │   Parser ──→ Analyser ──→ Planner ──→ Executor│
   │                                          │    │
   └──────────────────────────────────────────┼────┘
                                              ▼
   ┌───────────────────────────────────────────────┐
   │  SHARED MEMORY                                │
   │   buffer pool (8KB pages)   lock table   WAL  │
   │            │                              │   │
   └────────────┼──────────────────────────────┼───┘
                ▼                              ▼
          base/16384/16385              pg_wal/0000...24
          (table + index files)         (the log)
                     ↑                        ↑
              flushed LATER            flushed AT COMMIT
```

**Diagram 3 — before / after one INSERT (state change)**

```
BEFORE COMMIT                          AFTER COMMIT
─────────────────────────────          ─────────────────────────────
RAM  buffer page 4: [r1][r2]           RAM  page 4: [r1][r2][r3] dirty
RAM  WAL buffer:    (empty)            RAM  WAL buffer: (flushed)
DISK heap file:     [r1][r2]           DISK heap file:  [r1][r2]   ← unchanged!
DISK pg_wal:        ...                DISK pg_wal:     ...+ "insert r3"
DISK pg_xact:       txn 5001 = ?       DISK pg_xact:    txn 5001 = COMMITTED

                                       ↑ the row is DURABLE, and it is
                                         NOT in the table file yet
```

---

## Example 1 — basic

The shoebox failure, made concrete. Two Node.js processes updating a JSON file:

```js
// BROKEN — this is what "just use files" means
const fs = require('fs');

function addOrder(order) {
  const db = JSON.parse(fs.readFileSync('orders.json'));   // ① read
  db.orders.push(order);                                    // ② modify
  fs.writeFileSync('orders.json', JSON.stringify(db));      // ③ write
}
```

Run two processes concurrently:

```
  time    Process A                    Process B
  ────────────────────────────────────────────────────────
   t1     ① reads 100 orders
   t2                                  ① reads 100 orders
   t3     ② pushes order #101
   t4                                  ② pushes order #102
   t5     ③ writes 101 orders
   t6                                  ③ writes 101 orders   ← A's order is GONE
```

This is the **lost update** anomaly (Topic 43). And if the power dies during step ③, `orders.json` is truncated garbage — you lose all 100 orders, not just the new one.

The database version cannot lose the write:

```sql
INSERT INTO orders (user_id, total_paise) VALUES (7, 249900);
```

Two backends running this concurrently each append their own tuple to a page under a page-level pin. Neither reads-then-overwrites the other's work. And a crash mid-statement leaves the table exactly as it was, because the WAL record was never marked committed.

---

## Example 2 — production scenario

**The situation.** Your e-commerce API runs on 6 Node.js pods behind an ALB. Checkout writes to `orders`, `order_items`, and `payments`. During a flash sale you see:

- Support tickets: "I was charged twice."
- Metrics: p99 on `GET /products` jumped from 80ms to 6s.
- A nightly job reports 412 `order_items` rows whose `product_id` doesn't exist in `products`.

**The diagnosis, by problem class:**

| Symptom | Problem class | Root cause | Where we fix it |
|---|---|---|---|
| Charged twice | Concurrency | Checkout does `SELECT` then `INSERT` with no unique constraint on the idempotency key; two retries raced | Topic 24 (UNIQUE), 49 (concurrency control), 52 (idempotency) |
| `GET /products` 6s | Querying | `WHERE category_id = ? AND is_active` with no index → sequential scan of 4.1M rows = 51,000 page reads | Topic 10–18 |
| Orphan `order_items` | Integrity | A cleanup script deleted products; FK was never declared "because ORMs handle it" | Topic 23 |

**The point of this example:** three unrelated-looking incidents, three different problem classes, three completely different fixes. Adding Redis fixes none of them. Adding more pods makes the first and second *worse*, because more concurrent connections means more contention and more sequential scans.

Diagnosing the class before the fix is the skill. That is what Phases 2, 5 and 7 build.

---

## Common mistakes

**1. "We'll store it in JSON files / on S3 and read it in the app."**
- *Symptom:* works for 3 months, then silent data loss under concurrency, and every query becomes "download everything and filter in JavaScript."
- *Engine-level why:* the filesystem gives you no atomicity above a single `write()` syscall, no isolation between processes, and no access path other than "read the whole object."
- *Diagnose:* look for `readFileSync` + mutate + `writeFileSync` patterns, or S3 `getObject` in a request handler.
- *Fix:* a DBMS. Use files for **blobs** (images, PDFs) and a database for **facts**.

**2. Enforcing integrity only in application code.**
- *Symptom:* orphan rows, duplicate emails, negative stock — appearing only for records created by the one code path that forgot the check.
- *Engine-level why:* application checks are `SELECT` then `INSERT`, two statements, with a gap in between. Two concurrent requests both see "no duplicate" and both insert. A `UNIQUE` constraint is enforced inside the index insert itself, atomically, with no gap.
- *Diagnose:* `SELECT count(*) FROM order_items oi LEFT JOIN products p ON p.id = oi.product_id WHERE p.id IS NULL;`
- *Fix:* declare the constraint. Keep the app check too, for a nice error message — but the database is the one that's actually true.

**3. Believing `COMMIT` means "written to the table file."**
- *Symptom:* confusion when a table file's mtime doesn't change after inserts; panic-tuning `checkpoint` settings; or worse, turning off `synchronous_commit` and losing data on reboot.
- *Engine-level why:* commit flushes the **WAL**, not the heap. Heap pages are flushed lazily by the checkpointer. Durability is a property of the log.
- *Diagnose:* `SHOW synchronous_commit;` — if it's `off`, you have accepted a window of data loss.
- *Fix:* leave `synchronous_commit = on` unless you can name the exact rows you're willing to lose.

**4. Treating "database" and "DBMS" as interchangeable in design discussions.**
- *Symptom:* "let's give each tenant their own database" — and nobody knows whether that means a separate namespace (cheap) or a separate instance (expensive). Six months later you have 4,000 PostgreSQL databases in one cluster and `pg_dump` takes 9 hours.
- *Fix:* say cluster / database / schema / table precisely. Topic 27 covers multi-tenancy properly.

**5. One connection per request, no pool.**
- *Symptom:* under load, `FATAL: sorry, too many clients already`; server RAM climbs; everything slows down at once.
- *Engine-level why:* a PostgreSQL connection is an OS **process** with its own ~10MB of memory and its own entry in every shared data structure. 500 of them is 500 processes competing for CPU.
- *Fix:* a pool (Topic 65).

---

## Hands-on proof

Start a lab instance:

```bash
docker run -d --name pg-lab -e POSTGRES_PASSWORD=lab -p 5432:5432 postgres:16
docker exec -it pg-lab psql -U postgres
```

**PROVE IT #1 — a database is a directory of files.**

```sql
CREATE DATABASE shop;
\c shop
CREATE TABLE users (id bigserial PRIMARY KEY, email text, name text);
INSERT INTO users (email, name) SELECT 'u'||i||'@shop.in', 'User '||i
  FROM generate_series(1,10000) i;

-- Which file is this table?
SELECT pg_relation_filepath('users');
```
```
 pg_relation_filepath
----------------------
 base/16388/16390
```
```bash
docker exec -it pg-lab ls -la /var/lib/postgresql/data/base/16388/ | head
```
```
-rw------- 1 postgres postgres  647168 Aug  7 12:01 16390       ← 79 pages × 8KB
-rw------- 1 postgres postgres   24576 Aug  7 12:01 16390_fsm
-rw------- 1 postgres postgres  245760 Aug  7 12:01 16393       ← the pkey index
```

**PROVE IT #2 — the table is measured in 8KB pages, not rows.**

```sql
SELECT relname, relpages, reltuples, pg_size_pretty(pg_relation_size(oid))
FROM pg_class WHERE relname = 'users';
```
```
 relname | relpages | reltuples | pg_size_pretty
---------+----------+-----------+----------------
 users   |       79 |     10000 | 632 kB
```
79 pages × 8192 bytes = 647,168 bytes. 10,000 rows in 79 pages ≈ 126 rows per page.

**PROVE IT #3 — the page size is a compile-time constant.**

```sql
SHOW block_size;
```
```
 block_size
------------
 8192
```

**PROVE IT #4 — commit writes to the log, and you can watch the log position move.**

```sql
SELECT pg_current_wal_lsn();          -- 0/1A2B3C8
INSERT INTO users (email, name) VALUES ('new@shop.in','New');
SELECT pg_current_wal_lsn();          -- 0/1A2B4A0   ← advanced by ~216 bytes
SELECT pg_walfile_name(pg_current_wal_lsn());
```
```
     pg_walfile_name
--------------------------
 000000010000000000000001
```

**PROVE IT #5 — table names live in a table.**

```sql
SELECT oid, relname, relfilenode, relkind FROM pg_class WHERE relname LIKE 'users%';
```
```
  oid  |   relname   | relfilenode | relkind
-------+-------------+-------------+---------
 16390 | users       |       16390 | r        ← r = ordinary table
 16393 | users_pkey  |       16393 | i        ← i = index
```

**PROVE IT #6 — the lost update, live.** Open two `psql` sessions:

```sql
-- session 1                      -- session 2
BEGIN;
SELECT stock FROM products
  WHERE id=1;   -- 10
                                  BEGIN;
                                  SELECT stock FROM products WHERE id=1;  -- 10
UPDATE products SET stock = 9
  WHERE id=1;
COMMIT;
                                  UPDATE products SET stock = 9 WHERE id=1;
                                  COMMIT;
-- two units sold, stock went 10 → 9. One unit vanished.
```
This is the exact bug the rest of Phase 5 exists to prevent.

---

## The design decision framework

```
USE A DATABASE (DBMS) WHEN:
  ✓ More than one process or request may write the same data
  ✓ Losing a committed write is unacceptable
  ✓ You need to answer questions about a subset without reading everything
  ✓ There are rules the data must always satisfy (uniqueness, references, ranges)
  ✓ The data outlives any single deployment of your app

USE PLAIN FILES / OBJECT STORAGE WHEN:
  ✗ (i.e. avoid a DBMS when)
  ✓ The unit is a large opaque blob — images, video, PDFs, model weights
  ✓ It is written once and never partially updated
  ✓ There are no queries other than "give me this exact key"
  ✓ It is a build artefact, log stream, or cache that can be regenerated

THE SIGNAL TO LOOK FOR:
  Do two things need to be true at the same instant?
  ("the order exists AND the stock is decremented")
  If yes → you need transactions → you need a database.
  If the answer is "there is only ever one writer and one reader of one whole
  blob" → a file is genuinely fine, and cheaper.

THE HYBRID (what real systems do):
  Blob bytes → S3.  The row that says the blob exists, who owns it, and
  whether it is public → the database. Never the reverse.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create a table with 50,000 rows. Without counting rows, work out from `pg_class` how many rows fit in one 8KB page on average, and explain in one sentence why the number is not exactly `8192 / row_size`.

### Exercise 2 — medium (apply it)
Your teammate proposes: *"Sessions are ephemeral, so let's write them to a JSON file per pod instead of the database — it'll be faster."* You run 6 pods behind a load balancer. Write down (a) the exact user-visible bug this creates, (b) which of the four problems it is, and (c) two correct alternatives with the trade-off of each.

### Exercise 3 — hard (production simulation)
You inherit a service where `POST /orders` sometimes creates two orders for one click. The handler is:

```js
const existing = await db.query(
  'SELECT id FROM orders WHERE idempotency_key = $1', [key]);
if (existing.rows.length) return existing.rows[0];
const created = await db.query(
  'INSERT INTO orders (user_id, idempotency_key, total_paise) VALUES ($1,$2,$3) RETURNING *',
  [userId, key, total]);
return created.rows[0];
```

(a) Draw the exact two-request timeline that produces two orders.
(b) Name the problem class.
(c) Give a fix that works even with 6 pods and no shared application memory, and explain at the engine level *why* your fix cannot race — what makes it atomic where the code above is not.

---

## Mental model checkpoint

Answer from memory. If you can't, re-read.

1. Name the four problems a DBMS solves that a plain file cannot. For each, name the mechanism that solves it.
2. When `COMMIT` returns successfully, which file on disk is guaranteed to contain your change — the heap file or the WAL? Why that one?
3. Your table is called `orders`. What is it called on disk, and where does the mapping live?
4. Why does the database read 8KB when you asked for one 90-byte row?
5. A colleague says "we enforce uniqueness in the API, so we don't need a UNIQUE constraint." Give the two-request timeline that proves them wrong.
6. What is the difference between a cluster, a database, and a schema in PostgreSQL?
7. If you `INSERT` and the machine loses power one millisecond after the client sees "INSERT 0 1", is the row there when the server comes back? Walk through why.

---

## Quick reference card

| Concept | One line |
|---|---|
| Database | The organised data — a set of files |
| DBMS | The program that owns those files and mediates access |
| Cluster | One running instance + its data directory |
| Schema | A namespace inside a database |
| Heap file | The file holding a table's row data, as 8KB pages |
| WAL | The log written *before* the heap; the source of durability |
| Page | The unit of all database I/O |

**Numbers to memorise**

| Thing | Value |
|---|---|
| PostgreSQL page size | **8 KB** (8192 bytes, compile-time) |
| WAL segment file size | 16 MB (default) |
| Page header | 24 bytes |
| Item pointer | 4 bytes each |
| A connection in PostgreSQL | one OS **process**, ~5–10 MB |
| Rows per page, typical narrow table | ~50–200 |

**Key file paths**

| Path | Contents |
|---|---|
| `base/<db_oid>/<relfilenode>` | table and index data |
| `pg_wal/` | write-ahead log segments |
| `pg_xact/` | transaction commit status |
| `global/` | cluster-wide catalogs |

**Decisions**

| Question | Answer |
|---|---|
| Blob bytes? | Object storage |
| Facts about the blob? | Database |
| Two things must be true at once? | Database, in one transaction |
| Regenerable, single-writer, whole-file access? | A file is fine |

---

## When would I use this at work?

1. **Design review.** A teammate proposes keeping feature flags in a JSON file synced to every pod. You ask: is it ever written by more than one process? Does a stale flag cause a correctness bug or just a cosmetic one? That framing — which of the four problems applies — settles the argument in two minutes instead of two meetings.

2. **Incident triage at 2am.** Orders are duplicating. Instead of guessing, you classify: is this concurrency, recovery, querying, or integrity? Duplicates under retry = concurrency + missing constraint. You go straight to `\d orders` to check for a unique index rather than reading 400 lines of handler code.

3. **Explaining a data-loss postmortem.** Someone set `synchronous_commit = off` to hit a latency target and you lost 12 seconds of writes on a reboot. You can now explain to leadership exactly which physical step was skipped — the `fsync()` of the WAL at commit — and what the trade was.

---

## Connected topics

**Understand before this:** nothing. This is the entry point.

**This unlocks:**
- **02 — Database engine architecture**: the components that solve each of the four problems
- **04 — How data is stored on disk**: the 8KB page opened up byte by byte
- **41 — The write-ahead log**: why commit means "log flushed", in full detail
- **23/24 — Foreign keys and constraints**: the integrity half of the four problems
