# 30 — Functional Dependencies and Keys
## Phase: Normalisation

---

## ELI5 — The Simple Analogy

A school register.

If I tell you a **roll number**, you can tell me the student's name, their class, and their parent's phone number. One roll number, one answer to each. The roll number **determines** those things.

If I tell you a **class**, you can tell me who the class teacher is — one answer. But you *cannot* tell me a student's name, because a class has thirty of them. So class determines teacher, but not student.

And if I tell you a **student's name**, you cannot reliably tell me anything, because there are three Rahul Sharmas.

"X determines Y" is the whole idea. Written `X → Y`. It is a statement about the *rules of the domain*, not about the data currently in the table — if there happens to be only one Rahul Sharma today, name still doesn't determine anything, because tomorrow there will be two.

Every normal form in Topics 31–36 is a rule about which `X → Y` are allowed to live in the same table. This topic is the language.

---

## Where this fits in the big picture

```
   29 what normalisation is (the anomalies)
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 30 FUNCTIONAL DEPENDENCIES               │ ← YOU ARE HERE
        │ the language every normal form is        │
        │ defined in                               │
        └────────────────────┬─────────────────────┘
                             ▼
   31 1NF  ·  32 2NF  ·  33 3NF  ·  34 BCNF  ·  35 4NF  ·  36 5NF
        each is a rule stated in terms of FDs and keys
```

Topic 29 gave you the intuition — "a fact in the wrong place." **This topic makes "wrong place" precise**, so that 2NF, 3NF and BCNF become mechanical checks rather than judgement calls.

---

## What is this?

A **functional dependency** `X → Y` means: *for any two rows that agree on all the attributes in X, they must also agree on all the attributes in Y.*

From FDs you derive:

- **superkey** — a set of attributes that determines every attribute
- **candidate key** — a minimal superkey
- **primary key** — the candidate key you chose (Topic 22)
- **prime attribute** — one that appears in some candidate key
- **closure** `X⁺` — everything X determines, directly or indirectly

And the normal forms are then one-line statements about these.

---

## Why does it matter for a backend developer?

Because without FDs, "is this table normalised?" is a matter of taste, and with them it is a decidable question you can answer in five minutes.

```
 THE PRACTICAL PAYOFF:

 ① NORMALISATION BECOMES MECHANICAL
    2NF: "no non-prime attribute depends on part of a candidate key"
    3NF: "no non-prime attribute depends on a non-key"
    BCNF: "every determinant is a candidate key"
    ⇒ three checks. Not opinions.

 ② YOU CAN FIND THE REAL KEY
    Most tables have a surrogate `id`. That is NOT the candidate key of
    the DATA — it's an artefact you added. Finding the real candidate
    key(s) is how you discover the UNIQUE constraint you're missing.

 ③ YOU CAN PROVE A DECOMPOSITION IS SAFE
    Lossless join and dependency preservation are both stated in FDs.
    Without them, "split it into two tables" is a guess.

 ④ YOU CAN DETECT VIOLATIONS IN LIVE DATA
    Every claimed FD is a query:
      SELECT X FROM t GROUP BY X HAVING count(DISTINCT Y) > 1;
    ⇒ any row means either the FD is false, or your data is corrupt.
      Both are worth knowing.
```

That fourth one is the most immediately useful thing in this topic, and almost nobody runs it.

---

## The physical reality

### An FD is a constraint the engine may or may not be able to enforce

```
 X → Y  is a business rule. Where it lives determines whether the
 database can enforce it.

 CASE A — X is a candidate key of the table
   customers(id PK, name, phone, city)
   id → name, phone, city
   ⇒ ★ AUTOMATICALLY ENFORCED. The PRIMARY KEY index guarantees one
     row per id, so there is physically only one answer.

 CASE B — X is not a key, but is unique in this table
   customers(id PK, email UNIQUE, name)
   email → name
   ⇒ ENFORCED, by the UNIQUE index. Same mechanism.

 CASE C — X is not unique in this table          ★ THE PROBLEM CASE
   orders_flat(order_id, product_id, customer_id, customer_phone, …)
   customer_id → customer_phone
   ⇒ ★ CANNOT BE ENFORCED. customer_id repeats, and a CHECK cannot
     see other rows (Topic 24). Nothing prevents two different phone
     numbers for one customer_id.
   ⇒ THIS IS EXACTLY THE UPDATE ANOMALY FROM TOPIC 29, restated
     precisely: an FD whose determinant is not a key of the table
     it lives in.

 ★ THE UNIFYING INSIGHT:
   NORMALISATION = MOVING EVERY FD INTO A TABLE WHERE ITS DETERMINANT
   IS A KEY, so that the engine enforces it for free.
   That single sentence is what 2NF, 3NF and BCNF are all doing.
```

### Detecting an FD violation in live data

```sql
-- CLAIM: customer_id → customer_phone
SELECT customer_id, count(DISTINCT customer_phone) AS distinct_values
FROM orders_flat
GROUP BY customer_id
HAVING count(DISTINCT customer_phone) > 1;
```
```
 customer_id | distinct_values
-------------+-----------------
           7 |               2       ★ the FD is violated in the data
```
```
 TWO POSSIBLE MEANINGS, and you must decide which:
  ① the FD is TRUE and the data is CORRUPT
     → an update anomaly already happened. Fix the data, then normalise.
  ② the FD is FALSE — customers legitimately have multiple phones
     → your model is wrong. It's a 1:N relationship (Topic 35, 4NF).
 ★ The query cannot tell you which. Only the domain can.
```

---

## How it works — step by step

### The formal definition, and what it is *not*

