# 32 — Builder Pattern

## Category: LLD Patterns

---

## What is this?

The **Builder pattern** lets you construct a complex object **step by step**, instead of shoving everything into one giant constructor call. You collect the pieces one at a time, then say "okay, I'm done — give me the finished object."

Think of ordering at Subway. You don't shout `new Sandwich('footlong', 'italian herb', true, false, true, null, 'chipotle', false)` at the person behind the counter. You walk down the line: *bread… cheese… toasted… veggies… sauce…* and at the end they wrap it. Each step is named, the order is enforced (bread first, wrap last), and if you ask for a sauce with no bread, they stop you. That walk down the line **is** the Builder pattern, and the wrap at the end is `.build()`.

---

## Why does it matter?

Without Builder you get the **telescoping constructor problem**: a constructor that keeps growing optional parameters until nobody can read a call site. You end up writing `null, null, true, false` just to reach the one argument you care about, and a single misplaced `true` silently produces a wrong object that fails hours later, far from the bug.

**At work:** every time you touch a library that lets you chain calls — `knex('users').where('age', '>', 18).orderBy('name').limit(10)` — you're using a Builder. When you write one yourself it's usually for query construction, HTTP requests, complex config, or test fixtures (`aUser().withRole('admin').build()` makes tests readable).

**In interviews:** Builder shows up in almost every LLD "creational patterns" question, and the killer follow-up is always *"in JavaScript you have object literals — why would you ever write a Builder?"* Answering that honestly (rather than reciting the Gang of Four) separates a candidate who memorized patterns from one who understands them. This doc gives you that answer.

---

## The core idea — explained simply

### The Subway Sandwich Analogy

**Way 1 — the telescoping constructor.** You walk in and yell the whole order as one positional blast:

> "Footlong, italian-herb, provolone, toasted, no-lettuce, tomato, no-onion, chipotle, no-oil!"

The person behind the counter must remember the exact **order** of every slot. Miss one and everything after it shifts — you asked for tomato and got onion. Want to skip cheese but keep toasting? You still have to say "no cheese" out loud, just to hold the slot. This is `new Sandwich('footlong', 'italian', null, true, false, ...)`.

**Way 2 — the builder.** You walk down the line and say one thing at a time:

> bread → cheese → toast it → veggies → sauce → **wrap**

Every step is **named**, so slot order doesn't matter. Steps you don't want, you skip. And here's the important bit: **the wrap at the end is where validation happens.** If you never picked bread, the wrap step refuses. If you asked for double meat on a 3-inch, the wrap step refuses. Nothing incomplete leaves the counter.

**Mapping the analogy back to code:**

| Sandwich shop | Builder pattern | In code |
|---|---|---|
| The counter you walk along | The **Builder** object | `new SandwichBuilder()` |
| Each station (bread, cheese, sauce) | A **setter method** that returns the builder | `.bread('italian')` returns `this` |
| Walking to the next station | **Method chaining** | `.bread(...).cheese(...)` |
| Wrapping it up | The **`build()`** method | `.build()` |
| Counter staff refusing an invalid order | **Validation inside `build()`** | `if (!this._bread) throw ...` |
| The wrapped sandwich you carry out | The **Product** (immutable, finished) | `Object.freeze(new Sandwich(...))` |
| A "make me the usual #7 combo" preset | The **Director** | `Director.italianBMT(builder)` |

The one sentence to remember: **a Builder separates *how you specify* an object from *how it gets constructed*, and gives you exactly one place — `build()` — to check that the specification is complete and coherent.**

---

## Key concepts inside this topic

### 1. The problem it solves — the telescoping constructor

Here is the painful code. A pizza has a few required fields and a long tail of optional ones.

```javascript
// ❌ BAD: the telescoping constructor
class Pizza {
  constructor(size, crust, cheese, extraCheese, pepperoni, mushrooms,
              olives, onions, sauce, glutenFree, stuffedCrust, notes) {
    Object.assign(this, { size, crust, cheese, extraCheese, pepperoni, mushrooms,
                          olives, onions, sauce, glutenFree, stuffedCrust, notes });
  }
}

// A call site. Quick: does this pizza have olives?
const p = new Pizza('large', 'thin', true, false, true, false, true, false,
                    'marinara', false, false, null);
```

