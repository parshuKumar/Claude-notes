# 14 — Comparable vs Comparator; TreeMap, TreeSet, NavigableMap

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Topics 12 and 13 were about a cloakroom with numbered pegs. This one is about a
**bookshelf**.

On a bookshelf, nothing is hashed. Books sit in order. To find one you do not
compute a peg number — you walk to the middle, ask "is my book before or after
this one?", and repeat. About twenty questions gets you to any book among a
million. That is a sorted tree.

Two things follow immediately.

**First: you need a rule for "before or after".** Every book must be comparable
to every other book, and the answers must be consistent. If the rule says A is
before B, B is before C, and C is before A, the shelf cannot exist. There is no
place to put the books.

**Second — and this is the part people miss — the shelf has its own idea of
"the same book".** If two books compare as neither-before-nor-after, the shelf
concludes they are the same book and keeps only one. It never checks whether
they are actually identical. It has no way to. All it has is your before/after
rule.

So the cloakroom asks "same label?" (`equals`) and the shelf asks "neither before
nor after?" (`compareTo() == 0`). Those are two different questions, and when
your answers to them disagree, the shelf silently throws away books that the
cloakroom would have kept.

That, plus "what happens when your before/after rule is self-contradictory", is
the whole topic.

---

## The bridge from what you know

### What transfers

You sort arrays in TypeScript with a comparator function:

```ts
orders.sort((a, b) => a.total - b.total);              // ascending by total
orders.sort((a, b) => b.total - a.total);              // descending
orders.sort((a, b) => a.status.localeCompare(b.status));
```

Java's `Comparator` is the same idea with a name:

```java
orders.sort((a, b) -> Long.compare(a.total(), b.total()));
orders.sort(Comparator.comparingLong(Order::total));           // same thing, better
orders.sort(Comparator.comparingLong(Order::total).reversed());
```

The contract is identical: return negative if `a` comes first, zero if they tie,
positive if `b` comes first. Your instincts about writing comparators transfer
directly. **Verdict: HONEST ANALOGUE** for the comparator function itself.

### What does not transfer

| TypeScript | Java | Verdict |
|---|---|---|
| `arr.sort(fn)` — comparator passed per call | `Comparable` (built into the type) **or** `Comparator` (passed in) | **PARTIAL** — Java splits "the natural order of this type" from "an order for this call" |
| A bad comparator gives a weird order, silently | A bad comparator throws `IllegalArgumentException: Comparison method violates its general contract!` | **NO ANALOGUE** — Java detects some violations and refuses |
| `(a, b) => a.x - b.x` is idiomatic and fine | Subtraction overflows and is a real bug source | **PARTIAL** — `number` is float64 so subtraction is safe for you today; `int` is not |
| No sorted map/set in the standard library | `TreeMap`, `TreeSet`, `NavigableMap` | **NO ANALOGUE** — you have never had `floorEntry` or `subMap` |
| `Map` key equality is fixed | `TreeMap` key equality is **your comparator**, not `equals` | **NO ANALOGUE — and this is the trap** |
| `Array.prototype.sort` is stable (since ES2019) | `List.sort` / `Arrays.sort` on objects is stable (TimSort) | **HONEST ANALOGUE** |

That fifth row is where people from any dynamic-language background get hurt. In
JavaScript, a `Map` is a `Map`; you cannot change what "the same key" means. In
Java, moving a key type from `HashMap` to `TreeMap` **silently changes the
definition of key identity**, and it can merge entries that were previously
distinct. Nothing warns you.

---

## What is this?

Three related things.

### 1. `Comparable<T>` — a type's *natural* order

```java
public interface Comparable<T> {
    int compareTo(T other);
}
```

Implemented **by** the class. There is exactly one, and it is the order the type
means by default. `String` is alphabetical (by UTF-16 code unit), `Integer` is
numeric, `LocalDate` is chronological, an enum is by declaration order.

You implement it when your type has an obvious, uncontroversial order. If you
find yourself arguing about which order is "natural", it should be a
`Comparator` instead.

### 2. `Comparator<T>` — an order supplied at the call site

```java
@FunctionalInterface
public interface Comparator<T> {
    int compare(T a, T b);
    // plus ~20 default and static methods: comparing, thenComparing, reversed, ...
}
```

Passed **to** a sort, a `TreeMap`, a `PriorityQueue`. You can have as many as you
like. This is the direct equivalent of the function you pass to `.sort()` in
TypeScript, except that Java gives you a builder-style API for composing them.

### 3. `TreeMap` / `TreeSet` / `NavigableMap` — sorted collections

`TreeMap` is a **red-black tree**: a self-balancing binary search tree. Every
operation is O(log n), the entries are always in sorted order, and — because it
is sorted — you get an entire family of queries that a `HashMap` simply cannot
answer.

`TreeSet` is a `TreeMap` with dummy values, exactly as `HashSet` is a `HashMap`
with dummy values.

`NavigableMap` is the interface that exposes the order-aware queries:
`floorKey`, `ceilingKey`, `headMap`, `subMap`, `descendingMap`, and friends.
`TreeMap` implements it.

### The two contracts

**The `compareTo` / `compare` contract**, for all `x`, `y`, `z`:

| Clause | Meaning |
|---|---|
| **Antisymmetric** | `sgn(x.compareTo(y)) == -sgn(y.compareTo(x))` |
| **Transitive** | `x > y` and `y > z` implies `x > z` |
| **Substitutable** | `x.compareTo(y) == 0` implies `sgn(x.compareTo(z)) == sgn(y.compareTo(z))` for all `z` |
| **Throws consistently** | If `x.compareTo(y)` throws, so must `y.compareTo(x)` |

**And one that is "strongly recommended" but not required:**

> `(x.compareTo(y) == 0) == x.equals(y)`

That is called being **consistent with equals**. The Javadoc's exact phrasing is
worth internalising: it says a sorted set or map whose ordering is inconsistent
with equals "will behave strangely" and "violates the general contract of the
`Set`/`Map` interface, which is defined in terms of the `equals` method". It is
not a bug in `TreeMap`. It is `TreeMap` telling you, in advance, that it is going
to obey your comparator rather than your `equals`, and that if those disagree,
the resulting object is not really a `Map` any more.

---

## Why does it matter?

**1. `Arrays.sort` will throw at you, in production, from library code.**

`IllegalArgumentException: Comparison method violates its general contract!` is
one of the most confusing exceptions in Java. It comes from deep inside TimSort,
names none of your classes in the message, and — crucially — is **data
dependent**. Your comparator is broken from the day it was written and the
exception fires eighteen months later when the input finally has the shape that
exposes it.

