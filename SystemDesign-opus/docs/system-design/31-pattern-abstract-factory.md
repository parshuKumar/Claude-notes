# 31 — Abstract Factory Pattern
## Category: LLD Patterns

---

## What is this?

The **Abstract Factory** pattern gives you an object whose whole job is to produce a **family of related objects** that are meant to be used *together* — without your code ever naming the concrete classes it gets back.

Think of a **furniture showroom kit**. You walk in and say "I'll take the Victorian set." You don't pick a Victorian chair, then a Modern sofa, then an Art-Deco coffee table. You pick *one style*, and the showroom hands you a chair, a sofa, and a table that all match. The showroom is the factory. The style (Victorian, Modern) is the *family*. The chair/sofa/table are the *products*.

In code: instead of `new S3Storage()` and `new SqsQueue()` scattered around your app, you receive **one factory object** — `awsFactory` — and ask it for storage and a queue. Swap in `gcpFactory` and every piece of your app quietly switches to a fully-matched Google Cloud set. You changed one line.

---

## Why does it matter?

Because **some objects are only correct in combination**. That is the entire reason this pattern exists, and it's the thing beginners miss.

A `S3Storage` returns object keys like `s3://bucket/2024/07/file.json`. An `SqsQueue` knows how to parse that. A `PubSubQueue` does **not** — it expects `gs://bucket/...` and will happily accept the S3 key, publish it, and blow up three services downstream at 3am with `Invalid GCS URI`. Nothing in your type system stops you from wiring an S3 storage to a Pub/Sub queue. The mismatch is *silent at construction time* and *loud in production*.

Abstract Factory makes that mistake **impossible to express**. You cannot get an `S3Storage` and a `PubSubQueue` out of the same factory, because no such factory exists.

**Real-work angle:** you'll meet this every time you support more than one cloud, more than one database vendor, more than one platform, or — most commonly — every time you want a **fake in-memory version of your whole infrastructure for tests**. That last use is worth the pattern all by itself.

**Interview angle:** Abstract Factory is one half of a guaranteed follow-up pair. The interviewer asks you about Factory Method (topic 30), you answer well, and the next words out of their mouth are *"and how is that different from Abstract Factory?"* Most candidates fumble it. You won't, because you'll have the one-sentence answer: **Factory Method makes one product and varies by subclassing; Abstract Factory makes a family of products and varies by which factory object you inject.**

---

## The core idea — explained simply

### The IKEA Room-Set Analogy

You're furnishing a bedroom. Two ways to shop.

**Way 1 — buy piece by piece.** You order a bed frame from one catalogue, a mattress from another, a bedside table from a third. Everything arrives. The mattress is 200×160 and the frame is 190×140. The mattress does not fit the frame. Both items are *individually correct* and *jointly useless*. Nobody warned you, because nobody was looking at the combination.

**Way 2 — buy a room set.** You say "the MALM set, size double." IKEA hands you a frame, a mattress, and a table that were designed against the same measurements. You literally cannot receive a mismatched pair, because the set is the unit of purchase.

Abstract Factory is Way 2.

| In the analogy | In code | In our cloud example |
|---|---|---|
| The room set catalogue ("MALM double") | The **Abstract Factory interface** | `CloudFactory` — declares `createStorage()`, `createQueue()`, `createDatabase()` |
| One specific set ("MALM", "HEMNES") | A **Concrete Factory** | `AwsFactory`, `GcpFactory` |
| "A bed frame" as a category | An **Abstract Product** | `Storage`, `Queue`, `Database` base classes |
| The actual MALM frame you get | A **Concrete Product** | `S3Storage`, `SqsQueue`, `DynamoDb` |
| The measurements every MALM piece shares | The **hidden contract inside a family** | the key format `s3://…` that S3 emits and SQS understands |
| You, saying "one MALM set please" | The **client code** | your `IngestionService`, which never says `new S3Storage()` |

The crucial line in that table is the **hidden contract**. The frame and mattress share a measurement. `S3Storage` and `SqsQueue` share a URI format. That shared assumption is invisible, undeclared, and absolutely real — and the factory is what guarantees you never violate it.

So the rule: **when two or more objects only work correctly if they came from the same family, make the family the thing you choose, not the objects.**

---

## Key concepts inside this topic

### 1. The problem it solves — start with the painful code

Here is an ingestion service written the obvious way. It uploads a payload to blob storage, then publishes a pointer to a queue so a worker can pick it up later.