```
 X → Y  ("X functionally determines Y")

 FORMALLY: for any two tuples t1, t2 in the relation,
             if t1[X] = t2[X] then t1[Y] = t2[Y]

 ★ IT IS A STATEMENT ABOUT THE DOMAIN, NOT ABOUT THE CURRENT DATA.
   If your table happens to contain one row per city today, that does
   NOT mean city → mayor is false or true. You decide from the rules
   of the business.
   ⇒ Current data can DISPROVE an FD (a counterexample) but can never
     PROVE one.

 EXAMPLES:
   student_id  → student_name        ✓ one student, one name
   student_name → student_id         ✗ names are not unique
   (order_id, product_id) → quantity ✓ one line, one quantity
   order_id    → quantity            ✗ an order has many lines
   isbn        → title, author       ✓
   city        → state               ✓ in India (a city is in one state)
   state       → city                ✗ a state has many cities
```

### Types of dependency

```
 TRIVIAL          X → Y where Y ⊆ X
                  (order_id, product_id) → order_id
                  ⇒ always true, always ignored

 FULL             X → Y and NO proper subset of X determines Y
                  (order_id, product_id) → quantity
                  ⇒ you need BOTH; neither alone determines quantity

 PARTIAL          X → Y where a PROPER SUBSET of X already determines Y
                  (order_id, product_id) → order_date
                  but order_id → order_date on its own
                  ⇒ ★ this is what 2NF forbids (Topic 32)

 TRANSITIVE       X → Y and Y → Z, so X → Z indirectly,
                  where Y is not a key
                  order_id → customer_id → customer_city
                  ⇒ ★ this is what 3NF forbids (Topic 33)
```

### Armstrong's axioms — the rules for deriving new FDs

```
 THE THREE AXIOMS (sound and complete — everything derivable follows):

  REFLEXIVITY    if Y ⊆ X then X → Y                     (trivial FDs)
  AUGMENTATION   if X → Y then XZ → YZ                   (add to both sides)
  TRANSITIVITY   if X → Y and Y → Z then X → Z           ★ the useful one

 THREE DERIVED RULES (convenient shorthands):

  UNION          if X → Y and X → Z then X → YZ
  DECOMPOSITION  if X → YZ then X → Y and X → Z
  PSEUDO-TRANS.  if X → Y and WY → Z then WX → Z

 ⚠ THE ONE THAT IS NOT A RULE:
   X → Y and Z → Y does NOT give you X → Z or Z → X.
   Two things determining the same thing tells you nothing about
   their relationship. People assume this constantly and it is false.
```

### Closure — the algorithm you will actually use

```
 X⁺ = "everything X determines, directly or transitively"

 ALGORITHM:
   result := X
   REPEAT until result stops growing:
     for each FD  A → B  in F:
       if A ⊆ result then result := result ∪ B
   return result

 WORKED EXAMPLE
   R(order_id, product_id, order_date, customer_id, customer_city, quantity)
   F = { order_id → order_date, customer_id
       , customer_id → customer_city
       , (order_id, product_id) → quantity }

   Compute {order_id}⁺:
     start:  {order_id}
     apply order_id → order_date, customer_id
             {order_id, order_date, customer_id}
     apply customer_id → customer_city
             {order_id, order_date, customer_id, customer_city}
     apply (order_id,product_id) → quantity?  product_id ∉ result → no
     ⇒ {order_id}⁺ = {order_id, order_date, customer_id, customer_city}
     ⇒ NOT all attributes ⇒ order_id is NOT a superkey.

   Compute {order_id, product_id}⁺:
     start:  {order_id, product_id}
     apply order_id → order_date, customer_id     → add both
     apply customer_id → customer_city             → add
     apply (order_id,product_id) → quantity        → add
     ⇒ ALL SIX ATTRIBUTES ⇒ {order_id, product_id} IS a superkey.
     ⇒ and it is MINIMAL (neither alone works, shown above)
     ⇒ ★ therefore it is a CANDIDATE KEY.
```

### Finding all candidate keys — the practical procedure

```
 STEP 1 — classify every attribute by where it appears in F:
   LEFT-ONLY   appears only on the left of some FD, never on the right
               ★ MUST be in EVERY candidate key
   RIGHT-ONLY  appears only on the right
               ★ CANNOT be in any candidate key
   BOTH        appears on both sides    → may or may not be
   NEITHER     appears in no FD
               ★ MUST be in EVERY candidate key

 STEP 2 — let L = (left-only ∪ neither). Compute L⁺.
   If L⁺ = all attributes  → L is the ONLY candidate key. Done.

 STEP 3 — otherwise, add attributes from BOTH one at a time (then in
   pairs, etc.) and test each combination's closure. Keep the minimal
   ones.

 WORKED EXAMPLE
   R(A,B,C,D,E)  F = { A → BC, CD → E, B → D, E → A }
   left-only: none        (A appears right in E→A; B right in A→BC;
                           C right in A→BC; D right in B→D; E right in CD→E)
   right-only: none
   neither: none
   ⇒ every attribute is BOTH. Test singletons:
     {A}⁺ = A,B,C  (A→BC), then B→D ⇒ A,B,C,D, then CD→E ⇒ ALL  ★ key
     {B}⁺ = B,D                                                  not a key
     {C}⁺ = C                                                    not a key
     {D}⁺ = D                                                    not a key
     {E}⁺ = E,A → A,B,C → D → ALL                                ★ key
   Test pairs that don't contain A or E:
     {B,C}⁺ = B,C,D → CD→E ⇒ E → A ⇒ ALL                         ★ key
     {C,D}⁺ = C,D,E → A → B ⇒ ALL                                ★ key
   ⇒ CANDIDATE KEYS: {A}, {E}, {BC}, {CD}
   ⇒ PRIME attributes: A, B, C, D, E — all of them.
   ⇒ (a relation where every attribute is prime is automatically in 3NF
      — see Topic 33.)
```

