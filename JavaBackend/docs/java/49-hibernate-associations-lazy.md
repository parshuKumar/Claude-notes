# 49 — Hibernate II: Associations, Lazy vs Eager, `LazyInitializationException`, DTO Boundaries

## Phase: 5 — Spring Boot & Persistence
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: the association mapping across `Order` → `OrderLine` → `Product` → `Inventory`, `Order` → `Payment`, and the DTO boundary that keeps every one of them out of the HTTP layer

---

## Mechanical statement

> **A lazy `@ManyToOne` is a runtime-generated subclass proxy holding only the
> identifier.**
>
> **Touching any field other than the identifier triggers a `SELECT` — and if the
> session is closed, a `LazyInitializationException` instead.**

A lazy `@OneToMany` is the same idea in a different wrapper: not a subclass proxy but
a **collection wrapper** that runs its `SELECT` the first time you iterate, call
`size()`, or otherwise ask it anything about its contents.

Everything below follows from those two sentences.

---

## The bridge from what you know

### Lazy loading has no Prisma analogue at all

In Prisma, you say what you want:

```ts
const order = await prisma.order.findUnique({
  where: { id },
  include: { lines: { include: { product: true } } },   // explicit. always.
});
order.lines[0].product.name;   // already loaded. It is just data.
```

If you forget the `include`, `order.lines` is `undefined`. TypeScript tells you at
compile time. There is no third possibility.

Java has a third possibility:

```java
Order order = orderRepository.findById(id).orElseThrow();
order.getLines().get(0).getProduct().getName();
```

That line can do three completely different things depending on state you cannot see
in the source:

1. Return the name from memory, having issued no SQL — if everything was already
   loaded.
2. Issue **two** `SELECT`s silently, one for the lines and one for the product — if
   the session is open.
3. Throw `LazyInitializationException` — if the session is closed.

**Same line of code. Three behaviours. Nothing in the type system distinguishes
them.** That is the gap, stated exactly.

### The verdict table

| You know | Java | Verdict |
|---|---|---|
| `include: { lines: true }` — explicit, compile-checked | Fetch type declared on the mapping, resolved at runtime | **NO ANALOGUE** |
| Missing `include` → `undefined`, caught by TypeScript | Missing fetch → a proxy that silently queries or throws | **NO ANALOGUE** |
| Prisma returns plain nested objects | Hibernate returns entities with proxies inside them | **NO ANALOGUE** |
| TypeORM `relations: ['lines']` | `@EntityGraph` / fetch join | **PARTIAL** — TypeORM's is per-call and explicit; Java also has a *default* baked into the mapping |
| TypeORM lazy relations (`Promise<T[]>` properties) | Hibernate lazy collections | **PARTIAL** — TypeORM makes laziness visible in the type (`Promise`); Hibernate makes it invisible |
| Serializing a Prisma result is safe | Serializing an entity can query, explode, or recurse forever | **NO ANALOGUE** |

TypeORM's lazy relations are the closest thing you have, and the difference is
instructive: TypeORM types a lazy relation as `Promise<OrderLine[]>`, so **the type
system tells you** a database call may happen. Hibernate types it as
`List<OrderLine>`. The information is erased from the signature entirely. That
erasure is the root of every trap in this document.

### The one thing to unlearn immediately

You will be tempted to think of `LazyInitializationException` as a Hibernate bug or a
configuration mistake. It is neither.

> `LazyInitializationException` is the boundary telling you that your transaction
> ended before your serialization began.

It is a correct error message about a real design problem. The fix is to move the
boundary, not to silence the message.

---

## What is this?

### The four association annotations, and their defaults

```java
@ManyToOne   // many OrderLines -> one Product        DEFAULT: EAGER   <- the trap
@OneToOne    // one Order -> one Payment              DEFAULT: EAGER   <- also a trap
@OneToMany   // one Order -> many OrderLines          DEFAULT: LAZY
@ManyToMany  // rare, and usually a modelling mistake DEFAULT: LAZY
```

**Read that table again.** The two *to-one* associations default to `EAGER`. That is
a JPA specification decision from 2006, and it is wrong for essentially every
application written since. It is also invisible: you do not write `EAGER`, so
nothing in your code shows you it is happening.

**The rule, without exception:** every association in `orderflow` is written
`fetch = FetchType.LAZY`, explicitly, including the ones where LAZY is already the
default. Explicit is the point — a reader should never have to remember a default.

### Owning side and `mappedBy`

A bidirectional association has one foreign key column in the database and two Java
fields. Somebody has to own it.

```java
@Entity
public class OrderLine {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)   // <- THIS side owns the FK column
    private Order order;
}

@Entity
public class Order {
    @OneToMany(mappedBy = "order",                     // <- "the other side owns it"
               cascade = CascadeType.ALL,
               orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();
}
```

`mappedBy = "order"` means: *the `order` field on `OrderLine` owns this relationship;
I am the mirror.* Hibernate writes the foreign key from the owning side only.

**The consequence people trip over:** adding to `order.getLines()` without setting
`line.setOrder(order)` writes nothing, because the inverse side is not consulted when
generating SQL. That is why `Order` has an `addLine` helper that sets both sides.
Always write those helpers in pairs.

### Cascade and orphan removal

| Setting | Meaning |
|---|---|
| `cascade = CascadeType.PERSIST` | `persist` on the parent persists the children |
| `cascade = CascadeType.MERGE` | `merge` propagates |
| `cascade = CascadeType.REMOVE` | `remove` on the parent removes the children |
| `cascade = CascadeType.ALL` | all of the above, plus `DETACH` and `REFRESH` |
| `orphanRemoval = true` | removing a child from the collection **deletes its row** |

`CascadeType.ALL` + `orphanRemoval = true` is correct for a genuine
parent/child composition — an `Order` owns its `OrderLine`s; a line has no life
without its order. It is **wrong** for a reference — `OrderLine` must never cascade
to `Product`, or deleting an order deletes products.

**The test:** does the child exist independently? If yes, no cascade. `Order` →
`OrderLine`: cascade. `OrderLine` → `Product`: never.

### What a lazy association actually is

Two different mechanisms, and confusing them is a common source of wrong diagnoses.

**Lazy `@ManyToOne` / `@OneToOne` → a proxy object.**

Hibernate generates, at runtime, a **subclass** of your entity. That subclass
overrides every getter to first ask a `LazyInitializer` "have I been loaded yet?" It
holds the identifier and a reference to the session, and nothing else.

```java
OrderLine line = ...;                  // loaded
Product p = line.getProduct();         // returns a PROXY. No SELECT yet.
p.getId();                             // no SELECT - the proxy already knows the id
p.getSku();                            // *** SELECT product ... where id = ? *** now
```

**Lazy `@OneToMany` / `@ManyToMany` → a collection wrapper.**

The field is not a proxy subclass of anything. Hibernate replaces your `ArrayList`
with its own `PersistentBag` (or `PersistentSet`, or `PersistentList`), which
implements `List` and defers its query.

```java
Order order = ...;                     // loaded
List<OrderLine> lines = order.getLines();   // a PersistentBag. No SELECT yet.
lines.size();                          // *** SELECT ... from order_line where order_id = ? ***
```

Both are "lazy". One is a subclass, one is a wrapper. The exception you get is the
same; the debugging differs.

### `LazyInitializationException`

```
org.hibernate.LazyInitializationException: could not initialize proxy
  [com.orderflow.catalog.Product#4471] - no Session
```

*(That is the shape of the message. Your entity name and ID will differ.)*

It means: the proxy tried to run its `SELECT`, and the session it was created with is
gone. The session is gone because the transaction ended (Topic 48: the persistence
context is transaction-scoped and everything in it is detached at commit).

The stack trace will point **into Jackson**, or into whatever touched the proxy — not
into the code that created the boundary problem. That is why this error is confusing
the first time.

### `[BOOT 3.x DELTA]`

- Association semantics, fetch-type defaults, proxying and
  `LazyInitializationException` are **JPA specification behaviour** and are identical
  on Boot 2.x, 3.x and 4.x.
- `spring.jpa.open-in-view` has defaulted to `true` since Boot 1.x and **still does
  in Boot 4.1**. Boot logs a warning about it at startup on every version.
- The proxy library changed historically: Javassist was replaced by **ByteBuddy** as
  the default bytecode provider in Hibernate 5.3. Anything current uses ByteBuddy.
  The generated class name convention has varied across versions — you will see
  `$HibernateProxy$` in current Hibernate. **Do not memorise the exact marker;
  print it** (Proof 2 below) and read what your version produces.
