# 45 — Template Method Pattern
## Category: LLD Patterns

---

## What is this?

The **Template Method pattern** puts the **skeleton of an algorithm** in a base class, and lets subclasses **fill in the individual steps** — without ever changing the order those steps run in.

Think of a recipe for baking. Every bake follows the same skeleton: *prepare the batter → pour into a tin → bake → cool → decorate.* A chocolate cake and a lemon loaf differ in **what goes into** each step, but never in the **order of the steps**. You never bake before you pour. The recipe's *shape* is fixed; its *contents* vary.

In code: the base class owns a method (the "template method") that calls a sequence of steps. Some steps are implemented once, in the base, and are the same for everyone. Others are left abstract for subclasses to define. The subclass **cannot reorder them** — and that's the feature, not the limitation.

---

## Why does it matter?

Template Method is the pattern you reach for when you have **several processes that are 80% identical and 20% different**, and the 80% keeps getting copy-pasted.

Think about what you actually write in a backend:

- Every data export is: fetch → transform → format → save. Only the *format* differs (CSV vs PDF vs JSON).
- Every API request handler is: authenticate → validate → do the work → serialize → log. Only *do the work* differs.
- Every ETL job is: connect → extract → clean → load → disconnect. Only *extract* and *clean* differ.
- Every test is: setUp → run → tearDown.

Without Template Method, you copy that skeleton into every subclass and someone eventually forgets the `logAudit()` step, or calls `save()` before `format()`, or swallows an error in the `finally`. The invariant parts of the algorithm are the parts you **cannot afford** to leave to a subclass's discretion — so you don't. You lock them in the base.

**Interview angle:** The question is almost never *"what is Template Method"* — it's **"how is Template Method different from Strategy?"** They solve the same family of problem (varying part of a behaviour) with opposite mechanisms (inheritance vs composition). Section 7 is written specifically for that question. Expect the follow-up: *"and which would you actually use?"*

**Real-work angle:** You've used it. Every framework base class you've ever extended — a Jest `beforeEach`/`afterEach` lifecycle, a React class component's `render()` being called *by* React, a Mongoose middleware hook, a Passport `Strategy` subclass implementing `authenticate()` — is Template Method. The framework owns the skeleton; you fill in the holes. This is the **Hollywood Principle**: *"Don't call us, we'll call you."*

---

## The core idea — explained simply

### The Recipe Analogy

Open any baking cookbook to the "Loaf Cakes" chapter. Before the individual recipes, there's a page called **"The Master Method"**:

```
THE MASTER LOAF METHOD          (this never changes)
  1. Preheat oven to 180°C       ← same for every loaf
  2. Grease and line the tin     ← same for every loaf
  3. Make the batter             ← DIFFERENT per recipe
  4. Add the flavouring          ← DIFFERENT per recipe
  5. Pour into tin, bake 45 min  ← same for every loaf
  6. Cool on a rack 20 min       ← same for every loaf
  7. (Optional) Ice or glaze     ← OPTIONAL — many loaves skip this
```

Then the individual recipes are refreshingly short:

> **Lemon Loaf.** *Batter:* creamed butter, sugar, eggs, self-raising flour. *Flavouring:* zest of two lemons. *Glaze:* lemon juice + icing sugar.
>
> **Chocolate Loaf.** *Batter:* melted butter, sugar, eggs, plain flour, cocoa. *Flavouring:* 100g dark chocolate chips. *(No glaze.)*

Notice three things, because they are the entire pattern:

1. **The skeleton is written once, in one place.** No recipe repeats "preheat to 180°C." If the publisher decides fan ovens need 160°C, they change **one line** and every loaf in the book is corrected.
2. **A recipe cannot reorder the steps.** It cannot bake before greasing the tin. The master method decides the order; the recipe only fills the blanks. That's the *guarantee*.
3. **Step 7 is optional.** Chocolate loaf just... doesn't glaze. It doesn't have to say "no glaze" — the default is "do nothing." That optional step with a do-nothing default is called a **hook**.

| Cookbook | Template Method | In our code |
|---|---|---|
| The Master Loaf Method | **Template method** — the fixed algorithm | `DataExporter.export()` |
| "Preheat to 180°C" | **Invariant step** — concrete, in the base, shared | `save()`, `validate()` |
| "Make the batter" (blank) | **Abstract step** — subclass MUST fill it | `fetch()`, `format()` |
| "(Optional) glaze" | **Hook** — optional step, default = no-op | `beforeSave()` |
| Lemon Loaf recipe | **Concrete subclass** | `CsvExporter` |
| Chocolate Loaf recipe | Another **concrete subclass** | `PdfExporter` |
| The publisher fixing the oven temp once | **DRY** — change the skeleton in one place | Edit the base class |
| "You can't bake before greasing" | **The invariant is enforced, not requested** | `export()` is not overridden |

The one-sentence version: **the base class calls the subclass, never the other way round.** The subclass doesn't run the algorithm; the algorithm runs the subclass.

---

## Key concepts inside this topic

### 1. The problem it solves — start with the painful code

You need to export report data as CSV. Then someone asks for PDF. Then JSON. Here's what most codebases end up with:

