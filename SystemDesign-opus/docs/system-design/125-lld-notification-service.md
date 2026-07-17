# 125 — Design a Notification Service (Class Level)
## Category: LLD Case Study

---

## What is this?

A **notification service** is the piece of software that takes an event ("your order shipped", "someone liked your photo") and delivers a message to a user over one or more **channels** — email, SMS, push, in-app. This doc is the **class-level (LLD)** companion to the notification *system* HLD (98): there we worried about queues, retries, and scale; here we design the clean object model that any of those systems is built on top of.

Think of it as a **post office**. You hand over a letter (the event). The post office knows each recipient's preferences — some want it by courier, some by registered mail, some just pinned to a community board — and it delivers via exactly the channels each person opted into, without you ever touching an envelope.

---

## Why does it matter?

**Interview angle:** "Design a notification service" is one of the most common LLD prompts because it cleanly exercises three patterns at once — **Observer**, **Strategy/polymorphism**, and **Factory**. Interviewers use it to see whether you can turn a vague requirement ("notify users") into extensible classes where adding a new channel is a *new class, not an edit to old ones*.

**Real-work angle:** Every product grows channels over time. You start with email, then add push, then SMS, then Slack, then WhatsApp. If your first design hard-codes `if (channel === 'email') ... else if (channel === 'sms') ...`, every new channel means editing and re-testing a giant switch that touches everything. Get the object model right and each new channel is a self-contained, testable class. This is the **Open/Closed Principle** made concrete: open for extension, closed for modification.

---

## The core idea — explained simply

### The analogy: a newspaper subscription office

Picture the office that runs a newspaper's subscriptions.

- **Readers subscribe** to sections they care about — Sports, Finance, Weather. They don't get sections they didn't ask for.
- Each reader also picks **how** they receive it: printed to the door, emailed as PDF, or an app alert.
- When the Sports desk finishes a story, it doesn't phone every reader. It just **publishes** the story to the office. The office looks up who subscribed to Sports and delivers to each via their chosen method.
- Adding a brand-new delivery method (say, a fax edition) doesn't change how the Sports desk works. It's a new delivery clerk the office can hire.

Map it back to code:

| Newspaper office | Notification service | Pattern |
|---|---|---|
| Reader subscribes to "Sports" | `service.subscribe(user, NotificationType.SPORTS)` | **Observer** |
| Sports desk publishes a story | `service.notify(NotificationType.SPORTS, payload)` | **Observer** (event fanout) |
| Reader's chosen delivery methods | `user.enabledChannels` | preferences |
| Delivery clerk (courier / email / app) | `EmailChannel`, `PushChannel`, ... behind a common `Channel` interface | **Strategy** / polymorphism |
| Office hires the right clerk for a method | `ChannelFactory.create(channelType)` | **Factory** |
| Audit desk that logs every delivery | a generic observer | **Observer** |

The magic sentence to say in an interview: *"The service publishes an event; interested users are notified; each user is delivered to over their enabled channels through a common interface, and channels are constructed by a factory."* That one sentence names all three patterns.

---

## Key concepts inside this topic

We follow the standard 7-step LLD method (see [111 — LLD Approach Framework](./111-lld-approach-framework.md)).

### 1. Requirements & clarifying questions

**Functional requirements:**
- Send a notification to a user over **one or more channels**: email, SMS, push, in-app.
- Users have **per-channel preferences** — they opt into the channels they want.
- Users **subscribe** to notification **types** (e.g. `PROMOTION`, `ORDER_UPDATE`, `SECURITY_ALERT`); unsubscribed users get nothing.
- Content is **templated** — we store a template id + variables, not a pre-rendered string.
- **Extensible to new channels** (Slack, WhatsApp) without touching existing channel code.

**Clarifying questions to ask the interviewer:**
- *Do we need delivery guarantees / retries?* — Yes in production, but that's the **HLD (98)**: queues, dead-letter, at-least-once delivery. Here we design the classes; I'll leave a clean seam to plug a queue in.
- *Priorities?* — I'll model a `Priority` enum on the notification so a scheduler could later order dispatch; the ordering logic itself is an extension.
- *Should some notification types ignore preferences?* (e.g. security alerts) — Good edge case; I'll note it in extensions.
- *One user or fan-out to many?* — Both. `notify(type, payload)` fans out to all subscribers.

