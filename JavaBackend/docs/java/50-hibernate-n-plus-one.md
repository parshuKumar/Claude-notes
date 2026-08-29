# 50 — Hibernate III — N+1: detection and the real fixes

## Phase: 5 — Spring Boot & Persistence
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: `GET /orders` — a page of orders with their lines, each line's product, and the order's payment status. This is the endpoint Topic 65 hammers under load, so its statement count is the number that decides whether the gate passes.

---

## Mechanical statement

Every lazy association in Hibernate is a **proxy object** or a **proxy collection**.
The proxy holds the foreign key and nothing else. The moment any code reads a field
through that proxy, Hibernate issues a `SELECT` — right there, synchronously, on the
current JDBC connection.

So a loop over N orders that reads `order.getLines()` issues **1 + N** statements. A
loop that also reads `line.getProduct().getName()` issues one more per distinct
product not already in the persistence context. Nothing throws. Nothing logs at
`WARN`. The endpoint returns the correct JSON. The only signal is **the statement
count**, and you only see it if you ask for it.

That is the whole topic in one sentence: **the bug is invisible at the source level
and visible only at the statement level, so the fix must be proven with a counter,
not with your eyes.**

---

## The bridge from what you know

You already know what N+1 is. You have hit it in Prisma, you have hit it in TypeORM,
you have diagnosed it in raw SQL. **This document does not re-teach the concept.**
It teaches the four things that are genuinely different in Java.

### Difference 1 — Prisma makes N+1 structurally hard. Hibernate makes it structurally easy.

In Prisma, the shape of the fetch is a value you write:

```ts
const orders = await prisma.order.findMany({
  take: 50,
  include: { lines: { include: { product: true } }, payment: true },
});
```

If you did not write `include`, `orders[0].lines` is `undefined`. TypeScript tells you
so. There is no way to accidentally lazy-load, because there is no lazy loading —
Prisma resolves the whole graph up front (as a small fixed number of queries, one per
relation level) and hands you plain objects.

In Hibernate, the shape of the fetch is **not** in the query. It is in the annotations
on the entity, and it is overridden per call site by fetch joins and entity graphs. If
you did not fetch `lines`, `order.getLines()` still works. It returns a real,
populated list. It just cost you a round trip to get there, and you cannot tell from
the call site that it did.

| | Prisma / TypeORM `relations` | Hibernate |
|---|---|---|
| Where the fetch plan lives | in the query, at the call site | split between entity annotations and the query |
| Un-fetched relation | `undefined` (compile error if typed) | works fine, silently issues a SELECT |
| N+1 arises from | writing a loop of `findUnique` on purpose | reading a getter |
| Detection | reading your own code | counting statements |

**Verdict: the concept transfers completely. The failure mode does not.** In Prisma
you cause N+1 by *writing* a loop. In Hibernate you cause it by *not writing*
anything.

### Difference 2 — "detached" objects do not exist in your model

In Node you get plain objects back and the connection is returned to the pool. In
Hibernate the objects you get back are still wired to an open `Session`. That is what
makes lazy loading possible at all — and it is why `spring.jpa.open-in-view` exists,
why it defaults to `true`, and why it is the single worst default in Spring Boot for
this topic. It keeps the session (and the connection) open through JSON serialization,
so Jackson touching `order.getLines()` during response writing triggers the N+1
*after* your service method has returned. Your service looks clean. The N+1 happens in
the serializer.

You met that boundary in Topic 49. Here it is the delivery mechanism for the bug.

### Difference 3 — the fix is not one thing

In Prisma there is essentially one lever: `include`/`select`. In Hibernate there are
five, and they are not interchangeable:

| Fix | Use when |
|---|---|
| `JOIN FETCH` in JPQL | exactly one collection, no pagination |
| `@EntityGraph` on the repository method | same as fetch join, but declarative and per-use-case |
| `@BatchSize` / `default_batch_fetch_size` | several collections, or pagination is required |
| `@Fetch(SUBSELECT)` | one collection, the parent query is cheap to re-run |
| A DTO projection (constructor expression or interface projection) | you never needed the entities at all |

Choosing wrong does not throw. It just gives you a different statement count, or —
worse — the same count plus a `WARN` you did not read.

### Difference 4 — you prove it in a test

This is the actual senior skill. Not "I know what N+1 is". Not "I added a fetch join".
It is: **there is a test in the repository that fails if the statement count for
`GET /orders` goes above 4.** That test is the artefact. Everything else in this
document exists to let you write it.

---

## What is this?

**N+1** in Hibernate is the pattern where one query loads N parent rows, and then N
further queries load one child (or one child collection) each.

Hibernate produces it through three distinct mechanisms, and they need different fixes:

**Mechanism A — lazy collection.** `@OneToMany(fetch = LAZY)` gives you a
`PersistentBag`/`PersistentSet` — a collection proxy that is empty and
"uninitialised". Calling `size()`, iterating it, or `stream()`-ing it triggers
`SELECT * FROM order_line WHERE order_id = ?`. Once per parent.

**Mechanism B — lazy `@ManyToOne`.** `@ManyToOne(fetch = LAZY)` gives you a
runtime-generated subclass of `Product` that holds only the id. Reading `getName()`
triggers `SELECT * FROM product WHERE id = ?`. Once per *distinct* product not already
in the persistence context — the first-level cache deduplicates, which is why this one
has a variable count and is hard to reason about.

**Mechanism C — eager `@ManyToOne`, which is the default.** `@ManyToOne` and
`@OneToOne` are **EAGER by default in JPA**. If you load `OrderLine` entities with a
query, Hibernate must satisfy the eager `product` association. Sometimes it joins.
Sometimes — specifically when the entities come back from a query rather than a
`find()` by id — it issues a separate select per row. This is N+1 caused by an
annotation you never wrote, on associations you never touched.

Mechanism C is why Topic 49's rule ("LAZY everywhere, explicitly") is not a style
preference.

---

## Why does it matter?

Three things, in increasing order of career impact.

**1. It is a latency multiplier that scales with page size.**
Not with data size. With *page size*. A page of 20 is fine in staging. A page of 200
is 10× the statements in production. The endpoint that was 40 ms becomes 400 ms and
nobody changed any code — someone changed a default page size in a config file.

**2. It holds a connection for the whole time.**
Every one of those statements runs on the same pooled connection, serially. A request
issuing 351 statements holds a HikariCP connection for the sum of 351 round trips. At
pool size 10 and 50 concurrent requests you have converted a latency problem into a
saturation problem, and every unrelated endpoint starts timing out too. That is Topic
109's failure, triggered by Topic 50's bug.

**3. It is the most-asked persistence question in senior Java interviews, and the
mid-level answer is disqualifying.**
"N+1 is when you get one query per row, you fix it with a fetch join" is a mid answer.
The senior answer names the trade-offs, names the pagination trap, and — the part
almost nobody says — describes how the fix is *asserted* so it cannot regress.

---

## Machine-level reality

### Statements are round trips, and round trips are the whole cost

A statement against Postgres from a JVM in the same availability zone costs roughly:

```
send request        ~0.1–0.3 ms network
parse + plan        ~0.05–0.5 ms  (prepared statements skip most of parse/plan)
execute             the actual work — often microseconds for a PK lookup
return rows         ~0.1–0.3 ms network
JDBC/Hibernate      ResultSet -> entity hydration, snapshot copy, PC insertion
```

For a primary-key lookup, **execution is not the cost. The round trip is.** Assume a
conservative 0.3 ms per statement of pure overhead on a healthy same-AZ link. Then:

| Statements | Floor latency from round trips alone |
|---|---|
| 1 | 0.3 ms |
| 4 | 1.2 ms |
| 351 | **105 ms** |
| 1,251 (page of 200) | **375 ms** |

None of that shows up in `EXPLAIN ANALYZE`. Each individual query is fast. Postgres is
not the bottleneck; the *number of times you asked it* is. This is why a DBA looking at
slow-query logs will tell you the database is fine, and be correct, while your p99 is
400 ms.

