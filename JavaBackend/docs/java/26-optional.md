# 26 — Optional — Correct Use, and Why `get()` Is Null With Extra Steps

## Phase: 2 — Modern Java
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (spine starts at Topic 35)

---

## ELI5 anchor

Imagine you ask a warehouse clerk for the stock count of a product.

**The old way (returning `null`):** the clerk either hands you a slip of paper with a
number on it, or hands you *nothing at all* — an empty hand. You are not looking at
their hand. You put "what they gave you" into your pocket and walk off. Later you
open your pocket, find nothing, and fall over.

**The `Optional` way:** the clerk *always* hands you an envelope. The envelope either
has a slip inside it or it does not. You cannot read the number without opening the
envelope, and opening it forces you to notice whether it was empty.

That is the whole idea. The envelope makes "there might be nothing here" visible.

Now the honest part, and it is the point of this whole document:

**Nobody stops you from ripping the envelope open and reading whatever falls out.**
That move is called `.get()`. If the envelope was empty, you fall over exactly like
before — you just took one extra step to do it. Java's compiler will not warn you.
It only warns you in TypeScript because TypeScript's compiler was built to.

---

## The bridge from what you know

### What you have today

In TypeScript with `strictNullChecks` on:

```ts
interface Product { sku: string; name: string; }

function findProduct(sku: string): Product | null { /* ... */ }

const p = findProduct("SKU-4471");
console.log(p.name);
//          ^^^ Error: 'p' is possibly 'null'.   <- COMPILER STOPS YOU
```

You physically cannot ship that. The build fails. You are forced to narrow:

```ts
const p = findProduct("SKU-4471");
if (p !== null) {
  console.log(p.name);   // narrowed to Product, compiles
}
```

That guarantee — **the compiler refuses to build until you handle the absence** — is
what you are used to. Hold onto it, because Java does not give it to you.

### The Java version

```java
Optional<Product> p = catalog.findBySku("SKU-4471");
System.out.println(p.get().name());   // compiles fine. Throws at runtime if empty.
```

That compiles. It ships. It throws in production.

### The verdict — be precise about this

| TypeScript | Java | Verdict |
|---|---|---|
| `Product \| null` under `strictNullChecks` | `Optional<Product>` | **PARTIAL** — the *intent* transfers exactly; the *enforcement* does not. |
| Compiler refuses to compile unchecked access | Nothing refuses. `.get()` is a legal method call. | **NO ANALOGUE** — there is no compiler gate in stock Java. |
| Narrowing via `if (p !== null)` | `p.map(...)`, `p.ifPresent(...)`, `p.orElse(...)` | **PARTIAL** — same shape, but these are library methods, not language narrowing. |
| Optional chaining `p?.name` | `p.map(Product::name)` | **HONEST ANALOGUE** — genuinely the same operation. |
| Nullish coalescing `p?.name ?? "unknown"` | `p.map(Product::name).orElse("unknown")` | **HONEST ANALOGUE** |
| `strictNullChecks` covering *every* type in the program | — | **NO ANALOGUE in stock Java.** See below. |

### What actually gets you closer to `strictNullChecks`

`Optional` is a **return-type convention**. It is not a type-system guarantee. Every
`String` in your Java program can still be `null`, and nothing checks it.

The thing that genuinely approximates `strictNullChecks` is a pair of tools:

- **JSpecify** — a standard set of annotations (`@Nullable`, `@NonNull`,
  `@NullMarked`) that several vendors agreed on, so they finally mean the same thing
  across libraries. Spring Boot 4 / Framework 7 adopt JSpecify across their whole
  portfolio, so Spring's own APIs now tell you which returns can be null.
