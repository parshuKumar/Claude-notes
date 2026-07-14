# 46 — State Pattern

## Category: LLD Patterns

---

## What is this?

The **State pattern** lets an object change its behaviour when its internal state changes — so completely that it *looks like the object changed its class*. Instead of one big object full of `if (this.status === 'X')` checks, you give each state its own class, and the object simply delegates every call to whichever state object it is currently holding.

Think of a **traffic light**. A traffic light isn't a lamp that runs a giant `switch` on its colour every time a car arrives. It *is* red, then it *is* green, then it *is* amber — and each colour knows exactly one thing: what to do now, and which colour comes next. Swap the colour, and the light's entire behaviour changes.

---

## Why does it matter?

Almost every interesting object in a real system has a lifecycle: an `Order` goes CREATED → PAID → SHIPPED → DELIVERED. A `PullRequest` goes OPEN → APPROVED → MERGED. A `Document` goes DRAFT → IN_REVIEW → PUBLISHED. If you model that lifecycle with a status string and scattered `if` checks, three things go wrong:

1. **The transition rules live nowhere.** They're smeared across ten methods. Nobody can answer "can a SHIPPED order be cancelled?" without reading the whole class.
2. **Invalid transitions happen silently.** A missing `else` branch means `dispense()` on an empty machine just... returns `undefined`. No error, no log, a customer loses ₹50.
3. **Adding a state means editing every method.** New `REFUNDED` state? Open all twelve methods and add a branch. That's an Open/Closed Principle violation with a bug waiting in each one.

**Interview angle:** State is a *guaranteed* pattern in LLD rounds. "Design a vending machine", "Design an ATM", "Design an elevator" — the interviewer is explicitly checking whether you reach for State instead of a switch. The follow-up is always: *"How is this different from Strategy?"* (We answer that below.)

**Real-work angle:** Order/payment/subscription lifecycles, connection state machines (TCP, WebSocket reconnection), CI/CD job status, media players, and workflow engines all run on state machines. Getting the transitions into one auditable place is what stops a "shipped but unpaid" bug from reaching production.

---

## The core idea — explained simply

### The Board Game Analogy

Picture a board game where each player holds a **role card** — Thief, Healer, Warrior. The player is the same person all night, but the card they hold decides what the *same button* does:

- Press "Act" holding the **Warrior** card → you attack.
- Press "Act" holding the **Healer** card → you heal.
- Press "Act" holding the **Thief** card → you steal, *and the card tells you to swap to the Warrior card next turn.*

The player doesn't contain a switch on `myRole`. The player just says `this.card.act()`. And crucially — **the card decides which card you hold next.** That last part is the whole pattern.

| Board game | State pattern | Vending machine example |
|---|---|---|
| The player | **Context** — the object with the changing behaviour | `VendingMachine` |
| A role card | **State** — an object encapsulating behaviour for one mode | `HasMoneyState` |
| The pile of possible cards | The set of **concrete states** | Idle / HasMoney / Dispensing / SoldOut |
| "Press Act" | A method on the context | `machine.dispense()` |
| The player calling `card.act()` | **Delegation** — context forwards to current state | `this.state.dispense()` |
| The card telling you to swap cards | **Transition** — the state sets the next state | `machine.setState(new IdleState())` |
| "You can't heal, you're a Thief" | An **illegal transition**, rejected loudly | `throw new IllegalTransitionError(...)` |

The mental flip a beginner has to make: **behaviour lives in the state, not in the context.** The context becomes almost embarrassingly dumb — and that's a feature.

---

## Key concepts inside this topic

### 1. The problem it solves — the painful version first

Here is the vending machine everybody writes on their first attempt. Read it and count the ways it can hurt you.

