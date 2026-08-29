# 16 — Sets, EnumMap, EnumSet, and Choosing the Right Collection

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

A **set** is a guest list. One line per person. Asking "is Priya on the list?"
must be instant, and adding Priya twice must still leave one line.

Every set in Java is really a map with the values thrown away. `HashSet` is a
`HashMap` where every key points at the same meaningless placeholder object.
That is not an analogy — it is literally the implementation. So everything you
learned in Topics 12, 13, 14 and 15 about maps applies unchanged to sets. There
is nothing new to learn about how sets work.

Now the part that *is* new.

Suppose the guest list can only ever contain the seven people who work in your
office. Not "probably seven". **Exactly** seven, known when the building was
built, and it is a compile error to invite anyone else.

You would not build a hash table for that. You would take a strip of card with
seven boxes on it and tick the boxes. "Is Priya coming?" is "look at box 4".
Adding, removing, checking, taking the union of two lists — all of it becomes
ticking and comparing boxes. No hashing. No collisions. No allocation per guest.
Seven boxes fit in a single machine word, so the entire guest list *is* one
64-bit number, and "who is on both lists" is one AND instruction.

That is `EnumSet`. And `EnumMap` is the same idea for a map: an array with one
slot per constant, indexed directly.

When the key space is a fixed, small, known set of named things — which in a
business system it very often is — you should not be hashing at all.

---

## The bridge from what you know

### What transfers

`Set` maps cleanly onto what you know:

```ts
const shipped = new Set<string>();
shipped.add("SKU-4471");
shipped.has("SKU-4471");     // true
shipped.size;
```

```java
Set<String> shipped = new HashSet<>();
shipped.add("SKU-4471");
shipped.contains("SKU-4471");  // true
shipped.size();
```

**Verdict: PARTIAL analogue** — the shape matches, and the same caveat from
Topic 13 applies: JS `Set` uses SameValueZero (reference identity for objects),
Java uses your `hashCode`/`equals`.

### What does not transfer

| TypeScript | Java | Verdict |
|---|---|---|
| `Set` iterates in insertion order, guaranteed | `HashSet` order is unspecified; `LinkedHashSet` gives insertion order | **PARTIAL** — same fix as Topic 15 |
| No sorted set | `TreeSet` + `NavigableSet` | **NO ANALOGUE** |
| Set algebra by hand: `[...a].filter(x => b.has(x))` | `retainAll`, `removeAll`, `addAll` mutate in place | **PARTIAL** — Java's are destructive, yours are not |
| String-literal union types (`"PAID" \| "SHIPPED"`) exist only at compile time | `enum` is a real runtime type with identity, methods, and a fixed instance per constant | **PARTIAL — and the Java version is stronger** |
| `Set<Status>` is just a set of strings | `EnumSet<Status>` is a bitvector in a `long` | **NO ANALOGUE** |
| `Object.freeze()` — shallow, and the object is still the same object | `Set.of(...)` — genuinely immutable, rejects nulls, randomised iteration order | **PARTIAL** |

Two rows deserve expansion.

**On enums.** Your TypeScript instinct for a fixed set of states is a string
union: `type OrderStatus = "CREATED" | "PAID" | "SHIPPED"`. That is erased at
compile time; at runtime you have strings, and `Set<OrderStatus>` is a set of
strings with all the hashing that implies. Java's `enum` is a real class with
exactly one instance per constant, created at class-initialisation time. Two
consequences: `==` is *correct and preferred* for enums (unlike every other
reference type), and each constant has an `ordinal()` — a dense integer index —
which is what makes array-based collections possible.

**On `Set.of(...)` iteration order.** JavaScript's `Set` promises insertion
order. Java's `Set.of(...)` not only refuses to promise an order, it
**deliberately randomises it per JVM run**. Two runs of the same program with
the same code print the elements in different orders. That is not a bug; it is
an explicit design decision to stop code depending on an unspecified order. It
also produces one of the more confusing flaky-test experiences in Java, which is
Trap 3 below.

---

## What is this?

### The `Set` family

| Implementation | Backed by | Order | Nulls | Notes |
|---|---|---|---|---|
| `HashSet` | `HashMap` | none, unspecified | one | the default |
| `LinkedHashSet` | `LinkedHashMap` | insertion | one | order in the type (Topic 15) |
| `TreeSet` | `TreeMap` | sorted | none | `NavigableSet` queries (Topic 14) |
| `EnumSet` | a `long` or `long[]` | ordinal | none | for enum elements only |
| `Set.of(...)` | `ImmutableCollections` | **randomised per JVM run** | none | rejects duplicates at construction |
| `CopyOnWriteArraySet` | `CopyOnWriteArrayList` | insertion | one | O(n) `contains`; Topic 92 |
| `ConcurrentHashMap.newKeySet()` | `ConcurrentHashMap` | none | none | the concurrent `HashSet`; Topic 92 |

`HashSet`'s implementation is three lines you should know:

```java
private transient HashMap<E,Object> map;
private static final Object PRESENT = new Object();
public boolean add(E e) { return map.put(e, PRESENT) == null; }
```

So a `HashSet<Product>` costs exactly what a `HashMap<Product, Object>` costs.
There is no space saving from "not having values" — every entry still has a
`Node` with a `value` field pointing at the shared `PRESENT` singleton.

### `EnumSet`

An abstract class with two package-private implementations:

- **`RegularEnumSet`** — for enums with 64 or fewer constants. The entire set is
  **one `long` field**, one bit per constant, indexed by `ordinal()`.
- **`JumboEnumSet`** — for more than 64 constants. A `long[]`.

You never name either; you use the factories:

```java
EnumSet.noneOf(OrderStatus.class)          // empty
EnumSet.allOf(OrderStatus.class)           // every constant
EnumSet.of(PAID, SHIPPED)                  // these constants
EnumSet.range(CREATED, PAID)               // an ordinal range, inclusive
EnumSet.copyOf(someCollection)
EnumSet.complementOf(someEnumSet)          // everything not in it
```

What the bitvector buys:

| Operation | `HashSet<OrderStatus>` | `EnumSet<OrderStatus>` |
|---|---|---|
| `contains` | spread hash, mask, bucket walk, `equals` | `(bits & (1L << ordinal)) != 0` — one AND |
| `add` | possibly a resize, always a `Node` allocation | `bits \|= (1L << ordinal)` — one OR |
| memory for the whole set | ~48 bytes per element | **8 bytes total**, regardless of how many elements |
| iteration order | unspecified | ordinal (declaration) order, always |

That memory row is not a typo. A `RegularEnumSet` holding all eight of your order
statuses is one `long`. A `HashSet` holding the same eight is a table array plus
eight `Node` objects.

### `EnumMap`

```java
// conceptually:
private final K[] keyUniverse;    // every constant, cached
private final Object[] vals;      // one slot per constant, indexed by ordinal
private int size;
```

`get` is `vals[key.ordinal()]`. `put` is `vals[key.ordinal()] = v`. No hashing, no
collisions, no resize, no load factor. Keys iterate in ordinal order.

The constructor needs the `Class` object, because it must know the universe:

```java
Map<OrderStatus, Long> counts = new EnumMap<>(OrderStatus.class);
```

`EnumMap` allows `null` **values** (stored via an internal sentinel) but not
`null` keys.

### Immutable factories (Java 9+)

```java
Set.of("A", "B")            // immutable, no nulls, no duplicates, randomised order
Map.of("k", 1, "k2", 2)     // up to 10 pairs
Map.ofEntries(entry(...))   // more than 10
Set.copyOf(collection)      // a genuine copy
Collections.unmodifiableSet(s)   // a VIEW, not a copy  [LEGACY — still asked]
```

