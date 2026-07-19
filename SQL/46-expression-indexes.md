# Topic 46 — Expression Indexes
### SQL Mastery Curriculum — Phase 7: Indexes and Query Optimisation

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a library where every book is filed on the shelf by its title, spelled **exactly** as printed on the cover — capital letters, punctuation, everything. There is a card catalogue (the index) sorted the same way: `"A Tale of Two Cities"`, `"THE HOBBIT"`, `"the great gatsby"`. The catalogue lets a librarian jump straight to any title in seconds instead of walking every shelf.

Now a patron walks up and says: "I want the book whose title, **ignoring capitalisation**, is `the hobbit`." The librarian looks at the catalogue and freezes. The catalogue is sorted by the exact printed title. `"THE HOBBIT"` is filed under `T-H-E` in capitals; the patron's lowercase `the hobbit` doesn't line up with how the cards are ordered, because uppercase and lowercase sort differently. The catalogue is now **useless for this question**. The librarian has only one option left: walk every single shelf, lowercase each title in their head, and compare. A full scan of the entire library.

The fix is not to re-file every book. The fix is to build a **second catalogue** where each card is filed by the *lowercased* title: `"a tale of two cities"`, `"the great gatsby"`, `"the hobbit"`. Now when the patron asks for `the hobbit` in lowercase, the librarian goes straight to it. Every time a new book arrives, the librarian computes its lowercased title once and files a card in this second catalogue.

That second catalogue is an **expression index**. Instead of indexing the column's raw value, you index the *result of a function applied to* the column — `lower(title)` — and then any query that asks the same question, `WHERE lower(title) = 'the hobbit'`, can walk straight to the answer. The database pre-computes `lower(...)` for every row at write time and stores the sorted results, so at read time it never has to lower a single row on the fly.

The whole topic is that one idea: an index does not have to be on a raw column. It can be on **any expression** — as long as the expression always produces the same output for the same input, forever.

---

## 2. Connection to SQL Internals

A B-tree index in PostgreSQL is a sorted, balanced tree of **keys**. Normally the key is a column value: the index on `users(email)` stores each row's `email` value together with a pointer (a `ctid` — the physical heap tuple location) to the row. Lookups work because the tree is *ordered by that key*, so the planner can binary-search down the tree to find a range or a point.

The critical internal fact: **the planner can only use an index if the query's search key is comparable, using the tree's ordering, to the keys stored in the tree.** The index on `email` stores raw email strings sorted by the default `text` collation. A predicate `WHERE email = 'a@b.com'` produces a search key that is directly comparable to those stored keys — the planner descends the tree. But a predicate `WHERE lower(email) = 'a@b.com'` produces a search key on a *derived value* (`lower(email)`) that is **not** what the tree is sorted by. There is no ordering relationship the planner can exploit between "emails sorted by their raw bytes" and "the lowercase form I'm searching for." So the index is inapplicable, and the planner falls back to a **Sequential Scan**, computing `lower(email)` for every heap tuple it reads.

An **expression index** (also called a *functional index*) changes what the tree is sorted by. `CREATE INDEX ON users (lower(email))` builds a B-tree whose keys are the *outputs* of `lower(email)`, one per row, sorted by those outputs. Internally PostgreSQL:

1. At **build time**, scans the heap, evaluates `lower(email)` for every row, and inserts `(lower(email_value), ctid)` pairs into the tree in sorted order.
2. At **write time** (every INSERT/UPDATE that touches `email`), re-evaluates `lower(email)` for the new/changed row and maintains the tree entry. This is why the expression must be **IMMUTABLE** — the engine assumes the stored key never silently drifts from what the expression would produce today.
3. At **read time**, when a query contains an expression that **matches the indexed expression exactly**, the planner recognises it, computes the search key once from the query's constant side, and does an Index Scan.

This ties into several internals you already know:

- **Heap and MVCC**: the index stores `ctid` pointers into heap pages, exactly like a normal index. Dead tuples from MVCC still leave index entries that `VACUUM` must clean; expression indexes are no different — they are ordinary indexes that happen to key on a computed value.
- **The planner's expression matching**: PostgreSQL stores the index's key expression in its catalog (`pg_index.indexprs`). During planning, it compares the parsed query predicates against that stored expression tree. The match is **structural and exact** — `lower(email)` in the query must be the same expression node as `lower(email)` in the index definition. `lower(email || '')` will not match.
- **`work_mem` and sorts**: an expression index can also satisfy `ORDER BY lower(name)` without a Sort node, because the tree is already physically ordered by that expression — the same way a plain index provides pre-sorted output (relevant to Merge Joins, Topic 11, and Index-Only Scans, Topic 47).

So the mental model is: **an expression index is a completely ordinary B-tree — it just stores a function's output as the key instead of a raw column.** Everything else (heap pointers, VACUUM, MVCC visibility, page structure) is identical.

---

## 3. Logical Execution Order Context

```
FROM users
WHERE lower(email) = $1        ← the predicate that an expression index accelerates
GROUP BY ...
HAVING ...
SELECT ...
ORDER BY lower(name)           ← an expression index can also serve ORDER BY
LIMIT ...
```

Indexes are a **physical access-path** decision, not a logical clause — they do not appear in the logical order of operations at all. But *where* the indexed expression is referenced determines whether the index can help:

- **In `WHERE`**: the most common use. `WHERE lower(email) = $1` is evaluated logically after `FROM`, but physically the planner tries to satisfy it via an Index Scan on `users(lower(email))` *instead of* scanning the whole table and filtering. The index turns a post-FROM filter into a direct row-locating access path.
- **In `ORDER BY`**: `ORDER BY lower(name)` runs late in the logical order (after SELECT-list projection), but if an index on `lower(name)` exists, the planner can read rows in index order and **skip the Sort node entirely**.
- **In `JOIN ... ON`**: `ON lower(a.email) = lower(b.email)` — an expression index on each side lets the planner use a Merge Join or an indexed Nested Loop on the computed keys.
- **In `GROUP BY` / `DISTINCT`**: an index on `lower(name)` can feed a sorted grouping without an explicit sort.

The key principle that spans all of these: **the expression in the query must be evaluated the same way the index computed it, and the planner must be able to prove that.** The logical clause it appears in only decides *which* physical optimisation (Index Scan vs skip-Sort vs Merge Join) becomes available.

This connects directly to Topic 45 (Partial Indexes), which changed *which rows* an index covers; here we change *what value* the index keys on. The two are orthogonal and — as we will see in Section 6 — combinable.

---

## 4. What Is an Expression Index?

An **expression index** (functional index) is a B-tree (or other index type) whose key is the result of a **function or expression applied to one or more columns**, rather than the raw column value itself. It lets queries that filter or sort on that *same expression* use an index instead of computing the expression row-by-row in a sequential scan.

```sql
CREATE INDEX idx_users_lower_email
    ON users (lower(email));
--  │        │  │     │
--  │        │  │     └── the column the expression is applied to
--  │        │  └──────── the IMMUTABLE function forming the key
--  │        └─────────── the parenthesised EXPRESSION being indexed
--  │                     (parentheses are REQUIRED around any expression)
--  └──────────────────── ordinary CREATE INDEX — default access method is btree
```

### Annotated Full Syntax

```sql
CREATE [UNIQUE] INDEX [CONCURRENTLY] [IF NOT EXISTS] index_name
    ON table_name [USING method]
    ( expression [COLLATE collation] [opclass] [ASC|DESC] [NULLS {FIRST|LAST}] )
    [INCLUDE (column_list)]
    [WHERE predicate];
--  │        │            │           │          │        │           │        │
--  │        │            │           │          │        │           │        └── optional: PARTIAL expression index (Topic 45 combined)
--  │        │            │           │          │        │           └────────── optional covering columns (Index-Only Scan, Topic 47)
--  │        │            │           │          │        └──────────────────────── NULL ordering within the expression's sort
--  │        │            │           │          └───────────────────────────────── sort direction of the expression key
--  │        │            │           └──────────────────────────────────────────── operator class (e.g. text_pattern_ops for LIKE)
--  │        │            └──────────────────────────────────────────────────────── optional COLLATE for text expressions
--  │        └───────────────────────────────────────────────────────────────────── expression MUST be wrapped in parentheses
--  └────────────────────────────────────────────────────────────────────────────── UNIQUE enforces uniqueness on the EXPRESSION's value
```

