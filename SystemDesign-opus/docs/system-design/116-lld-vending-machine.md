# 116 — Design a Vending Machine
## Category: LLD Case Study

---

## What is this?

A **vending machine** is a small, self-contained state machine you talk to with your hands: you pick a slot, you feed it coins one at a time, and at some magic threshold it drops a can and spits back your change. It never asks you to "start over" — it *remembers* exactly where you are in the transaction, and its reaction to the very same action (inserting a coin) is completely different depending on that memory.

That "reaction depends on remembered position" is the whole reason interviewers love this problem. It is the cleanest possible textbook example of the **State design pattern** (recall [46 — State Pattern](./46-pattern-state.md)) — cleaner even than the ATM, because a vending machine has fewer distractions (no PINs, no bank network, no accounts) and the same core lesson.

---

## Why does it matter?

**Interview angle.** "Design a vending machine" is a top-5 LLD warm-up. The naive candidate writes one giant class with a `status` string and a forest of `if (status === "...")` checks in every method. The strong candidate *notices* that the machine is a finite state machine and reaches for the State pattern. The interviewer is explicitly watching for that jump — it is the single fact that separates a pass from a fail on this question.

**Real-work angle.** You will build "vending machines" constantly without calling them that: a checkout flow (cart → address → payment → confirmation), an onboarding wizard, a document approval pipeline (draft → review → approved → published), a video player (idle → buffering → playing → paused). Every one of them is an object whose legal operations depend on which step it is on. Get comfortable modelling that here, on a machine that sells soda, and you will recognise the shape everywhere.

---

## The core idea — explained simply

### The Choose-Your-Own-Adventure Book Analogy

Think of the machine as a choose-your-own-adventure book. You are always on exactly **one page**. Each page tells you which choices are legal *right now* and what page each choice sends you to. On the "You are in a dark cave" page, "light the torch" turns to page 40; "swim across the lake" isn't even listed, so you can't do it. On a *different* page, "swim across the lake" is suddenly the only option.

The vending machine is that book. Each **state** is a page. Each user action (`selectProduct`, `insertCoin`, `dispense`, `cancel`) is a choice printed on the page. Crucially, the *same choice text* — "insert a coin" — leads somewhere different, or nowhere at all, depending on which page you're on:

- On the **Idle** page, inserting a coin should be rejected: "please select a product first."
- On the **ItemSelected** page, inserting a coin *accumulates* toward the price.
- On the **HasMoney** page, the machine already has enough — extra coins are still accepted but the real next move is "dispense."

The State pattern makes each page a small object that only knows its own choices. You never write "if we're on page 12" logic — you just *turn to* the page object and let it answer.

| Analogy | Technical concept |
|---------|-------------------|
| The book | The `VendingMachine` (the context) |
| The page you're on | The `currentState` reference |
| The choices printed on a page | The methods of one State class |
| A choice not listed on this page | An illegal operation, rejected by the base default |
| "Turn to page 40" | `machine.setState(new HasMoneyState())` |
| Your inventory (coins in hand) | The machine's `insertedAmount` and coin float |
| The final "THE END" page | `DispensingState`, which drops the item and returns to Idle |

---

## Key concepts inside this topic

We follow the standard 7-step LLD method from [111 — LLD Approach Framework](./111-lld-approach-framework.md). For this problem, **State is the star** and everything else is supporting cast.

### 1. Requirements & clarifying questions

**Functional requirements.**

- The machine holds products in **slots**, each slot having a product name, a price, and a quantity.
- The user **selects** a product by code (e.g. `A1`).
- The user **inserts money incrementally** — coins and notes, one at a time. The machine tracks the running total.
- When enough money is inserted, the machine **dispenses** the product and **returns change**.
- The machine must handle the error cases: **insufficient money** (dispense refused), **out of stock**, **exact change unavailable**, and **cancellation** (refund everything inserted).

**Non-functional / constraints (keep it small — this is LLD, not HLD).**

- Single machine, single-threaded, in-memory. No network, no persistence required for the interview.
- Correctness of money handling is paramount: never dispense without payment, never keep the customer's money on a failed sale.

**Clarifying questions you should ask out loud** (this signals seniority):

