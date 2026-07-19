# Topic 75 — Live Coding Patterns
### SQL Mastery Curriculum — Phase 11: SQL for Backend Developer Interviews

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you are a chef in a cooking competition. The judges hand you a mystery box: five ingredients you have never combined before. The clock starts. Amateurs freeze, stare at the box, then panic-fry everything at once and hope it tastes good. The professional does something different — and it looks almost slow at first.

The professional picks up each ingredient and says out loud: "This is salmon, farmed not wild, so it is fattier. This is a blood orange, not a navel — more acid. I have thirty minutes." They are **clarifying the mystery box** before touching a knife. Then they taste a tiny piece raw — "the salmon is already slightly salted" — that is **looking at the sample data**. Then they cook the simplest correct dish that uses all five ingredients — a pan sear with a citrus glaze — **the naive correct solution**. Only once a complete, edible plate exists do they ask: "Can I plate this more beautifully? Can I get a crispier skin?" — that is **optimisation, last**.

A SQL live-coding interview is exactly this mystery box. You are handed a schema you have never seen, a vague question ("find the top customers"), and a ticking clock with a person watching you. The amateur reads the question once and immediately starts typing `SELECT * FROM ...`. The professional slows down for ninety seconds, clarifies what "top" means, eyeballs three sample rows, writes the simplest query that is definitely correct, checks the weird edge cases (what if two customers tie? what if a customer has zero orders? what if `amount` is NULL?), and only then says "now let me make this fast."

The entire skill of live-coding SQL is not knowing more syntax than the next candidate. It is having a **repeatable framework** so that when the mystery box lands and your heart rate spikes, your hands already know the first five moves. This topic is that framework, drilled through worked examples until it is muscle memory.

---

## 2. Connection to SQL Internals

Live coding is a *meta* topic — it is about how you think, not a single SQL feature. But the candidates who win are the ones who can connect their answer to what the engine actually does. Every framework step below maps to an internal concept you have studied in earlier phases:

- **"Verbalise the execution plan"** means narrating the logical order — `FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT` (Topic 03) — and then the *physical* plan the planner would pick: Seq Scan vs Index Scan, Nested Loop vs **Hash Join** vs **Merge Join** (Topic 17). Saying "the planner will hash the smaller `categories` table and probe it with `products`" out loud signals seniority instantly.

- **"Handle NULLs"** connects to three-valued logic (Topic 07): a predicate that returns `UNKNOWN` is not `TRUE`, so `WHERE` drops it, `INNER JOIN` drops it, but `COUNT(col)` skips NULLs while `COUNT(*)` does not.

- **"Handle ties"** connects to window functions (Phase 8): `ROW_NUMBER` vs `RANK` vs `DENSE_RANK` produce different results precisely at a tie, and the engine computes them during the **WindowAgg** node after the sort.

- **"Optimise after correctness"** connects to the **B-tree** (Topic 05), the **buffer pool** / `shared_hit` counters, `work_mem` spills (`Batches > 1`), and MVCC visibility checks that make `COUNT(*)` a heap scan rather than a metadata lookup.

- **"Decompose with CTEs"** connects to how Postgres 12+ can inline a non-recursive CTE into the main plan (or materialise it with `MATERIALIZED`) — a detail that separates the candidate who thinks CTEs are "just readability" from the one who knows the optimisation-fence history.

The internal concept to name explicitly in an interview is the **planner's cost model**: everything you do in the "optimise" phase is an attempt to give the planner a cheaper path — an index it can use for an Index Scan, statistics fresh enough (`ANALYZE`) to estimate row counts correctly, or a rewrite that turns a Nested Loop over a Seq Scan into a Hash Join.

---

## 3. Logical Execution Order Context

There is no single clause this topic "lives in" — instead, the framework is about being able to *walk the entire order backwards and forwards* on demand. The order you must be able to recite cold:

```
FROM          ← which tables, which join algorithm
  JOIN / ON   ← matching rows, fan-out happens here
WHERE         ← row filter (cannot see SELECT aliases, cannot see aggregates)
GROUP BY      ← collapse into groups
HAVING        ← group filter (CAN see aggregates)
SELECT        ← projection, aliases assigned here
DISTINCT      ← dedupe the projected rows
window fns    ← computed over the post-GROUP-BY result set
ORDER BY      ← sort (CAN see SELECT aliases)
LIMIT/OFFSET  ← final truncation
```

Why this matters in a live-coding setting specifically:

- When the interviewer asks "why can't I use the alias `total_revenue` in my `WHERE`?", the answer is: `WHERE` runs *before* `SELECT`, so the alias does not exist yet. Move it to `HAVING` (if it is an aggregate) or `ORDER BY` (which runs after `SELECT`).
- When they ask "why is my `WHERE COUNT(*) > 5` an error?", the answer is: aggregates are computed at `GROUP BY`, which is *after* `WHERE`. Use `HAVING`.
- When they ask "my `LEFT JOIN` filter dropped the NULL rows I wanted to keep" — a predicate in `WHERE` runs after the join and nullifies the outer-join padding; move it into the `ON` (Topic 12).
- When they ask "how do I get the top 3 per group?", you reason: window functions run *after* `GROUP BY` but *before* `ORDER BY`/`LIMIT`, so a plain `LIMIT` cannot express "per group" — you need `ROW_NUMBER() OVER (PARTITION BY ...)` wrapped in a subquery, then filter.

Being able to place any clause in this order *out loud, on demand* is the single highest-signal skill in the entire interview. It is the skeleton every worked example below hangs on.

---

## 4. What Is the Live-Coding Framework?

Live-coding SQL is the practice of taking an unfamiliar, under-specified problem to a correct, then optimised, query while narrating your reasoning so an interviewer can follow it. The framework has five ordered phases — **Clarify → Sample Data → Naive Correct → Edge Cases → Optimise** — and correctness always precedes speed.

```
┌─ THE FIVE-PHASE FRAMEWORK ────────────────────────────────────────┐
│                                                                    │
│  1. CLARIFY        Restate the problem. Pin down every vague word. │
│     └── "top" = by count or revenue? "recent" = how many days?    │
│         ties → keep all or pick one? output grain = one row per?  │
│                                                                    │
│  2. SAMPLE DATA    Ask for / sketch 3-5 rows per table.            │
│     └── confirms grain, FKs, nullability, and the shape of the    │
│         answer BEFORE you write a line of SQL                     │
│                                                                    │
│  3. NAIVE CORRECT  Write the simplest query that is DEFINITELY    │
│     └── right. Readable > clever. No premature optimisation.       │
│         State expected row count out loud.                        │
│                                                                    │
│  4. EDGE CASES     Walk NULLs, ties, empty tables, duplicates,    │
│     └── fan-out, division-by-zero, timezone boundaries. Fix each.  │
│                                                                    │
│  5. OPTIMISE       ONLY after it is correct. Name the index, the  │
│     └── join algorithm, the rewrite. Justify with EXPLAIN.         │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

Annotated breakdown of each phase:

```
CLARIFY
 │
 ├── restate the ask in your own words          → confirms shared understanding
 ├── define every ambiguous noun                → "active", "top", "recent", "churned"
 │     └── these words hide 80% of the bugs
 ├── confirm the OUTPUT GRAIN                    → "one row per customer" vs "per order"
 │     └── the single most important question
 ├── confirm tie-breaking rule                  → RANK vs ROW_NUMBER decision
 └── confirm scale                              → 1K rows? 100M? changes the plan

SAMPLE DATA
 │
 ├── ask for 3-5 example rows per table         → or invent them and confirm
 ├── identify primary keys and foreign keys     → drives join direction
 ├── identify nullable columns                  → NULL handling is a top source of bugs
 └── trace ONE expected output row by hand      → your correctness oracle

