# 48 — Mediator Pattern

## Category: LLD Patterns

---

## What is this?

The **Mediator pattern** stops objects from talking to each other directly and makes them talk **through a central object** instead. Every object knows only one thing: the mediator. The mediator knows everyone.

Think of an **airport control tower**. Planes approaching a busy airport do not radio each other to negotiate who lands first ("Hey Lufthansa 402, I'm at 3000 feet, mind if I cut in?"). That would be chaos — every plane would need a radio channel to every other plane. Instead, every plane talks only to the **tower**, and the tower tells each plane what to do. The planes are decoupled from each other; they are coupled only to the tower.

---

## Why does it matter?

Because **direct object-to-object references are the #1 source of unmaintainable code**, and they grow quadratically.

If you have 5 objects that each need to react to each other, you have up to **5 × 4 = 20 directed references** (10 undirected connections). At 10 objects it's 90. Every new object you add means editing every existing object. Change one class and five others break. This is called a **mesh** or **spaghetti** dependency graph.

**What breaks without it:**
- A UI form where the checkbox knows about the text field, which knows about the submit button, which knows about the dropdown... and adding a new widget means editing four files.
- A chat room where every `User` holds an array of every other `User`. Now implement "mute a user", "ban a user", "private message". Every rule has to be duplicated in every user object.
- Game entities that each hold references to each other so they can react to collisions.

**Interview angle:** Mediator shows up constantly as the answer to "how would you decouple these components?" It's also the classic **"contrast this with Observer"** question — interviewers love it because most candidates confuse the two.

**Real-work angle:** You have already used Mediator without naming it. Redux stores, Express's app object, Kubernetes' control plane, a message broker sitting between microservices — all mediators. Naming the pattern lets you reason about its cost (the mediator becomes a god object) instead of stumbling into it.

---

## The core idea — explained simply

### The Air Traffic Control Analogy

It's 4pm at Heathrow. Twelve aircraft are inbound, three are taxiing, two want to take off.

**The world WITHOUT a control tower:**

Every pilot has a radio and must coordinate with every other pilot. British Airways 117 must ask Emirates 8, KLM 1024, Lufthansa 402, and nine others whether it's safe to descend. Emirates 8 must do the same. If a new plane enters the airspace, it must establish contact with *everyone*. If one pilot's radio breaks, the whole negotiation collapses. Adding a new rule ("cargo planes yield to passenger planes") means retraining every single pilot.

**The world WITH a control tower:**

Every pilot talks to exactly one entity: the tower. "Heathrow tower, BA117 requesting descent." The tower — which knows every plane's position, fuel state, and priority — replies "BA117, descend to 4000, hold at Lambourne." Pilots never speak to each other. A new plane needs one radio frequency. A new rule is implemented in *one place*: the tower.

**The cost, honestly:** the tower is now the single most complicated, most critical, most knowledge-loaded thing in the system. If the tower goes down, nothing flies. That trade-off is the whole pattern.

**Mapping the analogy:**

| Air traffic control | Mediator pattern | In the chat-room example |
|---|---|---|
| Aircraft | **Colleague** (a component) | `User` |
| The control tower | **Mediator** | `ChatRoom` |
| Radio frequency to the tower | Colleague's reference to the mediator | `this.room` |
| "Tower, requesting descent" | `mediator.notify(sender, event)` | `room.send(this, text)` |
| Tower's rulebook (priority, spacing) | The mediator's coordination logic | mute lists, broadcast rules |
| Planes never radio each other | Colleagues hold **zero** references to each other | `User` has no array of other users |
| Tower knows everything about everyone | The mediator risks becoming a **god object** | `ChatRoom` grows fat |

---

## Key concepts inside this topic

### 1. The problem it solves — start with the painful code

Here is a login form with four widgets. It's the code every frontend dev has written and later regretted.