1. *"What payment types — coins only, notes too, cards?"* We'll model coins/notes as denominations now, and show card as an extension.
2. *"Do you select the product first, then pay, or pay first then select?"* Both flows exist in the real world. We'll pick **select-then-pay** (it makes the state machine cleaner) and note the alternative.
3. *"Can a user buy multiple items in one transaction?"* Assume **one item per transaction** for v1; make it an extension.
4. *"If the machine can't make exact change, do we dispense anyway, refuse, or round?"* We'll **refuse up front** (safest) and discuss alternatives.

### 2. Identify the core objects (nouns)

Underline the nouns in the requirements and you get your classes:

- **`VendingMachine`** — the context object the user interacts with; holds everything.
- **`Product`** — a name and price (the thing being sold).
- **`Slot`** (a.k.a. inventory cell) — a product plus a quantity plus a slot code.
- **`Inventory`** — the collection of slots, addressable by code.
- **`Coin` / `Note`** — enums of denominations with cash values.
- **`CoinFloat`** — the pile of cash the machine holds for making change.
- **The States** — `IdleState`, `ItemSelectedState`, `HasMoneyState`, `DispensingState`, `OutOfStockState`. These are objects too, and they are the heart of the design.

### 3. Behaviours (verbs) + who owns them

| Behaviour | Owner | Notes |
|-----------|-------|-------|
| `selectProduct(code)` | delegated by machine → current **State** | legal only from Idle |
| `insertCoin(coin)` | delegated → current **State** | legal only once an item is selected |
| `dispense()` | delegated → current **State** | legal only once enough money is in |
| `cancel()` | delegated → current **State** | legal any time money is held; refunds |
| `computeChange(amount)` | `ChangeStrategy` | pure algorithm, no state |
| `hasStock(code)` / `deduct(code)` | `Inventory` / `Slot` | data ownership |
| `addToFloat` / `dispenseChange` | `CoinFloat` | the cash reserve |

**The key insight (same as the ATM in [114 — ATM Machine](./114-lld-atm-machine.md)).** The machine's response to `insertCoin` or `dispense` *depends on its current state*. You cannot dispense before enough money is in. Inserting money means nothing before a product is selected. That branching-by-state is *exactly* what the State pattern exists to model — so instead of the machine itself branching, it **delegates every action to its current state object**, and each state object implements only the transitions that are legal for it.

### 4. Class diagram + state-machine diagram

**Class diagram (ASCII):**

```
                    ┌───────────────────────────┐
                    │      VendingMachine        │  ◄── the Context
                    ├───────────────────────────┤
                    │ - inventory : Inventory    │
                    │ - coinFloat : CoinFloat    │
                    │ - changeStrategy           │
                    │ - currentState : VendingState
                    │ - selectedSlot : Slot|null │
                    │ - insertedAmount : number  │
                    ├───────────────────────────┤
                    │ + selectProduct(code)      │─┐ all four just
                    │ + insertCoin(coin)         │ │ delegate to
                    │ + dispense()               │ │ currentState
                    │ + cancel()                 │─┘
                    │ + setState(s)              │
                    └───────────┬───────────────┘
                                │ holds one
                                ▼
                    ┌───────────────────────────┐
                    │  «abstract» VendingState   │
                    ├───────────────────────────┤
                    │ + selectProduct(code)      │  each defaults to
                    │ + insertCoin(coin)         │  "invalid operation"
                    │ + dispense()               │
                    │ + cancel()                 │
                    └───────────┬───────────────┘
          ┌─────────────┬───────┼────────┬──────────────┐
          ▼             ▼       ▼        ▼              ▼
   ┌───────────┐ ┌────────────┐ ┌──────────┐ ┌───────────────┐ ┌──────────────┐
   │ IdleState │ │ItemSelected│ │HasMoney  │ │DispensingState│ │OutOfStockState│
   └───────────┘ └────────────┘ └──────────┘ └───────────────┘ └──────────────┘

   ┌────────────┐   ┌────────┐   ┌────────┐        ┌─────────────────┐
   │ Inventory  │──►│  Slot  │──►│Product │        │  ChangeStrategy  │ (Strategy)
   └────────────┘   └────────┘   └────────┘        │ + change(amt,float)
   ┌────────────┐                                  └─────────────────┘
   │ CoinFloat  │   Coin (enum)   MachineState (enum)
   └────────────┘
```

**State-machine diagram (the important one):**

