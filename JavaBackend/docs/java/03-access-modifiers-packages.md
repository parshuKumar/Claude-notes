# 03 — Access Modifiers, Packages, and Encapsulation Boundaries

## Phase: 1 — Core Language
## Category: FOUNDATION
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Picture an office building.

- **`public`** is the street outside. Anyone can walk there. Once you put a bench on
  the street, removing it is a public event and someone will complain.
- **`protected`** is the family-and-staff area. Your own department can use it, and so
  can anyone who has *inherited* your job — but a stranger from another floor cannot,
  even if they can see the door.
- **package-private** (no keyword at all) is **your floor**. Everyone whose desk is on
  this floor can use it. Nobody on another floor can — and, importantly, nobody on the
  floor *above* or *below*, even though the sign on the stairwell says they are part of
  the same company.
- **`private`** is your desk drawer. Only you.

Three things to hold onto, because they are where the analogy earns its keep:

1. **Floors are not nested.** `com.orderflow.orders` and `com.orderflow.orders.internal`
   are two different floors. The name looks nested. The access rules are not. This is
   the single most common surprise for someone arriving from folder-based module systems.
2. **The floor door has no lock.** Anyone can print a badge that says they work on your
   floor — by declaring a class in your package name. Package-private is a *convention
   enforced by the compiler*, not a security boundary.
3. **Everything you put on the street is a support obligation.** Not a style point. A
   contract you have to keep, or break loudly.

---

## The bridge from what you know

### What you do today in Node/TypeScript

Your encapsulation boundary is the **module** — one file, and what it `export`s.

```ts
// src/payments/stripe-client.ts
const SECRET = process.env.STRIPE_KEY;    // not exported: genuinely unreachable

export class StripeClient {
  private retries = 3;                    // compile-time only
  #realSecret = SECRET;                   // truly private at runtime (ES private field)

  charge(amountMinor: number) { ... }
}
```

Plus a package-level boundary in `package.json`:

```json
{ "exports": { ".": "./dist/index.js", "./testing": "./dist/testing.js" } }
```

Anything not listed there cannot be imported by a consumer, even though it exists on
disk.

### The mapping, honestly

| You know | Java | Verdict |
|---|---|---|
| Not exporting a `const` from a module | `private` field | **PARTIAL** — a Node non-export is genuinely unreachable; Java `private` is reachable by reflection and by any other code in the same *top-level class*. |
| TS `private` | Java `private` | **PARTIAL** — TS `private` is erased at compile time and `obj['x']` reads it anyway. Java `private` is enforced by the JVM at link time. |
| TS `#private` field | Java `private` field | **HONEST ANALOGUE** — both are enforced by the runtime, both are per-instance. |
| TS `protected` | Java `protected` | **PARTIAL** — Java's `protected` *also* grants access to the whole package, and adds a subtle rule about which reference you access it through (below). |
| `package.json` `"exports"` | JPMS `exports` in `module-info.java` | **HONEST ANALOGUE** — both say "these paths are the published surface, the rest is mine". Topic 20. |
| A folder of files, imported freely between them | A Java package | **PARTIAL** — the folder maps to the package, but subfolders are unrelated packages, and there is no "the whole folder tree can see each other". |
| NestJS module `exports: [...]` array | — | **NO ANALOGUE.** A Nest module controls *which providers other modules can inject*. Java's access modifiers are about which code can *name a type or member*, at compile time. Spring's container has no equivalent of Nest's export list — any bean is injectable by any other bean if it is on the classpath. The nearest thing is Topic 36's component scanning, and it is weaker, not equivalent. |
| Barrel file `index.ts` re-exporting a public surface | An `api` package plus package-private impl classes | **PARTIAL** — a barrel is a convention nothing enforces; package-private is enforced by `javac`. |

### The part with no Node equivalent at all

**NO TYPESCRIPT ANALOGUE.**

`public` in Java is a *binary* compatibility promise, and Node has no concept of binary
compatibility. Your consumers `require` or `import` source (or transpiled source) and
everything is re-linked from scratch on every start; there is no separately compiled
artefact holding a fixed method descriptor that can go stale. In Java, a service compiled
against version 1 of your class and deployed against version 2 fails at *link time* —
`NoSuchMethodError`, `NoClassDefFoundError`, `AbstractMethodError` — with a clean build
and a green CI behind it.

That failure mode is what turns "which access modifier?" from a tidiness question into a
release-engineering one, and it is the single most important thing to internalise from
this topic. There is nothing in your Node experience that will generate the instinct for
you.

### The one genuinely new idea

> **In Java, the compilation unit and the encapsulation unit are different things.**

In Node, one file is one module is one boundary. In Java, one file is one *class* (or
close to it), and the boundary is the **package** — a set of files that agreed on a
name. That is why Java has a level of access you have no equivalent for:
package-private. It exists so that a group of collaborating classes can be intimate
with each other while presenting one small face to the outside.

That is exactly the thing your `index.ts` barrel file is trying to do, except the
compiler enforces it.

---

## What is this?

Java has **four** access levels: `public`, `protected`, package-private (written by
writing nothing), and `private`. They control which code may *name* a type, field,
method or constructor.

A **package** is a namespace declared by the `package` statement at the top of a file.
It is the unit that package-private access is scoped to.

Together they define your **encapsulation boundary**: the line between what you have
promised to keep working and what you are free to change on a Tuesday afternoon.

---

## Why does it matter?

