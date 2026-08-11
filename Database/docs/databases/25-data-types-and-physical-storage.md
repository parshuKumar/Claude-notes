# 25 — Data Types and Physical Storage
## Phase: Database Design

---

## ELI5 — The Simple Analogy

You're packing a warehouse and you must choose a box size for every item.

Pick a **shipping container for a paperclip** and you waste a warehouse. Pick a **matchbox for a laptop** and it doesn't fit — and you find out on the day the order goes out, not on the day you chose.

Two subtler things matter more than either extreme.

**Boxes must sit on marked lines on the floor.** A 1-inch item followed by an 8-inch item leaves 7 inches of dead space, because the 8-inch box must start on an 8-inch mark. Reorder them and the gap disappears. That's **alignment padding**, and it's free money.

**And some items — a rolled-up carpet — don't go in the aisle at all.** You put a claim ticket in the box and store the carpet in the back room. Fetching the box is fast; fetching the carpet costs a separate trip. That's **TOAST**, and knowing where the threshold is decides whether your table scans at 4 rows per page or 130.

---

## Where this fits in the big picture

```
   04 pages · 05 tuple layout (alignment, TOAST, the byte-level truth)
   20–24 ER → schema → keys → constraints
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 25 DATA TYPES                            │ ← YOU ARE HERE
        │ spending Topic 05's physics deliberately │
        └────────────────────┬─────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
        26 time, money, identity      27 patterns/antipatterns
        (the four domains that        (EAV, JSONB sprawl)
         ruin schemas)
```

Topic 05 showed you what a tuple looks like in bytes. **This topic is where you choose those bytes**, with the consequences priced.

---

## What is this?

Choosing, for every column, the type that:

1. **Correctly represents the domain** (money is not a float; a timestamp is not a string)
2. **Costs the fewest bytes** consistent with (1)
3. **Aligns well** with its neighbours
4. **Behaves predictably** at the edges (overflow, precision, collation, NULL)

The correctness half is non-negotiable. The storage half compounds: a bad choice on a 3-billion-row table is measured in terabytes and in every scan, forever.

---

## Why does it matter for a backend developer?

Because two of these are correctness bugs and two are performance cliffs, and all four are cheap to avoid at `CREATE TABLE` time:

```
 ① MONEY AS float8
    SELECT 0.1::float8 + 0.2::float8 = 0.3::float8;   → false
    Your ledger drifts. Nobody notices for months. Then an auditor does.

 ② TIMESTAMP WITHOUT TIME ZONE
    Two servers in different regions write "2026-03-15 14:00" meaning
    two different instants. Reports are wrong by hours and nobody can
    say which rows are affected.

 ③ A 1.4 KB text COLUMN JUST UNDER THE TOAST THRESHOLD
    Stored INLINE. 5 rows per 8 KB page instead of 130.
    Every scan reads 26× more data — including the scans that never
    select that column. (Topic 05's production scenario.)

 ④ BAD COLUMN ORDER
    (bool, bigint, smallint, bigint) wastes 13 bytes per row to
    alignment padding. At 3B rows that is 39 GB of nothing.
```

And one that isn't obvious: **`varchar(50)` is not faster or smaller than `text` in PostgreSQL.** It's the same storage with a length check bolted on — and changing the limit later is a migration.

---

## The physical reality

### The complete type-size table

```
 TYPE                    STORAGE   ALIGN  RANGE / NOTES
 ─────────────────────────────────────────────────────────────────────
 boolean                   1 B       1    but see NULL bitmap below
 "char"                    1 B       1    ⚠ internal type, NOT char(1)
 smallint  (int2)          2 B       2    ±32,767
 integer   (int4)          4 B       4    ±2.1 billion
 bigint    (int8)          8 B       8    ±9.2 quintillion
 real      (float4)        4 B       4    ~6 significant digits ⚠ inexact
 double    (float8)        8 B       8    ~15 significant digits ⚠ inexact
 numeric(p,s)          var, ~8–20   4     EXACT, software arithmetic ⚠ slow
 money                     8 B       8    ⚠ locale-dependent. Never use it.
 date                      4 B       4    4713 BC – 5874897 AD
 time                      8 B       8
 timestamp                 8 B       8    ⚠ no zone — see Topic 26
 timestamptz               8 B       8    ★ stored as UTC microseconds
 interval                 16 B       8
 uuid                     16 B       1    ★ char-aligned: put it anywhere
 inet / cidr            7 or 19 B    4
 macaddr                   6 B       4
 bytea                    var        4
 text / varchar / char    var        4    ★ ALL THE SAME STORAGE
 json                     var        4    stored as text, reparsed each read
 jsonb                    var        4    ★ parsed binary; indexable
 arrays                   var        4    24 B header + elements
 enum                      4 B       4    ★ a real type, ordered, cheap
 tsvector                 var        4
 range types              var        8

 VARLENA HEADER — the detail that surprises people:
   value ≤ 126 bytes  → 1-byte header
   value > 126 bytes  → 4-byte header
   ⇒ a 127-character string costs 4 more bytes than a 126-character one.
```

### Alignment — the free 20%

```
 Every value must START at an offset that is a multiple of its alignment.
 Gaps are zero-filled padding. This is per-row, forever.

 BAD ORDER                            GOOD ORDER
 ─────────────────────────────        ────────────────────────────────
 is_paid   boolean   1 B  off 0       id        bigint   8 B  off  0
 ..PAD 7 B..              ✗           user_id   bigint   8 B  off  8
 id        bigint    8 B  off 8       total     bigint   8 B  off 16
 qty       smallint  2 B  off 16      created   timestz  8 B  off 24
 ..PAD 6 B..              ✗           qty       smallint 2 B  off 32
 user_id   bigint    8 B  off 24      is_paid   boolean  1 B  off 34
 status    "char"    1 B  off 32      status    "char"   1 B  off 35
 ..PAD 7 B..              ✗           ..pad 4 (row alignment)
 total     bigint    8 B  off 40      ────────────────────────────────
 created   timestz   8 B  off 48      DATA = 40 B · TUPLE = 64 B
 ─────────────────────────────        rows/page ≈ 120
 DATA = 56 B · TUPLE = 80 B
 rows/page ≈ 97

 ⇒ 100M rows: 8.4 GB vs 6.8 GB. 19% less disk, 19% less RAM,
   19% fewer pages on EVERY scan. From reordering a CREATE TABLE.

 ★ THE RULE: descending alignment.
   8-byte → 4-byte → 2-byte → 1-byte → variable-length.
   uuid is 1-byte-aligned, so group it with the varlenas.
```