Everything wrong with this, named:

1. **Unreadable call sites.** `true, false, true, false` is a bit-string, not code. You can't review it without counting parameters against the constructor definition.
2. **Positional fragility.** Swap two booleans and the code still runs — it just makes the wrong pizza. Nothing will save you.
3. **Null padding.** To set `notes` you must pass every argument before it, even ones you don't care about.
4. **Combinatorial overloads.** In Java you'd write six overloaded constructors. JS has no overloads, so you get the one monster.
5. **No place for cross-field validation.** "Stuffed crust is not available on gluten-free" has nowhere natural to live — the constructor is already busy assigning, and throwing mid-assignment leaves you reasoning about half-built objects.

> **Reminder:** *validation* here means checking that a **combination** of fields makes sense, not just that each field has the right type. The second kind is easy; the first is what constructors are bad at.

### 2. The structure — who plays which role

Builder has four participants. Learn these names; interviewers use them.

| Participant | Role | In our pizza example |
|---|---|---|
| **Product** | The complex object being built. Ideally immutable once finished. | `Pizza` |
| **Builder** | Holds the partial state. One named step per field. Each step returns `this` so calls chain. | `PizzaBuilder` |
| **`build()`** | The terminal step. Validates the accumulated state, then constructs and returns the Product. | `PizzaBuilder.prototype.build` |
| **Director** *(optional)* | Knows *recipes* — canned sequences of builder steps that produce common variants. | `PizzaDirector.margherita()` |

The Director is the participant everyone forgets. Its job: hold the **step order and step combination** for a well-known variant, so that knowledge isn't copy-pasted across ten call sites. Build only one kind of object? You don't need a Director.

### 3. Full JavaScript implementation — Pizza, fixed

```javascript
// ✅ GOOD: Builder pattern.
// The Product is a frozen value object. It reads fields off the *builder* and
// never validates — build() already did that.
class Pizza {
  constructor(b) {
    Object.assign(this, {
      size: b._size, crust: b._crust, sauce: b._sauce,
      glutenFree: b._glutenFree, stuffedCrust: b._stuffedCrust, notes: b._notes,
      toppings: Object.freeze([...b._toppings]),  // COPY: the builder must not be
    });                                           // able to mutate us later
    Object.freeze(this);                          // finished pizza never changes
  }

  price() {   // base + $1.50/topping + $3 for stuffed crust
    const base = { small: 8, medium: 11, large: 14 }[this.size];
    return +(base + this.toppings.length * 1.5 + (this.stuffedCrust ? 3 : 0)).toFixed(2);
  }
}

class PizzaBuilder {
  constructor() {
    // Underscore prefix = "partial state, not the product yet".
    Object.assign(this, {
      _size: null, _crust: 'classic', _sauce: 'marinara',
      _glutenFree: false, _stuffedCrust: false, _toppings: [], _notes: '',
    });
  }

  size(s)        { this._size = s; return this; }
  crust(c)       { this._crust = c; return this; }
  sauce(s)       { this._sauce = s; return this; }
  glutenFree()   { this._glutenFree = true; return this; }
  stuffedCrust() { this._stuffedCrust = true; return this; }
  addTopping(t)  { this._toppings.push(t); return this; }  // additive: call it N times
  notes(n)       { this._notes = n; return this; }

  // The ONE place where "is this pizza coherent?" is answered.
  build() {
    const errors = [];
    if (!['small', 'medium', 'large'].includes(this._size)) {
      errors.push(`size must be small|medium|large, got "${this._size}"`);
    }
    // Cross-field rules — impossible to express in a constructor signature.
    if (this._glutenFree && this._stuffedCrust) {
      errors.push('stuffed crust is not available on gluten-free bases');
    }
    if (this._size === 'small' && this._toppings.length > 4) {
      errors.push('a small pizza holds at most 4 toppings');
    }
    if (errors.length) throw new Error(`Cannot build pizza:\n - ${errors.join('\n - ')}`);
    return new Pizza(this);
  }
}

// ---- Director: canned recipes, so step-order knowledge lives in ONE place.
class PizzaDirector {
  static margherita(b) {
    return b.size('medium').crust('thin').sauce('marinara')
            .addTopping('mozzarella').addTopping('basil').build();
  }
  static meatFeast(b) {
    return b.size('large').stuffedCrust()
            .addTopping('pepperoni').addTopping('sausage').addTopping('bacon').build();
  }
}
```