```javascript
// BAD — the skeleton is copy-pasted into every exporter.
// Read these two classes and count what's identical.
class CsvExporter {
  async export(reportId) {
    console.log(`[audit] export started: ${reportId}`);            // duplicated
    const rows = await db.query('SELECT * FROM report_rows WHERE report_id = $1', [reportId]);
    if (rows.length === 0) throw new Error('Nothing to export');   // duplicated
    const clean = rows.map(r => ({ ...r, amount: Number(r.amount) })); // duplicated
    const header = Object.keys(clean[0]).join(',');
    const body = clean.map(r => Object.values(r).join(',')).join('\n');
    const content = `${header}\n${body}`;
    await fs.writeFile(`/exports/${reportId}.csv`, content);       // duplicated (ish)
    console.log(`[audit] export finished: ${reportId}`);           // duplicated
    return `/exports/${reportId}.csv`;
  }
}

class PdfExporter {
  async export(reportId) {
    console.log(`[audit] export started: ${reportId}`);            // ...again
    const rows = await db.query('SELECT * FROM report_rows WHERE report_id = $1', [reportId]);
    const clean = rows.map(r => ({ ...r, amount: Number(r.amount) })); // ...again
    // ...and here the new dev forgot the empty check. Silent bug: a 0-row PDF ships.
    const content = renderPdf(clean);
    await fs.writeFile(`/exports/${reportId}.pdf`, content);
    // ...and forgot the audit log entirely. Compliance finds out in six months.
    return `/exports/${reportId}.pdf`;
  }
}
```

Look at what actually differs between them: **how you format the rows.** That's it. Everything else — the audit log, the query, the empty check, the numeric cleanup, the write — is identical and was **retyped from memory**, which is why the PDF one is missing two of them.

This is the failure mode Template Method exists to prevent. Not "duplicated code is ugly" — **duplicated code drifts.** Copies diverge silently, and the divergence is always a bug.

```javascript
// GOOD — the skeleton lives in ONE place and cannot be skipped.
class DataExporter {
  async export(reportId) {           // ← the TEMPLATE METHOD. Never overridden.
    this.#audit('started', reportId);
    const raw = await this.fetch(reportId);       // abstract-ish: subclass may override
    this.validate(raw);                            // invariant: always runs
    const rows = this.transform(raw);              // hook: default = identity
    const content = this.format(rows);             // abstract: subclass MUST implement
    const path = await this.save(reportId, content); // invariant
    this.#audit('finished', reportId);
    return path;
  }
  // ... steps below
}

class PdfExporter extends DataExporter {
  format(rows) { return renderPdf(rows); }   // the ONLY thing a PDF export differs by
  get extension() { return 'pdf'; }
}
```

The new developer writing `PdfExporter` **cannot** forget the audit log, because they never see it. It isn't their job. That's the point.

### 2. The structure — participants

```
AbstractClass
  + templateMethod()   ← final: defines the ORDER. Do not override.
  + step1()            ← concrete: invariant, shared by all
  # step2()            ← abstract: subclass MUST implement
  # step3()            ← hook: has a default (often a no-op), MAY be overridden
        △
        │ extends
  ConcreteClass
  + step2()            ← fills in the blank
```

| Participant | Job | Rule |
|---|---|---|
| **AbstractClass** | Owns the template method and the invariant steps | The template method is **final** — never override it |
| **Template method** | Calls the steps in a fixed order | Should be the *only* public entry point |
| **Abstract step** | A required blank | Base implementation throws — fail loudly, not silently |
| **Concrete step** | An invariant | Lives in the base, shared, never overridden |
| **Hook** | An optional extension point | Default implementation is a no-op or identity |
| **ConcreteClass** | Fills the blanks | Overrides *only* steps, never the template method |

### 3. The three kinds of step (this is the whole design decision)

Every method a base class exposes is one of three things, and choosing wrong is where Template Method goes bad.

| Kind | Base implementation | Subclass | Use for |
|---|---|---|---|
| **Invariant** | Real code | Must NOT override | Audit logging, validation, transactions, the algorithm's order |
| **Abstract** | `throw new Error('not implemented')` | MUST override | The genuinely varying part (`format()`) |
| **Hook** | No-op or sensible default | MAY override | Optional customisation (`beforeSave()`, `shouldCompress()`) |

**The design rule:** be *stingy* with what you expose. Every method a subclass can override is a promise you now have to keep forever, and a place the invariant can be broken. Start everything private, and open up exactly the steps that must vary.

**Why hooks matter:** without them, every subclass is forced to implement steps it doesn't care about (`beforeSave() {}` — empty, in ten files). With a hook, the base provides the boring default and the one subclass that needs something special overrides it.

### 4. Full JavaScript implementation