Being explicit about *what is LLD vs HLD* is exactly what earns points: you show you know retries/queues exist but you're scoping to the object design.

### 2. Identify core objects (nouns)

Pull the nouns straight out of the requirements:

- **`NotificationService`** — the coordinator (the post office). Holds subscriptions and generic observers.
- **`Notification`** — a value object: `type`, `templateId`, `variables` (payload), `priority`.
- **`Channel`** — an **interface/base class**: `send(user, renderedMessage)`. Concretes: `EmailChannel`, `SmsChannel`, `PushChannel`, `InAppChannel`.
- **`User`** — `id`, contact info, a **set of enabled `ChannelType`s**, and a **set of subscribed `NotificationType`s**.
- **`Template` / `TemplateEngine`** — renders a template id + variables into a message string.
- **`Observer`** — anything that reacts to a send: the users' channels, plus generic observers like an **audit logger** or **analytics tracker**.
- **`ChannelFactory`** — creates the right `Channel` from a `ChannelType`.

**Enums** (frozen constants): `ChannelType`, `NotificationType`, `Priority`.

### 3. Behaviours (verbs) + who owns them

Ownership is the heart of clean OO — each responsibility lives in exactly one class:

| Behaviour | Owner | Why |
|---|---|---|
| `subscribe(user, type)` / `unsubscribe` | `NotificationService` | it owns the subscription registry |
| `notify(type, payload)` — fan out | `NotificationService` | it knows who subscribed |
| decide *which channels* to use | `User` (its `enabledChannels`) | the user owns their own preferences |
| render content | `TemplateEngine` | single place that knows templates |
| actually deliver | each concrete `Channel` | each channel owns its own delivery mechanics |
| construct a channel | `ChannelFactory` | one place for construction logic |
| observe every send | generic observers | cross-cutting concerns stay out of the core loop |

Notice `NotificationService` never knows *how* an email is sent, and a `Channel` never knows *who* subscribed. Responsibilities don't leak.

### 4. The pattern design — three patterns, cleanly

This problem is the textbook showcase for three patterns working together.

**(a) Channels via a common interface + Strategy / polymorphism.**
We define an abstract `Channel` with one method, `send(user, message)`. Concrete `EmailChannel`, `SmsChannel`, `PushChannel`, `InAppChannel` each implement it their own way. The service iterates a user's enabled channels and calls `send` on each — **it never knows which concrete class it holds**. Each channel is an interchangeable *strategy* for delivery. Adding a `SlackChannel` is a brand-new class plus one factory case; **zero changes** to the service or the other channels. That's the Open/Closed Principle, and it's the abstraction-and-interfaces lesson (recall 27) made concrete. I'm using **Strategy/polymorphism** because delivery mechanics vary independently and I want them swappable.

**(b) Observer — the star.**
The system is modelled as **events with subscribers**. Interested parties *subscribe* to notification types; when an event fires, all subscribed parties are notified. Two flavours of observer:
- **Users** subscribe to notification *types* they care about (`subscribe(user, NotificationType.PROMOTION)`).
- **Generic observers** — an audit logger, an analytics tracker — observe *every* send, regardless of type.

I'm using **Observer** (recall [41 — Observer](./41-pattern-observer.md)) because the producer of an event (whoever calls `notify`) must not be coupled to the set of consumers. New consumers (a fraud-detection observer, a metrics counter) attach without the producer knowing they exist. This decoupling is *the* reason Observer is the natural fit here.

**(c) Factory — channel creation in one place.**
A `ChannelFactory.create(channelType)` maps a `ChannelType` enum to the right `Channel` instance. Construction lives in one spot; if a channel later needs config (API keys, endpoints), only the factory changes. I'm using **Factory** (recall [30 — Factory Method](./30-pattern-factory-method.md)) so the service asks for "a channel for `SMS`" without ever writing `new SmsChannel()` — it stays ignorant of concrete types.

### 5. Class diagram

