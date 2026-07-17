# 39 — Bridge Pattern

## Category: LLD Patterns

---

## What is this?

The **Bridge pattern** splits one big class hierarchy into **two independent hierarchies** — an *abstraction* (what the thing is) and an *implementation* (how it actually gets done) — and connects them with a reference instead of inheritance.

Think of a **TV remote and a TV**. The remote is the abstraction: it has `power()`, `volumeUp()`, `channelUp()`. The TV is the implementation: a Sony TV and a Samsung TV turn on in completely different ways internally. You don't build a `SonyBasicRemote`, `SonyUniversalRemote`, `SamsungBasicRemote`, `SamsungUniversalRemote`. You build remotes on one side, TVs on the other, and the remote **holds a reference** to whatever TV it's pointed at. That reference is the *bridge*.

---

## Why does it matter?

Without Bridge, you get **class explosion**. Every time you have two things that vary independently — shapes × renderers, notifications × delivery channels, reports × export formats, devices × remotes — and you model both with inheritance, the number of classes becomes the *product* of both dimensions, not the sum.

- 2 shapes × 3 renderers = **6 classes**
- Add a 3rd shape → **9 classes**
- Add a 4th renderer → **12 classes**

You are now maintaining 12 classes that mostly duplicate each other. Fix a bug in circle-drawing logic and you fix it in three places (or forget one).

**In interviews:** Bridge shows up whenever the interviewer says *"now we need to support a second database / a second rendering target / a second payment gateway."* If your instinct is `class MySQLUserRepository extends UserRepository` **and** `class CachedMySQLUserRepository extends MySQLUserRepository`, you're heading toward explosion. Bridge is the answer, and the follow-up question is always **"how is this different from Strategy?"** (we answer that honestly below).

**At work:** Bridge is the pattern behind almost every "driver" or "adapter layer" you've ever used. Your ORM has a `Query` abstraction bridged to a `Dialect` implementation. Your logger has a `Logger` abstraction bridged to a `Transport` implementation. Winston's `logger.info()` + `transports: [new transports.File(), new transports.Http()]` is Bridge in production.

---

## The core idea — explained simply

### The Power Plug Analogy

You have **appliances**: a laptop charger, a hair dryer, a phone charger. You have **wall sockets**: UK, US, EU, India.

The naive way to solve this is to manufacture a laptop charger with a UK plug welded on, another with a US plug welded on, another with an EU plug... You'd be manufacturing `appliances × sockets` distinct products. 4 appliances and 5 countries means 20 SKUs in the warehouse.

The real world solved it with a **travel adapter**. You make appliances on one side (with a standard connector), sockets on the other, and one small adapter *bridges* them. Now 4 appliances + 5 socket types = 9 things, and adding a 6th country costs exactly **one** new adapter — not four.

The key insight: **the appliance doesn't care how the electricity is delivered, only that it is.** The laptop needs "give me 5V". *How* the country's grid delivers it is not the laptop's business.

Map it back:

| Analogy | Bridge Pattern Term | In code |
|---------|--------------------|---------|
| Laptop charger, hair dryer | **Abstraction** — the high-level thing the user interacts with | `Shape`, `Notification` |
| "I need power delivered" | **The bridge** — a reference held by the abstraction | `this.renderer`, `this.sender` |
| UK socket, US socket | **Implementation** — the low-level mechanism | `SVGRenderer`, `EmailSender` |
| Adding a new country | Adding **one** implementor class | `WebGLRenderer` |
| Adding a new appliance | Adding **one** abstraction class | `Triangle` |
| Number of products | `appliances + sockets`, not `appliances × sockets` | `M + N`, not `M × N` |

The whole pattern is a single sentence: **prefer composition over inheritance when two dimensions vary independently.**

---

## Key concepts inside this topic

### 1. The problem it solves — the class explosion (BAD code)

You're building a drawing library. You have shapes. You need to render them to SVG (for the web), Canvas (for the browser), and eventually WebGL (for performance).

Your first instinct is inheritance:

```javascript
// ❌ BAD — one class per (shape, renderer) combination

class Shape {
  draw() { throw new Error('not implemented'); }
}

class SVGCircle extends Shape {
  constructor(x, y, radius) {
    super();
    this.x = x; this.y = y; this.radius = radius;
  }
  area() { return Math.PI * this.radius ** 2; }
  draw() { return `<circle cx="${this.x}" cy="${this.y}" r="${this.radius}"/>`; }
}

class CanvasCircle extends Shape {
  constructor(x, y, radius) {
    super();
    this.x = x; this.y = y; this.radius = radius;   // duplicated
  }
  area() { return Math.PI * this.radius ** 2; }      // duplicated AGAIN
  draw() { return `ctx.arc(${this.x}, ${this.y}, ${this.radius}, 0, 2*Math.PI)`; }
}

class WebGLCircle extends Shape { /* ...same fields and area() a THIRD time... */ }

class SVGSquare extends Shape { /* ... */ }
class CanvasSquare extends Shape { /* ... */ }
class WebGLSquare extends Shape { /* ... */ }

// 2 shapes × 3 renderers = 6 classes.
```

