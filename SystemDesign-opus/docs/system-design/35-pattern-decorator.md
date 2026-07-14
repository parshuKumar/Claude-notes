# 35 — Decorator Pattern

## Category: LLD Patterns

---

## What is this?

The **Decorator** pattern adds new responsibilities to an object **at runtime** by wrapping it in another object that has the **exact same interface**. The wrapper does a little extra work, then hands the call through to the thing it wraps.

Think of a coffee shop. You order an espresso. Then you say "with milk" — the barista doesn't reach for a completely different drink called `EspressoWithMilk`; she takes the espresso you already have and adds milk to it. Then "and caramel" — she adds caramel to that. At every step you still have *a drink*: something you can ask the price of and ask for a description of. Each addition wrapped the previous one, added its own cost, and passed the rest through.

---

## Why does it matter?

Because the obvious alternative — subclassing — explodes.

You have a `DataSource` and you want it optionally encrypted, optionally compressed, and optionally logged. With inheritance you need `EncryptedDataSource`, `CompressedDataSource`, `LoggedDataSource`, `EncryptedCompressedDataSource`, `EncryptedLoggedDataSource`, `CompressedLoggedDataSource`, `EncryptedCompressedLoggedDataSource`... For **n** independent optional features you need **2ⁿ − 1** subclasses. Three features = 7 classes. Five features = 31 classes. And ordering matters (compress-then-encrypt is not the same as encrypt-then-compress), so really you're heading toward permutations, not just combinations.

Decorator turns that combinatorial explosion into **n small classes** you compose at runtime.

**In interviews:** Decorator is the go-to answer for *"how would you add retries/logging/caching to this without touching the existing class?"* and *"how do you avoid subclass explosion?"* It's also one half of the classic pairing question: **Decorator vs Proxy** (same interface for both — so what's the difference?).

**At work:** You are already using Decorator whether you know it or not. `stream.pipe(gzip).pipe(cipher)` is Decorator. Express middleware is Decorator. A `withRetry(fn)` higher-order function is Decorator. Naming it lets you *design* with it instead of stumbling into it.

---

## The core idea — explained simply

### The Coffee Shop Analogy

You order an **Espresso** ($2.00). The barista hands you a cup. You can ask it two things: *what are you?* and *what do you cost?* It answers `"Espresso"`, `$2.00`.

You say **add milk**. She does not throw the cup away and fetch a different pre-made product — she takes **the cup you already have** and pours milk into it. Ask again: `"Espresso, Milk"`, `$2.00 + $0.50 = $2.50`.

You say **add caramel**. Same again, on the drink she already has: `"Espresso, Milk, Caramel"`, `$2.50 + $0.70 = $3.20`.

Notice three things — they *are* the pattern:

1. **After every step you still have a drink.** Same interface — you can always ask cost and description. That's *why* you can keep adding.
2. **Each add-on knows only its own bit** — add my cost to whatever the inner thing costs, append my name to whatever it's called. Milk has never heard of caramel.
3. **Order is visible in the result.** `"Espresso, Milk, Caramel"` differs from `"Espresso, Caramel, Milk"`. In coffee that's cosmetic; in software (compress-then-encrypt vs encrypt-then-compress) it changes everything.

Mapping back:

| Coffee shop | Decorator pattern term | In code |
|-------------|------------------------|---------|
| "A drink" — anything you can price and describe | **Component** (the interface) | `class Beverage { cost(); description(); }` |
| The plain espresso | **Concrete Component** | `class Espresso extends Beverage` |
| "An add-on, which is still a drink" | **Base Decorator** | `class AddOn extends Beverage { constructor(inner) }` |
| Milk, Caramel, ExtraShot | **Concrete Decorators** | `class Milk extends AddOn` |
| Pouring milk into the cup you're holding | **Wrapping** | `new Milk(new Espresso())` |
| The final cup in your hand | The **decorated object** | still just a `Beverage` to the caller |

In code, it's shorter than the explanation:

```javascript
// COMPONENT — the interface everything in this world shares.
class Beverage {
  cost() { throw new Error('not implemented'); }
  description() { throw new Error('not implemented'); }
}

// CONCRETE COMPONENT — the thing being decorated.
class Espresso extends Beverage {
  cost() { return 2.00; }
  description() { return 'Espresso'; }
}

// BASE DECORATOR — IS-A Beverage (so it's interchangeable) and HAS-A
// Beverage (so it can wrap). That dual nature is the entire trick.
class AddOn extends Beverage {
  constructor(inner) { super(); this.inner = inner; }
  cost() { return this.inner.cost(); }                 // default: pass through
  description() { return this.inner.description(); }
}

// CONCRETE DECORATORS — add my bit, delegate the rest.
class Milk extends AddOn {
  cost() { return this.inner.cost() + 0.50; }
  description() { return `${this.inner.description()}, Milk`; }
}
class Caramel extends AddOn {
  cost() { return this.inner.cost() + 0.70; }
  description() { return `${this.inner.description()}, Caramel`; }
}

const drink = new Caramel(new Milk(new Espresso()));
console.log(drink.description(), '=> $' + drink.cost().toFixed(2));
// Espresso, Milk, Caramel => $3.20
```

