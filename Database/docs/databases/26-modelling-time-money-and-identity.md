# 26 — Modelling Time, Money and Identity
## Phase: Database Design

---

## ELI5 — The Simple Analogy

Three things everyone assumes are simple, and none of them are.

**Time.** "The meeting is at 2pm." Whose 2pm? If you write it in a diary in Bengaluru and read it in London, you get two different moments. And if the government moves the clocks between now and then, the *same* diary entry means a different instant than it did when you wrote it.

**Money.** "₹0.10 plus ₹0.20." A computer using binary fractions gets ₹0.30000000000000004. Add it up a million times and your books don't balance — and nobody notices until an auditor does.

**Identity.** "Rahul Sharma." There are three of them in the building, one changed her surname last year, and one was typed with two spaces. Which one owes you money?

Each of these has a correct answer that takes one line of DDL, and a wrong answer that takes a quarter to unwind. This topic is those three lines and the reasoning behind them.

---

## Where this fits in the big picture

```
   25 data types (the byte-level menu)
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 26 TIME · MONEY · IDENTITY               │ ← YOU ARE HERE
        │ the four domains that ruin schemas       │
        └────────────────────┬─────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        27 antipatterns  28 migrations  Phase 4
        (soft deletes,   (fixing these  (temporal data
         audit columns)   in production)  and normalisation)
```

Topic 25 gave you the type menu. **This topic covers the four domains where choosing wrong is a *correctness* bug rather than a performance one** — and adds the fourth: temporal data (history), which is where time and identity collide.

---

## What is this?

Four modelling domains, each with a canonical correct answer and a widespread wrong one:

| Domain | Wrong | Right |
|---|---|---|
| **Time** | `timestamp`, local time, `text` | `timestamptz`, UTC, IANA zone names |
| **Money** | `float`, `money`, `numeric` on hot paths | `bigint` in minor units + a currency code |
| **Identity** | natural keys, "email is unique" | surrogate PK + `UNIQUE`, and know what a person *is* |
| **History** | `UPDATE` and lose the past | validity ranges + transaction time |

---

## Why does it matter for a backend developer?

Because these four produce the bugs that are hardest to detect and most expensive to fix:

```
 ① A REPORT WRONG BY HOURS
    Two services write `timestamp` (no zone). One runs in UTC, one in
    IST. "Sales on 15 March" is off by 5.5 hours of data, and you
    CANNOT tell which rows are affected — the information is gone.

 ② A LEDGER THAT DRIFTS
    float money. Sum a million transactions; you're off by fractions of
    a paisa. Reconciliation fails. Nobody can say when it started.

 ③ TWO ACCOUNTS FOR ONE PERSON
    "Email is the identity." Then someone signs up with a plus-address,
    or their company migrates domains, and now they have two accounts
    with two order histories and one very annoyed support ticket.

 ④ "WHAT DID WE CHARGE THEM IN MARCH?"
    You UPDATEd the price. March's answer is gone. (Case study 04.)
```

All four are one-line fixes at design time. All four are multi-week migrations afterwards, and ① is sometimes **unrecoverable** — you cannot retroactively determine which zone a naked timestamp meant.

---

## The physical reality

### `timestamp` vs `timestamptz` — the same 8 bytes, completely different meaning

```
 BOTH ARE 8 BYTES. Microseconds since 2000-01-01. Identical storage.
 The difference is entirely in INTERPRETATION.

 timestamp (WITHOUT TIME ZONE)
   Stores the literal wall-clock reading you gave it. No zone is
   recorded. It is a STRING OF DIGITS with a date-shaped type.
   '2026-03-15 14:00' means 14:00 — somewhere. Nobody knows where.

 timestamptz (WITH TIME ZONE)
   ★ Converts the input TO UTC using the session's TimeZone, stores UTC,
     and converts BACK to the session's TimeZone on output.
   ★ The value stored is an absolute INSTANT.
   ⚠ It does NOT store a time zone. The name is misleading. It stores
     an instant; the zone is used only for conversion at the boundary.

 ┌───────────────────────────────────────────────────────────────────┐
 │  SET TimeZone = 'Asia/Kolkata';                                   │
 │  INSERT: '2026-03-15 14:00'                                       │
 │    timestamp   → stored as 2026-03-15 14:00   (no conversion)     │
 │    timestamptz → stored as 2026-03-15 08:30Z  (converted to UTC)  │
 │                                                                   │
 │  SET TimeZone = 'UTC';    SELECT …                                │
 │    timestamp   → 2026-03-15 14:00   ⚠ SAME NUMBER, DIFFERENT      │
 │                                        INSTANT than intended       │
 │    timestamptz → 2026-03-15 08:30   ✓ the same instant, rendered  │
 │                                        in the reader's zone        │
 └───────────────────────────────────────────────────────────────────┘

 ★ THE UNRECOVERABLE PART: if two services with different TimeZone
   settings both wrote `timestamp`, the rows are now mixed and there is
   NO COLUMN that tells you which is which. The information was never
   stored. This is why it's a correctness bug, not a formatting one.
```

### Money — three representations, one right answer

```
 ₹2,499.00

 float8              4C 3D 88 A0 00 00 00 00   ⚠ cannot represent 0.1
 numeric(12,2)       variable, ~10 B, base-10000 digit array, software math
 bigint (paise)      00 00 00 00 00 03 D0 84   = 249900. Exact. 8 B. Hardware math.

 MEASURED (Topic 25): summing 10M values
   float8  388 ms   ⚠ result: 100000.00000018848
   numeric 2,104 ms  ✓ exact, 5.4× slower
   bigint  412 ms    ★ exact AND fast

 ⚠ AND MONEY IS NEVER JUST A NUMBER:
   amount_minor  bigint  NOT NULL     -- 249900
   currency      char(3) NOT NULL     -- 'INR'  (ISO 4217)
   ⇒ an amount without a currency is meaningless, and two amounts in
     different currencies must never be summed. Make the currency
     NOT NULL and CHECK that all legs of a transaction share it
     (case study 03's I6).

 ⚠ MINOR UNITS ARE NOT ALWAYS 100:
   JPY, KRW  → 0 decimal places (1 yen = 1 minor unit)
   BHD, KWD  → 3 decimal places (1 dinar = 1000 fils)
   ⇒ store the EXPONENT in your currency table; never hardcode /100.
```

