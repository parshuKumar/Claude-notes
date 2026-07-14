# 16 — SOLID — Liskov Substitution Principle (LSP)

## Category: LLD Fundamentals

---

## What is this?

The Liskov Substitution Principle says: **if code works with a base type, it must keep working when you hand it any subtype — without knowing or caring which subtype it got.**

Think of a **power socket**. Anything with the right plug must work in it: a lamp, a laptop charger, a kettle. If you invent a "SmartPlug" that fits the socket but explodes when you flip the switch, you haven't built a valid plug — you've built a trap. The socket (the caller) shouldn't have to inspect what's plugged in before turning the power on.

LSP is the rule that turns inheritance from a *syntax feature* into a *promise*: `extends` means "I can stand in for my parent everywhere, silently."

---

## Why does it matter?

Break LSP and your code rots in a very specific, very recognizable way: **the caller starts asking what type it's holding.**

```javascript
// The smell. Every LSP violation eventually produces this.
if (shape instanceof Square) shape.setSide(10);
else if (shape instanceof Rectangle) { shape.setWidth(10); shape.setHeight(5); }
```

Once that `instanceof` chain appears, you have lost every benefit inheritance was supposed to give you:

- **Polymorphism is dead.** You can no longer treat the collection uniformly.
- **Recall from [15 — Open/Closed Principle](./15-solid-open-closed.md)** that adding a new type should require *zero* edits to existing code. With an `instanceof` chain, every new subtype forces you to edit every caller. LSP violations *cause* OCP violations.
- **Tests lie.** Your test suite passes against the base class and blows up in production against the subclass.

**Interview angle:** LSP is the most misunderstood SOLID letter, so interviewers love it. If you can explain *why* `Square extends Rectangle` is wrong — and it's not "because a square isn't a rectangle in maths" — you instantly sound senior.

**Real-work angle:** Every time you write a plugin system, a repository interface, a payment-gateway abstraction, or a `BaseController`, you are making an LSP promise. Half the bugs in "why does this only break for the S3 storage backend?" are LSP violations.

---

## The core idea — explained simply

### The Rental Car Analogy

You run a car-rental company. Your contract with the customer says:

> *"You will be given **a car**. A car has a steering wheel, an accelerator, and a brake. Turn the key and it moves forward. Press the brake and it stops."*

The customer books a car. They don't know or care which one they get — a Honda, a Toyota, a Tesla. That's the whole point: **the contract is with `Car`, not with `Honda`.**

Now imagine you hand them one of these:

| Vehicle you hand over | What happens | LSP verdict |
|---|---|---|
| **Toyota** — steers, accelerates, brakes | Customer drives away happy | ✅ Valid subtype |
| **Tesla** — same controls, electric internals | Customer drives away happy, never notices | ✅ Valid subtype. *Internals may differ; behaviour may not.* |
| **Racing car** — needs a helmet + racing licence before it will start | Customer can't use it. You **strengthened the precondition** | ❌ Violation |
| **Toy car** — has all the controls but goes 3 km/h | Contract said "moves forward"… technically true, practically useless. You **weakened the postcondition** | ❌ Violation |
| **Rental with a hidden fuel-cutoff at 50 km** — dies mid-journey | You **broke an invariant** the customer relied on | ❌ Violation |
| **Car that throws "EngineLockedException" on Sundays** | New failure mode the base contract never mentioned. **New exception** | ❌ Violation |

Notice what the customer never has to do: **open the bonnet and check the brand before driving.** The moment your rental contract says *"if it's a racing car, bring a helmet"*, the abstraction has failed — you have leaked the subtype into the caller.

### Mapping the analogy back

| Analogy | Code |
|---|---|
| The rental contract | The base class's **public method signatures + documented behaviour** |
| The customer | The **caller** — any code holding a reference typed as the base class |
| "Any car will do" | **Polymorphism** — `cars.forEach(c => c.drive())` |
| Checking the brand before driving | `if (x instanceof Racing)` — **the LSP smell** |
| Racing licence required | **Strengthened precondition** |
| Toy car goes 3 km/h | **Weakened postcondition** |
| Hidden fuel cutoff | **Broken invariant** |
| "EngineLockedException" | **New exception type** |

