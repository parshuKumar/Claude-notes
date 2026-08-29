# 21 — Lambdas and Functional Interfaces

## Phase: 2 — Modern Java
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Imagine a job form with exactly one blank line on it: **"what to do"**.

Someone hands you a form that says at the top: *"I am a PriceRule. My one blank line
takes an order and gives back a number."* You fill in that one line. You hand it back.

That filled-in form is a **lambda**.

Three things follow from the picture, and they are the whole topic:

1. **The form came first.** You cannot write down "what to do" floating in mid-air.
   Somebody has to hand you a form with one blank line. In Java that form is a
   **functional interface**. The blank line is its single abstract method.

2. **The same words on two different forms are two different things.** Writing
   "add the tax" on a PriceRule form and on a DiscountRule form gives you two
   unrelated objects, even though the handwriting is identical. Java decides which
   form you filled in by looking at where you handed it — not at what you wrote.

3. **If your instructions mention something from your desk**, the form takes a
   photocopy of it at the moment you fill it in. It does not keep watching your desk.
   That photocopy rule is why Java refuses to let you write instructions that refer to
   something you are still changing.

In JavaScript there is no form. A function is just a value you can pass around. That
one difference explains almost every surprise below.

---

## The bridge from what you know

This is the strongest analogy in the entire curriculum. Use it hard, then unlearn
three precise things.

### What transfers, almost exactly

```ts
// TypeScript
const isHighValue = (order: Order) => order.totalPence() > 50_000;
orders.filter(isHighValue);
```

```java
// Java
Predicate<Order> isHighValue = order -> order.totalPence() > 50_000;
orders.stream().filter(isHighValue);
```

Same idea. Same reading experience. Same mental move: *behaviour as a value you pass
to something else*. Everything you know about callbacks, higher-order functions,
composition, and "pass a function instead of a flag" transfers directly.

**Verdict: HONEST ANALOGUE for the concept. PARTIAL for the mechanism.**

### Break 1 — a lambda is not a function value

In TypeScript, `(order: Order) => number` is a **type in its own right**. You can
declare it, alias it, and put it anywhere.

In Java there is no such type. `Order -> long` is not spellable. A lambda is always
an **instance of some interface**, and the compiler works out *which* interface from
the place you used it. That place is called the **target type**.

```java
Predicate<Order> a = order -> order.isPaid();   // becomes a Predicate
Function<Order, Boolean> b = order -> order.isPaid();  // same text, different type
```

`a` and `b` have no type relationship at all. You cannot assign one to the other. In
TypeScript both would be `(order: Order) => boolean` and interchangeable.

Consequence you will feel immediately: **you cannot write a variable of "function
type"** without naming an interface first. And a bare lambda with no target type is a
compile error:

```java
var f = order -> order.isPaid();   // does NOT compile: cannot infer type for var
```

There is nothing for `var` to infer *from*. The lambda has no type until something
tells it what to be.

### Break 2 — captured locals must be effectively final

This is the real difference and it deserves its own trap section below. Short version:

```ts
// TypeScript — fine
let count = 0;
orders.forEach(o => { count++; });
console.log(count);
```

```java
// Java — does not compile
int count = 0;
orders.forEach(o -> { count++; });   // error
```

JavaScript closures capture the **binding** — the variable itself. Java lambdas
capture the **value**, copied at the moment the lambda is created. Because a copy that
you can write to would be confusing (which one wins?), Java forbids writing to it at
all — and forbids the outer variable changing too.

**Verdict: NO ANALOGUE. This is a genuinely new rule.**

### Break 3 — `this` behaves like an arrow function, unlike an anonymous class

Here Java actually agrees with your instinct, and disagrees with older Java.

```java
class OrderService {
    private String name = "orders";

    void run() {
        Runnable lambda = () -> System.out.println(this.name);   // "orders"

        Runnable anon = new Runnable() {
            public void run() {
                System.out.println(this.getClass());  // the anonymous class, NOT OrderService
                // this.name would not compile — `this` is the anonymous instance
            }
        };
    }
}
```

A lambda has **no `this` of its own**. `this` inside it means the enclosing instance —
exactly like a TypeScript arrow function. An anonymous inner class *does* have its
own `this`, exactly like a TS `function () {}`.

So: **lambda ≈ arrow function, anonymous class ≈ `function`**. That mapping is
correct and worth memorising, because it also tells you when a lambda cannot replace
an anonymous class (when the body needs to refer to itself, or hold state).

### The summary table

| You know | Java | Verdict |
|---|---|---|
| Arrow function `(x) => y` | Lambda `x -> y` | **HONEST ANALOGUE** for reading and writing |
| Function type `(a: A) => B` | No such type; you name an interface | **NO ANALOGUE** — target typing replaces it |
| Closure over a mutable `let` | Effectively-final capture only | **NO ANALOGUE** — a new compile error |
| `this` in an arrow function | `this` in a lambda | **HONEST ANALOGUE** |
| `this` in a `function () {}` | `this` in an anonymous class | **HONEST ANALOGUE** |
| Every function is an object | Non-capturing lambdas are cached singletons | **PARTIAL** — allocation behaviour differs |
| Throwing anything from a callback | Checked exceptions do not fit lambdas | **NO ANALOGUE** — a real API design constraint |

---

## What is this?

**A functional interface** is an interface with exactly **one abstract method**. That
is the whole definition. It may also have any number of `default`, `static` and
`private` methods (Topic 04) — those do not count, because they already have bodies.

```java
@FunctionalInterface
public interface PricingRule {
    long apply(Order order, long currentTotalPence);   // exactly one abstract method
}
```

**A lambda** is an expression that produces an instance of a functional interface. The
compiler looks at where you put it, finds the expected interface, checks your
parameter list and return type against that interface's single method, and builds an
object that implements it.

```java
PricingRule flatFivePercentOff = (order, total) -> total - (total / 20);
```

That is it. There are no "function objects" in Java, only objects of an interface type
whose single method you supplied inline.

### What actually gets compiled

Here is the fact the master plan wants you to own:

> A lambda does **not** compile to an anonymous inner class. It compiles to a single
> `invokedynamic` instruction, whose call site is linked at first execution by
> `LambdaMetafactory`.

Concretely, `javac` does two things:

1. Moves your lambda body into a **synthetic private method** on the enclosing class,
   named something like `lambda$placeOrder$0`.
2. Emits an `invokedynamic` at the point where the lambda appears, with a bootstrap
   method pointing at `java.lang.invoke.LambdaMetafactory.metafactory`.

At runtime, the *first* time that line executes, `LambdaMetafactory` spins up a class
implementing the interface and returns an instance. The call site is then permanently
linked to that instance-producing code.