### Identity — three different things people conflate

```
 ① THE ROW'S IDENTITY      — the primary key (Topic 22)
                              immutable, meaningless, internal
 ② THE BUSINESS IDENTIFIER — email, phone, employee number
                              may change, may be reassigned, UNIQUE
 ③ THE REAL-WORLD ENTITY   — the actual human being
                              ★ has no reliable digital representation

 THE HARD TRUTH ABOUT ③:
   • emails get reassigned when a person leaves a company
   • phone numbers get recycled by carriers within months
   • names change, and are not unique
   • national IDs are not universal and are often illegal to store
   • the same person legitimately has multiple accounts
   ⇒ YOU CANNOT KEY ON A PERSON. You can only key on an ACCOUNT,
     and optionally maintain a separate, fuzzy "these accounts are
     probably the same person" linkage that is never a foreign key.
```

---

## How it works — step by step

### Time: the five rules

```
 RULE 1 — ALWAYS timestamptz. Never timestamp.
   The only legitimate use of `timestamp` is a wall-clock time with no
   instant attached: "the shop opens at 09:00" (which is 09:00 in
   whatever zone the shop is in). Even then, `time` is usually better.

 RULE 2 — STORE INSTANTS IN UTC; CONVERT AT THE EDGE.
   SET TimeZone = 'UTC' on the server AND in your connection string.
   Your application converts for display. The database never guesses.

 RULE 3 — FOR FUTURE EVENTS, STORE THE ZONE NAME TOO.
   ★ This is the rule people miss, and it matters.
   A meeting "at 14:00 Asia/Kolkata on 2027-03-15" is NOT the same as
   an instant, because a government can change the offset before then.
   ⇒ store BOTH:
        scheduled_at    timestamptz NOT NULL   -- best-known instant
        scheduled_zone  text        NOT NULL   -- 'Asia/Kolkata'
        scheduled_local timestamp   NOT NULL   -- 14:00, the human intent
     and recompute `scheduled_at` if the tz database updates.
   ⇒ For PAST events, the instant alone is sufficient and correct.

 RULE 4 — USE IANA NAMES, NEVER OFFSETS OR ABBREVIATIONS.
   'Asia/Kolkata'  ✓   handles DST and historical offset changes
   '+05:30'        ✗   loses DST; wrong half the year in most of the world
   'IST'           ✗✗  ambiguous: India, Ireland, and Israel all use it

 RULE 5 — DATES ARE NOT TIMESTAMPS.
   A birthday is a `date`. It has no instant and no zone. Storing it as
   timestamptz means it shifts by a day for users in some zones —
   a real, recurring bug.
```

```sql
-- The canonical shapes
CREATE TABLE orders (
  placed_at   timestamptz NOT NULL DEFAULT now()      -- ✓ a past instant
);

CREATE TABLE appointments (
  starts_at        timestamptz NOT NULL,              -- best-known instant
  starts_local     timestamp   NOT NULL,              -- the human intent
  timezone         text        NOT NULL               -- 'Asia/Kolkata'
    CHECK (timezone IN (SELECT name FROM pg_timezone_names))
);

CREATE TABLE users (
  date_of_birth date NOT NULL                         -- ✓ a date, not an instant
);

CREATE TABLE store_hours (
  opens_at  time NOT NULL,                            -- ✓ wall clock, no date
  closes_at time NOT NULL
);
```

### Money: the canonical shape

```sql
CREATE TABLE currencies (
  code     char(3) PRIMARY KEY,                       -- ISO 4217
  exponent smallint NOT NULL CHECK (exponent BETWEEN 0 AND 4),
  name     text NOT NULL
);
INSERT INTO currencies VALUES ('INR',2,'Indian Rupee'), ('JPY',0,'Yen'),
                              ('KWD',3,'Kuwaiti Dinar'), ('USD',2,'US Dollar');

CREATE TABLE payments (
  id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  amount_minor bigint  NOT NULL CHECK (amount_minor > 0),  -- ★ integer
  currency     char(3) NOT NULL REFERENCES currencies(code),
  -- and if you ever convert:
  fx_rate_ppm  bigint  NULL,        -- parts per million, integer again
  base_minor   bigint  NULL         -- the converted amount, also stored
);
```

```js
// Format at the edge — a single helper, used everywhere.
function formatMinor(amountMinor, currency, exponent) {
  const s = String(Math.abs(amountMinor)).padStart(exponent + 1, '0');
  const whole = s.slice(0, s.length - exponent) || '0';
  const frac  = exponent ? '.' + s.slice(-exponent) : '';
  return new Intl.NumberFormat('en-IN', { style: 'currency', currency })
    .format(Number(`${amountMinor < 0 ? '-' : ''}${whole}${frac}`));
}
formatMinor(249900, 'INR', 2);   // "₹2,499.00"
formatMinor(2499,   'JPY', 0);   // "¥2,499"
```

```
 ⚠ THE THREE MONEY RULES BEYOND THE TYPE:
   ① NEVER SUM ACROSS CURRENCIES. Enforce it:
        CHECK on a transaction that all legs share a currency
   ② ROUNDING IS A DECISION, NOT A DEFAULT.
        Write it down: half-even (banker's) is standard for finance.
        Store the rule in the computation record (case study 04).
   ③ FX RATES ARE INTEGERS TOO. parts-per-million, not float.
        And store the RATE USED, not just the result — you need to
        explain the number later.
```

