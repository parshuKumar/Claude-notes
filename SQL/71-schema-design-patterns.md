# Topic 71 — Schema Design Patterns
### SQL Mastery Curriculum — Phase 10: PostgreSQL Specific Power Features

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a giant office building that houses many different companies.

The **building** is your PostgreSQL *database*. Inside the building there are **floors** — each floor is a PostgreSQL *schema*. On each floor there are **filing cabinets** — those are your *tables*.

Now, when a courier walks in and says "take this package to the Accounts cabinet," the receptionist has a problem: three different floors each have a cabinet labelled "Accounts." Which one? The building has a rule — a **search order** — pinned at the front desk: "If someone doesn't name the floor, check floor 3 first, then floor 8, then the shared lobby." That pinned list is the `search_path`. The lobby that everyone shares by default is the `public` schema.

Now stretch the analogy. Suppose you rent this building out to many client companies (tenants). You have three ways to keep them apart:

1. **One big open-plan floor, everyone shares the cabinets, but every folder has a coloured sticker with the company name on it.** Cheap, dense, but one careless clerk grabbing the wrong colour leaks data. This is *shared schema with a `tenant_id` column*.
2. **One floor per company — same cabinet layout, but walls between them.** Clear separation, but when you renovate you must repaint 500 identical floors. This is *schema-per-tenant*.
3. **A whole separate building per company.** Maximum isolation, but now you run 500 buildings, 500 heating systems, 500 fire inspections. This is *database-per-tenant*.

Schema design is the architecture of that building: where the walls go, how folders are labelled, whether you file by a meaningless ticket number (surrogate key) or by something real like a passport number (natural key), whether you shred old folders or just stamp them "VOID" (soft delete), and whether you keep a logbook of who touched what (audit_logs).

Get the architecture wrong and every later decision — migrations, backups, connection pooling, security — inherits the pain. That's why this topic sits in the "Power Features" phase: it's not one keyword, it's the load-bearing structure everything else rests on.

---

## 2. Connection to SQL Internals

A schema is not a physical container. Internally, PostgreSQL stores every schema as a single row in the `pg_namespace` catalog table. Every table, view, sequence, and function carries a `relnamespace` (or equivalent) column in `pg_class` pointing back to its schema's OID. So a schema is a **namespace label in the catalog**, nothing more — there is no separate file, no separate heap, no separate WAL stream per schema. Two tables in two different schemas live in the same tablespace directory, the same buffer pool, the same WAL, the same MVCC snapshot machinery.

This matters enormously for the multi-tenant decision:

- **Schema-per-tenant** means one row in `pg_namespace` per tenant but *N tables per tenant multiplied by tenant count* rows in `pg_class`, `pg_attribute`, `pg_index`, `pg_statistic`. 5,000 tenants × 30 tables = 150,000 relations. The **system catalogs themselves become large tables**, and every planner lookup (`RelationCacheInitializePhase*`, `relcache`/`syscache` probes) pays for it. Backup tooling that iterates `pg_dump` per object slows down. `autovacuum` has 150,000 relations to consider each cycle.

- **Database-per-tenant** means a separate entry in `pg_database`, a separate physical directory under `base/<oid>/`, a separate set of system catalogs, its own `pg_class`, its own visibility map and free space map files. Crucially, **each database has its own connection**: a PostgreSQL backend process is bound to exactly one database for its lifetime. You cannot query across databases in one connection without foreign data wrappers. This is why database-per-tenant is brutal on connection pooling — the pooler can't multiplex tenants over one server connection.

- **Shared schema with `tenant_id`** keeps everything in one set of catalog rows. The isolation lives entirely in the **B-tree index on `(tenant_id, ...)`** and in the `WHERE tenant_id = $1` predicate (or a Row-Level Security policy — Topic 72). MVCC, the visibility map, the heap pages — all shared. A tenant's rows are physically interleaved on the same heap pages as other tenants' rows unless you partition (Topic 70) by `tenant_id`.

The `search_path` is resolved at **parse/analyze time**, not execution time. When you write `SELECT * FROM orders`, the parser walks the `search_path` schemas in order, probing the `syscache` (`RELNAMENSP`) for a relation named `orders` in each namespace until it finds one. The resolved OID is then baked into the plan. This is why changing `search_path` invalidates cached plans and why `SET search_path` inside a function affects name resolution for that function's body.

Surrogate keys touch the internals too. A `bigint GENERATED ... AS IDENTITY` column is backed by a **sequence** — a special single-row relation (`pg_class.relkind = 'S'`) whose value is advanced with a lightweight lock, outside the transaction's MVCC (sequences don't roll back — a rolled-back INSERT still burns the sequence value, leaving gaps). A `uuid` primary key has no sequence, but its **random distribution destroys B-tree insert locality**: consecutive inserts land on random leaf pages, causing page splits, poor cache hit ratios, and index bloat — the reason UUIDv7 (time-ordered) exists.

---

## 3. Logical Execution Order Context

Schema design decisions influence *every* phase of the logical execution order, but the one that binds most tightly is name resolution, which happens **before** the logical query pipeline even begins:

```
[parse/analyze]  ← search_path resolves unqualified names to schema-qualified OIDs
FROM tenant_schema.orders          ← which physical table? decided here
JOIN ...                            ← RLS / tenant_id predicates injected here or in WHERE
WHERE tenant_id = $1                ← the isolation predicate for shared-schema designs
GROUP BY ...
HAVING ...
SELECT ...
ORDER BY ...
LIMIT ...
```

Key ordering consequences for this topic:

- **`search_path` resolution precedes everything.** If two schemas both contain `orders` and `search_path = tenant_42, public`, the query silently binds to `tenant_42.orders`. Change the session's `search_path` (as a connection pooler might, if misconfigured) and the *same SQL text* hits a *different table*. This is the single most dangerous footgun of schema-per-tenant behind a pooler.

- **The `tenant_id` predicate lives in `WHERE`** for shared-schema designs — which means it is applied *after* `FROM`/`JOIN` logically, and the planner must use an index on `tenant_id` to avoid scanning other tenants' rows. Forgetting it doesn't error; it returns *every tenant's* data. Row-Level Security (Topic 72) moves this predicate into an automatically-injected `WHERE` clause the application cannot forget.

- **Soft deletes add a mandatory predicate too**: `WHERE deleted_at IS NULL` must be applied in the `WHERE` phase of *every* read query, or logically-deleted rows leak back. Like `tenant_id`, this is a predicate you cannot forget — which is exactly why partial indexes (`WHERE deleted_at IS NULL`) and views exist to enforce it.

- **Audit logging is a write-side, not read-side, concern** — it happens in `BEFORE`/`AFTER` triggers that fire during the executor's `ModifyTable` step, entirely outside the SELECT pipeline.

The theme: good schema design turns "a predicate the developer must remember to type" into "a structural guarantee the engine enforces." That shift is the whole point.

---

## 4. What Is Schema Design?

Schema design is the set of decisions about **how data is namespaced, keyed, isolated between tenants, retained over its lifecycle, and audited** — decisions made before the first query and expensive to reverse afterward. It spans PostgreSQL's `CREATE SCHEMA` namespacing, the `search_path` resolution mechanism, multi-tenant isolation strategies, primary-key selection (surrogate vs natural, UUID vs bigint), soft-delete conventions, and audit-trail patterns.

### Annotated syntax breakdown — creating and using schemas

```sql
CREATE SCHEMA IF NOT EXISTS tenant_42       -- create a namespace
       AUTHORIZATION app_user;              -- owned by this role
--     │              │           │
--     │              │           └── role that owns objects created here
--     │              └── the schema (namespace) name; a row in pg_namespace
--     └── idempotent: no error if it already exists

SET search_path TO tenant_42, public;
--  │             │  │          │
--  │             │  │          └── fallback schema, searched second
--  │             │  └── primary schema, searched first for unqualified names
--  │             └── the ordered resolution list for THIS session
--  └── session-level (or use SET LOCAL for transaction-scoped)

SELECT * FROM orders;          -- resolves to tenant_42.orders (first match wins)
SELECT * FROM public.orders;   -- schema-qualified: bypasses search_path entirely
```

### Annotated syntax breakdown — surrogate key choices

