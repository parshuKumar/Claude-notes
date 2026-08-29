# 27 — Records — Semantics, equals/hashCode, Compact Constructors, Serialization

## Phase: 2 — Modern Java
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (spine starts at Topic 35)

---

## ELI5 anchor

Think of a **shipping label** stuck on a parcel.

The label has fixed printed boxes: recipient, address, weight. Once printed, you
cannot rub out the weight and write a new one — the label is done.

Two parcels with **identical labels** are, for the purposes of the depot's sorting
machine, the same shipment. The machine does not care that they are two different
pieces of paper. It reads the boxes.

That is a **record**: a fixed set of boxes, filled in once, and two records with the
same box contents count as equal.

Now the trap, and it is the entire headline of this document.

One of the boxes says **"contents: see the attached inventory sheet"**. The label is
still unchangeable. But somebody can walk over to the inventory sheet and cross out
"12 widgets" and write "3 widgets". The label never changed. What the label *points at*
changed.

A record freezes the boxes. It does not freeze what is inside the things the boxes
point to. That is **shallow immutability**, and it is where people get hurt.

---

## The bridge from what you know

### What you write today

```ts
interface OrderLine {
  readonly sku: string;
  readonly quantity: number;
  readonly unitPriceMinorUnits: number;
}

const a: OrderLine = { sku: "SKU-1001", quantity: 2, unitPriceMinorUnits: 1999 };
const b: OrderLine = { sku: "SKU-1001", quantity: 2, unitPriceMinorUnits: 1999 };

a === b;                                  // false — different objects
JSON.stringify(a) === JSON.stringify(b);  // true, but that is a hack, not equality
```

You have no built-in value equality in JavaScript. You either write a comparison
function, use a library, or stringify and hope.

### The Java version

```java
public record OrderLine(String sku, long quantity, long unitPriceMinorUnits) { }
```

One line. And now:

```java
var a = new OrderLine("SKU-1001", 2, 1999);
var b = new OrderLine("SKU-1001", 2, 1999);

a == b;         // false — still two objects
a.equals(b);    // TRUE — generated, component by component
a.hashCode() == b.hashCode();   // true
a.toString();   // OrderLine[sku=SKU-1001, quantity=2, unitPriceMinorUnits=1999]
```

You did not write `equals`, `hashCode`, `toString`, the constructor, or the accessors.
The compiler generated all of them.

### The verdict

| TypeScript | Java record | Verdict |
|---|---|---|
| `interface` / `type` used as a data carrier | `record` | **PARTIAL** — same *role*, three concrete differences below. |
| Structural: any object with the right shape fits | Nominal: only an `OrderLine` is an `OrderLine` | **NO ANALOGUE** — a differently-named record with identical components is a different, incompatible type. This is Topic 02. |
| No value equality; you compare by hand | Generated component-wise `equals`/`hashCode` | **NO ANALOGUE** — you gain something JavaScript simply does not have. |
| `readonly` is compile-time only and shallow | `final` fields, also shallow | **HONEST ANALOGUE** — both are shallow, and both let you mutate what they point at. Your `readonly` instinct transfers exactly, including its blind spot. |
| Object literal `{ sku: "x", quantity: 2 }` | `new OrderLine("x", 2, 1999)` | **PARTIAL** — positional, not named. No optional fields. All components required. |
| Spread update `{ ...a, quantity: 3 }` | `new OrderLine(a.sku(), 3, a.unitPriceMinorUnits())` | **PARTIAL** — Java has no spread. Withers were proposed and are not in 21 or 25. You write it out or you write a builder. |
| `Object.freeze(a)` (shallow) | records (shallow) | **HONEST ANALOGUE** — including that both stop at one level deep. |

**The one you must internalise:** your `readonly` blind spot in TypeScript — that
`readonly items: Item[]` still lets someone call `items.push(...)` unless you typed it
`readonly Item[]` — is **exactly** the Java record trap, for exactly the same reason.
You already have the instinct. It just fires less often in TS because `readonly T[]`
exists and Java has no equivalent modifier.

---

## What is this?

A **record** is a class whose state is its declaration.

```java
public record OrderLine(String sku, long quantity, long unitPriceMinorUnits) { }
```

Those three things in the header are the **record components**. From them, the compiler
generates:

| Generated member | What it is |
|---|---|
| `private final String sku;` etc. | One private final field per component. |
| `public OrderLine(String, long, long)` | The **canonical constructor** — takes every component, in order. |
| `public String sku()` | An accessor per component. **No `get` prefix.** |
| `public boolean equals(Object)` | Component-by-component value equality. |
| `public int hashCode()` | Derived from all components. |
| `public String toString()` | `OrderLine[sku=SKU-1001, quantity=2, ...]` |

And the compiler imposes these rules:

- The record is implicitly `final`. You cannot subclass it.
- It implicitly extends `java.lang.Record`. It cannot extend anything else.
- It **can** implement interfaces. (This is what Topic 28 builds on.)
- It cannot declare additional **instance** fields. Static fields are fine.
- Its components' fields are `final` and cannot be reassigned.
- A nested record is implicitly `static`.

Records became final in Java 16. On a Java 21 baseline they are ordinary, boring,
fully-supported language. Nothing here is preview.

---

## Why does it matter?

**1. It removes the single largest source of `equals`/`hashCode` bugs.**
Topic 13 showed you how a hand-written `equals` that disagrees with `hashCode`
silently loses entries in a `HashMap`. Records generate both, together, from the same
component list, every time you add a component. The class of bug where somebody adds a
field and forgets to update `equals` becomes impossible.

**2. It makes "this is data, not behaviour" enforceable rather than a comment.**
A record cannot grow mutable state. Someone cannot quietly add a cache field to your
DTO six months later, because the compiler rejects it.

**3. It is the building block for Java's algebraic data types.**
`sealed interface` + records + pattern-matching `switch` is the honest analogue of your
TypeScript discriminated unions. That is Topics 28 and 29, and neither works without
this one.

**4. It fixes deserialization.**
Java serialization historically bypassed your constructor entirely — objects came back
from a stream without any validation running. Records do not: deserialization goes
**through the canonical constructor**, so your invariants hold on the way in. That is a
genuinely significant change and it is Topic 19's other half.

**5. And the reason it can hurt you:** all of that immutability is one level deep. A
record with a `List<OrderLine>` component is a mutable object wearing an immutable
label.

---

## Syntax breakdown

### The record declaration itself

```java
public record OrderLine(String sku, long quantity, long unitPriceMinorUnits) { }
//     ^^^^^^ ^^^^^^^^^ ^-------------- record components ---------------^  ^^^
//       |        |                                                         |
//       |     the name                                       the body (may be empty)
//    the keyword
```

| Bit of syntax | What it means |
|---|---|
| `record` | A restricted class declaration. Not a keyword everywhere — it is a *contextual* keyword, so a variable named `record` still compiles. |
| `(String sku, ...)` | The **header**. This is the state description. Everything is generated from it. |
| `{ }` | The body. Legal to leave empty. Put static factories, extra constructors, derived methods and overrides here. |

Accessing components:

```java
OrderLine line = new OrderLine("SKU-1001", 2, 1999);
line.sku();          // NOT line.getSku()
line.quantity();
```

The accessor is named after the component, with no prefix. This is deliberate — the
JavaBeans `getX` convention exists for tooling, and records were designed to state
what they are rather than to pretend to be beans.

### The compact constructor — the one genuinely new construct

This is the piece with no TypeScript equivalent, so read it twice.

```java
public record OrderLine(String sku, long quantity, long unitPriceMinorUnits) {

    public OrderLine {                    // <-- NO PARAMETER LIST. That is the whole point.
        Objects.requireNonNull(sku, "sku");
        if (quantity <= 0) {
            throw new IllegalArgumentException("quantity must be positive, was " + quantity);
        }
        sku = sku.trim().toUpperCase();   // reassigning the PARAMETER, not the field
    }
}
```

Read that carefully:

| Bit of syntax | What it means |
|---|---|
| `public OrderLine {` | A **compact canonical constructor**. No parentheses, no parameter list. The parameters are implicitly the record components, with the same names. |
| `sku`, `quantity` inside the body | These are the **constructor parameters**, not the fields. The fields do not exist yet. |
| `sku = sku.trim().toUpperCase();` | Legal, and idiomatic: you are **normalising the parameter**. |
| The end of the body | The compiler inserts `this.sku = sku; this.quantity = quantity; ...` automatically, using the parameters' *final* values. |

**Two rules that catch everyone:**

1. **You must not write `this.sku = sku;` in a compact constructor.** It is a compile
   error: `cannot assign a value to final variable sku`... actually the error you get
   is that you cannot assign to the field in a compact constructor, because the
   assignment is generated for you. Just assign to the parameter.
2. **You cannot return early** from a compact constructor in a way that skips the
   generated assignments — there is no way to skip them, which is exactly the property
   that makes them trustworthy.

### The explicit canonical constructor (when you need the long form)

```java
public record OrderLine(String sku, long quantity, List<String> tags) {

    public OrderLine(String sku, long quantity, List<String> tags) {   // full signature
        this.sku = Objects.requireNonNull(sku);
        this.quantity = quantity;
        this.tags = List.copyOf(tags);        // defensive copy - Topic 17
    }
}
```

Use the explicit form when you need to assign something *different* to a field than
what the parameter holds — which in practice means **defensive copying**. Use the
compact form for validation and normalisation. In the compact form you can also do the
copy by reassigning the parameter (`tags = List.copyOf(tags);`), and most people find
that clearer. Both work.

### Additional constructors must delegate

```java
public record OrderLine(String sku, long quantity, long unitPriceMinorUnits) {

    public OrderLine(String sku) {
        this(sku, 1, 0);          // MUST delegate to the canonical constructor
    }
}
```

Every constructor path ends at the canonical constructor. That is what guarantees your
validation cannot be bypassed — including by deserialization.

### Static factories, derived methods, overrides

```java
public record Money(long minorUnits, String currency) implements Comparable<Money> {

    public static Money zero(String currency) { return new Money(0, currency); }

    public Money plus(Money other) {                      // derived operation
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("currency mismatch");
        }
        return new Money(minorUnits + other.minorUnits, currency);   // returns a NEW record
    }

    public BigDecimal asMajorUnits() {                    // derived value, not stored
        return BigDecimal.valueOf(minorUnits, 2);
    }

    @Override public int compareTo(Money other) {
        return Long.compare(minorUnits, other.minorUnits);
    }

    @Override public String toString() {                  // overriding a generated member
        return asMajorUnits() + " " + currency;
    }
}
```

Note `plus` returns a new `Money`. That is the whole style: **transformations produce
new instances**. Same as your `{...a, quantity: 3}` habit, without the spread operator.

### What `equals` actually does

The generated `equals` compares, component by component:

- **Reference components** with `Objects.equals(a, b)` — so it calls *their* `equals`.
- **Primitive components** with `==` semantics — **except `float` and `double`, which
  use `Float.compare` / `Double.compare`.** That difference matters: it means
  `Double.NaN` equals `Double.NaN` inside a record, and `+0.0` does **not** equal
  `-0.0`. Both are the opposite of what bare `==` does.

The generated `hashCode` combines all components. **The exact algorithm is deliberately
unspecified.** Do not persist a record's `hashCode`, do not send it across a wire, and
do not assume it is stable between JVM versions. (Same rule as `String.hashCode`? No —
`String.hashCode` *is* specified. Records are not. Know the difference.)

### Local records

```java
public List<Report> summarise(List<Order> orders) {
    record Bucket(String status, long count) { }        // declared inside a method

    return orders.stream()
            .collect(groupingBy(o -> o.status().name(), counting()))
            .entrySet().stream()
            .map(e -> new Bucket(e.getKey(), e.getValue()))
            .map(b -> new Report(b.status(), b.count()))
            .toList();
}
```

A record declared inside a method. Perfect for the intermediate tuple you need in the
middle of a stream pipeline (Topics 23–24) and nowhere else. Before records, people
used `Map.Entry` or `Object[]` for this, and both were awful.

---

## Example 1 — minimal

```java
import java.util.*;

public record Point(int x, int y) {

    public Point {
        if (x < 0 || y < 0) {
            throw new IllegalArgumentException("negative coordinate: " + x + "," + y);
        }
    }

    public static void main(String[] args) {
        Point a = new Point(3, 4);
        Point b = new Point(3, 4);

        System.out.println(a);                    // Point[x=3, y=4]
        System.out.println("a == b      : " + (a == b));
        System.out.println("a.equals(b) : " + a.equals(b));

        Set<Point> seen = new HashSet<>();
        seen.add(a);
        seen.add(b);
        System.out.println("set size    : " + seen.size());   // 1, not 2

        new Point(-1, 0);   // throws from the compact constructor
    }
}
```

Run it. The set has **one** element even though you added two objects. That is
generated `equals`/`hashCode` doing exactly what Topic 13 told you to do by hand, with
no chance of getting it wrong.

---

## Example 2 — production scenario

`orderflow` needs a value object for money and a DTO for the order-placement request.
This is the shape you will actually write.

### The version that looks right and is not

```java
public record PlaceOrderCommand(
        UserId userId,
        List<OrderLine> lines,
        String idempotencyKey) {
}
```

Clean. One line. Immutable, apparently. And here is what happens:

```java
List<OrderLine> lines = new ArrayList<>();
lines.add(new OrderLine("SKU-1001", 2, 1999));

PlaceOrderCommand cmd = new PlaceOrderCommand(userId, lines, "idem-abc");

validator.validate(cmd);           // passes: total 3998, within the customer's limit

lines.add(new OrderLine("SKU-9999", 500, 999_99));   // caller still holds the list

orderService.place(cmd);           // places an order for 500 extra units
```

The record was never modified. `cmd.lines()` is the *same list object* the caller
still has a reference to. Validation ran against one state; execution ran against
another. This is a **time-of-check to time-of-use** bug, and in a payment path it is a
security issue, not just a correctness one.

It gets worse in the other direction:

```java
PlaceOrderCommand cmd = repository.loadPendingCommand(id);
cmd.lines().clear();               // a downstream caller "cleans up"
// the repository's cached copy is now empty
```

### The corrected version

```java
public record PlaceOrderCommand(
        UserId userId,
        List<OrderLine> lines,
        String idempotencyKey) {

    public PlaceOrderCommand {
        Objects.requireNonNull(userId, "userId");
        Objects.requireNonNull(idempotencyKey, "idempotencyKey");
        if (idempotencyKey.isBlank()) {
            throw new IllegalArgumentException("idempotencyKey must not be blank");
        }
        if (lines == null || lines.isEmpty()) {
            throw new IllegalArgumentException("an order must have at least one line");
        }
        lines = List.copyOf(lines);        // <-- the fix. Copy, and make it unmodifiable.
    }
}
```

`List.copyOf` does two jobs in one call:

1. It **copies**, so the caller's later mutations cannot reach inside the record.
2. It returns an **unmodifiable** list, so anyone who calls `cmd.lines().add(...)` gets
   `UnsupportedOperationException` at that line instead of silently corrupting state
   somewhere else.

Note it also rejects `null` elements, which is usually what you want. If you genuinely
need nulls inside, use
`Collections.unmodifiableList(new ArrayList<>(lines))` instead.

> `List.copyOf` on a list that is *already* an immutable `List.of(...)` returns the
> same instance rather than copying. So the defensive copy is free on the common path.
> That is a deliberate optimisation, not an accident.

### The full value object

```java
public record Money(long minorUnits, Currency currency) implements Comparable<Money> {

    public Money {
        Objects.requireNonNull(currency, "currency");
    }

    public static Money of(long minorUnits, String isoCode) {
        return new Money(minorUnits, Currency.getInstance(isoCode));
    }

    public Money plus(Money other) {
        requireSameCurrency(other);
        return new Money(Math.addExact(minorUnits, other.minorUnits), currency);
    }

    public Money times(long factor) {
        return new Money(Math.multiplyExact(minorUnits, factor), currency);
    }

    private void requireSameCurrency(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "cannot combine " + currency + " and " + other.currency);
        }
    }

    @Override public int compareTo(Money other) {
        requireSameCurrency(other);
        return Long.compare(minorUnits, other.minorUnits);
    }
}
```

Points worth naming:

- `long minorUnits`, not `double`. Topic 01, Trap 5.
- `Math.addExact` throws on overflow instead of wrapping silently. A record that can
  silently produce a negative total after overflow is worse than one that throws.
- `compareTo` and `equals` agree here (both use all state that matters), which is the
  `Comparable` consistency requirement from Topic 14. If they disagreed, `Money` in a
  `TreeSet` would behave differently from `Money` in a `HashSet`.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a record wrapping a mutable collection (the headline)

**Wrong:**
```java
public record Order(OrderId id, List<OrderLine> lines) { }
```

**Exact symptom:** intermittent and horrible. Pick your flavour:

- A validated order is placed with different contents than were validated. Support
  tickets say "I was charged for items I did not order."
- `HashSet<Order>` loses an element. You added it, `contains()` returns `false`,
  `size()` still counts it. This is exactly Topic 13's mutable-key bug: the record's
  `hashCode` includes `lines.hashCode()`, mutating the list changes the hash, and the
  entry is now in the wrong bucket — unreachable but still retained.
- A `ConcurrentModificationException` from a thread iterating the record's list while
  another thread mutates it, which is confusing because the record "is immutable".
- No exception at all, and a nightly reconciliation report that never balances.

**Root cause:** `final` is shallow. Topic 17 said this and it is worth repeating: a
`final` field prevents reassigning the *reference*. It says nothing about the object at
the other end. A record makes all its component fields `final` and nothing more.

```java
Order order = new Order(id, mutableList);
order.lines().add(extraLine);      // legal, compiles, mutates the "immutable" record
```

**Fix — copy on the way in, and use an unmodifiable type:**
```java
public record Order(OrderId id, List<OrderLine> lines) {
    public Order {
        lines = List.copyOf(lines);
    }
}
```

**Fix if you also expose it elsewhere** — the accessor is already safe once the stored
list is unmodifiable, so you do not need a second copy on the way out. That is a
difference from a hand-written class, where Topic 17 told you to copy at both ends: if
the *stored* object is genuinely immutable, one copy is enough.

**What you cannot fix this way:** components whose type is mutable and has no immutable
form.

```java
public record AuditEvent(Instant at, byte[] payload) { }   // byte[] is mutable, always
```

`byte[]` has no `copyOf`-and-freeze. You must copy in the constructor **and** copy in
the accessor:

```java
public record AuditEvent(Instant at, byte[] payload) {
    public AuditEvent {
        payload = payload.clone();
    }
    @Override public byte[] payload() {
        return payload.clone();
    }
}
```

And now you have a second, worse problem — see Trap 3.

**The rule:** every record component must be one of (a) a primitive, (b) an immutable
type (`String`, `Instant`, `UUID`, `BigDecimal`, another record made of these, an
enum), or (c) defensively copied into an unmodifiable form in the canonical
constructor. If it is none of those, your record is a mutable object with a misleading
declaration.

---

### Trap 2 — using a record as a JPA entity

**Wrong:**
```java
@Entity
@Table(name = "orders")
public record Order(@Id UUID id, UserId userId, Instant placedAt) { }
```

**Exact symptom:** it fails at application startup, not at query time — which is at
least merciful. You get something in this family:

```
org.hibernate.AnnotationException: Entity 'com.orderflow.orders.Order'
  has no identifier
```
or
```
org.hibernate.InstantiationException: No default constructor for entity:
  com.orderflow.orders.Order
```
or, in older/other stacks, a `PersistenceException` about the class not being a valid
entity. The exact wording depends on your Hibernate version, so do not memorise the
string — memorise the shape: **the context fails to start, complaining about
construction or identity.**

**Root cause:** three independent, unfixable reasons. Not one.

1. **JPA requires a no-argument constructor** (public or protected). A record only has
   the canonical constructor. You cannot add a no-arg one, because every record
   constructor must delegate to the canonical one, and the canonical one needs all
   components.
2. **JPA requires mutability.** Hibernate loads a row by constructing the entity and
   then *setting* its fields reflectively; it also needs to write into fields when
   refreshing, merging, and resolving lazy proxies. Record fields are `final`.
3. **Hibernate's dirty checking (Topic 48) is built on comparing a managed entity
   against a snapshot and issuing UPDATEs for changed fields.** An entity that cannot
   change has nothing to dirty-check, and an entity you can never modify cannot be
   updated at all. The whole persistence-context model assumes mutable identity-bearing
   objects.

There is also a fourth, subtler reason: **JPA entity equality is identity-based**, not
value-based. Two `Order` rows with the same column values but different primary keys
are different entities. A record's generated `equals` compares all components and
would call them equal — or, if you include the id, would call a not-yet-persisted
entity (id `null`) equal to another not-yet-persisted entity. Both behaviours break
`Set<Order>` inside a persistence context.

