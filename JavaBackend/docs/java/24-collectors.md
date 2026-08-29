# 24 — Streams II: Collectors, Grouping, Custom Collectors, Teeing

## Phase: 2 — Modern Java
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

At the end of the belt from Topic 23, someone has to catch the parts.

A **collector** is the instruction card you hand that person. It has exactly four
lines on it:

1. **"Fetch an empty box."** — the *supplier*.
2. **"When a part arrives, put it in the box like this."** — the *accumulator*.
3. **"If we opened two boxes, here is how to merge them into one."** — the *combiner*.
4. **"When the belt stops, do this to the box before handing it over."** — the
   *finisher*.

Everything in this topic is those four lines.

Two things follow immediately, and they are the whole reason this is a separate topic
from 23:

**Line 3 is usually never used.** With one worker there is only ever one box, so
nothing needs merging. Line 3 only runs when the work was split across several workers.
So if you write line 3 wrong, **the card works perfectly** — right up until someone
puts more workers on the belt. Then it produces wrong answers, quietly. That is
Topic 25, and it starts here.

**Line 2 mutates the box.** It does not build a new box each time. That is why this is
called a *mutable reduction* and why it is fast. Contrast with adding numbers, where
each step makes a brand-new value.

---

## The bridge from what you know

### The honest analogue

Modern JavaScript finally has one:

```ts
// JavaScript — Object.groupBy (and Map.groupBy)
const byStatus = Object.groupBy(orders, o => o.status);
// { PAID: [...], PENDING: [...] }
```

```java
// Java
Map<Status, List<Order>> byStatus = orders.stream()
        .collect(Collectors.groupingBy(Order::status));
```

Same job, same shape, same result. **Verdict: HONEST ANALOGUE** for the simple case.

Where Java goes further, and JS has nothing: the second argument.

```java
// "group by status, and for each group give me the count, not the list"
Map<Status, Long> countByStatus = orders.stream()
        .collect(Collectors.groupingBy(Order::status, Collectors.counting()));

// "group by customer, then by status, and sum the totals"
Map<CustomerId, Map<Status, Long>> report = orders.stream()
        .collect(Collectors.groupingBy(Order::customerId,
                 Collectors.groupingBy(Order::status,
                 Collectors.summingLong(Order::totalPence))));
```

That second argument is a **downstream collector**, and it composes arbitrarily deep.
In JavaScript you would write `Object.groupBy` and then map over the entries, building
intermediate arrays at each level. Java builds the final shape in one pass.

**Verdict: PARTIAL ANALOGUE** — the entry point matches, the composition has no JS
equivalent.

### The break — `reduce` versus `collect`

You have `Array.prototype.reduce`, and you probably use it for everything, including
building objects:

```ts
const byId = orders.reduce((acc, o) => { acc[o.id] = o; return acc; }, {});
```

That code mutates `acc` inside a `reduce`. In JavaScript it is idiomatic and fine.

In Java that pattern is **wrong**, and the language gives you a different method for
it:

| | `reduce` | `collect` |
|---|---|---|
| Accumulator | must be **immutable** — returns a new value | is **mutable** — is modified in place |
| Per element | allocates a result | no allocation |
| Correct for | sums, min, max, string concat via `BinaryOperator` | lists, maps, sets, `StringBuilder`, anything container-shaped |
| Under `.parallel()` | safe if associative | safe if the combiner is correct |

Using `reduce` with a mutable accumulator in Java compiles, works sequentially, and
breaks under parallelism. It is Trap 5 below.

### The summary table

| You know | Java | Verdict |
|---|---|---|
| `Object.groupBy` / `Map.groupBy` | `Collectors.groupingBy` | **HONEST ANALOGUE** |
| Nested `groupBy` + `map` over entries | Downstream collectors | **PARTIAL** — Java composes in one pass |
| `array.reduce((acc, x) => {...mutate...}, {})` | `collect(...)`, **not** `reduce(...)` | **NO ANALOGUE** — Java splits the two concepts |
| `array.join(", ")` | `Collectors.joining(", ")` | **HONEST ANALOGUE** |
| `new Map(orders.map(o => [o.id, o]))` — last duplicate wins | `Collectors.toMap` — duplicate **throws** | **PARTIAL** — the failure mode is the opposite |
| `Object.fromEntries` accepts anything | `toMap` rejects null values | **PARTIAL** |

That fifth row is worth a moment. `new Map([...])` in JavaScript silently keeps the
last duplicate. `Collectors.toMap` throws `IllegalStateException`. Java chose "tell me"
where JavaScript chose "guess". You will meet that difference in production.

---

## What is this?

`collect` is the general-purpose terminal operation. Everything else — `toList`,
`count`, `sum` — is a special case of it.

```java
<R, A> R collect(Collector<? super T, A, R> collector);
```

Three type parameters, and understanding them makes the rest trivial:

| Type | Meaning | Example for `groupingBy(Order::status)` |
|---|---|---|
| `T` | element type flowing down the pipeline | `Order` |
| `A` | **intermediate accumulation type** — the mutable box | `HashMap<Status, List<Order>>` |
| `R` | final result type handed back to you | `Map<Status, List<Order>>` |

`A` is usually invisible. It exists so that the container you build up can be a
different type from the thing you return — which is what makes `collectingAndThen` and
immutable results possible.

### The four functions, precisely

```java
public interface Collector<T, A, R> {
    Supplier<A>          supplier();      // 1. make a new empty container
    BiConsumer<A, T>     accumulator();   // 2. fold one element into a container
    BinaryOperator<A>    combiner();      // 3. merge two containers into one
    Function<A, R>       finisher();      // 4. convert container -> result
    Set<Characteristics> characteristics();
}
```

Walk `Collectors.toList()` through it:

| Function | For `toList()` |
|---|---|
| `supplier` | `ArrayList::new` |
| `accumulator` | `List::add` |
| `combiner` | `(left, right) -> { left.addAll(right); return left; }` |
| `finisher` | identity — the `ArrayList` *is* the result |
| `characteristics` | `IDENTITY_FINISH` |

### When each function actually runs

This table is the single most important thing in the topic.

| Function | Sequential stream | Parallel stream |
|---|---|---|
| `supplier` | once | **once per split** — as many times as the work divided |
| `accumulator` | once per element | once per element, spread across threads |
| `combiner` | **never** | once per merge — `splits − 1` times |
| `finisher` | once | once, at the very end |

Read row three again. **The combiner is never called on a sequential stream.**

So a combiner that returns the wrong thing, drops elements, or is not associative is
completely invisible in every test you write, every local run, and every code review.
It appears the first time someone adds `.parallel()`, or the first time a library
internally parallelises for you. The bug does not look like a combiner bug — it looks
like "our revenue report is sometimes short by a few thousand pounds".

That is why the master plan describes it as "a bug that only appears under load". It is
the reason Topic 25 exists.

### Characteristics

```java
Collector.Characteristics.IDENTITY_FINISH   // finisher is identity; skip it
Collector.Characteristics.UNORDERED         // result does not depend on encounter order
Collector.Characteristics.CONCURRENT        // one shared container is safe across threads
```

- `IDENTITY_FINISH` lets the JDK skip a cast-and-call. A pure optimisation hint, but
  declaring it when your finisher is *not* identity is a `ClassCastException` waiting to
  happen.