The distinction in that last pair matters and is Trap 5.

---

## Why does it matter?

**1. `EnumMap`/`EnumSet` are strictly better and almost nobody uses them.**

There is no trade-off to weigh. For enum keys, `EnumMap` is faster, smaller, has
deterministic iteration order, and is more type-safe than `HashMap`. It is a pure
win. The reason to know it is not performance — it is that reaching for it
signals you have thought about the key space, and it is one of the cheapest
"this person knows Java" markers in a code review.

**2. Choosing a collection is a decision you make dozens of times a week.**

Most of the time it does not matter. When it matters, it matters a lot: a
quadratic `removeAll`, an unbounded `HashSet` leak, a `TreeSet` that silently
merges two prices. This topic is the decision table that closes Phase 1's
collections arc.

**3. `Set.of`'s randomised iteration order will confuse you at least once.**

It is worth meeting deliberately, in a controlled setting, rather than at 4pm on
a Friday when CI has failed for the third time with an assertion that passes
locally.

---

## Syntax breakdown

### `enum` — more than a constant list

```java
public enum OrderStatus {
    CREATED(0),
    PENDING_PAYMENT(1),
    PAID(2),
    RESERVED(3),
    SHIPPED(4),
    DELIVERED(5),
    CANCELLED(9),
    REFUNDED(10);

    private final int priority;

    OrderStatus(int priority) { this.priority = priority; }   // implicitly private

    public int priority() { return priority; }

    public boolean isTerminal() {
        return this == DELIVERED || this == CANCELLED || this == REFUNDED;
    }
}
```

| Bit of syntax | What it means |
|---|---|
| `CREATED(0)` | a constant with a constructor argument. Each constant is an instance |
| the constructor | always private, whether you write it or not. You cannot create new constants at runtime |
| `this == DELIVERED` | `==` on enums is correct and idiomatic — exactly one instance exists per constant. This is the only reference type where `==` is preferred |
| `ordinal()` | the declaration index, 0-based. Inherited from `java.lang.Enum` |
| `values()` | a compiler-generated static method returning a **fresh array copy** every call |
| `valueOf("PAID")` | name lookup; throws `IllegalArgumentException` if unknown |

Two things to internalise now:

- **`values()` allocates a new array on every call.** It has to, because arrays
  are mutable and the JDK cannot hand out a shared one. Calling `values()` inside
  a loop is a real allocation cost. Cache it in a `private static final` array,
  or use `EnumSet.allOf(...)`.
- **`ordinal()` is declaration order and nothing more.** It is not a business
  identifier. Reordering constants is a source-compatible change that silently
  changes every ordinal. Never persist it. See Trap 4.

### `switch` over an enum

```java
String label = switch (status) {
    case CREATED, PENDING_PAYMENT -> "Awaiting payment";
    case PAID, RESERVED           -> "Preparing";
    case SHIPPED                  -> "On its way";
    case DELIVERED                -> "Complete";
    case CANCELLED, REFUNDED      -> "Closed";
};
```

This is a **switch expression** with arrow labels (standard since Java 14). Two
properties that matter here:

1. No `break`, no fall-through.
2. Over an enum with no `default`, the compiler checks **exhaustiveness**. Add a
   new constant and every such switch becomes a compile error, which is exactly
   the behaviour you get from a TypeScript discriminated union with a `never`
   check. Full treatment is Topic 29.

That exhaustiveness is a genuine reason to prefer a `switch` over an
`EnumMap`-of-handlers in some cases: the map compiles fine when you forget a
status, the switch does not.

### Static import for readability

```java
import static com.orderflow.orders.OrderStatus.*;

EnumSet<OrderStatus> open = EnumSet.of(CREATED, PENDING_PAYMENT, PAID, RESERVED);
```

`import static` brings the constants into scope unqualified. Use it sparingly —
it is worth it for enum constants in a file that is *about* those constants, and
it is a readability problem everywhere else.

---

## Example 1 — minimal

```java
import java.util.*;

public class EnumCollections {

    enum OrderStatus { CREATED, PENDING_PAYMENT, PAID, SHIPPED, DELIVERED, CANCELLED }

    public static void main(String[] args) {

        EnumSet<OrderStatus> open   = EnumSet.of(OrderStatus.CREATED,
                                                 OrderStatus.PENDING_PAYMENT,
                                                 OrderStatus.PAID);
        EnumSet<OrderStatus> closed = EnumSet.complementOf(open);

        System.out.println("open       : " + open);
        System.out.println("closed     : " + closed);
        System.out.println("all        : " + EnumSet.allOf(OrderStatus.class));
        System.out.println("range      : " + EnumSet.range(OrderStatus.PAID,
                                                           OrderStatus.DELIVERED));

        // Set algebra. These MUTATE the receiver.
        EnumSet<OrderStatus> a = EnumSet.of(OrderStatus.PAID, OrderStatus.SHIPPED);
        EnumSet<OrderStatus> b = EnumSet.of(OrderStatus.SHIPPED, OrderStatus.DELIVERED);

        EnumSet<OrderStatus> union = EnumSet.copyOf(a); union.addAll(b);
        EnumSet<OrderStatus> inter = EnumSet.copyOf(a); inter.retainAll(b);
        EnumSet<OrderStatus> diff  = EnumSet.copyOf(a); diff.removeAll(b);

        System.out.println("a union b  : " + union);
        System.out.println("a inter b  : " + inter);
        System.out.println("a minus b  : " + diff);

        EnumMap<OrderStatus, Long> counts = new EnumMap<>(OrderStatus.class);
        counts.put(OrderStatus.SHIPPED, 42L);
        counts.put(OrderStatus.CREATED, 7L);
        System.out.println("counts     : " + counts);   // ordinal order, not insertion
    }
}
```

Run it. Two things to notice:

1. Every printed set is in **declaration order**, regardless of how you built it.
   That determinism is free and it is why `EnumSet` output is safe to assert on
   in a test, unlike `HashSet` or `Set.of`.
2. `counts` prints `CREATED` before `SHIPPED` even though `SHIPPED` was inserted
   first. `EnumMap` is ordinal-ordered, not insertion-ordered.

Note also that `EnumSet.copyOf(a); union.addAll(b)` is three lines to do what one
`|` would do on the underlying `long`. `EnumSet` has no immutable-combine API;
the set-algebra methods all mutate. If you want an immutable result, copy first —
exactly as above.

---

## Example 2 — production scenario

`orderflow` needs an order state machine. Which transitions are legal, which
statuses count as "open", which handler runs per status, and how many orders sit
in each status right now.

### The version that ships and then loses money

```java
public class OrderStateService {

    private static final Set<String> OPEN_STATUSES =
        new HashSet<>(Arrays.asList("CREATED", "PENDING_PAYMENT", "PAID", "RESERVED"));

    private static final Map<String, List<String>> TRANSITIONS = new HashMap<>();
    static {
        TRANSITIONS.put("CREATED",         Arrays.asList("PENDING_PAYMENT", "CANCELLED"));
        TRANSITIONS.put("PENDING_PAYMENT", Arrays.asList("PAID", "CANCELLED"));
        TRANSITIONS.put("PAID",            Arrays.asList("RESERVED", "REFUNDED"));
        TRANSITIONS.put("RESERVED",        Arrays.asList("SHIPPED", "REFUNDED"));
        TRANSITIONS.put("SHIPPED",         Arrays.asList("DELIVERED"));
    }

    private final Map<String, Integer> countByStatus = new HashMap<>();

    public boolean canTransition(String from, String to) {
        List<String> allowed = TRANSITIONS.get(from);
        return allowed != null && allowed.contains(to);
    }

    public boolean isOpen(String status) {
        return OPEN_STATUSES.contains(status);
    }

    public void record(String status) {
        countByStatus.put(status, countByStatus.get(status) + 1);
    }
}
```

