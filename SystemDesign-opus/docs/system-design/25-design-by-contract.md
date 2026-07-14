# 25 — Design by Contract — Preconditions, Postconditions, Invariants
## Category: LLD Fundamentals

---

## What is this?

**Design by Contract (DbC)** says that every method is a legally binding agreement between two parties. The **caller** promises to satisfy the **preconditions** ("I will only pass you a positive amount"). The **method** promises to deliver the **postconditions** ("if you do, I guarantee your balance drops by exactly that much"). And the **class** promises to always maintain its **invariants** ("my balance is never negative — before any call, after any call, always").

It's a bank loan agreement. *You* promise to have a credit score above 700 and an income above X (preconditions on you). *The bank* promises to give you the money at 8% APR (postcondition on them). And the bank promises to remain solvent throughout (its invariant). If you lie about your income, the deal is off — and crucially, **it's your fault, not theirs**.

That last sentence is the entire point of DbC: it makes **blame** explicit, and blame is what tells you where to put the check.

---

## Why does it matter?

Without contracts, every function is written defensively against every other function, and nobody knows who is responsible for what. The symptoms:

- **The same validation, seven times.** `if (!amount || amount <= 0) return null;` appears at every layer, because nobody trusts anyone. Now the rule lives in seven places and you fix a bug in six of them.
- **Errors surface far from their cause.** Someone passes `amount = NaN`. Nothing throws. `balance` silently becomes `NaN`. Six hours later, a nightly report crashes with a message that has nothing to do with the actual bug. You spend a day bisecting.
- **Corrupt objects.** A `BankAccount` ends up with `balance = -400`. Nothing in the code stopped it, so nothing in the code tells you *when* it happened. You're now archaeologising the audit log.
- **Nobody knows what a method actually promises.** Does `withdraw()` return the new balance? Throw? Return `false`? Silently clamp to zero? Everyone guesses, and each caller guesses differently.

**In interviews:** DbC is how you talk about robustness at a senior level. When you say *"`withdraw` has a precondition that `amount <= balance`; that's the caller's job to check, so inside the method I assert it and fail fast rather than silently clamping"* — you've just demonstrated that you think about responsibility boundaries, not just happy paths. It's also the *only* rigorous way to explain [16 — Liskov Substitution](./16-solid-liskov-substitution.md), which is otherwise the SOLID principle everyone fumbles.

**At work:** it's the thinking behind TypeScript types, JSON-schema request validation, `assert` statements, database `CHECK` constraints, and API documentation. You already use contracts everywhere. DbC just gives you the vocabulary to place them deliberately instead of by accident.

---

## The core idea — explained simply

### The Restaurant Order Analogy

You walk into a restaurant and order a steak, medium-rare.

**Your preconditions (what YOU must guarantee before the kitchen will act):**
- You ordered something that's actually on the menu.
- You can pay.
- You told them the doneness.

If you order "a medium-rare unicorn", the waiter doesn't guess, doesn't quietly bring you chicken, and doesn't try to *fix* your order. He says **"we don't serve that"** and stops. Immediately. Loudly. At the moment you said it — not forty minutes later when a plate of something arrives.

**The kitchen's postconditions (what THEY guarantee, if you held up your end):**
- A steak arrives.
- It is medium-rare.
- It arrives within 25 minutes.

**The restaurant's invariants (always true, before and after every single order, no matter what):**
- The kitchen is food-safe.
- No dish leaves containing a declared allergen.
- The till never goes negative.

The invariant holds *between* orders too. It is not something the kitchen achieves at the end of the night; it is a permanent property of the restaurant. If serving your steak would make the kitchen unsafe, they don't serve it.

**And here's the crucial asymmetry, which is the whole reason DbC exists:**

> Checking your order is the **waiter's** job — at the door. Once the order reaches the kitchen, the chef **assumes it's valid** and just cooks. The chef does not re-verify that unicorn isn't on the menu.

That's boundary validation vs core assertions, and it's section 5.

### Mapping the analogy back

| Restaurant | Code | Who is to blame if violated? |
|---|---|---|
| "You must order from the menu" | **Precondition** — `amount > 0` | **The caller.** It's a *bug in their code*. |
| "We deliver a medium-rare steak" | **Postcondition** — `balance` dropped by exactly `amount` | **The method.** It's a *bug in my code*. |
| "The kitchen is always food-safe" | **Invariant** — `balance >= 0`, always | **The class.** Its state is corrupt. |
| Waiter rejects "unicorn" at the door | **Boundary validation** — HTTP 400 | Neither — expected user error |
| Chef assumes the order is valid | **Core assertion** — `assert(amount > 0)` | If it fires, a *programmer* made a mistake |

The three questions to ask about any method you write:

1. **What must be true for me to even start?** → preconditions.
2. **What do I promise once I'm done?** → postconditions.
3. **What is true about this object at all times, no matter which method ran?** → invariants.

---

## Key concepts inside this topic

### 1. Preconditions — the caller's obligation

A precondition is a condition the **caller must guarantee** before invoking the method. If a precondition is violated, the caller has a bug. The method has **no obligation whatsoever** — it may do anything, and the sanest "anything" is to throw immediately.

