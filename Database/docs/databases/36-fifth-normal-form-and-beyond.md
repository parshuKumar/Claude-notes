# 36 — Fifth Normal Form and Beyond
## Phase: Normalisation

---

## ELI5 — The Simple Analogy

A distributor keeps one list: **which agent sells which brand in which region.**

```
 Agent  | Brand   | Region
 ───────┼─────────┼────────
 Arjun  | Samsung | North
 Arjun  | Apple   | South
 Meera  | Samsung | South
```

Someone says: "this is just three separate facts — who represents which brand, who covers which region, and which brands sell in which region. Let's split it into three lists."

So they do. And when they join the three lists back together, a row appears that was never there: **Arjun | Samsung | South.**

Arjun *does* represent Samsung. Arjun *does* cover South. Samsung *is* sold in South. All three facts are true — and yet Arjun does not sell Samsung in the South. **The combination is a fact in its own right, and splitting destroyed it.**

Now flip it. If the business rule were *"an agent sells every brand they represent in every region they cover"*, then that row would be correct, and the three-way split would be a genuine improvement.

**Same table. Same data. The rule decides.** That is 5NF: the rare case where a table decomposes into *three or more* pieces without loss — and the far more common case where it doesn't, and splitting invents facts.

---

## Where this fits in the big picture

```
   31 1NF · 32 2NF · 33 3NF · 34 BCNF   (functional dependencies)
   35 4NF                                (multi-valued dependencies)
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 36 FIFTH NORMAL FORM     ← YOU ARE HERE  │
        │ JOIN dependencies —                      │
        │ the final generalisation                 │
        └────────────────────┬─────────────────────┘
                             ▼
                    37 the full worked example
                    38 ★ when to stop (where this
                       topic's practical answer lives)
```

The progression is one of increasing generality:

| Form | Dependency | Decomposes into |
|---|---|---|
| BCNF | functional — X determines one value | 2 pieces |
| 4NF | multi-valued — X determines a set | 2 pieces |
| **5NF** | **join — the relation is the join of N projections** | **3+ pieces** |

---

## What is this?

A **join dependency** `*(R₁, R₂, …, Rₙ)` holds on R when R equals the natural join of its projections onto R₁ … Rₙ — with no rows lost and, crucially, **no rows invented**.

A relation is in **5NF** (also called **project-join normal form**, PJNF) when every non-trivial join dependency is **implied by its candidate keys**.

The practical statement: **a relation is in 5NF if it cannot be losslessly decomposed into three or more smaller relations** — where "losslessly" means the join reconstructs exactly the original, no more and no less.

Every MVD is a join dependency (a binary one), so **5NF implies 4NF**. 5NF is the strongest normal form based on projection and join.

---

## Why does it matter for a backend developer?

Honestly? **Rarely — and knowing that is part of the lesson.**

```
 THE HONEST ASSESSMENT:

 ① GENUINE 5NF VIOLATIONS ARE RARE.
    They require a THREE-WAY (or higher) cyclic constraint with no
    two-way decomposition. Most tables in 4NF are already in 5NF.

 ② THE DANGEROUS MISTAKE IS THE OPPOSITE ONE.
    Far more damage is done by decomposing a table that is ALREADY in
    5NF — producing spurious rows on every join — than by leaving a
    genuine violation in place.

 ③ ★ THE REAL VALUE OF THIS TOPIC IS THE LOSSLESSNESS TEST.
    Every decomposition you make in Topics 32–35 must be lossless.
    5NF is where you learn to PROVE it, and to recognise the shape
    of a decomposition that invents data.
    ⇒ that skill applies constantly. The normal form itself rarely does.

 ④ AND IT NAMES A REAL BUSINESS SHAPE.
    "Agent sells brand in region", "supplier ships part to warehouse",
    "doctor treats condition at clinic" — three-way relationships where
    the combination is (or isn't) meaningful. Getting that judgement
    right matters even if you never say the words "fifth normal form".
```

So: learn to test losslessness, learn to recognise the three-way shape, and expect to *decide not to decompose* far more often than you decide to.

---

## The physical reality

### The spurious row, made concrete

```
 sales(agent, brand, region)
 ┌───────┬─────────┬────────┐
 │ Arjun │ Samsung │ North  │
 │ Arjun │ Apple   │ South  │
 │ Meera │ Samsung │ South  │
 └───────┴─────────┴────────┘   3 rows

 THE THREE PROJECTIONS:
   AB(agent,brand)      AR(agent,region)     BR(brand,region)
   ┌───────┬─────────┐  ┌───────┬────────┐   ┌─────────┬────────┐
   │ Arjun │ Samsung │  │ Arjun │ North  │   │ Samsung │ North  │
   │ Arjun │ Apple   │  │ Arjun │ South  │   │ Apple   │ South  │
   │ Meera │ Samsung │  │ Meera │ South  │   │ Samsung │ South  │
   └───────┴─────────┘  └───────┴────────┘   └─────────┴────────┘

 JOIN ALL THREE:
 ┌───────┬─────────┬────────┐
 │ Arjun │ Samsung │ North  │  ✓ original
 │ Arjun │ Apple   │ South  │  ✓ original
 │ Meera │ Samsung │ South  │  ✓ original
 │ Arjun │ Samsung │ South  │  ★ INVENTED. Not in the original.
 └───────┴─────────┴────────┘   4 rows

 ⇒ ★ THE DECOMPOSITION IS LOSSY. It produces a row asserting that
   Arjun sells Samsung in the South — which is FALSE.
 ⇒ THEREFORE the join dependency does NOT hold, and the table is
   ALREADY IN 5NF. Do not decompose it.
```

