# 02 — Nominal vs Structural Typing, and How It Reshapes Interface Design

## Phase: 1 — Core Language
## Category: FOUNDATION
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Think about getting into a members' club.

**Structural typing is a dress code.** The doorman looks at you. Black tie, polished
shoes? In you go. Nobody asks your name. Nobody checks a list. If you happen to be
dressed correctly, you *are* a member for all practical purposes — even if you walked
in off the street by accident.

**Nominal typing is a membership register.** The doorman looks up your name. You are a
member because you signed the register and were issued a card. Being dressed
identically to a member means nothing. You could be wearing the exact same suit as the
club chairman and still be turned away.

TypeScript is the dress code. Java is the register.

Two more pieces of the analogy that we will use all the way through this doc:

- **An adapter is a visitor badge.** A contractor arrives who can genuinely do the
  job, but is not on the register. You do not change the register. You issue a badge
  that says "this person is authorised for this purpose". That badge is a small piece
  of paperwork you had to write. In Java that paperwork is a class.
- **Accidental membership is a real risk under a dress code.** Someone in a black
  suit who is actually a funeral director gets waved in. Under a register that cannot
  happen. That is the thing nominal typing buys you, and it is worth more than it
  looks.

---

## The bridge from what you know

This topic *is* the bridge. So let us do it properly and slowly.

### What you write in TypeScript today

```ts
interface HasOrderId {
  orderId: string;
}

function audit(x: HasOrderId) {
  console.log(x.orderId);
}

// This class never mentions HasOrderId. It does not have to.
class PaymentAttempt {
  constructor(public orderId: string, public amountMinor: number) {}
}

audit(new PaymentAttempt("ORD-4471", 1999));   // compiles. Perfectly legal.
audit({ orderId: "ORD-4471" });                // also compiles.
audit({ orderId: "ORD-4471", anythingElse: 1 }); // also compiles (via a variable).
```

`PaymentAttempt` satisfies `HasOrderId` because its **shape** matches. The
relationship was never declared. It was *discovered* by the compiler.

This is the single most-used feature of TypeScript and you almost certainly do not
think of it as a feature any more. It is just how types work.

### What you must write in Java

```java
public interface HasOrderId {
    String orderId();
}

// Without "implements HasOrderId", this class is NOT a HasOrderId. Ever.
public final class PaymentAttempt implements HasOrderId {
    private final String orderId;
    private final long amountMinor;

    public PaymentAttempt(String orderId, long amountMinor) {
        this.orderId = orderId;
        this.amountMinor = amountMinor;
    }

    @Override public String orderId() { return orderId; }
    public long amountMinor()         { return amountMinor; }
}
```

Delete the words `implements HasOrderId` and the class still has a method called
`orderId()` that returns a `String`. It is still shaped exactly right. And it will no
longer compile when passed to `audit`:

```
error: incompatible types: PaymentAttempt cannot be converted to HasOrderId
```

**The shape is irrelevant. Only the declaration counts.**

### The honest verdict table

| You know (TypeScript) | Java | Verdict |
|---|---|---|
| `interface` matched by shape | `interface` matched by name + explicit `implements` | **PARTIAL** — the word is the same, the mechanism is not. Your design habits must change. |
| Anonymous object types `{ id: string; total: number }` | — | **NO ANALOGUE** — every type must be a named, declared type. |
| Structural narrowing (`if ('orderId' in x)`) | `instanceof` against a named type | **PARTIAL** — Java narrows too (Topic 29), but only to types that already exist by name. |
| A class accidentally satisfying an interface | — | **NO ANALOGUE**, and this is the point. Accidental conformance is impossible. |
| `type A = B` aliasing two shapes together | — | **NO ANALOGUE** — no type aliases at all. |
| Duck typing at runtime (`if (x.charge)`) | Reflection / `java.lang.reflect.Proxy` | **PARTIAL** — possible, but you leave the type system entirely and lose every compile-time guarantee. |
| Declaration merging, mapped types, conditional types | — | **NO ANALOGUE** |

### The part with no TypeScript equivalent at all

**NO TYPESCRIPT ANALOGUE.**

There is no TypeScript situation in which a type that has exactly the right members
*cannot be used*. In a structurally-typed runtime the compiler's only question is "does
the shape match", and the answer is computed from the members themselves — so
conformance is discovered, never granted. There is nothing to grant it with: TS
interfaces do not exist at runtime, so there is no artefact in which a declaration could
be recorded even if you wanted to make one.

Java's `implements` is recorded as a string in the class file and re-checked by the JVM
when a call site is linked. That is only possible in a runtime that keeps types. The
consequence — that you must write an adapter for a class you do not own, even a perfectly
shaped one — is therefore not a limitation you can work around with better TypeScript
habits. It is what having runtime types costs.

### Where Java looks structural but is not

There is exactly one place Java *feels* like it matches on shape, and it trips up
every TypeScript developer. Lambdas:

```java
Runnable      r = () -> System.out.println("go");
OrderCallback c = () -> System.out.println("go");   // your own interface, same shape
```

The same lambda expression satisfies both. That looks structural. It is not.

A lambda is not a value with a type of its own. It is a **poly expression**: it has no
type until you tell the compiler which named interface it should become. The compiler
looks at the *target type* on the left, checks that the target is an interface with
exactly one abstract method, and manufactures an instance of that named type.

Which means this still fails:

