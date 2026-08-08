# 21 — Translating ER to Relational Schema
## Phase: Database Design

---

## ELI5 — The Simple Analogy

You have an architect's drawing: rooms, doors between them, a note saying "this door is 900mm wide."

Now you need a **bill of materials**. That translation is not creative — it's mechanical. Every room becomes a set of walls. Every door becomes a frame, hinges, and a handle. A wide door becomes a specific frame part number. There's a lookup table, and a competent builder produces the same list every time.

Translating an ER diagram to a schema is exactly that. **Entity → table. 1:N → foreign key on the many side. M:N → junction table.** There are about eight rules, and once you know them, the translation is deterministic.

The judgement was spent in Topic 20 — getting the diagram right. This topic is the bill of materials, and the goal is that two engineers given the same diagram produce the same DDL.

---

## Where this fits in the big picture

```
   20 ER modelling — requirements → boxes and lines
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 21 ER → RELATIONAL SCHEMA                │ ← YOU ARE HERE
        │ boxes and lines → tables and keys        │
        └────────────────────┬─────────────────────┘
                             │
      ┌──────────────────────┼───────────────────────┐
      ▼                      ▼                       ▼
  22 primary keys      23 foreign keys        24 constraints
  (which key type?)    (what CASCADE?)        (the rules)
                             │
                             ▼
                    29–38 normalisation
                    (proves the result is correct)
```

Topic 20 produced a diagram. **This topic produces DDL you can run.** Topics 22–25 then refine the *choices* inside it (which key type, which cascade, which data type).

---

## What is this?

Eight mechanical rules that convert every ER construct into relational tables, keys, and constraints:

| ER construct | Becomes |
|---|---|
| Strong entity | a table with a primary key |
| Weak entity | a table with a composite PK including the parent's key |
| 1:N relationship | a foreign key column on the **many** side |
| 1:1 relationship | one table, or a FK + `UNIQUE` on the dependent side |
| M:N relationship | a junction table with two foreign keys |
| Relationship with attributes | a table (it was an entity all along) |
| Multi-valued attribute | its own table |
| Composite attribute | flattened columns, or its own table |
| Inheritance / subtypes | one of three strategies — a real decision |

Seven are mechanical. The eighth — inheritance — is the only one requiring judgement, and it gets its own section.

---

## Why does it matter for a backend developer?

Because the mechanical rules are where most schemas go wrong in *small*, expensive ways:

```
 1. THE FOREIGN KEY ON THE WRONG SIDE
    orders(id, ...) and customers(id, order_id)   ← reversed
    ⇒ a customer can have exactly one order. You discover this when
      customer #2 places their second order.

 2. THE JUNCTION TABLE WITH A SURROGATE KEY AND NO UNIQUE
    order_items(id, order_id, product_id, qty)
    ⇒ nothing stops the same product appearing twice on one order,
      and the "obvious" bug report arrives three months later.

 3. THE MISSING INDEX ON THE FOREIGN KEY
    PostgreSQL does NOT create one automatically. Every
    `DELETE FROM customers` then does a full scan of `orders`
    to check the constraint. (Topic 23.)

 4. THE 1:1 THAT ISN'T ENFORCED
    user_profiles(user_id) with a plain FK and no UNIQUE
    ⇒ it's silently 1:N, and one day a user has two profiles.
```

Each is a one-line fix at translation time and a migration afterwards. **The rules exist so you don't have to remember to think about these.**

---

## The physical reality

Every rule has a physical cost you can now price exactly, from Phases 1 and 2:

```
 RULE                          PHYSICAL COST
 ─────────────────────────────────────────────────────────────────────
 Entity → table                heap file + PK index
                               ~28 B/row overhead (Topic 05)
                               PK index ~16 B/row (Topic 11)

 1:N → FK column               +8 B per row
                               + an index you MUST create: ~16 B/row
                               + an FK check on every INSERT/UPDATE of
                                 the child: one index lookup on the parent
                               + a check on every DELETE of the parent:
                                 a lookup on the child index — WHICH IS
                                 WHY THE INDEX IS MANDATORY (Topic 23)

 M:N → junction table          a whole relation: heap + PK index
                               + a second index for the reverse direction
                               + a join at read time (Topic 19)
                               ⇒ typically ~50 B/row all-in

 Weak entity → composite PK    the PK is now (parent_id, discriminator)
                               ⇒ EVERY secondary index carries both
                                 columns → wider keys → lower fanout
                                 → possibly a taller tree (Topic 11)

 1:1 split → 2 tables          a second heap + PK + a join on every read
                               ⇒ worth it ONLY for the reasons in Topic 20

 ⇒ A 12-entity ER diagram with 18 relationships typically becomes
   ~15 tables and ~35 indexes. Every one of those indexes is a write
   tax (Topic 17). Translate deliberately, not reflexively.
```

---

## How it works — step by step

### RULE 1 — Strong entity → table

```
 ┌──────────────┐
 │   CUSTOMER   │
 │ ──────────── │      CREATE TABLE customers (
 │ id (PK)      │  →     id         bigserial PRIMARY KEY,
 │ email        │        email      text NOT NULL,
 │ name         │        name       text NOT NULL,
 │ created_at   │        created_at timestamptz NOT NULL DEFAULT now()
 └──────────────┘      );

 • one table per entity
 • one column per simple attribute
 • the identifier from Q2 becomes the PRIMARY KEY (Topic 22)
 • mandatory attributes (Q6) become NOT NULL
```

### RULE 2 — 1:N → foreign key on the many side

```
 ┌──────────┐ 1        N ┌──────────┐
 │ CUSTOMER ├────────────┤  ORDER   │
 └──────────┘            └──────────┘

 ★ THE FK GOES ON THE "MANY" SIDE. Say it out loud:
   "many ORDERS belong to one CUSTOMER" → orders carries customer_id.

 CREATE TABLE orders (
   id          bigserial   PRIMARY KEY,
   customer_id bigint      NOT NULL REFERENCES customers(id),
   --          ^^^^^^^^ NOT NULL because participation was MANDATORY (Q6)
   placed_at   timestamptz NOT NULL DEFAULT now(),
   status      text        NOT NULL
 );

 ★★★ AND THE INDEX. POSTGRESQL DOES NOT CREATE IT FOR YOU. ★★★
 CREATE INDEX idx_orders_customer ON orders (customer_id);

 WHY IT IS MANDATORY, not optional:
   • every "orders for this customer" query needs it
   • every DELETE or UPDATE of a customers row must verify no child
     rows reference it — WITHOUT the index that is a SEQUENTIAL SCAN
     of orders, per deleted parent row (Topic 23)

 ⚠ WHY NOT put customer_ids on the customer row?
     customers(id, order_ids bigint[])
   ✗ no referential integrity · ✗ no index on the reverse direction
   ✗ updating one order rewrites the whole customer row (Topic 05)
   ✗ violates 1NF (Topic 31)
```

