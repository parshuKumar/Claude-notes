# 04 — Interfaces vs Abstract Classes; default, static, private Methods; Diamond Resolution

## Phase: 1 — Core Language
## Category: FOUNDATION
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Two different things, both often called "a base type".

**An interface is a licence.** A driving licence says what you must be able to do:
start, steer, stop. It carries no engine, no fuel, no seat. You can hold as many
licences as you like — driving, forklift, first aid — and holding one says nothing
about who you are, only about what you can do.

**An abstract class is a half-built vehicle.** It already has a chassis, a fuel tank
with fuel in it, and three of the four wheels fitted. You inherit it and finish the
build. Because it is a physical thing you sit inside, **you can only sit in one at a
time.**

Carry these three refinements through the doc:

1. **A `default` method is a licence that comes with a courtesy car.** The licensing
   authority noticed that everyone was building the same gearbox, so they started
   including a basic one. You may still fit your own. The point was never "now licences
   have engines" — the point was that the authority could add a new requirement to the
   licence *without invalidating every existing licence holder*.
2. **A `static` method on an interface is the licensing office's front desk.** It
   belongs to the authority, not to any holder.
3. **The diamond problem is two licences that both came with a courtesy car.** You now
   have two cars in one parking space and somebody has to decide which one you drive.
   Java's answer is: it refuses to guess, and makes you say.

---

## The bridge from what you know

### What transfers cleanly

TypeScript has both concepts and you use both:

```ts
interface PaymentGateway {
  charge(idempotencyKey: string, amountMinor: number): Promise<PaymentResult>;
}

abstract class BaseGateway implements PaymentGateway {
  protected constructor(protected readonly client: HttpClient) {}

  abstract charge(k: string, a: number): Promise<PaymentResult>;

  protected buildHeaders(): Record<string, string> {   // shared implementation
    return { 'Idempotency-Key': crypto.randomUUID() };
  }
}
```

Java's version is nearly a transliteration:

```java
public interface PaymentGateway {
    PaymentResult charge(String idempotencyKey, long amountMinor);
}

public abstract class BaseGateway implements PaymentGateway {
    protected final HttpClient client;

    protected BaseGateway(HttpClient client) { this.client = client; }

    @Override public abstract PaymentResult charge(String idempotencyKey, long amountMinor);

    protected Map<String, String> buildHeaders() {
        return Map.of("Idempotency-Key", UUID.randomUUID().toString());
    }
}
```

**Verdict: HONEST ANALOGUE** for the basic shape. Single class inheritance, multiple
interface implementation, and abstract members work the same way in both languages.

### What does not transfer

| You know | Java | Verdict |
|---|---|---|
| `abstract class` with constructor + state | `abstract class` with constructor + state | **HONEST ANALOGUE** |
| TS `interface` — declarations only, no bodies | Java `interface` — **can carry method bodies** (`default`, `static`, `private`) | **PARTIAL** — the capability does not exist in TS at all. |
| Mixins via `applyMixins` / class expressions | `default` methods | **PARTIAL** — similar effect, entirely different mechanism, and Java's version cannot add state. |
| Structural conformance means "adding a method to an interface" is checked at every use site | Adding an abstract method to an interface **breaks every implementor**, potentially at runtime | **NO ANALOGUE.** In TS an interface has no runtime existence, so there is nothing to break at runtime. In Java, interfaces are real class files, method dispatch goes through them, and separately-compiled implementors can be left with a hole — which the JVM reports as `AbstractMethodError`. |
| — | **Diamond resolution rules** | **NO ANALOGUE.** TypeScript interfaces carry no implementations, so two supertypes can never supply competing bodies for the same method. Java can, and needs a written-down resolution order. |
| — | **`default` methods exist for binary compatibility** | **NO ANALOGUE.** TypeScript has no concept of binary compatibility: everything is recompiled from source, always. The entire motivation for `default` is a problem you have never had. |

### The part with no TypeScript equivalent at all

**NO TYPESCRIPT ANALOGUE.**

Diamond resolution cannot exist in TypeScript, and neither can the problem `default`
methods were invented to solve. Both absences have the same cause: **a TypeScript
interface has no runtime existence.** It is erased entirely, so it can carry no method
bodies, so two supertypes can never supply competing implementations of the same method —
there is nothing to compete. And because nothing is separately compiled and linked,
adding a member to an interface can never break already-running code; the worst it can do
is fail a type-check on the next build.

Java interfaces are real class files that real call sites link against. That is what makes
`default` bodies possible, what makes the diamond possible, and what makes
`AbstractMethodError` possible. All three are the same fact seen from three angles, and
none of them has a TypeScript shadow you can reason from.

### The one sentence that reframes everything

> `default` methods were **not** added so that interfaces could have behaviour. They
> were added so that `java.util.Collection` could gain a `stream()` method in Java 8
> without breaking every class on Earth that implemented `Collection`.

Hold that. It explains every design rule that follows — why `default` cannot access
state, why it cannot override `Object` methods, and why "put shared logic in a default
method" is usually the wrong instinct.

---

## What is this?

An **interface** declares a capability. It may contain abstract methods, `default`
methods (with a body, inheritable and overridable), `static` methods (not inherited),
`private` methods (helpers for the other two), and `public static final` constants. It
holds **no instance state** and has **no constructor**. A class may implement any number.

An **abstract class** is a partially-implemented class. It may have instance fields,
constructors, all four access levels, and abstract methods. A class may extend exactly
**one**.

**Diamond resolution** is the fixed set of rules the compiler applies when a class
inherits a method body from more than one place.

---

## Why does it matter?