```java
Runnable r = () -> System.out.println("go");
OrderCallback c = r;    // error: incompatible types: Runnable cannot be
                        // converted to OrderCallback
```

Two identically-shaped interfaces are still two unrelated types. The lambda was
flexible; the resulting object is not. That distinction is worth holding onto — it
comes back in Topic 21.

---

## What is this?

A **nominal type system** decides whether type `A` can be used where type `B` is
expected by looking at declared relationships — `extends`, `implements` — and nothing
else. A **structural type system** decides it by comparing the members of `A` and `B`.

Java is nominal. TypeScript is structural. Same keyword, opposite mechanism.

The practical consequence is that in Java, conformance is a **decision someone made
and wrote down**, not a fact the compiler discovered.

---

## Why does it matter?

Three things go wrong when you carry TypeScript instincts into a Java codebase.

1. **You cannot make someone else's class fit your interface.** In TypeScript, if a
   library returns an object with the right fields, it just works. In Java, if the
   Stripe SDK returns a `Charge` and your domain wants a `PaymentRecord`, the compiler
   will not connect them no matter how identical they look. You write an adapter. If
   you did not plan for that, you end up leaking the SDK type through your whole
   service, and six months later you cannot swap payment providers without touching
   two hundred files.

2. **You reach for `Map<String, Object>` to model an anonymous shape**, because that
   is the closest thing to `{ orderId: string; total: number }`. Every read from that
   map is a cast. Every cast is a `ClassCastException` waiting for the one code path
   your tests missed, and it throws at a line where you never typed the word "cast".

3. **You under-use the guarantee you were given.** Nominal typing means a
   `CustomerId` and a `ProductId` — both wrapping a `String` — can never be swapped by
   accident. In TypeScript you need branded types and a helper to get that. In Java it
   is free. Engineers coming from TS routinely model both as `String`, throw away the
   guarantee, and then ship a bug where a product ID is passed as a customer ID and the
   query returns zero rows with no error at all.

---

## Syntax breakdown

Only the genuinely new constructs. `public class` was Topic 01 territory.

### `implements` — the register entry

```java
public final class PaymentAttempt implements HasOrderId, Auditable {
```

| Bit of syntax | What it means |
|---|---|
| `implements HasOrderId` | A declaration of intent: "this class is a member of the `HasOrderId` club". The compiler now *requires* every abstract method of `HasOrderId` to be present. |
| `, Auditable` | You may implement any number of interfaces. Multiple *interface* inheritance is fine; multiple *class* inheritance is not (Topic 04). |
| `final class` | Nobody may subclass this. Relevant here because subclassing is the other way a type joins the register, and `final` shuts that door. |

### `@Override` — a claim the compiler checks

```java
@Override public String orderId() { return orderId; }
```

This annotation is not decoration. It means "I believe this method overrides or
implements something declared above me". If it does not — you typo'd the name, or the
parameter types drifted — you get a compile error instead of a silently-unused method.

In TypeScript you have no equivalent, because there is nothing to declare: a method
either matches the shape or it does not, and the failure surfaces at the call site.
`@Override` moves that failure to the definition site. Always write it.

### `extends` on an interface

```java
public interface RefundableOrder extends HasOrderId {
    long refundableAmountMinor();
}
```

An interface may extend other interfaces. A class implementing `RefundableOrder` is
automatically also a `HasOrderId` — the register entry is transitive.

### An anonymous class — a one-off register entry

```java
HasOrderId adHoc = new HasOrderId() {
    @Override public String orderId() { return "ORD-4471"; }
};
```

This creates an unnamed class that *does* declare `implements HasOrderId`. It is the
closest Java gets to writing an object literal that satisfies an interface — and note
that even here you had to name the interface. There is no way to write "an object with
an `orderId()` method" without naming a type.

For single-method interfaces you would write a lambda instead. Anonymous classes are
still what you use when the interface has two or more methods.

---

## Example 1 — minimal

Two types with identical shapes, and the compiler refusing to connect them.

```java
public class NominalDemo {

    interface Identified {
        String id();
    }

    // Same method. Same signature. Different name on the door.
    interface Labelled {
        String id();
    }

    record Sku(String id) implements Identified {}

    static void printIdentified(Identified x) {
        System.out.println(x.id());
    }

    static void printLabelled(Labelled x) {
        System.out.println(x.id());
    }

    public static void main(String[] args) {
        Sku sku = new Sku("SKU-4471");

        printIdentified(sku);     // fine — Sku declared "implements Identified"
        // printLabelled(sku);    // does NOT compile. Uncomment to see the error.
    }
}
```

Uncomment the last line and compile. You get:

```
error: incompatible types: Sku cannot be converted to Labelled
```

In TypeScript both calls would compile, and you would never even notice there were two
interfaces. That difference is the whole topic in five lines.

---

## Example 2 — production scenario

You are building the payments module of `orderflow`. Today you charge through Stripe.
Finance has already told you that within a year you will also support a domestic
provider for lower fees.

The Stripe SDK gives you back a class you do not control:

```java
// From the vendor SDK. You cannot edit this file, and it does not know your domain.
package com.stripe.model;

public final class Charge {
    public String getId()            { ... }
    public Long   getAmount()        { ... }
    public String getCurrency()      { ... }
    public String getStatus()        { ... }   // "succeeded" | "pending" | "failed"
    public String getFailureCode()   { ... }
    // ...and about forty more methods you do not want in your domain
}
```

