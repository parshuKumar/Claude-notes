# 114 — Design an ATM Machine
## Category: LLD Case Study

---

## What is this?

An ATM (Automated Teller Machine) is the box in the wall that lets you insert a card, type a PIN, and walk away with cash — or check your balance, deposit money, or print a mini-statement. The machine itself holds no account data; it is a **terminal** that talks to your bank over a network and physically dispenses or accepts cash.

The reason this is *the* canonical Low-Level Design (LLD) interview problem is that an ATM's behaviour changes completely depending on where you are in the flow. Trying to enter a PIN before inserting a card is meaningless. Trying to withdraw before the PIN is validated is a security hole. The machine is a **state machine**, and modelling it cleanly is the whole exercise. This makes it the textbook application of the **State design pattern** (recall [46 — State Pattern](./46-pattern-state.md)).

---

## Why does it matter?

**Interview angle.** The ATM is a favourite because a weak candidate writes one giant `handle()` method full of `if (status === "CARD_INSERTED") ... else if (status === "PIN_ENTERED") ...`. A strong candidate *recognises the state machine* and reaches for the State pattern. The interviewer is watching for exactly that leap: can you see that "the same action means different things in different states" is a modelling problem with a known solution?

**Real-work angle.** State machines are everywhere in backend engineering: order lifecycles (`placed → paid → shipped → delivered`), payment flows, document approval, connection handshakes, CI pipelines. If you cannot model a state machine cleanly, these systems rot into tangled boolean flags (`isPaid`, `isShipped`, `isCancelled`) where illegal combinations (`isShipped && !isPaid`) silently become possible. The ATM teaches you to make illegal transitions *impossible* by construction, not by discipline.

What breaks without it: bugs where a user withdraws before authenticating, states you forgot to handle, and a `switch` statement in every method that you must edit in five places every time you add a feature.

---

## The core idea — explained simply

### Analogy: a board game where the rules change per square

Picture a board game. On the "Start" square, the only legal move is *roll to begin*. On a "Jail" square, the only legal move is *pay bail* or *skip a turn*. Handing someone the dice on the Jail square does nothing — that action isn't legal *here*.

Now imagine writing the rulebook two ways:

- **Bad rulebook:** one enormous page titled "What to do", and every rule starts with "If you are on Start and you roll... but if you are in Jail and you roll... but if you are on Go and you roll...". Every action re-checks which square you're on. Add a new square and you must edit *every* action.
- **Good rulebook:** one small card *per square*. The Jail card lists only the moves legal in jail. The Start card lists only the moves legal at start. To find what's legal, you look at the card for the square you're standing on. Add a new square? Write one new card. You never touch the others.

That "one card per square" rulebook is the **State pattern**. Each square is a **state object**; each state object knows only the transitions legal *from itself*.

| Board game | ATM (technical) |
|---|---|
| The square you're standing on | The ATM's `currentState` |
| A rule card per square | A `State` class per state (`IdleState`, `HasCardState`, ...) |
| The moves listed on a card | The methods on that state class |
| "That move isn't legal here" | The base class's default "invalid operation" |
| Moving to a new square | `atm.setState(nextState)` |
| The player holding the board | The `ATM` **context** object |

The magic: the `ATM` object never asks "what state am I in?". It just forwards every action — `insertCard()`, `enterPin()`, `dispenseCash()` — to `currentState`, and the current state either does the right thing and transitions, or rejects the action. All the "which actions are legal where" logic lives in the state classes, one place each.

---

## Key concepts inside this topic

We follow the 7-step LLD method from [111 — LLD Approach Framework](./111-lld-approach-framework.md): clarify requirements, find nouns, find verbs, draw the diagram, implement, name the patterns, handle extensions.

### 1. Requirements & clarifying questions

**The happy path:** insert card → enter PIN → select an operation (withdraw / deposit / check balance / mini-statement) → the machine dispenses cash or accepts a deposit → eject the card.

**Functional requirements**
- Insert a card and read it.
- Validate a PIN against the bank. Wrong PIN gets a limited number of retries (say 3); exceeding the limit **blocks the card**.
- Operations: withdraw cash, deposit cash, check balance, print mini-statement.
- Withdrawal must reject **insufficient account balance** and **insufficient cash in the ATM**.
- The user can **cancel at any step** and get the card back.
- Eject the card at the end (or on error/cancel).