If your database is across an AZ boundary (1–2 ms RTT) or across a region, multiply.
N+1 is the bug that turns a network-topology decision into a customer-visible outage.

### Why Hibernate cannot batch these for you

JDBC batching (Topic 53) works for writes because the driver can send many `INSERT`s in
one round trip and nobody needs a result until the end. It does not work for these
reads because each SELECT's result is needed *before* the next one is issued — your
loop asked for `lines` on order 1, and the loop cannot continue until it has them.
The data dependency is in your control flow, not in Hibernate.

`@BatchSize` is Hibernate's escape hatch: it defers initialisation of *all* the
uninitialised proxies of the same type that are currently in the persistence context,
then loads them with one `WHERE parent_id IN (?, ?, ?, …)`. That works because
Hibernate knows about the other N−1 proxies even though your loop does not.

### Hydration is not free either

Every entity Hibernate returns is put in the persistence context **with a snapshot
copy of every field** (Topic 48). A fetch join returning 50 orders × 5 lines each ×
product = 50 + 250 + up-to-250 entities, each with a snapshot. That is real memory and
real allocation rate, on the request path, feeding the young generation.

This is the honest counter-argument to "just fetch join everything": you traded 351
round trips for one query that hydrates 550 managed entities you are about to throw
away after mapping to a DTO. **When you never needed the entities, the right fix is a
projection, not a fetch join** — one query, no persistence context, no snapshots, no
dirty checking at flush. Hold that thought; it is the production fix in Example 2.

---

## Example 1 — minimal

Two entities. One loop. Count the statements.

```java
package com.orderflow.orders;

import jakarta.persistence.*;
import java.util.*;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
    @SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50)
    private Long id;

    private String customerRef;

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private Set<OrderLine> lines = new LinkedHashSet<>();

    public Long getId() { return id; }
    public Set<OrderLine> getLines() { return lines; }
}
```

```java
package com.orderflow.orders;

import com.orderflow.catalog.Product;
import jakarta.persistence.*;

@Entity
@Table(name = "order_line")
public class OrderLine {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_line_seq")
    @SequenceGenerator(name = "order_line_seq", sequenceName = "order_line_seq", allocationSize = 50)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)          // NOT the default. The default is EAGER.
    @JoinColumn(name = "order_id")
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;

    private int quantity;
    private long unitPriceMinor;                 // money as long minor units — Topic 01

    public Product getProduct() { return product; }
    public int getQuantity() { return quantity; }
}
```

The N+1:

```java
@Transactional(readOnly = true)
public long totalUnits() {
    List<Order> orders = em.createQuery("select o from Order o", Order.class)
                           .setMaxResults(100)
                           .getResultList();     // statement 1

    long total = 0;
    for (Order order : orders) {
        for (OrderLine line : order.getLines()) {   // statement 2 .. 101
            total += line.getQuantity();
        }
    }
    return total;
}
```

**101 statements.** There is no loop over queries in that code. There is one `for`
loop over an in-memory list. The statements come from `order.getLines()`.

The fix, for exactly this shape:

```java
List<Order> orders = em.createQuery("""
        select o
        from Order o
        left join fetch o.lines
        """, Order.class)
    .setMaxResults(100)     // <-- this line is a trap. See Trap 1.
    .getResultList();
```

Without `setMaxResults`, that is **1 statement**. With it, it is 1 statement *and a
warning that Hibernate paginated in memory*, which means it loaded every order in the
table. Read Trap 1 before you use this.

---

## Example 2 — production scenario (on the project spine)

### The requirement

`GET /orders?page=0&size=50` for the `orderflow` customer dashboard. Each item needs:

- order id, customer reference, status, placed-at
- every order line: quantity, unit price, product SKU, product name
- payment status

### The real constraints

- 1,000,000 orders, 5,000,000 order lines, 100,000 products.
- Average 5 lines per order. The tail is long: the p99 order has 40 lines.
- Product access is skewed — roughly 2% of products appear in 60% of lines.
- Page size 50, default sort `placedAt desc`.
- **SLO: p99 < 200 ms**, from the Topic 65 baseline. This endpoint is 20% of the
  traffic mix.
- Postgres is same-AZ; measured statement overhead ~0.3 ms.

### The version that ships

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final OrderRepository orders;

    OrderController(OrderRepository orders) { this.orders = orders; }

    @GetMapping
    public Page<OrderSummary> list(Pageable pageable) {
        return orders.findAll(pageable).map(OrderSummary::from);
    }
}
```

```java
public record OrderSummary(Long id, String customerRef, String status,
                           Instant placedAt, String paymentStatus,
                           List<LineSummary> lines) {

    public static OrderSummary from(Order o) {
        return new OrderSummary(
            o.getId(), o.getCustomerRef(), o.getStatus().name(), o.getPlacedAt(),
            o.getPayment().getStatus().name(),                 // lazy @OneToOne -> SELECT
            o.getLines().stream()                              // lazy collection  -> SELECT
                .map(l -> new LineSummary(
                        l.getQuantity(),
                        l.getUnitPriceMinor(),
                        l.getProduct().getSku(),               // lazy @ManyToOne -> SELECT
                        l.getProduct().getName()))
                .toList());
    }
}
```

This is clean code. It reads well. It passes review. Count the statements:

| Source | Count |
|---|---|
| `findAll(pageable)` — the page query | 1 |
| `findAll(pageable)` — the `count(*)` query `Page` requires | 1 |
| `o.getLines()` — one per order | 50 |
| `l.getProduct()` — one per *distinct* product not already in the persistence context | ~150–250 |
| `o.getPayment()` — one per order | 50 |
| **Total** | **~252–352** |

The product count is the uncomfortable one. 50 orders × 5 lines = 250 product reads,
but the first-level cache deduplicates within the transaction, and product access is
skewed, so you get somewhere between 150 and 250. **It varies with the data.** Your
staging fixture with 3 orders and 4 products shows 12 statements and looks perfectly
healthy.

At 0.3 ms per statement that is **75–105 ms of pure round-trip floor**, plus query
execution, plus hydration of ~550 entities with snapshots. You will not hit a 200 ms
p99 under load, and when the pool saturates you will not hit it at all.

### The fix — and why it is a projection, not a fetch join

Look at what the endpoint actually returns: a flat DTO. It never mutates an `Order`.
It never needs dirty checking, snapshots, or a persistence context. Loading 550
managed entities to build 50 records is work you are paying for and throwing away.

**Step 1 — page the order ids only.** No joins, so pagination is correct at the
database.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("select o.id from Order o order by o.placedAt desc")
    Page<Long> findIdPage(Pageable pageable);
}
```

**Step 2 — one projection query for the header rows, one for the lines.**

```java
public record OrderHeaderRow(Long orderId, String customerRef, OrderStatus status,
                             Instant placedAt, PaymentStatus paymentStatus) {}

public record OrderLineRow(Long orderId, int quantity, long unitPriceMinor,
                           String sku, String productName) {}
```

```java
@Query("""
       select new com.orderflow.orders.OrderHeaderRow(
                o.id, o.customerRef, o.status, o.placedAt, p.status)
       from Order o
       left join o.payment p
       where o.id in :ids
       order by o.placedAt desc
       """)
List<OrderHeaderRow> findHeaders(@Param("ids") List<Long> ids);

@Query("""
       select new com.orderflow.orders.OrderLineRow(
                l.order.id, l.quantity, l.unitPriceMinor, pr.sku, pr.name)
       from OrderLine l
       join l.product pr
       where l.order.id in :ids
       """)
List<OrderLineRow> findLines(@Param("ids") List<Long> ids);
```

**Step 3 — assemble in memory.**