Now count the pain:

| Change requested | Classes you must write |
|------------------|------------------------|
| Add `Triangle` | 3 (SVGTriangle, CanvasTriangle, WebGLTriangle) |
| Add `PDFRenderer` | 2 (PDFCircle, PDFSquare) — and 3 once Triangle exists |
| Now 3 shapes × 4 renderers | **12 classes** |
| Fix a bug in circle geometry | Edit it in 4 places |

The geometry of a circle (its `x`, `y`, `radius`, its `area()`) has **nothing to do** with whether you draw it in SVG or WebGL. But inheritance forced you to fuse the two concerns into one class. That fusion is the bug.

**The exploded hierarchy looks like this:**

```
                          ┌──────────┐
                          │  Shape   │
                          └────┬─────┘
        ┌──────────┬──────────┼──────────┬──────────┬──────────┐
        ▼          ▼          ▼          ▼          ▼          ▼
  ┌─────────┐┌──────────┐┌─────────┐┌─────────┐┌──────────┐┌─────────┐
  │SVGCircle││CanvasCirc││WebGLCirc││SVGSquare││CanvasSqr ││WebGLSqr │
  └─────────┘└──────────┘└─────────┘└─────────┘└──────────┘└─────────┘

  6 classes. Add PDF renderer → 8. Add Triangle → 12. It MULTIPLIES.
```

### 2. The fix — bridge the two hierarchies (GOOD code)

Ask: *what are the two independent things varying here?* Answer: **shape geometry** and **rendering technology**. So make two hierarchies and give the shape a *reference* to a renderer.

```javascript
// ✅ GOOD — two hierarchies joined by a reference (the "bridge")

// ── IMPLEMENTATION side: renderers know HOW to draw primitives ──
// This is the "implementor interface". In JS we use a base class whose
// methods throw — that's our stand-in for an interface.
class Renderer {
  renderCircle(x, y, radius) { throw new Error('renderCircle not implemented'); }
  renderRect(x, y, w, h)     { throw new Error('renderRect not implemented'); }
}

class SVGRenderer extends Renderer {
  renderCircle(x, y, r) { return `<circle cx="${x}" cy="${y}" r="${r}"/>`; }
  renderRect(x, y, w, h) { return `<rect x="${x}" y="${y}" width="${w}" height="${h}"/>`; }
}

class CanvasRenderer extends Renderer {
  renderCircle(x, y, r) { return `ctx.arc(${x}, ${y}, ${r}, 0, 6.283)`; }
  renderRect(x, y, w, h) { return `ctx.fillRect(${x}, ${y}, ${w}, ${h})`; }
}

class WebGLRenderer extends Renderer {
  // A real WebGL renderer tessellates the circle into triangles.
  // The point is: the Shape has NO IDEA this happens.
  renderCircle(x, y, r) { return `gl.drawArrays(TRIANGLE_FAN, circleMesh(${x},${y},${r}))`; }
  renderRect(x, y, w, h) { return `gl.drawArrays(TRIANGLE_STRIP, quad(${x},${y},${w},${h}))`; }
}

// ── ABSTRACTION side: shapes know WHAT they are, not how they're drawn ──
class Shape {
  // The bridge: a Shape HAS-A Renderer. Not IS-A.
  constructor(renderer) {
    if (new.target === Shape) throw new Error('Shape is abstract');
    this.renderer = renderer;
  }
  draw() { throw new Error('draw not implemented'); }
  // Geometry lives here ONCE, shared by every renderer.
  area() { throw new Error('area not implemented'); }
}

class Circle extends Shape {
  constructor(renderer, x, y, radius) {
    super(renderer);
    this.x = x; this.y = y; this.radius = radius;
  }
  draw() { return this.renderer.renderCircle(this.x, this.y, this.radius); }
  area() { return Math.PI * this.radius ** 2; }   // written ONCE, not 3 times
}

class Square extends Shape {
  constructor(renderer, x, y, side) {
    super(renderer);
    this.x = x; this.y = y; this.side = side;
  }
  draw() { return this.renderer.renderRect(this.x, this.y, this.side, this.side); }
  area() { return this.side ** 2; }
}

// ── Demo ──
const svg = new SVGRenderer();
const gl = new WebGLRenderer();

const shapes = [
  new Circle(svg, 10, 10, 5),
  new Square(gl, 0, 0, 20),
  new Circle(gl, 50, 50, 12),
];
for (const s of shapes) {
  console.log(s.draw(), '| area =', s.area().toFixed(2));
}
// <circle cx="10" cy="10" r="5"/> | area = 78.54
// gl.drawArrays(TRIANGLE_STRIP, quad(0,0,20,20)) | area = 400.00
// gl.drawArrays(TRIANGLE_FAN, circleMesh(50,50,12)) | area = 452.39
```