```sql
CREATE TABLE orders (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
--            │      │         │      │  │          │
--            │      │         │      │  │          └── enforces uniqueness + NOT NULL, builds B-tree
--            │      │         │      │  └── the SQL-standard identity mechanism (backed by a sequence)
--            │      │         │      └── ALWAYS = app cannot override; BY DEFAULT = app may supply value
--            │      │         └── auto-generate the value on INSERT
--            │      └── the identity clause
--            └── 8-byte integer: range to 9.2 quintillion — the safe default for surrogate PKs

  public_id   uuid NOT NULL DEFAULT gen_random_uuid(),
--            │                     │
--            │                     └── random UUIDv4 (pgcrypto/core); use uuidv7() for time-ordered
--            └── 16 bytes; opaque, non-enumerable external identifier

  tenant_id   bigint NOT NULL,          -- the shared-schema isolation column
  deleted_at  timestamptz,              -- NULL = live row; non-NULL = soft-deleted
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now()
);
```

### Annotated syntax breakdown — the enforced-isolation indexes

```sql
-- Composite index leading with tenant_id: every tenant-scoped query uses it
CREATE INDEX orders_tenant_created_idx
  ON orders (tenant_id, created_at DESC)
  WHERE deleted_at IS NULL;             -- partial: index only live rows
--      │
--      └── smaller index, and it enforces the "always filter deleted" access path
```

`GENERATED ALWAYS AS IDENTITY` is the modern, standard-compliant replacement for `serial`/`bigserial` (which are legacy pseudo-types that create a sequence with awkward ownership semantics). Prefer identity columns in new designs.

---

## 5. Why Schema Design Mastery Matters in Production

1. **It is the least-reversible decision you make.** Changing a column filter is a code review. Migrating 5,000 tenants from schema-per-tenant to shared-schema is a multi-quarter project with data movement, downtime windows, and a rollback plan. The cost of a wrong schema decision compounds daily and is paid at the worst possible time — when you're already at scale.

2. **The `search_path` behind a pooler is a data-leak vector.** Transaction-mode poolers (PgBouncer) hand the same server connection to different clients between transactions. If your app sets `search_path` per request to select a tenant schema and the pooler doesn't reset it, request B can inherit request A's tenant context and read the wrong tenant's data. This is a real, shipped-in-production class of bug.

3. **`tenant_id` you forget to filter returns everyone's data.** In shared-schema designs, a missing `WHERE tenant_id = $1` is not a syntax error — it's a silent cross-tenant data breach. At 3am, under deadline, someone will forget. Row-Level Security (Topic 72) exists precisely because "remember to filter" is not a security control.

4. **UUID-vs-bigint decides your write throughput and index size.** Random UUIDv4 primary keys on a high-insert table cause index bloat, page splits, and a collapsing buffer-cache hit ratio. Teams discover at 50M rows that their "modern" UUIDv4 PK made writes 3–5× slower than a bigint would have, and the index is 40% bloated. By then the FK graph makes it hard to change.

5. **Soft deletes silently rot your queries and your unique constraints.** Every query must remember `WHERE deleted_at IS NULL`. Every unique constraint must become partial (`UNIQUE ... WHERE deleted_at IS NULL`) or a "deleted" row blocks re-creating the same natural key. Reporting queries double-count. Foreign keys point at logically-dead rows. Teams that bolt soft deletes on late spend months auditing every query.

6. **Migrations scale with your isolation model.** A shared-schema `ALTER TABLE orders ADD COLUMN ...` is one DDL statement. The same change in schema-per-tenant is *N* statements across *N* schemas, each taking a lock, and a partial failure leaves your tenants on different schema versions — a support and correctness nightmare. Your migration strategy is downstream of your isolation choice.

7. **Audit trails are a compliance requirement, not a nice-to-have.** SOC 2, HIPAA, PCI-DSS, and GDPR all require knowing who changed what and when. Retrofitting an `audit_logs` table and triggers onto a live system, backfilling nothing, and hoping is how teams fail audits. Designing it in from the start is cheap; adding it later is not.

---

## 6. Deep Technical Content

### 6.1 Schemas as Namespaces

A schema is a named container for tables, views, sequences, functions, types, and operators. Its jobs:

- **Prevent name collisions.** `billing.invoices` and `analytics.invoices` coexist.
- **Group by ownership / access.** Grant a role usage on one schema, not the whole database.
- **Organize a large database.** 400 tables in `public` is unnavigable; grouped into `auth`, `billing`, `catalog`, `reporting` it becomes legible.

```sql
CREATE SCHEMA billing;
CREATE SCHEMA analytics;

CREATE TABLE billing.invoices   (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, ...);
CREATE TABLE analytics.invoices (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, ...);  -- no conflict
```

Objects are addressed as `schema.object`. Without a schema qualifier, resolution falls to `search_path`.

### 6.2 The `public` Schema and `search_path`

Every fresh PostgreSQL database has a `public` schema. Historically, `public` was world-writable (any role could create objects in it) — a long-standing security wart. **As of PostgreSQL 15, the default is that ordinary users can no longer create objects in `public`**; only the database owner can, unless you re-grant `CREATE`. This is a deliberate hardening you should not undo.

The `search_path` is the ordered list of schemas consulted for unqualified names. Its default:

```sql
SHOW search_path;
-- "$user", public
```

`"$user"` is a special token that expands to a schema named after the current role, *if such a schema exists*; otherwise it is skipped. Then `public`. Resolution rules:

- **Reads** (referencing a table) probe each schema in order, first match wins.
- **Writes/creates** (`CREATE TABLE foo`) go into the **first** schema in the path that exists and is writable.

```sql
SET search_path TO tenant_42, shared, public;
CREATE TABLE widgets (...);   -- created in tenant_42 (first writable schema)
SELECT * FROM lookups;        -- found in shared if not in tenant_42
```

**Session vs transaction scope:**

```sql
SET search_path TO tenant_42;         -- rest of the SESSION (or connection)
SET LOCAL search_path TO tenant_42;   -- rest of the TRANSACTION only — auto-reverts on COMMIT/ROLLBACK
```

`SET LOCAL` is the pooler-safe form: it cannot leak past the transaction boundary. `SET` (session) is the footgun behind transaction-mode poolers.

**Function-scoped search_path:**

```sql
CREATE FUNCTION get_widget(id bigint) RETURNS widgets
LANGUAGE sql
SET search_path = tenant_42, public    -- pinned for this function's body
AS $$ SELECT * FROM widgets WHERE widget_id = id $$;
```

Pinning `search_path` on `SECURITY DEFINER` functions is a **mandatory security practice** — without it, a caller can prepend a malicious schema and hijack unqualified name resolution inside a privileged function.

### 6.3 Multi-Tenant Pattern 1 — Shared Schema with `tenant_id`

All tenants share the same tables; every tenant-owned row carries a `tenant_id` column, and every query filters on it.

```sql
CREATE TABLE orders (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id   bigint NOT NULL REFERENCES tenants(id),
  customer_id bigint NOT NULL,
  total_amount numeric(12,2) NOT NULL,
  status      text NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);

-- EVERY tenant-scoped index leads with tenant_id
CREATE INDEX orders_tenant_status_idx ON orders (tenant_id, status, created_at DESC);

-- EVERY query filters on tenant_id
SELECT * FROM orders WHERE tenant_id = $1 AND status = 'completed';
```

**Tradeoffs:**

| Dimension | Shared schema with tenant_id |
|-----------|------------------------------|
| Isolation | Weakest — logical only; a missing predicate leaks data. Mitigate with RLS (Topic 72). |
| Migrations | Best — one `ALTER TABLE` covers all tenants atomically. |
| Connection pooling | Best — one connection pool serves all tenants; no per-tenant `search_path` juggling. |
| Density / cost | Best — one set of catalog rows, one autovacuum surface, thousands of tenants per database. |
| Noisy-neighbor blast radius | Worst — one tenant's huge table or runaway query shares heap pages, buffer cache, and vacuum with everyone. |
| Per-tenant backup/restore | Hard — you must filter by tenant_id to extract or delete one tenant. |

This is the default choice for SaaS with many small tenants (B2C, freemium, long-tail B2B). Add RLS to make the isolation a structural guarantee rather than a convention.

### 6.4 Multi-Tenant Pattern 2 — Schema-per-Tenant

Each tenant gets its own schema containing an identical set of tables.

```sql
CREATE SCHEMA tenant_42;
CREATE TABLE tenant_42.orders (LIKE template.orders INCLUDING ALL);
-- ... repeat every table for every tenant

-- The app selects a tenant by setting search_path (ideally SET LOCAL, per transaction)
SET LOCAL search_path TO tenant_42;
SELECT * FROM orders;   -- resolves to tenant_42.orders
```

**Tradeoffs:**

