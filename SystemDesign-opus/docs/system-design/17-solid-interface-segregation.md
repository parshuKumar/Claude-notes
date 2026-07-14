# 17 — SOLID — Interface Segregation Principle (ISP)

## Category: LLD Fundamentals

---

## What is this?

The Interface Segregation Principle says: **no client should be forced to depend on methods it does not use.** If a class only needs `read()`, don't make it swallow an interface that also demands `write()`, `delete()`, `archive()`, and `replicate()`.

Think of a **restaurant menu**. A vegetarian doesn't want a 200-item menu where 180 items are meat — and more importantly, they *definitely* don't want a menu where ordering "the vegetarian set" obliges them to also order a steak they'll throw away. ISP says: split the giant menu into a veg menu, a dessert menu, a drinks menu. Each customer reads only what applies to them.

In code: **many small, focused contracts beat one giant one.**

---

## Why does it matter?

A fat interface is a **lie about what a class needs.** And that lie costs you three ways:

1. **Empty stubs everywhere.** Classes implement methods just to satisfy the contract — with `throw new Error('not supported')` or a silent `{}`. That is instantly an LSP violation too. (Recall from [16 — LSP](./16-solid-liskov-substitution.md) that a new exception in an override breaks substitutability.)

2. **False coupling.** Your `ReportGenerator` only ever calls `find()`. But because it depends on the fat `Repository`, a change to `Repository.bulkImport()` triggers a recompile, a redeploy, a re-review, and a re-test of `ReportGenerator`. It depends on code it doesn't use.

3. **Painful tests.** To test one method, you must construct a mock with 14 methods on it. Everyone's seen the 60-line mock object for a 3-line test.

**Interview angle:** ISP is the SOLID letter people give the shallowest answer to ("uh, small interfaces"). Show the *consequence* — that a fat interface forces implementers to lie, and lying implementers break LSP — and you're immediately ahead.

**Real-work angle:** Every `BaseService`, `BaseController`, `BaseRepository`, and `AbstractHandler` in your Node codebase is an ISP decision. The `BaseRepository` with 18 methods that every model extends — and half of them throw for half the models — is the single most common ISP violation in real Node apps.

---

## The core idea — explained simply

### The Swiss Army Knife vs. The Toolbox

You need to tighten one screw.

**Option A — the Swiss Army knife.** One object, 33 tools folded into it: screwdriver, corkscrew, saw, fish scaler, magnifying glass, toothpick. It *does* contain a screwdriver. But it's heavy — you carry 32 tools you'll never open. Sharpening the saw means taking the whole knife apart, and now the screwdriver is affected too. Hand it to someone who only fixes glasses and they still get the fish scaler. And if you want a fish-scaler-free model, you must build an entire second knife.

**Option B — the toolbox.** Separate tools. You pick up the screwdriver. That's it. The saw exists, in the box, but it has nothing to do with you today.

| Swiss Army knife | Toolbox | In code |
|---|---|---|
| One object with 33 capabilities | Many objects with 1 capability | Fat interface vs. **role interfaces** |
| Carry all 33 to use 1 | Carry only what you need | Client depends only on what it calls |
| Sharpening the saw disturbs the knife | Sharpening the saw disturbs the saw | **Change isolation** |
| Fish-scaler-less model = whole new knife | Just don't pack the fish scaler | Compose capabilities per implementer |
| A "screwdriver-only" knife must still *have* a fish scaler slot… blank | Nothing blank; nothing stubbed | No `throw new Error('not supported')` |

**The ISP move is always the same: take the big thing, and cut it along the lines of who actually uses what.** Not along the lines of what's convenient to store together.

### A note for JavaScript people

Here's the thing that confuses everybody: **JavaScript has no `interface` keyword.** There is no compiler forcing you to implement 14 methods. So does ISP even apply?

