# 20 — Entity-Relationship Modelling
## Phase: Database Design

---

## ELI5 — The Simple Analogy

Someone describes their business to you over chai:

> "We sell products. Customers place orders. An order can have several products in it, and each line says how many. Products belong to a category. We ship from warehouses, and a product can be stocked in several warehouses."

That's a paragraph. Your job is to turn it into a diagram of **boxes and lines** before you touch a keyboard.

Here's the trick that makes it mechanical rather than mystical: **the nouns become boxes, and the verbs become lines.** "Customers **place** orders" — two boxes, one line. "An order **has** products, and each line says **how many**" — that "how many" doesn't belong to the order or the product; it belongs to the *line between them*. That's the whole insight, and it's the one people miss.

Then you ask one question about every line — *"how many of each side?"* — and the answer determines the table structure mechanically. No creativity required.

---

## Where this fits in the big picture

```
   PHASE 1 (01–09) — data is bytes in pages
   PHASE 2 (10–19) — how to find it fast, and what a join costs
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ PHASE 3 — DATABASE DESIGN                │
        │ 20 ER MODELLING          ← YOU ARE HERE  │
        │    requirements → boxes and lines        │
        └────────────────────┬─────────────────────┘
                             ▼
                    21 ER → relational schema
                    22–25 keys, FKs, constraints, types
                    26–28 patterns and migrations
                             ▼
                    PHASE 4 — normalisation
                    (proves your design is correct)
```

**Why after Phase 2 and not before:** you now know that a join is ~400 ms over 8M rows, that an index costs a write on every insert, and that a wide row destroys scan performance. Those numbers are what let you make design decisions honestly rather than by folklore.

---

## What is this?

**Entity-relationship modelling** is the process of turning a description of a problem domain into a diagram of:

- **Entities** — the things the business cares about (`users`, `orders`, `products`)
- **Attributes** — the facts about each thing (`email`, `total_paise`)
- **Relationships** — how things connect (`users` *place* `orders`)
- **Cardinality** — how many of each side participate (one-to-many, many-to-many)
- **Participation** — whether it's optional or mandatory

The ER model is **logical**, not physical. It says nothing about indexes, data types, or tables. Topic 21 converts it to a schema; this topic gets the model right first.

---

## Why does it matter for a backend developer?

Because the cost of fixing a design error rises by roughly an order of magnitude at each stage:

```
 caught in the ER diagram   →  rub out a line.               5 minutes
 caught in the schema       →  rewrite a migration.          1 hour
 caught after code is built →  refactor the data layer.      1 week
 caught in production       →  migrate live data, dual-write,
                               backfill 400M rows.           1 quarter
```

And the errors that hurt most are **cardinality errors**, which are invisible in code until they aren't:

- You modelled "a user has one address." Two years later, users have a billing address and a shipping address. Every query, every form, every API response changes.
- You modelled "an order has a product." Then someone orders two different products. You now have `product_id_1`, `product_id_2`, `product_id_3` — and a bug when someone orders four.
- You put `quantity` on `products` instead of on the order line, so you cannot answer "how many did *this customer* buy."

Each of these is a five-second fix in a diagram and a quarter-long project in production. **This topic is entirely about front-loading that thinking.**

---

## The physical reality

The ER model is deliberately *not* physical — but it is not floating in the air either. Each element has a direct physical consequence you now understand from Phases 1 and 2:

```
 ER ELEMENT              PHYSICAL CONSEQUENCE (what it costs)
 ──────────────────────────────────────────────────────────────────────
 An ENTITY               → a heap file, a primary key index,
                           ~28 bytes of per-row overhead (Topic 05)

 An ATTRIBUTE            → bytes in every row, forever, plus alignment
                           padding. A wide entity = fewer rows per page
                           = slower scans (Topic 04)

 A 1:N RELATIONSHIP      → a foreign key column (8 bytes) + an index on
                           it (~16 bytes/row) + an FK check on every
                           write (Topic 23)

 An M:N RELATIONSHIP     → a whole extra table + 2 indexes + a join
                           at read time (~400 ms per 8M rows, Topic 19)

 An OPTIONAL attribute   → a NULL bit (~free) — cheaper than a sentinel
                           value (Topic 05)

 A WEAK ENTITY           → a composite primary key, which makes every
                           index on it wider and shallower-fanout (T11)

 ⇒ EVERY LINE YOU DRAW HAS A PRICE. Drawing a line you don't need is
   how schemas become slow. Not drawing one you do need is how data
   becomes wrong.
```