```javascript
// ---------- BAD: widgets reference each other directly (a mesh) ----------

class Checkbox {
  constructor(label) {
    this.label = label;
    this.checked = false;
    // To do its job, the checkbox must KNOW about other widgets:
    this.usernameField = null;
    this.passwordField = null;
    this.submitButton = null;
    this.statusLabel = null;
  }

  toggle() {
    this.checked = !this.checked;
    // "Remember me" mode: reveal an extra field, relabel the button, update status.
    if (this.checked) {
      this.usernameField.enable();
      this.submitButton.setText('Sign in & remember');
      this.statusLabel.setText('We will keep you logged in.');
    } else {
      this.submitButton.setText('Sign in');
      this.statusLabel.setText('');
    }
    // And it must re-run the validity rules that "belong" to the button:
    this.submitButton.setEnabled(
      this.usernameField.value.length > 0 && this.passwordField.value.length >= 8
    );
  }
}

class TextField {
  constructor(name) {
    this.name = name;
    this.value = '';
    this.submitButton = null;   // needs the button, to enable/disable it
    this.statusLabel = null;    // needs the label, to show errors
    this.peerField = null;      // needs the OTHER field, to check combined validity
  }

  setValue(v) {
    this.value = v;
    // Duplicated validity rule — the SAME logic as in Checkbox.toggle(). Now it lives twice.
    const ok = this.value.length > 0 && this.peerField.value.length >= 8;
    this.submitButton.setEnabled(ok);
    this.statusLabel.setText(ok ? '' : 'Password must be 8+ characters.');
  }
}
```

Count the problems:

1. **Wiring hell.** Every widget needs setters for every other widget. Construction is a 12-line ceremony.
2. **Duplicated rules.** The "is the form valid?" rule is written in `Checkbox` *and* in `TextField`. Change it and you must find every copy.
3. **Unreusable widgets.** `TextField` is hardcoded to a login form. You cannot drop it into a signup form.
4. **Untestable.** To unit-test the checkbox you must construct four other widgets.
5. **Quadratic growth.** Add a "Show password" toggle: you now edit all four existing classes.

The rule the mesh violates: **a component should know how to do its own job, not how the whole screen behaves.**

### 2. The structure — who plays which role

Mediator has only three participants. This is one of the simplest GoF patterns *structurally* — the difficulty is knowing when to reach for it.

| Role | What it does | In the chat example |
|---|---|---|
| **Mediator** (interface) | Declares one method the colleagues call, usually `notify(sender, event, payload)` | `ChatRoom` base class |
| **ConcreteMediator** | Holds references to all colleagues and contains ALL the coordination logic | `ModeratedChatRoom` |
| **Colleague** | Holds a reference to the mediator only. Notifies it on state change. Never references another colleague. | `User` |

```
   Colleague ──── mediator ────▶ Mediator (interface)
                                    ▲
                                    │ implements
                              ConcreteMediator
                                    │
                    holds refs to all colleagues (but colleagues
                    hold NO refs to each other)
```

The direction of dependencies matters: **colleagues depend on the mediator interface** (abstract, stable), **the concrete mediator depends on the colleagues** (concrete, changing). That's classic dependency inversion.

### 3. Full JavaScript implementation — a chat room

The canonical example. Notice: `User` never holds a reference to another `User`.

