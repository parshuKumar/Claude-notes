# 33 — Prototype Pattern

## Category: LLD Patterns

---

## What is this?

The **Prototype pattern** creates new objects by **copying an existing one** instead of building it from scratch. You keep one fully-configured specimen around — the *prototype* — and whenever you need another, you clone it and tweak the few fields that differ.

Think of a photocopier. Setting up an original document is expensive: you type it, format it, get sign-off. Making the 500th copy is nearly free — you press a button. If every copy had to be re-typed from the source material, you'd never ship. Prototype says: **do the expensive setup once, then stamp out copies.**

---

## Why does it matter?

Object construction is not always cheap. Some objects need a database round-trip, a 200KB config file parsed and validated, a TLS handshake, a compiled regex, or a deeply nested default tree assembled from six sources. If creating one costs 40ms and you need 10,000 of them, you've just spent 400 seconds. Cloning a ready-made instance might cost 20 microseconds — a 2000× difference.

**At work:** you clone default config objects before per-request overrides, you clone a pre-authenticated HTTP client so each caller can set its own timeout without disturbing anyone else's, you spawn 500 enemies in a game from one loaded model, you `structuredClone` state before a speculative update so you can roll back.

**In interviews:** Prototype is the pattern where JavaScript itself is the punchline. JS has **prototypal inheritance** — the language is literally built on this idea, and `Object.create` is the pattern as a language primitive. An interviewer asking "explain the Prototype pattern" in a JS role is often really asking "do you understand `__proto__`, and do you know the difference between *delegating* to a prototype and *copying* one?" Get that distinction right and you look senior. Also: **shallow vs deep clone** is one of the most common source-of-bugs questions in all of JS, and this is where it lives.

---

## The core idea — explained simply

### The Master Key Analogy

You run a hotel with 500 rooms. Every keycard needs the same base setup: the hotel's encryption keys, the elevator permissions, the gym-access flag, the Wi-Fi credentials, the property ID. That base setup takes real work to assemble — it comes from the property management system, gets validated, gets signed.

**The naive approach:** for every guest who checks in, go fetch the encryption keys, re-read the property config, re-derive the permissions, re-sign the card. Every check-in pays the full setup cost. At the Saturday-afternoon rush, the front desk grinds to a halt.

**The prototype approach:** at 6am, prepare **one** perfectly-configured *master template card*. It has everything except the room number and expiry. Then, for every guest: **copy the template**, stamp in *their* room number and *their* checkout date, hand it over. The expensive work happened once at 6am.

Now the subtle, crucial part. Suppose the template card carries a small **list of authorized doors** — and instead of giving each copy its own fresh list, your copier hands every card **a pointer to the template's one list**. Guest 12 asks for spa access; you add "spa" to *his* list — and because it's the same list, **all 500 guests now have spa access.** That is the **shallow-copy aliasing bug**, and it is the single most important thing in this doc.

**Mapping the analogy back to code:**

| Hotel | Prototype pattern | In code |
|---|---|---|
| The master template card | The **prototype** — a fully-configured specimen | `const baseCard = new Keycard({...})` |
| Copying the template for a guest | **`clone()`** | `baseCard.clone()` |
| Stamping in the room number | Post-clone customization | `card.roomNumber = 412` |
| The expensive 6am setup | Costly construction (DB read, parse, handshake) | done **once** |
| Every card pointing at the *same* door list | **Shallow copy** — nested objects are shared, not copied | `{ ...proto }` |
| Every card getting its *own* door list | **Deep copy** — nested objects are recursively copied | `structuredClone(proto)` |
| The drawer of named templates (staff / guest / VIP) | **Prototype registry** | `registry.get('vip')` |

One sentence: **Prototype = "copy a good instance instead of constructing a new one" — and the whole difficulty of the pattern is *how deep* the copy goes.**

---

## Key concepts inside this topic

### 1. The problem it solves — expensive construction, repeated

Here's the painful version. A `ReportGenerator` needs a heavy setup: it parses a template file, compiles regexes, and pulls branding from the database.

```javascript
// ❌ BAD: every instance pays the full construction cost
class ReportGenerator {
  constructor(db, templatePath) {
    // ~15ms: read + parse a big template from disk
    this.template = parseTemplate(fs.readFileSync(templatePath, 'utf8'));

    // ~3ms: compile the placeholder regexes
    this.placeholders = this.template.tokens.map((t) => new RegExp(`{{${t}}}`, 'g'));

    // ~25ms: a DB round-trip for company branding
    this.branding = db.querySync('SELECT logo, colors, footer FROM branding LIMIT 1');

    // per-report fields — the ONLY things that actually differ between instances
    this.title = 'Untitled';
    this.rows = [];
    this.filters = { from: null, to: null, tags: [] };
  }
}

// Generating 1,000 monthly reports:
for (const customer of customers) {              // 1,000 customers
  const gen = new ReportGenerator(db, './tpl.html');  // ~43ms of setup. EVERY TIME.
  gen.title = `${customer.name} — Monthly`;
  gen.rows = customer.rows;
  reports.push(gen.render());
}
// 1,000 × 43ms ≈ 43 seconds, and ~40ms of each 43ms was IDENTICAL work.
```

Read that loop again. The template is the same file. The regexes compile to the same objects. The branding row is the same row — you queried the database a thousand times for a value that never changed. Only `title`, `rows`, and `filters` actually vary.

**The Prototype fix:** build **one** generator, then clone it 1,000 times.

