# 121 — Design Splitwise / Expense Sharing
## Category: LLD Case Study

---

## What is this?

Splitwise is an app that tracks shared expenses among a group of people — a road trip, a shared apartment, a dinner — and figures out **who owes whom, and how much**. You add an expense ("I paid $120 for dinner, split it 4 ways"), and the app keeps a running ledger of everyone's balances so nobody has to do the mental arithmetic.

Think of it as a **shared tab at a bar**. Everyone throws money onto the tab at different times, orders different things, and at the end the bartender (the app) tells each person the single number they owe or are owed — after netting everything out.

---

## Why does it matter?

This is one of the most-asked LLD interview problems, and for good reason: it is small enough to finish in 45 minutes but rich enough to test two independent skills at once.

**Interview angle.** It checks whether you can (1) model real-world objects cleanly (users, groups, expenses, splits), (2) apply the **Strategy pattern** to handle the several ways an expense can be divided without a pile of `if/else`, and (3) implement a genuine **algorithm** — debt simplification — that reduces the number of payments needed to settle up. Most candidates nail the objects but freeze on the algorithm. If you can code the greedy min-transaction settlement, you stand out.

**Real-work angle.** Any product with a ledger — fintech wallets, marketplaces splitting payouts, corporate expense tools, group-buy apps — needs exactly this: accurate balance tracking that must never lose a cent, and a settlement step that minimizes transfers (each transfer can cost a fee, so fewer is literally cheaper). Getting the money math wrong erodes trust instantly.

---

## The core idea — explained simply

### Analogy: the office snack ledger

Imagine an office where people keep buying snacks for the group and scribbling IOUs on a whiteboard:

- Alice bought pizza, everyone owes her a slice's worth.
- Bob bought drinks, everyone owes him.
- Carol covered the cab, split among three riders.

By Friday the whiteboard is a mess of arrows. Nobody wants to make ten separate Venmo payments. What you really want is: **compute each person's single net position** (did they, in total, pay more than their fair share or less?), then arrange the fewest possible payments so everyone lands at zero.

Map the analogy to the technical pieces:

| Whiteboard thing | Technical concept |
|---|---|
| A person | `User` |
| A snack purchase | `Expense` (amount + who paid) |
| "split it 4 ways" / "Bob's share is $30" | `Split` + a `SplitStrategy` |
| The arrows between names | `balanceSheet[a][b]` — how much `a` owes `b` |
| Each person's overall +/- | **net balance** (total paid − total owed) |
| "make the fewest payments" | **debt simplification** (greedy algorithm) |

The two hard parts — flexible splitting and minimal settlement — are exactly the two things this doc makes shine.

---

## Key concepts inside this topic

We follow the standard 7-step LLD method: clarify → nouns → verbs → class diagram → the intellectual cores → full code → patterns & extensions.

### 1. Requirements & clarifying questions

**Functional requirements:**
- Users can be registered; users can be grouped (a trip, a flat).
- A user can **add an expense**: a total amount, who **paid** it, and the set of **participants** it is split among.
- Support four **split types**:
  - `EQUAL` — divide the total evenly.
  - `EXACT` — caller specifies each participant's exact owed amount; they must sum to the total.
  - `PERCENT` — caller specifies each participant's percentage; percentages must sum to 100.
  - `SHARES` — caller gives integer shares (e.g. 2:1:1); amount is split proportionally.
- Track **who owes whom** and show **balances** for a user (or everyone).
- **Simplify debts**: produce the minimum number of transactions that settles everyone up.
- **Settle up**: record that one user paid another, updating balances.

**Non-functional:**
- Money must be exact — never lose or invent a cent (watch floating-point rounding).
- Reads (show balances) are far more frequent than writes (add expense).

