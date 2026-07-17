# 40 — Flyweight Pattern
## Category: LLD Patterns

---

## What is this?

Flyweight is a memory-saving pattern. When you need a huge number of objects that are *mostly identical*, you stop duplicating the identical parts and share **one copy** of them across every object.

Think of a library. There might be 500 people who have read *The Hobbit*. The library does not print 500 copies of the book so that each person can own the words. It keeps a handful of physical copies (the shared, unchanging content) and tracks per-person data separately: who borrowed it, when it's due, which page they're on. The words are shared; the bookmark is yours.

That split — **what is shared** vs **what is yours** — is the whole pattern.

---

## Why does it matter?

Because objects are not free. Every JavaScript object has header overhead, every string field is a real allocation, and every image buffer you copy is real RAM. When your object count goes from 1,000 to 1,000,000, "a few KB each" quietly becomes "several gigabytes" and your Node process dies with `JavaScript heap out of memory`.

**Real-work angle.** You hit this whenever you have a *many, similar, long-lived* population of objects:
- A game rendering a forest, a particle system, or a tile map.
- A text editor holding every character on screen as an object.
- A trading system holding millions of order objects that all reference the same few hundred instruments.
- A log processor holding millions of parsed events that all reference the same handful of event schemas.

**Interview angle.** Flyweight is the pattern interviewers reach for when they want to see whether you can reason about *memory*, not just about class hierarchies. The question is almost never "implement flyweight" — it's "you have 10 million particles and 2 GB of RAM, what do you do?" The expected answer is: separate intrinsic from extrinsic state, share the intrinsic part behind a factory, and show the arithmetic.

If you don't know this pattern, your instinct is "just make a class with all the fields" — and that instinct scales to about 100,000 objects before it falls over.

---

## The core idea — explained simply

### The Forest Analogy

You're building a game. The map has **1,000,000 trees**.

Your first instinct is a `Tree` class where each tree knows everything about itself:

```js
// BAD — every tree owns a full copy of the heavy data
class Tree {
  constructor(x, y, scale, name, barkColor, leafColor, meshData, textureBitmap) {
    this.x = x;                        // 8 bytes
    this.y = y;                        // 8 bytes
    this.scale = scale;                // 8 bytes
    this.name = name;                  // "Oak"
    this.barkColor = barkColor;        // "#5A3A22"
    this.leafColor = leafColor;        // "#2E7D32"
    this.meshData = meshData;          // ~2 KB of vertex data
    this.textureBitmap = textureBitmap;// ~3 KB of pixels
  }
}
```

Now look at the actual data. How many *kinds* of tree are there? Three: Oak, Pine, Birch. So `meshData` and `textureBitmap` only ever have **three distinct values** — but you just allocated a million copies of them.

Here's the key observation: walk any two Oak trees in the forest and ask "what is different about them?" The answer is only their position and size. The mesh is the same. The texture is the same. The bark colour is the same. The name is the same.

So split the fields into two buckets:

| Bucket | Name | Meaning | In the forest | Where it lives |
|---|---|---|---|---|
| Shared | **Intrinsic state** | Identical across many objects, never changes, doesn't depend on context | `name`, `barkColor`, `leafColor`, `meshData`, `textureBitmap` | ONE `TreeType` object per kind — 3 total |
| Unique | **Extrinsic state** | Different for every object, depends on where/when/how the object is used | `x`, `y`, `scale` | On each individual `Tree` — 1,000,000 of them |

Then the million `Tree` objects each hold just three numbers plus a **pointer to their shared type**. A pointer is 8 bytes. It costs the same whether it points at 3 KB or 3 GB.

```
BEFORE: 1,000,000 trees × (mesh + texture + colours + coords)
AFTER:  1,000,000 trees × (coords + one pointer)  +  3 × (mesh + texture + colours)
```

**Naming the two halves precisely is the entire skill.** Everything else — the factory, the cache, the `Object.freeze` — is mechanics. If you put a mutable, per-object field into the shared bucket, the pattern doesn't just under-perform, it becomes a *correctness bug* (see concept 5). If you leave a shared field in the per-object bucket, you get no savings at all.

---

## Key concepts inside this topic

### 1. The problem it solves — start with the painful code

Here is code with no flyweight. It is honest, obvious, and it kills your process.