Why anyone cares:

- **No `Outer$1.class` file on disk.** An anonymous class creates one class file per
  lambda-like site at compile time. Lambdas do not. A class with 40 lambdas ships 1
  class file, not 41. You will verify this with `ls *.class` below.
- **Non-capturing lambdas are singletons.** If the lambda body uses nothing from its
  surroundings, `LambdaMetafactory` builds one instance and the call site returns that
  same instance forever. Zero allocation per call.
- **Capturing lambdas allocate.** If the body uses an enclosing local or `this`, those
  values are constructor arguments to the generated class, so a new instance is built
  each time the expression is evaluated.
- **The JVM keeps a translation strategy free hand.** Because the linkage is deferred
  to runtime, a future JVM could change how lambdas are represented without
  recompiling your code. That was the actual design motivation, not performance.

### The built-in functional interfaces

You will name these from memory. They live in `java.util.function`.

| Interface | Single method | Shape | You would have written |
|---|---|---|---|
| `Function<T,R>` | `R apply(T t)` | one in, one out | `(t: T) => R` |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | two in, one out | `(t: T, u: U) => R` |
| `Predicate<T>` | `boolean test(T t)` | one in, boolean out | `(t: T) => boolean` |
| `BiPredicate<T,U>` | `boolean test(T,U)` | two in, boolean out | |
| `Consumer<T>` | `void accept(T t)` | one in, nothing out | `(t: T) => void` |
| `BiConsumer<T,U>` | `void accept(T,U)` | two in, nothing out | |
| `Supplier<T>` | `T get()` | nothing in, one out | `() => T` |
| `UnaryOperator<T>` | `T apply(T t)` | `Function<T,T>` | `(t: T) => T` |
| `BinaryOperator<T>` | `T apply(T,T)` | `BiFunction<T,T,T>` | `(a: T, b: T) => T` |
| `Runnable` | `void run()` | nothing in, nothing out | `() => void` |
| `Callable<V>` | `V call() throws Exception` | like `Supplier` but may throw | |
| `Comparator<T>` | `int compare(T,T)` | Topic 14 | |

`UnaryOperator` and `BinaryOperator` are not new capabilities. They are named
shorthands, and they exist because `BinaryOperator<Long>` reads better than
`BiFunction<Long, Long, Long>` in a reduce signature.

### Primitive specialisations — and why they exist

This is Topic 01 coming back to collect.

`Function<Integer, Integer>` boxes on the way in and boxes on the way out. Every
single call allocates. So `java.util.function` also ships primitive variants:

```java
IntPredicate      // boolean test(int)      — no Integer allocated
IntFunction<R>    // R apply(int)
ToIntFunction<T>  // int applyAsInt(T)
IntUnaryOperator  // int applyAsInt(int)
IntBinaryOperator // int applyAsInt(int, int)
IntSupplier, IntConsumer
// and the same family for Long and Double
```

There are around 40 of these. Do not memorise the list. Memorise the **naming rule**:

- `IntXxx` — the *input* is `int`.
- `ToIntXxx` — the *output* is `int`.
- `IntToLongFunction` — input `int`, output `long`.

This is exactly why `IntStream`, `mapToInt` and `mapToObj` exist in Topic 23. They are
not stylistic alternatives; they are the boxing-free path.

---

## Why does it matter?

**1. Everything modern in Java is built on this.** Streams (23–25), `Optional` (26),
`CompletableFuture` (91), Spring's `RestClient`, JDBC template callbacks, Reactor
operators — all of them take functional interfaces. If lambdas are fuzzy, every one of
those APIs stays fuzzy.

**2. The capture rule changes how you write loops.** Coming from JS you will
reflexively accumulate into a mutable outer variable inside a callback. Java will stop
you at compile time, and the workaround most people reach for (`AtomicInteger`) hides
a design problem rather than solving one. Learning the right reflex here saves you
from a class of bug that outlives the compile error.

**3. Allocation in a hot path is a real cost.** A capturing lambda inside a loop that
runs ten million times is ten million objects. It is usually fine. It is occasionally
the whole problem. Knowing which lambdas allocate lets you answer that question
instead of guessing.

**4. It is the most-asked Phase 2 interview question.** "Is a lambda just an anonymous
class?" is a filter question. The answer is no, and the reasons are checkable in
`javap`.

---

## Syntax breakdown

### The lambda arrow, form by form

```java
// 1. one parameter, expression body, type inferred
order -> order.totalPence()

// 2. parentheses are optional for exactly one inferred parameter,
//    required for zero, two or more, or any explicit type
() -> LocalDateTime.now()
(order, discount) -> order.totalPence() - discount

// 3. explicit parameter types — legal, occasionally necessary
(Order order) -> order.totalPence()

// 4. `var` parameters — allows annotations on inferred params
(@NonNull var order) -> order.totalPence()

// 5. block body — needs braces, and needs `return` if non-void
order -> {
    long base = order.totalPence();
    long tax  = base / 5;
    return base + tax;
}

// 6. void body — an expression statement is allowed with no braces
order -> auditLog.record(order)
```

| Bit of syntax | What it means |
|---|---|
| `->` | Separates parameters from body. Java's arrow. Not `=>`. |
| `()` | Empty parameter list. Required — you cannot write ` -> expr`. |
| No braces | Expression body. Its value is the return value. |
| Braces | Block body. You must `return` explicitly unless the method is `void`. |
| Mixed types | **Illegal.** All parameters must be all-inferred, all-explicit, or all-`var`. `(Order o, discount) -> ...` does not compile. |

### `@FunctionalInterface`

```java
@FunctionalInterface
public interface PricingRule {
    long apply(Order order, long currentTotalPence);

    default PricingRule andThen(PricingRule next) {          // allowed
        return (order, total) -> next.apply(order, this.apply(order, total));
    }

    static PricingRule noOp() { return (order, total) -> total; }   // allowed
}
```

The annotation is **optional and non-load-bearing at runtime**. Any interface with one
abstract method can be a lambda target whether or not it is annotated.

What the annotation buys you is a **compile error if someone adds a second abstract
method**. That is the point: it declares "this interface is part of my API's lambda
surface, do not break it". Use it on every interface you intend callers to implement
with a lambda. Do not use it on interfaces that merely happen to have one method today.

Two rules that trip people:

- Methods that override a `public` method of `java.lang.Object` do not count toward
  the one-abstract-method budget. That is why `Comparator<T>` is a functional
  interface despite declaring `boolean equals(Object)`.
- `default`, `static` and `private` interface methods do not count. They have bodies.