**The one-line version:** *A subtype may do more, but it may never demand more, promise less, or surprise the caller.*

---

## Key concepts inside this topic

### 1. The four rules of a valid subtype

LSP is not vibes. It is four checkable rules. Recall from [25 — Design by Contract](./25-design-by-contract.md) that a method has a **precondition** (what must be true before you call it), a **postcondition** (what the method guarantees afterwards), and an **invariant** (what stays true about the object at all times).

| Rule | Meaning | Subtype must… |
|---|---|---|
| **1. Preconditions cannot be strengthened** | The subtype can't demand *more* from the caller | Accept everything the base accepted (it may accept *more*) |
| **2. Postconditions cannot be weakened** | The subtype can't guarantee *less* | Deliver everything the base promised (it may deliver *more*) |
| **3. Invariants must be preserved** | Rules that were always true about the base object stay true | Never break the base's own consistency rules |
| **4. No new exceptions** | The caller's `try/catch` must still be sufficient | Only throw exception types the base declared (or subtypes of them) |

Here they are as code you can feel:

```javascript
class PaymentGateway {
  // Precondition:  amount > 0
  // Postcondition: returns { id, status: 'CAPTURED' | 'DECLINED' }
  // Throws:        NetworkError only
  async charge(amount, currency) { throw new Error('not implemented'); }
}

// ❌ RULE 1 — strengthened precondition. The base accepted ANY positive amount.
class StrictGateway extends PaymentGateway {
  async charge(amount) {
    if (amount < 50) throw new Error('Minimum charge is 50');  // caller never agreed
  }
}

// ❌ RULE 2 — weakened postcondition. Base promised a TERMINAL status; the caller
// only handles CAPTURED/DECLINED, so 'PENDING' is silently treated as neither.
class SlowGateway extends PaymentGateway {
  async charge(amount) { return { id: 'p_1', status: 'PENDING' }; }
}

// ❌ RULE 4 — new exception. Every caller's
// `catch (e) { if (e instanceof NetworkError) retry(); }` now leaks RateLimitError.
class ThrottledGateway extends PaymentGateway {
  async charge(amount) { if (this.overQuota()) throw new RateLimitError('slow down'); }
}

// ✅ VALID — accepts everything the base did, promises everything the base did,
// throws nothing new. Internals completely different. Perfectly fine.
class StripeGateway extends PaymentGateway {
  async charge(amount, currency) {
    const res = await this.stripe.charges.create({ amount, currency });
    return { id: res.id, status: res.paid ? 'CAPTURED' : 'DECLINED' };
  }
}
```

**A useful mnemonic:** preconditions can only get *looser* going down the hierarchy; postconditions can only get *tighter*. Requirements shrink, guarantees grow.

### 2. The classic violation: Rectangle / Square

Every textbook uses this because it's the perfect trap: it is *mathematically* true that a square is a rectangle, and *behaviourally* false that a `Square` is a `Rectangle`.

```javascript
// ❌ BAD — the classic LSP violation
class Rectangle {
  constructor(width, height) { this._width = width; this._height = height; }
  setWidth(w)  { this._width = w; }
  setHeight(h) { this._height = h; }
  get area()   { return this._width * this._height; }
}

class Square extends Rectangle {
  constructor(side) { super(side, side); }

  // A square must keep its sides equal — so setting one MUST set the other.
  // That is the invariant of a Square, and it silently breaks Rectangle's.
  setWidth(w)  { this._width = w; this._height = w; }
  setHeight(h) { this._width = h; this._height = h; }
}

// A caller who has never heard of Square. It only knows the Rectangle contract,
// which implies: width and height are INDEPENDENT. This function is CORRECT.
function resizeToBanner(rect) {
  rect.setWidth(20);
  rect.setHeight(10);
  return rect.area;                                // any Rectangle: 20 * 10 = 200
}

console.log(resizeToBanner(new Rectangle(1, 1)));  // 200  ✅
console.log(resizeToBanner(new Square(1)));        // 100  ❌ WRONG
```

