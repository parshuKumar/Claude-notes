# 05 — Generics I: Declaration, Bounded Types, Generic Methods

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Think about shipping containers.

A container is a standard steel box. The box does not care what is inside it — that is
the whole point of containerisation. But the **manifest taped to the door** says
exactly what is inside: "1,200 kg refrigerated pharmaceuticals".

- **The container is the class.** `List`, `Repository`, `Cache`.
- **The manifest is the type parameter.** `List<Order>` — this box holds orders.
- **The customs officer at the port of loading is the compiler.** They read the
  manifest, check what you are putting in, and refuse the load if it does not match.
- **The crane operator is the JVM.** They never read the manifest. To them every box is
  a box. (That is Topic 06, and it explains most of the weird bits.)

Two refinements we carry through the doc:

1. **A bound is a requirement on the box, not on the contents.** "This box must be
   refrigerable" is `<T extends Refrigerable>`. It restricts which manifests are
   acceptable, and in exchange it lets the dockhands assume a plug socket exists.
2. **A bound on the shipping *line* is different from a bound on a single *voyage*.**
   "This company only ships refrigerated goods" is a class-level type parameter. "This
   particular sailing takes refrigerated goods" is a method-level one. Choosing the
   wrong level is the most common generics design mistake, and it is the mastery line
   for this topic.

---

## The bridge from what you know

### What transfers almost perfectly

TypeScript generics and Java generics look the same and mostly mean the same thing.

```ts
function firstOrThrow<T>(items: T[]): T {
  if (items.length === 0) throw new Error('empty');
  return items[0];
}

interface Repository<T, ID> {
  findById(id: ID): T | undefined;
  save(entity: T): T;
}

function largest<T extends { compareTo(other: T): number }>(items: T[]): T { ... }
```

```java
static <T> T firstOrThrow(List<T> items) {
    if (items.isEmpty()) throw new NoSuchElementException("empty");
    return items.get(0);
}

interface Repository<T, ID> {
    Optional<T> findById(ID id);
    T save(T entity);
}

static <T extends Comparable<T>> T largest(List<T> items) { ... }
```

**Verdict: HONEST ANALOGUE** for the basics. Type parameters, multiple type parameters,
constraints via `extends`, and inference at the call site all behave the way you expect.

### Where they diverge

| You know (TypeScript) | Java | Verdict |
|---|---|---|
| `<T>` on functions, classes, interfaces, type aliases | `<T>` on classes, interfaces, methods, constructors — **no type aliases exist** | **PARTIAL** |
| `T extends { compareTo(o: T): number }` — a *structural* constraint | `T extends Comparable<T>` — a *nominal subtype* constraint | **PARTIAL** — this is Topic 02 again. The candidate type must have **declared** `implements Comparable<T>`. Shape is not enough. |
| `<T = string>` default type parameters | — | **NO ANALOGUE.** Java has no defaults. You either supply the argument or use a raw type, which is a mistake (Trap 1). |
| `A & B` intersection types anywhere | `<T extends A & B>` — intersections only in a **bound** | **PARTIAL** |
| Conditional types, `infer`, mapped types, `keyof` | — | **NO ANALOGUE.** Java's type system cannot compute types from types. If you find yourself wanting `Pick<T, K>`, stop; the Java answer is a different class. |
| Generic constraints checked structurally at every call | Bounds checked nominally, once, at the declaration | **PARTIAL** |
| Type erasure at compile time | Type erasure at compile time | **HONEST ANALOGUE**, and it is the single closest match in the whole language — Topic 06 covers exactly how Java's differs. |
| Variance is inferred structurally | Invariance by default, wildcards at the use site | **PARTIAL** — Topic 07. `List<String>` is *not* a `List<Object>` in Java, and that surprises everyone. |
| `new T()` inside a generic function (via a constructor-typed parameter) | **impossible** | **NO ANALOGUE** — you must pass a `Class<T>` or a `Supplier<T>`. Topic 06 explains why. |

### The parts with no TypeScript equivalent at all

**NO TYPESCRIPT ANALOGUE.**

Two things in this topic have nothing to map onto.

**Default type parameters.** TypeScript's `<T = string>` has no Java counterpart. Java's
answer to "no type argument supplied" is the raw type, which is not a default — it is a
switch that turns generic checking off for the entire class (Trap 1). The reason is
historical rather than principled: raw types exist for Java 5 migration, and adding
defaults afterwards would have made "no argument written" ambiguous between "use the
default" and "I am legacy code".

**Computing a type from a type.** `Pick<T, K>`, `Partial<T>`, conditional types and
`infer` require the type system to *evaluate* — to take types as input and produce new
types as output. Java's type system does not compute; it only checks. This is not a
missing feature that might arrive later, it is a different kind of system. When you find
yourself reaching for a mapped type in Java, the answer is always a second class, a
projection interface, or a code generator — never a cleverer signature.

### The thing you must unlearn first

In TypeScript, a constraint describes a shape:

```ts
function total<T extends { amountMinor: number }>(items: T[]): number { ... }
total([{ amountMinor: 199 }, { amountMinor: 250 }]);   // works. Nothing declared anything.
```

In Java there is no such thing. `<T extends X>` means "T must be a subtype of X **by
declaration**". So the Java version of the above requires an interface *and* requires
every element type to have declared it:

```java
interface HasAmount { long amountMinor(); }

static <T extends HasAmount> long total(List<T> items) {
    long sum = 0;
    for (T item : items) sum += item.amountMinor();
    return sum;
}
```

If your element type is a vendor class that does not implement `HasAmount`, you are back
to Topic 02's adapter. That is not a generics limitation; it is nominal typing showing
through generics.

---

## What is this?

**Generics** let you parameterise a class, interface, method or constructor by a type,
so that one definition serves many types while the compiler still checks every use.

A **type parameter** (`<T>`) is a placeholder for a type, supplied at the point of use.
A **bounded type parameter** (`<T extends Number>`) restricts which types may be
supplied, and in exchange lets the body call the bound's methods.

A **generic method** declares its own type parameters, independently of the class it
lives in.

---

