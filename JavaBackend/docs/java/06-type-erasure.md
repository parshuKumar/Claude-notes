# 06 — Generics II: Type Erasure, Bridge Methods, Reifiable Types

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Back to the shipping containers from Topic 05.

At the **port of loading**, a customs officer reads the manifest taped to the door —
"1,200 kg refrigerated pharmaceuticals" — and checks that what you are loading matches.
Nothing gets on the ship unless the paperwork agrees. That officer is `javac`.

Then the container is loaded, and **the manifest is peeled off the door**. On the ship,
in the yard, on the truck — every container is just a steel box. The crane operator
cannot tell a box of pharmaceuticals from a box of engine parts. That is the JVM.

Three refinements we carry through the whole doc:

1. **A photocopy of the manifest is filed in the ship's office.** It is not on the door
   and the crane operator never looks at it, but it is on board and you can go and read
   it. That is the class file's `Signature` attribute — and it is the single biggest
   difference between Java's erasure and TypeScript's, where nothing is filed anywhere.
2. **At the port of unloading, someone opens the box and checks the contents.** Not
   because the paperwork said so — because the receiving warehouse has to put it
   somewhere. That check is the `checkcast` instruction the compiler inserted at every
   place you read a value out. When it fails, it fails at the *unloading dock*, not at
   the loading dock where the mistake was made.
3. **Some containers are special: they have the contents stencilled on the steel
   itself.** Those are **arrays**. `Order[]` really does know it holds orders, at
   runtime, forever. That single inconsistency — arrays reified, generics erased — is
   the root of half the weirdness in this topic.

---

## The bridge from what you know

### This is the closest analogue in the entire language

The master plan's translation table calls this out explicitly, and it is worth stating
plainly:

> **TypeScript's type erasure is the single closest honest match between the two
> languages.** Both erase generic type arguments before the code runs. Both check them
> only at compile time. Both leave you needing a runtime mechanism when you actually need
> the type at runtime.

You already know the consequences, because you have hit them:

```ts
function parse<T>(json: string): T {
  return JSON.parse(json);        // a lie. There is no check. T is a wish.
}

const order = parse<Order>('{"nonsense": true}');   // compiles, "works", is wrong

// which is why you reach for a runtime schema:
const order = OrderSchema.parse(JSON.parse(json));  // zod, io-ts, ajv
```

You reach for zod *because the type is gone at runtime*. Java's `Class<T>` tokens,
Jackson's `TypeReference` and Spring's `ParameterizedTypeReference` are the same move for
the same reason. Recognising that is most of the intuition transferred.

### Where Java differs — and be exact about this

| Aspect | TypeScript | Java |
|---|---|---|
| When types disappear | At compile time. `.js` has no trace whatsoever. | At compile time — but **erased signatures remain in the bytecode**. Methods really do exist as `save(Object)`. |
| Is any type info retained? | No. Nothing. (`emitDecoratorMetadata` emits a little, opt-in, and only for decorated members.) | **Yes** — a `Signature` attribute records the generic form, readable by reflection. Frameworks depend on it. |
| Runtime checks inserted | None. A wrong type just flows through and blows up later, or never. | **Yes** — `checkcast` is inserted at every point a generic value is read. The failure is loud: `ClassCastException`. |
| Method dispatch after erasure | No runtime dispatch at all; there are no types. | **Bridge methods** are synthesised so polymorphism keeps working after erasure. |
| Reified constructs | None. | **Arrays are reified.** `Order[]` knows its element type at runtime. |
| Why erasure was chosen | Types were bolted onto JavaScript; the runtime is not TypeScript's to change. | A deliberate **migration compatibility** decision for Java 5: existing bytecode and existing classes had to keep working, both source- and binary-compatible. |

The last row is the one worth understanding, because it is the *why* behind every rule
below.

### The parts with no TypeScript equivalent at all

**NO TYPESCRIPT ANALOGUE.**

Erasure itself is the closest honest match in the language — that is the whole point of
the table above. But three of its consequences have nothing on the TypeScript side to map
onto, and all three exist for the same reason: **TypeScript's output has no type system at
all, while Java's runtime keeps a real one.**

- **Bridge methods.** They exist to preserve *virtual dispatch* through an erased
  signature. JavaScript has no method descriptors and no link step, so there is no
  dispatch to preserve and nothing to synthesise.
- **Inserted `checkcast` instructions.** Java verifies at every read, so a wrong type
  throws loudly at a definite instruction. Compiled TypeScript performs no check
  anywhere; a wrong value simply flows onward and either corrupts something later or
  never surfaces at all.
- **The `Signature` attribute.** Java files the generic form away in the class file where
  reflection can read it back, which is what makes `TypeReference`, Spring's
  `ResolvableType` and Spring Data's entity discovery possible. TypeScript emits nothing
  equivalent, which is exactly why zod schemas are hand-written values rather than
  anything derived from your declared types.

### Why Java chose erasure — the decision, honestly

In 2004, `java.util.List` had been in production for six years. Millions of lines of code
used it raw. Java 5 had to add generics such that:

- Old source compiled against the new JDK. (Source compatibility.)
- Old *compiled* jars ran against the new JDK, and vice versa. (Binary compatibility.)
- `List<String>` could be passed to an old method taking a raw `List`.

Erasure achieves all three, because `List<String>` **is** `List` at runtime. Nothing to
migrate.

C# made the opposite choice two years later: reified generics, with the CLR itself
extended to carry type arguments. That gives `typeof(T)`, `new T()`, and
`List<int>` with no boxing — genuinely better, at the cost of a runtime change and a
break in the ecosystem. .NET could afford that; Java, with its installed base, decided it
could not.

Neither choice is wrong. But every "why can't I just…" in this topic traces back to that
one decision, and knowing that turns a list of arbitrary rules into a single consequence.

---

## What is this?

**Type erasure** is the compiler replacing every type parameter with its bound (or
`Object` if unbounded), then removing the type arguments, so that `List<String>` and
`List<Integer>` are the same class at runtime.

To keep the code correct after that, the compiler does two extra things: it **inserts
casts** wherever a generic value is read, and it **synthesises bridge methods** so
overriding still works across erased signatures.

A **reifiable type** is one whose full type information survives to runtime — the only
kinds of type you may use with `instanceof`, in a cast that is actually checked, as an
array element type, or in a `catch` clause.

---

## Why does it matter?

1. **`ClassCastException` at a line with no cast is the signature failure of this
   topic.** The classic is `objectMapper.readValue(json, List.class)`, which quietly
   gives you a `List<LinkedHashMap>`, and then explodes at the first `.orderId()` call
   three methods away. If you do not understand erasure, you will look for the bug at
   the crash site, and it is not there.