NAIVE CORRECT
 │
 ├── FROM the driving table first               → the grain you want to preserve
 ├── JOIN outward, watch fan-out                → 1-to-many multiplies rows
 ├── WHERE filters, GROUP BY collapses          → recite the logical order aloud
 └── SELECT only what is asked                  → no SELECT * in an interview

EDGE CASES
 │
 ├── NULL in join key / aggregate / filter      → three-valued logic
 ├── TIES at the ranking boundary               → RANK / DENSE_RANK / ROW_NUMBER
 ├── EMPTY input (customer with zero orders)     → INNER vs LEFT JOIN, COUNT=0
 ├── DUPLICATES / fan-out                        → COUNT(DISTINCT), pre-aggregate
 └── DIVISION BY ZERO                            → NULLIF(denominator, 0)

OPTIMISE
 │
 ├── read the plan aloud                         → Seq Scan? Nested Loop? spill?
 ├── propose the index                           → composite, covering, partial
 ├── propose the rewrite                         → EXISTS over IN, pre-aggregate
 └── re-check correctness did not change         → optimisation must be behaviour-neutral
```

---

## 5. Why Live-Coding Mastery Matters in Production

The interview is a proxy, but the skill is genuinely load-bearing on the job. Here is what breaks without it:

1. **Under-specified tickets ship the wrong query.** A product manager writes "show top customers on the dashboard." Without the *clarify* reflex, an engineer ships "top by lifetime order count" when finance meant "top by trailing-90-day revenue." The dashboard is technically a working query and completely wrong. The clarify phase is a production habit, not an interview trick.

2. **Silent NULL and fan-out bugs reach the database.** The same three edge cases that trip candidates — NULL join keys dropping rows (Topic 11), one-to-many fan-out inflating `SUM`/`COUNT` (Topic 20), division by zero on an empty cohort — are the exact bugs that produce a subtly wrong revenue report that nobody catches for a quarter. Engineers who have internalised the edge-case checklist catch these in code review.

3. **"Make it work then make it fast" prevents premature-optimisation rabbit holes.** Engineers who optimise before they have a correct query waste hours hand-tuning a `LATERAL` join that returns the wrong number. The discipline of *correct first* is what ships features on time.

4. **Verbalising the plan is how incidents get resolved.** At 2 a.m. when a query that ran in 20 ms now takes 40 s, the engineer who can read `EXPLAIN ANALYZE` out loud — "the Nested Loop's inner side lost its Index Scan and went to Seq Scan because statistics are stale" — resolves the incident. The one who only knows `SELECT` syntax pages someone senior.

5. **Decomposition with CTEs is maintainability.** A 200-line reporting query written as one nested pyramid is unreviewable and unmodifiable. The same logic decomposed into named CTEs — `active_customers`, `recent_orders`, `revenue_per_customer` — is a query the next engineer can actually change safely. The interview rewards this because production rewards it.

The cost of *not* having the framework is measured in wrong dashboards, quarter-long silent bugs, blown estimates, and unresolved 2 a.m. incidents. The interview is testing for exactly the reflexes that prevent those.

---

## 6. Deep Technical Content

### 6.1 The Clarify Phase — The Questions That Change the Query

Every vague word in a problem statement maps to a concrete SQL decision. Train yourself to hear the ambiguity:

| Vague word | Question to ask | SQL consequence |
|------------|-----------------|-----------------|
| "top N" | By what metric? Ties kept or broken? | `ORDER BY` column; `RANK` vs `ROW_NUMBER` |
| "recent" | How many days? Calendar or rolling? | `WHERE created_at >= NOW() - INTERVAL` |
| "active" | Logged in? Ordered? In last X days? | which table and predicate |
| "customer" | Include those with zero orders? | `INNER` vs `LEFT JOIN` |
| "revenue" | Gross, net, before/after refunds? | which columns, which `SUM` |
| "per month" | Calendar month or 30-day window? | `DATE_TRUNC('month', ...)` |
| "unique" | Distinct users or distinct sessions? | `COUNT(DISTINCT ...)` grain |
| "average order" | Per order or per customer? | where the `AVG`/`GROUP BY` sits |

The single most valuable clarify question is: **"What is the grain of the output — one row per what?"** If you can answer "one row per customer per month," the entire structure of the query is decided: you `GROUP BY customer_id, DATE_TRUNC('month', ...)`.

The second most valuable: **"What should happen to rows that don't match / are empty / are NULL?"** This one question pre-empts the three biggest edge-case bugs at once.

### 6.2 The Sample-Data Phase — Your Correctness Oracle

Before writing SQL, sketch tiny tables. This is not busywork — it is how you *verify* your query later. Invent 3-5 rows that deliberately include the edge cases you expect:

```
users                     orders
id | name    | region     id | user_id | amount | status     | created_at
---+---------+-------     ---+---------+--------+------------+------------
1  | Alice   | US         10 | 1       | 100.00 | completed  | 2026-07-01
2  | Bob     | US         11 | 1       | 50.00  | refunded   | 2026-07-02
3  | Carol   | EU         12 | 2       | 100.00 | completed  | 2026-07-03
4  | Dave    | NULL       13 | NULL    | 999.00 | completed  | 2026-07-04  ← orphan FK
                          14 | 4       | NULL   | completed  | 2026-06-01  ← NULL amount
                          -- Carol (id 3) has NO orders                    ← empty case
```

This 8-row sketch already contains: a NULL region (Dave), an orphaned order with NULL `user_id` (order 14 → note it will be dropped by INNER JOIN), a NULL amount (order 14), a customer with zero orders (Carol), a refunded order (should "revenue" include it?), and a tie in amount (Alice and Carol both at 100). When you write your query, you run it *against this sketch in your head* and check the output row by row. That is the oracle that catches bugs before the interviewer does.

### 6.3 The Naive-Correct Phase — Readable Beats Clever

The naive solution should be the simplest query you are *certain* is correct. Resist every urge to optimise. State the expected output shape aloud before running:

```sql
-- Ask: total completed revenue per customer, last 90 days
-- Grain: one row per customer who has >=1 completed order in window
-- Naive: straight join + group, no cleverness
SELECT
  u.id,
  u.name,
  SUM(o.amount) AS revenue      -- will revisit: refunds? NULL amount?
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed'
  AND o.created_at >= NOW() - INTERVAL '90 days'