**Fix — records are for the layer *around* the entity, not the entity:**

```java
// The entity: a normal mutable class. Boring on purpose.
@Entity
@Table(name = "orders")
public class OrderEntity {
    @Id private UUID id;
    private UUID userId;
    private Instant placedAt;
    protected OrderEntity() { }        // JPA's no-arg constructor
    // getters, setters, equals/hashCode on the business key only
}

// The record: the DTO / projection / domain value that leaves the persistence layer.
public record OrderSummary(UUID id, UUID userId, Instant placedAt, Money total) { }
```

And records are **excellent** as query projections:

```java
public interface OrderRepository extends JpaRepository<OrderEntity, UUID> {

    @Query("""
           select new com.orderflow.orders.OrderSummary(o.id, o.userId, o.placedAt, o.total)
           from OrderEntity o
           where o.userId = :userId
           """)
    List<OrderSummary> findSummariesFor(@Param("userId") UUID userId);
}
```

That constructor-expression projection selects only the columns you need, never
attaches anything to the persistence context, and hands your service layer an
immutable value. It is the fix for the "returning entities from read endpoints" problem
in Topic 47.

> Honesty flag: Hibernate 6 added support for records as `@Embeddable` types (via
> instantiators), which is a genuinely different case from records as `@Entity`. Whether
> your exact Hibernate version supports it, and how well, I am not going to assert.
> Records as `@Entity` is settled — it does not work and cannot. Records as
> `@Embeddable` — try it on your version and read the startup log. If it starts and a
> round-trip preserves the values, it works.

---

### Trap 3 — an array component

**Wrong:**
```java
public record AuditEvent(Instant at, byte[] payload) { }
```

**Exact symptom:**
```java
var a = new AuditEvent(now, new byte[]{1, 2, 3});
var b = new AuditEvent(now, new byte[]{1, 2, 3});

a.equals(b);          // FALSE
a.toString();         // AuditEvent[at=..., payload=[B@6d06d69c]
```

Deduplication silently stops working. A `Set<AuditEvent>` accumulates duplicates
forever, which shows up as unbounded memory growth in a long-running consumer. Logs
contain `[B@6d06d69c` where you expected content.

**Root cause:** the generated `equals` uses `Objects.equals` on reference components,
which calls `byte[].equals` — and arrays inherit `Object.equals`, which is reference
identity. Same for `hashCode` and `toString`. The compiler is doing exactly what it
promised; arrays just do not have value semantics in Java, ever.

**Fix — pick one:**

```java
// 1. Best: do not use an array. Wrap it in a type with real value semantics.
public record AuditEvent(Instant at, Payload payload) { }
public record Payload(String base64) { }        // or a ByteBuffer wrapper, or a String

// 2. If you must hold bytes: override all three, and copy at both boundaries.
public record AuditEvent(Instant at, byte[] payload) {

    public AuditEvent {
        payload = payload.clone();
    }

    @Override public byte[] payload() { return payload.clone(); }

    @Override public boolean equals(Object o) {
        return o instanceof AuditEvent(Instant otherAt, byte[] otherPayload)
            && at.equals(otherAt)
            && Arrays.equals(payload, otherPayload);
    }

    @Override public int hashCode() {
        return 31 * at.hashCode() + Arrays.hashCode(payload);
    }

    @Override public String toString() {
        return "AuditEvent[at=" + at + ", payload=" + payload.length + " bytes]";
    }
}
```

Notice that option 2 is four overrides and a defensive copy on every accessor call —
which is most of what a record was supposed to save you. That is the signal: **if you
are overriding `equals`, `hashCode`, `toString` and an accessor, the record is fighting
you.** Take option 1.

(The `o instanceof AuditEvent(Instant otherAt, byte[] otherPayload)` line is a record
deconstruction pattern. That is Topic 29 — it is shown here so you recognise it.)

---

### Trap 4 — `toString` leaking secrets into logs

**Wrong:**
```java
public record PaymentInstruction(
        UUID orderId,
        String cardNumber,
        String cvv,
        Money amount) { }
```

then, anywhere:
```java
log.info("processing {}", instruction);
```

**Exact symptom:** your log aggregator, your APM traces, and your on-call engineer's
terminal all contain:
```
processing PaymentInstruction[orderId=..., cardNumber=4111111111111111, cvv=123, amount=...]
```

You find this during a PCI audit, or when someone greps logs for a support ticket. It
is a reportable data incident, and the fix is not "delete the logs" — the data has
already been shipped to three retention systems.

**Root cause:** the generated `toString` includes **every** component. That is the
documented behaviour and it is usually helpful, which is precisely why nobody thinks
about it. It also fires implicitly: string concatenation, `String.format`, SLF4J's
`{}` placeholder, exception messages, and the debugger's variable view all call it.

**Fix:**
```java
public record PaymentInstruction(
        UUID orderId,
        String cardNumber,
        String cvv,
        Money amount) {

    @Override public String toString() {
        return "PaymentInstruction[orderId=" + orderId
             + ", cardNumber=****" + cardNumber.substring(cardNumber.length() - 4)
             + ", amount=" + amount + "]";
    }
}
```

**Better fix — make it impossible instead of remembering:**
```java
public record PaymentInstruction(UUID orderId, MaskedCard card, Money amount) { }

public record MaskedCard(String last4, String token) {
    // the full PAN never enters the process in this form
}
```

**Code review rule:** any record that holds a credential, token, PAN, national ID, or
personal contact detail gets an explicit `toString` override, and the review checklist
item is "does the generated `toString` print something we would not put in a log?"
This applies to `equals` too in one direction — a `toString` you forgot is a leak; a
`hashCode` over a secret is not, but do not persist it either.

---

### Trap 5 — expecting a record to survive framework binding without checking

**Wrong (the assumption, not the code):** "records are just classes, so my JSON
deserialization, my `@ConfigurationProperties` binding and my form binding will all
work."

**Exact symptom, when it does not:**
```
com.fasterxml.jackson.databind.exc.InvalidDefinitionException:
  Cannot construct instance of `com.orderflow.orders.PlaceOrderCommand`
  (no Creators, like default constructor, exist)
```
or a successfully constructed object where **every component is null or zero** — which
is worse, because it does not throw. You get a 200 response and a row full of nulls.

**Root cause:** a record has no no-arg constructor and no setters, so any framework
that binds by "construct then set" fails. Frameworks that bind by "call a constructor
with named arguments" need to know the *names* of the constructor parameters, and
Java's default compilation discards parameter names.

Two separate mechanisms are in play, and knowing which one applies saves you an hour:

- **For records specifically**, Jackson 2.12 and later use the reflection API for record
  components (`Class.getRecordComponents()`), which gives real component names without
  any compiler flag. So modern Jackson handles records out of the box.
- **For ordinary classes** (and for some other binders), you need `-parameters` on the
  compiler so names survive into the bytecode. Spring Boot's Maven and Gradle plugins
  set this for you; a hand-rolled build often does not.

> Honesty flag: which of those two paths your stack is on depends on your Jackson
> version, your Boot version, and your build plugin configuration. I will not guess.
> The Hands-on proof below has the exact command that settles it for your project.