### Capture — the exact rule

```java
void placeOrder(Order order) {
    long shippingPence = quoteShipping(order);       // effectively final: never reassigned

    Supplier<Long> total = () -> order.totalPence() + shippingPence;   // legal
}
```

**Effectively final** means: the compiler can see that you never assign to the variable
after its initialisation. You do not have to write `final`; you just have to not
reassign it.

What can be captured:

- A local variable that is `final` or effectively final. **Value copied at capture.**
- A parameter, under the same rule.
- `this`, and therefore any instance field — **by reference, not by value**. Fields are
  read live, every time the lambda runs, because what was captured is the object
  reference, not the field value.
- Any `static` field, read live.

That asymmetry is important and catches people:

```java
class OrderTotals {
    private long runningTotal = 0;         // a field, not a local

    Runnable adder(long amount) {
        return () -> runningTotal += amount;   // COMPILES. Fields are not restricted.
    }
}
```

The effectively-final rule applies **only to locals and parameters**. A field can be
mutated from a lambda all day long. Whether it *should* be is a different question —
see Trap 1.

### Ambiguity and target typing

Because the lambda has no type of its own, an overloaded method can make it
unresolvable:

```java
void schedule(Runnable task)      { ... }   // void, no args
void schedule(Callable<String> t) { ... }   // returns String, may throw

schedule(() -> loadReport());     // ambiguous IF loadReport() returns String
```

The compiler cannot pick. The fix is to make the target explicit:

```java
schedule((Runnable) () -> loadReport());
// or
Callable<String> task = () -> loadReport();
schedule(task);
```

You will meet this most often with `ExecutorService.submit`, which has exactly this
overload pair.

---

## Example 1 — minimal

A single functional interface, a lambda, and a call.

```java
public class MinimalLambda {

    @FunctionalInterface
    interface StockCheck {
        boolean hasEnough(int available, int requested);
    }

    static String decide(int available, int requested, StockCheck rule) {
        return rule.hasEnough(available, requested) ? "ACCEPT" : "REJECT";
    }

    public static void main(String[] args) {
        StockCheck strict  = (available, requested) -> available >= requested;
        StockCheck backOrder = (available, requested) -> true;

        System.out.println(decide(3, 5, strict));      // REJECT
        System.out.println(decide(3, 5, backOrder));   // ACCEPT
    }
}
```

Read the shape, not the logic. `decide` takes **behaviour** as a parameter. In
TypeScript you would have typed that parameter `(a: number, r: number) => boolean`. In
Java you had to declare `StockCheck` first, and that declaration is the price of
nominal typing (Topic 02).

---

## Example 2 — production scenario

`orderflow` needs a pricing pipeline. Marketing keeps adding rules: a percentage
promotion, a flat voucher, free shipping over a threshold, a loyalty tier discount.
Each rule must be independently testable, independently toggleable, and composable in
a defined order.

### The version people write first

```java
public class PricingService {

    public long priceOrder(Order order, PricingContext ctx) {
        long total = order.subtotalPence();

        if (ctx.promotionActive()) {
            total = total - (total * ctx.promotionPercent() / 100);
        }
        if (ctx.voucherPence() > 0) {
            total = Math.max(0, total - ctx.voucherPence());
        }
        if (total < 5_000) {
            total = total + 499;   // shipping
        }
        if (ctx.loyaltyTier() == LoyaltyTier.GOLD) {
            total = total - (total * 5 / 100);
        }
        return total;
    }
}
```

This works. It is also untestable rule-by-rule, the order is implicit in the source,
and every new campaign is an edit to a shared method. You know this shape; it is the
same problem you would solve in Nest with an array of transform functions.

### The functional-interface version

```java
@FunctionalInterface
public interface PricingRule {

    /** Returns the new total, in pence, given the order and the running total. */
    long apply(Order order, long currentTotalPence);

    /** Compose: this rule, then the next one. */
    default PricingRule andThen(PricingRule next) {
        return (order, total) -> next.apply(order, this.apply(order, total));
    }

    static PricingRule identity() {
        return (order, total) -> total;
    }

    static PricingRule chain(List<PricingRule> rules) {
        return rules.stream().reduce(identity(), PricingRule::andThen);
    }
}
```

```java
public final class PricingRules {

    private PricingRules() {}

    public static PricingRule percentOff(int percent) {
        if (percent < 0 || percent > 100) {
            throw new IllegalArgumentException("percent out of range: " + percent);
        }
        return (order, total) -> total - (total * percent / 100);
    }

    public static PricingRule voucher(long voucherPence) {
        return (order, total) -> Math.max(0, total - voucherPence);
    }

    public static PricingRule shippingUnder(long thresholdPence, long shippingPence) {
        return (order, total) -> total < thresholdPence ? total + shippingPence : total;
    }

    /** Non-capturing: this lambda uses nothing from its surroundings. */
    public static final PricingRule ROUND_UP_TO_PENNY =
            (order, total) -> total;   // already integral pence; kept for illustration
}
```

```java
public class PricingService {

    private final PricingRule pipeline;

    public PricingService(PricingCatalogue catalogue) {
        this.pipeline = PricingRule.chain(catalogue.activeRules());
    }

    public long priceOrder(Order order) {
        return pipeline.apply(order, order.subtotalPence());
    }
}
```

What each part bought:

| Change | What it gives you |
|---|---|
| `PricingRule` is a named interface | Nominal identity. A `Comparator<Order>` with the same shape cannot be passed by accident (Topic 02). |
| `default andThen` | Composition lives on the type, not in a utility class. |
| `PricingRules.percentOff(20)` returns a lambda | A **capturing** lambda — it captures `percent`. One allocation per rule constructed, not per order priced. That is the right place to pay it. |
| `ROUND_UP_TO_PENNY` is a `static final` field | A **non-capturing** lambda, built once at class-init time. Zero allocation forever. |
| `chain(...)` built once in the constructor | The composition cost is paid at startup, not per request. |

That last row is the senior move. A naive implementation calls
`PricingRule.chain(catalogue.activeRules())` inside `priceOrder`, which rebuilds the
whole lambda chain on every single order. Same output, N times the allocation. The
lambda knowledge is what tells you where the allocation lives.

> Note what is **not** in the interface: no `throws`. If a rule needs to call something
> that throws a checked exception, this signature cannot express it. That is Trap 4,
> and it is a genuine API design constraint you must decide about up front — see
> Topic 09.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — mutating a captured local (and the `AtomicInteger` "fix")

**Wrong:**
```java
long acceptedCount = 0;

orders.forEach(order -> {
    if (inventory.reserve(order)) {
        acceptedCount++;          // <-- the defect
    }
});
```

