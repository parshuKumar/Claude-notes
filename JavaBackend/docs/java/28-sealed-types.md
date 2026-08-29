# 28 — Sealed Types vs TypeScript Discriminated Unions

## Phase: 2 — Modern Java
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (spine starts at Topic 35)

---

## ELI5 anchor

Imagine a company that issues exactly **three** kinds of badge: Staff, Contractor,
Visitor. The rule is printed on the front door: *"the only badges that exist are these
three."*

Because that list is posted, the security guard's checklist can be complete. Staff go
to floor 4. Contractors go to floor 2. Visitors get an escort. There is no "anything
else" row on the checklist, because there is no anything else.

Now somebody invents a fourth badge: Auditor. They must go and update the sign on the
front door — the language forces them to. And the **instant they do**, every checklist
in the building that does not mention Auditor stops working. Not quietly. The building
refuses to open.

That is a **sealed type**. You publish the complete list of variants at the parent, and
the compiler uses that list to prove your handling code is complete. Add a variant and
every incomplete handler becomes a compile error.

That "the build breaks everywhere until you handle the new case" property is the entire
value. It is also exactly what your TypeScript discriminated unions do.

---

## The bridge from what you know

**This is it. This is the single closest analogue in the entire curriculum.**

Most of this course is me telling you "transfer this instinct, then unlearn one thing".
Here, the instinct transfers essentially whole. Read both columns and notice how little
you have to unlearn.

### The TypeScript you write today

```ts
// The variants
type Approved = { kind: "approved"; reference: string; amountMinor: number };
type Declined = { kind: "declined"; reasonCode: string };
type Failed   = { kind: "failed";   error: string; retryable: boolean };

// The union — the "sign on the front door"
type PaymentResult = Approved | Declined | Failed;

// The exhaustive handler
function describe(result: PaymentResult): string {
  switch (result.kind) {
    case "approved": return `paid ${result.amountMinor} ref ${result.reference}`;
    case "declined": return `declined: ${result.reasonCode}`;
    case "failed":   return `failed: ${result.error}`;
    default: {
      const never: never = result;      // the exhaustiveness trick
      return never;
    }
  }
}
```

Add a fourth member to `PaymentResult` and the `const never: never = result` line stops
compiling, because `result` is no longer `never` in the default branch. That is the
guarantee you rely on.

### The Java that does the same job

```java
// The sign on the front door
public sealed interface PaymentResult
        permits Approved, Declined, Failed { }

// The variants
public record Approved(String reference, long amountMinor) implements PaymentResult { }
public record Declined(String reasonCode)                  implements PaymentResult { }
public record Failed(String error, boolean retryable)      implements PaymentResult { }

// The exhaustive handler
static String describe(PaymentResult result) {
    return switch (result) {
        case Approved a -> "paid " + a.amountMinor() + " ref " + a.reference();
        case Declined d -> "declined: " + d.reasonCode();
        case Failed f   -> "failed: " + f.error();
        // NO default. The compiler already knows the list is complete.
    };
}
```

Add `Refunded` to the `permits` clause and **`describe` stops compiling**, with:

```
error: the switch expression does not cover all possible input values
```

Same guarantee. Same failure mode. Same day-one benefit.

### Side by side, line for line

| Concern | TypeScript | Java |
|---|---|---|
| Declare the variants | `type Approved = { kind: "approved"; ... }` | `record Approved(...) implements PaymentResult { }` |
| Declare the closed set | `type PaymentResult = Approved \| Declined \| Failed` | `sealed interface PaymentResult permits Approved, Declined, Failed` |
| Where the closed set is declared | at the **union**, separately from the members | at the **parent**, and the members must opt in with `implements` |
| Discriminate on | a **property value** (`result.kind === "approved"`) | the **type itself** (`case Approved a ->`) |
| Narrowing mechanism | structural, control-flow analysis on the discriminant | nominal, `instanceof`-style type test |
| Exhaustiveness | `never` trick, or `switch` with all cases returning under `strictNullChecks` + `noImplicitReturns` | built in — a `switch` over a sealed type needs no `default` |
| Failure when you add a variant | compile error at every `never` assertion | compile error at every `switch` |
| Carrying different data per variant | yes, that is the point | yes, that is the point |

### The four differences that are real — be precise about these

**1. Java narrows by type; TypeScript narrows by a discriminant value.**
TypeScript needs a literal property (`kind`) because its types are erased structural
shapes with no runtime identity. Java objects carry their class at runtime, so the
class *is* the discriminant. You write no `kind` field, and you cannot get it wrong or
forget to set it. That is a small win for Java.

**2. Sealing is declared at the parent, not at the union site.**
In TypeScript, `PaymentResult` is one union among many you can freely define:
```ts
type TerminalResult = Approved | Declined;      // a different union, same members
type RetryableResult = Failed;
```
That costs nothing. In Java, `Approved` declares `implements PaymentResult` and the
parent declares `permits Approved`. It is a **two-sided commitment** baked into both
class files. You cannot form an ad-hoc union of existing types after the fact.

You *can* have a type implement more than one sealed interface, which recovers some of
this:
```java
public sealed interface PaymentResult permits Approved, Declined, Failed { }
public sealed interface Terminal      permits Approved, Declined { }

public record Approved(String reference, long amountMinor)
        implements PaymentResult, Terminal { }
```
But both hierarchies must be planned. You cannot union types you do not control.

**3. Permitted subtypes must live in the same module, or — if unnamed — the same
package.**
This is a hard rule with no TypeScript counterpart. If your code is not in a JPMS module
(Topic 20), which describes most applications, then `Approved` must be in the *same
package* as `PaymentResult`. In a named module they may be in different packages of the
same module. **They can never be in a different module.** Sealing is a boundary you
control; you cannot seal across an artifact you do not own.

**4. Java's variants are types; TypeScript's union members can be anything.**
```ts
type Status = "pending" | "paid" | 404 | null;   // literals, primitives, null
```
Java has no equivalent. A sealed hierarchy's members are reference types, full stop.
Literal-union types have **no analogue** — the nearest thing is an enum, which is a
different tool with different powers.

### Verdict

**HONEST ANALOGUE.** `sealed interface` + records + pattern-matching `switch` is
Java's algebraic data type, and it delivers the same compile-error-on-new-variant
guarantee your TypeScript discriminated unions do. Transfer the instinct directly.
Unlearn only: the discriminant is the type not a property, sealing is a two-sided
declaration at the parent, and everything must live in one module.

---

## What is this?

A **sealed** type is a class or interface that declares, in its own source, the
complete list of types allowed to extend or implement it.

```java
public sealed interface PaymentResult permits Approved, Declined, Failed { }
```

Three things follow from that declaration:

1. **No other type can implement it.** Not in your code, not in a library, not by
   reflection or bytecode manipulation — the JVM enforces it at class load, not just
   `javac`. Attempting it gets you an `IncompatibleClassChangeError` at runtime or a
   compile error at build time.
2. **The compiler knows the complete list.** That is what makes exhaustiveness checking
   possible. A `switch` over `PaymentResult` that handles all three needs no `default`.
3. **Every permitted subtype must state what *it* allows.** Each one must be `final`,
   `sealed`, or `non-sealed`. There is no fourth option and no default — the compiler
   forces the decision.

Sealed classes became final in **Java 17**. On a Java 21 baseline this is ordinary,
fully-supported language. No preview flag.

### The three closure options for a permitted subtype