**Fix:**
1. Confirm `-parameters` is on. In Maven, the `maven-compiler-plugin` needs
   `<parameters>true</parameters>`; Spring Boot's parent POM sets it.
2. If you cannot, annotate explicitly:
   ```java
   public record PlaceOrderCommand(
           @JsonProperty("userId") UUID userId,
           @JsonProperty("lines") List<OrderLine> lines,
           @JsonProperty("idempotencyKey") String idempotencyKey) { }
   ```
3. Write one round-trip test per DTO record — serialize, deserialize, assert equality.
   With generated `equals`, that assertion is one line and it catches the entire class
   of binding failure.

---

## Hands-on proof

Every command below is one **you** run. I do not have a JVM and will not print output
and claim it is real. What follows is precisely what to look for and how to read it.

### Setup

```bash
mkdir -p ~/java-lab/27 && cd ~/java-lab/27
java --version      # expect 21 or 25
```

### Proof 1 — see everything the compiler generated

`OrderLine.java`:
```java
import java.util.Objects;

public record OrderLine(String sku, long quantity, long unitPriceMinorUnits) {
    public OrderLine {
        Objects.requireNonNull(sku);
        if (quantity <= 0) throw new IllegalArgumentException("quantity");
    }
}
```

```bash
javac OrderLine.java
javap -p OrderLine.class
```

**What to look for** in the listing:

| What you see | What it means |
|---|---|
| `public final class OrderLine extends java.lang.Record` | Records are `final` and extend `Record`. You cannot subclass one, and you cannot make one extend your own base class. |
| `private final java.lang.String sku;` and two `private final long` fields | One private final field per component. This is the "shallow immutability" you are about to test in Proof 3. |
| `public OrderLine(java.lang.String, long, long);` | The canonical constructor. Note it takes **all** components, in declaration order. Your compact constructor became this. |
| `public java.lang.String sku();` (no `get` prefix) | The generated accessor. |
| `public final java.lang.String toString();` / `public final int hashCode();` / `public final boolean equals(java.lang.Object);` | Generated, and note they are `final` on the *class file* level in the sense that the record itself is final. You can still override them in source. |

If any of those are missing, you compiled something other than a record — check for a
stray `class` keyword.

### Proof 2 — where `equals`/`hashCode`/`toString` actually come from

```bash
javap -c OrderLine.class
```

**What to look for** in the bodies of `equals`, `hashCode` and `toString`:

- An **`invokedynamic`** instruction, not a sequence of field loads and comparisons.
- In the constant-pool / BootstrapMethods section at the bottom of the listing, a
  bootstrap method referencing **`java/lang/runtime/ObjectMethods.bootstrap`**.

| What you see | What it means |
|---|---|
| `invokedynamic` in all three methods, plus an `ObjectMethods.bootstrap` entry | Expected. The compiler did **not** inline three method bodies. It emitted one indy call site per method, and `ObjectMethods.bootstrap` builds the actual implementation at first execution from a list of component getters. |
| A `BootstrapMethods:` section naming the component names as a single string like `sku;quantity;unitPriceMinorUnits` and method handles for each accessor | This is the recipe. It is why adding a component automatically updates all three methods — there is one source of truth. |
| Hand-written-looking bytecode with `getfield` and `if_acmpne` | You are looking at a normal class, or you overrode the method in source. |

**How to read the significance:** this is the same `invokedynamic` machinery as lambdas
(Topic 21) and string concatenation (Topic 18). Java increasingly moves "generate the
obvious code" out of `javac` and into a runtime bootstrap, so the strategy can improve
without recompiling your code. It also means you cannot read the equality algorithm out
of the bytecode — you read it out of the spec.

### Proof 3 — shallow immutability, demonstrated

`Shallow.java`:
```java
import java.util.*;

public class Shallow {
    record Order(String id, List<String> skus) { }

    record SafeOrder(String id, List<String> skus) {
        SafeOrder { skus = List.copyOf(skus); }
    }

    public static void main(String[] args) {
        List<String> skus = new ArrayList<>(List.of("SKU-1001"));

        Order unsafe = new Order("o-1", skus);
        System.out.println("before  : " + unsafe);
        skus.add("SKU-9999");                       // mutate the caller's list
        System.out.println("after   : " + unsafe);  // the "immutable" record changed

        // hashCode instability
        Set<Order> set = new HashSet<>();
        List<String> k = new ArrayList<>(List.of("A"));
        Order key = new Order("o-2", k);
        set.add(key);
        System.out.println("contains before mutate : " + set.contains(key));
        k.add("B");
        System.out.println("contains after  mutate : " + set.contains(key));
        System.out.println("set size still         : " + set.size());

        // the fixed version
        List<String> safeSkus = new ArrayList<>(List.of("SKU-1001"));
        SafeOrder safe = new SafeOrder("o-3", safeSkus);
        safeSkus.add("SKU-9999");
        System.out.println("safe    : " + safe);
        try {
            safe.skus().add("SKU-8888");
        } catch (UnsupportedOperationException e) {
            System.out.println("accessor list is unmodifiable: " + e.getClass().getName());
        }
    }
}
```

```bash
java Shallow.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `after` line contains `SKU-9999` while `before` did not | The headline proved. The record was never reassigned; the object it points at changed. |
| `contains before mutate : true` then `contains after mutate : false`, with `set size still : 1` | Topic 13's mutable-key corruption, reproduced through a record. The entry is unreachable but still occupying memory. This is a leak *and* a correctness bug. |
| `safe` line does **not** contain `SKU-9999` | `List.copyOf` in the compact constructor severed the link. |
| `accessor list is unmodifiable: java.lang.UnsupportedOperationException` | Confirmed: the stored list rejects mutation, so no second defensive copy is needed on the way out. |

### Proof 4 — deserialization runs your compact constructor

This is the claim from Topic 19 worth verifying yourself: records restore through the
canonical constructor, so validation cannot be bypassed.

`SerialRecord.java`:
```java
import java.io.*;

public class SerialRecord {

    record Quantity(long value) implements Serializable {
        Quantity {
            System.out.println("  >> compact constructor RAN with value=" + value);
            if (value <= 0) throw new IllegalArgumentException("must be positive: " + value);
        }
    }

    static class LegacyQuantity implements Serializable {
        final long value;
        LegacyQuantity(long value) {
            System.out.println("  >> LEGACY constructor RAN with value=" + value);
            if (value <= 0) throw new IllegalArgumentException("must be positive: " + value);
            this.value = value;
        }
    }

    static byte[] write(Object o) throws Exception {
        var bytes = new ByteArrayOutputStream();
        try (var out = new ObjectOutputStream(bytes)) { out.writeObject(o); }
        return bytes.toByteArray();
    }

    static Object read(byte[] b) throws Exception {
        try (var in = new ObjectInputStream(new ByteArrayInputStream(b))) {
            return in.readObject();
        }
    }

