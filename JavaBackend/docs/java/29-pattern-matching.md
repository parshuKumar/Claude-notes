# 29 — Pattern Matching — instanceof, switch, Record Patterns, Exhaustiveness

## Phase: 2 — Modern Java
## Category: CORE
## Java baseline: 21  |  Notes features from: 21, 25
## Project spine: N/A (spine starts at Topic 35)

---

## ELI5 anchor

You are sorting parcels at a depot.

**The old way.** You pick up a parcel. You squint at it: "is this a fragile one?" Yes.
So you put it down, get a fragile-parcel trolley, pick it up again as a fragile parcel,
and *then* read the fragile-parcel label. Two lifts, and if you guessed wrong on the
first squint, you drop it.

```java
if (event instanceof OrderPaid) {              // squint
    OrderPaid paid = (OrderPaid) event;        // pick it up AGAIN, as the right type
    process(paid.amount());                    // now read the label
}
```

**Pattern matching.** You pick it up once. "Is this a fragile parcel? If so, call it
`paid` and here is its weight and destination already unpacked." One lift.

```java
if (event instanceof OrderPaid paid) {         // test AND name AND cast, in one move
    process(paid.amount());
}
```

**Record patterns** go further. Not just "call it `paid`", but "open the box and hand me
what is inside":

```java
if (event instanceof OrderPaid(OrderId id, Instant at, Money amount, String ref)) {
    process(amount);        // `amount` is right there. No `paid.amount()` needed.
}
```

And the checklist idea from Topic 28 comes back. If the depot has posted the complete
list of parcel kinds, your sorting checklist can be **proved complete** by the compiler.

Then the sting, and it is the single most important sentence in this document:

**If you add a row to your checklist that says "anything else — put it in the corner",
the compiler stops checking your checklist for gaps.** It has no reason to. You told it
you handle everything. That row is called `default`, and adding it throws away the
entire guarantee.

---

## The bridge from what you know

### TypeScript's control-flow narrowing

```ts
function handle(event: OrderEvent): string {
  if (event.kind === "paid") {
    // TypeScript NARROWED event to OrderPaid here.
    return `paid ${event.amount}`;    // .amount is available, no cast written
  }
  return "other";
}
```

You do not write a cast. The compiler tracks that inside the `if`, `event` is
`OrderPaid`, and lets you reach its properties. That is **control-flow-based type
narrowing**, and it is one of TypeScript's best features.

### Java's version

```java
static String handle(OrderEvent event) {
    if (event instanceof OrderPaid paid) {
        return "paid " + paid.amount();     // `paid` is OrderPaid inside this block
    }
    return "other";
}
```

Same idea, different mechanics: TypeScript narrows the *existing* variable, Java
introduces a *new* variable that is already the narrowed type.

### The verdict

| TypeScript | Java | Verdict |
|---|---|---|
| `if (x.kind === "paid")` narrows `x` | `if (x instanceof OrderPaid p)` binds `p` | **PARTIAL** — same outcome, but Java gives you a new name rather than re-typing the old one. |
| Narrowing by a discriminant **property** | Narrowing by **type** | **PARTIAL** — Java has no property to forget to set. |
| `switch (x.kind)` + `never` check | `switch (x)` over a sealed type | **HONEST ANALOGUE** — including compile-time exhaustiveness. |
| Destructuring `const { amount, ref } = event` | Record pattern `OrderPaid(var amount, var ref)` | **PARTIAL** — Java's is **positional**, not by name, and it is a *type test plus* a destructure. |
| Nested destructuring `const {a: {b}} = x` | Nested record pattern `A(B(var c))` | **HONEST ANALOGUE** |
| Narrowing survives into later statements after an early return | Flow scoping does the same | **HONEST ANALOGUE** — see the `!(x instanceof T t)` form below. |
| `switch(true) { case x > 5: ... }` | `case Integer i when i > 5 ->` | **PARTIAL** — Java's guard is attached to a pattern, not standalone. |

**The one difference that will actually bite you:** TypeScript's destructuring is
**by property name**. Java's record deconstruction is **by position**.

```ts
const { amount, reference } = event;      // order irrelevant
```
```java
case OrderPaid(OrderId id, Instant at, Money amount, String reference) -> ...
//             ^ you must list EVERY component, IN ORDER
```

Reorder the record's components and every deconstruction pattern silently binds the
wrong names — if the types happen to be compatible. That is a real trap and it is Trap 5.

---

## What is this?

Pattern matching is **testing the shape of a value and extracting parts of it in one
expression**. Java has it in three places, all final in Java 21:

1. **`instanceof` with a pattern** (final since Java 16) — test, cast and name in one.
2. **`switch` with patterns** (final since Java 21) — dispatch on type, with guards,
   `null` handling, and exhaustiveness checking.
3. **Record patterns** (final since Java 21) — deconstruct a record into its components,
   nestable to any depth.

None of these need `--enable-preview` on Java 21. The only thing in this document that
is preview is **primitive types in patterns**, flagged `[JAVA 25]` below.

The reason all three matter together: **sealed types (Topic 28) + records (Topic 27) +
pattern-matching switch = an algebraic data type with compiler-checked exhaustiveness.**
Take away any one of the three and you lose the guarantee.

---

## Why does it matter?

**1. It eliminates the test-then-cast pair, which is a place bugs live.**
The old shape tests one type and casts to another whenever someone edits carelessly.
`if (x instanceof A) { B b = (B) x; }` compiles. Pattern matching makes it
unrepresentable.

**2. Exhaustiveness turns "did we handle every case?" into a build failure.**
This is Topic 28's payoff, and it is delivered here.

**3. It makes deeply-shaped data readable.**
Three levels of `getX().getY().getZ()` with null checks at each level becomes one
nested pattern that either matches entirely or does not match at all.

**4. Dominance errors catch ordering mistakes at compile time.**
Put a general case before a specific one and you get a compile error, not a silently
unreachable branch. Every `if`/`else if` chain you have ever written could shadow a
branch silently. This one cannot.

**5. And the reason it can hurt you:** one keyword — `default` — silently removes the
guarantee, and nothing warns you.

---

## Syntax breakdown

### 1. Type patterns in `instanceof`

```java
if (payload instanceof OrderPaid paid) {
//  ^^^^^^^            ^^^^^^^^^ ^^^^
//  the value          the type  the PATTERN VARIABLE
    use(paid.amount());
}
```

| Bit of syntax | What it means |
|---|---|
| `instanceof OrderPaid paid` | A **type pattern**. If `payload` is an `OrderPaid`, bind it to a new variable `paid` of type `OrderPaid`. |
| `paid` | The **pattern variable**. It is `final`-ish in practice, effectively a new local. |
| The whole expression | Evaluates to `boolean`, exactly like plain `instanceof`. |

**`null` never matches a type pattern.** `null instanceof OrderPaid paid` is `false`,
never an NPE. This is a genuine convenience — the null check comes free.

**Flow scoping** is the clever part. The pattern variable is in scope wherever the
compiler can prove the pattern matched:

```java
// In scope inside the if-block:
if (payload instanceof OrderPaid paid) { use(paid); }

// In scope in the && right-hand side:
if (payload instanceof OrderPaid paid && paid.amount().minorUnits() > 0) { ... }

// In scope AFTER the if, because the if returned:
if (!(payload instanceof OrderPaid paid)) {
    return "not a payment";
}
use(paid);          // legal! The compiler knows we only get here if it matched.

// NOT in scope — the compiler cannot prove it matched:
if (payload instanceof OrderPaid paid || somethingElse) {
    use(paid);      // compile error: paid may not have been initialized
}
```

