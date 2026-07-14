# 22 — UML Use Case Diagrams — How to Model Requirements
## Category: LLD Fundamentals

---

## What is this?

A **use case diagram** is a picture of *who* can use your system and *what* they can do with it. That's the whole idea. Stick figures on the outside (the users and other systems), ovals on the inside (the things they can do), a box around the ovals (the boundary of what you're building).

Think of the **menu at a restaurant**. The menu doesn't tell you how the chef cooks anything. It tells you who the menu is for (diners, and separately the wait staff who have their own "order pad" menu) and what can be ordered. A use case diagram is your system's menu: the complete list of orderable things, and who is allowed to order what.

---

## Why does it matter?

Because **it is the fastest legal way to derive an API in an interview.**

Most candidates, when asked "Design an ATM," freeze — they don't know where to start, so they start typing classes and hope structure emerges. It doesn't. The candidates who do well spend the first three minutes listing actors and use cases, and by minute four they have a service class with eight named methods on it and the interviewer is nodding.

**In interviews:**
- Every LLD round ("Design a parking lot / elevator / vending machine / BookMyShow") starts with requirements. Use cases *are* the requirements, in a form you can turn into code.
- Interviewers explicitly grade "did the candidate clarify scope before coding?" A use case list is scope, made visible.
- It stops you from designing features nobody asked for. If it's not a use case, don't build it.

**At work:**
- Product gives you a fuzzy doc. You reply with 12 use cases and 3 actors and ask "is this all of it?" Half the ambiguity dies right there.
- It surfaces the actors people forget — the **admin**, the **cron job**, the **payment provider's webhook**. Those forgotten actors are where the production incidents come from.

If you skip this step, the classic failure is: you build a beautiful `Order` class hierarchy and then discover on day nine that a *refunds team* also touches orders, and nothing in your design accounts for them.

---

## The core idea — explained simply

### The ATM Lobby Analogy

Stand outside an ATM lobby and just watch for a day. You will see exactly three kinds of people/things interacting with that machine:

1. **The customer.** Walks up, puts in a card, takes out money, sometimes deposits, sometimes just checks a balance, occasionally moves money between accounts. She *initiates* — she is the reason the machine wakes up.
2. **The bank's servers.** You never see them, but they're there. The ATM *calls them* for every single thing — verify the PIN, check the balance, debit the account. They never walk up and start a session on their own. They *respond*.
3. **The technician.** Comes once a week in a van, opens the back, refills the cash cassettes, runs diagnostics, leaves. He uses the machine in a completely different way than the customer — different door, different capabilities.

That's it. Everything the ATM does is triggered by one of those three, and everything it does falls into a short list of *goals*: authenticate, withdraw, deposit, check balance, transfer, refill.

Now here's the whole vocabulary of use case diagrams, mapped straight onto the lobby:

| What you saw in the lobby | UML name | Drawn as | Meaning |
|---|---|---|---|
| The customer | **Primary actor** | Stick figure on the LEFT | Starts the interaction. Wants something. |
| The bank's servers | **Secondary (supporting) actor** | Stick figure on the RIGHT | The system calls *it*. It never initiates. |
| The technician | Another **primary actor** | Stick figure (left, or bottom) | A different role with a different goal set |
| "Withdraw cash" | **Use case** | Oval | One complete goal, valuable on its own |
| The physical machine | **System boundary** | Rectangle around the ovals | What you are building. Actors are OUTSIDE it. |
| Customer → Withdraw | **Association** | Plain line | "This actor can do this" |
| Withdraw always needs a PIN check | **`<<include>>`** | Dashed arrow, points to the *included* case | Mandatory sub-goal, always runs |
| Withdrawing *may* print a receipt | **`<<extend>>`** | Dashed arrow, points *back at* the base case | Optional behaviour, sometimes runs |

