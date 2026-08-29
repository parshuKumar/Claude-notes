# 23 — Streams I: Pipeline Structure, Laziness, Intermediate vs Terminal

## Phase: 2 — Modern Java
## Category: CORE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Two ways to run a factory.

**Way one — the way your JavaScript array methods work.**
You have 10,000 raw parts in a bin. Station A processes all 10,000 and dumps them into
a new bin. Station B processes all 10,000 from that bin into a third bin. Station C
takes the first one it likes and you go home.

You built three bins. You did 30,000 units of work. You needed one part.

**Way two — the way a Java stream works.**
There are no bins between stations. One part goes onto the belt. It travels A → B → C
in one pass. Then the next part. When station C says "that one, I'm done", the belt
stops. Parts 2 through 10,000 were never touched.

Three consequences fall straight out of the picture:

1. **The belt does not move until someone at the end asks for output.** Building the
   stations does nothing. This is *laziness*, and it is the source of the most
   confusing bug in this topic — a pipeline you built that silently never ran.

2. **You cannot run the same belt twice.** The parts went through. There is no bin to
   go back to. A stream is consumed, permanently.

3. **A station that has to see everything before it can pass anything on** — sorting,
   for example — forces the belt to stop and pile up. Those stations are real, and
   they quietly delete the advantage.

That is the whole topic. Everything below is detail.

---

## The bridge from what you know

This is the analogy the master plan calls the central teaching point, so let us be
precise about where it holds and where it breaks.

### The surface is nearly identical

```ts
// TypeScript
const skus = orders
    .filter(o => o.status === "PAID")
    .map(o => o.sku)
    .slice(0, 10);
```

```java
// Java
List<String> skus = orders.stream()
    .filter(o -> o.status() == Status.PAID)
    .map(Order::sku)
    .limit(10)
    .toList();
```

Same reading experience. Same operations, mostly the same names. If you stop here you
will write correct Java most of the time.

### Break 1 — eager and materialising vs lazy and fused

`Array.prototype.filter` **returns a new array, right now**. So does `map`. Your
three-line chain above allocates two intermediate arrays and walks the data three
times.

A Java stream does neither. `filter` and `map` return a *stream*, which is a
description of work, not a container of results. Nothing executes. When the terminal
operation (`toList()`) finally runs, the elements are pushed through **all** the stages
one element at a time, in a single pass. There is no intermediate list at any point.

| | JS array methods | Java stream |
|---|---|---|
| When work happens | at each call | only at the terminal operation |
| Intermediate storage | a new array per step | none |
| Passes over the data | one per step | one, total |
| Can stop early | no (`some`/`find` can, `filter`+`map` cannot) | yes, throughout |
| Reusable | yes, it's an array | **no** |

**Verdict: PARTIAL ANALOGUE.** Transfer the vocabulary. Unlearn the execution model.

### Break 2 — short-circuiting reaches backwards

This is the payoff of laziness and it has no array-method equivalent:

```java
Optional<Order> first = orders.stream()
    .map(this::enrichWithPaymentStatus)    // an expensive remote call
    .filter(Order::isDisputed)
    .findFirst();
```

If the third order is disputed, `enrichWithPaymentStatus` runs **three times**, not
`orders.size()` times. The `findFirst` at the end reached back up the pipeline and
stopped the source.

The JavaScript equivalent runs `enrichWithPaymentStatus` on every single order before
`filter` even starts:

```ts
orders.map(enrich).filter(isDisputed)[0];   // enriches ALL of them
```

You would have to hand-write a loop, or use a generator, to get Java's behaviour. This
is the main reason streams are not just prettier loops.

### Break 3 — closest modern JS equivalents

Two honest partial matches worth knowing:

- **Generators and iterator helpers.** `Iterator.prototype.map/filter/take` (now
  shipping in modern engines) are lazy and single-pass, exactly like a stream. If you
  have used them, transfer that intuition directly. **HONEST ANALOGUE.**
- **`Object.groupBy`** ≈ `Collectors.groupingBy` — that pairing is Topic 24.

### The summary table

| You know | Java | Verdict |
|---|---|---|
| `array.map/filter/reduce` | `stream().map/filter/reduce` | **PARTIAL** — same names, different execution model |
| Iterator helpers / generators | Stream laziness | **HONEST ANALOGUE** |
| Reusing an array after mapping it | Reusing a stream | **NO ANALOGUE** — throws `IllegalStateException` |
| `for...of` with `break` | Short-circuiting terminal ops | **HONEST ANALOGUE** in effect, different in shape |
| `array.length` after a chain | `count()` — which may skip the pipeline entirely | **PARTIAL** — and this one is a trap |
| Numbers are numbers | `Stream<Long>` boxes; `LongStream` does not | **NO ANALOGUE** — Topic 01 returns |

---

## What is this?

A **stream** is a pipeline: a source, zero or more intermediate operations, and exactly
one terminal operation.

```java
List<String> result =
    orders.stream()                          // 1. SOURCE
          .filter(Order::isPaid)             // 2. INTERMEDIATE (lazy)
          .map(Order::sku)                   // 3. INTERMEDIATE (lazy)
          .distinct()                        // 4. INTERMEDIATE (lazy, stateful)
          .limit(10)                         // 5. INTERMEDIATE (lazy, short-circuiting)
          .toList();                         // 6. TERMINAL  <- everything runs here
```

Three things a stream is **not**:

- **Not a data structure.** It stores nothing. It has no `size()`. You cannot index it.
- **Not a collection.** It does not modify its source (with the exception of the
  side effects you put in, which you should not).
- **Not reusable.** One terminal operation per stream object. Ever.

### The execution model, mechanically

When the terminal operation runs, the JDK does *not* walk the pipeline stage by stage.
Internally each intermediate operation wraps the next one in a `Sink` — a small object
with `accept(element)` — producing a single chained consumer. The source's
`Spliterator` then pushes each element into the head of that chain.

Concretely, for `filter(p).map(f).forEach(c)` the machinery is closer to:

```java
for (Order o : orders) {           // conceptually — really a Spliterator
    if (p.test(o)) {               // filter's sink
        c.accept(f.apply(o));      // map's sink, then forEach's sink
    }
}
```

One loop. That is what "fused" means. It is also why streams can be roughly as fast as
a hand-written loop for simple cases — there is no intermediate structure to build.

Short-circuiting works because the sink chain can signal `cancellationRequested()`
upstream, and the source loop checks it.

> The `Spliterator` is the other half of this story: it is what knows how to iterate a
> source, and — crucially for Topic 25 — how to **split** it. You can ignore splitting
> entirely until then.

### Intermediate operations — the full working set

**Stateless** (each element handled independently; fully lazy, no buffering):

