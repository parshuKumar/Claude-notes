# Topic 72 — Row Level Security (RLS)
### SQL Mastery Curriculum — Phase 10: PostgreSQL Specific Power Features

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a giant office building with one enormous filing room. Every company in the city rents desk space in that same building, and all of their paper files live in that single shared room, in the same cabinets, mixed together.

Now, you could give each company a rule: "Please only open the folders with your own company's name on the tab." That's the honor system — and it's how most applications handle multi-tenancy: the *application code* promises to always add `WHERE tenant_id = 42` to every query. It works right up until one developer forgets the WHERE clause on one endpoint, and suddenly Company A is reading Company B's payroll.

Row Level Security is different. It's like hiring a guard who stands at the door of the filing room and clips an unremovable badge to every person who walks in. The badge says "You are Company 42." From that moment, *the cabinets themselves* refuse to open any folder that isn't tagged 42. The person can type any request they want — `SELECT * FROM everything` — and the room will still only physically hand them their own folders. The filter is no longer a promise the person makes; it's a law the room enforces.

The key mental shift: **without RLS, security lives in your application code and every query must remember to filter. With RLS, security lives in the database table itself, and the filter is applied automatically to every query — even the ones you forgot to filter, even the ones written by a future intern, even a raw `psql` session.** The database appends an invisible `AND (your_policy)` to every statement touching that table. You cannot query around it, because there is no query that turns it off.

That's RLS: the WHERE clause you can never forget, welded to the table instead of bolted to the query.

---

## 2. Connection to SQL Internals

RLS is not a storage feature — it changes nothing about how rows are laid out on heap pages, how B-trees are built, or how MVCC versions rows. What RLS touches is the **query rewrite / planning stage**, specifically the step between parse and plan where PostgreSQL expands views, resolves inheritance, and applies security barriers.

Here is the internal chain:

1. **Parser** turns your SQL text into a parse tree.
2. **Rewriter** (`src/backend/rewrite/`) is where RLS lives. When the rewriter sees a table that has RLS enabled, it looks up the applicable policies in the `pg_policy` catalog and **injects the policy expressions as additional qualifiers** on the range table entry. A `USING` clause becomes an extra `WHERE`-style qual; a `WITH CHECK` clause becomes a constraint evaluated after the row is computed for INSERT/UPDATE.
3. **Planner** then optimizes the rewritten tree exactly as if you had typed those predicates yourself. This is the crucial internal fact: **an RLS policy is, to the planner, just another predicate.** It participates in index selection, selectivity estimation, and join ordering like any hand-written WHERE clause.

Because the policy is a real predicate, it interacts with the same internals you already know:

- **B-tree indexes**: If your policy is `tenant_id = current_setting('app.tenant_id')::bigint`, and there is an index on `tenant_id`, the planner can use that index for the policy qual — an Index Scan or Index-Only Scan, not a full Seq Scan. RLS performance is *entirely* about whether the policy predicate is indexable and estimable. This is the same lesson as Topic 11's "index on the join column."
- **Security barriers / leakproofness**: PostgreSQL must prevent a clever user from writing a `WHERE` clause whose function *leaks* the value of a row they shouldn't see (e.g. a function that raises an error containing a hidden salary). To defend against this, RLS-protected tables are treated like security-barrier views: the planner will not push a non-`leakproof` user function below the RLS qual. This can block some optimizations — a subtle performance implication we cover in section 10.
- **MVCC**: RLS is evaluated per-row per-snapshot like any qual. The policy sees the same tuple visibility rules (Topic on MVCC) as the rest of the query — a row invisible to your snapshot is never even offered to the policy.
- **Planner caching (`plan_cache_mode`)**: Because policies reference session state like `current_setting('app.tenant_id')` and `current_user`, a prepared statement's cached plan can become subtly wrong across role changes. PostgreSQL invalidates RLS-affected plans when the role changes, which is why generic plans and RLS need care under connection pooling (section 11).

The one-sentence internal summary: **RLS is predicate injection at rewrite time; everything else — indexes, MVCC, cost estimation — treats the injected predicate exactly like a predicate you wrote yourself, with the single exception that leakproofness rules restrict how far it can be pushed.**

---

## 3. Logical Execution Order Context

RLS does not appear in the classic `FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT` list because it operates *before* logical execution even begins — it is a **rewrite-phase** transformation that modifies the query tree the logical order then runs on.

The correct mental model is a phase that sits ahead of everything:

```
── REWRITE PHASE (before logical execution) ──
   View expansion
   RLS policy injection            ← policies added as quals here
──────────────────────────────────────────────
FROM        table (RLS qual now attached to this range table entry)
JOIN        ...                    ← RLS quals attached to each RLS-enabled table in the join
WHERE       your predicates AND the injected USING predicate
GROUP BY    ...
HAVING      ...
SELECT      ...
window      ...
ORDER BY    ...
LIMIT       ...
```

Two consequences of *where* RLS injects matter enormously:

1. **The `USING` predicate behaves like an inlined WHERE qual on that table.** For a `SELECT`, it filters which rows are even visible before your own `WHERE` runs. Logically, `USING (tenant_id = 42)` and your typing `WHERE tenant_id = 42` produce the same row set — but RLS's version is un-forgettable and applies before joins can leak rows.

2. **For writes, `WITH CHECK` runs at a different point than `USING`.** `USING` is checked when a row is *read or targeted* (the SELECT/UPDATE/DELETE visibility gate). `WITH CHECK` is checked when a row is *produced* (the post-image of an INSERT or UPDATE). So an UPDATE runs `USING` against the *old* row (can I see/target it?) and `WITH CHECK` against the *new* row (am I allowed to leave it in this state?). This is the RLS analog of the ON-vs-WHERE distinction from Topic 11 and Topic 12: *where* in the pipeline the check fires changes the meaning.

3. **RLS applies per-table in a join.** If you join `orders` (RLS on) to `order_items` (RLS on), each table independently gets its policy qual injected on its own range table entry. A row is only visible if it passes its own table's policy — a join cannot smuggle a row past its table's RLS.

---

## 4. What Is Row Level Security?

Row Level Security is a PostgreSQL feature that lets you attach **row-visibility and row-writability policies directly to a table**, so that the database automatically restricts which rows each role can `SELECT`, `INSERT`, `UPDATE`, or `DELETE` — without any change to the queries the application sends. It is enabled per-table with `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` and configured with one or more `CREATE POLICY` statements.

Two clauses define a policy:
- **`USING (expr)`** — the *visibility / read* predicate. A row is visible (or targetable by UPDATE/DELETE) only if `expr` is TRUE for that row.
- **`WITH CHECK (expr)`** — the *write* predicate. A new or modified row is allowed only if `expr` is TRUE for the resulting row.

```sql
-- Step 1: turn RLS on for the table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
--         │      │
--         │      └── from this point, ALL access to orders is DENIED by default
--         │          for non-owner roles until a policy grants it
--         └── the target table

-- Step 2: define a policy
CREATE POLICY tenant_isolation            -- policy name (unique per table)
  ON orders                               -- the table the policy guards
  AS PERMISSIVE                           -- PERMISSIVE (default) = OR-combined with other permissive policies
                                          -- │  alternative: RESTRICTIVE = AND-combined (must also pass)
  FOR ALL                                 -- which commands: ALL | SELECT | INSERT | UPDATE | DELETE
                                          -- │  ALL covers select/insert/update/delete
  TO application_role                      -- which DB roles this policy applies to (default: PUBLIC)
                                          -- │  can list multiple roles, or PUBLIC for everyone
  USING (                                 -- READ/VISIBILITY predicate (select, and old-row for update/delete)
    tenant_id = current_setting('app.tenant_id')::bigint
                                          -- │  only rows whose tenant_id matches the session variable are visible
  )
  WITH CHECK (                            -- WRITE predicate (new-row for insert/update)
    tenant_id = current_setting('app.tenant_id')::bigint
                                          -- │  inserted/updated rows MUST carry the session's tenant_id
  );
```

### The exact default-deny semantics

Once you run `ALTER TABLE orders ENABLE ROW LEVEL SECURITY`:
- If **no policy** exists, the table is **default-deny**: non-owner roles see zero rows and can write nothing. (The table owner and superusers bypass RLS by default — see `FORCE` and `BYPASSRLS` below.)
- Each policy is **permissive by default**, meaning multiple permissive policies are combined with **OR** — a row is visible if *any* permissive policy's `USING` is TRUE.
- **Restrictive** policies (`AS RESTRICTIVE`) are combined with **AND** — a row must pass *every* restrictive policy in addition to at least one permissive policy.
- The final rule: **`(at least one PERMISSIVE USING is TRUE) AND (every RESTRICTIVE USING is TRUE)`**.

### Clause-to-command matrix

| Command | `USING` checked against | `WITH CHECK` checked against |
|---------|------------------------|------------------------------|
| `SELECT` | each candidate row (visibility) | — (not applicable) |
| `INSERT` | — (not applicable) | the new row (must satisfy) |
| `UPDATE` | the existing/old row (targetable?) | the new row (allowed result?) |
| `DELETE` | the existing/old row (targetable?) | — (not applicable) |

