# 23 — Class Relationships — Association, Aggregation, Composition, Inheritance
## Category: LLD Fundamentals

---

## What is this?

Classes almost never live alone. They point at each other, contain each other, and inherit from each other. **Class relationships** are the four standard ways two classes can be connected — and naming the right one is what turns a pile of classes into a *design*.

Think of a company org chart. Some people **are a** kind of employee (a Manager *is an* Employee). Some people **use** each other (a Recruiter *uses* an Interviewer's time). Some groups **have** members who existed before the group and will outlive it (a Project Team *has* Engineers — kill the project, the engineers keep their jobs). And some things **own** their parts absolutely (the company *owns* its Slack workspace — dissolve the company, the workspace is deleted).

Those four sentences are exactly Inheritance, Association, Aggregation, and Composition.

---

## Why does it matter?

Pick the wrong relationship and you get bugs that are expensive to unwind:

- **Wrong inheritance** → you end up with `class Square extends Rectangle` and a broken `setWidth`. You'll re-derive this pain in [16 — Liskov Substitution](./16-solid-liskov-substitution.md).
- **Composition where you meant aggregation** → deleting a Team deletes its Players from the database. Someone gets paged at 2am.
- **Aggregation where you meant composition** → you delete an Order but its OrderLine rows stay behind forever. Now your `order_lines` table has 40 million orphans and your finance reports are wrong.
- **Association everywhere** → every class holds a reference to every other class, nothing can be tested in isolation, and one change ripples through 30 files. That's the coupling problem in [24 — Coupling and Cohesion](./24-coupling-and-cohesion.md).

**In interviews:** "What's the difference between aggregation and composition?" is one of the most-asked LLD questions on the planet. Most candidates say "aggregation is weak, composition is strong" and stop. That answer is worth zero. What earns points is showing **lifecycle**: who calls `new`, who holds the only reference, and what happens on delete.

**At work:** every ORM decision (`onDelete: CASCADE` vs `SET NULL`), every DDD "aggregate root", every "should this be a foreign key or an embedded document" question is this topic wearing a different hat.

---

## The core idea — explained simply

### The Restaurant Analogy

Walk into a restaurant and look around. All four relationships are sitting right there.

**Inheritance — "is-a":** A `HeadChef` **is a** `Chef`. Anywhere the restaurant needs a Chef, a HeadChef will do. The HeadChef inherits everything a Chef can do (chop, cook, plate) and adds more (manage the roster). You can hand a HeadChef to any function expecting a Chef and nothing breaks. That's the test.

**Association — "uses-a":** A `Waiter` **uses** a `Chef`. The waiter walks up, hands over an order, walks away. The waiter didn't create the chef. The waiter doesn't own the chef. The chef doesn't belong to the waiter. They just talk. If the waiter quits, the chef keeps cooking. If the chef quits, the waiter is still employed. It's a *relationship of collaboration*, nothing more.

**Aggregation — "has-a" (weak ownership):** The `Restaurant` **has** `Chefs` on staff. The restaurant didn't create the chefs — they existed as people before they were hired, they have lives and other jobs. If the restaurant shuts down tomorrow, the chefs walk out the door and get hired somewhere else. **The parts survive the whole.**

**Composition — "owns-a" (strong ownership):** The `Restaurant` **owns** its `Kitchen`. The kitchen was built when the restaurant was built. It exists nowhere else. It cannot be "reassigned" to another restaurant. If the restaurant is demolished, the kitchen is demolished with it. **The parts die with the whole.**

### Mapping the analogy back

| Analogy | Relationship | Verb | Who calls `new`? | Can the part exist alone? | On delete of the whole |
|---|---|---|---|---|---|
| HeadChef → Chef | Inheritance | **is-a** | n/a (it's a type, not a field) | n/a | n/a |
| Waiter → Chef | Association | **uses-a** | Someone else | Yes | Part is untouched |
| Restaurant → Chefs | Aggregation | **has-a** | Someone else, passed in | Yes — chefs get new jobs | Part survives |
| Restaurant → Kitchen | Composition | **owns-a** | The whole, in its constructor | No | Part is destroyed |

The single question that separates aggregation from composition:

> **"If I delete the whole, does it make sense for the part to still exist?"**
>
> Yes → aggregation. No → composition.

---

## Key concepts inside this topic

### 1. Inheritance — "is-a"

`B extends A` means *every B is a valid A*. Not "B is similar to A". Not "B reuses A's code". **Every B is an A**, and every place in your codebase that accepts an A must work correctly when handed a B.

```javascript
class PaymentMethod {
  constructor(ownerId) {
    this.ownerId = ownerId;
  }
  // Subclasses must define how they actually move money.
  async charge(amountCents) {
    throw new Error('charge() must be implemented by a subclass');
  }
  describe() {
    return `${this.constructor.name} for owner ${this.ownerId}`;
  }
}

class CreditCard extends PaymentMethod {
  constructor(ownerId, last4) {
    super(ownerId);           // an "is-a" relationship always calls super()
    this.last4 = last4;
  }
  async charge(amountCents) {
    return { ok: true, via: `card ****${this.last4}`, amountCents };
  }
}

class UpiWallet extends PaymentMethod {
  constructor(ownerId, vpa) {
    super(ownerId);
    this.vpa = vpa;
  }
  async charge(amountCents) {
    return { ok: true, via: `upi ${this.vpa}`, amountCents };
  }
}

// The payoff: this function has NEVER heard of CreditCard or UpiWallet.
async function checkout(paymentMethod, amountCents) {
  return paymentMethod.charge(amountCents);   // works for ANY PaymentMethod
}
```

**The decision rule for inheritance:**
> Use it only when the subclass can be substituted for the parent **everywhere**, with no caller needing to know which it got. If you find yourself writing `if (x instanceof CreditCard)` in the caller, your inheritance is wrong.

**The trap.** People reach for inheritance to *reuse code*. That's the wrong reason. Inheritance is for *substitutability*. If you just want to reuse a method, use composition — that's the whole argument in [26 — Composition Over Inheritance](./26-composition-over-inheritance.md).

Bad, and very common:

```javascript
// BAD: Stack "is-a" Array? No. A Stack is not substitutable for an Array —
// an Array lets you splice() into the middle, which breaks stack semantics.
class Stack extends Array {
  push(x) { super.push(x); }
  pop() { return super.pop(); }
}
const s = new Stack();
s.push(1); s.push(2);
s.splice(0, 1);   // Inherited. Completely destroys the LIFO contract.
```

```javascript
// GOOD: Stack "has-a" array (composition). It exposes only stack operations.
class Stack {
  #items = [];                          // private field: nobody can splice it
  push(x) { this.#items.push(x); return this; }
  pop()   { return this.#items.pop(); }
  peek()  { return this.#items.at(-1); }
  get size() { return this.#items.length; }
}
```

### 2. Association — "uses-a"

Two classes know about each other and collaborate, but neither owns the other's lifecycle. This is the *loosest* relationship — and therefore the one you should reach for by default.

The classic textbook example: a **Doctor treats a Patient.**

```javascript
class Patient {
  constructor(id, name) {
    this.id = id;
    this.name = name;
    this.records = [];
  }
}

class Doctor {
  constructor(id, name) {
    this.id = id;
    this.name = name;
  }

  // The Doctor USES a Patient. It receives one, does something, and lets go.
  // Note: it does NOT store the patient as a field. That's a pure association.
  treat(patient, diagnosis) {
    patient.records.push({ by: this.name, diagnosis, at: new Date() });
    return `Dr. ${this.name} treated ${patient.name}`;
  }
}

// Lifecycle: nobody creates anybody. Both are created independently, outside.
const alice  = new Patient('p1', 'Alice');
const drBose = new Doctor('d1', 'Bose');

drBose.treat(alice, 'flu');

// Delete the doctor — the patient is completely unaffected.
// Delete the patient — the doctor is completely unaffected.
```

**Association can also be a stored reference**, and it still isn't ownership:

```javascript
class Order {
  constructor(id, customer) {
    this.id = id;
    this.customer = customer;   // a reference to someone else's object
  }
}
// Deleting an Order must NEVER delete the Customer. The Customer is associated,
// not owned. In SQL this is: customer_id REFERENCES customers(id) ON DELETE RESTRICT
```

**The decision rule for association:**
> If class A merely needs to *call methods on* B or *point at* B, and A neither created B nor controls when B dies, it's an association. This is your default. Prefer it.

### 3. Aggregation — "has-a", weak ownership, parts survive the whole

The whole holds a collection of parts, but the parts are created **outside** and are **passed in**. The parts have an independent life. They can be moved to a different whole. They can belong to several wholes at once.

**The canonical example: a Team has Players.**

```javascript
class Player {
  constructor(id, name) {
    this.id = id;
    this.name = name;
  }
}

class Team {
  #players = [];

  constructor(name) {
    this.name = name;
    // NOTE: the Team never calls `new Player(...)`. That is the tell.
  }

  // Players are handed to the team from the outside.
  addPlayer(player) {
    this.#players.push(player);
    return this;
  }

  removePlayer(playerId) {
    this.#players = this.#players.filter(p => p.id !== playerId);
    // The Player object itself is NOT destroyed. We just stopped pointing at it.
  }

  get roster() { return [...this.#players]; }

  disband() {
    this.#players = [];   // The team is gone. The players are fine.
  }
}
```

**The lifecycle demonstration — this is what interviewers want to see:**

```javascript
// 1. The PARTS are created first, by the OUTSIDE world.
const virat  = new Player('p1', 'Virat');
const rohit  = new Player('p2', 'Rohit');

// 2. The WHOLE is created, and the parts are INJECTED.
const rcb = new Team('RCB');
rcb.addPlayer(virat).addPlayer(rohit);

// 3. A part can belong to a SECOND whole at the same time. Perfectly legal.
const indiaSquad = new Team('India');
indiaSquad.addPlayer(virat);        // same Player object, two Teams

// 4. Destroy the whole.
rcb.disband();

// 5. The part is ALIVE AND WELL. It has an owner elsewhere and a life of its own.
console.log(virat.name);                  // 'Virat'  — still here
console.log(indiaSquad.roster.length);    // 1        — still on India's team
```

**The decision rule for aggregation:**
> The part is created outside and passed in (`addX(x)` or a constructor argument). Deleting the whole must NOT delete the part. In SQL: `ON DELETE SET NULL` or a join table.

### 4. Composition — "owns-a", strong ownership, parts die with the whole

The whole **creates** its parts, usually in its own constructor. Nobody outside ever holds a reference to a part. The part is meaningless on its own. When the whole is destroyed, the parts are destroyed.

**The canonical example: a House has Rooms.**

```javascript
class Room {
  constructor(name, areaSqm) {
    this.name = name;
    this.areaSqm = areaSqm;
  }
  describe() { return `${this.name} (${this.areaSqm} m²)`; }
}

class House {
  #rooms;   // private — the outside world can never grab a reference to a Room

  constructor(address, layout) {
    this.address = address;

    // THE TELL: the whole calls `new` on its own parts, inside itself.
    // No Room existed before this House. No Room was passed in.
    this.#rooms = layout.map(({ name, areaSqm }) => new Room(name, areaSqm));
  }

  // We expose DATA about the rooms, never the Room objects themselves.
  // Handing out the object would leak ownership and break composition.
  floorPlan() {
    return this.#rooms.map(r => r.describe());
  }

  get totalAreaSqm() {
    return this.#rooms.reduce((sum, r) => sum + r.areaSqm, 0);
  }

  demolish() {
    this.#rooms = [];   // The rooms cease to exist. Nothing else referenced them.
  }
}
```

**The lifecycle demonstration:**

```javascript
// 1. The WHOLE is created. It builds its OWN parts. Notice: no `new Room(...)`
//    anywhere in this calling code. You cannot create a Room for a House.
const house = new House('12 Baker St', [
  { name: 'Kitchen',  areaSqm: 14 },
  { name: 'Bedroom',  areaSqm: 20 },
  { name: 'Bathroom', areaSqm: 6 },
]);

console.log(house.totalAreaSqm);   // 40
console.log(house.floorPlan());    // ['Kitchen (14 m²)', ...]

// 2. Can you take a Room out and put it in another House? No. There is no API
//    for it, and `#rooms` is private. That impossibility IS the design.

// 3. Destroy the whole.
house.demolish();

// 4. The Rooms are unreachable — no other reference exists anywhere in the
//    program. The garbage collector reclaims them. They are gone.
console.log(house.totalAreaSqm);   // 0
```

**The decision rule for composition:**
> The whole calls `new` on the part inside its own constructor, and holds the *only* reference (private field). Deleting the whole must delete the part. In SQL: `ON DELETE CASCADE`.

Real ones you meet every day: `Order` **owns** its `OrderLine`s (an order line for a deleted order is nonsense). `Invoice` **owns** its `LineItem`s. `HttpRequest` **owns** its `Headers`. `Document` **owns** its `Paragraph`s.

### 5. The one-question decision procedure

Do this in order. Stop at the first `yes`.

1. **Is every A genuinely a kind of B, substitutable everywhere B is used?** → **Inheritance**.
2. **Does A create the part in its own constructor and hold the only reference — and is the part nonsense without A?** → **Composition**.
3. **Does A hold parts that were created outside, passed in, and that survive A's deletion?** → **Aggregation**.
4. **Does A just call methods on B / point at B, with no ownership at all?** → **Association**.

When you're torn between 2 and 3, ask the delete question out loud: *"I just dropped the Team row. Should the Player rows go too?"* If the answer makes a product manager wince, it's aggregation.

### 6. How it shows up in your database and your ORM

This is not academic. Every relationship has an exact schema consequence.

| Relationship | Typical schema | Delete behaviour |
|---|---|---|
| Inheritance | Single-table with a `type` column, or one table per subclass | n/a |
| Association | FK, or no FK at all (just a method call) | `ON DELETE RESTRICT` |
| Aggregation | FK on the part, nullable; or a join table for many-to-many | `ON DELETE SET NULL` |
| Composition | FK on the part, NOT NULL; or an embedded document in Mongo | `ON DELETE CASCADE` |

```javascript
// Composition in a document DB is literally embedding:
{ _id: 'order_1', total: 4200, lines: [ { sku: 'A', qty: 2 }, ... ] }
//                                      ^^^^^ lines cannot exist without the order

// Aggregation is referencing:
{ _id: 'team_1', name: 'RCB', playerIds: ['p1', 'p2'] }
// players live in their own collection and outlive the team
```

---

## Visual / Diagram description

### Diagram 1: The four relationships, UML-style

UML notation (which you should use on the whiteboard):
- **Inheritance:** solid line, **hollow triangle** pointing at the parent
- **Association:** plain solid line (arrow if one-directional)
- **Aggregation:** solid line with a **hollow (empty) diamond** at the *whole*
- **Composition:** solid line with a **filled (solid) diamond** at the *whole*

```
   INHERITANCE ("is-a")                 ASSOCIATION ("uses-a")
   ┌───────────────┐                    ┌──────────┐          ┌──────────┐
   │ PaymentMethod │                    │  Doctor  │─────────▶│ Patient  │
   │  + charge()   │                    │ +treat() │  treats  │ +records │
   └───────△───────┘                    └──────────┘          └──────────┘
           │  (hollow triangle)          neither owns the other's lifecycle
     ┌─────┴──────┐
     │            │
┌────┴─────┐ ┌────┴─────┐
│CreditCard│ │UpiWallet │
└──────────┘ └──────────┘


   AGGREGATION ("has-a", weak)          COMPOSITION ("owns-a", strong)
   ┌──────────┐          ┌────────┐     ┌──────────┐          ┌────────┐
   │   Team   │◇─────────│ Player │     │  House   │◆─────────│  Room  │
   │ +roster  │  0..*    │ +name  │     │+floorPlan│  1..*    │ +area  │
   └──────────┘          └────────┘     └──────────┘          └────────┘
      hollow diamond                        filled diamond
   Player is passed IN                  Room is `new`-ed INSIDE House
   Player survives Team                 Room dies with House
```

### Diagram 2: The lifecycle — who calls `new`, who holds the reference

This is the diagram that actually settles the aggregation-vs-composition argument.

```
  AGGREGATION                              COMPOSITION
  ═══════════                              ═══════════

  ┌──────────────────┐                     ┌──────────────────┐
  │  Outside world   │                     │  Outside world   │
  │  (main / caller) │                     │  (main / caller) │
  └────────┬─────────┘                     └────────┬─────────┘
           │                                        │
    new Player('Virat')                      new House(addr, layout)
           │                                        │
           ▼                                        ▼
     ┌──────────┐                            ┌──────────────┐
     │  Player  │◀──── caller still          │    House     │
     └────┬─────┘      holds a ref!          │              │
          │                                  │ constructor: │
    team.addPlayer(player)                   │  layout.map( │
          │                                  │   x => new   │──┐
          ▼                                  │     Room(x)) │  │ House
     ┌──────────┐                            │              │  │ calls
     │   Team   │  holds a ref               │  #rooms ─────┼──┘ `new`
     │  #players│  (one of TWO refs)         │  (the ONLY   │
     └──────────┘                            │   reference) │
                                             └──────┬───────┘
   team.disband()                                   │
          │                                  house.demolish()
          ▼                                         │
     ┌──────────┐                                   ▼
     │  Player  │  STILL ALIVE                ┌──────────────┐
     │  (Virat) │  caller's ref keeps it      │  Room objects│
     └──────────┘  reachable                  │  UNREACHABLE │
                                              │  → collected │
                                              └──────────────┘
```

Read it like this: in **aggregation** there are always **at least two references** to the part — the caller's and the whole's — so destroying the whole leaves one behind. In **composition** the whole holds the **only** reference, so destroying it makes the part unreachable, and unreachable means gone.

### Diagram 3: The decision flowchart (memorise this)

```
                  ┌───────────────────────────────────┐
                  │  I have class A and class B.      │
                  │  How do they relate?              │
                  └────────────────┬──────────────────┘
                                   │
                                   ▼
            ┌──────────────────────────────────────────┐
            │ Is EVERY A a kind of B, and can an A be  │
            │ used ANYWHERE a B is expected, with no   │
            │ caller ever checking which one it got?   │
            └──────┬────────────────────────┬──────────┘
                YES│                        │NO
                   ▼                        ▼
           ┌───────────────┐   ┌─────────────────────────────────┐
           │  INHERITANCE  │   │ Does A HOLD B as part of its    │
           │  A extends B  │   │ own state (a field / a list)?   │
           │    "is-a"     │   └──────┬───────────────┬──────────┘
           └───────────────┘       NO │               │ YES
                                      ▼               ▼
                          ┌───────────────┐  ┌──────────────────────────┐
                          │  ASSOCIATION  │  │ If I DELETE A, should B  │
                          │  a.use(b)     │  │ still exist on its own?  │
                          │   "uses-a"    │  └────┬────────────────┬────┘
                          └───────────────┘    YES│                │NO
                                                  ▼                ▼
                                        ┌──────────────┐  ┌──────────────┐
                                        │ AGGREGATION  │  │ COMPOSITION  │
                                        │ B passed IN  │  │ A does `new B`│
                                        │   "has-a"    │  │   "owns-a"   │
                                        │ ◇ hollow ◇   │  │  ◆ filled ◆  │
                                        │ SET NULL     │  │  CASCADE     │
                                        └──────────────┘  └──────────────┘
```

---

## Real world examples

### 1. Stripe's API objects — composition and association side by side

In Stripe's data model, a `PaymentIntent` **composes** its `charges` — a charge cannot exist without the payment intent that produced it, and it is returned as a nested list on the intent. That's a filled diamond.

But a `PaymentIntent` merely **associates** with a `Customer`. The customer object lives independently, is referenced by id (`customer: "cus_123"`), is shared across thousands of payment intents, and absolutely does not get deleted when a payment intent is cancelled. That's a plain line.

The two are one field apart in the same JSON object, and they have opposite delete semantics. This is the distinction doing real work.

### 2. The DOM — textbook composition

In the browser DOM, an `Element` **owns** its child nodes. Call `parent.remove()` and every descendant goes with it — you cannot have an orphaned child node dangling in the document tree. `appendChild` even *moves* a node rather than copying it, precisely because a node has exactly one parent. One owner, one reference, cascade on delete: composition, all the way down.

Compare `element.addEventListener(fn)` — the listener function is **associated**, not owned. It was created elsewhere, may be attached to many elements, and survives the element's removal.

### 3. Relational ORMs (Prisma, TypeORM, Sequelize)

Every ORM makes you declare this relationship explicitly, whether you understand it or not:

```javascript
// Prisma-style schema, expressed as a comment for reference:
// model Order {
//   id    String      @id
//   lines OrderLine[]                      // COMPOSITION
// }
// model OrderLine {
//   order   Order  @relation(fields: [orderId], references: [id], onDelete: Cascade)
//   orderId String                          // NOT NULL + Cascade = owned
// }
// model Team {
//   id      String   @id
//   players Player[]                        // AGGREGATION
// }
// model Player {
//   team   Team?  @relation(fields: [teamId], references: [id], onDelete: SetNull)
//   teamId String?                          // nullable + SetNull = not owned
// }
```

The `onDelete` line **is** the aggregation-vs-composition decision, written in SQL. Getting it wrong is how you either lose data or leak orphans.

---

## Trade-offs

| Relationship | Pros | Cons | Coupling |
|---|---|---|---|
| **Inheritance** | Polymorphism for free; least code for true is-a hierarchies | Tightest coupling that exists — subclass depends on parent's *internals*; fragile base class problem; a change to the parent breaks every child | **Highest** |
| **Composition** | Clear ownership; easy cleanup; strong encapsulation (part is private) | Part cannot be shared or reused elsewhere; whole gets bigger; harder to test the part alone | Medium-high |
| **Aggregation** | Parts are reusable and shareable across wholes; parts are independently testable | You must manage the part's lifecycle yourself → orphans and dangling references if you're careless | Medium |
| **Association** | Loosest; each class is independently testable and replaceable | Says nothing about ownership, so cleanup responsibility is implicit; over-use gives you a spaghetti object graph | **Lowest** |

**Rule of thumb:** Reach for the *weakest* relationship that expresses the truth. Start at association. Escalate to aggregation only when the whole genuinely needs to hold the parts. Escalate to composition only when the part is meaningless outside the whole. Reach for inheritance **last**, and only when substitutability is genuinely true — not because you want to reuse a method.

---

## Common interview questions on this topic

### Q1: "What's the difference between aggregation and composition?"
**Hint:** Do not say "one is weak and one is strong" and stop — that's the answer everyone gives. Answer with **lifecycle**: in composition, the whole *creates* the part inside its own constructor and holds the only reference, so deleting the whole deletes the part (`House`/`Room`, `Order`/`OrderLine`, `ON DELETE CASCADE`). In aggregation, the part is created *outside* and passed in, may be shared by several wholes, and outlives the whole (`Team`/`Player`, `ON DELETE SET NULL`). Then give the one-line test: *"if I delete the whole, does it make sense for the part to still exist?"*

### Q2: "When would you use composition instead of inheritance?"
**Hint:** Almost always. Inheritance is only correct when the subclass is genuinely substitutable everywhere the parent is used (that's LSP, doc 16). If you're inheriting to *reuse code*, that's the wrong reason — compose instead. Give the `Stack extends Array` example: the inherited `splice()` destroys LIFO semantics, so `Stack` is not substitutable for `Array`. `Stack` should *hold* a private array. See [26 — Composition Over Inheritance](./26-composition-over-inheritance.md).

### Q3: "Is a `Car` and its `Engine` composition or aggregation?"
**Hint:** This is a trick — **it depends on your domain**, and saying so is the correct answer. In a car-manufacturing system where engines are built into cars and never removed, it's composition. In a mechanic's shop system where engines are pulled out, refurbished, and dropped into a *different* car, it's aggregation. Always ask: "in this system, can the part be transferred or outlive the whole?" The relationship is a property of the *model*, not of the real-world object.

### Q4: "Draw the UML for a `Playlist` containing `Song`s. Which diamond?"
**Hint:** Hollow diamond — **aggregation**. A Song exists in the library independently of any playlist, the same Song appears in many playlists, and deleting a playlist obviously must not delete the song files. Contrast: a `Playlist` and its `PlaylistEntry` (song + position + added-at) *is* composition — the entry is meaningless without its playlist and cascades on delete. Naming that second class is what gets you the senior signal.

### Q5: "Give me a case where inheritance is clearly wrong."
**Hint:** `class Square extends Rectangle`. A Rectangle has independent `setWidth`/`setHeight`. A Square cannot honour both — setting the width must change the height. So a Square is *not* substitutable for a Rectangle, and any function that does `r.setWidth(5); r.setHeight(4); assert(r.area() === 20)` breaks. The "is-a" reads fine in English and is false in code. That's exactly the gap LSP exists to catch.

---

## Practice exercise

### The Music Streaming Model (~30 min)

Model a small music-streaming domain in JavaScript. Your classes: `User`, `Playlist`, `PlaylistEntry`, `Song`, `Artist`, `Album`, `PremiumUser`.

**Part A — Classify (10 min).** For each pair below, write down the relationship (inheritance / association / aggregation / composition) **and the one-sentence justification using the delete test**:

1. `PremiumUser` → `User`
2. `Playlist` → `PlaylistEntry`
3. `PlaylistEntry` → `Song`
4. `Album` → `Song`
5. `User` → `Playlist`
6. `Song` → `Artist`
7. `PlaybackService` → `Song`

**Part B — Implement (20 min).** Write the classes so that the relationships are *enforced by the code*, not just by comments:

- The composition relationships must have the whole call `new` on the part inside its own constructor, and store it in a **private (`#`) field**. There must be **no way** for outside code to obtain or transplant the part.
- The aggregation relationships must take the part as a **constructor argument or an `addX()` method**, and must have a `remove`/`disband` operation that provably leaves the part alive.
- Write a `demo()` function that proves both: destroy a `Playlist` and show its `Song`s are still playable; destroy an `Album` and show its `PlaylistEntry`... no wait — destroy an `Album` and show what *should* happen to a Song that has no other album (this one is a genuinely interesting design argument — write down which way you went and why).

**What to produce:** one `.js` file that runs with `node`, printing before/after state for each delete, plus a 7-line answer key for Part A. If your Part B code lets you `playlist.getEntry(0)` and hand that entry to another playlist, you've written aggregation while claiming composition — fix it.

---

## Quick reference cheat sheet

- **Inheritance = "is-a"** — `B extends A`. Only valid if B is substitutable for A *everywhere*. Tightest coupling. Use last.
- **Association = "uses-a"** — `a.doThing(b)`. No ownership at all. Loosest coupling. **Your default.**
- **Aggregation = "has-a"** — part is created outside and **passed in**. Part **survives** the whole. Hollow diamond `◇`. `ON DELETE SET NULL`.
- **Composition = "owns-a"** — whole calls `new` on the part **inside itself**, holds the **only** reference. Part **dies with** the whole. Filled diamond `◆`. `ON DELETE CASCADE`.
- **The one question:** *"If I delete the whole, should the part still exist?"* Yes → aggregation. No → composition.
- **The `new` test:** who calls `new`? If the whole does, in its own constructor, it's composition. If the caller does and hands it over, it's aggregation.
- **The reference-count test:** composition = whole holds the *only* reference (private field). Aggregation = at least two references exist.
- **Team/Player = aggregation.** House/Room, Order/OrderLine, Invoice/LineItem = **composition**.
- **Inheritance is for polymorphism, not code reuse.** Want to reuse a method? Compose.
- **`class Stack extends Array` is a bug**, not a design. Inherited `splice()` breaks the stack contract.
- **The same two real-world objects can be different relationships in different systems** — Car/Engine is composition in a factory, aggregation in a repair shop. It depends on your domain.
- **Prefer the weakest relationship that tells the truth.** Association < Aggregation < Composition < Inheritance, in order of increasing coupling.
- **UML:** hollow triangle = inheritance, plain line = association, hollow diamond = aggregation, filled diamond = composition. The diamond always sits on the **whole**.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [22 — UML Use Case Diagrams](./22-uml-use-case-diagrams.md) — modelling what actors can do, before you model how classes relate |
| **Next** | [24 — Coupling and Cohesion](./24-coupling-and-cohesion.md) — the forces that make you *prefer* the weaker relationships |
| **Related** | [20 — UML Class Diagrams](./20-uml-class-diagrams.md) — the notation (diamonds, triangles, multiplicities) for drawing all four |
| **Related** | [26 — Composition Over Inheritance](./26-composition-over-inheritance.md) — why the "is-a" arrow is the one you should reach for last |
| **Related** | [16 — SOLID: Liskov Substitution](./16-solid-liskov-substitution.md) — the formal rule that decides when inheritance is legitimate |