That third form — negate, return early, then use the variable — is the idiom you will
write most often. It is Java's answer to TypeScript's early-return narrowing, and it
keeps the happy path unindented.

### 2. Type patterns in `switch`

```java
static String describe(Object o) {
    return switch (o) {
        case Integer i  -> "int " + i;
        case String s   -> "string of length " + s.length();
        case int[] arr  -> "int array of " + arr.length;
        case null       -> "nothing";                 // explicit null case
        default         -> "something else";
    };
}
```

| Bit of syntax | What it means |
|---|---|
| `switch (o)` where `o` is a reference type | A **pattern switch**. Traditional switches only allowed `int`, `String`, and enums. |
| `case Integer i ->` | A type pattern as a case label. |
| `->` (arrow) | No fall-through, no `break` needed. This is the "enhanced switch" form. |
| `case null` | **Must be written explicitly** if you want to handle null. See Trap 3. |
| `default` | Matches anything not matched above — **including nothing you forgot**. See Trap 1. |
| `yield` (in a block body) | Returns a value from a `{ }` case body in a switch *expression*. |

Block bodies when you need statements:

```java
return switch (result) {
    case Approved a -> {
        auditLog.record(a);
        metrics.increment("payment.approved");
        yield OrderStatus.PAID;             // `yield`, not `return`
    }
    case Declined d -> OrderStatus.DECLINED;
};
```

### 3. Guards — `when`

```java
return switch (result) {
    case Approved a when a.capturedAmount().minorUnits() == 0 -> "zero-value approval";
    case Approved a                                            -> "approved " + a.reference();
    case Declined d when d.reason() == DeclineReason.SUSPECTED_FRAUD -> "fraud review";
    case Declined d                                            -> "declined";
    case Failed f                                              -> "failed";
};
```

| Bit of syntax | What it means |
|---|---|
| `when <boolean expression>` | A **guard**. The case matches only if the pattern matches **and** the guard is true. |
| Ordering | Guarded cases must come before the equivalent unguarded case, or the unguarded one dominates them (Trap 2). |

**Critical rule:** a **guarded** pattern **never** counts toward exhaustiveness. The
compiler cannot evaluate your boolean expression at compile time, so it must assume the
guard could be false. If every case for a variant is guarded, the switch is not
exhaustive. That is Trap 4.

### 4. Record patterns — deconstruction

```java
record Money(long minorUnits, String currency) { }
record Approved(String reference, Money amount) implements PaymentResult { }

return switch (result) {
    case Approved(String reference, Money amount) ->
            "ref " + reference + " for " + amount.minorUnits();
    ...
};
```

| Bit of syntax | What it means |
|---|---|
| `Approved(String reference, Money amount)` | A **record pattern**. Tests that the value is an `Approved`, then binds its components positionally. |
| Component count | You must list **every** component. There is no partial deconstruction and no `...rest`. |
| `var` | Allowed for any component: `case Approved(var reference, var amount) ->`. |
| Nesting | Any component pattern may itself be a record pattern. |

**Nested deconstruction:**

```java
case Approved(String reference, Money(long minorUnits, String currency)) ->
        "ref " + reference + ": " + minorUnits + " " + currency;
//        `minorUnits` and `currency` bound directly, two levels down.
```

**The nesting also tests types**, which is the powerful part:

```java
sealed interface Shape permits Circle, Rectangle { }
record Point(double x, double y) { }
record Circle(Point centre, double radius) implements Shape { }
record Rectangle(Point topLeft, Point bottomRight) implements Shape { }

static String describe(Object o) {
    return switch (o) {
        // matches only a Circle whose centre is exactly the origin
        case Circle(Point(var x, var y), var r) when x == 0 && y == 0 ->
                "circle at origin, radius " + r;
        case Circle(Point p, var r) -> "circle at " + p + " radius " + r;
        case Rectangle(Point tl, Point br) -> "rect " + tl + " to " + br;
        default -> "not a shape";
    };
}
```

A **record pattern does not match null**. `case Approved(var ref, var amount)` will not
match if the value is null, and — importantly — will also not match if `result` is an
`Approved` whose `amount` component is null, because `Money(long m, String c)` is itself
a pattern that null fails. That gives you deep null-safety for free, and it is a real
difference from writing `a.amount().minorUnits()` by hand.

### 5. `[JAVA 25]` Primitive types in patterns — JEP 507

Java 25 continues the work to allow primitive type patterns everywhere:

```java
// [JAVA 25 - PREVIEW] requires --enable-preview
Object o = 42;
if (o instanceof int i) { ... }

switch (value) {
    case int i when i > 100 -> "large int";
    case int i              -> "int " + i;
    case long l             -> "long " + l;
    ...
}
```

> **Honesty flag, and it matters here.** As of JDK 25 this is JEP 507, *Primitive Types
> in Patterns, instanceof, and switch* — a **preview** feature, not final. Preview
> features require `--enable-preview` and can change or be withdrawn (Java's string
> templates were withdrawn after two previews; see Topic 30). I am not going to assert
> its status on *your* JDK. **The command that settles it is in the Hands-on proof
> below**: try compiling without `--enable-preview` and read what `javac` says.

**The Java 21-compatible fallback** — use the boxed type:

```java
// Compiles on 21, no preview flag, no --enable-preview
if (o instanceof Integer i) {
    int primitive = i;                       // unboxing; can NPE if you got here wrong
}

switch (o) {
    case Integer i when i > 100 -> "large int";
    case Integer i              -> "int " + i;
    case Long l                 -> "long " + l;
    default                     -> "other";
}
```

This costs you an allocation on the boxing path (Topic 01) and it will not let you
match `int` against `long` with a widening test. For 99% of application code the boxed
form is fine. Do not build production code on a preview feature.

### 6. `[JAVA 25]` — a note on what is NOT new

**Everything else in this document is final in Java 21.** Type patterns in `instanceof`
(16), pattern switches (21), record patterns (21), guards (21), `case null` (21). If
something here does not compile for you, the cause is almost certainly a `--release` or
`--source` set below 21, not a missing preview flag. `javac --version` and your build's
compiler configuration settle it.

---

## Example 1 — minimal

```java
public class PatternBasics {

    sealed interface Shape permits Circle, Square, Rectangle { }
    record Circle(double radius)                  implements Shape { }
    record Square(double side)                    implements Shape { }
    record Rectangle(double width, double height) implements Shape { }

    // 1. instanceof pattern
    static String legacyName(Object o) {
        if (o instanceof Circle c) {
            return "circle r=" + c.radius();
        }
        return "not a circle";
    }

    // 2. exhaustive switch over a sealed type — NO default
    static double area(Shape shape) {
        return switch (shape) {
            case Circle c    -> Math.PI * c.radius() * c.radius();
            case Square s    -> s.side() * s.side();
            case Rectangle r -> r.width() * r.height();
        };
    }

    // 3. record deconstruction + guard
    static String classify(Shape shape) {
        return switch (shape) {
            case Circle(double r) when r > 100        -> "huge circle";
            case Circle(double r)                     -> "circle of radius " + r;
            case Square(double s)                     -> "square of side " + s;
            case Rectangle(double w, double h) when w == h -> "square-ish rectangle";
            case Rectangle(double w, double h)        -> w + " by " + h;
        };
    }

    public static void main(String[] args) {
        Shape[] shapes = { new Circle(2), new Circle(500), new Square(3), new Rectangle(4, 4) };
        for (Shape s : shapes) {
            System.out.printf("%-40s area=%.2f  %s%n", s, area(s), classify(s));
        }
    }
}
```

**Do the experiment.** Delete the `case Square s ->` line from `area` and compile:

```
error: the switch expression does not cover all possible input values
```