### Minimal cover — simplifying a set of FDs

```
 A MINIMAL (CANONICAL) COVER of F is an equivalent set where:
   ① every right-hand side is a SINGLE attribute
   ② no FD is redundant (removing it doesn't change what F implies)
   ③ no left-hand side has a redundant attribute

 PROCEDURE:
   ① split RHSs:  A → BC   becomes   A → B, A → C
   ② remove redundant FDs: for each A → B, check whether B ∈ A⁺
      computed WITHOUT that FD. If so, drop it.
   ③ remove extraneous LHS attributes: for AB → C, check whether
      C ∈ {A}⁺ or C ∈ {B}⁺. If so, shrink the left side.

 ★ WHY YOU CARE: the 3NF synthesis algorithm (Topic 33) works from a
   minimal cover and produces a decomposition that is BOTH lossless AND
   dependency-preserving. Without a minimal cover it produces redundant
   tables.
```

### The two correctness properties of a decomposition

```
 LOSSLESS JOIN — you can reconstruct R exactly
   Decomposing R into R1, R2 is lossless IF
     (R1 ∩ R2) → R1   OR   (R1 ∩ R2) → R2
   i.e. the SHARED attributes must be a key of at least one piece.
   ⇒ ★ NON-NEGOTIABLE. A lossy decomposition invents rows that were
     never there. (Topic 36 shows one.)

 DEPENDENCY PRESERVATION — every FD can still be checked on ONE table
   If an FD's attributes end up split across two tables, enforcing it
   requires a join — which means the engine cannot enforce it with a
   constraint.
   ⇒ DESIRABLE, but sometimes impossible.
   ⇒ ★ 3NF can ALWAYS achieve both. BCNF sometimes cannot achieve
     dependency preservation — which is exactly why 3NF is sometimes
     the correct final answer (Topic 34).
```

---

## Concept breakdown

```
FUNCTIONAL DEPENDENCY  X → Y
│  └── if two rows agree on X, they MUST agree on Y.
│      A statement about the DOMAIN. Data can disprove it, never prove it.
│
├── TRIVIAL      Y ⊆ X                              (ignore)
├── FULL         no proper subset of X determines Y
├── PARTIAL      a proper subset of X determines Y   → ★ 2NF forbids
└── TRANSITIVE   X → Y → Z, Y not a key             → ★ 3NF forbids

KEYS
├── SUPERKEY        determines every attribute
├── CANDIDATE KEY   a MINIMAL superkey (there may be several)
├── PRIMARY KEY     the candidate key you picked (Topic 22)
├── PRIME ATTRIBUTE in SOME candidate key
└── NON-PRIME       in NO candidate key

CLOSURE  X⁺
└── everything X determines. The one algorithm you'll actually run.
    X is a superkey ⟺ X⁺ = all attributes.

ARMSTRONG'S AXIOMS
├── reflexivity · augmentation · transitivity          (sound & complete)
└── derived: union · decomposition · pseudo-transitivity
    ⚠ X→Y and Z→Y does NOT imply anything about X and Z.

MINIMAL COVER
└── single-attribute RHS · no redundant FD · no extraneous LHS attribute
    ⇒ the input to 3NF synthesis.

DECOMPOSITION CORRECTNESS
├── LOSSLESS JOIN            shared attributes are a key of one piece
│                            ★ mandatory
└── DEPENDENCY PRESERVATION  every FD checkable on one table
                             ★ desirable; 3NF always achieves it,
                               BCNF sometimes cannot

★ THE UNIFYING SENTENCE
  Normalisation moves every FD into a table where its determinant is a
  key — so the engine enforces it for free with a PRIMARY KEY or UNIQUE
  index. An FD whose determinant is not a key is precisely an
  unenforceable rule, which is precisely an update anomaly.
```

---

## Diagrams

**Diagram 1 — big picture: FDs, keys, and the normal forms**

```
                      A SET OF FUNCTIONAL DEPENDENCIES  F
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
               CLOSURE X⁺     CANDIDATE KEYS   PRIME/NON-PRIME
                    │               │               │
                    └───────────────┼───────────────┘
                                    ▼
                     ┌──────────────────────────────┐
                     │  IS EACH FD's DETERMINANT    │
                     │  A KEY OF ITS TABLE?         │
                     └──────────────┬───────────────┘
                    ┌───────────────┼────────────────┐
                    ▼               ▼                ▼
            determinant is    determinant is    determinant is
            PART of a key     a NON-KEY         a non-key that
                    │         (non-prime RHS)   determines a PRIME
                    ▼               ▼            attribute
              ✗ 2NF violation  ✗ 3NF violation        ▼
              (Topic 32)       (Topic 33)      ✓ 3NF, ✗ BCNF
                                                (Topic 34)
```

**Diagram 2 — data flow: closure, step by step**

```
  F = { A → B,  B → C,  CD → E }
  Compute {A,D}⁺ :

  result = {A, D}
     │
     ├─ A → B ?   A ⊆ result ✓  →  result = {A, D, B}
     │
     ├─ B → C ?   B ⊆ result ✓  →  result = {A, D, B, C}
     │
     ├─ CD → E ?  {C,D} ⊆ result ✓ → result = {A, D, B, C, E}
     │
     └─ no more FDs apply  →  STOP

  {A,D}⁺ = {A, B, C, D, E} = ALL ATTRIBUTES
  ⇒ {A,D} is a SUPERKEY.
  Is it minimal?  {A}⁺ = {A,B,C} ✗   {D}⁺ = {D} ✗
  ⇒ ★ {A,D} is a CANDIDATE KEY.
```

**Diagram 3 — before/after: an FD in the wrong table**