| Operation | Signature shape | What it does |
|---|---|---|
| `filter(Predicate<T>)` | `Stream<T>` | keeps matching elements |
| `map(Function<T,R>)` | `Stream<R>` | transforms each element |
| `mapToInt/Long/Double(ToXFunction<T>)` | `IntStream` etc. | transforms **and unboxes** |
| `mapToObj(IntFunction<R>)` | `Stream<R>` | back from a primitive stream |
| `flatMap(Function<T, Stream<R>>)` | `Stream<R>` | one element → zero or more |
| `mapMulti(BiConsumer<T, Consumer<R>>)` | `Stream<R>` | flatMap without allocating a stream per element (Java 16+) |
| `peek(Consumer<T>)` | `Stream<T>` | observe without changing — **debugging only** |
| `boxed()` | `Stream<Integer>` | primitive stream → boxed stream |

**Stateful** (must see some or all elements before emitting; these are where laziness
partially breaks down):

| Operation | Buffering behaviour |
|---|---|
| `sorted()` / `sorted(Comparator)` | **Consumes the entire upstream** into a buffer, sorts, then emits. Fully blocking. |
| `distinct()` | Keeps a `HashSet` of everything seen. Can emit as it goes on an ordered stream, but memory grows with distinct count. |
| `limit(n)` | Short-circuiting. Cheap on an ordered stream. |
| `skip(n)` | Must count the first `n`. Cheap. |
| `takeWhile(Predicate)` / `dropWhile(Predicate)` | Short-circuiting; ordered-stream semantics (Java 9+). |

**The practical rule that follows:** put `filter` before `sorted`, and `limit` as early
as correctness allows. `sorted().filter()` sorts everything then throws most of it
away. `filter().sorted()` sorts only what survives. Same result, different cost, and
the compiler will not reorder them for you.

### Terminal operations — the full working set

| Category | Operations | Short-circuits? |
|---|---|---|
| Collect | `toList()`, `collect(Collector)`, `toArray()`, `toArray(IntFunction)` | no |
| Reduce | `reduce(...)` (3 overloads), `count()`, `sum()`, `min()`, `max()`, `average()`, `summaryStatistics()` | no |
| Search | `findFirst()`, `findAny()`, `anyMatch()`, `allMatch()`, `noneMatch()` | **yes** |
| Iterate | `forEach(Consumer)`, `forEachOrdered(Consumer)` | no |
| Escape | `iterator()`, `spliterator()` | n/a |

`allMatch` and `noneMatch` short-circuit on the *first counterexample*. `anyMatch`
short-circuits on the first match. All three return on an empty stream without touching
anything: `allMatch` and `noneMatch` are `true`, `anyMatch` is `false`. That vacuous
truth is worth remembering — it bites in validation code.

### `toList()` versus `collect(Collectors.toList())`

```java
List<String> a = stream.toList();                        // Java 16+, UNMODIFIABLE, allows nulls
List<String> b = stream.collect(Collectors.toList());    // modifiable ArrayList (unspecified, but is)
List<String> c = stream.collect(Collectors.toUnmodifiableList());   // unmodifiable, REJECTS nulls
```

Use `toList()` by default on Java 21. The two gotchas:

- `toList()` returns an unmodifiable list. Calling `.add()` on it throws
  `UnsupportedOperationException` — a runtime failure in code that compiled fine.
- `toUnmodifiableList()` throws `NullPointerException` on a null element;
  `toList()` permits nulls. They are not interchangeable.

`[LEGACY — still asked]` `collect(Collectors.toList())` is the Java 8 idiom and is
still everywhere. Interviewers ask what the difference is. The answer is mutability
and null policy, not performance.

### Sources

```java
collection.stream()                          // the common case
collection.parallelStream()                  // Topic 25
Arrays.stream(array)                         // arrays; also Arrays.stream(a, from, to)
Stream.of("a", "b", "c")                     // varargs
Stream.empty()
Stream.ofNullable(maybeNull)                 // 0 or 1 elements (Java 9+)
Stream.iterate(1, i -> i * 2)                // INFINITE — needs limit()
Stream.iterate(1, i -> i < 100, i -> i * 2)  // bounded 3-arg form (Java 9+)  <- prefer this
Stream.generate(Math::random)                // INFINITE, unordered
IntStream.range(0, 10)                       // 0..9,  no boxing
IntStream.rangeClosed(1, 10)                 // 1..10, no boxing
Files.lines(path)                            // MUST be closed — see Trap 5
new Random().ints(100, 0, 50)                // primitive stream of randoms
"a,b,c".chars()                              // IntStream of code units, not chars
Pattern.compile(",").splitAsStream("a,b,c")  // lazy split
map.entrySet().stream()                      // the Map entry point
```

### Primitive streams — and why they exist

Topic 01 collecting again:

```java
// BOXES: every totalPence becomes a Long object, then unboxes to add, then reboxes.
long total = orders.stream()
        .map(Order::totalPence)          // Stream<Long>
        .reduce(0L, Long::sum);

// NO BOXING: mapToLong produces a LongStream of raw longs.
long total = orders.stream()
        .mapToLong(Order::totalPence)    // LongStream
        .sum();
```

Same answer. The first allocates one `Long` per order plus one per partial sum. In a
5-million-row report that is real. In a 20-item cart it is nothing.

The primitive streams are `IntStream`, `LongStream`, `DoubleStream`. They add
`sum()`, `average()`, `max()`, `min()`, `summaryStatistics()` — which the object stream
does not have, because "sum of arbitrary objects" is meaningless.

```java
LongSummaryStatistics stats = orders.stream()
        .mapToLong(Order::totalPence)
        .summaryStatistics();
stats.getCount(); stats.getSum(); stats.getMin(); stats.getMax(); stats.getAverage();
```

One pass, five answers. This is the single most useful primitive-stream method and
almost nobody knows it exists.

Getting back: `.boxed()` for `LongStream` → `Stream<Long>`, `.mapToObj(f)` for
`LongStream` → `Stream<R>`.

### `[JAVA 25]` Stream Gatherers

Java 25 has `Stream.gather(Gatherer)` — a user-definable *intermediate* operation, the
missing counterpart to `Collector`. It lets you write sliding windows, fixed-size
batches, and stateful custom transforms as first-class pipeline stages.

```java
// [JAVA 25]
List<List<Order>> batches = orders.stream()
        .gather(Gatherers.windowFixed(100))
        .toList();
```

**Java 21 fallback:** there is no equivalent intermediate operation. You either collect
first and batch the list, write a custom `Spliterator`, or use a library
(`Guava.Lists.partition` after collecting). Do not build a Java 21 codebase around
`gather`; do know it exists, because it will come up.

---

## Why does it matter?

**1. The laziness bug is silent.** A pipeline with no terminal operation compiles,
runs, produces no error, and does nothing. If the pipeline was doing the work — sending
emails, writing rows — that work simply does not happen. There is no exception to find
in the logs. This is the single most expensive misunderstanding in this phase.

