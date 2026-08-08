# 05 — Row Storage Format and Tuple Layout
## Phase: Storage Internals

---

## ELI5 — The Simple Analogy

A shipping crate at a port.

The crate has a **manifest stapled to the lid** — who sent it, when, whether it's been cancelled, which items are missing. That's the tuple header. Below it are the goods, packed in a fixed order. But the port has a rule: **every item must start on a mark painted on the floor, 8 marks apart.** So if a small item ends between marks, you leave a gap and start the next item on the next mark. That wasted gap is **alignment padding**, and it is real — reorder the items and the same crate holds noticeably more.

And if something is too big for the crate — a car engine — you don't force it in. You put a **claim ticket** in the crate and store the engine in a separate warehouse. That's **TOAST**.

---

## Where this fits in the big picture

```
   04 Pages on disk (the folder)
              │
              ▼
   05 TUPLE LAYOUT  ← YOU ARE HERE   (one sheet inside the folder)
              │
      ┌───────┼────────────────┬──────────────────┐
      ▼       ▼                ▼                  ▼
 25 Data    46 MVCC       17 Write         06 Row vs
 types      (xmin/xmax     amplification    column store
 (choosing   live in this  (whole tuple
  them)      header)        rewritten)
```

Topic 04 told you how many rows fit in a page. **This topic tells you why that number is what it is, and how to change it.**

---

## What is this?

A **tuple** is one physical row version, stored as a contiguous byte string inside a page. It consists of a 23-byte header (padded to 24), an optional NULL bitmap, and the column values laid out in `attnum` order with alignment padding between them.

"Row version," not "row" — because MVCC means one logical row can have many tuples on disk simultaneously (Topic 46).

---

## Why does it matter for a backend developer?

Three concrete consequences you will hit:

1. **Column order changes table size.** Same columns, same data, different declaration order → up to 20–30% difference in bytes. That's 20–30% fewer pages, fewer I/Os, more of the table cached.
2. **Every row costs 24+ bytes of overhead before any data.** A table of `(int, int)` is not 8 bytes/row — it's 32. If you're designing a 500M-row join table, that's 12 GB of pure overhead.
3. **A `text` column that's usually 5 KB behaves completely differently from one that's usually 500 bytes** — the first is TOASTed out of line (cheap to skip, expensive to read), the second is inline (destroys your rows-per-page).

And the MVCC fields live here — `xmin`, `xmax`, `t_ctid` — so this is where you actually *see* why an update is a copy.

---

## The physical reality

### HeapTupleHeaderData — the exact 23 bytes

```
byte  0 ┌────────────────────────────────────────────────────────────┐
        │ t_xmin        (4 B)  XID that INSERTED this tuple           │
      4 │ t_xmax        (4 B)  XID that DELETED/UPDATED it (0 = live) │
      8 │ t_cid / t_xvac(4 B)  command id within the transaction      │
     12 │ t_ctid        (6 B)  (block, offset) — SELF, or the NEXT    │
        │                      version in an update chain             │
     18 │ t_infomask2   (2 B)  # of attributes + HOT/keys flags       │
     20 │ t_infomask    (2 B)  HAS_NULL, HAS_VARWIDTH, XMIN_COMMITTED,│
        │                      XMAX_COMMITTED, ... (hint bits)        │
     22 │ t_hoff        (1 B)  offset from tuple start to the DATA    │
     23 ├────────────────────────────────────────────────────────────┤
        │ NULL BITMAP  (optional) — 1 bit per column, present only if │
        │              the tuple has at least one NULL                │
        │              ceil(ncols/8) bytes, then padded to 8-byte     │
        │              alignment                                      │
   t_hoff├────────────────────────────────────────────────────────────┤
        │ COLUMN DATA — in attnum order, with alignment padding       │
        └────────────────────────────────────────────────────────────┘

t_hoff is 24 with no NULLs (23 rounded up to 8-byte alignment).
With NULLs and ≤ 8 columns, still 24 (the bitmap fits in the padding — free!).
With 9–64 columns and any NULL, t_hoff = 32.
```

### Alignment — the rule that wastes your bytes

Every data type has an alignment requirement. A value must start at a byte offset that is a multiple of its alignment. Gaps are filled with zero padding.

| Type | Storage | Align | Notes |
|---|---|---|---|
| `bool`, `"char"` | 1 | 1 | |
| `smallint` (int2) | 2 | 2 | |
| `int`, `real`, `date` | 4 | 4 | |
| `bigint`, `double`, `timestamptz`, `money` | 8 | 8 | |
| `uuid` | 16 | 1 | (char-aligned in PG!) |
| `numeric` | var | 4 | never use for hot paths — it's software math |
| `text`, `varchar`, `bytea`, `jsonb` | var | 4 | 1-byte header if ≤126 B, else 4-byte |