### When the three-way split *is* correct

```
 SAME TABLE, DIFFERENT BUSINESS RULE:

   "If an agent represents a brand, and that agent covers a region,
    and that brand is sold in that region — then the agent sells that
    brand in that region."

 ⇒ this CYCLIC rule is exactly the join dependency *(AB, AR, BR).
 ⇒ under it, the row (Arjun, Samsung, South) is not invented — it is
   REQUIRED. The original table was missing it.

 ⇒ NOW the three-way decomposition is lossless AND removes redundancy:
     3 agents × 4 brands × 5 regions could need up to 60 rows;
     the three projections need at most 3×4 + 3×5 + 4×5 = 47,
     and typically far fewer.

 ★★★ THE ENTIRE DECISION IS THIS ONE QUESTION:
   "Is the three-way combination implied by the three pairs,
    or is it an independent fact?"
     IMPLIED    → a join dependency holds → decompose (5NF)
     INDEPENDENT→ no join dependency → ★ already in 5NF → leave it
   ⇒ And it is a DOMAIN question. The data cannot answer it.
```

### Why "lossless" means two things

```
 A DECOMPOSITION IS LOSSLESS ONLY IF THE JOIN GIVES BACK **EXACTLY** R:

   ① NO ROWS LOST     — every original row reappears
   ② ★ NO ROWS GAINED — no row appears that was not there

 ⚠ ② IS THE ONE PEOPLE FORGET, and it is the dangerous one.
   A "lossy" decomposition almost never loses rows — projections keep
   everything. It INVENTS them. And an invented row is silently wrong
   data, which is worse than missing data because nothing signals it.

 THE TWO-WAY GUARANTEE (Topic 30):
   R → R1, R2 is lossless if (R1 ∩ R2) is a KEY of R1 or of R2.
   ⇒ a simple, checkable condition.

 ★ THERE IS NO EQUIVALENT SIMPLE CONDITION FOR THREE-WAY SPLITS.
   You must either know the join dependency from the domain, or run
   the CHASE algorithm (below).
```

---

## How it works — step by step

### Testing losslessness in SQL — the test you will actually use

```sql
-- Given R and a proposed decomposition into R1, R2, R3:
WITH rejoined AS (
  SELECT r1.a, r1.b, r2.c
  FROM r1 JOIN r2 USING (a) JOIN r3 USING (b, c)   -- the natural join
)
SELECT
  (SELECT count(*) FROM original) AS original_rows,
  (SELECT count(*) FROM rejoined) AS rejoined_rows,
  (SELECT count(*) FROM (SELECT * FROM rejoined EXCEPT SELECT * FROM original) x)
    AS spurious,     -- ★ MUST BE ZERO
  (SELECT count(*) FROM (SELECT * FROM original EXCEPT SELECT * FROM rejoined) x)
    AS lost;         -- ★ MUST BE ZERO
```
```
 ★ IF spurious > 0 → the decomposition is LOSSY. The join dependency
   does not hold. DO NOT DECOMPOSE.
 ⚠ AND: this tests your CURRENT data, which can DISPROVE a join
   dependency but never PROVE one (exactly like FDs, Topic 30).
   Zero spurious rows today might be an accident.
   ⇒ confirm with the domain before acting on it.
```

### The chase algorithm — the formal test

```
 To test whether a join dependency *(R1, R2, R3) holds given a set of
 constraints, build a tableau and "chase" it:

 ① Make one row per Rᵢ. In row i, put a DISTINGUISHED symbol (a, b, c…)
    for each attribute in Rᵢ, and a SUBSCRIPTED one elsewhere.

 EXAMPLE  R(A,B,C), decomposition (AB, BC, AC):
      A     B     C
   ┌─────┬─────┬─────┐
   │  a  │  b  │ c₁  │   row for AB
   │ a₂  │  b  │  c  │   row for BC
   │  a  │ b₃  │  c  │   row for AC
   └─────┴─────┴─────┘

 ② Apply the known dependencies repeatedly, equating symbols.
    For an FD X → Y: if two rows agree on X, make them agree on Y
    (prefer the distinguished symbol).

 ③ IF a row of all-distinguished symbols (a, b, c) appears
      → the join dependency HOLDS. The decomposition is lossless.
    IF the chase terminates without one
      → it does NOT hold. The decomposition is LOSSY.

 ⇒ ★ IN PRACTICE you will run the SQL test above, not the chase.
   But knowing the chase exists tells you the question is DECIDABLE,
   and that "it looks fine" is not a proof.
```

### The three-way shape, recognised