Look at what the call site becomes — and now answer "does it have olives?" in one second, without counting:

```javascript
const custom = new PizzaBuilder()
  .size('large').crust('thin')
  .addTopping('pepperoni').addTopping('olives')
  .notes('cut into squares')
  .build();
console.log(custom.price());                                  // 17

console.log(PizzaDirector.margherita(new PizzaBuilder()).price());   // 14

try { new PizzaBuilder().size('small').glutenFree().stuffedCrust().build(); }
catch (e) { console.log(e.message); }
// Cannot build pizza:
//  - stuffed crust is not available on gluten-free bases
```

### 4. The real payoff — a `QueryBuilder`

Pizza is a teaching toy. The place Builder genuinely earns its salary is when **the order of steps carries meaning** and **the final product is assembled from the accumulated steps**, not just copied out of fields. SQL is the canonical case.

```javascript
class Query {
  constructor(sql, params) {
    this.sql = sql;
    this.params = Object.freeze([...params]);
    Object.freeze(this);
  }
}

class QueryBuilder {
  constructor(table) {
    if (!table) throw new Error('QueryBuilder needs a table name');
    Object.assign(this, {
      _table: table, _columns: [], _wheres: [], _orderBy: [],
      _params: [],           // stays index-aligned with the $N placeholders
      _limit: null, _offset: null,
    });
  }

  select(...cols) { this._columns.push(...cols); return this; }
  limit(n)  { this._limit = n; return this; }
  offset(n) { this._offset = n; return this; }

  // Parameterised — we never string-concat user values into SQL (injection).
  // One call pushes a placeholder into _wheres AND a value into _params, keeping
  // them index-aligned. THIS is why a builder beats an options object: a single
  // step has a coherent side effect on two collections at once.
  where(col, op, value) {
    this._params.push(value);
    this._wheres.push(`${col} ${op} $${this._params.length}`);
    return this;
  }

  whereIn(col, values) {
    const ph = values.map((v) => `$${this._params.push(v)}`);  // push returns new length
    this._wheres.push(`${col} IN (${ph.join(', ')})`);
    return this;
  }

  orderBy(col, dir = 'ASC') {
    const d = dir.toUpperCase();
    if (d !== 'ASC' && d !== 'DESC') throw new Error(`bad direction: ${dir}`);
    this._orderBy.push(`${col} ${d}`);
    return this;
  }

  build() {
    // Validation: a rule that only makes sense once you see the WHOLE query.
    if (this._offset !== null && this._limit === null) {
      throw new Error('OFFSET without LIMIT is undefined behaviour — set a limit');
    }
    // Assembly: SQL clause order is FIXED, and it is NOT the order you called the
    // methods in. The builder decouples call order from output order.
    const parts = [
      `SELECT ${this._columns.length ? this._columns.join(', ') : '*'}`,
      `FROM ${this._table}`,
      this._wheres.length  ? `WHERE ${this._wheres.join(' AND ')}`  : null,
      this._orderBy.length ? `ORDER BY ${this._orderBy.join(', ')}` : null,
      this._limit  !== null ? `LIMIT ${this._limit}`   : null,
      this._offset !== null ? `OFFSET ${this._offset}` : null,
    ].filter(Boolean);

    return new Query(parts.join('\n'), this._params);
  }
}

// Demo — note .limit() is called BEFORE .where(), yet LIMIT still lands last.
const q = new QueryBuilder('orders')
  .select('id', 'total').limit(10)
  .where('status', '=', 'paid')
  .whereIn('region', ['IN', 'US'])
  .orderBy('created_at', 'DESC')
  .build();

console.log(q.sql);
// SELECT id, total / FROM orders / WHERE status = $1 AND region IN ($2, $3)
// ORDER BY created_at DESC / LIMIT 10
console.log(q.params);   // [ 'paid', 'IN', 'US' ]
```

