# 14 — Composite Index Column Order
## Phase: Indexes

---

## ELI5 — The Simple Analogy

A phone book.

It's sorted by **surname, then first name**. So you can answer:

- "Find Sharma" → flip straight there. ✓
- "Find Sharma, Arjun" → flip to Sharma, then scan a short run. ✓
- "Find everyone whose first name is Arjun" → **useless**. Arjuns are scattered across every surname in the book. You'd read all 900 pages.

Now imagine a second phone book sorted by **first name, then surname**. It answers the third question instantly and the first one not at all.

**These are two different books.** Not "the same information ordered differently" in any way that helps you — physically different objects, each answering a different set of questions. You cannot derive one from the other by reading it sideways.

That is the entire topic. `(a, b)` and `(b, a)` are different data structures, and picking the wrong one means the planner ignores your index while you insist it should work.

---

## Where this fits in the big picture

```
   11 B-tree structure ──▶ 12 lookup (Index Cond vs Filter)
                                        │
                           13 index types (partial, INCLUDE)
                                        │
                                        ▼
                     ┌─────────────────────────────────────┐
                     │ 14 COMPOSITE COLUMN ORDER           │ ← YOU ARE HERE
                     │ the highest-leverage index decision │
                     └──────────────────┬──────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
             15 selectivity      18 the planner       19 join algorithms
             (which column       (how it costs        (composite keys and
              first?)             a composite)         merge joins)
```

Topic 13 gave you the *variants*. **This topic gives you the single decision that most often separates a 0.3 ms query from a 4 second one** — and it costs nothing to get right if you know the rules.

---

## What is this?

A **composite (multi-column) index** builds its B-tree on a *tuple* of columns, compared left to right. `(a, b, c)` sorts by `a`; ties broken by `b`; ties broken by `c`.

The consequences all follow from that one sentence:

- The index can serve any **leftmost prefix**: `(a)`, `(a,b)`, `(a,b,c)` — but never `(b)` or `(c)` alone.
- After the first **range** predicate, later columns can no longer restrict the scan.
- It can satisfy `ORDER BY` only for a prefix, and only in matching (or exactly reversed) direction.

---

## Why does it matter for a backend developer?

Because column order is free to choose and expensive to get wrong, and the failure is silent — you get an index that "exists" and a query that's still slow.

Real numbers from Example 1 below, same query, same three columns, three orders:

```
 CREATE INDEX ON orders (status, merchant_id, created_at);   → 2,840 ms
 CREATE INDEX ON orders (merchant_id, status, created_at);   →     0.4 ms
 CREATE INDEX ON orders (merchant_id, created_at, status);   →   180 ms
                                                                ─────────
                                                          7,100× spread
```

Three indexes. Same columns. Same data. Same query. The only difference is the order of three identifiers in a `CREATE INDEX` statement.

And the second-order win: **one well-ordered composite index often replaces four single-column indexes** — which means 4× less write amplification, 4× less buffer pool, and fewer planner alternatives to get wrong (Topic 17).

---

## The physical reality

### What a composite key actually is on disk

```
 CREATE INDEX idx ON orders (merchant_id, status, created_at);

 LEAF ENTRY:
 ┌──────────────────────────────────────────────────────────────────┐
 │ IndexTupleData (8 B) │ merchant_id (8 B) │ status (varlena) │    │
 │                      │ created_at (8 B)  │ → TID (block,off)     │
 └──────────────────────────────────────────────────────────────────┘
   The key is ONE concatenated tuple, compared lexicographically.

 LEAF PAGE CONTENTS, in sorted order:
 ┌────────────────────────────────────────────────────────────────┐
 │ (12, 'awaiting', 2026-03-01 09:00) → (1204,3)                  │
 │ (12, 'awaiting', 2026-03-01 11:22) → (2891,7)                  │
 │ (12, 'awaiting', 2026-03-02 08:14) → (4102,1)  ← sorted by 3rd │
 │ (12, 'delivered',2026-01-01 00:04) → (8891,2)  ← status broke  │
 │ (12, 'delivered',2026-01-01 00:09) → (9004,5)     the tie      │
 │ (13, 'awaiting', 2026-02-11 14:00) → (1102,8)  ← merchant broke│
 │ (13, 'awaiting', 2026-02-11 14:02) → (1102,9)     the tie      │
 └────────────────────────────────────────────────────────────────┘
       ↑           ↑              ↑
    primary    secondary      tertiary
     sort        sort           sort
```

### Why the leftmost-prefix rule is forced by the physics

```
 QUERY: WHERE merchant_id = 12
   → all merchant 12 rows are CONTIGUOUS. One descent, then walk right.
     ✓✓✓ perfect

 QUERY: WHERE merchant_id = 12 AND status = 'awaiting'
   → within merchant 12, all 'awaiting' rows are contiguous.
     ✓✓✓ perfect — two columns narrowed during the descent

 QUERY: WHERE status = 'awaiting'                    ← NO merchant_id
   → 'awaiting' rows exist under EVERY merchant_id:
        (12,'awaiting',...)   (13,'awaiting',...)   (14,'awaiting',...)
        ...scattered across the ENTIRE index...
     ✗ There is no contiguous run to descend to. The tree is sorted by
       merchant_id first, and you gave no merchant_id.

 ⇒ THE INDEX IS SORTED BY THE FIRST COLUMN. Without a value for it,
   there is no starting point for the descent. This is not a limitation
   PostgreSQL chose — it is what "sorted by (a,b)" MEANS.
```

**One important nuance most people miss:** PostgreSQL *can* still use a composite index without the leading column — via a **full index scan** — when the index is much smaller than the table:

```
EXPLAIN SELECT count(*) FROM orders WHERE status = 'awaiting';
```
```
Aggregate
  ->  Index Only Scan using idx_merchant_status_created on orders
        Index Cond: (status = 'awaiting'::text)
        Filter: ...
```