```javascript
// ============ BAD — the switch-in-every-method vending machine ============
class VendingMachineBad {
  constructor(stock = 3) {
    this.state = 'IDLE';        // 'IDLE' | 'HAS_MONEY' | 'DISPENSING' | 'SOLD_OUT'
    this.stock = stock;
    this.balance = 0;
    this.selected = null;
  }

  insertCoin(amount) {
    switch (this.state) {
      case 'IDLE':
        this.balance += amount;
        this.state = 'HAS_MONEY';
        return `Inserted ₹${amount}`;
      case 'HAS_MONEY':
        this.balance += amount;
        return `Balance ₹${this.balance}`;
      case 'SOLD_OUT':
        return 'Sold out';          // BUG: silently swallows the coin. No refund.
      // BUG: 'DISPENSING' not handled -> falls through, returns undefined.
    }
  }

  selectItem(item) {
    switch (this.state) {
      case 'HAS_MONEY':
        this.selected = item;
        this.state = 'DISPENSING';
        return `Selected ${item}`;
      case 'IDLE':
        return 'Insert money first';
      // BUG: 'DISPENSING' and 'SOLD_OUT' missing -> undefined again.
    }
  }

  dispense() {
    switch (this.state) {
      case 'DISPENSING':
        this.stock -= 1;
        this.balance = 0;
        this.state = this.stock === 0 ? 'SOLD_OUT' : 'IDLE';
        return `Dispensed ${this.selected}`;
      default:
        return undefined;          // BUG: caller cannot tell "nothing happened" from success.
    }
  }
}

const bad = new VendingMachineBad();
bad.dispense();      // undefined — a free "nothing". No error, no log, no alarm.
bad.insertCoin(50);
bad.selectItem('Coke');
bad.insertCoin(50);  // undefined — the coin is GONE. Money lost, silently.
```

Four things are broken here, and they're the four things State fixes:

| Symptom | Root cause |
|---|---|
| Coins vanish, `undefined` returned | Missing `case` branches fail **silently** |
| "Can I insert a coin while dispensing?" is unanswerable | Transition rules are scattered across N methods |
| Adding `MAINTENANCE` means editing 3 methods | Every state is coupled to every method (OCP violation) |
| The `switch` in `insertCoin` will drift from the one in `dispense` | Duplicated state knowledge, no single source of truth |

### 2. The structure — participants

```
Context   → the object clients talk to. Holds a reference to ONE state. Delegates everything.
State     → the interface (in JS: a base class) every state must implement.
ConcreteState → one class per state. Implements the behaviour AND decides the next state.
```

The key rule that separates State from every other pattern: **a concrete state may call `context.setState(...)`.** States know about each other, and they drive the transitions themselves.

```
┌──────────────────────────────┐
│        VendingMachine        │  ◀── the CONTEXT
│  (context)                   │
│                              │
│  - state : State             │────────────────┐
│  - stock : number            │                │ delegates
│  - balance: number           │                │ every call
│                              │                │
│  + insertCoin(amount)        │                │
│  + selectItem(item)          │                ▼
│  + dispense()                │       ┌─────────────────────────┐
│  + refund()                  │       │      State (base)       │
│  + setState(s)  ◀────────────┼───┐   │  + insertCoin(m, amt)   │
└──────────────────────────────┘   │   │  + selectItem(m, item)  │
                                   │   │  + dispense(m)          │
        states call setState()     │   │  + refund(m)            │
        to move the machine ───────┘   │  + get name()           │
                                       └───────────▲─────────────┘
                                                   │ extends
                    ┌──────────────┬───────────────┼───────────────┐
                    │              │               │               │
            ┌───────┴──────┐ ┌─────┴───────┐ ┌─────┴────────┐ ┌────┴────────┐
            │  IdleState   │ │HasMoneyState│ │DispensingSt. │ │ SoldOutState│
            │              │ │             │ │              │ │             │
            │ insertCoin ✔ │ │ insertCoin ✔│ │ insertCoin ✘ │ │ insertCoin ✘│
            │ selectItem ✘ │ │ selectItem ✔│ │ selectItem ✘ │ │ selectItem ✘│
            │ dispense   ✘ │ │ dispense   ✘│ │ dispense   ✔ │ │ dispense   ✘│
            │ refund     ✘ │ │ refund     ✔│ │ refund     ✘ │ │ refund     ✘│
            └──────────────┘ └─────────────┘ └──────────────┘ └─────────────┘
                      ✔ = legal here   ✘ = throws IllegalTransitionError
```

### 3. The state transition table

Before writing a single line of a state machine, **write this table.** It is the design. In an interview, drawing this earns you more points than the code does.