2. **Generic arrays are unsound, and every library that needs one has to work around
   it.** `ArrayList` internally holds an `Object[]`, not a `T[]`, for exactly this
   reason. If you write a generic container yourself and get this wrong, the exception
   surfaces in *your caller's* code, naming a cast they never wrote.

3. **Frameworks live or die on the `Signature` attribute.** Spring resolving
   `Repository<Order, Long>` at startup, Jackson's `TypeReference`, `@Autowired
   List<PaymentGateway>` — all of them recover type arguments that "were erased". Knowing
   that erased and *unavailable* are different words is the difference between using
   those APIs and being mystified by them.

---

## Syntax breakdown

The new constructs here are mostly things you *cannot* write, plus the workarounds.

### What erasure does, concretely

```java
// You write:
class Box<T extends Number> {
    private T value;
    T get()          { return value; }
    void set(T v)    { value = v; }
}

// The compiler produces, in effect:
class Box {
    private Number value;                   // T -> its leftmost bound
    Number get()        { return value; }
    void set(Number v)  { value = v; }
}
```

| Rule | Result |
|---|---|
| Unbounded `<T>` | erases to `Object` |
| `<T extends Number>` | erases to `Number` |
| `<T extends Payment & Auditable>` | erases to `Payment` — the **leftmost** bound |
| `List<String>` | erases to `List` |
| `List<String>[]` | erases to `List[]` |
| `T[]` where `<T>` unbounded | erases to `Object[]` |

The "leftmost bound" rule is why bound order matters (Topic 05): it decides the compiled
descriptor, which decides binary compatibility.

### The cast the compiler inserts

```java
List<Order> orders = repo.findAll();
Order first = orders.get(0);
```

`List.get` returns `Object` after erasure. So the compiler writes the cast for you:

```java
Order first = (Order) orders.get(0);       // checkcast, inserted by javac
```

You will see this instruction in the hands-on section. It is the mechanism behind every
"ClassCastException on a line with no cast".

### Things you cannot write, and why

```java
class Broken<T> {
    T[]  array   = new T[10];               // error: generic array creation
    T    made    = new T();                 // error: type parameter T cannot be instantiated
    Class<T> c   = T.class;                 // error: cannot select from a type variable
    static T shared;                        // error: non-static type variable T ...

    void check(Object o) {
        if (o instanceof List<String>) { }  // error: illegal generic type for instanceof
        if (o instanceof List<?>)      { }  // legal — List<?> IS reifiable
    }

    void f(List<String> a) { }
    void f(List<Integer> b) { }             // error: both methods have same erasure
}

class GenericException<T> extends Exception { }   // error: a generic class may not
                                                  // extend java.lang.Throwable
```

| What fails | Because |
|---|---|
| `new T[10]` | The JVM needs a real element type to stamp on the array header. `T` has none. |
| `new T()` | No constructor exists to call; `T` is `Object` by then. |
| `T.class` | A `Class` object is a runtime thing; `T` is not. |
| `static T shared` | One class object exists for all instantiations, so there is no `T` to pick. |
| `instanceof List<String>` | The check would be a lie — the JVM cannot see the argument. |
| Two `f` with same erasure | Both compile to `f(List)`. The class file cannot hold two. |
| Generic `Throwable` | `catch (GenericException<Order> e)` could not be checked at runtime, because catch matching is a runtime type test. |

### The workarounds

```java
// 1. Class token — carry the type as a value.
class TypedCache<T> {
    private final Class<T> type;
    TypedCache(Class<T> type) { this.type = type; }

    T coerce(Object raw) { return type.cast(raw); }              // a REAL, checked cast

    @SuppressWarnings("unchecked")
    T[] newArray(int n) { return (T[]) java.lang.reflect.Array.newInstance(type, n); }
}

// 2. Supplier — carry the constructor as a value. Usually better than a Class token.
static <T> List<T> fill(int n, java.util.function.Supplier<T> factory) {
    List<T> out = new ArrayList<>(n);
    for (int i = 0; i < n; i++) out.add(factory.get());
    return out;
}
// call: fill(3, Order::new)

// 3. Super-type token — recover a full parameterised type from an anonymous subclass.
List<Order> orders = objectMapper.readValue(json, new TypeReference<List<Order>>() {});
```

| Bit of syntax | What it means |
|---|---|
| `Class<T> type` field | The type argument, smuggled in as an ordinary object. `type.cast(x)` performs a genuinely checked cast and throws `ClassCastException` *here*, at the boundary, not three frames later. |
| `Array.newInstance(type, n)` | Creates an array whose runtime element type really is `T`. This is the only way to make a genuine `T[]`. |
| `new TypeReference<List<Order>>() {}` | Note the trailing `{}` — it creates an **anonymous subclass**. That subclass's generic superclass is `TypeReference<List<Order>>`, and *that* is recorded in the class file's `Signature` attribute, so `getGenericSuperclass()` can read it back. This is the trick, and it is worth knowing by name. |
| `@SuppressWarnings("unchecked")` | Required for `(T[])`. Every one of these needs a comment stating the invariant you are promising the compiler. |

### `@SafeVarargs` and heap pollution

```java
@SafeVarargs
static <T> List<T> listOf(T... items) {       // items is really an Object[] at runtime
    return List.of(items);
}
```

A generic varargs parameter is a generic array, which cannot exist — so the compiler
creates an `Object[]` and warns you about **heap pollution**: a variable whose static type
promises `T` but whose runtime contents may not be `T`. `@SafeVarargs` says "I have
checked that this method only *reads* from the array and never stores it anywhere typed".
Applying it when the method leaks the array is how you produce Trap 4.

---

## Example 1 — minimal

Three facts about erasure, in one runnable file.

```java
import java.util.*;

public class ErasureBasics {
    public static void main(String[] args) {

        List<String> names  = new ArrayList<>();
        List<Integer> counts = new ArrayList<>();

        // 1. Same class at runtime.
        System.out.println(names.getClass() == counts.getClass());     // expect: true
        System.out.println(names.getClass().getName());                // expect: java.util.ArrayList

        // 2. The cast is inserted at the READ, so a raw write blows up later.
        List raw = names;
        raw.add(42);                          // no complaint here
        try {
            String s = names.get(0);          // the inserted checkcast fires HERE
            System.out.println(s);
        } catch (ClassCastException e) {
            System.out.println("CCE at the read: " + e.getMessage());
        }

        // 3. Arrays ARE reified. This is the inconsistency.
        Object[] objects = new String[2];     // legal: arrays are covariant
        try {
            objects[0] = 42;                  // the ARRAY checks its own element type
        } catch (ArrayStoreException e) {
            System.out.println("ASE at the write: " + e.getMessage());
        }
    }
}
```

Run it. The important comparison is between points 2 and 3: **a generic collection fails
at the read, an array fails at the write.** The array knows what it holds; the list does
not. That is reification versus erasure, demonstrated in eight lines, and it is why
generic arrays are forbidden — Java could not make them behave like either one.

---

## Example 2 — production scenario

`orderflow` consumes an order-events feed from an upstream service. The payload is a JSON
array of order summaries.

### The version that ships green and pages someone at 03:00

```java
public class OrderFeedConsumer {