- `UNORDERED` tells the pipeline it may drop ordering constraints. `toSet()` declares
  it; `toList()` does not.
- `CONCURRENT` means "don't split the container — give every thread the same one, I've
  made it thread-safe". Only meaningful with `UNORDERED` as well, and only used by
  `groupingByConcurrent` / `toConcurrentMap`. If you declare it and your container is
  not thread-safe, you get silent data corruption. Do not declare it.

---

## Why does it matter?

**1. `groupingBy` with a downstream collector replaces most of the loop code still
being written.** A "revenue by customer by month" report is three nested loops and two
`computeIfAbsent` calls in imperative Java, or four lines with collectors. Once you can
read nested downstream collectors, a large class of reporting code becomes trivial.

**2. `toMap` fails in two ways that `HashMap.put` does not.** Duplicate keys throw, and
null values throw. Both are runtime failures on data-dependent paths, so both ship.
Knowing the exact exception text turns a 40-minute investigation into a 30-second one.

**3. The combiner is the doorway to Topic 25.** You cannot reason about parallel streams
without knowing what a combiner is and when it runs. Everything about parallel
correctness reduces to "is your accumulate-then-combine actually associative".

**4. Writing a custom collector once makes the abstraction concrete.** After you have
implemented `supplier`/`accumulator`/`combiner`/`finisher` yourself, `groupingBy`'s
signature stops being intimidating and becomes obvious.

---

## Syntax breakdown

### The built-in collectors, organised by what you are trying to do

**Into a container:**
```java
Collectors.toList()                          // ArrayList, mutable  [LEGACY — prefer stream.toList()]
Collectors.toUnmodifiableList()              // immutable, rejects nulls
Collectors.toSet()                           // HashSet, UNORDERED
Collectors.toUnmodifiableSet()
Collectors.toCollection(TreeSet::new)        // YOU choose the implementation
Collectors.toCollection(LinkedHashSet::new)  // insertion order preserved
```

**Into a map:**
```java
Collectors.toMap(Order::id, Function.identity())
Collectors.toMap(Order::id, Function.identity(), (a, b) -> b)              // merge duplicates
Collectors.toMap(Order::id, Function.identity(), (a, b) -> b, TreeMap::new) // choose the map
Collectors.toUnmodifiableMap(k, v)
Collectors.toConcurrentMap(k, v)             // CONCURRENT — Topic 25
```

**Counting and arithmetic:**
```java
Collectors.counting()                        // Long
Collectors.summingInt(Order::lineCount)      // Integer
Collectors.summingLong(Order::totalPence)    // Long
Collectors.summingDouble(...)                // Double — not for money
Collectors.averagingLong(Order::totalPence)  // Double
Collectors.summarizingLong(Order::totalPence)// LongSummaryStatistics: count/sum/min/max/avg
Collectors.minBy(comparator)                 // Optional<T>
Collectors.maxBy(comparator)                 // Optional<T>
Collectors.reducing(0L, Order::totalPence, Long::sum)
```

**Strings:**
```java
Collectors.joining()
Collectors.joining(", ")
Collectors.joining(", ", "[", "]")           // delimiter, prefix, suffix
```

**Grouping and partitioning:**
```java
Collectors.groupingBy(Order::status)
Collectors.groupingBy(Order::status, Collectors.counting())
Collectors.groupingBy(Order::status, TreeMap::new, Collectors.toList())
Collectors.partitioningBy(Order::isPaid)                       // Map<Boolean, List<Order>>
Collectors.partitioningBy(Order::isPaid, Collectors.counting())// Map<Boolean, Long>
Collectors.groupingByConcurrent(Order::status)                 // Topic 25
```

`partitioningBy` is `groupingBy` with a `Predicate`. It always returns a map with
**both** `true` and `false` keys, even when one side is empty — which `groupingBy` does
not. That guarantee is the reason to use it.

**Adapting a downstream collector:**
```java
Collectors.mapping(Order::sku, Collectors.toList())        // transform, then collect
Collectors.flatMapping(o -> o.lines().stream(), toList())  // flatten, then collect (Java 9+)
Collectors.filtering(Order::isPaid, Collectors.toList())   // filter, then collect (Java 9+)
Collectors.collectingAndThen(toList(), List::copyOf)       // collect, then transform the result
Collectors.teeing(c1, c2, merger)                          // two collectors at once (Java 12+)
```

`filtering` versus a `filter` in the pipeline is a real distinction:

```java
// filter in the pipeline: statuses with NO paid orders vanish from the map entirely
orders.stream().filter(Order::isPaid)
      .collect(groupingBy(Order::status, counting()));

// filtering as a downstream: every status key survives, with a count of 0
orders.stream()
      .collect(groupingBy(Order::status, filtering(Order::isPaid, counting())));
```

Which you want depends on whether an empty group should appear in the report. That is a
product decision, and knowing both exist is what lets you make it.

### `teeing` — two answers, one pass

```java
// Java 12+, available on 21
record Summary(long count, long revenuePence) {}

Summary summary = orders.stream().collect(
        Collectors.teeing(
                Collectors.counting(),                          // collector 1
                Collectors.summingLong(Order::totalPence),      // collector 2
                (count, revenue) -> new Summary(count, revenue) // merge the two results
        ));
```

`teeing` feeds every element to **both** collectors and then combines the two results.
One pass over the source, two independent aggregations. Without it you would either
stream twice (two passes) or write a custom collector.

Nesting works, so you can tee a tee:

```java
record Report(long count, long revenue, Optional<Order> largest) {}

Report report = orders.stream().collect(
        Collectors.teeing(
                Collectors.teeing(
                        Collectors.counting(),
                        Collectors.summingLong(Order::totalPence),
                        Summary::new),
                Collectors.maxBy(Comparator.comparingLong(Order::totalPence)),
                (s, largest) -> new Report(s.count(), s.revenuePence(), largest)
        ));
```

Three aggregations, one pass. It gets unreadable fast — two levels is usually the
practical limit before a custom collector is clearer.

### Writing a custom collector

Two ways. `Collector.of(...)` is the one you will use:

```java
Collector<Order, ?, String> skuManifest = Collector.of(
        StringBuilder::new,                               // supplier
        (sb, order) -> sb.append(order.sku()).append('\n'), // accumulator
        (left, right) -> left.append(right),               // combiner
        StringBuilder::toString                            // finisher
);
```

Order of arguments: supplier, accumulator, combiner, then optionally a finisher, then
optionally characteristics. If you omit the finisher, `IDENTITY_FINISH` is implied and
`A` must equal `R`.

Implementing the interface directly is only worth it when you want a reusable named
type:

```java
public final class InventoryTally implements Collector<OrderLine, Map<Sku, Integer>, InventoryDelta> {

    @Override public Supplier<Map<Sku, Integer>> supplier() {
        return HashMap::new;
    }

    @Override public BiConsumer<Map<Sku, Integer>, OrderLine> accumulator() {
        return (map, line) -> map.merge(line.sku(), line.quantity(), Integer::sum);
    }

    @Override public BinaryOperator<Map<Sku, Integer>> combiner() {
        return (left, right) -> {
            right.forEach((sku, qty) -> left.merge(sku, qty, Integer::sum));
            return left;                                  // MUST return the merged container
        };
    }

    @Override public Function<Map<Sku, Integer>, InventoryDelta> finisher() {
        return InventoryDelta::new;
    }

    @Override public Set<Characteristics> characteristics() {
        return Set.of(Characteristics.UNORDERED);         // NOT IDENTITY_FINISH — we have a finisher
    }
}
```