**Worked example — the same table, two column orders:**

```
BAD ORDER                              GOOD ORDER
CREATE TABLE orders (                  CREATE TABLE orders (
  is_paid   bool,        -- 1 B          id        bigint,      -- 8 B  off 0
  id        bigint,      -- 8 B          user_id   bigint,      -- 8 B  off 8
  qty       smallint,    -- 2 B          total     bigint,      -- 8 B  off 16
  user_id   bigint,      -- 8 B          created   timestamptz, -- 8 B  off 24
  status    "char",      -- 1 B          qty       smallint,    -- 2 B  off 32
  total     bigint,      -- 8 B          is_paid   bool,        -- 1 B  off 34
  created   timestamptz  -- 8 B          status    "char"       -- 1 B  off 35
);                                     );

LAYOUT (offsets after the 24-byte header):
 off 0  is_paid   1 B                   off 0  id       8
 off 1  ....PAD 7 B....   ← wasted      off 8  user_id  8
 off 8  id        8 B                   off 16 total    8
 off 16 qty       2 B                   off 24 created  8
 off 18 ..PAD 6 B..       ← wasted      off 32 qty      2
 off 24 user_id   8 B                   off 34 is_paid  1
 off 32 status    1 B                   off 35 status   1
 off 33 ..PAD 7 B..       ← wasted      off 36 ..pad 4 (row alignment)
 off 40 total     8 B
 off 48 created   8 B
 DATA = 56 bytes                        DATA = 40 bytes
 TUPLE = 24 + 56 = 80 bytes             TUPLE = 24 + 40 = 64 bytes

 rows/page ≈ 8160/(80+4) = 97           rows/page ≈ 8160/(64+4) = 120
 100M rows → 8.4 GB                     100M rows → 6.8 GB     ← 20% smaller
```

**The rule: declare columns in descending order of alignment.** 8-byte types first, then 4, then 2, then 1-byte and variable-length last.

### TOAST — The Oversized-Attribute Storage Technique

A tuple cannot span pages. So what happens when a `text` value is 4 MB?

```
DECISION TREE when a tuple exceeds TOAST_TUPLE_THRESHOLD (~2000 bytes):

  1. Try compressing the largest varlena attribute (pglz or lz4)
  2. Still too big? Move it OUT OF LINE to the TOAST table, leaving an
     18-byte pointer in the tuple
  3. Repeat with the next-largest attribute until the tuple fits

TOAST TABLE — an ordinary table, one per parent table that needs it:
  pg_toast.pg_toast_16390 (chunk_id oid, chunk_seq int, chunk_data bytea)
  Each chunk is ~2000 bytes. A 4 MB value = ~2000 chunks.
  Indexed by (chunk_id, chunk_seq).

THE POINTER left in the main tuple (18 bytes):
  ┌──────────────────────────────────────────────────┐
  │ va_header(1) │ rawsize(4) │ extsize(4) │         │
  │ valueid(4)   │ toastrelid(4)                     │
  └──────────────────────────────────────────────────┘

PER-COLUMN STORAGE STRATEGY:
  PLAIN     never compress, never move out of line (fixed-width types)
  EXTENDED  compress, then move out of line if needed   ← DEFAULT for text/jsonb
  EXTERNAL  move out of line WITHOUT compressing        ← use for substring() access
  MAIN      compress, avoid moving out of line if possible
  ALTER TABLE docs ALTER COLUMN body SET STORAGE EXTERNAL;
```

**Why this matters enormously:** a TOASTed column costs *nothing* on a scan that doesn't select it — the main tuple only carries the 18-byte pointer. An inline 1,800-byte column costs you on **every** page read of that table, forever, whether you select it or not.

---

## How it works — step by step

**Building the tuple for `INSERT INTO orders (id, user_id, total_paise, status, notes) VALUES (91, 7, 249900, 'paid', <8KB of text>);`**