It works. Here is everything wrong with it.

**Stringly typed.** `canTransition("PAID", "SHIPPED")` compiles. So does
`canTransition("paid", "shipped")`, and `canTransition("PAYD", "SHIPPED")`. All
three return `false` — one because the transition is genuinely illegal, two
because of typos. The symptom is identical in all three cases and there is no way
to tell them apart from the outside. Orders get stuck in a status with no error
anywhere.

**Adding a status silently breaks it.** Someone adds `PARTIALLY_REFUNDED`. Every
`switch` on the status still compiles. `TRANSITIONS` has no entry, so
`canTransition` returns `false` for everything out of it. `OPEN_STATUSES` does
not include it, so it counts as closed. The order is now unreachable by any code
path. **Nothing fails to compile and nothing throws.**

**`List.contains` inside a lookup.** `allowed.contains(to)` is a linear scan of a
`List`. It is short, so this is a style problem rather than a performance one —
but it means the data structure does not express "a set of destinations", so
nothing stops someone adding a duplicate.

**`countByStatus.get(status) + 1` NPEs.** First call for any status returns
`null`, unboxing throws. Topic 01, Trap 2, unchanged.

**A `HashMap` for enum-shaped keys.** Hashing strings on a hot path to look up a
value from a fixed set of six.

### The corrected version

```java
public enum OrderStatus {
    CREATED, PENDING_PAYMENT, PAID, RESERVED, SHIPPED, DELIVERED, CANCELLED, REFUNDED;

    public boolean isTerminal() {
        return TERMINAL.contains(this);
    }

    private static final EnumSet<OrderStatus> TERMINAL =
        EnumSet.of(DELIVERED, CANCELLED, REFUNDED);
}
```

```java
public class OrderStateService {

    private static final EnumSet<OrderStatus> OPEN =
        EnumSet.of(CREATED, PENDING_PAYMENT, PAID, RESERVED);

    private static final EnumMap<OrderStatus, EnumSet<OrderStatus>> TRANSITIONS =
        new EnumMap<>(Map.of(
            CREATED,         EnumSet.of(PENDING_PAYMENT, CANCELLED),
            PENDING_PAYMENT, EnumSet.of(PAID, CANCELLED),
            PAID,            EnumSet.of(RESERVED, REFUNDED),
            RESERVED,        EnumSet.of(SHIPPED, REFUNDED),
            SHIPPED,         EnumSet.of(DELIVERED),
            DELIVERED,       EnumSet.noneOf(OrderStatus.class),
            CANCELLED,       EnumSet.noneOf(OrderStatus.class),
            REFUNDED,        EnumSet.noneOf(OrderStatus.class)));

    private final EnumMap<OrderStatus, Long> countByStatus = new EnumMap<>(OrderStatus.class);

    public boolean canTransition(OrderStatus from, OrderStatus to) {
        return TRANSITIONS.get(from).contains(to);       // two array reads and a bit test
    }

    public boolean isOpen(OrderStatus status) {
        return OPEN.contains(status);
    }

    public void record(OrderStatus status) {
        countByStatus.merge(status, 1L, Long::sum);      // no NPE, no check-then-act
    }

    /** Fails at startup if any status is unreachable from CREATED. */
    static {
        EnumSet<OrderStatus> reachable = EnumSet.of(CREATED);
        int before;
        do {
            before = reachable.size();
            EnumSet<OrderStatus> next = EnumSet.copyOf(reachable);
            for (OrderStatus s : reachable) next.addAll(TRANSITIONS.get(s));
            reachable = next;
        } while (reachable.size() != before);

        EnumSet<OrderStatus> orphans = EnumSet.complementOf(reachable);
        if (!orphans.isEmpty()) {
            throw new IllegalStateException("Unreachable order statuses: " + orphans);
        }
    }
}
```

What each change buys:

1. **`enum` instead of `String`** — `canTransition("PAYD", ...)` no longer
   compiles. Typos become compile errors instead of silent `false` returns.
2. **`EnumMap` for `TRANSITIONS`** — array indexing by ordinal. No hashing.
   Deterministic iteration order. And the map is declared over the enum, so a
   `Map.of` entry for a status that does not exist will not compile.
3. **`EnumSet` for the destination sets** — one `long` each rather than eight
   hash tables, and `contains` is a bit test.
4. **`merge(status, 1L, Long::sum)`** — Topic 01's fix for the counter NPE, in
   one call, with no check-then-act.
5. **The static initialiser reachability check** — this is the real win. Add
   `PARTIALLY_REFUNDED` and forget to wire it up, and the **application fails to
   start** with the status named in the message. The bug becomes impossible to
   deploy rather than merely unlikely.

That last one is the pattern worth stealing. `EnumSet.complementOf` gives you
"every constant I did not account for" as a one-liner, which turns a whole class
of "we forgot to handle the new case" bugs into startup failures.

> One caveat on the static block: an exception from a static initialiser becomes
> `ExceptionInInitializerError`, and every *subsequent* attempt to touch the class
> gives `NoClassDefFoundError` instead — a misleading second error that hides the
> real one. That is Topic 67's drill. For a startup invariant it is still the
> right place; just know what the failure looks like.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `HashMap`/`HashSet` for enum keys

**Wrong:**
```java
Map<OrderStatus, Long> counts = new HashMap<>();
Set<OrderStatus> open = new HashSet<>(List.of(CREATED, PAID));
```

**Exact symptom:** honestly, usually nothing you would notice. This is the one
trap in this document whose symptom is weak, and pretending otherwise would be
dishonest. What you get is: hashing and a `Node` allocation per entry where an
array index would do; iteration order that is unspecified rather than
declaration order, so any test asserting on it is fragile; and in a profile of a
genuinely hot loop, `Enum.hashCode` and `HashMap.getNode` showing up for work
that should be one array read.

**Root cause:** `Enum.hashCode()` is `super.hashCode()` — the identity hash. It
is fine, but it is entirely unnecessary work when `ordinal()` already gives you a
dense array index.

**Fix:** `new EnumMap<>(OrderStatus.class)` and `EnumSet.of(...)`. There is no
downside to weigh. The reason to do it is not the microseconds; it is the
determinism and the signal it sends about how you think about key spaces.

---

### Trap 2 — quadratic `removeAll` / `retainAll`

**Wrong:**
```java
Set<String> flagged = loadFlaggedSkus();            // 500 SKUs
List<String> catalogue = loadAllSkus();             // 800,000 SKUs

flagged.removeAll(catalogue);                       // O(500 * 800,000)
flagged.retainAll(catalogue);                       // also O(500 * 800,000)
```

**Exact symptom:** a single line takes minutes. A CPU profile shows nearly all
the time in `ArrayList.contains` and `ArrayList.indexOf`, under a frame named
`AbstractSet.removeAll` or `AbstractCollection.retainAll`. Nothing throws; it is
just inexplicably slow for what looks like a set operation.

