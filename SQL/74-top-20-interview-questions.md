# Topic 74 — The 20 Most Common SQL Interview Questions
### SQL Mastery Curriculum — Phase 11: SQL for Backend Developer Interviews

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine two candidates walk into a SQL interview and both get the same question: *"What's the difference between WHERE and HAVING?"*

The **junior** answers: "WHERE filters rows and HAVING filters groups." Correct — and it earns a nod. But the interviewer already knew the candidate could recite that; it's on every flashcard site.

The **principal engineer** answers: "They filter at different points in the logical execution order. WHERE runs before GROUP BY, so it filters individual rows before aggregation and can use an index to avoid reading them at all. HAVING runs after aggregation, so it filters on aggregate results like `SUM(total) > 1000`, which by definition can't exist until the groups are formed. That's why you can't reference `SUM()` in WHERE, and why moving a non-aggregate predicate from HAVING to WHERE is a real optimization — fewer rows enter the sort-and-group step."

Same question. Same first sentence. But the second answer demonstrates *why* — it shows the candidate understands the engine, not just the syntax.

This topic is the difference between those two answers, twenty times over. Every common SQL interview question has a flashcard answer that's technically correct and a principal-level answer that shows depth. Interviewers ask the easy version to hear the hard answer. This doc gives you both — and the follow-up they'll ask next.

---

## 2. Connection to SQL Internals

Interview questions are not abstract trivia — every "difference between X and Y" question maps to a concrete engine mechanism. The candidates who impress are the ones who name the mechanism.

| Interview question | The internal it's really testing |
|--------------------|----------------------------------|
| WHERE vs HAVING | Logical execution order; predicate pushdown |
| INNER vs LEFT JOIN | Row-preservation semantics; the ON-vs-WHERE trap |
| Which index for this query | B-tree structure, column order, selectivity, covering indexes |
| N+1 problem | Round-trip latency vs set-based execution |
| Window function vs GROUP BY | Whether rows collapse; the Window/Sort/WindowAgg nodes |
| UNION vs UNION ALL | The implicit DISTINCT sort/hash-aggregate pass |
| DELETE vs TRUNCATE | MVCC dead tuples, WAL, table locks, sequence reset |
| Transaction isolation | MVCC snapshots, visibility, lost updates, phantom rows |
| NULL / NOT IN trap | Three-valued logic; UNKNOWN is not FALSE |
| EXISTS vs IN | Semi-join transformation; short-circuit on first match |

Notice the recurring cast: **B-tree indexes**, **MVCC snapshots**, **the planner's join and predicate choices**, **work_mem sorts and hash tables**, and **three-valued logic**. Almost every question in this doc reduces to one of those five. If you can connect any interview question back to one of them, you're answering at the principal level. This doc is organized so that section 6 walks all twenty questions, and each one explicitly names the internal underneath.

---

## 3. Logical Execution Order Context

More SQL interview answers hinge on logical execution order than on any other single concept. Memorize it; you will reference it constantly:

```
FROM / JOIN     ← tables assembled, rows produced
WHERE           ← row-level filter (pre-aggregation)   ← index-usable
GROUP BY        ← rows collapse into groups
HAVING          ← group-level filter (post-aggregation)
SELECT          ← expressions/aliases computed here
DISTINCT        ← duplicate removal
window funcs    ← OVER() evaluated on the post-GROUP result set
ORDER BY        ← final sort (can use SELECT aliases)
LIMIT / OFFSET  ← final row cut
```

This single diagram answers, directly or indirectly:
- **WHERE vs HAVING** — WHERE is above GROUP BY, HAVING is below it.
- **Why can't I use a SELECT alias in WHERE?** — SELECT runs after WHERE; the alias doesn't exist yet.
- **Why can I use a SELECT alias in ORDER BY?** — ORDER BY runs after SELECT.
- **Why does a window function see all rows even with GROUP BY?** — windows run after grouping, over the grouped result.
- **Why does LEFT JOIN + a WHERE on the right table become an INNER JOIN?** — the WHERE runs after the join and discards the NULL-extended rows.

When you're stuck on an interview question about *why* something behaves a certain way, return to this list and ask "what has already run, and what hasn't yet?" It resolves most of them.

---

## 4. What Is a "Backend SQL Interview Question"?

A backend SQL interview question is a probe with two layers: a surface question that has a memorizable answer, and a hidden question — *do you understand the engine well enough to reason past the flashcard?* The interviewer asks layer one and grades layer two.

The four-part anatomy this doc uses for all twenty questions:

```
Q: <the question as asked>
│
├── Junior answer
│     └── Technically correct, flashcard-level. Earns a pass, not a hire.
│
├── Principal answer
│     └── Names the internal mechanism, states the edge cases,
│         gives the production consequence. Earns the hire.
│
└── Interviewer's follow-up
      └── The next question they ask to see if you *really* know it,
          or were just reciting. This is where most candidates fall.
```

The trap in most SQL interviews is that the junior answer *sounds complete*. "UNION removes duplicates, UNION ALL keeps them" is not wrong. But the interviewer is waiting to hear the word "sort" or "hash aggregate" — the reason UNION is slower — and if it doesn't come, they ask the follow-up. This doc front-loads the follow-up answers so you're never caught flat.

---

## 5. Why Interview-Question Mastery Matters in Production

It's tempting to treat interview prep as a separate skill from real engineering. It isn't — every question in this doc maps to a production incident that has actually happened:

1. **The NOT IN / NULL trap** returns zero rows silently in a nightly reconciliation job, and finance notices a week later that no discrepancies were "found" because the subquery contained one NULL.

2. **The N+1 problem** turns a 40ms endpoint into a 4-second endpoint under load because an ORM issued 800 queries instead of one JOIN — the classic cause of a p99 latency page.

3. **DELETE instead of TRUNCATE** on a 200M-row staging table generates 200M dead tuples and enough WAL to fill the disk, while TRUNCATE would have been instant.

4. **The wrong isolation level** lets two concurrent balance updates lose one of the increments — a lost update that shows up as a customer's money vanishing.

5. **The wrong index** (or a redundant one) means a query the interviewer asked about in the abstract is doing a Seq Scan on 50M rows in prod at 2am.

Mastering these questions is not about passing the interview — it's that the interview questions *are* the production failure modes, distilled. An engineer who can answer all twenty at the principal level is precisely the engineer who doesn't ship these bugs.

---

## 6. Deep Technical Content — The 20 Questions

Each question below carries three parts: the **junior answer** (correct but shallow), the **principal answer** (the internal + the edge case + the production cost), and the **follow-up** the interviewer asks next. This is the core of the doc.

### Q1. What's the difference between WHERE and HAVING?

**Junior:** WHERE filters rows; HAVING filters groups.

**Principal:** They act at different points in logical execution order (Topic 20, 21). WHERE runs *before* GROUP BY — it filters individual rows, cannot reference aggregate functions, and critically can use an index to avoid reading rows at all. HAVING runs *after* GROUP BY — it filters on aggregate results (`HAVING SUM(amount) > 1000`), which cannot exist before the groups are formed. Consequence: any non-aggregate predicate belongs in WHERE, not HAVING, because filtering before the sort/hash-aggregate step means fewer rows enter it.

```sql
-- Filters rows BEFORE grouping (index-usable), then groups (correct + fast)
SELECT customer_id, SUM(total_amount) AS spend
FROM orders
WHERE status = 'completed'          -- WHERE: row filter, pre-aggregation
GROUP BY customer_id
HAVING SUM(total_amount) > 1000;    -- HAVING: aggregate filter, post-aggregation
```

**Follow-up:** *"Can you put `status = 'completed'` in HAVING instead? Would it give the same result?"* — Yes, the result is identical (status is functionally dependent within the group here only if grouped by it; in general it would error unless status is in GROUP BY). The real answer: if `status` is not in GROUP BY, `HAVING status = 'completed'` errors. If it were, it works but is slower — you've grouped rows you'll then throw away. Always filter earliest.