- Bind-parameter logging category: `org.hibernate.orm.jdbc.bind` on Hibernate 6+,
  `org.hibernate.type.descriptor.sql.BasicBinder` on Hibernate 5. Confirm with
  `./mvnw -q dependency:list | grep -i hibernate-core`.

---

## Why does it matter?

**1. It is where the persistence layer leaks into every other layer.**

An entity with lazy associations is safe inside a transaction and dangerous outside
one. Since serialization always happens outside, "return the entity" is a defect that
compiles, passes a naive test, and fails in production or — worse — succeeds
expensively.

**2. `FetchType.EAGER` on `@ManyToOne` is a latent N+1 in every query you will ever
write against that entity.**

`OrderLine.product` being eager does not mean "one join". It means: every query that
returns `OrderLine`s also loads their `Product`s — including the query that only
wanted to count them, and including the report that loads 50,000 lines. And for a
query that returns entities in a `List`, Hibernate frequently issues one extra
`SELECT` **per row** rather than a join. That is Topic 50, and eager fetching is its
most common cause.

**3. The "obvious fix" is worse than the bug.**

`open-session-in-view` makes `LazyInitializationException` disappear. It does so by
keeping the persistence context — and, in the documented default configuration, the
database connection — open for the entire HTTP request, including view rendering and
JSON serialization. You trade a loud error for a silent capacity ceiling. That
trade is the failure drill below.

**4. The correct answer is a boundary, and boundaries are what senior engineers are
paid to place.**

Map the entity to a DTO **inside** the transaction. After that line, no proxy exists
and nothing downstream can query the database by accident. It is five lines of
mapping code and it eliminates an entire class of production incident.

---

## Machine-level reality

### 1. How the proxy is generated

At startup, for each entity that can be proxied, Hibernate uses **ByteBuddy** to
generate a subclass:

```
class Product$HibernateProxy$aB3xK9 extends Product implements HibernateProxy {
    private final LazyInitializer $$_hibernate_interceptor;   // holds entityName, id, session

    @Override public String getSku() {
        // if not yet initialized: run SELECT, populate the wrapped target
        return $$_hibernate_interceptor.getImplementation().getSku();
    }
    // ... every non-final getter and setter overridden the same way
}
```

Three consequences follow directly from "it is a subclass":

**(a) `final` blocks proxying.** A `final` entity class cannot be subclassed, so it
cannot be proxied — Hibernate falls back to eager loading and logs a warning at
startup that nobody reads. A `final` getter cannot be overridden, so touching it
returns whatever the uninitialized field holds — usually `null` — with no exception
and no query. **This is the quietest bug in this document.** Never mark an entity
class or its getters `final`.

**(b) A record can never be an entity.** Records are implicitly `final`. This is one
of the three reasons from Topic 48, and it is this one. Records remain excellent
projections and DTOs.

**(c) The no-arg constructor must exist** (at least `protected`), because the
generated subclass has to call `super()`.

### 2. Why `getClass()` and `instanceof` behave unexpectedly

```java
Product proxy = orderLine.getProduct();          // lazy proxy

proxy.getClass();                                 // Product$HibernateProxy$aB3xK9 - NOT Product.class
proxy.getClass() == Product.class;                // FALSE
proxy instanceof Product;                         // true - it IS a subclass
proxy.getClass().getName();                       // contains "$HibernateProxy$"
```

`instanceof` against the **declared** type works, because the proxy is a subclass.

`instanceof` against a **subtype** does not. If `Product` had subclasses in a JPA
inheritance hierarchy:

```java
Product proxy = orderLine.getProduct();          // a proxy of PRODUCT, the declared type
proxy instanceof DigitalProduct;                  // FALSE - even if row 4471 IS a DigitalProduct
```

The proxy was generated for the *declared association type*, and a proxy of `Product`
is not a `DigitalProduct` no matter what the row says. Hibernate cannot know without
issuing the very query the proxy exists to avoid. The escape hatch is
`Hibernate.unproxy(proxy)`, which initialises it and returns the real instance — and
therefore always issues the query.

This is a genuine, hard limitation of proxy-based laziness. If your domain relies on
polymorphic narrowing of an association, do not make it lazy.

### 3. Why an `equals` that calls `getClass()` breaks on proxied entities

This is Topic 13's contract meeting the proxy, and the result is silent data loss.

```java
// The IDE-generated equals. Correct for a POJO. Broken for an entity.
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;   // <- the defect
    return Objects.equals(sku, ((Product) o).sku);
}
```

```java
Product real  = productRepository.findBySku("SKU-4471").orElseThrow();  // Product
Product proxy = orderLine.getProduct();                                  // Product$HibernateProxy$...

real.equals(proxy);   // FALSE. getClass() differs.
proxy.equals(real);   // ALSO false, and by a different route:
                      // getClass() on the proxy returns the proxy class, and the
                      // fields read inside a proxy's own equals may be the
                      // uninitialized subclass fields, not the loaded ones.
```

Two things break as a result:

- `set.contains(product)` returns `false` for a product that is in the set.
- `list.remove(product)` silently does nothing — and with `orphanRemoval = true` on a
  collection, that means the row you meant to delete stays, or a different one goes.

**The fix, restating Topic 48's rule with the reason now visible:**

```java
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Product other)) return false;   // instanceof: a proxy passes
    return sku.equals(other.sku);                      // business key, stable from construction
}
@Override public int hashCode() { return sku.hashCode(); }
```

`instanceof` accepts the proxy because the proxy is a subclass. The business key is
readable — reading `other.sku` through the pattern variable calls the overridden
getter path on a proxy and initialises it, which is a query but a correct answer.

> If you want the strictly correct version that also handles proxies on both sides,
> Hibernate offers `Hibernate.getClass(o)`, which unwraps a proxy to its real entity
> class. Using it makes `equals` depend on Hibernate, which many teams dislike. The
> business-key `instanceof` version avoids the question entirely, which is why it is
> the recommendation.

### 4. The identifier is free; everything else is not

```java
Product proxy = orderLine.getProduct();
proxy.getId();       // NO query. The proxy was constructed with the id.
proxy.getSku();      // query.
```

This is genuinely useful. If all you need is the foreign key value — to build a URL,
to group by product, to pass to another query — take the ID off the proxy and issue
nothing.

`getReferenceById(id)` exploits the same fact in reverse: it hands you a proxy
**without any `SELECT` at all**, which is exactly what you want when you only need to
set a foreign key:

```java
Product ref = productRepository.getReferenceById(productId);   // no SELECT
OrderLine line = new OrderLine(ref, quantity, unitPriceMinor); // just sets the FK
```

The catch: if row `productId` does not exist, you do not find out on that line. You
find out at the first touch, or at flush, as an `EntityNotFoundException` from a
stack frame unrelated to the lookup. Use `getReferenceById` when you already know the
row exists (you validated it), and `findById` when you do not.

### 5. Collections are wrappers, not proxies

```java
Order order = orderRepository.findById(id).orElseThrow();

order.getLines().getClass().getName();
// org.hibernate.collection.spi.PersistentBag  (package has moved across versions - print it)

Hibernate.isInitialized(order.getLines());   // false until touched
order.getLines().size();                     // *** SELECT ... where order_id = ? ***
Hibernate.isInitialized(order.getLines());   // true
```

`Hibernate.isInitialized(x)` is the diagnostic. It works for both proxies and
collections, and it **does not** initialise them — which is exactly what you need
when debugging, because printing a lazy field in a debugger initialises it and hides
the bug you were chasing.

> **Debugger warning, and it costs people hours:** IntelliJ's variables view calls
> `toString()` and expands fields. On a lazy proxy that triggers the query. So the
> code works when you step through it and fails when you run it. If you are debugging
> lazy loading, turn off "enable toString() object view" for entity classes, or use
> `Hibernate.isInitialized` in a watch expression instead of expanding the node.

### 6. Why `@OneToOne` laziness often does not work

A lazy `@ManyToOne` works because the foreign key is on *this* row: Hibernate has the
ID without a query, so it can build a proxy.

The **inverse** side of a `@OneToOne` (the one with `mappedBy`) has no such column.
To know whether to hand you a proxy or `null`, Hibernate must query. So it queries —
and your `fetch = LAZY` is silently ignored.

For `orderflow`, `Order` → `Payment` is modelled with the FK on `Payment`, and the
`Order` side either does not exist as a field at all or is fetched explicitly when
needed. Not having the field is the cleaner answer: an association you cannot make
lazy is an association you should not have.

---

## Example 1 — minimal