```java
@Service
public class OrderQueryService {

    private final OrderRepository orders;

    OrderQueryService(OrderRepository orders) { this.orders = orders; }

    @Transactional(readOnly = true)
    public Page<OrderSummary> recentOrders(Pageable pageable) {
        Page<Long> idPage = orders.findIdPage(pageable);          // 1 + 1 (count)
        List<Long> ids = idPage.getContent();
        if (ids.isEmpty()) {
            return Page.empty(pageable);
        }

        List<OrderHeaderRow> headers = orders.findHeaders(ids);   // 1
        Map<Long, List<OrderLineRow>> linesByOrder =              // 1
                orders.findLines(ids).stream()
                      .collect(Collectors.groupingBy(OrderLineRow::orderId));

        List<OrderSummary> content = headers.stream()
                .map(h -> OrderSummary.of(h, linesByOrder.getOrDefault(h.orderId(), List.of())))
                .toList();

        return new PageImpl<>(content, pageable, idPage.getTotalElements());
    }
}
```

**Four statements. Constant. Independent of page size, independent of how many lines
an order has, independent of product skew.**

| | before | after |
|---|---|---|
| statements per request | ~252–352 | **4** |
| round-trip floor at 0.3 ms | 75–105 ms | 1.2 ms |
| managed entities hydrated | ~550 | 0 |
| snapshot memory per request | ~550 entity snapshots | none |
| varies with data skew | yes | no |

### Turn off open-in-view first

```yaml
spring:
  jpa:
    open-in-view: false
```

Leave it on and this refactor is pointless: a future colleague returns an entity from
a controller, Jackson walks it during serialization, and the N+1 comes back — outside
your transaction, outside your assertion, inside the response writer.

Boot logs a `WARN` at startup when `open-in-view` is left at its default. Turning it
off is a one-line change and it converts a silent N+1 into a loud
`LazyInitializationException` (Topic 49) that a test will catch.

> `[BOOT 3.x DELTA]` `spring.jpa.open-in-view` defaults to `true` and warns at startup
> on both 3.x and 4.x. The startup warning text has been reworded across versions —
> grep your own startup log for `open-in-view` rather than for an exact sentence.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a fetch join with `Pageable` / `setMaxResults`: pagination happens in memory

This is the most important trap in the document. It is also the one that survives code
review, because the code looks like the textbook fix.

**Wrong:**

```java
@Query("select o from Order o left join fetch o.lines")
Page<Order> findAllWithLines(Pageable pageable);     // page size 50
```

**Exact symptom — four of them, and you need to recognise all four:**

1. **A log line at `WARN` about applying `firstResult`/`maxResults` in memory.** The
   historical text is `HHH000104: firstResult/maxResults specified with collection
   fetch; applying in memory!`.
2. **Latency is stable at first and then collapses as the table grows** — because the
   cost is proportional to the *whole table*, not to the page.
3. **`EXPLAIN ANALYZE` of the generated SQL shows no `LIMIT`.** That is the giveaway:
   the SQL Hibernate sent has no `LIMIT` clause at all.
4. **Heap usage per request scales with total row count.** Under load you get an
   `OutOfMemoryError` from an endpoint that returns 50 items.

> **Version caveat, stated honestly:** the message code and the logging category for
> this warning have moved across Hibernate major versions (the category was under
> `org.hibernate.hql.internal.ast.QueryTranslatorImpl` in the 5.x line and lives under
> the `org.hibernate.orm.*` tree in 6+). **Do not memorise the code.** Verify on your
> version with the config below, which is better than the warning anyway.

**Root cause:** a fetch join on a collection multiplies rows. One order with 5 lines is
5 rows in the result set. A SQL `LIMIT 50` would cut you off in the middle of an order
and return 10 orders with truncated line lists — silently wrong data. Hibernate refuses
to do that. Instead it **removes the `LIMIT`, executes the query against the entire
table, hydrates every matching order, and slices the list in Java.**

At 1,000,000 orders and 5,000,000 lines it tries to materialise all of them to give
you 50.

**Fix — first, make it impossible to ignore:**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        query:
          fail_on_pagination_over_collection_fetch: true
```

That turns the warning into an exception at query time. Set it in every profile
including production. A thrown exception in staging is infinitely cheaper than a heap
exhaustion in production. **Verify the property is honoured on your Hibernate version**
by writing a deliberately-broken query in a test and asserting it throws — if it does
not throw, the property name changed and you need to check
`mvn dependency:tree | grep hibernate-core` and that version's migration guide.

**Fix — then, one of these three, in order of preference:**

**(a) ID-then-fetch, two queries.** Page the ids without a join; fetch the graph for
those ids. This is the general-purpose answer and the one to give in an interview.

```java
@Query("select o.id from Order o order by o.placedAt desc")
Page<Long> findIdPage(Pageable pageable);

@Query("select distinct o from Order o left join fetch o.lines where o.id in :ids")
List<Order> findWithLines(@Param("ids") List<Long> ids);
```

`in :ids` with 50 ids has no row-multiplication problem for pagination, because there
is no `LIMIT` — the `IN` list *is* the limit.

**(b) `@BatchSize`.** Keep the page query simple; let Hibernate batch the collection
initialisations.

```java
@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
@BatchSize(size = 50)                    // org.hibernate.annotations.BatchSize
private Set<OrderLine> lines = new LinkedHashSet<>();
```

Now `findAll(pageable)` is 1 + 1 (count) + **1** for all 50 line collections, because
Hibernate sees 50 uninitialised collection proxies and loads them with
`WHERE order_id IN (?, ?, …)`. Set it globally instead of per-association with:

```yaml
spring.jpa.properties.hibernate.default_batch_fetch_size: 50
```

This is the highest value-per-character change in this entire document, and most teams
have never set it.

**(c) The projection from Example 2**, when you never needed entities.

---

### Trap 2 — fetch-joining two collections: `MultipleBagFetchException`

**Wrong:**

```java
@Query("""
       select o from Order o
       left join fetch o.lines
       left join fetch o.statusHistory
       """)
List<Order> findAllFull();
```

**Exact symptom:** the application **fails to start**, not at call time. Spring Data
parses `@Query` at context initialisation, so you get a `BeanCreationException` whose
root cause is `MultipleBagFetchException: cannot simultaneously fetch multiple bags:
[com.orderflow.orders.Order.lines, com.orderflow.orders.Order.statusHistory]`.

If the query is built at runtime (Criteria, `createQuery`) you get it on first call
instead.

**Root cause:** a "bag" is Hibernate's name for a `List` with no `@OrderColumn` — an
unordered collection that permits duplicates. Fetch-joining two collections produces a
**Cartesian product**: an order with 5 lines and 4 history entries yields 20 rows.
Hibernate can de-duplicate a `Set` (it has element identity) but it cannot
de-duplicate a bag, so it would put 20 lines and 20 history rows into your two lists.
Rather than silently corrupt your data it refuses.

**This is Hibernate protecting you.** The exception is the correct behaviour.

**Fixes, ranked:**

1. **Do not fetch two collections in one query.** Fetch one, `@BatchSize` the other.
   This is almost always right — the Cartesian product is a real cost even when it is
   legal.
2. **Change `List` to `Set`.** Hibernate can then de-duplicate. This makes the query
   *legal* but does not make it *fast*: you still transfer 20 rows to build 9 objects.
   With 40 lines and 20 history entries you transfer 800 rows.
3. **Two separate queries into the same persistence context.** Fetch orders + lines,
   then fetch orders + history for the same ids. The second query's parents are already
   managed, so Hibernate populates the second collection on the same objects. Two clean
   queries, no Cartesian product.

> Note the `equals`/`hashCode` consequence of option 2: putting entities in a `Set`
> makes `hashCode()` load-bearing. Topic 13's rule applies with a Hibernate-specific
> twist — an entity whose `hashCode` depends on a generated `id` changes hash when it
> is persisted, and vanishes from any `HashSet` it was already in. Use the business
> key (`sku`, `orderRef`) or a constant `hashCode` for entities. Do not use the id.

---

### Trap 3 — "I fixed the N+1 by making it EAGER"

**Wrong:**

```java
@OneToMany(mappedBy = "order", fetch = FetchType.EAGER)
private Set<OrderLine> lines = new LinkedHashSet<>();
```

**Exact symptom:** the `GET /orders` statement count drops — and then:

- `GET /orders/{id}/status`, which needs one column, now also loads every line.
- The `count(*)` query that `Page` issues gets slower or gains a join.
- A completely unrelated report that loads 20,000 orders now loads 100,000 lines and
  runs out of heap.
- Statement count for the original endpoint may *increase*, because EAGER on a
  collection is often satisfied by a separate select per parent rather than a join.

**Root cause:** fetch type on the entity is a **global** decision. It applies to every
query in the application, including ones written years later by people who do not know
this annotation exists. You have not chosen a fetch plan; you have removed everyone
else's ability to choose one.

**Fix:** `LAZY` on every association without exception, and choose the fetch plan **per
use case**:

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @EntityGraph(attributePaths = {"lines", "lines.product", "payment"})
    Optional<Order> findWithGraphById(Long id);

    // no graph — the cheap path, for endpoints that need only the header
    Optional<Order> findById(Long id);
}
```