Notice three things an options object cannot give you here:

- `.where()` called three times **accumulates**, and each call pushes to *two* arrays that must stay index-aligned. An options object would need `{ where: [['age','>',18], ['country','=','IN']] }` — you've just reinvented the builder, badly, as nested arrays.
- **Call order ≠ output order.** You can call `.limit()` before `.where()`; the builder still emits `WHERE` before `LIMIT`. The builder owns the grammar.
- The product is a **different shape** from the input. You feed in method calls; you get out a `{ sql, params }` pair. That's a real transformation, not a field copy.

### 5. An `HttpRequestBuilder` — and the immutability benefit

```javascript
class HttpRequest {
  constructor({ method, url, headers, body, timeoutMs }) {
    Object.assign(this, { method, url, body, timeoutMs });
    this.headers = Object.freeze({ ...headers });   // copy, THEN freeze
    Object.freeze(this);                            // ← the payoff, see below
  }
}

class HttpRequestBuilder {
  constructor(baseUrl = '') {
    Object.assign(this, { _baseUrl: baseUrl, _method: 'GET', _path: '',
                          _headers: {}, _body: undefined, _timeoutMs: 5000 });
  }

  method(m)     { this._method = m.toUpperCase(); return this; }
  path(p)       { this._path = p; return this; }
  header(k, v)  { this._headers[k.toLowerCase()] = v; return this; }  // repeatable
  bearer(token) { return this.header('authorization', `Bearer ${token}`); }
  timeout(ms)   { this._timeoutMs = ms; return this; }

  // ONE user intent → TWO field changes. Exactly the coupling an options object
  // leaves the caller to remember (and forget).
  json(obj) {
    this._body = JSON.stringify(obj);
    return this.header('content-type', 'application/json');
  }

  build() {   // cross-field rules again: a GET with a body is nonsense
    if (!this._path && !this._baseUrl) throw new Error('no URL specified');
    if (this._body !== undefined && ['GET', 'HEAD'].includes(this._method)) {
      throw new Error(`${this._method} requests cannot carry a body`);
    }
    return new HttpRequest({
      method: this._method, url: `${this._baseUrl}${this._path}`,
      headers: this._headers, body: this._body, timeoutMs: this._timeoutMs,
    });
  }
}
```

**The immutability benefit, concretely.** The product is frozen, so a request handed to a retry loop, a logger, and a metrics collector **cannot be mutated by any of them**:

```javascript
// ❌ Mutable config object shared across three consumers
const cfg = { url: '/orders', headers: { 'x-trace': 'abc' } };
retryQueue.push(cfg);
logger.record(cfg);
cfg.headers['x-trace'] = 'REDACTED';   // logger "sanitizes" in place...
// ...and the retry queue now retries with a redacted trace header. Good luck
// finding that at 3am.

// ✅ Frozen product
const req = new HttpRequestBuilder('https://api.shop.com')
  .method('POST').path('/orders').bearer(token)
  .json({ sku: 'A1', qty: 2 }).timeout(2000).build();

req.timeoutMs = 60000;        // silently ignored (throws in strict mode)
req.headers['x-evil'] = '1';  // silently ignored
// Every consumer sees the same request forever. Retries are reproducible.
```

That's the deeper reason Builder survives in a language with object literals: **the builder is the only mutable phase, and it ends.** Mutation is confined to a short, local window between `new Builder()` and `.build()`; after that the value is fixed. Same shape as the "mutable draft → immutable snapshot" that Immer and Redux reducers reach for.


### 6. Node ecosystem examples — you already use these