Now put it back, and instead add `default -> 0;` to `area`. Delete `case Square s ->`
again. **It compiles.** Squares silently have area zero.

That contrast is the entire lesson of this document. Everything else is detail.

---

## Example 2 — production scenario

`orderflow` has a projector that maintains the order read model from a stream of
events, and a payment handler that reacts to gateway outcomes. Both are exhaustive
switches over sealed hierarchies (Topic 28).

### The version this replaces

```java
public void project(OrderEvent event) {
    if (event instanceof OrderPlaced) {
        OrderPlaced e = (OrderPlaced) event;
        readModel.insert(e.orderId(), e.userId(), e.lines());
    } else if (event instanceof OrderPaid) {
        OrderPaid e = (OrderPaid) event;
        readModel.setStatus(e.orderId(), "PAID");
        readModel.setPaidAmount(e.orderId(), e.amount());
    } else if (event instanceof OrderShipped) {
        OrderShipped e = (OrderShipped) event;
        readModel.setStatus(e.orderId(), "SHIPPED");
        readModel.setTracking(e.orderId(), e.trackingNumber());
    } else {
        log.warn("unhandled event type {}", event.getClass().getSimpleName());
    }
}
```

Four problems, and only one of them is verbosity:

1. **The `else` swallows every future event type**, logging a warning nobody reads. Add
   `OrderCancelled` and the read model silently stops updating for cancellations. You
   find out from a customer.
2. **Every branch is a test plus a separate cast**, so a copy-paste error can test one
   type and cast to another. It compiles.
3. **`e` is shadowed four times**, which is legal and confusing.
4. **Reordering the branches** can silently make one unreachable if the types are
   related. No error.

### The pattern-matching version

```java
@Transactional
public void project(OrderEvent event) {
    switch (event) {

        case OrderPlaced(OrderId id, Instant at, long seq, UserId user, List<OrderLine> lines) ->
                readModel.insert(id, user, lines, at, seq);

        case OrderPaid(OrderId id, Instant at, long seq, Money amount, String gatewayRef) -> {
            readModel.setStatus(id, OrderStatus.PAID, seq);
            readModel.recordPayment(id, amount, gatewayRef, at);
        }

        case OrderShipped(OrderId id, Instant at, long seq, String carrier, String tracking) -> {
            readModel.setStatus(id, OrderStatus.SHIPPED, seq);
            readModel.setTracking(id, carrier, tracking);
        }

        case OrderCancelled(OrderId id, Instant at, long seq, String reason, UserId by) -> {
            readModel.setStatus(id, OrderStatus.CANCELLED, seq);
            readModel.recordCancellation(id, reason, by, at);
        }

        case OrderRefunded(OrderId id, Instant at, long seq, Money amount, String reason) -> {
            readModel.setStatus(id, OrderStatus.REFUNDED, seq);
            readModel.recordRefund(id, amount, reason, at);
        }

        // NO default. Deliberately. See the comment below.
    }
}
```

```java
/*
 * There is no `default` in this switch, and there must never be one.
 * OrderEvent is sealed; the compiler proves this switch handles every variant.
 * Adding `default` would make that proof impossible, and a new event type would
 * be silently dropped from the read model instead of breaking the build.
 * If you are here because you added an event type: add a case. That is the design.
 */
```

**Write that comment.** It is the single highest-value comment in a codebase that uses
sealed types, because the thing it prevents is invisible.

### Guards and nesting, where the domain actually needs them

```java
public NextAction decide(PaymentResult result, Order order) {
    return switch (result) {

        // Deconstruct two levels: Approved holds a Money, Money holds minorUnits.
        case Approved(var attemptId, var at, var ref, Money(long minor, var currency))
                when minor < order.totalMinorUnits() ->
                NextAction.partialCapture(order.id(), minor, ref);

        case Approved(var attemptId, var at, var ref, var amount) ->
                NextAction.markPaid(order.id(), ref, amount);

        // Guard on an enum inside the variant
        case Declined(var attemptId, var at, DeclineReason reason, var msg)
                when reason == DeclineReason.INSUFFICIENT_FUNDS ->
                NextAction.offerAlternativePayment(order.id());

        case Declined(var attemptId, var at, DeclineReason reason, var msg)
                when reason == DeclineReason.SUSPECTED_FRAUD ->
                NextAction.holdForManualReview(order.id(), msg);

        // The UNGUARDED Declined case. This one is what makes the switch exhaustive.
        case Declined d ->
                NextAction.markDeclined(order.id(), d.reason());

        case Failed(var attemptId, var at, var msg, boolean retryable) when retryable ->
                NextAction.scheduleRetry(order.id(), at);

        case Failed f ->
                NextAction.markFailed(order.id(), f.technicalMessage());

        case PendingAuthentication(var attemptId, var at, URI url, Instant expires) ->
                NextAction.redirectTo3ds(order.id(), url, expires);
    };
}
```

Read the structure:

- **Guarded cases come first**, most specific first. Ordering is significant and the
  compiler enforces it (Trap 2).
- **Each variant ends with an unguarded case.** That is what makes the switch
  exhaustive. Remove any one of those three unguarded cases and the build fails
  (Trap 4).
- **Deconstruction gives you the fields inline**, so the guard reads
  `when minor < order.totalMinorUnits()` rather than
  `when a.capturedAmount().minorUnits() < order.totalMinorUnits()`.
- **No null checks anywhere.** A record pattern does not match null at any level.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — adding `default` to a switch over a sealed type

**This is the most important trap in this document. Everything else is secondary.**

**Wrong:**
```java
return switch (event) {
    case OrderPlaced p  -> handlePlaced(p);
    case OrderPaid pd   -> handlePaid(pd);
    case OrderShipped s -> handleShipped(s);
    default -> {
        log.warn("unhandled event {}", event.getClass().getSimpleName());
        yield NextAction.none();
    }
};
```

**Exact symptom:** at the moment you write it, **nothing**. It compiles. Tests pass.

Then a colleague adds `OrderCancelled` to the sealed interface. They run the build. It
is green. Every switch in the codebase that has a `default` compiles happily. They ship.

In production:
- Cancelled orders never leave `PENDING` in the read model.
- Inventory reservations are never released, so stock leaks until someone notices
  oversell errors.
- The warning line goes to a log stream nobody has alerted on. If you are lucky
  somebody greps for it three weeks later.
- Your only signal is a business metric: cancellation rate on the dashboard is zero.

There is **no exception, no error, no failing test, and no code review signal**, because
the diff that broke it does not touch any of the affected files.

**Root cause:** exhaustiveness checking is the compiler proving that the listed cases
cover the whole domain of the switch selector. `default` covers everything by
definition. So the proof obligation is discharged trivially and the compiler has
nothing left to check. `default` is not "a safety net that also lets me keep
exhaustiveness" — it is **mutually exclusive** with exhaustiveness checking.

Restated so it sticks: **`default` converts a compile-time guarantee into a runtime log
line.** You traded the one thing sealed types are for against a warning nobody reads.

**Fix:** delete the `default`. Let the compiler do its job.

```java
return switch (event) {
    case OrderPlaced p     -> handlePlaced(p);
    case OrderPaid pd      -> handlePaid(pd);
    case OrderShipped s    -> handleShipped(s);
    case OrderCancelled c  -> handleCancelled(c);      // forced to add this
    case OrderRefunded r   -> handleRefunded(r);       // and this
};
```

**"But I want a runtime safety net for events from a newer producer version."**
That is a legitimate concern in a distributed system, and the answer is **not**
`default`. Handle the unknown-input problem at the **deserialization boundary**, where
unknown input actually arrives, not in your domain switch:

```java
// At the edge: deserialization can genuinely encounter an unknown discriminant.
OrderEvent event;
try {
    event = mapper.readValue(payload, OrderEvent.class);
} catch (InvalidTypeIdException e) {
    deadLetterQueue.send(payload, "unknown event type: " + e.getTypeId());
    metrics.increment("events.unknown_type");
    return;
}
// Past this line the value IS one of the sealed variants. The switch stays exhaustive.
project(event);
```

Now the unknown-type case has a real handler in the one place it can occur, and the
domain logic keeps its compile-time proof.

**Code review rule, and put it in the team's checklist:**
> A `default` (or a `case null, default`) in a switch over a sealed type is a defect
> unless there is a comment explaining why the exhaustiveness guarantee is being given
> up. In almost every case the honest answer is that it should not be.

**Detection at scale:** grep is enough to start —
`grep -rn "default ->" --include=*.java` — then check each hit's selector type. For a
permanent guard, ErrorProne and similar tools have checks in this area; wire one into
CI so the rule outlives the person who knows it.

---

### Trap 2 — dominance: a general pattern before a specific one

**Wrong:**
```java
return switch (result) {
    case PaymentResult r -> "any result";          // matches EVERYTHING first
    case Approved a      -> "approved " + a.reference();
    case Declined d      -> "declined";
};
```

**Exact symptom:** a **compile error**, and this is the good news:
```
error: this case label is dominated by a preceding case label
    case Approved a      -> "approved " + a.reference();
         ^
```

**Root cause:** `PaymentResult r` is a pattern that matches every `PaymentResult`,
including every `Approved`. So `case Approved a` is unreachable. Java makes unreachable
case labels a **compile error**, not a warning and not a silent shadow.

**Why this is worth celebrating:** the equivalent `if`/`else if` chain compiles
silently and produces dead code:
```java
if (result instanceof PaymentResult r) { return "any result"; }
else if (result instanceof Approved a) { return "..."; }   // dead. No error. No warning.
```
Same for the `switch (type) { case "x": ... }` string dispatch you might have written
in TypeScript. The pattern switch is the only one of the three that catches it.

**The subtlety that catches people — guards do not dominate.**
```java
return switch (result) {
    case Approved a when a.amount().minorUnits() == 0 -> "zero";
    case Approved a                                   -> "normal";   // FINE. Not dominated.
};
```
A guarded pattern is *conditional*, so it cannot dominate anything — the compiler cannot
prove the guard is always true. That is why guarded cases must come **first**: the
unguarded `case Approved a` **does** dominate the guarded one, so the reverse order is
a compile error.

```java
// WRONG ORDER - compile error
case Approved a                                   -> "normal";
case Approved a when a.amount().minorUnits() == 0 -> "zero";   // dominated
```

**Fix:** order from most specific to most general. Guarded before unguarded, subtype
before supertype. If you are unsure, write it and let `javac` tell you — this is one of
the rare cases where the compiler is a complete oracle.

---

### Trap 3 — `null` in a pattern switch

**Wrong:**
```java
String label = switch (statusFromDatabase) {     // a String column that is nullable
    case "PENDING" -> "Awaiting payment";
    case "PAID"    -> "Paid";
    default        -> "Unknown";
};
```

**Exact symptom:**
```
java.lang.NullPointerException
    at com.orderflow.orders.OrderLabeller.label(OrderLabeller.java:14)
```
on a line that has a `default` clause — which is exactly why people find it baffling.
"I have a default. How can it NPE?"

**Root cause:** `switch` on a reference type throws `NullPointerException` when the
selector is null, **and `default` does not catch it**. That is deliberate: it preserves
the behaviour of every `switch` written before Java 21, so upgrading a JDK does not
silently change what your existing switches do with null.

The rule in full:

| Switch selector is null, and the switch has... | Result |
|---|---|
| no `case null` | **`NullPointerException`** — even if there is a `default` |
| `case null ->` | that case runs |
| `case null, default ->` | that combined case runs |
| a total type pattern like `case Object o ->` | still **NPE**, unless `case null` is present |

**Fix — decide what null means and say so:**
```java
String label = switch (statusFromDatabase) {
    case null      -> "Unknown (no status recorded)";     // explicit
    case "PENDING" -> "Awaiting payment";
    case "PAID"    -> "Paid";
    default        -> "Unknown status: " + statusFromDatabase;
};
```

or combine them when null and unknown mean the same thing:
```java
    case null, default -> "Unknown";
```

**And the better fix:** ask why the value is nullable at all. If it comes from a
database column, a `NOT NULL` constraint plus a check constraint is a stronger control
than a `case null`. If it comes from a sealed hierarchy in your own code, records with
validating compact constructors (Topic 27) mean it never can be null.

**A related good-news note:** a switch over a **sealed** type where all variants are
handled *still* NPEs on a null selector. The exhaustiveness proof is about the variants,
not about null. Handle null at the boundary; do not let it into your domain.

---

### Trap 4 — every case is guarded, so nothing is exhaustive

**Wrong:**
```java
return switch (result) {
    case Approved a when a.capturedAmount().minorUnits() > 0 -> "approved";
    case Declined d when d.reason() != null                  -> "declined";
    case Failed f   when f.retryable()                       -> "retry";
};
```

**Exact symptom:**
```
error: the switch expression does not cover all possible input values
```
and the confusing part is that **every variant is mentioned**. People stare at it and
count three variants and three cases and conclude the compiler is wrong.

**Root cause:** a **guarded pattern never contributes to exhaustiveness**. The compiler
would have to evaluate `a.capturedAmount().minorUnits() > 0` at compile time to know
whether the case fires, which it cannot do in general (and will not attempt even for
constant expressions in this position). So it treats every guarded case as "might not
match" and concludes the switch has gaps.

This is correct and it is a good design: the alternative would be a compiler that
sometimes proves guards and sometimes does not, which is a far worse thing to reason
about.

**Fix — every variant needs one unguarded case, and it goes last for that variant:**
```java
return switch (result) {
    case Approved a when a.capturedAmount().minorUnits() > 0 -> "approved";
    case Approved a                                          -> "zero-value approval";

    case Declined d when d.reason() == DeclineReason.SUSPECTED_FRAUD -> "fraud review";
    case Declined d                                                  -> "declined";

    case Failed f when f.retryable() -> "retry";
    case Failed f                    -> "permanent failure";
};
```

Six cases for three variants. That is the correct shape and it is more code than people
expect the first time. The compensation is that every branch is now reachable and
provably so.

---

### Trap 5 — deep deconstruction that breaks silently when a record changes

**Wrong:**
```java
record Address(String line1, String city, String postcode, String country) { }
record Customer(UserId id, String name, Address address) { }
record Order(OrderId id, Customer customer, Money total) { }

case Order(var orderId,
           Customer(var userId, var name, Address(var line1, var city, var postcode, var country)),
           var total) when country.equals("GB") -> applyUkVat(total);
```

**Exact symptom, version A — the loud one:** someone adds a `county` component to
`Address`. Every deconstruction pattern fails to compile:
```
error: incorrect number of nested patterns in record pattern
```
Annoying but safe. You fix them and move on.

**Exact symptom, version B — the quiet one, and the reason this is a trap:** someone
*reorders* `Address` to `(String line1, String city, String country, String postcode)`
— perhaps to match a form layout. All four components are `String`. **The pattern still
compiles.** `country` is now bound to the postcode and `postcode` to the country.

`when country.equals("GB")` is now comparing a postcode to `"GB"`. It is never true. UK
VAT stops being applied. There is no exception, no test failure unless you happened to
have a fixture that exercises it, and no compile error. You find it in a tax
reconciliation.

