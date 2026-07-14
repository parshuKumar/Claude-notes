# 42 — Strategy Pattern
## Category: LLD Patterns

---

## What is this?

The **Strategy pattern** lets you define a family of interchangeable algorithms, put each one in its own box, and swap them at runtime without the code that uses them knowing or caring which one it got.

Think of a **power drill**. The drill body is one thing: it grips, it spins, it has a trigger. The *bit* is another thing: wood bit, masonry bit, screwdriver bit. The drill doesn't know what bit is attached — it just spins whatever is in the chuck. Change the bit, change the behaviour, same drill. The drill is the **context**; the bits are the **strategies**.

---

## Why does it matter?

Because the single most common shape of bad code in the world is this:

```javascript
if (type === 'a')      { /* 30 lines */ }
else if (type === 'b') { /* 30 lines */ }
else if (type === 'c') { /* 30 lines */ }
```

Every time the business adds a new `type`, someone reopens that file, adds a branch, and risks breaking the other three. That function grows forever, it can never be unit-tested one branch at a time, and it becomes the file every code review fights over.

**Strategy is the fix.** It is also, by a wide margin, the most-used design pattern in real LLD interviews — payment methods, shipping carriers, pricing rules, sorting orders, compression algorithms, matching algorithms, notification channels, ranking algorithms. Almost every "design X" question has a Strategy hiding in it, and interviewers *expect* you to name it.

**At work:** Strategy is how you make a module extensible by other teams. They add a file; you don't touch yours.

**In interviews:** Naming it correctly, and then correctly distinguishing it from **State** and **Template Method**, is a reliable signal of seniority.

---

## The core idea — explained simply

### The Google Maps "Get Directions" Analogy

You open Google Maps, type a destination, and tap a little icon: car, bus, bike, or walk.

The app's job never changes. It always:
1. Takes your start point and your destination.
2. Computes a route.
3. Draws a blue line on the map and shows an ETA.

But *how* the route is computed is wildly different per mode:
- **Car:** prefer highways, avoid pedestrian zones, factor in live traffic.
- **Bus:** must match bus timetables, walk-to-stop segments, transfers.
- **Bike:** prefer cycle lanes, avoid highways, penalise steep hills.
- **Walk:** ignore one-way streets entirely, allow stairs and footpaths.

Four *completely unrelated* algorithms. Zero shared logic. But they all answer the same question and return the same shape of answer: `{ distance, duration, steps }`.

Now imagine Maps was written as `if (mode === 'car') { ...600 lines... } else if (mode === 'bus') { ... }`. Adding "scooter" means editing the same function that computes bus transfers. Insane.

Instead: each routing algorithm is its own object with the same method. The map screen holds *one* of them and calls it. Tapping the bike icon just swaps which object is held.

| Analogy piece | Pattern role | Name |
|---|---|---|
| The Maps screen (asks for a route, draws the line) | The class that *uses* an algorithm but doesn't implement it | **Context** |
| "Give me a route from A to B" | The shared contract every algorithm honours | **Strategy interface** |
| CarRouter, BusRouter, BikeRouter, WalkRouter | Each individual algorithm, self-contained | **Concrete Strategies** |
| You tapping the bike icon | The code that *picks* which algorithm to use | **The client** |

**The one sentence to remember:** the client picks the strategy, the context runs it, and the context has no idea which one it's running.

---

## Key concepts inside this topic

### 1. The problem it solves — start with the painful code

Here is a real shipping calculator. It starts innocent. It always does.

```javascript
// BAD — the growing if/else monster
function calculateShipping(order, carrier) {
  if (carrier === 'fedex') {
    // FedEx charges by weight, with a fuel surcharge
    let cost = 8.0 + order.weightKg * 2.5;
    if (order.isExpress) cost += 15.0;
    cost += cost * 0.06;                       // fuel surcharge
    return Number(cost.toFixed(2));

  } else if (carrier === 'ups') {
    // UPS charges by dimensional weight, whichever is greater
    const dimWeight = (order.lengthCm * order.widthCm * order.heightCm) / 5000;
    const billable = Math.max(order.weightKg, dimWeight);
    let cost = 6.5 + billable * 3.1;
    if (order.isExpress) cost *= 1.8;
    return Number(cost.toFixed(2));

  } else if (carrier === 'dhl') {
    // DHL charges by international zone
    const zoneRate = { 1: 12, 2: 18, 3: 26, 4: 34 }[order.zone] ?? 40;
    let cost = zoneRate + order.weightKg * 4.0;
    if (order.destinationCountry !== 'US') cost += 9.0;   // customs handling
    return Number(cost.toFixed(2));

  } else if (carrier === 'usps') {
    // ... 20 more lines
  } else if (carrier === 'local_courier') {
    // ... 20 more lines
  }
  throw new Error(`Unknown carrier: ${carrier}`);
}
```

Count the wounds:

1. **It violates the Open/Closed Principle** (recall from [15 — Open/Closed Principle](./15-solid-open-closed.md): *open for extension, closed for modification*). Adding "Royal Mail" means **editing this file**. You cannot extend it without modifying it. Every edit re-risks all five existing carriers, which is why this file needs a full regression test run for a change that logically touches nothing.
2. **It's untestable in isolation.** To test the DHL zone table you must call `calculateShipping`, which means constructing a whole `order` and passing a magic string. You can never test "the DHL algorithm" as a unit — only "the calculator, in DHL mode".
3. **It has zero cohesion.** Five unrelated algorithms, sharing nothing but a function body. Fuel surcharges and customs handling and dimensional weight all live in one scope. (Recall from [24 — Coupling and Cohesion](./24-coupling-and-cohesion.md): a function should do one thing.)
4. **The `if` chain is duplicated elsewhere.** Guaranteed. Somewhere there's a `getCarrierEta(order, carrier)` and a `getTrackingUrl(carrier)` with the exact same five branches, and one day someone will add a carrier to two of the three.
5. **Merge conflicts.** Five teams, one file.

### 2. The structure — who plays which role

```
Strategy (interface)   — declares the method every algorithm must have.
ConcreteStrategy       — one class per algorithm. Self-contained. Independently testable.
Context                — holds a Strategy and delegates to it. Never uses `if` on type.
Client                 — chooses the ConcreteStrategy and hands it to the Context.
```

The critical, non-obvious rule: **the Context must not know the list of strategies.** The moment your Context contains `if (strategy instanceof FedExStrategy)`, you have rebuilt the if/else you were escaping.

### 3. Full JavaScript implementation — the class-based refactor

JavaScript has no `interface` keyword, so the idiomatic stand-in is a base class whose methods throw. (Recall from [18 — Dependency Inversion](./18-solid-dependency-inversion.md): depend on the abstraction, not the concretion.)

```javascript
// shipping/ShippingStrategy.js
// The contract. Every carrier algorithm must honour exactly this shape.
export class ShippingStrategy {
  /** @returns {number} cost in USD */
  calculate(order) {
    throw new Error('ShippingStrategy.calculate() must be implemented');
  }

  /** @returns {number} estimated days in transit */
  estimateDays(order) {
    throw new Error('ShippingStrategy.estimateDays() must be implemented');
  }

  get name() {
    throw new Error('ShippingStrategy.name must be implemented');
  }
}
```

```javascript
// shipping/FedExStrategy.js
import { ShippingStrategy } from './ShippingStrategy.js';

export class FedExStrategy extends ShippingStrategy {
  // Config lives WITH the algorithm that uses it — not in a global constants file.
  constructor({ baseRate = 8.0, perKg = 2.5, fuelSurchargePct = 0.06 } = {}) {
    super();
    this.baseRate = baseRate;
    this.perKg = perKg;
    this.fuelSurchargePct = fuelSurchargePct;
  }

  get name() { return 'fedex'; }

  calculate(order) {
    let cost = this.baseRate + order.weightKg * this.perKg;
    if (order.isExpress) cost += 15.0;
    cost += cost * this.fuelSurchargePct;
    return round2(cost);
  }

  estimateDays(order) {
    return order.isExpress ? 1 : 3;
  }
}

const round2 = (n) => Number(n.toFixed(2));
```

```javascript
// shipping/UpsStrategy.js
import { ShippingStrategy } from './ShippingStrategy.js';

export class UpsStrategy extends ShippingStrategy {
  constructor({ divisor = 5000 } = {}) {
    super();
    this.divisor = divisor;   // UPS's dimensional-weight divisor, changes yearly
  }

  get name() { return 'ups'; }

  calculate(order) {
    // UPS bills on the GREATER of actual weight and dimensional weight.
    const dimWeight = (order.lengthCm * order.widthCm * order.heightCm) / this.divisor;
    const billable = Math.max(order.weightKg, dimWeight);
    let cost = 6.5 + billable * 3.1;
    if (order.isExpress) cost *= 1.8;
    return Number(cost.toFixed(2));
  }

  estimateDays(order) {
    return order.isExpress ? 2 : 5;
  }
}
```

```javascript
// shipping/DhlStrategy.js
import { ShippingStrategy } from './ShippingStrategy.js';

const ZONE_RATES = Object.freeze({ 1: 12, 2: 18, 3: 26, 4: 34 });

export class DhlStrategy extends ShippingStrategy {
  get name() { return 'dhl'; }

  calculate(order) {
    const zoneRate = ZONE_RATES[order.zone] ?? 40;
    let cost = zoneRate + order.weightKg * 4.0;
    if (order.destinationCountry !== 'US') cost += 9.0;  // customs handling fee
    return Number(cost.toFixed(2));
  }

  estimateDays(order) {
    return order.destinationCountry === 'US' ? 4 : 8;
  }
}
```

Now the **Context** — and notice there is not a single `if` about carriers in it:

```javascript
// shipping/ShippingCalculator.js
import { ShippingStrategy } from './ShippingStrategy.js';

export class ShippingCalculator {
  #strategy;

  constructor(strategy) {
    this.setStrategy(strategy);
  }

  // Runtime swappability is the whole point — this is what makes it Strategy
  // and not just "polymorphism".
  setStrategy(strategy) {
    if (!(strategy instanceof ShippingStrategy)) {
      throw new TypeError('strategy must be a ShippingStrategy');
    }
    this.#strategy = strategy;
    return this;
  }

  quote(order) {
    return {
      carrier: this.#strategy.name,
      cost: this.#strategy.calculate(order),
      etaDays: this.#strategy.estimateDays(order),
    };
  }
}
```

Using it:

```javascript
// main.js
import { ShippingCalculator } from './shipping/ShippingCalculator.js';
import { FedExStrategy } from './shipping/FedExStrategy.js';
import { UpsStrategy } from './shipping/UpsStrategy.js';
import { DhlStrategy } from './shipping/DhlStrategy.js';

const order = {
  weightKg: 2.4,
  lengthCm: 40, widthCm: 30, heightCm: 20,
  isExpress: true,
  zone: 2,
  destinationCountry: 'DE',
};

const calculator = new ShippingCalculator(new FedExStrategy());
console.log(calculator.quote(order));
// { carrier: 'fedex', cost: 22.26, etaDays: 1 }

// Swap the algorithm at RUNTIME. Same context object, different behaviour.
calculator.setStrategy(new UpsStrategy());
console.log(calculator.quote(order));
// { carrier: 'ups', cost: 38.51, etaDays: 2 }

// Compare all carriers — trivially, because they share one contract.
const all = [new FedExStrategy(), new UpsStrategy(), new DhlStrategy()]
  .map((s) => new ShippingCalculator(s).quote(order))
  .sort((a, b) => a.cost - b.cost);

console.table(all);   // cheapest first — impossible to write cleanly with if/else
```

**Now add a new carrier.** Royal Mail:

```javascript
// shipping/RoyalMailStrategy.js   <-- ONE NEW FILE
import { ShippingStrategy } from './ShippingStrategy.js';

export class RoyalMailStrategy extends ShippingStrategy {
  get name() { return 'royal_mail'; }
  calculate(order)    { return Number((4.2 + order.weightKg * 1.9).toFixed(2)); }
  estimateDays(order) { return 5; }
}
```

Files edited: **zero**. `ShippingCalculator` was not touched. `FedExStrategy` was not touched. No existing test can possibly break. *That* is the Open/Closed Principle actually happening, not just being talked about.

### 4. Second example — pluggable pricing

The same shape, different domain. Notice each strategy here carries its own **state/config**, which is exactly when classes beat plain functions.

```javascript
// pricing/PricingStrategy.js
export class PricingStrategy {
  apply(basePrice, ctx) { throw new Error('not implemented'); }
}

export class RegularPricing extends PricingStrategy {
  apply(basePrice) { return basePrice; }
}

export class MemberPricing extends PricingStrategy {
  // Tier config is STATE the strategy owns. A bare function couldn't hold this
  // without a closure or a global.
  constructor(tierDiscounts = { bronze: 0.05, silver: 0.10, gold: 0.20 }) {
    super();
    this.tierDiscounts = tierDiscounts;
  }
  apply(basePrice, { customerTier }) {
    const pct = this.tierDiscounts[customerTier] ?? 0;
    return basePrice * (1 - pct);
  }
}

export class SeasonalPricing extends PricingStrategy {
  constructor(discountPct, startsAt, endsAt) {
    super();
    this.discountPct = discountPct;
    this.startsAt = startsAt;
    this.endsAt = endsAt;
  }
  apply(basePrice, { now = new Date() } = {}) {
    const active = now >= this.startsAt && now <= this.endsAt;
    return active ? basePrice * (1 - this.discountPct) : basePrice;
  }
}

export class ClearancePricing extends PricingStrategy {
  // Clearance is aggressive but floored — never sell below cost.
  constructor(floorPrice) { super(); this.floorPrice = floorPrice; }
  apply(basePrice) { return Math.max(basePrice * 0.4, this.floorPrice); }
}
```

```javascript
// pricing/Product.js — the Context
export class Product {
  constructor(name, basePrice, pricingStrategy) {
    this.name = name;
    this.basePrice = basePrice;
    this.pricing = pricingStrategy;
  }
  finalPrice(ctx = {}) {
    return Number(this.pricing.apply(this.basePrice, ctx).toFixed(2));
  }
}

const headphones = new Product('Headphones', 200, new RegularPricing());
console.log(headphones.finalPrice());                            // 200.00

headphones.pricing = new MemberPricing();
console.log(headphones.finalPrice({ customerTier: 'gold' }));    // 160.00

headphones.pricing = new ClearancePricing(90);
console.log(headphones.finalPrice());                            // 90.00 (floor wins over 80)
```

### 5. The JS-idiomatic version — a strategy is often just a function

Here is the honest truth that most pattern tutorials (written for Java) will not tell you:

**In JavaScript, a strategy is usually just a function, and the family is usually just an object.**

Functions are first-class. You do not need a class with one method to pass behaviour around — you need a function.

```javascript
// shipping/strategies.js
const round2 = (n) => Number(n.toFixed(2));

export const shippingStrategies = {
  fedex: (o) => round2((8.0 + o.weightKg * 2.5 + (o.isExpress ? 15 : 0)) * 1.06),

  ups: (o) => {
    const dim = (o.lengthCm * o.widthCm * o.heightCm) / 5000;
    const billable = Math.max(o.weightKg, dim);
    const cost = 6.5 + billable * 3.1;
    return round2(o.isExpress ? cost * 1.8 : cost);
  },

  dhl: (o) => {
    const zoneRate = { 1: 12, 2: 18, 3: 26, 4: 34 }[o.zone] ?? 40;
    const customs = o.destinationCountry !== 'US' ? 9 : 0;
    return round2(zoneRate + o.weightKg * 4.0 + customs);
  },
};

export function quote(order, carrier) {
  const strategy = shippingStrategies[carrier];
  if (!strategy) throw new Error(`Unknown carrier: ${carrier}`);
  return strategy(order);
}
```

That is a complete, correct Strategy pattern. The object literal *is* the family. The lookup key *is* the selection. Each function *is* independently testable (`import { shippingStrategies } from ...; expect(shippingStrategies.dhl(order)).toBe(35)`). Adding a carrier is one new entry.

**Be honest in the interview:** say *"In JavaScript I'd normally reach for a map of functions — that's the same pattern with less ceremony. I'd escalate to classes when a strategy needs its own state or config, needs more than one method, or needs a lifecycle."*

The class version earns its keep when:

| Situation | Function | Class |
|---|---|---|
| One pure computation, no config | ✅ Best | Overkill |
| Strategy needs injected config (`new UpsStrategy({ divisor: 6000 })`) | Needs a factory/closure | ✅ Natural |
| Strategy has 2+ methods (`calculate` + `estimateDays` + `trackingUrl`) | Gets awkward | ✅ Natural |
| Strategy holds mutable state (a cache, a counter, an open connection) | ❌ Wrong tool | ✅ Right tool |
| Strategy needs a dependency injected (an HTTP client, a rate API) | Closure works | ✅ Cleaner |

A middle ground that JS people love — a **factory-of-functions**, which gives you config without classes:

```javascript
const makeUpsStrategy = ({ divisor = 5000 } = {}) => (o) => {
  const dim = (o.lengthCm * o.widthCm * o.heightCm) / divisor;
  return round2(6.5 + Math.max(o.weightKg, dim) * 3.1);
};

const strategies = { ups: makeUpsStrategy({ divisor: 6000 }) };
```

### 6. The self-registering registry — full OCP, no central edit

The function-map version still has one flaw: the map lives in one file, so adding a strategy means *editing* that file. Close, but not quite Open/Closed. A **registry** closes the gap.

