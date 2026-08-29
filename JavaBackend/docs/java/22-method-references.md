# 22 — Method References: All Four Forms

## Phase: 2 — Modern Java
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

You are filling in the same one-blank-line form from Topic 21. But this time, the
instruction you want to write is *already written down somewhere else*.

So instead of copying it out, you write a **pointer**: "do the thing on page 14".

That pointer is a **method reference**. `::` is the word "do".

There are only four kinds of pointer, and the difference between them is entirely
about **who the instruction gets done to**:

1. **"Do `Long.parseLong`."** The instruction needs nothing but its input. Static.
2. **"Do `println` — to *that* printer over there."** You point at a specific object
   *now*, and the pointer remembers it. Bound.
3. **"Do `length` — to whoever shows up."** You name only the *kind* of thing it
   happens to. The victim arrives later, as the first argument. Unbound.
4. **"Make a new one."** The instruction is "build one of these". Constructor.

Form 3 is the one that confuses everybody, and it is worth staring at:

```java
Function<String, Integer> f = String::length;
f.apply("SKU-4471");     // 7
```

You named a method that takes **zero** arguments. You got back a function that takes
**one**. The missing argument is the object the method runs on. It slid into the
front of the parameter list.

That slide is the entire content of this topic.

---

## The bridge from what you know

### What transfers

You already write this in TypeScript:

```ts
const ids = orders.map(o => o.id);
const nums = strings.map(Number);          // passing a function by name
setTimeout(this.retry.bind(this), 1000);   // binding a receiver
```

Java's `::` covers the middle and the third line, plus a case JavaScript does not
really have a name for.

| You know | Java | Verdict |
|---|---|---|
| Passing a named function: `map(Number)` | Static reference: `map(Long::parseLong)` | **HONEST ANALOGUE** |
| `obj.method.bind(obj)` | Bound reference: `obj::method` | **HONEST ANALOGUE** — same semantics, better syntax |
| `Array.prototype.map.call` / uncurried this | Unbound reference: `String::length` | **PARTIAL** — the shape exists in JS but nobody writes it, so the intuition is not there |
| `new Product(...)` as a value — you just write `(x) => new Product(x)` | Constructor reference: `Product::new` | **PARTIAL** — JS classes *are* values, Java classes are not |
| `obj.method` passed bare (loses `this`) | Does not compile in Java | **NO ANALOGUE** — Java's unbound form is explicit; JS's is an accident |

### The break that matters most

In JavaScript, `obj.method` passed as a value is a famous bug: it loses `this`.

```ts
const logger = { prefix: "orders", log(m: string) { console.log(this.prefix, m); } };
setTimeout(logger.log, 0);   // TypeError: cannot read prefix of undefined
```

Java makes the two cases **syntactically different**, so the bug cannot happen:

```java
logger::log        // BOUND. `logger` is captured now. `this` inside log is `logger`.
Logger::log        // UNBOUND. No receiver yet. It becomes the first parameter.
```

Lowercase-thing-before-`::` means an object; uppercase-thing means a type. That single
reading habit resolves most confusion about which form you are looking at.

### The other break — `expr::method` evaluates `expr` immediately

```java
Runnable r = order.payment()::retry;   // order.payment() runs RIGHT NOW
// ... much later ...
r.run();                               // retry() runs on the payment captured earlier
```

If `order.payment()` returns `null`, you get a `NullPointerException` **at the line
that creates the reference**, not at `r.run()`. In JavaScript, `() => a.b.retry()`
would defer everything. This is a real difference and it produces a genuinely
confusing stack trace. Trap 2 covers it.

---

## What is this?

A **method reference** is a compact way to write a lambda whose body does nothing
except call one existing method (or constructor) and pass its arguments straight
through.

```java
order -> order.totalPence()      →      Order::totalPence
s     -> Long.parseLong(s)       →      Long::parseLong
sku   -> new Product(sku)        →      Product::new
m     -> logger.info(m)          →      logger::info
```

The rules are the same as for a lambda: it is not a function value, it needs a
**target type**, and the compiler checks the method's signature against the target
interface's single abstract method.

**Critically, a method reference is not "faster" or "different" from the equivalent
lambda.** It compiles to the same `invokedynamic` + `LambdaMetafactory` mechanism. The
only compile-time difference is that a method reference to an existing method usually
does **not** need a synthetic `lambda$...$0` method — the bootstrap can point directly
at the target method. You will see that in `javap` below. It is a curiosity, not a
performance argument.

### The four forms, precisely

| # | Form | Syntax | Receiver | Example |
|---|---|---|---|---|
| 1 | Static | `Type::staticMethod` | none | `Long::parseLong` |
| 2 | Bound instance | `expr::instanceMethod` | the value of `expr`, captured now | `System.out::println` |
| 3 | Unbound instance | `Type::instanceMethod` | supplied as the **first argument** | `String::length` |
| 4 | Constructor | `Type::new` | none; produces a new instance | `ArrayList::new` |

There is a fifth thing you will meet that is really form 2 in disguise:

| 4b | Array constructor | `Type[]::new` | none | `String[]::new` — an `IntFunction<String[]>` |
| 2b | Superclass call | `super::method` | the enclosing `this`, dispatched to the super implementation | `super::toString` |

### Form 3 in full detail — the one that trips people

```java
// The method:  public int length()          — zero parameters, called ON a String
// The reference:
Function<String, Integer> f = String::length;
// The interface method:  Integer apply(String)   — ONE parameter
```

The compiler's rule is: for `Type::instanceMethod`, the target interface's first
parameter becomes the receiver, and the remaining parameters map to the method's
parameters.

An arity table makes it concrete:

| Method signature | Reference | Equivalent lambda | Target interface |
|---|---|---|---|
| `int String.length()` | `String::length` | `s -> s.length()` | `ToIntFunction<String>` |
| `boolean String.isEmpty()` | `String::isEmpty` | `s -> s.isEmpty()` | `Predicate<String>` |
| `boolean String.startsWith(String)` | `String::startsWith` | `(a, b) -> a.startsWith(b)` | `BiPredicate<String,String>` |
| `int String.compareTo(String)` | `String::compareTo` | `(a, b) -> a.compareTo(b)` | `Comparator<String>` |
| `String String.concat(String)` | `String::concat` | `(a, b) -> a.concat(b)` | `BinaryOperator<String>` |