```java
@Service
public class LazyDemoService {

    private final OrderRepository orders;

    @Transactional(readOnly = true)
    public String worksFine(Long orderId) {
        Order order = orders.findById(orderId).orElseThrow();
        // Session is OPEN. Touching the collection issues a SELECT and returns.
        return order.getLines().get(0).getProduct().getSku();
    }

    // NOTE: no @Transactional
    public String blowsUp(Long orderId) {
        Order order = orders.findById(orderId).orElseThrow();
        // The repository call got its own tiny transaction, which has now COMMITTED.
        // `order` is DETACHED (Topic 48). Its lines are an uninitialised PersistentBag
        // whose session is gone.
        return order.getLines().get(0).getProduct().getSku();
        //     ^^^^^^^^^^^^^^^ LazyInitializationException here
    }
}
```

Two methods. Identical bodies. One annotation of difference. One works, one throws.

**Neither is correct.** `worksFine` works by accident: it issues an uncontrolled
number of queries (one for the lines, then one per distinct product touched) and it
returns an entity graph the caller can keep exploring. The correct version returns
data, not entities:

```java
    @Transactional(readOnly = true)
    public String correct(Long orderId) {
        return orders.findFirstLineSku(orderId)      // a projection query
                     .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
```

---

## Example 2 — production scenario (on the project spine)

### The constraints

`GET /api/orders/{id}` — the order-detail endpoint, and the one Topic 65's load test
hammers.

- 1,000,000 orders, 5,000,000 order lines, 100,000 products in Postgres.
- Average order has 5 lines; the 99th percentile has 40.
- ~120 rps of order reads sustained, ~360 rps at promotion peak.
- **SLO:** p99 under 250 ms.
- HikariCP pool of 20 against an 8-core Postgres. Every in-flight request that holds
  a connection is one of twenty.

At 120 rps with a 250 ms p99 budget, Little's Law says roughly 30 requests are in
flight at any moment. **You have 20 connections.** If each request holds a connection
for its entire duration, you are already over capacity before any load spike. That
arithmetic is the whole reason `open-session-in-view` is dangerous here, and it is
why this endpoint must release its connection before serialization begins.

### The response the client actually needs

```java
package com.orderflow.orders.api;

public record OrderDetailResponse(
        Long orderId,
        String status,
        Instant placedAt,
        long totalMinor,
        String currency,
        List<OrderLineResponse> lines,
        PaymentSummary payment) { }

public record OrderLineResponse(
        String sku,
        String productName,
        int quantity,
        long unitPriceMinor,
        long lineTotalMinor) { }

public record PaymentSummary(String status, long amountMinor) { }
```

Records (Topic 27). Immutable, no proxies, no lifecycle, nothing Jackson can trip
over. Note what is **not** there: `costPriceMinor`, `supplierId`, `internalNotes`,
`Inventory`, the customer's wallet balance. The response is a deliberate subset, and
that is the security requirement from Topic 46 satisfied by construction.

### The repository — one query, explicit fetching

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    /**
     * Loads the order, its lines, and each line's product in ONE statement.
     *
     * - "join fetch" is JPQL for "join, and populate the association", as opposed to
     *   a plain "join" which filters but leaves the association lazy.
     * - "distinct" is needed because a join across a to-many multiplies the root row
     *   once per line; without it you get five copies of the same Order object.
     * - No pagination here. A fetch join with setMaxResults makes Hibernate paginate
     *   IN MEMORY - it warns and returns wrong-shaped results. That trap is Topic 50.
     *   This query is by primary key, so it does not arise.
     */
    @Query("""
           select distinct o from Order o
           left join fetch o.lines l
           left join fetch l.product
           where o.id = :id
           """)
    Optional<Order> findDetailById(@Param("id") Long id);
}
```

### The service — the boundary

```java
@Service
public class OrderQueryService {

    private final OrderRepository orders;
    private final PaymentRepository payments;

    @Transactional(readOnly = true)          // <- the boundary opens here
    public OrderDetailResponse getDetail(Long orderId) {

        Order order = orders.findDetailById(orderId)
                            .orElseThrow(() -> new OrderNotFoundException(orderId));

        Payment payment = payments.findByOrderId(orderId).orElse(null);

        // *** THE BOUNDARY ***
        // Everything below is plain data. After this method returns there is no
        // proxy anywhere in the returned object, so nothing downstream can issue a
        // query by accident - not Jackson, not a template, not a logging call.
        return new OrderDetailResponse(
                order.getId(),
                order.getStatus().name(),
                order.getPlacedAt(),
                order.getTotalMinor(),
                "GBP",
                order.getLines().stream()
                     .map(line -> new OrderLineResponse(
                             line.getProduct().getSku(),        // already fetched by the join
                             line.getProduct().getName(),
                             line.getQuantity(),
                             line.getUnitPriceMinor(),
                             line.getUnitPriceMinor() * line.getQuantity()))
                     .toList(),
                payment == null ? null
                                : new PaymentSummary(payment.getStatus().name(),
                                                     payment.getAmountMinor()));
    }
}                                            // <- transaction commits, connection released
```

### The controller — nothing to get wrong

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderQueryService orderQueries;

    @GetMapping("/{id}")
    public OrderDetailResponse getOrder(@PathVariable Long id) {
        return orderQueries.getDetail(id);   // a record. Serializing it cannot query.
    }
}
```

The controller has no `@Transactional`, no entity import, and no way to cause a lazy
load. That is the design goal: **make the mistake unrepresentable rather than
forbidden by convention.**

### The configuration that makes the boundary enforceable

```properties
spring.jpa.open-in-view=false
```

Set this on day one of any new service. With it `false`, a lazy load outside a
transaction throws immediately and loudly, in development, on the developer's
machine. With it `true`, the same code silently works in development and consumes a
connection per in-flight request in production.

**Turning it off is how you find the bugs while they are cheap.**

### The regression guard

```java
@Test
void orderDetailUsesAtMostTwoStatements() {
    Statistics stats = sessionFactory.getStatistics();
    stats.clear();

    orderQueryService.getDetail(seededOrderId);

    // 1: the order + lines + products fetch join.  2: the payment lookup.
    assertThat(stats.getPrepareStatementCount()).isLessThanOrEqualTo(2);
}
```

That assertion is what stops someone from adding a field to
`OrderLineResponse` that touches `line.getProduct().getInventory()` and quietly
turning a 2-query endpoint into a 42-query one. It is the substance of Topic 50 and
it starts here.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `FetchType.EAGER` on `@ManyToOne` (the JPA default!)

**Wrong** — and note that you write this by writing *nothing*:

```java
@ManyToOne                                   // no fetch attribute = EAGER
@JoinColumn(name = "product_id")
private Product product;
```

**Exact symptom — three, and they look unrelated until you know the cause:**

1. `select count(ol) from OrderLine ol` — a query returning a single number — shows
   joins to `product` in the SQL log. You are joining a table to count rows you are
   not returning.
2. A report loading 50,000 order lines issues **50,001 statements**: one for the
   lines, then one `select ... from product where id=?` per line. This is N+1, and
   `EAGER` is its most common cause, because a query returning entities in a `List`
   frequently resolves eager to-ones with per-row selects rather than a join.
3. You add `@ManyToOne` from `Product` to `Category`, and `Category` has an eager
   `@ManyToOne` to something else, and one `SELECT` on `OrderLine` now drags in four
   tables. Nobody changed the query.

**Root cause:** `EAGER` is a statement about the *mapping*, so it applies to every
query that returns that entity — including the ones that had no interest in the
association. You cannot opt out per query. You can only opt *in* per query, which is
what `LAZY` plus a fetch join gives you.

