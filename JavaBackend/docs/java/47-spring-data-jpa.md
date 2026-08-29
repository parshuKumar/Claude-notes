# 47 — Spring Data JPA: Repository Generation, Derived Queries, `@Query`, Projections

## Phase: 5 — Spring Boot & Persistence
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: `orderflow` repositories over Postgres — `ProductRepository`, `InventoryRepository`, `OrderRepository`, `OrderLineRepository`, `WalletRepository`, `PaymentRepository` — replacing every in-memory stub written in Topics 35–41

---

## ELI5 anchor

Imagine you hire an assistant who cannot be taught, but who reads perfectly.

You do not train them. You hand them a **form** with blank lines on it:

```
findByCustomerIdAndStatusOrderByPlacedAtDesc
```

The assistant reads that line, word by word, and *builds the machine* that does it.
`findBy` — a lookup. `CustomerId` — match this field. `And` — plus. `Status` —
match this one too. `OrderBy PlacedAt Desc` — newest first.

Two things follow from this, and they are the whole topic.

**First:** you wrote no code. There is no body to that method. The machine is built
from the *name*.

**Second — and this is the good part:** the assistant reads every form the moment
you walk in the door, not when you first ask for something. If you write
`findByCustmerId` with a typo, they stop you at the door: *"there is no field called
Custmer."* Not three weeks later at 2am when that endpoint is first called. At the
door.

That is Spring Data JPA. The forms are method names. The door is application
startup.

---

## The bridge from what you know

### Prisma and TypeORM → Spring Data JPA

You have written data access before. Some of it transfers cleanly. One piece has no
counterpart at all.

| You know | Java | Verdict |
|---|---|---|
| `prisma.product.findUnique({ where: { sku } })` | `productRepository.findBySku(sku)` | **PARTIAL** — same intent, but the Java version returns a *managed entity* whose later mutations are written automatically (Topic 48). Prisma returns a plain object. |
| `prisma.product.findMany({ select: { id: true, name: true } })` | An interface or record projection | **PARTIAL** — Prisma's `select` is per-call; a Spring Data projection is a declared type. |
| TypeORM `Repository<T>` with `find`, `findOne`, `save` | `JpaRepository<T, ID>` | **PARTIAL** — closest analogue in your toolkit, but TypeORM's repository is a class you call; Spring Data's is an **interface you never implement**. |
| `prisma.$queryRaw` | `@Query(nativeQuery = true)` | **HONEST ANALOGUE** |
| Prisma migrations | Flyway / Liquibase | **HONEST ANALOGUE** |
| — | **Derived queries: a method *name* parsed into a query at startup** | **NO ANALOGUE** — nothing in Prisma, TypeORM, Sequelize or Knex does this. |
| — | The persistence context and dirty checking | **NO ANALOGUE** — this is Topic 48 and it is the biggest gap in the phase. |

### The one that has no analogue, stated precisely

In Prisma, a query is a value you construct at call time. If you get it wrong, you
find out when the line executes.

In Spring Data, the query is derived from the method *name* by a parser called
`PartTree`, and that parsing happens **during `ApplicationContext` startup**, when
the repository proxy is created. Every property in the name is checked against the
entity's actual metamodel.

So:

```java
List<Order> findByCustmerId(long customerId);   // typo: "Custmer"
```

is not a runtime bug. It is a **startup failure**:

```
org.springframework.data.mapping.PropertyReferenceException:
  No property 'custmer' found for type 'Order'; Did you mean 'customerId'
```

*(That is the shape of the message, quoted from the exception class's documented
format — not captured from a run.)*

**Say this out loud in an interview, because it is a genuine advantage and most
people describe derived queries as merely convenient:** the whole class of "typo in
a query string discovered in production" is eliminated, because the query is
validated before the application accepts its first request. A JPQL string in
`@Query` gets the same treatment — Hibernate parses it at startup too. A *native*
SQL string does not, which is a real cost of `nativeQuery = true`.

### What you must stop assuming

Prisma is **stateless**. `prisma.order.update(...)` issues SQL and returns. Nothing
is remembered.

Spring Data JPA sits on Hibernate, and Hibernate is **stateful**. A repository
`findById` puts the entity into a persistence context that tracks it. If you then
change a field, the change is written at transaction commit **whether or not you
call `save`**. That mechanism is Topic 48. For today, hold this rule:

> `save()` on an entity you just loaded inside the same transaction is a no-op.
> The write happens anyway.

Do not fight it yet. Just stop expecting Prisma semantics.

---

## What is this?

**Spring Data JPA generates the implementation of a repository interface at
runtime.** You declare the interface; you never write a class.

### The mechanism, concretely

At startup, for every interface extending `Repository` (or its subinterfaces):