1. **`public` is a promise you cannot quietly withdraw.** In a monolith, making a
   helper `public` means any of two hundred files may start calling it, and you will
   not know until you try to change it and the build breaks in six modules. In a
   published library it is worse: a downstream service compiled against your old
   signature and deployed independently fails at *runtime* with
   `NoSuchMethodError` — a clean compile, a broken deploy.

2. **Getting `protected` wrong leaks mutable state into subclasses you do not
   control.** A `protected` field on a base entity means every subclass, in every
   package, in every future team's code, can write to it without going through your
   validation. Your invariants are gone and there is no single place to fix them.

3. **Believing package-private is a security boundary gets you a finding in a
   pentest.** It is not. Anyone can declare `package com.orderflow.payments;` in their
   own source tree and read your package-private members. Only JPMS (Topic 20) or a
   sealed jar changes that, and almost nobody deploys either.

---

## Syntax breakdown

Only the genuinely new constructs.

### The `package` declaration

```java
package com.orderflow.payments.stripe;
```

| Bit of syntax | What it means |
|---|---|
| Must be the **first** statement in the file (comments aside). | One package declaration per file, no exceptions. |
| The dots are **not** hierarchy for access purposes. | `com.orderflow.payments` cannot see package-private members of `com.orderflow.payments.stripe`, and vice versa. They are two flat, unrelated namespaces that happen to share a prefix. |
| The directory layout must match. | `src/main/java/com/orderflow/payments/stripe/StripeGateway.java`. `javac` and every build tool assume this. |
| No `package` statement at all | Puts the class in the **unnamed package**. Never do this outside a scratch file — unnamed-package classes cannot be imported by anything. |

### The four levels, written out

```java
package com.orderflow.orders;

public class Order {                       // visible everywhere
    public    long id;                     // street
    protected long version;                // subclasses (any package) + this package
              long internalSeq;            // <- package-private: NO keyword
    private   long checksum;               // this top-level class only
}
```

| Bit of syntax | What it means |
|---|---|
| Writing **no** modifier | This is a real, deliberate access level, not an omission. Java people call it "default" or "package-private". Prefer saying package-private — "default" makes it sound accidental. |
| `protected` | Package **plus** subclasses in other packages. It is strictly *wider* than package-private, which surprises people who read the list as an ordering of "increasing privacy". |
| `private` on a **nested** class member | Accessible from anywhere in the same *top-level* class, including sibling nested classes. `private` is scoped to the outermost class, not to the innermost one. |

### Top-level types can only be two of the four

```java
public class Order { }      // legal
       class OrderDraft { } // legal — package-private
// protected class X { }    // does NOT compile
// private   class Y { }    // does NOT compile
```

A file may contain **at most one** `public` top-level type, and its name must match the
filename. Other top-level types in the same file are package-private. That is a
lightweight way to keep a helper genuinely local.

### The subtle `protected` rule

This one is genuinely new and is a favourite interview question.

```java
package com.orderflow.orders;
public class Order {
    protected long version;
}
```

```java
package com.orderflow.subscriptions;
import com.orderflow.orders.Order;

public class Subscription extends Order {

    void bump(Order other, Subscription sibling) {
        this.version++;      // OK   — through my own type
        sibling.version++;   // OK   — through Subscription (my own type or a subtype)
        other.version++;     // does NOT compile
    }
}
```

The error is:
```
error: version has protected access in com.orderflow.orders.Order
```

**The rule:** from a subclass in a *different* package, you may access a `protected`
member only through a reference whose static type is your own class or a subclass of
it. You inherited the right to touch *your own* copy, not everybody's.

Inside the *same* package the restriction does not apply, because package access
already covers it.

### `sealed` — a different axis, mentioned so you do not confuse them `[JAVA 21]`

```java
public sealed interface PaymentResult
        permits PaymentResult.Captured, PaymentResult.Declined { }
```

`sealed` restricts **who may extend/implement**, not **who may see**. A `public sealed`
type is fully visible to everyone and extensible by nobody outside the `permits` list.
It is orthogonal to access modifiers. Full treatment in Topic 28.

---

## Example 1 — minimal

Two packages, one member, four outcomes.

```java
// file: src/com/orderflow/orders/Order.java
package com.orderflow.orders;

public class Order {
    public    String publicRef    = "PUB";
    protected String protectedRef = "PRO";
              String packageRef   = "PKG";
    private   String privateRef   = "PRI";

    // Same top-level class, so even private is reachable here.
    public String describe() {
        return publicRef + protectedRef + packageRef + privateRef;
    }
}
```

```java
// file: src/com/orderflow/orders/SameFloor.java
package com.orderflow.orders;              // SAME package

public class SameFloor {
    void read(Order o) {
        System.out.println(o.publicRef);     // OK
        System.out.println(o.protectedRef);  // OK  (package access covers it)
        System.out.println(o.packageRef);    // OK
        // System.out.println(o.privateRef); // does NOT compile
    }
}
```

```java
// file: src/com/orderflow/reporting/OtherFloor.java
package com.orderflow.reporting;           // DIFFERENT package
import com.orderflow.orders.Order;

public class OtherFloor {
    void read(Order o) {
        System.out.println(o.publicRef);       // OK
        // System.out.println(o.protectedRef); // does NOT compile
        // System.out.println(o.packageRef);   // does NOT compile
        // System.out.println(o.privateRef);   // does NOT compile
    }
}
```

