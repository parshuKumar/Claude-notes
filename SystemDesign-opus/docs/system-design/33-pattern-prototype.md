# 33 — Prototype Pattern

## Category: LLD Patterns

---

## What is this?

The **Prototype pattern** creates new objects by **copying an existing one** instead of building it from scratch. You keep one fully-configured specimen around — the *prototype* — and whenever you need another, you clone it and tweak the few fields that differ.

Think of a photocopier. Setting up an original document is expensive: you type it, format it, get sign-off. Making the 500th copy is nearly free — you press a button. If every copy had to be re-typed from the source material, you'd never ship. Prototype says: **do the expensive setup once, then stamp out copies.**

---

## Why does it matter?

Object construction is not always cheap. Some objects need a database round-trip, a 200KB config file parsed and validated, a TLS handshake, a compiled regex, or a deeply nested default tree assembled from six sources. If creating one costs 40ms and you need 10,000 of them, you've just spent 400 seconds. Cloning a ready-made instance might cost 20 microseconds — a 2000× difference.

**At work:** you clone default config objects before applying per-request overrides, clone a pre-authenticated HTTP client so each caller can set its own timeout without disturbing anyone else's, spawn 500 enemies in a game from one loaded model, or snapshot state before a speculative update so you can roll back.

**In interviews:** Prototype is the pattern where JavaScript itself is the punchline. JS has **prototypal inheritance** — the language is built on this idea, and `Object.create` is the pattern as a language primitive. An interviewer asking "explain the Prototype pattern" in a JS role is often really asking "do you understand `__proto__`, and do you know the difference between *delegating* to a prototype and *copying* one?" Also: **shallow vs deep clone** is one of the most common sources of bugs in all of JS, and this is where it lives.

---

## The core idea — explained simply

### The Master Key Analogy

You run a hotel with 500 rooms. Every keycard needs the same base setup: encryption keys, elevator permissions, gym-access flag, Wi-Fi credentials, property ID. That setup takes real work — it comes from the property management system, gets validated, gets signed.

**The naive approach:** for every guest who checks in, re-fetch the keys, re-read the config, re-derive the permissions, re-sign the card. Every check-in pays the full cost, and at the Saturday rush the front desk grinds to a halt.

**The prototype approach:** at 6am, prepare **one** perfectly-configured *master template card* — everything except the room number and expiry. Then for every guest: **copy the template**, stamp in *their* room number and checkout date, hand it over. The expensive work happened once, at 6am.

Now the crucial part. Suppose the template carries a small **list of authorized doors** — and instead of giving each copy its own fresh list, your copier hands every card **a pointer to the template's one list**. Guest 12 asks for spa access; you add "spa" to *his* list — and because it's the same list, **all 500 guests now have spa access.** That is the **shallow-copy aliasing bug**, and it is the single most important thing in this doc.

| Hotel | Prototype pattern | In code |
|---|---|---|
| The master template card | The **prototype** — a fully-configured specimen | `const baseCard = new Keycard({...})` |
| Copying the template for a guest, then stamping the room number | **`clone()`**, then customize | `const c = baseCard.clone(); c.room = 412;` |
| The expensive 6am setup | Costly construction (DB read, parse, handshake) — done **once** | |
| Every card pointing at the *same* door list | **Shallow copy** — nested objects shared, not copied | `{ ...proto }` |
| Every card getting its *own* door list | **Deep copy** — nested objects recursively copied | `structuredClone(proto)` |
| The drawer of named templates (staff / guest / VIP) | **Prototype registry** | `registry.get('vip')` |

One sentence: **Prototype = "copy a good instance instead of constructing a new one" — and the whole difficulty of the pattern is *how deep* the copy goes.**

---

## Key concepts inside this topic

### 1. The problem it solves — expensive construction, repeated

A `ReportGenerator` needs a heavy setup: parse a template file, compile regexes, pull branding from the database. Here's the painful version:

```javascript
// ❌ BAD: every instance pays the full construction cost
class ReportGenerator {
  constructor(db, templatePath) {
    this.template = parseTemplate(fs.readFileSync(templatePath, 'utf8'));  // ~15ms
    this.placeholders = this.template.tokens.map((t) => new RegExp(`{{${t}}}`, 'g')); // ~3ms
    this.branding = db.querySync('SELECT logo, colors FROM branding LIMIT 1');  // ~25ms
    // per-report fields — the ONLY things that differ between instances
    this.title = 'Untitled';
    this.rows = [];
    this.filters = { from: null, to: null, tags: [] };
  }
}

for (const customer of customers) {                     // 1,000 customers
  const gen = new ReportGenerator(db, './tpl.html');    // ~43ms of setup. EVERY TIME.
  gen.title = `${customer.name} — Monthly`;
  gen.rows = customer.rows;
  reports.push(gen.render());
}
// 1,000 × 43ms ≈ 43 seconds — and ~40ms of each 43ms was IDENTICAL work.
```