Read the last three rows carefully. **A one-parameter instance method becomes a
two-parameter function.** That is why `String::compareTo` is a valid `Comparator` —
`compare(a, b)` becomes `a.compareTo(b)`.

This is also exactly why the master plan calls this "the one that confuses people". It
is the only place in Java where an argument appears from nowhere.

---

## Why does it matter?

**1. Readability, which is not a small thing.** `orders.stream().map(Order::id)` says
"the ids". `orders.stream().map(o -> o.id())` says "for each o, take o's id". The
first is a noun; the second is a sentence. Across a codebase that difference compounds.

**2. Constructor references are how APIs get their containers.** You will meet
`Collectors.toCollection(TreeSet::new)`, `toMap(..., ..., ..., LinkedHashMap::new)`,
`Stream.toArray(Product[]::new)`, `orElseGet(Order::new)`. Every one of those needs a
`Supplier` or `IntFunction` that makes an object. Without `::new` you write
`() -> new TreeSet<>()` everywhere, which is fine but noisier — and in Topic 24 the
collector's supplier is *always* written this way.

**3. Ambiguity errors are unreadable if you do not know the forms.** When `javac`
tells you "reference to X is ambiguous" or "non-static method cannot be referenced
from a static context", it is telling you it cannot decide between form 1 and form 3.
Knowing there are two candidate forms turns a baffling error into a one-line fix.

**4. Bound references capture eagerly.** That is a real behavioural difference from the
equivalent lambda, and it produces NPEs at surprising lines. It is also the mechanism
behind a retention leak identical to Topic 21's.

---

## Syntax breakdown

### Form 1 — static method reference

```java
Type::staticMethod
```

```java
Function<String, Long>  parse   = Long::parseLong;      // s -> Long.parseLong(s)
BinaryOperator<Long>    sum     = Long::sum;            // (a,b) -> Long.sum(a,b)
Predicate<String>       isBlank = String::isBlank;      // WRONG — see note
IntBinaryOperator       max     = Math::max;            // (a,b) -> Math.max(a,b)
Supplier<Instant>       now     = Instant::now;         // () -> Instant.now()
```

> The `String::isBlank` line above is deliberately mislabelled: `isBlank()` is an
> **instance** method, so that is form 3, not form 1. The syntax is identical. This is
> the single most important thing to internalise: **forms 1 and 3 look the same.** The
> compiler distinguishes them by looking at whether the named method is static, and by
> the arity of the target interface.

### Form 2 — bound instance reference

```java
expression::instanceMethod
```

```java
Logger log = LoggerFactory.getLogger(OrderService.class);
Consumer<String> info = log::info;              // m -> log.info(m)

Consumer<String> out = System.out::println;     // System.out evaluated NOW

Order order = repository.load(id);
Supplier<Long> total = order::totalPence;       // () -> order.totalPence()

// `this` and `super`
Runnable r      = this::reconcile;
Supplier<String> s = super::toString;
```

**The receiver expression is evaluated once, at the point the reference is created.**
Not per invocation. Write that on the inside of your eyelids.

### Form 3 — unbound instance reference

```java
Type::instanceMethod
```

```java
Function<Order, Long>          total  = Order::totalPence;   // o -> o.totalPence()
ToIntFunction<String>          len    = String::length;      // s -> s.length()
Comparator<String>             cmp    = String::compareTo;   // (a,b) -> a.compareTo(b)
BiPredicate<String, String>    starts = String::startsWith;  // (a,b) -> a.startsWith(b)
```

The receiver type must be the target interface's **first parameter type**, or a
supertype of it.

### Form 4 — constructor reference

```java
Type::new
Type[]::new
```

```java
Supplier<List<Order>>          newList = ArrayList::new;      // () -> new ArrayList<>()
Function<String, Product>      newProd = Product::new;        // sku -> new Product(sku)
BiFunction<String, Long, Price> newPrice = Price::new;        // (c,a) -> new Price(c,a)
IntFunction<Product[]>         newArr  = Product[]::new;      // n -> new Product[n]
Supplier<Map<Sku, Integer>>    newMap  = HashMap::new;
Supplier<Set<Sku>>             newSet  = TreeSet::new;
```

**Overloaded constructors are resolved by the target type.** `ArrayList::new` is a
`Supplier<List<T>>` when the target has no parameters, and an `IntFunction<List<T>>`
(the initial-capacity constructor) when the target takes an `int`. The same three
characters mean two different constructors depending on where you put them. That is
target typing doing its job, and it is occasionally startling.

**Generic constructors:** `ArrayList::new` infers its type argument from the target.
You do not write `ArrayList<Order>::new` unless inference fails — and it is legal
syntax if you need it.

### What cannot be a method reference

You need a method reference to be a *pure forwarding call*. Anything else must be a
lambda:

```java
o -> o.totalPence() + 100        // arithmetic on the result — no reference possible
o -> service.charge(o, "GBP")    // a fixed extra argument — no reference possible
o -> { log(o); return o.id(); }  // two statements — no reference possible
(a, b) -> b.compareTo(a)         // arguments swapped — no reference possible
```

The last one matters: **method references cannot reorder arguments.** If the parameter
order does not line up exactly, you write the lambda.

### `[JAVA 25]` note

Nothing in method-reference syntax changed between Java 21 and 25. Java 21 already
supports all four forms plus array constructors. There is no `[JAVA 25]` feature to
flag here — a rare clean topic.

`[LEGACY — still asked]` Before Java 8 the equivalent was an anonymous class per
callback, and utility classes full of static helpers. Interviewers who learned Java 6
sometimes phrase this as "so it's like a function pointer?" — the honest answer is
"it's an instance of a functional interface produced by an `invokedynamic` call site",
which is Topic 21.

---

## Example 1 — minimal

All four forms in one file, each doing the same visible job.