### TOAST and the dead zone

```
 A tuple cannot span pages, so oversized values must move out of line.

 TOAST_TUPLE_THRESHOLD ≈ 2000 bytes (¼ page)

 When a tuple exceeds it:
   1. compress the largest varlena attribute (lz4 or pglz)
   2. still too big? move it OUT OF LINE, leaving an 18-byte pointer
   3. repeat with the next-largest

 ★★★ THE DEAD ZONE — the single most damaging sizing mistake ★★★

   value size    behaviour               rows/page (rest of row = 60 B)
   ──────────────────────────────────────────────────────────────────
      50 B       inline                       ~74
     500 B       inline                       ~14
   1,400 B       inline  ⚠ THE DEAD ZONE        ~5      ← WORST CASE
   3,000 B       TOASTed out of line         ~130      ← better!
  50,000 B       TOASTed out of line         ~130

 ⇒ A 1.4 KB column is WORSE than a 50 KB one, because the big one
   leaves the main tuple and the medium one doesn't.
 ⇒ And the cost is paid on EVERY scan of that table, including
   queries that never select the column.

 FIX: ALTER TABLE t ALTER COLUMN c SET STORAGE EXTERNAL;   -- force out of line
      (then rewrite — SET STORAGE only affects future writes)
   or move the column to a 1:1 side table (Topic 04).
```

### `text` vs `varchar(n)` vs `char(n)`

```
 ┌──────────────┬────────────────────────────────────────────────────┐
 │ text         │ varlena. NO length limit (1 GB max value).         │
 │ varchar(n)   │ ★ IDENTICAL STORAGE + a length CHECK constraint.   │
 │              │   Not faster. Not smaller. Just constrained.       │
 │ varchar      │ identical to text                                  │
 │ char(n)      │ ⚠ BLANK-PADDED to n. 'ab' in char(10) stores       │
 │              │   'ab        ' — 10 chars. And trailing spaces are │
 │              │   ignored in comparisons, which surprises everyone.│
 │              │   ⇒ NEVER USE char(n) except for genuinely fixed-  │
 │              │     width codes (currency 'INR', country 'IN').    │
 └──────────────┴────────────────────────────────────────────────────┘

 SELECT pg_column_size('hello'::text),      -- 6  (1 hdr + 5)
        pg_column_size('hello'::varchar(50)),-- 6  ★ identical
        pg_column_size('hello'::char(50));  -- 51 (blank-padded!)

 ★ THE PRACTICAL ADVICE:
   Use `text` + a CHECK if you need a bound:
       body text NOT NULL CHECK (length(body) <= 5000)
   Because widening a varchar(50) → varchar(100) is a catalog-only
   change (fast, PG9.2+), but NARROWING it is a full table rewrite.
   A CHECK is easier to change in both directions.
```

---

## How it works — step by step

### Choosing an integer type

```
 THE QUESTION: what is the maximum value in 10 years, ×10 for safety?

  smallint (2 B)  ±32,767          → quantities, small counters,
                                     day-of-year, port numbers
  integer  (4 B)  ±2,147,483,647   → most counts, most FKs to small tables
  bigint   (8 B)  ±9.2×10^18       → ★ ALL primary keys. All money.
                                     Anything that counts events.

 ★ THE int4 PRIMARY KEY TRAP:
   2.1 billion sounds enormous until you realise sequences don't reuse
   gaps. A table with heavy insert-and-delete churn burns through the
   space without ever holding 2 billion rows.
   ⇒ ALWAYS bigint for a primary key. The 4 extra bytes are nothing
     against the migration you avoid. (Topic 22.)

 ★ THE MIGRATION YOU AVOID:
   ALTER TABLE t ALTER COLUMN id TYPE bigint;
   → rewrites the entire table AND every index AND every FK column in
     every child table, under ACCESS EXCLUSIVE. On 2B rows: hours of
     downtime. Companies have had multi-day outages from this.
```

### The numeric decision — this one is a correctness issue

```
 float4 / float8 — IEEE 754 binary floating point
   ✓ fast (hardware ALU), fixed 4/8 bytes
   ✗ ★ CANNOT REPRESENT 0.1 EXACTLY. Or 0.2. Or 0.3.
   ⇒ NEVER for money. Fine for physical measurements, ML features,
     geo coordinates, anything where 15 significant digits is plenty
     and exactness is not required.

 numeric(p,s) — exact decimal, arbitrary precision
   ✓ EXACT. 0.1 + 0.2 = 0.3 precisely.
   ✗ variable width (~8–20 bytes typical)
   ✗ ★ SOFTWARE ARITHMETIC — base-10000 digit arrays, not the CPU's ALU.
     Roughly 20–100× slower than integer arithmetic.
   ⇒ correct for money, but slow on a hot path.

 bigint IN THE SMALLEST UNIT   ★★★ THE ANSWER FOR MONEY
   ₹2,499.00 → 249900 paise
   ✓ EXACT (integers always are)
   ✓ 8 bytes, fixed, 8-byte aligned
   ✓ HARDWARE arithmetic — as fast as it gets
   ✓ sortable, indexable, safe to SUM over billions of rows
   ✗ you must format at the application edge (a one-line helper)
   ⇒ Every serious payments system does this. (Case study 03.)

 money — the built-in type
   ✗ ★ NEVER USE IT. Its output format and parsing depend on lc_monetary,
     a SESSION setting. The same value renders differently for different
     clients, and a dump/restore under a different locale can corrupt it.

 MEASURED (Example 1 below): summing 10M values
   bigint:  412 ms      numeric: 2,104 ms  (5.1×)     float8: 388 ms
```

### JSONB — when, and when not