| Dimension | Schema-per-tenant |
|-----------|-------------------|
| Isolation | Medium-strong — namespaces separate tenants; no tenant_id predicate needed; a `search_path` mistake still crosses tenants. |
| Migrations | Painful — N schemas × M tables; must loop DDL over every schema; partial failure leaves version skew. |
| Connection pooling | Awkward — `search_path` is connection state; with transaction poolers you must `SET LOCAL` every transaction, or the pooler leaks tenant context. |
| Catalog bloat | Bad at high tenant counts — `pg_class`/`pg_attribute`/`pg_statistic` grow to hundreds of thousands of rows; planner and autovacuum slow down. |
| Per-tenant backup/restore | Easy — `pg_dump --schema=tenant_42` extracts exactly one tenant. |
| Per-tenant customization | Possible — a tenant could have extra columns/tables (usually a mistake, but possible). |

Sweet spot: **tens to low-hundreds of larger tenants** where per-tenant backup/export matters and the catalog stays manageable. Beyond ~1,000 tenants the catalog and migration costs dominate.

### 6.5 Multi-Tenant Pattern 3 — Database-per-Tenant

Each tenant gets a separate database (optionally on shared or separate instances).

```sql
CREATE DATABASE tenant_42 TEMPLATE tenant_template;
-- The app connects to the specific database per tenant
```

**Tradeoffs:**

| Dimension | Database-per-tenant |
|-----------|---------------------|
| Isolation | Strongest — separate catalogs, separate files, no cross-tenant query possible in one connection; strong blast-radius containment. |
| Migrations | Painful at scale — run every migration against every database; orchestration required; version skew risk. |
| Connection pooling | Worst — a backend is bound to one database; you cannot multiplex tenants over one server connection. N databases × pool size can exhaust `max_connections` fast. |
| Resource cost | Highest — per-database overhead (catalogs, background workers' attention, WAL is shared but per-db bookkeeping isn't free). |
| Per-tenant backup/restore | Best — `pg_dump`/`pg_restore` of one database; trivial to move a tenant to another server. |
| Compliance / data residency | Best — a tenant's data can live on a specific instance/region. |

Sweet spot: **few, large, high-value tenants** (enterprise contracts) with strict isolation, per-tenant SLAs, or data-residency requirements. Connection-pool exhaustion is the practical ceiling; poolers like PgBouncer help but cannot fully hide the per-database binding.

### 6.6 Choosing an Isolation Model — Decision Summary

```
Many small tenants, cost-sensitive, one product version
  → Shared schema + tenant_id + RLS

Tens–hundreds of medium tenants, need per-tenant export/backup
  → Schema-per-tenant (watch catalog size, automate migrations)

Few large enterprise tenants, strict isolation / residency / SLAs
  → Database-per-tenant (budget for connection management)

Hybrid: shared schema for the long tail + dedicated DB for whales
  → Common at scale; the isolation model becomes a per-tenant attribute
```

The hybrid ("pod" or "cell" architecture) is what most large SaaS converge on: default new tenants to a shared-schema cell, promote a tenant to its own database when they get big or demand isolation. This is only feasible if your application's data-access layer treats "how do I reach this tenant" as a lookup, not a hardcoded assumption.

### 6.7 Surrogate vs Natural Keys

A **surrogate key** is a synthetic, meaningless identifier (a `bigint` sequence value or a `uuid`) whose only job is to be unique. A **natural key** is a column (or set) that already identifies the row in the business domain (email, ISO country code, SKU, SSN).

```sql
-- Natural key
CREATE TABLE countries (
  iso_code  char(2) PRIMARY KEY,   -- 'US', 'IN', 'GB' — stable, meaningful, small
  name      text NOT NULL
);

-- Surrogate key
CREATE TABLE users (
  id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- meaningless, stable
  email  text NOT NULL UNIQUE                              -- natural key kept as a UNIQUE constraint
);
```

**Why surrogate keys usually win for entity tables:**

- **Natural keys change.** People change email, companies rebrand SKUs, a "permanent" business code gets reissued. A changing PK cascades through every FK — a nightmare. Surrogate keys never change.
- **Composite natural keys bloat FKs.** If the natural key is three columns, every referencing table repeats all three, and every join compares all three. A single `bigint` FK is smaller and faster.
- **Surrogate keys are uniform.** Every table has an `id bigint` — predictable joins, predictable tooling.

**When natural keys are right:**

- **Stable, small, standardized codes**: ISO country/currency codes, US state abbreviations. These genuinely never change and are meaningful in queries.
- **Pure junction tables**: a many-to-many link table's natural key *is* the pair of FKs — `PRIMARY KEY (order_id, product_id)` is often better than adding a surrogate.
- **Immutable external identifiers you don't control but must dedupe on**.

**The standard compromise**: surrogate PK for identity and FKs, *plus* a `UNIQUE` constraint on the natural key for correctness and lookups. You get stable references and business-rule enforcement. Remember from Topic 04 (constraints) that `UNIQUE` builds a B-tree you can also use for lookups on the natural key.

### 6.8 UUID vs bigint Identity

Both are surrogate keys. The choice hinges on **insert locality, key size, and enumerability**.

**bigint IDENTITY:**
- 8 bytes. Monotonically increasing → new rows append to the *right edge* of the B-tree → excellent insert locality, minimal page splits, hot pages stay cached.
- Enumerable — `id=1001` implies `id=1002` exists. Exposing sequential IDs in URLs leaks business volume and enables scraping (IDOR-adjacent).
- Requires the database (or a coordinator) to assign the value — you don't know the ID until after INSERT (though `RETURNING id` gives it back immediately).
- Central bottleneck concern is minimal in practice; sequence increment is cheap and non-transactional.

**UUID (v4, random):**
- 16 bytes — 2× the storage of bigint, in the PK and in *every FK that references it* and *every index that includes it*. On a wide FK graph this adds up.
- **Random distribution destroys insert locality.** Each insert targets a random leaf page → constant page splits, poor cache hit ratio, index bloat. On high-insert tables this is a measurable throughput loss (often 2–5×) and the index can bloat 30–50%.
- Non-enumerable — safe to expose externally; no volume leakage.
- **Client-generatable** — the application (or even the browser) can mint the ID *before* the round-trip, which simplifies optimistic UI, offline-first, and multi-master/sharded writes where a central sequence is impractical.

**UUID (v7, time-ordered)** — the modern reconciliation. UUIDv7 embeds a millisecond timestamp in the high bits, so values are *roughly* monotonic → insert locality close to bigint, while retaining non-enumerability and client-generation. PostgreSQL 18 ships `uuidv7()`; earlier versions use an extension or application-side generation.

```sql
-- bigint identity: best write locality, enumerable
id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY

-- UUIDv4: non-enumerable, client-generatable, poor write locality
id uuid PRIMARY KEY DEFAULT gen_random_uuid()

-- UUIDv7: non-enumerable, client-generatable, good write locality (PG18+)
id uuid PRIMARY KEY DEFAULT uuidv7()
```

**Common production pattern — dual keys:** internal `bigint` surrogate PK for FKs and joins (fast, small), plus an external `uuid` (`public_id`) exposed in APIs and URLs (non-enumerable). You get bigint's write performance internally and UUID's opacity externally.

```sql
CREATE TABLE orders (
  id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,   -- internal, FKs point here
  public_id uuid NOT NULL DEFAULT gen_random_uuid() UNIQUE,    -- external API identifier
  ...
);
```

### 6.9 Soft Deletes

A soft delete marks a row as deleted instead of removing it, preserving history and enabling "undelete."

```sql
ALTER TABLE orders ADD COLUMN deleted_at timestamptz;   -- NULL = live, non-NULL = deleted

-- "Delete"
UPDATE orders SET deleted_at = now() WHERE id = $1;

-- Every read must exclude soft-deleted rows
SELECT * FROM orders WHERE tenant_id = $1 AND deleted_at IS NULL;
```

**Design choices:**
- `deleted_at timestamptz` (nullable) is better than `is_deleted boolean` — it records *when*, and a NULL/non-NULL check is as cheap as a boolean.
- Consider `deleted_by bigint` to record *who*.

**The three hard problems soft deletes create:**

1. **Every query must remember the filter.** Miss it and dead rows resurface. Enforce structurally with a view or RLS, not discipline.
   ```sql
   CREATE VIEW active_orders AS SELECT * FROM orders WHERE deleted_at IS NULL;
   ```
2. **Unique constraints break.** A soft-deleted `email = 'x@y.com'` still occupies the unique index, blocking a new signup with the same email. Fix with a **partial unique index**:
   ```sql
   CREATE UNIQUE INDEX users_email_live_uidx
     ON users (email) WHERE deleted_at IS NULL;   -- uniqueness only among live rows
   ```
3. **Indexes and stats degrade.** Dead rows accumulate forever, bloating tables and indexes and skewing planner statistics. Use **partial indexes** on live rows and periodically archive truly-old soft-deleted rows to cold storage.
   ```sql
   CREATE INDEX orders_tenant_created_live_idx
     ON orders (tenant_id, created_at DESC) WHERE deleted_at IS NULL;
   ```