Why 100? `setHeight(10)` on a `Square` also reset the width to 10. The caller did nothing wrong. `Square` did.

**The invariant that got broken:** `Rectangle` guarantees *"setting height does not change width."* `Square` guarantees *"width always equals height."* **Those two invariants are incompatible.** You cannot inherit from a class whose invariants you must violate to stay correct. That's the whole lesson.

> **Key insight:** LSP is about *behaviour under mutation*, not about set theory. An **immutable** `Square` really is a valid `Rectangle` — with no setters there is no invariant to break. It's the mutators that create the contradiction.

### 3. The second classic: Bird / Penguin

```javascript
// ❌ BAD — every Bird can fly. Except… no.
class Bird {
  fly() { return `${this.constructor.name} is flying at 30 km/h`; }
  eat() { return `${this.constructor.name} is eating`; }
}

class Sparrow extends Bird {}

class Penguin extends Bird {
  // Rule 4 broken: a brand-new exception the caller never agreed to handle.
  fly() { throw new Error('Penguins cannot fly'); }
}

// A perfectly reasonable caller, written against the Bird contract:
function migrate(birds) { return birds.map(b => b.fly()); }

migrate([new Sparrow(), new Penguin()]);   // 💥 Error: Penguins cannot fly
```

The fix people *reach for* first is the wrong one:

```javascript
// ❌ STILL BAD — this is the LSP smell, not the LSP fix.
function migrate(birds) {
  return birds.map(b =>
    b instanceof Penguin ? `${b.constructor.name} is waddling` : b.fly()
  );
}
```

You just pushed the subtype's problem into the caller. Add an ostrich, a kiwi, an emu — you edit `migrate()` every single time. **The hierarchy is wrong; stop patching the callers.**

### 4. The real fix: re-model the hierarchy around behaviour

The mistake was putting `fly()` on `Bird`. Flying isn't what makes something a bird — it's a **capability that some birds have**. Model capabilities as their own types.

```javascript
// ✅ GOOD — the hierarchy now says only what is TRUE of every bird.
class Bird {
  constructor(name) { this.name = name; }
  eat()  { return `${this.name} is eating`; }
  move() { throw new Error('subclass must implement move()'); }  // abstract
}

// Separate, narrower contracts for the subsets that fly / don't fly.
class FlyingBird extends Bird {
  move() { return this.fly(); }
  fly()  { return `${this.name} is flying at 30 km/h`; }
}

class FlightlessBird extends Bird {
  move() { return this.walk(); }
  walk() { return `${this.name} is walking`; }
}

class Sparrow extends FlyingBird {}
class Penguin extends FlightlessBird {
  // Penguins ADD a capability. Adding is ALWAYS safe — it can never break a
  // caller that only knows the base contract.
  swim() { return `${this.name} is swimming at 10 km/h`; }
}

// The caller now has ZERO instanceof — and works for every Bird, forever.
function relocate(birds) { return birds.map(b => b.move()); }

relocate([new Sparrow('Sparrow'), new Penguin('Penguin')]);
// [ 'Sparrow is flying at 30 km/h', 'Penguin is walking' ]
```

Add an ostrich tomorrow? `class Ostrich extends FlightlessBird {}`. **`relocate()` is not touched.** That is OCP falling out of LSP for free.

The same re-modelling saves Rectangle/Square: stop pretending one *is* the other.

```javascript
// ✅ GOOD — a common contract for what they genuinely SHARE (area), and NO
// inheritance between them, because their invariants conflict.
class Shape {
  get area() { throw new Error('subclass must implement area'); }
}

class Rectangle extends Shape {
  constructor(w, h) { super(); this._w = w; this._h = h; }
  setWidth(w)  { this._w = w; }
  setHeight(h) { this._h = h; }
  get area()   { return this._w * this._h; }
}

class Square extends Shape {
  constructor(side) { super(); this._side = side; }
  setSide(s) { this._side = s; }        // its OWN mutator, its OWN invariant
  get area() { return this._side ** 2; }
}

// Any caller that only needs `area` is now fully polymorphic and safe:
const total = [new Rectangle(20, 10), new Square(5)]
  .reduce((sum, s) => sum + s.area, 0);   // 200 + 25 = 225
```