**2. Moving a key from `HashMap` to `TreeMap` can silently merge entries.**

`new BigDecimal("1.0")` and `new BigDecimal("1.00")` are `compareTo`-equal and
`equals`-unequal. In a `HashSet` they are two elements. In a `TreeSet` they are
one. If those are two distinct price points, you have just lost one, with no
error.

**3. `NavigableMap` answers questions you currently write loops for.**

"What is the volume-discount tier for a quantity of 47?" is `floorEntry(47)` —
one O(log n) call. Coming from TypeScript you have never had this operation
available in a standard library, so you will not reach for it until you know it
exists. That is the main reason this section exists.

---

## Syntax breakdown

### Method references in comparator factories

```java
Comparator.comparing(Order::customerId)
```

`Order::customerId` is a **method reference** — shorthand for
`order -> order.customerId()`. It is the *unbound instance method* form: the
receiver becomes the first (and here only) argument. Four forms exist and they
are Topic 22; this is the one you will use constantly.

### The comparator builder API

```java
Comparator<Order> byTotalThenDate =
    Comparator.comparingLong(Order::totalMinor)      // primary key
              .thenComparing(Order::placedAt)        // tie-break
              .thenComparing(Order::id);             // final tie-break
```

| Method | What it does |
|---|---|
| `Comparator.comparing(fn)` | Extract a `Comparable` key and compare by it. **Boxes.** |
| `Comparator.comparingInt/Long/Double(fn)` | Same, but the extractor returns a primitive. **No boxing.** Prefer these |
| `Comparator.comparing(fn, cmp)` | Extract a key, compare it with a supplied comparator |
| `.thenComparing(...)` | Tie-break. Only consulted when everything before it returned 0 |
| `.reversed()` | Reverse **everything composed so far**. See Trap 3 |
| `Comparator.naturalOrder()` / `reverseOrder()` | The type's `Comparable` order, and its inverse |
| `Comparator.nullsFirst(cmp)` / `nullsLast(cmp)` | Wrap a comparator so it tolerates null *elements* |

### The `NavigableMap` query family

```java
NavigableMap<Integer, BigDecimal> tiers = new TreeMap<>();

tiers.floorEntry(47);        // greatest key <= 47
tiers.ceilingEntry(47);      // least key    >= 47
tiers.lowerEntry(47);        // greatest key <  47   (strictly)
tiers.higherEntry(47);       // least key    >  47   (strictly)

tiers.firstEntry();          // smallest
tiers.lastEntry();           // largest
tiers.pollFirstEntry();      // remove and return smallest

tiers.headMap(50, true);     // everything <= 50, as a live VIEW
tiers.tailMap(10, false);    // everything >  10
tiers.subMap(10, true, 50, false);   // [10, 50)
tiers.descendingMap();       // the whole map, reversed, as a live view
tiers.navigableKeySet();
tiers.descendingKeySet();
```

Two things about the `*Map` views that will surprise you:

1. **They are views, not copies.** Writing to a view writes through to the
   backing map, and changes to the backing map appear in the view. This is the
   opposite of `Array.prototype.slice()`.
2. **They enforce their range.** `subMap(10, 50).put(99, x)` throws
   `IllegalArgumentException: key out of range`. That is a feature — the view is
   a bounded map, not a filtered snapshot.

`[LEGACY — still asked]` The Java 5 forms `headMap(k)`, `tailMap(k)` and
`subMap(a, b)` still exist; they hardcode inclusive-low, exclusive-high. The
`NavigableMap` forms with explicit booleans arrived in Java 6 and are what you
should write.

---

## Example 1 — minimal

```java
import java.math.BigDecimal;
import java.util.*;

public class ConsistentWithEquals {
    public static void main(String[] args) {
        BigDecimal a = new BigDecimal("1.0");
        BigDecimal b = new BigDecimal("1.00");

        System.out.println("a.equals(b)     : " + a.equals(b));
        System.out.println("a.compareTo(b)  : " + a.compareTo(b));

        Set<BigDecimal> hash = new HashSet<>(List.of(a, b));
        Set<BigDecimal> tree = new TreeSet<>(List.of(a, b));

        System.out.println("HashSet size    : " + hash.size());
        System.out.println("TreeSet size    : " + tree.size());
        System.out.println("TreeSet content : " + tree);
    }
}
```

Run it. `equals` says false, `compareTo` says 0, and the two sets disagree about
how many elements you have.

`BigDecimal` is the JDK's own, documented example of a class deliberately
inconsistent with `equals` — its `equals` compares scale as well as value,
because `1.0` and `1.00` are genuinely different in accounting terms, while its
`compareTo` compares numeric value only. Both are correct for their purpose.
The consequence is that `BigDecimal` behaves differently in a `TreeSet` than in a
`HashSet`, and the Javadoc says so explicitly.

Which one is "right" depends on your domain. Which one you *get* depends on which
collection someone picked, possibly for unrelated reasons, possibly years ago.

---

## Example 2 — production scenario

`orderflow` has volume-based pricing. Buy 1–9 units and you pay full price; 10–49
gets 5% off; 50–199 gets 12%; 200+ gets 20%. There is also an admin screen that
lists orders sorted by status priority, then newest first, then by ID.

### The version that ships and is wrong twice

```java
public class PricingService {

    private static final List<Tier> TIERS = List.of(
        new Tier(1,   BigDecimal.ZERO),
        new Tier(10,  new BigDecimal("0.05")),
        new Tier(50,  new BigDecimal("0.12")),
        new Tier(200, new BigDecimal("0.20"))
    );

    record Tier(int minQuantity, BigDecimal discount) { }

    public BigDecimal discountFor(int quantity) {
        BigDecimal found = BigDecimal.ZERO;
        for (Tier t : TIERS) {                       // linear scan, order-dependent
            if (quantity >= t.minQuantity()) {
                found = t.discount();
            }
        }
        return found;
    }
}
```

```java
public class OrderAdminView {

    public List<Order> forDashboard(List<Order> orders) {
        List<Order> copy = new ArrayList<>(orders);
        copy.sort((a, b) -> {
            int byStatus = a.status().priority() - b.status().priority();   // <-- defect
            if (byStatus != 0) return byStatus;
            return (int) (b.placedAt().toEpochMilli() - a.placedAt().toEpochMilli());  // <-- defect
        });
        return copy;
    }
}
```

Four problems.

**Pricing, problem one:** the lookup is a linear scan whose correctness depends
on `TIERS` being in ascending order. Nothing enforces that. Someone inserts a new
tier in the wrong place during a promo and the wrong discount ships. There is no
test that fails, because the test data was written from the same list.

