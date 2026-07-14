# 26 — Composition Over Inheritance — Why and When
## Category: LLD Fundamentals

---

## What is this?

**Inheritance** says *"a Duck IS-A Bird, so it gets everything Bird has."* **Composition** says *"a Duck HAS-A fly behaviour and HAS-A quack behaviour, and I'll hand those to it when I build it."*

Think of a person and their skills. Inheritance is like being born into a family — you inherit your parents' entire genome, whether you wanted the bad knees or not, and you can never give them back. Composition is like a toolbelt — you clip on a hammer, a screwdriver, and a tape measure, and you can swap any of them mid-job.

The Gang of Four (the four authors of the 1994 *Design Patterns* book) wrote one sentence that has aged better than almost anything else in software: **"Favor object composition over class inheritance."** This doc explains exactly why — and, crucially, when inheritance is still the right call.

---

## Why does it matter?

If you don't understand this, here is what happens to your code, in order:

1. You write a base class. It feels clean.
2. A new requirement arrives that doesn't fit. You add a subclass.
3. Another requirement arrives that cuts *across* the hierarchy. You add a flag to the base class, or you copy-paste a subclass.
4. Six months later you have a 9-level-deep hierarchy, a base class that nobody dares touch because 40 classes extend it, and subclasses that inherit methods they must override to throw `Error('not supported')`.

That last symptom — overriding a method just to disable it — is the smell that tells you inheritance was the wrong tool. It's also a direct violation of the Liskov Substitution Principle (recall from [17 — SOLID: Liskov Substitution](./17-solid-liskov-substitution.md) that a subclass must be usable *anywhere* the base class is, with no surprises).

**In interviews:** This is one of the most reliable follow-ups in an LLD round. You design a class hierarchy, and the interviewer says *"now the system needs a rubber duck that can't fly."* If your first instinct is `class RubberDuck extends Duck { fly() { throw new Error(); } }`, you've failed the question. The expected answer is to pull the varying behaviour out into its own object.

**At work:** Composition is what makes code testable. You can't easily mock a superclass — it's baked into the class at definition time. You *can* trivially pass a fake `PaymentProcessor` into a constructor. Almost every dependency-injection framework, every testing strategy, and every plugin system in the Node ecosystem rests on composition.

---

## The core idea — explained simply

### The Duck Simulator Disaster

You're building a duck simulator. (This example is famous — it opens the *Head First Design Patterns* book — and it's famous because it happens in real codebases every week, just with `Report` and `Invoice` instead of ducks.)

**Day 1.** Requirements: ducks quack and swim. Easy.

```javascript
class Duck {
  quack() { console.log('Quack!'); }
  swim()  { console.log('Paddling along...'); }
  display() { throw new Error('Subclass must implement display()'); }
}

class MallardDuck extends Duck {
  display() { console.log('I look like a mallard.'); }
}

class RedheadDuck extends Duck {
  display() { console.log('I look like a redhead.'); }
}
```

Beautiful. Two subclasses, zero duplication. Inheritance is great!

**Day 30.** Product says: "Ducks should fly." Obviously, you add `fly()` to the base class. One line, every duck gets it. This is the *exact moment* the design starts to rot.

```javascript
class Duck {
  quack() { console.log('Quack!'); }
  swim()  { console.log('Paddling along...'); }
  fly()   { console.log('Flying!'); }   // <-- the fatal line
  display() { throw new Error('Subclass must implement display()'); }
}
```

**Day 31.** Marketing adds a **RubberDuck**. It squeaks instead of quacking, and it *does not fly*. Rubber ducks are now flying around the simulator. You "fix" it:

```javascript
class RubberDuck extends Duck {
  quack() { console.log('Squeak!'); }        // override to change
  fly()   { /* do nothing — rubber ducks can't fly */ }  // override to DISABLE
  display() { console.log('I am a rubber duck.'); }
}
```

Overriding a method to make it do nothing is you *fighting your own base class*. Note it well.

**Day 45.** A **DecoyDuck** (a wooden hunting decoy). It can't fly and it can't quack — it makes no sound at all. Two more do-nothing overrides.

