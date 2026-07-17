# 91 — Data Lakes and Data Warehouses
## Category: HLD Components

---

## What is this?

A **data warehouse** is a separate database built for one job: answering big analytical questions ("what was our revenue per country per month for the last 3 years?"). A **data lake** is a giant, cheap bucket of raw files where you dump *everything* — logs, JSON events, CSVs, images — before you even know what you'll do with it.

Think of a **restaurant kitchen** vs a **warehouse of ingredients**. Your production database is the kitchen line: fast, precise, handling one order at a time under pressure. You do NOT let someone walk into that kitchen mid-dinner-rush and start counting every tomato in the building. Counting happens in the back warehouse, on a copy of the inventory, where nobody is waiting on you.

---

## Why does it matter?

Because the single most common junior mistake in a real company is running an analytics query against the production database — and taking down checkout.

**What breaks if you don't understand this:**
- The finance team asks for a "quick report." You write a `GROUP BY` over 3 years of orders. Ten seconds later, the site is down and you don't know why.
- Your ML team asks for "all the raw clickstream from last year." You deleted it, because you only kept the aggregated summary. That data is gone forever.
- Your dashboard takes 40 seconds to load and you keep adding indexes, which never helps, because indexes were never the problem — your *storage layout* is.

**Interview angle:** Almost every data-heavy design question ("design an analytics dashboard", "design Uber's surge pricing", "design a recommendation system") has a hidden second half: *where does the analytical data live, and how does it get there?* Saying "we'll add a read replica and query it" is a mid-level answer. Saying "OLTP stays on Postgres; we CDC into a columnar warehouse for OLAP because a row store scans 20x more bytes than it needs" is a senior answer.

**Real-work angle:** Every company past ~30 engineers has a warehouse or a lakehouse, an ingestion pipeline, and a dbt repo. You will touch this within your first year.

---

## The core idea — explained simply

### The Library Analogy

Imagine a library that stores information about 100 million books. Each book record has 5 facts: `title`, `author`, `country`, `year`, `price`.

**Library A (a row-oriented library)** keeps one index card per book, filed in a drawer. Each card has all 5 facts written on it. To find *"what is book #4,000,000's price?"* you walk to one drawer, pull one card, done — **instant**. This library is brilliant at "give me everything about ONE book."

But now the head librarian asks: *"What is the average price of all 100 million books?"* Librarian A must pull all 100 million cards, read the price off each one, and put it back. She physically handles the title, author, country and year on every card too — she can't avoid touching them, they're printed on the same card. She reads ~5x more information than she needed.

**Library B (a column-oriented library)** does something strange. It doesn't keep cards. It keeps **five long scrolls**: one scroll listing all 100M titles in order, one listing all 100M authors, one listing all 100M prices, and so on. To answer *"average price"*, Librarian B unrolls **only the price scroll** and adds it up. She never touches titles or authors at all. **5x less reading.** And because the country scroll is just `"India, India, India, USA, USA, ..."` repeated over and over, she compresses it into a note that says *"India × 41,000,000, then USA × 22,000,000..."* — the scroll shrinks by 100x.

But ask Library B *"tell me everything about book #4,000,000"* and she must unroll **five separate scrolls** and find position 4,000,000 in each. Terrible.

**That's the whole game.** Your production database is Library A. Your data warehouse is Library B. They are physically laid out for opposite questions, and no amount of indexing turns one into the other.

| In the analogy | In the system |
|---|---|
| Index card per book | **Row-oriented storage** (Postgres, MySQL) — a whole row is contiguous on disk |
| Long scroll per fact | **Columnar storage** (Redshift, BigQuery, Snowflake, ClickHouse, Parquet) |
| "Price of book #4M?" | **OLTP query** — point lookup by key, few rows, low latency |
| "Average price of all books?" | **OLAP query** — scan millions of rows, aggregate a few columns |
| Compressing "India × 41M" | **Run-length / dictionary encoding** — only possible when values are adjacent |
| The dinner-rush kitchen | Your **production DB**, which must never be blocked |
| The back warehouse | Your **data warehouse**, where slow queries are welcome |

---

## Key concepts inside this topic

### 1. OLTP vs OLAP — the foundational split

**OLTP** = **O**n**L**ine **T**ransaction **P**rocessing. Your production database. It serves the app.
**OLAP** = **O**n**L**ine **A**nalytical **P**rocessing. Your warehouse. It serves humans asking questions and dashboards.