It reads *every leaf page* of the index rather than the whole heap. That's better than a seq scan but nothing like a proper index lookup — you're scanning 100% of an index instead of 100% of a table. Don't mistake "the index appears in the plan" for "the index is working."

### The range-stops-the-descent rule

This is the second rule, and it's the one that produces the subtlest bugs.

```
 INDEX (merchant_id, created_at, status)

 QUERY: merchant_id = 12 AND created_at > '2026-03-01' AND status = 'awaiting'

 THE DESCENT:
   1. merchant_id = 12       → jump to the start of merchant 12    ✓
   2. created_at > 2026-03-01→ jump to the first qualifying entry  ✓
   3. status = 'awaiting'    → ✗ CANNOT NARROW.
        Within "merchant 12, created_at > 2026-03-01" the rows are
        sorted by created_at, and status varies freely inside that run:

           (12, 2026-03-01 09:00, 'delivered')
           (12, 2026-03-01 11:22, 'awaiting')   ← want
           (12, 2026-03-02 08:14, 'cancelled')
           (12, 2026-03-02 09:01, 'awaiting')   ← want
           (12, 2026-03-03 14:00, 'delivered')

        `status` is not sorted within the range, so it can only be
        used as a FILTER on each entry read.

 EXPLAIN shows this precisely:
   Index Cond: (merchant_id = 12 AND created_at > '2026-03-01')
   Filter: (status = 'awaiting')           ← the giveaway
   Rows Removed by Filter: 184,220         ← rows read, then discarded

 ⇒ THE RULE: EQUALITY COLUMNS FIRST, THE RANGE COLUMN LAST.
   Everything after the first inequality is a filter, not a search bound.

 CORRECT ORDER: (merchant_id, status, created_at)
   Index Cond: (merchant_id = 12 AND status = 'awaiting'
                AND created_at > '2026-03-01')
   Rows Removed by Filter: 0
```

### ORDER BY and direction

```
 INDEX (merchant_id, created_at DESC)

 The leaf level can be walked FORWARD or BACKWARD. So it satisfies:
   ORDER BY merchant_id ASC,  created_at DESC     ← forward walk  ✓
   ORDER BY merchant_id DESC, created_at ASC      ← backward walk ✓

 It does NOT satisfy:
   ORDER BY merchant_id ASC,  created_at ASC      ✗ mixed directions
   ORDER BY merchant_id DESC, created_at DESC     ✗ mixed directions

 ⇒ You need matching, or EXACTLY REVERSED, directions on every column.
   For mixed sorts, declare them explicitly:
     CREATE INDEX ON orders (merchant_id ASC, created_at DESC);

 ★ AND: when the leading columns are fixed by EQUALITY, they drop out
   of the ordering requirement:
     WHERE merchant_id = 12 ORDER BY created_at DESC
   is satisfied by (merchant_id, created_at DESC), because within
   merchant 12 the rows are already in created_at DESC order.
```

---

## How it works — step by step

### The descent, column by column

```
 INDEX (merchant_id, status, created_at)
 QUERY WHERE merchant_id = 12 AND status = 'awaiting'
         AND created_at >= '2026-03-01'
       ORDER BY created_at ASC LIMIT 50

 1. Build the search key: (12, 'awaiting', '2026-03-01').
 2. Descend the tree comparing the FULL TUPLE lexicographically:
      root:     is (12,'awaiting','2026-03-01') < (40,...)?  yes → left
      internal: ...                                          → child
      leaf:     binary search for the first entry ≥ the key
 3. Position found. Now WALK FORWARD:
      (12,'awaiting','2026-03-01 09:00')  ✓ emit
      (12,'awaiting','2026-03-01 11:22')  ✓ emit
      ...
      (12,'awaiting','2026-03-04 22:10')  ✓ emit   [50th → LIMIT satisfied,
                                                     STOP]
 4. Stop condition (if LIMIT hadn't fired): the first entry where
    merchant_id ≠ 12 OR status ≠ 'awaiting'. Both are equality bounds,
    so the scan range is exact.

 ★ THREE THINGS HAPPENED AT ONCE:
   • FILTER — all three predicates narrowed the scan (all in Index Cond)
   • ORDER  — output is already in created_at ASC order (no Sort node)
   • LIMIT  — the scan stops after 50 rows (streaming, not blocking)
   This is the ideal an index design should aim for. (Topic 10's
   "three things an index can do".)
```

### The four-step ordering procedure

```
 Given a query, derive the column order mechanically:

 STEP 1 — EQUALITY PREDICATES FIRST
   Every column compared with =, IN, or IS NULL goes at the front.
   These are the only ones that can narrow subsequent columns.

 STEP 2 — AMONG EQUALITY COLUMNS, ORDER BY... what?
   ★ For a SINGLE query, the order among equality columns does NOT
     matter for that query's performance — any order narrows to the
     same set.
   It matters for:
     (a) SHARED USE — put the column that MORE queries filter on first,
         so the index serves more leftmost prefixes. This is the real
         reason, and it's about index count, not speed.
     (b) SIZE — with PG13 deduplication, a low-cardinality leading
         column compresses better, making the index smaller.
   ⚠ The common advice "most selective column first" is largely a MYTH
     for equality predicates in PostgreSQL. Both orders narrow to the
     same rows in the same number of page reads. Optimise for reuse.

 STEP 3 — THE ORDER BY COLUMNS NEXT, IN ORDER, WITH DIRECTIONS
   If the query sorts, put those columns next, matching ASC/DESC.
   This deletes the Sort node and enables early LIMIT termination.

 STEP 4 — THE RANGE COLUMN LAST (at most one useful one)
   >, <, BETWEEN, LIKE 'x%'. Only the FIRST range column narrows;
   everything after it becomes a filter.
   ⚠ Often steps 3 and 4 are the SAME column (created_at), which is
     why the classic shape is:
         (equality..., sort_and_range_column)

 THEN: anything the query OUTPUTS but doesn't filter/sort on → INCLUDE.
```

### Worked derivation