**Non-functional requirements**
- Correctness and safety: never dispense without a validated PIN and sufficient funds; never lose track of cash.
- The ATM is a thin client; the **bank is the source of truth** for balances.

**Clarifying questions to ask the interviewer** (this scores points — it shows you scope before coding):
- Single currency or **multi-currency**? (Assume single, INR/USD, for now.)
- Is there a **per-transaction or daily withdrawal limit**? (Assume a per-transaction cap; note where a daily limit would plug in.)
- Do we **print a physical receipt**? (Assume we can log/print a mini-statement; hardware glossed.)
- One bank or a shared network (Visa/Plus)? (Assume the ATM calls one `BankService` interface; a real network is behind it.)
- Do we model the physical hardware (card reader, keypad, cash bins)? (We gloss `Screen`/`Keypad` and model the `CashDispenser` properly because the bill-breakdown is an interesting sub-problem.)

### 2. Identify the core objects (nouns)

Underline the nouns in the requirements:

| Object | Responsibility |
|---|---|
| `ATM` | The **context**. Holds the current state, the cash dispenser, the bank connection, and the currently inserted card. Forwards user actions to the current state. |
| `Card` | The physical card: card number, expiry, cardholder name. |
| `Account` | A bank account: number, balance, transaction history. Lives on the bank side. |
| `BankService` | The interface the ATM uses to talk to the bank: `validatePin`, `getBalance`, `debit`, `credit`. **Mockable** — in tests it's an in-memory fake; in production it's a network call. |
| `CashDispenser` | Manages the physical cash: how many bills of each denomination, whether an amount `canDispense`, and the **bill breakdown** when it does. |
| `Screen` / `Keypad` | I/O hardware. Glossed — we just `console.log` and pass values as method arguments. |
| `Transaction` | A record of one operation, with subtypes `Withdrawal`, `Deposit`, `BalanceInquiry`, `MiniStatement`. |
| **The States** | `ATMState` (base) and `IdleState`, `HasCardState`, `AuthenticatedState`, `OutOfServiceState`. The heart of the design. |

### 3. Behaviours (verbs) + who owns them

| Behaviour | Owner | Notes |
|---|---|---|
| `insertCard(card)` | current State | Only legal from `IdleState`. |
| `enterPin(pin)` | current State | Only legal from `HasCardState`; validates via `BankService`, counts retries. |
| `selectOperation(op, amount)` | current State | Only legal from `AuthenticatedState`. |
| `dispenseCash(amount)` | current State → `CashDispenser` | Checks account balance AND machine cash. |
| `ejectCard()` | current State | Legal from most states; returns to `IdleState`. |
| `cancel()` | current State | Abort and eject from anywhere. |
| `setState(next)` | `ATM` | The context switches its own current state. |
| `canDispense/dispense` | `CashDispenser` | The denomination breakdown sub-problem. |
| `validatePin/debit/credit` | `BankService` | The bank is the source of truth. |

**The crucial modelling insight.** Every one of these behaviours is *state-dependent*. `insertCard` means nothing if a card is already in. `enterPin` is nonsense before a card is inserted. `dispenseCash` before PIN validation is a security breach. This "the same action behaves differently — or is illegal — depending on the current state" is the exact shape the **State pattern** was invented for. Instead of every method beginning with a `switch (this.status)`, each *state* implements only the actions legal from it, and inherits a polite rejection for the rest.

### 4. Class diagram & state-machine diagram

**Class diagram** (the object structure, including the state hierarchy):