```javascript
// ─────────────────────────────────────────────────────────────
// ABSTRACT CLASS — owns the algorithm's SKELETON
// ─────────────────────────────────────────────────────────────
class DataExporter {
  constructor(dataSource, fileSystem) {
    // JS has no `abstract` keyword, so we enforce it manually.
    if (new.target === DataExporter) {
      throw new Error('DataExporter is abstract — extend it.');
    }
    this.db = dataSource;
    this.fs = fileSystem;
  }

  // ══ THE TEMPLATE METHOD ══════════════════════════════════
  // This is the invariant. The order NEVER changes, and no subclass
  // may override it. Everything a subclass can customise is a call below.
  async export(reportId) {
    const startedAt = Date.now();
    this.#audit('started', reportId);

    try {
      const raw       = await this.fetch(reportId);        // step 1 — overridable
      this.validate(raw);                                  // step 2 — INVARIANT
      const rows      = this.transform(raw);               // step 3 — hook (default: passthrough)
      const content   = this.format(rows);                 // step 4 — ABSTRACT (must implement)

      await this.beforeSave(reportId, content);            // step 5 — HOOK (default: no-op)
      const path      = await this.save(reportId, content);// step 6 — INVARIANT

      this.#audit('finished', reportId, Date.now() - startedAt);
      return path;
    } catch (err) {
      // Every exporter gets identical failure handling. For free. Always.
      this.#audit('failed', reportId, Date.now() - startedAt, err.message);
      throw err;
    }
  }

  // ── INVARIANT STEPS — same for everyone, subclasses must not touch ──
  validate(raw) {
    if (!Array.isArray(raw)) throw new Error('fetch() must return an array');
    if (raw.length === 0) throw new Error(`Report has no rows — refusing to export`);
    if (raw.length > 1_000_000) throw new Error('Report too large — use the streaming exporter');
  }

  async save(reportId, content) {
    const path = `/exports/${reportId}.${this.extension}`;
    await this.fs.writeFile(path, content);
    return path;
  }

  // Private: not even reachable from a subclass. The strongest guarantee JS offers.
  #audit(event, reportId, ms = 0, error = '') {
    const line = `[audit] ${event} report=${reportId} type=${this.constructor.name}` +
                 (ms ? ` ms=${ms}` : '') + (error ? ` error="${error}"` : '');
    console.log(line);
  }

  // ── DEFAULT / OVERRIDABLE STEPS ──────────────────────────
  // A sensible default. Most exporters read the same table; a subclass CAN change it.
  async fetch(reportId) {
    return this.db.query('SELECT * FROM report_rows WHERE report_id = ?', [reportId]);
  }

  // HOOK — default is identity. Override to reshape rows before formatting.
  transform(rows) { return rows; }

  // HOOK — default is a no-op. Override for compression, virus scan, notifications...
  async beforeSave(_reportId, _content) { /* intentionally empty */ }

  // ── ABSTRACT STEPS — subclass MUST implement ─────────────
  format(_rows) {
    throw new Error(`${this.constructor.name} must implement format()`);
  }
  get extension() {
    throw new Error(`${this.constructor.name} must implement get extension()`);
  }
}

// ─────────────────────────────────────────────────────────────
// CONCRETE SUBCLASSES — each fills in ONLY what differs
// ─────────────────────────────────────────────────────────────
class CsvExporter extends DataExporter {
  get extension() { return 'csv'; }

  format(rows) {
    const headers = Object.keys(rows[0]);
    const escape = (v) => {
      const s = String(v ?? '');
      // Only quote when we have to — that's the CSV spec, and it keeps files small.
      return /[",\n]/.test(s) ? `"${s.replace(/"/g, '""')}"` : s;
    };
    const body = rows.map(r => headers.map(h => escape(r[h])).join(','));
    return [headers.join(','), ...body].join('\n');
  }
}

class JsonExporter extends DataExporter {
  get extension() { return 'json'; }
  format(rows) { return JSON.stringify(rows, null, 2); }
}

class PdfExporter extends DataExporter {
  get extension() { return 'pdf'; }

  // This exporter cares about presentation, so it overrides the TRANSFORM hook.
  transform(rows) {
    return rows.map(r => ({
      ...r,
      amount: typeof r.amount === 'number'
        ? `$${r.amount.toFixed(2)}`      // human-readable money for a printed doc
        : r.amount,
    }));
  }

  format(rows) {
    // Stand-in for a real pdfkit/puppeteer render — the shape is what matters.
    const headers = Object.keys(rows[0]);
    const widths = headers.map(h =>
      Math.max(h.length, ...rows.map(r => String(r[h] ?? '').length)));
    const line = (cells) => '| ' + cells.map((c, i) => String(c ?? '').padEnd(widths[i])).join(' | ') + ' |';
    const rule = '+-' + widths.map(w => '-'.repeat(w)).join('-+-') + '-+';
    return ['%PDF-1.4 (simulated)', rule, line(headers), rule,
            ...rows.map(r => line(headers.map(h => r[h]))), rule].join('\n');
  }

  // This exporter also uses the BEFORE-SAVE hook. The other two don't — and
  // they don't have to write an empty method to say so. That is why hooks exist.
  async beforeSave(reportId, content) {
    console.log(`[pdf] compressing ${content.length} bytes for ${reportId}...`);
  }
}

// ─────────────────────────────────────────────────────────────
// DEMO
// ─────────────────────────────────────────────────────────────
const fakeDb = {
  async query(_sql, [reportId]) {
    if (reportId === 'empty') return [];
    return [
      { id: 1, customer: 'Acme, Inc.', amount: 1250.5, status: 'PAID' },
      { id: 2, customer: 'Globex',     amount: 99,     status: 'PENDING' },
      { id: 3, customer: 'Initech',    amount: 4300.75, status: 'PAID' },
    ];
  },
};
const fakeFs = { async writeFile(path, content) { console.log(`--- wrote ${path} ---\n${content}\n`); } };