Read that loop again. The template is the same file. The regexes compile to the same objects. The branding row is the same row — you queried the database a thousand times for a value that never changed. Only `title`, `rows`, and `filters` actually vary.

**The Prototype fix:** build **one** generator, then clone it 1,000 times.

```javascript
const prototypeGen = new ReportGenerator(db, './tpl.html');  // pay 43ms ONCE
for (const customer of customers) {
  const gen = prototypeGen.clone();                          // ~0.02ms
  gen.title = `${customer.name} — Monthly`;
  gen.rows = customer.rows;
  reports.push(gen.render());
}
// 43ms + (1,000 × 0.02ms) ≈ 63ms. ~680× faster. Now the trap.
```

### 2. Shallow vs deep clone — the aliasing bug, demonstrated

A **shallow clone** copies the object's own top-level properties. If a property holds a *primitive* (number, string, boolean), you get a real copy. If it holds a **reference** to another object or array, you copy **the reference, not the thing it points to** — so the original and the clone now point at the *same* nested object. Change it through one and it changes for the other. That shared-reference situation is called **aliasing**.

```javascript
// ❌ THE BUG — a clone() that spreads. Copies own enumerable props ONE level deep.
//    Looks right. Isn't.
ReportGenerator.prototype.clone = function () {
  return Object.assign(Object.create(Object.getPrototypeOf(this)), { ...this });
};

const proto  = new ReportGenerator(db, './tpl.html');
const acme   = proto.clone();
const globex = proto.clone();

acme.title = 'Acme — Monthly';        // ✅ fine: strings are primitives, real copy
acme.filters.tags.push('enterprise'); // ⚠️ filters is an OBJECT. Shared reference!
console.log(globex.filters.tags);     // [ 'enterprise' ]  ← Globex's report is now
                                      //   polluted with Acme's data.
console.log(proto.filters.tags);      // [ 'enterprise' ]  ← And the PROTOTYPE is
              // corrupted, so every FUTURE clone inherits the poison. It compounds.
```

This is the bug at its most vicious: **silent**, it **cross-contaminates unrelated tenants**, and because the prototype itself got mutated, it **compounds**. In a Node server with a module-level prototype, request 1 leaks state into request 4,000. You will not find this from the stack trace.

### 3. Fixing it — a proper `clone()`, and `structuredClone`

**Fix A — the explicit `clone()`.** Hand-write the copy and decide, field by field, what's shared and what's copied. Verbose, but *you* control the depth — and that control is the point.

```javascript
// ✅ GOOD: explicit clone. Every field is a decision. (The constructor takes a
// _skipInit flag so clone() can bypass the 43ms of heavy work.)
ReportGenerator.prototype.clone = function () {
  const copy = new ReportGenerator(null, null, /* _skipInit */ true);
  // SHARED ON PURPOSE — immutable + expensive. Copying these would defeat the
  // entire point of the pattern. A deliberate decision, not an accident.
  copy.template = this.template;
  copy.placeholders = this.placeholders;
  copy.branding = this.branding;
  // COPIED — mutable per-instance state. Every clone gets its OWN.
  copy.title   = this.title;                                         // primitive
  copy.rows    = [...this.rows];                                     // new array
  copy.filters = { ...this.filters, tags: [...this.filters.tags] };  // nested!
  return copy;
};
```

Note what the fix actually *is*: **not "deep-clone everything."** It's *deciding*. The template, regexes, and branding are shared **deliberately** — they're immutable and expensive, and sharing them is *why* cloning is fast. Only mutable per-instance state gets copied. A blind deep clone would duplicate a 200KB parsed template a thousand times and give back all the speed you were trying to win.

**Fix B — `structuredClone` (Node 17+, built in).** When the object is plain data and you genuinely want a full deep copy, don't hand-roll it and don't reach for lodash:

```javascript
const defaults = {
  backoff: { type: 'exponential', baseMs: 100, maxMs: 5_000 },
  hooks: { onRetry: [], onFail: [] },
  createdAt: new Date('2026-01-01'),
  allowlist: new Set(['api.acme.com']),
};

const cfg = structuredClone(defaults);   // deep, recursive, built-in
cfg.backoff.baseMs = 500;
cfg.hooks.onRetry.push(logRetry);
cfg.allowlist.add('api.globex.com');

console.log(defaults.backoff.baseMs, defaults.hooks.onRetry, defaults.allowlist.size);
// 100  []  1   ← all untouched. And `allowlist` is still a real Set, not `{}`.
```