```
 A 5NF VIOLATION NEEDS A CYCLIC CONSTRAINT ACROSS THREE ENTITIES:

        AGENT ─────────── BRAND
           │  ╲         ╱  │
           │    ╲     ╱    │
           │      ╲ ╱      │
           │      ╱ ╲      │
           │    ╱     ╲    │
           │  ╱         ╲  │
        REGION ────────────┘

 ⇒ three pairwise relationships, and a rule saying the three-way
   combination is DETERMINED by them.

 ★ THE TELL: the business rule contains "and … and … then".
   "If an agent represents a brand AND covers a region AND the brand
    sells in that region, THEN the agent sells it there."
   ⇒ that "then" is the join dependency.

 ★ THE ANTI-TELL: the three-way combination has a fact of its own.
   a quota, a commission rate, a start date, an id
   ⇒ then it is an ENTITY, not a derived combination. Already 5NF.
   (The same test as Topic 35's false MVD.)
```

### DKNF — the theoretical endpoint

```
 DOMAIN-KEY NORMAL FORM (Fagin, 1981)

 A relation is in DKNF if EVERY constraint on it is a logical
 consequence of:
   ① DOMAIN constraints (what values an attribute may take)
   ② KEY constraints (uniqueness)

 ★ IF A RELATION IS IN DKNF, IT HAS NO ANOMALIES OF ANY KIND. Ever.
   It is the theoretical endpoint of normalisation.

 ⚠ THREE REASONS IT IS NOT A PRACTICAL TARGET:
   ① There is no algorithm to achieve it. Unlike 3NF (synthesis) or
      BCNF (decomposition), there is no procedure.
   ② Not every relation CAN be put in DKNF.
   ③ Many real constraints are neither domain nor key constraints —
      "an order's total equals the sum of its lines", "a booking must
      not overlap another" — and cannot be expressed as either.

 ⇒ ★ ITS VALUE IS AS A YARDSTICK, not a target:
   "which of my business rules are NOT expressible as a domain or key
    constraint?" Those are exactly the rules that need a CHECK, an
    EXCLUDE, a trigger, or application logic — and knowing which is
    which is genuinely useful (Topic 24).

 6NF: decomposes every relation to irreducible components (one
      attribute per table plus the key). Used in temporal databases and
      some data-warehouse styles (anchor modelling). Never in OLTP.
```

---

## Concept breakdown

```
JOIN DEPENDENCY  *(R₁, R₂, …, Rₙ)
│  └── R equals the natural join of its projections onto R₁ … Rₙ,
│      with NO rows lost and ★ NO rows invented.
│
├── every MVD is a BINARY join dependency ⇒ 5NF implies 4NF
├── 5NF = PJNF: every non-trivial JD is implied by the candidate keys
└── practical form: R cannot be losslessly split into 3+ relations

LOSSLESS means TWO things
├── no rows LOST      (projections rarely lose rows)
└── ★ no rows GAINED  (this is the dangerous one — invented data)

THE TWO-WAY TEST (simple)
└── R1 ∩ R2 is a key of R1 or R2  ⇒ lossless

THE THREE-WAY TEST (no simple condition)
├── the chase algorithm (formal, decidable)
└── ★ the SQL test: join the projections, EXCEPT against the original,
     and require ZERO spurious rows

★ THE DECIDING QUESTION
  "Is the three-way combination IMPLIED by the three pairs,
   or is it an INDEPENDENT fact?"
    implied     → a join dependency → decompose
    independent → ★ already 5NF → leave it alone
  ⇒ A DOMAIN question. Data can disprove a JD, never prove one.

THE ANTI-TELL
  Does the combination carry a fact of its own — a quota, a rate, an
  id, a date? ⇒ it is an ENTITY. Not a JD. Do not decompose.

DKNF
└── every constraint follows from domain + key constraints. No
    anomalies possible. No algorithm; not always achievable.
    ★ Use it as a yardstick: "which rules are NOT domain-or-key?"

★ THE PRACTICAL SUMMARY
  Genuine 5NF violations are rare. The valuable skill is the
  LOSSLESSNESS TEST, which you apply to every decomposition from
  Topics 32–35.
```

---

## Diagrams

**Diagram 1 — big picture: the spurious row**

```
 ORIGINAL (3 rows)                     PROJECTIONS
 ┌───────┬─────────┬────────┐          AB          AR          BR
 │ Arjun │ Samsung │ North  │      ┌────┬────┐ ┌────┬─────┐ ┌────┬─────┐
 │ Arjun │ Apple   │ South  │      │Arj │Sam │ │Arj │North│ │Sam │North│
 │ Meera │ Samsung │ South  │      │Arj │App │ │Arj │South│ │App │South│
 └───────┴─────────┴────────┘      │Mee │Sam │ │Mee │South│ │Sam │South│
                                   └────┴────┘ └────┴─────┘ └────┴─────┘
                                        │           │            │
                                        └───────────┴────────────┘
                                                    │ natural join
                                                    ▼
                                   ┌───────┬─────────┬────────┐
                                   │ Arjun │ Samsung │ North  │ ✓
                                   │ Arjun │ Apple   │ South  │ ✓
                                   │ Meera │ Samsung │ South  │ ✓
                                   │ Arjun │ Samsung │ South  │ ★ INVENTED
                                   └───────┴─────────┴────────┘
                                                    ↑
                              4 rows from 3. The decomposition is LOSSY.
                              ⇒ no join dependency ⇒ ALREADY IN 5NF.
```