```
┌──────────────────────────────┐         ┌─────────────────────┐
│            ATM               │────────▶│    CashDispenser    │
│ (the Context)                │  has-a  │ denominations: {}   │
│  - currentState: ATMState    │         │ + canDispense(amt)  │
│  - currentCard: Card         │         │ + dispense(amt): {} │
│  - bank: BankService         │         └─────────────────────┘
│  - dispenser: CashDispenser  │
│  - pinAttempts: number       │         ┌─────────────────────┐
│  + setState(s)               │────────▶│    BankService      │
│  + insertCard(card)  ──┐     │  uses   │ + validatePin()     │
│  + enterPin(pin)       │     │         │ + getBalance()      │
│  + selectOperation()   │     │         │ + debit() / credit()│
│  + dispenseCash(amt)   │     │         └─────────────────────┘
│  + ejectCard()         │     │                   │ manages
│  + cancel()            │     │                   ▼
└────────┬───────────────┘     │         ┌─────────────────────┐
         │ delegates all       │         │      Account        │
         │ actions to          │         │ number, balance,    │
         ▼                     │         │ history[]           │
┌──────────────────────────────┐         └─────────────────────┘
│      ATMState (abstract)     │
│  #atm: ATM                   │◀─────────────┐  Card
│  + insertCard()   default →  │              │  number, pin,
│  + enterPin()     "invalid   │              │  expiry, holder
│  + selectOperation() operation│
│  + dispenseCash()  in this   │
│  + ejectCard()     state"    │
└──────────────────────────────┘
        ▲        ▲        ▲        ▲
        │        │        │        │   (inheritance: each overrides
  ┌─────┴──┐ ┌───┴────┐ ┌─┴──────┐ ┌──┴──────────┐  only the LEGAL
  │Idle    │ │HasCard │ │Authent-│ │OutOfService │   transitions)
  │State   │ │State   │ │icated  │ │State        │
  │        │ │        │ │State   │ │             │
  │insert  │ │enterPin│ │select  │ │(rejects all │
  │Card ✔  │ │eject ✔ │ │Op ✔    │ │ but eject)  │
  └────────┘ └────────┘ └────────┘ └─────────────┘
```

**State-machine diagram** (this is essential — it *is* the design). Circles are states, arrows are transitions labelled with the action that causes them:

```
                        insertCard(card)
        ┌────────┐  ─────────────────────▶  ┌───────────┐
        │        │                          │           │
   ┌───▶│  IDLE  │                          │  HAS_CARD │
   │    │        │  ◀─────────────────────  │           │
   │    └────────┘      ejectCard() /        └─────┬─────┘
   │        ▲           cancel()                   │
   │        │                                      │ enterPin(pin)
   │        │ ejectCard()                          │
   │        │ (after txn                    ┌──────┴───────┐
   │        │  or cancel)                   │              │
   │    ┌───┴──────────┐             wrong  │  correct     │  wrong PIN
   │    │              │             PIN &  ▼              ▼  (attempts
   │    │ AUTHENTICATED│◀────────  attempts  ok       [validate]  exhausted)
   │    │              │            remain      ┌──────────┐      │
   │    │ selectOp()   │◀───────────────────────│ (stays   │      │
   │    │  loops here  │       (retry PIN)      │ HAS_CARD)│      │
   │    └──────┬───────┘                        └──────────┘      │
   │           │                                                  ▼
   │           │ withdraw / deposit / balance          ┌──────────────────┐
   │           │ (transaction runs, then back          │  OUT_OF_SERVICE  │
   │           │  to AUTHENTICATED for more)           │  (card blocked / │
   │           ▼                                        │   no cash)       │
   └───── ejectCard() ◀──────────────────────────────  └──────────────────┘
                                    ejectCard() (retain/return card)
```

Read it as: you always start `IDLE`. `insertCard` moves you to `HAS_CARD`. A correct PIN moves you to `AUTHENTICATED`, where you can run many operations in a loop. Any error (too many wrong PINs, no cash) or a `cancel`/`ejectCard` sends you back toward `IDLE`. A blocked card or an empty cash bin drops the machine into `OUT_OF_SERVICE`.

### 5. Full JavaScript implementation (using the State pattern)

Real, complete, runnable Node.js. Copy it into `atm.js` and run `node atm.js`.