**2. Short-circuiting is a real algorithmic difference.** `findFirst` after an
expensive `map` is O(k), not O(n). If you write the JavaScript-shaped equivalent by
collecting between steps, you lose it and never notice, because the results are
identical.

**3. Operation order is a cost decision the compiler will not make for you.**
`sorted().filter().limit(10)` versus `filter().sorted().limit(10)` can be orders of
magnitude apart on a large source. You need to see it in review.

**4. `IntStream` versus `Stream<Integer>` is your first place to apply Topic 01.**
Boxing in a report that walks a million rows is measurable. Knowing `mapToLong` exists
means you write the boxing-free version by default and never have to fix it later.

**5. Everything downstream assumes it.** Collectors (24), parallel streams (25),
`Optional` (26), and Reactor's `Flux` (103–108) all build on the lazy-pipeline model.
Reactor goes further — nothing runs until `subscribe()` — and the master plan flags
that as "the bug you will write". Understanding stream laziness now is what makes that
one land softly.

---

## Syntax breakdown

### The pipeline shape

```java
source.stream()                 // 1. produce a Stream<T>
      .intermediateOp(...)      // 2. returns a Stream — zero or more of these
      .intermediateOp(...)
      .terminalOp(...);         // 3. returns something that is NOT a Stream
```

**How to tell an operation's kind without looking it up:** check the return type.
Returns a `Stream`/`IntStream`? Intermediate, lazy, nothing has happened. Returns
anything else — a `List`, a `long`, an `Optional`, `void`? Terminal, and the whole
pipeline just ran.

That one rule replaces memorising the tables above.

### `flatMap` — the one that needs explaining

```java
// Stream<Order>, each with a List<OrderLine>.  You want a Stream<OrderLine>.
Stream<OrderLine> lines = orders.stream()
        .flatMap(order -> order.lines().stream());
```

`map` would give you `Stream<List<OrderLine>>` — a stream of lists. `flatMap` takes a
function that returns a **stream per element**, and splices them all into one flat
stream. Exactly `Array.prototype.flatMap`.

The two-argument-source form is worth knowing:

```java
// Cartesian product: every order paired with every warehouse
Stream<Assignment> pairs = orders.stream()
        .flatMap(order -> warehouses.stream()
                                    .map(w -> new Assignment(order, w)));
```

`mapMulti` does the same job without creating a `Stream` object per element, which
matters when the per-element result is usually 0 or 1:

```java
// Java 16+, on 21 already
Stream<Payment> settled = orders.<Payment>mapMulti((order, downstream) -> {
    Payment p = order.payment();
    if (p != null && p.isSettled()) {
        downstream.accept(p);
    }
});
```

Use `flatMap` by default; reach for `mapMulti` when profiling says the per-element
stream allocation is real.

### `peek` — and the warning attached to it

```java
orders.stream()
      .peek(o -> log.debug("before filter: {}", o.id()))
      .filter(Order::isPaid)
      .peek(o -> log.debug("after filter: {}", o.id()))
      .toList();
```

`peek` is for **debugging a pipeline while you develop it**. Its Javadoc says so. Two
reasons not to ship it:

1. The JDK is permitted to skip it. `count()` on a sized stream with no
   size-changing operations does not traverse at all — so your `peek` never runs. That
   is Trap 3 and it is genuinely surprising.
2. A `peek` with a side effect is shared mutable state waiting for someone to add
   `.parallel()` (Topic 25).

### `Optional` returns

`findFirst`, `findAny`, `min`, `max`, and the one-arg `reduce` all return `Optional<T>`
because the stream may be empty. That is Topic 26; for now:

```java
Optional<Order> largest = orders.stream().max(Comparator.comparingLong(Order::totalPence));
long value = largest.map(Order::totalPence).orElse(0L);   // not .get()
```

---

## Example 1 — minimal

Prove laziness and short-circuiting in one file, with print statements as the evidence.

```java
import java.util.*;

public class LazinessDemo {

    static boolean isPaid(String order) {
        System.out.println("  filter tested " + order);
        return order.endsWith("PAID");
    }

    static String upper(String order) {
        System.out.println("  map ran on " + order);
        return order.toUpperCase();
    }

    public static void main(String[] args) {
        List<String> orders = List.of("a-PAID", "b-PENDING", "c-PAID", "d-PAID");

        System.out.println("--- building the pipeline ---");
        var pipeline = orders.stream()
                             .filter(LazinessDemo::isPaid)
                             .map(LazinessDemo::upper);

        System.out.println("--- pipeline built, nothing above should have printed ---");

        System.out.println("--- terminal: findFirst ---");
        Optional<String> first = pipeline.findFirst();
        System.out.println("result = " + first.orElse("none"));
    }
}
```

Run it and read the ordering, not the values. Two things to notice:

- Nothing printed between "building" and "pipeline built". The `filter` and `map`
  lambdas were never invoked.
- After `findFirst`, you see `filter tested a-PAID` then `map ran on a-PAID` and then
  it **stops**. Elements b, c and d were never fetched from the source. And notice the
  interleaving — filter and map alternate per element, they do not run in phases. That
  interleaving *is* fusion.

---

## Example 2 — production scenario

`orderflow` has a support endpoint: "show me the first three disputed orders for this
customer in the last 90 days, with their payment status". Payment status lives in a
separate service and each lookup is a 40 ms HTTP call.

### The version written with a JavaScript-shaped mental model

```java
public class DisputeService {

    public List<DisputeView> firstThreeDisputes(CustomerId customerId) {

        List<Order> all = orderRepository.findByCustomer(customerId);   // ~4,000 orders

        List<Order> recent = all.stream()
                .filter(o -> o.placedAt().isAfter(Instant.now().minus(90, DAYS)))
                .collect(Collectors.toList());                          // materialise

        List<DisputeView> enriched = recent.stream()
                .map(o -> new DisputeView(o, paymentClient.statusOf(o.paymentId())))
                .collect(Collectors.toList());                          // materialise

        List<DisputeView> disputed = enriched.stream()
                .filter(v -> v.paymentStatus() == PaymentStatus.DISPUTED)
                .collect(Collectors.toList());                          // materialise

        return disputed.subList(0, Math.min(3, disputed.size()));
    }
}
```

This is correct. It is also, on a customer with 800 orders in the window,
**800 × 40 ms = 32 seconds** of HTTP calls, to return three rows.

The mistake is not the streams. It is that `collect` between steps threw away laziness
and turned the pipeline back into the eager, materialising model you brought from
JavaScript.

### The version that uses the pipeline

```java
public class DisputeService {

    private static final int PAGE = 3;

    public List<DisputeView> firstThreeDisputes(CustomerId customerId) {

        Instant cutoff = Instant.now().minus(90, ChronoUnit.DAYS);

        return orderRepository.findByCustomer(customerId).stream()
                .filter(o -> o.placedAt().isAfter(cutoff))          // cheap, do it first
                .filter(Order::mayBeDisputed)                       // cheap local flag
                .map(this::toDisputeView)                           // expensive: 40 ms each
                .filter(v -> v.paymentStatus() == PaymentStatus.DISPUTED)
                .limit(PAGE)                                        // stops the source
                .toList();
    }

    private DisputeView toDisputeView(Order order) {
        return new DisputeView(order, paymentClient.statusOf(order.paymentId()));
    }
}
```