```js
// naive-forest.js — DO NOT SHIP THIS
class Tree {
  constructor(x, y, scale, name, barkColor, leafColor) {
    this.x = x;
    this.y = y;
    this.scale = scale;
    this.name = name;
    this.barkColor = barkColor;
    this.leafColor = leafColor;
    // The killers: each tree builds its OWN copy of the heavy assets.
    this.mesh = new Float32Array(500);      // 500 × 4 bytes  = 2,000 B
    this.texture = new Uint8Array(3000);    // 3000 × 1 byte  = 3,000 B
  }
}

const forest = [];
for (let i = 0; i < 1_000_000; i++) {
  forest.push(new Tree(Math.random() * 5000, Math.random() * 5000, 1, 'Oak', '#5A3A22', '#2E7D32'));
}
```

Run that with `node --max-old-space-size=4096` and you get:

```
FATAL ERROR: Ineffective mark-compacts near heap limit
Allocation failed - JavaScript heap out of memory
```

The bug is not a typo. The bug is *architectural*: you allocated a million copies of data that only had three distinct values.

### 2. The structure — who plays which role

Flyweight has exactly four participants. Learn these names; interviewers use them.

| Participant | In our forest | Job |
|---|---|---|
| **Flyweight** | `TreeType` | Holds ONLY intrinsic state. Immutable. Shared by many contexts. |
| **Flyweight Factory** | `TreeTypeFactory` | Owns a cache (a `Map`). Given intrinsic state, returns an *existing* flyweight if one matches; otherwise creates and caches one. Clients never `new` a flyweight directly. |
| **Context** | `Tree` | Holds extrinsic state (`x, y, scale`) plus a reference to its flyweight. Tiny. |
| **Client** | `Forest` | Creates contexts, asks the factory for flyweights, and passes extrinsic state in when operating. |

The one non-negotiable rule: **clients must go through the factory.** The moment someone calls `new TreeType(...)` directly, duplicates leak back in and the savings evaporate.

### 3. Full JavaScript implementation

```js
// flyweight-forest.js

/* ------------------------------------------------------------------ *
 * FLYWEIGHT — intrinsic state only. Immutable. There will be 3 of
 * these no matter how large the forest gets.
 * ------------------------------------------------------------------ */
class TreeType {
  constructor(name, barkColor, leafColor) {
    this.name = name;
    this.barkColor = barkColor;
    this.leafColor = leafColor;

    // The heavy shared assets. Loaded once per TYPE, never per tree.
    this.mesh = new Float32Array(500);    // ~2 KB of vertex data
    this.texture = new Uint8Array(3000);  // ~3 KB of pixel data

    // Freeze so no caller can mutate state that 1,000,000 trees rely on.
    // Without this, one bad assignment silently repaints the whole forest.
    Object.freeze(this);
  }

  // Extrinsic state is PASSED IN, never stored. This is the signature
  // that gives the pattern away: the flyweight is a pure function of
  // (its own intrinsic state, the caller's extrinsic state).
  draw(canvas, x, y, scale) {
    canvas.push(`${this.name}@(${x.toFixed(0)},${y.toFixed(0)}) x${scale} bark=${this.barkColor}`);
  }
}

/* ------------------------------------------------------------------ *
 * FLYWEIGHT FACTORY — the cache. Returns an EXISTING TreeType when one
 * with the same intrinsic state already exists.
 * ------------------------------------------------------------------ */
class TreeTypeFactory {
  static #cache = new Map();   // key -> TreeType
  static #created = 0;         // instrumentation, so we can prove the win

  static getTreeType(name, barkColor, leafColor) {
    // The key must contain EVERY intrinsic field. Miss one and you will
    // hand back a flyweight whose colour is wrong.
    const key = `${name}|${barkColor}|${leafColor}`;

    let type = this.#cache.get(key);
    if (!type) {
      type = new TreeType(name, barkColor, leafColor);
      this.#cache.set(key, type);
      this.#created++;
    }
    return type;                // <-- shared instance, not a copy
  }

  static get instancesCreated() { return this.#created; }
  static get cacheSize() { return this.#cache.size; }
}

/* ------------------------------------------------------------------ *
 * CONTEXT — extrinsic state + a reference. ~30 bytes of real payload.
 * ------------------------------------------------------------------ */
class Tree {
  constructor(x, y, scale, type) {
    this.x = x;          // 8 bytes (double)
    this.y = y;          // 8 bytes
    this.scale = scale;  // 8 bytes
    this.type = type;    // 8 bytes — a POINTER to the shared TreeType
  }

  draw(canvas) {
    // Hand our extrinsic state to the shared flyweight.
    this.type.draw(canvas, this.x, this.y, this.scale);
  }
}

/* ------------------------------------------------------------------ *
 * CLIENT
 * ------------------------------------------------------------------ */
class Forest {
  constructor() { this.trees = []; }

  plant(x, y, scale, name, barkColor, leafColor) {
    const type = TreeTypeFactory.getTreeType(name, barkColor, leafColor);
    this.trees.push(new Tree(x, y, scale, type));
  }

  draw() {
    const canvas = [];
    for (const tree of this.trees) tree.draw(canvas);
    return canvas;
  }
}

/* ------------------------------------------------------------------ *
 * DEMO
 * ------------------------------------------------------------------ */
function main() {
  const SPECIES = [
    ['Oak',   '#5A3A22', '#2E7D32'],
    ['Pine',  '#4E342E', '#1B5E20'],
    ['Birch', '#EFEBE9', '#7CB342'],
  ];

  const forest = new Forest();
  const before = process.memoryUsage().heapUsed;

  for (let i = 0; i < 1_000_000; i++) {
    const [name, bark, leaf] = SPECIES[i % 3];
    forest.plant(Math.random() * 5000, Math.random() * 5000, 0.8 + Math.random() * 0.4, name, bark, leaf);
  }

  const after = process.memoryUsage().heapUsed;

  console.log('trees planted        :', forest.trees.length.toLocaleString());
  console.log('TreeType objects made:', TreeTypeFactory.instancesCreated);  // 3
  console.log('flyweight cache size :', TreeTypeFactory.cacheSize);         // 3
  console.log('heap used (MB)       :', ((after - before) / 1024 / 1024).toFixed(1));

  // Proof of sharing: two different trees of the same species are backed
  // by the LITERAL SAME object in memory.
  const a = forest.trees.find(t => t.type.name === 'Oak');
  const b = forest.trees.filter(t => t.type.name === 'Oak')[500];
  console.log('same shared instance?:', a.type === b.type);                 // true
}

main();
```