```javascript
"use strict";

// ---------- Enums (frozen so they can't be mutated) ----------
const TransactionType = Object.freeze({
  WITHDRAW: "WITHDRAW",
  DEPOSIT: "DEPOSIT",
  BALANCE: "BALANCE",
  MINI_STATEMENT: "MINI_STATEMENT",
});

const ATMStatus = Object.freeze({
  IDLE: "IDLE",
  HAS_CARD: "HAS_CARD",
  AUTHENTICATED: "AUTHENTICATED",
  OUT_OF_SERVICE: "OUT_OF_SERVICE",
});

const MAX_PIN_ATTEMPTS = 3;

// ---------- Domain objects ----------
class Card {
  constructor(cardNumber, pin, accountNumber, holder) {
    this.cardNumber = cardNumber;
    this.pin = pin; // in reality the PIN never lives on the card; simplified
    this.accountNumber = accountNumber;
    this.holder = holder;
    this.blocked = false;
  }
}

class Account {
  constructor(accountNumber, balance) {
    this.accountNumber = accountNumber;
    this.balance = balance;
    this.history = []; // list of {type, amount, balanceAfter, at}
  }
}

// ---------- BankService: the source of truth (mockable) ----------
// In production these are network calls. Here it's an in-memory fake so
// the whole thing runs offline and is trivial to unit-test.
class BankService {
  constructor() {
    this.accounts = new Map(); // accountNumber -> Account
  }

  addAccount(account) {
    this.accounts.set(account.accountNumber, account);
  }

  validatePin(card, pin) {
    return !card.blocked && card.pin === pin;
  }

  getBalance(accountNumber) {
    return this.accounts.get(accountNumber).balance;
  }

  // returns true on success, false if insufficient funds
  debit(accountNumber, amount) {
    const acc = this.accounts.get(accountNumber);
    if (acc.balance < amount) return false;
    acc.balance -= amount;
    acc.history.push({ type: TransactionType.WITHDRAW, amount, balanceAfter: acc.balance, at: Date.now() });
    return true;
  }

  credit(accountNumber, amount) {
    const acc = this.accounts.get(accountNumber);
    acc.balance += amount;
    acc.history.push({ type: TransactionType.DEPOSIT, amount, balanceAfter: acc.balance, at: Date.now() });
    return true;
  }

  getMiniStatement(accountNumber, n = 5) {
    return this.accounts.get(accountNumber).history.slice(-n);
  }
}

// ---------- CashDispenser: the denomination sub-problem ----------
class CashDispenser {
  // denominations: { 2000: count, 500: count, 100: count }
  constructor(denominations) {
    this.denominations = denominations;
  }

  totalCash() {
    return Object.entries(this.denominations)
      .reduce((sum, [note, count]) => sum + Number(note) * count, 0);
  }

  // Greedy: can we make `amount` from available bills, biggest-first?
  // Returns the bill breakdown {note: count} or null if impossible.
  planDispense(amount) {
    if (amount <= 0 || amount % 100 !== 0) return null; // multiples of 100 only
    let remaining = amount;
    const plan = {};
    const notes = Object.keys(this.denominations).map(Number).sort((a, b) => b - a);
    for (const note of notes) {
      if (remaining <= 0) break;
      const needed = Math.floor(remaining / note);
      const use = Math.min(needed, this.denominations[note]);
      if (use > 0) {
        plan[note] = use;
        remaining -= note * use;
      }
    }
    return remaining === 0 ? plan : null; // null = can't make exact amount
  }

  canDispense(amount) {
    return this.planDispense(amount) !== null;
  }

  // physically remove the bills and return the breakdown
  dispense(amount) {
    const plan = this.planDispense(amount);
    if (!plan) throw new Error("Cannot dispense " + amount);
    for (const [note, count] of Object.entries(plan)) {
      this.denominations[note] -= count;
    }
    return plan;
  }
}

// ---------- State pattern: base class ----------
// Every action defaults to "not allowed here". Concrete states override
// ONLY the transitions that are legal from them. That single default is
// how every illegal transition is rejected in one place.
class ATMState {
  constructor(atm) {
    this.atm = atm;
  }
  get name() { return "ATMState"; }

  _reject(action) {
    console.log(`  [rejected] Cannot '${action}' while in ${this.name}.`);
  }

  insertCard(card) { this._reject("insertCard"); }
  enterPin(pin) { this._reject("enterPin"); }
  selectOperation(type, amount) { this._reject("selectOperation"); }
  dispenseCash(amount) { this._reject("dispenseCash"); }
  ejectCard() { this._reject("ejectCard"); }
  cancel() { this._reject("cancel"); }
}

// ---------- Concrete states ----------
class IdleState extends ATMState {
  get name() { return ATMStatus.IDLE; }

  insertCard(card) {
    if (card.blocked) {
      console.log("  Card is blocked. Cannot use this machine.");
      this.atm.setState(new OutOfServiceState(this.atm));
      return;
    }
    this.atm.currentCard = card;
    this.atm.pinAttempts = 0;
    console.log(`  Card inserted for ${card.holder}.`);
    this.atm.setState(new HasCardState(this.atm));
  }
}

class HasCardState extends ATMState {
  get name() { return ATMStatus.HAS_CARD; }

  enterPin(pin) {
    const card = this.atm.currentCard;
    if (this.atm.bank.validatePin(card, pin)) {
      console.log("  PIN correct. Authenticated.");
      this.atm.setState(new AuthenticatedState(this.atm));
      return;
    }
    this.atm.pinAttempts += 1;
    const left = MAX_PIN_ATTEMPTS - this.atm.pinAttempts;
    if (left <= 0) {
      card.blocked = true;
      console.log("  Wrong PIN. Attempts exhausted — CARD BLOCKED.");
      this.atm.setState(new OutOfServiceState(this.atm));
      return;
    }
    console.log(`  Wrong PIN. ${left} attempt(s) left.`);
    // stay in HasCardState so the user can retry
  }

  ejectCard() { this.atm._ejectAndReset("Card ejected."); }
  cancel() { this.atm._ejectAndReset("Cancelled. Card ejected."); }
}

class AuthenticatedState extends ATMState {
  get name() { return ATMStatus.AUTHENTICATED; }

  selectOperation(type, amount = 0) {
    const acct = this.atm.currentCard.accountNumber;
    switch (type) {
      case TransactionType.BALANCE:
        console.log(`  Balance: ${this.atm.bank.getBalance(acct)}`);
        return;
      case TransactionType.MINI_STATEMENT:
        console.log("  Mini-statement (last 5):");
        for (const t of this.atm.bank.getMiniStatement(acct)) {
          console.log(`    ${t.type} ${t.amount} -> bal ${t.balanceAfter}`);
        }
        return;
      case TransactionType.DEPOSIT:
        this.atm.bank.credit(acct, amount);
        console.log(`  Deposited ${amount}. New balance: ${this.atm.bank.getBalance(acct)}`);
        return;
      case TransactionType.WITHDRAW:
        this.dispenseCash(amount);
        return;
      default:
        console.log("  Unknown operation.");
    }
  }

  // withdrawal: check machine cash, then account funds, then dispense
  dispenseCash(amount) {
    const acct = this.atm.currentCard.accountNumber;
    if (!this.atm.dispenser.canDispense(amount)) {
      console.log(`  ATM cannot dispense ${amount} (insufficient/awkward cash). Try another amount.`);
      return;
    }
    if (this.atm.bank.getBalance(acct) < amount) {
      console.log(`  Insufficient account balance for ${amount}.`);
      return;
    }
    // debit first (source of truth), then release the cash
    const ok = this.atm.bank.debit(acct, amount);
    if (!ok) { console.log("  Debit failed."); return; }
    const bills = this.atm.dispenser.dispense(amount);
    const breakdown = Object.entries(bills).map(([n, c]) => `${c}x${n}`).join(", ");
    console.log(`  Dispensed ${amount} as [${breakdown}]. New balance: ${this.atm.bank.getBalance(acct)}`);
  }

  ejectCard() { this.atm._ejectAndReset("Transaction done. Card ejected."); }
  cancel() { this.atm._ejectAndReset("Cancelled. Card ejected."); }
}

class OutOfServiceState extends ATMState {
  get name() { return ATMStatus.OUT_OF_SERVICE; }
  // The only thing you can do is get your card back.
  ejectCard() { this.atm._ejectAndReset("Machine out of service. Card returned."); }
  cancel() { this.atm._ejectAndReset("Machine out of service. Card returned."); }
}

// ---------- ATM: the Context ----------
class ATM {
  constructor(bank, dispenser) {
    this.bank = bank;
    this.dispenser = dispenser;
    this.currentCard = null;
    this.pinAttempts = 0;
    this.currentState = new IdleState(this); // start idle
  }

  setState(state) {
    console.log(`  >> state: ${this.currentState.name} -> ${state.name}`);
    this.currentState = state;
  }

  // helper used by several states to return the card and reset to idle
  _ejectAndReset(message) {
    console.log(`  ${message}`);
    this.currentCard = null;
    this.pinAttempts = 0;
    this.setState(new IdleState(this));
  }

  // Public API — the ATM never checks its own state; it just delegates.
  // This is the payoff of the State pattern: no switch(status) anywhere.
  insertCard(card) { this.currentState.insertCard(card); }
  enterPin(pin) { this.currentState.enterPin(pin); }
  selectOperation(type, amount) { this.currentState.selectOperation(type, amount); }
  dispenseCash(amount) { this.currentState.dispenseCash(amount); }
  ejectCard() { this.currentState.ejectCard(); }
  cancel() { this.currentState.cancel(); }
}

// ---------- Demo main() ----------
function main() {
  // Bank with one account holding 5000.
  const bank = new BankService();
  bank.addAccount(new Account("ACC-1", 5000));
  const card = new Card("CARD-1", "1234", "ACC-1", "Asha");

  // ATM stocked with cash.
  const dispenser = new CashDispenser({ 2000: 5, 500: 10, 100: 20 });
  const atm = new ATM(bank, dispenser);

  console.log("\n=== Scenario 1: successful withdrawal ===");
  atm.insertCard(card);
  atm.enterPin("1234");
  atm.selectOperation(TransactionType.BALANCE);
  atm.selectOperation(TransactionType.WITHDRAW, 2600); // 1x2000 + 1x500 + 1x100
  atm.selectOperation(TransactionType.DEPOSIT, 1000);
  atm.selectOperation(TransactionType.MINI_STATEMENT);
  atm.ejectCard();

  console.log("\n=== Scenario 2: illegal action + wrong PIN then block ===");
  atm.enterPin("1234"); // rejected: no card yet
  const card2 = new Card("CARD-2", "9999", "ACC-1", "Asha");
  atm.insertCard(card2);
  atm.enterPin("0000"); // wrong (2 left)
  atm.enterPin("1111"); // wrong (1 left)
  atm.enterPin("2222"); // wrong -> blocked, OUT_OF_SERVICE
  atm.ejectCard();

  console.log("\n=== Scenario 3: insufficient funds ===");
  const card3 = new Card("CARD-3", "1234", "ACC-1", "Asha");
  atm.insertCard(card3);
  atm.enterPin("1234");
  atm.selectOperation(TransactionType.WITHDRAW, 999999); // too much
  atm.selectOperation(TransactionType.WITHDRAW, 250);    // not a multiple of 100 -> can't dispense
  atm.ejectCard();
}

main();
```

