# 20 — UML Class Diagrams — How to Model Your Design
## Category: LLD Fundamentals

---

## What is this?

A UML class diagram is a **map of your code's structure** — it shows every class as a box, what data each class holds, what it can do, and which classes are connected to which.

Think of it like the **cast list and relationship chart at the front of a Russian novel**: "Anna is married to Karenin, Anna is the lover of Vronsky, Vronsky is a friend of Levin." You haven't read a single page of plot yet, but you already understand the shape of the story. A class diagram does exactly that for a codebase — before you read one line of code, you know who exists and who talks to whom.

UML stands for **Unified Modeling Language** — a standard set of shapes agreed on in the 1990s so engineers everywhere draw the same picture the same way.

---

## Why does it matter?

**If you can't draw a class diagram**, three things go wrong:

1. **In interviews, you go silent.** An LLD interview is 45 minutes of you at a whiteboard. The interviewer says "design a parking lot." If you jump straight into typing code, you'll design yourself into a corner by minute 15 and won't be able to back out. Drawing the classes first is *cheap* to change. Erasing a box takes two seconds; refactoring 200 lines takes twenty minutes.

2. **At work, nobody can review your design.** A design doc without a class diagram is a wall of prose that reviewers skim and approve without reading. One diagram gets you real feedback: "wait, why does `Order` know about `EmailSender`?"

3. **You can't spot bad design.** Bad designs *look* bad on a diagram. A box with 25 methods is screaming "I violate Single Responsibility." A tangle of arrows going both ways is screaming "circular dependency." You cannot see this in code; you can see it instantly in a picture.

**The interview angle:** Interviewers grade LLD on whether your *relationships* are right — did you use composition where composition belongs? Did you invert a dependency? The diagram is the artifact they grade.

**The real-work angle:** Every design doc at a serious company has a class diagram or an equivalent. It's the fastest way to transfer a design from one brain to another.

---

## The core idea — explained simply

### The Lego Instruction Booklet Analogy

Open a Lego box and you get two things: a bag of bricks and an instruction booklet.

The **bag of bricks** is your classes. Each brick has a shape (its attributes) and studs that determine what it can connect to (its methods and interfaces).

The **instruction booklet** is your class diagram. It shows you:
- Which bricks exist (the boxes)
- How they snap together (the arrows)
- How many of each you need (the multiplicity — "2x red 4-stud brick")

Now here's the important part. Lego has **different kinds of connection**:

| Lego connection | UML relationship | What it means |
|---|---|---|
| A brick that's a *special version* of a brick (a sloped 2x4 is still a 2x4) | **Inheritance** | "is-a" — `SavingsAccount` **is an** `Account` |
| A brick that only *claims* to have standard studs so anything can attach | **Implementation** | "promises to behave like" — `StripeGateway` **implements** `PaymentGateway` |
| Two bricks sitting next to each other in the model, aware of each other | **Association** | "knows about" — `Student` **knows** `Course` |
| A brick pressed into a baseplate — pull it off and the brick survives fine | **Aggregation** | "has-a, but can live without me" — `Team` **has** `Player` |
| A brick *glued* into a bigger piece — smash the piece, the brick dies with it | **Composition** | "has-a, and owns its life" — `House` **owns** `Room` |
| You needed a screwdriver to build it, but the screwdriver isn't part of the model | **Dependency** | "uses temporarily" — `OrderService` **uses** `Logger` |

That table is 80% of what class diagrams are. The rest is notation.

**Why the distinction matters in code:** the difference between aggregation and composition is literally the difference between these two constructors:

```javascript
// AGGREGATION — the Player is passed in. It existed before the Team,
// and it will still exist after the Team is disbanded.
class Team {
  constructor(name, players) {
    this.name = name;
    this.players = players; // shared reference — not owned
  }
}

// COMPOSITION — the House CREATES its Rooms. No House, no Rooms.
// If the House is garbage collected, the Rooms go with it.
class House {
  constructor(address, roomSpecs) {
    this.address = address;
    this.rooms = roomSpecs.map(spec => new Room(spec)); // created here — owned
  }
}
```

The diagram forces you to make that choice *consciously*. That's the whole value.

---

## Key concepts inside this topic

### 1. The three-box class notation