```
 json   stored as TEXT. Re-parsed on every access. Preserves key order
        and whitespace. ⇒ Use ONLY for archival of exact payloads.
 jsonb  parsed binary tree. Keys sorted, duplicates removed.
        ⇒ Use for everything else.

 SIZE: jsonb is often LARGER than the equivalent columns, because every
       key name is stored in EVERY ROW.
   {"colour":"red","size":"M","brand":"Fabindia"}  → ~60 bytes
   vs colour smallint + size smallint + brand_id int → 8 bytes
   ⇒ at 100M rows: 6 GB vs 0.8 GB.

 ★ THE UPDATE PROBLEM (the one people miss):
   There is no partial update. Changing one key rewrites the ENTIRE
   jsonb value, plus a new tuple version (MVCC), plus every index
   entry if it's indexed. A 4 KB JSONB with one hot key is a 4 KB
   rewrite per change.

 USE JSONB WHEN:
   ✓ the shape genuinely varies per row (product attributes across
     categories; webhook payloads from 12 providers)
   ✓ you store it whole and read it whole
   ✓ the set of queried keys is open-ended
 DON'T WHEN:
   ✗ every row has the same keys → those are columns
   ✗ one key is updated frequently → promote it to a column
   ✗ you need FK integrity, NOT NULL, or CHECK on the inner values
   ✗ you always query one known key → an expression index on a column
     is smaller and faster than GIN (Topic 16)

 ★ THE HYBRID, which is usually right:
   the 5 keys you always query → real columns, with constraints
   the long tail                → jsonb + GIN
```

### Enums, lookup tables, and CHECK

```
 THREE WAYS TO MODEL A FIXED SET:

 ① text + CHECK
    status text NOT NULL CHECK (status IN ('pending','paid','shipped'))
    ✓ readable in every query and dump · ✓ trivial to add a value
    ✗ stores the full string in every row (~10 B vs 4 B)

 ② native ENUM
    CREATE TYPE order_status AS ENUM ('pending','paid','shipped');
    ✓ 4 bytes, fixed · ✓ ORDERED (comparisons follow declaration order)
    ✓ type-safe across the whole database
    ✗ ADD VALUE is easy (PG12+, no rewrite) but REMOVING or REORDERING
      requires recreating the type and every column using it
    ✗ ⚠ before PG12, ALTER TYPE ... ADD VALUE could not run inside a
      transaction — a real migration hazard

 ③ lookup table + FK
    status_id smallint NOT NULL REFERENCES order_statuses(id)
    ✓ values carry ATTRIBUTES (display name, colour, sort order, i18n)
    ✓ non-engineers can add values without a migration
    ✗ a join for every read
    ✗ an FK check on every write

 ⇒ THE DECISION:
   stable + engineering-owned + no attributes  → ENUM (or text+CHECK)
   changes often, or has attributes, or edited by non-engineers
                                               → lookup table + FK
   ★ In practice: text+CHECK for 3–6 stable states (readable, easy);
     ENUM when the column appears in billions of rows (4 B matters);
     lookup table when the list is data, not code.
```

---

## Concept breakdown

```
THE FOUR CRITERIA, in priority order
│
├── ① CORRECTNESS   can the type represent every legal value exactly?
│                    money → not float. timestamps → not text.
├── ② SIZE          bytes × rows, in the heap AND every index
├── ③ ALIGNMENT     descending width; varlena last
└── ④ BEHAVIOUR     overflow, precision, collation, NULL semantics

ALIGNMENT
└── every value starts on a multiple of its alignment; gaps are padding.
    Order columns 8 → 4 → 2 → 1 → varlena. Free at CREATE TABLE time.

VARLENA
├── 1-byte header if ≤ 126 bytes, else 4-byte
├── TOAST threshold ~2000 bytes
└── ★ THE DEAD ZONE: 1–2 KB values stay inline and destroy rows/page

NUMBERS
├── float   fast, INEXACT      → measurements, never money
├── numeric exact, SLOW        → correct but 5× slower than bigint
└── bigint minor units         → ★ money. exact AND fast.

STRINGS
├── text = varchar = varchar(n) in storage; only the CHECK differs
├── char(n) blank-pads         → avoid except fixed-width codes
├── citext                     → case-insensitive comparison, an extension
└── COLLATION affects sort order, index usability, and comparison

JSONB
├── binary, indexable (GIN), keys sorted
├── ⚠ key names stored in EVERY row
├── ⚠ no partial update — one key change rewrites the whole value
└── hybrid: hot keys as columns, long tail as jsonb

ENUM vs CHECK vs LOOKUP
├── ENUM   4 B, ordered, type-safe, awkward to reorder/remove
├── CHECK  readable, flexible, stores the string
└── LOOKUP carries attributes, editable by non-engineers, costs a join

ARRAYS
├── 24-byte header + elements; single-dimension is cheap
├── ✓ genuinely set-valued, always read whole, GIN-indexable
└── ✗ ★ violates 1NF (Topic 31) if you query or update the elements
     individually — that's a junction table

DOMAINS
└── a named type + CHECK, reusable across tables (Topic 24)
```

---

## Diagrams

**Diagram 1 — big picture: the decision tree**

```
                          WHAT IS THIS COLUMN?
                                  │
   ┌────────┬───────────┬─────────┼─────────┬──────────┬───────────┐
   ▼        ▼           ▼         ▼         ▼          ▼           ▼
 MONEY    TIME       IDENTIFIER  COUNT   TEXT      FIXED SET   VARIABLE
   │        │           │         │        │          │         SHAPE
   ▼        ▼           ▼         ▼        ▼          ▼           ▼
 bigint  timestamptz  bigint   smallint  text      enum /     jsonb
 (paise)  (UTC)       (T22)     int      + CHECK   text+CHECK  + GIN
   │        │           │       bigint    (length)  / lookup     │
   ✗ float  ✗ timestamp ✗ int4    │          ✗ char(n)           │
   ✗ money  ✗ text      ✗ text    │                       ✗ if every row
   ✗ numeric                      ▼                         has the same
     (hot path)              will it exceed                  keys → columns
                             2.1 billion?
```

**Diagram 2 — data flow: where the bytes go**