```
 BEFORE — the FD's determinant is NOT a key of the table
 ┌───────────────────────────────────────────────────────────────┐
 │ orders_flat (order_id, product_id, customer_id, cust_phone,…) │
 │   PK = (order_id, product_id)                                 │
 │                                                               │
 │   FD:  customer_id → cust_phone                               │
 │        └─────┬─────┘                                          │
 │        NOT a key of this table                                │
 │                                                               │
 │   ⇒ customer_id repeats                                       │
 │   ⇒ nothing prevents two phone values for one customer_id     │
 │   ⇒ ★ NO CONSTRAINT CAN EXPRESS THIS RULE HERE                │
 └───────────────────────────────────────────────────────────────┘

 AFTER — the FD moved to a table where its determinant IS the key
 ┌───────────────────────────────────────────────────────────────┐
 │ customers (id PK, phone, …)                                   │
 │   FD:  id → phone                                             │
 │        └┬┘                                                     │
 │        the PRIMARY KEY                                        │
 │                                                               │
 │   ⇒ ★ ENFORCED FOR FREE by the PK index. One row per id means │
 │     one phone per id, physically.                             │
 └───────────────────────────────────────────────────────────────┘
        ↑ THIS IS WHAT NORMALISATION DOES. Every step of 2NF, 3NF and
          BCNF is an instance of this one move.
```

---

## Example 1 — basic

**Step 1 — write down the FDs for a real table.**

```sql
CREATE TABLE enrolments (
  student_id   bigint,
  student_name text,
  student_email text,
  course_code  text,
  course_title text,
  course_credits int,
  instructor_id bigint,
  instructor_name text,
  semester     text,
  grade        text,
  PRIMARY KEY (student_id, course_code, semester)
);
```

```
 F, derived from the DOMAIN (not from the data):
   f1: student_id → student_name, student_email
   f2: course_code → course_title, course_credits
   f3: (course_code, semester) → instructor_id
   f4: instructor_id → instructor_name
   f5: (student_id, course_code, semester) → grade
   f6: student_email → student_id          ← emails are unique
```

**Step 2 — find the candidate keys.**

```
 Attributes: student_id, student_name, student_email, course_code,
             course_title, course_credits, instructor_id,
             instructor_name, semester, grade

 LEFT-ONLY (never on a right-hand side):  semester, course_code
   ⇒ ★ MUST be in every candidate key
 RIGHT-ONLY: student_name, course_title, course_credits,
             instructor_name, grade, instructor_id
   ⇒ CANNOT be in any candidate key
 BOTH: student_id, student_email

 L = {course_code, semester}
 L⁺ = {course_code, semester}
      + f2 → course_title, course_credits
      + f3 → instructor_id
      + f4 → instructor_name
      = {course_code, semester, course_title, course_credits,
         instructor_id, instructor_name}
   ⇒ missing student_*, grade. NOT a superkey.

 Add student_id:
   {course_code, semester, student_id}⁺
      + f1 → student_name, student_email
      + f5 → grade
      + everything above
      = ALL   ★ SUPERKEY, and minimal ⇒ CANDIDATE KEY

 Add student_email instead:
   {course_code, semester, student_email}⁺
      + f6 → student_id
      + f1 → student_name
      + f5 → grade
      = ALL   ★ ALSO A CANDIDATE KEY

 ⇒ TWO candidate keys:
      CK1 = (student_id, course_code, semester)
      CK2 = (student_email, course_code, semester)
 ⇒ PRIME attributes: student_id, student_email, course_code, semester
 ⇒ NON-PRIME: student_name, course_title, course_credits,
              instructor_id, instructor_name, grade
```

**★ The practical output:** you just discovered that `student_email` should have a `UNIQUE` constraint. Nothing in the DDL said so, but f6 does — and if the constraint isn't there, duplicate emails are possible and CK2 is a lie.

**Step 3 — classify each FD.**

```
 f1: student_id → student_name, student_email
     student_id is PART of CK1  ⇒ ★ PARTIAL dependency  → 2NF violation
 f2: course_code → course_title, credits
     course_code is PART of CK1 ⇒ ★ PARTIAL              → 2NF violation
 f3: (course_code, semester) → instructor_id
     PART of CK1                ⇒ ★ PARTIAL              → 2NF violation
 f4: instructor_id → instructor_name
     instructor_id is NON-PRIME ⇒ ★ TRANSITIVE           → 3NF violation
 f5: (student_id, course_code, semester) → grade
     the FULL candidate key     ⇒ ✓ fine
 f6: student_email → student_id
     student_email is PRIME, and it's a determinant that is NOT a
     candidate key by itself   ⇒ ✓ 3NF-ok (RHS is prime), ✗ BCNF
```

**Step 4 — verify the FDs against live data.**

```sql
INSERT INTO enrolments VALUES
 (1,'Arjun','arjun@uni.in','CS101','Algorithms',4,101,'Dr Rao','2026-S1','A'),
 (1,'Arjun','arjun@uni.in','CS102','Databases', 4,102,'Dr Iyer','2026-S1','B'),
 (2,'Meera','meera@uni.in','CS101','Algorithms',4,101,'Dr Rao','2026-S1','A');

-- f1: student_id → student_name ?
SELECT student_id, count(DISTINCT student_name) FROM enrolments
GROUP BY 1 HAVING count(DISTINCT student_name) > 1;      -- 0 rows ✓

-- now corrupt it, exactly as a partial UPDATE would
UPDATE enrolments SET student_name='Arjun Sharma'
WHERE student_id=1 AND course_code='CS101';

SELECT student_id, count(DISTINCT student_name) AS variants FROM enrolments
GROUP BY 1 HAVING count(DISTINCT student_name) > 1;
```
```
 student_id | variants
------------+----------
          1 |        2      ★ f1 violated. The schema PERMITS this.
```
**Run this query for every FD you claim. Any row means either the FD is false or the data is already corrupt.**