### Exact Semantics

Given a table:

```
users:
 id | email
----+---------------------
 1  | Alice@Example.com
 2  | bob@example.com
 3  | CAROL@EXAMPLE.COM
```

A **plain** index on `email` stores keys sorted by raw bytes: `Alice@Example.com`, `CAROL@EXAMPLE.COM`, `bob@example.com` (uppercase letters sort before lowercase in the C/default byte ordering). A query `WHERE lower(email) = 'bob@example.com'` cannot use it.

An **expression** index `CREATE INDEX ON users (lower(email))` stores:

```
lower(email)         → ctid
---------------------+-------
alice@example.com    → (0,1)
bob@example.com      → (0,2)
carol@example.com    → (0,3)
```

Now `WHERE lower(email) = 'bob@example.com'` binary-searches this tree to `(0,2)` in `O(log n)` and fetches exactly one heap row. The rule is absolute:

- **The query expression must match the index expression exactly** (same function, same argument, same nesting). `lower(email)` matches `lower(email)`; it does **not** match `upper(email)`, `lower(email || '')`, or `email` alone.
- **The function must be IMMUTABLE** — same inputs always yield the same output, with no dependence on locale-at-query-time, `now()`, session settings, or table data. `lower()` is immutable; `now()` and `random()` are not.
- **The index keys on the computed value**, so `UNIQUE` on `lower(email)` enforces case-insensitive uniqueness — `Alice@Example.com` and `alice@example.com` would collide.

---

## 5. Why Expression Index Mastery Matters in Production

1. **Case-insensitive lookups are everywhere.** Email logins, usernames, product SKUs, coupon codes — almost every "find by identifier" flow needs to be case-insensitive. The naive `WHERE lower(email) = $1` is **non-sargable** against a plain `email` index and silently degrades to a full table scan. On a 10M-row `users` table that is the difference between a 0.2 ms point lookup and a 3-second sequential scan **on your hottest code path** (login). Expression indexes are the fix, and knowing this separates engineers who ship fast auth from those who page the on-call at 2 a.m.

2. **The bug is invisible until scale.** With 10,000 rows a seq scan is ~2 ms and nobody notices. The `WHERE lower(email) = $1` query looks correct, returns correct results, and passes every test. At 5M rows it becomes the top query in `pg_stat_statements` by total time. Understanding *why* — non-sargability — lets you predict and prevent this before it becomes an incident.

3. **It unlocks correctness patterns you cannot express otherwise.** A **unique** expression index is the only clean way to enforce "emails are unique case-insensitively" or "one active subscription per user" (combined with partial indexes). Without it you resort to application-level checks that race under concurrency.

4. **Write-cost awareness.** Every expression index adds a per-write cost: the function is evaluated on every INSERT and on every UPDATE that touches the underlying column. A poorly chosen expression index on a write-heavy table (e.g. an index on `lower(description)` for a 500-char text column, updated on every request) can quietly double your write latency. Mastery means knowing when the read win is worth the write tax.