| Current state ↓ / Event → | `insertCoin` | `selectItem` | `dispense` | `refund` |
|---|---|---|---|---|
| **Idle** | → HasMoney | ✘ illegal | ✘ illegal | ✘ illegal |
| **HasMoney** | → HasMoney (add) | → Dispensing *(if balance ≥ price)* | ✘ illegal | → Idle (coins back) |
| **Dispensing** | ✘ illegal | ✘ illegal | → Idle, or → SoldOut if stock hits 0 | ✘ illegal |
| **SoldOut** | ✘ illegal (reject coin) | ✘ illegal | ✘ illegal | ✘ illegal |

Every cell is either a destination or an explicit rejection. **There are no blanks.** Blanks are what `undefined` was in the bad version.

### 4. Full JavaScript implementation

```javascript
// ============================================================================
// 46 — State Pattern: Vending Machine
// Run with: node vending-machine.js
// ============================================================================

// A dedicated error type so callers can catch *this* specifically, and so the
// failure is loud instead of a silently-returned `undefined`.
class IllegalTransitionError extends Error {
  constructor(stateName, action) {
    super(`Illegal action "${action}" while machine is in state "${stateName}"`);
    this.name = 'IllegalTransitionError';
    this.stateName = stateName;
    this.action = action;
  }
}

// ---------------------------------------------------------------------------
// The State "interface". JS has no interfaces, so we use a base class whose
// methods reject by default. This is the single place where illegal
// transitions are handled — every state inherits the rejection for free and
// only OVERRIDES the actions it actually allows.
// ---------------------------------------------------------------------------
class VendingState {
  get name() { return this.constructor.name; }

  insertCoin(_machine, _amount) { this._reject('insertCoin'); }
  selectItem(_machine, _item)   { this._reject('selectItem'); }
  dispense(_machine)            { this._reject('dispense'); }
  refund(_machine)              { this._reject('refund'); }

  _reject(action) {
    throw new IllegalTransitionError(this.name, action);
  }
}

// ---------------------------------------------------------------------------
// Concrete states. Each one knows: (a) what to do, (b) where to go next.
// ---------------------------------------------------------------------------

class IdleState extends VendingState {
  insertCoin(machine, amount) {
    machine.balance += amount;
    machine.setState(new HasMoneyState());   // Idle --insertCoin--> HasMoney
    return `Accepted ₹${amount}. Balance: ₹${machine.balance}`;
  }
  // selectItem / dispense / refund inherit the rejection. That is the point:
  // we did not have to write "you can't do that yet" four times.
}

class HasMoneyState extends VendingState {
  insertCoin(machine, amount) {
    machine.balance += amount;
    return `Balance: ₹${machine.balance}`;   // self-loop: stays in HasMoney
  }

  selectItem(machine, item) {
    const price = machine.priceOf(item);
    if (price === undefined) throw new Error(`Unknown item: ${item}`);
    if (machine.stockOf(item) === 0) throw new Error(`${item} is out of stock`);
    // A guard, not a transition: insufficient funds keeps us in HasMoney.
    if (machine.balance < price) {
      return `Need ₹${price - machine.balance} more for ${item}`;
    }
    machine.selected = item;
    machine.setState(new DispensingState());  // HasMoney --selectItem--> Dispensing
    return `Selected ${item} (₹${price}). Press dispense.`;
  }

  refund(machine) {
    const returned = machine.balance;
    machine.balance = 0;
    machine.setState(new IdleState());        // HasMoney --refund--> Idle
    return `Refunded ₹${returned}`;
  }
}

class DispensingState extends VendingState {
  // Deliberately NOT accepting coins here. In the bad version this was a
  // missing switch case that ate your money. Here it throws.
  dispense(machine) {
    const item = machine.selected;
    const price = machine.priceOf(item);
    const change = machine.balance - price;

    machine.inventory.set(item, machine.stockOf(item) - 1);
    machine.balance = 0;
    machine.selected = null;

    // The state decides the NEXT state, based on the world it just changed.
    if (machine.totalStock() === 0) {
      machine.setState(new SoldOutState());   // Dispensing --dispense--> SoldOut
    } else {
      machine.setState(new IdleState());      // Dispensing --dispense--> Idle
    }
    return `Dispensed ${item}${change > 0 ? `. Change: ₹${change}` : ''}`;
  }
}

class SoldOutState extends VendingState {
  // Every action rejects — inherited from the base. We only override to give
  // a friendlier message for the most likely mistake.
  insertCoin(machine) {
    this._reject('insertCoin');               // coin is refused, never swallowed
  }

  // Not a customer action — an operator action, so it lives on the machine,
  // but the state is what allows the machine back to life.
  restock(machine, item, qty) {
    machine.inventory.set(item, machine.stockOf(item) + qty);
    machine.setState(new IdleState());        // SoldOut --restock--> Idle
    return `Restocked ${item} x${qty}`;
  }
}

// ---------------------------------------------------------------------------
// The Context. Notice how thin it is: no `if`, no `switch`, no state logic.
// It holds data + a current state, and forwards every request.
// ---------------------------------------------------------------------------
class VendingMachine {
  constructor(catalog) {
    this.prices = new Map(Object.entries(catalog.prices));   // item -> price
    this.inventory = new Map(Object.entries(catalog.stock)); // item -> qty
    this.balance = 0;
    this.selected = null;
    this.history = [];
    this.state = this.totalStock() === 0 ? new SoldOutState() : new IdleState();
  }

  // --- the ONLY place a transition is recorded. One line to audit/log/trace. ---
  setState(next) {
    this.history.push(`${this.state.name} -> ${next.name}`);
    this.state = next;
  }

  // --- public API: pure delegation ---
  insertCoin(amount) { return this.state.insertCoin(this, amount); }
  selectItem(item)   { return this.state.selectItem(this, item); }
  dispense()         { return this.state.dispense(this); }
  refund()           { return this.state.refund(this); }

  restock(item, qty) {
    if (typeof this.state.restock !== 'function') {
      throw new IllegalTransitionError(this.state.name, 'restock');
    }
    return this.state.restock(this, item, qty);
  }

  // --- helpers the states use ---
  priceOf(item)  { return this.prices.get(item); }
  stockOf(item)  { return this.inventory.get(item) ?? 0; }
  totalStock()   { return [...this.inventory.values()].reduce((a, b) => a + b, 0); }
  get stateName() { return this.state.name; }
}

// ---------------------------------------------------------------------------
// Demo
// ---------------------------------------------------------------------------
function main() {
  const m = new VendingMachine({
    prices: { Coke: 40, Chips: 30 },
    stock:  { Coke: 1, Chips: 1 },
  });

  const say = (label, fn) => {
    try {
      console.log(`[${m.stateName.padEnd(15)}] ${label} -> ${fn()}`);
    } catch (err) {
      console.log(`[${m.stateName.padEnd(15)}] ${label} -> ✘ ${err.name}: ${err.message}`);
    }
  };

  say('dispense()',        () => m.dispense());        // ✘ illegal from Idle — LOUD
  say('insertCoin(20)',    () => m.insertCoin(20));    // Idle -> HasMoney
  say('selectItem(Coke)',  () => m.selectItem('Coke')); // guard: needs ₹20 more
  say('insertCoin(50)',    () => m.insertCoin(50));    // stays HasMoney, balance ₹70
  say('selectItem(Coke)',  () => m.selectItem('Coke')); // HasMoney -> Dispensing
  say('insertCoin(10)',    () => m.insertCoin(10));    // ✘ illegal — coin refused, not eaten
  say('dispense()',        () => m.dispense());        // Dispensing -> Idle, ₹30 change
  say('insertCoin(30)',    () => m.insertCoin(30));
  say('selectItem(Chips)', () => m.selectItem('Chips'));
  say('dispense()',        () => m.dispense());        // last item -> SoldOut
  say('insertCoin(50)',    () => m.insertCoin(50));    // ✘ illegal — machine is empty
  say('restock(Coke, 5)',  () => m.restock('Coke', 5)); // SoldOut -> Idle

  console.log('\nTransition log:');
  m.history.forEach((h, i) => console.log(`  ${i + 1}. ${h}`));
}

main();
```