## Why does it matter?

1. **Without generics, every read from a collection is a cast, and every cast is a
   runtime bomb.** Pre-Java-5 code stored `Object` and cast on retrieval. The failure
   mode is `ClassCastException` at a line that reads a value, arbitrarily far from the
   line that stored the wrong one. Raw types (Trap 1) put you straight back there, and
   they are still legal, which is why this matters today and not just historically.

2. **The bound is where your API's usability is decided.** Over-bound and callers cannot
   use your method for the type they have, so they copy-paste it. Under-bound and your
   method body cannot do anything useful, so you cast inside — reintroducing the
   `ClassCastException`. Getting the bound right is the actual skill.

3. **Type parameters placed on the wrong level force callers to name types they should
   not have to.** `new Validator<Order>()` everywhere, one instance per type, extra
   allocations, extra Spring beans. Moving the parameter from the class to the method is
   often a one-line change that removes a whole category of friction.

---

## Syntax breakdown

Only what is genuinely new. Naming conventions first, because they are load-bearing for
readability: `T` type, `E` element, `K` key, `V` value, `R` result, `U`/`S` second and
third. Single capital letters. Multi-letter names like `TEntity` are a C# habit and will
read as a class name to a Java reader.

### A generic class

```java
public class Page<T> {

    private final List<T> items;
    private final int totalCount;

    public Page(List<T> items, int totalCount) {
        this.items      = List.copyOf(items);
        this.totalCount = totalCount;
    }

    public List<T> items()   { return items; }
    public int totalCount()  { return totalCount; }
}
```

| Bit of syntax | What it means |
|---|---|
| `class Page<T>` | `T` is in scope for the entire class body — fields, methods, constructors, nested types. |
| `Page(List<T> items, ...)` | The constructor uses `T` but does **not** re-declare it. It comes from the class. |
| `new Page<>(list, 42)` | The **diamond**. The compiler infers `T` from the target type on the left. Java 7+. |
| `new Page<Order>(list, 42)` | Explicit. Equivalent, more verbose, occasionally necessary when inference cannot see the target. |

### A generic method

```java
public static <T> Optional<T> firstMatching(Collection<T> items, Predicate<? super T> test) {
    for (T item : items) {
        if (test.test(item)) return Optional.of(item);
    }
    return Optional.empty();
}
```

| Bit of syntax | What it means |
|---|---|
| `<T>` **before** the return type | This is the declaration of `T`, and its scope is this method only. This position is the single most confusing bit of Java generics syntax for newcomers — the type parameter list goes after the modifiers and before the return type. |
| `static <T>` | A static method can be generic. It cannot use the *class's* type parameters (see Trap 3), so it declares its own. |
| `Predicate<? super T>` | A wildcard. Fully covered in Topic 07; for now read it as "a predicate that can accept a `T`". |
| Calling it: `firstMatching(orders, o -> o.total() > 100)` | No type argument written. Java infers `T = Order` from the argument. |
| Explicit type witness: `Repos.<Order>firstMatching(orders, test)` | Rarely needed. When inference fails — usually with nested generic calls or `null` arguments — this is how you tell the compiler directly. Note the odd placement: after the dot, before the method name. |

### Bounds

```java
// upper bound: T must be Number or a subtype
static <T extends Number> double sum(List<T> values) {
    double total = 0;
    for (T v : values) total += v.doubleValue();   // legal: the bound guarantees this method
    return total;
}

// multiple bounds: at most one class, then any number of interfaces
static <T extends OrderEntity & Auditable & Serializable> void archive(T entity) { ... }

// recursive (f-bounded) bound
static <T extends Comparable<T>> T largest(List<T> items) { ... }
```

| Bit of syntax | What it means |
|---|---|
| `extends` (not `implements`) | In a bound, `extends` is used for both classes and interfaces. There is no `implements` in a type-parameter bound. |
| `& Auditable & Serializable` | Multiple bounds. **The class bound, if any, must come first.** All the rest must be interfaces. |
| No `super` bound on a declaration | `<T super X>` does not exist. `super` appears only in wildcards (Topic 07). This asymmetry is real and worth remembering. |
| `T extends Comparable<T>` | A **recursive bound**: `T` appears inside its own bound. Read it as "T is comparable *to itself*". |
| Erasure note | `<T>` erases to `Object`; `<T extends Number>` erases to `Number`. The bound is not just a check — it changes the compiled signature. Topic 06. |

### The recursive bound, decoded

This one is worth slowing down for, because the mastery line for this topic is writing it
without copying it.

```java
static <T extends Comparable<T>> T largest(List<T> items) {
    T best = items.get(0);
    for (T candidate : items) {
        if (candidate.compareTo(best) > 0) best = candidate;   // needs compareTo(T)
    }
    return best;
}
```

Why `Comparable<T>` and not just `Comparable`?

- `<T extends Comparable>` uses a **raw type**, so `compareTo` takes `Object` and you
  lose all checking. The compiler will warn.
- `<T extends Comparable<T>>` says "T can be compared *to a T*", which is exactly what
  the loop needs.

Read it left to right in English: *"for any type T, where T knows how to compare itself
to another T"*. It is not a magic incantation; it is a sentence.

### `<T extends Comparable<? super T>>` — the version you should usually write

There is a failure case for the simple recursive bound, and it appears constantly with
inheritance:

```java
class Payment implements Comparable<Payment> { ... }
class CardPayment extends Payment { }          // inherits compareTo(Payment)

List<CardPayment> payments = ...;
largest(payments);      // does NOT compile with <T extends Comparable<T>>
```

The error:
```
error: method largest in class Utils cannot be applied to given types
  ... inference variable T has incompatible bounds
      equality constraints: CardPayment
      upper bounds: Comparable<CardPayment>
```

`CardPayment` is a `Comparable<Payment>`, not a `Comparable<CardPayment>`. Relaxing the
bound fixes it:

```java
static <T extends Comparable<? super T>> T largest(List<T> items) { ... }
```