```
                       ┌───────────────────────────────────┐
                       │        NotificationService        │
                       │-----------------------------------│
                       │ subscribers: Map<NotifType,Set<User>>
                       │ observers: Observer[]  (audit...) │
                       │ templateEngine: TemplateEngine    │
                       │ factory: ChannelFactory           │
                       │-----------------------------------│
                       │ subscribe(user, type)             │
                       │ unsubscribe(user, type)           │
                       │ addObserver(observer)             │
                       │ notify(type, payload) ────────────┼──┐ fan-out
                       └──────────┬────────────────────────┘  │
                                  │ uses                        │ notifies
             ┌────────────────────┼───────────────┐            ▼
             ▼                    ▼                ▼    ┌────────────────┐
   ┌──────────────┐   ┌────────────────┐  ┌───────────┐│  «Observer»    │
   │TemplateEngine│   │ ChannelFactory │  │   User    ││ onNotify(evt)  │
   │ render(id,v) │   │ create(type)   │  │-----------│└────────────────┘
   └──────────────┘   └───────┬────────┘  │ enabled   │   ▲        ▲
                              │ creates    │  Channels │   │        │
                              ▼            │ subscribed│ AuditLog  Analytics
                    ┌──────────────────┐   │  Types    │  Observer  Observer
                    │   «Channel»      │   └───────────┘
                    │ send(user, msg)  │  (base / interface)
                    └───┬───┬───┬───┬──┘
                        │   │   │   │   (polymorphism)
          ┌─────────────┘   │   │   └──────────────┐
          ▼                 ▼   ▼                   ▼
   ┌────────────┐  ┌───────────┐ ┌────────────┐ ┌────────────┐
   │EmailChannel│  │ SmsChannel│ │PushChannel │ │InAppChannel│
   └────────────┘  └───────────┘ └────────────┘ └────────────┘
```

Read it top-down: the service holds subscriptions and generic observers; on `notify` it renders via the `TemplateEngine`, asks the `ChannelFactory` for each channel, and dispatches through the polymorphic `Channel` base. Users hold their own channel + type preferences.

### 6. Full JavaScript implementation

Complete and runnable — copy into `notification.js` and run `node notification.js`.