GROUP BY u.id, u.name
ORDER BY revenue DESC;
```

This is correct for the happy path. It is *deliberately* not the final answer — it excludes zero-order customers (INNER JOIN), it may double-count if you later add a join to `order_items`, and `SUM` of a NULL amount is fine but a NULL-only group would give NULL. You write it first anyway, because a correct-but-incomplete query you can *evolve* beats a clever-but-wrong query you have to *debug*.

### 6.4 The Edge-Case Phase — The Five-Point Checklist

Run every query through these five checks, out loud:

**1. NULLs.** Three places they bite:
- Join key NULL → INNER JOIN silently drops the row (Topic 11). `orders.user_id = NULL` never matches.
- Aggregate over NULL → `SUM`, `AVG`, `COUNT(col)` *skip* NULLs; `COUNT(*)` counts the row. So `AVG(amount)` ignores NULL-amount rows entirely — is that what you want?
- Filter on NULL → `WHERE amount > 100` drops NULL-amount rows (UNKNOWN is not TRUE); `WHERE amount IS NULL` is the only way to catch them.

**2. Ties.** At a ranking boundary the choice of window function is the whole answer:

```sql
-- Three customers tied for 2nd place by revenue. "Top 2" means...?
ROW_NUMBER() OVER (ORDER BY revenue DESC)  -- picks ONE arbitrarily → exactly 2 rows
RANK()       OVER (ORDER BY revenue DESC)  -- 1,2,2 → gaps → could return 3 rows
DENSE_RANK() OVER (ORDER BY revenue DESC)  -- 1,2,2,3 → no gaps → keeps all ties, next is 3
```

Always ask the interviewer which they want, then say why you chose the function.

**3. Empty input.** A customer with zero orders:
- INNER JOIN → excluded entirely.
- LEFT JOIN + `SUM` → `SUM` of zero rows is **NULL**, not 0. Wrap in `COALESCE(SUM(o.amount), 0)`.
- `COUNT(o.id)` after LEFT JOIN → correctly 0 (counts non-NULL ids). `COUNT(*)` → wrongly 1 (counts the padded NULL row).

**4. Duplicates / fan-out.** Joining one-to-many multiplies the driving row (Topic 11 §6.2). `SUM(o.amount)` after joining `orders` to `order_items` counts each order's amount once per item. Fix with `COUNT(DISTINCT o.id)` or pre-aggregate in a CTE.

**5. Division by zero.** Any ratio needs a guard:

```sql
ROUND(revenue / NULLIF(order_count, 0), 2)  -- NULLIF turns 0 into NULL → result NULL, not error
COALESCE(ROUND(revenue / NULLIF(order_count, 0), 2), 0)  -- if you want 0 instead of NULL
```

### 6.5 Decomposition — Building with CTEs

Complex problems become tractable when you name the intermediate results. A CTE-decomposed query reads like a paragraph:

```sql
WITH recent_completed AS (            -- step 1: the relevant orders
  SELECT user_id, id AS order_id, amount
  FROM orders
  WHERE status = 'completed'
    AND created_at >= NOW() - INTERVAL '90 days'
),
per_customer AS (                     -- step 2: collapse to one row per customer
  SELECT user_id,
         COUNT(*)     AS order_count,
         SUM(amount)  AS revenue
  FROM recent_completed
  GROUP BY user_id
),
ranked AS (                           -- step 3: rank them
  SELECT *,
         RANK() OVER (ORDER BY revenue DESC) AS revenue_rank
  FROM per_customer
)
SELECT u.name, r.order_count, r.revenue, r.revenue_rank
FROM ranked r
JOIN users u ON u.id = r.user_id
WHERE r.revenue_rank <= 10
ORDER BY r.revenue_rank;
```

Each CTE is independently checkable. In the interview you can literally say "let me build this bottom-up: first the relevant orders, then per-customer aggregates, then the ranking." That narration is worth as much as the query itself. Note (Topic 24): Postgres 12+ *inlines* non-recursive CTEs into the main plan by default, so this decomposition has **no performance cost** versus a nested subquery — you get readability for free. Add `MATERIALIZED` only when you *want* the optimisation fence (e.g., an expensive CTE referenced many times).

### 6.6 Verbalising the Execution Plan

Once you have a correct query, narrate what the engine will do — *before* you even run `EXPLAIN`. This is the single strongest seniority signal:

> "`users` has 100K rows, `orders` has 5M. The planner will probably filter `orders` by `status` and `created_at` first — if there's a composite index on `(status, created_at)` it's an Index Scan, otherwise a Seq Scan. Then it hashes the filtered `users` — the smaller side — and probes with the order rows, so I expect a Hash Join. The GROUP BY becomes a HashAggregate. Finally a Sort for the ORDER BY, or it may reuse the aggregate's output. If revenue-per-customer is highly selective the Sort is cheap."

You have just demonstrated knowledge of join algorithms, index usage, aggregation strategy, and the cost model — without running anything. When you then run `EXPLAIN ANALYZE` and it matches, you look like you can predict the future.

### 6.7 Optimise Last — The Rewrite Toolkit

Only after correctness, reach for these:

- **Index the filter + join columns.** `CREATE INDEX ON orders(status, created_at)` for the WHERE; `orders(user_id)` for the join.
- **EXISTS over IN for existence.** `WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id)` stops at the first match; `IN (SELECT ...)` may materialise the whole set (Topic 27).
- **Pre-aggregate to kill fan-out** (Topic 11 §6.8) — aggregate `order_items` in a CTE before joining, so the join is one-to-one.
- **`LATERAL` for top-N-per-group** — one index-driven probe per group beats a giant window sort when N is small.
- **Partial index for a hot predicate** — `CREATE INDEX ON orders(created_at) WHERE status = 'completed'` if almost all queries filter to completed.
- **Covering index** — add the selected columns with `INCLUDE` so the plan is Index-Only Scan and never touches the heap.

Always re-run against your sample-data oracle after a rewrite: an optimisation that changes the answer is a bug, not an optimisation.

### 6.8 Common Live-Coding Problem Archetypes

Most interview problems are one of a handful of shapes. Recognising the archetype instantly gives you the skeleton:

| Archetype | Signature phrase | Skeleton |
|-----------|------------------|----------|
| Top-N overall | "top 10 customers" | `GROUP BY` → `ORDER BY` → `LIMIT` |
| Top-N per group | "top 3 products per category" | `ROW_NUMBER() OVER (PARTITION BY ...)` then filter |
| Nth highest | "second highest salary" | `DENSE_RANK` = 2, or `OFFSET 1 LIMIT 1` |
| Running total | "cumulative revenue by day" | `SUM(...) OVER (ORDER BY day)` |
| Gaps / islands | "consecutive login streak" | `ROW_NUMBER` difference trick |
| Existence | "customers who never ordered" | `NOT EXISTS` / `LEFT JOIN ... IS NULL` |
| Pivot | "revenue by month as columns" | `SUM(CASE WHEN ... )` / `FILTER (WHERE ...)` |
| Dedupe latest | "most recent order per customer" | `DISTINCT ON` or `ROW_NUMBER=1` |
| Median / percentile | "median order value" | `PERCENTILE_CONT(0.5) WITHIN GROUP` |

When the interviewer speaks, silently classify the archetype first. "Top 3 per category" → you already know it is a partitioned `ROW_NUMBER` before they finish the sentence.

### 6.9 A Fully Narrated Walkthrough — the Framework in Real Time

To see the five phases as a continuous performance, here is a full transcript of how the answer *sounds* when narrated well. The prompt: *"Find users who spent more than the average user."*

**Clarify (spoken):** "Let me make sure I understand. 'Spent' — is that total lifetime completed-order revenue per user? And 'the average user' — average of *per-user totals*, or the average order amount across all orders? Those are different numbers. I'll assume: each user's total completed revenue, compared against the mean of those per-user totals. Also — do we include users with zero orders in the average? Including them drags the mean down. I'll assume we compute the average over users who have ordered, and return only users strictly above it. Ties on the boundary — a user exactly at the average — excluded, since 'more than'. Grain of output: one row per qualifying user."

**Sample data (spoken):** "Let me sketch. Alice 300, Bob 100, Carol 200, Dave 0 orders. Average of the three who ordered = 200. So strictly-more-than-200 returns just Alice. Carol at exactly 200 is excluded. Dave with no orders isn't in the average and isn't returned. That's my oracle: the answer should be `[Alice]`."

**Naive correct (typed and spoken):**

```sql
WITH user_spend AS (
  SELECT u.id, u.name, SUM(o.amount) AS total_spend
  FROM users u
  JOIN orders o ON o.user_id = u.id AND o.status = 'completed'
  GROUP BY u.id, u.name
)
SELECT id, name, total_spend
FROM user_spend
WHERE total_spend > (SELECT AVG(total_spend) FROM user_spend)
ORDER BY total_spend DESC;
-- "The CTE is per-user totals — only users who ordered, because INNER JOIN.
--  The subquery averages those totals. Against my sketch: avg=200, returns Alice. Correct."
```

**Edge cases (spoken, walking the checklist):** "NULLs — `SUM(amount)` skips NULL amounts, fine; `AVG` over the CTE skips nothing because every row has a total. Ties — 'more than' with `>` excludes the exact-average user, matches the ask. Empty — if no user has ordered, the CTE is empty, `AVG` of nothing is NULL, `total_spend > NULL` is UNKNOWN, zero rows returned — no crash, sensible. Fan-out — I only joined `orders`, not `order_items`, so no multiplication. Division by zero — none here, `AVG` handles the empty case as NULL. I think this is correct."

**Optimise (spoken):** "The CTE scans `orders` filtered by status — I'd want an index on `orders(status, user_id) INCLUDE (amount)` so it's an index-only scan feeding the aggregate. The average subquery re-scans the CTE; since Postgres 12 inlines this CTE, the planner may compute the aggregate over the same materialised result — I'd check `EXPLAIN` to confirm it isn't scanning `orders` twice. If it is, I'd add `MATERIALIZED` to the CTE to force single evaluation. Answer unchanged either way."

That transcript — clarify, oracle, correct, checklist, plan — is exactly what earns a strong-hire. Notice the candidate never once guessed silently; every assumption was surfaced and every edge case named.

---

## 7. EXPLAIN — Reading a Plan Live

Being able to read `EXPLAIN ANALYZE` out loud during the optimise phase is the highest-signal moment of the interview. Here is the plan for the naive query from §6.3, before any index exists:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.id, u.name, SUM(o.amount) AS revenue
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed'
  AND o.created_at >= NOW() - INTERVAL '90 days'
GROUP BY u.id, u.name
ORDER BY revenue DESC;
```

