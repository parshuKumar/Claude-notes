# 27 — Abstraction and Interfaces in Design
## Category: LLD Fundamentals

---

## What is this?

**Abstraction** is deciding what to *show* and what to *hide*. You expose a small, stable set of things a caller needs (`charge(amount)`), and you hide the messy details behind it (HTTP retries, API keys, currency conversion, vendor-specific error codes).

An **interface** is the written-down promise of what's exposed: the method names, their arguments, what they return, and what they throw. It's a contract, not code.

Think of a **wall socket**. The socket is the interface: two holes, 230V, that's the whole contract. Behind it could be a coal plant, a solar farm, or a diesel generator. Your laptop charger doesn't know and doesn't care. You can swap the entire power grid without changing a single appliance in the country — because everyone agreed on the shape of the hole.

---

## Why does it matter?

**If you get abstraction wrong, every change becomes a rewrite.** You wire Stripe's SDK directly into your `OrderService`, your `RefundJob`, your `AdminController`, and your webhook handler. Eighteen months later the business signs a deal with Razorpay for the Indian market. Now Stripe's `paymentIntent.id` is in your database schema, in your logs, in your Slack alerts, in your tests. What should have been "write one new class" becomes a six-week migration with an incident.

**If you get it right**, the blast radius of change shrinks to one file. New payment provider? New class. New notification channel? New class. Nothing else moves.

**Interview angle:** Almost every LLD interview is secretly an abstraction test. "Design a parking lot" is really "can you see that `Vehicle` is the abstraction and `Car`/`Bike`/`Truck` are details?" The candidates who fail usually fail by writing one giant class with a `switch` on a string type. The ones who pass name the role, program to it, and let implementations vary. Interviewers listen for the phrase "what varies here?"

**Real-work angle:** Abstraction is how you keep a codebase alive past year two. It's also how you make code testable — an interface is a seam, and a seam is where you insert a fake. If you can't test your `OrderService` without a network call to Stripe, you don't have an abstraction, you have a dependency you're stuck with.

The flip side, which most tutorials never say out loud: **abstraction is not free.** Every layer you add is a layer someone has to read through at 2am. This doc teaches both halves — how to find the right abstraction, and when to refuse to build one.

---

## The core idea — explained simply

### The Restaurant Waiter Analogy

You sit down at a restaurant. You say: **"One margherita pizza, please."**

That's the entire interface. Four words. You did not say:

- "Preheat the stone oven to 450°C"
- "Stretch 250g of 65%-hydration dough by hand"
- "The tomatoes should be San Marzano"
- "Fire it for 90 seconds, rotating once"

You don't know any of that. You don't *want* to know any of that. And here's the crucial part: **the kitchen can change everything about how the pizza is made and you never find out.** They swap the wood oven for a gas one. They hire a new pizzaiolo. They change the flour supplier. Your order — `orderPizza('margherita')` — never changes.

Now flip it. Imagine the waiter comes back and says: *"Sorry, we're out of San Marzano tomatoes, so the acidity balance will require you to re-specify your basil quantity."* Suddenly a kitchen detail has **leaked** through the interface and landed in your lap. You, the customer, now have to understand tomato chemistry. That's a **leaky abstraction**, and we'll come back to it — because in software it happens constantly.

Here's the mapping:

| Restaurant | Software | What it teaches |
|---|---|---|
| "One margherita, please" | `channel.send(user, message)` | The interface — small, stable, in the caller's language |
| The waiter | The abstract base class / contract | Guarantees the shape of the request |
| The kitchen | The concrete implementation | Free to change internally, forever |
| Menu item names | Method names — name the **role**, not the recipe | "Margherita", not "ThatThingGiuseppeMakes" |
| Swapping the oven | Swapping Stripe → Razorpay | Zero change for the caller |
| "Re-specify your basil" | ORM N+1 query problem | A leak — the detail escaped |
| Waiter, host, sommelier, busboy | Too many layers | Cost of abstraction — now who do I ask? |

The whole discipline of abstraction is this one question, asked over and over:

> **What is likely to change, and can I put a wall around it?**

That's it. Find what varies. Wrap it. Give the wrapper a name that describes the *job*, not the *worker doing it today*.

---

## Key concepts inside this topic

### 1. Finding the abstraction: identify what VARIES and encapsulate it

This is the single most useful sentence in object-oriented design, from the Gang of Four book: **"Encapsulate what varies."**

Practically: look at your code, find the parts that will be different next month, next customer, or next region — and pull *those* behind a boundary.

Here's the smell. A function with a `switch` on a type string, where each branch does something structurally similar:

```js
// BAD — every new provider means editing this file. The variation is naked.
async function sendNotification(type, user, message) {
  if (type === 'email') {
    const transport = nodemailer.createTransport({ host: 'smtp.sendgrid.net', port: 587 });
    await transport.sendMail({ to: user.email, subject: 'Notice', text: message });
  } else if (type === 'sms') {
    await twilioClient.messages.create({ to: user.phone, from: '+15551234', body: message });
  } else if (type === 'push') {
    await fetch('https://fcm.googleapis.com/fcm/send', {
      method: 'POST',
      headers: { Authorization: `key=${process.env.FCM_KEY}` },
      body: JSON.stringify({ to: user.deviceToken, notification: { body: message } }),
    });
  }
  // ...and in six months: else if (type === 'slack') { ... }
  // and 'whatsapp', and 'webpush', and 'in_app'...
}
```

What varies? **How a message gets delivered.** What stays the same? **That a message gets delivered to a user.** So the abstraction writes itself:

```js
// The role: "something that can deliver a message to a user."
// Not "an email sender". Not "a Twilio thing". A CHANNEL.
class NotificationChannel {
  async send(user, message) {
    throw new Error('NotificationChannel.send() must be implemented');
  }
}
```

The `if/else` chain didn't disappear — it moved to **one place** (a factory or a registry), and the *behaviour* now lives in separate, independently testable classes. We build the full version in section 6.

**The test:** if adding a new case means editing an existing `if/else` in three different files, you have missed an abstraction.

---

### 2. Name the ROLE, not the implementation

A bad name is a permanent tax. Names outlive vendors.

| Bad name | Why it's poison | Good name |
|---|---|---|
| `StripeWrapper` | Locks the concept to a vendor. When you add Razorpay, do you write `RazorpayWrapper` and a `WrapperChooser`? | `PaymentGateway` |
| `RedisHelper` | "Helper" means nothing. And now every caller *thinks in Redis*. | `Cache` |
| `S3Uploader` | You will one day be on GCS or R2. | `FileStore` / `BlobStore` |
| `KafkaProducerService` | Callers now `await kafka.produce()` — that word is in 200 files. | `EventPublisher` |
| `MongoUserDao` | Fine as a *class*, terrible as the *type callers depend on*. | Interface: `UserRepository`, impl: `MongoUserRepository` |
| `SendGridMailer` | Same trap. | `NotificationChannel` |

The rule: **the interface name should survive replacing the vendor.** Say the name out loud and ask, "If we switched providers tomorrow, would this name become a lie?" If yes, rename it now — it is cheap today and expensive in a year.

A corollary that catches people: **don't leak vendor concepts through the method signatures either.**

```js
// BAD — the name is fine, the signature is not. `paymentIntentId` is a Stripe word.
class PaymentGateway {
  async charge(amount, currency, paymentIntentId) {}  // Razorpay has no such thing
}

// GOOD — a neutral vocabulary the caller owns.
class PaymentGateway {
  /** @returns {Promise<{ transactionId: string, status: 'succeeded'|'failed' }>} */
  async charge({ amountMinor, currency, customerRef }) {}
}
```

The abstraction is only as good as its leakiest parameter.

---

### 3. Levels of abstraction: don't mix altitudes

A function should read like it was written by one person thinking at **one level of detail**. This is sometimes called the *Single Level of Abstraction Principle*.

Here's the smell — a function flying at 30,000 feet and 3 feet in the same paragraph:

```js
// BAD — "mixing altitudes". Business language and byte-level plumbing in one body.
async function checkout(cart, user) {
  const total = cart.items.reduce((s, i) => s + i.price * i.qty, 0);   // high level (business)

  const socket = net.connect(443, 'api.stripe.com');                    // LOW level (transport!)
  socket.setNoDelay(true);
  const payload = Buffer.from(JSON.stringify({ amount: total * 100 }));
  socket.write(`POST /v1/charges HTTP/1.1\r\nContent-Length: ${payload.length}\r\n\r\n`);
  socket.write(payload);
  // ...manual response parsing...

  await sendReceiptEmail(user, total);                                  // high level (business)
  await db.orders.insert({ userId: user.id, total, status: 'paid' });   // mid level
}
```

Reading this hurts because your brain has to change gears three times. Is `checkout` about *business rules* or about *TCP*? It's about both, which means it's testable as neither.

```js
// GOOD — every line in checkout() is at the same altitude: business steps.
async function checkout(cart, user) {
  const total = calculateTotal(cart);
  const receipt = await paymentGateway.charge({ amountMinor: total, currency: 'INR', customerRef: user.id });
  await orderRepository.save(new Order({ userId: user.id, total, transactionId: receipt.transactionId }));
  await notifier.notify(user, `Payment of ₹${total / 100} received.`);
  return receipt;
}
```

Now `checkout` reads like the description of checkout. The socket lives inside `StripePaymentGateway`, which is the only place that's allowed to know Stripe exists.