async function main() {
  for (const Exporter of [CsvExporter, JsonExporter, PdfExporter]) {
    const exporter = new Exporter(fakeDb, fakeFs);
    await exporter.export('rep-2024-Q3');
  }

  // The invariant proves itself: the empty-report guard fires for EVERY exporter,
  // and no subclass author had to remember it.
  try {
    await new CsvExporter(fakeDb, fakeFs).export('empty');
  } catch (err) {
    console.log(`caught as expected: ${err.message}`);
  }

  // The abstract guard fires too — a half-written subclass fails loudly, at once.
  class XmlExporter extends DataExporter { get extension() { return 'xml'; } }
  try {
    await new XmlExporter(fakeDb, fakeFs).export('rep-2024-Q3');
  } catch (err) {
    console.log(`caught as expected: ${err.message}`);  // "XmlExporter must implement format()"
  }
}

main();
```

Output (abridged):

```
[audit] started report=rep-2024-Q3 type=CsvExporter
--- wrote /exports/rep-2024-Q3.csv ---
id,customer,amount,status
1,"Acme, Inc.",1250.5,PAID
...
[audit] finished report=rep-2024-Q3 type=CsvExporter ms=1
[audit] started report=rep-2024-Q3 type=PdfExporter
[pdf] compressing 214 bytes for rep-2024-Q3...
--- wrote /exports/rep-2024-Q3.pdf ---
...
[audit] failed report=empty type=CsvExporter error="Report has no rows — refusing to export"
caught as expected: Report has no rows — refusing to export
caught as expected: XmlExporter must implement format()
```

**Count the lines in each subclass.** `JsonExporter` is *four lines*. It gets audit logging, validation, error handling, timing, and file writing for free — and it could not opt out of them if it tried. **That is the invariant staying in the base.** That's the whole pattern.

### 5. Where the hook earns its keep

Watch what happens when the product asks: *"PDFs must be compressed before upload, but CSVs must not."*

- **Without a hook:** you add an `if (this instanceof PdfExporter)` to the base class. Now the base knows about its subclasses. That's a circular dependency, and it will grow an `else if` for every future format.
- **With a hook:** `PdfExporter` overrides `beforeSave()`. **The base class does not change.** `CsvExporter` and `JsonExporter` do not change. You added a feature by adding code, not by editing code — the **Open/Closed Principle**, achieved.

The tell that you need a hook: *you're about to write `if (subclassType === ...)` in a base class.* Stop and add an overridable no-op instead.

### 6. A real Node.js ecosystem example

**Passport.js** is textbook Template Method. You extend `passport-strategy`'s base `Strategy` and implement exactly one method, `authenticate(req, options)`. Passport owns the whole skeleton — sessions, the request pipeline, `success()`/`fail()`/`error()` outcomes, redirect handling. Your subclass fills in *"how do I check this particular credential."* You never call Passport; Passport calls you.

**Jest / Mocha lifecycle:** `beforeAll → beforeEach → test → afterEach → afterAll`. That order is fixed by the framework. `beforeEach` is a **hook** with a default of "do nothing." Your test file fills in the blanks. You cannot make `afterEach` run before the test — and you'd be furious if you could, because then test isolation would be optional.

**Mongoose middleware:** `schema.pre('save', fn)` and `post('save', fn)` are hooks around Mongoose's fixed save skeleton (validate → pre hooks → write → post hooks). Mongoose guarantees validation runs; you customise around it.

**Express's `res.json()`:** internally always sets the content type, serializes, sets `Content-Length`, and sends — a fixed sequence with a customisable `app.set('json replacer')` hook.

### 7. Template Method vs Strategy — the interview question

They both let you vary part of a behaviour. They are **not** interchangeable, and knowing exactly why separates a mid-level answer from a senior one.

```
TEMPLATE METHOD (inheritance)         STRATEGY (composition)
                                      
    ┌──────────────────┐              ┌──────────────────┐      ┌──────────────────┐
    │  DataExporter    │              │  DataExporter    │──────▶│ «interface»      │
    │  + export()  ◀── skeleton       │  - formatter ────┼──────▶│   Formatter      │
    │  # format() ◀─── blank          │  + export()      │ HAS-A │  + format(rows)  │
    └────────▲─────────┘              └──────────────────┘      └────────▲─────────┘
             │ IS-A (fixed at                                            │
             │ class-definition time)                     ┌──────────────┼────────────┐
    ┌────────┴────────┐                              ┌────┴─────┐  ┌─────┴────┐  ┌────┴─────┐
    │  CsvExporter    │                              │CsvFormat │  │PdfFormat │  │JsonFormat│
    │  + format()     │                              └──────────┘  └──────────┘  └──────────┘
    └─────────────────┘
                                      exporter.formatter = new PdfFormatter();  ← swap at RUNTIME