Running it prints the state transitions (`IDLE -> HAS_CARD -> AUTHENTICATED -> ...`), proving the machine only permits legal actions and rejects the rest.

### 6. Design patterns used and WHY

**State (the star).** The `ATM` is the *context*; `ATMState` and its subclasses are the *states*. Recall [46 — State Pattern](./46-pattern-state.md): the pattern lets an object alter its behaviour when its internal state changes, so it *appears to change its class*. Without it, every method — `insertCard`, `enterPin`, `dispenseCash` — would begin with `switch (this.status) { case IDLE: ... case HAS_CARD: ... }`. That means the "what's legal in HAS_CARD" logic is smeared across five different methods, and adding a state forces you to edit all five. With State, each state class holds all its own rules, and adding `OutOfServiceState` touched *nothing else* — that is the **Open/Closed Principle** (open to extension, closed to modification) made concrete.

**Command (optional, for transactions).** Each operation can become a command object — `WithdrawCommand`, `DepositCommand`, `TransferCommand` — with a common `execute()` (recall [43 — Command Pattern](./43-pattern-command.md)). This is how real ATMs support *undo/reversal*, *queuing* transactions when the network is down, and *logging* every command for audit. In our demo we inlined the operations in `AuthenticatedState.selectOperation` for brevity, but the natural next refactor is `AuthenticatedState` holding a `Map<TransactionType, Command>`.

