# 04 — Subscription Billing
## Recomputing history without rewriting it

> **Read the brief. Close the file. Design it yourself. Then come back.**

---

## WHY THIS CASE STUDY EXISTS

Case study 03 established that money is an append-only ledger. This one adds the dimension that breaks most billing systems: **time**.

A subscription is not a fact. It is a fact *that was true between two dates*, changed mid-period, backdated by support three weeks later, and must still produce an invoice that a customer, an auditor, and a tax authority all agree with — including the invoice you already sent them.

The hardest question in billing is not "how much do they owe?" It is: **"what did we believe they owed, on the day we sent the invoice, and why is today's answer different?"** A system that cannot answer that will eventually send a corrected invoice that nobody can explain.

---

## 1. THE BRIEF

A B2B SaaS platform billing on usage plus seats.

> **40,000 customer accounts. 40M usage events/day. 8 currencies. 4 plan tiers.**
> **Monthly invoicing on the customer's own anniversary date — so invoices run every day, not on the 1st.**

Business rules:

1. **Seat-based + metered.** Base plan (per seat, per month) + metered overage (API calls, storage GB-hours, egress).
2. **Mid-cycle changes are prorated to the second.** Upgrade on day 12 of 30 → credit the unused portion of the old plan, charge the used portion of the new one.
3. **Invoices are immutable once issued.** A correction is a **credit note**, never an edit.
4. **Backdating happens.** Support routinely applies a discount "effective from last month." The system must recompute what *should* have been charged without altering what *was* charged.
5. **Usage arrives late and out of order.** An edge collector can be 6 hours behind. Events can duplicate.
6. **Tax is jurisdictional and changes.** The rate applied is the rate that was in force *on the invoice date*, not today's rate.
7. **Dunning.** Failed payments retry on a schedule; the subscription's state depends on how long it's been failing.
8. **Customers can see, and will dispute, every line.**

Infrastructure: PostgreSQL 16 (primary + 2 replicas), 40 Node.js pods, Kafka for usage ingest, S3 for invoice PDFs.

---

## 2. ACCESS PATTERN TABLE

| # | Operation | Peak rate | Rows read | Rows written | p99 budget | Staleness |
|---|---|---|---|---|---|---|
| A | Usage ingest — batched from Kafka | 40M/day (peak 2,000/s) | 0 | 2,000/batch | 2 s | 0 |
| B | `GET /usage/current` — customer dashboard | 400/s | ~30 | 0 | 300 ms | **5 min** |
| C | Nightly invoice run — ~1,300 accounts/day | 1,300/night | ~40k each | ~200 each | 4 h window | 0 |
| D | `POST /subscription/change` — upgrade/downgrade | 20/s | 6 | 4 | 400 ms | 0 |
| E | `GET /invoices/:id` | 800/s | ~60 | 0 | 200 ms | 60 s (replica) |
| F | Payment webhook (Stripe) | 200/s | 4 | 5 | 500 ms | 0 |
| G | Dunning sweeper | every 15 min | ~5,000 | ~200 | n/a | 0 |
| H | Support: "apply discount from last month" | 10/day | ~50 | ~10 | n/a | 0 |
| I | Finance: revenue recognition report | 1/day | 200M | 0 | n/a | 0 |

**What jumps out:**

- **A at 40M/day is the volume problem**, but it's append-only and batched — the easy part, structurally (Topic 73).
- **C is the correctness problem.** Each invoice reads ~40,000 usage rows and must produce a number that is *reproducible forever*.
- **H is the design problem.** A retroactive change must alter the *future* answer without altering the *past* record. This is the requirement that forces bitemporal modelling.
- **I reads 200M rows** — it belongs on a replica or a columnar rollup, never on the primary (Topic 06).

---

## 3. THE INVARIANTS

```
I1.  IMMUTABILITY: an issued invoice's line items and total NEVER change.
     Corrections are credit notes referencing the original.

I2.  REPRODUCIBILITY: recomputing an invoice from stored inputs, at any
     future date, must produce EXACTLY the number that was issued —
     even after plans, prices, tax rates, and discounts have changed.
     ★ This is the invariant that dictates the entire schema.

I3.  IDEMPOTENT USAGE: the same usage event ingested N times counts ONCE.

I4.  NO GAPS, NO OVERLAPS: for any subscription, the billing periods
     must tile time exactly. sum(period durations) = subscription lifetime.

I5.  PRORATION CONSERVATION: for a mid-cycle change,
       credit_for_unused + charge_for_new = what a clean period would cost
     (± rounding, which must be explicit and consistent)

I6.  ONE ACTIVE SUBSCRIPTION per (account, product) at any instant.

I7.  LEDGER LINKAGE: every invoice total has corresponding ledger
     entries (case study 03). Invoices and the ledger never disagree.
```

**I2 is the one everything hangs on.** If you can reproduce any past invoice exactly, disputes are answerable, audits pass, and corrections are safe. If you can't, every price change silently rewrites history.

---

## 4. ATTEMPT 1 — THE NAIVE SCHEMA

```sql
CREATE TABLE plans (
  id         bigserial PRIMARY KEY,
  name       text NOT NULL,
  price_minor bigint NOT NULL,          -- per seat per month
  currency   char(3) NOT NULL
);

CREATE TABLE subscriptions (
  id          bigserial PRIMARY KEY,
  account_id  bigint  NOT NULL REFERENCES accounts(id),
  plan_id     bigint  NOT NULL REFERENCES plans(id),
  seats       int     NOT NULL,
  status      text    NOT NULL DEFAULT 'active',
  started_at  timestamptz NOT NULL,
  next_bill_at timestamptz NOT NULL
);

CREATE TABLE usage_events (
  id          bigserial PRIMARY KEY,
  account_id  bigint  NOT NULL,
  metric      text    NOT NULL,
  quantity    bigint  NOT NULL,
  occurred_at timestamptz NOT NULL
);

CREATE TABLE invoices (
  id         bigserial PRIMARY KEY,
  account_id bigint NOT NULL,
  total_minor bigint NOT NULL,
  period_start timestamptz NOT NULL,
  period_end   timestamptz NOT NULL,
  status     text NOT NULL DEFAULT 'draft',
  issued_at  timestamptz
);
```