```java
public sealed interface PaymentResult permits Approved, Declined, Failed { }

public record Approved(String ref, long amountMinor) implements PaymentResult { }
//     ^^^^^^ records are implicitly FINAL — the hierarchy stops here

public final class Declined implements PaymentResult { }
//     ^^^^^ explicitly final — stops here

public sealed class Failed implements PaymentResult
        permits TimeoutFailure, GatewayFailure { }
//     ^^^^^^ the hierarchy continues, but still closed

public non-sealed class Declined implements PaymentResult { }
//     ^^^^^^^^^^ ESCAPE HATCH: anyone may now extend Declined.
//                Exhaustiveness over PaymentResult still works, but you can no longer
//                reason about what a Declined actually is.
```

`non-sealed` is the only hyphenated keyword in Java. It exists so a sealed hierarchy can
have one deliberately open branch — for example, a framework's base exception type that
users are meant to subclass. Use it rarely and on purpose; it is a hole you are cutting
in your own guarantee.

### `permits` is optional when everything is in one file

```java
// PaymentResult.java — all in one file
public sealed interface PaymentResult { }          // no permits clause needed

record Approved(String ref, long amountMinor) implements PaymentResult { }
record Declined(String reasonCode)            implements PaymentResult { }
record Failed(String error, boolean retryable) implements PaymentResult { }
```

The compiler infers the permitted set from the file. This is genuinely the nicest form
for a small closed hierarchy, and it puts the whole "union" on one screen — which is
the closest you get to reading a TypeScript union declaration.

---

## Why does it matter?

**1. It converts a whole class of runtime bug into a compile error.**
Before sealed types, adding a new payment outcome meant grepping for `instanceof
PaymentResult` and hoping. Every place you missed produced either a silent wrong
default or a runtime `IllegalStateException` — usually in production, usually on the
new code path, usually at the worst time. Now the build stops.

**2. It makes an interface a *specification* rather than an *invitation*.**
A plain `public interface PaymentResult` says "anyone may implement this, and I promise
to keep working with whatever you build". That is a support obligation (Topic 03). A
sealed one says "this is the complete set of outcomes; the type *is* the enumeration".

**3. It gives you variant-specific data, which an enum cannot.**
An enum's constants are fixed instances. `DECLINED` cannot carry a `reasonCode` unless
every constant has that field, including `APPROVED`, where it is meaningless and null.
A sealed hierarchy over records gives each variant exactly the data it needs.

**4. It is the prerequisite for everything in Topic 29.**
Pattern matching without exhaustiveness is just `instanceof` with nicer syntax. Sealing
is what makes the switch a *proof*.

**5. It models domains that actually have closed sets.**
Payment outcomes, order lifecycle events, HTTP result shapes, JSON node kinds,
expression trees, saga step results. These are closed by definition in your business,
and until Java 17 you had no way to say so.

---

## Syntax breakdown

### Sealed interface (the common case)

```java
public sealed interface OrderEvent
        permits OrderPlaced, OrderPaid, OrderShipped, OrderCancelled {

    OrderId orderId();          // a common accessor every variant must provide
    Instant occurredAt();
}
```

| Bit of syntax | What it means |
|---|---|
| `sealed` | A modifier on the class/interface declaration. Says "the subtype list is closed". |
| `permits A, B, C` | The complete list. Optional if all subtypes are in the same source file. |
| Methods in the body | Ordinary interface methods. Every variant must supply them. This is how you get a *common* API alongside variant-specific data. |

And a record satisfies `OrderId orderId()` automatically if it has an `orderId`
component — the generated accessor has exactly the right name and signature. That is
why records and sealed interfaces fit together so cleanly.

```java
public record OrderPlaced(OrderId orderId, Instant occurredAt, List<OrderLine> lines)
        implements OrderEvent { }
//     the generated orderId() and occurredAt() accessors implement the interface
```

### Sealed abstract class (when variants share state)

```java
public sealed abstract class PaymentAttempt
        permits CardAttempt, WalletAttempt {

    private final Instant startedAt;                 // shared state

    protected PaymentAttempt(Instant startedAt) {
        this.startedAt = startedAt;
    }

    public Instant startedAt() { return startedAt; }
    public abstract Money amount();
}

public final class CardAttempt extends PaymentAttempt {
    private final String last4;
    private final Money amount;
    public CardAttempt(Instant startedAt, String last4, Money amount) {
        super(startedAt);
        this.last4 = last4;
        this.amount = amount;
    }
    @Override public Money amount() { return amount; }
}
```

**Prefer the interface form.** A sealed abstract class forces variants to be classes,
which means you lose records — and losing records means writing `equals`/`hashCode` by
hand (Topic 13) and losing record deconstruction patterns (Topic 29). Reach for the
abstract class only when the variants genuinely share mutable or expensive state that
must not be duplicated.

### Nested variants — the tidiest form

```java
public sealed interface PaymentResult {

    record Approved(String reference, Money amount)      implements PaymentResult { }
    record Declined(DeclineReason reason, String detail) implements PaymentResult { }
    record Failed(String message, boolean retryable)     implements PaymentResult { }
}
```

Nested types inside an interface are implicitly `public static`, so this compiles as
written. The `permits` clause is inferred. Call sites read as
`PaymentResult.Approved`, which some teams like (it is self-documenting) and some
dislike (it is verbose). Both are defensible; pick one and be consistent.

### What the compiler enforces

```java
// 1. Every permitted subtype must be final, sealed, or non-sealed
public class Approved implements PaymentResult { }
// error: sealed, non-sealed or final modifiers expected

// 2. You cannot implement a sealed type you are not permitted to
public record Reversed(String ref) implements PaymentResult { }
// error: class is not allowed to extend sealed class: PaymentResult (as it is not listed in its permits clause)

// 3. Permitted subtypes must be in the same package (unnamed module)
//    or the same named module
// error: class is not allowed to extend sealed class from another package

// 4. Every name in permits must actually implement the type
public sealed interface PaymentResult permits Approved, Money { }
// error: invalid permits clause: Money does not implement PaymentResult
```

Four separate compile errors, all of them at the declaration rather than at a use site.
That is the good kind of error — it fires next to the mistake.

---

## Example 1 — minimal

```java
public class SealedBasics {

    sealed interface Shape permits Circle, Square, Rectangle { }

    record Circle(double radius)               implements Shape { }
    record Square(double side)                 implements Shape { }
    record Rectangle(double width, double h)   implements Shape { }

    static double area(Shape shape) {
        return switch (shape) {
            case Circle c    -> Math.PI * c.radius() * c.radius();
            case Square s    -> s.side() * s.side();
            case Rectangle r -> r.width() * r.h();
            // no default — the compiler knows there are exactly three
        };
    }

    public static void main(String[] args) {
        System.out.printf("circle    %.2f%n", area(new Circle(2)));
        System.out.printf("square    %.2f%n", area(new Square(3)));
        System.out.printf("rectangle %.2f%n", area(new Rectangle(2, 5)));
    }
}
```

**Now do the experiment that teaches the topic.** Add a fourth shape:

```java
record Triangle(double base, double height) implements Shape { }
```
and add `Triangle` to the `permits` clause. Do **not** touch `area`.

Compile. You get:
```
error: the switch expression does not cover all possible input values
```

That error is the product. Everything else in this document is detail.

---

## Example 2 — production scenario

`orderflow` calls an external payment gateway. The gateway can approve, decline for a
business reason, or fail technically. Those three outcomes need completely different
handling, and the set will grow — payment providers add outcomes.