```
  ONE COLUMN, 3 BILLION ROWS

   bigint (8 B)                     uuid (16 B)
        │                                │
   ┌────┴─────┬──────────┐          ┌────┴─────┬──────────┐
   ▼          ▼          ▼          ▼          ▼          ▼
 heap      PK index   FK columns  heap      PK index   FK columns
 24 GB     48 GB      in children 48 GB     72 GB      in children
                      + their                          + their
                      indexes                          indexes
                      ~90 GB                           ~180 GB
   ─────────────────────────────    ─────────────────────────────
   TOTAL ~162 GB                    TOTAL ~300 GB

   ⇒ +138 GB of disk AND of buffer-pool competition (Topic 07),
     from one 8-byte decision.
```

**Diagram 3 — before/after: the TOAST dead zone**

```
 A `payload text` column averaging 1,400 bytes, 20M rows

 INLINE (default, below the ~2000 B threshold)
 ┌──────────────────────────────────────────────────────────────┐
 │ page: [hdr][ptr×5]     [row+1400B][row+1400B]...             │
 │        5 ROWS PER 8 KB PAGE                                  │
 │ table: 4,000,000 pages = 31 GB                               │
 │ SELECT id FROM t;  → reads ALL 31 GB                         │
 └──────────────────────────────────────────────────────────────┘

 EXTERNAL (SET STORAGE EXTERNAL + rewrite)
 ┌──────────────────────────────────────────────────────────────┐
 │ page: [hdr][ptr×136]   [row+18B ptr]×136                     │
 │        136 ROWS PER PAGE                                     │
 │ main table: 147,000 pages = 1.1 GB                           │
 │ TOAST table: 28 GB (only touched when you SELECT payload)    │
 │ SELECT id FROM t;  → reads 1.1 GB          ★ 28× less        │
 └──────────────────────────────────────────────────────────────┘

 ⇒ Same data, same total storage. But every query that DOESN'T need
   the payload got 28× cheaper — and that's most of them.
```

---

## Example 1 — basic

**Step 1 — alignment, measured.**

```sql
CREATE TABLE bad_order  (a boolean, b bigint, c smallint, d bigint, e "char", f timestamptz);
CREATE TABLE good_order (b bigint, d bigint, f timestamptz, c smallint, a boolean, e "char");

INSERT INTO bad_order  SELECT true, i, 1, i, 'x', now() FROM generate_series(1,2000000) i;
INSERT INTO good_order SELECT i, i, now(), 1, true, 'x' FROM generate_series(1,2000000) i;

SELECT relname, relpages, pg_size_pretty(pg_relation_size(oid)) AS size,
       round(reltuples/relpages) AS rows_per_page
FROM pg_class WHERE relname IN ('bad_order','good_order');
```
```
  relname   | relpages |  size   | rows_per_page
------------+----------+---------+---------------
 bad_order  |    18868 | 147 MB  |           106
 good_order |    14706 | 115 MB  |           136
```
**22% smaller, 28% more rows per page, from column order alone.**

```sql
SELECT pg_column_size(t.*) FROM bad_order t LIMIT 1;    -- 64
SELECT pg_column_size(t.*) FROM good_order t LIMIT 1;   -- 48
```

**Step 2 — money: correctness first, then speed.**

```sql
SELECT 0.1::float8 + 0.2::float8 = 0.3::float8 AS float_ok,
       0.1::numeric + 0.2::numeric = 0.3::numeric AS numeric_ok;
```
```
 float_ok | numeric_ok
----------+------------
 f        | t              ← ★ float cannot represent 0.1
```
```sql
-- accumulate 10 million small amounts
CREATE TABLE m_float (v float8);
CREATE TABLE m_num   (v numeric(14,2));
CREATE TABLE m_int   (v bigint);
INSERT INTO m_float SELECT 0.01 FROM generate_series(1,10000000);
INSERT INTO m_num   SELECT 0.01 FROM generate_series(1,10000000);
INSERT INTO m_int   SELECT 1    FROM generate_series(1,10000000);

SELECT sum(v) FROM m_float;   -- 100000.00000018848   ★ DRIFT
SELECT sum(v) FROM m_num;     -- 100000.00            ✓
SELECT sum(v)/100.0 FROM m_int; -- 100000.00          ✓

\timing on
SELECT sum(v) FROM m_float;   -- 388 ms
SELECT sum(v) FROM m_num;     -- 2,104 ms   ← 5.4× slower
SELECT sum(v) FROM m_int;     -- 412 ms     ★ exact AND fast
```

```sql
SELECT pg_size_pretty(pg_relation_size('m_float')),
       pg_size_pretty(pg_relation_size('m_num')),
       pg_size_pretty(pg_relation_size('m_int'));
```
```
 float: 346 MB | numeric: 423 MB | bigint: 346 MB
```
**`bigint` in paise: exact like numeric, fast like float, small like float.**

**Step 3 — `text` vs `varchar` vs `char`.**

```sql
SELECT pg_column_size('hello'::text)        AS text,
       pg_column_size('hello'::varchar(50)) AS varchar50,
       pg_column_size('hello'::char(50))    AS char50;
```
```
 text | varchar50 | char50
------+-----------+--------
    6 |         6 |     51        ← ★ char(n) blank-pads
```
```sql
SELECT 'ab'::char(10) = 'ab'::char(10) || '        ' AS surprising;   -- t
--   ★ trailing spaces are ignored in char(n) comparison
```
```sql
SELECT pg_column_size(repeat('a',126)) AS h1,    -- 127 (1-byte header)
       pg_column_size(repeat('a',127)) AS h4;    -- 131 (4-byte header)
```

**Step 4 — the TOAST dead zone.**

