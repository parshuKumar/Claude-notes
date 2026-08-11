# 24 — Constraints in Depth
## Phase: Database Design

---

## ELI5 — The Simple Analogy

A form at a government office, and the clerk behind the desk.

**A validated web form** is polite. It greys out the submit button, shows a red message, tells you the pincode must be six digits. Lovely experience — and completely bypassable by anyone who walks up with a paper form, or emails the office directly, or is a different department with their own system.

**The clerk** is the constraint. She looks at the form and says "this pincode has five digits — I cannot accept this." Not "shouldn't." *Cannot.* It doesn't matter which form you used, which department you're from, or whether it's 3am. The rule lives at the desk where the data enters the building, and there is exactly one desk.

That's the whole argument for database constraints. Your application validation is the web form: essential for good UX, useless as a guarantee. The constraint is the clerk.

---

## Where this fits in the big picture

```
   20 ER modelling (Q6: mandatory or optional?)
   21 ER → schema · 22 primary keys · 23 foreign keys
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 24 CONSTRAINTS                           │ ← YOU ARE HERE
        │ business rules the engine enforces       │
        └────────────────────┬─────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        25/26 data types  28 migrations  29+ normalisation
        (the domain)      (adding them   (constraints ARE
                           safely)        the rules NFs formalise)
```

Topics 22 and 23 covered two specific constraints (`PRIMARY KEY`, `FOREIGN KEY`). **This topic covers the rest — and the one that most developers have never used, `EXCLUDE`, which is the most powerful of all.**

---

## What is this?

Six constraint types, each enforcing a rule the database will not let you break:

| Constraint | Enforces |
|---|---|
| `NOT NULL` | this value must be present |
| `CHECK` | this row must satisfy a predicate |
| `UNIQUE` | no two rows share these values |
| `PRIMARY KEY` | `UNIQUE` + `NOT NULL` + identity (Topic 22) |
| `FOREIGN KEY` | this value exists in another table (Topic 23) |
| `EXCLUDE` | **no two rows may satisfy a relation** — the general case |

Plus **domains** (a reusable named type + constraint) and **generated columns** (a value the engine computes and guarantees).

---

## Why does it matter for a backend developer?

Because "we validate in the application" is the single most expensive false economy in schema design, and the reason is mechanical rather than cultural:

```
 EVERY APPLICATION CHECK HAS THE SAME SHAPE:
     SELECT … ;                ← check
     -- ★ THE GAP
     INSERT/UPDATE … ;         ← act

 Two transactions both pass the check in the gap, both act.
 At 10 requests/second the gap is hit rarely enough that you blame
 something else. At 10,000/second you hit it hundreds of times a day.

 A CONSTRAINT HAS NO GAP. The check happens INSIDE the index insert or
 the row write, under the same page lock. There is nowhere for a second
 transaction to slip in.
```

And the second reason, which is about time rather than concurrency:

```
 An application check protects the code paths that existed when someone
 wrote it. It does not protect:
   • the migration script run at 2am
   • the admin console built by another team
   • the data import from the acquired company
   • the psql session during the incident
   • the code path added next quarter by someone who didn't know
 ⇒ A constraint protects ALL of them, forever, including the ones
   that don't exist yet.
```

---

## The physical reality

### Where each constraint is enforced

```
 CONSTRAINT     ENFORCED BY                      COST PER WRITE
 ──────────────────────────────────────────────────────────────────────
 NOT NULL       a bit in the tuple header        ~0 — checked while
                (the NULL bitmap, Topic 05)       building the tuple
                ⇒ FREE. There is no reason not to use it.

 CHECK          an expression evaluated in the   ~0.1–2 µs, depending
                executor before the row is        on the expression
                written                          ⇒ effectively free for
                                                  simple predicates

 UNIQUE         ★ A B-TREE INDEX. The check      one index insert:
                happens DURING the index          3–4 page reads +
                insert, under an exclusive        1 page write + WAL
                page lock.                       ⇒ same cost as any index

 PRIMARY KEY    UNIQUE + NOT NULL                same as UNIQUE

 FOREIGN KEY    four internal triggers running   ~2–5 µs (Topic 23)
                an indexed lookup

 EXCLUDE        ★ A GiST (or B-tree) INDEX.      one index insert +
                On insert, search the index for   a search: ~5–20 µs
                any row where ALL the declared    (GiST is pricier than
                operators return true.            B-tree)

 ⇒ NOT NULL and CHECK are essentially free. UNIQUE and EXCLUDE cost
   exactly one index each — which is the price of the guarantee, and
   usually a price you were paying anyway.
```

### `UNIQUE` — why there is no race window

```
 INSERT INTO users (email) VALUES ('a@shop.in');

 1. Build the tuple, write it to the heap.
 2. Insert into uq_users_email:
      a) descend the B-tree to the target leaf page
      b) ★ TAKE AN EXCLUSIVE BUFFER LOCK ON THAT PAGE
      c) scan the page for an existing entry with the same key
      d) found one? Check the heap tuple it points at:
           visible & live         → ERROR: duplicate key
           from an IN-PROGRESS txn→ ★ WAIT on that transaction's XID,
                                     then re-check after it resolves
           dead (deleted/aborted) → proceed
      e) insert the entry
      f) release the page lock
 3. Commit.

 ★ STEPS (c) AND (e) HAPPEN UNDER THE SAME LOCK. There is no instant at
   which another transaction can check-and-find-nothing and then insert.
 ★ STEP (d)'s WAIT is the subtle part: two concurrent inserts of the same
   value do not both succeed and do not both fail — the second BLOCKS
   until the first commits or aborts, then behaves correctly either way.
```

### `EXCLUDE` — the generalisation of `UNIQUE`

```
 UNIQUE (a, b)  means:  no two rows where  a = a  AND  b = b
                                               ↑          ↑
                                            always     always
                                            equality   equality

 EXCLUDE USING gist (a WITH =, b WITH &&)  means:
                no two rows where  a = a  AND  b OVERLAPS b
                                       ↑            ↑
                                    YOUR CHOICE OF OPERATOR

 ⇒ UNIQUE is EXCLUDE with every operator fixed to `=`.
   EXCLUDE lets you pick the operator per column.

 THE OPERATORS THAT MATTER:
   =    equality              (any type, needs btree_gist for scalars)
   &&   overlaps              (ranges, arrays, geometry)  ★ the big one
   @>   contains
   <@   contained by
   ~    same-as               (geometry)

 THE MENTAL MODEL:
   "There must not exist two rows for which ALL of these are true."
   Read it as a forbidden pattern, not as a uniqueness rule.
```