```javascript
const prototypeGen = new ReportGenerator(db, './tpl.html');  // pay 43ms ONCE
for (const customer of customers) {
  const gen = prototypeGen.clone();       // ~0.02ms
  gen.title = `${customer.name} — Monthly`;
  gen.rows = customer.rows;
  reports.push(gen.render());
}
// 43ms + (1,000 × 0.02ms) ≈ 63ms total. Roughly 680× faster.
```

That's the pattern in one page. **Now the trap.**

### 2. Shallow vs deep clone — the aliasing bug, demonstrated

A **shallow clone** copies the object's own top-level properties. If a property holds a *primitive* (number, string, boolean), you get a real copy. If it holds a **reference** to another object or array, you copy **the reference, not the thing it points to** — so the original and the clone now point at the *same* nested object. Change it through one, and it changes for the other. That shared-reference situation is called **aliasing**.

Watch it bite:

```javascript
// ❌ THE BUG
class ReportGenerator {
  // ...heavy constructor as above...
  clone() {
    // Spread copies own enumerable props — ONE level deep. Looks right. Isn't.
    return Object.assign(Object.create(Object.getPrototypeOf(this)), { ...this });
  }
}

const proto = new ReportGenerator(db, './tpl.html');

const acme  = proto.clone();
const globex = proto.clone();

acme.title = 'Acme — Monthly';        // ✅ fine: strings are primitives, real copy
globex.title = 'Globex — Monthly';    // ✅ fine

acme.filters.tags.push('enterprise'); // ⚠️ filters is an OBJECT. Shared reference!
acme.rows.push({ sku: 'A1', total: 90 });

console.log(globex.filters.tags);     // [ 'enterprise' ]   ← Globex's report is
console.log(globex.rows.length);      // 1                  ← now polluted with
console.log(proto.filters.tags);      // [ 'enterprise' ]   ← Acme's data. And the
                                      //                       PROTOTYPE is corrupted,
                                      //                       so every FUTURE clone
                                      //                       inherits the poison too.
```

This is the bug in its most vicious form: it's **silent**, it **cross-contaminates unrelated tenants**, and because the prototype itself got mutated, it **gets worse over time**. In a Node server with a module-level prototype, request 1 leaks state into request 4,000. You will not find this from the stack trace.

**The mental picture:**

```
SHALLOW CLONE — nested objects are SHARED
  proto ──┬── title: "Untitled"      (primitive → copied)
          └── filters: ─────────────┐
                                    ▼
  acme  ──┬── title: "Acme"          ┌─────────────────┐
          └── filters: ─────────────▶│ { tags: [...] } │  ◀── ONE object,
                                     └─────────────────┘      THREE owners.
  globex ─┬── title: "Globex"         ▲                       Anyone mutates it,
          └── filters: ───────────────┘                       everyone sees it.

DEEP CLONE — nested objects are COPIED
  proto  ─── filters: ──▶ { tags: [] }        ← its own
  acme   ─── filters: ──▶ { tags: ['ent'] }   ← its own
  globex ─── filters: ──▶ { tags: [] }        ← its own
```

### 3. Fixing it — a proper `clone()`, and `structuredClone`

**Fix A — the explicit `clone()`.** You hand-write the copy and decide, field by field, what's shared and what's copied. Verbose, but *you* control the depth, and that control is the point.

```javascript
// ✅ GOOD: explicit clone. You decide what's shared vs copied — per field.
class ReportGenerator {
  constructor(db, templatePath, _skipInit = false) {
    if (_skipInit) return;                       // clone() bypasses the heavy work
    this.template     = parseTemplate(fs.readFileSync(templatePath, 'utf8'));
    this.placeholders = this.template.tokens.map((t) => new RegExp(`{{${t}}}`, 'g'));
    this.branding     = db.querySync('SELECT logo, colors, footer FROM branding LIMIT 1');
    this.title   = 'Untitled';
    this.rows    = [];
    this.filters = { from: null, to: null, tags: [] };
  }

  clone() {
    const copy = new ReportGenerator(null, null, true);

    // SHARED ON PURPOSE — immutable and expensive. Copying them would defeat
    // the entire point of the pattern. This is a deliberate decision, not an accident.
    copy.template     = this.template;
    copy.placeholders = this.placeholders;
    copy.branding     = this.branding;

    // COPIED — per-instance mutable state. Every clone gets its OWN.
    copy.title   = this.title;                 // primitive
    copy.rows    = [...this.rows];             // new array (elements still shared —
                                               //   fine, rows are treated as read-only)
    copy.filters = {
      ...this.filters,
      tags: [...this.filters.tags],            // the nested array needs its own copy too
    };
    return copy;
  }

  render() { /* ...uses template + branding + rows... */ return `<report>${this.title}</report>`; }
}
```

Note what the fix actually is: **it is not "deep-clone everything."** It's *deciding*. The template, regexes, and branding are shared **deliberately** — they're immutable and expensive, and sharing them is why cloning is fast. Only the mutable per-instance state gets copied. A blind deep clone would duplicate a 200KB parsed template a thousand times and give back all the speed you were trying to win.

**Fix B — `structuredClone` (Node 17+, built into the language).** When the object is plain data and you genuinely want a full deep copy, don't hand-roll it and don't reach for lodash:

```javascript
const defaults = {
  retries: 3,
  backoff: { type: 'exponential', baseMs: 100, maxMs: 5_000 },
  headers: { 'user-agent': 'svc/1.0' },
  hooks: { onRetry: [], onFail: [] },
  createdAt: new Date('2026-01-01'),
  allowlist: new Set(['api.acme.com']),
};

const cfg = structuredClone(defaults);   // deep, recursive, built-in
cfg.backoff.baseMs = 500;
cfg.hooks.onRetry.push(logRetry);
cfg.allowlist.add('api.globex.com');

console.log(defaults.backoff.baseMs);    // 100  ← untouched
console.log(defaults.hooks.onRetry);     // []   ← untouched
console.log(defaults.allowlist.size);    // 1    ← untouched
```