**Step 5 — closure, in SQL, as a sanity check.**

```sql
-- "is (course_code, semester) a superkey?" — i.e. does it determine
-- every attribute uniquely?
SELECT course_code, semester,
       count(DISTINCT student_id) AS students,
       count(DISTINCT instructor_id) AS instructors
FROM enrolments GROUP BY 1,2;
```
```
 course_code | semester | students | instructors
-------------+----------+----------+-------------
 CS101       | 2026-S1  |        2 |           1
```
`students = 2` proves `(course_code, semester)` is **not** a superkey — it does not determine `student_id`. `instructors = 1` is consistent with f3, though it cannot prove it.

**Step 6 — the decomposition that follows.**

```sql
CREATE TABLE students (
  id bigint PRIMARY KEY,
  name text NOT NULL,
  email citext NOT NULL UNIQUE          -- ★ f6, now enforced
);
CREATE TABLE courses (
  code text PRIMARY KEY,
  title text NOT NULL,
  credits int NOT NULL CHECK (credits > 0)
);
CREATE TABLE instructors (
  id bigint PRIMARY KEY,
  name text NOT NULL
);
CREATE TABLE course_offerings (
  course_code text NOT NULL REFERENCES courses(code),
  semester text NOT NULL,
  instructor_id bigint NOT NULL REFERENCES instructors(id),
  PRIMARY KEY (course_code, semester)    -- ★ f3, now enforced
);
CREATE TABLE enrolments (
  student_id bigint NOT NULL REFERENCES students(id),
  course_code text NOT NULL,
  semester text NOT NULL,
  grade text,
  PRIMARY KEY (student_id, course_code, semester),   -- ★ f5, now enforced
  FOREIGN KEY (course_code, semester) REFERENCES course_offerings(course_code, semester)
);
```

**Every FD now lives in a table where its determinant is that table's key**, so every one is enforced by an index rather than by hope.

---

## Example 2 — production scenario

**The situation.** You inherit a `subscriptions` table. 180 million rows. Nobody documented the rules. Finance reports disagree with the billing system. Your job is to find out what is actually true.

```sql
CREATE TABLE subscriptions (
  id bigserial PRIMARY KEY,
  account_id bigint NOT NULL,
  account_name text NOT NULL,
  account_tier text NOT NULL,
  plan_code text NOT NULL,
  plan_name text NOT NULL,
  plan_price_minor bigint NOT NULL,
  plan_billing_period text NOT NULL,
  seats int NOT NULL,
  discount_pct int NOT NULL DEFAULT 0,
  started_at timestamptz NOT NULL,
  ended_at timestamptz NULL,
  monthly_total_minor bigint NOT NULL
);
```

**Step 1 — discover the FDs from the data.** You can't prove an FD, but you can *find candidates* and *disprove* wrong ones.

```sql
-- a generic FD candidate detector
CREATE OR REPLACE FUNCTION fd_check(tbl text, lhs text, rhs text)
RETURNS TABLE(lhs_expr text, rhs_expr text, groups bigint, violations bigint) AS $$
BEGIN
  RETURN QUERY EXECUTE format($f$
    SELECT %L, %L, count(*)::bigint,
           count(*) FILTER (WHERE n > 1)::bigint
    FROM (SELECT %s, count(DISTINCT (%s)) AS n FROM %I GROUP BY %s) t
  $f$, lhs, rhs, lhs, rhs, tbl, lhs);
END $$ LANGUAGE plpgsql;

SELECT * FROM fd_check('subscriptions','account_id','account_name')
UNION ALL SELECT * FROM fd_check('subscriptions','account_id','account_tier')
UNION ALL SELECT * FROM fd_check('subscriptions','plan_code','plan_name')
UNION ALL SELECT * FROM fd_check('subscriptions','plan_code','plan_price_minor')
UNION ALL SELECT * FROM fd_check('subscriptions','plan_code','plan_billing_period');
```
```
  lhs_expr  |      rhs_expr       | groups  | violations
------------+---------------------+---------+------------
 account_id | account_name        |  412008 |       8412   ⚠
 account_id | account_tier        |  412008 |     104882   ⚠⚠
 plan_code  | plan_name           |      12 |          4   ⚠
 plan_code  | plan_price_minor    |      12 |         12   ⚠⚠⚠
 plan_code  | plan_billing_period |      12 |          0   ✓
```

**Step 2 — interpret each result. This is where judgement enters.**

```
 plan_code → plan_billing_period    0 violations
   ⇒ consistent with the FD. Plausible and safe to assume.

 plan_code → plan_name              4 of 12 violated
   ⇒ ★ almost certainly an UPDATE ANOMALY. Plans get renamed
     ("Pro" → "Professional") and old rows keep the old name.
     THE FD IS TRUE; THE DATA IS CORRUPT.
   Confirm:
     SELECT plan_code, array_agg(DISTINCT plan_name) FROM subscriptions
     GROUP BY 1 HAVING count(DISTINCT plan_name)>1;
     → {Pro, Professional} · {Team, Teams} — clearly renames.

 plan_code → plan_price_minor       ALL 12 violated
   ⇒ ★ THE FD IS FALSE. Prices change over time. The price on a
     subscription row is a SNAPSHOT (Topics 26, 29), not a copy.
     The real FD is  (plan_code, started_at) → price
     ⇒ this is a TEMPORAL dependency, and it belongs in a
       plan_prices table with a validity range (Topic 26).

 account_id → account_tier          104,882 of 412,008 violated (25%)
   ⇒ ★ AMBIGUOUS. Two readings:
     (a) tiers change over time → it's a snapshot, like price
     (b) the tier is genuinely per-subscription, not per-account
   ⇒ THE DATA CANNOT DECIDE. Ask the business.
     Answer (from finance): tiers are per-account and change on
     upgrade. ⇒ it's (a): a temporal fact.

 account_id → account_name          8,412 violated (2%)
   ⇒ ★ CORRUPTION. Accounts get renamed; 2% is too low to be a
     modelled behaviour. THE FD IS TRUE, THE DATA IS CORRUPT.
```