Three rules for a correct combiner, and every one of them is a real bug people ship:

1. **It must return the merged container**, not `void`, and not one of the inputs
   unchanged.
2. **It must be associative**: `combine(combine(a,b),c)` must equal
   `combine(a,combine(b,c))`. Sums and set-unions are. "Take the first" and subtraction
   are not.
3. **It must not assume which side is bigger or came first.** The framework decides
   the split shape, and it changes with input size and core count.

---

## Example 1 — minimal

Every collector concept in one runnable file, small enough to hold in your head.

```java
import java.util.*;
import java.util.stream.*;
import static java.util.stream.Collectors.*;

public class CollectorBasics {

    record Order(String id, String status, long totalPence) {}

    public static void main(String[] args) {
        List<Order> orders = List.of(
                new Order("O-1", "PAID",    4995),
                new Order("O-2", "PENDING", 1250),
                new Order("O-3", "PAID",   12000),
                new Order("O-4", "PAID",     799));

        // 1. group
        Map<String, List<Order>> byStatus = orders.stream()
                .collect(groupingBy(Order::status));
        System.out.println("byStatus     = " + byStatus.keySet());

        // 2. group + downstream count
        Map<String, Long> countByStatus = orders.stream()
                .collect(groupingBy(Order::status, counting()));
        System.out.println("countByStatus= " + countByStatus);

        // 3. group + downstream sum
        Map<String, Long> revenueByStatus = orders.stream()
                .collect(groupingBy(Order::status, summingLong(Order::totalPence)));
        System.out.println("revenue      = " + revenueByStatus);

        // 4. group + mapping to a different element type
        Map<String, List<String>> idsByStatus = orders.stream()
                .collect(groupingBy(Order::status, mapping(Order::id, toList())));
        System.out.println("ids          = " + idsByStatus);

        // 5. partition
        Map<Boolean, Long> paidOrNot = orders.stream()
                .collect(partitioningBy(o -> o.status().equals("PAID"), counting()));
        System.out.println("partition    = " + paidOrNot);

        // 6. teeing — two aggregates, one pass
        String summary = orders.stream().collect(
                teeing(counting(),
                       summingLong(Order::totalPence),
                       (n, sum) -> n + " orders, " + sum + "p"));
        System.out.println("teeing       = " + summary);

        // 7. a custom collector
        Collector<Order, ?, String> manifest = Collector.of(
                StringBuilder::new,
                (sb, o) -> sb.append(o.id()).append(';'),
                StringBuilder::append,
                StringBuilder::toString);
        System.out.println("custom       = " + orders.stream().collect(manifest));
    }
}
```

Note what stayed constant across all seven: **one pass over `orders` each time**, and
the shape of the result was decided entirely by the collector, not by the pipeline.

---

## Example 2 — production scenario

`orderflow` needs a finance report: for each customer, for each month, the order count,
the gross revenue, the largest single order, and the set of distinct SKUs bought. It
must run over ~2 million orders in a nightly job and produce a stable, ordered output
for a CSV.

### The version people write first

```java
public class RevenueReportService {

    public Map<CustomerId, Map<YearMonth, MonthlyFigures>> build(List<Order> orders) {

        Map<CustomerId, Map<YearMonth, List<Order>>> grouped = new HashMap<>();

        for (Order order : orders) {
            YearMonth month = YearMonth.from(order.placedAt().atZone(UTC));
            grouped.computeIfAbsent(order.customerId(), k -> new HashMap<>())
                   .computeIfAbsent(month, k -> new ArrayList<>())
                   .add(order);
        }

        Map<CustomerId, Map<YearMonth, MonthlyFigures>> result = new HashMap<>();
        for (var customerEntry : grouped.entrySet()) {
            Map<YearMonth, MonthlyFigures> perMonth = new HashMap<>();
            for (var monthEntry : customerEntry.getValue().entrySet()) {
                List<Order> group = monthEntry.getValue();
                long count = group.size();
                long revenue = 0;
                Order largest = null;
                Set<Sku> skus = new HashSet<>();
                for (Order o : group) {
                    revenue += o.totalPence();
                    if (largest == null || o.totalPence() > largest.totalPence()) largest = o;
                    o.lines().forEach(l -> skus.add(l.sku()));
                }
                perMonth.put(monthEntry.getKey(),
                        new MonthlyFigures(count, revenue, largest, skus));
            }
            result.put(customerEntry.getKey(), perMonth);
        }
        return result;
    }
}
```

It works. It is also 30 lines, holds every order in memory grouped into lists before
computing anything, and has three places where a null slips through unnoticed.

### The collector version

```java
public class RevenueReportService {

    private static final ZoneId UTC = ZoneOffset.UTC;

    public SortedMap<CustomerId, SortedMap<YearMonth, MonthlyFigures>> build(Stream<Order> orders) {

        return orders.collect(
            Collectors.groupingBy(
                Order::customerId,
                TreeMap::new,                                    // stable, sorted output
                Collectors.groupingBy(
                    order -> YearMonth.from(order.placedAt().atZone(UTC)),
                    TreeMap::new,
                    monthlyFigures())));
    }

    /** All four aggregates for one month, computed in a single pass over that group. */
    private static Collector<Order, ?, MonthlyFigures> monthlyFigures() {
        return Collectors.teeing(
                Collectors.teeing(
                        Collectors.counting(),
                        Collectors.summingLong(Order::totalPence),
                        CountAndRevenue::new),
                Collectors.teeing(
                        Collectors.maxBy(Comparator.comparingLong(Order::totalPence)),
                        Collectors.flatMapping(
                                order -> order.lines().stream().map(OrderLine::sku),
                                Collectors.toCollection(TreeSet::new)),
                        LargestAndSkus::new),
                (cr, ls) -> new MonthlyFigures(
                        cr.count(), cr.revenuePence(), ls.largest().orElse(null), ls.skus()));
    }

    private record CountAndRevenue(long count, long revenuePence) {}
    private record LargestAndSkus(Optional<Order> largest, SortedSet<Sku> skus) {}
}
```

What each decision bought:

| Decision | Why |
|---|---|
| `Stream<Order>` parameter, not `List<Order>` | The caller can stream from a JDBC cursor or a paged repository. Nothing is ever fully in memory. |
| `TreeMap::new` as the map factory | `groupingBy` returns a `HashMap` by default, whose iteration order is unspecified and changes between runs. A CSV needs stable order. |
| Nested `teeing` | All four aggregates in one pass over each group. No intermediate `List<Order>` per group at all. |
| `flatMapping(...)` for SKUs | Goes from `Order` to its lines' SKUs inside the downstream, without materialising anything. Java 9+. |
| `toCollection(TreeSet::new)` | The SKU set is sorted, so the CSV column is deterministic. |
| `maxBy` returning `Optional` | Honest about the empty case. A group is never empty here, but the type says what the collector guarantees, not what your data happens to do. |

### Be honest about what this costs

Two levels of `teeing` is at the edge of readable. A reviewer who does not know
`teeing` will stall on `monthlyFigures()`. Three defensible positions:

1. **Ship it with the helper method named and a two-line comment.** The extraction into
   `monthlyFigures()` is what makes it survivable.