Uncomment each line one at a time and read the error. Four levels, four different
messages, ten minutes. Do it once and you will never guess again.

---

## Example 2 — production scenario

`orderflow` is growing. The payments module has become five classes: a gateway
interface, two vendor adapters, a retry helper and a decline-code translator. Another
team is starting to call into payments.

### The version that ships and then hardens into concrete

```java
package com.orderflow.payments;

public class StripePaymentGateway { ... }
public class RetryPolicy         { ... }
public class DeclineCodeMapper   { ... }
public class PaymentGatewayImpl  { ... }
```

Everything `public`, because that is what the IDE generates and nobody thought about it.

Six weeks later the orders team writes:

```java
// in com.orderflow.orders
DeclineCodeMapper mapper = new DeclineCodeMapper();
if (mapper.isRetryable(code)) { ... }
```

They found it in autocomplete. It compiled. It works.

**What this costs you, concretely.** You now want to change `isRetryable(String)` to
`isRetryable(DeclineReason)` because raw provider strings were a mistake. That is a
one-line change in your own head and a cross-team change in reality:

- The compiler flags it, so you find the orders-team call site. Good — in a monolith.
- If payments is a **separately built jar**, the orders service compiles against the
  old jar and only fails when the new one is deployed:
  ```
  java.lang.NoSuchMethodError: 'boolean
    com.orderflow.payments.DeclineCodeMapper.isRetryable(java.lang.String)'
  ```
  A clean build, a green CI, and a failure at first request in production. This is the
  exact failure shape Topic 32 will show you again from a different direction.

### The version with a deliberate boundary

Split into two packages: a published face, and an implementation floor.

```
com/orderflow/payments/            <- the published API. Small on purpose.
    PaymentGateway.java            public interface
    PaymentResult.java             public sealed interface + records
    DeclineReason.java             public enum
    PaymentsModule.java            public factory — the only way in

com/orderflow/payments/internal/   <- the implementation floor
    StripePaymentGateway.java      package-private class
    BetaPaymentGateway.java        package-private class
    RetryPolicy.java               package-private class
    DeclineCodeMapper.java         package-private class
```

```java
package com.orderflow.payments;

/** The published surface. Everything else in this module is ours to change. */
public interface PaymentGateway {
    PaymentResult charge(String idempotencyKey, long amountMinor, String currency);
}
```

```java
package com.orderflow.payments.internal;

import com.orderflow.payments.PaymentGateway;
import com.orderflow.payments.PaymentResult;

// No "public". This class does not exist as far as the rest of orderflow is concerned.
final class StripePaymentGateway implements PaymentGateway {

    private final StripeClient client;
    private final RetryPolicy retries;          // also package-private, also invisible outside

    StripePaymentGateway(StripeClient client, RetryPolicy retries) {
        this.client  = client;
        this.retries = retries;
    }

    @Override
    public PaymentResult charge(String idempotencyKey, long amountMinor, String currency) {
        ...
    }
}
```

Now, the only way in:

```java
package com.orderflow.payments.internal;

import com.orderflow.payments.PaymentGateway;

/** Public factory living on the implementation floor, exposing only the interface. */
public final class PaymentGatewayFactory {

    private PaymentGatewayFactory() { }        // no instances; this is a namespace

    public static PaymentGateway stripe(StripeClient client) {
        return new StripePaymentGateway(client, RetryPolicy.exponential(3));
    }
}
```

Note the shape: `StripePaymentGateway` is package-private, so nobody outside
`...payments.internal` can name it, hold it, subclass it, or mock it by type. But
`PaymentGatewayFactory` is public and hands back a `PaymentGateway`. Callers get
capability without identity.

### The awkward bit — say it out loud

The factory has to be `public` and it lives in a package called `internal`, which is
public in the JVM's eyes. **Any code anywhere can write
`import com.orderflow.payments.internal.PaymentGatewayFactory;`.** The word "internal"
is a convention, honoured by humans, ArchUnit tests, and nothing else.

Options, in ascending order of cost:

| Mechanism | Enforcement | Cost |
|---|---|---|
| Naming convention (`.internal`) | none — documentation only | free |
| Package-private classes | `javac`, for the *types*; the package itself is still open | free |
| ArchUnit test in CI ("no class outside `payments` may depend on `payments.internal`") | build failure, on every PR | one test file; **this is what most good Java teams actually do** |
| Separate Maven module | build failure, and a real artifact boundary | multi-module build (Topic 31) |
| JPMS `exports com.orderflow.payments;` and not `.internal` | `javac` **and** the JVM at run time | all-or-nothing across your dependency graph (Topic 20) |

The honest recommendation for a service like `orderflow`: package-private classes plus
an ArchUnit rule. It is 95% of the benefit for 5% of the cost, and it fails in CI where
you want failures to happen.

### What the boundary bought

Your public surface went from four classes and roughly thirty methods to one interface,
one sealed result type and one enum. Everything else is now yours to rewrite without a
conversation. That is not tidiness — it is the difference between shipping a change on
Tuesday and negotiating it for a sprint.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `public` by default because the IDE offered it

**Wrong:**
```java
public class DeclineCodeMapper {
    public boolean isRetryable(String providerCode) { ... }
}
```

**Exact symptom (in-repo):** you change the signature and get compile errors in four
modules you have never opened, one of which is owned by a team in another timezone. The
one-line refactor becomes a two-week coordination.