**Root cause:** record deconstruction is **positional**, not by name. The names in the
pattern are bindings you choose; they are not checked against the record's component
names. That is a deliberate language design decision (it is what lets you write
`case Approved(var a, var b)`), and it makes patterns structurally coupled to component
*order*.

**Fix — three levels, use whichever fits:**

1. **Do not deconstruct deeper than you need.** Bind the intermediate and use accessors
   for the rest:
   ```java
   case Order(var orderId, Customer c, var total) when c.address().country().equals("GB") ->
           applyUkVat(total);
   ```
   Accessors are name-based, so a reorder cannot silently rebind them. This is the fix
   you will use most.

2. **Make the components different types**, so a reorder cannot type-check:
   ```java
   record Address(AddressLine line1, City city, Postcode postcode, CountryCode country) { }
   ```
   Wrapper records around `String` are cheap (Topic 27) and they turn a silent
   misbinding into a compile error. This is the same argument as "do not have four
   `String` parameters in a row", and it applies to patterns too.

3. **Treat record component order as part of the public API.** Add a comment on the
   record saying so, and reject reordering PRs. Weakest of the three, because it relies
   on people.

**The general rule:** deconstruct one level for readability. Beyond that, the coupling
cost usually exceeds the benefit, and named accessors are safer.

---

## Hands-on proof

Every command below is one **you** run. I do not have a JVM and will not print output
and claim it is real. What follows is exactly what to look for and how to read each
possible result.

### Setup

```bash
mkdir -p ~/java-lab/29 && cd ~/java-lab/29
java --version      # expect 21 or 25
javac --version
```

`Events.java` — used by several proofs:
```java
public sealed interface Event permits Placed, Paid, Shipped { }

record Placed(String orderId, long lines)              implements Event { }
record Paid(String orderId, long amountMinor)          implements Event { }
record Shipped(String orderId, String trackingNumber)  implements Event { }
```

### Proof 1 — exhaustiveness, and watching `default` destroy it

**This is the proof that matters. Do this one even if you skip the others.**

`Exhaustive.java`:
```java
public class Exhaustive {
    static String describe(Event e) {
        return switch (e) {
            case Placed p  -> "placed " + p.lines() + " lines";
            case Paid pd   -> "paid " + pd.amountMinor();
            case Shipped s -> "shipped via " + s.trackingNumber();
        };
    }
    public static void main(String[] args) {
        System.out.println(describe(new Paid("o-1", 1999)));
    }
}
```

**Step 1 — baseline.**
```bash
javac Events.java Exhaustive.java && java Exhaustive
```
Expect it to compile and run.

**Step 2 — remove a case.** Delete the `case Shipped s ->` line. Recompile.

| What you see | What it means |
|---|---|
| `error: the switch expression does not cover all possible input values` | The guarantee, working. The compiler read `permits Placed, Paid, Shipped` and proved you missed one. |
| It compiles | Check your `--release`/`--source`. Pattern switches need 21+. Also check you did not leave a `default`. |

**Step 3 — the destructive step. Put `Shipped` back, then add a `default`:**
```java
        return switch (e) {
            case Placed p  -> "placed " + p.lines() + " lines";
            case Paid pd   -> "paid " + pd.amountMinor();
            case Shipped s -> "shipped via " + s.trackingNumber();
            default        -> "unknown event";
        };
```
Compile — fine. **Now delete `case Shipped s ->` again** and recompile.

| What you see | What it means |
|---|---|
| **It compiles cleanly.** No error, no warning. | **The single most important result in this document.** The `default` discharged the compiler's proof obligation, so it no longer checks for gaps. A missing variant is now a silent runtime behaviour instead of a build failure. |
| An error about `Shipped` | You did not actually save the file with the `default` in it. Re-check. |

**Step 4 — see the consequence.** Add `System.out.println(describe(new Shipped("o-1", "TRK-9")));`
to `main` and run. It prints `unknown event`. In `orderflow` that line is "the read
model silently stops updating for shipped orders".

**Step 5 — add a fourth variant** (`record Cancelled(String orderId) implements Event`,
plus `Cancelled` in `permits`). With the `default` present: compiles. Remove the
`default`: compile error at `describe`. **That is the difference, demonstrated in your
own terminal.**

### Proof 2 — dominance is an error, not a shadow

`Dominance.java`:
```java
public class Dominance {
    static String bad(Event e) {
        return switch (e) {
            case Event any -> "any";
            case Paid p    -> "paid";      // unreachable
            case Placed pl -> "placed";
            case Shipped s -> "shipped";
        };
    }
}
```

```bash
javac Events.java Dominance.java
```

| What you see | What it means |
|---|---|
| `error: this case label is dominated by a preceding case label` | Expected. Unreachable case labels are a compile error in a pattern switch. |
| No error | You wrote the cases in a different order. Re-check that `case Event any` is first. |

**Now do the contrast that shows why this matters.** Write the same logic as an
`if`/`else if` chain:
```java
static String badIf(Event e) {
    if (e instanceof Event any) return "any";
    else if (e instanceof Paid p) return "paid";      // dead code
    return "?";
}
```
Compile it. **It compiles with no error and no warning.** The `else if` is dead and
nothing tells you. That is the ordering bug the pattern switch makes impossible.

**Then check the guard nuance:**
```java
static String guarded(Event e) {
    return switch (e) {
        case Paid p    -> "paid";
        case Paid p when p.amountMinor() == 0 -> "zero";    // dominated -> error
        case Placed pl -> "placed";
        case Shipped s -> "shipped";
    };
}
```
Expect `error: this case label is dominated`. Swap the two `Paid` lines and it compiles.
**What it means:** unguarded dominates guarded, never the reverse — so guards go first.

### Proof 3 — null does not go to `default`

`NullProof.java`:
```java
public class NullProof {
    static String withDefault(Object o) {
        return switch (o) {
            case Integer i -> "int";
            case String s  -> "string";
            default        -> "other";
        };
    }

    static String withNullCase(Object o) {
        return switch (o) {
            case null      -> "null!";
            case Integer i -> "int";
            case String s  -> "string";
            default        -> "other";
        };
    }

    public static void main(String[] args) {
        System.out.println(withNullCase(null));
        try {
            System.out.println(withDefault(null));
        } catch (Exception e) {
            System.out.println("withDefault(null) threw " + e);
        }
    }
}
```

```bash
java NullProof.java
```

| What you see | What it means |
|---|---|
| `null!` then `withDefault(null) threw java.lang.NullPointerException` | Expected, and the point: `default` does **not** catch null. This preserves pre-21 switch behaviour so a JDK upgrade cannot silently change your code. |
| Both print a value | Your JDK behaves differently from what I described — report it with `java --version`. I would want to see that. |

Also try `case null, default -> "null or other";` in place of the two separate labels
and confirm it handles both.

### Proof 4 — how the switch is actually implemented

```bash
javac Events.java Exhaustive.java
javap -c Exhaustive.class
```

**What to look for** in `describe`:

- An **`invokedynamic`** instruction near the top, not a chain of `instanceof` tests.
- A `BootstrapMethods:` section referencing **`java/lang/runtime/SwitchBootstraps`**,
  usually `typeSwitch`.
- After the indy call, a `tableswitch` or `lookupswitch` on the **integer index** that
  the bootstrap returned.

| What you see | What it means |
|---|---|
| `invokedynamic` + `SwitchBootstraps.typeSwitch` + a `tableswitch` on an int | Expected. The pattern switch is compiled to "ask the runtime which case index matches, then jump". It is not N sequential `instanceof` checks, which matters for a switch with many cases. |
| A long sequence of `instanceof` and `ifeq` | You compiled an `if`/`else if` chain, or a very small switch the compiler chose to lower differently. Compare against a switch with five or more cases. |
| No `invokedynamic` at all | Check `--release`. |