5. **The IMMUTABLE trap causes real outages.** Developers try to index `WHERE date_trunc('day', created_at AT TIME ZONE '...')` or a custom function they forgot to mark IMMUTABLE, and either the `CREATE INDEX` fails outright or — worse — they lie by marking a non-immutable function IMMUTABLE, and the index returns **wrong results** (rows the query should find but doesn't) after a timezone or locale change. Understanding *why* immutability is mandatory prevents silent data-loss-shaped bugs.

---

## 6. Deep Technical Content

### 6.1 What "Non-Sargable" Actually Means

**SARGable** = "Search ARGument able" — a predicate the engine can satisfy by seeking into an index rather than scanning and testing every row. A predicate is sargable against an index when its **column appears bare on one side of a comparison** with a constant/parameter on the other:

```sql
WHERE email = $1                 -- SARGABLE against index on (email)
WHERE created_at >= $1           -- SARGABLE against index on (created_at)
WHERE id BETWEEN $1 AND $2       -- SARGABLE against index on (id)
```

The moment you **wrap the indexed column in a function or expression**, the predicate becomes **non-sargable** against the plain index, because the index stores the raw column value, not the function's output:

```sql
WHERE lower(email) = $1          -- NON-SARGABLE against index on (email)
WHERE email || '' = $1           -- NON-SARGABLE
WHERE created_at::date = $1      -- NON-SARGABLE against index on (created_at)
WHERE substr(sku, 1, 3) = $1     -- NON-SARGABLE
WHERE amount * 1.1 > $1          -- NON-SARGABLE against index on (amount)
```

The planner cannot "see through" the function. It has no general algebra to invert `lower()` and figure out which raw-email keys could possibly produce `'bob@example.com'`. So it gives up on the index and scans.

There are exactly **two** ways to make such a predicate fast:

1. **Rewrite the query** so the column is bare (only possible for *invertible, range-preserving* transforms — e.g. `created_at::date = $1` becomes `created_at >= $1 AND created_at < $1 + 1`).
2. **Build an expression index** on the exact expression — the general solution, and the only one available for `lower()`, hashing, JSON extraction, etc.

### 6.2 The Canonical Case: `lower(email)`

The single most common expression index in production:

```sql
-- The problem query (login by email, case-insensitive):
SELECT id, password_hash FROM users WHERE lower(email) = lower($1);

-- Without an expression index → Seq Scan, lower() computed on every row.

-- The fix:
CREATE INDEX idx_users_lower_email ON users (lower(email));

-- Now the SAME query does an Index Scan.
```

Two subtleties that trip people up:

- **Both sides should be lowered.** `WHERE lower(email) = lower($1)` (or lower the parameter in application code and pass `WHERE lower(email) = $1`). If you write `WHERE lower(email) = $1` and pass a mixed-case `$1`, the query is still *sargable* (it uses the index) but returns **no rows** for `'Bob@Example.com'` because the index stores lowercase keys and you're searching for a mixed-case key. The index works; your logic is wrong.
- **`lower($1)` is a constant-folding win.** `lower()` on a parameter is computed once at execution, not per row — it's on the constant side, so it costs nothing at scale.

### 6.3 The `citext` Alternative (and Why Expression Indexes Often Win)

PostgreSQL ships a `citext` (case-insensitive text) type. `email citext` makes `email = $1` case-insensitive automatically, and a plain index on `email` then serves case-insensitive lookups. It is a legitimate choice, but:

- `citext` bakes case-insensitivity into the **column type** — every comparison is case-insensitive, whether you want it or not.
- Expression indexes are **additive and opt-in** — the raw `email` remains case-sensitive text, and you add case-insensitive lookup only where you need it. You can have *both* a plain index (for exact-match audit) and a `lower(email)` index (for login).
- `citext` has some collation/`LIKE` edge cases and a slight per-comparison cost.

Rule of thumb: for a brand-new schema where the column is *always* case-insensitive, `citext` is clean. For existing tables, or where you want case-insensitivity only on specific query paths, the expression index is the more surgical tool.

### 6.4 The IMMUTABLE Requirement — In Full

The indexed expression **must be IMMUTABLE**: PostgreSQL classifies every function into one of three volatility categories:

| Category | Guarantee | Examples |
|----------|-----------|----------|
| `IMMUTABLE` | Same inputs → same output, always, forever. No DB/config/time dependence. | `lower()`, `upper()`, `md5()`, `abs()`, `length()`, `+`, arithmetic |
| `STABLE` | Same output within a single statement, but may change across statements. | `now()`, `current_date`, `timezone`-dependent casts, `current_setting()` |
| `VOLATILE` | May return different output on every call. | `random()`, `nextval()`, `clock_timestamp()` |

Only **IMMUTABLE** expressions are allowed in an index, because the index **physically stores the computed value** at write time and never re-derives it on read. If the expression could produce a different value later (STABLE/VOLATILE), the stored key would silently diverge from what the query computes now, and the Index Scan would return **wrong rows** — it would look up a key that no longer matches. PostgreSQL refuses to build such an index:

```sql
CREATE INDEX ON orders (age(now(), created_at));
-- ERROR:  functions in index expression must be marked IMMUTABLE
```

**The dangerous trap** is a timezone-dependent cast that *looks* immutable:

```sql
-- created_at is timestamptz. This cast depends on the SESSION timezone,
-- so it is only STABLE, not IMMUTABLE:
CREATE INDEX ON events (created_at::date);
-- ERROR:  functions in index expression must be marked IMMUTABLE
--   (because timestamptz → date depends on TimeZone setting)

-- A plain timestamp (no tz) → date IS immutable and works:
CREATE INDEX ON events ((created_at_local::date));   -- created_at_local is 'timestamp'

-- Or pin the timezone explicitly to make it immutable:
CREATE INDEX ON events ((timezone('UTC', created_at)::date));
```

**Do not "fix" this by marking your own function IMMUTABLE when it isn't.** If you write a wrapper function that internally reads the session timezone and mark it IMMUTABLE to satisfy the index builder, PostgreSQL will believe you, build the index, and then serve **silently wrong results** whenever the effective timezone differs from build time. This is a genuine correctness bug that survives every unit test and only manifests for users in other timezones. The rule: mark a function IMMUTABLE only if it *is* truly immutable.

### 6.5 Matching the Expression Exactly

The planner matches the query's expression against the stored index expression **structurally and exactly**. Semantically-equivalent-but-differently-written expressions do **not** match:

```sql
CREATE INDEX idx ON users (lower(email));

WHERE lower(email) = $1              -- MATCH → Index Scan
WHERE lower(email) IN ($1, $2)       -- MATCH → Index Scan (expanded to = ANY)
WHERE lower(email) LIKE $1           -- partial match (needs text_pattern_ops opclass for anchored LIKE)
WHERE lower(email) = lower($1)       -- MATCH (right side is a constant expr, folded)

WHERE upper(email) = $1              -- NO MATCH (different function)
WHERE lower(email || '') = $1        -- NO MATCH (different expression tree)
WHERE lower(trim(email)) = $1        -- NO MATCH (extra function)
WHERE email = $1 OR lower(email) = $2 -- only the lower(email) branch can use idx
```

This is why building the index and writing the query must be coordinated. A common failure: the index is `lower(email)` but the ORM emits `LOWER(email)` — case of the *function name* doesn't matter (SQL is case-insensitive for identifiers), but `lower(btrim(email))` vs `lower(email)` absolutely does.

Multi-column and mixed expression indexes follow the same rule per column:

```sql
CREATE INDEX ON users (lower(email), created_at);
-- Serves:  WHERE lower(email) = $1 AND created_at > $2
-- Serves:  WHERE lower(email) = $1  (leftmost prefix)
-- Does NOT serve:  WHERE created_at > $2 alone (not a leading column)
```

### 6.6 UNIQUE Expression Indexes — Case-Insensitive Constraints

A UNIQUE expression index enforces uniqueness on the **computed value**, which is the standard way to build a case-insensitive unique constraint:

```sql
CREATE UNIQUE INDEX idx_users_email_ci ON users (lower(email));

-- Now these collide:
INSERT INTO users (email) VALUES ('Alice@Example.com');  -- OK
INSERT INTO users (email) VALUES ('alice@example.com');  -- ERROR: duplicate key
--   because lower(both) = 'alice@example.com'
```

Note you **cannot** write this as a table `UNIQUE (lower(email))` constraint — SQL constraints only accept bare columns. Expression uniqueness *must* be a unique index. This is a frequent interview point: "how do you enforce case-insensitive unique emails?" → unique expression index (or `citext` + normal unique).

Combine with a **partial** predicate (Topic 45) for conditional uniqueness:

```sql
-- One active subscription per user, comparing case-insensitively on plan_code:
CREATE UNIQUE INDEX ON subscriptions (user_id, lower(plan_code))
    WHERE status = 'active';
```

### 6.7 Beyond `lower()` — The Full Range of Useful Expression Indexes

Expression indexes are not just for case folding. Common production patterns:

```sql
-- 1. JSONB field extraction — index a value buried in a JSON document
CREATE INDEX ON events ((payload->>'user_id'));
-- Serves: WHERE payload->>'user_id' = $1

-- 2. Computed/derived scalar — full name search
CREATE INDEX ON users (lower(first_name || ' ' || last_name));

-- 3. Date bucketing (immutable form) — group by day
CREATE INDEX ON logs (date_trunc('day', created_at_utc));  -- created_at_utc is timestamp (not tz)

-- 4. Hash of a large value — index a fingerprint instead of the whole blob
CREATE INDEX ON documents (md5(body));
-- Serves dedup checks: WHERE md5(body) = md5($1)

-- 5. Numeric transform — index a rounded/absolute value
CREATE INDEX ON transactions (abs(amount));

-- 6. Prefix / substring — index a SKU prefix
CREATE INDEX ON products (substr(sku, 1, 4));

-- 7. Coalesced value — treat NULL as a sentinel for grouping/uniqueness
CREATE INDEX ON tasks (coalesce(assignee_id, 0));

-- 8. Expression index for anchored LIKE with a non-default collation:
CREATE INDEX ON users (lower(email) text_pattern_ops);
-- Serves: WHERE lower(email) LIKE 'bob%'
```

The `text_pattern_ops` operator class deserves note: for `LIKE 'prefix%'` to use an index in a non-C locale, you need the `text_pattern_ops` (or `varchar_pattern_ops`) opclass, because the default collation's ordering doesn't align with byte-prefix matching. This applies to expression indexes exactly as to plain ones.

### 6.8 Maintenance and Write Cost

An expression index is a real index with an extra per-write cost: **the expression is evaluated on every INSERT and on every UPDATE that modifies a column the expression depends on.**

- **INSERT**: `lower(email)` is computed once per inserted row and the result inserted into the tree. Cheap for `lower()`; measurable for expensive expressions (e.g. `md5(large_text)`).
- **UPDATE**: only re-evaluated if the underlying column changes. An `UPDATE users SET last_login = now()` does **not** touch `email`, so PostgreSQL's **HOT (Heap-Only Tuple)** optimisation can skip updating the `lower(email)` index entirely — a genuine performance benefit of *not* indexing volatile columns. But `UPDATE users SET email = $1` must re-evaluate `lower(email)` and update the index, and it also disqualifies that update from HOT (any indexed column change does).
- **Build cost**: `CREATE INDEX` must evaluate the expression for every existing row. On a 100M-row table, an expensive expression makes the build slow and CPU-heavy. Use `CREATE INDEX CONCURRENTLY` to avoid locking writes (at the cost of a slower two-pass build).
- **Storage**: the tree stores the *computed* value. `md5(body)` stores a fixed 32-byte hash regardless of `body` size — sometimes *smaller* than indexing the raw column. `lower(email)` is roughly the same size as `email`.
- **Statistics**: PostgreSQL gathers statistics on the expression's output (in `pg_statistic` keyed to the index), so `ANALYZE` lets the planner estimate selectivity of `lower(email) = $1` accurately — something it cannot do for an *un-indexed* expression, where it falls back to a poor default estimate.

Every extra index also slows `VACUUM` (more index entries to clean) and consumes buffer-pool and disk. The discipline is identical to any index: add it because a real, measured query path needs it — not speculatively.

### 6.9 Interaction with Partial and Covering Indexes

Expression indexes compose with the other index modifiers:

```sql
-- Expression + PARTIAL (Topic 45): case-insensitive lookup, only active users
CREATE INDEX ON users (lower(email)) WHERE deleted_at IS NULL;

-- Expression + COVERING (Topic 47): serve an Index-Only Scan
CREATE INDEX ON users (lower(email)) INCLUDE (id, display_name);
-- WHERE lower(email) = $1  can return id, display_name without touching the heap

-- All three combined:
CREATE INDEX ON users (lower(email)) INCLUDE (id)
    WHERE deleted_at IS NULL;
```

The expression is the *key*; `INCLUDE` columns are stored as raw payload (never expressions); the `WHERE` restricts *which rows* get an entry. Each modifier answers a different question — *what value* (expression), *which rows* (partial), *what payload* (covering).

---

## 7. EXPLAIN — Expression Index in the Plan

### Before the index: Seq Scan with the function computed per row

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, display_name FROM users WHERE lower(email) = 'bob@example.com';
```

```
Seq Scan on users  (cost=0.00..24583.00 rows=1 width=36)
                   (actual time=812.4..3120.7 rows=1 loops=1)
  Filter: (lower(email) = 'bob@example.com'::text)
  Rows Removed by Filter: 999999
Buffers: shared hit=1088 read=13495
Planning Time: 0.12 ms
Execution Time: 3120.9 ms
```

**Reading it**:
- `Seq Scan on users` — every row is read; there is no usable index.
- `Filter: (lower(email) = ...)` — `lower()` is evaluated on **every one of the 1,000,000 rows**.
- `Rows Removed by Filter: 999999` — the scan touched a million rows to return one.
- `read=13495` — thousands of pages pulled from disk into the buffer pool.
- **3.1 seconds** for a single-row login lookup. On the hot path. This is the pathology expression indexes exist to cure.

### After the index: Index Scan

```sql
CREATE INDEX idx_users_lower_email ON users (lower(email));
ANALYZE users;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, display_name FROM users WHERE lower(email) = 'bob@example.com';
```

```
Index Scan using idx_users_lower_email on users
        (cost=0.42..8.44 rows=1 width=36)
        (actual time=0.038..0.041 rows=1 loops=1)
  Index Cond: (lower(email) = 'bob@example.com'::text)
Buffers: shared hit=4
Planning Time: 0.15 ms
Execution Time: 0.061 ms
```

**Reading it**:
- `Index Scan using idx_users_lower_email` — the planner **recognised** that `lower(email)` in the predicate matches the index's key expression.
- `Index Cond: (lower(email) = ...)` — note this is an **Index Cond**, not a Filter. The condition is used to *seek into the tree*, not to test rows after reading them. This is the signature of a working expression index. If you instead saw `lower(email)` under `Filter:` while `Index Cond` was empty, the index is not being used for the seek.
- `Buffers: shared hit=4` — a handful of pages: a few index pages plus one heap page. Versus 14,583 before.
- **0.061 ms** vs 3,120 ms — roughly a **50,000× improvement**.

### Diagnostic: the non-matching expression still scans

```sql
-- Index is on lower(email), but the query wraps it differently:
EXPLAIN (ANALYZE)
SELECT id FROM users WHERE lower(trim(email)) = 'bob@example.com';
```

```
Seq Scan on users  (cost=0.00..27083.00 rows=1 width=8)
                   (actual time=845.1..3301.2 rows=1 loops=1)
  Filter: (lower(btrim(email)) = 'bob@example.com'::text)
  Rows Removed by Filter: 999999
Execution Time: 3301.4 ms
```

**Reading it**: `lower(btrim(email))` does **not** structurally match the index's `lower(email)`, so the planner falls back to a Seq Scan. The lesson from Section 6.5 made visible: *exact* expression match is mandatory.

### Expression index serving ORDER BY (skip the Sort)

```sql
CREATE INDEX idx_users_lower_name ON users (lower(display_name));

EXPLAIN (ANALYZE)
SELECT id, display_name FROM users ORDER BY lower(display_name) LIMIT 20;
```

```
Limit  (cost=0.42..1.83 rows=20 width=36) (actual time=0.03..0.09 rows=20 loops=1)
  ->  Index Scan using idx_users_lower_name on users
            (cost=0.42..70472.42 rows=1000000 width=36)
            (actual time=0.03..0.08 rows=20 loops=1)
Planning Time: 0.11 ms
Execution Time: 0.11 ms
```

**Reading it**: there is **no Sort node**. Because the index is physically ordered by `lower(display_name)`, the planner reads the first 20 entries in order and stops. Without the index this would be a `Sort` over a million rows (or an external merge sort spilling to disk).

---

## 8. Query Examples

### Example 1 — Basic: Case-Insensitive Email Lookup

```sql
-- The index (created once):
CREATE INDEX idx_users_lower_email ON users (lower(email));

-- The query — case-insensitive login lookup.
-- lower($1) folds the parameter once; lower(email) matches the index key.
SELECT
    id,
    email,          -- the ORIGINAL, case-preserved email is still available
    password_hash,
    display_name
FROM users
WHERE lower(email) = lower($1);   -- $1 = 'Bob@Example.COM' → matches 'bob@example.com'
```

### Example 2 — Intermediate: JSONB Field + Date Bucket

```sql
-- audit_logs stores a JSONB `details` column and a timestamp.
-- We frequently filter by the actor stored inside the JSON and by day.

CREATE INDEX idx_audit_actor    ON audit_logs ((details->>'actor_id'));
CREATE INDEX idx_audit_day      ON audit_logs (date_trunc('day', occurred_at_utc));
--   occurred_at_utc is a plain `timestamp` (already normalised to UTC) → immutable.

-- Find everything a given actor did on a given day:
SELECT
    id,
    action,
    occurred_at_utc,
    details->>'target_id' AS target
FROM audit_logs
WHERE details->>'actor_id' = $1                          -- uses idx_audit_actor
  AND date_trunc('day', occurred_at_utc) = $2::timestamp -- uses idx_audit_day
ORDER BY occurred_at_utc DESC;
```

### Example 3 — Production Grade: Case-Insensitive Unique + Covering + Partial

```sql
-- CONTEXT:
--   users table: 40M rows, ~5,000 logins/second at peak.
--   Requirement: case-insensitive unique email among NON-deleted users,
--                and login must return id + password_hash without a heap fetch.
--   Soft-deleted users (deleted_at IS NOT NULL) should NOT block re-registration
--   of the same email, so the uniqueness is PARTIAL.

-- 1. Case-insensitive uniqueness, scoped to live users (partial + expression + unique):
CREATE UNIQUE INDEX CONCURRENTLY idx_users_email_ci_active
    ON users (lower(email))
    WHERE deleted_at IS NULL;

-- 2. Covering index for the login hot path (expression key + INCLUDE payload):
CREATE INDEX CONCURRENTLY idx_users_login
    ON users (lower(email))
    INCLUDE (id, password_hash)
    WHERE deleted_at IS NULL;

ANALYZE users;

-- The login query — expected: Index-Only Scan, ~0.05 ms, 3-4 buffer hits:
SELECT id, password_hash
FROM users
WHERE lower(email) = lower($1)
  AND deleted_at IS NULL;
```

```
-- EXPLAIN (ANALYZE, BUFFERS) for the login query:
Index Only Scan using idx_users_login on users
        (cost=0.56..4.58 rows=1 width=40)
        (actual time=0.031..0.033 rows=1 loops=1)
  Index Cond: (lower(email) = lower($1))
  Heap Fetches: 0
Buffers: shared hit=4
Planning Time: 0.14 ms
Execution Time: 0.052 ms
```

**Why it performs**: the partial predicate keeps the index small (only live users), the expression key makes `lower(email)` sargable, and the `INCLUDE` payload turns it into an **Index-Only Scan** (`Heap Fetches: 0`, Topic 47) — the heap is never touched. A 40M-row login lookup in 52 microseconds.

---

## 9. Wrong → Right Patterns

### Wrong 1: Non-sargable predicate against a plain column index

```sql
-- Schema: CREATE INDEX ON users (email);   -- plain index on raw email
-- WRONG: wrapping the indexed column in lower() defeats the plain index
SELECT id FROM users WHERE lower(email) = 'bob@example.com';
```

**Exact result**: correct rows, but via a **Seq Scan** — `lower()` is computed on every row (`Rows Removed by Filter: 999999`). At 3.1 seconds on 1M rows, this is the single most common "why is login slow?" production bug.

**Why it's wrong at the execution level**: the plain index on `email` is sorted by *raw* email bytes. `lower(email)` produces a derived value with no ordering relationship to those stored keys, so the planner cannot seek — it scans and filters (Section 6.1).

```sql
-- RIGHT: build an index on the exact expression the query uses
CREATE INDEX idx_users_lower_email ON users (lower(email));
SELECT id FROM users WHERE lower(email) = 'bob@example.com';   -- now Index Scan, 0.06 ms
```

### Wrong 2: Query expression doesn't match the index expression

```sql
-- Index exists: CREATE INDEX ON users (lower(email));
-- WRONG: adds trim() → different expression tree → no match
SELECT id FROM users WHERE lower(trim(email)) = 'bob@example.com';
```

**Exact result**: Seq Scan again (`Filter: (lower(btrim(email)) = ...)`), despite the index existing. The developer "added the index" and is baffled it's still slow.

**Why it's wrong**: the planner matches expressions **structurally and exactly** (Section 6.5). `lower(btrim(email))` ≠ `lower(email)`.

```sql
-- RIGHT (option A): make the query match the index
SELECT id FROM users WHERE lower(email) = 'bob@example.com';

-- RIGHT (option B): if trimming is genuinely required, index THAT expression
CREATE INDEX ON users (lower(trim(email)));
```

### Wrong 3: Forgetting to lower the parameter → index used, zero rows

```sql
-- Index: CREATE INDEX ON users (lower(email));  stores lowercase keys
-- WRONG: parameter passed with mixed case, only the column is lowered
SELECT id FROM users WHERE lower(email) = $1;   -- $1 = 'Bob@Example.com'
```

**Exact result**: **0 rows** — and the EXPLAIN shows a happy Index Scan, so it *looks* healthy. The index stores `'bob@example.com'`; you searched for `'Bob@Example.com'`; no key matches.

**Why it's wrong**: the index is doing exactly what you asked — it's the search key that's wrong. This is a *logic* bug wearing a *performance-looking* disguise (Section 6.2).

```sql
-- RIGHT: lower BOTH sides (or lower $1 in application code before binding)
SELECT id FROM users WHERE lower(email) = lower($1);
```

### Wrong 4: Marking a non-immutable function IMMUTABLE to force the index

```sql
-- WRONG: timestamptz::date depends on session TimeZone → only STABLE.
-- Developer wraps it and lies about volatility to get past the builder:
CREATE FUNCTION day_of(ts timestamptz) RETURNS date
  LANGUAGE sql IMMUTABLE AS $$ SELECT ts::date $$;   -- LIE: this is not immutable
CREATE INDEX ON events (day_of(created_at));
```

**Exact result**: the index builds, and queries run fast — until the server's timezone (or a session's `SET TimeZone`) differs from build time. Then `created_at::date` computes a *different* day than what's stored in the index, and the query **silently returns wrong rows** (misses valid matches, finds none). Passes every same-timezone test; breaks in production for other regions.

**Why it's wrong**: the index physically stored a value that the expression no longer reproduces (Section 6.4). Immutability is not a formality — it's the correctness contract the Index Scan relies on.

```sql
-- RIGHT: make the expression genuinely immutable by pinning the timezone
CREATE INDEX ON events ((timezone('UTC', created_at)::date));
-- and query it the same way:
SELECT * FROM events WHERE timezone('UTC', created_at)::date = $1;
```

### Wrong 5: A `date_trunc` filter that the plain timestamp index can't serve

```sql
-- Index: CREATE INDEX ON orders (created_at);   -- plain btree on the timestamp
-- WRONG: truncation wraps the column → non-sargable
SELECT count(*) FROM orders WHERE date_trunc('day', created_at) = '2026-07-18';
```

**Exact result**: Seq Scan, `date_trunc` evaluated per row.

**Why it's wrong**: same non-sargability as Wrong 1 — the function wraps the indexed column.

```sql
-- RIGHT (option A): rewrite as a sargable RANGE on the bare column (no new index!)
SELECT count(*) FROM orders
WHERE created_at >= '2026-07-18' AND created_at < '2026-07-19';
-- Uses the existing plain (created_at) index — often the BEST fix, no extra write cost.

-- RIGHT (option B): if you truly query by truncated day everywhere, index the expression
CREATE INDEX ON orders (date_trunc('day', created_at_utc));  -- created_at_utc is 'timestamp'
```

The meta-lesson from Wrong 5: an expression index is not always the answer — when the transform is a *range-preserving* one (truncation, casting to date), rewriting to a bare-column range reuses an existing index and adds zero write cost. Reach for an expression index when the transform is *not* range-invertible (`lower`, `md5`, JSON extraction).

---

## 10. Performance Profile

### Read-side: the payoff

| Table size | `WHERE lower(email)=$1`, no expr index (Seq Scan) | With expression index (Index Scan) |
|-----------|---------------------------------------------------|-------------------------------------|
| 10K rows | ~2 ms (invisible in tests) | ~0.05 ms |
| 1M rows | ~3,000 ms | ~0.06 ms |
| 10M rows | ~30,000 ms (or parallel seq scan, still seconds) | ~0.07 ms |
| 100M rows | table-scan territory — unusable on hot path | ~0.08 ms (a few more tree levels) |

The Index Scan cost grows as `O(log n)` — from 1M to 100M rows the lookup adds perhaps one B-tree level (~one extra buffer hit). The Seq Scan cost grows **linearly**. This is why the bug is invisible at 10K and catastrophic at 10M: the two lines cross and diverge violently.

### Write-side: the tax

| Operation | Cost added by an expression index |
|-----------|-----------------------------------|
| INSERT | one expression evaluation + one tree insert per row |
| UPDATE not touching the column | **zero** (HOT applies; index untouched) — a real reason not to index volatile columns |
| UPDATE touching the column | one expression re-evaluation + tree update; disqualifies the row from HOT |
| DELETE | tree entry marked dead; cleaned by VACUUM |

For `lower(email)`, the per-write expression cost is negligible (a memcpy-and-fold). For expensive expressions the tax is real:

- `md5(body)` on a 100 KB `body`: hashing 100 KB on every insert — measurable, but the stored key is only 32 bytes (small index).
- An index on `lower(long_description)` updated on every save: doubles the write's index-maintenance work and bloats the index to the size of the text.

### Scaling and memory characteristics

- **Index size**: equals the size of the *computed* values plus tree overhead. `lower(email)` ≈ size of `email`. `md5(body)` ≈ 32 bytes/row regardless of body size (can be far smaller than indexing the raw column). `substr(sku,1,4)` is tiny.
- **Buffer pool**: a smaller index (e.g. a partial expression index on live users only) keeps more of the hot index resident in `shared_buffers`, improving cache hit rates.
- **`work_mem`**: an expression index that provides pre-sorted order for `ORDER BY lower(name)` or `GROUP BY lower(name)` eliminates a Sort node — avoiding an external merge sort spill for large result sets.
- **Statistics**: `ANALYZE` computes a histogram on the *expression's output* (stored against the index). Without the index, the planner has no stats for `lower(email)` and guesses selectivity poorly, often mis-estimating row counts and picking bad join orders. So the expression index improves not just the direct lookup but the planner's decisions for the whole query.

### Optimisation techniques specific to expression indexes

1. **Prefer a bare-column range rewrite when the transform is range-preserving** (Wrong 5) — zero extra write cost.
2. **Make it partial** (Section 6.9) to shrink it to the rows queries actually touch.
3. **Make it covering** (`INCLUDE`) to reach Index-Only Scans on hot paths (Topic 47).
4. **Choose the cheapest equivalent expression** — if both `lower(x)` and a `citext` column solve the problem, weigh per-write cost.
5. **Build with `CONCURRENTLY`** on large live tables to avoid a write-blocking `ACCESS EXCLUSIVE` lock during the build.

---

## 11. Node.js Integration

### 11.1 Case-insensitive login with `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// The index (run once as a migration):
//   CREATE INDEX idx_users_lower_email ON users (lower(email));

async function findUserByEmail(email) {
  // Lower the parameter in JS OR in SQL — but be consistent.
  // Here we lower in SQL so the index expression and the arg align.
  const { rows } = await pool.query(
    `SELECT id, email, password_hash, display_name
     FROM users
     WHERE lower(email) = lower($1)
       AND deleted_at IS NULL`,
    [email]                       // $1 may be any case: 'Bob@Example.COM'
  );
  return rows[0] ?? null;         // null → no such user (or wrong case handled by lower())
}
```

### 11.2 The subtle bug: lowering in JS but not aligning with the index

```javascript
// If your index is on lower(email), you can pre-lower in JS and skip lower() on the column
// ONLY IF you also store/compare consistently. Safer pattern: lower in SQL on BOTH sides.

// RISKY — column not lowered; relies on data already being lowercase:
async function riskyLookup(email) {
  const { rows } = await pool.query(
    `SELECT id FROM users WHERE email = $1`,   // plain column, case-sensitive!
    [email.toLowerCase()]                       // returns 0 rows for 'Bob@Example.com' data
  );
  return rows[0] ?? null;
}

// CORRECT — matches the lower(email) index and is case-insensitive regardless of stored case:
async function safeLookup(email) {
  const { rows } = await pool.query(
    `SELECT id FROM users WHERE lower(email) = $1`,
    [email.toLowerCase()]        // pre-lowered arg matches the lowercase index keys
  );
  return rows[0] ?? null;
}
```

Note: passing an already-lowercased `$1` and comparing to `lower(email)` is a valid and efficient pattern — the index stores lowercase keys, the arg is lowercase, they match. Just never pass mixed-case to `WHERE lower(email) = $1` (Wrong 3).

### 11.3 Handling the unique-violation on a case-insensitive constraint

```javascript
// Index: CREATE UNIQUE INDEX idx_users_email_ci ON users (lower(email)) WHERE deleted_at IS NULL;

async function registerUser({ email, passwordHash, displayName }) {
  try {
    const { rows } = await pool.query(
      `INSERT INTO users (email, password_hash, display_name)
       VALUES ($1, $2, $3)
       RETURNING id`,
      [email, passwordHash, displayName]
    );
    return { ok: true, id: rows[0].id };
  } catch (err) {
    // 23505 = unique_violation. The CI unique index caught a case-variant duplicate,
    // e.g. 'Alice@x.com' when 'alice@x.com' already exists.
    if (err.code === '23505' && err.constraint === 'idx_users_email_ci') {
      return { ok: false, reason: 'email_taken' };
    }
    throw err;
  }
}
```

### 11.4 JSONB expression-index lookup

```javascript
// Index: CREATE INDEX ON events ((payload->>'user_id'));
async function eventsForUser(userId) {
  const { rows } = await pool.query(
    `SELECT id, action, occurred_at
     FROM events
     WHERE payload->>'user_id' = $1     -- matches the expression index exactly
     ORDER BY occurred_at DESC
     LIMIT 100`,
    [String(userId)]                     // ->> yields text; bind a string
  );
  return rows;
}
```

**Does the driver handle any of this?** `pg` is a thin driver — it sends your SQL verbatim and binds `$1` params. It does nothing special for expression indexes; correctness is entirely in *your SQL matching your index*. The one driver-relevant gotcha is type: `->>` returns `text`, so bind `String(userId)`, not a number, or the comparison won't match the indexed text values.

---

## 12. ORM Comparison

The recurring theme: **creating** an expression index is a migration/DDL concern (most ORMs support it, some awkwardly), and **using** it requires that the ORM emit a query whose expression matches the index exactly. ORMs that "helpfully" wrap columns can silently produce non-sargable predicates.

### Prisma

**Can Prisma do it?** — Creating: yes, via raw SQL migrations (and, for a fixed set, `@@index` with expression support in recent versions). Using: only if you write the matching expression, which usually means `$queryRaw` — Prisma's query API cannot express `lower(email) = $1`.

```typescript
// Migration (raw SQL — the reliable path):
//   CREATE INDEX idx_users_lower_email ON users (lower(email));

// Prisma's `where: { email: { equals: x, mode: 'insensitive' } }` generates
//   WHERE email ILIKE x   (or lower(email) = lower(x) on some providers)
// On PostgreSQL, `mode: 'insensitive'` emits ILIKE, which does NOT use a plain
// lower(email) btree index (ILIKE needs a different index strategy). So:

const user = await prisma.$queryRaw`
  SELECT id, password_hash FROM users
  WHERE lower(email) = lower(${email})
    AND deleted_at IS NULL
`;   // this matches idx_users_lower_email → Index Scan
```

**Where it breaks**: `mode: 'insensitive'` → `ILIKE`, which won't use your `lower(email)` btree index (you'd need a `text_pattern_ops` or trigram index for ILIKE). Developers assume "insensitive mode + an index = fast" and get a Seq Scan. **Verdict**: create the index in raw SQL, and do case-insensitive equality lookups via `$queryRaw` with `lower(...) = lower(...)`.

### Drizzle ORM

**Can Drizzle do it?** — Yes, both sides cleanly. Drizzle supports expression indexes in the schema and lets you build the matching predicate with `sql`.

```typescript
import { pgTable, serial, text, timestamp, index, uniqueIndex } from 'drizzle-orm/pg-core';
import { sql, eq } from 'drizzle-orm';

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  email: text('email').notNull(),
  deletedAt: timestamp('deleted_at'),
}, (t) => ({
  // Expression index in the schema:
  lowerEmail: index('idx_users_lower_email').on(sql`lower(${t.email})`),
  // Unique, partial, case-insensitive:
  ciEmail: uniqueIndex('idx_users_email_ci')
    .on(sql`lower(${t.email})`)
    .where(sql`${t.deletedAt} IS NULL`),
}));