**Know its limits, because interviewers ask:**

| Value | `{...spread}` | `JSON.parse(JSON.stringify())` | `structuredClone` |
|---|---|---|---|
| Nested objects/arrays | ❌ shared (aliased) | ✅ copied | ✅ copied |
| `Date` | ❌ shared | ⚠️ becomes a string | ✅ real `Date` |
| `Map` / `Set` | ❌ shared | ❌ becomes `{}` | ✅ copied |
| Circular refs | ✅ (shared anyway) | ❌ **throws** | ✅ handled |
| **Functions / methods** | ✅ shared (works) | ❌ dropped | ❌ **throws `DataCloneError`** |

| **Class identity (`instanceof`)** | ✅ preserved | ❌ lost | ❌ **lost — plain object back** |

Read the last two rows twice. **`structuredClone` on a class instance gives you back a plain object with the same data and none of the methods**, and it *throws* if any property holds a function. So: **`structuredClone` for data** (config trees, state snapshots, message payloads); **a hand-written `clone()` for objects with behaviour.** That's the rule.

### 4. JavaScript is special here — prototypal inheritance IS this pattern

This is what makes this doc different from the same doc written for Java. In a class-based language, "prototype" is a design pattern you bolt on. **In JavaScript, it's the object model.** Every JS object has a hidden link to another object — its **prototype** — and when you read a property the object doesn't have, the engine walks up that link and looks there. That walk is the **prototype chain**, and the mechanism is called **delegation**.

```javascript
// Object.create(proto) makes a NEW object whose prototype link points at `proto`.
// This is the GoF Prototype idea as a one-line language primitive.
const enemyProto = {
  hp: 100,
  attack() { return `${this.name} hits for ${this.damage} damage`; },
};

const goblin = Object.create(enemyProto);   // goblin's prototype IS enemyProto
goblin.name = 'Goblin';
goblin.damage = 7;

console.log(goblin.attack());              // "Goblin hits for 7 damage"
console.log(goblin.hp);                    // 100 ← NOT on goblin. Found by walking
                                           //   up the chain to enemyProto.
console.log(Object.hasOwn(goblin, 'hp'));  // false ← proof: it's borrowed

// Reading walks UP the chain. Writing does NOT — it creates an OWN property.
goblin.hp = 80;                     // creates hp ON goblin; enemyProto untouched
console.log(enemyProto.hp);         // 100  ✅

// But mutating a shared NESTED object DOES reach through — same aliasing bug!
enemyProto.loot = { gold: 0, items: [] };
const orc = Object.create(enemyProto);
orc.loot.gold = 50;                 // ⚠️ NOT a write to `orc` — it's a READ of
                                    //   orc.loot (→ enemyProto.loot), then a
                                    //   mutation of THAT shared object.
console.log(goblin.loot.gold);      // 50  ← every enemy is now rich. Same bug,
console.log(enemyProto.loot.gold);  // 50  ← different clothing.
```

**Delegation is not copying — that distinction is the whole interview answer.** And note JS gives you the aliasing footgun **twice**: once via shallow copies, once via the prototype chain. Same root cause — a reference you thought was yours is shared.

**Where `class` fits.** `class` is syntax over exactly this machinery; there is no second mechanism underneath:

```javascript
class Enemy {
  constructor(name) { this.name = name; }
  attack() { return `${this.name} attacks`; }   // lives on Enemy.prototype
}
const e = new Enemy('Orc');
console.log(Object.hasOwn(e, 'attack'));                    // false — not on instance
console.log(Object.hasOwn(Enemy.prototype, 'attack'));      // true  — on the prototype
console.log(e.attack === Enemy.prototype.attack);           // true — ONE function
```

Your class **methods are already prototype-shared**. Ten thousand `Enemy` instances hold *one* `attack` function between them — that's the Prototype pattern (and, incidentally, Flyweight) working for you for free, and it's why methods belong on the prototype rather than being re-created as arrow-function fields in the constructor.

> **`__proto__` vs `prototype`** — the classic confusion, worth nailing:
> - **`obj.__proto__`** (properly: `Object.getPrototypeOf(obj)`) is the link **an instance** follows when looking up a property. Every object has one.
> - **`Fn.prototype`** is a plain object hanging off a **constructor function**, which becomes the `__proto__` of instances that `new Fn()` creates. Only functions have it. So `new Enemy()` ⇒ `enemy.__proto__ === Enemy.prototype`.
> - Never *assign* to `__proto__` in real code — it deoptimizes the object in V8. Use `Object.create` at construction time.