```
                        selectProduct(inStock)
        ┌──────────────────────────────────────────────┐
        │                                               ▼
  ┌───────────┐                                 ┌───────────────┐
  │           │  selectProduct(outOfStock)      │ ItemSelected  │
  │   Idle    │───────────────┐                 │  (need money) │
  │           │               │                 └───────┬───────┘
  └───────────┘               ▼                   │     │  insertCoin
        ▲               ┌──────────────┐          │     │  (total < price)
        │               │ OutOfStock   │          │     ▼   (loops here)
        │               └──────┬───────┘          │  ┌─────────────────┐
        │        (auto-return) │                  │  │ still ItemSelected
        │◄─────────────────────┘                  │  └─────────────────┘
        │                                          │
        │  cancel  ◄──── refund inserted ────┐     │ insertCoin
        │                                    │     │ (total >= price)
        │                                    │     ▼
        │                              ┌───────────────┐
        │        dispense()            │   HasMoney    │
        │      drop item + change      │ (enough paid) │
        │      ┌───────────────────────┴───────┬───────┘
        │      ▼                                │ insertCoin (extra) → stays
   ┌────────────────┐                           │ cancel → refund → Idle
   │  Dispensing    │                           │
   └───────┬────────┘                           │
           │  auto-return to Idle               │
           └────────────────────────────────────┘
```

Read it as a book of pages: from **Idle** the only legal move is `selectProduct`. If the slot has stock you turn to **ItemSelected**; if it's empty you briefly visit **OutOfStock** and bounce back to Idle. In **ItemSelected** you insert coins; each coin keeps you on the page until the total reaches the price, at which point you turn to **HasMoney**. From **HasMoney**, `dispense` drops the item, makes change, and (via **Dispensing**) returns to Idle. `cancel` from any money-holding page refunds and returns to Idle.

### 5. Full JavaScript implementation (State pattern)

Everything below is one runnable file. Enums are `Object.freeze`; there is an abstract `VendingState` whose defaults reject illegal operations *in one place*; each concrete state implements only its legal transitions.

