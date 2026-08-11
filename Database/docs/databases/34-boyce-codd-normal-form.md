# 34 — Boyce-Codd Normal Form (BCNF)
## Phase: Normalisation

---

## ELI5 — The Simple Analogy

A tuition centre. Each row of the register says: **student, subject, tutor.**

The rules are:
- A tutor teaches exactly **one** subject. Mrs Iyer teaches only Physics.
- A student can take a subject from only **one** tutor at a time.

So `(student, subject)` identifies a row — and so does `(student, tutor)`, since the tutor implies the subject. Two overlapping keys, both perfectly valid.

But there is a third rule hiding: **tutor → subject.** And "tutor" is not a key on its own — Mrs Iyer teaches forty students.

The consequence: **you cannot record that Mr Nair teaches Chemistry until a student signs up with him.** And if his last student leaves, that fact vanishes. The register is in 3NF — every column depends on a key — and it still has an anomaly.

Now here is the part that makes BCNF interesting rather than just stricter. If you split the register to fix it, you get *(student, tutor)* and *(tutor, subject)*. The anomaly is gone. But the rule **"a student takes a subject from only one tutor"** now spans both tables, so no single constraint can enforce it. **You traded one guarantee for another.**

That trade is the whole topic.

---

## Where this fits in the big picture

```
   31 1NF · 32 2NF · 33 3NF
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 34 BOYCE-CODD NORMAL FORM ← YOU ARE HERE │
        │ EVERY determinant is a candidate key     │
        │ (3NF's escape clause removed)            │
        └────────────────────┬─────────────────────┘
                             ▼
                    35 4NF · 36 5NF (rarer still)
                    38 when to stop — where this
                       trade-off is actually decided
```

| Form | Condition on every non-trivial `X → A` |
|---|---|
| 3NF | X is a superkey **or** A is prime |
| **BCNF** | **X is a superkey.** Full stop. |

---

## What is this?

A relation is in **Boyce-Codd normal form** when, for every non-trivial functional dependency `X → A`:

> **X is a superkey.**

That's it. BCNF is 3NF with the "or A is prime" escape clause removed. It is sometimes called **3.5NF** because it sits between 3NF and 4NF in strictness.

The consequence is that BCNF **cannot always be achieved while preserving dependencies** — and that is not a flaw in the definition, it is a genuine mathematical fact about some relations.

---

## Why does it matter for a backend developer?

Two reasons, one theoretical and one very practical.

```
 ① THE THEORETICAL ONE: 3NF CAN STILL HAVE ANOMALIES.
    If a non-key attribute determines a PRIME attribute, 3NF permits it,
    and you get the insert and delete anomalies anyway.

 ② ★ THE PRACTICAL ONE: THIS IS WHERE NORMALISATION STOPS BEING
    AUTOMATIC AND BECOMES A JUDGEMENT CALL.

    Every form up to 3NF is a strict improvement: decompose, gain
    guarantees, lose nothing but a join.

    BCNF is the first form where decomposing can COST you something —
    the ability to enforce a business rule with a constraint. You must
    weigh two guarantees against each other, and there is no universally
    right answer.
```

And a third, honest reason: **BCNF violations are rare.** They require overlapping candidate keys, which most schemas don't have. You will spend far more time on 3NF. But when you *do* hit one, recognising it — and knowing that the "obvious" fix has a hidden cost — is what separates a considered decision from an accident.

---

## The physical reality

### The shape that causes it

```
 A BCNF violation requires:
   ① at least TWO candidate keys
   ② that OVERLAP (share at least one attribute)
   ③ and a non-key determinant among them

 THE CANONICAL EXAMPLE
   tutoring(student, subject, tutor)
   F = { (student, subject) → tutor      ← one tutor per student+subject
       , tutor → subject }               ← a tutor teaches one subject

   Candidate keys:
     {student, subject}⁺ = {student, subject, tutor}   ✓ ALL ⇒ key
     {student, tutor}⁺   = {student, tutor, subject}   ✓ ALL ⇒ key
   ⇒ TWO keys, OVERLAPPING on `student`.
   ⇒ PRIME: student, subject, tutor — ALL of them.

   3NF check:
     (student,subject) → tutor    X is a superkey            ✓
     tutor → subject              X is NOT a superkey
                                  BUT `subject` IS PRIME     ✓ 3NF ok!
   ⇒ ★ IN 3NF.

   BCNF check:
     tutor → subject              X is NOT a superkey        ✗
   ⇒ ★ NOT IN BCNF.
```

### The anomalies that survive 3NF

```
 tutoring
 ┌──────────┬───────────┬────────────┐
 │ student  │ subject   │ tutor      │
 ├──────────┼───────────┼────────────┤
 │ Arjun    │ Physics   │ Mrs Iyer   │
 │ Meera    │ Physics   │ Mrs Iyer   │  ← 'Mrs Iyer teaches Physics'
 │ Ravi     │ Physics   │ Mrs Iyer   │     stored THREE times
 │ Arjun    │ Chemistry │ Mr Nair    │
 └──────────┴───────────┴────────────┘

 ① INSERT ANOMALY
    Mr Rao is hired to teach Biology. No student has signed up.
    ⇒ `student` is part of both candidate keys ⇒ cannot be NULL
    ⇒ ★ THE FACT CANNOT BE RECORDED.

 ② DELETE ANOMALY
    Arjun drops Chemistry — his was Mr Nair's only row.
    ⇒ ★ "Mr Nair teaches Chemistry" is GONE.

 ③ UPDATE ANOMALY
    Mrs Iyer switches to teaching Maths.
    ⇒ every row mentioning her must change. A partial update leaves
      her teaching two subjects — which the rule `tutor → subject`
      forbids, and which no constraint on THIS table can prevent.

 ⇒ ★ ALL THREE ANOMALIES, IN A TABLE THAT IS IN 3NF.
```