Foreign keys are a subtlety: a hard `ON DELETE CASCADE` doesn't fire on a soft delete, so children remain "live" under a "deleted" parent. You must handle cascading soft deletes in application logic or triggers.

### 6.10 Audit Logs

An `audit_logs` table records who changed what, when, and how — for compliance, debugging, and forensics.

```sql
CREATE TABLE audit_logs (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  table_name  text        NOT NULL,
  row_id      bigint      NOT NULL,
  action      text        NOT NULL CHECK (action IN ('INSERT','UPDATE','DELETE')),
  actor_id    bigint,                          -- the user who made the change
  changed_at  timestamptz NOT NULL DEFAULT now(),
  old_data    jsonb,                           -- row image before (NULL for INSERT)
  new_data    jsonb                            -- row image after  (NULL for DELETE)
);
CREATE INDEX audit_logs_lookup_idx ON audit_logs (table_name, row_id, changed_at DESC);
```

**Two capture strategies:**

1. **Trigger-based (database-enforced)** — a trigger on each audited table writes to `audit_logs`. Cannot be bypassed by any code path (app, psql, migration); captures the true before/after images via `OLD`/`NEW`.

   ```sql
   CREATE FUNCTION audit_trigger() RETURNS trigger LANGUAGE plpgsql AS $$
   BEGIN
     INSERT INTO audit_logs(table_name, row_id, action, actor_id, old_data, new_data)
     VALUES (
       TG_TABLE_NAME,
       COALESCE(NEW.id, OLD.id),
       TG_OP,
       current_setting('app.current_user_id', true)::bigint,  -- set per transaction by the app
       CASE WHEN TG_OP IN ('UPDATE','DELETE') THEN to_jsonb(OLD) END,
       CASE WHEN TG_OP IN ('UPDATE','INSERT') THEN to_jsonb(NEW) END
     );
     RETURN COALESCE(NEW, OLD);
   END $$;

   CREATE TRIGGER orders_audit
     AFTER INSERT OR UPDATE OR DELETE ON orders
     FOR EACH ROW EXECUTE FUNCTION audit_trigger();
   ```

   The app passes the actor via `SET LOCAL app.current_user_id = '...'` at the start of each transaction; the trigger reads it with `current_setting(..., true)` (the `true` = "don't error if unset").