**How to spot it in your own code:** read a function top to bottom and, for each line, write one word — `business`, `domain`, `io`, `bytes`. If the words jump around, split the function.

---

### 4. Leaky abstractions (Joel Spolsky's Law)

> **"All non-trivial abstractions, to some degree, are leaky."** — Joel Spolsky, 2002

This is the honest half of the story. Abstractions hide details **until they can't**, and the moment they fail, you need to know exactly the thing you were promised you'd never need to know.

**Leak #1 — the ORM and the N+1 query problem.**

An ORM (Object-Relational Mapper — a library that lets you use database rows as if they were plain objects) is a *beautiful* abstraction. `user.posts` — look, no SQL! Until:

```js
// Looks innocent. It is not.
const users = await User.findAll();          // 1 query:  SELECT * FROM users;

for (const user of users) {
  const posts = await user.getPosts();       // N queries: SELECT * FROM posts WHERE user_id = ?
  console.log(user.name, posts.length);      //            ...once per user. 1,000 users = 1,001 queries.
}
```

At 1,000 users and 2ms per round-trip, that's **~2 seconds** of pure database latency for a page that should take 10ms. The fix requires you to know precisely the thing the ORM promised to hide:

```js
// The fix is SQL knowledge wearing an ORM costume: one JOIN, one round-trip.
const users = await User.findAll({ include: [Post] });   // 1 query with a LEFT JOIN
```

The abstraction leaked. You bought "you never need SQL" and the bill came due as "you need SQL *and* you need to know how this library generates it." That's strictly more knowledge than you started with. This is the classic case, and every backend engineer meets it.

**Leak #2 — TCP hides packet loss, until latency spikes.**

TCP's promise: "a reliable, ordered byte stream." Beautiful. You `socket.write()`, the bytes arrive, in order, guaranteed. Packets don't get lost, because TCP retransmits them for you.

Except the retransmission takes *time*. On a flaky mobile network, TCP's guarantee holds — but your p99 latency jumps from 40ms to 3 seconds, because underneath, TCP is silently retransmitting and backing off exponentially. Your "reliable stream" abstraction is intact; your users are staring at a spinner. To debug it you must drop through the abstraction and think about packets, RTT, congestion windows, and head-of-line blocking — the exact layer TCP existed to spare you.

**Leak #3 — the `Cache` that isn't transparent.** `cache.get(key)` looks like a fast map lookup. But it's a network call. It can time out. It can return stale data. The moment you have a cache stampede (a thousand requests all missing the same key at once and all hammering the DB), the "it's just a fast map" story dies.

**What to do about it:** you can't prevent leaks. You can only:
1. **Choose your leaks.** Prefer abstractions whose failure modes you're willing to learn.
2. **Learn one layer down.** The rule of thumb: *be fluent in the abstraction you use, and literate in the one beneath it.* Use an ORM, but know SQL. Use HTTP, but know TCP.
3. **Give the abstraction an escape hatch.** Good ORMs let you drop to raw SQL. Good HTTP clients expose the underlying socket options. An abstraction with no escape hatch turns a leak into a wall.

---

### 5. The genuine COST of abstraction

Tutorials sell abstraction like it's free. It isn't. Here's the bill:

**Indirection.** To answer "what actually happens when I call `notifier.notify()`?", you now open `NotificationService`, then `NotificationChannel`, then the factory that decided which subclass got injected, then finally `SmsChannel`. Four files to read one behaviour. That's four chances to lose the thread — especially for the new hire, and especially at 2am during an incident.

**More files, more names.** Each abstraction is a new noun the team must learn and agree on. Ten abstractions is a vocabulary. Fifty is a dialect nobody speaks.

**Harder to trace.** "Jump to definition" lands you on `async send(user, message) { throw new Error('Not implemented'); }` — technically correct, completely useless. Stack traces get taller. Debuggers step through wrappers.

**The wrong abstraction is worse than none.** Sandi Metz put it best: *"Duplication is far cheaper than the wrong abstraction."* If you guess wrong about what varies, you get a contract that fights every new requirement — and people start bolting on `options.stripeSpecificHack = true` flags to sneak past it. Now you have the cost of abstraction *and* the coupling.

**So when do you abstract?**

> **The Rule of Three.** First time: write it inline. Second time: notice the duplication, wince, but *wait* — two points can fit any line. Third time: now you can see the actual shape of the variation. **Abstract on the third concrete need, not the first imagined one.**

Day-one abstraction is a guess about the future. Day-ninety abstraction is a *fact about the past* — you have three real implementations in front of you, and their real differences tell you exactly where the seam goes. Speculative generality ("we might need a plugin system someday") is how codebases get a `AbstractStrategyFactoryProvider` that only ever has one implementation.

---

### 6. How to express an interface in JavaScript (which has none)