**Strategy (optional, for cash dispensing).** The bill-breakdown algorithm is a *strategy*. Today `CashDispenser.planDispense` is greedy (biggest note first). If a bank wanted "prefer smaller notes" or an optimal breakdown, you'd swap in a different `DispenseStrategy` without touching the states. The State pattern and Strategy look identical structurally; the difference is intent — State encodes *lifecycle transitions*, Strategy encodes *interchangeable algorithms*.

### 7. Extensions the interviewer will ask for

**"Add a deposit / transfer operation."** Deposit already exists via `TransactionType.DEPOSIT` handled in `AuthenticatedState`. A transfer is one more `case` (or, better, one more `Command` object). Notice we did **not** add a state — deposits and transfers are legal from exactly the same place (authenticated), so they're new *operations*, not new *states*. Knowing when something is a new state vs. a new operation is the senior insight.

**"Add a daily withdrawal limit."** Track `withdrawnToday` per account in `BankService`. In `AuthenticatedState.dispenseCash`, add a guard: `if (withdrawnToday + amount > DAILY_LIMIT) reject`. State machine unchanged.

**"Add multi-currency."** Give `CashDispenser` a per-currency set of bins (`{ INR: {...}, USD: {...} }`) and thread a `currency` argument through `selectOperation`/`dispenseCash`. The states don't change; only the dispenser and the withdrawal command grow a currency parameter.