**Day 60.** Someone adds a **RocketPoweredDuck**. And a **JetDuck**. And now product wants a duck whose flight behaviour can *change at runtime* (it gets a rocket strapped on mid-game). Inheritance cannot do that — the class of an object is fixed the moment `new` runs.

### The combinatorial explosion

Now watch what happens if you try to solve this by adding subclasses for each behaviour combination. You have 3 varying axes:

- **Flight:** flies with wings / can't fly / rocket-powered → 3 options
- **Sound:** quack / squeak / silent → 3 options
- **Appearance:** mallard / redhead / rubber / decoy → 4 options

Number of subclasses needed to cover every combination: **3 × 3 × 4 = 36 classes.**

```
Duck
├── FlyingQuackingMallardDuck
├── FlyingQuackingRedheadDuck
├── FlyingSqueakingRubberDuck
├── NonFlyingSqueakingRubberDuck
├── NonFlyingSilentDecoyDuck
├── RocketPoweredQuackingMallardDuck
├── FlyingSwimmingRubberDuck            <-- these names are a cry for help
├── NonFlyingSilentRubberDuck
├── ... 28 more ...
```

Add one new sound (say, "honk") and you don't add one class — you add **twelve** (3 flight × 4 appearance). The class count grows as the *product* of the axes, not the sum. That is the **combinatorial explosion of subclasses**, and it is the single clearest signal that you're modelling behaviour with inheritance when you should be modelling it with objects.

### The fix: hand the duck its behaviours

Stop asking *"what IS this duck?"* Start asking *"what does this duck DO, and can that change?"*

Flight and sound **vary**. So pull them out of the Duck entirely, put each behind its own tiny object, and *inject* those objects when the duck is constructed.

```javascript
// --- Behaviour objects: each one knows ONE thing ---
const FlyWithWings   = { fly: () => console.log('Flap flap — flying!') };
const FlyNoWay       = { fly: () => console.log('(cannot fly)') };
const FlyRocket      = { fly: () => console.log('WHOOSH — rocket flight!') };

const QuackNormal    = { makeSound: () => console.log('Quack!') };
const Squeak         = { makeSound: () => console.log('Squeak!') };
const MuteQuack      = { makeSound: () => console.log('...') };

// --- The Duck no longer OWNS flight or sound. It DELEGATES. ---
class Duck {
  // Behaviours arrive at construction — this is dependency injection.
  constructor({ flyBehavior, soundBehavior, name }) {
    this.flyBehavior = flyBehavior;
    this.soundBehavior = soundBehavior;
    this.name = name;
  }

  performFly()   { this.flyBehavior.fly(); }        // delegate, don't implement
  performQuack() { this.soundBehavior.makeSound(); }

  // Behaviour that is TRULY common to every duck stays here.
  swim() { console.log(`${this.name} is paddling.`); }

  // Runtime swap — impossible with inheritance.
  setFlyBehavior(behavior) { this.flyBehavior = behavior; }
}

const mallard = new Duck({ name: 'Mallard', flyBehavior: FlyWithWings, soundBehavior: QuackNormal });
const rubber  = new Duck({ name: 'Rubber',  flyBehavior: FlyNoWay,     soundBehavior: Squeak });
const decoy   = new Duck({ name: 'Decoy',   flyBehavior: FlyNoWay,     soundBehavior: MuteQuack });

rubber.performFly();   // (cannot fly)   -- no override needed, no lie told
mallard.performFly();  // Flap flap — flying!

mallard.setFlyBehavior(FlyRocket);  // strap on a rocket, mid-game
mallard.performFly();               // WHOOSH — rocket flight!
```

**36 classes became 1 class + 6 tiny behaviour objects.** Adding "honk" is now *one object*, not twelve classes. And every combination is expressible without writing any new class at all.

### And by the way — you just wrote the Strategy pattern

That is not "a composition trick." That is literally the **Strategy pattern**: *define a family of interchangeable algorithms, encapsulate each one, and make them swappable at runtime.* `FlyBehavior` is the strategy interface, `FlyWithWings` / `FlyNoWay` / `FlyRocket` are the concrete strategies, and `Duck` is the context that holds one. We'll formalize it in [42 — Strategy Pattern](./42-pattern-strategy.md), but you should recognize it now: **most of the time, "use composition instead of inheritance here" and "use Strategy here" are the same sentence.**