**The single most important rule:** an actor is a **role**, not a person. The same human being is a "Customer" when they withdraw ₹2000 and an "Admin" when they log into the bank's back office. Two actors, one human. And an actor doesn't have to be human at all — the Bank System is an actor, and so is a nightly `settlement-cron` if it pokes your service.

---

## Key concepts inside this topic

### 1. Actors — primary vs secondary, human vs system

An **actor** is anything outside your system that interacts with it. Four flavours, and you should be able to name which is which:

```
                    │ HUMAN                    │ EXTERNAL SYSTEM
────────────────────┼──────────────────────────┼──────────────────────────
PRIMARY             │ Customer, Admin,         │ Cron scheduler,
(initiates; wants   │ Technician, Driver       │ Partner system firing
something from you) │                          │ a webhook at you
────────────────────┼──────────────────────────┼──────────────────────────
SECONDARY           │ Rare, but real:          │ Bank core system, Stripe,
(you call IT; it    │ a human approver in a    │ SendGrid, an SMS gateway,
supports you)       │ manual review queue      │ your own Postgres? NO —
                    │                          │ see below
```

**Two traps:**

- **Your own database is not an actor.** It's *inside* the boundary; it's part of the system you're building. Stripe is an actor because it's owned by someone else and you cannot change it. Your Postgres you own.
- **"Time" is a legitimate actor.** If something happens because a clock ticked (nightly settlement, subscription renewal), draw a `Scheduler` actor. Everyone forgets this one, and then their design has no place for background jobs.

### 2. Use cases — what counts as one

A use case is **one complete goal that leaves the actor better off.** Test it with the *"coffee break test"*: if the actor could go get a coffee afterwards feeling like they achieved something, it's a use case.

| Candidate | Use case? | Why |
|---|---|---|
| "Withdraw Cash" | ✅ Yes | Actor leaves with money. Goal achieved. |
| "Check Balance" | ✅ Yes | Actor leaves knowing something they wanted to know. |
| "Enter PIN" | ❌ No | That's a *step*, not a goal. Nobody drives to an ATM to enter a PIN. |
| "Validate PIN with bank" | ⚠️ Only as an `<<include>>` | It's a reusable sub-goal, not a standalone one. |
| "Click the green button" | ❌ No | That's UI. Use cases are UI-independent. |
| "Manage Users" | ❌ Too vague | Split it: Create User, Suspend User, Reset Password. |

Name every use case **verb + noun**: `Withdraw Cash`, `Apply Coupon`, `Refill Cash`. If you can't phrase it as verb+noun, it's probably not a use case.

### 3. `<<include>>` — the "always happens" arrow

`<<include>>` means: **the base use case cannot complete without this. It runs every single time. The base case knows about it and explicitly calls it.**

The real reason it exists is **de-duplication**. You extract an include when three different use cases all need the same sub-step, and you're tired of drawing it three times.

```
   ( Withdraw Cash )──────<<include>>─────▶( Authenticate )
   ( Deposit Cash  )──────<<include>>─────▶( Authenticate )
   ( Transfer      )──────<<include>>─────▶( Authenticate )
```

Read the arrow as **"includes"**: *Withdraw Cash* **includes** *Authenticate*. The arrow points at the thing being included. Always.

In code, `<<include>>` is a **direct, unconditional call** from the base method:

```javascript
async withdrawCash(cardNumber, pin, amount) {
  const session = await this.#authenticate(cardNumber, pin); // <<include>> — always runs
  ...
}
```

### 4. `<<extend>>` — the "sometimes happens" arrow

`<<extend>>` means: **optional, conditional behaviour that hooks into the base use case at a defined point. The base use case does NOT know it exists.** The base runs perfectly fine on its own; the extension *decorates* it when some condition holds.

The classic: **`Apply Coupon` extends `Place Order`.** You can absolutely place an order with no coupon. The order flow doesn't need to know coupons exist. But *if* you have a coupon, the coupon behaviour plugs in at the "calculate total" extension point.