### The decomposition — and what it costs

```
 BCNF DECOMPOSITION (violating FD: tutor → subject)
   R1(tutor, subject)          tutor is the KEY
   R2(student, tutor)          (student, tutor) is the KEY

 ✓ LOSSLESS?  R1 ∩ R2 = {tutor}, and tutor is a KEY of R1  ⇒ YES
 ✓ Both in BCNF:
     R1: tutor → subject, tutor is the key                  ✓
     R2: no non-trivial FDs                                 ✓
 ✓ Anomalies gone: Mr Rao goes in R1 with no student.

 ✗★ DEPENDENCY PRESERVATION: LOST.
    The FD  (student, subject) → tutor  spans BOTH tables.
    Neither R1 nor R2 can enforce it.

 ⇒ WHAT THAT MEANS CONCRETELY:
    R2: (Arjun, Mrs Iyer)      -- Physics, via R1
    R2: (Arjun, Mr Sharma)     -- also Physics, via R1
    ⇒ ★ ARJUN NOW HAS TWO PHYSICS TUTORS.
      That violates the business rule. Both rows are perfectly legal
      in R2. No PRIMARY KEY, UNIQUE, CHECK or FK can stop it, because
      the rule needs a JOIN to evaluate.

 ★★★ THE TRADE, STATED PLAINLY:
   3NF:  keeps the constraint enforceable, permits the insert/delete
         anomaly and the redundancy.
   BCNF: removes the anomaly and the redundancy, makes the constraint
         unenforceable by the engine.
   ⇒ Neither is strictly better. You choose based on which failure
     costs more — see the framework.
```

---

## How it works — step by step

### The BCNF check

```
 ① find ALL candidate keys (Topic 30)
 ② for every non-trivial FD X → A in F:
      compute X⁺
      X⁺ = all attributes?  → X is a superkey  → ✓ fine
      otherwise             → ★ BCNF VIOLATION

 ★ NOTE HOW MUCH SIMPLER THIS IS THAN 3NF: no need to work out which
   attributes are prime. Just: is every determinant a superkey?

 WORKED EXAMPLE
   R(student, subject, tutor)
   F = { (student, subject) → tutor, tutor → subject }

   (student, subject)⁺ = ALL   ⇒ superkey  ✓
   tutor⁺ = {tutor, subject}   ≠ ALL       ⇒ ★ VIOLATION
```

### The decomposition algorithm

```
 WHILE some relation R is not in BCNF:
   ① pick a violating FD  X → A  in R  (X not a superkey)
   ② R1 := X⁺           (X and everything it determines)
   ③ R2 := X ∪ (R − X⁺) (X plus everything else)
   ④ replace R with R1 and R2; repeat on each

 ★ THE DECOMPOSITION IS ALWAYS LOSSLESS, because
     R1 ∩ R2 = X, and X is a KEY of R1 (X⁺ = R1 by construction).
 ★ IT IS NOT ALWAYS DEPENDENCY-PRESERVING. That is the cost.
 ⚠ AND the result can depend on WHICH violating FD you pick first —
   different choices give different (all valid) decompositions.

 APPLIED to tutoring:
   violating FD: tutor → subject
   R1 = tutor⁺ = {tutor, subject}
   R2 = {tutor} ∪ ({student,subject,tutor} − {tutor,subject})
      = {tutor, student}
   ⇒ tutors(tutor PK, subject) and enrolments(student, tutor)
```

### Proving the lost dependency, concretely

```sql
-- 3NF version: the rule IS enforceable
CREATE TABLE tutoring_3nf (
  student text NOT NULL,
  subject text NOT NULL,
  tutor   text NOT NULL,
  PRIMARY KEY (student, subject)      -- ★ enforces (student,subject) → tutor
);
INSERT INTO tutoring_3nf VALUES ('Arjun','Physics','Mrs Iyer');
INSERT INTO tutoring_3nf VALUES ('Arjun','Physics','Mr Sharma');
-- ERROR: duplicate key value violates unique constraint
--   ★ THE BUSINESS RULE IS ENFORCED.

-- BCNF version: the rule is NOT enforceable
CREATE TABLE tutors (tutor text PRIMARY KEY, subject text NOT NULL);
CREATE TABLE enrolments (student text NOT NULL, tutor text NOT NULL
                         REFERENCES tutors(tutor), PRIMARY KEY (student, tutor));
INSERT INTO tutors VALUES ('Mrs Iyer','Physics'),('Mr Sharma','Physics');
INSERT INTO enrolments VALUES ('Arjun','Mrs Iyer');
INSERT INTO enrolments VALUES ('Arjun','Mr Sharma');
-- ✓ ACCEPTED.  ★ Arjun now has two Physics tutors.
--   No constraint on either table can prevent it.
```

### What you can do about the lost dependency