### Mapping the analogy back

| In the duck story | In the technical concept |
|---|---|
| "Ducks fly" put on the base class | Behaviour hoisted into a superclass because it applied to *most* subclasses |
| RubberDuck overriding `fly()` to do nothing | Subclass rejecting inherited behaviour → Liskov violation |
| The 36-class name soup | Combinatorial explosion — classes grow as the product of varying axes |
| `flyBehavior` handed in at construction | Dependency injection / composition |
| `setFlyBehavior()` at runtime | Behaviour is now data, not identity — it can change |
| The behaviour objects | Concrete Strategy implementations |

---

## Key concepts inside this topic

### 1. The Fragile Base Class Problem

A base class is a **published contract with every subclass, forever** — but unlike a real API, the compiler never tells you when you break it.

You "safely" refactor a base class. Somewhere, three levels down, a subclass overrode a method the base class calls internally. Your change alters *when* that method is called, and a class you have never opened silently breaks.

```javascript
class Collection {
  add(item) { this.items.push(item); }
  addAll(items) { for (const i of items) this.add(item); }  // addAll calls add()
}

// A subclass that counts every insert.
class CountingCollection extends Collection {
  add(item)     { this.count += 1; super.add(item); }
  addAll(items) { this.count += items.length; super.addAll(items); }  // BUG: double-counts!
}
```

`addAll` counts the batch, then `super.addAll` loops calling `add`, which counts each one again. The subclass author wasn't careless — they simply couldn't know that `addAll` was implemented *in terms of* `add`. That's a **private implementation detail of the base class that leaked into the inheritance contract.**

Now suppose the base class team "optimizes" `addAll` to push directly instead of calling `add`. The double-count bug disappears — and a *different* subclass that relied on `add` being called for every item silently stops working. **The base class cannot be changed without possibly breaking subclasses it has never heard of.** That is the fragile base class problem.

Composition immunizes you: a wrapper only ever calls the *public* API of the thing it wraps.

```javascript
class CountingCollection {
  constructor(inner) { this.inner = inner; this.count = 0; }  // HAS-A, not IS-A
  add(item)     { this.count += 1; this.inner.add(item); }
  addAll(items) { for (const i of items) this.add(i); }        // I control my own flow
  size()        { return this.inner.size(); }
}
```

No internal calls to override. No surprises. (This is the Decorator pattern, [35](./35-pattern-decorator.md) — again, composition and a named pattern turn out to be the same thing.)

### 2. The "Gorilla Holding the Banana" Problem

Joe Armstrong, the creator of Erlang, put it best:

> *"You wanted a banana but what you got was a gorilla holding the banana and the entire jungle."*

You extend a class because you wanted **one method**. What you actually got is every field, every method, every lifecycle hook, and every dependency of that class and all *its* ancestors.

```javascript
// You wanted: a retry helper.
class BaseApiClient {
  constructor() {
    this.httpAgent   = new HttpAgent();      // you didn't ask for this
    this.metrics     = new MetricsReporter();// or this
    this.authToken   = null;                 // or this
    this.retryPolicy = new RetryPolicy();    // <-- this is the banana
  }
  retry(fn) { return this.retryPolicy.run(fn); }
  // ...plus 40 other methods you will never call
}

// You need retry inside a background job. So...
class EmailJob extends BaseApiClient {   // now EmailJob has an HTTP agent. Why?
  async run() { return this.retry(() => this.sendEmail()); }
}
```

`EmailJob` now drags an HTTP agent, a metrics reporter, and an auth token into every unit test, every startup path, and every mental model of the class. It has a *public* `authToken` field for no reason. Its test setup needs mocks for things it doesn't use.

The composed version takes the banana and leaves the jungle:

```javascript
class EmailJob {
  constructor(retryPolicy) { this.retryPolicy = retryPolicy; }  // exactly one dependency
  async run() { return this.retryPolicy.run(() => this.sendEmail()); }
}
```

**Rule:** if you're extending a class to get access to *one or two* of its methods, you want composition. Inheritance is not a code-reuse mechanism — it's a *subtyping* mechanism. Reusing code is a side effect, and using it *only* for that side effect is the mistake.

### 3. Composition Is Interception; Inheritance Is Substitution