```java
import java.util.*;
import java.util.function.*;

public class FourForms {

    record Product(String sku, long pricePence) {
        long pricePence() { return pricePence; }
    }

    public static void main(String[] args) {

        // FORM 1 — static
        Function<String, Long> parse = Long::parseLong;
        System.out.println(parse.apply("4995"));                 // 4995

        // FORM 2 — bound: receiver captured now
        Product widget = new Product("SKU-1", 4995);
        Supplier<Long> priceOfWidget = widget::pricePence;
        System.out.println(priceOfWidget.get());                 // 4995

        // FORM 3 — unbound: receiver becomes the first argument
        Function<Product, Long> priceOfAny = Product::pricePence;
        System.out.println(priceOfAny.apply(widget));            // 4995

        // FORM 4 — constructor
        BiFunction<String, Long, Product> make = Product::new;
        System.out.println(make.apply("SKU-2", 1999));

        // FORM 4b — array constructor
        IntFunction<Product[]> makeArray = Product[]::new;
        System.out.println(makeArray.apply(3).length);           // 3
    }
}
```

Compare forms 2 and 3 side by side. Same method, same class. Form 2 has **zero**
parameters because the receiver is already decided. Form 3 has **one** parameter
because the receiver is not.

---

## Example 2 — production scenario

`orderflow` has a reporting endpoint: given a batch of raw order rows from a CSV
import, build `Order` objects, index them, sort them, and produce a summary.

### The version written entirely with lambdas

```java
public class OrderImportService {

    public ImportResult importBatch(List<String> csvLines) {

        List<Order> orders = csvLines.stream()
                .map(line -> line.trim())
                .filter(line -> !line.isEmpty())
                .map(line -> parseOrder(line))
                .filter(order -> order != null)
                .sorted((a, b) -> a.placedAt().compareTo(b.placedAt()))
                .collect(Collectors.toCollection(() -> new ArrayList<>()));

        Map<CustomerId, List<Order>> byCustomer = new HashMap<>();
        orders.forEach(order -> {
            byCustomer.computeIfAbsent(order.customerId(), id -> new ArrayList<>())
                      .add(order);
        });

        orders.forEach(order -> auditLog.record(order));

        Order[] asArray = orders.toArray(n -> new Order[n]);

        return new ImportResult(orders, byCustomer, asArray);
    }
}
```

Every one of those lambdas is pure forwarding. Every one has a method reference.

### The same code with method references

```java
public class OrderImportService {

    private final AuditLog auditLog;
    private final OrderParser parser;

    public OrderImportService(AuditLog auditLog, OrderParser parser) {
        this.auditLog = auditLog;
        this.parser = parser;
    }

    public ImportResult importBatch(List<String> csvLines) {

        List<Order> orders = csvLines.stream()
                .map(String::trim)                                   // FORM 3 unbound
                .filter(Predicate.not(String::isEmpty))              // FORM 3 unbound
                .map(parser::parse)                                  // FORM 2 bound
                .filter(Objects::nonNull)                            // FORM 1 static
                .sorted(Comparator.comparing(Order::placedAt))       // FORM 3 unbound
                .collect(Collectors.toCollection(ArrayList::new));   // FORM 4 constructor

        Map<CustomerId, List<Order>> byCustomer = new HashMap<>();
        for (Order order : orders) {
            byCustomer.computeIfAbsent(order.customerId(), id -> new ArrayList<>())
                      .add(order);
        }

        orders.forEach(auditLog::record);                            // FORM 2 bound

        Order[] asArray = orders.toArray(Order[]::new);              // FORM 4b array

        return new ImportResult(orders, byCustomer, asArray);
    }
}
```

Walk through the interesting ones:

| Line | Form | Why it works |
|---|---|---|
| `String::trim` | 3 | `trim()` takes nothing; the `String` being mapped becomes the receiver. |
| `Predicate.not(String::isEmpty)` | 3 | `isEmpty()` returns `boolean` with no args → `Predicate<String>`. `Predicate.not` is cleaner than `s -> !s.isEmpty()`. |
| `parser::parse` | 2 | `parser` is a field. **This captures `this`** — see Trap 3. |
| `Objects::nonNull` | 1 | `Objects.nonNull(Object)` is static and takes one arg → `Predicate<Order>`. |
| `Comparator.comparing(Order::placedAt)` | 3 | The key extractor is a `Function<Order, Instant>`. This is where unbound references earn their keep. |
| `Collectors.toCollection(ArrayList::new)` | 4 | The collector needs a `Supplier<C>`. `ArrayList::new` is exactly that. Topic 24. |
| `orders.toArray(Order[]::new)` | 4b | `toArray` needs an `IntFunction<Order[]>` — "make me an array of size n". |

Note the one lambda that **survived**: `id -> new ArrayList<>()` inside
`computeIfAbsent`. You might reach for `ArrayList::new` — and it does not compile.
`computeIfAbsent` wants a `Function<K, V>`, so the supplier receives the key. A
`Supplier`-shaped `ArrayList::new` has the wrong arity. You would need
`k -> new ArrayList<>()`. **The same three characters that worked one line earlier
fail here, because the target type is different.** That is the lesson of this example.

> Also note: the `forEach` on the map building was rewritten as a plain `for` loop.
> Mutating an external `HashMap` from inside a lambda is Topic 21's Trap 1 in a
> different costume, and Topic 24 will replace it entirely with
> `Collectors.groupingBy(Order::customerId)`.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the static/unbound ambiguity

**Wrong:**
```java
Function<Integer, String> f = Integer::toString;
```

**Exact symptom:**
```
error: incompatible types: invalid method reference
        reference to toString is ambiguous
          both method toString(int) in Integer and method toString() in Integer match
```

**Root cause:** `Integer` has **both** `static String toString(int)` (form 1, one
argument) **and** `String toString()` (form 3, receiver-as-first-argument). Against a
target of `Function<Integer, String>` — one input, one output — both are a legal
reading. The compiler will not guess.

**Fix — say which one:**
```java
Function<Integer, String> f = i -> Integer.toString(i);   // explicitly the static one
Function<Integer, String> g = Object::toString;           // explicitly the instance one
```