```
 IF YOU CHOOSE BCNF, the rule must be enforced somewhere. Options,
 in descending order of strength:

 ① A MATERIALISED HELPER COLUMN + UNIQUE   ★ usually the best answer
    Denormalise `subject` back onto the enrolment as a GENERATED-style
    column kept in sync, then constrain it:

      ALTER TABLE enrolments ADD COLUMN subject text NOT NULL;
      ALTER TABLE enrolments ADD CONSTRAINT fk_ts
        FOREIGN KEY (tutor, subject) REFERENCES tutors(tutor, subject);
      ALTER TABLE tutors ADD CONSTRAINT uq_ts UNIQUE (tutor, subject);
      ALTER TABLE enrolments ADD CONSTRAINT uq_one_tutor
        UNIQUE (student, subject);            -- ★ THE RULE, ENFORCED

    ⇒ ★ AND NOTE WHAT THIS IS: you have reconstructed the 3NF table's
      guarantee by adding back the redundancy — with a composite FK
      ensuring the copy can never disagree.
      This is BCNF *plus* a controlled denormalisation, and it is
      frequently the right production answer. (Topic 21's Rule 8 trick.)

 ② A TRIGGER
      BEFORE INSERT: reject if the student already has a tutor for
      that subject.
    ⚠ NOT race-free without an explicit lock (Topic 24). And it must
      handle UPDATE and the concurrent-insert case.

 ③ SERIALIZABLE ISOLATION
    Let the check be a SELECT and rely on SSI to detect the conflict
    (Topic 50). Correct, but costs retries.

 ④ APPLICATION-LEVEL + RECONCILIATION
    Check in code, plus a scheduled query that finds violations.
    ⇒ ★ if you choose this, you have accepted that violations WILL
      occur; the reconciliation job is not optional (Topic 23's rule).

 ⑤ STAY IN 3NF
    ⇒ the constraint is free. You accept the insert/delete anomaly.
```

---

## Concept breakdown

```
BOYCE-CODD NORMAL FORM
│  └── for every non-trivial X → A:  X is a SUPERKEY.
│      (3NF minus the "or A is prime" escape clause)
│
├── REQUIRES, to be violated:
│     ① two or more candidate keys
│     ② that OVERLAP
│     ③ a non-key determinant
│   ⇒ ★ therefore BCNF violations are RARE. Most schemas have one
│     candidate key and are in BCNF the moment they are in 3NF.
│
├── DECOMPOSITION: R1 = X⁺,  R2 = X ∪ (R − X⁺)
│     ✓ always LOSSLESS
│     ✗ ★ NOT always DEPENDENCY-PRESERVING
│     ⚠ the result depends on which violating FD you pick first
│
└── ★ THE TRADE — the first time normalisation costs you something:
      3NF  : constraint enforceable · anomaly + redundancy remain
      BCNF : anomaly + redundancy gone · constraint NOT enforceable

WHEN 3NF ≠ BCNF — the exact condition
  a non-key attribute determines a PRIME attribute.
  ⇒ 3NF's escape clause allows it; BCNF does not.

THE PRACTICAL RESOLUTION (usually)
  BCNF decomposition + a controlled denormalisation:
    put the determined attribute back on the child, and use a
    COMPOSITE FOREIGN KEY to guarantee the copy cannot disagree,
    plus a UNIQUE to restore the lost rule.
  ⇒ you get: no insert/delete anomaly, no uncontrolled redundancy,
    AND the constraint. The cost is one extra column and one extra
    unique index on the parent.

★ 3NF vs BCNF IS THE FIRST GENUINELY DEBATABLE DECISION IN PHASE 4.
  Everything before it was a strict improvement.
```

---

## Diagrams

**Diagram 1 — big picture: where 3NF and BCNF differ**

```
                 FOR EVERY NON-TRIVIAL FD  X → A
                              │
                    Is X a SUPERKEY?
                              │
              ┌──────YES──────┴──────NO──────┐
              ▼                              ▼
        ✓ 3NF  ✓ BCNF                 Is A a PRIME attribute?
                                              │
                              ┌──────YES──────┴──────NO──────┐
                              ▼                              ▼
                       ✓ 3NF   ✗ BCNF                  ✗ 3NF  ✗ BCNF
                              │                              │
                   ★ THE GAP. Only reachable          decompose —
                     with OVERLAPPING candidate       a strict
                     keys. Rare. And decomposing      improvement,
                     may cost a dependency.           no trade-off.
```

**Diagram 2 — data flow: the two designs, and what each guarantees**

```
  3NF                                    BCNF
  ────────────────────────────           ──────────────────────────────
  tutoring(student, subject, tutor)      tutors(tutor PK, subject)
    PK (student, subject)                enrolments(student, tutor)
                                           PK (student, tutor)
  ┌──────────────────────────┐           ┌──────────────────────────┐
  │ ✓ "one tutor per student │           │ ✗ "one tutor per student │
  │   per subject" ENFORCED  │           │   per subject" — spans   │
  │   by the PRIMARY KEY     │           │   two tables. NOT        │
  │                          │           │   ENFORCEABLE.           │
  │ ✗ can't record a tutor   │           │ ✓ can record a tutor     │
  │   with no students       │           │   with no students       │
  │ ✗ last student leaves →  │           │ ✓ tutor's subject        │
  │   the tutor's subject    │           │   survives independently │
  │   is lost                │           │                          │
  │ ✗ 'Mrs Iyer → Physics'   │           │ ✓ stored ONCE            │
  │   stored once per student│           │                          │
  └──────────────────────────┘           └──────────────────────────┘

  ★ NEITHER DOMINATES. Choose by which failure costs more.
```

**Diagram 3 — before/after: the hybrid that gets both**