That IS-A **and** HAS-A line is the whole pattern. `Milk` *is* a `Beverage`, so anywhere a beverage is expected you can hand over a `Milk`. And `Milk` *holds* a `Beverage`, so it can wrap one. That's what lets them stack infinitely deep.

---

## Key concepts inside this topic

### 1. The problem it solves — subclass explosion

Here's the painful version. You have a `DataSource` that reads and writes a file. Now: some data must be encrypted, some compressed, some both, and audit-critical data must be logged.

```javascript
// ❌ BAD — one subclass per combination
class FileDataSource { write(data) {} read() {} }
class EncryptedFileDataSource extends FileDataSource { /* encrypt, then super.write */ }
class CompressedFileDataSource extends FileDataSource { /* compress, then super.write */ }
class LoggedFileDataSource extends FileDataSource { /* log, then super.write */ }

// Now the combinations...
class EncryptedCompressedFileDataSource extends FileDataSource { /* both */ }
class EncryptedLoggedFileDataSource extends FileDataSource { /* both */ }
class CompressedLoggedFileDataSource extends FileDataSource { /* both */ }
class EncryptedCompressedLoggedFileDataSource extends FileDataSource { /* all three */ }
// 7 classes for 3 features. And the encryption code is now copy-pasted into 4 of them.
```

Do the arithmetic, because the interviewer will:

| Optional features (n) | Subclasses needed (2ⁿ − 1) |
|---|---|
| 2 | 3 |
| 3 | 7 |
| 4 | 15 |
| 5 | 31 |
| 6 | 63 |
| 10 | 1023 |

And it gets worse than the number suggests:

- **Duplication.** The encryption code appears in `Encrypted…`, `EncryptedCompressed…`, `EncryptedLogged…`, `EncryptedCompressedLogged…`. Fix a bug in it four times.
- **Compile-time lock-in.** The combination is baked in when you write the class. You can't decide *at runtime* — "encrypt only if this user is in the EU" — without a factory full of `if`s.
- **New feature = double the classes.** Adding "retry" to the 7 above gives you 15.
- **Ordering is unrepresentable.** `EncryptedCompressed` — did it compress first or encrypt first? The class name can't say, and it matters enormously (encrypted bytes look random and compress to *nothing*, so encrypt-then-compress is a real bug).

Decorator replaces all of it with **3 classes**, composed at runtime, in whatever order you like.

### 2. The structure — who plays which role

| Participant | Role | In the DataSource example |
|-------------|------|---------------------------|
| **Component** | The shared interface. Both the real thing and every wrapper implement it. | `DataSource` (abstract `write`/`read`) |
| **Concrete Component** | The real object doing the actual work. | `FileDataSource` |
| **Base Decorator** | Abstract wrapper. IS-A Component, HAS-A Component. Delegates everything by default. | `DataSourceDecorator` |
| **Concrete Decorator** | Adds one specific responsibility, then delegates. | `EncryptionDecorator`, `CompressionDecorator` |
| **Client** | Holds a `Component` reference. Cannot tell how many wrappers are underneath it. | `saveUserProfile(source, data)` |

The non-negotiable rule: **a decorator must not change the interface.** If it adds a new public method the component doesn't have, it stops being interchangeable, and once it isn't interchangeable you can't stack it. (That's precisely how Decorator differs from Adapter — see section 6.)

### 3. Full JavaScript implementation — DataSource

Runnable with `node decorator-demo.js`. The crypto and compression are real Node built-ins.