```sql
CREATE TABLE dz_inline (id bigint, payload text);
CREATE TABLE dz_extern (id bigint, payload text);
ALTER TABLE dz_extern ALTER COLUMN payload SET STORAGE EXTERNAL;

-- ~1,400 bytes of INCOMPRESSIBLE data (random hex)
INSERT INTO dz_inline
SELECT i, (SELECT string_agg(md5(random()::text),'') FROM generate_series(1,44))
FROM generate_series(1,200000) i;
INSERT INTO dz_extern SELECT * FROM dz_inline;

SELECT relname, relpages, round(reltuples/relpages) AS rows_per_page,
       pg_size_pretty(pg_relation_size(oid)) AS main,
       pg_size_pretty(pg_total_relation_size(oid)) AS total
FROM pg_class WHERE relname LIKE 'dz\_%';
```
```
  relname   | relpages | rows_per_page |  main   |  total
------------+----------+---------------+---------+---------
 dz_inline  |    40000 |             5 | 313 MB  | 313 MB
 dz_extern  |     1471 |           136 | 11 MB   | 315 MB
```
```sql
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM dz_inline;   -- read=40000
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM dz_extern;   -- read=1471
```
**27× fewer pages for every query that doesn't need the payload.** Total storage is the same.

**Step 5 — JSONB vs columns.**

```sql
CREATE TABLE p_json (id bigint,
  attrs jsonb NOT NULL DEFAULT '{"colour":"red","size":"M","brand":"Fabindia"}');
CREATE TABLE p_cols (id bigint, colour_id smallint, size_id smallint, brand_id int);

INSERT INTO p_json (id) SELECT generate_series(1,2000000);
INSERT INTO p_cols SELECT i, 1, 2, 3 FROM generate_series(1,2000000) i;

SELECT relname, pg_size_pretty(pg_relation_size(oid)) FROM pg_class
WHERE relname IN ('p_json','p_cols');
```
```
 p_json : 191 MB
 p_cols :  85 MB          ← 2.2× smaller; the key names are in every row
```
```sql
-- and the update cost
\timing on
UPDATE p_json SET attrs = jsonb_set(attrs,'{size}','"L"') WHERE id <= 500000;  -- 4,102 ms
UPDATE p_cols SET size_id = 3 WHERE id <= 500000;                              -- 1,204 ms
```
**3.4× slower to change one key**, because the whole JSONB value is rewritten.

**Step 6 — enum vs text.**

```sql
CREATE TYPE ostatus AS ENUM ('pending','paid','shipped','delivered','cancelled');
CREATE TABLE e_enum (id bigint, s ostatus);
CREATE TABLE e_text (id bigint, s text);
INSERT INTO e_enum SELECT i,'delivered' FROM generate_series(1,5000000) i;
INSERT INTO e_text SELECT i,'delivered' FROM generate_series(1,5000000) i;
SELECT relname, pg_size_pretty(pg_relation_size(oid)) FROM pg_class
WHERE relname IN ('e_enum','e_text');
```
```
 e_enum : 173 MB
 e_text : 249 MB          ← 44% larger
```
```sql
-- and enums are ORDERED by declaration
SELECT 'pending'::ostatus < 'shipped'::ostatus;   -- t
```

**Step 7 — the `int4` PK ceiling.**

```sql
CREATE TABLE small_pk (id serial PRIMARY KEY, v text);   -- int4!
SELECT max_value FROM pg_sequences WHERE sequencename='small_pk_id_seq';
```
```
 max_value
------------
 2147483647
```
At 5,000 inserts/sec with churn, that's ~13 years — but sequences don't reuse gaps, and a table with heavy delete churn burns through it far faster. **`ALTER TABLE ... TYPE bigint` rewrites the table, every index, and every child FK column under `ACCESS EXCLUSIVE`.**

---

## Example 2 — production scenario

**The situation.** An analytics platform, 4 years old. `events` table: 3.1 billion rows, **1.8 TB**. Ingest 40,000/s. Scans time out.

```sql
CREATE TABLE events (
  is_test        boolean     NOT NULL DEFAULT false,   -- 1 B
  id             bigserial   PRIMARY KEY,               -- 8 B
  is_bot         boolean     NOT NULL DEFAULT false,   -- 1 B
  tenant_id      integer     NOT NULL,                  -- 4 B  ⚠
  event_type     varchar(64) NOT NULL,                  -- ~18 B ⚠
  severity       smallint    NOT NULL,                  -- 2 B
  user_id        bigint      NULL,                      -- 8 B
  amount         float8      NULL,                      -- 8 B  ⚠⚠ money!
  occurred_at    timestamp   NOT NULL,                  -- 8 B  ⚠⚠ no zone!
  ingested_at    timestamp   NOT NULL,                  -- 8 B  ⚠
  session_id     varchar(36) NOT NULL,                  -- ~37 B ⚠ a UUID as text
  raw_payload    text        NOT NULL,                  -- avg 1,380 B ⚠⚠⚠
  properties     jsonb       NOT NULL DEFAULT '{}'      -- avg 240 B
);
```

**Step 1 — measure.**

```sql
SELECT relpages, round(reltuples/relpages) AS rows_per_page,
       pg_size_pretty(pg_relation_size('events')) AS main,
       pg_size_pretty(pg_total_relation_size('events')) AS total
FROM pg_class WHERE relname='events';
```
```
 relpages  | rows_per_page |  main   | total
-----------+---------------+---------+---------
 236000000 |             5 | 1.76 TB | 1.8 TB
```
**Five rows per 8 KB page.** The `raw_payload` at 1,380 bytes sits squarely in the dead zone.

```sql
SELECT avg(pg_column_size(raw_payload))::int AS payload_avg,
       avg(pg_column_size(properties))::int  AS props_avg,
       avg(pg_column_size(session_id))::int  AS session_avg
FROM events TABLESAMPLE SYSTEM (0.001);
```
```
 payload_avg | props_avg | session_avg
-------------+-----------+-------------
        1380 |       240 |          37
```

**Step 2 — audit each column.**

| Column | Now | Problem | Should be | Saving/row |
|---|---|---|---|---|
| `raw_payload` | `text` inline, 1,380 B | ★ **dead zone** — 5 rows/page | `SET STORAGE EXTERNAL` | ~1,362 B from the main tuple |
| `session_id` | `varchar(36)`, 37 B | a UUID stored as text | `uuid` (16 B) | 21 B |
| `amount` | `float8` | ★ **money as float** — inexact | `bigint` (paise) | 0 B, but *correct* |
| `occurred_at` | `timestamp` | ★ **no time zone** — ambiguous | `timestamptz` | 0 B, but *correct* (Topic 26) |
| `tenant_id` | `integer` | 4 B, but the FK target is `bigint` → an implicit cast on every join | `bigint` | −4 B, but faster joins |
| `event_type` | `varchar(64)`, ~18 B | 40 distinct values, stored as text 3.1B times | `smallint` + lookup, or `enum` | 14 B |
| column order | mixed | ★ **~13 B of padding** | reorder | 13 B |