Read: *"T can be compared to itself **or to any supertype of itself**"*. That is what
`Collections.sort` and `Comparator.naturalOrder()` actually declare, and it is the form
you should reach for. The `? super` machinery itself is Topic 07 — you are meeting the
motivation first, which is the right order.

---

## Example 1 — minimal

```java
import java.util.*;

public class GenericsBasics {

    // Generic method with a recursive bound. Written, not copied.
    static <T extends Comparable<? super T>> T largest(List<T> items) {
        if (items.isEmpty()) throw new NoSuchElementException("empty");
        T best = items.get(0);
        for (T candidate : items) {
            if (candidate.compareTo(best) > 0) best = candidate;
        }
        return best;
    }

    // Generic class: the type parameter is on the class because every member uses it.
    record Page<T>(List<T> items, int totalCount) {
        Page {
            items = List.copyOf(items);          // compact constructor; Topic 27
        }
    }

    public static void main(String[] args) {
        System.out.println(largest(List.of(3, 17, 8)));                  // T inferred = Integer
        System.out.println(largest(List.of("SKU-1", "SKU-9", "SKU-4"))); // T inferred = String

        Page<String> page = new Page<>(List.of("a", "b"), 2);            // diamond
        System.out.println(page.totalCount());

        // largest(List.of(new Object(), new Object()));   // does NOT compile — no bound match
    }
}
```

Uncomment the last line and read the error. It is long and it names the bound. Being able
to read that message calmly is most of what "knowing generics" means in practice.

---

## Example 2 — production scenario

`orderflow` has five aggregate types — `Order`, `Product`, `Inventory`, `Wallet`,
`Payment` — each with its own repository, and the same six methods keep being rewritten.

### The version that starts as reasonable and becomes unworkable

```java
public class OrderRepository {
    public Optional<Order> findById(long id)  { ... }
    public List<Order> findAll()              { ... }
    public Order save(Order order)            { ... }
    public void deleteById(long id)           { ... }
}
public class ProductRepository { /* the same four methods, copy-pasted */ }
public class WalletRepository  { /* the same four methods, copy-pasted */ }
```

Twenty near-identical methods. Then someone adds soft-delete to three of them and not the
other two, and now `deleteById` means different things in different repositories — a
correctness problem that reads as a maintenance problem.

### The generic version, with the bound doing real work

First, the constraint. Every aggregate has an identity and a version:

```java
package com.orderflow.core;

public interface AggregateRoot<ID> {
    ID id();
    long version();
}
```

```java
package com.orderflow.core;

/**
 * T is bounded so the repository body can actually use id() and version().
 * ID is a second parameter because orders use long and SKUs use String.
 */
public interface Repository<T extends AggregateRoot<ID>, ID> {

    Optional<T> findById(ID id);
    List<T> findAll();
    T save(T entity);
    void deleteById(ID id);

    /** Default: expressible purely in terms of the abstract methods. See Topic 04. */
    default T getById(ID id) {
        return findById(id).orElseThrow(() ->
            new EntityNotFoundException(getClass().getSimpleName() + " id=" + id));
    }
}
```

```java
package com.orderflow.orders;

public interface OrderRepository extends Repository<Order, Long> {
    List<Order> findByCustomerIdAndStatus(CustomerId customerId, OrderStatus status);
}
```

Note what the bound bought. Because `T extends AggregateRoot<ID>`, this compiles inside a
shared base implementation:

```java
public abstract class JdbcRepository<T extends AggregateRoot<ID>, ID>
        implements Repository<T, ID> {

    @Override
    public T save(T entity) {
        if (entity.version() == 0) {            // legal ONLY because of the bound
            return insert(entity);
        }
        return updateWithOptimisticLock(entity, entity.version());
    }

    protected abstract T insert(T entity);
    protected abstract T updateWithOptimisticLock(T entity, long expectedVersion);
}
```

Remove `extends AggregateRoot<ID>` from the declaration and `entity.version()` stops
compiling, because `T` would erase to `Object`. **That is the bound earning its keep**:
it is not documentation, it is what makes the body possible.

### Now the level question — the actual mastery point

Add a bulk operation. Where does the type parameter go?

**Wrong level — on the class:**
```java
public class BulkImporter<T extends AggregateRoot<?>> {
    public int importAll(List<T> entities, Repository<T, ?> repo) { ... }
}
```
Callers must now write:
```java
new BulkImporter<Order>().importAll(orders, orderRepo);
new BulkImporter<Product>().importAll(products, productRepo);
new BulkImporter<Wallet>().importAll(wallets, walletRepo);
```
Three objects, three explicit type arguments, and in Spring, three beans or a prototype
scope — because one `BulkImporter<Order>` bean cannot serve products. You have made the
caller's life worse to hold a type parameter the class does not otherwise use.

**Right level — on the method:**
```java
public final class BulkImporter {

    private final TransactionTemplate tx;

    public BulkImporter(TransactionTemplate tx) { this.tx = tx; }

    public <T extends AggregateRoot<ID>, ID> int importAll(
            List<T> entities, Repository<T, ID> repo, int batchSize) {

        int written = 0;
        for (List<T> batch : partition(entities, batchSize)) {
            written += tx.execute(status -> {
                batch.forEach(repo::save);
                return batch.size();
            });
        }
        return written;
    }
}
```
Callers write:
```java
importer.importAll(orders,   orderRepo,   500);
importer.importAll(products, productRepo, 500);
```
No type arguments. One singleton bean. The type variable exists for exactly the duration
of the call, which is the only duration it was ever needed for.

**The rule, stated plainly:**

> **Put the type parameter on the class only if the class's *state* is parameterised by
> it.** `Page<T>` holds a `List<T>` — the field needs it, so it belongs on the class.
> `BulkImporter` holds a `TransactionTemplate` — nothing in its state mentions `T`, so
> `T` belongs on the method.

The diagnostic question: *does any field of this class mention `T`?* If no, move it to
the method.

### The `ID` parameter — a judgement call, stated honestly

`Repository<T, ID>` has two parameters and reads heavily. Spring Data does exactly this
(`JpaRepository<T, ID>`) and it is the right call when identifier types genuinely differ
— which in `orderflow` they do: `Order` uses `Long`, `Product` uses a `Sku` record.

