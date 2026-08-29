# 10 — Collections Framework: Interfaces, Contracts, Iteration, Fail-Fast

## Phase: 1 — Core Language
## Category: FOUNDATION
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

A library.

The **librarian** promises you certain things. Those promises are the *contract*:

- **The numbered shelf** (a `List`). Every book has a position. Position 3 is position
  3 until someone moves it. You may put two identical copies of the same book on it.
- **The members' register** (a `Set`). A name is either in it or it is not. Writing a
  name twice changes nothing. There is no "third name" — no positions at all.
- **The coat check** (a `Map`). You hand over a ticket, you get back exactly one coat.
  One ticket, one coat. Two people cannot share a ticket number.
- **The queue at the desk** (a `Queue` / `Deque`). Things come out in a defined order
  relative to how they went in.

Now the crucial bit, and it is the whole reason this is a FOUNDATION topic:

> **How the librarian achieves the promise is a separate decision from the promise
> itself.**

The numbered shelf could be a physical shelf (`ArrayList`) or a chain of boxes each
pointing at the next (`LinkedList`). Both keep the promise. They differ enormously in
how long it takes to fetch book 4,471.

And one more rule, which is this topic's headline trap:

> **If you rearrange the shelf while someone is walking along it reading the spines,
> the librarian will usually — not always — stop you and shout.**

"Usually, not always" is precise, not sloppy. It is `ConcurrentModificationException`,
and understanding *why* it is only "usually" separates mid from senior.

---

## The bridge from what you know

### What transfers

You already have four of these:

```ts
const lines: OrderLine[] = [];          // Array
const skus  = new Set<string>();        // Set
const stock = new Map<string, number>(); // Map
for (const line of lines) { ... }        // iteration protocol
```

```java
List<OrderLine>      lines = new ArrayList<>();
Set<String>          skus  = new HashSet<>();
Map<String, Integer> stock = new HashMap<>();
for (OrderLine line : lines) { ... }
```

The concepts map. `for...of` and the enhanced `for` loop are the same idea, and both
are sugar over an iterator protocol. **Verdict: PARTIAL analogue**, with four real
differences.

### Difference 1 — interface and implementation are separate types

In JavaScript, `Array` is both the contract and the implementation. There is one
array. In Java there is `List` (the contract) and `ArrayList`, `LinkedList`,
`CopyOnWriteArrayList`, `List.of(...)` and half a dozen others (the implementations).

This is why the idiomatic Java declaration is:

```java
List<OrderLine> lines = new ArrayList<>();
//  ^^^^ the contract          ^^^^^^^^^ the performance decision
```

Declare the **interface** on the left. It means you can change the implementation
later without touching any caller — and it means your method signatures say what they
need, not what they happen to have been given.

### Difference 2 — ordering and null policy are per-implementation promises

`HashSet` has **no** iteration order. Not insertion order, not sorted — *no defined
order*, and it can change when the set resizes. `LinkedHashSet` gives insertion order.
`TreeSet` gives sorted order. In JavaScript, `Set` and `Map` always preserve insertion
order, so you have never had to think about this. In Java it is a decision you make
per collection, and getting it wrong produces output that is stable in tests and
different in production.

Same for nulls: `HashMap` allows one null key and any number of null values.
`TreeMap` throws `NullPointerException` on a null key. `List.of(...)` rejects nulls
entirely. `ArrayDeque` rejects nulls. There is no single rule; it is per class.

### Difference 3 — `Map` is not a `Collection`

`Map` does **not** extend `Collection`. It is a separate branch of the hierarchy. You
get collection *views* of it: `keySet()`, `values()`, `entrySet()`. Those views are
live — changing the view changes the map, and vice versa.

### Difference 4 — mutation during iteration throws

```ts
const arr = [1, 2, 3];
for (const x of arr) { if (x === 2) arr.splice(1, 1); }   // no error, weird results
```

Java throws `ConcurrentModificationException`. That is a genuine improvement — a
silent wrong answer becomes a loud failure — but it is a **best-effort** improvement,
and the details matter. That is the core of this doc.

### Mapping table

| You know | Java | Verdict |
|---|---|---|
| `Array` | `List` interface, `ArrayList` implementation | **PARTIAL** — contract and implementation split |
| `Set` (insertion-ordered) | `HashSet` (**unordered**) / `LinkedHashSet` (insertion) | **PARTIAL** — you must choose the ordering |
| `Map` (insertion-ordered) | `HashMap` (**unordered**) / `LinkedHashMap` / `TreeMap` | **PARTIAL** — same |
| plain object as a dictionary | `Map` only | — Java has no "object as bag of keys" |
| `for...of` / `Symbol.iterator` | enhanced `for` / `Iterable` + `Iterator` | **HONEST ANALOGUE** |
| `Object.freeze(arr)` | `List.of(...)` / `Collections.unmodifiableList` | **PARTIAL** — Java distinguishes *immutable* from an *unmodifiable view* |
| Mutating an array while iterating: silent | `ConcurrentModificationException`: loud, usually | **PARTIAL** — and "usually" is the lesson |
| `arr.length` / `set.size` / `map.size` | `size()` on everything | **HONEST ANALOGUE** |
| No queue/deque type | `Queue`, `Deque`, `ArrayDeque`, `PriorityQueue` | **NO ANALOGUE** — a whole family you do not have |

---

## What is this?

### The hierarchy

```
Iterable<E>                       anything you can put in a for-each loop
 └── Collection<E>                add / remove / contains / size / iterator
      ├── List<E>                 ordered, indexed, duplicates allowed
      │     ArrayList, LinkedList, CopyOnWriteArrayList, List.of(...)
      ├── Set<E>                  no duplicates (by equals/hashCode)
      │     HashSet, LinkedHashSet, TreeSet, EnumSet, Set.of(...)
      └── Queue<E>                ordered for insertion/removal at ends
            └── Deque<E>          double-ended
                  ArrayDeque, LinkedList
            PriorityQueue         ordered by comparator, NOT by insertion

Map<K,V>                          SEPARATE — not a Collection
      HashMap, LinkedHashMap, TreeMap, EnumMap, ConcurrentHashMap, Map.of(...)
```

### The contract axes — how you actually choose

Pick from the **contract**, not from habit. There are five questions:

| Question | If the answer is... | You want |
|---|---|---|
| Do duplicates mean anything? | yes | `List` |
| | no, membership is the question | `Set` |
| Do I look things up by a key? | yes | `Map` |
| What order do I need? | insertion / positional | `ArrayList`, `LinkedHashSet`, `LinkedHashMap` |
| | sorted | `TreeSet`, `TreeMap` |
| | do not care | `HashSet`, `HashMap` (fastest) |
| | first-in-first-out or LIFO | `ArrayDeque` |
| | by priority | `PriorityQueue` |
| Will it be mutated after creation? | no | `List.of` / `Set.of` / `Map.of` |
| Will multiple threads touch it? | yes | `ConcurrentHashMap`, `CopyOnWriteArrayList` (Topics 92, 97) |

### The contract table you should be able to reproduce from memory