### `NOT VALID` — adding constraints to live tables

```
 ALTER TABLE orders ADD CONSTRAINT ck_total_positive
   CHECK (total_paise > 0);
   ⇒ takes ACCESS EXCLUSIVE and scans the WHOLE TABLE.
     On 500 GB that is a multi-minute write outage.

 ALTER TABLE orders ADD CONSTRAINT ck_total_positive
   CHECK (total_paise > 0) NOT VALID;
   ⇒ brief ACCESS EXCLUSIVE, no scan.
   ⇒ ★ ENFORCED FOR ALL NEW WRITES IMMEDIATELY.
   ⇒ existing rows are not checked, and the planner will not use the
     constraint for optimisation until it is validated.

 ALTER TABLE orders VALIDATE CONSTRAINT ck_total_positive;
   ⇒ SHARE UPDATE EXCLUSIVE — reads AND writes continue.
   ⇒ scans the table; fails if any existing row violates it.

 ★ WORKS FOR: CHECK and FOREIGN KEY.
 ✗ DOES NOT WORK FOR: NOT NULL (pre-PG18), UNIQUE, EXCLUDE — those
   need the index built, and `CREATE UNIQUE INDEX CONCURRENTLY` +
   `ADD CONSTRAINT … USING INDEX` is the equivalent trick.
```

---

## How it works — step by step

### `NOT NULL` — free, and underused

```sql
CREATE TABLE orders (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id bigint      NOT NULL,     -- Topic 20 Q6 said mandatory
  total_paise bigint      NOT NULL,
  status      text        NOT NULL,
  rider_id    bigint      NULL,         -- Q6 said optional
  created_at  timestamptz NOT NULL DEFAULT now()
);
```

```
 WHY IT MATTERS BEYOND CORRECTNESS:
  ① THREE-VALUED LOGIC DISAPPEARS.
     `WHERE status <> 'cancelled'` silently EXCLUDES rows where status
     IS NULL. That is a bug in every such query, forever, and it is
     invisible in testing with clean data.
  ② THE PLANNER GETS BETTER ESTIMATES (null_frac = 0, Topic 15).
  ③ `NOT EXISTS` / anti-joins behave predictably (Topic 19).
  ④ It documents intent. A nullable column says "absence is meaningful."
     If it isn't, say so.

 ⚠ ADDING IT TO A LIVE TABLE (pre-PG12) required a full-table
   ACCESS EXCLUSIVE scan. Since PG12, adding NOT NULL is still a scan,
   BUT a trick works:
      ALTER TABLE t ADD CONSTRAINT ck_x CHECK (col IS NOT NULL) NOT VALID;
      ALTER TABLE t VALIDATE CONSTRAINT ck_x;       -- weak lock
      ALTER TABLE t ALTER COLUMN col SET NOT NULL;  -- PG12+ uses the
                                                    -- proven CHECK and
                                                    -- SKIPS the scan
      ALTER TABLE t DROP CONSTRAINT ck_x;
   ★ This is the zero-downtime NOT NULL. Memorise it.
```

### `CHECK` — the domain rules

```sql
CREATE TABLE order_lines (
  order_id         bigint   NOT NULL,
  line_no          smallint NOT NULL,
  quantity         int      NOT NULL,
  unit_price_paise bigint   NOT NULL,
  discount_paise   bigint   NOT NULL DEFAULT 0,
  line_total_paise bigint   NOT NULL,
  status           text     NOT NULL,

  -- ① range / domain rules
  CONSTRAINT ck_qty       CHECK (quantity BETWEEN 1 AND 1000),
  CONSTRAINT ck_price     CHECK (unit_price_paise >= 0),
  CONSTRAINT ck_discount  CHECK (discount_paise >= 0
                                 AND discount_paise <= quantity * unit_price_paise),

  -- ② cross-column arithmetic invariants  ★ genuinely valuable
  CONSTRAINT ck_line_total CHECK (
    line_total_paise = quantity * unit_price_paise - discount_paise),

  -- ③ enumerations
  CONSTRAINT ck_status CHECK (status IN ('pending','picked','shipped','cancelled')),

  PRIMARY KEY (order_id, line_no)
);
```

```
 WHAT A CHECK CAN AND CANNOT SEE:
   ✓ columns of THE SAME ROW
   ✓ IMMUTABLE functions of those columns
   ✗ ★ OTHER ROWS — a CHECK cannot run a subquery
   ✗ ★ OTHER TABLES
   ✗ non-immutable functions: now(), current_user, random()
     ⇒ ERROR: cannot use column reference in DEFAULT / …
       or worse, it silently means something different at restore time

 ⚠ THE now() TRAP:
     CHECK (created_at <= now())          ← REJECTED, and rightly so
   Why: a constraint must hold FOREVER. `now()` changes, so a row valid
   today could be invalid at restore/`pg_upgrade` time, making the dump
   unrestorable.
   ⇒ For "must be in the past", use a trigger, or accept it in the app.

 ⚠ THE SUBQUERY TRAP:
     CHECK ((SELECT count(*) FROM order_lines WHERE order_id = …) <= 50)
   ⇒ NOT ALLOWED, and this is correct: it could not be enforced
     atomically under concurrency. For cross-row rules you need
     EXCLUDE, a UNIQUE index, or a trigger with explicit locking.
```

### `UNIQUE` — and its three variants

```sql
-- ① SIMPLE
CONSTRAINT uq_users_email UNIQUE (email)

-- ② PARTIAL — must be an INDEX, not a table constraint (Topic 13)
CREATE UNIQUE INDEX uq_one_active_sub ON subscriptions (user_id)
  WHERE state = 'active';
--   "at most one ACTIVE subscription per user; unlimited history"

-- ③ ON AN EXPRESSION — also must be an index
CREATE UNIQUE INDEX uq_users_email_ci ON users (lower(email))
  WHERE deleted_at IS NULL;
--   "case-insensitively unique among non-deleted users"

-- ④ NULLS
CREATE TABLE t (e text UNIQUE);
INSERT INTO t VALUES (NULL),(NULL),(NULL);   -- ✓ all succeed! NULL ≠ NULL
CREATE TABLE t2 (e text UNIQUE NULLS NOT DISTINCT);   -- PG15+
INSERT INTO t2 VALUES (NULL);
INSERT INTO t2 VALUES (NULL);                -- ✗ ERROR
```