`@EntityGraph` is the declarative form of a fetch join. Same SQL, same pagination trap,
but it composes with derived query methods and does not require you to write JPQL.

> `@ManyToOne` and `@OneToOne` are **EAGER by default in JPA**. You must write
> `fetch = FetchType.LAZY` explicitly on every single one. Checking this is a
> five-minute audit that pays for itself: `grep -rn "@ManyToOne\|@OneToOne" src/main/java`
> and confirm every hit has an explicit `LAZY`.

---

### Trap 4 — the fetch join returns duplicate parents (or does not, depending on version)

**Wrong (or at least, version-dependent):**

```java
List<Order> orders = em.createQuery(
        "select o from Order o left join fetch o.lines", Order.class).getResultList();
// how many elements?
```

**Exact symptom:** `orders.size()` is 5,000,000-ish instead of 1,000,000-ish, and every
order appears once per line. Your total is 5× too high. Or — on a newer Hibernate —
the size is correct and you conclude the problem does not exist, right up until you
run on an older version in another service.

**Root cause:** SQL returns one row per line. Hibernate returns one *result element*
per row, and the object identity guarantee of the persistence context means all rows
for order 42 point at the **same** `Order` instance — so you get the same object five
times, not five copies.

Historically you fixed this with `select distinct o`, plus
`hibernate.query.passDistinctThrough=false` so the `DISTINCT` did not reach the SQL
(where it would force a needless sort of the whole result set). Newer Hibernate lines
de-duplicate root entities automatically, per the JPA specification, and the
`passDistinctThrough` setting was removed.

> **Say this out loud, do not memorise it:** I am not going to assert which behaviour
> your exact Hibernate version has. **Confirm it in one test**, which takes 30 seconds
> and is worth more than any doc:
>
> ```java
> @Test
> void fetchJoinDeduplicatesRootEntities() {
>     // 3 orders, 5 lines each, seeded in @BeforeEach
>     List<Order> result = em.createQuery(
>             "select o from Order o left join fetch o.lines", Order.class).getResultList();
>     System.out.println("size = " + result.size());   // 3 or 15 — now you know
> }
> ```
>
> If it prints 15, add `distinct`. If it prints 3, your version de-duplicates and
> adding `distinct` would push a `DISTINCT` into SQL for nothing.

**Fix:** whichever the test says. And write the test, because this is exactly the kind
of behaviour that changes under you during a Boot upgrade.

---

### Trap 5 — you fixed the collection and forgot the association inside it

**Wrong:**

```java
@EntityGraph(attributePaths = {"lines"})            // lines only
Page<Order> findAllBy(Pageable pageable);
```

then in the mapper: `line.getProduct().getSku()`.

**Exact symptom:** the statement count drops from 352 to ~200 and everyone declares
victory. The endpoint is still 60 ms of round trips. The remaining statements are all
identical in shape — `select ... from product where id=?` — which is the fingerprint.

**Root cause:** the entity graph fetched `lines` but each `OrderLine.product` is still
a lazy proxy. You removed one N and left another.

**Fix:** name the full path in the graph.

```java
@EntityGraph(attributePaths = {"lines", "lines.product", "payment"})
Page<Order> findAllBy(Pageable pageable);
```

**But note what you just did:** that graph fetch-joins one collection (`lines`) plus
two to-ones, and combined with `Pageable` it re-arms Trap 1. Either drop the
`Pageable` and use ID-then-fetch, or put `@BatchSize` on `lines` and leave the graph
out entirely.

**The general lesson, which is the one to carry into an interview:** the statement
count after a fix is not "1". It is "a small constant you can name and defend". When
someone says "I fixed the N+1", the follow-up question is *"what is the count now, and
what happens to it at page size 200?"*

---

## Hands-on proof

You have no output from me here. These are settings you apply and readings you take.
For each one I tell you exactly what to look at and how to interpret every outcome you
might see.

### Setup — the two instruments

`src/main/resources/application-diag.yml`:

```yaml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate:
        generate_statistics: true
        query:
          fail_on_pagination_over_collection_fetch: true
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.stat: DEBUG
```

Run with `--spring.profiles.active=diag`.

Two separate instruments, doing two different jobs:

| Instrument | Answers | Does not answer |
|---|---|---|
| `org.hibernate.SQL=DEBUG` | *what shape* the statements are | how many there were (you cannot count log lines reliably under concurrency) |
| `generate_statistics` + `Statistics` | *how many* — a number you can assert on | what they looked like |

**You need both. The log tells you what to fix. The counter proves you fixed it.**

### Reading the SQL log

An illustration of the **shape** of a Hibernate SQL log line — this is the format, not
captured output:

```
DEBUG o.h.SQL : select o1_0.id,o1_0.customer_ref,o1_0.placed_at,o1_0.status from orders o1_0 order by o1_0.placed_at desc offset ? rows fetch first ? rows only
DEBUG o.h.SQL : select l1_0.order_id,l1_0.id,l1_0.product_id,l1_0.quantity,l1_0.unit_price_minor from order_line l1_0 where l1_0.order_id=?
DEBUG o.h.SQL : select l1_0.order_id,l1_0.id,l1_0.product_id,l1_0.quantity,l1_0.unit_price_minor from order_line l1_0 where l1_0.order_id=?
```

**What to look for:** the same statement text repeated with only the bind parameter
differing. That repetition *is* the N+1 fingerprint, and it is the one thing the log is
genuinely good at.

| What you see in the log | What it means |
|---|---|
| One `select ... from orders`, then many identical `select ... from order_line where order_id=?` | Mechanism A — lazy collection, one SELECT per parent. Fix with `@BatchSize` or a fetch join. |
| Many identical `select ... from product where id=?` | Mechanism B — lazy `@ManyToOne` on `OrderLine.product`. `@BatchSize` on the `Product` **class** fixes it globally. |
| A `select ... from order_line where order_id in (?,?,?,…)` | `@BatchSize` is working. |
| A single query with `left join` and no `LIMIT`/`fetch first` where you passed a `Pageable` | Trap 1. Stop and read it again. |
| No SQL at all when you expected some | You are reading a cached result (Topic 51) or the persistence context already had the entity (L1 hit). |

To see bind parameters as well as the statement:

```yaml
logging.level.org.hibernate.orm.jdbc.bind: TRACE
```

> `[BOOT 3.x DELTA]` The parameter-binding logging category was renamed when Hibernate
> moved to the `org.hibernate.orm.*` tree. On older lines it is
> `org.hibernate.type.descriptor.sql.BasicBinder`. **If neither prints anything on your
> version, set `logging.level.org.hibernate=TRACE` once, find the category name in the
> output, then narrow to it.** That procedure works on every version and is better
> than any category name I could give you.

### Reading the Statistics counters