**Clarifying questions to ask the interviewer** (saying these out loud earns points):
- *Partial payments?* Can an expense be paid by more than one person? (We'll keep single-payer for the core and note the extension.)
- *Multiple currencies?* Assume one currency for now; discuss conversion as an extension.
- *Settle-up flow?* Does settling record a real payment, or just clear the ledger? (We record it as a payment that zeroes the pair.)
- *Rounding rule?* When $10 splits 3 ways, who eats the extra cent? (We assign leftover cents to the first participants.)

### 2. Identify core objects (the nouns)

| Object | Responsibility |
|---|---|
| `User` | An identity: id, name, email. |
| `Group` | A named collection of users (optional scoping for expenses). |
| `Expense` | One transaction: amount, `paidBy`, description, and a list of `Split`s. |
| `Split` | One participant's obligation: a `user` and the `amount` they owe for this expense. |
| `SplitStrategy` | Abstract algorithm that turns (total, participants, params) → list of `Split`s, **with its own validation**. Subtypes: `EqualSplitStrategy`, `ExactSplitStrategy`, `PercentSplitStrategy`, `SharesSplitStrategy`. |
| `BalanceSheet` | The ledger: `balances[a][b]` = how much `a` owes `b`. Knows how to update, net out, and simplify. |
| `ExpenseManager` | The service/facade the outside world calls: `addExpense`, `getBalances`, `showBalances`, `simplifyDebts`, `settleUp`. |

### 3. Behaviours (the verbs) + who owns them

| Behaviour | Owner | Notes |
|---|---|---|
| `addExpense(paidBy, amount, participants, type, params)` | `ExpenseManager` | Picks the strategy, builds splits, validates, records the expense, updates balances. |
| `calculateSplits(total, participants, params)` | each `SplitStrategy` | The one method that varies. Each subtype validates its own inputs (exact sums to total; percents sum to 100). |
| `updateBalances(expense)` | `BalanceSheet` | For each split, the participant owes the payer; net symmetric pairs. |
| `getBalances(user)` | `ExpenseManager` → `BalanceSheet` | Everyone this user owes / is owed by. |
| `simplifyDebts()` | `BalanceSheet` | Compute net per user, then greedily match biggest debtor to biggest creditor. |
| `settleUp(from, to, amount)` | `ExpenseManager` | Records a payment; reduces `from`→`to` debt. |

The key invariant lives in `BalanceSheet`: we keep `balances[a][b]` as a directed edge and **net** it so that at most one of `a→b` / `b→a` is non-zero.

### 4. Class diagram

```
                       ┌───────────────────────┐
                       │     ExpenseManager     │  (service / facade)
                       │───────────────────────│
                       │ - users: Map           │
                       │ - expenses: Expense[]  │
                       │ - sheet: BalanceSheet  │
                       │───────────────────────│
                       │ + addExpense(...)      │
                       │ + getBalances(user)    │
                       │ + showBalances()       │
                       │ + simplifyDebts()      │
                       │ + settleUp(...)        │
                       └─────────┬──────────────┘
                                 │ owns
                 ┌───────────────┼────────────────────┐
                 ▼               ▼                     ▼
         ┌──────────────┐  ┌────────────┐      ┌────────────────┐
         │ BalanceSheet │  │  Expense   │◇────▶│     Split      │
         │──────────────│  │────────────│ many │────────────────│
         │ balances[a][b│  │ amount     │      │ user, amount   │
         │──────────────│  │ paidBy ────┼──┐   └────────────────┘
         │ update()     │  │ splits[]   │  │
         │ simplify()   │  │ description│  │ references
         └──────────────┘  └─────┬──────┘  │
                                 │ built by │
                                 ▼          ▼
                      ┌───────────────────┐  ┌────────┐
                      │  «abstract»       │  │  User  │
                      │  SplitStrategy    │  │────────│
                      │───────────────────│  │ id,name│
                      │ calculateSplits() │  └────────┘
                      └───────┬───────────┘
             ┌────────────┬───┴──────┬──────────────┐
             ▼            ▼          ▼              ▼
     ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌────────────┐
     │  Equal    │ │  Exact    │ │  Percent  │ │  Shares    │
     │  Strategy │ │  Strategy │ │  Strategy │ │  Strategy  │
     └───────────┘ └───────────┘ └───────────┘ └────────────┘
```

Read the diamonds (`◇`) as composition — an `Expense` **is made of** `Split`s; the `ExpenseManager` **owns** the `BalanceSheet`. The four strategy boxes all satisfy the same `calculateSplits` contract, so `ExpenseManager` never needs to know which one it holds. (Recall [23 — class relationships](./23-class-relationships.md) for how composition/aggregation are drawn.)

### 5. The two intellectual cores

This problem is interesting because it has **two** independent hard parts. Nail both.

#### Core A — split types via the STRATEGY pattern

The way an expense is divided **varies**, but the surrounding flow (validate → build splits → update balances) stays the same. That is the textbook trigger for **Strategy** (recall [42 — Strategy pattern](./42-pattern-strategy.md)): define a family of interchangeable algorithms behind one interface, and let the caller pick at runtime.

```js
class SplitStrategy {
  // returns [{ user, amount }] and throws on invalid params
  calculateSplits(totalAmount, participants, params) {
    throw new Error("abstract");
  }
}
```

`EqualSplitStrategy` splits evenly; `ExactSplitStrategy` reads exact amounts and asserts they sum to the total; `PercentSplitStrategy` reads percentages and asserts they sum to 100; `SharesSplitStrategy` splits proportionally to integer shares. Each class does its **own validation** — the rule for "valid" is part of the algorithm, so it belongs inside the algorithm.

**Why this matters:** adding a new split type (say, "by adjustment") is a **new class, zero changes to existing code** — that is the Open/Closed Principle in action. The alternative — one giant `if (type === 'EQUAL') ... else if (type === 'EXACT') ...` — grows unboundedly and gets re-tested end-to-end every time you touch it. Strategy makes each rule an isolated, unit-testable unit.

#### Core B — debt simplification (the greedy algorithm)

After many expenses the ledger is tangled. Classic example:

> A owes B $10, and B owes C $10.

That is **two** transactions. But B is a pass-through — money in equals money out. Simplify to:

> A owes C $10.

**One** transaction. B is settled without touching a dollar. The goal: **minimize the number of settlement transactions.**

The algorithm:

1. **Compute each person's NET balance** = (total they paid) − (total they owe). Sum over all edges.
   - Negative net → they **owe** money (a debtor).
   - Positive net → they are **owed** money (a creditor).
   - The nets always sum to zero (money is conserved).
2. Put debtors in one bucket, creditors in another.
3. **Greedily** take the biggest debtor and the biggest creditor. Settle `min(|debtor|, creditor)` — one transaction: debtor pays creditor that amount.
4. Whoever isn't fully settled stays in the pool with the reduced amount; repeat until everyone is zero.

Each transaction zeroes out **at least one** person, so with `n` people who have non-zero balances you need at most `n − 1` transactions. Greedily matching the largest amounts keeps the count near the minimum for typical inputs.

**Worked example** — four friends, nets already computed:

| Person | Net balance |
|---|---|
| Alice | −40 (owes 40) |
| Bob | −20 (owes 20) |
| Carol | +50 (owed 50) |
| Dave | +10 (owed 10) |

Debtors: Alice(40), Bob(20). Creditors: Carol(50), Dave(10).

- Biggest debtor Alice(40) ↔ biggest creditor Carol(50). Settle min(40,50)=40. **Alice → Carol $40.** Alice done. Carol now owed 10.
- Biggest debtor Bob(20) ↔ biggest creditor Carol(10). Settle min(20,10)=10. **Bob → Carol $10.** Carol done. Bob now owes 10.
- Biggest debtor Bob(10) ↔ biggest creditor Dave(10). Settle 10. **Bob → Dave $10.** Both done.

Result: **3 transactions** to settle 4 people. Without simplification, the raw per-expense ledger could easily have had 6–8 directed debts. The transaction count drops, and every payment is a clean netted number. This is the part that makes interviewers lean forward.

### 6. (Covered in the full implementation below)

### 7. (Design patterns and extensions are in their own sections below)

---

## Visual / Diagram description

Sequence of `addExpense` for a PERCENT split, then `simplifyDebts`:

```
Caller        ExpenseManager      PercentSplitStrategy    BalanceSheet
  │  addExpense(Bob, 100,        │                        │
  │   [A,B,C], PERCENT,          │                        │
  │   {percents:{A:50,B:30,C:20}})                        │
  │─────────────▶│               │                        │
  │              │ calculateSplits(100,[A,B,C],params)     │
  │              │──────────────▶│                        │
  │              │               │ validate Σ%==100 ✓     │
  │              │  [{A:50},{B:30},{C:20}]                 │
  │              │◀──────────────│                        │
  │              │ for each split: update(payer=Bob)       │
  │              │───────────────────────────────────────▶│
  │              │               │       balances[A][Bob]+=50
  │              │               │       balances[C][Bob]+=20
  │              │               │       (B is payer, skips self)
  │◀─────────────│  ok           │                        │
  │                                                        │
  │  simplifyDebts()                                       │
  │──────────────────────────────────────────────────────▶│
  │              net per user → greedy match → txn list    │
  │◀──────────────────────────────────────────────────────│
  │   [ {from:A, to:Bob, amount:50}, {from:C,to:Bob,20} ] │
```

The left-to-right flow shows the manager delegating the "how to split" decision entirely to the strategy, then folding the result into the `BalanceSheet`. `simplifyDebts` is a pure read over the sheet that returns a fresh, minimal transaction list without mutating balances.

---

## Real world examples

### Splitwise (the product itself)

Splitwise popularized the "simplify debts" feature as a headline capability: after a trip with many cross-payments, one tap collapses the web of IOUs into the fewest transfers. Conceptually this is the same net-balance + greedy matching described here (the exact production algorithm is proprietary, so treat this as **representative**). Their core data model — expenses with per-user shares and a running balance — mirrors our `Expense`/`Split`/`BalanceSheet`.

### Venmo / group payments

Venmo's "split" flows let one payer charge multiple friends. The per-request share calculation is a split strategy (equal or custom amounts), and the outstanding-request list is a lightweight balance sheet between two parties. **Representative**: the netting is simpler than Splitwise because payments settle immediately.

### Corporate expense tools (Expensify, Ramp)

When one employee fronts a team dinner and it must be allocated across cost centers or teammates, the allocation step is exactly a split strategy (by exact amount, by headcount = equal, or by percentage of budget). The reconciliation report that shows net owed per person/department is a balance sheet. **Representative** at the modeling level.

---

## Trade-offs

**Storing raw edges (`balances[a][b]`) vs. only net-per-user:**

| Approach | Pros | Cons |
|---|---|---|
| Directed edges `balances[a][b]` | Shows the *exact* history of who-owes-whom; matches expenses 1:1 | More entries; needs simplify step to settle cleanly |
| Only net per user | Compact; instant "your total" | Loses pairwise detail ("but I never borrowed from Dave!") |

Keep the edges for transparency; compute nets on demand for settlement.

**Eager vs. lazy simplification:**

| Approach | Pros | Cons |
|---|---|---|
| Simplify on every add | Ledger always minimal | Recomputes constantly; hides expense-level detail |
| Simplify on demand (our choice) | Cheap writes; detail preserved | Reader must call `simplifyDebts()` to see the minimal set |

**Money representation:**

| Approach | Pros | Cons |
|---|---|---|
| Floating-point dollars | Simple | Rounding errors; `0.1 + 0.2 !== 0.3` |
| Integer cents | Exact, no drift | Must convert at the edges |

**The sweet spot:** store directed edges, keep amounts as integer cents (or round consistently to 2 dp), simplify lazily when the user asks to settle up. This gives exact money, full history, and minimal transfers only when it matters.

---

## Common interview questions on this topic

### Q1: "How do you handle a split that doesn't divide evenly, like $10 among 3 people?"
**Hint:** $10 / 3 = $3.333... You can't pay a third of a cent. Compute the base share (round down to cents: $3.33), sum the three ($9.99), and assign the leftover ($0.01) to the first participant(s). The splits must sum **exactly** to the total — never let rounding invent or drop money. Show this explicitly in `EqualSplitStrategy`.

### Q2: "Why Strategy here instead of a switch statement?"
**Hint:** The split *rule* is the thing that varies; everything around it is constant. Strategy isolates each rule in its own class with its own validation, so a new type is a new class (Open/Closed). A switch grows without bound and forces you to re-test all branches on every change. Name the pattern and cite OCP.

### Q3: "Walk me through minimizing settlement transactions."
**Hint:** Compute each person's net (paid − owed). Nets sum to zero. Split into debtors (negative) and creditors (positive). Greedily match the largest debtor to the largest creditor, settle the min of the two, one transaction, repeat. At most `n−1` transactions for `n` non-zero people. Walk the 4-person example.

### Q4: "Is greedy actually optimal?"
**Hint:** Be honest. The general "minimum number of transactions" problem is equivalent to a subset-sum/partition problem and is NP-hard for the true optimum. Greedy gives at most `n−1` transactions and is optimal in practice for the vast majority of real inputs; the exact minimum would need exponential search. Interviewers reward candidates who state this nuance rather than claiming greedy is provably optimal.

### Q5: "How would you support an expense paid by multiple people?"
**Hint:** Generalize `paidBy` from a single user to a list of `{user, amountPaid}`. The split logic (who *owes* what) is unchanged; only `updateBalances` changes — credit each payer for what they put in. The net-balance math already handles it because net = total paid − total owed.

---

## Practice exercise

**Build a two-strategy Splitwise core and prove simplification (~30 min).**

1. Implement `EqualSplitStrategy` and `ExactSplitStrategy` (with validation).
2. Implement a `BalanceSheet` with `updateBalance(from, to, amount)` and `getNetBalances()`.
3. Add three expenses among users A, B, C, D using both split types.
4. Print the raw directed balances, then implement `simplifyDebts()` and print the minimal transaction list.
5. **Assert** two things: (a) the sum of all splits for each expense equals the expense total (no lost cents), and (b) the number of simplified transactions is ≤ (number of people with non-zero net − 1).

Produce a single `.js` file that runs with `node` and prints both the raw and simplified ledgers. Success = the simplified list has fewer entries than the raw list for at least one scenario.

---

## Quick reference cheat sheet

- **`SplitStrategy`** — abstract; one method `calculateSplits(total, participants, params)`; each subtype validates its own params.
- **EQUAL** — even split; assign leftover cents to the first participants so splits sum exactly to the total.
- **EXACT** — caller gives amounts; **must sum to the total** or throw.
- **PERCENT** — caller gives percentages; **must sum to 100** or throw.
- **SHARES** — split proportionally to integer shares (2:1:1).
- **`balances[a][b]`** — directed edge: how much `a` owes `b`; net symmetric pairs so only one direction is non-zero.
- **Net balance** — paid − owed; negative = debtor, positive = creditor; all nets sum to zero.
- **Debt simplification** — greedy: match biggest debtor to biggest creditor, settle `min`, repeat.
- **Bound** — at most `n−1` transactions for `n` non-zero people; true optimum is NP-hard (say so).
- **Strategy = OCP** — new split type is a new class, no edits to existing code.
- **Money** — use integer cents or round consistently; never let floating point lose a cent.
- **`ExpenseManager`** — the facade: `addExpense`, `getBalances`, `showBalances`, `simplifyDebts`, `settleUp`.

---

## Full JavaScript implementation

Save as `splitwise.js` and run `node splitwise.js`. It is complete and runnable.

```js
"use strict";

// ---- Enum via Object.freeze -------------------------------------------------
const SplitType = Object.freeze({
  EQUAL: "EQUAL",
  EXACT: "EXACT",
  PERCENT: "PERCENT",
  SHARES: "SHARES",
});

// round to cents to avoid floating-point drift (0.1 + 0.2 !== 0.3)
const round2 = (n) => Math.round(n * 100) / 100;

// ---- Domain entities --------------------------------------------------------
class User {
  constructor(id, name, email = "") {
    this.id = id;
    this.name = name;
    this.email = email;
  }
  toString() {
    return this.name;
  }
}

class Group {
  constructor(id, name) {
    this.id = id;
    this.name = name;
    this.members = new Map(); // userId -> User
  }
  addMember(user) {
    this.members.set(user.id, user);
    return this;
  }
}

// one participant's obligation for an expense
class Split {
  constructor(user, amount) {
    this.user = user;
    this.amount = round2(amount);
  }
}

// ---- STRATEGY hierarchy (Core A) -------------------------------------------
class SplitStrategy {
  // returns Split[]; each subtype validates its own params and throws on error
  calculateSplits(totalAmount, participants, params = {}) {
    throw new Error("calculateSplits() must be implemented by a subclass");
  }
}

class EqualSplitStrategy extends SplitStrategy {
  calculateSplits(totalAmount, participants) {
    const n = participants.length;
    if (n === 0) throw new Error("EQUAL split needs at least one participant");
    // base share rounded down to cents; leftover cents go to the first people
    const base = Math.floor((totalAmount / n) * 100) / 100;
    const splits = participants.map((u) => new Split(u, base));
    let leftover = round2(totalAmount - base * n); // e.g. 0.01 for $10/3
    let i = 0;
    while (leftover > 0.0001) {
      splits[i % n].amount = round2(splits[i % n].amount + 0.01);
      leftover = round2(leftover - 0.01);
      i++;
    }
    return splits;
  }
}

class ExactSplitStrategy extends SplitStrategy {
  // params.amounts = { userId: amountOwed }
  calculateSplits(totalAmount, participants, params) {
    const amounts = params.amounts || {};
    const splits = participants.map((u) => {
      if (amounts[u.id] === undefined)
        throw new Error(`EXACT split missing amount for ${u.name}`);
      return new Split(u, amounts[u.id]);
    });
    const sum = round2(splits.reduce((s, sp) => s + sp.amount, 0));
    if (sum !== round2(totalAmount))
      throw new Error(
        `EXACT splits (${sum}) must sum to total (${round2(totalAmount)})`
      );
    return splits;
  }
}

class PercentSplitStrategy extends SplitStrategy {
  // params.percents = { userId: percentage }
  calculateSplits(totalAmount, participants, params) {
    const percents = params.percents || {};
    const totalPct = round2(
      participants.reduce((s, u) => s + (percents[u.id] || 0), 0)
    );
    if (totalPct !== 100)
      throw new Error(`PERCENT splits must sum to 100 (got ${totalPct})`);
    return participants.map(
      (u) => new Split(u, (totalAmount * percents[u.id]) / 100)
    );
  }
}

class SharesSplitStrategy extends SplitStrategy {
  // params.shares = { userId: integerShares }, e.g. {A:2, B:1, C:1}
  calculateSplits(totalAmount, participants, params) {
    const shares = params.shares || {};
    const totalShares = participants.reduce((s, u) => s + (shares[u.id] || 0), 0);
    if (totalShares <= 0) throw new Error("SHARES split needs positive shares");
    return participants.map(
      (u) => new Split(u, (totalAmount * (shares[u.id] || 0)) / totalShares)
    );
  }
}

// registry so ExpenseManager just looks up a strategy by type
const STRATEGIES = Object.freeze({
  [SplitType.EQUAL]: new EqualSplitStrategy(),
  [SplitType.EXACT]: new ExactSplitStrategy(),
  [SplitType.PERCENT]: new PercentSplitStrategy(),
  [SplitType.SHARES]: new SharesSplitStrategy(),
});

// ---- Expense ----------------------------------------------------------------
class Expense {
  constructor(id, description, amount, paidBy, splits) {
    this.id = id;
    this.description = description;
    this.amount = round2(amount);
    this.paidBy = paidBy; // User
    this.splits = splits; // Split[]
  }
}

// ---- BalanceSheet: the ledger (Core B lives here) ---------------------------
class BalanceSheet {
  constructor() {
    // balances[a][b] = how much user a owes user b
    this.balances = new Map();
  }

  _edge(a, b) {
    if (!this.balances.has(a)) this.balances.set(a, new Map());
    const row = this.balances.get(a);
    if (!row.has(b)) row.set(b, 0);
    return row;
  }

  // debtor owes creditor `amount` more; net against the reverse edge
  update(debtorId, creditorId, amount) {
    if (debtorId === creditorId || amount === 0) return;
    // reduce any existing reverse debt first
    const reverse = this._edge(creditorId, debtorId).get(debtorId);
    const forward = this._edge(debtorId, creditorId).get(creditorId);
    let net = round2(forward + amount - reverse);
    // clear both directions, then set the surviving one
    this.balances.get(debtorId).set(creditorId, 0);
    this.balances.get(creditorId).set(debtorId, 0);
    if (net > 0) this.balances.get(debtorId).set(creditorId, net);
    else if (net < 0)
      this.balances.get(creditorId).set(debtorId, round2(-net));
  }

  // apply one expense: every participant owes the payer their split
  applyExpense(expense) {
    const payer = expense.paidBy.id;
    for (const split of expense.splits) {
      if (split.user.id === payer) continue; // payer doesn't owe themselves
      this.update(split.user.id, payer, split.amount);
    }
  }

  // all raw directed debts as {from, to, amount}
  rawTransactions() {
    const out = [];
    for (const [a, row] of this.balances) {
      for (const [b, amt] of row) {
        if (amt > 0) out.push({ from: a, to: b, amount: round2(amt) });
      }
    }
    return out;
  }

  // net per user = (owed to them) - (they owe). +ve = creditor, -ve = debtor
  netBalances() {
    const net = new Map();
    const bump = (id, delta) => net.set(id, round2((net.get(id) || 0) + delta));
    for (const [a, row] of this.balances) {
      for (const [b, amt] of row) {
        if (amt > 0) {
          bump(a, -amt); // a owes -> negative
          bump(b, amt); //  b is owed -> positive
        }
      }
    }
    return net;
  }

  // GREEDY debt simplification -> minimal-ish list of {from, to, amount}
  simplify() {
    const net = this.netBalances();
    const debtors = []; // {id, amount>0}
    const creditors = [];
    for (const [id, bal] of net) {
      if (bal < -0.0001) debtors.push({ id, amount: round2(-bal) });
      else if (bal > 0.0001) creditors.push({ id, amount: round2(bal) });
    }
    const txns = [];
    while (debtors.length && creditors.length) {
      // biggest debtor and biggest creditor
      debtors.sort((x, y) => y.amount - x.amount);
      creditors.sort((x, y) => y.amount - x.amount);
      const d = debtors[0];
      const c = creditors[0];
      const pay = round2(Math.min(d.amount, c.amount));
      txns.push({ from: d.id, to: c.id, amount: pay });
      d.amount = round2(d.amount - pay);
      c.amount = round2(c.amount - pay);
      if (d.amount <= 0.0001) debtors.shift();
      if (c.amount <= 0.0001) creditors.shift();
    }
    return txns;
  }
}

// ---- ExpenseManager: the facade ---------------------------------------------
class ExpenseManager {
  constructor() {
    this.users = new Map();
    this.expenses = [];
    this.sheet = new BalanceSheet();
    this._seq = 0;
  }

  addUser(user) {
    this.users.set(user.id, user);
    return user;
  }

  // participants: User[]; type: SplitType; params: strategy-specific
  addExpense(description, paidBy, amount, participants, type, params = {}) {
    const strategy = STRATEGIES[type];
    if (!strategy) throw new Error(`Unknown split type: ${type}`);
    const splits = strategy.calculateSplits(amount, participants, params);
    const expense = new Expense(++this._seq, description, amount, paidBy, splits);
    this.expenses.push(expense);
    this.sheet.applyExpense(expense);
    return expense;
  }

  getBalances(user) {
    const result = [];
    const id = user.id;
    // people this user owes
    const owes = this.sheet.balances.get(id);
    if (owes) for (const [to, amt] of owes)
      if (amt > 0) result.push(`${user.name} owes ${this.users.get(to).name}: $${amt}`);
    // people who owe this user
    for (const [a, row] of this.sheet.balances)
      if (row.get(id) > 0)
        result.push(`${this.users.get(a).name} owes ${user.name}: $${row.get(id)}`);
    return result;
  }

  showBalances() {
    const raw = this.sheet.rawTransactions();
    if (raw.length === 0) return ["(all settled)"];
    return raw.map(
      (t) => `${this.users.get(t.from).name} owes ${this.users.get(t.to).name}: $${t.amount}`
    );
  }

  simplifyDebts() {
    return this.sheet
      .simplify()
      .map((t) => ({
        from: this.users.get(t.from).name,
        to: this.users.get(t.to).name,
        amount: t.amount,
      }));
  }

  settleUp(fromUser, toUser, amount) {
    // fromUser pays toUser -> reduces what fromUser owes toUser
    this.sheet.update(toUser.id, fromUser.id, amount);
    this.expenses.push(
      new Expense(++this._seq, `settle: ${fromUser.name}->${toUser.name}`,
        amount, fromUser, [new Split(toUser, amount)])
    );
  }
}

// ---- DEMO -------------------------------------------------------------------
function main() {
  const mgr = new ExpenseManager();
  const alice = mgr.addUser(new User("A", "Alice"));
  const bob = mgr.addUser(new User("B", "Bob"));
  const carol = mgr.addUser(new User("C", "Carol"));
  const dave = mgr.addUser(new User("D", "Dave"));
  const all = [alice, bob, carol, dave];

  console.log("=== Adding expenses ===");

  // 1) EQUAL: Alice pays $120 dinner, split evenly among all 4 ($30 each)
  mgr.addExpense("Dinner", alice, 120, all, SplitType.EQUAL);
  console.log("Alice paid $120 dinner, split equally (4 x $30)");

  // 2) EXACT: Bob pays $50 cab; exact amounts owed
  mgr.addExpense("Cab", bob, 50, [bob, carol, dave], SplitType.EXACT, {
    amounts: { B: 10, C: 25, D: 15 },
  });
  console.log("Bob paid $50 cab; exact: Bob $10, Carol $25, Dave $15");

  // 3) PERCENT: Carol pays $200 hotel; percentages must sum to 100
  mgr.addExpense("Hotel", carol, 200, all, SplitType.PERCENT, {
    percents: { A: 25, B: 25, C: 25, D: 25 },
  });
  console.log("Carol paid $200 hotel; 25% each");

  // 4) SHARES: Dave pays $60 groceries; shares 2:1:1:2
  mgr.addExpense("Groceries", dave, 60, all, SplitType.SHARES, {
    shares: { A: 2, B: 1, C: 1, D: 2 },
  });
  console.log("Dave paid $60 groceries; shares A:2 B:1 C:1 D:2");

  console.log("\n=== Raw balances (directed, netted per pair) ===");
  const raw = mgr.showBalances();
  raw.forEach((line) => console.log("  " + line));
  console.log(`  -> ${raw.length} raw debt(s)`);

  console.log("\n=== Simplified debts (greedy, minimal transactions) ===");
  const simplified = mgr.simplifyDebts();
  simplified.forEach((t) => console.log(`  ${t.from} -> ${t.to}: $${t.amount}`));
  console.log(`  -> ${simplified.length} transaction(s) to settle everyone`);

  console.log("\n=== Trying an invalid EXACT split (should throw) ===");
  try {
    mgr.addExpense("Bad", alice, 100, all, SplitType.EXACT, {
      amounts: { A: 10, B: 10, C: 10, D: 10 }, // sums to 40, not 100
    });
  } catch (e) {
    console.log("  Rejected:", e.message);
  }

  console.log("\n=== Settle up: Alice pays Carol $30 ===");
  mgr.settleUp(alice, carol, 30);
  mgr.simplifyDebts().forEach((t) =>
    console.log(`  ${t.from} -> ${t.to}: $${t.amount}`)
  );
}

main();

module.exports = {
  SplitType, User, Group, Split, Expense, BalanceSheet, ExpenseManager,
  EqualSplitStrategy, ExactSplitStrategy, PercentSplitStrategy, SharesSplitStrategy,
};
```

Running it prints the four expenses being added, the raw netted ledger, the **smaller** simplified transaction set, a rejected invalid EXACT split, and the ledger after a settle-up — proving both the Strategy-based splits and the greedy simplification actually run.

---

## Design patterns used and WHY

| Pattern / technique | Where | Why it's the right call |
|---|---|---|
| **Strategy** (GoF) | `SplitStrategy` + the four subclasses | The split rule is what varies; the flow around it is constant. Each rule is an isolated, self-validating class. Adding a new type = new class, **no edits** to `ExpenseManager` (Open/Closed). This is the star of the design. |
| **Greedy algorithm** (not GoF, the algorithmic heart) | `BalanceSheet.simplify()` | Net each person, match biggest debtor to biggest creditor, settle the min, repeat. Yields ≤ `n−1` transactions and is near-optimal in practice. This is what elevates the problem above a plain CRUD model. |
| **Facade** | `ExpenseManager` | Hides strategies, the sheet, and the seq counter behind a small, intention-revealing API (`addExpense`, `simplifyDebts`, `settleUp`). |
| **Registry / lookup table** | `STRATEGIES` map | Replaces a `switch (type)` with an `O(1)` lookup; adding a type registers one entry. |

Name **Strategy** and **greedy simplification** explicitly in the interview — those two are the whole point.

---

## Extensions the interviewer will ask for

**"Add a new split type — by adjustment (+/−)."** Everyone splits equally, then per-person adjustments shift the shares (e.g. the person who ate dessert pays +$5). This is **trivial with Strategy**: write one new class, register it, touch nothing else.

```js
class AdjustmentSplitStrategy extends SplitStrategy {
  // params.adjustments = { userId: +/- delta }; deltas must sum to 0
  calculateSplits(totalAmount, participants, params) {
    const adj = params.adjustments || {};
    const totalAdj = round2(participants.reduce((s, u) => s + (adj[u.id] || 0), 0));
    if (totalAdj !== 0) throw new Error("adjustments must sum to 0");
    const base = round2((totalAmount - 0) / participants.length);
    return participants.map((u) => new Split(u, base + (adj[u.id] || 0)));
  }
}
// STRATEGIES[SplitType.ADJUSTMENT] = new AdjustmentSplitStrategy();
```

**"Multi-currency."** Give `Expense` a `currency`, and convert to a base currency at add-time using a rate table before touching the sheet. The sheet stays single-currency, so nets and simplification are unchanged. Store the original amount + rate for the receipt.

**"Partial settlements."** Already supported: `settleUp(from, to, amount)` reduces a specific edge by any amount; the remaining debt survives and simplifies normally next time.

**"Expense comments/attachments."** Add `comments: []` and `receiptUrl` fields to `Expense`. No behavioural change — the ledger doesn't care. This is why keeping expense data separate from the balance sheet pays off.

**"Simplify within a group only."** Scope `simplify()` to edges where both users belong to a `Group`. Build a filtered `BalanceSheet` from that group's expenses, then run the same greedy algorithm — the algorithm is agnostic to which subset of users it operates on.

Each extension slots in without rewriting the core: split rules extend via Strategy, the ledger and algorithm stay stable, and metadata rides on `Expense`. That absorb-without-rewrite property is exactly what the interviewer is probing for.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [111 — LLD approach framework](./111-lld-approach-framework.md) | The 7-step method (clarify → nouns → verbs → diagram → code → patterns → extend) we applied here. |
| **Next** | [126 — LLD food delivery](./126-lld-food-delivery.md) | Another service-oriented LLD combining strategies (pricing, matching) with real coordination logic. |
| **Related** | [42 — Strategy pattern](./42-pattern-strategy.md) | The pattern behind the interchangeable split types — the star of this design. |
| **Related** | [23 — class relationships](./23-class-relationships.md) | Composition/aggregation as drawn in the `Expense ◇— Split` and `Manager ◇— BalanceSheet` diagram. |
| **Related** | [112 — LLD parking lot](./112-lld-parking-lot.md) | A sibling LLD case study that also leans on Strategy (fee calculation) and clean object modeling. |