**The one physical rule that should influence your ER model:** an entity with 40 attributes where 3 are hot and 37 are cold should probably be *two* entities in a 1:1 relationship (Topic 04's vertical split). That's a physical concern leaking into the logical model — and it's legitimate, because you know the cost.

---

## How it works — step by step

### The interrogation method — five questions, in order

This is the repeatable procedure. It replaces intuition.

```
 QUESTION 1 — WHAT ARE THE NOUNS?
   Underline every noun in the requirements. Then filter:
     ✓ KEEP if it has its own identity and its own facts
       ("a customer has a name, an email, a signup date")
     ✗ DROP if it's just a property of another noun
       ("a customer's email" — email is an attribute, not an entity)
     ✗ DROP if it's a synonym ("client" = "customer")
     ? MAYBE if it's a value that repeats across rows
       ("Mumbai" appears in 400,000 addresses → is `city` an entity?
        Decide with Question 5.)

 QUESTION 2 — FOR EACH NOUN, WHAT UNIQUELY IDENTIFIES IT?
   If you can't answer, it's not an entity yet — it's an attribute
   or you haven't understood it. (Full treatment in Topic 22.)

 QUESTION 3 — WHAT ARE THE VERBS BETWEEN THE NOUNS?
   "customers PLACE orders" · "orders CONTAIN products"
   Each verb is a candidate relationship. Name it — an unnamed line
   is a line you haven't thought about.

 QUESTION 4 — FOR EACH RELATIONSHIP, ASK BOTH DIRECTIONS.
   ★ THIS IS THE MOST IMPORTANT STEP IN THE WHOLE TOPIC. ★
     "Can one CUSTOMER have many ORDERS?"     → yes
     "Can one ORDER have many CUSTOMERS?"     → no
     ⇒ 1:N
   ⚠ ALWAYS ASK BOTH. Asking one direction is how you get 1:N wrong
     when it's really M:N.
   Then ask the follow-up: "…and how will that change in three years?"
     "Can an order have many customers?" — no, today.
     But will you add gift orders? Split payments? Corporate accounts
     with multiple approvers? Ask it explicitly.

 QUESTION 5 — DOES THE RELATIONSHIP ITSELF HAVE FACTS?
   "An order contains a product" — how many? at what price?
   `quantity` and `unit_price` belong to NEITHER the order NOR the
   product. They belong to the RELATIONSHIP.
   ⇒ If a relationship has attributes, it is ALWAYS a table.
     This is the single most common modelling mistake.

 QUESTION 6 — IS PARTICIPATION MANDATORY OR OPTIONAL?
   "Must every order have a customer?"  → mandatory → NOT NULL FK
   "Must every customer have an order?" → optional  → nothing to enforce
   ⇒ This directly determines NOT NULL, and it is a business rule
     you must ASK about rather than assume.
```

### Cardinality — the four cases, and what each becomes

```
 1:1  ── one A relates to at most one B, and vice versa
        "a user has one profile"
        ⚠ ALWAYS ASK: why is this not one table?
          Legitimate reasons: (a) different access frequency — the hot/
          cold split from Topic 04; (b) different security (PII isolation);
          (c) genuinely optional with many attributes.
          Illegitimate reason: "it felt tidier."
        → FK with a UNIQUE constraint on the dependent side (Topic 21)

 1:N  ── one A relates to many B; each B relates to one A
        "a customer places many orders"
        → the MOST COMMON case. FK column on the MANY side.
        ⚠ The FK goes on the side that says "one" in "one A" —
          i.e. on `orders`, pointing at `customers`. People reverse this.

 M:N  ── many A relate to many B
        "orders contain many products; products appear in many orders"
        → REQUIRES a junction table. There is no other option — you
          cannot put a list in a column and keep it relational.
        ★ And a junction table almost ALWAYS has its own attributes
          (quantity, price, added_at), which means it was really an
          entity all along.

 N-ary ── three or more entities in ONE relationship
        "a SUPPLIER supplies a PRODUCT to a WAREHOUSE at a PRICE"
        ⚠ Rare and usually decomposable. If it genuinely isn't, it
          becomes a table with three FKs. Test it: can you split it
          into three binary relationships without losing information?
          If yes, split it. If no (Topic 36, 5NF), keep it ternary.
```

### Chen vs Crow's Foot notation

```
 CHEN (academic; entities in boxes, relationships in diamonds)

   ┌──────────┐        ◇────────◇        ┌──────────┐
   │ CUSTOMER │───1────< places >────N───│  ORDER   │
   └──────────┘        ◇────────◇        └──────────┘

 CROW'S FOOT (what you'll actually see and draw)

   ┌──────────┐                           ┌──────────┐
   │ CUSTOMER │──────────||──────────<────│  ORDER   │
   └──────────┘          │        │       └──────────┘
                         │        └── "many" (the crow's foot)
                         └── "exactly one" (mandatory)

   THE FOUR END SYMBOLS — read the symbol NEAREST the box it describes:
     ──||──   exactly one   (mandatory, single)
     ──o|──   zero or one   (optional, single)
     ──|<──   one or more   (mandatory, many)
     ──o<──   zero or more  (optional, many)

   ⇒ The INNER symbol is the maximum, the OUTER symbol the minimum.
     ──o<── : minimum zero, maximum many.
```

### Worked example — the interrogation, live

```
 REQUIREMENTS:
 "We sell products. Customers place orders. An order can have several
  products in it, and each line says how many and at what price. Products
  belong to one category. We ship from warehouses; a product can be
  stocked in several warehouses with different quantities. Each order is
  shipped from exactly one warehouse. Customers can save multiple delivery
  addresses and pick one per order."

 Q1 NOUNS:  products · customers · orders · line · category · warehouses
            · quantity · price · addresses
   ✓ entities: product, customer, order, category, warehouse, address
   ✗ attributes: quantity, price  (they describe a relationship — flag
     them for Q5)
   ? "line" — it's the relationship between order and product. Q5 decides.

 Q2 IDENTITY:
   product → SKU · customer → email/id · order → order number
   category → code · warehouse → code · address → (no natural key)
   ⇒ address has no natural identity of its own. It's owned by a
     customer. FLAG: possible weak entity.

 Q3 VERBS:
   customers PLACE orders
   orders CONTAIN products
   products BELONG TO category
   products ARE STOCKED IN warehouses
   orders SHIP FROM warehouse
   customers SAVE addresses
   orders DELIVER TO address

 Q4 CARDINALITY — both directions, every time:
   customer→order:   one customer, many orders? YES
                     one order, many customers? NO      ⇒ 1:N
   order→product:    one order, many products?  YES
                     one product, many orders?  YES     ⇒ M:N  ★
   product→category: one product, many categories? NO
                     one category, many products?  YES  ⇒ 1:N
   product→warehouse:one product, many warehouses? YES
                     one warehouse, many products?  YES ⇒ M:N  ★
   order→warehouse:  one order, many warehouses?  NO
                     one warehouse, many orders?  YES   ⇒ 1:N
   customer→address: one customer, many addresses? YES
                     one address, many customers?  NO   ⇒ 1:N (weak)
   order→address:    one order, many addresses?   NO
                     one address, many orders?    YES   ⇒ 1:N

 Q5 RELATIONSHIP ATTRIBUTES:
   order CONTAINS product  →  quantity, unit_price_at_time_of_order  ★
   product STOCKED IN warehouse → quantity_on_hand, reorder_level    ★
   ⇒ BOTH M:N relationships have attributes. Both become full entities:
       ORDER_LINE  and  STOCK_LEVEL

 Q6 PARTICIPATION:
   Every order MUST have a customer            → mandatory
   Every order MUST have ≥1 line               → mandatory (app-enforced)
   Every product MUST have a category          → ask! (probably yes)
   Every order MUST have a delivery address    → mandatory
   A customer NEED NOT have any orders         → optional
   ⇒ mandatory ⇒ NOT NULL on the FK
```

**Note what Q5 did:** it turned two "relationships" into two entities. `order_line` isn't a join table you tolerate — it's a first-class business object with a price, a quantity, and eventually a fulfilment status. **Most M:N relationships are entities in disguise.**

---

## Concept breakdown

```
ENTITY
│  └── A thing with independent identity and its own facts.
│      TEST: can you point at one and say "that one, specifically"?
│      TEST: does it have attributes that aren't just a foreign key?
│
├── STRONG ENTITY   exists independently        (customer, product)
└── WEAK ENTITY     exists only within a parent (order_line, address)
     └── identified by the parent's key + a local discriminator
         ⇒ becomes a COMPOSITE PRIMARY KEY (order_id, line_no)

ATTRIBUTE
│
├── SIMPLE          indivisible                     (email)
├── COMPOSITE       decomposable                    (address → line1, city, pin)
│                   ⚠ 1NF says decompose it IF you query the parts (Topic 31)
├── MULTI-VALUED    many per entity                 (phone numbers)
│                   ⚠ ALWAYS becomes its own entity. Never phone1/phone2/phone3.
├── DERIVED         computable from others          (order_total = Σ lines)
│                   ⚠ store it ONLY as a deliberate denormalisation (Topic 55)
└── KEY             uniquely identifies             (Topic 22)

RELATIONSHIP
│
├── DEGREE          binary (2 entities) · ternary (3) · recursive (1)
├── CARDINALITY     1:1 · 1:N · M:N
├── PARTICIPATION   mandatory (total) vs optional (partial)
└── ATTRIBUTES      ★ if it has any, it's an entity

RECURSIVE (SELF-REFERENCING) RELATIONSHIP
│   "an employee MANAGES employees" · "a category has SUBcategories"
│   "a post REPLIES TO a post"
├── 1:N recursive → a nullable FK to the same table (manager_id)
├── M:N recursive → a junction table with two FKs to the same table
│                   (users FOLLOWS users → follows(follower_id, followee_id))
└── ⚠ These are where hierarchies come from, and where recursive CTEs
     and depth limits become a design concern.

PARTICIPATION vs CARDINALITY — people conflate these
│   CARDINALITY:   HOW MANY on each side (1 or N)
└── PARTICIPATION: whether ZERO is allowed (optional vs mandatory)
    ⇒ "1:N mandatory on the N side" = every order MUST have a customer
      = NOT NULL. Cardinality alone doesn't tell you that.

THE FOUR TESTS FOR "IS THIS AN ENTITY?"
  1. Does it have identity? (can you point at one?)
  2. Does it have attributes beyond a foreign key?
  3. Do you need to talk about it independently?
  4. Does its lifecycle differ from its parent's?
  ⇒ 2+ yeses → entity. 0–1 → attribute.
```

---

## Diagrams

**Diagram 1 — big picture: the finished model**

```
 ┌─────────────┐                            ┌──────────────┐
 │  CATEGORY   │                            │   WAREHOUSE  │
 │ ─────────── │                            │ ──────────── │
 │ code        │                            │ code         │
 │ name        │                            │ city         │
 └──────┬──────┘                            └───┬──────┬───┘
        │ 1                                   1 │      │ 1
        │                                       │      │
        │ N                                     │      │ N
 ┌──────┴──────┐        ┌──────────────┐        │  ┌───┴──────────┐
 │   PRODUCT   │        │ ORDER_LINE   │        │  │ STOCK_LEVEL  │
 │ ─────────── │  1   N │ ──────────── │        │  │ ──────────── │
 │ sku         ├────────┤ quantity ★   │        │  │ qty_on_hand ★│
 │ name        │        │ unit_price ★ │        │  │ reorder_lvl ★│
 │ price       │        └──────┬───────┘        │  └───┬──────────┘
 └──────┬──────┘               │ N              │      │ N
        │ 1                    │                │      │
        └──────────────────────┼────────────────┼──────┘
                               │ 1              │
                        ┌──────┴───────┐        │
                        │    ORDER     ├────────┘
                        │ ──────────── │  N          1
                        │ order_number │
                        │ placed_at    │
                        │ status       │
                        └──┬────────┬──┘
                         N │        │ N
                           │        │
                         1 │        │ 1
                  ┌────────┴──┐  ┌──┴──────────┐
                  │ CUSTOMER  │  │   ADDRESS   │
                  │ ───────── │1 │ ─────────── │
                  │ email     ├──┤ line1, city │
                  │ name      │ N│ pin         │
                  └───────────┘  └─────────────┘

  ★ = attributes that live on the RELATIONSHIP, not on either entity.
      These are what turned two M:N relationships into entities.
```

**Diagram 2 — data flow: the interrogation, as a decision tree**

```
                        A NOUN FROM THE REQUIREMENTS
                                    │
                     Does it have its own identity?
                          ┌────NO───┴───YES────┐
                          ▼                    ▼
                   ATTRIBUTE of         Does it have attributes
                   something else       beyond a foreign key?
                                              │
                                   ┌────NO────┴────YES────┐
                                   ▼                      ▼
                          Does its lifecycle          ★ ENTITY
                          differ from its parent?
                                   │
                        ┌────NO────┴────YES────┐
                        ▼                      ▼
                 pure JUNCTION            WEAK ENTITY
                 (rare — most have        (composite PK
                  attributes)              from the parent)
```

**Diagram 3 — before/after: the cardinality error that costs a quarter**

```
 WHAT WAS MODELLED (v1, shipped in month 2)
 ┌──────────┐  1        1  ┌───────────┐
 │   USER   ├──────────────┤  ADDRESS  │      "a user has one address"
 └──────────┘              └───────────┘
   users(id, name, email, address_line1, address_city, address_pin)
   ⇒ address flattened INTO users. Simple. Fast. One table.

 WHAT REALITY TURNED OUT TO BE (month 14)
 ┌──────────┐  1        N  ┌───────────┐
 │   USER   ├──────────────┤  ADDRESS  │      "billing, shipping, work,
 └──────────┘              └───────────┘       and 3 saved addresses"

 THE MIGRATION THAT WASN'T IN THE PLAN:
   • create `addresses` table
   • backfill 40M rows from the flattened columns
   • dual-write for 3 weeks while both paths run
   • rewrite every query, form, API response, and export
   • add `default_address_id` and backfill it
   • drop 3 columns from a 40M-row table (a full rewrite, Topic 28)
   • fix the 11 places that assumed one address

 ⇒ COST IN MONTH 2, HAD YOU ASKED Q4 BOTH DIRECTIONS: one extra table.
   COST IN MONTH 14: one quarter.

 ★ THE QUESTION THAT WOULD HAVE CAUGHT IT:
   "Can one user have many addresses?" — asked in BOTH directions,
   and followed by "…will that still be true in three years?"
```

---

## Example 1 — basic

**The requirements** (a library, deliberately small so you can see the whole method):

> "Members borrow books. A book can be borrowed by many members over time, but only by one at a time. We record when it was borrowed and when it's due. Books have an author; an author writes many books. Each book has multiple physical copies, and we track which copy was borrowed."

**Q1 — nouns.**

```
 member ✓ (identity: membership number; attributes: name, email, joined)
 book   ✓ (identity: ISBN; attributes: title, published_year)
 author ✓ (identity: id; attributes: name, born)
 copy   ✓ (identity: barcode; attributes: acquired_at, condition)
 borrow ? (a verb — but it has attributes. Flag for Q5.)
 due date, borrowed date → attributes of the borrowing, not of anything else
```

**Q2 — identity.** All four have natural keys. `copy` is interesting: a barcode is globally unique, so it's a **strong** entity, not weak. If copies were numbered 1..N *within* a book, it would be weak with PK `(isbn, copy_no)`.

**Q3 — verbs.** `member BORROWS copy` · `book HAS copy` · `author WRITES book`

**Q4 — cardinality, both directions.**

```
 member↔copy:  one member, many copies borrowed?   YES (over time, and
                                                    several at once)
               one copy, many members?             YES (over time)
               ⇒ M:N ★
               ⚠ BUT "only one at a time" is a TEMPORAL constraint,
                 not a cardinality one. Cardinality is M:N; the
                 "one at a time" rule is an exclusion constraint
                 (Topic 24 / case study 02). Do not let it fool you
                 into modelling 1:N.

 book↔copy:    one book, many copies?              YES
               one copy, many books?               NO
               ⇒ 1:N

 author↔book:  one author, many books?             YES
               one book, many authors?             ⚠ ASK. Today: no.
                 In three years, co-authored books? ALMOST CERTAINLY.
               ⇒ model as M:N now. The cost is one table; the cost
                 of changing it later is a migration.
```

**Q5 — relationship attributes.**

```
 member BORROWS copy → borrowed_at, due_at, returned_at   ★★★
 ⇒ This is not a junction table. It is a LOAN — a first-class business
   entity with a lifecycle, a status, and probably fines later.
   NAME IT PROPERLY: `loans`, not `member_copies`.
```

**Q6 — participation.**

```
 Every copy MUST belong to a book        → mandatory
 Every loan MUST have a member and copy  → mandatory
 A member NEED NOT have any loans        → optional
 A book NEED NOT have any copies yet     → optional (ordered, not arrived)
```

**The model:**

```
 ┌──────────┐  M      N ┌──────────┐  1      N ┌──────────┐
 │  AUTHOR  ├──────◇────┤   BOOK   ├───────────┤   COPY   │
 │ ──────── │ authorship│ ──────── │           │ ──────── │
 │ name     │           │ isbn     │           │ barcode  │
 │ born     │           │ title    │           │ acquired │
 └──────────┘           └──────────┘           └────┬─────┘
                                                    │ 1
                                                    │
                                                    │ N
 ┌──────────┐  1                          N   ┌─────┴──────┐
 │  MEMBER  ├───────────────────────────────── │    LOAN    │
 │ ──────── │                                  │ ────────── │
 │ number   │                                  │ borrowed_at│
 │ name     │                                  │ due_at     │
 └──────────┘                                  │ returned_at│
                                               └────────────┘

 ★ Note: the M:N between member and copy became LOAN, a strong entity
   sitting between them, with two 1:N relationships into it. This is
   what every M:N-with-attributes becomes.
```

**The mistake this method prevents:**

```
 ✗ NAIVE MODEL: copies(barcode, isbn, borrowed_by_member_id, due_at)
   "a copy is borrowed by a member" — 1:N, one column, simple!

   WHAT BREAKS:
     • no borrowing HISTORY — returning a book sets the column NULL
       and the past is erased
     • cannot answer "who borrowed this in March?"
     • cannot compute fines
     • cannot count "how many books has this member ever borrowed?"
     • the returned_at concept has nowhere to live

 ⇒ ALL of that follows from missing Q5: the relationship had attributes,
   so it was always an entity.
```

---

## Example 2 — production scenario

**The situation.** You're the backend engineer on a food-delivery platform. The product lead sends a paragraph:

> "Restaurants have menus. A menu has sections like 'Starters' and 'Mains'. Each section has dishes. A dish can have options — size, spice level, add-ons — and options change the price. Customers order dishes with their chosen options. Orders go to one restaurant and are delivered by one rider. Riders work shifts at one or more zones. We need to know, for any order, exactly what was ordered, at what price, even if the menu changes later."

**Step 1 — run the interrogation.**

```
 Q1 NOUNS: restaurant · menu · section · dish · option · customer
           · order · rider · shift · zone · price
   ✓ entities: restaurant, section, dish, option, customer, order,
               rider, shift, zone
   ? menu — does it have its own identity and facts? "Restaurants have
     menus" — is a menu just "the set of sections"? ASK.
     Answer: restaurants have a LUNCH menu and a DINNER menu with
     different hours. ⇒ MENU IS AN ENTITY (it has valid hours).
   ✗ price — an attribute. But of WHAT? Flag for Q5.

 Q4 CARDINALITY (both directions, always):
   restaurant↔menu    1:N   (lunch, dinner, weekend)
   menu↔section       1:N
   section↔dish       ⚠ ASK BOTH WAYS.
                      "can a dish appear in two sections?" — 'Paneer
                      Tikka' in both 'Starters' and 'Chef's Specials'?
                      Answer: YES. ⇒ M:N ★
   dish↔option        M:N   (a 'Large' size option applies to many
                             dishes; a dish has many options)
   customer↔order     1:N
   order↔restaurant   N:1
   order↔rider        N:1  (optional — unassigned at creation)
   rider↔zone         M:N  (via shifts)
   rider↔shift        1:N
   shift↔zone         N:1

 Q5 RELATIONSHIP ATTRIBUTES — the important ones:
   section CONTAINS dish   → display_order, is_featured        ★
   dish HAS option         → price_delta_paise, is_default     ★★★
   order CONTAINS dish     → quantity, chosen options, PRICE   ★★★
```

**Step 2 — the requirement that changes everything.**

> *"…even if the menu changes later."*

That single clause is a **temporal** requirement, and it is the hardest thing in the brief. Follow it through:

```
 If `order_lines` stores dish_id and we read the price from `dishes`
 at invoice time, then when the restaurant raises prices next week,
 EVERY HISTORICAL ORDER'S TOTAL CHANGES.

 This is exactly case study 04's reproducibility problem (I2), and the
 answer is the same: the order line must SNAPSHOT what was ordered.

 ⇒ order_lines carries: dish_id (a reference, for analytics)
                      + dish_name  (a snapshot, for the receipt)
                      + unit_price_paise (a snapshot, for the total)
                      + chosen options WITH their price deltas snapshotted

 ★ THIS IS DELIBERATE DENORMALISATION, AND IT IS CORRECT.
   The reference answers "how many Paneer Tikkas did we sell?"
   The snapshot answers "what does this receipt say?"
   Storing only one of them loses a question you will be asked.
   (Topic 03's lesson, and Topic 55's theory.)
```

**Step 3 — the model.**

```
 ┌────────────┐ 1   N ┌──────────┐ 1   N ┌──────────┐
 │ RESTAURANT ├───────┤   MENU   ├───────┤ SECTION  │
 │ ────────── │       │ ──────── │       │ ──────── │
 │ name       │       │ name     │       │ name     │
 │ address    │       │ hours    │       │ position │
 └─────┬──────┘       └──────────┘       └────┬─────┘
       │ 1                                     │ 1
       │                                       │ N
       │                              ┌────────┴─────────┐
       │                              │ SECTION_DISH     │  ← M:N with
       │                              │ ──────────────── │    attributes
       │                              │ display_order ★  │
       │                              │ is_featured   ★  │
       │                              └────────┬─────────┘
       │                                       │ N
       │                                       │ 1
       │ N                            ┌────────┴─────────┐  1      N
       │                              │      DISH        ├───────────┐
       │                              │ ──────────────── │           │
       │                              │ name             │    ┌──────┴──────┐
       │                              │ base_price       │    │ DISH_OPTION │
       │                              └────────┬─────────┘    │ ─────────── │
       │                                       │              │ price_delta★│
 ┌─────┴──────┐  N       1  ┌───────────┐      │              │ is_default ★│
 │   ORDER    ├─────────────┤   RIDER   │      │              └──────┬──────┘
 │ ────────── │             │ ───────── │      │                     │ N
 │ placed_at  │             │ name      │      │                     │ 1
 │ status     │             └─────┬─────┘      │              ┌──────┴──────┐
 │ total ★    │                 1 │            │              │   OPTION    │
 └─────┬──────┘                   │ N          │              │ ─────────── │
     1 │                    ┌─────┴─────┐      │              │ name        │
       │ N                  │   SHIFT   │      │              │ group       │
 ┌─────┴────────────┐       │ ───────── │      │              └─────────────┘
 │   ORDER_LINE     │       │ starts_at │      │
 │ ──────────────── │  N  1 │ ends_at   │      │
 │ quantity      ★  ├───────┴─────┬─────┘      │
 │ dish_id (ref)   ─┼─────────────────────────┘
 │ dish_name    ★★  │             │ N
 │ unit_price   ★★  │             │ 1
 │ (SNAPSHOTS)      │       ┌─────┴─────┐
 └────────┬─────────┘       │   ZONE    │
        1 │                 │ ───────── │
          │ N               │ name      │
 ┌────────┴───────────┐     │ polygon   │
 │ ORDER_LINE_OPTION  │     └───────────┘
 │ ────────────────── │
 │ option_name   ★★   │  ← also snapshotted
 │ price_delta   ★★   │
 └────────────────────┘

 ★  = attributes that belong to the relationship
 ★★ = SNAPSHOTS, deliberately denormalised for reproducibility
```

**Step 4 — what the interrogation caught that a first draft wouldn't.**

| Caught by | The issue | Cost if missed |
|---|---|---|
| Q4, both directions on section↔dish | a dish appears in two sections | `section_id` on `dishes` → a whole migration when 'Chef's Specials' launches |
| Q5 on dish↔option | `price_delta` has nowhere to live | options modelled as a JSON blob → cannot query "how many large sizes sold" |
| Q5 on order↔dish | quantity and price have nowhere to live | the classic `product_id_1, product_id_2` disaster |
| "even if the menu changes" | historical prices | every past order's total changes on a price update — the case study 04 bug |
| Q1 on "menu" | is it an entity? | lunch/dinner hours have nowhere to live |
| Q4 on order↔rider, participation | rider is optional at creation | `NOT NULL` on `rider_id` → cannot create an order before assignment |

**Step 5 — one thing the ER model deliberately does *not* say.**

Nothing above mentions indexes, data types, partitioning, or whether `order_line` should be partitioned by month. **That's correct.** The ER model is the *what*. Topics 21–28 are the *how*. Mixing them is how you end up with a schema optimised for a query nobody runs and unable to express a fact the business needs.

---

## Common mistakes

**1. Asking cardinality in one direction only.**
- *Symptom:* a 1:N that's really M:N; discovered in production.
- *Why it happens:* "an order has many products" sounds complete. But "a product appears in many orders" is the other half, and together they make M:N.
- *Fix:* write both questions down, literally, for every line. Then ask "will this still be true in three years?"

**2. Missing relationship attributes (Q5).**
- *Symptom:* `quantity` ends up on `products` (wrong for every order) or in a JSON blob (unqueryable).
- *Engine-level why:* the fact "3 of these, in this order, at this price" is functionally dependent on the *pair*, not on either member (Topic 30).
- *Fix:* for every relationship, ask "does anything describe this *pairing*?" If yes, it's an entity — and name it as one (`loan`, `order_line`, `enrolment`), not `member_copies`.

**3. Modelling multi-valued attributes as columns.**
- *Symptom:* `phone1`, `phone2`, `phone3`; `tag1`…`tag5`.
- *Why it breaks:* a fourth phone number needs a migration; "find everyone with this phone" needs 3 OR clauses; NULL columns proliferate.
- *Fix:* a multi-valued attribute is always its own entity. (Topic 31 formalises this as 1NF.)

**4. Confusing cardinality with a temporal constraint.**
- *Symptom:* "a copy is borrowed by one member" → modelled 1:N → borrowing history destroyed.
- *Why:* "one at a time" is an exclusion constraint over a time range, not a cardinality of 1. The relationship is M:N over time.
- *Fix:* ask "one at a time, or one ever?" The former is M:N + a constraint (Topics 16, 24).

**5. Creating entities for things that are just values.**
- *Symptom:* a `colours` table with `(1,'red')`, joined everywhere, that nobody ever adds to.
- *Fix:* apply the four tests. If it has no attributes beyond a name and no independent lifecycle, it's an enum or a `text` column with a `CHECK`. (Topic 27 covers when a lookup table *is* right — mainly when it carries extra attributes or is edited by non-engineers.)

**6. Designing for the query instead of the domain.**
- *Symptom:* an ER model that mirrors one screen's API response, with duplicated facts.
- *Why it breaks:* screens change quarterly; the domain doesn't. And a model shaped for one query can't answer the second.
- *Fix:* model the domain first (this topic), normalise it (Phase 4), *then* denormalise deliberately for measured needs (Phase 6). In that order.

**7. Skipping participation (Q6).**
- *Symptom:* nullable foreign keys everywhere, and no one knows which are legitimately optional.
- *Fix:* ask "must every X have a Y?" for every relationship, and record the answer on the diagram. It becomes `NOT NULL` directly.

---

## Hands-on proof

You can't `EXPLAIN` an ER diagram — but you can **prove a model is wrong** by trying to ask it a question it can't answer.

**PROVE IT #1 — the missing-history test.**
```sql
-- The naive model
CREATE TABLE copies (barcode text PRIMARY KEY, isbn text,
                     borrowed_by bigint, due_at timestamptz);
INSERT INTO copies VALUES ('C001','978-1', 7, now()+interval '14 days');
UPDATE copies SET borrowed_by=NULL, due_at=NULL WHERE barcode='C001'; -- returned
INSERT INTO copies VALUES ('C001','978-1', 9, now()+interval '14 days')
  ON CONFLICT (barcode) DO UPDATE SET borrowed_by=9;

-- Now answer: "who borrowed C001 in March?"
SELECT ... ;   -- ★ IMPOSSIBLE. The information was overwritten.
```
**A model that cannot answer a question the business will ask is wrong, regardless of how clean it looks.**

**PROVE IT #2 — the cardinality test, in SQL.**
```sql
-- Claim: "an order has one product" (1:N)
CREATE TABLE orders_v1 (id bigserial PRIMARY KEY, product_id bigint, qty int);
INSERT INTO orders_v1 (product_id, qty) VALUES (88, 2);
-- Now: the customer also wants product 91 on the SAME order.
-- ★ You cannot express it. The model is wrong.
```

**PROVE IT #3 — relationship attributes have nowhere to live.**
```sql
CREATE TABLE order_products (order_id bigint, product_id bigint,
                             PRIMARY KEY (order_id, product_id));
-- Now record: 3 units, at ₹499 each, at the time of order.
-- ★ Nowhere to put them without altering the table — which proves
--   the relationship had attributes all along.
```

**PROVE IT #4 — count the questions your model can answer.**
```sql
-- For any candidate model, write the 10 questions the business will
-- ask. Then write the SQL for each. Any question you cannot express
-- is a modelling defect, not a query problem.
--
-- Library example:
--   1. Which copies are out right now?
--   2. Who has had this copy, ever?
--   3. How many books has member 7 borrowed this year?
--   4. What is the average loan duration?
--   5. Which books are never borrowed?
--   6. Who has overdue items?
--   7. Which author is most borrowed?
--   ⇒ The naive model answers 1 and 6. The correct model answers all 7.
```

**PROVE IT #5 — draw it, then explain it back.**
Show the diagram to someone who knows the domain but not databases, and read each line aloud as a sentence: *"Every order line belongs to exactly one order, and refers to exactly one dish."* If a sentence sounds wrong to them, the model is wrong. **This catches more errors than any technical review.**

---

## The design decision framework

```
IS IT AN ENTITY OR AN ATTRIBUTE?
  ENTITY when:
    ✓ it has independent identity (you can point at one)
    ✓ it has attributes beyond a foreign key
    ✓ you need to talk about it on its own
    ✓ its lifecycle differs from its would-be parent's
  ATTRIBUTE when:
    ✓ it's a single value describing exactly one entity
    ✓ it has no facts of its own
    ✓ it never needs independent existence

IS IT 1:N OR M:N?
  ★ ASK BOTH DIRECTIONS. Always. Write both questions down.
  ★ THEN ASK: "will this still be true in three years?"
  When in doubt, MODEL AS M:N.
    cost of M:N-when-you-needed-1:N  = one extra table, one extra join
    cost of 1:N-when-you-needed-M:N  = a production migration
    ⇒ the asymmetry is enormous. Default to the flexible option.

DOES THE RELATIONSHIP BECOME A TABLE?
  ✓ ALWAYS for M:N (there is no alternative)
  ✓ ALWAYS if it has attributes
  ✓ And if it has attributes, NAME IT AS AN ENTITY:
      loan · order_line · enrolment · assignment · membership
      NOT member_copies · order_products · student_courses

SHOULD 1:1 BE ONE TABLE OR TWO?
  ONE TABLE by default.
  TWO TABLES when:
    ✓ hot/cold column split — different access frequency (Topic 04)
    ✓ different security/PII isolation requirements (Topic 69)
    ✓ genuinely optional with many attributes (avoids NULL sprawl)
    ✓ different retention or partitioning needs
  ✗ NEVER just because "it feels tidier"

MANDATORY OR OPTIONAL?
  Ask the business, don't assume. The answer becomes NOT NULL, and
  getting it wrong in either direction is expensive:
    too strict → you can't create an order before assigning a rider
    too loose  → orphan rows and "why is customer_id null?" forever

THE SIGNAL TO LOOK FOR:
  Before writing any DDL, write down the 10 questions the business
  will ask this data, and check your model can express each one.

  • A question you cannot express     → a missing entity or relationship
  • A question needing 5+ joins       → possibly over-modelled; check
                                        whether an entity is really an
                                        attribute
  • A fact stored in two places       → a modelling error, UNLESS it is
                                        a deliberate historical snapshot
  • An attribute you'd have to update
    in many rows to change one fact   → it belongs on a different entity
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
For each pair, state the cardinality **in both directions** and say whether the relationship needs its own table. Justify each with Q5.
(a) `country` ↔ `city` · (b) `student` ↔ `course` · (c) `user` ↔ `profile_photo` · (d) `employee` ↔ `manager` · (e) `invoice` ↔ `payment` · (f) `article` ↔ `tag`

### Exercise 2 — medium (apply it)
Take this brief and produce a full ER diagram using the six-question method. Show your working for **every** question — especially Q4 in both directions and Q5.

> "A hospital has departments. Doctors work in one department but can consult in others. Patients have appointments with doctors. An appointment produces zero or more prescriptions; each prescription lists medicines with a dosage and duration. Medicines have a manufacturer. We must be able to reproduce exactly what was prescribed, at what dosage, even after the medicine's standard dosage guidance is updated."

Then list the five questions the business will ask, and write the join path for each against your model.

### Exercise 3 — hard (production simulation)
You inherit this schema from a shipped v1:

```sql
CREATE TABLE users (
  id bigserial PRIMARY KEY, email text, name text,
  address_line1 text, address_city text, address_pin text,
  phone1 text, phone2 text,
  favourite_category text,
  last_order_id bigint, last_order_total bigint, last_order_at timestamptz
);
CREATE TABLE orders (
  id bigserial PRIMARY KEY, user_id bigint,
  product_id_1 bigint, qty_1 int,
  product_id_2 bigint, qty_2 int,
  product_id_3 bigint, qty_3 int,
  total_paise bigint, shipped_from text, status text
);
CREATE TABLE products (
  id bigserial PRIMARY KEY, sku text, name text, price_paise bigint,
  category text, warehouse_stock text   -- '{"BLR":40,"BOM":12}'
);
```

(a) List **every** ER modelling error, and for each: the question that would have caught it, and the specific business question the current model cannot answer.
(b) Draw the corrected ER diagram, applying all six questions.
(c) Three of the errors are *deliberate denormalisations* that might be defensible. Identify them, and state the exact condition under which each would be correct.
(d) `orders.total_paise` is a derived attribute. Give the two arguments for storing it and the two against, and state which wins here and why.
(e) `products.warehouse_stock` is a JSON blob. Name three specific business questions it makes impossible or unbearably slow, and what it should be instead.
(f) The business now says: "we must show what a customer paid, even after prices change." Which parts of your corrected model does that change, and why is it *not* a normalisation failure?
(g) Rank the fixes by (business risk × migration cost) and give the order you'd actually ship them in.

---

## Mental model checkpoint

1. State the six interrogation questions in order. Which one is most often skipped, and what does skipping it cost?
2. Why must cardinality be asked in **both** directions? Give an example where one direction alone gives the wrong answer.
3. What does it mean when a relationship has attributes? What must you do?
4. Distinguish cardinality from participation. Which one becomes `NOT NULL`?
5. What are the four tests for "is this an entity?" Apply them to `city` in an addresses model.
6. Give three legitimate reasons to split a 1:1 relationship into two tables, and one illegitimate one.
7. Why is the asymmetry between "M:N when you needed 1:N" and "1:N when you needed M:N" so important to the default you choose?

---

## Quick reference card

**The six questions**

| # | Question | Produces |
|---|---|---|
| 1 | What are the nouns? | candidate entities |
| 2 | What identifies each? | primary keys (Topic 22) |
| 3 | What are the verbs? | relationships |
| 4 | **How many, in BOTH directions?** | cardinality |
| 5 | **Does the relationship have facts?** | junction entities |
| 6 | Mandatory or optional? | `NOT NULL` |

**Cardinality → structure**

| Cardinality | Becomes |
|---|---|
| 1:1 | one table, or FK + UNIQUE if split |
| 1:N | FK on the **many** side |
| M:N | a junction table — **always** |
| M:N with attributes | a named **entity** (`loan`, `order_line`) |
| recursive 1:N | nullable FK to the same table |
| recursive M:N | junction with two FKs to the same table |

**Crow's foot**

| Symbol | Means |
|---|---|
| `──||──` | exactly one |
| `──o|──` | zero or one |
| `──|<──` | one or more |
| `──o<──` | zero or more |

**Attribute types**

| Type | Handling |
|---|---|
| Simple | a column |
| Composite | decompose if you query the parts (Topic 31) |
| **Multi-valued** | **always its own entity** |
| Derived | compute it, unless deliberately denormalised (Topic 55) |

**The three rules**
1. Ask cardinality in **both** directions, then ask about three years from now.
2. If a relationship has attributes, it is an **entity** — and name it as one.
3. Write the 10 business questions **before** the DDL. A model that can't express one is wrong.

---

## When would I use this at work?

1. **A new feature kickoff.** Thirty minutes with the six questions and a whiteboard, before any code, catches the cardinality errors that would otherwise surface in month 14. The highest return on any half-hour you will spend on a project.

2. **Reviewing someone's migration.** `phone1, phone2` or `product_id_1, product_id_2` are instantly recognisable as multi-valued attributes flattened into columns, and Q5/Q4 give you the language to explain *why* it's wrong rather than just "that looks bad."

3. **Inheriting a legacy schema.** Reverse-engineering the ER model from existing tables tells you what the original authors believed about the domain — and comparing that to what the business now says is exactly where the bugs and the missing features live.

---

## Connected topics

**Understand before this:** 04–05 (what an entity costs physically), 19 (what a relationship costs at read time).

**This unlocks:**
- **21** — translating this diagram into actual tables, mechanically
- **22** — primary keys: the answer to Q2
- **23** — foreign keys: the physical form of every relationship line
- **24** — constraints: participation (Q6) and temporal rules
- **27** — schema patterns and antipatterns, several of which are ER errors
- **29–33** — normalisation, which *proves* whether your model is correct
- **Case studies 01–05** — every one begins with an implicit ER model