### 5. Why this is strictly better than the switch

**a) Illegal transitions fail loudly, in exactly one place.**
The base class's `_reject()` is the *only* place that decides what "you can't do that" means. Want to log every rejection to Datadog? One edit. Want to return a Result object instead of throwing? One edit. In the switch version that logic would need to appear in a `default:` branch of every method — and you'd forget one.

**b) Adding a state doesn't touch the existing states (Open/Closed Principle).**
Recall from [17 — Open/Closed Principle] that classes should be open for extension, closed for modification. Adding a `MaintenanceState`:

```javascript
class MaintenanceState extends VendingState {
  // Everything rejects by inheritance. Only the exit is defined.
  endMaintenance(machine) {
    machine.setState(new IdleState());
    return 'Back in service';
  }
}
```

That's it. **Zero lines changed** in `IdleState`, `HasMoneyState`, `DispensingState`, `SoldOutState`, or `VendingMachine`. Compare that to the switch version, where you'd add a `case 'MAINTENANCE':` to all four methods and hope you didn't miss one.

**c) Each state is independently testable.** `new HasMoneyState().refund(fakeMachine)` is a unit test with no setup. You cannot unit-test "the machine when it's in HAS_MONEY" in the switch version without driving it through a sequence of calls first.

**d) The transition log is free.** Because `setState()` is a chokepoint, you get an audit trail (`this.history`) for nothing. That single line is worth its weight in gold when a customer says "the machine took my money".