### Identity: what to key on, and what not to

```sql
CREATE TABLE users (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- ★ ① identity
  email      citext NOT NULL,                                   -- ② business id
  email_verified_at timestamptz NULL,
  deleted_at timestamptz NULL,
  created_at timestamptz NOT NULL DEFAULT now(),

  -- ★ unique among LIVE users, case-insensitively (Topic 13)
  CONSTRAINT ck_email_shape CHECK (email ~ '^[^@\s]+@[^@\s]+\.[^@\s]+$')
);
CREATE UNIQUE INDEX uq_users_email_live ON users (email) WHERE deleted_at IS NULL;
```

```
 WHY EACH PIECE:
   citext            case-insensitive comparison, so Arjun@x.in and
                     arjun@x.in are the same. (Or index lower(email).)
   ★ PARTIAL UNIQUE   a deleted user frees their address for reuse
   email_verified_at  ⚠ an UNVERIFIED email is not an identity. Until
                     it's verified it's a claim, and treating it as an
                     identifier is how account-takeover happens.
   NO natural PK      email changes; the PK must not.

 ⚠ WHAT ABOUT "THE SAME PERSON WITH TWO ACCOUNTS"?
   Do NOT try to enforce it with a constraint — you cannot. Model it
   explicitly as a SEPARATE, FUZZY linkage:

   CREATE TABLE identity_links (
     user_id       bigint NOT NULL REFERENCES users(id),
     linked_user_id bigint NOT NULL REFERENCES users(id),
     confidence    smallint NOT NULL CHECK (confidence BETWEEN 0 AND 100),
     method        text NOT NULL,        -- 'same_phone','manual_merge',…
     linked_at     timestamptz NOT NULL DEFAULT now(),
     CHECK (user_id < linked_user_id),   -- one row per pair
     PRIMARY KEY (user_id, linked_user_id)
   );
   ★ It is a HYPOTHESIS with a confidence, never a foreign key that
     other tables depend on. Merging accounts is a deliberate,
     audited operation — never an inference.
```

### History: the fourth domain, where time and identity collide

```
 THE QUESTION: "what was the price on 2026-03-15?"

 ① NO HISTORY (the default, and usually wrong for anything auditable)
    products(id, price_minor)
    UPDATE products SET price_minor = 250000 WHERE id = 88;
    ⇒ March's answer is GONE. Unrecoverable.

 ② VALIDITY RANGES (valid time) — "when was this true?"
    CREATE TABLE product_prices (
      product_id bigint NOT NULL REFERENCES products(id),
      price_minor bigint NOT NULL,
      currency char(3) NOT NULL,
      valid tstzrange NOT NULL,
      EXCLUDE USING gist (product_id WITH =, currency WITH =, valid WITH &&)
    );
    SELECT price_minor FROM product_prices
     WHERE product_id=88 AND currency='INR' AND valid @> '2026-03-15'::timestamptz;
    ✓ answers "what was true then"
    ✗ cannot answer "what did we BELIEVE then" — a backdated correction
      silently rewrites the past

 ③ BITEMPORAL (valid time + transaction time) — the full answer
    ... plus  recorded_at timestamptz NOT NULL DEFAULT now(),
              superseded_at timestamptz NULL
    ✓ "what is true about March?"     → valid @> March, superseded_at IS NULL
    ✓ "what did we BELIEVE on 31 Mar?" → valid @> March
                                          AND recorded_at <= '2026-03-31'
                                          AND (superseded_at IS NULL
                                               OR superseded_at > '2026-03-31')
    ★ The second query is what an auditor actually asks, and it is
      unanswerable without the second axis. (Case study 04.)

 ④ AUDIT LOG — a separate append-only record of every change
    ⇒ complements ②/③; it answers "who changed it and when", not
      "what was the value". Different question, different table.

 ⇒ CHOOSE BY WHAT YOU MUST ANSWER:
     nothing historical needed        → ① (most tables)
     "what was true then"             → ②
     "what did we believe then"       → ③  (money, billing, compliance)
     "who changed it"                 → ④, alongside any of the above
```

---

## Concept breakdown

```
TIME
├── timestamptz   an INSTANT, stored as UTC. ★ the default for everything
├── timestamp     a wall-clock reading with no zone. ⚠ almost always wrong
├── date          a calendar day, no instant, no zone (birthdays, holidays)
├── time          a wall-clock time of day (opening hours)
├── interval      a duration. ⚠ '1 month' is not a fixed length
└── tstzrange     a span of instants. ★ the basis of all temporal modelling
    ALWAYS '[)' bounds — half-open makes adjacency work

MONEY
├── amount_minor  bigint, in the smallest unit. Exact, fast, 8 bytes.
├── currency      char(3), ISO 4217, NOT NULL, FK to a currency table
├── exponent      per currency — NOT always 2 (JPY 0, KWD 3)
├── rounding      an explicit, recorded decision (half-even for finance)
└── fx rates      integers (parts per million), and store the rate used

IDENTITY — three distinct things
├── ① row identity      the PK. immutable, meaningless, internal
├── ② business id       email/phone/employee no. UNIQUE, but MUTABLE
└── ③ the real person   ★ has no reliable digital key. Model links as
                          a fuzzy hypothesis, never as a constraint.

HISTORY — four levels
├── ① none                     UPDATE in place
├── ② valid time               tstzrange + EXCLUDE
├── ③ bitemporal               + recorded_at / superseded_at
└── ④ audit log                who changed what, when — a different question

THE FOUR RULES YOU CANNOT NEGOTIATE
  1. timestamptz. Always.
  2. bigint minor units + a currency code. Always.
  3. Surrogate PK; business identifiers are UNIQUE, not keys.
  4. If someone will ask "what was it then?", you need validity ranges
     — and the decision must be made BEFORE the first UPDATE.
```