### The TypeScript instinct, transplanted

In TypeScript you would declare what you need and be done:

```ts
interface PaymentOutcome {
  getId(): string;
  getStatus(): string;
}
// A Stripe Charge already satisfies this. Zero code written.
```

In Java the equivalent interface is useless on its own, because `Charge` does not
declare `implements PaymentOutcome` and you cannot make it. So the tempting move is to
just use `Charge` everywhere.

### What that costs, concretely

```java
// The version that ships in week two and hurts in month nine.
public class OrderService {

    public void placeOrder(OrderRequest request) {
        Charge charge = stripeClient.charge(request.amountMinor(), request.currency());

        if ("succeeded".equals(charge.getStatus())) {          // vendor string, in your domain
            inventory.commitReservation(request.orderId());
        } else {
            orders.markFailed(request.orderId(), charge.getFailureCode());  // vendor code
        }
    }
}
```

`com.stripe.model.Charge` is now in the signature or body of your order service, your
repository, your event publisher and your tests. So is the string `"succeeded"`, which
is a Stripe vocabulary word, not an `orderflow` word.

When the second provider arrives, its SDK returns `TransactionResult` with a status of
`"OK"`. There is no shape you can widen to cover both, because nominal typing gives
you no way to say "either of these two vendor classes". You end up with an
`if (provider == STRIPE)` in every file that touches a payment.

### The Java-shaped design

Own your boundary type. Adapt at the edge, once.

```java
package com.orderflow.payments;

/** What orderflow means by "we tried to take money". Vendor-free. */
public sealed interface PaymentResult {

    record Captured(String providerRef, long amountMinor, String currency)
            implements PaymentResult {}

    record Declined(String providerRef, DeclineReason reason)
            implements PaymentResult {}

    record Pending(String providerRef)
            implements PaymentResult {}
}
```

```java
package com.orderflow.payments;

/** The only thing orderflow knows how to ask a payment provider to do. */
public interface PaymentGateway {
    PaymentResult charge(String idempotencyKey, long amountMinor, String currency);
}
```

```java
package com.orderflow.payments.stripe;

/** The visitor badge. This is the ONLY file that imports com.stripe.*. */
public final class StripePaymentGateway implements PaymentGateway {

    private final StripeClient client;

    public StripePaymentGateway(StripeClient client) {
        this.client = client;
    }

    @Override
    public PaymentResult charge(String idempotencyKey, long amountMinor, String currency) {
        Charge charge = client.create(idempotencyKey, amountMinor, currency);

        return switch (charge.getStatus()) {
            case "succeeded" -> new PaymentResult.Captured(
                                    charge.getId(), charge.getAmount(), charge.getCurrency());
            case "pending"   -> new PaymentResult.Pending(charge.getId());
            default          -> new PaymentResult.Declined(
                                    charge.getId(), DeclineReason.fromStripe(charge.getFailureCode()));
        };
    }
}
```

Now `OrderService` depends on `PaymentGateway` and `PaymentResult` only. Adding the
domestic provider is one new class in one new package. Nothing in `orders` changes.

### Read the trade honestly

You wrote a file you would not have written in TypeScript. That file is real cost:
about sixty lines, plus a mapping test, plus one more place a new decline code has to
be registered.

What you bought:

| Bought | Why it matters here |
|---|---|
| A grep-able boundary | `grep -r "com.stripe"` returns exactly one package. That is your blast radius for a vendor migration, and you can state it in a planning meeting with a number. |
| No accidental conformance | Nobody can pass a `RefundResult` where a `PaymentResult` is expected just because both happen to have `getId()` and `getStatus()`. |
| `implements` as documentation | The IDE's "find implementations" on `PaymentGateway` lists every provider. There is no equivalent query for "everything structurally compatible", because in TS that set is unbounded and unknowable. |
| Exhaustiveness later | Because `PaymentResult` is `sealed`, adding a fourth outcome becomes a compile error at every `switch`. That is Topic 28, and it is the closest true analogue to your discriminated unions. |

The rule of thumb, stated plainly:

> **In TypeScript, an interface is an assertion about shape. In Java, `implements` is
> a design commitment.** Write the adapter. It is the price of the boundary, and the
> boundary is what you were buying.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `Map<String, Object>` as a stand-in for an anonymous type

**Wrong:**
```java
// "I just need orderId and total here, I don't want a whole class."
public Map<String, Object> orderSummary(long orderId) {
    return Map.of("orderId", orderId,
                  "total",   order.totalMinor(),
                  "status",  order.status().name());
}

// ...somewhere else, three modules away
long total = (Long) summary.get("totalMinor");   // note the typo
```

**Exact symptom:** at runtime, on the one endpoint nobody load-tested:
```
java.lang.NullPointerException: Cannot invoke "java.lang.Long.longValue()"
  because the return value of "java.util.Map.get(Object)" is null
```
or, if the key exists but the type drifted:
```
java.lang.ClassCastException: class java.lang.Integer cannot be cast to
  class java.lang.Long (java.lang.Integer and java.lang.Long are in module
  java.base of loader 'bootstrap')
```
The stack trace points at a line containing no arithmetic and no obvious null.