1. **Adding a method to a published interface can break running code.** Not compile
   code — running code. A service compiled against version 1 of your interface, deployed
   against version 2, throws `AbstractMethodError` at first call. `default` exists to
   make that additive change safe. If you do not understand this, you will either be
   afraid to evolve interfaces at all or you will break a downstream team.

2. **Choosing an abstract class where an interface belonged permanently spends your one
   inheritance slot.** `OrderService extends BaseAuditedService` means it can never
   extend anything else — not a framework base class, not a test harness base. In a
   Spring codebase this collides with proxying (Topic 40) and with test infrastructure,
   and by the time you notice, twelve classes depend on the hierarchy.

3. **Diamond ambiguity is a compile error you must be able to resolve in thirty
   seconds.** It looks alarming the first time. The rules are three lines long. Not
   knowing them turns a trivial fix into an hour of trying random things.

---

## Syntax breakdown

Only what is genuinely new.

### Interface member kinds

```java
public interface PricingRule {

    // 1. abstract — no body. Implicitly public abstract.
    long apply(long amountMinor, PricingContext ctx);

    // 2. default — has a body. Inherited by implementors; may be overridden.
    default long applyAll(long amountMinor, List<PricingContext> contexts) {
        long running = amountMinor;
        for (PricingContext ctx : contexts) {
            running = apply(running, ctx);
        }
        return clampToZero(running);            // calls the private helper below
    }

    // 3. static — belongs to the interface itself. NOT inherited.
    static PricingRule identity() {
        return (amount, ctx) -> amount;
    }

    // 4. private — helper for default/static methods only. Java 9+.
    private static long clampToZero(long v) {
        return Math.max(0L, v);
    }

    // 5. constant — implicitly public static final.
    int MAX_DISCOUNT_PERCENT = 90;
}
```

| Bit of syntax | What it means |
|---|---|
| `long apply(...);` with no modifiers | Implicitly `public abstract`. You cannot make an interface method package-private or protected (except `private` for helpers). **Everything on an interface is part of your public surface** — connect this to Topic 03. |
| `default` | A method body that implementors inherit. It is *virtual*: an implementing class can override it and the override wins. |
| `static` on an interface | Not inherited. `PricingRule.identity()` works; `someRule.identity()` and `MyRule.identity()` do **not** compile. This differs from static methods on classes, which *are* inherited into subclasses. |
| `private` / `private static` | Java 9+. Lets `default` and `static` methods share code without exposing it. Before Java 9 you had to either duplicate or make the helper public. |
| `int MAX_DISCOUNT_PERCENT = 90;` | Implicitly `public static final`. There is no such thing as an interface instance field. Putting constants here is the "constant interface" antipattern — see Trap 3. |

### Abstract class members

```java
public abstract class BaseGateway implements PaymentGateway {

    private   final HttpClient client;      // instance state — impossible on an interface
    protected final RetryPolicy retries;    // protected — impossible on an interface

    protected BaseGateway(HttpClient client, RetryPolicy retries) {   // constructor
        this.client  = client;
        this.retries = retries;
    }

    protected abstract String vendorName();          // subclass must supply

    public final String describe() {                 // final: subclasses may NOT override
        return vendorName() + " via " + client.baseUrl();
    }
}
```

| Bit of syntax | What it means |
|---|---|
| `abstract class` | Cannot be instantiated. `new BaseGateway(...)` is a compile error. |
| `protected BaseGateway(...)` | Abstract classes have constructors. They run as part of subclass construction, via an implicit or explicit `super(...)`. Interfaces have none — that is the deepest difference between the two. |
| `protected abstract String vendorName();` | A hook. This is the template-method pattern: `describe()` is fixed, `vendorName()` is the variable part. |
| `public final String describe()` | `final` here means "this algorithm is not negotiable". Combined with an abstract hook it is how you publish behaviour without publishing the whole implementation. Note for later: `final` methods cannot be advised by a CGLIB proxy (Topic 40). |

### `Interface.super.method()` — the disambiguator

```java
public class DiscountedVatRule implements DiscountRule, VatRule {
    @Override
    public long apply(long amountMinor, PricingContext ctx) {
        long afterDiscount = DiscountRule.super.apply(amountMinor, ctx);
        return VatRule.super.apply(afterDiscount, ctx);
    }
}
```

| Bit of syntax | What it means |
|---|---|
| `DiscountRule.super.apply(...)` | "Call the `default` implementation from *this specific* direct superinterface." The only place this syntax is legal is inside a class or interface that directly implements/extends the named type. |
| Why it exists | It is the *only* way to reach a specific inherited default once two of them collide. `super.apply(...)` alone is ambiguous and will not compile. |

---

## The diamond resolution rules — exactly three

When a type inherits a method with a body from more than one place, the compiler applies
these in order:

**Rule 1 — the class always wins.**
A method inherited from a superclass (concrete or abstract) beats any `default` method
from any interface, even if the interface is "more specific". If your superclass has a
`toString()` and your interface has a `default toString()`... well, that one is
separately illegal (see below), but the principle holds for every other method.

**Rule 2 — the most specific interface wins.**
If `B extends A` and both declare a `default m()`, then a class implementing `B` (or
both `A` and `B`) gets `B`'s. Subinterface beats superinterface. This is the same
"more derived wins" intuition you already have.

**Rule 3 — otherwise it is an error, and you must choose.**
If two unrelated interfaces both supply a `default m()`, the compiler refuses:
```
error: class DiscountedVatRule inherits unrelated defaults for
  apply(long,PricingContext) from types DiscountRule and VatRule
```
You resolve it by overriding `m()` in your class and, if you want one of the inherited
bodies, calling it with `Interface.super.m()`.

Memorise those three lines. That is the whole of it.

### The extra rule people forget

**A `default` method may never override a method from `java.lang.Object`.**