```javascript
"use strict";

// ─────────────────────────────────────────────────────────────
// 1. ENUMS (Object.freeze — JS has no native enum)
// ─────────────────────────────────────────────────────────────

// Coin/note denominations, keyed by name, valued in cents to avoid
// floating-point money bugs (never store money as 0.1 dollars).
const Coin = Object.freeze({
  PENNY:   { name: "PENNY",   value: 1 },
  NICKEL:  { name: "NICKEL",  value: 5 },
  DIME:    { name: "DIME",    value: 10 },
  QUARTER: { name: "QUARTER", value: 25 },
  DOLLAR:  { name: "DOLLAR",  value: 100 },
});

// A label for the machine's current state — handy for logging/tests.
const MachineState = Object.freeze({
  IDLE: "IDLE",
  ITEM_SELECTED: "ITEM_SELECTED",
  HAS_MONEY: "HAS_MONEY",
  DISPENSING: "DISPENSING",
  OUT_OF_STOCK: "OUT_OF_STOCK",
});

// ─────────────────────────────────────────────────────────────
// 2. DOMAIN OBJECTS: Product, Slot, Inventory, CoinFloat
// ─────────────────────────────────────────────────────────────

class Product {
  constructor(name, priceCents) {
    this.name = name;
    this.price = priceCents; // in cents
  }
}

class Slot {
  constructor(code, product, quantity) {
    this.code = code;
    this.product = product;
    this.quantity = quantity;
  }
  hasStock() { return this.quantity > 0; }
  deduct()   { this.quantity -= 1; }
}

class Inventory {
  constructor() { this.slots = new Map(); }
  addSlot(slot) { this.slots.set(slot.code, slot); }
  getSlot(code) { return this.slots.get(code) || null; }
}

// The pile of coins the machine keeps for making change. Tracks the
// count of each denomination so it can tell you when it *cannot* pay.
class CoinFloat {
  constructor() {
    // Map<coinName, count>
    this.counts = new Map(Object.keys(Coin).map((k) => [k, 0]));
  }
  add(coinName, count = 1) {
    this.counts.set(coinName, this.counts.get(coinName) + count);
  }
  remove(coinName, count = 1) {
    this.counts.set(coinName, this.counts.get(coinName) - count);
  }
  countOf(coinName) { return this.counts.get(coinName); }
  totalValue() {
    let sum = 0;
    for (const [name, cnt] of this.counts) sum += Coin[name].value * cnt;
    return sum;
  }
}

// ─────────────────────────────────────────────────────────────
// 3. STRATEGY: change-making algorithm (a small sub-problem)
// ─────────────────────────────────────────────────────────────

// Greedy change-making. Works for canonical coin systems (1,5,10,25,100)
// where greedy is provably optimal. Returns a Map<coinName,count> or null
// if the float physically cannot make exact change.
class GreedyChangeStrategy {
  change(amountCents, coinFloat) {
    if (amountCents === 0) return new Map();
    const result = new Map();
    let remaining = amountCents;

    // Largest denomination first.
    const denoms = Object.keys(Coin).sort(
      (a, b) => Coin[b].value - Coin[a].value
    );

    for (const name of denoms) {
      const coinValue = Coin[name].value;
      const available = coinFloat.countOf(name);
      // How many of this coin do we want, limited by what we actually have?
      let want = Math.floor(remaining / coinValue);
      const use = Math.min(want, available);
      if (use > 0) {
        result.set(name, use);
        remaining -= use * coinValue;
      }
    }

    // If we couldn't reach exactly zero, the float can't make this change.
    if (remaining !== 0) return null;
    return result;
  }
}

// ─────────────────────────────────────────────────────────────
// 4. STATE PATTERN: abstract base + concrete states
// ─────────────────────────────────────────────────────────────

// The base class implements every operation as "invalid by default".
// This is the single place illegal operations are rejected — concrete
// states override ONLY the operations that are legal for them.
class VendingState {
  constructor(machine) { this.machine = machine; }
  get label() { return "UNKNOWN"; }

  selectProduct(code) { this._invalid("select a product"); }
  insertCoin(coin)    { this._invalid("insert money"); }
  dispense()          { this._invalid("dispense"); }
  cancel()            { this._invalid("cancel"); }

  _invalid(action) {
    console.log(`  [rejected] cannot ${action} while in ${this.label} state`);
  }
}

// IDLE: waiting for a selection. The only legal move is selectProduct.
class IdleState extends VendingState {
  get label() { return MachineState.IDLE; }

  selectProduct(code) {
    const slot = this.machine.inventory.getSlot(code);
    if (!slot) {
      console.log(`  [rejected] no slot with code ${code}`);
      return;
    }
    if (!slot.hasStock()) {
      console.log(`  slot ${code} (${slot.product.name}) is OUT OF STOCK`);
      // Briefly enter OutOfStock, which auto-returns to Idle.
      this.machine.setState(new OutOfStockState(this.machine));
      this.machine.currentState.handle();
      return;
    }
    this.machine.selectedSlot = slot;
    console.log(
      `  selected ${slot.product.name} @ ${fmt(slot.product.price)} — please insert money`
    );
    this.machine.setState(new ItemSelectedState(this.machine));
  }
}

// ITEM_SELECTED: a product is chosen, we are accumulating money.
class ItemSelectedState extends VendingState {
  get label() { return MachineState.ITEM_SELECTED; }

  insertCoin(coin) {
    this.machine.insertedAmount += coin.value;
    this.machine.acceptCoin(coin); // physically goes into the float
    const price = this.machine.selectedSlot.product.price;
    console.log(
      `  inserted ${coin.name} (${fmt(coin.value)}); ` +
      `total = ${fmt(this.machine.insertedAmount)} / ${fmt(price)}`
    );
    if (this.machine.insertedAmount >= price) {
      console.log(`  enough money in — you may dispense`);
      this.machine.setState(new HasMoneyState(this.machine));
    }
  }

  cancel() { this.machine.refundAndReset(); }
}

// HAS_MONEY: enough has been paid; the real next move is dispense.
class HasMoneyState extends VendingState {
  get label() { return MachineState.HAS_MONEY; }

  // Extra coins are still accepted (customer overpays); stay here.
  insertCoin(coin) {
    this.machine.insertedAmount += coin.value;
    this.machine.acceptCoin(coin);
    console.log(
      `  extra ${coin.name}; total = ${fmt(this.machine.insertedAmount)}`
    );
  }

  dispense() {
    const slot = this.machine.selectedSlot;
    const price = slot.product.price;
    const changeDue = this.machine.insertedAmount - price;

    // Can we actually make the change? Decide BEFORE dropping the item.
    const changeCoins = this.machine.changeStrategy.change(
      changeDue,
      this.machine.coinFloat
    );
    if (changeCoins === null) {
      console.log(
        `  [rejected] cannot make ${fmt(changeDue)} in change — refunding`
      );
      this.machine.refundAndReset();
      return;
    }

    this.machine.setState(new DispensingState(this.machine));
    this.machine.currentState.handle(changeCoins, changeDue);
  }

  cancel() { this.machine.refundAndReset(); }
}

// DISPENSING: a transient state that does the physical work then resets.
class DispensingState extends VendingState {
  get label() { return MachineState.DISPENSING; }

  handle(changeCoins, changeDue) {
    const slot = this.machine.selectedSlot;
    slot.deduct();
    console.log(`  >>> DISPENSED ${slot.product.name}`);
    if (changeDue > 0) {
      // Physically remove the change coins from the float.
      for (const [name, cnt] of changeCoins) {
        this.machine.coinFloat.remove(name, cnt);
      }
      console.log(`  >>> RETURNED change ${fmt(changeDue)} as ${describe(changeCoins)}`);
    } else {
      console.log(`  >>> exact payment, no change`);
    }
    this.machine.resetTransaction();
    this.machine.setState(new IdleState(this.machine));
    console.log(`  (ready — back to IDLE)`);
  }
}

// OUT_OF_STOCK: transient; announces and returns to Idle.
class OutOfStockState extends VendingState {
  get label() { return MachineState.OUT_OF_STOCK; }
  handle() {
    this.machine.resetTransaction();
    this.machine.setState(new IdleState(this.machine));
  }
}

// ─────────────────────────────────────────────────────────────
// 5. CONTEXT: VendingMachine — delegates every action to currentState
// ─────────────────────────────────────────────────────────────

class VendingMachine {
  constructor(inventory, coinFloat, changeStrategy = new GreedyChangeStrategy()) {
    this.inventory = inventory;
    this.coinFloat = coinFloat;
    this.changeStrategy = changeStrategy;
    this.selectedSlot = null;
    this.insertedAmount = 0;
    // Coins the CUSTOMER put in during THIS transaction (for refunds).
    this.pendingCoins = [];
    this.currentState = new IdleState(this);
  }

  // ---- public API: pure delegation to the current state ----
  selectProduct(code) { this.currentState.selectProduct(code); }
  insertCoin(coin)    { this.currentState.insertCoin(coin); }
  dispense()          { this.currentState.dispense(); }
  cancel()            { this.currentState.cancel(); }

  // ---- state transition helper ----
  setState(state) { this.currentState = state; }
  get stateLabel() { return this.currentState.label; }

  // ---- money bookkeeping ----
  acceptCoin(coin) {
    // A customer coin becomes part of the float AND is remembered so a
    // cancel can refund the exact coins.
    this.coinFloat.add(coin.name, 1);
    this.pendingCoins.push(coin);
  }

  refundAndReset() {
    const total = this.insertedAmount;
    // Return the exact coins we accepted this transaction.
    for (const coin of this.pendingCoins) this.coinFloat.remove(coin.name, 1);
    console.log(`  >>> REFUNDED ${fmt(total)}`);
    this.resetTransaction();
    this.setState(new IdleState(this));
    console.log(`  (cancelled — back to IDLE)`);
  }

  resetTransaction() {
    this.selectedSlot = null;
    this.insertedAmount = 0;
    this.pendingCoins = [];
  }
}

// ─────────────────────────────────────────────────────────────
// 6. small formatting helpers
// ─────────────────────────────────────────────────────────────

function fmt(cents) { return `$${(cents / 100).toFixed(2)}`; }
function describe(coinMap) {
  return [...coinMap.entries()].map(([n, c]) => `${c}x${n}`).join(", ");
}

// ─────────────────────────────────────────────────────────────
// 7. DEMO — main()
// ─────────────────────────────────────────────────────────────

function main() {
  // Build inventory.
  const inv = new Inventory();
  inv.addSlot(new Slot("A1", new Product("Cola", 125), 2));   // $1.25
  inv.addSlot(new Slot("A2", new Product("Water", 100), 0));  // out of stock
  inv.addSlot(new Slot("B1", new Product("Chips", 75), 5));   // $0.75

  // Seed the change float so the machine can make change.
  const float = new CoinFloat();
  float.add("QUARTER", 10);
  float.add("DIME", 10);
  float.add("NICKEL", 10);

  const machine = new VendingMachine(inv, float);

  console.log("=== SCENARIO 1: buy Cola with overpay + change ===");
  console.log(`state: ${machine.stateLabel}`);
  machine.selectProduct("A1");          // Idle -> ItemSelected
  console.log(`state: ${machine.stateLabel}`);
  machine.insertCoin(Coin.DOLLAR);      // $1.00, still short of $1.25
  machine.insertCoin(Coin.QUARTER);     // $1.25, now HasMoney
  console.log(`state: ${machine.stateLabel}`);
  machine.insertCoin(Coin.QUARTER);     // extra -> $1.50
  machine.dispense();                   // drop Cola, return $0.25 change
  console.log(`state: ${machine.stateLabel}\n`);

  console.log("=== SCENARIO 2: try to buy out-of-stock Water ===");
  machine.selectProduct("A2");          // rejected, stays Idle
  console.log(`state: ${machine.stateLabel}\n`);

  console.log("=== SCENARIO 3: illegal ops are rejected ===");
  machine.dispense();                   // can't dispense from Idle
  machine.insertCoin(Coin.DIME);        // can't insert from Idle
  console.log(`state: ${machine.stateLabel}\n`);

  console.log("=== SCENARIO 4: select Chips then CANCEL for refund ===");
  machine.selectProduct("B1");          // Idle -> ItemSelected ($0.75)
  machine.insertCoin(Coin.QUARTER);     // $0.25
  machine.insertCoin(Coin.DIME);        // $0.35
  machine.cancel();                     // refund $0.35, back to Idle
  console.log(`state: ${machine.stateLabel}\n`);

  console.log("=== SCENARIO 5: buy remaining Cola, exact payment ===");
  machine.selectProduct("A1");
  machine.insertCoin(Coin.DOLLAR);
  machine.insertCoin(Coin.QUARTER);     // exactly $1.25 -> HasMoney
  machine.dispense();                   // exact, no change
  console.log(`state: ${machine.stateLabel}`);
  console.log(`Cola remaining: ${inv.getSlot("A1").quantity}`);
}

main();
```