**Root cause:** you moved the contract out of the type system and into string keys.
The compiler has nothing to check. A rename in the producer is invisible to the
consumer. This is what a structurally-typed language gives you for free and a nominal
one does not — so people fake it, and the fake has no compile-time half.

**Fix:** declare the type. A record is three words.
```java
public record OrderSummary(long orderId, long totalMinor, OrderStatus status) {}
```
Rename `totalMinor` now and every consumer fails to compile, which is what you wanted.

---

### Trap 2 — defining an interface you cannot make a third-party class implement

**Wrong:**
```java
public interface PaymentOutcome {
    String getId();
    String getStatus();
}

void handle(PaymentOutcome outcome) { ... }

handle(stripeClient.charge(...));    // Charge "obviously" fits
```

**Exact symptom:** a compile error, not a runtime one — which is the good news:
```
error: incompatible types: com.stripe.model.Charge cannot be converted to
  com.orderflow.payments.PaymentOutcome
```

**Root cause:** you designed the interface as a *shape description*, which is the
TypeScript move. In Java an interface only helps if the implementor declares it, and
you cannot edit the vendor's source.

**Fix:** either an adapter class (as in Example 2), or — if the interface is truly
just "get me two fields" — a plain function type at the call site:
```java
void handle(String id, String status) { ... }
handle(charge.getId(), charge.getStatus());
```
Do not add an interface that no reachable class can ever implement. It is dead weight
that reads like a contract.

---

### Trap 3 — modelling distinct identifiers as the same type

**Wrong:**
```java
public Order findOrder(String customerId, String productId) { ... }

// three files away, arguments in the wrong order
Order o = findOrder(productId, customerId);
```

**Exact symptom:** no exception. No error. The query runs and returns zero rows. The
endpoint returns an empty list with HTTP 200. You find it from a business metric —
"conversion from the order-history page dropped to 0% for 4% of users" — or from a
customer complaint, days later.

**Root cause:** two conceptually different things share one type, so the compiler
cannot tell them apart. In TypeScript you would reach for a branded type
(`type CustomerId = string & { __brand: 'CustomerId' }`) precisely because structural
typing gives you nothing here. In Java, nominal typing gives it to you for free — and
you declined to take it.

**Fix:** name the types.
```java
public record CustomerId(String value) {
    public CustomerId {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("customerId must not be blank");
        }
    }
}
public record ProductId(String value) { /* same validation */ }

public Order findOrder(CustomerId customerId, ProductId productId) { ... }
```
Now the swapped call is:
```
error: incompatible types: ProductId cannot be converted to CustomerId
```
This is the *upside* of nominal typing, and TypeScript engineers systematically
under-claim it because in TS it costs something and in Java it does not.

> Cost note: each wrapper is an object allocation (Topic 01). On a hot path with
> millions of IDs that is a real number, and Project Valhalla's value types are the
> eventual answer. For an order-lookup API called a few thousand times a second, it is
> noise. Do not skip the type to save an allocation you have not measured.

---

### Trap 4 — assuming two same-shaped functional interfaces are interchangeable

**Wrong:**
```java
public interface PriceAdjuster {
    long adjust(long amountMinor);
}

java.util.function.LongUnaryOperator vatRule = amount -> amount * 120 / 100;

PriceAdjuster adjuster = vatRule;    // "same shape, surely?"
```

**Exact symptom:**
```
error: incompatible types: java.util.function.LongUnaryOperator cannot be
  converted to com.orderflow.pricing.PriceAdjuster
```

**Root cause:** the lambda was flexible; the *object it became* is a `LongUnaryOperator`
and nothing else. Target typing happens once, at the point the lambda is written.

**Fix:** either re-target the lambda, or bridge with a method reference.
```java
PriceAdjuster adjuster = vatRule::applyAsLong;   // one method reference, done
```
This is worth knowing because it is how you convert between the JDK's functional
interfaces and your domain's ones without writing an adapter class. Method references
are Topic 22.

---

### Trap 5 — reflection-based binding, which reintroduces structural matching with no compile check

**Wrong:**
```java
public record OrderSummary(long orderId, long totalMinor) {}

// Someone renames the JSON field in the producer service:
// {"orderId": 4471, "total_minor": 1999}
OrderSummary s = objectMapper.readValue(json, OrderSummary.class);
```

**Exact symptom:** at runtime, in the consumer, in production:
```
com.fasterxml.jackson.databind.exc.UnrecognizedPropertyException:
  Unrecognized field "total_minor" (class com.orderflow.orders.OrderSummary),
  not marked as ignorable
```
or, with `FAIL_ON_UNKNOWN_PROPERTIES` disabled — which most teams do — **no error at
all** and `totalMinor` silently equal to `0`. Every order shows a total of £0.00.

**Root cause:** Jackson, JPA, Spring's `@ConfigurationProperties` and every other
reflection-based binder match by *name and shape at runtime*. That is structural
typing — but performed after compilation, so the compiler cannot help. You get the
looseness of TypeScript with none of the checking.

**Fix:** treat the wire format as its own contract, not as your domain class.
- Make the mapping explicit: `@JsonProperty("total_minor") long totalMinor`.
- Turn `FAIL_ON_UNKNOWN_PROPERTIES` **on** for anything you control end to end, so a
  drift is loud rather than silent.
- Contract-test the boundary against a recorded payload, so a producer rename breaks a
  test rather than a customer.

