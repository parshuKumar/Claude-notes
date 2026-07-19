# Topic 13 — FULL OUTER JOIN in Depth
### SQL Mastery Curriculum — Phase 3: JOINs — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Two teams ran the same event on different clipboards.

- **Clipboard A** — the *registration* list: everyone who signed up, with their name and the seat they reserved.
- **Clipboard B** — the *attendance* list: everyone who actually walked in, with their name and the time they scanned their badge.

Now you have to produce **one master sheet** for the post-event review. And critically — the review is about *discrepancies*. You do not want to lose anyone.

- Someone who **registered and attended** → one row, seat + scan time both filled in.
- Someone who **registered but never showed** → keep the row anyway. Seat is filled in; scan time is blank.
- Someone who **showed up but never registered** (walked in off the street) → keep that row too. Scan time is filled in; seat is blank.

That is a FULL OUTER JOIN. It keeps **every** row from **both** lists. Where a name exists on both clipboards, the two halves are stitched into one row. Where a name exists on only one clipboard, you still get the row — the missing half is filled with **NULL**.

Compare the earlier joins:
- **INNER JOIN** (Topic 11): only people on *both* clipboards. No-shows and walk-ins vanish.
- **LEFT JOIN** (Topic 12): everyone on clipboard A, plus matches from B. Walk-ins (B-only) still vanish.
- **RIGHT JOIN** (Topic 12): everyone on clipboard B, plus matches from A. No-shows (A-only) still vanish.
- **FULL OUTER JOIN**: *nobody* vanishes. It is `LEFT ∪ RIGHT` in one pass.

The whole reason FULL OUTER JOIN exists is the phrase "I cannot afford to lose a row from either side." That is a surprisingly rare requirement in day-to-day CRUD — which is exactly why this is a LOW-priority interview topic. But when you're **reconciling two sources of truth**, it is the single cleanest tool in SQL.

---

## 2. Connection to SQL Internals

FULL OUTER JOIN is the union of three disjoint row sets:

```
FULL OUTER JOIN (A, B) =
    { (a, b) | a ∈ A, b ∈ B, ON(a,b) = TRUE }        -- the inner matches
  ∪ { (a, NULL) | a ∈ A, no b ∈ B with ON(a,b)=TRUE } -- unmatched left rows
  ∪ { (NULL, b) | b ∈ B, no a ∈ A with ON(a,b)=TRUE } -- unmatched right rows
```

Two properties fall out of this definition and drive everything else in the doc:

1. **NULLs appear on *both* sides.** In a LEFT JOIN, only the right-side columns are ever NULL-extended. In a FULL OUTER JOIN, *either* side's columns can be entirely NULL for a given output row. This is the single most important mechanical fact — it breaks the usual "check for NULL on the right to find non-matches" trick, because non-matches live on both sides.

2. **The engine must track which right-side rows were matched.** For a LEFT JOIN, the engine walks the left side and emits NULL-extensions for left rows that found nothing. For FULL OUTER, it must *additionally* remember which right-side rows were **never** touched, and emit those at the end. This bookkeeping is what constrains the physical algorithms.

At the physical layer, PostgreSQL implements FULL OUTER JOIN with only **two** of its three join strategies:

1. **Hash Full Join** — builds a hash table from one side, probes with the other, and crucially marks hash-table entries as "matched." After the probe pass, it makes a second pass over the hash table emitting any **unmarked** entries as NULL-extended rows. This requires the hashed side to be materialized and mark-capable.
2. **Merge Full Join** — sorts both inputs on the join key (or reads them pre-sorted from indexes) and walks both streams in lockstep. When one stream is ahead, the lagging stream's rows are emitted NULL-extended. Merge naturally sees "gaps" on both sides, so it handles FULL cleanly.