Typical output:

```
trees planted        : 1,000,000
TreeType objects made: 3
flyweight cache size : 3
heap used (MB)       : 89.4
same shared instance?: true
```

One million trees. **Three** `TreeType` objects. That `a.type === b.type` returning `true` is the pattern working — identity, not equality.

### 4. Show the memory math — this IS the payoff

Do this arithmetic out loud in an interview. It's what separates "I read about flyweight" from "I understand why it exists."

**Per-object sizes.**

| Field | Naive `Tree` | Flyweight `Tree` |
|---|---|---|
| `x`, `y`, `scale` (3 doubles) | 24 B | 24 B |
| `type` pointer | — | 8 B |
| `name` string ref | 8 B | — (on shared type) |
| `barkColor`, `leafColor` string refs | 16 B | — |
| `mesh` (Float32Array, 500 floats) | ~2,000 B | — |
| `texture` (Uint8Array, 3,000 bytes) | ~3,000 B | — |
| **Payload per tree** | **~5,048 B (~5 KB)** | **~32 B** |

**Totals for 1,000,000 trees.**

| | Naive | Flyweight |
|---|---|---|
| Per-tree cost | 1,000,000 × 5,048 B | 1,000,000 × 32 B |
| = | **5,048,000,000 B ≈ 5.0 GB** | **32,000,000 B ≈ 32 MB** |
| Shared type cost | 0 | 3 × ~5,048 B ≈ **15 KB** |
| **Total** | **≈ 5.0 GB** | **≈ 32 MB** |
| Outcome | Node's default heap cap is ~4 GB → **process dies** | Fits comfortably in a laptop tab |
| Ratio | — | **~157× smaller** |

Step by step, the way you'd say it at a whiteboard:

1. Heavy assets per tree: 2 KB mesh + 3 KB texture = **5 KB**.
2. Naive: 5 KB × 1,000,000 = **5 GB**. Node's old-space default is ~4 GB. Dead.
3. Flyweight: coords (24 B) + pointer (8 B) = **32 B** per tree → 32 B × 1e6 = **32 MB**.
4. Plus the shared types: 3 × 5 KB = **15 KB** — a rounding error.
5. **5 GB → 32 MB.** That's the pattern.