### Q2. INNER JOIN vs LEFT JOIN — and the trap.

**Junior:** INNER returns only matching rows; LEFT returns all left rows plus matches, NULLs where no match.

**Principal:** Correct, but the interview gold is the **WHERE-after-LEFT-JOIN trap** (Topic 12). A predicate on the right table in the WHERE clause silently converts a LEFT JOIN back into an INNER JOIN, because WHERE runs after the join and the NULL-extended rows fail `right.col = 'x'` (NULL comparison → UNKNOWN → dropped). The fix is to move the predicate into the ON clause, where it's evaluated *during* the join and doesn't remove left rows.

```sql
-- BUG: this is secretly an INNER JOIN — customers with no 2025 orders vanish
SELECT c.id, o.id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.created_at >= '2025-01-01';   -- NULL >= '...' is UNKNOWN → row dropped

-- RIGHT: predicate in ON preserves unmatched customers
SELECT c.id, o.id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.created_at >= '2025-01-01';
```

**Follow-up:** *"How do you find customers with NO orders at all?"* — LEFT JOIN then `WHERE o.id IS NULL` (the anti-join), or `NOT EXISTS`. The IS NULL check is legitimate here because you're deliberately keeping the unmatched rows.

### Q3. How do you choose which column to index for a query?

**Junior:** Index the column in the WHERE clause.

**Principal:** Index for **selectivity and access pattern**, not just presence in WHERE. A B-tree index helps when it narrows to a small fraction of rows; on a low-cardinality column like `status` with two values, the planner will Seq Scan anyway because reading 50% of the table via random index lookups is slower than a sequential read. For composite indexes, column order follows the **equality-then-range** rule: columns used with `=` first, then one range column, because a B-tree can only range-scan on the last used column. A **covering index** (via `INCLUDE`) lets the query be answered from the index alone (Index Only Scan), avoiding heap fetches entirely.

```sql
-- Query: WHERE customer_id = $1 AND created_at >= $2 ORDER BY created_at
-- Right index: equality column first, range/sort column second
CREATE INDEX idx_orders_cust_created ON orders (customer_id, created_at);
-- Now the index satisfies the filter AND provides the sort order (no Sort node)
```

**Follow-up:** *"You have indexes on (a) and (a, b). Is the single-column one redundant?"* — Usually yes; `(a, b)` can serve any query that `(a)` serves (leftmost-prefix rule), so drop `(a)` unless it's used for Index Only Scans where the narrower index is cheaper to keep hot in cache.

### Q4. What is the N+1 query problem and how do you fix it?

**Junior:** It's when you run one query to get a list, then one query per item — N+1 total. Fix it with a JOIN.

**Principal:** The cost isn't the queries themselves — it's **round-trip latency**. Each query is a separate network round-trip and planner invocation; at 1ms RTT, 800 items become 800ms of pure waiting, serialized (Topic 18). The engine could do the work in a single Hash Join in a few ms. Fixes, in order of preference: (1) a JOIN to fetch everything set-based, (2) eager-loading via the ORM (`include`/`with`), (3) a single `WHERE id = ANY($1)` batched query when a JOIN isn't natural. ORMs cause this by default with lazy relations — accessing `order.customer` inside a loop fires a query each iteration.

```javascript
// N+1: 1 query for orders + N queries for each customer
const orders = await Order.findAll();
for (const o of orders) { o.customer = await Customer.findByPk(o.customerId); } // BAD

// Fixed: one query, set-based
const orders = await Order.findAll({ include: [{ model: Customer }] });
```

**Follow-up:** *"You batched with `id = ANY(ARRAY[...])` of 10,000 ids — now it's slow again. Why?"* — A giant array can blow past what the planner will index-probe efficiently, and parameter/parse overhead grows; chunk into batches of a few hundred to low thousands, or join against a temp table / `unnest`.

### Q5. Window functions vs GROUP BY — when do you use each?

**Junior:** GROUP BY collapses rows into one per group; window functions compute across rows but keep every row.

**Principal:** The key distinction is **row preservation** (Topic 30). GROUP BY reduces N rows to one per group — you lose the detail. A window function (`OVER (PARTITION BY ...)`) computes the aggregate but *attaches it to every original row*, so you can show a row alongside its group's total, rank, or running sum without a self-join. Use GROUP BY when you want the summary; use a window when you want per-row context *and* the aggregate. Windows also do things GROUP BY can't: `ROW_NUMBER`, `LAG/LEAD`, running totals, top-N-per-group.

```sql
-- Each order row PLUS that customer's total spend and rank — no collapse, no self-join
SELECT id, customer_id, total_amount,
       SUM(total_amount)  OVER (PARTITION BY customer_id) AS cust_total,
       ROW_NUMBER()       OVER (PARTITION BY customer_id ORDER BY total_amount DESC) AS rnk
FROM orders;
```

**Follow-up:** *"How do you get the single most expensive order per customer?"* — `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total_amount DESC)` in a subquery, then `WHERE rnk = 1`. Discuss `RANK` vs `ROW_NUMBER` for ties: RANK keeps all tied rows, ROW_NUMBER picks exactly one arbitrarily.

### Q6. UNION vs UNION ALL.

**Junior:** UNION removes duplicates, UNION ALL keeps them.

**Principal:** The mechanism behind "removes duplicates" is the cost: **UNION performs an implicit DISTINCT**, which requires a sort or hash-aggregate pass over the combined result — O(N log N) or a hash build across all rows. UNION ALL just concatenates the two streams with zero extra work. So UNION ALL is strictly cheaper and should be the default *unless you actually need dedup*. Many developers reflexively write UNION and pay for a dedup they don't need. If you know the two sets are disjoint (e.g., different status values), UNION ALL is correct and free.

```sql
-- UNION ALL: no dedup pass, just concatenation — use when sets can't overlap
SELECT id FROM active_users
UNION ALL
SELECT id FROM archived_users;   -- disjoint by construction → dedup would be wasted work
```

**Follow-up:** *"Both branches select from the same table with different WHERE clauses — could you avoid UNION entirely?"* — Yes, combine into one scan with `WHERE cond_a OR cond_b`, or a `CASE`/`FILTER` aggregate. One pass beats two scans plus a merge.

### Q7. DELETE vs TRUNCATE vs DROP.

**Junior:** DELETE removes rows (can filter with WHERE), TRUNCATE removes all rows fast, DROP removes the table.

**Principal:** The engine-level differences drive real decisions (Topic 40s). DELETE is a **row-by-row, MVCC-logged** operation: each deleted row becomes a dead tuple that VACUUM must later reclaim, every deletion is WAL-logged, triggers fire, and it's fully transactional/rollback-able. TRUNCATE deallocates the table's data files wholesale — it's near-instant regardless of row count, generates minimal WAL, resets sequences with `RESTART IDENTITY`, but takes an `ACCESS EXCLUSIVE` lock and doesn't fire row triggers. On a 200M-row table, DELETE can run for minutes and bloat the table with dead tuples plus massive WAL; TRUNCATE is milliseconds. TRUNCATE is transactional in PostgreSQL (unlike MySQL) — you *can* roll it back.

```sql
DELETE FROM staging_events WHERE created_at < '2025-01-01'; -- filtered, MVCC, slow at scale
TRUNCATE staging_events RESTART IDENTITY;                    -- all rows, instant, resets seq
```

**Follow-up:** *"You TRUNCATE a table that's referenced by a foreign key. What happens?"* — It fails unless you `TRUNCATE ... CASCADE` (which truncates the referencing tables too) or there are no rows referencing it. This is a safety feature; CASCADE is dangerous — it can wipe far more than intended.

### Q8. Explain transaction isolation levels.

**Junior:** READ COMMITTED, REPEATABLE READ, SERIALIZABLE — higher levels prevent more anomalies.