```java
public interface Auditable {
    default String toString() { return "audit"; }   // does NOT compile
}
```
```
error: default method toString in interface Auditable overrides a member of
  java.lang.Object
```

Why: every object already has an `Object` implementation, and Rule 1 says the class
always wins — so the interface default could never be reached. Rather than let you write
dead code, the compiler forbids it. `equals`, `hashCode` and `toString` therefore cannot
be given interface defaults, which is why `Comparator` and friends *re-declare* `equals`
as abstract to document intent rather than to supply behaviour.

This is a reliable interview question and a genuinely useful piece of design reasoning.

---

## Example 1 — minimal

The diamond, in the smallest form that shows all three rules.

```java
public class Diamond {

    interface Fees {
        default long apply(long amountMinor) { return amountMinor + 199; }
    }

    interface Vat {
        default long apply(long amountMinor) { return amountMinor * 120 / 100; }
    }

    // Rule 3: two unrelated defaults -> compile error unless we choose.
    static class Both implements Fees, Vat {
        @Override public long apply(long amountMinor) {
            return Vat.super.apply(Fees.super.apply(amountMinor));   // fees, then VAT
        }
    }

    // Rule 2: PremiumVat is more specific than Vat, so it wins with no ambiguity.
    interface PremiumVat extends Vat {
        @Override default long apply(long amountMinor) { return amountMinor * 105 / 100; }
    }
    static class Premium implements Vat, PremiumVat { }     // no override needed

    // Rule 1: the class wins over any interface default.
    static class Base { public long apply(long amountMinor) { return amountMinor; } }
    static class ClassWins extends Base implements Vat { }  // Base.apply wins; no error

    public static void main(String[] args) {
        System.out.println(new Both().apply(1000));       // fees then VAT
        System.out.println(new Premium().apply(1000));    // PremiumVat wins
        System.out.println(new ClassWins().apply(1000));  // Base wins -> unchanged
    }
}
```

Run it. Then delete the `apply` override in `Both` and recompile — that error message is
one you should be able to recognise instantly.

Then, for the most instructive result: delete `Base`'s `apply` method body and make
`Base` abstract with `public abstract long apply(long);`. Does `ClassWins` still compile?
Reason it out before you try. (Rule 1 says the class wins — but an *abstract* class
method with no body leaves the class abstract. This is the edge people get wrong.)

---

## Example 2 — production scenario

`orderflow` publishes an internal library, `orderflow-payments-api`, consumed by three
services that are built and deployed independently. Today:

```java
package com.orderflow.payments;

public interface PaymentGateway {
    PaymentResult charge(String idempotencyKey, long amountMinor, String currency);
}
```

Three implementations exist across two repositories: `StripeGateway`, `BetaGateway`, and
a `StubGateway` in another team's test fixtures that you do not know about.

### The change finance asks for

"We need to be able to void an authorisation before capture."

The obvious move:

```java
public interface PaymentGateway {
    PaymentResult charge(String idempotencyKey, long amountMinor, String currency);
    void void_(String providerRef);          // new abstract method
}
```

### What actually happens

**In your repo:** compile errors everywhere, which you fix. Fine.

**In the other team's repo, which nobody rebuilt:** nothing. Their `StubGateway` still
compiles against the old jar. They pick up your new version transitively in their next
release. Then, at runtime, on the first request that reaches the stub:

```
java.lang.AbstractMethodError: Receiver class com.otherteam.StubGateway does not
  define or inherit an implementation of the resolved method 'void void_(java.lang.String)'
  of interface com.orderflow.payments.PaymentGateway
```

Green build. Green tests. Failure in production. **This is the exact problem `default`
was invented for**, and it is worth seeing once so it stops being abstract.

### The `default`-based evolution

```java
public interface PaymentGateway {

    PaymentResult charge(String idempotencyKey, long amountMinor, String currency);

    /**
     * Voids an uncaptured authorisation.
     * <p>Default: not supported. Providers that support voiding must override.
     * @throws UnsupportedOperationException if this gateway cannot void
     */
    default void voidAuthorisation(String providerRef) {
        throw new UnsupportedOperationException(
            getClass().getSimpleName() + " does not support voiding authorisations");
    }

    /** Capability probe so callers can branch without catching an exception. */
    default boolean supportsVoid() {
        return false;
    }
}
```

Now the other team's `StubGateway` still compiles, still runs, and if anyone calls
`voidAuthorisation` on it they get an exception whose message names the class — a
diagnosable failure at the right place, not an `AbstractMethodError` from the JVM's
linker.

`StripeGateway` overrides both:

```java
final class StripeGateway implements PaymentGateway {
    @Override public PaymentResult charge(...) { ... }
    @Override public void voidAuthorisation(String providerRef) { client.void_(providerRef); }
    @Override public boolean supportsVoid() { return true; }
}
```

And the caller stops guessing:

```java
if (gateway.supportsVoid()) {
    gateway.voidAuthorisation(ref);
} else {
    refunds.scheduleFullRefund(ref);      // slower path, but correct
}
```

### Be honest about what this default is

`throw new UnsupportedOperationException` is not a *behaviour*. It is a **migration
device**. You have deliberately traded a compile-time obligation for a runtime one, in
exchange for not breaking three services on a Friday.

That trade is correct here, and it has an expiry date. The follow-up work is:

1. Every implementor overrides it, or explicitly documents that it does not.
2. `supportsVoid()` exists so callers branch on a boolean rather than catching an
   exception for control flow.
3. Once every implementor is migrated, you *may* promote it to abstract — in a major
   version, with a deprecation cycle.

If you skip step 3 forever, you have a `Collection`-shaped interface where half the
methods throw at runtime. The JDK itself lives with exactly this
(`List.of(...).add(x)` throws `UnsupportedOperationException`) and it is widely
considered a design wart, not a pattern to copy.