**Root cause:** read `AbstractSet.removeAll`. It picks a strategy based on sizes:
if `this.size() > c.size()` it iterates `c` and calls `this.remove(...)` — good,
hashed. Otherwise it iterates `this` and calls **`c.contains(...)`** on each
element. When `c` is an `ArrayList`, `contains` is a linear scan. So a *small*
set minus a *large* list is the quadratic case.

`AbstractCollection.retainAll` is worse: it always iterates `this` and calls
`c.contains(...)`, with no size heuristic at all. So `retainAll` with a `List`
argument is quadratic regardless of the sizes.

**Fix:** make the argument a hashed collection.
```java
Set<String> catalogueSet = new HashSet<>(catalogue);   // O(n) once
flagged.removeAll(catalogueSet);                       // now O(500)
flagged.retainAll(catalogueSet);
```

**The general rule:** the cost of `a.removeAll(b)` and `a.retainAll(b)` depends
on the `contains` cost of **`b`**, not of `a`. Any `Collection` argument to a bulk
set operation should be a `Set`. This one rule prevents the whole family of bugs.

---

### Trap 3 — `Set.of` iteration order is randomised per run

**Wrong:**
```java
private static final Set<String> CHANNELS = Set.of("WEB", "MOBILE", "PARTNER");

// in a test:
assertEquals("WEB,MOBILE,PARTNER", String.join(",", CHANNELS));
```

**Exact symptom:** the test passes, then fails, then passes, with no code change
between runs. The failure message shows the same three elements in a different
order. Rerunning "fixes" it. It fails roughly at random in CI and is the classic
"just retry the pipeline" flake.

**Root cause:** `ImmutableCollections` — the class behind `Set.of` and `Map.of` —
computes a `SALT` value once per JVM from the system clock, and uses it to vary
the iteration order and the probe sequence. This is **deliberate**. The JDK
authors chose to make the unspecified order visibly unstable so that code cannot
accidentally come to depend on it. Compare `HashSet`, whose order is also
unspecified but happens to be stable for the same insertions in the same JVM
version — which is precisely why people accidentally depend on it.

**Fix:** if order matters, use a collection that has one.
```java
private static final Set<String> CHANNELS =
    Collections.unmodifiableSet(new LinkedHashSet<>(List.of("WEB", "MOBILE", "PARTNER")));
// or, if the assertion does not actually care about order:
assertEquals(Set.of("WEB", "MOBILE", "PARTNER"), CHANNELS);
```
And when the elements are enum constants, `EnumSet` gives you declaration order
for free, which is both meaningful and stable.

---

### Trap 4 — persisting `ordinal()`

**Wrong:**
```java
@Entity
public class Order {
    @Enumerated(EnumType.ORDINAL)          // stores 0, 1, 2, ... in the column
    private OrderStatus status;
}
```

Or, without JPA:
```java
redis.set("order:" + id + ":status", String.valueOf(status.ordinal()));
```

**Exact symptom:** the day someone inserts a new constant in the middle of the
enum — `PARTIALLY_REFUNDED` between `SHIPPED` and `DELIVERED`, because that is
where it belongs conceptually — **every existing row in the database silently
changes meaning.** Delivered orders become shipped. Refunded orders become
cancelled. There is no migration, no error, no log line. The code compiles, the
tests pass (the test data was written after the change), and the corruption is
retroactive across all history.

This is, without exaggeration, one of the worst bugs in this entire document.
There is no exception to catch and no moment at which it happens.

**Root cause:** `ordinal()` is the declaration index. Reordering or inserting
constants is a source-compatible change with no compile-time consequence — but
every persisted ordinal now points at a different constant.

**Fix:**
```java
@Enumerated(EnumType.STRING)      // stores "DELIVERED"
private OrderStatus status;
```
Or, for a stable wire format independent of the Java name, an explicit code:
```java
public enum OrderStatus {
    CREATED("CRT"), PAID("PAY"), SHIPPED("SHP");
    private final String code;
    OrderStatus(String code) { this.code = code; }
    public String code() { return code; }
    // plus a static Map<String, OrderStatus> for the reverse lookup
}
```

The rule: **`ordinal()` is for `EnumSet` and `EnumMap` to use internally. It is
never yours to persist, serialize, or send over a wire.** Note the nuance —
`EnumSet` and `EnumMap` depend on ordinals and that is completely safe, because
they live in memory and are rebuilt from the current class every run. It is
crossing a persistence or network boundary that kills you. And `EnumSet`'s own
serialized form is safe too: it uses a proxy that writes the constants
themselves, not their ordinals.

---

### Trap 5 — `unmodifiableSet` is a view, not a copy

**Wrong:**
```java
public class ProductCatalogue {
    private final Set<String> skus = new HashSet<>();

    public Set<String> allSkus() {
        return Collections.unmodifiableSet(skus);      // looks safe
    }
}
```

**Exact symptom:** a caller holds the returned set, iterates it later, and gets a
`ConcurrentModificationException` — or simply sees different contents than when
they received it. In a concurrent service, the caller's "snapshot" changes
underneath them mid-iteration.

**Root cause:** `Collections.unmodifiableSet` returns a **wrapper view**. It
blocks writes *through the wrapper*, but the underlying set is the same object
and can still be mutated by whoever holds it. The caller has an immutable handle
onto a mutable thing.

**Fix:**
```java
public Set<String> allSkus() {
    return Set.copyOf(skus);         // a genuine, immutable, defensive copy
}
```
`Set.copyOf` (Java 10+) copies unless the argument is already an immutable
`Set.of`-family collection, in which case it returns it as-is. That is the
correct semantics: you always get back something nobody else can change.

The general principle — defensive copying at both the constructor and the getter
— is Topic 17.

**Related trap in the same family:**
```java
Set.of("A", "A");                    // IllegalArgumentException: duplicate element: A
Set.of("A", null);                   // NullPointerException
Set.of("A").contains(null);          // NullPointerException, not false
```
All three are deliberate. The first is genuinely useful — a duplicate in a
literal is almost always a copy-paste bug and failing loudly at construction is
the right call. But note the blast radius: in a `static final` field, that
`IllegalArgumentException` becomes an `ExceptionInInitializerError` at class
load, and every subsequent touch of the class gives you a misleading
`NoClassDefFoundError` instead (Topic 67).

---

## Hands-on proof

Everything here is a command **you** run. I do not have a JVM and will not
fabricate output.

### Setup

```bash
mkdir -p ~/java-lab/16 && cd ~/java-lab/16
java --version
```

### Proof 1 — `Set.of` really does reorder between runs

`SaltProof.java`:
```java
import java.util.*;

public class SaltProof {
    public static void main(String[] args) {
        System.out.println("Set.of        : " + Set.of("WEB", "MOBILE", "PARTNER", "POS", "API"));
        System.out.println("Map.of keys   : " + Map.of("a",1,"b",2,"c",3,"d",4,"e",5).keySet());
        System.out.println("HashSet       : " + new HashSet<>(List.of("WEB","MOBILE","PARTNER","POS","API")));
        System.out.println("LinkedHashSet : " + new LinkedHashSet<>(List.of("WEB","MOBILE","PARTNER","POS","API")));
    }
}
```

```bash
for i in 1 2 3 4 5; do java SaltProof.java; echo "---"; done
```

| What you see | What it means |
|---|---|
| The `Set.of` line differs between runs | Confirmed: the per-JVM `SALT` randomises iteration order. This is deliberate |
| The `HashSet` line is identical every run | Confirmed and important: `HashSet` order is *unspecified* but *stable* for identical input on one JVM version. That stability is why people accidentally depend on it and get burned on an upgrade |
| The `LinkedHashSet` line is identical every run and matches the input order | Confirmed: order in the type |
| `Set.of` is identical across all five runs | Possible, especially with few elements — randomisation does not guarantee a *different* order each time. Add more elements and run 20 times. Absence of variation is not evidence of stability |