What changed and what it bought:

| Change | Effect |
|---|---|
| No `collect` between stages | One pass. No intermediate lists. |
| `limit(3)` at the end | Short-circuits **backwards** — the source stops producing once three survive. |
| Cheap filters before the expensive `map` | The 40 ms call runs only for candidates. |
| `Order::mayBeDisputed` as a pre-filter | A local flag that eliminates most orders before any HTTP call. |

On a customer whose 7th recent order is the third dispute, `toDisputeView` runs **7
times**, not 800. Roughly 280 ms instead of 32 seconds. Same output, same tests.

### What this version still gets wrong

Be honest about it:

- **It hides N sequential HTTP calls behind a `map`.** The pipeline reads as if it were
  free. Someone will later add a `.sorted()` before the `limit` and destroy the
  short-circuiting without realising, because `sorted` must consume everything. Put a
  comment on the `map`.
- **A real fix is a bulk endpoint or a join**, not a cleverer pipeline. `statusOf` per
  order is an N+1 in a different costume — you will meet the database version in
  Topic 50.
- **Adding `.parallel()` here would be actively wrong.** It is I/O-bound, and Topic 25
  explains exactly what it would do to the rest of the JVM.
- **`Instant.now()` was hoisted into `cutoff`.** In the first version it was called
  inside the predicate — once per element. That is a correctness smell as well as a
  cost: the cutoff moved slightly during the scan.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a pipeline with no terminal operation

**Wrong:**
```java
public void notifyLateOrders(List<Order> orders) {
    orders.stream()
          .filter(Order::isLate)
          .map(this::buildNotification)
          .peek(notificationSender::send);       // <-- the defect: peek is intermediate
}
```

**Exact symptom:** the method returns successfully. No exception. No log line. No
notification is ever sent. Your integration test that asserts on the sender mock fails
with "wanted but not invoked" and you cannot see why, because the code plainly says
`send`.

Worse variant — the method compiles and there is not even a mock to fail:

```java
orders.stream().filter(Order::isLate).map(this::buildNotification);   // statement, discarded
```

Your IDE may warn "result of `map` is ignored". `javac` will not.

**Root cause:** `peek` and `map` are intermediate. They return a `Stream`. Nothing in a
stream runs until a **terminal** operation demands elements. There is no terminal
operation here, so the source is never traversed and neither lambda is ever called.

**Fix:**
```java
orders.stream()
      .filter(Order::isLate)
      .map(this::buildNotification)
      .forEach(notificationSender::send);       // terminal — this runs
```

**How to catch it before production:**
- The return-type rule: if the last call in the chain returns a `Stream`, nothing ran.
- Turn on ErrorProne's `ReturnValueIgnored` / `StreamResourceLeak`, or IntelliJ's
  "Result of method call ignored" inspection, and treat it as an error.
- In code review: **any `peek` performing an action, rather than logging, is a bug.**

---

### Trap 2 — reusing a consumed stream

**Wrong:**
```java
Stream<Order> paid = orders.stream().filter(Order::isPaid);

long count = paid.count();
List<Order> list = paid.toList();      // <-- second terminal operation
```

**Exact symptom:**
```
java.lang.IllegalStateException: stream has already been operated upon or closed
	at java.base/java.util.stream.AbstractPipeline.<init>(AbstractPipeline.java:203)
	at java.base/java.util.stream.ReferencePipeline$StatelessOp.<init>(...)
	at com.orderflow.ReportService.build(ReportService.java:57)
```

It throws on the **second** use, at runtime, on whatever input path happens to hit that
branch. If the first branch is the common one and the second is an error path, this
ships.

The subtler form, which people hit more often:

```java
private final Stream<Order> auditStream = loadOrders().stream();   // a FIELD
```
Now the second request to touch that bean throws. Streams are not beans, caches, or
fields.

**Root cause:** a stream is a one-shot pipeline over a source. The terminal operation
consumes it and marks the pipeline `linkedOrConsumed`. There is nothing to replay —
that is the price of not materialising intermediate results.

**Fix — re-create the stream, or materialise once:**
```java
// Option A: two streams from the same source
long count = orders.stream().filter(Order::isPaid).count();
List<Order> list = orders.stream().filter(Order::isPaid).toList();

// Option B: materialise once, derive both  (usually better — one pass)
List<Order> paid = orders.stream().filter(Order::isPaid).toList();
long count = paid.size();

// Option C: a Supplier<Stream<T>> when you genuinely need a re-runnable pipeline
Supplier<Stream<Order>> paidStreams = () -> orders.stream().filter(Order::isPaid);
long count = paidStreams.get().count();
List<Order> list = paidStreams.get().toList();
```

Option B is right far more often than people think. If you need two answers from one
source, one pass plus a list beats two passes.

> Note the contrast with your JS instinct: `const paid = orders.filter(isPaid)` gives
> you an array you can use forever. The Java equivalent of that array is Option B's
> `List`, not the `Stream`.

---

### Trap 3 — `count()` skips the pipeline and your `peek` never runs

**Wrong:**
```java
long processed = orders.stream()
        .peek(auditLog::record)      // <-- side effect
        .count();
```

**Exact symptom:** `processed` is correct. `auditLog` is empty. Nothing was recorded,
no exception was thrown, and the count is right — so a test on the count passes while
the audit trail silently does not exist.

**Root cause:** since Java 9, `count()` is allowed to compute its answer without
executing the pipeline, when it can determine the size from the source and no operation
changes the count. `orders` is a `List`, so its `Spliterator` reports an exact size;
`peek` does not change the count; therefore the JDK returns `orders.size()` and never
traverses anything.

This is spec-sanctioned behaviour, documented on `count()`. It exists precisely because
the JDK is allowed to assume `peek` is side-effect-free.

Add a `.filter(...)` before the `count()` and the optimisation no longer applies — the
size becomes unknown — and suddenly your `peek` runs again. **The behaviour changes
based on an unrelated line.** That is the worst kind of bug.

**Fix — never put an action in `peek`:**
```java
long processed = orders.stream()
        .map(order -> { auditLog.record(order); return order; })   // still ugly
        .count();

// Better: say what you mean.
orders.forEach(auditLog::record);
long processed = orders.size();
```

**The rule:** `peek` is for reading during development. Any observable effect belongs in
`forEach` or a `Collector`.

---

### Trap 4 — boxing an entire report

**Wrong:**
```java
long revenuePence = orders.stream()
        .map(Order::totalPence)          // Stream<Long> — boxes every value
        .reduce(0L, Long::sum);          // unbox, add, re-box, per element
```

