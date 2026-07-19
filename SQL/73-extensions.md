# Topic 73 — Essential Extensions
### SQL Mastery Curriculum — Phase 10: PostgreSQL Specific Power Features

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine PostgreSQL is a brand-new smartphone. Out of the box it makes calls, sends texts, takes photos — everything a phone must do. But it doesn't yet have a maps app, a password manager, or a photo-editing suite. Those aren't built into the phone because not everyone needs them, and bundling them all would make the phone bloated, slow to boot, and harder to keep secure.

Instead, there's an **app store**. When you need maps, you install a maps app. When you need to edit photos, you install an editor. Each app plugs into the phone's operating system, gets its own icon, and behaves as if it were always there — but you only pay the cost (storage, memory, attack surface) for the apps you actually installed.

PostgreSQL **extensions** are exactly this app store. The core database ships lean. When you need UUIDs generated in the database, you install `pgcrypto` or `uuid-ossp`. When you need fuzzy text search that tolerates typos, you install `pg_trgm`. When you need to know which queries are burning your CPU, you install `pg_stat_statements`. When you need to store and query geographic coordinates, you install `PostGIS`. Each one adds new functions, new data types, new operators, sometimes new index methods — and to your SQL they look completely native. `gen_random_uuid()` doesn't feel like a plugin; it feels like it was always part of the language.

And here's the crucial part that trips people up: **installing the app on the phone (the binary on the server) is a different step from adding the icon to your particular home screen (running `CREATE EXTENSION` in your particular database).** The maps app can be downloaded to the phone's storage, but until you put it on *your* home screen, *you* can't tap it. A PostgreSQL extension can be available on the server (its files sit in the `SHAREDIR/extension` directory) but not yet created inside your database. Two separate steps, two separate failure modes. Keep them straight and extensions stop being mysterious.

---

## 2. Connection to SQL Internals

An extension is not magic bolted onto a running server. It is a **packaged, versioned bundle of SQL objects and (optionally) compiled C code** that PostgreSQL knows how to install, track, and remove as a single unit. Understanding what the engine does underneath demystifies every gotcha in this topic.

**The control file and the SQL script.** On disk, every extension is described by two file types in `pg_config --sharedir`/`extension/`:
- `NAME.control` — metadata: default version, whether it needs superuser, whether it is relocatable to another schema, its dependencies on other extensions.
- `NAME--VERSION.sql` — the actual `CREATE FUNCTION`, `CREATE TYPE`, `CREATE OPERATOR`, `CREATE OPERATOR CLASS` statements that build the objects.

When you run `CREATE EXTENSION pg_trgm;`, the backend reads the control file, resolves the version, runs the SQL script inside a single transaction, and — this is the internals part — records every object the script created in the **`pg_depend`** catalog with a dependency type of `DEPENDENCY_EXTENSION`. That dependency link is why `DROP EXTENSION pg_trgm;` can cleanly remove every function, operator, and operator class at once: the engine walks `pg_depend`, finds everything owned by the extension's entry in **`pg_extension`**, and drops it. You did not have to remember 40 function names; the catalog remembered for you.

**Where the new capabilities plug into the engine:**
- **New data types** (`citext`, `hstore`, `uuid`, `geometry`) register in `pg_type` with input/output functions written in C. The heap stores them as ordinary tuples — MVCC, WAL, the buffer pool all treat a `geometry` column exactly like any other varlena column. Extensions do not bypass the storage engine.
- **New operators and functions** register in `pg_operator` and `pg_proc`. The planner treats `%` (the pg_trgm similarity operator) like any other operator — it looks up whether an index **operator class** supports it.
- **New index support** is the deepest hook. `pg_trgm` ships **GIN and GiST operator classes** (`gin_trgm_ops`, `gist_trgm_ops`). These teach the existing GIN and GiST access methods how to index trigrams. PostGIS ships a GiST operator class for `geometry`. The extension does not write a new index engine — it plugs into the generic access-method framework (`pg_am`, `pg_opclass`, `pg_amop`, `pg_amproc`) that GIN, GiST, B-tree, BRIN, and SP-GiST all expose. This is why a `pg_trgm` GIN index participates in the buffer pool, WAL logging, and MVCC visibility exactly like a B-tree index does.
- **Background workers and shared memory hooks.** Some extensions (`pg_stat_statements`, `TimescaleDB`, `PostGIS` raster) need more than SQL — they need to run C code at server startup, hook into the executor, or allocate shared memory. Those must be listed in `shared_preload_libraries` in `postgresql.conf`, which requires a **server restart**. `pg_stat_statements` hooks the executor's `ExecutorEnd` to accumulate per-query statistics in shared memory. This is why you cannot just `CREATE EXTENSION pg_stat_statements;` and have it work — the shared-memory counters must be allocated at boot.

So an extension touches the catalog (`pg_extension`, `pg_depend`, `pg_proc`, `pg_type`, `pg_operator`, `pg_opclass`), optionally the access-method framework, and optionally shared memory and the executor hooks. Nothing about it is outside the normal machinery — it is the normal machinery, packaged and versioned.

---

## 3. Logical Execution Order Context

Extensions are not a clause in a SELECT, so they don't sit inside the `FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT` pipeline the way, say, a JOIN does. But the *objects* an extension provides absolutely do, and knowing where each one enters the pipeline matters:

```
FROM users u
JOIN ...                          ← a pg_trgm GIN index can drive an Index Scan here
WHERE u.name % 'jonh'             ← extension operator (%) evaluated in WHERE phase
  AND u.email = crypt($1, u.email) ← pgcrypto function evaluated per-row in WHERE
GROUP BY ...
HAVING similarity(u.name,'jonh')>0.4 ← extension function in HAVING
SELECT gen_random_uuid(),         ← pgcrypto function evaluated in SELECT phase
       similarity(u.name,'jonh')  ← runs once per surviving row
ORDER BY u.name <-> 'jonh'        ← pg_trgm distance operator drives ORDER BY (KNN)
LIMIT 10;                         ← the <-> operator + GiST enables top-N without full sort
```

Two subtleties that only extensions introduce:

1. **The `%` operator and the `similarity()` function look interchangeable but enter the pipeline differently for the planner.** `WHERE name % 'jonh'` is a boolean predicate the planner can satisfy with a `gin_trgm_ops` index (index scan, filter phase). `WHERE similarity(name, 'jonh') > 0.4` is a function call the planner treats as an opaque filter — it usually cannot use the index for it unless rewritten. Same logical result, very different plan. This is the extension-era version of the "SARGable predicate" rule from Topic 40.

2. **The `<->` distance operator enables ordering to be index-driven.** Normally `ORDER BY` (Topic 24) runs *after* the result set is built, as a Sort node. But a GiST index over `gist_trgm_ops` (or PostGIS spatial data) supports a **KNN index scan**: the index returns rows already in nearest-first order, so `ORDER BY name <-> 'jonh' LIMIT 10` skips the Sort node entirely. The ORDER BY is effectively pushed down into the FROM/index-access phase. This is the same trick PostGIS uses for "10 nearest coffee shops."

The takeaway: an extension does not change the logical execution order, but it adds operators and functions that can be evaluated at *different* stages, and some of them (the indexable operators) let the planner move work earlier in the pipeline than a plain function ever could.

---

## 4. What Is an Extension?

A PostgreSQL **extension** is a named, versioned collection of database objects — functions, data types, operators, operator classes, index support, casts, background workers — that is installed into a database as a single managed unit via `CREATE EXTENSION`, tracked in the system catalogs, and removable as a unit via `DROP EXTENSION`.

```sql
CREATE EXTENSION [ IF NOT EXISTS ] extension_name
    [ WITH ]  [ SCHEMA schema_name ]
              [ VERSION version ]
              [ CASCADE ];
```

```
CREATE EXTENSION
│                    └── the command; requires CREATE privilege on the DB,
│                        and often superuser (depends on the extension's control file)
│
IF NOT EXISTS
│   └── skip (with a NOTICE, not an error) if the extension is already installed
│       in this database — makes migration scripts idempotent
│
extension_name
│   └── must match a .control file present in SHAREDIR/extension on the SERVER.
│       If the file isn't there → ERROR: could not open extension control file.
│       That means the OS package (e.g. postgresql-contrib) isn't installed.
│
WITH
│   └── optional noise word; purely for readability
│
SCHEMA schema_name
│   └── install the extension's objects into this schema (e.g. SCHEMA extensions).
│       Only works if the extension is "relocatable" or has no hardcoded schema.
│       Best practice: keep extensions out of public — put them in a dedicated schema.
│
VERSION version
│   └── install a specific version instead of the control file's default_version.
│       Rarely needed; used to pin or to match a replica.
│
CASCADE
│   └── automatically CREATE EXTENSION for any required dependency extensions too.
│       e.g. `CREATE EXTENSION postgis_tiger_geocoder CASCADE` pulls in postgis
│       and fuzzystrmatch. Without CASCADE you get: ERROR: required extension "x"
│       is not installed.
```

Companion commands:

```sql
ALTER EXTENSION pg_trgm UPDATE [ TO 'new_version' ];  -- run the upgrade script
ALTER EXTENSION pg_trgm SET SCHEMA extensions;         -- relocate its objects
ALTER EXTENSION postgis ADD FUNCTION my_fn();          -- adopt an object into the ext
DROP EXTENSION [ IF EXISTS ] pg_trgm [ CASCADE | RESTRICT ];
--                                     │           └── (default) refuse if other
--                                     │               objects depend on it
--                                     └── also drop dependent objects (an index that
--                                         uses gin_trgm_ops will be dropped too!)
```

The critical mental model, restated because it is the #1 source of confusion: **the extension's files being present on the server (the OS package) is separate from the extension being created in your database (the catalog entry).** `CREATE EXTENSION` is a per-database operation. Install it in `mydb` and it is *not* in `otherdb`. Install the OS package once and it is available to every database on that cluster, but created in none of them until you say so.

---

## 5. Why Extension Mastery Matters in Production

1. **You will reinvent — badly — what an extension already does perfectly.** Teams routinely hand-roll UUID generation, application-side password hashing with inconsistent cost factors, or `LIKE '%term%'` search that sequential-scans a 20M-row table on every keystroke. `gen_random_uuid()`, `crypt()`, and a `pg_trgm` GIN index solve all three correctly, faster, and with less code. Not knowing they exist is a direct, measurable cost.

2. **`pg_stat_statements` is the single most valuable diagnostic in PostgreSQL, and it must be set up *before* you have a fire.** You cannot retroactively learn which query pattern consumed 60% of yesterday's CPU if you never enabled it. Every production PostgreSQL should have it in `shared_preload_libraries` from day one. Engineers who don't know it exists debug slow databases blind.

3. **The two-step install trips up migrations and CI.** A migration that runs `CREATE EXTENSION pgcrypto;` fails in CI or on a managed host where the contrib package isn't installed, or where the DB user lacks superuser. Understanding the server-vs-database split, and the managed-provider allowlist, is the difference between a migration that runs everywhere and one that works on your laptop and nowhere else.

4. **`DROP EXTENSION ... CASCADE` can silently destroy indexes and columns.** Because objects depend on the extension via `pg_depend`, dropping `pg_trgm` with CASCADE drops every index built with `gin_trgm_ops`. Dropping `citext` with CASCADE can drop columns typed `citext`. This is a data-availability incident waiting to happen if you don't understand the dependency graph.

5. **Choosing the wrong tool wastes months.** Storing time-series metrics in a plain table and watching it degrade at 500M rows, when `TimescaleDB` hypertables would have kept inserts and range queries fast — or storing geographic points as two `float8` columns and computing Haversine in application code, when `PostGIS` gives you indexed nearest-neighbor for free — are architecture-level mistakes. Knowing what these extensions *are for* is a senior-engineer judgment call.