2. **Write a named custom `Collector<Order, ?, MonthlyFigures>`** implementing the
   interface. More code, far more readable at the call site, and testable in isolation.
   This is the right answer if the report has more than four aggregates.
3. **Do it in SQL.** Two `GROUP BY` columns and four aggregate functions is exactly what
   a database is for, and it will beat any JVM version on 2 million rows. This is the
   answer a senior engineer gives first, before showing they *can* do it in Java.

Position 3 is not a cop-out. The reason to know this material is that sometimes the
data is not in one database, and then you need it.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `Collectors.toMap` and duplicate keys

**Wrong:**
```java
Map<Sku, Product> catalogue = products.stream()
        .collect(Collectors.toMap(Product::sku, Function.identity()));
```

**Exact symptom:**
```
java.lang.IllegalStateException: Duplicate key SKU-4471
  (attempted merging values Product[sku=SKU-4471, price=1999]
                        and Product[sku=SKU-4471, price=2199])
	at java.base/java.util.stream.Collectors.duplicateKeyException(Collectors.java:135)
	at java.base/java.util.stream.Collectors.lambda$uniqKeysMapAccumulator$1(...)
	at com.orderflow.CatalogueLoader.load(CatalogueLoader.java:44)
```

It works in every test, because your fixtures have unique SKUs. It fails the first time
production data has a duplicate — often after a bad import, on startup, taking the
whole service down.

**Root cause:** the two-argument `toMap` uses a merge function that throws. That is
deliberate: `HashMap.put` silently overwrites, and the API designers decided silent data
loss was worse than a loud failure. The JavaScript `new Map([...])` behaviour — last one
wins — is exactly what they refused to default to.

**Fix — choose what a duplicate means, explicitly:**
```java
// Last wins (the JS default, now stated out loud):
.collect(Collectors.toMap(Product::sku, Function.identity(), (existing, replacement) -> replacement))

// First wins:
.collect(Collectors.toMap(Product::sku, Function.identity(), (existing, replacement) -> existing))

// Merge meaningfully:
.collect(Collectors.toMap(Product::sku, Function.identity(),
         (a, b) -> a.updatedAt().isAfter(b.updatedAt()) ? a : b))

// Duplicates are a data error and I want to know: keep the default, and let it throw.
// But add context, because "SKU-4471" alone is not enough at 3am:
.collect(Collectors.groupingBy(Product::sku))   // then assert every list has size 1
```

**The related trap:** `toMap` uses `equals`/`hashCode` on the key (Topic 13). A key
class with a broken `hashCode` produces *duplicate-looking* entries that never collide,
so the map silently grows past the number of logical keys and lookups miss. That one
does not throw at all.

---

### Trap 2 — `Collectors.toMap` throws on a null **value**

**Wrong:**
```java
Map<OrderId, PaymentId> paymentByOrder = orders.stream()
        .collect(Collectors.toMap(Order::id, Order::paymentId));   // paymentId may be null
```

**Exact symptom:**
```
java.lang.NullPointerException
	at java.base/java.util.HashMap.merge(HashMap.java:1367)
	at java.base/java.util.stream.Collectors.lambda$uniqKeysMapAccumulator$1(Collectors.java:180)
	at com.orderflow.PaymentIndex.build(PaymentIndex.java:29)
```

The NPE has **no message naming your code**, and the top frames are all JDK internals.
Nothing in the trace says "your value was null". Engineers stare at `Order::id` looking
for the null and it is not there.

**Root cause:** `toMap` is implemented with `HashMap.merge`, and `merge` is specified to
throw `NullPointerException` if the value is null — because for `merge`, a null value
means "remove the mapping", which would be indistinguishable from "never added it".

This is a genuine behavioural difference from the map you would have built by hand:

```java
Map<OrderId, PaymentId> m = new HashMap<>();
for (Order o : orders) m.put(o.id(), o.paymentId());   // null values: perfectly fine
```

`HashMap.put(k, null)` is legal and stores a null. `HashMap.merge(k, null, f)` throws.
`toMap` uses the second. That is the whole trap.

**Fix — decide what a missing payment means:**
```java
// A. Exclude them
Map<OrderId, PaymentId> m = orders.stream()
        .filter(o -> o.paymentId() != null)
        .collect(Collectors.toMap(Order::id, Order::paymentId));

// B. Model absence honestly (Topic 26)
Map<OrderId, Optional<PaymentId>> m = orders.stream()
        .collect(Collectors.toMap(Order::id, o -> Optional.ofNullable(o.paymentId())));

// C. Substitute a sentinel, if the domain has one
.collect(Collectors.toMap(Order::id, o -> Objects.requireNonNullElse(o.paymentId(), PaymentId.NONE)))

// D. Build the map by hand when null values are genuinely meaningful
Map<OrderId, PaymentId> m = new HashMap<>();
orders.forEach(o -> m.put(o.id(), o.paymentId()));
```

**How to recognise it instantly:** an NPE whose stack trace goes through
`HashMap.merge` and `Collectors.lambda$uniqKeysMapAccumulator` means a null **value**
in a `toMap`. Not a null key, not a null element.

> Note: `groupingBy` has the mirror-image problem with **keys**. A null classifier
> result throws `NullPointerException` too, because the accumulator calls
> `Objects.requireNonNull` on the key. So `groupingBy(Order::couponCode)` blows up the
> moment an order has no coupon. Same diagnosis, different function.

---

### Trap 3 — a broken combiner that only fails in parallel

**Wrong:**
```java
Collector<Order, ?, List<String>> skus = Collector.of(
        ArrayList::new,
        (list, order) -> list.add(order.sku()),
        (left, right) -> left);            // <-- the defect: right is discarded
```

**Exact symptom:** every unit test passes. Every local run is correct. In production,
under load, the report is *sometimes* short — and the amount it is short by varies
between runs on identical input. There is no exception, no log line, and no way to
reproduce it on a developer laptop with a small dataset.

If a library ever calls `.parallel()` on your behalf, or someone adds it to speed up a
slow report, the bug arrives with no code change in your file.

**Root cause:** the combiner is only invoked when the stream was split. Sequentially it
is dead code, so nothing tests it. `(left, right) -> left` throws away every element
accumulated in the right-hand container.

**Fix:**
```java
Collector<Order, ?, List<String>> skus = Collector.of(
        ArrayList::new,
        (list, order) -> list.add(order.sku()),
        (left, right) -> { left.addAll(right); return left; });   // merge, then return
```

**How to actually test a combiner — this is the part people skip:**

```java
@Test
void combinerMergesBothSides() {
    Collector<Order, List<String>, List<String>> c = skusCollector();

    List<String> left  = c.supplier().get();
    List<String> right = c.supplier().get();
    c.accumulator().accept(left,  order("SKU-1"));
    c.accumulator().accept(right, order("SKU-2"));

    List<String> merged = c.combiner().apply(left, right);
    assertThat(merged).containsExactlyInAnyOrder("SKU-1", "SKU-2");
}

@Test
void parallelAndSequentialAgree() {
    List<Order> input = manyOrders(10_000);
    assertThat(input.parallelStream().collect(skusCollector()))
        .containsExactlyInAnyOrderElementsOf(input.stream().collect(skusCollector()));
}
```

The second test is the one that matters and the one nobody writes. **Any custom
collector must have a parallel-versus-sequential equivalence test.** Make it a review
rule.