```
 BCNF ALONE — the rule is lost
 ┌────────────────────────────────────────────────────────────────┐
 │ tutors(tutor PK, subject)                                      │
 │ enrolments(student, tutor)  PK (student, tutor)                │
 │                                                                │
 │ INSERT (Arjun, Mrs Iyer)    -- Physics                         │
 │ INSERT (Arjun, Mr Sharma)   -- also Physics                    │
 │ ⇒ ★ BOTH ACCEPTED. Arjun has two Physics tutors.               │
 └────────────────────────────────────────────────────────────────┘

 ★ BCNF + CONTROLLED DENORMALISATION — both guarantees
 ┌────────────────────────────────────────────────────────────────┐
 │ tutors(tutor PK, subject, UNIQUE (tutor, subject))             │
 │ enrolments(student, tutor, subject,                            │
 │            PK (student, tutor),                                │
 │            UNIQUE (student, subject),        ← ★ the lost rule │
 │            FOREIGN KEY (tutor, subject)                        │
 │              REFERENCES tutors(tutor, subject))  ← ★ the copy  │
 │                                                     cannot lie │
 │                                                                │
 │ INSERT (Arjun, Mrs Iyer, 'Physics')     ✓                      │
 │ INSERT (Arjun, Mr Sharma, 'Physics')                           │
 │ ⇒ ERROR: duplicate key on uq (student, subject)   ★ ENFORCED   │
 │ INSERT (Arjun, Mrs Iyer, 'Chemistry')                          │
 │ ⇒ ERROR: FK violation — Mrs Iyer teaches Physics  ★ ENFORCED   │
 └────────────────────────────────────────────────────────────────┘
        ↑ one extra column, one extra unique index, both rules kept
```

---

## Example 1 — basic

**Step 1 — build it and confirm it's in 3NF.**

```sql
CREATE TABLE tutoring (
  student text NOT NULL,
  subject text NOT NULL,
  tutor   text NOT NULL,
  PRIMARY KEY (student, subject)
);
CREATE UNIQUE INDEX uq_tutoring_alt ON tutoring (student, tutor);  -- the 2nd key

INSERT INTO tutoring VALUES
 ('Arjun','Physics','Mrs Iyer'),
 ('Meera','Physics','Mrs Iyer'),
 ('Ravi','Physics','Mrs Iyer'),
 ('Arjun','Chemistry','Mr Nair');
```

```
 F = { (student, subject) → tutor, tutor → subject }
 Candidate keys: {student, subject} and {student, tutor}
 Prime: student, subject, tutor — ALL

 3NF:  tutor → subject: tutor is not a superkey, BUT subject is PRIME ⇒ ✓
 BCNF: tutor → subject: tutor is not a superkey ⇒ ★ VIOLATION
```

**Step 2 — the redundancy, and the anomalies.**

```sql
SELECT tutor, subject, count(*) AS stored_this_many_times
FROM tutoring GROUP BY 1,2;
```
```
   tutor   | subject   | stored_this_many_times
-----------+-----------+------------------------
 Mrs Iyer  | Physics   |                      3     ★ one fact, three rows
 Mr Nair   | Chemistry |                      1
```

```sql
-- ① INSERT ANOMALY: hire Mr Rao for Biology, no students yet
INSERT INTO tutoring (tutor, subject) VALUES ('Mr Rao','Biology');
-- ERROR: null value in column "student" violates not-null constraint

-- ② DELETE ANOMALY: Arjun drops Chemistry
DELETE FROM tutoring WHERE student='Arjun' AND subject='Chemistry';
SELECT * FROM tutoring WHERE tutor='Mr Nair';
```
```
 (0 rows)      ★ "Mr Nair teaches Chemistry" no longer exists
```

```sql
-- ③ UPDATE ANOMALY: a partial change
UPDATE tutoring SET subject='Maths' WHERE tutor='Mrs Iyer' AND student='Arjun';
SELECT tutor, array_agg(DISTINCT subject) FROM tutoring
GROUP BY 1 HAVING count(DISTINCT subject) > 1;
```
```
  tutor   |    array_agg
----------+-----------------
 Mrs Iyer | {Maths,Physics}     ★ violates tutor → subject; nothing prevented it
```

**★ All three anomalies, in a table that satisfies 3NF.**

**Step 3 — decompose to BCNF.**

```sql
DROP TABLE tutoring;
CREATE TABLE tutors (
  tutor text PRIMARY KEY,
  subject text NOT NULL
);
CREATE TABLE enrolments (
  student text NOT NULL,
  tutor text NOT NULL REFERENCES tutors(tutor) ON DELETE RESTRICT,
  PRIMARY KEY (student, tutor)
);

INSERT INTO tutors VALUES ('Mrs Iyer','Physics'),('Mr Nair','Chemistry'),
                          ('Mr Rao','Biology');          -- ★ no students needed
INSERT INTO enrolments VALUES ('Arjun','Mrs Iyer'),('Meera','Mrs Iyer'),
                              ('Ravi','Mrs Iyer'),('Arjun','Mr Nair');
```

```sql
-- ① insert anomaly: gone
SELECT * FROM tutors WHERE tutor='Mr Rao';      -- ✓ Biology, zero students

-- ② delete anomaly: gone
DELETE FROM enrolments WHERE student='Arjun' AND tutor='Mr Nair';
SELECT * FROM tutors WHERE tutor='Mr Nair';     -- ✓ still Chemistry

-- ③ update anomaly: gone — one row per tutor
UPDATE tutors SET subject='Maths' WHERE tutor='Mrs Iyer';
SELECT count(DISTINCT subject) FROM tutors WHERE tutor='Mrs Iyer';   -- always 1
```

**Step 4 — and the cost.**

```sql
-- restore Mrs Iyer to Physics and add a second Physics tutor
UPDATE tutors SET subject='Physics' WHERE tutor='Mrs Iyer';
INSERT INTO tutors VALUES ('Mr Sharma','Physics');

INSERT INTO enrolments VALUES ('Arjun','Mrs Iyer');
INSERT INTO enrolments VALUES ('Arjun','Mr Sharma');    -- ✓ ACCEPTED

SELECT e.student, t.subject, e.tutor FROM enrolments e
JOIN tutors t ON t.tutor=e.tutor WHERE e.student='Arjun';
```
```
 student | subject |   tutor
---------+---------+------------
 Arjun   | Physics | Mrs Iyer
 Arjun   | Physics | Mr Sharma      ★ TWO PHYSICS TUTORS
```
**The business rule is violated, and no constraint on either table can prevent it.** Detecting it requires a join:
```sql
SELECT e.student, t.subject, count(*) FROM enrolments e
JOIN tutors t ON t.tutor=e.tutor GROUP BY 1,2 HAVING count(*)>1;
```