If `WITH CHECK` is omitted on a policy that needs it, PostgreSQL falls back to using the `USING` expression as the check. This is a common and usually desirable default: `USING (tenant_id = X)` with no explicit `WITH CHECK` means writes must also land in tenant X.

---

## 5. Why RLS Mastery Matters in Production

1. **It is the only tenant-isolation mechanism that survives a forgotten WHERE clause.** Application-enforced isolation (`WHERE tenant_id = ?` in every query) is one missing predicate away from a cross-tenant data breach. In a codebase with hundreds of endpoints and multiple developers, "every query remembers to filter" is not a security guarantee — it's a hope. RLS moves the guarantee into the table, where no endpoint can bypass it. The cost of *not* mastering it is a data-leak incident and a compliance failure.

2. **It is defense-in-depth against SQL injection and ORM escape hatches.** Even if an attacker injects `OR 1=1` or a developer drops to `$queryRaw`, the RLS predicate is still appended by the rewriter. A successful injection on an RLS-protected table still only returns the current tenant's rows. Without RLS, an injection is a full-table dump.

3. **It enables safe multi-tenant SaaS on a single shared database.** The single-database, shared-schema multi-tenant model (cheapest to operate, hardest to secure) becomes viable because RLS provides hard row isolation. Getting the `current_setting('app.tenant_id')` pattern and the per-request `SET LOCAL` wiring right is what makes this architecture safe under a connection pool.

4. **Getting it wrong is silently catastrophic in two opposite directions.** A too-loose policy leaks data with no error. A too-tight policy (or a forgotten `WITH CHECK`) silently returns zero rows or silently drops writes — an UPDATE that "succeeds" with `0 rows affected` because the new row failed the check, or an INSERT that the app thinks worked. Both failure modes are invisible without deliberate testing, which is why understanding the exact `USING`/`WITH CHECK`/`PERMISSIVE`/`RESTRICTIVE` semantics is a production necessity, not trivia.

5. **The performance characteristics are non-obvious and can dominate query cost.** Because the policy is an injected predicate evaluated on every row of every query, a non-indexable or expensive policy (e.g. one that calls a function or a subquery per row) can turn a fast query into a slow one across your entire application at once. Mastering the index interaction (section 10) is what keeps RLS free.

---

## 6. Deep Technical Content

### 6.1 Enabling, disabling, and the default-deny gate

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;   -- policies now apply to non-owners
ALTER TABLE orders DISABLE ROW LEVEL SECURITY;  -- policies exist but are inert
```

Enabling RLS with **no policies defined** is the maximally restrictive state: every non-owner role sees zero rows. This is deliberate and safe — you turn on the gate first, then punch specific holes with policies. A frequent mistake is enabling RLS in a migration and forgetting to add the policy, which makes the table appear empty to the application (which connects as a non-owner role) while looking full to the migration author (who connects as the owner and bypasses RLS).

Enabling RLS affects **`SELECT`, `INSERT`, `UPDATE`, `DELETE`**. It does **not** affect `TRUNCATE` (that's governed by table privileges) and does not affect the table owner unless `FORCE ROW LEVEL SECURITY` is set (section 6.7).

### 6.2 `USING` — the read/visibility predicate

`USING` answers: *"which existing rows can this role see or target?"* It is evaluated for `SELECT` (which rows are returned), and for the *old* image in `UPDATE`/`DELETE` (which rows can be modified or removed).

```sql
CREATE POLICY p_visible ON orders
  FOR SELECT
  USING (tenant_id = current_setting('app.tenant_id')::bigint);
```

Rows where the expression is FALSE or NULL are **silently omitted** — never an error, just absent. This is exactly the NULL-drop behavior from Topic 11's INNER JOIN and Topic 07's three-valued logic: `USING` keeps rows only where the predicate is TRUE, and NULL is not TRUE.

A critical, often-missed consequence for `UPDATE`/`DELETE`: **rows you cannot see, you cannot modify or delete.** `DELETE FROM orders WHERE id = 999` when row 999 belongs to another tenant does not error — it deletes 0 rows, because the `USING` predicate hid row 999 from the DELETE's target set. Your application sees "0 rows affected," which it must be prepared to interpret as "not found *or* not permitted."

### 6.3 `WITH CHECK` — the write predicate

`WITH CHECK` answers: *"is the row this write is about to leave in the table allowed?"* It is evaluated against the **new row image** for `INSERT` and `UPDATE`. If it evaluates to FALSE or NULL, the entire statement **raises an error** and rolls back:

```
ERROR:  new row violates row-level security policy for table "orders"
```

Note the asymmetry with `USING`: a `USING` failure *silently hides* a row; a `WITH CHECK` failure *loudly errors*. This is intentional. Hiding a row you can't read is normal filtering; attempting to write a row you're not allowed to write is a violation worth surfacing.

```sql
CREATE POLICY p_insert ON orders
  FOR INSERT
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);

-- Session is tenant 42:
INSERT INTO orders (tenant_id, total_amount) VALUES (42, 100);  -- OK
INSERT INTO orders (tenant_id, total_amount) VALUES (99, 100);  -- ERROR: violates RLS
```

The `WITH CHECK` on INSERT is what prevents a tenant from *planting* rows in another tenant's space. Without it (using only `USING`), a tenant could insert `tenant_id = 99` rows they then couldn't see — a data-integrity hole and a potential denial-of-service (bloating another tenant's data).

**The UPDATE double-check.** An `UPDATE` runs *both* predicates at different points:

```sql
CREATE POLICY p_update ON orders
  FOR UPDATE
  USING      (tenant_id = current_setting('app.tenant_id')::bigint)   -- can I target this old row?
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);  -- is the new row still mine?
```

- `USING` gates *which rows the UPDATE is even allowed to touch* (old image).
- `WITH CHECK` gates *what the row is allowed to become* (new image).

With the policy above, tenant 42 can update its own rows but **cannot change a row's `tenant_id` to 99** — the new image would fail `WITH CHECK`, erroring the statement. This is how RLS prevents "moving" a row out of your tenant.

### 6.4 The default-check fallback

If a policy provides `USING` but omits `WITH CHECK`, PostgreSQL uses the `USING` expression *as* the `WITH CHECK`. So:

```sql
-- These two are equivalent for a FOR ALL policy:
CREATE POLICY p ON orders FOR ALL USING (tenant_id = X);
CREATE POLICY p ON orders FOR ALL USING (tenant_id = X) WITH CHECK (tenant_id = X);
```

Conversely, a `FOR INSERT` policy has **no** `USING` (there is no old row to read) — it takes only `WITH CHECK`. Writing `USING` on a `FOR INSERT` policy is a syntax error.

| `FOR` | Accepts `USING`? | Accepts `WITH CHECK`? |
|-------|:---:|:---:|
| `SELECT` | yes | no |
| `INSERT` | no | yes |
| `UPDATE` | yes | yes |
| `DELETE` | yes | no |
| `ALL` | yes | yes |

### 6.5 PERMISSIVE vs RESTRICTIVE — the combination algebra

This is the single most misunderstood part of RLS. When multiple policies apply to the same command and role, they combine by a fixed boolean rule:

```
row_allowed =  ( OR of all PERMISSIVE policies' predicates )
             AND
               ( AND of all RESTRICTIVE policies' predicates )
```

- **PERMISSIVE (default):** grants access. Multiple permissive policies are **OR**ed — think of each as *"here is another set of rows you may see."* Adding a permissive policy can only *widen* access.
- **RESTRICTIVE (`AS RESTRICTIVE`):** narrows access. Restrictive policies are **AND**ed on top — think of each as *"but also, the row must additionally satisfy this."* Adding a restrictive policy can only *narrow* access.

A subtle and important rule: **if there are only RESTRICTIVE policies and no PERMISSIVE policy, nothing is visible.** The `OR of permissive` term is empty (defaults to FALSE), and `FALSE AND anything = FALSE`. You must have at least one permissive policy to grant a baseline, then restrictive policies to carve it down.

```sql
-- Permissive baseline: you can see your tenant's rows
CREATE POLICY tenant_read ON orders
  AS PERMISSIVE FOR SELECT
  USING (tenant_id = current_setting('app.tenant_id')::bigint);

-- Permissive addition (OR): support staff can ALSO see flagged rows
CREATE POLICY support_read ON orders
  AS PERMISSIVE FOR SELECT
  TO support_role
  USING (is_flagged_for_support = TRUE);
-- Effect for a support_role session: (tenant match) OR (is_flagged) → sees MORE

-- Restrictive addition (AND): NOBODY may see soft-deleted rows, ever
CREATE POLICY hide_deleted ON orders
  AS RESTRICTIVE FOR SELECT
  USING (deleted_at IS NULL);
-- Effect: ((tenant match) OR (is_flagged)) AND (deleted_at IS NULL)
```

Use **restrictive** policies for invariants that must hold *regardless* of any grant — "never show deleted rows," "never show rows outside the active date range." Use **permissive** policies to enumerate the *ways* a role may gain access.

### 6.6 `FOR` — scoping policies to specific commands

You can (and often should) write separate policies per command to express different read vs write rules:

```sql
-- Everyone in the tenant can READ all tenant orders
CREATE POLICY tenant_select ON orders
  FOR SELECT
  USING (tenant_id = current_setting('app.tenant_id')::bigint);

-- But only the owner of a row can UPDATE it
CREATE POLICY owner_update ON orders
  FOR UPDATE
  USING      (tenant_id = current_setting('app.tenant_id')::bigint
              AND created_by = current_setting('app.user_id')::bigint)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint
              AND created_by = current_setting('app.user_id')::bigint);

-- And only admins can DELETE
CREATE POLICY admin_delete ON orders
  FOR DELETE
  TO admin_role
  USING (tenant_id = current_setting('app.tenant_id')::bigint);
```

A `FOR ALL` policy applies its `USING` to select/update/delete and its `WITH CHECK` to insert/update — a single policy governing every command. Prefer command-specific policies when read and write rules diverge; prefer `FOR ALL` when they are identical (like pure tenant isolation).

### 6.7 `FORCE ROW LEVEL SECURITY` — closing the owner bypass

By default, **the table owner is exempt from RLS** on their own table. This is a footgun: if your application connects as the role that owns the tables (a very common misconfiguration), RLS is silently *not applied* and every query sees every row.

```sql
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
```

`FORCE` makes RLS apply **even to the table owner**. The only remaining bypass is a role with the `BYPASSRLS` attribute (and superusers). Best practice:

- Own the tables with a dedicated `app_owner`/migration role.
- Have the application connect as a *separate*, non-owner `app_user` role.
- Additionally set `FORCE ROW LEVEL SECURITY` as belt-and-suspenders, so that even if the app accidentally connects as the owner, RLS still applies.

Superusers and `BYPASSRLS` roles are *never* subject to RLS, even with `FORCE`. `FORCE` closes the owner hole, not the superuser hole.

### 6.8 `BYPASSRLS` — the intentional escape hatch

```sql
CREATE ROLE analytics_reader BYPASSRLS LOGIN PASSWORD '...';
ALTER ROLE etl_service BYPASSRLS;
```

A role with `BYPASSRLS` ignores all RLS policies on all tables — as if RLS were disabled for that role's sessions. This is the correct mechanism for:

- **Batch/ETL jobs** that must see all tenants' rows to build cross-tenant aggregates or replicate data.
- **Superuser-adjacent maintenance** roles that need unfiltered access without being full superusers.
- **Backup/dump** processes (`pg_dump` run as a non-superuser needs `BYPASSRLS` to dump all rows; otherwise it dumps only visible rows — a classic "my backup is missing data" bug).

`BYPASSRLS` is a *role attribute*, not a per-table grant — it's all-or-nothing across every RLS table. Grant it sparingly and never to the interactive application role.

### 6.9 The multi-tenant `current_setting` pattern in full

The canonical SaaS pattern stores the current tenant in a **session (or transaction) configuration parameter** and references it from the policy:

```sql
-- Policy references a namespaced GUC (grand unified config) variable
CREATE POLICY tenant_isolation ON orders
  FOR ALL
  USING (tenant_id = current_setting('app.tenant_id')::bigint)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);