Every class is a rectangle split into **three compartments**:

```
┌──────────────────────────┐
│         Book             │  ← Compartment 1: the class NAME
├──────────────────────────┤
│ - isbn: string           │  ← Compartment 2: ATTRIBUTES (data)
│ - title: string          │
│ - price: number          │
│ - stock: number          │
├──────────────────────────┤
│ + getPrice(): number     │  ← Compartment 3: METHODS (behaviour)
│ + reduceStock(qty): void │
│ - validateIsbn(): boolean│
└──────────────────────────┘
```

Format rules:
- **Attribute:** `visibility name: type`
- **Method:** `visibility name(params): returnType`
- If a class has no attributes or no methods, you may collapse that compartment — but on a whiteboard, just leave it empty.

**Abstract classes and interfaces** get a stereotype label above the name:

```
┌──────────────────────────┐        ┌──────────────────────────┐
│    <<interface>>         │        │      <<abstract>>        │
│    PaymentGateway        │        │        Account           │
├──────────────────────────┤        ├──────────────────────────┤
│                          │        │ # balance: number        │
├──────────────────────────┤        ├──────────────────────────┤
│ + charge(amt): Receipt   │        │ + deposit(amt): void     │
│ + refund(txnId): boolean │        │ + calcInterest(): number │ ← abstract, italic in
└──────────────────────────┘        └──────────────────────────┘   real UML; on a
                                                                    whiteboard, mark {abstract}
```

JavaScript has no `interface` keyword, so you fake it with a base class that throws:

```javascript
// The <<interface>> box, in JS
class PaymentGateway {
  async charge(amount, token) { throw new Error('Not implemented'); }
  async refund(txnId)         { throw new Error('Not implemented'); }
}
```

### 2. Visibility markers

Four symbols, memorize them — they cost you nothing and they make you look fluent:

| Symbol | Means | JS equivalent |
|---|---|---|
| `+` | **public** — anyone can call it | a normal property/method |
| `-` | **private** — only this class | `#field` (real private) or `_field` (convention) |
| `#` | **protected** — this class and subclasses | no real support; `_field` by convention |
| `~` | **package** — same module (rare, skip it) | module-scoped, not exported |
| _underline_ | **static** — belongs to the class, not to instances | `static` |

```
┌──────────────────────────────┐
│         OrderId              │
├──────────────────────────────┤
│ - value: string              │   private instance field
│ ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾         │
│ + PREFIX: string             │   ← underlined = STATIC constant
├──────────────────────────────┤
│ ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾      │
│ + generate(): OrderId        │   ← underlined = STATIC factory method
│ + toString(): string         │
└──────────────────────────────┘
```

On a whiteboard you can't easily underline inside a box, so just write `<static>` after it. Nobody will mark you down.

```javascript
class OrderId {
  static PREFIX = 'ORD';           // + PREFIX (underlined)
  #value;                          // - value

  constructor(value) { this.#value = value; }

  static generate() {              // + generate() (underlined)
    return new OrderId(`${OrderId.PREFIX}-${Date.now()}`);
  }

  toString() { return this.#value; }  // + toString()
}
```

### 3. The six relationship arrows — and exactly how to draw each in ASCII

This is the part people get wrong. Learn the *shape of the arrowhead* and the *line style*, because those two things carry all the meaning.

**a) Inheritance ("is-a") — solid line, hollow triangle pointing at the PARENT**

```
        ┌──────────────┐
        │   Account     │          ← the PARENT (general)
        └───────△───────┘
                │                  ← hollow triangle sits ON the parent
        ┌───────┴───────┐
        │               │
┌───────┴──────┐ ┌──────┴───────┐
│SavingsAccount│ │CheckingAccount│  ← the CHILDREN (specific)
└──────────────┘ └──────────────┘
```
Read as: `SavingsAccount` **is an** `Account`. JS: `class SavingsAccount extends Account {}`

**b) Implementation ("promises to behave like") — DASHED line, hollow triangle**