```js
// ATTEMPT 1 — the invoice run
async function generateInvoice(accountId, periodStart, periodEnd) {
  const sub = await db.query(
    `SELECT s.seats, p.price_minor, p.currency
       FROM subscriptions s JOIN plans p ON p.id = s.plan_id
      WHERE s.account_id = $1 AND s.status = 'active'`, [accountId]);

  const base = sub.rows[0].seats * sub.rows[0].price_minor;

  const usage = await db.query(
    `SELECT metric, sum(quantity) AS q FROM usage_events
      WHERE account_id = $1 AND occurred_at >= $2 AND occurred_at < $3
      GROUP BY metric`, [accountId, periodStart, periodEnd]);

  const overage = usage.rows.reduce((t, r) => t + computeOverage(r.metric, r.q), 0);

  return db.query(
    `INSERT INTO invoices (account_id, total_minor, period_start, period_end, status, issued_at)
     VALUES ($1,$2,$3,$4,'issued',now()) RETURNING *`,
    [accountId, base + overage, periodStart, periodEnd]);
}
```

Plausible. Ships in a week. Now watch every invariant break.

---

## 5. WHERE IT BREAKS

### 5.1 Failure one: I2 dies the first time a price changes

```
 2026-03-15  Invoice #4471 issued.  Plan 'pro' = ₹2,000/seat. 10 seats.
             total = ₹20,000. PDF emailed. Payment collected.

 2026-06-01  Marketing raises 'pro' to ₹2,500/seat.
             UPDATE plans SET price_minor = 250000 WHERE id = 3;

 2026-07-02  Customer disputes invoice #4471.
             Support recomputes: 10 seats × ₹2,500 = ₹25,000.
             The stored invoice says ₹20,000.

             WHICH IS RIGHT? Nobody can tell, because the input
             (the price on 2026-03-15) WAS OVERWRITTEN.
```

**One `UPDATE` destroyed the reproducibility of every historical invoice.** And it's worse than it looks: the invoice total was *stored*, so the number survived — but the *explanation* didn't. You have a total you cannot justify, which in an audit is nearly as bad as a wrong total.

### 5.2 Failure two: `subscriptions` has no history at all

```sql
UPDATE subscriptions SET seats = 25 WHERE id = 88;   -- customer added 15 seats
```

```
 What was the seat count on 2026-03-15? UNKNOWABLE.
 When did it change? UNKNOWABLE.
 Was the mid-cycle change prorated? UNANSWERABLE.
 Did they downgrade and re-upgrade? Invisible.

 I4 (no gaps/overlaps) cannot even be CHECKED — there are no periods
 to check, just a single mutable row.
```

### 5.3 Failure three: duplicate usage, silently double-billed

```sql
-- The Kafka consumer retries after a timeout. Same batch, ingested twice.
SELECT metric, sum(quantity) FROM usage_events
WHERE account_id = 8812 AND occurred_at >= '2026-03-01' AND occurred_at < '2026-04-01'
GROUP BY metric;
```
```
     metric      |    sum
-----------------+------------
 api_calls       |  84,102,884      ← should be 42,051,442
```

**The customer is billed twice for the same API calls.** There is no uniqueness constraint, no event id, no deduplication. Violates I3, and you find out when they complain.

### 5.4 Failure four: the invoice run is a full scan per account

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT metric, sum(quantity) FROM usage_events
WHERE account_id = 8812 AND occurred_at >= '2026-03-01' AND occurred_at < '2026-04-01'
GROUP BY metric;
```
```
HashAggregate  (actual time=18420.1..18420.2 rows=4 loops=1)
  ->  Bitmap Heap Scan on usage_events  (actual time=2104.2..18102.4 rows=42051442)
        Recheck Cond: (account_id = 8812)
        Filter: ((occurred_at >= ...) AND (occurred_at < ...))
        Rows Removed by Filter: 388102229          ← ⚠ 388 MILLION
        Heap Blocks: exact=8912004
        Buffers: shared hit=41204 read=8870800     ← 68 GB
Execution Time: 18421.9 ms
```

**18.4 seconds per account × 1,300 accounts = 6.6 hours.** The 4-hour window is blown, and it gets worse every month because `usage_events` grows forever with no partitioning. The `Rows Removed by Filter: 388102229` is the giveaway — the index has `account_id` but not `occurred_at` (Topic 12).

### 5.5 Failure five: proration is impossible

```
 Customer upgrades from 'starter' (₹500/seat) to 'pro' (₹2,000/seat)
 on day 12 of a 30-day period, and goes from 10 seats to 25.

 UPDATE subscriptions SET plan_id = 3, seats = 25 WHERE id = 88;

 At invoice time, the code reads the CURRENT plan and seats:
     25 × ₹2,000 = ₹50,000 for the whole month.

 CORRECT ANSWER:
     12/30 × 10 × ₹500   = ₹2,000   (starter, first 12 days)
   + 18/30 × 25 × ₹2,000 = ₹30,000  (pro, remaining 18 days)
                          = ₹32,000

 OVERCHARGED BY ₹18,000. And the customer can see it.

 There is no way to fix this without knowing WHEN the change happened
 and WHAT the values were before. The schema stores neither.
```

### 5.6 Failure six: backdating rewrites issued invoices

```
 Support applies a 20% discount "effective from 1 March."
 The obvious implementation: recompute and UPDATE the March invoice.

 ✗ Violates I1. The customer already has the PDF. The payment is
   reconciled. The tax filing used the old number. Changing it means
   the ledger (case study 03) no longer matches the invoice.
```

---

## 6. ATTEMPT 2 — THE FIX THAT ISN'T ENOUGH

**Version the plans.** The obvious response to failure 5.1:

```sql
CREATE TABLE plan_versions (
  id          bigserial PRIMARY KEY,
  plan_id     bigint NOT NULL,
  price_minor bigint NOT NULL,
  valid_from  timestamptz NOT NULL,
  valid_to    timestamptz NOT NULL DEFAULT 'infinity'
);
ALTER TABLE invoices ADD COLUMN plan_version_id bigint REFERENCES plan_versions(id);
```

Now the invoice records *which* price it used, and historical prices are never overwritten. Genuine progress: **I2 is partially restored.**

Add an index for the invoice run:

```sql
CREATE INDEX idx_usage_acct_time ON usage_events (account_id, occurred_at);
```
```
 invoice run: 18.4 s/account → 1.9 s/account → 41 minutes total ✓