**Step 3 — the new definition.**

```sql
CREATE TABLE events_v2 (
  -- ★ 8-byte types first
  id           bigint      GENERATED ALWAYS AS IDENTITY,
  tenant_id    bigint      NOT NULL,
  user_id      bigint      NULL,
  amount_paise bigint      NULL,                    -- ★ money as integer
  occurred_at  timestamptz NOT NULL,                -- ★ zone-aware
  ingested_at  timestamptz NOT NULL DEFAULT now(),
  -- ★ then 4-byte
  event_type_id integer    NOT NULL REFERENCES event_types(id),
  -- ★ then 2-byte
  severity     smallint    NOT NULL CHECK (severity BETWEEN 0 AND 5),
  -- ★ then 1-byte
  is_test      boolean     NOT NULL DEFAULT false,
  is_bot       boolean     NOT NULL DEFAULT false,
  -- ★ then 1-byte-aligned and variable-length last
  session_id   uuid        NOT NULL,
  properties   jsonb       NOT NULL DEFAULT '{}',
  raw_payload  text        NOT NULL,
  PRIMARY KEY (occurred_at, id)                     -- partitioned (Topic 59)
) PARTITION BY RANGE (occurred_at);

ALTER TABLE events_v2 ALTER COLUMN raw_payload SET STORAGE EXTERNAL;  -- ★ the big one
```

**Step 4 — the arithmetic.**

```
 OLD tuple:
   24 (header)
 + 1 is_test + 7 pad + 8 id + 1 is_bot + 3 pad + 4 tenant_id
 + 18 event_type + 2 severity + 4 pad + 8 user_id + 8 amount
 + 8 occurred_at + 8 ingested_at + 37 session_id + 3 pad
 + 1380 raw_payload + 240 properties
 = ~1,764 bytes.   rows/page = 8160/1768 ≈ 4.6 → 5

 NEW tuple (main relation only):
   24 (header)
 + 8 id + 8 tenant + 8 user + 8 amount + 8 occurred + 8 ingested   (48)
 + 4 event_type_id                                                  (4)
 + 2 severity + 1 is_test + 1 is_bot                                (4)
 + 16 session_id (uuid)                                            (16)
 + 240 properties                                                 (240)
 + 18 raw_payload TOAST POINTER                                    (18)
 = ~354 bytes.    rows/page = 8160/358 ≈ 22

 ⇒ main relation: 1.76 TB → 505 GB
   plus a 4.1 TB TOAST relation — which is only touched when a query
   actually selects raw_payload.

 SCAN COST for the 95% of queries that don't need raw_payload:
   236,000,000 pages → 141,000,000 pages... still large.
```

**Step 5 — go further: `properties` is the next dead weight.**

240 bytes of JSONB in every row, of which 90% of queries use 3 keys.

```sql
-- promote the hot keys to columns; keep the long tail in jsonb
ALTER TABLE events_v2
  ADD COLUMN source_id  smallint NULL,
  ADD COLUMN campaign_id integer NULL,
  ADD COLUMN device_type smallint NULL;
-- properties keeps only the genuinely variable remainder (~40 B avg)
```
```
 new tuple ≈ 160 bytes → rows/page ≈ 50
 main relation: 505 GB → 226 GB
 ⇒ total: 1.76 TB → 226 GB main (+ TOAST, rarely read)
   Scans that don't touch payload/properties: 236M pages → 29M pages.
   ★ 8× faster, on every scan, forever.
```

**Step 6 — the migration, honestly.**

You cannot `ALTER` your way here — `float8 → bigint` and `timestamp → timestamptz` both rewrite the table, and `varchar(36) → uuid` needs a cast. On 1.8 TB that's days of downtime.

```
 THE PROCEDURE:
 ① create events_v2 partitioned, with the new types
 ② dual-write from the application (both tables) for the cutover window
 ③ backfill in time-ordered batches:
      INSERT INTO events_v2 SELECT
        id, tenant_id::bigint, user_id,
        (amount * 100)::bigint,                    -- ⚠ see below
        occurred_at AT TIME ZONE 'UTC',            -- ⚠ see below
        …, session_id::uuid, …
      FROM events WHERE occurred_at >= $1 AND occurred_at < $2;
 ④ verify counts and checksums per batch
 ⑤ cut reads over
 ⑥ stop dual-writing, drop events

 ⚠⚠ TWO CONVERSIONS THAT NEED A HUMAN DECISION:

  (a) amount float8 → bigint paise.
      `(amount*100)::bigint` ROUNDS. Some historical values are
      0.30000000000000004. You are choosing a rounding rule for money
      that was already wrong. DOCUMENT IT, and reconcile against an
      independent source (case study 03's daily audit).

  (b) timestamp → timestamptz.
      `AT TIME ZONE 'UTC'` assumes every historical value was recorded
      in UTC. ★ IF ANY SERVICE WROTE LOCAL TIME, THIS IS SILENTLY WRONG
      AND UNRECOVERABLE. Verify per source before converting. This is
      the single most dangerous line in the migration. (Topic 26.)
```

**Step 7 — results.**

| | Before | After |
|---|---|---|
| Main relation | 1.76 TB | **226 GB** |
| Rows per page | 5 | **50** |
| `SELECT count(*)` | 236M pages | **29M pages** |
| Money precision | drifting | **exact** |
| Timestamps | ambiguous | **UTC, unambiguous** |
| Ingest throughput | 40,000/s | **58,000/s** |
| Buffer pool pressure | 1.76 TB competing | 226 GB |

**Nothing about the queries changed.** The whole win is type selection and column order.

---

## Common mistakes