```js
// BAD — concrete classes hard-wired everywhere
import { S3Storage } from './aws/s3-storage.js';
import { SqsQueue } from './aws/sqs-queue.js';
import { DynamoDb } from './aws/dynamo-db.js';

class IngestionService {
  constructor() {
    // Every dependency is nailed to AWS at construction time.
    this.storage = new S3Storage('uploads-bucket');
    this.queue   = new SqsQueue('ingest-queue');
    this.db      = new DynamoDb('documents');
  }

  async ingest(doc) {
    const ref = await this.storage.put(doc.id, doc.body); // returns "s3://uploads-bucket/abc"
    await this.queue.publish({ ref });                    // SQS knows how to read "s3://..."
    await this.db.save({ id: doc.id, ref });
  }
}
```

Three problems, in increasing order of nastiness.

**(a) You cannot test it.** Instantiating `IngestionService` reaches for real AWS SDK clients. Your unit test now needs credentials, or a mountain of `jest.mock()` calls that break whenever you rename a file.

**(b) You cannot port it.** A customer demands GCP. You now grep for `new S3Storage` across 40 files. You will miss one.

**(c) The real killer — the half-migration.** Somebody starts the GCP move and gets halfway:

```js
// The bug that Abstract Factory exists to prevent
this.storage = new S3Storage('uploads-bucket'); // still AWS
this.queue   = new PubSubQueue('ingest-topic'); // migrated to GCP
```

This compiles. It runs. It even *appears to work*, because publishing a message is fire-and-forget. Then the worker on the other side does:

```js
// inside the GCP-side worker
const { bucket, object } = parseGcsUri(msg.ref); // msg.ref === "s3://uploads-bucket/abc"
// -> throws: Invalid GCS URI, expected gs:// scheme
```

The failure surfaces in a *different process*, *minutes later*, with a stack trace that points at the innocent worker. That is a genuinely expensive bug, and the compiler was never going to catch it — because `PubSubQueue` and `SqsQueue` have the same method signature. **They differ only in what they assume about the data.** That unwritten assumption is what "family" means.

### 2. The structure — who plays which role

| Role | Meaning | In our example |
|---|---|---|
| **Abstract Factory** | Declares one creator method per product type | `CloudFactory` |
| **Concrete Factory** | Implements all creators, returning one coherent family | `AwsFactory`, `GcpFactory`, `FakeFactory` |
| **Abstract Product** | The interface the client is allowed to depend on | `Storage`, `Queue`, `Database` |
| **Concrete Product** | Real implementation, belongs to exactly one family | `S3Storage`, `PubSubQueue`, `Firestore`, … |
| **Client** | Uses only abstract types; receives the factory | `IngestionService` |

The defining structural property: **two parallel hierarchies.** One hierarchy of factories, and one hierarchy *per product type*. A concrete factory picks exactly one leaf from each product hierarchy — and always the leaves from its own column.

### 3. Full JavaScript implementation — the cloud-provider kit

JavaScript has no `interface` keyword, so we express abstract products as base classes that throw. This is the idiomatic JS way to say "you must override me."

```js
// ---------- Abstract products (the contracts the client may depend on) ----------

class Storage {
  /** @returns {Promise<string>} an opaque reference to the stored blob */
  async put(_key, _body) { throw new Error('Storage.put() not implemented'); }
  async get(_ref)        { throw new Error('Storage.get() not implemented'); }
}

class Queue {
  /** Accepts a message containing a storage ref produced by THIS family's Storage. */
  async publish(_msg) { throw new Error('Queue.publish() not implemented'); }
}

class Database {
  async save(_record) { throw new Error('Database.save() not implemented'); }
}

// ---------- AWS family ----------

class S3Storage extends Storage {
  constructor(bucket) { super(); this.bucket = bucket; this.blobs = new Map(); }

  async put(key, body) {
    this.blobs.set(key, body);
    // The family's shared contract lives right here: the s3:// URI scheme.
    return `s3://${this.bucket}/${key}`;
  }

  async get(ref) {
    const key = ref.replace(`s3://${this.bucket}/`, '');
    return this.blobs.get(key);
  }
}

class SqsQueue extends Queue {
  constructor(queueName) { super(); this.queueName = queueName; this.sent = []; }

  async publish(msg) {
    // SQS-side worker will parse this ref. It ONLY understands s3://.
    if (!msg.ref.startsWith('s3://')) {
      throw new Error(`SqsQueue: cannot handle ref "${msg.ref}" — expected s3:// scheme`);
    }
    this.sent.push(msg);
    return { messageId: `sqs-${this.sent.length}` };
  }
}