```

`current_setting('app.tenant_id')` reads a custom parameter. The `app.` prefix is required — PostgreSQL only allows custom GUCs that contain a dot (a namespace), so `app.tenant_id`, `app.user_id`, `app.role` are valid; a bare `tenant_id` is not.

Two forms of `current_setting` matter:
- `current_setting('app.tenant_id')` — raises an error if the parameter was never set. Safer for a policy: an unset tenant means "fail closed," not "see everything."
- `current_setting('app.tenant_id', true)` — the `true` (missing_ok) second argument returns NULL instead of erroring if unset. **Avoid this in policies** unless you deliberately want "unset ⇒ NULL ⇒ predicate is NULL ⇒ zero rows." A NULL comparison yields zero rows (fail-closed for reads) but can be surprising; the erroring form makes the misconfiguration loud.

The value is set per request (section 6.10 and section 11). The policy then automatically scopes every query on `orders` to the current tenant with no query change.

### 6.10 `SET` vs `SET LOCAL` — why transaction scope is mandatory under pooling

```sql
SET app.tenant_id = '42';        -- session scope: lasts until connection closes or reset
SET LOCAL app.tenant_id = '42';  -- transaction scope: reset at COMMIT/ROLLBACK
```

Under a connection pool (which every Node.js backend uses), physical connections are **reused across requests and across tenants**. If you use plain `SET`, the value `app.tenant_id = 42` **persists on that connection** after the request ends. The next request — possibly tenant 99 — checks out the same connection, and if *it* forgets to set the value, it inherits tenant 42's context and reads tenant 42's data. This is a severe cross-tenant leak.

`SET LOCAL` scopes the value to the **current transaction only**. At `COMMIT` or `ROLLBACK`, PostgreSQL automatically resets it. This is exactly what you want under pooling: the tenant context lives precisely as long as the request's transaction and cannot bleed into the next checkout of that connection.

The iron rule: **with a connection pool, always use `SET LOCAL` inside an explicit transaction, never plain `SET`.** The full Node.js wiring is in section 11.

### 6.11 Policies can reference `current_user`, roles, and subqueries

Policy predicates are arbitrary boolean expressions. Beyond session GUCs, they commonly use:

```sql
-- Role-based: a row is visible if the current DB role is its owner
USING (owner_role = current_user)

-- Membership via a lookup table (subquery in a policy — beware per-row cost)
USING (tenant_id IN (
  SELECT tenant_id FROM user_tenants
  WHERE user_id = current_setting('app.user_id')::bigint
))

-- Function-based (encapsulate complex logic; mark STABLE, and ideally leakproof-safe)
USING (can_access_order(id, current_setting('app.user_id')::bigint))
```

Subqueries and function calls in policies are powerful but are the primary source of RLS performance problems: a naive subquery-per-row policy runs the subquery for every candidate row. Mitigations (caching the membership set, marking functions `STABLE`, indexing) are in section 10.

### 6.12 Policies and `pg_policy` / `pg_policies`

Inspect existing policies via the catalog and the friendly view:

```sql
-- Human-readable
SELECT schemaname, tablename, policyname, permissive, roles, cmd, qual, with_check
FROM pg_policies
WHERE tablename = 'orders';

-- Is RLS enabled / forced on the table?
SELECT relname, relrowsecurity, relforcerowsecurity
FROM pg_class
WHERE relname = 'orders';
```

`qual` shows the `USING` expression; `with_check` shows the `WITH CHECK`; `permissive` is `PERMISSIVE`/`RESTRICTIVE`; `cmd` is the `FOR` command. This is the first thing to check when RLS "isn't working."

### 6.13 Interaction with views and functions

- **Views** run with the *view owner's* permissions by default and historically bypassed the querying user's RLS. Since PostgreSQL 15, `CREATE VIEW ... WITH (security_invoker = true)` makes the view evaluate RLS as the *invoking* user — essential if you build views over RLS-protected tables and want the caller's tenant context to apply.
- **`SECURITY DEFINER` functions** run as their owner; if that owner has `BYPASSRLS` or owns the table without `FORCE`, the function silently bypasses RLS. Audit `SECURITY DEFINER` functions carefully — they are a common RLS bypass.

### 6.14 What RLS does *not* protect

- **Column visibility**: RLS filters *rows*, not columns. Use `GRANT`/column privileges or views for column-level control.
- **Aggregate leakage**: A `COUNT(*)` still only counts visible rows, which is correct — but be aware that error messages, timing, and unique-constraint violations can leak the *existence* of hidden rows (e.g. inserting a duplicate key that belongs to a hidden row errors with a constraint violation, revealing the row exists). RLS is row-visibility, not a covert-channel-proof sandbox.
- **`TRUNCATE`, `DROP`, DDL**: governed by table privileges, not RLS.

---

## 7. EXPLAIN — RLS Predicate Injection in the Plan

The whole point of reading EXPLAIN on an RLS table is to confirm two things: (1) the policy predicate actually appears in the plan, and (2) it uses an index rather than forcing a Seq Scan.

### With an index on the policy column (the good case)

```sql
SET app.tenant_id = '42';
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_amount, status
FROM orders
WHERE status = 'completed';
-- Note: we did NOT write tenant_id in the query — RLS injects it.
```

```
Index Scan using orders_tenant_status_idx on orders
    (cost=0.43..312.10 rows=210 width=20)
    (actual time=0.03..0.71 rows=198 loops=1)
  Index Cond: ((tenant_id = 42) AND (status = 'completed'::text))
  Buffers: shared hit=64
Planning Time: 0.20 ms
Execution Time: 0.79 ms
```

**Reading it:**
- `Index Cond: ((tenant_id = 42) AND (status = 'completed'))` — the `tenant_id = 42` term is the **injected RLS policy**, even though we never typed it. `current_setting('app.tenant_id')::bigint` was resolved to the constant `42` at plan time (a STABLE expression), so it's a usable index condition.
- The composite index `orders_tenant_status_idx (tenant_id, status)` serves both the RLS predicate and the user's `status` predicate in one Index Scan — this is the ideal: RLS costs essentially nothing because it rides the same index the query already needed.
- Low `Buffers` and sub-millisecond execution confirm no full scan.

### Without a usable index (the bad case)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_amount FROM orders WHERE status = 'completed';
```