**Delegation vs cloning — when to use which:**

| | **Delegation** (`Object.create`) | **Cloning** (`clone()` / `structuredClone`) |
|---|---|---|
| Memory | One shared base; instances hold only their diffs | Each instance holds a full copy |
| Base changed at runtime | **All** delegates see it instantly | Existing clones unaffected |
| Mutating shared nested state | Leaks to everyone (the footgun above) | Isolated (if the clone went deep enough) |
| Use it for | Shared behaviour + immutable defaults | Mutable per-instance state |

The right design usually uses **both**: delegate the behaviour and the immutable heavy stuff, clone the mutable state. That is exactly what `ReportGenerator.clone()` above does.

### 5. Full JavaScript implementation — registry + game entity spawning

Games are the textbook use case: load an entity's heavy definition once, then spawn hundreds by cloning. A **prototype registry** is the keyed collection of specimens — ask for a name, get a fresh clone. Complete and runnable:

```javascript
'use strict';

// loadModel() pretends to read a mesh off disk; buildAiTree() compiles a behaviour tree.
function loadModel(path) {
  const end = Date.now() + 30;                 // simulate ~30ms of disk + parse work
  while (Date.now() < end) { /* burn */ }
  return { path, vertices: 12_000, textures: ['diffuse', 'normal'] };
}
const buildAiTree = (k) => Object.freeze({ kind: k, nodes: ['patrol', 'chase', 'attack'] });

class Enemy {
  // `_bare` lets clone() construct an empty shell and skip the heavy work.
  constructor({ name, modelPath, aiKind, hp, damage, loot } = {}, _bare = false) {
    if (_bare) return;
    // --- EXPENSIVE + IMMUTABLE: loaded once, then SHARED by every clone ---
    this.model = loadModel(modelPath);        // ~30ms
    this.ai    = buildAiTree(aiKind);         // frozen behaviour tree
    // --- CHEAP + MUTABLE: every clone must get its OWN copy ---
    Object.assign(this, {
      name, hp, damage, maxHp: hp, id: null,
      position: { x: 0, y: 0 },                            // nested object
      loot: { gold: loot.gold, items: [...loot.items] },   // nested object
      buffs: new Set(),                                    // Set
    });
  }
  clone() {
    const copy = new Enemy({}, /* _bare */ true);
    copy.model = this.model;   // SHARED: immutable + expensive. Why cloning is fast.
    copy.ai    = this.ai;      // SHARED
    // COPIED: mutable per-instance state. Miss ANY of these and you get the aliasing
    // bug — one goblin taking damage would hurt every goblin.
    Object.assign(copy, {
      name: this.name, hp: this.hp, maxHp: this.maxHp, damage: this.damage,
      position: { ...this.position },
      loot: { gold: this.loot.gold, items: [...this.loot.items] },
      buffs: new Set(this.buffs), id: null,   // a clone is NOT the same entity
    });
    return copy;
  }

  takeDamage(n) { this.hp = Math.max(0, this.hp - n); return this; }
  addBuff(b)    { this.buffs.add(b); return this; }
  toString() { return `${this.name}#${this.id} hp=${this.hp}/${this.maxHp} ` +
                      `pos=(${this.position.x},${this.position.y}) gold=${this.loot.gold}`; }
}

class PrototypeRegistry {
  #prototypes = new Map();
  #nextId = 1;
  register(key, prototype) {
    if (typeof prototype.clone !== 'function') throw new Error(`"${key}" needs clone()`);
    this.#prototypes.set(key, prototype);
    return this;
  }

  // ALWAYS hands out a clone — never the stored specimen. Hand out the prototype
  // itself and the caller will mutate it, poisoning every future spawn.
  spawn(key, overrides = {}) {
    const proto = this.#prototypes.get(key);
    if (!proto) throw new Error(`unknown prototype "${key}"`);
    const entity = proto.clone();
    entity.id = this.#nextId++;
    return Object.assign(entity, overrides);
  }
}