1. `@EnableJpaRepositories` (implied by Boot's auto-configuration, Topic 42) scans
   for repository interfaces.
2. For each one, a `JpaRepositoryFactoryBean` creates a **JDK dynamic proxy**
   (Topic 40 — interface-based, so JDK not CGLIB) implementing your interface.
3. Calls that match methods on `JpaRepository`/`CrudRepository` are delegated to an
   instance of **`SimpleJpaRepository`**, a real Spring class with real code, which
   uses an `EntityManager`.
4. Calls that match *your declared* methods are routed to a
   `QueryExecutorMethodInterceptor`, which has already resolved each one to a
   `RepositoryQuery` object during startup.

That resolution uses a **query lookup strategy**, and the default is
`CREATE_IF_NOT_FOUND`:

| Order | What is checked |
|---|---|
| 1 | Is there a `@Query` annotation on the method? Use it. |
| 2 | Is there a named query (`@NamedQuery`, or `EntityName.methodName` in `orm.xml`)? Use it. |
| 3 | Otherwise, parse the method name with `PartTree`. |
| 4 | If the name cannot be parsed against the entity's properties, **fail the context**. |

### The interface hierarchy

```
Repository<T, ID>                    marker only; no methods
  └─ CrudRepository<T, ID>           save, findById, findAll, delete, count, existsById
       └─ ListCrudRepository<T, ID>  same, but returns List instead of Iterable
  └─ PagingAndSortingRepository<T, ID>   findAll(Pageable), findAll(Sort)
       └─ ListPagingAndSortingRepository
  └─ JpaRepository<T, ID>            all of the above + flush(), saveAndFlush(),
                                     deleteAllInBatch(), getReferenceById()
```

`JpaRepository` is the one you will extend by default. It is also the one that leaks
JPA specifics into your interface — `flush()` and `getReferenceById()` are not
portable concepts. Some teams extend `ListCrudRepository` deliberately to keep the
persistence technology out of the signature. Both positions are defensible; know
that the choice exists.

### `[BOOT 3.x DELTA]`

- `ListCrudRepository` and `ListPagingAndSortingRepository` arrived in **Spring Data
  3.0 / Boot 3.0**. On Boot 2.x, `CrudRepository.findAll()` returns `Iterable<T>`
  and you write `StreamSupport` boilerplate.
- The `Limit` parameter type (`findByStatus(OrderStatus s, Limit limit)`) arrived in
  **Spring Data 3.2 / Boot 3.2**.
- Boot 4.1 rides **Spring Data 2025.1**. Do not memorise the module version numbers;
  read them from the BOM:
  ```bash
  ./mvnw -q dependency:list | grep -i "spring-data"
  ```
- **Jackson 3 is standard in Boot 4.** If you serialize projections, the
  configuration property names and some package names differ from Jackson 2. This
  bites when you copy a `spring.jackson.*` snippet from a 3.x blog post.

---

## Why does it matter?

**1. It deletes the layer where most data-access bugs live.**

A hand-written DAO is 200 lines of `EntityManager` boilerplate per entity, and every
one of those lines is a place to forget a null check, mistype a JPQL alias, or leave
a `Statement` unclosed. Spring Data replaces it with declarations that are validated
at startup.

**2. The startup validation is a genuine, statable advantage.**

A service that starts is a service whose queries parse. Combine that with a
Testcontainers Postgres in CI (Topic 61) and "the context loads" becomes a real
signal. This is strictly better than the Prisma/TypeORM situation where a mistyped
`where` clause is a runtime error.

**3. Projections are how you keep entities out of your HTTP layer.**

This is the load-bearing reason for a senior engineer. Returning a JPA entity from a
controller drags the persistence context, lazy proxies and your database schema into
your wire format. A projection is a read-only shape that Hibernate can satisfy with
a narrower `SELECT` — fewer columns, no entity in the persistence context, no
snapshot memory (Topic 48), and nothing lazy to explode later (Topic 49).

**4. The abstraction hides cost, and hidden cost is what you are paid to see.**

`findAll()` on a 1,000,000-row table is one method call and one OOM. `Page<T>` runs
a second `COUNT(*)` query that you never wrote. `@Modifying` bypasses the
persistence context and leaves it stale. Every one of these is invisible in the
source. That is the trade Spring Data makes, and the traps section below is the
price list.

---

## Syntax breakdown

### The entities (established here, unchanged in Topics 48 and 49)

These are the `orderflow` entities. Topic 48 explains what every annotation *means*;
today you need them to exist so the repositories have something to query.

```java
package com.orderflow.catalog;

import jakarta.persistence.*;      // NOT javax.persistence - Jakarta EE 11

@Entity
@Table(name = "product")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(name = "product_seq", sequenceName = "product_seq", allocationSize = 50)
    private Long id;

    @Column(nullable = false, unique = true, length = 32)
    private String sku;

    @Column(nullable = false)
    private String name;

    @Column(name = "price_minor", nullable = false)
    private long priceMinor;          // pence. Never double. Topic 01.

    @Column(nullable = false)
    private boolean active;

    protected Product() { }           // JPA requires a no-arg constructor
    // getters, setters, equals/hashCode - Topic 48 covers why these are hard
}
```

```java
package com.orderflow.orders;

@Entity
@Table(name = "orders")               // "order" is a reserved word in SQL
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
    @SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50)
    private Long id;

    @Column(name = "customer_id", nullable = false)
    private Long customerId;

    @Enumerated(EnumType.STRING)      // never ORDINAL
    @Column(nullable = false, length = 20)
    private OrderStatus status;

    @Column(name = "placed_at", nullable = false)
    private Instant placedAt;

    @Column(name = "total_minor", nullable = false)
    private long totalMinor;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();
    // ...
}
```

```java
@Entity
@Table(name = "order_line")
public class OrderLine {
    @Id @GeneratedValue(...) private Long id;

    @ManyToOne(fetch = FetchType.LAZY)      // LAZY is NOT the default. Topic 49.
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    private int quantity;
    @Column(name = "unit_price_minor") private long unitPriceMinor;
}
```

`Inventory` (one row per product: `quantityAvailable`, `quantityReserved`, a
`@Version` field arriving in Topic 52), `Wallet` (`customerId`, `balanceMinor`,
`@Version`) and `Payment` (`order`, `status`, `amountMinor`, `externalRef`) follow
the same shape.

> **Records cannot be JPA entities.** A `record` is implicitly `final`, has no no-arg
> constructor, and has `final` fields. JPA needs to subclass the entity for proxying
> (Topic 49), needs a no-arg constructor to instantiate it, and needs to set fields
> after construction. All three are impossible for a record. Records are excellent
> **projections and DTOs** (this topic) and illegal as entities. Topic 27 gave you
> the record; this is where its boundary is.

### `JpaRepository`

```java
public interface ProductRepository extends JpaRepository<Product, Long> { }
```

| Bit | What it means |
|---|---|
| `interface`, not `class` | You never implement it. The proxy does. |
| `Product` | The entity type. Must be an `@Entity`. |
| `Long` | The type of the `@Id` field. Getting this wrong is a startup failure. |
| No `@Repository` needed | Spring Data registers it. Adding `@Repository` is harmless and some teams do it for readability; it changes nothing functionally. |

You inherit roughly: `save`, `saveAll`, `saveAndFlush`, `findById`, `findAll`,
`findAllById`, `count`, `existsById`, `delete`, `deleteById`, `deleteAllInBatch`,
`flush`, `getReferenceById`.

### Derived query keywords

The subject (`find` / `read` / `get` / `query` / `search` / `stream`), then optional
modifiers, then `By`, then the predicate.

| Keyword | Example method name | Generated predicate |
|---|---|---|
| (equality) | `findBySku` | `sku = ?` |
| `And` / `Or` | `findBySkuAndActive` | `sku = ? and active = ?` |
| `Between` | `findByPlacedAtBetween` | `placed_at between ? and ?` |
| `LessThan` / `GreaterThan` | `findByPriceMinorLessThan` | `price_minor < ?` |
| `LessThanEqual` / `GreaterThanEqual` | `findByPriceMinorGreaterThanEqual` | `price_minor >= ?` |
| `IsNull` / `IsNotNull` | `findByExternalRefIsNull` | `external_ref is null` |
| `Like` / `NotLike` | `findByNameLike` | `name like ?` (you supply the `%`) |
| `StartingWith` / `EndingWith` / `Containing` | `findByNameContaining` | `name like %?%` (Spring supplies the `%`) |
| `IgnoreCase` | `findBySkuIgnoreCase` | `upper(sku) = upper(?)` |
| `In` / `NotIn` | `findBySkuIn(Collection<String>)` | `sku in (?, ?, ...)` |
| `True` / `False` | `findByActiveTrue` | `active = true` |
| `After` / `Before` | `findByPlacedAtAfter` | `placed_at > ?` |
| `OrderBy...Asc/Desc` | `findByCustomerIdOrderByPlacedAtDesc` | adds `order by` |
| `Top` / `First` | `findTop10ByCustomerIdOrderByPlacedAtDesc` | `limit 10` |
| `Distinct` | `findDistinctByStatus` | `select distinct` |
| Nested property | `findByProduct_Sku` | joins to `product` and matches `sku` |

Subject variants that change the return handling:

| Prefix | Returns |
|---|---|
| `countBy...` | `long` |
| `existsBy...` | `boolean` |
| `deleteBy...` / `removeBy...` | `long` or `void` — **loads then deletes one by one** unless annotated `@Modifying @Query` |
| `streamBy...` | `Stream<T>` — must be closed and used inside a transaction |

**The underscore matters.** `findByProduct_Sku` explicitly says "traverse the
`product` association, then its `sku`". Without the underscore, `findByProductSku`
first looks for a property literally called `productSku` and only then falls back to
splitting. Use the underscore when there is any ambiguity; it is also
self-documenting.

### Return types

| Declared return | Meaning |
|---|---|
| `Optional<Product>` | **Use this for a single result.** Topic 26. Zero rows → empty; two rows → `IncorrectResultSizeDataAccessException`. |
| `Product` | Same, but `null` for zero rows. Avoid. |
| `List<Product>` | Zero rows → empty list, never `null`. |
| `Page<Product>` | Content + total element count. **Runs a second `COUNT` query.** |
| `Slice<Product>` | Content + "is there a next page". **No count query.** |
| `Stream<Product>` | Cursor-backed. Needs an open transaction and a `try-with-resources`. |
| `long` / `boolean` | For `countBy` / `existsBy`. |

### `@Query` — JPQL

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("""
           select o from Order o
           where o.customerId = :customerId
             and o.status in :statuses
             and o.placedAt >= :since
           order by o.placedAt desc
           """)
    List<Order> findRecentByCustomerAndStatus(@Param("customerId") Long customerId,
                                              @Param("statuses") Collection<OrderStatus> statuses,
                                              @Param("since") Instant since);
}
```

| Bit | What it means |
|---|---|
| `select o from Order o` | **JPQL, not SQL.** `Order` is the *entity class name*, not the table `orders`. Fields are Java field names, not columns. |
| Text block (`"""`) | Topic 30. Multi-line JPQL without concatenation. |
| `:customerId` | A named parameter. |
| `@Param("customerId")` | Binds the method argument to it. **Required** unless you compile with `-parameters`; Spring Boot's Maven plugin enables that flag, so it often works without — but be explicit, because it fails confusingly when someone builds outside the plugin. |
| Startup validation | Hibernate parses this JPQL at startup. A misspelled field fails the context, exactly like a derived query. |

### `@Query` — native

```java
@Query(value = """
       select p.sku, sum(ol.quantity) as units
       from order_line ol
       join product p on p.id = ol.product_id
       join orders o on o.id = ol.order_id
       where o.placed_at >= :since
       group by p.sku
       order by units desc
       limit :limit
       """,
       nativeQuery = true)
List<TopSellerRow> findTopSellersSince(@Param("since") Instant since,
                                       @Param("limit") int limit);
```

| Difference | Consequence |
|---|---|
| `nativeQuery = true` | The string is passed to the driver verbatim. Table and column names, not entity and field names. |
| **Not validated at startup** | A typo here is a runtime error. This is the real cost of native queries and the reason to prefer JPQL when JPQL can express it. |
| Database-specific | `limit` is Postgres/MySQL; Oracle wants something else. You have chosen your database. That is usually fine — say so deliberately. |
| Pagination | Works, but you must supply a `countQuery` for `Page<T>` because Spring cannot derive a count from arbitrary SQL. |

Reach for native SQL when JPQL genuinely cannot express it: window functions,
CTEs, `INSERT ... ON CONFLICT`, `SKIP LOCKED`, Postgres-specific operators. Not for
"it feels faster".

### `@Modifying`

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("update Product p set p.active = false where p.id in :ids")
int deactivate(@Param("ids") Collection<Long> ids);
```

| Attribute | What it does | Why you need it |
|---|---|---|
| `@Modifying` | Tells Spring this is `executeUpdate()`, not `getResultList()`. | Without it: `Not supported for DML operations`. |
| `flushAutomatically = true` | Flush pending changes **before** running the update. | Otherwise your in-memory changes are written *after* the bulk update and clobber it. |
| `clearAutomatically = true` | Clear the persistence context **after**. | The bulk update bypasses the first-level cache entirely. Without clearing, an entity already loaded still shows `active = true`. This is Trap 4. |

The method must be inside a transaction — put `@Transactional` on the calling
service method (Topic 54), not on the repository.

### Projections

**Interface projection (closed)** — Spring generates a proxy; Hibernate selects only
the named columns.

```java
public interface ProductSummary {
    Long getId();
    String getSku();
    String getName();
    long getPriceMinor();
}
```

```java
List<ProductSummary> findByActiveTrue();
```

**Interface projection (open)** — uses SpEL. **Fetches the whole entity**, because
the expression may touch anything. Convenient and a performance trap.

```java
public interface ProductLabel {
    @Value("#{target.sku + ' - ' + target.name}")
    String getLabel();
}
```

**Class / record projection (DTO)** — the entity is *not* returned; the constructor
is called directly. A `record` is the right shape here (Topic 27).

```java
public record ProductSummaryDto(Long id, String sku, String name, long priceMinor) { }
```

```java
@Query("""
       select new com.orderflow.catalog.ProductSummaryDto(p.id, p.sku, p.name, p.priceMinor)
       from Product p where p.active = true
       """)
List<ProductSummaryDto> findActiveSummaries();
```

The `new fully.qualified.Name(...)` form is a **JPQL constructor expression**. The
package must be fully qualified. It is verbose; it is also the most explicit and the
easiest to read six months later.

**Dynamic projection** — one query, caller picks the shape.

```java
<T> List<T> findByActiveTrue(Class<T> type);
```

```java
List<Product>        full    = repo.findByActiveTrue(Product.class);
List<ProductSummary> summary = repo.findByActiveTrue(ProductSummary.class);
```

| Projection kind | Columns fetched | Entity in persistence context? |
|---|---|---|
| Closed interface | only those named | **no** |
| Open interface (`@Value` SpEL) | **all** | yes |
| Record / class (constructor expression) | only those named | **no** |
| Entity | all, plus eager associations | **yes** — snapshot, dirty checking, the lot |

### `Pageable`, `Page`, `Slice`, `Sort`, `Limit`

```java
Page<Order>  findByCustomerId(Long customerId, Pageable pageable);
Slice<Order> findByStatus(OrderStatus status, Pageable pageable);
List<Order>  findByStatus(OrderStatus status, Sort sort);
List<Order>  findByStatus(OrderStatus status, Limit limit);   // Spring Data 3.2+
```

```java
Pageable page = PageRequest.of(0, 20, Sort.by("placedAt").descending());
```

| Type | Queries issued | Gives you |
|---|---|---|
| `Page<T>` | **2** — the page, then `select count(*)` | `getTotalElements()`, `getTotalPages()` |
| `Slice<T>` | **1** — fetches `size + 1` rows | `hasNext()` only |
| `List<T>` | 1 | nothing about paging |

At 1,000,000 orders, that count query is the expensive half of the request. Use
`Slice` unless the UI genuinely renders "page 47 of 812".

---

## Example 1 — minimal

```java
package com.orderflow.catalog;

import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface ProductRepository extends JpaRepository<Product, Long> {

    Optional<Product> findBySku(String sku);

    boolean existsBySku(String sku);
}
```

That is the entire file. No implementation class, no `@Repository`, no
`EntityManager`.

Using it:

```java
@Service
public class ProductService {

    private final ProductRepository products;

    ProductService(ProductRepository products) {     // constructor injection, Topic 39
        this.products = products;
    }

    @Transactional(readOnly = true)
    public Optional<Product> findBySku(String sku) {
        return products.findBySku(sku);
    }
}
```

Three things worth naming:

1. `Optional<Product>` (Topic 26) makes absence part of the signature. The
   controller turns `Optional.empty()` into the Topic 46 `ProblemDetail` 404.
2. `@Transactional(readOnly = true)` is not decoration. It sets the Hibernate flush
   mode to `MANUAL`, which skips dirty checking on this path entirely (Topic 48),
   and it tells the driver this is a read-only transaction. On a read path serving
   400 rps, that is a real saving.
3. `existsBySku` issues `select count(*) ... where sku = ?` rather than loading the
   row. If you only need a yes/no, do not load the entity.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` at the Topic 65 baseline, on Postgres:

- 100,000 products, 1,000,000 orders, 5,000,000 order lines.
- Skew is real: about 200 products account for 40% of all order lines. A handful of
  customers have over 5,000 orders each.
- Traffic: ~400 rps catalogue reads, ~120 rps order reads, ~60 rps order placements.
- **SLO:** p99 under 250 ms for reads. The database is a single Postgres instance
  with 8 cores; a HikariCP pool of 20 (Topic 109 revisits that number).

At 120 rps of order reads with a p99 budget of 250 ms, you can afford roughly two
database round trips per request and nothing wasteful. A stray `COUNT(*)` over
1,000,000 rows spends most of that budget by itself.

### The catalogue list endpoint — a projection, not an entity

The storefront lists products. It needs `sku`, `name`, `priceMinor` and nothing
else. `Product` also has `costPriceMinor`, `supplierId` and `internalNotes`, which
must never leave the building (this is the same requirement as Topic 46's security
review).

```java
package com.orderflow.catalog;

public record ProductSummary(Long id, String sku, String name, long priceMinor) { }
```

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    Optional<Product> findBySku(String sku);

    boolean existsBySku(String sku);

    /**
     * Catalogue listing. A constructor expression, so Hibernate selects four columns
     * and puts nothing in the persistence context. No snapshot, no dirty checking,
     * no lazy proxy that can explode during serialization (Topic 49).
     */
    @Query("""
           select new com.orderflow.catalog.ProductSummary(p.id, p.sku, p.name, p.priceMinor)
           from Product p
           where p.active = true
           """)
    Slice<ProductSummary> findActiveSummaries(Pageable pageable);

    /** Used by the order-placement path: fetch many products by SKU in one query. */
    List<Product> findBySkuIn(Collection<String> skus);
}
```

Why `Slice` and not `Page`: the storefront shows an infinite-scroll list. It needs
"is there more", not "how many pages". `Page` would add
`select count(*) from product where active = true` to all 400 requests per second.

Why `findBySkuIn` and not a loop of `findBySku`: an order with 5 lines becomes one
query, not five. This is the same idea as the N+1 problem you already know from SQL,
and Topic 50 makes it rigorous for associations.

### The order-read endpoint

```java
package com.orderflow.orders;

public interface OrderRepository extends JpaRepository<Order, Long> {

    Optional<Order> findById(Long id);           // inherited; restated for clarity

    /** Customer order history. Slice: infinite scroll, no count query. */
    Slice<Order> findByCustomerIdOrderByPlacedAtDesc(Long customerId, Pageable pageable);

    /**
     * The admin console genuinely renders "page N of M", so it pays for the count.
     * Note the different return type on an otherwise identical query - the return
     * type is what decides whether the count query runs.
     */
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);

    /**
     * Derived queries stop paying off around three predicates. This one has four
     * and a range, so it is a @Query. Compare it to what the derived name would be:
     *   findByCustomerIdAndStatusInAndPlacedAtBetweenAndTotalMinorGreaterThanEqual
     * That name is unreadable, and renaming a field silently breaks it in a way
     * that is caught at startup but is still miserable to fix.
     */
    @Query("""
           select o from Order o
           where o.customerId = :customerId
             and o.status in :statuses
             and o.placedAt between :from and :to
             and o.totalMinor >= :minTotalMinor
           order by o.placedAt desc
           """)
    List<Order> search(@Param("customerId") Long customerId,
                       @Param("statuses") Collection<OrderStatus> statuses,
                       @Param("from") Instant from,
                       @Param("to") Instant to,
                       @Param("minTotalMinor") long minTotalMinor);

    /** Idempotency check on order placement. Topic 116 makes this a real constraint. */
    Optional<Order> findByIdempotencyKey(String idempotencyKey);
}
```

### The write path

```java
public interface InventoryRepository extends JpaRepository<Inventory, Long> {

    Optional<Inventory> findByProductId(Long productId);

    List<Inventory> findByProductIdIn(Collection<Long> productIds);

    /**
     * The atomic conditional decrement. This is a native query on purpose:
     * JPQL cannot express "update only if the current value is sufficient, and
     * tell me how many rows you touched".
     *
     * clearAutomatically = true is MANDATORY here. Without it, an Inventory
     * already loaded in this persistence context still reports the OLD quantity
     * for the rest of the transaction. See Trap 4.
     *
     * Topic 52 compares this against @Version and PESSIMISTIC_WRITE with numbers.
     */
    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query(value = """
           update inventory
              set quantity_available = quantity_available - :qty
            where product_id = :productId
              and quantity_available >= :qty
           """, nativeQuery = true)
    int tryDecrement(@Param("productId") Long productId, @Param("qty") int qty);
}
```

```java
public interface WalletRepository extends JpaRepository<Wallet, Long> {
    Optional<Wallet> findByCustomerId(Long customerId);
}

public interface PaymentRepository extends JpaRepository<Payment, Long> {
    Optional<Payment> findByOrderId(Long orderId);
    Optional<Payment> findByExternalRef(String externalRef);
    List<Payment> findByStatusAndCreatedAtBefore(PaymentStatus status, Instant cutoff);
}
```

### Replacing the stubs

Topics 35–41 wired `InMemoryProductRepository` and friends. The swap is:

1. Delete the in-memory classes.
2. Make sure the *interface* your services depend on is the Spring Data interface,
   or that your own interface is what `ProductRepository extends JpaRepository`
   implements. Constructor injection (Topic 39) means nothing else changes.
3. Add Flyway migrations for the tables. **Set `spring.jpa.hibernate.ddl-auto=validate`
   in every environment including local.** `update` silently mutates your schema and
   produces a database nobody can recreate; `create-drop` deletes data. `validate`
   fails loudly when the entity and the schema disagree, which is exactly what you
   want at startup.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — returning entities from read endpoints

**Wrong:**

```java
@GetMapping("/api/orders/{id}")
public Order getOrder(@PathVariable Long id) {
    return orderRepository.findById(id).orElseThrow(() -> new OrderNotFoundException(id));
}
```

**Exact symptom:** one of three, and which one you get depends on configuration.

1. `org.hibernate.LazyInitializationException: could not initialize proxy [com.orderflow.orders.Order.lines] - no Session` — a 500 whose stack trace is inside Jackson, not inside your code.
2. With `open-session-in-view` enabled (Boot's default is `true` and it logs a
   warning at startup), no exception — instead the response contains every order
   line, every product on every line, and possibly the inventory of each. A 400-byte
   response becomes 90 KB, and the SQL log shows dozens of queries per request.
3. `StackOverflowError` during serialization, if `OrderLine` has a back-reference to
   `Order` and both are serialized.

**Root cause:** an entity is a persistence type with lazy associations and a
lifecycle. Jackson walks it depth-first with no idea that touching a getter can
issue SQL.

**Fix:** return a DTO or projection. The controller maps the entity to a record
inside the transaction. This is the whole substance of Topic 49, and the boundary is
non-negotiable.

---

### Trap 2 — `Page<T>` where `Slice<T>` would do

**Wrong:**

```java
Page<Order> findByCustomerIdOrderByPlacedAtDesc(Long customerId, Pageable pageable);
```

for an infinite-scroll UI that never displays a total.

**Exact symptom:** in the SQL log, **two** statements per request — the `select`, and
then a `select count(o1_0.id) from orders o1_0 where o1_0.customer_id = ?`. The
endpoint's p99 sits at roughly double what the page query alone costs. For a
customer with 5,000 orders it is worse than double, because the count cannot use a
covering index the way the limited page query can.

**Root cause:** `Page` promises `getTotalElements()`. Spring Data has no way to know
you never call it, so it always runs the count.

**Fix:** return `Slice<T>`. It fetches `size + 1` rows and reports `hasNext()`. One
query. If you genuinely need a total, consider an approximate count from
`pg_class.reltuples` for large tables, or cache the total.

---

### Trap 3 — the unbounded `findAll`

**Wrong:**

```java
List<Product> all = productRepository.findAll();
return all.stream().filter(p -> p.isActive()).toList();
```

**Exact symptom:** works perfectly in dev with 40 seeded products. In production
with 100,000 products: a several-second request, a spike in old-generation heap, and
under memory pressure an `OutOfMemoryError: Java heap space`. Because every one of
those 100,000 entities is *managed*, Hibernate also holds a **snapshot** of each
(Topic 48), so the memory cost is roughly double what you estimated.

**Root cause:** filtering in Java what the database can filter in SQL, and no bound
on the result set.

**Fix:** push the predicate into the query and bound the result.

```java
Slice<ProductSummary> findActiveSummaries(Pageable pageable);
```

**The review rule:** any repository method returning `List<T>` with no `Pageable`,
no `Limit`, and no `Top`/`First` needs a written justification that the result set is
bounded by a business constraint. "There will never be many" is not one.

---

### Trap 4 — `@Modifying` without `clearAutomatically`

**Wrong:**

```java
@Modifying
@Query("update Product p set p.active = false where p.id = :id")
void deactivate(@Param("id") Long id);
```

used like this:

```java
@Transactional
public void deactivateAndAudit(Long id) {
    Product product = productRepository.findById(id).orElseThrow();
    productRepository.deactivate(id);
    auditLog.record("deactivated", product.isActive());   // logs TRUE
}
```

**Exact symptom:** the database row is correctly `active = false`, the SQL log shows
the `update` executing — and the audit entry says `true`. No exception. The bug is
detected weeks later when someone reconciles the audit log against the data.

**Root cause:** a bulk `@Modifying` query goes straight to the database and does not
touch the persistence context. The `Product` you loaded is still in the first-level
cache with its old field values, and every subsequent read in this transaction is
served from that cache rather than from the database. Topic 48 explains the
first-level cache; this is the first time it bites you.

**Fix:** `@Modifying(clearAutomatically = true, flushAutomatically = true)`. Then be
aware of the consequence: clearing detaches **every** entity in the persistence
context, so anything you were holding is now detached and its pending changes are
lost. That is Topic 48's detached state, and it is why bulk updates belong in their
own narrow transaction rather than in the middle of a business method.

---

### Trap 5 — offset pagination at depth, and the deep-page cliff

**Wrong:** `PageRequest.of(4000, 20)` against `orders`.

**Exact symptom:** page 1 responds in 8 ms; page 4,000 responds in 900 ms. The SQL
is identical apart from `offset 80000`. Nothing in the application log explains it.
`EXPLAIN (ANALYZE, BUFFERS)` shows the plan reading and discarding 80,000 rows.

**Root cause:** `OFFSET n` in Postgres means "produce the rows and throw the first n
away". Cost grows linearly with the offset. This is a database fact, not a Spring
Data fact — but Spring Data's `Pageable` makes it so easy to write that nobody
notices.

**Fix:** keyset (seek) pagination. Instead of "page 4000", ask for "the 20 orders
placed before this timestamp and id":

```java
@Query("""
       select o from Order o
       where o.customerId = :customerId
         and (o.placedAt < :beforeAt
              or (o.placedAt = :beforeAt and o.id < :beforeId))
       order by o.placedAt desc, o.id desc
       """)
List<Order> findPageAfter(@Param("customerId") Long customerId,
                          @Param("beforeAt") Instant beforeAt,
                          @Param("beforeId") Long beforeId,
                          Limit limit);
```

With an index on `(customer_id, placed_at desc, id desc)` this is constant time at
any depth. You already know this from your SQL background; the point is that Spring
Data does not warn you.

---

## Hands-on proof

Everything below is a command **you** run. I have no JVM, no Spring app and no
Postgres, so nothing here is presented as captured output. What follows is exactly
what to run and how to read every outcome.

### Setup — turn on the *right* SQL logging

```properties
# application-local.properties

# --- do NOT rely on this one ---
# spring.jpa.show-sql=true

# --- use these instead ---
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE
```

**Why `show-sql` is inferior, precisely:**

| `spring.jpa.show-sql=true` | `logging.level.org.hibernate.SQL=DEBUG` |
|---|---|
| Writes to `System.out`, bypassing your logging framework | Goes through SLF4J/Logback like everything else |
| No timestamp, no thread name, no MDC, no correlation ID | Full log pattern, so you can correlate with a request |
| Cannot be turned on per-package or at runtime | Can be changed at runtime via the Actuator loggers endpoint |
| Cannot be captured by your log aggregator | Ships to your aggregator with everything else |
| Shows the statement with `?` placeholders only | Pairs with the bind logger to show real parameter values |

`show-sql` is a five-minute-demo feature. Never enable it in a shared environment.

**`[BOOT 3.x DELTA]` — the bind-parameter logger category has changed across
Hibernate versions.** On Hibernate 5 it was
`org.hibernate.type.descriptor.sql.BasicBinder`; on Hibernate 6+ it is
`org.hibernate.orm.jdbc.bind`. I am not going to assert which one your build
resolves to. **Confirm it on your machine:**

```bash
./mvnw -q dependency:list | grep -i hibernate-core
```

Then set the category that matches your major version, restart, and check that bind
values actually appear. If they do not, you have the wrong category — try the other
one. This is a two-minute check and it is more reliable than any blog post.

**Never enable bind logging in production.** Bound parameters include customer IDs,
email addresses and payment references.

### Proof 1 — the startup validation is real

Deliberately break a derived query:

```java
List<Order> findByCustmerId(Long customerId);   // typo
```

```bash
./mvnw -q spring-boot:run
```

**What to look for:** whether the application starts at all.

| What you see | What it means |
|---|---|
| Startup fails with `PropertyReferenceException: No property 'custmer' found for type 'Order'` | The expected result. The parser validated the name against the entity metamodel. This is the advantage over Prisma, demonstrated. |
| Startup fails with a different message naming `orderRepository` | Still a startup failure — read the `Caused by` chain to the root. |
| **Startup succeeds** | You did not actually break anything Spring Data parses. Check you edited a *declared* method and not an inherited one, and that the repository is being scanned. |

Now do the same with a JPQL `@Query` (misspell a field) — it should also fail at
startup. Then do it with `nativeQuery = true` — it should **start fine and fail at
call time**. That contrast is the argument for preferring JPQL, made on your own
machine.

### Proof 2 — count the queries a `Page` costs

```java
@Test
void pageIssuesACountQuery() { /* call the Page-returning method */ }
```

with SQL logging on, then:

```bash
./mvnw -q test -Dtest=OrderRepositoryTest 2>&1 | grep -c "^.*org.hibernate.SQL"
```

**What to look for:** the number of statements per repository call.

| What you see | What it means |
|---|---|
| 2 statements for a `Page<T>` call, 1 for the equivalent `Slice<T>` | Confirms Trap 2 on your own code. |
| 1 statement for `Page<T>` | Spring Data optimises away the count when the result fits in a single page (page 0 and fewer than `size` results). Try a page size smaller than your seed data. |
| Many more than expected | You are loading entities with associations. That is Topic 50, and grepping a log is the wrong tool — see Measurement below and Topic 50's statistics counter. |

Counting log lines with `grep -c` works for a single test. It does not scale and it
is not an assertion. Topic 50 replaces it with a query counter you can assert on.

### Proof 3 — prove a projection selects fewer columns

Run the entity-returning method and the projection-returning method against the same
table, with `logging.level.org.hibernate.SQL=DEBUG`.

**What to look for:** the column list after `select`.

| What you see | What it means |
|---|---|
| Projection query lists only your four columns | A closed interface or constructor-expression projection. Correct. |
| Projection query lists every column | You used an **open** interface projection (one with `@Value`/SpEL). Open projections fetch the whole entity. Convert to a closed interface or a record. |
| Projection query has extra joins | Your projection names a nested property that requires a join. Sometimes fine; check the plan. |

### Proof 4 — measure it in the database, not the log

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.* FROM orders o
WHERE o.customer_id = 4471
ORDER BY o.placed_at DESC
LIMIT 20 OFFSET 80000;
```

**What to look for:** `Rows Removed by Filter`, the buffer counts, and whether the
plan is an index scan or a sequential scan.

| What you see | What it means |
|---|---|
| An index scan with a small `shared hit` count | Healthy. |
| A large number of rows produced then discarded for the offset | Trap 5. Move to keyset pagination. |
| `Seq Scan on orders` | No usable index. Spring Data will never tell you this; only the plan will. |

You already know how to read this. The point of including it is that **Spring Data
gives you no visibility into query cost at all** — you must go to the database.

### Proof 5 — real query counting with `datasource-proxy`

Grepping logs does not scale. Wrap the `DataSource` and count for real.

```xml
<dependency>
  <groupId>net.ttddyy</groupId>
  <artifactId>datasource-proxy</artifactId>
  <version><!-- check Maven Central for the current version --></version>
  <scope>test</scope>
</dependency>
```

```java
@TestConfiguration
class QueryCountingConfig {

    @Bean
    static BeanPostProcessor dataSourceProxy() {
        return new BeanPostProcessor() {
            @Override
            public Object postProcessAfterInitialization(Object bean, String name) {
                if (bean instanceof DataSource ds) {
                    return ProxyDataSourceBuilder.create(ds)
                            .name("orderflow-test")
                            .countQuery()
                            .build();
                }
                return bean;
            }
        };
    }
}
```

Then in a test, reset the counter, call the repository, and assert on the count.
p6spy is the alternative and does the same job with a different configuration style.

**What to look for:** an exact number you can put in an `assertThat`. That is the
difference between "I looked at the log and it seemed fine" and a regression that
cannot come back. Topic 50 builds this into a reusable test rule; Topic 61 runs it
against a real Postgres via Testcontainers so the counts are truthful.

---

## Practice exercises

### 1 — Easy: the repository layer

Create `ProductRepository` and `OrderRepository` for `orderflow` with:

- `Optional<Product> findBySku(String sku)`
- `boolean existsBySku(String sku)`
- `List<Product> findBySkuIn(Collection<String> skus)`
- `Slice<Order> findByCustomerIdOrderByPlacedAtDesc(Long customerId, Pageable pageable)`

Then:

1. Deliberately misspell a property in one derived method. Run the app. Paste the
   startup failure and identify the exception type.
2. Fix it. Add a JPQL `@Query` with a deliberately misspelled field. Run again.
   Note whether it fails at startup.
3. Add a `nativeQuery = true` method with a misspelled column. Run again. Note when
   it fails.

**Write down:** which of the three failed at startup and which at call time, and what
that tells you about when to reach for a native query.

### 2 — Medium: combining earlier topics (01–46)

Build the catalogue listing endpoint end to end.

Requirements:

- A `record ProductSummary(Long id, String sku, String name, long priceMinor)`
  (Topic 27). Write a one-paragraph note in the code explaining why this record could
  not be the `@Entity` itself.
- A JPQL constructor-expression query returning `Slice<ProductSummary>`.
- The service returns `Optional` where a single product may be absent (Topic 26); the
  controller converts absence to a Topic 46 `ProblemDetail` 404 with `type`
  `.../product-not-found`.
- `priceMinor` is a `long` in pence, never a `double` (Topic 01). The response DTO
  formats it for display; the storage type never changes.
- Constructor injection throughout (Topic 39). No field injection.
- `@Transactional(readOnly = true)` on the read path, with a comment saying what it
  actually changes.

**Then measure.** With `logging.level.org.hibernate.SQL=DEBUG`, compare the
generated SQL for:
- `Slice<ProductSummary>` via constructor expression,
- `Page<ProductSummary>` via the same,
- `Slice<Product>` returning the entity.

Report the column lists and the statement counts. Explain the difference between the
first two in one sentence, and between the first and third in one sentence.

### 3 — Hard: production simulation

Replace every in-memory stub from Topics 35–41 with a real Postgres-backed
repository, and prove the result is not slower than the stub for the wrong reasons.

**Part A — schema and seed.** Flyway migrations for `product`, `inventory`, `orders`,
`order_line`, `wallet`, `payment`. Set `spring.jpa.hibernate.ddl-auto=validate`.
Seed 100,000 products, 1,000,000 orders and 5,000,000 order lines, with realistic
skew: about 200 products should carry 40% of the lines, and a few customers should
have more than 5,000 orders each. (Generating this efficiently is itself a lesson —
Topic 53 explains why a naive JPA insert loop takes hours.)

**Part B — the endpoints.**
- `GET /api/products?page=&size=` → `Slice<ProductSummary>`.
- `GET /api/orders?customerId=&page=&size=` → `Slice<OrderSummary>`.
- `POST /api/orders` → uses `findBySkuIn` for pricing and `tryDecrement` for
  inventory.

**Part C — the measurements.** For each read endpoint, record:
1. The exact statements issued (SQL logging).
2. The query count (datasource-proxy or p6spy).
3. `EXPLAIN (ANALYZE, BUFFERS)` for the generated SQL at page 0 and at page 4,000.
4. Wall-clock p50/p95/p99 under a small k6 script at 50 rps.

**Part D — fix what you find.** You will find at least: a missing index, a `Page`
that should be a `Slice`, and a deep-offset cliff. Fix each, re-measure, and report
before/after. State which fix mattered most and why.

**Part E — write the honest caveat.** Your p99 numbers here are wall-clock timings
from a load script on a laptop. Write three sentences on why they are not
trustworthy as absolute numbers, what they *are* good for, and which topic
(65 for the load methodology, 77 for microbenchmarking) makes them rigorous.

**Part F — the argument.** Your team lead proposes dropping Spring Data and hand-
writing DAOs with `JdbcClient` "for control". Write the strongest case *for* their
position and the strongest case against. Then state what you would actually do and
under what condition your answer flips.

---

## Interview questions

### Q1 — "How does Spring Data JPA turn an interface into a working repository?"

**Mid-level answer:** "Spring generates an implementation at runtime based on the
method names and the entity type."

**Senior answer:** "At context startup, a `JpaRepositoryFactoryBean` creates a JDK
dynamic proxy for the interface — interface-based, so JDK proxies rather than CGLIB,
which is Topic 40's distinction. Inherited `CrudRepository` methods delegate to a
real `SimpleJpaRepository` instance wrapping an `EntityManager`. Declared methods go
through a `QueryExecutorMethodInterceptor` that resolved each one during startup
using the `CREATE_IF_NOT_FOUND` lookup strategy: `@Query` first, then a named query,
then the `PartTree` name parser. The important consequence is that resolution
happens at startup, so a bad method name or bad JPQL fails the context rather than a
request. Native SQL is the exception — it isn't parsed, so it fails at call time."

**What separates them:** naming the actual collaborators, the lookup strategy, and
turning "it's generated" into a *falsifiable* claim about when failures surface.

**Follow-up:** "Which proxy type, and why does it matter?" JDK dynamic proxy,
because the target is an interface. It matters because Topic 40's self-invocation
trap does not apply here (you never call your own repository methods from inside a
repository), but it does explain why you cannot put a `@Transactional` default
method on the interface and expect it to be advised the way you would like.

---

### Q2 — "When does a derived query stop being the right choice?"

**Mid-level answer:** "When the method name gets too long to read."

**Senior answer:** "Around three predicates, empirically — past that the name is
unreadable and, worse, unreviewable, because nobody can check
`findByCustomerIdAndStatusInAndPlacedAtBetweenAndTotalMinorGreaterThanEqual` against
intent in a diff. But the sharper rule is capability, not length: derived queries
cannot express joins with fetch semantics, subqueries, aggregations, `case`, or
anything database-specific. Once I need a fetch join or a projection with a
constructor expression, it's `@Query` with JPQL. If JPQL can't express it —
window functions, CTEs, `on conflict`, `skip locked` — it's native, and I accept
that I've lost startup validation and portability, and I document why."

**What separates them:** a capability-based rule rather than an aesthetic one, and
naming what native SQL *costs* rather than treating it as a free escape hatch.

**Follow-up:** "What do you lose with `nativeQuery = true`?" Startup validation,
portability, and JPQL's automatic entity mapping — plus you must supply a
`countQuery` yourself if you want a `Page`.

---

### Q3 — "Why not return the entity from your read endpoints?"

**Mid-level answer:** "Because you should use DTOs to decouple the API from the
database."

**Senior answer:** "Three concrete reasons, in order of how badly they bite. One:
lazy associations. Jackson walks getters, a lazy proxy issues a SELECT when touched,
and if the transaction has closed you get `LazyInitializationException` — a 500 from
serialization, with a stack trace that points nowhere useful. Two: the object graph.
Even when it works, you serialize far more than you meant to, and a bidirectional
association gives you infinite recursion. Three: the entity carries internal columns
— cost price, supplier ID — that are now in your public API by accident. A closed
interface projection or a record via a constructor expression fixes all three and
also narrows the SELECT, so it's less I/O and no snapshot memory in the persistence
context. The decoupling argument is true but it's the weakest of the four."

**What separates them:** ranking concrete failure modes above the architectural
platitude, and knowing the projection is a *performance* win as well as a design one.

**Follow-up:** "What about `@JsonIgnore` on the lazy field?" It works and it is a
trap: it fixes today's endpoint and leaves the entity in the serialization layer, so
the next field someone adds reintroduces the bug. Also `@JsonIgnore` is now a
persistence class carrying HTTP-layer annotations.

---

### Q4 — "What is the difference between `Page` and `Slice`?"

**Mid-level answer:** "`Page` knows the total number of elements, `Slice` only knows
whether there's a next page."

**Senior answer:** "The difference in the API is the total; the difference that
matters is that `Page` issues a second `select count(*)` on every call. On a
1,000,000-row table with a customer filter, that count is frequently the more
expensive of the two queries, because it can't be satisfied by the same limited
index scan the page query uses. So the rule is: `Slice` unless the UI genuinely
renders 'page 47 of 812'. For infinite scroll, `Page` is pure waste. And if I need a
total on a large table I'd look at whether an approximate count from `reltuples` is
acceptable, or cache it — an exact count of a mutable million-row table is a
question with an expensive answer and usually a low-value one."

**What separates them:** naming the second query as the real cost, and treating
"do we need an exact total" as a product question.

**Follow-up:** "Spring Data sometimes skips the count query — when?" When the result
is known to fit: page 0 with fewer results than the page size, or a last page it can
infer. Good candidates volunteer that this makes naive log-based verification
misleading.

---

### Q5 — "You added a `@Modifying` bulk update and now an audit log shows the wrong value. What happened?"

**Mid-level answer:** "Probably a caching issue — maybe the entity is stale."

**Senior answer:** "It's the first-level cache. A `@Modifying` bulk update executes
directly against the database and never goes through the persistence context, so any
entity already loaded in that context keeps its old field values, and every read of
it for the rest of the transaction is served from the identity map rather than from
the database. The fix is `@Modifying(clearAutomatically = true, flushAutomatically =
true)` — flush first so pending dirty-checked changes aren't written *after* the
bulk update and clobber it, clear afterwards so stale entities are dropped. The
consequence to be aware of is that clearing detaches *everything* in the context,
including entities the caller still holds, so their pending changes are silently
lost. That's why I'd keep a bulk update in its own narrow transaction rather than
mixed into a business method."

**What separates them:** naming the first-level cache as the mechanism, explaining
why *both* attributes are needed for different reasons, and volunteering the
collateral damage of `clearAutomatically`.

**Follow-up:** "What is the first-level cache, exactly?" That is Topic 48, and this
question is the natural bridge into it.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A derived query name is validated at startup. A JPQL `@Query` is validated at
   startup. A native `@Query` is not. What would it take for Spring to validate the
   native one too, and why do you think it does not try?

2. `existsBySku` and `findBySku(...).isPresent()` return the same boolean. Describe
   every way they differ at the database, in memory, and in the persistence context.

3. `Page<T>` runs a count query; `Slice<T>` does not. Design a third return type
   that would be strictly better than both for infinite scroll. What information
   would it carry, and what would it cost?

4. A record cannot be a JPA entity but is an excellent projection. State the three
   specific properties of records that make each half of that sentence true.

5. Your repository interface extends `JpaRepository`, which exposes `flush()` and
   `getReferenceById()` to every caller. Argue for extending `ListCrudRepository`
   instead. Then argue against. Which do you actually pick for `orderflow`, and why?

6. Spring Data hides the query. Name three costs that become invisible as a result,
   and for each, name the tool that makes it visible again.

7. You are told "we don't need pagination, this table only has 300 rows". Under what
   circumstances is that a reasonable engineering decision, and what would you write
   down so that it stays reasonable?

---

## Quick reference card

### Interfaces

```java
interface X extends JpaRepository<Entity, IdType>          // the default choice
interface X extends ListCrudRepository<Entity, IdType>      // keeps JPA out of the signature
```

### Derived-query cheat sheet

```
findBy / readBy / getBy / queryBy / searchBy / streamBy
countBy / existsBy / deleteBy / removeBy
Distinct  Top<N>  First<N>
And  Or  Between  LessThan  LessThanEqual  GreaterThan  GreaterThanEqual
Is  Equals  IsNull  IsNotNull  Like  NotLike  StartingWith  EndingWith  Containing
In  NotIn  True  False  After  Before  IgnoreCase
OrderBy<Property>Asc / Desc
Property_NestedProperty      <- underscore forces association traversal
```

### `@Query`

```java
@Query("select o from Order o where o.customerId = :id")           // JPQL: entity + field names
@Query(value = "select * from orders where customer_id = :id",     // native: table + column names
       nativeQuery = true)
@Query(value = "...", countQuery = "...")                          // native + Page needs this
@Modifying(clearAutomatically = true, flushAutomatically = true)   // for update/delete
@Param("id")                                                       // bind a named parameter
```

### Projections

| Kind | Declaration | Fetches |
|---|---|---|
| Closed interface | `interface S { String getSku(); }` | only named columns |
| Open interface | `@Value("#{target.a + target.b}")` | **the whole entity** |
| Record / DTO | `select new com.x.S(p.sku, p.name)` | only named columns |
| Dynamic | `<T> List<T> findByActiveTrue(Class<T> type)` | depends on the type passed |

### Paging

```java
PageRequest.of(0, 20, Sort.by("placedAt").descending())
Page<T>   // 2 queries: content + count
Slice<T>  // 1 query: fetches size + 1
Limit.of(20)   // Spring Data 3.2+
```

### Logging

```properties
logging.level.org.hibernate.SQL=DEBUG          # the statements
logging.level.org.hibernate.orm.jdbc.bind=TRACE # the parameters (Hibernate 6+; CONFIRM on your version)
# spring.jpa.show-sql=true                      # System.out only. Do not use.
spring.jpa.hibernate.ddl-auto=validate          # everywhere, including local
```

### Checklist

- [ ] Every single-result method returns `Optional<T>`.
- [ ] Every `List<T>` return is bounded by `Pageable`, `Limit` or `Top`.
- [ ] Read endpoints return projections, never entities.
- [ ] `Slice` unless the UI shows a total.
- [ ] Derived names stay under about three predicates; past that, `@Query`.
- [ ] Native queries carry a comment explaining why JPQL was insufficient.
- [ ] Every `@Modifying` has `clearAutomatically` and `flushAutomatically`.
- [ ] `ddl-auto=validate`; the schema comes from Flyway.
- [ ] `@Transactional` is on the service, not the repository.
- [ ] Query counts are asserted in tests, not eyeballed in logs.

---

## When would I use this at work?

**1. Standing up a new service.**
The repository layer is the first thing you write after the entities, and getting the
projection habit right on day one costs nothing. Retrofitting it after twelve
endpoints return entities is a multi-sprint project that touches every client.

**2. Diagnosing a slow endpoint.**
The first three questions are always: how many statements did this request issue, is
there a `COUNT` you did not ask for, and is the `SELECT` fetching columns nobody
reads. All three are Spring Data questions, and all three are answered in ten
minutes with SQL logging plus one `EXPLAIN`.

**3. Reviewing someone else's PR.**
`List<Order> findByCustomerId(Long id)` with no `Pageable` is a production incident
waiting for the first customer with 50,000 orders. Catching it in review costs one
comment. Catching it in production costs an incident, a hotfix and an apology to a
client team.

---

## Connected topics

**Prerequisites:**
- **26 — Optional**: the correct return type for a single-result query.
- **27 — Records**: the right shape for a projection — and the reason a record can
  never be an `@Entity`.
- **30 — Text blocks**: multi-line JPQL without concatenation.
- **35 / 36 — `ApplicationContext` and component scanning**: how repository
  interfaces are discovered at startup.
- **39 — Injection styles**: repositories are constructor-injected like anything else.
- **40 — Proxying**: your repository *is* a JDK dynamic proxy. This is the same
  mechanism, applied to an interface with no implementation at all.
- **42 — Auto-configuration**: `JpaRepositoriesAutoConfiguration` is what turns the
  scanning on without you writing `@EnableJpaRepositories`.
- **44 — REST controllers**: where the projection lands.
- **46 — `ProblemDetail`**: `Optional.empty()` from a repository becomes a 404
  problem body.

**This unlocks:**
- **48 — Persistence context**: what "managed" means for the entity a repository just
  returned, and why `save()` is a no-op for it. The first-level cache from Trap 4,
  explained properly.
- **49 — Associations and lazy loading**: why Trap 1 is not a Jackson problem.
- **50 — N+1**: how to count queries as an assertion rather than by grepping.
- **51 — Caching**: the second-level cache sits behind these same repository calls.
- **52 — Locking**: `@Version` and `@Lock` on repository methods; the
  `tryDecrement` native query compared against the alternatives with real numbers.
- **53 — Batching**: why seeding 5,000,000 order lines through `saveAll` is slow, and
  what `GenerationType.IDENTITY` does to it.
- **54 / 55 — `@Transactional`**: `readOnly = true`, transaction boundaries, and why
  a repository call outside a transaction gets its own implicit one.
- **61 — Testcontainers**: repository tests against real Postgres, because H2 will
  happily accept SQL that Postgres rejects.
- **109 — HikariCP**: every repository call takes a connection from the pool for the
  duration of its transaction.

---

*Java baseline 21, running on JDK 25, Spring Boot 4.1 / Framework 7.0, Spring Data
2025.1, `jakarta.persistence.*` throughout. Do not memorise Spring Data or Hibernate
version numbers — the Boot BOM decides them, and `./mvnw dependency:list` is the only
answer you should trust for your build. The one version fact worth carrying: the
Hibernate bind-parameter logging category changed between Hibernate 5 and 6, so
confirm which one your build uses before concluding that parameter logging is
broken.*