6. **Version and upgrade discipline.** Extensions are versioned. Upgrading PostgreSQL major versions, restoring dumps, and running replicas all require the extension versions to line up. `pg_dump` records `CREATE EXTENSION` (not the extension's internal objects), so a restore fails if the target server lacks the extension binary. Knowing this prevents 3 a.m. restore failures.

---

## 6. Deep Technical Content

### 6.1 Discovering Extensions — Installed vs Available

There are two questions and they have two different catalog answers.

**"What is available on this server (the binary/files are present)?"** — `pg_available_extensions`:

```sql
SELECT name, default_version, installed_version, comment
FROM pg_available_extensions
ORDER BY name;
```

```
       name        | default_version | installed_version |               comment
-------------------+-----------------+-------------------+--------------------------------------
 citext            | 1.6             |                   | data type for case-insensitive strings
 hstore            | 1.8             |                   | data type for storing (key,value) pairs
 pg_stat_statements| 1.10            | 1.10              | track planning and execution stats
 pg_trgm           | 1.6             | 1.6               | text similarity via trigram matching
 pgcrypto          | 1.3             |                   | cryptographic functions
 uuid-ossp         | 1.1             |                   | generate UUIDs
```

Rows where `installed_version` is NULL are **available but not created in this database**. Rows where it is populated are created here.

**"What is actually created in *this* database?"** — `pg_extension` (the catalog) or the friendlier meta-command:

```sql
SELECT extname, extversion, n.nspname AS schema
FROM pg_extension e
JOIN pg_namespace n ON n.oid = e.extnamespace
ORDER BY extname;
```

In `psql`, the meta-commands are faster:

```
\dx              -- list installed extensions (name, version, schema, description)
\dx+ pg_trgm     -- list every object that extension owns
```

To see which **versions you could upgrade to**:

```sql
SELECT * FROM pg_available_extension_versions
WHERE name = 'pg_trgm';
```

**The mental checklist when an extension "won't install":**
1. `SELECT * FROM pg_available_extensions WHERE name = 'x';` — no row? The OS package is missing on the server. Install `postgresql-contrib` (or the vendor package, e.g. `postgresql-16-postgis-3`), or on a managed host, check whether the provider allows it at all.
2. Row exists, `installed_version` NULL? Just run `CREATE EXTENSION x;` — but check whether it needs superuser (many contrib ones do) or `shared_preload_libraries` (pg_stat_statements, timescaledb).

### 6.2 pg_stat_statements — The Query Profiler

`pg_stat_statements` accumulates, in shared memory, one row per normalized query (constants stripped, so `WHERE id = 1` and `WHERE id = 2` collapse to `WHERE id = $1`). It is the primary tool for "what is slow / what is frequent / what is expensive."

**Setup (requires a restart):**

```
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000          # max distinct statements tracked
pg_stat_statements.track = top          # top | all | none (all = incl. nested)
pg_stat_statements.track_utility = on   # also track DDL/utility commands
```

Then, once per database:

```sql
CREATE EXTENSION pg_stat_statements;
```

**The queries that matter.** Find the biggest total-time consumers (the ones worth optimizing — high total time = frequency × per-call cost):

```sql
SELECT
  queryid,
  substring(query, 1, 80)                 AS query,
  calls,
  round(total_exec_time::numeric, 1)      AS total_ms,
  round(mean_exec_time::numeric, 3)       AS mean_ms,
  round(stddev_exec_time::numeric, 3)     AS stddev_ms,
  rows,
  round(100.0 * shared_blks_hit /
        nullif(shared_blks_hit + shared_blks_read, 0), 1) AS cache_hit_pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

Reading it:
- **High `total_ms`, low `mean_ms`, huge `calls`** → a cheap query run millions of times. Fix with caching, batching, or an N+1 fix (Topic 18), not by optimizing the query itself.
- **High `mean_ms`** → a genuinely slow query. Take its `query` to `EXPLAIN ANALYZE`.
- **High `stddev_ms`** → inconsistent — sometimes fast, sometimes slow. Often a parameter-sensitive plan, lock contention, or cache-cold behavior.
- **Low `cache_hit_pct`** → the query reads from disk a lot; may need a covering index or more RAM.

Columns to know: on PostgreSQL 13+, timing splits into `total_plan_time`/`total_exec_time` (older versions had a single `total_time`). `wal_records`/`wal_bytes` reveal write amplification. Reset the counters after a deploy to measure the new code cleanly:

```sql
SELECT pg_stat_statements_reset();
```

**Gotcha:** the constants are stripped, so you cannot see the *actual* parameter values that were slow — only the shape. Pair it with `auto_explain` or `log_min_duration_statement` when you need the offending literals.

### 6.3 pgcrypto — UUIDs, Hashing, and Encryption

`pgcrypto` provides cryptographic primitives. Three families matter in practice.

**(a) UUID generation.** Since PostgreSQL 13, `gen_random_uuid()` is actually built into core, but historically it came from `pgcrypto`, and on older servers you still `CREATE EXTENSION pgcrypto;` to get it:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE sessions (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- v4, random
  user_id     BIGINT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`gen_random_uuid()` returns a **version-4 (random) UUID**. Because it is random, consecutive inserts scatter across the B-tree — see the performance section for why random UUIDs hurt index locality and what UUIDv7 fixes.

**(b) Password hashing with `crypt()` and `gen_salt()`.** This is the correct way to store passwords *if* you hash in the database. `crypt()` uses a salted, deliberately-slow algorithm; `gen_salt('bf', 12)` produces a bcrypt salt with cost factor 12:

```sql
-- Store a hashed password
INSERT INTO users (email, password_hash)
VALUES ($1, crypt($2, gen_salt('bf', 12)));   -- $2 = plaintext password

-- Verify a login: re-hash the input with the STORED hash as the salt
SELECT id
FROM users
WHERE email = $1
  AND password_hash = crypt($2, password_hash);
--                                └── passing the stored hash as the "salt" extracts
--                                    the original salt+cost and reproduces the hash.
--                                    Equal → password correct. Constant-time within crypt.
```

The elegance: you never store the plaintext, never store the salt separately (it's embedded in the bcrypt string `$2a$12$...`), and verification is a single indexed lookup plus one `crypt()` call. `gen_salt()` supports `'bf'` (bcrypt), `'md5'`, `'xdes'`, `'des'` — use bcrypt. Cost factor 12 is a reasonable 2020s default; raise it as hardware improves.

**Note on where to hash:** many teams hash in the *application* (bcrypt/argon2 libraries) instead, to keep CPU-heavy hashing off the database and to use argon2 (which pgcrypto lacks). Both are valid; the database approach centralizes the logic and guarantees consistency but competes for DB CPU.

**(c) Digest and HMAC:**

```sql
SELECT encode(digest('hello', 'sha256'), 'hex');       -- one-way hash → hex string
SELECT encode(hmac('msg', 'secret_key', 'sha256'), 'hex'); -- keyed MAC for signatures
```

**(d) Symmetric encryption** (PGP-style, for encrypting column values at rest):

```sql
-- Encrypt on write
UPDATE users SET ssn_enc = pgp_sym_encrypt($1, current_setting('app.enc_key'))
WHERE id = $2;

-- Decrypt on read (only for authorized paths)
SELECT pgp_sym_decrypt(ssn_enc, current_setting('app.enc_key')) FROM users WHERE id = $1;
```

Caveat: the key must come from somewhere. Passing it per-session via a GUC (as above) or via the connection keeps it out of the table, but the decrypted value and the key live in server memory and can appear in logs if you're careless. This encrypts data at rest in the table; it does not protect against a compromised DB superuser.

### 6.4 uuid-ossp — The Older UUID Toolkit

Before `gen_random_uuid()` was ubiquitous, `uuid-ossp` was the standard UUID source. It offers multiple UUID algorithms:

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";   -- note the required quotes (hyphen in name)

SELECT uuid_generate_v4();   -- random (equivalent to gen_random_uuid())
SELECT uuid_generate_v1();   -- time + MAC address based (leaks MAC + time!)
SELECT uuid_generate_v1mc(); -- time-based with random node instead of MAC
SELECT uuid_generate_v5(uuid_ns_url(), 'https://example.com'); -- deterministic (namespace + name, SHA-1)
SELECT uuid_generate_v3(uuid_ns_dns(), 'example.com');         -- deterministic (MD5)
```

**Modern guidance:** for a plain random UUID, prefer core `gen_random_uuid()` — no extension needed on PG 13+. Reach for `uuid-ossp` only when you specifically need **v5 deterministic** UUIDs (same input → same UUID, useful for idempotent keys derived from natural data) or the legacy v1/v3 algorithms. Note the mandatory double-quotes around `"uuid-ossp"` because of the hyphen.

### 6.5 pg_trgm — Trigram Similarity and Fuzzy Search

This is the workhorse extension for **typo-tolerant search** and **fast `LIKE '%term%'`**. A *trigram* is a three-character substring. pg_trgm decomposes strings into their set of trigrams (padding the ends), then measures similarity as the overlap of two trigram sets.

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

SELECT show_trgm('cat');
-- {"  c"," ca","at ","cat"}   -- padded start/end trigrams

SELECT similarity('jonh', 'john');   -- 0.5  (0 = nothing in common, 1 = identical)
SELECT 'jonh' <-> 'john';            -- 0.5  (<-> is DISTANCE = 1 - similarity)
```

**The operators pg_trgm adds:**

| Operator | Meaning | Typical use |
|----------|---------|-------------|
| `a % b` | similarity ≥ `pg_trgm.similarity_threshold` (default 0.3) | fuzzy WHERE filter |
| `a <-> b` | distance = 1 − similarity | ORDER BY nearest (KNN) |
| `a %> b`, `a <->> b` | word-similarity variants (match a word inside a longer string) | search a short term inside long text |
| `a %>> b`, `a <<-> b` | strict word-similarity variants | phrase-in-document matching |

**The two index strategies — this is the key decision:**

```sql
-- GIN: faster for containment / LIKE / % queries, slower to build & update,
--      does NOT support the <-> distance operator (no KNN ordering).
CREATE INDEX idx_users_name_gin ON users USING gin (name gin_trgm_ops);

-- GiST: supports the <-> KNN distance operator (ORDER BY ... <-> ... LIMIT n),
--       generally smaller but slower for pure % containment on large tables.
CREATE INDEX idx_users_name_gist ON users USING gist (name gist_trgm_ops);
```

Rule of thumb: **use `gin_trgm_ops` for `LIKE`/`ILIKE`/`%` filtering; use `gist_trgm_ops` when you need `ORDER BY col <-> 'term' LIMIT n` nearest-neighbor ranking.** You can have both if you need both behaviors.

**The killer feature — accelerating `LIKE '%substring%'`.** Normally a leading-wildcard `LIKE` cannot use a B-tree index (Topic 40) and forces a Seq Scan. A `gin_trgm_ops` index changes that:

```sql
-- Without the trigram index: Seq Scan on 20M rows, ~seconds.
-- With gin_trgm_ops: Bitmap Index Scan on the trigram index, ~milliseconds.
SELECT id, name
FROM users
WHERE name ILIKE '%smith%';
```

The index extracts trigrams from `'smith'` and finds candidate rows whose trigram sets contain them, then rechecks. This works for `LIKE`, `ILIKE`, `~` and `~*` regex (for the literal portions), and `%` similarity — all from one GIN index.

**Fuzzy / typo-tolerant search:**

```sql
-- "Find users whose name is similar to what was typed, best matches first"
SELECT id, name, similarity(name, 'jonh smyth') AS score
FROM users
WHERE name % 'jonh smyth'                 -- similarity ≥ threshold; uses gin/gist index
ORDER BY name <-> 'jonh smyth'            -- nearest first; uses gist index (KNN)
LIMIT 10;
```

**Tuning the threshold** (session-scoped; the `%` operator respects it):

```sql
SET pg_trgm.similarity_threshold = 0.4;   -- stricter; fewer, closer matches
SELECT set_limit(0.4);                     -- older function-based equivalent
SELECT show_limit();                       -- read current threshold
```

**Gotcha:** `similarity(col, 'x') > 0.4` in `WHERE` does **not** use the index (opaque function), whereas `col % 'x'` **does** (indexable operator). If you want a custom threshold *and* index usage, set `pg_trgm.similarity_threshold` and use `%`, rather than writing `similarity() >` .

### 6.6 citext — Case-Insensitive Text

`citext` is a text type that compares case-insensitively. It exists because the common alternative — `WHERE lower(email) = lower($1)` — is verbose, easy to forget in one query out of twenty (creating a case-sensitivity bug), and needs a functional index to be fast.

```sql
CREATE EXTENSION IF NOT EXISTS citext;

CREATE TABLE users (
  id     BIGINT PRIMARY KEY,
  email  CITEXT UNIQUE NOT NULL   -- 'Alice@X.com' and 'alice@x.com' collide → UNIQUE works
);

INSERT INTO users VALUES (1, 'Alice@Example.com');
SELECT * FROM users WHERE email = 'alice@example.com';  -- MATCHES, no lower() needed
INSERT INTO users VALUES (2, 'ALICE@example.com');       -- ERROR: duplicate key (correct!)
```

The huge win is the **case-insensitive UNIQUE constraint** — with plain `text`, `Alice@x.com` and `alice@x.com` are two distinct rows, letting duplicate accounts slip in. With `citext`, the UNIQUE index enforces case-insensitive uniqueness natively. Indexes on `citext` columns work normally (equality and pattern matching), no functional index required.

**Gotchas:** `citext` folds case using the database's collation/locale rules, which for non-ASCII (Turkish dotless i, German ß) can surprise you. It compares case-insensitively but *stores* the original casing (so you still see `Alice@Example.com` on read). `LIKE` on citext is case-insensitive; if you need a case-sensitive pattern match on a citext column, cast to `text`.

### 6.7 hstore — Key/Value Pairs in a Column

`hstore` stores a set of string key→string value pairs in a single column. It predates `jsonb` and is now largely superseded by it, but it remains common in legacy schemas and is lighter for flat string maps.

```sql
CREATE EXTENSION IF NOT EXISTS hstore;

CREATE TABLE products (
  id     BIGINT PRIMARY KEY,
  attrs  HSTORE
);

INSERT INTO products VALUES (1, 'color => red, size => "XL", weight => 2kg');

SELECT attrs -> 'color'            FROM products WHERE id = 1;   -- 'red' (value by key)
SELECT attrs ? 'size'              FROM products WHERE id = 1;   -- true (key exists)
SELECT attrs @> 'color => red'     FROM products;               -- rows containing that pair
SELECT akeys(attrs), avals(attrs)  FROM products;               -- arrays of keys / values

-- GIN index for containment queries
CREATE INDEX idx_products_attrs ON products USING gin (attrs);
```

**hstore vs jsonb — the modern verdict:** use `jsonb` for anything new. `jsonb` supports nested structures, numbers, booleans, arrays, and richer operators, and has the same GIN indexing. `hstore` only stores flat string→string maps. The only reasons to still choose hstore: an existing schema uses it, or you have a genuinely flat string-map and want the marginally smaller storage/simpler semantics. Know it exists because you *will* meet it in old databases.

### 6.8 PostGIS — Spatial Data (The Concept)

`PostGIS` turns PostgreSQL into a full geographic information system. It is by far the largest and most powerful extension in the ecosystem — an entire product built on the extension mechanism.

What it adds conceptually:
- **New types:** `geometry` (planar, Cartesian) and `geography` (spherical, on the WGS84 earth model). Columns hold points, lines, polygons, multipolygons.
- **Hundreds of functions:** `ST_Distance`, `ST_DWithin`, `ST_Contains`, `ST_Intersects`, `ST_Area`, `ST_Buffer`, `ST_Transform` (reproject between coordinate systems), and so on.
- **GiST (and SP-GiST) spatial indexing:** a GiST operator class over `geometry` bounding boxes enables indexed spatial queries and KNN.

The canonical query it makes trivial and fast — **"find the 10 nearest coffee shops to this point"**:

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE INDEX idx_shops_geom ON shops USING gist (geom);  -- spatial GiST index

SELECT id, name,
       geom <-> ST_SetSRID(ST_MakePoint(-73.99, 40.73), 4326) AS dist
FROM shops
ORDER BY geom <-> ST_SetSRID(ST_MakePoint(-73.99, 40.73), 4326)  -- KNN via GiST
LIMIT 10;

-- "Every shop within 500 meters" — ST_DWithin is index-accelerated
SELECT id, name
FROM shops
WHERE ST_DWithin(geom::geography,
                 ST_MakePoint(-73.99, 40.73)::geography, 500);
```

Without PostGIS you would store lat/lng as two floats, compute Haversine distance in application code for *every* row (no index possible), and sort in the app — untenable past a few thousand rows. PostGIS makes it an indexed millisecond query. You don't need to memorize its 800 functions; you need to know it exists, that it's the answer to any "distance / contains / within / nearest" requirement, and that it plugs into GiST for indexing.

### 6.9 TimescaleDB — Time-Series (The Concept)

`TimescaleDB` is an extension that specializes PostgreSQL for **time-series workloads**: high-volume append-mostly data keyed by time (metrics, sensor readings, financial ticks, events).

The core idea is the **hypertable**: a table that looks and queries like a normal PostgreSQL table but is transparently **partitioned by time (and optionally by a space key)** into many physical "chunks" under the hood.

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;   -- needs shared_preload_libraries + restart

CREATE TABLE metrics (
  time    TIMESTAMPTZ NOT NULL,
  device  BIGINT,
  value   DOUBLE PRECISION
);
SELECT create_hypertable('metrics', 'time');  -- convert to a hypertable
```

Why it matters:
- **Insert performance stays flat as data grows** — new rows go to the current (small, cache-resident) time chunk, so the B-tree you're inserting into stays small. A plain table's single index degrades as it grows into the hundreds of millions of rows.
- **Time-range queries prune chunks** — `WHERE time > now() - interval '1 day'` only touches the recent chunks; older chunks are skipped entirely (chunk exclusion, akin to partition pruning from Topic 62).
- **Automatic retention and compression** — drop chunks older than N days in O(1) (`DROP` a chunk, not `DELETE` millions of rows), and columnar-compress old chunks for 10×+ storage savings.
- **Continuous aggregates** — incrementally-maintained materialized rollups (hourly/daily averages) that refresh only the changed time buckets.

Conceptually: reach for TimescaleDB when your dominant access pattern is "recent time window of append-only data," your table will exceed hundreds of millions of rows, and you need retention/rollups. For general OLTP it adds no value. It is the "PostgreSQL is now a time-series database" extension.

### 6.10 Extension Schema Placement and Search Path

A frequently overlooked best practice: don't dump extension objects into `public`. Put them in a dedicated schema and add it to the search path. This keeps `\d` in `public` clean, avoids name collisions, and makes it obvious what is "yours" vs "the extension's."

```sql
CREATE SCHEMA IF NOT EXISTS extensions;
CREATE EXTENSION pg_trgm SCHEMA extensions;
-- Make its functions/operators resolvable without qualification:
ALTER DATABASE mydb SET search_path = "$user", public, extensions;
```

Caveat: some extensions (notably PostGIS historically) are **not relocatable** or expect specific schemas; check the control file's `relocatable = true|false`. And moving an extension's schema after objects depend on it can be disruptive — decide at creation time.

### 6.11 Versioning, Upgrades, and Dumps

- **`pg_dump` dumps `CREATE EXTENSION name;`, not the extension's internal objects.** This keeps dumps small and portable — but the restore target *must* have the extension binary available, or the restore fails at `CREATE EXTENSION`. This is the #1 cause of "my backup won't restore on the new server": the contrib/PostGIS package isn't installed there.
- **Upgrading an extension** runs its migration scripts: `ALTER EXTENSION pg_trgm UPDATE;` moves from the installed version to `default_version` via `NAME--old--new.sql` scripts. Do this after a PostgreSQL minor/major upgrade that shipped a newer extension.
- **Replicas and logical replication** must have matching extension versions; a function signature that changed between versions can break replication.
- **Objects you created that depend on the extension** (a trigram index, a citext column) are dumped normally and will be recreated *after* the `CREATE EXTENSION` line — provided the extension restores first. Dependency ordering is handled by `pg_dump`.

### 6.12 Managed Providers and the Allowlist

On RDS, Cloud SQL, Azure, Supabase, Neon, etc., you are **not** a filesystem superuser — you cannot install arbitrary OS packages. Each provider ships a curated **allowlist** of extensions they support, and often gates `shared_preload_libraries` changes behind a parameter group + reboot.

- `pg_stat_statements`, `pgcrypto`, `pg_trgm`, `citext`, `hstore`, `uuid-ossp`, `PostGIS` are almost universally supported.
- `TimescaleDB` availability varies (Timescale Cloud, or self-managed; some clouds restrict it).
- Anything requiring a truly untrusted C library may be blocked.
- The install command is the same (`CREATE EXTENSION`), but check the provider's docs and note that `shared_preload_libraries` edits happen through the provider's config UI/parameter group, then a managed reboot — not by editing `postgresql.conf` directly.

`pg_available_extensions` on a managed instance shows you exactly what that provider permits — always check it before writing a migration that assumes an extension exists.

---

## 7. EXPLAIN — pg_trgm Index in the Plan

The most instructive EXPLAIN for this topic is fuzzy/substring search with and without a trigram index.

### Without the trigram index (the problem)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name FROM users WHERE name ILIKE '%smith%';
```

```
Seq Scan on users  (cost=0.00..48225.00 rows=1900 width=27)
                   (actual time=0.42..1180.6 rows=2140 loops=1)
  Filter: (name ~~* '%smith%'::text)
  Rows Removed by Filter: 1997860
  Buffers: shared hit=512 read=35713
Planning Time: 0.12 ms
Execution Time: 1181.4 ms
```

**Reading it:**
- `Seq Scan` — no index can serve a leading-wildcard `ILIKE`, so the whole 2M-row table is read.
- `Rows Removed by Filter: 1997860` — nearly every row scanned is thrown away. Classic wasted work.
- `read=35713` — 35K pages read from disk. ~1.2 seconds. Unacceptable for a search-as-you-type box.

### With `gin_trgm_ops`

```sql
CREATE INDEX idx_users_name_gin ON users USING gin (name gin_trgm_ops);

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name FROM users WHERE name ILIKE '%smith%';
```

```
Bitmap Heap Scan on users  (cost=88.50..3540.20 rows=1900 width=27)
                           (actual time=2.10..9.8 rows=2140 loops=1)
  Recheck Cond: (name ~~* '%smith%'::text)
  Rows Removed by Index Recheck: 61
  Heap Blocks: exact=2088
  Buffers: shared hit=2210 read=94
  ->  Bitmap Index Scan on idx_users_name_gin
                           (cost=0.00..88.02 rows=1900 width=0)
                           (actual time=1.75..1.75 rows=2201 loops=1)
        Index Cond: (name ~~* '%smith%'::text)
Planning Time: 0.28 ms
Execution Time: 10.3 ms
```

**Reading it:**
- `Bitmap Index Scan on idx_users_name_gin` — the trigram GIN index found candidate rows by matching the trigrams of `'smith'` (` sm`, `smi`, `mit`, `ith`, `th `).
- `Index Cond: (name ~~* '%smith%')` — the `ILIKE` predicate is now index-driven. This is the whole point: a leading-wildcard pattern became indexable.
- `Rows Removed by Index Recheck: 61` — GIN is lossy for trigrams, so it over-returns slightly and the heap recheck confirms the actual `ILIKE`. A small, expected number.
- `read=94` vs 35713 before, and **10 ms vs 1181 ms** — a ~115× speedup.

### KNN fuzzy ranking with `gist_trgm_ops`

```sql
CREATE INDEX idx_users_name_gist ON users USING gist (name gist_trgm_ops);

EXPLAIN (ANALYZE)
SELECT id, name FROM users
ORDER BY name <-> 'jonh smyth'
LIMIT 10;
```

```
Limit  (cost=0.41..4.05 rows=10 width=31)
       (actual time=0.9..3.6 rows=10 loops=1)
  ->  Index Scan using idx_users_name_gist on users
        (cost=0.41..728000.00 rows=2000000 width=31)
        (actual time=0.9..3.5 rows=10 loops=1)
        Order By: (name <-> 'jonh smyth'::text)
Planning Time: 0.3 ms
Execution Time: 3.7 ms
```

**Reading it:**
- `Index Scan ... Order By: (name <-> 'jonh smyth')` — the GiST index returns rows in **nearest-first order**. There is **no Sort node**. The ORDER BY was pushed into the index access (a KNN scan).
- The `Limit` stops after 10 rows — the index only had to walk far enough to produce the 10 closest names, not sort 2M rows. This is why GiST + `<->` is the right tool for "top-N similar."

### What to look for

| Symptom | Meaning | Fix |
|---------|---------|-----|
| Seq Scan + `Rows Removed by Filter` huge on `ILIKE '%x%'` | No trigram index | `CREATE INDEX ... gin (col gin_trgm_ops)` |
| `similarity(col,'x') > 0.4` still Seq Scans despite a GIN index | Function form isn't indexable | Use `col % 'x'` + `SET pg_trgm.similarity_threshold` |
| Sort node before a `<->` ORDER BY | Using GIN (no KNN) instead of GiST | Add `gist (col gist_trgm_ops)` |
| Large `Rows Removed by Index Recheck` | Very lossy / short search term | Longer search term, or accept the recheck |

---

## 8. Query Examples

### Example 1 — Basic: Enable an extension and use it

```sql
-- Enable pgcrypto and use a database-generated UUID primary key.
CREATE EXTENSION IF NOT EXISTS pgcrypto;   -- idempotent; safe to re-run in migrations

CREATE TABLE sessions (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- server generates the id
  user_id     BIGINT      NOT NULL REFERENCES users(id),
  token_hash  TEXT        NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at  TIMESTAMPTZ NOT NULL
);

INSERT INTO sessions (user_id, token_hash, expires_at)
VALUES (42, encode(digest('raw-token', 'sha256'), 'hex'), now() + interval '7 days')
RETURNING id;   -- returns the generated UUID
```

### Example 2 — Intermediate: Fuzzy search with pg_trgm

```sql
-- Typo-tolerant customer lookup for a support console.
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX IF NOT EXISTS idx_users_name_gin ON users USING gin (name gin_trgm_ops);

-- Agent typed "jonh smyth"; return the 10 closest real names with a score.
SELECT
  id,
  name,
  email,
  round(similarity(name, 'jonh smyth')::numeric, 3) AS match_score
FROM users
WHERE name % 'jonh smyth'                 -- indexable similarity filter (>= threshold)
ORDER BY similarity(name, 'jonh smyth') DESC
LIMIT 10;
-- % uses the GIN index to shortlist; similarity() then scores/ranks the shortlist.
```

### Example 3 — Production Grade: Case-insensitive accounts + password auth + audit

```sql
-- Context:
--   users: 8M rows. Login is by email (case-insensitive) + password.
--   Requirements: no duplicate accounts differing only by email case;
--   passwords stored as bcrypt; every login attempt audited.
--   Indexes: citext UNIQUE on email (auto), audit_logs on (user_id, created_at).
--   Perf expectation: login = one index seek on email + one crypt() call, < 3 ms.

CREATE EXTENSION IF NOT EXISTS citext;
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE users (
  id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email          CITEXT UNIQUE NOT NULL,          -- case-insensitive uniqueness
  password_hash  TEXT   NOT NULL,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Registration (application passes plaintext as $2; hashing happens in-DB):
INSERT INTO users (email, password_hash)
VALUES ($1, crypt($2, gen_salt('bf', 12)))
ON CONFLICT (email) DO NOTHING            -- case-insensitive conflict via citext
RETURNING id;

-- Login: verify credentials and audit in one round trip via a CTE.
WITH attempt AS (
  SELECT id
  FROM users
  WHERE email = $1                                   -- citext: case-insensitive, indexed
    AND password_hash = crypt($2, password_hash)     -- re-derive with stored salt/cost
)
INSERT INTO audit_logs (user_id, action, success, created_at)
SELECT
  COALESCE((SELECT id FROM attempt), NULL),
  'login',
  (SELECT id FROM attempt) IS NOT NULL,
  now()
RETURNING user_id, success;
```

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM users
WHERE email = 'Alice@Example.com'
  AND password_hash = crypt('secret', password_hash);
```

```
Index Scan using users_email_key on users
      (cost=0.43..8.46 rows=1 width=8)
      (actual time=0.05..2.41 rows=1 loops=1)
  Index Cond: (email = 'Alice@Example.com'::citext)
  Filter: (password_hash = crypt('secret'::text, password_hash))
  Buffers: shared hit=4
Planning Time: 0.14 ms
Execution Time: 2.48 ms
```

Reading it: the `citext` UNIQUE index resolves the email with a single index seek (`Index Cond`, case-insensitive). The `crypt()` verification is a per-row `Filter` — but it runs on exactly one candidate row, so the deliberately-slow bcrypt cost is paid once (~2 ms at cost 12), keeping total latency under 3 ms as designed.

---

## 9. Wrong → Right Patterns

### Wrong 1: Assuming CREATE EXTENSION on one database covers all databases

```sql
-- In database `app_prod`:
CREATE EXTENSION pg_trgm;   -- success

-- Later, a migration runs in `app_analytics` and does:
CREATE INDEX idx ON reports USING gin (title gin_trgm_ops);
-- ERROR: operator class "gin_trgm_ops" does not exist for access method "gin"
```

**Why it's wrong at the execution level:** `CREATE EXTENSION` writes a row to *this database's* `pg_extension` and registers the operator class in *this database's* `pg_opclass`. `app_analytics` never had it created, so `gin_trgm_ops` is undefined there. Extensions are per-database, not per-cluster.

```sql
-- RIGHT: create the extension in every database that needs it.
\c app_analytics
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx ON reports USING gin (title gin_trgm_ops);
```

### Wrong 2: Using similarity() in WHERE and losing the index

```sql
-- WRONG: expects the trigram index to help, but it can't.
CREATE INDEX idx_users_name_gin ON users USING gin (name gin_trgm_ops);

SELECT id, name FROM users
WHERE similarity(name, 'smith') > 0.4;   -- Seq Scan! function call is opaque to planner
```

**Why it's wrong:** the planner sees `similarity(name, 'smith') > 0.4` as an arbitrary function returning a float compared to a constant. There is no operator class entry that maps this comparison to the GIN index, so it falls back to a Seq Scan and computes `similarity` for all 2M rows.

```sql
-- RIGHT: use the indexable % operator, control the cutoff via the GUC.
SET pg_trgm.similarity_threshold = 0.4;
SELECT id, name FROM users
WHERE name % 'smith';                    -- Bitmap Index Scan on gin_trgm_ops
```

### Wrong 3: DROP EXTENSION CASCADE destroying indexes and columns

```sql
-- WRONG: "we're not using pg_trgm anymore, drop it."
DROP EXTENSION pg_trgm;
-- ERROR: cannot drop extension pg_trgm because other objects depend on it
-- DETAIL: index idx_users_name_gin depends on operator class gin_trgm_ops

-- So the developer "fixes" it:
DROP EXTENSION pg_trgm CASCADE;
-- SILENTLY drops idx_users_name_gin — every fuzzy search now Seq Scans in prod.
```

**Why it's wrong:** `pg_depend` records that `idx_users_name_gin` depends on `gin_trgm_ops`, which the extension owns. `CASCADE` obeys those dependencies and drops the index. For `citext`, CASCADE can even drop **columns** typed `citext`, i.e. data loss.

```sql
-- RIGHT: understand and remove dependents deliberately, or don't drop at all.
SELECT indexname FROM pg_indexes WHERE indexdef LIKE '%gin_trgm_ops%';  -- audit first
DROP INDEX idx_users_name_gin;   -- remove the dependent knowingly
DROP EXTENSION pg_trgm;          -- now RESTRICT (default) succeeds cleanly
```

### Wrong 4: pg_stat_statements without shared_preload_libraries

```sql
-- WRONG:
CREATE EXTENSION pg_stat_statements;   -- succeeds (creates the view)
SELECT * FROM pg_stat_statements;
-- ERROR: pg_stat_statements must be loaded via "shared_preload_libraries"
```

**Why it's wrong:** the SQL objects (the view, the functions) get created, but the actual counters live in shared memory that is only allocated at server startup, and the executor hook that populates them is only installed if the library is preloaded. Without the preload, the view has nothing to read.

```sql
-- RIGHT: add to postgresql.conf (or the managed parameter group), then RESTART.
--   shared_preload_libraries = 'pg_stat_statements'
-- Then, once, per database:
CREATE EXTENSION pg_stat_statements;
-- Now the view returns data.
```

### Wrong 5: Storing lat/lng as floats and computing distance in the app

```sql
-- WRONG: two float columns, Haversine in application code.
SELECT id, name, lat, lng FROM shops;   -- fetch ALL shops
-- ...then loop in Node.js computing distance to the user, sort, take 10.
-- At 500K shops this transfers everything and cannot use any index.
```

**Why it's wrong:** no index can accelerate "nearest by great-circle distance" over two independent float columns; you sequential-scan and ship the whole table to the app every request.

```sql
-- RIGHT: PostGIS geometry + GiST index → indexed KNN in the database.
CREATE EXTENSION IF NOT EXISTS postgis;
ALTER TABLE shops ADD COLUMN geom geometry(Point, 4326);
UPDATE shops SET geom = ST_SetSRID(ST_MakePoint(lng, lat), 4326);
CREATE INDEX idx_shops_geom ON shops USING gist (geom);

SELECT id, name
FROM shops
ORDER BY geom <-> ST_SetSRID(ST_MakePoint($1, $2), 4326)   -- KNN, index-driven
LIMIT 10;
```

---

## 10. Performance Profile

### pg_trgm indexes at scale

| Table size | `ILIKE '%x%'` no index | `gin_trgm_ops` index | Notes |
|-----------|------------------------|----------------------|-------|
| 100K rows | ~30–80 ms Seq Scan | ~2–5 ms | Index barely needed but still wins |
| 1M rows | ~300–500 ms Seq Scan | ~5–15 ms | Clear win |
| 10M rows | ~3–6 s Seq Scan | ~15–60 ms | Mandatory for interactive search |
| 100M rows | ~30–60 s (unusable) | ~50–200 ms | GIN size becomes a real cost |

- **GIN build cost:** trigram GIN indexes are large (they store many trigrams per row) and slow to build; budget significant time and use `maintenance_work_mem`. Consider `CREATE INDEX CONCURRENTLY` to avoid locking writes on a live table.
- **Write amplification:** every `INSERT`/`UPDATE` to an indexed text column must update the GIN index for every changed trigram. GIN has a `fastupdate` pending list to batch this, but heavy write workloads on trigram-indexed columns pay a real cost. Watch autovacuum.
- **GiST vs GIN memory/CPU:** GiST indexes are smaller and support KNN (`<->`) but are slower for pure `%`/`ILIKE` containment on large tables. Choose per query pattern (Section 6.5). Having both a GIN and a GiST index on the same column is legitimate when you need both containment filtering and KNN ranking.
- **Short search terms are pathological:** a 1–2 character search yields few/degenerate trigrams; the index over-returns and the heap recheck dominates. Enforce a minimum query length (e.g. ≥ 3 chars) in the application.

### UUID primary keys and index locality

- **Random UUIDs (`gen_random_uuid`, v4)** scatter inserts uniformly across the B-tree. Each insert dirties a *different* leaf page, so the working set of hot pages is large, cache hit ratio drops, and you get more WAL and more page splits than a sequential key. At high insert rates on 100M+ rows this is measurable (slower inserts, larger index, worse cache behavior) versus a `BIGINT` identity.
- **Fix — time-ordered UUIDs (UUIDv7):** UUIDv7 embeds a timestamp prefix so new values sort near each other, restoring the append-at-the-end locality of a serial key while keeping global uniqueness. PostgreSQL 18 adds `uuidv7()` in core; before that, generate UUIDv7 in the application or via a SQL function. Prefer UUIDv7 over v4 for high-write primary keys.

### pgcrypto CPU cost

- `crypt()` with bcrypt cost 12 is **intentionally slow** — roughly a few milliseconds per call. That is a feature for password hashing (slows brute force) but a liability if you accidentally call it per-row over a large set. Never `crypt()` in a WHERE that scans many rows; always narrow to a single candidate first (as in Example 3).
- `digest`/`hmac` (SHA-256) are cheap; safe at volume.
- `pgp_sym_encrypt/decrypt` are heavier; encrypt/decrypt only the specific columns and rows you must, never whole-table.

### pg_stat_statements overhead

- Overhead is low (a small per-query bookkeeping cost and shared-memory writes), and it is worth it on essentially every production server. `pg_stat_statements.max` too high wastes shared memory; the default range (5000–10000) is fine for most. `track = all` (including nested statements inside functions) adds overhead — use `track = top` unless you specifically need function-internal visibility.

### General extension scaling notes

- Extensions add **no query-time overhead just by being installed** — an unused extension's functions cost nothing. The cost is only in the objects you actually use (indexes to maintain, functions to call).
- `shared_preload_libraries` extensions (pg_stat_statements, timescaledb) consume some shared memory at all times; size accordingly.

---

## 11. Node.js Integration

### 11.1 Enabling an extension from a migration

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Idempotent enablement — safe to run on every deploy.
async function ensureExtensions() {
  // Note: CREATE EXTENSION cannot be parameterized ($1) — the name is an
  // identifier, not a value. Use a hardcoded allowlist, never string interpolation
  // of user input, to avoid injection.
  const required = ['pgcrypto', 'pg_trgm', 'citext'];
  for (const name of required) {
    await pool.query(`CREATE EXTENSION IF NOT EXISTS ${name}`);
  }
}
```

### 11.2 Fuzzy search endpoint with parameter binding

```javascript
async function searchUsers(term) {
  if (!term || term.trim().length < 3) return [];   // enforce min length (perf)
  const { rows } = await pool.query(
    `SELECT id, name, email,
            round(similarity(name, $1)::numeric, 3) AS score
     FROM users
     WHERE name % $1                       -- $1 bound as the trigram query (indexed)
     ORDER BY similarity(name, $1) DESC
     LIMIT 10`,
    [term.trim()]
  );
  return rows;
}
```

### 11.3 Registration and login with pgcrypto

```javascript
async function registerUser(email, plaintextPassword) {
  const { rows } = await pool.query(
    `INSERT INTO users (email, password_hash)
     VALUES ($1, crypt($2, gen_salt('bf', 12)))   -- hashing happens in the DB
     ON CONFLICT (email) DO NOTHING
     RETURNING id`,
    [email, plaintextPassword]
  );
  return rows[0] ?? null;   // null if the (case-insensitive) email already exists
}

async function verifyLogin(email, plaintextPassword) {
  const { rows } = await pool.query(
    `SELECT id
     FROM users
     WHERE email = $1
       AND password_hash = crypt($2, password_hash)`,   // re-derive with stored salt
    [email, plaintextPassword]
  );
  return rows[0]?.id ?? null;   // null → invalid credentials
}
```

### 11.4 Reading pg_stat_statements for an ops dashboard

```javascript
async function topQueriesByTotalTime(limit = 20) {
  const { rows } = await pool.query(
    `SELECT queryid,
            left(query, 100)                    AS query,
            calls,
            round(total_exec_time::numeric, 1)  AS total_ms,
            round(mean_exec_time::numeric, 3)   AS mean_ms
     FROM pg_stat_statements
     ORDER BY total_exec_time DESC
     LIMIT $1`,
    [limit]
  );
  return rows;
}
```

**Note on `node-postgres` and types:** the `pg` driver returns `uuid` and `citext` values as JavaScript strings, `hstore` as a string unless you register a type parser, and PostGIS `geometry` as WKB hex (use a helper like `wkx` to parse, or select `ST_AsGeoJSON(geom)` and get JSON). For `hstore`, `pg` ships an optional `pg.types` parser you register once. Bottom line: the driver has no special extension awareness — you handle the extension types the same way you'd handle any custom type, often by shaping the SELECT to return a JS-friendly representation.

---

## 12. ORM Comparison

The recurring theme: ORMs can *enable* extensions via migrations and *use* the resulting columns/types, but the extension-specific operators (`%`, `<->`, `@>`, `ST_DWithin`) almost always require a raw-SQL escape hatch.

### Prisma

**Can it?** Partially. Prisma supports enabling extensions declaratively (preview/GA feature `postgresqlExtensions`) and has native `Unsupported("citext")`, `@db.Uuid`, and a `citext` type mapping. It does **not** express `%`, `<->`, or PostGIS operators.

```prisma
// schema.prisma
datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [pgcrypto, pg_trgm, citext]   // Prisma emits CREATE EXTENSION in migrations
}
model User {
  id    String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  email String @unique @db.Citext
  name  String
}
```

```typescript
// Fuzzy search must drop to raw SQL:
const results = await prisma.$queryRaw`
  SELECT id, name, similarity(name, ${term}) AS score
  FROM "User" WHERE name % ${term}
  ORDER BY similarity(name, ${term}) DESC LIMIT 10`;
```

**Where it breaks:** any trigram/spatial operator; `crypt()` in queries. **Verdict:** great for enabling extensions and mapping `citext`/`uuid`; use `$queryRaw` for search and crypto.

### Drizzle ORM

**Can it?** Yes for schema/types; operators via the `sql` tag. Drizzle has a `customType` mechanism and pg-native `uuid()`; `citext`/`hstore`/`geometry` are defined as custom types.

```typescript
import { sql } from 'drizzle-orm';
// Fuzzy search:
const rows = await db.execute(sql`
  SELECT id, name FROM users
  WHERE name % ${term}
  ORDER BY similarity(name, ${term}) DESC LIMIT 10`);

// UUID default in schema:
// id: uuid('id').primaryKey().default(sql`gen_random_uuid()`)
```

**Where it breaks:** no first-class `%`/`<->` builders; use `sql`. **Verdict:** clean; the `sql` tag makes extension operators readable and parameterized.

### Sequelize

**Can it?** Enable via a migration; use via `Op` for some patterns and `sequelize.literal` / raw for operators. `DataTypes.UUID` + `defaultValue: Sequelize.literal('gen_random_uuid()')` works.

```javascript
// migration:
await queryInterface.sequelize.query('CREATE EXTENSION IF NOT EXISTS pg_trgm');

// fuzzy search:
const [rows] = await sequelize.query(
  `SELECT id, name FROM users WHERE name % :t
   ORDER BY similarity(name, :t) DESC LIMIT 10`,
  { replacements: { t: term }, type: QueryTypes.SELECT });
```

**Where it breaks:** no operator support for `%`/`<->`; `citext` needs a custom type or raw DDL. **Verdict:** use raw queries for anything extension-specific.

### TypeORM

**Can it?** Enable via a migration (`queryRunner.query('CREATE EXTENSION ...')`); columns typed `uuid` natively (`@PrimaryGeneratedColumn('uuid')`), `citext`/`hstore` via `type: 'citext'` / `type: 'hstore'` column options. Operators via QueryBuilder `.where()` raw fragments.

```typescript
const rows = await repo.createQueryBuilder('u')
  .where('u.name % :t', { t: term })
  .orderBy('similarity(u.name, :t)', 'DESC')
  .limit(10)
  .getMany();
```

**Where it breaks:** the raw `%`/`<->` fragments have no type safety; PostGIS needs the `typeorm` + custom transformer or raw SQL. **Verdict:** QueryBuilder with raw predicate strings works; drop to `.query()` for crypto/spatial.

### Knex.js

**Can it?** Yes — most transparent. Enable via `knex.raw('CREATE EXTENSION ...')` (or `knex.schema` in a migration); operators via `knex.raw` / `.whereRaw`.

```javascript
// migration:
await knex.raw('CREATE EXTENSION IF NOT EXISTS pg_trgm');

// fuzzy search:
const rows = await knex('users')
  .select('id', 'name', knex.raw('similarity(name, ?) AS score', [term]))
  .whereRaw('name % ?', [term])
  .orderByRaw('similarity(name, ?) DESC', [term])
  .limit(10);
```

**Where it breaks:** nothing structurally — but you're writing SQL fragments, so it's "raw with a query-builder wrapper." **Verdict:** best fit for extension-heavy work; SQL-transparent with proper `?` binding.

### ORM Summary Table

| ORM | Enable extension | Map citext/uuid | Trigram `%` / `<->` | crypt()/PostGIS | Verdict |
|-----|------------------|-----------------|---------------------|-----------------|---------|
| Prisma | Declarative `extensions=[]` | `@db.Citext`, `@db.Uuid` | `$queryRaw` | `$queryRaw` | Best for enable + type mapping |
| Drizzle | migration | `customType` / `uuid()` | `sql` tag | `sql` tag | Clean, parameterized |
| Sequelize | raw in migration | `DataTypes.UUID`; citext custom | raw query | raw query | Raw for extension features |
| TypeORM | migration `query()` | `@PrimaryGeneratedColumn('uuid')`, `type:'citext'` | QueryBuilder raw `.where` | raw `.query()` | Works via raw predicates |
| Knex | `knex.raw` | manual column types | `.whereRaw` / `knex.raw` | `.raw` | Most transparent |

---

## 13. Practice Exercises

### Exercise 1 — Basic

You are on a fresh PostgreSQL cluster. Write the statements to:
1. Discover whether `pg_trgm` is *available* on the server (files present) but not yet created in your current database.
2. Create it (idempotently) into a dedicated schema called `extensions`.
3. List every object the extension owns.
4. Confirm the installed version.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines prior topics)