### Why not an abstract class here?

You could put `voidAuthorisation` on an `AbstractPaymentGateway` and have everyone
extend it. Three reasons not to:

| Reason | Consequence |
|---|---|
| It spends the implementor's single inheritance slot | `StubGateway` may already extend a test base class. You have made adoption a refactor for them. |
| It does not help the class that already exists | An existing implementor that does not extend your base is still broken. `default` reaches every implementor automatically; a base class reaches only those who opt in. |
| It couples the API to an implementation | `orderflow-payments-api` is supposed to be interfaces and value types. A base class drags in fields, a constructor and a lifecycle. |

The rule of thumb that falls out:

> **Interface for the contract. `default` for evolving the contract. Abstract class only
> when you genuinely need shared instance state or a constructor.** If the shared thing
> is behaviour with no state, prefer a `static` helper or composition over an abstract
> class.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — adding an abstract method to a published interface

**Wrong:** the `void_(String)` change above, with no `default`.

**Exact symptom:**
```
java.lang.AbstractMethodError: Receiver class com.otherteam.StubGateway does not
  define or inherit an implementation of the resolved method
  'void void_(java.lang.String)' of interface com.orderflow.payments.PaymentGateway
```
at first call, in production, after a clean build.

**Root cause:** interfaces are real types with real dispatch. Separately compiled
implementors are not recompiled by your change, so the JVM discovers the missing
implementation at link time — which is the first time that method is invoked, not at
startup.

**Fix:** add it as a `default`, with a companion capability probe. Promote to abstract
only in a major version after every implementor has migrated. If the method genuinely
cannot have a sane default, that is a signal you are adding a *new* contract, not
evolving the old one — consider a separate interface.

---

### Trap 2 — the unresolved diamond

**Wrong:**
```java
class DiscountedVatRule implements DiscountRule, VatRule { }   // both have default apply()
```

**Exact symptom:**
```
error: class DiscountedVatRule inherits unrelated defaults for
  apply(long,PricingContext) from types DiscountRule and VatRule
```

**Root cause:** Rule 3. Two unrelated interfaces supplied competing bodies and Java
refuses to pick — because either choice would be a silent, arbitrary decision about your
business logic. Note that a language which *did* pick (C++ picks by declaration order in
some configurations) would have applied VAT before or after the discount without telling
you, which is a wrong invoice.

**Fix:** override and choose explicitly.
```java
@Override public long apply(long amountMinor, PricingContext ctx) {
    return VatRule.super.apply(DiscountRule.super.apply(amountMinor, ctx), ctx);
}
```
And notice what you just had to decide: discount *then* VAT. That ordering is a business
rule. The compile error forced you to write it down. This error is a feature.

---

### Trap 3 — state smuggled into an interface

**Wrong:**
```java
public interface RateLimited {
    Map<String, Integer> COUNTS = new ConcurrentHashMap<>();   // implicitly static final

    default boolean allow(String tenantId) {
        return COUNTS.merge(tenantId, 1, Integer::sum) < 100;
    }
}
```

**Exact symptom:** the limiter counts across **every implementor and every instance in
the JVM**, because `COUNTS` is a single static field. Tenant A's requests exhaust
Tenant B's budget. Observable as: a rate-limit rejection rate that correlates with total
platform traffic rather than per-tenant traffic, and support tickets from customers who
demonstrably sent 3 requests and got a 429.

Worse variants leak: a `static Map` keyed by `this` retains every implementor instance
forever — an unbounded static cache, which is Topic 79's drill.

**Root cause:** every field on an interface is implicitly `public static final`. There
is no such thing as per-instance interface state. `default` methods can therefore only
work with their parameters and with abstract methods they call back into.

**Fix:** if the behaviour needs state, it is not an interface concern. Use composition:
```java
public interface RateLimited {
    RateLimiter limiter();                                  // abstract: implementor supplies
    default boolean allow(String tenantId) { return limiter().tryAcquire(tenantId); }
}
```
The `default` method now delegates to state the implementor owns. This is the correct
shape for almost every "I want a default with state" situation.

---

### Trap 4 — calling an overridable method from a constructor

**Wrong:**
```java
public abstract class BaseGateway {
    private final String description;

    protected BaseGateway() {
        this.description = "gateway:" + vendorName();     // calls a subclass method
    }

    protected abstract String vendorName();
}

final class StripeGateway extends BaseGateway {
    private final String vendor = "stripe";               // assigned AFTER super() runs
    @Override protected String vendorName() { return vendor.toUpperCase(); }
}
```

**Exact symptom:**
```
java.lang.NullPointerException: Cannot invoke "java.lang.String.toUpperCase()"
  because "this.vendor" is null
	at com.orderflow.payments.StripeGateway.vendorName(StripeGateway.java:4)
	at com.orderflow.payments.BaseGateway.<init>(BaseGateway.java:6)
```
on a field the reader can see is definitely initialised, three lines below.

**Root cause:** Java runs the superclass constructor to completion *before* any subclass
field initialiser. During `BaseGateway.<init>`, `vendor` is still `null` (its default),
but virtual dispatch already resolves `vendorName()` to the subclass override. The
object is in a half-built state and the base class is calling into it.

Note the stack trace shape: `<init>` frames with a subclass method above a superclass
constructor. Once you recognise that shape, this diagnosis takes ten seconds.

**Fix:** do not call overridable methods from a constructor. Either make the hook a
constructor parameter:
```java
protected BaseGateway(String vendorName) { this.description = "gateway:" + vendorName; }
```
or compute lazily on first access, or make the method `final` so it cannot be overridden.

This bug has no TypeScript equivalent worth relying on — TS/JS field initialisers run in
a different order relative to `super()`, so the intuition you have does not transfer.
Treat it as new.