### 5. The other fix: prefer composition

Sometimes there's no clean hierarchy to find. Then stop inheriting and **hold** the behaviour instead of **being** it. (This is the theme of [26 — Composition Over Inheritance](./26-composition-over-inheritance.md).)

```javascript
// ✅ GOOD — capabilities are objects you plug in, not ancestors you inherit.
const CanFly  = { move: (self) => `${self.name} is flying`  };
const CanWalk = { move: (self) => `${self.name} is walking` };
const CanSwim = { move: (self) => `${self.name} is swimming` };

class Bird {
  // The movement strategy is INJECTED, not baked into the class tree.
  constructor(name, locomotion) { this.name = name; this.locomotion = locomotion; }
  move() { return this.locomotion.move(this); }
  eat()  { return `${this.name} is eating`; }
}
[new Bird('Sparrow', CanFly), new Bird('Penguin', CanSwim), new Bird('Ostrich', CanWalk)]
  .forEach(b => console.log(b.move()));   // flying / swimming / walking
```

No hierarchy, no violation — and a flying-*and*-swimming duck is now trivial: compose two strategies. Inheritance would have needed a `FlyingSwimmingBird` class, then a `FlyingSwimmingWalkingBird`, and so on. That combinatorial explosion is itself the signal that you should have composed.

### 6. How to spot a violation in a code review

Scan for these five tells. Any one of them means "look harder."

| Tell | What it usually means |
|---|---|
| `if (x instanceof Sub)` in a caller | The base contract is a lie; the caller is patching it |
| `throw new Error('not supported')` in an override | Rule 4 — a new exception; the method doesn't belong on the base |
| An override that's an **empty body** `{}` | Silently weakened postcondition — worse than throwing, because it fails *quietly* |
| An override that adds a **new validation/guard clause** | Rule 1 — strengthened precondition |
| A subclass with a **`canX()` flag** the caller must check first | You've moved the `instanceof` into a boolean. Same disease |

That last one is the sneaky one. `class Penguin extends Bird { canFly() { return false; } }` *looks* tidy — but the caller still ends up writing `if (b.canFly()) b.fly();`. Same disease, better disguise. The test never changes: **can the caller use the base type without asking any questions?** If not, LSP is broken.

---

## Visual / Diagram description

### Diagram 1: The violation — the caller is forced to look inside

```
                     ┌──────────────────────────┐
                     │   migrate(birds)         │
                     │   ── the CALLER ──       │
                     │  if (b instanceof        │◀── THE SMELL.
                     │      Penguin) { ... }    │    The caller had to
                     │  else b.fly();           │    open the bonnet.
                     └────────────┬─────────────┘
                                  │ holds refs typed as Bird
                                  ▼
                     ┌──────────────────────────┐
                     │        Bird              │   contract says
                     │  + fly()   ◀── a LIE     │   "every bird flies"
                     │  + eat()                 │
                     └──────────┬───────────────┘
                                │ extends
                 ┌──────────────┴──────────────┐
                 ▼                             ▼
       ┌───────────────────┐         ┌────────────────────────┐
       │     Sparrow       │         │       Penguin          │
       │  (inherits fly)   │         │  fly() { throw ... }   │◀── breaks the
       │        ✅         │         │           ❌           │    contract
       └───────────────────┘         └────────────────────────┘
```

Read it top-down: the contract on `Bird` promises `fly()`. `Penguin` cannot honour it, so it throws. That broken promise travels **upward** into the caller, which is now forced to defend itself with `instanceof`. **A broken subtype always leaks into its callers.**

### Diagram 2: The fix — re-modelled hierarchy, caller knows nothing