The general lesson: **every time a framework does the matching, you are back in
structural-typing land without the compiler.** Annotations are inert metadata (see the
translation table in the master plan) — they only do something because a processor read
them at runtime. Serialization hazards in full are Topic 19; binding is Topic 43.

---

## Hands-on proof

Everything below is a command **you** run. I do not have a JVM and will not print
output and claim it is real. What I can give you exactly is what to look for and how to
read each possible result.

### Setup

```bash
mkdir -p ~/java-lab/02 && cd ~/java-lab/02
java --version      # expect 21 or 25
```

### Proof 1 — shape is not enough

`ShapeIsNotEnough.java`:
```java
public class ShapeIsNotEnough {

    interface Identified { String id(); }
    interface Labelled   { String id(); }

    record Sku(String id) implements Identified {}

    static void takeLabelled(Labelled x) { System.out.println(x.id()); }

    public static void main(String[] args) {
        takeLabelled(new Sku("SKU-4471"));
    }
}
```

```bash
javac ShapeIsNotEnough.java
```

**What to look for:** the compiler must reject this.

| What you see | What it means |
|---|---|
| `error: incompatible types: Sku cannot be converted to Labelled` | The expected result. You have observed nominal typing refusing a perfect structural match. |
| It compiles | You accidentally wrote `implements Labelled` somewhere, or `Labelled` has no abstract methods. Check the file. |
| A different error mentioning `record` | Your JDK is older than 16. Check `java --version`; this curriculum needs 21+. |

Now add `, Labelled` to the record declaration and recompile. It passes. **The only
thing that changed is a declaration** — no method bodies moved.

### Proof 2 — the declaration is physically recorded in the class file

```bash
javac ShapeIsNotEnough.java
javap -v 'ShapeIsNotEnough$Sku.class' | head -40
```

**What to look for:** near the top of the output, a line beginning with
`interfaces:` and, in the class declaration line, the word `implements` followed by the
interface names. Also look in the constant pool for `Class` entries naming
`ShapeIsNotEnough$Identified`.

**How to read it:** the interface names are stored as **strings in the class file**.
The JVM checks conformance by comparing those names — plus the defining class loader —
at link time. This is why it is called *nominal*: the check is literally a name
comparison, at runtime as well as compile time.

Contrast: nothing about the *shape* of `Sku` is compared to anything. There is no
structural check anywhere in the JVM.

> Note on `javap -v` output volume: it is long. Pipe to `head` or `grep -i interface`.
> Reading class files properly is Topic 76; today you are only confirming one fact.

### Proof 3 — the lambda is flexible, the object is not

`TargetTyping.java`:
```java
public class TargetTyping {

    interface PriceAdjuster { long adjust(long amountMinor); }

    public static void main(String[] args) {
        PriceAdjuster vat = amount -> amount * 120 / 100;          // fine
        java.util.function.LongUnaryOperator op = amount -> amount * 120 / 100;  // also fine

        PriceAdjuster broken = op;      // should NOT compile
        System.out.println(broken.adjust(1000));
    }
}
```

```bash
javac TargetTyping.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `error: incompatible types: LongUnaryOperator cannot be converted to PriceAdjuster` | Expected. The lambda's flexibility was a *compile-time* conversion, used up at the point of assignment. |
| It compiles | Impossible on a correct JDK 21. Re-check that you did not declare `PriceAdjuster extends LongUnaryOperator`. |

Now replace `= op;` with `= op::applyAsLong;` and recompile. It passes. **One method
reference is the entire adapter** for single-method interfaces — which is why Java
needs far fewer adapter classes than this topic might make you fear.

### Proof 4 — reflection can do structural matching, and that is the danger

`DuckTyping.java`:
```java
import java.lang.reflect.Method;

public class DuckTyping {

    record Sku(String id) {}
    record Wallet(String id) {}

    static String callId(Object anything) throws Exception {
        Method m = anything.getClass().getMethod("id");   // matched by NAME, at runtime
        return (String) m.invoke(anything);
    }