```

Add an event id for dedup:

```sql
ALTER TABLE usage_events ADD COLUMN event_id uuid NOT NULL;
CREATE UNIQUE INDEX uq_usage_event ON usage_events (event_id);
```
```
 duplicate billing: fixed ✓
```

**Three real fixes. And these remain broken:**

| Still broken | Why versioning plans doesn't help |
|---|---|
| Seat/plan history (5.2) | `subscriptions` is still a single mutable row. Versioning the *plan* doesn't version the *subscription*. |
| Proration (5.5) | Still no record of *when* the change happened. |
| Backdating (5.6) | Still no way to say "what we believed then" vs "what is true now." |
| Tax rate history | Same problem as plans, one layer up. |
| Discounts | Same problem again. |

**The realisation:** patching one table at a time is whack-a-mole. **Every input to an invoice needs a validity period, and the invoice needs to record which version of each input it used.** That's not a fix — it's a modelling paradigm, and it has a name.

---

## 7. THE PRODUCTION DESIGN

```
 PROBLEM                              SOLUTION                        TOPIC
 ──────────────────────────────────────────────────────────────────────────
 Overwritten inputs destroy I2   →  BITEMPORAL versioning of every
                                    billing input                     26, 17
 No subscription history         →  Subscription PERIODS, not a row   26
 Backdating vs immutability      →  valid_time vs transaction_time,
                                    two separate axes                 26
 Invoice can't be reproduced     →  Invoices store a frozen SNAPSHOT
                                    of every input they used          55
 40M usage rows/day              →  Partitioned + pre-aggregated
                                    hourly rollups                    59, 73
 Corrections                     →  Credit notes, never edits         03
```

### 7.1 The central idea: two time axes

This is the concept the whole design rests on. Most developers have never used it explicitly, and once you have, you see the need for it everywhere.

```
 VALID TIME  ("effective time", "business time")
   WHEN THE FACT WAS TRUE IN THE REAL WORLD.
   "The customer had 25 seats from 12 March to 30 April."

 TRANSACTION TIME  ("system time", "knowledge time")
   WHEN WE RECORDED IT.
   "We learned about those 25 seats on 2 April at 14:32."

 ★ BACKDATING = valid_from is in the past, recorded_at is now.
   THESE ARE ORTHOGONAL. You need BOTH to answer:

     "What do we believe TODAY about MARCH?"
         → filter valid_time ∈ March, recorded_at ≤ now
     "What did we believe ON 31 MARCH about MARCH?"    ★ THE AUDIT QUESTION
         → filter valid_time ∈ March, recorded_at ≤ 2026-03-31

   The second query is what makes an issued invoice defensible. Without
   transaction time, you can only ever answer the first — which means
   every backdated change silently rewrites what you "always knew."
```

```
                    transaction time (when we learned it) ──▶
 valid    ┌────────────────────────────────────────────────────┐
 time     │                                                    │
   │      │   seats=10          seats=10                       │
   │      │   (recorded 1 Mar)  (still believed)               │
   ▼      │                                                    │
 March    │─────────────────────┼──────────────────────────────│
          │                     │                              │
 April    │   seats=10          │  seats=25                    │
          │   (believed)        │  (recorded 2 Apr, valid      │
          │                     │   from 12 Mar — BACKDATED)   │
          └─────────────────────┴──────────────────────────────┘
                                ↑
                    The March invoice, issued 31 Mar, used seats=10.
                    That was CORRECT given what we knew then.
                    Today we know it should have been 25 from 12 Mar.
                    ⇒ ISSUE A CREDIT NOTE for the difference.
                      DO NOT touch the March invoice.