**Exact symptom:** correct answer, and no error. On a 5-million-order nightly report,
the GC log (`-Xlog:gc`) shows a high young-collection rate; an allocation profile
(async-profiler in `alloc` mode, or JFR `ObjectAllocationSample`) names
`java.lang.Long` as a top allocator. Wall-clock time is meaningfully worse than the
loop it replaced, which is how the "streams are slow" folklore starts.

**Root cause:** `map` on an object stream must produce objects. `Order::totalPence`
returns a `long`, which gets boxed into a `Long` immediately, and every partial sum in
the reduce gets boxed too — running totals blow past the `Integer`/`Long` cache range
(Topic 01) after the first few elements.

**Fix:**
```java
long revenuePence = orders.stream()
        .mapToLong(Order::totalPence)    // LongStream — raw longs
        .sum();
```

**And while you are there**, if you need more than one statistic:
```java
LongSummaryStatistics stats = orders.stream()
        .mapToLong(Order::totalPence)
        .summaryStatistics();
// count, sum, min, max, average — one pass, no boxing
```

**Honesty check:** on a 20-item cart this is invisible and you should not care. The
reason to write `mapToLong` by default is not that boxing is always expensive; it is
that it costs you nothing to write the version that never becomes a problem.

---

### Trap 5 — an unclosed `Files.lines` stream

**Wrong:**
```java
public long countErrorLines(Path logFile) throws IOException {
    return Files.lines(logFile)
                .filter(line -> line.contains("ERROR"))
                .count();
}
```

**Exact symptom:** works fine in tests. In production, after some hours of a job that
scans thousands of files:
```
java.nio.file.FileSystemException: /var/log/orderflow/app-9912.log:
  Too many open files
```
or, on a Linux container, an `IOException` with `EMFILE`. `lsof -p <pid> | wc -l` grows
monotonically. Heap looks fine — this is a **native** resource leak, not a heap leak.

**Root cause:** `Files.lines` returns a stream backed by an open file handle. The stream
implements `AutoCloseable`. A terminal operation does **not** close it. Nothing closes
it until the object is garbage collected and a `Cleaner` runs — which may be much later,
or never under low heap pressure.

Most streams do not need closing. The ones that do are the I/O-backed ones:
`Files.lines`, `Files.list`, `Files.walk`, `Files.find`, `DirectoryStream`-backed
sources, and JDBC result-set streams from some drivers.

**Fix — try-with-resources (Topic 08):**
```java
public long countErrorLines(Path logFile) throws IOException {
    try (Stream<String> lines = Files.lines(logFile)) {
        return lines.filter(line -> line.contains("ERROR"))
                    .count();
    }
}
```

**How to catch it:** ErrorProne's `StreamResourceLeak` check flags exactly this.
Also, in review: **`Files.` followed by a stream method must be inside a
try-with-resources.** No exceptions.

**Related trap in the same family — the infinite stream:**
```java
Stream.iterate(1, i -> i + 1).filter(i -> i % 7 == 0).toList();   // never returns
```
**Symptom:** the thread hangs at 100% CPU; a thread dump (`jcmd <pid> Thread.print`)
shows it inside `ReferencePipeline$Head.forEach`. **Fix:** `limit(n)` before the
terminal, or use the three-argument `Stream.iterate(seed, hasNext, next)` from Java 9
which has a built-in termination condition.

---

## Hands-on proof

Commands you run. I have no JVM; nothing below is invented output.

### Setup

```bash
mkdir -p ~/java-lab/23 && cd ~/java-lab/23
java --version
```

### Proof 1 — nothing runs before the terminal operation

Use `LazinessDemo.java` from Example 1.

```bash
java LazinessDemo.java
```

**What to look for:** the relative position of the `filter tested` / `map ran on` lines
against the three `---` banners.

| What you see | What it means |
|---|---|
| No `filter`/`map` lines between "building" and "pipeline built" | Confirmed lazy. The intermediate operations stored your lambdas and returned. |
| `filter tested a-PAID`, `map ran on a-PAID`, then stop | Confirmed fused **and** short-circuiting. Elements b–d were never pulled. |
| All four elements tested | Check you used `findFirst()` and not `toList()`. `toList()` is not short-circuiting. |
| `filter` runs for all four, *then* `map` runs for all four | You are not looking at a stream. Check you did not `collect` in between. |

Now change `findFirst()` to `toList()` and re-run.

**What to look for:** filter and map still interleave per element (fusion holds), but
now every element is processed (no short-circuiting). Two properties, independently
observed.

### Proof 2 — reuse throws, and the message names the reason

`Reuse.java`:
```java
import java.util.*;
import java.util.stream.*;

public class Reuse {
    public static void main(String[] args) {
        List<String> skus = List.of("SKU-1", "SKU-2", "SKU-3");
        Stream<String> s = skus.stream().filter(x -> x.endsWith("1"));

        System.out.println("count = " + s.count());
        System.out.println("list  = " + s.toList());   // second terminal op
    }
}
```

```bash
java Reuse.java
```

**What to look for:** the exception type and message.

| What you see | What it means |
|---|---|
| `IllegalStateException: stream has already been operated upon or closed` | Expected. Memorise this string — it is the fingerprint of stream reuse. |
| The stack trace names `AbstractPipeline.<init>` | The check happens when the *next* stage is constructed, not when it runs. That is why the trace points at pipeline internals rather than your logic. |
| No exception, both lines print | You created two separate streams. Re-check that `s` is a variable used twice. |

Now try storing the stream in a static field and calling a method twice. Same
exception, different call path — worth doing once so you recognise it when a Spring
bean holds one.

### Proof 3 — `count()` really does skip the pipeline

`CountSkip.java`:
```java
import java.util.*;

public class CountSkip {
    public static void main(String[] args) {
        List<String> orders = List.of("a", "b", "c", "d");

        System.out.println("--- peek + count ---");
        long n1 = orders.stream()
                        .peek(o -> System.out.println("  peeked " + o))
                        .count();
        System.out.println("count = " + n1);

        System.out.println("--- peek + filter + count ---");
        long n2 = orders.stream()
                        .peek(o -> System.out.println("  peeked " + o))
                        .filter(o -> true)
                        .count();
        System.out.println("count = " + n2);
    }
}
```

```bash
java CountSkip.java
```

**What to look for:** whether `peeked` lines appear in each section.

| What you see | What it means |
|---|---|
| **No** `peeked` lines in section 1, four `peeked` lines in section 2 | The documented behaviour, observed. `count()` computed 4 from the list's exact size without traversing. Adding a `filter` made the size unknown and forced traversal. |
| `peeked` lines in both sections | Your JDK chose to traverse anyway. This is permitted — `count()` *may* skip, it is not obliged to. Record what you saw and treat the optimisation as possible, not guaranteed. That is the correct lesson either way. |
| No `peeked` lines in **either** section | Unexpected. Check that section 2 really has the `filter`. |