```javascript
// ============================================================
//  MEDIATOR INTERFACE
// ============================================================

/**
 * Base mediator. In JS we have no `interface` keyword, so we use a base class
 * whose methods throw — the standard "abstract method" idiom.
 */
class ChatMediator {
  register(user) { throw new Error('register() not implemented'); }
  send(sender, text, toName = null) { throw new Error('send() not implemented'); }
}

// ============================================================
//  COLLEAGUE
// ============================================================

class User {
  #room;                       // the ONLY outward reference this object has

  constructor(name, room) {
    this.name = name;
    this.#room = room;
    this.inbox = [];
    room.register(this);       // the colleague announces itself to the mediator
  }

  // A user "requests" — it never decides who receives the message.
  say(text) {
    this.#room.send(this, text);
  }

  whisper(text, toName) {
    this.#room.send(this, text, toName);
  }

  // The mediator calls this. The user just displays what it is given.
  receive(fromName, text, isPrivate = false) {
    const line = `${isPrivate ? '(private) ' : ''}${fromName}: ${text}`;
    this.inbox.push(line);
    console.log(`  [${this.name}] ${line}`);
  }
}

// ============================================================
//  CONCRETE MEDIATOR — all coordination logic lives HERE
// ============================================================

class ModeratedChatRoom extends ChatMediator {
  #users = new Map();          // name -> User
  #muted = new Set();          // names who may not broadcast
  #banned = new Set();

  register(user) {
    this.#users.set(user.name, user);
    // Coordination rule #1: announce joins to everyone already present.
    for (const other of this.#users.values()) {
      if (other !== user) other.receive('system', `${user.name} joined`);
    }
  }

  mute(name)  { this.#muted.add(name); }
  ban(name)   { this.#banned.add(name); this.#users.delete(name); }

  send(sender, text, toName = null) {
    // ALL the rules live in one place. Adding a rule = editing one file.
    if (this.#banned.has(sender.name)) return;
    if (this.#muted.has(sender.name) && !toName) {
      sender.receive('system', 'You are muted and cannot broadcast.');
      return;
    }
    if (text.length > 200) {
      sender.receive('system', 'Message too long (max 200 chars).');
      return;
    }

    if (toName) {                                   // private message
      const target = this.#users.get(toName);
      if (!target) return sender.receive('system', `No such user: ${toName}`);
      return target.receive(sender.name, text, true);
    }

    for (const user of this.#users.values()) {      // broadcast
      if (user !== sender) user.receive(sender.name, text);
    }
  }
}

// ============================================================
//  DEMO
// ============================================================

function main() {
  const room = new ModeratedChatRoom();

  const alice = new User('alice', room);
  const bob   = new User('bob', room);
  const carol = new User('carol', room);

  console.log('--- normal broadcast ---');
  alice.say('hey everyone');

  console.log('--- private message ---');
  bob.whisper('psst, standup is at 10', 'alice');

  console.log('--- moderation: carol gets muted ---');
  room.mute('carol');
  carol.say('spam spam spam');       // blocked by the mediator, not by carol

  console.log('--- carol can still whisper ---');
  carol.whisper('am I muted?', 'alice');

  console.log('\nalice inbox:', alice.inbox);
}

main();
```

Run it with `node chat.js`. The output shows joins, a broadcast, a whisper, and a blocked message.

**The key line to point at in an interview:** `class User` contains **no reference to any other `User`**. Adding a fourth user changes zero lines of `User`. Adding a "no links in first message" rule changes zero lines of `User`.

### 4. Second implementation — the form dialog (the mesh, fixed)

Same idea, applied to the painful code from concept 1.

```javascript
// ---------- GOOD: widgets know only the mediator ----------

class Widget {
  constructor(name, mediator) {
    this.name = name;
    this.mediator = mediator;
    mediator.register(this);
  }
  // Every widget reports events upward. It never reaches sideways.
  changed(event, payload) {
    this.mediator.notify(this, event, payload);
  }
}

class TextField extends Widget {
  constructor(name, mediator) { super(name, mediator); this.value = ''; this.enabled = true; }
  setValue(v) { this.value = v; this.changed('input'); }
  enable()    { this.enabled = true; }
}

class Checkbox extends Widget {
  constructor(name, mediator) { super(name, mediator); this.checked = false; }
  toggle() { this.checked = !this.checked; this.changed('toggle'); }
}

class Button extends Widget {
  constructor(name, mediator) { super(name, mediator); this.text = 'Sign in'; this.enabled = false; }
  setEnabled(v) { this.enabled = v; }
  setText(t)    { this.text = t; }
  click()       { if (this.enabled) this.changed('click'); }
}

class Label extends Widget {
  constructor(name, mediator) { super(name, mediator); this.text = ''; }
  setText(t) { this.text = t; }
}

/** The ConcreteMediator: knows every widget, owns every rule. */
class LoginDialog {
  #widgets = new Map();
  register(w) { this.#widgets.set(w.name, w); }
  get(n)      { return this.#widgets.get(n); }

  notify(sender, event) {
    const user   = this.get('username');
    const pass   = this.get('password');
    const remember = this.get('remember');
    const submit = this.get('submit');
    const status = this.get('status');

    // ONE definition of "the form is valid". Not duplicated anywhere.
    const valid = user.value.length > 0 && pass.value.length >= 8;

    if (event === 'input' || event === 'toggle') {
      submit.setEnabled(valid);
      submit.setText(remember.checked ? 'Sign in & remember' : 'Sign in');
      status.setText(valid ? '' : 'Password must be 8+ characters.');
    }

    if (event === 'click' && sender === submit) {
      status.setText(`Signing in ${user.value}${remember.checked ? ' (remembered)' : ''}...`);
    }
  }
}

// ---- demo ----
const dialog   = new LoginDialog();
const username = new TextField('username', dialog);
const password = new TextField('password', dialog);
const remember = new Checkbox('remember', dialog);
const submit   = new Button('submit', dialog);
const status   = new Label('status', dialog);

username.setValue('parshuram');
console.log(status.text, '| button enabled:', submit.enabled);   // needs a password → disabled

password.setValue('hunter2!!');
remember.toggle();
console.log('button:', submit.text, '| enabled:', submit.enabled);

submit.click();
console.log(status.text);
```