**Exact symptom (separate artifacts):**
```
java.lang.NoSuchMethodError: 'boolean
  com.orderflow.payments.DeclineCodeMapper.isRetryable(java.lang.String)'
	at com.orderflow.orders.OrderService.handleDecline(OrderService.java:88)
```
Green build, green tests, failure at first production request after deploy. This is
*binary* incompatibility, and it is invisible to `javac`.

**Root cause:** `public` is a compatibility promise. You made one without deciding to.

**Fix:** default to the narrowest level that compiles and widen deliberately. In
review, treat a new `public` member as a change that needs a reason, the same way you
would treat a new database column.

---

### Trap 2 — assuming a subpackage is "inside" its parent

**Wrong:**
```java
package com.orderflow.payments;

public class PaymentService {
    void run(com.orderflow.payments.internal.RetryPolicy p) {   // package-private class
        p.nextDelayMillis(1);
    }
}
```

**Exact symptom:**
```
error: RetryPolicy is not public in com.orderflow.payments.internal;
  cannot be accessed from outside package
```

**Root cause:** package names are dotted strings, not a tree. `com.orderflow.payments`
and `com.orderflow.payments.internal` are as unrelated, access-wise, as
`com.orderflow.payments` and `java.util`. Coming from a filesystem-based module system
this is genuinely counter-intuitive, and it is worth over-learning: **there is no such
thing as a subpackage, access-wise.**

**Fix:** decide which side of the boundary the type lives on, and put it there. If two
packages need intimate access to each other, they are one package. Do not fight this by
making things `public`.

---

### Trap 3 — `protected` mutable state on a base entity

**Wrong:**
```java
package com.orderflow.orders;

public abstract class AuditableEntity {
    protected Instant updatedAt;          // subclasses can write it directly
    protected String  updatedBy;
}
```

**Exact symptom:** a subclass in another package sets `updatedAt` directly and skips
your `touch()` method, which also stamps `updatedBy`. Under Hibernate this is worse
than it sounds — the field is now dirty, so a flush emits an `UPDATE` you cannot
attribute:
```
Hibernate: update orders set updated_at=?, updated_by=?, version=? where id=? and version=?
```
with `updated_by` null. Your audit trail has holes, and the compliance report that
depends on it fails a quarterly review. Nothing threw. Nothing logged.

**Root cause:** `protected` on a *field* publishes your representation. Every subclass —
including ones written next year by people who never read your class — becomes a place
your invariant can be broken.

**Fix:** `private` fields, `protected` behaviour.
```java
public abstract class AuditableEntity {
    private Instant updatedAt;
    private String  updatedBy;

    protected void touch(String actor) {      // one place, invariant intact
        this.updatedAt = Instant.now();
        this.updatedBy = Objects.requireNonNull(actor, "actor");
    }

    public Instant updatedAt() { return updatedAt; }
}
```
Rule of thumb: **`protected` is for methods you intend subclasses to call or override.
It is almost never right on a field.**

---

### Trap 4 — widening access "just for the test"

**Wrong:**
```java
public class OrderTotalCalculator {
    // was private; made public so OrderTotalCalculatorTest could call it
    public long applyTieredDiscount(long subtotalMinor, CustomerTier tier) { ... }
}
```

**Exact symptom:** eighteen months later, `applyTieredDiscount` is called from three
production classes that should have gone through `total()`. Two of them forget to apply
VAT afterwards. The observable failure is a **finance reconciliation break**: order
totals in the ledger disagree with invoiced amounts for a subset of customers, and the
discrepancy is proportional to the VAT rate. Nobody can find it from a stack trace
because there is no exception — the numbers are just wrong.

**Root cause:** test convenience became a production API. Nothing marked it as
test-only, so nothing stopped it being used.

**Fix:** make it **package-private**, and put the test in the same package. Maven and
Gradle both compile `src/test/java` against `src/main/java` with the *same* package
names, so a test in `com.orderflow.pricing` can call package-private members of
production code in `com.orderflow.pricing`. This is the single most useful practical
application of package-private in day-to-day Java, and it has no Jest equivalent —
Jest lets you reach into a module's unexported internals only by hacking the module
system, whereas Java gives you a first-class, compiler-enforced way to say "visible to
my own tests, invisible to everyone else".

If you truly must keep it public, annotate the intent:
```java
@VisibleForTesting  // Guava, or your own annotation
public long applyTieredDiscount(...) { ... }
```
That is documentation, not enforcement. Prefer package-private.

---

### Trap 5 — treating package-private as a security boundary

**Wrong:**
```java
package com.orderflow.payments;

class ApiKeys {                                   // package-private, "so it's safe"
    static final String STRIPE_SECRET = System.getenv("STRIPE_KEY");
}
```

**Exact symptom:** none, until someone shows you this in a code review or a pentest
report. A dependency — or a colleague's utility jar, or a test fixture — contains:

```java
package com.orderflow.payments;                    // same name, different jar

public final class Oops {
    public static String leak() { return ApiKeys.STRIPE_SECRET; }
}
```

It compiles. It runs. On the classic classpath, packages are not sealed and there is no
check that two classes claiming the same package came from the same place.

**Root cause:** package-private is enforced by the *compiler* against a *name*. Anyone
can use that name.

**Fix, in ascending strength:**
- Do not put secrets in a class at all. Read them from the environment or a secret
  manager at the point of use (Topic 43).
- Seal the jar (`Sealed: true` in the manifest) — the classloader then refuses classes
  for a sealed package from a different source. Rarely used, but real.