**Principal:** Each level is defined by which **read anomalies** it permits, and PostgreSQL implements them via **MVCC snapshots** (Topic on MVCC). READ COMMITTED (the default) takes a fresh snapshot per statement — you never see uncommitted data, but two statements in the same transaction can see different committed data (non-repeatable read). REPEATABLE READ takes one snapshot at the first statement and holds it for the whole transaction — no non-repeatable reads or phantoms, but you can hit serialization failures on write conflicts. SERIALIZABLE adds predicate-level conflict detection (SSI) so the outcome is as if transactions ran one at a time, at the cost of more serialization-failure retries.

| Level | Dirty read | Non-repeatable read | Phantom | Lost update risk |
|-------|-----------|---------------------|---------|------------------|
| READ COMMITTED | No | Yes | Yes | Yes (read-modify-write) |
| REPEATABLE READ | No | No | No (in PG) | Detected → error |
| SERIALIZABLE | No | No | No | Prevented |

**Follow-up:** *"At READ COMMITTED, `balance = balance - 100` under concurrency — is it safe?"* — A single UPDATE statement is atomic and re-reads the row under a row lock, so `UPDATE accounts SET balance = balance - 100` is safe. But a *read-then-write* in application code (`SELECT balance` then `UPDATE ... SET balance = $newValue`) is a lost-update bug — use `SELECT ... FOR UPDATE` or do the arithmetic in SQL.

### Q9. The NULL / NOT IN trap.

**Junior:** `NOT IN` checks that a value isn't in a list.

**Principal:** `NOT IN (subquery)` is a **landmine when the subquery can return NULL** (Topic 07). `x NOT IN (1, 2, NULL)` expands to `x <> 1 AND x <> 2 AND x <> NULL`, and `x <> NULL` is UNKNOWN, so the whole expression can never be TRUE — the query returns **zero rows** with no error. This silently breaks reconciliation queries the moment one NULL sneaks into the subquery. Use `NOT EXISTS` instead, which uses proper existence semantics and is NULL-safe, or add `WHERE col IS NOT NULL` to the subquery.

```sql
-- BUG: returns ZERO rows if any customer_id in orders is NULL
SELECT * FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);

-- RIGHT: NULL-safe anti-join
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

**Follow-up:** *"Why doesn't `IN` (the positive form) have the same problem?"* — `x IN (1, NULL)` is `x = 1 OR x = NULL`; if `x = 1` it's TRUE regardless of the NULL, so positive IN still finds matches. It's only the negation, where every term must be TRUE, that the UNKNOWN poisons.

### Q10. EXISTS vs IN — which is faster?

**Junior:** They're interchangeable; use whichever reads better.

**Principal:** Modern PostgreSQL often plans both as a **semi-join**, so performance can be equal — but they differ semantically and at the edges (Topic 27). `EXISTS` **short-circuits** on the first matching row and is NULL-safe. `IN` with a subquery materializes/scans the value set and has the NULL trap in its negation. Rule of thumb: use `EXISTS` for correlated existence checks (especially anti-joins), and `IN` for small constant lists. `IN` can be faster for a tiny literal list; `EXISTS` scales better for correlated subqueries against large tables because it stops at the first hit per outer row.

```sql
-- EXISTS: stops at first matching order per customer, NULL-safe
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'completed');
```

**Follow-up:** *"Does `SELECT 1` vs `SELECT *` inside EXISTS matter for performance?"* — No. EXISTS only checks row existence, not the select list; the planner ignores it entirely. `SELECT 1` is convention, not optimization.

### Q11. What does an index actually do, and when does it hurt?

**Junior:** An index makes queries faster by letting the database find rows without scanning the whole table.

**Principal:** A B-tree index is a sorted, balanced tree that turns O(N) scans into O(log N) lookups for selective predicates and provides ordering for free. But indexes are **not free** — every INSERT/UPDATE/DELETE must maintain every index on the table, so write-heavy tables pay a per-index write tax. Indexes consume disk and cache. And a **non-selective index is worse than none**: the planner ignores it, but you still pay to maintain it. Also, an index on a column you then wrap in a function (`WHERE lower(email) = ...`) won't be used unless it's an expression index on `lower(email)`.

**Follow-up:** *"Your UPDATE-heavy table has 9 indexes and writes are slow. What do you do?"* — Audit `pg_stat_user_indexes` for `idx_scan = 0` (never-used indexes) and drop them; consolidate overlapping composite indexes via the leftmost-prefix rule. Fewer, well-chosen indexes speed writes without hurting the reads that matter.

### Q12. GROUP BY — which columns must appear in the SELECT?

**Junior:** Every non-aggregated column in SELECT must be in GROUP BY.

**Principal:** Correct, and the reason is that after GROUP BY, each output row *is* a group — a column that isn't grouped or aggregated has no single well-defined value for the group, so it's an error in PostgreSQL (MySQL historically let it through with `ONLY_FULL_GROUP_BY` off, returning an arbitrary value — a portability trap). The exception is **functional dependency**: if you `GROUP BY` a primary key, PostgreSQL lets you SELECT other columns of that table ungrouped, because the PK determines them uniquely.

```sql
-- Valid: grouping by PK lets you select functionally-dependent columns
SELECT u.id, u.name, COUNT(o.id) AS orders
FROM users u JOIN orders o ON o.user_id = u.id
GROUP BY u.id;   -- u.name allowed because u.id is the PK
```

**Follow-up:** *"How do you get one aggregate plus a non-aggregated column that ISN'T functionally dependent — e.g., the product name of the top-selling item per category?"* — That's a top-N-per-group; use `DISTINCT ON` or a `ROW_NUMBER` window, not plain GROUP BY.

### Q13. What's the difference between `COUNT(*)`, `COUNT(col)`, and `COUNT(DISTINCT col)`?

**Junior:** `COUNT(*)` counts rows, `COUNT(col)` counts non-null values, `COUNT(DISTINCT col)` counts unique values.

**Principal:** Precisely — and the distinctions bite in production. `COUNT(*)` counts all rows including those with NULLs. `COUNT(col)` **skips NULLs**, so `COUNT(*)` vs `COUNT(col)` reveals how many NULLs exist. `COUNT(DISTINCT col)` requires a sort or hash to dedup — it's markedly more expensive at scale. The classic bug: after a one-to-many JOIN, `COUNT(o.id)` counts the multiplied rows (once per item), so you need `COUNT(DISTINCT o.id)` to count orders (Topic 11).

**Follow-up:** *"`COUNT(*)` on a 500M-row table is slow. How do you make it fast?"* — Exact count requires a full scan (or index-only scan). For approximate, read `reltuples` from `pg_class` (updated by ANALYZE). For fast exact counts of a filtered subset, a covering index enables an Index Only Scan; for a maintained live count, a trigger-updated summary row or a materialized aggregate.

### Q14. How does LIMIT/OFFSET pagination scale, and what's better?

**Junior:** `LIMIT 20 OFFSET 10000` skips 10,000 rows and returns the next 20.

**Principal:** The problem is that **OFFSET still reads and discards** all skipped rows — `OFFSET 10000` scans 10,020 rows to return 20, so deep pages get linearly slower and page 5,000 is catastrophic. **Keyset (cursor) pagination** fixes it: instead of an offset, filter on the last-seen sorted key (`WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 20`), which uses the index to jump directly to the position in O(log N) — every page is equally fast. The trade-off: you can't jump to an arbitrary page number, only next/previous.

```sql
-- Keyset: O(log N) per page, index-backed, no rows discarded
SELECT * FROM orders
WHERE (created_at, id) < ($1, $2)     -- last row of previous page
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

**Follow-up:** *"Your keyset sorts by a non-unique column like created_at. What breaks?"* — Ties at the boundary can skip or repeat rows. Fix by making the sort key unique — append a tiebreaker (`id`) and compare as a row/tuple `(created_at, id)`.

### Q15. What is a covering index / Index Only Scan?

**Junior:** An index that contains all the columns a query needs.