```javascript
'use strict';

// ---------- Enums (frozen constants) ----------
const ChannelType = Object.freeze({
  EMAIL: 'EMAIL',
  SMS: 'SMS',
  PUSH: 'PUSH',
  IN_APP: 'IN_APP',
});

const NotificationType = Object.freeze({
  PROMOTION: 'PROMOTION',
  ORDER_UPDATE: 'ORDER_UPDATE',
  SECURITY_ALERT: 'SECURITY_ALERT',
});

const Priority = Object.freeze({
  LOW: 1,
  NORMAL: 2,
  HIGH: 3,
});

// ---------- Channel base (interface) + concretes : STRATEGY/polymorphism ----------
// Base declares the contract. Any subclass MUST implement send().
class Channel {
  get type() {
    throw new Error('Channel subclass must define a type getter');
  }
  // send(user, renderedMessage) -> delivery record
  send(_user, _message) {
    throw new Error('Channel.send() must be implemented by subclass');
  }
}

// Each concrete channel only knows how to deliver over ITS medium.
// We "deliver" by logging + returning a record so the demo runs without real providers.
class EmailChannel extends Channel {
  get type() { return ChannelType.EMAIL; }
  send(user, message) {
    console.log(`   [EMAIL] to ${user.email}: "${message}"`);
    return { channel: this.type, to: user.email, status: 'SENT' };
  }
}

class SmsChannel extends Channel {
  get type() { return ChannelType.SMS; }
  send(user, message) {
    console.log(`   [SMS]   to ${user.phone}: "${message}"`);
    return { channel: this.type, to: user.phone, status: 'SENT' };
  }
}

class PushChannel extends Channel {
  get type() { return ChannelType.PUSH; }
  send(user, message) {
    console.log(`   [PUSH]  to device(${user.deviceToken}): "${message}"`);
    return { channel: this.type, to: user.deviceToken, status: 'SENT' };
  }
}

class InAppChannel extends Channel {
  get type() { return ChannelType.IN_APP; }
  send(user, message) {
    // In-app just drops the message into the user's inbox in memory.
    user.inbox.push(message);
    console.log(`   [IN_APP] to inbox of ${user.id}: "${message}"`);
    return { channel: this.type, to: user.id, status: 'SENT' };
  }
}

// ---------- Factory : one place that maps ChannelType -> Channel ----------
class ChannelFactory {
  constructor() {
    // Registry pattern: extendable without editing a switch. Add Slack -> one line.
    this._registry = new Map([
      [ChannelType.EMAIL, () => new EmailChannel()],
      [ChannelType.SMS, () => new SmsChannel()],
      [ChannelType.PUSH, () => new PushChannel()],
      [ChannelType.IN_APP, () => new InAppChannel()],
    ]);
  }
  register(channelType, creatorFn) {
    this._registry.set(channelType, creatorFn);
  }
  create(channelType) {
    const creator = this._registry.get(channelType);
    if (!creator) throw new Error(`No channel registered for ${channelType}`);
    return creator();
  }
}

// ---------- User : owns their own preferences ----------
class User {
  constructor({ id, email, phone, deviceToken }) {
    this.id = id;
    this.email = email;
    this.phone = phone;
    this.deviceToken = deviceToken;
    this.enabledChannels = new Set();       // which channels they opted into
    this.subscribedTypes = new Set();       // which notification types they want
    this.inbox = [];                        // for IN_APP delivery
  }
  enableChannel(channelType) { this.enabledChannels.add(channelType); return this; }
  disableChannel(channelType) { this.enabledChannels.delete(channelType); return this; }
  hasChannel(channelType) { return this.enabledChannels.has(channelType); }
}

// ---------- Template + TemplateEngine ----------
// HLD lesson: store the template id + variables, not the rendered text,
// so wording/localization can change without reprocessing stored events.
class Template {
  constructor(id, text) {
    this.id = id;
    this.text = text; // e.g. "Hi {name}, your order {orderId} shipped!"
  }
}

class TemplateEngine {
  constructor() {
    this._templates = new Map();
  }
  register(template) { this._templates.set(template.id, template); return this; }
  render(templateId, variables = {}) {
    const tpl = this._templates.get(templateId);
    if (!tpl) throw new Error(`Unknown template: ${templateId}`);
    // Replace {key} tokens with variable values.
    return tpl.text.replace(/\{(\w+)\}/g, (_, key) =>
      key in variables ? String(variables[key]) : `{${key}}`
    );
  }
}

// ---------- Generic Observer base (audit / analytics) ----------
class Observer {
  onNotify(_event) {
    throw new Error('Observer.onNotify() must be implemented');
  }
}

class AuditLogObserver extends Observer {
  constructor() { super(); this.log = []; }
  onNotify(event) {
    const entry = `AUDIT | user=${event.user.id} type=${event.type} ` +
                  `channel=${event.record.channel} status=${event.record.status}`;
    this.log.push(entry);
    console.log(`   ${entry}`);
  }
}

class AnalyticsObserver extends Observer {
  constructor() { super(); this.counts = new Map(); }
  onNotify(event) {
    const key = `${event.type}:${event.record.channel}`;
    this.counts.set(key, (this.counts.get(key) || 0) + 1);
  }
}

// ---------- NotificationService : the coordinator (OBSERVER + FACTORY + STRATEGY) ----------
class NotificationService {
  constructor(templateEngine, channelFactory) {
    this.templateEngine = templateEngine;
    this.factory = channelFactory;
    // subscribers: NotificationType -> Set<User>   (the Observer registry)
    this._subscribers = new Map();
    // generic observers that see EVERY send
    this._observers = [];
    // simple cache so we build each Channel object once
    this._channelCache = new Map();
  }

  // --- Observer registration ---
  subscribe(user, notificationType) {
    if (!this._subscribers.has(notificationType)) {
      this._subscribers.set(notificationType, new Set());
    }
    this._subscribers.get(notificationType).add(user);
    user.subscribedTypes.add(notificationType);
    return this;
  }

  unsubscribe(user, notificationType) {
    this._subscribers.get(notificationType)?.delete(user);
    user.subscribedTypes.delete(notificationType);
    return this;
  }

  addObserver(observer) { this._observers.push(observer); return this; }

  _getChannel(channelType) {
    if (!this._channelCache.has(channelType)) {
      this._channelCache.set(channelType, this.factory.create(channelType));
    }
    return this._channelCache.get(channelType);
  }

  // --- The core fan-out ---
  // notification = { type, templateId, variables, priority }
  notify(notification) {
    const { type, templateId, variables = {}, priority = Priority.NORMAL } = notification;
    const audience = this._subscribers.get(type) || new Set();
    console.log(`\n>> notify(${type}) priority=${priority} to ${audience.size} subscriber(s)`);

    for (const user of audience) {
      const message = this.templateEngine.render(templateId, { ...variables, name: user.id });

      // Respect each user's channel preferences (the User owns this).
      const targetChannels = [...user.enabledChannels];
      if (targetChannels.length === 0) {
        console.log(`   (user ${user.id} has no enabled channels — skipped)`);
        continue;
      }

      for (const channelType of targetChannels) {
        const channel = this._getChannel(channelType);   // Factory-built, polymorphic
        const record = channel.send(user, message);       // Strategy: we don't know the concrete class
        // Fan out to generic observers (audit, analytics) — Observer.
        const event = { type, user, record, priority, message };
        for (const obs of this._observers) obs.onNotify(event);
      }
    }
  }
}

// ---------- Demo ----------
function main() {
  // 1. Wire up engine + templates
  const engine = new TemplateEngine();
  engine.register(new Template('promo', 'Hi {name}, {discount}% off ends tonight!'));
  engine.register(new Template('order', 'Hi {name}, your order {orderId} has shipped.'));

  // 2. Service with a factory
  const service = new NotificationService(engine, new ChannelFactory());

  // 3. Generic observers see EVERY send
  const audit = new AuditLogObserver();
  const analytics = new AnalyticsObserver();
  service.addObserver(audit).addObserver(analytics);

  // 4. Users with DIFFERENT channel preferences
  const alice = new User({ id: 'alice', email: 'a@x.com', deviceToken: 'dev-A' })
    .enableChannel(ChannelType.EMAIL)
    .enableChannel(ChannelType.PUSH);

  const bob = new User({ id: 'bob', phone: '+1-555-0100' })
    .enableChannel(ChannelType.SMS);

  const carol = new User({ id: 'carol', email: 'c@x.com' })
    .enableChannel(ChannelType.EMAIL)
    .enableChannel(ChannelType.IN_APP);

  // 5. Subscriptions (Observer): who wants which type
  service.subscribe(alice, NotificationType.PROMOTION)   // alice wants promos
         .subscribe(bob, NotificationType.PROMOTION)     // bob too
         .subscribe(carol, NotificationType.ORDER_UPDATE); // carol only order updates
  // NOTE: carol is NOT subscribed to PROMOTION -> she must get nothing below.

  // 6. Fire a PROMOTION event
  service.notify({
    type: NotificationType.PROMOTION,
    templateId: 'promo',
    variables: { discount: 25 },
    priority: Priority.HIGH,
  });
  // Expect: alice over EMAIL+PUSH, bob over SMS, carol gets NOTHING.

  // 7. Fire an ORDER_UPDATE event
  service.notify({
    type: NotificationType.ORDER_UPDATE,
    templateId: 'order',
    variables: { orderId: 'ORD-42' },
  });
  // Expect: only carol, over EMAIL + IN_APP.

  // 8. Show the analytics observer aggregated everything
  console.log('\n== Analytics counts ==');
  for (const [key, count] of analytics.counts) console.log(`   ${key} = ${count}`);
  console.log('\n== Carol inbox (in-app) ==', carol.inbox);
}

main();
```