    public static void main(String[] args) throws Exception {
        System.out.println("--- writing record ---");
        byte[] rec = write(new Quantity(5));
        System.out.println("--- reading record ---");
        System.out.println("read back: " + read(rec));

        System.out.println("--- writing legacy class ---");
        byte[] leg = write(new LegacyQuantity(5));
        System.out.println("--- reading legacy class ---");
        System.out.println("read back: " + ((LegacyQuantity) read(leg)).value);
    }
}
```

```bash
java SerialRecord.java
```

**What to look for:** which constructors print during the *reading* phases.

| What you see | What it means |
|---|---|
| `>> compact constructor RAN` appears under `--- reading record ---` | **The key result.** Record deserialization goes through the canonical constructor, so your validation and your defensive copies run on data arriving from a stream. |
| `>> LEGACY constructor RAN` does **not** appear under `--- reading legacy class ---` | Also key, and the contrast is the point. Classic Java serialization allocates the object without running any constructor, then writes fields directly. Every invariant you enforce in a constructor is bypassed. This is Topic 19's core hazard, seen directly. |
| Both constructors print during reading | Unexpected — report it with your `java --version`. |

**Bonus:** hand-edit the serialized bytes so `value` is `-5` (or, more simply, write a
`Quantity` with a value your constructor allows, then change the validation rule and
re-run the read against the saved bytes). The record read will **throw**. The legacy
read will happily hand you an invalid object. That difference is the whole argument.

### Proof 5 — does your build preserve parameter names?

```bash
javac -parameters OrderLine.java && javap -v OrderLine.class | grep -i methodparameters
javac OrderLine.java             && javap -v OrderLine.class | grep -i methodparameters
```

**What to look for:**

| What you see | What it means |
|---|---|
| A `MethodParameters` section present with `-parameters`, absent without it | Confirmed how the flag behaves. Frameworks that bind by parameter name need this for ordinary classes. |
| For a record: `javap -p` still shows a `Record` attribute listing component names either way | Records carry their component names in the class file regardless of `-parameters`. That is why Jackson 2.12+ can construct records without the flag. |

Then check your real project:
```bash
./mvnw help:effective-pom | grep -A3 parameters
# or
./gradlew compileJava --info 2>&1 | grep -- -parameters
```

**How to read it:** if `-parameters` is absent and you are binding non-record DTOs by
name, that is a latent failure waiting for the first DTO someone adds without
`@JsonProperty`.

---

## Practice exercises

### 1 — Easy: convert and validate

Convert this class to a record with identical behaviour, then add the validation.

```java
public final class InventoryReservation {
    private final String sku;
    private final long quantity;
    private final Instant expiresAt;

    public InventoryReservation(String sku, long quantity, Instant expiresAt) {
        this.sku = sku;
        this.quantity = quantity;
        this.expiresAt = expiresAt;
    }

    public String getSku() { return sku; }
    public long getQuantity() { return quantity; }
    public Instant getExpiresAt() { return expiresAt; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof InventoryReservation)) return false;
        InventoryReservation that = (InventoryReservation) o;
        return quantity == that.quantity && sku.equals(that.sku);
    }

    @Override public int hashCode() { return Objects.hash(sku, quantity, expiresAt); }
}
```

Requirements:
- Add a compact constructor rejecting a null or blank `sku`, a non-positive
  `quantity`, and an `expiresAt` in the past.
- Add a static factory `forMinutes(String sku, long qty, long minutes)`.
- Add a derived method `boolean isExpired(Clock clock)`.

Then answer: **the original class has a bug that the record version silently fixes.**
Name it, and say what its production symptom would be. (Look at `equals` and
`hashCode` together. This is Topic 13.)

### 2 — Medium: the audit (combines Topics 01–26)

This class has **seven** distinct defects drawn from Topics 01, 13, 17, 19, 24, 26 and
this one. Find them all, state the observable production symptom for each, and rewrite
it.

```java
@Entity
public record CheckoutSession(
        @Id Long id,
        Long userId,
        List<OrderLine> lines,
        Optional<String> couponCode,
        Double totalAmount,
        byte[] clientFingerprint,
        String paymentToken) implements Serializable {

    public CheckoutSession {
        if (lines == null) lines = new ArrayList<>();
    }

    public boolean isSameUser(CheckoutSession other) {
        return this.userId == other.userId;
    }

    public double totalWithTax() {
        return totalAmount * 1.20;
    }

    public List<OrderLine> expensiveLines() {
        return lines.stream()
                .filter(l -> l.unitPriceMinorUnits() > 10000)
                .collect(Collectors.toList());
    }
}
```

Hints without spoilers: two defects are from Topic 01, one from Topic 26, one from
Topic 24, one is a security/logging issue from this topic, one is structural and makes
the application fail to start, and one silently breaks deduplication.

For each defect write one sentence starting with "The on-call engineer sees...".

### 3 — Hard: production simulation on `orderflow`

Build the read model for `GET /orders/{id}` end to end, using records correctly, and
prove the immutability.

**Part A — the model.** Define:

```java
record Money(long minorUnits, String currency) { }
record OrderLineView(String sku, String productName, long quantity, Money lineTotal) { }
record PaymentView(String provider, String status, Money amount, Instant at) { }
record OrderView(UUID orderId, UUID userId, Instant placedAt, String status,
                 List<OrderLineView> lines, List<PaymentView> payments, Money orderTotal) { }