| Implementation | Order | Duplicates | Nulls | `contains` cost | Notes |
|---|---|---|---|---|---|
| `ArrayList` | positional | yes | yes | O(n) | default `List` |
| `LinkedList` | positional | yes | yes | O(n) | also a `Deque`; Topic 11 |
| `List.of(...)` | positional | yes | **no** | O(n) | immutable, throws on mutation |
| `HashSet` | **none** | no | one null | O(1) avg | default `Set` |
| `LinkedHashSet` | insertion | no | one null | O(1) avg | |
| `TreeSet` | sorted | no | **no** | O(log n) | needs `Comparable`/`Comparator` |
| `Set.of(...)` | **none** | no | **no** | O(1) avg | immutable; throws on duplicate args |
| `HashMap` | **none** | keys unique | one null key, null values | O(1) avg | default `Map`; Topic 12 |
| `LinkedHashMap` | insertion or access | keys unique | as `HashMap` | O(1) avg | LRU base; Topic 15 |
| `TreeMap` | sorted by key | keys unique | **no null key** | O(log n) | `NavigableMap`; Topic 14 |
| `ArrayDeque` | insertion (both ends) | yes | **no** | O(n) | best queue/stack; Topic 11 |
| `PriorityQueue` | **heap order, not sorted iteration** | yes | **no** | O(n) | `poll()` is ordered; iteration is not |

The `PriorityQueue` row catches people constantly: iterating a `PriorityQueue` does
**not** give you sorted order. Only repeated `poll()` does.

### Iteration and fail-fast

The enhanced `for` loop is sugar:

```java
for (OrderLine line : lines) { ... }
```
compiles to roughly
```java
for (Iterator<OrderLine> it = lines.iterator(); it.hasNext(); ) {
    OrderLine line = it.next();
    ...
}
```

Every non-concurrent `java.util` collection keeps an `int modCount` field, incremented
on every **structural** modification (one that changes the size — add, remove, clear;
*not* `set(i, x)`). When you create an iterator, it snapshots `modCount` into
`expectedModCount`. On each `next()` and `remove()` it compares them:

```java
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

That is the entire mechanism. Now the part that matters:

> **`modCount` is a plain, non-`volatile` `int`. Fail-fast is a debugging aid, not a
> guarantee.**

Three consequences follow, and a senior candidate names all three:

1. **Cross-thread modifications may not be seen at all.** Without `volatile`, there is
   no happens-before edge (Topic 86). Thread A's increment may sit in a store buffer
   or a register; Thread B's iterator may never observe it. You get silently wrong
   results instead of an exception.
2. **`int` overflow.** After 2^32 structural modifications the counter wraps. If it
   wraps to exactly the expected value, the check passes. Astronomically unlikely,
   and it is why the Javadoc says "best-effort".
3. **Some single-threaded bugs are missed entirely.** The famous one: removing the
   **second-to-last** element from an `ArrayList` in a for-each loop does not throw. It
   just silently skips the last element. Details in Trap 1.

The Javadoc says this outright: *"Fail-fast iterators throw
`ConcurrentModificationException` on a best-effort basis. Therefore, it would be
wrong to write a program that depended on this exception for its correctness."*

### Fail-safe (weakly consistent) iterators

The concurrent collections take the opposite approach. `ConcurrentHashMap`'s iterator
is **weakly consistent**: it never throws `ConcurrentModificationException`, it
reflects the state at some point at or since creation, and it may or may not show you
concurrent modifications. `CopyOnWriteArrayList` iterates over an immutable snapshot
taken at `iterator()` time — so it never throws and never sees later changes, at the
cost of copying the whole array on every write.

Neither is "better". They answer different questions: fail-fast says *"you have a bug,
here it is"*; weakly consistent says *"I will not throw, but I make weaker promises
about what you see"*. Topics 92 and 97 go deep.

---

## Why does it matter?

**1. It is the choice you make most often.** Nearly every method you write touches a
collection. Choosing from the contract rather than from habit costs five seconds and
compounds across a codebase; choosing `ArrayList` for everything produces O(n) lookups
inside loops that were fine in a 500-row fixture and are 20 minutes on real data.

**2. The failures are silent.** `ConcurrentModificationException` at least shouts. The
worse cases do not: a for-each that skips the last element, a `HashSet` iteration order
that differs between your laptop and CI, a `HashMap` shared across request threads that
loses entries with no exception at all. All three produce wrong data and a green
dashboard.

**3. Returning the wrong thing makes it your API forever.** Hand a caller your internal
mutable list and it is now part of your contract — they will mutate it, and you will
find out when a `ConcurrentModificationException` appears in *their* stack trace.
Deciding copy-versus-view at the point you write the getter costs nothing; retrofitting
it costs a coordinated change across teams.

**4. Everything downstream assumes it.** Hibernate returns collections. Streams consume
them. Jackson serialises them. Spring injects them. The contracts in this topic are the
vocabulary of Phases 2 through 5, and every later topic assumes you can pick from them
without stopping to think.

---

## Syntax breakdown

New constructs only. Topics 05–07 covered generics and wildcards, which you will see
throughout these signatures.

### Declaring to the interface

```java
List<OrderLine> lines = new ArrayList<>();
Set<String>     skus  = new LinkedHashSet<>();
Map<String,Integer> stock = new HashMap<>();
```

| Bit | What it means |
|---|---|
| `List<OrderLine>` on the left | The contract. Everything downstream sees only this. |
| `new ArrayList<>()` | The implementation, chosen here and changeable here alone. |
| `<>` (the diamond) | Type arguments inferred from the left-hand side. Topic 05. |

For **parameters**, go one step further and use the widest interface that works:

```java
void audit(Collection<? extends OrderLine> lines)   // not List, not ArrayList
```
`Collection` because you only iterate; `? extends` because you only read (Topic 07).

### Factory methods for immutable collections (Java 9+)

```java
List<String> skus   = List.of("SKU-4471", "SKU-9002");
Set<String>  active = Set.of("PENDING", "CONFIRMED");
Map<String,Integer> limits = Map.of("gold", 10, "silver", 5);
Map<String,Integer> big    = Map.ofEntries(
        Map.entry("gold", 10), Map.entry("silver", 5), Map.entry("bronze", 2));
```

| Bit | What it means |
|---|---|
| `List.of` | **Immutable**. Every mutator throws `UnsupportedOperationException`. |
| Null handling | Rejects null elements with `NullPointerException` at construction. |
| `Set.of` duplicates | Throws `IllegalArgumentException` on duplicate arguments — unlike `new HashSet<>(list)`, which silently dedupes. |
| `Map.of` | Overloads up to 10 key-value pairs. Beyond that, `Map.ofEntries`. |
| Iteration order | `Set.of` and `Map.of` iteration order is **deliberately randomised per JVM run**, to stop you depending on it. |

That last row is worth pausing on. Since Java 9 the immutable set/map implementations
apply a per-JVM salt to their iteration order. A test that passes locally can fail in
CI purely because the order differed. That is intentional: it makes an unstated
dependency on order fail loudly rather than in production.

### Views versus copies

```java
List<String> live = new ArrayList<>(List.of("a", "b", "c"));