Run it and you'll see the promotion delivered to **alice over email+push** and **bob over sms**, **carol receives nothing** (she never subscribed to promos), the order update reaches **only carol over email+in-app**, and the **audit observer logs every single send** while analytics tallies them. That output *proves* Observer (right audience), preferences (right channels), and polymorphism (right delivery) are all working together.

### 7. Design patterns used and WHY

- **Observer (the star)** — `NotificationService` is the *subject*; users and generic observers are *observers*. `subscribe`/`notify` decouple the event producer from consumers. New consumers (fraud detection, metrics) attach without changing the producer. Chosen because notification is fundamentally *publish an event → fan out to whoever cares*.
- **Strategy / polymorphism** — `Channel.send` is one interface with interchangeable implementations. The service dispatches without knowing the concrete channel. Chosen because delivery mechanics vary independently and must be swappable/extensible (OCP).
- **Factory** — `ChannelFactory.create(type)` centralizes construction. Chosen so the service never writes `new SmsChannel()` and channel wiring/config lives in one place.

**Connection to the HLD ([98](./98-hld-notification-system.md)):** this is the *class design*. The HLD wraps it with **message queues** (Kafka/SQS) between `notify` and the channels for async delivery, **retries + dead-letter queues** for reliability, **rate limiting** per provider, and **horizontal scale** across worker fleets. Same objects; the HLD adds the distributed plumbing.