---

### Trap 5 — reaching for an abstract class to share three lines of logic

**Wrong:**
```java
public abstract class BaseOrderService {
    protected String correlationId() { return MDC.get("correlationId"); }
    protected void audit(String action) { log.info("audit action={} cid={}", action, correlationId()); }
}

public class OrderService  extends BaseOrderService { ... }
public class RefundService extends BaseOrderService { ... }
public class WalletService extends BaseOrderService { ... }
```

**Exact symptom:** eighteen months later you need `OrderService` to also extend a
framework base class, or a test wants a different `audit` implementation, and you cannot.
The concrete failure is usually a compile error at the point you try:
```
error: no interface expected here
```
or
```
error: class OrderService cannot extend more than one class
```
and the cost is a multi-day refactor across every subclass, because the base class's
`protected` members are now part of the contract with all three.

The subtler production symptom in Spring: `BaseOrderService` grows a
`@PostConstruct` or a `@Transactional` method, and the proxying rules (Topic 40) start
interacting with the hierarchy in ways nobody predicted — for example the base method
being `final` for safety silently makes it un-advisable, so `@Transactional` does
nothing and a rollback that should have happened does not.

**Root cause:** inheritance was used for code reuse rather than for an is-a
relationship. You spent an irreplaceable slot on three lines of logging.

**Fix:** composition, or a `default` method if it genuinely belongs to the contract.
```java
public final class AuditLog {           // a collaborator, injected
    public void record(String action) { ... }
}

public class OrderService {
    private final AuditLog audit;
    public OrderService(AuditLog audit) { this.audit = audit; }
}
```
You know this rule from TypeScript — "favour composition over inheritance". What is
different in Java is the *price* of getting it wrong: TS has structural typing and you
can usually escape; Java's single inheritance slot is genuinely irrecoverable without
touching every subclass.

---

## Hands-on proof

Every command below is one **you** run. No invented output.

### Setup

```bash
mkdir -p ~/java-lab/04 && cd ~/java-lab/04
java --version
```

### Proof 1 — the diamond error, and the fix

Save the `Diamond.java` from Example 1, but delete the `apply` override in `Both`.

```bash
javac Diamond.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `error: class Both inherits unrelated defaults for apply(long) from types Fees and Vat` | Rule 3, exactly. Two unrelated interfaces, competing bodies, compiler refuses. |
| No error | You made one interface extend the other, so Rule 2 resolved it silently. Check the declarations. |
| An error about `Fees.super` | You left the disambiguating call in a class that does not *directly* implement `Fees`. That syntax is only legal in a direct implementor. |

Restore the override and recompile. Then verify Rule 1 by adding a concrete `apply` to
`Base` and confirming `ClassWins` compiles with no override at all.

### Proof 2 — `default` methods live in the interface's class file, not in yours

```bash
javac Diamond.java
javap -p 'Diamond$Vat.class'
javap -p 'Diamond$Premium.class'
```

**What to look for:**

- In `Diamond$Vat`, a method `apply(long)` that is **not** abstract — it has a body.
- In `Diamond$Premium`, look for whether an `apply` method appears at all.

| What you see | What it means |
|---|---|
| `Vat` lists `public default long apply(long)` (or a non-abstract `apply`) | Confirmed: the body is compiled into the *interface* class file. This is what made `default` binary-compatible — implementors did not need recompiling. |
| `Premium` lists no `apply` of its own | Confirmed: the implementor inherits the body at link time; nothing was copied into the subclass. Contrast with a hypothetical "copy the code in" design, which would not have been binary compatible. |
| `Premium` *does* list an `apply` | You overrode it, or the compiler generated a bridge (Topic 06). Check whether the flags say `synthetic`. |

> If your `javap` output uses different wording than I describe, trust your output.
> Reading class files properly is Topic 76; here you are confirming one structural fact.

### Proof 3 — reproduce `AbstractMethodError` with separate compilation

This is the most valuable ten minutes in this doc. You are going to make a clean build
fail at runtime.

```bash
mkdir -p v1 v2 app out

cat > v1/PaymentGateway.java <<'EOF'
public interface PaymentGateway {
    String charge(long amountMinor);
}
EOF

cat > app/StubGateway.java <<'EOF'
public class StubGateway implements PaymentGateway {
    public String charge(long amountMinor) { return "STUB-OK"; }
}
EOF

cat > app/Main.java <<'EOF'
public class Main {
    public static void main(String[] args) {
        PaymentGateway g = new StubGateway();
        System.out.println(g.charge(1999));
        System.out.println(g.voidAuthorisation("REF-1"));
    }
}
EOF
```

Step 1 — compile the app against **v1** (remove the `voidAuthorisation` line from
`Main` for this step, then add it back after step 2):

```bash
javac -d out v1/PaymentGateway.java app/StubGateway.java app/Main.java
java -cp out Main
```

Step 2 — now write **v2** of the interface with a new *abstract* method, compile **only
the interface**, and overwrite it in `out`:

```bash
cat > v2/PaymentGateway.java <<'EOF'
public interface PaymentGateway {
    String charge(long amountMinor);
    String voidAuthorisation(String ref);     // NEW, abstract
}
EOF