```
        ┌──────────────────┐
        │  <<interface>>    │
        │  PaymentGateway   │
        └────────△──────────┘
                 ╎                 ← DASHED (use ╎ or : or ¦ on a whiteboard)
        ┌────────┴─────────┐
        │                  │
┌───────┴──────┐  ┌────────┴─────┐
│StripeGateway │  │ PaypalGateway│
└──────────────┘  └──────────────┘
```
Same triangle as inheritance, but the line is dashed. Read as: `StripeGateway` **implements** `PaymentGateway`.
Memory hook: **dashed = a promise, not a bloodline.**

**c) Association ("knows about") — plain solid line, optional open arrow for direction**

```
┌───────────┐  places   ┌───────────┐
│ Customer   │──────────▶│   Order    │
└───────────┘  1     0..*└───────────┘
```
A `Customer` holds a reference to `Order`s. No ownership claim, no lifecycle claim. Just: this object can reach that object. The verb on the line (`places`) and the numbers (multiplicity) are what make it readable.

**d) Aggregation ("has-a, shared") — HOLLOW diamond on the OWNER side**

```
┌───────────┐         ┌───────────┐
│ ShoppingCart│◇───────│   Book     │
└───────────┘ 1    0..*└───────────┘
```
The `Book` exists in the catalogue whether or not it's in your cart. Delete the cart → the Book survives. **Hollow diamond = weak ownership.**

**e) Composition ("has-a, owned") — FILLED diamond on the OWNER side**

```
┌───────────┐         ┌───────────┐
│   Order    │◆───────│ OrderLine  │
└───────────┘ 1    1..*└───────────┘
```
An `OrderLine` has no meaning outside its `Order`. Delete the Order → the OrderLines are meaningless and go away. **Filled diamond = strong ownership, cascading delete.**

> Whiteboard tip: draw the diamond as `<>` for hollow and `<#>` (scribbled in) for filled. Nobody expects perfect glyphs.

**f) Dependency ("uses temporarily") — DASHED line, open arrowhead (>, not a triangle)**

```
┌────────────────┐         ┌───────────┐
│  OrderService   │- - - - ->│  Logger    │
└────────────────┘  «uses»  └───────────┘
```
`OrderService` takes a `Logger` as a method parameter, or constructs one inside a method, but doesn't store it as a field. The weakest relationship. If you're not sure between association and dependency: **is it a field? → association. Is it just a local variable / parameter? → dependency.**

**The whole cheat-strip, one place:**

```
  ───────▷    inheritance      (solid line, hollow triangle)   "is-a"
  - - - -▷    implementation   (dashed line, hollow triangle)  "implements"
  ────────    association      (solid line)                    "knows-a"
  ◇───────    aggregation      (hollow diamond at owner)       "has-a (shared)"
  ◆───────    composition      (filled diamond at owner)       "owns-a (exclusive)"
  - - - ->    dependency       (dashed line, open arrow)        "uses"
```

### 4. Multiplicity — the little numbers on the line

Multiplicity says **how many** objects sit at each end. Write it near the box it describes.

| Notation | Read as | Example |
|---|---|---|
| `1` | exactly one | An `Order` has exactly **1** `Customer` |
| `0..1` | zero or one (optional) | An `Order` has **0..1** `Coupon` |
| `1..*` | one or more (at least one) | An `Order` has **1..*** `OrderLine` |
| `*` (or `0..*`) | any number, including zero | A `Customer` has ***** `Order`s |
| `2..5` | a specific range | A `Match` has **2..5** `Player`s |

Read a line from **left to right using the multiplicity at the FAR end**:

```
┌───────────┐  1        0..*  ┌───────────┐
│ Customer   │─────────────────│   Order    │
└───────────┘                 └───────────┘
```
- Customer → Order: read the number next to Order = `0..*` → "one Customer has zero-or-more Orders"
- Order → Customer: read the number next to Customer = `1` → "one Order has exactly one Customer"

That's a **one-to-many**. Two `*`s would be a **many-to-many** (and in a real DB that means a join table — a great thing to say out loud in an interview).

### 5. Worked example — a small online bookstore

Requirements, in one breath: *customers browse books, add them to a cart, place an order, and pay by card or PayPal. Orders can be physical or digital books. Every order emits a confirmation.*