Yes — and *more*, not less. In Java the compiler at least *tells you* you've been handed a fat contract. In JS, the fat contract is invisible until it explodes at runtime. In JS, an "interface" is any of:

- a **base class** with methods that throw `'not implemented'` (the pattern from [02 — HLD vs LLD](./02-hld-vs-lld.md))
- a **duck-typed shape** — "whatever you pass me, it must have `.read()` and `.close()`"
- a **mixin** — an object of methods you splice into a class
- a **plain object of functions** you pass around (a role object / port)

ISP applies to all four. The principle isn't about a keyword. It's about **how many methods a client is required to know about.**

---

## Key concepts inside this topic

### 1. The classic violation: the fat `Worker` interface

The textbook example, in JS.

```javascript
// ❌ BAD — a fat "interface" as a base class.
// It bundles two unrelated concerns: DOING WORK and BEING ALIVE.
class Worker {
  work()  { throw new Error('Not implemented'); }
  eat()   { throw new Error('Not implemented'); }
  sleep() { throw new Error('Not implemented'); }
}

class HumanWorker extends Worker {
  work()  { return 'assembling the chassis'; }
  eat()   { return 'eating lunch'; }
  sleep() { return 'sleeping 8 hours'; }
}

class RobotWorker extends Worker {
  work()  { return 'welding at 200 units/hour'; }

  // ⚠️ THE STUBS. This is the sound of ISP being violated.
  eat()   { throw new Error('Robots do not eat'); }   // new exception → also breaks LSP
  sleep() { /* silently does nothing */ }             // weakened postcondition → also breaks LSP
}
```

Now the damage shows up in the **client**:

```javascript
// The client only ever needs work(). But it inherited the whole fat contract,
// so it can't safely trust anything, and defensive junk creeps in:
class ProductionLine {
  runShift(workers) {
    workers.forEach(w => {
      w.work();
      if (!(w instanceof RobotWorker)) w.eat();   // ← the smell, again
    });
  }
}
```

Notice: **the `RobotWorker` did nothing wrong.** The *interface* was wrong. It bundled `work()` (which every worker does) with `eat()`/`sleep()` (which only biological workers do). Every fat interface eventually forces some implementer to lie.

### 2. The fix: role interfaces

Cut the fat interface along the lines of **who uses what**. Each resulting piece is a **role interface** — a contract for one capability, usually 1-3 methods.

```javascript
// ✅ GOOD — three small roles instead of one fat contract.
class Workable { work()  { throw new Error('Not implemented'); } }
class Feedable { eat()   { throw new Error('Not implemented'); } }
class Restable { sleep() { throw new Error('Not implemented'); } }
```

JS has single inheritance, so you can't `extends Workable, Feedable`. That's fine — this is exactly what **mixins** are for (next section). But first, the payoff at the client:

```javascript
// ✅ The client now depends on EXACTLY what it uses. Nothing more.
class ProductionLine {
  // Duck-typed: "give me anything that can work()". Robots, humans, whatever.
  runShift(workables) { return workables.map(w => w.work()); }  // no instanceof, no stubs
}

class CafeteriaScheduler {
  // A DIFFERENT client with a DIFFERENT need. It never even sees work().
  serveLunch(feedables) { return feedables.map(f => f.eat()); }
}
```

`ProductionLine` can now be handed robots and humans together and never know the difference. `CafeteriaScheduler` is simply never handed robots — because robots aren't `Feedable`, and nobody had to write a stub to say so.

### 3. Mixins — how JS actually composes roles

A **mixin** is a function that takes a class and returns a new class with extra methods glued on. It's JavaScript's answer to "implement multiple interfaces."