Compare with the bad version: `TextField` is now **generic** — you can reuse it in a signup form, a search bar, anywhere. The form-specific behaviour lives in `LoginDialog` and nowhere else.

### 5. A real Node.js ecosystem example

**`EventEmitter` used as a bus, and Redux, are mediators.**

- **Node's `EventEmitter` as a shared bus.** When modules `emit` and `on` against a single shared emitter instead of importing each other, that emitter is a mediator. Note the difference from Observer (next section): in Observer the *subject* pushes to *its own* subscribers; used as a mediator, the emitter is a neutral third party that neither side owns.

- **Redux.** Components never call each other's setters. A component `dispatch`es an action to the **store** (the mediator). The store's reducers hold all the coordination rules and produce new state, which flows back down. Redux even has the classic mediator smell: the root reducer becomes a giant switch that knows about everything.

- **Express's `app` object.** Routers, middleware, and error handlers do not reference each other; they all register on `app`, and `app` sequences them. (The *chaining* is Chain of Responsibility — topic 47 — but the registration hub is mediator-flavoured.)

- **Message brokers at the HLD level.** Kafka/RabbitMQ between microservices is the same idea one zoom level up: services publish to the broker instead of calling each other, turning an n×n service mesh into a hub-and-spoke.

### 6. When NOT to use it / how it's abused

**The god object.** This is the pattern's built-in failure mode and you must say it out loud in an interview. The mediator absorbs every rule that involves more than one colleague. Over two years, `ChatRoom` becomes a 2,000-line class that knows about muting, banning, emoji, threads, reactions, read receipts, and rate limits. You didn't remove the complexity — you **moved it**. You traded *many small coupled classes* for *one huge class everything depends on*.

Mitigations:
- Split by concern: `ModerationMediator`, `PresenceMediator` — several small mediators instead of one giant one.
- Keep colleagues dumb but keep the mediator **thin**: it should *route and coordinate*, and delegate real logic to services.
- If the mediator's `notify()` is a 200-line if/else chain, consider a Strategy (topic 42) or a rules table keyed by event.

**Don't use it when:**
- You only have **2-3 components** with simple links. A mediator adds indirection you don't need. The mesh is only painful once it's actually a mesh.
- The relationship is genuinely **one-to-many broadcast** with no coordination logic. That's Observer, not Mediator.
- Components are already decoupled by a queue or an API boundary.

**Don't over-abstract:** in the frontend world, "lift state up to the parent component" *is* the mediator pattern, and React devs have used it for a decade without ceremony. You don't always need a `Mediator` base class.

### 7. Related patterns and how they differ (the #1 interview follow-up)

**Mediator (48) vs Observer (41)** — the classic confusion.

| | **Observer (41)** | **Mediator (48)** |
|---|---|---|
| Core intent | One subject notifies **its own** dependents when it changes | Many peers coordinate **through a third party** |
| Direction | One-to-many, **one-way** (subject → observers) | Many-to-many, **two-way** (colleague ↔ mediator ↔ colleague) |
| Who owns the logic? | Each observer decides what to do with the event | The mediator decides who is affected and how |
| Who knows whom? | Subject holds a list of observers; observers know the subject's event shape | Colleagues know only the mediator; mediator knows all |
| Typical shape | `emitter.on('order:paid', handler)` | `room.send(user, text)` → room decides recipients |
| Example | A price change notifies 3 dashboards | A chat room enforces mute rules and routes messages |

The one-line answer: **Observer broadcasts; Mediator arbitrates.** Observer has no opinion about what happens next — it just fans out. Mediator contains *rules* about who should react and how. And they compose: a mediator often *uses* Observer internally to talk to its colleagues.

**Mediator (48) vs Facade (36).**