class DynamoDb extends Database {
  constructor(table) { super(); this.table = table; this.rows = new Map(); }
  async save(record) { this.rows.set(record.id, record); return { ok: true }; }
}

// ---------- GCP family ----------

class GcsStorage extends Storage {
  constructor(bucket) { super(); this.bucket = bucket; this.blobs = new Map(); }
  async put(key, body) {
    this.blobs.set(key, body);
    return `gs://${this.bucket}/${key}`;   // different scheme — different family
  }
  async get(ref) {
    const key = ref.replace(`gs://${this.bucket}/`, '');
    return this.blobs.get(key);
  }
}

class PubSubQueue extends Queue {
  constructor(topic) { super(); this.topic = topic; this.sent = []; }
  async publish(msg) {
    if (!msg.ref.startsWith('gs://')) {
      // THIS is the production 3am error, reproduced faithfully.
      throw new Error(`PubSubQueue: invalid URI "${msg.ref}" — expected gs:// scheme`);
    }
    this.sent.push(msg);
    return { messageId: `psub-${this.sent.length}` };
  }
}

class Firestore extends Database {
  constructor(collection) { super(); this.collection = collection; this.docs = new Map(); }
  async save(record) { this.docs.set(record.id, record); return { ok: true }; }
}

// ---------- Abstract factory + concrete factories ----------

class CloudFactory {
  createStorage()  { throw new Error('createStorage() not implemented'); }
  createQueue()    { throw new Error('createQueue() not implemented'); }
  createDatabase() { throw new Error('createDatabase() not implemented'); }
}

class AwsFactory extends CloudFactory {
  constructor({ bucket, queueName, table }) { super(); this.cfg = { bucket, queueName, table }; }
  createStorage()  { return new S3Storage(this.cfg.bucket); }
  createQueue()    { return new SqsQueue(this.cfg.queueName); }
  createDatabase() { return new DynamoDb(this.cfg.table); }
}

class GcpFactory extends CloudFactory {
  constructor({ bucket, topic, collection }) { super(); this.cfg = { bucket, topic, collection }; }
  createStorage()  { return new GcsStorage(this.cfg.bucket); }
  createQueue()    { return new PubSubQueue(this.cfg.topic); }
  createDatabase() { return new Firestore(this.cfg.collection); }
}

// ---------- Client — knows NOTHING about S3, GCS, SQS or Pub/Sub ----------

class IngestionService {
  /** @param {CloudFactory} factory — the ONE seam through which a whole family enters */
  constructor(factory) {
    this.storage = factory.createStorage();
    this.queue   = factory.createQueue();
    this.db      = factory.createDatabase();
  }

  async ingest(doc) {
    const ref = await this.storage.put(doc.id, doc.body);
    await this.queue.publish({ ref, docId: doc.id });
    await this.db.save({ id: doc.id, ref, at: Date.now() });
    return ref;
  }
}
```

Now the demo — including the failure the pattern prevents.

```js
async function main() {
  const doc = { id: 'doc-42', body: 'hello world' };

  // --- AWS family: coherent, works ---
  const aws = new IngestionService(
    new AwsFactory({ bucket: 'uploads', queueName: 'ingest-q', table: 'documents' })
  );
  console.log('AWS ->', await aws.ingest(doc));   // AWS -> s3://uploads/doc-42

  // --- GCP family: also coherent, also works. Same client code. ---
  const gcp = new IngestionService(
    new GcpFactory({ bucket: 'uploads', topic: 'ingest-t', collection: 'documents' })
  );
  console.log('GCP ->', await gcp.ingest(doc));   // GCP -> gs://uploads/doc-42

  // --- The mixed family: what the pattern makes UNREACHABLE ---
  // You can only build this by bypassing the factory and calling `new` yourself:
  const frankenstein = Object.create(IngestionService.prototype);
  frankenstein.storage = new S3Storage('uploads');    // AWS
  frankenstein.queue   = new PubSubQueue('ingest-t'); // GCP  <-- mismatch
  frankenstein.db      = new Firestore('documents');

  try {
    await frankenstein.ingest(doc);
  } catch (err) {
    console.error('MIXED ->', err.message);
    // MIXED -> PubSubQueue: invalid URI "s3://uploads/doc-42" — expected gs:// scheme
  }
}