**The same family of error, different message:**
```java
Function<Order, Long> f = Order::totalPence;   // fine
Supplier<Long>       g = Order::totalPence;    // error
```
gives
```
error: invalid method reference
        non-static method totalPence() cannot be referenced from a static context
```

**Root cause of that one:** you wrote form 3 syntax, but the target interface
(`Supplier`) takes **no** parameters, so there is nowhere for the receiver to come
from. The compiler falls back to reading it as form 1 — a static call — and there is
no such static method.

**How to diagnose in one second:** count the target interface's parameters, count the
method's parameters. If `target params == method params`, it is form 1 or 2. If
`target params == method params + 1`, it is form 3. If neither, that is your bug.

---

### Trap 2 — a bound reference evaluates its receiver immediately

**Wrong:**
```java
public Runnable retryLater(Order order) {
    return order.payment()::retry;    // payment() may return null
}
```

**Exact symptom:**
```
java.lang.NullPointerException: Cannot invoke "com.orderflow.Payment.retry()"
  because the return value of "com.orderflow.Order.payment()" is null
	at com.orderflow.RetryScheduler.retryLater(RetryScheduler.java:41)
```

The stack trace points at line 41 — the line that **created** the `Runnable` — not at
the scheduler thread that eventually ran it. Engineers reading that trace look for the
call to `run()` and cannot find it, because it never happened.

**Root cause:** in `expr::method`, the JLS says `expr` is evaluated **when the method
reference expression is evaluated**, and the result is captured. A null receiver
throws right there.

Contrast with the lambda, which defers everything:
```java
return () -> order.payment().retry();   // NPE happens at run(), on the scheduler thread
```

Neither behaviour is "correct" — they are different, and you must know which you wrote.

**Fix:** decide explicitly.
```java
// If you want to fail fast at scheduling time (usually right):
Payment payment = Objects.requireNonNull(order.payment(),
        () -> "order " + order.id() + " has no payment to retry");
return payment::retry;

// If the payment genuinely may appear later:
return () -> {
    Payment p = order.payment();
    if (p == null) { log.warn("no payment for order {}", order.id()); return; }
    p.retry();
};
```

**The general rule:** `a.b()::c` runs `a.b()` now. If `a.b()` is expensive, has side
effects, or can be null, that is a decision you are making, so make it on purpose.

---

### Trap 3 — a bound reference to a field method captures `this`

**Wrong:**
```java
public class OrderPageRenderer {

    private final byte[] templateCache = new byte[16_000_000];   // 16 MB
    private final AuditLog auditLog;

    public void subscribe(EventBus bus) {
        bus.onOrderPlaced(auditLog::record);   // looks harmless
    }
}
```

**Exact symptom:** memory grows steadily under load and never comes down. A heap dump
(Topic 79) opened in MAT shows `OrderPageRenderer` instances retained, with an
incoming reference path through the event bus's listener list, via a
`OrderPageRenderer$$Lambda$...` object holding an `OrderPageRenderer` field.

**Root cause:** `auditLog` is an instance field. Reading it requires `this`. So the
generated lambda class captures `this`, not `auditLog` — the whole renderer, including
its 16 MB template cache, is now reachable from the event bus forever.

This is exactly Topic 21's Trap 2, but it hides better here because `auditLog::record`
*looks* like it captured only the audit log.

**Fix:** copy the collaborator into a local first, so only it is captured.
```java
public void subscribe(EventBus bus) {
    AuditLog log = this.auditLog;      // local — capture stops here
    bus.onOrderPlaced(log::record);
}
```

**How to check without a heap dump:** in `javap -p` on the enclosing class, look at
whether the synthetic method for that site is `static`. Static means nothing from the
instance was captured. Non-static means `this` came along. Details in Hands-on Proof 3.

---

### Trap 4 — reaching for `::new` where the target needs an argument

**Wrong:**
```java
Map<CustomerId, List<Order>> byCustomer = new HashMap<>();
byCustomer.computeIfAbsent(customerId, ArrayList::new).add(order);
```

**Exact symptom:** it compiles on some type combinations and produces nonsense, and on
others it fails with:
```
error: incompatible types: invalid method reference
        no suitable constructor found for ArrayList(CustomerId)
```

**Root cause:** `computeIfAbsent(K key, Function<? super K, ? extends V> f)` passes the
**key** to the function. So `ArrayList::new` is being read as
`new ArrayList<>(customerId)` — which does not exist, or worse, if the key were an
`Integer`, would silently resolve to `new ArrayList<>(int initialCapacity)` and
allocate a list of that capacity. That second case compiles and is a real bug.

**Fix:**
```java
byCustomer.computeIfAbsent(customerId, k -> new ArrayList<>()).add(order);
```

**The rule:** before writing `Type::new`, ask what arity the target interface has.
`Supplier` → zero args. `Function` → one arg, which becomes a constructor argument.
`IntFunction` → one `int` arg, which for `ArrayList` hits the capacity constructor and
for `Type[]::new` is the array length.

The `Integer`-key version of this bug is worth remembering as a code-review pattern:
**a constructor reference plus an integer-typed input is a latent silent bug.**

---

### Trap 5 — assuming a method reference is "faster" than a lambda

**Wrong reasoning:** "I converted all the lambdas to method references for
performance."

**Exact symptom:** none — the code is fine. The symptom is a PR review comment nobody
can defend, and a team habit of preferring an unreadable method reference over a clear
lambda.

**Root cause:** both compile to `invokedynamic` linked by `LambdaMetafactory`. The only
mechanical difference: a method reference to an existing method can point the bootstrap
handle directly at that method, so `javac` often does not emit a synthetic
`lambda$...$0`. That saves one method in the class file and, at most, one level of
indirection that the JIT would inline away anyway.

Capture behaviour is identical: `parser::parse` capturing `this` allocates exactly like
`x -> parser.parse(x)` does. `Long::parseLong` capturing nothing is a cached singleton
exactly like `s -> Long.parseLong(s)` is.

