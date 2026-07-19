# Topic 60 — Bulk Operations
### SQL Mastery Curriculum — Phase 9: Advanced SQL Patterns

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you work in the mailroom of a huge office. Two ways to deliver 10,000 letters:

**The one-at-a-time way**: You pick up a single letter, walk to the elevator, ride to the right floor, find the desk, drop the letter, ride back down, and repeat. 10,000 round trips. Each trip is short, but the walking, waiting for the elevator, and riding up and down dominate. The actual "drop the letter" part is trivial — the *overhead of the trip* is what kills you.

**The bulk way**: You load all 10,000 letters onto a cart, take the elevator once, and walk floor by floor dropping stacks. One trip. The letters are the same; the *number of trips* collapsed from 10,000 to 1.

That is the entire story of bulk operations in SQL. A single-row `INSERT` is one letter, one trip: parse the statement, plan it, start a transaction, write to the WAL, flush, acknowledge over the network — then throw all that setup away and do it again for the next row. A **bulk** operation loads the whole cart at once: one parse, one plan, one WAL batch, one network round trip, one transaction. The rows written are identical. What collapses is the *per-row overhead* — and at scale, the overhead is 95% of the cost.

The mailroom has a second lesson. If a colleague insists on re-sorting the entire mail sorting rack every single time you drop one letter (rebuilding an index on every row), you want to say: "Stop re-sorting after each letter. Let me drop all 10,000, *then* sort the rack once." That is what dropping indexes before a big load and rebuilding after achieves.

Bulk operations are not a different kind of work. They are the same work with the trips, the sorting, and the paperwork amortized.

---

## 2. Connection to SQL Internals

Bulk operations touch nearly every internal subsystem of PostgreSQL, which is exactly why they are worth mastering. Each single-row write triggers a cascade; batching amortizes each stage.

**1. The WAL (Write-Ahead Log).** Every data modification writes a WAL record *before* the change is considered durable. Under the default `synchronous_commit = on`, each `COMMIT` forces an `fsync()` of the WAL to disk — a physical disk flush that can take 1–10ms on spinning disks, hundreds of microseconds on SSD. 10,000 single-row auto-committed inserts = 10,000 fsyncs. 10,000 rows in one transaction = **one** fsync. This single fact accounts for most of the speedup.

**2. The MVCC heap.** Every inserted row becomes a new heap tuple stamped with the inserting transaction's `xmin`. `UPDATE` does not overwrite — it writes a *new* tuple version and marks the old one dead (`xmax`), leaving bloat that `VACUUM` must later reclaim. Bulk `UPDATE` of 10M rows creates 10M dead tuples. This is why bulk updates and deletes interact heavily with autovacuum (Topic on VACUUM) and can double table size transiently.

**3. Statement parsing and planning.** Each distinct SQL string is parsed, rewritten, and planned. 10,000 separate `INSERT` statements = 10,000 parse+plan cycles. One multi-row `INSERT` or one `COPY` = one cycle. Prepared statements amortize parse+plan across executions but still pay per-execution protocol overhead.

**4. The buffer pool (`shared_buffers`).** Rows are written into 8KB pages in shared memory, marked dirty, and flushed lazily by the background writer / checkpointer. Bulk loads dirty pages fast; if they outrun the checkpointer you get checkpoint spikes. `COPY` uses an optimization: for a table created or truncated in the same transaction, it can skip WAL entirely (`wal_level = minimal`) and write pages directly.

**5. Index maintenance.** Every index on the table must be updated for every inserted/updated row — a B-tree descent and possible page split per index per row. With 5 indexes, a bulk insert does 5× the index work of the heap write. This is why the fastest bulk loads drop indexes, load the heap, and rebuild indexes in bulk (a sorted bottom-up build is far cheaper than N random insertions).

**6. The network protocol.** Each `pool.query()` is a client-server round trip. `COPY FROM STDIN` streams rows in a continuous pipe, eliminating per-row protocol overhead — it is the fastest possible ingestion path in the wire protocol.

**7. Lock and trigger machinery.** Row-level triggers (`FOR EACH ROW`) fire once per row regardless of batching. `COPY` fires row triggers too, but statement-level triggers fire once. Foreign-key checks are per-row constraint triggers — deferring them to commit batches the validation.

Bulk operations are the discipline of moving work from *per-row* to *per-batch* across all seven subsystems at once.

---

## 3. Logical Execution Order Context

Bulk operations are DML (`INSERT`, `UPDATE`, `DELETE`), not queries, so the `FROM→WHERE→GROUP BY→…→ORDER BY→LIMIT` pipeline applies only to the *source* of the data being written, not to the write itself. But the ordering that matters for bulk work is a different pipeline — the **write pipeline**:

```
1. Parse & plan the DML statement
2. Evaluate the row source
     - INSERT ... SELECT: the SELECT runs the full FROM→…→ORDER BY pipeline
     - INSERT ... VALUES:  the VALUES list is materialized
     - COPY:               rows are read from the stream, no planning per row
     - UPDATE ... FROM:    the join (target ⋈ source) is planned like a SELECT
     - DELETE ... USING:   the join (target ⋈ using) is planned like a SELECT
3. For each result row:
     a. BEFORE ROW triggers fire (can modify or skip the row)
     b. The heap tuple is written / updated / marked dead
     c. Index entries are inserted / updated
     d. AFTER ROW triggers fire
     e. Constraint (FK, CHECK) triggers fire — unless DEFERRED
4. AFTER STATEMENT triggers fire (once)
5. On COMMIT: deferred constraints validated, WAL flushed (fsync), locks released
```

Three consequences you must internalize:

- **`UPDATE ... FROM` and `DELETE ... USING` are planned like joins.** The target table is joined to the source; the planner picks Nested Loop / Hash / Merge exactly as in Topic 11. A bulk update's speed is a *join* speed. Index the join column on the source.

- **`RETURNING` runs after the write**, projecting the final post-write row values (including defaults and trigger modifications), and materializes them to send back to the client — a factor when returning millions of rows.

- **`ON CONFLICT` (upsert, from Topic 59) is evaluated at index-insertion time (step 3c).** The conflict is detected when the unique index insert fails, then the `DO UPDATE`/`DO NOTHING` path is taken. This is why `ON CONFLICT` requires a unique or exclusion constraint — it hooks the index machinery.

The mental model: for a query, "what runs first" is about `FROM` vs `WHERE`. For a bulk write, "what runs first" is about *the row source* vs *the per-row write cascade* vs *the commit*.

---

## 4. What Is a Bulk Operation?

A **bulk operation** is any DML statement that inserts, updates, or deletes many rows in a single statement (or a small number of statements), amortizing per-statement and per-transaction overhead across all the rows. The canonical forms are multi-row `INSERT`, `COPY`, `UPDATE ... FROM`, `DELETE ... USING`, and `unnest()`-driven set-based writes.

### Multi-row INSERT

```sql
INSERT INTO products (sku, name, price, category_id)
VALUES
  ('SKU-001', 'Widget',  9.99, 3),   -- ┐
  ('SKU-002', 'Gadget', 19.99, 3),   -- │ one statement, one parse/plan,
  ('SKU-003', 'Gizmo',  29.99, 5)    -- ┘ one WAL batch, one round trip
RETURNING id, sku;                    -- ← optional: get generated ids back
```

Annotated:

```
INSERT INTO products              ← target table
  (sku, name, price, category_id) ← explicit column list (always specify — never rely on positional)
VALUES
  (...), (...), (...)             ← multiple row tuples in ONE VALUES clause = one statement
RETURNING id, sku;               ← projects post-insert values (serial ids, defaults) back to client
```

### COPY — the high-throughput bulk loader

```sql
COPY products (sku, name, price, category_id)  -- target + column list
FROM STDIN                                       -- read rows from the client stream
WITH (FORMAT csv, HEADER true, DELIMITER ',');   -- format options
-- ...CSV data streamed here...
```

Annotated:

```
COPY products (col, ...)   ← target table and columns
FROM STDIN                 ← source: STDIN (client stream) | 'filename' (server-side file) | PROGRAM 'cmd'
WITH (
  FORMAT csv,              ← csv | text (default, tab-delimited) | binary
  HEADER true,             ← skip/emit a header line (csv/text)
  DELIMITER ',',           ← field separator
  NULL '\N',               ← string that represents SQL NULL
  QUOTE '"',               ← quote character (csv)
  FORCE_NOT_NULL (col),    ← treat empty quoted string as '' not NULL
  FREEZE true              ← mark tuples frozen (skip later vacuum freeze; same-txn-created table only)
);
```

### UPDATE ... FROM — update with a join

```sql
UPDATE order_items oi                    -- target table (aliased)
SET unit_price = p.price,                -- SET uses columns from the joined source
    updated_at = NOW()
FROM products p                          -- the source table to join against
WHERE oi.product_id = p.id               -- the JOIN condition (and any filter)
  AND oi.unit_price <> p.price;          -- only touch rows that actually differ
```

Annotated:

```
UPDATE order_items oi   ← target; each matching target row is updated once
SET col = expr          ← expr may reference the FROM tables (this is the "join")
FROM products p         ← source table(s); like a join's right side
WHERE join_cond         ← MUST include the join predicate, else Cartesian product (danger!)
  AND filter_cond;      ← extra filters restrict which rows are touched
```

### DELETE ... USING — delete with a join

```sql
DELETE FROM sessions s                    -- target table
USING users u                             -- source table to join against
WHERE s.user_id = u.id                    -- join condition
  AND u.status = 'deleted';               -- filter: delete sessions of deleted users
```

### unnest() — arrays into rows for set-based writes

```sql
-- Turn parallel arrays (one round trip) into a rowset, then INSERT/UPDATE/upsert
INSERT INTO products (sku, name, price)
SELECT * FROM unnest(
  $1::text[],        -- array of skus
  $2::text[],        -- array of names
  $3::numeric[]      -- array of prices
) AS t(sku, name, price)
ON CONFLICT (sku) DO UPDATE SET price = EXCLUDED.price;
```