**How to read it:** whichever way it goes on your JDK, the takeaway is the same. You
cannot rely on `peek` running. Therefore `peek` cannot carry an action.

### Proof 4 — the boxing difference is visible in a GC log

`BoxingReport.java`:
```java
import java.util.*;
import java.util.stream.*;

public class BoxingReport {
    public static void main(String[] args) {
        int n = 20_000_000;
        List<Long> values = LongStream.range(0, n).boxed().collect(Collectors.toList());

        long start = System.nanoTime();
        long result;
        if (args[0].equals("boxed")) {
            result = values.stream().map(v -> v * 2).reduce(0L, Long::sum);
        } else {
            result = values.stream().mapToLong(v -> v * 2).sum();
        }
        System.out.printf("%s result=%d elapsedMs=%d%n",
                args[0], result, (System.nanoTime() - start) / 1_000_000);
    }
}
```

```bash
javac BoxingReport.java
java -Xmx3g -Xlog:gc:file=boxed.log:time,uptime BoxingReport boxed
java -Xmx3g -Xlog:gc:file=prim.log:time,uptime  BoxingReport prim
wc -l boxed.log prim.log
```

**What to look for:** the number of GC lines in each log, and the reported elapsed
times.

| What you see | What it means |
|---|---|
| Substantially more GC lines in `boxed.log` | The boxed pipeline allocated a `Long` per element and per partial sum. This is Trap 4, measured. |
| Similar GC counts | Possible — escape analysis (Topic 75) may have eliminated some allocations, or your heap is large enough that the young gen absorbed it. Report what you got; do not force the expected answer. |
| Elapsed times are close | Also possible and worth reporting honestly. Allocation is cheap in the young generation; the difference is often smaller than folklore claims. |

**How to read the elapsed number: do not trust it.** This is a naive `nanoTime` loop,
which Topic 77 will show you measures dead-code elimination and JIT warm-up state as
much as it measures your code. The **GC line count** is the trustworthy signal here,
because allocation count is not something the JIT hides. Keep both numbers; you will
revisit them.

### Proof 5 — the file-handle leak is real and countable

`LeakLines.java`:
```java
import java.io.*;
import java.nio.file.*;
import java.util.stream.*;

public class LeakLines {
    public static void main(String[] args) throws Exception {
        Path f = Files.writeString(Path.of("sample.log"), "ERROR one\nINFO two\nERROR three\n");

        for (int i = 0; i < 20_000; i++) {
            long n = Files.lines(f).filter(l -> l.contains("ERROR")).count();  // NOT closed
            if (i % 5_000 == 0) System.out.println(i + " -> " + n);
        }
        System.out.println("done — press enter to exit");
        System.in.read();
    }
}
```

```bash
javac LeakLines.java
java LeakLines &
PID=$!
# in another shell, while it runs / waits:
lsof -p $PID | grep sample.log | wc -l
ulimit -n
```

**What to look for:** the count of open handles to `sample.log`.

| What you see | What it means |
|---|---|
| A large and growing number of open handles | The leak, observed. Each unclosed `Files.lines` holds a descriptor until GC eventually runs a `Cleaner`. |
| A small number | GC ran and the cleaners fired. Lower `-Xmx` to reduce GC pressure, or raise the loop count. The leak is real but its visibility depends on GC timing — which is exactly why it is dangerous. |
| `Too many open files` before you can measure | Even better. That is the production symptom, reproduced. Note your `ulimit -n`. |

Now wrap it in try-with-resources and repeat.

**What to look for:** the handle count stays flat. **How to read it:** the difference
between "collected eventually" and "released deterministically" is the entire argument
for try-with-resources, and it is why `Stream` extends `AutoCloseable` at all.

---

## Practice exercises

### 1 — Easy: classify and predict

**Part A.** For each operation below, write down: intermediate or terminal; stateless or
stateful; short-circuiting or not.

```
filter  map  mapToLong  flatMap  peek  sorted  distinct  limit  skip
takeWhile  dropWhile  boxed  forEach  toList  count  reduce  anyMatch
allMatch  findFirst  min  max  summaryStatistics  iterator
```

**Part B.** Predict, before running, exactly what this prints and in what order:

```java
Stream.of("a", "bb", "ccc", "dddd")
      .peek(s -> System.out.println("peek1 " + s))
      .filter(s -> s.length() > 1)
      .peek(s -> System.out.println("peek2 " + s))
      .map(String::toUpperCase)
      .peek(s -> System.out.println("peek3 " + s))
      .limit(2)
      .forEach(s -> System.out.println("out   " + s));
```

Write your prediction down first. Then run it. If you were wrong, write one sentence
explaining which property you mis-modelled.

**Part C.** Change `limit(2)` to `sorted()` followed by `limit(2)` and predict again.
Explain the difference in terms of what `sorted` must do before it can emit anything.

### 2 — Medium: the audit (combines Topics 01, 08, 10, 12, 14, 21, 22)

The fragment below contains **six** defects from Topics 01–23. Find each, state the
exact observable symptom (an exception with its message, a wrong value, a silent
no-op, or a resource metric that grows), and rewrite the method correctly.

```java
public class NightlyReconciliation {

    private Stream<Payment> pendingPayments;

    public ReconResult run(Path settlementFile) throws IOException {

        this.pendingPayments = paymentRepository.findAll().stream()
                .filter(p -> p.status() == PaymentStatus.PENDING);

        long pendingCount = pendingPayments.count();

        double totalPending = pendingPayments
                .map(Payment::amountPence)
                .reduce(0L, Long::sum);

        Files.lines(settlementFile)
             .map(line -> line.split(","))
             .peek(parts -> settlementStore.save(parts[0], parts[1]))
             .count();

        List<Payment> sortedTop = paymentRepository.findAll().stream()
                .sorted(Comparator.comparing(Payment::amountPence))
                .filter(p -> p.status() == PaymentStatus.SETTLED)
                .limit(10)
                .collect(Collectors.toList());
        sortedTop.add(Payment.SENTINEL);

        return new ReconResult(pendingCount, totalPending, sortedTop);
    }
}
```

Hints, one per defect, in no particular order: what happens on the second terminal
operation; what `count()` is permitted to skip; what `Files.lines` holds open; what
`double` does to money; which order `sorted` and `filter` should be in and why; what
`collect(toList())` returns versus `toList()` and which one that `.add` would break on.

### 3 — Hard: production simulation on `orderflow`

Build the dispute endpoint from Example 2 and prove the short-circuiting is real.

**Part A.** Write a `FakePaymentClient` whose `statusOf` sleeps 40 ms and increments an
`AtomicInteger callCount`. Generate 4,000 synthetic orders for one customer, of which
800 fall inside a 90-day window, and seed disputes so that the third dispute is the
17th order in the window.

**Part B.** Implement both versions: the `collect`-between-stages one and the
single-pipeline one. Run each, and record `callCount` and wall-clock time.