**Fix — use the right criterion, which is readability:**

| Use a method reference when | Use a lambda when |
|---|---|
| The body is exactly one forwarding call | Anything else happens |
| The name of the method reads as a noun in context (`Order::id`) | The parameter name carries meaning (`line -> parseCsv(line, schema)`) |
| It is a `Comparator` key extractor | Arguments need reordering or fixing |
| It supplies a container (`ArrayList::new`) | The target arity does not match |

`Order::id` is clearer than `o -> o.id()`. `Integer::parseInt` in a chain of six
mapping steps where every step is a bare `::` is *less* clear than named lambda
parameters. Judge per site.

---

## Hands-on proof

Commands you run. No fabricated output — what follows is what to type and how to read
whatever you get.

### Setup

```bash
mkdir -p ~/java-lab/22 && cd ~/java-lab/22
java --version
```

### Proof 1 — the unbound form really does shift the receiver

`ReceiverShift.java`:
```java
import java.util.function.*;

public class ReceiverShift {
    public static void main(String[] args) {

        // one parameter in, receiver supplied at call time
        Function<String, Integer> unbound = String::length;
        System.out.println("unbound  : " + unbound.apply("SKU-4471"));

        // zero parameters in, receiver fixed at creation time
        String sku = "SKU-4471";
        Supplier<Integer> bound = sku::length;
        System.out.println("bound    : " + bound.get());

        // two parameters in, first becomes the receiver
        BiPredicate<String, String> starts = String::startsWith;
        System.out.println("biPred   : " + starts.test("SKU-4471", "SKU-"));

        // an instance method with one parameter used as a two-argument Comparator
        java.util.Comparator<String> cmp = String::compareTo;
        System.out.println("comparator: " + cmp.compare("a", "b"));
    }
}
```

```bash
java ReceiverShift.java
```

**What to look for:** all four lines print, and the arities differ even though only two
distinct methods are involved.

| What you see | What it means |
|---|---|
| All four print a value | Confirmed: `length()` served both a one-arg and a zero-arg interface, and `startsWith(String)` served a two-arg interface. The receiver slot is real. |
| A compile error on the `Comparator` line | Check you imported `java.util.Comparator` or used the fully-qualified name as above. |

Now deliberately break it. Change `Function<String, Integer> unbound = String::length;`
to `Supplier<Integer> unbound = String::length;` and recompile.

**What to look for:** `non-static method length() cannot be referenced from a static
context`. **How to read it:** the compiler had no first parameter to put the receiver
in, so it tried form 1 (static) and failed. That error message *always* means "I read
your `Type::method` as a static call because there was no room for a receiver".

### Proof 2 — a method reference does not always create a `lambda$` method

`SyntheticCount.java`:
```java
import java.util.function.*;

public class SyntheticCount {

    static Function<String, Integer> viaLambda() {
        return s -> s.length();
    }

    static Function<String, Integer> viaMethodRef() {
        return String::length;
    }

    static Function<String, Integer> viaLambdaWithWork() {
        return s -> s.length() + 1;
    }
}
```

```bash
javac SyntheticCount.java
javap -p SyntheticCount.class
```

**What to look for:** which `lambda$...` methods appear.

| What you see | What it means |
|---|---|
| `lambda$viaLambda$0` and `lambda$viaLambdaWithWork$2` (or similar), but **nothing** for `viaMethodRef` | Expected. The method reference's bootstrap handle points straight at `String.length`, so `javac` had no body to move. |
| A `lambda$viaMethodRef$N` also appears | Possible if the compiler needed an adapter — e.g. for a boxing conversion. Note that `Integer` here does require boxing, so an adapter may legitimately appear. If it does, that is the honest answer; record what you see. |
| Nothing at all | You forgot `-p`. |

**How to read it:** this is a curiosity, not a performance argument. The point is to see
that a method reference and a lambda are the *same mechanism* with a slightly different
amount of glue.

Confirm the mechanism is shared:
```bash
javap -c -p -v SyntheticCount.class | sed -n '/BootstrapMethods/,$p'
```
**What to look for:** `LambdaMetafactory.metafactory` for **all three** sites. If both
lambdas and method references bootstrap through the same factory, the "method
references are faster" claim has nowhere to live.

### Proof 3 — does this reference capture `this`?

`CaptureCheck.java`:
```java
import java.util.function.*;

public class CaptureCheck {

    private final String tenant = "acme";

    Supplier<String> capturesThis() {
        return this::tenantName;          // instance method on `this`
    }

    static Supplier<String> capturesNothing() {
        return CaptureCheck::staticName;  // static
    }

    Supplier<String> capturesLocalOnly(String name) {
        return name::toUpperCase;         // bound to a local, not to `this`
    }

    private String tenantName()      { return tenant; }
    private static String staticName() { return "static"; }
}
```

```bash
javac CaptureCheck.java
javap -c -p CaptureCheck.class
```

**What to look for:** the bytecode of each method, just before the `invokedynamic`.

| What you see before `invokedynamic` | What it means |
|---|---|
| `aload_0` (pushes `this`) | **`this` is captured.** The generated object holds a reference to the whole `CaptureCheck`. This is Trap 3's leak shape. |
| `aload_1` (pushes the parameter) | Only the parameter is captured. Safe. |
| Nothing pushed — `invokedynamic` is the first instruction | Non-capturing. The call site returns a cached singleton. |

**How to read it:** `aload_0` immediately before an `invokedynamic` is the single most
useful bytecode pattern in this phase. It is the mechanical answer to "does this lambda
retain my bean?"

Cross-check at runtime by printing the generated class names:
```java
System.out.println(capturesNothing() == capturesNothing());   // expect true
```
A `true` here means non-capturing and cached, consistent with the bytecode.

### Proof 4 — constructor references resolve by target type

`CtorTarget.java`:
```java
import java.util.*;
import java.util.function.*;

public class CtorTarget {
    public static void main(String[] args) {

        Supplier<List<String>>   noArgs   = ArrayList::new;   // new ArrayList<>()
        IntFunction<List<String>> withCap = ArrayList::new;   // new ArrayList<>(int)

        System.out.println(noArgs.get().size());        // 0
        System.out.println(withCap.apply(100).size());  // 0  — capacity, not size

        Function<Collection<String>, List<String>> copy = ArrayList::new;
        System.out.println(copy.apply(List.of("a", "b")).size());   // 2
    }
}
```