**Principal:** A covering index lets PostgreSQL answer a query **entirely from the index**, skipping the heap fetch (the random I/O to read the actual table row) — that's an **Index Only Scan** in EXPLAIN, and it's dramatically faster because it avoids touching table pages. You build one by putting filter/sort columns in the index key and payload columns in `INCLUDE (...)`. Caveat: Index Only Scans still consult the **visibility map**; if the table has many recently-updated rows not yet vacuumed, PostgreSQL must visit the heap anyway to check tuple visibility, degrading the benefit — so keep the table well-vacuumed.

```sql
CREATE INDEX idx_orders_cover ON orders (customer_id) INCLUDE (total_amount, status);
-- SELECT total_amount, status FROM orders WHERE customer_id = $1 → Index Only Scan
```

**Follow-up:** *"EXPLAIN says 'Index Only Scan' but shows a high 'Heap Fetches' number. What's wrong?"* — The visibility map is stale; those fetches are visibility checks against the heap. Run `VACUUM` (or let autovacuum catch up) to update the visibility map and eliminate them.

### Q16. Correlated vs non-correlated subquery.

**Junior:** A correlated subquery references the outer query; a non-correlated one doesn't.

**Principal:** The distinction is about **execution frequency and planning**. A non-correlated subquery is independent — it can be evaluated once and its result reused (the planner may materialize it or fold it into a join). A correlated subquery references an outer column, so *logically* it runs once per outer row — which sounds O(N×M) and slow, but modern planners frequently **decorrelate** it into a semi-join or hash join, collapsing it to O(N+M). The interview point: don't assume correlated = slow; check EXPLAIN. It's slow only when the planner *can't* decorrelate (e.g., certain aggregate correlations).

**Follow-up:** *"Rewrite a correlated `SELECT (SELECT MAX...)` per-row into something the planner handles better."* — Convert to a JOIN against a grouped subquery or a window function; both let the aggregate be computed once over partitions instead of re-run per outer row.

### Q17. What is a deadlock and how do you prevent it?

**Junior:** Two transactions each hold a lock the other wants, so neither proceeds.

**Principal:** Exactly — a cycle in the lock wait-for graph. PostgreSQL **detects** deadlocks automatically (after `deadlock_timeout`) and kills one transaction with a `deadlock detected` error; your app must catch and retry. Prevention is about **consistent lock ordering**: if every transaction acquires rows/tables in the same order (e.g., always lower account id before higher), no cycle can form. Other levers: keep transactions short, touch rows in a deterministic order, and use `SELECT ... FOR UPDATE` with `ORDER BY` to lock in a fixed sequence.

```sql
-- Lock both accounts in a deterministic order to prevent deadlock cycles
SELECT * FROM accounts WHERE id IN ($1, $2) ORDER BY id FOR UPDATE;
```

**Follow-up:** *"You can't guarantee lock order across services. Now what?"* — Catch the deadlock error (SQLSTATE 40P01) and retry with backoff; make the transaction idempotent so retry is safe. Also consider `SELECT ... FOR UPDATE NOWAIT` / `SKIP LOCKED` for queue-style workloads to fail fast instead of blocking.

### Q18. Primary key vs unique key vs index.

**Junior:** A primary key uniquely identifies a row and can't be NULL; a unique key enforces uniqueness but allows one NULL; an index speeds lookups.

**Principal:** A PRIMARY KEY is a UNIQUE constraint plus NOT NULL plus the table's canonical identity (one per table, the default target of foreign keys). A UNIQUE constraint enforces uniqueness and **allows multiple NULLs** in standard SQL/PostgreSQL, because NULL ≠ NULL — a frequent surprise (`UNIQUE(email)` still allows many rows with NULL email). Both PK and UNIQUE are **implemented via a unique B-tree index** under the hood, so they provide the lookup speed of an index for free. A plain index provides neither uniqueness nor NOT-NULL — it's purely a performance structure.

**Follow-up:** *"You need at most one NULL email but many non-null uniques — no wait, you need email unique but treat NULLs as distinct AND forbid duplicate NULLs. How?"* — Standard UNIQUE allows many NULLs; to forbid duplicate NULLs use `UNIQUE NULLS NOT DISTINCT` (PG 15+), or a partial unique index / a coalesced expression index depending on the exact rule.

### Q19. What's the difference between `CHAR`, `VARCHAR`, and `TEXT` (and why it rarely matters in PostgreSQL)?

**Junior:** `CHAR(n)` is fixed length, `VARCHAR(n)` is variable up to n, `TEXT` is unlimited.