If every aggregate in your system used `Long`, the second parameter would be pure
ceremony and you should drop it. Generic parameters are not free: each one is another
thing a reader must track and another place inference can fail. Two is comfortable, three
is a smell, four means you are modelling something the type system is not going to help
you with.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — raw types

**Wrong:**
```java
List orders = new ArrayList();      // no type argument at all
orders.add(new Order(1L));
orders.add("SKU-4471");             // compiles fine — it is a raw List

for (Object o : orders) {
    Order order = (Order) o;        // explodes on the second element
    process(order);
}
```

**Exact symptom:** first a compiler warning you probably have suppressed:
```
Note: Main.java uses unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.
```
then, at runtime:
```
java.lang.ClassCastException: class java.lang.String cannot be cast to
  class com.orderflow.orders.Order (java.lang.String is in module java.base
  of loader 'bootstrap'; com.orderflow.orders.Order is in unnamed module
  of loader 'app')
```
The stack trace points at the **read**, not at the bad `add`. In a service where the list
is populated in one request and drained in another, those can be minutes and several
classes apart.

**Root cause:** a raw type turns off generic checking. And it does so far more
aggressively than people expect — using a raw type erases the generics of the *entire*
class, including methods whose signatures have nothing to do with the class's type
parameter:

```java
List<String> names = new ArrayList<>();
List raw = names;
Iterator<String> it = raw.iterator();     // WARNING: raw.iterator() returns raw Iterator
```

**Fix:** never write a raw type. Turn the warning into an error in your build:
```
javac -Xlint:rawtypes,unchecked -Werror ...
```
For Maven, `<compilerArgs>` on `maven-compiler-plugin`; for Gradle,
`options.compilerArgs`. Doing this on a legacy codebase produces a lot of output — fix
the new code and ratchet.

> `[LEGACY — still asked]` Raw types exist only for Java 5 migration compatibility. You
> will still meet them in old code and in interview questions. Recognise them, never
> write them.

---

### Trap 2 — the type parameter on the wrong level

**Wrong:**
```java
public class OrderValidator<T extends AggregateRoot<?>> {
    public List<Violation> validate(T entity) { ... }   // T used ONLY here
}
```

**Exact symptom:** not an exception — an API that is annoying in a way people work around.
Callers write `new OrderValidator<Order>()` at every use site. In Spring you get:
```
org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean
  of type 'com.orderflow.validation.OrderValidator<com.orderflow.catalog.Product>'
  available
```
because you registered `OrderValidator<Order>` as a bean and Spring resolves generics on
injection points. The observable outcome is a startup failure, or — if someone "fixes" it
with a raw-typed bean — a `ClassCastException` at runtime.

**Root cause:** the type variable's scope is wider than its use. No field of the class
mentions `T`.

**Fix:** move it.
```java
public class Validator {
    public <T extends AggregateRoot<?>> List<Violation> validate(T entity) { ... }
}
```
One singleton bean, no type argument at the call site, inference does the work.

---

### Trap 3 — using a class type parameter from a static member

**Wrong:**
```java
public class Cache<K, V> {
    private final Map<K, V> entries = new HashMap<>();

    public static <K, V> Cache<K, V> empty() { return new Cache<>(); }   // fine

    public static Cache<K, V> broken() { return new Cache<>(); }         // NOT fine
}
```

**Exact symptom:**
```
error: non-static type variable K cannot be referenced from a static context
```

**Root cause:** `K` and `V` are bound per *instance*. A static member belongs to the
class, of which there is exactly one at runtime regardless of type arguments — because of
erasure (Topic 06), `Cache<String,Order>` and `Cache<Long,Wallet>` are the same class
object. There is no instance to take `K` from.

**Fix:** declare fresh type parameters on the static method itself, as `empty()` does.
Note that the `K, V` in `empty()` are *different variables* that happen to share names —
a fact worth knowing when you are reading someone else's code and wondering why they
shadow.

---

### Trap 4 — over-bounding

**Wrong:**
```java
// "All my entities extend OrderEntity, so I'll bound on that."
public <T extends OrderEntity> String describe(T entity) {
    return entity.id() + ":" + entity.version();
}
```

**Exact symptom:** six months later, you need the same helper for `Payment`, which does
not extend `OrderEntity`. The compiler says:
```
error: method describe in class Describer cannot be applied to given types;
  required: T
  found:    Payment
  reason: inference variable T has incompatible bounds
      equality constraints: Payment
      upper bounds: OrderEntity
```
So someone copy-pastes the method with a different bound. Now there are two, and the next
person adds a third. The observable cost is duplicated logic drifting apart — the version
in `describe` gets a fix that the copy in `describePayment` does not, and a log line
somewhere is silently wrong.

**Root cause:** the bound named a *class in your hierarchy* rather than the *capability
the body actually needs*. The body needs `id()` and `version()` — nothing else.

**Fix:** bound on the smallest interface that makes the body compile.
```java
public <T extends AggregateRoot<?>> String describe(T entity) { ... }
```
The discipline: write the body first, then set the bound to the minimum that compiles.
This is the Interface Segregation Principle showing up as a type-system decision — you
already know the principle; here is where Java makes you spend it.

---

### Trap 5 — the recursive bound that rejects your subclass

**Wrong:**
```java
static <T extends Comparable<T>> T largest(List<T> items) { ... }

class Payment implements Comparable<Payment> { ... }
class CardPayment extends Payment { }

List<CardPayment> cards = ...;
CardPayment best = largest(cards);
```

**Exact symptom:**
```
error: method largest in class Utils cannot be applied to given types;
  reason: inference variable T has incompatible bounds
      equality constraints: CardPayment
      upper bounds: Comparable<CardPayment>
```
The message is long and mentions "inference variable", which sends people to Stack
Overflow rather than to the bound.

**Root cause:** `CardPayment` inherits `compareTo(Payment)`, so it is a
`Comparable<Payment>` and *not* a `Comparable<CardPayment>`. Your bound demanded the
latter.