// --------------------------------- DEMO ---------------------------------
function main() {
  const t0 = Date.now();
  const registry = new PrototypeRegistry()
    .register('goblin', new Enemy({ name: 'Goblin', modelPath: '/models/goblin.glb',
      aiKind: 'melee', hp: 30, damage: 5, loot: { gold: 10, items: ['rusty-dagger'] } }))
    .register('orc', new Enemy({ name: 'Orc', modelPath: '/models/orc.glb',
      aiKind: 'melee', hp: 80, damage: 12, loot: { gold: 40, items: ['axe'] } }));
  console.log(`2 prototypes loaded in ${Date.now() - t0}ms`);   // ~60ms

  const t1 = Date.now();
  const horde = [];
  for (let i = 0; i < 500; i++) {
    horde.push(registry.spawn(i % 5 === 0 ? 'orc' : 'goblin',
                              { position: { x: (i * 37) % 200, y: (i * 61) % 200 } }));
  }
  console.log(`500 enemies spawned in ${Date.now() - t1}ms`);   // single-digit ms
  console.log('(constructing all 500 would have cost ~500 x 30ms = ~15,000ms)\n');

  // --- Isolation: mutate ONE clone, check its sibling (both are goblins) ---
  const [, a, b] = horde;
  a.takeDamage(25).addBuff('poisoned');
  a.loot.gold += 999;
  a.loot.items.push('stolen-crown');

  console.log('mutated  :', a.toString());
  console.log('neighbour:', b.toString());                                  // untouched
  console.log('separate loot arrays:', a.loot.items !== b.loot.items);      // true ✅
  console.log('model SHARED on purpose:', a.model === b.model);             // true ✅

  // --- Contrast: the shallow-clone bug, live ---
  const bad = Object.assign(Object.create(Object.getPrototypeOf(b)), { ...b }); // ❌
  bad.loot.gold = 0;                       // mutating `bad`...
  console.log('victim goblin gold after its shallow clone was mutated:', b.loot.gold);
  // → 0. We just robbed a different enemy by "copying" one. That is the bug.
}

main();
```

### 6. When NOT to use it / how it's abused

- **Construction is already cheap.** If your constructor assigns five fields, cloning is *slower* than `new` (a copy walk on top of allocation) and you've added a `clone()` to maintain. Just call the constructor.
- **The object has identity.** A `User` with a database primary key, a `Socket`, a file handle, an open DB connection — these are not values; copying them is meaningless or dangerous (two objects "owning" one file descriptor). Prototype is for **value-ish** objects.
- **Blind deep-cloning everything.** The most common abuse. `structuredClone(hugeObject)` per request duplicates the megabytes of read-only config you were trying to *share*, turning a fast pattern into a slow one. Cloning is a *decision per field*, not a switch. If nobody mutates the nested state at all, `Object.freeze` it and share — clone only what gets written to.
- **`JSON.parse(JSON.stringify(x))` as a clone.** It silently corrupts `Date` (→ string), `Map`/`Set` (→ `{}`), `undefined` and functions (dropped), `NaN`/`Infinity` (→ `null`), and **throws** on circular refs. A lossy serializer wearing a clone costume.
- **Cloning to dodge a design problem.** If you're cloning because a shared mutable singleton keeps getting corrupted, the bug isn't "we need copies" — it's the singleton. Fix the ownership.

### 7. Related patterns and how they differ (the #1 interview follow-up)

| Pattern | What it does | vs Prototype |
|---|---|---|
| **Factory Method / Abstract Factory** (30, 31) | A method (or family of them) decides **which class** to instantiate. Uses `new`. | Factory constructs from a class; Prototype copies an existing *instance* and needs no class hierarchy — the specimen carries the config. A prototype registry is often a cheaper substitute for a factory hierarchy: adding a variant means registering a specimen, not writing a subclass. |
| **Builder** (32) | Assembles one complex object **step by step**, then `build()`s it. | Builder = construct from parts. Prototype = copy a finished whole. **They pair beautifully:** use a Builder to construct the prototype *once*, then clone it forever (knex does exactly this — a builder with `.clone()`). |
| **Flyweight** (40) | Shares one immutable object across many contexts to save memory. | Prototype **copies** so each instance owns its state; Flyweight **shares** to avoid copies. Our `Enemy` uses **both**: the mesh is a flyweight, the hp/position are cloned. Naming that in an interview is a strong signal. |
| **Memento** (49) | Snapshots an object's state so it can be restored later (undo). | A Memento *is* a deep clone — but Prototype clones to **create new objects**, Memento clones to **restore an old one**. Same mechanism, opposite direction. |
| **Singleton** (29) | Exactly one instance, globally. | The literal opposite. A prototype registry is the *safe* alternative when you were reaching for a mutable singleton: "here is the canonical one — never touch it, take a copy." |

---

## Visual / Diagram description

### Diagram 1: Participants

```
        ┌────────────────────────────────────┐
        │       «interface» Prototype         │
        │       + clone() : Prototype         │
        └─────────────────┬──────────────────┘
                          │ implements
              ┌───────────┴────────────┐
              ▼                        ▼
  ┌───────────────────────┐  ┌───────────────────────┐
  │        Enemy          │  │      ApiClient        │
  │  - model   (SHARED)   │  │  - agent   (SHARED)   │
  │  - ai      (SHARED)   │  │  - baseUrl (SHARED)   │
  │  - hp      (COPIED)   │  │  - headers (COPIED)   │
  │  - loot    (COPIED)   │  │  - timeout (COPIED)   │
  │  + clone() : Enemy    │  │  + clone(): ApiClient │
  └───────────────────────┘  └───────────────────────┘
              ▲  stores specimens; hands out clones
  ┌───────────┴─────────────────────────────┐
  │          PrototypeRegistry               │
  │  - #prototypes : Map<string, Prototype>  │
  │  + register(key, proto)                  │
  │  + spawn(key, overrides) : Prototype     │
  │      └─ returns proto.clone(), NEVER the │
  │         stored specimen itself           │
  └─────────────────────────────────────────┘