`unnest()` expands N parallel arrays into a table of N columns, letting you send an entire batch as a handful of array parameters — the key to parameterized bulk upserts from Node.js.

---

## 5. Why Bulk Operations Mastery Matters in Production

1. **Ingestion throughput is often the whole system.** ETL pipelines, event ingestion, CSV imports, data migrations, backfills, and analytics loads live or die by write throughput. The gap between naive and bulk is not 2×; it is routinely **50–500×**. A nightly job that takes 6 hours with per-row inserts finishes in 3 minutes with `COPY`. That difference decides whether the batch window is met.

2. **The fsync wall is real and unforgiving.** With `synchronous_commit = on` and auto-commit, each insert costs one disk flush. On a database limited to ~1,000 fsyncs/second, that is a hard ceiling of ~1,000 rows/second no matter how fast your CPU is. Batching 1,000 rows per transaction turns that into ~1,000,000 rows/second of the same hardware. Engineers who do not understand this blame the disk, the network, or PostgreSQL — never the missing `BEGIN`.

3. **The N+1 write problem.** The read-side N+1 (Topic 18) has a write-side twin: a loop in application code issuing one `INSERT` per iteration. It looks innocent in a code review and destroys throughput in production. Recognizing it and collapsing it into one bulk statement is a core senior skill.

4. **Bulk updates and deletes create bloat and lock contention.** A `DELETE FROM events WHERE created_at < now() - interval '90 days'` touching 50M rows holds locks, generates 50M dead tuples, bloats the table, and can block autovacuum or replication. Knowing to *batch* the delete (delete in chunks of 10K in separate transactions) is what separates a maintenance job that runs cleanly from one that pages the on-call at 3am.

5. **Index and trigger overhead is multiplicative.** Loading 100M rows into a table with 6 indexes and a per-row audit trigger does 100M × (6 index updates + 1 trigger call + 1 heap write). Dropping indexes and disabling non-essential triggers for the load, then rebuilding, is the difference between a 4-hour load and a 20-minute one. This is standard practice for initial data loads and large migrations.

6. **Transaction sizing is a Goldilocks problem.** Too small (1 row/txn) and fsync dominates. Too large (10M rows/txn) and you hold locks forever, bloat the WAL, risk a single failure rolling back hours of work, and can exhaust memory on `RETURNING`. The senior engineer knows the sweet spot is typically **1,000–10,000 rows per transaction** and can explain why.

Without bulk mastery, the symptoms are always the same: "the import is too slow," "the migration timed out," "the delete locked the table," "the database CPU is fine but throughput is terrible." Every one traces back to per-row overhead that should have been amortized.

---

## 6. Deep Technical Content

### 6.1 Multi-row INSERT — mechanics and limits

A multi-row `INSERT ... VALUES` collapses N rows into one statement. The savings:

- **One parse/plan** instead of N.
- **One network round trip** instead of N.
- **One transaction** (in auto-commit mode, the whole statement is one implicit transaction → one fsync) instead of N.

```sql
-- 1 statement inserts 4 rows, 1 fsync, 1 round trip
INSERT INTO categories (name, parent_id) VALUES
  ('Electronics', NULL),
  ('Laptops',      1),
  ('Phones',       1),
  ('Accessories',  1);
```

**Limits and gotchas:**

- **Parameter limit.** The PostgreSQL wire protocol caps bound parameters at **65,535** ($1…$65535) per statement. With a 10-column table, that is a hard ceiling of `floor(65535/10) = 6553` rows per parameterized multi-row insert. Exceed it and you get `bind message has N parameter formats but M parameters` or a protocol error. Batch accordingly.

- **Statement size.** Even with literal (non-parameterized) values, extremely large statements strain the parser and memory. Practical batch sizes are 500–5,000 rows.

- **All rows share one type resolution.** Every tuple must be assignment-compatible with the column types; a single bad row fails the entire statement (all-or-nothing within the statement's transaction).

- **`DEFAULT` keyword** can be used per-cell: `VALUES ('a', DEFAULT, NOW())` uses the column default for the middle column.

**Multi-row INSERT vs COPY:** multi-row `INSERT` is convenient, fully supports `RETURNING`, `ON CONFLICT`, and expressions in `VALUES`. But it is still slower than `COPY` because it goes through the full executor per row and pays parameter binding. Rule of thumb: use multi-row `INSERT` up to a few thousand rows; use `COPY` for tens of thousands and up.

### 6.2 COPY — the fast path

`COPY` is PostgreSQL's bulk I/O command. It bypasses much of the per-row executor overhead and reads/writes rows in a tight loop.

**Directions:**
- `COPY table FROM 'file' / STDIN / PROGRAM` — load into the table.
- `COPY table TO 'file' / STDOUT / PROGRAM` — export from the table.
- `COPY (SELECT ...) TO ...` — export the result of a query (not valid for FROM).

**Server-side vs client-side:**
- `COPY ... FROM '/path/file.csv'` reads a file **on the database server's filesystem**, and requires superuser or the `pg_read_server_files` role. Rarely usable in managed cloud databases.
- `COPY ... FROM STDIN` reads from the **client connection stream**. This is what application code and `psql`'s `\copy` use. No special privileges beyond `INSERT` on the table. This is the portable, production path.

**`psql \copy`** is a client-side wrapper: `\copy products FROM 'local.csv' CSV HEADER` runs `COPY ... FROM STDIN` under the hood and streams your local file. Use it for ad-hoc loads.

**Formats:**
- `text` (default): tab-delimited, `\N` for NULL, backslash escapes.
- `csv`: comma-delimited, quoting rules, `HEADER` support, most common for imports.
- `binary`: PostgreSQL's on-wire binary representation. Fastest (no text parsing, no float rounding) but not human-readable and version/type-sensitive. Used by high-performance loaders.

**Why COPY is fast:**
1. No per-row parse/plan — one command, streamed rows.
2. Rows are fed directly into the tuple-forming and heap-insert path.
3. It batches WAL records efficiently.
4. **WAL-skip optimization**: if the table was created or truncated in the *same transaction* and `wal_level = minimal`, `COPY` skips WAL for the table's data entirely (it will be fsync'd once at commit). On cloud providers `wal_level` is usually `replica` or `logical`, so this does not apply — but the general executor-bypass speedup still does.
5. `FREEZE true` marks the loaded tuples as frozen so a later `VACUUM` need not rewrite them (only valid when the table was created/truncated in the same transaction).

**COPY still runs constraints and triggers:** `NOT NULL`, `CHECK`, `UNIQUE`, foreign keys, and **row-level triggers all fire** during `COPY`. Only the executor overhead is bypassed, not correctness. A per-row `BEFORE INSERT` trigger will fire 10M times during a 10M-row `COPY` and can dominate the load time.

**COPY error handling:** historically `COPY` was all-or-nothing — one malformed row aborts the whole load. PostgreSQL 17 added `ON_ERROR ignore` and `LOG_VERBOSITY` to skip and report bad rows:

```sql
COPY products FROM STDIN WITH (FORMAT csv, HEADER true, ON_ERROR ignore, LOG_VERBOSITY verbose);
```

On older versions, the pattern is: `COPY` into a staging table with all-text columns, then `INSERT ... SELECT` with validation/casting into the real table so bad rows can be filtered.

### 6.3 UPDATE ... FROM — bulk update via join

Standard SQL has no clean "update with join"; PostgreSQL provides `UPDATE ... FROM`. The target is joined to one or more source tables, and each matching target row is updated using values from the source.

```sql
-- Denormalize: copy the current product price onto each order_item
UPDATE order_items oi
SET unit_price = p.price
FROM products p
WHERE oi.product_id = p.id
  AND oi.status = 'pending';   -- only update not-yet-finalized items
```

**Critical gotcha — the missing join condition = Cartesian catastrophe.** If you forget the `WHERE oi.product_id = p.id`, PostgreSQL joins *every* target row to *every* source row. Each target row is updated once (the standard says the result is implementation-defined when a target row matches multiple source rows — PostgreSQL picks an arbitrary matching source row), but the intent is destroyed and you may update every row to a nonsensical value. **Always include the full join predicate.**

**Multiple-match behavior.** If one target row matches *multiple* source rows, the target row is updated **once** using values from an **arbitrary** one of the matches. This is non-deterministic — if it can happen, your query is ambiguous. Guard against it by ensuring the source side is unique on the join key (e.g., join to a subquery that aggregates or `DISTINCT`s the source).

```sql
-- SAFE: aggregate the source so each target matches at most one row
UPDATE products p
SET last_sold_at = latest.max_created
FROM (
  SELECT oi.product_id, MAX(o.created_at) AS max_created
  FROM order_items oi
  JOIN orders o ON o.id = oi.order_id
  GROUP BY oi.product_id
) latest
WHERE p.id = latest.product_id;
```

**Self-join updates** require the target aliased and the source as a separate reference:

```sql
-- Set each employee's salary to their manager's salary (cap at manager)
UPDATE employees e
SET salary = m.salary
FROM employees m
WHERE e.manager_id = m.id
  AND e.salary > m.salary;
```

**`UPDATE ... FROM` is planned as a join** — check `EXPLAIN`. If the source is large, ensure an index on the join column so the planner can use Hash or Merge Join instead of a Nested Loop with a Seq Scan.

### 6.4 DELETE ... USING — bulk delete via join

Symmetric to `UPDATE ... FROM`. The `USING` clause introduces source tables to join against; matching target rows are deleted.

```sql
-- Delete all order_items belonging to cancelled orders
DELETE FROM order_items oi
USING orders o
WHERE oi.order_id = o.id
  AND o.status = 'cancelled';
```

Equivalent forms exist with a subquery, but `USING` is usually clearer and lets the planner join directly:

```sql
-- Subquery form (also correct, sometimes planned differently)
DELETE FROM order_items
WHERE order_id IN (SELECT id FROM orders WHERE status = 'cancelled');
```

**`USING` with multiple tables** joins all of them:

```sql
DELETE FROM sessions s
USING users u, accounts a
WHERE s.user_id = u.id
  AND u.account_id = a.id
  AND a.status = 'closed';
```