javac -d out v2/PaymentGateway.java     # only the interface is recompiled
java -cp out Main                        # StubGateway.class is the OLD one
```

**What to look for:**

| What you see | What it means |
|---|---|
| `charge` still prints, then `java.lang.AbstractMethodError` naming `StubGateway` and `voidAuthorisation` | The target result. A clean compile, an untouched implementor, and a runtime linkage failure. This is the problem `default` solves. |
| `NoSuchMethodError` instead | Your `Main` was compiled against v1 and calls a method the *interface* no longer has — you changed the wrong side. Recheck which files you recompiled. |
| It works | `StubGateway.class` got recompiled. Make sure step 2 only compiles the interface. |
| Failure at startup rather than at the call | Depends on when the JVM resolves the call site. Both are possible; the JVM is permitted to resolve lazily. Note which yours did — that is real information about your JDK. |

Step 3 — repeat with `default String voidAuthorisation(String ref) { return "UNSUPPORTED"; }`
in v2 instead of an abstract method. Recompile only the interface. Run again.

**What to look for:** it should now run to completion, printing the default's value.
That single difference — abstract versus `default` — is the entire binary-compatibility
argument, demonstrated.

### Proof 4 — `default` cannot override `Object`

```bash
cat > DefaultObject.java <<'EOF'
public interface Auditable {
    default String toString() { return "audit"; }
}
EOF
javac DefaultObject.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `error: default method toString in interface Auditable overrides a member of java.lang.Object` | Expected. Rule 1 means the class always wins, so the default would be unreachable; the compiler forbids writing dead code. |
| It compiles | Impossible on JDK 21. Check you did not name the method something else. |

Now try `default boolean equals(Object o)` and `default int hashCode()`. Same error.
Then look at `java.util.Comparator` in your IDE and notice it *declares* `equals` as
abstract — that is documentation of a contract, not an implementation.

### Proof 5 — static interface methods are not inherited