```
 SELECT id, user_id, total_paise
 FROM orders
 WHERE merchant_id = $1 AND status = 'paid' AND created_at >= $2
 ORDER BY created_at DESC
 LIMIT 50;

 STEP 1 equality:  merchant_id, status
 STEP 2 order among them: does anything else filter on status alone? No.
                          Do many queries filter on merchant_id? Yes.
                          → merchant_id first (more reusable prefix)
 STEP 3 ORDER BY:  created_at DESC
 STEP 4 range:     created_at (same column — perfect)
 OUTPUT-only:      id, user_id, total_paise → INCLUDE

 ⇒ CREATE INDEX idx_orders_merchant_paid
      ON orders (merchant_id, status, created_at DESC)
      INCLUDE (id, user_id, total_paise);

 …and since `status = 'paid'` is a constant here, consider going further:
 ⇒ CREATE INDEX idx_orders_merchant_paid
      ON orders (merchant_id, created_at DESC)
      INCLUDE (id, user_id, total_paise)
      WHERE status = 'paid';                      -- partial (Topic 13)
   Smaller index, status drops out of the key entirely. Use this when
   'paid' is a minority of rows and always a literal in the SQL.
```

---

## Concept breakdown

```
COMPOSITE / MULTI-COLUMN INDEX
│  └── B-tree whose key is a TUPLE, compared left to right,
│      lexicographically. Sorted by col1; ties by col2; ties by col3.
│
├── LEFTMOST PREFIX RULE
│    (a,b,c) serves:  (a) · (a,b) · (a,b,c)
│    (a,b,c) does NOT serve:  (b) · (c) · (b,c)
│    ⇒ ONE index (a,b,c) replaces THREE indexes (a), (a,b), (a,b,c)
│      ★ this is the biggest practical win: fewer indexes, less write tax
│
├── EQUALITY-THEN-RANGE RULE
│    Columns before the first inequality NARROW the scan (Index Cond).
│    Columns after it only FILTER (Filter + Rows Removed by Filter).
│    ⇒ At most ONE range column can be "used", and it must be last.
│
├── ORDERING RULE
│    Satisfies ORDER BY over a leftmost prefix, in matching or exactly
│    reversed directions. Equality-bound leading columns drop out of
│    the ordering requirement.
│
└── COLUMN ORDER AMONG EQUALITIES
     ⚠ NOT about selectivity (a myth). It's about which prefixes you
       want available to OTHER queries.

THE THREE THINGS TO CHECK IN EXPLAIN
│
├── Index Cond: (...)          ← columns that narrowed the scan.  GOOD
├── Filter: (...)              ← columns applied after fetch.     BAD
│   + Rows Removed by Filter   ← the size of your mistake
└── Sort node present?         ← ORDER BY not satisfied by the index

MULTIPLE SINGLE-COLUMN INDEXES vs ONE COMPOSITE
│
├── The planner CAN combine single-column indexes with BitmapAnd:
│      Bitmap Index Scan on idx_a  → bitmap A
│      Bitmap Index Scan on idx_b  → bitmap B
│      BitmapAnd(A,B)
├── But it must scan BOTH indexes fully for the matching values, then
│   intersect in memory. A composite narrows during ONE descent.
└── ⇒ Composite is usually 5–50× better. BitmapAnd is the fallback,
     not the goal.

COLUMN ORDER AND INDEX SIZE (PG13+)
│
└── Deduplication compresses runs of equal keys. A low-cardinality
    LEADING column produces long runs → smaller index.
      (status, merchant_id, created_at)  → status dedups well
      (merchant_id, status, created_at)  → merchant dedups less
    ⚠ This is a size argument, not a speed argument. Never reorder for
      size at the cost of the leftmost-prefix rule.
```

---

## Diagrams

**Diagram 1 — big picture: two orders are two different structures**

```
   INDEX (merchant_id, status)              INDEX (status, merchant_id)
   ┌───────────────────────────┐            ┌───────────────────────────┐
   │ (12, 'awaiting')          │            │ ('awaiting', 12)          │
   │ (12, 'awaiting')          │            │ ('awaiting', 13)          │
   │ (12, 'delivered')         │            │ ('awaiting', 14)          │
   │ (12, 'delivered')         │            │ ('awaiting', 15)          │
   │ (13, 'awaiting')          │            │ ('delivered', 12)         │
   │ (13, 'cancelled')         │            │ ('delivered', 12)         │
   │ (14, 'awaiting')          │            │ ('delivered', 13)         │
   └───────────────────────────┘            └───────────────────────────┘
     WHERE merchant_id=12  ✓✓✓                WHERE merchant_id=12  ✗
     WHERE status='awaiting' ✗                WHERE status='awaiting' ✓✓✓
     WHERE both            ✓✓✓                WHERE both            ✓✓✓

   ⇒ Neither is "better". They answer different questions.
     The one you need depends on which column you ALWAYS have.
```

**Diagram 2 — data flow: where the range column stops the descent**

```
  INDEX (merchant_id, created_at, status)     INDEX (merchant_id, status, created_at)
  ─────────────────────────────────────       ──────────────────────────────────────
   descend: merchant_id = 12                   descend: merchant_id = 12
        ↓ narrowed to 1.2M rows                     ↓ narrowed to 1.2M rows
   descend: created_at > '2026-03-01'          descend: status = 'awaiting'
        ↓ narrowed to 190k rows                     ↓ narrowed to 4,100 rows
   ✗ status: NOT SORTED HERE                   descend: created_at > '2026-03-01'
        ↓                                           ↓ narrowed to 3,982 rows
   read 190,000 index entries                  read 3,982 index entries
   read 190,000 heap tuples                    read 3,982 heap tuples
        ↓                                           ↓
   FILTER status='awaiting'                    emit — nothing discarded
        ↓
   discard 186,018   ← 97.9% wasted

        2,840 ms                                      0.4 ms
```

**Diagram 3 — before/after: one composite replaces four singles**