List<String> unmodifiableView = Collections.unmodifiableList(live);  // VIEW
List<String> immutableCopy    = List.copyOf(live);                   // COPY
List<String> sub              = live.subList(0, 2);                  // VIEW
Set<String>  keys             = someMap.keySet();                    // VIEW
```

| Expression | Is it a copy? | What happens if `live` changes |
|---|---|---|
| `Collections.unmodifiableList(live)` | no — a wrapper | the "unmodifiable" list changes too |
| `List.copyOf(live)` | yes | unaffected |
| `live.subList(0,2)` | no — a window | structural change to `live` makes the sublist throw |
| `map.keySet()` | no | reflects the map; `remove` on it removes from the map |

"Unmodifiable" means *you cannot modify it through this reference*. It does not mean
"nothing can change". This distinction is the source of Trap 4.

### Removing safely during iteration

```java
// 1. removeIf — preferred, one line, no iterator visible
lines.removeIf(line -> line.quantity() == 0);

// 2. explicit Iterator.remove — when the condition needs more than a predicate
for (Iterator<OrderLine> it = lines.iterator(); it.hasNext(); ) {
    OrderLine line = it.next();
    if (line.quantity() == 0) {
        it.remove();        // updates expectedModCount too — this is the point
    }
}
```

`Iterator.remove()` is the **only** legal way to structurally modify a collection
while iterating it, because it is the only path that keeps `expectedModCount` in sync.

### `entrySet` iteration

```java
for (Map.Entry<String, Integer> e : stock.entrySet()) {
    System.out.println(e.getKey() + " -> " + e.getValue());
    e.setValue(e.getValue() - 1);      // legal: setValue writes THROUGH to the map
}
```
`setValue` on an entry is not a structural modification, so it does not trip
fail-fast. `stock.put(newKey, v)` inside the loop *is* structural, and does.

### Modern `Map` methods you should reach for

```java
stock.getOrDefault("SKU-4471", 0);
stock.putIfAbsent("SKU-4471", 0);
stock.computeIfAbsent(sku, k -> new ArrayList<>()).add(line);
stock.merge(sku, quantity, Integer::sum);          // Topic 01, Trap 4
stock.compute(sku, (k, v) -> v == null ? 1 : v + 1);
stock.forEach((k, v) -> log.info("{} = {}", k, v));
```

`computeIfAbsent` returning the value and letting you chain `.add(...)` replaces the
get-null-check-put-add pattern entirely, and it is atomic on `ConcurrentHashMap`
(Topic 92).

---

## Example 1 — minimal

The contract differences, made visible in eight lines.

```java
import java.util.*;

public class ContractsMinimal {
    public static void main(String[] args) {

        List<String> skus = new ArrayList<>(List.of("SKU-9002", "SKU-4471", "SKU-9002"));
        Set<String>  hash = new HashSet<>(skus);
        Set<String>  link = new LinkedHashSet<>(skus);
        Set<String>  tree = new TreeSet<>(skus);

        System.out.println("list   : " + skus);   // duplicates kept, order kept
        System.out.println("hash   : " + hash);   // deduped, order UNDEFINED
        System.out.println("linked : " + link);   // deduped, insertion order
        System.out.println("tree   : " + tree);   // deduped, sorted

        Map<String, Integer> counts = new HashMap<>();
        for (String sku : skus) {
            counts.merge(sku, 1, Integer::sum);
        }
        System.out.println("counts : " + counts);
    }
}
```

Run it twice. Then run it on a different JDK build. `hash` may print in a different
order; the other three may not. If your code depends on `hash`'s order, you have a bug
that reproduces on someone else's laptop and not yours.

---

## Example 2 — production scenario

`orderflow` receives a batch of order-intake messages. For each batch it must:

- keep every line, in the order received, for audit (duplicates are real — a customer
  can order two of the same SKU on separate lines),
- know the **distinct** SKUs so it can fetch prices in one query rather than N,
- accumulate a per-SKU quantity total to check against inventory,
- track which order IDs it has already seen this batch, for idempotency,
- and process line-level failures in arrival order without recursion.

Five different jobs. Five different contracts. Using an `ArrayList` for all of them is
the mistake this example exists to prevent.

```java
package com.orderflow.intake;

import java.util.*;

public final class BatchIntake {

    public record OrderLine(long orderId, String sku, int quantity) { }

    public record BatchResult(
            List<OrderLine> accepted,
            List<String> rejected,
            Map<String, Integer> quantityBySku,
            Set<String> distinctSkus) { }

    private final InventoryClient inventory;
    private final PriceClient prices;

    public BatchIntake(InventoryClient inventory, PriceClient prices) {
        this.inventory = inventory;
        this.prices = prices;
    }

    public BatchResult accept(List<OrderLine> incoming) {

        // 1. Audit trail: order matters, duplicates are meaningful. -> List
        List<OrderLine> accepted = new ArrayList<>(incoming.size());   // pre-sized

        // 2. Idempotency within the batch: membership only. -> Set
        //    LinkedHashSet, not HashSet, because the rejected report must be
        //    reproducible for support tickets.
        Set<Long> seenOrderIds = new LinkedHashSet<>();

        // 3. Per-SKU totals: keyed lookup, no order needed. -> HashMap
        Map<String, Integer> quantityBySku = new HashMap<>();

        // 4. Distinct SKUs for a single batched price fetch.
        //    LinkedHashSet keeps the IN () clause stable, which makes the
        //    query plan and the query cache behave predictably.
        Set<String> distinctSkus = new LinkedHashSet<>();

        // 5. Rejections, reported in arrival order. -> ArrayDeque used as a FIFO
        Deque<String> rejections = new ArrayDeque<>();

        for (OrderLine line : incoming) {

            if (!seenOrderIds.add(line.orderId())) {
                // add() returns false if already present: check-and-insert in one step
                rejections.addLast("duplicate order " + line.orderId() + " in batch");
                continue;
            }
            if (line.quantity() <= 0) {
                rejections.addLast("non-positive quantity on " + line.sku());
                continue;
            }

            accepted.add(line);
            distinctSkus.add(line.sku());
            quantityBySku.merge(line.sku(), line.quantity(), Integer::sum);
        }

        // ONE query for all prices, instead of one per line. The Set made this possible.
        Map<String, Long> priceBySku = prices.fetchAll(distinctSkus);

        // Remove any line whose SKU has no price. removeIf, NOT a for-each with remove.
        accepted.removeIf(line -> {
            boolean missing = !priceBySku.containsKey(line.sku());
            if (missing) rejections.addLast("no price for " + line.sku());
            return missing;
        });

        return new BatchResult(
                List.copyOf(accepted),                    // immutable COPY, not a view
                List.copyOf(rejections),
                Map.copyOf(quantityBySku),
                Set.copyOf(distinctSkus));
    }