---

## Visual / Diagram description

Sequence of a single `notify(PROMOTION)` call:

```
Caller        NotificationService    TemplateEngine   ChannelFactory   EmailChannel   AuditObserver
  │  notify(PROMOTION) │                   │                │              │              │
  ├───────────────────►│                   │                │              │              │
  │                    │ lookup subscribers│                │              │              │
  │                    ├──(alice, bob)     │                │              │              │
  │                    │ render('promo',v) │                │              │              │
  │                    ├──────────────────►│                │              │              │
  │                    │◄── "Hi alice,25%off"                │              │              │
  │                    │ for each enabled channel of alice   │              │              │
  │                    │ create(EMAIL)     │                │              │              │
  │                    ├───────────────────────────────────►│              │              │
  │                    │◄────────── EmailChannel ────────────┤              │              │
  │                    │ send(alice, msg)  │                │              │              │
  │                    ├──────────────────────────────────────────────────►│              │
  │                    │◄──────────── record{SENT} ──────────────────────────┤             │
  │                    │ onNotify(event)   │                │              │              │
  │                    ├─────────────────────────────────────────────────────────────────►│
  │                    │  (repeat for PUSH, then bob's SMS)  │              │              │
```

Every arrow is one method call. The service is the hub; it talks to the engine once per user, to the factory once per channel type, to each channel per delivery, and to every observer per delivery.

---

## Real world examples

### Firebase Cloud Messaging (FCM) — Google
FCM is a push-notification backbone. Conceptually (representative), your app publishes a message to a *topic*; devices *subscribe* to topics — a direct Observer analogue. FCM abstracts device platforms (Android/iOS/web) behind one send API, mirroring our polymorphic `Channel`.

### Amazon SNS (Simple Notification Service)
SNS is literally a pub/sub service: publishers send to a *topic*, and *subscriptions* fan out to endpoints — email, SMS, HTTP, SQS queues, Lambda. That's Observer plus channel polymorphism at cloud scale; each endpoint type is effectively a `Channel` strategy, and SNS handles the queue/retry layer the HLD (98) describes.

### Courier / Knock (notification infrastructure startups)
These products (representative) offer exactly this model as a service: per-user channel preferences, templated content, and a single API to fan a notification out across email/SMS/push/Slack. Their public docs emphasize "add a channel without changing your code" — the OCP payoff our design targets.

---

## Trade-offs

**Observer for fan-out:**

| Pros | Cons |
|---|---|
| Producer fully decoupled from consumers | Harder to trace flow (implicit wiring) |
| Add observers with zero producer changes | Can leak memory if you never unsubscribe |
| Cross-cutting concerns (audit) stay separate | Ordering of observers is not guaranteed |

**Channel polymorphism (Strategy):**

| Pros | Cons |
|---|---|
| New channel = new class, no edits (OCP) | More classes / files to navigate |
| Each channel unit-tested in isolation | Shared behaviour can get duplicated |

**Factory for channel creation:**

| Pros | Cons |
|---|---|
| One place for construction & config | Slight indirection ("where is it built?") |
| Registry makes new channels one line | Overkill if you'll only ever have one channel |

**The sweet spot:** use all three when you expect the channel list and the consumer list to *grow*. If you'll forever have exactly one channel and no observers, a plain function is fine — don't pattern-golf a problem that isn't extensible.

---

## Common interview questions on this topic