```
 BEFORE — "one index per column"
 ┌──────────────────────────────────────────────────────────────────┐
 │ idx_merchant   (merchant_id)              1,804 MB               │
 │ idx_status     (status)                   1,204 MB               │
 │ idx_created    (created_at)               1,804 MB               │
 │ idx_user       (user_id)                  1,804 MB               │
 │                                          ──────────              │
 │                                           6,616 MB               │
 │ every INSERT: 4 B-tree descents + 4 leaf writes + 4 WAL records  │
 │ the query: BitmapAnd over 2 indexes, 190k-entry bitmap → 640 ms  │
 └──────────────────────────────────────────────────────────────────┘

 AFTER — one well-ordered composite + one for the other access path
 ┌──────────────────────────────────────────────────────────────────┐
 │ idx_merchant_status_created                                      │
 │   (merchant_id, status, created_at DESC)  2,410 MB               │
 │   ↳ also serves (merchant_id) and (merchant_id, status) queries   │
 │ idx_user_created (user_id, created_at DESC) 2,410 MB             │
 │                                          ──────────              │
 │                                           4,820 MB   (−27%)      │
 │ every INSERT: 2 descents + 2 writes + 2 WAL records  (−50%)      │
 │ the query: single Index Scan, 3,982 entries → 0.4 ms (1,600×)    │
 └──────────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
CREATE TABLE orders (
  id          bigserial PRIMARY KEY,
  merchant_id bigint      NOT NULL,
  user_id     bigint      NOT NULL,
  status      text        NOT NULL,
  total_paise bigint      NOT NULL,
  created_at  timestamptz NOT NULL
);

INSERT INTO orders (merchant_id, user_id, status, total_paise, created_at)
SELECT (random()*2000)::bigint,
       (random()*500000)::bigint,
       CASE WHEN random() < 0.94 THEN 'delivered'
            WHEN random() < 0.97 THEN 'cancelled'
            WHEN random() < 0.995 THEN 'shipped' ELSE 'awaiting' END,
       (random()*500000)::bigint,
       now() - (random()*400)::int * interval '1 day'
FROM generate_series(1, 8000000);
VACUUM ANALYZE orders;
```

**The query we're optimising:**

```sql
SELECT id, user_id, total_paise FROM orders
WHERE merchant_id = 412 AND status = 'awaiting' AND created_at >= '2026-03-01'
ORDER BY created_at DESC LIMIT 50;
```

**Order A — the worst: low-cardinality equality first, range in the middle.**

```sql
CREATE INDEX idx_a ON orders (status, created_at, merchant_id);
VACUUM ANALYZE orders;
EXPLAIN (ANALYZE, BUFFERS) <the query>;
```
```
Limit  (actual time=2841.1..2841.2 rows=12 loops=1)
  ->  Index Scan Backward using idx_a on orders
        (actual time=2841.1..2841.2 rows=12 loops=1)
        Index Cond: ((status = 'awaiting') AND (created_at >= '2026-03-01'))
        Filter: (merchant_id = 412)
        Rows Removed by Filter: 39882                ← ⚠
        Buffers: shared hit=8104 read=31998
Execution Time: 2841.6 ms
```
`merchant_id` came after the range column → it can only filter. 39,882 rows read and discarded.

**Order B — the correct one: equalities first, range last.**

```sql
CREATE INDEX idx_b ON orders (merchant_id, status, created_at DESC);
DROP INDEX idx_a;
VACUUM ANALYZE orders;
EXPLAIN (ANALYZE, BUFFERS) <the query>;
```
```
Limit  (actual time=0.043..0.089 rows=12 loops=1)
  ->  Index Scan using idx_b on orders
        Index Cond: ((merchant_id = 412) AND (status = 'awaiting')
                     AND (created_at >= '2026-03-01'))
        Buffers: shared hit=17
Execution Time: 0.112 ms
```
**All three predicates in `Index Cond`. `Rows Removed by Filter: 0`. No Sort node. 40,102 buffers → 17. 2,842 ms → 0.11 ms — 25,800×.**

**Order C — range before the last equality: the subtle mistake.**

```sql
CREATE INDEX idx_c ON orders (merchant_id, created_at DESC, status);
DROP INDEX idx_b;
VACUUM ANALYZE orders;
EXPLAIN (ANALYZE, BUFFERS) <the query>;
```
```
Limit  (actual time=178.4..180.1 rows=12 loops=1)
  ->  Index Scan using idx_c on orders
        Index Cond: ((merchant_id = 412) AND (created_at >= '2026-03-01'))
        Filter: (status = 'awaiting')
        Rows Removed by Filter: 2104                 ← ⚠
        Buffers: shared hit=2130
Execution Time: 180.4 ms
```
**Much better than A (merchant_id is leading), much worse than B (status became a filter).** This is the order people write most often, because "put the date after the id" feels natural.

**Prove the leftmost-prefix rule.**

```sql
DROP INDEX idx_c;
CREATE INDEX idx_b ON orders (merchant_id, status, created_at DESC);
VACUUM ANALYZE orders;

EXPLAIN SELECT count(*) FROM orders WHERE merchant_id = 412;
--  Index Only Scan using idx_b   ✓ prefix (a)

EXPLAIN SELECT count(*) FROM orders WHERE merchant_id = 412 AND status='awaiting';
--  Index Only Scan using idx_b   ✓ prefix (a,b)

EXPLAIN SELECT count(*) FROM orders WHERE status = 'awaiting';
--  Index Only Scan using idx_b
--    Index Cond: (status = 'awaiting')
--    ⚠ this is a FULL INDEX SCAN — it reads every leaf page.
--      Check the buffers:
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM orders WHERE status = 'awaiting';
```
```
Aggregate  (actual time=1104.2..1104.2 rows=1 loops=1)
  ->  Index Only Scan using idx_b on orders (actual rows=39894)
        Index Cond: (status = 'awaiting'::text)
        Heap Fetches: 0
        Buffers: shared hit=42104         ← scanned the WHOLE index
Execution Time: 1104.9 ms
```
**Better than a seq scan (42k index pages vs 98k heap pages), but 2,700× worse than a real lookup.** "The index appears in the plan" ≠ "the index is working."