// Querying with the matching expression:
const rows = await db.select({ id: users.id })
  .from(users)
  .where(sql`lower(${users.email}) = lower(${input})`);   // matches the index
```

**Where it breaks**: nothing structural — you must remember to write `sql\`lower(...)\`` in the predicate; `eq(users.email, x)` alone is case-sensitive and won't use the expression index. **Verdict**: best-in-class — expression indexes and their matching predicates are both first-class.

### Sequelize

**Can Sequelize do it?** — Creating: yes, via `indexes` with `fields: [sequelize.fn('lower', sequelize.col('email'))]` or a raw migration. Using: via `where: sequelize.where(sequelize.fn('lower', sequelize.col('email')), input)`.

```javascript
// Model definition:
const User = sequelize.define('User', { email: DataTypes.STRING }, {
  indexes: [{
    name: 'idx_users_lower_email',
    fields: [sequelize.fn('lower', sequelize.col('email'))],
  }],
});

// Query that matches the index:
const user = await User.findOne({
  where: sequelize.where(
    sequelize.fn('lower', sequelize.col('email')),
    input.toLowerCase()
  ),
});
```

**Where it breaks**: `where: { email: input }` is case-sensitive; the case-insensitive form requires the verbose `sequelize.where(sequelize.fn(...))`, which developers avoid, falling back to `Op.iLike` (→ ILIKE, no btree index). The index definition via `sequelize.fn` also silently no-ops on some dialect/version combos — verify the generated DDL. **Verdict**: works but verbose; confirm the migration actually created the expression index, and use `sequelize.where(fn(...))` to hit it.