**How to read the significance:** same machinery as records (Topic 27) and lambdas
(Topic 21). Java keeps moving code generation from `javac` into runtime bootstraps so
the strategy can improve without recompiling your code. It also means the case-matching
cost is not obviously linear in the number of cases — but do **not** conclude anything
about performance from this; that is a JMH question (Topic 77), not a `javap` question.

### Proof 5 — is `[JAVA 25]` primitive-pattern support preview on your JDK?

This is the one place in this document where I am genuinely uncertain about your
environment, so here is the command that settles it rather than a claim.

`PrimitivePattern.java`:
```java
public class PrimitivePattern {
    public static void main(String[] args) {
        Object o = 42;
        if (o instanceof int i) {
            System.out.println("matched int " + i);
        }
    }
}
```

**Step 1 — try it with no flags at all:**
```bash
javac PrimitivePattern.java
```

| What you see | What it means |
|---|---|
| `error: primitive types in patterns, instanceof, and switch is a preview feature and is disabled by default` (or similar wording naming *preview*) | Confirmed: on your JDK this is **preview**. Do not use it in production code. Use the boxed `Integer` fallback. |
| It compiles cleanly | The feature is final on your JDK. Note your `java --version` — that is newer than my Java 25 information, and you should trust your compiler over this document. |
| `error: unexpected type / required: reference` | Your JDK predates the feature entirely. Use the boxed fallback. |

**Step 2 — if it said preview, confirm it works with the flag:**
```bash
javac --release 25 --enable-preview PrimitivePattern.java
java  --enable-preview PrimitivePattern
```

| What you see | What it means |
|---|---|
| A warning that you are using preview features, then `matched int 42` | Confirmed preview and working. Preview features may change or be removed in a later release — Java's string templates were withdrawn after two previews (Topic 30). Keep them out of anything you deploy. |
| `error: invalid flag: --enable-preview` on `javac` without `--release` | Preview needs `--release <current>` alongside `--enable-preview`. |
| `java.lang.UnsupportedClassVersionError` at runtime | You compiled with preview on one JDK and ran on another. Preview class files are pinned to the exact JDK version that produced them, by design. |

**Step 3 — the 21-compatible version, which you should actually write:**
```java
Object o = 42;
if (o instanceof Integer i) {
    int primitive = i;
    System.out.println("matched Integer " + primitive);
}
```
```bash
javac --release 21 PrimitivePatternBoxed.java && java PrimitivePatternBoxed
```
Expect a clean compile with no flags. This is the code you ship.

---

## Practice exercises

### 1 — Easy: eliminate the casts

Rewrite each of these with pattern matching. No explicit casts, no `getClass()`
comparisons.

```java
// (a)
static String render(Object o) {
    if (o instanceof String) {
        String s = (String) o;
        return s.isEmpty() ? "(empty)" : s;
    } else if (o instanceof Integer) {
        Integer i = (Integer) o;
        return "#" + i;
    } else if (o == null) {
        return "(null)";
    }
    return o.toString();
}

// (b) - use the negate-and-return idiom
static long lineTotal(Object o) {
    if (o instanceof OrderLine) {
        OrderLine line = (OrderLine) o;
        return line.quantity() * line.unitPriceMinorUnits();
    }
    throw new IllegalArgumentException("not an order line");
}

// (c) - use a nested record pattern
record Money(long minorUnits, String currency) { }
record Payment(String reference, Money amount) { }

static String describe(Object o) {
    if (o instanceof Payment) {
        Payment p = (Payment) o;
        if (p.amount() != null && p.amount().currency().equals("GBP")) {
            return "GBP payment of " + p.amount().minorUnits();
        }
    }
    return "other";
}
```

Then answer, for (c): **how many null checks did the record pattern remove, and why?**

### 2 — Medium: the audit (combines Topics 01–28)

This method contains **seven** distinct defects drawn from Topics 01, 13, 26, 27, 28 and
this one. Find them all, state the observable production symptom for each, and rewrite
it.

```java
public String handle(Object event) {
    switch (event) {
        case Object any -> {
            metrics.increment("events.total");
        }
        case OrderPaid p when p.amount() != null -> {
            Long total = totals.get(p.orderId());
            totals.put(p.orderId(), total == null ? p.amount() : total + p.amount());
            return "paid";
        }
        case OrderPlaced pl -> {
            for (Long id : processedIds) {
                if (id == pl.orderId()) return "duplicate";
            }
            processedIds.add(pl.orderId());
            return "placed";
        }
        default -> {
            log.warn("unhandled: " + event);
            return "unknown";
        }
    }
}
```

Hints without spoilers: one defect makes it fail to compile immediately (which one, and
what is the exact error?); one is the `default` problem; one is a Topic 01 `==`
comparison; one is a Topic 01 boxing/allocation issue; one is a `switch` structure
problem (statement vs expression); one is a guard that should not be a guard; and one is
that `Object` is the wrong selector type entirely.

For each, write one sentence starting "The on-call engineer sees...".

### 3 — Hard: production simulation on `orderflow`

Build the order read-model projector, then attack your own exhaustiveness guarantee and
prove it holds.

**Part A — the model.** Reuse or rebuild the `OrderEvent` sealed hierarchy from
Topic 28 with five variants: `OrderPlaced`, `OrderPaid`, `OrderShipped`,
`OrderCancelled`, `OrderRefunded`. Every variant is a record carrying `OrderId`,
`Instant occurredAt`, `long sequence`, plus its own data.

**Part B — three exhaustive consumers, no `default` anywhere:**
1. `ReadModelProjector.project(OrderEvent)` — updates an in-memory read model.
2. `OrderStatusMachine.next(OrderStatus, OrderEvent)` — returns the next status, and
   must **reject invalid transitions** (a `Paid` event on an already-`CANCELLED` order).
   Use guards for the transition rules.
3. `NotificationRouter.route(OrderEvent)` — decides which customer notification to
   send. At least two variants must use **record deconstruction with a guard**, and at
   least one must use a **nested** record pattern two levels deep.

**Part C — attack the guarantee.** Run these five experiments and record, for each,
whether you got a compile error, a runtime exception, or silence:

| Experiment | Predict, then verify |
|---|---|
| Add a sixth variant `OrderPartiallyRefunded` | ? |
| Add a `default` to `ReadModelProjector`, then add the sixth variant | ? |
| Make every case in `NotificationRouter` guarded | ? |
| Put `case OrderEvent e ->` first in `OrderStatusMachine` | ? |
| Pass `null` to `project(...)` | ? |

Write down your prediction **before** compiling each one. Then paste the actual result.
Where you were wrong, write one sentence about what your mental model got wrong.

**Part D — the boundary.** Your projector consumes from Kafka. A producer running a
newer version emits `OrderPartiallyRefunded` and your consumer does not know the type.
Design the handling. Requirements:
- The domain switch must stay exhaustive with no `default`.
- The unknown type must not crash the consumer or silently disappear.
- You must be able to reprocess those messages after deploying the handler.
Write the code for the boundary, and one paragraph on where exactly the compile-time
guarantee ends and what replaces it. (Topics 28 and 116 are relevant.)

**Part E — the readability argument.** Take your deepest nested record pattern from
Part B and rewrite it using accessors instead. Which is clearer? Now reorder two
same-typed components of the innermost record and re-run your tests. Report which
version caught it. This is Trap 5, on your own code.

---

## Interview questions