```java
package com.orderflow.diagnostics;

import jakarta.persistence.EntityManagerFactory;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.springframework.stereotype.Component;

@Component
public class QueryCounter {

    private final Statistics stats;

    public QueryCounter(EntityManagerFactory emf) {
        this.stats = emf.unwrap(SessionFactory.class).getStatistics();
    }

    public void reset() { stats.clear(); }

    public long statements()   { return stats.getPrepareStatementCount(); }
    public long queries()      { return stats.getQueryExecutionCount(); }
    public long entitiesLoaded(){ return stats.getEntityLoadCount(); }
    public long collectionsLoaded() { return stats.getCollectionLoadCount(); }
}
```

**The counter that matters is `getPrepareStatementCount()`.** Understand why:

| Counter | Counts | Trap |
|---|---|---|
| `getPrepareStatementCount()` | every JDBC `PreparedStatement` Hibernate prepared | **this is your N+1 number** |
| `getQueryExecutionCount()` | JPQL/HQL/Criteria/native query executions | **does not count lazy loads.** An N+1 of 350 lazy loads shows as `1` here. Using this counter to prove a fix is the classic mistake. |
| `getEntityLoadCount()` | entities hydrated | rises with fetch joins even when statements fall — useful for spotting the Cartesian product |
| `getCollectionLoadCount()` | collection initialisations | 50 here with 1 statement means `@BatchSize` worked |

Say that middle row out loud once more, because it is a genuine trap and it is the
reason people believe they have fixed an N+1 when they have not:
**`getQueryExecutionCount()` does not see lazy loads.**

### Statement counting from outside Hibernate

Hibernate's counter is Hibernate's own bookkeeping. To count what actually reached the
driver — including statements from Spring Data internals, Flyway, or a `JdbcTemplate`
somewhere in the call path — wrap the `DataSource`:

- **datasource-proxy** or **p6spy** — wrap the `DataSource` bean, log or count every
  statement with timing.