Run it with `node vending.js`. You'll see the state label change after each action, illegal operations rejected in one place, change computed correctly, an out-of-stock bounce, and a clean cancel-with-refund — proof the machine runs.

### 6. Design patterns used and WHY

**State (the star).** Recall [46 — State Pattern](./46-pattern-state.md). Each state is a class; the machine delegates every action to `currentState`. Contrast this explicitly with the naive approach — a single class where *every* method opens with:

```javascript
// THE ANTI-PATTERN we are avoiding:
insertCoin(coin) {
  switch (this.status) {
    case "IDLE":          /* reject */ break;
    case "ITEM_SELECTED": /* accumulate */ break;
    case "HAS_MONEY":     /* accept extra */ break;
    case "DISPENSING":    /* reject */ break;
    // ...and this SAME switch is copy-pasted into dispense(), cancel(), selectProduct()...
  }
}
```

That giant `switch(this.status)` duplicated across four methods is a maintenance trap: add one new state and you must find and edit *every* switch, and forgetting one is a silent bug. The State pattern turns "one method, many states" into "one state, many methods" — adding `OutOfStockState` touched exactly one new class and left the others untouched. That is the **Open/Closed Principle** in action: open for extension (new state class), closed for modification (existing states unchanged).

**Strategy.** Recall [42 — Strategy Pattern](./42-pattern-strategy.md). Change-making is a self-contained algorithm with interchangeable implementations. `GreedyChangeStrategy` is injected into the machine; you could swap in a `DynamicProgrammingChangeStrategy` (for non-canonical denominations where greedy is wrong) without touching the machine or any state. The machine depends on the *interface* (`change(amount, float)`), not the algorithm.

