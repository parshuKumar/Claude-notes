# 77 — Capstone I: Design a Schema From Requirements
## Phase: Capstones

---

## ELI5 — The Simple Analogy

An architect handed a napkin sketch and asked to design a building.

The napkin says *"a school, about 400 students, near the river."* A junior architect starts drawing rooms. **An experienced one starts asking questions**: how many students in ten years? Does the river flood? Who inspects it? What happens when the roof leaks at 2 a.m.?

★ **The drawing is the easy part. The interrogation is the job.**

And the thing that separates the two: ★ **the experienced architect knows which decisions are cheap to change later and which are poured in concrete.** Paint colour: change it any time. Foundation depth: never. **So they spend their thinking budget on the foundations and make defensible defaults everywhere else.**

★ **A database schema has the same structure.** Column names are paint. **Primary keys, partition keys, tenancy boundaries and transaction boundaries are concrete.**

---

## Where this fits in the big picture

```
   Phases 1–8: ★ 76 topics of mechanism, decision and consequence
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 77 CAPSTONE I: DESIGN ← YOU ARE HERE         │
        │ ★ one problem, end to end, every phase used  │
        └────────────────────┬─────────────────────────┘
                             ▼
              78 scale it under pressure
              79 review it as a principal engineer
```

★ **This is not new material.** It is the **method** for applying everything already covered, in the order a real design happens, with the decisions made explicit and defensible.

---

## What is this?

A repeatable procedure for turning requirements into a schema you can defend.

```
 ★ TEN STEPS. ★ THE FIRST FIVE PRODUCE NO SQL AT ALL.

 ① ★ INTERROGATE THE REQUIREMENTS      (Topic 20's six questions)
 ② ★ ENUMERATE ACCESS PATTERNS         — ★ the real specification
 ③ ★ IDENTIFY INVARIANTS               — ★ what must never be true
 ④ ★ FIND THE HOT PATHS                — ★ where the volume is
 ⑤ ★ DECIDE THE ONE-WAY DOORS          — ★ keys, tenancy, boundaries
 ⑥ MODEL ENTITIES AND RELATIONSHIPS    (20–23)
 ⑦ NORMALISE, THEN DECIDE WHAT TO BEND (29–38, 53–55)
 ⑧ INDEX FROM THE ACCESS PATTERNS      (10–19)
 ⑨ CONSTRAIN THE INVARIANTS            (24, 43, 50)
 ⑩ ★ WRITE DOWN WHAT YOU CHOSE AND WHY

 ⇒ ★ THE DISCIPLINE IS THE ORDER. Starting at ⑥ — which is where
   most designs start — ★ produces a schema that cannot answer
   the questions it will be asked.
```

★ **And the framing that makes it defensible:** *every decision is either a **one-way door** (expensive to reverse) or a **two-way door** (cheap). Spend your judgement on the one-way doors and take sensible defaults on the rest.*

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE SCHEMA IS THE LONGEST-LIVED ARTEFACT YOU WILL
   PRODUCE, AND THE HARDEST TO CHANGE.

   application code   ⇒ ★ rewritten every 2–3 years
   the framework      ⇒ ★ replaced
   the infrastructure ⇒ ★ migrated
   ★ THE SCHEMA       ⇒ ★ still there, ★ with every mistake
                        still in it

 ★ AND THE ASYMMETRY IS BRUTAL:
   ★ adding a column          ⇒ minutes
   ★ adding an index          ⇒ minutes (CONCURRENTLY)
   ★ changing a primary key   ⇒ ★ a multi-quarter migration (72)
   ★ adding tenancy later     ⇒ ★ every table, every query, every
                                index (69)
   ★ changing a partition key ⇒ ★ dual-write + backfill + shadow
                                reads (72, 62)

 ⇒ ★ THE ONE-WAY DOORS ARE WHERE THE THINKING GOES.