```javascript
// ✅ GOOD — each mixin is ONE role. Compose only the roles you actually have.
const Workable   = (Base) => class extends Base { work()     { return `${this.name} is working`; } };
const Feedable   = (Base) => class extends Base { eat()      { return `${this.name} is eating`; } };
const Restable   = (Base) => class extends Base { sleep()    { return `${this.name} is sleeping`; } };
const Chargeable = (Base) => class extends Base { recharge() { return `${this.name} is charging`; } };

class Entity { constructor(name) { this.name = name; } }

// A human works, eats, and sleeps.
class HumanWorker extends Workable(Feedable(Restable(Entity))) {}

// A robot works and recharges. It has NO eat(). It never had to stub one.
class RobotWorker extends Workable(Chargeable(Entity)) {}

// --- demo ---
const line = new ProductionLine();
console.log(line.runShift([new HumanWorker('Ana'), new RobotWorker('R2')]));
// [ 'Ana is working', 'R2 is working' ]   ← robots and humans, side by side

console.log(new CafeteriaScheduler().serveLunch([new HumanWorker('Ana')]));
// [ 'Ana is eating' ]   ← R2 is simply never in this list. No stub said so.
```

The capability list of each class is now **declarative and readable at a glance**: `Workable(Chargeable(Entity))` tells you everything a robot can do, in one line.

### 4. Duck-typed role objects — the most idiomatic JS approach

You often don't need classes at all. In Node, the most common ISP-compliant "interface" is just **a plain object with the functions the client needs**, passed in.

```javascript
// ✅ GOOD — the "interface" is the shape of the argument. Nothing more.
// This function's contract is visible in its own signature: it needs ONE method.
async function exportUsers(reader, formatter) {
  const users = await reader.findAll();          // ← reader must have findAll(). That's all.
  return users.map(formatter);                   // ← formatter must be callable. That's all.
}

// Any of these satisfies it. None of them extends anything.
const pgReader     = { findAll: () => pool.query('SELECT * FROM users').then(r => r.rows) };
const memoryReader = { findAll: async () => [{ id: 1, name: 'Ana' }] };   // ← test double, 1 line
```

Look at that test double: **one line.** That's the real, everyday payoff of ISP. When the client depends on a 1-method role, the fake is 1 method. When it depends on a 20-method `Repository`, the fake is 20 methods (19 of which are `throw new Error('unused')`).

A useful sanity check on any function you write: **can you describe its dependencies' contracts in one sentence each?** If not, they're too fat.

### 5. The real Node example: fat `Repository` vs. `Readable` / `Writable` roles

This is the one you will actually hit at work.

```javascript
// ❌ BAD — the God-Repository. Every model extends it. Every client depends on all of it.
class Repository {
  async findById(id)          { throw new Error('Not implemented'); }
  async findAll(filter)       { throw new Error('Not implemented'); }
  async findPaginated(p, l)   { throw new Error('Not implemented'); }
  async save(entity)          { throw new Error('Not implemented'); }
  async update(id, patch)     { throw new Error('Not implemented'); }
  async delete(id)            { throw new Error('Not implemented'); }
  async bulkInsert(entities)  { throw new Error('Not implemented'); }
  async beginTransaction()    { throw new Error('Not implemented'); }
  async archive(id)           { throw new Error('Not implemented'); }
  async restore(id)           { throw new Error('Not implemented'); }
}

// A read-only view backed by an analytics replica. It CANNOT write. But the fat
// base forces it to pretend it has opinions about writing.
class AnalyticsUserRepository extends Repository {
  async findById(id)  { return this.db.query('SELECT ... FROM user_analytics ...'); }
  async findAll(f)    { return this.db.query('SELECT ... FROM user_analytics ...'); }

  // ⚠️ EIGHT STUBS. Eight lies. This class is 20% code and 80% apology.
  async save()             { throw new Error('Read-only replica'); }
  async update()           { throw new Error('Read-only replica'); }
  async delete()           { throw new Error('Read-only replica'); }
  async bulkInsert()       { throw new Error('Read-only replica'); }
  async beginTransaction() { throw new Error('Read-only replica'); }
  async archive()          { throw new Error('Read-only replica'); }
  async restore()          { throw new Error('Read-only replica'); }
  async findPaginated()    { throw new Error('Not supported on this view'); }
}

// ❌ And the client: it READS. It never writes. But it has been handed a 10-method
// dependency, so its unit test must now mock all 10 methods.
class UserReportService {
  constructor(userRepository) { this.repo = userRepository; }  // which 10-method thing? who knows
  async monthlySignups() {
    const users = await this.repo.findAll({ createdAfter: startOfMonth() });
    return users.length;
  }
}
```