```javascript
withdraw(amountCents) {
  // PRECONDITIONS — the caller's job. If these fail, THE CALLER is broken.
  requires(Number.isInteger(amountCents), 'amount must be an integer number of cents');
  requires(amountCents > 0,               'amount must be positive');
  requires(amountCents <= this.#balance,  'insufficient funds');
  // ...
}
```

The critical mental shift: a violated precondition is **not** an "error to handle". It is a **defect to fix**. You don't catch it and retry — you crash, read the stack trace, and fix the caller.

### 2. Postconditions — the method's promise

A postcondition is what the method **guarantees to be true when it returns**, *provided* the preconditions held. It's what the caller gets to rely on without re-checking.

```javascript
withdraw(amountCents) {
  requires(amountCents > 0, 'amount must be positive');
  requires(amountCents <= this.#balance, 'insufficient funds');

  const balanceBefore = this.#balance;      // capture the "old" state

  this.#balance -= amountCents;             // the actual work

  // POSTCONDITIONS — MY promise. If these fail, MY code is broken.
  ensures(this.#balance === balanceBefore - amountCents,
          'balance must decrease by exactly the withdrawn amount');
  ensures(this.#balance >= 0, 'balance must never go negative');

  return this.#balance;
}
```

Notice `ensures(this.#balance === balanceBefore - amountCents)`. Not "decreased". Not "is smaller". **Decreased by exactly `amountCents`.** That precision is what makes a postcondition worth writing — it catches the fee you accidentally applied twice, the rounding bug, the off-by-one.

### 3. Invariants — the class's permanent promise