---

### Trap 4 — `groupingBy` gives you a `HashMap`, and you assumed an order

**Wrong:**
```java
Map<YearMonth, Long> revenueByMonth = orders.stream()
        .collect(Collectors.groupingBy(Order::month, Collectors.summingLong(Order::totalPence)));

revenueByMonth.forEach((month, revenue) -> csv.writeRow(month, revenue));
```

**Exact symptom:** the CSV rows come out in an arbitrary order. Worse, the order is
*stable enough* to look intentional during development, then changes when the data set
grows past a `HashMap` resize threshold (Topic 12), or when a key's hash distribution
shifts. A downstream diff-based reconciliation job starts reporting thousands of false
changes.

**Root cause:** `groupingBy`'s two-argument form specifies only "some `Map`". The
implementation returns a `HashMap`, whose iteration order is a function of hash codes
and table size — not of insertion, and not of the key's natural order.

**Fix — name the map you want:**
```java
// Sorted by key
Map<YearMonth, Long> byMonth = orders.stream()
        .collect(Collectors.groupingBy(Order::month, TreeMap::new,
                 Collectors.summingLong(Order::totalPence)));

// Encounter order preserved
Map<YearMonth, Long> byMonth = orders.stream()
        .collect(Collectors.groupingBy(Order::month, LinkedHashMap::new,
                 Collectors.summingLong(Order::totalPence)));

// Enum keys — always this (Topic 16)
Map<Status, Long> byStatus = orders.stream()
        .collect(Collectors.groupingBy(Order::status,
                 () -> new EnumMap<>(Status.class),
                 Collectors.counting()));
```

**The general rule:** if the result is going anywhere a human or another system will
read in order — a CSV, an API response, a log line — the default `HashMap` is wrong.
Say which map you want.

That `EnumMap` line is free performance and free ordering, and Topic 16 says there is
no reason ever to use `HashMap` with enum keys.

---

### Trap 5 — using `reduce` where you needed `collect`

**Wrong:**
```java
List<String> skus = orders.stream()
        .reduce(new ArrayList<String>(),
                (list, order) -> { list.add(order.sku()); return list; },   // mutates!
                (a, b) -> { a.addAll(b); return a; });
```

**Exact symptom:** correct sequentially. Under `.parallel()`, you get a mixture of
`ArrayIndexOutOfBoundsException`, `NullPointerException` from inside `ArrayList.add`,
silently missing elements, and occasionally a correct answer — different on each run.
Classic data race symptoms, and impossible to pin down from a stack trace that points
at JDK collection internals.

**Root cause:** `reduce`'s contract requires the accumulator to be **associative and
non-interfering, and to not mutate its arguments**. The identity value
`new ArrayList<>()` is evaluated **once**, and under parallelism every split uses that
*same* list as its starting point. Multiple threads then call `add` on one unsynchronised
`ArrayList`.

This is the direct translation of the JavaScript idiom
`orders.reduce((acc, o) => { acc.push(o.sku); return acc; }, [])`. It is fine in
JavaScript because there is one thread. It is wrong in Java because there might not be.

**Fix — use the method designed for mutable containers:**
```java
List<String> skus = orders.stream()
        .map(Order::sku)
        .toList();

// or, when you need a specific container:
List<String> skus = orders.stream()
        .map(Order::sku)
        .collect(Collectors.toCollection(ArrayList::new));
```

**The distinction to keep:**
- `reduce` → immutable accumulation. Every step produces a new value. Sums, mins,
  maxes, `String::concat` (which is O(n²) — use `joining` instead).
- `collect` → mutable accumulation. A container is modified in place. The framework
  guarantees each container is confined to one thread until the combiner merges them.

If your accumulator has a `return acc;` after mutating `acc`, you wanted `collect`.

---

## Hands-on proof

Commands you run. No fabricated output below.

### Setup

```bash
mkdir -p ~/java-lab/24 && cd ~/java-lab/24
java --version
```

### Proof 1 — the combiner never runs sequentially

`CombinerVisibility.java`:
```java
import java.util.*;
import java.util.stream.*;

public class CombinerVisibility {

    static Collector<Integer, ?, List<Integer>> noisy() {
        return Collector.of(
                () -> { System.out.println("  supplier   on " + Thread.currentThread().getName());
                        return new ArrayList<Integer>(); },
                (list, x) -> list.add(x),
                (left, right) -> { System.out.println("  COMBINER   on " + Thread.currentThread().getName());
                                   left.addAll(right); return left; },
                list -> { System.out.println("  finisher   on " + Thread.currentThread().getName());
                          return list; });
    }

    public static void main(String[] args) {
        List<Integer> input = IntStream.range(0, 1_000).boxed().toList();

        System.out.println("--- SEQUENTIAL ---");
        System.out.println("size = " + input.stream().collect(noisy()).size());

        System.out.println("--- PARALLEL ---");
        System.out.println("size = " + input.parallelStream().collect(noisy()).size());
    }
}
```

```bash
java CombinerVisibility.java
```

**What to look for:** whether `COMBINER` appears in each section, and which thread names
appear.

| What you see | What it means |
|---|---|
| Sequential: one `supplier`, no `COMBINER`, one `finisher`, all on `main` | The core claim, observed. Your combiner is dead code in a sequential pipeline. |
| Parallel: several `supplier` lines, several `COMBINER` lines, thread names like `ForkJoinPool.commonPool-worker-1` | Splitting happened. Each split got its own container; the combiner merged them pairwise. Topic 25 explains the pool. |
| Parallel section shows no `COMBINER` and only `main` | Your machine reported one available processor, so the common pool has parallelism 0 and everything ran on the calling thread. Check with `Runtime.getRuntime().availableProcessors()`. This is exactly the container trap in Topic 25. |
| Both sizes are `1000` | Correct. Now go break the combiner and re-run — see Proof 2. |

### Proof 2 — break the combiner and watch only the parallel path fail

Edit `noisy()` so the combiner is `(left, right) -> left;` (dropping `right`).

```bash
java CombinerVisibility.java
```

**What to look for:** the two `size =` lines.

| What you see | What it means |
|---|---|
| Sequential `1000`, parallel something less than 1000 | The bug, isolated. Identical code, identical input, two different answers, no exception. |
| Both `1000` | No splitting occurred — see the one-processor row above. Force it: run with `-Djava.util.concurrent.ForkJoinPool.common.parallelism=4`. |
| The parallel number differs between runs | Expected, and it is the worst property of this bug class: it is not reproducible, so it cannot be bisected. |

Run it five times and record the parallel numbers.

**How to read it:** this is the empirical justification for the review rule "every
custom collector needs a parallel-vs-sequential equivalence test". You have now seen a
bug that no sequential test can catch.

### Proof 3 — the two `toMap` failures, with their exact traces