```
                     ┌──────────────────────────┐
                     │   relocate(birds)        │
                     │   ── the CALLER ──       │
                     │   b.move()               │◀── no instanceof.
                     │                          │    no questions asked.
                     └────────────┬─────────────┘
                                  │ holds refs typed as Bird
                                  ▼
                     ┌──────────────────────────┐
                     │        Bird              │   contract only promises
                     │  + eat()                 │   what is TRUE of EVERY
                     │  + move()   «abstract»   │   bird.
                     └──────────┬───────────────┘
                                │ extends
                 ┌──────────────┴───────────────┐
                 ▼                              ▼
    ┌──────────────────────┐        ┌──────────────────────────┐
    │     FlyingBird       │        │    FlightlessBird        │
    │  + move() → fly()    │        │  + move() → walk()       │
    │  + fly()             │        │  + walk()                │
    └──────────┬───────────┘        └────────────┬─────────────┘
               │ extends                         │ extends
               ▼                                 ▼
    ┌──────────────────────┐        ┌──────────────────────────┐
    │      Sparrow    ✅   │        │   Penguin           ✅   │
    │                      │        │  + swim() ◀── ADDS a     │
    │                      │        │     capability; breaks   │
    │                      │        │     nothing.             │
    └──────────────────────┘        └──────────────────────────┘
```

Every arrow now carries a promise that is actually kept. Adding `Ostrich` under `FlightlessBird` requires editing **zero** existing boxes.

### Diagram 3: The contract funnel — the shape of a legal subtype

```
   PRECONDITIONS (what the caller must supply)
           ├─────────────────┤          Base accepts this much
   ├─────────────────────────────────┤  Subtype may accept MORE (never less)

   POSTCONDITIONS (what the callee must guarantee)
   ├─────────────────────────────────┤  Base promises this much
           ├─────────────────┤          Subtype may promise MORE (never less)

   ═══▶ Requirements SHRINK ▼   ·   Guarantees GROW ▲   going DOWN the hierarchy
```

Draw this and you can re-derive rules 1 and 2 from scratch in an interview.

---

## Real world examples

### 1. Node.js Streams — LSP done right

Node's stream API is a large, working LSP contract. `fs.createReadStream()`, `process.stdin`, an HTTP request object, a `zlib.createGunzip()` output — all are `Readable` streams. Any code written against the `Readable` contract (`.on('data')`, `.on('end')`, `.pipe()`) works with every one of them, which is why `source.pipe(transform).pipe(destination)` works no matter what you plug in.

`pipe()` never does `if (source instanceof FileStream)`. It can't — it has no idea what it was handed, and it doesn't need to. That's LSP holding across a dozen unrelated implementations, and it's exactly what makes streams composable.

### 2. Java's `Collections.unmodifiableList()` — a violation in a standard library

Java hands you a `List` whose `add()` throws `UnsupportedOperationException`. It *is* a `List` by type, but it breaks the `List` contract twice over — rule 4 (a new exception) and rule 2 (`add()` no longer guarantees the element is added).

The result is exactly the smell this doc predicts: real Java codebases carry defensive `try/catch (UnsupportedOperationException)` and "is this list actually mutable?" checks. It's a useful example because even world-class API designers get bitten — and the cost is paid by *every caller*, forever. (JavaScript has the same wart: `Object.freeze()` makes writes silently no-op in sloppy mode and throw in strict mode.)

### 3. ORM base classes — where LSP breaks in production Node apps

A very common real pattern:

```javascript
class BaseRepository {
  async save(entity) { /* Contract: persists the entity, returns it with an id. */ }
  async delete(id)   { /* Contract: removes the entity PERMANENTLY. */ }
}

// ❌ Breaks LSP. The contract said "removes permanently". This does not.
class AuditedRepository extends BaseRepository {
  async delete(id) { return this.update(id, { deletedAt: new Date() }); }  // soft delete!
}
```

Any service written against `BaseRepository` — say a GDPR "erase my data" job — will believe the row is gone. It isn't. This is a **weakened postcondition**, and it's a compliance incident, not just a design smell.

The fix is the usual one: don't overload one method with two different promises. Give the base a `delete(id)` that really deletes, and put `softDelete(id)` on a separate `SoftDeletable` role that only the callers who actually *want* it will ask for.

---

## Trade-offs