**Principal:** In PostgreSQL specifically, `VARCHAR(n)` and `TEXT` are **the same underlying type** with identical storage and performance; `VARCHAR(n)` just adds a length-check constraint. `CHAR(n)` is actually *slower* — it blank-pads to n characters and those pads waste space and cause surprising trailing-space comparison behavior. So the idiomatic PostgreSQL choice is `TEXT` (or `VARCHAR` without a length when you don't have a real business limit), and enforce length with a `CHECK` if needed. This differs from some other databases where CHAR/VARCHAR have real performance distinctions — a good "do you know PostgreSQL specifically" probe.

**Follow-up:** *"Then why would anyone use `VARCHAR(255)`?"* — Cargo-culted from MySQL/older systems where 255 was a storage boundary. In PostgreSQL it offers no benefit over TEXT except documenting/enforcing a max length — and changing that length later requires a table rewrite in some cases, so `TEXT` + `CHECK` is more flexible.

### Q20. How do you find and fix a slow query?

**Junior:** Run EXPLAIN and add an index.

**Principal:** A disciplined workflow, not a reflex (Topic on EXPLAIN). (1) Reproduce with `EXPLAIN (ANALYZE, BUFFERS)` to get *actual* rows, timing, and I/O — not just estimates. (2) Look for the **red flags**: a Seq Scan on a large table with a selective filter, a big gap between estimated and actual rows (stale statistics → run `ANALYZE`), a Nested Loop with a huge `loops` count, a Sort or Hash spilling to disk (`work_mem` too small), or Hash Batches > 1. (3) Only then decide the fix — often an index, sometimes a rewrite (avoid a function on an indexed column, pre-aggregate to kill a fan-out, keyset instead of OFFSET), sometimes more `work_mem`. The point: *measure the plan first, then target the actual bottleneck* — adding a random index is guessing.

**Follow-up:** *"EXPLAIN estimates 10 rows but ANALYZE shows 2 million actual. What's the single most likely cause and fix?"* — Stale or insufficient statistics; run `ANALYZE` (or raise the column's statistics target for skewed data). The planner chose a bad plan because it believed the wrong row count.

---

## 7. EXPLAIN — Reading a Plan Under Interview Pressure

Interviewers love handing you an EXPLAIN and asking "what's wrong?" Here's the pattern that recurs — a Seq Scan where an index should be used, plus the fixed version.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 42 AND status = 'completed';
```

**Before an index — the plan you're asked to diagnose:**

```
Seq Scan on orders  (cost=0.00..21730.00 rows=6 width=64)
                    (actual time=0.03..118.4 rows=5 loops=1)
  Filter: ((customer_id = 42) AND (status = 'completed'))
  Rows Removed by Filter: 999995
Buffers: shared hit=8730
Planning Time: 0.2 ms
Execution Time: 118.6 ms
```

**Reading it:** `Seq Scan` reads all 1M rows; `Rows Removed by Filter: 999995` is the smoking gun — the engine touched a million rows to return five. The `Filter` (not `Index Cond`) confirms no index is being used. 118ms for five rows is the interview's cue.

**After adding `CREATE INDEX ON orders (customer_id, status);`:**

```
Index Scan using orders_customer_id_status_idx on orders
        (cost=0.42..25.10 rows=6 width=64)
        (actual time=0.03..0.05 rows=5 loops=1)
  Index Cond: ((customer_id = 42) AND (status = 'completed'))
Buffers: shared hit=8
Planning Time: 0.3 ms
Execution Time: 0.07 ms
```

**Reading it:** Now `Index Scan` with `Index Cond` (equality on both columns, resolved by the composite index), `Buffers` dropped from 8730 to 8, and execution went 118ms → 0.07ms — a ~1700× improvement. This before/after is the single most common EXPLAIN interview exchange.

**The EXPLAIN red-flag cheat sheet interviewers probe:**

| Symptom in the plan | What it means | Fix |
|---------------------|---------------|-----|
| `Seq Scan` + high `Rows Removed by Filter` | No usable index for a selective filter | Add index on the filter column(s) |
| Estimated rows ≫ or ≪ actual rows | Stale statistics | `ANALYZE` the table |
| `Nested Loop` with large `loops=` and Seq Scan inner | Missing index on inner join column | Index the inner join key |
| `Sort` / `Hash` shows `Disk:` usage | Spilling — `work_mem` too small | Raise `work_mem` or reduce rows first |
| `Hash Join` with `Batches: > 1` | Hash spilled to disk | Raise `work_mem` |
| `Index Only Scan` + high `Heap Fetches` | Stale visibility map | `VACUUM` the table |

### The N+1 anti-pattern in EXPLAIN terms

You rarely see N+1 in a single EXPLAIN — that's the point, it's *N separate* plans. But the interviewer may ask you to contrast the two shapes. The N+1 version issues 800 of these, one per order:

```
Index Scan using customers_pkey on customers  (cost=0.42..8.44 rows=1 loops=1)
  Index Cond: (id = $1)
-- ~0.1ms each, but × 800 round-trips + 800 planner invocations = ~800ms wall clock
```

The set-based replacement issues one plan for all of them:

```
Hash Join  (cost=27.50..410.00 rows=800 width=72) (actual time=0.4..2.1 rows=800 loops=1)
  Hash Cond: (o.customer_id = c.id)
  ->  Index Scan using orders_pkey on orders o  (Index Cond: (id = ANY($1)))
  ->  Hash  ->  Seq Scan on customers c
-- One round-trip, one plan, ~2ms. The 400× win is round-trips eliminated, not CPU.
```

**Reading it:** the individual per-order lookup is *fast* (0.1ms) — that's what fools people. The cost is the 800× network round-trip and parse/plan overhead, which a single Hash Join collapses into one. This is the plan-level proof that N+1 is a latency problem, not a per-query cost problem.

### The anti-join (NOT EXISTS) plan

The interview claim "NOT EXISTS plans as an anti-join" is verifiable:

```sql
EXPLAIN ANALYZE
SELECT c.id FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

```
Hash Anti Join  (cost=520.00..1830.00 rows=250 width=8)
                (actual time=6.2..24.1 rows=243 loops=1)
  Hash Cond: (c.id = o.customer_id)
  ->  Seq Scan on customers c   (actual rows=1000 loops=1)
  ->  Hash  ->  Seq Scan on orders o  (actual rows=30000 loops=1)
```

**Reading it:** the node type is literally `Hash Anti Join` — the planner recognized the NOT EXISTS as an anti-join and executes it as one hash pass, O(N+M), NULL-safe. Contrast this with the `NOT IN` form on a NULLable column, which cannot become an anti-join (the NULL semantics forbid it) and degrades to a slower, and often *wrong*, plan.

---

## 8. Query Examples — The Q&A Made Concrete

### Example 1 — Basic: The WHERE-vs-HAVING answer, demonstrated

```sql
-- The interview question in code form: filter rows, then filter groups.
SELECT
  customer_id,
  COUNT(*)            AS order_count,
  SUM(total_amount)   AS lifetime_spend
FROM orders
WHERE status = 'completed'          -- row filter, BEFORE grouping (index-usable)
GROUP BY customer_id
HAVING SUM(total_amount) > 1000     -- aggregate filter, AFTER grouping
ORDER BY lifetime_spend DESC;
```

### Example 2 — Intermediate: The LEFT JOIN trap, both wrong and right

```sql
-- WRONG — silently an INNER JOIN; customers with no recent orders disappear
SELECT c.id, c.name, o.id AS order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.created_at >= CURRENT_DATE - INTERVAL '30 days';

-- RIGHT — predicate in ON; every customer kept, order_id NULL if none recent
SELECT c.id, c.name, o.id AS order_id
FROM customers c
LEFT JOIN orders o
  ON o.customer_id = c.id
  AND o.created_at >= CURRENT_DATE - INTERVAL '30 days';
```

### Example 3 — Production Grade: Top-N-per-group without the GROUP BY trap

```sql
-- "Most expensive completed order per customer, with its rank and the customer's total."
-- Table sizes: orders ~20M rows, customers ~1M.
-- Index available: idx_orders_cust_created ON orders(customer_id, created_at)
--                  plus a supporting index on (customer_id, total_amount DESC)
-- Perf expectation: window over partitions backed by index → single scan, no self-join.
WITH ranked AS (
  SELECT
    o.id,
    o.customer_id,
    o.total_amount,
    o.created_at,
    ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.total_amount DESC) AS rn,
    SUM(o.total_amount) OVER (PARTITION BY o.customer_id) AS cust_lifetime
  FROM orders o
  WHERE o.status = 'completed'
)
SELECT
  c.id          AS customer_id,
  c.name,
  r.id          AS top_order_id,
  r.total_amount AS top_order_value,
  r.cust_lifetime
FROM ranked r
INNER JOIN customers c ON c.id = r.customer_id
WHERE r.rn = 1                       -- keep only the top order per customer
ORDER BY r.cust_lifetime DESC
LIMIT 100;
```

```
Limit  (cost=... rows=100)
  ->  Sort (cust_lifetime DESC)
        ->  Hash Join  (customers ⋈ ranked)
              ->  Subquery Scan on ranked
                    ->  WindowAgg
                          ->  Index Scan using idx_orders_cust_amount on orders
                                Index Cond: (status via partial/filter)
-- Single ordered scan feeds the WindowAgg; no self-join, no per-row correlated subquery.
```

### Example 4 — Production Grade: NULL-safe reconciliation (the NOT IN trap avoided)

```sql
-- "Find customers who have never placed a completed order" — a nightly reconciliation job.
-- Table sizes: customers ~1M, orders ~20M. orders.customer_id CAN be NULL (guest checkout).
-- The naive NOT IN version returns ZERO rows the moment one NULL exists in the subquery.
-- Index available: idx_orders_cust_status ON orders(customer_id, status)
-- Perf expectation: Hash Anti Join, single pass, O(N+M), correct under NULLs.
SELECT c.id, c.name, c.email
FROM customers c
WHERE NOT EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.customer_id = c.id
    AND o.status = 'completed'
)
ORDER BY c.created_at DESC;
```

```
Sort  (cost=... )  (actual rows=41230 loops=1)
  Sort Key: c.created_at DESC
  ->  Hash Anti Join  (cost=... rows=41230)
        Hash Cond: (c.id = o.customer_id)
        ->  Seq Scan on customers c  (actual rows=1000000 loops=1)
        ->  Hash
              ->  Index Scan using idx_orders_cust_status on orders o
                    Index Cond: (status = 'completed')
-- Hash Anti Join = the engine recognized NOT EXISTS as an anti-join.
-- Correct even with NULL customer_id rows in orders — they simply never match.
```

---

## 9. Wrong → Right Patterns

These are the exact wrong answers/queries interviewers wait to catch, the concrete failure, and the fix.

### Wrong 1: `NOT IN` with a NULLable subquery (the silent zero-row bug)

```sql
-- WRONG: if ANY order has customer_id = NULL, this returns ZERO rows, no error
SELECT * FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
-- Execution reality: NOT IN expands to id <> every value; id <> NULL is UNKNOWN,
-- so no row can ever satisfy the whole AND-chain → empty result set.

-- RIGHT: NOT EXISTS is NULL-safe and plans as an anti-join
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

### Wrong 2: Predicate on the outer table of a LEFT JOIN in WHERE

```sql
-- WRONG: turns the LEFT JOIN into an INNER JOIN, hiding order-less customers
SELECT c.id, COUNT(o.id) AS n
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'completed'   -- NULL rows fail this → dropped → LEFT becomes INNER
GROUP BY c.id;

-- RIGHT: move it into ON so unmatched customers survive with n = 0
SELECT c.id, COUNT(o.id) AS n
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.status = 'completed'
GROUP BY c.id;
```

### Wrong 3: Counting a fanned-out column after a one-to-many JOIN

```sql
-- WRONG: COUNT(o.id) counts order_items rows, not orders
SELECT c.id, COUNT(o.id) AS orders_placed
FROM customers c
JOIN orders o       ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id  = o.id
GROUP BY c.id;   -- an order with 5 items is counted 5 times

-- RIGHT: COUNT(DISTINCT o.id) — or pre-aggregate items before joining
SELECT c.id, COUNT(DISTINCT o.id) AS orders_placed
FROM customers c
JOIN orders o       ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id  = o.id
GROUP BY c.id;
```

### Wrong 4: Deep OFFSET pagination

```sql
-- WRONG: reads and throws away 100,000 rows to return 20 — gets slower every page
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 100000;

-- RIGHT: keyset — index jumps straight to the boundary, every page equally fast
SELECT * FROM orders
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

### Wrong 5: DELETE to empty a huge staging table

```sql
-- WRONG: 200M dead tuples, massive WAL, minutes of runtime, then VACUUM debt
DELETE FROM staging_events;

-- RIGHT: instant, minimal WAL, resets the identity sequence
TRUNCATE staging_events RESTART IDENTITY;
```

---

## 10. Performance Profile

Interviewers often turn a definition question into a scaling question: "and how does that behave at 100M rows?" Here's the profile behind the twenty answers.

| Pattern | 1M rows | 10M rows | 100M rows | The lever |
|---------|---------|----------|-----------|-----------|
| Selective `WHERE` + B-tree index | <1ms | <1ms | ~1ms | Index selectivity |
| Seq Scan (no usable index) | ~100ms | ~1s | ~10s+ | Add index |
| `OFFSET 100000` pagination | ~50ms | ~50ms | ~50ms + grows with offset | Keyset instead |
| `UNION` (dedup) vs `UNION ALL` | dedup adds a sort pass | 2–3× the ALL cost | can spill to disk | Drop needless dedup |
| `COUNT(DISTINCT col)` | sort/hash pass | noticeable | expensive, may spill | Approximate / covering index |
| `DELETE` whole table | seconds + VACUUM | tens of seconds | minutes + WAL flood | TRUNCATE |
| `TRUNCATE` whole table | ~ms | ~ms | ~ms | (already optimal) |
| N+1 (per-row queries) | N × RTT | N × RTT | unusable | One JOIN / batch |

Key scaling truths to state in an interview:
- **Index lookups scale logarithmically**; sequential scans scale linearly. That gap is the whole game at 100M rows.
- **OFFSET cost is proportional to the offset**, not the page size — the reason keyset pagination exists.
- **Any operation with an implicit sort or hash** (UNION, DISTINCT, ORDER BY without an index, `COUNT(DISTINCT)`) risks spilling to disk once the working set exceeds `work_mem`, at which point it slows 5–10×.
- **DELETE generates dead tuples and WAL proportional to rows touched**; TRUNCATE is O(1) in rows because it drops data files. This is the single starkest scaling contrast in the doc.

---

## 11. Node.js Integration

The interview sometimes shifts to "show me how you'd run this from your service." Parameterized `pool.query` with `pg`, correct result handling, and the NULL-safe patterns.

### 11.1 Parameterized query — never string-concatenate (SQL injection)

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// $1/$2 binding — the interview-correct way; the driver never interpolates into SQL text
async function topCustomers(minSpend, sinceDays) {
  const { rows } = await pool.query(
    `SELECT customer_id, SUM(total_amount) AS spend
     FROM orders
     WHERE status = 'completed'
       AND created_at >= NOW() - ($2 || ' days')::INTERVAL
     GROUP BY customer_id
     HAVING SUM(total_amount) > $1
     ORDER BY spend DESC`,
    [minSpend, sinceDays]
  );
  return rows;
}
```

### 11.2 Killing an N+1 with a single set-based query

```javascript
// Instead of N per-order customer lookups, one batched query with = ANY($1)
async function ordersWithCustomers(orderIds) {
  const { rows } = await pool.query(
    `SELECT o.id, o.total_amount, c.id AS customer_id, c.name
     FROM orders o
     JOIN customers c ON c.id = o.customer_id
     WHERE o.id = ANY($1)`,   // $1 is a JS array → PG array; one round-trip
    [orderIds]
  );
  return rows;
}
```

### 11.3 The NULL-safe anti-join (avoids the NOT IN trap in app code)

```javascript
async function customersWithNoOrders() {
  const { rows } = await pool.query(
    `SELECT c.id, c.name
     FROM customers c
     WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)`
  );
  return rows;   // correct even if orders.customer_id contains NULLs
}
```

### 11.4 Read-modify-write done safely (the isolation answer, in code)

```javascript
// Do the arithmetic IN SQL under a row lock — no lost update, no explicit FOR UPDATE needed
async function debitBalance(accountId, amount) {
  const { rows } = await pool.query(
    `UPDATE accounts
     SET balance = balance - $2
     WHERE id = $1 AND balance >= $2
     RETURNING balance`,
    [accountId, amount]
  );
  if (rows.length === 0) throw new Error('insufficient funds or account missing');
  return rows[0].balance;
}
```

A note on ORMs: most of these patterns (parameterization, eager-loading to kill N+1, `NOT EXISTS`) are expressible in Prisma/Drizzle/Sequelize/TypeORM/Knex, but the NULL-safe anti-join and window-based top-N usually need a raw escape hatch — covered next.

---

## 12. ORM Comparison

The interview angle here: "your ORM makes this easy — but do you know when it silently does the wrong thing?" The two ORM traps that map directly to this doc are **N+1 by default** (lazy relations) and **LEFT-vs-INNER join defaults**.

### Prisma

**Can it?** Kills N+1 via `include` (batches or joins); expresses HAVING/aggregates via `groupBy` + `having`. Window functions and `NOT EXISTS` need `$queryRaw`.

```typescript
// Eager-load to avoid N+1 — Prisma issues a single batched query, not one-per-order
const orders = await prisma.order.findMany({
  where: { status: 'completed' },
  include: { customer: true },   // required relation → INNER-JOIN semantics
});

// Aggregate + HAVING
const spenders = await prisma.order.groupBy({
  by: ['customerId'],
  where: { status: 'completed' },
  _sum: { totalAmount: true },
  having: { totalAmount: { _sum: { gt: 1000 } } },
});

// NULL-safe anti-join & windows need raw SQL
const noOrders = await prisma.$queryRaw`
  SELECT c.id FROM customers c
  WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)`;
```

**Where it breaks:** No explicit INNER-vs-LEFT control (inferred from FK nullability); no window functions. **Verdict:** great for CRUD and N+1 avoidance, raw SQL for analytics.

### Drizzle

**Can it?** Yes, and it's the most SQL-transparent — explicit `.innerJoin()`/`.leftJoin()`, `sql` template for windows and `NOT EXISTS`.

```typescript
const rows = await db
  .select({ id: orders.id, name: customers.name })
  .from(orders)
  .innerJoin(customers, eq(customers.id, orders.customerId))  // explicit, no LEFT-default trap
  .where(eq(orders.status, 'completed'));
```

**Where it breaks:** Windows/`NOT EXISTS` need the `sql` tag, but stay type-checked. **Verdict:** best typed control; the join type is always explicit, so no accidental LEFT/INNER surprise.

### Sequelize

**Can it?** Yes — but its `include` defaults to **LEFT OUTER JOIN**; you must set `required: true` for INNER. This is the single most common Sequelize interview gotcha.

```javascript
const orders = await Order.findAll({
  where: { status: 'completed' },
  include: [{ model: Customer, required: true }], // ← required:true = INNER JOIN
});
```

**Where it breaks:** Forgetting `required: true` silently keeps unmatched rows (LEFT). Windows/anti-joins → `sequelize.query()`. **Verdict:** usable, but the LEFT default is a footgun — always be explicit.

### TypeORM

**Can it?** Yes via QueryBuilder — `innerJoin`/`leftJoin` are explicit; lazy relations cause N+1 if you loop over them.

```typescript
const orders = await dataSource.getRepository(Order)
  .createQueryBuilder('o')
  .innerJoinAndSelect('o.customer', 'c')   // explicit INNER + eager select, no N+1
  .where('o.status = :s', { s: 'completed' })
  .getMany();
```

**Where it breaks:** `relations`/lazy loading in a loop = N+1; windows via `.getRawMany()`. **Verdict:** explicit joins are good; watch lazy relations.

### Knex

**Can it?** Yes — `.join()` defaults to INNER (unlike Sequelize), `.leftJoin()` explicit, `knex.raw()` for windows/`NOT EXISTS`.

```javascript
const rows = await knex('orders as o')
  .join('customers as c', 'c.id', 'o.customer_id')  // .join() = INNER by default (safe)
  .where('o.status', 'completed')
  .select('o.id', 'c.name');
```

**Where it breaks:** Complex ON / windows need `knex.raw()`. **Verdict:** most transparent; INNER default matches SQL's own.

### ORM Summary

| ORM | Default JOIN | N+1 fix | Window / NOT EXISTS | Interview gotcha |
|-----|--------------|---------|---------------------|------------------|
| Prisma | Inferred from FK | `include` (batched) | `$queryRaw` | No explicit join type |
| Drizzle | Must specify | manual join | `sql` tag (typed) | none major |
| Sequelize | **LEFT (danger!)** | `include`+`required` | `.query()` | forgetting `required: true` |
| TypeORM | Explicit | `innerJoinAndSelect` | `.getRawMany()` | lazy-relation N+1 |
| Knex | **INNER** | manual join | `knex.raw()` | raw for complex ON |

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given `orders(id, customer_id, total_amount, status, created_at)`:

Write a query that returns each `customer_id` with their number of completed orders and total completed spend, but only for customers whose completed spend exceeds 500. State explicitly which predicate goes in WHERE and which in HAVING, and why.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines JOINs + NULL logic)

Given `customers(id, name)` and `orders(id, customer_id, status)` where `orders.customer_id` may be NULL for guest checkouts:

1. Write a query listing customers who have **never** placed an order — and make it correct even though `orders.customer_id` contains NULLs. Explain why `NOT IN` would be wrong here.
2. Write a query listing all customers with their completed-order count, including customers with zero (so they appear with 0, not omitted). Identify the trap that would accidentally drop the zero-order customers.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A reporting query paginates the orders feed with `ORDER BY created_at DESC LIMIT 50 OFFSET 500000` and times out in production. Separately, a nightly job empties a 300M-row `audit_logs` staging table with `DELETE FROM audit_logs` and floods the WAL/disk.

1. Rewrite the pagination to be O(log N) per page and explain what the OFFSET version was doing that made it slow.
2. Fix the nightly cleanup and explain the MVCC/WAL difference between your fix and DELETE.
3. The keyset query sorts by `created_at`, which is not unique. Show how you keep it correct at page boundaries.

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

The interviewer says: *"Here's a query that's supposed to return every customer along with whether they have any completed order. It returns fewer rows than there are customers. Find and fix the bug, then tell me the isolation level you'd use if two requests could update the same customer's status concurrently, and why."*

```sql
SELECT c.id, c.name, o.status
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'completed';
-- Write your corrected query and your isolation-level reasoning here
```

---

## 14. Interview Questions

Section 6 already gave the twenty in Q/junior/principal/follow-up form. This section adds the **meta-questions** interviewers use to probe how you *think about* SQL — the ones with no flashcard answer.

### Junior Level

**Q: Walk me through what happens, in order, when the database runs `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT`.**

A: FROM/JOIN assembles the rows; WHERE filters individual rows before any grouping; GROUP BY collapses rows into groups; HAVING filters those groups; SELECT computes the output expressions and aliases; ORDER BY sorts (and may reference SELECT aliases because it runs after SELECT); LIMIT cuts the final rows. The single most useful consequence: WHERE can't see aggregates (they don't exist yet) and can't see SELECT aliases, while ORDER BY can see both.

---

**Q: When would you NOT add an index?**

A: On a small table (a Seq Scan is already cheap); on a low-selectivity column where the planner would ignore it anyway; on a write-heavy table where the maintenance cost outweighs read benefit; and when an existing composite index already covers the query via its leftmost prefix. An index you never use is pure write-tax and disk.

---

**Q: What's the difference between a query that's slow because of the plan versus slow because of data volume?**

A: A bad plan is fixable — wrong join algorithm, missing index, stale statistics — and EXPLAIN ANALYZE shows the mismatch (e.g., Seq Scan removing 99.99% of rows). Data-volume slowness is when the plan is already optimal and you're genuinely processing a lot of rows; the fix shifts to reducing the work (pre-aggregation, keyset pagination, materialized views) or scaling the hardware, not tweaking the plan.

### Principal Level

**Q: A candidate says "I always use UNION to be safe." Critique that.**

A: "Safe" here means they're paying for an implicit DISTINCT — a full sort or hash-aggregate over the combined result — on every query, whether or not duplicates are possible. When the two branches are disjoint by construction (different status partitions, different tables of distinct entities), the dedup can never remove anything, so it's pure wasted CPU and possible disk spill at scale. The principal default is UNION ALL, reaching for UNION only when overlap is genuinely possible and you want it removed. "Always UNION" trades a correctness non-issue for a real performance cost.

---

**Q: Explain why `NOT IN (subquery)` is considered dangerous but `IN (subquery)` isn't, at the level of three-valued logic.**

A: `NOT IN (v1, v2, ...)` is `x <> v1 AND x <> v2 AND ...`. If any vi is NULL, `x <> NULL` evaluates to UNKNOWN, and `TRUE AND UNKNOWN` is UNKNOWN — so the whole predicate can never be TRUE, and the query returns zero rows silently. `IN` is `x = v1 OR x = v2 OR ...`; a single matching non-NULL term makes it TRUE regardless of any NULL in the list, because `TRUE OR UNKNOWN` is TRUE. The asymmetry is that negation requires *every* comparison to hold, so one UNKNOWN poisons the conjunction, whereas the positive form only needs one TRUE. The fix is `NOT EXISTS`, which uses row-existence semantics rather than value comparison and is immune to the NULL.

---

**Q: Two concurrent transactions both do `SELECT balance` then `UPDATE balance = $new`. One update is lost. Diagnose it at the isolation/MVCC level and give three distinct fixes.**

A: This is a lost update. At READ COMMITTED, each transaction reads the same original balance in its snapshot, computes independently, and the second write overwrites the first — MVCC gives each a consistent read but doesn't serialize the read-modify-write. Fixes: (1) Do the arithmetic in SQL under the implicit row lock: `UPDATE accounts SET balance = balance - $amt` — the UPDATE re-reads and locks the row, so it's atomic. (2) Pessimistic lock: `SELECT ... FOR UPDATE` before the read, forcing the second transaction to wait. (3) Optimistic concurrency / higher isolation: add a `version` column and `WHERE version = $read_version` (retry on zero rows affected), or run at REPEATABLE READ/SERIALIZABLE and retry the serialization failure. The first is simplest and usually correct.

---

**Q: You're handed an EXPLAIN ANALYZE where estimated rows is 12 and actual rows is 3.2 million, and the query is slow. What's the story and the fix?**

A: The planner built its plan believing only 12 rows would flow through — so it likely chose a Nested Loop (great for tiny inputs, catastrophic for millions) and skipped indexes or hashing that would suit the real volume. The root cause is almost always stale or insufficient statistics: an `ANALYZE` hasn't run since a big data change, or the column has skew the default statistics target under-samples. Fix: `ANALYZE` the table (or raise `ALTER TABLE ... ALTER COLUMN ... SET STATISTICS`), then re-plan. If the estimate is wrong because of correlated columns, extended statistics (`CREATE STATISTICS`) can fix the dependency the planner is missing.

---

### Rapid-Fire Round (the one-liners interviewers fire in the last five minutes)

These come fast; the interviewer wants a crisp, correct sentence, not an essay.

**Q: Does `WHERE 1=1` do anything?** — No, it's a no-op the planner drops; developers add it so every real condition can be appended as `AND ...` when building SQL programmatically.

**Q: Is `SELECT COUNT(1)` faster than `SELECT COUNT(*)`?** — No. Identical plans; `COUNT(*)` is the idiomatic form. The `1` is not "counting the constant" — the planner treats them the same.

**Q: `WHERE col = NULL` — why no rows?** — `= NULL` is UNKNOWN, never TRUE. Use `IS NULL`. Equality never matches NULL.

**Q: Can a `PRIMARY KEY` be composite?** — Yes; the uniqueness applies to the tuple of columns, and it builds one composite unique B-tree.

**Q: `TRUNCATE` inside a transaction — can you roll it back in PostgreSQL?** — Yes, PostgreSQL's TRUNCATE is transactional (unlike MySQL, where it implicitly commits).

**Q: Does `ORDER BY` guarantee row order without it?** — No. Without `ORDER BY`, row order is undefined regardless of index or insertion order; never rely on incidental ordering.

**Q: `LIMIT` without `ORDER BY` — which rows?** — Arbitrary, non-deterministic. Always pair `LIMIT` with a deterministic `ORDER BY`.

**Q: Difference between `DISTINCT` and `GROUP BY` with no aggregate?** — Functionally equivalent for dedup; both may use the same sort/hash. Use `DISTINCT` for pure dedup, `GROUP BY` when you'll add aggregates.

**Q: What does `RETURNING` do?** — Returns the affected rows from `INSERT`/`UPDATE`/`DELETE` in one round-trip — no separate SELECT, no lost-update race between write and read-back.

**Q: `INT` vs `BIGINT` for a primary key at scale?** — `INT` caps at ~2.1B; a high-volume table will overflow it. Use `BIGINT` (or `bigint` identity) for anything that could exceed 2 billion rows over its lifetime — migrating later means a full table rewrite.

---

## 15. Mental Model Checkpoint

1. A query has `WHERE status = 'x'` and also `GROUP BY customer_id HAVING status = 'x'`. Which is valid, which errors, and if both were valid which is faster — and why does the logical execution order decide all three answers?

2. You write `LEFT JOIN orders o ON o.customer_id = c.id` and then `WHERE o.total > 100`. Before running it, predict whether order-less customers appear in the result. Now move `o.total > 100` into the ON clause and predict again. What changed and why?

3. A subquery in `NOT IN` returns the values `{4, 7, NULL}`. Reason through, term by term, why the outer query returns no rows — and state the exact three-valued-logic step that causes it.

4. Two indexes exist: `(customer_id)` and `(customer_id, created_at)`. Which queries can use each, and is the single-column one redundant? Explain via the leftmost-prefix rule.

5. `DELETE FROM t` and `TRUNCATE t` both empty the table. Describe what each leaves behind for the storage engine (dead tuples, WAL, sequence values, locks) and when that difference forces your hand.

6. A window function `SUM(x) OVER (PARTITION BY g)` and a `GROUP BY g` with `SUM(x)` compute the same sum. What's in the result set of each, and when does only the window version answer the question?

7. Deep OFFSET pagination and keyset pagination both return "page 5000." One reads 100,020 rows, the other reads ~20. Explain the mechanism, and what breaks in keyset if the sort column has duplicate values.

---

## 16. Quick Reference Card

```text
LOGICAL ORDER:  FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT

WHERE vs HAVING     WHERE = pre-aggregation row filter (index-usable)
                    HAVING = post-aggregation group filter (aggregates only)

LEFT JOIN trap      predicate on right table in WHERE → becomes INNER JOIN
                    fix: put the predicate in ON

NOT IN + NULL       NOT IN with a NULL in the set → ZERO rows. Use NOT EXISTS.
IN + NULL           positive IN is fine — one TRUE wins over UNKNOWN.

EXISTS vs IN        both plan as semi-join; EXISTS short-circuits, NULL-safe.
                    SELECT 1 vs SELECT * inside EXISTS: no difference.

UNION vs UNION ALL  UNION = implicit DISTINCT (sort/hash pass). Default to UNION ALL.

DELETE vs TRUNCATE  DELETE: MVCC dead tuples + WAL + triggers, WHERE-able, rollback.
                    TRUNCATE: drops data files, instant, RESTART IDENTITY, ACCESS EXCL lock,
                    transactional in PostgreSQL, CASCADE for FK'd tables.

ISOLATION           READ COMMITTED (default): per-statement snapshot; read-modify-write can lose.
                    REPEATABLE READ: txn-wide snapshot, no phantoms (PG).
                    SERIALIZABLE: SSI, as-if-serial, retry on failure.
                    Lost update fix: balance = balance - $x, or FOR UPDATE, or version column.

PAGINATION          OFFSET reads+discards skipped rows → deep pages slow.
                    Keyset: WHERE (sortcol, id) < ($1,$2) ORDER BY ... LIMIT n → O(log N).

INDEXING            equality cols first, one range col last (leftmost-prefix).
                    covering: INCLUDE(...) → Index Only Scan (keep table VACUUMed).
                    low-selectivity / function-wrapped col → index unused.

COUNTS              COUNT(*) all rows; COUNT(col) skips NULL; COUNT(DISTINCT) = sort/hash.
                    after 1-to-many JOIN: COUNT(DISTINCT o.id), not COUNT(o.id).

SLOW QUERY          EXPLAIN (ANALYZE, BUFFERS) first. Seq Scan + Rows Removed → index.
                    est ≫≪ actual → ANALYZE. Sort/Hash Disk: → work_mem.

INTERVIEW ONE-LINERS
  "WHERE filters rows before grouping; HAVING filters groups after — different execution phases."
  "A WHERE on the right side of a LEFT JOIN silently makes it an INNER JOIN."
  "NOT IN with a NULL returns zero rows; use NOT EXISTS."
  "UNION pays for a DISTINCT you probably don't need — default to UNION ALL."
  "TRUNCATE drops the data files; DELETE logs every row as a dead tuple."
  "OFFSET still reads the rows it skips; keyset pagination doesn't."
  "N+1 is a latency problem, not a query problem — one JOIN replaces N round-trips."
```

---

## Connected Topics

**Internals this builds on:**
- **Three-Valued Logic / NULL in Depth (Topic 07)** — the engine behind the NOT IN trap and the LEFT-JOIN-WHERE trap.
- **B-tree Indexes & the Planner** — selectivity, leftmost-prefix, covering indexes, Index Only Scans.
- **MVCC & Isolation** — snapshots, lost updates, dead tuples behind DELETE vs TRUNCATE.

**Prior/next SQL topics:**
- **Topic 11 — INNER JOIN in Depth**: fan-out and `COUNT(DISTINCT)`, referenced in Q2/Q13.
- **Topic 12 — LEFT/RIGHT JOIN**: the ON-vs-WHERE trap in full.
- **Topic 18 — The N+1 Query Problem**: the deep treatment behind Q4.
- **Topic 27 — EXISTS and NOT EXISTS**: the semi-join semantics behind Q9/Q10.
- **Topic 30 — Window Functions**: the row-preservation model behind Q5.
- **Topic 73 — Essential Extensions** (previous): the ecosystem tooling that rounds out production SQL.
- **Topic 75 — Live Coding Patterns** (next): the hands-on companion — writing these queries live under interview time pressure, from blank editor to correct result.