**"The dispenser must give the minimum number of bills."** This is the interesting sub-problem. Our `planDispense` is **greedy** — take as many of the biggest note as possible, then the next. Greedy gives the minimum bill count for "canonical" denomination systems (like `2000/500/100`), and it's O(number of denominations). But greedy can *fail* even when a solution exists: with notes `{ 400: 1, 300: 3 }` and a target of `900`, greedy takes one 400, then can't make 500 from 300s and reports failure, though `3×300` works. The robust fix is a **dynamic-programming coin-change** (like the DP problems you've studied) that always finds the fewest bills if any combination works — at the cost of an O(amount × denominations) table. In an interview, implement greedy, then *name* its failure case and say "for arbitrary denominations I'd switch to DP coin-change." That contrast is exactly what they want to hear.

The takeaway: **State absorbs new stages of the flow, Command absorbs new operations, Strategy absorbs new algorithms.** Each extension slots into exactly one of the three without disturbing the others.

---

## Visual / Diagram description

The two diagrams in Key Concept 4 are the ones to memorise. On a whiteboard:

1. Draw the **state-machine** first (four circles: `IDLE`, `HAS_CARD`, `AUTHENTICATED`, `OUT_OF_SERVICE`) and label the arrows with actions. This proves you *see* the state machine before writing a line of code — the single highest-signal move in this interview.
2. Then draw the **class diagram**: `ATM` (context) at the centre, `has-a` arrows to `CashDispenser` and `BankService`, and an inheritance tree from `ATMState` down to the four concrete states. Point out that `ATM`'s public methods are one-liners that delegate to `currentState`.

Say out loud: "The ATM holds a `currentState` reference. Every public call forwards to it. Each state overrides only its legal transitions and inherits a rejection for the rest. Transitions happen by the state calling `atm.setState(next)`." If you can narrate that, you have communicated the entire design.

---

## Real world examples

### Diebold Nixdorf / NCR (ATM manufacturers)

Real ATMs run a stack (commonly on the **CEN/XFS** standard — eXtensions for Financial Services) where the terminal application is explicitly a **finite state machine**. Vendors ship state-flow editors where each screen/step is a state with defined transitions and timeouts. Our `IdleState → HasCardState → AuthenticatedState` mirrors that model directly. *(Representative — the internal class design is proprietary, but the FSM approach is industry-standard and documented in XFS.)*

### Payment networks (Visa / Mastercard authorization)

When the ATM asks the bank to `debit`, that request travels through an authorization network as an ISO 8583 message. The ATM is the thin client; the **issuer bank is the source of truth** for the balance — exactly why our `BankService` owns balances and the ATM never does. The two-step "debit at the bank, then release cash" ordering in `dispenseCash` reflects the real principle that you must authorize before you dispense.

### Backend order/payment lifecycles (any e-commerce)

The same pattern models an order: `CREATED → PAID → SHIPPED → DELIVERED`, with `CANCELLED`/`REFUNDED` as error transitions. Teams that model this with State objects (or a library like XState) avoid the classic bug where an order is `shipped` but `unpaid`. The ATM is the teaching version of every backend lifecycle you'll build.

---

## Trade-offs

**State pattern vs. a giant switch:**

| | State pattern | `switch(status)` in every method |
|---|---|---|
| Add a new state | One new class, touch nothing else (OCP) | Edit every method that switches |
| Where "legal in X" lives | One place per state | Smeared across all methods |
| Illegal transitions | Rejected once, in the base class | Easy to forget a case |
| Readability | Each state reads top-to-bottom | Nested conditionals everywhere |
| Number of classes | More classes (can feel heavy) | Fewer classes, one big file |
| Best for | Rich lifecycles with many actions | 2 states, 1 action (overkill otherwise) |

**Debit-then-dispense vs. dispense-then-debit:**

| Order | Pro | Con |
|---|---|---|
| Debit first (ours) | Never gives cash you didn't deduct | If the dispenser jams after debit, you owe a reversal |
| Dispense first | Cash and debit feel atomic to the user | Risk of dispensing without deducting on a crash |

Real ATMs debit first and run a **reconciliation/reversal** if the physical dispense fails — a Command with an `undo()` is the clean way to model that.

**The sweet spot:** reach for the State pattern the moment you have 3+ states each responding to multiple actions differently. For a two-state toggle, a boolean is fine — don't gold-plate.

---

## Common interview questions on this topic

### Q1: "Why the State pattern here instead of an enum and a switch?"
**Hint:** Because behaviour is state-dependent across *many* methods. A switch duplicates the state check in every method and violates OCP — adding a state means editing them all. State localises each state's rules in one class; adding `OutOfServiceState` touches nothing existing. Mention that illegal transitions get rejected once, in the base class default.

### Q2: "How do you prevent withdrawing before PIN validation?"
**Hint:** You don't *check* for it — you make it *impossible*. `HasCardState` simply doesn't implement `dispenseCash`, so it inherits the base "invalid operation" rejection. Only `AuthenticatedState` implements withdrawal. The type/state system enforces the rule; there's no `if (authenticated)` to forget.

### Q3: "The ATM has cash but can't dispense the exact amount — how?"
**Hint:** The denomination breakdown. If the machine has only ₹2000 notes and you ask for ₹2500, `planDispense` returns null. Explain greedy (biggest-first), show its failure case on non-canonical denominations (`{400, 300}` for 900), and say you'd use DP coin-change for a guaranteed-minimum breakdown with arbitrary notes.

### Q4: "Where does the daily withdrawal limit live?"
**Hint:** Not in the state machine — it's a business rule on the account. Track `withdrawnToday` in `BankService`, guard inside `AuthenticatedState.dispenseCash`. Emphasise it's a new *rule*, not a new *state* — states are about lifecycle stages, not policy.

### Q5: "How would you support undo / reversal of a transaction?"
**Hint:** Model each operation as a **Command** (recall [43 — Command](./43-pattern-command.md)) with `execute()` and `undo()`. If the physical dispense fails after the bank debit, call `undo()` to credit the account back. Commands also let you queue transactions during network outages and log them for audit.

---

## Practice exercise

**Add a `TRANSFER` operation using the Command pattern (30–40 min).**

Starting from the code above:
1. Create a `Command` base class with `execute()` and `undo()`.
2. Implement `WithdrawCommand`, `DepositCommand`, and a new `TransferCommand` (debit source account, credit destination account).
3. Refactor `AuthenticatedState.selectOperation` to look the command up in a `Map<TransactionType, Command>` and call `execute()`.
4. In `WithdrawCommand.execute()`, if `dispenser.dispense()` throws *after* the debit, call `undo()` to reverse the debit — and print the reversal.
5. Extend `main()` with a successful transfer and a failed dispense that triggers a reversal.

**Produce:** an updated `atm.js` that runs, showing a transfer and a reversal, with the state machine unchanged. If you had to add or edit a *state* to do this, re-read Key Concept 7 — you shouldn't have to.

---

## Quick reference cheat sheet

- **ATM = state machine.** Recognising this and reaching for the State pattern is the whole interview.
- **Context** = the `ATM` object; it holds `currentState` and delegates every action to it — no `switch(status)` anywhere.
- **State classes** each implement only their *legal* transitions; the base class rejects everything else by default.
- **Illegal-by-construction:** you can't withdraw in `HasCardState` because the method simply isn't there — no runtime check to forget.
- **The four states:** `IdleState`, `HasCardState`, `AuthenticatedState`, `OutOfServiceState`.
- **Transitions** happen via `atm.setState(next)`, called from inside a state.
- **BankService is the source of truth** for balances; the ATM is a thin client. Mock it for tests.
- **Wrong PIN** → count attempts → block the card at the limit → `OutOfServiceState`.
- **Withdrawal guards:** machine can dispense the amount AND account has the funds; **debit before dispense**.
- **CashDispenser** solves the bill-breakdown; **greedy** works for canonical notes, **DP coin-change** for arbitrary ones.
- **New stage → new State. New operation → new Command. New algorithm → new Strategy.**
- **OCP payoff:** adding `OutOfServiceState` touched no existing state class.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [46 — State Pattern](./46-pattern-state.md) | The pattern this whole design is built on — read it first if the State idea is fuzzy. |
| **Next** | [116 — Vending Machine](./116-lld-vending-machine.md) | The other canonical State-pattern problem (`NoCoin → HasCoin → Dispensing`); practise the same modelling. |
| **Related** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step method (requirements → nouns → verbs → diagram → code → patterns → extensions) we followed here. |
| **Related** | [43 — Command Pattern](./43-pattern-command.md) | Models each ATM operation as an executable/undoable command — the clean way to add transfer, reversal, and audit. |
| **Related** | [112 — Parking Lot](./112-lld-parking-lot.md) | Another classic LLD case study; contrast its Strategy/Factory focus with this doc's State focus. |