    public static void main(String[] args) throws Exception {
        System.out.println(callId(new Sku("SKU-4471")));
        System.out.println(callId(new Wallet("WAL-0001")));
        System.out.println(callId("a plain string"));      // has no id()
    }
}
```

```bash
java DuckTyping.java
```

**What to look for:** the first two lines succeed; the third throws.

| What you see | What it means |
|---|---|
| Two values printed, then `java.lang.NoSuchMethodException: java.lang.String.id()` | Expected. Reflection matched `Sku` and `Wallet` purely on shape — real duck typing — and the failure moved from compile time to runtime. |
| A `ClassCastException` on the cast to `String` | You changed a return type. Note that reflection gave you no help there either. |
| An `InaccessibleObjectException` | The class is not exported/open to your code. That is Topic 20 (JPMS) and confirms that reflection is not unconditional. |

**How to read the result:** you *can* have structural typing in Java. It costs you
every compile-time guarantee, and it is exactly what Jackson, Hibernate and Spring do
under the hood. When one of those fails at startup or at first request, this is the
mechanism that failed.

### Proof 5 — count your vendor blast radius

In any real Java repo you have access to:

```bash
grep -rl --include='*.java' 'com\.stripe\.' src/main/java | sort | uniq -c | wc -l
grep -rl --include='*.java' 'com\.stripe\.' src/main/java | sed 's#/[^/]*$##' | sort -u
```

**What to look for:** the number of *packages* (second command) that mention the
vendor.

| What you see | What it means |
|---|---|
| One package | You have an adapter boundary. A provider migration is scoped and estimable. |
| More than three packages | The vendor type has leaked into your domain. This is the concrete, countable cost of skipping the adapter — and it is the number to bring to planning. |
| Zero | Either you have no such dependency, or the import is wildcarded/shaded. Check the POM. |

This is not a JVM measurement; it is a design measurement. It is also the one from this
topic you will actually use at work.

---

## Practice exercises

Write real files, run them, and keep your notes on what the compiler said.

### 1 — Easy: make the compiler refuse, then make it accept

Create two interfaces `Shippable` and `Deliverable`, each with a single method
`String destinationPostcode();`. Create a record `Parcel` that implements only
`Shippable`.

1. Write a method that takes a `Deliverable` and call it with a `Parcel`. Record the
   exact compile error text.
2. Make it compile in **three different ways**, and for each, write one sentence on
   what you gave up:
   - add `implements Deliverable` to `Parcel`
   - write an adapter class
   - write a lambda or method reference that produces a `Deliverable`
3. Finally, write the TypeScript equivalent of step 1 and confirm it compiles with no
   changes at all. Note in one sentence what the TS compiler had to know that the Java
   compiler refused to infer.

### 2 — Medium: the identifier audit (combines Topic 01)

Here is a fragment of an `orderflow` service. It contains **four** distinct defects
rooted in this topic, plus **one** carried over from Topic 01.

```java
public class FulfilmentService {

    public Map<String, Object> summarise(String orderId, String customerId) {
        Map<String, Object> out = new HashMap<>();
        out.put("orderId", orderId);
        out.put("customer", customerId);
        out.put("totalDue", 19.99);
        return out;
    }

    public void reserve(String productId, String warehouseId, Integer quantity) {
        Integer onHand = stockByProduct.get(productId);
        if (onHand > quantity) {
            stockByProduct.put(productId, onHand - quantity);
        }
    }

    public boolean sameOrder(Long a, Long b) {
        return a == b;
    }