### Q1: "How do you add a WhatsApp channel without touching existing code?"
**Hint:** Create `class WhatsAppChannel extends Channel` implementing `send`, then `factory.register(ChannelType.WHATSAPP, () => new WhatsAppChannel())`. Add the enum value. The service loop is untouched — that's the Open/Closed Principle, and it's why we used Factory + polymorphism.

### Q2: "Where does user preference live, and why not in the service?"
**Hint:** In the `User` object (`enabledChannels`). The service owns *who subscribed to what type*; the user owns *how they want to be reached*. Keeping preference in `User` means a user's data travels with them and the service stays a stateless coordinator.

### Q3: "Which pattern is doing the fan-out, and why is it the right one?"
**Hint:** Observer. The producer of an event shouldn't know or care who consumes it. `subscribe`/`notify` lets you attach new consumers (analytics, audit, a new user) without editing the producer — the defining benefit of Observer.

### Q4: "A security alert must reach the user even if they disabled channels. How?"
**Hint:** Add a `mandatory`/`overridePreferences` flag on the notification (or per-type policy). For those types, the service uses a fallback channel set instead of `user.enabledChannels`. It's a small branch in the dispatch loop, not a redesign — the seam is already there.

### Q5: "How does this class design relate to the HLD version (98)?"
**Hint:** This is the object model that runs inside a worker. The HLD adds a queue between `notify` and dispatch (async, buffering), retries + DLQ (reliability), rate limits per provider, and horizontal scaling. Same classes, distributed plumbing around them.

---

## Practice exercise

### Build and extend the service (~30 min)

1. Copy the implementation into `notification.js` and run it. Confirm the expected output: alice email+push, bob sms, carol nothing on the promo; carol email+in-app on the order update; audit logs every send.
2. **Add a `SlackChannel`.** New class extending `Channel`, add `ChannelType.SLACK`, register it in the factory, enable it for a user, and prove it delivers — *without editing `NotificationService`.*
3. **Add a mandatory type.** Give `Notification` an `overridePreferences: true` option; when set (e.g. `SECURITY_ALERT`), send over email even if the user disabled it. Fire one and verify a user with no channels still gets the email.
4. **Add a `unsubscribe` demo.** Unsubscribe bob from promos, fire another promo, confirm only alice receives it.

Produce: the updated file and a short note on which class each change touched (you should find each change is *localized*).

---

## Quick reference cheat sheet

- **Notification service** — turns an event into deliveries over one or more channels per user preferences.
- **This is LLD (class design)**; the queue/retry/scale layer is the HLD, [98](./98-hld-notification-system.md).
- **`Channel` base + concretes** — one `send(user, message)` interface; Email/SMS/Push/InApp implement it (Strategy/polymorphism).
- **`ChannelFactory.create(type)`** — single place that builds channels; use a registry so new channels are one line (Factory).
- **Observer is the star** — `subscribe(user, type)` + `notify(type, payload)` decouple the event producer from consumers.
- **Two observer flavours** — users subscribed to a *type*, and generic observers (audit, analytics) that see *every* send.
- **User owns preferences** — `enabledChannels` and `subscribedTypes` live on `User`, not the service.
- **TemplateEngine** — store template id + variables, render at send time; never store pre-rendered text.
- **Enums via `Object.freeze`** — `ChannelType`, `NotificationType`, `Priority`.
- **OCP payoff** — adding a channel is a *new class + factory line*, zero edits to existing code.
- **Add a mandatory type** — an `overridePreferences` flag branches the dispatch loop for security alerts.
- **Watch for** — unbounded observer lists (unsubscribe them), and untraceable flow (document the wiring).

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [41 — Observer Pattern](./41-pattern-observer.md) | The core pattern this design is built on — subscribe + event fanout. |
| **Next** | [98 — HLD: Notification System](./98-hld-notification-system.md) | The system-level companion: queues, retries, delivery guarantees, scale. |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) | Channels are interchangeable delivery strategies behind one interface. |
| **Related** | [30 — Factory Method](./30-pattern-factory-method.md) | `ChannelFactory` centralizes channel construction. |
| **Related** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step method (nouns → verbs → classes → patterns) used throughout. |