**Diagram 2 — data flow: the decision**

```
              A THREE-COLUMN RELATIONSHIP TABLE
                              │
              Does the three-way combination carry
              a fact of its own? (quota, rate, id, date)
                              │
              ┌──────YES──────┴──────NO──────┐
              ▼                              ▼
      ★ IT IS AN ENTITY              Is the combination IMPLIED
        Already in 5NF.              by the three pairs?
        Give it a proper key.        (ask the DOMAIN)
        DO NOT DECOMPOSE.                    │
                              ┌──────NO──────┴──────YES──────┐
                              ▼                              ▼
                    ★ ALREADY IN 5NF                 A JOIN DEPENDENCY
                      the combination is             ⇒ decompose into
                      an independent fact.              3 projections
                      DO NOT DECOMPOSE.               ⇒ ★ THEN VERIFY
                                                        with the SQL test
```

**Diagram 3 — before/after: when the split IS right**

```
 RULE: "an agent sells every brand they represent in every region
        they cover, provided the brand is sold there"

 BEFORE — the full cross-section, materialised
 ┌──────────────────────────────────────────────────────────────┐
 │ sales(agent, brand, region)                                  │
 │ 20 agents × 8 brands × 6 regions, filtered by the rule       │
 │ ≈ 640 rows, and EVERY row is derivable from the three pairs  │
 │                                                              │
 │ "Arjun now covers East"                                      │
 │   ⇒ insert one row per (brand he represents × sold in East)  │
 │   ⇒ 5 rows. Miss one and the data asserts something false.   │
 └──────────────────────────────────────────────────────────────┘

 AFTER — three projections
 ┌──────────────────────────────────────────────────────────────┐
 │ agent_brands(agent, brand)       20×3 =  60 rows             │
 │ agent_regions(agent, region)     20×2 =  40 rows             │
 │ brand_regions(brand, region)      8×4 =  32 rows             │
 │                                        ───────                │
 │                                        132 rows              │
 │                                                              │
 │ "Arjun now covers East"  ⇒ ONE row in agent_regions.         │
 │ The 5 sales rows are DERIVED by the join. Always consistent. │
 └──────────────────────────────────────────────────────────────┘
        640 → 132 rows, and partial-update inconsistency is impossible

 ⚠ BUT ONLY IF THE RULE IS TRUE. If Arjun can cover East without
   selling Samsung there, this design cannot express it.
```

---

## Example 1 — basic

**Step 1 — build the table and test the three-way decomposition.**

```sql
CREATE TABLE sales (
  agent  text NOT NULL,
  brand  text NOT NULL,
  region text NOT NULL,
  PRIMARY KEY (agent, brand, region)
);
INSERT INTO sales VALUES
 ('Arjun','Samsung','North'),
 ('Arjun','Apple','South'),
 ('Meera','Samsung','South');

CREATE VIEW ab AS SELECT DISTINCT agent, brand  FROM sales;
CREATE VIEW ar AS SELECT DISTINCT agent, region FROM sales;
CREATE VIEW br AS SELECT DISTINCT brand, region FROM sales;
```

```sql
-- ★ THE LOSSLESSNESS TEST
WITH rejoined AS (
  SELECT ab.agent, ab.brand, ar.region
  FROM ab JOIN ar ON ar.agent = ab.agent
          JOIN br ON br.brand = ab.brand AND br.region = ar.region
)
SELECT (SELECT count(*) FROM sales)   AS original,
       (SELECT count(*) FROM rejoined) AS rejoined,
       (SELECT count(*) FROM (SELECT * FROM rejoined EXCEPT SELECT * FROM sales) x)
         AS spurious,
       (SELECT count(*) FROM (SELECT * FROM sales EXCEPT SELECT * FROM rejoined) x)
         AS lost;
```
```
 original | rejoined | spurious | lost
----------+----------+----------+------
        3 |        4 |        1 |    0
```

```sql
-- which row was invented?
WITH rejoined AS (
  SELECT ab.agent, ab.brand, ar.region FROM ab
  JOIN ar ON ar.agent=ab.agent JOIN br ON br.brand=ab.brand AND br.region=ar.region)
SELECT * FROM rejoined EXCEPT SELECT * FROM sales;
```
```
 agent |  brand  | region
-------+---------+--------
 Arjun | Samsung | South      ★ Arjun does NOT sell Samsung in the South
```

**★ One spurious row ⇒ the join dependency does not hold ⇒ the table is already in 5NF. Do not decompose.**

**Step 2 — now make the rule true, and re-test.**

```sql
-- add the row the cyclic rule requires
INSERT INTO sales VALUES ('Arjun','Samsung','South');

WITH rejoined AS (
  SELECT ab.agent, ab.brand, ar.region FROM ab
  JOIN ar ON ar.agent=ab.agent JOIN br ON br.brand=ab.brand AND br.region=ar.region)
SELECT (SELECT count(*) FROM sales) AS original,
       (SELECT count(*) FROM rejoined) AS rejoined,
       (SELECT count(*) FROM (SELECT * FROM rejoined EXCEPT SELECT * FROM sales) x)
         AS spurious;
```
```
 original | rejoined | spurious
----------+----------+----------
        4 |        4 |        0     ★ lossless — the JD holds NOW
```