```
   ( Apply Coupon )──────<<extend>>─────▶( Place Order )
        ^                                      ^
   the OPTIONAL thing                    the BASE thing
   (arrow starts here)                   (arrow points here)
```

Notice the arrow points the **opposite way** from `<<include>>`. That's not UML being annoying — it's meaningful. The arrow always points toward the thing that is *depended upon* or *added to*:

- `<<include>>`: the base **depends on** the included case → arrow points at the included case.
- `<<extend>>`: the extension **adds to** the base → arrow points at the base.

### 5. include vs extend — the distinction everyone gets wrong

Burn this table into your head. In interviews, getting this backwards is a visible, expensive mistake.

| | `<<include>>` | `<<extend>>` |
|---|---|---|
| **Runs** | Always. Every time. 100%. | Sometimes. Only if a condition holds. |
| **Arrow direction** | Base ──▶ Included | Extension ──▶ Base |
| **Who knows about whom** | Base **knows** about the included case and calls it | Base is **oblivious**; extension knows the base |
| **Remove it and…** | Base is **broken** — it can't function | Base still works fine, just with less flair |
| **Why it exists** | Reuse a mandatory step across several use cases | Add optional/exceptional behaviour without touching the base |
| **Code analogy** | A **private helper method** called unconditionally | A **hook / plugin / decorator / if-block** on an optional path |
| **Example** | Place Order **includes** Validate Payment | Apply Coupon **extends** Place Order |

**The 5-second test:** ask *"Can the base use case succeed without this?"*
- **No** → `<<include>>`.
- **Yes** → `<<extend>>`.

Sanity-check: Can you place an order without validating payment? No — that's fraud. So it's `include`. Can you place an order without applying a coupon? Obviously yes. So it's `extend`.

**The classic wrong answer:** candidates draw `Place Order <<include>> Apply Coupon` because "the order flow calls the coupon code." Doesn't matter what the code does — the question is *does it always run*. It doesn't. It's an extend.

**Honest advice:** in a real interview, `<<include>>` is worth using and `<<extend>>` is worth *mentioning*. Many teams (and Martin Fowler, loudly) argue `<<extend>>` causes more confusion than it's worth. Use include freely. Use extend once, correctly, to show you know the difference, and move on.

### 6. Actor generalization

Actors can inherit from each other, exactly like classes. Draw a **hollow triangle arrow** pointing from the specific actor to the general one.

```
              ┌──────────────┐
              │  BankStaff   │   (general role: can Check Balance)
              └──────▲───────┘
                     │  △  (hollow triangle = generalization)
          ┌──────────┴──────────┐
    ┌─────┴──────┐       ┌──────┴──────┐
    │ Technician │       │   Auditor   │
    └────────────┘       └─────────────┘
   + Refill Cash          + Export Logs
```

Meaning: **a Technician can do everything BankStaff can do, plus Refill Cash.** It's Liskov substitution for actors (recall from [16 — Liskov Substitution] that a subtype must be usable anywhere the supertype is).

Use it when two actors share most of their use cases and you're tired of drawing the same lines twice. Don't use it to model *organizational hierarchy* — a Manager is not a "kind of" Employee for use-case purposes unless the Manager genuinely inherits every one of the Employee's capabilities.

### 7. The system boundary box

The rectangle is not decoration. **It is the scope of the sprint.** Ovals inside = you're building it. Stick figures outside = you're not.

In an interview, drawing the box and saying *"I'm treating the bank's core banking system as an external actor — I'm not designing that"* is a scope-control move. It buys you permission to not design a bank.

---

## Visual / Diagram description

### Diagram 1: The complete ATM use case diagram