### RULE 3 — 1:1 → one table, or FK + UNIQUE

```
 ┌──────────┐ 1        1 ┌──────────────┐
 │   USER   ├────────────┤ USER_PROFILE │
 └──────────┘            └──────────────┘

 OPTION A — ONE TABLE (the default)
   CREATE TABLE users (id …, email …, bio text, avatar_url text);

 OPTION B — TWO TABLES, when Topic 20's criteria apply
   CREATE TABLE user_profiles (
     user_id bigint PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
     --      ^^^^^^^^^^^^^^^^^ ★ the FK IS the PK. This enforces 1:1
     --                          for free — a PK is unique, so a user
     --                          can have at most one profile.
     bio        text,
     avatar_url text
   );

 ⚠ THE COMMON BUG:
   CREATE TABLE user_profiles (
     id      bigserial PRIMARY KEY,        -- ✗ pointless surrogate
     user_id bigint REFERENCES users(id)   -- ✗ NOT UNIQUE
   );
   ⇒ silently 1:N. A user can have two profiles. Nothing stops it.

 ★ WHICH SIDE OWNS THE KEY?
   The DEPENDENT side — the one that cannot exist alone. A profile
   without a user is meaningless; a user without a profile is fine.
   ⇒ user_profiles carries user_id, not the reverse.
   And if participation is optional on both sides (rare), pick the
   side queried less often, so the common query needs no join.
```

### RULE 4 — M:N → junction table

```
 ┌──────────┐ M        N ┌──────────┐
 │  ORDER   ├────◇───────┤ PRODUCT  │
 └──────────┘            └──────────┘

 CREATE TABLE order_items (
   order_id   bigint NOT NULL REFERENCES orders(id)   ON DELETE CASCADE,
   product_id bigint NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
   PRIMARY KEY (order_id, product_id)
   --          ^^^^^^^^^^^^^^^^^^^^^ ★ THE COMPOSITE PK IS THE POINT.
   --          It enforces "this product appears at most once per order"
   --          for free. A surrogate `id` PK would NOT.
 );
 -- ★ AND the reverse-direction index. The PK covers (order_id, …)
 --   but nothing covers product_id alone (Topic 14: leftmost prefix).
 CREATE INDEX idx_order_items_product ON order_items (product_id);

 ★ NOTE THE ASYMMETRIC CASCADES:
   ON DELETE CASCADE on order_id  — deleting an order removes its lines
   ON DELETE RESTRICT on product_id — you must NOT be able to delete a
     product that appears in historical orders. (Topic 23.)

 ⚠ THE TWO CLASSIC MISTAKES:
   (a) a surrogate `id` PK with no UNIQUE on (order_id, product_id)
       ⇒ duplicates allowed
   (b) forgetting the reverse index
       ⇒ "which orders contain product X?" is a seq scan
```

### RULE 5 — Relationship with attributes → a named entity

```
 ┌──────────┐ M   ◇──────────────◇   N ┌──────────┐
 │  ORDER   ├─────│ quantity     │──────┤ PRODUCT  │
 └──────────┘     │ unit_price   │      └──────────┘
                  ◇──────────────◇

 CREATE TABLE order_lines (
   order_id         bigint  NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
   line_no          smallint NOT NULL,
   product_id       bigint  NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
   quantity         int     NOT NULL CHECK (quantity > 0),
   unit_price_paise bigint  NOT NULL CHECK (unit_price_paise >= 0),
   product_name     text    NOT NULL,      -- ★ SNAPSHOT (Topic 20, case 04)
   PRIMARY KEY (order_id, line_no)
 );
 CREATE UNIQUE INDEX uq_order_lines_product
   ON order_lines (order_id, product_id);   -- ★ still enforce "once per order"
 CREATE INDEX idx_order_lines_product ON order_lines (product_id);

 ★ NOTE THE NAME: `order_lines`, not `order_products`.
   Once a relationship has attributes it is a business object, and
   naming it as one makes every conversation about it clearer.

 ★ WHY (order_id, line_no) AND NOT (order_id, product_id) AS THE PK?
   Because a line has a position on the invoice, and because the same
   product COULD legitimately appear twice (different options, different
   gift recipients). Using line_no as the discriminator keeps that open
   while the UNIQUE index enforces today's rule — which you can drop
   without changing the PK. A PK is forever; a UNIQUE index is a policy.
```

### RULE 6 — Weak entity → composite PK

```
 ┌──────────┐ 1        N ┌───────────────┐
 │  ORDER   ├────────────┤ ORDER_LINE    │  (weak: no independent identity)
 └──────────┘            └───────────────┘

 PRIMARY KEY (order_id, line_no)
             └────┬────┘ └──┬───┘
             parent key   local discriminator

 ✓ identity is inherited from the parent
 ✓ ON DELETE CASCADE is almost always correct (the child cannot
   outlive the parent, by definition)
 ⚠ COST: every secondary index on this table carries BOTH columns
   → wider keys → lower fanout → possibly a taller tree (Topic 11)
 ⚠ AND: any table referencing this one needs a two-column FK.
   ⇒ If many tables will reference it, consider a surrogate PK plus
     a UNIQUE on (order_id, line_no). You keep the constraint and get
     a narrow FK. (Topic 22 covers this trade properly.)
```

### RULE 7 — Multi-valued and composite attributes

```
 MULTI-VALUED: "a customer has several phone numbers"
   ✗ customers(phone1, phone2, phone3)
   ✓ CREATE TABLE customer_phones (
       customer_id bigint NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
       phone       text   NOT NULL,
       kind        text   NOT NULL,      -- 'mobile','work','home'
       is_primary  boolean NOT NULL DEFAULT false,
       PRIMARY KEY (customer_id, phone)
     );
     -- ★ "at most one primary phone per customer" — Topic 13
     CREATE UNIQUE INDEX uq_primary_phone ON customer_phones (customer_id)
       WHERE is_primary;

 COMPOSITE: "address = line1 + line2 + city + pin"
   OPTION A — flatten (when there's exactly one, always read together)
     customers(…, address_line1 text, address_city text, address_pin text)
   OPTION B — its own table (when multi-valued, or shared, or reusable)
     addresses(id, customer_id, line1, city, pin, kind)
   ⇒ Decide with Topic 20's Q4: can a customer have MANY addresses?
     If yes — and it almost always becomes yes — Option B.
```

### RULE 8 — Inheritance / subtypes — the one real decision