```

### 7.2 The schema

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- ═══════════════════════════════════════════════════════════════════
-- PRODUCT CATALOGUE — every price is a versioned fact
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE plans (
  id   bigserial PRIMARY KEY,
  code text NOT NULL UNIQUE,          -- 'starter','pro','enterprise'
  name text NOT NULL
);

CREATE TABLE plan_prices (
  id            bigserial   PRIMARY KEY,
  plan_id       bigint      NOT NULL REFERENCES plans(id),
  currency      char(3)     NOT NULL,
  seat_price_minor bigint   NOT NULL CHECK (seat_price_minor >= 0),
  included_units jsonb      NOT NULL DEFAULT '{}',   -- {"api_calls": 1000000}
  valid          tstzrange  NOT NULL,                -- VALID TIME
  recorded_at    timestamptz NOT NULL DEFAULT now(), -- TRANSACTION TIME
  recorded_by    text        NOT NULL,

  -- I4 for prices: no two prices for the same plan+currency may overlap
  EXCLUDE USING gist (plan_id WITH =, currency WITH =, valid WITH &&)
);

CREATE TABLE metered_rates (
  id          bigserial  PRIMARY KEY,
  plan_id     bigint     NOT NULL REFERENCES plans(id),
  metric      text       NOT NULL,           -- 'api_calls','storage_gb_hours'
  currency    char(3)    NOT NULL,
  tiers       jsonb      NOT NULL,           -- [{"up_to":1e6,"rate":0},{"up_to":null,"rate":2}]
  valid       tstzrange  NOT NULL,
  recorded_at timestamptz NOT NULL DEFAULT now(),
  EXCLUDE USING gist (plan_id WITH =, metric WITH =, currency WITH =, valid WITH &&)
);

CREATE TABLE tax_rates (
  id            bigserial  PRIMARY KEY,
  jurisdiction  text       NOT NULL,          -- 'IN-KA','US-CA','GB'
  tax_type      text       NOT NULL,          -- 'GST','VAT','SALES'
  rate_bp       int        NOT NULL,          -- basis points: 1800 = 18%
  valid         tstzrange  NOT NULL,
  recorded_at   timestamptz NOT NULL DEFAULT now(),
  EXCLUDE USING gist (jurisdiction WITH =, tax_type WITH =, valid WITH &&)
);

-- ═══════════════════════════════════════════════════════════════════
-- SUBSCRIPTION PERIODS — the subscription IS its history
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE subscriptions (
  id          bigserial   PRIMARY KEY,
  account_id  bigint      NOT NULL REFERENCES accounts(id),
  product_id  bigint      NOT NULL,
  currency    char(3)     NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  cancelled_at timestamptz NULL
);

-- ★ THE KEY TABLE. Each row is "these terms held for this stretch of time."
CREATE TABLE subscription_periods (
  id              bigserial   PRIMARY KEY,
  subscription_id bigint      NOT NULL REFERENCES subscriptions(id),
  plan_id         bigint      NOT NULL REFERENCES plans(id),
  seats           int         NOT NULL CHECK (seats > 0),
  valid           tstzrange   NOT NULL,               -- VALID TIME
  recorded_at     timestamptz NOT NULL DEFAULT now(), -- TRANSACTION TIME
  superseded_at   timestamptz NULL,                   -- when this ROW stopped
                                                      -- being our belief
  change_reason   text        NOT NULL,               -- 'signup','upgrade',
                                                      -- 'seat_change','backdated_fix'
  changed_by      text        NOT NULL,

  -- I4 + I6: among CURRENTLY-BELIEVED rows, periods must not overlap
  EXCLUDE USING gist (subscription_id WITH =, valid WITH &&)
    WHERE (superseded_at IS NULL)
);
CREATE INDEX idx_subper_current ON subscription_periods (subscription_id, lower(valid))
  WHERE superseded_at IS NULL;

-- ═══════════════════════════════════════════════════════════════════
-- USAGE — append-only, partitioned, deduplicated
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE usage_events (
  event_id    uuid        NOT NULL,
  account_id  bigint      NOT NULL,
  metric      text        NOT NULL,
  quantity    bigint      NOT NULL CHECK (quantity >= 0),
  occurred_at timestamptz NOT NULL,
  ingested_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (occurred_at, event_id)
) PARTITION BY RANGE (occurred_at);
-- I3: dedup within a partition (the natural window; events never arrive
--     more than a few hours late, and partitions are daily)
CREATE UNIQUE INDEX ON usage_events (event_id, occurred_at);
CREATE INDEX idx_usage_acct ON usage_events (account_id, metric, occurred_at);

-- ★ HOURLY ROLLUP — pattern C reads this, not the raw events
CREATE TABLE usage_rollup_hourly (
  account_id  bigint      NOT NULL,
  metric      text        NOT NULL,
  hour        timestamptz NOT NULL,
  quantity    bigint      NOT NULL,
  event_count int         NOT NULL,
  sealed_at   timestamptz NULL,     -- ★ set once the hour can no longer change
  PRIMARY KEY (account_id, metric, hour)
) PARTITION BY RANGE (hour);

-- ═══════════════════════════════════════════════════════════════════
-- INVOICES — immutable, self-contained, reproducible
-- ═══════════════════════════════════════════════════════════════════
CREATE TYPE invoice_state AS ENUM ('draft','issued','paid','void','uncollectible');

CREATE TABLE invoices (
  id              bigint        PRIMARY KEY,       -- ULID
  account_id      bigint        NOT NULL REFERENCES accounts(id),
  number          text          NOT NULL UNIQUE,   -- 'INV-2026-000041208' — legally sequential
  state           invoice_state NOT NULL DEFAULT 'draft',
  currency        char(3)       NOT NULL,
  period          tstzrange     NOT NULL,
  subtotal_minor  bigint        NOT NULL,
  tax_minor       bigint        NOT NULL,
  total_minor     bigint        NOT NULL,
  issued_at       timestamptz   NULL,
  due_at          timestamptz   NULL,
  idempotency_key text          NOT NULL,

  -- ★★★ THE REPRODUCIBILITY SNAPSHOT ★★★
  -- Every input used, frozen at issue time. This is what makes I2 true.
  computation_inputs jsonb      NOT NULL,
  computation_version int       NOT NULL,          -- which code version computed it

  CHECK (total_minor = subtotal_minor + tax_minor),
  CHECK (state = 'draft' OR issued_at IS NOT NULL)
);
CREATE UNIQUE INDEX uq_invoice_idem ON invoices (idempotency_key);
CREATE INDEX idx_invoices_account ON invoices (account_id, issued_at DESC);
-- I1: enforced by grants, not convention
CREATE RULE no_update_issued AS ON UPDATE TO invoices
  WHERE OLD.state <> 'draft' AND NEW.state = OLD.state DO INSTEAD NOTHING;

CREATE TABLE invoice_lines (
  id            bigserial   PRIMARY KEY,
  invoice_id    bigint      NOT NULL REFERENCES invoices(id),
  line_no       int         NOT NULL,
  kind          text        NOT NULL,       -- 'seat','metered','proration_credit',
                                            -- 'proration_charge','discount','tax'
  description   text        NOT NULL,
  quantity      numeric     NOT NULL,
  unit_price_minor bigint   NOT NULL,
  amount_minor  bigint      NOT NULL,
  period        tstzrange   NULL,
  source_refs   jsonb       NOT NULL,       -- {"plan_price_id":88,"sub_period_id":41}
  UNIQUE (invoice_id, line_no)
);

-- ═══════════════════════════════════════════════════════════════════
-- CREDIT NOTES — the ONLY way to change what a customer owes
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE credit_notes (
  id                 bigint      PRIMARY KEY,
  number             text        NOT NULL UNIQUE,
  original_invoice_id bigint     NOT NULL REFERENCES invoices(id),
  reason             text        NOT NULL,
  amount_minor       bigint      NOT NULL CHECK (amount_minor > 0),
  currency           char(3)     NOT NULL,
  issued_at          timestamptz NOT NULL DEFAULT now(),
  computation_inputs jsonb       NOT NULL
);
```

**Every decision justified:**