```
 UNIQUE CONSTRAINT vs UNIQUE INDEX — the difference that matters:
                                    CONSTRAINT   INDEX
   partial (WHERE …)                    ✗          ✓
   on an expression                     ✗          ✓
   INCLUDE columns                      ✓          ✓
   ★ can be an FK target                ✓          ✗
   ★ CREATE/DROP CONCURRENTLY           ✗          ✓
   shows in \d as a constraint          ✓          ✗

 ⇒ Use a CONSTRAINT when a foreign key must reference it.
   Use an INDEX when you need partial/expression, or must build it
   on a live table.
 ⇒ THE BRIDGE: build the index concurrently, then promote it:
      CREATE UNIQUE INDEX CONCURRENTLY uq_x ON t (col);
      ALTER TABLE t ADD CONSTRAINT uq_x UNIQUE USING INDEX uq_x;
```

### `EXCLUDE` — the one nobody uses, and should

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;   -- to mix `=` with `&&`

CREATE TABLE room_bookings (
  id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  room_id bigint    NOT NULL REFERENCES rooms(id),
  during  tstzrange NOT NULL,
  status  text      NOT NULL DEFAULT 'confirmed',

  CONSTRAINT no_double_booking
    EXCLUDE USING gist (room_id WITH =, during WITH &&)
    WHERE (status <> 'cancelled')        -- ★ partial: cancelled frees the slot
);
```

```
 READ IT AS A FORBIDDEN PATTERN:
   "There must not exist two rows where the room_id values are EQUAL
    AND the during ranges OVERLAP, among rows whose status is not
    cancelled."

 ⇒ Double-booking becomes IMPOSSIBLE. Not unlikely — impossible.
   No application code, no lock, no retry loop.

 THE PROBLEMS THIS SOLVES, which nothing else does cleanly:
   • room / resource / seat booking          (case study 02)
   • employee shift overlap
   • price or rate validity periods          (case study 04)
   • subscription / contract terms
   • IP address range allocation             (with inet + &&)
   • "one active X per Y over a time window"

 ★ ALWAYS USE HALF-OPEN RANGES: '[)' bounds.
   [14:00, 16:00) and [16:00, 18:00) do NOT overlap. With '[]' they do,
   and you get a spurious conflict at every boundary. This single detail
   causes more EXCLUDE bugs than everything else combined.
```

### Domains and generated columns

```sql
-- DOMAIN: a named, reusable type + constraint
CREATE DOMAIN email AS citext
  CHECK (VALUE ~ '^[^@\s]+@[^@\s]+\.[^@\s]+$');
CREATE DOMAIN paise AS bigint CHECK (VALUE >= 0);
CREATE DOMAIN pincode AS char(6) CHECK (VALUE ~ '^[1-9][0-9]{5}$');

CREATE TABLE customers (
  id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email   email   NOT NULL UNIQUE,        -- validated everywhere it's used
  balance paise   NOT NULL DEFAULT 0,
  pin     pincode NOT NULL
);
 ✓ the rule is declared ONCE and applies to every column of that type
 ✓ self-documenting: `paise` says more than `bigint`
 ⚠ changing a domain's constraint revalidates every table using it
 ⚠ some ORMs and tools handle domains poorly — check yours first

-- GENERATED COLUMN: computed by the engine, cannot be wrong
CREATE TABLE order_lines (
  quantity         int    NOT NULL,
  unit_price_paise bigint NOT NULL,
  discount_paise   bigint NOT NULL DEFAULT 0,
  line_total_paise bigint GENERATED ALWAYS AS
                   (quantity * unit_price_paise - discount_paise) STORED
);
 ✓ STRONGER than a CHECK: the value cannot be supplied at all, so it
   cannot disagree with its inputs
 ✓ indexable, unlike a view
 ⚠ STORED only (PG has no VIRTUAL yet) — it costs disk
 ⚠ the expression must be IMMUTABLE
```

---

## Concept breakdown

```
THE SIX CONSTRAINTS
│
├── NOT NULL   presence. FREE. Use it wherever Q6 said mandatory.
├── CHECK      a predicate over THIS ROW ONLY.
│              ✗ no subqueries · ✗ no other tables · ✗ no now()/random()
├── UNIQUE     no duplicates. IS an index. No race window.
├── PRIMARY KEY UNIQUE + NOT NULL + identity (Topic 22)
├── FOREIGN KEY referential integrity (Topic 23)
└── EXCLUDE    ★ no two rows satisfy a RELATION. The general case.
               UNIQUE is EXCLUDE with every operator = `=`.

WHERE A RULE CAN LIVE — and what each level can guarantee
│
├── DATABASE CONSTRAINT   ✓ atomic · ✓ every code path · ✓ forever
│                         ✗ poor error messages · ✗ needs a migration
├── DATABASE TRIGGER      ✓ cross-row/cross-table rules
│                         ✗ ★ NOT automatically race-free — a trigger
│                           that SELECTs then decides has the SAME GAP
│                           as application code, unless it takes an
│                           explicit lock
├── APPLICATION           ✓ great messages · ✓ easy to change
│                         ✗ ★ has a gap · ✗ only the paths you wrote
└── CLIENT/UI             ✓ instant feedback · ✗ no guarantee at all
 ⇒ USE ALL FOUR. The database for the GUARANTEE, the app for the
   MESSAGE. They are not alternatives.