```
 1. Compute the header.
      t_xmin = 5001 (my XID), t_xmax = 0, t_ctid = (self, filled after placement)
      t_infomask2 = 5 attributes
      t_infomask: HEAP_HASVARWIDTH set (we have text columns)
      Any NULLs? No → no NULL bitmap → t_hoff = 24

 2. Lay out the columns in attnum order with alignment.
      attnum 1  id          bigint  align 8 → offset  0, 8 bytes
      attnum 2  user_id     bigint  align 8 → offset  8, 8 bytes
      attnum 3  total_paise bigint  align 8 → offset 16, 8 bytes
      attnum 4  status      text    align 4 → offset 24, 1+4 = 5 bytes
                                              ('paid' ≤126 B → 1-byte varlena header)
      attnum 5  notes       text    align 4 → offset 29 → pad to 32, then...

 3. SIZE CHECK: 24 + 32 + 8192 = 8,248 bytes.  > TOAST_TUPLE_THRESHOLD (2000).
      → TOAST kicks in on the largest varlena: `notes`.

 4. COMPRESS `notes` with lz4. 8192 → 3100 bytes. Tuple would be 3,156. Still > 2000.

 5. MOVE `notes` OUT OF LINE:
      - allocate a new chunk_id in pg_toast.pg_toast_16390
      - split 3100 compressed bytes into 2 chunks (2000 + 1100)
      - INSERT both chunks (each is a normal heap insert WITH ITS OWN WAL RECORD)
      - INSERT 2 entries into the toast index
      - replace `notes` in the main tuple with an 18-byte pointer

 6. FINAL TUPLE: 24 (header) + 8+8+8 (bigints) + 5 (status) + pad 3 + 18 (toast ptr)
              = 74 bytes, padded to 8-byte boundary = 80 bytes.

 7. Written into the page as in Topic 04.

⚠ ONE INSERT produced: 1 heap tuple + 2 toast tuples + 2 toast index entries
  + 1 pkey index entry + 6 WAL records. This is why "just store the payload
  in the row" is not free.
```

**Reading it back:**

```
 SELECT id, status FROM orders WHERE id = 91;
   → 1 heap page read. The toast pointer is never dereferenced. FAST.

 SELECT notes FROM orders WHERE id = 91;
   → 1 heap page read + toast index lookup + 2 toast heap reads + decompress.
   "de-TOASTing" — this is why SELECT * on a table with big text columns
   is dramatically slower than selecting the columns you need.
   *** THIS is the case where SELECT * genuinely costs more (Topic 04, mistake #2). ***
```

---

## Concept breakdown

```
TUPLE
│  └── One physical ROW VERSION. Not "a row" — MVCC means many tuples per row.
│
├── HEADER (23 B → 24 with padding)
│    ├── t_xmin      who created it        ─┐
│    ├── t_xmax      who killed it          ├── THE MVCC FIELDS (Topic 46)
│    ├── t_ctid      self, or next version ─┘
│    ├── t_infomask  hint bits — cached "was xmin's txn committed?" answers,
│    │               set lazily on first read after commit. This is why the
│    │               FIRST read after a big write is slower (it writes hint
│    │               bits back, dirtying pages).
│    └── t_hoff      where the data starts
│
├── NULL BITMAP (optional)
│    └── 1 bit per column, present only if ANY column is NULL.
│        ⇒ A NULL column costs ~0 bytes of data. A NOT NULL column of the
│          same type costs its full width. "NULL is expensive" is a myth;
│          the bitmap is usually free (it fits in existing padding).
│
└── DATA
     ├── attnum order — the order columns were DECLARED (ALTER TABLE ADD
     │   appends, so added columns are always last)
     ├── alignment padding between values
     └── varlena values: 1-byte header if ≤126 bytes, else 4-byte header

TOAST — The Oversized-Attribute Storage Technique
│
├── Threshold ~2000 bytes (TOAST_TUPLE_THRESHOLD = 1/4 of a page)
├── Strategy: compress → move out of line → repeat on next-largest column
├── Chunks of ~2000 bytes in a side table with its own index
└── Max value size: 1 GB
```

---

## Diagrams

**Diagram 1 — big picture: a tuple in its page in its file**

```
 base/16388/16390  (heap file)
 └── page 1204 (8192 B)
      └── line pointer [3] → offset 7920, length 80
           └── ┌──────────────────────────────────────────────┐  ← byte 7920
               │ t_xmin 5001 │ t_xmax 0 │ t_cid 0             │
               │ t_ctid (1204,3) │ infomask2 │ infomask │hoff│
               ├──────────────────────────────────────────────┤  ← byte 7944 (hoff=24)
               │ id 91        (8 B)                           │
               │ user_id 7    (8 B)                           │
               │ total 249900 (8 B)                           │
               │ status 'paid'(5 B) │ pad (3 B)               │
               │ notes → TOAST pointer (18 B)                 │
               └──────────────────────────────────────────────┘  ← byte 8000
                                        │
                                        ▼
                    pg_toast.pg_toast_16390
                    ┌───────────┬───────────┬────────────────┐
                    │ chunk_id  │ chunk_seq │ chunk_data     │
                    │ 41022     │     0     │ <2000 bytes>   │
                    │ 41022     │     1     │ <1100 bytes>   │
                    └───────────┴───────────┴────────────────┘
```