```

Every prototype exposes one method, `clone()`. Inside it, each field is a decision: **shared** (immutable + expensive) or **copied** (mutable). The registry holds the specimens and always returns a clone, never the specimen.

### Diagram 2: Shallow vs deep — draw this from memory

```
  ORIGINAL                     SHALLOW CLONE  { ...orig }
  ┌──────────────┐             ┌──────────────┐
  │ hp:      100 │             │ hp:      100 │  ← primitives: truly copied ✅
  │ name:  "Orc" │             │ name:  "Orc" │
  │ loot:    ●───┼──────┐      │ loot:    ●───┼──┐
  └──────────────┘      │      └──────────────┘  │
                        ▼                        │
                 ┌─────────────────┐             │
                 │ { gold: 40,     │◀────────────┘  ⚠️ SAME OBJECT.
                 │   items: [...] }│                clone.loot.gold = 0
                 └─────────────────┘                zeroes the ORIGINAL too.

  A DEEP CLONE — structuredClone(orig), or a hand-written clone() — draws a
  SECOND box for `loot`, so each owner has its own. ✅ Mutate one freely.
```

---

## Real world examples

### 1. Node.js configuration defaults — cloning before override

Nearly every configurable Node library keeps a module-level `defaults` object and produces per-instance config by **copying defaults and merging overrides**. Axios does this: `axios.create(config)` merges instance config over library defaults, then each request merges again over the instance's. The bug this guards against is exactly the aliasing bug: if the library handed the *same* nested `headers` object to every instance, one caller setting an `Authorization` header would leak their bearer token into every other caller's requests. That's a security incident, not a style issue — which is why "clone the defaults, don't share them" is load-bearing.

### 2. Game engines — entity spawning from a prefab

Unity calls it a *Prefab*, Unreal a *Blueprint class*, and the runtime call is literally `Instantiate(prefab)` — take a fully-configured template entity and stamp out a copy at a new position. The heavy assets (mesh, textures, compiled animation graphs) are **shared by reference** across every instance; only mutable per-entity state (transform, health, current AI node) is copied. That split is exactly the `shared vs copied` decision in our `Enemy.clone()`, and it's why 500 goblins cost 500 small state objects, not 500 meshes. Browser games on three.js use the same architecture: one loaded `Geometry`/`Material` (shared), many `Mesh` instances (cloned).

### 3. Cloning a configured HTTP client

A pattern you'll write yourself within a year of backend work:

```javascript
class ApiClient {
  constructor({ baseUrl, headers = {}, timeoutMs = 5000, agent }) {
    this.agent = agent;      // an http.Agent — SHARED on purpose: it owns the TCP
                             // connection pool. Cloning it would kill keep-alive.
    Object.assign(this, { baseUrl, timeoutMs, headers: { ...headers } });  // COPIED
  }

  clone(overrides = {}) {    // configurable copy that still reuses the pool
    return new ApiClient({
      agent:     this.agent,                                          // SHARED
      baseUrl:   overrides.baseUrl ?? this.baseUrl,
      headers:   { ...this.headers, ...(overrides.headers ?? {}) },   // COPIED
      timeoutMs: overrides.timeoutMs ?? this.timeoutMs,               // COPIED
    });
  }
}

const base = new ApiClient({ baseUrl: 'https://api.acme.com',
                             agent: new http.Agent({ keepAlive: true }) });