### The version everybody writes first

```java
public class PaymentResponse {
    private final boolean success;
    private final String reference;       // null unless success
    private final String reasonCode;      // null unless declined
    private final String errorMessage;    // null unless failed
    private final boolean retryable;      // meaningless unless failed
    private final Long amountMinor;       // null unless success

    // 20 lines of getters
}
```

and then, at every call site:

```java
PaymentResponse response = gateway.charge(request);

if (response.isSuccess()) {
    orderRepository.markPaid(orderId, response.getReference());
} else if (response.getReasonCode() != null) {
    orderRepository.markDeclined(orderId, response.getReasonCode());
} else {
    // ...failed? Or a success with a null reference? Nobody is sure.
    retryQueue.enqueue(orderId);
}
```

Six months later the gateway adds `PENDING_3DS` — the payment is neither approved nor
declined; the customer must complete an authentication step. Someone adds a
`pending3dsUrl` field to `PaymentResponse` and sets `success = false`.

**Every existing call site now routes 3DS payments into the retry queue.** No compile
error. No exception. Real customers, mid-checkout, quietly dropped into a retry loop
that will never succeed. You find out from a conversion-rate dashboard three days
later.

That is the bug sealed types exist to prevent.

### The sealed version

```java
package com.orderflow.payments;

import java.time.Instant;

/**
 * The complete set of outcomes a payment attempt can have.
 * Adding a variant here is a breaking change ON PURPOSE: every exhaustive
 * switch over PaymentResult will fail to compile until it is handled.
 */
public sealed interface PaymentResult
        permits PaymentResult.Approved,
                PaymentResult.Declined,
                PaymentResult.Failed,
                PaymentResult.PendingAuthentication {

    /** Every outcome knows which attempt it belongs to. */
    PaymentAttemptId attemptId();
    Instant occurredAt();

    record Approved(
            PaymentAttemptId attemptId,
            Instant occurredAt,
            String gatewayReference,
            Money capturedAmount) implements PaymentResult { }

    record Declined(
            PaymentAttemptId attemptId,
            Instant occurredAt,
            DeclineReason reason,
            String gatewayMessage) implements PaymentResult { }

    record Failed(
            PaymentAttemptId attemptId,
            Instant occurredAt,
            String technicalMessage,
            boolean retryable) implements PaymentResult { }

    record PendingAuthentication(
            PaymentAttemptId attemptId,
            Instant occurredAt,
            URI challengeUrl,
            Instant expiresAt) implements PaymentResult { }
}

public enum DeclineReason {
    INSUFFICIENT_FUNDS, CARD_EXPIRED, SUSPECTED_FRAUD, LIMIT_EXCEEDED, DO_NOT_HONOUR
}
```

Notice the layering: `DeclineReason` is an **enum**, because it is a closed set of
labels with no per-label data. `PaymentResult` is a **sealed interface**, because each
variant carries different data. Using the right one of those two is a real design skill
and interviewers probe it.

Now the call site:

```java
@Transactional
public OrderStatus applyPaymentResult(OrderId orderId, PaymentResult result) {
    return switch (result) {

        case PaymentResult.Approved a -> {
            orders.markPaid(orderId, a.gatewayReference(), a.capturedAmount());
            events.publish(new OrderPaid(orderId, a.occurredAt(), a.capturedAmount()));
            yield OrderStatus.PAID;
        }

        case PaymentResult.Declined d -> {
            orders.markDeclined(orderId, d.reason());
            inventory.releaseReservation(orderId);        // give the stock back
            events.publish(new OrderDeclined(orderId, d.occurredAt(), d.reason()));
            yield OrderStatus.DECLINED;
        }

        case PaymentResult.Failed f -> {
            if (f.retryable()) {
                retryScheduler.scheduleRetry(orderId, f.occurredAt());
                yield OrderStatus.PAYMENT_RETRY_SCHEDULED;
            }
            orders.markFailed(orderId, f.technicalMessage());
            inventory.releaseReservation(orderId);
            yield OrderStatus.PAYMENT_FAILED;
        }

        case PaymentResult.PendingAuthentication p -> {
            orders.markAwaitingAuthentication(orderId, p.challengeUrl(), p.expiresAt());
            yield OrderStatus.AWAITING_3DS;
        }
    };
}
```

What this bought:

- **No `default`.** The compiler proved the list is complete.
- **No null checks.** `a.gatewayReference()` is non-null *by construction*, because
  `Approved` is the only variant that has one and its constructor requires it.
- **No boolean flag archaeology.** There is no `isSuccess()` to misinterpret.
- **The day someone adds `PartiallyCaptured`**, this method fails to compile, along
  with the reporting job, the webhook handler, and the reconciliation batch. All four
  get fixed in the same PR, by the person who understands the new variant.

That last point is the one to say out loud in an interview: sealing turns "who else
handles payment results?" from an archaeology exercise into a compiler output.

### The same shape for events

```java
public sealed interface OrderEvent
        permits OrderPlaced, OrderPaid, OrderShipped, OrderCancelled, OrderRefunded {

    OrderId orderId();
    Instant occurredAt();
    long sequence();
}

public record OrderPlaced(OrderId orderId, Instant occurredAt, long sequence,
                          UserId userId, List<OrderLine> lines) implements OrderEvent { }

public record OrderPaid(OrderId orderId, Instant occurredAt, long sequence,
                        Money amount, String gatewayReference) implements OrderEvent { }

public record OrderShipped(OrderId orderId, Instant occurredAt, long sequence,
                           String carrier, String trackingNumber) implements OrderEvent { }

public record OrderCancelled(OrderId orderId, Instant occurredAt, long sequence,
                             String reason, UserId cancelledBy) implements OrderEvent { }

public record OrderRefunded(OrderId orderId, Instant occurredAt, long sequence,
                            Money amount, String reason) implements OrderEvent { }
```

Every event projector, every read-model updater, and every saga step handler
(Topic 117) switches exhaustively over this. Adding an event type breaks all of them at
compile time, which is precisely the behaviour you want from an event-sourced system.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a permitted subtype in a different package

**Wrong:**
```java
// com/orderflow/payments/PaymentResult.java
package com.orderflow.payments;
public sealed interface PaymentResult permits Approved, Declined { }

// com/orderflow/payments/card/Approved.java
package com.orderflow.payments.card;              // <-- different package
public record Approved(String ref) implements PaymentResult { }
```

**Exact symptom:** a compile error, and the wording matters because it is not obvious:
```
error: class is not allowed to extend sealed class from another package
```
or, depending on which file the compiler reaches first:
```
error: invalid permits clause
  (Approved is in an unnamed module; cannot be permitted from a different package)
```

If your build is multi-module, you may instead see the sealed type resolve to a
different class entirely and get a confusing "cannot find symbol".

**Root cause:** sealing must be *verifiable*. The JVM checks the `PermittedSubclasses`
attribute at class load. For that check to be meaningful, the permitted types must be
somewhere the sealing type's authority extends: the same named module, or — if you are
not using JPMS (Topic 20), which is most applications — the same package, because a
package in the unnamed module is the only boundary available.

This is a **hard architectural constraint**, not a lint. You cannot seal a type in
`orderflow-api` and put a variant in `orderflow-payments`. Ever.

**Fix — pick one:**