Java has `interface`. C# has `interface`. Go has `interface`. **JavaScript has nothing.** There is no keyword, no compile-time check, no enforcement. So we simulate it. Five ways, each with a real trade-off.

**(a) Abstract base class that throws.** The most common, most explicit approach.

```js
// The contract, enforced at runtime.
export class NotificationChannel {
  constructor() {
    // Prevent anyone from instantiating the abstract class itself.
    if (new.target === NotificationChannel) {
      throw new TypeError('NotificationChannel is abstract and cannot be instantiated directly');
    }
  }

  /**
   * @param {{id: string, email?: string, phone?: string}} user
   * @param {string} message
   * @returns {Promise<{channel: string, ok: boolean}>}
   */
  async send(user, message) {
    throw new Error(`${this.constructor.name} must implement send()`);
  }

  /** Can this channel reach this particular user? (No phone number → no SMS.) */
  supports(user) {
    throw new Error(`${this.constructor.name} must implement supports()`);
  }
}
```

Pro: self-documenting, fails loudly, gives you a place to put shared code. Con: the failure happens at *runtime*, not at write-time — you find out in production if you forgot a method and never tested that path.

**(b) Duck typing.** "If it walks like a duck and quacks like a duck, it's a duck." No base class at all — just pass anything with a `send` method.

```js
// No inheritance anywhere. The "interface" is purely social convention.
const consoleChannel = {
  async send(user, message) { console.log(`[console] → ${user.id}: ${message}`); return { channel: 'console', ok: true }; },
  supports() { return true; },
};

await notificationService.notify(user, 'Hi', [consoleChannel]);  // just works
```

Pro: zero ceremony, trivially easy to write a test fake. Con: nothing is written down. A typo (`snd`) fails only when that line executes. This is how most idiomatic JS actually works, and it's fine for small, well-tested surfaces.

**(c) Plain object / function contracts.** An "interface" can just be a function signature. This is deeply idiomatic in Node.

```js
// The contract IS the type: (user, message) => Promise<void>
const emailChannel = async (user, message) => { /* ... */ };
const smsChannel   = async (user, message) => { /* ... */ };

async function notify(user, message, channels) {
  await Promise.allSettled(channels.map((send) => send(user, message)));
}
```

Express middleware — `(req, res, next) => {}` — is exactly this. So is Node's error-first callback. So is `Array.prototype.map`'s callback. Pro: lightest thing that works, composes beautifully. Con: only works when the contract is a *single* operation. Two methods and you need an object.

**(d) JSDoc `@typedef` / `@implements`.** Write the contract down as a type, and let your editor check it. This is the underrated option.

```js
/**
 * @typedef {Object} Channel
 * @property {(user: User, message: string) => Promise<void>} send
 * @property {(user: User) => boolean} supports
 */

/** @implements {Channel} */
export class EmailChannel {
  async send(user, message) { /* ... */ }
  supports(user) { return Boolean(user.email); }
}
```

With `// @ts-check` at the top of the file (or `checkJs: true` in `jsconfig.json`), VS Code will **actually red-underline** a missing or mistyped method — real static checking, in plain `.js`, no build step, no TypeScript compiler in your pipeline. Pro: the only option that catches errors before runtime while staying pure JS. Con: comments can drift from code if nobody turns the checker on.

**(e) TypeScript's real `interface`.** If you have the option, this is the answer.

```ts
export interface NotificationChannel {
  send(user: User, message: string): Promise<SendResult>;
  supports(user: User): boolean;
}

export class SlackChannel implements NotificationChannel {
  // Compile error if send() is missing, misspelled, or has the wrong signature. Guaranteed.
  async send(user: User, message: string): Promise<SendResult> { /* ... */ }
  supports(user: User): boolean { return Boolean(user.slackId); }
}
```

The key insight: **TS interfaces vanish at runtime.** They're erased during compilation — pure compile-time checking, zero JS emitted, no performance cost. That's why they're strictly better than the throwing-base-class trick when you can use them: same guarantee, earlier, for free.

**Which to pick:**

| Situation | Use |
|---|---|
| TypeScript available | Real `interface` (e) |
| Plain JS, want real checking | JSDoc + `@ts-check` (d) |
| Plain JS, need shared code + loud failure | Abstract base class (a) |
| Contract is one function | Plain function contract (c) |
| Tests, quick fakes, tiny scripts | Duck typing (b) |

---

### 7. The interface is a testing seam

The reason this matters *today*, not just in eighteen months, is testing. An interface is a place where you can cut the code and insert something else.

```js
// Because NotificationService depends on the CONTRACT and not on Twilio,
// this test runs in 2ms, offline, on a plane, with zero API keys.
const sent = [];
const fakeChannel = { supports: () => true, async send(user, msg) { sent.push({ user, msg }); } };

const service = new NotificationService([fakeChannel]);
await service.notify({ id: 'u1' }, 'Order shipped');

assert.equal(sent.length, 1);
assert.equal(sent[0].msg, 'Order shipped');
```