**Step 5 — the hybrid that keeps both guarantees.**

```sql
DROP TABLE enrolments;
ALTER TABLE tutors ADD CONSTRAINT uq_tutor_subject UNIQUE (tutor, subject);

CREATE TABLE enrolments (
  student text NOT NULL,
  tutor   text NOT NULL,
  subject text NOT NULL,                     -- ★ the controlled copy
  PRIMARY KEY (student, tutor),
  CONSTRAINT uq_one_tutor_per_subject UNIQUE (student, subject),   -- ★ the rule
  CONSTRAINT fk_tutor_subject
    FOREIGN KEY (tutor, subject) REFERENCES tutors(tutor, subject) -- ★ the copy
    ON UPDATE CASCADE                                              --   cannot lie
);

INSERT INTO enrolments VALUES ('Arjun','Mrs Iyer','Physics');       -- ✓
INSERT INTO enrolments VALUES ('Arjun','Mr Sharma','Physics');
-- ERROR: duplicate key value violates unique constraint
--        "uq_one_tutor_per_subject"                    ★ THE RULE IS BACK

INSERT INTO enrolments VALUES ('Meera','Mrs Iyer','Chemistry');
-- ERROR: violates foreign key constraint "fk_tutor_subject"
--   ★ the copy cannot disagree with the tutor's real subject
```

```sql
-- and the copy stays correct automatically when a tutor changes subject
UPDATE tutors SET subject='Maths' WHERE tutor='Mrs Iyer';
SELECT * FROM enrolments WHERE tutor='Mrs Iyer';
```
```
 student |  tutor   | subject
---------+----------+---------
 Arjun   | Mrs Iyer | Maths        ★ ON UPDATE CASCADE kept it in sync
```

**Cost:** one extra column (~10 bytes/row) and one extra unique index on `tutors`. **Benefit:** BCNF's anomaly-freedom *and* 3NF's enforceable rule.

---

## Example 2 — production scenario

**The situation.** A clinic booking system, 4 years old. 40 million rows.

```sql
CREATE TABLE appointments (
  patient_id  bigint NOT NULL,
  slot_start  timestamptz NOT NULL,
  doctor_id   bigint NOT NULL,
  room_id     bigint NOT NULL,
  notes       text,
  PRIMARY KEY (patient_id, slot_start)
);
```

The business rules, as stated by the clinic:

1. A patient has at most one appointment per slot.
2. A doctor is in at most one room per slot.
3. A doctor has at most one appointment per slot.
4. A room hosts at most one appointment per slot.

**Step 1 — derive the FDs and find the keys.**

```
 F = { (patient_id, slot_start) → doctor_id, room_id       [rule 1]
     , (doctor_id, slot_start)  → room_id, patient_id      [rules 2,3]
     , (room_id, slot_start)    → doctor_id, patient_id    [rule 4] }

 CANDIDATE KEYS:
   {patient_id, slot_start}⁺ = ALL   ✓
   {doctor_id,  slot_start}⁺ = ALL   ✓
   {room_id,    slot_start}⁺ = ALL   ✓
 ⇒ ★ THREE candidate keys, all overlapping on `slot_start`.
 ⇒ PRIME: patient_id, doctor_id, room_id, slot_start — all except `notes`.

 BCNF check: every determinant IS a superkey  ⇒ ★ ALREADY IN BCNF.
```

**★ So far this is a well-designed table.** The interesting part is what happens when the business adds a rule.

**Step 2 — the new requirement.**

> *"A doctor is always assigned to the same room for a whole clinic session (a morning or afternoon block), not per appointment."*

```sql
ALTER TABLE appointments ADD COLUMN session_id bigint NOT NULL;
```
```
 NEW FD:  (doctor_id, session_id) → room_id

 Is (doctor_id, session_id) a superkey?
   {doctor_id, session_id}⁺ = {doctor_id, session_id, room_id}
   ≠ all attributes ⇒ ★ NOT a superkey
 Is room_id prime? YES (it's in a candidate key)
 ⇒ 3NF: ✓ OK (the escape clause)
 ⇒ BCNF: ✗ ★ VIOLATION
```

**Step 3 — the anomalies the new rule introduces.**

```sql
-- ① INSERT: assign Dr Rao to Room 4 for tomorrow morning, no bookings yet
INSERT INTO appointments (doctor_id, session_id, room_id) VALUES (12, 8891, 4);
-- ERROR: null value in column "patient_id" violates not-null constraint
--   ★ the room assignment cannot be recorded until a patient books

-- ② DELETE: the last patient in a session cancels
DELETE FROM appointments WHERE doctor_id=12 AND session_id=8891;
--   ★ "Dr Rao is in Room 4 this session" is gone. The receptionist
--     has to re-enter it, and until she does, the next booking can
--     put him in a different room.

-- ③ UPDATE: the session moves to Room 7
UPDATE appointments SET room_id=7 WHERE doctor_id=12 AND session_id=8891;
--   ⇒ touches every appointment in the session. A partial update leaves
--     the doctor in two rooms simultaneously — which rule 2 forbids and
--     which no constraint on this table can prevent.
```