// Per-request clients: each carries its own trace id and timeout, and all of them
// reuse ONE warm TCP connection pool.
const forRequest = base.clone({ headers: { 'x-trace': traceId }, timeoutMs: 1500 });
```

Prototype at its most honest: the expensive thing (a connection pool with warm TLS sessions) is *deliberately shared*; the cheap mutable thing (headers, timeout) is copied. Getting that split right is the skill; `clone()` is just where you write it down.

---

## Trade-offs

| Pros | What it buys you |
|---|---|
| **Skips expensive construction** | Pay the DB read / file parse / handshake once, amortized over N instances. 500 spawns cost 500 shallow state copies, not 500 asset loads. |
| **Config lives in data, not classes** | Add a variant by registering a specimen — no new subclass, no factory edit. Prototypes can even be loaded from JSON at startup; a factory hierarchy is fixed in code. |
| **Composes with Builder & Flyweight** | Build the prototype once, share its heavy immutable parts, clone the rest. |

| Cons | What it costs you |
|---|---|
| **Shallow-copy aliasing bugs** | The #1 killer. Silent, cross-tenant, and it corrupts the prototype itself, so it worsens over time. |
| **`clone()` must be maintained** | Add a field, forget to copy it, get a shared-reference bug months later. No compiler catches this. And a hand-written `clone()` infinitely recurses on circular references unless you track visited nodes. |
| **Deep clone can be expensive** | Blindly deep-cloning a fat object gives back all the performance you were buying. And `structuredClone` loses class identity: plain object back, no methods, no `instanceof`; it *throws* on function-valued props. |
| **Wrong for objects with identity** | DB rows, sockets, file handles, connections — copying them is meaningless or unsafe. |

**The sweet spot:** Use Prototype when construction is **expensive** and the object is **value-like** (data + behaviour, no external identity). In `clone()`, treat every field as an explicit decision — **share** what is immutable and expensive, **copy** what is mutable and per-instance. Pure data ⇒ `structuredClone`; has methods ⇒ hand-write `clone()`. And remember JS gives you delegation for free: **delegate the behaviour, clone the state.**

---

## Common interview questions on this topic

### Q1: "What's the difference between a shallow and a deep clone? Show me a bug."
**Hint:** Shallow copies top-level properties only — nested objects are copied *by reference*, so the original and clone share them (**aliasing**). Show the two-line bug: `const b = {...a}; b.tags.push('x');` → `a.tags` also has `'x'`. Then name the fixes and their limits: `structuredClone` (deep; handles `Date`/`Map`/`Set`/cycles, but **drops methods, throws on functions, loses `instanceof`**), a hand-written `clone()` (full control, must be maintained), and explicitly *not* `JSON.parse(JSON.stringify())` (lossy on `Date`, `Map`, `Set`, `undefined`, functions; throws on cycles).

### Q2: "JavaScript has prototypal inheritance. Isn't the Prototype pattern redundant here?"
**Hint:** Related but not the same — the difference is **delegation vs copying**. `Object.create(proto)` gives you an object that *points at* the prototype and looks up missing properties there; nothing is copied, and a later change to the prototype is visible to every delegate. The GoF pattern *copies* — each clone owns its state and is thereafter independent. JS is unusual in giving you delegation as a language primitive, and the best designs use both: delegate shared behaviour (class methods already live on `Enemy.prototype` — one function object for all instances) and clone the mutable state.

### Q3: "Prototype vs Factory — when would you pick each?"
**Hint:** Factory constructs from a **class** — good when variants are known at compile time and differ in *behaviour*. Prototype copies an **instance** — good when variants differ in *configuration*, when the set of variants is loaded at runtime (from JSON/DB), or when construction is expensive. Killer line: with a factory, adding an enemy type means writing a subclass and editing a switch; with a prototype registry, it means registering one more specimen — often from a config file, with no code change.

### Q4: "You have 500 game enemies from one prototype and mutating one changes all of them. Debug it."
**Hint:** Shallow clone. Some nested field — `position`, `loot`, `stats`, a `Set` of buffs — is copied by reference, so all 500 share one object. Find it by checking `a.loot === b.loot` (should be `false`; if `true`, that's your field). Fix: copy it in `clone()`. Then say the thing that shows depth: *don't* fix it by deep-cloning everything — the mesh and AI tree **must** stay shared or you'll blow up memory and load time. Cloning is a per-field decision.

### Q5: "What's the relationship between Prototype and Flyweight? And Memento?"
**Hint:** Prototype and Flyweight are opposite responses to the same pressure. Flyweight **shares** one immutable object across many contexts to save memory; Prototype **copies** so each instance owns its state. A good `clone()` uses both at once — share the mesh (flyweight), copy the hp (prototype). Memento is a deep clone with a different *intent*: Prototype clones to create a **new** object, Memento clones to **restore an old one** (undo). Same mechanism, opposite direction.

---

## Practice exercise

### Build a `DocumentTemplate` prototype system (~30-40 min)

You're building a document service. Loading a template is expensive (parse + validate + fetch branding); creating a document from one must be cheap.

**What to build, in one runnable `.js` file:**

1. A `DocumentTemplate` class whose constructor takes a `name` and simulates ~40ms of expensive work (busy-wait, like `loadModel` above) to produce: `schema` (frozen, **immutable**), `branding` (frozen `{ logo, primaryColor, footer }`, **immutable**), `sections` (array of `{ heading, body }`, **mutable**), `metadata` (`{ author: null, tags: [], createdAt: null }`, **mutable + nested**), and `reviewers` (a `Set`, **mutable**).
2. A `clone()` that **shares** `schema` and `branding` by reference but gives each clone its own `sections`, `metadata` (including the nested `tags` array), and `reviewers`. Hand-write it — do **not** use `structuredClone`, because the class has methods.
3. A `PrototypeRegistry` with `register(key, proto)` and `create(key, overrides)`. `create()` must return a **clone**, never the stored specimen. Register three templates: `invoice`, `contract`, `report`.
4. A `main()` demo that times registering the 3 templates (~120ms) and then creating **200 documents** by cloning (single-digit ms) — print both. Then **prove isolation**: mutate one document (push a section, add a tag, add a reviewer, set the author) and assert a sibling document *and the registered prototype* are unchanged. Then **prove deliberate sharing**: assert `docA.schema === docB.schema` is `true`, and explain in a comment why that is correct and not a bug.
5. **Bug-hunt bonus.** Add `badClone()` that does `return Object.assign(Object.create(Object.getPrototypeOf(this)), { ...this })`. Use it to make two documents, mutate one's `metadata.tags`, print the other's, and state in a comment exactly which line causes the leak.

**What to produce:** the runnable file, plus a closing comment block listing — field by field — which fields you shared and which you copied, and the rule you used to decide. That table *is* the Prototype pattern; everything else is syntax.

---

## Quick reference cheat sheet

- **Prototype** = create new objects by **cloning a configured instance**, not constructing from scratch.
- **Use it when construction is expensive:** DB round-trip, file parse, TLS handshake, compiled regexes, big default trees.
- **The aliasing bug:** a shallow copy (`{...obj}`, `Object.assign`) shares nested objects by reference — so mutating a clone's nested field changes the original *and every other clone*. Silent, cross-tenant, and it poisons the prototype, so it compounds over time.
- **`structuredClone(x)`** = built-in deep clone (Node 17+). Handles nested objects, `Date`, `Map`, `Set`, `RegExp`, **circular refs** — but **throws** on functions and **loses class identity** (plain object back, no methods, no `instanceof`). Use it for **data**; hand-write `clone()` for objects with **behaviour**. And **never** `JSON.parse(JSON.stringify(x))` — it kills `Date`, `Map`, `Set`, `undefined` and functions, and throws on cycles.
- **`clone()` is a decision per field:** **share** what is immutable + expensive (mesh, template, connection pool); **copy** what is mutable + per-instance (hp, headers, tags).
- **JS is special:** prototypal inheritance *is* this idea in the language. `Object.create(proto)` links an object to a prototype — that's **delegation**, not copying. Delegates *see* later prototype changes and share nested state; clones are independent snapshots.
- **`obj.__proto__` vs `Fn.prototype`:** `__proto__` is the link an *instance* follows; `Fn.prototype` is what instances get linked to. `new Enemy()` ⇒ `e.__proto__ === Enemy.prototype`. And your class methods are **already prototype-shared** — 10,000 instances, one `attack` function. Free Flyweight.
- **Prototype registry:** a `Map<string, prototype>`; `create(key)` returns a **clone**, never the stored specimen. **Don't use Prototype for:** cheap constructors, or objects with identity (DB rows, sockets, file handles, connections).
- **vs Builder:** Builder constructs from parts; Prototype copies a finished whole. Best combo: **Builder makes the prototype once, then clone forever** (knex's `.clone()`). **vs Flyweight:** Flyweight *shares* to save memory, Prototype *copies* to isolate state — a good `clone()` does both at once.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [32 — Builder Pattern](./32-pattern-builder.md) — construct step by step; use a Builder to make the prototype, then clone it forever |
| **Next** | [34 — Adapter Pattern](./34-pattern-adapter.md) — the first structural pattern: making incompatible interfaces talk |
| **Related** | [40 — Flyweight Pattern](./40-pattern-flyweight.md) — the mirror image: *share* immutable state instead of copying it |
| **Related** | [49 — Memento Pattern](./49-pattern-memento.md) — a deep clone with a different intent: snapshot to *restore*, not to *create* |