---

## Diagrams

**Diagram 1 — big picture: the four domains and their failure modes**

```
 ┌─────────────┬──────────────────────┬───────────────────────────────┐
 │ DOMAIN      │ WRONG                │ FAILURE MODE                  │
 ├─────────────┼──────────────────────┼───────────────────────────────┤
 │ TIME        │ timestamp            │ reports wrong by hours;       │
 │             │ local time           │ ★ UNRECOVERABLE — the zone    │
 │             │ '+05:30' offsets     │   was never stored            │
 ├─────────────┼──────────────────────┼───────────────────────────────┤
 │ MONEY       │ float                │ ledger drifts, silently, for  │
 │             │ money type           │ months. Reconciliation fails. │
 │             │ no currency column   │ ₹ summed with $               │
 ├─────────────┼──────────────────────┼───────────────────────────────┤
 │ IDENTITY    │ email as PK          │ email change = cascade to 14  │
 │             │ unverified email     │ tables; account takeover      │
 │             │ as identity          │                               │
 ├─────────────┼──────────────────────┼───────────────────────────────┤
 │ HISTORY     │ UPDATE in place      │ "what was the price in March?"│
 │             │                      │ ★ the answer no longer exists │
 └─────────────┴──────────────────────┴───────────────────────────────┘
```

**Diagram 2 — data flow: what `timestamptz` actually does**

```
  APPLICATION                POSTGRESQL                    DISK
  (Asia/Kolkata)             SET TimeZone='UTC'
       │
  '2026-03-15 14:00'
       │
       ├─ timestamptz ──▶ parse with client TimeZone ──▶ 2026-03-15T08:30Z
       │                  (or the session default)         ★ an INSTANT
       │
       └─ timestamp ────▶ ⚠ store the digits verbatim ──▶ 2026-03-15 14:00
                            NO conversion, NO zone            ★ ambiguous

  READING BACK (session TimeZone = 'America/New_York'):
       timestamptz ──▶ 2026-03-15 04:30-04:00   ✓ same instant, local render
       timestamp   ──▶ 2026-03-15 14:00         ⚠ a DIFFERENT instant now
```

**Diagram 3 — before/after: the two time axes**

```
                 transaction time (when we recorded it) ──▶
   valid    ┌──────────────────────────────────────────────────┐
   time     │  1 Mar          31 Mar           2 Apr           │
     │      │                                                   │
     ▼      │  price=₹2000    price=₹2000                       │
   MARCH    │  (recorded      (still believed)                  │
            │   1 Jan)                                          │
            │──────────────────────┼───────────────────────────│
            │                      │  price=₹1800               │
   APRIL    │                      │  (recorded 2 Apr, VALID    │
            │                      │   FROM 1 Mar — BACKDATED)  │
            └──────────────────────┴───────────────────────────┘
                                   ↑
   "What is true about March?"     → ₹1800   (current belief)
   "What did we believe on 31 Mar?"→ ₹2000   (the invoice we sent)
   ★ BOTH are correct answers to DIFFERENT questions. One time axis
     can only answer one of them.
```

---

## Example 1 — basic

**Step 1 — the timestamp trap, live.**

```sql
SHOW TimeZone;                                     -- UTC (set it explicitly!)

CREATE TABLE t_demo (
  naive timestamp   NOT NULL,
  aware timestamptz NOT NULL
);

SET TimeZone = 'Asia/Kolkata';
INSERT INTO t_demo VALUES ('2026-03-15 14:00', '2026-03-15 14:00');
SELECT * FROM t_demo;
```
```
        naive        |          aware
---------------------+-------------------------
 2026-03-15 14:00:00 | 2026-03-15 14:00:00+05:30
```
```sql
SET TimeZone = 'America/New_York';
SELECT * FROM t_demo;
```
```
        naive        |          aware
---------------------+-------------------------
 2026-03-15 14:00:00 | 2026-03-15 04:30:00-04:00
```
**The `aware` value rendered as the same instant in a different zone — correct. The `naive` value is still "14:00", which is now a completely different moment.** And there is no column that tells you it was meant to be IST.

```sql
SET TimeZone = 'UTC';
SELECT aware AT TIME ZONE 'UTC' AS stored_utc FROM t_demo;
```
```
     stored_utc
---------------------
 2026-03-15 08:30:00        ← what's actually on disk
```

**Step 2 — DST, and why offsets are wrong.**

```sql
SET TimeZone = 'Europe/London';
SELECT '2026-01-15 12:00'::timestamptz AS winter,
       '2026-07-15 12:00'::timestamptz AS summer;
```
```
         winter          |         summer
-------------------------+-------------------------
 2026-01-15 12:00:00+00  | 2026-07-15 12:00:00+01
```
**The same wall-clock time is a different offset in January and July.** Storing `'+00:00'` as "the London zone" is wrong for half the year.

```sql
-- and abbreviations are ambiguous
SELECT count(*) FROM pg_timezone_abbrevs WHERE abbrev = 'IST';
SELECT abbrev, utc_offset FROM pg_timezone_abbrevs WHERE abbrev IN ('IST','CST','BST');
```
```
 abbrev | utc_offset
--------+------------
 IST    | 05:30:00       ← India… or Ireland (+01), or Israel (+02)
 CST    | -06:00:00      ← US Central… or China (+08)
```

**Step 3 — a date is not a timestamp.**