```
Seq Scan on orders  (cost=0.00..24500.00 rows=205 width=16)
                    (actual time=0.02..118.4 rows=198 loops=1)
  Filter: ((tenant_id = 42) AND (status = 'completed'::text))
  Rows Removed by Filter: 999802
  Buffers: shared hit=8200 read=4300
Planning Time: 0.18 ms
Execution Time: 121.9 ms
```

**Reading it:**
- The RLS predicate is now in `Filter`, not `Index Cond` — every one of the ~1M rows is read from the heap and the policy is evaluated per row. `Rows Removed by Filter: 999802` is the tell: the table was fully scanned and 99.98% of rows were discarded by the (mostly RLS) filter.
- 12,500 buffers touched, 122 ms — this single missing index makes *every query on the table* slow, because RLS forces the predicate onto every query. This is why "index the policy column" is the number-one RLS performance rule.

### Policy with a per-row subquery (the dangerous case)

```sql
-- Policy: USING (tenant_id IN (SELECT tenant_id FROM user_tenants WHERE user_id = current_setting('app.user_id')::bigint))
EXPLAIN (ANALYZE)
SELECT id FROM orders WHERE status = 'completed';
```

```
Seq Scan on orders  (cost=0.00..9999999.00 rows=100 width=8)
                    (actual time=0.05..842.0 rows=198 loops=1)
  Filter: ((status = 'completed') AND (SubPlan 1))
  Rows Removed by Filter: 999802
  SubPlan 1
    ->  Index Scan using user_tenants_user_idx on user_tenants
          Index Cond: (user_id = 55)
          (actual rows=1 loops=... )
Buffers: shared hit=...
Execution Time: 851.3 ms
```

**Reading it:**
- `SubPlan 1` under the Filter means the membership subquery is (potentially) re-evaluated per row. PostgreSQL may cache a truly correlated-free subquery as an `InitPlan`, but a policy that appears correlated can degrade to per-row execution.
- The fix (section 10): make the subquery an uncorrelated `InitPlan` (evaluate the tenant set once), or resolve tenant membership in application code and pass a concrete `app.tenant_id`, restoring the clean indexable equality of the good case.

### What to look for

| Symptom in plan | Meaning | Fix |
|-----------------|---------|-----|
| RLS term in `Index Cond` | Policy rides an index — ideal | nothing |
| RLS term in `Filter` + high `Rows Removed by Filter` | Full scan, per-row policy eval | index the policy column |
| `SubPlan` (not `InitPlan`) under Filter | Per-row policy subquery | rewrite to uncorrelated / resolve in app |
| `current_setting(...)` not folded to a constant | Non-STABLE usage blocks index | ensure STABLE context, cast correctly |

---

## 8. Query Examples

### Example 1 — Basic: Single-tenant isolation on one table

```sql
-- Setup: enable RLS and add one FOR ALL policy
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;   -- apply to owner too (belt-and-suspenders)

CREATE POLICY tenant_isolation ON orders
  FOR ALL                                                  -- governs select/insert/update/delete
  USING      (tenant_id = current_setting('app.tenant_id')::bigint)   -- read/target gate
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);  -- write gate

-- Per request:
BEGIN;
SET LOCAL app.tenant_id = '42';
SELECT id, total_amount FROM orders;   -- automatically only tenant 42's orders
COMMIT;
```

### Example 2 — Intermediate: Separate read and write rules + a restrictive invariant

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- READ: anyone in the tenant may see the tenant's orders
CREATE POLICY tenant_read ON orders
  AS PERMISSIVE FOR SELECT
  USING (tenant_id = current_setting('app.tenant_id')::bigint);

-- WRITE (insert/update): only within the tenant, and only your own rows
CREATE POLICY own_write ON orders
  AS PERMISSIVE FOR ALL
  USING      (tenant_id = current_setting('app.tenant_id')::bigint
              AND created_by = current_setting('app.user_id')::bigint)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint
              AND created_by = current_setting('app.user_id')::bigint);

-- INVARIANT (restrictive): soft-deleted rows are invisible to EVERYONE, always
CREATE POLICY hide_soft_deleted ON orders
  AS RESTRICTIVE FOR SELECT
  USING (deleted_at IS NULL);

-- Net SELECT visibility for a normal user session:
--   (tenant_read: tenant match)              -- permissive OR-group
--   AND (hide_soft_deleted: deleted_at IS NULL)  -- restrictive AND
```

### Example 3 — Production Grade: Multi-table tenancy with pooling-safe context

**Context:** SaaS with `orders` (50M rows across 4,000 tenants, largest tenant ~2M rows) and `order_items` (200M rows). App connects as non-owner `app_user`. Requirement: hard tenant isolation on both tables, admin-role delete, and index support so RLS is free.

```sql
-- ── Roles ──────────────────────────────────────────────
CREATE ROLE app_owner NOLOGIN;                         -- owns tables, runs migrations
CREATE ROLE app_user  LOGIN PASSWORD '...';            -- the app connects as this (NOT owner)
CREATE ROLE etl_reader LOGIN PASSWORD '...' BYPASSRLS; -- cross-tenant analytics/backup

-- ── Indexes that make the RLS predicate free ──────────
-- Composite: tenant_id FIRST so the injected equality is the leading column
CREATE INDEX orders_tenant_created_idx     ON orders     (tenant_id, created_at DESC);
CREATE INDEX order_items_tenant_order_idx  ON order_items(tenant_id, order_id);

-- ── RLS on orders ─────────────────────────────────────
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

CREATE POLICY orders_tenant ON orders
  AS PERMISSIVE FOR ALL TO app_user
  USING      (tenant_id = current_setting('app.tenant_id')::bigint)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);

CREATE POLICY orders_no_delete_by_default ON orders
  AS RESTRICTIVE FOR DELETE TO app_user
  USING (current_setting('app.role', true) = 'admin');   -- only admin sessions may delete

-- ── RLS on order_items (denormalized tenant_id for a direct, indexable predicate) ──
ALTER TABLE order_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE order_items FORCE ROW LEVEL SECURITY;

CREATE POLICY items_tenant ON order_items
  AS PERMISSIVE FOR ALL TO app_user
  USING      (tenant_id = current_setting('app.tenant_id')::bigint)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);

-- ── Per-request usage (pooling-safe) ──────────────────
BEGIN;
SET LOCAL app.tenant_id = '42';
SET LOCAL app.user_id   = '5501';
SET LOCAL app.role      = 'member';

SELECT o.id, o.total_amount, oi.product_id, oi.quantity
FROM orders o
JOIN order_items oi ON oi.order_id = o.id      -- BOTH tables independently RLS-scoped to tenant 42
WHERE o.created_at >= now() - interval '30 days'
ORDER BY o.created_at DESC
LIMIT 100;
COMMIT;
```

**Why `order_items` carries a denormalized `tenant_id`:** without it, the `order_items` policy would need a subquery back to `orders` (`order_id IN (SELECT id FROM orders WHERE tenant_id = ...)`) — a per-row subquery that defeats indexing. Denormalizing `tenant_id` onto `order_items` turns the policy into a plain indexable equality, keeping RLS free even at 200M rows. The `WITH CHECK` on the insert guarantees the denormalized value can't be forged.

**Performance expectation:** Both tables filter via `Index Cond: (tenant_id = 42 AND ...)` on the composite indexes; the join is planned exactly as if you had hand-written the tenant filters. Expected: single-digit milliseconds for the 100-row page.

```
Limit  (actual time=0.31..2.10 rows=100 loops=1)
  ->  Nested Loop  (actual time=0.30..2.01 rows=100 loops=1)
        ->  Index Scan using orders_tenant_created_idx on orders o
              Index Cond: ((tenant_id = 42) AND (created_at >= now() - '30 days'::interval))
        ->  Index Scan using order_items_tenant_order_idx on order_items oi
              Index Cond: ((tenant_id = 42) AND (order_id = o.id))
Execution Time: 2.2 ms
```

---

## 9. Wrong → Right Patterns

### Wrong 1: App connects as the table owner (RLS silently not applied)

```sql
-- Migration and app both connect as 'app_owner', which owns 'orders'
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
  FOR ALL USING (tenant_id = current_setting('app.tenant_id')::bigint);

-- App session as app_owner:
SET app.tenant_id = '42';
SELECT count(*) FROM orders;   -- returns ALL tenants' rows!
```

**Why it's wrong at the execution level:** the table owner is exempt from RLS by default. The rewriter checks `pg_class.relforcerowsecurity`; with `FORCE` off and the current role being the owner, it injects **no** policy qual. The query runs with zero RLS filtering. This is the most common RLS failure — "I enabled RLS but it does nothing" — and it leaks every tenant.

```sql
-- RIGHT: force RLS AND connect as a non-owner role
ALTER TABLE orders FORCE ROW LEVEL SECURITY;   -- apply even to the owner

-- and/or: app connects as a dedicated non-owner role
CREATE ROLE app_user LOGIN PASSWORD '...';
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO app_user;
-- app_user is NOT the owner, so RLS applies; FORCE makes it airtight either way
```

### Wrong 2: Plain `SET` under a connection pool (cross-tenant leak)

```sql
-- Request A (tenant 42) checks out a pooled connection:
SET app.tenant_id = '42';          -- SESSION scope — persists after the request!
SELECT ... FROM orders;
-- request A ends, connection returned to pool WITHOUT reset