**Exact symptom:** it does not compile. `javac` says:

```
error: local variables referenced from a lambda expression must be final or
       effectively final
```

pointing at `acceptedCount` inside the lambda.

**Root cause:** the lambda captured the *value* of `acceptedCount` when it was created.
There is no shared binding, only a copy in the generated class. Java forbids writing to
that copy because it would silently diverge from the outer variable — a bug that JS
avoids only because JS captures the binding itself.

**The workaround people reach for:**
```java
AtomicLong acceptedCount = new AtomicLong();
orders.forEach(order -> {
    if (inventory.reserve(order)) {
        acceptedCount.incrementAndGet();
    }
});
```

This compiles. The reference `acceptedCount` never changes, so it is effectively final;
you mutate the *object* it points at.

**Why that is usually the wrong answer.** Two reasons:

1. It reintroduces exactly the shared-mutable-state problem the restriction was
   pointing at. If this loop ever becomes `orders.parallelStream()` (Topic 25), the
   `AtomicLong` will still be correct — but any *other* mutable state in that lambda
   will not be, and you have taught yourself the pattern is fine.
2. It is almost always a sign you wanted a reduction, not a loop.

**Fix — say what you actually meant:**
```java
long acceptedCount = orders.stream()
        .filter(inventory::reserve)
        .count();
```

or, when you need the accepted orders too:

```java
List<Order> accepted = orders.stream()
        .filter(inventory::reserve)
        .toList();
long acceptedCount = accepted.size();
```

**When `AtomicLong` genuinely is right:** when the counter is legitimately shared
across threads and its lifetime is not the loop — a metrics counter, a rate limiter,
an idempotency guard. The test is: *does the atomic outlive the lambda?* If yes,
keep it. If it was created two lines above purely to dodge the compiler, you have a
reduction in disguise.

> Aside: `inventory::reserve` above has a side effect inside a `filter`, which is its
> own smell. Topic 23 covers why side-effecting stream operations are a trap; here it
> is kept only to keep the diff focused on capture.

---

### Trap 2 — assuming `this` means the lambda

**Wrong:**
```java
public class RetryingPaymentClient {

    private final String name = "payments";

    public Runnable retryTask() {
        return new Runnable() {
            public void run() {
                log.info("retrying {}", this.name);   // does not compile
            }
        };
    }
}
```

**Exact symptom:** `error: cannot find symbol — symbol: variable name, location:
class RetryingPaymentClient$1`. Note the class name in the error: `$1`. That is the
anonymous class the compiler generated, and `this` refers to *it*.

**Root cause:** an anonymous inner class is a real class with its own `this`. Yours is
called `RetryingPaymentClient$1` and it has no field called `name`.

**Fix (in the anonymous class):** `RetryingPaymentClient.this.name`.

**Better fix — use a lambda, where the problem does not exist:**
```java
public Runnable retryTask() {
    return () -> log.info("retrying {}", this.name);   // `this` is the RetryingPaymentClient
}
```

**Why this matters beyond the compile error.** A lambda that mentions `this` — even
implicitly, by reading a field — captures the **enclosing instance**. If you store that
lambda in a long-lived registry, you have pinned the whole enclosing object in memory:

```java
class OrderPageController {
    private final byte[] renderedPage = new byte[8_000_000];   // big
    private final String tenantId;

    void register(EventBus bus) {
        bus.onOrderPlaced(order -> audit(tenantId, order));    // captures `this`!
    }
}
```

`tenantId` is a field, so the lambda captured `this`, so the 8 MB array is now reachable
for as long as the bus holds the listener. **Exact symptom:** a heap dump (Topic 79)
shows `OrderPageController` instances retained by a `lambda$register$0` reference from
the event bus, long after the request ended.

**Fix:** copy what you need into a local first, so only that value is captured.
```java
void register(EventBus bus) {
    String tenant = this.tenantId;                  // local copy
    bus.onOrderPlaced(order -> audit(tenant, order));  // captures only the String
}
```

---

### Trap 3 — capturing a lambda inside a hot loop

**Wrong:**
```java
for (OrderLine line : lines) {                       // 5,000,000 lines
    long lineId = line.id();
    repository.findProduct(line.productId())
              .ifPresent(p -> auditLog.record(lineId, p.sku()));   // captures lineId
}
```

**Exact symptom:** no error, correct results. A profiler (Topic 78) shows a
meaningful share of CPU in allocation; a GC log shows a high young-collection rate;
an allocation profile names `PricingService$$Lambda$14` or similar as a top allocator.

**Root cause:** the lambda captures `lineId`, so `LambdaMetafactory`'s generated class
must be instantiated with that value. One instance per iteration.

**Fix, in order of preference:**

1. **Do nothing.** Five million small short-lived objects die in the young generation
   and cost very little. Escape analysis (Topic 75) may remove the allocation entirely.
   This is the correct answer the overwhelming majority of the time.
2. If a profiler actually blames it, make the lambda non-capturing by passing the data
   through the call instead of the closure — often that means a method reference
   (Topic 22) or restructuring so the loop body is a plain method call.
3. Hoist the whole thing out of the loop if the same behaviour applies to every
   iteration.

**The point of this trap is the diagnosis, not the fix.** Knowing that
*capturing = allocating* means you can read `$$Lambda$` entries in an allocation
profile and know instantly what they are.

---

### Trap 4 — a checked exception inside a lambda

**Wrong:**
```java
List<Receipt> receipts = orderIds.stream()
        .map(id -> receiptStore.load(id))   // load() throws IOException
        .toList();
```

**Exact symptom:**
```
error: unreported exception java.io.IOException; must be caught or declared to be thrown
```
on the `.map(...)` line — even though your enclosing method declares `throws IOException`.

**Root cause:** `Function.apply` is declared `R apply(T t)` with **no `throws` clause**.
Your lambda is implementing that method. A method cannot throw a checked exception its
interface does not declare. The enclosing method's `throws` is irrelevant — the lambda
body is a different method.

**Fix A — wrap at the boundary (most common):**
```java
List<Receipt> receipts = orderIds.stream()
        .map(id -> {
            try {
                return receiptStore.load(id);
            } catch (IOException e) {
                throw new ReceiptUnavailableException(id, e);   // unchecked domain exception
            }
        })
        .toList();
```

**Fix B — a checked-throwing functional interface of your own:**
```java
@FunctionalInterface
interface ThrowingFunction<T, R, E extends Exception> {
    R apply(T t) throws E;
}

static <T, R> Function<T, R> unchecked(ThrowingFunction<T, R, ? extends Exception> f) {
    return t -> {
        try {
            return f.apply(t);
        } catch (Exception e) {
            throw new UncheckedIOException0(e);   // pick your wrapper
        }
    };
}
```