```

---

## The physical reality

### Step 1 — the interrogation (Topic 20's six questions, extended)

```
 ★ NEVER ACCEPT A REQUIREMENT AT FACE VALUE. ASK:

 ★ ① "ONE OR MANY?" — for every relationship, in both directions
    "a user has an address" ⇒ ★ one? billing and shipping?
      historical ones? shared between users?
    ⇒ ★ this single question determines FK direction and whether
      a join table exists.

 ★ ② "ALWAYS, OR SOMETIMES?" — every NULLable column is a
    business question
    "orders have a delivery date" ⇒ ★ before dispatch? for digital
      goods? for cancelled orders?
    ⇒ ★ NOT NULL is a claim about the business, not the data.

 ★ ③ "CAN IT CHANGE?" — and if so, ★ do we need the old value?
    ⇒ ★ THE SNAPSHOT-vs-COPY TEST (Topic 54's gate 3).
    ⇒ ★ "the price" on an order is ★ never the product's current
      price. It is `unit_price_charged`, frozen, ★ and that is
      not denormalisation.

 ★ ④ "WHAT IS THE VOLUME, IN THREE YEARS?"
    ⇒ ★ rows per table, ★ rows per parent, ★ writes per second on
      the hottest key
    ⇒ ★ this decides partitioning (59), hot rows (61) and whether
      a maintained counter is viable (55).

 ★ ⑤ "WHAT MUST NEVER BE TRUE?"
    ⇒ ★ the invariants. ★ These become constraints, not comments.

 ★ ⑥ "WHO IS ALLOWED TO SEE IT?"
    ⇒ ★ tenancy. ★ THE MOST EXPENSIVE THING TO ADD LATER (69).

 ★ ⑦ "HOW LONG DO WE KEEP IT?"  ← ★ the one nobody asks
    ⇒ ★ retention decides partitioning, and ★ retrofitting
      partitioning to a 4 TB table is a project (59).

 ★ ⑧ "WHAT HAPPENS WHEN IT FAILS HALFWAY?"
    ⇒ ★ transaction boundaries, idempotency keys, outbox (52).
```

### Step 2 — access patterns are the actual specification

```
 ★ WRITE THEM AS SENTENCES, WITH VOLUMES. ★ Not as an ERD.

   ★ "Given a customer, list their last 20 orders, newest first."
     ⇒ ★ 8,000/min · ★ p99 < 100 ms
   ★ "Given an order id, show the order with all its items."
     ⇒ ★ 12,000/min · p99 < 50 ms
   ★ "Given a seller and a date range, total revenue."
     ⇒ ★ 40/min · p99 < 2 s
   ★ "Find orders by tracking number."
     ⇒ ★ 40,000/min · ★ p99 < 20 ms

 ⇒ ★ EACH SENTENCE BECOMES:
   • ★ an index (10–19)
   • ★ or a denormalisation, IF the gates pass (54)
   • ★ or a rollup, if it's an aggregate (56)
 ⇒ ★ AND A PATTERN WITH NO INDEX IS A SEQUENTIAL SCAN YOU HAVE
   NOT NOTICED YET.

 ★ THE TEST OF COMPLETENESS: ★ can you point at the index that
   serves every sentence? ★ If not, the design is unfinished.
```

### Step 3 — invariants become constraints, not documentation

```
 ★ EVERY "MUST NEVER" IS A CONSTRAINT, AN EXCLUSION, OR AN
   ISOLATION LEVEL. ★ Never a code comment.

   "an order's total must equal the sum of its items"
     ⇒ ★ a maintained column + a reconciler, or computed on read
   "stock must never go negative"
     ⇒ ★ CHECK (stock >= 0) + an atomic UPDATE (43, 49)
   "no two bookings may overlap for one room"
     ⇒ ★ EXCLUDE USING gist (24, 43)
   "an email is unique per tenant"
     ⇒ ★ UNIQUE (tenant_id, email)
   "a shipment cannot be delivered before it is dispatched"
     ⇒ ★ CHECK (delivered_at IS NULL OR delivered_at >= dispatched_at)
   "an org may not exceed its seat limit"
     ⇒ ★ no constraint can express it ⇒ ★ SERIALIZABLE, or a
       counter row (43, 50)

 ⇒ ★ THE CLASSIFICATION MATTERS:
   ★ per-row rule            ⇒ CHECK
   ★ uniqueness              ⇒ UNIQUE
   ★ overlap                 ⇒ EXCLUDE
   ★ referential             ⇒ FOREIGN KEY
   ★ across rows (count/sum) ⇒ ★ SERIALIZABLE or a materialised
                               counter — ★ the only case needing
                               isolation
```

### Step 5 — the one-way doors, in order of cost

```
 ★ ① TENANCY MODEL — ★ THE MOST EXPENSIVE TO CHANGE
    shared tables + tenant_id + ★ RLS (69)
      ✓ one schema, one connection pool, easy cross-tenant ops
      ✓ ★ RLS makes isolation structural
      ✗ ★ tenant_id must lead EVERY index
    schema-per-tenant
      ✓ strong isolation, easy per-tenant restore
      ✗ ★ N schemas × M tables migrations; ★ connection churn
    database-per-tenant
      ✓ strongest isolation
      ✗ ★ does not scale past a few hundred tenants
    ⇒ ★ DEFAULT: shared + tenant_id + FORCED RLS.
    ⇒ ★ AND: ★ ADD IT ON DAY ONE EVEN IF THERE IS ONE TENANT.
      Retrofitting touches every table, query and index.

 ★ ② PRIMARY KEY TYPE (22)
    bigserial  ✓ small, ordered, ★ great B-tree locality
               ✗ ★ leaks volume; ★ collides across shards
    ★ UUIDv7   ✓ ★ globally unique, ★ time-ordered (★ good
                 locality, unlike v4), opaque
               ✗ 16 bytes
    UUIDv4     ★ ✗ NEVER as a PK — ★ random insert points destroy
                 B-tree locality and cache hit rates
    Snowflake  ✓ 8 bytes, ordered, ★ shard-aware
               ✗ needs a generator
    ⇒ ★ DEFAULT: ★ bigserial for internal ids, ★ UUIDv7 for
      anything exposed externally or generated client-side.

 ★ ③ MONEY AND TIME (26)
    ★ money ⇒ ★ bigint in minor units. ★ NEVER float. ★ NEVER
      `money`.
    ★ time ⇒ ★ timestamptz, always. ★ `timestamp` is a bug
      waiting for a deployment in another region.
    ⇒ ★ THESE ARE NOT PREFERENCES. They are correctness.

 ★ ④ PARTITIONING AND RETENTION (59, 73)
    ⇒ ★ if a table has a retention policy, ★ partition it FROM
      DAY ONE. Retrofitting to 4 TB is a project.
    ⇒ ★ and the partition key is as permanent as a shard key.

 ★ ⑤ AGGREGATE / TRANSACTION BOUNDARIES (39, 70)
    ⇒ ★ what must change atomically defines the aggregate.
    ⇒ ★ in PostgreSQL this is free — ★ so choose boundaries for
      CLARITY, not for the transaction model.

 ★ ⑥ IDEMPOTENCY (52)
    ⇒ ★ every externally-triggered write path needs a key,
      ★ enforced by UNIQUE. ★ Adding it later means auditing
      every handler.
```

### The default schema conventions, decided once

```
 ★ DECIDE THESE ONCE, WRITE THEM DOWN, AND STOP RE-DEBATING THEM.

 ★ NAMING
   tables: ★ plural, snake_case            orders, order_items
   columns: ★ snake_case                    created_at
   ★ FKs: <singular>_id                     customer_id
   ★ booleans: is_/has_                     is_active
   ★ timestamps: _at                        shipped_at
   ★ money: _minor                          total_minor
   ★ indexes: idx_<table>_<cols>[_<pred>]

 ★ EVERY TABLE GETS
   ✓ ★ a primary key (★ never composite unless partitioned)
   ✓ ★ created_at timestamptz NOT NULL DEFAULT now()
   ✓ ★ updated_at, maintained by a trigger
   ✓ ★ tenant_id, if multi-tenant — ★ leading every index
   ✓ ★ NOT NULL by default; ★ nullable is the exception you justify

 ★ NEVER
   ✗ ★ float for money · ✗ `timestamp` without a zone
   ✗ ★ VARCHAR(n) as validation (★ use CHECK; ★ text is free)
   ✗ ★ soft deletes by default (27) — ★ they poison every query
   ✗ ★ EAV (27) — ★ jsonb instead (70)
   ✗ ★ polymorphic FKs (27) — ★ they cannot be enforced
   ✗ ★ storing an average (56, 73) — ★ store sum and count
```

---

## How it works — step by step

### The worked problem

```
 ★ THE BRIEF, AS RECEIVED:

 "We're building a B2B wholesale marketplace for India. Buyers
  place orders with sellers. Sellers manage catalogue and stock.
  We need payments, invoicing with GST, shipment tracking, and
  reporting. Expect 5,000 sellers, 80,000 buyers, growing fast.
  Launch in four months."

 ★ THAT IS THE NAPKIN. ★ Now the interrogation.
```

### Step 1 — the interrogation, with the answers that changed the design

```
 ★ Q: "One buyer per order, or can an order span sellers?"
 ★ A: ★ "One seller per order. A basket across sellers splits
      into multiple orders at checkout."
 ⇒ ★ CHANGES EVERYTHING: orders have a seller_id, and there is
   a `baskets` concept that is NOT an order.

 ★ Q: "Is the price on an order the product's current price?"
 ★ A: ★ "No — the price agreed at order time. Sellers change
      prices weekly, and disputes reference the agreed price."
 ⇒ ★ SNAPSHOT (54 gate 3). ★ unit_price_charged_minor, frozen.
   ★ Not denormalisation. ★ No sync obligation.

 ★ Q: "What must never be true?"
 ★ A: ★ "Stock can't go negative. An invoice total must match its
      lines. GST must be computed at order time and never change.
      A buyer must never see another buyer's orders."
 ⇒ ★ FOUR INVARIANTS ⇒ ★ CHECK, a maintained total, a snapshot,
   ★ and RLS.

 ★ Q: "How long do you keep orders? Shipment events?"
 ★ A: ★ "Orders: 7 years (statutory). Shipment scan events:
      18 months."
 ⇒ ★ TWO DIFFERENT RETENTION POLICIES ⇒ ★ scan_events is
   partitioned from day one (59, 73).

 ★ Q: "In three years — orders per day? Scan events per order?"
 ★ A: ★ "Maybe 40,000 orders/day. Each shipment gets ~20 scans."
 ⇒ ★ orders: 40k/day × 365 × 3 = ★ 44M rows ⇒ ★ fine unpartitioned
   ⇒ ★ scan_events: 40k × 20 × 365 = ★ 292M/year ⇒ ★ MUST partition

 ★ Q: "What is the busiest single row?"
 ★ A: ★ "A few large sellers do 30% of volume."
 ⇒ ★ any per-seller counter is a ★ HOT ROW candidate (61).
   ⇒ ★ measure before adding one.

 ★ Q: "What happens if a payment webhook arrives twice?"
 ★ A: ★ "...it shouldn't."
 ⇒ ★ IT WILL (52). ★ Idempotency keys on every external write path.

 ★ Q: "Can a buyer belong to more than one organisation?"
 ★ A: ★ "Yes — a purchasing agent may buy for several companies."
 ⇒ ★ users ≠ organisations. ★ A membership table. ★ AND the
   tenancy boundary is the ORGANISATION, not the user.
   ⇒ ★ THIS ANSWER ALONE PREVENTED A REWRITE.
```

### Step 2 — the access patterns, with volumes

```
 ★ READS
 R1  order by id, with items and seller       ★ 12,000/min · 50 ms
 R2  a buyer org's orders, newest first        ★ 8,000/min · 100 ms
 R3  a seller's orders, filtered by status     ★ 4,000/min · 100 ms
 R4  order by external reference (tracking)    ★ 40,000/min · 20 ms
 R5  catalogue search within a seller          ★ 20,000/min · 200 ms
 R6  stock for a set of SKUs                   ★ 30,000/min · 20 ms
 R7  a seller's revenue for a month            ★ 40/min · 2 s
 R8  shipment scans for an order               ★ 6,000/min · 100 ms
 R9  invoices for an org, by financial year    ★ 200/min · 500 ms

 ★ WRITES
 W1  place an order (order + items + stock)    ★ 30/sec peak
 W2  record a payment (idempotent)             ★ 20/sec
 W3  ingest a shipment scan                    ★ 200/sec
 W4  seller updates a price                    ★ 5/sec
 W5  seller updates stock                      ★ 50/sec

 ⇒ ★ NOTE R4: ★ 40,000/min for a lookup by an EXTERNAL string.
   ★ That is a UNIQUE index, and it is the highest-volume read
   in the system. ★ It would have been easy to miss.
```

### Step 5 — the one-way doors, decided and recorded

```markdown
## One-way doors   (decided 2026-08-27, reviewed by: platform, product)

★ TENANCY: shared tables + `org_id` + ★ FORCED RLS
  ⇒ ★ the boundary is the ORGANISATION, not the user (a user may
    belong to several).
  ⇒ ★ `org_id` LEADS every index on a tenant-scoped table (69).
  ⇒ ★ REJECTED schema-per-tenant: ★ 5,000 sellers × 14 tables =
    70,000 objects; ★ migrations become a project.

★ PRIMARY KEYS: `bigserial` internally; ★ UUIDv7 for anything
  exposed in a URL or generated by a client.
  ⇒ ★ REJECTED UUIDv4: ★ random insert points, poor B-tree
    locality (22).

★ MONEY: `bigint`, ★ paise. ★ GST rates as `numeric(5,4)`.
  ⇒ ★ money is bigint; ★ a RATE is not money and needs exactness
    at 4 decimal places.

★ TIME: ★ `timestamptz` everywhere. ★ Application timezone is IST
  for display only.

★ PARTITIONING: `scan_events` ★ by month from day one (292M/year,
  18-month retention). ★ `orders` unpartitioned (44M in 3 years,
  7-year retention ⇒ ★ revisit at 200M).

★ IDEMPOTENCY: ★ every external write path (payments, webhooks,
  order placement) carries a key with a ★ UNIQUE constraint (52).

★ AGGREGATE BOUNDARY: ★ an order + its items + the stock
  decrement are ONE transaction. ★ The invoice and the outbox
  event are in the SAME transaction. ★ Everything else is async.
```

### Steps 6–9 — the schema, decision by decision

```sql
-- ★ ═══ IDENTITY AND TENANCY ═══
CREATE TABLE organisations (
  id          bigserial   PRIMARY KEY,
  ★ public_id uuid        NOT NULL UNIQUE DEFAULT uuidv7(),  -- ★ external
  legal_name  text        NOT NULL,
  ★ gstin     char(15)    UNIQUE
              ★ CHECK (gstin ~ '^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}Z[0-9A-Z]{1}$'),
  kind        text        NOT NULL CHECK (kind IN ('buyer','seller','both')),
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now()
);
-- ★ the GSTIN CHECK is an INVARIANT, not validation-in-code.
--   ★ It is 15 chars with a known structure; the database can
--   enforce what the application might forget.

CREATE TABLE users (
  id         bigserial   PRIMARY KEY,
  public_id  uuid        NOT NULL UNIQUE DEFAULT uuidv7(),
  email      citext      NOT NULL UNIQUE,       -- ★ citext: case-insensitive
  created_at timestamptz NOT NULL DEFAULT now()
);

-- ★ THE ANSWER TO "can a user belong to several orgs?"
CREATE TABLE memberships (
  user_id  bigint NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  org_id   bigint NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
  role     text   NOT NULL CHECK (role IN ('owner','buyer','viewer')),
  created_at timestamptz NOT NULL DEFAULT now(),
  ★ PRIMARY KEY (user_id, org_id)
);
CREATE INDEX ON memberships (org_id, user_id);   -- ★ the other direction
```

```sql
-- ★ ═══ CATALOGUE ═══
CREATE TABLE products (
  id          bigserial   PRIMARY KEY,
  ★ org_id    bigint      NOT NULL REFERENCES organisations(id),  -- the seller
  sku         text        NOT NULL,
  name        text        NOT NULL,
  ★ list_price_minor bigint NOT NULL CHECK (list_price_minor >= 0),
  ★ gst_rate  numeric(5,4) NOT NULL CHECK (gst_rate >= 0 AND gst_rate <= 1),
  status      text        NOT NULL DEFAULT 'active'
                          CHECK (status IN ('active','discontinued')),
  ★ attrs     jsonb       NOT NULL DEFAULT '{}',    -- ★ variable (70)
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now(),
  ★ UNIQUE (org_id, sku)         -- ★ SKU unique PER SELLER, not globally
);
-- ★ R5: catalogue search within a seller
ALTER TABLE products ADD COLUMN search tsvector
  ★ GENERATED ALWAYS AS (
      setweight(to_tsvector('simple',  coalesce(sku,'')),  'A') ||
      setweight(to_tsvector('english', coalesce(name,'')), 'A')) STORED;
CREATE INDEX ON products USING gin (search);
CREATE INDEX ON products (★ org_id, status, name);   -- ★ org_id LEADS

-- ★ STOCK IS A SEPARATE, NARROW TABLE — ★ it is updated 50/sec
--   and `products` has 4 indexes (46, 54).
CREATE TABLE stock (
  product_id bigint PRIMARY KEY REFERENCES products(id),
  ★ on_hand  integer NOT NULL ★ CHECK (on_hand >= 0),   -- ★ INVARIANT
  reserved   integer NOT NULL DEFAULT 0 CHECK (reserved >= 0),
  updated_at timestamptz NOT NULL DEFAULT now()
) ★ WITH (fillfactor = 70);          -- ★ HOT updates (46)
```

```sql
-- ★ ═══ ORDERS ═══
CREATE TABLE orders (
  id            bigserial   PRIMARY KEY,
  ★ public_id   uuid        NOT NULL UNIQUE DEFAULT uuidv7(),
  ★ buyer_org_id  bigint    NOT NULL REFERENCES organisations(id),
  ★ seller_org_id bigint    NOT NULL REFERENCES organisations(id),
  placed_by_user_id bigint  NOT NULL REFERENCES users(id),

  status        text        NOT NULL DEFAULT 'pending'
                CHECK (status IN ('pending','confirmed','dispatched',
                                  'delivered','cancelled')),
  -- ★ MAINTAINED TOTALS: sum and tax, ★ never an average (56)
  ★ subtotal_minor bigint   NOT NULL DEFAULT 0 CHECK (subtotal_minor >= 0),
  ★ gst_minor      bigint   NOT NULL DEFAULT 0 CHECK (gst_minor >= 0),
  ★ total_minor    bigint   ★ GENERATED ALWAYS AS
                              (subtotal_minor + gst_minor) STORED,

  -- ★ SNAPSHOTS — frozen at order time, ★ no sync obligation (54)
  ★ buyer_gstin_at_order  char(15),
  ★ ship_to_address       jsonb NOT NULL,

  placed_at     timestamptz NOT NULL DEFAULT now(),
  confirmed_at  timestamptz,
  dispatched_at timestamptz,
  delivered_at  timestamptz,

  -- ★ INVARIANT: the lifecycle cannot go backwards
  ★ CHECK (confirmed_at  IS NULL OR confirmed_at  >= placed_at),
  ★ CHECK (dispatched_at IS NULL OR dispatched_at >= confirmed_at),
  ★ CHECK (delivered_at  IS NULL OR delivered_at  >= dispatched_at),
  ★ CHECK (status <> 'delivered' OR delivered_at IS NOT NULL),
  ★ CHECK (buyer_org_id <> seller_org_id)      -- ★ can't sell to yourself
);

-- ★ R2: a buyer org's orders, newest first
CREATE INDEX ON orders (★ buyer_org_id, placed_at DESC);
-- ★ R3: a seller's orders by status — ★ PARTIAL, because 94% are
--   terminal states nobody filters on
CREATE INDEX ON orders (★ seller_org_id, placed_at DESC)
  ★ WHERE status IN ('pending','confirmed','dispatched');
```

```sql
CREATE TABLE order_items (
  id          bigserial PRIMARY KEY,
  ★ order_id  bigint    NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id  bigint    NOT NULL REFERENCES products(id),
  -- ★ SNAPSHOTS: what was agreed, ★ not what the product says now
  ★ sku_at_order        text   NOT NULL,
  ★ name_at_order       text   NOT NULL,
  ★ unit_price_charged_minor bigint NOT NULL CHECK (unit_price_charged_minor >= 0),
  ★ gst_rate_charged    numeric(5,4) NOT NULL,
  qty         integer   NOT NULL ★ CHECK (qty > 0),
  ★ line_total_minor bigint ★ GENERATED ALWAYS AS
                              (qty * unit_price_charged_minor) STORED,
  ★ UNIQUE (order_id, product_id)     -- ★ one line per product
);
CREATE INDEX ON order_items (order_id, id);   -- ★ R1
```
```
 ★ NOTE WHAT THE SNAPSHOTS BUY:
   ★ a seller can rename a product, change its price or
     discontinue it, ★ and every historical order is unaffected.
   ★ NO TRIGGER. NO RECONCILER. NO OBLIGATION.
   ⇒ ★ this is Topic 54's gate 3 doing real work.
```

```sql
-- ★ ═══ PAYMENTS AND IDEMPOTENCY ═══
CREATE TABLE payments (
  id           bigserial   PRIMARY KEY,
  order_id     bigint      NOT NULL REFERENCES orders(id),
  ★ idempotency_key text   NOT NULL ★ UNIQUE,       -- ★ (52)
  amount_minor bigint      NOT NULL CHECK (amount_minor > 0),
  ★ gateway_ref text       NOT NULL,
  status       text        NOT NULL
               CHECK (status IN ('pending','captured','failed','refunded')),
  created_at   timestamptz NOT NULL DEFAULT now(),
  ★ UNIQUE (gateway_ref)     -- ★ the provider's id is also unique
);
```

```sql
-- ★ ═══ SHIPMENT SCANS — ★ partitioned from day one (59, 73) ═══
CREATE TABLE scan_events (
  order_id   bigint      NOT NULL,
  scanned_at timestamptz NOT NULL,
  ★ event_id uuid        NOT NULL DEFAULT uuidv7(),
  hub_code   text        NOT NULL,
  status     text        NOT NULL,
  ★ PRIMARY KEY (order_id, scanned_at, event_id)   -- ★ key includes
) ★ PARTITION BY RANGE (scanned_at);               --   the partition col
-- ★ R8: scans for an order — served by the PK's leading column
-- ★ no FK to orders: ★ a partitioned table's FK would need
--   scanned_at in the parent. ★ Enforced by the ingest path,
--   and reconciled nightly. ★ DOCUMENTED AS A DELIBERATE CHOICE.

CREATE TABLE scan_events_default PARTITION OF scan_events DEFAULT;  -- ★ seatbelt
```

```sql
-- ★ ═══ THE OUTBOX (52) ═══
CREATE TABLE outbox (
  id              bigserial   PRIMARY KEY,
  aggregate_type  text        NOT NULL,
  aggregate_id    bigint      NOT NULL,
  event_type      text        NOT NULL,
  payload         jsonb       NOT NULL,
  ★ idempotency_key text      NOT NULL UNIQUE,
  created_at      timestamptz NOT NULL DEFAULT now(),
  published_at    timestamptz
);
CREATE INDEX ON outbox (id) ★ WHERE published_at IS NULL;   -- ★ partial
★ ALTER TABLE outbox SET (autovacuum_vacuum_scale_factor = 0.01,
                          autovacuum_vacuum_cost_delay = 0);  -- ★ a queue (47)
```

```sql
-- ★ ═══ ROW-LEVEL SECURITY (69) ═══
DO $$ DECLARE t text;
BEGIN
  FOREACH t IN ARRAY ARRAY['products','stock','orders','order_items','payments']
  LOOP
    EXECUTE format('ALTER TABLE %I ENABLE ROW LEVEL SECURITY', t);
    EXECUTE format('ALTER TABLE %I ★ FORCE ROW LEVEL SECURITY', t);
  END LOOP;
END $$;

-- ★ orders are visible to BOTH parties — ★ the policy encodes that
CREATE POLICY org_orders ON orders FOR SELECT
  USING (★ buyer_org_id  = current_setting('app.org_id', true)::bigint
      OR ★ seller_org_id = current_setting('app.org_id', true)::bigint);
CREATE POLICY org_orders_ins ON orders FOR INSERT
  ★ WITH CHECK (buyer_org_id = current_setting('app.org_id', true)::bigint);
-- ★ current_setting(…, true) ⇒ NULL when unset ⇒ ★ FAILS CLOSED
```

### The write path — one transaction, correctly

```js
// ★ W1: place an order. ★ The aggregate boundary, exactly.
async function placeOrder(ctx, { idempotencyKey, sellerOrgId, lines }) {
  return withTenant(ctx.orgId, ctx.userId, async (tx) => {
    // ★ ① claim the idempotency key FIRST (52)
    const claim = await tx.query(
      `INSERT INTO order_idempotency (key, buyer_org_id)
       VALUES ($1,$2) ON CONFLICT (key) DO NOTHING RETURNING key`,
      [idempotencyKey, ctx.orgId]);
    if (claim.rowCount === 0) return replayStoredResponse(idempotencyKey);

    // ★ ② decrement stock ATOMICALLY — ★ no read-then-write (43, 49)
    //    ★ sorted by product_id: ★ deterministic lock order (48)
    for (const l of [...lines].sort((a,b) => a.product_id - b.product_id)) {
      const { rowCount } = await tx.query(
        `UPDATE stock SET on_hand = on_hand - $1, updated_at = now()
          WHERE product_id = $2 AND ★ on_hand >= $1`,
        [l.qty, l.product_id]);
      if (rowCount === 0) throw new AppError('INSUFFICIENT_STOCK', l);
    }

    // ★ ③ the order, with SNAPSHOTS taken now
    const { rows: [o] } = await tx.query(
      `INSERT INTO orders (buyer_org_id, seller_org_id, placed_by_user_id,
                           buyer_gstin_at_order, ship_to_address)
       SELECT $1, $2, $3, org.gstin, $4
         FROM organisations org WHERE org.id = $1
       RETURNING id, public_id`,
      [ctx.orgId, sellerOrgId, ctx.userId, JSON.stringify(ctx.shipTo)]);

    // ★ ④ items, with the price and GST rate FROZEN
    await tx.query(
      `INSERT INTO order_items (order_id, product_id, sku_at_order,
              name_at_order, unit_price_charged_minor, gst_rate_charged, qty)
       SELECT $1, p.id, p.sku, p.name, p.list_price_minor, p.gst_rate, v.qty
         FROM unnest($2::bigint[], $3::int[]) AS v(pid, qty)
         JOIN products p ON p.id = v.pid`,
      [o.id, lines.map(l => l.product_id), lines.map(l => l.qty)]);

    // ★ ⑤ totals, computed from the items just inserted
    await tx.query(
      `UPDATE orders o SET
         subtotal_minor = t.sub,
         gst_minor = t.gst
       FROM (SELECT sum(line_total_minor) AS sub,
                    sum(round(line_total_minor * gst_rate_charged)) AS gst
               FROM order_items WHERE order_id = $1) t
       WHERE o.id = $1`, [o.id]);

    // ★ ⑥ the outbox event — ★ same transaction (52)
    await tx.query(
      `INSERT INTO outbox (aggregate_type, aggregate_id, event_type,
                           payload, idempotency_key)
       VALUES ('order', $1, 'OrderPlaced', $2, $3)`,
      [o.id, { order_id: o.id }, `order-placed:${o.id}`]);

    return { order_id: o.public_id };
  });
}
// ★ NOTE WHAT IS **NOT** IN HERE (45, 52, 65):
//   ✗ no HTTP call · ✗ no payment gateway · ✗ no email
//   ⇒ ★ all of those are driven by the outbox event, OUTSIDE
//     the transaction.
```

### Step 10 — the design record

```markdown
## Schema design record — wholesale marketplace   (2026-08-27)

★ ACCESS PATTERNS → INDEXES
| # | Pattern | Index | Verified |
|---|---|---|---|
| R1 | order + items | `orders_pkey`, `order_items(order_id,id)` | ✓ 0.1 ms |
| R2 | buyer's orders | `orders(buyer_org_id, placed_at DESC)` | ✓ 0.3 ms |
| R3 | seller's open orders | ★ partial on status | ✓ 0.4 ms |
| R4 | by tracking number | `UNIQUE(gateway_ref)` | ✓ 0.08 ms |
| R5 | catalogue search | ★ `gin(search)` + `(org_id,status,name)` | ✓ 2.1 ms |
| R6 | stock for SKUs | `stock_pkey` (`= ANY`) | ✓ 0.2 ms |
| R7 | seller revenue | ★ **rollup table** (56) | ✓ 40 ms |
| R8 | order scans | `scan_events` PK | ✓ 0.9 ms |
| R9 | invoices by FY | `invoices(org_id, issued_at)` | ✓ 12 ms |

★ INVARIANTS → ENFORCEMENT
| Invariant | Mechanism |
|---|---|
| stock ≥ 0 | ★ `CHECK` + atomic `UPDATE … WHERE on_hand >= $1` |
| total = subtotal + gst | ★ `GENERATED` column — cannot drift |
| lifecycle ordering | ★ 4 `CHECK` constraints |
| SKU unique per seller | ★ `UNIQUE (org_id, sku)` |
| org isolation | ★ **FORCED RLS**, fails closed |
| no duplicate payments | ★ `UNIQUE (idempotency_key)`, `UNIQUE (gateway_ref)` |
| GSTIN format | ★ `CHECK` with a regex |

★ DELIBERATE OMISSIONS
- ★ no FK from `scan_events` to `orders` — the partition key would
  have to appear in the parent. ★ Enforced at ingest, reconciled
  nightly. ★ Reviewed and accepted.
- ★ no per-seller order counter — ★ 30% of volume is a few
  sellers ⇒ hot row (61). ★ Computed from the rollup instead.
- ★ `orders` not partitioned — 44M rows in 3 years.
  ★ REVISIT AT 200M or if retention changes.

★ ONE-WAY DOORS: (the table above)
★ SIGNED: platform lead, product owner.
```

---

## Concept breakdown

```
★ TEN STEPS — ★ THE FIRST FIVE PRODUCE NO SQL
   ① interrogate  ② ★ access patterns  ③ invariants
   ④ hot paths  ⑤ ★ one-way doors  ⑥ model  ⑦ normalise then bend
   ⑧ index from patterns  ⑨ constrain invariants  ⑩ ★ write it down
   ⇒ ★ starting at ⑥ produces a schema that cannot answer its
     questions

★ THE EIGHT INTERROGATION QUESTIONS
   one-or-many · always-or-sometimes · ★ can it change (and do we
   need the old value) · ★ volume in 3 years · ★ what must never
   be true · ★ who may see it · ★ how long do we keep it ·
   ★ what if it fails halfway

★ ACCESS PATTERNS ARE THE SPECIFICATION
   ★ sentences with VOLUMES and LATENCY TARGETS
   ⇒ ★ each becomes an index, a denormalisation (54) or a
     rollup (56)
   ⇒ ★ COMPLETENESS TEST: can you name the index for every
     sentence?

★ INVARIANTS BECOME CONSTRAINTS, CLASSIFIED
   per-row ⇒ CHECK · unique ⇒ UNIQUE · overlap ⇒ EXCLUDE ·
   referential ⇒ FK · ★ across rows ⇒ SERIALIZABLE or a counter

★ ONE-WAY DOORS — ★ where the thinking goes
   ★ tenancy (most expensive) · ★ PK type · ★ money and time ·
   ★ partitioning · aggregate boundaries · ★ idempotency
   ⇒ ★ two-way doors get defaults, decided once, not re-debated

★ THE SNAPSHOT TEST DOES REAL WORK HERE
   `unit_price_charged`, `sku_at_order`, `buyer_gstin_at_order`,
   `ship_to_address`
   ⇒ ★ frozen by design ⇒ ★ NO trigger, NO reconciler, NO
     obligation (54 gate 3)

★ THE WRITE PATH ENCODES SEVERAL PHASES AT ONCE
   ★ idempotency key first (52) · ★ atomic stock decrement (43,49)
   · ★ deterministic lock order (48) · ★ snapshots taken now (54)
   · ★ outbox in the same transaction (52)
   · ★ NO external calls inside (45, 65)

★ THE DESIGN RECORD IS THE DELIVERABLE
   patterns→indexes · invariants→mechanisms · ★ deliberate
   omissions · ★ one-way doors · ★ signed
```

---

## Diagrams

**Diagram 1 — big picture: the ten steps and what each produces**

```
  ★ NO SQL IS WRITTEN UNTIL STEP ⑥.

  ① ★ INTERROGATE ──────────► ★ 8 questions, answered by the
                               business, ★ that change the design
                               ("a user may belong to several orgs"
                                ⇒ ★ a membership table + org-level
                                  tenancy)
  ② ★ ACCESS PATTERNS ──────► ★ 9 read + 5 write sentences,
                               ★ with volumes and latency targets
  ③ ★ INVARIANTS ───────────► ★ 7 "must never be true" statements
  ④ ★ HOT PATHS ────────────► ★ 40k orders/day · 292M scans/year
                               · ★ 30% of volume from a few sellers
  ⑤ ★ ONE-WAY DOORS ────────► ★ tenancy · PK type · money/time ·
                               partitioning · boundaries ·
                               idempotency  ★ — SIGNED
  ─────────────────── ★ ONLY NOW: SQL ───────────────────
  ⑥ MODEL ──────────────────► entities, relationships, FKs
  ⑦ NORMALISE, THEN BEND ───► ★ 3NF, then snapshots (not copies)
                               and ★ one rollup
  ⑧ INDEX ──────────────────► ★ one index per access pattern,
                               ★ verified with EXPLAIN
  ⑨ CONSTRAIN ──────────────► ★ each invariant → a mechanism
  ⑩ ★ RECORD ───────────────► ★ the design record: patterns→
                               indexes, invariants→mechanisms,
                               ★ deliberate omissions, ★ signed

 ★ THE FAILURE MODE OF SKIPPING ①–⑤:
   ★ a beautiful ERD that cannot serve R4 (40,000/min) because
   nobody wrote R4 down.
```

**Diagram 2 — data flow: one-way vs two-way doors**

```
  ★ TWO-WAY DOORS — ★ decide fast, change later, ★ don't debate
  ┌───────────────────────────────────────────────────────────────┐
  │ column names          ★ minutes to change                      │
  │ adding a column       ★ minutes (with a default: instant)      │
  │ adding an index       ★ minutes (CONCURRENTLY)                 │
  │ CHECK constraints     ★ minutes (NOT VALID, then VALIDATE)     │
  │ a new table           ★ minutes                                │
  │ ★ ⇒ TAKE THE DEFAULT. Write it down once. Move on.            │
  └───────────────────────────────────────────────────────────────┘

  ★ ONE-WAY DOORS — ★ this is where the thinking goes
  ┌───────────────────────────────────────────────────────────────┐
  │ ★ TENANCY MODEL       ★ every table, query, index (69)         │
  │                       ★ ⇒ months                               │
  │ ★ PRIMARY KEY TYPE    ★ every FK, every URL, every index (22)  │
  │                       ★ ⇒ quarters                             │
  │ ★ PARTITION KEY       ★ dual-write + backfill + shadow (72,62) │
  │                       ★ ⇒ quarters                             │
  │ ★ MONEY AS FLOAT      ★ every stored value is already wrong    │
  │                       ★ ⇒ unrecoverable                        │
  │ ★ `timestamp` no zone ★ every stored value ambiguous           │
  │                       ★ ⇒ unrecoverable                        │
  │ ★ NO IDEMPOTENCY KEY  ★ audit every handler (52)               │
  │                       ★ ⇒ weeks, and duplicates already exist  │
  │ ★ NO PARTITIONING     ★ retrofit to 4 TB (59)                  │
  │                       ★ ⇒ a project                            │
  └───────────────────────────────────────────────────────────────┘

 ★ THE RULE: ★ if reversing it costs more than a sprint, it is a
   one-way door and it goes in the signed record.
```

**Diagram 3 — before/after: the interrogation that saved a rewrite**

```
 ✗ WITHOUT THE INTERROGATION — what the napkin implies
 ┌───────────────────────────────────────────────────────────────┐
 │ users(id, email, ★ company_name, gstin)                        │
 │ orders(id, ★ user_id, seller_id, total, created_at)            │
 │ order_items(id, order_id, ★ product_id, qty)                   │
 │                                                                │
 │ ★ THE FOUR BUGS ALREADY PRESENT:                               │
 │ ① ★ tenancy is the USER ⇒ ★ a purchasing agent buying for      │
 │    three companies is impossible                               │
 │ ② ★ no price on the line ⇒ ★ the order total changes when the  │
 │    seller changes the price. ★ Disputes become unanswerable.   │
 │ ③ ★ `total` is a float ⇒ ★ ₹0.01 errors, compounding           │
 │ ④ ★ no retention thinking ⇒ ★ scan_events unpartitioned ⇒      │
 │    292M rows/year and a DELETE-based retention job (59)        │
 │                                                                │
 │ ★ ALL FOUR ARE ONE-WAY DOORS. ★ All four are found in month 8. │
 └───────────────────────────────────────────────────────────────┘

 ✓ AFTER EIGHT QUESTIONS
 ┌───────────────────────────────────────────────────────────────┐
 │ ★ Q: "can a user belong to several organisations?"             │
 │   A: "yes" ⇒ ★ users ≠ orgs; a membership table;               │
 │              ★ TENANCY IS THE ORG                              │
 │ ★ Q: "is the order price the current price?"                   │
 │   A: "no, the agreed price" ⇒ ★ unit_price_charged_minor,      │
 │              ★ a SNAPSHOT with no sync obligation (54)          │
 │ ★ Q: "money?" ⇒ ★ bigint paise. ★ GST rate numeric(5,4).       │
 │ ★ Q: "how long do you keep scan events?"                       │
 │   A: "18 months" ⇒ ★ PARTITIONED FROM DAY ONE (59)             │
 │                                                                │
 │ ★ FOUR QUESTIONS. ★ FOUR ONE-WAY DOORS, OPENED CORRECTLY.      │
 │ ★ Cost: ★ 30 minutes of conversation.                          │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**A small design, end to end: a URL shortener.**

```
 ★ THE BRIEF: "shorten URLs, track clicks, show stats."

 ★ STEP ① — INTERROGATE
   Q: "who owns a link — a user, or is it anonymous?"
   A: ★ "both. Anonymous links exist; logged-in users own theirs."
   ⇒ ★ nullable owner, ★ and RLS cannot be the only isolation.

   Q: "can a short code be reused after deletion?"
   A: ★ "no — never. Old links must not resolve to new targets."
   ⇒ ★ soft delete? ★ NO (27) — ★ a `status` column, and the
     UNIQUE stays.

   Q: "volume in three years?"
   A: ★ "10M links, ★ 2 billion clicks."
   ⇒ ★ clicks MUST be partitioned. ★ Links need not be.

   Q: "how long do you keep clicks?"
   A: ★ "raw for 90 days; ★ daily totals forever."
   ⇒ ★ tiered retention (73).

   Q: "what must never be true?"
   A: ★ "two links with the same code. A click counted twice."
   ⇒ ★ UNIQUE, ★ and idempotency on ingest.

 ★ STEP ② — ACCESS PATTERNS
   R1 ★ resolve a code → target   ★ 50,000/sec · ★ p99 < 5 ms
   R2 a user's links               ★ 100/min · 200 ms
   R3 daily clicks for a link      ★ 500/min · 500 ms
   R4 top links for a user         ★ 50/min · 1 s
   W1 create a link                ★ 200/sec
   W2 ★ record a click             ★ 50,000/sec

 ⇒ ★ R1 AND W2 DOMINATE BY THREE ORDERS OF MAGNITUDE.
   ★ THE ENTIRE DESIGN SERVES THOSE TWO.
```

```sql
-- ★ STEP ⑤ — ONE-WAY DOORS
--   ★ code is the natural key and is externally visible ⇒ it IS
--     the lookup key. ★ No surrogate needed for R1.
--   ★ clicks partitioned by day from day one.
--   ★ daily rollup, kept forever; raw clicks 90 days.

CREATE TABLE links (
  ★ code       text        PRIMARY KEY
               ★ CHECK (code ~ '^[a-zA-Z0-9]{6,12}$'),
  target_url   text        NOT NULL CHECK (length(target_url) <= 2048),
  owner_id     bigint      REFERENCES users(id),      -- ★ nullable
  status       text        NOT NULL DEFAULT 'active'
               CHECK (status IN ('active','disabled')),
  created_at   timestamptz NOT NULL DEFAULT now()
) ★ WITH (fillfactor = 100);      -- ★ append-only, never updated

-- ★ R1 is a PK lookup: ★ 0.02 ms. Nothing more is needed.
-- ★ R2
CREATE INDEX ON links (owner_id, created_at DESC) WHERE owner_id IS NOT NULL;
```
```sql
-- ★ W2: 50,000 clicks/sec. ★ APPEND-ONLY, PARTITIONED (73)
CREATE TABLE clicks (
  code        text        NOT NULL,
  clicked_at  timestamptz NOT NULL,
  ★ click_id  uuid        NOT NULL DEFAULT uuidv7(),
  country     char(2),
  referrer_host text
) ★ PARTITION BY RANGE (clicked_at);
-- ★ NO PRIMARY KEY, NO INDEX on the fact table.
--   ★ Nothing looks up a click by id. ★ R3 reads the ROLLUP.
--   ⇒ ★ this is Topic 73's finding: time-series needs ~one index,
--     and here it needs ★ zero.

-- ★ the rollup, kept forever
CREATE TABLE clicks_daily (
  code  text NOT NULL,
  day   date NOT NULL,
  ★ n   bigint NOT NULL,
  PRIMARY KEY (code, day)
);
```
```sql
-- ★ R3 and R4 read the rollup
SELECT day, n FROM clicks_daily
 WHERE code = $1 AND day >= $2 ORDER BY day;          -- ★ 0.3 ms

SELECT code, sum(n) AS total FROM clicks_daily
 WHERE code IN (SELECT code FROM links WHERE owner_id = $1)
   AND day >= current_date - 30
 GROUP BY 1 ORDER BY 2 DESC LIMIT 20;                 -- ★ 12 ms
```
```
 ★ THE WHOLE DESIGN IN ONE SENTENCE:
   ★ R1 is a primary-key lookup, ★ W2 is an unindexed append to
   a partition, ★ and everything else reads a rollup.
 ⇒ ★ THAT IS WHAT "DESIGN FROM ACCESS PATTERNS" PRODUCES.
```

**Prove each pattern is served.**
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT target_url FROM links WHERE code = 'aB3xY9';
```
```
 Index Scan using links_pkey on links  (actual time=0.018..0.019 rows=1)
   Buffers: shared hit=3
 Execution Time: ★ 0.031 ms
```
```sql
EXPLAIN (ANALYZE) INSERT INTO clicks (code, clicked_at, country)
VALUES ('aB3xY9', now(), 'IN');
```
```
 Insert on clicks  (actual time=0.041..0.041 rows=0)
 Execution Time: ★ 0.058 ms        — ★ no index to maintain
```

**And prove the omissions were deliberate.**
```sql
-- ★ "why no index on clicks.code?"
EXPLAIN (ANALYZE) SELECT count(*) FROM clicks
 WHERE code = 'aB3xY9' AND clicked_at >= current_date;
```
```
 ->  Seq Scan on clicks_2026_08_27  (actual rows=412)
 Execution Time: ★ 84 ms
 ⇒ ★ AND THAT QUERY IS NEVER RUN. ★ R3 reads clicks_daily.
   ★ An index here would cost 50,000 writes/sec to serve a query
     nobody makes. ★ DOCUMENTED AS A DELIBERATE OMISSION.
```

---

## Example 2 — production scenario

**The situation.** The wholesale marketplace from "How it works" is now being designed for real, with four months to launch. This is the review that happened at week two — **before any code was written.**

**Step 1 — the access-pattern audit found a gap.**

```
 ★ THE REVIEWER'S QUESTION: ★ "point at the index for every
   pattern."

   R1 order + items          ⇒ ★ orders_pkey, order_items(order_id,id) ✓
   R2 buyer's orders         ⇒ ★ (buyer_org_id, placed_at DESC) ✓
   R3 seller's open orders   ⇒ ★ partial index ✓
   R4 by tracking number     ⇒ ★ ...
   ★ SILENCE.

 ⇒ ★ R4 IS 40,000/min — ★ THE HIGHEST-VOLUME READ IN THE SYSTEM —
   ★ and the tracking number was going to live in
   `orders.metadata->>'tracking'` with a GIN index (70).
 ⇒ ★ GIN CANNOT SERVE AN EQUALITY LOOKUP AS WELL AS A B-TREE,
   and the field is not optional.
```
```sql
-- ★ THE FIX: promote it (70)
ALTER TABLE orders ADD COLUMN ★ tracking_number text ★ UNIQUE;
CREATE INDEX ON orders (tracking_number) WHERE tracking_number IS NOT NULL;
```
```
 ★ MEASURED: ★ GIN containment 4.2 ms → ★ B-tree unique 0.08 ms.
   ★ 52×, on the highest-volume query in the system.
 ⇒ ★ FOUND BY WALKING THE LIST, ★ not by profiling — ★ because
   there was no code yet.
```

**Step 2 — an invariant with no mechanism.**

```
 ★ THE INVARIANT: ★ "an invoice total must match the sum of its
   lines."
 ★ THE PROPOSED MECHANISM: ★ "the application computes it."
 ⇒ ★ REJECTED. ★ Three services will write invoices within a year.
```
```sql
-- ★ OPTION A: a GENERATED column ⇒ ★ impossible; it spans rows.
-- ★ OPTION B: a maintained column + a trigger (55)
--   ⇒ ★ correct, but a hot row for large sellers? ★ MEASURE:
--     ★ invoices have ~8 lines, written once. ★ NOT hot. ✓
-- ★ OPTION C: computed on read
--   ⇒ ★ R9 is 200/min over ~8 rows. ★ 0.4 ms. ✓

-- ★ CHOSEN: ★ C, computed on read, ★ PLUS a nightly reconciler
--   against a stored total used for the PDF (which must be
--   immutable once issued).
CREATE TABLE invoices (
  id bigserial PRIMARY KEY,
  org_id bigint NOT NULL,
  ★ total_minor bigint NOT NULL,     -- ★ FROZEN at issue (a snapshot)
  issued_at timestamptz NOT NULL DEFAULT now(),
  ★ CHECK (total_minor >= 0)
);
-- ★ the reconciler: ★ an ISSUED invoice's stored total must equal
--   its lines, ★ forever.
SELECT count(*) FROM invoices i
 WHERE i.total_minor <> (SELECT coalesce(sum(line_total_minor),0)
                           FROM invoice_lines WHERE invoice_id = i.id);
-- ★ alert if > 0. ★ This is a legal document; drift is not
--   acceptable, and ★ detection must be independent of the writer.
```

**Step 3 — a one-way door nearly opened wrong.**

```
 ★ THE PROPOSAL: ★ "partition orders by month too — it's
   consistent."
 ★ THE CHALLENGE: ★ "what does that cost?"

 ⒜ ★ every UNIQUE must include placed_at ⇒ ★ `public_id` and
    `tracking_number` ★ can no longer be globally unique (59)
    ⇒ ★ and R4 is a lookup BY tracking_number with no date
    ⇒ ★ IT WOULD SCAN EVERY PARTITION.
 ⒝ ★ 44M rows in 3 years is ★ comfortably unpartitioned
 ⒞ ★ retention is 7 years ⇒ ★ nothing to DROP for a long time

 ⇒ ★ REJECTED, ★ and the reason recorded: ★ "partitioning orders
   would break the highest-volume read in the system."
 ⇒ ★ REVISIT TRIGGER: ★ 200M rows, ★ or if retention changes.
```

**Step 4 — the hot-row check, done before building.**

```sql
-- ★ THE PROPOSAL: ★ `organisations.order_count`, maintained by a
--   trigger, for a seller dashboard.
-- ★ THE CHECK (54 gate 4, 61):
SELECT seller_org_id, count(*) AS orders_per_day
  FROM orders WHERE placed_at > now() - interval '1 day'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 3;
```
```
 seller_org_id | orders_per_day
---------------+----------------
        ★ 1204 |       ★ 12,088     -- ★ 30% of volume
           882 |          4,102
```
```
 ★ 12,088/day = ★ 0.14/sec on the hottest row. ★ Not hot.
 ⇒ ★ BUT: ★ launch projections say 10× in 18 months ⇒ ★ 1.4/sec.
   ★ Still fine.
 ⇒ ★ AND at 100× ⇒ ★ 14/sec. ★ Still fine.
 ⇒ ★ DECISION: ★ the counter is safe. ★ Recorded, with the
   threshold at which it stops being safe (★ ~800/sec, Topic 61).
 ⇒ ★ THE POINT: ★ the check took two minutes and the answer was
   "yes". ★ Not every hot-row candidate is a hot row — ★ but you
   must measure to know.
```

**Step 5 — the transaction boundary, challenged.**

```
 ★ THE PROPOSAL: ★ "place the order, then call the payment
   gateway, then confirm — all in one transaction, so it's atomic."
 ★ THE CHALLENGE: ★ "how long does the gateway take?"
 ★ ANSWER: ★ "p99 4.1 seconds."

 ⇒ ★ THAT IS A 4.1-SECOND ROW LOCK ON `stock` (45, 65)
   ⇒ ★ Little's Law: ★ 30 orders/sec × 4.1 s = ★ 123 connections
     held, ★ just for order placement.
   ⇒ ★ AND every other buyer of the same SKU blocks.

 ⇒ ★ THE BOUNDARY REDRAWN:
   ★ TXN 1: idempotency key + stock decrement + order + items
            + outbox event.   ★ ~8 ms.
   ★ THEN:  the gateway call, ★ OUTSIDE any transaction, ★ with
            the same idempotency key passed through (52).
   ★ TXN 2: record the payment + update the order status
            + outbox event.   ★ ~4 ms.
   ★ SWEEPER: any order stuck in `pending` for >5 minutes is
            reconciled against the gateway. ★ Never guessed (52).
 ⇒ ★ MEASURED IN A PROTOTYPE: ★ 123 connections → ★ 1.
```

**Step 6 — what the review produced.**

| Finding | Severity | Fix |
|---|---|---|
| R4 (40k/min) had no index | ★ **critical** | promote `tracking_number` to a column, **52×** |
| Invoice total had no enforcement | ★ high | a frozen total + an independent reconciler |
| Proposed partitioning `orders` | ★ high | **rejected** — would break R4 |
| Gateway call inside the transaction | ★ **critical** | boundary redrawn, **123 connections → 1** |
| Per-seller counter | medium | ★ measured — safe, threshold recorded |
| `scan_events` FK omitted | low | ★ documented as deliberate + reconciled |
| No `updated_at` trigger | low | added to the conventions |

```
 ★ SIX LESSONS:
 ① ★ WALKING THE ACCESS-PATTERN LIST FOUND THE MOST EXPENSIVE
   BUG IN THE DESIGN, ★ before any code existed. ★ The
   completeness test — "point at the index for every pattern" —
   is the highest-value review question in this topic.
 ② ★ "THE APPLICATION COMPUTES IT" IS NOT AN ENFORCEMENT
   MECHANISM when three services will write the table.
 ③ ★ CONSISTENCY IS NOT A REASON TO PARTITION. ★ Partitioning
   `orders` would have broken the highest-volume read.
 ④ ★ THE HOT-ROW CHECK TOOK TWO MINUTES AND THE ANSWER WAS
   "SAFE". ★ Measuring is cheap; assuming is not — ★ in either
   direction.
 ⑤ ★ THE TRANSACTION BOUNDARY WAS THE MOST DANGEROUS PROPOSAL,
   and it looked like the SAFEST one ("so it's atomic").
 ⑥ ★ EVERY DELIBERATE OMISSION WAS RECORDED WITH ITS REASON AND
   ITS REVISIT TRIGGER. ★ That is what makes a design defensible
   rather than merely finished.
```

---

## Common mistakes

**1. Starting with the ERD.**
- *Symptom:* a beautiful model that cannot serve the highest-volume query, discovered in month eight.
- *Fix:* steps ①–⑤ produce no SQL. The access patterns *are* the specification.

**2. Not writing access patterns as sentences with volumes.**
- *Symptom:* "we need to query orders" — which cannot be turned into an index.
- *Fix:* "given a buyer org, list their last 20 orders newest first, 8,000/min, p99 < 100 ms."

**3. Treating invariants as documentation.**
- *Symptom:* "the application ensures it" — until a second service, a bulk import, or a psql session doesn't.
- *Fix:* classify each invariant and give it a mechanism the database enforces.

**4. Missing the snapshot-vs-copy distinction.**
- *Symptom:* an order's total changes when a seller updates a price; disputes become unanswerable.
- *Fix:* ask "if the source changes, must this change too?" and name the column accordingly.

**5. Deferring tenancy.**
- *Symptom:* retrofitting `org_id` and RLS to every table, query and index.
- *Fix:* add it on day one even with one tenant. It is the most expensive one-way door.

**6. `float` for money, `timestamp` without a zone.**
- *Symptom:* unrecoverable — every stored value is already wrong or ambiguous.
- *Fix:* `bigint` minor units, `timestamptz`. These are correctness, not preference.

**7. Not asking about retention.**
- *Symptom:* a 4 TB table with a `DELETE`-based retention job and no partitioning.
- *Fix:* if a table has a retention policy, partition it from day one.

**8. External calls inside a transaction.**
- *Symptom:* row locks held for a third party's p99; the connection pool exhausted.
- *Fix:* redraw the boundary — write, call outside, write again, with a sweeper.

**9. Partitioning for consistency rather than need.**
- *Symptom:* every unique constraint must include the partition key, breaking global lookups.
- *Fix:* partition when there is a retention or size reason, not for symmetry.

**10. Adding a maintained counter without checking the distribution.**
- *Symptom:* correctness fine, throughput destroyed on the busiest key.
- *Fix:* measure writes/sec on the hottest key and record the threshold at which it stops being safe.

**11. Indexing everything, or indexing nothing.**
- *Symptom:* a fact table with five unused indexes, or a 40,000/min query with none.
- *Fix:* one index per access pattern, verified with `EXPLAIN`; every omission recorded.

**12. No design record.**
- *Symptom:* nobody can say why a decision was made, so it is re-debated or reversed badly.
- *Fix:* patterns → indexes, invariants → mechanisms, deliberate omissions, one-way doors, signed.

---

## Hands-on proof

**PROVE IT #1–#4 — Example 1** (the URL shortener: a PK lookup at 0.031 ms serving 50,000/sec, an unindexed partitioned append at 0.058 ms, the rollup serving R3/R4, and the deliberate omission of an index on `clicks.code` justified by the query nobody runs).

**PROVE IT #5 — the completeness test, mechanised.**
```sql
-- ★ for every access pattern, run EXPLAIN and assert the index
--   is used. ★ Put this in CI.
DO $$
DECLARE r record; plan text;
BEGIN
  FOR r IN SELECT * FROM access_patterns LOOP
    EXECUTE 'EXPLAIN (FORMAT JSON) ' || r.query INTO plan;
    IF plan LIKE '%Seq Scan%' AND r.expects_index THEN
      RAISE EXCEPTION '★ pattern % uses a Seq Scan: %', r.id, r.query;
    END IF;
  END LOOP;
END $$;
```

**PROVE IT #6 — a snapshot survives a source change.**
```sql
INSERT INTO products (org_id, sku, name, list_price_minor, gst_rate)
VALUES (1,'SKU-1','Widget', 100000, 0.1800) RETURNING id \gset
-- place an order for it …
UPDATE products SET list_price_minor = 150000, name = 'Widget Pro'
 WHERE id = :id;

SELECT sku_at_order, name_at_order, unit_price_charged_minor
  FROM order_items WHERE product_id = :id;
```
```
 sku_at_order | name_at_order | unit_price_charged_minor
--------------+---------------+--------------------------
 SKU-1        | ★ Widget      |                 ★ 100000
 ★ THE ORDER IS UNAFFECTED. ★ No trigger. No reconciler.
```

**PROVE IT #7 — RLS fails closed.**
```sql
SET ROLE app_service;
SELECT set_config('app.org_id','', false);
SELECT count(*) FROM orders;
```
```
 count
-------
   ★ 0        — ★ unset ⇒ NULL ⇒ the policy matches nothing.
```

**PROVE IT #8 — the generated total cannot drift.**
```sql
UPDATE orders SET total_minor = 1 WHERE id = 1;
```
```
 ★ ERROR: column "total_minor" can only be updated to DEFAULT
```

**PROVE IT #9 — the transaction boundary, measured.**
```bash
# ★ with the gateway call inside
pgbench -f /tmp/order_with_gateway.sql -c 30 -j 8 -T 30 shop | grep tps
# ★ with it outside
pgbench -f /tmp/order_split.sql -c 30 -j 8 -T 30 shop | grep tps
```
```
 ★ inside:  tps = ★ 7.2      (30 clients × 4.1 s hold time)
 ★ outside: tps = ★ 2,884.1  ★ 400×
```

---

## The design decision framework

```
★★★ THE FIRST FIVE STEPS PRODUCE NO SQL.
    ★ THAT IS THE DISCIPLINE. ★★★

 ① ★ INTERROGATE — eight questions, answered by the BUSINESS
    one-or-many · always-or-sometimes · ★ can it change (and do we
    need the old value) · ★ volume in 3 years · ★ what must never
    be true · ★ who may see it · ★ how long do we keep it ·
    ★ what if it fails halfway
    ⇒ ★ 30 minutes of conversation opens the one-way doors
      correctly.

 ② ★ ACCESS PATTERNS AS SENTENCES, WITH VOLUMES AND LATENCY
    ⇒ ★ each becomes an index (10–19), a denormalisation IF the
      gates pass (54), or a rollup (56)
    ⇒ ★ COMPLETENESS TEST: ★ point at the index for EVERY pattern.
      ★ This is the highest-value review question there is.

 ③ ★ INVARIANTS, CLASSIFIED AND GIVEN A MECHANISM
    per-row ⇒ CHECK · unique ⇒ UNIQUE · overlap ⇒ EXCLUDE ·
    referential ⇒ FK · ★ across rows ⇒ SERIALIZABLE / a counter
    ⇒ ★ "the application ensures it" is not a mechanism.

 ④ ★ HOT PATHS — MEASURE, DON'T ASSUME
    rows per table in 3 years · rows per parent ·
    ★ writes/sec on the HOTTEST key
    ⇒ ★ decides partitioning (59), counters (55, 61), and whether
      a design survives its own success.

 ⑤ ★ ONE-WAY DOORS — SPEND THE JUDGEMENT HERE
    ★ tenancy (most expensive) · ★ PK type · ★ money & time ·
    ★ partitioning · aggregate boundaries · ★ idempotency
    ⇒ ★ two-way doors get a default, decided once, ★ never
      re-debated.

 ⑥–⑨ ★ ONLY NOW: model, normalise then bend, index from the
    patterns, constrain the invariants.
    ⇒ ★ normalise first (29–38); ★ bend only through Topic 54's
      gates; ★ snapshots are not bends.

 ⑩ ★ THE DESIGN RECORD IS THE DELIVERABLE
    ✓ patterns → indexes, ★ each verified with EXPLAIN
    ✓ invariants → mechanisms
    ✓ ★ DELIBERATE OMISSIONS, ★ with reasons
    ✓ ★ ONE-WAY DOORS, ★ with the alternatives rejected and why
    ✓ ★ REVISIT TRIGGERS, in numbers
    ✓ ★ signed by someone who owns the consequences

 ★ AND THE THREE QUESTIONS THAT FIND MOST DESIGN BUGS:
   ★ ① "point at the index for this pattern"
   ★ ② "what enforces this invariant when a second service writes?"
   ★ ③ "how long is the transaction, and what is inside it?"
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Take a one-sentence brief ("a library lending system") and produce only steps ①–⑤: the eight interrogation questions with plausible answers, the access patterns with volumes, the invariants, the hot paths, and the one-way doors. **Write no SQL.**

### Exercise 2 — medium (apply it)
Design the URL shortener completely. For each access pattern, write the query and prove with `EXPLAIN` which index serves it. Then justify, in writing, every index you did *not* create — including the one on `clicks.code`.

### Exercise 3 — hard (production simulation)
Design the wholesale marketplace from the brief in "How it works", then conduct the week-two review.

(a) Write the eight interrogation questions and the answer to each that would change your design. Identify which four answers open one-way doors.
(b) Write all nine read and five write patterns with volumes and latency targets.
(c) List seven invariants and classify each by mechanism.
(d) Produce the schema. For every table, justify: the primary key, whether it is partitioned, and which indexes exist.
(e) Run the completeness test. One pattern has no index — find it, and quantify the fix.
(f) One invariant's proposed mechanism is "the application computes it". Explain why that fails and give two alternatives with their trade-offs.
(g) Someone proposes partitioning `orders` for consistency. Give three reasons to reject it and the revisit trigger.
(h) Someone proposes a per-seller order counter. Run the check and record both the answer and the threshold at which it changes.
(i) Someone proposes putting the payment-gateway call inside the order transaction "so it's atomic". Compute the connection cost with Little's Law and redraw the boundary.
(j) Write the design record, including every deliberate omission with its reason.

---

## Mental model checkpoint

1. Name the ten steps. Which produce no SQL, and why does that matter?
2. Give the eight interrogation questions. Which one is most often skipped?
3. Why are access patterns the specification rather than the ERD?
4. What is the completeness test, and why is it the highest-value review question?
5. Classify five kinds of invariant and the mechanism for each. Which needs an isolation level?
6. Define a one-way door. Name six, in order of cost.
7. Why must tenancy be added on day one even with one tenant?
8. Give the snapshot-vs-copy test and three examples from an order schema.
9. Why does partitioning a table break global unique constraints, and when does that matter?
10. What must you measure before adding a maintained counter?
11. Why is "one transaction for atomicity" often the most dangerous proposal?
12. What does a design record contain, and what makes it defensible rather than merely finished?

---

## Quick reference card

**★ Ten steps** — ①–⑤ produce **no SQL**:
interrogate → ★ access patterns → invariants → hot paths → ★ one-way doors → model → normalise-then-bend → index → constrain → ★ record.

**★ Eight questions:** one-or-many · always-or-sometimes · ★ can it change (do we need the old value) · ★ volume in 3 years · ★ what must never be true · ★ who may see it · ★ how long do we keep it · ★ what if it fails halfway.

**★ Invariant → mechanism**

| Kind | Mechanism |
|---|---|
| per-row | `CHECK` |
| uniqueness | `UNIQUE` |
| overlap | `EXCLUDE USING gist` |
| referential | `FOREIGN KEY` |
| ★ across rows (count/sum) | ★ `SERIALIZABLE` or a counter row |

**★ One-way doors** (cost order): tenancy · PK type · money/time · partitioning · aggregate boundaries · idempotency.
**Two-way doors:** names, columns, indexes, `CHECK`s — **take the default, write it down once.**

**★ Defaults:** `bigint` minor units · `timestamptz` · `NOT NULL` by default · `text` + `CHECK` (not `VARCHAR(n)`) · `created_at`/`updated_at` · ★ `org_id` leading every index · ★ no soft deletes · ★ no EAV · ★ no polymorphic FKs · ★ store sum and count, never an average.

**★ The three questions that find most design bugs:**
① *point at the index for this pattern* · ② *what enforces this when a second service writes?* · ③ *how long is the transaction, and what is inside it?*

---

## When would I use this at work?

1. **Every new service or major feature.** The first five steps take an afternoon and are the difference between a schema that serves its workload and one that has to be rewritten. The interrogation alone — eight questions, thirty minutes — routinely opens one-way doors correctly that would otherwise be found in month eight.

2. **Reviewing someone else's design.** The completeness test ("point at the index for every access pattern") finds more real bugs than reading the DDL, and it works before any code exists.

3. **When a design decision is being debated.** Classify it: one-way or two-way. Two-way doors get a default and five minutes; one-way doors get the analysis and go in the signed record. That single distinction ends most architecture arguments.

4. **Writing the design record.** The value is not the decision — it is the **rejected alternatives with reasons** and the **revisit triggers in numbers**. That is what makes a schema defensible to a reviewer, and what stops the same debate recurring every quarter.

---

## Connected topics

**Understand before this:** all of Phases 1–8 — this topic is their application. Especially 20–28 (modelling), 29–38 (normalisation), 54 (the gates), 59 (partitioning), 52 (idempotency and the outbox), 61 (hot rows), 69 (tenancy and RLS).

**This unlocks:**
- **78** — the same system under load: what breaks first, and in what order
- **79** — the principal-engineer review checklist