- **NullAway** (or ErrorProne, or IntelliJ's inspections, or the Checker Framework) —
  a *build-time checker* that reads those annotations and **fails your build** when
  you dereference something annotated `@Nullable` without checking it.

`@NullMarked` on a package says "everything in here is non-null unless marked
`@Nullable`" — which is precisely `strictNullChecks`' default. NullAway then enforces
it. That combination is the honest answer to "how do I get my TypeScript guarantee
back", and it is a build configuration decision, not a language feature.

> One-line honesty flag: exact NullAway/JSpecify versions and their Gradle/Maven
> wiring change often, and I am not going to quote a config block I cannot verify.
> Check the current JSpecify and NullAway READMEs when you wire it up. The *concept*
> is stable; the coordinates are not.

---

## What is this?

`java.util.Optional<T>` is a container class that holds either **one value** or
**nothing**. It has exactly two states: present, or empty.

It was added in Java 8 for one specific job: **to let a method signature say "this may
return nothing" in a way the caller cannot miss by accident.**

Three facts that define everything else in this document:

1. **It is a value, not a keyword.** `Optional` is an ordinary final class. It has no
   special compiler support at all. No narrowing, no flow analysis, no warnings.
2. **It was designed for return types only.** Brian Goetz (Java's language architect)
   has stated this directly: it was intended as a *limited* mechanism for library
   method return types where "no result" needed to be clearly represented. Not fields,
   not parameters, not collection elements.
3. **It is an object.** Wrapping a value allocates. Usually irrelevant. Occasionally
   it is not, and you should know which case you are in.

---

## Why does it matter?

Four things go wrong in production, and each has a distinct signature.

**1. `.get()` in an incident report.**
`NoSuchElementException: No value present` is the second-most-common preventable
production exception in Java after `NullPointerException`, and it is embarrassing
because the code *looks* defensive. Someone typed `Optional` and thought that was the
work.

**2. A slow endpoint nobody can explain.**
`orElse(loadDefaultsFromDatabase())` evaluates the database call **on every single
call**, including the 99% where a value was present. The profile shows a query nobody
can find in the code path they think is executing.

**3. A serialization failure at a service boundary.**
`Optional` is not `Serializable`. An `Optional` field in a class you serialize throws.
And Jackson's default handling of it produces a JSON shape your consumers did not
agree to.

**4. Code that is objectively worse than the null check it replaced.**
```java
if (order.getCoupon().isPresent()) {
    applyDiscount(order.getCoupon().get());
}
```
That is a null check with two extra allocations and more characters. If your
`Optional` code contains `isPresent()` followed by `get()`, you have converted a
readable null check into an unreadable one.

---

## Syntax breakdown

### Creating an Optional

```java
Optional<Product> a = Optional.of(product);          // product MUST NOT be null
Optional<Product> b = Optional.ofNullable(product);  // product MAY be null
Optional<Product> c = Optional.empty();              // explicitly nothing
```

| Construct | What it means | Failure mode |
|---|---|---|
| `Optional.of(x)` | "I promise `x` is not null." | Throws `NullPointerException` immediately if `x` is null. **This is a feature** — it fails at the source instead of three frames later. |
| `Optional.ofNullable(x)` | "`x` might be null; turn null into empty." | Never throws. This is your bridge from legacy null-returning APIs. |
| `Optional.empty()` | "Definitely nothing." | Returns a shared singleton — no allocation. |

The choice between `of` and `ofNullable` is a **statement about your knowledge**. Use
`of` when you know it is non-null, so that a violated assumption blows up loudly at
the exact line. Reaching for `ofNullable` everywhere "to be safe" throws away that
signal.

### Getting the value out — the whole menu

```java
Optional<Product> p = catalog.findBySku(sku);

// --- TERMINAL: produce a value ---
p.orElse(Product.UNKNOWN);              // fallback value (ALWAYS evaluated - see Trap 3)
p.orElseGet(() -> loadDefault());       // fallback computed LAZILY, only if empty
p.orElseThrow();                        // throws NoSuchElementException if empty
p.orElseThrow(() -> new ProductNotFoundException(sku));   // your own exception

// --- TERMINAL: do something ---
p.ifPresent(prod -> log.info("found {}", prod.sku()));
p.ifPresentOrElse(
    prod -> log.info("found {}", prod.sku()),
    ()   -> log.warn("missing sku {}", sku));

// --- TRANSFORM: stay inside the Optional ---
p.map(Product::name);                   // Optional<Product> -> Optional<String>
p.flatMap(Product::activePromotion);    // when the mapper ITSELF returns Optional
p.filter(prod -> prod.isActive());      // present -> empty if predicate fails
p.or(() -> catalog.findByLegacyCode(sku));  // Java 9+: try another source

// --- INSPECT ---
p.isPresent();                          // boolean
p.isEmpty();                            // Java 11+, reads better than !isPresent()
p.stream();                             // Java 9+: 0-or-1 element Stream (see below)

// --- DO NOT ---
p.get();                                // legal, compiles, and the subject of Trap 1
```

### `map` vs `flatMap` — the one that trips everyone

`map` wraps whatever the function returns. `flatMap` does not.

```java
record Product(String sku, String name) {
    Optional<Promotion> activePromotion() { ... }
}

Optional<Product> p = catalog.findBySku(sku);

p.map(Product::name);              // Optional<String>            <- correct
p.map(Product::activePromotion);   // Optional<Optional<Promotion>>  <- almost never what you want
p.flatMap(Product::activePromotion);  // Optional<Promotion>      <- correct
```

**The rule:** if the function you are passing already returns an `Optional`, use
`flatMap`. Otherwise `map`. This is the exact same rule as `Array.prototype.flatMap`
in JavaScript, and the exact same rule as `Promise` auto-flattening — except Java does
not auto-flatten, so you must say which one you meant.

### `Optional.stream()` — the collection-flattening idiom

```java
// Java 9+. Turns 0-or-1 into a stream, so empties simply vanish.
List<Product> found = skus.stream()
        .map(catalog::findBySku)        // Stream<Optional<Product>>
        .flatMap(Optional::stream)      // Stream<Product>  -- empties dropped
        .toList();
```

Before Java 9 this required `.filter(Optional::isPresent).map(Optional::get)`, which
you will still see in older code. `flatMap(Optional::stream)` is the modern form and
never calls `get()`.

### The chained form, read as one sentence

```java
String label = catalog.findBySku(sku)          // maybe a Product
        .filter(Product::isActive)             // ...if it is active
        .map(Product::name)                    // ...take its name
        .map(String::toUpperCase)              // ...upper-cased
        .orElse("UNKNOWN PRODUCT");            // ...or this if anything above was absent
```

Read that top to bottom in English. That readability is the actual payoff of
`Optional`. `isPresent()`/`get()` has none of it.

---

## Example 1 — minimal

```java
import java.util.Optional;

public class OptionalBasics {

    static Optional<String> lookup(String key) {
        return switch (key) {
            case "a" -> Optional.of("apple");
            default  -> Optional.empty();
        };
    }

    public static void main(String[] args) {
        System.out.println(lookup("a").map(String::toUpperCase).orElse("NONE"));
        System.out.println(lookup("z").map(String::toUpperCase).orElse("NONE"));

        // The line that ships bugs:
        System.out.println(lookup("z").get());
    }
}
```

Run it. The first two lines print `APPLE` and `NONE`. The third throws. Note that the
third line compiles with zero warnings — that is the honest difference from
TypeScript, and it is why the rest of this document exists.

---

## Example 2 — production scenario

`orderflow` needs to price an order line. A product may have an active promotion; a
customer may have a loyalty tier; a tier may have a discount override. Any of the
three can be absent.

### The version that ships and then pages someone

```java
public class OrderPricingService {

    private final ProductCatalog catalog;
    private final PromotionRepository promotions;
    private final LoyaltyService loyalty;

    public long priceLineInMinorUnits(String sku, long quantity, UserId userId) {

        Optional<Product> product = catalog.findBySku(sku);
        Product p = product.get();                                  // (1)

        long base = p.unitPriceMinorUnits() * quantity;

        Optional<Promotion> promo = promotions.activeFor(sku);
        long afterPromo = base;
        if (promo.isPresent()) {                                    // (2)
            afterPromo = promo.get().applyTo(base);
        }

        // (3) computeDefaultTier() hits the database on EVERY call
        LoyaltyTier tier = loyalty.tierFor(userId)
                                  .orElse(loyalty.computeDefaultTier(userId));

        return tier.applyTo(afterPromo);
    }
}
```

Three separate defects, three separate symptoms:

1. `product.get()` — a request for a deleted SKU produces
   `NoSuchElementException: No value present`, which reaches the client as a 500. The
   correct outcome was a 404 with a message naming the SKU.
2. `isPresent()`/`get()` — not a bug, but it is a null check wearing a costume. It is
   two allocations and four lines to express one `map`.
3. `orElse(loyalty.computeDefaultTier(userId))` — this argument is evaluated **before
   `orElse` is called**, because Java evaluates arguments eagerly. So the default-tier
   computation runs on every request, including the 97% of requests where the customer
   *has* a tier. You find this as an unexplained query in the profile.

### The corrected version

```java
public class OrderPricingService {

    private final ProductCatalog catalog;
    private final PromotionRepository promotions;
    private final LoyaltyService loyalty;

    public long priceLineInMinorUnits(String sku, long quantity, UserId userId) {

        Product product = catalog.findBySku(sku)
                .orElseThrow(() -> new ProductNotFoundException(sku));   // (1)

        long base = product.unitPriceMinorUnits() * quantity;

        long afterPromo = promotions.activeFor(sku)                      // (2)
                .map(promo -> promo.applyTo(base))
                .orElse(base);

        LoyaltyTier tier = loyalty.tierFor(userId)                       // (3)
                .orElseGet(() -> loyalty.computeDefaultTier(userId));

        return tier.applyTo(afterPromo);
    }
}
```

What each change bought:

1. `orElseThrow(supplier)` converts absence into a **domain exception carrying the
   SKU**. Topic 46's `@ControllerAdvice` maps `ProductNotFoundException` to a 404 with
   an RFC 9457 `ProblemDetail` body. The absence is now part of your API contract
   instead of a stack trace.
2. `map(...).orElse(base)` — one expression, no `get()`, no mutable local. Note
   `orElse(base)` is fine here: `base` is an already-computed `long`, so eager
   evaluation costs nothing.
3. `orElseGet(supplier)` — the lambda only runs when the tier is absent. Same
   behaviour, 97% fewer queries.

### And the signature that made this possible

```java
public interface ProductCatalog {
    Optional<Product> findBySku(String sku);       // absence is IN the signature
    List<Product>     findByCategory(String cat);  // NEVER Optional<List<...>>
}
```

The second line is a rule, not a style preference. **Never return
`Optional<Collection>`.** An empty collection already means "nothing here", so
`Optional<List<Product>>` gives the caller two ways to say the same thing and forces
them to handle both. Return an empty list.

Same rule for `Optional<Optional<T>>`, `Optional<Boolean>` (use a three-state enum if
you genuinely have three states), and any array type.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — calling `get()` on an empty Optional

**Wrong:**
```java
Product p = catalog.findBySku(sku).get();
```

**Exact symptom:**
```
java.util.NoSuchElementException: No value present
    at java.base/java.util.Optional.get(Optional.java:143)
    at com.orderflow.pricing.OrderPricingService.priceLineInMinorUnits(OrderPricingService.java:24)
```
The client sees a 500. Your error rate spikes for exactly the requests that reference
missing data — which is often a specific tenant or a specific stale cache entry, so it
looks like a partial outage rather than a bug.

**Root cause:** `get()` is documented to throw if empty. It exists because Java 8's API
needed *some* extraction method, and it was named badly. The name says "get me the
value", which sounds unconditional. Java 10 added `orElseThrow()` with no arguments as
an exact synonym with an honest name, precisely because of this.

**Fix — pick by what absence *means*:**

```java
// absence is a client error -> domain exception -> 404 (Topic 46)
Product p = catalog.findBySku(sku)
        .orElseThrow(() -> new ProductNotFoundException(sku));

// absence is normal -> a default
long price = catalog.findBySku(sku)
        .map(Product::unitPriceMinorUnits)
        .orElse(0L);

// absence means skip this step entirely
catalog.findBySku(sku).ifPresent(this::reserveStock);
```

**Code review rule:** `.get()` and `.orElseThrow()` with no argument are both
"I assert this is present". If that assertion is genuinely justified, use
`orElseThrow()` — it reads as an assertion. If you cannot justify it, you needed one
of the other three forms. Ban bare `.get()` with a static analysis rule; every serious
Java linter has one.

---

### Trap 2 — `Optional` as a field

**Wrong:**
```java
public class Order implements Serializable {
    private Optional<String> couponCode;    // <-- the defect
    private Optional<Wallet> wallet;
}
```

**Exact symptom — three different ones, depending on what touches it:**

1. **Java serialization:**
   ```
   java.io.NotSerializableException: java.util.Optional
   ```
   `Optional` deliberately does not implement `Serializable`. That was a design
   decision to discourage exactly this. (Topic 19 covers why you should not be using
   Java serialization anyway — but this is how the mistake surfaces.)

2. **JPA / Hibernate:** the entity fails to map, or maps to a column of a type nobody
   expects. Hibernate needs to read and write the field reflectively and has no
   attribute converter for `Optional` by default.

3. **Jackson:** without the `jackson-datatype-jdk8` module registered, older Jackson 2
   serializes `Optional` as a bean and emits something like
   `{"couponCode":{"present":true}}` instead of `{"couponCode":"SAVE10"}` — a JSON
   shape your API consumers never agreed to. With the module registered (or on
   Jackson 3, which folds the Java 8 datatypes in by default) it serializes as the
   value or `null`.

   > Honesty flag: whether *your* build already registers that module depends on your
   > Jackson version and whether Spring Boot's `ObjectMapper` auto-configuration
   > picked it up. Do not guess. The Hands-on proof below gives you the command that
   > settles it for your exact classpath.

**Root cause:** an `Optional` field is a *second* pointer for every value: the field
points to an `Optional`, which points to the value. That is an extra object per
instance and an extra dereference per read, in exchange for expressing something the
field's own null-ness already expressed. It also creates a third state you must handle
— the field itself can be `null`, so you now have `null`, `Optional.empty()`, and
present.

**Fix:** the field is nullable; the *accessor* returns `Optional`.

```java
public final class Order {
    private final String couponCode;        // may be null. Document it.

    public Optional<String> couponCode() {  // absence is in the API, not the layout
        return Optional.ofNullable(couponCode);
    }
}
```

For a record, same idea (Topic 27 goes deeper):

```java
public record Order(OrderId id, String couponCode /* nullable */) {
    public Optional<String> maybeCouponCode() {
        return Optional.ofNullable(couponCode);
    }
}
```

Note the accessor is named `maybeCouponCode`, not `couponCode` — a record's generated
accessor already owns the name `couponCode()`.

---

### Trap 3 — `orElse(expensiveCall())` evaluated eagerly

**Wrong:**
```java
LoyaltyTier tier = loyalty.tierFor(userId)
        .orElse(loyalty.computeDefaultTier(userId));   // <-- runs ALWAYS
```

**Exact symptom:** a database query or HTTP call that appears in your APM trace on
requests where, reading the code, it obviously should not run. Query counts are
roughly double what you predicted. In the worst version, the "fallback" writes a row,
and you get rows created for users who already had a tier.

**Root cause:** this is not an `Optional` rule, it is a **Java language** rule. Java
evaluates all method arguments before invoking the method. `orElse` is an ordinary
method. Its argument is computed, then passed, then `orElse` decides whether to use
it. There is no laziness anywhere.

This is the same trap as `logger.debug("state: " + expensiveToString())` — the
concatenation happens whether or not debug is enabled.

**Fix:**
```java
LoyaltyTier tier = loyalty.tierFor(userId)
        .orElseGet(() -> loyalty.computeDefaultTier(userId));
```

`orElseGet` takes a `Supplier<T>` (Topic 21). The lambda body does not run until
`orElseGet` decides it needs it.

**The rule:** if the fallback is a *literal, a constant, or an already-computed local*,
`orElse` is clearer. If it is a **method call**, use `orElseGet`. Same for
`orElseThrow(() -> new ...)` — building the exception allocates and often does string
formatting, so it belongs in a supplier.

---

### Trap 4 — `Optional` as a method parameter

**Wrong:**
```java
public List<Order> search(Optional<OrderStatus> status,
                          Optional<Instant> from,
                          Optional<Instant> to) { ... }
```

**Exact symptom:** every call site becomes noise —
`search(Optional.of(PAID), Optional.empty(), Optional.empty())`. Worse, this is legal
and compiles:
```java
search(null, null, null);   // NPE inside the method on status.isPresent()
```
So you have paid the ceremony cost and still get a `NullPointerException`. You have
also made the method impossible to call from a framework that binds parameters by
reflection.

**Root cause:** `Optional` communicates absence *from* a method *to* its caller. On the
way in, the caller already has a perfectly good way to say "I am not supplying this":
overloads, a builder, a parameter object, or a documented nullable parameter.

**Fix — pick one:**

```java
// 1. Overloads, when there are two or three shapes
public List<Order> search(OrderStatus status) { ... }
public List<Order> search(OrderStatus status, Instant from, Instant to) { ... }

// 2. A parameter object, when there are many optional filters
public record OrderSearchCriteria(OrderStatus status, Instant from, Instant to) {
    public static OrderSearchCriteria all() { return new OrderSearchCriteria(null, null, null); }
}
public List<Order> search(OrderSearchCriteria criteria) { ... }

// 3. A documented nullable parameter, annotated so a checker can enforce it
public List<Order> search(@Nullable OrderStatus status) { ... }
```

Option 3 with JSpecify `@Nullable` plus NullAway in the build is the one that actually
gives you the TypeScript-shaped guarantee, because the checker fails the build if a
caller passes a possibly-null value where `@NonNull` is expected.

---

### Trap 5 — `Optional.of()` on something that can be null

**Wrong:**
```java
public Optional<Wallet> walletFor(UserId id) {
    return Optional.of(wallets.get(id));   // Map.get returns null for missing keys
}
```

**Exact symptom:**
```
java.lang.NullPointerException
    at java.base/java.util.Objects.requireNonNull(Objects.java:233)
    at java.base/java.util.Optional.of(Optional.java:113)
```
An NPE thrown *from inside `Optional`* — the class you added specifically to avoid
NPEs. This confuses people badly the first time.

**Root cause:** `Optional.of` calls `Objects.requireNonNull` on its argument by design.
It is the "I promise this is non-null" constructor.

**Fix:** `Optional.ofNullable(wallets.get(id))`.

**But read this before you go replace every `of` with `ofNullable`:** the throw is
often the *correct* behaviour. If `wallets.get(id)` returning null means your data is
corrupt, you want to know at that line, not to silently return `empty()` and have the
caller render "no wallet" for a user who definitely has one. Choose `of` when null
would be a bug, `ofNullable` when null is an expected outcome. The method name is
documentation for the next reader.

---

## Hands-on proof

Everything below is a command **you** run. I do not have a JVM and will not print
output and claim it is real. What follows is exactly what to look for and how to read
each possible result.

### Setup

```bash
mkdir -p ~/java-lab/26 && cd ~/java-lab/26
java --version      # expect 21 or 25
```

### Proof 1 — `get()` is null with extra steps

`GetProof.java`:
```java
import java.util.Optional;
import java.util.Map;

public class GetProof {
    public static void main(String[] args) {
        Map<String, String> catalog = Map.of("SKU-1001", "Widget");

        // The old way
        try {
            String name = catalog.get("SKU-4471");
            System.out.println(name.length());
        } catch (Exception e) {
            System.out.println("NULL WAY   : " + e);
        }

        // The Optional way, done wrong
        try {
            String name = Optional.ofNullable(catalog.get("SKU-4471")).get();
            System.out.println(name.length());
        } catch (Exception e) {
            System.out.println("OPTIONAL   : " + e);
        }
    }
}
```

```bash
java GetProof.java
```

**What to look for:** two exception lines.

| What you see | What it means |
|---|---|
| `NULL WAY : java.lang.NullPointerException: Cannot invoke "String.length()" because ... is null` | Baseline. The classic NPE, with Java 14+'s helpful message naming the receiver. |
| `OPTIONAL : java.util.NoSuchElementException: No value present` | The point of the exercise. **Different exception name, identical failure.** You wrote more code, allocated an object, and still crashed on absent data. |
| Anything else, or no exception | You edited the SKU. Both lookups must miss. |

**How to read it:** the `Optional` version is not safer. It is only safer *if you use
the methods that force you to handle the empty case*. The container does nothing on
its own.

### Proof 2 — `orElse` evaluates eagerly, `orElseGet` does not

`EagerProof.java`:
```java
import java.util.Optional;

public class EagerProof {

    static String expensiveDefault() {
        System.out.println("  >> expensiveDefault() RAN");
        return "DEFAULT";
    }

    public static void main(String[] args) {
        Optional<String> present = Optional.of("ACTUAL");

        System.out.println("orElse with a value PRESENT:");
        System.out.println("  result = " + present.orElse(expensiveDefault()));

        System.out.println("orElseGet with a value PRESENT:");
        System.out.println("  result = " + present.orElseGet(EagerProof::expensiveDefault));
    }
}
```

```bash
java EagerProof.java
```

**What to look for:** how many times `>> expensiveDefault() RAN` appears.

| What you see | What it means |
|---|---|
| `RAN` printed once, under the `orElse` heading only | Expected. `orElse`'s argument was evaluated even though the value was present; `orElseGet`'s supplier was never invoked. |
| `RAN` printed twice | You passed `expensiveDefault()` (with parentheses) to `orElseGet` instead of the method reference `EagerProof::expensiveDefault`. That calls the method and passes its *result*, defeating the laziness. This is itself a real and common bug — note it. |
| `RAN` printed zero times | You changed `present` to `Optional.empty()`. Re-read: the whole point is that a value **is** present. |

**Why this matters:** you have now proved that `orElse` is not "the fallback". It is
"a value you always compute, that is sometimes used".

### Proof 3 — see the eager evaluation in the bytecode

`EagerBytecode.java`:
```java
import java.util.Optional;

public class EagerBytecode {
    static String compute() { return "d"; }

    static String withOrElse(Optional<String> o)    { return o.orElse(compute()); }
    static String withOrElseGet(Optional<String> o) { return o.orElseGet(EagerBytecode::compute); }
}
```

```bash
javac EagerBytecode.java
javap -c EagerBytecode.class
```

**What to look for**, comparing the two methods:

- In `withOrElse`: an `invokestatic` call to `compute` appears **before** the
  `invokevirtual` call to `Optional.orElse`. The argument is on the stack before the
  call happens. That is eager evaluation, visible.
- In `withOrElseGet`: there is **no** call to `compute` at all. Instead you see an
  `invokedynamic` that builds a `Supplier` object, then `Optional.orElseGet`. The
  actual call to `compute` lives inside a synthetic lambda method
  (something like `lambda$withOrElseGet$0`) that only runs if `orElseGet` invokes it.

| What you see | What it means |
|---|---|
| `invokestatic ... compute` inside `withOrElse` | Confirmed: the fallback is computed unconditionally. |
| `invokedynamic` + no direct `compute` call inside `withOrElseGet` | Confirmed: the fallback is wrapped in a `Supplier` and deferred. This is Topic 21's `LambdaMetafactory` at work. |
| Your listing differs from this description | Trust your listing. Paste it. Bytecode reading in depth is Topic 76. |

### Proof 4 — `Optional` is not `Serializable`

`SerialProof.java`:
```java
import java.io.*;
import java.util.Optional;

public class SerialProof {
    static class Order implements Serializable {
        Optional<String> couponCode = Optional.of("SAVE10");
    }

    public static void main(String[] args) throws Exception {
        try (ObjectOutputStream out =
                 new ObjectOutputStream(new ByteArrayOutputStream())) {
            out.writeObject(new Order());
            System.out.println("serialized OK");
        } catch (Exception e) {
            System.out.println("FAILED: " + e);
        }
    }
}
```

```bash
java SerialProof.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `FAILED: java.io.NotSerializableException: java.util.Optional` | Expected. `Optional` deliberately omits `Serializable`. This is the JDK authors telling you not to put it in a field. |
| `serialized OK` | Something is very unusual — check you did not change the field type. Report it. |

Then change the field to `String couponCode = "SAVE10";` and re-run. It serializes.
That is the fix from Trap 2, proven.

### Proof 5 — what Jackson actually does with your classpath

This one needs real jars, and the answer genuinely depends on your versions. Do not
take my word for it — settle it.

**Inside an existing Spring Boot project** (fastest route, and it tests the exact
`ObjectMapper` your app uses):

```java
// src/test/java/.../OptionalJsonTest.java
@SpringBootTest
class OptionalJsonTest {
    @Autowired ObjectMapper mapper;

    record Dto(Optional<String> couponCode) {}

    @Test void whatShapeIsIt() throws Exception {
        System.out.println(mapper.writeValueAsString(new Dto(Optional.of("SAVE10"))));
        System.out.println(mapper.writeValueAsString(new Dto(Optional.empty())));
        System.out.println(mapper.getRegisteredModuleIds());
    }
}
```

**What to look for:**

| What you see | What it means |
|---|---|
| `{"couponCode":"SAVE10"}` then `{"couponCode":null}` | The Java 8 datatype module is active. `Optional` marshals to the value or `null`. Still do not put it in a field — but at least it is not corrupting your contract. |
| `{"couponCode":{"present":true}}` or similar bean-shaped output | The module is **not** registered. Your API is emitting a shape no consumer expects. This is the failure in Trap 2. |
| The registered-module list contains a `jdk8` entry | Confirms which of the above you are in, without guessing from the output shape. |

**Also run this** to see what is on your classpath at all:
```bash
./mvnw dependency:tree | grep -i jdk8
# or
./gradlew dependencies --configuration runtimeClasspath | grep -i jdk8
```

> Honesty flag, stated once: Jackson 3 folds the Java 8 datatypes in by default and
> Jackson 2 required the separate `jackson-datatype-jdk8` module. Which one Spring
> Boot wires for you depends on your Boot version. The commands above tell you the
> truth for your build; I am not going to assert it for you.

---

## Practice exercises

Write real files, run them, and keep the output.

### 1 — Easy: rewrite the ceremony away

Here are five fragments. Rewrite each as a single expression with no `isPresent()`,
no `get()`, and no mutable local variable. Then state, for each, whether you chose
`orElse` or `orElseGet` and why.

```java
// (a)
Optional<Product> p = catalog.findBySku(sku);
String name;
if (p.isPresent()) { name = p.get().name(); } else { name = "unknown"; }

// (b)
Optional<Wallet> w = wallets.findFor(userId);
if (w.isPresent()) { auditLog.record(w.get()); }

// (c)
Optional<Promotion> promo = promotions.activeFor(sku);
long price = basePrice;
if (promo.isPresent() && promo.get().isValidOn(today)) {
    price = promo.get().applyTo(basePrice);
}

// (d)
Optional<Product> p = catalog.findBySku(sku);
if (!p.isPresent()) { throw new IllegalStateException("no product " + sku); }
Product product = p.get();

// (e)
List<Product> found = new ArrayList<>();
for (String s : skus) {
    Optional<Product> op = catalog.findBySku(s);
    if (op.isPresent()) { found.add(op.get()); }
}
```

For (e), use `Optional::stream` (Topic 23), not `filter`/`map`.

### 2 — Medium: the audit (combines Topics 01–25)

This class contains **six** distinct defects drawn from Topics 01, 13, 17, 23, 24 and
this one. Find them all. For each, state the **observable production symptom** — what
the on-call engineer actually sees, not "it's bad practice" — then rewrite the class.

```java
public class WalletSettlementService implements Serializable {

    private Optional<Wallet> cachedWallet;
    private final Map<Long, Long> debitedByUser = new HashMap<>();
    private final List<Long> settledOrderIds = new ArrayList<>();

    public double settle(Long orderId, Long userId, Optional<Double> amount) {

        for (Long settled : settledOrderIds) {
            if (settled == orderId) {
                return 0.0;
            }
        }

        Wallet wallet = walletRepo.findFor(userId)
                                  .orElse(walletRepo.createDefaultWallet(userId));

        double due = amount.get();

        if (wallet.balance() > due) {
            wallet.debit(due);
            Long current = debitedByUser.get(userId);
            debitedByUser.put(userId, current == null ? (long) due : current + (long) due);
        }

        settledOrderIds.add(orderId);
        return wallet.balance();
    }

    public Optional<List<Payment>> paymentsFor(Long orderId) {
        List<Payment> results = payments.stream()
                .filter(p -> p.orderId() == orderId)
                .collect(Collectors.toList());
        return results.isEmpty() ? Optional.empty() : Optional.of(results);
    }
}
```

Hints on where to look, without giving the answers away: two of the defects are from
Topic 01, one is from Topic 13/17, one is a `Collectors` choice from Topic 24, and two
are from this topic. The `Optional<List<...>>` return is one of the six.

### 3 — Hard: production simulation on `orderflow`

Build a small `CheckoutQuoteService` that composes four lookups, each of which can be
absent, and prove your composition is correct *and* lazy.

**Part A — the domain.** Write these interfaces and simple in-memory implementations:

```java
interface ProductCatalog     { Optional<Product> findBySku(String sku); }
interface PromotionRepo      { Optional<Promotion> activeFor(String sku); }
interface LoyaltyService     { Optional<LoyaltyTier> tierFor(UserId id);
                               LoyaltyTier computeDefaultTier(UserId id); }
interface WalletRepository   { Optional<Wallet> findFor(UserId id); }
```

Every in-memory implementation must **print a line when it is called** (e.g.
`>> LoyaltyService.computeDefaultTier`). That instrumentation is the whole point of
Part C.

**Part B — the service.** Write:

```java
Quote quote(String sku, long quantity, UserId userId);
```

Rules you must satisfy:
- No `.get()` anywhere.
- No `isPresent()` anywhere.
- A missing SKU throws `ProductNotFoundException` carrying the SKU.
- A missing promotion means no discount, not an error.
- A missing loyalty tier falls back to `computeDefaultTier`, which is expensive.
- A missing wallet means the quote is returned with `payableFromWallet = 0` — not an
  error, and `computeDefaultTier` must **not** be called just to discover that.
- Money is `long` minor units throughout (Topic 01, Trap 5).

**Part C — prove the laziness.** Write four scenarios and record which instrumentation
lines print in each:

| Scenario | `computeDefaultTier` should print? |
|---|---|
| Product present, promo present, tier present, wallet present | no |
| Product present, promo absent, tier present, wallet present | no |
| Product present, promo present, tier **absent**, wallet present | yes, exactly once |
| Product **absent** | no — the exception must be thrown before anything else runs |

Run all four and paste the actual output. If any row disagrees with the table, your
composition is eager somewhere — find where.

**Part D — argue the other side.** Now write the same service using nullable returns
plus JSpecify `@Nullable` annotations instead of `Optional`. Compare the two on:
allocation count per quote, readability, and what a *new* team member is likely to get
wrong in each. State which you would ship for `orderflow` and under what condition
your answer flips. "Optional is best practice" is not an answer.

---

## Interview questions

### Q1 — "Why shouldn't you call `Optional.get()`?"

**Mid-level answer:** "Because it throws `NoSuchElementException` if the Optional is
empty. You should check `isPresent()` first."

**Senior answer:** "`get()` throws `NoSuchElementException: No value present`, so it
reinstates exactly the failure `Optional` existed to make visible — it is null with
extra steps and a different exception name. The deeper problem with 'check
`isPresent()` first' is that it is still a null check with ceremony: it costs an
allocation and reads worse than the `if (x != null)` it replaced. The value of
`Optional` is in `map`/`flatMap`/`filter`/`orElseGet`/`orElseThrow`, where the empty
case is handled by the shape of the expression rather than by a branch you might
forget. Java 10 added no-arg `orElseThrow()` as an exact synonym for `get()` precisely
because the name `get` misleads people. I ban bare `get()` with a lint rule."

**What separates them:** the mid-level answer knows the exception. The senior answer
identifies that `isPresent()`/`get()` is *also* wrong, names the alternative API
surface, and knows the `orElseThrow()` renaming history — which shows they have read
why the API is shaped this way rather than just how to use it.

**Interviewer's follow-up:** "When is `orElseThrow()` with no argument actually the
right call?" They want you to say: when absence is genuinely a programming error or a
broken invariant — an assertion — and you want a stack trace rather than a domain
error. If absence is a *client* condition, you want the supplier form with a domain
exception.

---

### Q2 — "What's wrong with an `Optional` field?"

**Mid-level answer:** "`Optional` was designed for return types, not fields. It adds an
extra object."

**Senior answer:** "Four concrete costs. One, it is not `Serializable` — deliberately —
so a field of that type throws `NotSerializableException`. Two, it does not map in
JPA, so it cannot be an entity field. Three, Jackson's default handling depends on
whether the Java 8 datatype module is registered, so you can silently emit
`{\"coupon\":{\"present\":true}}` instead of the value and break consumers. Four, it
adds a pointer hop and an allocation per instance, and it creates a *three*-state
field, because the `Optional` reference itself can be null — so you have not removed a
null check, you have added one. The pattern I use is a nullable field with an accessor
that returns `Optional`: absence is in the API, not in the memory layout."

**What separates them:** the senior answer gives symptoms, not principles. "Not
`Serializable`" and "three states, because the reference can be null" are things you
only say if you have hit them.

**Interviewer's follow-up:** "So how do you document that the field can be null?"
They are looking for JSpecify `@Nullable` plus a build-time checker like NullAway — the
answer that shows you know `Optional` is a convention and annotations plus a checker
are the enforcement.

---

### Q3 — "`orElse` versus `orElseGet` — does it matter?"

**Mid-level answer:** "`orElseGet` takes a supplier so it's lazy. `orElse` takes a
value. Use `orElseGet` if the default is expensive."

**Senior answer:** "It matters, and the reason is a Java language rule rather than an
`Optional` rule: arguments are evaluated before the call. So `orElse(loadDefault())`
runs `loadDefault()` on every invocation, present or not — it is the same trap as
string-concatenating into a `logger.debug` call. The symptom is a query in your APM
trace on a code path where you can see it should not run, and query counts roughly
double what you predicted. My rule is: literal or already-computed local, use
`orElse`; method call, use `orElseGet`. Same for `orElseThrow` — building the exception
does string formatting and fills in a stack trace, so it belongs in the supplier form.
And you can see the difference directly in `javap -c`: `orElse` shows an
`invokestatic` to the method before the `orElse` call; `orElseGet` shows an
`invokedynamic` building a `Supplier` and no direct call at all."

**What separates them:** framing it as argument-evaluation order rather than as an API
quirk, naming the observable symptom, and knowing it is verifiable in bytecode.

**Interviewer's follow-up:** "Is there a case where `orElse` with a method call is
fine?" Yes — when the method is trivial and pure, or when you *want* it evaluated for
its side effect, though that is a smell. They are checking whether you apply the rule
mechanically or understand it.

---

### Q4 — "Coming from TypeScript, is `Optional` the same as `string | null` with `strictNullChecks`?"

**Mid-level answer:** "Yes, roughly — both represent a value that might not be there."

**Senior answer:** "The intent is the same; the enforcement is not, and that is the
whole difference. Under `strictNullChecks` the *compiler* refuses to build code that
dereferences a possibly-null value — narrowing is a language feature with control-flow
analysis behind it. In Java, `Optional` is an ordinary library class with no compiler
support: `optional.get()` compiles with no warning, and separately every reference type
in the program can still be null regardless of any `Optional` anywhere. So `Optional`
is a *return-type convention* that documents absence; it is not a type-system
guarantee. The thing that actually approximates `strictNullChecks` is JSpecify
annotations — `@NullMarked` at the package level makes non-null the default, matching
TypeScript's default — plus a build-time checker like NullAway or the Checker Framework
that fails the build. Spring Boot 4 adopted JSpecify across its portfolio, so
Spring's own APIs now declare their nullability. That combination is the honest answer
to the question."

**What separates them:** refusing the easy equivalence, naming *compiler enforcement*
as the axis, and knowing the tooling that closes the gap. Interviewers moving people
from TypeScript ask this specifically to see whether the candidate imports a false
sense of safety.

**Interviewer's follow-up:** "What would it cost to turn NullAway on in an existing
codebase?" They want to hear that it is incremental — you annotate package by package
with `@NullMarked` — and that the first pass surfaces a large number of genuine
latent bugs plus a lot of noise from unannotated third-party libraries.

---

### Q5 — "Should this method return `Optional<List<Order>>`?"

**Mid-level answer:** "It's fine — it tells the caller there might be no orders."

**Senior answer:** "No. An empty list already means 'no orders', so `Optional<List<>>`
gives you two representations of the same state and forces every caller to handle both
— and callers will get it wrong in different ways, so some paths NPE and some render
'none'. Same rule for arrays, and for `Optional<Optional<T>>`. Return an empty
immutable list. The one place I would reconsider is if empty and absent genuinely mean
different things to the caller — 'no orders' versus 'this customer does not exist' —
and in that case I would not encode the distinction as a nesting, I would either throw
a `CustomerNotFoundException` or return a small result type that names both states,
probably a sealed interface (Topic 28)."

**What separates them:** stating the rule, then correctly identifying the one case
where the rule bends, and reaching for a sealed type rather than a nested `Optional`
when the domain genuinely has two absences.

**Interviewer's follow-up:** "What about `Optional<Boolean>`?" Same answer, more
strongly: three states expressed as a nested container is worse than a three-constant
enum. `Optional<Boolean>` also unboxes (Topic 01) and can NPE.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `Optional` has no compiler support, yet it is widely considered a good addition to
   Java. What exactly does it buy if it cannot enforce anything? Answer in terms of
   *who reads the signature* rather than in terms of safety.

2. Why did the JDK authors deliberately make `Optional` **not** implement
   `Serializable`? What message were they sending, and did it work?

3. `Optional.empty()` returns a shared singleton, so it allocates nothing.
   `Optional.of(x)` allocates. Given that, is "Optional allocates" a good reason to
   avoid it on a hot path? What would you need to measure before deciding — and which
   JIT optimisation might make the allocation disappear entirely?

4. TypeScript's `?.` short-circuits an entire chain on the first `null`. Java's
   `.map().map().map()` does the same thing. Given that, why does Java need both `map`
   and `flatMap` when TypeScript's `?.` needs no such distinction?

5. Consider `Optional<T>` as a *monad* (a container with `of` and `flatMap` obeying
   some laws). Name one other Java type you already know with the same shape, and one
   from Node. What does recognising the shared shape actually let you do?

6. A colleague proposes a lint rule: "every method that can return null must return
   `Optional` instead." Give the strongest argument for this rule, then the strongest
   argument against it. Which do you act on, and where exactly do you draw the
   boundary?

7. If Java added `strictNullChecks`-style enforcement tomorrow, would `Optional` still
   have a job? Be specific about what it would and would not still be for.

---

## Quick reference card

### Creation

```java
Optional.of(x)            // x must be non-null; NPEs immediately if it is
Optional.ofNullable(x)    // null becomes empty; the bridge from legacy APIs
Optional.empty()          // shared singleton, no allocation
```

### Extraction — in order of preference

```java
opt.map(f).orElse(fallback)          // 1. transform then default
opt.orElseGet(() -> compute())       // 2. lazy default
opt.orElseThrow(() -> new DomainEx()) // 3. absence is an error with meaning
opt.ifPresent(x -> sideEffect(x))    // 4. do something or nothing
opt.orElseThrow()                    // 5. an assertion. Rare and deliberate.
opt.get()                            // never. Lint it out of the codebase.
```

### Transformation

```java
opt.map(Product::name)               // T -> U            => Optional<U>
opt.flatMap(Product::promotion)      // T -> Optional<U>  => Optional<U>
opt.filter(Product::isActive)        // present -> empty if predicate fails
opt.or(() -> otherSource())          // Java 9+: fall back to another Optional
opt.stream()                         // Java 9+: 0-or-1 Stream, for flatMap in pipelines
opt.isEmpty()                        // Java 11+
```

### Where `Optional` belongs

| Position | Use it? | Instead |
|---|---|---|
| Method return type | **yes** — this is the whole design intent | — |
| Field | **no** | nullable field + `Optional` accessor |
| Constructor / method parameter | **no** | overloads, parameter object, or `@Nullable` |
| Record component | **no** | nullable component + `maybeX()` accessor |
| Collection element (`List<Optional<T>>`) | **no** | filter the empties out with `Optional::stream` |
| Wrapping a collection (`Optional<List<T>>`) | **no** | return an empty list |
| JPA entity attribute | **no** — it will not map | nullable column, `Optional` on the repository method |
| Map value | **no** | `map.getOrDefault`, or `Optional.ofNullable(map.get(k))` at the call site |

### Gotchas checklist

- [ ] Never `.get()`. Use `orElseThrow()` if you genuinely mean "assert present".
- [ ] `isPresent()` + `get()` is a null check with ceremony. Use `map`/`ifPresent`.
- [ ] `orElse(methodCall())` runs the method every time. Use `orElseGet`.
- [ ] `orElseThrow(() -> new X())` — supplier form, so the exception is built lazily.
- [ ] `Optional.of(mightBeNull)` throws NPE from inside `Optional`. Use `ofNullable`.
- [ ] `map` when the mapper returns `T`; `flatMap` when it returns `Optional<T>`.
- [ ] Never a field, never a parameter, never wrapping a collection.
- [ ] `Optional` is not `Serializable`, does not map in JPA, and its JSON shape
      depends on your Jackson module setup.
- [ ] `Optional` is a convention. JSpecify + NullAway is the enforcement.

---

## When would I use this at work?

**1. Designing a repository interface, before any code exists.**
`Optional<Product> findBySku(String)` versus `Product findBySku(String)` is a decision
you make once and every caller lives with. The `Optional` version puts "this can miss"
in the signature, so a new joiner cannot fail to see it — and Spring Data JPA
(Topic 47) already returns `Optional` from its derived finders, so you are matching the
idiom your framework uses. This is the single highest-value place `Optional` pays for
itself.

**2. Reviewing a pull request and seeing `.get()`.**
It is a one-second catch and it is always worth making. Ask the author what absence
*means* here: a 404, a default, or a skipped step. Those three answers map to
`orElseThrow(supplier)`, `orElse`/`orElseGet`, and `ifPresent`. The conversation is
more valuable than the fix, because it surfaces that nobody had decided.

**3. Chasing an unexplained query in an APM trace.**
You see a database call on a span where the code plainly should not make one. Grep for
`orElse(` in that call path. A method call inside `orElse` is the most common cause,
and it takes about thirty seconds to confirm once you know to look. Same instinct
applies to `getOrDefault(key, expensiveCall())` and `logger.debug("..." + expensive())`
— all three are the same argument-evaluation rule.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and autoboxing**: `Optional<Integer>` boxes, and
  `OptionalInt`/`OptionalLong`/`OptionalDouble` exist to avoid it.
- **21 — Lambdas and functional interfaces**: `orElseGet` takes a `Supplier`,
  `map` takes a `Function`, `filter` takes a `Predicate`. The laziness in this topic
  *is* the deferred-execution property of a lambda.
- **22 — Method references**: `Optional::stream`, `Product::name` — the forms you use
  constantly here.
- **23–24 — Streams and collectors**: `Optional` and `Stream` share `map`/`filter`/
  `flatMap`, and `flatMap(Optional::stream)` is the bridge between them. `findFirst`
  and `findAny` return `Optional`; `max`/`min`/`reduce` do too.

**This unlocks:**
- **27 — Records**: why a record component is never `Optional`, and how to expose an
  `Optional` accessor alongside a nullable component.
- **28 — Sealed types**: what you reach for when absence has *more than one meaning* —
  `Optional` can only say "nothing", a sealed result type can say *which* nothing.
- **29 — Pattern matching**: the modern alternative to unwrapping — deconstructing a
  sealed result instead of unwrapping an `Optional`.
- **46 — Error handling and `ProblemDetail`**: where `orElseThrow(() -> new
  ProductNotFoundException(sku))` becomes an HTTP 404 with a stable machine-readable
  body. `Optional` at the repository, a domain exception at the service, a
  `ProblemDetail` at the edge — that is the full chain.
- **47 — Spring Data JPA and projections**: Spring Data's derived query methods return
  `Optional<T>` natively; this is where you will use the topic every day. Also where
  `Optional` on an *entity field* fails to map.
- **117 — Sagas and compensations**: a saga step's outcome is not "value or nothing" —
  it is "succeeded, failed retryably, or failed permanently". That is a sealed result
  type, not an `Optional`, and recognising the difference is the point.

---

*Java baseline 21. `Optional` has been stable since Java 8; `or`, `ifPresentOrElse` and
`stream` arrived in Java 9, `orElseThrow()` no-arg in Java 10, and `isEmpty` in Java 11
— all long since final. Nothing here changed between 21 and 25. The moving part is the
tooling around nullability: JSpecify and NullAway are the active area, and Spring
Boot 4's portfolio-wide JSpecify adoption is the most consequential recent change.*