**The ISP fix — split by capability, not by entity:**

```javascript
// ✅ GOOD — small roles. Name them after the CAPABILITY, not the table.
class ReadableRepo {
  async findById(id)    { throw new Error('Not implemented'); }
  async findAll(filter) { throw new Error('Not implemented'); }
}
class WritableRepo {
  async save(entity)      { throw new Error('Not implemented'); }
  async update(id, patch) { throw new Error('Not implemented'); }
  async delete(id)        { throw new Error('Not implemented'); }
}
class BulkWritableRepo { async bulkInsert(entities) { throw new Error('Not implemented'); } }
class ArchivableRepo {
  async archive(id) { throw new Error('Not implemented'); }
  async restore(id) { throw new Error('Not implemented'); }
}
```

Compose per implementation, using the mixin form so a class can hold several roles:

```javascript
// ✅ Read-only replica: it IS Readable. It is NOT Writable. Zero stubs. Zero lies.
class AnalyticsUserRepository extends ReadableRepo {
  constructor(db) { super(); this.db = db; }
  async findById(id)    { const r = await this.db.query('SELECT * FROM user_analytics WHERE id=$1', [id]); return r.rows[0]; }
  async findAll(filter) { const r = await this.db.query('SELECT * FROM user_analytics WHERE created_at > $1', [filter.createdAfter]); return r.rows; }
}

// ✅ The primary repo genuinely does both. Mixins let it hold BOTH roles.
const Readable = (Base) => class extends Base {
  async findById(id)    { const r = await this.db.query('SELECT * FROM users WHERE id=$1', [id]); return r.rows[0]; }
  async findAll(filter) { const r = await this.db.query('SELECT * FROM users WHERE created_at > $1', [filter.createdAfter]); return r.rows; }
};
const Writable = (Base) => class extends Base {
  async save(user)        { const r = await this.db.query('INSERT INTO users(name) VALUES($1) RETURNING *', [user.name]); return r.rows[0]; }
  async update(id, patch) { const r = await this.db.query('UPDATE users SET name=$2 WHERE id=$1 RETURNING *', [id, patch.name]); return r.rows[0]; }
  async delete(id)        { await this.db.query('DELETE FROM users WHERE id=$1', [id]); }
};

class Db { constructor(db) { this.db = db; } }
class PostgresUserRepository extends Readable(Writable(Db)) {}
```

And the client declares exactly the role it needs — in its own name:

```javascript
// ✅ The dependency's NAME now documents the contract. "reader" — it reads. Done.
class UserReportService {
  constructor(userReader) { this.reader = userReader; }   // needs ReadableRepo. 2 methods.
  async monthlySignups() {
    const users = await this.reader.findAll({ createdAfter: startOfMonth() });
    return users.length;
  }
}

// ✅ The test double is now trivially small. THIS is the payoff.
const fakeReader = { findAll: async () => [{ id: 1 }, { id: 2 }, { id: 3 }] };
await new UserReportService(fakeReader).monthlySignups();   // 3
```

Compare the two test setups honestly: **one line vs. a 10-method mock.** That difference, multiplied across a few hundred tests, is the difference between a suite people maintain and a suite people delete.

### 6. How fat is too fat? Practical heuristics

There's no magic number, but these are the signals engineers actually use:

| Signal | Reading |
|---|---|
| An implementer has **any** `throw new Error('not supported')` | ISP violated. Full stop. That method belongs on a narrower role |
| An implementer has an **empty method body** to satisfy the contract | Worse — silent failure. Same diagnosis |
| No single client calls **more than half** the methods | The interface is serving the *implementer's* convenience, not any client's need |
| The **test mock** is longer than the test | Your dependency is too fat |
| Two methods on the interface **always change for different reasons** | That's SRP ([14](./14-solid-single-responsibility.md)) telling you the same thing ISP is |

**The client-first rule:** *Design interfaces from the caller's side, not the implementer's side.* Ask "what does this client need?" — not "what can this class do?" A fat interface is almost always the result of someone listing everything the class *can* do and calling that a contract.

### 7. The counter-pressure: don't shatter everything into one-method interfaces

ISP taken to the extreme gives you 40 one-method roles and a wiring nightmare — a constructor taking twelve collaborators. That's not clean; that's shrapnel. **Cohesion is the brake pedal.** Methods *always used together by the same client* belong together: `save()`, `update()`, and `delete()` genuinely travel as a pack for write-side clients, so keep them as one `WritableRepo`. Splitting them into `Savable`, `Updatable`, `Deletable` buys you nothing and costs you three imports.

**The grouping test:** *Does any real client use A without B?* If yes, split. If no client ever uses one without the other, they belong in the same role.

---

## Visual / Diagram description

### Diagram 1: The fat interface — everybody depends on everything

```
   ┌─────────────────┐   ┌──────────────────┐   ┌──────────────────────┐
   │ UserReportSvc   │   │ SignupService    │   │  DataImportJob       │
   │ (reads only)    │   │ (reads + writes) │   │  (bulk writes only)  │
   └────────┬────────┘   └────────┬─────────┘   └──────────┬───────────┘
            │                     │                        │
            │  depends on ALL 10  │  depends on ALL 10     │ depends on ALL 10
            └──────────┬──────────┴───────────┬────────────┘
                       ▼                      ▼
        ┌────────────────────────────────────────────────────┐
        │              Repository  «fat interface»            │
        │  + findById()      + save()        + bulkInsert()   │
        │  + findAll()       + update()      + beginTx()      │
        │  + findPaginated() + delete()      + archive()      │
        │                                    + restore()      │
        └───────────────┬────────────────────────┬────────────┘
                        │ extends                │ extends
                        ▼                        ▼
      ┌──────────────────────────┐   ┌──────────────────────────────┐
      │ PostgresUserRepository   │   │  AnalyticsUserRepository     │
      │  implements all 10  ✅   │   │  implements 2                │
      │                          │   │  STUBS 8 ⚠️  ← the lies      │
      └──────────────────────────┘   └──────────────────────────────┘
```

Every arrow into the fat box is a client that knows about 10 methods to use 2 or 3. And the bottom-right box is where the interface's dishonesty finally becomes a runtime error.

### Diagram 2: Segregated roles — everybody depends on exactly what they use

```
   ┌─────────────────┐   ┌──────────────────┐   ┌──────────────────────┐
   │ UserReportSvc   │   │ SignupService    │   │  DataImportJob       │
   └────────┬────────┘   └────┬────────┬────┘   └──────────┬───────────┘
            │                 │        │                   │
            │ needs           │ needs  │ needs             │ needs
            ▼                 ▼        ▼                   ▼
   ┌──────────────────┐  ┌─────────────────┐   ┌──────────────────────┐
   │  ReadableRepo    │  │  WritableRepo   │   │  BulkWritableRepo    │
   │  «role»          │  │  «role»         │   │  «role»              │
   │  + findById()    │  │  + save()       │   │  + bulkInsert()      │
   │  + findAll()     │  │  + update()     │   │                      │
   │                  │  │  + delete()     │   │                      │
   └────────┬─────────┘  └────────┬────────┘   └──────────┬───────────┘
            │                     │                       │
            │  ┌──────────────────┴───────────────────────┘
            │  │        composed via mixins
            ▼  ▼                                    ▼
   ┌───────────────────────────────┐   ┌────────────────────────────┐
   │  PostgresUserRepository       │   │  AnalyticsUserRepository   │
   │  = Readable(Writable(         │   │  extends ReadableRepo      │
   │      BulkWritable(Db)))       │   │                            │
   │  ZERO stubs ✅                │   │  ZERO stubs ✅             │
   └───────────────────────────────┘   └────────────────────────────┘
```