    public void ship(String customerId, String productId) {
        reserve(customerId, productId, 1);
    }
}
```

For each defect: state the **observable symptom** an on-call engineer would see (an
exception message shape, a wrong business number, a silently-empty response — not "bad
practice"), then rewrite the class. Use nominal types to make at least two of the
defects into compile errors rather than runtime ones.

### 3 — Hard: production simulation — the second payment provider

You are the `orderflow` payments owner. The service currently uses one provider whose
SDK you cannot modify.

**Part A.** Write a small "vendor SDK" you do not get to change:
```java
package vendor.alpha;
public final class AlphaCharge {
    public String reference()  { return "ALPHA-" + System.nanoTime(); }
    public long   minorUnits() { return 1999L; }
    public String state()      { return "CAPTURED"; }
}
```
and a second one, `vendor.beta.BetaTxn`, with methods `txnId()`, `amountPence()` and
`resultCode()` returning `"00"` for success. Note that the two vendors agree on
nothing: not method names, not units, not vocabulary.

**Part B.** Design the `orderflow` boundary: a `sealed interface PaymentResult` and a
`PaymentGateway` interface. Write two adapters. Write an `OrderService.placeOrder` that
compiles against neither vendor package.

**Part C.** Prove the boundary holds:
```bash
grep -rl --include='*.java' 'vendor\.' src/main/java
```
The result must list exactly two files. If it lists three, you leaked. Fix it and
re-run — the grep is the test.

**Part D.** Now break it deliberately. Add a third outcome, `Refunded`, to
`PaymentResult`. Recompile the whole project **without touching any other file**.

- How many compile errors did you get, and in which files?
- Would the equivalent change in a TypeScript discriminated union have produced the
  same set of errors? Where would it have differed, and why?
- If you got **zero** errors, your `switch` statements have a `default` branch. Explain
  why that `default` is a liability here, and what you would replace it with. (Sealed
  types and exhaustiveness are Topic 28 — you are meeting the motivation early, on
  purpose.)

**Part E.** Argue the other side honestly. Under what circumstances is the adapter
genuinely not worth writing? Give a concrete condition, not "when it's a small
project".

---

## Interview questions

### Q1 — "TypeScript interfaces are structural, Java's are nominal. So what?"

**Mid-level answer:** "Java checks types by name, so a class has to say `implements`.
TypeScript just checks the shape."

**Senior answer:** "The mechanism difference is one sentence; the design consequence is
the real answer. Structural typing means a type can satisfy a contract *by accident* —
which is convenient, and also means you can never enumerate the set of things that
satisfy an interface. Nominal typing makes conformance a deliberate declaration, so
`implements` becomes a searchable design commitment and 'find implementations' is a
complete answer. The cost is adapters: I cannot make a vendor class satisfy my domain
interface, so I write a mapping class at the boundary. That cost is exactly what buys
me a one-package blast radius when we swap providers. So it's a trade, and I'd rather
pay it at the edge than have vendor types in my order service."

**What separates them:** the mid answer states the rule. The senior answer names both
sides of the trade — accidental compatibility versus adapter boilerplate — and gives a
concrete operational payoff (blast radius) rather than an abstract one.

**Follow-up the interviewer asks:** "Where in Java does something *look* structural?"
They want lambdas and target typing, and they are checking whether you can explain that
the flexibility is at the conversion point only.

---

### Q2 — "You need a Stripe `Charge` to satisfy your `PaymentResult` interface. How?"

**Mid-level answer:** "You can't — Java doesn't do that. I'd use the `Charge` class
directly, or make my own class extend it."

**Senior answer:** "You write an adapter, and you put it in the only package that
imports the vendor namespace. Extending `Charge` is usually not available — SDK model
classes are frequently `final`, and even when they aren't, inheriting a vendor class
means inheriting its lifecycle, its serialization and its future changes. The adapter
also does more than type conversion: it normalises vocabulary. Stripe says
`"succeeded"`, the other provider says `"00"`, and my domain says `Captured`. If I
skip the adapter, that vendor vocabulary ends up in my order service and in my
database."

**What separates them:** recognising that the adapter is doing *semantic* translation,
not just type juggling, and knowing why inheritance is the wrong tool here.

**Follow-up:** "What if the vendor SDK class is an interface rather than a final
class — would you implement it?" They want you to say no, and to say why: implementing
a vendor interface makes every future method they add a compile break for you.

---

### Q3 — "Both of these are `String`. Is that a problem?"

```java
public Order findOrder(String customerId, String productId);
```

**Mid-level answer:** "It's fine, but you have to be careful about argument order."

**Senior answer:** "It's a defect waiting for a refactor. Java's type system will
happily let you swap them, and the failure mode is silent — the query returns zero rows
and the endpoint returns an empty 200. You find it from a conversion metric, not a
stack trace. The fix is a record wrapper per identifier, which nominal typing makes
free: `findOrder(CustomerId, ProductId)` turns the swap into a compile error. In
TypeScript you'd need branded types to get the same guarantee, which is why TS teams
often skip it — but in Java there's no excuse. The cost is one allocation per ID, which
matters on a hot path and doesn't on a lookup endpoint."

**What separates them:** naming the *silent* failure mode, knowing the TypeScript
comparison well enough to explain why the habit exists, and pricing the fix rather than
declaring it free.

**Follow-up:** "When would you not wrap?" They are probing whether you optimise on
evidence. A good answer mentions hot loops, high-cardinality collections, and Project
Valhalla as the eventual escape.

---

### Q4 — "Jackson deserialized this JSON into my record with no `implements` anywhere. Doesn't that make Java structurally typed?"

**Mid-level answer:** "Jackson uses reflection, so it's different from the type
system."

**Senior answer:** "It's genuinely structural matching — Jackson compares JSON property
names to constructor parameters or accessors at runtime. But it happens *after*
compilation, so you get TypeScript's looseness with none of TypeScript's checking. A
producer renaming a field is invisible to `javac` and shows up as an
`UnrecognizedPropertyException`, or worse, as a silent zero if the team disabled
`FAIL_ON_UNKNOWN_PROPERTIES`. That's why I treat the wire format as its own contract:
explicit `@JsonProperty` mappings, unknown-property failure switched on for internal
services, and a contract test against a recorded payload so a rename breaks CI rather
than a customer."

**What separates them:** identifying that reflection is *real* structural typing at the
wrong time, and having a concrete mitigation rather than a shrug.

**Follow-up:** "Same question for JPA field mapping and for
`@ConfigurationProperties`." They want to see you generalise: any reflective binder has
this shape.

---

### Q5 — "Give me a case where nominal typing is worse than structural, and defend it."

**Mid-level answer:** "It's more verbose. You have to write more code."

**Senior answer:** "Retrofitting. If two teams independently define
`interface HasOrderId` with an identical method, structural typing unifies them at zero
cost and nominal typing gives you two incompatible types plus an adapter — pure
ceremony for no safety gain, because the types genuinely mean the same thing. The
second case is testing: in Jest I can hand a function an object literal with the three
fields it reads; in Java I need a full implementation or a mock, which is why Mockito
exists and why testability becomes a *design* property rather than a test-time trick.
The honest answer is that nominal typing pushes the cost forward — you pay at
definition time to avoid paying at debugging time. On a codebase that lives five years
with rotating owners, that's the right trade. On a two-week spike, it isn't."

**What separates them:** giving a case where nominal typing genuinely loses, not a
strawman, and framing the difference as *when* you pay rather than *whether*.

**Follow-up:** "How does Go's approach differ from both?" Optional depth. Go's
interfaces are structural but the interface is declared by the *consumer*, which is a
third point in the design space — mentioning it shows you have thought about the
trade-off rather than memorised two options.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Java could have added structural interfaces at any point in the last twenty years
   without breaking existing code — a new keyword, opt-in. It has not. What would break
   in the *runtime*, not the language, if a class could satisfy an interface it never
   declared? Think about what the JVM actually does at a call site.

2. `implements` gives you "find all implementations" as a complete, finite answer. In
   TypeScript that query has no answer. Is a complete answer always useful? Name a
   situation where it actively misleads you.

3. You have a `record CustomerId(String value)` and a `record ProductId(String value)`.
   They erase to different classes, so the compiler distinguishes them. Now they both
   go through Jackson to JSON. What happened to the distinction, and at which exact
   moment was it lost?

4. A lambda can become a `Runnable` or a `PriceAdjuster` depending only on the variable
   you assign it to. Is that structural typing? Argue both sides, then commit to an
   answer and say what evidence would settle it.

5. Nominal typing prevents accidental compatibility. Give a concrete example where
   accidental compatibility would have been *correct* and Java's refusal cost you real
   time. Then say whether you would still choose nominal, and why.

6. Your team has 40 adapter classes and someone proposes generating them with an
   annotation processor or MapStruct. What does that give back, and what does it quietly
   take away? (Hint: think about where the failure surfaces when the source class
   changes.)

7. The master plan calls this a **PARTIAL** analogue rather than **NO ANALOGUE**. Do you
   agree? What would have to be true for it to deserve **NO ANALOGUE**?

---

## Quick reference card

### Syntax

```java
interface OrderView { String id(); }                  // declare a contract
class Product implements OrderView, Auditable { ... }     // join the register (many allowed)
interface Refundable extends OrderView { ... }        // interfaces extend interfaces
record Sku(String id) implements OrderView {}         // records can implement too
@Override public String id() { ... }                  // compiler-checked claim