```javascript
import { gzipSync, gunzipSync } from 'node:zlib';
import { createCipheriv, createDecipheriv, randomBytes, scryptSync } from 'node:crypto';

// ===================================================================
// COMPONENT — the interface. Every real source and every decorator
// implements exactly this. Nothing more, nothing less.
// ===================================================================
class DataSource {
  /** @param {Buffer} data */
  write(data) { throw new Error('DataSource.write() not implemented'); }
  /** @returns {Buffer} */
  read() { throw new Error('DataSource.read() not implemented'); }
}

// ===================================================================
// CONCRETE COMPONENT — the one that actually stores bytes.
// (In-memory here so the demo runs anywhere; swap for fs.writeFileSync.)
// ===================================================================
class InMemoryDataSource extends DataSource {
  constructor(name) { super(); this.name = name; this.buffer = Buffer.alloc(0); }
  write(data) {
    this.buffer = Buffer.from(data);
    console.log(`  [store]   wrote ${this.buffer.length} bytes to "${this.name}"`);
  }
  read() {
    console.log(`  [store]   read ${this.buffer.length} bytes from "${this.name}"`);
    return this.buffer;
  }
}

// ===================================================================
// BASE DECORATOR — IS-A DataSource (so it's swappable) and
// HAS-A DataSource (so it can wrap). Default behaviour: pure delegation.
// Subclasses override only the part they care about.
// ===================================================================
class DataSourceDecorator extends DataSource {
  constructor(wrappee) {
    super();
    if (!(wrappee instanceof DataSource)) {
      throw new TypeError('A decorator must wrap a DataSource');
    }
    this.wrappee = wrappee;
  }
  write(data) { return this.wrappee.write(data); }
  read() { return this.wrappee.read(); }
}

// ===================================================================
// CONCRETE DECORATOR 1 — Compression.
// Note the symmetry: whatever write() does on the way IN,
// read() must undo on the way OUT. Every decorator obeys this.
// ===================================================================
class CompressionDecorator extends DataSourceDecorator {
  write(data) {
    const compressed = gzipSync(data);
    console.log(`  [gzip]    ${data.length} B -> ${compressed.length} B`);
    return this.wrappee.write(compressed);   // delegate the compressed bytes inward
  }
  read() {
    const compressed = this.wrappee.read();  // get bytes from inside
    const data = gunzipSync(compressed);
    console.log(`  [gunzip]  ${compressed.length} B -> ${data.length} B`);
    return data;
  }
}

// ===================================================================
// CONCRETE DECORATOR 2 — Encryption (AES-256-CBC).
// The IV is prepended to the ciphertext so read() can recover it.
// ===================================================================
class EncryptionDecorator extends DataSourceDecorator {
  constructor(wrappee, password) {
    super(wrappee);
    this.key = scryptSync(password, 'a-fixed-salt', 32); // 32 bytes for aes-256
  }
  write(data) {
    const iv = randomBytes(16);
    const cipher = createCipheriv('aes-256-cbc', this.key, iv);
    const body = Buffer.concat([cipher.update(data), cipher.final()]);
    const payload = Buffer.concat([iv, body]);            // iv || ciphertext
    console.log(`  [encrypt] ${data.length} B -> ${payload.length} B`);
    return this.wrappee.write(payload);
  }
  read() {
    const payload = this.wrappee.read();
    const iv = payload.subarray(0, 16);
    const body = payload.subarray(16);
    const decipher = createDecipheriv('aes-256-cbc', this.key, iv);
    const data = Buffer.concat([decipher.update(body), decipher.final()]);
    console.log(`  [decrypt] ${payload.length} B -> ${data.length} B`);
    return data;
  }
}

// ===================================================================
// CONCRETE DECORATOR 3 — Audit logging. Adds behaviour but changes
// NOTHING about the bytes. A decorator is free to be a pure observer.
// ===================================================================
class AuditLogDecorator extends DataSourceDecorator {
  constructor(wrappee, actor) { super(wrappee); this.actor = actor; }
  write(data) {
    console.log(`  [audit]   ${this.actor} wrote ${data.length} B at ${new Date().toISOString()}`);
    return this.wrappee.write(data);
  }
  read() {
    console.log(`  [audit]   ${this.actor} read at ${new Date().toISOString()}`);
    return this.wrappee.read();
  }
}

// ===================================================================
// CLIENT — takes ANY DataSource. It cannot tell, and does not care,
// how many layers are underneath. That is the entire payoff.
// ===================================================================
function saveProfile(source, profile) {
  source.write(Buffer.from(JSON.stringify(profile)));
}
function loadProfile(source) {
  return JSON.parse(source.read().toString());
}

// ===================================================================
// DEMO
// ===================================================================
function main() {
  const profile = {
    id: 'u_1',
    name: 'Ada Lovelace',
    bio: 'Mathematician. '.repeat(20),   // repetitive -> compresses well
  };

  // --- Case A: plain, no decorators at all
  console.log('\nA) Plain store');
  const plain = new InMemoryDataSource('plain.json');
  saveProfile(plain, profile);
  console.log('  ->', loadProfile(plain).name);

  // --- Case B: compress, THEN encrypt. Order chosen deliberately:
  // compression first (real data is compressible), encryption second
  // (ciphertext is high-entropy and would not compress at all).
  console.log('\nB) Audit -> Encrypt -> Compress -> Store');
  const secure = new AuditLogDecorator(
    new EncryptionDecorator(
      new CompressionDecorator(
        new InMemoryDataSource('secure.bin')
      ),
      'correct-horse-battery-staple'
    ),
    'alice@corp.com'
  );
  saveProfile(secure, profile);
  console.log('  ---');
  console.log('  ->', loadProfile(secure).name);

  // --- Case C: same three features, DIFFERENT composition, decided at
  // RUNTIME from config. Impossible with subclasses without a class per case.
  console.log('\nC) Runtime-configured stack');
  const config = { compress: true, encrypt: false, audit: true };
  let ds = new InMemoryDataSource('configured.bin');
  if (config.compress) ds = new CompressionDecorator(ds);
  if (config.encrypt)  ds = new EncryptionDecorator(ds, 'pw');
  if (config.audit)    ds = new AuditLogDecorator(ds, 'bob@corp.com');
  saveProfile(ds, profile);
  console.log('  ->', loadProfile(ds).name);
}

main();
```