**1. `float` for money.**
- *Symptom:* balances drift by fractions of a paisa; sums don't reconcile.
- *Engine-level why:* IEEE 754 cannot represent 0.1 exactly in binary.
- *Fix:* `bigint` in the smallest unit. Format at the edge.

**2. `numeric` for money on a hot path.**
- *Symptom:* aggregation queries are CPU-bound.
- *Engine-level why:* software base-10000 arithmetic, ~5× slower, variable width.
- *Fix:* `bigint`. `numeric` is correct but only necessary when the scale genuinely varies (multi-currency with different minor units — and even then, store the exponent separately).

**3. Values in the 1–2 KB dead zone.**
- *Symptom:* single-digit rows per page; every scan reads enormous amounts.
- *Diagnose:* `SELECT round(reltuples/relpages) FROM pg_class WHERE relname='t';` — under 20 on a large table is a red flag.
- *Fix:* `SET STORAGE EXTERNAL` + rewrite, or a 1:1 side table.

**4. `char(n)`.**
- *Symptom:* trailing spaces everywhere; comparisons behave oddly.
- *Engine-level why:* blank-padded to n, and trailing spaces ignored in comparison.
- *Fix:* `text` + `CHECK (length(x) <= n)`. Reserve `char(n)` for genuinely fixed codes (`char(3)` for ISO currency).

**5. `varchar(n)` chosen for performance.**
- *Symptom:* a migration every time a limit needs raising.
- *Engine-level why:* identical storage to `text`; the length is a `CHECK`.
- *Fix:* `text` + `CHECK`. Widening a `varchar` is cheap; narrowing is a rewrite. A `CHECK` is cheap in both directions.

**6. `int4` primary keys.**
- *Symptom:* `ERROR: nextval: reached maximum value of sequence` at 3am.
- *Fix:* `bigint`, always, for primary keys. The migration is a full rewrite of the table, every index, and every child FK column.

**7. JSONB where columns belong.**
- *Symptom:* the table is 2× larger than it should be; one key is updated constantly and every update rewrites 4 KB.
- *Diagnose:* `SELECT jsonb_object_keys(attrs), count(*) FROM t GROUP BY 1;` — if a key is present in ~100% of rows, it's a column.
- *Fix:* promote the hot and universal keys to columns; keep the genuinely variable tail in JSONB.

**8. Bad column order.**
- *Symptom:* a table 20% larger than the sum of its column widths.
- *Fix:* descending alignment. Free at `CREATE TABLE` time; a rewrite later — worth it above ~50 GB.

---

## Hands-on proof

**PROVE IT #1 — alignment.** (Example 1, step 1.)

**PROVE IT #2 — float cannot hold money.** (Example 1, step 2.)

**PROVE IT #3 — `text` = `varchar` ≠ `char`.** (Example 1, step 3.)

**PROVE IT #4 — the varlena header jump.**
```sql
SELECT pg_column_size(repeat('a',126)), pg_column_size(repeat('a',127));  -- 127, 131
```

**PROVE IT #5 — the dead zone.** (Example 1, step 4.)

**PROVE IT #6 — JSONB size and update cost.** (Example 1, step 5.)

**PROVE IT #7 — audit your own schema for type problems.**
```sql
-- ① tables with terrible rows-per-page
SELECT relname, relpages, round(reltuples/nullif(relpages,0)) AS rows_per_page,
       pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_class WHERE relkind='r' AND relpages > 10000
ORDER BY rows_per_page ASC LIMIT 20;

-- ② money stored as float, and timestamps without a zone
SELECT c.relname, a.attname, t.typname
FROM pg_attribute a JOIN pg_class c ON c.oid=a.attrelid
JOIN pg_type t ON t.oid=a.atttypid
WHERE c.relkind='r' AND a.attnum>0 AND NOT a.attisdropped
  AND c.relnamespace='public'::regnamespace
  AND (t.typname IN ('float4','float8','money','timestamp')
       OR (t.typname='varchar' AND a.attname LIKE '%uuid%')
       OR (t.typname='int4' AND a.attname='id'));

-- ③ column order: how much padding are you paying?
SELECT c.relname, a.attnum, a.attname, t.typname, t.typalign, t.typlen
FROM pg_attribute a JOIN pg_class c ON c.oid=a.attrelid
JOIN pg_type t ON t.oid=a.atttypid
WHERE c.relname='events' AND a.attnum>0 AND NOT a.attisdropped
ORDER BY a.attnum;
--  typalign: c=1 i=4 d=8 s=2. Scan for a 'c' or 's' sitting before a 'd'.
```

---

## The design decision framework