```
Sort  (cost=48210.55..48460.55 rows=100000 width=48)
      (actual time=612.4..628.9 rows=87213 loops=1)
  Sort Key: (sum(o.amount)) DESC
  Sort Method: quicksort  Memory: 9812kB
  ->  HashAggregate  (cost=38120.00..39620.00 rows=100000 width=48)
                     (actual time=520.1..560.3 rows=87213 loops=1)
        Group Key: u.id, u.name
        Batches: 1  Memory Usage: 14337kB
        ->  Hash Join  (cost=3400.00..33120.00 rows=1000000 width=22)
                       (actual time=42.1..380.7 rows=980142 loops=1)
              Hash Cond: (o.user_id = u.id)
              ->  Seq Scan on orders o  (cost=0.00..27000.00 rows=1000000 width=14)
                                        (actual time=0.03..210.4 rows=980142 loops=1)
                    Filter: ((status = 'completed') AND
                             (created_at >= (now() - '90 days'::interval)))
                    Rows Removed by Filter: 4019858
              ->  Hash  (cost=1800.00..1800.00 rows=100000 width=12)
                        (actual time=41.8..41.8 rows=100000 loops=1)
                    Buckets: 131072  Batches: 1  Memory Usage: 5514kB
                    ->  Seq Scan on users u  (cost=0.00..1800.00 rows=100000 width=12)
                                             (actual time=0.01..18.2 rows=100000 loops=1)
Buffers: shared hit=1204 read=26918
Planning Time: 0.42 ms
Execution Time: 634.1 ms
```

**Reading it aloud, bottom-up:**

- `Seq Scan on orders` with `Rows Removed by Filter: 4019858` — this is the smoking gun. The engine read all 5M order rows and threw away 4M because there is **no index on `(status, created_at)`**. This scan is 210 ms of the 634 ms.
- `Seq Scan on users` builds the `Hash` — 100K rows, fits in `work_mem` (`Batches: 1`), fine.
- `Hash Join` probes with the 980K surviving order rows — this is the correct algorithm choice for a large equi-join with no useful index.
- `HashAggregate` collapses to 87,213 customers — no spill (`Batches: 1`).
- `Sort` on `sum(o.amount)` DESC — in-memory quicksort, 9.8 MB, cheap.
- `Buffers: read=26918` — 26,918 pages read from disk (not cache), dominated by the orders Seq Scan.

**The fix you propose:**

```sql
CREATE INDEX idx_orders_status_created ON orders (status, created_at) INCLUDE (user_id, amount);
```

After this index, the `Seq Scan on orders` becomes an **Index Scan** (or Bitmap Index Scan) that reads only the ~980K matching rows instead of all 5M, and because the index `INCLUDE`s `user_id` and `amount` it can be an **Index-Only Scan** — no heap access at all. Expected drop: from ~634 ms to ~120 ms. That is the complete optimise-phase narration an interviewer wants to hear: identify the offending node, name the cost, propose the index, predict the new plan.

---

## 8. Query Examples

### Example 1 — Basic: Second-Highest Value (a classic warm-up)

```sql
-- Q: "Find the second-highest order amount."
-- Clarify: ties? If two orders share the max, is the 2nd-highest the
--          same value or the next distinct value? Assume: next DISTINCT value.
SELECT MAX(amount) AS second_highest
FROM orders
WHERE amount < (SELECT MAX(amount) FROM orders);
-- Reads: the max below the overall max = the second distinct value.
-- Edge case: only one distinct amount → inner < outer is empty → MAX = NULL. Correct.
-- Edge case: empty table → both NULL. Correct (no error).
```

### Example 2 — Intermediate: Top 3 Products per Category (top-N-per-group archetype)

```sql
-- Q: "Show the top 3 products by revenue in each category."
-- Clarify: revenue = SUM(quantity * unit_price) from completed orders.
--          Ties at rank 3 → keep all (use RANK), confirmed with interviewer.
WITH product_revenue AS (
  SELECT
    p.category_id,
    p.id   AS product_id,
    p.name AS product_name,
    SUM(oi.quantity * oi.unit_price) AS revenue
  FROM order_items oi
  JOIN orders   o ON o.id = oi.order_id AND o.status = 'completed'
  JOIN products p ON p.id = oi.product_id
  GROUP BY p.category_id, p.id, p.name
),
ranked AS (
  SELECT *,
    RANK() OVER (PARTITION BY category_id ORDER BY revenue DESC) AS rk
  FROM product_revenue
)
SELECT c.name AS category, r.product_name, r.revenue, r.rk
FROM ranked r
JOIN categories c ON c.id = r.category_id
WHERE r.rk <= 3
ORDER BY c.name, r.rk;
-- Why RANK not ROW_NUMBER: interviewer wanted all products tied at 3rd kept.
-- Why the CTE: keeps the aggregation grain (per product) separate from the
--   window (per category), which is far easier to reason about and verify.
```

### Example 3 — Production Grade: Customer Cohort Retention Report

```sql
-- CONTEXT
--   users:        2M rows, PK on id, index on (created_at)
--   orders:       40M rows, index on (user_id), index on (status, created_at)
--   Question: "For customers who signed up in each month of 2026,
--              what fraction placed at least one completed order within
--              30 days of signup?" (activation rate per signup cohort)
--   Grain of output: one row per signup-month cohort.
--   Perf expectation: < 300 ms; must not fan-out; must handle
--                     cohorts with zero activations (division by zero guard).

WITH cohorts AS (                          -- one row per user, tagged by signup month
  SELECT
    id AS user_id,
    DATE_TRUNC('month', created_at) AS cohort_month,
    created_at AS signup_at
  FROM users
  WHERE created_at >= '2026-01-01'
    AND created_at <  '2027-01-01'
),
activated AS (                             -- did this user order within 30 days? (existence, no fan-out)
  SELECT c.user_id, c.cohort_month
  FROM cohorts c
  WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = c.user_id
      AND o.status  = 'completed'
      AND o.created_at >= c.signup_at
      AND o.created_at <  c.signup_at + INTERVAL '30 days'
  )
)
SELECT
  c.cohort_month,
  COUNT(*)                                  AS cohort_size,
  COUNT(a.user_id)                          AS activated_users,
  ROUND(
    100.0 * COUNT(a.user_id) / NULLIF(COUNT(*), 0)  -- guard: empty cohort → NULL not error
  , 2)                                      AS activation_rate_pct
FROM cohorts c
LEFT JOIN activated a ON a.user_id = c.user_id   -- LEFT keeps non-activated users in the count
GROUP BY c.cohort_month
ORDER BY c.cohort_month;
```