```bash
cat > StaticNotInherited.java <<'EOF'
interface Rule {
    static Rule identity() { return null; }
}
class MyRule implements Rule { }

public class StaticNotInherited {
    public static void main(String[] args) {
        Rule.identity();       // OK
        MyRule.identity();     // expect a compile error
    }
}
EOF
javac StaticNotInherited.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `error: cannot find symbol ... symbol: method identity() location: class MyRule` | Expected. Interface statics belong to the interface only — unlike class statics, which *are* inherited by subclasses. |
| It compiles | You declared `identity()` on a class, not an interface. Recheck. |

This asymmetry is deliberate: it avoids a second, worse diamond problem for static
methods, where there would be no `Interface.super` escape hatch.

---

## Practice exercises

### 1 — Easy: prove all three resolution rules

Build a single file with three interfaces (`Fees`, `Vat`, `PremiumVat extends Vat`) and
a class `Base` with a concrete `apply`.

Write four classes, one per scenario, and for each: predict the outcome in a comment
first, then compile and record whether you were right.

1. `implements Fees, Vat` with no override → which rule, what error?
2. `implements Vat, PremiumVat` → which rule, which body runs?
3. `extends Base implements Vat` → which rule, which body runs?
4. `extends Base implements Vat` where `Base.apply` is `abstract` with no body → does it
   compile? Why is this the edge case?

Then add a `default toString()` to `Fees` and record the exact error.

### 2 — Medium: evolve an interface without breaking anyone (combines Topics 02 and 03)

You own `com.orderflow.payments.PaymentGateway`, published as a jar. Three implementors
exist, one of which is in a repo you cannot rebuild.

Add two capabilities: `voidAuthorisation(String ref)` and
`partialRefund(String ref, long amountMinor)`.

Requirements:
- Neither addition may break the un-rebuilt implementor at runtime. Prove it with the
  separate-compilation technique from Proof 3.
- Callers must be able to determine capability **without catching an exception for
  control flow**.
- Everything on an interface is `public` (Topic 03). Show that your design does not
  accidentally publish anything you would not want to support for five years.
- One of the two capabilities is arguably a *different contract* rather than an evolution
  of this one. Decide which, argue it, and if you agree, show the alternative design
  (a second interface) and say what it costs the caller.

Write the deprecation plan for eventually making them abstract: what has to be true, in
what order, and how you would know.

### 3 — Hard: production simulation — the pricing pipeline

`orderflow` needs a composable pricing pipeline. Rules include: tiered customer
discount, promotional code, VAT by country, and a payment-method surcharge. Rules must
be orderable, and finance must be able to see the order.

**Part A.** Model it. You have three candidate designs:
1. `interface PricingRule` + a list, applied in order.
2. `abstract class AbstractPricingRule` with a template method.
3. `default` methods on `PricingRule` providing composition helpers
   (`andThen`, `identity`) in the style of `java.util.function.Function`.

Implement design 1 fully. Then implement just enough of designs 2 and 3 to compare.

**Part B.** Now introduce the diamond deliberately. Create
`interface Discountable { default long apply(...) }` and
`interface Taxable { default long apply(...) }`, and a rule that is both. Capture the
compile error, resolve it with `Interface.super`, and then answer: **what business
decision did the compiler force you to make explicit?** Write that decision down as
you would in a design doc.

**Part C.** Ordering correctness. Apply VAT before discount and then after, on a
£100.00 order with a 10% discount and 20% VAT. Show both numbers. State which is
correct for your jurisdiction and where in the code that decision is now visible. (If it
is visible in only one place, your design is good. If it is visible in three, say why.)

**Part D.** Break it in production shape. Add a fifth rule type to the published
interface as an abstract method, and run the Proof 3 separate-compilation drill against
an implementor you did not rebuild. Capture the `AbstractMethodError`. Then fix it with
`default` and re-run.

**Part E.** Argue against yourself. Under what circumstances would design 2 (the abstract
class) actually be the right choice here? Give a concrete condition — "when the rules
share state that must be constructed once" is a start, but be specific about *what*
state and *why* composition would not do.

---

## Interview questions

### Q1 — "Interfaces can have default methods now. Why were they added?"

**Mid-level answer:** "So interfaces can have implementations, like abstract classes.
It lets you share code between implementors."

**Senior answer:** "For interface evolution, specifically binary compatibility.
`Collection` needed to gain `stream()` in Java 8, and adding an abstract method to an
interface that thousands of classes implement would break every one of them — not at
compile time for anyone who rebuilds, but at *runtime* with `AbstractMethodError` for
anyone who doesn't. A `default` method compiles into the interface's own class file, so
existing implementors keep working untouched. The 'multiple inheritance of behaviour'
framing is a side effect, and treating it as the purpose leads people to put logic in
interfaces that should be in a collaborator — because a `default` method has no access
to instance state, which is the constraint that tells you what it is really for."

**What separates them:** naming the actual motivation, naming the exact runtime error
that motivated it, and drawing the design consequence from the no-state constraint.

**Follow-up:** "So could they have just copied the default body into every implementing
class at compile time?" That would work for source compatibility and fail for binary
compatibility — the whole point was to not recompile implementors.

---

### Q2 — "A class inherits the same default method from two interfaces. What happens?"

**Mid-level answer:** "There's a conflict — I think you have to override it."

**Senior answer:** "Three rules, in order. One: a method from the class hierarchy always
beats any interface default, even a more specific one. Two: between interfaces, the most
specific wins — a subinterface beats its superinterface. Three: two *unrelated*
interfaces both supplying a body is a compile error,
`inherits unrelated defaults for m() from types A and B`, and you resolve it by
overriding and optionally delegating with `A.super.m()`. That error is a feature: with
pricing rules, letting the language silently pick would decide whether VAT applies before
or after a discount, which is a wrong invoice. And one extra rule people forget: a
`default` can never override an `Object` method, because Rule 1 would make it
unreachable, so the compiler rejects it outright."

**What separates them:** all three rules in order, the `Object` exception, and — the
real differentiator — explaining *why* Java refuses to pick rather than treating the
error as an inconvenience.

**Follow-up:** "Write the disambiguating syntax." They want `A.super.m()` and, ideally,
that it is only legal in a *direct* implementor.

---

### Q3 — "Interface or abstract class — how do you choose?"

**Mid-level answer:** "Interface if you need multiple inheritance, abstract class if you
have shared code."

**Senior answer:** "I start from what I need to *carry*. Instance state or a constructor
means abstract class — those are the two things an interface genuinely cannot do. Shared
behaviour with no state is not a reason: that is a `default` method or, better, a
collaborator. And the cost of the abstract class is real — it spends the implementor's
single inheritance slot forever, which is a decision I am making on behalf of people who
haven't written their class yet. In a Spring codebase there is a second cost: base
classes interact with proxying, and a `final` method in a base class silently becomes
un-advisable, so `@Transactional` on it does nothing. My default is: interface for the
contract, `default` for evolving it, composition for shared behaviour, abstract class
only when there is state that must be constructed."

**What separates them:** identifying state and constructors as the *only* genuine
discriminators, pricing the inheritance slot as a cost imposed on others, and knowing
the framework interaction.

**Follow-up:** "When is the base class genuinely right?" A good answer names the template
method pattern where the algorithm is fixed and only hooks vary, and notes that even
then, a `final` public method plus `protected abstract` hooks is what makes it safe.

---

### Q4 — "Why does this NPE?"

```java
abstract class Base {
    Base() { this.label = "base:" + name(); }
    final String label;
    protected abstract String name();
}
class Impl extends Base {
    private final String vendor = "stripe";
    @Override protected String name() { return vendor.toUpperCase(); }
}
```

**Mid-level answer:** "Something's null. Maybe `vendor` isn't set yet?"

**Senior answer:** "`vendor` is null during `Base`'s constructor. Java runs the
superclass constructor to completion before any subclass field initialiser or the
subclass constructor body — but virtual dispatch is already live, so `name()` resolves to
`Impl`'s override and reads a field that still holds its default. The stack trace gives
it away: a subclass method frame sitting directly above a superclass `<init>` frame. The
rule is never call an overridable method from a constructor. The fixes are to pass the
value as a constructor parameter, compute it lazily on first access, or make the method
`final`. This one is worth calling out to anyone coming from TypeScript, because JS field
initialisation ordering relative to `super()` is different enough that the intuition
doesn't transfer."

**What separates them:** naming the initialisation order precisely, naming the stack-trace
*shape* as the diagnostic, and giving three fixes with different trade-offs.

**Follow-up:** "Does making the field non-final change anything?" No — the ordering is the
issue, not finality. They are checking whether you actually understand the mechanism.

---

### Q5 — "Your interface has ten `default` methods. Is that a smell?"

**Mid-level answer:** "Probably — interfaces should be small."

**Senior answer:** "It depends what they are. Composition combinators are fine —
`Function.andThen`, `Comparator.thenComparing`, `Predicate.negate` are all defaults and
they belong there, because they are derived operations expressible purely in terms of the
abstract method. What is a smell is defaults that need state, or defaults that throw
`UnsupportedOperationException` as a permanent design rather than a migration step. The
first is impossible to do correctly — interface fields are `public static final`, so
'state' means a shared static, which in a multi-tenant service means one tenant's counter
affecting another's. The second is what `List.of(...).add(x)` does, and it is widely
regarded as a wart: the type says you can add, the runtime says you cannot, so the
compiler can no longer help. I would ask which category each of the ten falls into
before calling it."

**What separates them:** distinguishing derived operations from smuggled state, naming
the JDK's own wart honestly rather than treating the JDK as automatically exemplary, and
declining to answer without more information.

**Follow-up:** "How would you turn an `UnsupportedOperationException` default into
something the compiler can check?" They want capability probes, a split interface, or
sealed types with exhaustive matching (Topic 28).

---

## Mental model checkpoint

1. `default` methods gave interfaces bodies but not state. If Java had also allowed
   instance fields on interfaces, exactly which of the three diamond rules would have had
   to change, and what new runtime problem would you have created?

2. Rule 1 says "the class always wins", even when the interface default is more specific
   and more recently written. Construct a case where that rule gives you the *wrong*
   behaviour, then argue whether the rule is still correct.

3. Static methods on interfaces are not inherited; static methods on classes are. Give
   the design reason. (Hint: what would `MyRule.identity()` mean if `MyRule` implemented
   two interfaces that both had `identity()`?)

4. `AbstractMethodError` happens at link time, not at class-load time. Why does that
   distinction matter operationally — what does it change about how you would find this
   in a running service?

5. TypeScript has no diamond problem. Is that because it is structurally typed, or
   because its interfaces carry no implementations? Which of the two is the load-bearing
   reason, and what would break if TS added default implementations tomorrow?

6. You have an abstract class with one abstract method and no state. Convert it to an
   interface. What did you gain, what did you lose, and is there any observable
   difference at runtime?

7. A `default` method that throws `UnsupportedOperationException` moves an obligation
   from compile time to runtime. Name another place in Java where the language makes the
   same trade deliberately, and say whether you think it was the right call there.

---

## Quick reference card

### What each can hold

| | `interface` | `abstract class` |
|---|---|---|
| Abstract methods | yes | yes |
| Method bodies | yes (`default`, `static`, `private`) | yes |
| Instance fields | **no** | yes |
| Constructors | **no** | yes |
| Constants | yes (implicitly `public static final`) | yes |
| Access levels on members | `public` (plus `private` helpers) | all four |
| How many can a class have | many | **one** |
| Can override `Object` methods | **no** | yes |
| Can be a lambda target | yes, if exactly one abstract method | no |

### Syntax

```java
interface R {
    long apply(long v);                                   // abstract, implicitly public
    default long twice(long v) { return apply(apply(v)); } // inheritable body
    static  R identity() { return v -> v; }               // NOT inherited
    private static long clamp(long v) { return Math.max(0, v); }  // Java 9+
    int MAX = 90;                                         // public static final
}