`structuredClone` handles nested objects, arrays, `Date`, `Map`, `Set`, `RegExp`, typed arrays, and even **circular references** (`JSON.parse(JSON.stringify(x))` throws on those, and also silently destroys `Date` → string, drops `undefined`, drops functions, and mangles `Map`/`Set` into `{}`).

**Know its limits, because interviewers ask:**

| Value | `{...spread}` | `JSON.parse(JSON.stringify())` | `structuredClone` |
|---|---|---|---|
| Nested objects/arrays | ❌ shared (aliased) | ✅ copied | ✅ copied |
| `Date` | ❌ shared | ⚠️ becomes a string | ✅ real `Date` |
| `Map` / `Set` | ❌ shared | ❌ becomes `{}` | ✅ copied |
| Circular refs | ✅ (shared anyway) | ❌ **throws** | ✅ handled |
| **Functions / methods** | ✅ shared (works) | ❌ dropped | ❌ **throws `DataCloneError`** |
| **Class identity (`instanceof`)** | ✅ preserved | ❌ lost | ❌ **lost — returns a plain object** |
| Getters/setters | ❌ flattened to values | ❌ flattened | ❌ flattened |

Read the last two rows twice. **`structuredClone` on a class instance gives you back a plain object with the same data and none of the methods**, and it *throws* if any property holds a function. So: `structuredClone` for **data** (config trees, state snapshots, message payloads); a hand-written `clone()` for **objects with behaviour**. That's the rule.

### 4. JavaScript is special here — prototypal inheritance IS this pattern

Here's the part that makes this doc different from the same doc written for Java.

In a class-based language, "prototype" is a design pattern you bolt on. **In JavaScript, it's the object model.** Every JS object has a hidden link to another object — its **prototype** — and when you read a property that the object doesn't have, the engine walks up that link and looks there. That walk is the **prototype chain**, and the mechanism is called **delegation**.

```javascript
// Object.create(proto) makes a NEW object whose prototype link points at `proto`.
// This is the GoF Prototype pattern as a one-line language primitive.
const enemyProto = {
  hp: 100,
  speed: 5,
  attack() { return `${this.name} hits for ${this.damage} damage`; },
};

const goblin = Object.create(enemyProto);   // goblin's prototype IS enemyProto
goblin.name = 'Goblin';
goblin.damage = 7;

console.log(goblin.attack());                    // "Goblin hits for 7 damage"
console.log(goblin.hp);                          // 100 ← NOT on goblin. Found by
                                                 //   walking up to enemyProto.
console.log(Object.hasOwn(goblin, 'hp'));        // false ← proof: it's borrowed
console.log(Object.getPrototypeOf(goblin) === enemyProto);  // true
```

**Delegation is not copying — and that distinction is the whole interview answer.**

```javascript
// Reading walks UP the chain. Writing does NOT — it creates an OWN property.
goblin.hp = 80;                     // creates hp ON goblin; enemyProto untouched
console.log(enemyProto.hp);         // 100  ✅

// But mutating a shared NESTED object does reach through — same aliasing bug!
enemyProto.loot = { gold: 0, items: [] };
const orc = Object.create(enemyProto);
orc.loot.gold = 50;                 // ⚠️ NOT a write to `orc` — it's a READ of
                                    //   orc.loot (→ enemyProto.loot), then a
                                    //   mutation of THAT shared object.
console.log(goblin.loot.gold);      // 50  ← every enemy is now rich. Same bug,
console.log(enemyProto.loot.gold);  // 50  ← different clothing.
```

So JS gives you the aliasing footgun **twice**: once via shallow copies, once via the prototype chain. Same root cause: a reference you thought was yours is shared.

**Where `class` fits.** `class` is syntax over exactly this machinery — there is no second mechanism hiding underneath:

```javascript
class Enemy {
  constructor(name) { this.name = name; }
  attack() { return `${this.name} attacks`; }   // lives on Enemy.prototype
}
const e = new Enemy('Orc');

console.log(Object.hasOwn(e, 'attack'));                     // false — not on the instance
console.log(Object.hasOwn(Enemy.prototype, 'attack'));       // true  — it's on the prototype
console.log(Object.getPrototypeOf(e) === Enemy.prototype);   // true
console.log(e.attack === Enemy.prototype.attack);            // true — ONE function object,
                                                             //   shared by every instance
```

Your class **methods are already prototype-shared**. Ten thousand `Enemy` instances hold *one* `attack` function between them, delegating to it — that's the Prototype pattern (and, incidentally, the Flyweight pattern) working for you, for free, and it's why methods belong on the prototype rather than being re-created as arrow-function fields in the constructor.

> **`__proto__` vs `prototype`** — the classic confusion, worth nailing:
> - **`obj.__proto__`** (properly: `Object.getPrototypeOf(obj)`) is the link **an instance** follows when looking up a property. Every object has one.
> - **`Fn.prototype`** is a plain object hanging off a **constructor function**, which becomes the `__proto__` of instances that `new Fn()` creates. Only functions have it.
> - `new Enemy()` ⇒ `enemy.__proto__ === Enemy.prototype`. Never assign to `__proto__` in real code — it deoptimizes the object in V8. Use `Object.create` at construction time.

**Delegation vs cloning — when to use which:**