The lesson to take is not "Set.of is unreliable". It is the opposite: an
unspecified order that visibly varies is **safer** than an unspecified order that
happens to be stable, because the first one cannot be accidentally depended on.

### Proof 2 — read the EnumSet and EnumMap source

This is a short, readable pair of files and worth thirty minutes.

```bash
unzip -o "$JAVA_HOME/lib/src.zip" \
  'java.base/java/util/EnumSet.java' \
  'java.base/java/util/RegularEnumSet.java' \
  'java.base/java/util/EnumMap.java' \
  'java.base/java/util/HashSet.java' \
  -d ~/java-lab/16/src

grep -n "private long elements" ~/java-lab/16/src/java.base/java/util/RegularEnumSet.java
grep -n -A4 "public boolean contains" ~/java-lab/16/src/java.base/java/util/RegularEnumSet.java
grep -n -A6 "public V get" ~/java-lab/16/src/java.base/java/util/EnumMap.java
grep -n "PRESENT" ~/java-lab/16/src/java.base/java/util/HashSet.java
```

| What you see | What it means |
|---|---|
| `private long elements;` in `RegularEnumSet` | Confirmed: the whole set for a ≤64-constant enum is **one 64-bit field** |
| `contains` doing `(elements & (1L << ((Enum<?>)e).ordinal())) != 0` | Confirmed: a shift, an AND, a compare. No hashing, no branching on collisions |
| `EnumMap.get` doing `vals[((Enum<?>)key).ordinal()]` behind a type check | Confirmed: an array read |
| `private static final Object PRESENT = new Object();` in `HashSet` | Confirmed: a `HashSet` is a `HashMap` whose values all point at one shared dummy |
| A `SerializationProxy` class in `EnumSet` | Confirmed: the serialized form writes the constants, not the bitvector — which is why serializing an `EnumSet` survives a reordering of the enum, unlike a persisted `ordinal()` |

Also read `AbstractSet.removeAll` for Trap 2 — it is about ten lines and the
size heuristic is right there:

```bash
grep -n -A18 "public boolean removeAll" ~/java-lab/16/src/java.base/java/util/AbstractSet.java 2>/dev/null \
  || unzip -p "$JAVA_HOME/lib/src.zip" 'java.base/java/util/AbstractSet.java' | grep -n -A18 "public boolean removeAll"
```

**What to look for:** the `if (size() > c.size())` branch. Confirm for yourself
which side is the quadratic one, rather than taking my word for it.

### Proof 3 — measure the memory difference

```bash
curl -o jol-cli.jar \
  https://repo1.maven.org/maven2/org/openjdk/jol/jol-cli/0.17/jol-cli-0.17-full.jar
```

`MemProof.java`:
```java
import java.util.*;
import org.openjdk.jol.info.GraphLayout;

public class MemProof {
    enum Status { A,B,C,D,E,F,G,H }

    public static void main(String[] args) {
        EnumSet<Status> enumSet = EnumSet.allOf(Status.class);
        Set<Status> hashSet     = new HashSet<>(EnumSet.allOf(Status.class));

        EnumMap<Status, Long> enumMap = new EnumMap<>(Status.class);
        Map<Status, Long> hashMap     = new HashMap<>();
        for (Status s : Status.values()) { enumMap.put(s, 1L); hashMap.put(s, 1L); }

        System.out.println("EnumSet  : " + GraphLayout.parseInstance(enumSet).totalSize());
        System.out.println("HashSet  : " + GraphLayout.parseInstance(hashSet).totalSize());
        System.out.println("EnumMap  : " + GraphLayout.parseInstance(enumMap).totalSize());
        System.out.println("HashMap  : " + GraphLayout.parseInstance(hashMap).totalSize());
    }
}
```

```bash
java -cp jol-cli.jar MemProof.java
```

| What you see | What it means |
|---|---|
| `EnumSet` far smaller than `HashSet` | Confirmed. The `EnumSet` is one `long` plus a header and a reference to the cached universe array |
| `EnumMap` smaller than `HashMap` | Confirmed: one array of 8 slots versus a table plus 8 `Node` objects |
| The gap is smaller than you expected | Likely: `totalSize` follows references, so it may be counting the shared enum constants and the cached `values()` array in both. Try `parseInstance(x).toFootprint()` and read the per-class breakdown instead of the total |
| A `NoClassDefFoundError` for JOL | Your `-cp` did not take. Single-file source launch plus a classpath can be fiddly — compile with `javac -cp jol-cli.jar MemProof.java` and run with `java -cp .:jol-cli.jar MemProof` |

> Object layout in full is Topic 69. Here you are collecting one fact — the
> relative sizes — not learning the tool.

### Proof 4 — reproduce the quadratic removeAll

`BulkOps.java`:
```java
import java.util.*;

public class BulkOps {
    public static void main(String[] args) {
        int n = 200_000;
        List<String> big = new ArrayList<>(n);
        for (int i = 0; i < n; i++) big.add("SKU-" + i);
        Set<String> bigSet = new HashSet<>(big);

        for (String mode : new String[] { "list", "set" }) {
            Collection<String> arg = mode.equals("list") ? big : bigSet;
            Set<String> small = new HashSet<>();
            for (int i = 0; i < 500; i++) small.add("SKU-" + i);

            long t0 = System.nanoTime();
            small.removeAll(arg);
            long ms = (System.nanoTime() - t0) / 1_000_000;
            System.out.printf("removeAll(%s) : %d ms, remaining=%d%n", mode, ms, small.size());
        }
    }
}
```

```bash
java BulkOps.java
```

| What you see | What it means |
|---|---|
| `list` far slower than `set` | Confirmed: the small set iterates itself and calls `ArrayList.contains` 500 times over 200,000 elements |
| Both fast | Your `n` is too small, or the JIT hoisted something. Raise `n` to 2,000,000 and re-run |
| `set` is slower | Surprising — check that `bigSet` is actually a `HashSet` and not accidentally the list |

Then swap `removeAll` for `retainAll` and re-run. `retainAll` has no size
heuristic, so it should be slow with a `List` argument regardless.

> These are wall-clock timings from a naive loop, which Topic 77 will show you is
> the wrong way to benchmark. They are adequate here only because the difference
> is orders of magnitude, not percentages. Note that limitation when you record
> your result.

### Proof 5 — the ordinal trap, made concrete

`OrdinalTrap.java`:
```java
import java.util.*;

public class OrdinalTrap {
    enum V1 { CREATED, PAID, SHIPPED, DELIVERED }
    enum V2 { CREATED, PAID, PARTIALLY_REFUNDED, SHIPPED, DELIVERED }   // inserted in the middle

    public static void main(String[] args) {
        int persisted = V1.SHIPPED.ordinal();
        System.out.println("stored ordinal for SHIPPED (v1) : " + persisted);
        System.out.println("that ordinal now means (v2)     : " + V2.values()[persisted]);

        System.out.println("stored NAME for SHIPPED (v1)    : " + V1.SHIPPED.name());
        System.out.println("that name now means (v2)        : " + V2.valueOf(V1.SHIPPED.name()));
    }
}
```

```bash
java OrdinalTrap.java
```