### TypeORM

**Can TypeORM do it?** — Creating: yes, `@Index('idx', { synchronize: false })` plus a raw migration, since the decorator can't express `lower(email)` directly in older versions. Using: via QueryBuilder with a raw expression.

```typescript
// Migration:
//   await queryRunner.query(`CREATE INDEX idx_users_lower_email ON users (lower(email))`);

const user = await dataSource.getRepository(User)
  .createQueryBuilder('u')
  .where('lower(u.email) = lower(:email)', { email: input })   // matches the index
  .getOne();
```

**Where it breaks**: the entity decorators can't declaratively define an arbitrary expression index in a portable way, so you maintain it in a migration and mark the entity index `synchronize: false` to stop TypeORM trying to drop/recreate it. `find({ where: { email } })` is case-sensitive and won't use it. **Verdict**: use a raw migration for the index and QueryBuilder with a raw `lower(...)` predicate; keep `synchronize` from clobbering it.

### Knex.js

**Can Knex do it?** — Creating: yes, via `knex.raw` in a migration (the schema builder's `.index()` takes columns, not expressions). Using: via `whereRaw`.

```javascript
// Migration:
exports.up = (knex) =>
  knex.raw(`CREATE INDEX idx_users_lower_email ON users (lower(email))`);

// Query:
const user = await knex('users')
  .whereRaw('lower(email) = lower(?)', [input])   // matches the index
  .first();
```

**Where it breaks**: `.index(['email'])` cannot express an expression — you must drop to `knex.raw`. `.where('email', input)` is case-sensitive. **Verdict**: fully capable through `raw`/`whereRaw`; Knex is transparent, so what you write is what runs — matching the index is straightforward.

### ORM Summary Table

| ORM | Create expr index | Emit matching predicate | Main pitfall | Verdict |
|-----|-------------------|-------------------------|--------------|---------|
| Prisma | Raw SQL migration | `$queryRaw` only | `mode:'insensitive'` → ILIKE, ignores btree index | Raw SQL both ends |
| Drizzle | `index().on(sql\`lower()\`)` | `sql\`lower()=lower()\`` | must remember the `sql` predicate | Best support |
| Sequelize | `fields:[fn('lower',col)]` | `where(fn('lower',col),...)` | verbose; `iLike` bypasses index | Works, verify DDL |
| TypeORM | Raw migration + `synchronize:false` | QueryBuilder raw predicate | decorators can't express it | Migration + QueryBuilder |
| Knex | `knex.raw` migration | `whereRaw('lower(...)')` | `.index()` can't do expressions | Transparent via raw |

---

## 13. Practice Exercises

### Exercise 1 — Easy

The `users` table has a `username` column. Logins are currently `WHERE username = $1` and are case-sensitive, but product wants case-insensitive login and 100M rows are coming.

1. Write the DDL to make `WHERE lower(username) = lower($1)` fast.
2. Write the login query.
3. Also enforce that usernames are unique **case-insensitively**.

```sql
-- 1. Expression index:
-- Write your DDL here

-- 2. Login query:
-- Write your query here

-- 3. Case-insensitive unique constraint:
-- Write your DDL here
```

### Exercise 2 — Medium (combines Topic 45 Partial Indexes)

The `orders` table (25M rows) has `status`, `customer_email`, and `created_at`. Support agents look up **only pending** orders by the customer's email, case-insensitively:

```sql
SELECT id, total_amount, created_at
FROM orders
WHERE lower(customer_email) = lower($1)
  AND status = 'pending';
```

Only ~0.5% of orders are `pending`. Design the single most efficient index for this query and explain in one sentence why it beats a plain `lower(customer_email)` index.

```sql
-- Write your DDL here
-- One-sentence justification:
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A reporting query buckets audit events by day and actor:

```sql
SELECT date_trunc('day', occurred_at) AS day, count(*)
FROM audit_logs                       -- occurred_at is timestamptz
WHERE date_trunc('day', occurred_at) = $1
GROUP BY 1;
```

A junior engineer runs `CREATE INDEX ON audit_logs (date_trunc('day', occurred_at));` and it **fails**.

1. Why does the `CREATE INDEX` fail? (Be precise about volatility.)
2. Give **two** correct fixes: one that adds an expression index, and one that requires **no new index at all**.
3. If the table is 200M rows and reports always query by UTC day, which fix do you ship and why?

```sql
-- 1. Reason for failure:
-- 2a. Expression-index fix:
-- Write your DDL + query here
-- 2b. No-new-index fix:
-- Write your query here
-- 3. Decision + reasoning:
```

### Exercise 4 — Interview Simulation

You're told: "We added `CREATE INDEX ON users (lower(email))` last week, but the login query `WHERE lower(email) = $1` is *still* doing a Seq Scan in production. The index definitely exists. What are all the possible reasons, and how do you diagnose each?"

Write your structured answer (aim for at least four distinct causes with a diagnostic for each).

```sql
-- Write your answer here (prose + any diagnostic SQL)
```

---

## 14. Interview Questions

### Question 1 — Why is `WHERE lower(email) = $1` slow, and how do you fix it?

**The question**: "Our login endpoint does `SELECT ... FROM users WHERE lower(email) = $1`. There's an index on `email`. It's slow at scale. Why, and what do you do?"

**Junior answer**: "Add an index on `email`." (There already is one; this misses that the existing index is unusable here.)

**Principal answer**: "The predicate is **non-sargable** against the plain `email` index. That index is sorted by the raw email value, but the query searches on `lower(email)` — a derived value with no ordering relationship to the stored keys, so the planner can't seek into the tree and falls back to a Seq Scan, computing `lower()` on every row. The fix is an **expression index**: `CREATE INDEX ON users (lower(email))`, which stores the lowercased value as the key. The query then does an Index Scan because its expression matches the index's key expression exactly. I'd build it `CONCURRENTLY` on a live table, run `ANALYZE`, and confirm in EXPLAIN that `lower(email)` appears as an `Index Cond`, not a `Filter`."

**Interviewer follow-up**: "Would `citext` also solve it?" → "Yes — `email citext` makes plain `=` case-insensitive and a normal index serves it. Trade-off: `citext` makes the *whole column* case-insensitive at the type level, while an expression index is opt-in per query path and leaves the raw column case-sensitive. For an existing table or selective case-insensitivity, I prefer the expression index."

### Question 2 — Why must the indexed expression be IMMUTABLE?

**The question**: "PostgreSQL rejects `CREATE INDEX ON events (created_at::date)` with 'functions in index expression must be marked IMMUTABLE'. Explain the requirement and what would happen if it were relaxed."

**Junior answer**: "It's just a rule; wrap it in a function marked IMMUTABLE and it works." (Dangerously wrong — this creates a correctness bug.)

**Principal answer**: "An index physically **stores the computed value** at write time and never re-derives it on read; the Index Scan trusts that recomputing the expression today yields the same key that was stored. That's only true for IMMUTABLE functions. `created_at::date` where `created_at` is `timestamptz` depends on the session `TimeZone`, so it's only **STABLE** — the same row could map to a different date under a different timezone. If PostgreSQL allowed it, then after a timezone change the query would compute a key that no longer matches what's stored, and the Index Scan would **silently miss rows**. The correct fix is to make the expression genuinely immutable — pin the timezone: `((timezone('UTC', created_at))::date)` — not to lie by marking a non-immutable function IMMUTABLE, which builds the index but returns wrong results across timezones."

**Interviewer follow-up**: "So why is `lower()` allowed?" → "Because `lower()` on `text` with a fixed collation is deterministic — same input, same output, forever, with no session/time/config dependence. (Caveat: locale-dependent case folding is why PostgreSQL ties immutability to a deterministic collation; the default `lower()` qualifies.)"

### Question 3 — The index exists but isn't used

**The question**: "There's an index on `lower(email)` and the query filters on `lower(trim(email))`. It's doing a Seq Scan. Why?"

**Junior answer**: "Maybe the statistics are stale; run ANALYZE." (Won't help — the expressions don't match.)

**Principal answer**: "The planner matches the query expression against the index's key expression **structurally and exactly**. `lower(trim(email))` is a different expression tree than `lower(email)` — the extra `trim()` node means no match, so the index is inapplicable and it scans. Two fixes: either make the query match the index (`lower(email) = $1` if trimming isn't actually needed), or, if trimming is a real requirement, index the exact expression the query uses: `CREATE INDEX ON users (lower(trim(email)))`. The diagnostic is EXPLAIN — the indexed expression should appear as `Index Cond`; if it's under `Filter` with a Seq Scan, the expressions don't line up."

**Interviewer follow-up**: "What if the query is `WHERE lower(email) = $1` exactly and it *still* seq-scans?" → "Then check: (a) is `$1` selective enough — for a low-cardinality expression the planner may correctly prefer a scan; (b) is the index `INVALID` from a failed `CONCURRENTLY` build (check `pg_index.indisvalid`); (c) is there a type mismatch forcing a cast around the column; (d) was `ANALYZE` run so the planner has selectivity stats for the expression."

### Question 4 — Write cost and when NOT to add one

**The question**: "You maintain a write-heavy table updated thousands of times per second. When would you *avoid* adding an expression index, and what's the cheaper alternative for a truncation predicate?"

**Junior answer**: "Indexes always help; add it." (Ignores write amplification.)

**Principal answer**: "Every expression index adds per-write cost: the expression is evaluated on every INSERT and on every UPDATE that touches the underlying column, and indexing a column disqualifies its updates from the HOT optimisation. On a hot write path, an expensive expression (say `md5(large_body)` or an index on a frequently-updated text column) can materially raise write latency and index bloat. Two mitigations: make it **partial** so only the queried subset is indexed, and — crucially — recognise when you don't need an expression index at all. For a **range-preserving** transform like `date_trunc('day', ts) = $1` or `ts::date = $1`, rewrite to a bare-column range: `ts >= $1 AND ts < $1 + interval '1 day'`. That reuses the existing plain `(ts)` index and adds **zero** write cost. Reserve expression indexes for transforms that aren't range-invertible — `lower`, `md5`, JSONB extraction."

**Interviewer follow-up**: "How do you decide the read win is worth the write tax?" → "Measure both: `pg_stat_statements` for the read query's total time and call rate, and a load test of the write path with and without the index. If the read path is hot (login) and the expression is cheap (`lower`), it's a clear win; if the read is rare and the expression is costly on a write-heavy table, I'd look at the range-rewrite or a partial index first."

---

## 15. Mental Model Checkpoint

1. You have `CREATE INDEX ON users (lower(email))`. Explain, in terms of what the B-tree physically stores and what it's sorted by, why `WHERE lower(email) = $1` can seek but `WHERE upper(email) = $1` cannot.

2. A predicate is "non-sargable." Define what that means precisely, and give the two fundamentally different ways to make `WHERE created_at::date = $1` fast — noting which one adds write cost and which doesn't.

3. Why does PostgreSQL forbid non-IMMUTABLE expressions in an index? Walk through the exact failure mode that would occur if it allowed `created_at::date` (timestamptz) and the server timezone later changed.

4. Your index is on `lower(email)` and you run `WHERE lower(email) = $1` with `$1 = 'Bob@X.com'`. EXPLAIN shows a healthy Index Scan, yet the query returns zero rows for a user who exists. What's wrong, and why does EXPLAIN look fine?

5. When would you reach for `citext` instead of an expression index, and when the reverse? Name one behavioural difference beyond performance.

6. You add `CREATE INDEX ON users (lower(email))` to a table where `email` is updated on almost every request (say it doubles as a "last contact" field). What happens to write performance and HOT, and would your answer change if the frequently-updated column were a *different*, non-indexed column?

7. Explain how a single index `CREATE INDEX ON users (lower(email)) INCLUDE (id, password_hash) WHERE deleted_at IS NULL` combines three orthogonal ideas, and state which part answers "what value," "which rows," and "what payload."

---

## 16. Quick Reference Card

```sql
-- ── CREATE ──────────────────────────────────────────────────────────────
-- Basic expression index (note the REQUIRED parentheses around the expression):
CREATE INDEX idx ON users (lower(email));

-- Unique / case-insensitive uniqueness (can't be a table UNIQUE constraint):
CREATE UNIQUE INDEX ON users (lower(email));

-- JSONB field:
CREATE INDEX ON events ((payload->>'user_id'));   -- double parens for operator exprs

-- Immutable date bucket (use a plain 'timestamp' or pin the tz):
CREATE INDEX ON logs (date_trunc('day', created_at_utc));
CREATE INDEX ON logs ((timezone('UTC', created_at)::date));   -- timestamptz → immutable

-- Combined with partial (Topic 45) + covering (Topic 47):
CREATE INDEX ON users (lower(email)) INCLUDE (id) WHERE deleted_at IS NULL;

-- Anchored LIKE on an expression (needs the pattern opclass):
CREATE INDEX ON users (lower(email) text_pattern_ops);   -- serves lower(email) LIKE 'a%'

-- Build without blocking writes on a live table:
CREATE INDEX CONCURRENTLY ...

-- ── USE (query MUST match the index expression EXACTLY) ────────────────────
WHERE lower(email) = lower($1)          -- ✓ matches lower(email) index
WHERE lower(email) IN ($1, $2)          -- ✓ matches
ORDER BY lower(name)                    -- ✓ index provides order, skips Sort
WHERE upper(email) = $1                 -- ✗ different function → Seq Scan
WHERE lower(trim(email)) = $1           -- ✗ different expression tree → Seq Scan

-- ── RULES OF THUMB ─────────────────────────────────────────────────────────
-- • Non-sargable = column wrapped in a function → plain index unusable.
-- • Fix 1 (range-preserving transform): rewrite to a bare-column range. NO write cost.
--     created_at::date = $1  →  created_at >= $1 AND created_at < $1 + 1 day
-- • Fix 2 (non-invertible transform: lower/md5/jsonb): build an expression index.
-- • Expression MUST be IMMUTABLE. timestamptz::date is only STABLE → pin the tz.
-- • Never mark a non-immutable function IMMUTABLE — silent wrong-result bug.
-- • Write cost: expr evaluated on INSERT + on UPDATE of the underlying column;
--   indexing a column blocks HOT for its updates.
-- • Case-insensitive UNIQUE = unique expression index (or citext + unique).
-- • Confirm in EXPLAIN: indexed expr under `Index Cond` (good), not `Filter` (bad).

-- ── INTERVIEW ONE-LINERS ───────────────────────────────────────────────────
-- "lower(email)=$1 is non-sargable against a plain email index; index the expression."
-- "An expression index is an ordinary B-tree that stores a function's output as the key."
-- "IMMUTABLE is mandatory because the computed key is stored, never recomputed on read."
-- "The query expression must match the index expression structurally and exactly."
-- "For range-preserving transforms, prefer a bare-column range over a new index."
```

---

## Connected Topics

- **Topic 42 — B-tree Index Fundamentals**: what the tree stores and how it's sorted — the substrate an expression index reuses, keyed on a computed value instead of a raw column.
- **Topic 43 — Index Scan vs Seq Scan vs Bitmap**: the access-path choice; expression indexes flip a `WHERE lower(email)=$1` from Seq Scan to Index Scan.
- **Topic 44 — Composite / Multi-Column Indexes**: expression keys can be columns in a composite index (`(lower(email), created_at)`), following the same leftmost-prefix rules.
- **Topic 45 — Partial Indexes**: the orthogonal "which rows" modifier — combine with expression indexes for case-insensitive uniqueness scoped to live rows.
- **Topic 47 — Index-Only Scans**: add `INCLUDE (...)` to an expression index to serve the query entirely from the index (`Heap Fetches: 0`) — the covering half of the production example.
- **Topic 07 — NULL in Depth**: `lower(NULL)` is NULL; expression indexes store NULL keys per the same three-valued rules, relevant to `coalesce()`-based expression indexes.
- **Topic 11 — INNER JOIN in Depth**: expression indexes on both sides of `ON lower(a.email)=lower(b.email)` enable indexed Nested Loop / Merge Joins on computed keys.
- **Internals — Function Volatility (IMMUTABLE/STABLE/VOLATILE)**: the classification that gates which expressions are indexable and why timezone-dependent casts are rejected.