**★ This is the most important skill in the topic:** a violation count tells you *something is wrong*, and only the domain tells you *which thing*. A high violation rate suggests the FD is false (a real 1:N or temporal relationship); a low rate suggests corruption.

**Step 3 — the derived column, and the FD it hides.**

```sql
SELECT count(*) FROM subscriptions
WHERE monthly_total_minor
   <> round(plan_price_minor * seats * (100 - discount_pct) / 100.0);
```
```
 count
-------
  2104        ★ 2,104 rows where the stored total disagrees with its inputs
```
```
 ⇒ the FD is:  (plan_price_minor, seats, discount_pct) → monthly_total_minor
 ⇒ it is DERIVED. It should not be stored at all — or if it is stored
   for performance, it must be a GENERATED column so it CANNOT disagree
   (Topic 24).
 ⇒ 2,104 rows are billing the customer an amount that doesn't match
   their plan. Finance's discrepancy, found.
```

**Step 4 — the candidate keys, and what's missing.**

```sql
-- is (account_id, plan_code, started_at) unique?
SELECT count(*) AS total,
       count(DISTINCT (account_id, plan_code, started_at)) AS distinct_triples
FROM subscriptions;
```
```
  total    | distinct_triples
-----------+------------------
 180412008 |        180411996      ← 12 duplicates
```
```sql
SELECT account_id, plan_code, started_at, count(*) FROM subscriptions
GROUP BY 1,2,3 HAVING count(*) > 1;
```
```
 account_id | plan_code |       started_at       | count
------------+-----------+------------------------+-------
      88214 | PRO       | 2025-06-01 00:00:00+00 |     2      ★ double-billed
```
**Twelve accounts have been billed twice** because the natural candidate key has no `UNIQUE` constraint. The `id` surrogate hid it.

**Step 5 — the corrected model.**

```sql
CREATE TABLE accounts (
  id bigint PRIMARY KEY,
  name text NOT NULL                      -- ★ account_id → account_name, enforced
);

CREATE TABLE account_tiers (              -- ★ the temporal fact
  account_id bigint NOT NULL REFERENCES accounts(id),
  tier text NOT NULL,
  valid tstzrange NOT NULL,
  EXCLUDE USING gist (account_id WITH =, valid WITH &&)
);

CREATE TABLE plans (
  code text PRIMARY KEY,
  name text NOT NULL,                     -- ★ plan_code → plan_name, enforced
  billing_period text NOT NULL            -- ★ plan_code → billing_period, enforced
);

CREATE TABLE plan_prices (                -- ★ the temporal fact
  plan_code text NOT NULL REFERENCES plans(code),
  price_minor bigint NOT NULL CHECK (price_minor >= 0),
  valid tstzrange NOT NULL,
  EXCLUDE USING gist (plan_code WITH =, valid WITH &&)
);

CREATE TABLE subscriptions (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  account_id bigint NOT NULL REFERENCES accounts(id),
  plan_code text NOT NULL REFERENCES plans(code),
  seats int NOT NULL CHECK (seats > 0),
  discount_pct int NOT NULL DEFAULT 0 CHECK (discount_pct BETWEEN 0 AND 100),
  -- ★ the SNAPSHOT: what we actually charge for THIS subscription
  charged_price_minor bigint NOT NULL CHECK (charged_price_minor >= 0),
  -- ★ GENERATED: cannot disagree with its inputs
  monthly_total_minor bigint GENERATED ALWAYS AS
    (charged_price_minor * seats * (100 - discount_pct) / 100) STORED,
  started_at timestamptz NOT NULL,
  ended_at timestamptz NULL,
  -- ★ the candidate key that was missing
  CONSTRAINT uq_sub UNIQUE (account_id, plan_code, started_at),
  CHECK (ended_at IS NULL OR ended_at > started_at)
);
```

**Step 6 — what the FD analysis found, summarised.**

| Finding | Evidence | Consequence |
|---|---|---|
| `plan_code → plan_name` violated | 4/12 plans | rename corruption; reports split by plan |
| `plan_code → price` **false** | 12/12 | a temporal fact, needs a validity range |
| `account_id → tier` **false** | 25% | a temporal fact |
| `account_id → name` violated | 2% | corruption |
| derived total disagrees | 2,104 rows | **customers billed wrong** |
| candidate key not enforced | 12 duplicates | **accounts double-billed** |

**Two of the six are money bugs**, and both were found by writing down dependencies and testing them — not by reading code.

---

## Common mistakes

**1. Deriving FDs from the data instead of the domain.**
- *Symptom:* "there's only one row per city, so city is a key."
- *Why it's wrong:* current data can *disprove* an FD but never *prove* one. Today's uniqueness is an accident.
- *Fix:* FDs come from business rules. Use data only to find counterexamples.

**2. Assuming the surrogate `id` is the candidate key.**
- *Symptom:* duplicate business rows despite a primary key.
- *Why:* `id` is a superkey by construction and tells you nothing. The *natural* candidate key is what needs a `UNIQUE` constraint.
- *Fix:* find the real candidate keys from the FDs, and declare them.

**3. Confusing a violated FD with a false FD.**
- *Symptom:* removing a "redundant" column that was actually a snapshot.
- *Fix:* look at the violation *rate* and ask the domain. High rate → the FD is probably false (temporal or 1:N). Low rate → corruption.