- **knex.js** — the canonical JS Builder. `knex('users').where('age','>',18).orderBy('name').limit(10)` accumulates clauses; `.toSQL()` (or awaiting the thenable) is the `build()` step.
- **superagent** — `request.post('/api/pet').set('X-API-Key','foo').send({name:'Manny'})`. The terminal `.then()`/`.end()` is `build()` and send, fused.
- **Node core `URLSearchParams`** — `.append(k, v)` repeatedly, then `.toString()` assembles. Accumulate-then-assemble is exactly Builder's shape. **Mongoose** does the same with `.exec()`.

Say **one** of these out loud in an interview and you've proven you understand the pattern rather than the textbook.

### 7. The JS-idiomatic alternative — the plain options object (be honest)

Here is the thing most pattern tutorials won't tell you: **in JavaScript, the options object solves 80% of what Builder was invented for.**

```javascript
// The options-object version. Named. Order-independent. Skippable. Readable.
class Pizza {
  constructor({ size, crust = 'classic', sauce = 'marinara',
                toppings = [], glutenFree = false, stuffedCrust = false } = {}) {
    if (!size) throw new Error('size is required');
    if (glutenFree && stuffedCrust) throw new Error('no stuffed GF crust');
    Object.assign(this, { size, crust, sauce, toppings: [...toppings],
                          glutenFree, stuffedCrust });
    Object.freeze(this);
  }
}
const p = new Pizza({ size: 'large', crust: 'thin', toppings: ['pepperoni'] });
```

That kills the telescoping constructor **dead**. Named parameters, defaults, skip what you don't need, validate in the constructor, freeze the result. Builder was invented for Java and C++, where this destructuring-with-defaults trick does not exist. In JS it does. If a candidate builds a 60-line `PizzaBuilder` for the code above, a good interviewer will ask why — and "the Gang of Four said so" is the wrong answer.

**So when does a Builder still earn its keep in JavaScript?** Four cases:

| Case | Why the options object loses | Example |
|---|---|---|
| **Step-order / grammar matters** | You must accumulate calls and emit them in a *different, fixed* order, or one call touches several internal collections at once. | `QueryBuilder.where()` pushing to `_wheres` and `_params` in lockstep |
| **Repeatable / additive steps** | `.addTopping()` × 5, `.where()` × 3, `.header()` × 4. Encoding "call this N times" in a literal means nested arrays-of-tuples — worse than the builder. | `superagent.set()`, `knex.where()` |
| **Many variants of one product** | You want named recipes (a **Director**) instead of ten copies of the same 8-key literal scattered across the codebase. | `PizzaDirector.meatFeast()`, `aUser().asAdmin().build()` fixtures |
| **Immutable product from a mutable draft** | You want a construction *phase* that ends. An options object gives you one shot; a builder can be passed around, configured incrementally by different layers, and sealed at the end. | Building an HTTP request across middleware layers, then freezing |

**Everything else — just use the options object.** A Builder that is only `.setA().setB().setC().build()` with no validation, no ordering, no repetition, and no Director is a 60-line reimplementation of `{ a, b, c }`. That's not a pattern, that's ceremony.

### 8. Two refinements worth knowing

**(a) The immutable / persistent builder.** Have each step return a *new* builder instead of mutating `this` — `#with(patch) { return new QB({ ...this._s, ...patch }); }`. Builders become safe to share and fork: `const base = qb.from('orders').where("status='paid'")` can be branched twice without the branches corrupting each other. Cost is one small object per step — irrelevant, don't optimize it away. This is exactly the trap knex's `.clone()` exists to solve.

**(b) Fluent interface ≠ Builder.** jQuery's `$el.addClass().css().show()` chains, but every call takes effect immediately on a live object, there's no terminal `build()`, and no product is produced. Builder is fluent *and* has a build step yielding a separate, finished thing. No `.build()`, no product ⇒ not a Builder. Interviewers probe this.

---

## Visual / Diagram description

### Diagram 1: The participants