**Prove the ORDER BY direction rule.**

```sql
EXPLAIN SELECT * FROM orders WHERE merchant_id=412 ORDER BY created_at DESC LIMIT 20;
--  Index Scan using idx_b        ✓ no Sort
EXPLAIN SELECT * FROM orders WHERE merchant_id=412 ORDER BY created_at ASC LIMIT 20;
--  Index Scan Backward using idx_b   ✓ no Sort (exactly reversed)
EXPLAIN SELECT * FROM orders WHERE merchant_id=412
        ORDER BY status ASC, created_at ASC LIMIT 20;
```
```
Limit
  ->  Sort                                    ← ⚠ mixed directions
        Sort Key: status, created_at
        ->  Index Scan using idx_b on orders
```
Index is `(merchant_id, status ASC, created_at DESC)`. The query wants `status ASC, created_at ASC` — neither matching nor fully reversed. A Sort appears, and `LIMIT 20` now costs a full materialisation.

**Prove that among equality columns, order barely matters for speed.**

```sql
CREATE INDEX idx_ms ON orders (merchant_id, status);
CREATE INDEX idx_sm ON orders (status, merchant_id);
VACUUM ANALYZE orders;

SET enable_seqscan=off;
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM orders
 WHERE merchant_id=412 AND status='awaiting';
```
Force each with `SET enable_indexscan`/`pg_hint_plan`, or just drop one at a time. You'll find both do ~the same buffer count for *this* query. The difference is entirely about which *other* queries each can serve:

```sql
SELECT pg_size_pretty(pg_relation_size('idx_ms')),
       pg_size_pretty(pg_relation_size('idx_sm'));
```
```
 pg_size_pretty | pg_size_pretty
----------------+----------------
 240 MB         | 182 MB          ← (status,...) dedups better — a SIZE win
```
But `idx_sm` cannot serve `WHERE merchant_id = 412` alone, and `idx_ms` can. **Reusability beats 24% of size, almost always.**

---

## Example 2 — production scenario

**The situation.** An order-management API. Six endpoints hit `orders` (8M rows, 3.2 GB). Someone added an index per endpoint over two years: **nine indexes, 14 GB.** Writes are slow, and three of the six endpoints are still slow.

**Step 1 — write down the access patterns.** (Step 1 of the case-study method — this is the actual work.)

| # | Endpoint | Predicate | Sort | Output | Rate |
|---|---|---|---|---|---|
| Q1 | merchant dashboard | `merchant_id = ? AND status = ? AND created_at >= ?` | `created_at DESC` | id, user_id, total | 4,000/s |
| Q2 | merchant "all orders" | `merchant_id = ?` | `created_at DESC` | id, status, total | 1,200/s |
| Q3 | customer order list | `user_id = ?` | `created_at DESC` | id, merchant_id, status, total | 9,000/s |
| Q4 | fulfilment worker | `status = 'awaiting' AND warehouse_id = ?` | `priority DESC, created_at` | id, payload | 8,000/s |
| Q5 | order detail | `id = ?` | — | * | 12,000/s |
| Q6 | finance export | `created_at >= ? AND created_at < ?` | `created_at` | many | 2/hour |

**Step 2 — the existing nine indexes.**

```sql
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS size, idx_scan,
       pg_get_indexdef(indexrelid) AS def
FROM pg_stat_user_indexes WHERE relname='orders'
ORDER BY pg_relation_size(indexrelid) DESC;
```
```
        indexrelname          |  size   |  idx_scan  | def
------------------------------+---------+------------+------------------------------
 idx_orders_created_status     | 2410 MB |     412008 | (created_at, status)
 idx_orders_status_created     | 2410 MB |   88104229 | (status, created_at)
 idx_orders_merchant           | 1804 MB |   14200881 | (merchant_id)
 idx_orders_user               | 1804 MB |   98004112 | (user_id)
 idx_orders_created            | 1804 MB |       8102 | (created_at)
 idx_orders_status             | 1204 MB |          0 | (status)
 idx_orders_warehouse          | 1804 MB |    2100448 | (warehouse_id)
 idx_orders_merchant_status    |  980 MB |   41200884 | (merchant_id, status)
 orders_pkey                   | 1804 MB |  412008841 | (id)
```

**Step 3 — apply the four-step procedure to each query.**

```
 Q1: eq {merchant_id, status} · sort/range {created_at DESC} · out {id,user_id,total}
     → (merchant_id, status, created_at DESC) INCLUDE (user_id, total_paise)
       ★ also serves Q2 as the (merchant_id) prefix... but Q2 sorts by
         created_at without a status filter, so the prefix alone leaves a Sort.
         Check whether that matters: Q2 is 1,200/s. It does.

 Q2: eq {merchant_id} · sort {created_at DESC}
     → needs (merchant_id, created_at DESC)
     ⚠ CONFLICT with Q1's ideal. Two options:
       (a) two indexes: (merchant_id, status, created_at) + (merchant_id, created_at)
       (b) ONE index (merchant_id, created_at DESC) INCLUDE (status, ...)
           → Q1 then filters status post-descent. Rows Removed by Filter
             = merchant's orders in the date range that aren't 'awaiting'.
             For a merchant with 4,000 orders in range and 6% 'awaiting',
             that's 3,760 wasted — acceptable at 4,000/s? Measure it.
       DECISION: Q1 is 4,000/s and status='awaiting' is 0.5% of rows →
       a PARTIAL index is better than either:
         (merchant_id, created_at DESC) WHERE status='awaiting'   [tiny]
       plus (merchant_id, created_at DESC) for Q2.                [general]

 Q3: eq {user_id} · sort {created_at DESC} · out {merchant_id,status,total}
     → (user_id, created_at DESC) INCLUDE (merchant_id, status, total_paise)

 Q4: eq {status='awaiting' (constant!), warehouse_id} · sort {priority DESC, created_at}
     → PARTIAL: (warehouse_id, priority DESC, created_at) WHERE status='awaiting'
       INCLUDE (payload)
       ★ status leaves the key entirely — it's fixed by the predicate (Topic 13)

 Q5: pkey. Already covered.

 Q6: 2/hour, full range scan. NO INDEX — a seq scan on a replica is correct.
     (An index matching 100% of a date range is pure write tax — Topic 10.)
```

