# 37 — Proxy Pattern

## Category: LLD Patterns

---

## What is this?

A **Proxy** is a stand-in object that has the **exact same interface** as the real object, sits in front of it, and **controls access** to it. Callers can't tell the difference — they think they're talking to the real thing.

Think of a **celebrity's agent**. You want to book the celebrity for an event, so you call... the agent. The agent takes the same kinds of requests the celebrity would (`bookEvent()`, `checkAvailability()`), but decides whether to bother the celebrity at all: filtering out spam, checking your budget, remembering that they already told you "no" last week, or forwarding the request to a celebrity who's on another continent. From your side, you "talked to the celebrity." You actually talked to a proxy.

---

## Why does it matter?

The Proxy pattern is the answer to a question every real system eventually asks: **"How do I add lazy loading / auth / caching / logging / remote calls without changing the object that does the work, and without changing every caller?"**

Without Proxy, that logic leaks everywhere. Every caller has to remember to check `if (!image.loaded) image.load()`, or `if (user.isAdmin)`, or `if (cache.has(key))`. Miss it once and you have a bug. Duplicate it 40 times and you have a maintenance nightmare.

**At real work you use Proxy constantly, usually without naming it:**
- Every ORM lazy relation (`user.posts` fires a SQL query the first time you touch it) is a virtual proxy.
- Every gRPC/tRPC/RPC **client stub** is a remote proxy — that's literally what an RPC client *is*.
- Vue 3's reactivity system is built on JavaScript's native `Proxy` with `get`/`set` traps.
- `Object.freeze`-like immutability wrappers, permission wrappers, and memoization wrappers are all proxies.

**In interviews:** Proxy is the #1 pattern people confuse with Decorator and Adapter. Being able to say the one-line difference cleanly — *Adapter changes the interface, Decorator adds behaviour, Proxy controls access* — instantly signals that you actually understand structural patterns instead of having memorized names.

---

## The core idea — explained simply

### The Celebrity's Agent Analogy

You're an event organiser. You want to book a famous singer.

You call a number. A person picks up and says "Yes, this is the singer's office." Now watch what that person does with four different callers:

1. **A serious promoter calls.** The agent checks the calendar and says "Yes, she's free March 3rd." But here's the thing — the singer is asleep. The agent never woke her. The agent only *actually contacts the singer* when a real, confirmed booking needs her signature. That's a **virtual proxy**: don't create/load the expensive thing until you truly need it.

2. **A random stranger calls asking for her home address.** The agent says "No." The request never reaches the singer at all. That's a **protection proxy**: check permissions, then forward or reject.

3. **You call the agent in London; the singer is in Tokyo.** The agent takes your request, translates it into a fax/email/whatever, sends it across the world, waits for the reply, and reads the reply back to you as if she were standing there. That's a **remote proxy**: a local object standing in for a thing that lives on another machine.

4. **You ask "what's her standard fee?" for the tenth time this week.** The agent doesn't call Tokyo again. She just says the number from memory. That's a **caching / smart proxy**: remember the answer, skip the expensive work.

In all four cases, **you dialled the same number and asked the same questions.** The interface never changed. What changed is what happened *before* (and sometimes *instead of*) the real work.

| In the analogy | In code |
|---|---|
| The singer | **RealSubject** — the object that does the actual work |
| The agent | **Proxy** — same interface, controls access to RealSubject |
| "Can you sing on March 3rd?" | The **Subject interface** — the shared method signatures |
| The event organiser | The **Client** — doesn't know or care that it's talking to a proxy |
| Singer is asleep until needed | **Virtual proxy** (lazy initialisation) |
| Agent screens callers | **Protection proxy** (access control) |
| Singer is in Tokyo | **Remote proxy** (network stub) |
| Agent remembers the fee | **Caching / smart proxy** (memoization) |

The whole pattern rests on one rule: **the proxy and the real object must be interchangeable.** If the client has to know it's holding a proxy, you've failed.

---

## Key concepts inside this topic

### 1. The problem it solves — the code before the pattern

Here's an image gallery. Loading a high-res image costs ~5 MB of memory and ~200 ms of disk I/O. A gallery page shows 100 thumbnails but the user usually views only 3.

**Bad code — the caller manages laziness by hand:**