**Part C.** Now break the fast version deliberately: insert `.sorted(Comparator
.comparing(Order::placedAt))` immediately before the `limit(3)`. Re-run, record
`callCount`. Explain in one sentence why the number changed, in terms of what `sorted`
must consume.

**Part D.** Fix it while keeping the sort requirement. There is more than one answer —
sort the source before mapping, sort only the survivors, or push the ordering into the
repository query. Pick one, implement it, and state the trade-off you accepted.

**Part E.** Add a `.peek(o -> auditLog.record(o.id()))` immediately before the
`limit(3)`. Now answer: is the audit trail correct? Is it *complete*? Is "we audited
every order we looked at" the same as "we audited every order the customer has"? This
is a design question, not a Java question, and it is the one that actually causes
incidents.

**Part F.** Replace `paymentClient.statusOf(...)` per order with a single bulk call
`statusOfAll(List<PaymentId>)`. What does that do to the pipeline shape, and what did
you lose? (You lost short-circuiting. Was that a good trade? Give the condition under
which your answer flips.)

---

## Interview questions

### Q1 — "How is a Java stream different from `Array.prototype.map`?"

**Mid-level answer:** "Streams are lazy — nothing happens until you call a terminal
operation like `collect`. Array methods run immediately."

**Senior answer:** "Three differences that matter. First, execution: array methods are
eager and each one materialises a new array, so a three-step chain does three passes
and allocates two intermediate arrays. A stream is lazy and *fused* — the intermediate
operations build a chain of sinks, and the terminal operation pushes each element
through the whole chain in one pass with no intermediate storage. Second,
short-circuiting: `findFirst` after an expensive `map` reaches back up the pipeline and
stops the source, so an expensive mapping runs k times instead of n. You cannot get
that from `map().filter()[0]` without hand-writing a loop. Third, a stream is
single-use — a second terminal operation throws `IllegalStateException: stream has
already been operated upon or closed` — because there is no materialised result to
replay. The closest JS equivalent is actually iterator helpers or generators, not array
methods."

**What separates them:** naming fusion and the sink chain rather than just "lazy", the
short-circuiting consequence with its cost implication, and knowing which JS feature
is the real analogue.

**Interviewer's follow-up:** "Does laziness ever cost you?" They want stateful
operations — `sorted` buffers everything, so a `sorted` in the middle of your pipeline
throws away the single-pass property.

---

### Q2 — "This code sends no notifications. Why?"

```java
orders.stream().filter(Order::isLate).peek(sender::send);
```

**Mid-level answer:** "You need to add `.collect(Collectors.toList())` at the end."

**Senior answer:** "There's no terminal operation, so the pipeline never executes and
neither lambda is ever invoked — `filter` and `peek` both return `Stream`, which is the
tell. Adding a terminal would make it run, but I wouldn't fix it that way, because
`peek` is the second bug. `peek` is documented as a debugging aid and the JDK is
allowed to skip it: `count()` on a sized source with no size-changing operations
returns the size without traversing at all, so even *with* a terminal operation this
could silently do nothing. Actions belong in `forEach`. So:
`orders.stream().filter(Order::isLate).forEach(sender::send)` — or honestly, if all I'm
doing is filter-and-act, a plain enhanced `for` loop is clearer and lets checked
exceptions propagate."

**What separates them:** diagnosing the missing terminal via the return type, spotting
that `peek` is a second independent defect, and being willing to say a loop might be
the better answer.

**Interviewer's follow-up:** "How would you stop this reaching production?" ErrorProne's
`ReturnValueIgnored`, an IDE inspection promoted to an error, or a review rule that
`peek` must not have effects.

---

### Q3 — "When would you use `IntStream` instead of `Stream<Integer>`?"

**Mid-level answer:** "When you're working with numbers — it avoids autoboxing."

**Senior answer:** "Whenever the values are primitives and I don't need them as
objects, which is most numeric pipelines. `map(Order::totalPence).reduce(0L,
Long::sum)` boxes each value and each partial sum, and partial sums exceed the wrapper
cache immediately, so it's one allocation per element plus one per accumulation.
`mapToLong(Order::totalPence).sum()` allocates nothing. The primitive streams also give
you `sum`, `average`, `min`, `max` and `summaryStatistics` that the object stream
doesn't have — `summaryStatistics` in particular gives five aggregates in one pass and
is badly underused. That said, I wouldn't rewrite a 20-element cart calculation for
this; I write `mapToLong` by default because it costs nothing, not because boxing is
always expensive. If someone claimed it mattered on a specific path I'd want an
allocation profile, not an argument."

**What separates them:** naming *both* allocation sites (values and partial sums),
volunteering `summaryStatistics`, and calibrating the claim rather than overselling it.

**Interviewer's follow-up:** "How do you get from an `IntStream` back to objects?"
`boxed()` or `mapToObj(...)`.

---

### Q4 — "What happens if you call two terminal operations on one stream?"

**Mid-level answer:** "It throws an exception because streams can only be used once."

**Senior answer:** "`IllegalStateException: stream has already been operated upon or
closed`, thrown when the second pipeline stage is constructed — which is why the stack
trace points into `AbstractPipeline.<init>` rather than at your logic. The reason is
structural: a stream doesn't hold results, it holds a description of work over a
source, so there's nothing to replay. That's the same property that gives you fusion
and single-pass execution. Practically it means three things: never store a `Stream` in
a field or return one from a long-lived API unless the caller owns closing it; if you
need two answers from one source, materialise once with `toList()` and derive both
rather than streaming twice; and if you genuinely need a re-runnable pipeline, pass a
`Supplier<Stream<T>>`, not a `Stream<T>`."

**What separates them:** connecting the restriction to the design property that buys
fusion, plus the three concrete API consequences — especially "don't return a `Stream`
from a bean".

**Interviewer's follow-up:** "When *is* it right to return a `Stream` from a method?"
When it is lazily backed by a resource the caller should close, and you say so in the
Javadoc — `Files.lines` is the JDK's own example.

---

### Q5 — "Where would you put `filter` relative to `sorted`, and why?"

**Mid-level answer:** "Filter first — you sort fewer elements."

**Senior answer:** "Filter first, and the reason is bigger than element count. `sorted`
is a *stateful, fully-blocking* intermediate operation: it has to consume the entire
upstream into a buffer before it can emit a single element. So a `sorted` in the middle
of a pipeline destroys two properties at once — the single-pass fusion, and any
short-circuiting downstream of it. If I have `filter().sorted().limit(10)` I sort only
the survivors; if I have `sorted().filter().limit(10)` I sort everything and the
`limit` can't stop the source, because the source was already drained by the sort.
On a large collection that's the difference between n log k and n log n plus a full
buffer allocation. And if the source is a database query, the honest answer is that
neither belongs in Java — push the `ORDER BY` and the `WHERE` into SQL and let the
index do it."

**What separates them:** identifying `sorted` as fully-blocking and naming *which*
properties it destroys, plus escalating to "this should be in the query" rather than
optimising the wrong layer.

**Interviewer's follow-up:** "Which other operations are stateful?" `distinct`,
`sorted`, `limit`, `skip`, `takeWhile`, `dropWhile` — and only `sorted` and `distinct`
are the memory-hungry ones.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Streams are lazy but not *reusable*, while your JS arrays are eager but reusable.
   Is there a design that gets both? What would it cost?

2. `count()` may skip the pipeline entirely. That optimisation is only sound because the
   spec says `peek` must be side-effect-free. Was building an optimisation on a
   documented-but-unenforceable contract a good decision? Argue both sides.

3. `sorted()` destroys single-pass fusion. Given that, why does the Stream API include
   it at all rather than forcing you to collect and sort a list?

4. A stream does not modify its source, yet you can write a lambda that does. What
   would it take for the language to actually prevent that, and why do you think it
   does not?

5. Reactor's `Mono` is also lazy, but nothing runs until `subscribe()` rather than until
   a terminal operation. Name the property streams have that `Mono` does not, and vice
   versa. (Topic 103 will check your answer.)

6. `Files.lines` returns a closeable stream. Most streams are not closeable. Would it
   have been better to make *every* stream require closing? What broke that idea?

7. You are writing a team guideline: "when to use a stream and when to use a `for`
   loop". Write it in three bullets. Then name the case where your guideline gives the
   wrong answer.

---

## Quick reference card

### Pipeline anatomy

```java
source.stream()          // Collection, Arrays, Stream.of, IntStream.range, Files.lines
      .intermediate()    // returns a Stream — LAZY, nothing runs
      .intermediate()
      .terminal();       // returns non-Stream — EVERYTHING runs here, once