**Step 4 — the final index set.**

```sql
-- Q1 (4,000/s) — tiny partial index, status out of the key
CREATE INDEX CONCURRENTLY idx_orders_merchant_awaiting
  ON orders (merchant_id, created_at DESC)
  INCLUDE (id, user_id, total_paise)
  WHERE status = 'awaiting';

-- Q2 (1,200/s)
CREATE INDEX CONCURRENTLY idx_orders_merchant_created
  ON orders (merchant_id, created_at DESC)
  INCLUDE (status, total_paise);

-- Q3 (9,000/s) — the highest-traffic non-PK path
CREATE INDEX CONCURRENTLY idx_orders_user_created
  ON orders (user_id, created_at DESC)
  INCLUDE (merchant_id, status, total_paise);

-- Q4 (8,000/s) — partial, status fixed
CREATE INDEX CONCURRENTLY idx_orders_fulfilment
  ON orders (warehouse_id, priority DESC, created_at ASC)
  INCLUDE (id)
  WHERE status = 'awaiting';

-- drop the nine
DROP INDEX CONCURRENTLY idx_orders_created_status, idx_orders_status_created,
  idx_orders_merchant, idx_orders_user, idx_orders_created, idx_orders_status,
  idx_orders_warehouse, idx_orders_merchant_status;
```

**Step 5 — results.**

| | Before | After |
|---|---|---|
| Indexes (excl. PK) | 8 | **4** |
| Index size | 14.2 GB | **5.1 GB** (−64%) |
| Q1 p99 | 840 ms | **0.4 ms** |
| Q2 p99 | 210 ms | **1.1 ms** |
| Q3 p99 | 92 ms | **0.6 ms** |
| Q4 p99 | 4,100 ms | **0.3 ms** |
| Inserts/sec | 5,400 | **11,200** (+107%) |
| WAL/hour | 280 GB | **142 GB** |

**Note the shape of the win.** Every query got faster *and* there are half as many indexes. That's the signature of correct composite design: you're not trading reads for writes, you're removing indexes that were badly ordered and therefore doing work without delivering value.

**Step 6 — the rule that prevents recurrence.**

> Every index PR must include: the exact query it serves, its call rate, and an `EXPLAIN (ANALYZE, BUFFERS)` showing `Rows Removed by Filter: 0` and no `Sort` node above the scan.

---

## Common mistakes

**1. Putting the range column before an equality column.**
- *Symptom:* index is used, `Rows Removed by Filter` is large, query is 100× slower than expected.
- *Engine-level why:* everything after the first inequality is unsorted within the range, so it can only filter.
- *Diagnose:* compare `Index Cond` to your `WHERE` clause. Any predicate that appears in `Filter` instead is a column in the wrong position.
- *Fix:* equality columns first, range last.

**2. Believing "most selective column first."**
- *Symptom:* teams reorder equality columns based on cardinality and see no improvement.
- *Engine-level why:* for pure equality predicates, both orders narrow to the same leaf range in the same number of descents. The B-tree doesn't care which column eliminated more rows — it does one lexicographic descent either way.
- *Fix:* order equality columns by **which prefixes other queries need**, not by selectivity. (The selectivity rule *does* apply to which column to put first when only *some* queries have both — because that determines what the prefix serves.)

**3. Duplicating a prefix as its own index.**
- *Symptom:* `(merchant_id)` and `(merchant_id, status)` both exist.
- *Engine-level why:* the composite already serves the prefix. The single-column index is pure write tax.
- *Diagnose:* the redundancy query in Topic 10.
- *Fix:* drop the prefix. ⚠ Never on a `UNIQUE` index — `UNIQUE(a)` and `INDEX(a,b)` enforce different things.

**4. Mixed ORDER BY directions.**
- *Symptom:* a Sort node appears despite a "matching" index; `LIMIT` stops helping.
- *Engine-level why:* the leaf list can be walked forward or backward, giving exactly two orderings. Mixed directions are neither.
- *Fix:* declare directions explicitly: `(a ASC, b DESC)`.

**5. Assuming an index in the plan means the index is working.**
- *Symptom:* `Index Only Scan using idx_b` — and 42,000 buffers read.
- *Engine-level why:* without the leading column, PostgreSQL may do a **full index scan** — reading every leaf page. It's cheaper than a seq scan, so the planner picks it, but it's not a lookup.
- *Diagnose:* buffers, not plan node names. A real lookup on a 3-level tree reads tens of pages, not tens of thousands.

**6. Too many columns in the key.**
- *Symptom:* a 6-column index that's 4× the size and one level deeper.
- *Engine-level why:* every key byte reduces fanout (Topic 11) and is stored in internal nodes too.
- *Fix:* only filter/sort columns belong in the key. Output-only columns go in `INCLUDE`, which keeps internal nodes narrow.

**7. Ignoring the `IN` / `= ANY` case.**
- *Symptom:* `WHERE merchant_id = ANY($1) AND created_at > $2` behaves like a range on the first column.
- *Engine-level why:* PostgreSQL handles this as multiple descents (one per array element), so subsequent columns *do* still narrow within each. But the planner may choose a bitmap scan instead, losing ordering.
- *Diagnose:* look for `Index Cond: (merchant_id = ANY (...))` and whether a Sort appeared.
- *Fix:* usually fine; if ordering matters, consider a `LATERAL` join over the array so each element gets its own ordered scan.

---

## Hands-on proof