```sql
-- and it has already happened
SELECT doctor_id, session_id, array_agg(DISTINCT room_id) AS rooms, count(*)
FROM appointments GROUP BY 1,2 HAVING count(DISTINCT room_id) > 1
ORDER BY 4 DESC LIMIT 5;
```
```
 doctor_id | session_id |  rooms  | count
-----------+------------+---------+-------
      1204 |     412008 | {3,7}   |    18
       882 |     398412 | {2,5}   |    14
      ...
 ⇒ 2,104 sessions where a doctor is recorded in two rooms.
   ★ The clinic's room-allocation report has been wrong for two years.
```

**Step 4 — the BCNF decomposition, and what it costs.**

```sql
-- violating FD: (doctor_id, session_id) → room_id
-- R1 = {doctor_id, session_id, room_id}
-- R2 = {doctor_id, session_id} ∪ (R − R1)
--    = {doctor_id, session_id, patient_id, slot_start, notes}

CREATE TABLE session_rooms (
  doctor_id  bigint NOT NULL REFERENCES doctors(id),
  session_id bigint NOT NULL REFERENCES sessions(id),
  room_id    bigint NOT NULL REFERENCES rooms(id),
  PRIMARY KEY (doctor_id, session_id)
);
CREATE TABLE appointments (
  patient_id bigint NOT NULL REFERENCES patients(id),
  slot_start timestamptz NOT NULL,
  doctor_id  bigint NOT NULL,
  session_id bigint NOT NULL,
  notes      text,
  PRIMARY KEY (patient_id, slot_start),
  FOREIGN KEY (doctor_id, session_id) REFERENCES session_rooms(doctor_id, session_id)
);
```

```
 ✓ ANOMALIES GONE: a room can be assigned before any booking; it
   survives the last cancellation; changing it is one row.
 ✓ The 2,104 double-room sessions become impossible.

 ✗ ★ LOST DEPENDENCIES — and this is serious here:
     (room_id, slot_start) → doctor_id, patient_id      [rule 4]
   `room_id` is no longer in `appointments`. Enforcing "a room hosts
   at most one appointment per slot" now requires a JOIN.
   ⇒ ★ TWO PATIENTS CAN BE BOOKED INTO THE SAME ROOM AT THE SAME TIME.
     That is a real-world safety problem, not a data-tidiness one.

     Also lost: (doctor_id, slot_start) → patient_id   [rule 3]
     ⇒ a doctor can be double-booked.
```

**Step 5 — the resolution: BCNF plus controlled denormalisation.**

```sql
CREATE TABLE session_rooms (
  doctor_id  bigint NOT NULL REFERENCES doctors(id),
  session_id bigint NOT NULL REFERENCES sessions(id),
  room_id    bigint NOT NULL REFERENCES rooms(id),
  PRIMARY KEY (doctor_id, session_id),
  CONSTRAINT uq_session_room UNIQUE (doctor_id, session_id, room_id),  -- ★ FK target
  CONSTRAINT uq_room_per_session UNIQUE (session_id, room_id)          -- one doctor
);                                                                     --  per room

CREATE TABLE appointments (
  patient_id bigint NOT NULL REFERENCES patients(id),
  slot_start timestamptz NOT NULL,
  doctor_id  bigint NOT NULL,
  session_id bigint NOT NULL,
  room_id    bigint NOT NULL,        -- ★ the controlled copy
  notes      text,
  PRIMARY KEY (patient_id, slot_start),                       -- rule 1 ✓
  CONSTRAINT uq_doctor_slot UNIQUE (doctor_id, slot_start),   -- rule 3 ✓
  CONSTRAINT uq_room_slot   UNIQUE (room_id,   slot_start),   -- rule 4 ✓
  CONSTRAINT fk_session_room
    FOREIGN KEY (doctor_id, session_id, room_id)              -- ★ the copy cannot
    REFERENCES session_rooms(doctor_id, session_id, room_id)  --   disagree
    ON UPDATE CASCADE
);
```

```
 ⇒ WHAT THIS ACHIEVES:
   ✓ BCNF's anomalies gone (a room assignment exists independently)
   ✓ ALL FOUR business rules enforced by the engine
   ✓ the copied room_id CANNOT disagree with the session's real room
   ✓ ON UPDATE CASCADE keeps it in sync when a session moves rooms
   COST: 8 bytes per appointment + one extra unique index on
         session_rooms. On 40M rows: ~320 MB + ~1.2 GB of index.

 ★ THIS IS THE ANSWER MOST PRODUCTION SYSTEMS SHOULD REACH.
   Not "3NF or BCNF" — BCNF for the anomaly-freedom, plus a
   composite-FK-guarded copy to restore the constraints.
```

**Step 6 — an alternative worth knowing: `EXCLUDE` for the temporal version.**

If appointments were ranges rather than fixed slots, the unique constraints become exclusion constraints (Topics 16, 24):

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE appointments ADD COLUMN during tstzrange NOT NULL;
ALTER TABLE appointments
  ADD CONSTRAINT no_room_overlap  EXCLUDE USING gist (room_id   WITH =, during WITH &&),
  ADD CONSTRAINT no_doctor_overlap EXCLUDE USING gist (doctor_id WITH =, during WITH &&),
  ADD CONSTRAINT no_patient_overlap EXCLUDE USING gist (patient_id WITH =, during WITH &&);