Two different questions, two different answers:

- **"Can I use a `RubberDuck` everywhere the code expects a `Duck`, with zero surprises?"** → That's a subtyping question. If yes, inheritance is *legal*.
- **"Do I want to reuse this behaviour?"** → That's a reuse question. Inheritance is the wrong tool for it. Compose.

The strongest formulation: **inheritance is for what a thing IS; composition is for what a thing HAS or DOES.** A `SavingsAccount` genuinely *is* an `Account`. A `Duck` doesn't *have to be* a flying thing — it *has* a flight behaviour.

### 4. Mixins — Composition at the Class Level

JavaScript has no multiple inheritance, but it has something better: **mixins**. A mixin is a function that takes a class and returns a *new* class with extra behaviour bolted on. You compose capabilities instead of inheriting them.

```javascript
// A mixin is just: (Base) => class extends Base { ...extra stuff... }
const Serializable = (Base) => class extends Base {
  toJSON() { return JSON.stringify({ ...this }); }
};

const Timestamped = (Base) => class extends Base {
  constructor(...args) {
    super(...args);
    this.createdAt = new Date();
  }
  age() { return Date.now() - this.createdAt.getTime(); }
};

const Auditable = (Base) => class extends Base {
  #log = [];
  record(event) { this.#log.push({ event, at: Date.now() }); }
  auditTrail() { return [...this.#log]; }
};

// Pick exactly the capabilities you want, per class.
class Order extends Auditable(Timestamped(Serializable(Object))) {
  constructor(id, total) { super(); this.id = id; this.total = total; }
}

class Invoice extends Timestamped(Serializable(Object)) {   // no auditing needed here
  constructor(id) { super(); this.id = id; }
}

const o = new Order('ord_1', 4999);
o.record('created');
console.log(o.toJSON(), o.auditTrail(), o.age());
```

Mixins fix the *combination* problem (you pick capabilities à la carte, no 36-class explosion) but they're still `extends` under the hood — so they **do not** fix the fragile-base-class problem, and name collisions between two mixins are silent (the last one wins). Use them for cross-cutting, additive capabilities (serialization, event emitting, timestamps). Do **not** use them for the core varying behaviour of your domain — that's a job for plain behaviour objects.

### 5. Behaviour Objects vs. Plain Functions

In JS, a "behaviour object" doesn't need to be a class at all. Three escalating levels of ceremony:

```javascript
// Level 1 — a plain function. Perfect when the behaviour has no state.
const flyWithWings = () => console.log('Flap flap');
class Duck1 { constructor(fly) { this.fly = fly; } performFly() { this.fly(); } }

// Level 2 — a plain object. Use when the behaviour needs multiple methods.
const wingedFlight = {
  fly()          { console.log('Flap flap'); },
  maxAltitude()  { return 3000; },
};

// Level 3 — a class. Use when the behaviour has its OWN state or config.
class RocketFlight {
  constructor(fuelKg) { this.fuelKg = fuelKg; }   // strategy with state
  fly() {
    if (this.fuelKg <= 0) return console.log('Out of fuel — gliding.');
    this.fuelKg -= 1;
    console.log(`WHOOSH (${this.fuelKg}kg fuel left)`);
  }
}
```

Start at Level 1. Escalate only when you need to. A JS engineer who reaches for a 3-class hierarchy to model "how a thing flies" is writing Java with JS syntax.

### 6. When Inheritance IS the Right Answer

Be balanced here — "always compose" is as dumb as "always inherit." Inheritance earns its keep in four situations:

**a) A true, stable IS-A taxonomy.** The relationship is real, permanent, and won't grow new axes. `HttpError extends Error`. `SavingsAccount extends Account`. `Circle extends Shape`. A circle will never stop being a shape.

**b) Framework base classes with template methods.** The framework owns the algorithm's skeleton; you fill in the blanks. This is the Template Method pattern ([45](./45-pattern-template-method.md)) and it's inheritance done right.