```

| | **Template Method** | **Strategy** |
|---|---|---|
| **Mechanism** | **Inheritance** (IS-A) | **Composition** (HAS-A) |
| **What varies** | *Steps within* an algorithm | The *whole* algorithm |
| **Who owns the skeleton** | The base class — locked | The context — or there isn't one |
| **When is the choice made** | At **class definition time** (compile time, conceptually) — you pick a subclass | At **runtime** — swap the object in a field |
| **Can you change it mid-flight?** | No. A `CsvExporter` is a `CsvExporter` forever. | Yes. `exporter.setFormatter(new PdfFormatter())` |
| **Reuse of the varying part** | Locked inside its subclass — can't reuse `CsvExporter.format` elsewhere | `CsvFormatter` is a free-standing object; reuse it anywhere |
| **Combinations** | 3 fetchers × 3 formatters = **9 subclasses** (explosion) | 3 fetchers + 3 formatters = **6 objects**, mix freely |
| **Testing** | Must instantiate the subclass and run the whole skeleton | Test `CsvFormatter.format()` in complete isolation. Much nicer. |
| **Coupling** | Tight — subclass sees all protected base state | Loose — strategy only sees what you pass it |
| **The guarantee** | **Strong**: the order and the invariant steps *cannot* be bypassed | **Weak**: the context has to choose to enforce anything |

**The one-liner to say in an interview:**
> *"Template Method varies the steps **inside** a fixed algorithm using inheritance, and the algorithm's skeleton is locked at class-definition time. Strategy varies **the whole algorithm** using composition, and it can be swapped at runtime. Template Method gives you a stronger guarantee — subclasses literally can't reorder the steps — at the cost of rigidity, tight coupling, and a subclass explosion if more than one thing varies."*

**And the honest follow-up:** *"which would you actually use?"* — Most teams reach for Strategy, for the reasons in Section 8. But when the invariant genuinely **must** be enforced — an audit log for compliance, a transaction boundary, a security check — Template Method's inability to be bypassed is exactly the property you want. **You use inheritance when you want to take a choice away from the subclass author.**

### 8. When NOT to use it — the LSP risk, and why teams prefer composition

**a) The Liskov Substitution Principle risk.**
LSP (from SOLID) says: *anywhere the base class works, a subclass must work too, without surprises.* Template Method makes this easy to break, because subclasses can quietly violate what the base assumes:

```javascript
// LSP VIOLATION — the base promised format() returns a string...
class BrokenExporter extends DataExporter {
  get extension() { return 'bin'; }
  format(rows) { return Buffer.from(JSON.stringify(rows)); }  // ...returns a Buffer.
  // save() now writes something the caller of export() doesn't expect.
  // Nothing crashed. Nothing warned. The contract just... broke.
}

// WORSE — a subclass that "helpfully" skips a step
class SneakyExporter extends DataExporter {
  validate(_raw) { /* our data is always fine, skip it */ }   // deleted the invariant!
}
```
`SneakyExporter` is the nightmare: **a subclass reached into the base and deleted a guarantee.** The base class's promise ("we never export an empty report") is now a lie, and there is no compiler to stop it. This is why invariant steps should be **private (`#`)** or at minimum documented as final — the fewer methods you expose, the fewer promises can be broken.

**b) The fragile base class problem.** Every subclass depends on the base's internals. Change a step's signature, its return type, or the order of calls, and you break every subclass in the codebase — possibly ones you don't own. Inheritance is the tightest coupling in OOP. It is a **permanent** commitment.

**c) The subclass explosion.** Template Method handles *one* axis of variation gracefully. The moment a second axis appears — "we also need to export to S3 as well as disk" — you face `CsvDiskExporter`, `CsvS3Exporter`, `PdfDiskExporter`, `PdfS3Exporter`... 2 formats × 2 destinations = 4 classes; add JSON and FTP and you have 9. Composition would need 3 + 3 = **6 objects, freely combined**. (The pattern that fixes this is **Bridge**, topic 39.)

**d) No runtime swapping.** A user toggling "export as PDF instead" means constructing a whole new object. With Strategy, it's one assignment.

**e) Testing pain.** To test `CsvExporter.format()`, you must construct an exporter with a database and a filesystem, then run the full skeleton. With Strategy, `new CsvFormatter().format(rows)` is a pure function call with no dependencies. This alone converts a lot of teams.

**Why many teams prefer composition anyway.** The classic advice — *"favour composition over inheritance"* (from the Gang of Four themselves) — exists because inheritance is a compile-time, permanent, transitive relationship, while composition is a runtime, swappable, local one. If you're unsure, **start with Strategy.** Turning composition into inheritance later is easy; unwinding an inheritance hierarchy that four teams depend on is a quarter of work.

**Use Template Method when all of these are true:**
1. The algorithm's **order truly must not vary** (and enforcing it matters — compliance, transactions, security).
2. There is **exactly one axis** of variation.
3. The subclasses are **owned by you or your team** (a public framework base class is a forever-API).
4. The invariant part is **substantial** — if the base class only holds two lines, the ceremony isn't worth it.

### 9. Related patterns and how they differ

| Pattern | Relationship |
|---|---|
| **Strategy** (42) | The composition-based alternative. Same problem, opposite mechanism. See the table in section 7 — this is *the* interview comparison. |
| **Factory Method** (30) | Is a *specialisation* of Template Method: the "step" the subclass fills in happens to be "create an object." |
| **Bridge** (39) | The escape hatch when you have two axes of variation and Template Method is exploding into N×M subclasses. |
| **Decorator** (35) | Also extends behaviour — but *wraps* at runtime and can be stacked, rather than fixing a skeleton at definition time. |
| **Chain of Responsibility** (47) | Express middleware also composes a "pipeline of steps" — but the order is assembled at runtime by the app, not fixed by a base class. Use CoR when the pipeline itself should be configurable. |
| **Hollywood Principle** | Not a pattern, a principle, and the soul of this one: *"Don't call us, we'll call you."* Inversion of control — the framework calls your code. |