- **Postgres `pg_stat_statements`** — the database's own view. `CREATE EXTENSION
  pg_stat_statements;`, then:

```sql
SELECT calls, rows, mean_exec_time, query
FROM pg_stat_statements
WHERE query LIKE '%order_line%'
ORDER BY calls DESC
LIMIT 20;
```

**How to read it:** a query with `calls` in the hundreds of thousands and a
`mean_exec_time` in microseconds is an N+1 signature. The database will report itself
as healthy — every individual call is fast. `calls` is the column that tells the truth.
Reset the view with `SELECT pg_stat_statements_reset();` before a run.

---

## Failure drill

**This drill is mandatory.** Master plan, Topic 50: *turn on SQL logging and a Hibernate
`Statistics` query counter, load 100 orders, count the queries, fix, re-count. The fix
is only proven by the counter.*

### Setup

You need a Postgres with real data. Use the Topic 61 Testcontainers setup — a shared
singleton container — not H2. H2 will happily run the fetch-join-plus-pagination query
and give you different behaviour.

```java
@SpringBootTest
@Testcontainers
@ActiveProfiles("diag")
class OrderNPlusOneDrillTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17");

    @Autowired EntityManagerFactory emf;
    @Autowired OrderQueryService orderQueryService;
    @Autowired TestDataSeeder seeder;

    Statistics stats;

    @BeforeEach
    void setUp() {
        seeder.seed(100, 5, 40);   // 100 orders, 5 lines each, drawn from 40 products
        stats = emf.unwrap(SessionFactory.class).getStatistics();
        stats.clear();
    }
}
```

Seed **100 orders with 5 lines each drawn from 40 products**. The product count matters:
too few and the L1 cache hides mechanism B; too many and the number is unstable.

### Step 1 — measure the broken version

```java
@Test
void step1_measureTheDamage() {
    stats.clear();
    List<OrderSummary> page = brokenOrderService.list(PageRequest.of(0, 100));

    System.out.println("statements       = " + stats.getPrepareStatementCount());
    System.out.println("queryExecutions  = " + stats.getQueryExecutionCount());
    System.out.println("entitiesLoaded   = " + stats.getEntityLoadCount());
    System.out.println("collectionsLoaded= " + stats.getCollectionLoadCount());
    assertThat(page).hasSize(100);
}
```

**What to capture:** all four numbers, written down. Not "it was a lot" — the numbers.

**How to read what you get:**

| What you see | What it means |
|---|---|
| `statements` ≥ 200, `queryExecutions` = 1 or 2 | Textbook N+1. The gap between the two counters *is* the lazy-loading. This is the reading you want from step 1. |
| `statements` ≈ `queryExecutions` and both small | Something is already fetching eagerly, or `open-in-view` is on and the loading happens after your service returns. Check `spring.jpa.open-in-view: false`. |
| `collectionsLoaded` = 100, `statements` ≈ 100 + something | Mechanism A confirmed: one statement per collection. |
| `entitiesLoaded` ≈ 100 + 500 + 40 | 100 orders, 500 lines, 40 distinct products. Confirms the L1 cache is de-duplicating products. |
| all counters zero | `hibernate.generate_statistics` is not on. Check the profile is active. |

Now correlate with the SQL log: find the repeated statement, and note which table it
hits. That tells you which of mechanisms A/B/C you are looking at.

### Step 2 — apply exactly one fix and re-measure

Change **one thing**, re-run, record. Do this for each fix separately. The point of the
drill is the comparison table, not any single number.

| Fix | Change | Predict the count before you run |
|---|---|---|
| `@BatchSize(size = 50)` on `Order.lines` | one annotation | ? |
| `default_batch_fetch_size: 50` in yml | one property, applies to `Product` too | ? |
| `@EntityGraph({"lines","lines.product","payment"})` | on the repository method | ? |
| ID-then-fetch, two queries | the Example 2 service | ? |
| Full DTO projection | the Example 2 service | ? |

Write your prediction down *before* running. Being wrong is the useful part — it tells
you which mechanism you had mis-modelled.

### Step 3 — the assertion, which is the actual deliverable

The drill is not finished when the number goes down. It is finished when **the number
cannot go back up without a red build.**

```java
@Test
@DisplayName("GET /orders issues a constant 4 statements regardless of page size")
void ordersPageHasConstantStatementCount() {
    seeder.seed(500, 5, 100);

    for (int pageSize : new int[] { 10, 50, 200 }) {
        stats.clear();
        Page<OrderSummary> page = orderQueryService.recentOrders(PageRequest.of(0, pageSize));

        assertThat(page.getContent()).hasSize(pageSize);
        assertThat(stats.getPrepareStatementCount())
            .as("page size %d must not change the statement count", pageSize)
            .isEqualTo(4);
    }
}
```

**Why the loop over page sizes is the important part:** asserting `== 4` at one page
size proves you counted correctly today. Asserting the *same* 4 at page sizes 10, 50
and 200 proves the count is **independent of N**, which is the actual property "no
N+1" means. A test that only checks one page size passes for a fix that is still
linear, just with a smaller constant.

**Constraints on this assertion, which you must know before you ship it:**

- `Statistics` is **per `SessionFactory` and global across threads.** Do not run this
  test class in parallel with other persistence tests. Mark it
  `@Execution(SAME_THREAD)` or keep the suite single-threaded.
- `generate_statistics = true` has measurable overhead. Enable it in the test profile
  and in a diagnostic profile. **Do not leave it on in the production profile** unless
  you have measured the cost at the Topic 65 baseline and accepted it.
- The service call must complete its transaction inside the measured window. With
  `open-in-view: false` and `@Transactional` on the service method, it does.

### What the fix proves

Not "the code is faster". You have not measured time and you should not claim to have.

What it proves is a **structural property**: the number of database round trips for
this endpoint is a constant that does not depend on the size of the result. That is a
statement about the algorithm, and unlike a timing it is exactly reproducible on any
machine, in CI, at any data volume. It is the strongest kind of performance claim you
can make, and it is the reason to assert on counts rather than on milliseconds.

---

## Measurement

### The counters that make the claim falsifiable

```java
Statistics s = emf.unwrap(SessionFactory.class).getStatistics();
```

| Counter | Use it to |
|---|---|
| `getPrepareStatementCount()` | prove the N+1 fix — **the primary number** |
| `getQueryExecutionCount()` | see how many *queries you wrote* ran; the gap to the above is lazy loading |
| `getEntityLoadCount()` | detect the Cartesian product: statements fall, this rises sharply |
| `getCollectionLoadCount()` | confirm `@BatchSize` (many collections, few statements) |
| `getCollectionFetchCount()` | collections that needed their own statement, vs. ones satisfied by a join |
| `getFlushCount()` | catch an accidental write path in a read endpoint |
| `getSessionOpenCount()` | catch `open-in-view` or a stray `@Transactional(REQUIRES_NEW)` |

### Database-side confirmation

```sql
SELECT pg_stat_statements_reset();
-- run the endpoint once
SELECT calls, total_exec_time, mean_exec_time, rows, query
FROM pg_stat_statements
ORDER BY calls DESC LIMIT 10;
```

**How to read it:** for a single request, a row with `calls = 50` and `query LIKE
'%order_line%where order_id = $1%'` is your N+1, stated by the database itself. This is
the reading to bring to a DBA who says the database is fine — they are right about
`mean_exec_time` and wrong about the endpoint.

Use `EXPLAIN ANALYZE` on the **fixed** query, not the broken one. The broken one has
nothing to explain; each statement is a trivial index lookup. On the fixed one, check
that the `IN (...)` list uses an index scan on `order_line(order_id)` and not a
sequential scan — a fetch join into a table with no FK index converts an N+1 into one
enormous sequential scan, which is a different bug with the same latency.

### Why `System.nanoTime()` around this is the wrong instrument

You will be tempted to write:

```java
long t0 = System.nanoTime();
orderQueryService.recentOrders(PageRequest.of(0, 50));
long ms = (System.nanoTime() - t0) / 1_000_000;    // WRONG
```

That number is not measuring what you think, for four independent reasons:

1. **No warmup.** The first few hundred invocations run interpreted or in C1. The JIT
   has not compiled the mapper, the Hibernate hydration path, or Jackson's serializer.
   You are timing a cold JVM, and the cold number can be 10–50× the steady-state one.
2. **One sample.** A single measurement of a latency distribution tells you nothing
   about the distribution. The value you need is p99, and one sample has no p99.
3. **Dead-code elimination and constant folding.** If you do not consume the result,
   the JIT may eliminate work you meant to time. This bites harder in micro-loops than
   in a full service call, but it is real.
4. **Everything else on the box.** Connection acquisition, GC pauses, another test's
   container, your laptop's thermal state. All folded into one number with no way to
   separate them.

**Statement counts have none of these problems.** They are exact, deterministic, and
identical on your laptop and in CI. That is why the assertion is on the count.

When you genuinely need a time:

- **Topic 77 (JMH)** for anything below the service boundary — forks, warmup
  iterations, `Blackhole`, `@State`. Never a hand-rolled loop.
- **Topic 65 (the load baseline)** for the endpoint as a whole — open-model arrival
  rate, realistic dataset, p50/p95/p99/p999. The N+1 fix should move p99 on the
  `GET /orders` scenario, and *that* is the number you put in a PR description.
- **Topic 118 (Micrometer)** for continuous measurement in production.

The honest framing for a PR: *"statement count for `GET /orders` goes from ~350 to a
constant 4, asserted in `OrderNPlusOneDrillTest`. Topic 65 baseline re-run shows p99
moving from X to Y."* The first half is proof. The second half is impact. You need
both, and only the first half is free.

---

## Practice exercises

### 1 — easy: find the mechanism

Seed 20 orders, 4 lines each, 10 products. Turn on `org.hibernate.SQL=DEBUG` and
`generate_statistics=true`.

Write four variants of a method that returns the total quantity across all orders:

- **(a)** iterate `order.getLines()`
- **(b)** `select sum(l.quantity) from OrderLine l` — one aggregate query
- **(c)** `select o from Order o left join fetch o.lines`, then iterate
- **(d)** `@BatchSize(size = 20)` on `Order.lines`, then iterate as in (a)

For each: record `getPrepareStatementCount()`, `getQueryExecutionCount()`,
`getEntityLoadCount()`, and paste the distinct SQL statement shapes.

Then answer in prose: **why is (b) the correct production answer even though (c) and
(d) also have low statement counts?** (Hint: count the rows transferred, and count the
entities hydrated. The right answer involves neither fetch strategy.)

### 2 — medium: combines Topics 13, 26, 47 and 49

Take the `Order`/`OrderLine`/`Product` model and do all four:

**(a)** Change `Order.lines` from `Set` to `List` and add a second collection
`Order.statusHistory` as a `List`. Write a query that fetch-joins both. Capture the
exact exception and its full message. Explain, in terms of rows, what Hibernate refused
to do.

**(b)** Change both to `Set`. The query now runs. Compute — by hand, before running —
how many rows the SQL returns for one order with 6 lines and 3 history entries. Verify
with `getEntityLoadCount()` and by counting rows in a `pg_stat_statements` reading.

**(c)** Give `Order` an `equals`/`hashCode` based on its generated `id`. Create a new
`Order`, add it to a `HashSet`, persist it, then call `set.contains(order)`. Explain
the result using Topic 13's contract. Then write the version that is correct for an
entity, and state the rule as one sentence.

**(d)** Write a repository method returning `Optional<OrderSummary>` where
`OrderSummary` is a Spring Data **interface projection** (Topic 47) rather than a
constructor expression. Compare the generated SQL to the constructor-expression
version. Which columns does each select? Which one would you ship, and does your answer
change if `Order` gains a 2 KB `notes` column?

### 3 — hard: production simulation, advancing the spine

Build the real `GET /orders` endpoint against the Topic 65 dataset shape.

**Part A — seed realistically.** 200,000 orders, 1,000,000 lines, 20,000 products.
Skewed: 400 products ("hot") must appear in 60% of lines. Line counts per order must
follow a long tail — most orders have 2–5 lines, the p99 order has 40. A uniform
fixture will make every fix look equally good and teach you nothing.

**Part B — four implementations**, each behind the same interface:

1. Naive: `findAll(pageable)` + entity-walking mapper.
2. `@EntityGraph` covering `lines`, `lines.product`, `payment`, with `Pageable`.
3. `default_batch_fetch_size: 50` + naive mapper.
4. ID-then-fetch with DTO projections (Example 2).

**Part C — for each, record:** `getPrepareStatementCount()`, `getEntityLoadCount()`,
rows returned by the SQL (from `pg_stat_statements`), and whether any `WARN` appeared
about in-memory pagination. Build the comparison table.

**Part D — the trap.** Implementation 2 will either warn about in-memory pagination or
throw, depending on whether you set
`hibernate.query.fail_on_pagination_over_collection_fetch`. Run it **both ways** at
200,000 orders. Record what happens to heap in the warning case — take a heap histogram
with `jcmd <pid> GC.class_histogram` during the request. Then write two sentences on
why a `WARN` was the wrong severity for this condition.

**Part E — the regression barrier.** Write `OrderListStatementCountTest` asserting a
constant count across page sizes 10/50/200, wire it into CI, and — this is the part
that counts — **deliberately reintroduce the N+1 in a scratch commit and confirm the
test goes red.** A regression test you have never seen fail is not a regression test.

**Part F — argue the other side.** Implementation 4 is fastest and hydrates no
entities. Name two concrete situations in `orderflow` where you would still ship
implementation 3 instead. (Consider: what happens when the endpoint later needs to
*write*; and what happens to `OrderSummary` when `Order` gains three more fields.)

---

## Interview questions

### Q1 — "You have `GET /orders` returning 50 orders with their lines. Walk me through finding and fixing the N+1."

**Mid-level answer:** "I'd turn on SQL logging, see the repeated queries, and add a
`JOIN FETCH` to the query so the lines come back in one go."

**Senior answer:** "First I'd separate detection from proof. SQL logging tells me the
*shape* — I'm looking for the same statement repeated with a different bind parameter,
which tells me which association is lazy. But I'd measure with
`Statistics.getPrepareStatementCount()`, because you cannot count log lines reliably
and because I want a number I can assert on. Note that `getQueryExecutionCount()` is
the wrong counter — it doesn't see lazy loads at all, so an N+1 of 350 shows up as 1
there.

Then the fix depends on what the endpoint does. This one maps straight to a DTO and
never mutates anything, so I'd use a projection: page the ids, then two constructor-
expression queries, assemble in memory. Four statements, constant regardless of page
size, and zero managed entities — no snapshots, no dirty-check work at flush.

If it did need entities, I'd use `@BatchSize` rather than a fetch join, because a fetch
join with `Pageable` makes Hibernate paginate in memory — it strips the `LIMIT`, loads
the whole table and slices in Java.

And the deliverable isn't the fix, it's the test: assert the statement count is a
constant across page sizes 10, 50 and 200. One page size only proves I counted; three
prove the count doesn't depend on N."

**What separates them:** four things. Naming the *wrong* counter and why. Choosing a
projection over a fetch join with a reason (no entities needed). Knowing the
pagination trap unprompted. And treating the regression test as the deliverable rather
than the fix.

**Follow-up the interviewer asks:** *"You said `@BatchSize` — what SQL does that
generate, and what happens if the batch size doesn't divide the number of proxies
evenly?"* They want `WHERE order_id IN (?, ?, …)`, and they want you to know Hibernate
pads or splits the `IN` list into fixed sizes to keep the prepared-statement cache
effective rather than generating a distinct statement per list length.

---

### Q2 — "Why does adding `setMaxResults` to a query with `join fetch` make it slower rather than faster?"

**Mid-level answer:** "I don't think it would — `setMaxResults` should limit the rows."

**Senior answer:** "It gets dramatically slower, and it's a correctness issue first. A
collection fetch join multiplies rows: one order with 5 lines is 5 result rows. A SQL
`LIMIT 50` would cut you off mid-order and hand back orders with truncated line lists —
silently wrong data. Hibernate won't do that, so it drops the `LIMIT` entirely,
executes against the whole table, hydrates every matching order, and applies the offset
and limit in Java.

The symptoms: a `WARN` about applying `firstResult`/`maxResults` in memory, no `LIMIT`
in the logged SQL, latency proportional to total table size rather than page size, and
heap that scales with row count. At a million orders it's an `OutOfMemoryError` from an
endpoint returning 50 items.

I set `hibernate.query.fail_on_pagination_over_collection_fetch=true` in every profile
so it throws instead of warning, because that warning gets lost in a log stream. The
fix is ID-then-fetch: page the ids with no join, then fetch the graph for those ids
with `where id in (:ids)`."

**What separates them:** understanding that it's a **correctness** constraint that
forces the performance behaviour, not a Hibernate limitation. And converting the
warning into a failure as a policy decision.

**Follow-up:** *"Does the same thing happen with a `@ManyToOne` fetch join?"* No — a
to-one join doesn't multiply rows, so `LIMIT` is safe and Hibernate keeps it. Knowing
*why* the two cases differ is the point of the question.

---

### Q3 — "Someone changed an association to `FetchType.EAGER` to fix an N+1 and the P99 got worse. Explain."

**Mid-level answer:** "EAGER loads too much data."

**Senior answer:** "Three separate things happened. First, fetch type on the entity is
global — it applies to every query in the application, including the cheap ones. An
endpoint that needed one column now loads the whole collection. Second, for a
collection, EAGER is often satisfied by a *separate select per parent* rather than a
join, so the statement count can actually go up, not down. Third, if it did join, and
there are two eager collections in the graph, you get a Cartesian product — rows
multiply, wire transfer multiplies, and hydration allocates proportionally.

The underlying mistake is treating fetch type as a fetch *plan*. Fetch type is a
default; the plan belongs at the call site, as an entity graph or a fetch join, chosen
per use case. My rule is LAZY on every association with no exceptions — including the
`@ManyToOne`s and `@OneToOne`s, which are EAGER by default in JPA and are the ones
people forget.

I'd revert the annotation, put the graph on the specific repository method, and add the
statement-count assertion so the next person doesn't need to rediscover this."

**What separates them:** knowing EAGER-on-a-collection can *increase* statements,
knowing `@ManyToOne` defaults to EAGER, and framing it as "fetch type is a default,
fetch plan is per-call-site".

**Follow-up:** *"How would you audit an existing codebase for this?"* A `grep` for
`@ManyToOne`/`@OneToOne` checking each has explicit `LAZY`, plus an ArchUnit test that
fails the build on any association without an explicit fetch type. The ArchUnit answer
is the one that shows you have done this at scale.

---

### Q4 — "How do you stop an N+1 from coming back six months later?"

**Mid-level answer:** "Code review, and keep SQL logging on in dev."

**Senior answer:** "Neither of those survives a busy quarter. Three layers:

**A test that fails.** A statement-count assertion using Hibernate `Statistics` on
every high-traffic read path, run against real Postgres via Testcontainers — not H2,
because H2's behaviour around collection fetches and locking differs. Assert across
several page sizes so the test proves independence from N, not just a lower constant.
And I'd verify the test actually fails by reintroducing the bug once.

**A structural change.** `spring.jpa.open-in-view: false`. With it on, an N+1 can
appear in the Jackson serializer, after the service method returned and outside any
assertion window. Turning it off converts that silent N+1 into a
`LazyInitializationException` that a test catches.

**A production signal.** A Micrometer metric on statements-per-request, or a
`pg_stat_statements` alert on `calls` growth for a given query fingerprint. That
catches the case where a fix is fine at the seeded volume and degrades on real skew.

The code-review layer is real but it's the weakest of the four, because the defect is
invisible at the source level. That's the whole reason this bug class needs a
mechanical barrier."

**What separates them:** proposing a barrier that does not depend on human attention,
and being explicit that review is the weak layer *because of the specific mechanism* of
this bug.

**Follow-up:** *"What's the overhead of leaving `generate_statistics` on in
production?"* The honest answer is "measurable, and I'd measure it at the Topic 65
baseline before deciding" — not a number you invented. If they push, the shape of the
answer is: it is per-statement bookkeeping on the hot path, cheap relative to a
network round trip, but it is not free and it is global mutable state across threads.

---

### Q5 — "When is N+1 acceptable?"

**Mid-level answer:** "It's never acceptable, you should always fix it."

**Senior answer:** "It's acceptable when N is bounded and small and the fix costs more
than the problem. A page of 20 with one extra query per row is 20 statements — about
6 ms of round trips. If that endpoint is 0.1% of traffic and the fix is a
denormalisation with an invalidation story, I'd leave it and write the reason in a
comment.

What is never acceptable is an *unbounded* N. If N comes from a page size, a request
parameter, a batch size, or the row count of a table, the endpoint has no worst case —
it fails when the data grows, which is exactly when you can least afford it. That's the
distinction I'd actually apply: not 'is there an N+1', but 'what bounds N, and who can
change that bound'. If someone can change N by editing a config file, it's unbounded.

The second-order argument is the connection: every one of those N statements holds a
pooled connection for the whole sequence. So an N+1 in a low-traffic endpoint can still
take out high-traffic ones by saturating the pool. That's a systemic risk that doesn't
show up in the endpoint's own latency."

**What separates them:** refusing the absolutist answer, replacing it with a **bounded
vs unbounded** test, and naming the connection-pool coupling — the reason a "harmless"
N+1 is not locally contained.

**Follow-up:** *"Give me an example where the fix is worse than the bug."* A good
answer: fetch-joining two collections to remove 10 statements, producing a Cartesian
product that transfers 800 rows to build 9 objects. Statement count improves; wire
bytes, hydration cost and GC pressure all get worse.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `getQueryExecutionCount()` reports 1 while `getPrepareStatementCount()` reports 350.
   Explain the gap precisely — what kind of database access does the first counter not
   see, and why is that distinction built into Hibernate's model rather than being an
   oversight?

2. A fetch join and `@BatchSize` can both reduce the statement count for the same
   endpoint to a small constant. Under what data shape does `@BatchSize` produce
   *strictly less* work than the fetch join, and under what shape is the fetch join
   strictly better? Answer in terms of rows transferred, not statements.

3. Hibernate refuses to push `LIMIT` into a collection-fetch query. Suppose you were
   designing Hibernate: what would you have to change about the SQL to make `LIMIT`
   correct in that case, and what would that cost? (Consider a window function or a
   lateral join.) Then say why Hibernate does not do that by default.

4. `MultipleBagFetchException` protects you from silent data corruption. Changing
   `List` to `Set` makes the query legal. Has the underlying cost gone away? What
   exactly did the `Set` change, and what did it not change?

5. `spring.jpa.open-in-view` defaults to `true`. Argue the case *for* that default —
   what problem was it solving, for whom, in what era? Then say what changed about how
   we build services that made it the wrong default.

6. You assert `getPrepareStatementCount() == 4` in CI. Name three legitimate code
   changes that would break that test without introducing any performance problem, and
   say how you would tell those apart from a real regression when the build goes red.

7. Prisma has no lazy loading, so it cannot produce this bug the way Hibernate does.
   What does Hibernate buy in exchange for that risk? Name a concrete thing you can do
   with a managed entity graph that you cannot do with Prisma's plain objects — and
   then say whether `GET /orders` needed it.

---

## Quick reference card

### Detect

```yaml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate:
        generate_statistics: true
        query.fail_on_pagination_over_collection_fetch: true