⚠ **But this proves nothing about the rule.** The data is now *consistent with* a join dependency; it does not establish one. Only the business can say whether "Arjun sells Samsung in the South" was a fact you had to record, or one that follows automatically.

**Step 3 — the anti-tell: when the combination has its own fact.**

```sql
ALTER TABLE sales ADD COLUMN annual_quota_minor bigint;
UPDATE sales SET annual_quota_minor = 5000000 WHERE agent='Arjun' AND brand='Samsung';
```
```
 ★ NOW THE COMBINATION CARRIES A FACT OF ITS OWN.
   A quota is per (agent, brand, region). It cannot be derived from
   the three pairs.
 ⇒ NO join dependency can hold — decomposing would have nowhere to
   put the quota.
 ⇒ ★ THE TABLE IS AN ENTITY. Already in 5NF, permanently.
```

**Step 4 — the practical version: verify every decomposition you make.**

```sql
-- A REUSABLE LOSSLESSNESS CHECKER for a 2-way split
CREATE OR REPLACE FUNCTION check_lossless_2(
  orig text, r1 text, r2 text, join_cols text)
RETURNS TABLE(original bigint, rejoined bigint, spurious bigint, lost bigint) AS $$
BEGIN
  RETURN QUERY EXECUTE format($f$
    WITH rj AS (SELECT * FROM %I NATURAL JOIN %I)
    SELECT (SELECT count(*) FROM %I)::bigint,
           (SELECT count(*) FROM rj)::bigint,
           (SELECT count(*) FROM (SELECT * FROM rj EXCEPT SELECT * FROM %I) x)::bigint,
           (SELECT count(*) FROM (SELECT * FROM %I EXCEPT SELECT * FROM rj) x)::bigint
  $f$, r1, r2, orig, orig, orig);
END $$ LANGUAGE plpgsql;
```
**Run this after every decomposition in Topics 32–35.** `spurious = 0` and `lost = 0` is the definition of correct.

**Step 5 — a lossy two-way split, for contrast.**

```sql
CREATE TABLE r (a int, b int, c int);
INSERT INTO r VALUES (1,10,100),(2,10,200);

-- ✓ LOSSLESS: the shared attribute `a` is a key of both pieces
CREATE TABLE ok1 AS SELECT DISTINCT a, b FROM r;
CREATE TABLE ok2 AS SELECT DISTINCT a, c FROM r;
SELECT count(*) FROM (SELECT * FROM ok1 NATURAL JOIN ok2) x;      -- 2 ✓

-- ✗ LOSSY: the shared attribute `b` is a key of NEITHER piece
CREATE TABLE bad1 AS SELECT DISTINCT a, b FROM r;
CREATE TABLE bad2 AS SELECT DISTINCT b, c FROM r;
SELECT count(*) FROM (SELECT * FROM bad1 NATURAL JOIN bad2) x;
```
```
 count
-------
     4        ★ FOUR rows from TWO. Two invented.
```
```sql
SELECT * FROM (SELECT * FROM bad1 NATURAL JOIN bad2) x EXCEPT SELECT * FROM r;
```
```
 a | b  |  c
---+----+-----
 1 | 10 | 200      ★ invented
 2 | 10 | 100      ★ invented
```
**This is the two-way case of exactly the same failure**, and it is the one you will actually encounter.

---

## Example 2 — production scenario

**The situation.** A pharmaceutical distributor. The `distribution_rights` table has 41 million rows, and a proposal is on the table to "normalise it properly" into three tables.

```sql
CREATE TABLE distribution_rights (
  distributor_id bigint NOT NULL,
  product_id     bigint NOT NULL,
  territory_id   bigint NOT NULL,
  PRIMARY KEY (distributor_id, product_id, territory_id)
);
```

**The proposal:**
```sql
-- "these are just three independent facts"
CREATE TABLE distributor_products  (distributor_id, product_id);
CREATE TABLE distributor_territories(distributor_id, territory_id);
CREATE TABLE product_territories   (product_id, territory_id);
```

**Step 1 — run the test before agreeing.**

```sql
WITH dp AS (SELECT DISTINCT distributor_id, product_id FROM distribution_rights),
     dt AS (SELECT DISTINCT distributor_id, territory_id FROM distribution_rights),
     pt AS (SELECT DISTINCT product_id, territory_id FROM distribution_rights),
     rejoined AS (
       SELECT dp.distributor_id, dp.product_id, dt.territory_id
       FROM dp JOIN dt ON dt.distributor_id = dp.distributor_id
               JOIN pt ON pt.product_id = dp.product_id
                      AND pt.territory_id = dt.territory_id)
SELECT (SELECT count(*) FROM distribution_rights) AS original,
       (SELECT count(*) FROM rejoined) AS rejoined,
       (SELECT count(*) FROM (SELECT * FROM rejoined
                              EXCEPT SELECT * FROM distribution_rights) x) AS spurious;
```
```
 original  | rejoined  | spurious
-----------+-----------+-----------
  41204882 | 188402114 |  147197232
```

**★ 147 million invented rows.** The proposed decomposition would grant 147 million distribution rights that do not exist.