```javascript
// BAD: every caller has to remember the loading dance
class HighResImage {
  constructor(filename) {
    this.filename = filename;
    this.pixels = null; // 5MB when loaded
  }

  loadFromDisk() {
    console.log(`[disk] reading ${this.filename} (200ms, 5MB)...`);
    this.pixels = Buffer.alloc(5 * 1024 * 1024);
  }

  display() {
    console.log(`Displaying ${this.filename}`);
  }
}

// Now EVERY call site has to do this:
const img = new HighResImage('sunset.png');
if (!img.pixels) img.loadFromDisk(); // <-- forget this line = crash
img.display();

// And a gallery of 100 images...
const gallery = files.map(f => {
  const i = new HighResImage(f);
  i.loadFromDisk();   // 100 x 5MB = 500MB of RAM, for 3 images the user will look at
  return i;
});
```

Three separate failures here:
- **Leaked responsibility.** The *caller* knows about `pixels` and `loadFromDisk`. That's the image's business, not the caller's.
- **Wasted resources.** 500 MB allocated to show 3 images.
- **Fragile.** One call site that forgets the `if (!img.pixels)` check is a null-pointer bug in production.

The fix: give the caller an object that *looks exactly like* `HighResImage` but doesn't load anything until `display()` is actually called.

### 2. The structure — who plays which role

```
        ┌──────────────────────────┐
        │      «interface»          │
        │        Subject            │      Client depends ONLY on this.
        │  + request()              │      It never learns which of the
        └────────────┬──────────────┘      two below it is holding.
                     │ implements
        ┌────────────┴────────────────┐
        │                             │
┌───────▼──────────┐        ┌─────────▼────────────┐
│   RealSubject     │        │        Proxy          │
│                   │◀───────│  - realSubject        │
│  + request()      │  holds │  + request() {        │
│    (the real work)│  a ref │      // pre-checks    │
└───────────────────┘        │      real.request()   │
                             │      // post-actions  │
                             └───────────────────────┘
                                        ▲
                                        │ uses
                                 ┌──────┴──────┐
                                 │   Client     │
                                 └─────────────┘
```

| Participant | Job |
|---|---|
| **Subject** | The shared interface. In JS this is a base class or just an agreed shape (duck typing). |
| **RealSubject** | The object that actually does the work. Has no idea a proxy exists. |
| **Proxy** | Same interface. Holds a reference (or a *recipe* for creating one) to the RealSubject. Adds access control, then delegates. |
| **Client** | Talks to `Subject`. Cannot tell the difference. This is the point. |

The critical detail: the proxy often **creates the RealSubject itself**, lazily, rather than receiving it. That's what makes virtual proxies possible.

### 3. Flavour A — Virtual Proxy (lazy loading)

Create the expensive thing only on first real use.

```javascript
// The "interface" — in JS, a base class that documents the contract.
class Image {
  display() { throw new Error('Not implemented'); }
  getFilename() { throw new Error('Not implemented'); }
}

// RealSubject: expensive to build.
class HighResImage extends Image {
  constructor(filename) {
    super();
    this.filename = filename;
    // Constructing it IS the expensive part — unavoidable, which is the whole point.
    console.log(`  [disk] loading ${filename} — 200ms, 5MB allocated`);
    this.pixels = Buffer.alloc(5 * 1024 * 1024);
  }

  display() { console.log(`  [render] ${this.filename}`); }
  getFilename() { return this.filename; }
}

// Proxy: identical interface, defers construction.
class LazyImageProxy extends Image {
  constructor(filename) {
    super();
    this.filename = filename;
    this.real = null; // not built yet — costs ~0 bytes
  }

  // Cheap metadata answered WITHOUT touching the real object.
  // This is the hidden superpower of virtual proxies.
  getFilename() { return this.filename; }

  display() {
    if (this.real === null) {
      console.log(`  [proxy] first use of ${this.filename} — creating real image now`);
      this.real = new HighResImage(this.filename);
    }
    this.real.display();
  }
}
```

The gallery now costs almost nothing to build:

```javascript
const files = ['a.png', 'b.png', 'c.png', 'd.png', 'e.png'];
const gallery = files.map(f => new LazyImageProxy(f)); // 0 disk reads, ~0 MB

console.log(gallery.map(i => i.getFilename())); // still 0 disk reads

gallery[2].display(); // ONLY c.png hits the disk
gallery[2].display(); // second call reuses the loaded real object
```

Same trick, different resource: a database connection.