Using `citext` (this topic) and what you know about UNIQUE constraints and indexes (earlier phases), design a `users` table where:
- `email` is case-insensitively unique (`Bob@x.com` and `bob@x.com` cannot coexist),
- the primary key is a database-generated UUID,
- there is a fuzzy-searchable `full_name` column with an appropriate index for `ILIKE '%term%'` and for `ORDER BY full_name <-> 'term' LIMIT n`.

Then write the single query that returns the 5 closest name matches to `'katharine'`, with their similarity score, using the index.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; naive answer is wrong)

Your app stores passwords via `crypt(pw, gen_salt('bf', 10))`. Security asks you to (a) raise the bcrypt cost to 12 for all *future* logins and (b) transparently upgrade each existing user's stored hash to cost 12 **the next time they successfully log in** (you cannot re-hash offline because you don't have plaintext). Also, a junior wrote the "find users to force-migrate" query as:

```sql
SELECT id FROM users WHERE password_hash LIKE '$2a$10$%';
```

1. Explain what is wrong or fragile about the junior's detection query.
2. Write the correct login-time flow (in SQL + a sentence of app logic) that verifies the password against the old hash and, on success, updates the stored hash to cost 12 — all without ever storing plaintext.
3. Why can this upgrade only happen at login time and never in a batch job?

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