**Nested Loop cannot do a FULL OUTER JOIN.** A nested loop drives from an outer relation into an inner one; it can NULL-extend the outer side (that's how it does LEFT/RIGHT), but it has no efficient way to discover inner rows that *no* outer row matched without a second full scan. So the planner will never emit a `Nested Loop Full Join` node. This is a real constraint: FULL OUTER JOIN **requires an equi-join** to use Hash, or **sort-friendly inputs** to use Merge. A non-equi FULL OUTER JOIN (e.g. `ON a.x < b.y`) can only be done by Merge with a mergeable condition — and if the condition isn't mergeable, the planner simply **cannot plan the query** and throws:

```
ERROR:  FULL JOIN is only supported with merge-joinable or hash-joinable join conditions
```

Memory-wise, Hash Full Join needs `work_mem` proportional to the hashed side (like any hash join), plus the match-mark bitmap. Merge Full Join needs sort memory unless both sides arrive pre-sorted from indexes. These costs are covered in Sections 7 and 10.

---

## 3. Logical Execution Order Context

```
FROM tableA
FULL OUTER JOIN tableB ON condition   ← evaluated in the FROM/JOIN phase
WHERE ...                              ← applied AFTER the join, to the NULL-extended result
GROUP BY ...
HAVING ...
SELECT ...
DISTINCT ...
ORDER BY ...
LIMIT ...
```

The ordering rule that mattered so much for LEFT JOIN (Topic 12) matters **even more** here, and in both directions.

**The ON vs WHERE trap — now symmetric.** For an INNER JOIN, a predicate in `ON` and the same predicate in `WHERE` are interchangeable. For a LEFT JOIN, putting a filter on the *right* table in `WHERE` silently converts it into an INNER JOIN (because NULL-extended rows fail the `WHERE` and get dropped). For a **FULL OUTER JOIN**, a `WHERE` predicate on **either** table can demote the join:

- A `WHERE` predicate on the **left** table drops the right-only (NULL-left) rows → the FULL becomes a **RIGHT-side-filtered LEFT-ish** join.
- A `WHERE` predicate on the **right** table drops the left-only (NULL-right) rows → the FULL becomes a **LEFT-side-filtered RIGHT-ish** join.
- A `WHERE` predicate that touches columns from **both** tables can drop *all* three categories of NULL-extended rows, collapsing the whole thing back to an INNER JOIN.

```sql
-- This LOOKS like a full outer join but behaves like an inner join:
SELECT a.id, b.id
FROM a
FULL OUTER JOIN b ON b.a_id = a.id
WHERE a.status = 'active'      -- kills every (NULL, b) right-only row
  AND b.status = 'active';     -- kills every (a, NULL) left-only row
-- Net effect: only fully-matched rows where BOTH statuses are 'active' survive.
-- You wrote FULL OUTER JOIN and got INNER JOIN semantics.
```

The fix is the same as for LEFT JOIN: **filters that should not eliminate unmatched rows belong in the `ON` clause**, or must be written NULL-tolerantly in `WHERE`:

```sql
-- Keep the full-outer semantics: move the predicates into ON...
FULL OUTER JOIN b
  ON b.a_id = a.id
 AND a.status = 'active'
 AND b.status = 'active'
-- ...or make WHERE NULL-tolerant:
WHERE (a.status = 'active' OR a.id IS NULL)
  AND (b.status = 'active' OR b.id IS NULL)
```

Because the join produces the NULL-extended rows *before* `WHERE` runs, `WHERE` is also where the reconciliation/diff pattern lives — you deliberately keep only the rows where one side IS NULL (the mismatches). More on that in Section 6.

---

## 4. What Is FULL OUTER JOIN?

FULL OUTER JOIN returns every row from the left table and every row from the right table. Where the ON condition matches, the left and right rows are combined into one output row. Where a row on either side has no match, it is still returned, with the columns of the other table filled with NULL. It is the set union of what a LEFT JOIN and a RIGHT JOIN on the same condition would each produce.

```sql
SELECT column_list
FROM table_a a
FULL [OUTER] JOIN table_b b ON a.key = b.key;
│                              │
│                              └── join predicate; must be merge- or hash-joinable
└── FULL is required; OUTER is optional noise — "FULL JOIN" == "FULL OUTER JOIN"
```

Annotated breakdown:

```sql
SELECT
  a.id   AS a_id,        -- NULL when this output row is a right-only (unmatched-B) row
  b.id   AS b_id,        -- NULL when this output row is a left-only  (unmatched-A) row
  a.name,                -- from the left table; NULL on right-only rows
  b.amount               -- from the right table; NULL on left-only rows
FROM table_a a           -- LEFT input; ALL of its rows are preserved
FULL OUTER JOIN          -- ↑ keyword: preserve unmatched rows from BOTH inputs
  table_b b              -- RIGHT input; ALL of its rows are preserved
  ON a.key = b.key;      -- match condition
│    │        │
│    │        └── right-side key
│    └── left-side key
└── evaluated during the JOIN phase, producing NULL-extended rows on both sides
```

### Exact Semantics

Given:

```
table_a:              table_b:
id | name             a_id | amount
---+--------          -----+-------
1  | Alice            1    | 100
2  | Bob              1    | 250
3  | Charlie          4    | 900     ← a_id=4 has no matching row in table_a
                      NULL | 50      ← NULL key, matches nothing
```

```sql
SELECT a.id AS a_id, a.name, b.a_id AS b_key, b.amount
FROM table_a a
FULL OUTER JOIN table_b b ON b.a_id = a.id;
```

```
a_id | name    | b_key | amount
-----+---------+-------+-------
1    | Alice   | 1     | 100     ← matched (Alice ↔ 100)
1    | Alice   | 1     | 250     ← matched (Alice ↔ 250) — one-to-many, Alice appears twice
2    | Bob     | NULL  | NULL    ← left-only: Bob has no row in table_b
3    | Charlie | NULL  | NULL    ← left-only: Charlie has no row in table_b
NULL | NULL    | 4     | 900     ← right-only: a_id=4 has no row in table_a
NULL | NULL    | NULL  | 50      ← right-only: NULL key never matches, still preserved
```

Read the six rows carefully — they are the entire behaviour of FULL OUTER JOIN:

- **Matched rows** behave exactly like INNER JOIN, including one-to-many multiplication (Alice → 2 rows).
- **Left-only rows** (Bob, Charlie) behave exactly like the "extra" rows a LEFT JOIN adds — right columns NULL.
- **Right-only rows** (`a_id=4`, and the NULL-key row) behave exactly like the "extra" rows a RIGHT JOIN adds — left columns NULL.
- **A NULL join key** matches nothing (three-valued logic, Topic 07: `NULL = anything` is UNKNOWN), so its row survives as an *unmatched* row on whichever side it lives — it is **preserved**, not dropped. Contrast INNER JOIN, where the NULL-key row is silently *dropped*.

The output row count is therefore:

```
rows(FULL OUTER) = rows(INNER matches)
                 + count(left rows with no match)
                 + count(right rows with no match)
```

which is exactly `rows(LEFT JOIN) + count(right rows with no match)`, or equivalently `rows(LEFT) + rows(RIGHT) − rows(INNER)`.

---

## 5. Why FULL OUTER JOIN Mastery Matters in Production

FULL OUTER JOIN is the least-used of the four core joins — but the situations where it *is* the right tool are high-stakes ones, and the situations where a developer *reaches for it wrongly* are common. Mastery means both knowing the handful of real use cases and recognising when someone has used it as a crutch.

1. **Reconciliation is a correctness problem, not a convenience.** When you compare "what the ledger says" against "what the payment processor says," or "expected inventory" against "counted inventory," dropping a row means hiding a discrepancy. A LEFT JOIN hides rows that exist only in the right source; a FULL OUTER JOIN is the only join that surfaces mismatches on *both* sides in a single query. Getting this wrong means a financial break goes unnoticed.

2. **The NULL-on-both-sides mental model prevents silent bugs.** Every downstream `SELECT`, `WHERE`, `COALESCE`, and `GROUP BY` on a FULL OUTER JOIN must account for the fact that *any* column can be NULL. Grouping by `a.id` alone loses the right-only rows into a single NULL bucket. Filtering by `a.status` silently demotes the join. Engineers who don't internalise this ship reports that quietly under-count.

3. **Knowing it is expensive prevents its overuse.** FULL OUTER JOIN forbids Nested Loop, forbids non-mergeable/hashable conditions, and often materializes both sides. On large tables it is one of the more expensive joins you can write. A senior engineer knows to ask "do I actually need both-sided preservation, or would a LEFT JOIN (or a `UNION` of two targeted queries) be cheaper and clearer?"

4. **It reveals data-model problems.** If a FULL OUTER JOIN between two tables that *should* be in perfect correspondence returns any NULL-extended rows, you have a referential-integrity or ETL-sync bug. FULL OUTER JOIN is a diagnostic instrument, not just a query.

5. **Interviewers use it to test join reasoning, not memorisation.** Because it's rarely used, being fluent in it signals you understand *all* joins deeply — you can derive its row count, predict its NULL pattern, and explain why Nested Loop can't implement it. That's a strong senior signal even though the operator itself is niche.

---

## 6. Deep Technical Content

### 6.1 NULLs on Both Sides — The Defining Behaviour

The single fact that separates FULL OUTER JOIN from every other join: **a given output row may have all-NULL left columns, or all-NULL right columns.** You can no longer assume "the driving table's columns are always populated."

```sql
SELECT
  e.id   AS emp_id,       -- NULL for department rows that have no employee
  e.name AS emp_name,     -- NULL likewise
  d.id   AS dept_id,      -- NULL for employees with no (matching) department
  d.name AS dept_name     -- NULL likewise
FROM employees e
FULL OUTER JOIN departments d ON d.id = e.department_id;
```

Three categories of row come back:
- **Matched**: `emp_id` and `dept_id` both non-NULL.
- **Employee with no department** (or a department_id pointing nowhere): `dept_id`/`dept_name` NULL.
- **Department with no employees**: `emp_id`/`emp_name` NULL.

To *classify* which category a row is in, you cannot test the join key against a literal — you test whether each side's key is NULL:

```sql
SELECT
  CASE
    WHEN e.id IS NOT NULL AND d.id IS NOT NULL THEN 'matched'
    WHEN e.id IS NOT NULL AND d.id IS NULL     THEN 'employee_orphan'
    WHEN e.id IS NULL     AND d.id IS NOT NULL THEN 'empty_department'
  END AS row_kind,
  COALESCE(e.name, '(no employee)')   AS emp_name,
  COALESCE(d.name, '(no department)') AS dept_name
FROM employees e
FULL OUTER JOIN departments d ON d.id = e.department_id;
```

**Gotcha:** use a **non-nullable** column (usually the primary key) as your "is this side present?" probe. If you test a *nullable* column — say `WHEN e.middle_name IS NULL` — you'll misclassify matched rows whose `middle_name` happens to be NULL. Always probe on the PK.

### 6.2 The Reconciliation / Diff Pattern — The Reason FULL OUTER JOIN Exists

This is the flagship use case. You have two datasets that *should* agree, and you want every discrepancy: rows only in A, rows only in B, and rows in both whose values differ.

```sql
-- Reconcile our internal ledger against the payment processor's settlement file.
-- ledger(txn_ref, amount_cents)        — what our system recorded
-- settlements(txn_ref, amount_cents)   — what the processor says it settled
SELECT
  COALESCE(l.txn_ref, s.txn_ref)  AS txn_ref,
  l.amount_cents                  AS ledger_amount,
  s.amount_cents                  AS settled_amount,
  CASE
    WHEN l.txn_ref IS NULL                     THEN 'missing_in_ledger'
    WHEN s.txn_ref IS NULL                     THEN 'missing_in_settlement'
    WHEN l.amount_cents <> s.amount_cents      THEN 'amount_mismatch'
    ELSE 'ok'
  END AS status
FROM ledger l
FULL OUTER JOIN settlements s ON s.txn_ref = l.txn_ref
WHERE l.txn_ref IS NULL          -- only in settlements → we never recorded it
   OR s.txn_ref IS NULL          -- only in ledger → processor never settled it
   OR l.amount_cents <> s.amount_cents;  -- both exist but disagree
```

Notes on why this works and why each piece is necessary:

- **`FULL OUTER JOIN`** is mandatory — a LEFT JOIN would miss `missing_in_ledger` rows (present only in settlements); a RIGHT JOIN would miss `missing_in_settlement`.
- **`COALESCE(l.txn_ref, s.txn_ref)`** reconstructs the key regardless of which side is NULL. The join key is your identity; you must surface it from whichever side is populated.
- **The `WHERE`** deliberately keeps only the discrepancies. Note it is written NULL-tolerantly — `l.txn_ref IS NULL OR s.txn_ref IS NULL OR ...` — so it does **not** demote the join before the mismatch logic runs. (If you wrote `WHERE l.amount_cents <> s.amount_cents` alone, the `<>` is UNKNOWN when either side is NULL, so the missing-row cases would be dropped — the exact bug FULL OUTER JOIN was meant to catch.)
- **The `<>` comparison** itself is NULL-unsafe on purpose here because the `IS NULL` branches already handle the missing cases. If instead you wanted "values differ *including* the missing cases folded together," use `IS DISTINCT FROM` (Section 6.3).

This diff pattern generalises to: config-drift detection (desired vs actual state), inventory audits (system count vs physical count), data-migration validation (old table vs new table), and cross-region replication checks (primary vs replica snapshot).

### 6.3 `IS DISTINCT FROM` — NULL-Safe Comparison for Diffs

When comparing values that may themselves be NULL, plain `<>` returns UNKNOWN and the row escapes your mismatch filter. `IS DISTINCT FROM` treats NULL as a comparable value: `NULL IS DISTINCT FROM 5` is TRUE, `NULL IS DISTINCT FROM NULL` is FALSE.

```sql
SELECT
  COALESCE(a.k, b.k) AS k,
  a.val AS a_val,
  b.val AS b_val
FROM snapshot_a a
FULL OUTER JOIN snapshot_b b ON b.k = a.k
WHERE a.val IS DISTINCT FROM b.val;   -- catches every difference, NULLs included:
--   a.val=5,  b.val=5    → NOT distinct → excluded (they agree)
--   a.val=5,  b.val=7    → distinct     → included (value mismatch)
--   a.val=5,  b.val=NULL → distinct     → included (right-only row)
--   a.val=NULL,b.val=NULL→ NOT distinct → excluded (both genuinely null, agree)
```

This single predicate replaces the three-way `l.txn_ref IS NULL OR s.txn_ref IS NULL OR l.amount <> s.amount` when you want the *value* column compared NULL-safely. It is the idiomatic diff filter and pairs with FULL OUTER JOIN constantly.

### 6.4 Reconstructing the Key with COALESCE — and Its Traps

Because either side can be NULL, the join key must be rebuilt from both:

```sql
COALESCE(a.id, b.a_id) AS id
```

Traps:
- **Type must match.** `a.id` and `b.a_id` must be the same (or coercible) type; COALESCE returns the common type.
- **Don't COALESCE two *different* semantic keys.** `COALESCE(a.email, b.phone)` is nonsense. Only coalesce the two representations of the *same* identity.
- **GROUP BY must use the coalesced key, not one side.** See 6.6.
- **Ordering:** `ORDER BY COALESCE(a.id, b.a_id)` sorts by reconstructed identity; ordering by `a.id` alone dumps all right-only rows (NULL a.id) together at one end.

### 6.5 Simulating FULL OUTER JOIN Where It Isn't Supported (MySQL and friends)

MySQL (through 8.x) does **not** support `FULL OUTER JOIN`. The canonical workaround — worth knowing because interviewers ask it and because you'll port queries — is `LEFT JOIN UNION RIGHT-anti-join`:

```sql
-- Portable FULL OUTER JOIN emulation:
SELECT a.id AS a_id, a.name, b.a_id AS b_key, b.amount
FROM a
LEFT JOIN b ON b.a_id = a.id           -- all A rows + matches (covers matched + left-only)

UNION ALL

SELECT a.id, a.name, b.a_id, b.amount
FROM a
RIGHT JOIN b ON b.a_id = a.id          -- all B rows + matches
WHERE a.id IS NULL;                     -- keep ONLY right-only rows (avoid double-counting matches)
```

Key point: the second arm filters `WHERE a.id IS NULL` so matched rows (already emitted by the LEFT JOIN arm) aren't emitted twice. Use `UNION ALL` (not `UNION`) once you've de-duplicated this way — `UNION` would pay for an unnecessary distinct sort, and could wrongly collapse legitimately-identical data rows. PostgreSQL supports FULL OUTER JOIN natively, so you never need this here — but understanding *why* the emulation works cements what the native operator does.

### 6.6 Aggregation Over a FULL OUTER JOIN — The GROUP BY Key Trap

Grouping after a FULL OUTER JOIN is where correctness quietly slips. If you `GROUP BY a.id`, every right-only row has `a.id = NULL` and they **all collapse into a single NULL group** — you've merged unrelated records.

```sql
-- WRONG: right-only rows all fall into one NULL bucket
SELECT a.id, COUNT(*), SUM(b.amount)
FROM a
FULL OUTER JOIN b ON b.a_id = a.id
GROUP BY a.id;         -- ← all (NULL, b) rows grouped together under a.id = NULL

-- RIGHT: group by the reconstructed identity
SELECT COALESCE(a.id, b.a_id) AS id, COUNT(*), SUM(b.amount)
FROM a
FULL OUTER JOIN b ON b.a_id = a.id
GROUP BY COALESCE(a.id, b.a_id);
```

The same discipline applies to `DISTINCT`, window `PARTITION BY`, and `ORDER BY`: operate on the coalesced key, never on one raw side.

### 6.7 One-to-Many and Many-to-Many Under FULL OUTER JOIN

Matched rows multiply exactly as in INNER JOIN (Topic 11 §6.2–6.3). A left row matching 3 right rows produces 3 matched output rows. FULL OUTER JOIN adds unmatched rows *on top* of that multiplication; it does not change the multiplication itself. So the fan-out reasoning is:

```
total rows = Σ(matches per left row, min 1 to preserve unmatched left)
           + count(right rows that matched no left row)
```

Concretely, a left row with 0 matches contributes 1 (NULL-extended) row; a left row with k≥1 matches contributes k rows; then every never-matched right row adds 1 more. This makes FULL OUTER JOIN over two many-sided tables genuinely explosive and rarely what you want — reconciliation is almost always **one-to-one on a key**, which is the well-behaved case.

### 6.8 Chaining FULL OUTER JOINs Across Three or More Sources

Reconciling three sources (e.g. ledger vs processor vs bank) chains FULL OUTER JOINs — but the key-coalescing compounds and must be done at each step:

```sql
SELECT
  COALESCE(l.ref, p.ref, k.ref) AS ref,
  l.amount AS ledger_amt,
  p.amount AS processor_amt,
  k.amount AS bank_amt
FROM ledger l
FULL OUTER JOIN processor p ON p.ref = l.ref
FULL OUTER JOIN bank k
  ON k.ref = COALESCE(l.ref, p.ref)   -- ← MUST coalesce; a ref present only in p
                                       --   would be NULL in l, so ON k.ref = l.ref misses it
GROUP BY 1, l.amount, p.amount, k.amount;
```

The subtle bug: the second FULL OUTER JOIN's `ON` must match against `COALESCE(l.ref, p.ref)`, not `l.ref` alone. A reference that exists only in `processor` has `l.ref = NULL` at that point; joining `bank` on `l.ref` would fail to line it up. This is a classic senior-level gotcha — three-way reconciliation is where naive chaining breaks.

### 6.9 FULL OUTER JOIN vs UNION — Different Shapes for Different Jobs

Both combine two datasets, but along different axes:
- **UNION** stacks rows **vertically** — same columns, more rows. Use it to concatenate compatible result sets.
- **FULL OUTER JOIN** stitches rows **horizontally** on a key — matching rows become wider (both sides' columns), non-matching rows are NULL-padded.

You reach for FULL OUTER JOIN when you need side-by-side comparison of two sources keyed on a shared identity. You reach for UNION when you're appending like-shaped rows. Confusing the two is a common design error: people UNION two sources and then can't tell which source a row came from or spot per-key disagreements.

### 6.10 Why FULL OUTER JOIN Is Rarely Used — An Honest Accounting

It deserves stating plainly, because interviewers probe it:
- **Most schemas have a directional relationship.** Orders belong to customers; you drive from one to the other with LEFT/INNER. Bidirectional "both may lack the other" relationships are uncommon in normalized OLTP.
- **Reconciliation is periodic, not per-request.** The main legitimate use is batch data-quality/finance jobs, not the hot path of an app. So it lives in analytics and ETL, not in the API layer most developers work in daily.
- **It's expensive and constrained** (no Nested Loop, needs hashable/mergeable conditions) — so even where it *could* be used, engineers often prefer a cheaper LEFT JOIN plus a targeted anti-join.
- **ORMs barely support it** (Section 12) — pushing developers toward raw SQL, which further limits casual use.

That rarity is the *reason it's a LOW-priority interview topic* — but the reconciliation pattern it enables is one of the highest-value queries a data engineer writes.

---

## 7. EXPLAIN — FULL OUTER JOIN in the Plan

### Hash Full Join (equi-join, the common case)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT COALESCE(l.txn_ref, s.txn_ref) AS ref, l.amount_cents, s.amount_cents
FROM ledger l
FULL OUTER JOIN settlements s ON s.txn_ref = l.txn_ref;
```

```
Hash Full Join  (cost=3084.00..8140.00 rows=205000 width=48)
                (actual time=41.2..212.7 rows=204880 loops=1)
  Hash Cond: (s.txn_ref = l.txn_ref)
  ->  Seq Scan on settlements s  (cost=0.00..2710.00 rows=150000 width=24)
                                  (actual time=0.02..22.4 rows=150000 loops=1)
  ->  Hash  (cost=1540.00..1540.00 rows=100000 width=24)
             (actual time=40.8..40.8 rows=100000 loops=1)
        Buckets: 131072  Batches: 1  Memory Usage: 5890kB
        ->  Seq Scan on ledger l  (cost=0.00..1540.00 rows=100000 width=24)
                                    (actual time=0.01..12.1 rows=100000 loops=1)
Buffers: shared hit=4250
Planning Time: 0.4 ms
Execution Time: 224.9 ms
```

**Reading it:**
- **`Hash Full Join`** — note the node type. Not `Hash Join`; the `Full` tells you the engine is doing the extra bookkeeping. It builds a hash of `ledger` (the smaller side → hash side), probes with `settlements`, marks matched hash entries, then does a final sweep emitting unmatched `ledger` entries as NULL-extended.
- **`Batches: 1`** — the whole hash fits in `work_mem` (5.9 MB used). If `Batches > 1`, the hash spilled to disk — expensive, and for a FULL join especially painful because the match-marking must survive the spill. Bump `work_mem`.
- **`rows=204880`** ≈ 100k matched-or-left + 104,880 right-only. The actual matches both sides, then adds the never-matched settlements — that's why the output exceeds either input.
- No `Nested Loop` option exists here; if you forced one off you'd still get Hash or Merge, never NL.

### Merge Full Join (both sides indexed / pre-sorted on key)

```sql
EXPLAIN (ANALYZE)
SELECT COALESCE(a.id, b.a_id) AS id, a.name, b.amount
FROM snapshot_a a
FULL OUTER JOIN snapshot_b b ON b.a_id = a.id;
```

```
Merge Full Join  (cost=0.85..9200.00 rows=210000 width=44)
                 (actual time=0.08..180.3 rows=209500 loops=1)
  Merge Cond: (a.id = b.a_id)
  ->  Index Scan using snapshot_a_pkey on snapshot_a a
        (actual time=0.03..48.1 rows=100000 loops=1)
  ->  Index Scan using snapshot_b_a_id_idx on snapshot_b b
        (actual time=0.03..70.6 rows=150000 loops=1)
Planning Time: 0.3 ms
Execution Time: 192.0 ms
```

**Reading it:**
- **`Merge Full Join`** — both inputs arrive pre-sorted from index scans, so **no Sort node** is needed. Merge walks both streams; when one key stream advances past the other, the lagging side's rows are emitted NULL-extended. Merge sees gaps on both sides naturally, which is why it can do FULL cleanly.
- No hash table → no `work_mem` pressure for the join itself. Good for very large inputs where a hash wouldn't fit.
- If the inputs were **not** pre-sorted, you'd see `Sort` nodes above each Seq Scan, adding O(N log N) — and then Hash Full Join is often cheaper.

### The Error You Must Recognise

```sql
EXPLAIN
SELECT * FROM a FULL OUTER JOIN b ON a.x < b.y;   -- non-equi condition
```

```
ERROR:  FULL JOIN is only supported with merge-joinable or hash-joinable join conditions
```

`a.x < b.y` is neither hashable (hash needs equality) nor mergeable in a way that supports FULL preservation, so the planner **refuses to plan the query at all**. This is not a runtime slowness — it's a hard parse/plan error. Recognising it instantly marks you as someone who understands the physical constraints from Section 2.

### What to Look For

| Symptom | Meaning | Action |
|---------|---------|--------|
| `Hash Full Join` with `Batches > 1` | Hash side spilled to disk | Increase `work_mem`; hash the smaller side |
| `Sort` nodes under `Merge Full Join` | Inputs not pre-sorted | Add indexes on both join keys |
| `ERROR: FULL JOIN is only supported with merge/hash-joinable...` | Non-equi ON condition | Rewrite as equi-join, or emulate via UNION |
| Output rows ≫ both inputs | Fan-out from many-to-many matching | Reconcile on a unique key, not a many-side column |
| Right-only rows collapsing in GROUP BY | Grouped by one raw side | `GROUP BY COALESCE(a.k, b.k)` |

---

## 8. Query Examples

### Example 1 — Basic: Employees ↔ Departments, Lose Nobody

```sql
-- Show every employee AND every department, even orphans on either side.
SELECT
  e.id        AS employee_id,    -- NULL for empty departments
  e.name      AS employee_name,  -- NULL for empty departments
  d.id        AS department_id,  -- NULL for employees with no valid department
  d.name      AS department_name -- NULL likewise
FROM employees e
FULL OUTER JOIN departments d ON d.id = e.department_id
ORDER BY department_name NULLS LAST, employee_name NULLS LAST;
-- Rows come in three flavours: matched, employee-with-no-dept, dept-with-no-employees.
```

### Example 2 — Intermediate: Reconciliation with Classification

```sql
-- Compare our order records against the warehouse's shipment records.
-- orders(id, ...)          — an order should exist for every shipment
-- shipments(order_id, ...) — a shipment should exist for every fulfilled order
SELECT
  COALESCE(o.id, s.order_id) AS order_id,
  CASE
    WHEN o.id IS NULL THEN 'shipment_without_order'   -- warehouse shipped a phantom
    WHEN s.order_id IS NULL THEN 'order_without_shipment' -- we owe a shipment
    ELSE 'reconciled'
  END AS state
FROM orders o
FULL OUTER JOIN shipments s ON s.order_id = o.id
WHERE o.status = 'fulfilled' OR o.id IS NULL   -- NULL-tolerant: don't drop right-only rows
ORDER BY state, order_id;
```

### Example 3 — Production Grade: Daily Payments Reconciliation

**Context:** `payments` holds ~2M rows/month (our system's record of charges). `processor_settlements` holds ~2M rows/month (the PSP's settlement export, loaded nightly). Both are indexed on `(external_ref)`. We run this every morning to catch breaks before they age. Expectation: a Hash Full Join over ~2M × ~2M rows, ~1–3 s cold, sub-second warm; result is only the *breaks* (typically a few hundred rows), so the output is tiny even though the scan is large.

```sql
-- Indexes assumed:
--   CREATE INDEX ON payments(external_ref);
--   CREATE INDEX ON processor_settlements(external_ref);
--   Partial by day via WHERE on created_at / settled_on to bound the scan.
WITH our_side AS (
  SELECT external_ref, amount_cents, currency
  FROM payments
  WHERE created_at >= CURRENT_DATE - INTERVAL '1 day'
    AND status = 'captured'
),
their_side AS (
  SELECT external_ref, amount_cents, currency
  FROM processor_settlements
  WHERE settled_on = CURRENT_DATE - INTERVAL '1 day'
)
SELECT
  COALESCE(o.external_ref, t.external_ref) AS external_ref,
  o.amount_cents                            AS our_amount,
  t.amount_cents                            AS their_amount,
  o.currency                                AS our_currency,
  t.currency                                AS their_currency,
  CASE
    WHEN o.external_ref IS NULL              THEN 'unrecorded_settlement'  -- money moved, no payment row
    WHEN t.external_ref IS NULL              THEN 'unsettled_payment'      -- captured but PSP didn't settle
    WHEN o.amount_cents IS DISTINCT FROM t.amount_cents THEN 'amount_break'
    WHEN o.currency     IS DISTINCT FROM t.currency     THEN 'currency_break'
    ELSE 'ok'
  END AS break_type
FROM our_side o
FULL OUTER JOIN their_side t ON t.external_ref = o.external_ref
WHERE o.external_ref IS NULL
   OR t.external_ref IS NULL
   OR o.amount_cents IS DISTINCT FROM t.amount_cents
   OR o.currency     IS DISTINCT FROM t.currency
ORDER BY break_type, external_ref;
```

```
-- EXPLAIN (ANALYZE, BUFFERS) shape:
Sort  (cost=... rows=340 width=...) (actual time=1180..1180 rows=312 loops=1)
  Sort Key: (CASE ...), (COALESCE(o.external_ref, t.external_ref))
  ->  Hash Full Join  (cost=48200..96500 rows=340 width=...)
                      (actual time=520..1170 rows=312 loops=1)
        Hash Cond: (t.external_ref = o.external_ref)
        Filter: ((o.external_ref IS NULL) OR (t.external_ref IS NULL)
                 OR (o.amount_cents IS DISTINCT FROM t.amount_cents)
                 OR (o.currency IS DISTINCT FROM t.currency))
        Rows Removed by Filter: 1998xxx     -- the reconciled rows, filtered out
        ->  CTE Scan on their_side t  (actual rows=2000000 loops=1)
        ->  Hash  (actual rows=2000000 loops=1)
              Buckets: 2097152  Batches: 4  Memory Usage: ...   -- ← spilled! bump work_mem
              ->  CTE Scan on our_side o  (actual rows=2000000 loops=1)
Execution Time: ~1200 ms
```

Note the `Batches: 4` — at this volume the hash spills; raising `work_mem` for this maintenance session (`SET work_mem = '256MB';`) collapses it to `Batches: 1` and roughly halves runtime. The `Rows Removed by Filter` line confirms almost everything reconciled and only 312 real breaks survived.

---

## 9. Wrong → Right Patterns

### Wrong 1: A WHERE filter silently demotes FULL OUTER JOIN to INNER

```sql
-- WRONG: intended to reconcile, but this drops all unmatched rows on both sides
SELECT COALESCE(l.ref, s.ref) AS ref, l.amount, s.amount
FROM ledger l
FULL OUTER JOIN settlements s ON s.ref = l.ref
WHERE l.status = 'posted'          -- ← kills every right-only (l.* = NULL) row
  AND s.batch  = 'BATCH_2026_07_18'; -- ← kills every left-only  (s.* = NULL) row
-- Result: only fully-matched rows survive. You wrote FULL, got INNER.
-- The reconciliation reports ZERO breaks — and hides every real one.
```

**Why (execution level):** the join produces NULL-extended rows first; then `WHERE l.status = 'posted'` evaluates to UNKNOWN for right-only rows (`l.status` is NULL) → dropped. Same for `s.batch` on left-only rows. Both categories of "break" are eliminated before you ever see them.

```sql
-- RIGHT: put source-scoping predicates in the ON clause (or NULL-tolerant WHERE)
SELECT COALESCE(l.ref, s.ref) AS ref, l.amount, s.amount
FROM (SELECT * FROM ledger      WHERE status = 'posted')            l
FULL OUTER JOIN
     (SELECT * FROM settlements WHERE batch = 'BATCH_2026_07_18')   s
  ON s.ref = l.ref
WHERE l.ref IS NULL OR s.ref IS NULL OR l.amount IS DISTINCT FROM s.amount;
-- Scoping each source BEFORE the join preserves full-outer semantics.
```

### Wrong 2: GROUP BY on one raw side collapses right-only rows

```sql
-- WRONG: right-only rows all have a.id = NULL → merged into one bogus group
SELECT a.id, COUNT(*) AS n, SUM(b.qty) AS total_qty
FROM inventory_expected a
FULL OUTER JOIN inventory_counted b ON b.sku = a.sku
GROUP BY a.id;
-- Every counted-but-not-expected SKU lands in the single a.id = NULL bucket.
```

**Why:** `a.id` is NULL for all right-only rows; SQL groups all NULLs together, so unrelated "unexpected" SKUs are summed into one meaningless row.

```sql
-- RIGHT: group by the reconstructed key
SELECT COALESCE(a.sku, b.sku) AS sku, COUNT(*) AS n, SUM(b.qty) AS total_qty
FROM inventory_expected a
FULL OUTER JOIN inventory_counted b ON b.sku = a.sku
GROUP BY COALESCE(a.sku, b.sku);
```

### Wrong 3: Using `<>` for the value diff, losing the missing-row breaks

```sql
-- WRONG: <> is UNKNOWN when either side is NULL → missing rows escape the filter
SELECT COALESCE(a.k, b.k) AS k
FROM a FULL OUTER JOIN b ON b.k = a.k
WHERE a.val <> b.val;    -- only catches rows where BOTH sides exist and differ
-- Rows that exist only in a (b.val NULL) or only in b (a.val NULL) are DROPPED —
-- exactly the breaks you most wanted to find.
```

**Why:** `5 <> NULL` is UNKNOWN, not TRUE; `WHERE` keeps only TRUE. Every one-sided row fails.

```sql
-- RIGHT: IS DISTINCT FROM is NULL-safe and catches all three break categories
SELECT COALESCE(a.k, b.k) AS k, a.val, b.val
FROM a FULL OUTER JOIN b ON b.k = a.k
WHERE a.val IS DISTINCT FROM b.val;
```

### Wrong 4: Reaching for FULL OUTER JOIN when LEFT JOIN suffices

```sql
-- WRONG (wasteful): only orders matter; you don't need to preserve customer-only rows
SELECT o.id, c.name
FROM orders o
FULL OUTER JOIN customers c ON c.id = o.customer_id;
-- This forces a Hash/Merge Full Join and drags in every customer who never ordered,
-- which you then have to filter back out. Slower and semantically noisy.
```

**Why:** you only ever consume rows that have an order; the right-only (customer-with-no-order) rows are dead weight the engine still computes and materializes.

```sql
-- RIGHT: LEFT JOIN — preserve orders, attach customer if present
SELECT o.id, c.name
FROM orders o
LEFT JOIN customers c ON c.id = o.customer_id;
-- Cheaper (can use Nested Loop with an index), and expresses the real intent.
```

### Wrong 5: Non-equi FULL OUTER JOIN — unplannable

```sql
-- WRONG: range condition on a FULL OUTER JOIN
SELECT *
FROM events e
FULL OUTER JOIN windows w
  ON e.ts >= w.start_ts AND e.ts < w.end_ts;
-- ERROR: FULL JOIN is only supported with merge-joinable or hash-joinable join conditions
```

**Why:** the condition is neither an equality (hashable) nor mergeable in a FULL-preserving way; the planner cannot build a valid plan.

```sql
-- RIGHT: if you truly need both-sided preservation on a range, emulate via UNION,
-- or (usually) rethink — reconciliation should key on an equality, not a range.
SELECT e.id, w.name FROM events e LEFT JOIN windows w
  ON e.ts >= w.start_ts AND e.ts < w.end_ts
UNION ALL
SELECT NULL, w.name FROM windows w
WHERE NOT EXISTS (SELECT 1 FROM events e WHERE e.ts >= w.start_ts AND e.ts < w.end_ts);
```

---

## 10. Performance Profile

FULL OUTER JOIN is, per row of input, the **most expensive** of the four core joins, for structural reasons established in Section 2.

| Property | INNER | LEFT | FULL OUTER |
|----------|-------|------|-----------|
| Nested Loop available? | Yes | Yes | **No** |
| Requires hashable/mergeable ON? | No (NL handles anything) | No | **Yes** |
| Sides that can be NULL-extended | none | right | **both** |
| Extra bookkeeping | none | none | **match-mark + second pass** |

### Algorithm choice

| Algorithm | When chosen | Memory | Notes |
|-----------|------------|--------|-------|
| Hash Full Join | Equi-join, no useful sort order | O(smaller side) + mark bitmap | Second pass emits unmatched hashed rows |
| Merge Full Join | Both sides sorted on key (indexes) | O(1) for join (sort mem if not pre-sorted) | Scales to very large inputs without a big hash |

### Scaling behaviour

| Scale (per side) | Hash Full Join | Merge Full Join (pre-sorted) |
|------------------|----------------|------------------------------|
| 100K × 100K | ~100–250 ms, fits work_mem | ~150–250 ms, index scans |
| 1M × 1M | ~1–3 s; watch `Batches` (spill) | ~1–2 s if both indexed, else + big sorts |
| 10M × 10M | Several seconds; spill likely → raise work_mem, or partition by key range | Best option *if* both pre-sorted; otherwise sort cost dominates |
| 100M × 100M | Avoid as a single query; **partition/bucket by key** and reconcile per-bucket, or run as a batch job off-hours | Only viable with clustered/indexed order on both sides |

### Optimisation techniques specific to FULL OUTER JOIN

1. **Bound both sides first.** Reconciliation is almost always per-period. Filter *inside* CTEs/subqueries (`WHERE settled_on = :day`) so the Full Join operates on a day's slice, not the whole history. (And remember: filter *before* the join, never in an outer `WHERE`, to keep semantics — Section 3.)
2. **Reconcile on a unique key.** One-to-one keys keep output ≈ input size. FULL OUTER JOIN on many-sided columns explodes (Section 6.7).
3. **Give both keys an index.** Enables Merge Full Join (no big hash) and speeds the bounded scans. For hash, indexes on the filter columns keep the build side small.
4. **Raise `work_mem` for the session** running the reconciliation so the hash stays single-batch (`Batches: 1`). Do it per-session (`SET LOCAL work_mem`), not globally.
5. **Project only needed columns before the join.** Narrower rows → smaller hash / cheaper sort. Select just `(key, value)` in the CTEs.
6. **Partition huge reconciliations by key range** (e.g. `ref` hash-bucketed) and Full-Join bucket-by-bucket, so each hash fits in memory. This is how you reconcile hundreds of millions of rows without a monster hash.

---

## 11. Node.js Integration

### 11.1 Basic reconciliation query with `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function reconcilePaymentsForDay(day /* 'YYYY-MM-DD' */) {
  const { rows } = await pool.query(
    `WITH our_side AS (
       SELECT external_ref, amount_cents, currency
       FROM payments
       WHERE created_at::date = $1 AND status = 'captured'
     ),
     their_side AS (
       SELECT external_ref, amount_cents, currency
       FROM processor_settlements
       WHERE settled_on = $1
     )
     SELECT
       COALESCE(o.external_ref, t.external_ref) AS external_ref,
       o.amount_cents AS our_amount,
       t.amount_cents AS their_amount,
       CASE
         WHEN o.external_ref IS NULL THEN 'unrecorded_settlement'
         WHEN t.external_ref IS NULL THEN 'unsettled_payment'
         WHEN o.amount_cents IS DISTINCT FROM t.amount_cents THEN 'amount_break'
         ELSE 'ok'
       END AS break_type
     FROM our_side o
     FULL OUTER JOIN their_side t ON t.external_ref = o.external_ref
     WHERE o.external_ref IS NULL
        OR t.external_ref IS NULL
        OR o.amount_cents IS DISTINCT FROM t.amount_cents
     ORDER BY break_type, external_ref`,
    [day]                        // $1 — bind the day, never string-concatenate dates
  );
  return rows;                   // typically small: only the breaks
}
```

### 11.2 Handling NULL-on-either-side in JS

```javascript
// Every column can be null. Reconstruct identity and classify defensively.
function classifyBreak(row) {
  // external_ref is COALESCE'd server-side, so it's always present.
  const ourAmt   = row.our_amount;    // null => row exists only on processor side
  const theirAmt = row.their_amount;  // null => row exists only on our side
  return {
    ref: row.external_ref,
    kind: row.break_type,
    delta: (ourAmt != null && theirAmt != null) ? ourAmt - theirAmt : null,
    onlyOurs:  theirAmt == null,
    onlyTheirs: ourAmt == null,
  };
}
```

### 11.3 Raising work_mem for a heavy reconciliation, safely

```javascript
// Bump work_mem for THIS reconciliation only, on a dedicated connection.
async function reconcileHeavy(day) {
  const client = await pool.connect();
  try {
    await client.query('SET LOCAL work_mem = $1', ['256MB']); // SET LOCAL → scoped to txn
    await client.query('BEGIN');
    const { rows } = await client.query(/* the FULL OUTER JOIN query */, [day]);
    await client.query('COMMIT');
    return rows;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();  // connection returns to pool with default work_mem
  }
}
```

Note: `SET LOCAL` requires a transaction (it resets at COMMIT), which is why we wrap in BEGIN/COMMIT. This prevents a global `work_mem` bump from bloating every other query's memory.

### 11.4 A note on ORMs

No mainstream Node ORM has first-class FULL OUTER JOIN support (Section 12). In practice you will write reconciliation queries as **raw SQL** through `pool.query` / the ORM's raw escape hatch. This is normal and correct — reconciliation is analytical SQL, not entity CRUD, so bypassing the ORM's object mapper is the right call.

---

## 12. ORM Comparison

FULL OUTER JOIN is the join ORMs support **worst**, because it doesn't map onto the object-graph "load related entity" model ORMs are built around. Expect to drop to raw SQL almost everywhere.

### Prisma

**Can Prisma do FULL OUTER JOIN?** — **No**, not through its query API. Prisma's relation loading (`include`) only generates INNER or LEFT joins based on FK nullability. There is no full-outer relation concept.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// The ONLY way — raw SQL:
const breaks = await prisma.$queryRaw`
  SELECT COALESCE(l.txn_ref, s.txn_ref) AS txn_ref,
         l.amount_cents AS ledger_amount,
         s.amount_cents AS settled_amount
  FROM ledger l
  FULL OUTER JOIN settlements s ON s.txn_ref = l.txn_ref
  WHERE l.txn_ref IS NULL
     OR s.txn_ref IS NULL
     OR l.amount_cents IS DISTINCT FROM s.amount_cents
`;
```

**Where it breaks:** everything — there is no typed builder path. **Verdict:** `$queryRaw` only.

---

### Drizzle ORM

**Can Drizzle do FULL OUTER JOIN?** — **Yes**, it has an explicit `.fullJoin()` method. This is the best typed support among Node ORMs.

```typescript
import { db } from './db';
import { ledger, settlements } from './schema';
import { eq, sql } from 'drizzle-orm';

const breaks = await db
  .select({
    ref:      sql<string>`COALESCE(${ledger.txnRef}, ${settlements.txnRef})`,
    ledgerAmt:   ledger.amountCents,     // typed as nullable in the result
    settledAmt:  settlements.amountCents,
  })
  .from(ledger)
  .fullJoin(settlements, eq(settlements.txnRef, ledger.txnRef))
  .where(sql`${ledger.txnRef} IS NULL
          OR ${settlements.txnRef} IS NULL
          OR ${ledger.amountCents} IS DISTINCT FROM ${settlements.amountCents}`);
```

**Where it breaks:** Drizzle correctly infers that *both* sides' columns become **nullable** in the result type after a full join — good. The `IS DISTINCT FROM` / `COALESCE` diff logic still needs the `sql` tag (no typed operator). **Verdict:** the one ORM you can use for FULL OUTER JOIN with real type safety; still reach for `sql` on the diff predicate.

---

### Sequelize

**Can Sequelize do FULL OUTER JOIN?** — **No** via `include` (it only does INNER via `required:true` or LEFT by default). No `fullOuter` option exists.

```javascript
const { QueryTypes } = require('sequelize');

const breaks = await sequelize.query(
  `SELECT COALESCE(l.txn_ref, s.txn_ref) AS txn_ref,
          l.amount_cents AS ledger_amount, s.amount_cents AS settled_amount
   FROM ledger l
   FULL OUTER JOIN settlements s ON s.txn_ref = l.txn_ref
   WHERE l.txn_ref IS NULL OR s.txn_ref IS NULL
      OR l.amount_cents IS DISTINCT FROM s.amount_cents`,
  { type: QueryTypes.SELECT, replacements: {} }
);
```

**Where it breaks:** no builder support at all. **Verdict:** `sequelize.query()` raw.

---

### TypeORM

**Can TypeORM do FULL OUTER JOIN?** — **Not directly.** The QueryBuilder exposes `leftJoin`, `innerJoin`, but **no `fullJoin`**. Some versions let you smuggle it via a raw join expression, but it's fragile.

```typescript
// No fullJoin() exists — use raw SQL through the DataSource:
const breaks = await dataSource.query(`
  SELECT COALESCE(l.txn_ref, s.txn_ref) AS txn_ref,
         l.amount_cents AS ledger_amount, s.amount_cents AS settled_amount
  FROM ledger l
  FULL OUTER JOIN settlements s ON s.txn_ref = l.txn_ref
  WHERE l.txn_ref IS NULL OR s.txn_ref IS NULL
     OR l.amount_cents IS DISTINCT FROM s.amount_cents
`);
```

**Where it breaks:** the QueryBuilder has no first-class node; entity hydration wouldn't make sense when either side can be entirely NULL anyway. **Verdict:** `dataSource.query()` raw.

---

### Knex.js

**Can Knex do FULL OUTER JOIN?** — **Yes**, via `.fullOuterJoin()`.

```javascript
const breaks = await knex('ledger as l')
  .fullOuterJoin('settlements as s', 's.txn_ref', 'l.txn_ref')
  .select(
    knex.raw('COALESCE(l.txn_ref, s.txn_ref) AS txn_ref'),
    'l.amount_cents as ledger_amount',
    's.amount_cents as settled_amount'
  )
  .whereNull('l.txn_ref')
  .orWhereNull('s.txn_ref')
  .orWhereRaw('l.amount_cents IS DISTINCT FROM s.amount_cents');
```

**Where it breaks:** `.fullOuterJoin()` works, but the NULL-tolerant `WHERE` mixing `whereNull`/`orWhereNull`/`orWhereRaw` is error-prone — a stray `.where()` (which ANDs) can silently demote the join (Section 9, Wrong 1). Wrap the OR-group in a callback to be safe. **Verdict:** usable; guard the WHERE grouping carefully.

---

### ORM Summary Table

| ORM | FULL OUTER method | Native support | Result nullability typed? | Verdict |
|-----|-------------------|----------------|---------------------------|---------|
| Prisma | none | No | n/a | `$queryRaw` only |
| Drizzle | `.fullJoin()` | **Yes** | Yes (both sides nullable) | Best; `sql` for diff predicate |
| Sequelize | none | No | n/a | `sequelize.query()` raw |
| TypeORM | none | No | n/a | `dataSource.query()` raw |
| Knex | `.fullOuterJoin()` | Yes | Untyped | Works; guard WHERE grouping |

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `employees(id, name, salary, department_id)`
- `departments(id, name, location)`

Write a query that returns **every** employee and **every** department, including employees whose `department_id` matches no department, and departments with no employees. For each row return `employee_name`, `department_name`, and a `row_kind` column that is `'matched'`, `'employee_without_dept'`, or `'empty_department'`. Order so that empty departments and orphan employees are easy to spot.

```sql
-- Write your query here
```

Then: write a second query that returns **only** the two problem categories (orphan employees and empty departments), and a count of each.

---

### Exercise 2 — Medium (combines Topics 07, 11, 12)

Given:
- `products(id, sku, name)`
- `expected_stock(sku, qty)` — what the ERP thinks we have
- `counted_stock(sku, qty)` — last night's physical count

Write a query that reconciles `expected_stock` against `counted_stock` on `sku` and returns, for every SKU that appears in **either** table:
- `sku` (reconstructed so it's never NULL),
- `expected_qty`, `counted_qty`,
- `discrepancy` = counted − expected (treat a missing side as 0),
- `issue` = one of `'not_counted'`, `'not_expected'`, `'over'`, `'short'`, `'ok'`.

Only output SKUs where there is an actual problem (exclude `'ok'`). Use `IS DISTINCT FROM` where appropriate. Bonus: join to `products` to also show `name` without dropping SKUs that exist in neither product master (hint: which join, in which direction?).

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

You are asked: *"Give me, per customer, the total we charged (from `payments`) versus the total the processor settled (from `processor_settlements`), for yesterday — including customers who appear in only one of the two sources."*

A colleague writes:

```sql
SELECT p.customer_id,
       SUM(p.amount_cents)  AS charged,
       SUM(s.amount_cents)  AS settled
FROM payments p
FULL OUTER JOIN processor_settlements s ON s.external_ref = p.external_ref
WHERE p.created_at::date = CURRENT_DATE - 1
GROUP BY p.customer_id;
```

1. Identify **all** the bugs (there are at least three: a demotion bug, a grouping-key bug, and a fan-out/customer-id bug).
2. Write the correct query. Note that `customer_id` lives only on `payments`; settlements carry `external_ref` but not `customer_id`, so a settlement-only row has no customer — decide how to represent it.
3. State what index would make it fast on 10M-row daily volumes.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

On a whiteboard, given two tables `a(id, v)` and `b(id, v)`:

1. Write a FULL OUTER JOIN that returns every `id` present in either table, with `a.v` and `b.v` side by side.
2. Modify it to return only the `id`s where the two `v` values **disagree**, counting a value present on only one side as a disagreement.
3. Your interviewer says: "We're on MySQL, which has no FULL OUTER JOIN. Rewrite it." Produce the `UNION`-based emulation and explain why the second arm needs its anti-join filter.
4. Follow-up: "Why can't PostgreSQL use a Nested Loop for this?"

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What does a FULL OUTER JOIN return that an INNER JOIN doesn't?**

A: An INNER JOIN returns only rows that match on both sides. A FULL OUTER JOIN returns those matched rows *plus* every unmatched row from the left table (with the right columns NULL) *plus* every unmatched row from the right table (with the left columns NULL). Nothing from either table is dropped. It's the union of what a LEFT JOIN and a RIGHT JOIN would each return.

---

**Q: In a FULL OUTER JOIN result, which columns can be NULL?**

A: Any of them. For a matched row, both sides are populated. For a left-only row, all the right table's columns are NULL. For a right-only row, all the left table's columns are NULL. This is different from a LEFT JOIN, where only the right side is ever NULL-extended. That's why you reconstruct the join key with `COALESCE(a.key, b.key)` — either raw side might be NULL.

---

**Q: How do you find rows that exist in table A but not table B, and vice versa, in one query?**

A: FULL OUTER JOIN them on the key, then filter `WHERE a.key IS NULL OR b.key IS NULL`. Rows where `a.key IS NULL` exist only in B; rows where `b.key IS NULL` exist only in A. This is the reconciliation/diff pattern, and FULL OUTER JOIN is the only single join that surfaces both directions at once.

---

### Principal Level

**Q: Why can PostgreSQL never use a Nested Loop for a FULL OUTER JOIN, and what does that imply for the ON condition?**

A: A Nested Loop drives from an outer relation into an inner one; it can NULL-extend the *outer* side when the inner scan finds nothing (that's how it does LEFT/RIGHT joins), but it has no efficient way to discover *inner* rows that no outer row ever matched — that would require a second full pass tracking which inner rows were touched, which isn't what the operator does. A FULL join needs both-sided unmatched detection, so PostgreSQL only implements it with **Hash Full Join** (build a hash, mark matched entries, sweep the unmarked ones afterward) or **Merge Full Join** (walk two sorted streams, emit the lagging side's gaps). The implication: the ON condition must be **hashable** (an equality) or **mergeable**. A non-equi condition like `a.x < b.y` is neither, so the planner refuses with `ERROR: FULL JOIN is only supported with merge-joinable or hash-joinable join conditions`. There is no fallback the way Nested Loop is the universal fallback for INNER joins.

---

**Q: A reconciliation query written as `FULL OUTER JOIN ... WHERE a.status = 'x' AND b.batch = 'y'` reports zero breaks in production, but you know breaks exist. What happened?**

A: The `WHERE` clause demoted the FULL OUTER JOIN to an INNER JOIN. The join produces NULL-extended rows first; then `WHERE a.status = 'x'` evaluates to UNKNOWN for every right-only row (where `a.status` is NULL) and drops it, and `WHERE b.batch = 'y'` drops every left-only row. Only fully-matched rows where both predicates hold survive — precisely the "ok" rows, with every break filtered out. The fix is to scope each source *before* the join (in a subquery/CTE, or in the `ON` clause), and keep the outer `WHERE` NULL-tolerant, e.g. `WHERE a.key IS NULL OR b.key IS NULL OR a.val IS DISTINCT FROM b.val`. This is the FULL-OUTER analogue of the classic LEFT-JOIN "predicate in WHERE turns it into INNER" trap, except here it bites on *both* sides.

---

**Q: You need to reconcile two 100M-row tables. A single `FULL OUTER JOIN` either spills to disk or runs out of memory. How do you make it tractable?**

A: Several levers, in order. (1) **Bound the data** — reconciliation is almost always per-period; filter each side inside a CTE to one day/hour so you're never joining full history. (2) **Ensure it's a one-to-one key join** so output ≈ input, not a fan-out explosion. (3) **Index both join keys** so the planner can pick a **Merge Full Join** reading both sides pre-sorted, avoiding a giant hash entirely. (4) If Hash is chosen, **raise `work_mem` per-session** (`SET LOCAL`) to keep `Batches: 1`. (5) When even a bounded slice is huge, **partition by key range** — hash-bucket the key and reconcile bucket-by-bucket so each hash fits in memory; this is embarrassingly parallel and how large-scale recon jobs are actually built. (6) Project only `(key, value)` columns before the join to shrink the hash/sort footprint. Finally, run it as an off-peak batch job, not on the request path.

---

**Q: When would you deliberately choose a `UNION` of two queries over a FULL OUTER JOIN, or vice versa?**

A: They combine data along different axes. `UNION` stacks rows *vertically* — same columns, more rows — so you use it to append like-shaped result sets (and it's the standard way to *emulate* FULL OUTER JOIN on engines like MySQL that lack it). FULL OUTER JOIN stitches rows *horizontally* on a key, so matched rows become wider (both sides' columns side by side) and unmatched rows are NULL-padded. You want FULL OUTER JOIN when the whole point is per-key comparison — "what does source A say vs source B for the same id." You want UNION when the sources are independent lists you're concatenating. Choosing UNION when you needed per-key comparison leaves you unable to tell which source a row came from or to spot value disagreements; choosing FULL OUTER JOIN when you just wanted to concatenate wastes a hash/merge and forces awkward COALESCE gymnastics.

---

## 15. Mental Model Checkpoint

1. **Table A has 1,000 rows, Table B has 1,500 rows. 800 of A's rows match exactly one row in B (and vice versa). How many rows does `A FULL OUTER JOIN B` on the key return? Break it into the three categories.**

2. **In a FULL OUTER JOIN result, you see a row where `a.id` is NULL. What does that tell you about the row's origin? Which table did it come from?**

3. **You write `GROUP BY a.customer_id` after a FULL OUTER JOIN. What silently wrong thing happens to the right-only rows, and how do you fix it?**

4. **Why does `WHERE a.amount <> b.amount` fail to catch rows that exist only in A, and what predicate fixes it?**

5. **You move `b.status = 'active'` from the `ON` clause into the `WHERE` clause of a FULL OUTER JOIN. Which category of rows disappears, and what has the join effectively become?**

6. **Explain, in terms of the physical algorithm, why `FULL OUTER JOIN ... ON a.ts BETWEEN b.start AND b.end` throws an error while the same condition works fine in a LEFT JOIN.**

7. **You're chaining three FULL OUTER JOINs to reconcile three sources. Why must the third join's `ON` clause use `COALESCE(a.ref, b.ref)` instead of just `a.ref`?**

---

## 16. Quick Reference Card

```sql
-- Syntax (OUTER is optional)
SELECT ...
FROM a
FULL [OUTER] JOIN b ON a.key = b.key   -- must be equi (hashable) or mergeable

-- Semantics: keep ALL rows from BOTH sides; NULL-extend the missing side.
-- rows(FULL) = rows(LEFT) + rows(RIGHT) − rows(INNER)

-- Reconstruct the key (either side may be NULL):
COALESCE(a.key, b.key) AS key

-- The reconciliation / diff pattern (the reason it exists):
SELECT COALESCE(a.key, b.key) AS key, a.val, b.val,
       CASE WHEN a.key IS NULL THEN 'only_in_b'
            WHEN b.key IS NULL THEN 'only_in_a'
            WHEN a.val IS DISTINCT FROM b.val THEN 'mismatch'
            ELSE 'ok' END AS status
FROM a FULL OUTER JOIN b ON b.key = a.key
WHERE a.key IS NULL OR b.key IS NULL OR a.val IS DISTINCT FROM b.val;

-- NULL-safe value comparison (catches missing-side breaks that <> misses):
a.val IS DISTINCT FROM b.val

-- Classify a row (probe on the PK, never a nullable column):
CASE WHEN a.id IS NOT NULL AND b.id IS NOT NULL THEN 'matched'
     WHEN b.id IS NULL THEN 'left_only'
     ELSE 'right_only' END

-- GROUP BY / ORDER BY / PARTITION BY must use the coalesced key:
GROUP BY COALESCE(a.key, b.key)          -- NOT GROUP BY a.key (collapses right-only!)

-- ON vs WHERE trap (symmetric!):
--   WHERE a.col = x   → drops right-only rows (a.* NULL)  → demotes toward INNER
--   WHERE b.col = y   → drops left-only  rows (b.* NULL)  → demotes toward INNER
--   Put source filters in ON, or scope each side in a subquery, or write WHERE NULL-tolerantly.

-- Emulate on engines without FULL OUTER (e.g. MySQL):
SELECT ... FROM a LEFT JOIN b ON b.k=a.k
UNION ALL
SELECT ... FROM a RIGHT JOIN b ON b.k=a.k WHERE a.k IS NULL;  -- anti-join avoids dup matches

-- Performance rules of thumb:
--   • No Nested Loop — Hash Full Join or Merge Full Join only
--   • ON must be hashable (=) or mergeable, else: ERROR (unplannable)
--   • Index BOTH keys → enables Merge Full Join (no giant hash)
--   • Bound both sides in CTEs before joining (per-day recon)
--   • Reconcile on a UNIQUE key; many-to-many explodes
--   • Watch Hash Full Join `Batches > 1` → raise work_mem (SET LOCAL)
--   • Partition by key range for 100M-row reconciliations

-- EXPLAIN node names:
--   Hash Full Join      → equi-join, second pass emits unmatched hashed rows
--   Merge Full Join     → both inputs sorted; emits lagging side's gaps
--   ERROR: FULL JOIN is only supported with merge/hash-joinable conditions

-- Interview one-liners:
--   "FULL OUTER = LEFT ∪ RIGHT; nobody gets dropped."
--   "It's the only join with NULLs possible on BOTH sides."
--   "It's the reconciliation join — surface breaks on both sides at once."
--   "No Nested Loop; the ON condition must be equi or mergeable."
--   "A WHERE filter on either side silently demotes it to INNER."
--   "Rarely used because most relationships are directional — hence LOW priority."

-- ORM support:
--   Drizzle  .fullJoin()        ✔ (both sides typed nullable)
--   Knex     .fullOuterJoin()   ✔ (untyped; guard WHERE grouping)
--   Prisma / Sequelize / TypeORM → raw SQL only
```

---

## Connected Topics

- **Topic 07 — NULL in Depth**: Three-valued logic is why NULL keys are preserved (not dropped) as unmatched rows, why `<>` misses one-sided breaks, and why `IS DISTINCT FROM` and `IS NULL` probes are the correct tools.
- **Topic 11 — INNER JOIN in Depth**: The matched-rows core of FULL OUTER JOIN behaves identically, including one-to-many fan-out; `rows(FULL) = rows(LEFT) + rows(RIGHT) − rows(INNER)`.
- **Topic 12 — LEFT JOIN and RIGHT JOIN**: FULL OUTER JOIN is their union; the ON-vs-WHERE demotion trap you learned there applies here symmetrically, on both sides.
- **Topic 14 — CROSS JOIN**: The other end of the join spectrum — CROSS JOIN keeps every *pair* unconditionally; FULL OUTER JOIN keeps every *row* while still matching on a condition.
- **Topic 17 — JOIN Performance Deep Dive**: Why only Hash and Merge implement FULL OUTER JOIN, hash match-marking, merge gap emission, spill/`work_mem` tuning.
- **Topic 20 — GROUP BY Fundamentals**: Grouping after a FULL OUTER JOIN must use the coalesced key or right-only rows collapse into the NULL bucket.
- **Topic 26 — UNION, INTERSECT, EXCEPT**: The vertical-combination alternative; the FULL-OUTER emulation pattern and when set operators express a diff more cleanly.
- **Topic 27 — EXISTS and NOT EXISTS**: `NOT EXISTS` anti-joins are the targeted, cheaper way to get "rows only in one side" when you don't need both directions at once.