**PROVE IT #1 — the three orders (Example 1).** Build `idx_a`, `idx_b`, `idx_c` one at a time and record `Index Cond`, `Filter`, `Rows Removed by Filter`, and buffers for each.

**PROVE IT #2 — leftmost prefix, measured in buffers.**
```sql
CREATE INDEX ix ON orders (merchant_id, status, created_at DESC);
VACUUM ANALYZE orders;
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM orders WHERE merchant_id=412;
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM orders WHERE merchant_id=412 AND status='awaiting';
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM orders WHERE status='awaiting';
```
Buffers: ~20, ~8, ~42,000. The third "uses" the index and reads all of it.

**PROVE IT #3 — see the sorted tuple in the leaf.**
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
SELECT itemoffset, ctid, data FROM bt_page_items('ix', 3) LIMIT 6;
```
The `data` column is the concatenated key bytes — `merchant_id` then `status` then `created_at`, in that byte order. You can literally see the sort priority.

**PROVE IT #4 — ORDER BY direction.**
```sql
CREATE INDEX ix_dir ON orders (merchant_id, created_at DESC);
EXPLAIN SELECT * FROM orders WHERE merchant_id=412 ORDER BY created_at DESC LIMIT 10; -- no Sort
EXPLAIN SELECT * FROM orders WHERE merchant_id=412 ORDER BY created_at ASC  LIMIT 10; -- Backward, no Sort
EXPLAIN SELECT * FROM orders ORDER BY merchant_id ASC, created_at ASC LIMIT 10;       -- Sort appears
```

**PROVE IT #5 — composite beats BitmapAnd.**
```sql
CREATE INDEX ix1 ON orders (merchant_id);
CREATE INDEX ix2 ON orders (status);
DROP INDEX IF EXISTS ix;
VACUUM ANALYZE orders;
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM orders WHERE merchant_id=412 AND status='awaiting';
-- BitmapAnd over two indexes

CREATE INDEX ix ON orders (merchant_id, status);
DROP INDEX ix1, ix2;
VACUUM ANALYZE orders;
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM orders WHERE merchant_id=412 AND status='awaiting';
-- single Index Only Scan, far fewer buffers
```

**PROVE IT #6 — deduplication and leading-column cardinality.**
```sql
CREATE INDEX ix_ms ON orders (merchant_id, status);
CREATE INDEX ix_sm ON orders (status, merchant_id);
SELECT relname, pg_size_pretty(pg_relation_size(oid)) FROM pg_class
WHERE relname IN ('ix_ms','ix_sm');
```
The low-cardinality-leading version is smaller — a real but secondary consideration.

**PROVE IT #7 — the `INCLUDE` vs key trade.**
```sql
CREATE INDEX ix_key  ON orders (merchant_id, status, created_at, total_paise, user_id);
CREATE INDEX ix_incl ON orders (merchant_id, status, created_at) INCLUDE (total_paise, user_id);
SELECT c.relname, i.tree_level, pg_size_pretty(i.index_size::bigint)
FROM pg_class c, LATERAL pgstatindex(c.oid) i WHERE c.relname IN ('ix_key','ix_incl');
```
`INCLUDE` keeps internal nodes narrow → often one level shallower and smaller overall.

---

## The design decision framework

```
THE FOUR-STEP PROCEDURE — apply to every query, mechanically:

  1. EQUALITY columns first        (=, IN, IS NULL)
  2. Among them, order by REUSE    (which prefix do other queries need?)
                                    NOT by selectivity — that's a myth
                                    for equality predicates
  3. ORDER BY columns next         with matching ASC/DESC directions
  4. The RANGE column LAST         (>, <, BETWEEN, LIKE 'x%')
                                    only ONE can be used; the rest filter

  THEN: output-only columns → INCLUDE
  THEN: is any equality column a CONSTANT in the SQL? → move it to a
        PARTIAL predicate and drop it from the key entirely (Topic 13)

WHEN TO BUILD ONE COMPOSITE vs SEVERAL SINGLES:
  ✓ COMPOSITE when queries filter on the columns TOGETHER, always
  ✓ COMPOSITE when one column is always present (it becomes the prefix)
  ✓ SINGLES when queries filter on the columns INDEPENDENTLY and the
    planner's BitmapAnd is good enough (rare — measure it)
  → Default to composite. It is almost always fewer indexes AND faster.

WHEN TWO QUERIES WANT DIFFERENT ORDERS:
  1. Can one be a PARTIAL index? (a constant predicate) → usually yes,
     and it's tiny
  2. Can one tolerate a Filter? Compute Rows Removed × call rate
  3. Can one tolerate a Sort? Only if there's no LIMIT
  4. Otherwise: two indexes. That's a legitimate answer — just price it.

THE SIGNAL TO LOOK FOR:
      EXPLAIN (ANALYZE, BUFFERS) <your query>

  Three things, in priority order:
  ① Is every WHERE column in `Index Cond`?
       Any column in `Filter` is in the wrong position → move it before
       the range column.
  ② Is there a `Sort` node above the scan?
       Your ORDER BY isn't satisfied → add those columns after the
       equalities, with matching directions.
  ③ Is `Rows Removed by Filter` large?
       Multiply by the call rate. That's your daily waste.

  If ① and ② are clean and buffers are still high → the problem is
  heap fetches (Topic 12: add INCLUDE) or selectivity (Topic 15).
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build the 8M-row `orders` table. Create `(status, created_at, merchant_id)` and run the Example 1 query with `EXPLAIN (ANALYZE, BUFFERS)`. Write down which predicates landed in `Index Cond` vs `Filter`, and explain — using the sort order of the leaf level — exactly why each one ended up where it did. Then predict the `Rows Removed by Filter` for `(merchant_id, created_at, status)` *before* creating it.