**Diagram 2 — data flow: how alignment padding accumulates**

```
  DECLARED ORDER: bool, bigint, smallint, bigint, text

  offset:  0    1                    8         16  18            24
          ┌────┬────────────────────┬─────────┬───┬──────────────┬──────────
          │bool│  P A D D I N G     │ bigint  │i2 │  PADDING     │ bigint
          │ 1B │       7 B  ✗       │   8 B   │2B │    6 B  ✗    │  8 B
          └────┴────────────────────┴─────────┴───┴──────────────┴──────────
                    13 bytes of pure waste, per row, forever

  REORDERED:      bigint, bigint, smallint, bool, text

  offset:  0              8              16  18   19
          ┌──────────────┬──────────────┬───┬────┬───────────
          │   bigint     │   bigint     │i2 │bool│  text
          │     8 B      │     8 B      │2B │ 1B │
          └──────────────┴──────────────┴───┴────┴───────────
                    0 bytes of waste
```

**Diagram 3 — before/after: what an UPDATE does to the tuple**

```
BEFORE:  UPDATE orders SET status='shipped' WHERE id=91;

  page 1204                                    page 1204
  [3] → ┌──────────────────────┐               [3] → ┌──────────────────────┐
        │ xmin=5001  xmax=0    │                     │ xmin=5001 xmax=5099  │◀ killed
        │ ctid=(1204,3)        │        ⇒            │ ctid=(1204,9)        │─┐
        │ id=91 status='paid'  │                     │ id=91 status='paid'  │ │
        └──────────────────────┘                     └──────────────────────┘ │
                                               [9] → ┌──────────────────────┐◀┘
                                                     │ xmin=5099 xmax=0     │
                                                     │ ctid=(1204,9)        │
                                                     │ id=91 status='shipped'│
                                                     └──────────────────────┘

  • The ENTIRE row is copied, including the 18-byte TOAST pointer
    (TOAST chunks are NOT rewritten if that column didn't change — a real saving)
  • Old tuple stays until VACUUM
  • If NO indexed column changed AND it fit on the same page → HOT update,
    indexes untouched. Otherwise every index gets a new entry.
```

---

## Example 1 — basic

Prove alignment costs real bytes.

```sql
CREATE TABLE bad_order  (a bool, b bigint, c smallint, d bigint, e "char", f bigint);
CREATE TABLE good_order (b bigint, d bigint, f bigint, c smallint, a bool, e "char");

INSERT INTO bad_order  SELECT true, i, 1, i, 'x', i FROM generate_series(1,500000) i;
INSERT INTO good_order SELECT i, i, i, 1, true, 'x' FROM generate_series(1,500000) i;

SELECT relname, relpages, pg_size_pretty(pg_relation_size(oid)),
       round(reltuples/relpages) AS rows_per_page
FROM pg_class WHERE relname IN ('bad_order','good_order');
```
```
  relname   | relpages | pg_size_pretty | rows_per_page
------------+----------+----------------+---------------
 bad_order  |     3922 | 31 MB          |           128
 good_order |     3186 | 25 MB          |           157
```
**Identical data. 19% less disk, 23% more rows per page, 19% fewer I/Os on every scan, forever.** Confirm the per-tuple size directly:

```sql
SELECT pg_column_size(t.*) FROM bad_order  t LIMIT 1;   -- 56
SELECT pg_column_size(t.*) FROM good_order t LIMIT 1;   -- 48
```

Now prove NULLs are nearly free:

```sql
CREATE TABLE with_nulls AS SELECT i AS a, NULL::bigint AS b, NULL::text AS c
                           FROM generate_series(1,500000) i;
CREATE TABLE no_nulls   AS SELECT i AS a, i::bigint AS b, 'x'::text AS c
                           FROM generate_series(1,500000) i;
SELECT relname, relpages FROM pg_class WHERE relname IN ('with_nulls','no_nulls');
```
```
  relname   | relpages
------------+----------
 with_nulls |     1959
 no_nulls   |     2735    ← storing actual values costs MORE than NULLs
```

---

## Example 2 — production scenario