```
                            ┌──────────────────────────────────────────────┐
                            │              ATM SYSTEM                       │
                            │                                              │
                            │            ┌──────────────────┐              │
   ┌───┐                    │   ┌───────▶│   Authenticate   │◀──────┐      │
   │ O │  Customer          │   │        └──────────────────┘       │      │
   │/|\│ ───────────────────┼───┤                 ▲                 │      │        ┌───┐
   │/ \│  (primary,         │   │                 │<<include>>      │      │        │ O │
   └───┘   human)           │   │                 │                 │      │────────│/|\│
                            │   │        ┌────────┴─────────┐       │      │        │/ \│
        association ────────┼───┼───────▶│  Withdraw Cash   │───────┼──────┤        └───┘
        (plain line)        │   │        └──────────────────┘       │      │   Bank System
                            │   │                 ▲                 │      │   (secondary,
                            │   │       <<extend>>│                 │      │    external)
                            │   │        ┌────────┴─────────┐       │      │
                            │   │        │  Print Receipt   │       │      │
                            │   │        └──────────────────┘       │      │
                            │   │                                   │      │
                            │   │        ┌──────────────────┐       │      │
                            │   ├───────▶│  Deposit Cash    │───────┼──────┤
                            │   │        └──────────────────┘       │      │
                            │   │                 │ <<include>>     │      │
                            │   │                 └─────────────────┘      │
                            │   │                     (Authenticate)       │
                            │   │        ┌──────────────────┐              │
                            │   ├───────▶│  Check Balance   │──────────────┤
                            │   │        └──────────────────┘              │
                            │   │                                          │
                            │   │        ┌──────────────────┐              │
                            │   └───────▶│  Transfer Funds  │──────────────┘
                            │            └──────────────────┘              │
                            │                                              │
   ┌───┐                    │            ┌──────────────────┐              │
   │ O │  Technician        │            │   Refill Cash    │              │
   │/|\│ ───────────────────┼───────────▶│                  │              │
   │/ \│  (primary,         │            └──────────────────┘              │
   └───┘   human)           │                     │ <<include>>            │
                            │                     ▼                        │
                            │            ┌──────────────────┐              │
                            │            │  Run Diagnostics │              │
                            │            └──────────────────┘              │
                            └──────────────────────────────────────────────┘
```

**How to read it, line by line:**

- **Customer** is on the left because she is the **primary actor** — she starts every session. She has plain association lines to five use cases: Authenticate, Withdraw Cash, Deposit Cash, Check Balance, Transfer Funds.
- **Bank System** is on the right because it is a **secondary actor** — the ATM calls *it*. Withdraw, Deposit, Check Balance, Transfer and Authenticate all reach out to it. It never initiates anything.
- **Technician** is a second primary actor with a completely disjoint capability: `Refill Cash`. He does not use the customer's use cases at all (different door, different key). `Refill Cash` **includes** `Run Diagnostics` — the machine always self-tests after a refill, no exceptions.
- **Withdraw / Deposit / Transfer / Check Balance all `<<include>> Authenticate`** — you cannot do any of them without a valid session. Arrow points at `Authenticate`, the included case.
- **`Print Receipt` `<<extend>>s` `Withdraw Cash`** — arrow points *back at* Withdraw. The customer taps "no receipt" half the time and the withdrawal completes perfectly. Optional. That's an extend.
- The **box** is the ATM. Everything I'm building is inside it. The bank's core ledger is outside — not my problem today.

### Diagram 2: The arrow-direction cheat card (redraw this from memory)

```
   INCLUDE — mandatory, base calls it, arrow points AT the sub-goal
   ┌────────────────┐                        ┌──────────────────┐
   │   Place Order  │- - - <<include>> - - -▶│ Validate Payment │
   └────────────────┘                        └──────────────────┘
        "the base"                              "always runs"

   EXTEND — optional, base is unaware, arrow points AT the base
   ┌────────────────┐                        ┌──────────────────┐
   │  Apply Coupon  │- - - <<extend>>  - - -▶│   Place Order    │
   └────────────────┘                        └──────────────────┘
     "the optional"                             "the base"
```