| Approach | Pros | Cons |
|---|---|---|
| **Deep inheritance hierarchy** | Reuse comes free; feels natural for real "is-a" relationships | Every level is a contract you must keep. One bad override poisons all callers, and violations are hard to spot |
| **Re-model into narrower base types** (`FlyingBird` / `FlightlessBird`) | Restores true polymorphism; callers stop asking questions; new types slot in for free | More classes; you must *find* the right axis of variation, and you may guess wrong |
| **Composition (inject the capability)** | No hierarchy to break; combinations are trivial (fly + swim); each behaviour is testable alone | Slightly more wiring at construction; loses the "free" reuse of `extends` |
| **Leave the violation, use `instanceof` in callers** | Zero refactor today | Every new subtype edits every caller. Violates OCP. Technical debt with compound interest |

What you give up by enforcing LSP is the convenience of "just `extends` the thing that's closest," and you end up with more, smaller classes. In exchange you never write an `instanceof` chain across 40 call sites, and each of those small classes is far easier to test in isolation.

**Rule of thumb:** Before writing `class B extends A`, ask one question — *"Can I hand a `B` to every single function that currently takes an `A`, and have all of them still behave correctly, with no code changes?"* If the honest answer is "no, except for…", **do not inherit.** Re-model, or compose.

---

## Common interview questions on this topic

### Q1: "Explain the Liskov Substitution Principle in one sentence."
**Hint:** "Any code written against a base type must keep working, unchanged and correctly, when given any subtype." Then immediately give the tell: *"and the way you know it's broken is that the caller starts writing `instanceof` checks."* That second half is what separates a memorised definition from understanding.

### Q2: "A square IS a rectangle in mathematics. So why is `class Square extends Rectangle` wrong?"
**Hint:** Because inheritance is about **behavioural** substitutability, not set membership. `Rectangle` has an invariant — *"width and height are independent"* — and `Square` has a contradicting one — *"width always equals height."* A `Square` that honours its own invariant must break `Rectangle`'s `setHeight()` postcondition. Bonus point: *"an immutable Square would be a perfectly legal Rectangle — the problem is the mutators, not the geometry."*

### Q3: "What are the formal rules for a valid subtype?"
**Hint:** Four. Preconditions can't be strengthened (can't demand more from the caller). Postconditions can't be weakened (can't promise less). Invariants must be preserved. No new exception types. Summarise: *"requirements shrink going down, guarantees grow going down."*

### Q4: "How does LSP relate to the other SOLID principles?"
**Hint:** LSP is the *enabler* of OCP. OCP says "extend without modifying existing code" — but extension only works if the new subtype is substitutable, i.e. LSP holds. A broken subtype forces `instanceof` in the callers, which means you *must* modify existing code for every new type, which is precisely an OCP violation. LSP is also what makes the abstractions in DIP ([18](./18-solid-dependency-inversion.md)) trustworthy — an injected fake is only a valid stand-in if it honours the port's contract.

### Q5: "You inherit a codebase where `PDFExporter`, `CSVExporter`, and `ImageExporter` all extend `Exporter`, but `ImageExporter.setDelimiter()` throws. How do you fix it?"
**Hint:** `setDelimiter()` doesn't belong on `Exporter` — it's only meaningful for delimited text formats. That method is on the base class purely because two of the three subtypes wanted it. Pull it down into a `DelimitedExporter` (which `CSV` and `TSV` extend) or make it a separate role/mixin. This is LSP and ISP ([17](./17-solid-interface-segregation.md)) pointing at the same bug from two directions: a fat base class forces some subtype to lie.

---

## Practice exercise

### The Account Hierarchy Audit

You've been handed this code. It's real — variations of it ship in production fintech apps.