```bash
java CtorTarget.java
```

**What to look for:** three different constructors selected by three different target
types, from identical source text.

| What you see | What it means |
|---|---|
| `0`, `0`, `2` | Correct. The third call picked the copy constructor. Note the second printed `0` and not `100` — capacity is not size, which is the silent-bug shape from Trap 4. |
| A compile error on any line | Read which constructor it says is missing; that names the target type it inferred. |

Now break it deliberately: add
```java
Function<Integer, List<String>> oops = ArrayList::new;
System.out.println(oops.apply(5).size());
```
**What to look for:** it compiles (autoboxing lets `Integer` reach the `int` capacity
constructor via unboxing) and prints `0`, not `5`. **How to read it:** this is exactly
the `computeIfAbsent` bug from Trap 4 — a constructor reference that silently found a
different constructor than you meant.

### Proof 5 — the eager receiver

`EagerReceiver.java`:
```java
import java.util.function.*;

public class EagerReceiver {

    static String value = "first";

    static String current() {
        System.out.println("current() called");
        return value;
    }

    public static void main(String[] args) {
        System.out.println("--- creating method reference ---");
        Supplier<Integer> viaRef = current()::length;

        System.out.println("--- creating lambda ---");
        Supplier<Integer> viaLambda = () -> current().length();

        value = "second-and-longer";

        System.out.println("--- invoking ---");
        System.out.println("ref    : " + viaRef.get());
        System.out.println("lambda : " + viaLambda.get());
    }
}
```

```bash
java EagerReceiver.java
```

**What to look for:** *where* the two `current() called` lines appear, and the two
lengths.

| What you see | What it means |
|---|---|
| One `current() called` between "creating method reference" and "creating lambda"; the other between "invoking" and the first result | The method reference evaluated its receiver eagerly; the lambda deferred. This is Trap 2, isolated. |
| `ref` prints the length of `"first"` and `lambda` prints the length of `"second-and-longer"` | Same conclusion from the values instead of the ordering. |
| Both `current() called` lines appear before "invoking" | Re-read your source; you probably wrote `current()::length` twice. |

---

## Practice exercises

### 1 — Easy: classify and convert

**Part A.** For each of the following, write down which of the four forms it is, and the
exact equivalent lambda:

```java
Integer::parseInt
System.out::println
Order::totalPence
HashMap::new
String[]::new
Objects::requireNonNull
this::reconcile
String::compareToIgnoreCase
Instant::now
List::size
```

**Part B.** For each, write down a target interface it would satisfy, with full generic
arguments.

**Part C.** Two of them are ambiguous or invalid against at least one plausible target.
Find them, write the code that produces the error, and record the exact message.

### 2 — Medium: rewrite the import service (combines Topics 01, 10, 12, 13, 14, 21)

Take the lambda-heavy `OrderImportService` from Example 2 and do the following:

**Part A.** Convert every convertible lambda to a method reference. Leave the ones that
cannot convert, and write a one-line comment on each explaining *why* it cannot.

**Part B.** The sort is `Comparator.comparing(Order::placedAt)`. Extend it to sort by
`placedAt` descending, then by `totalPence` descending, then by `id` ascending as a
tiebreak. Use only `Comparator` static and default methods plus method references —
no lambda bodies. Then answer: why does `.reversed()` at the end of the chain not do
what a newcomer expects? (Topic 14.)

**Part C.** `byCustomer` is a `HashMap<CustomerId, List<Order>>`. Make `CustomerId` a
class with a broken `equals`/`hashCode` (Topic 13) and observe what happens to the
grouping. Then fix it. State the exact observable symptom, not the principle.

**Part D.** Replace `orders.toArray(Order[]::new)` with `orders.toArray(new Order[0])`
and explain which one you would use and why. Both are correct; there is a real
argument each way.

### 3 — Hard: production simulation on `orderflow`

Build a small `EventBus` and prove the retention leak, then fix it.

**Part A.** Write:
```java
interface OrderListener { void onOrder(Order order); }

class EventBus {
    private final List<OrderListener> listeners = new ArrayList<>();
    void subscribe(OrderListener l) { listeners.add(l); }
    void publish(Order o) { listeners.forEach(l -> l.onOrder(o)); }
}
```

**Part B.** Write `OrderPageRenderer` holding a `private final byte[] cache = new
byte[16_000_000];` and an `AuditLog auditLog` field. Give it a `subscribe(EventBus)`
that registers `auditLog::record`.

**Part C.** In a loop, create 200 renderers, subscribe each, and drop the reference.
Force a GC and take a heap dump:
```bash
java -Xmx1g -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=leak.hprof LeakDemo
# or, while it runs:
jcmd <pid> GC.heap_dump /tmp/leak.hprof
jcmd <pid> GC.class_histogram | head -30
```

Report: how many `OrderPageRenderer` instances survive, and how much heap they hold.
Use the class histogram first — it is cheaper than opening MAT — then open the dump if
you want the retention path.

**Part D.** Apply Trap 3's fix (local copy). Re-run. Report the new histogram counts.

**Part E.** Now the honest part. `auditLog::record` reads *cleaner* than the local-copy
version. Argue for keeping the leaky version with a different fix — a `WeakReference`
listener list, or explicit `unsubscribe` in a lifecycle callback. Then say which you
would actually ship in a Spring service and why. (Topic 37 has the lifecycle hooks.)

**Part F.** Before running any of this, use `javap -c -p` on your renderer and predict
from the `aload_0` presence which version leaks. Then check whether your prediction
matched the heap dump. If it did not, work out which one lied and why.

---

## Interview questions

### Q1 — "What are the four forms of method reference?"

**Mid-level answer:** "Static method, instance method, constructor, and… one on an
object. `Class::method`, `obj::method`, `Class::new`."