`ToMapFailures.java`:
```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class ToMapFailures {

    record Product(String sku, String name, String supplierId) {}

    public static void main(String[] args) {
        List<Product> dup = List.of(
                new Product("SKU-1", "widget", "S1"),
                new Product("SKU-1", "widget v2", "S2"));

        List<Product> nullValue = List.of(
                new Product("SKU-2", "gadget", null));

        try {
            dup.stream().collect(Collectors.toMap(Product::sku, Function.identity()));
        } catch (RuntimeException e) {
            System.out.println("DUPLICATE -> " + e.getClass().getName() + ": " + e.getMessage());
            e.printStackTrace(System.out);
        }

        try {
            nullValue.stream().collect(Collectors.toMap(Product::sku, Product::supplierId));
        } catch (RuntimeException e) {
            System.out.println("NULL VALUE -> " + e.getClass().getName() + ": " + e.getMessage());
            e.printStackTrace(System.out);
        }

        // the contrast: HashMap.put accepts the null value happily
        Map<String, String> byHand = new HashMap<>();
        nullValue.forEach(p -> byHand.put(p.sku(), p.supplierId()));
        System.out.println("HashMap.put with null value -> " + byHand);
    }
}
```

```bash
java ToMapFailures.java
```

**What to look for:** the two exception types, the two messages, and the top frames.

| What you see | What it means |
|---|---|
| `IllegalStateException: Duplicate key SKU-1 (attempted merging values ... and ...)` | Trap 1. Note that the message names the key and both values — that is usually enough to find the bad data. |
| `NullPointerException` with **no message**, top frame `java.util.HashMap.merge` | Trap 2. Note there is nothing in the trace pointing at your value extractor. This is why it is hard to diagnose without knowing the shape. |
| The final line prints a map containing a null value | The contrast that makes the point: `put` allows what `merge` forbids, so `toMap` is stricter than the hand-written loop it replaced. |
| An NPE with a helpful message naming a method | Your JDK has helpful NPE messages enabled for this path. Record what you got — it is better than what I described, and the diagnosis is the same. |

### Proof 4 — `groupingBy` order is not what you assume

`GroupOrder.java`:
```java
import java.util.*;
import java.util.stream.*;
import static java.util.stream.Collectors.*;

public class GroupOrder {
    public static void main(String[] args) {
        List<String> months = List.of(
                "2026-01","2026-02","2026-03","2026-04","2026-05",
                "2026-06","2026-07","2026-08","2026-09","2026-10","2026-11","2026-12");

        System.out.println("default  : " + months.stream().collect(groupingBy(m -> m, counting())).keySet());
        System.out.println("TreeMap  : " + months.stream().collect(groupingBy(m -> m, TreeMap::new, counting())).keySet());
        System.out.println("Linked   : " + months.stream().collect(groupingBy(m -> m, LinkedHashMap::new, counting())).keySet());
    }
}
```

```bash
java GroupOrder.java
```

**What to look for:** whether the first line is in the same order as the other two.

| What you see | What it means |
|---|---|
| The `default` line is in a different order from `TreeMap`/`Linked` | Trap 4, observed. `HashMap` iteration order follows hash codes, not insertion or natural order. |
| All three look the same | Possible — `HashMap` order is *unspecified*, not *random*, and with these keys it may coincide. Do not conclude it is safe. Add more keys, or keys with larger hash spread, and try again. The unspecified-ness is the point, not any particular ordering. |
| `TreeMap` is sorted, `Linked` matches the input order | Correct, and this is why you name the map factory. |

### Proof 5 — verify a collector's characteristics

`Characteristics.java`:
```java
import java.util.*;
import java.util.stream.*;
import static java.util.stream.Collectors.*;

public class Characteristics {
    public static void main(String[] args) {
        System.out.println("toList   : " + toList().characteristics());
        System.out.println("toSet    : " + toSet().characteristics());
        System.out.println("toMap    : " + toMap(Object::toString, Object::toString).characteristics());
        System.out.println("joining  : " + joining().characteristics());
        System.out.println("counting : " + counting().characteristics());
        System.out.println("grouping : " + groupingBy(Object::toString).characteristics());
    }
}
```

```bash
java Characteristics.java
```

**What to look for:** which collectors declare `IDENTITY_FINISH` and which declare
`UNORDERED`.

| What you see | What it means |
|---|---|
| `toSet` includes `UNORDERED`, `toList` does not | Sets have no meaningful encounter order, so the pipeline is free to ignore ordering constraints for them. Lists are not. |
| `joining` and `counting` do **not** include `IDENTITY_FINISH` | They have real finishers — `StringBuilder` → `String`, and a boxing step. |
| `groupingBy` includes `UNORDERED` but not `IDENTITY_FINISH` (or the reverse on your JDK) | Record what you see. The lesson is that these flags are per-collector facts you can query, not something to guess. |

**How to read it:** when you write your own collector, this is the output you are
choosing to produce. Declaring `IDENTITY_FINISH` when `A != R` is a `ClassCastException`
at collection time; declaring `CONCURRENT` on a non-thread-safe container is silent
corruption.

---

## Practice exercises

### 1 — Easy: express each result with one collector expression

Given `List<Order>` where `Order` is `record Order(String id, String customerId,
Status status, long totalPence, List<OrderLine> lines)`, write a single stream
expression for each:

1. A `List<String>` of all order ids.
2. A `Set<String>` of distinct customer ids, sorted.
3. `Map<Status, Long>` — count per status, with **every** status present even at zero.
4. `Map<Status, Long>` — total revenue per status.
5. `Map<Boolean, List<Order>>` — split into over-£100 and not.
6. A single `String` of all ids joined with `", "` inside square brackets.
7. `Map<String, List<String>>` — customer id to the list of their order ids.
8. `Map<String, Optional<Order>>` — customer id to their largest order.
9. `LongSummaryStatistics` over all order totals.
10. `Map<Status, Set<String>>` — status to the distinct SKUs in orders of that status.

For 3, note that `groupingBy` will not produce absent statuses. State how you would get
them, and whether `partitioningBy`'s always-both-keys guarantee is relevant.

### 2 — Medium: the audit (combines Topics 01, 12, 13, 14, 16, 22, 23)

Find the **seven** defects, give the exact observable symptom of each, and rewrite.

```java
public class CustomerSummaryService {

    public Map<Customer, Double> revenueByCustomer(List<Order> orders) {

        Map<Customer, Double> byCustomer = orders.stream()
                .collect(Collectors.toMap(
                        Order::customer,
                        o -> o.totalPence() / 100.0));

        Map<Status, List<Order>> byStatus = orders.stream()
                .collect(Collectors.groupingBy(Order::status));

        List<String> couponCodes = orders.stream()
                .collect(Collectors.groupingBy(Order::couponCode))
                .keySet().stream()
                .collect(Collectors.toList());
        couponCodes.sort(null);

        List<String> allSkus = orders.stream()
                .reduce(new ArrayList<String>(),
                        (list, o) -> { o.lines().forEach(l -> list.add(l.sku())); return list; },
                        (a, b) -> { a.addAll(b); return a; });

        return byCustomer;
    }
}
```

Hints, one per defect: what `toMap` does with two orders from the same customer; what
`Customer` needs for that to work at all (Topic 13); what `double` does to money
(Topic 01); what `groupingBy` does when `couponCode` is null; what map type
`groupingBy` returned and whether `Status` keys deserve better (Topic 16); what
`sort(null)` requires of the elements (Topic 14); which method `reduce` should have
been.

### 3 — Hard: production simulation on `orderflow`

**Part A.** Write a named `Collector<Order, ?, MonthlyFigures>` — implementing the
interface, not `Collector.of` — that computes count, gross revenue in pence, the
largest order, and the distinct SKU set, in one pass. Choose your accumulation type `A`
deliberately and write down why.

**Part B.** Write the two tests from Trap 3: a direct combiner test, and a
parallel-versus-sequential equivalence test over at least 100,000 synthetic orders.