main();
```

Read that last block again. To produce the bug we had to **fight the pattern** — bypass the constructor and hand-assemble the object graph. When the only way in is `new IngestionService(factory)`, the mixed family is not a bug you have to remember not to write. It is a state the code cannot reach.

### 4. The lighter example — a themed UI kit

Same shape, smaller stakes, and the classic textbook framing. A theme is a family: a dark Button must sit next to a dark Checkbox and open a dark Dialog. A light Button in a dark Dialog is not a crash — it's just *wrong*, and wrongness spreads component by component until nobody knows which theme a screen is supposed to be.

```js
class Button   { render() { throw new Error('not implemented'); } }
class Checkbox { render() { throw new Error('not implemented'); } }
class Dialog   { render(_children) { throw new Error('not implemented'); } }

// Dark family
class DarkButton   extends Button   { render() { return '<button class="btn btn--dark">OK</button>'; } }
class DarkCheckbox extends Checkbox { render() { return '<input class="cb cb--dark" type="checkbox">'; } }
class DarkDialog   extends Dialog   { render(children) { return `<div class="dlg dlg--dark">${children.join('')}</div>`; } }

// Light family
class LightButton   extends Button   { render() { return '<button class="btn btn--light">OK</button>'; } }
class LightCheckbox extends Checkbox { render() { return '<input class="cb cb--light" type="checkbox">'; } }
class LightDialog   extends Dialog   { render(children) { return `<div class="dlg dlg--light">${children.join('')}</div>`; } }

class ThemeFactory {
  createButton()   { throw new Error('not implemented'); }
  createCheckbox() { throw new Error('not implemented'); }
  createDialog()   { throw new Error('not implemented'); }
}

class DarkThemeFactory extends ThemeFactory {
  createButton()   { return new DarkButton(); }
  createCheckbox() { return new DarkCheckbox(); }
  createDialog()   { return new DarkDialog(); }
}

class LightThemeFactory extends ThemeFactory {
  createButton()   { return new LightButton(); }
  createCheckbox() { return new LightCheckbox(); }
  createDialog()   { return new LightDialog(); }
}

// Client: one function, both themes, zero conditionals.
function renderSettingsScreen(theme /* : ThemeFactory */) {
  const dialog = theme.createDialog();
  return dialog.render([
    theme.createCheckbox().render(),
    theme.createButton().render(),
  ]);
}

const isNight = new Date().getHours() >= 19;
console.log(renderSettingsScreen(isNight ? new DarkThemeFactory() : new LightThemeFactory()));
```

Notice what vanished: there is **not a single `if (theme === 'dark')` inside the rendering code**. The one and only branch in the whole system is the line that picks the factory. That is the payoff — the conditional collapses from N places to 1.

### 5. A real Node.js ecosystem example

**Knex / Sequelize dialect objects.** When you configure Knex with `client: 'pg'` versus `client: 'mysql2'`, Knex loads a *dialect* — an object that produces a matched set: a **connection pool**, a **query compiler**, a **schema compiler**, and a **column compiler**. These must agree. Postgres identifiers are quoted with `"double quotes"`, MySQL with `` `backticks` ``. A Postgres query compiler paired with a MySQL column compiler would emit SQL that neither database accepts. Knex never lets that happen, because the dialect object — the concrete factory — hands you all four pieces or none.

**`node:test` / Vitest environment providers** work the same way: choosing `environment: 'jsdom'` swaps in a family of globals (`document`, `window`, `navigator`) that were built to be consistent with each other, rather than letting you mix a jsdom `document` with a Node `window` shim.

And the one you'll actually build yourself — **the test-double family**:

```js
// A third family, sitting alongside AwsFactory and GcpFactory.
class FakeStorage extends Storage {
  constructor() { super(); this.blobs = new Map(); }
  async put(key, body) { this.blobs.set(key, body); return `fake://${key}`; }
  async get(ref)       { return this.blobs.get(ref.replace('fake://', '')); }
}
class FakeQueue extends Queue {
  constructor() { super(); this.published = []; }   // assertable!
  async publish(msg) {
    if (!msg.ref.startsWith('fake://')) throw new Error('FakeQueue: foreign ref');
    this.published.push(msg);
    return { messageId: `fake-${this.published.length}` };
  }
}
class FakeDb extends Database {
  constructor() { super(); this.rows = new Map(); }
  async save(record) { this.rows.set(record.id, record); return { ok: true }; }
}