**Factory (optional).** State creation (`new HasMoneyState(this.machine)`) could be centralised behind a small `StateFactory` so states are pooled/reused rather than re-instantiated. For five states it's overkill, but worth naming as the natural next step if states multiply.

### 7. Extensions the interviewer asks for

**"Add card / mobile payment."** Introduce a `PaymentStrategy` (Strategy again): `CoinPayment`, `CardPayment`, `MobilePayment`. A `CardPayment` skips the incremental-coin loop — on select, the machine goes straight to a `ProcessingPaymentState` that authorises the card, then to `Dispensing`. The State machine absorbs this as one new state; no existing state changes.

**"Add a maintenance / admin mode to refill and collect cash."** Add a `MaintenanceState`. A key/passcode transition from Idle enters it; in that state `selectProduct`/`insertCoin` are all rejected (base default handles that free), and new admin methods (`refill(code, qty)`, `collectCash()`) are legal only there. Because illegal ops are already rejected centrally, the admin state is almost pure addition.

**"Support selecting multiple items."** Change the transaction model: keep a `cart` (list of slots) instead of a single `selectedSlot`, and let `ItemSelected` accept further `selectProduct` calls to add to the cart. `HasMoney` compares against the cart total. The states barely change — the money math moves from one price to a sum.

**"Handle the machine being unable to make exact change."** Already done: `HasMoneyState.dispense()` asks the strategy for change *before* dropping the item and refunds if it returns `null`. The stronger version is to check at **selection time** — a `canMakeChangeFor(price)` guard in `IdleState.selectProduct` that refuses the sale up front with "exact change only" rather than after the customer has paid. That is a one-method addition to a single state.