**Part C.** Break your combiner in three different ways and record which tests catch
which:
- drop the right-hand side entirely;
- merge but return the wrong container;
- use a non-associative merge (e.g. `left.size() > right.size() ? left : right`).

For each, state whether the sequential test caught it. Then state, in one sentence, what
that tells you about test coverage as a proxy for correctness. (Topic 63 will make this
argument formally with mutation testing.)

**Part D.** Now build the full two-level report from Example 2 using your collector.
Run it over 2,000,000 synthetic orders across 50,000 customers and 24 months. Record
peak heap with:
```bash
java -Xmx2g -Xlog:gc:file=report.log:time,uptime RevenueReport
jcmd <pid> GC.heap_info
```

**Part E.** Compare against the naive version that groups into
`Map<CustomerId, Map<YearMonth, List<Order>>>` first. Report peak heap for both.
Explain the difference in terms of what each version holds in memory at its peak.

**Part F.** Now write the equivalent SQL `GROUP BY`. Argue for whichever version you
would actually ship, and name the specific circumstance that would change your answer.
Be concrete: "if the orders live in two different databases" is a real answer;
"it depends" is not.

---

## Interview questions

### Q1 — "What are the four functions of a `Collector`, and when does each run?"

**Mid-level answer:** "Supplier makes the container, accumulator adds elements, combiner
merges containers, finisher converts to the result. The combiner is for parallel
streams."

**Senior answer:** "Supplier, accumulator, combiner, finisher, plus a characteristics
set. Sequentially: supplier once, accumulator once per element, combiner **never**,
finisher once. In parallel: supplier once per split, accumulator once per element spread
across threads, combiner once per merge — splits minus one — finisher once at the end.
The 'combiner never runs sequentially' part is the one that matters in practice, because
it means a broken combiner is dead code in every test you write and every local run. It
surfaces the first time anything parallelises the stream, and it surfaces as wrong
numbers rather than an exception. So my rule is that any custom collector gets a
parallel-versus-sequential equivalence test over a large input, not just a unit test of
each function. The combiner also has to be associative and has to return the merged
container — `(a, b) -> a` compiles and silently drops half your data."

**What separates them:** the exact run counts, and the leap to a testing rule. The
mid-level answer knows the roles; the senior answer knows the failure mode and what to
do about it.

**Interviewer's follow-up:** "How would you write that equivalence test?" They want
`parallelStream().collect(c)` compared against `stream().collect(c)` over enough data
to force splitting, and awareness that on a small input or a one-core machine it will
not split at all.

---

### Q2 — "`Collectors.toMap` is throwing `IllegalStateException: Duplicate key`. What do you do?"

**Mid-level answer:** "Add a merge function — `(a, b) -> b` — so it takes the last one."

**Senior answer:** "First I'd resist adding the merge function reflexively, because
`toMap` throwing is a *feature*: `HashMap.put` would have silently overwritten, and the
API designers decided loud beats silent. So the first question is whether a duplicate
SKU is legal in this domain. If it is, I pick the merge rule that matches the business
meaning — last-write-wins, first-wins, or an actual merge like 'keep the one with the
later `updatedAt`' — and I write it explicitly so the next reader knows a decision was
made. If duplicates are a data error, I leave it throwing but I add context, because
`Duplicate key SKU-4471` alone doesn't tell me which import produced it. And I'd check
the key's `equals`/`hashCode` while I'm there — a broken `hashCode` produces the
opposite bug, where logically-equal keys don't collide and the map quietly holds
duplicates without ever throwing."

**What separates them:** treating the exception as information rather than an obstacle,
and volunteering the equals/hashCode failure mode, which is the silent version of the
same problem.

**Interviewer's follow-up:** "What about a null value?" They want: `toMap` uses
`HashMap.merge`, which throws NPE on a null value, unlike `put` — and the stack trace
names JDK internals only, so you have to recognise the shape.

---

### Q3 — "What's the difference between `reduce` and `collect`?"

**Mid-level answer:** "`collect` is for building collections, `reduce` is for combining
into a single value."

**Senior answer:** "They're both reductions; the difference is mutability. `reduce` is
an *immutable* reduction — the accumulator must return a new value each step and must
not mutate its arguments. `collect` is a *mutable* reduction — a container is modified
in place, and the framework guarantees each container is thread-confined until the
combiner merges them. So `collect` avoids one allocation per element, which is why
building a list with `collect` is right and building one with `reduce` is wrong. The
wrong version is seductive because it's the idiomatic JavaScript pattern —
`reduce((acc, x) => { acc.push(x); return acc; }, [])` — and it works fine
sequentially in Java too. Under `.parallel()` it breaks badly, because `reduce`'s
identity value is evaluated once and every split starts from that same list, so you get
concurrent unsynchronised `ArrayList.add` and a mixture of exceptions and missing
elements. The tell in code review is a `return acc;` after mutating `acc` inside a
`reduce`."

**What separates them:** naming the mutable/immutable distinction as *the* difference,
knowing why the wrong version is tempting for someone with a JS background, and giving
a concrete review heuristic.

**Interviewer's follow-up:** "Is `String` concatenation with `reduce` wrong?" It's
correct but O(n²) because strings are immutable; `Collectors.joining` uses a
`StringBuilder`.

---

### Q4 — "You need count, sum, and the maximum from one stream. How?"

**Mid-level answer:** "Stream it three times, or collect to a list and compute all three
from the list."

**Senior answer:** "Three options depending on the shape. If they're all numeric
aggregates of the same field, `mapToLong(Order::totalPence).summaryStatistics()` gives
count, sum, min, max and average in one pass with no boxing — that's the answer most
people don't know exists. If I need the maximum *order* rather than the maximum value,
`Collectors.teeing` lets me run two collectors over one pass and merge their results —
`teeing(counting(), maxBy(comparator), MyRecord::new)`. And nesting `teeing` gets me
three or four aggregates, though two levels is where it stops being readable and I'd
write a named custom collector instead. Collecting to a list first also works and is
perfectly fine at small scale — but it materialises everything, so on a two-million-row
nightly job it's the difference between streaming from a cursor and holding the whole
result set in heap."

**What separates them:** `summaryStatistics` and `teeing` both volunteered, plus a
scale-dependent judgement rather than a single dogmatic answer.

**Interviewer's follow-up:** "When does the collect-to-a-list version become wrong?"
When the source is unbounded or does not fit in heap — a JDBC cursor, a file, a Kafka
topic.

---

### Q5 — "What map does `groupingBy` return, and does it matter?"

**Mid-level answer:** "A `HashMap`. It usually doesn't matter."

**Senior answer:** "The two-argument form specifies only 'some Map' and in practice
returns a `HashMap`, so iteration order is a function of hash codes and table size —
unspecified, and it changes when the table resizes. That's fine if you're doing lookups
and nothing else. It's a bug the moment the result is written in order: a CSV, an API
response, a log line, anything a diff-based downstream job consumes. It's especially
nasty because the order is stable enough during development to look deliberate, then
changes in production when the data grows past a resize threshold. So I name the map
factory: `TreeMap::new` for sorted keys, `LinkedHashMap::new` for encounter order, and
`() -> new EnumMap<>(Status.class)` whenever the key is an enum — `EnumMap` is an
ordinal-indexed array, so it's both ordered and faster than hashing, and there's no
reason to use `HashMap` with enum keys at all. Same reasoning applies to `toSet`, which
gives a `HashSet`; `toCollection(LinkedHashSet::new)` when order matters."