**The same Cartesian danger applies** — omit the join predicate and you delete far more than intended. `DELETE` mistakes are unrecoverable without a backup. Always run the equivalent `SELECT` first, wrap in a transaction, and inspect the row count before committing.

### 6.5 unnest()-based bulk upsert — the parameterized batch pattern

The problem: from application code you want to insert/upsert a batch of rows using **bound parameters** (never string-concatenated values — SQL injection and no plan caching). But a multi-row `INSERT ... VALUES ($1,$2),($3,$4),...` needs a variable number of placeholders and hits the 65,535 parameter cap.

The solution: pass each *column* as one array parameter, and `unnest()` the parallel arrays into rows. Now a batch of any size uses a **fixed** number of parameters (one per column), and the statement text is constant (great for plan caching).

```sql
INSERT INTO products (sku, name, price, category_id)
SELECT sku, name, price, category_id
FROM unnest(
  $1::text[],       -- skus:        ['SKU-1','SKU-2',...]
  $2::text[],       -- names:       ['Widget','Gadget',...]
  $3::numeric[],    -- prices:      [9.99, 19.99, ...]
  $4::int[]         -- category_ids:[3, 3, ...]
) AS t(sku, name, price, category_id)
ON CONFLICT (sku) DO UPDATE
  SET name = EXCLUDED.name,
      price = EXCLUDED.price,
      category_id = EXCLUDED.category_id,
      updated_at = NOW();
```

**Why this is the go-to pattern:**
- **Fixed parameter count** (4 params here) regardless of batch size — send 10,000 rows in 4 array parameters. (The arrays themselves can be large, but they count as *one* parameter each.)
- **Constant SQL text** → the server caches the plan; no re-parse per batch.
- **Fully parameterized** → injection-safe.
- **Combines with `ON CONFLICT`** → set-based upsert (Topic 59) over the whole batch in one statement.
- **Type safety** via explicit `::type[]` casts on each array.

**Gotcha — array length mismatch.** `unnest` of multiple arrays of *unequal* length pads the shorter arrays with NULLs to the length of the longest. Always ensure your parallel arrays are the same length in application code, or you will silently insert NULLs.

**`unnest` WITH ORDINALITY** adds a row-number column if you need to preserve input order:

```sql
SELECT * FROM unnest($1::text[]) WITH ORDINALITY AS t(sku, position);
```

### 6.6 Batch sizing and transaction sizing

These are two distinct knobs, often conflated:

- **Batch size** = how many rows per SQL *statement* (rows per `INSERT`, per `COPY` chunk, per `unnest` array).
- **Transaction size** = how many rows per `COMMIT` (how many statements between `BEGIN` and `COMMIT`).

**Transaction size dominates fsync cost.** With `synchronous_commit = on`, cost is one fsync per commit. So you want *many* rows per transaction. But not unbounded:

| Transaction size | Consequence |
|---|---|
| 1 row / txn | fsync per row → throughput capped at disk IOPS (~hundreds–few thousand rows/s) |
| 1,000–10,000 rows / txn | **sweet spot** — fsync amortized, locks/WAL/memory bounded |
| 1,000,000+ rows / txn | long-held locks, huge WAL growth, replication lag, a single error rolls back everything, high memory for `RETURNING`, subxact/lock table pressure |