**The situation.** `webhook_events` table, 900M rows, 640 GB. It stores every incoming payment-gateway webhook.

```sql
CREATE TABLE webhook_events (
  id            bigserial PRIMARY KEY,
  received_at   timestamptz NOT NULL,
  provider      text NOT NULL,          -- 'razorpay' | 'stripe' — ~8 bytes
  event_type    text NOT NULL,          -- 'payment.captured'   — ~18 bytes
  order_id      bigint,
  processed     boolean NOT NULL DEFAULT false,
  raw_payload   text NOT NULL,          -- avg 1,400 bytes  ⚠
  signature     text NOT NULL           -- 64 bytes
);
```

The reprocessing job — `SELECT id, order_id FROM webhook_events WHERE processed = false AND received_at > now() - interval '1 day'` — reads 40 GB and takes 25 minutes.

**Step 1 — where are the bytes?**

```sql
SELECT pg_size_pretty(pg_relation_size('webhook_events'))       AS main,
       pg_size_pretty(pg_total_relation_size('webhook_events')
                    - pg_relation_size('webhook_events'))       AS toast_and_idx;
```
```
  main   | toast_and_idx
---------+---------------
 612 GB  | 28 GB
```

**Only 28 GB is TOAST.** That means `raw_payload` at 1,400 bytes is **below the ~2000-byte threshold — it is stored INLINE.** Every one of the 900M rows carries 1,400 bytes of JSON that the reprocessing job never reads.

```sql
SELECT avg(pg_column_size(raw_payload)) FROM webhook_events TABLESAMPLE SYSTEM (0.01);
-- 1387
SELECT round(reltuples/relpages) AS rows_per_page FROM pg_class WHERE relname='webhook_events';
-- 5      ← FIVE rows per 8KB page
```

**Step 2 — the three fixes, measured**

| Fix | Mechanism | Result |
|---|---|---|
| **(a) Force `raw_payload` out of line** — `ALTER TABLE webhook_events ALTER COLUMN raw_payload SET STORAGE EXTERNAL;` then rewrite | value moves to the TOAST table; main tuple carries an 18-byte pointer | rows/page 5 → ~85. Main relation 612 GB → **~38 GB**. The job's scan drops from 40 GB to 2.5 GB. Note: `SET STORAGE` only affects *future* writes — existing rows need `VACUUM FULL` / `pg_repack` |
| **(b) Reorder columns** on the rebuilt table: `id, received_at, order_id` (8B each) then `processed` (1B) then the varlenas | removes ~14 bytes of padding per row | another ~10% off the main relation |
| **(c) Partial index** — `CREATE INDEX ON webhook_events (received_at) WHERE NOT processed` | the index contains only unprocessed rows — typically a few thousand, not 900M | the job's index is 200 KB instead of 30 GB (Topic 13) |

**Step 3 — what you'd actually do**

Fix (c) alone takes the job from 25 minutes to 40 milliseconds — it changes the *access path*, which beats any storage optimisation. Fixes (a) and (b) still matter, because they cut 570 GB of storage cost and make every *other* query on this table faster.

**And the real lesson:** the `raw_payload` at 1,400 bytes was the worst possible size. At 500 bytes it would be a minor cost. At 3,000 bytes TOAST would have handled it automatically and the problem would never have existed. **The dangerous zone is values just under the TOAST threshold** — big enough to destroy rows-per-page, small enough that PostgreSQL leaves them inline.

---

## Common mistakes

**1. Ignoring column order in large tables.**
- *Symptom:* a table is 25% bigger than the sum of its column widths suggests.
- *Engine-level why:* alignment padding between values of different widths.
- *Diagnose:* `SELECT pg_column_size(t.*) FROM tbl t LIMIT 1;` and compare with the theoretical minimum. Or query `pg_attribute` ordered by `attnum` and hand-compute.
- *Fix:* declare 8-byte types first, descending to 1-byte, variable-length last. Costs nothing, applies at `CREATE TABLE` time. For existing tables it needs a rewrite — worth it above ~50 GB.

**2. Values in the 1–2 KB "dead zone."**
- *Symptom:* rows-per-page in single digits; scans read enormous amounts of data that no query uses.
- *Engine-level why:* under TOAST_TUPLE_THRESHOLD, so stored inline, so present in every page read.
- *Diagnose:* `SELECT round(reltuples/relpages) FROM pg_class WHERE relname='...';` — if under 20 on a large table, investigate.
- *Fix:* `SET STORAGE EXTERNAL` (or `EXTENDED` + rewrite), or move the column to a 1:1 side table.