class FakeFactory extends CloudFactory {
  constructor() {
    super();
    // Hold references so the TEST can inspect what the service did.
    this.storage = new FakeStorage();
    this.queue   = new FakeQueue();
    this.db      = new FakeDb();
  }
  createStorage()  { return this.storage; }
  createQueue()    { return this.queue; }
  createDatabase() { return this.db; }
}

// The test — no mocking library, no network, no credentials, runs in 2ms.
import { test } from 'node:test';
import assert from 'node:assert/strict';

test('ingest stores the blob, queues its ref, and records it', async () => {
  const fake = new FakeFactory();
  const svc  = new IngestionService(fake);

  await svc.ingest({ id: 'doc-1', body: 'payload' });

  assert.equal(fake.storage.blobs.get('doc-1'), 'payload');
  assert.equal(fake.queue.published.length, 1);
  assert.equal(fake.queue.published[0].ref, 'fake://doc-1');
  assert.ok(fake.db.rows.has('doc-1'));
});
```

This is the argument that wins over sceptical teammates. The fake family isn't a testing hack bolted on — it is *just another concrete factory*, indistinguishable in shape from the production ones. You got a full in-memory clone of your infrastructure for the price of one extra class.

### 6. When NOT to use it — and the pain you're signing up for

**Don't use it when there is only one product type.** If your factory has exactly one `create()` method, you don't have a family — you have a Factory Method (topic 30), or more likely a plain function. `createLogger(env)` does not need three classes.

**Don't use it when the families will never grow.** "We might support Azure one day" is not a family. Build it when the second family actually arrives; the refactor is mechanical and you'll know the real shape by then.

**Don't use it when the products don't actually constrain each other.** If your `Logger` and your `Cache` share no hidden contract — a Datadog logger works perfectly fine next to a Redis cache — then bundling them into a factory buys you nothing but indirection. Plain dependency injection (pass the two objects into the constructor) is simpler and more flexible, because it lets you mix on purpose.

**The real pain — adding a new product to the family.** Say you now need `createSecretsManager()`. You must add it to `CloudFactory`, `AwsFactory`, `GcpFactory`, **and** `FakeFactory`, plus write three new concrete products. Adding a new *family* is cheap (one new class, nothing else changes). Adding a new *product type* is expensive (every factory changes). That asymmetry is the pattern's defining trade-off, and interviewers love to hear you name it unprompted.

**Common abuse:** the factory that grew a `create(kind)` method with a `switch` inside it. The moment you write `factory.create('storage')` instead of `factory.createStorage()`, you've thrown away the one thing the pattern gave you — a compile-time-ish guarantee about *which* products exist.

### 7. Related patterns and how they differ — the #1 interview follow-up

| | **Factory Method** (topic 30) | **Abstract Factory** (this topic) |
|---|---|---|
| **What it creates** | ONE product | A FAMILY of related products |
| **How you vary it** | Subclass the creator and override the method | Inject a different factory *object* |
| **Unit of substitution** | A method | An object |
| **Number of create methods** | One (`createTransport()`) | One per product type (`createStorage`, `createQueue`, `createDatabase`) |
| **Relies on** | Inheritance | Composition (the factory is passed in) |
| **Typical trigger** | "This subclass needs a different kind of X" | "These N objects must all come from the same vendor/theme/family" |
| **Guarantee it gives** | The right *product* | The right *combination* |

The mental soundbite: **Factory Method varies one product by subclass. Abstract Factory varies a whole family by parameter.** And note the implementation detail that ties them together — each `createX()` method *inside* an Abstract Factory is itself a Factory Method. Abstract Factory is usually built out of Factory Methods.

Two more neighbours:

- **Builder** (topic 32): Builder constructs **one complex object step by step** (`.setEngine().setWheels().build()`). Abstract Factory returns **several finished objects in one go**. Builder cares about *how* one thing is assembled; Abstract Factory cares about *which set* of things you get.
- **Singleton** (topic 29): concrete factories are frequently singletons — you want exactly one `AwsFactory` for the process. Different jobs, commonly combined.

---

## Visual / Diagram description

The signature of Abstract Factory is **two parallel hierarchies**. Draw them side by side; if you can reproduce this on a whiteboard, you understand the pattern.

```
        FACTORY HIERARCHY                          PRODUCT HIERARCHIES
                                        ┌────────────┐ ┌────────────┐ ┌────────────┐
   ┌────────────────────────┐           │ «abstract» │ │ «abstract» │ │ «abstract» │
   │     «abstract»         │           │  Storage   │ │   Queue    │ │  Database  │
   │     CloudFactory       │           ├────────────┤ ├────────────┤ ├────────────┤
   ├────────────────────────┤           │ put(k,b)   │ │ publish(m) │ │ save(r)    │
   │ createStorage()  ──────┼───────────▶ get(ref)   │ │            │ │            │
   │ createQueue()    ──────┼───────────────────────────▶          │ │            │
   │ createDatabase() ──────┼──────────────────────────────────────────▶          │
   └───────────┬────────────┘           └─────▲──────┘ └─────▲──────┘ └─────▲──────┘
               │ extends                      │              │              │
      ┌────────┴─────────┐                    │ extends      │              │
      │                  │                    │              │              │