```
 ┌────────────────┐
 │    PAYMENT     │   common: id, order_id, amount_paise, status, created_at
 └───────┬────────┘
    ┌────┴─────┬──────────────┐
    ▼          ▼              ▼
 ┌───────┐ ┌────────┐ ┌──────────────┐
 │ CARD  │ │  UPI   │ │ NET_BANKING  │
 │ last4 │ │ vpa    │ │ bank_code    │
 │ brand │ │ app    │ │ ref_no       │
 └───────┘ └────────┘ └──────────────┘

 ═══ STRATEGY A: SINGLE TABLE (one table, nullable subtype columns) ═══
   CREATE TABLE payments (
     id bigserial PRIMARY KEY, order_id bigint NOT NULL,
     amount_paise bigint NOT NULL, method text NOT NULL,
     card_last4 text, card_brand text,        -- NULL unless card
     upi_vpa text, upi_app text,              -- NULL unless upi
     bank_code text, bank_ref_no text,        -- NULL unless netbanking
     CHECK (
       (method='card'       AND card_last4 IS NOT NULL AND upi_vpa IS NULL)
    OR (method='upi'        AND upi_vpa    IS NOT NULL AND card_last4 IS NULL)
    OR (method='netbanking' AND bank_code  IS NOT NULL AND card_last4 IS NULL)
     )
   );
   ✓ no joins · ✓ one place to query · ✓ easy to add a new column
   ✗ nullable sprawl · ✗ the CHECK grows quadratically with subtypes
   ✗ wide rows hurt scans (Topic 04)
   → BEST when: few subtypes, few subtype-specific columns, and you
     always query across all types.

 ═══ STRATEGY B: TABLE PER SUBTYPE (no parent table) ═══
   CREATE TABLE card_payments (id …, order_id …, amount_paise …, last4 …);
   CREATE TABLE upi_payments  (id …, order_id …, amount_paise …, vpa …);
   ✓ no NULLs · ✓ narrow rows · ✓ per-type constraints are natural
   ✗ ★ "all payments for this order" is a UNION across N tables
   ✗ ★ no single FK target — you cannot reference "a payment"
   ✗ common columns duplicated in every table
   → BEST when: subtypes are queried separately and never together,
     and nothing needs to reference the supertype.

 ═══ STRATEGY C: PARENT + CHILD TABLES (class-table inheritance) ═══ ★
   CREATE TABLE payments (
     id bigserial PRIMARY KEY, order_id bigint NOT NULL REFERENCES orders(id),
     amount_paise bigint NOT NULL CHECK (amount_paise > 0),
     method text NOT NULL CHECK (method IN ('card','upi','netbanking')),
     status text NOT NULL, created_at timestamptz NOT NULL DEFAULT now(),
     UNIQUE (id, method)          -- ★ enables the child-side check below
   );
   CREATE TABLE card_payments (
     payment_id bigint PRIMARY KEY REFERENCES payments(id) ON DELETE CASCADE,
     method     text NOT NULL GENERATED ALWAYS AS ('card') STORED,
     last4      text NOT NULL CHECK (last4 ~ '^[0-9]{4}$'),
     brand      text NOT NULL,
     FOREIGN KEY (payment_id, method) REFERENCES payments(id, method)
     -- ★ this composite FK GUARANTEES a card_payments row can only
     --   attach to a payments row whose method is 'card'.
     --   Without it, nothing stops a UPI payment having a card child.
   );
   ✓ no NULLs · ✓ a single FK target · ✓ per-subtype constraints
   ✓ "all payments" is one scan of the parent
   ✗ one join to get subtype details
   ✗ ⚠ nothing forces a child row to EXIST — "every card payment has a
     card_payments row" must be enforced in the application or by a
     deferred constraint trigger
   → BEST when: you need a single FK target, subtypes have several
     distinct columns, and you query both across and within types.
     ★ THE DEFAULT CHOICE for most real systems.
```

---

## Concept breakdown

```
THE EIGHT RULES
│
├── 1. strong entity        → table + PK
├── 2. 1:N                  → FK on the MANY side + AN INDEX ON IT
├── 3. 1:1                  → one table, OR FK-as-PK on the dependent side
├── 4. M:N                  → junction table, PK = (fk1, fk2)
├── 5. relationship + attrs → a NAMED entity table
├── 6. weak entity          → composite PK (parent_key, discriminator)
├── 7. multi-valued attr    → its own table
└── 8. inheritance          → single-table / table-per-subtype / parent+child

THE FOUR THINGS PEOPLE FORGET, EVERY TIME
│
├── ① AN INDEX ON EVERY FOREIGN KEY
│      PostgreSQL creates one for a PK and for UNIQUE — NEVER for an FK.
│      Without it: parent DELETE/UPDATE does a seq scan of the child.
├── ② UNIQUE ON A 1:1 DEPENDENT SIDE
│      or make the FK the PK, which gives it for free.
├── ③ THE COMPOSITE PK ON A JUNCTION TABLE
│      a surrogate `id` alone permits duplicates.
└── ④ THE REVERSE-DIRECTION INDEX ON A JUNCTION TABLE
       PK (a,b) serves lookups by `a`, never by `b` alone (Topic 14).

NAMING — it matters more than it looks
│
├── tables: plural nouns          orders · order_lines · customer_phones
├── junction with attributes: a real business noun
│      loans · enrolments · assignments · order_lines
│      NOT member_books · student_courses · order_products
├── FK columns: <singular_parent>_id     customer_id · product_id
├── indexes: idx_<table>_<cols>          idx_orders_customer_created
└── constraints: uq_ / fk_ / ck_ / ex_ prefixes
   ⇒ consistent names make an error message tell you where to look.

DEFERRABLE CONSTRAINTS — for circular references
│   "every order has a primary shipment; every shipment has an order"
│   → orders.primary_shipment_id and shipments.order_id reference
│     each other. Neither can be inserted first.
└── ALTER TABLE orders ADD CONSTRAINT fk_primary_shipment
      FOREIGN KEY (primary_shipment_id) REFERENCES shipments(id)
      DEFERRABLE INITIALLY DEFERRED;
    ⇒ checked at COMMIT, not per statement. Use sparingly — it hides
      errors until the end of the transaction.
```

---

## Diagrams

**Diagram 1 — big picture: the translation table**