**Step 2 — inspect what was invented.**

```sql
WITH dp AS (…), dt AS (…), pt AS (…), rejoined AS (…)
SELECT r.distributor_id, r.product_id, r.territory_id
FROM (SELECT * FROM rejoined EXCEPT SELECT * FROM distribution_rights) r
LIMIT 5;
```
```
 distributor_id | product_id | territory_id
----------------+------------+--------------
           4471 |       8812 |          204
           4471 |       8812 |          207
 …
```
```sql
-- what IS product 8812?
SELECT p.name, p.schedule, t.name AS territory, t.country
FROM products p, territories t WHERE p.id=8812 AND t.id=204;
```
```
      name      | schedule |  territory  | country
----------------+----------+-------------+---------
 Alprazolam 0.5 | H1       | Sri Lanka   | LK
```

**★ A Schedule H1 controlled substance, in a territory where this distributor holds no licence for it.** The decomposition would have created a data structure asserting a right that is illegal to exercise.

**Step 3 — why the join dependency does not hold.**

```
 ASK THE DOMAIN:
   "If a distributor is licensed for product P, and operates in
    territory T, and product P is registered in territory T —
    does that distributor automatically have the right to
    distribute P in T?"

 ANSWER: ★ NO. Distribution rights are granted per (distributor,
   product, territory) by contract and by regulatory licence. All
   three pairwise facts can be true while the specific right does not
   exist — because the licence for that combination was never granted,
   or has lapsed, or is held exclusively by a competitor.

 ⇒ THE COMBINATION IS AN INDEPENDENT FACT.
 ⇒ NO join dependency. ★ THE TABLE IS ALREADY IN 5NF.
 ⇒ The proposal is not a normalisation; it is data corruption.
```

**Step 4 — the anti-tell confirms it.**

```sql
-- does the combination carry facts of its own?
ALTER TABLE distribution_rights
  ADD COLUMN granted_at date,
  ADD COLUMN expires_at date,
  ADD COLUMN licence_no text,
  ADD COLUMN exclusive boolean;
```
```
 ★ YES — every one of these is per-combination:
   a licence number is issued for (distributor, product, territory)
   an expiry date is per licence
   exclusivity is per combination
 ⇒ these facts have NOWHERE to live in the three-way split.
 ⇒ CONCLUSIVE: it is an ENTITY, not a derived combination.
```

**Step 5 — what the table actually needed.**

The table wasn't unnormalised — it was **under-modelled**. It was a bare three-column relationship where a first-class entity belonged:

```sql
CREATE TABLE distribution_rights (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  distributor_id bigint NOT NULL REFERENCES distributors(id) ON DELETE RESTRICT,
  product_id     bigint NOT NULL REFERENCES products(id)     ON DELETE RESTRICT,
  territory_id   bigint NOT NULL REFERENCES territories(id)  ON DELETE RESTRICT,
  licence_no  text NOT NULL,
  granted_at  date NOT NULL,
  expires_at  date NOT NULL,
  exclusive   boolean NOT NULL DEFAULT false,
  CONSTRAINT uq_right UNIQUE (distributor_id, product_id, territory_id),
  CONSTRAINT uq_licence UNIQUE (licence_no),
  CHECK (expires_at > granted_at)
);
CREATE INDEX ON distribution_rights (product_id, territory_id);
CREATE INDEX ON distribution_rights (expires_at) WHERE expires_at > current_date;

-- ★ and the exclusivity rule, which the three-way split could never
--   have expressed: only ONE distributor may hold an exclusive right
--   for a (product, territory) at a time
CREATE UNIQUE INDEX uq_exclusive_right
  ON distribution_rights (product_id, territory_id)
  WHERE exclusive AND expires_at > current_date;
```

**Step 6 — the outcome.**

| | Proposed 3-way split | What was shipped |
|---|---|---|
| Rows | 3 tables, ~2.1M total | 41M (unchanged — correct) |
| Spurious rights created | **147,197,232** | 0 |
| Per-licence facts (number, expiry, exclusivity) | **nowhere to store** | 4 columns |
| Exclusivity rule | inexpressible | one partial unique index |
| Regulatory correctness | **violated** | enforced |

**The "normalisation" would have been a compliance incident.** The lossless test caught it in one query, before any code was written.

---

## Common mistakes

**1. Decomposing a three-column table because "it's just three relationships."**
- *Symptom:* joins produce rows that were never inserted.
- *Engine-level why:* the projections lose the information about *which* combinations existed; the join reconstructs all of them.
- *Fix:* run the lossless test **before** decomposing. `spurious > 0` ends the discussion.

**2. Treating zero spurious rows as proof.**
- *Symptom:* decomposing based on today's data, then discovering the invented rows next quarter.
- *Engine-level why:* like FDs and MVDs, data can *disprove* a join dependency but never *prove* one.
- *Fix:* confirm with the domain — is the combination *implied*, or an independent fact?

**3. Missing the anti-tell.**
- *Symptom:* halfway through the migration, nobody can say where the licence number goes.
- *Fix:* ask first — "does the three-way combination carry any fact of its own?" If yes, it's an entity and you're done.