---

## Visual / Diagram description

### Diagram 1 — Class diagram

```
              ┌──────────────────────────────────────────────────┐
              │              «abstract»                          │
              │              DataExporter                        │
              │──────────────────────────────────────────────────│
              │ - db, fs                                         │
              │──────────────────────────────────────────────────│
              │ + export(reportId)     ◀── TEMPLATE METHOD       │
              │       calls, in this exact order:                │
              │         1. #audit('started')     [invariant]     │
              │         2. fetch()               [default]       │
              │         3. validate()            [INVARIANT]     │
              │         4. transform()           [HOOK]          │
              │         5. format()              [ABSTRACT]      │
              │         6. beforeSave()          [HOOK]          │
              │         7. save()                [INVARIANT]     │
              │         8. #audit('finished')    [invariant]     │
              │──────────────────────────────────────────────────│
              │ + validate(raw)     { real code — do not override}│
              │ + save(id, content) { real code — do not override}│
              │ - #audit(...)       { private — CANNOT override } │
              │ + fetch(id)         { sensible default }          │
              │ + transform(rows)   { hook: returns rows as-is }  │
              │ + beforeSave(...)   { hook: no-op }               │
              │ + format(rows)      { THROWS — must implement }   │
              │ + get extension()   { THROWS — must implement }   │
              └───────────────────────▲──────────────────────────┘
                                      │ extends
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
┌─────────┴──────────┐   ┌────────────┴───────┐   ┌───────────────┴──────┐
│   CsvExporter      │   │   JsonExporter     │   │    PdfExporter       │
│────────────────────│   │────────────────────│   │──────────────────────│
│ + format(rows)     │   │ + format(rows)     │   │ + format(rows)       │
│ + get extension()  │   │ + get extension()  │   │ + get extension()    │
│   → 'csv'          │   │   → 'json'         │   │   → 'pdf'            │
│                    │   │                    │   │ + transform(rows)  ◀─┼── overrides
│ (4 lines total)    │   │ (2 lines total)    │   │ + beforeSave(...)  ◀─┼── the HOOKS
└────────────────────┘   └────────────────────┘   └──────────────────────┘
```

The thing to notice: **`export()` appears exactly once**, in the base. No subclass has it. That single fact is why the audit log can never be forgotten.

### Diagram 2 — Sequence: `new PdfExporter().export('rep-1')`

```
 Client        PdfExporter          DataExporter (base)          DB / FS
   │                │                        │                       │
   │ export('rep-1')│                        │                       │
   ├───────────────────────────────────────▶ │  (inherited method)   │
   │                │                        │                       │
   │                │        #audit('started') ──▶ console           │
   │                │                        │                       │
   │                │◀── this.fetch('rep-1') ┤   [base default]      │
   │                │                        ├──── SELECT * ───────▶ │
   │                │                        │◀─── rows ─────────────┤
   │                │                        │                       │
   │                │        this.validate(raw)  [INVARIANT — always runs,
   │                │                        │    throws if 0 rows]  │
   │                │                        │                       │
   │                │◀── this.transform(rows)┤   ◀── PDF's OVERRIDE fires
   │                │  format money as $1,250.50                     │
   │                ├───── formatted rows ──▶│                       │
   │                │                        │                       │
   │                │◀── this.format(rows) ──┤   ◀── PDF's OVERRIDE fires
   │                ├───── pdf string ──────▶│                       │
   │                │                        │                       │
   │                │◀── this.beforeSave() ──┤   ◀── PDF's HOOK fires
   │                │   "compressing..."     │       (Csv/Json: no-op)
   │                │                        │                       │
   │                │        this.save(...)  │   [INVARIANT]         │
   │                │                        ├──── writeFile ──────▶ │
   │                │                        │                       │
   │                │        #audit('finished') ──▶ console          │
   │◀── '/exports/rep-1.pdf' ─────────────────┤                      │
```

Read the arrows carefully. **The base class calls *down* into the subclass** at steps 4, 5, and 6 — that's the inversion of control. The client called `export()` on a `PdfExporter`, but the method that ran is the *base's*, and it reached back down into PDF's overrides at exactly the three points it chose to.

---

## Real world examples

### 1. Passport.js (Node authentication)

Every Passport strategy — `passport-local`, `passport-jwt`, `passport-google-oauth20`, and the hundreds of community ones — extends a base `Strategy` class and implements exactly **one** method: `authenticate(req, options)`. Passport owns everything else: hooking into the Express request pipeline, session serialization, and the `this.success(user)` / `this.fail()` / `this.error(err)` outcome protocol. A strategy author cannot skip session handling or invent a fourth outcome. The framework has taken those choices away — deliberately — and that's why 500+ third-party strategies interoperate.

### 2. Jest and Mocha test lifecycles