Each extension is a new state class or a new strategy — never an edit to a giant switch. That is the payoff of modelling the machine as a state machine from the start.

---

## Visual / Diagram description

The two diagrams above (class diagram + state-machine diagram) are the ones to reproduce on a whiteboard. Practise drawing the **state-machine diagram** first and from memory: five boxes (Idle, ItemSelected, HasMoney, Dispensing, OutOfStock) and the labelled arrows between them. If you can draw that and name what each arrow's *trigger* and *guard* are (e.g. `insertCoin [total >= price]` moves ItemSelected → HasMoney), you have understood the problem. The class diagram then falls out mechanically: one context, one abstract state, N concrete states, plus the plain data objects.

---

## Real world examples

### Real vending hardware (representative)

Physical vending machines run embedded firmware that is *literally* a finite state machine — the MDB (Multi-Drop Bus) protocol between the coin mechanism, bill validator, and main controller defines explicit states and legal transitions. The controller refuses to fire a motor (dispense) unless it is in the paid state, exactly as our `HasMoneyState` gates `dispense()`. This is not a metaphor; it is how the real devices are specified.

### Stripe / payment checkout flows (representative)

A payment intent in Stripe moves through documented states — `requires_payment_method → requires_confirmation → processing → succeeded` (or `canceled`). Each API call is only legal in certain states and the server rejects out-of-order calls, which is the server-side twin of our base-class "invalid operation" default. The same State-machine discipline that stops us dispensing before payment stops a charge being confirmed before a method is attached.

### Frontend UI wizards (React state machines)

Libraries like **XState** exist precisely to model UI flows (checkout, onboarding, video players) as explicit finite state machines instead of tangled booleans. Teams reach for it for the same reason we reached for the State pattern here: a `isLoading && !hasError && hasSelected` soup of flags becomes unmaintainable, while named states with named transitions stay readable as they grow.

---

## Trade-offs

**State pattern vs. a status flag + switches:**

| | State pattern | `switch(status)` everywhere |
|---|---|---|
| Adding a new state | One new class, others untouched (OCP) | Edit every method's switch; easy to miss one |
| Where illegal ops live | One place (the base default) | Repeated in every method |
| Readability | Each state reads as a small unit | One class grows unboundedly |
| Cost for tiny machines | More classes/boilerplate | Fewer files, quick to write |
| Debugging transitions | Explicit `setState` calls to trace | Implicit; scattered flag mutations |

**Greedy vs. DP for change-making:**

| | Greedy | Dynamic programming |
|---|---|---|
| Speed | O(denominations) | O(amount × denominations) |
| Correctness | Only for *canonical* coin sets (US/EUR) | Always optimal (fewest coins) |
| Complexity | Trivial | More code |

**Rule of thumb:** Reach for the State pattern the moment an object has 3+ named modes *and* the same method behaves differently per mode — a vending machine, an order lifecycle, a media player. Below that bar, a single status flag is fine and the pattern is over-engineering. For change-making, greedy is correct for real currency; only switch to DP if the interviewer invents weird denominations.

---

## Common interview questions on this topic

### Q1: "Why the State pattern here instead of a status enum and if/else?"

**Hint:** Because the machine's reaction to the *same* action depends on its mode, and there are several modes. A status enum forces the same `switch(status)` into every method; adding a state means editing them all and risking a missed case. State localises each mode's behaviour to one class and each *action* to one method — adding a state is a pure addition (Open/Closed). Point at how `OutOfStockState` slotted in without touching the others.