```java
// 1. Same package. Simplest and correct for most domains.
package com.orderflow.payments;
public sealed interface PaymentResult permits Approved, Declined { }
public record Approved(String ref) implements PaymentResult { }

// 2. Nested inside the interface. Guarantees the constraint is satisfied.
public sealed interface PaymentResult {
    record Approved(String ref) implements PaymentResult { }
    record Declined(String reason) implements PaymentResult { }
}

// 3. A real JPMS module (Topic 20), if you genuinely need multiple packages.
//    module-info.java, and the variants in other packages of the SAME module.
```

Option 2 is what most teams land on, because it makes the constraint impossible to
violate accidentally and it puts the whole union in one file — which is also the
closest reading experience to a TypeScript union.

**Design consequence worth stating:** a sealed hierarchy is a **cohesion forcing
function**. If you find your variants wanting to live in different packages, that is
evidence they are not really one closed set. Listen to it.

---

### Trap 2 — reaching for a sealed hierarchy where an enum was correct

**Wrong:**
```java
public sealed interface OrderStatus
        permits Pending, Paid, Shipped, Cancelled { }

public record Pending()   implements OrderStatus { }
public record Paid()      implements OrderStatus { }
public record Shipped()   implements OrderStatus { }
public record Cancelled() implements OrderStatus { }
```

**Exact symptom:** not a crash — a slow accumulation of friction that shows up as:

- You cannot write `switch (status) { case PENDING -> ... }` with the constant names;
  you write type patterns for four types with no data.
- You cannot use `EnumSet`/`EnumMap` (Topic 16), so `Set<OrderStatus>` becomes a
  `HashSet` of records — hashing and allocation where a bit vector would do.
- JPA `@Enumerated(EnumType.STRING)` no longer applies; you write a converter.
- Jackson serializes `{"status":{}}` instead of `"PAID"` unless you configure it.
- `OrderStatus.values()` does not exist, so you cannot iterate the set.
- `new Paid()` allocates a fresh object every time, and `paid1 == paid2` is false
  (though `.equals` works, since records generate it).

**Root cause:** the variants carry **no data**. When variants differ only in identity,
you want fixed singleton instances with names — which is precisely what an enum is.

**Fix:**
```java
public enum OrderStatus {
    PENDING, PAID, SHIPPED, CANCELLED;
}
```

Enums also give you exhaustiveness in a `switch` with no `default`, so you lose nothing
on that front.

**The decision rule, stated crisply:**

| Situation | Reach for |
|---|---|
| Closed set of **labels**, no per-variant data | `enum` |
| Closed set of **shapes**, each carrying different data | `sealed interface` + records |
| Closed set of labels where each label has the *same* fields | `enum` with constructor fields |
| Open set — third parties must add variants | plain `interface` (accept the obligation) |
| Closed set today, but the *library consumer* must extend one branch | `sealed` with one `non-sealed` branch |

**The inverse mistake is more common and worse:** using an enum where variants need
different data, which produces the "every constant has every field and most are null"
class from the production example above. If you find yourself writing
`if (status == DECLINED) { use(status.getReasonCode()); }`, you needed a sealed
hierarchy.

---

### Trap 3 — a sealed hierarchy crossing a serialization boundary without type info

**Wrong:**
```java
public sealed interface PaymentResult permits Approved, Declined, Failed { }

// Publishing it to Kafka / returning it from an API
kafkaTemplate.send("payment-results", objectMapper.writeValueAsString(result));
```

**Exact symptom, on the way out:** the JSON contains the variant's fields but nothing
saying *which* variant it is:
```json
{"attemptId":"pa-91","occurredAt":"2026-08-29T10:00:00Z","gatewayReference":"ref-1","capturedAmount":{"minorUnits":1999,"currency":"GBP"}}
```

**Exact symptom, on the way in:**
```
com.fasterxml.jackson.databind.exc.InvalidDefinitionException:
  Cannot construct instance of `com.orderflow.payments.PaymentResult`
  (no Creators, like default constructor, exist):
  abstract types either need to be mapped to concrete types,
  have custom deserializer, or contain additional type information
```

Consumers cannot tell an `Approved` from a `Declined` except by guessing from which
fields are present — which is the exact fragility you removed on the Java side.

**Root cause — and this is the honest, important part of the whole document:**

**JSON has no nominal types.** Your compile-time exhaustiveness guarantee stops at the
serializer. On the wire you are back in TypeScript's world, where the only way to
discriminate is a **discriminant property**. Java's sealed types buy you safety
*inside one JVM*; they buy you nothing across a network boundary unless you put the
discriminant in the payload yourself.

**Fix — put the discriminant back explicitly:**
```java
@JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,
    include = JsonTypeInfo.As.PROPERTY,
    property = "kind")                       // <-- literally TypeScript's `kind`
@JsonSubTypes({
    @JsonSubTypes.Type(value = PaymentResult.Approved.class, name = "approved"),
    @JsonSubTypes.Type(value = PaymentResult.Declined.class, name = "declined"),
    @JsonSubTypes.Type(value = PaymentResult.Failed.class,   name = "failed"),
    @JsonSubTypes.Type(value = PaymentResult.PendingAuthentication.class, name = "pending_3ds")
})
public sealed interface PaymentResult permits ... { }
```

which produces:
```json
{"kind":"approved","attemptId":"pa-91",...}
```

**Read that JSON and then re-read the TypeScript at the top of this document.** They are
the same shape. That is not a coincidence — it is the same problem with the same
solution. Java lets you skip the discriminant *in memory* because the JVM keeps the
class; the moment you leave the JVM you need it back.

**Two follow-on hazards:**

1. The `name` strings are now a **wire contract**. Renaming the Java record is safe;
   renaming the `name = "approved"` string breaks every consumer. Treat those strings
   like column names.
2. `@JsonSubTypes` is a **second list of variants**, and the compiler does not check it
   against `permits`. Add a variant to `permits` and forget `@JsonSubTypes` and you get
   a runtime serialization failure, not a compile error. Write a test that asserts
   `PaymentResult.class.getPermittedSubclasses().length` equals the number of
   registered subtypes — one assertion, closes the gap permanently. The Hands-on proof
   below shows how.

> Honesty flag: Jackson's support for sealed types improved across 2.x, and Jackson 3
> may infer more without annotations. Whether *your* version needs `@JsonSubTypes`
> explicitly, I will not assert. Write the round-trip test in the Hands-on section; it
> settles it in thirty seconds for your exact classpath.

---

### Trap 4 — `non-sealed` used to make a compile error go away

**Wrong:** a colleague needs to extend `Declined` from a test package. The compiler
complains. They change:
```java
public final class Declined implements PaymentResult { }
```
to
```java
public non-sealed class Declined implements PaymentResult { }
```

**Exact symptom:** nothing, immediately. That is the problem. Six months later:

- Someone writes `class DeclinedWithRetry extends Declined` in a different package.
- Every `switch` that had `case Declined d ->` now silently handles
  `DeclinedWithRetry` as a plain `Declined`, ignoring its extra semantics.
- No compile error, because `DeclinedWithRetry` **is** a `Declined` and the switch is
  still exhaustive over `PaymentResult`.
- Refunds are computed on the base variant's fields. Money is wrong.

**Root cause:** `non-sealed` does not weaken exhaustiveness over the *parent* — the
switch still compiles and still covers `PaymentResult`. What it destroys is your
ability to reason about what a `Declined` **is**. You went from "a `Declined` is exactly
this record" to "a `Declined` is at least this record and possibly more", and every
pattern match silently accepts the "possibly more" case.