```
 ER DIAGRAM                          RELATIONAL SCHEMA
 ─────────────────────────           ─────────────────────────────────

 ┌──────────┐                        CREATE TABLE customers (
 │ CUSTOMER │            ──────▶       id bigserial PRIMARY KEY, …
 └──────────┘                        );

 ┌──────┐1     N┌──────┐             CREATE TABLE orders (
 │ CUST ├───────┤ ORDER│  ──────▶      customer_id bigint NOT NULL
 └──────┘       └──────┘                 REFERENCES customers(id), …
                                     );
                                     CREATE INDEX … ON orders(customer_id);
                                                        ↑ NEVER AUTOMATIC

 ┌──────┐M     N┌──────┐             CREATE TABLE order_items (
 │ ORDER├───◇───┤ PROD │  ──────▶      order_id bigint REFERENCES …,
 └──────┘       └──────┘               product_id bigint REFERENCES …,
                                       PRIMARY KEY (order_id, product_id)
                                     );
                                     CREATE INDEX … ON order_items(product_id);

 ┌──────┐M ◇───◇ N┌──────┐           CREATE TABLE order_lines (
 │ ORDER├──│ qty│──┤ PROD │ ──────▶     order_id …, line_no …,
 └──────┘  ◇───◇   └──────┘             product_id …, quantity …,
                                        PRIMARY KEY (order_id, line_no)
                                      );

 ┌──────┐1     1┌──────┐             CREATE TABLE profiles (
 │ USER ├───────┤ PROF │  ──────▶      user_id bigint PRIMARY KEY
 └──────┘       └──────┘                 REFERENCES users(id), …
                                     );   ↑ FK IS the PK ⇒ 1:1 enforced
```

**Diagram 2 — data flow: where each rule's cost lands**

```
   ER CONSTRUCT              WRITE PATH COST            READ PATH COST
 ────────────────────────────────────────────────────────────────────────
   entity → table            heap insert + PK insert    1 page (by PK)

   1:N → FK + index          + FK check (1 lookup on    join: nested loop
                               the PARENT index)          or hash (T19)
                             + child index insert
                             ⇒ ~2× the write cost

   M:N → junction            2 FK checks + 2 index      2 joins
                               inserts per row
                             ⇒ ~3× the write cost

   weak → composite PK       wider index keys →         same
                               lower fanout (T11)

   1:1 split → 2 tables      2 heap inserts,            + 1 join on
                               2 PK inserts               every read
                             ⇒ only worth it for
                               Topic 20's reasons

 ★ A 15-table schema typically means ~35 indexes. At ~80 B of WAL each
   per insert (Topic 17), that's the write budget you just committed to.
```

**Diagram 3 — before/after: the four forgotten things**

```
 WHAT PEOPLE WRITE                    WHAT IT SHOULD BE
 ────────────────────────────         ────────────────────────────────────
 CREATE TABLE orders (                CREATE TABLE orders (
   id bigserial PRIMARY KEY,            id bigserial PRIMARY KEY,
   customer_id bigint                   customer_id bigint NOT NULL
     REFERENCES customers(id)             REFERENCES customers(id)
 );                                       ON DELETE RESTRICT
                                      );
                                      CREATE INDEX idx_orders_customer
                                        ON orders(customer_id);      ← ①

 CREATE TABLE profiles (              CREATE TABLE profiles (
   id bigserial PRIMARY KEY,            user_id bigint PRIMARY KEY
   user_id bigint                         REFERENCES users(id)
     REFERENCES users(id)                 ON DELETE CASCADE,     ← ②
 );                                   );

 CREATE TABLE order_items (           CREATE TABLE order_items (
   id bigserial PRIMARY KEY,            order_id bigint NOT NULL …,
   order_id bigint …,                   product_id bigint NOT NULL …,
   product_id bigint …                  PRIMARY KEY (order_id,
 );                                                  product_id)  ← ③
                                      );
                                      CREATE INDEX idx_oi_product
                                        ON order_items(product_id); ← ④

 ⇒ ① missing FK index: parent DELETE = seq scan of the child
   ② missing UNIQUE: the "1:1" is silently 1:N
   ③ surrogate PK on a junction: duplicates permitted
   ④ missing reverse index: "orders containing product X" = seq scan
```

---

## Example 1 — basic

Translate the library model from Topic 20, applying every rule.

```sql
-- ═══ RULE 1: strong entities ═══
CREATE TABLE members (
  id         bigserial   PRIMARY KEY,
  number     text        NOT NULL UNIQUE,          -- the natural key from Q2
  name       text        NOT NULL,
  email      text        NOT NULL,
  joined_at  timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX uq_members_email ON members (lower(email));  -- Topic 13

CREATE TABLE authors (
  id   bigserial PRIMARY KEY,
  name text      NOT NULL,
  born date
);

CREATE TABLE books (
  id             bigserial PRIMARY KEY,
  isbn           text      NOT NULL UNIQUE,
  title          text      NOT NULL,
  published_year smallint
);

-- ═══ RULE 4: M:N (authorship) — no attributes, so a pure junction ═══
CREATE TABLE book_authors (
  book_id   bigint   NOT NULL REFERENCES books(id)   ON DELETE CASCADE,
  author_id bigint   NOT NULL REFERENCES authors(id) ON DELETE RESTRICT,
  position  smallint NOT NULL DEFAULT 1,     -- first author, second author…
  PRIMARY KEY (book_id, author_id)           -- ★ RULE 4: composite PK
);
CREATE INDEX idx_book_authors_author ON book_authors (author_id);  -- ★ reverse

-- ═══ RULE 2: 1:N (a book has many copies) ═══
CREATE TABLE copies (
  id          bigserial   PRIMARY KEY,
  barcode     text        NOT NULL UNIQUE,          -- a real natural key
  book_id     bigint      NOT NULL REFERENCES books(id) ON DELETE RESTRICT,
  acquired_at date        NOT NULL DEFAULT current_date,
  condition   text        NOT NULL DEFAULT 'good'
              CHECK (condition IN ('good','fair','poor','withdrawn'))
);
CREATE INDEX idx_copies_book ON copies (book_id);   -- ★ RULE 2: the FK index

-- ═══ RULE 5: the relationship WITH attributes → a named entity ═══
CREATE TABLE loans (
  id          bigserial   PRIMARY KEY,
  member_id   bigint      NOT NULL REFERENCES members(id) ON DELETE RESTRICT,
  copy_id     bigint      NOT NULL REFERENCES copies(id)  ON DELETE RESTRICT,
  borrowed_at timestamptz NOT NULL DEFAULT now(),
  due_at      timestamptz NOT NULL,
  returned_at timestamptz NULL,
  CHECK (due_at > borrowed_at),
  CHECK (returned_at IS NULL OR returned_at >= borrowed_at)
);
CREATE INDEX idx_loans_member ON loans (member_id, borrowed_at DESC);
CREATE INDEX idx_loans_copy   ON loans (copy_id, borrowed_at DESC);

-- ★ THE TEMPORAL RULE from Topic 20: "only one member at a time".
--   This is NOT cardinality — it's an exclusion constraint (Topics 16, 24).
CREATE INDEX idx_loans_open ON loans (copy_id) WHERE returned_at IS NULL;
CREATE UNIQUE INDEX uq_loans_one_open_per_copy
  ON loans (copy_id) WHERE returned_at IS NULL;   -- ★ at most one open loan

-- for the overdue sweep (Topic 13: partial index on the rare case)
CREATE INDEX idx_loans_overdue ON loans (due_at) WHERE returned_at IS NULL;
```

**Now prove the model answers the questions the naive one couldn't:**