### 6. When NOT to use it / how it's abused

| Situation | Verdict |
|---|---|
| 2 states, 1 method (`on`/`off` toggle) | **Don't.** A boolean is fine. Four classes to model a light switch is parody. |
| States exist but no behaviour differs — only data | **Don't.** If `status` is just a label you render, it's an enum, not a State pattern. |
| Behaviour differs but there are **no transitions between behaviours** | That's **Strategy**, not State. See below. |
| 4+ states × 3+ actions, with real transition rules | **Yes.** This is the sweet spot. |
| 20+ states, transitions driven by config | Consider a **state machine library** (XState) or a **transition table** driving a generic engine — 20 hand-written classes becomes its own mess. |

**The classic abuse:** creating a fresh state object on every transition (`new IdleState()`) when states are stateless. It works, but allocates. If your states hold no instance data, make them **singletons** — one shared instance each. That's the Flyweight idea from [40 — Flyweight Pattern]:

```javascript
const STATES = Object.freeze({
  IDLE:       new IdleState(),
  HAS_MONEY:  new HasMoneyState(),
  DISPENSING: new DispensingState(),
  SOLD_OUT:   new SoldOutState(),
});
// then: machine.setState(STATES.IDLE);
```

**The other abuse:** letting the *context* decide transitions ("machine.js has a big `if` picking the next state"). If you do that, you've rebuilt the switch — you just hid it. The whole point is that **the state decides.**

### 7. Related patterns and how they differ — the #1 interview follow-up

**State vs Strategy — structurally identical, opposite intent.**

Both have a context delegating to a swappable object behind a common interface. If you drew the UML for both, you could not tell them apart. The difference is *who swaps, why, and whether the objects know about each other.*

| | **Strategy** ([42](./42-pattern-strategy.md)) | **State** (this doc) |
|---|---|---|
| **Who chooses the object?** | The **client / caller** ("sort with quicksort") | The **states themselves**, from the inside |
| **Do the objects know about each other?** | No. `QuickSort` has never heard of `MergeSort`. | **Yes.** `HasMoneyState` explicitly constructs `DispensingState`. |
| **Does the object change during a single operation?** | No — you pick one and run it | Yes — that's the *entire point* |
| **Intent** | "Do the same job a different way" | "Be a different thing depending on where you are in a lifecycle" |
| **A wrong swap is...** | Just a different (valid) result | An **illegal transition** — a bug worth throwing on |
| **Typical trigger** | Config, user preference, A/B test | An event arriving (coin inserted, payment confirmed) |

The one-liner to say in an interview: **"Same structure, different intent. Strategies are interchangeable and mutually ignorant; states are sequenced and know their successors. If the objects trigger their own replacement, it's State."**

**Other neighbours:**