```sql
CREATE TABLE bd (name text, dob_date date, dob_ts timestamptz);
SET TimeZone = 'Asia/Kolkata';
INSERT INTO bd VALUES ('Arjun','1995-03-15','1995-03-15 00:00');
SET TimeZone = 'America/Los_Angeles';
SELECT name, dob_date, dob_ts::date AS dob_from_ts FROM bd;
```
```
 name  |  dob_date  | dob_from_ts
-------+------------+-------------
 Arjun | 1995-03-15 | 1995-03-14      ← ★ off by a day
```
**A birthday stored as `timestamptz` shifts by a day for users in western zones.** It's a `date`.

**Step 4 — money.**

```sql
SELECT 0.1::float8 + 0.2::float8 AS f,
       0.1::numeric + 0.2::numeric AS n,
       (10 + 20)::bigint AS paise;
```
```
          f          |  n  | paise
---------------------+-----+-------
 0.30000000000000004 | 0.3 |    30
```
```sql
-- accumulate a million times
CREATE TABLE acc (f float8, n numeric(14,2), p bigint);
INSERT INTO acc SELECT 0.01, 0.01, 1 FROM generate_series(1,1000000);
SELECT sum(f) AS float_sum, sum(n) AS numeric_sum, sum(p)/100.0 AS bigint_sum FROM acc;
```
```
     float_sum      | numeric_sum | bigint_sum
--------------------+-------------+------------
 10000.000000188848 |    10000.00 |   10000.00
                ↑ ★ drift, after only a million rows
```

**Step 5 — non-100 minor units.**

```sql
CREATE TABLE currencies (code char(3) PRIMARY KEY, exponent smallint NOT NULL);
INSERT INTO currencies VALUES ('INR',2),('USD',2),('JPY',0),('KWD',3);

-- ¥2,499 and ₹2,499.00 and KD 2.499 — three different minor-unit counts
SELECT code, exponent, 2499 AS minor,
       (2499::numeric / power(10, exponent))::text AS displayed
FROM currencies;
```
```
 code | exponent | minor | displayed
------+----------+-------+-----------
 INR  |        2 |  2499 | 24.99
 JPY  |        0 |  2499 | 2499
 KWD  |        3 |  2499 | 2.499
```
**Hardcoding `/100` is wrong for two of these four.**

**Step 6 — identity.**

```sql
CREATE EXTENSION IF NOT EXISTS citext;
CREATE TABLE users (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email citext NOT NULL,
  email_verified_at timestamptz NULL,
  deleted_at timestamptz NULL
);
CREATE UNIQUE INDEX uq_users_email_live ON users (email) WHERE deleted_at IS NULL;

INSERT INTO users (email) VALUES ('Arjun@Shop.IN');
INSERT INTO users (email) VALUES ('arjun@shop.in');
-- ERROR: duplicate key value violates unique constraint "uq_users_email_live"
--   ★ citext made the comparison case-insensitive

UPDATE users SET deleted_at = now() WHERE email = 'arjun@shop.in';
INSERT INTO users (email) VALUES ('arjun@shop.in');   -- ✓ the address is free again
```

**Step 7 — history with validity ranges.**

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE TABLE product_prices (
  product_id  bigint    NOT NULL,
  currency    char(3)   NOT NULL,
  price_minor bigint    NOT NULL CHECK (price_minor >= 0),
  valid       tstzrange NOT NULL,
  recorded_at timestamptz NOT NULL DEFAULT now(),
  EXCLUDE USING gist (product_id WITH =, currency WITH =, valid WITH &&)
);

INSERT INTO product_prices VALUES
  (88,'INR',200000,'[2025-01-01, 2026-06-01)'),
  (88,'INR',250000,'[2026-06-01, infinity)');

-- what was the price on 15 March 2026?
SELECT price_minor FROM product_prices
WHERE product_id=88 AND currency='INR' AND valid @> '2026-03-15'::timestamptz;
```
```
 price_minor
-------------
      200000        ← ★ still answerable, forever
```
```sql
-- and overlaps are impossible
INSERT INTO product_prices VALUES (88,'INR',220000,'[2026-03-01, 2026-09-01)');
-- ERROR: conflicting key value violates exclusion constraint
```

---

## Example 2 — production scenario

**The situation.** A payments platform, 4 years old, operating in 6 countries. Three incidents in one month:

1. **Finance:** "the March India revenue report differs by ₹4.2 lakh depending on who runs it."
2. **Reconciliation:** the ledger is off by ₹18.44 against the bank, and has been drifting for ~14 months.
3. **Support:** a customer has two accounts with split order history and cannot merge them.

**Step 1 — the time bug.**

```sql
SELECT c.relname, a.attname, t.typname
FROM pg_attribute a JOIN pg_class c ON c.oid=a.attrelid
JOIN pg_type t ON t.oid=a.atttypid
WHERE t.typname = 'timestamp' AND c.relkind='r'
  AND c.relnamespace='public'::regnamespace AND a.attnum>0;
```
```
   relname   |   attname    | typname
-------------+--------------+-----------
 payments    | captured_at  | timestamp     ⚠
 payments    | settled_at   | timestamp     ⚠
 orders      | placed_at    | timestamp     ⚠
 refunds     | issued_at    | timestamp     ⚠
```

```sql
-- which services wrote these, and in which zone?
SELECT date_trunc('hour', captured_at) AS h, count(*)
FROM payments WHERE captured_at::date = '2026-03-15' GROUP BY 1 ORDER BY 1;
```
```
          h          | count
---------------------+-------
 2026-03-15 00:00:00 |  1204
 2026-03-15 01:00:00 |   882
 ...
 2026-03-15 09:00:00 |  8412     ← two peaks, 5.5 h apart
 ...
 2026-03-15 14:30:00 |  8104     ← ★ the same daily peak, in IST