2. **Application-level** — the app writes audit rows in the same transaction as the change. More flexible (can log business context the DB doesn't know) but *bypassable* — any code path that skips the audit call leaves a gap. Weaker for compliance.

**Scaling audit_logs:** it is append-only and grows without bound — the textbook case for **range partitioning by `changed_at`** (Topic 70). Partition monthly, keep hot months on fast storage, detach and archive old partitions. Consider `jsonb` diffs (only changed keys) instead of full row images to cut volume. For extreme volume, stream to an external log store (e.g., Kafka → data lake) instead of an in-database table.

---

## 7. EXPLAIN — Tenant Isolation in the Plan

The most important EXPLAIN reading for this topic is proving that a tenant-scoped query touches *only* the tenant's rows via the leading-column index, not a full scan across all tenants.

### Good: composite index on `(tenant_id, ...)` drives an Index Scan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_amount, created_at
FROM orders
WHERE tenant_id = 42 AND status = 'completed'
ORDER BY created_at DESC
LIMIT 50;
```

```
Limit  (cost=0.43..38.20 rows=50 width=24)
       (actual time=0.03..0.21 rows=50 loops=1)
  ->  Index Scan using orders_tenant_status_idx on orders
        (cost=0.43..1140.55 rows=1510 width=24)
        (actual time=0.03..0.19 rows=50 loops=1)
        Index Cond: ((tenant_id = 42) AND (status = 'completed'))
  Buffers: shared hit=54
Planning Time: 0.14 ms
Execution Time: 0.25 ms
```

**Reading it:**
- `Index Cond: (tenant_id = 42) AND (status = 'completed')` — both predicates are satisfied *inside the index*, so the scan visits only this tenant's completed orders.
- The index leads with `tenant_id`, then `status`, then `created_at DESC` — so the ORDER BY is satisfied by the index order too (no separate Sort node).
- `LIMIT 50` stops the scan after 50 index entries — `actual rows=50`, `Buffers: shared hit=54`. Cheap and bounded regardless of total table size.

### Bad: `tenant_id` not the leading column → cross-tenant Seq Scan

```sql
-- Index is on (status, created_at) — tenant_id NOT leading
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_amount FROM orders
WHERE tenant_id = 42 AND status = 'completed';
```

```
Seq Scan on orders  (cost=0.00..48210.00 rows=1490 width=20)
                    (actual time=2.1..312.6 rows=1503 loops=1)
  Filter: ((tenant_id = 42) AND (status = 'completed'))
  Rows Removed by Filter: 1998497
Buffers: shared hit=812 read=25400
Planning Time: 0.12 ms
Execution Time: 312.9 ms
```

**Reading it:**
- `Seq Scan` — no usable index leads with `tenant_id`, so PostgreSQL reads the *entire* table.
- `Rows Removed by Filter: 1998497` — it scanned ~2M rows belonging to *other tenants* and threw them away. This is the shape of a query that works correctly but scans everyone's data.
- `Buffers: shared ... read=25400` — 25,400 pages read from disk. 312ms vs 0.25ms — a 1,000× difference driven entirely by the index's leading column.

**The lesson:** in shared-schema multi-tenancy, `tenant_id` must be the **leading column** of the indexes your queries use. An index on `(status, created_at)` cannot efficiently isolate one tenant.

### Soft-delete partial index in the plan

```sql
EXPLAIN (ANALYZE)
SELECT id FROM orders
WHERE tenant_id = 42 AND deleted_at IS NULL
ORDER BY created_at DESC LIMIT 20;
```

```
Limit  (cost=0.43..12.30 rows=20 width=8) (actual time=0.02..0.06 rows=20 loops=1)
  ->  Index Scan using orders_tenant_created_live_idx on orders
        (actual time=0.02..0.05 rows=20 loops=1)
        Index Cond: (tenant_id = 42)
Planning Time: 0.10 ms
Execution Time: 0.08 ms
```

**Reading it:** the partial index `... WHERE deleted_at IS NULL` already excludes soft-deleted rows, so `deleted_at IS NULL` appears as an *implicit* condition satisfied by the index's very definition — there's no `Filter: (deleted_at IS NULL)` line and no dead rows are visited. The index is also smaller (only live rows), improving cache efficiency.

---

## 8. Query Examples

### Example 1 — Basic: Tenant-scoped read with soft-delete filter

```sql
-- The canonical shared-schema read: always tenant-filtered, always live-only.
SELECT
  id,
  public_id,          -- external identifier exposed to the API
  total_amount,
  status,
  created_at
FROM orders
WHERE tenant_id = 42          -- isolation predicate (leading index column)
  AND deleted_at IS NULL      -- soft-delete predicate
ORDER BY created_at DESC
LIMIT 100;
```

### Example 2 — Intermediate: Partial unique index enabling email reuse after soft delete

```sql
-- Users can be soft-deleted, and later a new user may claim the same email.
CREATE TABLE users (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  public_id  uuid NOT NULL DEFAULT gen_random_uuid() UNIQUE,
  tenant_id  bigint NOT NULL,
  email      text   NOT NULL,
  deleted_at timestamptz
);

-- Uniqueness of email applies ONLY among live rows, and per tenant.
CREATE UNIQUE INDEX users_tenant_email_live_uidx
  ON users (tenant_id, email)
  WHERE deleted_at IS NULL;

-- Now this sequence succeeds:
UPDATE users SET deleted_at = now() WHERE tenant_id = 42 AND email = 'a@ex.com';  -- soft delete
INSERT INTO users (tenant_id, email) VALUES (42, 'a@ex.com');                     -- allowed: old row is not "live"
```

### Example 3 — Production Grade: Auditable soft delete with actor context

```sql
-- Context: orders table, ~40M rows, shared-schema multi-tenant, ~5k tenants.
-- Indexes:
--   orders_pkey (id)
--   orders_tenant_created_live_idx (tenant_id, created_at DESC) WHERE deleted_at IS NULL
--   audit_logs_lookup_idx (table_name, row_id, changed_at DESC)
--   audit_logs is monthly-range-partitioned on changed_at (Topic 70)
-- audit_trigger fires AFTER UPDATE and writes the before/after image to audit_logs.
-- Perf expectation: the UPDATE is a single-row PK update (< 1ms) plus one audit INSERT;
-- the whole transaction is a handful of buffer touches.

BEGIN;

-- Establish who is performing the action, for the audit trigger to read.
SET LOCAL app.current_user_id = '9981';          -- pooler-safe: transaction-scoped

-- Soft-delete a single order; the AFTER UPDATE trigger records old/new images.
UPDATE orders
SET    deleted_at = now(),
       updated_at = now()
WHERE  tenant_id = 42
  AND  id = 8842301
  AND  deleted_at IS NULL          -- idempotent: no-op if already deleted
RETURNING id, public_id, deleted_at;

COMMIT;
```

```sql
EXPLAIN (ANALYZE, BUFFERS)
UPDATE orders SET deleted_at = now()
WHERE tenant_id = 42 AND id = 8842301 AND deleted_at IS NULL;
```

```
Update on orders  (cost=0.44..8.46 rows=0 width=0)
                  (actual time=0.09..0.09 rows=0 loops=1)
  ->  Index Scan using orders_pkey on orders  (cost=0.44..8.46 rows=1 width=14)
        (actual time=0.04..0.05 rows=1 loops=1)
        Index Cond: (id = 8842301)
        Filter: ((tenant_id = 42) AND (deleted_at IS NULL))
  Buffers: shared hit=6
Planning Time: 0.15 ms
Trigger orders_audit: time=0.11 calls=1
Execution Time: 0.28 ms
```

**Reading it:** the update finds the row by primary key (`Index Cond: id = 8842301`), verifies `tenant_id` and liveness as a cheap `Filter`, touches 6 buffers, and the `Trigger orders_audit` line shows the audit write cost (0.11ms) folded into the same transaction. Total 0.28ms — a soft delete plus a full audit trail is nearly free at row granularity.

---

## 9. Wrong → Right Patterns

### Wrong 1: Forgetting the `tenant_id` predicate — silent cross-tenant leak

```sql
-- WRONG: no tenant filter — returns EVERY tenant's orders
SELECT id, total_amount, status
FROM orders
WHERE status = 'completed';
-- RESULT: rows from all 5,000 tenants. No error. A data breach that looks like a working query.
```

Why it's wrong at the execution level: there is no `tenant_id` in the `WHERE` clause, so the planner has no isolation predicate to apply. The query is *correct SQL* and returns *correct rows for what it asked* — it just asked for everyone. Nothing in the engine flags this as dangerous.

```sql
-- RIGHT (application discipline): always include the tenant predicate
SELECT id, total_amount, status
FROM orders
WHERE tenant_id = $1 AND status = 'completed';

-- BETTER (structural guarantee): Row-Level Security (Topic 72) injects the
-- predicate automatically so it CANNOT be forgotten.
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.tenant_id')::bigint);
-- Now even the "WRONG" query above returns only the current tenant's rows.
```

### Wrong 2: `SET search_path` (session) behind a transaction pooler

```sql
-- WRONG: session-level SET leaks across pooled connections
SET search_path TO tenant_42;   -- persists on the server connection
SELECT * FROM orders;
-- With PgBouncer in transaction mode, the NEXT client to borrow this server
-- connection inherits search_path = tenant_42 and reads the WRONG tenant.
```

Why it's wrong at the execution level: `SET search_path` mutates *session state* on the physical backend. A transaction-mode pooler returns that backend to the pool after `COMMIT`, still carrying the mutated `search_path`. The next transaction — possibly a different tenant's request — resolves unqualified names against `tenant_42`.

```sql
-- RIGHT: transaction-scoped, auto-reverts on COMMIT/ROLLBACK
BEGIN;
SET LOCAL search_path TO tenant_42;   -- cannot outlive this transaction
SELECT * FROM orders;
COMMIT;   -- search_path reverts; the pooled connection is clean for the next borrower
```

### Wrong 3: Plain unique constraint with soft deletes

```sql
-- WRONG: full unique constraint blocks email reuse after soft delete
CREATE TABLE users (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email text NOT NULL UNIQUE,      -- covers deleted rows too
  deleted_at timestamptz
);
UPDATE users SET deleted_at = now() WHERE email = 'a@ex.com';
INSERT INTO users (email) VALUES ('a@ex.com');
-- ERROR: duplicate key value violates unique constraint "users_email_key"
-- The soft-deleted row still occupies the unique index.
```

Why it's wrong at the execution level: a `UNIQUE` constraint builds a B-tree over *all* rows, deleted or not. The soft-deleted row's `email` entry is still present in the index, so the insert collides.

```sql
-- RIGHT: partial unique index over live rows only
CREATE TABLE users (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email text NOT NULL,
  deleted_at timestamptz
);
CREATE UNIQUE INDEX users_email_live_uidx ON users (email) WHERE deleted_at IS NULL;
-- Now the deleted row is NOT in the (partial) index, so reuse is allowed.
```

### Wrong 4: Random UUIDv4 PK on a high-insert table

```sql
-- WRONG: random UUID PK on a table taking thousands of inserts/sec
CREATE TABLE events (
  id   uuid PRIMARY KEY DEFAULT gen_random_uuid(),   -- random → scattered inserts
  ...
);
-- Each insert lands on a random B-tree leaf → constant page splits,
-- collapsing cache hit ratio, 30–50% index bloat, 2–5× slower writes at scale.
```

Why it's wrong at the execution level: the B-tree is ordered by key. Random keys mean each insert targets an unpredictable leaf page that is likely not in cache, forcing a read, then often a page split (a write-amplifying restructure). Sequential keys append to the same hot right-edge page.

```sql
-- RIGHT (PG18+): time-ordered UUIDv7 — non-enumerable AND insert-local
CREATE TABLE events (
  id   uuid PRIMARY KEY DEFAULT uuidv7(),
  ...
);

-- RIGHT (any version): bigint internally, UUID only as external identifier
CREATE TABLE events (
  id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- append-only, cache-friendly
  public_id uuid NOT NULL DEFAULT gen_random_uuid() UNIQUE    -- opaque external ID
);
```

### Wrong 5: Running one migration and assuming all schema-per-tenant tenants got it

```sql
-- WRONG: schema-per-tenant, but the migration only altered ONE schema
ALTER TABLE tenant_42.orders ADD COLUMN refund_reason text;
-- Tenants 1..41 and 43..5000 still lack the column.
-- The application now crashes for every tenant except 42, OR silently
-- diverges — some tenants on schema v2, most on v1.
```

Why it's wrong: schema-per-tenant has *N* physical copies of every table. DDL is not global; it must be applied per schema. A loop that fails halfway leaves version skew.

```sql
-- RIGHT: iterate every tenant schema, ideally idempotently and transactionally-per-schema
DO $$
DECLARE s text;
BEGIN
  FOR s IN SELECT nspname FROM pg_namespace WHERE nspname LIKE 'tenant_%'
  LOOP
    EXECUTE format(
      'ALTER TABLE %I.orders ADD COLUMN IF NOT EXISTS refund_reason text', s
    );
  END LOOP;
END $$;
-- Better still: a migration tool that tracks per-schema version and can resume.
```

---

## 10. Performance Profile

### Isolation-model cost at scale

| Metric | Shared schema | Schema-per-tenant | Database-per-tenant |
|--------|---------------|-------------------|---------------------|
| Relations in catalog (30 tables) | 30 | 30 × tenants | 30 per DB (separate catalogs) |
| Catalog pressure at 5k tenants | negligible | ~150k `pg_class` rows — planner/relcache slowdown | isolated per DB |
| `ALTER TABLE` cost | 1 statement | N statements + N locks | N statements across N DBs |
| Connection multiplexing | 1 pool, all tenants | 1 pool + `SET LOCAL` per txn | 1 pool *per database* — exhausts `max_connections` fast |
| Noisy-neighbor isolation | weak (shared buffers/vacuum) | medium | strong |
| Per-tenant restore | filter by tenant_id (hard) | `pg_dump --schema` (easy) | `pg_dump` DB (easiest) |

### Key size ripple effect (bigint vs UUID)

At 100M rows with 3 foreign keys per row:

| | bigint PK+FKs | UUIDv4 PK+FKs |
|---|---|---|
| PK size (heap) | 8 B/row = ~0.8 GB | 16 B/row = ~1.6 GB |
| PK index | ~2.1 GB | ~3.5 GB (plus 30–50% bloat from random inserts) |
| 3 FK columns | 24 B/row = ~2.4 GB | 48 B/row = ~4.8 GB |
| Insert throughput | baseline | ~0.3–0.5× (random-leaf page splits, cache misses) |
| Join comparison | 8-byte integer compare | 16-byte compare |

UUIDv7 recovers most of the *throughput* loss (locality) but not the *size* cost (still 16 bytes everywhere).

### Soft-delete degradation over time

Soft-deleted rows never leave the heap. At 100M rows with 20% soft-deleted:

- The table carries 20M dead-but-visible rows — heap 20% larger, sequential scans 20% slower.
- **Partial indexes** (`WHERE deleted_at IS NULL`) index only the 80M live rows — the index stays lean and its planner statistics reflect live data.
- Without partial indexes, every index carries the dead rows too, and every index scan pays for them.
- Mitigation at extreme scale: **archive** old soft-deleted rows to a cold `orders_archive` table (or a detached partition) on a schedule, keeping the hot table lean.

### audit_logs growth

Append-only and unbounded — the defining case for **range partitioning by `changed_at`** (Topic 70). Monthly partitions let you:
- Keep the current month hot and well-cached; older months on cheaper storage.
- `DETACH` and archive/drop old partitions in O(1) (a catalog operation) instead of a massive `DELETE` that bloats and vacuums.
- Keep per-partition indexes small.

Store `jsonb` *diffs* (changed keys only) rather than full `OLD`/`NEW` images to cut volume by 5–20× on wide tables.

### Optimization checklist for this topic

- Lead every tenant-scoped index with `tenant_id`.
- Make all "live-row" indexes partial: `WHERE deleted_at IS NULL`.
- Make natural-key unique constraints partial when soft deletes exist.
- Prefer `bigint` identity internally; expose UUID (v4 or v7) only as an external ID.
- Partition `audit_logs` (and other append-only tables) by time.
- Behind a pooler, use `SET LOCAL`, never session `SET`, for `search_path` and audit context.

---

## 11. Node.js Integration

### 11.1 Tenant-scoped query with the pg pool

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Shared-schema: tenant_id is ALWAYS a bound parameter, never string-interpolated.
async function getCompletedOrders(tenantId, limit = 100) {
  const { rows } = await pool.query(
    `SELECT id, public_id, total_amount, status, created_at
       FROM orders
      WHERE tenant_id = $1
        AND deleted_at IS NULL
        AND status = 'completed'
      ORDER BY created_at DESC
      LIMIT $2`,
    [tenantId, limit]           // $1, $2 — parameterized, injection-safe
  );
  return rows;
}
```

### 11.2 Setting audit + tenant context safely with `SET LOCAL` in a transaction

```javascript
// A helper that runs work inside a transaction with tenant + actor context set.
// SET LOCAL is pooler-safe: it auto-reverts on COMMIT/ROLLBACK, so a borrowed
// PgBouncer connection never leaks context to the next request.
async function withTenantContext(tenantId, actorId, fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    // Parameterizing SET LOCAL requires set_config(); it accepts $ params.
    await client.query(
      `SELECT set_config('app.tenant_id', $1, true),   -- true = LOCAL (transaction)
              set_config('app.current_user_id', $2, true)`,
      [String(tenantId), String(actorId)]
    );
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();          // returned to pool with context already reverted
  }
}

// Usage: the audit trigger reads app.current_user_id; RLS reads app.tenant_id.
await withTenantContext(42, 9981, async (client) => {
  await client.query(
    `UPDATE orders SET deleted_at = now()
      WHERE tenant_id = $1 AND id = $2 AND deleted_at IS NULL`,
    [42, 8842301]
  );
});
```

Note: `SET LOCAL search_path = $1` cannot bind a parameter directly (it's a utility command), which is why `set_config('search_path', $1, true)` is the parameterized form. The same applies to setting tenant/actor GUCs.

### 11.3 Client-generated UUID for optimistic UI

```javascript
import { randomUUID } from 'node:crypto';

// The client mints the external id before the round-trip — no need to wait for
// the DB to assign it. Works because public_id is a UUID the app controls.
async function createOrder(tenantId, customerId, amount) {
  const publicId = randomUUID();     // UUIDv4, generated app-side
  const { rows } = await pool.query(
    `INSERT INTO orders (tenant_id, public_id, customer_id, total_amount, status)
     VALUES ($1, $2, $3, $4, 'pending')
     RETURNING id, public_id, created_at`,   // id is the internal bigint
    [tenantId, publicId, customerId, amount]
  );
  return rows[0];
}
```

### 11.4 Schema-per-tenant routing (when you must)

```javascript
// If you run schema-per-tenant, resolve the schema per transaction with SET LOCAL.
// Validate the schema name against an allowlist — it cannot be a bound parameter,
// so an unvalidated value here is an injection vector.
const TENANT_SCHEMA = /^tenant_\d+$/;

async function queryInTenantSchema(schema, sqlText, params) {
  if (!TENANT_SCHEMA.test(schema)) throw new Error('invalid tenant schema');
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query(`SET LOCAL search_path TO ${schema}`); // validated, not user-free-text
    const res = await client.query(sqlText, params);
    await client.query('COMMIT');
    return res.rows;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}
```

**Do ORMs handle this?** Partially. All major ORMs parameterize `tenant_id` filters fine. None *automatically* enforce a tenant predicate on every query — that's why RLS (Topic 72) is the real safety net. For `SET LOCAL`/`set_config` context, most ORMs expose a raw-query escape hatch or a per-transaction hook; the tenant-context pattern above is typically implemented in a repository/middleware layer, not by the ORM itself.

---

## 12. ORM Comparison

The relevant question for schema design is: **can the ORM model `bigint`+`uuid` dual keys, express soft-delete filtering globally, enforce a tenant predicate, and set per-transaction context (for RLS/audit)?**

### Prisma

**Can Prisma do it?** Partially. Prisma models surrogate keys well, has soft-delete *middleware*, but has weak native multi-tenant enforcement.

```typescript
// schema.prisma
model Order {
  id        BigInt    @id @default(autoincrement())   // bigint identity
  publicId  String    @unique @default(uuid()) @map("public_id") @db.Uuid
  tenantId  BigInt    @map("tenant_id")
  deletedAt DateTime? @map("deleted_at")
  @@index([tenantId, createdAt(sort: Desc)])
}

// Soft-delete + tenant filter via a Client Extension (modern) or middleware (legacy).
const prisma = new PrismaClient().$extends({
  query: {
    order: {
      async findMany({ args, query }) {
        args.where = { ...args.where, deletedAt: null };  // force live-only
        return query(args);
      },
    },
  },
});
```

**Where it breaks:** the extension only guards the models/operations you wire up — a raw query or a forgotten model bypasses it. `BigInt` surfaces as JS `BigInt` (serialization friction with JSON). For real tenant isolation you still need RLS + `set_config` via `$executeRaw` at the start of each interactive transaction.

**Verdict:** Good for surrogate-key modeling; soft-delete via extensions is workable but bypassable; lean on RLS for tenancy, not Prisma.

### Drizzle

**Can Drizzle do it?** Yes, with explicit, typed control — and it's honest about not hiding anything.

```typescript
import { pgTable, bigint, uuid, timestamp } from 'drizzle-orm/pg-core';
import { sql, eq, and, isNull } from 'drizzle-orm';

export const orders = pgTable('orders', {
  id:        bigint('id', { mode: 'bigint' }).primaryKey().generatedAlwaysAsIdentity(),
  publicId:  uuid('public_id').defaultRandom().notNull().unique(),
  tenantId:  bigint('tenant_id', { mode: 'bigint' }).notNull(),
  deletedAt: timestamp('deleted_at', { withTimezone: true }),
});

// Soft-delete + tenant filter are explicit predicates (no magic).
const rows = await db.select().from(orders)
  .where(and(eq(orders.tenantId, 42n), isNull(orders.deletedAt)));

// Per-transaction context for RLS/audit:
await db.transaction(async (tx) => {
  await tx.execute(sql`SELECT set_config('app.tenant_id', ${'42'}, true)`);
  await tx.update(orders).set({ deletedAt: new Date() })
    .where(and(eq(orders.tenantId, 42n), eq(orders.id, 8842301n)));
});
```

**Where it breaks:** no built-in global soft-delete — you must add the `isNull(deletedAt)` predicate yourself (a helper wrapper is the common fix). That explicitness is arguably a feature.

**Verdict:** Best fit for this topic — precise control over key types, `set_config`, and partial-index-friendly predicates, with no hidden behavior.

### Sequelize

**Can Sequelize do it?** Yes — it has *native* soft deletes (`paranoid`) and default scopes.

```javascript
const Order = sequelize.define('Order', {
  id:       { type: DataTypes.BIGINT, primaryKey: true, autoIncrement: true },
  publicId: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, unique: true },
  tenantId: { type: DataTypes.BIGINT, allowNull: false },
}, {
  paranoid: true,          // soft delete: destroy() sets deletedAt; queries auto-filter it
  deletedAt: 'deleted_at',
  defaultScope: { where: {} },  // add a tenant scope per-request (below)
});

// Tenant scoping is typically applied per request via a scope or hook.
const scoped = Order.scope({ method: ['forTenant', tenantId] });
await scoped.findAll();   // adds WHERE tenant_id = ?
```

**Where it breaks:** `paranoid` mode's auto-filter is convenient but its interaction with unique constraints is exactly the partial-index problem — Sequelize doesn't create partial unique indexes for you; you add them by hand in a migration. `paranoid: true` can be bypassed with `{ paranoid: false }`, and raw queries ignore it.

**Verdict:** Strongest *native* soft-delete ergonomics of the five, but you still hand-craft partial unique indexes and tenant enforcement.

### TypeORM

**Can TypeORM do it?** Yes — it has `@DeleteDateColumn` for soft deletes and `softDelete()`/`restore()`.

```typescript
@Entity()
export class Order {
  @PrimaryGeneratedColumn({ type: 'bigint' })
  id: string;                          // bigint surfaces as string in TypeORM

  @Column({ type: 'uuid', default: () => 'gen_random_uuid()', unique: true })
  publicId: string;

  @Column({ type: 'bigint' })
  tenantId: string;

  @DeleteDateColumn({ name: 'deleted_at', type: 'timestamptz' })
  deletedAt?: Date;                    // softRemove/softDelete set this; finds auto-filter it
}

// tenant context / RLS:
await dataSource.transaction(async (m) => {
  await m.query(`SELECT set_config('app.tenant_id', $1, true)`, ['42']);
  await m.getRepository(Order).softDelete({ tenantId: '42', id: '8842301' });
});
```

**Where it breaks:** `@DeleteDateColumn` auto-filters `find*` calls but *not* `createQueryBuilder` unless you call `.withDeleted()` awareness correctly — easy to accidentally include or exclude dead rows. `bigint` maps to `string`. Partial unique indexes for the soft-delete case are manual.

**Verdict:** Good native soft-delete support comparable to Sequelize; watch the QueryBuilder-vs-repository inconsistency.

### Knex

**Can Knex do it?** Yes — it's a query builder, so everything is explicit and nothing is enforced for you.

```javascript
// Migration: dual keys + partial unique + partial index
await knex.schema.createTable('orders', (t) => {
  t.bigIncrements('id').primary();
  t.uuid('public_id').notNullable().defaultTo(knex.raw('gen_random_uuid()')).unique();
  t.bigInteger('tenant_id').notNullable();
  t.timestamp('deleted_at', { useTz: true });
});
await knex.raw(`CREATE UNIQUE INDEX users_email_live_uidx
                ON users (email) WHERE deleted_at IS NULL`);   // partial: raw SQL

// Query: predicates are explicit
const rows = await knex('orders')
  .where({ tenant_id: 42 })
  .whereNull('deleted_at')
  .orderBy('created_at', 'desc');

// Context in a transaction:
await knex.transaction(async (trx) => {
  await trx.raw(`SELECT set_config('app.tenant_id', ?, true)`, ['42']);
  await trx('orders').where({ tenant_id: 42, id: 8842301 }).update({ deleted_at: knex.fn.now() });
});
```

**Where it breaks:** zero enforcement — no global soft-delete, no tenant guard, no key-type opinion. Everything is your responsibility, which is both the risk and the point.

**Verdict:** Most transparent; partial indexes and `set_config` are trivial via `.raw()`. Wrap the soft-delete/tenant predicates in helpers to avoid omissions.

### ORM Summary Table

| ORM | bigint+uuid dual key | Native soft delete | Partial unique index | Tenant enforcement | `set_config` for RLS/audit |
|-----|---------------------|--------------------|--------------------|--------------------|----------------------------|
| Prisma | Yes (`BigInt` friction) | Extension/middleware (bypassable) | Manual (raw) | None native — use RLS | `$executeRaw` |
| Drizzle | Yes, clean | Manual predicate | Manual (typed) | Manual — use RLS | `sql\`set_config\`` |
| Sequelize | Yes | Yes (`paranoid`) | Manual | Scopes/hooks | raw query |
| TypeORM | Yes (`bigint`→string) | Yes (`@DeleteDateColumn`) | Manual | Manual — use RLS | `.query()` |
| Knex | Yes | No (helper) | Trivial (`.raw`) | None (helper) | `.raw()` |

The consistent takeaway: **no ORM safely enforces tenant isolation for you** — that belongs to Row-Level Security (Topic 72). ORMs vary mainly in soft-delete ergonomics; partial indexes are always hand-written.

---

## 13. Practice Exercises

### Exercise 1 — Easy

You have a shared-schema table:

```sql
CREATE TABLE products (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id  bigint NOT NULL,
  sku        text NOT NULL,
  name       text NOT NULL,
  deleted_at timestamptz
);
```

1. Write the index that makes tenant-scoped, live-only listing queries fast (ordered by name).
2. Write a `UNIQUE` constraint that enforces "SKU is unique per tenant, but only among live rows" so a soft-deleted SKU can be reused.
3. Write the read query that lists a tenant's live products ordered by name.

```sql
-- Write your queries here
```

### Exercise 2 — Medium (combines soft delete + audit + prior topics)

Using `orders(id, tenant_id, status, total_amount, deleted_at, updated_at)` and the `audit_logs` table from section 6.10:

1. Write a single transaction that soft-deletes order `8842301` for tenant `42`, sets the audit actor to user `9981` in a pooler-safe way, and is idempotent (a second run does nothing).
2. Then write a query against `audit_logs` that returns the full change history (action, actor, changed_at, and the `status` value before and after) for that one order, newest first. (Hint: extract from `jsonb` with `->>`.)

```sql
-- Write your queries here
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A team runs **schema-per-tenant** with ~800 tenant schemas (`tenant_1` … `tenant_800`). They need to add a `NOT NULL` column `region text` with a default `'us-east'` to the `orders` table in *every* tenant schema, and record which schemas were migrated.

The naive answer — a single `ALTER TABLE orders ADD COLUMN region text NOT NULL DEFAULT 'us-east'` — is wrong for at least two reasons. 

1. Explain both reasons (think: which table does the unqualified name hit, and what does one big `ALTER` across 800 schemas do to locks/failure recovery?).
2. Write a resumable procedure that adds the column to every `tenant_%` schema idempotently and logs each success into a `migration_log(schema_name text, applied_at timestamptz)` table.

```sql
-- Write your solution here
```

### Exercise 4 — Interview Simulation

You're designing the primary keys for a new multi-tenant SaaS. The `events` table will take ~5,000 inserts/sec sustained, rows are referenced by 4 other tables, and event IDs appear in customer-facing webhook URLs. Product wants IDs that "don't leak how many events exist."

Design the key strategy. Justify: internal PK type, whether/how you expose an external ID, index and FK size implications, and write-throughput implications. State what you'd choose on PostgreSQL 18 vs PostgreSQL 14.

```sql
-- Write your DDL and a short justification here
```

---

## 14. Interview Questions

### Q1 — "How would you make a SaaS application multi-tenant in PostgreSQL?"

**Junior answer:** "Add a `tenant_id` column to every table and filter on it in queries."

**Principal answer:** "There are three models, and the right one depends on tenant count, size, and isolation requirements. *Shared schema with `tenant_id`* is the default for many small tenants — best density, one migration covers everyone, one connection pool — but isolation is only as strong as your `WHERE tenant_id` discipline, so I'd back it with Row-Level Security so the predicate can't be forgotten, and lead every index with `tenant_id`. *Schema-per-tenant* suits tens-to-hundreds of larger tenants that need per-tenant export, but it bloats the catalog and makes migrations an N-schema loop with version-skew risk. *Database-per-tenant* gives the strongest isolation and per-tenant residency for a handful of enterprise tenants, but a backend binds to one database so it wrecks connection pooling. Most large SaaS end up hybrid: shared cells for the long tail, dedicated databases for whales, with tenant routing as a data-layer lookup."

**Interviewer follow-up:** "You picked shared schema. A developer ships a query without the tenant filter. What happens and how do you prevent it structurally?" *(Expected: cross-tenant data leak, no error; prevent with RLS injecting the predicate automatically, plus tests and query linting — never rely on discipline.)*

### Q2 — "UUID or bigint for primary keys?"

**Junior answer:** "UUID, because it's globally unique and you can generate it anywhere."

**Principal answer:** "It's a tradeoff between write locality, size, and enumerability. `bigint` identity is 8 bytes and monotonic, so inserts append to the B-tree's right edge — great cache behavior and no page-split storms — but IDs are enumerable, which leaks volume if exposed. Random UUIDv4 is non-enumerable and client-generatable, but its randomness scatters inserts across leaf pages, causing splits, cache misses, and 30–50% index bloat — 2–5× slower writes at scale — and it's 16 bytes in the PK *and every FK and index*. My default is `bigint` internally for FKs and joins, plus a `uuid` `public_id` exposed externally. If I need a single UUID PK, I use UUIDv7 (time-ordered) to recover insert locality while keeping non-enumerability — native `uuidv7()` on PG18, extension or app-side before that."

**Interviewer follow-up:** "Your table takes 5,000 inserts/sec and you're on PostgreSQL 14 with a UUIDv4 PK. Symptoms and fix?" *(Expected: index bloat, falling cache hit ratio, page splits, slowing writes; fix by moving to bigint PK + uuid external id, or app-generated UUIDv7, and REINDEX to reclaim bloat.)*

### Q3 — "How do you implement soft deletes correctly?"

**Junior answer:** "Add an `is_deleted` boolean and filter `WHERE is_deleted = false`."

**Principal answer:** "I use a nullable `deleted_at timestamptz` — it records *when*, not just *whether*, and is as cheap to filter. Then I handle the three problems soft deletes create. First, every read must exclude dead rows; I enforce that structurally with a view or RLS rather than trusting every query. Second, unique constraints break — a soft-deleted email still occupies the unique index and blocks reuse — so I make them partial: `UNIQUE ... WHERE deleted_at IS NULL`. Third, dead rows accumulate and bloat indexes and skew stats, so my access-path indexes are partial on `deleted_at IS NULL`, and I archive very old soft-deleted rows to cold storage. I also watch FK cascades: `ON DELETE CASCADE` doesn't fire on a soft delete, so children need their own cascading-soft-delete handling."

**Interviewer follow-up:** "A user was soft-deleted, now they try to sign up again with the same email and get a duplicate-key error. Why, and the one-line fix?" *(Expected: the full unique index still contains the dead row; replace it with a partial unique index `WHERE deleted_at IS NULL`.)*

### Q4 — "What's dangerous about `SET search_path` behind a connection pooler?"

**Junior answer:** "Nothing — you set it to the tenant's schema and query."

**Principal answer:** "Session-level `SET search_path` mutates state on the physical backend. A transaction-mode pooler like PgBouncer returns that backend to the pool after commit still carrying the mutated path, so the next client's transaction resolves unqualified names against the previous tenant's schema — a silent cross-tenant read. The fix is `SET LOCAL` (or `set_config(..., true)`), which is transaction-scoped and auto-reverts on commit/rollback, so a borrowed connection is always clean. This is also why `SECURITY DEFINER` functions must pin `search_path` explicitly — otherwise a caller can prepend a hostile schema and hijack name resolution."

**Interviewer follow-up:** "How would you parameterize the tenant in `SET LOCAL search_path`?" *(Expected: you can't bind a parameter to a utility command; use `set_config('search_path', $1, true)`, and validate/allowlist the schema name to avoid injection.)*

---

## 15. Mental Model Checkpoint

1. A query `SELECT * FROM orders` runs against a database where both `tenant_42` and `public` contain an `orders` table, with `search_path = tenant_42, public`. Which table does it hit, and at what phase is that decided?

2. You have 6,000 tenants and 40 tables each. Under schema-per-tenant, roughly how many rows land in `pg_class`, and what concretely slows down as a result?

3. A row was soft-deleted six months ago. It still shows up in a monthly revenue report. What are the two most likely root causes, and which one is a schema problem vs a query problem?

4. Why does a random UUIDv4 primary key hurt *write* throughput specifically, and why does UUIDv7 fix most of it but not all of the cost?

5. You run `ALTER TABLE orders ADD COLUMN x int` once and half your schema-per-tenant tenants start erroring. Explain what happened and what the correct operation looks like.

6. In shared-schema multi-tenancy, why must `tenant_id` be the *leading* column of your indexes, and what does EXPLAIN show if it isn't?

7. An audit trigger reads `current_setting('app.current_user_id', true)`. Why the second argument `true`, and why must the app set that GUC with `SET LOCAL` / `set_config(..., true)` rather than a session `SET`?

---

## 16. Quick Reference Card

```sql
-- SCHEMAS & search_path ------------------------------------------------------
CREATE SCHEMA billing AUTHORIZATION app_user;
SHOW search_path;                          -- "$user", public
SET LOCAL search_path TO tenant_42;        -- transaction-scoped (pooler-safe)
SELECT set_config('search_path', $1, true);-- parameterized, transaction-scoped
-- public is no longer world-CREATE by default since PG15 — keep it that way.
-- Pin search_path on SECURITY DEFINER functions.

-- MULTI-TENANT MODELS --------------------------------------------------------
-- Shared schema + tenant_id : best density/migrations/pooling; weakest isolation → add RLS
-- Schema-per-tenant         : easy per-tenant backup; catalog bloat; N-schema migrations
-- Database-per-tenant       : strongest isolation; wrecks connection pooling
-- Index rule: lead EVERY tenant-scoped index with tenant_id.
CREATE INDEX ON orders (tenant_id, created_at DESC) WHERE deleted_at IS NULL;

-- KEYS -----------------------------------------------------------------------
id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY   -- default surrogate; best write locality
public_id uuid DEFAULT gen_random_uuid() UNIQUE      -- external, non-enumerable id
id uuid PRIMARY KEY DEFAULT uuidv7()                 -- PG18: non-enumerable + insert-local
-- Natural key → keep as UNIQUE alongside a surrogate PK (junction tables may use it as PK).

-- SOFT DELETE ----------------------------------------------------------------
deleted_at timestamptz                               -- NULL = live
UPDATE t SET deleted_at = now() WHERE id = $1;        -- "delete"
... WHERE deleted_at IS NULL                          -- EVERY read
CREATE UNIQUE INDEX ON t (email) WHERE deleted_at IS NULL;  -- partial unique = reuse after delete
CREATE INDEX ON t (tenant_id, created_at) WHERE deleted_at IS NULL;  -- partial access index

-- AUDIT ----------------------------------------------------------------------
-- Trigger-based = unbypassable; capture to_jsonb(OLD)/to_jsonb(NEW).
-- actor via SET LOCAL app.current_user_id + current_setting('app.current_user_id', true).
-- audit_logs is append-only → RANGE PARTITION by changed_at (Topic 70); store jsonb diffs.
```

**Perf rules of thumb**
- Lead tenant indexes with `tenant_id`; else Seq Scan across all tenants.
- `bigint` internal PK, `uuid` external id — best of both.
- All "live" indexes and unique constraints are partial (`WHERE deleted_at IS NULL`).
- Behind a pooler: `SET LOCAL` / `set_config(..., true)`, never session `SET`.
- Partition append-only audit tables by time.

**Interview one-liners**
- "Schemas are namespaces — rows in `pg_namespace`, not physical containers."
- "A missing `tenant_id` filter isn't an error, it's a breach — that's why RLS exists."
- "Random UUID PKs trade write locality for opacity; UUIDv7 or bigint+public_id buys both."
- "Soft delete = a nullable `deleted_at`, partial unique indexes, and archiving — not just a flag."
- "`SET LOCAL`, never session `SET`, or the pooler leaks tenant context."

---

## Connected Topics

- **Topic 70 — Table Partitioning** *(previous)*: The mechanism for taming append-only `audit_logs` and for physically isolating tenants by `tenant_id` range/list partitions; detach-to-archive replaces bloating DELETEs.
- **Topic 72 — Row Level Security (RLS)** *(next)*: Turns "remember to filter `tenant_id`" into an engine-enforced predicate — the structural fix for shared-schema isolation this topic keeps pointing to.
- **Topic 04 — Constraints & Keys**: `UNIQUE`, `PRIMARY KEY`, `GENERATED ... AS IDENTITY`, and how a natural key survives as a `UNIQUE` constraint beside a surrogate PK.
- **Topic 07 — NULL in Depth**: Why `deleted_at IS NULL` (not `= NULL`) filters live rows, and how NULL semantics power partial indexes.
- **Topic 30 — Indexes (B-tree internals)**: Why monotonic `bigint` keys append to the right edge while random UUIDs cause page splits and bloat — the physical basis of the UUID-vs-bigint decision.
- **Topic 40 — Triggers**: The `AFTER INSERT/UPDATE/DELETE` machinery behind unbypassable audit logging and cascading soft deletes.
- **Topic 55 — Connection Pooling (PgBouncer)**: Why transaction-mode pooling makes session `SET search_path` dangerous and forces `SET LOCAL`/`set_config`, and why database-per-tenant strains `max_connections`.
- **Topic 60 — JSON/JSONB**: Storing `old_data`/`new_data` row images and change diffs in `audit_logs`.