```javascript
class LazyDbConnection {
  constructor(config) {
    this.config = config;
    this.conn = null;
  }

  async #connect() {
    if (!this.conn) {
      console.log('[db] opening TCP connection + TLS handshake (~30ms)');
      this.conn = { query: async (sql) => [{ ok: true, sql }] }; // stand-in for pg.Client
    }
    return this.conn;
  }

  // A worker process that never runs a query never pays the 30ms or holds a socket.
  async query(sql) {
    const conn = await this.#connect();
    return conn.query(sql);
  }
}
```

### 4. Flavour B — Protection Proxy (authorization)

Check the caller's permissions *before* delegating. The real object stays blissfully unaware of auth.

```javascript
class DocumentStore {
  constructor() {
    this.docs = new Map([['salaries-2026', 'CEO: 900k, CTO: 750k...']]);
  }
  read(id) { return this.docs.get(id) ?? null; }
  write(id, content) { this.docs.set(id, content); return true; }
  delete(id) { return this.docs.delete(id); }
}

class ProtectedDocumentStore {
  // Same three methods. Same signatures. That is the contract.
  constructor(realStore, currentUser) {
    this.real = realStore;
    this.user = currentUser;
  }

  #require(role) {
    if (!this.user.roles.includes(role)) {
      // Fail loudly and identically for every method — one place to change.
      throw new Error(`AccessDenied: ${this.user.name} needs role "${role}"`);
    }
  }

  read(id)            { this.#require('reader'); return this.real.read(id); }
  write(id, content)  { this.#require('editor'); return this.real.write(id, content); }
  delete(id)          { this.#require('admin');  return this.real.delete(id); }
}
```

Why not just put the `if (user.isAdmin)` inside `DocumentStore.delete()`? Because then `DocumentStore` — a storage class — now depends on your auth system, your user model, and your role names. Swap auth providers and you're editing storage code. **Proxy keeps the policy (who may do what) separate from the mechanism (how bytes are stored).**

### 5. Flavour C — Remote Proxy (this is what an RPC client IS)

The real object lives in a different process, on a different machine. The proxy is a **local stub**: it looks like a normal object with normal methods, but each method serialises the call, ships it over the network, and deserialises the reply.

```javascript
// This class exists ONLY on the server.
class UserService {
  async getUser(id)  { return { id, name: 'Ada Lovelace' }; }
  async banUser(id)  { return { id, banned: true }; }
}

// This class exists on the CLIENT. Identical method names. No business logic at all.
class UserServiceStub {
  constructor(baseUrl) { this.baseUrl = baseUrl; }

  // Every method is the same three lines: marshal -> send -> unmarshal.
  async #call(method, args) {
    const res = await fetch(`${this.baseUrl}/rpc`, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ method, args }),
    });
    if (!res.ok) throw new Error(`RPC ${method} failed: ${res.status}`);
    const { result, error } = await res.json();
    if (error) throw new Error(error); // remote errors become local exceptions
    return result;
  }

  async getUser(id) { return this.#call('getUser', [id]); }
  async banUser(id) { return this.#call('banUser', [id]); }
}

// Client code is IDENTICAL whether `users` is the real service or the stub:
async function suspendAbuser(users, id) {
  const user = await users.getUser(id);
  await users.banUser(id);
  return `${user.name} banned`;
}
```

That's the entire idea behind gRPC, tRPC, Thrift, and Java RMI. `protoc` generates the stub class for you — but the generated code is exactly the shape above. **When an interviewer asks "where have you seen Proxy in the wild?", "every RPC client stub" is the strongest possible answer**, because it shows you know the pattern is load-bearing infrastructure, not a textbook curiosity.

The honest catch: a remote proxy *pretends* the call is local, but it isn't. It can be slow (1–100 ms vs 0.001 ms), it can fail with a timeout, and it can fail *after* the work succeeded. Never let a remote proxy hide that from the caller — surface timeouts and retries, don't swallow them.

### 6. Flavour D — Caching / Smart Proxy (memoize expensive calls)

```javascript
class PricingEngine {
  async quote(sku, qty) {
    console.log(`  [engine] recomputing quote for ${sku} x${qty} — 400ms of tax rules`);
    await new Promise(r => setTimeout(r, 400));
    return { sku, qty, total: qty * 19.99 };
  }
}

class CachingPricingProxy {
  constructor(real, ttlMs = 60_000) {
    this.real = real;
    this.ttlMs = ttlMs;
    this.cache = new Map();   // key -> { value, expiresAt }
    this.inFlight = new Map(); // key -> Promise  (prevents the "cache stampede")
  }

  async quote(sku, qty) {
    const key = `${sku}:${qty}`;
    const hit = this.cache.get(key);
    if (hit && hit.expiresAt > Date.now()) {
      console.log(`  [cache] HIT ${key}`);
      return hit.value;
    }

    // If 50 requests for the same key arrive while it's cold, only ONE goes downstream.
    if (this.inFlight.has(key)) return this.inFlight.get(key);

    const promise = this.real.quote(sku, qty).then(value => {
      this.cache.set(key, { value, expiresAt: Date.now() + this.ttlMs });
      this.inFlight.delete(key);
      return value;
    });
    this.inFlight.set(key, promise);
    return promise;
  }
}
```