**Why not one giant transaction?**
1. **Lock duration** — locks (row locks on updated rows, the table's `RowExclusiveLock`) are held until commit; long transactions block DDL and bloat cleanup.
2. **WAL and bloat** — dead tuples from a bulk `UPDATE`/`DELETE` cannot be vacuumed until the transaction commits *and* no older snapshot needs them. A long transaction also holds back the "oldest xmin," blocking autovacuum **globally**.
3. **Replication lag** — a 10GB transaction ships to replicas as one unit at commit.
4. **All-or-nothing risk** — a failure at row 9,999,999 rolls back everything; you restart from scratch.
5. **Memory** — `RETURNING` on a huge single transaction buffers results.

**Batch size** is bounded by the parameter cap (65,535), statement memory, and diminishing returns (past a few thousand rows the per-statement overhead is already amortized).

**Practical recipe for loading N million rows:**
```
for each chunk of 5,000 rows:
    BEGIN
    COPY (or unnest-insert) the 5,000 rows
    COMMIT
```
5,000 rows/statement, 5,000 rows/txn, one fsync per chunk. This balances throughput against lock/WAL/bloat pressure and gives natural restart points.

### 6.7 Disabling and deferring indexes and triggers for large loads

For **initial loads** and **large migrations** (loading into an empty or newly-created table), the fastest path removes per-row maintenance work and does it once, in bulk, at the end.

**Drop and rebuild indexes.** Inserting a row into a B-tree is a random descent + possible page split — O(log n) with random I/O, per index, per row. Building an index from scratch on a full table is a *sort* + sequential bottom-up build — far cheaper per row.

```sql
BEGIN;
DROP INDEX idx_products_category;      -- drop secondary indexes (NOT the PK if FKs need it)
-- ... COPY 50M rows ...
CREATE INDEX idx_products_category ON products (category_id);  -- one bulk build
COMMIT;
```

Rule: for a load that adds a *large fraction* of the table, drop-and-rebuild wins. For appending a *small fraction* to a huge existing table, keeping indexes is usually better (rebuilding scans the whole table).

**Disable triggers.** Row triggers fire per row. If the trigger's work is not needed during the load (e.g., an audit trigger you will backfill differently, or a denormalization trigger you will recompute in bulk):

```sql
ALTER TABLE products DISABLE TRIGGER USER;   -- disable user triggers (keeps FK/constraint triggers)
-- ... bulk load ...
ALTER TABLE products ENABLE TRIGGER USER;
```
`DISABLE TRIGGER USER` disables user-defined triggers but keeps internal constraint triggers. `DISABLE TRIGGER ALL` also disables FK enforcement — dangerous, use only when you will re-validate.

**Defer foreign-key and unique constraints.** FK checks are per-row constraint triggers. If the constraint is declared `DEFERRABLE`, defer validation to commit so the whole batch is checked at once (and so out-of-order inserts across mutually-referencing tables succeed):

```sql
BEGIN;
SET CONSTRAINTS ALL DEFERRED;   -- check deferrable constraints at COMMIT, not per row
-- ... bulk insert into both parent and child tables in any order ...
COMMIT;                          -- all FK checks run here, once
```

**Drop and re-add FK / add NOT VALID.** For very large loads you can drop the FK, load, and re-add it. Re-adding validates the whole table (a single scan). To avoid the validation lock, add it `NOT VALID` then `VALIDATE CONSTRAINT` separately (takes a weaker lock):

```sql
ALTER TABLE order_items ADD CONSTRAINT fk_product
  FOREIGN KEY (product_id) REFERENCES products(id) NOT VALID;   -- fast, no scan
ALTER TABLE order_items VALIDATE CONSTRAINT fk_product;          -- scans, but SHARE UPDATE lock
```

**Tune load-time settings** (session or system):
- `SET synchronous_commit = off;` — do not wait for WAL fsync per commit (risk: lose last few commits on crash, but no torn data). Huge speedup for many small transactions.
- `SET maintenance_work_mem = '2GB';` — speeds `CREATE INDEX` and constraint validation.
- Increase `max_wal_size` / `checkpoint_timeout` — fewer checkpoints during the load.
- Consider `UNLOGGED` table for scratch/staging data (no WAL at all; lost on crash, not replicated) then `ALTER TABLE ... SET LOGGED` or copy into the real table.

**The canonical fast bulk-load procedure:**
```
1. Create table (or TRUNCATE) — enables WAL-skip if wal_level=minimal
2. Drop secondary indexes; disable non-essential triggers
3. SET synchronous_commit = off; SET maintenance_work_mem high
4. COPY the data in chunked transactions
5. Recreate indexes; re-enable triggers
6. ADD / VALIDATE foreign keys (NOT VALID then VALIDATE)
7. ANALYZE the table (refresh planner statistics!)
```

Step 7 is non-negotiable: after a big load the statistics are stale and the planner will make terrible choices until `ANALYZE` runs. Autovacuum will eventually do it, but run it explicitly so the very next queries are planned correctly.

### 6.8 INSERT ... SELECT — server-side bulk copy

When the source data already lives in the database, `INSERT ... SELECT` moves it entirely server-side — no data crosses the network to the client and back.

```sql
-- Archive old orders into a history table in one server-side statement
INSERT INTO orders_archive (id, customer_id, total_amount, status, created_at)
SELECT id, customer_id, total_amount, status, created_at
FROM orders
WHERE created_at < NOW() - INTERVAL '2 years';
```

This is often the single fastest way to move/transform data between tables because it never leaves the server. Combine with `ON CONFLICT` for idempotent backfills and with a `WHERE` for chunking. The reverse — copying *out* to another table conditionally — pairs with a chunked `DELETE` for archive-then-purge jobs.

### 6.9 pg-copy-streams — COPY from Node.js

The `pg` driver does not natively expose `COPY`; the companion package **`pg-copy-streams`** provides `copyFrom`/`copyTo` streams that plug into the wire protocol's COPY subprotocol. This is the fastest ingestion path available from Node.js — you pipe a readable stream of CSV/text/binary straight into PostgreSQL. Full code is in Section 11.

---

## 7. EXPLAIN — Bulk Operations in the Plan

### 7.1 Multi-row INSERT

```sql
EXPLAIN (ANALYZE, BUFFERS)
INSERT INTO categories (name, parent_id) VALUES
  ('A', NULL), ('B', 1), ('C', 1), ('D', 1);
```

```
Insert on categories  (cost=0.00..0.05 rows=0 width=0)
                       (actual time=0.089..0.090 rows=0 loops=1)
  ->  Values Scan on "*VALUES*"  (cost=0.00..0.05 rows=4 width=36)
                                  (actual time=0.002..0.006 rows=4 loops=1)
Buffers: shared hit=8 dirtied=1
Planning Time: 0.040 ms
Execution Time: 0.180 ms
```

**Reading it:**
- `Insert on categories` — the ModifyTable node; `rows=0` because without `RETURNING` nothing is projected out.
- `Values Scan on "*VALUES*"` — the 4-row VALUES list is the row source, scanned once (`rows=4`).
- `Buffers: shared ... dirtied=1` — one heap page dirtied by the four small rows.
- Contrast with 4 separate `INSERT`s: 4 planning cycles, 4 executions, potentially 4 fsyncs.

### 7.2 UPDATE ... FROM — it is a join plan

```sql
EXPLAIN (ANALYZE, BUFFERS)
UPDATE order_items oi
SET unit_price = p.price
FROM products p
WHERE oi.product_id = p.id
  AND oi.status = 'pending';
```

```
Update on order_items oi  (cost=308.00..21450.00 rows=0 width=0)
                          (actual time=145.2..145.2 rows=0 loops=1)
  ->  Hash Join  (cost=308.00..21450.00 rows=48000 width=42)
                 (actual time=4.1..96.7 rows=47912 loops=1)
        Hash Cond: (oi.product_id = p.id)
        ->  Seq Scan on order_items oi  (cost=0.00..18500.00 rows=48000 width=14)
              Filter: (status = 'pending')
              Rows Removed by Filter: 452088
        ->  Hash  (cost=180.00..180.00 rows=8000 width=36)
              Buckets: 8192  Batches: 1  Memory Usage: 620kB
              ->  Seq Scan on products p  (cost=0.00..180.00 rows=8000 width=36)
  Buffers: shared hit=12050 read=4200 dirtied=1820
Planning Time: 0.35 ms
Execution Time: 146.1 ms
```

**Reading it:**
- The top `Update on order_items` node performs the writes; the child is a **Hash Join** — confirming `UPDATE ... FROM` is executed as a join.
- `products` (smaller) is hashed; `order_items` is scanned and probed.
- `Rows Removed by Filter: 452088` — the `status='pending'` filter is applied during the scan; only 47,912 rows are actually updated.
- `dirtied=1820` — 1,820 heap pages were modified (new tuple versions written; MVCC bloat begins here).
- **Optimization signal**: the `Seq Scan on order_items` with a filter suggests an index on `order_items(status)` (or a partial index) could avoid scanning all 500K rows if `pending` is selective. And an index on `products(id)` (the PK — already present) lets the planner consider a Nested Loop for small target sets.

### 7.3 DELETE ... USING

```sql
EXPLAIN (ANALYZE, BUFFERS)
DELETE FROM order_items oi
USING orders o
WHERE oi.order_id = o.id
  AND o.status = 'cancelled';
```

```
Delete on order_items oi  (cost=1250.00..9800.00 rows=0 width=0)
                          (actual time=88.4..88.4 rows=0 loops=1)
  ->  Hash Join  (cost=1250.00..9800.00 rows=12000 width=12)
                 (actual time=6.2..40.1 rows=11840 loops=1)
        Hash Cond: (oi.order_id = o.id)
        ->  Seq Scan on order_items oi  (cost=0.00..8000.00 rows=500000 width=12)
        ->  Hash  (cost=1100.00..1100.00 rows=12000 width=8)
              ->  Index Scan using idx_orders_status on orders o
                    Index Cond: (status = 'cancelled')
                    (actual rows=3100 loops=1)
  Buffers: shared hit=9200 read=1400 dirtied=980
Planning Time: 0.28 ms
Execution Time: 89.0 ms
```

**Reading it:**
- Again a Hash Join under the `Delete` node.
- `orders` is filtered via `idx_orders_status` (3,100 cancelled orders) and hashed.
- `order_items` is seq-scanned and probed; 11,840 rows deleted (avg ~3.8 items/order).
- `dirtied=980` — deleted tuples are marked dead (set `xmax`), dirtying pages; space is reclaimed later by VACUUM, not immediately.

### 7.4 unnest()-based bulk insert

```sql
EXPLAIN (ANALYZE)
INSERT INTO products (sku, name, price)
SELECT sku, name, price
FROM unnest($1::text[], $2::text[], $3::numeric[]) AS t(sku, name, price);
```

```
Insert on products  (cost=0.00..25.00 rows=0 width=0)
                     (actual time=2.10..2.10 rows=0 loops=1)
  ->  Function Scan on unnest t  (cost=0.00..25.00 rows=1000 width=68)
                                  (actual time=0.30..0.85 rows=1000 loops=1)
Planning Time: 0.06 ms
Execution Time: 3.40 ms
```

**Reading it:**
- `Function Scan on unnest` — the parallel arrays are expanded into a 1,000-row set-returning function scan, then fed to `Insert`.
- One statement, one plan, 1,000 rows inserted. The estimate `rows=1000` comes from the default set-returning-function estimate; for very large arrays the estimate may be off but it does not affect correctness.
- No per-row round trip — the whole batch arrived in three array parameters.

### 7.5 What to look for in bulk-write plans

| Symptom | Likely Problem | Fix |
|---|---|---|
| Nested Loop + Seq Scan on source in `UPDATE...FROM` | No index on source join column | Index the join column; ANALYZE |
| `Hash ... Batches > 1` on the source hash | Source hash spills `work_mem` | Raise `work_mem` for the session |
| Huge `Rows Removed by Filter` on target scan | Non-selective or unindexed filter | Add (partial) index on the filter column |
| `dirtied` pages ≈ table size after UPDATE | Full-table rewrite (bloat incoming) | Chunk the update; schedule VACUUM |
| Long Execution Time on `RETURNING` bulk insert | Materializing millions of returned rows | Drop RETURNING or page it |

---

## 8. Query Examples

### Example 1 — Basic: multi-row INSERT with RETURNING

```sql
-- Insert three categories in one statement; get their generated ids back
INSERT INTO categories (name, parent_id)
VALUES
  ('Electronics', NULL),   -- top-level
  ('Laptops',      1),     -- child of Electronics
  ('Phones',       1)      -- child of Electronics
RETURNING id, name;        -- returns the serial ids assigned by the sequence
```

Result:
```
 id |    name
----+-------------
  1 | Electronics
  2 | Laptops
  3 | Phones
```

One parse, one plan, one transaction, one fsync — versus three separate inserts.

### Example 2 — Intermediate: UPDATE ... FROM to denormalize a computed value

```sql
-- Backfill each order's total_amount from its line items (one set-based statement).
-- Without UPDATE...FROM this would be an N+1 loop of per-order UPDATEs.
UPDATE orders o
SET total_amount = agg.order_total,
    updated_at   = NOW()
FROM (
  SELECT order_id,
         SUM(quantity * unit_price) AS order_total   -- aggregate line items per order
  FROM order_items
  GROUP BY order_id
) agg
WHERE o.id = agg.order_id
  AND o.total_amount IS DISTINCT FROM agg.order_total;  -- skip already-correct rows (avoids dead tuples)
```

Notes:
- The subquery guarantees **one source row per order**, avoiding the arbitrary-match ambiguity.
- `IS DISTINCT FROM` (NULL-safe inequality) means rows already correct are not rewritten — no needless dead tuples, no wasted WAL.

### Example 3 — Production grade: chunked bulk upsert with unnest()

**Scenario:** A nightly sync ingests ~2,000,000 product price updates from a supplier feed into a `products` table (currently ~5M rows, 4 secondary indexes). We must upsert (insert new SKUs, update existing prices) idempotently, without locking the table for the whole 2M rows, and without exhausting the parameter limit.

**Strategy:** chunk into 5,000-row batches; each batch is a single `unnest`-driven `INSERT ... ON CONFLICT`; each batch is its own transaction (natural restart points, bounded locks/WAL).

```sql
-- One batch (executed ~400 times, once per 5,000-row chunk, each in its own txn)
INSERT INTO products (sku, name, price, category_id, updated_at)
SELECT sku, name, price, category_id, NOW()
FROM unnest(
  $1::text[],       -- 5,000 skus
  $2::text[],       -- 5,000 names
  $3::numeric[],    -- 5,000 prices
  $4::int[]         -- 5,000 category ids
) AS t(sku, name, price, category_id)
ON CONFLICT (sku) DO UPDATE
  SET price       = EXCLUDED.price,
      name        = EXCLUDED.name,
      category_id = EXCLUDED.category_id,
      updated_at  = NOW()
WHERE products.price IS DISTINCT FROM EXCLUDED.price   -- only update rows whose price changed
   OR products.name  IS DISTINCT FROM EXCLUDED.name;
```

**Table size:** 5M rows, 4 secondary indexes, unique index on `sku`.
**Index availability:** unique index on `products(sku)` is required for `ON CONFLICT (sku)`.
**Performance expectation:** ~5–15ms per 5,000-row batch → ~2–6s total server-side for 400 batches (network/driver overhead adds to wall-clock). The `WHERE ... IS DISTINCT FROM` guard skips unchanged rows, minimizing dead tuples and index churn.

**EXPLAIN for one batch:**
```sql
EXPLAIN (ANALYZE, BUFFERS)
INSERT INTO products (sku, name, price, category_id, updated_at)
SELECT sku, name, price, category_id, NOW()
FROM unnest($1::text[], $2::text[], $3::numeric[], $4::int[])
  AS t(sku, name, price, category_id)
ON CONFLICT (sku) DO UPDATE SET price = EXCLUDED.price;
```
```
Insert on products  (cost=0.00..90.00 rows=0 width=0)
                     (actual time=8.90..8.90 rows=0 loops=1)
  Conflict Resolution: UPDATE
  Conflict Arbiter Indexes: products_sku_key
  Tuples Inserted: 320
  Conflicting Tuples: 4680
  ->  Function Scan on unnest t  (cost=0.00..50.00 rows=5000 width=72)
                                  (actual time=1.10..3.40 rows=5000 loops=1)
  Buffers: shared hit=48200 read=1900 dirtied=2100
Planning Time: 0.09 ms
Execution Time: 9.60 ms
```

**Reading it:** `Conflict Arbiter Indexes: products_sku_key` confirms the unique index drives conflict detection. `Tuples Inserted: 320` new SKUs, `Conflicting Tuples: 4680` existing SKUs routed to the `DO UPDATE` path. Buffer reads reflect probing the 4 secondary indexes.

---

## 9. Wrong → Right Patterns

### Wrong 1: The write-side N+1 — a loop of single-row inserts

```javascript
// WRONG: one INSERT per row, in auto-commit → one fsync per row
for (const p of products) {                       // 10,000 iterations
  await pool.query(
    'INSERT INTO products (sku, name, price) VALUES ($1, $2, $3)',
    [p.sku, p.name, p.price]
  );                                              // 10,000 round trips, 10,000 fsyncs
}
```

**What actually happens:** 10,000 network round trips, 10,000 parse/plan cycles, and — because each `pool.query` auto-commits — **10,000 fsyncs**. On a disk that sustains ~1,000 fsyncs/s this loop takes ~10 seconds *minimum*, regardless of CPU. It is the write-side twin of the read N+1 (Topic 18).

```javascript
// RIGHT: one unnest-driven bulk insert — 1 statement, 1 transaction, 1 fsync
await pool.query(
  `INSERT INTO products (sku, name, price)
   SELECT * FROM unnest($1::text[], $2::text[], $3::numeric[])`,
  [products.map(p => p.sku), products.map(p => p.name), products.map(p => p.price)]
);
```

The right version runs in single-digit milliseconds. Same rows written; the 10,000 fsyncs collapsed to 1.

### Wrong 2: UPDATE ... FROM with a missing join predicate

```sql
-- WRONG: no join condition between target and source → Cartesian match
UPDATE order_items oi
SET unit_price = p.price
FROM products p;                 -- forgot: WHERE oi.product_id = p.id
-- EVERY order_item row is updated. Each is matched to an ARBITRARY product,
-- so every unit_price becomes some random product's price. Data destroyed.
```

**Why it is wrong at the execution level:** with no join predicate the plan is a Cartesian product (`Nested Loop` with no `Join Filter`). Each target row matches all source rows; PostgreSQL updates each target row once using an arbitrary source row. There is no error — it "succeeds" and silently corrupts every row.

```sql
-- RIGHT: full join predicate + a filter to touch only relevant rows
UPDATE order_items oi
SET unit_price = p.price
FROM products p
WHERE oi.product_id = p.id       -- the essential join condition
  AND oi.status = 'pending';     -- restrict to rows that should change
```

Always dry-run the join as a `SELECT count(*) FROM order_items oi JOIN products p ON ...` first and confirm the count matches expectations.

### Wrong 3: Everything in one giant transaction

```sql
-- WRONG: delete 50M rows in a single transaction
BEGIN;
DELETE FROM events WHERE created_at < NOW() - INTERVAL '90 days';  -- 50M rows
COMMIT;
```

**Why it is wrong:** the transaction holds a `RowExclusiveLock` and 50M row locks until commit; it generates 50M dead tuples that cannot be vacuumed until it commits; it holds back the global `xmin` horizon (blocking autovacuum everywhere); it bloats WAL by gigabytes (replication lag); and if it fails at row 49,999,999 the whole thing rolls back. It can also run for tens of minutes, blocking DDL and stressing memory.

```sql
-- RIGHT: chunked delete, each chunk its own transaction
-- (loop this in application code or a DO block until 0 rows deleted)
DELETE FROM events
WHERE id IN (
  SELECT id FROM events
  WHERE created_at < NOW() - INTERVAL '90 days'
  ORDER BY id
  LIMIT 10000                      -- delete 10K at a time
);
-- COMMIT between chunks; VACUUM can reclaim space as you go
```

Each chunk commits quickly, releases locks, lets autovacuum reclaim space between chunks, and gives natural restart points.

### Wrong 4: String-concatenating a multi-row INSERT

```javascript
// WRONG: building VALUES by string concatenation
const values = products
  .map(p => `('${p.sku}', '${p.name}', ${p.price})`)  // SQL injection + no plan cache
  .join(',');
await pool.query(`INSERT INTO products (sku, name, price) VALUES ${values}`);
// - SQL INJECTION if any name contains a quote or '); DROP TABLE ...
// - a DIFFERENT SQL string every call → server re-parses/re-plans every time
// - hits the statement-size / parameter limits unpredictably
```

**Why it is wrong:** unescaped interpolation is a textbook injection vector, and the ever-changing statement text defeats the plan cache. A name like `O'Brien` breaks it outright.

```javascript
// RIGHT: fixed SQL text, parameterized arrays via unnest
await pool.query(
  `INSERT INTO products (sku, name, price)
   SELECT * FROM unnest($1::text[], $2::text[], $3::numeric[])`,
  [products.map(p => p.sku), products.map(p => p.name), products.map(p => p.price)]
);
// injection-safe, constant SQL text (cached plan), fixed 3-parameter count
```

### Wrong 5: Loading into a table with all indexes and triggers live

```sql
-- WRONG: bulk-load 100M rows into a table with 6 indexes + an audit trigger, as-is
COPY events FROM STDIN WITH (FORMAT csv);
-- Every row: 1 heap write + 6 index descents/splits + 1 trigger call.
-- The index maintenance and trigger dominate; load takes hours.
```

```sql
-- RIGHT: strip per-row work, load, rebuild in bulk, refresh stats
BEGIN;
ALTER TABLE events DISABLE TRIGGER USER;         -- silence per-row triggers
DROP INDEX idx_events_user, idx_events_type, idx_events_created,
           idx_events_session, idx_events_ip, idx_events_status;  -- drop secondaries
SET LOCAL synchronous_commit = off;
SET LOCAL maintenance_work_mem = '2GB';
COMMIT;

-- ... COPY the 100M rows in chunked transactions ...

BEGIN;
CREATE INDEX idx_events_user    ON events (user_id);   -- bulk sorted builds
CREATE INDEX idx_events_type    ON events (event_type);
CREATE INDEX idx_events_created ON events (created_at);
CREATE INDEX idx_events_session ON events (session_id);
CREATE INDEX idx_events_ip      ON events (ip_address);
CREATE INDEX idx_events_status  ON events (status);
ALTER TABLE events ENABLE TRIGGER USER;
COMMIT;

ANALYZE events;   -- refresh statistics so the planner is not blind
```

Building six indexes once on the full table (six sorts) is dramatically cheaper than 600M random index insertions. The load drops from hours to minutes.

---

## 10. Performance Profile

### 10.1 The fsync ceiling (why transaction sizing dominates)

| Approach | Fsyncs for 1M rows | Typical throughput |
|---|---|---|
| 1 row / statement, auto-commit | 1,000,000 | ~500–2,000 rows/s (disk-bound) |
| 1,000 rows / txn (multi-row INSERT) | 1,000 | ~100,000–300,000 rows/s |
| `COPY`, 1M rows / txn | 1 | ~500,000–2,000,000 rows/s |
| `COPY` + `synchronous_commit=off` | 0 waited | CPU/parse-bound, highest |

The per-row auto-commit path is capped by disk IOPS for the WAL fsync — pure CPU speed is irrelevant. This is the single most important performance fact about bulk writes.

### 10.2 Method comparison

| Method | Speed | Parameterized | ON CONFLICT | Expressions | RETURNING | Best for |
|---|---|---|---|---|---|---|
| Single-row INSERT loop | slowest | yes | yes | yes | yes | never (in a loop) |
| Multi-row INSERT VALUES | fast | yes (≤65535 params) | yes | yes | yes | up to a few thousand rows |
| unnest() + SELECT | fast | yes (fixed params) | yes | yes | yes | parameterized batches, upserts |
| INSERT ... SELECT | fast | n/a (server-side) | yes | yes | yes | server-side data movement |
| COPY FROM STDIN | fastest | no (stream) | no | no | no | tens of thousands+ rows |
| COPY (binary) | fastest+ | no | no | no | no | maximum-throughput loaders |

`COPY` cannot do `ON CONFLICT` or expressions — for upserts, `COPY` into a staging table then `INSERT ... SELECT ... ON CONFLICT` from staging.

### 10.3 Scaling: bulk INSERT/COPY

- **1M rows:** `COPY` in ~1–4s (indexes present); sub-second heap-only (indexes dropped). Multi-row INSERT batched at 1K/txn: ~5–20s.
- **10M rows:** `COPY` with indexes: ~30–90s. Drop-indexes-load-rebuild: often ~2–4× faster overall. Watch checkpoint spikes — raise `max_wal_size`.
- **100M rows:** always drop secondary indexes, disable triggers, `synchronous_commit=off`, chunked transactions, then rebuild + `ANALYZE`. Difference between the naive and tuned path is measured in hours.

### 10.4 Scaling: bulk UPDATE/DELETE

These are join-bound *and* bloat-bound:
- **Join cost**: index the source join column. A 10M-row `UPDATE...FROM` with a Hash Join over an indexed source is minutes; a Nested Loop with a Seq-Scanned source is hours.
- **Bloat cost**: every updated/deleted row is a dead tuple. Updating 100M rows can transiently ~double table size until VACUUM reclaims it. Chunk the operation so autovacuum keeps pace, and consider `VACUUM` between chunks.
- **HOT updates**: if an `UPDATE` does not change any *indexed* column and the page has free space, PostgreSQL uses a Heap-Only Tuple update that avoids updating indexes — much cheaper. Leave `fillfactor` headroom (e.g., 80) on frequently-updated tables to enable HOT.
- **Lock footprint**: `UPDATE`/`DELETE` take `RowExclusiveLock` on the table and row locks on touched rows — chunking bounds how long they are held and how much they block replication.

### 10.5 Index interactions

- **N indexes multiply write cost** ~linearly: each insert updates all N indexes.
- **`CREATE INDEX` locks writes**; use `CREATE INDEX CONCURRENTLY` to build without blocking (slower, cannot run in a transaction block, can leave an invalid index on failure).
- **Unique indexes are mandatory for `ON CONFLict`** and add a duplicate check per insert.
- **Partial indexes** (`WHERE status='pending'`) shrink the index and speed both the load and the filtered scans in bulk updates.

### 10.6 Key optimization levers (summary)

```
synchronous_commit = off      -- skip per-commit fsync wait (durability tradeoff)
maintenance_work_mem = high   -- faster CREATE INDEX / VALIDATE CONSTRAINT
work_mem = high (session)     -- avoid hash spills in UPDATE...FROM / DELETE...USING joins
max_wal_size = high           -- fewer checkpoints during the load
UNLOGGED table for staging    -- no WAL at all (lost on crash, not replicated)
fillfactor < 100              -- headroom for HOT updates on hot tables
Drop indexes + rebuild        -- bulk sorted build beats N random insertions
DISABLE TRIGGER USER          -- skip per-row trigger cost
NOT VALID + VALIDATE          -- add FKs with a weaker lock
ANALYZE after the load        -- refresh statistics (never skip)
```

---

## 11. Node.js Integration

### 11.1 Multi-row INSERT via unnest (the workhorse)

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Bulk insert a batch of products with fixed parameter count via unnest().
async function bulkInsertProducts(products) {
  if (products.length === 0) return { inserted: 0 };
  const { rows } = await pool.query(
    `INSERT INTO products (sku, name, price, category_id)
     SELECT * FROM unnest($1::text[], $2::text[], $3::numeric[], $4::int[])
     ON CONFLICT (sku) DO UPDATE
       SET price = EXCLUDED.price, name = EXCLUDED.name, updated_at = NOW()
     RETURNING id, sku`,                       // RETURNING works with unnest inserts
    [
      products.map(p => p.sku),                // $1 — one array parameter per column
      products.map(p => p.name),               // $2
      products.map(p => p.price),              // $3
      products.map(p => p.categoryId),         // $4
    ]
  );
  return { inserted: rows.length, ids: rows };
}
```

### 11.2 Chunked transactions (transaction sizing)

```javascript
// Load a large array in bounded-size transactions: throughput without huge locks/WAL.
async function bulkLoadChunked(rows, chunkSize = 5000) {
  const client = await pool.connect();
  try {
    let done = 0;
    for (let i = 0; i < rows.length; i += chunkSize) {
      const chunk = rows.slice(i, i + chunkSize);
      await client.query('BEGIN');
      await client.query(
        `INSERT INTO products (sku, name, price)
         SELECT * FROM unnest($1::text[], $2::text[], $3::numeric[])`,
        [chunk.map(r => r.sku), chunk.map(r => r.name), chunk.map(r => r.price)]
      );
      await client.query('COMMIT');            // one fsync per chunk, natural restart point
      done += chunk.length;
    }
    return { loaded: done };
  } catch (err) {
    await client.query('ROLLBACK');            // only the current chunk rolls back
    throw err;
  } finally {
    client.release();                          // ALWAYS release back to the pool
  }
}
```

### 11.3 COPY FROM STDIN with pg-copy-streams (fastest ingestion)

```javascript
import pg from 'pg';
import { from as copyFrom } from 'pg-copy-streams';
import { pipeline } from 'node:stream/promises';
import { Readable } from 'node:stream';

const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Stream rows straight into PostgreSQL via the COPY subprotocol — no per-row round trips.
async function bulkCopyProducts(rows) {
  const client = await pool.connect();
  try {
    const stream = client.query(
      copyFrom(`COPY products (sku, name, price)
                FROM STDIN WITH (FORMAT csv)`)
    );
    // Produce CSV lines from the source data (escape per CSV rules in real code)
    const source = Readable.from(function* () {
      for (const r of rows) {
        const name = `"${String(r.name).replace(/"/g, '""')}"`;  // CSV-escape quotes
        yield `${r.sku},${name},${r.price}\n`;
      }
    }());
    await pipeline(source, stream);            // backpressure-aware streaming
    return { copied: rows.length };
  } finally {
    client.release();
  }
}
```

For piping a large file straight from disk to the database:

```javascript
import fs from 'node:fs';

async function copyFromCsvFile(path) {
  const client = await pool.connect();
  try {
    const dbStream = client.query(
      copyFrom(`COPY products (sku, name, price)
                FROM STDIN WITH (FORMAT csv, HEADER true)`)
    );
    await pipeline(fs.createReadStream(path), dbStream);  // file → socket → PostgreSQL
  } finally {
    client.release();
  }
}
```

### 11.4 UPDATE ... FROM driven by a batch (bulk update from app data)

```javascript
// Apply a batch of (id, newPrice) updates in ONE statement via unnest + UPDATE...FROM.
async function bulkUpdatePrices(updates) {
  const { rowCount } = await pool.query(
    `UPDATE products p
     SET price = v.price, updated_at = NOW()
     FROM unnest($1::int[], $2::numeric[]) AS v(id, price)
     WHERE p.id = v.id
       AND p.price IS DISTINCT FROM v.price`,   // skip unchanged → fewer dead tuples
    [updates.map(u => u.id), updates.map(u => u.price)]
  );
  return { updated: rowCount };
}
```

**A note on ORMs:** `COPY` is not exposed by any mainstream ORM — you always drop to the driver (`pg-copy-streams`) for it. Multi-row inserts *are* supported by most ORMs (`createMany`, bulk `insert`), but `unnest`-based upserts, `UPDATE ... FROM`, and `DELETE ... USING` frequently require raw SQL. Section 12 covers exactly where each ORM helps and where you must escape to raw SQL.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do bulk operations?** — Partially. `createMany` generates a single multi-row `INSERT`. `updateMany`/`deleteMany` apply one *filter-based* `UPDATE`/`DELETE` (same value/condition for all matched rows — **not** a per-row update-with-join). `upsert` is single-row; batch upsert needs raw SQL. No `COPY`, no `UPDATE ... FROM`, no `unnest`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

// Multi-row INSERT — one statement, skips duplicates on the unique key
await prisma.product.createMany({
  data: products.map(p => ({ sku: p.sku, name: p.name, price: p.price })),
  skipDuplicates: true,               // → ON CONFLICT DO NOTHING semantics
});

// updateMany: ONE value applied to ALL matched rows (not a per-row join update)
await prisma.product.updateMany({
  where: { categoryId: 3 },
  data: { discontinued: true },       // same value for every matched row
});

// Per-row bulk upsert / UPDATE...FROM → raw SQL with unnest
await prisma.$executeRaw`
  INSERT INTO products (sku, name, price)
  SELECT * FROM unnest(${products.map(p => p.sku)}::text[],
                       ${products.map(p => p.name)}::text[],
                       ${products.map(p => p.price)}::numeric[])
  ON CONFLICT (sku) DO UPDATE SET price = EXCLUDED.price`;
```

**Where it breaks:** `updateMany` cannot set different values per row (no join), so batch price updates from external data require `$executeRaw` with `unnest`. No `COPY` support at all. `createMany` does not return created rows (`RETURNING` unavailable) except in recent versions via `createManyAndReturn`.

**Verdict:** `createMany`/`deleteMany` are genuinely useful. For per-row bulk updates, batch upserts, and `COPY`, drop to `$executeRaw`/`$queryRaw` or `pg-copy-streams`.

---

### Drizzle ORM

**Can Drizzle do bulk operations?** — Multi-row insert and filter-based update/delete yes; `onConflictDoUpdate` for upsert yes; `UPDATE ... FROM` and `COPY` no (raw SQL / driver).

```typescript
import { db } from './db';
import { products } from './schema';
import { sql, inArray, eq } from 'drizzle-orm';

// Multi-row INSERT with batch upsert (ON CONFLICT)
await db.insert(products)
  .values(rows.map(r => ({ sku: r.sku, name: r.name, price: r.price })))
  .onConflictDoUpdate({
    target: products.sku,
    set: { price: sql`excluded.price`, name: sql`excluded.name` },
  });

// Per-row UPDATE...FROM via raw SQL (unnest)
await db.execute(sql`
  UPDATE products p
  SET price = v.price
  FROM unnest(${ids}::int[], ${prices}::numeric[]) AS v(id, price)
  WHERE p.id = v.id`);
```

**Where it breaks:** no `UPDATE ... FROM` builder and no `COPY`; both need `db.execute(sql\`...\`)` or the raw driver. Multi-row `values()` still respects the 65,535 parameter cap, so chunk large batches yourself.

**Verdict:** Excellent for multi-row insert and `ON CONFLICT` upserts with full typing. Use `sql` template + `unnest` for join-updates; `pg-copy-streams` for `COPY`.

---

### Sequelize

**Can Sequelize do bulk operations?** — `bulkCreate` (multi-row INSERT, optional `updateOnDuplicate` for upsert), `update` (filter-based), `destroy` (filter-based delete). No `UPDATE ... FROM`, no `COPY`.

```javascript
const { Product } = require('./models');

// Multi-row INSERT with upsert on conflict
await Product.bulkCreate(
  rows.map(r => ({ sku: r.sku, name: r.name, price: r.price })),
  {
    updateOnDuplicate: ['price', 'name'],   // → ON CONFLICT (unique) DO UPDATE these cols
  }
);

// Filter-based update (one value for all matched rows)
await Product.update({ discontinued: true }, { where: { categoryId: 3 } });

// Per-row join update → raw query
await sequelize.query(
  `UPDATE products p SET price = v.price
   FROM unnest(:ids::int[], :prices::numeric[]) AS v(id, price)
   WHERE p.id = v.id`,
  { replacements: { ids, prices }, type: QueryTypes.UPDATE }
);
```

**Where it breaks:** `updateOnDuplicate` requires a unique constraint and updates the same columns for all; there is no per-row conditional. `bulkCreate` builds one large parameterized statement — very large batches hit the parameter limit, so pass reasonable chunks. No `COPY`.

**Verdict:** `bulkCreate` + `updateOnDuplicate` covers most upsert needs. Everything join-shaped or `COPY`-shaped goes through `sequelize.query()` / the raw driver.

---

### TypeORM

**Can TypeORM do bulk operations?** — Multi-row insert via `.insert().values([...])` or `save([...])`; `.orUpdate()` for upsert; `.update()`/`.delete()` are filter-based via QueryBuilder. No `UPDATE ... FROM`, no `COPY`.

```typescript
import { AppDataSource } from './data-source';
import { Product } from './entities/Product';

// Multi-row INSERT with ON CONFLICT DO UPDATE
await AppDataSource.createQueryBuilder()
  .insert()
  .into(Product)
  .values(rows.map(r => ({ sku: r.sku, name: r.name, price: r.price })))
  .orUpdate(['name', 'price'], ['sku'])     // update these cols, conflict target = sku
  .execute();

// Per-row join update → raw SQL
await AppDataSource.query(
  `UPDATE products p SET price = v.price
   FROM unnest($1::int[], $2::numeric[]) AS v(id, price)
   WHERE p.id = v.id`,
  [ids, prices]
);
```

**Where it breaks:** `save()` on an array can silently issue one statement *per entity* (N round trips) rather than a true multi-row insert — use `.insert().values([...])` for a real bulk insert. No `UPDATE ... FROM` builder, no `COPY`.

**Verdict:** Use the InsertQueryBuilder with `.orUpdate()` for bulk upserts. Avoid `save([])` for large batches. Raw SQL / `pg-copy-streams` for join-updates and `COPY`.

---

### Knex.js

**Can Knex do bulk operations?** — Multi-row `insert([...])`, `.onConflict().merge()` for upsert, filter-based `.update()`/`.del()`. No first-class `UPDATE ... FROM` (use `.updateFrom`-style raw or `whereRaw`), no `COPY` (use `pg-copy-streams` on the underlying connection).

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

// Multi-row INSERT with upsert; batchInsert chunks automatically
await knex('products')
  .insert(rows.map(r => ({ sku: r.sku, name: r.name, price: r.price })))
  .onConflict('sku')
  .merge(['name', 'price']);           // ON CONFLICT (sku) DO UPDATE SET name, price

// Auto-chunked bulk insert (handles the parameter limit for you)
await knex.batchInsert('products', rows, 1000);   // 1000 rows per statement

// Per-row join update via raw
await knex.raw(
  `UPDATE products p SET price = v.price
   FROM unnest(?::int[], ?::numeric[]) AS v(id, price)
   WHERE p.id = v.id`,
  [ids, prices]
);
```

**Where it breaks:** `UPDATE ... FROM` has no builder — use `knex.raw`. `COPY` requires grabbing the raw `pg` connection (`knex.client.acquireConnection()`) and piping via `pg-copy-streams`. `batchInsert` is the nicest built-in for chunking.

**Verdict:** Most SQL-transparent of the five. `.onConflict().merge()` and `batchInsert` are excellent; raw for join-updates and `COPY`.

---

### ORM Summary Table

| ORM | Multi-row INSERT | Batch upsert | UPDATE...FROM (per-row) | COPY | Verdict |
|---|---|---|---|---|---|
| Prisma | `createMany` | `$executeRaw` (updateMany is filter-only) | Raw SQL | No | Good inserts; raw for joins |
| Drizzle | `.insert().values([])` | `.onConflictDoUpdate()` | `sql` + unnest | No | Best typed inserts/upserts |
| Sequelize | `bulkCreate` | `updateOnDuplicate` | `sequelize.query()` | No | bulkCreate covers upsert |
| TypeORM | `.insert().values([])` | `.orUpdate()` | Raw SQL | No | Avoid save([]) for bulk |
| Knex | `insert([])` / `batchInsert` | `.onConflict().merge()` | `knex.raw` | Raw driver | Most transparent |

Universal truth: **no ORM exposes `COPY`** — the single fastest ingestion path always requires `pg-copy-streams` on the raw connection.

---

## 13. Practice Exercises

### Exercise 1 — Easy

You have an array of 500 new users in your Node.js app, each `{ email, name, country }`. The junior on your team wrote a `for` loop calling `INSERT` 500 times.

1. Rewrite it as a single bulk insert using `unnest()` with three array parameters.
2. Make it idempotent: if an email already exists, update the `name` and `country` instead of erroring. (Assume a unique index on `users(email)`.)

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines prior topics)

Given:
- `orders(id, customer_id, total_amount, status, created_at)`
- `order_items(id, order_id, product_id, quantity, unit_price)`

The `orders.total_amount` column has drifted out of sync with the actual line items. Write **one** `UPDATE ... FROM` statement that recomputes `total_amount` for every `pending` order as `SUM(quantity * unit_price)` of its items. Only rewrite rows whose stored total actually differs from the computed one (to avoid generating dead tuples for already-correct rows). Recall from Topic 11 how a one-to-many join fans out — make sure your source produces exactly one row per order.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation, naive answer is wrong)

You must delete all `sessions` belonging to users whose account was closed more than 30 days ago. The `sessions` table has 80,000,000 rows; roughly 5,000,000 qualify for deletion. A colleague proposes:

```sql
DELETE FROM sessions s USING users u
WHERE s.user_id = u.id AND u.closed_at < NOW() - INTERVAL '30 days';
```

1. Explain concretely what goes wrong if you run this as a single statement on a live production database (locks, bloat, WAL, replication, xmin horizon).
2. Rewrite it as a **chunked** delete (10,000 rows per transaction) that can be looped safely from application code, and explain why each chunk should be its own transaction.
3. What indexes make both the join and the chunk-selection fast?

```sql
-- Write your chunked delete here
```

---

### Exercise 4 — Interview simulation

You are handed a CSV file with 40,000,000 rows to load into a fresh `events(id, user_id, event_type, payload, created_at)` table that will have 5 secondary indexes and a per-row audit trigger. The load must finish inside a 30-minute maintenance window.

1. Write the end-to-end procedure (SQL + which settings to change) that loads the data as fast as possible.
2. Which step, if skipped, would make the *next day's* queries mysteriously slow, and why?
3. Why is `COPY FROM STDIN` (via `pg-copy-streams`) preferable to a multi-row `INSERT` here?

```sql
-- Write your load procedure here
```

---

## 14. Interview Questions

### Junior Level

**Q: How do you insert multiple rows into a table in a single statement, and why is it better than one `INSERT` per row?**

A: Use a multi-row `VALUES` list:

```sql
INSERT INTO products (name, category_id, price)
VALUES
  ('Keyboard', 3, 49.99),
  ('Mouse',    3, 24.99),
  ('Monitor',  3, 199.00);
```

It is better because a single statement means one round trip to the server, one parse/plan, and one transaction commit (one `fsync`) instead of N of each. For a few hundred rows this is often 10-50x faster than a loop of single-row inserts. The round-trip savings alone matter enormously when the app and database are on different hosts.

**Follow-up — Is there a limit to how many rows you can put in one `VALUES` list?** There is no hard row limit, but PostgreSQL caps a query at 65,535 bind parameters, so `rows × columns` must stay under that (e.g. 5 columns → ~13,000 rows max per parameterized statement). Practically, batches of 500-1,000 rows are the sweet spot; beyond that the per-statement gains flatten and very large statements consume more memory and hold locks longer.

---

**Q: What is `COPY` and when would you reach for it instead of `INSERT`?**

A: `COPY` is PostgreSQL's bulk-load command that streams rows from a file or STDIN directly into a table, bypassing most of the per-row overhead of `INSERT` (no per-statement parse/plan, minimal protocol chatter). Reach for it when loading thousands to millions of rows — a CSV import, a data migration, seeding. `COPY orders FROM '/path/orders.csv' CSV HEADER;` or, from an app, stream via `COPY orders FROM STDIN`. It is typically 5-10x faster than even multi-row `INSERT` for large volumes.

**Follow-up — Can you run `COPY FROM` a file from your application code?** `COPY FROM '/file'` reads a file on the *server's* filesystem and requires superuser or the `pg_read_server_files` role, so app code usually cannot use it. Instead use `COPY ... FROM STDIN` and stream the data over the connection (e.g. `pg-copy-streams` in Node, `copy_expert` in psycopg). That needs no special file access and works through the normal connection.

---

**Q: How do you update one table using values from another table in PostgreSQL?**

A: Use `UPDATE ... FROM`:

```sql
UPDATE order_items oi
SET   unit_price = p.price
FROM  products p
WHERE oi.product_id = p.id
  AND oi.unit_price <> p.price;
```

The `FROM` clause names the source table, and the `WHERE` clause both joins the two tables and filters which rows to touch. The analogous delete is `DELETE FROM ... USING ...`.

**Follow-up — What is the danger if the join in `UPDATE ... FROM` is not one-to-one?** If a target row matches multiple source rows, PostgreSQL does not error — it picks one matching source row arbitrarily and updates the target once with that value. The result is non-deterministic. Always ensure the join key is unique on the source side (or aggregate first), otherwise you silently get whichever value the executor happened to reach.

---

### Principal Level

**Q: You need to upsert 50,000 rows (insert new, update existing). Compare row-by-row `INSERT ... ON CONFLICT` in a loop versus a single set-based upsert, and describe the fastest correct approach.**

A: A loop of 50,000 single-row `INSERT ... ON CONFLICT` statements pays 50,000 parse/plan/round-trip cycles and, if each is its own transaction, 50,000 `fsync`s — minutes of work. The set-based approach collapses this into one statement. The fastest correct pattern is to pass the whole batch as arrays and `unnest` them into a derived table, then upsert in one shot:

```sql
INSERT INTO products (id, name, price)
SELECT * FROM unnest(
  $1::int[],          -- ids
  $2::text[],         -- names
  $3::numeric[]       -- prices
)
ON CONFLICT (id) DO UPDATE
SET name  = EXCLUDED.name,
    price = EXCLUDED.price
WHERE products.name <> EXCLUDED.name
   OR products.price <> EXCLUDED.price;   -- skip no-op updates
```

This sends one statement with three array parameters (dodging the 65,535-parameter limit that a multi-row `VALUES` list would hit), does one parse/plan, and one commit. The `WHERE` on the `DO UPDATE` avoids writing dead tuples for rows that did not actually change — which reduces bloat and WAL. For truly enormous batches, `COPY` into an `UNLOGGED` staging table and then a single `INSERT ... SELECT ... ON CONFLICT` from staging is even faster.

**Follow-up — Why does the order in which you supply the rows matter for a concurrent upsert?** When two transactions upsert overlapping key sets in different orders, each can hold a lock on a row the other needs, producing a deadlock. Sorting the batch by the conflict key before sending it gives every writer a consistent lock-acquisition order and eliminates that class of deadlock.

---

**Q: A nightly job must delete 5,000,000 rows from an 80,000,000-row `sessions` table. A single `DELETE` statement "works" in staging but causes an incident in production. Explain what goes wrong and how you fix it.**

A: A single 5M-row `DELETE` is one transaction, so it (1) holds row locks on all 5M rows until commit, blocking or slowing concurrent writers; (2) generates one giant WAL record that can saturate replication and delay standbys; (3) marks 5M tuples dead but cannot reclaim them until every snapshot older than the transaction is gone — meanwhile the transaction pins the `xmin` horizon, so `VACUUM` across the *whole database* cannot remove any dead tuples created during its (possibly long) run, causing table and index bloat; (4) if it is rolled back near the end, all the work is wasted.

The fix is chunked deletion — delete in bounded batches, each its own transaction:

```sql
DELETE FROM sessions
WHERE id IN (
  SELECT s.id
  FROM   sessions s
  JOIN   users u ON u.id = s.user_id
  WHERE  u.closed_at < NOW() - INTERVAL '30 days'
  LIMIT  10000
);
```

Loop this from the application until it deletes 0 rows, committing after each chunk and optionally sleeping briefly between chunks. Each chunk holds few locks, produces small WAL records replicas can keep up with, and — because it commits — advances the `xmin` horizon so autovacuum can reclaim space as you go. An index on `sessions(user_id)` (for the join) and on `users(closed_at)` (for the predicate) keeps chunk selection cheap.

**Follow-up — Why must each chunk be a separate transaction rather than one transaction with periodic `COMMIT` inside a plain function?** In PostgreSQL a plain `LANGUAGE sql`/`plpgsql` function runs inside a single surrounding transaction and cannot commit mid-way, so the locks and the `xmin` pin persist for the whole run — defeating the point. You need either application-side looping with a real commit per chunk, or a `PROCEDURE` invoked by `CALL` (procedures can `COMMIT` between chunks).

---

**Q: Design the fastest way to bulk-load 40,000,000 rows into a table that will ultimately have five secondary indexes and a per-row audit trigger, within a tight maintenance window.**

A: The cost of a naive load is dominated by index maintenance and trigger firing per row. The winning strategy is to remove that cost during the load and pay it once, in bulk, afterward:

1. Load into the table with **only the primary key** present — drop or defer the five secondary indexes and disable the audit trigger (`ALTER TABLE ... DISABLE TRIGGER`) first.
2. Use **`COPY FROM STDIN`**, not multi-row `INSERT` — `COPY` has the lowest per-row overhead and streams without the parameter limit.
3. If the table can be recreated on failure, make it `UNLOGGED` during the load (or set `wal_level` considerations aside for a fresh table created in the same transaction, which skips WAL for the load) to cut WAL volume drastically; then `SET LOGGED` afterward.
4. Raise `maintenance_work_mem` so the subsequent index builds are fast.
5. After the data is in, **`CREATE INDEX` for all five secondary indexes** — building an index once over sorted data is far cheaper than 40M incremental insertions into five B-trees.
6. Re-enable the trigger, and if the audit rows are needed for the loaded data, generate them in one set-based `INSERT ... SELECT` rather than per row.
7. Run **`ANALYZE`** so the planner has fresh statistics.

**Follow-up — Which single skipped step silently degrades the *next day's* query performance rather than the load itself?** Skipping `ANALYZE`. The load succeeds and the indexes exist, but the planner's statistics are stale/empty, so it misestimates row counts and picks bad plans (seq scans instead of index scans, wrong join orders). Queries that should be milliseconds become seconds — a failure that shows up in production traffic the next day, not in the load window.

---

## 15. Mental Model Checkpoint

1. **You have a 5-column table and want to insert rows with a single parameterized multi-row `INSERT`. Roughly how many rows can you send before you hit PostgreSQL's 65,535 bind-parameter limit, and how would you send 100,000 rows instead?**

2. **You run a multi-row `INSERT` with 1,000 `VALUES` rows and row #500 violates a unique constraint. How many of the 1,000 rows end up inserted, and why?**

3. **`COPY orders FROM '/data/orders.csv'` fails with a permissions error when your application runs it, but the same command works in `psql` as the DB superuser. What is different, and what should the application use instead?**

4. **In `UPDATE order_items oi SET unit_price = p.price FROM products p WHERE oi.product_id = p.id`, what happens if a single `order_items` row could match two different `products` rows? Does PostgreSQL raise an error?**

5. **A colleague deletes 5,000,000 rows in one `DELETE` and notices that `VACUUM` afterward reclaims almost no space until the transaction fully finishes and other long transactions end. Explain the `xmin` horizon reasoning.**

6. **Why does an `unnest`-based bulk upsert (arrays passed as parameters) scale to far larger batches than a multi-row `VALUES` upsert, even though both are "one statement"?**

7. **You are chunk-deleting 10,000 rows per transaction in a loop. Someone suggests wrapping the entire loop in a single `plpgsql` function with the loop inside. Why does that defeat the purpose, and what construct actually lets you commit between chunks?**

---

## 16. Quick Reference Card

```sql
-- MULTI-ROW INSERT (one round trip, one commit)
INSERT INTO products (name, category_id, price)
VALUES ('Keyboard', 3, 49.99),
       ('Mouse',    3, 24.99),
       ('Monitor',  3, 199.00);
-- Limit: rows * columns < 65,535 bind params. Sweet spot 500-1,000 rows/batch.
-- All-or-nothing: one constraint violation rolls back the whole statement.

-- COPY (fastest bulk load; lowest per-row overhead)
COPY orders FROM '/srv/data/orders.csv' WITH (FORMAT csv, HEADER);  -- server file, needs privilege
COPY orders FROM STDIN WITH (FORMAT csv);                           -- from app, no file privilege
-- App drivers: pg-copy-streams (Node), copy_expert / copy (psycopg)

-- UPDATE ... FROM (update target using another table)
UPDATE order_items oi
SET    unit_price = p.price
FROM   products p
WHERE  oi.product_id = p.id
  AND  oi.unit_price <> p.price;      -- skip no-op writes (less bloat/WAL)
-- Non-unique join => arbitrary source row chosen, NO error. Ensure unique source key.

-- DELETE ... USING (delete target using another table)
DELETE FROM sessions s
USING  users u
WHERE  s.user_id = u.id
  AND  u.closed_at < NOW() - INTERVAL '30 days';

-- UNNEST BULK UPSERT (arrays as params -> dodges 65,535 param limit)
INSERT INTO products (id, name, price)
SELECT * FROM unnest($1::int[], $2::text[], $3::numeric[])
ON CONFLICT (id) DO UPDATE
SET name = EXCLUDED.name, price = EXCLUDED.price
WHERE products.name <> EXCLUDED.name OR products.price <> EXCLUDED.price;
-- Sort batch by conflict key before sending => avoids deadlocks under concurrency.

-- CHUNKED DELETE (loop from app; commit per chunk)
DELETE FROM sessions
WHERE id IN (
  SELECT id FROM sessions
  WHERE last_seen_at < NOW() - INTERVAL '90 days'
  LIMIT 10000
);
-- Repeat until 0 rows. Each chunk = own txn => small locks, small WAL,
-- advances xmin horizon so autovacuum reclaims space as you go.
-- Commit between chunks: app loop, or CALL a PROCEDURE (procedures can COMMIT).

-- FAST BULK LOAD RECIPE (millions of rows)
-- 1. Drop secondary indexes; DISABLE audit triggers
-- 2. COPY FROM STDIN (not INSERT)
-- 3. UNLOGGED / high maintenance_work_mem during load
-- 4. CREATE INDEX afterward (build once > 40M incremental inserts)
-- 5. Re-enable triggers; ANALYZE  <-- skip this and next-day queries go slow

-- BATCH SIZING RULES OF THUMB
--   Multi-row INSERT : 500-1,000 rows
--   COPY             : stream freely, no batching needed
--   Chunked DML      : 1,000-10,000 rows/txn (balance lock time vs overhead)
```

---

## Connected Topics

- **INSERT Fundamentals**: The single-row `INSERT` that multi-row `VALUES`, `COPY`, and `unnest` loads all optimize away — bulk operations are the scaled-up form of the same statement.
- **INSERT ... ON CONFLICT (Upsert)**: The conflict-handling clause that the `unnest` bulk upsert relies on; understand `DO UPDATE`, `EXCLUDED`, and the no-op-skipping `WHERE` before batching it.
- **UPDATE and DELETE Fundamentals**: `UPDATE ... FROM` and `DELETE ... USING` are the multi-table extensions; the same predicate/lock rules apply, just at scale.
- **Transactions and Isolation**: Why each chunk should be its own transaction — commit boundaries, lock duration, and rollback cost drive every batch-sizing decision.
- **MVCC, VACUUM and the xmin Horizon**: The reason a giant single `DELETE` bloats the whole database — dead tuples, snapshot pinning, and why chunking lets autovacuum keep up.
- **Indexes and Index Maintenance**: Why you drop secondary indexes before a bulk load and rebuild after — per-row B-tree insertion versus one bulk `CREATE INDEX`.
- **WAL and Replication**: How bulk DML volume affects write-ahead logging and standby lag; `UNLOGGED` tables and why huge transactions stress replicas.
- **Query Planner Statistics (ANALYZE)**: The step that silently sabotages next-day performance if skipped after a large load — stale stats lead to bad plans.
- **Triggers**: Disabling and re-enabling per-row triggers around a bulk load, and replacing per-row audit writes with one set-based `INSERT ... SELECT`.
- **Deadlocks and Lock Ordering**: Why sorting a batch by its conflict key before an upsert eliminates a whole class of concurrent-writer deadlocks.