```sql
-- 1. Which copies are out right now?
SELECT c.barcode, b.title, m.name, l.due_at
FROM loans l JOIN copies c ON c.id=l.copy_id JOIN books b ON b.id=c.book_id
JOIN members m ON m.id=l.member_id WHERE l.returned_at IS NULL;

-- 2. Who has had copy C001, ever?      ← the naive model could not answer this
SELECT m.name, l.borrowed_at, l.returned_at
FROM loans l JOIN members m ON m.id=l.member_id
JOIN copies c ON c.id=l.copy_id WHERE c.barcode='C001'
ORDER BY l.borrowed_at DESC;

-- 3. Average loan duration               ← nor this
SELECT avg(returned_at - borrowed_at) FROM loans WHERE returned_at IS NOT NULL;

-- 4. Books never borrowed                ← nor this
SELECT b.title FROM books b
WHERE NOT EXISTS (                         -- ★ NOT EXISTS, not NOT IN (T19)
  SELECT 1 FROM copies c JOIN loans l ON l.copy_id=c.id WHERE c.book_id=b.id);

-- 5. Enforce "one member at a time" — prove it
INSERT INTO loans (member_id, copy_id, due_at)
VALUES (1, 1, now()+interval '14 days');
INSERT INTO loans (member_id, copy_id, due_at)
VALUES (2, 1, now()+interval '14 days');
-- ERROR: duplicate key value violates unique constraint
--        "uq_loans_one_open_per_copy"    ★ enforced by the DATABASE
```