**Now count:** 2 shapes + 3 renderers = **5 classes** (plus 2 base classes). Adding `PDFRenderer` costs **1 class** and zero changes to any shape. Adding `Triangle` costs **1 class** and zero changes to any renderer.

| Approach | 2×3 | 3×4 | 5×6 | Cost of one new renderer |
|----------|-----|-----|-----|--------------------------|
| Inheritance (BAD) | 6 | 12 | 30 | N new classes (one per shape) |
| Bridge (GOOD) | 5 | 7 | 11 | **1 class** |

That is the entire value proposition: **M × N becomes M + N.**

### 3. The structure — who plays which role

The Gang-of-Four names, mapped to our code. Learn these; interviewers use them.

| GoF Role | What it does | In the shape example | In the notification example |
|----------|-------------|----------------------|-----------------------------|
| **Abstraction** | Defines the high-level interface the client uses. **Holds the reference to the Implementor.** | `Shape` | `Notification` |
| **Refined Abstraction** | A concrete variant of the abstraction. Extends it, does *not* implement the low-level work. | `Circle`, `Square` | `Alert`, `Reminder`, `Escalation` |
| **Implementor** | The interface for the low-level mechanism. Deliberately *primitive* — it doesn't mirror the Abstraction's API. | `Renderer` | `MessageSender` |
| **Concrete Implementor** | A real mechanism. | `SVGRenderer`, `CanvasRenderer`, `WebGLRenderer` | `EmailSender`, `SMSSender`, `SlackSender` |
| **The bridge** | The `this.renderer` / `this.sender` field. That's it. That's the bridge. | `Shape#renderer` | `Notification#sender` |

**The subtle rule most people miss:** the Implementor interface should be **primitive operations**, not a mirror of the Abstraction. `Renderer` exposes `renderCircle` / `renderRect` — building blocks. It does **not** expose `drawSquare()`. The Abstraction *composes* primitives into higher-level behaviour. If your Implementor has exactly the same methods as your Abstraction, you've built a pass-through wrapper, not a Bridge.

### 4. Full JavaScript implementation — Notifications × Senders

Here's the version you'll actually meet at work. You have a notification system:

- **Notification types** (abstraction): `Alert` (urgent, fires once), `Reminder` (polite, repeats), `Escalation` (retries and widens the audience).
- **Delivery channels** (implementation): Email, SMS, Slack.

These vary *completely* independently. An escalation policy has nothing to do with SMTP. Without Bridge you'd write `EmailAlert`, `SlackAlert`, `SMSAlert`, `EmailReminder`, `SlackReminder`, ... 9 classes, and adding Push Notifications makes it 12.