    private final ObjectMapper mapper = new ObjectMapper();

    public long totalMinorUnits(String json) throws Exception {
        List<OrderSummary> summaries = mapper.readValue(json, List.class);   // <- the defect

        long total = 0;
        for (OrderSummary s : summaries) {
            total += s.totalMinor();
        }
        return total;
    }
}

public record OrderSummary(String orderId, long totalMinor) {}
```

This compiles with a warning most builds suppress. It runs. And it throws:

```
java.lang.ClassCastException: class java.util.LinkedHashMap cannot be cast to
  class com.orderflow.orders.OrderSummary (java.util.LinkedHashMap is in module
  java.base of loader 'bootstrap'; com.orderflow.orders.OrderSummary is in unnamed
  module of loader 'app')
	at com.orderflow.orders.OrderFeedConsumer.totalMinorUnits(OrderFeedConsumer.java:11)
```

Line 11 is the **for loop**. There is no cast on line 11. The `readValue` on line 8,
which is where the mistake is, appears nowhere in the trace.

### Why, exactly

Three things happened in sequence:

1. `mapper.readValue(json, List.class)` was told to produce a `List`. Nothing more.
   Jackson has no idea about `OrderSummary`, so it does what it does for an unknown
   object shape: builds a `LinkedHashMap` per element.
2. Because `List.class` is `Class<List>`, the call's static type is a raw `List`, which
   the compiler happily assigns to `List<OrderSummary>` with an unchecked warning. **No
   check happens at this line.** The list genuinely contains maps.
3. The enhanced-for loop compiles to `iterator()` plus `next()`, which returns `Object`
   after erasure, so the compiler inserted `checkcast com/orderflow/orders/OrderSummary`.
   That instruction is on line 11. It is the first moment anything is verified.

**The mistake is at line 8; the exception is at line 11.** That distance is the entire
practical cost of erasure, and it is why "read the stack trace and look one step
upstream" is the habit this topic should install.

### The fix — a super-type token

```java
public class OrderFeedConsumer {

    private static final TypeReference<List<OrderSummary>> SUMMARY_LIST =
        new TypeReference<>() {};                       // note the {} — anonymous subclass

    private final ObjectMapper mapper;

    public OrderFeedConsumer(ObjectMapper mapper) { this.mapper = mapper; }

    public long totalMinorUnits(String json) throws JsonProcessingException {
        List<OrderSummary> summaries = mapper.readValue(json, SUMMARY_LIST);

        long total = 0;
        for (OrderSummary s : summaries) total += s.totalMinor();
        return total;
    }
}
```

Now Jackson knows the element type, constructs real `OrderSummary` records, and — this is
the part that matters — a malformed payload fails **inside `readValue`** with a
Jackson exception naming the offending field, rather than as a `ClassCastException` in
your business logic.

### How the token works, since "the type is erased" would suggest it cannot

`new TypeReference<List<OrderSummary>>() {}` creates an anonymous class. Anonymous classes
are real classes with real class files. The generic superclass of that class file is
recorded in its `Signature` attribute as `TypeReference<List<OrderSummary>>`.
`TypeReference`'s constructor calls `getClass().getGenericSuperclass()` and reads it back.

So: **the type argument was not erased from the class file. It was erased from the
runtime type system.** Those are different statements, and confusing them is the most
common misunderstanding about Java generics.

You will confirm this with `javap -v` in the hands-on section.

The same mechanism powers:

| API | What it recovers |
|---|---|
| Jackson `TypeReference<T>` | the full parameterised type for deserialization |
| Spring `ParameterizedTypeReference<T>` | the response type for `RestClient` / `RestTemplate` |
| Spring `ResolvableType` | injection-point generics, so `@Autowired List<PaymentGateway>` works |
| Guice / Guava `TypeToken<T>` | generic binding keys |
| Hibernate / Spring Data | `Repository<Order, Long>` — the entity type is read off the interface's generic superinterface at startup |

If you have ever wondered how Spring Data knows your repository is for `Order`, that is
the answer: it reads the `Signature` attribute of your interface.

### The other production shape — a generic array in your own code

You write a small ring buffer for the payment retry queue:

```java
public class RingBuffer<T> {
    private final T[] slots;                 // looks reasonable

    @SuppressWarnings("unchecked")
    public RingBuffer(int capacity) {
        this.slots = (T[]) new Object[capacity];    // the only thing that compiles
    }