**4. Believing `X → Y` and `Z → Y` implies a relationship between X and Z.**
- *Symptom:* an invalid decomposition.
- *Fix:* it doesn't. Only the three axioms hold.

**5. Ignoring dependency preservation.**
- *Symptom:* a decomposition where enforcing a business rule needs a join, so it can't be a constraint.
- *Fix:* check it. 3NF can always preserve dependencies; BCNF sometimes can't (Topic 34).

**6. Not checking losslessness.**
- *Symptom:* joining the pieces back produces rows that never existed.
- *Fix:* the shared attributes must be a key of at least one piece. (Topic 36 shows a lossy example.)

**7. Storing a derived value without `GENERATED`.**
- *Symptom:* the stored total disagrees with its inputs on 2,104 rows.
- *Fix:* a derived value is an FD from its inputs. Either don't store it, or make it `GENERATED ALWAYS AS … STORED` so it cannot disagree (Topic 24).

---

## Hands-on proof

**PROVE IT #1 — the FD violation detector.** (Example 1, step 4.)

**PROVE IT #2 — the generic checker.**
```sql
-- (the fd_check function from Example 2, step 1)
SELECT * FROM fd_check('enrolments','student_id','student_name');
SELECT * FROM fd_check('enrolments','course_code','course_title');
SELECT * FROM fd_check('enrolments','(course_code, semester)','instructor_id');
```

**PROVE IT #3 — data can disprove but not prove.**
```sql
CREATE TABLE t (a int, b int);
INSERT INTO t VALUES (1,10),(2,20),(3,30);
-- "a → b holds!"  — it does, in this data.
SELECT a, count(DISTINCT b) FROM t GROUP BY a HAVING count(DISTINCT b)>1;  -- 0 rows
INSERT INTO t VALUES (1,99);
-- ★ now it doesn't. The FD was never proven; it was just untested.
```

**PROVE IT #4 — find the real candidate key.**
```sql
-- test every plausible combination for uniqueness
SELECT count(*) AS total,
       count(DISTINCT (account_id, plan_code, started_at)) AS ck1,
       count(DISTINCT (account_id, started_at)) AS ck2,
       count(DISTINCT account_id) AS ck3
FROM subscriptions;
-- the smallest combination whose distinct count equals total is your
-- candidate key. Then DECLARE IT.
```

**PROVE IT #5 — a derived column that disagrees with its inputs.**
```sql
SELECT count(*) FROM subscriptions
WHERE monthly_total_minor <> round(plan_price_minor*seats*(100-discount_pct)/100.0);
-- then fix it permanently:
ALTER TABLE subscriptions DROP COLUMN monthly_total_minor;
ALTER TABLE subscriptions ADD COLUMN monthly_total_minor bigint
  GENERATED ALWAYS AS (charged_price_minor*seats*(100-discount_pct)/100) STORED;
```

**PROVE IT #6 — lossless vs lossy decomposition.**
```sql
CREATE TABLE r (a int, b int, c int);
INSERT INTO r VALUES (1,10,100),(2,10,200);

-- LOSSLESS: split on b→? no. Split so the shared attribute is a key of one piece.
CREATE TABLE r1 AS SELECT DISTINCT a, b FROM r;      -- a is a key of r1
CREATE TABLE r2 AS SELECT DISTINCT a, c FROM r;      -- a is a key of r2
SELECT count(*) FROM (SELECT r1.a,r1.b,r2.c FROM r1 JOIN r2 USING (a)) x;   -- 2 ✓

-- LOSSY: shared attribute b is a key of NEITHER piece
CREATE TABLE s1 AS SELECT DISTINCT a, b FROM r;
CREATE TABLE s2 AS SELECT DISTINCT b, c FROM r;
SELECT count(*) FROM (SELECT s1.a,s1.b,s2.c FROM s1 JOIN s2 USING (b)) x;
```
```
 count
-------
     4        ★ FOUR rows from an original TWO. Two are invented.
```

---

## The design decision framework