### Q1 — "Why does a switch over a sealed type not need a `default`?"

**Mid-level answer:** "Because the compiler knows all the possible subtypes from the
`permits` clause, so if you handle them all the switch is complete."

**Senior answer:** "That is the mechanism. The consequence is the part that matters: not
only does it *not need* a `default` — it must **not have** one. Exhaustiveness checking
is the compiler proving the listed cases cover the selector's whole domain. `default`
covers everything by definition, so the proof obligation is discharged trivially and the
compiler stops looking for gaps. That means adding a `default` silently converts a
compile-time guarantee into a runtime log line: someone adds a variant, the build stays
green, and the new case falls into `default` in every consumer. In an event-driven
service that shows up as a read model that silently stops updating, with no exception
anywhere and a business metric as your only signal. If I genuinely need to handle
unknown input — a newer producer emitting a type this consumer does not know — I handle
that at the **deserialization boundary**, where unknown input actually arrives: catch
the unknown type id, dead-letter the message, emit a metric. The domain switch keeps its
proof. I also put a comment above every such switch saying 'no default, deliberately',
because the thing it prevents is invisible in a diff."

**What separates them:** "must not have one" rather than "does not need one", the exact
production symptom, and the boundary-versus-domain distinction as the alternative.

**Interviewer's follow-up:** "How would you enforce that across a team?" Good answers:
a review checklist item, a grep in CI as a starting point, and an ErrorProne-style
static check for a permanent guard. Weak answer: "we'd remember".

---

### Q2 — "What's the difference between pattern matching and TypeScript's narrowing?"

**Mid-level answer:** "They both let you avoid casts after checking a type."

**Senior answer:** "Same outcome, different mechanics, and one real gap. TypeScript
narrows the **existing** variable through control-flow analysis on a discriminant
property value — the type is erased, so it needs a runtime property like `kind` to test.
Java introduces a **new** pattern variable and narrows by the actual runtime type,
because JVM objects carry their class — so there is no discriminant field to declare,
set, or get wrong. Flow scoping is the same in both: `if (!(x instanceof T t)) return;`
leaves `t` in scope afterwards, exactly like TypeScript's early-return narrowing. Where
they genuinely differ is destructuring: TypeScript's is **by property name**, Java's
record patterns are **by position**, and you must list every component. That means
reordering two same-typed record components silently rebinds every deconstruction
pattern that mentions them, with no compile error. So I keep deconstruction shallow —
one level — and use accessors below that, because accessors are name-based and cannot
silently rebind. And Java has one thing TypeScript does not: putting a general pattern
before a specific one is a **compile error**, where an `if`/`else if` chain in either
language silently produces dead code."

**What separates them:** knowing positional-versus-named destructuring is the real
hazard, having a stated rule about deconstruction depth, and knowing dominance checking
exists at all.

**Interviewer's follow-up:** "Show me the flow-scoping case that surprises people." The
`if (!(o instanceof T t)) return; use(t);` form. Then the one that does *not* work:
`if (o instanceof T t || other) { use(t); }` fails to compile.

---

### Q3 — "Why doesn't this switch compile? Every variant is covered."

```java
return switch (result) {
    case Approved a when a.amount() > 0 -> "approved";
    case Declined d when d.reason() != null -> "declined";
    case Failed f when f.retryable() -> "retry";
};
```

**Mid-level answer:** "I'd need to see the error — maybe the sealed interface has
another variant?"

**Senior answer:** "It does not compile because **guarded patterns never contribute to
exhaustiveness**. The compiler would have to evaluate `a.amount() > 0` at compile time
to know whether that case ever fires, which it cannot do in general, so it treats every
guarded case as 'might not match' and concludes there are gaps. The error is 'the switch
expression does not cover all possible input values', which is confusing precisely
because all three variants are named. The fix is one unguarded case per variant, placed
after that variant's guarded cases — so three variants with guards becomes six cases,
which is more than people expect. Ordering matters too and the compiler enforces it: an
unguarded `case Approved a` **dominates** a guarded one, so the reverse order is a
compile error about a dominated case label. And I would resist the temptation to fix it
with a `default`, which makes the error go away and takes the guarantee with it."

**What separates them:** knowing the rule without needing to see the error, the "six
cases for three variants" shape, and connecting it back to the `default` trap rather
than treating them as unrelated.

**Interviewer's follow-up:** "Is there a case where a guard *does* count?" No — not even
`when true`. The compiler does not attempt to prove guards, deliberately, because a
compiler that sometimes proves them would be far harder to reason about.

---

### Q4 — "What happens when you switch on null?"

**Mid-level answer:** "It throws a `NullPointerException`, same as switching on a null
String always did."

**Senior answer:** "It throws NPE, **and `default` does not catch it** — which is the
part that confuses people, because they look at a switch with a `default` and cannot see
how it NPE'd. That behaviour is deliberate: it preserves what every pre-Java-21 switch
did with null, so a JDK upgrade cannot silently change the behaviour of existing code.
To handle null you write an explicit `case null ->`, or `case null, default ->` when
null and unknown mean the same thing. Note also that a *pattern* never matches null:
`null instanceof Payment p` is false rather than throwing, and a record pattern fails at
every nesting level if any component is null — which is actually a feature, because
`case Approved(var ref, Money(long minor, var cur))` gives you deep null-safety with no
null checks written. The design question I would raise, though, is why the value is
nullable at all. If it is a database column, a `NOT NULL` constraint is a stronger
control than a `case null`; if it is my own domain type, records with validating compact
constructors mean it cannot be null. I want null handled at the boundary, not inside
domain logic."

**What separates them:** knowing `default` does not catch null and *why* (backward
compatibility), the pattern-versus-selector distinction, and pushing back to "why is it
nullable" rather than accepting the premise.

**Interviewer's follow-up:** "Does a switch over a fully-handled sealed type still NPE
on null?" Yes. Exhaustiveness is about the variants, not about null.

---

### Q5 — "When is pattern matching the wrong tool?"

**Mid-level answer:** "When you can use polymorphism instead — put the method on the
class."

**Senior answer:** "That is the right instinct and the honest version has a condition on
it. The two are duals: a virtual method makes **adding a variant** cheap and **adding an
operation** expensive — you touch every class. An exhaustive switch makes **adding an
operation** cheap and **adding a variant** expensive — you touch every switch. So the
question is which axis moves in your domain. `orderflow`'s payment outcomes change
rarely and the number of things we do with them grows constantly — reporting, metrics,
notifications, the state machine, reconciliation — so switches are right. A rendering
library where new shape types arrive constantly and the operations are fixed should use
virtual methods. The other, more important criterion: a virtual method belongs on the
type only if the behaviour is **intrinsic** to it. `PaymentResult` should not know about
HTTP status codes, Micrometer counters, or which email template to send — that would
drag the presentation and infrastructure layers into a domain type. Those operations
belong in switches in the layers that own them, which is exactly what sealed types plus
exhaustiveness are for. And pattern matching is also wrong for a hierarchy you do not
control, because there is no exhaustiveness to be had — that is just `instanceof` with
nicer syntax, and a visitor or a strategy map is usually clearer."

**What separates them:** naming the expression-problem trade explicitly, using
"intrinsic to the type" as the deciding criterion rather than a blanket rule, and
knowing pattern matching without sealing buys almost nothing.

**Interviewer's follow-up:** "Can you have both?" Yes — a common accessor on the sealed
interface for what *is* intrinsic (`orderId()`, `occurredAt()`) and switches for what is
not. That hybrid is what the production example in this document does.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `default` and exhaustiveness checking are mutually exclusive. Could the language have
   made `default` *and* exhaustiveness coexist — say, a `default` that only fires for
   variants added after compilation? Design it. Then say why it probably was not done.