```
      ┌────────────────────────────────────────────┐
      │            Director (optional)              │
      │  Knows RECIPES: which steps, in which order │
      │  + margherita(builder) : Pizza              │
      │  + meatFeast(builder)  : Pizza              │
      └──────────────────┬─────────────────────────┘
                         │ drives (calls steps on)
                         ▼
      ┌────────────────────────────────────────────┐
      │        Builder  (MUTABLE partial state)     │
      │  - _size, _crust, _toppings[], _sauce ...   │
      │  + size(s)       : this  ─┐                │
      │  + crust(c)      : this   │ each returns   │
      │  + addTopping(t) : this   │ `this`         │
      │  + sauce(s)      : this  ─┘  → chaining    │
      │  + build()       : Pizza ◀── VALIDATES here│
      └──────────────────┬─────────────────────────┘
                         │ creates (exactly once)
                         ▼
      ┌────────────────────────────────────────────┐
      │        Product  (IMMUTABLE, frozen)         │
      │  + size, crust, toppings, sauce  + price()  │
      │  (no setters — cannot change after build)   │
      └────────────────────────────────────────────┘
```

Read it top to bottom: the Director (if present) knows the recipe, the Builder holds the half-finished object and all the mutation, and the Product falls out of `build()` fully formed and frozen. The **only** arrow into Product is from `build()` — nothing else may construct one.

### Diagram 2: The two phases (the mental model that matters)

```
   ┌──────── MUTABLE PHASE ───────┐  ┌──── IMMUTABLE PHASE ────┐
 new Builder()                    │  │                         │
    ├─ .size('large')     ─┐      │  │   Pizza { frozen }      │
    ├─ .crust('thin')      │state │  │        ├──▶ retryQueue  │
    ├─ .addTopping('pep')  │grows │  │        ├──▶ logger      │
    └─ .notes('squares')  ─┘step  │  │        └──▶ metrics     │
                                  │  │                         │
                .build() ═════════╪═▶│  All three see the SAME │
                   │              │  │  object. None can       │
              ┌────▼─────┐        │  │  corrupt it for the     │
              │ VALIDATE │ throws │  │  others.                │
              │ size? GF │ if bad │  │                         │
              │ +stuffed?│        │  │                         │
              └──────────┘        │  │                         │
   └──────────────────────────────┘  └─────────────────────────┘
```

The whiteboard version of this doc: **mutation is a phase, and `build()` is the wall at the end of it.** Everything before the wall is a draft; nothing after the wall can change. Draw this, explain it, and you've explained Builder.

---

## Real world examples

### 1. knex.js — the SQL query builder every Node backend has touched

Knex builds SQL by accumulating clauses on a query object. `knex('users').select('id','name').where('age','>',18).whereIn('country',['IN','US']).orderBy('name').limit(10)` produces a parameterised statement plus a bindings array — precisely the `{ sql, params }` product from our `QueryBuilder`. The build step is `.toSQL()` (or awaiting the query, since knex query builders are thenables). Knex's builders are **mutable and chainable**, which is why `.clone()` exists: if you keep a partially built query as a shared base and branch off it, you must clone or the branches corrupt each other. That single API method is the best real-world evidence for the immutable-builder refinement above.

### 2. superagent — HTTP request building

`superagent.post('/api/pet').set('X-API-Key','foobar').set('Accept','application/json').send({ name: 'Manny' }).timeout({ response: 5000 })` is our `HttpRequestBuilder`, shipped. Notice `.set()` is **repeatable** (call it once per header) — the exact case an options object handles clumsily. Notice also that `.send()` sets both the body and the content-type header, one call touching two fields, another builder tell. The terminal step (`.then()`/`.end()`) fuses build-and-execute; a stricter design would separate them, which is what makes requests replayable.

### 3. Test fixture builders — the use case that converts skeptics

The most common place working JS/TS teams hand-write Builders is test data: `aUser().withRole('admin').inOrg('acme').verified().build()`. Each builder starts from a valid default, and the test names only the fields it cares about. Compare with a 15-key object literal repeated across 40 tests: add a required field to `User`, and the builder version needs **one** edit (the default) while the literal version needs **40**. The Director appears naturally as named presets (`aUser().asAdmin()`). This is the strongest "Builder in JS is worth it" argument you can make in an interview, because it's about *change cost*, not aesthetics.