### Q2: "What happens to the customer's money if they cancel mid-transaction?"

**Hint:** The machine remembers the exact coins accepted this transaction (`pendingCoins`) and refunds them, then resets and returns to Idle. Key design decision: customer coins go into the float immediately *and* are logged for refund — so a cancel returns coins from the float. Never keep the money on a failed or cancelled sale.

### Q3: "The machine has enough money but can't make exact change — what do you do?"

**Hint:** Decide *before* dispensing. `HasMoneyState.dispense()` asks the change strategy for the coins first; if it returns `null` (float can't cover it), refund and abort rather than short-changing the customer or giving the item free. Better still, guard at selection time with "exact change only" so the customer never pays into a dead end.

### Q4: "How would you add card payments?"

**Hint:** Strategy for payment + one new state. `CardPayment` skips the coin-accumulation loop; select transitions to a `ProcessingPaymentState` that authorises, then to Dispensing. Existing states don't change — that's the whole point of having modelled it as a state machine.

### Q5: "Is greedy change-making always correct?"

**Hint:** No — only for *canonical* denomination systems (like US 1/5/10/25/100). For an adversarial set like {1, 3, 4} making 6, greedy gives 4+1+1 (three coins) but optimal is 3+3 (two). Real currency is canonical, so greedy is safe; mention DP as the general-correct fallback and that this is why change-making is behind a Strategy interface.

---

## Practice exercise

### Build the "exact change only" guard and a second product purchase

Take the implementation above and extend it (30–40 min):

1. Add a method `canMakeChangeFor(price)` to `VendingMachine` that, given a slot's price and the largest single coin a customer might pay with, checks whether the float could return change for the worst-case overpayment.
2. Modify `IdleState.selectProduct` so that if the float is nearly empty and cannot guarantee change, it *refuses the selection up front* with an "EXACT CHANGE ONLY — cannot select" message and stays in Idle.
3. Write a demo that drains the float (set it to only 1 quarter), then tries to buy a $0.75 item with a $1.00 coin — prove the guard fires *before* the customer's money is taken.

**Deliverable:** a runnable `node` script whose output shows a normal purchase succeeding, then the same purchase refused once the float is empty, with the machine never having accepted the customer's dollar in the refused case.

---

## Quick reference cheat sheet

- **State pattern = one object, changing behaviour.** The machine delegates every action to its `currentState`; each state is a class implementing only its legal transitions.
- **The states:** Idle → ItemSelected → HasMoney → Dispensing → (back to) Idle, plus OutOfStock and cancel/refund edges.
- **Context object** = `VendingMachine`; it holds inventory, float, selected slot, inserted amount, and the current state.
- **Illegal ops live in ONE place** — the abstract `VendingState` base defaults every action to "invalid"; concrete states override only what's legal.
- **Adding a state is Open/Closed** — one new class, existing states untouched; no giant `switch` to edit.
- **Store money in integer cents**, never floats — avoids `0.1 + 0.2 !== 0.3` money bugs.
- **Change-making is a Strategy** — greedy is correct for canonical currency; swap for DP if denominations are weird.
- **Decide change feasibility BEFORE dispensing** — refund if the float can't make exact change; never short-change or give it free.
- **Cancel refunds the exact coins** accepted this transaction, then returns to Idle.
- **Enums via `Object.freeze`** — `Coin` (with cent values) and `MachineState` labels.
- **Extensions are new states or new strategies** — card payment, admin/maintenance mode, multi-item cart — never edits to a switch.
- **Interview tell:** the moment you say "the response to insertCoin depends on the machine's mode," you should be drawing a state machine.

---

## Connected topics

| Direction | Topic | Why |
|-----------|-------|-----|
| **Previous** | [114 — Design an ATM Machine](./114-lld-atm-machine.md) | The other canonical State-pattern LLD; same lesson, more moving parts (PINs, accounts, bank network) |
| **Next** | [123 — Design a Rate Limiter (LLD)](./123-lld-rate-limiter.md) | Another object-modelling LLD where Strategy (algorithm choice) is central |
| **Related** | [46 — State Pattern](./46-pattern-state.md) | The pattern that *is* this problem — study the pure version alongside |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) | Used here for change-making; the interchangeable-algorithm pattern |
| **Related** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step nouns→verbs→classes→patterns method we followed above |