**Fix:** `fetch = FetchType.LAZY` on every `@ManyToOne` and `@OneToOne`, written
explicitly even where it is already the default. Then add fetch joins or
`@EntityGraph` per use case:

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "product_id", nullable = false)
private Product product;
```

**How to audit an existing codebase in one command:**

```bash
grep -rn "@ManyToOne\|@OneToOne" src/main/java | grep -v "FetchType.LAZY"
```

Every line that prints is either eager or relying on a default. Both need a decision.

---

### Trap 2 — returning a JPA entity from a controller

**Wrong:**

```java
@GetMapping("/{id}")
public Order getOrder(@PathVariable Long id) {
    return orderRepository.findById(id).orElseThrow();
}
```

**Exact symptom** — which of three you get depends on `open-session-in-view` and on
your mappings, and all three are bad:

1. **With `open-in-view=false`:**
   ```
   HttpMessageNotWritableException: Could not write JSON:
     could not initialize proxy [com.orderflow.orders.Order.lines] - no Session
   ```
   A 500 whose stack trace is entirely inside Jackson and Spring's message
   converters, with none of your classes near the top.

2. **With `open-in-view=true` (the default):** no exception. Instead, a 400-byte
   response becomes 90 KB, containing every line, every product, and — if `Product`
   has an association to `Inventory` — every inventory row too. The SQL log shows
   dozens of statements per request. Latency triples. Nothing is logged as an error.

3. **With a bidirectional association and no `@JsonIgnore`:**
   ```
   StackOverflowError
   ```
   Jackson serializes `Order` → `lines` → each `OrderLine` → its `order` → `lines` →
   forever. The stack trace is 1,024 frames of alternating Jackson serializer calls.

**Root cause:** Jackson walks getters. A getter on an entity can execute SQL. Jackson
has no idea, and no way to find out.

**Fix:** map to a record inside the transaction, as Example 2 does. That is the whole
fix, and it is not negotiable.

**Fixes that look right and are not:**

| "Fix" | Why it fails |
|---|---|
| `@JsonIgnore` on `lines` | Fixes today's field. The next association someone adds reintroduces the bug. Also puts an HTTP-layer annotation on a persistence class. |
| `@JsonManagedReference` / `@JsonBackReference` | Solves only the recursion, not the accidental queries or the leaked internal columns. |
| Jackson's Hibernate module (serialising uninitialised proxies as `null`) | Now your API silently returns `null` for data that exists, depending on what a previous query happened to load. The response shape becomes nondeterministic. |
| `open-session-in-view=true` | Trap 3. |

---

### Trap 3 — "fixing" it with `open-session-in-view`

**Wrong:**

```properties
spring.jpa.open-in-view=true       # or simply not setting it - this is the DEFAULT
```

**Exact symptom:** the `LazyInitializationException` disappears. Everything works in
development and in your integration tests. Then, under production load:

- `hikaricp.connections.active` sits pinned at the pool maximum.
- `hikaricp.connections.pending` is consistently above zero.
- `hikaricp.connections.timeout` starts incrementing.
- Requests fail with
  `HikariPool-1 - Connection is not available, request timed out after 30000ms`.
- **Endpoints that touch no database at all start failing too**, because their
  threads are blocked waiting for a connection they never needed — or because the
  application threads are all occupied.
- Throughput collapses at a level well below what a single query's cost would
  predict, and adding application instances does not help, because the bottleneck is
  a fixed number of Postgres connections.

**Root cause:** `OpenEntityManagerInViewInterceptor` keeps the persistence context
open for the **entire HTTP request** — past the service layer, through the controller
return, through Jackson serialization, until the response is written. Boot itself
warns about this at startup; the warning's shape is:

```
spring.jpa.open-in-view is enabled by default. Therefore, database queries may be
performed during view rendering. Explicitly configure spring.jpa.open-in-view to
disable this warning
```

The widely-documented consequence — and the reason Boot warns — is that the database
connection is held, or re-acquired and then held, for that whole window rather than
only for the transaction. **Do not take my word for the exact connection-release
mechanics on your Hibernate version.** Measure it: the failure drill below gives you
the exact experiment, and `hikaricp.connections.active` under load is the number that
settles it on your stack.

The deeper problem is not even the connection. It is that OSIV **removes the signal**.
With it on, an N+1 introduced during serialization produces no error, no warning and
no test failure — only a slightly slower endpoint that nobody attributes to a
mapping change made three sprints ago.

**Fix:**

```properties
spring.jpa.open-in-view=false
```

...and then fix the `LazyInitializationException`s it reveals, one by one, with DTO
boundaries. There will be some. Finding them on a laptop is the cheapest place to
find them.

Topic 109 covers pool sizing, Little's Law, and the pool-versus-thread-pool deadlock
properly. This is where you first meet the ceiling.

---

### Trap 4 — `equals` using `getClass()` on an entity

**Wrong** — the default your IDE generates:

```java
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    return Objects.equals(sku, ((Product) o).sku);
}
```

**Exact symptom:** a `Set<Product>` contains a product, and `contains` returns
`false`. Removing an `OrderLine` from `order.getLines()` silently fails, so with
`orphanRemoval = true` the row is never deleted — or, in the swap case, the wrong row
is. `size()` and the contents disagree. No exception at any point.

The tell that identifies it in ten seconds:

```java
System.out.println(product.getClass().getName());
// com.orderflow.catalog.Product$HibernateProxy$aB3xK9
```

If you see `$HibernateProxy$`, you are holding a proxy, and any `getClass()`
comparison against the entity class is `false`.

**Root cause:** Hibernate hands out proxy **subclasses**, so `getClass()` returns the
generated class, not your entity class. This is Topic 13's contract broken by a
mechanism Topic 13 could not have warned you about.

**Fix:** `instanceof` plus a business key.

```java
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Product other)) return false;
    return sku.equals(other.sku);
}
@Override public int hashCode() { return sku.hashCode(); }
```

`instanceof` accepts the subclass. The business key is stable from construction, so
the hash never changes — Topic 13 and Topic 48 satisfied together.

---

### Trap 5 — `getReferenceById` on a row that does not exist

**Wrong:**

```java
@Transactional
public void addLine(Long orderId, Long productId, int qty) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    Product product = productRepository.getReferenceById(productId);   // no SELECT
    order.addLine(new OrderLine(product, qty, product.getPriceMinor()));
}
```

**Exact symptom:**
```
jakarta.persistence.EntityNotFoundException: Unable to find
  com.orderflow.catalog.Product with id 999999
```
thrown from `product.getPriceMinor()` — or, if you never touch the product, thrown at
**flush**, from inside the transaction commit, with a stack trace that does not
mention `addLine` at all. The response is a 500 rather than the 404 that the
situation actually is.

**Root cause:** `getReferenceById` deliberately does not query. It hands you a proxy
built from the ID you supplied. Whether that row exists is not checked until the
proxy is initialised.

**Fix:** use `getReferenceById` only when existence is already established.

```java
Product product = productRepository.findById(productId)
        .orElseThrow(() -> new ProductNotFoundException(productId));   // -> Topic 46's 404