Read the output and you'll see the call travelling **down** through the wrappers on `write`, and back **up** through them on `read`, each layer undoing what it did. That mirror symmetry is the discipline that keeps a decorator stack correct.

### 4. Second example — decorating an HTTP client

The DataSource example is the textbook one. This is the one you'll actually build at work: cross-cutting concerns (retry, logging, caching, rate-limiting) layered onto a client — all sharing the same `get(url)` interface.

```javascript
// ---- COMPONENT ----
class HttpClient {
  /** @returns {Promise<{status:number, body:any}>} */
  async get(url) { throw new Error('not implemented'); }
}

// ---- CONCRETE COMPONENT: the real network call ----
class FetchHttpClient extends HttpClient {
  constructor() { super(); this.callCount = 0; }
  async get(url) {
    this.callCount++;
    // Simulated: fails the first 2 times, then succeeds. Shows retry working.
    if (this.callCount <= 2) {
      const err = new Error('ECONNRESET');
      err.retryable = true;
      throw err;
    }
    return { status: 200, body: { url, data: 'hello', fetchedAt: Date.now() } };
  }
}

// ---- BASE DECORATOR ----
class HttpClientDecorator extends HttpClient {
  constructor(inner) { super(); this.inner = inner; }
  async get(url) { return this.inner.get(url); }
}

// ---- Retry: catch, back off, try again. Exponential backoff with jitter. ----
class RetryDecorator extends HttpClientDecorator {
  constructor(inner, { maxAttempts = 3, baseDelayMs = 50 } = {}) {
    super(inner);
    this.maxAttempts = maxAttempts;
    this.baseDelayMs = baseDelayMs;
  }
  async get(url) {
    let lastErr;
    for (let attempt = 1; attempt <= this.maxAttempts; attempt++) {
      try {
        return await this.inner.get(url);
      } catch (err) {
        lastErr = err;
        if (!err.retryable || attempt === this.maxAttempts) throw err;
        // 50ms, 100ms, 200ms... + jitter so a thousand clients don't retry in lockstep
        const delay = this.baseDelayMs * 2 ** (attempt - 1) + Math.random() * 20;
        console.log(`    [retry]  attempt ${attempt} failed (${err.message}); waiting ${delay | 0}ms`);
        await new Promise((r) => setTimeout(r, delay));
      }
    }
    throw lastErr;
  }
}

// ---- Logging: pure observation, zero effect on the result ----
class LoggingDecorator extends HttpClientDecorator {
  async get(url) {
    const start = Date.now();
    try {
      const res = await this.inner.get(url);
      console.log(`    [log]    GET ${url} -> ${res.status} in ${Date.now() - start}ms`);
      return res;
    } catch (err) {
      console.log(`    [log]    GET ${url} -> ERROR ${err.message} in ${Date.now() - start}ms`);
      throw err;   // decorators must not swallow errors they don't handle
    }
  }
}

// ---- Caching: may skip the inner call entirely ----
class CachingDecorator extends HttpClientDecorator {
  constructor(inner, ttlMs = 5000) { super(inner); this.ttlMs = ttlMs; this.cache = new Map(); }
  async get(url) {
    const hit = this.cache.get(url);
    if (hit && Date.now() - hit.at < this.ttlMs) {
      console.log(`    [cache]  HIT  ${url}`);
      return hit.res;
    }
    console.log(`    [cache]  MISS ${url}`);
    const res = await this.inner.get(url);
    this.cache.set(url, { res, at: Date.now() });
    return res;
  }
}

// ---- DEMO: order matters, a LOT ----
async function httpDemo() {
  // Cache OUTSIDE retry means: a cache hit costs nothing and never retries.
  // Retry INSIDE means: transient failures are absorbed before they reach the cache.
  // Logging outermost means: the log line reports total wall time, retries included.
  const client = new LoggingDecorator(
    new CachingDecorator(
      new RetryDecorator(
        new FetchHttpClient(),
        { maxAttempts: 4 }
      ),
      10_000
    )
  );

  console.log('\nFirst call (network flaky, retries kick in):');
  await client.get('https://api.example.com/users/1');

  console.log('\nSecond call (served from cache, no network, no retry):');
  await client.get('https://api.example.com/users/1');
}

httpDemo();
```

**Why ordering is a design decision, not an accident:**

| Stack (outermost first) | Behaviour |
|---|---|
| `Log(Cache(Retry(Fetch)))` | Cache hits are free. Retries happen below the cache, so a successful retry populates it. Log shows total time. |
| `Log(Retry(Cache(Fetch)))` | Retry re-consults the cache each attempt — pointless on a miss, and a cached *error* would be retried forever. Usually wrong. |
| `Cache(Log(Retry(Fetch)))` | Cache hits produce **no log line at all** — you'd think your service was idle. Usually wrong. |