```javascript
class ReportJob {
  // The framework controls the sequence. You cannot get it wrong.
  async run() {
    const raw = await this.fetch();        // your job
    const rows = this.transform(raw);      // your job
    await this.publish(rows);              // framework's job
    this.metrics.increment('report.done'); // framework's job
  }
  async fetch()  { throw new Error('subclass must implement fetch()'); }
  transform(raw) { return raw; }           // sensible default, override optional
  async publish(rows) { /* shared, real implementation */ }
}

class DailySalesReport extends ReportJob {
  async fetch() { return this.db.query('SELECT * FROM sales WHERE day = CURRENT_DATE'); }
  transform(rows) { return rows.map(r => ({ ...r, total: r.qty * r.price })); }
}
```

**c) Error hierarchies.** `catch (e) { if (e instanceof ValidationError) ... }` requires a real prototype chain. Composition can't give you `instanceof`.

```javascript
class AppError extends Error {
  constructor(message, { status = 500, cause } = {}) {
    super(message, { cause });
    this.name = new.target.name;   // 'ValidationError', not 'AppError'
    this.status = status;
    Error.captureStackTrace?.(this, new.target);
  }
}
class ValidationError extends AppError { constructor(m) { super(m, { status: 400 }); } }
class NotFoundError  extends AppError { constructor(m) { super(m, { status: 404 }); } }
```

**d) Extending a platform/library class you don't own.** `class MyStream extends Transform`, `class MyEmitter extends EventEmitter`. The library *requires* the prototype chain to work. Just keep the hierarchy one level deep.

### 7. The Decision Rules