Both arrows are **dashed** (they're dependencies, not associations). The difference is purely which end they point at — and that direction encodes *who depends on whom*.

---

## The payoff — how use cases become your class design

This is the part nobody teaches, and it's the reason to bother with any of this.

**The mapping is mechanical:**

| Use case diagram element | Becomes, in code |
|---|---|
| **Primary actor** | A **role** (an enum/permission) and often a **facade or controller** per actor |
| **Use case (oval)** | A **public method** on a service/facade — one per oval, same name, camelCased |
| **`<<include>>`** | A **private helper method**, called unconditionally by the base method |
| **`<<extend>>`** | An **optional parameter / hook / strategy**, guarded by an `if` |
| **Secondary actor** | An **injected dependency** (an interface you call out to) |
| **System boundary** | The **module/package** you're writing |
| **Actor generalization** | **Role inheritance** in your permission check |

Watch it happen. Here is that exact ATM diagram, turned into JavaScript with zero creative leaps:

```javascript
// ─── The actors become ROLES ───────────────────────────────────────────
const Role = Object.freeze({
  CUSTOMER:   'CUSTOMER',
  TECHNICIAN: 'TECHNICIAN',
});

// ─── The SECONDARY actor becomes an injected dependency (an interface) ──
// We don't build the bank. We call it. That's what "secondary actor" means.
class BankGateway {
  async verifyPin(cardNumber, pin)        { throw new Error('Not implemented'); }
  async getBalance(accountId)             { throw new Error('Not implemented'); }
  async debit(accountId, amount)          { throw new Error('Not implemented'); }
  async credit(accountId, amount)         { throw new Error('Not implemented'); }
  async transfer(fromId, toId, amount)    { throw new Error('Not implemented'); }
}

// ─── The SYSTEM BOUNDARY becomes this one class ────────────────────────
// Every OVAL in the diagram is a PUBLIC method here. Nothing more, nothing less.
class AtmService {
  #bank;          // secondary actor
  #cashDispenser; // internal component (inside the boundary — not an actor)
  #sessions = new Map();

  constructor({ bank, cashDispenser }) {
    this.#bank = bank;                   // DIP: depend on the abstraction (topic 18)
    this.#cashDispenser = cashDispenser;
  }

  // ── USE CASE: Authenticate ───────────────────────────────────────────
  // It's an oval AND an <<include>> target, so it's public (Customer can
  // trigger it directly) and also reused internally by the others.
  async authenticate(cardNumber, pin) {
    const account = await this.#bank.verifyPin(cardNumber, pin);
    if (!account) throw new Error('Invalid PIN');

    const sessionId = crypto.randomUUID();
    this.#sessions.set(sessionId, { accountId: account.id, role: Role.CUSTOMER });
    return sessionId;
  }

  // ── USE CASE: Withdraw Cash ──────────────────────────────────────────
  // includes Authenticate, extended by Print Receipt.
  async withdrawCash(sessionId, amount, { printReceipt = false } = {}) {
    const session = this.#requireSession(sessionId, Role.CUSTOMER); // <<include>>

    if (amount % 100 !== 0) throw new Error('Amount must be a multiple of 100');
    if (!this.#cashDispenser.canDispense(amount)) throw new Error('Insufficient cash in ATM');

    await this.#bank.debit(session.accountId, amount); // debit BEFORE dispensing:
    this.#cashDispenser.dispense(amount);              // never hand out unfunded cash

    // <<extend>> — optional, conditional, base works fine without it.
    // Notice it is an `if`, not a call the base flow depends on.
    const receipt = printReceipt ? this.#printReceipt(session.accountId, 'WITHDRAWAL', amount) : null;

    return { dispensed: amount, receipt };
  }

  // ── USE CASE: Deposit Cash ───────────────────────────────────────────
  async depositCash(sessionId, envelope) {
    const session = this.#requireSession(sessionId, Role.CUSTOMER); // <<include>>
    const amount = this.#cashDispenser.countNotes(envelope);
    await this.#bank.credit(session.accountId, amount);
    return { deposited: amount };
  }

  // ── USE CASE: Check Balance ──────────────────────────────────────────
  async checkBalance(sessionId) {
    const session = this.#requireSession(sessionId, Role.CUSTOMER); // <<include>>
    return this.#bank.getBalance(session.accountId);
  }

  // ── USE CASE: Transfer Funds ─────────────────────────────────────────
  async transferFunds(sessionId, toAccountId, amount) {
    const session = this.#requireSession(sessionId, Role.CUSTOMER); // <<include>>
    if (session.accountId === toAccountId) throw new Error('Cannot transfer to self');
    return this.#bank.transfer(session.accountId, toAccountId, amount);
  }

  // ── USE CASE: Refill Cash (TECHNICIAN only) ──────────────────────────
  // Different actor → different permitted role. This line IS the actor
  // association from the diagram, enforced at runtime.
  async refillCash(sessionId, notesByDenomination) {
    this.#requireSession(sessionId, Role.TECHNICIAN); // <<include>>
    this.#cashDispenser.load(notesByDenomination);
    return this.#runDiagnostics();                    // <<include>> — always runs after a refill
  }

  // ─── PRIVATE HELPERS === the <<include>> use cases ────────────────────
  // Every one of these exists in the diagram as an oval with an <<include>>
  // arrow pointing at it. That's not a coincidence — that's the mapping.

  #requireSession(sessionId, requiredRole) {
    const session = this.#sessions.get(sessionId);
    if (!session) throw new Error('Not authenticated');
    if (session.role !== requiredRole) throw new Error('Forbidden for this role');
    return session;
  }

  #runDiagnostics() {
    return { cashLevel: this.#cashDispenser.total(), printerOk: true, cardReaderOk: true };
  }

  #printReceipt(accountId, type, amount) {
    return `--- ATM RECEIPT ---\nAcct: ${accountId}\n${type}: ${amount}\n${new Date().toISOString()}`;
  }
}
```

**Look at what just happened.** You did not "design" anything. You transcribed a diagram. The public method list of `AtmService` is *exactly* the list of ovals. The private methods are *exactly* the `<<include>>` targets. The constructor dependency is *exactly* the secondary actor. The role check on `refillCash` is *exactly* the actor association line.

### The interview move: open EVERY LLD round this way

The next time you hear "Design a parking lot" — do not touch a class. Say this out loud:

> "Let me start with the actors. I see a **Driver**, a **Parking Attendant**, and an **Admin**. There's also a **Payment Gateway** as a secondary actor.
>
> The Driver can: **Find Available Spot**, **Park Vehicle**, **Unpark Vehicle**, **Pay Ticket**.
> The Attendant can: **Issue Ticket Manually**, **Override Barrier**.
> The Admin can: **Add Parking Floor**, **Set Pricing**, **View Occupancy Report**.
>
> `Pay Ticket` **includes** `Calculate Fee` — that always runs. `Apply Monthly Pass Discount` **extends** `Calculate Fee` — only some drivers have a pass.
>
> So my `ParkingLotService` has these methods…"

You have now, in ninety seconds:
- scoped the problem,
- named your actors,
- derived a **complete public API for free**,
- and shown you understand include vs extend.

The interviewer's next sentence will be "great, let's go deeper on X." That is the sound of a round going well.

---

## Real world examples

### 1. AWS IAM — actors and use cases became the permission model

AWS's entire authorization system is literally a use-case diagram compiled into JSON. An IAM policy statement names a **principal** (the actor), an **action** like `s3:PutObject` (the use case), and a **resource** (the system boundary). IAM **roles** are actor generalization made real — you define `ReadOnlyAccess` and then a more specific role that inherits it and adds write actions. When AWS ships a new feature, the first artifact they publish is the new list of *actions* — the new ovals.

### 2. Stripe — the secondary-actor-becomes-a-webhook story

When your checkout system does `Place Order`, Stripe is a **secondary actor**: you call it, it responds. But Stripe's `payment_intent.succeeded` webhook makes Stripe a **primary actor** too — it initiates a call into your system on its own schedule. Teams that model this correctly build a webhook handler as a first-class entry point with its own use cases (`Confirm Payment`, `Handle Chargeback`). Teams that forget it treat webhooks as an afterthought and end up with orders stuck in `PENDING` forever because nothing ever flipped them. The forgotten actor is where the outage lives.

### 3. Jira / GitHub Issues — use case names became the API surface

Look at GitHub's REST API for issues: `create`, `update`, `lock`, `assign`, `add labels`, `close`. Those are verb+noun use cases, one endpoint each. And the actor generalization is visible in the permission model: `read` < `triage` < `write` < `maintain` < `admin`, each inheriting the capabilities of the one below. That's a textbook actor generalization hierarchy shipped as a product feature.

---

## Trade-offs

| Pro of use case diagrams | Con |
|---|---|
| Fastest way to agree on **scope** with a non-technical stakeholder | Says **nothing** about *how* anything works — no logic, no data, no order of steps |
| Mechanically derives your public API | Tempts people into drawing 60 ovals for a CRUD app, which teaches nobody anything |
| Surfaces forgotten actors (admin, cron, webhooks) | `<<extend>>` genuinely confuses people; some teams ban it outright |
| Cheap — 5 minutes on a whiteboard | Not executable, drifts from reality unless maintained |
| UI-independent, so it survives redesigns | Can hide complexity: "Match Rider to Driver" is one innocent oval and six months of work |

| Situation | Do you draw one? |
|---|---|
| LLD interview, minute 1 | **Yes** — verbally at minimum. Free points. |
| New product, fuzzy requirements | **Yes** — it's the cheapest ambiguity-killer you own. |
| Adding one field to an existing endpoint | **No.** Absurd overkill. |
| Explaining an existing system to a new hire | **Yes** — it's the best one-page orientation artifact there is. |
| Your team already has clear PRDs with user stories | **Probably not** — user stories *are* use cases, in sentence form. Don't duplicate. |

**Rule of thumb:** draw the actors and the use case *list* always; draw the full diagram with include/extend arrows only when there's real reuse or real optionality worth showing. A bulleted list of "Actor → can do these things" delivers 80% of the value in 10% of the time.

---

## Common interview questions on this topic

### Q1: "What's the difference between `<<include>>` and `<<extend>>`?"
**Hint:** Include = **always** runs, base **calls** it, arrow points **at** the included case, base is broken without it (Place Order includes Validate Payment). Extend = **sometimes** runs, base is **unaware** of it, arrow points **back at** the base, base works fine without it (Apply Coupon extends Place Order). The one-line test: *"can the base succeed without it?"* Yes → extend. No → include.

### Q2: "Is the database an actor?"
**Hint:** No — your own database is *inside* the system boundary; it's an implementation detail of the thing you're building. An actor must be **external** and **not yours to change**. Stripe, an SMS gateway, the bank's core ledger, a partner API — those are actors. Your Postgres isn't. (Grey zone worth mentioning: a *shared* database owned by another team behaves like an external system, and modelling it as an actor is defensible.)

### Q3: "How do use cases relate to the classes you'll actually write?"
**Hint:** One use case → one public method on a service/facade. One `<<include>>` → one private helper. One secondary actor → one injected dependency (an interface). One primary actor → one role in your permission check. This is *the* answer that separates candidates — say it explicitly and then show the method list you derived.

### Q4: "Can an actor be a non-human?"
**Hint:** Yes, and forgetting this is the classic bug. External systems (payment gateways, email providers), other services, and **time itself** (a scheduler/cron firing a nightly job) are all actors. If something can wake your system up, it's a primary actor, even if it has no hands.

### Q5: "You've drawn 30 use cases for a to-do app. Is that good?"
**Hint:** No — it's a smell. You've almost certainly modelled UI *steps* ("Click Save", "Open Menu") or CRUD noise as goals. Collapse them to real goals that pass the coffee-break test. A to-do app has maybe 6 real use cases. Fewer, sharper ovals beat more.

---

## Practice exercise

### Derive an API for a Food Delivery App — from actors, in 30 minutes

Do this on paper first. No editor until step 4.

**Step 1 — Actors (5 min).** List every actor for a food-delivery system (think Swiggy/DoorDash). You should find at least **five**. Mark each one as `primary` or `secondary`, and `human` or `system`. Hint: you're missing at least two if you only wrote Customer, Restaurant, and Delivery Partner.

**Step 2 — Use cases (10 min).** For each actor, list what they can do. Verb + noun. Apply the coffee-break test to every one — delete anything that's a *step* rather than a *goal*. You should land on roughly 12–16 ovals total.

**Step 3 — Relationships (5 min).** Find and draw:
- At least **two `<<include>>`** relationships (something mandatory reused across use cases — where does authentication go? where does fee calculation go?).
- At least **one `<<extend>>`** (something genuinely optional — tipping? scheduling for later? adding delivery instructions?).
- At least **one actor generalization** (is there a "kind of" relationship among your human actors?).

Draw the boundary box. Explicitly write, outside the box, the things you are **not** building.

**Step 4 — The payoff (10 min).** Now open your editor and write **only the class skeleton** for `FoodDeliveryService` — public method signatures for every oval, private method signatures for every `<<include>>`, and a constructor that injects every secondary actor as an interface. No method bodies. Just the shape.

**What to produce:** one hand-drawn diagram + one JS file of ~40 lines that is pure signatures. When you're done, look at your class: you designed an API without ever "designing" anything. That's the skill.

---

## Quick reference cheat sheet

- **Actor** = a **role**, not a person. One human can be two actors. A cron job can be an actor.
- **Primary actor** = initiates, wants something (draw LEFT). **Secondary actor** = you call it, it supports you (draw RIGHT).
- **Use case** = one complete **goal** (verb + noun) that leaves the actor better off. Not a step. Not a click.
- **System boundary box** = your scope. Ovals inside = you build it. Stick figures outside = you don't.
- **`<<include>>` = ALWAYS happens.** Base calls it. Arrow points **at the included** case. Base breaks without it.
- **`<<extend>>` = SOMETIMES happens.** Base is oblivious. Arrow points **back at the base**. Base works without it.
- **The test:** *"Can the base succeed without this?"* Yes → extend. No → include.
- **Place Order includes Validate Payment. Apply Coupon extends Place Order.** Memorize this pair.
- **Actor generalization** = hollow triangle, specific → general. Technician *is a* BankStaff with extra powers.
- **Your own DB is NOT an actor.** It's inside the boundary. Stripe *is* an actor.
- **The payoff mapping:** oval → public method | include → private helper | secondary actor → injected dependency | primary actor → role/permission.
- **Open every LLD interview with:** "Let me list the actors and what each can do." You'll have your API in 90 seconds.
- **Don't over-draw.** A list of "Actor → capabilities" beats a 40-oval spaghetti diagram.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [21 — UML Sequence Diagrams](./21-uml-sequence-diagrams.md) — once you know *what* a use case is, sequence diagrams show *how* one plays out step by step |
| **Next** | [23 — Class Relationships](./23-class-relationships.md) — association, aggregation, composition and inheritance between the classes you just derived |
| **Related** | [04 — Functional vs Non-Functional Requirements](./04-functional-vs-non-functional-requirements.md) — use cases ARE your functional requirements, drawn |
| **Related** | [20 — UML Class Diagrams](./20-uml-class-diagrams.md) — the ovals become the methods on the boxes you draw here |
| **Related** | [36 — Facade Pattern](./36-pattern-facade.md) — the single service class you derive from a use case diagram is, structurally, a facade |