logging:
  level:
    org.hibernate.SQL: DEBUG            # statement shapes
    org.hibernate.stat: DEBUG           # per-session summary
    org.hibernate.orm.jdbc.bind: TRACE  # bind params (category name is version-dependent)
```

```java
Statistics s = emf.unwrap(SessionFactory.class).getStatistics();
s.clear();
// ... exercise the endpoint ...
s.getPrepareStatementCount();   // THE number
s.getQueryExecutionCount();     // does NOT count lazy loads
s.getEntityLoadCount();         // rises on a Cartesian product
s.getCollectionLoadCount();     // many + few statements == @BatchSize working
```

### Fix — choose by situation

| Situation | Fix |
|---|---|
| One collection, no pagination | `join fetch` or `@EntityGraph` |
| One collection, **with** pagination | ID-then-fetch, or `@BatchSize` |
| Several collections | `@BatchSize` on all but one |
| Lazy `@ManyToOne` repeated across rows | `@BatchSize` on the target **class**, or `default_batch_fetch_size` |
| You never needed the entities | DTO projection — constructor expression or interface projection |
| Whole-application default | `spring.jpa.properties.hibernate.default_batch_fetch_size: 50` |

### Annotations

```java
@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
@BatchSize(size = 50)                       // org.hibernate.annotations.BatchSize
private Set<OrderLine> lines = new LinkedHashSet<>();