| What you see | What it means |
|---|---|
| The ordinal that meant `SHIPPED` now resolves to `PARTIALLY_REFUNDED` | Confirmed. Now imagine that is 40 million rows and the change went out on a Tuesday |
| The name resolves correctly in both versions | Confirmed: `@Enumerated(STRING)` and name-based serialization survive reordering |
| `IllegalArgumentException` from `valueOf` | Would mean the constant was *renamed*, which name-based storage does not survive either. That is why a stable explicit `code()` is the strongest option |

---

## Practice exercises

### 1 — Easy: rebuild set algebra with EnumSet

Given an `OrderStatus` enum of eight constants, define:
- `OPEN` — statuses where the order can still change
- `PAID_STATES` — statuses implying money has been taken
- `TERMINAL` — statuses that cannot transition anywhere

Then compute and print, using only `EnumSet` operations:
1. Statuses that are both open and paid.
2. Statuses that are paid but not terminal.
3. Statuses in none of the three sets (use `complementOf`).
4. Whether `OPEN` and `TERMINAL` are disjoint — without writing a loop.

Requirements:
- No `if` statements and no loops except in printing.
- Every intermediate result must itself be an `EnumSet`.
- For (3), write an assertion that fails if the answer is non-empty, and explain
  in a comment why that assertion is a useful thing to have in a real codebase.

### 2 — Medium: the audit (combines Topics 01, 10, 11, 12, 13, 14, 15)

Here is an `orderflow` fulfilment service with **seven** defects from Topics
01–16. Find them all, state the exact observable symptom of each, and rewrite it.

```java
public class FulfilmentService {

    private static final Set<String> SHIPPABLE =
        Set.of("PAID", "RESERVED", "PAID");

    private final Map<String, Integer> pendingBySku = new HashMap<>();
    private final Set<Order> inFlight = new HashSet<>();

    public void queue(Order order, List<String> skus) {
        inFlight.add(order);
        for (String sku : skus) {
            pendingBySku.put(sku, pendingBySku.get(sku) + 1);
        }
    }

    public void complete(Order order) {
        order.setStatus("DELIVERED");
        inFlight.remove(order);
    }

    public List<String> shippableSkus(List<String> candidateSkus) {
        Set<String> result = new HashSet<>(pendingBySku.keySet());
        result.retainAll(candidateSkus);
        return new ArrayList<>(result);
    }

    public String summary() {
        return String.join(",", SHIPPABLE);
    }
}
```

Hints so you look in the right places: one from Topic 01, one from Topic 12, one
from Topic 13, one from Topic 14 or 15, and three from this topic. One defect
prevents the class from loading at all — identify it and say what the *second*,
misleading error is that everyone will see instead.

### 3 — Hard: production simulation

Build the `orderflow` order state machine properly and prove each property.

**Part A — the machine.** Implement `OrderStatus` with eight constants and a
`TRANSITIONS` map as an `EnumMap<OrderStatus, EnumSet<OrderStatus>>`. Add the
static reachability check from Example 2.

**Part B — prove it fails loudly.** Add a ninth constant, `PARTIALLY_REFUNDED`,
and deliberately do **not** wire it into `TRANSITIONS`. Run the application.
Capture:
- the exception type and message on first touch of the class;
- the exception type on the *second* touch;
- why they differ, and which one an on-call engineer would see first in a real
  stack trace.

**Part C — the persistence trap.** Serialize 10,000 orders two ways: once storing
`status.ordinal()`, once storing `status.name()`. Now insert `PARTIALLY_REFUNDED`
in the *middle* of the enum, recompile, and deserialize both files. Report:
- how many orders changed meaning under the ordinal encoding;
- how many under the name encoding;
- what happens if you *rename* a constant instead of reordering.
Then write two sentences on which encoding you would choose for a public API
contract and why it is neither of these.

**Part D — measure the collection choice.** Build the `TRANSITIONS` lookup three
ways: `EnumMap<OrderStatus, EnumSet<OrderStatus>>`,
`HashMap<OrderStatus, HashSet<OrderStatus>>`, and
`HashMap<String, List<String>>`. Run 50 million `canTransition` calls against
each under Flight Recorder:
```bash
java -XX:StartFlightRecording=duration=60s,filename=enum.jfr EnumRun
java -XX:StartFlightRecording=duration=60s,filename=hash.jfr HashRun
java -XX:StartFlightRecording=duration=60s,filename=str.jfr  StringRun
```
Report where the CPU goes in each and what allocates in each.

**Part E — argue the other side.** Given your numbers, is the `EnumMap` version
worth it *on performance grounds alone* for a service handling 1,000 orders per
second? Answer honestly. Then state the reason you would still use `EnumMap`, and
make sure that reason is not about speed.

---

## Interview questions

### Q1 — "Why use EnumSet instead of HashSet for enum values?"

**Mid-level answer:** "`EnumSet` is optimised for enums — it's faster and uses
less memory because it's designed for them."

**Senior answer:** "Because it isn't a hash table at all. For an enum with 64 or
fewer constants you get `RegularEnumSet`, whose entire state is a single `long`
field with one bit per constant, indexed by `ordinal()`. `contains` is
`(elements & (1L << ordinal)) != 0` — one shift, one AND, one compare, no
hashing, no collision handling, no branch on bucket contents. `add` is an OR.
Union, intersection and difference are OR, AND and AND-NOT on that word, so
`retainAll` between two enum sets is one machine instruction rather than an
iteration. Memory is 8 bytes for the whole set regardless of how many elements
are in it, against roughly 48 bytes per element for a `HashSet`. Over 64
constants you get `JumboEnumSet`, which is a `long[]` and the same operations
word by word.

Two properties beyond performance that I care about more. Iteration is in
declaration order, always — so output is deterministic and safe to assert on,
which `HashSet` is not. And `EnumSet.complementOf` gives me 'every constant I
have not accounted for' as a one-liner, which I use for startup invariant checks:
if a new enum constant is not handled, the application fails to boot with the
constant named."

**What separates them:** naming the actual representation and the actual
instructions, and then saying that determinism and the complement operation
matter more than the speed.

**Follow-up:** "Is there any case where `HashSet` beats `EnumSet` for enum
keys?" Essentially no, which is what makes the question interesting. The honest
edges: you need `null` as an element (`EnumSet` rejects it), or you need
thread-safe mutation (neither is safe, but `ConcurrentHashMap.newKeySet()` is).

---

### Q2 — "How does EnumMap work, and why not just use ordinal() as an array index yourself?"

**Mid-level answer:** "`EnumMap` uses the ordinal as an array index internally.
Doing it yourself would work the same way."

**Senior answer:** "It holds a cached copy of the key universe — the `values()`
array — and a parallel `Object[]` with one slot per constant. `get` is
`vals[key.ordinal()]` behind a type check; `put` writes that slot. No hashing, no
resize, no load factor, no collisions, deterministic ordinal-order iteration.

You could hand-roll it, and I would not. What `EnumMap` gives you that a raw
array does not: it is a `Map`, so it composes with every API that takes one —
`Collectors.groupingBy` with an `EnumMap` supplier, `merge`, `computeIfAbsent`,
`forEach`. It type-checks keys, so a constant from a different enum throws rather
than indexing into your array. It tracks `size` correctly and distinguishes
'absent' from 'present with a null value' using an internal sentinel — a raw
array conflates those. And it handles `values()` being a defensive copy that
allocates on each call, which a naive hand-rolled version usually gets wrong by
calling `values()` in a loop.