    public interface InventoryClient { Map<String, Integer> available(Set<String> skus); }
    public interface PriceClient     { Map<String, Long>    fetchAll(Set<String> skus); }
}
```

Every choice, and what it bought:

| Choice | Contract reason | What breaks without it |
|---|---|---|
| `List` for `accepted` | duplicates are meaningful; order is the audit trail | a `Set` would silently merge two legitimate lines for the same SKU — the customer is charged once for two items |
| `new ArrayList<>(incoming.size())` | capacity known up front | repeated grow-and-copy; Topic 11 |
| `Set` for `seenOrderIds` | membership question, O(1) | a `List.contains` makes the loop O(n²); at 50k lines that is 1.25 billion comparisons |
| `add()` returning `boolean` | check-and-insert in one operation | a separate `contains` then `add` is two lookups and a race waiting to happen (Topic 92) |
| `LinkedHashSet` not `HashSet` for `distinctSkus` | stable `IN (...)` ordering | the generated SQL differs run to run, defeating statement caching and making plans hard to compare |
| `Map` + `merge` | keyed accumulation, no boxing dance | the get/null-check/put pattern from Topic 01, Trap 4 |
| `ArrayDeque` for rejections | FIFO with no index access | `LinkedList` works but allocates a node per entry (Topic 11) |
| `removeIf` not for-each + `remove` | fail-fast safe | `ConcurrentModificationException`, or worse, a silently skipped line |
| `List.copyOf` on return | callers cannot mutate our internals | a caller adds to `accepted` and corrupts a result another thread is reading |

That last row is the one people skip. Returning your internal mutable collection makes
it part of your API, forever.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `ConcurrentModificationException` (the headline)

**Wrong:**
```java
for (OrderLine line : lines) {
    if (line.quantity() == 0) {
        lines.remove(line);      // structural modification during iteration
    }
}
```

**Exact symptom:**
```
java.util.ConcurrentModificationException
	at java.base/java.util.ArrayList$Itr.checkForComodification(ArrayList.java:1013)
	at java.base/java.util.ArrayList$Itr.next(ArrayList.java:967)
	at com.orderflow.intake.BatchIntake.accept(BatchIntake.java:54)
```
Note the frame: `ArrayList$Itr.next`. The exception is thrown by the **iterator**, on
the call *after* the modification, not by `remove`. Line numbers vary by JDK build.

**Root cause:** `remove` incremented `modCount`. The iterator's `expectedModCount` is
now stale, and `next()` compares them.

**Fix:**
```java
lines.removeIf(line -> line.quantity() == 0);
```
or, when the condition needs more than a predicate, an explicit `Iterator` and
`it.remove()`.

**And now the part that makes this senior-level — the silent variant.** Remove the
**second-to-last** element and no exception is thrown at all:

```java
List<String> l = new ArrayList<>(List.of("a", "b", "c"));
for (String s : l) {
    if (s.equals("b")) l.remove(s);     // removes index 1 of 3
}
System.out.println(l);                   // prints [a, c] — no exception
```

Why: after removing, `size` is 2 and the iterator's `cursor` is 2. `ArrayList`'s
`hasNext()` is `cursor != size`, which is now `2 != 2` — **false**. The loop exits
normally. `checkForComodification` is never reached, because it lives in `next()`, and
`next()` is never called again.

The consequence: **the last element is silently skipped.** In `orderflow` terms, one
order line is never processed and nothing anywhere records that. This is the case that
proves fail-fast is a debugging aid and not a correctness mechanism.

---

### Trap 2 — mutating an immutable collection

**Wrong:**
```java
List<String> statuses = List.of("PENDING", "CONFIRMED");
statuses.add("SHIPPED");
```
or, more realistically, a method returns `List.of(...)` and a caller three modules
away adds to it.

**Exact symptom:**
```
java.lang.UnsupportedOperationException
	at java.base/java.util.ImmutableCollections.uoe(ImmutableCollections.java:142)
	at java.base/java.util.ImmutableCollections$AbstractImmutableCollection.add(...)
```
**with no message.** That empty message is the tell — `UnsupportedOperationException`
from `ImmutableCollections` never has one, so if you see a bare UOE, look for a
`List.of` / `Set.of` / `Map.of` / `Arrays.asList` / `Collections.unmodifiable*`
upstream.

There is a second symptom shape: `List.of("A", null)` throws
`NullPointerException` **at construction**, which is surprising if you are used to
`Arrays.asList` accepting nulls.

**Root cause:** `List.of` returns an immutable implementation whose mutators all throw.
This is correct behaviour, and it is why `List.of` is the right default for constants.

**Fix:** be explicit about intent at the point of creation.
```java
List<String> mutable = new ArrayList<>(List.of("PENDING", "CONFIRMED"));
```
And when returning from a method, return the immutable one deliberately — the UOE at
the caller is the contract working, not a bug.

---

### Trap 3 — `Arrays.asList` is neither a copy nor immutable

**Wrong:**
```java
String[] skuArray = readSkusFromFile();
List<String> skus = Arrays.asList(skuArray);
skus.add("SKU-NEW");             // UnsupportedOperationException
skus.set(0, "SKU-CHANGED");      // succeeds — and MUTATES skuArray
```

**Exact symptom:** two separate ones.
- `add`/`remove` throw `UnsupportedOperationException` (fixed-size).
- `set` succeeds and writes **through to the backing array**. Elsewhere in your code,
  `skuArray[0]` has silently changed. The bug appears as a value that changed with no
  assignment anywhere near it.

And a third, worse, with primitives:
```java
int[] quantities = {1, 2, 3};
List<int[]> wrong = Arrays.asList(quantities);   // size 1! A List of ONE int[]
System.out.println(wrong.size());                // 1, not 3
```
Because `int` is not an object, the varargs parameter binds the whole array as a
single element (Topic 01 and Topic 06 together).

**Root cause:** `Arrays.asList` returns a fixed-size **view** over the array, not a
copy and not an immutable list.

**Fix:**
```java
List<String> skus = new ArrayList<>(Arrays.asList(skuArray));  // mutable copy
List<String> fixed = List.of(skuArray);                        // immutable copy
List<Integer> qty = Arrays.stream(quantities).boxed().toList(); // primitives
```

---

### Trap 4 — "unmodifiable" is a view, not a guarantee

**Wrong:**
```java
public class Order {
    private final List<OrderLine> lines = new ArrayList<>();