Read the two diagrams side by side and the principle is visible geometrically: **in the fat version, every client's arrow lands on the same box. In the segregated version, each arrow lands on the smallest box that satisfies it.**

### Diagram 3: The dependency-weight test

```
  Client          Methods it CALLS   Methods it DEPENDS ON   Waste
  ─────────────────────────────────────────────────────────────────
  FAT REPOSITORY:
  UserReportSvc      2  ████            10  ████████████████   80%
  SignupService      4  ████████        10  ████████████████   60%
  DataImportJob      1  ██              10  ████████████████   90%

  SEGREGATED ROLES:
  UserReportSvc      2  ████             2  ████                0%
  SignupService      4  ████████         5  ██████████         20%
  DataImportJob      1  ██               1  ██                  0%
```

That "waste" column is not academic. It is *exactly* the surface area of code that can break your class without your class ever calling it — and *exactly* the size of the mock you must write to test it.

---

## Real world examples

### 1. Node.js streams — `Readable`, `Writable`, `Duplex`, `Transform`

Node did not ship one `Stream` class with `read()`, `write()`, `transform()`, and `pipe()` on it. It shipped **separate role classes** and composed them: `Readable` (pull data out — `fs.createReadStream`, `process.stdin`, an HTTP request), `Writable` (push data in — `fs.createWriteStream`, `process.stdout`, an HTTP response), `Duplex` (genuinely both, on independent channels — a TCP socket), and `Transform` (a Duplex whose output is a function of its input — `zlib.createGzip`).

A `Writable` file stream has **no `read()` method that throws.** It simply isn't `Readable`. That's ISP, shipped in the standard library, and it's why `.pipe()` composes so cleanly across unrelated implementations. Notice too that the roles are named after **what the client can do with them**, never after what they're made of.

### 2. Express middleware — the one-function interface

Express's extension contract is exactly one shape: `(req, res, next) => void`. Not a `Middleware` base class with `onRequest()`, `onResponse()`, `onError()`, `onUpgrade()`, and `priority()` that every plugin must half-implement.

The result is that the ecosystem — `cors`, `helmet`, `morgan`, `body-parser`, thousands of others — all satisfy the contract with a plain function and zero stubs. A maximally segregated interface produced a maximally large ecosystem. That is not a coincidence: **the smaller the contract, the more things can honestly satisfy it.**

### 3. The AWS SDK v3 modular clients — ISP at the package level

AWS SDK v2 was a single package: `require('aws-sdk')` gave you every service — S3, DynamoDB, EC2, Lambda, all of it — tens of megabytes, to upload one file.

SDK v3 split it into `@aws-sdk/client-s3`, `@aws-sdk/client-dynamodb`, and so on, and split each client's operations into individually importable commands. A Lambda that only does `PutObject` now imports only `S3Client` and `PutObjectCommand`, and its bundle shrinks by an order of magnitude.

That's ISP applied at the *package* level rather than the class level, and it shows the principle generalises: **don't force a consumer to take a dependency on capabilities it doesn't use — whether the unit is a method, a class, or an npm package.**

---

## Trade-offs