Same three classes, three completely different systems. That's the power *and* the danger.

### 5. A real Node.js ecosystem example

Node is built on Decorator. Once you see it, you can't unsee it:

**Streams and `.pipe()`.** This is the canonical one:
```javascript
fs.createReadStream('big.log')
  .pipe(zlib.createGzip())        // Decorator: still a stream, now compressed
  .pipe(crypto.createCipheriv(…)) // Decorator: still a stream, now encrypted
  .pipe(fs.createWriteStream('big.log.gz.enc'));
```
Every element is a `Stream`. Each transform wraps the data flow, adds one behaviour, and passes it on. Same interface, stackable, order-sensitive — a textbook decorator chain.

**Express middleware.** Each middleware receives `(req, res, next)`, may do work before calling `next()`, and may do work after. `app.use(compression())` wraps every downstream handler with gzip. `app.use(morgan('combined'))` wraps them with logging. The handler signature never changes. (Purists point out this is *also* Chain of Responsibility — see [47](./47-pattern-chain-of-responsibility.md) — and both readings are correct. Middleware chains where every link runs and the innermost handler does the real work behave like decorators; chains where one link *handles and stops* behave like Chain of Responsibility.)

**Higher-order functions.** In JS you don't always need classes. A function that takes a function and returns a function with the same signature *is* a decorator:
```javascript
const withRetry = (fn, attempts = 3) => async (...args) => {
  for (let i = 1; i <= attempts; i++) {
    try { return await fn(...args); }
    catch (e) { if (i === attempts) throw e; }
  }
};
const withTiming = (fn) => async (...args) => {
  const t = Date.now();
  try { return await fn(...args); }
  finally { console.log(`took ${Date.now() - t}ms`); }
};

// Same interface in, same interface out — so they compose:
const robustFetchUser = withTiming(withRetry(fetchUser));
await robustFetchUser('u_1');   // callers see no difference
```
Same pattern, no `class` keyword. Reach for the functional form when the component has **one** method; reach for classes when it has several (you'd otherwise have to wrap each one).

**Others:** `axios` interceptors, `winston` format chains, and `util.promisify` (which is technically an Adapter — it *changes* the interface from callbacks to promises).

### 6. When NOT to use it / how it's abused

**Don't use it when the interface has many methods.** A `Repository` with 20 methods means every decorator must delegate all 20 just to modify one. That's 19 lines of noise per decorator. Fix: shrink the interface first, or use a `Proxy` object / higher-order function.

**Don't stack more than ~4 deep.** A stack trace through eight wrappers is genuinely miserable to debug, and nobody can tell you what the effective behaviour is any more. If you need eight, you probably want a **pipeline with a declared, named configuration** rather than hand-nested constructors.

**Never let a decorator change the interface.** Adding `client.clearCache()` to `CachingDecorator` seems harmless — but the moment you wrap that in `RetryDecorator`, `clearCache()` is gone (the retry decorator doesn't have it). Once you need to reach past a wrapper for a method, the stack is broken. Either put the method on the Component interface or don't add it.

**Identity gets weird.** `decorated instanceof FileDataSource` is `false`. `decorated === original` is `false`. If any code relies on object identity, reference equality, or `instanceof` against the *concrete* class, decorators will break it silently.

**Don't decorate to add unconditional behaviour.** If *every* `DataSource` must be logged, no exceptions, put the logging in the base class or a single choke point. Decorator earns its keep only when the behaviour is genuinely **optional or composable**.

**The order trap.** `Encrypt(Compress(store))` compresses then encrypts — good. `Compress(Encrypt(store))` encrypts then compresses — and ciphertext is indistinguishable from random noise, so gzip makes it *bigger*. The pattern will not warn you. Document the intended order.

### 7. Related patterns and how they differ (the #1 interview follow-up)

| Pattern | Interface | Purpose | The giveaway |
|---------|-----------|---------|--------------|
| **Decorator** | **Same** as wrapped | **Add** responsibilities | The wrapper *always* calls through and *adds* something |
| **Proxy** | **Same** as wrapped | Control **access** | The wrapper may *refuse* or *defer* the call |
| **Adapter** | **Different** from wrapped | Convert an interface | Method names change |
| **Facade** | **New**, over many objects | Simplify a subsystem | Wraps several classes, not one |
| **Composite** | Same, but wraps **many** children | Treat a tree like a leaf | One-to-many; Decorator is one-to-one |
| **Strategy** | N/A — you *inject* a behaviour | Swap an algorithm | Changes the *insides*; Decorator changes the *skin* |
| **Chain of Responsibility** | Same handler signature | Pass a request until someone handles it | A link may *stop* the chain; a decorator always continues |

**Decorator vs Proxy** — this is *the* question, because the class diagrams are identical.
The difference is **intent**, and you can see it in the code:
- A **decorator** always delegates and *enhances*: `log(); return this.inner.get(url);`
- A **proxy** decides *whether* to delegate at all: `if (!user.isAdmin) throw new Forbidden(); return this.inner.get(url);` or `this.real ??= new HeavyThing(); return this.real.get(url);` (lazy loading), or it delegates *over a network* (remote proxy).

Also: a decorator is handed its wrappee from outside (`new Milk(espresso)`) and you can stack many. A proxy typically **creates or manages** its subject itself, and you have one. Rule: **Decorator adds to what the object does; Proxy controls whether/when/how you reach the object.** A `CachingDecorator` that skips the network is honestly on the borderline — many people call that a caching *proxy*, and they're not wrong. Say so in an interview; the nuance scores points.

**Decorator vs Adapter** — Decorator *must* preserve the interface, precisely because decorators stack: `new Retry(new Log(new Cache(x)))` only type-checks if all four are the same type. An Adapter *deliberately changes* the interface, so adapters don't stack. See [34 — Adapter](./34-pattern-adapter.md).

**Decorator vs Strategy** — Strategy changes an object's *internal* algorithm by injecting it (`new Sorter(new QuickSort())`). Decorator leaves the object untouched and wraps it from the *outside*. Strategy = skin-deep change to the guts; Decorator = guts untouched, new skin.

---

## Visual / Diagram description

### Diagram 1: The class structure

```
              ┌──────────────────────────────┐
              │      «interface» Component    │   Everything below IS one of these.
              │          DataSource           │   The client only ever sees this type.
              │                               │
              │  + write(data)                │
              │  + read(): data               │
              └───────┬───────────────┬───────┘
                      │ implements    │ implements
                      │               │
   ┌──────────────────▼──────┐   ┌────▼──────────────────────────────┐
   │  CONCRETE COMPONENT      │   │      BASE DECORATOR               │
   │  InMemoryDataSource      │   │      DataSourceDecorator          │
   │                          │   │                                   │
   │  - buffer: Buffer        │   │  - wrappee: DataSource  ◀──────┐  │
   │  + write(data)  «real»   │   │  + write(data) → wrappee.write │  │
   │  + read(): data «real»   │   │  + read()      → wrappee.read  │  │
   └──────────────────────────┘   └────────────┬──────────────────┼──┘
                                               │ extends          │
                    ┌──────────────────────────┼──────────────┐   │
                    │                          │              │   │ HAS-A a Component
        ┌───────────▼─────────┐  ┌─────────────▼───────┐  ┌───▼───┴──────────┐
        │ CompressionDecorator│  │ EncryptionDecorator │  │ AuditLogDecorator│
        │                     │  │  - key: Buffer      │  │  - actor: string │
        │ + write(): gzip     │  │ + write(): encrypt  │  │ + write(): log   │
        │ + read():  gunzip   │  │ + read():  decrypt  │  │ + read():  log   │
        └─────────────────────┘  └─────────────────────┘  └──────────────────┘
             CONCRETE DECORATORS — each adds ONE responsibility, then delegates
```

The two arrows out of `DataSourceDecorator` are the heart of it: it **extends** `DataSource` (IS-A, so it can stand in for one) *and* **holds** a `DataSource` (HAS-A, so it can wrap one). That self-referential loop is what makes stacking work.

### Diagram 2: The runtime object graph — what `write()` actually does

```
CLIENT holds ONE reference. It thinks it has a plain DataSource.

  saveProfile(source, data)
        │
        │  source.write(bytes)
        ▼
┌──────────────────────────────────────────────────────────────────┐
│ AuditLogDecorator                                                 │
│   → log "alice wrote 340 B"                                       │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │ EncryptionDecorator                                       │   │
│   │   → AES-256 encrypt (128 B → 144 B)                       │   │
│   │   ┌───────────────────────────────────────────────────┐   │   │
│   │   │ CompressionDecorator                               │   │   │
│   │   │   → gzip (340 B → 128 B)                           │   │   │
│   │   │   ┌────────────────────────────────────────────┐   │   │   │
│   │   │   │ InMemoryDataSource   ◀── CONCRETE COMPONENT│   │   │   │
│   │   │   │   → actually stores the 144 bytes          │   │   │   │
│   │   │   └────────────────────────────────────────────┘   │   │   │
│   │   └───────────────────────────────────────────────────┘   │   │
│   └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘

  write:  data travels DOWN   ─────▶ log ─▶ encrypt ─▶ gzip ─▶ store
  read:   data travels back UP ◀──── log ◀─ decrypt ◀─ gunzip ◀─ store
                                             (each layer UNDOES its own step)
```

Draw this on a whiteboard as **nested boxes, not a flat list** — the nesting is the point. And say the sentence out loud: *"the client holds one reference and cannot tell how deep the nesting goes."*

---

## Real world examples

### 1. Node.js core — the stream pipeline

`zlib.createGzip()`, `crypto.createCipheriv()`, and `stream.Transform` in general are decorators over a stream. Every one of them is *itself* a `Duplex` stream, so it can be piped into another. This is why you can write `readStream.pipe(gzip).pipe(cipher).pipe(writeStream)` and why the order is load-bearing — compress before encrypt or you save nothing. Node deliberately did not create `GzipEncryptedFileWriteStream`; it gave you composable transforms. That decision is the 2ⁿ argument, made by the core team.

### 2. Java's `java.io` — the pattern's most famous case study (worth knowing even as a JS dev)

`new BufferedReader(new InputStreamReader(new FileInputStream("f.txt")))`. Three decorators. `FileInputStream` reads raw bytes; `InputStreamReader` decorates it to produce characters; `BufferedReader` decorates *that* to add buffering and `readLine()`. This is the example every interviewer over 30 has in their head. Naming it signals you know the pattern's history — but also note its most cited flaw: the constructor nesting is famously unreadable, which is the real-world cost of deep decorator stacks.

### 3. Express / Koa / Fastify — middleware as decoration

`app.use(helmet())` adds security headers. `app.use(compression())` gzips responses. `app.use(cors())` adds CORS headers. `app.use(morgan())` logs. None of them changes the `(req, res)` contract, all of them wrap what comes after, and you choose the order — and getting the order wrong is a classic production bug (put `compression()` *after* your response is already sent and it does nothing; put `helmet()` after an error handler and headers go missing). Same pattern, same order-sensitivity, same failure mode as the DataSource stack.

---

## Trade-offs

| Pros | What it buys you |
|------|------------------|
| **Kills subclass explosion** | n features → n classes, not 2ⁿ |
| **Runtime composition** | Decide the stack from config, per-request, per-tenant |
| **Single Responsibility** | Retry logic lives in exactly one class |
| **Open/Closed** | Add a feature by adding a class; never edit the working ones |
| **Order is explicit** | Encrypt-then-compress vs compress-then-encrypt is visible in the code |
| **Testable in isolation** | Test `RetryDecorator` against a fake component; no network needed |

| Cons | What it costs you |
|------|-------------------|
| **Many small classes** | Each trivial; the *set* is a lot to hold in your head |
| **Ugly construction** | `new A(new B(new C(new D(x))))` — Java's `BufferedReader` is the cautionary tale |
| **Hard to debug** | Stack traces run through every wrapper; "who added this header?" gets hard |
| **Identity breaks** | `instanceof ConcreteClass` and `===` fail against a decorated object |
| **Order is invisible in the type** | The compiler cannot tell you the stack is upside-down |
| **Removing a decorator mid-stack is hard** | You must rebuild the chain |
| **Interface must stay small** | A 20-method component makes every decorator 19 lines of boilerplate |

**The sweet spot:** Use Decorator for **optional, composable, cross-cutting behaviours** on a **narrow interface** (1–4 methods), stacked no more than ~4 deep, with the stack **built in one place** (a factory or a config-driven builder) so a reader can see the whole thing at once. If the behaviour is mandatory for everyone, put it in the base. If the interface is wide, use a higher-order function or a JS `Proxy` instead.

---

## Common interview questions on this topic

### Q1: "What's the difference between Decorator and Proxy? Their diagrams look identical."
**Hint:** Both share the wrapped object's interface — the difference is **intent**. A decorator *always* delegates and *adds* something (log it, retry it, compress it). A proxy decides *whether, when, or where* to delegate: it may refuse (access-control proxy), defer (lazy-loading/virtual proxy), or send the call elsewhere (remote proxy). Also: you hand a decorator its wrappee from outside and stack several; a proxy usually manages its own subject and there's one. Honest nuance worth saying out loud: a *caching* wrapper sits right on the line, and people legitimately call it either.

### Q2: "Why not just use inheritance to add these features?"
**Hint:** Do the arithmetic on the whiteboard: n independent optional features needs 2ⁿ − 1 subclasses (3 features = 7 classes; 5 = 31), the shared logic gets duplicated across them, ordering can't be expressed in a class name, and the combination is fixed at *compile* time so you can't choose it from config or per-request. Decorator: n classes, composed at runtime, order explicit.

### Q3: "Does the order of decorators matter? Give an example where it does."
**Hint:** Enormously. **Compress-then-encrypt** shrinks data; **encrypt-then-compress** does nothing at all, because ciphertext is high-entropy and incompressible — you've just wasted CPU. Another: `Cache(Retry(fetch))` means a cache hit costs nothing, while `Retry(Cache(fetch))` re-checks the cache on every retry attempt. And a `Cache` *outside* a `Log` means cache hits are never logged, so your dashboards go quiet. The stack is a design decision — write it once, in one factory, and comment why.

### Q4: "How would you do Decorator without classes, in idiomatic JS?"
**Hint:** Higher-order functions. `const withRetry = (fn) => async (...args) => { ... await fn(...args) ... }` — same signature in, same signature out, so they compose: `withLogging(withRetry(withCache(fetchUser)))`. Same pattern, no ceremony. Use HOFs when the component has one method; use classes when it has several (otherwise you'd have to wrap each method by hand). For a whole object, JS's built-in `Proxy` with a `get` trap can decorate every method generically.

### Q5: "Where is Decorator in the Node ecosystem?"
**Hint:** `stream.pipe()` chains (`readStream.pipe(gzip).pipe(cipher)` — every transform is a stream, so they stack); Express middleware; axios interceptors; winston format chains; and any `withX(fn)` higher-order function. Bonus point: mention `java.io`'s `new BufferedReader(new InputStreamReader(new FileInputStream(f)))` — the pattern's most famous example, and also the best illustration of its readability cost.

---

## Practice exercise

### The Notification Sender Stack

Build a decorator stack for sending notifications. Target: ~30–40 minutes. Produce one runnable `notify.js`.

**1. Component** — `class Notifier` with a single method:
```javascript
async send(userId, message) -> { ok: boolean, channel: string }
```

**2. Concrete Components** (two of them, so you can prove decorators are reusable across components):
- `EmailNotifier` — logs `"EMAIL to u_1: ..."`, returns `{ ok: true, channel: 'email' }`
- `SmsNotifier` — but make it **fail randomly ~50% of the time** by throwing an error with `err.retryable = true`

**3. Base Decorator** — `NotifierDecorator`, holding an inner `Notifier`, delegating `send` by default.

**4. Four Concrete Decorators**, each ~10 lines:
- `RetryNotifier(inner, maxAttempts)` — retries retryable errors with exponential backoff
- `RateLimitNotifier(inner, maxPerMinute)` — keeps timestamps in a `Map<userId, number[]>`; throws `RateLimitError` if a user exceeds the quota. (Ask yourself: is this still a *decorator*, or has it become a *proxy*? Write your answer as a comment — this is exactly the interview question.)
- `QuietHoursNotifier(inner, startHour, endHour)` — if the local hour is inside the quiet window, skip sending and return `{ ok: false, channel: 'suppressed' }`
- `MetricsNotifier(inner)` — counts `sent`, `failed`, `suppressed` on a public `stats` object

**5. A `main()` demo** that:
- Builds the stack `Metrics(QuietHours(RateLimit(Retry(SmsNotifier))))`
- Sends 10 notifications for the same user and prints the final `stats`
- Then **rebuilds the same four decorators in a different order** and explains, in a comment, what changed. (Specifically: what happens to your metrics when `Metrics` is *inside* `RateLimit` instead of outside? Do rate-limited sends still get counted?)

**Stretch goal:** rewrite the four decorators as higher-order functions (`withRetry`, `withRateLimit`, …) and compare which version you'd rather maintain.

---

## Quick reference cheat sheet

- **Decorator** wraps an object to add responsibilities at runtime, keeping the **exact same interface**.
- **The dual nature:** a decorator **IS-A** Component (so it's interchangeable) and **HAS-A** Component (so it can wrap). That loop is what makes stacking possible.
- **Five participants:** Component (interface) → Concrete Component (the real thing) → Base Decorator (delegates everything) → Concrete Decorators (add one thing each) → Client (holds a Component, knows nothing).
- **The killer argument:** n optional features need **2ⁿ − 1 subclasses** but only **n decorators**. 5 features = 31 classes vs 5.
- **Each decorator does one thing, then delegates.** If it doesn't delegate, it's not a decorator.
- **Symmetry rule:** whatever `write()` does on the way in, `read()` must undo on the way out.
- **Order is a design decision.** compress→encrypt ≫ encrypt→compress (ciphertext is incompressible). Cache-outside-retry ≫ retry-outside-cache.
- **Never add a public method** a decorator-only method disappears the moment someone wraps you.
- **Identity breaks:** `decorated instanceof ConcreteClass` is `false`. Don't rely on `===` or concrete `instanceof`.
- **Keep the interface narrow** (1–4 methods) and the stack shallow (≤ ~4) — deep stacks are debugging hell.
- **In JS, higher-order functions** (`withRetry(fn)`) are decorators without the ceremony. Use them for single-method components.
- **Decorator vs Proxy:** same interface for both — Decorator *adds behaviour and always delegates*; Proxy *controls access and may refuse or defer*.
- **Decorator vs Adapter:** Decorator **keeps** the interface (so it stacks); Adapter **changes** it (so it doesn't).
- **Node examples:** `stream.pipe()` transform chains, Express middleware, axios interceptors, `withX(fn)` HOFs.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [34 — Adapter Pattern](./34-pattern-adapter.md) — the wrapper that *changes* the interface, where Decorator preserves it |
| **Next** | [36 — Facade Pattern](./36-pattern-facade.md) — the wrapper that simplifies a whole subsystem behind one entry point |
| **Related** | [37 — Proxy Pattern](./37-pattern-proxy.md) — same interface as Decorator, but controls access rather than adding behaviour; the #1 confusion |
| **Related** | [26 — Composition Over Inheritance](./26-composition-over-inheritance.md) — Decorator is the sharpest possible argument for composition |