---

## Trade-offs

| Pros | What it buys you |
|---|---|
| **Readable call sites** | Named steps beat positional booleans. Code review becomes possible. |
| **One validation point** | Cross-field rules live in `build()`, not scattered or duplicated. |
| **Immutable products** | Frozen output is safe to share across retries, logs, caches. |
| **Input order ≠ output order** | Caller sets things in any order; the builder emits them in the required grammar. |
| **Repeatable steps** | `.addX()` N times is natural; an options literal has to nest arrays. |
| **Named variants via Director** | Recipes live in one place, not copy-pasted across the codebase. |

| Cons | What it costs you |
|---|---|
| **Boilerplate** | ~1 method per field, plus a product class. 3× the lines of an options object. |
| **Two classes to sync** | Add a field → touch the builder, the product, and the validation. |
| **Runtime-only errors** | Plain JS won't warn about a missing `.size()` until `build()` throws. (TS can, with staged builder types.) |
| **Easy to over-apply** | A builder with 3 setters, no validation, and no ordering is pure ceremony. |
| **Mutable-builder aliasing** | Reusing one builder for two products silently cross-contaminates. Clone or go immutable. |

**Rule of thumb:** In JavaScript, **reach for the options object first.** Upgrade to a Builder only when at least one of these is true: (1) steps repeat, (2) step order or output grammar matters, (3) one call must mutate several internal fields coherently, (4) you need named recipes for many variants, or (5) you must hand a *frozen* product to multiple consumers. If none apply, `new Thing({ a, b, c })` is the better engineering.

---

## Common interview questions on this topic

### Q1: "JavaScript has object literals with defaults and destructuring. Why would you ever write a Builder?"
**Hint:** Concede the point first — the options object kills the telescoping constructor, and for a flat bag of optional fields it's the right call. Then name the four cases where it doesn't stretch: repeatable/additive steps (`.where()` × 3), grammar where call order ≠ emit order (SQL clauses), one call that must update several fields in lockstep (`.where()` pushing to both the fragment list and the params list), and wanting a Director for named variants. Close with a real example: knex.

### Q2: "How is Builder different from Factory?"
**Hint:** Factory answers ***which*** class to instantiate, in one call, based on some input — `createShape('circle')`. Builder answers ***how*** to assemble ***one*** complex object, over many calls. Factory hides the type; Builder hides the assembly. Mnemonic: *Factory = one shot, picks a type. Builder = many shots, one type.* They compose fine — a factory can hand you the right builder.

### Q3: "Why put validation in `build()` instead of in each setter?"
**Hint:** Because the interesting rules are *cross-field* and can't be evaluated until you've seen everything. `.glutenFree()` alone is fine; `.glutenFree()` + `.stuffedCrust()` is not. Setter-level validation also breaks order-independence — validating "size ≤ 4 toppings" inside `.addTopping()` would demand that `.size()` be called first. Per-field type checks *can* go in the setter (fail fast, better stack trace); coherence checks must go in `build()`.

### Q4: "What breaks if I reuse the same builder to make two products?"
**Hint:** The second product inherits the first one's accumulated state — leftover toppings, leftover WHERE clauses. And if the product holds a *reference* to a builder array instead of a copy, mutating the builder afterwards mutates the already-built product. Two fixes: copy collections into the product (`[...builder._toppings]`), and either reset the builder in `build()`, return a fresh builder each time, or go fully immutable (each step returns a new builder). Knex ships `.clone()` for exactly this.

### Q5: "Is jQuery's `$el.addClass().css().fadeIn()` the Builder pattern?"
**Hint:** No — that's a **fluent interface**. Chaining is a syntax, not a pattern. Builder = fluent chaining **plus a terminal `build()` that produces a separate, finished product**. jQuery's chain mutates a live object and never produces anything new. If there's no `build()` and no product, it isn't Builder.