```

**Kind test:** returns `Stream` → intermediate. Returns anything else → terminal.

### Intermediate operations

```
STATELESS  filter map mapToInt/Long/Double mapToObj flatMap mapMulti peek boxed
STATEFUL   sorted distinct limit skip takeWhile dropWhile
BLOCKING   sorted (buffers everything), distinct (grows with cardinality)
```

### Terminal operations

```
COLLECT       toList()  collect(...)  toArray()  toArray(Order[]::new)
REDUCE        reduce(...)  count()  sum()  min()  max()  average()  summaryStatistics()
SEARCH        findFirst()  findAny()  anyMatch()  allMatch()  noneMatch()      <- short-circuit
ITERATE       forEach()  forEachOrdered()
```

### Primitive streams — reach for these with numbers

```java
orders.stream().mapToLong(Order::totalPence).sum();
orders.stream().mapToLong(Order::totalPence).summaryStatistics();   // 5 aggregates, 1 pass
IntStream.range(0, n).mapToObj(i -> build(i)).toList();
longStream.boxed();          // back to Stream<Long>
```

### `toList` variants

| Call | Mutable? | Nulls? | Java |
|---|---|---|---|
| `stream.toList()` | no | allowed | 16+ — **default on 21** |
| `collect(Collectors.toList())` | yes (in practice) | allowed | 8 — `[LEGACY]` |
| `collect(Collectors.toUnmodifiableList())` | no | **throws NPE** | 10+ |

### Gotchas checklist

- [ ] No terminal operation → nothing runs, silently.
- [ ] Second terminal operation → `IllegalStateException: stream has already been operated upon or closed`.
- [ ] Never store a `Stream` in a field or return one without a closing contract.
- [ ] `peek` is debugging only; `count()` may skip the whole pipeline.
- [ ] `sorted()` buffers everything and kills downstream short-circuiting.
- [ ] Put cheap `filter`s before expensive `map`s; put `limit` as early as correctness allows.
- [ ] `Files.lines`/`list`/`walk`/`find` → try-with-resources, always.
- [ ] `Stream.iterate`/`generate` are infinite; use `limit` or the 3-arg `iterate`.
- [ ] `mapToLong` over `map` for numbers.
- [ ] `stream.toList()` is unmodifiable — `.add()` throws at runtime.
- [ ] Mutating the source collection during a stream → `ConcurrentModificationException`.
- [ ] `allMatch`/`noneMatch` are `true` on an empty stream.

---

## When would I use this at work?

**1. A report endpoint that got slow.**
Someone wrote `findAll().stream().map(expensiveEnrichment).filter(...).limit(20)`. You
reorder to filter first and the p99 drops by an order of magnitude, with no algorithm
change and no new dependency. This is the most common real win from this topic.

**2. Diagnosing "the job ran but nothing happened".**
An overnight batch reports success and writes zero rows. You look at the code, see the
chain ends in `.map(...)` with no terminal operation, and you have the answer in ten
seconds instead of adding logging and re-running a two-hour job.

**3. Reviewing a `Files.walk` in a scheduled task.**
You spot the missing try-with-resources and ask for it. Six months later the service
does not hit `Too many open files` at 3am, and nobody ever knows you prevented it.
This is the least glamorous and highest-value use of this topic.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and boxing**: why `IntStream`/`LongStream` and `mapToLong` exist.
- **08 — Exceptions and try-with-resources**: why `Files.lines` must be closed, and why
  checked exceptions do not fit inside `map`.
- **10 — Collections Framework**: `Collection.stream()` is the source you will use 95%
  of the time; `ConcurrentModificationException` if you mutate while streaming.
- **14 — Comparable/Comparator**: what `sorted()` takes, and why the comparator must be
  a total order.
- **21 — Lambdas**: every stream operation takes a functional interface.
- **22 — Method references**: `map(Order::sku)`, `filter(Objects::nonNull)`.

**This unlocks:**
- **24 — Collectors**: what `collect()` actually does, `groupingBy`, and custom
  collectors.
- **25 — Parallel streams**: the `Spliterator` that this topic deliberately skipped,
  and what splitting does to everything above.
- **26 — Optional**: what `findFirst`, `min` and `max` return, and how to use it without
  `get()`.
- **27 — Records**: the natural element type for stream pipelines.
- **77 — JMH**: why the `nanoTime` numbers in Proof 4 are not trustworthy.
- **91 — `CompletableFuture`**: the async analogue of pipeline composition.
- **103–108 — Reactor**: `Flux` is this model plus backpressure and asynchrony. The
  laziness lesson here transfers directly, and the "nothing runs without `subscribe()`"
  trap there is exactly Trap 1 in a different library.
- **50 — Hibernate N+1**: Example 2's per-order remote call is the same shape as an
  N+1, and the fix is the same shape too.

---

*Java baseline 21. The core Stream API is unchanged since Java 8; `takeWhile`,
`dropWhile`, `ofNullable` and the three-argument `iterate` arrived in 9, `toList()` and
`mapMulti` in 16 — all available on 21. `Stream.gather` and `Gatherers` are `[JAVA 25]`
and have no 21 equivalent.*