A search box over a 40M-row `products(name, description)` table is timing out. The current query is:

```sql
SELECT id, name FROM products
WHERE name ILIKE '%' || $1 || '%' OR description ILIKE '%' || $1 || '%'
ORDER BY name;
```

Walk through: what does EXPLAIN show today, which extension and which index type(s) fix it, what is the exact DDL, does the `OR` across two columns change your indexing strategy, and what minimum-search-length guard would you add and why?

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What is a PostgreSQL extension, and how do you install one?**

A: An extension is a packaged bundle of database objects — functions, types, operators, sometimes index support — that you add to a database as a unit. You install it with `CREATE EXTENSION extension_name;` (often `CREATE EXTENSION IF NOT EXISTS name;` in migrations so it's idempotent). Examples are `pg_trgm` for fuzzy search, `pgcrypto` for hashing and UUIDs, and `citext` for case-insensitive text. You can see what's available with `SELECT * FROM pg_available_extensions;` and what's installed with `\dx`.

**Follow-up:** *Does `CREATE EXTENSION` in one database make it available in every database on the server?* — No. `CREATE EXTENSION` is per-database. The extension's *files* on the server are shared cluster-wide (installed via an OS package), but you must run `CREATE EXTENSION` in each database that needs it.

---

**Q: How would you make a `LIKE '%term%'` search fast?**

A: A leading-wildcard `LIKE`/`ILIKE` can't use a normal B-tree index, so it Seq Scans. Install `pg_trgm` and create a GIN index with the trigram operator class: `CREATE INDEX ON t USING gin (col gin_trgm_ops);`. The trigram index lets PostgreSQL find candidate rows by matching three-character substrings of the search term, turning a full-table scan into a fast bitmap index scan.

**Follow-up:** *What index would you use if you also needed to return results ranked by closeness, best match first?* — A GiST index with `gist_trgm_ops`, which supports the `<->` distance operator for `ORDER BY col <-> 'term' LIMIT n` (KNN). GIN doesn't support `<->`.

---

**Q: How do you store passwords securely in PostgreSQL?**

A: Use `pgcrypto`'s `crypt()` with `gen_salt('bf', 12)` for bcrypt: store `crypt(password, gen_salt('bf', 12))`. To verify, compare `password_hash = crypt(input, password_hash)` — passing the stored hash back as the salt re-derives it. The salt and cost are embedded in the stored string, and bcrypt is deliberately slow to resist brute force. (Alternatively, hash in the application with a bcrypt/argon2 library.) Never store plaintext or a fast unsalted hash like MD5.

**Follow-up:** *Why pass the stored hash as the salt during verification?* — Because a bcrypt hash string encodes its own salt and cost factor. Feeding it back to `crypt()` reproduces the exact same hashing parameters, so if the input password matches, the output equals the stored hash.

---

### Principal Level

**Q: You run `CREATE EXTENSION pg_stat_statements;` and it succeeds, but `SELECT * FROM pg_stat_statements;` errors that it must be loaded via shared_preload_libraries. Explain precisely why, and what the correct setup is.**

A: `pg_stat_statements` has two parts: the SQL objects (the view and its accessor functions), which `CREATE EXTENSION` installs into the catalog, and a C library that hooks the executor (`ExecutorEnd`) and accumulates per-normalized-query statistics in **shared memory**. Shared memory is sized and allocated only at server startup, and the executor hook is only installed if the library is preloaded. So `CREATE EXTENSION` alone creates a view with no backing store. The fix: add `shared_preload_libraries = 'pg_stat_statements'` to `postgresql.conf` (or the managed parameter group), **restart** the server, then `CREATE EXTENSION`. On a managed host you edit the parameter group and trigger a reboot rather than editing the file. This is also why it should be enabled on day one — you can't retroactively capture statistics for load that already happened.

**Follow-up:** *How do you use it to decide what to optimize?* — Order by `total_exec_time` (frequency × per-call cost) to find where the CPU actually goes; distinguish high-`calls`/low-`mean` (fix with batching/caching/N+1) from high-`mean` (take to `EXPLAIN ANALYZE`); watch `stddev_exec_time` for plan instability and `shared_blks_read` for cache misses. Reset with `pg_stat_statements_reset()` after a deploy to measure new code cleanly.

---

**Q: A teammate wants to drop `pg_trgm` because "we switched to Elasticsearch." What do you check before running `DROP EXTENSION`, and what's the danger of `CASCADE`?**

A: First audit dependencies: any index built with `gin_trgm_ops`/`gist_trgm_ops` depends on the extension via `pg_depend`. A plain `DROP EXTENSION pg_trgm;` (RESTRICT is the default) will *refuse* with a DETAIL listing those indexes — which is the safe behavior. The danger is `DROP EXTENSION pg_trgm CASCADE;`: CASCADE silently drops every dependent object, i.e. all those trigram indexes, so any residual `LIKE '%x%'`/`%` query in the app instantly reverts to Seq Scans in production. Worse, for a type-providing extension like `citext`, CASCADE can drop the **columns** typed `citext` — actual data loss. Correct procedure: enumerate dependents (`pg_indexes` filtered on the opclass, or `pg_depend`), remove them deliberately with review, confirm nothing in the app still uses the operators, then `DROP EXTENSION` with the default RESTRICT so the engine double-checks you.

**Follow-up:** *How does `pg_dump` treat this extension, and why does it matter for restores?* — `pg_dump` emits `CREATE EXTENSION pg_trgm;`, not the extension's internal function/operator definitions, plus the `CREATE INDEX ... gin_trgm_ops` for your indexes. On restore, the target server must already have the `pg_trgm` binary available or the restore fails at `CREATE EXTENSION`. So a restore onto a server missing the contrib package breaks — a classic backup-doesn't-restore incident.

---

**Q: When would you reach for PostGIS or TimescaleDB instead of modeling in plain tables, and what specifically do they give you that you can't easily replicate?**

A: **PostGIS** when the workload involves geometry/geography — distance, containment, intersection, nearest-neighbor. The thing you can't easily replicate is **indexed** spatial queries: a GiST operator class over bounding boxes makes `ST_DWithin` (within radius) and `<->` (KNN nearest) index-accelerated. Storing lat/lng as two floats and computing Haversine in app code cannot use an index — you Seq Scan and sort in the app, which collapses past a few thousand rows. **TimescaleDB** when you have high-volume, append-mostly, time-keyed data (metrics, ticks, sensor data) that will exceed hundreds of millions of rows. The hypertable transparently partitions by time into chunks, so inserts stay fast (you always append into a small recent chunk rather than a giant single B-tree), time-range queries prune old chunks, retention is an O(1) chunk drop instead of a mass `DELETE`, and continuous aggregates maintain rollups incrementally. You *could* hand-roll time partitioning with native declarative partitioning (Topic 62), but you'd rebuild retention, compression, chunk management, and incremental rollups yourself. For general OLTP, neither adds value — they're specialized tools for spatial and time-series shapes respectively.

**Follow-up:** *TimescaleDB needs `shared_preload_libraries`; what does that imply for adopting it?* — It requires a server restart and provider support (not every managed host allows it), so it's an infrastructure decision, not just a migration. You confirm it's on the provider's allowlist, set the preload via the parameter group, reboot, then `CREATE EXTENSION timescaledb;` and `create_hypertable()`.

---

## 15. Mental Model Checkpoint

1. You ran `CREATE EXTENSION pgcrypto;` successfully in database `A`. In database `B` on the same server, `gen_random_uuid()` errors as undefined (on a pre-13 server). Why — and what are the two distinct install layers involved?

2. `WHERE name % 'smith'` uses your `gin_trgm_ops` index, but `WHERE similarity(name,'smith') > 0.3` Seq Scans, even though they can return the same rows. Explain the planner's reasoning for the difference.

3. You have both a `gin_trgm_ops` and a `gist_trgm_ops` index on `users.name`. For `SELECT ... WHERE name ILIKE '%lee%'` which does the planner prefer, and for `ORDER BY name <-> 'lee' LIMIT 10` which is required? Why can't one index serve both optimally?

4. During verification, why is `password_hash = crypt(input, password_hash)` correct, and what would break if you instead stored the salt in a separate column and forgot to include the cost factor?

5. A `DROP EXTENSION citext CASCADE;` ran in a migration and a table lost a column. Trace the catalog mechanism (`pg_depend`) that caused this, and state the safe procedure.

6. Your `pg_dump` restores fine on staging but fails on a new production host at the very first extension line. What is almost certainly different about the two servers, and why does `pg_dump` not include the extension's function bodies?

7. You switched a high-write table's primary key from `BIGINT` identity to `gen_random_uuid()` and insert throughput dropped while the index grew faster. Explain the index-locality mechanism, and what UUIDv7 changes about it.

---

## 16. Quick Reference Card

```sql
-- ── Discover ────────────────────────────────────────────────
SELECT * FROM pg_available_extensions;          -- available on server (files present)
SELECT * FROM pg_available_extension_versions WHERE name='pg_trgm';
\dx                                              -- installed in THIS database
\dx+ pg_trgm                                     -- objects the extension owns
SELECT extname, extversion FROM pg_extension;    -- catalog view of installed

-- ── Install / manage ────────────────────────────────────────
CREATE EXTENSION IF NOT EXISTS pg_trgm;          -- idempotent (per DATABASE!)
CREATE EXTENSION postgis SCHEMA extensions;      -- keep out of public
CREATE EXTENSION postgis_topology CASCADE;       -- auto-install dependencies
ALTER  EXTENSION pg_trgm UPDATE;                 -- run upgrade scripts
DROP   EXTENSION pg_trgm;                         -- RESTRICT default (safe: refuses if used)
DROP   EXTENSION citext CASCADE;                  -- DANGER: drops dependent cols/indexes

-- ── pg_stat_statements (needs shared_preload_libraries + restart) ──
SELECT queryid, left(query,80), calls,
       round(total_exec_time::numeric,1) total_ms,
       round(mean_exec_time::numeric,3)  mean_ms
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;
SELECT pg_stat_statements_reset();

-- ── pgcrypto ────────────────────────────────────────────────
gen_random_uuid()                                -- v4 UUID (core since PG13)
crypt(pw, gen_salt('bf', 12))                    -- store bcrypt hash
password_hash = crypt(input, password_hash)      -- verify
encode(digest('x','sha256'),'hex')               -- sha-256 hex
pgp_sym_encrypt(txt, key) / pgp_sym_decrypt(...) -- column encryption

-- ── uuid-ossp (legacy / deterministic) ──────────────────────
CREATE EXTENSION "uuid-ossp";                    -- note the quotes
uuid_generate_v4();  uuid_generate_v5(uuid_ns_url(), 'x');  -- deterministic

-- ── pg_trgm (fuzzy / LIKE acceleration) ─────────────────────
CREATE INDEX ON t USING gin  (col gin_trgm_ops);  -- LIKE/ILIKE/% containment (fast)
CREATE INDEX ON t USING gist (col gist_trgm_ops); -- supports <-> KNN ordering
col % 'term'          -- indexable similarity filter (>= similarity_threshold)
col <-> 'term'        -- distance = 1 - similarity; ORDER BY for KNN (GiST)
similarity(a,b)       -- 0..1 score (NOT indexable in WHERE — use % instead)
SET pg_trgm.similarity_threshold = 0.4;           -- controls the % operator
-- Gotcha: similarity()>x in WHERE = Seq Scan; use col % 'x' for index.

-- ── citext / hstore ─────────────────────────────────────────
email CITEXT UNIQUE          -- case-insensitive uniqueness, indexed
attrs HSTORE; attrs->'k'; attrs?'k'; attrs @> 'k=>v'   -- flat kv (prefer jsonb for new)

-- ── PostGIS / TimescaleDB (concepts) ────────────────────────
CREATE INDEX ON shops USING gist (geom);          -- spatial index
ORDER BY geom <-> ST_MakePoint(x,y) LIMIT 10;     -- nearest-N (indexed)
ST_DWithin(a::geography, b::geography, 500)       -- within 500m (indexed)
SELECT create_hypertable('metrics','time');       -- time-series partitioning
```

**Perf rules of thumb**
- Trigram GIN turns `ILIKE '%x%'` from Seq Scan into ~ms bitmap scan; mandatory past ~1M rows.
- Use `col % 'x'` (indexable), not `similarity(col,'x') > n` (Seq Scan), for filtering.
- GIN for containment/`%`; GiST for `<->` KNN ranking. Both if you need both.
- `crypt()` bcrypt is intentionally slow (~ms) — only ever on a single narrowed row, never a scan.
- Random UUID v4 hurts index locality on high-write tables; prefer UUIDv7.
- `pg_stat_statements` needs a restart and should be on from day one.
- Enforce a ≥3-char minimum on fuzzy search inputs (short terms defeat trigrams).

**Interview one-liners**
- "`CREATE EXTENSION` is per-database; the OS package is per-cluster — two separate layers."
- "A leading-wildcard `LIKE` can't use B-tree; `pg_trgm` GIN makes it indexable."
- "`DROP EXTENSION ... CASCADE` silently drops dependent indexes and columns — audit `pg_depend` first."
- "`pg_dump` records `CREATE EXTENSION`, not the extension's internals — the restore target must have the binary."
- "`similarity()` in WHERE isn't indexable; the `%` operator is."
- "`citext` gives you a case-insensitive UNIQUE constraint for free."
- "PostGIS = indexed spatial (nearest/within); TimescaleDB = hypertables for time-series scale."

---

## Connected Topics

- **Topic 72 — Row Level Security**: the previous topic; like extensions, RLS is a PostgreSQL-specific power feature — and `pgcrypto`/`citext` columns are frequently what RLS policies filter on.
- **Topic 74 — The 20 Most Common SQL Interview Questions**: the next topic; extension knowledge (fuzzy search, UUIDs, pg_stat_statements) shows up as senior-level differentiators there.
- **Topic 40 — Indexes and SARGability**: why a leading-wildcard `LIKE` can't use a B-tree — the exact problem `pg_trgm` GIN/GiST solves.
- **Topic 62 — Table Partitioning**: TimescaleDB hypertables are transparent time-partitioning; chunk exclusion is partition pruning.
- **Topic 18 — The N+1 Query Problem**: `pg_stat_statements` is how you *detect* N+1 in production (high `calls`, low `mean_ms`).
- **Topic 24 — ORDER BY and Sorting**: the `<->` KNN operator lets a GiST index satisfy `ORDER BY` without a Sort node.
- **Topic 07 — NULL in Depth**: extension operators (`%`, `@>`) follow the same three-valued-logic rules as any predicate.
- **SQL Internals — GIN and GiST access methods**: extensions plug new operator classes into these generic index frameworks; `gin_trgm_ops` and `gist_trgm_ops` are the canonical examples.