This is subtle and it is why `non-sealed` deserves a comment explaining itself every
single time.

**Fix — for the actual need (testing), do not open the hierarchy:**
```java
// Tests construct the real variant. Records make this trivial.
var declined = new PaymentResult.Declined(
        new PaymentAttemptId("pa-1"), Instant.now(),
        DeclineReason.INSUFFICIENT_FUNDS, "insufficient funds");
```

**Fix — for the legitimate need (a framework base type users extend):**
```java
public sealed interface DomainError permits ValidationError, NotFoundError, PluginError { }

public record ValidationError(String field, String message) implements DomainError { }
public record NotFoundError(String resource, String id)     implements DomainError { }

/** Deliberately open: plugin authors define their own error types.
 *  Every switch must therefore treat PluginError as an opaque leaf. */
public non-sealed interface PluginError extends DomainError {
    String pluginId();
    String detail();
}
```

The comment is doing real work. It tells the next reader that `case PluginError p ->`
must not assume anything beyond the declared interface methods.

---

### Trap 5 — sealing a type that is part of your published API

**Wrong:** you publish `orderflow-client` as a library, and it exposes:
```java
public sealed interface Notification permits EmailNotification, SmsNotification { }
```

**Exact symptom:** a consumer team files an issue: *"we need a Slack notification and we
cannot implement your interface."* They cannot — not with a workaround, not with
reflection, not with a bytecode agent, because the JVM enforces `PermittedSubclasses`
at class load and will throw:
```
java.lang.IncompatibleClassChangeError: class com.other.SlackNotification
  cannot inherit from sealed class com.orderflow.client.Notification
```

You now have to choose between a breaking API change and telling them no.

**Root cause:** you made a design decision — "this set is closed" — and published it as
a contract. That is a legitimate decision. It just was not one anyone made
deliberately; someone typed `sealed` because it was the modern-looking option.

**Root cause, restated as the real rule:** **sealing is a statement about who owns the
set of variants.** Seal when *you* own the complete list and adding to it is your job.
Do not seal when your consumers are supposed to extend it — that is exactly what a
plain interface is for, and the "anyone can implement this" support obligation
(Topic 03) is the price of extensibility.

**Fix — decide deliberately, and consider the hybrid:**
```java
public sealed interface Notification permits EmailNotification, SmsNotification, CustomNotification { }

public record EmailNotification(String to, String subject, String body) implements Notification { }
public record SmsNotification(String number, String text)               implements Notification { }

/** The extension point. Consumers implement this, not Notification directly. */
public non-sealed interface CustomNotification extends Notification {
    String channelId();
    String render();
}
```

You keep exhaustiveness (three cases, no `default`), you keep the ability to add
first-party variants as a breaking change you control, and consumers get a documented
extension point with a defined contract. This hybrid is a genuinely good pattern and
worth having in your pocket for interviews.

---

## Hands-on proof

Every command below is one **you** run. I do not have a JVM and will not print output
and claim it is real. What follows is exactly what to look for and how to read each
possible result.

### Setup

```bash
mkdir -p ~/java-lab/28 && cd ~/java-lab/28
java --version      # expect 21 or 25. Sealed types are final since 17 — no preview flag.
```

### Proof 1 — the permitted list is in the class file

`PaymentResult.java`:
```java
public sealed interface PaymentResult permits Approved, Declined, Failed { }

record Approved(String reference, long amountMinor) implements PaymentResult { }
record Declined(String reasonCode)                  implements PaymentResult { }
record Failed(String error, boolean retryable)      implements PaymentResult { }
```

```bash
javac PaymentResult.java
javap -v PaymentResult.class | grep -A5 -i permitted
```

**What to look for:**

| What you see | What it means |
|---|---|
| A `PermittedSubclasses:` attribute listing `Approved`, `Declined`, `Failed` | The sealing is **in the bytecode**, not just a `javac` convention. This is what the JVM checks at class load, and it is why sealing cannot be defeated by compiling against a modified source. |
| The `flags:` line for the interface includes `ACC_INTERFACE`, and the class is not marked final | Interfaces are not final; the sealing comes from the attribute, not from a modifier bit. |
| No `PermittedSubclasses` at all | You compiled something that is not sealed. Check for a typo in `sealed`. |

Also check a variant:
```bash
javap -v Approved.class | head -20
```
Look for `final` in the flags. Records are implicitly final, which is why they satisfy
the "final, sealed, or non-sealed" requirement with no extra keyword.

### Proof 2 — try to break the seal, and read the error

`Rogue.java`:
```java
public record Reversed(String reference) implements PaymentResult { }
```

```bash
javac Rogue.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `error: class is not allowed to extend sealed class: PaymentResult (as it is not listed in its permits clause)` | Expected. The seal holds at compile time. |
| It compiles | You accidentally added `Reversed` to the `permits` clause, or `PaymentResult.class` on your classpath is stale. Recompile `PaymentResult.java` first. |

Now the harder version — prove the **runtime** check exists too:

```bash
# 1. Temporarily add Reversed to the permits clause and compile everything.
# 2. Then REMOVE Reversed from the permits clause and recompile ONLY PaymentResult.java:
javac PaymentResult.java
# 3. Now run something that loads Reversed against the new PaymentResult.class.
```

**What to look for:** `java.lang.IncompatibleClassChangeError` naming the class and the
sealed supertype. **What it means:** the JVM verifies `PermittedSubclasses` at class
load. Sealing is not a compiler-only convention like `private` erasure or generics
(Topic 06) — it has runtime teeth. That is a genuine difference from TypeScript, where
every guarantee evaporates at `tsc` output.

### Proof 3 — exhaustiveness, and watching it break

`Exhaustive.java`:
```java
public class Exhaustive {
    static String describe(PaymentResult r) {
        return switch (r) {
            case Approved a -> "approved " + a.reference();
            case Declined d -> "declined " + d.reasonCode();
            case Failed f   -> "failed "   + f.error();
        };
    }

    public static void main(String[] args) {
        System.out.println(describe(new Approved("ref-1", 1999)));
        System.out.println(describe(new Declined("INSUFFICIENT_FUNDS")));
        System.out.println(describe(new Failed("timeout", true)));
    }
}
```

```bash
javac PaymentResult.java Exhaustive.java && java Exhaustive
```

It runs. Now **add a variant**. Edit `PaymentResult.java`:

```java
public sealed interface PaymentResult
        permits Approved, Declined, Failed, PendingAuthentication { }

record PendingAuthentication(String challengeUrl) implements PaymentResult { }
```

```bash
javac PaymentResult.java Exhaustive.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `error: the switch expression does not cover all possible input values` pointing at `describe` | **The result the whole topic exists for.** You added a variant and the build broke at every incomplete handler, before any test ran. This is your TypeScript `never` check, built into the language. |
| It compiles fine | Either you left a `default` clause in the switch (see Topic 29 — this is the most important thing in that document), or you used a `switch` *statement* with no result, which has different exhaustiveness rules. Check both. |
| `error: patterns in switch statements are a preview feature` | Your `--source`/`--release` is set below 21. Check `javac --version` and any `-source` flag in your build. |

### Proof 4 — the permitted list, at runtime