**3. `SELECT *` on tables with TOASTed columns in a loop.**
- *Symptom:* an API endpoint is slow and the DB shows high I/O on `pg_toast_*` relations.
- *Engine-level why:* every de-TOAST is an index lookup plus N chunk reads plus decompression, per row, per column.
- *Diagnose:* `SELECT relname, heap_blks_read FROM pg_statio_user_tables WHERE relname LIKE 'pg_toast%';` — or just compare `SELECT id` vs `SELECT *` timings.
- *Fix:* select the columns you need. This is the one place where it genuinely matters (contrast with Topic 04's mistake #2).

**4. Believing NULL columns waste space, so using `''` or `0` sentinels.**
- *Symptom:* bloated tables and, worse, `WHERE x <> ''` logic bugs and broken aggregates.
- *Engine-level why:* a NULL sets a bit in a bitmap that usually fits in existing header padding and stores **zero** data bytes. A `''` text value costs at least 1 byte plus alignment; a `0` bigint costs 8.
- *Fix:* use NULL for "unknown/absent." It's smaller *and* semantically correct.

**5. Adding columns with `ALTER TABLE ADD COLUMN` and expecting good layout.**
- *Symptom:* a well-ordered table gradually degrades as columns are appended over years.
- *Engine-level why:* new columns get the highest `attnum` and are laid out last, regardless of alignment. A `bigint` added after a `bool` will be padded to alignment.
- *Fix:* accept it for small tables; on very large tables, plan a periodic rewrite (`pg_repack` with a reordered definition) as part of a migration.

**6. `numeric` for money on hot paths.**
- *Symptom:* CPU-bound aggregation queries; larger-than-expected rows.
- *Engine-level why:* `numeric` is variable-length, 4-byte aligned, and arithmetic is done in software (base-10000 digit arrays), not by the CPU's ALU.
- *Fix:* store money as `bigint` in the smallest unit (paise/cents). Exact, 8 bytes, hardware arithmetic. Format at the application edge. (Topic 26.)

---

## Hands-on proof

**PROVE IT #1 — see the header fields.**
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE TABLE t (a bigint, b text);
INSERT INTO t VALUES (1, 'hello');
SELECT lp, lp_len, t_xmin, t_xmax, t_ctid, t_infomask2, t_infomask, t_hoff
FROM heap_page_items(get_raw_page('t', 0));
```
```
 lp | lp_len | t_xmin | t_xmax | t_ctid | t_infomask2 | t_infomask | t_hoff
----+--------+--------+--------+--------+-------------+------------+--------
  1 |     38 |    912 |      0 | (0,1)  |           2 |       2050 |     24
```
38 bytes = 24 header + 8 (bigint) + 6 (`'hello'` = 1-byte varlena header + 5 chars). `t_hoff = 24` — no NULL bitmap needed.

**PROVE IT #2 — the NULL bitmap appears.**
```sql
INSERT INTO t VALUES (2, NULL);
SELECT lp, lp_len, t_infomask & 1 AS has_null, t_hoff
FROM heap_page_items(get_raw_page('t',0));
```
```
 lp | lp_len | has_null | t_hoff
----+--------+----------+--------
  1 |     38 |        0 |     24
  2 |     32 |        1 |     24     ← bitmap present, still hoff=24 (fits in padding)
```

**PROVE IT #3 — alignment padding, directly.**
```sql
SELECT pg_column_size(row(true, 1::bigint));    -- 40  (24 hdr + 1 + 7 pad + 8)
SELECT pg_column_size(row(1::bigint, true));    -- 33  (24 hdr + 8 + 1)
```

**PROVE IT #4 — the varlena header switches at 126 bytes.**
```sql
SELECT pg_column_size(repeat('a', 126)),   -- 127  (1-byte header)
       pg_column_size(repeat('a', 127));   -- 131  (4-byte header) — 4 bytes for 1 char
```

**PROVE IT #5 — watch TOAST engage.**
```sql
CREATE TABLE big (id int, body text);
INSERT INTO big VALUES (1, repeat('a', 100));
INSERT INTO big VALUES (2, repeat('a', 100000));
SELECT reltoastrelid::regclass FROM pg_class WHERE relname='big';   -- pg_toast.pg_toast_XXXXX
SELECT id, pg_column_size(body) FROM big;
```
```
 id | pg_column_size
----+----------------
  1 |            104
  2 |            206      ← 100,000 chars compressed to 206 bytes (all 'a')
```
```sql
-- defeat compression with random data:
INSERT INTO big VALUES (3, (SELECT string_agg(md5(random()::text),'') FROM generate_series(1,200)));
SELECT id, pg_column_size(body) FROM big WHERE id=3;   -- ~6400, stored OUT OF LINE
SELECT count(*) FROM pg_toast.pg_toast_16xxx;          -- chunks exist
```

**PROVE IT #6 — inline vs external, in rows-per-page.**
```sql
CREATE TABLE inline_pay  (id bigint, payload text);
CREATE TABLE ext_pay     (id bigint, payload text);
ALTER TABLE ext_pay ALTER COLUMN payload SET STORAGE EXTERNAL;

INSERT INTO inline_pay SELECT i, (SELECT string_agg(md5(random()::text),'')
                                  FROM generate_series(1,44)) FROM generate_series(1,20000) i;
INSERT INTO ext_pay    SELECT * FROM inline_pay;

SELECT relname, relpages, round(reltuples/nullif(relpages,0)) AS rows_per_page,
       pg_size_pretty(pg_relation_size(oid)) AS main_only
FROM pg_class WHERE relname IN ('inline_pay','ext_pay');
```
```
  relname   | relpages | rows_per_page | main_only
------------+----------+---------------+-----------
 inline_pay |     5000 |             4 | 39 MB
 ext_pay    |      147 |           136 | 1176 kB    ← 34× fewer pages to scan
```

---

## The design decision framework

```
ORDER COLUMNS BY DESCENDING ALIGNMENT WHEN:
  ✓ The table will exceed ~10 GB or ~50M rows
  ✓ The table is frequently scanned or range-scanned
  ✓ You are creating it now (it's free at CREATE TABLE time)
  Order: 8-byte (bigint, timestamptz, double) → 4-byte (int, date, real)
       → 2-byte (smallint) → 1-byte (bool, "char") → variable (text, jsonb, uuid*)
  *uuid is 1-byte-aligned in PG, so it can go anywhere; put it with the varlenas.

DON'T BOTHER WHEN:
  ✗ The table is small and fully cached
  ✗ It would require a rewrite of a live 500 GB table for a 5% win
  ✗ Readability of the DDL matters more than 5% of a 200 MB table

FORCE STORAGE EXTERNAL WHEN:
  ✓ A varlena column is typically 500 B – 2 KB (the "dead zone")
  ✓ That column is rarely in the SELECT list
  ✓ You use substring()/length() on it (EXTERNAL allows partial fetch;
    compressed values must be fully decompressed)

MOVE THE COLUMN TO A 1:1 SIDE TABLE INSTEAD WHEN:
  ✓ The column is read by < 10% of queries AND
  ✓ The parent row is updated frequently (so you also stop rewriting it) AND
  ✓ You want the win to apply to indexes and WAL too, not just page density

USE bigint-in-smallest-unit FOR MONEY, ALWAYS:
  ✓ 8 bytes, exact, hardware arithmetic, sortable, indexable
  ✗ never float (rounding), rarely numeric (slow + variable width)

THE SIGNAL TO LOOK FOR:
      SELECT relname, round(reltuples/nullif(relpages,0)) AS rows_per_page
      FROM pg_class WHERE relkind='r' AND relpages > 1000
      ORDER BY 2 ASC LIMIT 20;
  Any large table with rows_per_page < 20 has a storage-layout problem.
  Check for (a) a varlena column in the 1–2 KB dead zone, (b) bad column
  order, (c) bloat. In that order — the first is usually the big one.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Given `CREATE TABLE s (a bool, b timestamptz, c int, d bigint, e smallint);`, compute by hand: the offset of each column, the total padding wasted, and `pg_column_size(row(...))`. Then verify with `pg_column_size`. Then write the optimal column order and compute the saving per row and per 100M rows.

### Exercise 2 — medium (apply it)
Build two tables holding 50,000 rows with an 800-byte random (incompressible) text column — one default, one `SET STORAGE EXTERNAL`. Measure: main relation size, TOAST size, total size, rows-per-page, and the time for `SELECT count(*)` and `SELECT length(payload) FROM ... LIMIT 1000`. Explain each difference. Then state the exact condition under which EXTERNAL is the *wrong* choice.

### Exercise 3 — hard (production simulation)
A `messages` table has 2 billion rows, 1.1 TB, and this definition:

```sql
CREATE TABLE messages (
  is_read     boolean NOT NULL DEFAULT false,
  id          bigserial PRIMARY KEY,
  is_deleted  boolean NOT NULL DEFAULT false,
  conversation_id bigint NOT NULL,
  sender_id   bigint NOT NULL,
  priority    smallint NOT NULL DEFAULT 0,
  sent_at     timestamptz NOT NULL,
  body        text NOT NULL,             -- avg 900 bytes
  metadata    jsonb                       -- avg 300 bytes
);
```
The hot query is `SELECT id, sender_id, sent_at FROM messages WHERE conversation_id = $1 ORDER BY sent_at DESC LIMIT 50`, called 40,000 times/sec. `is_read` is updated constantly.

(a) Compute the current per-tuple size including padding, and the optimal one.
(b) Identify every reason this table's rows-per-page is bad, ranked by impact.
(c) Design the fix. You may change the schema, storage parameters, indexes, and split tables. Justify each change with the mechanism it exploits.
(d) `is_read` is updated on nearly every message. Explain at the tuple level what one `UPDATE messages SET is_read=true WHERE id=$1` currently costs, and how much your design reduces it. Name the specific mechanism.
(e) Which of your changes require a full table rewrite, and how would you deploy them on a live 1.1 TB table?

---

## Mental model checkpoint

1. Draw the tuple header from memory. Which three fields belong to MVCC, and what does each do?
2. Why does declaring `bool, bigint` cost 8 more bytes per row than `bigint, bool`?
3. Is a NULL column more or less expensive than a stored value? Explain via the NULL bitmap.
4. At what value size does TOAST engage, and what are the two things it tries in order?
5. Why is a 1,500-byte text column often worse for performance than a 15,000-byte one?
6. What is `t_ctid` normally, and what does it become after an UPDATE?
7. `SELECT id FROM docs` vs `SELECT *` — when does the difference actually matter, and why? (Two different answers: heap scan vs TOAST.)

---

## Quick reference card

| Component | Size |
|---|---|
| Tuple header | 23 B → 24 with alignment |
| Line pointer (in page) | 4 B |
| Minimum row cost | **28 bytes** before any data |
| NULL bitmap | ceil(ncols/8) B, usually free (fits in padding) |
| varlena header | 1 B if value ≤ 126 B, else 4 B |
| TOAST pointer | 18 B |
| TOAST threshold | ~2000 B (¼ page) |
| TOAST chunk | ~2000 B |
| Max value | 1 GB |
| Max inline tuple | ~8160 B |

**Alignment cheat sheet**

| Align | Types |
|---|---|
| 8 | `bigint`, `double precision`, `timestamp`, `timestamptz`, `money`, `interval`(part) |
| 4 | `int`, `real`, `date`, `numeric`, `text`, `varchar`, `jsonb`, `bytea` |
| 2 | `smallint` |
| 1 | `bool`, `"char"`, `uuid` |

**Storage strategies**

| Strategy | Compress | Out of line | Use for |
|---|---|---|---|
| `PLAIN` | no | no | fixed-width types (forced) |
| `EXTENDED` | yes | yes | default for varlena |
| `EXTERNAL` | no | yes | dead-zone columns; substring access |
| `MAIN` | yes | last resort | small-ish, always-read columns |

**The three rules**
1. 8-byte types first, varlena last.
2. Values in the 1–2 KB dead zone → `EXTERNAL` or a side table.
3. Money = `bigint` in paise. Never float.

---

## When would I use this at work?

1. **Designing a table you know will hit 500M rows.** Ordering columns correctly at `CREATE TABLE` time costs zero effort and saves 20% of storage, RAM, and scan time for the life of the system. Doing it later costs a maintenance window.

2. **A table with terrible scan performance and no obvious cause.** `rows_per_page` of 4 immediately tells you there's a fat inline column. You find the 1.4 KB JSON payload, move it out of line, and the scan gets 30× faster without touching a query.

3. **Explaining write amplification in a design review.** Someone wants a `view_count` column on a 2 KB-wide `articles` table, incremented on every page view. You can show that each increment rewrites 2 KB of tuple + 2 KB of WAL + every index entry, and propose a separate counters table or Redis write-behind — with the byte math to back it up (Topic 61).

---

## Connected topics

**Understand before this:** 04 (pages, line pointers, TIDs).

**This unlocks:**
- **06** — row store vs column store: why *this* layout is bad for analytics
- **17** — write amplification: the cost of rewriting a whole tuple
- **25** — choosing data types, now that you know what each one costs
- **26** — money as bigint, timestamps as timestamptz
- **46** — MVCC: `t_xmin`, `t_xmax` and `t_ctid` are the whole mechanism
- **47** — VACUUM: reclaiming the dead tuples this layout creates