┌─────┴──────┐    ┌──────┴─────┐        ┌─────┴──────┐ ┌─────┴──────┐ ┌─────┴──────┐
│ AwsFactory │    │ GcpFactory │        │ S3Storage  │ │  SqsQueue  │ │  DynamoDb  │  ← AWS family
├────────────┤    ├────────────┤        ├────────────┤ ├────────────┤ ├────────────┤
│createStorage│   │createStorage│       │ "s3://..." │ │ needs s3://│ │            │
│createQueue  │   │createQueue  │       └────────────┘ └────────────┘ └────────────┘
│createDatabase│  │createDatabase│      ┌────────────┐ ┌────────────┐ ┌────────────┐
└──────┬──────┘   └──────┬──────┘       │ GcsStorage │ │PubSubQueue │ │ Firestore  │  ← GCP family
       │ produces        │ produces     ├────────────┤ ├────────────┤ ├────────────┤
       └── AWS row ──────┼──────────────▶ "gs://..." │ │ needs gs://│ │            │
                         └── GCP row ───▶            │ │            │ │            │
                                        └────────────┘ └────────────┘ └────────────┘
```

Read it as a **grid**. The columns are product types (Storage, Queue, Database). The rows are families (AWS, GCP, Fake). A concrete factory is *a row*: it hands you one cell from each column, always from its own row. The bug this pattern kills is **reading across two different rows.**

Now the runtime view — how a family flows into the client:

```
   config.CLOUD = "gcp"
          │
          ▼
   ┌───────────────┐   "give me the factory for this family"
   │  main.js      │──────────────────────────────┐
   │ (composition  │                              │
   │  root)        │                              ▼
   └───────┬───────┘                     ┌──────────────────┐
           │ new IngestionService(factory)│    GcpFactory    │
           ▼                              └────────┬─────────┘
   ┌──────────────────────────┐                    │
   │    IngestionService      │  createStorage()   │
   │  (the CLIENT)            │◀───── GcsStorage ──┤
   │                          │  createQueue()     │
   │  this.storage : Storage  │◀───── PubSubQueue ─┤
   │  this.queue   : Queue    │  createDatabase()  │
   │  this.db      : Database │◀───── Firestore ───┘
   └──────────────────────────┘
        ▲
        │  Depends ONLY on the abstract types on the left.
        │  The word "Gcs" appears nowhere inside this class.
```

The key thing this second diagram shows: **there is exactly one place in the entire program that knows a concrete class name — the composition root** (`main.js`, the line that picks the factory). Everything downstream sees only `Storage`, `Queue`, `Database`. That single point of knowledge is what makes swapping a family a one-line change.

---

## Real world examples

### Cross-platform UI toolkits (Qt, Flutter, React Native)

Qt's widget system is the canonical case. A `QStyle` object is a concrete factory: choose `QWindowsStyle` and every button, scrollbar, and menu your app draws comes from the Windows family; choose `QMacStyle` and you get the macOS family. Application code calls the same `QPushButton` API throughout. Flutter does the same with `MaterialApp` vs `CupertinoApp` — each supplies a coherent set of widgets, transitions, and scroll physics. You never mix a Cupertino scroll bounce with a Material ripple, because the family is chosen once at the app root.

### Database driver families (Knex, Sequelize, JDBC)

As described above: Knex's dialect object bundles a query compiler, schema compiler, column compiler, and connection pool that all agree on identifier quoting, parameter placeholders (`$1` in Postgres, `?` in MySQL), and type mapping. In the JDBC world the same idea appears as `java.sql.Driver` producing a matched `Connection` → `Statement` → `ResultSet` chain. Try to mix vendors mid-chain and nothing works — which is precisely why the vendor is chosen once, at the driver.

### Test doubles: a `FakeFactory` for the whole infrastructure

This is the one you'll use most, and it's genuinely valuable rather than academic. Teams that have an `AwsFactory` and a `GcpFactory` almost always add a third: an in-memory family (`FakeStorage` + `FakeQueue` + `FakeDb`) whose products still enforce the family contract (`fake://` refs). Integration-style tests then run against the real `IngestionService` with zero mocking library, zero network, and full assertability — you can inspect `fake.queue.published` directly. LocalStack and the Firebase emulator suite are the heavyweight version of the same idea; a `FakeFactory` is the 100-line version that runs in milliseconds.