```

Then keep `getReferenceById` for the case it is genuinely good at: setting a foreign
key on a row you have already validated, without paying for a `SELECT`. Under the
order-placement path at 60 rps with 5 lines each, that is 300 avoided queries per
second — a real saving, taken deliberately, with the existence check done once
up front.

---

## Hands-on proof

Everything below is a command **you** run. I have no JVM, no Spring app and no
Postgres. Nothing here is captured output.

### Setup

```properties
# application-local.properties
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE     # Hibernate 6+; confirm your version
spring.jpa.properties.hibernate.generate_statistics=true
spring.jpa.open-in-view=false
spring.jpa.hibernate.ddl-auto=validate
management.endpoints.web.exposure.include=metrics,health
```

```bash
./mvnw -q dependency:list | grep -i hibernate-core   # confirm the major version
```

### Proof 1 — `Hibernate.isInitialized` shows laziness without destroying it

```java
@Transactional(readOnly = true)
public void lazinessProbe(Long orderId) {
    Order order = orders.findById(orderId).orElseThrow();

    System.out.println("lines initialized? " + Hibernate.isInitialized(order.getLines()));
    System.out.println("lines class      : " + order.getLines().getClass().getName());

    int n = order.getLines().size();                       // this triggers the SELECT

    System.out.println("after size()     : " + Hibernate.isInitialized(order.getLines()) + " n=" + n);
}
```

**What to look for:** the two booleans, and where the `select ... from order_line`
line appears in the SQL log relative to the printed lines.

| What you see | What it means |
|---|---|
| `false`, a `PersistentBag`/`PersistentList` class name, then `true`, with the `select` between them | The expected result. The collection is a wrapper that queries on first access. |
| `true` on the first line | Something already initialised it. A fetch join in the query, an eager mapping, or your debugger expanded the node — see the debugger warning above. |
| A plain `java.util.ArrayList` class name | The collection was never managed. The entity is detached, or you are looking at a DTO. |

**Why `isInitialized` and not a debugger:** it answers the question without changing
the answer. Expanding the node in an IDE calls `toString()` and initialises it.

### Proof 2 — the `$HibernateProxy$` marker

```java
@Transactional(readOnly = true)
public void proxyProbe(Long orderLineId) {
    OrderLine line = orderLines.findById(orderLineId).orElseThrow();
    Product product = line.getProduct();

    System.out.println("class      : " + product.getClass().getName());
    System.out.println("== Product : " + (product.getClass() == Product.class));
    System.out.println("instanceof : " + (product instanceof Product));
    System.out.println("initialized: " + Hibernate.isInitialized(product));
    System.out.println("id (free)  : " + product.getId());
    System.out.println("initialized: " + Hibernate.isInitialized(product));   // still false
    System.out.println("sku (costs): " + product.getSku());                    // <- SELECT here
    System.out.println("initialized: " + Hibernate.isInitialized(product));   // now true
}
```

**What to look for:** the class name, and where the `select ... from product` line
lands in the SQL log.

| What you see | What it means |
|---|---|
| A class name containing `$HibernateProxy$`, `== Product` false, `instanceof` true | The expected result. This is the mechanical statement made visible. |
| Class name is exactly `com.orderflow.catalog.Product` | The association is **eager**, or something already initialised it. Check the mapping for a missing `fetch = FetchType.LAZY` — this is Trap 1, found. |
| The `select` appears **before** `id (free)` | Not lazy. Same diagnosis as above. |
| The `select` appears between `id (free)` and `sku (costs)` | Correct: the identifier is free, everything else is not. |
| A different marker string than `$HibernateProxy$` | Your Hibernate version uses a different naming convention. Trust your output over this document — that is why you printed it. |

### Proof 3 — reproduce the exception on purpose

Add a temporary endpoint on the `local` profile that returns the entity directly:

```java
@Profile("local")
@RestController
class LeakyOrderController {
    private final OrderRepository orders;
    @GetMapping("/local/orders/{id}")
    Order raw(@PathVariable Long id) { return orders.findById(id).orElseThrow(); }
}
```

```bash
curl -i http://localhost:8080/local/orders/1
```

**What to look for:** the status, and what the stack trace's top frames name.

| What you see | What it means |
|---|---|
| 500, and `LazyInitializationException ... - no Session` in the log, with Jackson frames on top | `open-in-view=false` is working and the boundary is being enforced. This is the *good* outcome. |
| 200 with a large body and many `select` lines in the log | `open-in-view` is `true`. You have Trap 3. Check the property and the startup warning. |
| 500 with `StackOverflowError` and alternating Jackson frames | Bidirectional recursion. Two problems at once. |

### Proof 4 — count the statements a fetch join saves

Run `getDetail(orderId)` twice: once with `findById`, once with `findDetailById`
(the fetch join), with statistics on.

```java
Statistics stats = sessionFactory.getStatistics();
stats.clear();
orderQueryService.getDetail(orderId);
System.out.println("statements = " + stats.getPrepareStatementCount());
System.out.println("entities   = " + stats.getEntityLoadCount());
```

**What to look for:** the statement count for an order with 5 lines and 5 distinct
products.

| What you see | What it means |
|---|---|
| `findById` version: around 7 statements (order, lines, 5 products) | N+1, observed. Topic 50 formalises it. |
| `findDetailById` version: around 2 | The fetch join collapsed it into one query plus the payment lookup. |
| `findDetailById` version still high | The `join fetch` is missing a level — check that `l.product` is fetched, not just `o.lines`. |
| Both identical and low | Something is eagerly fetching already. Check the mappings. |

Now seed an order with 40 lines and repeat. The fetch-join count should not move; the
naive count should grow roughly with the line count. **That difference is the whole
argument, in two numbers.**

### Proof 5 — watch the connection metric

```bash
curl -s localhost:8080/actuator/metrics/hikaricp.connections.active | jq .
curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending | jq .
curl -s localhost:8080/actuator/metrics/hikaricp.connections.timeout | jq .
```

**What to look for:** `active` while a load test is running.

| What you see | What it means |
|---|---|
| `active` well below the pool max, `pending` at 0 | Connections are released promptly. Healthy. |
| `active` pinned at the pool max, `pending` above 0 | Connections are held longer than the work needs them. Under OSIV this is the expected shape. |
| `timeout` incrementing | Requests are failing to get a connection at all. This is the outage. |

This metric is the instrument for the failure drill.

---

## Failure drill

**This drill is mandatory.** It is the one from the master plan's failure-drill map:
*return an entity with a lazy collection from a controller and get the exception;
then "fix" it by enabling `open-session-in-view` and show the connection is now held
for the whole request.*

### Setup

```properties
# application-drill.properties
spring.jpa.open-in-view=false
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.connection-timeout=5000
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.SQL=DEBUG
management.endpoints.web.exposure.include=metrics,health
```

Seed at least 1,000 orders with 5 lines each. Ten connections and a five-second
timeout make the effect visible on a laptop; production numbers would just take
longer to reach the same place.

```java
@Profile("drill")
@RestController
@RequestMapping("/drill/orders")
public class DrillOrderController {

    private final OrderRepository orders;
    private final OrderQueryService queries;

    /** PART A / B: returns the ENTITY. */
    @GetMapping("/{id}/entity")
    public Order raw(@PathVariable Long id) {
        return orders.findById(id).orElseThrow();
    }

    /** PART C: returns a DTO mapped inside the transaction. */
    @GetMapping("/{id}/dto")
    public OrderDetailResponse dto(@PathVariable Long id) {
        return queries.getDetail(id);
    }
}
```

### Part A — the exception

```bash
# spring.jpa.open-in-view=false
curl -i http://localhost:8080/drill/orders/1/entity
```

**What to capture:** the HTTP status, the exception class name, and which frames are
at the top of the stack trace.

| What you see | What it means |
|---|---|
| 500, `LazyInitializationException: could not initialize proxy [...Order.lines] - no Session`, Jackson frames on top | The expected result. The transaction ended at the service boundary; serialization happened after it. |
| 200 with a full body | `open-in-view` is not actually `false`. Check the active profile and the startup warning. |
| `StackOverflowError` | Bidirectional recursion is firing before the lazy load does. Add `@JsonIgnore` on the back-reference **only for this drill** so you observe the intended failure, then remove it. |

**What it proves:** the exception is not a bug. It is the boundary telling you the
transaction ended before serialization began. Note that the stack trace names
Jackson, not the line that made the design mistake — that is why this error is
confusing the first time, and why recognising the shape is worth a drill.

### Part B — the "fix" that moves the cost

```properties
spring.jpa.open-in-view=true
```

Restart. Re-run the same `curl`.

**What to capture:**
1. The status and the body size (`curl -s ... | wc -c`).
2. The number of `select` lines in the SQL log for that one request.
3. `hikaricp.connections.active` **while a load test is running**.

Now run load against it:

```bash
k6 run --vus 30 --duration 60s drill-entity.js       # 30 virtual users, pool of 10
```

While it runs, in another terminal, poll the metrics once a second:

```bash
while true; do
  curl -s localhost:8080/actuator/metrics/hikaricp.connections.active \
    | jq -r '"active=" + (.measurements[0].value|tostring)'
  curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending \
    | jq -r '"pending=" + (.measurements[0].value|tostring)'
  sleep 1