| | **Delegation** (`Object.create`) | **Cloning** (`clone()` / `structuredClone`) |
|---|---|---|
| Memory | One shared copy of the base; instances hold only their diffs | Each instance holds a full copy |
| Base changes at runtime | **All** delegates see the change instantly | Existing clones are unaffected |
| Mutating shared nested state | Leaks to everyone (the footgun above) | Isolated (if the clone was deep enough) |
| Cost per instance | ~free | Proportional to copy depth |
| Use it for | Shared behaviour + immutable defaults (methods, constants, big read-only blobs) | Mutable per-instance state |

The right design usually uses **both**: delegate the behaviour and the immutable heavy stuff, clone the mutable state. That is precisely what the `ReportGenerator.clone()` above does.

### 5. The Prototype Registry

Once you're cloning, you need somewhere to keep the specimens. A **prototype registry** is a keyed collection of pre-configured prototypes: ask for a name, get a fresh clone.

```javascript
class PrototypeRegistry {
  #prototypes = new Map();

  register(key, prototype) {
    if (typeof prototype.clone !== 'function') {
      throw new Error(`prototype "${key}" must implement clone()`);
    }
    this.#prototypes.set(key, prototype);
    return this;
  }

  // Always hands out a CLONE — never the stored specimen itself. If we returned the
  // prototype, the caller would mutate it and poison every future spawn.
  create(key, overrides = {}) {
    const proto = this.#prototypes.get(key);
    if (!proto) {
      throw new Error(`unknown prototype "${key}". Known: ${[...this.#prototypes.keys()].join(', ')}`);
    }
    return Object.assign(proto.clone(), overrides);
  }

  keys() { return [...this.#prototypes.keys()]; }
}
```

That `create()` returning a clone rather than the stored object is the whole safety story of the registry. Get it wrong and you have a Singleton pretending to be a factory.

### 6. Full JavaScript implementation — game entity spawning

Games are the textbook Prototype use case: load an entity's heavy definition once (mesh, stats, AI config), then spawn hundreds by cloning. Here's a complete, runnable version.

```javascript
'use strict';

// ---------------------------------------------------------------------------
// Product: an entity. Construction is expensive — pretend loadModel() reads a
// mesh off disk and buildAiTree() compiles a behaviour tree.
// ---------------------------------------------------------------------------
function loadModel(path) {
  // Simulate ~30ms of disk + parse work.
  const end = Date.now() + 30;
  while (Date.now() < end) { /* burn */ }
  return { path, vertices: 12_000, textures: ['diffuse', 'normal'] };
}
function buildAiTree(kind) {
  return Object.freeze({ kind, nodes: ['patrol', 'chase', 'attack', 'flee'] });
}

class Enemy {
  // The `_bare` flag lets clone() construct an empty shell and skip the heavy work.
  constructor({ name, modelPath, aiKind, hp, damage, loot } = {}, _bare = false) {
    if (_bare) return;

    // --- EXPENSIVE, IMMUTABLE: loaded once, then SHARED by every clone ---
    this.model = loadModel(modelPath);      // ~30ms
    this.ai    = buildAiTree(aiKind);       // frozen behaviour tree

    // --- CHEAP, MUTABLE: every clone must get its OWN copy ---
    this.name      = name;
    this.hp        = hp;
    this.maxHp     = hp;
    this.damage    = damage;
    this.position  = { x: 0, y: 0 };                 // nested object → must deep-copy
    this.loot      = { gold: loot.gold, items: [...loot.items] };  // nested → deep-copy
    this.buffs     = new Set();                      // Set → must copy
    this.id        = null;
  }

  clone() {
    const copy = new Enemy({}, /* _bare */ true);

    // Shared on purpose: immutable and expensive. This is WHY cloning is fast.
    copy.model = this.model;
    copy.ai    = this.ai;

    // Copied: mutable per-instance state. Miss ANY of these and you get the
    // aliasing bug — one goblin taking damage would hurt every goblin.
    copy.name     = this.name;
    copy.hp       = this.hp;
    copy.maxHp    = this.maxHp;
    copy.damage   = this.damage;
    copy.position = { ...this.position };
    copy.loot     = { gold: this.loot.gold, items: [...this.loot.items] };
    copy.buffs    = new Set(this.buffs);
    copy.id       = null;                   // a clone is NOT the same entity
    return copy;
  }

  takeDamage(n) { this.hp = Math.max(0, this.hp - n); return this; }
  addBuff(b)    { this.buffs.add(b); return this; }
  toString() {
    return `${this.name}#${this.id} hp=${this.hp}/${this.maxHp} ` +
           `pos=(${this.position.x},${this.position.y}) gold=${this.loot.gold} ` +
           `buffs=[${[...this.buffs]}]`;
  }
}

// ---------------------------------------------------------------------------
// Registry (from §5), plus an id counter so spawned entities are distinguishable.
// ---------------------------------------------------------------------------
class PrototypeRegistry {
  #prototypes = new Map();
  #nextId = 1;

  register(key, prototype) {
    if (typeof prototype.clone !== 'function') throw new Error(`"${key}" needs clone()`);
    this.#prototypes.set(key, prototype);
    return this;
  }

  spawn(key, overrides = {}) {
    const proto = this.#prototypes.get(key);
    if (!proto) throw new Error(`unknown prototype "${key}"`);
    const entity = proto.clone();           // never hand out the specimen itself
    entity.id = this.#nextId++;
    return Object.assign(entity, overrides);
  }
}

// ---------------------------------------------------------------------------
// DEMO
// ---------------------------------------------------------------------------
function main() {
  console.log('--- Loading prototypes (the expensive part) ---');
  const t0 = Date.now();

  const registry = new PrototypeRegistry()
    .register('goblin', new Enemy({
      name: 'Goblin', modelPath: '/models/goblin.glb', aiKind: 'melee',
      hp: 30, damage: 5, loot: { gold: 10, items: ['rusty-dagger'] },
    }))
    .register('orc', new Enemy({
      name: 'Orc', modelPath: '/models/orc.glb', aiKind: 'melee',
      hp: 80, damage: 12, loot: { gold: 40, items: ['axe', 'hide'] },
    }));

  console.log(`2 prototypes loaded in ${Date.now() - t0}ms\n`);

  console.log('--- Spawning 500 enemies by CLONING ---');
  const t1 = Date.now();
  const horde = [];
  for (let i = 0; i < 500; i++) {
    horde.push(registry.spawn(i % 5 === 0 ? 'orc' : 'goblin', {
      position: { x: (i * 37) % 200, y: (i * 61) % 200 },
    }));
  }
  console.log(`500 enemies spawned in ${Date.now() - t1}ms`);
  console.log(`(constructing them would have cost ~500 x 30ms = ~15,000ms)\n`);

  console.log('--- Proving isolation: mutate ONE clone, check the others ---');
  const [a, b] = horde;
  a.takeDamage(25).addBuff('poisoned');
  a.loot.gold += 999;
  a.loot.items.push('stolen-crown');
  a.position.x = 42;

  console.log('mutated :', a.toString());
  console.log('neighbour:', b.toString());
  console.log('loot arrays are separate objects:',
              a.loot.items !== b.loot.items);                    // true  ✅
  console.log('but the MODEL is shared (deliberately):',
              a.model === b.model);                              // true  ✅ — the
                                                                 //   whole point
  console.log('shared model memory: 1 mesh, not 500.\n');

  console.log('--- What the shallow-clone bug would have looked like ---');
  const bad = Object.assign(Object.create(Object.getPrototypeOf(b)), { ...b }); // ❌
  bad.loot.gold = 0;                    // mutating `bad`...
  console.log('victim goblin gold after mutating its shallow clone:', b.loot.gold);
  //  → 0. We just robbed a different enemy by "copying" one. This is the bug.
}

main();
```

Run it. The output makes the pattern's whole value proposition visible in one screen: 500 spawns in single-digit milliseconds, isolated mutable state, one shared mesh, and a live demonstration of the aliasing bug at the end.

### 7. When NOT to use it / how it's abused

- **Construction is already cheap.** If your constructor just assigns five fields, cloning is *slower* than `new` (you pay for a copy walk on top of allocation) and you've added a `clone()` method to maintain. Just call the constructor.
- **The object has identity.** A `User` with a database primary key, a `Socket`, a file handle, an open DB connection — these are not values, and copying them is meaningless or actively dangerous (two objects "owning" one file descriptor). Prototype is for **value-ish** objects.
- **Blind deep-cloning everything.** The most common abuse. `structuredClone(hugeObject)` on every request duplicates the megabytes of read-only config you were trying to *share*, and turns a fast pattern into a slow one. Cloning is a *decision per field*, not a switch.
- **`JSON.parse(JSON.stringify(x))` as a clone.** It silently corrupts `Date` (→ string), `Map`/`Set` (→ `{}`), `undefined` (dropped), functions (dropped), `NaN`/`Infinity` (→ `null`), and **throws** on circular refs. It's a lossy serializer wearing a clone costume. Use `structuredClone`.
- **Cloning to dodge a design problem.** If you're cloning because a shared mutable singleton keeps getting corrupted, the bug isn't "we need copies" — it's the shared mutable singleton. Fix the ownership, don't paper over it.
- **Deep-cloning where immutability would be simpler.** If nobody mutates the nested state, sharing it is free and correct. Freeze it (`Object.freeze`) and share. Clone only what actually gets written to.

### 8. Related patterns and how they differ (the #1 interview follow-up)

| Pattern | What it does | vs Prototype |
|---|---|---|
| **Factory Method** (30) | A method decides **which class** to instantiate. Uses `new`. | Factory constructs from a class; Prototype copies an existing *instance*. Prototype needs no class hierarchy at all — the specimen carries the config. |
| **Abstract Factory** (31) | Creates **families** of related objects. | Same distinction, one level up. A registry of prototypes is often a cheaper substitute for a factory hierarchy: adding a new variant means registering a specimen, not writing a subclass. |
| **Builder** (32) | Assembles one complex object **step by step**, then `build()`s it. | Builder = construct from parts. Prototype = copy a finished whole. **They pair beautifully:** use a Builder to construct the prototype *once*, then clone it forever. (knex does exactly this — a builder with a `.clone()`.) |
| **Flyweight** (40) | Shares one immutable object across many contexts to save memory. | Prototype **copies** to give each instance its own state; Flyweight **shares** to avoid copies. Note our `Enemy` uses **both**: the mesh is a flyweight (shared), the hp/position are cloned. Naming that in an interview is a strong signal. |
| **Memento** (49) | Snapshots an object's state so it can be restored later (undo). | A Memento *is* a deep clone — but the intent differs: Prototype clones to **create new objects**, Memento clones to **restore an old one**. Same mechanism, opposite direction. |
| **Singleton** (29) | Exactly one instance, globally. | The literal opposite. Where Singleton says "there is only one," a prototype registry says "here is the canonical one — never touch it, take a copy." A registry is the *safe* alternative when you were reaching for a mutable singleton. |

---

## Visual / Diagram description

### Diagram 1: Participants

```
        ┌────────────────────────────────────┐
        │        «interface» Prototype        │
        │                                    │
        │  + clone() : Prototype              │
        └─────────────────┬──────────────────┘
                          │ implements
              ┌───────────┴────────────┐
              ▼                        ▼
  ┌───────────────────────┐  ┌───────────────────────┐
  │        Enemy          │  │      HttpClient       │
  │                       │  │                       │
  │  - model   (SHARED)   │  │  - agent   (SHARED)   │
  │  - ai      (SHARED)   │  │  - baseUrl (SHARED)   │
  │  - hp      (COPIED)   │  │  - headers (COPIED)   │
  │  - loot    (COPIED)   │  │  - timeout (COPIED)   │
  │                       │  │                       │
  │  + clone() : Enemy    │  │  + clone(): HttpClient│
  └───────────────────────┘  └───────────────────────┘
              ▲
              │ stores specimens; hands out clones
  ┌───────────┴────────────────────────────┐
  │         PrototypeRegistry               │
  │                                        │
  │  - #prototypes : Map<string, Prototype>│
  │                                        │
  │  + register(key, proto)                 │
  │  + spawn(key, overrides) : Prototype    │
  │      └─ returns proto.clone(), NEVER    │
  │         the stored specimen itself      │
  └────────────────────────────────────────┘
```

Every prototype exposes one method, `clone()`. Inside it, each field is a decision: **shared** (immutable + expensive) or **copied** (mutable). The registry holds the specimens and — critically — always returns a clone, never the specimen.

### Diagram 2: Shallow vs deep — draw this one from memory

```
  ORIGINAL                     SHALLOW CLONE  { ...orig }
  ┌──────────────┐             ┌──────────────┐
  │ hp:      100 │             │ hp:      100 │  ← primitive: truly copied ✅
  │ name:  "Orc" │             │ name:  "Orc" │  ← primitive: truly copied ✅
  │ loot:    ●───┼──────┐      │ loot:    ●───┼──┐
  └──────────────┘      │      └──────────────┘  │
                        ▼                        │
                 ┌─────────────────┐             │
                 │ { gold: 40,     │◀────────────┘  ⚠️ SAME OBJECT.
                 │   items: [...] }│                clone.loot.gold = 0
                 └─────────────────┘                also zeroes the ORIGINAL.


  ORIGINAL                     DEEP CLONE  structuredClone(orig) / clone()
  ┌──────────────┐             ┌──────────────┐
  │ hp:      100 │             │ hp:      100 │
  │ loot:    ●───┼──┐          │ loot:    ●───┼──┐
  └──────────────┘  │          └──────────────┘  │
                    ▼                            ▼
          ┌─────────────────┐          ┌─────────────────┐
          │ { gold: 40 }    │          │ { gold: 40 }    │  ✅ two separate
          └─────────────────┘          └─────────────────┘     objects
```

### Diagram 3: The JS prototype chain (delegation, not copying)

```
   goblin  ──────▶ enemyProto ──────▶ Object.prototype ──────▶ null
   ┌──────────┐   ┌───────────────┐   ┌──────────────────┐
   │name:"Gob"│   │hp: 100        │   │hasOwnProperty()  │
   │damage: 7 │   │speed: 5       │   │toString()        │
   └──────────┘   │attack()       │   └──────────────────┘
                  └───────────────┘

   goblin.damage  → found on goblin           (own property, 0 hops)
   goblin.hp      → not on goblin → hop up → found on enemyProto  (1 hop)
   goblin.attack  → not on goblin → hop up → found on enemyProto  (1 hop)
   goblin.toString→ not on goblin → hop → hop → Object.prototype  (2 hops)
   goblin.wings   → walks to null → undefined

   WRITING is different:
   goblin.hp = 80   creates an OWN `hp` on goblin (SHADOWS the prototype's).
                    enemyProto.hp is still 100. ✅

   BUT:
   goblin.loot.gold = 50   is a READ of goblin.loot (→ enemyProto.loot),
                           then a MUTATION of that shared object.
                           EVERY enemy now has 50 gold. ⚠️ Same aliasing bug.
```

The lesson in one line, whiteboard-ready: **reads delegate up the chain; writes to the object itself shadow; but writes *through* a delegated reference reach the shared object and hit everyone.**

---

## Real world examples

### 1. Node.js configuration defaults — cloning before override

Nearly every configurable Node library keeps a module-level `defaults` object and produces per-instance config by **copying defaults and merging overrides**. Axios does this (`axios.create(config)` merges instance config over library defaults, then each request merges again over the instance's). The bug this pattern is guarding against is exactly the one in §2: if the library handed out the *same* nested `headers` object to every instance, one caller setting an `Authorization` header would leak their bearer token into every other caller's requests. That's a security incident, not a style issue — which is why "clone the defaults, don't share them" is load-bearing here.

### 2. Game engines — entity spawning from a prefab

Unity calls it a *Prefab*, Unreal calls it a *Blueprint class*, and the runtime call is literally `Instantiate(prefab)` — take a fully-configured template entity and stamp out a copy at a new position. The heavy assets (mesh, textures, compiled animation graphs) are **shared by reference** across every instance; only the mutable per-entity state (transform, health, current AI node) is copied. That split is the exact `shared vs copied` decision in our `Enemy.clone()`, and it's why 500 goblins cost 500 small state objects, not 500 meshes. The same architecture appears in browser games built on three.js: one loaded `Geometry`/`Material` (shared), many `Mesh` instances (cloned).

### 3. Cloning a configured HTTP client

A pattern you'll write yourself within a year of backend work:

```javascript
class ApiClient {
  constructor({ baseUrl, headers = {}, timeoutMs = 5000, agent }) {
    this.baseUrl = baseUrl;
    this.agent = agent;              // an http.Agent — SHARED on purpose:
                                     //   it owns the TCP connection pool. Cloning
                                     //   it would defeat keep-alive entirely.
    this.headers = { ...headers };
    this.timeoutMs = timeoutMs;
  }

  // Hand out an independently-configurable copy that still reuses the pool.
  clone(overrides = {}) {
    return new ApiClient({
      baseUrl:   overrides.baseUrl   ?? this.baseUrl,
      agent:     this.agent,                              // SHARED (connections!)
      headers:   { ...this.headers, ...(overrides.headers ?? {}) },  // COPIED
      timeoutMs: overrides.timeoutMs ?? this.timeoutMs,   // COPIED
    });
  }
}

const base = new ApiClient({ baseUrl: 'https://api.acme.com',
                             headers: { 'x-app': 'checkout' },
                             agent: new http.Agent({ keepAlive: true }) });

// Per-request-scoped clients. Each carries its own trace id and timeout;
// all of them reuse ONE warm TCP connection pool.
const forRequest = base.clone({ headers: { 'x-trace': traceId }, timeoutMs: 1500 });
```

This is Prototype at its most honest: the expensive thing (the connection pool with its warm TLS sessions) is *deliberately shared*, the cheap mutable thing (headers, timeout) is copied. Getting that split right is the skill; the `clone()` method is just where you write it down.

---

## Trade-offs

| Pros | What it buys you |
|---|---|
| **Skips expensive construction** | Pay the DB read / file parse / handshake once, amortized over N instances. |
| **Scales with instance count** | 500 spawns cost 500 shallow state copies, not 500 asset loads. |
| **Config lives in data, not classes** | Add a new enemy variant by registering a specimen — no new subclass, no factory edit. |
| **Runtime-configurable variants** | Prototypes can be loaded from JSON at startup; a factory hierarchy is fixed at compile time. |
| **Composes with Builder & Flyweight** | Build the prototype once (Builder), share its heavy immutable parts (Flyweight), clone the rest. |

| Cons | What it costs you |
|---|---|
| **Shallow-copy aliasing bugs** | The #1 killer. Silent, cross-tenant, and it corrupts the prototype itself so it worsens over time. |
| **`clone()` must be maintained** | Add a field, forget to copy it in `clone()`, get a shared-reference bug months later. No compiler catches this. |
| **Deep clone can be expensive** | Blindly deep-cloning a fat object gives back all the performance you were buying. |
| **Circular references** | A hand-written `clone()` will infinitely recurse unless you track visited nodes. (`structuredClone` handles it.) |
| **`structuredClone` loses class identity** | Returns a plain object: no methods, no `instanceof`, and it *throws* on function-valued properties. |
| **Wrong for objects with identity** | DB rows, sockets, file handles, connections — copying them is meaningless or unsafe. |

**The sweet spot:** Use Prototype when construction is **expensive** and the object is **value-like** (data + behaviour, no external identity). In `clone()`, treat every field as an explicit decision — **share** what is immutable and expensive; **copy** what is mutable and per-instance. When the object is pure data, `structuredClone` is your deep copy; when it has methods, hand-write `clone()`. And remember JS gives you delegation (`Object.create`) for free: **delegate the behaviour, clone the state.**

---

## Common interview questions on this topic

### Q1: "What's the difference between a shallow and a deep clone? Show me a bug."
**Hint:** Shallow copies top-level properties only — nested objects are copied *by reference*, so the original and the clone share them (**aliasing**). Show the two-line bug: `const b = {...a}; b.tags.push('x');` → `a.tags` also has `'x'`. Then name the fixes and their limits: `structuredClone` (deep, handles `Date`/`Map`/`Set`/cycles, but **drops methods, throws on functions, loses `instanceof`**), a hand-written `clone()` (full control, must be maintained), and explicitly *not* `JSON.parse(JSON.stringify())` (lossy on `Date`, `Map`, `Set`, `undefined`, functions; throws on cycles).

### Q2: "JavaScript has prototypal inheritance. Isn't the Prototype pattern redundant here?"
**Hint:** They're related but not the same, and the difference is **delegation vs copying**. `Object.create(proto)` gives you an object that *points at* the prototype and looks up missing properties there — nothing is copied, and a later change to the prototype is visible to every delegate. The GoF pattern *copies* — each clone gets its own state and is thereafter independent. JS is unusual in giving you delegation as a language primitive, and the best designs use both: delegate the shared behaviour (that's what class methods already do — they live on `Enemy.prototype`, one function object for all instances) and clone the mutable state.

### Q3: "Prototype vs Factory — when would you pick each?"
**Hint:** Factory constructs from a **class** — good when the variants are known at compile time and differ in *behaviour*. Prototype copies an **instance** — good when variants differ in *configuration*, when the set of variants is loaded at runtime (from JSON/DB), or when construction is expensive. Killer line: with a factory, adding an enemy type means writing a subclass and editing a switch; with a prototype registry, it means registering one more specimen — often from a config file, with no code change.

### Q4: "You have 500 game enemies from one prototype and mutating one changes all of them. Debug it."
**Hint:** Shallow clone. Some nested field — `position`, `loot`, `stats`, a `Set` of buffs — is being copied by reference, so all 500 share one object. Find it by checking `a.loot === b.loot` (should be `false`; if `true`, that's your field). Fix: copy it in `clone()`. Then say the thing that shows depth: *don't* fix it by deep-cloning everything — the mesh and AI tree **must** stay shared or you'll blow up memory and load time. Cloning is a per-field decision.

### Q5: "What's the relationship between Prototype and Flyweight? And Memento?"
**Hint:** Prototype and Flyweight are opposite responses to the same pressure. Flyweight **shares** one immutable object across many contexts to save memory; Prototype **copies** so each instance owns its state. A good `clone()` uses both at once — share the mesh (flyweight), copy the hp (prototype). Memento is a deep clone with a different *intent*: Prototype clones to create a **new** object, Memento clones to **restore an old one** (undo). Same mechanism, opposite direction.

---

## Practice exercise

### Build a `DocumentTemplate` prototype system (~30-40 min)

You're building a document service. Loading a template is expensive (parse + validate + fetch branding); creating a document from one must be cheap.

**What to build, in one runnable `.js` file:**

1. A `DocumentTemplate` class. Its constructor takes a `name` and simulates ~40ms of expensive work (busy-wait, like `loadModel` above) to produce:
   - `schema` — a frozen object describing the fields (expensive, **immutable**)
   - `branding` — a frozen `{ logo, primaryColor, footer }` (expensive, **immutable**)
   - `sections` — an array of `{ heading, body }` objects (**mutable**, per-document)
   - `metadata` — `{ author: null, tags: [], createdAt: null }` (**mutable**, nested)
   - `reviewers` — a `Set` (**mutable**)
2. A `clone()` method that shares `schema` and `branding` by reference but gives each clone its own `sections`, `metadata` (including the nested `tags` array), and `reviewers`. Do **not** use `structuredClone` here — you must hand-write it, because the class has methods.
3. A `PrototypeRegistry` with `register(key, proto)` and `create(key, overrides)`. `create()` must return a **clone**, never the stored specimen.
4. Register three templates: `invoice`, `contract`, `report`.
5. A `main()` demo that:
   - Times registering the 3 templates (expect ~120ms).
   - Times creating **200 documents** by cloning (expect single-digit ms). Print both numbers.
   - **Proves isolation:** mutate one document (push a section, add a tag, add a reviewer, set the author) and assert that a sibling document and the registered prototype are **unchanged**.
   - **Proves deliberate sharing:** assert `docA.schema === docB.schema` is `true`, and explain in a comment why that's correct and not a bug.
6. **The bug-hunt bonus.** Add a second method, `badClone()`, that does `return Object.assign(Object.create(Object.getPrototypeOf(this)), { ...this })`. Use it to create two documents, mutate one's `metadata.tags`, and print the other's. Write a one-line comment stating exactly which line causes the leak and why.

**What to produce:** the runnable file, plus a short comment block at the bottom listing — field by field — which fields you shared and which you copied, and the rule you used to decide. That table *is* the Prototype pattern; everything else is syntax.

---

## Quick reference cheat sheet

- **Prototype** = create new objects by **cloning a configured instance**, not by constructing from scratch.
- **Use it when construction is expensive:** DB round-trip, file parse, TLS handshake, compiled regexes, big default trees.
- **Shallow copy** (`{...obj}`, `Object.assign`) copies top-level props; **nested objects are shared by reference** → aliasing bugs.
- **The aliasing bug:** mutate a clone's nested field, and the original *and every other clone* change with it. Silent, cross-tenant, and it poisons the prototype so it gets worse over time.
- **`structuredClone(x)`** = built-in deep clone (Node 17+). Handles nested objects, `Date`, `Map`, `Set`, `RegExp`, **circular refs**.
- **`structuredClone` limits:** **throws** on functions, **loses class identity** (returns a plain object, no methods, no `instanceof`), flattens getters. Use it for **data**; hand-write `clone()` for objects with **behaviour**.
- **Never** `JSON.parse(JSON.stringify(x))` — kills `Date`, `Map`, `Set`, `undefined`, functions; throws on cycles.
- **`clone()` is a decision per field:** **share** what is immutable + expensive (mesh, template, connection pool); **copy** what is mutable + per-instance (hp, headers, tags).
- **JS is special:** prototypal inheritance *is* this idea in the language. `Object.create(proto)` links an object to a prototype — that's **delegation**, not copying.
- **Delegation vs copying:** delegates *see* later changes to the prototype and share its nested state; clones are independent snapshots.
- **`obj.__proto__` vs `Fn.prototype`:** `__proto__` is the link an *instance* follows; `Fn.prototype` is the object *instances get linked to*. `new Enemy()` ⇒ `e.__proto__ === Enemy.prototype`.
- **Your class methods are already prototype-shared** — 10,000 instances, one `attack` function. Free Flyweight.
- **Prototype registry:** a `Map<string, prototype>`; `create(key)` returns a **clone**, never the stored specimen.
- **Don't use it for:** cheap constructors, or objects with identity (DB rows, sockets, file handles, connections).
- **vs Builder:** Builder constructs from parts; Prototype copies a finished whole. Best combo: **Builder makes the prototype once, then clone forever** (that's knex's `.clone()`).
- **vs Flyweight:** Flyweight *shares* to save memory; Prototype *copies* to isolate state. A good `clone()` does both at once.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [32 — Builder Pattern](./32-pattern-builder.md) — construct step by step; use a Builder to make the prototype, then clone it forever |
| **Next** | [34 — Adapter Pattern](./34-pattern-adapter.md) — the first structural pattern: making incompatible interfaces talk |
| **Related** | [40 — Flyweight Pattern](./40-pattern-flyweight.md) — the mirror image: *share* immutable state instead of copying it |
| **Related** | [49 — Memento Pattern](./49-pattern-memento.md) — a deep clone with a different intent: snapshot to *restore*, not to *create* |