```sql
EXPLAIN (ANALYZE, BUFFERS)  -- (same query)
```

```
Sort  (cost=? actual time=248.9..249.0 rows=12 loops=1)
  Sort Key: c.cohort_month
  ->  HashAggregate  (actual time=248.1..248.7 rows=12 loops=1)
        Group Key: c.cohort_month
        ->  Hash Right Join  (actual time=88.2..214.5 rows=310204 loops=1)
              Hash Cond: (a.user_id = c.user_id)
              ->  Subquery Scan on a  (actual time=44.1..150.3 rows=142889 loops=1)
                    ->  Hash Semi Join  (actual time=44.1..138.9 rows=142889 loops=1)
                          Hash Cond: (c_1.user_id = o.user_id)
                          Join Filter: (o.created_at >= c_1.signup_at AND
                                        o.created_at < c_1.signup_at + '30 days')
                          ->  Index Scan using users_created_at_idx on users
                                (actual rows=310204 loops=1)
                          ->  Hash  (actual rows=? loops=1)
                                ->  Index Scan using orders_status_created_idx on orders o
                                      Index Cond: (status = 'completed')
              ->  Hash  (actual rows=310204 loops=1)
                    ->  Index Scan using users_created_at_idx on users
Buffers: shared hit=? read=?
Execution Time: 249.4 ms
```