`beforeAll → beforeEach → test → afterEach → afterAll`, in that order, every time. You cannot reorder them, and you'd be horrified if you could — test isolation depends on `afterEach` running *after* the test, always, even when the test throws. The `before`/`after` functions are **hooks**: they default to no-ops, and most tests never define them. Your test file is the "subclass"; the runner owns the skeleton.

### 3. React class components (and every UI framework's render loop)

The lifecycle `constructor → render → componentDidMount → (setState) → render → componentDidUpdate → componentWillUnmount` is a template method owned by React's reconciler. `render()` is the **abstract step** — you must implement it. `componentDidMount()` is a **hook** — optional, defaults to nothing. You never call `render()` yourself; React calls it, at the moment React chooses. Pure Hollywood Principle. (React's move to hooks-and-functions is, not coincidentally, a move *away* from inheritance and *toward* composition — the same industry drift described in section 8.)

---

## Trade-offs

| Pros | What it buys you |
|---|---|
| Eliminates duplication | The 80% that's identical is written once |
| **Enforces** the invariant | A subclass literally cannot skip the audit log or reorder steps |
| Subclasses are tiny | `JsonExporter` is 4 lines and correct |
| One place to fix | Change the skeleton once; every subclass is corrected |
| Open/Closed via hooks | New behaviour by adding a class, not editing the base |
| Clear extension points | The abstract methods *document* what's meant to vary |

| Cons | The cost you pay |
|---|---|
| Inheritance = tight coupling | The most rigid relationship in OOP, and permanent |
| Fragile base class | Changing the base can break every subclass, including ones you don't own |
| No runtime swapping | The algorithm is fixed once the object is constructed |
| Subclass explosion | Two axes of variation → N×M classes |
| LSP violations are easy | A subclass can quietly break a base-class promise, with no compiler to catch it |
| Hard to unit-test a step | You must build the whole object and run the skeleton |
| Overridable ≠ safe | Every exposed method is a forever-API you have to support |

**The sweet spot:** Use Template Method when the **order of steps is a guarantee you need to enforce** — a compliance audit log, a transaction boundary, an auth check — and there's exactly **one** axis of variation. Keep the invariant steps **private (`#`)** so they physically cannot be overridden, expose the absolute minimum, and give every optional step a no-op default.

**Rule of thumb:** *If you'd be comfortable letting a subclass author skip the step, it's a hook. If you'd fire them for skipping it, it belongs in the template method — and it should be private.* And if a second axis of variation shows up, stop and refactor to Strategy or Bridge before the class count doubles.

---

## Common interview questions on this topic

### Q1: "What's the difference between Template Method and Strategy?"
**Hint:** *The* question. Template Method = **inheritance**, varies the **steps inside** a fixed algorithm, skeleton locked at class-definition time, subclass can't reorder anything. Strategy = **composition**, varies **the whole algorithm**, swappable at runtime with one assignment. Template Method gives a stronger *guarantee* (the invariant can't be bypassed); Strategy gives more *flexibility* (runtime swap, isolated unit tests, no subclass explosion with N×M combinations). Close with: "most teams default to Strategy, but I'd reach for Template Method when the step order is a compliance or safety guarantee I need to *enforce*, not merely request."

### Q2: "What is a hook method and why do you need one?"
**Hint:** An **optional** step in the skeleton with a default implementation — usually a no-op or an identity function. Subclasses override it only if they care. Two reasons it's essential: (1) without it, every subclass must implement every step, including ones it doesn't need — ten files with `beforeSave() {}` in them; (2) it's how you honour Open/Closed. The tell that you need one: *you're about to write `if (this instanceof PdfExporter)` in the base class.* Add a hook instead, and the base never changes again.

### Q3: "How does Template Method violate the Liskov Substitution Principle, and how do you defend against it?"
**Hint:** LSP says a subclass must be usable anywhere the base is, with no surprises. But nothing stops a subclass from overriding `validate()` with an empty body — silently deleting a guarantee the base *promised* to callers. Defences: (a) make invariant steps **private (`#`)** so they're physically unoverridable; (b) make the template method itself unoverridable and the only public entry point; (c) expose the *minimum* number of overridable steps — every one is a promise you can no longer keep; (d) `throw` in unimplemented abstract steps so a half-written subclass fails loudly and immediately, rather than doing nothing quietly.

### Q4: "You have 3 export formats and 3 storage destinations. Design it."
**Hint:** **Recognise the trap.** Two axes of variation → pure Template Method gives you 3 × 3 = **9 subclasses** (`CsvS3Exporter`, `PdfDiskExporter`, ...), and adding a 4th format means 3 more. The fix is composition on the second axis: keep the skeleton in the base, but **inject a `Storage` strategy** (`new CsvExporter(new S3Storage())`). Now it's 3 + 3 = **6 objects**, combined freely. Name it: that hybrid is the **Bridge** pattern (topic 39). Saying "Template Method for the steps, Strategy for the destination" is the senior answer.

### Q5: "Name a Template Method you've used without realising it."
**Hint:** Pick from: Jest's `beforeEach`/`afterEach` lifecycle; React's `render()` being called by React; Passport's `Strategy.authenticate()`; Mongoose's `pre('save')` hooks; Express's `res.json()`. In every case the framework owns the skeleton and calls *your* code at fixed points — the **Hollywood Principle**: *"Don't call us, we'll call you."* Say that phrase; it's the concept's name and interviewers listen for it.