`Reflect.java`:
```java
import java.util.Arrays;

public class Reflect {
    public static void main(String[] args) {
        Class<?> c = PaymentResult.class;
        System.out.println("isSealed           : " + c.isSealed());
        System.out.println("permittedSubclasses: " +
                Arrays.toString(c.getPermittedSubclasses()));
        for (Class<?> sub : c.getPermittedSubclasses()) {
            System.out.printf("  %-24s isRecord=%-5s isFinal=%s%n",
                    sub.getSimpleName(),
                    sub.isRecord(),
                    java.lang.reflect.Modifier.isFinal(sub.getModifiers()));
        }
    }
}
```

```bash
javac PaymentResult.java Reflect.java && java Reflect
```

**What to look for:**

| What you see | What it means |
|---|---|
| `isSealed : true` and a full list of subclasses | Confirmed. The list is available reflectively, which is how frameworks and serializers can discover your variants without annotations. |
| `isRecord=true isFinal=true` for each | Records satisfy the closure requirement automatically. |
| An empty array with `isSealed=true` | Impossible in practice — report it. |

### Proof 5 — close the `@JsonSubTypes` gap with a test

This is the assertion from Trap 3, and it is worth writing in every real project.

```java
@Test
void everyPermittedSubtypeIsRegisteredWithJackson() {
    var permitted = PaymentResult.class.getPermittedSubclasses();

    var registered = PaymentResult.class.getAnnotation(JsonSubTypes.class);
    assertThat(registered).as("@JsonSubTypes must be present").isNotNull();

    var registeredClasses = Arrays.stream(registered.value())
            .map(JsonSubTypes.Type::value)
            .collect(Collectors.toSet());

    assertThat(registeredClasses)
        .as("every permitted subtype must have a wire discriminant")
        .containsExactlyInAnyOrder(permitted);
}
```

**What to look for:**

| What you see | What it means |
|---|---|
| The test passes | Your compile-time variant list and your wire-format variant list agree. |
| The test fails naming a class present in `permits` but missing from `@JsonSubTypes` | Exactly the gap this test exists to catch. Someone added a variant and the compiler forced them to fix the switches but had nothing to say about serialization. |

Then write the round-trip test that settles the version question:
```java
@Test
void roundTripsThroughJson() throws Exception {
    PaymentResult original = new PaymentResult.Declined(
            new PaymentAttemptId("pa-1"), Instant.parse("2026-08-29T10:00:00Z"),
            DeclineReason.INSUFFICIENT_FUNDS, "insufficient funds");

    String json = mapper.writeValueAsString(original);
    System.out.println(json);                       // LOOK at this
    PaymentResult back = mapper.readValue(json, PaymentResult.class);

    assertThat(back).isEqualTo(original);           // free, thanks to record equals
}
```

**What to look for in the printed JSON:** a `"kind"` (or whatever you named it)
property. If it is absent, deserialization into the interface type cannot work, and
your consumers are guessing. That single printed line tells you whether your wire
contract is honest.

---

## Practice exercises

### 1 — Easy: translate the union

Here is a TypeScript discriminated union from a real checkout flow. Translate it to
Java: a sealed interface, records for the variants, and an exhaustive switch that
returns an HTTP status code. Put everything in one file.

```ts
type StockCheck =
  | { kind: "available"; sku: string; quantity: number }
  | { kind: "partially_available"; sku: string; available: number; requested: number }
  | { kind: "out_of_stock"; sku: string; restockEta: string | null }
  | { kind: "unknown_sku"; sku: string };

function statusFor(check: StockCheck): number {
  switch (check.kind) {
    case "available": return 200;
    case "partially_available": return 206;
    case "out_of_stock": return 409;
    case "unknown_sku": return 404;
  }
}
```

Then:
- Add a fifth variant, `Discontinued(String sku, Instant discontinuedAt)`. Compile
  **without** touching `statusFor`. Paste the exact error message.
- The TypeScript `restockEta: string | null` is nullable. Decide how to represent that
  in Java and justify it in one sentence — note that `Optional` as a record component is
  banned (Topic 26).
- Answer: why does the Java version have no `kind` field?

### 2 — Medium: replace the boolean-flag response (combines Topics 01–27)

Refactor this class into a sealed hierarchy, then fix every other defect in it. There
are **six** defects total, drawn from Topics 01, 13, 17, 26, 27 and this one.

```java
public class InventoryReservationResponse {
    public boolean reserved;
    public String sku;
    public Integer reservedQuantity;
    public Integer availableQuantity;
    public String failureReason;
    public Optional<Instant> retryAfter;
    public List<String> alternativeSkus;
    public Double reservationCostEstimate;

    public boolean isSameSku(InventoryReservationResponse other) {
        return this.sku == other.sku;
    }
}
```

Your deliverable:
- A sealed interface `ReservationResult` with variants that carry only the data that
  variant actually has. Name them from the domain, not from the fields.
- An exhaustive switch mapping each variant to an action in `orderflow`'s checkout
  flow.
- A short note for each of the six defects: the topic it comes from, and the observable
  production symptom.
- One sentence answering: **which fields in the original were meaningless in which
  states, and what did that cost the callers?**

### 3 — Hard: production simulation on `orderflow`

Model the complete payment lifecycle as a sealed hierarchy, then prove the guarantee
holds end to end — including across the wire, where it does not hold for free.

**Part A — the model.** Define `PaymentResult` with these variants: `Approved`,
`Declined`, `Failed`, `PendingAuthentication`, `PartiallyCaptured`. Every variant
carries `PaymentAttemptId attemptId` and `Instant occurredAt`, declared as methods on
the interface. Variant-specific data is up to you, but `Declined` must carry a
`DeclineReason` **enum** — and you must write one sentence explaining why that inner
type is an enum while the outer one is sealed.