| Pattern | Relationship to State |
|---|---|
| [42 — Strategy](./42-pattern-strategy.md) | Same shape. See table above. The single most common follow-up question. |
| [43 — Command](./43-pattern-command.md) | Commands are the *events* that drive a state machine. Command + State = an event-sourced workflow. |
| [41 — Observer](./41-pattern-observer.md) | Pairs beautifully: `setState()` emits `stateChanged`, and listeners (UI, audit log, metrics) react. |
| [29 — Singleton](./29-pattern-singleton.md) / [40 — Flyweight](./40-pattern-flyweight.md) | Stateless state objects should be shared singletons, not re-allocated. |
| [45 — Template Method](./45-pattern-template-method.md) | A base `State` class with shared `_reject()` + hooks *is* a template method — the two compose naturally. |

---

## Visual / Diagram description

### Diagram 1: The state machine (circles + labelled arrows)

This is the diagram to draw on the whiteboard **before** writing code. Nodes are states, arrows are events.

```
                          restock
        ┌───────────────────────────────────────────────┐
        │                                               │
        │                                               │
        ▼                insertCoin                     │
   ╭─────────╮  ────────────────────────────▶  ╭──────────────╮
   │  IDLE   │                                 │  HAS_MONEY   │◀─┐ insertCoin
   │         │  ◀────────────────────────────  │              │──┘  (self-loop:
   ╰─────────╯            refund               ╰──────────────╯      add balance)
        ▲                                              │
        │                                              │ selectItem
        │ dispense                                     │ [balance >= price]
        │ [stock > 0]                                  ▼
        │                                      ╭──────────────╮
        └──────────────────────────────────────│  DISPENSING  │
                                               │              │
                                               ╰──────────────╯
                                                       │
                                                       │ dispense
                                                       │ [stock == 0]
                                                       ▼
                                               ╭──────────────╮
                                               │   SOLD_OUT   │
                                               │              │
                                               ╰──────────────╯
                                                       │
                                                       └── restock ──▶ (back to IDLE, top arrow)

   Legend:  ╭───╮ = state       ──event──▶ = transition      [guard] = condition
   Any arrow NOT drawn here = IllegalTransitionError. There is no "silently do nothing".
```

**What it shows:** four states, seven legal transitions, two guards. Every event/state pair not on this picture throws. When someone asks "can you insert a coin while it's dispensing?" you point at the picture: there is no arrow, so no.

### Diagram 2: The call flow — who calls whom

```
 CLIENT                CONTEXT                     CURRENT STATE
   │                (VendingMachine)              (HasMoneyState)
   │                       │                             │
   │  selectItem('Coke')   │                             │
   ├──────────────────────▶│                             │
   │                       │  state.selectItem(this,'Coke')
   │                       ├────────────────────────────▶│
   │                       │                             │ check price/stock (guard)
   │                       │                             │ machine.selected = 'Coke'
   │                       │                             │
   │                       │   setState(new Dispensing)  │  ◀── THE STATE
   │                       │◀────────────────────────────┤      TRIGGERS THE
   │                       │ history.push(...)           │      TRANSITION
   │                       │ this.state = DispensingState│
   │                       │                             │
   │   "Selected Coke"     │◀────────────────────────────┤
   │◀──────────────────────┤                             │
   │                       │                             │
   │                    (the machine is now a DIFFERENT object,
   │                     behaviourally — same instance, new class of behaviour)
```

**What it shows:** the context never decides anything. It forwards the call, and the state calls back into `setState()`. That back-arrow — state → context → new state — is the signature of the State pattern, and it is *exactly* the arrow that Strategy does not have.

---

## Real world examples

### 1. Stripe — PaymentIntent lifecycle

Stripe's `PaymentIntent` object is a public, documented state machine: `requires_payment_method` → `requires_confirmation` → `requires_action` → `processing` → `succeeded` (or `canceled`). Every API call is an event against that machine, and calling `confirm()` on an intent in the wrong status returns a **`400` with an explicit error**, not a silent no-op — precisely the "fail loudly, in one place" property. The status is exposed to you *because* the transition rules are the contract: your integration code is written against the arrows, not against Stripe's internals.

### 2. TCP — the canonical state machine