Reading: the `EXISTS` became a **Hash Semi Join** (the planner's semi-join transform — it stops at the first match per user, no fan-out), both `users` scans use the `created_at` index, and the whole thing lands at 249 ms, under the 300 ms budget. The `NULLIF` guard means a hypothetical empty cohort yields `NULL` rather than a divide-by-zero error. This is a complete, defensible, production-grade answer.

### Example 4 — The Gaps-and-Islands Curveball: Longest Login Streak

```sql
-- Q: "What is each user's longest streak of consecutive days with at least one session?"
-- Clarify: 'consecutive days' = calendar days, no gaps. Multiple sessions same day
--          count as one active day. Output grain: one row per user, their max streak.
-- Archetype recognised instantly: gaps-and-islands → the ROW_NUMBER difference trick.
WITH active_days AS (                         -- step 1: distinct active calendar days per user
  SELECT DISTINCT user_id, DATE(created_at) AS day
  FROM sessions
),
grouped AS (                                  -- step 2: the island key
  SELECT
    user_id,
    day,
    -- consecutive days share the same (day - row_number) offset:
    day - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY day))::INT AS island
  FROM active_days
),
streaks AS (                                  -- step 3: measure each island
  SELECT user_id, island, COUNT(*) AS streak_len
  FROM grouped
  GROUP BY user_id, island
)
SELECT user_id, MAX(streak_len) AS longest_streak   -- step 4: pick the max island per user
FROM streaks
GROUP BY user_id
ORDER BY longest_streak DESC;
-- Why it works: for a run of consecutive dates, `day` and the row number both
--   increase by 1 in lockstep, so `day - row_number` is constant WITHIN a run and
--   jumps at each gap. Grouping by that constant collapses each run into one island.
-- Edge cases voiced: a single active day → streak_len 1 (correct); a user with no
--   sessions → absent from output (INNER via the DISTINCT source; use LEFT JOIN to
--   users if zero-streak users must appear as 0).
```

This is the archetype interviewers use to separate memorisers from reasoners. The candidate who *names* it ("this is gaps-and-islands, the row-number-difference trick") and then derives *why* the offset is constant within a run demonstrates exactly the pattern-recognition-plus-first-principles the framework builds.

---

## 9. Wrong → Right Patterns

### Wrong 1: Optimising before it is correct

```sql
-- WRONG: candidate jumps straight to a "clever" LATERAL before confirming grain,
--        and it returns the wrong number because of an unhandled tie + fan-out
SELECT u.name, top.amount
FROM users u
CROSS JOIN LATERAL (
  SELECT o.amount FROM orders o WHERE o.user_id = u.id
  ORDER BY o.amount DESC LIMIT 1
) top;
-- Interviewer asked for "total revenue per customer", NOT "largest single order".
-- The candidate optimised a query that answers the wrong question.
```

```sql
-- RIGHT: clarify first, write the naive-correct total, THEN optimise if needed
SELECT u.id, u.name, COALESCE(SUM(o.amount), 0) AS total_revenue
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'completed'
GROUP BY u.id, u.name
ORDER BY total_revenue DESC;
-- Correct grain (per customer), correct metric (total), keeps zero-order customers.
```

### Wrong 2: SUM after a one-to-many join (fan-out)

```sql
-- WRONG: joining orders to order_items multiplies each order's amount by its item count
SELECT u.name, SUM(o.amount) AS revenue
FROM users u
JOIN orders      o  ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id     -- fan-out! one order → many items
GROUP BY u.name;
-- An order of $100 with 3 items is summed as $300. Silently, catastrophically wrong.
```

```sql
-- RIGHT: pre-aggregate items, or don't join what you don't need to sum
SELECT u.name, SUM(o.amount) AS revenue
FROM users u
JOIN orders o ON o.user_id = u.id
GROUP BY u.name;
-- If you genuinely need item data too, pre-aggregate in a CTE keyed by order_id first.
```

### Wrong 3: SUM over a LEFT JOIN returns NULL, not 0

```sql
-- WRONG: assuming zero-order customers show 0 revenue
SELECT u.name, SUM(o.amount) AS revenue
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'completed'
GROUP BY u.name;
-- Carol has no completed orders → her row has one NULL-padded order →
-- SUM(NULL) = NULL, so Carol shows revenue = NULL, not 0. Breaks downstream math.
```

```sql
-- RIGHT: COALESCE the aggregate
SELECT u.name, COALESCE(SUM(o.amount), 0) AS revenue
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'completed'
GROUP BY u.name;
```

### Wrong 4: WHERE on a LEFT-JOINed table turns it into an INNER JOIN

```sql
-- WRONG: intending to keep all customers, but the WHERE nukes the padded rows
SELECT u.name, COUNT(o.id) AS completed_orders
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed'          -- runs AFTER the join; NULL-padded rows fail this
GROUP BY u.name;
-- Customers with zero completed orders have o.status = NULL → 'completed' comparison
-- is UNKNOWN → row dropped. Silently became an INNER JOIN. Zero-order customers vanish.
```

```sql
-- RIGHT: move the predicate into the ON clause so it filters BEFORE padding
SELECT u.name, COUNT(o.id) AS completed_orders
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'completed'
GROUP BY u.name;
-- Now customers with no completed order keep a NULL-padded row → COUNT(o.id) = 0.
```

### Wrong 5: ROW_NUMBER when the problem wanted ties kept

```sql
-- WRONG: "give me everyone tied for the highest salary" but ROW_NUMBER picks one
SELECT name, salary
FROM (
  SELECT name, salary, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
  FROM employees
) t
WHERE rn = 1;
-- If three people earn 200K, this returns exactly ONE of them, arbitrarily.
```

```sql
-- RIGHT: RANK keeps all ties at the top
SELECT name, salary
FROM (
  SELECT name, salary, RANK() OVER (ORDER BY salary DESC) AS rk
  FROM employees
) t
WHERE rk = 1;
-- All three 200K earners returned. Choose the window function to match the tie rule.
```

### Wrong 6: Division without a zero guard

```sql
-- WRONG: average order value crashes for a customer with zero orders
SELECT u.name, SUM(o.amount) / COUNT(o.id) AS avg_order_value
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.name;
-- Zero-order customer: COUNT(o.id) = 0 → division by zero ERROR aborts the whole query.
```

```sql
-- RIGHT: NULLIF the denominator
SELECT u.name,
       ROUND(SUM(o.amount) / NULLIF(COUNT(o.id), 0), 2) AS avg_order_value
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.name;
-- Zero-order customer → NULLIF(0,0)=NULL → result NULL, no crash.
```

---

## 10. Performance Profile

Live-coding rarely tests raw throughput, but the *optimise* phase does, and interviewers escalate scale to see if your plan holds. Here is how the archetypes behave as data grows:

| Pattern | 1M rows | 10M rows | 100M rows | Dominant cost |
|---------|---------|----------|-----------|---------------|
| Top-N overall (`GROUP BY`+`ORDER BY`+`LIMIT`) | ~50 ms | ~400 ms | ~4 s | HashAggregate + full sort |
| Top-N-per-group (`ROW_NUMBER` window) | ~120 ms | ~1.2 s | ~14 s | WindowAgg sort of *all* rows |
| Top-N-per-group (`LATERAL` + index) | ~8 ms | ~15 ms | ~30 ms | one index probe per group |
| Existence (`EXISTS`/semi-join) | ~20 ms | ~90 ms | ~600 ms | index probe, stops at first match |
| Running total (`SUM OVER`) | ~80 ms | ~700 ms | ~8 s | sort + sequential window frame |
| Correlated subquery (naive) | ~200 ms | ~15 s | minutes | re-executes inner per outer row |

Key takeaways to voice in an interview:

- **The window-function top-N-per-group is O(N log N) because it sorts every row**, even though you only keep 3 per group. When N per group is small and you have an index on `(group_id, metric DESC)`, a `LATERAL` join is dramatically faster because it probes the index for exactly the top rows per group. This is the classic "your window query is correct but doesn't scale" follow-up.
- **`work_mem` governs whether HashAggregate and Sort spill.** At 100M rows the aggregate may exceed `work_mem` and spill to disk (`Batches > 1` in the plan) — 5-10x slower. Voice: "at this scale I'd check `work_mem` or pre-aggregate incrementally."
- **A correlated subquery that re-runs per outer row is the scaling cliff.** The planner sometimes rescues it via semi-join transform, but not always; rewriting to a JOIN or `EXISTS` is the reliable fix.
- **Index the sort/filter, not just the join.** The most common 100M-row win is a composite index matching `WHERE` + `ORDER BY` so the plan skips the sort entirely (`Index Scan` returns rows pre-sorted).

The senior move is to *tie the scale to the plan*: "at 1M this Seq Scan is fine; at 100M I need the composite index or this becomes a 4-second query, and if it's on a dashboard that's a timeout."

---

## 11. Node.js Integration

In a live-coding round you may be asked to wrap the query in application code. Keep it boring and correct: parameterised queries (never string interpolation), explicit column selection, and result handling that accounts for the empty case.

### 11.1 Parameterised query with the edge cases handled

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Top customers by trailing-window revenue — the naive-correct query from §6,
// hardened: COALESCE for zero-order customers, params for injection safety.
async function topCustomersByRevenue({ days = 90, limit = 10 }) {
  const { rows } = await pool.query(
    `SELECT
       u.id,
       u.name,
       COALESCE(SUM(o.amount), 0) AS revenue
     FROM users u
     LEFT JOIN orders o
       ON o.user_id = u.id
      AND o.status = 'completed'
      AND o.created_at >= NOW() - ($1 || ' days')::INTERVAL
     GROUP BY u.id, u.name
     ORDER BY revenue DESC
     LIMIT $2`,
    [days, limit]                      // $1, $2 — bound, never interpolated
  );
  return rows;                          // [] if no users — caller must handle empty
}
```

### 11.2 Handling ties explicitly at the boundary

```javascript
// "Everyone tied for the top revenue" — RANK, so the result length is variable.
async function topRevenueTier() {
  const { rows } = await pool.query(
    `SELECT name, revenue FROM (
       SELECT u.name,
              COALESCE(SUM(o.amount), 0) AS revenue,
              RANK() OVER (ORDER BY COALESCE(SUM(o.amount), 0) DESC) AS rk
       FROM users u
       LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'completed'
       GROUP BY u.id, u.name
     ) t
     WHERE rk = 1`
  );
  // rows.length may be > 1 on a tie — do NOT assume rows[0] is "the" winner.
  return rows;
}
```

### 11.3 Decomposed report, streaming for large result sets

```javascript
// A large export — use a cursor so the whole result set never lives in memory at once.
import Cursor from 'pg-cursor';

async function* streamActivationReport() {
  const client = await pool.connect();
  try {
    const cursor = client.query(new Cursor(
      `WITH cohorts AS (
         SELECT id AS user_id, DATE_TRUNC('month', created_at) AS cohort_month, created_at AS signup_at
         FROM users WHERE created_at >= '2026-01-01'
       )
       SELECT cohort_month, COUNT(*) AS size
       FROM cohorts GROUP BY cohort_month ORDER BY cohort_month`
    ));
    let batch;
    do {
      batch = await cursor.read(500);   // 500 rows at a time
      for (const row of batch) yield row;
    } while (batch.length > 0);
  } finally {
    client.release();                   // always release, even on throw
  }
}
```

**ORM note:** the clarify/edge-case reasoning is identical whether you write raw SQL or use an ORM — the difference is only *where* the bug hides. ORMs make fan-out and default-LEFT-JOIN bugs *easier* to introduce because the join is implicit. For any query with aggregation, ranking, or non-trivial NULL handling, raw parameterised SQL is the defensible interview answer.

---

## 12. ORM Comparison

Live-coding interviews are almost always raw SQL, but a common follow-up is "how would you do this in your ORM, and where does it break?" Know the honest answer for each.

### Prisma

**Can it?** Simple joins and `_count`/`_sum` aggregates via `groupBy`, yes. Window functions, `RANK`, `DENSE_RANK`, top-N-per-group — no, needs `$queryRaw`.

```typescript
// Prisma groupBy handles the naive per-customer total, but NOT ranking or ties.
const rows = await prisma.order.groupBy({
  by: ['userId'],
  where: { status: 'completed' },
  _sum: { amount: true },
  orderBy: { _sum: { amount: 'desc' } },
  take: 10,
});
// Ranking / ties / windows → escape hatch:
const ranked = await prisma.$queryRaw`
  SELECT u.name, SUM(o.amount) AS revenue,
         RANK() OVER (ORDER BY SUM(o.amount) DESC) AS rk
  FROM users u JOIN orders o ON o.user_id = u.id
  WHERE o.status = 'completed'
  GROUP BY u.id, u.name`;
```

**Where it breaks:** no window functions, no `HAVING` on computed aggregates beyond simple cases, no `NULLIF`/`COALESCE` in the typed API. **Verdict:** great for the naive-correct grouped query; `$queryRaw` for anything with ranking or NULL guards.

### Drizzle

**Can it?** Closest to SQL — window functions expressible via `sql` template, `.groupBy()`, `.having()` all first-class.

```typescript
import { sql, eq, desc } from 'drizzle-orm';
const rows = await db
  .select({
    name: users.name,
    revenue: sql<number>`COALESCE(SUM(${orders.amount}), 0)`,
    rk: sql<number>`RANK() OVER (ORDER BY COALESCE(SUM(${orders.amount}), 0) DESC)`,
  })
  .from(users)
  .leftJoin(orders, sql`${orders.userId} = ${users.id} AND ${orders.status} = 'completed'`)
  .groupBy(users.id, users.name);
```

**Where it breaks:** the window function itself must be a `sql` template (not fully typed), but it composes cleanly. **Verdict:** the best ORM for live-coding-style queries — you can express nearly everything without dropping to raw.

### Sequelize

**Can it?** Aggregates yes; window functions only via `sequelize.literal()`. Default `include` is LEFT JOIN — the exact §9 Wrong-4 trap.

```javascript
const rows = await Order.findAll({
  attributes: ['userId', [fn('COALESCE', fn('SUM', col('amount')), 0), 'revenue']],
  where: { status: 'completed' },
  group: ['userId'],
  order: [[literal('revenue'), 'DESC']],
  limit: 10,
});
```

**Where it breaks:** ranking/windows need `literal()` (raw string, no safety); the LEFT-JOIN-by-default `include` silently reintroduces the WHERE-nullifies-outer-join bug. **Verdict:** fine for grouped aggregates; use `sequelize.query()` for anything with a window function.

### TypeORM

**Can it?** QueryBuilder handles joins, `groupBy`, `having`; window functions via `.addSelect('RANK() OVER (...)', 'rk')` raw string.

```typescript
const rows = await ds.createQueryBuilder(User, 'u')
  .leftJoin('u.orders', 'o', "o.status = 'completed'")
  .select('u.name', 'name')
  .addSelect('COALESCE(SUM(o.amount), 0)', 'revenue')
  .groupBy('u.id').addGroupBy('u.name')
  .orderBy('revenue', 'DESC')
  .getRawMany();
```

**Where it breaks:** window functions and `NULLIF` are raw strings with no type safety; `getRawMany` vs `getMany` confusion loses aggregate columns. **Verdict:** workable with `getRawMany`; raw strings for windows.

### Knex

**Can it?** The most SQL-transparent — `.sum()`, `.groupBy()`, `.havingRaw()`, and window functions via `knex.raw()`.

```javascript
const rows = await knex('users as u')
  .leftJoin('orders as o', function () {
    this.on('o.user_id', 'u.id').andOnVal('o.status', 'completed');
  })
  .select('u.name')
  .select(knex.raw('COALESCE(SUM(o.amount), 0) AS revenue'))
  .groupBy('u.id', 'u.name')
  .orderBy('revenue', 'desc')
  .limit(10);
```

**Where it breaks:** window functions are `knex.raw()` strings — no builder sugar. **Verdict:** most transparent; the `.andOnVal()` correctly puts the status filter in the ON clause, avoiding the §9 Wrong-4 trap.

### Summary

| ORM | Grouped aggregate | Window / RANK | Zero-guard (COALESCE/NULLIF) | Live-coding verdict |
|-----|-------------------|---------------|------------------------------|---------------------|
| Prisma | `groupBy` (typed) | `$queryRaw` only | raw only | Naive query great, raw for ranking |
| Drizzle | `.groupBy()` | `sql` template | `sql` template | Best overall — nearly all in-builder |
| Sequelize | `fn/col` | `literal()` | `fn('COALESCE')` | Watch LEFT-JOIN default |
| TypeORM | QueryBuilder | raw addSelect | raw string | Use `getRawMany` |
| Knex | `.sum().groupBy()` | `knex.raw()` | `knex.raw()` | Most transparent |

The interview-honest statement: *"For the aggregation I'd use the ORM's grouped API, but the moment ranking, window functions, or NULL guards appear I drop to raw parameterised SQL, because expressing them in the ORM is either impossible or less readable than the SQL itself."*

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given `employees(id, name, salary, department_id)`, write a query returning the **highest-paid employee in each department**. Then answer: what happens if two employees tie for the highest salary in a department — does your query return one or both? Rewrite it to return both.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topics 11, 12, 20)

Given `users(id, name, created_at)` and `orders(id, user_id, amount, status, created_at)`, write **one** query returning, for every user (including those who have never ordered):
- `total_orders` (completed only)
- `total_revenue` (completed only, must show `0` not `NULL` for zero-order users)
- `avg_order_value` (must not crash for zero-order users)
- `first_order_at` and `last_order_at`

Walk your five-point edge-case checklist as you write it, and note next to each clause which check it satisfies.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

Given `orders(id, user_id, amount, status)` and `order_items(id, order_id, product_id, quantity, unit_price)`, a teammate wrote this to get revenue and item-count per user:

```sql
SELECT u.name, SUM(o.amount) AS revenue, COUNT(oi.id) AS items
FROM users u
JOIN orders o      ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY u.name;
```

1. Identify why `revenue` is wrong.
2. Rewrite it so both `revenue` (per-order amount, counted once) and `items` (total line items) are correct in a single query.
3. State the index you would add if `order_items` has 50M rows.

```sql
-- Write your corrected query here
```

### Exercise 4 — Interview Simulation

You are handed only this: *"We think some customers are being double-charged. Find them."* No schema, no definition of "double-charged." Write out (a) the five clarifying questions you would ask, (b) three sample rows you would ask to see, and (c) your naive-correct query given the reasonable assumption that a double-charge is two `completed` payments for the same `order_id` within 5 minutes. Tables available: `payments(id, order_id, amount, status, created_at)`.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — "Walk me through how you'd approach a SQL problem you've never seen before."

**Junior answer:** "I'd read the question, look at the tables, and start writing the query, then fix it if the output looks wrong."

**Principal answer:** "Five phases, in order. First I *clarify* — I restate the problem and pin down every vague word, especially the output grain: one row per what? Then I ask about ties, NULLs, and whether empty/zero cases should appear. Second, I look at *sample data* — three to five rows per table — to confirm keys, nullability, and trace one expected output row by hand as my correctness oracle. Third, I write the *naive-correct* query: the simplest thing I'm certain is right, readable over clever, and I state the expected row count out loud. Fourth, I run my *edge-case* checklist — NULLs, ties, empty input, fan-out, division by zero — and fix each. Only fifth do I *optimise*: I read the plan, name the offending node, propose an index or rewrite, and re-verify the answer didn't change. Correctness always precedes speed."

**Interviewer follow-up:** "Which of those five phases do candidates skip most, and what does it cost them?" *(Expected: clarify — skipping it means solving the wrong problem correctly, the most expensive failure because the code runs and looks right.)*

### Q2 — "You've written a correct query. The interviewer says 'now imagine orders has 100 million rows.' What do you do?"

**Junior answer:** "Add an index on the columns in the query."

**Principal answer:** "I run `EXPLAIN ANALYZE` and read it bottom-up. I'm looking for the dominant cost: a Seq Scan with a high `Rows Removed by Filter` means the filter has no supporting index — I'd add a composite index matching the `WHERE` plus, if possible, the `ORDER BY` so the sort disappears too. If there's a `ROW_NUMBER` window doing a full sort just to keep 3 rows per group, I'd rewrite to a `LATERAL` join that probes an index per group — O(groups × log N) instead of O(N log N). I'd check `Batches > 1` on hashes and sorts, which signals a `work_mem` spill to disk. And critically, after any rewrite I re-run against my sample data to confirm the answer is unchanged — an optimisation that changes results is a bug."

**Interviewer follow-up:** "Your composite index is `(status, created_at)`. Why that column order and not `(created_at, status)`?" *(Expected: equality columns before range columns — `status = 'completed'` is equality, `created_at >=` is a range; a B-tree can only range-scan efficiently after all equality predicates are satisfied, so the equality column must come first.)*

### Q3 — "Give me the second-highest salary. Now — what did you assume, and how would each assumption change your query?"

**Junior answer:** "`SELECT MAX(salary) FROM employees WHERE salary < (SELECT MAX(salary) FROM employees)` — done."

**Principal answer:** "That's my naive answer, and it assumes 'second-highest' means the second *distinct* value. But I made three assumptions I should surface. One: ties — if two people share the top salary, do we want the value just below the max (my query, distinct values) or literally the 2nd row when sorted (which would still be the max)? Two: empty or single-value tables — my query correctly returns NULL rather than erroring, which is usually desired. Three: 'the employee' vs 'the salary' — if they want the *person*, ties mean multiple people, and I'd switch to `DENSE_RANK() = 2` to return everyone at that salary. So the one-liner is fine, but which of these they mean changes it from a scalar subquery to a `DENSE_RANK` window."

**Interviewer follow-up:** "Write the version that returns *all* employees earning the second-highest distinct salary." *(Expected: `DENSE_RANK() OVER (ORDER BY salary DESC)` in a subquery, filter `= 2` — DENSE_RANK because it assigns the same rank to ties and no gaps.)*

### Q4 — "Your query has a LEFT JOIN but the rows you wanted to keep disappeared. Why?"

**Junior answer:** "Maybe the join condition is wrong, let me check the ON clause."

**Principal answer:** "Almost always it's a predicate on the right-hand table sitting in `WHERE` instead of `ON`. Logically the `LEFT JOIN` pads unmatched left rows with NULLs, but that padding happens in the FROM/JOIN phase; `WHERE` runs *after* and evaluates `right_table.status = 'completed'` as `UNKNOWN` for those NULL-padded rows, which fails the filter and drops them — silently turning the LEFT JOIN into an INNER JOIN. The fix is to move any filter on the right-hand table into the `ON` clause so it applies before padding. A filter on the *left* table can stay in `WHERE`; only right-table predicates need to move."

**Interviewer follow-up:** "Is there ever a case where you *do* want the right-table filter in `WHERE` on a LEFT JOIN?" *(Expected: yes — the anti-join pattern `WHERE right.id IS NULL` to find left rows with no match, which deliberately exploits the NULL-padding to find non-matches.)*

---

## 15. Mental Model Checkpoint

1. You are asked to "find the top 5 customers." Before writing a single line, what are the four clarifying questions whose answers most change the query — and for each, name the specific SQL construct the answer selects?

2. A `LEFT JOIN` between `users` and `orders`, then `SUM(orders.amount)`, returns `NULL` for a customer with no orders. Walk the logical execution order that produces the `NULL`, and give the one-function fix.

3. You have a `ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC)` filtered to `<= 3`. The interviewer says the table is now 100M rows. Why does this query's cost grow faster than the number of rows you keep, and what rewrite fixes it?

4. Explain why moving a predicate from `ON` to `WHERE` is safe for an INNER JOIN but changes the result for a LEFT JOIN. Root it in the logical execution order.

5. Given a payments table, you write `SUM(amount) / COUNT(id)` and it crashes on production data but not on your test data. What is different about production, and what is the two-character-plus-one-word fix?

6. A candidate says "I'll optimise as I go." Why is this the wrong order, and construct a concrete scenario where optimise-first produces a query that runs fast and returns the wrong answer.

7. You claim a query "returns one row per customer," but `COUNT(*)` of the result is larger than `COUNT(DISTINCT customer_id)`. What single phenomenon explains this, which join introduced it, and how do you confirm it in one query?

---

## 16. Quick Reference Card

```sql
-- ═══ THE FIVE-PHASE FRAMEWORK ═══
-- 1. CLARIFY   → grain? ties? NULLs? empty? scale? define every vague word
-- 2. SAMPLE    → 3-5 rows/table; trace ONE output row by hand = your oracle
-- 3. NAIVE     → simplest CERTAINLY-correct query; readable > clever
-- 4. EDGE      → NULLs · ties · empty · fan-out · divide-by-zero
-- 5. OPTIMISE  → read plan, name index/rewrite, re-verify answer unchanged