The `inFlight` map is the detail that separates a toy from a real one. Recall from **[59 — Caching in Depth](./59-caching-in-depth.md)** that a *cache stampede* is when a cold key gets hit by N concurrent requests and all N do the expensive work. Collapsing them into one promise fixes it in four lines.

A **smart proxy** is the same shape but does bookkeeping other than caching: reference counting, lazy locking, access logging, metrics. `logger`, `timing`, and `retry` wrappers you've written are all smart proxies.

### 7. JavaScript's native `Proxy` — the pattern as a language feature

JS is one of the few mainstream languages where Proxy is **built into the runtime**. `new Proxy(target, handler)` returns an object that forwards every operation to `target`, unless the handler defines a **trap** for it.

```javascript
const target = { name: 'Ada', role: 'engineer' };

const logged = new Proxy(target, {
  // Trap: fires on EVERY property read, including ones that don't exist.
  get(obj, prop, receiver) {
    console.log(`[get] ${String(prop)}`);
    return Reflect.get(obj, prop, receiver); // always delegate via Reflect
  },
  // Trap: fires on EVERY property write. Return false (in strict mode) to throw.
  set(obj, prop, value, receiver) {
    if (prop === 'role' && value !== 'engineer' && value !== 'manager') {
      throw new TypeError(`Invalid role: ${value}`);
    }
    console.log(`[set] ${String(prop)} = ${value}`);
    return Reflect.set(obj, prop, value, receiver);
  },
  has(obj, prop) { return Reflect.has(obj, prop); },     // trap for `in`
  deleteProperty(obj, prop) { return Reflect.deleteProperty(obj, prop); },
});

logged.name;            // [get] name
logged.role = 'manager'; // [set] role = manager
// logged.role = 'pirate'; // TypeError
```

Two enormously important things are built on exactly this:

**a) Vue 3 reactivity.** When you write `const state = reactive({ count: 0 })`, Vue returns a `Proxy`. The `get` trap **tracks** which component is currently rendering and records "this component depends on `count`". The `set` trap **triggers** a re-render of every component that was recorded. Vue 2 had to use `Object.defineProperty` per-key, which is why it couldn't detect newly added properties or array index writes — the native `Proxy` fixed that class of bug outright. Simplified:

```javascript
let activeEffect = null;
const depsMap = new Map(); // prop -> Set<effect>

function reactive(obj) {
  return new Proxy(obj, {
    get(t, p, r) {
      if (activeEffect) {
        if (!depsMap.has(p)) depsMap.set(p, new Set());
        depsMap.get(p).add(activeEffect); // TRACK
      }
      return Reflect.get(t, p, r);
    },
    set(t, p, v, r) {
      const ok = Reflect.set(t, p, v, r);
      (depsMap.get(p) ?? []).forEach(fn => fn()); // TRIGGER
      return ok;
    },
  });
}

function watchEffect(fn) { activeEffect = fn; fn(); activeEffect = null; }

const state = reactive({ count: 0 });
watchEffect(() => console.log('count is', state.count)); // "count is 0"
state.count = 5;                                          // "count is 5" — automatically
```

That is a virtual proxy and a smart proxy fused together, and it is roughly 40 lines from being Vue's actual core.

**b) ORM lazy relations.** In an ORM (Prisma, TypeORM, Sequelize), `user.posts` shouldn't fire a `SELECT` when you load the user — most code paths never touch it. So the ORM hands you a proxy whose `get` trap notices you touched `posts` and only *then* issues the query:

```javascript
function withLazyRelations(row, relations) {
  return new Proxy(row, {
    get(target, prop, receiver) {
      // Already-loaded columns behave completely normally.
      if (prop in target) return Reflect.get(target, prop, receiver);
      // An unloaded relation? Load it now, cache it on the row, return it.
      if (prop in relations) {
        const value = relations[prop](target); // e.g. db.query('SELECT * FROM posts WHERE user_id=?')
        target[prop] = value;                  // memoize so the 2nd access is free
        return value;
      }
      return undefined;
    },
  });
}

const user = withLazyRelations(
  { id: 7, name: 'Ada' },
  { posts: (u) => { console.log(`[sql] SELECT * FROM posts WHERE user_id=${u.id}`); return [{ id: 1 }]; } }
);

console.log(user.name);  // "Ada"  — no SQL
console.log(user.posts); // [sql] ... then [{id:1}]
console.log(user.posts); // cached — no second query
```

This is also the source of the infamous **N+1 query problem**: loop over 100 users touching `user.posts` and the proxy dutifully fires 100 queries. The proxy is doing exactly what you told it to. That's the cost of invisible laziness.

### 8. When NOT to use it / how it gets abused

- **You have one call site.** If exactly one place needs the caching, just cache there. A proxy class to serve a single caller is ceremony.
- **The laziness is a lie.** Virtual proxies make a 200 ms disk read look like a property access. That's fine — until someone puts it in a tight loop. If the deferred work is *expensive and unbounded*, an explicit `await load()` is more honest than magic.
- **Proxy chains.** `CachingProxy(LoggingProxy(RetryProxy(AuthProxy(realThing))))` is technically elegant and operationally a nightmare — a stack trace goes through four wrappers and nobody can tell which layer swallowed the error. Three proxies deep, stop and reconsider.
- **Native `Proxy` on hot paths.** JS `Proxy` traps are meaningfully slower than direct property access (roughly an order of magnitude for a `get`). Fine for a config object or a reactive store; not fine inside a per-frame render loop.
- **Native `Proxy` breaks identity assumptions.** `proxy !== target`, `instanceof` can surprise you, and private class fields (`#x`) throw when accessed through a proxy of a class instance. Always use `Reflect.*` inside traps rather than touching `obj[prop]` directly, or you'll break `this` binding on getters.

### 9. Related patterns and how they differ — the #1 interview follow-up

All three of these wrap an object. The wrapping is the *only* thing they have in common.

| Pattern | Interface vs the wrapped object | What it does | One-liner |
|---|---|---|---|
| **Proxy** | **Same** | Controls **access** — decides *whether/when* the real call happens | "You can't/needn't talk to it directly" |
| **Decorator** | **Same** | **Adds behaviour** around a call that always happens | "…and also do this" |
| **Adapter** | **Different** | **Translates** an incompatible interface | "Make the plug fit the socket" |
| **Facade** | **New, simpler** | Hides a **whole subsystem** behind one method | "One button for ten steps" |

Sharper distinctions:

- **Proxy vs Decorator.** Structurally near-identical; the *intent* differs. A Decorator's job is to enrich the result (`compress(encrypt(stream))` — both always run). A Proxy's job is to guard the door (it may *never* call the real object — a protection proxy that rejects, a cache proxy that hits). Also: a decorator is normally handed the object to wrap; a proxy often **creates the object itself** (that's how lazy loading works). And decorators are designed to stack; proxies usually aren't.
- **Proxy vs Adapter.** Adapter exists *because* the interfaces don't match — you have `StripeSDK.charge()` and your code wants `PaymentGateway.pay()`. Proxy exists precisely *because* they DO match. If you changed the method names, you wrote an Adapter.
- **Proxy vs Facade.** A Facade fronts many objects with a new, simpler interface. A Proxy fronts one object with the identical interface.
- **Related in spirit:** **Flyweight** (a shared object referenced by many) and **Singleton** (a proxy over global access to one instance) both use proxy-ish indirection.

---

## Visual / Diagram description

### Diagram 1: The four flavours, same shape

```
            CLIENT
              │  calls display() / read() / getUser() / quote()
              │  (has NO idea which box is below)
              ▼
   ┌──────────────────────────────┐
   │           PROXY               │  same interface as RealSubject
   │  ┌────────────────────────┐   │
   │  │  VIRTUAL:  is it built? │──┼── no ──▶ build it, THEN delegate
   │  │  PROTECTION: may they?  │──┼── no ──▶ throw AccessDenied  ✗ never delegates
   │  │  CACHING:  cache hit?   │──┼── yes ─▶ return cached       ✗ never delegates
   │  │  REMOTE:   serialize    │──┼─────────▶ network hop
   │  └────────────────────────┘   │
   └───────────────┬───────────────┘
                   │ delegate (maybe)
                   ▼
        ┌──────────────────────┐
        │     REAL SUBJECT      │  does the actual work.
        │  (expensive / remote  │  Knows NOTHING about the proxy.
        │   / privileged)       │
        └──────────────────────┘
```

The two `✗` arrows are the heart of the pattern: **a proxy is allowed to not call the real object at all.** A decorator never has that option.

### Diagram 2: Remote proxy across the network boundary

```
   CLIENT MACHINE                    │           SERVER MACHINE
                                     │
  ┌──────────────┐                   │        ┌───────────────────┐
  │ Application  │                   │        │  RPC Dispatcher    │
  │  code        │                   │        │  (unmarshals)      │
  └──────┬───────┘                   │        └─────────┬─────────┘
         │ users.getUser(7)          │                  │ getUser(7)
         ▼                           │                  ▼
  ┌──────────────────┐   HTTP/gRPC   │        ┌───────────────────┐
  │ UserServiceStub  │══════════════▶│═══════▶│   UserService      │
  │   (the PROXY)    │◀══════════════│◀═══════│  (the REAL one)    │
  │ marshal/unmarshal│   {name:"Ada"}│        │   hits the DB      │
  └──────────────────┘               │        └───────────────────┘
                                     │
      Looks like a local call.   NETWORK BOUNDARY   Is not a local call.
```

Draw this on a whiteboard and label the boundary. The interview point is: **the stub is a proxy, and the whole value of RPC is that the client code doesn't change when the callee moves to another machine.**

### Diagram 3: The lazy-load timeline

```
 t=0ms    new LazyImageProxy('sunset.png')      → 0 bytes,  0 ms
 t=1ms    proxy.getFilename()                   → 0 bytes,  0 ms   (answered by proxy)
 t=2ms    ... user never scrolls to it ...      → 0 bytes,  0 ms
 ─────────────────────────────────────────────────────────────────
 t=900ms  proxy.display()   ── FIRST real use ──▶ new HighResImage()
                                                 → 5 MB,  200 ms
 t=1100ms proxy.display()   ── cached real obj ──▶ 0 extra bytes, 0 ms
```

---

## Real world examples

### 1. gRPC / Protocol Buffers — the generated client stub

When you compile a `.proto` file, `protoc` generates a client class with one method per RPC defined in the service. Calling `client.GetUser({id: 7})` looks like a method call, but the generated code serialises the arguments to protobuf bytes, writes them onto an HTTP/2 stream, waits, and parses the response. This is a textbook remote proxy — the generated stub exists solely so that your application code can pretend a network service is a local object. Google's entire internal service mesh is built on this indirection.

### 2. Vue 3 — reactivity via native `Proxy`

Vue 3 rewrote its reactivity core around `new Proxy()`. `reactive(obj)` returns a proxy whose `get` trap records a dependency between the currently-rendering component and the property being read, and whose `set` trap re-runs every effect that depended on that property. This is publicly documented in Vue's "Reactivity in Depth" guide, and it's the reason Vue 3 (unlike Vue 2's `Object.defineProperty` approach) can detect property *additions*, deletions, and direct array index assignment. MobX 6 uses the same native-`Proxy` technique.

### 3. Nginx / Envoy as a reverse proxy — the pattern at the infrastructure layer

Proxy isn't only an object-level idea. A reverse proxy speaks the **same protocol** (HTTP) as the backend it fronts — the client sends the identical request either way. What it adds is access control: TLS termination, rate limiting, auth checks, caching of static responses, and routing to whichever backend is healthy. Same interface, controlled access, callers unaware. When you understand the object pattern, **[57 — Reverse Proxy](./57-reverse-proxy.md)** stops being a new concept and becomes the same concept at a different altitude.

### 4. ORMs — Hibernate, Prisma, Sequelize

Hibernate famously returns *proxy subclasses* of your entities; touching an unloaded association triggers the SQL. Modern JS ORMs do the equivalent with lazy accessors or native `Proxy`. The upside is that you never load data you don't use; the downside is the N+1 query problem, which is entirely a side effect of laziness being invisible.

---

## Trade-offs

| Pro | What you gain |
|---|---|
| **Open/Closed** | Add caching/auth/logging without touching the real class or any caller |
| **Lazy cost** | Expensive resources (5 MB images, DB sockets, 30 ms handshakes) are paid for only when used |
| **Separation of concerns** | Storage code doesn't know about auth; business code doesn't know about the network |
| **Transparent substitution** | Swap real ↔ proxy in tests, in dev, per-environment — the client never changes |
| **Location transparency** | A remote proxy lets a service move machines without a client rewrite |

| Con | What it costs |
|---|---|
| **Indirection** | One more class, one more stack frame, one more place a bug can hide |
| **Hidden latency** | `user.posts` looks free, actually costs a SQL round trip → N+1 queries |
| **Debugging pain** | Stack traces get longer; deep proxy chains obscure who did what |
| **Native `Proxy` perf** | Trap dispatch is ~10x slower than a plain property read; avoid in hot loops |
| **Response-time surprises** | Lazy first-call is slow, subsequent calls fast → confusing p50 vs p99 latency profiles |
| **Interface drift** | Add a method to RealSubject and forget the proxy → silent breakage (real risk in JS, no compiler to catch it) |

**Rule of thumb:** Reach for Proxy when the *access* to an object needs a policy — *when* (lazy), *whether* (auth), *from where* (remote), or *how often* (cache) — and that policy has no business living inside the object itself. If you're adding new *capability* rather than gating access, you want a Decorator. If you're changing the method names, you want an Adapter.

---

## Common interview questions on this topic

### Q1: "What's the difference between Proxy and Decorator? They look identical."
**Hint:** Structurally they are nearly identical — same interface, hold a reference, delegate. The difference is **intent and control**. A Decorator *always* calls the wrapped object and adds behaviour around the result ("and also do this"). A Proxy *may refuse to call it at all* — a protection proxy that throws, a cache proxy that returns a hit, a virtual proxy that hasn't even created the object yet. Second tell: a decorator is *given* the object it wraps; a proxy often *creates* it. Third: decorators are designed to stack; proxies usually aren't.

### Q2: "Give me a real-world Proxy you've actually used."
**Hint:** The strongest answers: (1) any RPC/gRPC client stub — it IS a remote proxy, that's the whole reason generated clients exist; (2) ORM lazy relations — `user.posts` fires SQL on first touch, which is also where N+1 comes from; (3) Vue 3 / MobX reactivity built on native JS `Proxy` `get`/`set` traps; (4) a memoization wrapper around an expensive pricing or geocoding call. Say *which flavour* it is — virtual, protection, remote, or caching — that's what they're listening for.

### Q3: "How does JavaScript's native `Proxy` work, and what are traps?"
**Hint:** `new Proxy(target, handler)` returns an exotic object that forwards every fundamental operation to `target` unless the handler defines a **trap** for that operation. Traps include `get`, `set`, `has` (for `in`), `deleteProperty`, `apply` (for function calls), and `construct` (for `new`). Inside a trap you should always delegate via `Reflect.get(...)`/`Reflect.set(...)` so getters and `this`-binding behave correctly. Mention the cost: traps are roughly an order of magnitude slower than direct property access.

### Q4: "Isn't a virtual proxy just lazy initialization? Why bother with a class?"
**Hint:** It is lazy initialization — the *pattern* part is that the laziness is invisible to the caller because the proxy shares the subject's interface. You could write `if (!this.x) this.x = build()` inside the real class, but then (a) the real class carries construction logic it shouldn't own, and (b) you can't have a *cheap* object that answers metadata questions (`getFilename()`) without the expensive one existing at all. The proxy lets you have a real, usable object that has cost you nothing yet.

### Q5: "When does Proxy become an anti-pattern?"
**Hint:** Three answers. (1) **Hidden cost**: making a network call or a disk read look like a property access is how you get N+1 queries and mystery p99 spikes. (2) **Proxy chains**: four layers of wrapping means a stack trace nobody can read and no one knows which layer ate the error. (3) **Interface drift in dynamic languages**: JS has no compiler to tell you the proxy is missing a method the real subject grew last week. Also mention that a proxy serving exactly one call site is just ceremony — inline the logic.

### Q6: "How would you build a rate limiter using this pattern?"
**Hint:** It's a protection proxy where the "permission" is a token bucket instead of a role. Same interface as the real API client; each method first calls `this.#consumeToken()`, which either passes, delays, or throws `RateLimitExceeded`. The real API client stays completely ignorant of rate limits — which means you can reuse it against a different provider with different limits by swapping only the proxy.

---

## Practice exercise

### Build a four-layer proxy around a weather API

Create `weather.js` in Node. You'll build one `RealSubject` and three proxies over it, all with the **identical** interface `async getForecast(city)`.

**Step 1 — The RealSubject.** Write `WeatherApi` with `async getForecast(city)`. Simulate the real thing: `await new Promise(r => setTimeout(r, 300))`, log `[api] HTTP call for ${city}`, and return `{ city, tempC: 20 + Math.floor(Math.random() * 10) }`. Keep a public counter `this.callCount` so you can *prove* the proxies work.

**Step 2 — `CachingWeatherProxy`.** Same `getForecast(city)`. Cache by city with a 5-second TTL. Include the `inFlight` promise-collapsing trick so 10 simultaneous requests for `'delhi'` cause exactly **one** API call.

**Step 3 — `ProtectionWeatherProxy`.** Constructor takes `(real, user)`. Only allow `getForecast` if `user.plan === 'pro'` or the city is in a free-tier allowlist `['delhi','london']`; otherwise `throw new Error('UpgradeRequired')`.

**Step 4 — `LoggingWeatherProxy`** (a smart proxy). Log the city, the duration in ms, and whether it threw.

**Step 5 — Compose and demo.** In a `main()`:
```javascript
const real  = new WeatherApi();
const api   = new LoggingWeatherProxy(
                new ProtectionWeatherProxy(
                  new CachingWeatherProxy(real),
                  { name: 'sam', plan: 'free' }));

await Promise.all([api.getForecast('delhi'), api.getForecast('delhi'), api.getForecast('delhi')]);
await api.getForecast('delhi');            // should be a cache HIT
try { await api.getForecast('tokyo'); }    // should throw UpgradeRequired
catch (e) { console.log('blocked:', e.message); }
console.log('real API calls made:', real.callCount); // MUST print 1
```

**Then answer, in a comment at the bottom of the file:** the composition order above puts Protection *outside* Caching. What breaks if you swap them — Caching outside Protection? (Think about a free user who benefits from a cache entry populated by a pro user's request for a paid city.) This ordering question is exactly the kind of thing a good interviewer will probe.

**Bonus:** rewrite `CachingWeatherProxy` using JavaScript's native `Proxy` with an `apply`/`get` trap instead of a hand-written class, and note which version you'd rather debug at 3 a.m.

---

## Quick reference cheat sheet

- **Proxy = same interface + controls access.** If the interface changed, it's an Adapter, not a Proxy.
- **The four flavours: Virtual** (lazy-load the expensive thing), **Protection** (auth check before delegating), **Remote** (local stub for a remote object), **Caching/Smart** (memoize + bookkeeping).
- **A proxy may never call the real object.** A decorator always does. That's the cleanest one-line distinction.
- **The proxy often *creates* the RealSubject**, lazily — a decorator is handed the object it wraps.
- **An RPC client stub IS a remote proxy.** gRPC's generated client is the pattern, shipped at planetary scale.
- **JS has a native `Proxy`:** `new Proxy(target, handler)` with traps — `get`, `set`, `has`, `deleteProperty`, `apply`, `construct`.
- **Always delegate from a trap via `Reflect.get/set(...)`** so getters and `this`-binding keep working.
- **Vue 3 reactivity = `get` trap TRACKS dependencies, `set` trap TRIGGERS re-renders.** MobX 6 does the same.
- **ORM lazy relations are virtual proxies** — and are also the direct cause of the **N+1 query problem**.
- **Add promise-collapsing (`inFlight` map) to any caching proxy** or a cold key gets stampeded by N concurrent callers.
- **Cost of Proxy:** indirection, hidden latency, longer stack traces, and (native `Proxy`) ~10x slower property access.
- **Don't chain 4 proxies deep.** Nobody can debug `Caching(Logging(Retry(Auth(real))))`.
- **Reverse proxies (Nginx, Envoy) are this exact pattern** at the network layer: same protocol, controlled access.
- **Interview trio to memorize:** Adapter changes the interface. Decorator adds behaviour. **Proxy controls access.**

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [36 — Facade Pattern](./36-pattern-facade.md) — the other "wrap it" structural pattern; Facade simplifies a subsystem, Proxy guards one object |
| **Next** | [38 — Composite Pattern](./38-pattern-composite.md) — treat single objects and groups of objects uniformly through one shared interface |
| **Related** | [35 — Decorator Pattern](./35-pattern-decorator.md) — same structure, opposite intent; the #1 pattern Proxy gets confused with |
| **Related** | [34 — Adapter Pattern](./34-pattern-adapter.md) — changes the interface, whereas Proxy deliberately keeps it identical |
| **Related** | [57 — Reverse Proxy](./57-reverse-proxy.md) — the same idea one altitude up, at the network boundary |