**Fix:**
```java
static <T extends Comparable<? super T>> T largest(List<T> items) { ... }
```
"Comparable to itself or any supertype of itself." This is what `Collections.sort`,
`Collections.max` and `Comparator.naturalOrder` all declare. Copy the JDK here — not
because the JDK is always right, but because this particular shape has been beaten on for
twenty years.

**How to recognise it in the wild:** any error containing "inference variable T has
incompatible bounds" where one bound is a `Comparable<Something>` is this bug. Go add
`? super`.

---

## Hands-on proof

Every command below is one **you** run. I do not have a JVM and will not fabricate
output.

### Setup

```bash
mkdir -p ~/java-lab/05 && cd ~/java-lab/05
java --version
```

### Proof 1 — raw types disable more than you think

`RawDamage.java`:
```java
import java.util.*;

public class RawDamage {
    public static void main(String[] args) {
        List<String> names = new ArrayList<>(List.of("a", "b"));

        List raw = names;                       // raw type
        Iterator it = raw.iterator();           // note: raw Iterator, not Iterator<String>

        raw.add(42);                            // compiles: raw list accepts anything

        for (String s : names) {                // ClassCastException expected here
            System.out.println(s.length());
        }
    }
}
```

```bash
javac -Xlint:unchecked,rawtypes RawDamage.java
java RawDamage
```

**What to look for:** warnings at compile time, then a failure at run time.

| What you see | What it means |
|---|---|
| Warnings naming `unchecked call to add(E)` and `found raw type: List` | The compiler told you. This is why `-Werror` on these two lints is worth the noise. |
| `java.lang.ClassCastException: class java.lang.Integer cannot be cast to class java.lang.String` on the **for** line, not the `add` line | The key observation. The compiler inserted a cast at the *read*, so the blast site is far from the crime scene. Topic 06 shows you that cast in bytecode. |
| No warnings at all | Your build is suppressing lints, or you are on an unusual compiler. Check for `@SuppressWarnings` and for `-nowarn`. |

### Proof 2 — the bound is what makes the body compile

`BoundMatters.java`:
```java
import java.util.*;

public class BoundMatters {
    interface HasAmount { long amountMinor(); }
    record Line(long amountMinor) implements HasAmount { }

    // Version A — no bound.
    static <T> long sumA(List<T> items) {
        long total = 0;
        for (T item : items) total += item.amountMinor();   // expect a compile error
        return total;
    }

    // Version B — bounded.
    static <T extends HasAmount> long sumB(List<T> items) {
        long total = 0;
        for (T item : items) total += item.amountMinor();   // expect this to compile
        return total;
    }
}
```

```bash
javac BoundMatters.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| Exactly one error, on `sumA`: `error: cannot find symbol ... symbol: method amountMinor() location: variable item of type T` | The expected result. Unbounded `T` erases to `Object`, so only `Object`'s methods are available. |
| Errors on both | You wrote the bound wrong — check it is `extends HasAmount`, not `implements`. |
| No errors | You accidentally bounded `sumA` too. |

**How to read it:** the bound is not a comment. Delete it and the body stops compiling.
That is the shortest possible statement of what a bound is *for*.

### Proof 3 — see the erased signature the bound produces

```bash
javac BoundMatters.java
javap -s BoundMatters.class
```

**What to look for:** the `descriptor:` line under each method.

| What you see | What it means |
|---|---|
| `sumB` has a descriptor mentioning `java/util/List` and returning `J` (long) | Expected. The erased signature. Note the type parameter itself is gone from the descriptor. |
| A separate `Signature:` attribute (visible with `javap -v`) carrying the generic form | Also expected. Generic information *is* in the class file — it just is not used by the JVM for checks. This is the precise thing that makes Java's erasure different from TypeScript's, and it is Topic 06. |

Run the same on a class with `<T>` unbounded and one with `<T extends Number>`, and
compare the descriptors. The bound changes the erasure — `Object` versus `Number` — which
is a fact you will need in Topic 06 for bridge methods.

> If your `javap` wording differs from what I describe, trust your output. Reading class
> files properly is Topic 76.

### Proof 4 — the recursive-bound failure, and the fix

`RecursiveBound.java`:
```java
import java.util.*;

public class RecursiveBound {

    static class Payment implements Comparable<Payment> {
        final long amountMinor;
        Payment(long a) { this.amountMinor = a; }
        @Override public int compareTo(Payment o) { return Long.compare(amountMinor, o.amountMinor); }
    }
    static class CardPayment extends Payment {
        CardPayment(long a) { super(a); }
    }

    static <T extends Comparable<T>> T largestStrict(List<T> items) {
        T best = items.get(0);
        for (T c : items) if (c.compareTo(best) > 0) best = c;
        return best;
    }

    static <T extends Comparable<? super T>> T largestRelaxed(List<T> items) {
        T best = items.get(0);
        for (T c : items) if (c.compareTo(best) > 0) best = c;
        return best;
    }

    public static void main(String[] args) {
        List<CardPayment> cards = List.of(new CardPayment(199), new CardPayment(999));
        System.out.println(largestRelaxed(cards).amountMinor);   // expect OK
        System.out.println(largestStrict(cards).amountMinor);    // expect a compile error
    }
}
```

```bash
javac RecursiveBound.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| One error on the `largestStrict` call mentioning `inference variable T has incompatible bounds` | The expected result, and the error text you should learn to recognise on sight. `CardPayment` is a `Comparable<Payment>`, not a `Comparable<CardPayment>`. |
| Both calls compile | `CardPayment` declares its own `compareTo(CardPayment)`. Check the class. |
| An error on `largestRelaxed` too | The `? super` is mistyped, or `Payment` does not implement `Comparable` at all. |

Now comment out the `largestStrict` call and run. The relaxed version works, and that is
the form you should write from now on.

### Proof 5 — inference, and where it gives up