---

## Trade-offs

| Pros | What it buys you |
|---|---|
| **Family coherence is guaranteed** | Mixed-family bugs become unrepresentable, not merely discouraged |
| **Client depends only on abstractions** | Satisfies the Dependency Inversion Principle; the client is genuinely portable |
| **Swapping families is a one-line change** | Only the composition root knows a concrete class name |
| **Conditionals collapse** | `if (provider === 'aws')` appears once, not in 40 files |
| **Testing becomes trivial** | The fake family is just another factory — no mocking framework required |

| Cons | What it costs you |
|---|---|
| **Class explosion** | 3 product types × 3 families + 4 factory classes = 16 classes for what was 3 `new` calls |
| **New product type = edit every factory** | Adding `createSecretsManager()` touches the abstract factory and *all* concretes |
| **Indirection tax** | A newcomer tracing "where does `storage` come from?" must hop through the factory |
| **Over-abstraction risk** | Wildly overkill if the second family never arrives |
| **Hides real differences** | If GCS genuinely lacks a feature S3 has, the common interface either drops it or lies about it |

| | Simple factory | Factory Method | Abstract Factory | Plain DI |
|---|---|---|---|---|
| Products created | 1 | 1 | Many (a family) | Whatever you pass |
| Family coherence enforced | No | No | **Yes** | No |
| Cost | Tiny | Small | High | Tiny |
| Use when | Construction is just messy | Subclasses need different products | Products must match | Products are independent |