**Part B — three exhaustive consumers.** Write three independent classes that each
switch exhaustively over `PaymentResult` with no `default`:
1. `OrderStateMachine` — maps a result to the next `OrderStatus`.
2. `PaymentMetricsRecorder` — increments a counter per outcome (Topic 118's shape).
3. `CustomerNotificationRouter` — decides which email/SMS to send, or none.

**Part C — the guarantee, demonstrated.** Add a sixth variant, `Chargeback`. Compile.
Record how many compile errors you get and where. Then answer: **how would you have
found those three sites without sealed types?** Be specific — name the grep you would
have run and what it would have missed.

**Part D — the wire boundary.** Serialize each variant to JSON and back.
- First without `@JsonTypeInfo`. Record the exact failure.
- Then with it. Paste the JSON for two variants.
- Write the test from Proof 5 that asserts `getPermittedSubclasses()` matches
  `@JsonSubTypes`. Add `Chargeback` to `permits` only, run the test, and confirm it
  fails.
- Finally, write two paragraphs: **where exactly does the compile-time guarantee stop,
  and what replaces it past that point?** This is the question that separates people who
  have used sealed types from people who have read about them.

**Part E — the consumer question.** A second team wants to add a `CryptoSettlement`
variant from their own module. Explain, in terms a non-Java engineer would understand,
why they cannot — and give them two options with the trade-off of each stated honestly.

---

## Interview questions

### Q1 — "What do sealed types give you that an ordinary interface doesn't?"

**Mid-level answer:** "They restrict which classes can implement the interface, so you
control the hierarchy."

**Senior answer:** "Restriction is the mechanism; **exhaustiveness** is the value.
Because the compiler knows the complete list of permitted subtypes, a `switch` over a
sealed type needs no `default`, and the moment somebody adds a variant every incomplete
switch becomes a compile error. That converts 'did we handle the new payment outcome
everywhere?' from an archaeology exercise into a build failure. It is exactly the
guarantee a TypeScript discriminated union plus an exhaustive switch gives, and it is
the closest true analogue between the two languages. Secondarily, it changes what an
interface *means* — a public non-sealed interface is an invitation with a permanent
support obligation, a sealed one is a specification. And it is enforced by the JVM at
class load through the `PermittedSubclasses` attribute, not just by `javac`, so you
cannot defeat it with a separate compilation."

**What separates them:** naming exhaustiveness as the payoff rather than restriction,
the compile-error-on-new-variant framing, and knowing the guarantee has runtime teeth.

**Interviewer's follow-up:** "Where does the guarantee stop?" At the process boundary.
JSON, protobuf, a database column — none of them carry Java's nominal types, so you must
reintroduce an explicit discriminant. Saying this unprompted is a strong signal.

---

### Q2 — "Sealed interface or enum? How do you choose?"

**Mid-level answer:** "Enums are for constants, sealed interfaces are for classes."

**Senior answer:** "The test is whether the variants carry **different data**. An enum's
constants are fixed singleton instances, so every constant has the same fields — if
`DECLINED` needs a `reasonCode` and `APPROVED` needs a `gatewayReference`, an enum
forces both fields onto both constants and half of them are null in every state. That
is the 'boolean flag plus six nullable fields' response object that every codebase has
one of, and the symptom is callers writing null checks to infer which state they are
in. A sealed interface over records gives each variant precisely its own data, and the
compiler stops you from reading a field that does not exist in that state. Going the
other way: if the variants carry no data — a closed set of labels — an enum is strictly
better, because you get `values()`, `EnumSet`/`EnumMap` with a bit-vector
representation, `@Enumerated(EnumType.STRING)` in JPA, and free JSON serialization,
none of which a sealed hierarchy gives you. Both give exhaustive switches, so that is
not the deciding factor. In practice I use both together — `PaymentResult` sealed, with
a `DeclineReason` enum inside the `Declined` variant."

**What separates them:** the "different data per variant" test stated as a rule, naming
what you *lose* by choosing sealed when enum was right, and the composite example.
Almost everyone gets the first half; very few name the `EnumSet`/JPA/Jackson losses.

**Interviewer's follow-up:** "Can an enum implement a sealed interface?" Yes — enums are
implicitly final, so they satisfy the closure rule, and it is a genuinely useful pattern
for a hierarchy where one branch is a fixed set of constants.

---

### Q3 — "How is this different from a TypeScript discriminated union?"

**Mid-level answer:** "They do the same thing — both let you switch on all the cases."

**Senior answer:** "The guarantee is the same, and I'd call it an honest analogue —
that is unusual in this direction. Four real differences. First, TypeScript narrows
**structurally on a discriminant property value**; Java narrows **nominally on the
type**, because JVM objects carry their class at runtime, so there is no `kind` field
to declare or forget. Second, TypeScript's union is declared **at the union site** and
costs nothing, so you can form as many overlapping unions over the same members as you
like; Java declares the closure **at the parent** and each variant must opt in, so it is
a two-sided commitment you cannot form after the fact over types you do not own —
though a type can implement several sealed interfaces if you plan for it. Third, Java
requires every permitted subtype to be in the same module, or the same package if
you're in the unnamed module. That is a hard architectural constraint with no
TypeScript equivalent. Fourth, TypeScript unions can contain literals, primitives and
null — `\"paid\" | 404 | null` — and Java's variants must all be reference types. And
one thing Java gains: the seal is enforced by the JVM at class load, whereas every
TypeScript guarantee disappears in the emitted JavaScript."

**What separates them:** giving four specific differences rather than one, and knowing
which direction each difference favours. Interviewers hiring people from TypeScript ask
this to find out whether the candidate has actually written both or is pattern-matching
on vocabulary.

**Interviewer's follow-up:** "Which do you prefer?" There is no right answer, but a good
one names a trade: Java's version is more verbose and more constrained about layout, and
you never forget to set a discriminant or set the wrong one.

---

### Q4 — "What is `non-sealed` for, and when have you used it?"

**Mid-level answer:** "It lets a subclass of a sealed class be extended further."

**Senior answer:** "It is a deliberate hole in your own guarantee, and the honest use
case is a published extension point. If I ship a library with
`sealed interface DomainError permits ValidationError, NotFoundError, PluginError`
and I make `PluginError` `non-sealed`, my switches stay exhaustive over three cases
while plugin authors get a documented type to implement. The trap is what `non-sealed`
does **not** cost you and therefore what it silently costs you: exhaustiveness over the
*parent* still holds, so nothing breaks at compile time — but you have lost the ability
to reason about what that branch *is*. Every `case PluginError p ->` must now treat it
as opaque and use only the declared interface methods, because a subtype may carry
semantics the switch is ignoring. I have seen that go wrong: someone made a variant
`non-sealed` to satisfy a test, another team subclassed it, and refund logic ran against
the base variant's fields with no error anywhere. So my rule is that every `non-sealed`
gets a comment saying why, and if the reason is 'a test needed it', that is not a
reason — records make constructing the real variant trivial."

**What separates them:** knowing that `non-sealed` preserves parent exhaustiveness
(counterintuitive, and the source of the danger), a concrete failure story, and the
"comment it or don't use it" review rule.

**Interviewer's follow-up:** "Is there any way to get exhaustiveness back over a
`non-sealed` branch?" No — that is the trade. You can add a runtime check or a visitor,
but the compile-time proof is gone by construction.

---

### Q5 — "You're designing the public API of a shared library. Do you seal it?"

**Mid-level answer:** "Sealing is more modern and safer, so probably yes."

**Senior answer:** "It depends entirely on **who owns the variant list**, and that is a
product decision before it is a technical one. If the complete set is mine and adding to
it is my team's job — payment outcomes, order events, protocol message kinds — then
sealing is right, because I want every consumer's build to break when the set changes,
and because I want to be able to add a method to the interface without breaking
implementors that do not exist. If consumers are supposed to extend it — a plugin
contract, a strategy interface, a transport abstraction — then sealing is a bug: they
will hit `IncompatibleClassChangeError` at class load with no workaround at all, not
even reflection, because the JVM enforces the permitted list. There is a hybrid I like:
seal the type, but include one `non-sealed` extension-point subinterface. Consumers
implement that, my switches stay exhaustive over the first-party variants, and the
extension point has a documented contract rather than being an accident. The other
constraint to check up front is packaging: permitted subtypes must be in the same
module, or the same package outside JPMS — so if my variants want to live in different
artifacts, the domain is telling me it is not really a closed set, and I should
listen."

**What separates them:** framing it as ownership rather than as a style choice, knowing
the exact consumer-side failure mode, offering the hybrid, and using the packaging
constraint as design feedback rather than as an obstacle.

**Interviewer's follow-up:** "How would you un-seal it later if you got it wrong?"
Removing `sealed` is source- and binary-compatible for consumers, so it is a
non-breaking change — which is a genuinely reassuring answer and means sealing is a
lower-risk default than people assume. Adding a *variant* is the breaking change.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Sealing is enforced by the JVM at class load, not just by `javac`. Generics
   (Topic 06) are the opposite — erased entirely, with no runtime enforcement. Why did
   Java make different choices for these two features? What would break if sealing were
   compile-time only?

2. A type can implement more than one sealed interface. Given that, how close can you
   get to TypeScript's ability to form arbitrary unions over existing types? Where
   exactly does the approach run out?

3. `non-sealed` preserves exhaustiveness over the parent. Argue that this is the right
   design. Then argue it is a trap. Which argument do you act on in a code review?

4. Permitted subtypes must share a module or package. Is that a technical necessity or a
   design choice? Construct the argument for it being necessary — what would the JVM
   have to do to verify a seal across modules, and why might that be unacceptable?

5. Your sealed hierarchy is serialized to Kafka and consumed by a Python service. What
   *exactly* survives the boundary, and what does not? Now answer the same question for
   a consumer that is a second Java service on a different deployment cadence.

6. Enums give exhaustiveness too. So does a sealed hierarchy of dataless records. Beyond
   `EnumSet` and framework integration, is there any *semantic* difference between the
   two? Think about identity, and about what `==` means for each.

7. Adding a variant to a sealed type breaks every consumer's build. That is presented as
   the feature. Describe a realistic situation where it is instead a serious operational
   problem, and say what you would do about it.

---

## Quick reference card

### Declaration forms

```java
// interface, explicit permits
public sealed interface PaymentResult permits Approved, Declined, Failed { }

// interface, permits inferred (all variants in the same file)
public sealed interface PaymentResult { }
record Approved(String ref) implements PaymentResult { }

// nested variants (implicitly public static; permits inferred)
public sealed interface PaymentResult {
    record Approved(String ref) implements PaymentResult { }
    record Declined(String reason) implements PaymentResult { }
}

// sealed abstract class, when variants share state
public sealed abstract class PaymentAttempt permits CardAttempt, WalletAttempt { }

// the three closure options for every permitted subtype
public record Approved(String ref) implements PaymentResult { }        // implicitly final
public final class Declined implements PaymentResult { }               // explicitly final
public sealed class Failed implements PaymentResult permits A, B { }   // continues, closed
public non-sealed class Other implements PaymentResult { }             // open. Comment it.
```

### The rules the compiler enforces

| Rule | Error you get if you break it |
|---|---|
| Every permitted subtype is `final`, `sealed`, or `non-sealed` | `sealed, non-sealed or final modifiers expected` |
| Only permitted types may implement | `class is not allowed to extend sealed class` |
| Same module, or same package if unnamed | `...from another package` / invalid permits clause |
| Everything in `permits` must actually implement it | `invalid permits clause` |
| `permits` may be omitted only if all subtypes are in the same file | `sealed class must have subclasses` |

### Sealed vs enum vs interface

| | `enum` | `sealed` + records | plain `interface` |
|---|---|---|---|
| Closed set | yes | yes | no |
| Exhaustive switch, no `default` | yes | yes | no |
| Different data per variant | no | **yes** | yes |
| Fixed singleton instances | yes | no | no |
| `values()` / `EnumSet` / `EnumMap` | yes | no | no |
| JPA `@Enumerated`, free JSON | yes | needs config | needs config |
| Third parties can extend | no | no (unless `non-sealed`) | yes |

### Reflection API

```java
PaymentResult.class.isSealed();                  // boolean
PaymentResult.class.getPermittedSubclasses();    // Class<?>[]
Approved.class.isRecord();                       // boolean
```

### Gotchas checklist

- [ ] Variants must be in the same package (or same named module). Nest them if unsure.
- [ ] No per-variant data? You wanted an `enum`.
- [ ] Different data per variant but you used an `enum`? You wanted this.
- [ ] Crossing a wire? Add `@JsonTypeInfo` + `@JsonSubTypes` — and test that the
      registered list matches `getPermittedSubclasses()`.
- [ ] Every `non-sealed` gets a comment justifying the hole.
- [ ] Do not seal a type your consumers are supposed to implement.
- [ ] Adding a variant is a **breaking change**. Removing `sealed` is not.
- [ ] Prefer `sealed interface` over `sealed abstract class` so variants can be records.
- [ ] Never add a `default` to a switch over a sealed type. That is Topic 29's
      single most important point, and it silently destroys everything above.

---

## When would I use this at work?

**1. Any time you catch yourself writing a response object with a boolean and six
nullable fields.**
`isSuccess()`, `getErrorCode()`, `getRetryAfter()` — the shape where half the fields are
null in every state and callers reverse-engineer the state from which ones are
populated. That is a sealed hierarchy that has not been written yet. Payment results,
validation outcomes, external API responses, and command results are the usual
suspects. The refactor is mechanical and the payoff is immediate: every caller's null
checks disappear.

**2. Modelling domain events for an event-sourced or event-driven service.**
`orderflow`'s `OrderEvent` hierarchy is switched over by the projector, the read-model
updater, the audit log, the webhook publisher and the saga coordinator (Topic 117).
Adding `OrderPartiallyRefunded` breaks all five builds, so all five get handled in the
PR that introduces the event, by the person who understands it. Without sealing, you
find the fifth one in production two weeks later.

**3. Reviewing an interface someone just made `public`.**
The question "should this be sealed?" is really "do we own the complete list of
implementations, and do we want to keep owning it?" Asking it at review time is cheap.
Asking it after a consumer has shipped an implementation is a compatibility
negotiation. This is Topic 03's "everything public is a support obligation", with a
language feature attached.

---

## Connected topics

**Prerequisites:**
- **02 — Nominal vs structural typing**: sealed types are nominal narrowing. This is
  why Java needs no `kind` discriminant field and TypeScript does.
- **03 — Access modifiers and packages**: the same-package constraint, and the
  "everything public is a support obligation" framing that makes sealing a design tool
  rather than a syntax preference.
- **04 — Interfaces vs abstract classes**: sealed applies to both; why the interface
  form is almost always better here.
- **16 — Enums, `EnumSet`, `EnumMap`**: the alternative closed set, and everything you
  give up by not using it when it fits.
- **20 — JPMS modules**: the module-boundary rule for permitted subtypes.
- **27 — Records**: the variants. Sealed hierarchies over non-records lose
  deconstruction patterns and generated equality, and are rarely worth it.

**This unlocks:**
- **29 — Pattern matching**: the consumer side. Exhaustiveness, record deconstruction,
  `when` guards, dominance ordering. Sealing without pattern matching is only half the
  feature; pattern matching without sealing is only `instanceof` with better syntax.
- **19 — Serialization** (revisited): where the guarantee stops, and why the wire needs
  an explicit discriminant.
- **46 — Error handling and `ProblemDetail`**: a sealed `DomainError` hierarchy mapped
  exhaustively to HTTP statuses in one `@ControllerAdvice`, with the compiler
  guaranteeing no error type is unmapped.
- **47 — Spring Data JPA projections**: sealed result types as the return of a query
  that can legitimately produce different shapes.
- **117 — Sagas and compensations**: a saga step's outcome is `Succeeded`,
  `FailedRetryable`, or `FailedTerminal` — a sealed hierarchy, and the compensation
  logic is an exhaustive switch over it. This is the payoff of this topic at
  distributed-systems scale, and it is where `Optional` (Topic 26) is definitively not
  enough, because absence has more than one meaning.

---

*Java baseline 21. Sealed classes were previewed in 15 and 16 and became **final in
Java 17**, so nothing here needs `--enable-preview` on a 21 baseline. Exhaustive
pattern-matching `switch` over sealed types became final in **Java 21**. Nothing in this
topic changed between 21 and 25. If your compiler rejects the switch syntax, your
`--release`/`--source` is set below 21 — check with `javac --version` and inspect your
build's compiler configuration; that is the only thing that would explain it.*