| Ask yourself | Answer | Do this |
|---|---|---|
| Is this a genuine, permanent **IS-A**? (An X *is* a Y in the real domain) | Yes | Inheritance is allowed |
| Is this really **HAS-A** / **CAN-DO**? | Yes | **Compose** |
| Am I extending just to **reuse a method**? | Yes | **Compose** — you want the banana, not the gorilla |
| Would a subclass need to **override a method to disable it**? | Yes | **Compose** — inheritance is already lying |
| Does the behaviour need to **change at runtime**? | Yes | **Compose** (Strategy) — a class is fixed at `new` |
| Are there **2+ independent axes of variation**? | Yes | **Compose** — otherwise combinatorial explosion |
| Do I need `instanceof` checks (errors, framework dispatch)? | Yes | Inheritance |
| Does a framework/library **require** me to extend its class? | Yes | Inheritance — keep it 1 level deep |
| Is the hierarchy already **3+ levels deep**? | Yes | Stop. Refactor to composition. |
| Do I control the base class, and will it stay frozen? | No (don't control it, or it churns) | **Compose** — fragile base class risk |

**The one-line test:** say the relationship out loud. *"A rubber duck is a duck"* — sounds fine, so inheritance is tempting. Now say *"A rubber duck is a flying duck"* — that's a lie, and lies in your type system become bugs.

---

## Visual / Diagram description

### Diagram 1: The inheritance explosion vs. the composed design

```
    ══════════ INHERITANCE (the trap) ══════════

                    ┌───────────────┐
                    │     Duck      │   fly()  quack()  swim()
                    └───────┬───────┘   ▲ behaviour lives HERE
            ┌───────────────┼───────────────┬──────────────┐
            ▼               ▼               ▼              ▼
      ┌──────────┐   ┌───────────┐   ┌───────────┐  ┌────────────┐
      │ Mallard  │   │ Redhead   │   │ RubberDuck│  │ DecoyDuck  │
      │  Duck    │   │  Duck     │   │           │  │            │
      │          │   │           │   │ fly()  ✗  │  │ fly()   ✗  │
      │          │   │           │   │ quack()✗  │  │ quack() ✗  │
      └──────────┘   └───────────┘   └───────────┘  └────────────┘
                                       ▲              ▲
                          ✗ = overridden to DO NOTHING (the smell)

      Add "rocket flight" + "honk"?  →  3 flight × 3 sound × 4 look = 36 CLASSES


    ══════════ COMPOSITION (the fix) ══════════

                        ┌─────────────────────────┐
                        │         Duck            │
                        │                         │
                        │  - flyBehavior   ───────┼──┐  HAS-A
                        │  - soundBehavior ───────┼──┼──┐
                        │                         │  │  │
                        │  + performFly()  ───────┼──┘  │  delegates
                        │  + performQuack()───────┼─────┘
                        │  + setFlyBehavior(b)    │  ◀── runtime swap!
                        │  + swim()               │
                        └─────────────────────────┘
                                 │                         │
                ┌────────────────▼──────┐   ┌──────────────▼────────┐
                │  «interface»          │   │  «interface»          │
                │   FlyBehavior         │   │   SoundBehavior       │
                │   + fly()             │   │   + makeSound()       │
                └───────────┬───────────┘   └───────────┬───────────┘
                            │ implements                │ implements
          ┌─────────────────┼──────────────┐   ┌────────┼─────────┐
          ▼                 ▼              ▼   ▼        ▼         ▼
    ┌───────────┐   ┌────────────┐  ┌──────────┐ ┌────────┐ ┌──────────┐
    │FlyWithWings│  │  FlyNoWay  │  │FlyRocket │ │ Quack  │ │ Squeak   │
    └───────────┘   └────────────┘  └──────────┘ └────────┘ └──────────┘

      Add "rocket flight" + "honk"?  →  2 SMALL OBJECTS. Zero new Duck classes.
```

**Read it like this.** In the top half, behaviour lives *in* the hierarchy — so every new behaviour must become a new branch, and any subclass that doesn't want a behaviour has to actively refuse it (the `✗` marks). In the bottom half, `Duck` holds *references* to behaviour objects (the `HAS-A` lines) and forwards calls to them (`delegates`). The set of behaviours grows sideways, independently of the set of ducks. The two dashed families never multiply against each other.

### Diagram 2: What a call actually does (sequence)

```
  Client              Duck                 FlyBehavior(FlyRocket)
    │                  │                            │
    │ performFly()     │                            │
    ├─────────────────▶│                            │
    │                  │  this.flyBehavior.fly()    │
    │                  ├───────────────────────────▶│
    │                  │                            │ (burns fuel,
    │                  │                            │  prints WHOOSH)
    │                  │◀───────────────────────────┤
    │◀─────────────────┤                            │
    │                  │
    │ setFlyBehavior(FlyNoWay)                      │
    ├─────────────────▶│  ── swaps the reference ──▶ (FlyNoWay)
    │                  │
    │ performFly()     │
    ├─────────────────▶│──────────▶ "(cannot fly)"
```

`Duck.performFly()` contains **no flight logic at all** — it's a one-line forward. That single arrow of indirection is the entire trick, and it's what buys you runtime swapping, independent testing of each behaviour, and freedom from the class explosion.

---

## Real world examples

### 1. React — the industry's most public inheritance retreat

React originally shipped `React.createClass` and then ES6 class components, with code reuse handled by **mixins** and later **higher-order components** (a component that wraps a component). Both created exactly the problems above: wrapper hell, name collisions between mixins, and "which HOC injected this prop?" debugging.

In 2018 React introduced **Hooks** and the official guidance became: *don't use inheritance for code reuse in React; use composition.* `useState`, `useEffect`, and custom hooks are behaviour objects you *call into* rather than a base class you *descend from*. The React docs still carry a section titled "Composition vs Inheritance" with the blunt conclusion: *"we haven't found any use cases where we would recommend creating component inheritance hierarchies."* That's a mainstream framework, at massive scale, arriving at the GoF advice from first principles.

### 2. Node.js Streams — inheritance where it belongs

Node's stream module is the counter-example: it *asks* you to extend. `class MyTransform extends Transform { _transform(chunk, enc, cb) { ... } }`. This is the Template Method pattern — Node owns the backpressure algorithm, the buffering, the event emission, and the state machine, and you fill in one hole (`_transform`). The hierarchy is **stable** (it hasn't grown a new axis in a decade) and **shallow** (Stream → Duplex → Transform → yours). That's exactly the profile where inheritance wins.

But notice: once you have your streams, you combine them by **composition** — `.pipe()` chains them into a pipeline, and each stream stays independent. Inheritance to *build* a stream; composition to *use* them together.

### 3. Game engines — Unity, Unreal, and the ECS shift

Early game engines used deep inheritance: `GameObject → Character → Player → ArcheryPlayer`. Then the designer asks for a flying archer that's also a vehicle, and you hit the duck problem at industrial scale.