The one thing I would flag: `EnumMap` depends on ordinals, which is completely
safe in memory because it rebuilds from the current class every run. It is
crossing a persistence or wire boundary with an ordinal that is catastrophic, and
those are separate concerns that people conflate."

**What separates them:** the sentinel for null values, the `values()` allocation
point, and cleanly separating "ordinals in memory are fine" from "ordinals on
disk are a disaster".

**Follow-up:** "So what does `@Enumerated(ORDINAL)` do and why is it the default
in JPA?" It stores the integer, it is the default for historical reasons, and it
is one of the few JPA defaults you should override on sight.

---

### Q3 — "Walk me through choosing a collection for a new piece of code."

**Mid-level answer:** "`ArrayList` for lists, `HashMap` for maps, `HashSet` for
sets, unless you need ordering, in which case `LinkedHashMap` or `TreeMap`."

**Senior answer:** "I ask five questions in order, and the first one rules out
most of the space.

**Is the key space a fixed set of named constants?** If yes, `EnumMap` /
`EnumSet` and the rest of the questions do not apply.

**Do I need duplicates, and does position mean anything?** Positional and
duplicated is a `List` — `ArrayList` unless it is strictly a queue or stack, in
which case `ArrayDeque` beats both `LinkedList` and `Stack`. Unique is a `Set`.
Key-to-value is a `Map`.

**Is iteration order part of the contract?** If a human or another system will
observe the order, it belongs in the type — `LinkedHashMap` / `LinkedHashSet` for
insertion order, `TreeMap` / `TreeSet` for sorted. 'It happens to be stable' is
not a contract, and `HashMap`'s order changes on resize.

**Do I need range or nearest-match queries?** `floorEntry`, `subMap`,
`headSet` — that is `TreeMap` / `TreeSet` and nothing else can do it without a
full scan.

**Is it shared across threads?** If yes, none of the above, and I go to
`ConcurrentHashMap`, `ConcurrentHashMap.newKeySet()`, `ConcurrentSkipListMap` or
`CopyOnWriteArrayList`, each with a very different write-cost profile.

Then two things I check regardless: is it bounded — because an unbounded
collection used as a cache is the most common leak in Java — and are the keys
immutable, because Topic 13's mutable-key bug is silent."

**What separates them:** an ordered decision procedure rather than a list of
classes, putting the enum question first because it eliminates the most cases,
and closing on boundedness and key immutability, which are the two properties
that cause production incidents rather than inefficiency.

**Follow-up:** "You said `ArrayDeque` beats `LinkedList`. Why?" Topic 11:
contiguous array, no per-node allocation, no pointer chasing, far better cache
behaviour. `LinkedList` wins essentially never at realistic sizes.

---

### Q4 — "`set.removeAll(list)` is taking minutes. Why?"

**Mid-level answer:** "Because `List.contains` is O(n), so it's doing a linear
scan for each element."

**Senior answer:** "Right, and the detail worth having is which side it scans.
`AbstractSet.removeAll` picks a strategy from the sizes: if the set is larger
than the argument, it iterates the argument and calls `set.remove`, which is
hashed and fast. Otherwise it iterates the set and calls `argument.contains` on
each element — and if the argument is an `ArrayList` that is a linear scan per
element. So the quadratic case is a *small* set minus a *large* list, which is
counter-intuitive: making the set bigger can make the operation faster.

`retainAll` is worse, because `AbstractCollection.retainAll` has no size
heuristic at all — it always iterates the receiver and calls `contains` on the
argument. So `retainAll` with a `List` argument is quadratic regardless of sizes.

The general rule I apply in review: the cost of a bulk set operation depends on
the `contains` cost of the **argument**, so any `Collection` passed to
`removeAll`, `retainAll` or `containsAll` should be a `Set`. Wrapping in
`new HashSet<>(list)` is one O(n) pass and turns quadratic into linear. In a
profile the tell is time inside `ArrayList.indexOf` under an `AbstractSet` or
`AbstractCollection` frame."

**What separates them:** knowing the size heuristic exists and which side it
protects, knowing `retainAll` lacks it, and stating the rule in a form that
applies without reading the source each time.

**Follow-up:** "What about `list.removeAll(set)`?" `ArrayList.removeAll` iterates
itself once calling `set.contains`, which is O(1) — so that direction is linear.
The asymmetry is the point.

---

### Q5 — "Why does `Set.of(...)` iterate in a different order on every run?"

**Mid-level answer:** "Its order is unspecified, so you shouldn't rely on it. I
didn't know it actually changes between runs."

**Senior answer:** "It is deliberate. `ImmutableCollections` computes a `SALT`
once per JVM from the system clock and uses it to vary both the probe sequence
and the iteration order. The reasoning is that an unspecified order which is
nonetheless *stable* is worse than one that visibly is not — because stable
means code accidentally comes to depend on it, and then a JDK upgrade or an extra
element silently breaks something in production. Randomising per run pushes that
failure into the developer's own test loop, where it is cheap.

It is the same lesson as `HashMap`, which is also unspecified but happens to be
stable for identical input on one JDK version — which is exactly why people
depend on it and get burned on an upgrade. `Set.of` refuses to let you.

Practically: if a test asserts on iteration order of a `Set.of`, that test is
wrong, not flaky. If order is part of the requirement, put it in the type —
`LinkedHashSet` for insertion order, `TreeSet` for sorted, `EnumSet` for
declaration order. And note the other two deliberate strictnesses in the same
family: `Set.of` rejects nulls with an NPE, including on `contains(null)`, and
rejects duplicate elements at construction with an `IllegalArgumentException` —
which in a `static final` field surfaces as `ExceptionInInitializerError` and
then a misleading `NoClassDefFoundError` on every subsequent touch."

**What separates them:** knowing it is a design decision with a stated rationale
rather than an accident, and connecting it to why `HashMap`'s accidental
stability is the more dangerous case.

**Follow-up:** "Have you ever seen a test depend on `HashMap` order?" Everyone
has. The good answer describes catching it in review by noticing an assertion on
a joined string built from a `keySet()`.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A `HashSet` is a `HashMap` with a shared dummy value. Estimate the wasted
   memory per element from that design, then argue whether a purpose-built
   `HashSet` would have been worth writing.

2. `EnumSet` is abstract with two hidden implementations chosen by constant
   count. What does that buy over exposing both, and what does it cost you as a
   user?

3. `EnumSet` and `EnumMap` depend entirely on `ordinal()`, yet persisting
   `ordinal()` is catastrophic. Reconcile those two facts in one sentence.

4. `Set.of` randomises iteration order to prevent accidental dependence.
   `HashMap` does not. Argue that `HashMap` should, then argue that it must not.

5. `AbstractSet.removeAll` has a size heuristic and `AbstractCollection.retainAll`
   does not. Is that an oversight or a decision? Make the case for each.

6. `EnumSet.complementOf` lets you compute "everything I have not handled". Name
   two other places in a codebase where "the complement of what I accounted for"
   would catch a real class of bug, and say what data structure you would need
   there.

7. TypeScript models a fixed state set as a string union that vanishes at
   runtime. Java's `enum` is a runtime type with identity, ordinals and methods.
   Name one thing the TypeScript version does better, and be specific.

---

## Quick reference card

### The decision table