```javascript
// ============================================================
// IMPLEMENTATION HIERARCHY — "how does a message physically go out?"
// Primitive operations only. Knows nothing about alerts or reminders.
// ============================================================
class MessageSender {
  /** @returns {Promise<{channel:string, id:string}>} */
  async send(recipient, subject, body) {
    throw new Error('send() not implemented');
  }
  // Channels have real, different constraints. Expose them so the
  // abstraction can adapt without knowing WHICH channel it is.
  get maxLength() { return Infinity; }
  get name() { return this.constructor.name; }
}

class EmailSender extends MessageSender {
  constructor(smtpClient) { super(); this.smtp = smtpClient; }
  async send(recipient, subject, body) {
    const id = await this.smtp.sendMail({ to: recipient, subject, html: body });
    return { channel: 'email', id };
  }
}

class SMSSender extends MessageSender {
  constructor(twilioClient) { super(); this.twilio = twilioClient; }
  // SMS has a hard 160-char limit — a real constraint the abstraction must respect.
  get maxLength() { return 160; }
  async send(recipient, subject, body) {
    // SMS has no concept of a subject line, so we fold it into the text.
    const text = `${subject}: ${body}`.slice(0, this.maxLength);
    const id = await this.twilio.messages.create({ to: recipient, body: text });
    return { channel: 'sms', id };
  }
}

class SlackSender extends MessageSender {
  constructor(slackClient) { super(); this.slack = slackClient; }
  get maxLength() { return 3000; }
  async send(channelId, subject, body) {
    const res = await this.slack.chat.postMessage({
      channel: channelId, text: `*${subject}*\n${body}`,
    });
    return { channel: 'slack', id: res.ts };
  }
}

// ============================================================
// ABSTRACTION HIERARCHY — "what KIND of notification is this?"
// Holds a MessageSender. Never knows which one.
// ============================================================
class Notification {
  constructor(sender, { recipient, title, message }) {
    if (new.target === Notification) throw new Error('Notification is abstract');
    this.sender = sender;          // ◀── THE BRIDGE
    this.recipient = recipient;
    this.title = title;
    this.message = message;
  }

  // Shared helper: every notification respects the channel's length limit,
  // but no notification knows what that limit IS. That's the sender's business.
  _fit(text) {
    const max = this.sender.maxLength;
    return text.length <= max ? text : text.slice(0, max - 3) + '...';
  }

  async notify() { throw new Error('notify() not implemented'); }
}

class Alert extends Notification {
  // Fire once, loudly, immediately.
  async notify() {
    const body = this._fit(`[URGENT] ${this.message}`);
    const res = await this.sender.send(this.recipient, `🚨 ${this.title}`, body);
    console.log(`Alert delivered via ${res.channel} (${res.id})`);
    return [res];
  }
}

class Reminder extends Notification {
  constructor(sender, opts) {
    super(sender, opts);
    this.times = opts.times ?? 1;   // polite, repeated nudges
  }
  async notify() {
    const results = [];
    for (let i = 1; i <= this.times; i++) {
      const suffix = i > 1 ? ` (reminder ${i} of ${this.times})` : '';
      const body = this._fit(this.message + suffix);
      results.push(await this.sender.send(this.recipient, this.title, body));
    }
    console.log(`Reminder sent ${results.length}x via ${results[0].channel}`);
    return results;
  }
}

class Escalation extends Notification {
  // Retry, then widen the audience. Notice: this ENTIRE policy is
  // channel-agnostic. It works over email, SMS, Slack, or a channel
  // invented next year. That is the payoff.
  constructor(sender, opts) {
    super(sender, opts);
    this.chain = opts.chain ?? [];              // ['@oncall', '@lead', '@cto']
    this.retriesPerLevel = opts.retriesPerLevel ?? 2;
  }
  async notify() {
    for (const [level, target] of this.chain.entries()) {
      for (let attempt = 1; attempt <= this.retriesPerLevel; attempt++) {
        try {
          const body = this._fit(`L${level} escalation: ${this.message}`);
          const res = await this.sender.send(target, this.title, body);
          console.log(`Escalation reached ${target} via ${res.channel}`);
          return [res];                          // someone got it — stop
        } catch (err) {
          console.warn(`  attempt ${attempt} to ${target} failed: ${err.message}`);
        }
      }
    }
    throw new Error('Escalation exhausted the entire chain');
  }
}

// ============================================================
// DEMO — the same abstraction over three different implementations
// ============================================================
// Fake clients so this file runs standalone.
const fakeSmtp   = { sendMail: async () => 'msg-' + Date.now() };
const fakeTwilio = { messages: { create: async () => 'sm-' + Date.now() } };
const fakeSlack  = { chat: { postMessage: async () => ({ ts: '167.' + Date.now() }) } };

async function main() {
  const email = new EmailSender(fakeSmtp);
  const sms   = new SMSSender(fakeTwilio);
  const slack = new SlackSender(fakeSlack);

  const dbAlert = {
    title: 'DB CPU at 97%',
    message: 'Primary Postgres has been above 95% CPU for 4 minutes.',
  };

  // Same Alert class. Three channels. Zero new subclasses.
  await new Alert(email, { ...dbAlert, recipient: 'ops@acme.io' }).notify();
  await new Alert(sms,   { ...dbAlert, recipient: '+15551234567' }).notify();
  //   ^ auto-truncated to 160 chars — the Alert never knew SMS existed

  await new Reminder(slack, {
    recipient: '#eng-standup', title: 'Standup in 5',
    message: 'Join the huddle.', times: 2,
  }).notify();

  await new Escalation(slack, {
    title: 'Checkout service down',
    message: 'Error rate 100% for 3 minutes.',
    chain: ['@oncall', '@eng-lead', '@cto'],
  }).notify();
}

main().catch(console.error);
```

**Run it and count what you'd need without Bridge:** 3 notification types × 3 channels = 9 classes, and every one of them would re-implement the escalation retry loop or the reminder loop. With Bridge: 3 + 3 = 6, and the escalation logic exists exactly once.

Adding **Push notifications** tomorrow: write `class PushSender extends MessageSender`. Done. Alert, Reminder, and Escalation all support push instantly, and you changed **zero lines** in the abstraction hierarchy. That's the Open/Closed Principle actually paying rent.

### 5. A real Node.js ecosystem example

**Winston (the logging library)** is textbook Bridge.

```javascript
import winston from 'winston';

const logger = winston.createLogger({          // ◀── Abstraction
  level: 'info',
  transports: [                                 // ◀── Implementors, injected
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'app.log' }),
    new winston.transports.Http({ host: 'logs.acme.io' }),
  ],
});

logger.info('order placed', { orderId: 42 });   // abstraction calls implementor
```