```
**Two distributions, offset by exactly 5.5 hours.** The India service ran with `TimeZone=Asia/Kolkata`; the gateway webhook consumer ran in UTC. Both wrote `timestamp`.

```
 ★ CAN YOU FIX THE HISTORY?
   PARTIALLY, and only with external evidence:
     • the gateway's own API has the authoritative timestamps → backfill
       from there for payments (14M rows, feasible)
     • `orders.placed_at` has no external source. You must INFER the
       writer from another column (which pod, which region) and hope
       the mapping was stable.
   ⇒ For rows where you cannot determine the writer, the instant is
     UNRECOVERABLE. Document the affected range. This is why it is a
     correctness bug and not a formatting one.
```

**Step 2 — the money bug.**

```sql
SELECT c.relname, a.attname, t.typname
FROM pg_attribute a JOIN pg_class c ON c.oid=a.attrelid JOIN pg_type t ON t.oid=a.atttypid
WHERE t.typname IN ('float4','float8','money') AND c.relkind='r'
  AND a.attname ~ '(amount|price|total|fee|balance)';
```
```
   relname   |   attname   | typname
-------------+-------------+---------
 payments    | amount      | float8    ⚠
 refunds     | amount      | float8    ⚠
 fees        | fee_amount  | float8    ⚠
```
```sql
-- quantify the drift
SELECT sum(amount) AS float_sum,
       sum(round(amount::numeric, 2)) AS rounded_sum,
       sum(amount) - sum(round(amount::numeric,2)) AS drift
FROM payments WHERE captured_at >= '2025-01-01';
```
```
    float_sum     |  rounded_sum  |        drift
------------------+---------------+----------------------
 4821904412.44018 | 4821904412.44 | 0.00018400000035762787
```
Small per-query — but the *reconciliation* compares running totals computed in different orders, and floating-point addition is not associative. **That is where the ₹18.44 came from: the same numbers summed in a different order give a different answer.**

```sql
-- prove it
SELECT (SELECT sum(amount) FROM payments ORDER BY id) AS by_id,
       (SELECT sum(amount) FROM payments ORDER BY random()) AS by_random;
-- these differ in the last digits
```

**Step 3 — the identity bug.**

```sql
SELECT lower(email), count(*) FROM users GROUP BY 1 HAVING count(*) > 1 LIMIT 5;
```
```
      lower       | count
------------------+-------
 arjun@shop.in    |     2      ← 'Arjun@Shop.IN' and 'arjun@shop.in'
 meera@example.in |     2
```
`email` is `text` with a plain `UNIQUE` — case-sensitive. **1,842 duplicate-by-case accounts.** And merging them is hard because both have orders, payments, and a ledger balance.

**Step 4 — the fixes, in dependency order.**

```sql
-- ═══ ① TIME — the most urgent, because new rows keep making it worse ═══
-- Stop the bleeding first: pin the zone everywhere.
ALTER DATABASE payments SET TimeZone = 'UTC';
ALTER ROLE app SET TimeZone = 'UTC';
-- and in the connection string: options='-c TimeZone=UTC'

-- Add the correct columns alongside; do NOT convert in place yet.
ALTER TABLE payments ADD COLUMN captured_at_tz timestamptz NULL;

-- Backfill from the authoritative source where one exists.
UPDATE payments p SET captured_at_tz = g.captured_at
FROM gateway_events g WHERE g.payment_ref = p.gateway_ref;

-- For the rest, infer the writer's zone from a recorded attribute.
UPDATE payments SET captured_at_tz =
  CASE written_by_region
    WHEN 'in' THEN captured_at AT TIME ZONE 'Asia/Kolkata'
    ELSE            captured_at AT TIME ZONE 'UTC'
  END
WHERE captured_at_tz IS NULL AND written_by_region IS NOT NULL;

-- ★ And record what you could NOT determine. This is honest engineering.
CREATE TABLE data_quality_notes (
  table_name text, column_name text, issue text,
  affected_range tstzrange, affected_rows bigint, noted_at timestamptz DEFAULT now()
);
INSERT INTO data_quality_notes VALUES
 ('payments','captured_at','zone indeterminate; ±5.5h uncertainty',
  '[2022-04-01,2023-11-14)', 412008);

-- ═══ ② MONEY ═══
ALTER TABLE payments ADD COLUMN amount_minor bigint NULL,
                     ADD COLUMN currency char(3) NULL;
UPDATE payments SET
  amount_minor = round(amount::numeric * 100)::bigint,   -- ⚠ a ROUNDING DECISION
  currency = 'INR'