Every TCP connection is a state machine with CLOSED, LISTEN, SYN_SENT, SYN_RECEIVED, ESTABLISHED, FIN_WAIT_1/2, TIME_WAIT, and more. Receiving a SYN in ESTABLISHED does something completely different from receiving a SYN in LISTEN — same event, different behaviour, because the *state* changed. RFC 793 literally ships the transition diagram. Kernel implementations use a state table (a table-driven variant of this pattern) rather than one class per state, which is what you do when the state count gets large.

### 3. GitHub — Pull Request lifecycle

A PR moves through OPEN → (review requested) → APPROVED / CHANGES_REQUESTED → MERGED or CLOSED, with a reopened path back to OPEN. The UI buttons literally change with the state: the merge button is disabled until required checks pass and reviews approve. Conceptually, "which buttons are available" *is* the state object's method set — a state that doesn't implement `merge()` renders no merge button.

---

## Trade-offs

| Pros | What it buys you |
|---|---|
| **Single Responsibility** | Each state class holds the behaviour of exactly one mode |
| **Open/Closed** | New states are new files; existing states are untouched |
| **Illegal transitions are impossible to ignore** | The base class rejects by default — you must opt *in* to legality |
| **The transition graph is discoverable** | Grep for `setState(` and you have the entire machine |
| **Trivially testable** | Each state is a tiny class with no setup |
| **Free audit trail** | `setState()` is one chokepoint for logging/metrics/events |

| Cons | The price you pay |
|---|---|
| **Class explosion** | 6 states = 6 files. For a 2-state toggle this is absurd. |
| **Indirection** | Reading `machine.dispense()` no longer tells you what happens — you must find the current state |
| **States must know each other** | `HasMoneyState` imports `DispensingState`. Coupling is real (though intentional). Circular imports in Node need care. |
| **Shared data lives in the context** | States are behaviour-only; anything shared must be pushed onto the context, which can bloat it |
| **Allocation churn** | Naively `new`-ing a state per transition. Fix: singleton states. |

**Rule of thumb:** Reach for State when you can honestly write a transition table with **3+ states and 2+ actions**, and at least one cell is an *illegal* transition you care about. Below that, an enum plus a couple of `if`s is not technical debt — it's proportionate. Above ~10 states, stop hand-writing classes and move to a declarative transition table or XState.

---

## Common interview questions on this topic

### Q1: "State and Strategy have identical UML. How do you tell them apart?"
**Hint:** By intent and by who drives the swap. In Strategy, the **client** picks the object, and strategies never reference each other — `QuickSort` doesn't know `MergeSort` exists. In State, the **state objects pick their own successor** by calling `context.setState(next)`, so they must know each other. Also: strategies are interchangeable at any time (any strategy is always valid); states are sequenced (most transitions are *illegal*, and that's the value).

### Q2: "Where do you store the transition rules?"
**Hint:** Inside each concrete state — a state owns the outgoing arrows from itself. That way adding a state is additive. The alternative is a central transition table (`{ IDLE: { insertCoin: 'HAS_MONEY' } }`) driving a generic engine, which is better past ~10 states because you can *see* the whole machine at once. Name the trade-off: per-class = extensible but scattered; table = centralised but you edit one shared object.

### Q3: "What happens on an invalid action, like `dispense()` in IDLE?"
**Hint:** Throw a typed error (`IllegalTransitionError`) from the base `State` class. Say why: default-reject means new states are *safe by default* — you must explicitly opt into each legal action. If the domain prefers not throwing (e.g. an HTTP layer), return a `Result` object — but keep the decision in one place. Never return `undefined`; that's how coins disappear.

### Q4: "Should state objects be singletons or created fresh?"
**Hint:** If states hold no instance data (the usual case — data lives on the context), share one frozen instance per state. Saves allocation and makes `machine.state === STATES.IDLE` a valid identity check. Create fresh instances only if a state carries per-visit data (e.g. a retry counter for that entry into the state).

### Q5: "How would you persist a state machine to a database?"
**Hint:** Persist the **state name** (a string/enum column), not the object. On load, map the name back to a state instance via a registry (`{ IDLE: STATES.IDLE, ... }`). Bonus points: also persist the transition log as an append-only table — that's event sourcing, and it lets you replay/audit *why* an order ended up cancelled.