### Exercise 2 — medium (apply it)
You have three queries on `orders`:
```sql
Q1: WHERE user_id=$1 AND status=$2 ORDER BY created_at DESC LIMIT 20
Q2: WHERE user_id=$1 ORDER BY created_at DESC LIMIT 50
Q3: WHERE status='awaiting' AND created_at < now() - interval '1 hour'
```
(a) Derive the ideal index for each using the four-step procedure.
(b) Determine the minimum number of indexes that serves all three well, and justify every compromise.
(c) Build them, measure all three, and report `Index Cond` / `Filter` / `Sort` for each.
(d) For any query you compromised on, compute the cost: `Rows Removed by Filter × call rate`, assuming Q1=3,000/s, Q2=800/s, Q3=200/s.

### Exercise 3 — hard (production simulation)
A `events` table, 1.4 billion rows, 620 GB, has eleven indexes totalling 380 GB. Five hot queries:

```sql
Q1 (18,000/s): WHERE tenant_id=$1 AND kind=$2 AND occurred_at >= $3
               ORDER BY occurred_at DESC LIMIT 100
Q2 ( 6,000/s): WHERE tenant_id=$1 AND actor_id=$2
               ORDER BY occurred_at DESC LIMIT 50
Q3 ( 2,000/s): WHERE tenant_id=$1 AND occurred_at BETWEEN $2 AND $3
               AND severity >= 3 ORDER BY severity DESC, occurred_at DESC
Q4 (   400/s): WHERE kind='alert' AND acknowledged = false
               ORDER BY occurred_at ASC LIMIT 200
Q5 (     2/h): WHERE occurred_at >= $1 AND occurred_at < $2  -- export
```
`kind` has 40 distinct values; `severity` is 1–5; `acknowledged` is 99.4% true; `tenant_id` has 40,000 values with one tenant holding 38% of all rows.

(a) Derive the ideal index for each query with the four-step procedure, showing your reasoning at each step.
(b) Find the minimum index set. State every compromise and quantify it (`Rows Removed × rate`).
(c) Q3 sorts by `severity DESC, occurred_at DESC` but ranges on `occurred_at`. Explain why you cannot satisfy both the range and the sort with one index, and give two designs with their trade-offs.
(d) Q4's `acknowledged = false` is 0.6% of rows. Design the index that exploits this and explain why it's 100× smaller than the alternative.
(e) One tenant is 38% of the table. Explain what this does to the planner's row estimates for `tenant_id = <that tenant>` versus a small tenant, and what mechanism handles it (name the statistic).
(f) Q5 has no useful index. Justify having none, and say where the query should run instead.
(g) Estimate total index size before and after, and the change in write amplification per insert.

---

## Mental model checkpoint

1. Why can't `(a, b, c)` serve `WHERE b = 5`? Explain from the physical sort order, not the rule.
2. State the equality-then-range rule and explain the mechanism — why does a column after a range predicate only filter?
3. Is "most selective column first" correct for equality predicates? Explain your answer, and say what *should* determine the order.
4. `(a ASC, b DESC)` — which four `ORDER BY` clauses does it satisfy without a Sort, and which doesn't it?
5. You see `Index Only Scan using idx_abc` with 42,000 buffers read. What's happening, and how do you tell this apart from a real lookup?
6. When does a column belong in the key vs in `INCLUDE`? What does the choice do to tree height?
7. When is a composite index worse than two single-column indexes? Give a concrete case.

---

## Quick reference card

**The four-step procedure**

| Step | Columns | Why |
|---|---|---|
| 1 | Equality (`=`, `IN`, `IS NULL`) | only these narrow subsequent columns |
| 2 | (order among them) by **reuse**, not selectivity | determines which prefixes are available |
| 3 | `ORDER BY` columns, matching directions | deletes the Sort node, enables early LIMIT |
| 4 | The range column, **last** | only one can narrow; the rest filter |
| 5 | Output-only columns → `INCLUDE` | index-only scan, narrow internal nodes |
| 6 | Constant equality → `WHERE` predicate | partial index; column leaves the key |

**The rules**

| Rule | |
|---|---|
| Leftmost prefix | `(a,b,c)` serves `(a)`, `(a,b)`, `(a,b,c)` — never `(b)` |
| Equality then range | everything after the first inequality is a filter |
| ORDER BY | prefix only, matching **or exactly reversed** directions |
| One composite replaces N prefixes | biggest practical win: fewer indexes |
| `INCLUDE` ≠ key | not searchable, not sortable, but keeps fanout high |

**EXPLAIN checklist**

| See this | Means |
|---|---|
| `Index Cond:` has all your predicates | ✓ correct order |
| `Filter:` has one of them | ✗ that column is after the range column |
| `Rows Removed by Filter:` large | ✗ the size of your mistake |
| `Sort` node above the scan | ✗ ORDER BY not satisfied |
| `Index Only Scan` with huge buffers | ✗ full index scan, not a lookup |

---

## When would I use this at work?

1. **Any "the index isn't being used" ticket.** Nine times out of ten the index exists and *is* being used — just with the predicate in `Filter` instead of `Index Cond`. Reordering three identifiers fixes it, and the four-step procedure tells you the order without guessing.

2. **Index consolidation.** When a table accumulates one index per endpoint, applying the procedure to the access-pattern table typically halves the index count while making every query faster. That's the rare change that's a pure win on both reads and writes.

3. **Design review of a new endpoint.** Before the code merges, you can state the exact index it needs — including whether the constant predicate should become a partial index — and require the `EXPLAIN` output in the PR. That prevents the accumulation in point 2 from ever happening.

---

## Connected topics

**Understand before this:** 11 (B-tree sort order), 12 (`Index Cond` vs `Filter`, `INCLUDE`), 13 (partial indexes — often the better answer than a wider key).

**This unlocks:**
- **15** — selectivity and statistics: how the planner estimates what your composite will return
- **16** — non-B-tree access methods where these rules don't apply
- **17** — the write cost you avoid by having fewer, better indexes
- **18** — the planner's cost model for composite scans
- **19** — merge joins, which depend on index ordering
- **59** — partitioning, where the partition key interacts with composite key design