DEFERRABILITY
├── NOT DEFERRABLE (default)   checked per statement
├── DEFERRABLE INITIALLY IMMEDIATE   opt in per transaction  ★ prefer
└── DEFERRABLE INITIALLY DEFERRED    always at COMMIT
   ✓ available for: UNIQUE · PRIMARY KEY · FOREIGN KEY · EXCLUDE
   ✗ NOT available for: CHECK, NOT NULL (they're per-row anyway)

ADDING TO A LIVE TABLE
├── CHECK, FOREIGN KEY     → NOT VALID, then VALIDATE  (weak lock)
├── NOT NULL               → the CHECK trick above (PG12+)
└── UNIQUE, EXCLUDE        → CREATE INDEX CONCURRENTLY, then
                             ADD CONSTRAINT … USING INDEX
```

---

## Diagrams

**Diagram 1 — big picture: where each rule is enforced**

```
   ┌──────────────────────────────────────────────────────────────┐
   │ CLIENT / UI          instant feedback, zero guarantee        │
   ├──────────────────────────────────────────────────────────────┤
   │ APPLICATION          good messages, HAS A GAP,               │
   │                      only the code paths you wrote           │
   ├──────────────────────────────────────────────────────────────┤
   │ TRIGGER              cross-row rules, but ALSO has a gap     │
   │                      unless it locks explicitly              │
   ├══════════════════════════════════════════════════════════════┤
   │ ★ CONSTRAINT         atomic · every path · forever · no gap  │
   │   NOT NULL · CHECK · UNIQUE · PK · FK · EXCLUDE              │
   └──────────────────────────────────────────────────────────────┘
          ↑ THE GUARANTEE LIVES HERE, AND ONLY HERE.
            Everything above is user experience.
```

**Diagram 2 — data flow: the gap, and its absence**

```
  APPLICATION CHECK                    UNIQUE CONSTRAINT
  ─────────────────────────            ────────────────────────────
   txn A          txn B                 txn A            txn B
     │              │                     │                │
   SELECT         SELECT                INSERT           INSERT
   "free?"        "free?"                 │                │
     │              │                  descend          descend
   → yes          → yes                    │                │
     │              │                  LOCK PAGE        wait for
     │              │                     │              the page
     ▼              ▼                  scan: free?          │
   INSERT         INSERT               → yes                │
     │              │                  insert               │
   COMMIT         COMMIT               unlock ──────────▶ LOCK PAGE
     │              │                     │              scan: FOUND
     ▼              ▼                  COMMIT            → wait on A's xid
  ★ TWO ROWS. Both "valid".               │              A commits
                                          ▼              → ERROR ✓
                                     ★ ONE ROW.
      ↑ THE GAP                        ↑ NO GAP
```

**Diagram 3 — before/after: `EXCLUDE` vs everything else**

```
 THE REQUIREMENT: "no two bookings for the same room may overlap"

 ATTEMPT 1 — application check
   SELECT 1 FROM bookings WHERE room_id=$1 AND during && $2;
   if none: INSERT
   ✗ two dispatchers, same second → both see none → both insert
   Measured in case study 02: 12 double-bookings/month

 ATTEMPT 2 — SELECT … FOR UPDATE on the room row
   ✓ correct
   ✗ serialises ALL bookings for that room, even non-overlapping ones
   ✗ a lock convoy on popular rooms (case study 01's problem)

 ATTEMPT 3 — a trigger that checks
   ✗ ★ SAME GAP as attempt 1, unless the trigger takes an explicit
     lock — at which point it is attempt 2 with extra steps

 ★ EXCLUDE CONSTRAINT
   EXCLUDE USING gist (room_id WITH =, during WITH &&)
   ✓ checked INSIDE the index insert, under a page lock
   ✓ concurrent NON-overlapping bookings do NOT block each other
   ✓ concurrent OVERLAPPING bookings: one waits, then gets a clean error
   ✓ works for the 2am psql session and the import script too
   ⇒ 12 double-bookings/month → 0, structurally
```

---

## Example 1 — basic

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE EXTENSION IF NOT EXISTS citext;
```

**Step 1 — `NOT NULL` and three-valued logic.**

```sql
CREATE TABLE t_null (id int, status text);
INSERT INTO t_null VALUES (1,'active'),(2,'cancelled'),(3,NULL);
SELECT count(*) FROM t_null WHERE status <> 'cancelled';
```
```
 count
-------
     1        ← ★ expected 2. Row 3 vanished. NULL <> 'cancelled' is NULL.
```
```sql
SELECT count(*) FROM t_null WHERE status IS DISTINCT FROM 'cancelled';   -- 2 ✓
```
**Every `<>` query in your codebase has this bug on every nullable column.** `NOT NULL` deletes the entire class.

**Step 2 — `CHECK` catches what the app forgot.**

```sql
CREATE TABLE order_lines (
  order_id bigint NOT NULL, line_no smallint NOT NULL,
  quantity int NOT NULL,
  unit_price_paise bigint NOT NULL,
  discount_paise bigint NOT NULL DEFAULT 0,
  line_total_paise bigint NOT NULL,
  PRIMARY KEY (order_id,line_no),
  CONSTRAINT ck_qty CHECK (quantity BETWEEN 1 AND 1000),
  CONSTRAINT ck_price CHECK (unit_price_paise >= 0),
  CONSTRAINT ck_discount CHECK (discount_paise BETWEEN 0 AND quantity*unit_price_paise),
  CONSTRAINT ck_total CHECK (line_total_paise = quantity*unit_price_paise - discount_paise)
);

INSERT INTO order_lines VALUES (1,1, 2, 50000, 0, 100000);       -- ✓
INSERT INTO order_lines VALUES (1,2, 0, 50000, 0, 0);
-- ERROR:  new row violates check constraint "ck_qty"
INSERT INTO order_lines VALUES (1,3, 2, 50000, 200000, 0);
-- ERROR:  new row violates check constraint "ck_discount"
INSERT INTO order_lines VALUES (1,4, 2, 50000, 0, 99999);
-- ERROR:  new row violates check constraint "ck_total"   ← ★ the arithmetic bug
```
`ck_total` is the valuable one: it catches a rounding or currency bug in *any* code path that ever writes an order line.

**Step 3 — what `CHECK` cannot do.**

```sql
ALTER TABLE order_lines ADD CONSTRAINT ck_max_lines
  CHECK ((SELECT count(*) FROM order_lines ol WHERE ol.order_id = order_id) <= 50);
-- ERROR:  cannot use subquery in check constraint

ALTER TABLE order_lines ADD CONSTRAINT ck_recent CHECK (created_at <= now());
-- ERROR:  functions in check constraint must be marked IMMUTABLE
```
Both refusals are correct. A subquery could not be enforced atomically; `now()` would make a valid row invalid at restore time.

**Step 4 — `UNIQUE`, and the NULL surprise.**

```sql
CREATE TABLE u1 (email text UNIQUE);
INSERT INTO u1 VALUES (NULL),(NULL),(NULL);   -- ✓ all three
SELECT count(*) FROM u1;                       -- 3

CREATE TABLE u2 (email text UNIQUE NULLS NOT DISTINCT);   -- PG15+
INSERT INTO u2 VALUES (NULL);
INSERT INTO u2 VALUES (NULL);
-- ERROR:  duplicate key value violates unique constraint "u2_email_key"
```

**Step 5 — partial unique: the pattern that solves real problems.**

```sql
CREATE TABLE subscriptions (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id bigint NOT NULL, plan text NOT NULL,
  state text NOT NULL CHECK (state IN ('active','cancelled','expired'))
);
CREATE UNIQUE INDEX uq_one_active ON subscriptions (user_id) WHERE state='active';

INSERT INTO subscriptions (user_id,plan,state) VALUES (7,'pro','cancelled');  -- ✓
INSERT INTO subscriptions (user_id,plan,state) VALUES (7,'pro','expired');    -- ✓
INSERT INTO subscriptions (user_id,plan,state) VALUES (7,'pro','active');     -- ✓
INSERT INTO subscriptions (user_id,plan,state) VALUES (7,'team','active');
-- ERROR:  duplicate key value violates unique constraint "uq_one_active"
```
**History preserved, invariant enforced, zero race window.**

**Step 6 — `EXCLUDE`, and why `'[)'` matters.**

```sql
CREATE TABLE bookings (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  room_id bigint NOT NULL,
  during tstzrange NOT NULL,
  status text NOT NULL DEFAULT 'confirmed',
  EXCLUDE USING gist (room_id WITH =, during WITH &&) WHERE (status <> 'cancelled')
);

INSERT INTO bookings (room_id,during) VALUES
  (12, '[2026-04-01 14:00, 2026-04-01 16:00)');            -- ✓
INSERT INTO bookings (room_id,during) VALUES
  (12, '[2026-04-01 16:00, 2026-04-01 18:00)');            -- ✓ touching, not overlapping
INSERT INTO bookings (room_id,during) VALUES
  (12, '[2026-04-01 15:00, 2026-04-01 17:00)');
-- ERROR:  conflicting key value violates exclusion constraint
INSERT INTO bookings (room_id,during) VALUES
  (13, '[2026-04-01 15:00, 2026-04-01 17:00)');            -- ✓ different room
```

Now the closed-bound mistake:
```sql
CREATE TABLE b_closed (
  room_id bigint NOT NULL, during tstzrange NOT NULL,
  EXCLUDE USING gist (room_id WITH =, during WITH &&)
);
INSERT INTO b_closed VALUES (1, '[2026-04-01 14:00, 2026-04-01 16:00]');
INSERT INTO b_closed VALUES (1, '[2026-04-01 16:00, 2026-04-01 18:00]');
-- ERROR: conflicting key value  ← ★ they share the instant 16:00
```
**Always `'[)'`.**

**Step 7 — no race window, proved.**

```sql
-- session 1
BEGIN;
INSERT INTO bookings (room_id,during) VALUES (99,'[2026-05-01 10:00,2026-05-01 12:00)');
-- do NOT commit
-- session 2
INSERT INTO bookings (room_id,during) VALUES (99,'[2026-05-01 11:00,2026-05-01 13:00)');
-- ⏸ BLOCKS
-- session 1
COMMIT;
-- session 2 immediately:
-- ERROR: conflicting key value violates exclusion constraint
```
Session 2 **waited on session 1's transaction** rather than seeing "no conflict." That wait *is* the enforcement.

**Step 8 — generated columns beat `CHECK`.**

```sql
CREATE TABLE gl (
  quantity int NOT NULL, unit_price_paise bigint NOT NULL,
  discount_paise bigint NOT NULL DEFAULT 0,
  line_total_paise bigint GENERATED ALWAYS AS
    (quantity*unit_price_paise - discount_paise) STORED
);
INSERT INTO gl (quantity,unit_price_paise) VALUES (2,50000);
SELECT * FROM gl;                                  -- line_total_paise = 100000
INSERT INTO gl (quantity,unit_price_paise,line_total_paise) VALUES (2,50000,1);
-- ERROR: cannot insert a non-DEFAULT value into column "line_total_paise"
--   ★ stronger than CHECK: the wrong value cannot even be OFFERED.
```

---

## Example 2 — production scenario

**The situation.** A 5-year-old booking platform, 2.1 TB. The team's philosophy has been "validate in the application." An audit finds:

```sql
SELECT 'negative balances'   AS issue, count(*) FROM accounts WHERE balance_paise < 0
UNION ALL SELECT 'null status', count(*) FROM bookings WHERE status IS NULL
UNION ALL SELECT 'invalid status', count(*) FROM bookings
  WHERE status NOT IN ('pending','confirmed','cancelled','completed')
UNION ALL SELECT 'double bookings', count(*) FROM (
  SELECT a.id FROM bookings a JOIN bookings b
    ON a.room_id=b.room_id AND a.id<b.id AND a.during && b.during
   WHERE a.status<>'cancelled' AND b.status<>'cancelled') x
UNION ALL SELECT 'line total mismatch', count(*) FROM order_lines
  WHERE line_total_paise <> quantity*unit_price_paise - discount_paise
UNION ALL SELECT 'duplicate active subs', count(*) FROM (
  SELECT user_id FROM subscriptions WHERE state='active'
   GROUP BY 1 HAVING count(*)>1) y;
```
```
        issue          | count
-----------------------+--------
 negative balances     |     41
 null status           |   8412
 invalid status        |    118      ← 'Cancelled','CANCELLED','cancelled '
 double bookings       |    847      ← ★ real customers, real refunds
 line total mismatch   |   2104      ← ★ a currency rounding bug in 2023
 duplicate active subs |    312      ← ★ 312 customers billed twice
```

**Step 2 — the root causes.**

| Issue | Why the app check failed |
|---|---|
| Negative balances | a 2024 refund path skipped the balance check entirely |
| NULL status | a data import from an acquisition set no status |
| Invalid status | three services each normalise casing differently |
| Double bookings | the `SELECT`-then-`INSERT` gap, hit 847 times in 5 years |
| Line total mismatch | a rounding bug shipped for 11 days in 2023 |
| Duplicate active subs | a retry storm during a payment-provider outage |

**Every one is a code path or a race the check didn't cover.** Not one is "someone forgot to validate."

**Step 3 — clean, then constrain. Cleaning is a business decision.**

```sql
-- ① INVALID STATUS — normalise. Safe, mechanical.
UPDATE bookings SET status = lower(trim(status))
WHERE status <> lower(trim(status));

-- ② NULL STATUS — 8,412 rows from an import. Infer, don't guess blindly.
UPDATE bookings SET status = CASE
  WHEN during << tstzrange(now(),'infinity') THEN 'completed'
  ELSE 'confirmed' END
WHERE status IS NULL;

-- ③ NEGATIVE BALANCES — 41 rows, MONEY. ★ NOT a bulk UPDATE.
--    Case study 03's principle: unexplained money needs a home.
INSERT INTO reconciliation_queue (reason, payload)
SELECT 'NEGATIVE_BALANCE', to_jsonb(a) FROM accounts a WHERE balance_paise < 0;

-- ④ DOUBLE BOOKINGS — 847 rows with real customers. ★ NOT bulk-fixable.
INSERT INTO manual_review_queue (reason, payload)
SELECT 'DOUBLE_BOOKING', jsonb_build_object('a',a.id,'b',b.id)
FROM bookings a JOIN bookings b ON a.room_id=b.room_id AND a.id<b.id
  AND a.during && b.during
WHERE a.status<>'cancelled' AND b.status<>'cancelled';

-- ⑤ LINE TOTAL MISMATCH — historical invoices. ★ DO NOT "correct" them:
--    they were sent to customers and reconciled. Issue credit notes
--    instead (case study 04's I1). Flag, don't rewrite.
INSERT INTO manual_review_queue (reason, payload)
SELECT 'LINE_TOTAL_MISMATCH', to_jsonb(ol) FROM order_lines ol
WHERE line_total_paise <> quantity*unit_price_paise - discount_paise;
```

**Step 4 — add the constraints without downtime.**

```sql
-- ── NOT NULL, the zero-downtime way (PG12+) ──
ALTER TABLE bookings ADD CONSTRAINT ck_status_nn
  CHECK (status IS NOT NULL) NOT VALID;                    -- brief lock
ALTER TABLE bookings VALIDATE CONSTRAINT ck_status_nn;     -- weak lock, scans
ALTER TABLE bookings ALTER COLUMN status SET NOT NULL;     -- ★ no scan: PG
                                                           --   uses the proven CHECK
ALTER TABLE bookings DROP CONSTRAINT ck_status_nn;

-- ── CHECK constraints ──
ALTER TABLE bookings ADD CONSTRAINT ck_status
  CHECK (status IN ('pending','confirmed','cancelled','completed')) NOT VALID;
ALTER TABLE bookings VALIDATE CONSTRAINT ck_status;

ALTER TABLE accounts ADD CONSTRAINT ck_balance_nonneg
  CHECK (balance_paise >= 0 OR allows_negative) NOT VALID;
-- ⚠ leave NOT VALID until the 41 reconciliation cases are resolved.
--   New writes are enforced NOW; that is the important half.

ALTER TABLE order_lines ADD CONSTRAINT ck_line_total
  CHECK (line_total_paise = quantity*unit_price_paise - discount_paise) NOT VALID;
-- ⚠ also stays NOT VALID: the 2,104 historical rows are IMMUTABLE
--   (they were invoiced). This is a legitimate permanent NOT VALID.

-- ── UNIQUE, built concurrently then promoted ──
CREATE UNIQUE INDEX CONCURRENTLY uq_one_active_sub
  ON subscriptions (user_id) WHERE state='active';
-- (fails if the 312 duplicates remain — resolve them first)

-- ── EXCLUDE: the big one. No CONCURRENTLY for exclusion constraints,
--    so build the index first, then attach.
CREATE INDEX CONCURRENTLY idx_bookings_excl
  ON bookings USING gist (room_id, during) WHERE (status <> 'cancelled');
ALTER TABLE bookings ADD CONSTRAINT no_double_booking
  EXCLUDE USING gist (room_id WITH =, during WITH &&)
  WHERE (status <> 'cancelled');
-- ⚠ this DOES take ACCESS EXCLUSIVE and build its own index.
--   Schedule it in a maintenance window, or accept ~90 s on this table.
```

**Step 5 — a permanent `NOT VALID` is a legitimate design, not a failure.**

```
 ★ AN UNDER-APPRECIATED PATTERN:
   `ck_line_total NOT VALID` enforces the rule on every write from now
   on, while explicitly acknowledging that 2,104 historical rows predate
   it and must not be rewritten.

   This is exactly right for immutable history. The alternative — no
   constraint — means the bug can recur. "Correcting" the history would
   violate case study 04's I1.

   ⇒ Document it. A NOT VALID constraint with a comment explaining why
     is better engineering than either extreme.

   COMMENT ON CONSTRAINT ck_line_total ON order_lines IS
     'NOT VALID: 2104 rows predate the 2023 rounding fix and are
      immutable invoiced records. Enforced for all writes since 2026-08.';
```

**Step 6 — results.**

| | Before | After |
|---|---|---|
| Double bookings | 847 (ongoing ~14/month) | **0, structurally impossible** |
| Invalid/NULL status | 8,530 | **0, impossible** |
| Duplicate active subs | 312 | **0, impossible** |
| Line-total bugs | recurring | **impossible on new writes** |
| Write throughput | baseline | **−4%** (one GiST index + 6 CHECKs) |
| Application validation code | 2,400 lines | kept — for **error messages** |

**4% of write throughput to make six classes of bug impossible.**

**Step 7 — the app keeps its checks, for a different job.**

```js
// The DB gives the guarantee. The app gives the message.
try {
  await db.query('INSERT INTO bookings (room_id, during) VALUES ($1, $2)', [roomId, range]);
} catch (e) {
  if (e.code === '23P01' && e.constraint === 'no_double_booking')
    return { status: 409, error: 'That room is already booked for this time.' };
  if (e.code === '23514' && e.constraint === 'ck_status')
    return { status: 400, error: 'Invalid booking status.' };
  if (e.code === '23505' && e.constraint === 'uq_one_active_sub')
    return { status: 409, error: 'You already have an active subscription.' };
  throw e;
}
```
**Map constraint names to user-facing messages.** That is the correct division: the database decides *whether*, the application decides *how to say it*.

---

## Common mistakes

**1. "We validate in the application."**
- *Symptom:* invariant violations appear in production despite thorough validation.
- *Engine-level why:* `SELECT`-then-`INSERT` has a gap; and the check covers only the code paths that existed when it was written.
- *Fix:* constraints for the guarantee, app checks for the message. Both.

**2. Nullable columns that shouldn't be.**
- *Symptom:* `WHERE status <> 'cancelled'` silently drops rows.
- *Engine-level why:* `NULL <> 'x'` is `NULL`, not `true`.
- *Fix:* `NOT NULL` wherever Topic 20's Q6 said mandatory. It's free.

**3. Using `SELECT FOR UPDATE` where `EXCLUDE` would do.**
- *Symptom:* lock convoys on popular resources.
- *Engine-level why:* `FOR UPDATE` serialises *all* operations on that row; `EXCLUDE` only conflicts on genuinely overlapping rows.
- *Fix:* an exclusion constraint. It's more correct *and* more concurrent.

**4. Closed range bounds in `EXCLUDE`.**
- *Symptom:* back-to-back bookings rejected as overlapping.
- *Fix:* always `'[)'`.

**5. `ALTER TABLE ADD CONSTRAINT` without `NOT VALID` on a big table.**
- *Symptom:* a multi-minute write outage.
- *Fix:* `NOT VALID`, then `VALIDATE CONSTRAINT` separately.

**6. Believing a trigger is race-free.**
- *Symptom:* a trigger-based "no overlap" check still produces overlaps.
- *Engine-level why:* a trigger that runs `SELECT` and then decides has exactly the same gap as application code. Triggers are transactional, not serialising.
- *Fix:* `EXCLUDE` or `UNIQUE` for anything enforceable by an index. If you genuinely need a trigger, it must take an explicit lock — and then you own the deadlock risk.

**7. Enum-as-`CHECK` when the list changes weekly.**
- *Symptom:* a migration every time marketing adds a status.
- *Fix:* a `CHECK` is right for a stable, engineering-owned list (5 order states). A lookup table + FK is right when non-engineers change it (Topic 27).

**8. Treating a permanent `NOT VALID` as a failure.**
- *Symptom:* refusing to add a constraint because history violates it.
- *Fix:* `NOT VALID` + a `COMMENT` explaining why is a legitimate, documented design. New writes are protected; immutable history is respected.

---

## Hands-on proof

**PROVE IT #1 — three-valued logic.** (Example 1, step 1.)

**PROVE IT #2 — `CHECK` cannot see other rows.** (Example 1, step 3.)

**PROVE IT #3 — `UNIQUE` has no race window.**
```sql
CREATE TABLE r (k int UNIQUE);
-- session 1: BEGIN; INSERT INTO r VALUES (1);        -- no commit
-- session 2: INSERT INTO r VALUES (1);               -- ⏸ BLOCKS
-- session 1: COMMIT;
-- session 2: ERROR: duplicate key value violates unique constraint
```

**PROVE IT #4 — `EXCLUDE` has no race window.** (Example 1, step 7.)

**PROVE IT #5 — half-open vs closed bounds.** (Example 1, step 6.)

**PROVE IT #6 — the zero-downtime `NOT NULL`.**
```sql
CREATE TABLE big (id bigserial PRIMARY KEY, v text);
INSERT INTO big (v) SELECT 'x' FROM generate_series(1,3000000);
\timing on
ALTER TABLE big ALTER COLUMN v SET NOT NULL;         -- full scan, ACCESS EXCLUSIVE
ALTER TABLE big ALTER COLUMN v DROP NOT NULL;

ALTER TABLE big ADD CONSTRAINT ck_v CHECK (v IS NOT NULL) NOT VALID;  -- instant
ALTER TABLE big VALIDATE CONSTRAINT ck_v;                             -- weak lock
ALTER TABLE big ALTER COLUMN v SET NOT NULL;         -- ★ near-instant: no scan
ALTER TABLE big DROP CONSTRAINT ck_v;
```

**PROVE IT #7 — generated columns cannot be wrong.** (Example 1, step 8.)

**PROVE IT #8 — audit your own constraints.**
```sql
SELECT conrelid::regclass AS table_name, conname,
  CASE contype WHEN 'c' THEN 'CHECK' WHEN 'f' THEN 'FOREIGN KEY'
               WHEN 'p' THEN 'PRIMARY KEY' WHEN 'u' THEN 'UNIQUE'
               WHEN 'x' THEN 'EXCLUDE' END AS kind,
  convalidated AS validated, condeferrable AS deferrable,
  pg_get_constraintdef(oid) AS definition
FROM pg_constraint WHERE connamespace='public'::regnamespace
ORDER BY conrelid::regclass::text, contype;

-- tables with NO constraints beyond a PK — the suspicious ones
SELECT c.relname, count(con.oid) FILTER (WHERE con.contype <> 'p') AS non_pk
FROM pg_class c LEFT JOIN pg_constraint con ON con.conrelid=c.oid
WHERE c.relkind='r' AND c.relnamespace='public'::regnamespace
GROUP BY 1 HAVING count(con.oid) FILTER (WHERE con.contype <> 'p') = 0;
```

---

## The design decision framework

```
FOR EVERY COLUMN AND EVERY BUSINESS RULE, ASK:

 ① IS ABSENCE MEANINGFUL?
     NO  → NOT NULL. It is free and it removes three-valued logic
           from every query on that column.
     YES → nullable, and document what NULL means.

 ② IS THERE A DOMAIN RULE ON THIS ROW?
     (positive amounts, valid enum, arithmetic invariant, format)
     → CHECK. Also free. Prefer a GENERATED column for derived values —
       it is stronger than a CHECK because the wrong value cannot be
       offered at all.

 ③ MUST THESE VALUES BE UNIQUE?
     always                    → UNIQUE constraint (FK-targetable)
     only for a subset         → partial UNIQUE INDEX
     case-insensitively        → UNIQUE INDEX on lower(col), or citext
     among non-deleted rows    → partial UNIQUE INDEX WHERE deleted_at IS NULL

 ④ IS THE RULE "NO TWO ROWS MAY …" WITH SOMETHING OTHER THAN `=`?
     → ★ EXCLUDE. This is the question people never ask, and it covers:
       overlapping time ranges · overlapping IP ranges · intersecting
       geometry · "one active X per Y per period"
     → remember btree_gist for scalar `=`, and ALWAYS '[)' bounds.

 ⑤ DOES THE RULE SPAN ROWS OR TABLES?
     → a CHECK cannot. Options, in order of preference:
       (a) restructure so an EXCLUDE or UNIQUE can express it   ★ best
       (b) a trigger WITH an explicit lock (you own the deadlock risk)
       (c) the application + a scheduled reconciliation query
     → (c) is legitimate IF you actually write the reconciliation job.

WHERE THE RULE LIVES:
  DATABASE  → the guarantee. Always, for anything that must be true.
  APP       → the message. Map constraint names to user-facing errors.
  ⇒ Not alternatives. Both, always.

ADDING TO A LIVE TABLE:
  CHECK / FK      → NOT VALID, then VALIDATE
  NOT NULL        → the CHECK trick (PG12+)
  UNIQUE          → CREATE UNIQUE INDEX CONCURRENTLY, then USING INDEX
  EXCLUDE         → needs ACCESS EXCLUSIVE; schedule it

THE SIGNAL TO LOOK FOR:
      -- tables with no constraints beyond a primary key
      SELECT c.relname FROM pg_class c
      LEFT JOIN pg_constraint con ON con.conrelid = c.oid
      WHERE c.relkind='r' AND c.relnamespace='public'::regnamespace
      GROUP BY 1
      HAVING count(con.oid) FILTER (WHERE con.contype <> 'p') = 0;

  Every table in that list is trusting application code entirely.
  For each, write down the invariants and ask which the engine could
  enforce. In practice you will find 3–8 per table, most of them free.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create a table with a nullable `status` column and 3 rows, one NULL. Show that `WHERE status <> 'x'` and `WHERE status IS DISTINCT FROM 'x'` return different counts. Then add every constraint you can justify to an `order_lines` table (at least: 2 `NOT NULL`, 3 `CHECK`, 1 `UNIQUE`, 1 generated column) and demonstrate each rejecting a bad row.

### Exercise 2 — medium (apply it)
Implement "an employee may not be scheduled for two overlapping shifts, and may not work more than one shift per location per day."
(a) Write the `EXCLUDE` constraint for the first rule and prove it works, including the boundary case of back-to-back shifts.
(b) The second rule needs a different mechanism — identify it and implement it.
(c) Show a two-session test proving each has no race window.
(d) Explain why a trigger-based implementation of (a) would still be racy, and what it would have to do to be correct.
(e) Add both to a table that already has 10M rows, with zero downtime.

### Exercise 3 — hard (production simulation)
A 2.1 TB booking platform has no constraints beyond primary keys. An audit finds: 847 double bookings, 8,412 NULL statuses, 118 invalid status strings, 41 negative balances, 2,104 line-total mismatches, 312 duplicate active subscriptions.

(a) For each, write the detection query and the constraint that would make it impossible.
(b) Three of the six must **not** be bulk-corrected. Identify which, explain why for each, and give the correct handling — referencing the relevant case-study principle.
(c) Give the complete zero-downtime procedure to add all six, in dependency order, with the lock level of each step and what is blocked during it.
(d) One constraint must remain permanently `NOT VALID`. Identify it, explain why that's correct rather than a failure, and write the `COMMENT` documenting it.
(e) The `EXCLUDE` constraint cannot be added concurrently. Give three options for a 24/7 system and their trade-offs.
(f) Estimate the write-throughput cost of all six and justify it against the business risk.
(g) Write the Node.js error-mapping layer that turns each constraint violation into a user-facing message with the right HTTP status.
(h) Write the CI check that fails a migration adding a table with no constraints beyond its PK.

---

## Mental model checkpoint

1. Name the six constraint types and what each enforces. Which two are essentially free?
2. Why does an application check have a race window that a `UNIQUE` constraint does not? Describe what the second transaction actually does.
3. Three things a `CHECK` constraint cannot do. Why is each restriction correct rather than an oversight?
4. What is `EXCLUDE`, and how does it generalise `UNIQUE`? Give two problems only it solves cleanly.
5. Why must range bounds be `'[)'` in an exclusion constraint?
6. Give the four-step zero-downtime `NOT NULL` procedure and explain why step 3 doesn't scan.
7. Is a trigger race-free? Explain, and say what it would have to do to become so.

---

## Quick reference card

| Constraint | Enforces | Cost | Deferrable |
|---|---|---|---|
| `NOT NULL` | presence | ~free | ✗ |
| `CHECK` | a predicate on one row | ~free | ✗ |
| `UNIQUE` | no duplicates | one index | ✓ |
| `PRIMARY KEY` | unique + not null + identity | one index | ✓ |
| `FOREIGN KEY` | referential integrity | 4 triggers | ✓ |
| `EXCLUDE` | **no two rows satisfy a relation** | one GiST index | ✓ |

**`CHECK` cannot:** use subqueries · reference other tables · use non-immutable functions (`now()`, `random()`, `current_user`).

**Adding to a live table**

| Constraint | Procedure |
|---|---|
| `CHECK`, `FOREIGN KEY` | `NOT VALID` → `VALIDATE CONSTRAINT` |
| `NOT NULL` | `CHECK (col IS NOT NULL) NOT VALID` → `VALIDATE` → `SET NOT NULL` → drop the CHECK |
| `UNIQUE` | `CREATE UNIQUE INDEX CONCURRENTLY` → `ADD CONSTRAINT … USING INDEX` |
| `EXCLUDE` | needs `ACCESS EXCLUSIVE`; schedule it |

**Error codes to map in your app**

| Code | Constraint |
|---|---|
| `23502` | not-null violation |
| `23503` | foreign key violation |
| `23505` | unique violation |
| `23514` | check violation |
| `23P01` | **exclusion violation** |

**The patterns worth memorising**

```sql
-- one active X per Y, with history
CREATE UNIQUE INDEX ON subscriptions (user_id) WHERE state='active';

-- soft-delete-aware, case-insensitive uniqueness
CREATE UNIQUE INDEX ON users (tenant_id, lower(email)) WHERE deleted_at IS NULL;

-- no overlapping time ranges per resource
EXCLUDE USING gist (room_id WITH =, during WITH &&) WHERE (status <> 'cancelled')

-- a derived value that cannot be wrong
total_paise bigint GENERATED ALWAYS AS (qty * unit_price - discount) STORED
```

---

## When would I use this at work?

1. **Any booking, scheduling, or resource-allocation feature.** `EXCLUDE` turns "we need a lock and a retry loop" into one line of DDL that is both more correct and more concurrent than anything you'd write by hand.

2. **After any data-integrity incident.** Instead of adding another application check, ask "which constraint would have made this impossible?" and add it `NOT VALID`. New writes are protected within the hour, and history is cleaned at your own pace.

3. **Reviewing a new table.** Every column gets `NOT NULL` unless absence is meaningful; every enum gets a `CHECK`; every derived value gets `GENERATED`; every "no two rows may…" rule gets `UNIQUE` or `EXCLUDE`. Five minutes, and it removes whole categories of future bug.

---

## Connected topics

**Understand before this:** 20 (Q6: mandatory or optional), 13 (partial and expression indexes), 16 (GiST — the engine behind `EXCLUDE`), 22–23 (PK and FK).

**This unlocks:**
- **25/26** — data types and domains: the other half of "what values are legal"
- **27** — antipatterns, most of which are missing constraints
- **28** — zero-downtime migrations, where `NOT VALID` lives
- **29–38** — normalisation: the normal forms are constraints stated as theory
- **50** — serialisable isolation, the general answer to cross-row rules
- **Case studies 02, 03, 04** — every one is built on an `EXCLUDE` or a partial `UNIQUE`