- **Abstraction:** `Logger` — levels, formatting, metadata, child loggers.
- **Implementor:** `Transport` — knows only `log(info, callback)`. Primitive.
- **The bridge:** the `logger.transports` array.

`Logger` has never heard of TCP or the filesystem. `Transport` has never heard of log levels. Adding a Datadog transport doesn't touch `Logger`. Without Bridge, winston would ship `FileLoggerWithLevels`, `HttpLoggerWithLevels`, `ConsoleLoggerWithChildren`... explosion.

**Other genuine Bridges in the Node world:**

| Library | Abstraction | Implementor |
|---------|-------------|-------------|
| **Knex / Sequelize** | `QueryBuilder` (`.where().join().limit()`) | `Dialect` (Postgres, MySQL, SQLite, MSSQL) |
| **Nodemailer** | `Mailer` (compose, attach, template) | `Transport` (SMTP, SES, sendmail) |
| **Passport** | `Authenticator` (`authenticate()`, sessions) | `Strategy` (Local, OAuth2, JWT, SAML) |
| **Node `stream`** | `Readable` / `Writable` API | The `_read` / `_write` source (file, socket, memory) |

Knex is the clearest: **1 query builder × 4 SQL dialects.** Without Bridge that's `PostgresQueryBuilder`, `MySQLQueryBuilder`, ... each re-implementing `.where()` chaining. With Bridge, `.where()` is written once and each dialect just answers "how do you quote an identifier?"

### 6. When NOT to use it / how it's abused

**Don't use Bridge when:**

- **There is only one implementation and no credible second one.** `Shape` with only `SVGRenderer` is an extra layer for nothing. YAGNI. Refactoring *into* Bridge later is easy — the right interface is obvious once you have two concrete cases.
- **The two dimensions aren't actually independent.** If `Circle` ever needs `if (this.renderer instanceof WebGLRenderer)`, they're not independent and your Bridge is fake. That `instanceof` check is the smell.
- **One dimension is stable forever.** Two dimensions where one never grows is just... one dimension.

**How it gets abused:**

1. **Bridge to nowhere.** An Implementor whose methods just forward the Abstraction's methods one-for-one. That's a pass-through wrapper. The Implementor must be *primitive operations* that the Abstraction actually *composes* into something bigger.
2. **Interface bloat.** Someone grows `Renderer` to `renderCircle`, `renderRect`, `renderTriangle`, `renderBezier`, `renderText`, `renderGradient`. Now adding a shape *does* touch every renderer — the explosion is back, just relocated. Keep the Implementor interface small.
3. **Premature bridging.** Two hierarchies before you have a second implementor is speculative generality. You will guess the interface wrong.

**Rule:** you earn the right to Bridge when you can name at least **two** concrete things on *each* side.

### 7. Related patterns and how they differ (the #1 interview follow-up)

Be honest: **Bridge and Strategy look identical in code.** Both are "object holds a reference to another object and delegates to it." If you diff the JS, you'll struggle to tell them apart. The difference is **intent and structure**, not syntax.

| | **Bridge** | **Strategy** |
|--|-----------|-------------|
| **Intent** | *Structural.* Stop two hierarchies from multiplying into each other. | *Behavioural.* Swap one interchangeable algorithm. |
| **How many hierarchies?** | **Two** — both the abstraction and the implementation have subclasses. | **One and a half** — the strategies vary; the context is usually a single class. |
| **When is it wired?** | Usually at **construction**, and it usually stays. Designed **up front**, before the classes exist. | Often swapped at **runtime**, per call. Frequently retrofitted onto existing code. |
| **What varies?** | *What the thing is* AND *how it's done* — independently. | *How one specific operation is done.* |
| **The other side is...** | A whole abstraction with its own subclasses and its own logic. | A dumb algorithm object with basically one method. |
| **Example** | `Notification`(Alert/Reminder/Escalation) × `Sender`(Email/SMS/Slack) | `sort(array, comparator)` — swap `byPrice` for `byDate` |

**The tell:** if the *left* side (the thing holding the reference) also has a subclass hierarchy that keeps growing, you have a Bridge. If the left side is one class and only the right side varies, you have Strategy.

In our notification example: `Alert`, `Reminder`, `Escalation` are three refined abstractions with real, different logic. That's a hierarchy. Combined with three senders, that's a Bridge. If instead we had a single `Notification` class that took a `sender`, that would be Strategy.

**Other neighbours:**