(Real V8 heap usage will be somewhat higher than the raw payload — every JS object carries a hidden-class pointer and header, roughly 16–24 bytes, plus array backing-store overhead. That's why the demo prints ~90 MB rather than exactly 32 MB. The *ratio* is what matters, and the ratio holds.)

The rule you can carry away: **your saving is roughly `heavyBytes × (N − distinctTypes)`.** Flyweight pays off when `N` is huge, `distinctTypes` is small, and `heavyBytes` is not tiny. If any of those three fails, don't bother.

### 5. The hard requirement: intrinsic state MUST be immutable

This is where flyweight bites people. Watch what happens without `Object.freeze`:

```js
// THE BUG — a mutable flyweight
class MutableTreeType {
  constructor(name, barkColor) {
    this.name = name;
    this.barkColor = barkColor;
    // no Object.freeze()
  }
}

const oakType = new MutableTreeType('Oak', '#5A3A22');
const treeA = { x: 10, y: 20, type: oakType };
const treeB = { x: 90, y: 40, type: oakType };
// ... 999,998 more trees, all pointing at the SAME oakType

// Autumn arrives. A well-meaning dev writes:
treeA.type.barkColor = '#8D6E63';   // "just this one tree, right?"

console.log(treeB.type.barkColor);  // '#8D6E63'  ← EVERY oak changed
```

One assignment repainted a million trees. And it's a nightmare to debug, because the mutation site and the symptom are in completely different parts of the codebase — the stack trace points at a tree that isn't even on screen.

`Object.freeze(this)` in the flyweight's constructor makes this a loud failure instead of a silent one:

```js
'use strict';
const oak = new TreeType('Oak', '#5A3A22', '#2E7D32');
oak.barkColor = '#8D6E63';
// TypeError: Cannot assign to read only property 'barkColor' of object '#<TreeType>'
```

Two caveats you should know:
- `Object.freeze` is **shallow**. `oak.mesh` is still a mutable `Float32Array` — freezing the object stops you *reassigning* `oak.mesh`, not writing into `oak.mesh[3]`. If that matters, hand out a read-only view or copy on read.
- In **non-strict** mode, writing to a frozen property fails *silently*. ES modules and class bodies are always strict, so you're fine there — but if you're in a plain CommonJS script, add `'use strict'`.

**Mental model:** a flyweight is a *value*, not an *entity*. It has no identity of its own and no lifecycle. If you ever feel the urge to mutate one, you've put extrinsic state in the wrong bucket.

### 6. Second example: glyphs in a text editor

Open a 500,000-character document. Naive editors modelled each character on screen as an object carrying its font, size, glyph bitmap, and kerning table — a few hundred bytes each, most of it duplicated.

But there are only ~128 distinct character *codes* in common use. So share one `Glyph` per **code**, and store the per-position data (row, column, which style run it belongs to) in the document.

```js
class Glyph {
  constructor(charCode) {
    this.charCode = charCode;
    this.char = String.fromCharCode(charCode);
    this.width = 8;                          // advance width in px
    this.bitmap = new Uint8Array(8 * 16);    // 128 B rasterised glyph
    Object.freeze(this);
  }

  // Position is EXTRINSIC — passed in, never stored on the glyph.
  render(ctx, row, col) {
    return `${this.char}@[r${row},c${col}] w=${this.width}`;
  }
}

class GlyphFactory {
  static #cache = new Map();
  static get(charCode) {
    if (!this.#cache.has(charCode)) this.#cache.set(charCode, new Glyph(charCode));
    return this.#cache.get(charCode);
  }
  static get size() { return this.#cache.size; }
}

class Document {
  constructor() { this.chars = []; }   // [{ glyph, row, col }]

  insert(text, row) {
    [...text].forEach((ch, col) => {
      this.chars.push({ glyph: GlyphFactory.get(ch.charCodeAt(0)), row, col });
    });
  }

  render() { return this.chars.map(c => c.glyph.render(null, c.row, c.col)); }
}

const doc = new Document();
for (let row = 0; row < 10_000; row++) doc.insert('the quick brown fox jumps over the lazy dog', row);

console.log('characters placed:', doc.chars.length);   // 430,000
console.log('Glyph objects    :', GlyphFactory.size);  // 27  ← 26 letters + space
```

430,000 characters on screen. **27** glyph objects in memory. Same shape as the forest: many contexts, few flyweights.

### 7. A real Node.js ecosystem example

**JavaScript string interning** is flyweight built into the language. V8 keeps a table of string literals; every occurrence of `'user'` in your source resolves to the *same* heap object.

```js
const a = 'user';
const b = 'user';
console.log(a === b);  // true — literally the same object in V8's string table

// Built at runtime, so it may not be interned:
const c = ['u','s','e','r'].join('');
console.log(c === a);          // true (value equality; V8 often internalises this too)
```

This is why parsing a 10 GB JSON log file where the key `"level"` appears 40 million times doesn't cost 40 million string allocations — V8 hands back the same interned string. You get flyweight for free without asking.

**Other flyweights hiding in your `node_modules`:**
- **React** shares one *component definition* (the function/class) across every element instance. `<Button/>` used 10,000 times creates 10,000 tiny element descriptors (`{ type, props, key }`) that all point at the same `Button` function. `type` is the flyweight; `props` is the extrinsic state.
- **Connection pools** (`pg.Pool`, `mysql2`) reuse a small set of expensive TCP+TLS connections across thousands of logical queries. The connection is the shared heavyweight; the query is extrinsic.
- **`Buffer.allocUnsafe` slab allocation** — Node keeps a pre-allocated 8 KB slab and carves small buffers out of it rather than making a fresh allocation per buffer.

### 8. When NOT to use it / how it's abused

Flyweight is a **memory** optimization, and it is not free. Do not reach for it until you have measured.

**Skip it when:**
- **N is small.** 500 objects × 5 KB = 2.5 MB. Who cares. You added a factory, a cache, and an indirection to save nothing.
- **Objects are mostly unique.** If 900,000 of your million trees have a distinct mesh, your cache holds 900,000 entries and you've built a memory *leak* with extra steps.
- **The "heavy" data isn't heavy.** Sharing three short strings saves you a pointer's worth of memory and costs you a class.
- **You need per-object identity or lifecycle.** If a tree needs to be individually mutated, versioned, or garbage-collected, don't try to force it into shared state.

**Ways it gets abused:**
- **The unbounded cache.** `Map` holds strong references forever. If flyweight keys come from user input (per-tenant, per-session), your "cache" grows without bound. Fix: bound it, use an LRU, or use a `WeakRef`/`FinalizationRegistry` if you truly need collectible flyweights.
- **The leaky key.** If your cache key doesn't include *every* intrinsic field, you'll return a flyweight whose colour is silently wrong. This bug is subtle and horrible.
- **The mutable flyweight** — see concept 5.
- **Premature optimization.** Profile first: `node --inspect` + a heap snapshot in Chrome DevTools, or `process.memoryUsage().heapUsed`. If the heavy objects aren't in the top 3 retainers, you're solving a problem you don't have.

**The CPU trade.** You are swapping RAM for indirection. Every access now goes `tree → type → mesh` instead of `tree → mesh`: an extra pointer hop and a likely cache miss. Every creation goes through a `Map.get` and a string-key build. On a hot render loop that runs 60×/second, that overhead is real. Flyweight wins because RAM is the wall you actually hit — but say the cost out loud, don't pretend it doesn't exist.

### 9. Related patterns and how they differ

This is the #1 interview follow-up. "How is that different from a singleton / object pool / cache?"

| Pattern | Intent | Instances | Mutable? | Key difference from Flyweight |
|---|---|---|---|---|
| **Flyweight** | Save memory by sharing immutable intrinsic state across many objects | Many flyweights (one per distinct intrinsic value) | **No — must be immutable** | — |
| **Singleton** | Guarantee exactly ONE instance of a class, globally | Exactly 1 | Usually yes | Flyweight has *many* instances (3 tree types). Singleton is about global access; Flyweight is about memory. |
| **Object Pool** | Reuse *expensive-to-create* objects (DB connections, threads) | Fixed-size pool | Yes — objects are checked out, mutated, returned | Pool objects are **borrowed exclusively**; flyweights are **shared concurrently**. Pool saves *CPU/setup time*; Flyweight saves *RAM*. |
| **Cache / Memoization** | Avoid recomputing an expensive result | As many as you cache | Values usually immutable | A cache is about *time*; flyweight is about *space*. The flyweight factory happens to use a cache, but that's mechanism, not intent. |
| **Prototype** | Create new objects by cloning an existing one | Many, independent | Yes | Prototype gives each client its own **copy**. Flyweight gives every client the **same** object. Opposite answers to "should we duplicate?" |
| **Composite** | Treat individual objects and groups uniformly (trees of objects) | Many | Yes | Often *combined* with Flyweight — the leaves of a Composite tree are frequently flyweights (this is the classic document-editor design). |

The one-liner to memorise: **Singleton = one object, period. Flyweight = few objects, many references. Pool = few objects, lent out one at a time.**

---

## Visual / Diagram description

**Class diagram.**

```
                  ┌──────────────────────────────┐
                  │      TreeTypeFactory         │   FLYWEIGHT FACTORY
                  ├──────────────────────────────┤
                  │ - #cache: Map<string,TreeType>│
                  ├──────────────────────────────┤
                  │ + getTreeType(name,bark,leaf) │──┐ returns EXISTING
                  │     : TreeType                │  │ instance if key hits
                  └──────────────┬───────────────┘  │
                                 │ creates/caches    │
                                 ▼                   │
   ┌─────────────┐  type    ┌──────────────────────┐ │
   │    Tree     │─────────▶│      TreeType        │◀┘  FLYWEIGHT
   │  (CONTEXT)  │  1       ├──────────────────────┤    (immutable,
   ├─────────────┤   ▲      │ + name       (intrin)│     Object.freeze)
   │ + x   ✱     │   │      │ + barkColor  (intrin)│
   │ + y   ✱     │   │ many │ + leafColor  (intrin)│
   │ + scale ✱   │   │      │ + mesh    ~2KB       │
   │ + type ────────┘       │ + texture ~3KB       │
   ├─────────────┤          ├──────────────────────┤
   │ + draw(cvs) │          │ + draw(cvs,x,y,scale)│  ← extrinsic PASSED IN
   └──────▲──────┘          └──────────────────────┘
          │ 1,000,000
   ┌──────┴──────┐
   │   Forest    │   CLIENT
   ├─────────────┤
   │ + trees[]   │
   │ + plant(...)│  ── always goes through the factory, never `new TreeType`
   │ + draw()    │
   └─────────────┘

   ✱ = extrinsic state (unique per Tree)
```

Read it like this: `Forest` (client) calls `TreeTypeFactory.getTreeType(...)`. The factory checks its `Map`. On a miss it constructs and freezes a `TreeType`; on a hit it returns the one it already has. Either way `Forest` gets back a reference, hands it to a new `Tree`, and the `Tree` stores it in `type`. A million `Tree` objects end up holding one of only three references.

**Memory layout — before and after.**

```
BEFORE (naive) — 1,000,000 × ~5 KB ≈ 5.0 GB   → HEAP OOM
┌────────────────────────────────────────────────────────────┐
│ Tree #1   │ x y s │ "Oak" │ bark │ leaf │ MESH 2KB │ TEX 3KB │
├────────────────────────────────────────────────────────────┤
│ Tree #2   │ x y s │ "Oak" │ bark │ leaf │ MESH 2KB │ TEX 3KB │  ← identical
├────────────────────────────────────────────────────────────┤
│ Tree #3   │ x y s │ "Oak" │ bark │ leaf │ MESH 2KB │ TEX 3KB │  ← identical
├────────────────────────────────────────────────────────────┤
│    ...  999,997 more, each carrying its own 5 KB copy  ...  │
└────────────────────────────────────────────────────────────┘
             └──── unique ────┘ └────── DUPLICATED 1e6 TIMES ──────┘


AFTER (flyweight) — 1,000,000 × 32 B + 3 × 5 KB ≈ 32 MB   → fine
┌──────────────────────────┐
│ Tree #1 │ x y s │ ptr ───┼──┐
├──────────────────────────┤  │      SHARED POOL (3 objects, ~15 KB total)
│ Tree #2 │ x y s │ ptr ───┼──┤      ┌──────────────────────────────┐
├──────────────────────────┤  ├─────▶│ TreeType "Oak"   MESH│TEXTURE│ frozen
│ Tree #3 │ x y s │ ptr ───┼──┘      ├──────────────────────────────┤
├──────────────────────────┤         │ TreeType "Pine"  MESH│TEXTURE│ frozen
│  ... 999,997 more, each ─┼────────▶├──────────────────────────────┤
│      just 32 bytes ...   │         │ TreeType "Birch" MESH│TEXTURE│ frozen
└──────────────────────────┘         └──────────────────────────────┘
         └── extrinsic ──┘                    └── intrinsic ──┘
```

The picture to burn in: **the wide, duplicated blocks on the left collapse into three boxes on the right, and a million 8-byte arrows point at them.**

---

## Real world examples

### Java's `Integer.valueOf` and JS string interning

Java caches boxed `Integer` objects for −128…127 and returns the same instance for repeated values — a textbook flyweight in the standard library. V8 does the analogous thing for strings: every string literal in your source is *internalised* into a shared table, so the millionth occurrence of `"level"` in a parsed log file costs one pointer, not one allocation. It's why JSON parsing at scale is survivable. You're already using flyweight; you just didn't write it.

### React elements vs component definitions

When you render `<Button label="Save"/>` 10,000 times in a virtualised list, React does **not** create 10,000 copies of the `Button` function. It creates 10,000 lightweight element descriptors — roughly `{ type: Button, props: { label: 'Save' }, key: null }` — every one of which holds the *same reference* in `type`. The component definition (with its code, its `defaultProps`, its display name) is the flyweight; `props` and `key` are the extrinsic state. This is why React's reconciler can compare `oldElement.type === newElement.type` with a cheap identity check to decide whether to reuse a DOM node.

### Game particle systems and font rasterisers

In a particle system, a single explosion may spawn 50,000 particles. Each particle stores position, velocity, and remaining lifetime (extrinsic, ~40 bytes); all of them reference one shared `ParticleType` holding the sprite texture, blend mode, and colour ramp (intrinsic, potentially hundreds of KB). Font rendering does the same with a **glyph atlas**: a rasterised glyph for a given (character, font, size) is cached once and blitted to many screen positions. Both are conceptually the exact `Tree`/`TreeType` split, and both are cited in the original *Design Patterns* book as the motivating cases.

---

## Trade-offs

| Pros | Cons |
|---|---|
| Massive memory reduction when N is large and distinct types are few (100×+ is routine) | Adds a factory, a cache, and a second class — more moving parts to understand |
| Makes previously impossible object counts feasible (1M trees, 10M particles) | Every access goes through an extra pointer hop — potential CPU cost in hot loops |
| Shared data loaded/parsed once (faster startup for heavy assets) | Intrinsic state **must** be immutable; a single mutation is a global bug |
| Identity comparison (`a.type === b.type`) becomes a cheap, meaningful check | Cache key must include every intrinsic field or you return the wrong flyweight |
| The intrinsic/extrinsic split often clarifies the domain model | Unbounded caches leak; you may need an LRU or `WeakRef` |
| | Harder to read: "where does this tree's colour come from?" is now two hops away |
| | Trades memory for CPU — the opposite trade may be the one you want |

**Rule of thumb:** reach for Flyweight only when all three are true — (1) you have **10,000+** objects, (2) the number of *distinct* heavy configurations is **small** (dozens, not thousands), and (3) a heap snapshot proves the duplicated data is actually a top retainer. If you can't state the memory math in numbers, you're not ready to apply it.

**The sweet spot:** huge, homogeneous, read-only populations — rendering, simulation, parsing, tile maps. Anywhere the phrase "a million of these, but really only a handful of kinds" is true.

---

## Common interview questions on this topic

### Q1: "What is the difference between intrinsic and extrinsic state?"

**Hint:** Intrinsic = context-independent, identical across many objects, immutable, stored *inside* the flyweight and shared (the oak's mesh and bark colour). Extrinsic = context-dependent, unique per object, stored on the client/context object or passed in as a method argument (the tree's x, y, scale). The test: "if I move this tree, does this field change?" If yes, it's extrinsic. Getting this split right is the entire pattern; everything else is plumbing.

### Q2: "Why must a flyweight be immutable? What happens if it isn't?"

**Hint:** Because one object is referenced by potentially millions of contexts. Mutating `oakType.barkColor` changes *every* oak in the forest simultaneously, and the crash site is nowhere near the mutation site — a debugging nightmare. Enforce it with `Object.freeze(this)` in the constructor (in strict mode a write then throws a `TypeError` instead of failing silently). Note that `Object.freeze` is shallow — a `Float32Array` field is still writable element-by-element.

### Q3: "Flyweight vs Singleton vs Object Pool — how are they different?"

**Hint:** Singleton = exactly *one* instance, global access, usually mutable — it's about *access control*. Object Pool = a small set of expensive objects *lent out exclusively* and returned (DB connections) — it's about *creation cost / CPU*. Flyweight = a *few immutable* objects *shared concurrently* by many references — it's about *memory*. A flyweight factory happens to hold a cache, but the intent is space, not time.

### Q4: "You have 10 million particles and 2 GB of RAM. Walk me through it."

**Hint:** Do the math out loud. If each particle naively carries a 5 KB sprite + colour ramp, that's 50 GB — impossible. Split: intrinsic = sprite, blend mode, colour ramp (say 20 distinct particle types); extrinsic = position, velocity, lifetime (~40 bytes each). New total = 10M × 40 B + 20 × 5 KB ≈ 400 MB + 100 KB. Fits. Then mention the next step if 400 MB is still too much: switch from an array-of-objects to typed arrays (structure-of-arrays), which removes the per-object header overhead entirely.

### Q5: "What could go wrong with the flyweight factory's cache?"

**Hint:** Three things. (1) **Unbounded growth** — if keys derive from user input, the `Map` becomes a leak; bound it with an LRU. (2) **Incomplete keys** — if the key omits an intrinsic field, you hand back a flyweight with the wrong data, silently. (3) **Bypassed factory** — if any client calls `new TreeType()` directly, duplicates re-enter and the savings vanish; consider a private constructor pattern or a lint rule.

---

## Practice exercise

### Build a Chess Board with Flyweight Pieces

Write a single file, `chess-flyweight.js`, and produce a working demo.

**Requirements:**

1. **Identify the split first.** Before writing code, write a comment block at the top listing which piece fields are intrinsic and which are extrinsic. (Hint: a piece's *type* and *colour* determine its movement rules and its sprite; its *square* does not.)
2. **`PieceType` (flyweight)** — holds `kind` (`'pawn' | 'rook' | ...`), `color` (`'white' | 'black'`), a `sprite` (fake it with `new Uint8Array(2048)`), and a `movePattern` array. Freeze it with `Object.freeze`.
3. **`PieceTypeFactory`** — a static `Map` cache keyed on `kind|color`. Expose an `instancesCreated` counter.
4. **`Piece` (context)** — holds `row`, `col`, and a reference to its `PieceType`. Give it a `describe()` method that delegates to the flyweight, passing position in.
5. **`Board`** — sets up a standard 32-piece opening position via the factory.
6. **Demo (`main`)**: print the number of `Piece` objects (32) and the number of `PieceType` objects created. It must be **12** (6 kinds × 2 colours), not 32. Assert that both white rooks share the identical `PieceType` object with `===`.
7. **Prove the immutability guard.** In a `try/catch`, attempt `somePiece.type.color = 'green'` under `'use strict'` and print the `TypeError` you catch.
8. **Do the math.** Print a table comparing naive (32 × ~2 KB) vs flyweight (32 × 24 B + 12 × 2 KB). Then answer in a comment: *at what board size / piece count would this actually be worth doing?* (This is the point — for 32 pieces it is **not** worth it, and you should say so. Now imagine a Go board simulation running 10 million positions.)

**What to produce:** one runnable `.js` file whose output shows 32 pieces, 12 piece types, a `true` identity check, a caught `TypeError`, and your memory table.

---

## Quick reference cheat sheet

- **Flyweight** — share one immutable object across many contexts to save memory. Many objects, few *kinds*.
- **Intrinsic state** — shared, immutable, context-independent. Lives *inside* the flyweight. (mesh, texture, bark colour)
- **Extrinsic state** — unique per object, context-dependent. Lives on the context or is *passed in* as an argument. (x, y, scale)
- **The split is the pattern.** Everything else — factory, `Map`, freeze — is mechanics.
- **Four participants** — Flyweight, Flyweight Factory, Context, Client. Clients must NEVER `new` a flyweight directly.
- **The factory returns existing instances.** Cache key must contain *every* intrinsic field, or you'll return the wrong data.
- **Immutability is mandatory.** `Object.freeze(this)` in the constructor. One mutation changes every object that shares it.
- **`Object.freeze` is shallow** — a `Float32Array` field is still writable element-by-element.
- **The math:** 1M × 5 KB ≈ 5 GB (dies) → 1M × 32 B + 3 × 5 KB ≈ 32 MB. **~157× smaller.**
- **Saving ≈ `heavyBytes × (N − distinctTypes)`.** Big N, small distinct count, non-trivial heavy data — or don't bother.
- **It trades CPU for RAM** — extra pointer hop per access, `Map` lookup per creation.
- **Watch the cache** — a strong-reference `Map` keyed on user input is a memory leak wearing a hat.
- **Not a Singleton** (one instance, global access) and **not an Object Pool** (exclusive lending, saves setup cost).
- **In the wild:** JS string interning, `Integer.valueOf`, React component `type` refs, font glyph atlases, particle systems, connection pools.
- **Measure first.** Heap snapshot in Chrome DevTools or `process.memoryUsage().heapUsed`. If duplicated data isn't a top retainer, this is premature optimization.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [39 — Proxy Pattern](./37-pattern-proxy.md) — like Flyweight, it inserts an indirection between client and the real object; Proxy does it to control *access*, Flyweight to control *memory*. |
| **Next** | [41 — Chain of Responsibility](./47-pattern-chain-of-responsibility.md) — moves from structural patterns into behavioural ones; Express middleware is the canonical Node example. |
| **Related** | [30 — Singleton Pattern](./29-pattern-singleton.md) — the #1 confusion point. Singleton = exactly one instance; Flyweight = a few shared instances. Know the difference cold. |
| **Related** | [37 — Composite Pattern](./38-pattern-composite.md) — Flyweight's classic partner. The leaves of a Composite document tree (characters, tiles) are almost always flyweights. |