    public List<OrderLine> lines() {
        return Collections.unmodifiableList(lines);   // looks safe
    }
    void addLine(OrderLine l) { lines.add(l); }
}
```

**Exact symptom:** a caller holds the "unmodifiable" list, iterates it, and gets
`ConcurrentModificationException` — because another code path called `addLine` and
mutated the *backing* list underneath the view. The caller's stack trace shows them
iterating a list they were told was unmodifiable, and they will file a bug against
you.

**Root cause:** `Collections.unmodifiableList` returns a wrapper that forbids
modification **through that reference**. It does not freeze or copy anything. The
backing list is still fully mutable.

**Fix:** return a genuine immutable copy when the caller may hold it beyond the call.
```java
public List<OrderLine> lines() { return List.copyOf(lines); }
```
Note the trade honestly: `List.copyOf` allocates. For a hot path returning a large
collection, an unmodifiable view is the right call — but then document that it is
live, and never hand it to something that will hold it. Defensive copying in full is
Topic 17.

---

### Trap 5 — `List.remove(int)` versus `List.remove(Object)`

**Wrong:**
```java
List<Integer> productIds = new ArrayList<>(List.of(100, 200, 300));
productIds.remove(200);        // you meant "remove the value 200"
```

**Exact symptom:**
```
java.lang.IndexOutOfBoundsException: Index 200 out of bounds for length 3
```
or, worse, if the list happens to be long enough, **no exception at all** and the
wrong element is removed. In `orderflow`: you intended to remove product 200 from a
basket; you removed whatever was at position 200.

**Root cause:** `List` has two overloads — `remove(int index)` and
`remove(Object o)`. The literal `200` is an `int`, so the index overload wins. Java
prefers a match without boxing over one that requires it (Topic 01).

**Fix:**
```java
productIds.remove(Integer.valueOf(200));    // forces the Object overload
```
And the design fix: do not use `List` when you mean `Set`, and prefer `long`/`Long`
identifiers over `Integer` so this overload ambiguity cannot arise.

---

### Trap 6 — sharing a `HashMap` across threads

**Wrong:**
```java
private final Map<String, Integer> stockCache = new HashMap<>();   // shared by request threads
```

**Exact symptom — and note that it is *not* `ConcurrentModificationException`:**
`size()` disagrees with the number of entries you can iterate. A key you just `put`
returns `null` from `get`. Entries vanish. On older JDKs the classic symptom was a
thread spinning at 100% CPU forever inside `HashMap.get`, because concurrent resize
built a cycle in a bucket's linked list.

**Root cause:** `HashMap` has no synchronisation and no memory-model guarantees.
Concurrent `put` during a resize can lose entries or corrupt the table. `modCount`
would not save you here — it is not `volatile`, so the other thread's increment may
never even be visible.

> **Version note, stated precisely:** the infinite-loop-on-resize symptom was specific
> to Java 7's `transfer()` implementation, which reversed bucket order. Java 8
> restructured resize to preserve order and split each bin into a lo/hi list, which
> removes *that particular* cycle. `HashMap` is still **not thread-safe** on Java 8
> through 25 — you can still lose updates and observe torn state. Do not read "they
> fixed the infinite loop" as "it is safe now".

**Fix:** `ConcurrentHashMap`, and use its atomic methods rather than check-then-act.
```java
private final Map<String, Integer> stockCache = new ConcurrentHashMap<>();
stockCache.computeIfAbsent(sku, this::loadStock);   // atomic
```
Topics 92 and 97 cover why `computeIfAbsent` is atomic and why
`if (!map.containsKey(k)) map.put(k, v)` is not.

---

## Hands-on proof

Commands **you** run. I have no JVM and will not invent output.

### Setup

```bash
mkdir -p ~/java-lab/10 && cd ~/java-lab/10
java --version
```

### Proof 1 — the loud CME and the silent one

`CmeProof.java`:
```java
import java.util.*;

public class CmeProof {

    static void loud() {
        List<String> l = new ArrayList<>(List.of("a", "b", "c", "d"));
        try {
            for (String s : l) {
                if (s.equals("b")) l.remove(s);     // removes index 1 of 4
            }
            System.out.println("loud   : NO exception, result = " + l);
        } catch (ConcurrentModificationException e) {
            System.out.println("loud   : CME thrown, result = " + l);
        }
    }

    static void silent() {
        List<String> l = new ArrayList<>(List.of("a", "b", "c"));
        try {
            for (String s : l) {
                if (s.equals("b")) l.remove(s);     // removes SECOND-TO-LAST
            }
            System.out.println("silent : NO exception, result = " + l);
        } catch (ConcurrentModificationException e) {
            System.out.println("silent : CME thrown, result = " + l);
        }
    }

    public static void main(String[] args) { loud(); silent(); }
}
```

```bash
java CmeProof.java
```

**What to look for:** whether each method threw.

| What you see | What it means |
|---|---|
| `loud : CME thrown` | Expected. Removing element 1 of 4 leaves `cursor=1`, `size=3`, so `hasNext()` is true, `next()` runs, and the comodification check fires. |
| `silent : NO exception, result = [a, c]` | **This is the lesson.** Removing the second-to-last element leaves `cursor == size`, so `hasNext()` returns false and the loop exits before any check. Element `c` was never visited. The list is "correct" here but a real loop body would have skipped processing it. |
| Both throw | You are on a JDK whose `ArrayList` iterator differs, or you changed the list length. Print `l.size()` inside the loop to see the cursor arithmetic. |
| Neither throws | Check you are removing by object, not calling `removeIf`. |

**How to read it:** these two runs differ only by the length of the list. That is how
thin the guarantee is. Fail-fast detects most bugs of this shape and guarantees
nothing.

### Proof 2 — `modCount` is not `volatile`

`FailFastRace.java`:
```java
import java.util.*;
import java.util.concurrent.*;

public class FailFastRace {
    public static void main(String[] args) throws Exception {
        int cme = 0, wrong = 0, clean = 0;

        for (int trial = 0; trial < 200; trial++) {
            List<Integer> list = new ArrayList<>();
            for (int i = 0; i < 5_000; i++) list.add(i);

            CountDownLatch go = new CountDownLatch(1);
            Thread mutator = new Thread(() -> {
                try { go.await(); } catch (InterruptedException e) { return; }
                for (int i = 0; i < 5_000; i++) list.add(i);
            });
            mutator.start();
            go.countDown();

            try {
                long sum = 0;
                for (int v : list) sum += v;      // reading while another thread writes
                if (sum < 0) wrong++; else clean++;
            } catch (ConcurrentModificationException e) {
                cme++;
            } catch (RuntimeException e) {
                wrong++;                           // e.g. IndexOutOfBounds from a resize
            }
            mutator.join();
        }
        System.out.printf("CME=%d  other-failure=%d  completed-without-complaint=%d%n",
                cme, wrong, clean);
    }
}
```

```bash
java FailFastRace.java
java FailFastRace.java     # run it several times
```

**What to look for:** the three counts, and how much they vary between runs.

| What you see | What it means |
|---|---|
| A mix of CME, other failures, and clean completions | Exactly the point. The same racy program produces three different outcomes. Fail-fast caught *some* of them. |
| `completed-without-complaint` greater than zero | Some iterations read a concurrently-mutated list and reported nothing wrong. Those are the silent data bugs. |
| `other-failure` greater than zero (often `ArrayIndexOutOfBoundsException` or a null) | The iterator saw torn internal state. This is what "not thread-safe" actually looks like. |
| Numbers differ substantially between runs, or on a different machine | Expected and important: this is timing-dependent. **You cannot test your way to confidence here** — which is why the rule is "do not share a `HashMap`/`ArrayList` across threads", not "test it". |

> This program is deliberately racy and its output is non-deterministic by design. Do
> not tune it until it "works". The variability *is* the result.

### Proof 3 — immutable set/map iteration order is randomised per JVM run

`OrderSalt.java`:
```java
import java.util.*;

public class OrderSalt {
    public static void main(String[] args) {
        System.out.println("Set.of  : " + Set.of("a","b","c","d","e","f","g","h"));
        System.out.println("HashSet : " + new HashSet<>(List.of("a","b","c","d","e","f","g","h")));
        System.out.println("Linked  : " + new LinkedHashSet<>(List.of("a","b","c","d","e","f","g","h")));
    }
}
```

```bash
for i in 1 2 3 4 5; do java OrderSalt.java; done
```

**What to look for:** which lines change between the five runs.

| What you see | What it means |
|---|---|
| `Set.of` order differs between runs | Correct. Since Java 9 the immutable collections apply a per-JVM salt (`SALT` in `ImmutableCollections`) specifically to break any code that depends on their order. |
| `HashSet` order is the same every run but is not insertion order | `HashSet` order is a deterministic function of hash codes and table size — stable for the same inputs on the same JVM, but not a promise, and it changes when the table resizes. |
| `Linked` order is always insertion order | The only one of the three you may depend on. |
| `Set.of` order does not change | Possible on some builds — the salt is derived per JVM instance and the effect on small sets can be subtle. Increase the element count to 20 and retry before concluding anything. |

**How to read it:** if a test asserts on `Set.of(...).toString()`, it is a flaky test
waiting to happen, and the JDK is trying to tell you.

### Proof 4 — an unmodifiable view is not a copy

`ViewProof.java`:
```java
import java.util.*;