**Senior answer:** "Static — `Long::parseLong`. Bound instance — `logger::info`, where
the receiver is a specific object captured at the point the reference is created.
Unbound instance — `String::length`, where the receiver is not fixed and instead
becomes the first parameter of the target interface. And constructor — `ArrayList::new`,
plus the array form `Order[]::new` which is an `IntFunction`. The unbound one is the
one worth explaining: `String::length` names a zero-argument method but satisfies
`Function<String, Integer>`, because the compiler slides the receiver into the first
parameter slot. That's also why `String::compareTo` is a valid `Comparator<String>` —
a one-argument instance method becomes a two-argument function."

**What separates them:** naming the receiver behaviour for each form rather than
listing syntax, and giving the `Comparator` example, which shows they have actually
reasoned about arity rather than memorised four bullet points.

**Interviewer's follow-up:** "`Integer::toString` — which form is it?" The answer is
"it's ambiguous, and that's a compile error, because `Integer` has both a static
`toString(int)` and an instance `toString()`."

---

### Q2 — "Is `order.payment()::retry` the same as `() -> order.payment().retry()`?"

**Mid-level answer:** "Yes, method references are shorthand for lambdas."

**Senior answer:** "No — the receiver evaluation differs. In `expr::method`, `expr` is
evaluated when the method reference is *created*, and the result is captured. The
lambda defers everything to invocation time. Three consequences. If `payment()` returns
null, the method reference throws an NPE at the line that creates the `Runnable`, and
your stack trace points at the scheduling code rather than the executing thread —
genuinely confusing to debug. If `payment()` is expensive, the reference pays once and
the lambda pays per call. And if the payment object can be swapped later, the reference
holds the old one and the lambda picks up the new one. I'd choose deliberately: the
eager form is usually what I want, because failing fast at registration beats failing
on a background thread an hour later."

**What separates them:** knowing the evaluation timing is specified behaviour, and
naming the *debugging* consequence — a misleading stack trace — not just the semantic
one.

**Interviewer's follow-up:** "Which would you use for a scheduled retry?" They want you
to pick eager plus an explicit `requireNonNull` with a message.

---

### Q3 — "Are method references faster than lambdas?"

**Mid-level answer:** "I think so — there's less code generated."

**Senior answer:** "No. Both compile to `invokedynamic` bootstrapped by
`LambdaMetafactory`; you can see it in `javap -v` under `BootstrapMethods`. The only
compile-time difference is that a method reference to an existing method often needs no
synthetic `lambda$...$0` adapter method, because the bootstrap handle can point
directly at the target. That saves a method in the class file and an indirection the
JIT inlines anyway. Capture behaviour is identical too — `parser::parse` where `parser`
is a field captures `this` and allocates, exactly like the equivalent lambda. So the
choice is a readability one: method reference when the body is a single forwarding call
and the name reads well at the call site, lambda when parameter names carry meaning or
arguments need reordering."

**What separates them:** refusing the premise with a checkable mechanism, and then
supplying the criterion that *does* matter.

**Interviewer's follow-up:** "When is a lambda strictly more readable?" Good answers:
when the parameter name documents the domain, or when there are three chained `::` in a
row and the reader loses track of what is flowing.

---

### Q4 — "Why doesn't `computeIfAbsent(key, ArrayList::new)` work?"

**Mid-level answer:** "It needs a lambda because the types don't match."

**Senior answer:** "`computeIfAbsent` takes a `Function<? super K, ? extends V>` — the
mapping function receives the *key*. So `ArrayList::new` is being resolved against a
one-argument target, which means the key is being passed to an `ArrayList`
constructor. With a `String` key there's no such constructor and you get a compile
error. With an `Integer` key it compiles, because unboxing reaches
`ArrayList(int initialCapacity)` — and now you have a silent bug: you get an empty list
with a strange capacity, not the list you meant. That second case is the dangerous one
and it's a code-review pattern I look for: a constructor reference wherever the input
is integer-typed. The fix is `k -> new ArrayList<>()`. The general habit is to check
the target interface's arity before writing `::new`."

**What separates them:** the `Integer`-key case that compiles. That is the difference
between "I hit this error once" and "I understand target typing".

**Interviewer's follow-up:** "Where does `ArrayList::new` work?" They want
`Collectors.toCollection`, `orElseGet`, `Supplier` positions — anything zero-arg.

---

### Q5 — "You see `auditLog::record` registered on a long-lived listener list. Any concerns?"

**Mid-level answer:** "Not really — it's just passing a method."

**Senior answer:** "Depends what `auditLog` is. If it's a local variable or a
parameter, the generated object captures only the `AuditLog` and I'm fine. If it's an
instance field — which it usually is in a Spring bean — reading it requires `this`, so
the captured object holds the entire enclosing bean, not just the audit log. On a
long-lived bus that's a retention leak; in a heap dump you see the enclosing class
retained through a `$$Lambda$` object in the listener list. I can check it without
running anything: `javap -c -p` and look for an `aload_0` immediately before the
`invokedynamic`. `aload_0` is `this`. If it's there, the bean is captured. The fix is
either a local copy of the collaborator, or — usually better in Spring — an explicit
unsubscribe in `@PreDestroy`, because the real problem is unbounded listener lifetime,
not the capture."

**What separates them:** the field-versus-local distinction, the static-analysis check
that needs no profiler, and identifying that the capture is a symptom of a lifetime
problem rather than the root cause.

**Interviewer's follow-up:** "How would you find this in an existing codebase?" Heap
dump plus MAT dominator tree (Topic 79), or a lint rule on listener registration.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. The unbound form moves the receiver into the first parameter. Why did the language
   designers choose that over requiring an explicit placeholder, like `String::_::length`
   or some equivalent? What did they trade away?

2. Forms 1 and 3 are syntactically identical. Given that ambiguity produces real
   compile errors, was making them look the same a mistake? Argue both sides.

3. `expr::method` evaluates `expr` eagerly, but a lambda defers. Both are defensible.
   What would break if method references were lazy instead?