`Inference.java`:
```java
import java.util.*;

public class Inference {
    static <T> List<T> pairOf(T a, T b) { return List.of(a, b); }

    public static void main(String[] args) {
        var a = pairOf("SKU-1", "SKU-2");                 // T = String
        var b = pairOf("SKU-1", 42);                      // T = ?  -- look at what it picks
        List<String> c = pairOf(null, null);              // inference from the target type
        var d = Inference.<String>pairOf(null, null);     // explicit type witness

        System.out.println(a + " " + b + " " + c + " " + d);
    }
}
```

```bash
javac -Xlint:all Inference.java
```

**What to look for:** whether line `b` compiles, and what type it produced.

| What you see | What it means |
|---|---|
| It compiles, and `b`'s type is an intersection like `Serializable & Comparable<...>` | Expected. Java infers the **least upper bound** of `String` and `Integer`, which is a synthesised intersection type you cannot write by hand. Hover it in your IDE — the answer is instructive and slightly alarming. |
| Line `c` compiles | Expected. Inference uses the *target type* on the left when the arguments carry no information. This is the same mechanism as lambda target typing (Topic 02). |
| An error on `c` or `d` | Check your syntax on the type witness: it is `ClassName.<T>method(...)`, with the type argument after the dot. |

**How to read it:** Java's inference is stronger than people assume — it reads the target
type, not just the arguments. But it will silently produce a type you did not intend
(line `b`) rather than refuse. `var` makes that invisible, which is one concrete argument
against `var` on a generic call (Topic 30).

---

## Practice exercises

### 1 — Easy: write the recursive bound from scratch

Without copying from this document or the internet:

1. Write `static <T ...> T largest(List<T> items)` that returns the greatest element.
   Get the bound right on your own. Then test it with `List<Integer>`, `List<String>`, and
   a `List<CardPayment>` where `CardPayment extends Payment implements Comparable<Payment>`.
2. When the third case fails, read the error, name the reason in one sentence, and fix
   the bound.
3. Now write the same thing taking a `Comparator<T>` instead of requiring `Comparable`.
   Which version has the better API, and under what circumstances does your answer flip?
4. Finally: make `largest` return `Optional<T>` instead of throwing on empty. What did
   that change about the caller's code? (Topic 26 argues about this properly; give your
   view now and revisit it later.)

### 2 — Medium: fix the API (combines Topics 02, 03, 04)

Here is a real-shaped `orderflow` utility class with **six** defects — four from this
topic, one from Topic 02, one from Topic 03.

```java
public class EntityHelper<T extends OrderEntity> {

    public static Map CACHE = new HashMap();

    public List describe(List entities) {
        List out = new ArrayList();
        for (Object e : entities) {
            out.add(((OrderEntity) e).getId() + ":" + ((OrderEntity) e).getVersion());
        }
        return out;
    }

    public T findLargest(List<T> entities) {
        T best = entities.get(0);
        for (T e : entities) {
            if (((Comparable) e).compareTo(best) > 0) best = e;
        }
        return best;
    }

    public static T identity(T value) { return value; }

    public Map<String, Object> summarise(String customerId, String orderId) {
        return Map.of("customer", customerId, "order", orderId);
    }
}
```

For each defect: name it, give the **observable symptom** (compile error text, exception
message shape, or a business consequence — not "bad practice"), and then rewrite the
whole class. Your rewrite must:
- have no raw types and produce zero warnings under `-Xlint:unchecked,rawtypes`;
- put each type parameter at the correct level, and justify each choice in one line;
- bound on capability, not on your class hierarchy;
- fix the Topic 02 defect with nominal types;
- fix the Topic 03 defect by narrowing access, and say what breaks for callers if it was
  already published.

### 3 — Hard: production simulation — a generic outbox

`orderflow` needs a transactional outbox: domain events are written to a table in the
same transaction as the business change, then published asynchronously. (The full outbox
pattern is Topic 115; today you are building the *type design*, not the delivery
guarantees.)

**Part A.** Design the types.
- `interface DomainEvent { String aggregateId(); Instant occurredAt(); }`
- `OrderPlaced`, `PaymentCaptured`, `InventoryReserved` records implementing it.
- `interface OutboxWriter` with a method that accepts any `DomainEvent`.
- `interface EventHandler<E extends DomainEvent> { void handle(E event); }`

Decide, for each type parameter you introduce: class level or method level? Write one
line of justification per decision using the "does a field mention T?" test.

**Part B.** Write a `HandlerRegistry` that stores handlers and dispatches an incoming
`DomainEvent` to the right one.

You will hit a wall: `Map<Class<? extends DomainEvent>, EventHandler<?>>` cannot be
dispatched from without an unchecked cast. Try it. Capture the exact compiler error and
the exact `@SuppressWarnings("unchecked")` you end up needing.

Then answer: **why is this cast actually safe, and what invariant are you promising the
compiler that it cannot verify?** Write that invariant as a comment above the
suppression. (This is the single most common legitimate use of `@SuppressWarnings` in
production Java, and being able to justify it is what separates a reviewer who approves
it from one who does not.)

**Part C.** Now make the wall visible. Deliberately register a handler under the wrong
key:
```java
registry.register(OrderPlaced.class, (EventHandler) paymentCapturedHandler);
```
Dispatch an `OrderPlaced`. Capture the exact exception and its stack trace. Note **which
frame** it points at, and how far that is from the `register` call that caused it.

**Part D.** Fix it so that misregistration is impossible. There are at least two
approaches:
1. Make `register` generic — `<E extends DomainEvent> void register(Class<E>, EventHandler<E>)`
   — so the compiler ties key and handler together.
2. Have the handler declare its own event type — `Class<E> eventType()` — so there is no
   key to get wrong.

Implement both. Then argue which you would ship and why, including what each one costs
when someone needs a handler for two event types.

**Part E.** Argue against generics here. Under what circumstances would you skip all of
this and use `Object` plus a `switch` on a sealed interface (Topic 28)? Give a concrete
condition, and say what evidence would tell you which situation you are in.

---

## Interview questions

### Q1 — "What do generics actually give you?"

**Mid-level answer:** "Type safety — you don't have to cast, and you can't put the wrong
thing in a collection."