    public T[] snapshot() { return slots.clone(); } // <- the defect
}
```

`slots` is really an `Object[]`. Inside the class nobody notices, because every read goes
through an inserted cast that happens to be to `Object`. But `snapshot()` hands that array
to a caller who was promised a `T[]`:

```java
RingBuffer<Payment> buffer = new RingBuffer<>(16);
Payment[] payments = buffer.snapshot();
```

```
java.lang.ClassCastException: class [Ljava.lang.Object; cannot be cast to
  class [Lcom.orderflow.payments.Payment;
	at com.orderflow.payments.RetryService.drain(RetryService.java:42)
```

Again: the exception is in the *caller*, at a line that contains no cast, naming array
types (`[L...` is the JVM's notation for "array of"). The defect is in your constructor.

**Fix — the way the JDK does it:** keep the field `Object[]`, cast on the way out
element-by-element, and never hand out the raw array typed as `T[]`.

```java
public class RingBuffer<T> {
    private final Object[] slots;            // honest about what it is
    private final int capacity;

    public RingBuffer(int capacity) {
        this.slots = new Object[capacity];
        this.capacity = capacity;
    }

    @SuppressWarnings("unchecked")           // safe: only Ts are ever written to slots
    public T get(int i) { return (T) slots[i]; }

    public List<T> snapshot() {              // return a List, not an array
        List<T> out = new ArrayList<>(capacity);
        for (Object o : slots) out.add((T) o);
        return List.copyOf(out);
    }
}
```

Open `java.util.ArrayList` in your IDE and look at the field declaration. It is
`transient Object[] elementData;`, with a comment. The JDK made exactly this decision for
exactly this reason. **When your generic container needs an array, hold `Object[]` and
never let a `T[]` escape.**

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `readValue(json, List.class)` / raw deserialization

**Wrong:**
```java
List<OrderSummary> summaries = mapper.readValue(json, List.class);
```

**Exact symptom:**
```
java.lang.ClassCastException: class java.util.LinkedHashMap cannot be cast to
  class com.orderflow.orders.OrderSummary
```
thrown at the **first use** of an element, not at the deserialization call. In a service
that deserializes on one thread and processes on another (a Kafka consumer, say), those
can be in different classes and different log contexts entirely.

**Root cause:** `List.class` is `Class<List>` — raw. Jackson builds `LinkedHashMap`s for
unknown element shapes. The unchecked assignment is not verified; the compiler's inserted
`checkcast` at the read is the first real check.

**Fix:** `new TypeReference<List<OrderSummary>>() {}`, or Jackson's
`mapper.getTypeFactory().constructCollectionType(List.class, OrderSummary.class)`. In
Spring, `ParameterizedTypeReference`. Then turn `-Xlint:unchecked -Werror` on so the next
one fails the build.

---

### Trap 2 — `new T[]` and the `(T[]) new Object[]` "fix"

**Wrong:**
```java
private final T[] slots = (T[]) new Object[capacity];
public T[] snapshot() { return slots.clone(); }
```

**Exact symptom:** in the caller, not in your class:
```
java.lang.ClassCastException: class [Ljava.lang.Object; cannot be cast to
  class [Lcom.orderflow.payments.Payment;
```
Note the `[L...;` notation — seeing that in a CCE means "array of", and it is a strong
signal that you are looking at exactly this bug.

**Root cause:** arrays are **reified**; generics are **erased**. An array created as
`new Object[n]` has `Object` stamped in its header for life. The `(T[])` cast erases to
`(Object[])`, which is a no-op, so nothing fails until someone assigns it to a variable of
the real array type — and *that* cast is checked.

**Fix:** hold `Object[]` internally and never publish a `T[]`; return a `List<T>` instead.
If you genuinely need a real `T[]` (usually only when interoperating with an API that
demands one), take a `Class<T>` and use `Array.newInstance(type, n)`.

---

### Trap 3 — overloads with the same erasure

**Wrong:**
```java
public void publish(List<OrderPlaced> events)     { ... }
public void publish(List<PaymentCaptured> events) { ... }
```

**Exact symptom:** a compile error, which is the good news:
```
error: name clash: publish(List<PaymentCaptured>) and publish(List<OrderPlaced>)
  have the same erasure
```

**Root cause:** both erase to `publish(List)`. A class file cannot hold two methods with
the same name and descriptor. This is not a language nicety — it is a physical limit of
the class-file format.

**Fix:** distinct names (`publishOrderEvents` / `publishPaymentEvents`), or a single
generic method `<E extends DomainEvent> void publish(List<E> events)`, or — usually best —
a sealed event hierarchy plus one method (Topic 28).

> The confusing near-miss: `void f(List<String>)` and `void f(Set<String>)` compile fine,
> because `List` and `Set` are different erasures. Only the *type arguments* vanish, not
> the container.

---

### Trap 4 — heap pollution through generic varargs

**Wrong:**
```java
@SafeVarargs
static <T> T[] leak(T... items) {
    return items;                       // hands the Object[] out, typed as T[]
}

String[] names = leak("a", "b");        // looks fine
```

**Exact symptom:** either
```
java.lang.ClassCastException: class [Ljava.lang.Object; cannot be cast to
  class [Ljava.lang.String;
```
at the assignment, or — in the sneakier version where the array is stored rather than
returned —
```
java.lang.ArrayStoreException: java.lang.Integer
```
at a write that looks perfectly typed.

**Root cause:** a generic varargs parameter cannot be a real `T[]`, so the compiler
creates an `Object[]`. `@SafeVarargs` suppresses the warning; it does not make anything
safe. It is a promise you made, and here you broke it.

**Fix:** apply `@SafeVarargs` only when the method **reads** from the varargs array and
never stores or returns it. If you need to return the items, return a `List<T>`:
```java
@SafeVarargs
static <T> List<T> listOf(T... items) { return List.of(items); }   // copies; safe
```
`@SafeVarargs` is only legal on `static`, `final`, or `private` methods, precisely because
an overridable method's promise cannot be enforced on subclasses.

---

### Trap 5 — the bridge method's hidden cast (the one nobody sees coming)

**Wrong:**
```java
public class Order implements Comparable<Order> {
    private final long totalMinor;
    @Override public int compareTo(Order other) {
        return Long.compare(totalMinor, other.totalMinor);
    }
}

// somewhere else, via a raw or wildcard-typed collection
TreeSet set = new TreeSet();          // raw
set.add(new Order(199));
set.add("SKU-4471");                  // compiles: raw set
```

**Exact symptom:**
```
java.lang.ClassCastException: class java.lang.String cannot be cast to
  class com.orderflow.orders.Order
	at com.orderflow.orders.Order.compareTo(Order.java)
	at java.base/java.util.TreeMap.put(TreeMap.java:...)
```
The top frame names `Order.compareTo` — **a method you wrote, on a line you did not
write a cast on**. Developers stare at `compareTo`, which is obviously correct, and lose
an hour.

**Root cause:** `Comparable<T>` erases to `compareTo(Object)`. Your method is
`compareTo(Order)`, which does not override the erased signature, so `javac` synthesised a
**bridge method**:

```java
// generated by the compiler, flagged ACC_BRIDGE ACC_SYNTHETIC
public int compareTo(Object o) {
    return compareTo((Order) o);       // <- the cast that throws
}
```

`TreeMap` calls the erased `compareTo(Object)`. The bridge casts. The cast fails. The
stack frame is attributed to your class because the bridge *is* in your class file, with
no line number of its own.

**Fix:** the immediate one is to never use raw collections — the raw `TreeSet` is what let
a `String` in. The lasting one is to recognise the shape: **a `ClassCastException` whose
top frame is one of your own methods that contains no cast is almost always a bridge
method.** You will see this bridge in `javap` in the hands-on section, and after that it
will never confuse you again.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM. Where I describe what to look for
in `javap` output, I am telling you which instructions and flags to search for — not
quoting output.

### Setup

```bash
mkdir -p ~/java-lab/06 && cd ~/java-lab/06
java --version
```

### Proof 1 — same class, different type arguments

`SameClass.java`:
```java
import java.util.*;

public class SameClass {
    public static void main(String[] args) {
        List<String>  a = new ArrayList<>();
        List<Integer> b = new ArrayList<>();
        System.out.println(a.getClass() == b.getClass());
        System.out.println(a.getClass().getName());

        Object[] arr = new String[1];
        System.out.println(arr.getClass().getName());   // arrays: contrast
    }
}
```

```bash
java SameClass.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `true`, then `java.util.ArrayList` | Expected. One class object serves every type argument. This is erasure, observed directly. |
| The array line prints `[Ljava.lang.String;` | The contrast that matters: the array **kept** its element type. `[L...;` is the JVM's array notation, and recognising it speeds up every array-related CCE you will ever read. |
| `false` on the first line | Impossible on any conforming JVM. Check you did not construct different implementations. |

### Proof 2 — the inserted `checkcast`

`InsertedCast.java`:
```java
import java.util.*;

public class InsertedCast {
    static long total(List<Long> amounts) {
        long sum = 0;
        for (Long amount : amounts) {
            sum += amount;
        }
        return sum;
    }
}
```

```bash
javac InsertedCast.java
javap -c InsertedCast.class
```

**What to look for** in the disassembly of `total`:

- `invokeinterface java/util/Iterator.next` returning `Object`.
- immediately after it, a **`checkcast`** instruction naming `java/lang/Long`.
- then `invokevirtual java/lang/Long.longValue` (the unboxing from Topic 01).

| What you see | What it means |
|---|---|
| A `checkcast` you never wrote | The core mechanism. `Iterator.next()` returns `Object` after erasure, so the compiler inserts the cast. Every "CCE at a line with no cast" is this instruction. |
| No `checkcast` at all | You may have used a primitive-specialised path, or the loop was over an array rather than a `List`. Check the source. |
| `checkcast` naming `java/lang/Number` | Your parameter was declared with a bound like `<T extends Number>`. The cast is to the **erasure**, which is the bound — good, that is the next proof. |

### Proof 3 — the erasure is the bound, and the generic form is still filed away

`BoundErasure.java`:
```java
import java.util.*;

public class BoundErasure {
    static <T extends Number> double sum(List<T> values) {
        double total = 0;
        for (T v : values) total += v.doubleValue();
        return total;
    }

    static <T> T firstOf(List<T> values) { return values.get(0); }
}
```

```bash
javac BoundErasure.java
javap -s BoundErasure.class          # -s prints the erased descriptors
javap -v BoundErasure.class | grep -A1 -i 'Signature'
```

**What to look for:**

| What you see | What it means |
|---|---|
| `sum`'s `descriptor:` mentions `java/util/List` and returns `D` (double) — with no `T` anywhere | The erased signature. This is what the JVM links against. |
| `firstOf`'s descriptor returns `Ljava/lang/Object;` | Unbounded `T` erased to `Object`. |
| A `Signature:` attribute per method carrying the generic form, e.g. text containing `<T:Ljava/lang/Number;>` | **The photocopy in the ship's office.** The generic information *is* in the class file. It is not used by the JVM for verification, but reflection can read it — which is why `TypeReference` and `ParameterizedTypeReference` work at all. |
| No `Signature` attribute | You compiled with an unusual flag, or `grep` missed it because of formatting. Try `javap -v BoundErasure.class \| grep -i -B2 -A2 signature`. |

**How to read it:** this single output kills the most common misconception. "Erased" does
not mean "deleted from the class file". It means "not used by the runtime type system".

### Proof 4 — bridge methods, with `javap -c` (the required proof for this topic)

`BridgeDemo.java`:
```java
public class BridgeDemo {

    interface Repository<T> {
        T save(T entity);
    }

    static class Order { }

    static class OrderRepository implements Repository<Order> {
        @Override public Order save(Order entity) { return entity; }
    }
}
```

```bash
javac BridgeDemo.java
javap -c -p 'BridgeDemo$OrderRepository.class'
```

**What to look for — this is the whole point of the exercise:**

1. **Two `save` methods**, not one. One takes `BridgeDemo$Order` and returns
   `BridgeDemo$Order`. The other takes `java/lang/Object` and returns
   `java/lang/Object`.
2. In the body of the `Object`-taking one: a **`checkcast`** naming `BridgeDemo$Order`,
   followed by an `invokevirtual` to the `Order`-taking `save`, then `areturn`.
3. That method is short — typically load `this`, load the argument, `checkcast`,
   `invokevirtual`, `areturn`.

| What you see | What it means |
|---|---|
| Two `save` methods, the `Object` one delegating via a `checkcast` | The bridge method, exactly as described. `Repository.save` erased to `save(Object)`, your `save(Order)` does not override that descriptor, so `javac` generated one that does. |
| Only one `save` method | You forgot `-p` (bridges are synthetic and may be hidden), or the interface is not generic. Re-run with `-p`. |
| A third `save` | Check for covariant return types or a second interface — each distinct erased signature that needs bridging gets its own. |

Now confirm the flags — `javap -c` shows the code, `javap -v` shows the access flags:

```bash
javap -v -p 'BridgeDemo$OrderRepository.class' | grep -B3 -A3 -i bridge
```

**What to look for:** a `flags:` line on the `Object`-taking `save` containing
**`ACC_BRIDGE`** and **`ACC_SYNTHETIC`**.

| What you see | What it means |
|---|---|
| `ACC_BRIDGE, ACC_SYNTHETIC` on the `Object` variant | Definitive. `ACC_SYNTHETIC` means "the compiler made this, not the programmer"; `ACC_BRIDGE` means specifically "this exists to preserve polymorphism after erasure". |
| No flags line matching | Your `grep` may have missed it due to line wrapping. Drop the `grep` and read the method's attribute block directly. |
| `ACC_SYNTHETIC` without `ACC_BRIDGE` | You are looking at a different generated member — an accessor for a private nested-class field, for instance. Find the `save(Object)` entry specifically. |

**How to read the whole thing:** you wrote one method. The class file has two. The second
one exists because `TreeMap`, `Collections.sort` and every other pre-generics-shaped
caller invokes the *erased* signature. The bridge's `checkcast` is the exact instruction
that throws in Trap 5 — and now you have seen it, so the stack trace will never mystify
you.

> Version note: bridge generation is specified behaviour and stable across Java 8–25.
> What can differ between JDK versions is the exact `javap` formatting and whether
> synthetic members are shown without `-p`. If your output differs from this description,
> trust your output and re-run with `-p -v`.

### Proof 5 — `instanceof` and what is reifiable

`Reifiable.java`:
```java
import java.util.*;

public class Reifiable {
    static void check(Object o) {
        System.out.println(o instanceof List<?>);        // legal
        // System.out.println(o instanceof List<String>);   // uncomment: expect an error
        System.out.println(o instanceof ArrayList);      // legal (raw, but reifiable)
        System.out.println(o instanceof String[]);       // legal — arrays are reified
    }
    public static void main(String[] args) { check(new ArrayList<String>()); }
}
```

```bash
javac Reifiable.java
```

**What to look for:** uncomment the middle line.

| What you see | What it means |
|---|---|
| `error: illegal generic type for instanceof` | Expected. `List<String>` is not reifiable — the JVM cannot perform that test, so the language forbids asking. |
| `List<?>` compiles | Expected. An unbounded wildcard type **is** reifiable: the test is just "is it a List", which the JVM can do. |
| `String[]` compiles | The inconsistency, once more. Arrays carry their element type. |

### Proof 6 — reproduce the bridge-method `ClassCastException`

```bash
cat > BridgeCCE.java <<'EOF'
import java.util.*;

public class BridgeCCE {
    static class Order implements Comparable<Order> {
        final long totalMinor;
        Order(long t) { this.totalMinor = t; }
        @Override public int compareTo(Order o) { return Long.compare(totalMinor, o.totalMinor); }
    }

    @SuppressWarnings({"unchecked","rawtypes"})
    public static void main(String[] args) {
        TreeSet set = new TreeSet();          // raw on purpose
        set.add(new Order(199));
        set.add("SKU-4471");                  // expect the explosion here
    }
}
EOF
java BridgeCCE.java
```

**What to look for:** the exception and, more importantly, **the top stack frame**.

| What you see | What it means |
|---|---|
| `ClassCastException: class java.lang.String cannot be cast to class BridgeCCE$Order`, top frame `BridgeCCE$Order.compareTo` | The target result. The frame names a method of yours that contains no cast — because the frame is the *bridge*, which shares the method name. This is the signature you should now recognise instantly. |
| The frame names `TreeMap.compare` instead | Your JDK inlined or ordered things differently, or the comparison went through a `Comparator`. Still the same mechanism; note what yours did. |
| No exception | `TreeSet` compared the `String` against nothing because it was the first element. Add a second `Order` after the `String`, or reorder the adds. |

---

## Practice exercises

### 1 — Easy: find the bridge

1. Write a generic interface `Validator<T> { boolean isValid(T value); }` and a class
   `OrderValidator implements Validator<Order>`.
2. Compile it and run `javap -c -p` on the implementation. Count the `isValid` methods.
   Write down the descriptor of each.
3. Run `javap -v -p` and find the `ACC_BRIDGE` flag. Paste the flags line.
4. Now make `OrderValidator` implement `Validator<Object>` instead (adjusting the method
   signature). Recompile and re-run `javap`. **Is there still a bridge?** Explain why or
   why not in one sentence — this is the question that proves you understand what bridges
   are for.
5. Finally, add a covariant return: a class `Base { Object get() }` and
   `Sub extends Base { String get() }`. Is there a bridge here too? What does that tell
   you about whether bridges are a *generics* feature specifically?

### 2 — Medium: the deserialization incident (combines Topics 01, 02, 05)

You are handed this bug report: *"`/orders/summary` returns HTTP 500 intermittently.
Stack trace attached. Started after the upstream team's release."*

```java
public class SummaryController {

    private final ObjectMapper mapper = new ObjectMapper();
    private final Map<String, Integer> cachedTotals = new HashMap<>();

    public Map<String, Object> summary(String upstreamJson, String customerId, String orderId)
            throws Exception {

        List<OrderSummary> rows = mapper.readValue(upstreamJson, List.class);

        long total = 0;
        for (OrderSummary row : rows) {
            total += row.totalMinor();
        }

        int cached = cachedTotals.get(customerId);
        if (cached == total) {
            return Map.of("status", "unchanged");
        }

        return Map.of("customer", customerId, "order", orderId, "total", total / 100.0);
    }
}
```

1. There are **five** defects: two from this topic, one from Topic 01, one from Topic 02,
   one from Topic 05. Find them all.
2. For each, give the **exact exception message shape or business symptom**, and say
   which line the failure surfaces on versus which line contains the mistake. The gap
   between those two is the point of the exercise.
3. Explain, in terms of erasure, why the compiler did not stop any of the type-related
   ones.
4. Rewrite the method. It must compile clean under
   `javac -Xlint:unchecked,rawtypes -Werror`.
5. Write the one-line build configuration change that would have caught the worst defect
   at compile time, and say honestly what it would cost to turn on across an existing
   codebase.

### 3 — Hard: production simulation — a type-safe event registry

Build the dispatcher from Topic 05's Exercise 3, and now break it properly.

**Part A.** Implement:
```java
public final class HandlerRegistry {
    private final Map<Class<? extends DomainEvent>, EventHandler<?>> handlers = new HashMap<>();

    public <E extends DomainEvent> void register(Class<E> type, EventHandler<E> handler) { ... }

    public void dispatch(DomainEvent event) { ... }   // needs an unchecked cast
}
```
Capture the exact compiler error you get in `dispatch` before adding the suppression.
Then write the suppression **with a comment stating the invariant** that makes it safe.

**Part B.** Break the invariant. Add a `registerRaw(Class<?>, EventHandler)` method (raw
on purpose), register a `PaymentCapturedHandler` under `OrderPlaced.class`, and dispatch
an `OrderPlaced`.
- Capture the exception and its full stack trace.
- Identify which frame it lands in. Is it `dispatch`? Is it the handler? Is it a bridge?
- Run `javap -c -p` on your handler class and find the instruction that actually threw.

**Part C.** Now make it impossible. Remove `registerRaw`. Prove the compiler now rejects
the mismatched registration, and paste the error.

**Part D.** Try the array version. Write
`public <E extends DomainEvent> E[] drainAll(Class<E> type)` two ways:
1. `(E[]) new Object[n]` — then call it and assign to `OrderPlaced[]`.
2. `Array.newInstance(type, n)` — then do the same.

Capture the exception from version 1 (it should name `[Ljava.lang.Object;`) and confirm
version 2 works. Then answer: **why does version 2 work when erasure supposedly removed
`E`?** Your answer must mention where the type actually came from.

**Part E.** The framework question. Add a method that returns `List<E>` and have a caller
recover the element type at runtime using
`new TypeReference<List<OrderPlaced>>() {}.getType()` — or, without Jackson, by writing
your own three-line super-type token using `getGenericSuperclass()`.
- Print the recovered type.
- Then run `javap -v` on your anonymous subclass's class file (it will be named
  something like `YourClass$1.class`) and find the `Signature` attribute that made it
  possible.
- Write two sentences explaining, to a TypeScript developer, why this trick has no
  TypeScript equivalent.

**Part F.** Argue the other side. Given sealed interfaces and pattern matching (Topic 28
and 29), is the generic registry the right design at all? Give the alternative, and state
the concrete condition under which each wins.

---

## Interview questions

### Q1 — "What is type erasure and why does Java have it?"

**Mid-level answer:** "Generic type information is removed at compile time, so
`List<String>` and `List<Integer>` are both just `List` at runtime. It's for backward
compatibility."

**Senior answer:** "Type arguments are replaced by their bound — `Object` if unbounded —
and the type arguments are dropped from the runtime type system. The reason is
specifically *migration* compatibility for Java 5: `java.util.List` had been in
production for six years, and generics had to be both source- and binary-compatible with
existing compiled code. Erasure achieves that because `List<String>` genuinely *is*
`List` at runtime, so old jars and new jars interoperate with no migration. C# made the
opposite call two years later — reified generics, at the cost of changing the CLR. Two
details people usually get wrong: erasure does not mean the information is deleted from
the class file. The `Signature` attribute retains the generic form, which is how
reflection, Jackson's `TypeReference` and Spring's `ResolvableType` recover it. And to
keep polymorphism working, the compiler inserts casts at every read and synthesises
bridge methods at the override seam."

**What separates them:** naming migration compatibility specifically rather than "backward
compatibility" generally, the C# contrast, and — the big one — knowing that erased is not
the same as absent.

**Follow-up:** "So how does Spring Data know your repository is for `Order`?" They want
`getGenericInterfaces()` / the `Signature` attribute.

---

### Q2 — "What is a bridge method?"

**Mid-level answer:** "Something the compiler generates for generics. I've seen it in
stack traces."

**Senior answer:** "It preserves polymorphism after erasure. If `Repository<T>` declares
`T save(T)`, that erases to `Object save(Object)`. Your `OrderRepository.save(Order)` has
a different descriptor, so it does not override the erased method — which would break
dispatch through the interface. So `javac` synthesises `Object save(Object)` in your
class, flagged `ACC_BRIDGE` and `ACC_SYNTHETIC`, whose body casts the argument to `Order`
and delegates to your real method. You can see it with `javap -c -p`. The practical
consequence is a stack trace that confuses people: if a raw collection lets a wrong type
in, the `ClassCastException`'s top frame is *your* method — `Order.compareTo`, say —
which contains no cast at all. The cast is in the bridge, and the bridge shares the method
name. Bridges also appear for covariant return types, so they are not purely a generics
mechanism."

**What separates them:** the exact flags, the delegating cast, the stack-trace shape as a
practical diagnostic, and knowing bridges predate/exceed generics via covariant returns.

**Follow-up:** "Show me the `javap` command." `javap -c -p` for the code, `javap -v -p`
for the `ACC_BRIDGE` flag. If they cannot produce the command, they have read about
bridges rather than looked at one.

---

### Q3 — "Why can't you write `new T[10]`?"

**Mid-level answer:** "Because `T` is erased, so the JVM doesn't know what type of array
to create."

**Senior answer:** "Because arrays are **reified** and generics are **erased**, and the
two cannot be reconciled. An array carries its element type in its header for life and
checks every store against it — that is why `Object[] a = new String[1]; a[0] = 42;`
throws `ArrayStoreException` at the *write*. A generic collection has no such check; its
cast is at the *read*. If `new T[10]` were allowed, `T` would be `Object` by then, so the
array would silently be an `Object[]` with no store check — you would have an array that
claims to be `T[]`, accepts anything, and fails somewhere else entirely. Java forbids the
creation rather than ship that. The workarounds are: hold `Object[]` internally and never
publish it as `T[]` — that is exactly what `ArrayList.elementData` does — or pass a
`Class<T>` and use `Array.newInstance`, which produces a genuinely reified `T[]` because
the token carried the type in as a value."

**What separates them:** contrasting write-time versus read-time checking, citing
`ArrayList`'s actual field, and knowing both workarounds and when each applies.

**Follow-up:** "So is array covariance a good design?" The honest answer is no — it exists
because Java 1.0 had no generics and needed `Arrays.sort(Object[])` to work; the price is
`ArrayStoreException`. That is Topic 07.

---

### Q4 — "This throws `ClassCastException` on a line with no cast. Explain."

```java
List<OrderSummary> rows = mapper.readValue(json, List.class);
for (OrderSummary row : rows) { total += row.totalMinor(); }   // <- CCE here
```

**Mid-level answer:** "Jackson didn't deserialize into the right type — you need a
`TypeReference`."

**Senior answer:** "Right fix, and here is the mechanism. `List.class` is `Class<List>`,
so `readValue` returns a raw `List` and Jackson, having no element type, builds
`LinkedHashMap` per element. The assignment to `List<OrderSummary>` is unchecked — nothing
is verified there, which is why the trace does not mention that line. The enhanced-for
compiles to `iterator().next()`, which returns `Object` after erasure, so `javac` inserted
a `checkcast` to `OrderSummary`, and *that* is the instruction on the failing line. The
mistake is on line one; the explosion is on line two. The fix is
`new TypeReference<List<OrderSummary>>() {}`, which also moves the failure inside
`readValue` where Jackson can name the offending field. And I'd turn on
`-Xlint:unchecked -Werror` so the next one fails the build instead of production."

**What separates them:** locating the inserted `checkcast`, explaining why the assignment
line is absent from the trace, and adding the build-level prevention rather than just the
code fix.

**Follow-up:** "How would you find this in a codebase before it fires?" Good answers:
`-Xlint:unchecked`, a grep for `readValue(.*\.class)`, ErrorProne, or a static-analysis
rule.

---

### Q5 — "Java erases generics. So does TypeScript. Are they the same?"

**Mid-level answer:** "Pretty much — neither has the types at runtime."

**Senior answer:** "Closest match in the language, and worth being exact about the
differences. Both drop type arguments before execution, and both therefore need a runtime
mechanism when you actually need the type — zod in TypeScript, a `Class<T>` token or
`TypeReference` in Java, for the same reason. Four real differences. One: Java keeps the
*erased* signature in the bytecode, so a method genuinely exists as `save(Object)`;
TypeScript's output has no signatures at all. Two: Java inserts `checkcast` at every read,
so a wrong type throws loudly at a definite point — in TypeScript a wrong type just
flows through and corrupts something later, or never surfaces. Three: Java synthesises
bridge methods so virtual dispatch survives erasure; TypeScript has no runtime dispatch
to preserve. Four: Java files a `Signature` attribute in the class file, so reflection can
read the generic form back — which is how Spring resolves `Repository<Order, Long>` and
how `TypeReference` works. TypeScript has literally nothing equivalent; that is why zod
schemas are values you write by hand rather than something derived from the type. And
Java has one reified construct TypeScript has none of: arrays."

**What separates them:** four precise differences rather than a vibe, and the zod
comparison, which shows they understand *why* both ecosystems arrived at runtime tokens.

**Follow-up:** "If you could reify Java's generics tomorrow, would you?" They want you to
weigh it: yes for `new T[]`, `instanceof List<String>`, and unboxed `List<int>`; no for
the migration cost and the existing bytecode. Project Valhalla is the real-world version
of this conversation.

---

## Mental model checkpoint

1. Erasure keeps the `Signature` attribute in the class file but the JVM ignores it for
   type checks. Why not have the JVM check it? What would break — think about jars
   compiled before Java 5.

2. Arrays are reified and check on write; generic collections are erased and check on
   read. For each, name a bug that the *other* design would have caught earlier. Then say
   which design you would choose for a new language.

3. `<T extends Payment & Auditable>` erases to `Payment` — the leftmost bound. What
   happens to binary compatibility if you reorder those two bounds in a published
   library? Reason it through the descriptor.

4. A bridge method's cast can throw. Could the compiler have made bridges throw a
   *better* exception — one naming the registration site rather than the call site?
   Sketch what that would require.

5. `TypeReference<List<Order>>() {}` needs the trailing `{}`. Explain to a colleague why
   removing those two characters breaks it, in one sentence, without using the word
   "erasure".

6. You have argued that erasure was the right call for Java 5. Is it still the right call
   in 2026, given that almost no production code predates Java 8? What would migrating to
   reified generics actually cost today?

7. TypeScript engineers write zod schemas by hand and keep them in sync with their types.
   Java engineers write `TypeReference` tokens. Both are duplication forced by erasure.
   Which ecosystem's version is worse, and why?

---

## Quick reference card

### Erasure rules

| Declared | Erases to |
|---|---|
| `<T>` | `Object` |
| `<T extends Number>` | `Number` |
| `<T extends A & B>` | `A` (leftmost) |
| `List<String>` | `List` |
| `T[]` (unbounded `T`) | `Object[]` |
| `Map<K, V>` | `Map` |

### Reifiable types — the complete list

Usable with `instanceof`, as an array element type, and in a `catch` clause:

- primitives (`int`, `long`, …)
- non-generic classes and interfaces (`String`, `Order`)
- raw types (`List`, `Map`)
- types with **only unbounded wildcards** (`List<?>`, `Map<?, ?>`)
- arrays whose element type is reifiable (`String[]`, `List<?>[]`)

Not reifiable: `List<String>`, `T`, `List<? extends Number>`, `T[]`.

### What you cannot write

```java
new T[10]                    // generic array creation
new T()                      // cannot instantiate a type variable
T.class                      // cannot select from a type variable
static T field;              // non-static type variable in static context
o instanceof List<String>    // illegal generic type for instanceof
void f(List<A>); void f(List<B>);        // same erasure
class E<T> extends Exception             // generic Throwable forbidden
catch (MyException<Order> e)             // cannot catch a parameterised type
```

### Workarounds

| Need | Use |
|---|---|
| Create a `T` | `Supplier<T>` (preferred) or `Class<T>` + `getDeclaredConstructor().newInstance()` |
| Create a `T[]` | `Array.newInstance(type, n)` with a `Class<T>` token |
| Store an array in a generic class | Hold `Object[]`; never publish it as `T[]`; return `List<T>` |
| A checked runtime cast | `Class<T>.cast(x)` — fails at the boundary, not downstream |
| Deserialize a `List<Order>` | `new TypeReference<List<Order>>() {}` / `ParameterizedTypeReference` |
| Read generic info at runtime | `getGenericSuperclass()`, `getGenericInterfaces()`, Spring's `ResolvableType` |

### Diagnostic table — reading the exception

| Symptom | Almost certainly |
|---|---|
| CCE at a read, mentioning `LinkedHashMap` | raw `readValue(..., List.class)` |
| CCE mentioning `[Ljava.lang.Object;` | a `(T[]) new Object[]` that escaped |
| CCE whose top frame is your own cast-free method | a **bridge method** |
| `ArrayStoreException` | array covariance — a real reified check firing |
| `name clash: ... have the same erasure` | two overloads differing only in type arguments |
| `inference variable T has incompatible bounds` | Topic 05 — you need `? super` |

### Gotchas checklist

- [ ] `List<String>.class` does not exist. `List.class` is `Class<List>`.
- [ ] Erased ≠ absent. The `Signature` attribute keeps the generic form.
- [ ] The compiler's `checkcast` is at the **read**; the mistake was at the **write**.
- [ ] Arrays check on write (`ArrayStoreException`); generics check on read (CCE).
- [ ] Never let a `(T[]) new Object[]` escape your class.
- [ ] `@SafeVarargs` is a promise, not a guarantee. Only on `static`/`final`/`private`.
- [ ] Two overloads differing only in type arguments will not compile.
- [ ] Every `@SuppressWarnings("unchecked")` needs a comment naming the invariant.
- [ ] `-Xlint:unchecked,rawtypes -Werror` on new code. Ratchet on old code.
- [ ] `javap -c -p` for the bridge body; `javap -v -p` for `ACC_BRIDGE`.

---

## When would I use this at work?

**1. Diagnosing a `ClassCastException` in a service you did not write.**
The exception is at the read; the bug is at the write. Knowing that, and knowing the
three signature shapes (`LinkedHashMap`, `[Ljava.lang.Object;`, a cast-free frame of your
own), turns a multi-hour hunt into a targeted search. This is the highest-value thing in
the topic.

**2. Writing or reviewing any deserialization boundary.**
Kafka consumers, REST clients, cache reads, config binding — every one of them has to
carry a type in as a value. `TypeReference`, `ParameterizedTypeReference`, and Jackson's
`constructCollectionType` are the tools, and reaching for them reflexively prevents the
single most common erasure bug in production Java.

**3. Building a generic abstraction that other teams will use.**
A registry, a cache, a container, an event dispatcher — the moment you need an array or a
`new T()`, you are in this topic. Getting it right means the failure happens at *your*
boundary with a message naming the mistake, rather than in a caller's code with a message
naming nothing useful.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and wrappers**: why `List<int>` is impossible — erasure needs
  everything to be an `Object`, which is why boxing exists in collections at all.
- **02 — Nominal typing**: `Class<T>` tokens and `implements` are both name-based; this
  is the same machinery seen from the runtime side.
- **05 — Generics I**: the bound you place decides the erasure. That connection is
  load-bearing here.

**This unlocks:**
- **07 — Variance, PECS, wildcards**: why generics are invariant (erasure gives no runtime
  check) while arrays are covariant (they do have one) — and why array covariance is
  regarded as a design mistake.
- **12/13 — HashMap and equals/hashCode**: `Map<K,V>` erases to `Map<Object,Object>`, so
  every correctness guarantee comes from `equals`/`hashCode`, not from the type system.
- **19 — Serialization**: `TypeReference`, schema evolution, and why cross-service
  boundaries need explicit schemas rather than Java types.
- **21–24 — Lambdas and streams**: `Collector<T,A,R>` and `Function<T,R>` all erase; the
  `Signature` attribute is what lets your IDE still show you the types.
- **47 — Spring Data JPA**: the entity type is read off your repository interface's
  generic superinterface at startup. Now you know from where.
- **76 — Bytecode**: reading class files properly, including everything you just
  `javap`-ed.
- **83 — Native image**: reflection over `Signature` attributes is invisible to
  closed-world analysis, which is why `TypeReference`-heavy code needs reflection
  configuration.

---

*Java baseline 21. Erasure has been unchanged since Java 5, and bridge-method generation
is specified behaviour stable across 8–25. The only thing that varies between JDK versions
is `javap`'s output formatting and whether synthetic members appear without `-p` — if your
output differs from the descriptions above, trust your output. Project Valhalla proposes
specialised generics over value types, which would give a limited form of reification for
those cases; it has not landed as of Java 25, and nothing in this topic should be planned
around it.*