If you find yourself unable to test a class without mocking a *library* (`jest.mock('twilio')`), that's the abstraction telling you it's missing. Library mocking is the smell; a real seam is the cure.

---

### 8. Full worked example: `NotificationChannel`

Here is the whole thing, end to end. Three implementations, one service that knows none of them by name.

```js
// ---------- THE CONTRACT (from section 6a) ----------
class NotificationChannel {
  constructor() {
    if (new.target === NotificationChannel) throw new TypeError('NotificationChannel is abstract');
  }
  /** @returns {Promise<{channel: string, ok: boolean}>} */
  async send(user, message) { throw new Error(`${this.constructor.name} must implement send()`); }
  /** Can this channel even reach this user? No phone number → no SMS. */
  supports(user) { throw new Error(`${this.constructor.name} must implement supports()`); }
  get name() { return this.constructor.name; }
}

// ---------- THE IMPLEMENTATIONS: each owns exactly ONE vendor ----------
class EmailChannel extends NotificationChannel {
  constructor(smtp) { super(); this.smtp = smtp; }          // smtp = nodemailer transport, injected
  supports(user) { return Boolean(user.email); }
  async send(user, message) {
    await this.smtp.sendMail({ to: user.email, subject: 'Notification', text: message });
    return { channel: 'email', ok: true };
  }
}

class SmsChannel extends NotificationChannel {
  constructor(twilio, fromNumber) { super(); this.twilio = twilio; this.from = fromNumber; }
  supports(user) { return Boolean(user.phone); }
  async send(user, message) {
    // Twilio's vocabulary (`body`, `from`) stops HERE. It never escapes this file.
    await this.twilio.messages.create({ to: user.phone, from: this.from, body: message });
    return { channel: 'sms', ok: true };
  }
}

class PushChannel extends NotificationChannel {
  constructor(fcmKey) { super(); this.fcmKey = fcmKey; }
  supports(user) { return Boolean(user.deviceToken); }
  async send(user, message) {
    const res = await fetch('https://fcm.googleapis.com/fcm/send', {
      method: 'POST',
      headers: { Authorization: `key=${this.fcmKey}`, 'Content-Type': 'application/json' },
      body: JSON.stringify({ to: user.deviceToken, notification: { body: message } }),
    });
    if (!res.ok) throw new Error(`FCM failed: ${res.status}`);
    return { channel: 'push', ok: true };
  }
}

// ---------- THE CONSUMER: knows the ROLE, not a single vendor ----------
class NotificationService {
  /** @param {NotificationChannel[]} channels — injected. The service never constructs them. */
  constructor(channels) { this.channels = channels; }

  async notify(user, message) {
    const usable = this.channels.filter((c) => c.supports(user));
    // allSettled, not all: a dead SMS provider must not stop the email from going out.
    const results = await Promise.allSettled(usable.map((c) => c.send(user, message)));
    return results.map((r, i) => ({
      channel: usable[i].name,
      ok: r.status === 'fulfilled',
      error: r.status === 'rejected' ? r.reason.message : null,
    }));
  }
}
```

Search `NotificationService` for the strings `twilio`, `smtp`, `fcm`, `email`, `sms`. **They do not appear.** That absence is the abstraction.

**Now the payoff. Six months later, product wants Slack alerts:**

```js
// ONE new file. Nothing else in the codebase is touched.
class SlackChannel extends NotificationChannel {
  constructor(webhookUrl) { super(); this.webhookUrl = webhookUrl; }
  supports(user) { return Boolean(user.slackId); }
  async send(user, message) {
    await fetch(this.webhookUrl, {
      method: 'POST',
      body: JSON.stringify({ channel: user.slackId, text: message }),
    });
    return { channel: 'slack', ok: true };
  }
}

// Wiring — the ONLY place that names concrete classes (composition root):
const service = new NotificationService([
  new EmailChannel(smtpTransport),
  new SmsChannel(twilioClient, '+15551234'),
  new PushChannel(process.env.FCM_KEY),
  new SlackChannel(process.env.SLACK_WEBHOOK),   // ← the entire change
]);

console.log(await service.notify(
  { id: 'u1', email: 'a@b.com', phone: '+919812345678', slackId: 'C123' },
  'Your order has shipped.'
));
// → [ { channel: 'EmailChannel', ok: true, error: null },
//     { channel: 'SmsChannel',   ok: true, error: null },
//     { channel: 'SlackChannel', ok: true, error: null } ]
//   PushChannel was skipped: supports() returned false — no deviceToken.
```

**Lines changed in `NotificationService` to add Slack: zero.** Say that out loud in an interview, then name the two principles you just demonstrated:

- **Open/Closed Principle (topic 15)** — the service is *open for extension* (new channels) and *closed for modification* (its source never changes). The `if/else` chain from section 1 was the opposite: every extension *was* a modification.
- **Dependency Inversion Principle (topic 18)** — `NotificationService` (high-level policy: "tell the user things") does not depend on `SmsChannel` (low-level detail: "call Twilio"). **Both** depend on the `NotificationChannel` abstraction. The arrow of dependency got *inverted*: `SmsChannel` now points *up* at the contract instead of the service pointing *down* at Twilio.

Abstraction is the machinery. OCP and DIP are what you get paid in.

---

## Visual / Diagram description

### Diagram 1 — The abstraction and its implementations

```
                    ┌──────────────────────────────┐
                    │    NotificationService       │   ← the CONSUMER
                    │──────────────────────────────│     knows only the contract
                    │ - channels: Channel[]        │
                    │──────────────────────────────│
                    │ + notify(user, message)      │
                    └───────────────┬──────────────┘
                                    │ depends on (uses)
                                    ▼
                    ┌──────────────────────────────┐
                    │  «abstract» NotificationChannel │   ← the CONTRACT / the wall socket
                    │──────────────────────────────│
                    │ + send(user, message)  ✱     │   ✱ = abstract, throws if not overridden
                    │ + supports(user): boolean ✱  │
                    └───────────────△──────────────┘
                                    │ implements / extends
        ┌──────────────┬────────────┴────────┬──────────────────┐
        │              │                     │                  │
┌───────┴──────┐ ┌─────┴───────┐  ┌──────────┴─────┐  ┌─────────┴──────┐
│ EmailChannel │ │ SmsChannel  │  │  PushChannel   │  │  SlackChannel  │
│──────────────│ │─────────────│  │────────────────│  │────────────────│
│ smtp client  │ │ twilio sdk  │  │  FCM http api  │  │  slack webhook │
│ + send()     │ │ + send()    │  │  + send()      │  │  + send()      │
│ + supports() │ │ + supports()│  │  + supports()  │  │  + supports()  │
└──────────────┘ └─────────────┘  └────────────────┘  └────────────────┘
                                                       ▲
                                                       └── ADDED LATER.
                                                           NotificationService: 0 lines changed.
```

**Read it like this:** the arrow from `NotificationService` points *down* to the abstraction, and the arrows from the four concrete channels point *up* to the same abstraction. Nothing points sideways. `NotificationService` has no line to `SmsChannel` — it has never heard of Twilio. That empty space between the service and the concrete classes **is** the abstraction doing its job.

### Diagram 2 — Where the details are allowed to live

```
   ┌──────────────────────────────────────────────────────────┐
   │  APPLICATION LAYER  — speaks business: "notify the user"  │
   │  checkout()   NotificationService   OrderService          │
   └───────────────────────────┬──────────────────────────────┘
                               │  ← THE WALL. Nothing below
   ═══════════════════════════════════════════════════════════      leaks above it.
                               │
   ┌───────────────────────────┴──────────────────────────────┐
   │  ADAPTER LAYER — speaks vendor: SMTP, Twilio, FCM, Slack  │
   │  EmailChannel   SmsChannel   PushChannel   SlackChannel   │
   └───────────────────────────┬──────────────────────────────┘
                               │
   ┌───────────────────────────┴──────────────────────────────┐
   │  TRANSPORT — sockets, TLS handshakes, retries, HTTP/2     │
   └──────────────────────────────────────────────────────────┘
```

The "mixing altitudes" bug from section 3 is exactly what happens when a line from the bottom box appears in the top box. If you ever see `net.connect()` in the same function as `chargeCustomer()`, you have punched a hole through the wall. Redraw this diagram on a whiteboard and point at the hole — interviewers love that.

---

## Real world examples

### Node.js Streams

Node's `stream` module is one of the cleanest abstractions in the ecosystem. A `Readable` is a contract: it emits `'data'`, it emits `'end'`, it can be `.pipe()`d. A file read (`fs.createReadStream`), an HTTP request body (`req`), a gzip decompressor, and a socket are all Readables — and `pipeline(source, gzip, destination)` works identically across all of them, because it depends on the *role* ("something that produces bytes"), never on the implementation.

And it leaks, exactly as Spolsky predicts: **backpressure**. Streams hide flow control beautifully until your fast producer overwhelms a slow consumer, memory balloons, and now you must understand `highWaterMark`, `.pause()`, `.resume()`, and why `pipe()` handles this but a naive `on('data')` loop does not. The abstraction is excellent *and* it leaks. Both things are true.

### Stripe's API as a payment abstraction

Stripe's own public API is itself an abstraction over dozens of card networks, banks, and regional payment rails (Visa, Mastercard, ACH, SEPA, UPI, iDEAL). You call `charge`; Stripe hides the fact that a European SEPA debit and a US card charge are wildly different processes with different settlement times.