Pull out the nouns → those are your classes. Pull out the verbs → those are your methods. (You'll do this formally in [22 — UML Use Case Diagrams](./22-uml-use-case-diagrams.md).)

**The class diagram:**

```
                        ┌─────────────────────────────┐
                        │      <<interface>>           │
                        │      PaymentGateway          │
                        ├─────────────────────────────┤
                        │ + charge(amt, token): Receipt│
                        └──────────────△──────────────┘
                                       ╎  implements (dashed + triangle)
                        ┌──────────────┴──────────────┐
                        ╎                             ╎
              ┌─────────┴────────┐          ┌─────────┴────────┐
              │  StripeGateway    │          │  PaypalGateway    │
              ├──────────────────┤          ├──────────────────┤
              │ - apiKey: string  │          │ - clientId: string│
              ├──────────────────┤          ├──────────────────┤
              │ + charge(): Receipt│         │ + charge(): Receipt│
              └──────────────────┘          └──────────────────┘
                        △
                        │  «uses» (dashed, open arrow)
                        ╎
                        ╎
  ┌──────────────┐      ╎         ┌──────────────────────────────┐
  │   Customer    │      ╎         │        OrderService           │
  ├──────────────┤      ╎         ├──────────────────────────────┤
  │ - id: string  │      └ ─ ─ ─ ─│ - gateway: PaymentGateway     │
  │ - email:string│                │ - bus: EventBus              │
  ├──────────────┤                ├──────────────────────────────┤
  │ + getId()     │                │ + placeOrder(cart): Order     │
  └──────┬───────┘                └───────────┬──────────────────┘
         │ 1                                   │ «uses»
         │            places                   ▼
         │ 0..*                     ┌──────────────────┐
  ┌──────▼────────────────┐         │    EventBus       │
  │        Order           │         ├──────────────────┤
  ├───────────────────────┤         │ + emit(evt, data) │
  │ - id: string           │         │ + on(evt, fn)     │
  │ - status: OrderStatus  │         └──────────────────┘
  │ - placedAt: Date       │
  ├───────────────────────┤
  │ + total(): number      │
  │ + markPaid(): void     │
  └──────◆────────────────┘
         │ 1        composition (filled diamond — no Order, no lines)
         │
         │ 1..*
  ┌──────▼────────────────┐
  │      OrderLine         │
  ├───────────────────────┤
  │ - qty: number          │
  │ - unitPrice: number    │
  ├───────────────────────┤
  │ + subtotal(): number   │
  └──────◇────────────────┘
         │ 1        aggregation (hollow diamond — Book outlives the line)
         │
         │ 1
  ┌──────▼────────────────┐
  │     <<abstract>>       │
  │        Book            │
  ├───────────────────────┤
  │ # isbn: string         │
  │ # title: string        │
  │ # price: number        │
  ├───────────────────────┤
  │ + getPrice(): number   │
  │ + deliver(): string    │  {abstract}
  └───────────△───────────┘
              │  inheritance (solid + hollow triangle)
      ┌───────┴────────┐
      │                │
┌─────┴──────┐  ┌──────┴───────┐
│PhysicalBook│  │ DigitalBook   │
├────────────┤  ├──────────────┤
│- weightKg  │  │- fileSizeMb   │
├────────────┤  ├──────────────┤
│+ deliver() │  │+ deliver()    │
└────────────┘  └──────────────┘
```

**Read it out loud, and you have narrated your entire design:**
- A `Customer` places `0..*` `Order`s (association, one-to-many).
- An `Order` **owns** `1..*` `OrderLine`s (composition — kill the order, the lines are meaningless).
- An `OrderLine` **references** exactly `1` `Book` (aggregation — the Book lives in the catalogue independently).
- `Book` is abstract; `PhysicalBook` and `DigitalBook` **are** Books (inheritance) and each implements `deliver()` differently.
- `OrderService` **uses** a `PaymentGateway` and an `EventBus` (dependency).
- `StripeGateway` and `PaypalGateway` **implement** `PaymentGateway` — so `OrderService` never knows which one it got. (That's the Dependency Inversion Principle from [18 — DIP](./18-solid-dependency-inversion.md), drawn as a picture.)

### 6. The matching JavaScript — line for line

```javascript
// ── Enum (frozen object — JS has no enum keyword) ────────────────────
const OrderStatus = Object.freeze({
  PENDING: 'PENDING',
  PAID:    'PAID',
  SHIPPED: 'SHIPPED',
});

// ── <<abstract>> Book  +  its two subclasses (inheritance) ───────────
class Book {
  constructor(isbn, title, price) {
    if (new.target === Book) throw new Error('Book is abstract');
    this._isbn  = isbn;   // # protected (convention)
    this._title = title;
    this._price = price;
  }
  getPrice() { return this._price; }              // + getPrice(): number
  deliver()  { throw new Error('Not implemented'); } // abstract
}

class PhysicalBook extends Book {                 // PhysicalBook ──▷ Book
  constructor(isbn, title, price, weightKg) {
    super(isbn, title, price);
    this._weightKg = weightKg;
  }
  deliver() { return `Shipping ${this._title} (${this._weightKg}kg) by courier`; }
}

class DigitalBook extends Book {                  // DigitalBook ──▷ Book
  constructor(isbn, title, price, fileSizeMb) {
    super(isbn, title, price);
    this._fileSizeMb = fileSizeMb;
  }
  deliver() { return `Emailing download link for ${this._title}`; }
}

// ── OrderLine ◇── Book  (AGGREGATION: the Book is PASSED IN, not created) ──
class OrderLine {
  constructor(book, qty) {
    this.book = book;                 // shared reference — the catalogue owns it
    this.qty  = qty;
    this.unitPrice = book.getPrice(); // snapshot the price at order time
  }
  subtotal() { return this.unitPrice * this.qty; }
}

// ── Order ◆── OrderLine  (COMPOSITION: the Order CREATES its lines) ────────
class Order {
  #lines = [];
  constructor(customerId) {
    this.id         = `ORD-${Date.now()}`;
    this.customerId = customerId;      // association back to Customer
    this.status     = OrderStatus.PENDING;
    this.placedAt   = new Date();
  }
  // The Order constructs its own OrderLines → it owns their lifecycle.
  addLine(book, qty) { this.#lines.push(new OrderLine(book, qty)); }
  get lines()        { return [...this.#lines]; }   // defensive copy
  total()            { return this.#lines.reduce((s, l) => s + l.subtotal(), 0); }
  markPaid()         { this.status = OrderStatus.PAID; }
}

// ── <<interface>> PaymentGateway + implementations ───────────────────
class PaymentGateway {
  async charge(amount, token) { throw new Error('Not implemented'); }
}
class StripeGateway extends PaymentGateway {
  constructor(apiKey) { super(); this._apiKey = apiKey; }
  async charge(amount, token) { return { txnId: `stripe_${Date.now()}`, amount }; }
}
class PaypalGateway extends PaymentGateway {
  constructor(clientId) { super(); this._clientId = clientId; }
  async charge(amount, token) { return { txnId: `pp_${Date.now()}`, amount }; }
}

// ── OrderService - - -> PaymentGateway, EventBus  (DEPENDENCY) ───────
class OrderService {
  // Injected, not constructed → OrderService depends on the ABSTRACTION.
  constructor(gateway, eventBus) {
    this.gateway  = gateway;
    this.eventBus = eventBus;
  }
  async placeOrder(customer, cartItems, paymentToken) {
    const order = new Order(customer.id);
    for (const { book, qty } of cartItems) order.addLine(book, qty);

    const receipt = await this.gateway.charge(order.total(), paymentToken);
    order.markPaid();
    this.eventBus.emit('order:placed', { orderId: order.id, receipt });
    return order;
  }
}
```

Notice: **you can reconstruct the diagram from the code, and the code from the diagram.** That round-trip is the test of a good diagram.

### 7. How to draw this fast on a whiteboard

You have ~4 minutes to get boxes on the board before you start losing the room. Do this:

1. **Boxes first, empty.** Write only class names. Spread them out — leave room for arrows. Put abstractions at the TOP, concrete things at the BOTTOM. Arrows will then mostly point upward, which is what a healthy design looks like.
2. **Arrows second.** Say the relationship out loud as you draw it: *"An Order owns its lines — filled diamond."* This is free points; you're demonstrating vocabulary.
3. **Multiplicity third.** Just the numbers. Ten seconds.
4. **Fill in fields and methods LAST**, and only for the 2–3 classes the interviewer is actually interested in. Do not lovingly detail all twelve boxes. Nobody cares that `Customer` has an `email`.
5. **Leave whitespace.** You will be asked to add a feature ("now support gift cards"). If your board is full, you're stuck.

**The single biggest mistake:** drawing every arrow as a plain line. If every relationship looks the same, you've communicated nothing. The arrowheads *are* the design.

---

## Visual / Diagram description

### Diagram 1: The relationship strength spectrum

The six relationships are not six unrelated things — they're a **spectrum of coupling**, from "barely knows it exists" to "cannot exist without it."

```
 WEAKEST                                                          STRONGEST
 coupling                                                          coupling
    │                                                                  │
    ▼                                                                  ▼
┌─────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐
│Dependency│  │Association │  │Aggregation │  │Composition │  │ Inheritance  │
│  «uses»  │  │  «knows»   │  │  «has-a»   │  │  «owns-a»  │  │    «is-a»    │
├─────────┤  ├────────────┤  ├────────────┤  ├────────────┤  ├──────────────┤
│ a local  │  │  a field   │  │  a field,  │  │  a field,  │  │  the SAME    │
│ variable │  │            │  │  passed in │  │  created   │  │  object,     │
│ or param │  │            │  │            │  │  inside    │  │  specialized │
├─────────┤  ├────────────┤  ├────────────┤  ├────────────┤  ├──────────────┤
│- - - - ->│  │──────────  │  │◇───────    │  │◆───────    │  │───────▷      │
└─────────┘  └────────────┘  └────────────┘  └────────────┘  └──────────────┘
    │              │               │               │                 │
 Logger in     Customer →      Cart ◇ Book     Order ◆ Line     Savings ▷ Account
 a method       Order          (Book lives     (Line dies       (Savings IS
                               without cart)   with Order)       an Account)
```

**What the diagram shows:** as you move right, you give up flexibility for expressiveness. Inheritance is the strongest and therefore the most dangerous — a change to the parent breaks every child. This is exactly why the Gang of Four said *"favor composition over inheritance"* — see [26 — Composition Over Inheritance](./26-composition-over-inheritance.md). When in doubt, move **left** on this spectrum.

### Diagram 2: The "smell test" — what a bad diagram looks like

```
   BAD: the God Object                     GOOD: split by responsibility
┌──────────────────────────┐        ┌──────────┐  ┌──────────┐  ┌────────────┐
│      OrderManager         │        │  Order    │  │ Pricing   │  │  Shipping   │
├──────────────────────────┤        ├──────────┤  ├──────────┤  ├────────────┤
│ + createOrder()          │        │+ total() │  │+ apply   │  │+ quote()    │
│ + calculateTax()         │        │+ addLine │  │  Discount│  │+ dispatch() │
│ + applyDiscount()        │        └────┬─────┘  └────△─────┘  └─────△──────┘
│ + chargeCard()           │             │ «uses»      ╎ «uses»       ╎
│ + printInvoice()         │             └─────────────┴──────────────┘
│ + sendEmail()            │                            │
│ + updateInventory()      │                   ┌────────┴─────────┐
│ + scheduleDelivery()     │                   │   OrderService    │
│ + refund()               │                   └──────────────────┘
│ + generateReport()       │
└──────────────────────────┘
   ↑ 9 methods, 5 reasons             ↑ each box has ONE reason to change
     to change. Nothing else in
     the diagram. This is a smell.
```

**What the diagram shows:** a single box with a long method list and no relationships is *always* a design smell. Real designs are many small boxes with clear arrows. If your whiteboard has one giant box, the interviewer already knows your grade.

---

## Real world examples

### 1. Java's Collections Framework — inheritance and implementation, drawn

The `java.util` hierarchy is the most-taught class diagram in software, and it is genuinely well designed: `Collection` is an interface; `List`, `Set`, `Queue` are interfaces that **extend** it; `ArrayList`, `LinkedList` are classes that **implement** `List`. Every arrow in that diagram is a hollow triangle. The reason `List<String> x = new ArrayList<>()` is idiomatic Java is precisely the shape of that diagram — you code against the interface box, not the implementation box. Node's ecosystem has the same idea informally: any object with a `.pipe()` method and the right events is treated as a `Stream`.

### 2. Stripe's SDKs — the interface box in the wild

Stripe's official libraries expose one gateway-shaped surface (`charges`, `refunds`, `customers`) and every language binding implements it identically. Applications that wrap Stripe behind their own `PaymentGateway` interface — exactly the diagram we drew above — can add a second processor (Adyen, Braintree) by adding **one new box with one dashed triangle**, and zero changes to `OrderService`. Companies that call `stripe.charges.create()` directly from their business logic have that call site scattered across 200 files, and migrating processors becomes a multi-quarter project. Same code, different diagram.

### 3. React's component model — composition, not inheritance

React's docs explicitly recommend composition over inheritance for reusing code between components. In class-diagram terms: React deliberately gives you no inheritance arrows between components. A `<Dialog>` doesn't `extend` a `<Panel>`; it **contains** one (`children`). That is a filled/hollow diamond, not a triangle. It's a real, load-bearing architecture decision, and it's the reason React component trees stay refactorable at scale.

---

## Trade-offs

| Decision | Benefit | Cost |
|---|---|---|
| **Draw a full, detailed class diagram** | Everyone understands the design; bad structure is visible immediately | Goes stale the moment code changes; can take longer than writing the code |
| **Draw a sketch-level diagram (boxes + arrows only)** | Fast, disposable, perfect for whiteboards and design reviews | Ambiguous — reviewers may fill the gaps with different assumptions |
| **Draw no diagram, just code** | Nothing to keep in sync | Nobody can review the design; you discover structural mistakes only after writing 300 lines |

| Relationship choice | Choose it when | What you give up |
|---|---|---|
| **Inheritance** (`is-a`) | The subtype is *genuinely substitutable* for the parent (see [16 — LSP](./16-solid-liskov-substitution.md)) | Flexibility. Deep hierarchies are rigid, and a parent change breaks all children |
| **Composition** (`owns-a`) | The part has no meaning outside the whole | Slightly more code (delegation methods) |
| **Aggregation** (`has-a`) | The part is shared and outlives the whole | You must handle the case where the shared object mutates under you |
| **Dependency injection** (`uses` an interface) | You want swappable implementations and testability | An extra abstraction layer that may never be swapped |

**The sweet spot:** in an interview, draw a **sketch-level diagram** with correct arrowheads and multiplicity, and detail only the 2–3 classes at the heart of the problem. At work, keep a diagram at the *module* level (10–15 boxes max) in the design doc, and let the code be the source of truth below that. A diagram that tries to document every class is a diagram nobody will ever update.

---

## Common interview questions on this topic

### Q1: "What's the difference between aggregation and composition? Show me in code."
**Hint:** Lifecycle ownership. Composition = the whole **creates** the part and the part **dies** with it (`this.rooms = specs.map(s => new Room(s))` — filled diamond). Aggregation = the part is **passed in**, pre-existing, and survives the whole (`constructor(players) { this.players = players }` — hollow diamond). The one-line test: *"if I delete the container, should the contained object be deleted too?"* Yes → composition. No → aggregation.

### Q2: "When would you use an interface instead of an abstract class?"
**Hint:** Abstract class = shared *implementation* plus a contract ("all Books have a price and know how to compute it, but each delivers differently"). Interface = pure contract, no state ("anything that can charge a card"). Rule: if you'd have to duplicate real code across subclasses, use an abstract class. If you only need a promise about method names — and especially if a class needs to satisfy *several* contracts — use interfaces. In JS both are base classes, so say the intent out loud: *"I'd mark this one as a pure interface — no state, all methods throw."*

### Q3: "Your diagram has a bidirectional association. Is that a problem?"
**Hint:** Usually yes. `Order → Customer` **and** `Customer → Order` means each must keep the other in sync, and you now have a circular dependency that's painful to serialize, test, and garbage-collect. Prefer one direction and let the other side query a repository (`orderRepo.findByCustomer(id)`). Only keep it bidirectional if navigation in both directions is genuinely hot-path.

### Q4: "How do you decide what goes in the diagram vs what you leave out?"
**Hint:** Include anything the *interviewer will ask a follow-up about*: the abstractions, the polymorphic hierarchies, the relationships with interesting lifecycles. Leave out DTOs, config objects, and getters/setters. A good rule: if a box has no arrows, it probably doesn't belong on the board.

### Q5: "Walk me through your diagram."
**Hint:** This is the real question, and it's asked every time. Narrate it as **sentences**, not as a list of boxes. *"A Customer places zero-or-more Orders. Each Order owns one-or-more OrderLines — composition, because a line means nothing without its order. Each line references exactly one Book from the catalogue — aggregation, because the Book outlives the order. OrderService depends on the PaymentGateway *interface*, so I can swap Stripe for PayPal without touching order logic."* Four sentences, and you've demonstrated relationships, multiplicity, lifecycle reasoning, and DIP.

---

## Practice exercise

### The Bookstore, Extended

Take the bookstore diagram above and **extend it on paper (or in a text file) before writing any code.**

The product manager adds three requirements:
1. **Reviews:** a Customer can leave at most one Review per Book. A Review has a rating (1–5) and text.
2. **Discounts:** an Order can have zero or one Coupon applied. There are two kinds of coupon: `PercentOffCoupon` and `FlatAmountCoupon`, and more will be added later.
3. **Wishlist:** a Customer has exactly one Wishlist, which contains any number of Books. Deleting the Customer deletes the Wishlist.

**What to produce:**

**Part A (15 min) — the diagram.** Redraw the class diagram in ASCII with the three new features. For every new line you draw, you must pick the correct arrowhead and write the multiplicity at both ends. Specifically decide, and be ready to defend:
- Is `Customer → Wishlist` aggregation or composition?
- Is `Wishlist → Book` aggregation or composition?
- Is `Coupon` an abstract class or an interface? Why?
- What relationship connects `Order` and `Coupon`, and what's its multiplicity?

**Part B (20 min) — the code.** Write the JavaScript for the three new features so that it *exactly matches* your diagram: composition means the constructor calls `new`; aggregation means the object is passed in; implementation means a base class whose methods throw.

**Part C (5 min) — the check.** Hand your code to a friend (or reread it tomorrow) and try to redraw the diagram *from the code alone*. If the two diagrams don't match, your code and your design have already diverged — find out which one lied.

---

## Quick reference cheat sheet

- **Three compartments:** class name / attributes (`- name: type`) / methods (`+ name(args): returnType`).
- **Visibility:** `+` public, `-` private, `#` protected, `~` package, **underline = static**.
- **Stereotypes:** `<<interface>>` and `<<abstract>>` go above the class name; abstract methods are italic (or marked `{abstract}`).
- **Inheritance** = solid line + **hollow triangle** → "is-a" → `extends`.
- **Implementation** = **dashed** line + hollow triangle → "implements" → base class whose methods throw.
- **Association** = plain solid line → "has a reference to" → it's a field.
- **Aggregation** = **hollow diamond** at the owner → part is passed in and outlives the whole.
- **Composition** = **filled diamond** at the owner → part is `new`'d inside and dies with the whole.
- **Dependency** = dashed line + **open arrow** → used as a parameter or local variable, not stored.
- **Multiplicity:** `1`, `0..1`, `1..*`, `*`, `2..5`. Read the number at the FAR end of the line.
- **Relationship strength, weakest → strongest:** dependency < association < aggregation < composition < inheritance. When unsure, pick the weaker one.
- **Whiteboard order:** empty boxes → arrows (say them out loud) → multiplicity → details for 2–3 key classes only.
- **Smell:** one giant box with 10 methods and no arrows = God Object. Many small boxes with clear arrows = a real design.
- **The round-trip test:** a good diagram can be turned into code, and that code redrawn as the same diagram.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [19 — DRY, KISS, YAGNI](./19-dry-kiss-yagni.md) — the principles that keep the boxes on your diagram few and focused |
| **Next** | [21 — UML Sequence Diagrams](./21-uml-sequence-diagrams.md) — the class diagram shows structure; the sequence diagram shows the flow through it |
| **Related** | [23 — Class Relationships](./23-class-relationships.md) — a full deep-dive on association vs aggregation vs composition vs inheritance |
| **Related** | [02 — HLD vs LLD](./02-hld-vs-lld.md) — class diagrams are the primary artifact of the LLD zoom level |