---

## Practice exercise

### Build a Payment Processing Pipeline — Then Refactor It to Strategy

Write `payments.js`, runnable with `node payments.js`. Two parts, and **part 2 is the actual lesson.**

**Part 1 — Template Method (~20 min).**

Build an abstract `PaymentProcessor` whose template method `process(order)` runs, in this exact order:

1. `validate(order)` — **INVARIANT.** Amount > 0, currency is one of `USD`/`EUR`/`INR`, `order.id` is present. Throws otherwise. Lives in the base.
2. `authenticate()` — **abstract.** Each provider authenticates differently (API key, OAuth token, HMAC signature).
3. `applyFees(order)` — **hook**, default returns the amount unchanged.
4. `charge(order, amount)` — **abstract.** The actual provider call.
5. `recordTransaction(order, result)` — **INVARIANT**, and make it **private (`#`)**. It must be impossible for a subclass to skip. Log `{ orderId, provider, amount, status, at }`.
6. Wrap 2–5 in `try/catch`: on failure, still record the transaction as `FAILED`, then rethrow.

Now write three processors:
- `StripeProcessor` — fee: **2.9% + $0.30** (overrides the hook)
- `PayPalProcessor` — fee: **3.4% + $0.49** (overrides the hook)
- `BankTransferProcessor` — **no fee at all** (does *not* override the hook — proving why the default exists)

Demo it: process a $100 order through all three, print the fee each charged, then process an invalid order (amount = -5) and show that **every** processor rejects it identically without any of them implementing the check.

**Part 2 — Break it, then fix it (~20 min). This is the part that teaches.**

1. The business now needs each processor to support **two destinations**: charge a card *or* charge a stored wallet balance. Write out (don't implement — just list) the class names a pure Template Method solution would require. You should get **6**. Now add a fourth provider. Count again. **Write the number down.**
2. Now refactor: keep the skeleton in the base, but **inject** the fee calculation as a strategy object (`new StripeProcessor(new PercentPlusFlatFee(0.029, 0.30))`). Prove you can now swap a processor's fee schedule **at runtime** with one assignment — something the inheritance version physically cannot do.
3. Write a **unit test for the fee calculation alone** — no database, no processor, no order. `new PercentPlusFlatFee(0.029, 0.30).calculate(100) === 103.20`. Note how impossible that was in Part 1, where testing the fee meant constructing a whole processor and running the entire pipeline.

**Produce:** the file, the console output, and a **three-sentence written answer** to: *"Which parts of this design should stay Template Method, and which should be Strategy — and why?"* (The answer you're looking for: `validate` and `recordTransaction` stay in the base because they are **guarantees you must enforce**; fees and charging become strategies because they're **variations you want to swap and test independently**.)

---

## Quick reference cheat sheet

- **One line:** The base class defines the algorithm's **skeleton**; subclasses fill in the **steps** — but never the order.
- **Template method:** the method that calls the steps in a fixed sequence. **Never override it.** Make it the only public entry point.
- **Three kinds of step:** **invariant** (base owns it, don't override), **abstract** (subclass must implement — base throws), **hook** (optional, base default is a no-op).
- **Hooks exist so subclasses aren't forced to implement steps they don't care about** — and so you never write `if (this instanceof X)` in a base class.
- **JS has no `abstract`:** enforce it with `if (new.target === Base) throw` in the constructor, and `throw new Error('must implement')` in unimplemented steps.
- **Make invariant steps private (`#`)** — that's the only way JS makes them truly unoverridable.
- **The Hollywood Principle:** *"Don't call us, we'll call you."* The base calls the subclass. That's inversion of control.
- **vs Strategy — the interview answer:** Template Method = **inheritance**, varies *steps inside* an algorithm, locked at class-definition time. Strategy = **composition**, varies the *whole* algorithm, swappable at runtime.
- **Template Method's superpower:** the invariant **cannot be skipped**. Use it when skipping a step is a bug you can't tolerate (audit logs, transactions, auth).
- **Template Method's weakness:** two axes of variation → **N×M subclass explosion.** Refactor to Strategy or Bridge before it happens.
- **The LSP risk:** a subclass can silently override an invariant with an empty body and delete a guarantee. Private steps and a minimal overridable surface are your only defences.
- **Testing:** a Strategy object can be unit-tested alone; a Template Method step usually can't. This alone is why many teams prefer composition.
- **Default to Strategy when unsure.** Composition → inheritance is an easy refactor. Inheritance → composition, after four teams have subclassed you, is not.
- **You already use it:** Jest lifecycles, React's `render()`, Passport strategies, Mongoose `pre('save')` hooks.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [44 — Iterator Pattern](./44-pattern-iterator.md) — traverse a collection without exposing its internals |
| **Next** | [46 — State Pattern](./46-pattern-state.md) — an object changes its behaviour when its internal state changes |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) — the composition-based alternative; *the* interview comparison for this topic |
| **Related** | [39 — Bridge Pattern](./39-pattern-bridge.md) — the fix when two axes of variation cause a subclass explosion |
| **Related** | [30 — Factory Method Pattern](./30-pattern-factory-method.md) — a Template Method whose varying step happens to be "create an object" |