**Count the objects created:** 6 tables, 6 primary keys, 8 foreign keys, 11 indexes. **Every index maps to a rule or a named query** — none is speculative (Topic 17's discipline).

---

## Example 2 — production scenario

**The situation.** You're translating the food-delivery ER model from Topic 20. It has 12 entities, and one requirement makes it non-mechanical: *"reproduce exactly what was ordered, at what price, even if the menu changes."*

**Step 1 — the mechanical part.**

```sql
CREATE TABLE restaurants (
  id bigserial PRIMARY KEY, name text NOT NULL, city text NOT NULL
);

CREATE TABLE menus (
  id            bigserial PRIMARY KEY,
  restaurant_id bigint NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  name          text   NOT NULL,                  -- 'Lunch','Dinner'
  active_from   time   NOT NULL,
  active_to     time   NOT NULL,
  CHECK (active_to > active_from)
);
CREATE INDEX idx_menus_restaurant ON menus (restaurant_id);

CREATE TABLE sections (
  id       bigserial PRIMARY KEY,
  menu_id  bigint   NOT NULL REFERENCES menus(id) ON DELETE CASCADE,
  name     text     NOT NULL,
  position smallint NOT NULL,
  UNIQUE (menu_id, position)
);
CREATE INDEX idx_sections_menu ON sections (menu_id);

CREATE TABLE dishes (
  id                bigserial PRIMARY KEY,
  restaurant_id     bigint NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  name              text   NOT NULL,
  base_price_paise  bigint NOT NULL CHECK (base_price_paise >= 0),
  is_available      boolean NOT NULL DEFAULT true
);
CREATE INDEX idx_dishes_restaurant ON dishes (restaurant_id);

-- ★ RULE 5: section↔dish is M:N WITH attributes → a named entity
CREATE TABLE section_dishes (
  section_id    bigint   NOT NULL REFERENCES sections(id) ON DELETE CASCADE,
  dish_id       bigint   NOT NULL REFERENCES dishes(id)   ON DELETE CASCADE,
  display_order smallint NOT NULL,
  is_featured   boolean  NOT NULL DEFAULT false,
  PRIMARY KEY (section_id, dish_id)
);
CREATE INDEX idx_section_dishes_dish ON section_dishes (dish_id);  -- reverse

CREATE TABLE options (
  id            bigserial PRIMARY KEY,
  restaurant_id bigint NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  group_name    text   NOT NULL,       -- 'Size','Spice Level'
  name          text   NOT NULL,       -- 'Large','Extra Hot'
  UNIQUE (restaurant_id, group_name, name)
);

-- ★ RULE 5 again: dish↔option is M:N WITH a price delta
CREATE TABLE dish_options (
  dish_id           bigint  NOT NULL REFERENCES dishes(id)   ON DELETE CASCADE,
  option_id         bigint  NOT NULL REFERENCES options(id)  ON DELETE CASCADE,
  price_delta_paise bigint  NOT NULL DEFAULT 0,
  is_default        boolean NOT NULL DEFAULT false,
  PRIMARY KEY (dish_id, option_id)
);
CREATE INDEX idx_dish_options_option ON dish_options (option_id);
-- ★ "at most one default per dish per option group" — Topic 13
CREATE UNIQUE INDEX uq_dish_default_option ON dish_options (dish_id, option_id)
  WHERE is_default;
```

**Step 2 — where the mechanical rules stop.**

The naive translation of "order contains dish" would be:

```sql
-- ✗ MECHANICALLY CORRECT, BUSINESS-INCORRECT
CREATE TABLE order_lines (
  order_id bigint NOT NULL REFERENCES orders(id),
  line_no  smallint NOT NULL,
  dish_id  bigint NOT NULL REFERENCES dishes(id),
  quantity int NOT NULL,
  PRIMARY KEY (order_id, line_no)
);
-- price comes from dishes.base_price_paise at read time
```

**This satisfies every rule and breaks the requirement.** When the restaurant raises prices, every historical order's total changes — case study 04's reproducibility failure (I2), and case study 03's "store events, derive state" lesson.

**Step 3 — the snapshot, and the rule that governs it.**

```sql
CREATE TABLE orders (
  id              bigserial   PRIMARY KEY,
  customer_id     bigint      NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,
  restaurant_id   bigint      NOT NULL REFERENCES restaurants(id) ON DELETE RESTRICT,
  rider_id        bigint      NULL REFERENCES riders(id) ON DELETE SET NULL,
  --              ^^^^ NULL: participation was OPTIONAL at creation (Q6)
  address_id      bigint      NOT NULL REFERENCES addresses(id) ON DELETE RESTRICT,
  status          text        NOT NULL,
  subtotal_paise  bigint      NOT NULL,
  total_paise     bigint      NOT NULL,
  placed_at       timestamptz NOT NULL DEFAULT now(),
  idempotency_key uuid        NOT NULL
);
CREATE UNIQUE INDEX uq_orders_idem ON orders (idempotency_key);
CREATE INDEX idx_orders_customer ON orders (customer_id, placed_at DESC);
CREATE INDEX idx_orders_restaurant ON orders (restaurant_id, placed_at DESC);
CREATE INDEX idx_orders_rider ON orders (rider_id) WHERE rider_id IS NOT NULL;

CREATE TABLE order_lines (
  order_id          bigint   NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  line_no           smallint NOT NULL,

  -- ── THE REFERENCE: for analytics. "How many Paneer Tikkas did we sell?"
  dish_id           bigint   NOT NULL REFERENCES dishes(id) ON DELETE RESTRICT,
  --                ^^^^^^^^^ RESTRICT: you must never delete a dish that
  --                          appears in order history

  -- ── THE SNAPSHOT: for the receipt. Frozen at order time. NEVER updated.
  dish_name         text     NOT NULL,
  unit_price_paise  bigint   NOT NULL CHECK (unit_price_paise >= 0),
  quantity          int      NOT NULL CHECK (quantity > 0),
  line_total_paise  bigint   NOT NULL CHECK (line_total_paise >= 0),

  PRIMARY KEY (order_id, line_no)
);
CREATE INDEX idx_order_lines_dish ON order_lines (dish_id);   -- for analytics

CREATE TABLE order_line_options (
  order_id          bigint   NOT NULL,
  line_no           smallint NOT NULL,
  option_id         bigint   NOT NULL REFERENCES options(id) ON DELETE RESTRICT,
  option_name       text     NOT NULL,          -- ★ snapshot
  price_delta_paise bigint   NOT NULL,          -- ★ snapshot
  PRIMARY KEY (order_id, line_no, option_id),
  FOREIGN KEY (order_id, line_no)
    REFERENCES order_lines(order_id, line_no) ON DELETE CASCADE
  --  ^^^^^^^^^^^^^^^^^^ ★ a COMPOSITE FK, because order_lines has a
  --                       composite PK. This is Rule 6's cost, made real.
);
```

**Step 4 — the rule that decides reference vs snapshot.**

```
 STORE A REFERENCE (FK) when the question is about the CURRENT state
 or about aggregating across entities:
   "how many Paneer Tikkas did we sell in March?"
   "which dishes are most ordered?"
   → needs dish_id, joined to the live dishes table

 STORE A SNAPSHOT (copied value) when the question is about a
 HISTORICAL FACT that must never change:
   "what does this receipt say?"
   "what did the customer agree to pay?"
   → needs dish_name and unit_price_paise as they were AT ORDER TIME

 ★ STORE BOTH WHEN BOTH QUESTIONS EXIST. They are not redundant —
   they answer different questions, and dropping either loses one.

 ⚠ THIS IS NOT A NORMALISATION VIOLATION.
   `unit_price_paise` on the order line is NOT functionally dependent
   on `dish_id` — it depends on (dish_id, order_time). It is a fact
   about the ORDER, not about the DISH. (Topic 30 formalises this;
   Topic 38 explains why it isn't over-denormalisation.)
```

**Step 5 — inheritance: apply Rule 8.**

Payments have card / UPI / net-banking subtypes. Which strategy?

```
 REQUIREMENTS CHECK:
   • Does anything need to reference "a payment" generically?
       YES — refunds reference a payment, the ledger references a payment.
       ⇒ rules out Strategy B (table-per-subtype, no parent).
   • Do subtypes have several distinct columns?
       YES — 2–4 each, and more coming (wallet, EMI, BNPL).
       ⇒ Strategy A would mean 12+ nullable columns and a CHECK that
         grows quadratically.
   • Do you query across types?
       YES — "all payments for this order", "today's revenue".
       ⇒ Strategy B's UNION is unacceptable.

 ⇒ STRATEGY C (parent + child), as in Rule 8.
```

```sql
CREATE TABLE payments (
  id            bigserial   PRIMARY KEY,
  order_id      bigint      NOT NULL REFERENCES orders(id) ON DELETE RESTRICT,
  amount_paise  bigint      NOT NULL CHECK (amount_paise > 0),
  method        text        NOT NULL CHECK (method IN ('card','upi','netbanking')),
  status        text        NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now(),
  UNIQUE (id, method)            -- ★ target for the child composite FK
);
CREATE INDEX idx_payments_order ON payments (order_id);

CREATE TABLE card_payments (
  payment_id bigint PRIMARY KEY REFERENCES payments(id) ON DELETE CASCADE,
  method     text   NOT NULL GENERATED ALWAYS AS ('card') STORED,
  last4      text   NOT NULL CHECK (last4 ~ '^[0-9]{4}$'),
  brand      text   NOT NULL,
  FOREIGN KEY (payment_id, method) REFERENCES payments(id, method)
);

CREATE TABLE upi_payments (
  payment_id bigint PRIMARY KEY REFERENCES payments(id) ON DELETE CASCADE,
  method     text   NOT NULL GENERATED ALWAYS AS ('upi') STORED,
  vpa        text   NOT NULL,
  app        text   NOT NULL,
  FOREIGN KEY (payment_id, method) REFERENCES payments(id, method)
);
```

The composite FK is the clever part: `card_payments.method` is a generated column always equal to `'card'`, and the FK `(payment_id, method) → payments(id, method)` therefore **guarantees** that a `card_payments` row can only attach to a payment whose method is `'card'`. Without it, nothing stops a UPI payment having a card child row.

**Step 6 — the object count, and the honesty check.**

```sql
SELECT count(*) FILTER (WHERE relkind='r') AS tables,
       count(*) FILTER (WHERE relkind='i') AS indexes
FROM pg_class WHERE relnamespace='public'::regnamespace;
```
```
 tables | indexes
--------+---------
     16 |      41
```

**41 indexes.** Each one costs ~80 B of WAL per insert (Topic 17). Before shipping, audit them: every index must map either to a rule (PK, FK, unique constraint) or to a named query in the access-pattern table. In this schema, 24 are rule-mandated and 17 serve named queries. **Zero are speculative.** That's the discipline.

---

## Common mistakes

**1. No index on a foreign key.**
- *Symptom:* `DELETE FROM customers WHERE id=7` takes 40 seconds.
- *Engine-level why:* PostgreSQL creates an index for `PRIMARY KEY` and `UNIQUE`, but **never** for `REFERENCES`. Every parent delete/update must verify no children reference it — a sequential scan without the index.
- *Diagnose:*
  ```sql
  SELECT c.conrelid::regclass AS child, a.attname AS fk_column
  FROM pg_constraint c JOIN pg_attribute a
    ON a.attrelid=c.conrelid AND a.attnum = ANY(c.conkey)
  WHERE c.contype='f' AND NOT EXISTS (
    SELECT 1 FROM pg_index i WHERE i.indrelid=c.conrelid
      AND (i.indkey::int2[])[0] = a.attnum);
  ```
- *Fix:* index every FK column. Make it part of the translation ritual.

**2. Surrogate PK on a junction table with no UNIQUE.**
- *Symptom:* the same product twice on one order.
- *Fix:* `PRIMARY KEY (order_id, product_id)`, or a surrogate PK **plus** `UNIQUE (order_id, product_id)`.

**3. 1:1 without UNIQUE.**
- *Symptom:* a user with two profiles.
- *Fix:* make the FK the primary key of the dependent table. Free, and impossible to forget later.

**4. The FK on the wrong side of 1:N.**
- *Symptom:* a customer can have exactly one order.
- *Fix:* say it aloud — "many *orders* belong to one *customer*" — and put the FK on the side the sentence starts with.

**5. Missing the reverse index on a junction table.**
- *Symptom:* "which orders contain product X?" is a seq scan.
- *Engine-level why:* `PRIMARY KEY (order_id, product_id)` serves lookups by `order_id`; `product_id` alone is not a leftmost prefix (Topic 14).
- *Fix:* `CREATE INDEX ON order_items (product_id);`

**6. Choosing single-table inheritance because it's fewer tables.**
- *Symptom:* 40 columns, 30 of them NULL on any row, and a `CHECK` nobody can read.
- *Fix:* apply Rule 8's criteria. Strategy C is usually right for anything with 2+ subtype columns and a need for a generic FK target.

**7. Storing only a reference where a snapshot is needed (or vice versa).**
- *Symptom:* historical invoices change when prices change; or you can't answer "how many did we sell?" because you only stored the name.
- *Fix:* apply the reference-vs-snapshot rule. When both questions exist, **store both**. It is not redundancy.

**8. Cascade chosen by habit.**
- *Symptom:* deleting a product silently deletes ten years of order history.
- *Fix:* `CASCADE` only when the child genuinely cannot exist without the parent (weak entities). `RESTRICT` for anything historical. (Topic 23.)

---

## Hands-on proof

**PROVE IT #1 — PostgreSQL does not index your foreign keys.**
```sql
CREATE TABLE parent (id bigserial PRIMARY KEY);
CREATE TABLE child (id bigserial PRIMARY KEY,
                    parent_id bigint REFERENCES parent(id));
SELECT indexname, indexdef FROM pg_indexes WHERE tablename='child';
```
```
   indexname   |                      indexdef
---------------+-----------------------------------------------------
 child_pkey    | CREATE UNIQUE INDEX child_pkey ON child USING btree (id)
```
**Only the PK.** Nothing on `parent_id`.

**PROVE IT #2 — what that costs.**
```sql
INSERT INTO parent SELECT generate_series(1,1000);
INSERT INTO child (parent_id) SELECT (random()*1000)::int+1
FROM generate_series(1,2000000);
VACUUM ANALYZE child;

\timing on
DELETE FROM parent WHERE id = 500;      -- Time: 412.882 ms
CREATE INDEX idx_child_parent ON child (parent_id);
DELETE FROM parent WHERE id = 501;      -- Time: 0.914 ms   ← 452×
```

**PROVE IT #3 — the junction table without a composite PK.**
```sql
CREATE TABLE oi_bad (id bigserial PRIMARY KEY, order_id bigint, product_id bigint);
INSERT INTO oi_bad (order_id, product_id) VALUES (1, 88), (1, 88), (1, 88);
SELECT * FROM oi_bad;      -- ★ three identical rows. Nothing stopped it.

CREATE TABLE oi_good (order_id bigint, product_id bigint,
                      PRIMARY KEY (order_id, product_id));
INSERT INTO oi_good VALUES (1,88);
INSERT INTO oi_good VALUES (1,88);
-- ERROR: duplicate key value violates unique constraint "oi_good_pkey"
```

**PROVE IT #4 — the 1:1 that isn't.**
```sql
CREATE TABLE prof_bad (id bigserial PRIMARY KEY, user_id bigint REFERENCES parent(id));
INSERT INTO prof_bad (user_id) VALUES (1),(1),(1);   -- ★ three profiles, one user

CREATE TABLE prof_good (user_id bigint PRIMARY KEY REFERENCES parent(id));
INSERT INTO prof_good VALUES (1);
INSERT INTO prof_good VALUES (1);
-- ERROR: duplicate key value violates unique constraint "prof_good_pkey"
```

**PROVE IT #5 — the composite FK guards subtype integrity.**
```sql
INSERT INTO payments (order_id, amount_paise, method, status)
VALUES (1, 50000, 'upi', 'captured') RETURNING id;   -- say id = 9
INSERT INTO card_payments (payment_id, last4, brand) VALUES (9, '4242', 'visa');
-- ERROR: insert or update on table "card_payments" violates foreign key
--        constraint "card_payments_payment_id_method_fkey"
-- ★ A card child cannot attach to a UPI payment. Enforced structurally.
```

**PROVE IT #6 — find every unindexed FK in your database.**
```sql
SELECT c.conrelid::regclass AS child_table,
       (SELECT string_agg(a.attname, ',' ORDER BY x.ord)
          FROM unnest(c.conkey) WITH ORDINALITY AS x(attnum, ord)
          JOIN pg_attribute a ON a.attrelid=c.conrelid AND a.attnum=x.attnum
       ) AS fk_columns,
       c.confrelid::regclass AS parent_table,
       pg_size_pretty(pg_relation_size(c.conrelid)) AS child_size
FROM pg_constraint c
WHERE c.contype = 'f'
  AND NOT EXISTS (
    SELECT 1 FROM pg_index i
    WHERE i.indrelid = c.conrelid
      AND (i.indkey::int2[])[0:array_length(c.conkey,1)-1] = c.conkey)
ORDER BY pg_relation_size(c.conrelid) DESC;
```
**Run this on your production database today.** Most schemas have several.

---

## The design decision framework

```
THE TRANSLATION CHECKLIST — run it for every entity, every time:

  □ 1. Table created, plural name
  □ 2. Primary key chosen (Topic 22)
  □ 3. NOT NULL on every attribute Q6 said was mandatory
  □ 4. CHECK constraints for domain rules (positive amounts, valid enums)
  □ 5. UNIQUE on every natural key
  □ 6. FOREIGN KEY on the many side of every 1:N
  □ 7. ★ AN INDEX ON EVERY FOREIGN KEY COLUMN
  □ 8. ON DELETE chosen DELIBERATELY (CASCADE/RESTRICT/SET NULL)
  □ 9. Junction tables have a COMPOSITE PK
  □ 10. Junction tables have a REVERSE-DIRECTION INDEX
  □ 11. 1:1 dependent side has the FK AS its PK (or a UNIQUE)
  □ 12. Every index maps to a rule or a named query — no speculation

INHERITANCE — pick with these questions:
  Does anything need a generic FK to the supertype?
    YES → rules out table-per-subtype
  How many subtype-specific columns?
    0–2  → single table with a CHECK
    3+   → parent + child
  Do you query across all subtypes together?
    YES → rules out table-per-subtype
  ⇒ DEFAULT: parent + child (Strategy C), with the composite-FK guard.

REFERENCE vs SNAPSHOT:
  question about CURRENT state or aggregation  → reference (FK)
  question about a HISTORICAL fact             → snapshot (copied value)
  BOTH questions exist                         → ★ store BOTH
  ⇒ Not redundancy. Different questions. (Topics 30, 38, case study 04.)

THE SIGNAL TO LOOK FOR:
  After translating, run the unindexed-FK query (PROVE IT #6). Then:

      SELECT relname, count(*) AS index_count
      FROM pg_stat_user_indexes GROUP BY 1 ORDER BY 2 DESC;

  • a table with more indexes than columns   → probably over-indexed
  • an FK with no index                      → a latent DELETE outage
  • a junction table with a surrogate PK and
    no UNIQUE on the pair                    → duplicates are possible
  • a "1:1" table whose FK isn't the PK      → it's actually 1:N

  ★ AND: for every table, ask "can I express all 10 business questions
    from Topic 20?" If not, the translation lost something the diagram had.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Translate this ER fragment to DDL, applying all twelve checklist items. State which rule produces each object.
> A `playlist` belongs to one `user` and contains many `tracks`; a `track` appears in many playlists, and each appearance has a position. A `track` belongs to one `album`; an `album` has many `tracks` and one `artist`.

Then run PROVE IT #6 against your result and confirm it returns zero rows.

### Exercise 2 — medium (apply it)
Take the hospital model you built in Topic 20's Exercise 2 and translate it fully. Then:
(a) list every foreign key and justify its `ON DELETE` action,
(b) identify which relationships needed a snapshot rather than a reference, and why,
(c) apply Rule 8 to `prescription` (which has medicine-type subtypes) and justify your strategy choice against all three criteria,
(d) count your tables and indexes, and justify every index against a rule or a named query,
(e) write the five business queries from Topic 20 against your schema and confirm each is expressible.

### Exercise 3 — hard (production simulation)
Here is a real ER model for a B2B logistics platform:

```
 CUSTOMER 1─N CONTRACT N─1 SERVICE_LEVEL
 CONTRACT 1─N SHIPMENT
 SHIPMENT N─1 ORIGIN_HUB (hub)   SHIPMENT N─1 DEST_HUB (hub)
 SHIPMENT 1─N PARCEL
 PARCEL   1─N SCAN_EVENT  N─1 HUB
 PARCEL   N─M HAZARD_CLASS  (with: quantity, packaging_code)
 SHIPMENT N─1 CARRIER
 CARRIER  N─M HUB  (with: cutoff_time, service_days)
 SCAN_EVENT has subtypes: PICKUP, TRANSIT, DELIVERY_ATTEMPT, EXCEPTION
   — each with 3–5 distinct attributes; ~40M events/day
 Requirement: "a shipment's contracted price and service level must be
 reproducible for 7 years, even after the contract is renegotiated."
```

(a) Produce the complete DDL. Every table, PK, FK, index, constraint.
(b) `SHIPMENT` has **two** foreign keys to `HUB`. Show how to declare and name both, and what indexes are needed.
(c) Apply Rule 8 to `SCAN_EVENT`. Justify your strategy against all three criteria, and account for the 40M events/day volume in your choice.
(d) The 7-year reproducibility requirement changes at least two tables. Identify them and show the reference-vs-snapshot split, with the reasoning for each field.
(e) `PARCEL N─M HAZARD_CLASS` has attributes. Show the table, its PK, both indexes, and the cascade choices with justification.
(f) Count your indexes. For each, state whether it is rule-mandated or query-driven, and name the query for the latter.
(g) `SCAN_EVENT` is 40M rows/day. Which of your translation decisions would you revisit given that volume, and what would you change? (Partitioning, key choice, index count.)
(h) Write the unindexed-FK detection query and confirm your schema passes.

---

## Mental model checkpoint

1. State the eight translation rules from memory.
2. Which side of a 1:N gets the foreign key? Give the sentence that makes it unambiguous.
3. Why does a junction table need a *composite* primary key rather than a surrogate `id`?
4. What does PostgreSQL automatically index, and what does it never index? What's the consequence?
5. How do you enforce 1:1 in the schema? What happens if you forget?
6. Name the three inheritance strategies, one advantage and one drawback of each, and which is the usual default.
7. When do you store a reference *and* a snapshot of the same fact? Why is that not a normalisation violation?

---

## Quick reference card

**The eight rules**

| ER | Relational |
|---|---|
| Strong entity | table + PK |
| Weak entity | composite PK `(parent_key, discriminator)` |
| 1:N | FK on the **many** side + **an index on it** |
| 1:1 | one table, or FK-as-PK on the dependent side |
| M:N | junction, `PRIMARY KEY (fk1, fk2)` + reverse index |
| Relationship + attributes | a **named** entity table |
| Multi-valued attribute | its own table |
| Inheritance | single-table / table-per-subtype / **parent+child** |

**What PostgreSQL indexes automatically**

| Constraint | Index created? |
|---|---|
| `PRIMARY KEY` | ✓ unique |
| `UNIQUE` | ✓ unique |
| `EXCLUDE` | ✓ GiST |
| **`REFERENCES` (FK)** | ✗ **never — you must create it** |

**The four things people forget**
1. An index on every FK column.
2. `UNIQUE` (or FK-as-PK) on a 1:1 dependent side.
3. A composite PK on a junction table.
4. The reverse-direction index on a junction table.

**Naming**

| Object | Convention |
|---|---|
| table | plural noun — `orders`, `order_lines` |
| junction with attributes | a real business noun — `loans`, not `member_books` |
| FK column | `<singular_parent>_id` |
| index | `idx_<table>_<cols>` |
| unique | `uq_<table>_<cols>` |

**Reference vs snapshot:** current state or aggregation → reference. Historical fact → snapshot. Both questions → **both**.

---

## When would I use this at work?

1. **Writing any migration.** The twelve-item checklist takes two minutes and catches the four forgotten things — each of which is a production incident waiting for the right amount of data.

2. **Reviewing a teammate's migration.** "Is there an index on that foreign key?" and "what's the PK on that junction table?" catch the two most common defects in about ten seconds, and you can point at the rule rather than at taste.

3. **Auditing an inherited schema.** Running the unindexed-FK query on a legacy database usually finds several — each one a `DELETE` that will time out the first time someone runs it on a large parent table.

---

## Connected topics

**Understand before this:** 20 (the ER diagram this translates), 11–14 (why an index on an FK matters, and what a composite key costs).

**This unlocks:**
- **22** — primary keys: which key type for each table you just created
- **23** — foreign keys: the `ON DELETE` decisions made here, in full
- **24** — constraints: `CHECK`, `UNIQUE`, `EXCLUDE` as business rules
- **25/26** — data types for every column you just declared
- **27** — the antipatterns this chapter's rules prevent
- **29–38** — normalisation, which verifies the result is correct