| Requirement | Use |
|---|---|
| enum keys, any purpose | **`EnumMap` / `EnumSet`** — always, no trade-off |
| O(1) lookup, order irrelevant | `HashMap` / `HashSet` |
| insertion order observable | `LinkedHashMap` / `LinkedHashSet` |
| bounded cache with LRU eviction | `LinkedHashMap` access-order (Topic 15), or Caffeine |
| sorted iteration, range/nearest queries | `TreeMap` / `TreeSet` |
| positional, duplicates allowed | `ArrayList` |
| queue or stack | `ArrayDeque` — never `Stack`, rarely `LinkedList` |
| small fixed constant, never mutated | `Set.of` / `Map.of` / `List.of` |
| defensive copy for a getter | `Set.copyOf` / `Map.copyOf` / `List.copyOf` |
| shared across threads | `ConcurrentHashMap`, `ConcurrentHashMap.newKeySet()`, `ConcurrentSkipListMap` (Topic 92) |
| dense small integer keys | `boolean[]`, `BitSet`, or a primitive-collection library |

### EnumSet / EnumMap API

```java
EnumSet.noneOf(OrderStatus.class)
EnumSet.allOf(OrderStatus.class)
EnumSet.of(PAID, SHIPPED)
EnumSet.range(CREATED, PAID)          // inclusive, by ordinal
EnumSet.copyOf(collection)
EnumSet.complementOf(set)             // "everything I did not handle"

set.addAll(other)      // union       (mutates)
set.retainAll(other)   // intersection(mutates)
set.removeAll(other)   // difference  (mutates)
Collections.disjoint(a, b)            // no shared elements

new EnumMap<>(OrderStatus.class)
new EnumMap<>(existingMap)
map.merge(status, 1L, Long::sum)
Collectors.groupingBy(Order::status, () -> new EnumMap<>(OrderStatus.class), counting())
```

### Immutable vs unmodifiable

| | copies? | rejects null? | rejects duplicates? | order |
|---|---|---|---|---|
| `Set.of(...)` | n/a | yes, NPE | yes, `IllegalArgumentException` | randomised per JVM |
| `Set.copyOf(c)` | yes | yes, NPE | silently deduplicates | randomised per JVM |
| `Collections.unmodifiableSet(s)` | **no — a view** | inherits | inherits | inherits |
| `new LinkedHashSet<>(c)` wrapped unmodifiable | yes | inherits | deduplicates | insertion |

### Set sizing (Java 19+, available on 21)

```java
HashSet.newHashSet(n)              // sized for n ELEMENTS, no resize
LinkedHashSet.newLinkedHashSet(n)
HashMap.newHashMap(n)
```
`[LEGACY — still asked]` The old idiom was `new HashSet<>((int)(n / 0.75f) + 1)`.

### Gotchas checklist

- [ ] Enum keys? `EnumMap`/`EnumSet`. Every time. There is no trade-off.
- [ ] Never persist or transmit `ordinal()`. `@Enumerated(STRING)` or an explicit code.
- [ ] `Set.of` iteration order changes between JVM runs. Never assert on it.
- [ ] `Set.of` rejects duplicates at construction — fatal in a `static final` field.
- [ ] Any `Collection` argument to `removeAll`/`retainAll`/`containsAll` should be a `Set`.
- [ ] `retainAll` has no size heuristic. It is quadratic with a `List` argument, always.
- [ ] `Collections.unmodifiableSet` is a view. `Set.copyOf` is a copy.
- [ ] `values()` allocates a fresh array on every call. Cache it or use `EnumSet.allOf`.
- [ ] `==` is correct and preferred for enums. It is the only reference type where that holds.
- [ ] An unbounded `Set` used as a "seen" cache is a leak (Topic 79).
- [ ] `EnumSet`/`EnumMap` are not thread-safe. Neither is anything else in this table except the concurrent ones.

---

## When would I use this at work?

**1. Every state machine, permission set, and feature flag group.**
`orderflow`'s order statuses, a user's roles, which notification channels are
enabled for a customer, which payment methods a region supports. All of these are
"a subset of a fixed named universe", and all of them are `EnumSet`. The moment
you write `Set<String>` for something whose values are enumerable at compile
time, you have given away type safety for nothing.

**2. Catching a forgotten case at startup instead of in production.**
`EnumSet.complementOf(handled)` in a static initialiser turns "someone added a
status and forgot to wire it up" from a silent runtime hole into a boot failure
with the constant named in the message. That single pattern has more incident-
prevention value than anything else in this document, and it costs six lines.

**3. Reviewing a `removeAll` or a `retainAll`.**
You see `set.retainAll(someList)` in a PR. You ask what the list's size is. If it
is large, that line is quadratic and the author does not know it. The fix is one
`new HashSet<>(...)` wrapper. This is a thirty-second review comment that has
prevented real outages.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and boxing:** why `EnumMap<Status, Long>` still boxes its
  *values* even though its keys are free, and why `merge(k, 1L, Long::sum)` is the
  counter idiom.
- **06 — Type erasure:** why `EnumSet.noneOf(OrderStatus.class)` needs the `Class`
  object at all — erasure means the set cannot discover its own element type at
  runtime without being told.
- **10 — Collections Framework:** the `Set` and `Map` contracts, fail-fast
  iteration, and the `AbstractSet`/`AbstractCollection` base classes whose bulk
  operations produce Trap 2.
- **12 — HashMap internals:** `HashSet` is a `HashMap`. Everything about
  spreading, buckets and resize applies unchanged.
- **13 — equals/hashCode:** unchanged for `HashSet`; irrelevant for `EnumSet`,
  because enum identity is guaranteed by the language.
- **14 — Comparable/TreeMap:** `TreeSet` is a `TreeMap`, with the same
  comparator-defines-identity trap.
- **15 — LinkedHashMap:** `LinkedHashSet` is a `LinkedHashMap`.

**This unlocks:**
- **17 — Immutability:** `Set.copyOf` versus `unmodifiableSet`, defensive copying
  at the constructor and the getter, and safe publication of a shared `EnumSet`.
- **19 — Serialization:** why `EnumSet` uses a serialization proxy, and why that
  makes it safe across an enum reordering when a persisted ordinal is not.
- **24 — Collectors:** `groupingBy` with an `EnumMap` supplier, and
  `toUnmodifiableSet`.
- **28/29 — Sealed types and pattern matching:** exhaustive `switch` over an enum
  is the compile-time counterpart to the runtime `complementOf` check here.
- **47/48 — Spring Data and Hibernate:** `@Enumerated(STRING)` versus `ORDINAL`,
  and entity `Set` associations where Topic 13's traps become Hibernate bugs.
- **67 — Class initialization:** `ExceptionInInitializerError` followed by a
  misleading `NoClassDefFoundError` — exactly what Trap 5's duplicate in a
  `static final Set.of` produces.
- **79 — Memory leaks:** the unbounded "seen" set, which is the same leak shape as
  the unbounded cache.
- **92 — Concurrent collections:** `ConcurrentHashMap.newKeySet()`,
  `CopyOnWriteArraySet` and why none of the collections in this document are safe
  to share.

---

*Java baseline 21. `EnumSet`, `EnumMap` and the `Set` hierarchy have been stable
since Java 5; the immutable `Set.of` / `Map.of` factories arrived in Java 9 and
`Set.copyOf` in Java 10. `HashSet.newHashSet(n)` and
`LinkedHashSet.newLinkedHashSet(n)` are Java 19+, so available on this baseline
but not on Java 17. `LinkedHashSet` gained the `SequencedSet` methods
(`getFirst`, `addFirst`, `reversed`) in Java 21 — see Topic 15. Nothing in this
topic differs between Java 21 and Java 25. Project Valhalla may eventually make
value-based collections cheaper, but as of Java 25 it has not landed, so plan
around what exists.*