**Senior answer:** "Compile-time checking of a contract that would otherwise be a runtime
cast. Before generics you stored `Object` and cast on retrieval, so the failure was a
`ClassCastException` at the *read*, arbitrarily far from the bad write — and that is
still exactly what happens today if you use a raw type. But the more useful framing for
design is: the type parameter is a parameter, and the **bound is its contract**. An
unbounded `T` erases to `Object`, so the body can only call `Object` methods. The moment
you write `<T extends AggregateRoot<ID>>`, the body can call `id()` and `version()`. The
bound is not documentation — delete it and the body stops compiling. Getting the bound
minimal is what makes the API reusable, and getting it on the right level — class versus
method — is what keeps callers from having to name types."

**What separates them:** framing the bound as the contract, connecting it to erasure, and
naming the level decision as a distinct design axis.

**Follow-up:** "So when does the bound belong on the class?" They want the test: only when
the class's *state* is parameterised by it.

---

### Q2 — "Class-level or method-level type parameter? How do you decide?"

**Mid-level answer:** "If several methods use it, put it on the class."

**Senior answer:** "The test I use is: does any *field* of the class mention `T`?
`Page<T>` holds a `List<T>`, so it belongs on the class. A `BulkImporter` that holds only
a `TransactionTemplate` should not be generic at all — the type parameter belongs on the
method, because that is the only scope in which it is needed. Getting this wrong is not
just verbose: it forces callers to write `new Validator<Order>()` at every use site, and
in Spring it means one bean per type argument, so you get a
`NoSuchBeanDefinitionException` naming a parameterised type at startup. Moving it to the
method is usually a one-line change that removes the whole problem. Several methods
sharing a `T` is a hint, not a rule — if they share it only in their signatures and not in
any state, each one can declare its own."

**What separates them:** a concrete, checkable test rather than a heuristic, and knowing
the Spring failure mode that makes it a real bug rather than a style preference.

**Follow-up:** "Give me a class where the answer is genuinely ambiguous." A good answer
reaches for something like a builder or a `Comparator`-holding sorter, and reasons about
it rather than declaring.

---

### Q3 — "Explain `<T extends Comparable<T>>`. Why is `T` inside its own bound?"

**Mid-level answer:** "It means T has to be comparable. The `<T>` inside makes it compare
to its own type instead of `Object`."

**Senior answer:** "It is a recursive — f-bounded — bound, and it reads as 'T is a type
that knows how to compare itself to a T'. The alternative, `<T extends Comparable>`, is a
raw type, so `compareTo` takes `Object` and you have lost the checking you came for. But
in practice I write `<T extends Comparable<? super T>>`, because the strict form breaks on
inheritance: if `CardPayment extends Payment` and `Payment implements Comparable<Payment>`,
then `CardPayment` is a `Comparable<Payment>` and not a `Comparable<CardPayment>`, so the
strict bound rejects it with an 'inference variable T has incompatible bounds' error.
`? super T` says 'comparable to itself or any supertype', which is what
`Collections.sort` and `Comparator.naturalOrder` actually declare. The same recursive
shape shows up in self-typed builders — `<SELF extends Builder<SELF>>` — so that
`withX()` returns the subclass type rather than the base."

**What separates them:** knowing the inheritance failure case *and* the fix, recognising
the error message, and generalising the recursive-bound pattern to builders.

**Follow-up:** "Write the self-typed builder signature." They are checking you can produce
`abstract class Builder<SELF extends Builder<SELF>>` and explain the unchecked
`(SELF) this` cast it needs.

---

### Q4 — "Why is this a compile error?"

```java
public class Cache<K, V> {
    public static Cache<K, V> empty() { return new Cache<>(); }
}
```

**Mid-level answer:** "You can't use the class's generics in a static method — you have
to declare them on the method."

**Senior answer:** "`non-static type variable K cannot be referenced from a static
context`. The reason is erasure: `Cache<String, Order>` and `Cache<Long, Wallet>` are the
*same* class at runtime, so there is exactly one `Cache.class` and one copy of every
static member. `K` and `V` are bound per instantiation of the *type*, which has no runtime
existence — there is no instance to read them from. The fix is to declare fresh
parameters on the method: `public static <K, V> Cache<K, V> empty()`. Those are different
variables that happen to share names, which is worth knowing when you are reading someone
else's code and wondering about the shadowing."

**What separates them:** giving the erasure-based reason rather than the rule, and knowing
that the method's `K, V` are genuinely different variables.

**Follow-up:** "So can a static field be of type `T`?" No, for the same reason — and
this leads naturally into Topic 06.

---

### Q5 — "You need a generic method to create an instance of `T`. How?"

**Mid-level answer:** "You'd use reflection — `T.class.newInstance()` or something like
that."

**Senior answer:** "You can't write `new T()` at all, because `T` is erased and there is
no constructor to call at runtime. There are three honest options. First, pass a
`Supplier<T>` — my default, because it is compile-time safe, works with lambdas and
constructor references, and handles constructors with arguments. Second, pass a
`Class<T>` token and use `type.getDeclaredConstructor().newInstance()` — necessary when
you also need the `Class` for something else, like a Jackson binding or a cast, but it
moves every failure to runtime as a `ReflectiveOperationException` and requires a
no-argument constructor. Third, in a framework, `ParameterizedTypeReference` or a
super-type token, which recovers the type argument from an anonymous subclass's generic
supertype — that is how `RestTemplate.exchange` and Jackson's `TypeReference` work.
`newInstance()` on `Class` directly is deprecated since Java 9, because it propagated
checked exceptions in a way that defeated the compiler."

**What separates them:** three options with trade-offs, knowing the deprecation, and
knowing the super-type-token trick that makes frameworks work.

**Follow-up:** "How does `ParameterizedTypeReference` recover the type if it's erased?"
This is the gateway to Topic 06 — the generic supertype *is* retained in the class file's
`Signature` attribute, which is exactly the thing TypeScript has no equivalent of.

---

## Mental model checkpoint

1. `<T>` erases to `Object` and `<T extends Number>` erases to `Number`. Given that the
   bound changes the compiled signature, what happens to *binary compatibility* if you
   relax a bound from `Number` to `Object` in a published library? Reason it out before
   Topic 06 tells you.

2. Java has `<T extends X>` but not `<T super X>`. Wildcards have both. Construct a
   situation where a lower-bounded type *parameter* would be useful, then work out what
   the compiler would not be able to do with it.

3. The "does a field mention `T`?" test decides class versus method level. Find a case
   where it gives the wrong answer, and say what better test you would use there.

4. TypeScript checks `T extends { compareTo(o: T): number }` structurally; Java checks
   `T extends Comparable<T>` nominally. Name a concrete `orderflow` situation where the
   structural version is genuinely better, and one where the nominal version is.

5. Multiple bounds require the class bound first: `<T extends Payment & Auditable>`. Why
   does the order matter to the compiler, given that erasure only keeps the first bound?

6. You are designing a public API. Which is the more expensive mistake to fix later — a
   bound that is too tight, or one that is too loose? Argue it in terms of who has to
   change code.

7. Generic type parameters are conventionally one capital letter. That is unusually
   terse for Java, which is otherwise verbose. Why did this convention survive, and what
   does it tell you about how type parameters are meant to be read?

---

## Quick reference card

### Syntax

```java
class Page<T> { ... }                              // generic class
interface Repository<T extends AggregateRoot<ID>, ID> { ... }   // bounded + 2 params