```javascript
class BankAccount {
  constructor(owner, balance = 0) { this.owner = owner; this.balance = balance; }

  // Precondition:  amount > 0
  // Postcondition: balance decreases by exactly `amount`; returns the new balance
  // Invariant:     balance NEVER goes negative
  withdraw(amount) {
    if (amount <= 0) throw new Error('Amount must be positive');
    if (amount > this.balance) throw new Error('Insufficient funds');
    this.balance -= amount;
    return this.balance;
  }

  deposit(amount) {
    if (amount <= 0) throw new Error('Amount must be positive');
    this.balance += amount;
    return this.balance;
  }
}

class FixedDepositAccount extends BankAccount {
  withdraw(amount) { throw new Error('Cannot withdraw before maturity'); }
}

class OverdraftAccount extends BankAccount {
  constructor(owner, balance, overdraftLimit) {
    super(owner, balance);
    this.overdraftLimit = overdraftLimit;
  }
  withdraw(amount) {
    if (amount > this.balance + this.overdraftLimit) throw new Error('Over limit');
    this.balance -= amount;   // balance can now go NEGATIVE
    return this.balance;
  }
}
```

**Produce these four things (~30 min):**

1. **The audit.** For `FixedDepositAccount` and `OverdraftAccount`, state exactly **which of the four LSP rules** each breaks, naming the specific precondition / postcondition / invariant / exception involved. (Hint: `OverdraftAccount` breaks a *different* rule than the obvious one — read the base class's documented **invariant**, not just its code.)

2. **The proof.** Write `runPayroll(accounts, amount)` — it withdraws `amount` from every account and returns the total. Write it **honestly**, as someone who has only ever read `BankAccount`. Then show, with a runnable script, the two different ways it breaks.

3. **The wrong fix.** Rewrite `runPayroll` using `instanceof` to make it "work" — then write one sentence on why this fix is worse than the bug.

4. **The right fix.** Re-model the hierarchy so `runPayroll` needs zero `instanceof` and zero try/catch, and adding a `SavingsAccount` tomorrow edits zero existing files. (Strong hint: not every account is withdrawable. What *narrower base type* does `runPayroll` actually need — and what does that tell you about where `withdraw()` should live?)

**Deliverable:** one file, `accounts.js` — the audit as comments, the broken version, and the fixed version with a `main()` demo showing `runPayroll` handling every account type without a single type check.

---

## Quick reference cheat sheet

- **LSP in one line:** a subtype must be usable *anywhere* the base type is, with no caller changes and no surprises.
- **The #1 smell:** `instanceof` (or a `canX()` flag) appearing in a **caller**. If the caller has to ask what type it holds, LSP is already broken.
- **Rule 1 — Preconditions can't be strengthened.** The subtype may not demand more from the caller (no new validations, no narrower input range).
- **Rule 2 — Postconditions can't be weakened.** The subtype may not promise less (no `PENDING` where the base promised a final answer; no empty override bodies).
- **Rule 3 — Invariants must hold.** Whatever was always true of the base object stays true (balance never negative; width independent of height).
- **Rule 4 — No new exceptions.** The caller's existing `catch` must still be sufficient.
- **The mnemonic:** going *down* a hierarchy, **requirements shrink, guarantees grow.**
- **A subtype may always ADD.** New methods (`Penguin.swim()`) are always safe. It's *changing* inherited behaviour that kills you.
- **Rectangle/Square is about mutation, not maths.** Immutable shapes have no invariant to break — the setters create the contradiction.
- **`throw new Error('not supported')` in an override** is an LSP violation with a confession attached. **An empty override `{}` is worse** — it fails silently, in production, at 3am.
- **LSP enables OCP.** Break LSP and every new subtype forces edits to every caller — which *is* an OCP violation.
- **Two fixes only:** re-model the hierarchy around what's *actually* common, or drop inheritance and **compose** the varying behaviour.
- **The pre-commit question:** *"Can I pass this subclass to every existing function that takes the base — with no edits — and be confident all of them stay correct?"* If not, don't `extends`.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [15 — SOLID: Open/Closed Principle](./15-solid-open-closed.md) — LSP is what makes OCP's extension points actually safe to extend |
| **Next** | [17 — SOLID: Interface Segregation Principle](./17-solid-interface-segregation.md) — the other half of the "fat base class" problem: don't force clients to depend on methods they don't use |
| **Related** | [26 — Composition Over Inheritance](./26-composition-over-inheritance.md) — the escape hatch when no hierarchy can honour every contract |
| **Related** | [25 — Design by Contract](./25-design-by-contract.md) — preconditions, postconditions, and invariants, which are the vocabulary LSP is written in |