```
**Same three rules, generalised from "same slot" to "overlapping time" — and still engine-enforced.**

**Step 7 — results.**

| | Before | After |
|---|---|---|
| Sessions with two rooms | 2,104 | **0, impossible** |
| Room assigned before booking | impossible | **one row in `session_rooms`** |
| Assignment survives last cancellation | ✗ | **✓** |
| Rule 1 (patient/slot) | ✓ | ✓ |
| Rule 2 (doctor/session/room) | ✗ | **✓** |
| Rule 3 (doctor/slot) | ✓ | ✓ |
| Rule 4 (room/slot) | ✓ | ✓ |
| Extra storage | — | ~1.5 GB |

---

## Common mistakes

**1. Assuming 3NF implies BCNF.**
- *Symptom:* insert and delete anomalies in a table that "passed normalisation."
- *Engine-level why:* 3NF's escape clause permits a non-key to determine a prime attribute.
- *Fix:* run the BCNF check separately — it's simpler than 3NF's (no need to identify prime attributes).

**2. Decomposing to BCNF without checking what dependencies you lose.**
- *Symptom:* a business rule that was enforced by a `PRIMARY KEY` becomes unenforceable, and violations appear weeks later.
- *Fix:* before decomposing, list every FD and check which ones span the new tables. Then decide deliberately.

**3. Treating "3NF vs BCNF" as a binary.**
- *Symptom:* an argument in a design review with no good answer.
- *Fix:* the hybrid — BCNF decomposition plus a composite-FK-guarded copy — usually gets both properties for one column and one index.

**4. Enforcing a lost dependency with a naive trigger.**
- *Symptom:* the rule is violated under concurrency anyway.
- *Engine-level why:* a trigger that runs `SELECT` and then decides has the same race window as application code (Topic 24).
- *Fix:* a constraint if at all possible; otherwise a trigger with an explicit lock, or `SERIALIZABLE`, and know which you chose.

**5. Chasing BCNF on a table with one candidate key.**
- *Symptom:* wasted analysis.
- *Engine-level why:* a BCNF violation requires **overlapping** candidate keys. One key ⇒ 3NF and BCNF coincide.
- *Fix:* find the candidate keys first. If there's one, you're done after 3NF.

**6. Forgetting that the decomposition depends on which FD you pick.**
- *Symptom:* two engineers produce different (both valid) decompositions and argue.
- *Fix:* it's genuinely non-deterministic. Pick by which resulting tables map better to the domain entities.

---

## Hands-on proof

**PROVE IT #1 — 3NF holds, BCNF fails, anomalies exist.** (Example 1, steps 1–2.)
**PROVE IT #2 — decomposition removes the anomalies.** (Example 1, step 3.)
**PROVE IT #3 — and loses the dependency.** (Example 1, step 4.)
**PROVE IT #4 — the hybrid restores it.** (Example 1, step 5.)

**PROVE IT #5 — find tables with overlapping candidate keys (BCNF candidates).**
```sql
-- tables with a PK plus one or more UNIQUE constraints that SHARE a column
SELECT c.relname,
       string_agg(DISTINCT con.conname || ' (' ||
         (SELECT string_agg(a.attname, ',' ORDER BY x.ord)
            FROM unnest(con.conkey) WITH ORDINALITY x(attnum, ord)
            JOIN pg_attribute a ON a.attrelid=c.oid AND a.attnum=x.attnum) || ')',
         '; ') AS keys
FROM pg_constraint con
JOIN pg_class c ON c.oid = con.conrelid
WHERE con.contype IN ('p','u') AND c.relnamespace='public'::regnamespace
GROUP BY c.oid, c.relname
HAVING count(*) > 1;
-- ★ any table here has multiple candidate keys. If they OVERLAP,
--   check for a BCNF violation.
```

**PROVE IT #6 — detect the violation in data.**
```sql
-- "does a non-key determinant exist?"  e.g. tutor → subject
SELECT count(*) FILTER (WHERE n=1) AS consistent,
       count(*) FILTER (WHERE n>1) AS violated,
       count(*) AS groups
FROM (SELECT tutor, count(DISTINCT subject) n FROM tutoring GROUP BY 1) t;
-- consistent = groups  ⇒ tutor → subject holds
-- and tutor is NOT a superkey  ⇒ ★ BCNF violation
```

**PROVE IT #7 — the composite FK guarantees the copy.**
```sql
INSERT INTO enrolments VALUES ('Meera','Mrs Iyer','Chemistry');
-- ERROR: violates foreign key constraint "fk_tutor_subject"
-- ★ the denormalised copy is PROVABLY consistent with its source
```

---

## The design decision framework

```
THE CHECK — simpler than 3NF's:
  ① find all candidate keys
  ② for every non-trivial X → A:  is X⁺ = all attributes?
       YES → fine.   NO → ★ BCNF violation.
  ⇒ no need to identify prime attributes.

★ FIRST, CHECK WHETHER IT CAN EVEN HAPPEN:
  A BCNF violation requires TWO OR MORE OVERLAPPING candidate keys.
  ⇒ if the table has ONE candidate key, 3NF ⇒ BCNF. Stop.
  ⇒ this is why BCNF violations are rare.