```

Requirements:
- Every collection component must be defensively copied to an unmodifiable form.
- `Money` must reject a null currency and must throw on arithmetic overflow.
- `OrderView` must validate that `orderTotal` equals the sum of its line totals, and
  the exception message must name both numbers.
- No component may be `Optional` (Topic 26). Where a payment may be absent, decide what
  to do instead and justify it in a comment.

**Part B — prove it.** Write a test class (plain `main` is fine) that attempts, for each
collection component, to mutate the record's state from outside — once through the
list handed to the constructor, once through the accessor. Both must fail or have no
effect. Paste the output.

**Part C — the equality question.** Put 10,000 `OrderView` instances into a `HashSet`,
where 5,000 are duplicates by value. Assert the set size. Now change `OrderLineView` to
hold a `byte[] auditTrail` component instead of `String productName`, re-run, and
report what happens to the set size and why. Then fix it two ways and say which you
would ship.

**Part D — the projection.** Write the Spring Data JPA constructor-expression query
that populates `OrderView` directly, without loading `OrderEntity`. State exactly what
that buys you compared to loading entities and mapping — name at least three things,
one of which must reference Hibernate's persistence context (Topic 48) and one of which
must reference N+1 (Topic 50). You do not need a running database; the query text and
the reasoning are the deliverable.

---

## Interview questions

### Q1 — "Are records immutable?"

**Mid-level answer:** "Yes — all the fields are final, so you can't change a record
after you create it."

**Senior answer:** "They are **shallowly** immutable, which is a different claim.
Every component field is `final`, so the reference cannot be reassigned — but if a
component is a mutable type, the record's observable state changes freely.
`record Order(String id, List<Line> lines)` lets any holder of the original list call
`add` and change what the record reports. The fix is a defensive copy in the canonical
constructor — `lines = List.copyOf(lines)` — which both copies and makes it
unmodifiable, so the accessor needs no second copy. Arrays are the case with no clean
fix, because `byte[]` has no immutable form and the generated `equals` compares arrays
by identity — so a record with an array component is broken in two ways at once. This
is the same shallowness as `final` in Topic 17 and the same shallowness as
`readonly` in TypeScript; it is not special to records."

**What separates them:** the word "shallowly", a concrete reproduction, the
`List.copyOf` detail that one copy suffices, and generalising it back to `final`
rather than treating it as a record quirk.

**Interviewer's follow-up:** "What happens to a record with a mutable component used as
a `HashMap` key?" They want: the `hashCode` includes the mutable component, mutating it
moves the logical bucket, and the entry becomes unreachable but retained — a
correctness bug and a memory leak simultaneously. That is Topic 13, and connecting the
two is the answer they are hoping for.

---

### Q2 — "Can a record be a JPA entity?"

**Mid-level answer:** "I don't think so — records are immutable and JPA needs setters."

**Senior answer:** "No, and there are three independent reasons, any one of which is
fatal. JPA requires a no-arg constructor, which a record cannot have because every
record constructor must delegate to the canonical one. Hibernate populates and
refreshes entities by writing fields reflectively, and record fields are final. And
the persistence context's dirty checking works by diffing a managed entity against a
snapshot and emitting UPDATEs — an entity that cannot change has nothing to diff, so
the entire model does not apply. There is a fourth, more conceptual reason: JPA entity
equality is identity-based on the primary key, while a record's generated `equals` is
value-based over all components, so two unpersisted entities with null ids would
compare equal and break any `Set` inside a session. Where records **do** belong in the
persistence layer is as projections — a JPQL constructor expression selecting straight
into a record DTO, which avoids the persistence context entirely, avoids lazy proxies
leaking into serialization, and lets me select only the columns I need."

**What separates them:** three reasons instead of one, the identity-versus-value
equality point which almost nobody raises, and pivoting to where records *are* the
right tool rather than stopping at "no".

**Interviewer's follow-up:** "So where do you draw the line between your entity and
your record?" They are checking whether you have a layering opinion. A good answer:
entities stay inside the transactional boundary and never leave the service layer;
records cross every boundary — HTTP, messaging, caching.

---

### Q3 — "How do a record's `equals` and `hashCode` actually get generated?"

**Mid-level answer:** "The compiler writes them for you based on all the components."

**Senior answer:** "`javac` does not emit the method bodies. It emits an
`invokedynamic` call site in each of `equals`, `hashCode` and `toString`, bootstrapped
by `java.lang.runtime.ObjectMethods.bootstrap`, which receives the component names and
a method handle per accessor and builds the implementation on first call. You can see
it in `javap -c` — three `invokedynamic` instructions and a `BootstrapMethods` section.
The semantics: reference components use `Objects.equals`, primitives use `==`
semantics, except `float` and `double` which use `Float.compare`/`Double.compare` — so
`NaN` equals `NaN` inside a record and `+0.0` does not equal `-0.0`, both opposite to
bare `==`. The `hashCode` algorithm is deliberately **unspecified**, so it must never
be persisted or sent over a wire. And the practical value is that the component list is
the single source of truth: adding a component updates all three methods, which
eliminates the 'someone added a field and forgot `equals`' bug class entirely."

**What separates them:** knowing it is `invokedynamic` rather than generated bodies,
the float/double exception, and knowing `hashCode` is unspecified — which is the one
with a real operational consequence.

**Interviewer's follow-up:** "Why put it behind `invokedynamic` instead of just
emitting the code?" Because the strategy can be improved in a later JDK without
recompiling anyone's code, and because it keeps the class files smaller. Same reasoning
as lambdas (Topic 21) and string concatenation (Topic 18).

---

### Q4 — "Compact constructor versus canonical constructor — when do you use which?"

**Mid-level answer:** "The compact one is shorter. You use it for validation."

**Senior answer:** "The compact form has no parameter list; its parameters are
implicitly the components, and the compiler appends `this.x = x` for every component at
the end of the body. So inside it you are manipulating **parameters**, not fields —
which is why `sku = sku.trim()` is idiomatic normalisation and `this.sku = sku` is a
compile error. Use compact for validation and normalisation, which is the large
majority. Use the explicit canonical form when you need a field to hold something
structurally different from what the parameter holds — mainly defensive copying,
although you can do that in the compact form too by reassigning the parameter. Any
additional constructor must delegate to the canonical one with `this(...)`, and that
delegation rule is what makes the canonical constructor a genuine chokepoint: every
construction path goes through it, **including deserialization**, which is why records
finally make deserialization respect invariants. Classic serialization allocates the
object and writes fields without calling any constructor at all."

**What separates them:** "parameters, not fields" stated precisely, and connecting the
delegation rule to the serialization guarantee — which is the non-obvious payoff.

**Interviewer's follow-up:** "Prove the deserialization claim." A good answer is the
experiment: put a `println` in the compact constructor, serialize, deserialize, see it
fire — and contrast with an ordinary class where it does not.

---

### Q5 — "You have `record User(String email, String passwordHash)`. Any concerns?"

**Mid-level answer:** "It looks fine — it's immutable and gives you equals for free."

**Senior answer:** "Two. First, `toString` is generated over every component, so
`log.info(\"user {}\", user)` puts the password hash into your log aggregator, your
APM traces, and every exception message that interpolates the object. That is a data
incident discovered during an audit, and you cannot un-ship logs. I would override
`toString` to mask it, or better, restructure so the sensitive value is never a
component of a widely-passed record. Second, `equals` compares the hash with
`String.equals`, which is not constant-time — if this record is ever used in an
authentication comparison path that is a timing side channel, and I would use
`MessageDigest.isEqual` instead. Neither is a record-specific bug exactly, but records
make the first one *easier to commit*, because the `toString` you never wrote is the
one that leaks."

**What separates them:** spotting the generated-`toString` leak at all, and the
willingness to say "records make this easier to get wrong" rather than defending the
feature. The timing point is a bonus that signals security awareness.

**Interviewer's follow-up:** "How would you catch this across a large codebase?" Good
answers: a custom ArchUnit or ErrorProne rule that flags records with components whose
names match a sensitive pattern and that do not override `toString`; plus log scrubbing
as defence in depth, never as the primary control.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A record cannot declare extra instance fields, but it *can* declare static fields
   and methods. Why is that restriction placed exactly there? What would break if
   records could hold one extra non-component field?

2. Records are implicitly `final` and cannot extend a class. Given that, is a record a
   good fit for a type hierarchy? Answer before reading Topic 28, then check yourself
   after.

3. The generated `hashCode` is unspecified while `String.hashCode` is specified by the
   JLS. Name a concrete thing you can do with a `String` hash that you must never do
   with a record hash, and explain what would break.

4. `List.copyOf` on an already-immutable list returns the same instance instead of
   copying. Is that safe? Construct the argument that it is, from first principles about
   what "immutable" has to mean for the optimisation to be sound.

5. TypeScript's `{...a, quantity: 3}` has no Java equivalent for records. Design the
   API you would want. Then say what makes it hard for the language to provide —
   specifically, what would it have to do about the compact constructor's validation?

6. You are reviewing a PR that converts a 40-field mutable DTO into a record. The
   canonical constructor now takes 40 positional arguments. Is this an improvement?
   Argue both sides, then say what you would actually do.

7. Records serialize through the canonical constructor, which means your validation runs
   on untrusted input. Is that a security *improvement* over classic serialization, or
   does it just move the problem? Be specific about what it does and does not protect
   against. (Topic 19.)

---

## Quick reference card

### Declaration forms

```java
record Point(int x, int y) { }                              // minimal

record Point(int x, int y) {                                // compact constructor
    Point {                                                 // no parameter list
        if (x < 0) throw new IllegalArgumentException();
        x = Math.min(x, 1000);                              // reassign the PARAMETER
    }
}

record Box(List<String> items) {                            // explicit canonical
    Box(List<String> items) {
        this.items = List.copyOf(items);                    // assign the FIELD
    }
}

record Point(int x, int y) {
    Point(int x) { this(x, 0); }                            // must delegate
    static Point origin() { return new Point(0, 0); }       // static factory
    double distance() { return Math.hypot(x, y); }          // derived method
}

record Pair<A, B>(A first, B second) { }                    // generic
record Bucket(String k, long n) { }                         // legal inside a method (local record)
```

### What you get, what you cannot have

| Generated for you | Forbidden |
|---|---|
| private final field per component | extra instance fields |
| canonical constructor | extending a class |
| accessor per component (no `get`) | being subclassed (implicitly final) |
| `equals` (component-wise) | non-final component fields |
| `hashCode` (unspecified algorithm) | a no-arg constructor |
| `toString` (all components) | native methods |
| implicit `extends java.lang.Record` | — (interfaces ARE allowed) |

### `equals` semantics per component type

| Component type | Comparison used |
|---|---|
| reference (`String`, `Instant`, another record) | `Objects.equals` — calls its `equals` |
| `int`, `long`, `boolean`, `char`, `byte`, `short` | `==` semantics |
| `float`, `double` | `Float.compare` / `Double.compare` — **NaN equals NaN**, `+0.0 != -0.0` |
| **array** (`byte[]`, `String[]`) | `Objects.equals` → **reference identity**. Almost always a bug. |

### Defensive-copy cheat sheet

```java
list  = List.copyOf(list);                              // copies AND makes unmodifiable
set   = Set.copyOf(set);
map   = Map.copyOf(map);
bytes = bytes.clone();                                  // and clone() in the accessor too
date  = Instant.from(date);                             // already immutable, no copy needed
// String, Instant, UUID, BigDecimal, LocalDate, enums: immutable already. Nothing to do.
```

### Gotchas checklist

- [ ] Every mutable component gets a defensive copy in the canonical constructor.
- [ ] `List.copyOf` rejects null elements. Use `unmodifiableList(new ArrayList<>(x))` if
      you need nulls (and then ask why you need nulls).
- [ ] Array components break `equals`, `hashCode` and `toString`. Avoid or override all
      three plus the accessor.
- [ ] Generated `toString` prints **every** component. Override it for secrets and PII.
- [ ] Records cannot be JPA `@Entity`. They are excellent as JPA projections.
- [ ] Never `Optional` as a component (Topic 26). Nullable component plus a
      `maybeX()` accessor.
- [ ] Compact constructor manipulates **parameters**; `this.x = x` there is an error.
- [ ] Every extra constructor must `this(...)` to the canonical one.
- [ ] Do not persist or transmit a record's `hashCode` — the algorithm is unspecified.
- [ ] Test JSON round-tripping for every DTO record. With generated `equals` it is one
      assertion.

---

## When would I use this at work?

**1. Every DTO, command, event and query result you write.**
This is not an occasional tool. In a modern Java service, essentially every object that
crosses a boundary — HTTP request bodies, HTTP responses, Kafka event payloads, cache
values, query projections, method parameter objects — is a record. The moment you
declare one you have correct `equals`/`hashCode`, so it works properly in sets, maps
and test assertions with no further thought. `assertThat(actual).isEqualTo(expected)`
on a whole response object, instead of twelve field-by-field assertions, is the daily
payoff.

**2. Killing an N+1 by returning a projection instead of an entity.**
You find `GET /orders` issuing 300 queries. The fix is a JPQL constructor expression
selecting straight into an `OrderSummary` record. You get the columns you need, no
persistence-context attachment, no lazy proxies leaking into your JSON serializer, and
no `LazyInitializationException`. That is Topics 47 and 50, and the record is the thing
that makes the fix a two-line change rather than a refactor.

**3. Reviewing a PR where somebody added a field.**
Under the old style you would check: did they update `equals`? `hashCode`? `toString`?
The builder? The copy constructor? With a record, adding a component updates all of it
atomically, and the compiler forces every construction site to be updated too. The
review becomes "is this field the right thing to add", which is the question that
actually deserves human attention.

---

## Connected topics

**Prerequisites:**
- **13 — The equals/hashCode contract**: records are the mechanised, un-forgettable
  version of everything that topic told you to do by hand. The mutable-key corruption
  it taught you reappears here as Trap 1.
- **17 — Immutability, `final`, safe publication**: the shallowness of `final` *is* the
  shallowness of records. Records also give you safe publication for free, because all
  fields are final — a correctly-constructed record is visible to other threads without
  synchronisation (Topic 88).
- **19 — Serialization**: the constructor-bypass hazard, and how records close it.
- **14 — Comparable/Comparator**: if your record implements `Comparable`, keep
  `compareTo` consistent with the generated `equals` or `TreeSet` and `HashSet` will
  disagree about your data.
- **24 — Collectors**: local records are the right container for the intermediate tuple
  in a grouping pipeline.
- **26 — Optional**: why a component is never `Optional`, and what to do instead.

**This unlocks:**
- **28 — Sealed types**: records are the *variants*. `sealed interface PaymentResult
  permits Approved, Declined, Failed` with each variant a record is Java's algebraic
  data type, and it does not work without this topic.
- **29 — Pattern matching**: record **deconstruction patterns** —
  `case Approved(String reference, Money amount)` — bind a record's components directly.
  That is only possible because a record's component list is part of its public API.
- **46 — Error handling and `ProblemDetail`**: the error contract you return is a
  record; the domain exceptions carry records.
- **47 — Spring Data JPA projections**: constructor-expression queries into record DTOs
  — the highest-value everyday use of this topic.
- **117 — Sagas and compensations**: saga steps and their outcomes are modelled as
  records inside a sealed hierarchy; the event log is a stream of immutable records.

---

*Java baseline 21. Records were previewed in 14 and 15 and became final in **Java 16**,
so on a 21 baseline nothing here is preview and no `--enable-preview` flag is needed.
Record **patterns** (deconstruction) became final in **Java 21** — that is Topic 29.
Nothing in this topic changed between 21 and 25. The notable open proposal is
"withers" (derived record creation, the equivalent of your spread update); it is not in
21 or 25, so write the constructor call out or use a builder.*