### Q6: "Two threads/requests act on the same machine at once. What breaks?"
**Hint:** Read-modify-write on `this.state` is a race — two `selectItem` calls can both see `HAS_MONEY`. In Node the event loop protects you *within* a synchronous tick, but not across `await` boundaries. Fixes: keep the whole transition synchronous, or use an optimistic-concurrency version column (`UPDATE orders SET state=? WHERE id=? AND state=?` and check rows-affected). Related: [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md).

---

## Practice exercise

### Build an `Order` state machine (30–40 min)

Model an e-commerce order with these states: **CREATED, PAID, SHIPPED, DELIVERED, CANCELLED, REFUNDED.**

**Step 1 — Design first (10 min, no code).** Draw the transition table. Events: `pay()`, `ship(trackingId)`, `deliver()`, `cancel(reason)`, `refund()`. Rules:
- An order can be cancelled from CREATED or PAID, but **not** once SHIPPED.
- Refund is only legal from DELIVERED or CANCELLED-after-PAID.
- DELIVERED and REFUNDED are terminal (no outgoing arrows except REFUNDED from DELIVERED).
Fill in **every** cell: destination or ✘.

**Step 2 — Implement (20 min).** Write it in the shape above:
- `OrderState` base class with `_reject(action)` throwing `IllegalTransitionError`.
- One class per state, overriding only its legal actions.
- An `Order` context with `setState()`, a `history` array, and pure-delegation methods.
- States as frozen singletons in a `STATES` object.

**Step 3 — Prove it.** Write a `main()` that:
1. Runs the happy path CREATED → PAID → SHIPPED → DELIVERED → REFUNDED and prints the history.
2. Tries `order.cancel()` on a SHIPPED order and catches the `IllegalTransitionError`.
3. Tries `order.pay()` twice and shows the second one throwing.

**Step 4 — The interviewer's twist.** Now add a **RETURN_REQUESTED** state between DELIVERED and REFUNDED. Count how many *existing* lines you had to change. If the answer isn't "zero, plus one arrow in DeliveredState", your states are doing too much.

**Deliverable:** one runnable `order.js` file plus your transition table as a comment at the top.

---

## Quick reference cheat sheet

- **State pattern** = an object's behaviour changes with its internal state; each state is its own class.
- **Context** holds a `state` reference and **delegates every method** to it — the context has zero `if`s about state.
- **The state decides the next state.** `state.action(context)` calls `context.setState(next)`. This back-arrow is the pattern's signature.
- **Base class rejects by default.** Concrete states override *only* their legal actions — illegal transitions throw for free.
- **Draw the transition table before you code.** Rows = states, columns = events, cells = destination or ✘. No blanks.
- **`setState()` is a chokepoint** — free logging, metrics, event emission, and audit trail.
- **Adding a state touches no existing state** — that's the Open/Closed win, and it's what you say out loud in the interview.
- **State vs Strategy:** same UML, opposite intent. Strategy = client picks, strategies are mutually ignorant. State = states pick their successor, so they know each other.
- **Make stateless states singletons** — don't `new` a state on every transition.
- **Persist the state *name***, not the object; rehydrate through a registry.
- **Don't use it for a boolean toggle.** 3+ states and 2+ actions, with real illegal cells, is the bar.
- **Past ~10 states**, switch to a declarative transition table or a library like XState.
- **Terminal states** (DELIVERED, REFUNDED) are states whose every action rejects — they cost you nothing.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [45 — Template Method Pattern](./45-pattern-template-method.md) — the base `State` class with a shared `_reject()` is a template method in action |
| **Next** | [47 — Chain of Responsibility Pattern](./47-pattern-chain-of-responsibility.md) — the other "objects hand off to each other" behavioural pattern |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) — identical structure, opposite intent; the #1 interview follow-up to this topic |
| **Related** | [116 — Design a Vending Machine](./116-lld-vending-machine.md) — this exact machine, taken to a full LLD answer |
| **Related** | [114 — Design an ATM Machine](./114-lld-atm-machine.md) — Idle → CardInserted → PinEntered → TransactionSelected; State pattern is the expected answer |