Be honest about Fix B: it is clever, it is used by libraries, and it makes stack traces
harder to read. Prefer Fix A in application code.

**Fix C — do not use a stream.** A plain `for` loop can propagate the checked exception
untouched. When the operation genuinely throws checked exceptions and you genuinely
want them propagated, the loop is the honest tool.

This is the concrete, mechanical reason Spring made its entire data-access hierarchy
unchecked (Topic 09). Checked exceptions do not compose through functional interfaces.

---

### Trap 5 — reaching for a lambda where you needed state

**Wrong:**
```java
Comparator<Order> byMostRecentThenValue = (a, b) -> { ... 30 lines ... };
```
or
```java
Runnable poller = () -> {
    // needs to remember its own retry count between invocations
};
```

**Exact symptom:** for the second case, either it does not compile (you cannot mutate a
captured local) or you smuggle state in via an `AtomicInteger` and the class becomes
unreadable. For the first, nobody can test the comparator's clauses independently.

**Root cause:** a lambda is a *stateless* implementation of one method. If the thing
needs fields, a name, its own `this`, or independent tests, it is a class.

**Fix:** write the class.
```java
final class RetryingPoller implements Runnable {
    private int attempts = 0;
    @Override public void run() { ... }
}
```

**The rule:** a lambda that does not fit on about three lines, or that wants to remember
something, has outgrown being a lambda. `Comparator.comparing(...).thenComparing(...)`
(Topic 14) covers most comparator cases; anything more goes in a named type.

---

## Hands-on proof

Everything below is a command **you** run. I have no JVM here and will not print
invented output. What follows is exactly what to type, exactly what to look for, and
how to read each possible result.

### Setup

```bash
mkdir -p ~/java-lab/21 && cd ~/java-lab/21
java --version    # expect 21 or 25
```

### Proof 1 — a lambda is not an anonymous class (the class-file count)

`TwoStyles.java`:
```java
import java.util.function.Predicate;

public class TwoStyles {

    static Predicate<String> asLambda() {
        return sku -> sku.startsWith("SKU-");
    }

    static Predicate<String> asAnonymousClass() {
        return new Predicate<String>() {
            @Override public boolean test(String sku) {
                return sku.startsWith("SKU-");
            }
        };
    }

    public static void main(String[] args) {
        System.out.println(asLambda().test("SKU-1"));
        System.out.println(asAnonymousClass().test("SKU-1"));
    }
}
```

```bash
javac TwoStyles.java
ls *.class
```

**What to look for:** how many `.class` files exist.

| What you see | What it means |
|---|---|
| `TwoStyles.class` **and** `TwoStyles$1.class` | Expected. The `$1` file is the anonymous class, generated at compile time. There is **no** file for the lambda — it has no class of its own on disk. |
| Only `TwoStyles.class` | You compiled a version with no anonymous class. Re-check the source. |
| `TwoStyles$1.class` and `TwoStyles$2.class` | You have two anonymous classes. The lambda still produced none. |

**How to read it:** the count is the proof. Add ten more lambdas to the file, recompile,
and the file count does not change. Add one more anonymous class and it goes up by one.
That asymmetry is the entire "lambdas are not inner classes" claim, made visible with
`ls`.

### Proof 2 — see the `invokedynamic` and the `LambdaMetafactory` bootstrap

```bash
javap -c -p TwoStyles.class
```

**What to look for**, in the disassembly of `asLambda()`:

- An `invokedynamic` instruction. It will reference a bootstrap method index, printed
  like `#N,  0` next to a method name such as `test`.
- **No** `new` instruction and **no** `invokespecial ... <init>` for a class you wrote.

Then, in the disassembly of `asAnonymousClass()`:

- A `new` instruction naming `TwoStyles$1`.
- `invokespecial TwoStyles$1.<init>`.

| What you see in `asLambda` | What it means |
|---|---|
| `invokedynamic #N, 0  // InvokeDynamic #0:test:()Ljava/util/function/Predicate;` | Confirmed. The lambda is a dynamic call site producing a `Predicate`; nothing is constructed at this bytecode. |
| `new #M // class TwoStyles$1` | You are reading the wrong method. Scroll to `asLambda`. |
| `getstatic` of a field, then `areturn` | Also possible on some builds where the compiler hoists a constant lambda. Rare; if you see it, note it and move on. |

To see the bootstrap method that links the call site:

```bash
javap -c -p -v TwoStyles.class | sed -n '/BootstrapMethods/,$p'
```

**What to look for:** a `BootstrapMethods:` section naming
`java/lang/invoke/LambdaMetafactory.metafactory`, with three static arguments — the
erased method signature, a `REF_invokeStatic` or `REF_invokeSpecial` handle pointing at
`TwoStyles.lambda$asLambda$0`, and the instantiated signature.

| What you see | What it means |
|---|---|
| `LambdaMetafactory.metafactory` in `BootstrapMethods` | This is the linkage mechanism. At runtime, the first execution of that `invokedynamic` calls this and gets back a `CallSite`. |
| `LambdaMetafactory.altMetafactory` | Same thing, extended form. Used for serializable lambdas, intersection types, or extra marker interfaces. |
| `StringConcatFactory.makeConcatWithConstants` | That is string concatenation (Topic 18), not a lambda. Different bootstrap, same `invokedynamic` mechanism. |

### Proof 3 — find the synthetic method your lambda body became

```bash
javap -p TwoStyles.class
```

`-p` means "show private and synthetic members".

**What to look for:** a method whose name starts with `lambda$`.

| What you see | What it means |
|---|---|
| `private static boolean lambda$asLambda$0(java.lang.String);` | Your lambda body. `javac` moved it into a real private method on `TwoStyles`. `static` because the body captured nothing — no `this` needed. |
| `private boolean lambda$asLambda$0(java.lang.String);` (no `static`) | The body captured `this`. This is the instance-capture case. |
| Nothing starting with `lambda$` | You forgot `-p`. Synthetic members are hidden without it. |

**How to read it:** the naming is `lambda$<enclosingMethod>$<index>`. This is why
lambda frames appear in stack traces as `MyClass.lambda$placeOrder$3` — now that string
is readable rather than mysterious.

### Proof 4 — non-capturing lambdas are the same instance; capturing ones are not