-- Request B (tenant 99) checks out the SAME physical connection
-- ... and a code path forgets to set the tenant:
SELECT ... FROM orders;            -- inherits app.tenant_id = 42 → reads TENANT 42'S DATA
```

**Why it's wrong at the execution level:** `SET` (session scope) writes the GUC onto the physical backend and it survives until the connection closes or is explicitly reset. Pools reuse backends across requests/tenants, so the value bleeds forward. The RLS policy faithfully applies `tenant_id = 42` — to the *wrong* request. RLS did its job; the *context* was stale.

```sql
-- RIGHT: transaction-scoped context, auto-reset at COMMIT/ROLLBACK
BEGIN;
SET LOCAL app.tenant_id = '99';    -- lives only for THIS transaction
SELECT ... FROM orders;            -- correctly scoped to tenant 99
COMMIT;                            -- app.tenant_id automatically cleared
```

### Wrong 3: Omitting `WITH CHECK` on writes (tenants can plant rows elsewhere)

```sql
-- Policy provides only SELECT visibility, no write check
CREATE POLICY tenant_read ON orders
  FOR SELECT USING (tenant_id = current_setting('app.tenant_id')::bigint);
-- ...and a separate permissive INSERT policy with a too-broad check, or none:
CREATE POLICY tenant_insert ON orders
  FOR INSERT WITH CHECK (true);    -- allows ANY tenant_id!

-- Session is tenant 42:
INSERT INTO orders (tenant_id, total_amount) VALUES (99, 100);  -- succeeds!
-- Tenant 42 just planted a row in tenant 99's data (which 42 can't even see)
```

**Why it's wrong:** `WITH CHECK (true)` (or a missing insert policy combined with a permissive grant) means the new-row predicate is always satisfied. RLS validates visibility on read but never constrains the written `tenant_id`. The attacker can write into arbitrary tenants — data corruption and a DoS vector.

```sql
-- RIGHT: constrain the written row to the session's tenant
CREATE POLICY tenant_write ON orders
  FOR ALL
  USING      (tenant_id = current_setting('app.tenant_id')::bigint)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);
-- INSERT of tenant_id = 99 while session is 42 now ERRORs: violates RLS policy
```

### Wrong 4: Non-indexable policy predicate (every query goes Seq Scan)

```sql
-- Policy uses a per-row function that isn't STABLE / isn't indexable
CREATE POLICY tenant_isolation ON orders
  FOR ALL
  USING (lower(tenant_slug) = lower(current_setting('app.tenant_slug')));
-- lower(tenant_slug) is not directly indexable unless an expression index exists
```

**Why it's wrong at the execution level:** `lower(tenant_slug)` cannot use a plain B-tree on `tenant_slug`, so the RLS predicate lands in `Filter` and forces a Seq Scan on *every* query touching `orders` — the policy is applied to all queries. `Rows Removed by Filter` will be enormous.

```sql
-- RIGHT: policy on a plain, indexed, equality-friendly column
CREATE POLICY tenant_isolation ON orders
  FOR ALL
  USING      (tenant_id = current_setting('app.tenant_id')::bigint)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);
CREATE INDEX ON orders (tenant_id);
-- OR, if you must match on slug, add the matching expression index:
-- CREATE INDEX ON orders (lower(tenant_slug));
```

### Wrong 5: Expecting an error on a cross-tenant UPDATE/DELETE

```sql
-- Session is tenant 42; row 999 belongs to tenant 99
DELETE FROM orders WHERE id = 999;
-- Result: "DELETE 0" — no error. App logic assumes the row was deleted.
UPDATE orders SET status = 'void' WHERE id = 999;
-- Result: "UPDATE 0" — no error. App reports success.
```

**Why it's wrong:** the `USING` predicate hides row 999 from the statement's target set, so the DELETE/UPDATE matches zero rows. This is *not* an error — it's an empty match, identical to `WHERE id = <nonexistent>`. Application code that treats "statement succeeded" as "row was modified" will silently misbehave.

```sql
-- RIGHT: check the affected row count and treat 0 as not-found-or-forbidden
-- (in Node: result.rowCount === 0 → 404/403), or use RETURNING to confirm:
UPDATE orders SET status = 'void'
WHERE id = 999
RETURNING id;
-- rows.length === 0 → the row was invisible (wrong tenant) or absent; handle explicitly
```

---

## 10. Performance Profile

RLS adds **zero storage overhead** and **zero write-amplification** — it is pure predicate injection. Its entire performance story is: *is the injected predicate cheap and indexable?* Because the predicate is applied to **every query on the table**, a bad policy is a table-wide tax, and a good one is free.

### The governing rule

> An RLS policy costs the same as writing its predicate by hand in `WHERE`. Make the policy predicate a plain, indexed equality on a column whose value comes from a plan-time constant (a resolved `current_setting`), and RLS is effectively free.

### Scaling behavior

| Table size | Indexed equality policy (`tenant_id = 42`) | Non-indexed / functional policy | Per-row subquery policy |
|-----------|-------------------------------------------|--------------------------------|-------------------------|
| 1M rows | Index Scan, ~0.5–2 ms | Seq Scan, ~80–150 ms | Seq Scan + subplan, ~300 ms–1 s |
| 10M rows | Index Scan, ~1–3 ms (scales with *result*, not table) | Seq Scan, ~0.8–1.5 s | often multi-second |
| 100M rows | Index Scan, still ~1–5 ms per tenant page | Seq Scan, ~8–15 s (unusable) | pathological |

The key insight: with an indexable equality policy, query cost scales with the **number of rows the tenant actually has**, not the total table size — exactly as if you had hand-written the filter. Without an index, cost scales with **total table size on every query**, which is fatal past a few million rows.

### `current_setting` folding and STABLE-ness

`current_setting('app.tenant_id')::bigint` is a **STABLE** expression: constant within a single statement. The planner evaluates it once and folds it into a constant, making `tenant_id = <const>` an index condition (as seen in section 7's good plan). If you wrap it in something the planner treats as volatile, or compare it to a non-indexed expression, folding fails and you drop to a Filter. Always cast to the column's type (`::bigint`) so the comparison is a same-type equality the index supports.

### Leakproofness and lost optimizations

Because RLS tables are treated as security barriers, PostgreSQL will **not push a non-`leakproof` function below the RLS qual** — it must apply the security predicate first so user functions can't observe hidden rows. Practical effect: a `WHERE expensive_nonleakproof_fn(col)` clause that could otherwise be evaluated early is instead evaluated *after* RLS filtering. Usually this is fine (RLS filtering reduces the row count first), but for selective user predicates over huge tables it can cost you. Built-in operators like `=` on common types are leakproof, so ordinary equality predicates are unaffected.

### Optimization techniques specific to RLS

1. **Index the policy column, leading.** For `tenant_id = X`, put `tenant_id` as the **first** column of your composite indexes so the RLS equality is the leading key: `(tenant_id, created_at)`, `(tenant_id, status)`. Every query already carries the tenant equality, so tenant-leading indexes serve virtually all access paths.
2. **Denormalize the tenant key onto child tables.** As in Example 3, put `tenant_id` directly on `order_items` so its policy is a direct equality instead of a subquery back to `orders`. Storage cost is trivial; the planning win is enormous.
3. **Resolve membership in the app, not per-row in the policy.** Prefer a concrete `SET LOCAL app.tenant_id` over a policy subquery like `tenant_id IN (SELECT ... FROM user_tenants ...)`. If you must do membership in SQL, make it an uncorrelated `InitPlan` (evaluated once) — e.g. build the set from a session variable, not from a per-row correlation.
4. **Mark policy helper functions `STABLE` (or `IMMUTABLE`) and keep them cheap.** A `VOLATILE` function in a policy is re-evaluated per row and blocks constant folding.
5. **Partition-align with the tenant key when tenants are huge.** For 100M+ row tables with a few very large tenants, partitioning by `tenant_id` (or a hash of it) lets the RLS equality prune partitions in addition to using the index.
6. **Watch prepared-statement generic plans.** Under RLS, a cached generic plan built for one role/context can be suboptimal after context changes; if you see plan instability, consider `plan_cache_mode = force_custom_plan` for the affected statements.

### CPU and memory

RLS itself allocates no extra memory — it adds predicate evaluations (CPU) proportional to rows examined. With an index, rows examined ≈ rows returned, so the added CPU is negligible. Without an index, the added CPU is per-row-of-table and dominates.

---

## 11. Node.js Integration

The core requirement in Node.js is **transaction-scoped tenant context on the same pooled connection that runs the query**. You must acquire a client from the pool, `BEGIN`, `SET LOCAL` the context, run your queries, and `COMMIT` — all on that one client — then release it.

### 11.1 The per-request pattern (the safe core)

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
// IMPORTANT: DATABASE_URL must authenticate as a NON-OWNER role (e.g. app_user),
// and the tables must have FORCE ROW LEVEL SECURITY, so RLS actually applies.

/**
 * Runs `fn` inside a transaction with tenant context set via SET LOCAL.
 * The context is automatically cleared at COMMIT/ROLLBACK, so it can never
 * leak to the next checkout of this pooled connection.
 */
async function withTenant(tenantId, userId, fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    // set_config(name, value, is_local=true) is parameterizable — SET LOCAL is not.
    await client.query(
      `SELECT set_config('app.tenant_id', $1, true),
              set_config('app.user_id',   $2, true)`,
      [String(tenantId), String(userId)]
    );
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();   // returns to pool; SET LOCAL already reset by COMMIT/ROLLBACK
  }
}

// Usage in a request handler:
async function listRecentOrders(tenantId, userId) {
  return withTenant(tenantId, userId, async (client) => {
    const { rows } = await client.query(
      `SELECT id, total_amount, status, created_at
         FROM orders
        WHERE created_at >= now() - interval '30 days'
        ORDER BY created_at DESC
        LIMIT 100`   // no tenant filter here — RLS injects it
    );
    return rows;
  });
}
```