| Approach | Pros | Cons |
|---|---|---|
| **One fat interface** | Everything's in one place; one file to look at; easy to "just add a method" | Forces stubs and lies; drags LSP down with it; giant mocks; a change to any method ripples to every client |
| **Segregated role interfaces** | Clients depend only on what they use; tiny test doubles; change isolation; implementers never stub | More files/types; you must *choose* the seams; slightly more wiring at the composition root |
| **Duck typing (no declared interface at all)** | Maximum flexibility, zero ceremony, very idiomatic JS | The contract is implicit — nothing documents it but the call site; typos surface at runtime |
| **One-method interfaces for everything** | Perfect ISP compliance on paper | Constructor with 12 collaborators; wiring noise; low cohesion. Over-correction |

What you give up is a single obvious place to add a new method, some file count, and the comfort of `extends BaseRepository`. What you get is test doubles that are one line instead of forty, and the ability to add a read-only replica without writing eight `throw` statements.

**The sweet spot:** Group methods by **who calls them together**, not by what entity they operate on. If every client that calls `save()` also calls `update()` and `delete()`, those three are one role. If *any* real client needs `findAll()` without ever writing, reads are a separate role. Split at the seams your **clients** reveal — and stop splitting the moment no client would notice.

---

## Common interview questions on this topic

### Q1: "What is the Interface Segregation Principle?"
**Hint:** "No client should be forced to depend on methods it doesn't use." Then go one level deeper immediately: *"and the tell is that some implementer is forced to write `throw new Error('not supported')` — which means a fat interface doesn't just violate ISP, it forces a downstream LSP violation too."* Linking the two letters is what makes the answer land.

### Q2: "JavaScript doesn't have interfaces. Does ISP still apply?"
**Hint:** Yes, and it matters more, because nothing checks it for you. In JS an "interface" is a base class with throwing methods, a mixin, a duck-typed argument shape, or a plain object of functions passed in. ISP is about **how many methods a client must know about** — that question exists in every language, keyword or not.