`CaptureIdentity.java`:
```java
import java.util.function.Predicate;

public class CaptureIdentity {

    static Predicate<String> nonCapturing() {
        return sku -> sku.startsWith("SKU-");
    }

    static Predicate<String> capturing(String prefix) {
        return sku -> sku.startsWith(prefix);
    }

    public static void main(String[] args) {
        System.out.println("non-capturing same instance? "
                + (nonCapturing() == nonCapturing()));

        System.out.println("capturing same instance?     "
                + (capturing("SKU-") == capturing("SKU-")));

        System.out.println("non-capturing class: " + nonCapturing().getClass().getName());
        System.out.println("capturing class:     " + capturing("X").getClass().getName());
    }
}
```

```bash
java CaptureIdentity.java
```

**What to look for:** the two boolean lines, and the two class names.

| What you see | What it means |
|---|---|
| non-capturing `true`, capturing `false` | The expected result. The non-capturing call site is linked to one cached instance; the capturing one builds a new object per evaluation, because the captured `prefix` is constructor state. |
| both `false` | Possible in principle — the caching of non-capturing lambdas is a HotSpot implementation choice, not a spec guarantee. Note it honestly and do not build a design on the identity. |
| Class names containing `$$Lambda` | Correct. These are the runtime-spun classes. They have no `.class` file on disk, which is why Proof 1's `ls` showed nothing. |
| The two class names differ from each other | Expected — two distinct call sites, two distinct generated classes. |

**How to read it:** `false` on line 2 is the allocation you were told about in Trap 3,
observed directly. Nothing else in this file changed except whether the body mentions
`prefix`.

> Do not write production code that depends on lambda identity. This experiment is
> diagnosis, not a contract. The JVM is explicitly free to change it.

### Proof 5 — the capture rule is a compile-time rule, not a runtime one

`CaptureRule.java`:
```java
import java.util.List;

public class CaptureRule {
    public static void main(String[] args) {
        List<String> skus = List.of("SKU-1", "SKU-2", "SKU-3");

        int count = 0;
        skus.forEach(sku -> count++);      // remove this line to compile
        System.out.println(count);
    }
}
```

```bash
javac CaptureRule.java
```

**What to look for:** the exact error text.

| What you see | What it means |
|---|---|
| `local variables referenced from a lambda expression must be final or effectively final` | The message you must be able to recognise instantly. It names the variable and the line. |
| It compiles | You changed `count` to a field or made it an array/atomic. That is Trap 1's workaround — check whether you meant a reduction. |

Now prove *effectively final* is about assignment, not the `final` keyword: remove the
`count++`, leave `count` as a plain `int`, and use it inside the lambda. It compiles
with no `final` anywhere. Then add `count = 5;` **after** the lambda and watch it break
again — capture is checked against the whole method body, not just the lines above.

---

## Practice exercises

Write real files, run them, and keep your output.

### 1 — Easy: build the interface, then break it

**Part A.** Define `@FunctionalInterface interface SkuValidator { boolean isValid(String sku); }`.
Write three implementations as lambdas: prefix check, length check, and one that
requires both. Write a method `List<String> rejects(List<String> skus, SkuValidator v)`.

**Part B.** Add a second abstract method to `SkuValidator`. Record the exact compile
error and which line it points at. Remove it, then instead add a `default` method and
a `static` method and confirm both compile.

**Part C.** Try to write `var v = sku -> sku.startsWith("SKU-");`. Record the error and
explain in one sentence why the compiler cannot help you here but TypeScript could.

### 2 — Medium: the audit (combines Topics 01, 02, 04, 08, 13, 14)

Below is a fragment from a payment reconciliation job. It contains **six** distinct
defects drawn from Topics 01–21. Find them, state the exact symptom each produces (a
compile error message, an exception type and message, or an observable production
behaviour — not "bad practice"), and rewrite it correctly.

```java
public class ReconciliationJob {

    private final List<Long> settledPaymentIds = new ArrayList<>();

    public double reconcile(List<Payment> payments, Long expectedTotal) {

        double total = 0.0;

        payments.forEach(p -> {
            for (Long settled : settledPaymentIds) {
                if (settled == p.getId()) {
                    return;
                }
            }
            total = total + p.getAmount();
            settledPaymentIds.add(p.getId());
        });

        Comparator<Payment> byAmount = new Comparator<Payment>() {
            public int compare(Payment a, Payment b) {
                return (int) (a.getAmount() - b.getAmount());
            }
        };
        payments.sort(byAmount);

        Runnable report = () -> {
            System.out.println("expected " + expectedTotal
                    + " actual " + total
                    + " on " + this.getClass().getSimpleName());
        };
        report.run();

        return total - expectedTotal;
    }
}
```

Hints, one per topic, no more: think about `==` on boxed types (01); think about what
`(int)` does to a `long` difference and what `Comparator` requires (14); think about
what `return` means inside a lambda passed to `forEach` (21); think about `double` and
money (01); think about what `this` refers to in that lambda versus in the anonymous
class above it (21); think about the effectively-final rule (21).

### 3 — Hard: production simulation on `orderflow`

Build the pricing pipeline from Example 2 for real, then measure the thing you were
told about.

**Part A.** Implement `PricingRule`, `PricingRules`, and a `PricingService` that
composes a chain of five rules: 10% promotion, £5 voucher, £4.99 shipping under £50,
5% gold-tier discount, and a floor at zero. Write unit tests for each rule in
isolation, and one test for the composed order.

**Part B.** Write two versions of `PricingService.priceOrder`:
- **V1:** builds the chain inside `priceOrder` on every call.
- **V2:** builds the chain once in the constructor.

Price 5,000,000 synthetic orders with each. Run both under:
```bash
java -Xlog:gc:file=v1.log:time,uptime -Xmx512m PricingBench v1
java -Xlog:gc:file=v2.log:time,uptime -Xmx512m PricingBench v2
```
Report young-collection counts and total time from the logs.

**Part C.** Add `-XX:+PrintCompilation` to one run and skim for
`PricingRules$$Lambda` entries. Note whether the lambda bodies got JIT-compiled and
inlined.

**Part D.** Now the judgement call. Suppose V1 and V2 measure nearly identically. Give
the argument for **still** writing V2 that does not appeal to performance. Then give
the strongest argument against your own answer.

**Part E.** Make one rule need to call a service that throws a checked
`PricingCatalogueUnavailableException`. Show what breaks, then choose between Trap 4's
Fix A, B and C. Write down which you chose and the cost you accepted.

> Note: you are timing with wall clock again. Topic 77 will show you why those numbers
> are not trustworthy. Keep the logs; you will revisit them.

---

## Interview questions

### Q1 — "Is a lambda just syntactic sugar for an anonymous inner class?"

**Mid-level answer:** "Basically yes — it's a shorter way to write a single-method
class. The compiler generates the class for you."