The industry's answer was the **Entity-Component-System** architecture, and Unity's core API (`GameObject.AddComponent<Rigidbody>()`) is composition made literal: an entity is a bag of components, and each component contributes one behaviour. You build a flying, shooting, damageable enemy by *attaching* Flight, Weapon, and Health components — not by finding it a place in a taxonomy. It is the duck simulator's fix, applied to millions of objects per frame.

---

## Trade-offs

| | Inheritance | Composition |
|---|---|---|
| **Coupling** | Very tight — subclass depends on base *internals* | Loose — depends only on a small public contract |
| **Reuse granularity** | All-or-nothing (the whole gorilla) | Precise (just the banana) |
| **Runtime flexibility** | None — fixed at `new` | Full — swap behaviours any time |
| **Boilerplate** | Low (free method reuse) | Higher (wire up + forward calls) |
| **Indirection / readability** | Direct — one class to read... | ...but you must jump between objects to trace a call |
| **Testability** | Hard — can't inject a fake superclass | Easy — pass in a fake collaborator |
| **Handles N axes of variation** | Badly — N axes → product explosion | Well — N axes → N small families |
| **`instanceof` / type dispatch** | Works | Doesn't work |
| **Refactoring the base** | Dangerous — fragile base class | Safe — public API only |

**Where composition genuinely costs you:** more objects to wire, more files, more indirection. A stack trace through five composed collaborators is harder to follow than one through a class body. If a thing genuinely has *one* behaviour that will *never* vary, a base class is simpler and you should just use it. Composition applied to non-varying behaviour is over-engineering, and violates YAGNI ([19 — DRY, KISS, YAGNI](./19-dry-kiss-yagni.md)).

**Rule of thumb:** Use inheritance for a **stable IS-A taxonomy** — nouns that will always be that noun (errors, framework base classes, true subtypes). Use composition for **everything that varies, everything that might vary, and anything you're only extending in order to reuse.** When you're genuinely unsure, compose: it is far cheaper to collapse an unnecessary indirection later than it is to untangle a hierarchy 40 subclasses deep.

---

## Common interview questions on this topic

### Q1: "The GoF said 'favor composition over inheritance.' Why?"
**Hint:** Three reasons, name them all. (1) **Fragile base class** — inheritance couples you to the base class's *internals*, not just its API, so a safe-looking base change breaks unknown subclasses. (2) **Runtime rigidity** — a class is fixed at `new`; a composed behaviour can be swapped. (3) **Combinatorial explosion** — with N independent axes of variation, subclass count grows as the *product* of the axes; with composition it grows as the *sum*. Then say the word "Strategy" — that's the pattern that operationalizes it.

### Q2: "Design a duck simulator. Now add a rubber duck that can't fly."
**Hint:** This is the trap. Do *not* override `fly()` to do nothing — say out loud that overriding a method to disable it is a Liskov violation and a design smell. Instead: identify what varies (flight, sound), extract each into a behaviour interface, inject them at construction, and delegate. Then name it: "this is the Strategy pattern." Bonus point: add `setFlyBehavior()` and note that inheritance could never do that.

### Q3: "When would you actually *choose* inheritance?"
**Hint:** Interviewers ask this to see if you're a dogmatist. Give four concrete cases: (1) a true, stable IS-A taxonomy; (2) framework template-method base classes (`extends Transform`, `extends Component`) where the framework owns the algorithm skeleton; (3) error hierarchies, because `instanceof` needs a real prototype chain; (4) library classes you're required to extend. Add the constraint: keep it shallow (1–2 levels) and never override to disable.

### Q4: "What's the 'gorilla holding the banana' problem?"
**Hint:** Joe Armstrong's line about OO. You extend a class to get one method (the banana) and you inherit its entire state, dependency graph, and API surface (the gorilla and the jungle). Give a concrete cost: your unit test for `EmailJob` now needs an HTTP agent mock because the base class constructor builds one. The fix is to inject just the collaborator you need.

### Q5: "What are JS mixins, and what problem do they *not* solve?"
**Hint:** A mixin is `(Base) => class extends Base { ... }` — a function returning a subclass, letting you compose capabilities à la carte and dodge the class explosion. What it does NOT solve: it's still `extends`, so you keep the fragile base class coupling; and two mixins defining the same method name collide silently (last applied wins). Use for cross-cutting additive capabilities (serializable, timestamped); don't use for core varying domain behaviour.