| Pattern | Difference from Bridge |
|---------|-----------------------|
| **[Adapter](./34-pattern-adapter.md)** | Adapter makes an *existing incompatible* interface work — it's a rescue applied **after** the fact. Bridge is designed **up front**, and both sides are yours. Adapter fixes; Bridge prevents. |
| **[Abstract Factory](./30-pattern-factory-method.md)** | Often *paired* with Bridge: the factory decides which Concrete Implementor to inject. Factory creates; Bridge connects. |
| **[State](./46-pattern-state.md)** | Same code shape again (object delegates to a swappable object), but State's whole point is that the object **swaps itself out** as it changes. Bridge's implementor is stable. |
| **[Decorator](./35-pattern-decorator.md)** | Decorator wraps the *same* interface to add behaviour (stacking). Bridge connects *two different* interfaces to separate concerns. |

---

## Visual / Diagram description

### Diagram 1: The two hierarchies joined by a bridge

```
        ABSTRACTION HIERARCHY                    IMPLEMENTATION HIERARCHY
        (what the thing IS)                      (how it's actually DONE)

     ┌──────────────────────┐                   ┌──────────────────────────┐
     │      Shape           │                   │       Renderer           │
     │  (abstract)          │      the BRIDGE   │     (interface)          │
     │                      │  ═══════════════▶ │                          │
     │  # renderer ─────────┼───────────────────│  + renderCircle(x,y,r)   │
     │  + draw()            │   HAS-A, not IS-A │  + renderRect(x,y,w,h)   │
     │  + area()            │                   │                          │
     └──────────┬───────────┘                   └────────────┬─────────────┘
                │ extends                                    │ implements
       ┌────────┴────────┐              ┌──────────────┬─────┴────────┬──────────────┐
       ▼                 ▼              ▼              ▼              ▼
  ┌─────────┐      ┌──────────┐   ┌───────────┐ ┌────────────┐ ┌─────────────┐
  │ Circle  │      │  Square  │   │SVGRenderer│ │CanvasRender│ │WebGLRenderer│
  │         │      │          │   │           │ │            │ │             │
  │ x,y,r   │      │ x,y,side │   │ <circle/> │ │ ctx.arc()  │ │ gl.draw()   │
  │ draw()  │      │ draw()   │   │ <rect/>   │ │ fillRect() │ │ gl.draw()   │
  └─────────┘      └──────────┘   └───────────┘ └────────────┘ └─────────────┘

   GROWS DOWNWARD independently          GROWS DOWNWARD independently
   (add Triangle = +1 class)             (add PDFRenderer = +1 class)

   ══════════════════════════════════════════════════════════════════════
   2 + 3 = 5 classes.   Inheritance version: 2 × 3 = 6, and it MULTIPLIES.
```

**What to notice:** the two trees never touch except through that one horizontal `═══▶` line — the reference field. Each tree grows *downward* on its own. Adding a leaf on the left costs nothing on the right, and vice versa. That horizontal line is the whole pattern.

### Diagram 2: Call flow — client → abstraction → implementor

```
 ┌────────┐   1. circle.draw()        ┌──────────────┐
 │ Client │──────────────────────────▶│   Circle     │
 │ code   │                           │ (Refined     │
 └────────┘                           │  Abstraction)│
                                      └──────┬───────┘
                                             │ 2. this.renderer.renderCircle(
                                             │        this.x, this.y, this.radius)
                                             │    "I know WHAT. You know HOW."
                                             ▼
                                      ┌──────────────┐
                                      │ WebGLRenderer│
                                      │ (Concrete    │
                                      │  Implementor)│
                                      └──────┬───────┘
                                             │ 3. gl.drawArrays(...)
                                             ▼
                                      ┌──────────────┐
                                      │  GPU / DOM   │
                                      └──────────────┘

 The Client talks ONLY to the abstraction.
 The Abstraction talks ONLY to the implementor interface.
 Neither ever mentions WebGL by name after construction.
```

---

## Real world examples

### 1. JDBC / node-postgres — the driver model

The most famous Bridge in software is the **database driver**. Java's JDBC made it canonical; Node repeats it with `knex`, `sequelize`, and `typeorm`.

- **Abstraction:** the query builder / connection API — a generic SQL-shaped interface.
- **Implementor:** the *driver / dialect* — `pg`, `mysql2`, `sqlite3`, `tedious`.
- **The bridge:** the ORM holds a driver instance chosen from a connection string.

Your application writes `knex('users').where('age', '>', 21).limit(10)` exactly once. The query builder composes it, then hands primitive operations to the dialect, which knows that Postgres quotes identifiers with `"` and MySQL with `` ` ``. Swap the connection string from `postgres://` to `mysql://` and **not one line of application code changes.** That's Bridge doing its job.

### 2. React and the renderer split

React deliberately split itself into two packages, and the reason is Bridge.

- **Abstraction:** `react` — components, hooks, state, the element tree. This package knows nothing about the DOM.
- **Implementor:** `react-dom`, `react-native`, `ink` (React for terminal UIs), and custom renderers built on `react-reconciler`.