public class ViewProof {
    public static void main(String[] args) {
        List<String> backing = new ArrayList<>(List.of("SKU-1", "SKU-2"));
        List<String> view    = Collections.unmodifiableList(backing);
        List<String> copy    = List.copyOf(backing);

        backing.add("SKU-3");

        System.out.println("backing : " + backing);
        System.out.println("view    : " + view);
        System.out.println("copy    : " + copy);

        try { view.add("SKU-4"); }
        catch (UnsupportedOperationException e) {
            System.out.println("view.add threw UOE, message = " + e.getMessage());
        }
    }
}
```

```bash
java ViewProof.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `view` shows three elements, `copy` shows two | The "unmodifiable" list changed. It is a wrapper, not a snapshot. This is Trap 4 in two lines. |
| `message = null` | `UnsupportedOperationException` from the JDK's immutable collections carries no message. Remember the shape — a bare UOE in a stack trace means you hit an immutable or fixed-size collection. |
| `view` and `copy` both show two | You called `List.copyOf` on the view, or moved the `add`. Re-check the order of statements. |

### Proof 5 — the two `remove` overloads

`RemoveOverload.java`:
```java
import java.util.*;

public class RemoveOverload {
    public static void main(String[] args) {
        List<Integer> ids = new ArrayList<>(List.of(100, 200, 300));
        try {
            ids.remove(200);
            System.out.println("remove(200) ok, list = " + ids);
        } catch (IndexOutOfBoundsException e) {
            System.out.println("remove(200) threw: " + e.getMessage());
        }
        List<Integer> ids2 = new ArrayList<>(List.of(100, 200, 300));
        ids2.remove(Integer.valueOf(200));
        System.out.println("remove(Integer.valueOf(200)) -> " + ids2);
    }
}
```

```bash
java RemoveOverload.java
javap -c RemoveOverload.class | grep -n "remove"
```

**What to look for:**

| Where | What to look for | What it means |
|---|---|---|
| runtime | `IndexOutOfBoundsException: Index 200 out of bounds for length 3` | The `int` overload was selected. Overload resolution prefers no-boxing. |
| runtime, second call | `[100, 300]` | The `Object` overload removed by value. |
| `javap -c` | one call site with descriptor `(I)Ljava/lang/Object;` and one with `(Ljava/lang/Object;)Z` | The two overloads are visibly different methods in the bytecode. The compiler chose, not the runtime. |

### Proof 6 — confirm the fail-fast mechanism in the JDK source

This one is reading, not running, and it is worth ten minutes.

```bash
# Locate the sources bundle that ships with most JDKs
ls $JAVA_HOME/lib/src.zip
mkdir -p /tmp/jdksrc && cd /tmp/jdksrc && unzip -o "$JAVA_HOME/lib/src.zip" \
    'java.base/java/util/ArrayList.java' > /dev/null
grep -n "modCount" java.base/java/util/ArrayList.java | head -40
grep -n "checkForComodification" -A4 java.base/java/util/ArrayList.java
grep -n "public boolean hasNext" -A3 java.base/java/util/ArrayList.java
```

**What to look for:**

| What to look for | What it means |
|---|---|
| The declaration of `modCount` (it is inherited from `AbstractList`) and whether it has `volatile` | It does not. That single missing keyword is the entire "best-effort" caveat. |
| `checkForComodification` is called from `next()` and `remove()` | Not from `hasNext()`. That asymmetry is why Proof 1's silent case exists. |
| `hasNext()` is `cursor != size` | Confirms the arithmetic: remove the second-to-last element and `cursor == size` ends the loop early. |
| `src.zip` is missing | Some JDK distributions omit it. Read the same file at the OpenJDK GitHub mirror instead; the logic is identical. |

---

## Practice exercises

### 1 — Easy: choose from the contract

For each requirement, name the interface **and** the implementation, and give the one
sentence of contract that decides it. If two would work, say which you would pick and
why.

1. The lines of an order, as submitted, for display on an invoice.
2. The set of order statuses that count as "open", used in `if (open.contains(s))`.
3. A per-SKU running total during a 2M-row import.
4. The last 100 order IDs seen, for in-process deduplication, oldest evicted first.
5. Orders waiting to be retried, processed oldest-first.
6. Products sorted by price for a "cheapest first" page.
7. A lookup from `OrderStatus` (an enum) to a handler.
8. A configuration map built once at startup and read by many request threads.

Then: for numbers 2, 4 and 7, name a *later* topic that gives a better answer than
your first instinct, and say what it improves.

### 2 — Medium: combines Topics 01, 05, 06, 07 and 08

Write a `BatchDeduplicator<T>` with this API:

```java
public final class BatchDeduplicator<T> {
    public BatchDeduplicator(int maxTracked) { ... }
    public List<T> acceptAll(Collection<? extends T> incoming) { ... }
    public int rejectedCount() { ... }
}
```

Requirements:
- Returns only the items not seen before, **in arrival order**.
- Tracks at most `maxTracked` items; beyond that, the oldest tracked item is
  forgotten. (Do this with a plain `LinkedHashSet` and explicit eviction — Topic 15
  gives you the `LinkedHashMap` shortcut later.)
- The parameter must accept `List<OrderLine>`, `Set<OrderLine>` and
  `List<PhysicalLine>`. Justify the wildcard against Topic 07's rule in a comment.
- The returned list must be immutable, and you must state in a comment why you chose
  a copy rather than an unmodifiable view (Trap 4).
- Throws `IllegalArgumentException` (unchecked — justify against Topic 09) if
  `maxTracked < 1`.
- Must **not** throw `NullPointerException` if `incoming` contains a null. Decide what
  it should do instead and defend it.

Then write three tests: one proving the arrival order is preserved, one proving
eviction happens at exactly `maxTracked`, and one proving that mutating the input
collection after the call does not affect the result.

### 3 — Hard: production simulation on `orderflow`

**Part A — build the wrong version on purpose.** Implement `BatchIntake` from
Example 2, but using `ArrayList` for *everything*: `List` for `seenOrderIds`, `List`
for `distinctSkus`, and a `List<Map.Entry<String,Integer>>` instead of the map. Feed
it 50,000 lines across 5,000 distinct SKUs.

```bash
java -Xmx512m -Xlog:gc:file=wrong.log:time,uptime BatchIntake wrong 50000
```

Record the wall-clock time. Then run with 5,000, 10,000, 20,000 and 50,000 lines and
plot time against N. **State whether the curve is linear or superlinear, and derive
the reason from your own code** — do not guess, point at the specific line.

**Part B — build the right version** from Example 2 and repeat the same series.

```bash
java -Xmx512m -Xlog:gc:file=right.log:time,uptime BatchIntake right 50000
```

Report both curves. Then count young collections in each GC log
(`grep -c "Pause Young" wrong.log`) and say whether the difference came from
allocation or from algorithmic work. Be honest if you cannot tell from this evidence
alone, and say what evidence would settle it.