---

## Practice exercise

### Refactor the Notification Hierarchy

You've inherited (pun intended) this real-shaped mess. Someone modelled notifications with inheritance and it has already started to rot:

```javascript
class Notification {
  send(user, msg)  { throw new Error('implement send'); }
  format(msg)      { return msg; }
  retry()          { /* retry 3 times */ }
  logDelivery()    { /* writes to the audit DB */ }
}

class EmailNotification extends Notification { send(u, m) { /* SMTP */ } }
class SmsNotification extends Notification { send(u, m) { /* Twilio */ } }

// Then these appeared:
class UrgentEmailNotification extends EmailNotification { retry() { /* 10 times */ } }
class SilentSmsNotification extends SmsNotification { logDelivery() { /* no-op !! */ } }
class UrgentSilentEmailNotification extends UrgentEmailNotification { logDelivery() { /* no-op !! */ } }
```

**Your task (~30 min). Produce a single JavaScript file containing:**

1. **The diagnosis (as comments at the top).** Name the axes of variation you can see. If a third channel (Push) and a third retry policy (exponential backoff) are added, how many classes does the current design need? Show the multiplication.
2. **The refactor.** One `Notification` class that composes three injected behaviours: a **transport** (`send`), a **retry policy** (`run(fn)`), and a **delivery logger** (`record`). Write at least two concrete implementations of each — including a `NoOpLogger` and a `NoRetry` — so that "silent" and "no retry" become *objects*, never overrides.
3. **A `main()` demo** that builds three notifications (urgent-silent-email, normal SMS, no-retry push), sends each, and then **swaps one notification's retry policy at runtime** and re-sends — proving the thing inheritance could not do.
4. **Two sentences** at the bottom naming which design pattern you just implemented and identifying the Context / Strategy / Concrete Strategy participants in your code.

**Stretch:** write a `LoggingTransport` that *wraps* another transport (`constructor(inner)`) and logs before delegating. Note in a comment which pattern that is.

---

## Quick reference cheat sheet

- **Inheritance = IS-A. Composition = HAS-A / CAN-DO.** Say the relationship out loud; if it's a lie, don't inherit.
- **"Favor composition over class inheritance"** — Gang of Four, 1994. Still the single best OO design heuristic.
- **The killer smell:** a subclass **overriding a method to disable it** (`fly() { /* nothing */ }`). Inheritance is now lying about the type. Refactor immediately.
- **Fragile base class:** subclasses depend on the base's *internals*, not just its API. You cannot safely change a base class you don't fully control.
- **Gorilla/banana:** extending for one method drags in the whole class's state and dependencies. Inject the collaborator instead.
- **Combinatorial explosion:** N independent axes of variation → subclasses grow as the **product**; composed behaviours grow as the **sum**. 3×3×4 = 36 classes vs. 1 class + 10 objects.
- **Composition = Strategy.** "Extract what varies, inject it, delegate to it" IS the Strategy pattern ([42](./42-pattern-strategy.md)).
- **Composition enables runtime swapping.** A class is fixed the moment `new` runs; a field is not.
- **Composition enables testing.** You can pass in a fake collaborator; you cannot pass in a fake superclass.
- **JS mixins** = `(Base) => class extends Base {...}`. Great for additive cross-cutting capabilities; still `extends`, so fragile-base-class risk and silent name collisions remain.
- **Inheritance IS right for:** true stable taxonomies, framework template-method base classes (`extends Transform`), error hierarchies needing `instanceof`, and library classes you must extend.
- **Keep any hierarchy ≤ 2 levels deep.** Three or more levels is a refactor ticket.
- **Don't over-correct.** Composition costs indirection and wiring. If behaviour will never vary, a plain method on a plain class is fine.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [25 — Design by Contract](./25-design-by-contract.md) — the preconditions/postconditions a subclass must not violate; the formal basis for "don't override to disable" |
| **Next** | [27 — Abstraction and Interfaces in Design](./27-abstraction-and-interfaces.md) — how to *find* the right behaviour interface to compose against in the first place |
| **Related** | [23 — Class Relationships](./23-class-relationships.md) — the formal definitions of association, aggregation, composition, and inheritance |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) — the named pattern that this entire doc has been secretly teaching you |