done
```

*(Use `Monitor`-style polling or `watch`; a foreground `sleep` loop in a script is
fine here since you are watching it.)*

| What you see | What it means |
|---|---|
| No exception, a much larger body, many `select` lines per request | OSIV made the error disappear by doing the queries during serialization. The work did not go away; it moved to a place with no transaction around it. |
| `active` pinned at 10, `pending` consistently above 0, `timeout` incrementing, request failures at ~5 s | **The point of the drill.** The pool is saturated because connections are held past the transaction, for the whole request. |
| `active` low and stable | Your Hibernate version releases the connection more aggressively than the common case, or your load is too light. Raise the VU count, add think-time-free requests, and re-check. **If it still stays low, that is a real finding on your stack — write it down.** I am not going to assert a mechanism your own measurement contradicts. |
| Latency p99 far above the single-request latency | Queueing for connections, not query cost. This is the shape of pool exhaustion. |

**What it proves:** OSIV converts a loud, local, development-time error into a
silent, global, production-time capacity ceiling. The `LazyInitializationException`
was the cheaper failure.

**Forward reference:** Topic 109 covers HikariCP sizing, Little's Law, and the
pool-versus-thread-pool deadlock. What you have just produced is the simplest version
of that failure: work holding a connection longer than it needs one. Keep these
numbers; you will re-run this experiment there.

### Part C — the real fix

```properties
spring.jpa.open-in-view=false
```

Restart and hit `/drill/orders/{id}/dto` under the same load.

**What to capture:** the same three things, plus `getPrepareStatementCount()`.

| What you see | What it means |
|---|---|
| No exception, ~2 statements per request, `active` well below 10, `pending` at 0 | The expected result. The DTO boundary released the connection before serialization. |
| Statement count above 2 | The fetch join is not covering everything the DTO reads. Find the field that triggers the extra query; that is exactly the N+1 Topic 50 teaches you to assert against. |
| An exception | Your mapping code touches an association the fetch join does not load. Same diagnosis: read the SQL log and extend the fetch join. |

### Write it up

Three numbers, three rows, one table:

| Configuration | Statements/request | Peak `hikaricp.connections.active` | p99 at 30 VUs | Failures |
|---|---|---|---|---|
| Entity + `open-in-view=false` | — | — | — | 100% (exception) |
| Entity + `open-in-view=true` | | | | |
| DTO + `open-in-view=false` | | | | |

That table is the deliverable. It is also, almost verbatim, the answer to the
interview question "why is open-session-in-view bad?" — with your own numbers behind
it instead of a blog post's.

---

## Measurement

### Eyeballing the SQL log does not scale

For one drill request it is fine. For anything real it is not:

- One order-detail request can issue 2 statements or 45. At 120 rps across 20 threads
  the log lines interleave and you cannot attribute them to a request.
- **You cannot assert on a log.** A change that adds 40 queries produces a longer
  log, and nobody reads a longer log.
- Statement text changes across Hibernate versions, so any grep you write is fragile.

### Use `Statistics`, and assert on it

```properties
spring.jpa.properties.hibernate.generate_statistics=true
```

(Note the `spring.jpa.properties.` prefix. Omitting it silently does nothing.)

```java
SessionFactory sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);
Statistics stats = sessionFactory.getStatistics();
stats.clear();                       // ALWAYS. Counters are cumulative per SessionFactory.
```

| Counter | What it tells you about associations |
|---|---|
| `getPrepareStatementCount()` | **The headline number.** Total JDBC statements. This is what an N+1 inflates. |
| `getEntityLoadCount()` | How many entity instances were materialised — a lazy collection of 40 lines adds 40. |
| `getQueryExecutionCount()` | JPQL/HQL/criteria queries. A fetch join keeps this at 1 where a lazy walk raises it. |
| `getCollectionFetchCount()` | How many lazy collections were fetched **separately**. If this is above zero on an endpoint you thought used a fetch join, the fetch join is not covering it. |
| `getCollectionLoadCount()` | Collections loaded in total. |
| `getSessionOpenCount()` | Persistence contexts created. More than one per request means your boundaries are not where you think. |

`getCollectionFetchCount()` is the counter specific to this topic. It is the direct
measurement of "how many lazy collections did I trip over".

### Asserting on a query count is what stops a regression

```java
@Test
void orderDetailStaysAtTwoStatements() {
    Statistics stats = sessionFactory.getStatistics();
    stats.clear();

    orderQueryService.getDetail(seededOrderId);

    assertThat(stats.getPrepareStatementCount()).isLessThanOrEqualTo(2);
    assertThat(stats.getCollectionFetchCount()).isZero();   // the fetch join covered it
}
```

**This test is the deliverable, not the fix.** Anyone can remove an N+1 once. The
assertion is what stops it coming back when someone adds a field to the response
record six months from now — and it fails in CI, with a number, rather than as a
latency regression nobody attributes correctly.

This is exactly where Topic 50 picks up: it makes this a reusable test rule, applies
it to the `GET /orders` list endpoint, and covers the full menu of fixes (fetch join,
`@EntityGraph`, `@BatchSize`, subselect, projection) with the fetch-join-plus-
pagination trap.

For a language-agnostic count that also catches statements Hibernate does not issue
(Flyway, native `JdbcClient` calls), wrap the `DataSource` with `datasource-proxy` or
p6spy — the setup is in Topic 47's hands-on section.

### Do NOT time this with `System.nanoTime()`

```java
long start = System.nanoTime();
orderQueryService.getDetail(orderId);
long elapsed = System.nanoTime() - start;    // this number means very little
```

Wrong for at least five compounding reasons:

1. **JIT state.** Early invocations are interpreted; you are timing compilation.
2. **Database cache state.** The first read of a page is cold in Postgres shared
   buffers; the second is warm. Orders of magnitude apart.
3. **Connection acquisition.** The first call may create a pooled connection.
4. **GC.** A young collection inside the window adds milliseconds unrelated to your
   code.
5. **The persistence context is stateful (Topic 48).** A second call in the same
   context hits the identity map and issues no SQL at all — you may be timing a
   `HashMap` lookup.

Topic 77 is the full treatment. Until then: **counters, not timings.** "45 statements
became 2" is a finding that survives review. "14 ms became 11 ms" is noise.

### In production

Put `hikaricp.connections.active`, `hikaricp.connections.pending` and
`hikaricp.connections.timeout` on a dashboard next to your RED metrics (Topic 118).
Those three, together, are the early warning for everything in this document: a lazy
load that escaped its boundary shows up as connections held longer than the work
justifies, long before it shows up as an error.

`[BOOT 3.x DELTA]` — Boot's auto-configured `hibernate.*` Micrometer metrics were
deprecated during the 3.x line in favour of Hibernate's own Micrometer integration,
and availability differs by version. Confirm rather than assume:

```bash
curl -s localhost:8080/actuator/metrics | jq -r '.names[]' | grep -iE "hibernate|hikari"
```

`hikaricp.*` metrics come from HikariCP itself and are reliably present when the pool
is auto-configured.

---

## Practice exercises

### 1 — Easy: make laziness visible

On `orderflow`:

1. Write a `@Transactional(readOnly = true)` probe that loads one `OrderLine`, gets
   its `Product`, and prints: the product's `getClass().getName()`,
   `Hibernate.isInitialized(product)` before and after calling `getId()`, and again
   after calling `getSku()`.
2. Do the same for `order.getLines()` — print the collection's class name and
   `isInitialized` before and after `size()`.
3. Now change `OrderLine.product` to `@ManyToOne` with **no** `fetch` attribute
   (making it eager) and re-run. Record every line that changed.

**Write down:** at which exact call does the `select` appear in each configuration,
and what the class name tells you before any SQL is issued at all.

### 2 — Medium: combining earlier topics (01–48)

Build `GET /api/orders/{id}` properly, end to end.

Requirements:

- `OrderDetailResponse`, `OrderLineResponse` and `PaymentSummary` as **records**
  (Topic 27). Add a comment stating why these cannot be entities (Topic 48's three
  reasons — and note that reason (a) in Machine-level reality above is *this* topic's
  proxy requirement).
- `Optional` from the repository, `orElseThrow` in the service, and a Topic 46
  `ProblemDetail` 404 with `type` `.../order-not-found`.
- `equals`/`hashCode` on `Order` and `Product` using a business key and `instanceof`
  (Topics 13, 48). Then write a test that puts a real `Product` and a **proxy** of the
  same row in a `HashSet` and asserts `size() == 1`. Then change `equals` to use
  `getClass()` and watch the test fail. Report the size.
- Money stays `long` minor units everywhere (Topic 01).
- `spring.jpa.open-in-view=false`.
- Every `@ManyToOne` and `@OneToOne` in the codebase explicitly `FetchType.LAZY`.
  Prove it with the `grep` from Trap 1 and paste the (empty) output.

Then measure with `hibernate.generate_statistics`:

1. `getPrepareStatementCount()` for the naive `findById` version.
2. The same for the `join fetch` version.
3. The same for an order with 40 lines, both ways.

Report all four numbers and explain the shape of the difference in one sentence.

**Finally:** add the regression test asserting `getPrepareStatementCount() <= 2` and
`getCollectionFetchCount() == 0`. Then deliberately add a field to
`OrderLineResponse` that reads `line.getProduct().getInventory().getQuantityAvailable()`
and confirm the test fails. Paste the failure. That failing test is the point of the
exercise.

### 3 — Hard: production simulation

Advance `orderflow`'s `GET /api/orders` list endpoint — the one Topic 65 hammers —
and produce the numbers to defend it.

**Part A — the dataset.** 100,000 products, 1,000,000 orders, 5,000,000 order lines,
with skew: 200 products carry 40% of the lines; some customers have 5,000+ orders;
order line counts follow a realistic distribution with a p99 of about 40.

**Part B — build it three ways.** `GET /api/orders?customerId=&size=20`, returning
for each order: id, status, placedAt, totalMinor, line count, and the first three
product names.

1. **Naive:** load `Order` entities, walk `getLines()` and `getProduct()` in the
   mapping code.
2. **Fetch join:** one JPQL query with `join fetch`. **You will hit the fetch-join-
   plus-pagination trap here** — Hibernate warns that it is applying the limit in
   memory. Capture that warning verbatim, explain what it means for a 1,000,000-row
   table, and note that Topic 50 gives you the two-query fix.
3. **Projection:** a JPQL constructor expression, or two queries (page of order IDs,
   then lines for those IDs), returning records and loading no entities at all.

**Part C — measure each.** For all three: `getPrepareStatementCount()`,
`getEntityLoadCount()`, `getCollectionFetchCount()`, and `EXPLAIN (ANALYZE, BUFFERS)`
on the generated SQL.

**Part D — the OSIV experiment at scale.** Run each of the three under k6 at 100 rps
for 3 minutes with a pool of 20, with `open-in-view` both `false` and `true` (six
runs). Record peak `hikaricp.connections.active`, `pending`, `timeout`, and p50/p95/
p99. Produce the six-row table.

**Part E — argue it.** Which implementation ships, and why? Then state the condition
under which your answer flips. Then answer the harder question: **is there any
service for which `open-session-in-view=true` is the right choice?** Make the
strongest case you can for "yes" before giving your own answer.

**Part F — the honest caveat.** Your p99 numbers came from a load script against a
laptop Postgres. Write three sentences on why they are not trustworthy as absolute
numbers, what they *are* good for (relative comparison under identical conditions),
and which topics make them rigorous — 65 for load methodology, 77 for
microbenchmarking, 109 for pool sizing.

---

## Interview questions

### Q1 — "What is `LazyInitializationException` and how do you fix it?"

**Mid-level answer:** "It happens when you access a lazy association outside the
session. You fix it by enabling `open-session-in-view`, or by using `EAGER` fetching,
or by calling `Hibernate.initialize()`."

**Senior answer:** "It means a proxy tried to run its `SELECT` and the session it
belonged to is gone — because the transaction ended and everything in the persistence
context was detached. It's not a bug; it's the boundary telling me my transaction
ended before my serialization began. The fix is to map to a DTO inside the
transaction, so no proxy ever leaves the service layer. The three fixes people reach
for are all worse: `EAGER` makes it a latent N+1 on every query that touches the
entity, including ones that don't need the association; `Hibernate.initialize` fixes
one call site and leaves the design; and `open-session-in-view` makes the error
disappear by holding the persistence context — and in the documented default
configuration, the connection — for the whole request, which turns a loud
development-time error into a silent production capacity ceiling. I've measured that:
with a pool of 10 under 30 concurrent users, `hikaricp.connections.active` pins at
the maximum and `pending` climbs."

**What separates them:** reframing the exception as correct feedback rather than a
fault, and rejecting all three common fixes with a specific reason for each. The
measured detail is what makes it credible rather than recited.

**Follow-up:** "Is `open-session-in-view` ever right?" A good answer concedes the
case: a small internal service, a server-rendered template that genuinely needs the
graph, low concurrency, no SLO. And then says it is a decision to be made once, with
the pool metric watched, not a default to be left on by accident.

---

### Q2 — "Why is `FetchType.EAGER` on a `@ManyToOne` a problem, given that it's the JPA default?"

**Mid-level answer:** "It loads data you might not need, so it's slower."

**Senior answer:** "Because it's a property of the *mapping*, not the query, so it
applies to every query that returns that entity — including a `count`, including a
report that loads 50,000 rows, including queries written years later by people who
never look at the mapping. You can't opt out per query; you can only opt in, which is
what `LAZY` plus a fetch join or an `@EntityGraph` gives you. And the failure mode
isn't a bigger join — for a query returning entities in a list, Hibernate frequently
resolves eager to-ones with one extra `SELECT` per row, so eager fetching is the most
common *cause* of N+1, not a cure for it. The default is a 2006 specification decision
that predates the access patterns we actually have. My rule is `LAZY` written
explicitly on every association, including the ones where it's already the default,
because a reader shouldn't have to remember a default. It's one grep to audit:
`@ManyToOne` or `@OneToOne` without `FetchType.LAZY`."

**What separates them:** mapping-versus-query as the structural reason, knowing eager
often means N+1 rather than a join, and having an audit command ready.

**Follow-up:** "How do you fetch the association when you do need it?" `join fetch`
in JPQL for a specific query, `@EntityGraph` when you want it declarative per use
case, `@BatchSize` when you need several collections, and a projection when you never
needed the entities. Topic 50 is where those get ranked.

---

### Q3 — "Walk me through what a lazy `@ManyToOne` actually is at runtime."

**Mid-level answer:** "It's a placeholder that Hibernate fills in when you access
it."

**Senior answer:** "It's a subclass of my entity, generated at runtime by ByteBuddy,
implementing `HibernateProxy` and holding a `LazyInitializer` with the entity name,
the identifier and a session reference. Every non-final getter is overridden to ask
the initializer to load the real instance first. Three consequences follow. The
identifier getter is free — the proxy already has it — so
`getReferenceById` plus setting a foreign key issues no `SELECT` at all, which is a
real saving on a write path. `getClass()` returns the generated class, so any
`equals` written with `getClass() != o.getClass()` silently returns false against a
proxy of the same row — that's a `HashSet` losing entities, which is Topic 13's
contract broken by a mechanism you can't see. And because it's a subclass, a `final`
entity class or a `final` getter can't be proxied: Hibernate falls back to eager and
logs a warning nobody reads, which is also one of the three reasons a record can
never be an entity. Collections are different — they're not subclass proxies but
`PersistentBag`-style wrappers that query on first access."

**What separates them:** naming ByteBuddy and `LazyInitializer`, and deriving three
concrete consequences from "it is a subclass" rather than listing facts.

**Follow-up:** "What breaks if the association is polymorphic?" `instanceof` against
a *subtype* fails on a proxy, because the proxy was generated for the declared type.
`Hibernate.unproxy` is the escape hatch and it always issues the query. Strong
candidates conclude that polymorphic narrowing and lazy loading do not mix.

---

### Q4 — "How do you prove you fixed an N+1?"

**Mid-level answer:** "I'd look at the SQL log and see fewer queries."

**Senior answer:** "A log tells me it's fixed today; it doesn't stop it coming back.
I assert on a number. `hibernate.generate_statistics=true`, then
`getPrepareStatementCount()` and `getCollectionFetchCount()` in a test, with
`stats.clear()` first because the counters are cumulative per `SessionFactory`. The
test asserts the order-detail endpoint stays at two statements. Then I make the test
prove itself: I add a field to the response record that touches an association the
fetch join doesn't cover, and confirm the test fails. A test I haven't seen fail
isn't a guard. I'd also run it against real Postgres via Testcontainers rather than
H2, because H2 accepts SQL Postgres rejects and its plans are meaningless. And I would
not use a `nanoTime` timing — JIT state, cold buffers and the first-level cache make
it noise, and '45 statements became 2' survives review in a way '14 ms became 11 ms'
does not."

**What separates them:** the assertion as the deliverable, `clear()` as a real
gotcha, deliberately failing the test, and rejecting timing with specific reasons.

**Follow-up:** "What if the extra queries come from something other than Hibernate?"
Wrap the `DataSource` with `datasource-proxy` or p6spy and count at the JDBC level —
that also catches Flyway, native `JdbcClient` calls and anything else.

---

### Q5 — "Should you ever return a JPA entity from a controller?"

**Mid-level answer:** "No, you should use DTOs to decouple the API from the
database."

**Senior answer:** "No, and the decoupling argument is the weakest of the reasons.
Concretely: Jackson walks getters, and a getter on an entity can execute SQL, so
serialization becomes an uncontrolled query source — either an exception when the
session is closed, or a silent explosion of statements when it isn't. A bidirectional
association gives you infinite recursion and a `StackOverflowError`. The entity
carries internal columns — cost price, supplier ID — that are now in my public API by
accident, which is a finding in any security review. And the entity's field names
become my wire contract, so a schema refactor is a breaking API change. Mapping to a
record inside the transaction fixes all four and also lets me narrow the `SELECT`
with a projection, so it's less I/O too. The band-aids — `@JsonIgnore`,
`@JsonManagedReference`, the Jackson Hibernate module — each fix one symptom and
leave the entity in the serialization layer, so the next field someone adds
reintroduces the bug."

**What separates them:** four concrete failure modes ranked above the architectural
principle, and naming why each band-aid is insufficient rather than merely
suboptimal.

**Follow-up:** "That's a lot of mapping code. How do you keep it cheap?" Honest
answers: records make it terse; JPQL constructor expressions skip the entity
entirely; MapStruct generates it at compile time if the volume justifies a
dependency. A weak answer reaches for reflective mapping, which reintroduces the
proxy-touching problem it was meant to solve.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A lazy `@ManyToOne` is a subclass proxy; a lazy `@OneToMany` is a collection
   wrapper. Why could Hibernate not use the same mechanism for both? What property of
   a collection makes the wrapper approach available and the subclass approach
   unnecessary?

2. The identifier is free on a proxy but every other field costs a `SELECT`. Design
   an API that makes that cost visible in the type system, the way TypeORM's
   `Promise<T[]>` does. What would you give up in exchange?

3. `open-session-in-view` is on by default, and Boot logs a warning about it on every
   startup. Given that the maintainers clearly consider it a hazard, why do you think
   the default has never been flipped? Argue the maintainers' side properly.

4. `instanceof` works on a proxy for the declared type but fails for a subtype. Is
   that a bug, a limitation, or a correct consequence of the design? Defend your
   answer, then construct the strongest counter-argument.

5. You cannot make an entity immutable (Topic 17) and you cannot make it a record
   (Topic 27), partly because Hibernate needs to subclass it. Suppose that constraint
   vanished tomorrow. What else about JPA would have to change before immutable
   entities became workable?

6. A DTO boundary inside the transaction solves lazy loading, over-fetching, internal
   field leakage and recursion. Name the costs of that boundary — there are at least
   three — and say which of them you would accept and which you would try to reduce.

7. Your team has a service with `open-in-view=true` and 200 endpoints returning
   entities. You cannot fix it in one sprint. Design the migration: what do you change
   first, what metric tells you it is working, and how do you stop new endpoints from
   making it worse while the migration is in flight?

---

## Quick reference card

### Defaults (memorise the first two)

| Annotation | JPA default | What you write |
|---|---|---|
| `@ManyToOne` | **EAGER** | `fetch = FetchType.LAZY` |
| `@OneToOne` | **EAGER** | `fetch = FetchType.LAZY` |
| `@OneToMany` | LAZY | `fetch = FetchType.LAZY` (explicit anyway) |
| `@ManyToMany` | LAZY | `fetch = FetchType.LAZY` (and reconsider the model) |

### Mapping

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "order_id", nullable = false)     // the OWNING side: has the FK
private Order order;

@OneToMany(mappedBy = "order",                       // the INVERSE side: mirrors it
           cascade = CascadeType.ALL,                // only for true composition
           orphanRemoval = true)                     // removing from the list DELETES
private List<OrderLine> lines = new ArrayList<>();

// always set both sides
public void addLine(OrderLine line) { lines.add(line); line.setOrder(this); }
```