> **Warning, and it matters:** you are timing with a wall clock, which Topic 77 will
> show you is not a benchmark. The JIT has not warmed up, dead code may be eliminated,
> and GC noise is unbounded. This measurement is good enough to see an O(n²) curve —
> a shape you can see through noise — and not good enough for anything else. Do not
> quote these numbers in a design document.

**Part C — the CME hunt.** Introduce three bugs deliberately and record the exact
symptom of each:
1. A `for`-each over `accepted` that calls `accepted.remove(line)` — the loud CME.
2. The same, but arranged so the removed element is second-to-last — the silent skip.
   Prove the skip by counting processed lines, not by looking for an exception.
3. A second thread mutating `quantityBySku` while the main thread iterates it. Run it
   50 times and tabulate how many runs threw CME, how many threw something else, and
   how many completed with a wrong total.

**Part D — the API review.** `BatchResult` currently returns `List.copyOf(...)`.
A colleague proposes returning the internal `ArrayList` directly to avoid the copy,
arguing that callers "won't modify it". Write the review comment. Then write the
strongest version of *their* argument (there is one — name the condition under which
they are right), and say what you would need to see to agree.

---

## Interview questions

### Q1 — "What is `ConcurrentModificationException` and how do you avoid it?"

**Mid-level answer:** "It's thrown when you modify a collection while iterating it.
You avoid it by using `Iterator.remove()` or `removeIf`, or by collecting the items
to remove and removing them after the loop."

**Senior answer:** "Mechanically it's a `modCount` check: every structural
modification increments a counter on the collection, the iterator snapshots it at
creation, and `next()` compares them. The important part is that `modCount` is a plain
non-`volatile` `int`, so it is explicitly **best-effort** — the Javadoc says so. Three
consequences. First, across threads there is no happens-before edge, so the writer's
increment may never be visible to the reader; you get silently corrupt results instead
of an exception. Second, the single-threaded case can miss too: remove the
second-to-last element of an `ArrayList` in a for-each and `cursor` equals `size`, so
`hasNext()` returns false, the loop exits cleanly, and the last element is silently
skipped — no exception at all. Third, it is a **debugging aid**, so you must never
write code that relies on catching it. The fixes are `removeIf`, an explicit
`Iterator.remove`, or a concurrent collection with a weakly-consistent iterator if
there are genuinely multiple threads."

**What separates them:** knowing it is best-effort *and why*, and being able to
produce the single-threaded case where it silently does not fire. That second case is
the one that separates people who have read the source from people who have read a
blog post.

**Follow-up:** "So how would you detect the silent case?" A count of processed
elements versus `size()`, or an assertion in a test — not an exception handler.

---

### Q2 — "You need a collection of order statuses to check membership against. What do you use?"

**Mid-level answer:** "A `HashSet<String>` — `contains` is O(1)."

**Senior answer:** "If they are an enum — and they should be — `EnumSet`. It is a long
bitvector for up to 64 constants, so `contains` is a bit test with no hashing and no
per-element allocation, and it iterates in ordinal order for free. `HashSet<String>`
is the answer if they are genuinely strings from an external system, and then I would
use `Set.of(...)` so it is immutable and cannot be mutated by a caller. What I would
not do is a `List` — `contains` is a linear scan, which is invisible at three elements
and quadratic inside a loop over 50,000 order lines. The general rule is that I pick
from the contract: is the question 'is it in here?' or 'what is at position 3?'"

**What separates them:** naming `EnumSet` and its representation, and articulating the
selection rule rather than a preference.

**Follow-up:** "How much faster is `EnumSet`, and how would you know?" The honest
answer includes "I would measure with JMH before claiming a number" — Topic 77.

---

### Q3 — "What is the difference between `List.of(x)`, `Arrays.asList(x)`, and `Collections.unmodifiableList(x)`?"

**Mid-level answer:** "`List.of` is immutable, `Arrays.asList` is fixed-size, and
`unmodifiableList` wraps a list so you can't change it."

**Senior answer:** "Three genuinely different things. `List.of` is a true immutable
copy: all mutators throw `UnsupportedOperationException` with no message, it rejects
null elements at construction, and since Java 9 its iteration order for the `Set`/`Map`
variants is salted per JVM run specifically to break code that depends on it.
`Arrays.asList` returns a fixed-size **view over the array** — `add` and `remove`
throw, but `set` succeeds and writes through to the backing array, which is a real
aliasing bug source; and `Arrays.asList(intArray)` gives you a `List<int[]>` of size
one, because primitives don't box into varargs.
`Collections.unmodifiableList` is a **wrapper**, not a copy: it forbids modification
through that reference only, and the backing list can still change underneath —
so a caller holding it can get a `ConcurrentModificationException` from a list they
were told was unmodifiable. When I return a collection from a method I use
`List.copyOf`, and I accept the allocation, unless a profile tells me otherwise."

**What separates them:** the write-through behaviour of `Arrays.asList`, and knowing
that "unmodifiable" says nothing about the backing collection.

**Follow-up:** "When is the unmodifiable view the right choice?" Hot paths, large
collections, and short-lived callers — with documentation that it is live.

---

### Q4 — "Why is `Map` not a `Collection`?"

**Mid-level answer:** "Because a map holds pairs, not single elements."

**Senior answer:** "Because `Collection`'s contract is about single elements — `add(E)`,
`remove(Object)`, `contains(Object)`, `iterator()` — and none of those has a sensible
single meaning for a key-value pair. What would `Collection<Entry>` mean for
`contains`? Instead `Map` gives you three **views** — `keySet()`, `values()`,
`entrySet()` — each of which *is* a `Collection`, and each of which is live: removing
from `keySet()` removes from the map. That is a deliberate design that gives you
collection operations where they are meaningful without pretending a map is a bag of
entries. Practically it matters because those views are not copies: iterating
`entrySet()` and calling `setValue` writes through to the map and is *not* a
structural modification, so it does not trip fail-fast, whereas a `put` of a new key
inside the same loop does."

**What separates them:** framing it as a contract mismatch, and knowing the views are
live with the `setValue` nuance.

**Follow-up:** "So can you remove from a map while iterating it?" Yes — via
`entrySet().iterator().remove()` or `map.values().removeIf(...)`, not via `map.remove`.

---

### Q5 — "A `HashMap` is shared between request threads and entries are disappearing. Diagnose it."

**Mid-level answer:** "`HashMap` isn't thread-safe. Use `ConcurrentHashMap` or
synchronise access."

**Senior answer:** "Right conclusion, and the diagnosis is worth stating because the
symptom is unusual: you do **not** get `ConcurrentModificationException` here.
`modCount` is not `volatile`, so a writer's increment need not be visible to a reader
at all — there is no happens-before edge, so you get silently wrong state rather than
a loud failure. What you actually observe is `size()` disagreeing with what you can
iterate, a `get` returning null immediately after a successful `put`, or lost updates.
On Java 7 there was also a famous infinite loop in `get` caused by concurrent resize
building a cycle in a bucket; Java 8 restructured resize so *that specific* cycle can't
form, which people misread as 'HashMap is fine now'. It is not — it is still unsafe on
21 and 25. The fix is `ConcurrentHashMap` plus its **atomic** methods:
`computeIfAbsent`, `merge`, `putIfAbsent`. Swapping the class but keeping
`if (!map.containsKey(k)) map.put(k, v)` fixes nothing, because two atomic operations
are not one atomic operation."