```javascript
// shipping/registry.js
class StrategyRegistry {
  #strategies = new Map();

  register(name, strategy) {
    if (this.#strategies.has(name)) {
      throw new Error(`Strategy "${name}" is already registered`);
    }
    this.#strategies.set(name, strategy);
    return this;
  }

  get(name) {
    const s = this.#strategies.get(name);
    if (!s) {
      throw new Error(
        `Unknown carrier "${name}". Available: ${[...this.#strategies.keys()].join(', ')}`
      );
    }
    return s;
  }

  list() { return [...this.#strategies.keys()]; }

  // Because every strategy shares one contract, the registry can do things
  // the if/else version never could — like quote ALL of them at once.
  quoteAll(order) {
    return [...this.#strategies.entries()]
      .map(([name, s]) => ({ carrier: name, cost: s.calculate(order) }))
      .sort((a, b) => a.cost - b.cost);
  }
}

export const shippingRegistry = new StrategyRegistry();
```

```javascript
// shipping/FedExStrategy.js  (bottom of the file — the strategy registers ITSELF)
import { shippingRegistry } from './registry.js';
shippingRegistry.register('fedex', new FedExStrategy());
```

```javascript
// shipping/index.js — the only file that knows the strategies exist
import './FedExStrategy.js';
import './UpsStrategy.js';
import './DhlStrategy.js';
import './RoyalMailStrategy.js';   // <-- the ONLY line added for a new carrier
export { shippingRegistry } from './registry.js';
```

```javascript
// usage
import { shippingRegistry } from './shipping/index.js';

const best = shippingRegistry.quoteAll(order)[0];
console.log(`Cheapest: ${best.carrier} at $${best.cost}`);
console.log(`Supported: ${shippingRegistry.list().join(', ')}`);
```

Now business logic never edits, never grows an `if`. In a bigger app you'd auto-load the directory (`fs.readdirSync('./shipping')` + dynamic `import()`) and even that one import line disappears — dropping a file into a folder adds a carrier. That is what "plugin architecture" means, and it is Strategy all the way down.

### 7. A real Node.js ecosystem example

Strategy is everywhere in Node, usually without the name attached.

**Passport.js** is the textbook case. `passport.use(new LocalStrategy(...))`, `passport.use(new GoogleStrategy(...))`, `passport.use(new JwtStrategy(...))`. The library literally calls them Strategies. Passport is the Context: it knows *"authenticate this request, then call `done(err, user)`"*. It has no idea whether that meant checking a bcrypt hash or doing an OAuth redirect. `passport.authenticate('google')` is the client selecting a strategy by name from a registry — exactly section 6.

**`Array.prototype.sort(comparator)`** — the comparator you pass is a strategy. `sort` is the context; it owns the invariant (the sorting algorithm, the swapping); you inject the *comparison* algorithm. `[...].sort((a,b) => a.price - b.price)` vs `(a,b) => b.rating - a.rating)` — same context, swapped strategy, at runtime.

**`crypto.createHash('sha256')`** and **`zlib.createGzip()` vs `zlib.createBrotliCompress()`** — a family of interchangeable algorithms behind one stream interface, selected by the caller.

**Winston / Pino transports and formatters** — you compose a logger from a chosen format strategy and one or more destination strategies.

### 8. When NOT to use it / how it's abused

**Don't use Strategy when:**

- **There are exactly two branches and they will never grow.** `isWeekend ? weekendRate : weekdayRate` is fine. Four files to replace a ternary is not a win, it's a career in indirection. YAGNI (recall [19 — DRY, KISS, YAGNI](./19-dry-kiss-yagni.md)).
- **The "algorithms" share 90% of their code.** Then you've got a Template Method (topic 45) or a shared helper, not a family of distinct algorithms. Strategy is for algorithms that are genuinely *different*, not variations on a theme.
- **The variation is data, not behaviour.** If every carrier is `base + perKg * weight` and only the two numbers differ, you don't need five classes — you need a config table: `{ fedex: { base: 8, perKg: 2.5 }, ... }`. Strategy is for differing *logic*. Don't build a class hierarchy to hold two numbers.

**How it's abused:**

- **The leaky context.** `context.execute()` does `if (this.strategy.name === 'dhl') { addCustoms() }`. You've reintroduced the if/else *and* added indirection. Every carrier-specific detail belongs inside its strategy.
- **The fat interface.** The base class grows `calculate`, `estimateDays`, `trackingUrl`, `supportsWeekend`, `requiresCustomsForm`... and now `RoyalMailStrategy` has to implement six methods it doesn't care about, half of them returning `null`. That's an [17 — Interface Segregation](./17-solid-interface-segregation.md) violation. Keep the strategy interface *narrow*.
- **Strategy where a plain function would do.** In JS, wrapping a one-line pure function in a class with a constructor, a base class, and a `.execute()` is cargo-culting Java. Don't.

### 9. Related patterns and how they differ — the #1 interview follow-up

This is the section that gets you hired. Interviewers ask it every single time.

**Strategy vs [State](./46-pattern-state.md)** — *structurally identical, completely different intent.* Both are "a context holding a pluggable object it delegates to". The difference:

| | Strategy | State |
|---|---|---|
| **Who chooses the object?** | The **client**, from outside | The **state object itself**, from inside |
| **Does it change during a task?** | No — you pick FedEx and it stays FedEx | Yes — that's the entire point |
| **Do the objects know each other?** | No — `FedExStrategy` has never heard of `UpsStrategy` | Yes — `PendingState` knows the next state is `PaidState` |
| **Intent** | "Vary *how* one thing is done" | "Vary *what* is possible as the object moves through a lifecycle" |
| **Example** | Shipping cost algorithm | Order lifecycle: PENDING → PAID → SHIPPED → DELIVERED |

The killer line: **"A Strategy is handed to the context and never replaces itself. A State replaces itself — `this.order.setState(new PaidState())` — that self-transition is what makes it State."**

**Strategy vs [Template Method](./45-pattern-template-method.md)** — both vary parts of an algorithm.

| | Strategy | Template Method |
|---|---|---|
| **Mechanism** | Composition — the context *has-a* strategy | Inheritance — the subclass *is-a* base |
| **What varies** | The **whole** algorithm | Individual **steps**; the skeleton is fixed |
| **When it's chosen** | Runtime — swap any time | Design time — baked into the class |
| **Coupling** | Loose | Tight (base ↔ subclass) |

Template Method says *"the order of steps is fixed, fill in the steps."* Strategy says *"here, take this entire algorithm."*

**Strategy vs [Bridge](./39-pattern-bridge.md)** — Bridge looks like Strategy (an abstraction holding an implementation) but its intent is structural: Bridge exists to stop a *class explosion* along two independent axes (e.g. 3 shapes × 4 renderers = 12 classes → 3 + 4 = 7). Bridge's implementation object is usually set once at construction and represents a *platform*; Strategy's is swapped freely and represents a *behaviour*. Rule of thumb: **Bridge is about structure and is decided up front; Strategy is about behaviour and is decided at runtime.**

**Strategy vs [Decorator](./35-pattern-decorator.md)** — Decorator *wraps* and *adds to* behaviour (each layer calls the next). Strategy *replaces* behaviour entirely (only one runs).

**Strategy + [Factory](./30-pattern-factory-method.md)** — they pair constantly. Something has to turn the string `"fedex"` into a `FedExStrategy` object. That something is a Factory (or the registry in section 6). Saying *"I'd use a Factory to select the Strategy from the carrier code"* in an interview shows you understand how patterns compose.

---

## Visual / Diagram description

### Diagram 1: The Strategy class diagram

```
                    ┌──────────────────────────────┐
   client picks     │      ShippingCalculator      │
   the strategy     │         «Context»            │
        │           ├──────────────────────────────┤
        │           │ - strategy: ShippingStrategy │◆──────┐
        └──────────▶│ + setStrategy(s)             │       │  HAS-A
                    │ + quote(order)               │       │ (composition)
                    └──────────────────────────────┘       │
                                                           │
                                                           ▼
                              ┌────────────────────────────────────────┐
                              │           ShippingStrategy             │
                              │        «abstract / interface»          │
                              ├────────────────────────────────────────┤
                              │ + calculate(order): number   {abstract}│
                              │ + estimateDays(order): number{abstract}│
                              └────────────────────────────────────────┘
                                                 △
                                                 │  IS-A (implements)
                     ┌───────────────────────────┼───────────────────────────┐
                     │                           │                           │
        ┌────────────┴───────┐      ┌────────────┴───────┐      ┌────────────┴───────┐
        │   FedExStrategy    │      │    UpsStrategy     │      │    DhlStrategy     │
        ├────────────────────┤      ├────────────────────┤      ├────────────────────┤
        │ - baseRate         │      │ - divisor          │      │ - zoneRates        │
        │ + calculate()      │      │ + calculate()      │      │ + calculate()      │
        │ + estimateDays()   │      │ + estimateDays()   │      │ + estimateDays()   │
        └────────────────────┘      └────────────────────┘      └────────────────────┘
                                                 △
                                                 │  new carrier = ONE new class,
                                    ┌────────────┴───────┐   zero edits above
                                    │ RoyalMailStrategy  │
                                    └────────────────────┘
```

`◆──` is composition ("Context HAS-A Strategy"). `△──` is inheritance ("FedEx IS-A ShippingStrategy"). The arrow from Context points at the **abstraction**, never at a concrete class — that's Dependency Inversion, and it's why the diamond of subclasses can grow forever without the Context noticing.

### Diagram 2: Sequence — swapping at runtime

```
 Client            ShippingCalculator        FedExStrategy      UpsStrategy
   │                       │                      │                  │
   │ new(FedExStrategy)    │                      │                  │
   ├──────────────────────▶│                      │                  │
   │                       │                      │                  │
   │ quote(order)          │                      │                  │
   ├──────────────────────▶│  calculate(order)    │                  │
   │                       ├─────────────────────▶│                  │
   │                       │◀─────────────────────┤  22.26           │
   │◀──────────────────────┤  {fedex, 22.26, 1d}  │                  │
   │                       │                      │                  │
   │ setStrategy(Ups)      │                      │                  │
   ├──────────────────────▶│ ═══ strategy swapped ══════════════════▶│
   │                       │                      │                  │
   │ quote(order)          │                      │  calculate(order)│
   ├──────────────────────▶├─────────────────────────────────────────▶
   │                       │◀─────────────────────────────────────────┤ 38.51
   │◀──────────────────────┤  {ups, 38.51, 2d}    │                  │
   │                       │                      │                  │
```

Read it left to right: the Client never talks to a strategy directly. It hands one to the Calculator and asks for a quote. Between the two `quote()` calls **the Calculator's code did not change** — only the object it holds did. That is "swap the algorithm at runtime", drawn.

---

## Real world examples

### 1. Passport.js — Strategy is literally the API

Passport, the Node authentication middleware used by a huge slice of the Express ecosystem, is built entirely on this pattern and names it explicitly. There are 500+ published strategies (`passport-local`, `passport-google-oauth20`, `passport-jwt`, `passport-saml`). Each one implements a single method — `authenticate(req, options)` — and calls `this.success(user)` or `this.fail()`. Passport core is the Context: it maintains a `Map` of registered strategies (`passport.use(name, strategy)`), and `passport.authenticate('google')` is a registry lookup that returns Express middleware. Passport core has never been modified to support a new auth provider. That's Open/Closed in production, on npm, for a decade.

### 2. Uber — matching and pricing algorithms

Uber's dispatch system must select a driver for a rider, and the *rule* for "best driver" differs by product and geography: nearest-driver, batched-matching (wait a few seconds and solve an assignment problem across many riders at once), pool-matching (optimise for shared routes), and airport-queue (FIFO by driver arrival time at the taxi lot). These are genuinely different algorithms behind one contract — `match(riders, drivers) → assignments`. Likewise pricing composes surge multipliers, promotions, and fixed upfront pricing as swappable rules. A new city launching a new product configures a different strategy; the dispatch pipeline itself is untouched.

### 3. Compression and hashing in Node core

`zlib.createGzip()`, `zlib.createDeflate()`, `zlib.createBrotliCompress()` all return a Transform stream with the same interface. Your HTTP server picks one based on the client's `Accept-Encoding` header and pipes into it — the piping code has no `if (encoding === 'br')` in its body, just a lookup. Similarly, `crypto.createHash(algo)` selects from a family of digest algorithms by name at runtime. Both are Strategy-by-registry: the algorithm family is open, the code that uses it is closed.

---

## Trade-offs

| Benefit | What it buys you |
|---|---|
| **Open/Closed in practice** | New behaviour = new file. Existing files, and their tests, are untouched. |
| **Unit-testable algorithms** | `new DhlStrategy().calculate(order)` — no context, no mocks, no magic strings. |
| **Kills the if/else chain** | One conditional at selection time, and none in the business logic. |
| **Runtime swappability** | A/B tests, feature flags, per-tenant behaviour, config-driven pricing. |
| **Parallel work** | Five teams, five files, zero merge conflicts. |
| **Composability** | Because they share a contract, you can loop over the whole family (`quoteAll`). |

| Cost | What it hurts |
|---|---|
| **More files/objects** | 3 carriers → 5 files instead of 1 function. Real cost in a small codebase. |
| **The client must know the strategies** | *Somebody* has to pick. You've moved the `if`, not deleted it — usually into a factory or a registry, which is a much better place for it. |
| **Indirection cost when reading** | "Where does the cost actually get computed?" now needs a jump. IDEs help; new joiners still feel it. |
| **Wrong abstraction is expensive** | If the strategy interface turns out to be wrong (`calculate` needed to be async), you change every strategy. Design the contract carefully. |
| **Over-application** | Two branches wrapped in four classes is a net loss. |

**Rule of thumb:** Reach for Strategy the **second** time you write `else if` on a type — not the first. And in JavaScript, start with a **map of functions**; promote to classes only when a strategy needs its own state, config, dependencies, or a second method. If you're in an interview, say both sentences out loud.

---

## Common interview questions on this topic

### Q1: "What's the difference between the Strategy pattern and the State pattern? They look identical."
**Hint:** They *are* structurally identical — a context delegating to a swappable object. The difference is **intent and who does the swapping**. Strategy: the client picks the algorithm from outside, and the strategy never changes itself; the strategies don't know each other. State: the object transitions *itself* to the next state from inside (`order.setState(new ShippedState())`), so the states know each other and form a graph. Strategy varies *how* one operation is done; State varies *what operations are legal* as an object moves through a lifecycle. If someone drew the diagrams with the labels removed, you'd need the intent to tell them apart.

### Q2: "You've replaced the if/else with a Strategy — but something still has to choose which strategy. Haven't you just moved the if/else?"
**Hint:** Yes, and that's the point — you've moved it from *deep inside the business logic* (where it grows and mixes with unrelated code) to *one place at the boundary* (a factory, a registry, or a config lookup). And you can delete it entirely: a `Map` lookup or a self-registering registry means adding a strategy touches no shared code at all. The five 30-line algorithm bodies are now unconditional, and *that's* what was actually killing you.

### Q3: "In JavaScript, would you actually build the class hierarchy?"
**Hint:** Usually no. Say so — it's a maturity signal. In JS the idiomatic Strategy is a plain object or `Map` of functions: `const strategies = { fedex: o => ..., ups: o => ... }`. It's the same pattern with no ceremony, and each function is trivially testable. Escalate to classes when a strategy needs (a) injected config, (b) more than one method, (c) mutable state, or (d) injected dependencies. A factory-of-functions (`makeUpsStrategy({ divisor })`) covers the middle ground.

### Q4: "How is Strategy different from Template Method?"
**Hint:** Composition vs inheritance. Strategy swaps the **whole algorithm** at **runtime** via a `has-a` relationship. Template Method fixes the **skeleton** at **design time** in a base class and lets subclasses override **individual steps** via `is-a`. Template Method locks the order of steps in one place (that's its payoff); Strategy locks nothing. If the algorithms share a fixed sequence of steps → Template Method. If they're genuinely different algorithms → Strategy. And per [26 — Composition Over Inheritance](./26-composition-over-inheritance.md), prefer Strategy when in doubt.

### Q5: "When would you NOT use Strategy?"
**Hint:** (1) Two branches that will never grow — a ternary is fine, YAGNI. (2) The "algorithms" differ only in *data* (a base rate and a per-kg rate) — use a config table, not five classes. (3) They share 90% of their logic — that's Template Method or a shared helper. (4) When you'd need a class per one-line pure function in JS — just use functions.

### Q6: "A new carrier needs to hit an external rate API and it's async. What breaks?"
**Hint:** The contract. `calculate(order): number` becomes `async calculate(order): Promise<number>`, which forces a change in *every* strategy and in the Context. That's the real cost of Strategy: the interface is a commitment. Good answer: make the contract async from day one if any implementation might ever be I/O-bound (`await Promise.resolve(x)` on a sync strategy costs a microtask, nothing more), and add caching/timeouts *inside* the async strategy or in a decorator wrapping it — not in the Context.

---

## Practice exercise

### Build a pluggable notification system (~30-40 min)

A `NotificationService` must send a message to a user. Today it does:

```javascript
function notify(user, message, channel) {
  if (channel === 'email') { /* build MIME, call SMTP */ }
  else if (channel === 'sms') { /* truncate to 160 chars, call Twilio */ }
  else if (channel === 'push') { /* build FCM payload, call FCM */ }
  else if (channel === 'slack') { /* build blocks JSON, POST webhook */ }
}
```

**Produce all four of these:**

1. **The class-based Strategy version.** A `NotificationStrategy` base class with an **async** `send(user, message)` and a `canHandle(user)` method (SMS can't send to a user with no phone number). Concrete strategies: `EmailStrategy`, `SmsStrategy` (must truncate to 160 chars — that logic belongs *inside* SMS, nowhere else), `PushStrategy`, `SlackStrategy`. A `NotificationService` context with `setStrategy()` and `async notify(user, message)`. Fake the actual sending with `console.log` + `await new Promise(r => setTimeout(r, 50))`.

2. **The function-map version** of the same thing. Write both, then write two sentences comparing them: which would you actually ship, and why?

3. **A self-registering registry** with a `sendToAll(user, message)` method that fans out to every strategy where `canHandle(user)` is true. Then add a fifth channel — `WebhookStrategy` — and prove to yourself you edited **zero** existing files (except the one `import` line in `index.js`).

4. **One unit test** (plain `node:test` or `assert`) that verifies `SmsStrategy` truncates a 300-character message to 160 — without constructing a `NotificationService` at all. That test being possible *is* the payoff.

**Bonus:** add a `FallbackStrategy` that takes an ordered array of other strategies and tries each until one succeeds. Notice that it *is* a strategy itself (it implements `send`), so the Context can't tell the difference. That's the Composite pattern (topic 38) sneaking in — and it only works because everything shares one contract.

---

## Quick reference cheat sheet

- **Strategy** = a family of interchangeable algorithms, each in its own object, swappable at runtime.
- **Three roles:** **Context** (holds and delegates), **Strategy** (the contract), **ConcreteStrategy** (each algorithm).
- **The smell it cures:** a growing `if (type === ...) else if (...)` chain over a *type* — especially one duplicated in several functions.
- **The payoff:** adding behaviour = **one new file, zero edits**. That is [15 — Open/Closed](./15-solid-open-closed.md) made real.
- **The Context must never `if` on the strategy type.** If it does, you've rebuilt the thing you were escaping.
- **In JavaScript, a strategy is usually just a function** and the family is a plain object or `Map`. Say this in the interview.
- **Promote to classes when** the strategy needs config, state, dependencies, or more than one method.
- **Registry = full OCP.** Strategies register themselves; nothing central is ever edited.
- **Strategy vs State:** identical structure. The **client** picks a Strategy and it never changes itself; a **State** transitions *itself* to the next State.
- **Strategy vs Template Method:** composition + whole algorithm + runtime, vs inheritance + fixed skeleton + design time.
- **Strategy vs Bridge:** Bridge is structural (kill a class explosion, chosen up front); Strategy is behavioural (chosen at runtime).
- **Strategy pairs with Factory:** the Factory turns `"fedex"` into a `FedExStrategy`.
- **Keep the strategy interface narrow** — a fat base class forces empty methods on every implementation.
- **Real Node:** Passport strategies, `Array.sort(comparator)`, `zlib.createGzip/createBrotliCompress`, `crypto.createHash(algo)`, Winston transports.
- **Don't use it** for two never-growing branches, or when only *data* varies — use a config table.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [41 — Observer Pattern](./41-pattern-observer.md) — the other behavioural workhorse; Observer broadcasts events, Strategy swaps algorithms |
| **Next** | [43 — Command Pattern](./43-pattern-command.md) — encapsulates a *request* as an object, the way Strategy encapsulates an *algorithm* |
| **Related** | [46 — State Pattern](./46-pattern-state.md) — the same structure with the opposite intent; the #1 interview follow-up to Strategy |
| **Related** | [45 — Template Method Pattern](./45-pattern-template-method.md) — the inheritance-based alternative; fixed skeleton, varying steps |
| **Related** | [15 — SOLID: Open/Closed Principle](./15-solid-open-closed.md) — the principle Strategy exists to satisfy |
| **Related** | [39 — Bridge Pattern](./39-pattern-bridge.md) — looks like Strategy, but is structural: it kills a class explosion |