**Pricing, problem two:** it is O(n) per quote. With four tiers that is fine.
With four hundred customer-specific tiers on a hot path it is not, and by then
the code will have been copied into three other services.

**Sorting, problem one:** `a.priority() - b.priority()` is **integer
subtraction**. If priorities are small it works. If anyone ever uses
`Integer.MIN_VALUE` as a sentinel, or a computed score near the int limits, the
subtraction overflows and the sign flips. Now the comparator claims `a < b` and
`b < a` simultaneously.

**Sorting, problem two:** the epoch-millis subtraction is a `long` cast down to
`int`. Two timestamps more than about 24.8 days apart produce a difference that
does not fit in an `int`, and the cast silently truncates — frequently flipping
the sign. This one is guaranteed to happen on any dataset older than a month.

The observable symptom of both sorting defects is the same, and it is famous:

```
java.lang.IllegalArgumentException: Comparison method violates its general contract!
    at java.base/java.util.TimSort.mergeLo(TimSort.java:781)
    at java.base/java.util.TimSort.mergeAt(TimSort.java:518)
    ...
    at java.base/java.util.Arrays.sort(Arrays.java:1307)
    at java.base/java.util.List.sort(List.java:507)
    at com.orderflow.admin.OrderAdminView.forDashboard(OrderAdminView.java:9)
```

Notice what the message does **not** say: it does not name your comparator, it
does not name the elements involved, and it does not tell you which clause you
broke. The first three frames are all JDK internals. This is why the exception is
so widely misdiagnosed as a JDK bug.

Notice also that it fires only sometimes. TimSort only merges runs when the array
is at least 32 elements; below that it uses a plain binary insertion sort that
never detects anything. So a broken comparator on a small dashboard **silently
produces a wrong order**, and the same code on a bigger dataset throws. Both are
the same bug.

### The corrected version

```java
public class PricingService {

    // Sorted by construction. The data structure enforces the invariant.
    private static final NavigableMap<Integer, BigDecimal> TIERS = new TreeMap<>(Map.of(
        1,   BigDecimal.ZERO,
        10,  new BigDecimal("0.05"),
        50,  new BigDecimal("0.12"),
        200, new BigDecimal("0.20")
    ));

    public BigDecimal discountFor(int quantity) {
        Map.Entry<Integer, BigDecimal> tier = TIERS.floorEntry(quantity);
        return tier == null ? BigDecimal.ZERO : tier.getValue();
    }
}
```

```java
public class OrderAdminView {

    private static final Comparator<Order> DASHBOARD_ORDER =
        Comparator.comparingInt((Order o) -> o.status().priority())
                  .thenComparing(Order::placedAt, Comparator.reverseOrder())
                  .thenComparing(Order::id);

    public List<Order> forDashboard(List<Order> orders) {
        return orders.stream().sorted(DASHBOARD_ORDER).toList();
    }
}
```

What each change buys:

1. **`TreeMap` + `floorEntry`** — the tier lookup is O(log n), and the ordering
   invariant is maintained by the data structure rather than by whoever edits
   the list next. `floorEntry` means "the greatest key less than or equal to
   this", which is exactly the definition of a pricing tier.
2. **`comparingInt`** — no subtraction, so no overflow. Internally it uses
   `Integer.compare`, which is branch-based, not arithmetic. Also no boxing
   (Topic 01), because the extractor returns a primitive.
3. **`thenComparing(Order::placedAt, Comparator.reverseOrder())`** — reverses
   *only* the date, which is what was wanted. Writing `.reversed()` at the end
   would have reversed the status priority too. See Trap 3.
4. **`.thenComparing(Order::id)`** — a total tie-break. Without a final
   discriminator, two orders with the same status and timestamp can appear in
   either order, and the dashboard shuffles between page loads. Adding a unique
   final key makes the sort deterministic.

> `floorEntry` returns `null` when the quantity is below the lowest tier. The
> null check is not defensive padding — it is handling a real case (quantity 0,
> or a tier table that does not start at 1). `Optional.ofNullable(...)` is an
> alternative; Topic 26 covers when that is worth it.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — subtraction in a comparator

**Wrong:**
```java
Comparator<Order> byTotal = (a, b) -> (int) (a.totalMinor() - b.totalMinor());
Comparator<Product> byStock = (a, b) -> a.stock() - b.stock();
```

**Exact symptom:** on small inputs, a silently wrong order. On inputs of 32 or
more elements:
```
java.lang.IllegalArgumentException: Comparison method violates its general contract!
    at java.base/java.util.TimSort.mergeLo(...)
```
Data-dependent, so it appears months after the code was written, usually during a
traffic spike when a page size grows.

**Root cause:** `int` subtraction overflows. `5 - Integer.MIN_VALUE` is not a
large positive number; it wraps to a negative one. A `long` cast to `int` simply
discards the high bits. Either way the sign is wrong for some pairs, so
antisymmetry and transitivity both break, and TimSort's merge invariant fails.

**Fix:**
```java
Comparator<Order> byTotal = Comparator.comparingLong(Order::totalMinor);
Comparator<Product> byStock = Comparator.comparingInt(Product::stock);
// or, explicitly:
(a, b) -> Long.compare(a.totalMinor(), b.totalMinor());
```
`Integer.compare` and `Long.compare` are implemented with comparisons, not
arithmetic. They cannot overflow. **Never write subtraction in a comparator.**
Treat it as a hard rule, not a guideline — the exceptions are not worth the
cognitive load of checking.

---

### Trap 2 — a comparator inconsistent with `equals`, in a `TreeMap`

**Wrong:**
```java
// Case-insensitive SKU index
Map<String, Product> bySku = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
bySku.put("SKU-4471", widget);
bySku.put("sku-4471", gadget);        // silently REPLACES widget
```

**Exact symptom:** `bySku.size()` is 1 when you inserted 2. No exception, no
warning. In a `HashMap` the same two puts give you 2 entries. Somebody
"optimises" a `HashMap` to a `TreeMap` to get sorted output and quietly loses
half the catalogue.

Related and nastier:
```java
Set<BigDecimal> prices = new TreeSet<>(List.of(
    new BigDecimal("19.99"), new BigDecimal("19.990")));
prices.size();       // 1
```

**Root cause:** `TreeMap` never calls `equals`. Not once, anywhere in `put`,
`get`, `containsKey` or `remove`. It determines key identity **entirely** by
`compare(a, b) == 0`. The `Map` interface's contract is defined in terms of
`equals`, so a `TreeMap` with an inconsistent comparator is, by the Javadoc's own
words, not honouring the `Map` contract.