| Decision | Why |
|---|---|
| `tstzrange valid` on every price/rate/period | I2 — the input as of any date is recoverable |
| `EXCLUDE USING gist (... WITH &&)` | I4 — gaps are allowed, overlaps are structurally impossible (case study 02's technique) |
| `recorded_at` + `superseded_at` | the transaction-time axis — separates "what's true" from "what we knew" |
| `computation_inputs jsonb` on invoices | ★ the reproducibility snapshot — see 7.3 |
| `computation_version` | when the billing *algorithm* changes, you know which one produced this number |
| Usage partitioned by `occurred_at` | 40M/day; retention = `DROP PARTITION` (Topic 59) |
| `usage_rollup_hourly.sealed_at` | marks an hour as final so the invoice run can trust it |
| Credit notes, not edits | I1 |
| `number` separate from `id` | invoice numbers are legally sequential per year; ULIDs are not |

### 7.3 The reproducibility snapshot — the heart of the design

```jsonc
// invoices.computation_inputs for INV-2026-000041208
{
  "engine_version": 7,
  "computed_at": "2026-03-31T18:00:04.881Z",
  "period": ["2026-03-01T00:00:00Z", "2026-04-01T00:00:00Z"],

  "subscription_periods": [
    { "id": 9104, "plan_id": 2, "plan_code": "starter", "seats": 10,
      "valid": ["2026-03-01T00:00:00Z", "2026-03-12T14:22:10Z"],
      "seconds": 998530 },
    { "id": 9188, "plan_id": 3, "plan_code": "pro", "seats": 25,
      "valid": ["2026-03-12T14:22:10Z", "2026-04-01T00:00:00Z"],
      "seconds": 1679870 }
  ],

  "plan_prices_used": [
    { "id": 441, "plan_id": 2, "seat_price_minor": 50000, "currency": "INR",
      "valid_from": "2025-01-01T00:00:00Z" },
    { "id": 452, "plan_id": 3, "seat_price_minor": 200000, "currency": "INR",
      "valid_from": "2025-06-01T00:00:00Z" }
  ],

  "metered_rates_used": [
    { "id": 88, "metric": "api_calls", "tiers": [{"up_to":1000000,"rate":0},
                                                 {"up_to":null,"rate":2}] }
  ],

  "usage_totals": {
    "api_calls": { "quantity": 4205144, "rollup_hours": 744,
                   "last_sealed_hour": "2026-03-31T23:00:00Z" },
    "storage_gb_hours": { "quantity": 18402, "rollup_hours": 744 }
  },

  "tax": { "jurisdiction": "IN-KA", "type": "GST", "rate_bp": 1800,
           "tax_rate_id": 12 },

  "proration": { "method": "seconds", "period_seconds": 2678400,
                 "rounding": "half_even", "rounding_unit": "minor" },

  "discounts": []
}
```

**Why this matters more than it looks:** the invoice does not merely reference `plan_price_id = 441` — it **embeds the value**. Even if someone `DELETE`s a plan version, drops a rollup partition after retention, or migrates the schema, the invoice can still be explained line by line, forever, from a single row.

This is a deliberate, *correct* denormalisation (Topic 55). The rule that makes it safe: **the snapshot is the record of a computation, not a cache of current state.** It is written once and never refreshed.

### 7.4 Proration — computed to the second

```js
// Given: period, and the subscription_periods that overlap it.
// I5: credit + charge must equal what a clean period would cost.

function prorate(periodStart, periodEnd, subPeriods, priceLookup, rounding) {
  const periodSeconds = (periodEnd - periodStart) / 1000;
  const lines = [];

  for (const sp of subPeriods) {
    // clip the subscription period to the invoice period
    const from = new Date(Math.max(sp.validFrom, periodStart));
    const to   = new Date(Math.min(sp.validTo,   periodEnd));
    const seconds = (to - from) / 1000;
    if (seconds <= 0) continue;

    // ★ price is looked up AS OF `from` — not as of now
    const price = priceLookup(sp.planId, from);
    const fullMonth = BigInt(sp.seats) * BigInt(price.seatPriceMinor);
    const amount = roundMinor(
      (fullMonth * BigInt(Math.round(seconds))) / BigInt(Math.round(periodSeconds)),
      rounding);

    lines.push({
      kind: 'seat',
      description: `${price.planCode} — ${sp.seats} seats (${days(seconds)} days)`,
      quantity: sp.seats,
      unitPriceMinor: price.seatPriceMinor,
      amountMinor: amount,
      period: [from, to],
      sourceRefs: { planPriceId: price.id, subPeriodId: sp.id },
    });
  }
  return lines;
}
```

The SQL that feeds it:

```sql
-- Every subscription period overlapping the invoice window, as currently believed
SELECT sp.id, sp.plan_id, sp.seats,
       GREATEST(lower(sp.valid), $2) AS eff_from,
       LEAST(COALESCE(upper(sp.valid), $3), $3) AS eff_to
FROM subscription_periods sp
WHERE sp.subscription_id = $1
  AND sp.superseded_at IS NULL              -- ★ current belief only
  AND sp.valid && tstzrange($2, $3)         -- ★ && = overlaps. Uses the GiST index.
ORDER BY lower(sp.valid);
```

```sql
-- The price in force at a given instant
SELECT id, seat_price_minor, included_units
FROM plan_prices
WHERE plan_id = $1 AND currency = $2 AND valid @> $3::timestamptz;
--                                     ★ @> = contains
```

Two operators — `&&` (overlaps) and `@>` (contains) — replace an entire category of buggy `BETWEEN` logic, and both are GiST-indexable.

### 7.5 Changing a subscription (pattern D)

```js
async function changeSubscription({ subscriptionId, newPlanId, newSeats,
                                    effectiveAt, reason, actor, idempotencyKey }) {
  return withTransaction(async (tx) => {
    await tx.query("SET LOCAL lock_timeout = '2s'");

    // Serialise changes to this subscription — cheap, no row contention
    await tx.query('SELECT pg_advisory_xact_lock($1)', [subscriptionId]);

    const eff = effectiveAt ?? (await tx.query('SELECT now()')).rows[0].now;

    // Close the currently-open period at `eff`
    const open = await tx.query(
      `SELECT id, plan_id, seats, valid FROM subscription_periods
        WHERE subscription_id=$1 AND superseded_at IS NULL AND upper(valid)='infinity'
        FOR UPDATE`, [subscriptionId]);

    if (open.rowCount) {
      const cur = open.rows[0];
      if (eff <= lower(cur.valid))
        throw new AppError('EFFECTIVE_DATE_BEFORE_PERIOD_START');

      // ★ Supersede + re-insert, rather than UPDATE the range in place.
      //   Keeps the transaction-time history intact.
      await tx.query(
        `UPDATE subscription_periods SET superseded_at = now() WHERE id = $1`, [cur.id]);
      await tx.query(
        `INSERT INTO subscription_periods
           (subscription_id, plan_id, seats, valid, change_reason, changed_by)
         VALUES ($1,$2,$3, tstzrange(lower($4::tstzrange), $5, '[)'), $6, $7)`,
        [subscriptionId, cur.plan_id, cur.seats, cur.valid, eff, 'closed_by_change', actor]);
    }

    // Open the new period. The EXCLUDE constraint guarantees no overlap.
    const created = await tx.query(
      `INSERT INTO subscription_periods
         (subscription_id, plan_id, seats, valid, change_reason, changed_by)
       VALUES ($1,$2,$3, tstzrange($4,'infinity','[)'), $5, $6)
       RETURNING id`,
      [subscriptionId, newPlanId, newSeats, eff, reason, actor]);

    await tx.query(
      `INSERT INTO outbox (topic, payload) VALUES ('subscription.changed', $1)`,
      [JSON.stringify({ subscriptionId, periodId: created.rows[0].id, effectiveAt: eff })]);

    return { periodId: created.rows[0].id };
  });
}
```

**If `effectiveAt` is in the past, this is a backdate** — and it works without special-casing: the new period's `valid` starts in the past, `recorded_at` is now, and any already-issued invoice covering that time is untouched. The reconciler (7.7) then detects the discrepancy and proposes a credit note.

### 7.6 The invoice run (pattern C)

```js
async function issueInvoice({ accountId, periodStart, periodEnd, idempotencyKey }) {
  return withTransaction(async (tx) => {
    // Idempotent: a re-run of the nightly job cannot double-invoice
    const inv = await tx.query(
      `INSERT INTO invoices (id, account_id, number, state, currency, period,
                             subtotal_minor, tax_minor, total_minor,
                             idempotency_key, computation_inputs, computation_version)
       VALUES ($1,$2,$3,'draft',$4,tstzrange($5,$6,'[)'),0,0,0,$7,'{}'::jsonb,$8)
       ON CONFLICT (idempotency_key) DO NOTHING
       RETURNING id`,
      [ulid(), accountId, await nextInvoiceNumber(tx), currency,
       periodStart, periodEnd, idempotencyKey, ENGINE_VERSION]);

    if (inv.rowCount === 0) {
      const prev = await tx.query(
        'SELECT id, number, total_minor, state FROM invoices WHERE idempotency_key=$1',
        [idempotencyKey]);
      return { replayed: true, ...prev.rows[0] };
    }
    const invoiceId = inv.rows[0].id;

    // ── Gather every input, and RECORD what we gathered ──
    const subPeriods = await loadSubscriptionPeriods(tx, accountId, periodStart, periodEnd);
    const prices     = await loadPricesAsOf(tx, subPeriods);
    const rates      = await loadMeteredRatesAsOf(tx, subPeriods);
    const usage      = await loadSealedRollups(tx, accountId, periodStart, periodEnd);
    const tax        = await loadTaxRateAsOf(tx, accountId, periodEnd);
    const discounts  = await loadDiscountsAsOf(tx, accountId, periodStart, periodEnd);

    // ⚠ Refuse to invoice on unsealed usage — better late than wrong
    if (usage.unsealedHours > 0)
      throw new RetryableError(`${usage.unsealedHours} unsealed usage hours`);

    const lines = [
      ...prorate(periodStart, periodEnd, subPeriods, prices, ROUNDING),
      ...meteredLines(usage, rates, subPeriods),
      ...discountLines(discounts),
    ];
    const subtotal = lines.reduce((a, l) => a + l.amountMinor, 0n);
    const taxAmt   = roundMinor(subtotal * BigInt(tax.rateBp) / 10000n, ROUNDING);

    await insertLines(tx, invoiceId, lines);
    await tx.query(
      `UPDATE invoices SET state='issued', issued_at=now(),
              due_at = now() + interval '14 days',
              subtotal_minor=$2, tax_minor=$3, total_minor=$4,
              computation_inputs=$5
        WHERE id=$1 AND state='draft'`,
      [invoiceId, subtotal, taxAmt, subtotal + taxAmt,
       buildSnapshot({ subPeriods, prices, rates, usage, tax, discounts })]);

    // I7: the ledger and the invoice move together (case study 03)
    await postTransactionInTx(tx, {
      idempotencyKey: `invoice:${invoiceId}`,
      kind: 'invoice_issued', currency,
      legs: [
        { accountId: receivableAccount(accountId), amount:  subtotal + taxAmt },
        { accountId: REVENUE[currency],            amount: -subtotal },
        { accountId: TAX_PAYABLE[currency],        amount: -taxAmt },
      ],
    });

    await tx.query(`INSERT INTO outbox (topic, payload) VALUES ('invoice.issued',$1)`,
                   [JSON.stringify({ invoiceId })]);
    return { invoiceId };
  });
}
```

### 7.7 Backdating — the reconciler (pattern H)

```
 THE FLOW when support backdates a discount to 1 March:

 1. INSERT a discount row with valid = ['2026-03-01', ...), recorded_at = now().
    ★ Nothing existing is modified.

 2. The RECONCILER (nightly) asks, for each issued invoice:
       "If I recomputed this invoice TODAY, using current beliefs about
        that period, would I get the same number?"

 3. If not, it creates a DISCREPANCY row — it does NOT act automatically.

 4. A human (or a rule, for small amounts) approves → a CREDIT NOTE is
    issued, referencing the original invoice, with its own snapshot
    explaining the difference.

 5. The original invoice is byte-for-byte unchanged. The customer's
    balance reflects invoice − credit note. I1 holds. I2 holds for both.
```

```sql
-- The reconciler's core query: recompute vs stored
SELECT i.id, i.number, i.total_minor AS issued_total,
       r.total_minor AS recomputed_total,
       r.total_minor - i.total_minor AS delta
FROM invoices i
CROSS JOIN LATERAL recompute_invoice(i.account_id, i.period, now()) r
WHERE i.state IN ('issued','paid')
  AND i.issued_at > now() - interval '90 days'
  AND r.total_minor <> i.total_minor;
```

```sql
CREATE TABLE invoice_discrepancies (
  id           bigserial   PRIMARY KEY,
  invoice_id   bigint      NOT NULL REFERENCES invoices(id),
  detected_at  timestamptz NOT NULL DEFAULT now(),
  issued_total bigint      NOT NULL,
  recomputed_total bigint  NOT NULL,
  delta_minor  bigint      NOT NULL,
  cause        jsonb       NOT NULL,   -- which inputs changed, and how
  resolution   text        NULL,       -- 'credit_note','additional_invoice',
                                       -- 'no_action','under_threshold'
  resolved_by  text        NULL,
  credit_note_id bigint    NULL REFERENCES credit_notes(id)
);
```

**Note what this buys you:** the reconciler is also a *bug detector*. If a code change alters the billing algorithm, every historical invoice's recomputation drifts, and the discrepancy table fills up on the first nightly run. You find out in a day, not in an audit.

### 7.8 Usage ingest and rollups (patterns A, B)

```js
// Batched from Kafka. Idempotent by construction.
async function ingestUsage(batch) {          // 2,000 events
  await withTransaction(async (tx) => {
    await tx.query(
      `INSERT INTO usage_events (event_id, account_id, metric, quantity, occurred_at)
       SELECT (e->>'event_id')::uuid, (e->>'account_id')::bigint,
              e->>'metric', (e->>'quantity')::bigint, (e->>'occurred_at')::timestamptz
         FROM jsonb_array_elements($1::jsonb) e
       ON CONFLICT (event_id, occurred_at) DO NOTHING`,     -- ★ I3
      [JSON.stringify(batch)]);
  });
}
```

```sql
-- Rollup worker, every 5 minutes. Only unsealed hours.
INSERT INTO usage_rollup_hourly (account_id, metric, hour, quantity, event_count)
SELECT account_id, metric, date_trunc('hour', occurred_at), sum(quantity), count(*)
FROM usage_events
WHERE occurred_at >= $1 AND occurred_at < $2
GROUP BY 1,2,3
ON CONFLICT (account_id, metric, hour) DO UPDATE
  SET quantity = EXCLUDED.quantity, event_count = EXCLUDED.event_count
WHERE usage_rollup_hourly.sealed_at IS NULL;      -- ★ never touch a sealed hour

-- Seal hours once the late-arrival window has passed
UPDATE usage_rollup_hourly SET sealed_at = now()
WHERE sealed_at IS NULL AND hour < now() - interval '8 hours';
```

**`sealed_at` is the contract between ingest and billing.** The invoice run refuses to bill on unsealed hours; the rollup worker refuses to modify sealed ones. Late events after sealing go to a `late_usage` table and are billed in the *next* period, with a line item saying so.

---

## 8. THE NUMBERS

| Metric | Attempt 1 | Attempt 2 | Production |
|---|---|---|---|
| Invoice run (1,300 accounts) | 6.6 h ✗ | 41 min | **8 min** |
| Per-account invoice compute | 18.4 s | 1.9 s | **0.37 s** |
| Buffers read per invoice | 8.87M | 210k | **1,840** (rollups, not events) |
| Reproduce a 2-year-old invoice | **impossible** | partial | **exact, from one row** |
| Proration correctness | wrong by ₹18,000 | wrong | **exact to the second** |
| Duplicate usage billing | yes | fixed | fixed |
| Backdating breaks issued invoices | yes | yes | **impossible** |
| Overlapping subscription periods | undetectable | undetectable | **structurally impossible** |
| Billing-algorithm regressions | found by customers | found by customers | **found by the reconciler in 24 h** |
| `usage_events` growth | unbounded | unbounded | **partitioned, 90-day retention via DROP** |

---

## 9. FAILURE MODES AND WHAT YOU MONITOR

| Failure | What happens | Design response |
|---|---|---|
| **Usage arrives after sealing** | Would under-bill | Goes to `late_usage`; billed next period with an explicit line item. Alert if late volume > 0.1% |
| **Invoice run crashes mid-batch** | Partial invoicing | Per-account transactions + `idempotency_key`. Re-running the job is a no-op for completed accounts |
| **Two invoice runs overlap** | Double invoicing | `uq_invoice_idem` on `(account, period, run_date)` makes it impossible |
| **Price changed during a run** | Half the batch uses old prices | Every price lookup is `valid @> <period boundary>`, not `now()`. The run is deterministic regardless of when it executes |
| **Backdated change after payment** | Customer already paid the old amount | Credit note; refund via the ledger. Never edit |
| **Rounding drift** | Sum of lines ≠ total | `CHECK (total = subtotal + tax)` at the DB level; explicit `rounding: half_even` in the snapshot; a daily check that `sum(lines) = subtotal` |
| **Billing engine version bump** | Old invoices "recompute" differently | `computation_version` in the snapshot; the reconciler ignores discrepancies caused solely by a version change unless flagged |
| **Currency confusion** | Charging USD amounts in INR | `currency` on every row; `CHECK` that all lines match the invoice currency |
| **Tax rate changes retroactively** | Government does this | `tax_rates` is versioned identically. Reconciler proposes credit notes |
| **Clock skew across pods** | Wrong period boundaries | All `now()` from PostgreSQL. Period boundaries computed server-side |

**The five alerts:**

```sql
-- 1. Invoice/ledger disagreement (I7) — PAGE
SELECT count(*) FROM invoices i WHERE i.state IN ('issued','paid')
 AND NOT EXISTS (SELECT 1 FROM ledger_transactions t
                  WHERE t.idempotency_key = 'invoice:'||i.id AND t.state='posted');

-- 2. Line sum ≠ subtotal — PAGE
SELECT i.id FROM invoices i
JOIN LATERAL (SELECT sum(amount_minor) s FROM invoice_lines l WHERE l.invoice_id=i.id) x ON true
WHERE i.state<>'draft' AND x.s <> i.subtotal_minor;

-- 3. Discrepancy backlog — ticket
SELECT count(*), sum(abs(delta_minor)) FROM invoice_discrepancies WHERE resolution IS NULL;

-- 4. Unsealed hours blocking the run — page if it blocks the window
SELECT count(*) FROM usage_rollup_hourly WHERE sealed_at IS NULL AND hour < now() - interval '12 hours';

-- 5. Overlap check (belt and braces — the EXCLUDE constraint should make this impossible)
SELECT a.subscription_id FROM subscription_periods a JOIN subscription_periods b
  ON a.subscription_id=b.subscription_id AND a.id<b.id AND a.valid && b.valid
WHERE a.superseded_at IS NULL AND b.superseded_at IS NULL;
```

---

## 10. WHAT BREAKS AT 10×

**400,000 accounts, 400M usage events/day, 13,000 invoices/night.**

| Wall | Why | Next design |
|---|---|---|
| Invoice run window | 13,000 × 0.37 s = 80 min serial | Parallelise by account shard — invoices are perfectly independent. 16 workers → 5 min |
| `usage_events` at 400M/day | 146B rows/year | Daily partitions + compress sealed partitions + 30-day hot retention, older to object storage as Parquet (Topic 06) |
| Rollup worker | 400M rows/day to aggregate | Move the rollup into the stream: aggregate in Kafka Streams / Flink and write only hourly rollups to PostgreSQL. Raw events go to object storage, not the OLTP database |
| Reconciler | recomputing 13,000 invoices/night × 90 days | Only recompute invoices whose *inputs* changed — track which periods a backdated row touches and recompute just those |
| `computation_inputs` JSONB size | ~8 KB × 4.7M invoices/year = 38 GB/year | Fine — it TOASTs out of line (Topic 05) and is never read on the hot path. This is the right place to spend storage |
| Revenue report (pattern I) | 200M rows | Columnar rollup fed by CDC (Topic 76). Never on the primary |

---

## 11. THE SEVEN QUESTIONS

1. Define valid time and transaction time. Give the exact query that answers "what did we believe on 31 March about March?" and say why you can't answer it with one time axis.
2. Why does the invoice store `computation_inputs` as an embedded snapshot rather than just foreign keys to the price rows it used? Name three failure modes the snapshot survives.
3. Why is a backdated change *not* an update to the existing subscription period? What does `superseded_at` preserve that a range edit would destroy?
4. Explain the `EXCLUDE USING gist (subscription_id WITH =, valid WITH &&) WHERE (superseded_at IS NULL)` constraint. Why is the `WHERE` clause essential?
5. What does `sealed_at` on the rollup table protect against, in both directions?
6. Why does the invoice run refuse to bill on unsealed usage hours instead of billing what it has?
7. The reconciler finds a ₹340 discrepancy on a paid invoice. Walk through what happens next, and name the invariant that forbids the obvious shortcut.

---

## 12. TRANSFERABLE LESSONS

**1. Two time axes, not one.**
Any system where facts can be corrected after the fact — billing, HR, insurance, medical records, compliance — needs valid time *and* transaction time. One axis lets you say what is true; you need the second to say what you *knew*, which is what every audit and every dispute actually asks about. → Reappears in **17 (EHR bitemporal records)**, **16 (inventory as-of)**, **18 (parcel state history)**.

**2. Never store a reference where the referent can change; store a snapshot.**
`plan_price_id = 441` is a promise that row 441 still means what it meant. `{"seat_price_minor": 50000}` is a fact. For anything that must be reproducible years later, embed the value. This is deliberate denormalisation with a clear rule: **snapshots record computations, never cache current state.** → Reappears in **12 (order line prices)**, **03 (ledger metadata)**.

**3. Model periods, not states with timestamps.**
`status='active', changed_at=...` cannot express history, cannot be checked for gaps, and cannot be prorated. `valid tstzrange` + an exclusion constraint gives you history, structural non-overlap, and `&&`/`@>` operators that replace a whole category of off-by-one `BETWEEN` bugs. → Reappears in **02 (seat holds)**, **15 (tenant contracts)**, **17 (EHR)**.

**4. Corrections are new records. Always.**
Credit notes, reversing ledger entries, superseded periods — the pattern is identical: **append the correction, link it to the original, never mutate.** The moment you allow an edit, you lose the ability to explain any number. → Reappears in **03 (reversing entries)**, **08 (audit log)**.

**5. Seal your inputs before you compute on them.**
`sealed_at` creates an explicit boundary between "still arriving" and "final." Without it, the same computation run twice gives different answers and nobody knows which is right. Any batch process reading a stream needs this. → Reappears in **20 (analytics watermarks)**, **07 (telemetry)**.

**6. A reconciler that recomputes and compares is a bug detector, not just a correction tool.**
It catches backdated changes *and* code regressions *and* data corruption, on a daily cadence, before customers do. **Any system with a derivable output should recompute and diff it on a schedule.** → Reappears in **03 (daily ledger audit)**, **16 (stock reconciliation)**.

**7. Make the invariant checkable, then check it every day.**
Every invariant in section 3 has a query in section 9. Design makes violations hard; the daily check makes them *visible*. You need both, because the bug you didn't anticipate is the one that reaches production.

---

## FILES TO READ NEXT

- **05 — Social feed fan-out** (this folder): Tier 2 opens; a completely different problem shape
- **03 — Payment ledger** (this folder): re-read section 12 and compare lesson 4 here with lesson 1 there
- **26 — Modelling time, money and identity** (curriculum): the bitemporal pattern in full
- **24 — Constraints in depth** (curriculum): exclusion constraints and range operators
- **59 — Partitioning** (curriculum): the usage-events retention design
- **55 — Denormalisation patterns** (curriculum): why the snapshot is correct denormalisation