- JPMS: a non-exported package in a named module is inaccessible at both compile and
  run time, and split packages are outright forbidden. This is the only complete
  answer, and Topic 20 explains why almost nobody adopts it.

> Version note: on the classpath (no `module-info.java`) split packages are permitted
> and this leak works. Inside a named module they are forbidden. If you are unsure which
> mode your app runs in, the command that settles it is in the hands-on section.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM; I will describe what to look for
and how to read each outcome, not invent output.

### Setup

```bash
mkdir -p ~/java-lab/03/src/com/orderflow/orders \
         ~/java-lab/03/src/com/orderflow/reporting \
         ~/java-lab/03/src/com/orderflow/orders/internal
cd ~/java-lab/03
java --version
```

### Proof 1 — four levels, four errors

Create the three files from Example 1 under `src/com/orderflow/...`, then:

```bash
javac -d out $(find src -name '*.java')
```

Uncomment one commented-out line at a time and recompile.

**What to look for:** the exact wording differs per level. Collect all four.

| What you see | What it means |
|---|---|
| `error: privateRef has private access in Order` | `private` is class-scoped. Expected from any other class, same package or not. |
| `error: packageRef is not public in Order; cannot be accessed from outside package` | Package-private, viewed from another package. Note the wording says *"is not public"* — Java describes it by what it lacks. |
| `error: protectedRef has protected access in Order` | Protected, from a non-subclass in another package. |
| No error for `publicRef` from anywhere | Expected. This is the promise you just made. |
| An error you did not expect from the *same* package | Your directory layout does not match your `package` statements. Check that the file sits at `src/com/orderflow/orders/`. |

### Proof 2 — the `protected`-through-your-own-type rule

`src/com/orderflow/subscriptions/Subscription.java`:
```java
package com.orderflow.subscriptions;
import com.orderflow.orders.Order;

public class Subscription extends Order {
    void bump(Order other, Subscription sibling) {
        this.version++;       // expect OK
        sibling.version++;    // expect OK
        other.version++;      // expect a compile error
    }
}
```
(Add `protected long version;` to `Order` first.)

```bash
javac -d out $(find src -name '*.java')
```

**What to look for:**

| What you see | What it means |
|---|---|
| Exactly one error, on the `other.version++` line: `error: version has protected access in com.orderflow.orders.Order` | The expected result. Protected access from another package is granted through your own type only. |
| No errors at all | `Subscription` is in the same package as `Order`. Check the `package` line. |
| Three errors | `version` is not `protected` — check you did not leave it `private`. |

**How to read it:** the compiler is enforcing *why* protected exists — you inherited
the field, so you may manage your own copy. It was never a licence to reach into
arbitrary instances of the parent.

### Proof 3 — `javap` shows the flags the JVM actually enforces

```bash
javap -p -c out/com/orderflow/orders/Order.class | head -30
```

**What to look for:** each field and method line carries its modifiers. Look for lines
beginning `public`, `protected`, `private`, and lines with **no modifier at all** —
that last group is your package-private members.

For the raw access flags:
```bash
javap -v out/com/orderflow/orders/Order.class | grep -A2 'packageRef\|privateRef'
```
Look for a `flags:` line. `ACC_PRIVATE` and `ACC_PROTECTED` appear explicitly;
package-private members have **neither** `ACC_PUBLIC`, `ACC_PRIVATE` nor
`ACC_PROTECTED`. The absence *is* the encoding.

**How to read it:** this is proof that access control is not a compiler-only fiction.
The flags are in the class file, and the JVM re-checks them when it links a call site.
That is the difference from TypeScript's `private`, which leaves no trace at all.

### Proof 4 — package-private is not a security boundary

```bash
mkdir -p hostile/com/orderflow/orders
cat > hostile/com/orderflow/orders/Oops.java <<'EOF'
package com.orderflow.orders;

public final class Oops {
    public static String leak(Order o) { return o.packageRef; }
}
EOF

javac -cp out -d hostile-out hostile/com/orderflow/orders/Oops.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| It compiles with no error | Expected on the classpath. You have just read a package-private field from a completely separate compilation, by declaring the same package name. Package-private is a convention `javac` enforces against a *name*, not a boundary. |
| `error: package com.orderflow.orders is not accessible` or a split-package error | You are compiling in module mode. Check for a `module-info.java`. Named modules forbid split packages — which is exactly the protection the classpath lacks. |

Now run it and confirm the JVM agrees:
```bash
java -cp out:hostile-out com.orderflow.orders.Oops
```
(Add a `main` that prints `leak(new Order())`.)

| What you see | What it means |
|---|---|
| The value printed | Confirmed end to end: neither compiler nor JVM stopped the leak. |
| `java.lang.IllegalAccessError` | You are running with sealed packages or in module mode. Worth noting which — see the next proof. |

### Proof 5 — which mode am I actually in?

Uncertainty is real here: whether split packages are permitted depends on classpath vs
module path, and most Spring Boot apps run on the classpath even on Java 21. The command
that settles it:

```bash
java -XshowSettings:properties -version 2>&1 | grep -i 'class.path\|module.path'
jar --describe-module --file build/libs/orderflow.jar   # for a built jar
```

**How to read it:** if `--describe-module` reports "no module-info.class" and derives an
automatic module name from the filename, you are on the classpath with an automatic
module — no strong encapsulation. If it prints `exports` and `requires` clauses, you
have a real named module and the rules above tighten. Topic 20 covers the difference
properly.

### Proof 6 — the boundary test you would actually run in CI

This one is not a JVM command; it is the practice. Add ArchUnit to your test scope and
write:

```java
@AnalyzeClasses(packages = "com.orderflow")
class ModuleBoundaryTest {