| | **Facade (36)** | **Mediator (48)** |
|---|---|---|
| Intent | Simplify access to a **complex subsystem** | Decouple **peers** from each other |
| Traffic direction | **One-way**: client → facade → subsystem. The subsystem doesn't know the facade exists. | **Two-way**: colleagues call the mediator, the mediator calls colleagues back. |
| Do the inner objects know about it? | No — a facade can be bolted onto legacy code that is unaware of it | Yes — colleagues explicitly hold a mediator reference |
| Adds behaviour? | No, it only re-packages existing calls | Yes, it *contains* the coordination rules |

One-liner: **a facade is a receptionist you call; a mediator is a switchboard the staff also call you through.**

**Others:**
- **Command (43):** requests to the mediator can be reified as Command objects, giving you undo and logging of coordination events.
- **Chain of Responsibility (47):** also decouples sender from receiver, but linearly — the request travels down a chain until someone handles it. Mediator is a hub, not a line.
- **Singleton (29):** mediators are often single instances per scope (one chat room, one store). Beware of making that a global singleton — it makes testing miserable.

---

## Visual / Diagram description

### Diagram 1: BEFORE — the mesh (5 components, 10 connections)

```
                      ┌───────────┐
             ┌────────│  Button   │────────┐
             │        └─────┬─────┘        │
             │              │              │
             │        ┌─────┴─────┐        │
      ┌──────┴─────┐  │           │  ┌─────┴──────┐
      │  Checkbox  │──┼───────────┼──│   Label    │
      └──────┬─────┘  │           │  └─────┬──────┘
             │  ╲     │           │     ╱  │
             │   ╲    │           │    ╱   │
             │    ╲   │           │   ╱    │
      ┌──────┴─────┐ ╲│           │╱ ┌─────┴──────┐
      │  Username  │──┼───────────┼──│  Password  │
      └────────────┘  └───────────┘  └────────────┘

  5 components, every pair connected  =  5×4/2 = 10 connections
  Add a 6th widget  →  6×5/2 = 15 connections (+5 classes to edit)
  Growth is O(n²).
```

### Diagram 2: AFTER — the hub (5 components, 5 connections)

```
                      ┌───────────┐
                      │  Button   │
                      └─────┬─────┘
                            │
      ┌────────────┐        │        ┌────────────┐
      │  Checkbox  │───┐    │    ┌───│   Label    │
      └────────────┘   │    │    │   └────────────┘
                       ▼    ▼    ▼
                   ┌─────────────────┐
                   │    MEDIATOR     │   ◀── all rules live here
                   │  (LoginDialog)  │
                   └─────────────────┘
                       ▲         ▲
      ┌────────────┐   │         │   ┌────────────┐
      │  Username  │───┘         └───│  Password  │
      └────────────┘                 └────────────┘

  5 components, 5 connections  =  O(n)
  Add a 6th widget  →  6 connections (+0 classes to edit)
  Every arrow is BIDIRECTIONAL: widget → mediator (notify),
  mediator → widget (setEnabled, setText).
```

**What the two diagrams show:** the pattern *is* this picture. On the left, edges grow as n²; the code that implements those edges is spread across every class. On the right, edges grow as n; all the logic is concentrated in one box. The whole trade-off is visible: the mesh distributes complexity but multiplies it; the hub concentrates complexity but linearises it. **If that centre box gets too fat, you have a god object — that's the price of the picture on the right.**

### Diagram 3: Class diagram

```
┌──────────────────────┐            ┌───────────────────────────┐
│   «interface»        │            │        Widget             │
│   Mediator           │◀───────────│  (Colleague, abstract)    │
│                      │  mediator  │                           │
│ + notify(sender, ev) │            │  # mediator : Mediator    │
└──────────┬───────────┘            │  + changed(event)         │
           │                        └────────────┬──────────────┘
           │ implements                          │ extends
           │                        ┌────────────┼────────────┬─────────────┐
┌──────────▼───────────┐            │            │            │             │
│   LoginDialog        │      ┌─────▼─────┐ ┌────▼─────┐ ┌────▼───┐  ┌──────▼────┐
│  (ConcreteMediator)  │      │ TextField │ │ Checkbox │ │ Button │  │   Label   │
│                      │      └───────────┘ └──────────┘ └────────┘  └───────────┘
│  - widgets : Map     │           ▲             ▲            ▲             ▲
│  + register(w)       │           │             │            │             │
│  + notify(sender,ev) │───────────┴─────────────┴────────────┴─────────────┘
└──────────────────────┘        holds references to all colleagues
                                (colleagues hold NONE to each other)
```