**Fix, choose one:**
- Make the comparator consistent with `equals` — add a final tie-break on
  something unique, so nothing ties unless it is genuinely the same key.
- Normalise before insertion: store the SKU uppercased, and uppercase on lookup
  too. Now `equals` and `compareTo` agree because there is only one form.
- Use a `LinkedHashMap` (Topic 15) if what you actually wanted was predictable
  iteration rather than sorting.

**Diagnostic that settles it:**
```java
assert (a.compareTo(b) == 0) == a.equals(b) : "comparator inconsistent with equals";
```
Run with `-ea`. If you cannot make that hold, you know which collection you are
allowed to use.

---

### Trap 3 — `.reversed()` reverses the whole chain

**Wrong:**
```java
Comparator<Order> c = Comparator.comparing(Order::status)
                                .thenComparing(Order::placedAt)
                                .reversed();     // intended: newest first
```

**Exact symptom:** the list is sorted by status in *descending* order too. Nothing
throws. A dashboard that is supposed to show `PENDING` first shows `SHIPPED`
first, and the bug report is "the sorting is wrong" with no further detail.

**Root cause:** `.reversed()` is a method on the *composed* comparator, not on
the last step. `a.thenComparing(b).reversed()` is `reverse(a then b)`, not
`a then reverse(b)`. The fluent syntax reads left to right; the semantics do not.

**Fix — reverse the specific key:**
```java
Comparator<Order> c = Comparator.comparing(Order::status)
                                .thenComparing(Order::placedAt, Comparator.reverseOrder());
```

The two-argument `thenComparing(keyExtractor, keyComparator)` is the form you
want and the one most people do not know exists. Same for
`Comparator.comparing(fn, cmp)` at the head of a chain.

A quick way to remember it: `.reversed()` applies to everything to its **left**.
If that is not what you want, do not use it.

---

### Trap 4 — nulls

**Wrong:**
```java
TreeMap<String, Product> bySku = new TreeMap<>();
bySku.put(null, widget);                      // NullPointerException

Comparator<Order> c = Comparator.comparing(Order::couponCode);   // may be null
orders.sort(c);                               // NullPointerException, mid-sort
```

**Exact symptom:** for the map, an immediate NPE from `TreeMap.put`, with
`compare` in the stack. For the sort, an NPE thrown from inside `TimSort` after
an unpredictable number of comparisons — so it is data-dependent and may appear
only when a particular record has a null coupon.

Worse: an NPE from a partially-completed sort can leave your `List` in a
**partially reordered state**, because `List.sort` writes back as it goes.

**Root cause:** `TreeMap` with natural ordering calls `compareTo` on the key, and
`null.compareTo(...)` throws. Even with a comparator, `TreeMap.put` on an empty
map calls `compare(key, key)` as a type check, so your comparator sees the null
regardless. And `Comparator.comparing` calls the key extractor and then
`compareTo` on the result — null extracted key, same NPE.

**Fix:**
```java
Comparator<Order> c = Comparator.comparing(
        Order::couponCode,
        Comparator.nullsLast(Comparator.naturalOrder()));
```
And for the map: `HashMap` allows one null key, `TreeMap` and
`ConcurrentHashMap` allow none. If null keys are meaningful in your domain, they
usually should not be — but if they are, `HashMap` is the only choice, and that
constraint should be written down.

---

### Trap 5 — `PriorityQueue` iteration is not sorted

**Wrong:**
```java
Queue<Order> urgent = new PriorityQueue<>(Comparator.comparingInt(Order::priority));
urgent.addAll(orders);
for (Order o : urgent) {            // <-- not in priority order
    dispatch(o);
}
```

**Exact symptom:** orders are dispatched in an order that looks *almost* sorted —
the first one is always correct, the rest are not. The bug survives review
because a test with three elements happens to come out right.

**Root cause:** `PriorityQueue` is a binary **heap** in an array. A heap
guarantees only that the root is the minimum. Its iterator walks the backing
array in index order, which is heap order, not sorted order. `Queue`'s contract
never promised otherwise; only `poll()` returns elements in priority order.

**Fix:**
```java
while (!urgent.isEmpty()) {
    dispatch(urgent.poll());        // draining IS the sorted traversal
}
// or, if you need to keep the collection:
orders.stream().sorted(Comparator.comparingInt(Order::priority)).forEach(this::dispatch);
```

Related: `PriorityQueue.remove(Object)` and `contains(Object)` use `equals`, not
the comparator, and are O(n). So `PriorityQueue` is a third collection with yet
another equality story. Know which one you are in.

---

## Hands-on proof

Everything here is a command **you** run. I do not have a JVM and will not
fabricate output.

### Setup

```bash
mkdir -p ~/java-lab/14 && cd ~/java-lab/14
java --version
```

### Proof 1 — produce the contract-violation exception on demand

`BreakTimSort.java`:
```java
import java.util.*;

public class BreakTimSort {
    record Item(int score) { }

    public static void main(String[] args) {
        int n = args.length > 0 ? Integer.parseInt(args[0]) : 64;

        // Scores spread across the full int range, so subtraction overflows.
        Random rnd = new Random(42);            // fixed seed: reproducible
        List<Item> items = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            items.add(new Item(rnd.nextInt()));  // full range, including negatives
        }

        Comparator<Item> broken = (a, b) -> a.score() - b.score();   // overflows
        try {
            items.sort(broken);
            System.out.println("n=" + n + " : no exception. Sorted? " + isSorted(items));
        } catch (IllegalArgumentException e) {
            System.out.println("n=" + n + " : " + e.getMessage());
        }

        Collections.shuffle(items, new Random(42));
        items.sort(Comparator.comparingInt(Item::score));            // correct
        System.out.println("n=" + n + " : correct comparator, sorted? " + isSorted(items));
    }

    static boolean isSorted(List<Item> l) {
        for (int i = 1; i < l.size(); i++)
            if (l.get(i - 1).score() > l.get(i).score()) return false;
        return true;
    }
}
```

```bash
java BreakTimSort.java 8
java BreakTimSort.java 64
java BreakTimSort.java 5000
```

| What you see | What it means |
|---|---|
| `n=8 : no exception. Sorted? false` | Confirmed and important: below 32 elements TimSort uses binary insertion sort and never checks the invariant. The comparator is equally broken; it just fails **silently** |
| `n=64` or `n=5000` printing `Comparison method violates its general contract!` | Confirmed: you produced the famous exception on demand, from a comparator that looks completely ordinary |
| `n=64 : no exception` but `Sorted? false` | Possible. The check is a heuristic on merge runs, not a proof. Raise `n` and re-run. **Absence of the exception is not evidence the comparator is correct** |
| The correct comparator always prints `sorted? true` | Confirmed: `comparingInt` uses `Integer.compare`, which cannot overflow |