-- ═══ EDGE-CASE CHECKLIST (memorise) ═══
COALESCE(SUM(x), 0)                  -- SUM over LEFT JOIN empty group → NULL, not 0
COUNT(o.id)  vs  COUNT(*)            -- COUNT(col) skips NULL-padded rows; COUNT(*) doesn't
/ NULLIF(denominator, 0)             -- divide-by-zero guard → NULL not error
right_tbl.pred IN ON, not WHERE      -- keep LEFT JOIN from collapsing to INNER
COUNT(DISTINCT o.id)                 -- undo fan-out from one-to-many join

-- ═══ TIE RULES ═══
ROW_NUMBER()  -- 1,2,3,4  → exactly N rows, breaks ties arbitrarily
RANK()        -- 1,2,2,4  → keeps ties, LEAVES gaps
DENSE_RANK()  -- 1,2,2,3  → keeps ties, NO gaps  (use for "Nth distinct value")

-- ═══ ARCHETYPE → SKELETON ═══
top-N overall        → GROUP BY … ORDER BY … LIMIT
top-N per group      → ROW_NUMBER/RANK OVER (PARTITION BY g ORDER BY m DESC) then filter
                       …or LATERAL + index when N small & table huge
Nth highest distinct → DENSE_RANK = N   |   OFFSET N-1 LIMIT 1
running total        → SUM(x) OVER (ORDER BY t)
exists / never       → [NOT] EXISTS (SELECT 1 …)  |  LEFT JOIN … WHERE r.id IS NULL
latest per group     → DISTINCT ON (g) … ORDER BY g, t DESC   |   ROW_NUMBER = 1
pivot                → SUM(x) FILTER (WHERE cond)   |   SUM(CASE WHEN … END)
median / pct         → PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY x)