static <T> T first(List<T> xs) { ... }             // generic method: <T> before return type
static <T extends Number> double sum(List<T> xs) { ... }        // upper bound
static <T extends Payment & Auditable> void x(T t) { ... }      // multiple: class first
static <T extends Comparable<? super T>> T max(List<T> xs) { }  // the bound to memorise

new Page<>(items, 2)                               // diamond — infer from target
Utils.<String>pairOf(null, null)                   // explicit type witness
```

### Naming conventions

| Letter | Means |
|---|---|
| `T` | Type |
| `E` | Element (collections) |
| `K`, `V` | Key, Value |
| `R` | Result / return |
| `U`, `S` | Second, third type |
| `SELF` | Self type in a recursive builder bound |

### Decision table

| Question | Answer |
|---|---|
| Where does the type parameter go? | On the class **only** if a field mentions it. Otherwise on the method. |
| What bound do I use? | Write the body first; use the smallest bound that makes it compile. |
| `Comparable<T>` or `Comparable<? super T>`? | Almost always `? super T`. Copy the JDK. |
| Static method needs `T`? | Declare a fresh `<T>` on the method. Class parameters are unavailable. |
| Need `new T()`? | `Supplier<T>` first, `Class<T>` token second. Never possible directly. |
| Two type parameters? | Fine. Three is a smell. Four means rethink. |

### Gotchas checklist

- [ ] Never write a raw type. Enable `-Xlint:rawtypes,unchecked`.
- [ ] A raw type erases the generics of the **whole class**, not just the one parameter.
- [ ] `<T super X>` does not exist on declarations. Only on wildcards.
- [ ] Multiple bounds: class first, then interfaces, joined by `&`.
- [ ] Static members cannot use class type parameters.
- [ ] Prefer `<T extends Comparable<? super T>>` over `<T extends Comparable<T>>`.
- [ ] "inference variable T has incompatible bounds" almost always means you need
      `? super`.
- [ ] `var` on a generic call hides an inferred type you may not have intended.
- [ ] Bound on capability (an interface), never on your class hierarchy.
- [ ] Every `@SuppressWarnings("unchecked")` needs a comment stating the invariant you
      are promising.

---

## When would I use this at work?

**1. Writing any shared abstraction in a service codebase.**
A repository base, a paged response, a retry helper, an event dispatcher — all of these
want a type parameter, and the level decision comes up every single time. The "does a
field mention T?" test takes five seconds and prevents a class of API friction that is
expensive to undo once callers exist.

**2. Reviewing a PR that adds `@SuppressWarnings("unchecked")`.**
Sometimes it is correct — a heterogeneous registry keyed by `Class<T>` genuinely cannot
be expressed in Java's type system. Your job in review is to ask what invariant the
author is promising and to require it in a comment. An unjustified suppression is a
`ClassCastException` scheduled for a future date.

**3. Reading a compiler error nobody else on the team can parse.**
"inference variable T has incompatible bounds" makes people give up and change the
signature semi-randomly. Recognising it as the `? super` case turns a half-day into two
minutes, and it happens most often exactly when someone introduces a subclass — which is
to say, during refactors, under time pressure.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and wrappers**: `List<int>` is illegal; you need to be comfortable
  that boxing is why.
- **02 — Nominal typing**: `<T extends HasAmount>` requires a *declared* implementation,
  not a matching shape. This is the difference from TypeScript generics.
- **04 — Interfaces and abstract classes**: bounds name interfaces, and generic interfaces
  with `default` methods are where the two topics meet.

**This unlocks:**
- **06 — Type erasure**: why `new T[]` is illegal, why static members cannot see `T`, and
  what the compiler generates at the seam between a generic interface and its
  implementor.
- **07 — Variance, PECS, wildcard capture**: `? extends` and `? super` in full. Today you
  met `? super T` as a fix; Topic 07 explains why it works.
- **10–16 — Collections**: every collection interface is generic; the JDK's declarations
  are the best available worked examples of bound placement.
- **21–24 — Lambdas, method references and streams**: `Function<T,R>`, `Predicate<T>`,
  `Collector<T,A,R>` — you will read those signatures without effort now.
- **26 — Optional**: `Optional<T>` and why `flatMap`'s signature looks the way it does.
- **47 — Spring Data JPA**: `JpaRepository<T, ID>` is exactly the design from Example 2,
  and Spring resolves those type arguments at runtime via the `Signature` attribute.

---

*Java baseline 21. Generics arrived in Java 5 and the declaration syntax has not changed
since. Java 7 added the diamond, Java 8 improved target-type inference substantially, and
Java 9 allowed the diamond on anonymous classes. Nothing in this topic differs between 21
and 25. Project Valhalla would eventually allow generics over primitives, which would
remove the boxing cost from Topic 01 — it has not landed, and the declaration syntax you
learn here is expected to survive it.*