---

## Practice exercise

### Build a `NotificationBuilder` (~30-40 min)

You're designing the notification system for an e-commerce app. A notification can go out over one or more channels, and the rules are fiddly. Write it with a Builder.

1. A frozen `Notification` product: `recipientId`, `channels` (any of `email`/`sms`/`push`), `subject`, `body`, `priority` (`low`|`normal`|`urgent`, default `normal`), `scheduledAt` (optional `Date`), `attachments` (array).
2. `NotificationBuilder` with chainable steps: `.to()`, `.viaEmail()`, `.viaSms()`, `.viaPush()`, `.subject()`, `.body()`, `.priority()`, `.scheduleAt()`, `.attach()`. The channel and attach steps are **repeatable/additive**.
3. Validation **all inside `build()`**, collecting *every* error before throwing (don't bail on the first): `recipientId` and `body` required; at least one channel; `email` ⇒ `subject` required; `sms` ⇒ `body.length` ≤ 160; `urgent` may not be scheduled for the future; attachments only allowed on `email`.
4. The product must be **frozen**, and its `channels`/`attachments` arrays must be *copies*. Prove it: build a notification, then push a channel onto the builder, then assert the product is unchanged.
5. A `NotificationDirector` with two recipes: `orderShipped(b, userId, trackingNo)` and `passwordReset(b, userId, token)` — the second must be `urgent`, push + email, no attachments.
6. A `main()` demo: build one valid notification directly, one via each Director recipe, and one **failing** build (SMS with a 300-char body plus an attachment) — catch it and print the full list of validation failures.

**What to produce:** one runnable `.js` file. Then, in a closing comment, answer in 3-4 sentences: *would a plain options object have been good enough here?* Point at the specific requirement that justifies (or doesn't justify) the Builder. Either answer is defensible — what's graded is whether you can name the deciding factor.

---

## Quick reference cheat sheet

- **Builder** = construct a complex object step by step; separate *specification* from *construction*.
- **The pain it kills:** the telescoping constructor — `new Pizza('large', true, false, null, true, ...)`.
- **Participants:** Product (immutable result), Builder (mutable partial state), `build()` (validate + construct), Director (optional; knows recipes).
- **Chaining trick:** every step does `return this`. That's the entire mechanism.
- **`build()` is the wall:** all cross-field validation happens there; nothing after it can mutate the product.
- **Freeze the product:** `Object.freeze(this)` + copy arrays in (`[...builder._toppings]`), so consumers can't corrupt each other.
- **Director** = named recipes (`margherita()`, `meatFeast()`), so step-order knowledge lives in one place.
- **In JS, the options object usually wins:** `new Pizza({ size, cheese })` gives named, optional, defaulted params for free.
- **Builder still earns its keep when:** steps repeat (`.where()` ×3), call order ≠ output order (SQL grammar), one call updates several fields in lockstep, you need many named variants, or you need a frozen product shared across consumers.
- **Fluent ≠ Builder:** no terminal `build()` and no separate product means it's just a fluent interface (jQuery).
- **Reuse hazard:** one builder → two products leaks state. Reset, re-create, or make each step return a *new* builder (knex's `.clone()`).
- **Real JS Builders:** knex, superagent, Mongoose chains, `URLSearchParams`, and test fixture builders (`aUser().asAdmin().build()`).
- **Builder vs Factory:** Factory = one call, picks *which* class. Builder = many calls, assembles *one* object.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [31 — Abstract Factory Pattern](./31-pattern-abstract-factory.md) — creating families of related objects; Builder handles one complex object instead |
| **Next** | [33 — Prototype Pattern](./33-pattern-prototype.md) — the other way to avoid expensive construction: clone instead of build |
| **Related** | [30 — Factory Method Pattern](./30-pattern-factory-method.md) — the #1 interview comparison: Factory picks a type, Builder assembles one |
| **Related** | [26 — Composition Over Inheritance](./26-composition-over-inheritance.md) — why a Director composing builder steps beats a hierarchy of PizzaSubclasses |