An invariant is true about the object **at every observable moment**: after the constructor, and before *and* after every public method. It may be temporarily false *inside* a method (that's fine — mid-transfer, money is briefly nowhere), but it must be restored before you hand control back.

For `BankAccount`, the invariants are:
- `balance >= 0` — you cannot be overdrawn.
- `balance` is an integer number of cents — never a float (floats and money are a lifelong enemy: `0.1 + 0.2 !== 0.3`).
- Every transaction in the history sums to the current balance.

```javascript
#checkInvariants() {
  invariant(Number.isInteger(this.#balance), 'balance must be integer cents');
  invariant(this.#balance >= 0,              'balance must never be negative');

  const sum = this.#transactions.reduce((s, t) => s + t.deltaCents, 0);
  invariant(sum === this.#balance, 'transaction history must sum to the balance');
}
```

Then you call `#checkInvariants()` at the end of the constructor and at the exit of every public mutator. That's it. That one habit eliminates an entire category of "how did this object get into this state?" bugs, because the object *cannot* get into that state — it dies at the exact line that would have corrupted it.

### 4. The assertion helpers and the full worked example

First the helpers. Three functions, distinguished by **who is to blame** — that's the only reason they're separate.

```javascript
// contracts.js
// One flag to strip contract checks in production if you ever need the speed.
// Default them ON — in almost every app the cost is nanoseconds and the value is enormous.
const CONTRACTS_ENABLED = process.env.CONTRACTS !== 'off';

class ContractError extends Error {
  constructor(kind, message) {
    super(`[${kind}] ${message}`);
    this.name = 'ContractError';
    this.kind = kind;
  }
}

/** The CALLER broke the deal. Their bug. */
function requires(condition, message) {
  if (CONTRACTS_ENABLED && !condition) throw new ContractError('PRECONDITION', message);
}

/** I broke my own promise. MY bug. */
function ensures(condition, message) {
  if (CONTRACTS_ENABLED && !condition) throw new ContractError('POSTCONDITION', message);
}

/** The object's state is corrupt. A bug SOMEWHERE in this class. */
function invariant(condition, message) {
  if (CONTRACTS_ENABLED && !condition) throw new ContractError('INVARIANT', message);
}

module.exports = { requires, ensures, invariant, ContractError };
```

Three different names for what is fundamentally the same `if (!x) throw`. Why bother? Because when a `ContractError` lands in your logs at 3am, the prefix tells you **where to look before you've read a single line of code**:

- `[PRECONDITION]` → go look at the **caller**.
- `[POSTCONDITION]` → go look at **this method**.
- `[INVARIANT]` → go look at **this whole class**; something corrupted it.

Now the account.

```javascript
// bank-account.js
const { requires, ensures, invariant } = require('./contracts');

class BankAccount {
  #balanceCents;
  #transactions;

  /**
   * INVARIANTS (true before and after EVERY public method, forever):
   *   I1. balanceCents is a non-negative integer
   *   I2. sum of all transaction deltas === balanceCents
   */
  constructor(ownerId, openingBalanceCents = 0) {
    // Preconditions on construction — you can't even be BORN invalid.
    requires(typeof ownerId === 'string' && ownerId.length > 0, 'ownerId required');
    requires(Number.isInteger(openingBalanceCents), 'opening balance must be integer cents');
    requires(openingBalanceCents >= 0, 'opening balance cannot be negative');

    this.ownerId = ownerId;
    this.#balanceCents = openingBalanceCents;
    this.#transactions = openingBalanceCents > 0
      ? [{ type: 'OPEN', deltaCents: openingBalanceCents, at: Date.now() }]
      : [];

    // The constructor's job: establish the invariants. Everything after just preserves them.
    this.#checkInvariants();
  }

  get balanceCents() { return this.#balanceCents; }
  get history() { return [...this.#transactions]; }   // copy — don't leak the internals

  #checkInvariants() {
    invariant(Number.isInteger(this.#balanceCents), 'I1: balance must be integer cents');
    invariant(this.#balanceCents >= 0,              'I1: balance must never be negative');
    const sum = this.#transactions.reduce((s, t) => s + t.deltaCents, 0);
    invariant(sum === this.#balanceCents,           'I2: history must sum to balance');
  }

  /**
   * Withdraw money.
   *
   * PRECONDITIONS  (caller guarantees):
   *   P1. amountCents is an integer
   *   P2. amountCents > 0
   *   P3. amountCents <= balanceCents          ← this is what keeps I1 true
   *
   * POSTCONDITIONS (this method guarantees):
   *   Q1. balanceCents === old(balanceCents) - amountCents   (exactly)
   *   Q2. exactly one new WITHDRAW transaction was appended
   *   Q3. the return value === the new balanceCents
   */
  withdraw(amountCents) {
    requires(Number.isInteger(amountCents),      'P1: amount must be integer cents');
    requires(amountCents > 0,                    'P2: amount must be positive');
    requires(amountCents <= this.#balanceCents,  'P3: insufficient funds');

    // Capture the "old" state so the postconditions can be checked precisely.
    const before = this.#balanceCents;
    const txCountBefore = this.#transactions.length;

    // --- the actual work ---
    this.#balanceCents -= amountCents;
    this.#transactions.push({ type: 'WITHDRAW', deltaCents: -amountCents, at: Date.now() });
    // -----------------------

    ensures(this.#balanceCents === before - amountCents,       'Q1: exact decrease');
    ensures(this.#transactions.length === txCountBefore + 1,   'Q2: one transaction added');
    this.#checkInvariants();                                    // I1, I2 restored

    return this.#balanceCents;                                  // Q3
  }

  /**
   * PRECONDITIONS:  amountCents is an integer > 0
   * POSTCONDITIONS: balanceCents === old(balanceCents) + amountCents
   */
  deposit(amountCents) {
    requires(Number.isInteger(amountCents), 'amount must be integer cents');
    requires(amountCents > 0,               'amount must be positive');

    const before = this.#balanceCents;

    this.#balanceCents += amountCents;
    this.#transactions.push({ type: 'DEPOSIT', deltaCents: amountCents, at: Date.now() });

    ensures(this.#balanceCents === before + amountCents, 'exact increase');
    this.#checkInvariants();

    return this.#balanceCents;
  }
}

module.exports = { BankAccount };
```

Now watch it work:

```javascript
const { BankAccount, ContractError } = require('./bank-account');

const acc = new BankAccount('u1', 10_000);   // ₹100.00 in paise/cents

acc.withdraw(2_500);
console.log(acc.balanceCents);               // 7500 — Q1 verified by the code itself

// --- Precondition violations: FAIL FAST, at the exact call site, with blame attached ---
try { acc.withdraw(-100); }
catch (e) { console.log(e.message); }        // [PRECONDITION] P2: amount must be positive

try { acc.withdraw(999_999); }
catch (e) { console.log(e.message); }        // [PRECONDITION] P3: insufficient funds

try { acc.withdraw(10.5); }
catch (e) { console.log(e.message); }        // [PRECONDITION] P1: amount must be integer cents

// The balance is STILL 7500. Not NaN. Not -992499. Not silently clamped to 0.
// Nothing was corrupted, because nothing ran. That is the entire value proposition.
console.log(acc.balanceCents);               // 7500
```

Compare that to the version without contracts, which is what most code actually looks like:

```javascript
// The "normal" version. Every one of these lines is a latent 3am page.
class SloppyAccount {
  withdraw(amount) {
    this.balance -= amount;      // withdraw(-100)  → balance goes UP. Free money.
                                 // withdraw(NaN)   → balance becomes NaN. Silently.
                                 // withdraw(99999) → balance goes negative. Silently.
    return this.balance;         // The bug surfaces in the nightly report,
  }                              // 14 hours and 400 transactions later.
}
```

The contract version *cannot* reach any of those states. The bug is caught **at the line that caused it**, with a message naming the broken rule and the party at fault.

### 5. Fail fast — and where to validate (boundary vs core)

**Fail fast** means: the instant you detect that reality has diverged from your assumptions, stop. Do not continue. Do not "handle" it, do not substitute a default, do not `return null`.

The reasoning is about the **cost of distance**. Every line of code executed between a bug's *cause* and its *symptom* is a line you'll have to read while debugging.

```
FAIL FAST                    FAIL SLOW (defaults, nulls, silent clamps)
─────────                    ─────────────────────────────────────────
withdraw(NaN)                withdraw(NaN)
    ↓                            ↓
  💥 THROW                    balance = NaN        (no error, feels fine)
                                 ↓
  Stack trace points          saved to DB          (no error, feels fine)
  at the caller.                 ↓
  Fix: 2 minutes.             150 more txns        (all now NaN)
                                 ↓
                              nightly report crashes
                                 ↓
                              💥 "Cannot read properties of undefined"
                                 in a file you've never opened

                              Fix: 6 hours + a data-repair migration
```

**But — and this is the part people get wrong — fail fast does NOT mean "throw at the user."**

There are two completely different kinds of "bad input", and they get two completely different treatments:

| | **Boundary** (edge of the system) | **Core** (domain classes) |
|---|---|---|
| **Where** | HTTP handler, CLI parser, queue consumer, form | `BankAccount`, `PricingPolicy`, domain logic |
| **Input from** | The outside world — untrusted, hostile, dumb | *Your own code*, which you control |
| **Bad input means** | A user made a mistake. **Expected.** | A **programmer** made a mistake. **A bug.** |
| **Technique** | **Validation** — check and respond politely | **Contracts** — assert and die loudly |
| **Response** | `400 Bad Request` + a helpful message | Throw. Crash. Page someone. |
| **Tools** | zod, Joi, JSON Schema, express-validator | `requires` / `ensures` / `invariant` |

Concretely — the same rule, enforced twice, for two different reasons:

```javascript
// ===== BOUNDARY: validate. A human typed this. Be kind, list every problem at once. =====
app.post('/accounts/:id/withdraw', async (req, res) => {
  const { amount } = req.body;

  const problems = [];
  if (typeof amount !== 'number' || !Number.isFinite(amount)) problems.push('amount must be a number');
  if (amount <= 0)                                            problems.push('amount must be positive');
  if (!Number.isInteger(amount * 100))                        problems.push('max 2 decimal places');

  if (problems.length) {
    return res.status(400).json({ errors: problems });   // NOT a crash. Expected outcome.
  }

  const account = await repo.load(req.params.id);
  if (amount * 100 > account.balanceCents) {
    return res.status(422).json({ error: 'Insufficient funds' });  // business rule, told nicely
  }

  // Everything past this line is TRUSTED. The boundary did its job.
  const newBalance = account.withdraw(Math.round(amount * 100));
  await repo.save(account);
  res.json({ balanceCents: newBalance });
});

// ===== CORE: assert. If `requires()` fires HERE, it is not a user's fault. =====
// It means some code path reached withdraw() without passing the boundary — a BUG.
// So we don't return 400. We throw, and someone gets paged, because we want to know.
```

The boundary check and the `requires()` check are the same condition — and that is **not duplication**, because they serve different masters. The boundary check produces a *user-facing error message*. The `requires()` check produces a *stack trace for an engineer*. Delete the boundary check and users get 500s for typos. Delete the `requires()` and a background job, a migration script, or a new caller who forgot the rules will silently corrupt your data.

**Where to put the checks, in one line:** *validate at the edges, assert in the core, and trust everything in between.*

### 6. Contracts vs defensive programming

**Defensive programming** says: *"assume nothing, check everything, never crash."* Contracts say: *"state the assumptions explicitly, check them at the boundary, and crash loudly if the assumption is violated in the core."*

The difference is not paranoia level. It's **who is responsible**, and whether you *hide* violations or *surface* them.

```javascript
// DEFENSIVE (the common instinct). Looks safe. Is a landmine.
withdraw(amount) {
  if (!amount || amount <= 0) return this.balance;         // silently does nothing
  if (amount > this.balance) amount = this.balance;        // silently changes the amount!
  this.balance -= amount;
  return this.balance;
}
// Caller asks to withdraw ₹500 with ₹300 available → gets ₹300 and NO INDICATION.
// The bug that sent the wrong amount is now INVISIBLE. It ships. It runs for a year.
// Worse: this method now has NO postcondition you can state. "Balance decreases by
// amount... or by some other amount... or not at all." That's not a contract. That's a shrug.
```

```javascript
// CONTRACT. Refuses to guess. The bug in the caller is now impossible to ignore.
withdraw(amountCents) {
  requires(amountCents > 0, 'amount must be positive');
  requires(amountCents <= this.#balanceCents, 'insufficient funds');
  this.#balanceCents -= amountCents;
  ensures(...);                                 // a promise you can actually state
  return this.#balanceCents;
}
```

Defensive programming's cardinal sin is the **silent recovery**. It converts a loud bug into a quiet wrong answer, and a quiet wrong answer in a financial system is far more expensive than a crash. A crash costs you one page. A silent clamp costs you a reconciliation project.

**When defensive coding IS right:** at the boundary (untrusted input), and against genuinely unreliable things you don't control — a flaky network, a third-party API, a disk that's full. Retry a failed HTTP call, absolutely. But never "recover" from a caller who passed a negative amount, because that's not a *failure*, it's a *defect*.

| | Defensive programming | Design by Contract |
|---|---|---|
| Assumption | Everyone is lying to me | We agreed on the rules; violations are bugs |
| On bad input from *my own code* | Handle it, return a default | **Crash.** It's a defect. |
| On bad input from the *outside* | Handle it | Handle it (validate at the boundary) — same thing |
| Where checks live | Everywhere, duplicated | Boundary (validate) + core (assert) |
| Result | Bugs hide; the same rule lives in 7 files | Bugs surface at their cause; the rule lives in 1 place |

### 7. DbC and Liskov Substitution — the rule that makes LSP precise

Recall from [16 — Liskov Substitution](./16-solid-liskov-substitution.md) that a subclass must be usable anywhere its parent is used, without the caller noticing. DbC turns that fuzzy sentence into a **checkable rule**:

> **A subclass may weaken preconditions and strengthen postconditions. Never the reverse.**
>
> "Demand no more, promise no less."

Think about why, from the caller's seat. The caller wrote its code against the **parent's** contract. It satisfies the parent's preconditions and relies on the parent's postconditions. If you hand it a subclass:

- If the subclass **demands MORE** (a stronger precondition) → the caller, who only promised what the parent asked for, now fails. **It breaks.** ❌
- If the subclass **demands LESS** (a weaker precondition) → the caller is over-delivering. Harmless. ✅
- If the subclass **promises LESS** (a weaker postcondition) → the caller relied on the parent's promise and now gets less. **It breaks.** ❌
- If the subclass **promises MORE** (a stronger postcondition) → the caller gets a bonus it didn't ask for. Harmless. ✅

And invariants can only be **strengthened**, never weakened — a subclass must uphold every promise the parent made about its state, and may add its own.

```javascript
class Account {
  // PRE:  amount > 0 AND amount <= balance
  // POST: balance === old(balance) - amount
  withdraw(amountCents) { /* ... */ }
}

// ✅ LEGAL — WEAKENS the precondition. It accepts everything the parent accepted,
//    AND MORE (it also allows going into the overdraft). Every existing caller
//    still works untouched, because it was already satisfying the stricter rule.
class OverdraftAccount extends Account {
  #limitCents = 50_000;
  // PRE:  amount > 0 AND amount <= balance + overdraftLimit   ← WEAKER. Accepts more.
  // POST: balance === old(balance) - amount                    ← same promise, kept.
  withdraw(amountCents) {
    requires(amountCents > 0, 'amount must be positive');
    requires(amountCents <= this.balanceCents + this.#limitCents, 'exceeds overdraft limit');
    // ...
  }
  // NOTE: the class invariant must weaken to match: `balance >= -limit`, not `balance >= 0`.
  // That's a red flag worth noticing — see the warning below.
}

// ❌ ILLEGAL — STRENGTHENS the precondition. Demands MORE than the parent.
//    Every caller written against Account now randomly explodes when handed one of these.
class MinimumBalanceAccount extends Account {
  // PRE: amount > 0 AND amount <= balance - 10_000   ← STRONGER. Rejects calls the parent accepted.
  withdraw(amountCents) {
    requires(amountCents <= this.balanceCents - 10_000, 'must keep ₹100 minimum');
    // A caller doing `if (amt <= acc.balance) acc.withdraw(amt)` — which is CORRECT
    // against Account's contract — now throws. The subclass broke its parent's promise.
  }
}

// ❌ ILLEGAL — WEAKENS the postcondition. Promises LESS than the parent.
class FeeAccount extends Account {
  // POST: balance === old(balance) - amount - fee   ← NOT what the parent promised.
  withdraw(amountCents) {
    super.withdraw(amountCents);
    this.balanceCents -= 5_000;   // A caller computing `expected = old - amount` is now WRONG.
  }
}
```

A subtle and important warning on `OverdraftAccount`: it weakens the *precondition* legally, but to do so it must also weaken the *class invariant* from `balance >= 0` to `balance >= -limit`. Invariants may only be **strengthened** by subclasses. So strictly, `OverdraftAccount` should **not** extend an `Account` whose invariant is `balance >= 0`. The clean fix is to lift the shared behaviour into a base whose invariant is `balance >= -overdraftLimit` (with `overdraftLimit = 0` for a plain account). Working this out on a whiteboard — *"wait, that breaks the invariant, so the hierarchy is wrong"* — is a genuinely strong interview moment, because it shows contracts *catching a design error before the code exists*.

**The mnemonic, and it's worth memorising verbatim:**

```
        ┌──────────────────────────────────────────────┐
        │   A subclass may:                            │
        │                                              │
        │      DEMAND NO MORE   (weaker preconditions) │
        │      PROMISE NO LESS  (stronger postconds)   │
        │      UPHOLD ALL       (stronger invariants)  │
        └──────────────────────────────────────────────┘
```

Every LSP violation you will ever see is a breach of one of those three lines. `Square extends Rectangle` breaks it too: `Rectangle.setWidth` postcondition says *"width changed and height did NOT"*; `Square.setWidth` changes the height, so it **weakens the postcondition**. Illegal. That's the rigorous proof, and it's much stronger than hand-waving about "squares aren't really rectangles."

---

## Visual / Diagram description

### Diagram 1: The contract sandwich

```
   ┌────────────────────────────────────────────────────────────────┐
   │                          CALLER                                 │
   │                                                                │
   │     account.withdraw(2500)                                     │
   └────────────────────────────┬───────────────────────────────────┘
                                │
                                │  the caller MUST guarantee:
                                ▼
   ╔════════════════════════════════════════════════════════════════╗
   ║  PRECONDITIONS  (requires)     — CALLER's obligation            ║
   ║    P1. Number.isInteger(amount)                                ║
   ║    P2. amount > 0                                              ║
   ║    P3. amount <= balance                                       ║
   ║                                                                ║
   ║    ✗ violated → [PRECONDITION] throw  → THE CALLER HAS A BUG   ║
   ╚════════════════════════════════════════════════════════════════╝
                                │
   ┌────────────────────────────▼───────────────────────────────────┐
   │  INVARIANTS hold here  ──►  balance >= 0, history sums         │
   ├────────────────────────────────────────────────────────────────┤
   │                                                                │
   │      const before = this.#balance;      ← snapshot the "old"   │
   │                                                                │
   │      this.#balance -= amount;           ← THE ACTUAL WORK      │
   │      this.#transactions.push(...)                              │
   │                                          (invariants may be    │
   │                                           temporarily FALSE    │
   │                                           in here — that's OK) │
   │                                                                │
   ├────────────────────────────────────────────────────────────────┤
   │  INVARIANTS restored  ──►  #checkInvariants()                  │
   └────────────────────────────┬───────────────────────────────────┘
                                │
                                │  the method MUST guarantee:
                                ▼
   ╔════════════════════════════════════════════════════════════════╗
   ║  POSTCONDITIONS  (ensures)     — METHOD's obligation            ║
   ║    Q1. balance === before - amount     (EXACTLY)                ║
   ║    Q2. exactly one transaction appended                        ║
   ║    Q3. return value === new balance                            ║
   ║                                                                ║
   ║    ✗ violated → [POSTCONDITION] throw → *I* HAVE A BUG          ║
   ╚════════════════════════════════════════════════════════════════╝
                                │
                                ▼
                    return this.#balanceCents
```

The three bands are the whole idea. **Top band = your fault, caller. Bottom band = my fault. The rails running down both sides = the class's fault.**

### Diagram 2: Boundary vs core — where each kind of check lives

```
        UNTRUSTED                          TRUSTED
   ┌─── the outside world ───┐   ┌──── your own code ────────────────┐
   │                         │   │                                   │
   │  ┌───────────────────┐  │   │  ┌──────────────┐  ┌───────────┐ │
   │  │ HTTP request      │  │   │  │ OrderService │  │BankAccount│ │
   │  │ CLI args          │──┼───┼─▶│              │─▶│           │ │
   │  │ Kafka message     │  │   │  │              │  │ requires()│ │
   │  │ Webhook           │  │   │  └──────────────┘  │ ensures() │ │
   │  │ CSV upload        │  │   │                    │ invariant()│ │
   │  └─────────┬─────────┘  │   │                    └───────────┘ │
   └────────────┼────────────┘   └───────────────────────────────────┘
                │                                    ▲
                ▼                                    │
   ╔════════════════════════╗                        │
   ║   THE BOUNDARY         ║   everything past      │
   ║                        ║   here is TRUSTED  ────┘
   ║   VALIDATE  (zod/Joi)  ║
   ║                        ║
   ║   bad input =          ║      inside the core:
   ║   EXPECTED             ║      CONTRACTS (assert)
   ║        ↓               ║
   ║   400 Bad Request      ║      bad input = A BUG
   ║   "amount must be > 0" ║           ↓
   ║                        ║      💥 throw + page someone
   ║   (polite, listed,     ║
   ║    no crash)           ║      (loud, immediate, fatal)
   ╚════════════════════════╝
```

The same condition (`amount > 0`) is checked in both places, and it is **not** duplication — the boundary check writes a message for a *human*; the core assertion writes a stack trace for an *engineer*. One is a feature; the other is a tripwire.

---

## Real world examples

### 1. TypeScript — preconditions, checked by the compiler

A type annotation is a precondition that a machine enforces before your program ever runs:

```typescript
withdraw(amountCents: number): number
```

That signature *is* the precondition `typeof amount === 'number'`, and the compiler makes it impossible to violate. It's the strongest form of contract available: violations aren't caught at runtime, they're caught at *write* time.

But note the limit, and interviewers love this follow-up: types can express `number`, but they **cannot** express `amount > 0 && amount <= balance`. That's a *value* constraint, not a *shape* constraint, and it depends on runtime state. Types get you the cheap half of the contract for free. Runtime assertions are still required for the half that matters most.

### 2. Eiffel — the language that invented it

Bertrand Meyer designed Eiffel in 1986 around this idea, and it has `require`, `ensure`, `invariant`, and `old` as **first-class keywords**:

```
withdraw (amount: INTEGER)
    require
        positive: amount > 0
        sufficient_funds: amount <= balance
    do
        balance := balance - amount
    ensure
        balance_decreased: balance = old balance - amount
    end
invariant
    non_negative_balance: balance >= 0
```

That `old balance` is exactly the `const before = this.#balance` trick we wrote by hand — Eiffel gives it to you as syntax. The compiler even *inherits* contracts down the hierarchy and enforces the LSP rule (weaken preconditions, strengthen postconditions) **at compile time**. Every idea in this doc is a JavaScript reconstruction of that 1986 design.

### 3. Node.js core and database CHECK constraints

Node's own standard library uses fail-fast contracts everywhere. Pass a bad argument and you get `ERR_INVALID_ARG_TYPE` immediately — a thrown `TypeError` at the call site, not `undefined` three frames later. That error code *is* a precondition violation, reported with blame attached.

And your database has been doing DbC the whole time:

```sql
CREATE TABLE accounts (
  id            TEXT PRIMARY KEY,
  balance_cents BIGINT NOT NULL,
  CONSTRAINT non_negative_balance CHECK (balance_cents >= 0)   -- ← a class INVARIANT,
);                                                             --   enforced by the engine
```

That `CHECK` is the invariant `balance >= 0`, enforced by a system that *cannot be bypassed* — not by a buggy service, not by a migration script, not by an intern with a psql prompt. It's the last line of defence, and it's why you write the invariant in *both* the domain class (fast feedback, good error messages) and the schema (absolute guarantee). Two enforcement points, one rule.

---

## Trade-offs

| | Pros | Cons |
|---|---|---|
| **Runtime contracts everywhere** | Bugs surface at their cause, not 6 hours downstream; the code documents itself; refactors are safe | Runtime cost (usually nanoseconds, but real in a hot loop); more lines per method; noisy if applied to trivial getters |
| **Contracts only in the core domain** | Great value/cost ratio; the boundary already validates | A new caller that bypasses the boundary (a script, a migration, a queue consumer) can still corrupt state |
| **Fail fast (throw)** | Loud, immediate, unambiguous; a crash is honest | A crash in production is a crash; you *must* have the boundary layer, or a user typo becomes a 500 |
| **Defensive (clamp, default, return null)** | Never crashes; feels "safe" | Converts loud bugs into quiet wrong answers; you can no longer state a postcondition at all |
| **Types instead of contracts** | Free, compile-time, zero runtime cost | Can only express *shape*, never *value* (`amount > 0`) or *relationships between state* |

**The sweet spot:**
- **Validate at every boundary** (HTTP, CLI, queue, file import) with a real schema library. Bad input from the outside is *expected*, and gets a 400, not a stack trace.
- **Assert in the core** with `requires` / `ensures` / `invariant`. Bad input from your own code is a *bug*, and gets a crash.
- **Put the hard invariants in the database too** (`CHECK`, `NOT NULL`, `FOREIGN KEY`). Belt and braces on anything involving money.
- **Skip contracts on trivial code.** A getter doesn't need a precondition. Apply them where state can be *corrupted* and where the invariant is *non-obvious*.
- **Rule of thumb:** if a violation means *a user made a mistake*, validate and respond politely. If it means *a programmer made a mistake*, assert and crash. Ask "who is to blame?" and the right technique falls out automatically.

---

## Common interview questions on this topic

### Q1: "What's the difference between a precondition, a postcondition, and an invariant?"
**Hint:** Frame it as **blame**, which is what actually distinguishes them. Precondition = what the **caller** must guarantee before calling (violation = *caller's* bug). Postcondition = what the **method** guarantees on return (violation = *my* bug). Invariant = what the **class** guarantees at all times, before and after every public method (violation = the object is *corrupt*). Then give `BankAccount.withdraw`: PRE `0 < amount <= balance`; POST `balance === old(balance) - amount`; INVARIANT `balance >= 0`. Add the nuance that the invariant may be temporarily false *inside* a method but must be restored before returning.

### Q2: "How does Design by Contract relate to the Liskov Substitution Principle?"
**Hint:** DbC is what makes LSP *precise* and checkable. The rule: a subclass may **weaken preconditions** and **strengthen postconditions**, never the reverse — *"demand no more, promise no less"* — and invariants may only be strengthened. Explain from the caller's seat: the caller was written against the *parent's* contract, so a subclass that demands more will reject calls the caller was entitled to make. Then prove `Square extends Rectangle` is illegal with it: `Rectangle.setWidth`'s postcondition is "width changed AND height unchanged"; `Square.setWidth` changes the height, so it *weakens the postcondition*. That's a rigorous proof, not a vibe.

### Q3: "Isn't this just defensive programming?"
**Hint:** No — they're near-opposites in philosophy. Defensive programming *recovers* from bad input (clamp it, default it, return null), which turns a loud bug into a quiet wrong answer that ships to production and runs for a year. Contracts *refuse* — they crash, so the defect gets fixed. The tell: after a defensive `if (amount > balance) amount = balance;`, you can no longer *state* a postcondition — "balance decreases by amount, or by some other amount, or not at all" is not a contract. Then concede the right nuance: defensive coding is correct at the *boundary* (untrusted input) and against genuinely unreliable I/O; contracts are correct in the *core*.

### Q4: "Where do you put validation — everywhere, or one place?"
**Hint:** Two places, for two different audiences. **Validate at the boundary** (HTTP handler, queue consumer, CLI) with a schema library — bad input from a *human* is expected, and gets `400 Bad Request` with a friendly, complete list of problems. **Assert in the core** with contracts — bad input from *your own code* is a bug, and gets a crash. This is not duplication: the boundary check writes a message for a user; the assertion writes a stack trace for an engineer. Everything between the two is trusted, and that trust is what stops the same `if (!amount)` from appearing in seven files.

### Q5: "Should contracts be enabled in production?"
**Hint:** Give a real answer, not a dogma. Preconditions and invariants: **yes, keep them on** — the cost is a few nanoseconds and the alternative is silent data corruption in a system handling money. Expensive postconditions (e.g. an O(n) `#checkInvariants` that re-sums a 100k-row history on every call) are the only real candidates for a flag — sample them, or run them only in tests and staging. The classic guidance is "assertions off in production", and that guidance is **wrong for anything financial**: a crash costs you one page, a corrupt balance costs you a reconciliation project. Know the trade-off and name the amount at stake.

---

## Practice exercise

### The Contracted Inventory (~35 min)

Build an `InventoryItem` class for a warehouse, with contracts enforced in code.

**The domain:** an item has a `sku`, an `onHand` count (physically in the warehouse), and a `reserved` count (promised to orders but not yet shipped).

**Part A — Write the contract FIRST, in comments, before any code (10 min).** For each of these, write down the preconditions, the postconditions, and the class invariants. Do this on paper before you type.

- `constructor(sku, onHand)`
- `reserve(qty)` — promise stock to an order
- `release(qty)` — a cancelled order gives stock back
- `ship(qty)` — reserved stock physically leaves the building
- `restock(qty)` — new stock arrives

The invariants are the interesting part. At minimum: `onHand >= 0`, `reserved >= 0`, and — the one everybody misses — **`reserved <= onHand`** (you cannot promise more than you physically have). Getting that third one is the point of the exercise.

**Part B — Implement it (15 min).** Write `contracts.js` with `requires` / `ensures` / `invariant` (steal the ones from section 4), then `InventoryItem` with a private `#checkInvariants()` called at the end of the constructor and every mutator. Every postcondition must be *exact* (`ensures(this.#onHand === before - qty)`, not "it got smaller").

**Part C — Break it, on purpose (10 min).** Write a `demo()` that attempts each of these and prints the `ContractError`:
1. `reserve(999)` when only 10 are on hand → which precondition fires?
2. `ship(5)` when only 2 are reserved → which one fires?
3. `release(-1)` → which one?
4. Now **deliberately introduce a bug**: make `ship(qty)` decrement `onHand` but *forget* to decrement `reserved`. Run it. **Which contract catches you, and how quickly?** (Answer: `invariant(reserved <= onHand)` — it fires on the very next call, at the line that broke it, instead of surfacing as an impossible stock report next quarter. Write down how long that bug would have survived without the invariant.)

**Part D — LSP (5 min).** Now write `class BackorderableItem extends InventoryItem`, which *allows* reserving more than `onHand`. Ask yourself the hard question: **does this violate LSP?** (It weakens `reserve`'s precondition — that's legal. But it also weakens the class invariant `reserved <= onHand` — and that is **not** legal.) Write one paragraph on how you'd restructure the hierarchy to fix it. This paragraph is the actual deliverable of the whole exercise.

**What to produce:** one runnable `.js` file, the contract comments from Part A, and the Part D paragraph.

---

## Quick reference cheat sheet

- **Precondition** — what the **caller** must guarantee *before* the call. Violated ⇒ **the caller has a bug**.
- **Postcondition** — what the **method** guarantees *on return*, if the preconditions held. Violated ⇒ **I have a bug**.
- **Invariant** — what the **class** guarantees *at all times* — after the constructor, before and after every public method. Violated ⇒ **the object is corrupt**.
- **The whole point is blame.** Ask "whose fault is it?" and the right kind of check falls out automatically.
- **Invariants may be temporarily false *inside* a method** — that's fine. They must be restored before you return.
- **Fail fast:** every line between a bug's cause and its symptom is a line you'll debug at 3am. Throw at the cause.
- **`requires` / `ensures` / `invariant`** — three names for one `if (!x) throw`, separated so the error message tells you *where to look first*.
- **Snapshot the old state** (`const before = this.#balance`) so postconditions can be **exact**: `=== before - amount`, never "it decreased".
- **Boundary = validate** (HTTP/CLI/queue → `400`, expected, be polite). **Core = assert** (contracts → crash, it's a bug). **Trust everything in between.**
- **That's not duplication.** The boundary check writes for a *user*; the assertion writes for an *engineer*.
- **Defensive programming's sin is the silent recovery** — clamping turns a loud bug into a quiet wrong answer. And it destroys your ability to even *state* a postcondition.
- **DbC + LSP:** a subclass may **weaken preconditions** and **strengthen postconditions**. Never the reverse. Invariants may only be **strengthened**.
- **Memorise:** *"Demand no more, promise no less, uphold all."*
- **Money is integer cents, never floats.** `0.1 + 0.2 !== 0.3`. Make it an invariant: `Number.isInteger(balanceCents)`.
- **Put the hard invariants in the database too** (`CHECK (balance_cents >= 0)`) — the only enforcement point nothing can bypass.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [24 — Coupling and Cohesion](./24-coupling-and-cohesion.md) — contracts are what make a decoupled boundary trustworthy enough to rely on |
| **Next** | [26 — Composition Over Inheritance](./26-composition-over-inheritance.md) — when the contract rules show your hierarchy is wrong, composition is the way out |
| **Related** | [16 — SOLID: Liskov Substitution](./16-solid-liskov-substitution.md) — DbC is what turns LSP from a slogan into a checkable rule |
| **Related** | [27 — Abstraction and Interfaces in Design](./27-abstraction-and-interfaces.md) — an interface is a contract with the implementation left blank |
| **Related** | [13 — OOP: The Four Pillars](./13-oop-four-pillars.md) — encapsulation (private `#` fields) is what makes an invariant *enforceable* in the first place |