**Why `set_config(..., true)` instead of `SET LOCAL`:** `SET LOCAL` does not accept bind parameters (`$1`), so interpolating the tenant id into a `SET LOCAL` string risks SQL injection. `set_config('app.tenant_id', $1, true)` is the parameterized, injection-safe equivalent — the third argument `true` means `is_local` (transaction scope), exactly like `SET LOCAL`.

### 11.2 The critical mistake to avoid

```javascript
// WRONG: pool.query() gives you an ARBITRARY connection per call.
// The SET LOCAL and the SELECT can run on DIFFERENT connections,
// and SET LOCAL outside a transaction is a no-op / warning.
await pool.query(`SET LOCAL app.tenant_id = '42'`);  // runs on connection A, no txn
await pool.query(`SELECT * FROM orders`);            // may run on connection B — no context!
// Result: zero rows (fail-closed) or, worse, stale context from a prior SET.
```

Always pin context and queries to **one** `client` inside **one** transaction, as in 11.1.

### 11.3 Interpreting zero-row writes

```javascript
async function voidOrder(tenantId, userId, orderId) {
  return withTenant(tenantId, userId, async (client) => {
    const { rows } = await client.query(
      `UPDATE orders SET status = 'void'
        WHERE id = $1
        RETURNING id`,
      [orderId]
    );
    if (rows.length === 0) {
      // Row was invisible (belongs to another tenant) OR doesn't exist.
      // RLS makes these indistinguishable — treat as 404/403 deliberately.
      throw new NotFoundError(`Order ${orderId} not found`);
    }
    return rows[0];
  });
}
```

Because RLS turns cross-tenant targets into empty matches (section 9, Wrong 5), always check `rowCount`/`RETURNING` and never assume a write touched a row.

### 11.4 A guard against forgetting context

```javascript
// Optional belt-and-suspenders: a policy that fails closed when unset,
// plus an assertion in app code.
async function assertTenantSet(client) {
  const { rows } = await client.query(
    `SELECT current_setting('app.tenant_id', true) AS t`
  );
  if (!rows[0].t) throw new Error('Tenant context not set — refusing to query');
}
```

Because the policy uses `current_setting('app.tenant_id')::bigint` (the erroring form, not `, true`), a query with unset context will *error* rather than silently return everything — the database itself fails closed. The app-side assertion just surfaces the misconfiguration earlier and more clearly.

### 11.5 Do ORMs handle this?

No mainstream Node ORM sets RLS context automatically — you must wire the `set_config`/`SET LOCAL` yourself, and you must ensure the ORM runs your queries on the *same* transaction-bound connection. Every ORM below supports a "run inside this transaction/connection" primitive that you combine with a raw `set_config` call. Details in section 12.

---

## 12. ORM Comparison