**What separates them:** knowing the order is unspecified rather than merely "insertion
order isn't kept", the failure mode where it changes under growth, and reaching for
`EnumMap` unprompted.

**Interviewer's follow-up:** "Does `groupingBy` produce keys for empty groups?" No — a
group only exists if at least one element mapped to it. `partitioningBy` always
produces both `true` and `false`, which is the reason to prefer it for a boolean split.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. The combiner is only called under parallelism, which makes it structurally
   untestable by default. Was including it in the `Collector` interface the right design?
   What would the alternative have cost?

2. `toMap` throws on duplicate keys while `HashMap.put` overwrites. Both are defensible.
   Which would you have chosen as the default, and what does your answer say about your
   view of API design?

3. `toMap` throws NPE on a null *value* because it is built on `HashMap.merge`. Is that
   a deliberate design decision or an implementation leak? Does it matter which?

4. `groupingBy` requires non-null keys, but `HashMap` allows one null key. Explain why
   the collector is stricter than the container it builds.

5. `collect` and `reduce` both perform reductions. If you were designing the API from
   scratch, would you unify them? What would you lose?

6. You have written a custom collector and it passes every sequential test. Describe,
   without running anything, the argument you would make in a code review that it is
   still not proven correct.

7. Example 2's report can be written as SQL, as nested collectors, or as loops. Rank
   them for a team of five mid-level engineers maintaining the code for three years.
   Then rank them for a one-off migration script. Explain why the rankings differ.

---

## Quick reference card

### The `Collector` contract

```java
Collector<T, A, R>       // T = element, A = mutable accumulator, R = result

supplier()      Supplier<A>          // once sequentially, once per split in parallel
accumulator()   BiConsumer<A, T>     // once per element
combiner()      BinaryOperator<A>    // NEVER sequentially; splits-1 times in parallel
finisher()      Function<A, R>       // once
characteristics()                    // IDENTITY_FINISH, UNORDERED, CONCURRENT
```

### Building one

```java
Collector.of(supplier, accumulator, combiner)                   // A == R, IDENTITY_FINISH
Collector.of(supplier, accumulator, combiner, finisher)
Collector.of(supplier, accumulator, combiner, finisher, characteristics...)
```

### Cheat sheet by intent

| I want | Collector |
|---|---|
| a list | `stream.toList()` (no collector needed) |
| a specific collection | `toCollection(TreeSet::new)` |
| a map | `toMap(key, value, mergeFn, mapFactory)` |
| grouped | `groupingBy(classifier, mapFactory, downstream)` |
| a boolean split, both keys guaranteed | `partitioningBy(predicate, downstream)` |
| a count | `counting()` |
| a sum | `summingLong(f)` |
| five aggregates in one pass | `summarizingLong(f)` or `mapToLong(f).summaryStatistics()` |
| a joined string | `joining(", ", "[", "]")` |
| the biggest element | `maxBy(comparator)` → `Optional<T>` |
| transform then collect | `mapping(f, downstream)` |
| flatten then collect | `flatMapping(f, downstream)` |
| filter, keeping empty groups | `filtering(p, downstream)` |
| post-process the result | `collectingAndThen(downstream, finisher)` |
| two aggregates, one pass | `teeing(c1, c2, merger)` |

### Gotchas checklist

- [ ] `toMap` throws `IllegalStateException: Duplicate key ...` — supply a merge function on purpose.
- [ ] `toMap` throws NPE on a **null value** (it uses `HashMap.merge`, not `put`).
- [ ] `groupingBy` throws NPE on a **null key** from the classifier.
- [ ] `groupingBy` returns a `HashMap` — unspecified order. Name the map factory.
- [ ] Enum keys → `() -> new EnumMap<>(X.class)`, always.
- [ ] `groupingBy` omits empty groups; `partitioningBy` always has both keys.
- [ ] The combiner **never runs sequentially**. Test it explicitly, and test parallel-vs-sequential.
- [ ] A combiner must merge **and return** the container, and must be associative.
- [ ] Never `reduce` with a mutable accumulator — that is what `collect` is for.
- [ ] `collect(toList())` is mutable; `stream.toList()` is not. `.add()` on the latter throws.
- [ ] `Collectors.summingDouble` on money is Topic 01's `double` bug wearing a collector.
- [ ] Do not declare `CONCURRENT` unless your container is genuinely thread-safe.

---

## When would I use this at work?

**1. Every reporting or summary endpoint.**
"Orders per status", "revenue per customer per month", "top 10 SKUs by volume" —
`groupingBy` with a downstream collector is the standard shape, and it replaces the
30-line nested-loop version with something a reviewer can check by reading. This is the
single most common use.

**2. Building lookup indexes at startup.**
`Map<Sku, Product>` from a catalogue load, `Map<CustomerId, List<Order>>` from a page of
results. This is where `toMap`'s duplicate-key exception either saves you from silent
data loss or takes your service down on startup — and knowing which you want is the
whole skill.

**3. Reviewing someone's custom collector.**
The combiner is the first thing you look at, and "is there a parallel-vs-sequential
test?" is the first question you ask. Nobody else on the team will ask it, because the
sequential tests all pass. This is the highest-leverage review comment in this phase.

---

## Connected topics

**Prerequisites:**
- **23 — Streams I**: `collect` is a terminal operation; everything about laziness and
  single-pass execution still applies.
- **21 — Lambdas** and **22 — Method references**: the four collector functions are
  functional interfaces, and `ArrayList::new` / `TreeMap::new` are constructor
  references.
- **01 — Primitives and boxing**: why `summingLong` beats `summingDouble` for money and
  why `summarizingLong` avoids boxing.
- **12 — HashMap internals**: why `groupingBy`'s default iteration order changes on
  resize.
- **13 — equals/hashCode**: `toMap` and `groupingBy` keys depend on it; a broken
  `hashCode` produces silent duplicates.
- **14 — Comparable/Comparator**: `maxBy`, `minBy`, `TreeMap::new`, `TreeSet::new`.
- **16 — EnumMap/EnumSet**: `groupingBy` with an enum classifier should always name
  `EnumMap`.

**This unlocks:**
- **25 — Parallel streams**: the combiner's reason for existing, the common
  `ForkJoinPool`, and why splitting behaviour depends on the source.
- **26 — Optional**: what `maxBy`/`minBy` return.
- **27 — Records**: the natural result type for a `teeing` merger.
- **50 — Hibernate N+1** and **47 — Spring Data projections**: where the honest answer
  is "do this aggregation in SQL, not in the JVM".
- **63 — Mutation testing**: the formal version of Exercise 3's argument that passing
  tests do not prove correctness.
- **77 — JMH**: how to compare the collector version against the loop version properly.
- **79 — Heap dumps**: where you find the memory difference from Exercise 3 Part E.

---

*Java baseline 21. `Collectors` has been stable since Java 8; `flatMapping` and
`filtering` arrived in 9, `teeing` in 12, `toUnmodifiableList/Set/Map` in 10 — all
available on 21. Nothing in this topic changed in Java 25, though `Stream.gather` and
`Gatherers` (25) give you the intermediate-operation counterpart to `Collector` that
the API had been missing since Java 8.*