-- ═══ PLAN-READING ONE-LINERS ═══
-- Seq Scan + high "Rows Removed by Filter"  → missing index on filter cols
-- Nested Loop + Seq Scan inner              → missing index on inner join col
-- Batches > 1 on Hash/Sort                  → work_mem spill, 5-10x slower
-- estimated rows ≫ actual rows              → stale stats, run ANALYZE
-- equality cols BEFORE range cols in index  → (status, created_at) not reverse

-- ═══ INTERVIEW ONE-LINERS ═══
-- "What is the grain of the output — one row per what?"  ← ask this FIRST, always
-- "Correct first, fast second."
-- "Let me trace one row through my sample data to confirm."
-- "That's my naive answer; now let me walk the edge cases."
-- "At 100M rows this Seq Scan becomes the bottleneck — here's the index."
```

---

## Connected Topics

- **Topic 03 — Logical Query Processing Order**: the FROM→…→LIMIT skeleton every framework step and every "why can't I use this alias here" answer hangs on.
- **Topic 07 — NULL in Depth**: three-valued logic behind the NULL edge cases — dropped join keys, skipped aggregates, UNKNOWN filters.
- **Topic 11 — INNER JOIN in Depth**: fan-out and NULL-key exclusion, the two most common live-coding correctness bugs.
- **Topic 12 — LEFT JOIN and RIGHT JOIN**: the ON-vs-WHERE trap that silently collapses an outer join to inner.
- **Topic 17 — JOIN Performance Deep Dive**: Nested Loop / Hash / Merge selection you narrate in the optimise phase.
- **Topic 20 — GROUP BY Fundamentals**: collapsing multiplied rows, `COUNT(DISTINCT)`, HAVING vs WHERE.
- **Topic 24 — CTEs**: decomposition, inlining behaviour, the `MATERIALIZED` optimisation fence.
- **Topic 27 — EXISTS and NOT EXISTS**: the semi-join existence pattern and its scaling advantage over IN and correlated subqueries.
- **Phase 8 — Window Functions**: ROW_NUMBER / RANK / DENSE_RANK, the tie-handling core of most ranking problems.
- **Topic 74 — Top 20 Interview Questions**: the question bank this framework equips you to attack.
- **Topic 76 — System Design SQL Questions**: the next scale up — from writing one query live to designing the schema and access patterns behind it.