**What separates them:** knowing the symptom is silence rather than an exception,
getting the Java 7-vs-8 history right without overclaiming, and flagging that
check-then-act does not become safe just because the map did.

**Follow-up:** "How would you prove it in a test?" You largely cannot — it is
timing-dependent. The honest answer is a stress harness that shows failures *sometimes*
plus a static rule that shared mutable collections must be concurrent types. That
leads to Topic 87 and jcstress.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `modCount` could have been made `volatile`, which would make fail-fast reliable
   across threads. It was not. What would that have cost, and on which operations?
   Was the trade correct?

2. `hasNext()` does not call `checkForComodification`, but `next()` does. Suppose it
   did call it. Which of Proof 1's two cases would change, and would any *correct*
   program break?

3. Since Java 9, `Set.of` randomises its iteration order per JVM run. That makes some
   tests flaky. Argue that this was the right decision anyway, then make the strongest
   case against it.

4. `Map` is not a `Collection`, but `Map.Entry` is not a `Collection` either and yet
   `entrySet()` is. Sketch a design where `Map extends Collection<Entry<K,V>>`. What
   specifically becomes ambiguous?

5. You are told "always return `List.copyOf` from getters". Name a concrete
   `orderflow` situation where following that rule would be a performance defect, and
   say what you would do instead and how you would document it.

6. JavaScript's `Map` and `Set` always preserve insertion order and Java's `HashMap`
   and `HashSet` do not. What does Java gain from not making that promise? Would you
   make the same choice designing a language today?

7. A colleague suggests wrapping every shared collection in
   `Collections.synchronizedList(...)` to make the codebase thread-safe. Explain, with
   a concrete example, why that does not achieve what they think it does.

---

## Quick reference card

### Choosing

```
Need positions or duplicates?           -> List        (ArrayList)
Need "is it in here?"                   -> Set         (HashSet / EnumSet)
Need key -> value?                       -> Map         (HashMap / EnumMap)
Need FIFO or LIFO?                       -> Deque       (ArrayDeque)
Need sorted order?                       -> TreeSet / TreeMap
Need insertion order in a Set/Map?       -> LinkedHashSet / LinkedHashMap
Need it shared across threads?           -> ConcurrentHashMap / CopyOnWriteArrayList
Never changes after construction?        -> List.of / Set.of / Map.of
```

### Null policy — memorise this, it is not uniform

| Allows null | Does not allow null |
|---|---|
| `ArrayList`, `LinkedList` | `List.of`, `Set.of`, `Map.of` (elements, keys and values) |
| `HashMap` (one null key, null values) | `TreeMap` keys, `TreeSet` elements |
| `HashSet` (one null) | `ArrayDeque`, `PriorityQueue` |
| `LinkedHashMap`, `LinkedHashSet` | `ConcurrentHashMap` (keys and values) |

### Iteration rules

- Enhanced `for` is an `Iterator` in disguise.
- The only legal structural modification during iteration is `Iterator.remove()`.
- `removeIf` is the one-liner and is preferred.
- `entry.setValue(v)` is **not** structural. `map.put(newKey, v)` is.
- `PriorityQueue` iteration is **not** in priority order. Only `poll()` is.
- `ConcurrentHashMap` and `CopyOnWriteArrayList` iterators never throw CME.

### Fail-fast in one paragraph

`modCount` is a plain `int` on the collection, incremented on structural change.
The iterator snapshots it as `expectedModCount` and compares in `next()` and
`remove()`. It is **best-effort**: not `volatile`, so cross-thread changes may be
invisible; `hasNext()` does not check, so removing the second-to-last element exits
the loop silently. Never write code that depends on catching it.

### API cheatsheet

```java
list.removeIf(pred);                 map.getOrDefault(k, def);
list.subList(a, b);                  map.putIfAbsent(k, v);
List.copyOf(c);                      map.computeIfAbsent(k, fn);
Collections.unmodifiableList(l);     map.merge(k, v, fn);
set.add(x)      // false if present  map.entrySet() / keySet() / values();
Collections.emptyList();             new ArrayList<>(expectedSize);
c.toArray(new T[0]);                 Collections.frequency(c, o);
```

---

## When would I use this at work?

**1. Every method signature you write.**
Parameter types should be the widest interface that works — `Collection<? extends T>`
rather than `ArrayList<T>` — and return types should say what the caller can rely on.
This is a five-second decision made hundreds of times, and it is the difference
between a codebase where implementations can change and one where they cannot.

**2. Reviewing a loop that touches a collection.**
Three greppable questions: does anything mutate the collection inside a for-each; is
there a `.contains()` on a `List` inside a loop; is a mutable internal collection being
returned. Those three cover most collection defects you will ever see in review, and
each takes seconds to check.

**3. Diagnosing a "data sometimes goes missing" bug.**
The shape — no exception, no error metric, a count that does not add up — points at
either a swallowed exception (Topic 09) or a collection misuse: a silently skipped
element from the fail-fast gap, or a shared `HashMap`. Knowing both shapes turns a
multi-day hunt into a targeted search.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and autoboxing**: collections hold objects only; `List<Integer>`
  boxes, and `remove(int)` vs `remove(Object)` is an overload trap that depends on it.
- **04 — Interfaces**: the whole framework is an argument for programming to
  interfaces, and `default` methods (`removeIf`, `forEach`) were added to
  `Collection` without breaking a single implementor.
- **05–07 — Generics**: every signature here is generic, and `addAll(Collection<?
  extends E>)` is PECS in the JDK.
- **08 — Exceptions**: `ConcurrentModificationException` and
  `UnsupportedOperationException` are both unchecked, and both are telling you about a
  defect rather than an event.

**This unlocks:**
- **11 — List implementations**: `ArrayList` vs `LinkedList` vs `ArrayDeque`, and why
  cache behaviour beats Big-O.
- **12 — HashMap internals**: what actually happens inside `get` and `put`.
- **13 — equals/hashCode**: the contract every `Set` and `Map` key depends on;
  breaking it makes elements vanish with no exception.
- **14 — Comparable/Comparator, TreeMap**: the sorted contracts.
- **15 — LinkedHashMap and LRU**: access order, and a cache in six lines.
- **16 — EnumSet/EnumMap**: the answer to Q2.
- **17 — Immutability and defensive copying**: Trap 4 in full.
- **23–24 — Streams and collectors**: `Collectors.toList` vs `stream().toList()`, and
  `groupingBy` replacing most of Example 2's loop.
- **92 / 97 — Concurrent collections**: weakly-consistent iterators, `ConcurrentHashMap`
  atomics, and why check-then-act still breaks.

---

*Java baseline 21. The framework has been stable since Java 8, with the immutable
factory methods arriving in Java 9 and `stream().toList()` in Java 16 — all available
on the 21 baseline. The one behaviour that has genuinely changed across versions is
`HashMap`'s resize implementation between Java 7 and 8, which removed a specific
infinite-loop symptom without making `HashMap` thread-safe. Sequenced collections
(`SequencedCollection`, `getFirst()`, `reversed()`) arrived in Java 21 and are worth
knowing exist; they add ordering methods to existing interfaces without changing any
of the contracts above.*