### Diagnostics

```java
Hibernate.isInitialized(x)          // true/false WITHOUT initialising it
Hibernate.initialize(x)             // force it (a fix for one call site, not a design)
Hibernate.unproxy(x)                // get the real instance - always queries
Hibernate.getClass(x)               // the real entity class behind a proxy
x.getClass().getName()              // contains "$HibernateProxy$" if it is one
entityManager.contains(x)           // managed or detached? (Topic 48)
```

### Fetching explicitly

```java
@Query("select distinct o from Order o left join fetch o.lines l left join fetch l.product where o.id = :id")
// "distinct" because a to-many join multiplies the root row
// NEVER combine a to-many fetch join with setMaxResults - Hibernate paginates in memory (Topic 50)

@EntityGraph(attributePaths = { "lines", "lines.product" })
Optional<Order> findById(Long id);
```

### Properties

```properties
spring.jpa.open-in-view=false                              # day one, every service
spring.jpa.properties.hibernate.generate_statistics=true    # note the properties. prefix
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE             # Hibernate 6+; CONFIRM
spring.jpa.hibernate.ddl-auto=validate
```

### Metrics to watch

```
hikaricp.connections.active     <- pinned at max = connections held too long
hikaricp.connections.pending    <- above 0 = requests queueing for a connection
hikaricp.connections.timeout    <- incrementing = the outage
```