The reconciler emits primitive host operations — `createInstance`, `appendChild`, `commitUpdate`. `react-dom` turns those into DOM calls; `react-native` turns them into native view calls; `ink` turns them into ANSI escape codes.

Without this split, React would need a `DOMComponent`, a `NativeComponent`, a `TerminalComponent`... and `useState` would be re-implemented in each. Instead: components on one side, host renderers on the other, joined by the reconciler's host-config interface.

### 3. Payment orchestration at scale

Any company processing payments in multiple countries hits this dimensionally.

- **Abstraction (payment intents):** `OneTimeCharge`, `Subscription`, `Installment`, `Refund`, `Payout`. Each has real business logic — retry schedules, proration, dunning.
- **Implementor (gateways):** Stripe, Adyen, Razorpay, PayPal, a local bank rail.

5 intent types × 5 gateways = 25 classes if you inherit. Add a sixth gateway to enter a new market — a routine business event — and you'd write 5 more classes, re-implementing dunning logic in each. With Bridge: `PaymentIntent` holds a `PaymentGateway`, the gateway exposes primitives (`authorize`, `capture`, `refund`, `void`), and the new market costs **one** class. This is why every mature payments team builds a gateway-agnostic core.

---

## Trade-offs

| Pros | What you actually get |
|------|----------------------|
| **Kills the class explosion** | `M + N` classes instead of `M × N`. At 8×8 that's 16 vs 64. |
| **Both sides evolve independently** | New renderer = 1 class, zero edits to shapes. Open/Closed Principle, for real. |
| **Logic written once** | Circle geometry, escalation retry loops — implemented one time, not once per implementor. |
| **Testable** | Inject a `FakeSender` and unit-test `Escalation`'s retry logic with zero network calls. |
| **Swappable at config or runtime** | `shape.renderer = webgl`. The client never says "WebGL" by name. |

| Cons | The cost you pay |
|------|------------------|
| **More indirection** | `draw()` now hops through two objects. Deeper stack traces; harder for newcomers to follow. |
| **More upfront design** | You must correctly identify the two dimensions *and* design a primitive Implementor interface. Guess wrong and you refactor. |
| **Premature abstraction risk** | With one implementor, Bridge is pure overhead. |
| **Leaky when dimensions aren't independent** | An `instanceof` check inside the abstraction means the split was wrong. |
| **Implementor interface is hard to change** | Adding a method to `Renderer` forces every concrete renderer to implement it — the old pain, relocated. |
| **Tiny runtime cost** | One extra property lookup + call per operation. Irrelevant except in a hot render loop. |

**Rule of thumb:** Reach for Bridge when you can point at **two dimensions that both have at least two members today and will credibly grow tomorrow** — and when a change in one has no business forcing a change in the other. One shape and one renderer? Write the simple class. Five shapes and a third renderer incoming? Bridge it *now*, before you write the 15th class.

---

## Common interview questions on this topic

### Q1: "What's the difference between Bridge and Strategy? They look like the same code."
**Hint:** Admit the code is nearly identical — that earns you credit, because it's true. Then give the two real differences. (1) **Structure:** Bridge has *two* hierarchies that both grow; Strategy has one context and a family of algorithms. (2) **Intent:** Bridge is structural — it exists to stop `M × N` class explosion between two independently-varying dimensions, and it's designed up front. Strategy is behavioural — it exists to swap one algorithm at runtime, and it's often retrofitted. Concrete test: "does the object *holding* the reference also have a growing subclass tree?" Yes → Bridge. No → Strategy.

### Q2: "Show me the class explosion and how Bridge fixes it."
**Hint:** Draw the 2×3 = 6-class shape hierarchy, then say the magic sentence: *"adding a renderer costs N classes, adding a shape costs M classes — it's quadratic."* Redraw as two hierarchies with a `has-a` arrow. State the result: 2 + 3 = 5 classes, and a new renderer now costs exactly one. Then generalize: `M × N → M + N`.

### Q3: "How is Bridge different from Adapter? Both have an object delegating to another."
**Hint:** **Timing and ownership.** Adapter is *reactive* — you have an existing class with the wrong interface (a third-party library, a legacy module) and you wrap it to fit. Adapter is applied after the fact to code you often don't control. Bridge is *proactive* — you design both hierarchies yourself, up front, to keep them decoupled. Adapter fixes an incompatibility; Bridge prevents a coupling.

### Q4: "What should go in the Implementor interface?"
**Hint:** **Primitive operations**, not a mirror of the Abstraction. `Renderer` should expose `renderCircle` / `renderRect` — building blocks the abstraction *composes*. If `Renderer` had a `drawSquare()` method, the abstraction would be a hollow pass-through and you'd have gained nothing. Keep the Implementor interface small: every method you add is a method every concrete implementor must write.