// one-off implementation, no name
OrderView v = new OrderView() { public String id() { return "X"; } };

// single-method interface: a lambda, target-typed
PriceAdjuster p = amount -> amount * 120 / 100;

// convert between same-shaped functional interfaces
PriceAdjuster p2 = someLongUnaryOperator::applyAsLong;
```

### The rules, compressed

| Question | Java's answer |
|---|---|
| Does shape make `A` a `B`? | No. Never. |
| What makes `A` a `B`? | `A implements B`, `A extends B`, or a supertype of `A` does. |
| Can I add `implements` to a class I don't own? | No. Write an adapter. |
| Can two interfaces with identical methods be swapped? | No. |
| Can one lambda satisfy two same-shaped interfaces? | Yes — but each conversion is separate; the resulting objects are not interchangeable. |
| Does anything in Java match on shape? | Reflection, and every framework built on it. At runtime, unchecked. |

### Costs at a glance

| | Structural (TS) | Nominal (Java) |
|---|---|---|
| Retrofitting a foreign type | free | adapter class |
| Accidental conformance | possible | impossible |
| "Find all implementations" | unanswerable | complete |
| Distinct-but-same-shaped IDs | needs branded types | free |
| Test doubles | object literal | class, anonymous class, or mock |
| Where a mismatch surfaces | call site, compile time | definition site, compile time |

### Gotchas checklist

- [ ] Never use `Map<String, Object>` to model a shape. Declare a record.
- [ ] Never define an interface no reachable class can implement.
- [ ] Wrap distinct identifiers in distinct types — it is free here, unlike in TS.
- [ ] Always write `@Override`. It converts a silent typo into a compile error.
- [ ] Two same-shaped functional interfaces are not assignable; use a method reference.
- [ ] Any reflective binder (Jackson, JPA, `@ConfigurationProperties`) is structural
      matching with no compile check. Contract-test those boundaries.
- [ ] Vendor types belong in exactly one package. Verify with `grep`, not with intent.

---

## When would I use this at work?

**1. The first week on a Java codebase, reading an unfamiliar service.**
You open `OrderService` and see `import com.stripe.model.Charge`. That single import
tells you the vendor boundary was never drawn, and it predicts what the next migration
will cost. In a TypeScript codebase that signal does not exist in the same form,
because a structural interface would have hidden it. Learning to read `implements` and
imports as *architecture* is the fastest way to orient in an unfamiliar Java repo.

**2. Designing the contract for a new integration.**
Before you write the client, you write the domain type you wish the vendor returned.
Then the client's job is defined: produce that type. This inverts the order you
probably work in today, where the vendor's response type propagates outward because it
already fits everywhere structurally.

**3. Reviewing a pull request that adds a `String` parameter.**
`void refund(String orderId, String paymentId)` — two strings, same type, adjacent
positions. You ask for wrapper records. It takes the author ten minutes and removes an
entire class of silent bug that would otherwise be found by a finance reconciliation
three weeks later. This is the highest-value, lowest-effort thing from this topic.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and wrappers**: you need to be comfortable that `Long` and `long`
  are different types before "two identical shapes are different types" lands.

**This unlocks:**
- **03 — Access modifiers and packages**: nominal typing decides *whether* a type can
  be used; access modifiers decide *where*. Together they are your encapsulation
  boundary.
- **04 — Interfaces vs abstract classes**: now that `implements` is a commitment, the
  question of what a well-designed interface contains becomes real.
- **05 / 06 / 07 — Generics**: Java generics are nominal and invariant, and neither of
  those makes sense until this topic does. `List<String>` is not a `List<Object>` for
  reasons rooted here.
- **21 — Lambdas**: target typing, explained properly.
- **27 — Records**: nominal tuples. Two records with identical components are still
  different types, which is the whole reason they are useful as identifiers.
- **28 — Sealed types**: your discriminated unions, and the payoff for the `sealed
  interface PaymentResult` you wrote in Example 2.
- **39 — Injection styles**: `@Qualifier` exists because two beans can implement the
  same nominal interface, and Spring needs a tiebreak.
- **83 — Native image**: reflection-based structural matching is exactly what
  closed-world analysis cannot see, which is why it needs configuration files.

---

*Java baseline 21. Nothing in this topic changed between 21 and 25 — Java has been
nominally typed since 1.0 and there is no proposal to change that. The only moving part
is which types are cheap to declare: records (16) made wrapper types practical, and
Project Valhalla would eventually make them free. Neither changes the rule.*