@ManyToOne(fetch = FetchType.LAZY)          // ALWAYS explicit — the default is EAGER
@JoinColumn(name = "product_id")
private Product product;

@BatchSize(size = 50)                       // on the CLASS: batches all lazy refs to it
@Entity public class Product { ... }

@EntityGraph(attributePaths = {"lines", "lines.product", "payment"})
Optional<Order> findWithGraphById(Long id);

@Fetch(FetchMode.SUBSELECT)                 // one re-run of the parent query as a subselect
private Set<OrderLine> lines;
```

### Gotchas checklist

- [ ] `@ManyToOne` and `@OneToOne` are **EAGER by default**. Every one needs explicit `LAZY`.
- [ ] `getQueryExecutionCount()` does not see lazy loads. Use `getPrepareStatementCount()`.
- [ ] Fetch join + `Pageable` ⇒ in-memory pagination. Make it throw, don't let it warn.
- [ ] Two collection fetch joins ⇒ `MultipleBagFetchException`, or a Cartesian product if you used `Set`.
- [ ] `spring.jpa.open-in-view: false`, always.
- [ ] Entity `hashCode` must not depend on a generated id (Topic 13).
- [ ] Assert the count at **several** page sizes, not one.
- [ ] `Statistics` is global across threads — do not assert counts in parallel tests.
- [ ] Test against real Postgres (Topic 61), not H2.
- [ ] A fetch join replaces round trips with hydration cost. Sometimes that is a bad trade.

---

## When would I use this at work?

**1. Reviewing a PR that adds a "simple" list endpoint.**
Someone returns `repository.findAll(pageable).map(Dto::from)` and the mapper walks two
associations. You do not need to run it; you can count the statements from the diff.
The comment you leave is not "this is N+1" — it is "what is the statement count at page
size 200, and where is the assertion?" That reframes the review from opinion to
measurement, and it is the single highest-leverage habit from this topic.

**2. Diagnosing a p99 regression after a config change.**
Latency doubled, no code shipped. Someone raised a default page size, or turned on a
feature flag that adds a field to a response. `pg_stat_statements` ordered by `calls`
finds it in under a minute, and the fingerprint — a trivially fast query called
hundreds of thousands of times — is unmistakable once you have seen it.

**3. Sizing a new service before it exists.**
Product wants an endpoint returning 500 orders with full line detail. Before writing
any code you can say: naive is ~2,500 statements, ~750 ms of round trips alone, and it
holds a connection the whole time — so at 20 rps that endpoint alone needs 15
connections. That is a design conversation, held before the sprint rather than during
the incident. You will do the full version of this in Topics 109 and 129.

---

## Connected topics

**Prerequisites:**
- **13 — equals/hashCode contract**: entities in a `Set` (which is the fix for
  `MultipleBagFetchException`) make `hashCode` load-bearing. An id-based `hashCode` on
  an entity is a defect.
- **48 — Persistence context and dirty checking**: why every hydrated entity costs a
  snapshot, and why the first-level cache makes the `@ManyToOne` N+1 count vary with
  data.
- **49 — Associations and lazy proxies**: the proxy mechanism that makes N+1 silent,
  and the DTO boundary that makes projections the right answer for read endpoints.
- **47 — Spring Data JPA**: `@EntityGraph`, interface projections, and why derived
  query methods are parsed at startup.

**This unlocks / is used by:**
- **54 — `@Transactional` I**: the transaction boundary is what determines whether a
  lazy load succeeds or throws. Every fix here assumes you know where it starts and
  ends.
- **55 — Isolation and the connection pool**: N+1 holds a pooled connection for the sum
  of all its round trips. That is how a read bug becomes a service-wide outage.
- **61 — Testcontainers**: the statement-count assertion must run against real Postgres.
  H2 gives different behaviour for collection fetches and will pass tests that should
  fail.
- **65 — GATE, load baseline**: `GET /orders` is 20% of the traffic mix. Its statement
  count is the difference between passing and failing the gate.
- **77 — JMH**: why the timing you were about to write around this is not a
  measurement.
- **109 — HikariCP**: the pool arithmetic that turns statements-per-request into a
  concurrency limit.
- **110 — Spring Cache / Redis**: the other way to cut round trips — with an entirely
  different correctness cost, which is Topic 51's subject.
- **118 — Micrometer**: turning statements-per-request into a production signal.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0, `jakarta.persistence.*`.
Hibernate property names and logging categories have moved across major versions; where
this document gives a category or a message code it also gives the procedure to confirm
it on your version. Prefer the procedure. Confirm your exact Hibernate version with
`mvn dependency:tree | grep hibernate-core` before trusting any property name here.*