**4. Chasing 5NF as a goal.**
- *Symptom:* weeks spent on a form that almost never applies.
- *Fix:* most 4NF tables are already in 5NF. Spend the time on 3NF, where the violations actually are (Topic 38).

**5. Forgetting the two-way lossless condition.**
- *Symptom:* a routine 3NF decomposition that invents rows.
- *Engine-level why:* the shared attributes weren't a key of either piece.
- *Fix:* `R1 ∩ R2` must be a key of `R1` or `R2`. Check it every time.

**6. Treating DKNF as a target.**
- *Symptom:* a schema design that never finishes.
- *Fix:* there is no algorithm and it isn't always achievable. Use it as a yardstick for "which rules need a CHECK, an EXCLUDE, or a trigger?"

---

## Hands-on proof

**PROVE IT #1 — the spurious row.** (Example 1, step 1.)
**PROVE IT #2 — the JD holds after adding the required row.** (Example 1, step 2.)
**PROVE IT #3 — the anti-tell.** (Example 1, step 3.)
**PROVE IT #4 — a lossy two-way split.** (Example 1, step 5.)

**PROVE IT #5 — the reusable three-way checker.**
```sql
CREATE OR REPLACE FUNCTION check_lossless_3(
  orig text, c1 text, c2 text, c3 text)
RETURNS TABLE(original bigint, rejoined bigint, spurious bigint, lost bigint) AS $$
BEGIN
  RETURN QUERY EXECUTE format($f$
    WITH p12 AS (SELECT DISTINCT %I, %I FROM %I),
         p13 AS (SELECT DISTINCT %I, %I FROM %I),
         p23 AS (SELECT DISTINCT %I, %I FROM %I),
         rj  AS (SELECT p12.%I, p12.%I, p13.%I
                   FROM p12 JOIN p13 USING (%I) JOIN p23 USING (%I, %I))
    SELECT (SELECT count(*) FROM %I)::bigint,
           (SELECT count(*) FROM rj)::bigint,
           (SELECT count(*) FROM (SELECT * FROM rj EXCEPT
                                  SELECT %I,%I,%I FROM %I) x)::bigint,
           (SELECT count(*) FROM (SELECT %I,%I,%I FROM %I EXCEPT
                                  SELECT * FROM rj) x)::bigint
  $f$, c1,c2,orig, c1,c3,orig, c2,c3,orig,
       c1,c2,c3, c1, c2,c3,
       orig, c1,c2,c3,orig, c1,c2,c3,orig);
END $$ LANGUAGE plpgsql;

SELECT * FROM check_lossless_3('sales','agent','brand','region');
```

**PROVE IT #6 — find three-column relationship tables worth checking.**
```sql
SELECT c.relname,
       string_agg(a.attname, ', ' ORDER BY x.ord) AS pk_columns,
       (SELECT count(*) FROM pg_attribute a2 WHERE a2.attrelid=c.oid
          AND a2.attnum>0 AND NOT a2.attisdropped) AS total_columns
FROM pg_constraint con
JOIN pg_class c ON c.oid=con.conrelid
CROSS JOIN LATERAL unnest(con.conkey) WITH ORDINALITY x(attnum, ord)
JOIN pg_attribute a ON a.attrelid=c.oid AND a.attnum=x.attnum
WHERE con.contype='p' AND cardinality(con.conkey) = 3
GROUP BY c.oid, c.relname;
-- ★ a 3-column PK covering ALL columns = a candidate for the 5NF question
--   a 3-column PK with EXTRA columns = an entity. Already 5NF. Stop.
```

---

## The design decision framework