### Q3: "How do you decide where to split an interface?"
**Hint:** Split by **client usage**, not by implementation convenience. Map each client to the methods it actually calls. Methods that always appear together in the same client stay together (that's cohesion); a method that some clients need and others never touch is a separate role. Concretely: if any real caller reads without ever writing, `Readable` and `Writable` are two roles.

### Q4: "Isn't ISP just SRP for interfaces?"
**Hint:** They're close cousins but not the same. **SRP** is about *why a class changes* — one reason to change. **ISP** is about *what a client is forced to depend on*. A class can have a single responsibility yet still expose a fat interface where different clients use disjoint method subsets. In practice they converge: applying ISP usually reveals an SRP violation in the implementer, and vice versa.

### Q5: "Can you over-apply ISP? What does that look like?"
**Hint:** Yes — the failure mode is a constructor with a dozen one-method collaborators and a wiring file nobody wants to touch. Cohesion is the counterweight. If no client ever uses `save()` without `update()`, splitting them into `Savable` and `Updatable` adds ceremony and buys nothing. The test is empirical: *does a real client use one without the other?* If not, don't split.

---

## Practice exercise

### Refactor the God-Printer

You've inherited an office-device library. Here's the code:

```javascript
class OfficeDevice {
  print(doc)       { throw new Error('Not implemented'); }
  scan()           { throw new Error('Not implemented'); }
  fax(doc, number) { throw new Error('Not implemented'); }
  staple(doc)      { throw new Error('Not implemented'); }
  duplexPrint(doc) { throw new Error('Not implemented'); }
  emailScan(to)    { throw new Error('Not implemented'); }
  getInkLevel()    { throw new Error('Not implemented'); }
}

class SimpleLaserPrinter extends OfficeDevice {
  print(doc)    { return `printing ${doc}`; }
  getInkLevel() { return 42; }
  scan()        { throw new Error('This printer cannot scan'); }
  fax()         { throw new Error('This printer cannot fax'); }
  staple()      { throw new Error('This printer cannot staple'); }
  duplexPrint() { throw new Error('No duplex unit'); }
  emailScan()   { throw new Error('This printer cannot scan'); }
}

class AllInOneMachine extends OfficeDevice { /* implements all 7 */ }

class NetworkScanner extends OfficeDevice {
  scan()        { return 'scanned.pdf'; }
  emailScan(to) { return `emailed scan to ${to}`; }
  // ...and 5 throwing stubs
}

// Client A: only ever PRINTS.
class PayslipJob {
  constructor(device) { this.device = device; }
  run(payslips) { return payslips.map(p => this.device.print(p)); }
}

// Client B: only ever SCANS and EMAILS.
class DocumentIntakeJob {
  constructor(device) { this.device = device; }
  run(to) { this.device.scan(); return this.device.emailScan(to); }
}
```

**Produce these (~35 min):**

1. **The usage map.** Draw a small table: rows = clients (`PayslipJob`, `DocumentIntakeJob`), columns = the 7 methods, and tick which client calls which. Then add rows for the three devices, ticking which they can *honestly* implement. **The seams in your interface are literally visible in this table** — the columns cluster.

2. **Define the roles.** From that table, define 3-5 role "interfaces" (base classes or mixins — your call, but justify which). Name them after capabilities (`Printable`, `Scannable`, …), never after devices.

3. **Rebuild the three devices** using mixins, composing only the roles each device honestly has. **Success criterion: your file contains zero `throw new Error('not supported')`.** If one is left, a role is still too fat.

4. **Rewrite the two clients** so each depends only on its role, and rename the constructor parameter to say so (`printer`, not `device`).

5. **Write the test doubles.** Show a unit test for `PayslipJob` whose fake is **one line long**, then one sentence comparing it to the fake you'd have needed against the original `OfficeDevice`.

6. **Add a `FaxOnlyLegacyMachine`.** Count the existing files you had to edit. (Zero — that's ISP paying you back with OCP.)

---

## Quick reference cheat sheet

- **ISP in one line:** no client should be forced to depend on methods it doesn't use.
- **The #1 smell:** `throw new Error('not supported')` or an **empty method body** in an implementer. That's the interface admitting it's too fat.
- **Fat interfaces cause LSP violations.** The stub *is* the violation — a new exception or a weakened postcondition. ISP and LSP fail together.
- **In JS an "interface" is:** a throwing base class, a mixin, a duck-typed argument shape, or a plain object of functions. **No keyword needed for the principle to apply.**
- **Mixins are JS's multiple-interface mechanism:** `class Robot extends Workable(Chargeable(Entity)) {}` — the capability list reads as documentation.
- **Design from the client's side, not the implementer's.** Ask "what does this caller need?", never "what can this class do?"
- **Split by co-usage:** methods that are always called together by the same client belong together. A method some clients never touch is a separate role.
- **Name roles after capabilities** (`Readable`, `Writable`, `Archivable`) — never after entities (`UserRepository`).
- **The mock-size test:** if your test double is longer than your test, your dependency is too fat.
- **The half rule:** if no single client calls more than half the interface's methods, it's serving the implementer, not the callers.
- **Cohesion is the brake.** Don't shatter into 1-method interfaces — a constructor with 12 collaborators is an over-correction.
- **It scales beyond classes.** AWS SDK v3's per-service packages are ISP applied to npm dependencies.
- **The payoff you'll feel first:** tiny test doubles. That alone justifies the refactor.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [16 — SOLID: Liskov Substitution Principle](./16-solid-liskov-substitution.md) — the stubs that ISP violations force are themselves LSP violations |
| **Next** | [18 — SOLID: Dependency Inversion Principle](./18-solid-dependency-inversion.md) — once your interfaces are small, you can inject them; DIP is what you *do* with the roles ISP gives you |
| **Related** | [24 — Coupling and Cohesion](./24-coupling-and-cohesion.md) — ISP is the "reduce coupling" lever; cohesion is the brake that stops you over-splitting |
| **Related** | [27 — Abstraction and Interfaces in Design](./27-abstraction-and-interfaces.md) — how to find the right abstractions in the first place |