### Q5: "When would you NOT use Bridge?"
**Hint:** When there's only one implementation and no credible second one (YAGNI — you'll guess the interface wrong). When the two dimensions aren't truly independent (you'll end up with `instanceof` checks inside the abstraction — the smell that the split is fake). When one dimension is frozen forever. Bridge's cost is indirection plus upfront design; it only pays off when both dimensions actually grow.

---

## Practice exercise

### The Report Exporter Bridge

You're building a reporting module. Today it works like this — and it is exploding:

```
SalesReportPDF, SalesReportCSV, SalesReportHTML,
InventoryReportPDF, InventoryReportCSV, InventoryReportHTML,
PayrollReportPDF, PayrollReportCSV, PayrollReportHTML
```

9 classes. Product now wants an **Excel** exporter and a **Compliance** report. Write it out: that's 3 more + 4 more = **16 classes.**

**Your task (~30 minutes):**

1. **Identify the two dimensions.** Write them down and name the abstraction and the implementor.
2. **Design the Implementor interface** (`Exporter`). Keep it **primitive** — resist the urge to add `exportSalesReport()`. Three or four methods maximum: think `header(cols)`, `row(values)`, `footer(summary)`, `finish()`.
3. **Write the JavaScript:** an `Exporter` base class plus `CSVExporter`, `JSONExporter`, `HTMLExporter` (skip real PDF — just emit a string). Then a `Report` abstract class holding an `exporter`, plus `SalesReport` and `InventoryReport`. Each report defines *what data it fetches and how it summarises*, never *how it's formatted*.
4. **Write a `main()` demo** that builds a `SalesReport` with a `CSVExporter` and an `InventoryReport` with an `HTMLExporter`, and prints both.
5. **Then prove it:** add `MarkdownExporter`. Count the lines you changed in `SalesReport` and `InventoryReport`. The answer must be **zero**. If it isn't, your Implementor interface isn't primitive enough — go fix it.
6. **Do the arithmetic** in a comment at the top: how many classes does the inherited version need at 4 reports × 4 formats? How many does your Bridge need? Write both numbers.

**What to produce:** one runnable `.js` file, roughly 120–160 lines, that prints three differently-formatted reports — from report classes you never touched after step 3.

---

## Quick reference cheat sheet

- **Bridge in one line:** split one hierarchy into two — *abstraction* and *implementation* — and connect them with a reference so they vary independently.
- **The math:** turns `M × N` classes into `M + N`. That is the entire reason the pattern exists.
- **The trigger:** you notice class names with **two nouns glued together** — `SVGCircle`, `EmailAlert`, `SalesReportPDF`. That's an explosion in progress.
- **HAS-A, not IS-A:** the abstraction *holds* an implementor. Composition over inheritance, applied to whole hierarchies.
- **The bridge is literally one field:** `this.renderer` / `this.sender`. There's no magic.
- **Implementor = primitive operations.** `renderCircle(x,y,r)`, not `drawTheWholeShape()`. The abstraction composes primitives.
- **Abstraction owns the WHAT; Implementor owns the HOW.** If the abstraction ever asks *which* implementor it has (`instanceof`), the design is broken.
- **vs Strategy:** identical code, different intent. Bridge = two growing hierarchies, structural, designed up front. Strategy = one context, swappable algorithm, behavioural, often retrofitted.
- **vs Adapter:** Adapter is a rescue for an interface you don't control, applied afterwards. Bridge is a prevention you design in advance for code you own.
- **Node examples:** Winston (`Logger` × `Transport`), Knex (`QueryBuilder` × `Dialect`), Nodemailer (`Mailer` × `Transport`), Passport (`Authenticator` × `Strategy`), React (`react` × `react-dom`/`react-native`).
- **Don't bridge with one implementor.** You'll guess the interface wrong. Wait for the second concrete case.
- **The payoff test:** adding a new implementor must cost **exactly one class and zero edits elsewhere.** If it costs more, you don't have a Bridge.
- **Interview move:** always volunteer the Strategy comparison before they ask. It's the guaranteed follow-up.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [38 — Composite Pattern](./38-pattern-composite.md) — the other structural pattern for taming hierarchies, by making trees uniform |
| **Next** | [40 — Flyweight Pattern](./40-pattern-flyweight.md) — the structural pattern for taming *object count* instead of *class count* |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) — the pattern Bridge is constantly confused with; read them back to back |
| **Related** | [33 — Adapter Pattern](./34-pattern-adapter.md) — same delegation shape, opposite timing: a rescue, not a prevention |
| **Related** | [28 — SOLID Principles](./14-solid-single-responsibility.md) — Bridge is the Open/Closed and Dependency Inversion principles made concrete |