---

## Real world examples

### 1. Redux (and the Flux architecture at Facebook)

Facebook created Flux in 2014 precisely because their MVC views had become a mesh — a view updating a model that updated another view that updated another model, with cascading updates nobody could trace. Flux's answer: components never talk to each other. They **dispatch** an action to a central **store**, the store applies reducers, and new state flows down. That store is a textbook mediator: unidirectional inputs, centralized rules, components decoupled from each other. It also exhibits the god-object symptom, which is why large Redux apps split reducers by slice — several smaller mediators instead of one.

### 2. Kubernetes control plane

Pods, kubelets, schedulers, and controllers do not talk to each other. Every component talks to the **API server**, which is backed by etcd. The kubelet on node 7 does not call the scheduler; it watches the API server. The scheduler does not call kubelets; it writes a binding to the API server. This is hub-and-spoke on an infrastructure scale — you can add a new controller without touching any existing component, which is exactly the O(n) property from diagram 2.

### 3. Air traffic control / Slack-style chat backends

Real chat backends generalize the `ChatRoom` example: clients hold a WebSocket to a **channel service** and never to each other. The channel service is the mediator that applies membership rules, mutes, rate limits, and fan-out. In peer-to-peer chat (the mesh), n clients need n² connections — which is exactly why WebRTC group calls above a handful of participants switch from a peer mesh to an **SFU** (Selective Forwarding Unit): a media mediator.

---

## Trade-offs

| Pros | What you gain |
|---|---|
| **Decoupling** | Colleagues have zero knowledge of each other; they are reusable in other contexts |
| **Single Responsibility** | Coordination logic lives in one place instead of being smeared across n classes |
| **Open/Closed** | Add a new colleague or a new rule without editing existing colleagues |
| **O(n) connections** | Wiring cost grows linearly instead of quadratically |
| **Testability** | Test a colleague with a fake mediator; test the mediator with fake colleagues |

| Cons | What you pay |
|---|---|
| **God object risk** | The mediator accumulates every cross-component rule and becomes huge |
| **Single point of failure / bottleneck** | Everything routes through one object; a bug there breaks everything |
| **Indirection** | Reading the code, you can no longer see "who reacts to this" at the call site — you must go to the mediator |
| **Over-engineering for small n** | With 2-3 components, the mediator is pure ceremony |
| **Hidden coupling** | The mediator's `notify()` string events are an untyped contract; a typo fails silently |

| Situation | Use Mediator? |
|---|---|
| 5+ components with cross-cutting rules | Yes |
| Components need to be reused elsewhere | Yes |
| Simple one-way broadcast, no rules | No — use Observer (41) |
| Just want to simplify an API surface | No — use Facade (36) |
| 2 components, one link | No — direct reference is fine |

**The sweet spot:** reach for Mediator when the *number of relationships* is what's hurting, not the number of objects — and keep the mediator thin by delegating real work to services. **Rule of thumb: if you can't add a new component without editing existing ones, you need a mediator. If your mediator has grown past ~200 lines, split it.**

---

## Common interview questions on this topic

### Q1: "What's the difference between Mediator and Observer?"
**Hint:** Observer = one-to-many, one-way broadcast; the subject owns its subscribers and has no opinion about what they do. Mediator = many-to-many, two-way; a neutral third party that *contains the coordination rules* and decides who is affected. Observer *notifies*; Mediator *arbitrates*. They compose — a mediator often uses Observer internally to reach its colleagues.

### Q2: "Mediator centralises everything. Isn't that just moving the mess?"
**Hint:** Yes, partially — and the honest answer scores points. You trade O(n²) distributed coupling for O(n) coupling plus one class with high cohesion of *coordination* logic. That's a win *if* the mediator stays a router. It becomes a loss when the mediator absorbs business logic and turns into a god object. Mitigations: multiple focused mediators, delegate to services, drive `notify()` from a rules table rather than a giant if/else.

### Q3: "Is Mediator the same as Facade?"
**Hint:** No. Facade is one-way — clients call it, the subsystem doesn't know it exists, and it adds no new behaviour. Mediator is two-way — the colleagues explicitly hold a reference to it, and it *adds* the coordination logic that used to live between them.