And it leaks: SEPA takes *days* to settle, while a card auth is instant. So Stripe cannot fully hide it — it surfaces the leak deliberately via webhooks and a `status` field (`pending` → `succeeded`), forcing you to write asynchronous, eventually-consistent code. That's a *well-managed* leak: rather than pretending, the abstraction gives you a controlled vocabulary for the thing it couldn't hide.

### Database drivers behind a Repository

Most serious Node codebases eventually put a `UserRepository` between the app and the database — `findById`, `save`, `findByEmail` — rather than sprinkling `db.query('SELECT ...')` through controllers. The role name is `UserRepository`; the implementation may be `PostgresUserRepository` or `InMemoryUserRepository` (for tests). This is a representative, extremely common pattern rather than any one company's secret, and it's the abstraction that makes "swap Postgres for DynamoDB in this one service" a scoped project instead of a rewrite.

---

## Trade-offs

**Abstracting a varying detail:**

| Pros | Cons |
|---|---|
| New implementations need zero changes to callers (OCP) | More files, more indirection |
| Vendor swap has a blast radius of one class | "Jump to definition" lands on a throwing stub |
| Real unit tests without network/mocks | New engineers must learn your vocabulary |
| Forces you to name the domain concept clearly | A **wrong** abstraction is worse than duplication |
| The `if/else` chain collapses to one factory | Stack traces get taller; debugging is slower |

**When to abstract vs. when to wait:**

| Signal | Verdict |
|---|---|
| One implementation, no second one on the roadmap | **Don't.** Inline it. |
| Two implementations, both very similar | **Wait.** You can't yet see what truly varies. |
| Three implementations, or a real vendor swap is scheduled | **Abstract now.** The shape is visible. |
| You literally cannot unit test the class without mocking a library | **Abstract now** — you need the seam. |
| "We might need a plugin system someday" | **Don't.** That's speculative generality. |

**Rule of thumb:** Abstract on the **second or third concrete need**, never on the first imagined one. When you do, **name the role, not the vendor** — `PaymentGateway`, not `StripeWrapper` — because the name is the part you can never cheaply undo. And accept up front that your abstraction *will* leak; pick one whose leaks you're prepared to understand.

---

## Common interview questions on this topic

### Q1: "JavaScript has no `interface` keyword. How do you enforce a contract?"

**Hint:** Name all five and their trade-offs: (1) abstract base class whose methods `throw new Error('Not implemented')` — runtime enforcement, gives you a home for shared code; (2) duck typing — zero ceremony, zero safety; (3) plain function/object contracts, e.g. Express middleware's `(req, res, next)`; (4) JSDoc `@typedef` + `@implements` with `// @ts-check` — real static checking in plain JS; (5) TypeScript's `interface`, which is erased at compile time so it costs nothing at runtime. Then say which you'd pick and why. Bonus: mention `new.target` to block instantiating the abstract class itself.

### Q2: "What is a leaky abstraction? Give a concrete example you've hit."

**Hint:** State Spolsky's law — *all non-trivial abstractions leak*. Then give the ORM/N+1 example with numbers: `findAll()` then `.getPosts()` in a loop = 1 + N queries; 1,000 users × 2ms = ~2 seconds instead of ~10ms; the fix (`include: [Post]` → one JOIN) requires the SQL knowledge the ORM promised to hide. Offer TCP as a second: it hides packet loss, but retransmission and exponential backoff surface as p99 latency spikes. Close with the practical rule: be fluent in the abstraction, literate one layer beneath.

### Q3: "Isn't more abstraction always better?"

**Hint:** No — this is a trap question and the right answer is a firm no. Name the costs: indirection, file sprawl, harder tracing, cognitive overhead, and Sandi Metz's *"duplication is far cheaper than the wrong abstraction."* Give the Rule of Three: abstract on the third concrete need, because two data points fit any line. Guessing wrong gives you coupling *and* complexity, and people start smuggling vendor-specific flags through the contract to escape it.

### Q4: "How would you design a notification system that supports email, SMS, and push — and might add Slack later?"

**Hint:** Don't start coding. Say the sentence: *"What varies is the delivery mechanism; what's stable is 'deliver a message to a user'."* Define `NotificationChannel` with `send(user, message)` and `supports(user)`. `NotificationService` takes an array of channels **injected via the constructor** and never names a concrete one. Then land the point: adding `SlackChannel` = one new file, zero changes to the service — that's OCP and DIP. Mention `Promise.allSettled` so one failing channel doesn't kill the rest.

### Q5: "What's wrong with naming a class `StripeWrapper`?"