**Senior answer:** "No, and the difference is checkable. An anonymous class produces a
`.class` file at compile time — `Outer$1.class` — and a `new` plus `invokespecial` at
the call site. A lambda produces neither. `javac` moves the body into a synthetic
private method, `lambda$method$0`, and emits a single `invokedynamic` whose bootstrap
is `LambdaMetafactory.metafactory`. The class is spun at runtime on first execution.
Three consequences: no class-file explosion, non-capturing lambdas get cached as a
single instance so they allocate nothing, and the JVM keeps freedom to change the
representation without recompiling anyone's code — which was the actual design goal.
There's also a semantic difference: `this` in a lambda is the enclosing instance,
whereas an anonymous class has its own `this`."

**What separates them:** the mid answer describes the source. The senior answer
describes the bytecode, names the mechanism, and gives a semantic difference on top of
the mechanical one — and every claim is verifiable with `javap` and `ls`.

**Interviewer's follow-up:** "So when would you still write an anonymous class?" They
want: when you need state, when you need `this` to mean the callback itself, or when
you must implement an interface with more than one abstract method.

---

### Q2 — "Why does this not compile, and what would you write instead?"

```java
int total = 0;
lines.forEach(line -> total += line.amountPence());
```

**Mid-level answer:** "Because `total` isn't final. Java requires captured variables to
be final or effectively final. I'd use an `AtomicInteger`."

**Senior answer:** "It doesn't compile because a lambda captures the *value* of a local,
not the binding — the value becomes constructor state on the generated class. Allowing
writes would give you two diverging copies, so the language forbids it outright. The
`AtomicInteger` trick compiles because you're mutating the object rather than the
reference, but I'd treat reaching for it as a signal I've written a reduction as a
loop. Here I'd write `lines.stream().mapToLong(Line::amountPence).sum()` — and
`mapToLong` specifically, so the running total never gets boxed. An atomic is right
when the counter genuinely outlives the lambda and is genuinely shared across threads;
it's wrong when it was created two lines earlier purely to satisfy the compiler."

**What separates them:** the mid answer knows the rule and the workaround. The senior
answer knows *why* the rule exists, treats the workaround as a diagnostic signal
rather than a solution, and reaches for the primitive-specialised stream without being
asked — connecting back to boxing.

**Interviewer's follow-up:** "JavaScript lets you do this. Why doesn't Java?" They want
capture-by-value versus capture-by-binding, and ideally the observation that Java's
choice is what makes lambdas safe to hand to another thread.

---

### Q3 — "Which lambdas allocate?"

**Mid-level answer:** "Lambdas are objects, so they all allocate. It's usually not a
problem."

**Senior answer:** "It depends on capture. A non-capturing lambda — the body uses
nothing from its enclosing scope — is instantiated once by `LambdaMetafactory` and the
`invokedynamic` call site returns that same instance forever. You can observe it:
`nonCapturing() == nonCapturing()` is `true` on HotSpot. A capturing lambda takes the
captured values as constructor arguments, so evaluating the expression builds a new
object each time. In practice that matters in two places: a capturing lambda inside a
hot loop, where you'd see `$$Lambda` classes near the top of an allocation profile;
and a lambda that captures `this` and gets stored somewhere long-lived, which retains
the whole enclosing object and shows up as a leak in a heap dump. I wouldn't
restructure code on that reasoning without a profile — escape analysis often removes
the allocation anyway."

**What separates them:** the capturing/non-capturing distinction, the observability
(what it looks like in a profile, what it looks like in a heap dump), and the refusal
to optimise without evidence.

**Interviewer's follow-up:** "How would you confirm the allocation is real rather than
scalar-replaced?" They want an allocation profiler — async-profiler in alloc mode, or
JFR's `ObjectAllocationSample` — not a stopwatch.

---

### Q4 — "Your `map` lambda needs to call a method that throws `IOException`. What happens?"

**Mid-level answer:** "You have to wrap it in a try/catch inside the lambda."

**Senior answer:** "It won't compile: `Function.apply` has no `throws` clause, and a
lambda implementing it can't throw a checked exception the interface doesn't declare.
The enclosing method's `throws` is irrelevant because the lambda body is a separate
method. Three options. Wrap it in the lambda and rethrow as an unchecked domain
exception — my default in application code, because the domain exception is a better
API anyway. Or define my own `ThrowingFunction` with a `throws E` and an adapter —
which works but degrades stack traces and adds a concept to the codebase. Or don't use
a stream: a plain `for` loop propagates the checked exception with no ceremony, and if
the operation genuinely throws and I genuinely want it propagated, the loop is more
honest. This is the mechanical reason Spring's whole data-access hierarchy is
unchecked — checked exceptions don't compose through functional interfaces."