class C implements A, B {
    @Override public long apply(long v) {
        return B.super.apply(A.super.apply(v));            // disambiguate
    }
}
```

### Diamond resolution, in order

1. **Class wins** over any interface default.
2. **Most specific interface wins** (subinterface beats superinterface).
3. **Otherwise: compile error.** Override and use `Interface.super.method()`.
4. **Never**: a `default` may not override `equals`, `hashCode` or `toString`.

### Gotchas checklist

- [ ] Adding an **abstract** method to a published interface is a binary break →
      `AbstractMethodError` at runtime, clean compile.
- [ ] `default` is for interface *evolution*, not for sharing logic.
- [ ] Interface fields are `public static final`. There is no instance state, ever.
- [ ] Interface `static` methods are not inherited; class `static` methods are.
- [ ] `default` cannot override `Object` methods. Compile error, by design.
- [ ] `Interface.super.m()` is legal only in a direct implementor.
- [ ] Never call an overridable method from a constructor — subclass fields are still
      at their defaults.
- [ ] An abstract class spends the implementor's single inheritance slot. Permanently.
- [ ] Every interface member is public — that is a Topic 03 support obligation.
- [ ] `final` methods cannot be advised by a CGLIB proxy (Topic 40). Choose knowingly.

---

## When would I use this at work?

**1. Adding a method to an interface other teams implement.**
This is the single most common place this topic pays. You will reach for the abstract
method, remember `AbstractMethodError`, add a `default` plus a capability probe, and
write the deprecation plan in the PR description. That turns a potential incident into a
paragraph.

**2. Reviewing a new `abstract class` in a PR.**
One question: "what instance state or constructor does this need?" If the answer is
"none", it should be an interface or a collaborator, and you have just saved the next
person a multi-day refactor. If the answer is real, ask whether the public methods are
`final` and whether the hooks are documented — because those hooks are an API for
subclasses.

**3. Debugging a `NullPointerException` in an `<init>` frame.**
You see a subclass method above a superclass constructor in the stack trace and you know
the answer before reading the code. This shape appears in Spring codebases with base
service classes surprisingly often.

---

## Connected topics

**Prerequisites:**
- **02 — Nominal vs structural typing**: `implements` is a declaration, which is why
  adding to an interface is a contract change rather than a shape change.
- **03 — Access modifiers**: everything on an interface is public, and `final` methods on
  abstract classes are an access-adjacent decision with proxying consequences.

**This unlocks:**
- **05/06/07 — Generics**: generic interfaces and bounded type parameters build directly
  on this; bridge methods (Topic 06) are generated precisely at the interface/implementor
  seam you just studied.
- **21 — Lambdas and functional interfaces**: a lambda target is exactly "an interface
  with one abstract method" — `default` and `static` members do not count, and now you
  know why.
- **26 — Optional** and **23/24 — Streams**: `Stream` and `Optional` are built almost
  entirely from `default` combinators. You will read them differently now.
- **27/28 — Records and sealed types**: the modern alternative to an abstract-class
  hierarchy. `sealed interface` + records is usually a better answer than
  `abstract class` + subclasses.
- **40 — Proxying**: interface-based beans get JDK dynamic proxies; class-based get
  CGLIB subclasses, which cannot override `final` methods. Your choice here determines
  which.
- **47 — Spring Data JPA**: repository interfaces with `default` methods are a real,
  common pattern — and one that behaves differently from a generated query method.

---

*Java baseline 21. `default` and `static` interface methods arrived in Java 8;
`private` interface methods in Java 9; `sealed` interfaces in Java 17. Nothing in this
topic changed between 21 and 25. The resolution rules have been stable since Java 8 and
there is no proposal to change them.*