    @ArchTest
    static final ArchRule internals_are_private_to_their_module =
        noClasses().that().resideOutsideOfPackage("com.orderflow.payments..")
                   .should().dependOnClassesThat()
                   .resideInAPackage("com.orderflow.payments.internal..");
}
```

**What to look for:** run it against your current repo before you tidy anything.

| What you see | What it means |
|---|---|
| The rule passes | Your boundary is real today. Keep the test so it stays real. |
| A violation list naming classes and the exact dependency | This is your leak inventory, generated rather than guessed. It is also the most useful artefact from this entire topic. |
| A `ClassResolutionException` | ArchUnit could not find classes on the test classpath. Check your build config, not the rule. |

---

## Practice exercises

### 1 — Easy: the visibility matrix, by experiment

Build the four-package lab: `com.orderflow.orders`, `com.orderflow.orders.internal`,
`com.orderflow.reporting`, and `com.orderflow.subscriptions` (whose class extends
`Order`).

Fill in this table **by compiling**, not by reasoning, and paste the exact error text
for each `no`:

| Accessing from | `public` | `protected` | package-private | `private` |
|---|---|---|---|---|
| Same class | | | | |
| Same package, different class | | | | |
| Subclass, different package (via `this`) | | | | |
| Subclass, different package (via a parent-typed reference) | | | | |
| Unrelated class, different package | | | | |
| `com.orderflow.orders.internal` (subpackage) | | | | |

Two of these rows surprise most people. Note which two surprised you.

### 2 — Medium: the API surface audit (combines Topics 02 and 03)

Take this `orderflow` pricing package:

```java
package com.orderflow.pricing;

public class PricingService {
    public Map<String, Object> quote(String customerId, String productId, Integer qty) { ... }
    public long applyTieredDiscount(long subtotalMinor, String tier) { ... }
    public long applyVat(long amountMinor, String countryCode) { ... }
    public static Map<String, Long> TIER_THRESHOLDS = new HashMap<>();
}