**The sweet spot:** reach for Abstract Factory when you have **two or more product types that share a hidden contract**, and **at least two real families** (a fake/test family counts as one — and it's usually your second). If you have one product type, use a Factory Method. If the products don't constrain each other, use plain dependency injection and keep the freedom to mix.

**Rule of thumb:** *if mixing two of these objects would be a bug, they belong to a family — and the family should be the thing you choose.*

---

## Common interview questions on this topic

### Q1: "What's the difference between Factory Method and Abstract Factory?"

**Hint:** The guaranteed follow-up. Lead with the one-liner: *Factory Method creates one product and varies it by subclassing the creator; Abstract Factory creates a family of related products and varies it by injecting a different factory object.* Then add depth: Factory Method leans on inheritance, Abstract Factory on composition; and each `createX()` method inside an Abstract Factory is itself a Factory Method — the patterns compose rather than compete. Finish with the *why*: Abstract Factory exists specifically to guarantee **combinations**, not individual objects.

### Q2: "Why not just pass a `provider: 'aws' | 'gcp'` string around and switch on it?"

**Hint:** Because that string gets switched on in *every* file that constructs anything — a shotgun conditional. Adding Azure means finding all of them. Worse, nothing forces the switches to stay in sync: one file gets an `azure` branch, another silently falls through to the AWS default, and you're back to a mixed family. Abstract Factory reduces the number of places that branch on the provider from N to exactly one.

### Q3: "What's the biggest weakness of Abstract Factory?"

**Hint:** Adding a **new product type** to the family. A new family is cheap — one new class, nothing else changes (Open/Closed Principle satisfied). But `createSecretsManager()` forces an edit to the abstract factory and *every* concrete factory, which violates Open/Closed in the other axis. Name this asymmetry explicitly; it shows you've actually maintained one of these.

### Q4: "How would you use this pattern to make a service testable?"

**Hint:** Add a third family: `FakeFactory` producing `FakeStorage` + `FakeQueue` + `FakeDb`, all in-memory, all still enforcing the family's ref contract. The service under test is constructed with the fake factory and exercised for real — no `jest.mock`, no network, no credentials, and you can assert directly on `fake.queue.published`. This is the answer that makes interviewers nod; it turns an "academic" pattern into a tooling win.

### Q5: "JavaScript has no interfaces. Does that break the pattern?"

**Hint:** No — it just changes the enforcement mechanism. Use base classes whose methods `throw new Error('not implemented')` (fails loudly on the first call), or plain object literals of factory functions (`{ createStorage: () => new S3Storage(...) }`) since JS has first-class functions and doesn't need classes at all. TypeScript restores compile-time checking with a real `interface CloudFactory`. The *shape* of the pattern — one seam, one family — is language-independent.

---

## Practice exercise

### Build a Notification Kit with three families

You're building notifications for a SaaS product. A notification has three collaborating pieces that **must come from the same family**:

- a **Renderer** — turns a payload into a body string
- a **Transport** — actually sends the body
- a **DeliveryLog** — records what was sent

The hidden contract: each renderer emits a body in a specific *format*, and only its sibling transport can send that format.

**Your task (~30 min):**

1. Write abstract base classes `Renderer` (`render(payload) -> string`), `Transport` (`send(to, body)`), and `DeliveryLog` (`record(entry)`), each throwing `not implemented`.
2. Build three families:
   - **Email**: `HtmlRenderer` (returns `<html>...</html>`), `SmtpTransport` (throws unless the body starts with `<html`), `EmailLog`.
   - **SMS**: `PlainTextRenderer` (returns a plain string, **max 160 chars**), `TwilioTransport` (throws if the body contains `<` or exceeds 160 chars), `SmsLog`.
   - **Fake**: `FakeRenderer` (returns `fake:<payload.title>`), `FakeTransport` (pushes into a public `this.sent` array; throws unless the body starts with `fake:`), `FakeLog`.
3. Write `NotificationFactory` (abstract) plus `EmailFactory`, `SmsFactory`, `FakeFactory`.
4. Write the client — `NotificationService` — which takes a factory in its constructor and exposes `async notify(to, payload)`. It must not mention a single concrete class.
5. Run all three families through the same `notify()` call.
6. **Prove the pattern earns its keep:** hand-assemble a mixed service (`HtmlRenderer` + `TwilioTransport`), call `notify()`, and show the error it throws. Write one sentence explaining why the factory makes that state unreachable.
7. **Stretch:** write a `node:test` case using `FakeFactory` that asserts `fake.transport.sent[0].body === 'fake:Welcome'` — with no mocking library.

**Deliverable:** one file, `notification-kit.js`, that runs with `node notification-kit.js` and prints the three successful sends plus the mixed-family error.

---

## Quick reference cheat sheet

- **One-line definition:** a factory of factories — creates a *family* of related objects that are designed to be used together, without naming their concrete classes.
- **The trigger:** you need a **matched set**, and mixing families is a bug.
- **The five roles:** Abstract Factory, Concrete Factory, Abstract Product, Concrete Product, Client.
- **The shape:** two parallel hierarchies — one of factories, one per product type. Think of a **grid**: columns are product types, rows are families, a concrete factory is a row.
- **The guarantee:** you cannot get an `S3Storage` and a `PubSubQueue` out of the same factory, because no such factory exists.
- **vs Factory Method:** Factory Method = ONE product, varied by **subclassing**. Abstract Factory = a FAMILY, varied by **injecting a different factory object**.
- **They compose:** each `createX()` inside an Abstract Factory *is* a Factory Method.
- **vs Builder:** Builder assembles ONE complex object step by step; Abstract Factory returns SEVERAL finished objects at once.
- **The asymmetry (know this cold):** adding a new **family** is cheap (one class). Adding a new **product type** is expensive (every factory changes).
- **Only one place** in the whole program names a concrete class — the composition root.
- **Best real payoff:** a `FakeFactory` giving you an in-memory clone of your entire infrastructure, with zero mocking library.
- **Skip it when:** there's only one product type, the second family will never exist, or the products don't actually constrain each other (then use plain DI).
- **Abuse smell:** `factory.create('storage')` with a `switch` inside — you just threw away the whole guarantee.
- **The rule of thumb:** if mixing two objects would be a bug, they're a family — make the family the thing you choose.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [30 — Factory Method Pattern](./30-pattern-factory-method.md) — creates ONE product by subclass override; Abstract Factory is built out of Factory Methods, and telling them apart is the #1 interview follow-up |
| **Next** | [32 — Builder Pattern](./32-pattern-builder.md) — the other creational pattern: assembles ONE complex object step by step, rather than returning a whole family at once |
| **Related** | [29 — Singleton Pattern](./29-pattern-singleton.md) — concrete factories are very often singletons; you want exactly one `AwsFactory` per process |
| **Related** | [02 — HLD vs LLD](./02-hld-vs-lld.md) — why patterns like this live squarely in the low-level design half of an interview, and how they surface there |