**What separates them:** naming the precise reason (the interface's signature, not the
method's), giving three options with a stated default, and connecting it to a real
framework design decision.

**Interviewer's follow-up:** "Which would you pick for a library versus an
application?" They want you to notice those are different answers.

---

### Q5 — "What does `@FunctionalInterface` do?"

**Mid-level answer:** "It marks the interface so you can use it with lambdas."

**Senior answer:** "Nothing at runtime, and nothing at the use site either — any
interface with exactly one abstract method can be a lambda target, annotated or not.
What it does is make `javac` fail if someone adds a second abstract method. So it's a
declaration of intent: 'callers implement this with a lambda, so adding a method here
is a breaking change'. Two subtleties: methods overriding public `Object` methods
don't count toward the one-method budget — that's why `Comparator` qualifies despite
declaring `equals` — and `default`, `static` and `private` interface methods don't
count either, which is what lets `Comparator` and `Function` carry `andThen`,
`compose` and `thenComparing`. I put it on interfaces I intend as a lambda surface, not
on every single-method interface I happen to have."

**What separates them:** knowing the annotation is a compile-time contract on the
*author*, not an enabler for the *caller*, plus the two exclusion rules and a
concrete example of each.

**Interviewer's follow-up:** "Why do `Function` and `Comparator` have all those default
methods?" They want interface evolution (Topic 04) plus composition on the type.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Java could have added a real function type — `(Order) -> long` as a first-class
   type, the way TypeScript has. It deliberately did not, choosing target typing onto
   interfaces instead. Give the strongest argument for that choice, then the strongest
   argument against it.

2. `Predicate<Order>` and `Function<Order, Boolean>` have the same shape and are
   completely unrelated types. Given Topic 06 (erasure), what would actually break if
   Java made them assignable to each other?

3. A lambda captures locals by value but reads fields live. Explain why that asymmetry
   is not arbitrary — what would the alternative cost?

4. Non-capturing lambdas being cached singletons is an implementation choice, not a
   spec guarantee. Name one thing you must therefore never do, and one design decision
   you can still legitimately base on it.

5. You are reviewing a PR that replaces a 40-line anonymous `Comparator` with a
   `Comparator.comparing(...).thenComparing(...)` chain. Under what circumstances is
   that change a regression rather than an improvement?

6. Topic 01 taught you that boxing allocates. `IntPredicate` exists so that
   `Predicate<Integer>` is not the only option. Why did the JDK ship ~40 hand-written
   primitive interfaces instead of solving this generically?

7. A lambda has no `this` of its own. What does that make impossible to write as a
   lambda, and does that limitation ever cause you real trouble in practice?

---

## Quick reference card

### Lambda forms

```java
() -> expr                       // no parameters
x -> expr                        // one parameter, inferred, no parens needed
(x, y) -> expr                   // two or more, parens required
(Order x) -> expr                // explicit type
(var x) -> expr                  // var, allows annotations
x -> { stmt; return expr; }      // block body needs explicit return
x -> { stmt; }                   // void body
```

**Illegal:** mixing inferred and explicit parameter types in one lambda.

### The interfaces you will use daily

```java
Function<T,R>        R  apply(T)
BiFunction<T,U,R>    R  apply(T,U)
Predicate<T>         boolean test(T)
Consumer<T>          void accept(T)
Supplier<T>          T  get()
UnaryOperator<T>     T  apply(T)          // Function<T,T>
BinaryOperator<T>    T  apply(T,T)        // BiFunction<T,T,T>
Runnable             void run()
Callable<V>          V  call() throws Exception
Comparator<T>        int compare(T,T)
```

### Primitive variants — the naming rule

```
IntPredicate         boolean test(int)          // input is int
ToIntFunction<T>     int applyAsInt(T)          // output is int
IntUnaryOperator     int applyAsInt(int)        // both
IntToLongFunction    long applyAsLong(int)      // input int, output long
```
Same family for `Long` and `Double`. Reach for these whenever the value is a number
and the call is hot — Topic 01 explains what you are avoiding.

### Composition helpers

```java
predicate.and(other)      predicate.or(other)     predicate.negate()
Predicate.not(other)      Predicate.isEqual(x)
function.andThen(after)   function.compose(before)   Function.identity()
consumer.andThen(after)
BinaryOperator.minBy(cmp) BinaryOperator.maxBy(cmp)
Comparator.comparing(f).thenComparing(g).reversed()
```

Note the direction: `f.andThen(g)` is `g(f(x))`; `f.compose(g)` is `f(g(x))`.

### Capture rules

| Thing | Capturable? | Read how? |
|---|---|---|
| Local variable | Only if final or effectively final | Value, copied at capture |
| Parameter | Only if final or effectively final | Value, copied at capture |
| Instance field | Always | Live, via a captured `this` |
| Static field | Always | Live |
| `this` | Always | The **enclosing** instance |

### Gotchas checklist

- [ ] A lambda has no type of its own. `var f = x -> ...` never compiles.
- [ ] Captured locals must be effectively final; reassignment *anywhere* in the method breaks it.
- [ ] `AtomicX` to dodge that rule usually means you wanted a reduction.
- [ ] `this` in a lambda = enclosing instance. In an anonymous class = the anonymous instance.
- [ ] Capturing `this` (including reading any field) retains the enclosing object.
- [ ] Non-capturing lambdas are cached; capturing lambdas allocate per evaluation.
- [ ] Checked exceptions cannot escape a lambda unless the interface declares them.
- [ ] `return` inside a `forEach` lambda is `continue`, not `break`, and not a method return.
- [ ] Overloaded methods taking different functional interfaces can make a lambda ambiguous.
- [ ] `@FunctionalInterface` protects the author, not the caller.

---

## When would I use this at work?

**1. Replacing a boolean-flag parameter with behaviour.**
You find `chargeCustomer(order, true, false, true)` in the codebase. Nobody knows what
the booleans mean. Passing a `PricingRule` or a `Predicate<Order>` instead makes the
call site say what it does, and makes each behaviour independently testable. This is
the single most common day-to-day use, and it is the same refactor you would already
do in TypeScript.

**2. Reading a stack trace with `lambda$` in it.**
A production NPE has a frame `OrderService.lambda$placeOrder$2(OrderService.java:88)`.
Before this topic that string is noise. After it, you know exactly what it is: the
third lambda inside `placeOrder`, moved into a synthetic method, and line 88 is inside
the lambda body. Straight to the line.

**3. Reviewing a listener registration.**
Someone registers a lambda on a long-lived event bus, and the lambda reads an instance
field. You know that captures `this`, which retains the whole enclosing bean. You ask
for a local copy in review, instead of finding it in a heap dump six months later when
the service starts OOMing under sustained load.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and boxing**: why the ~40 primitive functional interfaces exist at
  all, and why `ToLongFunction` is not the same as `Function<T, Long>`.
- **02 — Nominal typing**: why you must name an interface before you can pass
  behaviour, where TypeScript needs nothing.
- **04 — Interfaces, default/static/private methods**: why `default andThen` on a
  functional interface is legal and does not break the single-abstract-method rule.
- **06 — Type erasure**: why `Predicate<Order>` and `Predicate<Payment>` are the same
  class at runtime, and why lambda generic bounds vanish in `javap`.
- **08/09 — Exceptions**: the checked-exception trap, and why Spring's data-access
  hierarchy is unchecked.

**This unlocks:**
- **22 — Method references**: the four forms, and how `Product::sku` replaces
  `p -> p.sku()`.
- **23 — Streams I**: every stream operation takes one of these interfaces.
- **24 — Collectors**: a `Collector` is literally four functional interfaces bundled
  together.
- **25 — Parallel streams**: why a lambda with captured mutable state is a correctness
  bug the moment `.parallel()` appears.
- **26 — Optional**: `map`, `flatMap`, `orElseGet`, `ifPresent` — all lambda-taking.
- **91 — `CompletableFuture`**: `thenApply` takes a `Function`, `thenAccept` a
  `Consumer`; the shapes you learned here are the whole API.
- **100 — `ForkJoinPool`**: the pool that runs the lambdas from Topic 25.
- **77 — JMH**: how to measure the allocation claims in this doc properly, instead of
  with the wall clock you used in Exercise 3.

---

*Java baseline 21. Lambdas and `java.util.function` have been stable since Java 8 and
nothing in this topic changed through Java 25. `LambdaMetafactory` remains the linkage
mechanism; if that ever changes, your source will not, which is precisely the point of
the `invokedynamic` design.*