public class VatTable {
    public Map<String, Integer> ratesByCountry = new HashMap<>();
}
```

1. List every element of the **public surface** and, for each, write the sentence you
   would have to say in a design review to justify keeping it public.
2. Identify the two defects that come from **Topic 02**, not from this topic. State
   their observable production symptoms.
3. `public static Map TIER_THRESHOLDS` is not merely a visibility problem. Name the
   *second* problem with it, and describe what an on-call engineer sees when it bites
   under concurrent load. (You do not need Phase 9 to say something useful here.)
4. Rewrite the package with a deliberate boundary: a small public face, package-private
   implementation, and types instead of `String`/`Map` where Topic 02 says they belong.
5. State exactly which of your changes would be a **binary**-incompatible change for a
   downstream service that was compiled against the old version, and what error it would
   see at runtime.

### 3 — Hard: production simulation — drawing a module boundary under pressure

`orderflow` payments is one package, fully public, and three other teams call into it.
You have been asked to make a provider swap possible without a cross-team release train.

**Part A.** Build a two-module Maven project (Topic 31 arrives later; use the simplest
possible parent POM, or just two source roots and two `javac` invocations if you prefer
to avoid the build tool today):
- `orderflow-payments-api` — interfaces, sealed result types, enums. No implementation.
- `orderflow-payments-impl` — everything else, all package-private except one public
  factory.

**Part B.** Make `orderflow-orders` depend on the **api** module only. Prove it: build
`orders` with `-impl` absent from the compile classpath and present only at runtime.
Record what happens if you get it wrong — you want to see the failure shape, which is
`NoClassDefFoundError` at first use, not at startup.

**Part C.** Now simulate the incident. Change a method signature on a package-private
impl class, rebuild **only** `-impl`, and run against the old `-orders` build.
- Does anything fail? Why not?
- Now do the same to a method on the **api** interface. Rebuild only `-api`. Run.
- Capture the exact exception. It should be `NoSuchMethodError` or `AbstractMethodError`
  depending on which side you changed and how. Explain the difference between the two in
  your own words.

**Part D.** Write the ArchUnit rule that would have prevented the original problem, and
deliberately violate it to confirm the test fails.

**Part E.** Argue the other side honestly. You have now spent real effort on a boundary.
Name a concrete situation in which drawing it would have been the wrong call — and say
what evidence would tell you which situation you are in.

---

## Interview questions

### Q1 — "Walk me through Java's access modifiers."

**Mid-level answer:** "Public is everywhere, private is the class, protected is
subclasses, and default is the package."

**Senior answer:** "Four levels, and the ordering is not what people expect: `private`,
then package-private, then `protected`, then `public` — `protected` is *wider* than
package-private, because it is package access **plus** subclasses in other packages.
Two details matter in practice. First, `protected` access from a subclass in a different
package only works through a reference of your own type, not through an arbitrary parent
reference. Second, packages are flat: `com.x.y` and `com.x.y.internal` are unrelated for
access, which is the thing that surprises people from folder-based module systems. And
package-private is the only real intra-artifact boundary you get before JPMS — I use it
deliberately to keep implementation classes out of the published surface, because
everything public is a support obligation."

**What separates them:** the correct *ordering*, the protected-through-your-own-type
rule, the flat-package fact, and framing `public` as an obligation rather than a
setting.

**Follow-up:** "Which of those does the JVM enforce, and which is compiler-only?" All
four are in the class file and re-checked at link time — but package-private can be
defeated by declaring the same package name, which is a `javac`-level convention, not a
runtime guarantee.

---

### Q2 — "Is package-private a security boundary?"

**Mid-level answer:** "It stops other packages from accessing it, so yes, kind of."

**Senior answer:** "No. On the classpath, anyone can declare a class in
`com.orderflow.payments` from a completely different jar and read your package-private
members — split packages are permitted and there is no check that classes claiming a
package share an origin. It is an *encapsulation* boundary, enforced by `javac` against
a name, which is genuinely useful for keeping an API small. It is not a *security*
boundary. If I need one, the options are a sealed jar, or a named JPMS module where a
non-exported package is inaccessible at compile and run time and split packages are
forbidden outright. In practice, secrets should not be in a class at all — they come
from the environment or a secret manager at the point of use."

**What separates them:** knowing split packages defeat it, distinguishing encapsulation
from security, and giving the actual mitigation rather than only the theory.

**Follow-up:** "You're on Spring Boot 3 or 4 — are you in module mode?" Almost certainly
not; fat jars run on the classpath. They are checking whether you know what your own
deployment actually does.

---

### Q3 — "You need to test a private method. What do you do?"

**Mid-level answer:** "Make it public, or use reflection to call it in the test."

**Senior answer:** "Neither, usually. First I'd ask whether the private method is
really a separate responsibility that wants to be its own class with its own public
API — most 'I need to test a private method' cases are a missing collaborator. If it
genuinely belongs where it is, I make it **package-private** and put the test in the
same package; Maven and Gradle compile `src/test/java` against `src/main/java` with the
same package names, so that just works and the method stays invisible to production
callers in other packages. Reflection in tests I'd avoid: it survives renames silently,
so the test keeps passing while testing nothing. And widening to `public` for a test is
how a test helper becomes a production API that nobody meant to publish — I have seen
that produce a finance reconciliation break, because a caller used the discount helper
directly and skipped VAT."

**What separates them:** treating it as a design signal first, knowing the
same-package test trick, and naming the concrete downstream damage of the lazy fix.

**Follow-up:** "How is that different from what you'd do in a Jest test?" They are
probing the honest comparison: in Node you would either export it or reach into the
module, and there is no compiler-enforced middle ground.

---

### Q4 — "What is wrong with a `protected` field?"

**Mid-level answer:** "It exposes state to subclasses, which breaks encapsulation."

**Senior answer:** "It publishes your representation to every subclass, in every
package, forever — including ones written by people who never read your class. That
kills your ability to change the field, and it kills your invariants, because there is
no longer a single place where the field is set. The specific version that bites in
`orderflow` is an auditable base entity: a subclass writes `updatedAt` directly, skips
the `touch()` method that also stamps `updatedBy`, and now under Hibernate you get a
dirty-check `UPDATE` with a null actor. Nothing throws; you find it when a compliance
report has holes. The rule I follow: `protected` is for methods a subclass is meant to
call or override. On a field it is almost always wrong — make the field `private` and
give subclasses a `protected` method that maintains the invariant."

**What separates them:** an observable symptom instead of a principle, and a stated rule
rather than a warning.

**Follow-up:** "So should base classes have any protected members at all?" They want the
template-method pattern named, and ideally the observation that a `protected abstract`
hook is a *contract with subclasses* and deserves as much design care as a public API.

---

### Q5 — "How do you enforce a module boundary in a Spring Boot monolith?"

**Mid-level answer:** "Use packages and keep things private where you can."

**Senior answer:** "Layered, cheapest first. Package-private implementation classes plus
one public interface and one public factory gets you compiler enforcement of the
*types*. The package itself is still open, so I add an ArchUnit rule in CI —
'no class outside `com.orderflow.payments..` may depend on `com.orderflow.payments.internal..`'
— which fails the PR rather than the deploy. That combination is what I would actually
ship. Above that: separate Maven modules give you a real artifact boundary and force
the dependency direction, at the cost of a multi-module build. JPMS is the only thing
that gives runtime enforcement, and it is all-or-nothing across your whole dependency
graph, which is why adoption stalled outside the JDK. There's also a Spring-specific
option — Spring Modulith — which does the ArchUnit-style verification with Spring
semantics built in, if the team is already invested."

**What separates them:** giving an ordered set of options with costs, naming ArchUnit as
the pragmatic default, and being honest about JPMS rather than recommending it
reflexively.

**Follow-up:** "What breaks first when the boundary is only a convention?" Good answers
mention that the first violation is always urgent and reasonable, and that without a CI
check nobody ever finds the second one.

---

## Mental model checkpoint

1. `protected` is wider than package-private. Given that, why does the keyword list
   `private / protected / public` read like an ordering to almost everyone? What would
   a better keyword have been, and what would it have broken?

2. Packages are flat for access. Design a language where they were hierarchical —
   `com.x.y.internal` visible to `com.x.y` but not the reverse. What new failure mode
   would you have created?

3. Your test source root uses the same package names as production code. That is how
   package-private testing works. What does it imply about the relationship between
   `src/test/java` and `src/main/java` at compile time, and what could go wrong if you
   also shipped test classes in the jar?

4. Reflection can read a `private` field (with `setAccessible(true)`, subject to module
   rules). Does that mean `private` is only advisory? Argue both sides, then commit.

5. You are designing a library, not a service. Which of the arguments in this doc get
   *stronger*, and which get weaker? Name at least one that flips direction entirely.

6. A colleague proposes "make everything public, we're all adults here — Python manages
   fine". Give the strongest version of their argument, then the strongest rebuttal that
   does not appeal to convention.

7. Nominal typing (Topic 02) decides *whether* a type can be used somewhere. Access
   modifiers decide *where*. Construct a case where you need both mechanisms together
   and neither alone would do the job.

---

## Quick reference card

### The matrix

| Accessing code | `public` | `protected` | package-private | `private` |
|---|---|---|---|---|
| Same class | yes | yes | yes | yes |
| Same package | yes | yes | yes | no |
| Subclass, other package, via own type | yes | yes | no | no |
| Subclass, other package, via parent-typed ref | yes | **no** | no | no |
| Unrelated class, other package | yes | no | no | no |
| Subpackage (`a.b.c` accessing `a.b`) | yes | no | **no** | no |

### Syntax

```java
package com.orderflow.payments.internal;   // first statement in the file