### Counters to assert

```java
stats.clear();                          // ALWAYS first - counters are cumulative
stats.getPrepareStatementCount()        // the headline number
stats.getCollectionFetchCount()         // lazy collections fetched separately
stats.getEntityLoadCount()              // entities materialised
```

### Checklist

- [ ] Every `@ManyToOne` and `@OneToOne` is explicitly `FetchType.LAZY`.
- [ ] `spring.jpa.open-in-view=false`.
- [ ] No controller returns an `@Entity`. Ever.
- [ ] Mapping to DTOs happens **inside** the transaction.
- [ ] Response types are records.
- [ ] `equals`/`hashCode` use `instanceof` and a business key, never `getClass()`.
- [ ] No entity class or getter is `final`.
- [ ] Cascade only for true parent/child composition; never to a reference.
- [ ] Bidirectional associations have a helper that sets both sides.
- [ ] `getReferenceById` only where existence is already established.
- [ ] Every read endpoint has a query-count assertion in a test.
- [ ] That assertion has been seen to fail at least once.

---

## When would I use this at work?

**1. Day one on any new Spring Boot service.**
Set `spring.jpa.open-in-view=false` and write `FetchType.LAZY` on every association
before there are any endpoints. Both cost nothing on day one. Retrofitting them into
a service with 200 endpoints returning entities is a quarter of work, and the
migration has no natural stopping point.

**2. Diagnosing an endpoint that got slower with no query change.**
Someone added a field to a response DTO that touches one more association. The SQL
did not change; the *number* of SQL statements did. `getCollectionFetchCount()` and
`getPrepareStatementCount()` find it in minutes. Without them you are reading a log
and guessing.

**3. In an incident where the pool is exhausted and endpoints that touch no database
are also failing.**
The instinct is to raise the pool size, and it makes things worse — Postgres runs a
process per connection, so more connections means more contention. The right first
question is "what is holding connections longer than the work needs them", and OSIV
plus lazy loading during serialization is the most common answer in a Spring
codebase. Topic 109 is the full treatment; this topic is why you look there first.

---

## Connected topics

**Prerequisites:**
- **13 — equals/hashCode contract**: broken by proxies in a way Topic 13 could not
  have warned you about. `getClass()` is the defect.
- **17 — Immutability**: entities cannot be immutable, partly because Hibernate must
  subclass them. DTOs can be, and that is where the boundary pays off.
- **26 — Optional**: repository returns, and how absence reaches Topic 46's 404.
- **27 — Records**: the correct DTO shape — and implicitly `final`, which is exactly
  why a record cannot be an entity.
- **40 — Proxying**: the same idea as Spring's CGLIB proxies — a runtime-generated
  subclass — applied to entities instead of beans. If you understood
  `$$SpringCGLIB$$`, you already understand `$HibernateProxy$`.
- **44 — REST controllers**: where the serialization happens, and therefore where the
  boundary must already have been crossed.
- **46 — `ProblemDetail`**: never touch an entity inside an exception handler; the
  transaction is already rolled back and the context is gone.
- **47 — Spring Data JPA**: projections and `@Query`, which are the tools this topic
  spends.
- **48 — Persistence context**: why the session is closed in the first place, and
  what "detached" means. This topic is the consequence of that one.

**This unlocks:**
- **50 — N+1**: the systematic detection and the full menu of fixes, including the
  fetch-join-plus-pagination trap you will hit in Exercise 3.
- **51 — Caching**: the second-level cache does not cache associations by default,
  which surprises people who expected it to solve lazy loading.
- **52 — Locking**: `@Version` on `Inventory` and `Wallet`; an optimistic-lock retry
  needs a fresh load, which means a fresh persistence context.
- **53 — Batching**: association-heavy writes, and why `IDENTITY` kills batching.
- **54 / 55 — `@Transactional`**: the boundary itself — propagation, `readOnly`, and
  why a transaction pins a connection for its whole life.
- **61 — Testcontainers**: query-count assertions are only truthful against real
  Postgres; H2 accepts SQL Postgres rejects and its plans mean nothing.
- **77 — JMH**: why the timing approach rejected in Measurement is wrong, in detail.
- **109 — HikariCP**: pool sizing, Little's Law, and the pool-versus-thread-pool
  deadlock. The failure drill above is the entry-level version of that lesson.
- **118 — Metrics**: `hikaricp.*` on a dashboard beside RED metrics is the early
  warning for everything in this document.

---

*Java baseline 21, running on JDK 25, Spring Boot 4.1 / Framework 7.0,
`jakarta.persistence.*` throughout. Association semantics, fetch-type defaults and
`LazyInitializationException` are JPA specification behaviour and are identical across
Boot 2.x, 3.x and 4.x; `spring.jpa.open-in-view` has defaulted to `true` on every one
of them. What is version-dependent is diagnostic plumbing: the proxy class-name
marker and the bind-parameter logging category have both changed across Hibernate
versions. Print them and read what your build produces rather than trusting any
document, including this one.*