```
FOR EVERY COLUMN, IN THIS ORDER:

 ① CORRECTNESS — what can this legally hold?
      money            → bigint in minor units.  NEVER float, NEVER money type
      timestamps       → timestamptz.            NEVER timestamp (Topic 26)
      identifiers      → bigint or uuid.         NEVER text
      a fixed set      → enum / text+CHECK / lookup table
      free text        → text + CHECK(length)
      measurements     → float8 is fine (exactness not required)
      variable shape   → jsonb, for the genuinely variable part only

 ② SIZE — the smallest type that holds the 10-year maximum × 10
      quantities, small counters       → smallint
      most counts, FKs to small tables → integer
      ★ primary keys, money, anything counting events → bigint

 ③ ALIGNMENT — declare in descending width
      8-byte → 4-byte → 2-byte → 1-byte → uuid → varlena
      Free at CREATE TABLE. A rewrite later.

 ④ TOAST — check every varlena column's average size
      < 500 B          → inline is fine
      500 B – 2 KB     → ★ THE DEAD ZONE. SET STORAGE EXTERNAL,
                         or move to a 1:1 side table
      > 2 KB           → TOAST handles it automatically

STRINGS: `text` + CHECK, essentially always.
  varchar(n) only if a tool demands it. char(n) only for fixed codes.

JSONB: only for the genuinely variable part.
  If a key appears in >90% of rows, it is a column.
  If a key is updated frequently, it is a column.
  If you need a constraint on it, it is a column.

ENUMS: text+CHECK for 3–6 stable states in a normal table.
  Native enum when the column appears in billions of rows (4 B matters).
  Lookup table + FK when the list is data (has attributes, edited by
  non-engineers).

THE SIGNAL TO LOOK FOR:
      SELECT relname, relpages, round(reltuples/nullif(relpages,0)) AS rpp
      FROM pg_class WHERE relkind='r' AND relpages > 10000
      ORDER BY rpp ASC LIMIT 20;

  • rows_per_page < 20 on a large table → a fat inline column. Find it:
        SELECT attname, avg(pg_column_size(col)) …  on a TABLESAMPLE
  • any float/money/timestamp column → a correctness bug, fix regardless
    of size
  • a table 20%+ larger than sum(column widths) → alignment padding
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Given `CREATE TABLE s (a boolean, b timestamptz, c integer, d bigint, e smallint, f uuid, g text);`, compute by hand: each column's offset, total padding wasted, and the tuple size. Verify with `pg_column_size(row(...))`. Then write the optimal declaration order and compute the saving per row and per 500M rows.

### Exercise 2 — medium (apply it)
Build two 200,000-row tables with an 800-byte incompressible text column — one default, one `SET STORAGE EXTERNAL`. Measure: main size, TOAST size, total size, rows/page, and the time for `SELECT count(*)`, `SELECT id`, and `SELECT payload`. Explain every difference. Then state the exact condition under which `EXTERNAL` is the **wrong** choice.

### Exercise 3 — hard (production simulation)
A 1.8 TB `events` table, 3.1 billion rows, 5 rows per page:

```sql
CREATE TABLE events (
  is_test boolean NOT NULL, id serial PRIMARY KEY, is_bot boolean NOT NULL,
  tenant_id integer NOT NULL, event_type varchar(64) NOT NULL,
  severity smallint NOT NULL, user_id bigint, amount float8,
  occurred_at timestamp NOT NULL, ingested_at timestamp NOT NULL,
  session_id varchar(36) NOT NULL, raw_payload text NOT NULL,
  properties jsonb NOT NULL DEFAULT '{}'
);
```
`raw_payload` averages 1,380 B; `properties` 240 B, of which 3 keys are present in >95% of rows.

(a) Compute the current tuple size including all padding, showing your working.
(b) Identify every type problem, classify each as **correctness** or **performance**, and rank by impact.
(c) Write the corrected DDL with optimal column order, and compute the new tuple size and rows/page.
(d) `id serial` is `int4`. Compute when it exhausts at 40,000 inserts/s, and describe exactly what `ALTER TABLE ... TYPE bigint` would do to a 1.8 TB table.
(e) `properties` has 3 near-universal keys. Show the hybrid design and compute the additional saving.
(f) Two conversions in the backfill are **dangerous**. Identify both, explain the specific way each can silently corrupt data, and give the verification you'd run *before* converting.
(g) Give the complete zero-downtime migration: dual-write, backfill strategy, verification, cutover, rollback.
(h) Write the CI check that fails a migration adding a `float`/`money`/`timestamp` column or an `int4` primary key.

---

## Mental model checkpoint

1. Why is `float8` wrong for money and `numeric` merely suboptimal? What's the right answer and why does it win on all three axes?
2. What is alignment padding? Give the ordering rule and the typical saving.
3. What is the TOAST dead zone, and why is a 1.4 KB value worse than a 50 KB one?
4. Is `varchar(50)` smaller or faster than `text` in PostgreSQL? What actually differs, and what should you use instead?
5. At what value size does the varlena header change, and by how much?
6. Name three signals that a JSONB key should be promoted to a column.
7. Why is `int4` a dangerous primary key type, and what exactly does fixing it later require?

---

## Quick reference card

| Domain | Use | Never |
|---|---|---|
| Money | `bigint` in minor units | `float`, `money`, `numeric` on hot paths |
| Timestamps | `timestamptz` | `timestamp`, `text` |
| Primary keys | `bigint` / `uuid` | `int4`, `text` |
| Identifiers | `bigint` / `uuid` | `varchar` holding a UUID |
| Strings | `text` + `CHECK(length)` | `char(n)`, `varchar(n)` for perf |
| Fixed sets | `enum` / `text`+`CHECK` / lookup | free text |
| Variable shape | `jsonb` (the variable part only) | jsonb for universal keys |

**Sizes**

| Type | Bytes | Align |
|---|---|---|
| `boolean`, `"char"` | 1 | 1 |
| `smallint` | 2 | 2 |
| `integer`, `real`, `date`, `enum` | 4 | 4 |
| `bigint`, `float8`, `timestamptz` | 8 | 8 |
| `uuid` | 16 | **1** |
| varlena header | 1 B (≤126) / 4 B (>126) | 4 |
| TOAST pointer | 18 | |
| TOAST threshold | ~2000 B | |

**The five rules**
1. Money = `bigint` in paise. Always.
2. Timestamps = `timestamptz`. Always.
3. Primary keys = `bigint`. Always.
4. Columns in descending alignment: 8 → 4 → 2 → 1 → varlena.
5. Any varlena column averaging 500 B – 2 KB → `SET STORAGE EXTERNAL`.

---

## When would I use this at work?

1. **Every `CREATE TABLE`.** Five rules, thirty seconds, and you avoid a rewrite of a multi-terabyte table later. Alignment alone is typically 20% of storage and scan time for free.

2. **A table with mysteriously slow scans.** `rows_per_page` of 5 immediately identifies a fat inline column, and `SET STORAGE EXTERNAL` gives a 20–30× improvement without touching a query.

3. **Auditing an inherited schema.** The type-audit query finds `float` money and `timestamp` columns in minutes — both correctness bugs that will otherwise surface as an unreconcilable ledger or a report that's wrong by hours.

---

## Connected topics

**Understand before this:** 04 (pages, rows-per-page), 05 (tuple layout, alignment, TOAST — the physics this topic spends).

**This unlocks:**
- **26** — time, money, and identity: the four domains that ruin schemas, in depth
- **27** — antipatterns, several of which are type choices (EAV, JSONB sprawl)
- **28** — migrations, including how to change a column type without downtime
- **16** — indexing JSONB, once you've decided what belongs in it
- **59** — partitioning, where the partition key's type matters