IF YOU FIND ONE, DECIDE DELIBERATELY:

 STAY IN 3NF WHEN:
   ✓ the lost dependency encodes a rule you MUST enforce atomically
   ✓ the redundancy is small (few rows per determinant value)
   ✓ the insert/delete anomaly is not a real business problem
     ("we never need a tutor with no students")

 DECOMPOSE TO BCNF WHEN:
   ✓ the insert/delete anomaly IS a real business problem
   ✓ the redundancy is large (many rows per determinant value)
   ✓ the lost dependency can be enforced another way

 ★ USUALLY: DO BOTH — BCNF + a controlled denormalisation:
     ① decompose to BCNF
     ② put the determined attribute BACK on the child table
     ③ UNIQUE on the parent over (determinant, determined)
     ④ COMPOSITE FOREIGN KEY from the child, so the copy cannot lie
     ⑤ UNIQUE on the child to restore the lost rule
   ⇒ anomaly-freedom AND enforceability.
     Cost: one column + one index. Almost always worth it.

 IF YOU CANNOT ENFORCE IT WITH A CONSTRAINT, you owe:
   • a trigger WITH an explicit lock, or SERIALIZABLE, or
   • an application check PLUS a scheduled reconciliation query
     ⇒ and the reconciliation job is NOT optional (Topic 23's rule)

THE SIGNAL TO LOOK FOR:
  Run PROVE IT #5 — tables with a PK plus overlapping UNIQUE
  constraints. For each, list the FDs and test each determinant's
  closure. Then, for any violation, ask:
    "if I decompose, which FD spans the new tables?"
    "what happens in the real world if that rule is broken?"
  ⇒ if the answer is "two patients in one room," you enforce it —
    with the hybrid, not by staying in 3NF.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build the `tutoring` table. Show formally that it is in 3NF and not in BCNF, naming the escape clause that saves it. Produce all three anomalies. Decompose to BCNF, then demonstrate the lost dependency by inserting two Physics tutors for one student. Finally, build the hybrid and show that both rules are enforced.

### Exercise 2 — medium (apply it)
Given `R(city, street, pincode)` with `F = { (city, street) → pincode, pincode → city }`:
(a) find all candidate keys,
(b) show that R is in 3NF but not BCNF,
(c) decompose to BCNF and identify which FD is lost,
(d) construct concrete data where the lost FD is violated in the decomposition but not in the original,
(e) design the hybrid that keeps both, and state its cost,
(f) argue whether, for a postal-address system specifically, you would ship 3NF, BCNF, or the hybrid.

### Exercise 3 — hard (production simulation)
A clinic's `appointments` table has PK `(patient_id, slot_start)` and enforces four rules via three overlapping candidate keys. A new requirement adds `(doctor_id, session_id) → room_id`. An audit finds 2,104 sessions where a doctor is recorded in two rooms.

(a) Show that the table was in BCNF before the new rule and is not after.
(b) Write the detection query for the 2,104 sessions, and explain what report has been wrong.
(c) Perform the BCNF decomposition and identify **every** dependency lost.
(d) Two of the lost dependencies are patient-safety rules. Name them and describe the real-world failure.
(e) Design the hybrid that restores all four rules. Show every constraint and say which rule it encodes.
(f) Compute the storage cost of the hybrid on 40M appointments.
(g) Rewrite the constraints for the case where appointments are time *ranges* rather than fixed slots.
(h) Give the migration plan (Topic 28), including how you resolve the 2,104 conflicting sessions — which is not a bulk update.

---

## Mental model checkpoint

1. State the BCNF condition in one sentence. How does it differ from 3NF's?
2. What three structural conditions must hold for a BCNF violation to be possible? Why does that make them rare?
3. Give the BCNF decomposition algorithm. Which property does it always guarantee, and which does it not?
4. Explain concretely what "losing a dependency" costs — not in theory, but in what the database can no longer prevent.
5. Why does 3NF have the "or A is prime" escape clause? What does it buy?
6. Describe the hybrid (BCNF + controlled denormalisation). Which two constraints make the copy trustworthy?
7. You find a BCNF violation. What is the *first* question you ask before decomposing?

---

## Quick reference card

**BCNF:** for every non-trivial `X → A`, **X is a superkey.**

| | 3NF | BCNF |
|---|---|---|
| Condition | X superkey **or** A prime | X superkey |
| Always achievable losslessly | ✓ | ✓ |
| Always dependency-preserving | ✓ | **✗** |
| Anomalies possible | yes (rarely) | no |

**A violation requires:** ≥2 candidate keys · overlapping · a non-key determinant. **One candidate key ⇒ 3NF = BCNF.**

**Decomposition:** `R1 = X⁺`, `R2 = X ∪ (R − X⁺)`. Always lossless; not always dependency-preserving; result depends on which FD you pick.

**★ The hybrid — usually the right production answer**

```sql
-- ① decompose to BCNF
CREATE TABLE parent (det, determined, PRIMARY KEY (det),
                     UNIQUE (det, determined));          -- ③ FK target
-- ② put the copy back on the child
CREATE TABLE child (…, det, determined,
  UNIQUE (other, determined),                            -- ⑤ the lost rule
  FOREIGN KEY (det, determined)                          -- ④ the copy can't lie
    REFERENCES parent (det, determined) ON UPDATE CASCADE);
```

**The decision:** stay in 3NF if the lost rule must be atomic and the redundancy is small. Decompose if the anomaly is real. **Usually: do both.**

---

## When would I use this at work?

1. **A table with a PK *and* a unique constraint that share a column.** That's the signature. Check whether a non-key determines something — if so, you have a BCNF question rather than a 3NF one.

2. **When decomposition "obviously" improves a schema and someone hesitates.** Being able to name the lost dependency — "we can no longer prevent two patients in one room" — turns a taste argument into a specific, checkable risk.

3. **Designing booking, scheduling, or allocation systems.** These naturally produce overlapping candidate keys, so BCNF questions arise genuinely — and the hybrid with a composite FK is the pattern that resolves them.

---

## Connected topics

**Understand before this:** 30 (closure, superkeys, prime attributes, dependency preservation), 33 (3NF and its escape clause), 24 (`UNIQUE`, composite FKs, `EXCLUDE`).

**This unlocks:**
- **35–36** — 4NF and 5NF, which generalise beyond functional dependencies
- **37** — the worked example, taken through BCNF
- **38** — when to stop, where this trade-off is decided in practice
- **Case study 02** — exclusion constraints on overlapping bookings, the temporal version of this shape