### Q4: "Give me a mediator you've used without knowing it."
**Hint:** Redux store, React "lift state to the common parent", Express's `app`, a shared `EventEmitter` bus, an API gateway, a Kafka broker between microservices, Kubernetes' API server, a WebRTC SFU. Any of these shows you understand it beyond the GoF book.

### Q5: "How would you stop the mediator from becoming a god object?"
**Hint:** (a) Split by concern — `ModerationMediator` + `PresenceMediator` instead of one `ChatRoom`. (b) Keep the mediator as routing only; push domain logic into services it calls. (c) Replace the if/else in `notify()` with a `Map<event, handler>` table so each rule is its own small function. (d) Watch for the smell: if the mediator needs to read a colleague's internals to decide, your colleague API is too thin.

---

## Practice exercise

### The Smart Home Mediator

Build a smart-home controller in Node. You have five devices, and today they call each other directly (the mesh).

**Devices and their rules:**
- `MotionSensor` — fires `motion` events
- `LightBulb` — `turnOn()` / `turnOff()`
- `Thermostat` — `setTemp(c)`, exposes `currentTemp`
- `SecurityCamera` — `startRecording()` / `stopRecording()`
- `Alarm` — `arm()` / `disarm()`, exposes `armed`

**The coordination rules (all of them must live in the mediator, not in the devices):**
1. Motion detected + alarm armed → start recording AND sound the alarm.
2. Motion detected + alarm disarmed → turn the light on, and turn it off after 30 seconds of no motion (simulate with `setTimeout`).
3. Thermostat reads below 18°C → turn the light on (people are home and cold) only if the alarm is disarmed.
4. Arming the alarm → turn all lights off and start recording.

**What to produce:**
1. First write the **bad version** with direct references and count how many references each device needs. Write the number in a comment.
2. Then write the **mediator version**: a `SmartHomeHub` with `register(device)` and `notify(sender, event, payload)`. Devices must have **exactly one** outward reference — the hub.
3. Add a **sixth** device (`DoorLock`, with rule: arming the alarm also locks the door) and count how many *existing* classes you had to edit. It should be **zero**.
4. Finish with a `main()` that arms the alarm, fires motion, and prints the resulting device states.

That step-3 count is the whole point of the pattern. Write it down.

---

## Quick reference cheat sheet

- **Intent:** define an object that encapsulates how a set of objects interact, so they don't refer to each other explicitly.
- **The picture:** turn an n×n **mesh** into a **hub-and-spoke** — O(n²) connections become O(n).
- **Three roles:** `Mediator` (interface) → `ConcreteMediator` (holds all colleagues + all rules) ← `Colleague` (holds only the mediator).
- **The golden test:** can a colleague be constructed and tested without constructing any other colleague? If yes, you have a real mediator.
- **Colleagues report, they don't decide:** `mediator.notify(this, 'input')` — never `otherWidget.setEnabled(false)`.
- **The cost:** the mediator becomes a **god object**. Say this out loud in interviews; it's the expected follow-up.
- **vs Observer (41):** Observer broadcasts one-to-many, one-way, no rules. Mediator arbitrates many-to-many, two-way, with rules.
- **vs Facade (36):** Facade is a one-way simplifying wrapper the subsystem doesn't know about. Mediator is a two-way hub the colleagues explicitly depend on.
- **vs Chain of Responsibility (47):** CoR passes a request down a line; Mediator routes from a hub.
- **In the wild:** Redux store, React lift-state-up, Express `app`, EventEmitter bus, API gateway, Kafka broker, Kubernetes API server, WebRTC SFU.
- **Keep it thin:** the mediator should *route*, not compute. Delegate domain logic to services.
- **Don't use it** for 2-3 objects, or for pure broadcast with no coordination logic.
- **Smell that you need it:** you cannot add a new component without editing every existing one.
- **Smell that you overused it:** `notify()` is a 200-line if/else and every feature touches the mediator.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [47 — Chain of Responsibility Pattern](./47-pattern-chain-of-responsibility.md) — also decouples sender from receiver, but along a line instead of through a hub |
| **Next** | [49 — Memento Pattern](./49-pattern-memento.md) — capture and restore an object's state without breaking encapsulation |
| **Related** | [41 — Observer Pattern](./41-pattern-observer.md) — the pattern most often confused with Mediator; know the difference cold |
| **Related** | [36 — Facade Pattern](./36-pattern-facade.md) — the other "central object" pattern; one-way and behaviour-free, unlike Mediator |