```
YOU ARE LOOKING AT A THREE-COLUMN RELATIONSHIP TABLE.

 ① DOES THE COMBINATION CARRY A FACT OF ITS OWN?
    (a quota, a rate, a licence number, a date, an id, a status)
    YES → ★ IT IS AN ENTITY. Already in 5NF. Give it a surrogate key,
          a UNIQUE on the triple, and the columns it needs. STOP.
    NO  → continue.

 ② RUN THE LOSSLESS TEST ON YOUR REAL DATA.
      join the three projections, EXCEPT against the original
      • spurious > 0 → ★ NO join dependency. Already in 5NF. STOP.
      • spurious = 0 → a candidate. Continue to ③.

 ③ ★ ASK THE DOMAIN — the data cannot answer this:
      "If A relates to B, and A relates to C, and B relates to C,
       does the three-way fact FOLLOW AUTOMATICALLY?"
      NO  → the combination is an independent fact. Already 5NF. STOP.
      YES → a genuine join dependency. Decompose.

 ④ AFTER DECOMPOSING, RE-RUN THE TEST. spurious must be 0.

★ EXPECT TO STOP AT ① OR ②. Genuine 5NF violations are rare, and the
  cost of getting it wrong is asymmetric:
    failing to decompose a true JD  → some redundancy. Recoverable.
    decomposing a false JD          → ★ INVENTED DATA. In Example 2,
                                       147 million false regulatory
                                       rights. Not recoverable, and
                                       possibly a compliance incident.
  ⇒ WHEN IN DOUBT, DO NOT DECOMPOSE.

THE REAL TAKEAWAY — USE THE LOSSLESS TEST EVERYWHERE:
  Every decomposition in Topics 32–35 must satisfy:
    two-way:   R1 ∩ R2 is a key of R1 or R2
    any split: join the pieces, EXCEPT both ways, require 0 and 0
  ⇒ ★ THIS is the skill this topic buys you. Run it on every
    decomposition you ship.

DKNF — as a yardstick, not a target:
  "Which of my business rules are NOT expressible as a domain
   constraint or a key constraint?"
  ⇒ those need a CHECK, an EXCLUDE, a trigger, or application logic
    plus reconciliation. Knowing which is which is the useful part
    (Topic 24).
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build the `sales` table from Example 1. Run the three-way lossless test and identify the spurious row. Then add the row the cyclic rule requires and re-run. Explain precisely why the second result does **not** prove that a join dependency holds.

### Exercise 2 — medium (apply it)
For each table, decide whether it is in 5NF. Justify using **both** the anti-tell and the domain question, and state what decomposing would cost if you were wrong:
(a) `teaching(lecturer, course, semester)` · (b) `prescriptions(doctor, patient, drug, dosage, prescribed_at)` · (c) `supply(supplier, part, warehouse)` where any supplier who stocks a part ships it to any warehouse that stocks it · (d) `assignments(employee, project, role)` · (e) `flight_routes(airline, origin, destination)`

### Exercise 3 — hard (production simulation)
A pharmaceutical distributor's `distribution_rights` table has 41 million rows with PK `(distributor_id, product_id, territory_id)`. A senior engineer proposes decomposing it into three pairwise tables, arguing it is a 5NF violation.

(a) Write the query that tests the proposal, and predict its result.
(b) The test returns 147 million spurious rows. Explain in domain terms what each spurious row asserts, and why some of them are regulatory violations.
(c) State the domain question that settles it, and the answer that makes the proposal wrong.
(d) Apply the anti-tell. Name four facts that would have nowhere to live after the split.
(e) The table wasn't over-normalised or under-normalised — it was under-*modelled*. Explain the distinction, and design the corrected schema.
(f) One business rule — exclusivity — is expressible only in the un-decomposed design. Write the constraint and explain why the three-way split could never enforce it.
(g) Write the general-purpose lossless checker you would add to your team's migration tooling, and say at which point in a migration it should run.

---

## Mental model checkpoint

1. Define a join dependency. Why does "lossless" mean two separate things, and which is the dangerous one?
2. Why does 5NF imply 4NF?
3. Give the simple two-way losslessness condition. Why is there no equivalent for three-way splits?
4. State the two questions that decide whether a three-column table is already in 5NF.
5. Why can data disprove a join dependency but never prove one?
6. What is the asymmetry between failing to decompose a true JD and decomposing a false one?
7. What is DKNF, and why is it a yardstick rather than a target?

---

## Quick reference card

**Join dependency `*(R₁,…,Rₙ)`:** R equals the natural join of its projections — no rows lost, **no rows invented**.

**5NF (PJNF):** every non-trivial join dependency is implied by the candidate keys. Practically: R cannot be losslessly split into 3+ relations.

| | |
|---|---|
| Every MVD is a binary JD | ⇒ 5NF implies 4NF |
| Two-way lossless test | `R1 ∩ R2` is a key of `R1` or `R2` |
| Three-way test | no simple condition — use the chase, or the SQL test |
| Frequency | **rare**; most 4NF tables are already in 5NF |

**★ The two questions**

1. *Does the combination carry a fact of its own?* — yes → an **entity**, already 5NF.
2. *Is the three-way fact implied by the three pairs?* — no → already 5NF.

**The SQL test — run it on every decomposition**
```sql
SELECT count(*) FROM (SELECT * FROM rejoined EXCEPT SELECT * FROM original) x;  -- must be 0
SELECT count(*) FROM (SELECT * FROM original EXCEPT SELECT * FROM rejoined) x;  -- must be 0
```

**⚠ Asymmetry:** failing to decompose costs redundancy (recoverable). Decomposing a false JD **invents data** (not recoverable). **When in doubt, don't.**

**DKNF:** every constraint follows from domain + key constraints ⇒ no anomalies possible. No algorithm, not always achievable. Use as a yardstick.

---

## When would I use this at work?

1. **Whenever someone proposes splitting a three-column relationship table.** One query settles it, and in the pharmaceutical example it prevented 147 million invented regulatory rights.

2. **After *every* decomposition, at any normal form.** The lossless test belongs in your migration tooling, run against production data before cutover. `spurious = 0` and `lost = 0` is the definition of a correct decomposition.

3. **When a table looks like a relationship but keeps growing columns.** A licence number, an expiry, an exclusivity flag — that's the anti-tell telling you it was always an entity, and the right fix is a surrogate key and a `UNIQUE`, not a split.

---

## Connected topics

**Understand before this:** 30 (lossless decomposition), 34 (BCNF), 35 (4NF and the false-MVD trap — the same asymmetry).

**This unlocks:**
- **37** — the full worked example, with losslessness verified at each step
- **38** — when to stop, where this topic's practical answer ("almost always before here") is made explicit
- **24** — the DKNF yardstick applied: which rules need `CHECK`, `EXCLUDE`, or a trigger