WHERE amount_minor IS NULL;
-- ⚠ DOCUMENT THE RULE. Some float values are 0.30000000000000004;
--   you are choosing half-up. Reconcile the result against the bank
--   statement BEFORE cutting over (case study 03's daily audit).

ALTER TABLE payments ALTER COLUMN amount_minor SET NOT NULL,
                     ALTER COLUMN currency SET NOT NULL,
                     ADD CONSTRAINT ck_amount_pos CHECK (amount_minor > 0);

-- ═══ ③ IDENTITY ═══
-- 1,842 case-duplicate pairs. NOT a bulk merge — accounts have money.
CREATE TABLE account_merge_queue AS
SELECT lower(email) AS email_lower, array_agg(id ORDER BY created_at) AS user_ids
FROM users GROUP BY 1 HAVING count(*) > 1;
-- a human (or a rule, for zero-balance accounts) resolves each.

-- Then make it impossible to recur:
ALTER TABLE users ALTER COLUMN email TYPE citext;
DROP INDEX uq_users_email;
CREATE UNIQUE INDEX CONCURRENTLY uq_users_email_live
  ON users (email) WHERE deleted_at IS NULL;
```

**Step 5 — prevention.**

```sql
-- A CI check, run against a schema dump
SELECT c.relname, a.attname, t.typname, 'FORBIDDEN TYPE' AS reason
FROM pg_attribute a JOIN pg_class c ON c.oid=a.attrelid JOIN pg_type t ON t.oid=a.atttypid
WHERE c.relkind='r' AND a.attnum>0 AND NOT a.attisdropped
  AND c.relnamespace='public'::regnamespace
  AND (t.typname IN ('timestamp','money','float4','float8')
       AND NOT (t.typname LIKE 'float%' AND a.attname ~ '(lat|lon|score|ratio|pct)'));
-- fail the build if this returns any row
```

**Results:**

| | Before | After |
|---|---|---|
| March report variance | ±₹4.2 lakh | **0** |
| Ledger drift | ₹18.44, growing | **0, exact integers** |
| Duplicate accounts | 1,842 | 0, and impossible |
| Rows with unrecoverable timestamps | — | **412,008, documented** |

**That last row is the honest one.** Some of the damage could not be undone, and the correct engineering response is to quantify and document it rather than pretend otherwise.

---

## Common mistakes

**1. `timestamp` instead of `timestamptz`.**
- *Symptom:* reports differ by hours; two services disagree about "today."
- *Engine-level why:* no zone is stored, so the value is a bare wall-clock reading. Two writers in different zones produce indistinguishable rows.
- *Fix:* `timestamptz` everywhere; pin `TimeZone=UTC` on the database, the role, and the connection string. **Adding it later may be unrecoverable.**

**2. Storing an offset instead of a zone name for future events.**
- *Symptom:* recurring meetings drift by an hour after a DST transition.
- *Fix:* store the IANA name (`Asia/Kolkata`) alongside the instant, and recompute when the tz database updates.

**3. A birthday as `timestamptz`.**
- *Symptom:* users in western zones see their birthday a day early.
- *Fix:* `date`. It has no instant.

**4. `float` for money.**
- *Symptom:* drift that appears months later and cannot be traced.
- *Engine-level why:* binary floating point cannot represent 0.1; and addition is not associative, so the same rows summed in a different order give a different total.
- *Fix:* `bigint` in minor units.

**5. Hardcoding `/100`.**
- *Symptom:* JPY amounts 100× too small; KWD 10× too large.
- *Fix:* an `exponent` column on a currency table.

**6. Summing across currencies.**
- *Symptom:* a "total revenue" figure that adds ₹ to $.
- *Fix:* `currency NOT NULL` on every amount, and a `CHECK` that all legs of a transaction share it.

**7. Treating an unverified email as an identity.**
- *Symptom:* account takeover by registering a victim's address before they do.
- *Fix:* `email_verified_at`. An unverified address is a claim, not an identifier.

**8. `UPDATE`ing a value someone will ask about historically.**
- *Symptom:* "what did we charge in March?" has no answer.
- *Fix:* decide *before the first UPDATE*. Validity ranges are cheap to add on day one and a migration afterwards.

---

## Hands-on proof

**PROVE IT #1 — the timestamp trap.** (Example 1, step 1.)
**PROVE IT #2 — DST and ambiguous abbreviations.** (Example 1, step 2.)
**PROVE IT #3 — a date shifts if stored as a timestamp.** (Example 1, step 3.)
**PROVE IT #4 — float drift.** (Example 1, step 4.)

**PROVE IT #5 — float addition is not associative.**
```sql
SELECT (0.1::float8 + 0.2::float8) + 0.3::float8 = 0.1::float8 + (0.2::float8 + 0.3::float8);
-- f      ★ the same numbers, different grouping, different result
```

**PROVE IT #6 — validity ranges answer historical questions.** (Example 1, step 7.)

**PROVE IT #7 — audit your own schema.**
```sql
-- forbidden types
SELECT c.relname, a.attname, t.typname FROM pg_attribute a
JOIN pg_class c ON c.oid=a.attrelid JOIN pg_type t ON t.oid=a.atttypid
WHERE c.relkind='r' AND a.attnum>0 AND NOT a.attisdropped
  AND c.relnamespace='public'::regnamespace
  AND t.typname IN ('timestamp','money','float4','float8');

-- amounts with no currency column on the same table
SELECT c.relname, a.attname FROM pg_attribute a JOIN pg_class c ON c.oid=a.attrelid
WHERE a.attname ~ '(amount|price|total|fee|balance)' AND a.attnum>0
  AND NOT EXISTS (SELECT 1 FROM pg_attribute a2 WHERE a2.attrelid=c.oid
                    AND a2.attname ~ 'currency');
```

**PROVE IT #8 — check your server's zone settings.**
```sql
SHOW TimeZone;
SELECT name, utc_offset, is_dst FROM pg_timezone_names WHERE name='Asia/Kolkata';
SELECT count(*) FROM pg_timezone_abbrevs WHERE abbrev='IST';
```

---

## The design decision framework

```
TIME — for every temporal column, ask:
  Is it an INSTANT (something happened)?          → timestamptz
  Is it a CALENDAR DAY (birthday, holiday)?       → date
  Is it a WALL-CLOCK TIME (opening hours)?        → time
  Is it a DURATION?                               → interval, or an
                                                     integer of seconds
  Is it a FUTURE event a human scheduled?         → ★ timestamptz
                                                     + IANA zone name
                                                     + the local wall time
  Is it a SPAN?                                   → tstzrange, '[)' bounds
  ⇒ AND: pin TimeZone=UTC on the database, the role, AND the connection.

MONEY — always:
  amount_minor  bigint NOT NULL
  currency      char(3) NOT NULL REFERENCES currencies(code)
  + exponent per currency (never hardcode /100)
  + an explicit, recorded rounding rule
  + FX rates as integers, and store the rate USED
  ✗ float · ✗ the money type · ✗ numeric on a 10k/s path
  ✓ numeric IS acceptable for low-volume columns where readability wins

IDENTITY:
  PK           surrogate, immutable (Topic 22)
  business id  UNIQUE, case-insensitive if it's an email, partial if
               soft deletes exist
  verification an unverified identifier is a CLAIM, not an identity
  person       ★ cannot be keyed. Model links as a fuzzy hypothesis
               table with a confidence, never as an FK.

HISTORY — ask ONE question, before the first UPDATE:
  "Will anyone ever ask what this was at some past time?"
    NO                       → update in place
    "what was TRUE then"     → validity ranges + EXCLUDE
    "what did we BELIEVE then" → bitemporal (+ recorded_at/superseded_at)
    "who changed it"         → an audit table, alongside either
  ★ The cost of adding history on day one is one column and one
    constraint. The cost of adding it later is that the history you
    wanted does not exist.

THE SIGNAL TO LOOK FOR — run the audit query in PROVE IT #7.
  Any `timestamp` column   → a latent correctness bug. Fix now; it gets
                             worse with every row.
  Any float/money amount   → drift is already happening.
  Any amount with no       → someone will sum across currencies.
    currency column
  Any table you UPDATE that
    finance/legal asks about → you need validity ranges.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create a table with `timestamp` and `timestamptz` columns. Insert the same literal under `TimeZone='Asia/Kolkata'`, then read it back under three different session zones. Explain exactly what each column shows and why. Then demonstrate the birthday-off-by-one bug and fix it.

### Exercise 2 — medium (apply it)
Model a "scheduled recurring meeting" that must survive a government DST rule change:
(a) design the table, justifying every column,
(b) show the query that finds "meetings starting in the next hour" and explain why it must use the instant, not the local time,
(c) show what must happen when the IANA tz database is updated,
(d) explain why storing only `timestamptz` is insufficient, and why storing only the local time + zone is also insufficient.

### Exercise 3 — hard (production simulation)
A 4-year-old payments platform in 6 countries has: `timestamp` columns written by services in 3 different zones, `float8` money columns, `email text UNIQUE` (case-sensitive) with 1,842 case-duplicate accounts, and prices that are `UPDATE`d in place while finance asks quarterly "what did we charge in month X?"

(a) Write the audit queries that find all four problems.
(b) For the timestamps: give the procedure to determine each row's original zone, and state precisely which rows are **unrecoverable** and how you'd document that.
(c) For the money: the conversion `round(amount*100)` is a rounding decision on values that are already wrong. Explain the risk, and give the verification you'd run before and after.
(d) For the duplicates: explain why a bulk merge is unsafe, and design the resolution process including what happens to each account's ledger balance.
(e) Design the bitemporal price table, and show the two queries that answer "what is true about March" and "what did we believe on 31 March."
(f) Give the full zero-downtime migration order for all four, with the reasoning for the ordering.
(g) Write the CI check that prevents each of the four from recurring.

---

## Mental model checkpoint

1. `timestamp` and `timestamptz` are both 8 bytes. What differs, and why does it make one of them a correctness bug?
2. Why must a future scheduled event store an IANA zone name in addition to an instant?
3. Why is a birthday a `date` and not a `timestamptz`?
4. Give three reasons `bigint` in minor units beats both `float` and `numeric` for money.
5. Why can't you hardcode `/100` when formatting money?
6. Name the three distinct things people mean by "identity." Which one cannot be keyed, and what do you do instead?
7. What question decides whether you need validity ranges, and when must you answer it?

---

## Quick reference card

**Time**

| Need | Type |
|---|---|
| An instant | `timestamptz` |
| A calendar day | `date` |
| A wall-clock time | `time` |
| A span | `tstzrange` with `'[)'` |
| A future human-scheduled event | `timestamptz` + IANA zone + local wall time |
| **Never** | `timestamp`, offsets, abbreviations |

**Money**

```sql
amount_minor bigint  NOT NULL CHECK (amount_minor >= 0),
currency     char(3) NOT NULL REFERENCES currencies(code)
-- + exponent per currency, + explicit rounding, + FX as integers
```

**Identity**

```sql
id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,   -- ① row identity
email citext NOT NULL,                                    -- ② business id
email_verified_at timestamptz NULL,                       -- a claim until verified
deleted_at timestamptz NULL
-- CREATE UNIQUE INDEX … ON users (email) WHERE deleted_at IS NULL;
-- ③ the person: a fuzzy link table with confidence, never an FK
```

**History**

| Question | Design |
|---|---|
| none | update in place |
| "what was true then" | `valid tstzrange` + `EXCLUDE` |
| "what did we believe then" | + `recorded_at`, `superseded_at` |
| "who changed it" | a separate audit table |

**The four non-negotiables:** `timestamptz` · `bigint` minor units + currency · surrogate PK + `UNIQUE` business id · decide on history *before* the first `UPDATE`.

---

## When would I use this at work?

1. **Every new table.** Four rules, applied in thirty seconds, that prevent the two bugs (zone loss, money drift) that are hardest to detect and sometimes impossible to reverse.

2. **A report that gives different answers to different people.** The `timestamp` audit query finds it in ten seconds, and knowing the failure is *unrecoverable for some rows* changes the conversation from "fix it" to "quantify and document it."

3. **When finance asks "what did we charge in March?"** If the answer is "we don't know," you now have the vocabulary to explain why, and the design (validity ranges + transaction time) that makes it answerable from now on.

---

## Connected topics

**Understand before this:** 22 (surrogate keys), 24 (`EXCLUDE`, `CHECK`), 25 (the type menu).

**This unlocks:**
- **27** — soft deletes and audit columns, which are identity and history patterns
- **28** — migrating these four in production
- **36** — temporal normalisation
- **69** — PII, GDPR, and why identity modelling is a compliance concern
- **Case studies 03, 04** — money and bitemporal modelling in full production designs