public    class A { }      // top-level: only public or package-private
          class B { }      // package-private — a real level, not an omission
final     class C { }      // no subclasses
public sealed interface D permits E, F { } // restricts extension, not visibility

class Order {
    private   long checksum;   // this top-level class only (incl. nested classes)
              long seq;        // package-private
    protected long version;    // package + subclasses (own-type reference only)
    public    long id;         // a support obligation
}
```

### Enforcement strength, in order

| Mechanism | Compile-time | Run-time | Cost |
|---|---|---|---|
| Naming convention (`.internal`) | no | no | free |
| Package-private | yes (types/members) | yes (JVM re-checks flags) | free |
| ArchUnit rule in CI | build failure | no | one test |
| Separate Maven module | yes | no | multi-module build |
| Sealed jar manifest | no | yes | packaging config |
| JPMS named module | yes | yes | whole dependency graph |

### Gotchas checklist

- [ ] Package-private is a **decision**. Write a comment saying why, so nobody "fixes"
      it to `public`.
- [ ] `com.x.y.internal` is not inside `com.x.y`. There are no subpackages, access-wise.
- [ ] `protected` on a field is almost always wrong. `protected` on a method is fine.
- [ ] `protected` from another package works through your own type only.
- [ ] Never widen access for a test. Narrow to package-private and move the test.
- [ ] `public` in a separately-built artifact is a **binary** compatibility promise.
      Breaking it gives `NoSuchMethodError` at runtime with a clean build.
- [ ] Package-private is not security. Split packages defeat it on the classpath.
- [ ] One `public` top-level type per file, matching the filename.
- [ ] A `private` member is visible to the whole top-level class, including sibling
      nested classes.

---

## When would I use this at work?

**1. The first PR you open in a new Java codebase.**
You will add a method. The IDE will offer `public`. Choosing package-private instead
takes one keystroke fewer and permanently shrinks the surface you are responsible for.
Doing this consistently is the difference between a module you can refactor and one you
cannot.

**2. When a refactor turns out to be impossible.**
You want to change a signature and discover eleven callers across four teams. That is
not a refactoring problem — it is an access-modifier decision that someone made two
years ago without noticing. The fix going forward is a boundary and an ArchUnit rule;
the fix today is a deprecation cycle. Knowing which of those you are in, quickly, is the
skill.

**3. Splitting a monolith, or deciding not to.**
Before extracting a service, draw the boundary *inside* the monolith with packages and
a CI rule, and run it for a quarter. If the rule keeps failing for good reasons, the
boundary is in the wrong place and extracting it would have produced a distributed
version of the same mess. This is the cheapest architectural experiment available to
you, and it is pure Topic 03.

---

## Connected topics

**Prerequisites:**
- **02 — Nominal vs structural typing**: access controls *where* a name may be used;
  nominal typing controls *whether* it fits. You need both to design a boundary.

**This unlocks:**
- **04 — Interfaces vs abstract classes**: interface members are implicitly `public`,
  which is why "put it on the interface" is a bigger commitment than it looks.
- **09 — Exception API design**: an exception type in a public signature is part of your
  public surface.
- **17 — Immutability**: `private final` fields plus defensive copying is the complete
  encapsulation story, and `final` adds a memory-model guarantee on top.
- **20 — JPMS and jlink**: the only mechanism that makes any of this enforceable at
  runtime, and why almost nobody uses it.
- **31/32 — Maven modules and dependency resolution**: where `public` becomes a *binary*
  compatibility promise across independently built artifacts.
- **36 — Component scanning**: Spring will happily instantiate a package-private
  `@Component`; visibility and bean-ness are separate questions.
- **40 — Proxying**: CGLIB proxies cannot advise `private`, `static` or `final` methods,
  so an access decision here silently determines whether `@Transactional` works.
- **58–60 — Testing**: package-private plus a same-package test is the idiomatic way to
  test internals without publishing them.

---

*Java baseline 21. The four access levels have not changed since Java 1.0 and will not.
What has changed is the enforcement layer around them: JPMS (Java 9) added real runtime
encapsulation for named modules, and the strong-encapsulation-by-default change in Java
16–17 closed most reflective back doors into the JDK's own internals. None of that
altered the four keywords; it altered what happens when you try to go around them.*