2. Guards never contribute to exhaustiveness, not even `when true`. Argue that this is
   the right design. Then construct the strongest case that the compiler should special-
   case constant guards, and say why you would still reject it.

3. Record deconstruction is positional. TypeScript's is by name. What would Java need to
   change to support name-based deconstruction, and what would it cost? Think about what
   record component names are and are not, at the bytecode level (Topic 27).

4. A pattern switch compiles to `invokedynamic` + `SwitchBootstraps.typeSwitch` +
   a `tableswitch`, rather than a chain of `instanceof` tests. What does that buy, and
   what would you have to measure before claiming any performance benefit? (Topic 77.)

5. `default` does not catch null, in order to preserve pre-21 behaviour. Was backward
   compatibility worth that inconsistency? Argue both sides. What would you have done?

6. Pattern matching plus sealed types is the "expression problem" trade: easy to add
   operations, hard to add variants. Name a part of `orderflow` where the opposite trade
   is correct, and say how you would recognise that in a design review before writing
   any code.

7. You inherit a codebase with 200 switches over a sealed type, and every one has a
   `default -> log.warn(...)`. You cannot fix them all this sprint. What do you do
   first, and how do you decide the order?

---

## Quick reference card

### The three forms

```java
// 1. instanceof pattern
if (o instanceof OrderPaid paid) { use(paid.amount()); }
if (!(o instanceof OrderPaid paid)) { return; }  use(paid);   // flow scoping

// 2. switch with type patterns
String s = switch (o) {
    case Integer i -> "int " + i;
    case String str when str.isEmpty() -> "empty string";
    case String str -> "string " + str;
    case null -> "null";
    default -> "other";
};

// 3. record patterns (deconstruction)
switch (result) {
    case Approved(String ref, Money(long minor, String cur)) -> ...;   // nested
    case Declined(var reason, var msg) -> ...;                          // var
}
```

### Exhaustiveness rules

| Situation | Exhaustive? |
|---|---|
| Sealed type, all variants covered, no `default` | **yes** — this is the goal |
| Sealed type, all variants covered, plus a `default` | "yes" but **unchecked** — the guarantee is gone |
| Sealed type, one variant missing, no `default` | **compile error** — this is the feature |
| Every case guarded with `when` | **no** — guards never count |
| A total pattern like `case Object o ->` on an `Object` selector | yes |
| Non-sealed selector type, no `default` | compile error — you need a `default` or a total pattern |
| Enum selector, all constants, no `default` | yes |

### Ordering rules

| Rule | Consequence of breaking it |
|---|---|
| Subtype cases before supertype cases | `error: this case label is dominated by a preceding case label` |
| Guarded cases before the unguarded case for the same type | same error |
| Guarded cases cannot dominate anything | (so a guarded case after an unguarded one of the same type is the error) |

### Null

| Switch has | null selector does |
|---|---|
| nothing about null | **NPE** — even with a `default` |
| `case null ->` | matches that case |
| `case null, default ->` | matches that case |
| any pattern (`case Payment p`) | never matches null |
| a record pattern with a null component | does not match |

### `[JAVA 25]` and fallbacks

| Feature | Status | 21-compatible fallback |
|---|---|---|
| `instanceof` type pattern | final since 16 | — |
| pattern `switch` | final since 21 | — |
| record patterns | final since 21 | — |
| `when` guards | final since 21 | — |
| `case null` | final since 21 | — |
| **primitive types in patterns (JEP 507)** | **preview in 25** — verify with the Proof 5 command | use the boxed wrapper: `case Integer i ->` |

### Gotchas checklist

- [ ] **Never** put a `default` in a switch over a sealed type. Add a comment saying so.
- [ ] Handle unknown input at the deserialization boundary, not with `default`.
- [ ] Guards never count toward exhaustiveness. Every variant needs one unguarded case.
- [ ] Guarded cases go **before** the unguarded case for the same type.
- [ ] Subtype cases go before supertype cases, or you get a dominance error.
- [ ] `default` does not catch null. Write `case null` if null is possible.
- [ ] Record patterns are **positional**. Reordering same-typed components silently
      rebinds. Keep deconstruction one level deep; use accessors below that.
- [ ] `yield`, not `return`, to produce a value from a `{ }` case body in a switch
      expression.
- [ ] Primitive type patterns are preview on Java 25. Use boxed types in shipped code.

---

## When would I use this at work?

**1. Every consumer of a sealed domain hierarchy.**
Event projectors, state machines, notification routers, metrics recorders, error-to-HTTP
mappers. In a codebase that uses sealed types, exhaustive pattern switches are the
default way you consume them, and the payoff is that adding a variant produces a
compile error at each consumer rather than a bug at each consumer. This is the everyday
use and it is most of the value.

**2. Mapping domain errors to HTTP responses in one place.**
Topic 46's `@ControllerAdvice` over a sealed `DomainError` hierarchy: one exhaustive
switch turns each error variant into a `ProblemDetail` with the right status. Add an
error type and the compiler makes you decide its status code. Without sealing plus
pattern matching this is a chain of `instanceof` blocks that silently falls through to a
500 for anything new.

**3. Parsing and validating semi-structured input.**
JSON trees, protocol frames, configuration values, webhook payloads — anything where you
receive an `Object` or a small tagged union and must decide what it is. Nested record
patterns turn "check it's an object, check it has a `data` field, check that's an array,
check the first element is an object" into a single pattern that either matches
completely or does not match, with no intermediate null checks.

---

## Connected topics

**Prerequisites:**
- **27 — Records**: record patterns deconstruct records. The component list being part
  of the public API is exactly what makes deconstruction possible.
- **28 — Sealed types**: the source of exhaustiveness. Pattern matching without sealing
  is `instanceof` with better syntax; sealing without pattern matching is only half the
  feature. These two topics are one idea split across two documents.
- **02 — Nominal vs structural typing**: why Java narrows by type and TypeScript needs a
  discriminant property.
- **16 — Enums**: enum switches have their own exhaustiveness rules, and the same
  `default`-destroys-the-check hazard applies.
- **01 — Autoboxing**: why the Java 21 fallback for primitive patterns costs an
  allocation, and why `case Integer i` unboxes when you assign it to an `int`.
- **21 — Lambdas**: same `invokedynamic` machinery underneath.

**This unlocks:**
- **46 — Error handling and `ProblemDetail`**: exhaustive mapping from a sealed error
  hierarchy to HTTP status codes.
- **47 — Spring Data JPA and projections**: switching over sealed query-result types.
- **117 — Sagas and compensations**: the saga coordinator is an exhaustive switch over
  step outcomes, and compensation logic is a switch over what has already succeeded.
  Adding a step outcome breaks the coordinator's build, which is exactly what you want
  in code that moves money.
- **76 — Reading bytecode**: `SwitchBootstraps.typeSwitch` and what the runtime actually
  does with your cases.
- **103–108 — Reactive**: pattern-matching over sealed signal types is common in
  handling `Flux` events and error channels.

---

*Java baseline 21. Type patterns in `instanceof` became final in **Java 16**; pattern
`switch`, record patterns, `when` guards and `case null` all became final in **Java 21**
— so none of the main content here needs `--enable-preview`. The one exception is
**primitive types in patterns (JEP 507)**, which is a **preview** feature on JDK 25;
Proof 5 gives you the exact command to confirm that on your JDK, and the boxed-wrapper
fallback is what you ship. If the pattern-switch syntax in this document does not
compile for you at all, the cause is a `--release` or `--source` below 21, not a missing
preview flag — check `javac --version` and your build's compiler configuration.*