RLS is a **schema/session** feature, not a query feature — so the ORM question is *not* "can it write RLS?" (that's DDL you run in a migration) but "**can it reliably set transaction-scoped tenant context on the same connection that runs my queries?**" That is where ORMs differ.

### Prisma

**Can Prisma do it?** — Partially. Prisma cannot express `SET LOCAL` in its query API, and its connection pooling historically made per-request session context hard. The supported approach is `$transaction` with interactive transactions plus `$executeRaw`.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

async function withTenant<T>(tenantId: bigint, fn: (tx: Prisma.TransactionClient) => Promise<T>) {
  return prisma.$transaction(async (tx) => {
    // All calls on `tx` share ONE connection/transaction — required for SET LOCAL.
    await tx.$executeRaw`SELECT set_config('app.tenant_id', ${String(tenantId)}, true)`;
    return fn(tx);
  });
}

// Usage:
const orders = await withTenant(42n, (tx) =>
  tx.order.findMany({ where: { createdAt: { gte: new Date(Date.now() - 30 * 864e5) } } })
);
```

**Where it breaks:** Prisma's own connection pool and, especially, external poolers (PgBouncer in *transaction* mode) can break session assumptions; you must keep everything inside `$transaction` so it's one connection. Prisma migrations can run the `ALTER TABLE ... ENABLE RLS` / `CREATE POLICY` DDL only via raw SQL migration files (`prisma migrate` with a hand-written `.sql`). Prisma has no model for policies.

**Verdict:** Workable with `$transaction` + `$executeRaw set_config`. Put all RLS DDL in raw SQL migrations. Never rely on `prisma.$queryRaw` outside a transaction for RLS-scoped reads.

---

### Drizzle ORM

**Can Drizzle do it?** — Yes, cleanly. Drizzle has first-class `transaction()` and, notably, built-in helpers for RLS in recent versions (`pgPolicy`, `crudPolicy`, and RLS-aware schema definitions).

```typescript
import { db } from './db';
import { orders } from './schema';
import { sql, gte } from 'drizzle-orm';

async function withTenant<T>(tenantId: bigint, fn: (tx: typeof db) => Promise<T>) {
  return db.transaction(async (tx) => {
    await tx.execute(sql`SELECT set_config('app.tenant_id', ${String(tenantId)}, true)`);
    return fn(tx as unknown as typeof db);
  });
}

const rows = await withTenant(42n, (tx) =>
  tx.select().from(orders).where(gte(orders.createdAt, new Date(Date.now() - 30 * 864e5)))
);

// Drizzle can also DECLARE policies in schema (generates the CREATE POLICY DDL):
// export const orders = pgTable('orders', {...}, (t) => [
//   pgPolicy('tenant_isolation', {
//     for: 'all',
//     using: sql`tenant_id = current_setting('app.tenant_id')::bigint`,
//     withCheck: sql`tenant_id = current_setting('app.tenant_id')::bigint`,
//   }),
// ]);
```

**Where it breaks:** the `sql` template is still needed for the `set_config` call and for policy predicates; type-safety stops at the SQL boundary. Otherwise excellent.

**Verdict:** Best-in-class Node support for RLS — it can both declare policies in the schema and set context per transaction. Preferred if you're choosing an ORM for an RLS-heavy app.

---

### Sequelize

**Can Sequelize do it?** — Yes, via managed transactions; you must run the `set_config` on the transaction's connection.

```javascript
const { sequelize, Order } = require('./models');

async function withTenant(tenantId, fn) {
  return sequelize.transaction(async (t) => {
    await sequelize.query(
      `SELECT set_config('app.tenant_id', $1, true)`,
      { bind: [String(tenantId)], transaction: t }   // ← MUST pass transaction: t
    );
    return fn(t);
  });
}

const orders = await withTenant(42, (t) =>
  Order.findAll({
    where: { createdAt: { [Op.gte]: new Date(Date.now() - 30 * 864e5) } },
    transaction: t,   // ← every query must ride the same transaction
  })
);
```

**Where it breaks:** the #1 bug is forgetting `{ transaction: t }` on *either* the `set_config` call or a query — that call silently runs on a different pooled connection with no tenant context. There is no compile-time protection; a single missed `transaction: t` is a leak or a zero-row surprise. Sequelize has no policy modeling — DDL goes in raw migrations.

**Verdict:** Works, but error-prone. Wrap it so every query in a request is forced through the transaction, and lint for stray queries missing `transaction:`.

---

### TypeORM

**Can TypeORM do it?** — Yes, via `QueryRunner` (an explicit single connection) or `dataSource.transaction`.

```typescript
import { AppDataSource } from './data-source';
import { Order } from './entities/Order';

async function withTenant<T>(tenantId: bigint, fn: (m: EntityManager) => Promise<T>) {
  const qr = AppDataSource.createQueryRunner();   // pins ONE connection
  await qr.connect();
  await qr.startTransaction();
  try {
    await qr.query(`SELECT set_config('app.tenant_id', $1, true)`, [String(tenantId)]);
    const res = await fn(qr.manager);              // use qr.manager for all queries
    await qr.commitTransaction();
    return res;
  } catch (e) {
    await qr.rollbackTransaction();
    throw e;
  } finally {
    await qr.release();
  }
}

const orders = await withTenant(42n, (m) =>
  m.getRepository(Order).find({ where: { /* ... */ } })
);
```

**Where it breaks:** you must route every repository/query through `qr.manager` (the query-runner's manager); calling the global repository escapes the transaction and the context. TypeORM models no policies — raw SQL migrations only.

**Verdict:** `QueryRunner` gives explicit connection control, which is exactly what RLS needs. Slightly verbose but reliable when you funnel all access through `qr.manager`.

---

### Knex.js

**Can Knex do it?** — Yes, and its transaction object *is* a bound connection, which fits RLS well.

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

async function withTenant(tenantId, fn) {
  return knex.transaction(async (trx) => {
    await trx.raw(`SELECT set_config('app.tenant_id', ?, true)`, [String(tenantId)]);
    return fn(trx);   // trx is a connection-bound query builder
  });
}

const orders = await withTenant(42, (trx) =>
  trx('orders')
    .where('created_at', '>=', new Date(Date.now() - 30 * 864e5))
    .orderBy('created_at', 'desc')
    .limit(100)   // no tenant filter — RLS supplies it
);
```

**Where it breaks:** the same discipline applies — every query must use `trx`, not the root `knex`. Knex's `?` placeholder in `.raw()` keeps the `set_config` value injection-safe. No policy modeling; DDL via `knex.raw` in migrations.

**Verdict:** Knex's transaction-as-connection model is a natural fit; the `set_config` + `trx` pattern is clean and explicit. Strong choice for RLS.

---

### ORM Summary Table

| ORM | Set context on same conn | Declare policies in schema | Main pitfall | Verdict |
|-----|:---:|:---:|--------------|---------|
| Prisma | `$transaction` + `$executeRaw` | No (raw SQL migration) | PgBouncer/session assumptions | Workable; keep all in `$transaction` |
| Drizzle | `transaction()` + `sql set_config` | **Yes** (`pgPolicy`) | `sql` template at boundary | Best RLS support |
| Sequelize | `transaction()` + `{transaction:t}` | No | forgetting `transaction: t` | Works, error-prone |
| TypeORM | `QueryRunner` + `qr.manager` | No | escaping via global repo | Reliable, verbose |
| Knex | `transaction()` + `trx` | No | using root `knex` not `trx` | Natural fit |

**Universal rule across all ORMs:** RLS DDL lives in raw SQL migrations; RLS *context* must be set with `set_config('app.tenant_id', $1, true)` on the exact transaction-bound connection that runs your queries. No ORM removes the need to be deliberate about this.

---

## 13. Practice Exercises

### Exercise 1 — Basic

You have a `payments` table with columns `(id, tenant_id, amount, status, created_at)`. Currently any authenticated role can read every payment.

1. Enable Row Level Security on `payments` and add a single policy so that a session can only read, insert, update, and delete payments belonging to the tenant identified by the session variable `app.tenant_id`.
2. Make sure inserting or updating a payment into a *different* tenant is rejected.
3. Ensure the policy also applies to the table owner.

```sql
-- Write your DDL here
```

Then: as a `psql` session, show the two commands you'd run to read tenant 7's payments.

---

### Exercise 2 — Medium (combines Topic 11 JOINs)

You have `orders(id, tenant_id, customer_id, status, created_at)` and `order_items(id, tenant_id, order_id, product_id, quantity, unit_price)`, both with RLS enabling tenant isolation via `app.tenant_id`.

Write the query (assuming context is already set for tenant 42) that returns, per `status`, the number of distinct orders and the total revenue (`quantity * unit_price`) for orders created in the last 90 days. Then answer: which columns must be indexed so that the injected RLS predicate on *both* tables uses an Index Scan rather than a Seq Scan, and why does `order_items` need its own `tenant_id` column instead of joining back to `orders` in the policy?

```sql
-- Write your query here
-- Write your CREATE INDEX statements here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A team enabled RLS on `orders` with:

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
  FOR ALL USING (tenant_id = current_setting('app.tenant_id')::bigint);
```

Their Node service connects using the `orders`-owning role and sets context with `pool.query("SET app.tenant_id = '" + tenantId + "'")` at the start of each request. In staging everything looks correct. In production under load, two incidents occur: (a) some requests occasionally return *another* tenant's orders, and (b) a security review finds a tenant can create rows with a foreign `tenant_id`.

1. Identify every distinct root cause (there are at least four).
2. Rewrite the DDL and the Node context-setting code to fix all of them.
3. Explain why the bug was invisible in staging but appeared under production load.

```sql
-- Write corrected DDL here
```
```javascript
// Write corrected Node.js context handling here
```

---

### Exercise 4 — Interview Simulation

You're asked, in a system design interview: *"We're building a multi-tenant SaaS on a single shared PostgreSQL database. Walk me through how you'd guarantee that tenant A can never read or write tenant B's data, even if a developer forgets a WHERE clause or an attacker finds a SQL injection. Cover the schema, the roles, the per-request wiring under a connection pool, how backups/analytics still see all tenants, and the performance implications at 100M rows."*

Write your complete answer as you'd speak it, including the exact DDL and the Node.js per-request pattern.

```sql
-- Write your supporting DDL here
```

---

## 14. Interview Questions

### Q1 — What is Row Level Security and how does it differ from filtering in application code?

**Junior answer:** "RLS lets you restrict which rows a user can see. Instead of writing `WHERE tenant_id = ?` in your queries, the database does it for you when you create a policy."

**Principal answer:** "RLS attaches row-visibility and row-writability predicates directly to a table via `CREATE POLICY`, and PostgreSQL injects those predicates at the *rewrite* stage of every statement touching the table — before planning. The critical difference from application-side filtering is *where the guarantee lives*. App-side `WHERE tenant_id = ?` is a promise every query must remember to keep; one forgotten clause, one raw query, one SQL injection, and isolation is gone. RLS moves the predicate into the table, so it's applied unconditionally — to queries you forgot to filter, to `$queryRaw`, even to a successful injection, even to an interactive `psql` session. It's the difference between security-by-convention and security-by-construction. Mechanically it's still just a predicate: it participates in index selection and cost estimation exactly like a hand-written WHERE, so a well-designed policy on an indexed column is free, while a bad one taxes every query on the table."

**Interviewer follow-up:** "You said 'even to the owner' — is that true by default?" *(Expected: no — the owner is exempt unless you set `FORCE ROW LEVEL SECURITY`; and superuser/`BYPASSRLS` always bypass. The safe pattern is to connect as a non-owner role AND set FORCE.)*

---

### Q2 — Explain `USING` vs `WITH CHECK`, and what happens on an UPDATE.

**Junior answer:** "`USING` is for reads and `WITH CHECK` is for writes. UPDATE uses both."

**Principal answer:** "`USING` is the *visibility/target* predicate — evaluated against existing rows: it decides which rows a SELECT returns and which rows an UPDATE/DELETE is even allowed to touch. `WITH CHECK` is the *write* predicate — evaluated against the *new row image* for INSERT and UPDATE, deciding whether the resulting row is permitted. They fire at different points, which is why UPDATE runs both: `USING` against the *old* row (may I target it?) and `WITH CHECK` against the *new* row (is the result allowed?). That's what stops a tenant from re-assigning a row's `tenant_id` to someone else — the new image fails `WITH CHECK`. There's also an asymmetry in failure behavior: a `USING` miss *silently hides/omits* the row, while a `WITH CHECK` miss *raises* `new row violates row-level security policy` and rolls back. And if you omit `WITH CHECK` on a policy that needs one, PostgreSQL reuses the `USING` expression as the check."

**Interviewer follow-up:** "A `DELETE ... WHERE id = 999` for a row in another tenant — error or no error?" *(Expected: no error, `DELETE 0`; the `USING` predicate hid the row from the target set, so it's an empty match. App code must treat rowCount 0 as not-found-or-forbidden.)*

---

### Q3 — PERMISSIVE vs RESTRICTIVE: how do multiple policies combine?

**Junior answer:** "Permissive allows things and restrictive blocks things. If you have several, they all apply."

**Principal answer:** "The combination is a fixed boolean formula: `(OR of all PERMISSIVE predicates) AND (AND of all RESTRICTIVE predicates)`. Permissive policies are OR-ed — each one *widens* access by adding another set of rows you may see; that's the default. Restrictive policies are AND-ed on top — each one *narrows* access with an additional condition every row must also satisfy. The subtle trap is that restrictive policies can't grant anything: if you have only restrictive policies and no permissive one, the permissive OR-term is empty and defaults to FALSE, so `FALSE AND anything` = nothing visible. You need at least one permissive policy to establish a baseline of access, then restrictive policies to carve out invariants like 'never show soft-deleted rows' or 'never show rows outside the active window' — invariants that must hold regardless of how access was granted."

**Interviewer follow-up:** "Give me a concrete case where you'd reach for RESTRICTIVE." *(Expected: a global invariant such as `deleted_at IS NULL` or a compliance rule like 'PII rows require an elevated `app.clearance`' that must apply on top of every permissive grant, including future ones.)*

---

### Q4 — Under a connection pool, what's the correct way to set tenant context, and why is the obvious way dangerous?

**Junior answer:** "Run `SET app.tenant_id = '42'` at the start of the request."

**Principal answer:** "The obvious `SET` is session-scoped, and that's the danger. Poolers reuse the same physical backend across requests and tenants; a session `SET` persists on the connection after the request ends, so the next request that checks out that connection — possibly a different tenant — inherits the stale value, and if it forgets to set its own, it reads the previous tenant's data. The RLS policy applies faithfully; it's just applied with the wrong context. The correct approach is transaction-scoped: `SET LOCAL` (or, parameter-safe, `set_config('app.tenant_id', $1, true)`) inside an explicit `BEGIN`/`COMMIT`, run on the *same* pooled client that runs the queries. Transaction scope means the value is automatically cleared at COMMIT/ROLLBACK, so it can't bleed into the next checkout. In Node that means `pool.connect()` → `BEGIN` → `set_config(...)` → queries → `COMMIT` → `release()`, never bare `pool.query('SET ...')` followed by `pool.query('SELECT ...')` since those two can land on different connections."

**Interviewer follow-up:** "Why `set_config($1, true)` rather than `SET LOCAL app.tenant_id = ...`?" *(Expected: `SET LOCAL` can't take bind parameters, so string-interpolating the tenant id risks injection; `set_config(name, value, is_local=true)` is the parameterized equivalent with identical transaction scope.)*

---

### Q5 — Your RLS query is doing a Seq Scan on a 50M-row table on every request. Diagnose and fix.

**Junior answer:** "Add an index on the table."

**Principal answer:** "First confirm it's the policy predicate causing it: `EXPLAIN (ANALYZE, BUFFERS)` should show the RLS term. If it's in `Filter` with a large `Rows Removed by Filter`, the policy predicate isn't indexable. The usual culprits: (1) the policy compares a function of the column (`lower(tenant_slug) = ...`) instead of a plain column, so no B-tree applies — fix by policing on a plain `tenant_id` column with an index, or add the matching expression index; (2) the policy uses a per-row subquery (`tenant_id IN (SELECT ... FROM user_tenants ...)`) that shows up as a `SubPlan` — fix by denormalizing `tenant_id` onto the table and resolving membership in the app so the policy becomes a plain equality; (3) the index exists but doesn't lead with `tenant_id`, so the injected equality can't drive it — rebuild composite indexes as `(tenant_id, ...)`. The goal is to get the RLS term into `Index Cond` so query cost scales with the tenant's row count, not the whole table. Because RLS applies to every query on the table, this one fix speeds up the entire application surface at once."

**Interviewer follow-up:** "Does leakproofness affect this?" *(Expected: RLS tables are security barriers, so PostgreSQL won't push non-leakproof user functions below the RLS qual; ordinary equality operators are leakproof so unaffected, but an expensive non-leakproof `WHERE` predicate may be forced to run after RLS filtering.)*

---

## 15. Mental Model Checkpoint

1. You `ALTER TABLE orders ENABLE ROW LEVEL SECURITY` but write no policies, then query as a non-owner role. How many rows do you get, and why? How does the answer change if you query as the table's owner?

2. A session sets `app.tenant_id = '42'`. A `DELETE FROM orders WHERE id = 999` runs, where row 999 belongs to tenant 99. What is the outcome — error, or `DELETE 0`? Which clause (`USING` or `WITH CHECK`) is responsible, and what must application code do with the result?

3. You have one PERMISSIVE policy (`tenant_id = X`) and one RESTRICTIVE policy (`deleted_at IS NULL`). Write the exact boolean expression the rewriter effectively appends to a SELECT. Now remove the permissive policy, leaving only the restrictive one — what does a SELECT return, and why?

4. Explain precisely why plain `SET app.tenant_id` is unsafe under a connection pool but `SET LOCAL` (or `set_config(..., true)`) is safe. Walk through two requests sharing one physical connection.

5. Your application connects as the role that *owns* the tables, and RLS "isn't doing anything." What are the two independent fixes, and which one would you keep even after applying the other?

6. Why does putting `tenant_id` directly on `order_items` (rather than joining back to `orders` inside the policy) matter for performance? Describe what the EXPLAIN plan looks like in each case.

7. A tenant runs `UPDATE orders SET tenant_id = 99 WHERE id = <their own row>`. With a `FOR ALL` policy that has both `USING (tenant_id = current)` and `WITH CHECK (tenant_id = current)`, what happens, and which clause stops it?

---

## 16. Quick Reference Card

```sql
-- ENABLE (default-deny for non-owners once on)
ALTER TABLE t ENABLE ROW LEVEL SECURITY;
ALTER TABLE t FORCE  ROW LEVEL SECURITY;   -- apply to the OWNER too (close the owner bypass)

-- POLICY skeleton
CREATE POLICY name ON t
  AS { PERMISSIVE | RESTRICTIVE }          -- PERMISSIVE(default)=OR-combined; RESTRICTIVE=AND-combined
  FOR { ALL | SELECT | INSERT | UPDATE | DELETE }
  TO role[, ...]                           -- default PUBLIC
  USING      (read/visibility predicate)   -- SELECT + old row of UPDATE/DELETE
  WITH CHECK (write predicate);            -- new row of INSERT/UPDATE

-- Clause applicability
--   SELECT → USING only          INSERT → WITH CHECK only
--   UPDATE → USING (old) + WITH CHECK (new)    DELETE → USING only
--   omit WITH CHECK → USING is reused as the check

-- Combination rule
--   allowed = (OR of PERMISSIVE USING) AND (AND of RESTRICTIVE USING)
--   only-restrictive, no-permissive ⇒ NOTHING visible

-- Failure behavior
--   USING miss      → row silently hidden / 0 rows matched (no error)
--   WITH CHECK miss → ERROR: new row violates row-level security policy (rollback)

-- Multi-tenant policy (canonical)
CREATE POLICY tenant_isolation ON orders
  FOR ALL
  USING      (tenant_id = current_setting('app.tenant_id')::bigint)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);
CREATE INDEX ON orders (tenant_id, created_at);   -- tenant_id LEADING → RLS rides the index

-- Session context (POOLING-SAFE): transaction scope only
BEGIN;
SET LOCAL app.tenant_id = '42';                   -- or, parameter-safe:
SELECT set_config('app.tenant_id', $1, true);     -- true = is_local (txn scope)
-- ... queries ...
COMMIT;                                           -- context auto-cleared

-- Bypass mechanisms
CREATE ROLE etl BYPASSRLS;                         -- ignores ALL policies (ETL/backup/pg_dump)
-- superusers and BYPASSRLS roles are NEVER subject to RLS, even with FORCE

-- Inspect
SELECT * FROM pg_policies WHERE tablename = 'orders';
SELECT relrowsecurity, relforcerowsecurity FROM pg_class WHERE relname = 'orders';
```

**Performance rules of thumb**
- Policy = injected predicate applied to EVERY query on the table → make it a plain, indexed equality.
- Put `tenant_id` FIRST in composite indexes so the RLS equality is the leading key.
- Denormalize the tenant key onto child tables; avoid per-row subquery policies (`SubPlan` in EXPLAIN).
- Indexed policy ⇒ cost scales with the tenant's rows; unindexed policy ⇒ cost scales with the whole table, on every query.
- `current_setting(...)::type` is STABLE and folds to a constant → indexable; keep policy functions STABLE and cheap.

**Interview one-liners**
- "RLS is the WHERE clause welded to the table — un-forgettable, applied at rewrite time to every statement."
- "`USING` = which rows you can see/target; `WITH CHECK` = what rows you're allowed to write."
- "PERMISSIVE = OR (widens), RESTRICTIVE = AND (narrows); restrictive-only means nobody sees anything."
- "Owner is exempt by default — use `FORCE ROW LEVEL SECURITY` and connect as a non-owner role."
- "Under a pool, `SET LOCAL`/`set_config(...,true)` in a transaction, never plain `SET` — or you leak tenants."
- "Index the policy column, leading — RLS is only free when its predicate rides an index."

---

## Connected Topics

- **Topic 71 — Schema Design Patterns** (previous): the multi-tenant schema choices (shared-schema + `tenant_id`, schema-per-tenant, database-per-tenant) that RLS secures; RLS makes the cheapest shared-schema model safe.
- **Topic 73 — Essential Extensions** (next): extensions like `pg_stat_statements` (to spot RLS-induced Seq Scans across the app) and how some extensions interact with policy-protected tables.
- **Topic 07 — NULL in Depth**: three-valued logic is why a `USING` predicate that evaluates to NULL hides the row — RLS keeps only TRUE, never NULL.
- **Topic 11 — INNER JOIN in Depth**: RLS is applied per-table in a join; each table's policy filters its own rows before the join can combine them — a join cannot smuggle a hidden row through.
- **Topic 12 — LEFT/RIGHT JOIN**: the ON-vs-WHERE placement lesson mirrors RLS's `USING`-vs-`WITH CHECK` distinction — *where* in the pipeline a predicate fires changes its meaning.
- **Topic — MVCC & Visibility**: RLS predicates are evaluated per-row per-snapshot; a tuple invisible to your snapshot is never even offered to the policy.
- **Topic — Query Planner & EXPLAIN**: RLS predicate injection, `Index Cond` vs `Filter`, and security-barrier / leakproofness push-down restrictions all live in the planner.
- **Topic — Roles & Privileges**: `GRANT`, table ownership, `BYPASSRLS`, and `FORCE ROW LEVEL SECURITY` together determine whether a policy is actually enforced for a given session.
- **Topic — Connection Pooling**: the `SET LOCAL` / transaction-scoped-context requirement is a direct consequence of pooled connection reuse across tenants.