```
THE PROCEDURE — for any table you are designing or auditing:

 ① WRITE DOWN THE FDs, FROM THE DOMAIN.
    For each column, ask: "what determines this?"
    Get them confirmed by someone who knows the business.

 ② TEST EACH FD AGAINST THE DATA.
      SELECT <lhs>, count(DISTINCT <rhs>) FROM t
      GROUP BY 1 HAVING count(DISTINCT <rhs>) > 1;
    • 0 violations           → consistent; proceed
    • FEW violations (<5%)   → the FD is TRUE, the data is CORRUPT.
                               Fix the data. You have found an update
                               anomaly that already happened.
    • MANY violations (>20%) → the FD is probably FALSE. Ask why:
                               temporal? a real 1:N? a snapshot?

 ③ FIND ALL CANDIDATE KEYS.
    left-only ∪ neither must be in every key. Compute closures.
    ⇒ ★ DECLARE THEM AS UNIQUE CONSTRAINTS. This is the single most
      valuable practical output of the analysis, and it's usually missing.

 ④ CLASSIFY EVERY FD.
    determinant = a full candidate key      → ✓ fine
    determinant = PART of a candidate key   → ✗ 2NF violation (T32)
    determinant = a non-key, RHS non-prime  → ✗ 3NF violation (T33)
    determinant = a non-key, RHS prime      → 3NF-ok, ✗ BCNF (T34)

 ⑤ BEFORE DECOMPOSING, CHECK IT'S NOT A SNAPSHOT.
    "If the source value changes, SHOULD this change too?"
      YES → a copy. Normalise it away.
      NO  → a snapshot. Keep it. (Topics 26, 29)

 ⑥ VERIFY THE DECOMPOSITION.
    LOSSLESS:  shared attributes are a key of at least one piece.
               ★ mandatory
    DEPENDENCY-PRESERVING: every FD checkable on one table.
               ★ desirable; if you must lose one, prefer 3NF (T34)

THE SIGNAL TO LOOK FOR:
  Run ② for every FD you believe. Then:
  • any low-rate violation      → corruption. Fix it, then add the
                                   constraint that prevents recurrence.
  • any derived column          → make it GENERATED, or drop it
  • any natural key with no
    UNIQUE constraint           → ★ duplicates are already possible,
                                   and probably already exist
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Given `R(A,B,C,D,E)` with `F = { A → BC, CD → E, B → D, E → A }`:
(a) compute `{A}⁺`, `{B}⁺`, `{BC}⁺`, `{CD}⁺`,
(b) find all candidate keys, showing your working,
(c) list the prime and non-prime attributes,
(d) for each FD, state whether its determinant is a candidate key.

### Exercise 2 — medium (apply it)
Take the `enrolments` table from Example 1.
(a) Write out all FDs from the domain and justify each.
(b) Find both candidate keys with the closure algorithm.
(c) Build the table, insert data, and use the `fd_check` function to test every FD.
(d) Deliberately corrupt one FD with a partial `UPDATE`, then detect it.
(e) Decompose the table and prove the decomposition is lossless by joining the pieces and comparing row counts.
(f) Identify which FD is *not* dependency-preserved by your decomposition, if any.

### Exercise 3 — hard (production simulation)
You inherit a 180M-row `subscriptions` table with the columns from Example 2 and no documentation. Finance reports disagree with billing.

(a) Write a generic FD-discovery query that, for every pair of columns, reports the violation rate. (Hint: `information_schema.columns` + dynamic SQL.)
(b) Run it conceptually and classify each result as: FD true & data corrupt / FD false & temporal / FD false & 1:N / derived value.
(c) Two findings are money bugs. Identify them and explain how each happened.
(d) `account_id → account_tier` has a 25% violation rate. Give both possible interpretations, say what evidence would distinguish them, and what you'd ask the business.
(e) Find the real candidate key of the table, prove it isn't enforced, and quantify the damage.
(f) Design the corrected schema. Show which FD each constraint now enforces.
(g) One column must remain "redundant." Identify it and justify it in FD terms.
(h) Write the monitoring query that would catch a recurrence of each of the six findings.

---

## Mental model checkpoint

1. Define `X → Y` precisely. Why can data disprove an FD but never prove one?
2. What's the difference between a superkey, a candidate key, and a prime attribute?
3. State Armstrong's three axioms. What common inference is *not* valid?
4. Run the closure algorithm on `{A}` given `F = {A→B, B→C, CD→E}`. Is `{A}` a superkey?
5. How do you find candidate keys? What role do left-only and right-only attributes play?
6. What are the two correctness properties of a decomposition? Which is mandatory, and which can 3NF always achieve but BCNF sometimes cannot?
7. An FD check returns 25% violations. Give two interpretations and say what distinguishes them.

---

## Quick reference card

**Definition:** `X → Y` — any two rows agreeing on X must agree on Y. A statement about the **domain**.

| Term | Meaning |
|---|---|
| Trivial FD | `Y ⊆ X` |
| Full FD | no proper subset of X determines Y |
| **Partial FD** | a proper subset of a key determines Y → **2NF violation** |
| **Transitive FD** | `X → Y → Z`, Y not a key → **3NF violation** |
| Superkey | determines all attributes (`X⁺` = everything) |
| Candidate key | a minimal superkey |
| Prime attribute | in some candidate key |

**Armstrong:** reflexivity · augmentation · **transitivity**
Derived: union · decomposition · pseudo-transitivity
⚠ `X→Y` and `Z→Y` implies **nothing** about X and Z.

**Closure algorithm**
```
result := X
repeat until stable:
  for each A → B in F:  if A ⊆ result then result := result ∪ B
```

**Finding candidate keys:** left-only ∪ neither must be in every key. Compute its closure; extend with `both` attributes until minimal superkeys are found.

**Decomposition correctness**

| Property | Condition | Status |
|---|---|---|
| Lossless join | shared attributes are a key of one piece | **mandatory** |
| Dependency preservation | every FD checkable on one table | desirable; 3NF always, BCNF sometimes not |

**The detection query — run it for every FD you believe**
```sql
SELECT <lhs>, count(DISTINCT <rhs>) FROM t GROUP BY 1
HAVING count(DISTINCT <rhs>) > 1;
```
Few violations → the data is corrupt. Many → the FD is false.

**The unifying sentence:** normalisation moves every FD into a table where its determinant is a key, so the engine enforces it for free.

---

## When would I use this at work?

1. **Auditing any inherited table.** The FD checker finds update anomalies that have already happened, missing `UNIQUE` constraints, and derived columns that disagree with their inputs — three classes of bug, in about ten minutes.

2. **Deciding whether a column is redundant or a snapshot.** The violation *rate* plus one question to the business ("if the source changes, should this?") settles it — and getting it wrong deletes historical accuracy.

3. **Justifying a decomposition in a design review.** "This FD's determinant isn't a key of this table, so the rule can't be enforced here" is a precise, checkable argument, unlike "this feels denormalised."

---

## Connected topics

**Understand before this:** 22 (keys), 24 (constraints — what an enforceable FD looks like), 29 (the anomalies FDs explain).

**This unlocks:**
- **31–34** — 1NF, 2NF, 3NF, BCNF: each is one sentence in this vocabulary
- **35–36** — multi-valued and join dependencies, the generalisations
- **37** — the worked example, using the closure algorithm throughout
- **38** — when to stop, including the dependency-preservation trade-off