4. A method reference cannot reorder its arguments — `(a, b) -> b.compareTo(a)` has no
   `::` form. Is that a limitation worth fixing, or is it a useful constraint? What
   would a fix look like?

5. `ArrayList::new` means three different constructors depending on target type. Compare
   that to how TypeScript would handle passing a class as a factory. Which is safer, and
   what is each one's failure mode?

6. Topic 21 said a lambda has no type of its own. Method references have exactly the
   same property. If Java added structural function types tomorrow, which of the four
   forms would still need to exist?

7. You are writing a code-review guideline for your team on when to use `::` versus a
   lambda. Write it in two sentences. Then find the case your rule gets wrong.

---

## Quick reference card

### The four forms

```java
Type::staticMethod        // 1. static            Long::parseLong
expr::instanceMethod      // 2. bound             logger::info,  this::retry,  super::toString
Type::instanceMethod      // 3. unbound           String::length  (receiver = first arg)
Type::new                 // 4. constructor       ArrayList::new
Type[]::new               // 4b. array ctor       Order[]::new    (IntFunction<Order[]>)
```

### Arity check — the one-second diagnosis

| Relationship | Form |
|---|---|
| target params **==** method params, method is `static` | 1 |
| target params **==** method params, receiver written before `::` | 2 |
| target params **==** method params **+ 1**, type written before `::` | 3 |
| target params **==** constructor params | 4 |
| none of the above | your compile error |

### Common references you will write constantly

```java
Objects::nonNull          Objects::isNull         Objects::requireNonNull
Objects::toString         Objects::equals         Objects::hash
String::trim              String::strip           String::isBlank      String::toUpperCase
String::valueOf           String::length          String::compareTo
Integer::parseInt         Long::parseLong         Long::sum            Integer::compare
Math::max                 Math::min               Math::abs
ArrayList::new            HashMap::new            HashSet::new         TreeMap::new
Order[]::new              String[]::new
Function.identity()       // NOT Object::hashCode — identity() is the no-op Function
Comparator.comparing(Order::placedAt)
Comparator.comparingLong(Order::totalPence)        // no boxing — Topic 01
```

### Errors and what they mean

| Message | Meaning | Fix |
|---|---|---|
| `non-static method X() cannot be referenced from a static context` | You wrote form 3 but the target has no parameter for the receiver | Add the receiver (form 2) or fix the target type |
| `reference to X is ambiguous` | The type has both a static and an instance method with matching arity | Write the lambda explicitly |
| `invalid method reference — no suitable constructor` | `Type::new` against a target that passes an argument | Use `k -> new Type<>()` |
| `incompatible types: cannot infer type-variable(s)` | The target type is not determined at that position | Assign to a typed local first |
| `bad return type in method reference` | The method returns something the interface does not accept | Usually a `void`/value mismatch |

### Gotchas checklist

- [ ] `Type::method` and `Type::staticMethod` look identical. Arity disambiguates.
- [ ] `expr::method` evaluates `expr` **immediately** — NPE at creation, not invocation.
- [ ] A reference to a field's method (`field::m`) captures `this`, not the field.
- [ ] `aload_0` before `invokedynamic` in `javap -c -p` = `this` was captured.
- [ ] `::new` picks a constructor by target arity — the `int` capacity one is a trap.
- [ ] Method references cannot reorder or add arguments.
- [ ] `::` is not faster than a lambda; the mechanism is identical.
- [ ] `Comparator.comparingLong(Order::totalPence)` avoids boxing that
      `Comparator.comparing` would incur.

---

## When would I use this at work?

**1. Every comparator you ever write.**
`Comparator.comparing(Order::placedAt).thenComparing(Order::id)` is the standard Java
sort idiom, and it is entirely built from unbound method references. Sorting a page of
orders in a REST endpoint is a daily task; this is the shape it takes.

**2. Supplying containers to APIs that need them.**
`Collectors.toCollection(LinkedHashSet::new)` when you need insertion order preserved
through a grouping. `Optional.orElseGet(Order::empty)` when the fallback is expensive.
`stream.toArray(Product[]::new)` at a boundary where an array is required. You will
write `::new` in these positions constantly and never think about it again — once.

**3. Reading someone else's stream pipeline in review.**
Six chained method references with no parameter names is common in Java codebases.
Being able to say instantly "that's an unbound reference so the element is the
receiver, and that one's bound so it captured the service field" is the difference
between reviewing the code and skimming past it. It is also how you catch Trap 3's
capture leak before it ships.

---

## Connected topics

**Prerequisites:**
- **21 — Lambdas and functional interfaces**: method references are the same mechanism,
  the same target typing, and the same capture rules. Do not read this topic first.
- **01 — Primitives and boxing**: why `Comparator.comparingLong(Order::totalPence)`
  exists alongside `Comparator.comparing`.
- **04 — Interfaces**: `Comparator`'s `default` methods (`thenComparing`, `reversed`)
  are what make reference chains composable.
- **13 — equals/hashCode**: relevant the moment a method reference feeds a map key.
- **14 — Comparable/Comparator**: the biggest single consumer of unbound references.

**This unlocks:**
- **23 — Streams I**: `map(Order::id)`, `filter(Objects::nonNull)`,
  `forEach(auditLog::record)` — the pipeline vocabulary.
- **24 — Collectors**: `toCollection(TreeSet::new)`, `groupingBy(Order::customerId)`,
  and the four collector functions, which are frequently written as references.
- **25 — Parallel streams**: `Collector`'s supplier (`ArrayList::new`) and combiner are
  what the parallel path actually calls.
- **26 — Optional**: `map(Order::customerId)`, `orElseGet(Order::empty)`.
- **27 — Records**: record accessors (`Order::id`) are the ideal unbound-reference
  targets — that pairing is deliberate.
- **91 — `CompletableFuture`**: `thenApply(Order::confirm)`,
  `exceptionally(this::fallback)`.
- **79 — Heap dumps**: where you find the capture leak from Trap 3.

---

*Java baseline 21. Method-reference syntax and semantics have been stable since Java 8
and are unchanged through Java 25. All four forms, plus array constructor references
and `super::`, are available on 21 with no flags.*