**Hint:** It names the implementation, not the role, and that name propagates into every calling file, every test, and often the DB schema and log lines. When Razorpay arrives you either write an absurd `RazorpayWrapper` alongside it, or do a 200-file rename. Name it `PaymentGateway`. Then push further: the *signature* must also be vendor-neutral — `charge({ amountMinor, currency, customerRef })`, never `charge(paymentIntentId)`, because `paymentIntent` is a Stripe noun and it leaks the vendor straight through the wall.

---

## Practice exercise

### Build a `FileStore` abstraction and swap the backend

**Time: ~30-40 minutes.** Produce a single Node.js file (ESM) that runs with `node filestore.js`.

**Step 1 — Write the contract.** Create an abstract class `FileStore` with three methods that all `throw new Error('Not implemented')`:
- `async save(key, buffer)` → returns `{ url }`
- `async get(key)` → returns a `Buffer`
- `async delete(key)` → returns `void`

Use `new.target` in the constructor to make `new FileStore()` throw.

**Step 2 — Two implementations.**
- `LocalDiskFileStore` — uses `node:fs/promises`, writes to `./uploads/<key>`, returns `{ url: 'file://...' }`.
- `InMemoryFileStore` — backs onto a `Map`. This is your test double.

**Step 3 — A consumer that knows neither.** Write `class AvatarService { constructor(fileStore) {...} async uploadAvatar(userId, buffer) {...} }`. It must call `this.fileStore.save(...)` and **must not contain the words `fs`, `disk`, `Map`, or `memory` anywhere.** That constraint is the exercise.

**Step 4 — Prove the payoff.** In a `main()` block, run `AvatarService` twice: once with `new LocalDiskFileStore()`, once with `new InMemoryFileStore()`. Same service, zero changes.

**Step 5 — The real test.** Now add a **third** implementation, `S3FileStore` (you may stub the AWS call with a `console.log` and a fake URL). Before you write it, **count how many lines of `AvatarService` you expect to change.** The answer must be **zero**. If it isn't, your abstraction leaked — find out where and fix the contract.

**Stretch:** add a `@typedef` + `// @ts-check` to the top of the file and confirm your editor flags it if you delete `delete()` from one implementation.

---

## Quick reference cheat sheet

- **Abstraction = hiding what varies.** Find the thing that will be different next quarter and put a wall around it.
- **"Encapsulate what varies"** is the whole game. A `switch` on a type string is a missed abstraction announcing itself.
- **Name the ROLE, not the vendor.** `PaymentGateway` not `StripeWrapper`; `Cache` not `RedisHelper`; `FileStore` not `S3Uploader`. The name is the hardest thing to undo.
- **Vendor-neutral signatures too.** `charge({ amountMinor })`, never `charge(paymentIntentId)` — a vendor noun in a parameter leaks straight through the wall.
- **One function, one altitude.** `openTcpSocket()` and `chargeCustomer()` must never share a function body.
- **Spolsky's Law: all non-trivial abstractions leak.** ORMs leak as N+1 queries. TCP leaks as latency spikes. Caches leak as staleness and stampedes.
- **Be fluent in the abstraction, literate one layer down.** Use the ORM; know SQL anyway.
- **Abstraction costs real money:** indirection, file sprawl, taller stack traces, a vocabulary to learn.
- **"Duplication is far cheaper than the wrong abstraction."** — Sandi Metz. Believe her.
- **Rule of Three:** abstract on the second or third *concrete* need, never on the first *imagined* one.
- **JS has no `interface`.** Five options: abstract base class that throws; duck typing; plain function contracts; JSDoc `@typedef`/`@implements` with `@ts-check`; TypeScript `interface` (erased at compile time, free at runtime).
- **An interface is a testing seam.** If you can only test by mocking a library, the abstraction is missing.
- **The payoff sentence:** *"Adding `SlackChannel` required zero changes to `NotificationService`."* That's OCP + DIP, in one line.
- **Escape hatches matter.** An abstraction with no way to drop a level turns a leak into a wall.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [26 — Encapsulation and Information Hiding](./26-encapsulation-and-information-hiding.md) — abstraction decides *what* to hide; encapsulation is the mechanism that *enforces* the hiding |
| **Next** | [28 — Composition vs Inheritance](./28-composition-vs-inheritance.md) — once you have an abstraction, this is how you decide to reuse it: `has-a` or `is-a` |
| **Related** | [15 — Open/Closed Principle](./15-open-closed-principle.md) — adding `SlackChannel` without touching `NotificationService` *is* OCP; abstraction is the machinery that makes OCP possible |
| **Related** | [18 — Dependency Inversion Principle](./18-dependency-inversion-principle.md) — DIP is the rule "depend on the abstraction, not the concretion"; this doc is how you build the abstraction it tells you to depend on |
| **Related** | [33 — Strategy Pattern](./33-strategy-pattern.md) — the `NotificationChannel` family is Strategy: interchangeable algorithms behind one contract |