| | **OLTP** (production DB) | **OLAP** (warehouse) |
|---|---|---|
| **Typical query** | `SELECT * FROM orders WHERE id = 8412` | `SELECT country, SUM(total) FROM orders GROUP BY country` |
| **Rows touched per query** | 1 – 100 | 1,000,000 – 10,000,000,000 |
| **Columns touched per query** | All of them (`SELECT *`) | 2 – 6 out of 80 |
| **Concurrency** | 10,000+ queries/sec | 10 – 100 queries/**minute** |
| **Latency target** | 1 – 50 ms (a user is waiting) | 1 – 60 s (a dashboard is loading) |
| **Optimized for** | **Latency** per query | **Throughput** (bytes scanned/sec) |
| **Writes** | Constant, tiny, random | Bulk loads / appends, huge, batched |
| **Schema** | **Normalized** (3NF) — no duplicated data | **Denormalized** (star schema) — duplication is fine |
| **Storage layout** | **Row-oriented** | **Columnar** |
| **Data age** | Current state ("what is the balance NOW") | History ("what was the balance every day for 3 years") |
| **Examples** | Postgres, MySQL, DynamoDB | Snowflake, BigQuery, Redshift, ClickHouse, DuckDB |

> Recall from [61 — Databases: SQL vs NoSQL] that we chose a database by its *access pattern*. This is the same rule, one level up: OLTP and OLAP are two access patterns so different they need two different *systems*, not two different tables.

### 2. Why you must NEVER run heavy analytics on your production database

This is the interview-ready answer. Memorize the mechanism, not just the conclusion.

An analyst runs this against your live Postgres at 8pm on Black Friday:

```sql
-- Innocent-looking. Career-ending.
SELECT country, DATE_TRUNC('month', created_at) AS month, SUM(total_amount)
FROM orders
WHERE created_at > NOW() - INTERVAL '3 years'
GROUP BY 1, 2;
```

Four things happen, in this order:

1. **It consumes all the I/O.** There is no index that helps — the query genuinely needs 3 years of rows. Postgres does a sequential scan over, say, 300 GB. Your disk has finite IOPS, and this query eats every one of them. Your normal `SELECT * FROM cart WHERE user_id = ?` now waits in line behind it.

2. **It evicts the buffer pool.** This is the killer, and the part most candidates miss. Postgres keeps a hot cache in RAM (`shared_buffers`) holding the pages your fast OLTP queries hit constantly — the user table, the active carts, the hot indexes. Your 300 GB scan pulls 300 GB of cold, ancient pages through that cache, **flushing every hot page out**. Now even after the analyst's query finishes, every OLTP query is missing cache and hitting disk. Your p99 latency stays broken for *minutes afterwards*.

3. **It holds locks / bloats MVCC.** A long-running transaction stops `VACUUM` from cleaning up dead rows. Table bloat grows. Under some isolation levels or with DDL in flight, it blocks writers outright.

4. **Checkout times out.** The app's DB connection pool (say 50 connections) fills with queries that used to take 5 ms and now take 5 s. Requests queue. Health checks fail. The load balancer pulls instances. You are now on an incident call.

**One sentence for the interview:** *"Analytics on the production DB doesn't just make the analytics slow — it makes the transactional workload slow, because it saturates I/O and evicts the buffer-pool cache that OLTP depends on. That contention is the entire reason data warehouses exist."*

*(And no, "just use a read replica" isn't a full fix: replicas are still row-oriented, still scan every column, and long queries on a replica can stall replication or get cancelled by conflicts. It buys you isolation, not efficiency.)*

### 3. Row-oriented vs COLUMNAR storage — the technical heart of the topic

Take one table:

```sql
CREATE TABLE orders (
  id          BIGINT,       -- 8 bytes
  user_id     BIGINT,       -- 8 bytes
  country     VARCHAR(2),   -- 2 bytes
  created_at  TIMESTAMP,    -- 8 bytes
  price       NUMERIC(10,2) -- 8 bytes
);                          -- ≈ 34 bytes/row
```

**Row-oriented layout on disk** (Postgres, MySQL). A whole row lives contiguously in a page:

```
 PAGE 1                                    PAGE 2
┌──────────────────────────────────────┐  ┌────────────────────────────────┐
│ id=1 │ u=90 │ IN │ 2024-01-02 │ 19.99│  │ id=3 │ u=12 │ US │ ... │ 45.00 │
├──────────────────────────────────────┤  ├────────────────────────────────┤
│ id=2 │ u=41 │ US │ 2024-01-02 │ 7.50 │  │ id=4 │ u=90 │ IN │ ... │ 12.25 │
└──────────────────────────────────────┘  └────────────────────────────────┘
   ▲
   └── to read `price`, the disk still delivers id, user_id, country, created_at.
       You cannot fetch 8 bytes out of the middle of a page. You fetch the page.
```

**Columnar layout on disk** (Parquet, Redshift, BigQuery, ClickHouse). Each column lives contiguously in its own set of blocks:

```
 id block          user_id block     country block      created_at block   price block
┌────────────┐    ┌────────────┐    ┌────────────┐     ┌──────────────┐   ┌────────────┐
│ 1,2,3,4,5… │    │ 90,41,12,… │    │ IN,US,US,… │     │ 2024-01-02,… │   │19.99,7.50, │
│            │    │            │    │            │     │              │   │45.00,12.25 │
└────────────┘    └────────────┘    └────────────┘     └──────────────┘   └────────────┘
                                          ▲                                     ▲
                                          │                                     │
                     "IN,IN,IN,IN,US,US…" compresses to               SELECT AVG(price)
                     RLE: [IN×41M][US×22M] — 100x smaller             reads ONLY this block
```

**Do the arithmetic. This is what makes it land.**

100,000,000 orders × 34 bytes/row = **3.4 GB** of raw table data.

Query: `SELECT AVG(price) FROM orders;`

| | Bytes read from disk | Why |
|---|---|---|
| **Row store** | **3.4 GB** | Must pull every page; every page carries all 5 columns |
| **Column store (uncompressed)** | 100M × 8 B = **0.8 GB** | Only the `price` blocks. **4.2x less I/O** |
| **Column store (compressed)** | ≈ **0.2 – 0.3 GB** | Numeric columns compress 3-4x when values are adjacent and similar |

Now the query that really shows it off: `SELECT country, COUNT(*) FROM orders GROUP BY country`, where there are only **50 distinct countries** across 100M rows.

- Row store: **3.4 GB** scanned. Same as before — it always reads everything.
- Column store: the `country` column is 200 MB raw, but it's 100M values drawn from 50 distinct strings. **Dictionary encoding** maps each country to a 1-byte code (50 < 256), giving 100 MB. Then **run-length encoding** collapses long runs of identical adjacent codes into `(value, count)` pairs. Real result: **a few MB.**

**That is a 1000x reduction in bytes read, and it comes from the layout, not from a faster disk.** This is why a $0.02 BigQuery query beats your carefully-indexed Postgres, and why "just add an index" cannot save a row store from a full-table aggregate — an index helps you *skip* rows, and an OLAP query doesn't want to skip rows, it wants *all* of them but only *some columns*.

Two more columnar superpowers worth naming in an interview:
- **Vectorized execution:** because a column is a tight array of identical-typed values, the CPU can process 1024 values per loop iteration using SIMD instructions instead of one row at a time through a branchy interpreter. Often another 10x.
- **Predicate pushdown / min-max pruning:** each block stores its own min and max. `WHERE created_at > '2025-01-01'` lets the engine skip entire blocks without decompressing them.

**The cost of columnar:** single-row inserts are miserable. Writing one order means touching 5 separate column files. Updating one field means rewriting a block. Columnar stores are built for **append + bulk load**, not for `UPDATE users SET last_login = NOW()` 10,000 times a second. That's precisely why you keep both systems.

### 4. The data warehouse and the star schema

A **data warehouse** is a columnar, SQL-native, **curated** store of your business's history.

- **Schema-on-WRITE:** you define the tables up front, and data must conform before it lands. If a row doesn't fit the schema, it's rejected or quarantined. The result is data you can *trust* — when finance says "revenue," everyone gets the same number.
- Structured only. Trusted. Governed. Slower to change.

The dominant modelling pattern is the **star schema**: one big central **fact table** (one row per business event — an order, a click, a payment) surrounded by small **dimension tables** (the nouns: customer, product, date, store).

```
                      ┌────────────────────┐
                      │   dim_date         │
                      │  date_key (PK)     │
                      │  day, month, qtr   │
                      │  is_holiday        │
                      └─────────┬──────────┘
                                │
┌──────────────────┐   ┌────────▼──────────────────┐   ┌──────────────────┐
│  dim_customer    │   │      fact_orders          │   │   dim_product    │
│  customer_key PK │   │  ─────────────────────    │   │  product_key PK  │
│  name            │◀──┤  order_id                 ├──▶│  name            │
│  country         │   │  customer_key   (FK)      │   │  category        │
│  segment         │   │  product_key    (FK)      │   │  brand           │
│  signup_cohort   │   │  date_key       (FK)      │   │  cost_price      │
└──────────────────┘   │  store_key      (FK)      │   └──────────────────┘
                       │  ── measures ──           │
                       │  quantity      INT        │
                       │  unit_price    NUMERIC    │
                       │  discount      NUMERIC    │
                       │  total_amount  NUMERIC    │
                       └────────────┬──────────────┘
                                    │
                          ┌─────────▼──────────┐
                          │    dim_store       │
                          │  store_key (PK)    │
                          │  city, region      │
                          └────────────────────┘
```

The fact table holds **numbers you aggregate** (measures) plus **foreign keys**. It has billions of rows. The dimensions hold **text you filter and group by** and have thousands of rows.

Now the crucial point: **this schema is denormalized on purpose, and that is the exact opposite of what OLTP taught you.**

In OLTP you normalize to avoid duplication, because an update must happen in one place. In OLAP, the data is **immutable history** — a completed order from 2023 never changes — so duplication costs you nothing in correctness. What it *buys* you is fewer joins. Joining a 5-billion-row fact table is expensive; a star schema needs at most one hop from the fact to each dimension, never a chain of six.

| | OLTP (3NF) | OLAP (star) |
|---|---|---|
| **Goal** | Avoid update anomalies | Minimize joins per query |
| **Storage** | Minimal | Redundant, and that's fine (storage is $23/TB/month) |
| **Joins in a typical query** | 1-2 | 1-4, always fact→dim, never dim→dim→dim |
| **Data mutability** | Rows change constantly | Facts are append-only history |

**Slowly Changing Dimensions (SCD), briefly.** A customer moves from India to Germany. If you overwrite `dim_customer.country` (that's **SCD Type 1**), every historical order they ever placed retroactively becomes a German order, and last year's revenue-by-country report silently changes. To preserve history you use **SCD Type 2**: insert a *new row* for that customer with `valid_from` / `valid_to` / `is_current` flags, keep the old row, and let each fact point at the version that was current when the order happened. Interviewers love this because it proves you understand that a warehouse stores *history*, not *current state*.

### 5. The data lake — and the data swamp

A **data lake** is just files in cheap object storage (recall [78 — Blob Storage]: S3, GCS, Azure Blob). No database engine, no server. Just a bucket:

```
s3://acme-lake/
  raw/orders/dt=2026-07-12/part-0001.parquet
  raw/clickstream/dt=2026-07-12/events-0001.json.gz
  raw/support_tickets/dt=2026-07-12/dump.csv
  raw/product_images/sku-8814.jpg
  raw/app_logs/dt=2026-07-12/api-nginx.log.gz
```

- **Schema-on-READ:** you dump the data now, and only decide what it *means* when you query it. A JSON blob with 40 unknown fields is fine — land it, figure it out later.
- **Cheap:** roughly **$23/TB/month** on S3 standard, and cents per TB on Glacier. A warehouse charges more and makes you define a schema first.
- **Any format:** Parquet, JSON, CSV, Avro, logs, images, audio. A warehouse cannot store your product photos. A lake can, and your ML team needs them.
- **Keeps everything, including data you don't yet know you need.** This is the killer property. In 2024 nobody thought the raw "hover before click" events mattered. In 2026 your recommendation model needs them. If you'd only kept the aggregate, you'd have nothing.

**Now the honest failure mode.** A lake with no governance becomes a **data swamp**:

- 4 petabytes of Parquet and nobody knows which folder is authoritative.
- Three teams have written `revenue_final`, `revenue_final_v2`, and `revenue_actual_USE_THIS`. All three disagree.
- Nobody knows who owns a dataset, when it last updated, or whether it's still being written.
- Nobody knows which files contain PII, so when a GDPR deletion request arrives, nobody can honour it.
- The cost of *finding* trustworthy data exceeds the cost of *re-collecting* it.

The cure is a **data catalog** (AWS Glue Catalog, Unity Catalog, DataHub, Amundsen): a searchable registry of what each dataset is, its schema, its owner, its freshness, and its lineage (which pipeline produced it from what). **A lake without a catalog is a swamp.** Say that sentence in the interview.

### 6. The data lakehouse — the convergence that won

For a decade you had to choose: cheap+flexible (lake) or reliable+fast (warehouse). Then a simple, clever idea won: **keep the lake's cheap open files, and add a transactional metadata layer on top of them.**

That layer — **Delta Lake**, **Apache Iceberg**, or **Apache Hudi** — is a set of metadata/manifest files sitting next to your Parquet, tracking exactly which files belong to which version of a table.

```
┌───────────────────────────────────────────────────────┐
│  Query engines: Spark │ Trino │ DuckDB │ Snowflake    │
└───────────────────────────┬───────────────────────────┘
                            │  reads the table, not the files
┌───────────────────────────▼───────────────────────────┐
│  TABLE FORMAT (Iceberg / Delta / Hudi)                │  ← the new layer
│  • which parquet files make up version 47             │
│  • column stats per file (min/max) for pruning        │
│  • ACID commit log — atomic swap of the file list     │
│  • schema + its evolution history                     │
└───────────────────────────┬───────────────────────────┘
┌───────────────────────────▼───────────────────────────┐
│  OBJECT STORAGE (S3 / GCS) — plain Parquet files      │  ← still just a lake
└───────────────────────────────────────────────────────┘
```

What that thin layer buys you:

| Capability | Why it matters |
|---|---|
| **ACID transactions** | A failed Spark job no longer leaves half-written garbage that a dashboard reads. Readers see version 46 or version 47, never a mix. |
| **Schema enforcement + evolution** | A bad upstream deploy can't silently pollute the table with a `price` column that's suddenly a string. You can still *add* a column without rewriting petabytes. |
| **Time travel** | `SELECT * FROM orders VERSION AS OF 42` — reproduce last Tuesday's numbers exactly, or roll back a bad load in one command. |
| **Efficient upserts and deletes** | The one that actually forced adoption. |

That last row deserves its own paragraph. **GDPR / CCPA give a user the right to be forgotten.** In a raw lake, one person's rows are smeared across ten thousand immutable Parquet files inside a petabyte. Deleting them means finding and rewriting every affected file by hand. Iceberg/Delta give you `DELETE FROM events WHERE user_id = 8812` — the layer rewrites only the touched files and atomically commits a new version. A plain lake makes compliance a multi-week engineering project; a lakehouse makes it a SQL statement. **This is the strongest single argument for the lakehouse, and it's the one interviewers rarely hear.**

### 7. ETL vs ELT — and why the industry flipped

**ETL — Extract, Transform, Load (the old way, ~1990–2015):**

```
Postgres ──extract──▶ [ Informatica / a big ETL box ] ──transform──▶ Warehouse
                        (join, clean, aggregate here)                  (load only
                                                                    the clean result)
```

You transformed on a *separate* machine **before** loading, because the warehouse (a Teradata appliance costing millions) was too expensive and too slow to waste on cleaning. You loaded only the polished, final tables.

**ELT — Extract, Load, Transform (the modern way):**

```
Postgres ──extract──▶ Warehouse/Lake ──transform (SQL, in-place)──▶ Warehouse
                       (land RAW first)      run BY the warehouse
```

You land the **raw** data first, untouched, and then transform it **inside** the warehouse using the warehouse's own massive parallel compute.

**Why it flipped — two forces:**
1. **Storage got absurdly cheap.** S3 at $23/TB/month means keeping raw data forever costs less than the meeting where you'd debate deleting it.
2. **Warehouse compute got enormous and elastic.** Snowflake/BigQuery can throw hundreds of cores at one query for 90 seconds and then release them. Your separate ETL box can't compete with that, and it sits idle 95% of the day.

**The decisive benefit of ELT — say this one out loud:**

> With ELT, **you keep the raw data forever, so you can always RE-DERIVE everything.**

Find a bug in your revenue logic that's been wrong for 8 months? Fix the SQL, re-run the transform over the raw history, and every downstream table is corrected. Business changes the definition of an "active user" from 28-day to 7-day? Recompute the whole 5-year history in an hour.

With ETL, the pre-transform data **is gone**. The bug is permanently baked into your history. Your only options are to apologise or to backfill from a source that may no longer exist.

**dbt** is the tool that defined the modern **T**. You write transformations as plain `SELECT` statements in version-controlled `.sql` files; dbt works out the dependency graph, materializes each model as a table or view *inside* the warehouse, runs tests (`not_null`, `unique`, `accepted_values`), and generates lineage docs. It turned "the analytics SQL nobody understands" into a normal software repo with PRs, CI, and tests.

```sql
-- models/marts/fct_daily_revenue.sql   (a dbt model — just SQL, run by the warehouse)
{{ config(materialized='incremental', unique_key='revenue_date') }}

SELECT
    o.order_date            AS revenue_date,
    c.country,
    COUNT(DISTINCT o.order_id) AS orders,
    SUM(o.total_amount)        AS gross_revenue,
    SUM(o.discount)            AS total_discount
FROM {{ ref('stg_orders') }} o          -- ref() builds the DAG; dbt knows stg_orders runs first
JOIN {{ ref('dim_customer') }} c USING (customer_key)
WHERE o.status = 'COMPLETED'
{% if is_incremental() %}
  -- only reprocess the recent window on scheduled runs; a full refresh replays all history
  AND o.order_date >= (SELECT MAX(revenue_date) FROM {{ this }}) - INTERVAL '3 days'
{% endif %}
GROUP BY 1, 2
```

### 8. Separation of storage and compute

The old warehouse (Teradata, early Redshift) bolted disks to CPUs: to store more data you bought more machines, and those machines' CPUs sat idle. Worse, **every team shared the same CPUs** — the marketing team's monster query slowed the finance team's dashboard, and you were back to the contention problem you built the warehouse to escape.

**Snowflake and BigQuery decoupled them.** Data lives once, in object storage. Compute is a separate, ephemeral cluster you spin up on demand.

```
                          ┌──────────────────────────────────┐
                          │  SHARED STORAGE  (S3/GCS)        │
                          │  ONE copy of the data            │
                          └───┬──────────┬──────────┬────────┘
                              │          │          │
              ┌───────────────▼──┐ ┌─────▼───────┐ ┌▼──────────────────┐
              │ Warehouse "BI"   │ │ "DATA_SCI"  │ │ "ETL"             │
              │ SMALL, always on │ │ 4XL, bursty │ │ LARGE, 2am only   │
              │ dashboards       │ │ one huge Q  │ │ nightly loads     │
              └──────────────────┘ └─────────────┘ └───────────────────┘
                 independent CPUs — no contention between them
```

Why this is a big deal, in three bullets you can say in an interview:
- **Elastic per-query scale.** One monster year-end query? Resize the compute cluster to 64x for 10 minutes, then back down. No data movement, because the data never lived on the compute nodes.
- **Pay nothing when idle.** Compute auto-suspends after 60 seconds. Your storage bill continues at $23/TB; your compute bill goes to zero overnight. In the coupled world, those servers billed 24/7.
- **No cross-team contention.** Each team gets its own compute cluster reading the *same single copy* of the data. The data-science team's runaway query cannot slow the CEO's dashboard. It's the OLTP/OLAP isolation principle, applied *within* the warehouse.

---

## Visual / Diagram description

### Diagram 1 — the end-to-end modern data platform (draw this on the whiteboard)

```
  SOURCES                INGESTION                  STORAGE (the lakehouse)               CONSUMERS
┌──────────────┐      ┌─────────────────┐   ┌──────────────────────────────────────┐   ┌─────────────┐
│ OLTP Postgres│──CDC▶│                 │   │  ┌────────────────────────────────┐  │   │  BI / dash  │
│ (orders,users│      │  Debezium       │   │  │ BRONZE  — raw, immutable       │  │──▶│ Looker,     │
│  never query │      │      +          │──▶│  │ exact copy of source, append   │  │   │ Tableau,    │
│  it for OLAP)│      │  Kafka topics   │   │  │ schema-on-read, keep FOREVER   │  │   │ Metabase    │
└──────────────┘      │                 │   │  └───────────────┬────────────────┘  │   └─────────────┘
                      │  (topic 87:     │   │                  │ clean, dedupe,    │
┌──────────────┐      │   streaming)    │   │                  │ cast types, PII   │   ┌─────────────┐
│ App events   │─────▶│                 │   │  ┌───────────────▼────────────────┐  │   │  ML /       │
│ clickstream  │      └─────────────────┘   │  │ SILVER — cleaned, conformed    │  │──▶│  feature    │
└──────────────┘                            │  │ one row per real entity, typed │  │   │  store      │
                      ┌─────────────────┐   │  └───────────────┬────────────────┘  │   └─────────────┘
┌──────────────┐      │ Fivetran /      │   │                  │ aggregate, join   │
│ SaaS APIs    │─────▶│ Airbyte         │──▶│                  │ into star schema  │   ┌─────────────┐
│ Stripe,      │      │ (batch pulls)   │   │  ┌───────────────▼────────────────┐  │   │  Reverse    │
│ Salesforce   │      └─────────────────┘   │  │ GOLD — marts, star schema      │  │──▶│  ETL back   │
└──────────────┘                            │  │ fct_orders + dim_* , trusted   │  │   │  to Salesf. │
                                            │  └────────────────────────────────┘  │   └─────────────┘
                                            │       ▲ dbt runs the arrows above    │
                                            │       Iceberg/Delta = ACID + catalog │
                                            └──────────────────────────────────────┘
```

**What the diagram shows.** Data leaves the OLTP database *without being queried analytically* — **CDC (Change Data Capture)** tails Postgres's write-ahead log, so it reads the replication stream rather than running `SELECT`s, meaning it costs the production DB almost nothing. Debezium publishes each `INSERT`/`UPDATE`/`DELETE` as an event onto Kafka (recall [87 — Batch vs Stream Processing]). Event streams from the app join the same pipeline. Everything lands in **bronze** as raw, immutable files.

Then the **medallion architecture** promotes data through three layers:

| Layer | Contains | Who touches it |
|---|---|---|
| **Bronze** | Raw, exactly as it arrived. Ugly. Never deleted. | Pipelines only |
| **Silver** | Cleaned, deduplicated, correctly typed, PII handled. One row per real-world entity. | Data engineers, ML |
| **Gold** | Business-level aggregates and the star schema. Trusted. This is what "revenue" means. | Analysts, execs, dashboards |

The rule: **each layer is derived only from the layer above it, by code in version control.** If gold is wrong, you fix the SQL and re-derive it from bronze — you never hand-patch gold.

### Diagram 2 — the one-line reason all of this exists

```
   ┌─────────────────────┐                       ┌──────────────────────┐
   │   PRODUCTION DB     │   CDC (log-tailing,   │     WAREHOUSE        │
   │   row-oriented      │   near-zero cost)     │     columnar         │
   │   10k QPS, 5ms      │ ────────────────────▶ │   50 queries/hr, 20s │
   │   normalized        │                       │   denormalized/star  │
   │                     │                       │                      │
   │   ✗ NEVER point a   │                       │   ✓ point every      │
   │     dashboard here  │                       │     dashboard here   │
   └─────────────────────┘                       └──────────────────────┘
        users are waiting                            nobody is waiting
```

---

## Real world examples

### Netflix

Netflix runs one of the largest data lakes in the world on S3 — hundreds of petabytes of playback events, device telemetry, and A/B test results. They **created Apache Iceberg** precisely because plain files on S3 (via Hive) couldn't give them atomic commits, safe schema evolution, or correct results while data was being rewritten underneath a running query. Iceberg is now an industry-standard table format supported by Snowflake, Databricks, Trino, and AWS. Notably, Netflix's playback service itself runs on **Cassandra** — an OLTP store — and analytics never touch it directly; events flow through Kafka into the lake. That's the OLTP/OLAP split at planetary scale.

### Uber

Uber built **Apache Hudi** to solve a specific lakehouse problem: their lake needed **upserts**. A trip record changes several times (requested → matched → in-progress → completed → rated), and rewriting a whole partition of Parquet on every change was untenable. Hudi added incremental upserts and record-level indexes on top of lake files. Uber's platform ingests from Kafka into the lake, and Presto/Trino queries it for everything from surge-pricing analysis to city-ops dashboards.

### A typical mid-size SaaS company (representative architecture)

The most common stack you will actually meet: **Postgres** (OLTP) → **Fivetran or Airbyte** (managed ingestion) → **Snowflake or BigQuery** (columnar warehouse, storage/compute separated) → **dbt** (the T in ELT, building staging → intermediate → mart models) → **Looker or Metabase** (BI). Total setup time: a couple of weeks. This is ELT end to end: raw tables land untouched in a `RAW` schema, dbt builds everything else from them, and nobody has ever pointed a dashboard at production.

---

## Trade-offs

### Warehouse vs Lake vs Lakehouse

| | **Data Warehouse** | **Data Lake** | **Lakehouse** |
|---|---|---|---|
| **Stores** | Structured tables only | Anything (files, images, JSON) | Anything; tables get ACID |
| **Schema** | On WRITE (enforced up front) | On READ (interpret later) | On WRITE for tables, on READ for raw |
| **Cost/TB** | Higher (managed) | Lowest ($23/TB/mo) | Lake cost + small metadata overhead |
| **Query speed** | Fastest for SQL analytics | Slow, no stats, no ACID | Near-warehouse (stats + pruning) |
| **ACID transactions** | Yes | **No** | Yes (Iceberg/Delta/Hudi) |
| **GDPR row deletes** | Easy (`DELETE`) | **Painful** — rewrite files by hand | Easy (`DELETE`, atomic) |
| **ML / raw features** | Poor — data pre-aggregated | Excellent — everything kept | Excellent |
| **Failure mode** | Rigid, slow to add sources, vendor lock-in | **Data swamp** | Extra operational complexity, newer tooling |

### ETL vs ELT

| | **ETL** | **ELT** |
|---|---|---|
| **Transform runs** | On a separate box, before load | Inside the warehouse, after load |
| **Raw data kept?** | **No — it's gone** | **Yes, forever** |
| **Fix a logic bug** | Backfill from source (if it still exists) | Re-run the transform. Done. |
| **Warehouse cost** | Lower (only clean data lands) | Higher (raw data lands too) |
| **Compliance** | Easier — PII can be stripped before landing | Harder — raw PII lands in the lake, needs masking + governance |
| **Best when** | Strict PII rules, or the sink is genuinely weak | Almost always, today |

### Row store vs Column store

| | **Row (Postgres)** | **Column (BigQuery/ClickHouse)** |
|---|---|---|
| **Point lookup by PK** | ~1 ms | 100 ms – seconds (must touch N column files) |
| **Aggregate over 100M rows** | Minutes, and it breaks prod | 1 – 5 s |
| **Single-row `UPDATE`** | Trivial | Expensive (rewrite blocks) |
| **Compression ratio** | 2 – 3x | 5 – 20x+ |
| **Concurrent writers** | Thousands | Few; prefers bulk append |

**The sweet spot:** Keep your OLTP database small, normalized, row-oriented, and *sacred* — never run analytics on it. CDC the changes out into a lakehouse (Iceberg or Delta on S3). Land raw in **bronze** and keep it forever. Transform with dbt into **silver** and **gold**. Point every dashboard and ML job at gold.

**Rule of thumb:** If a query scans more than ~1% of a table and aggregates rather than fetching, it does not belong on your production database. And if you're a startup with under ~100 GB of data, **don't build a lakehouse** — a nightly dump into BigQuery, or even DuckDB reading Parquet on S3, will carry you for years. The medallion cathedral is a solution to a scale problem you may not have yet.

---

## Common interview questions on this topic

### Q1: "Why can't we just run our analytics queries on a read replica of the production database?"

**Hint:** A replica fixes *isolation* (the analyst no longer steals the primary's IOPS) but not *efficiency*. It's still row-oriented, so `SELECT AVG(price)` still drags all 5 columns off disk — you're doing the same 20x-too-much I/O, just somewhere else. It still can't compress a low-cardinality column 100x, still can't vectorize, still has no columnar min/max pruning. And it introduces new problems: long analytical queries on a Postgres replica get killed by recovery conflicts, or block replication and cause the replica to lag. A read replica is a fine *stopgap* below ~50 GB. Past that, you need a different storage layout, not a different server.

### Q2: "Explain columnar storage and quantify why it's faster."

**Hint:** Draw the two layouts. Then do the arithmetic out loud: 100M rows × 5 columns × ~8 bytes = 3.4 GB. A row store reads all 3.4 GB for `AVG(price)` because a whole row is contiguous and disks read pages, not fields. A column store reads only the price column: 0.8 GB, and ~0.2 GB after compression — **10-17x less I/O**. Then add the two multipliers: low-cardinality columns hit dictionary + run-length encoding (100M rows, 50 countries → a few MB, ~1000x), and columnar layout enables SIMD/vectorized execution (another ~10x on CPU). Close with the cost: single-row writes and updates are terrible, which is exactly why you keep the row store for OLTP.

### Q3: "Data lake or data warehouse — which would you pick, and why?"

**Hint:** Refuse the false choice, then justify it. Warehouse = structured, schema-on-write, trusted, fast SQL, but can't hold raw JSON/images and throws away what it doesn't model. Lake = cheap, holds everything (critical for ML, where today's junk is tomorrow's feature), but with no ACID, no schema enforcement, and no catalog it degenerates into a **data swamp**. Modern answer: a **lakehouse** — open Parquet on S3 plus an Iceberg/Delta table layer, giving you the lake's cost and flexibility *with* ACID, schema evolution, time travel, and — the clincher — the ability to satisfy a GDPR deletion request with a single `DELETE` instead of a two-week file-rewriting project.

### Q4: "Why did the industry move from ETL to ELT?"

**Hint:** Two enabling forces and one decisive benefit. Enablers: object storage got absurdly cheap (~$23/TB/month, so hoarding raw data is free), and warehouse compute got elastic and enormous (Snowflake/BigQuery can throw hundreds of cores at one query and then release them — a dedicated ETL box can't). The decisive benefit: **ELT keeps the raw data, so you can always re-derive.** Bug in your revenue logic from 8 months ago? Fix the SQL, replay from bronze, everything downstream is corrected. Under ETL, the pre-transform data is gone and the bug is permanent history. Name dbt as the tool that made the in-warehouse T a normal software practice — SQL models, a DAG, tests, PRs, lineage.

### Q5: "How does data get from the production DB to the warehouse without hurting production?"

**Hint:** **Change Data Capture.** Don't poll with `SELECT * WHERE updated_at > ?` — that's a repeated scan on your hottest table and it silently misses hard-deletes. Instead, tail the **write-ahead log** (Postgres logical replication / MySQL binlog) with Debezium. The DB is already writing that log for durability, so reading it costs production essentially nothing, and it captures every insert, update *and* delete in exact commit order. Debezium publishes to Kafka; a sink writes those events into the bronze layer; the silver layer collapses the change stream into current state (or keeps full history — that's SCD Type 2). Mention the trade-off: CDC gives you low latency and completeness but adds Kafka to your ops surface; a nightly `pg_dump` is dumber, simpler, and completely fine if the business can tolerate day-old numbers.

---

## Practice exercise

### Build a tiny lakehouse on your laptop (30-40 minutes)

You'll feel the row-vs-column difference in your own terminal — that's the point.

**Setup.** `npm install duckdb-async`. DuckDB is an embedded columnar OLAP engine — think "SQLite for analytics." No server.

**Step 1 — generate 5,000,000 fake orders** in Node with 5 columns (`id`, `user_id`, `country` drawn from only 20 distinct values, `created_at` spread over 3 years, `price`). Write them once as a **CSV** (a row-ish format) and once as **Parquet** (columnar) using DuckDB's `COPY (...) TO 'orders.parquet' (FORMAT PARQUET)`.

**Step 2 — compare the files on disk.** Run `ls -lh`. Note the sizes. Explain in one sentence *why* the Parquet file is several times smaller, using the word "dictionary encoding" and the fact that `country` has only 20 distinct values.

**Step 3 — time two queries against each file** from Node, using `console.time()`:
```js
import { Database } from 'duckdb-async';

const db = await Database.create(':memory:');

// OLAP query: touches ONE column out of five.
console.time('parquet-agg');
await db.all(`SELECT country, AVG(price) FROM 'orders.parquet' GROUP BY country`);
console.timeEnd('parquet-agg');

console.time('csv-agg');
await db.all(`SELECT country, AVG(price) FROM 'orders.csv' GROUP BY country`);
console.timeEnd('csv-agg');
```

**Step 4 — now flip it.** Time an *OLTP-shaped* query on the Parquet file: `SELECT * FROM 'orders.parquet' WHERE id = 4123456`. Compare that to the same lookup in a SQLite/Postgres table with a primary-key index on `id`. Which wins, and by how much?

**What to produce:** a short table of your four timings plus the two file sizes, and three sentences answering:
1. Why is the columnar aggregate so much faster?
2. Why does the row store still win the point lookup?
3. Given both results, what is the *architectural* conclusion for a real company?

If your write-up ends with something like *"keep both, and CDC from one to the other"* — you've got it.

---

## Quick reference cheat sheet

- **OLTP** = many tiny queries, few rows by key, low latency, normalized, **row**-oriented. Your production DB.
- **OLAP** = few huge queries, millions of rows, a handful of columns, high throughput, denormalized, **columnar**. Your warehouse.
- **Never run analytics on production.** The reason isn't "it's slow" — it's that a full scan **saturates I/O and evicts the buffer-pool cache**, so your fast OLTP queries stay slow for minutes afterwards. Checkout goes down.
- **A read replica is not a warehouse.** It fixes contention, not the 20x-too-much-I/O problem, because it's still row-oriented.
- **Columnar wins** because `SELECT AVG(price)` reads only the price column: often **10-50x less I/O**, plus 5-20x compression (dictionary + run-length on low-cardinality columns), plus SIMD vectorization, plus min/max block pruning.
- **Columnar loses** at single-row inserts, updates, and point lookups. That's why you keep both systems.
- **Warehouse** = schema-on-**write**, structured, curated, trusted, SQL. **Lake** = schema-on-**read**, raw files in blob storage, cheap, keeps everything.
- **Star schema** = one central **fact** table (measures + FKs, billions of rows) surrounded by small **dimension** tables. **Denormalized on purpose** — joins are the expensive thing at analytical scale, and facts are immutable history so duplication is harmless.
- **SCD Type 2** = version your dimension rows (`valid_from`/`valid_to`/`is_current`) so last year's report doesn't silently change when a customer moves country.
- **Data swamp** = a lake with no catalog, no owners, no lineage. Petabytes nobody can find or trust. **A lake without a catalog is a swamp.**
- **Lakehouse** = cheap open lake files + a transactional table layer (**Iceberg / Delta / Hudi**) → ACID, schema evolution, **time travel**, and efficient upserts/deletes. The GDPR argument is the strongest one: deleting one user out of a petabyte becomes a `DELETE` statement.
- **ETL** transforms before loading and **throws the raw data away**. **ELT** lands raw first and transforms in-warehouse — so you can always **re-derive** history when logic changes or a bug is found. Storage got cheap; warehouse compute got huge. **dbt** owns the modern T.
- **Medallion:** **bronze** (raw, immutable, forever) → **silver** (cleaned, conformed, typed) → **gold** (aggregated marts / star schema, trusted). Each layer derived from the one above it, by code in git.
- **CDC** (Debezium tailing the WAL/binlog) moves data out of OLTP at near-zero cost and captures deletes — never poll with `updated_at >`.
- **Separation of storage & compute** (Snowflake/BigQuery): one copy of the data, many independent compute clusters. Burst up for one query, pay zero when idle, and no team's query slows another's.
- **Don't over-build.** Under ~100 GB, a nightly dump into BigQuery — or DuckDB over Parquet — beats a medallion cathedral.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [87 — Batch vs Stream Processing](./87-batch-vs-stream-processing.md) — the ingestion half of this picture: Kafka/CDC streams and nightly batch jobs are how data actually gets into the lake |
| **Next** | [68 — Event-Driven Architecture](./68-event-driven-architecture.md) — the events flowing through your services are the raw material your bronze layer captures |
| **Related** | [61 — Databases: SQL vs NoSQL](./61-databases-sql-vs-nosql.md) — the OLTP side of the split; a warehouse is what you add *next to* whatever you chose there |
| **Related** | [78 — Blob Storage](./78-blob-storage.md) — a data lake *is* blob storage (S3/GCS) with Parquet files and a catalog on top |
| **Related** | [62 — Database Indexing](./62-database-indexing.md) — indexes make a row store skip rows; understanding why that never helps a full-table aggregate is what makes columnar click |