That third row is the most important one in this doc. The exception detects
*some* violations, not all. A comparator that never throws can still be wrong.

### Proof 2 — the legacy escape hatch, and why it is not a fix

There is a system property that reverts `Arrays.sort` to the pre-Java-7 merge
sort, which does not perform the check.

```bash
java -Djava.util.Arrays.useLegacyMergeSort=true BreakTimSort.java 5000
```

| What you see | What it means |
|---|---|
| No exception, and `Sorted? false` | Confirmed: the flag suppresses the *detection*, not the *defect*. The order is still wrong |
| The exception still fires | The property may have been removed or renamed in your JDK. That is version-specific information worth knowing — see the note below |

> **Genuine uncertainty, stated plainly:** this property has existed since Java 7
> and I believe it is still honoured on 21 and 25, but I cannot verify it without
> a JVM and it is exactly the kind of legacy switch that gets removed. The
> command above settles it in five seconds. `[LEGACY — still asked]` —
> interviewers ask about it precisely to see whether you volunteer that using it
> is hiding a bug rather than fixing one.

### Proof 3 — TreeMap ignores equals

`TreeVsHash.java`:
```java
import java.math.BigDecimal;
import java.util.*;

public class TreeVsHash {
    public static void main(String[] args) {
        List<BigDecimal> prices = List.of(
            new BigDecimal("19.99"),
            new BigDecimal("19.990"),
            new BigDecimal("19.9900"));

        System.out.println("HashSet : " + new HashSet<>(prices).size() + " -> " + new HashSet<>(prices));
        System.out.println("TreeSet : " + new TreeSet<>(prices).size() + " -> " + new TreeSet<>(prices));

        Map<String, String> ci = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
        ci.put("SKU-1", "widget");
        ci.put("sku-1", "gadget");
        System.out.println("case-insensitive TreeMap : " + ci);

        Map<String, String> hm = new HashMap<>();
        hm.put("SKU-1", "widget");
        hm.put("sku-1", "gadget");
        System.out.println("HashMap                  : " + hm);
    }
}
```

```bash
java TreeVsHash.java
```

| What you see | What it means |
|---|---|
| `HashSet : 3`, `TreeSet : 1` | Confirmed: `equals` distinguishes scale, `compareTo` does not. Same data, two different answers |
| `TreeMap` shows one entry, and the value is `gadget` | Confirmed: the second `put` was treated as a replacement. Note the *key* stays as first inserted — `put` replaces the value, not the key |
| `HashMap` shows two entries | Confirmed: `HashMap` uses `equals`, which is case-sensitive for `String` |
| The `TreeMap` key printed is `sku-1` | Would mean `put` replaced the key too. It should not — check your JDK and report it, because that would contradict the documented behaviour |

That last row is worth thinking about: when a `TreeMap.put` matches an existing
key, the **old key object is kept** and only the value is replaced. So the
surviving key is the one inserted first. In a case-insensitive index, the casing
you get back is whichever casing arrived first, which is almost never what anyone
intended.

### Proof 4 — the NavigableMap query family

`Tiers.java`:
```java
import java.math.BigDecimal;
import java.util.*;

public class Tiers {
    public static void main(String[] args) {
        NavigableMap<Integer, BigDecimal> tiers = new TreeMap<>();
        tiers.put(1,   new BigDecimal("0.00"));
        tiers.put(10,  new BigDecimal("0.05"));
        tiers.put(50,  new BigDecimal("0.12"));
        tiers.put(200, new BigDecimal("0.20"));

        for (int q : new int[] { 0, 1, 9, 10, 47, 50, 199, 200, 10_000 }) {
            System.out.printf("qty %-6d floor=%-14s ceiling=%-14s%n",
                q, tiers.floorEntry(q), tiers.ceilingEntry(q));
        }

        System.out.println("headMap(50, true)  = " + tiers.headMap(50, true));
        System.out.println("subMap(10,true,200,false) = " + tiers.subMap(10, true, 200, false));
        System.out.println("descending         = " + tiers.descendingMap());

        // Views write through, and enforce their range.
        NavigableMap<Integer, BigDecimal> mid = tiers.subMap(10, true, 200, false);
        mid.put(100, new BigDecimal("0.15"));
        System.out.println("after view put, backing map = " + tiers);
        try {
            mid.put(500, BigDecimal.ONE);
        } catch (IllegalArgumentException e) {
            System.out.println("out-of-range put rejected: " + e.getMessage());
        }
    }
}
```

```bash
java Tiers.java
```

| What you see | What it means |
|---|---|
| `qty 0` shows `floor=null` | Confirmed: no key is <= 0. This is the null case your production code must handle |
| `qty 47` shows `floor=10=0.05` | Confirmed: `floorEntry` is exactly "the tier this quantity falls into" |
| `qty 50` shows `floor=50=0.12` | Confirmed: `floor` is inclusive. `lowerEntry(50)` would give `10` |
| The backing map contains key 100 after the view `put` | Confirmed: views are live, not snapshots. This is unlike anything in JS |
| `out-of-range put rejected: key out of range` | Confirmed: a sub-map view is a bounded map and enforces its bounds |

### Proof 5 — read the Javadoc that defines the trap

The "consistent with equals" language is not folklore. Read it in the source.

```bash
unzip -o "$JAVA_HOME/lib/src.zip" \
  'java.base/java/lang/Comparable.java' \
  'java.base/java/util/Comparator.java' \
  'java.base/java/util/TreeMap.java' \
  'java.base/java/util/SortedMap.java' \
  -d ~/java-lab/14/src

grep -n -A6 "consistent with equals" ~/java-lab/14/src/java.base/java/util/SortedMap.java
grep -n -B2 -A12 "strongly recommended" ~/java-lab/14/src/java.base/java/lang/Comparable.java
```

| What you see | What it means |
|---|---|
| `SortedMap` saying a map with an inconsistent comparator "violates the general contract of the `Map` interface" | Confirmed: this is documented behaviour, not a bug. `TreeMap` warned you |
| `Comparable` calling consistency with `equals` "strongly recommended" but not required | Confirmed: it is a recommendation precisely because `BigDecimal` needed to break it |
| The `BigDecimal` example cited in `Comparable`'s own Javadoc | Confirmed: the JDK uses its own class as the cautionary tale |

Then read `TimSort` itself if you want the mechanism behind the exception:
```bash
unzip -p "$JAVA_HOME/lib/src.zip" 'java.base/java/util/TimSort.java' \
  | grep -n -B15 "Comparison method violates"
```
**What to look for:** the exception is thrown from `mergeLo`/`mergeHi` when the
merge runs off the end of a run — the algorithm detected that its invariant was
false. It is a **consistency check inside the merge**, which is why it is a
heuristic and why it needs at least 32 elements (TimSort's `MIN_MERGE`) to have
anything to merge.

---

## Practice exercises

### 1 — Easy: build a comparator chain and get `reversed()` right

Given a `record Order(long id, OrderStatus status, Instant placedAt, long totalMinor)`,
write four comparators:

1. By total, ascending.
2. By total, descending.
3. By status ascending, then total **descending**, then id ascending.
4. By customer email, case-insensitive, nulls last, then by id.

Requirements:
- No subtraction anywhere.
- Comparator 3 must **not** use `.reversed()` at the end of the chain. Prove it
  is right by sorting a list where the naive `.reversed()` version differs, and
  print both.
- For comparator 4, feed it a list containing at least one null email and show it
  does not throw.

### 2 — Medium: the audit (combines Topics 01, 05, 07, 10, 12, 13)

Here is an `orderflow` reporting service with **six** defects from Topics 01–14.
Find them all, state the exact observable symptom of each, and rewrite it.

```java
public class OrderReport {

    private final Map<Product, Integer> unitsSold = new TreeMap<>(
            (a, b) -> a.name().compareToIgnoreCase(b.name()));

    public void record(Product p, Integer qty) {
        Integer current = unitsSold.get(p);
        unitsSold.put(p, current + qty);
    }

    public List<Product> topSellers(int limit) {
        List<Product> all = new ArrayList<>(unitsSold.keySet());
        all.sort((a, b) -> unitsSold.get(b) - unitsSold.get(a));
        return all.subList(0, limit);
    }

    public Product cheapestAbove(double minPrice) {
        TreeSet<Product> byPrice = new TreeSet<>(
                Comparator.comparingDouble(Product::price));
        byPrice.addAll(unitsSold.keySet());
        return byPrice.higher(new Product("", "", minPrice));
    }
}
```

Hints: one defect from Topic 01, one from Topic 12 or 13, three from Topic 14,
and one that throws `IndexOutOfBoundsException` on a small dataset. Say for each
whether it throws, silently corrupts, or only shows up at scale.

### 3 — Hard: production simulation

Build the `orderflow` volume-pricing engine and prove the `TreeMap` version is
both correct and faster than the list scan.

**Part A — correctness.** Implement `discountFor(int quantity)` twice: once with
the linear list scan from Example 2, once with `NavigableMap.floorEntry`.
Generate 400 tiers with random ascending boundaries. Assert that both
implementations agree for every quantity from 0 to 100,000. Then **shuffle the
list** in the linear version and re-run the assertion. Report what happens and
explain why the `TreeMap` version was unaffected.

**Part B — the contract violation, deliberately.** Write a `Comparator<Tier>`
that compares by `discount` with a tolerance of 0.001 ("tiers within a tenth of
a percent are the same tier"). Sort 5,000 tiers with it. Capture the exception.
Then explain, in one paragraph, why **no** `hashCode` could make this comparator
consistent with `equals` — connect it to Topic 13's Trap 4.

**Part C — the TreeMap identity trap, deliberately.** Build a
`TreeMap<Tier, BigDecimal>` keyed by a comparator that compares only
`minQuantity`. Insert two `Tier` objects with the same `minQuantity` and
different discounts. Show that:
- `map.size()` is 1;
- the surviving *key* is the first inserted;
- the surviving *value* is the second inserted;
- the equivalent `HashMap` has 2 entries.

Write down which of those four facts surprised you.

**Part D — measure it.** With 400 tiers and 10 million lookups, compare the two
implementations under Flight Recorder:
```bash
java -XX:StartFlightRecording=duration=60s,filename=scan.jfr ScanRun
java -XX:StartFlightRecording=duration=60s,filename=tree.jfr TreeRun
```
Report where the CPU goes in each. Then argue the other side: at what tier count
does the linear scan actually win, and why? (Think about branch prediction and
cache lines from Topic 11 — the answer is not "never".)

---

## Interview questions

### Q1 — "`Comparable` or `Comparator`? When do you pick each?"

**Mid-level answer:** "`Comparable` is the natural ordering, implemented on the
class. `Comparator` is a custom ordering you pass in. Use `Comparable` if there's
one obvious order and `Comparator` otherwise."

**Senior answer:** "Same distinction, but the deciding question I actually ask
is: *is this order a property of the type, or a property of this use case?*
`LocalDate` is chronological — that is a fact about dates, so it is `Comparable`.
`Order` sorted by dashboard priority is a fact about the admin screen, so it is a
`Comparator` that lives next to the screen. The failure mode of getting it wrong
is that a `Comparable` implementation becomes a de-facto API: it silently
determines what `TreeSet`, `PriorityQueue`, `Collections.sort` and
`Stream.sorted()` do, from every call site in the codebase, and changing it later
is a breaking change you cannot see in any signature. So my bias is strongly
toward `Comparator` unless the ordering is genuinely intrinsic and stable. And
if I do implement `Comparable`, I make it consistent with `equals`, because
otherwise I have made every sorted collection in the codebase behave oddly and
nobody will connect the two."

**What separates them:** framing it as "property of the type vs property of the
use case", and recognising that `Comparable` is an implicit API surface with
compatibility obligations.

**Follow-up:** "Can a class implement `Comparable` for more than one type?"
Technically `Comparable<A>` and `Comparable<B>` cannot both be implemented —
erasure forbids it (Topic 06). That connection is the answer they want.

---

### Q2 — "What is `Comparison method violates its general contract!` and how do you fix it?"

**Mid-level answer:** "Your comparator is inconsistent. Usually it's subtraction
overflowing — use `Integer.compare` instead."

**Senior answer:** "It comes from TimSort, which is Java's sort for object arrays
since 7. TimSort finds ascending and descending runs and merges them, and the
merge relies on the comparator being a genuine total order. When a merge runs off
the end of a run, that invariant has been proven false and it throws rather than
silently corrupting the array. Three things follow that matter. First, the cause
is almost always subtraction overflow — `a.x - b.x` for ints near the limits, or
a `long` difference cast to `int` — and the fix is `Integer.compare` /
`Comparator.comparingInt`, which compare rather than subtract. Second, it is a
**heuristic**: it detects some violations, not all, so a comparator that has
never thrown is not thereby correct. Third, and the one people get wrong: below
32 elements TimSort uses binary insertion sort and does no merging, so the same
broken comparator produces a silently wrong order instead of an exception. That
means the exception appearing is actually the *good* outcome — it means the bug
became visible. The other causes worth naming are non-transitive 'equals within a
tolerance' comparators, comparators that read mutable state, and a comparator
that is fine but whose input is being mutated concurrently during the sort."

**What separates them:** knowing it is a heuristic, knowing about the 32-element
threshold, and reframing the exception as detection rather than as the problem.

**Follow-up:** "Someone sets `-Djava.util.Arrays.useLegacyMergeSort=true` and the
exception goes away. What do you say in review?" They want you to say the data is
still being sorted wrongly and the flag has only removed the detector — and
ideally to note that this is the same shape as "fix the failing test by deleting
the assertion".

---

### Q3 — "Why does a comparator inconsistent with `equals` break `TreeMap` but not `HashMap`?"

**Mid-level answer:** "`HashMap` uses `hashCode` and `equals`; `TreeMap` uses the
comparator. So they disagree about which keys are the same."

**Senior answer:** "Because they answer 'is this the same key?' with different
methods. `HashMap` computes a bucket from `hashCode` and confirms with `equals`;
`TreeMap` navigates the tree and treats `compare(a, b) == 0` as identity. It
never calls `equals` — not in `put`, `get`, `containsKey` or `remove`. So the
same key type in the two maps can give you different sizes for the same
insertions. `BigDecimal` is the canonical example and it is the JDK's own: `1.0`
and `1.00` are `equals`-unequal and `compareTo`-equal, so a `HashSet` holds two
and a `TreeSet` holds one. The consequence people underestimate is that the
`Map` interface's contract is defined in terms of `equals`, so a `TreeMap` with
an inconsistent comparator is documented as violating the `Map` contract — the
Javadoc says so in `SortedMap`. It is not a `TreeMap` bug, it is `TreeMap`
telling you in advance that it obeys your comparator. Practically, the way this
bites is someone converting a `HashMap` to a `TreeMap` for sorted output and
silently merging entries. My rule is: if a comparator is going into a sorted
collection, it must have a unique final tie-break, so nothing ties unless it is
genuinely the same key."

**What separates them:** knowing `TreeMap` never calls `equals` at all, citing
the `BigDecimal` case from the JDK's own docs, and offering the tie-break rule as
a preventive practice.

**Follow-up:** "What surviving key does `TreeMap.put` keep when it matches?" The
first-inserted key; only the value is replaced. Most people guess wrong.

---

### Q4 — "When would you use `TreeMap` over `HashMap`?"

**Mid-level answer:** "When you need the keys sorted, or when you need to iterate
in order."

**Senior answer:** "Sorted iteration is the obvious one, but if that is all I
need I'd usually collect and sort at the boundary rather than pay O(log n) on
every write. The reason I actually reach for `TreeMap` is the `NavigableMap`
queries — `floorEntry`, `ceilingEntry`, `subMap`, `headMap` — because those are
operations a `HashMap` cannot perform at all, at any cost, without a full scan.
Range and nearest-match lookups: pricing tiers by quantity, rate cards by
effective date, a schedule keyed by timestamp where I need 'the last entry before
now'. Those are O(log n) in a `TreeMap` and O(n) in anything else. The trades I'd
state: `TreeMap` is O(log n) rather than O(1) for point lookups, it allocates a
node per entry with parent/left/right pointers so it is heavier per entry than a
`HashMap` bucket node, it pointer-chases rather than scanning contiguously so it
is worse for cache locality, and it rejects null keys. And the correctness trade
is the big one — it defines key identity by the comparator, so I have to check
consistency with `equals` before I switch."

**What separates them:** naming the `NavigableMap` operations as the actual
reason rather than sortedness, and volunteering all four trades including the
correctness one.

**Follow-up:** "Give me a real query `floorEntry` answers." Any of: pricing tier
for a quantity, the applicable tax rate for a date, the current leader in a
score-keyed map, the most recent snapshot at or before a timestamp.

---

### Q5 — "Sort orders by status, then newest first, then by ID. Write the comparator."

**Mid-level answer:**
```java
Comparator.comparing(Order::status).thenComparing(Order::placedAt).thenComparing(Order::id).reversed()
```
"...actually, wait, `reversed()` reverses everything. Let me think."

**Senior answer:**
```java
Comparator.comparing(Order::status)
          .thenComparing(Order::placedAt, Comparator.reverseOrder())
          .thenComparing(Order::id);
```
"`.reversed()` applies to the whole composed comparator, so putting it at the end
would reverse the status ordering too — that is the classic mistake and it is
silent. The two-argument `thenComparing(keyExtractor, keyComparator)` reverses
only that one key. Three other things I'd say about this comparator: the final
tie-break on `id` is not optional — without a unique last key the sort is
non-deterministic for tied rows, so a dashboard reshuffles between page loads and
paginated results can skip or duplicate a row. If `status` is an enum, its
natural order is declaration order, so the enum's constant order is now a
load-bearing part of the UI and I'd add a comment or an explicit `priority()`
method rather than relying on it. And if any extracted key is a primitive I'd use
`comparingInt`/`comparingLong` to avoid boxing on every comparison — an n log n
number of allocations for nothing."

**What separates them:** getting `.reversed()` right first time, and volunteering
the determinism argument for the final tie-break, which is the part that actually
causes production bugs in paginated APIs.

**Follow-up:** "Why does a non-deterministic sort break pagination?" Because
`LIMIT/OFFSET` over an unstable order can return the same row on two pages and
skip another entirely. It is the same reason a database query needs a unique tie-
break in its `ORDER BY` — which you already know from SQL, so say so.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `compareTo` returns an `int`, not an enum of LESS/EQUAL/GREATER. What does
   that design buy, and what specific bug class does it enable? Would you have
   made the same choice?

2. The Javadoc "strongly recommends" consistency with `equals` but does not
   require it. Name the class in the JDK that forced that hedge, and explain why
   requiring consistency would have been the wrong call.

3. `TreeMap.put` on a matching key keeps the **old** key and the **new** value.
   Give a defensible reason for that asymmetry.

4. TimSort's contract check needs at least 32 elements to fire. Argue that this
   makes the exception *more* useful than a check that always fired, then argue
   the opposite.

5. A `PriorityQueue` guarantees only its head. `TreeSet` maintains full order.
   Both are O(log n) for insert. What is `PriorityQueue` buying with that weaker
   guarantee, and when does it matter?

6. You have a comparator that is correct but expensive — it does a database
   lookup per comparison. Sorting 10,000 elements is n log n comparisons. What
   are your options, and which one is the `Comparator` API actively encouraging
   you toward?

7. In TypeScript you pass a comparator function to every `.sort()` call and there
   is no `Comparable` equivalent. What does Java's split between the two buy?
   What does it cost? Which design would you choose for a new language?

---

## Quick reference card

### Contract summary

```
compareTo / compare:
  sgn(x.compareTo(y)) == -sgn(y.compareTo(x))          antisymmetric
  x > y && y > z  =>  x > z                            transitive
  x.compareTo(y)==0  =>  sgn(x.cmp(z)) == sgn(y.cmp(z))  substitutable
  STRONGLY RECOMMENDED: (x.compareTo(y)==0) == x.equals(y)

TreeMap/TreeSet identity is compare()==0. equals() is NEVER called.
HashMap/HashSet identity is hashCode() + equals(). compareTo is never called.
```

### Comparator builder cheat sheet

```java
Comparator.naturalOrder()
Comparator.reverseOrder()
Comparator.comparing(Order::status)                       // boxes if key is primitive
Comparator.comparingInt(Order::priority)                  // no boxing  <- prefer
Comparator.comparingLong(Order::totalMinor)
Comparator.comparingDouble(Product::weightKg)
Comparator.comparing(Order::couponCode, Comparator.nullsLast(Comparator.naturalOrder()))
    .thenComparing(Order::placedAt, Comparator.reverseOrder())   // reverse ONE key
    .thenComparing(Order::id);                                   // unique tie-break
Comparator.<Order>naturalOrder().reversed()                      // reverses EVERYTHING left of it
```

### NavigableMap query map

| You want | Call |
|---|---|
| the tier a value falls into | `floorEntry(v)` |
| the next scheduled item at or after now | `ceilingEntry(now)` |
| strictly before / strictly after | `lowerEntry(v)` / `higherEntry(v)` |
| smallest / largest | `firstEntry()` / `lastEntry()` |
| remove smallest (a sorted queue) | `pollFirstEntry()` |
| everything in a range | `subMap(lo, true, hi, false)` |
| reverse iteration | `descendingMap()` / `descendingKeySet()` |

All `*Map` and `*Set` results are **live views** that write through and enforce
their range.

### Choosing

| Need | Use |
|---|---|
| O(1) point lookup, no order | `HashMap` |
| insertion order | `LinkedHashMap` (Topic 15) |
| sorted iteration + range queries | `TreeMap` |
| only ever need the minimum | `PriorityQueue` |
| enum keys | `EnumMap` (Topic 16) |
| sorted **and** concurrent | `ConcurrentSkipListMap` (Topic 92) |

### Gotchas checklist

- [ ] Never subtract in a comparator. `Integer.compare` / `comparingInt`.
- [ ] `.reversed()` reverses everything to its left. Use `thenComparing(fn, cmp)` for one key.
- [ ] Always end a sort comparator with a unique tie-break, or pagination breaks.
- [ ] `TreeMap` never calls `equals`. Check consistency before switching from `HashMap`.
- [ ] `TreeMap`/`TreeSet` reject null keys. `HashMap` allows one.
- [ ] `PriorityQueue` iteration is heap order, not sorted order. Only `poll()` is ordered.
- [ ] `comparing` boxes; `comparingInt`/`Long`/`Double` do not.
- [ ] Below 32 elements TimSort silently tolerates a broken comparator.
- [ ] A comparator that has never thrown is not thereby correct.
- [ ] `useLegacyMergeSort` hides the detector, not the defect.

---

## When would I use this at work?

**1. Any "which bracket does this fall into" question.**
Volume pricing, tax bands, shipping-weight tiers, rate limits by plan, SLA
thresholds. All of them are `NavigableMap.floorEntry`. Once you have seen it, the
`if/else if/else if` ladder in the codebase looks like what it is: a hand-rolled,
un-testable, order-dependent tree lookup.

**2. Any list an API returns with pagination.**
The moment you write `ORDER BY` in SQL or `.sorted()` in Java for a paginated
endpoint, you add a unique tie-break. You already know this from SQL; the Java
side is the same rule and most Java developers do not apply it. Catching it in
review prevents a whole class of "customers report seeing the same order twice"
tickets.

**3. Triage of a `Comparison method violates its general contract!` page.**
It is 2am, the stack trace names only JDK classes, and the service is throwing on
a specific dataset. You go straight to the comparator, look for subtraction or a
tolerance, and you know it has been broken since the day it was written. Ten
minutes, not two hours — and you know to check whether *smaller* datasets have
been silently mis-sorted the whole time, which is the question nobody else asks.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and boxing:** why `comparingInt` beats `comparing`, and why
  subtraction overflows in a way your TypeScript instincts do not warn you about.
- **10 — Collections Framework:** `SortedMap`/`NavigableMap` as contracts, and
  `Iterator.remove` on the view collections.
- **12 — HashMap internals:** the contrast that makes this topic land. Hash
  lookup versus tree navigation, and two different definitions of key identity.
- **13 — equals/hashCode contract:** required. "Consistent with equals" is
  meaningless unless you know exactly what `equals` promises.

**This unlocks:**
- **15 — LinkedHashMap:** the third ordering story — neither hash nor sorted, but
  insertion or access order.
- **16 — Sets and EnumSet:** `TreeSet` versus `EnumSet` for a small ordered key
  space, and why enum natural order is declaration order.
- **21/22 — Lambdas and method references:** `Comparator` is a functional
  interface and `Order::totalMinor` is a method reference; this topic is your
  first heavy use of both.
- **23/24 — Streams:** `Stream.sorted(cmp)`, `Collectors.groupingBy` with a
  `TreeMap` supplier, `max`/`min` taking a comparator.
- **27 — Records:** records do not implement `Comparable` automatically, which is
  deliberate — there is no obvious natural order over several components.
- **47 — Spring Data JPA:** `Sort` and `Pageable` are the database-side version
  of this, and the tie-break rule is identical.
- **92 — Concurrent collections:** `ConcurrentSkipListMap` is the thread-safe
  `NavigableMap`, using a skip list rather than a red-black tree.

---

*Java baseline 21. `Comparable`, `Comparator` and `TreeMap` have been stable
since Java 8 added the comparator builder methods; nothing here differs between
Java 21 and Java 25. TimSort has been the object-array sort since Java 7 and the
contract-violation exception dates from then. The one thing I could not verify
without a JVM is whether `-Djava.util.Arrays.useLegacyMergeSort` is still honoured
on 21/25 — Proof 2 settles that in five seconds, and either answer is worth
knowing.*
