# 48 — Hibernate I: Persistence Context, Entity States, Dirty Checking, Flush Modes

## Phase: 5 — Spring Boot & Persistence
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: the `orderflow` entity model — `Order`, `OrderLine`, `Product`, `Inventory`, `Wallet`, `Payment` — and the transactional order-placement path that writes without a single `save()` call

---

## Mechanical statement

> **The persistence context is a first-level cache plus a snapshot of every loaded
> entity's field values.**
>
> **At flush, Hibernate walks every managed entity, compares its current field values
> against that snapshot, and emits an `UPDATE` for whatever differs.**
>
> **You never call `save()` for an already-managed entity. The diff does it.**

Read that three times. Everything in this document is a consequence of it.

---

## The bridge from what you know

This is the largest single conceptual gap in Phase 5. Be exact about it.

### Prisma is stateless. Hibernate is not.

In Prisma you write:

```ts
const order = await prisma.order.findUnique({ where: { id } });
order.status = 'PAID';
await prisma.order.update({ where: { id }, data: { status: 'PAID' } });
```

Three facts about that code:

1. `order` is a **plain object**. Prisma has forgotten it exists the moment it
   returns.
2. Mutating `order.status` in memory does **nothing** to the database.
3. The `update` call is what issues SQL. No call, no write.

Now the Java equivalent:

```java
@Transactional
public void markPaid(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.setStatus(OrderStatus.PAID);
    // no save(). no update(). no flush(). nothing.
}
```

**That method writes to the database.** An `UPDATE orders SET status=? WHERE id=?`
is issued at commit. There is no line of code you can point at that causes it.

That is not a trick or a Spring feature. It is the JPA specification working as
designed, and it is the thing your Prisma model has no slot for.

### What is actually going on

`findById` did not return you a plain object. It:

1. Ran the `SELECT`.
2. Built an `Order` instance.
3. Put it in a **map** keyed by `(Order.class, 42L)` — the first-level cache.
4. Copied every one of its field values into a separate `Object[]` — the
   **snapshot**.

At commit, Hibernate iterates that map. For each entity it compares the live field
values against the snapshot. `status` differs. So it emits an `UPDATE`.

`order.setStatus(...)` was not a database call. It was a plain Java setter. The
database call happened because a *diff* found a difference later.

### The closest analogue you do have

**TypeORM's `EntityManager` with change tracking.** If you have used:

```ts
const order = await manager.findOne(Order, { where: { id } });
order.status = 'PAID';
await manager.save(order);
```

...then TypeORM did compare the loaded entity against a remembered state to decide
which columns to include in the `UPDATE`. That is genuinely the same idea. Say so in
an interview if TypeORM is on your CV — it shows you know the difference between the
two ORMs you have used rather than treating "ORM" as one thing.

The differences that remain:

| TypeORM `EntityManager` | Hibernate persistence context |
|---|---|
| You still call `save()` | `save()` on a managed entity is a **no-op**; the write happens without it |
| Tracking is per-`save` | Tracking is continuous for the whole transaction |
| No identity guarantee | Two lookups of the same row in one context return the **same object** (`==` is true) |
| No automatic flush before queries | A query touching a dirty table forces a flush first |

### The verdict table

| You know | Java | Verdict |
|---|---|---|
| `prisma.order.update()` issues SQL | Dirty checking at flush issues SQL | **NO ANALOGUE** — the trigger is a diff, not a call |
| Prisma returns plain objects | Hibernate returns *managed* entities | **NO ANALOGUE** |
| TypeORM `EntityManager` change tracking | Persistence context | **PARTIAL** — the closest thing you have, and worth naming |
| `prisma.$transaction` | `@Transactional` | **PARTIAL** — Spring's also defines the persistence context boundary, which Prisma's does not |
| Reloading gives you a fresh object | Reloading gives you the **same** object from the identity map | **NO ANALOGUE** |

---

## What is this?

### The four states

Every entity instance, at every moment, is in exactly one of four states. Learn to
name them on sight; this vocabulary is how every Hibernate conversation is
conducted.

```
        new Order()
             |
             v
       [ TRANSIENT ]  -- not in any persistence context, no database row
             |
             |  em.persist(order)
             v
       [ MANAGED ]    -- in the persistence context, tracked, snapshot taken
          ^  |  ^
          |  |  |
  em.merge|  |  |  em.find / query result
          |  |  |
          |  |  +--- em.detach(order), em.clear(), em.close(),
          |  |          or the transaction ends
          |  v
       [ DETACHED ]   -- has an identity, has a row, but is NOT tracked
             |
             |  em.remove(order)   (from MANAGED)
             v
       [ REMOVED ]    -- scheduled for DELETE at the next flush
```

| State | In the context? | Has an ID? | Row exists? | What happens at flush |
|---|---|---|---|---|
| **Transient** | no | usually not | no | nothing — Hibernate does not know it exists |
| **Managed** | yes | yes | yes (or will after this flush) | dirty-checked; `INSERT` or `UPDATE` as needed |
| **Detached** | no | yes | yes | **nothing.** Mutations are silently discarded |
| **Removed** | yes | yes | yes, until flush | `DELETE` |

**The two states people get wrong are managed and detached, and they get them wrong
in opposite directions:** they are surprised a managed entity was written, and
surprised a detached one was not.

### The operations

| Operation | From | To | What it does |
|---|---|---|---|
| `persist(e)` | transient | managed | Schedules an `INSERT`. May assign the ID immediately (IDENTITY) or from a sequence. Does **not** necessarily hit the database yet. |
| `find(Class, id)` | — | managed | Checks the identity map first. Only queries the database on a miss. |
| `merge(e)` | detached | **a different managed instance** | Copies the detached entity's state onto a managed copy and **returns that copy**. The instance you passed in stays detached. This is the single most misused JPA method. |
| `remove(e)` | managed | removed | Schedules a `DELETE`. |
| `detach(e)` | managed | detached | Evicts one entity. Pending changes to it are discarded. |
| `clear()` | managed | detached | Evicts **everything**. |
| `flush()` | — | — | Runs dirty checking now and pushes SQL to the database. Does **not** commit. |
| `close()` | managed | detached | Ends the context. |

**`merge` deserves a second look**, because its return value is the whole point:

```java
Order detached = ...;                    // came from somewhere outside the transaction
detached.setStatus(OrderStatus.PAID);

em.merge(detached);                      // WRONG-ish: return value ignored
detached.setTotalMinor(9999);            // this change is LOST - detached is still detached

Order managed = em.merge(detached);      // right
managed.setTotalMinor(9999);             // this one is dirty-checked
```

### Where Spring puts the boundary

In a Spring application you almost never touch `EntityManager` directly. The
boundary is set by `@Transactional`:

```
@Transactional method entered
   -> transaction manager opens a transaction
   -> an EntityManager (= a Hibernate Session = a persistence context) is created
      and bound to the current thread
   -> your repository calls all share it
   -> method returns normally
   -> flush()   <- dirty checking runs HERE
   -> commit()
   -> EntityManager closed; every entity it held becomes DETACHED
```

**The persistence context is transaction-scoped.** One `@Transactional` method call
= one persistence context. Two service calls = two contexts, and an entity loaded in
the first is detached by the time the second runs.

The `EntityManager` you inject with `@PersistenceContext` is itself a proxy that
looks up the thread-bound real one — the same mechanism as Topic 38's scoped
proxies. This is why a `@Transactional` method that is not actually proxied (Topic
40's self-invocation trap) also has no persistence context of the kind you expect.

### Flush modes

**Flush** means: run dirty checking, and send the resulting SQL to the database.
It is not the same as commit. A flush inside a transaction can still be rolled back.

| Mode | When Hibernate flushes |
|---|---|
| `AUTO` (JPA default) | Before commit, **and** before executing a query whose tables overlap the pending changes |
| `COMMIT` | Only at commit |
| `MANUAL` (Hibernate-specific) | Only when you call `flush()` yourself |

Spring sets **`MANUAL`** when you write `@Transactional(readOnly = true)`. That is
the concrete thing `readOnly` does at the Hibernate level: it turns dirty checking
off for that transaction. It is not merely a hint to the driver.

That `AUTO` behaviour — flushing before a query — is the source of most "why did
that write happen *there*?" confusion, and it is covered in Machine-level reality
below.

### `[BOOT 3.x DELTA]`

- Persistence-context semantics are JPA specification behaviour and are **identical**
  on Boot 3.x, 4.x, and going back to Boot 1.x. Nothing here is version-specific.
- What *does* differ by version is **logging category names and metrics
  availability**:
  - Hibernate 5: bind parameters logged under
    `org.hibernate.type.descriptor.sql.BasicBinder`.
  - Hibernate 6+: bind parameters logged under `org.hibernate.orm.jdbc.bind`.
  - Boot's auto-configured `hibernate.*` Micrometer metrics were deprecated during
    the 3.x line in favour of Hibernate's own Micrometer integration, and their
    availability differs by version. **I am not going to assert which applies to
    your build.** Confirm both with the commands in Measurement below rather than
    trusting any document, including this one.
- `spring.jpa.open-in-view` has defaulted to `true` since Boot 1.x and still does in
  Boot 4. Boot logs a warning at startup about it. Topic 49 is where that warning
  gets acted on.

---

## Why does it matter?

**1. It is the difference between "I use an ORM" and "I know what my ORM will do".**

Every production Hibernate bug worth the name is a persistence-context bug: a write
that happened when nobody meant it to, a write that did not happen when everyone
assumed it would, a query that ran at a surprising moment, or a context holding
100,000 entities. None of these produce a stack trace pointing at the cause.

**2. The failures are silent by construction.**

A detached mutation does not throw. It does not log. It returns normally and the
data is simply not there afterwards. You find it from a business number — a status
that never advances, a balance that never changes — days later.

**3. The memory model is not what you think.**

Every managed entity costs its own fields **plus** a snapshot array of the same
values. Loading 100,000 entities to change one field on each is roughly twice the
memory you estimated, plus a dirty check over 100,000 × (field count) comparisons at
every flush. This is a design error with a specific, explainable cost, not a vague
"that seems inefficient".

**4. It is the staple senior Java interview question.**

"You loaded an entity, changed a field, never called save, and the row changed —
explain." If you can walk that from `SELECT` through snapshot through diff through
`ActionQueue` through `UPDATE`, you are demonstrably past the mid-level line. If you
say "Spring does it automatically", you are not.

---

## Machine-level reality

This section is about what is actually in memory and what actually executes. It is
the difference between using JPA and being able to predict JPA.

### 1. The snapshot doubles the memory per loaded entity

When Hibernate loads an entity it creates an `EntityEntry` in the persistence
context. Among other things, that entry holds a `loadedState` — an `Object[]` with
one slot per persistent property, containing the values as they were when the entity
was read.

So a loaded `OrderLine` costs you:

- the `OrderLine` object itself (header + fields, Topic 69),
- an `Object[]` of its property values (each primitive **boxed** into that array —
  Topic 01's cost, per field, per entity),
- an `EntityEntry` and its bookkeeping,
- an entry in the identity map,
- an entry in the entity-key map.

**Rule of thumb: budget roughly 2× the naive object size, and more for
primitive-heavy entities because the snapshot boxes them.** Do not quote a precise
multiplier; measure it with JOL and a heap dump (Topics 69 and 79) when it matters.

**The consequence is a design rule.** This:

```java
@Transactional
public void deactivateAllDiscontinued() {
    List<Product> all = productRepository.findByDiscontinuedTrue();  // 100,000 rows
    all.forEach(p -> p.setActive(false));
}
```

loads 100,000 entities, allocates 100,000 snapshots, does 100,000 × (field count)
comparisons at flush, and emits 100,000 individual `UPDATE` statements. It is a
design error, and the reason is now precise rather than aesthetic.

The fix is a bulk update that never loads anything (Topic 47's `@Modifying`), with
its consequences understood:

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("update Product p set p.active = false where p.discontinued = true")
int deactivateAllDiscontinued();
```

One statement. Zero snapshots. And an empty persistence context afterwards, which is
a cost you accept knowingly.

### 2. Dirty checking is O(entities × properties), at every flush

Hibernate does not know which entity you touched. There is no interception on a
plain setter — the entity is your class, and `setStatus` is your method. So at flush
it must check **all** of them.

For each managed entity, for each persistent property, it compares the current value
against the snapshot slot using the property type's comparison logic.

Two consequences:

- A persistence context with 100,000 entities pays a full scan **at every flush**,
  and under `FlushMode.AUTO` a flush can happen before every query.
- Clearing the context periodically during a large batch is not a nicety. It is what
  keeps the dirty check bounded. Topic 53 makes this a concrete pattern.

> **Bytecode enhancement** can change this: Hibernate can instrument your entity
> classes at build time so setters record dirtiness directly, making the flush check
> O(dirty entities) instead. It is off by default, it has real caveats, and it is not
> something to enable casually. Know that it exists; do not reach for it as a first
> response.

### 3. Flush ordering is Hibernate's, not yours

This surprises everyone once.

Hibernate does not execute statements in the order you called the methods. It
collects **actions** in an `ActionQueue` and executes them, at flush, in a fixed
order by action type:

```
1. orphan removal for collections
2. entity INSERTs
3. entity UPDATEs
4. collection removals
5. collection updates
6. collection recreations
7. entity DELETEs        <- last
```

So this code:

```java
@Transactional
public void replaceLine(Order order, OrderLine oldLine, OrderLine newLine) {
    order.getLines().remove(oldLine);   // you think: DELETE
    order.getLines().add(newLine);      // you think: INSERT
}
```

executes the `INSERT` **before** the `DELETE`. If there is a unique constraint on
`(order_id, product_id)`, the insert violates it:

```
org.hibernate.exception.ConstraintViolationException: could not execute statement
  [ERROR: duplicate key value violates unique constraint "uq_order_line_order_product"]
```

*(That is the shape of the message. Your text will differ.)*

The code reads correctly. The SQL log shows the statements in the "wrong" order. The
fix is an explicit `em.flush()` between the remove and the add — which forces the
delete out before the insert is queued — or a design that does not require the
swap.

**This is the number one reason to read the SQL log in *order*, not just to count
lines.**

### 4. `FlushMode.AUTO` flushes before a query that touches a dirty table

Under the JPA default, Hibernate tries to keep queries consistent with your
in-memory changes. Before running a query, it checks whether any pending change
touches a table the query reads. If so, it flushes first.

```java
@Transactional
public void placeOrder(...) {
    Order order = new Order(...);
    orderRepository.save(order);                          // INSERT queued (or executed)

    // This query reads the "orders" table, which has a pending change.
    // Hibernate flushes FIRST, then runs the query.
    long todayCount = orderRepository.countByCustomerIdAndPlacedAtAfter(customerId, startOfDay);

    if (todayCount > MAX_ORDERS_PER_DAY) {
        throw new TooManyOrdersException();               // rolls back - but the INSERT already ran
    }
}
```

The `INSERT` executes before the count query. It is rolled back when the exception
propagates, so the data is correct — but the SQL log shows a write you did not
expect at a point you did not expect, and any database trigger, constraint or
`LISTEN/NOTIFY` fires at that moment.

**This is where "surprise write ordering" comes from**, and it is why the SQL log
sometimes shows an `INSERT` before a `SELECT` you wrote first.

### 5. The identity map guarantees `==` for the same row

Within one persistence context:

```java
Order a = orderRepository.findById(42L).orElseThrow();
Order b = orderRepository.findById(42L).orElseThrow();

a == b;          // TRUE. One SELECT was issued, not two.
a.equals(b);     // true, obviously
```

The second `findById` hit the identity map and never touched the database. This is
the *first-level cache*, and it is not optional and cannot be disabled.

Across two persistence contexts (two `@Transactional` methods), `a == b` is
**false**, and two `SELECT`s were issued.

Two consequences worth carrying:

- Inside a transaction, `==` on entities works. Outside, it does not. So **your code
  must never depend on `==` for entities**, because the same code is correct in a
  test that runs inside one transaction and wrong in production where it does not.
- This is exactly the trap you already met in Topic 01 with the `Integer` cache:
  identity comparison that accidentally works in the small case. Same shape, bigger
  blast radius.

### 6. `equals`/`hashCode` on an entity — the Topic 13 contract, made harder

Topic 13 taught you: equal objects must have equal hash codes, and a key's hash must
not change while it is in a hash-based collection.

A JPA entity breaks this by default, because a generated ID is **null before
persist**:

```java
Order order = new Order(...);            // id == null
Set<Order> batch = new HashSet<>();
batch.add(order);                        // hashed with id == null

orderRepository.save(order);             // id is now 90210 - the hash CHANGED

batch.contains(order);                   // FALSE. The object is in the set and unfindable.
batch.remove(order);                     // does nothing. size() still 1.
```

That is Topic 13's drill, reproduced by JPA rather than by a deliberate mutation.
And it is worse here, because `@OneToMany` collections are frequently `Set`s and
Hibernate itself puts your entities in them.

**The rules that actually work:**

1. **Best:** use a business key that is assigned at construction and never changes —
   `Product.sku`, `Order.idempotencyKey`. `equals` and `hashCode` use only that.
2. **If there is no business key:** assign a `UUID` in the constructor, before
   persist, and use it. You control the ID, so it is never null and never changes.
3. **If you must use the generated ID:** `hashCode()` must return a **constant**
   (e.g. `getClass().hashCode()`), and `equals` compares IDs with a null-safe check
   that returns false when either ID is null. This is correct but degrades every
   `HashSet` of entities to a linked list. Accept that trade knowingly.
4. **Never** use `getClass() != o.getClass()` in `equals`. Hibernate hands you
   *proxy subclasses*, so a proxy of `Product` is not `Product.class` and your
   `equals` returns false against the real thing. Use `instanceof` — and Topic 49
   explains exactly why proxies exist.

Option 1 is right roughly 80% of the time in a domain like `orderflow`, because real
business entities have real business keys.

---

## Example 1 — minimal

The whole topic in fifteen lines.

```java
package com.orderflow.orders;

@Service
public class OrderStatusService {

    private final OrderRepository orders;

    OrderStatusService(OrderRepository orders) {
        this.orders = orders;
    }

    @Transactional
    public void markPaid(Long orderId) {
        Order order = orders.findById(orderId)          // 1. SELECT. Entity is now MANAGED.
                            .orElseThrow(() -> new OrderNotFoundException(orderId));

        order.setStatus(OrderStatus.PAID);              // 2. A plain Java setter. No SQL.

        // 3. Method returns. Spring commits.
        //    Commit triggers flush. Flush diffs `order` against its snapshot.
        //    `status` differs -> UPDATE orders set status=? where id=?
    }
}
```

There is no `save`. There is no `update`. The row changes.

Now the same method with one line added:

```java
    @Transactional
    public void markPaidButDont(Long orderId) {
        Order order = orders.findById(orderId).orElseThrow();

        entityManager.detach(order);                    // <- now DETACHED

        order.setStatus(OrderStatus.PAID);              // still a plain setter

        // Commit. Flush runs. `order` is not in the context, so it is not visited.
        // NO SQL IS ISSUED. The row is unchanged. Nothing throws.
    }
```

Same code, one extra line, opposite outcome, no error either way. **That is the
failure drill below, and it is why this topic is a DIFFERENTIATOR.**

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` at the Topic 65 baseline:

- 100,000 products, 1,000,000 orders, 5,000,000 order lines in Postgres.
- ~60 order placements per second sustained; 180/s at promotion peak.
- **SLO:** p99 under 600 ms for `POST /api/orders`.
- HikariCP pool of 20 against an 8-core Postgres. A transaction holds a connection
  for its entire lifetime (Topic 55), so at 60 rps a 300 ms transaction consumes 18
  connection-seconds per second — that is the whole pool.

Every millisecond a transaction stays open is a millisecond of connection held. The
persistence context lives exactly as long as the transaction. Those two sentences
are the same sentence.

### The entity model

These are the entities established in Topic 47, restated with the annotations that
matter for *this* topic. They do not change again in Topic 49.

```java
package com.orderflow.orders;

import jakarta.persistence.*;

@Entity
@Table(name = "orders")                    // "order" is a SQL reserved word
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
    @SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50)
    private Long id;

    /**
     * Business key, assigned at construction, never changes.
     * This is what equals/hashCode use. See Machine-level reality point 6.
     */
    @Column(name = "idempotency_key", nullable = false, unique = true, updatable = false, length = 64)
    private String idempotencyKey;

    @Column(name = "customer_id", nullable = false)
    private Long customerId;

    @Enumerated(EnumType.STRING)           // STRING, never ORDINAL: ordinals break on reorder
    @Column(nullable = false, length = 20)
    private OrderStatus status;

    @Column(name = "placed_at", nullable = false)
    private Instant placedAt;

    @Column(name = "total_minor", nullable = false)
    private long totalMinor;               // pence. Topic 01.

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();

    protected Order() { }                  // JPA needs this. Not public.

    public Order(String idempotencyKey, Long customerId, Instant placedAt) {
        this.idempotencyKey = idempotencyKey;
        this.customerId = customerId;
        this.placedAt = placedAt;
        this.status = OrderStatus.PENDING;
        this.totalMinor = 0L;
    }

    /** Keeps both sides of the association consistent. Always write these in pairs. */
    public void addLine(OrderLine line) {
        lines.add(line);
        line.setOrder(this);
        this.totalMinor += line.getUnitPriceMinor() * line.getQuantity();
    }

    public void markPaid()      { this.status = OrderStatus.PAID; }
    public void markCancelled() { this.status = OrderStatus.CANCELLED; }

    @Override
    public boolean equals(Object o) {
        // instanceof, NOT getClass() - Hibernate hands out proxy subclasses (Topic 49)
        if (this == o) return true;
        if (!(o instanceof Order other)) return false;
        return idempotencyKey.equals(other.idempotencyKey);
    }

    @Override
    public int hashCode() {
        return idempotencyKey.hashCode();   // stable from construction. Topic 13 satisfied.
    }

    // getters; setters only where mutation is a legitimate domain operation
}
```

```java
@Entity
@Table(name = "order_line",
       uniqueConstraints = @UniqueConstraint(name = "uq_order_line_order_product",
                                             columnNames = { "order_id", "product_id" }))
public class OrderLine {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_line_seq")
    @SequenceGenerator(name = "order_line_seq", sequenceName = "order_line_seq", allocationSize = 50)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)     // LAZY is NOT the default. Topic 49.
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    @Column(nullable = false)
    private int quantity;

    @Column(name = "unit_price_minor", nullable = false)
    private long unitPriceMinor;

    protected OrderLine() { }
    // ...
}
```

`Product` (`sku` as the business key), `Inventory` (`productId`,
`quantityAvailable`, `quantityReserved`, and a `@Version` arriving in Topic 52),
`Wallet` (`customerId`, `balanceMinor`, `@Version`) and `Payment` (`order`,
`status`, `amountMinor`, `externalRef`) follow the same conventions:
`GenerationType.SEQUENCE` with a pooled allocation size, `EnumType.STRING`, money as
`long` minor units, a business key for `equals`/`hashCode`, and every `@ManyToOne`
explicitly `LAZY`.

> **`GenerationType.SEQUENCE`, not `IDENTITY`, and `allocationSize = 50` on purpose.**
> `IDENTITY` forces Hibernate to execute each `INSERT` immediately to obtain the
> generated key, which disables JDBC batching entirely and also means an `INSERT`
> fires the moment you call `persist` rather than at flush. That is Topic 53, and it
> is a choice made here so you do not have to migrate later.

### The order-placement path

```java
package com.orderflow.orders;

@Service
public class OrderPlacementService {

    private final OrderRepository orders;
    private final ProductRepository products;
    private final InventoryRepository inventory;
    private final WalletRepository wallets;
    private final PaymentRepository payments;

    // constructor injection - Topic 39

    @Transactional                          // <- the persistence context boundary
    public Long placeOrder(PlaceOrderCommand command) {

        // ---- 1. Idempotency. One query, no entity loaded on the happy path.
        Optional<Order> existing = orders.findByIdempotencyKey(command.idempotencyKey());
        if (existing.isPresent()) {
            return existing.get().getId();
        }

        // ---- 2. Price the lines. ONE query for all SKUs, not one per line.
        List<Product> found = products.findBySkuIn(command.skus());
        Map<String, Product> bySku = found.stream()
                .collect(Collectors.toMap(Product::getSku, Function.identity()));

        for (String sku : command.skus()) {
            if (!bySku.containsKey(sku)) {
                throw new ProductNotFoundException(sku);   // -> Topic 46's 404
            }
        }

        // ---- 3. Build the order. TRANSIENT until persist.
        Order order = new Order(command.idempotencyKey(), command.customerId(), Instant.now());
        for (OrderLineCommand line : command.lines()) {
            Product product = bySku.get(line.sku());
            order.addLine(new OrderLine(product, line.quantity(), product.getPriceMinor()));
        }

        orders.save(order);                 // TRANSIENT -> MANAGED. This save() is REQUIRED:
                                            // a transient entity is invisible to dirty checking.
                                            // Cascade ALL makes the lines managed too.

        // ---- 4. Reserve inventory with an atomic conditional UPDATE.
        //         Deliberately does NOT load Inventory entities: no snapshots, no
        //         read-modify-write race. Topic 52 compares this to @Version.
        for (OrderLineCommand line : command.lines()) {
            Long productId = bySku.get(line.sku()).getId();
            int updated = inventory.tryDecrement(productId, line.quantity());
            if (updated == 0) {
                throw new InsufficientStockException(line.sku(), line.quantity(), 0);
            }
        }

        // ---- 5. Debit the wallet. THIS is dirty checking in production.
        Wallet wallet = wallets.findByCustomerId(command.customerId())
                               .orElseThrow(() -> new WalletNotFoundException(command.customerId()));

        if (wallet.getBalanceMinor() < order.getTotalMinor()) {
            throw new InsufficientFundsException(order.getTotalMinor(), wallet.getBalanceMinor(), "GBP");
        }

        wallet.debit(order.getTotalMinor());   // a plain setter inside the domain object.
                                               // NO wallets.save(wallet). None is needed.
                                               // The UPDATE happens at flush, because
                                               // `wallet` is MANAGED and balanceMinor
                                               // now differs from its snapshot.

        // ---- 6. Record the payment.
        Payment payment = new Payment(order, PaymentStatus.CAPTURED, order.getTotalMinor());
        payments.save(payment);                // transient -> managed, so save() IS needed

        order.markPaid();                      // managed. Again: no save().

        return order.getId();

        // ---- Method returns. Spring commits:
        //        flush -> dirty check over Order, its OrderLines, Wallet, Payment
        //              -> ActionQueue: INSERTs (order, lines, payment),
        //                              then UPDATEs (wallet.balance, order.status)
        //              -> commit
        //              -> persistence context closed; every entity above is now DETACHED
    }
}
```

### Read this method for the two rules

**`save()` is called exactly where an entity is transient** — the new `Order` and the
new `Payment`. Nowhere else.

**`save()` is called nowhere for entities that were loaded** — `wallet` was loaded,
mutated, and written. `order` was already managed after step 3, and `markPaid()`
needed no second `save`.

If a reviewer adds `wallets.save(wallet)` to step 5 "for clarity", it is harmless
(Spring Data's `save` on a managed entity calls `merge`, which returns the same
instance and changes nothing) — but it is a lie about how the code works, and the
next person will believe it and be confused when a different mutation writes without
one. Reject it in review with a comment.

### The thing that will bite you next

Everything returned from this method is **detached** by the time the controller
serializes a response. That is Topic 49, and it is why the controller must map to a
DTO *inside* the transaction, not outside it.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — mutating a DETACHED entity and being surprised nothing was written

**Wrong:**

```java
// no @Transactional here - this is a plain public method on a @Service
public Order loadOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}

@Transactional
public void applyDiscount(Order order, long discountMinor) {
    order.setTotalMinor(order.getTotalMinor() - discountMinor);   // detached!
}
```

Called as `applyDiscount(loadOrder(42L), 500)`.

**Exact symptom:** the method returns normally. No exception, no warning, no log
line. The SQL log for the second transaction contains **no `UPDATE` at all** — that
is the observable evidence. The row still has the old total. A test that reloads the
order inside the *same* transaction may even pass, which is what lets this ship.

**Root cause:** `loadOrder` had no transaction, so the repository call got its own
short implicit one, which committed and closed immediately. The returned `Order` is
**detached**. `applyDiscount` opened a *new* persistence context that has never seen
this instance. At flush, Hibernate iterates its own managed entities; `order` is not
among them.

**Fix:** do not pass entities across transaction boundaries. Pass identifiers.

```java
@Transactional
public void applyDiscount(Long orderId, long discountMinor) {
    Order order = orderRepository.findById(orderId).orElseThrow();  // MANAGED here
    order.setTotalMinor(order.getTotalMinor() - discountMinor);
}
```

If you genuinely have a detached entity — from an HTTP request body, a cache, a
queue — use `merge` **and use its return value**:

```java
Order managed = entityManager.merge(detached);
managed.setTotalMinor(...);
```

**The review rule:** an entity in a method signature that crosses a transaction
boundary is a defect. Take an ID.

---

### Trap 2 — mutating a MANAGED entity and being surprised it WAS written

The mirror image, and the more dangerous of the two because it corrupts data rather
than failing to change it.

**Wrong:**

```java
@Transactional(readOnly = false)     // or just @Transactional
public OrderQuote quoteWithPromotion(Long orderId, String promoCode) {
    Order order = orderRepository.findById(orderId).orElseThrow();

    // "just calculating a preview, not saving anything"
    long discount = promotionEngine.discountFor(order, promoCode);
    order.setTotalMinor(order.getTotalMinor() - discount);        // <- writes to the DB

    return new OrderQuote(order.getId(), order.getTotalMinor(), discount);
}
```

**Exact symptom:** customers who *view* a promotional quote get the discount
permanently applied to their order, whether or not they accept it. The SQL log shows
`update orders set total_minor=? where id=?` on a code path that is named `quote`.
Finance notices the revenue gap before engineering notices the bug.

**Root cause:** there is no such thing as a scratch copy of a managed entity. The
setter is a database write with a delay. "I'm only calculating" is not a state a
managed entity can be in.

**Fix — three options, in order of preference:**

1. **Do not load the entity.** Use a projection (Topic 47). A `record OrderTotals`
   is not managed and cannot be dirty-checked.
2. **`@Transactional(readOnly = true)`.** This sets Hibernate's flush mode to
   `MANUAL`, so dirty checking never runs and no `UPDATE` is emitted. It is a
   genuine safety net on every read path, and it costs nothing.
3. **Detach explicitly** before mutating, if you really must work on a copy.

Option 2 is the one to institutionalise: **every read-only service method gets
`@Transactional(readOnly = true)`.** It converts this entire class of bug into a
no-op. Make it a review checklist item.

---

### Trap 3 — loading 100,000 entities to change one field

**Wrong:**

```java
@Transactional
public void expireStalePendingOrders(Instant cutoff) {
    List<Order> stale = orderRepository.findByStatusAndPlacedAtBefore(OrderStatus.PENDING, cutoff);
    stale.forEach(Order::markCancelled);    // 120,000 orders on a bad morning
}
```

**Exact symptom:** a batch job whose duration grows superlinearly with the row count.
The SQL log shows one `SELECT` and then 120,000 individual `UPDATE` statements. Heap
usage climbs steadily and stays high for the whole transaction; on a constrained
container you get `OutOfMemoryError: Java heap space`. The transaction holds one
HikariCP connection for its entire multi-minute life, so under any concurrent load
the pool starves and *unrelated* endpoints start timing out (Topic 109).

**Root cause:** three costs compounding.

1. 120,000 entities, each with a snapshot — roughly double the naive memory.
2. Dirty checking scans all 120,000 × (field count) at flush.
3. 120,000 round trips, because updates to *different* rows with *different* changed
   columns do not batch well even with batching configured (Topic 53).

**Fix:** a bulk update that loads nothing.

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("""
       update Order o set o.status = com.orderflow.orders.OrderStatus.CANCELLED
       where o.status = com.orderflow.orders.OrderStatus.PENDING and o.placedAt < :cutoff
       """)
int expireStalePendingOrders(@Param("cutoff") Instant cutoff);
```

One statement. No entities. Then be explicit about what you gave up: no dirty
checking means no `@PreUpdate` callbacks, no optimistic-lock version bump, and no
domain events. If those matter, chunk the work instead — process 500 at a time, each
in its own transaction, calling `clear()` between chunks. Topic 53 is the full
treatment.

---

### Trap 4 — `equals`/`hashCode` using a generated ID

**Wrong:**

```java
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;   // two defects on one line
    return Objects.equals(id, ((Order) o).id);
}
@Override public int hashCode() { return Objects.hash(id); }     // null before persist
```

**Exact symptom — two of them, and they look unrelated:**

1. An `Order` added to a `HashSet` before `save()` becomes unfindable after `save()`.
   `set.contains(order)` returns `false` while `set.size()` still counts it — Topic
   13's exact failure, reproduced. With `@OneToMany` on a `Set`, orphan removal then
   deletes rows you did not mean to delete, or fails to delete rows you did.
2. `order.equals(orderProxy)` returns `false` even though both refer to row 42,
   because `getClass()` on a Hibernate proxy returns something like
   `com.orderflow.orders.Order$HibernateProxy$aB3xK9` — not `Order.class`.

**Root cause:** the generated ID is `null` at construction and non-null after
persist, so the hash code mutates while the object is in a hash-based collection —
a direct violation of the Topic 13 contract. Separately, `getClass()` is unsafe on
proxied entities.

**Fix:** the business key.

```java
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Order other)) return false;    // instanceof, so proxies match
    return idempotencyKey.equals(other.idempotencyKey);
}
@Override public int hashCode() { return idempotencyKey.hashCode(); }
```

If there is genuinely no business key, assign a `UUID` in the constructor and use
that. Falling back to a constant `hashCode` plus a null-safe ID `equals` is correct
but degrades every entity `HashSet` to linear scan — take that trade knowingly, not
by accident.

---

### Trap 5 — relying on `==` because it worked in the test

**Wrong:**

```java
@Transactional
public boolean isSameOrder(Long id, Order candidate) {
    return orderRepository.findById(id).orElseThrow() == candidate;   // identity comparison
}
```

**Exact symptom:** the unit test — which runs entirely inside one `@Transactional`
test method — passes. In production, where the candidate came from a previous
request, it returns `false` every time. The feature that depends on it silently never
triggers. No exception, no log.

**Root cause:** the identity map guarantees `==` **within one persistence context**.
Across contexts, two loads of row 42 produce two distinct objects. The test's single
transaction hid the difference. This is structurally identical to the `Integer` cache
in Topic 01: identity comparison that accidentally works in the small case.

**Fix:** never `==` on entities. Use `equals` with a business key (Trap 4), or
compare IDs. And write persistence tests where the write and the read are in
**different** transactions — a test that does everything in one transaction is
testing a scenario production never runs. Topic 61's Testcontainers setup is where
this becomes routine.

---

## Hands-on proof

Everything below is a command **you** run. I have no JVM, no Spring app and no
Postgres. Nothing here is captured output. What follows is exactly what to run and
how to read every result.

### Setup

```properties
# src/main/resources/application-local.properties

logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE
spring.jpa.properties.hibernate.generate_statistics=true
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.open-in-view=false
```

```bash
# confirm your Hibernate major version - the bind logging category depends on it
./mvnw -q dependency:list | grep -i hibernate-core
```

**On the bind-parameter category, `[BOOT 3.x DELTA]`:** Hibernate 5 used
`org.hibernate.type.descriptor.sql.BasicBinder`; Hibernate 6+ uses
`org.hibernate.orm.jdbc.bind`. Set the one matching your version, restart, and check
that parameter values actually appear in the log. If they do not, switch to the other
category. Two minutes of checking beats any assertion I could make here.

**Never enable bind logging in a shared or production environment.** Bound
parameters include customer IDs, wallet balances and payment references.

A log line under `org.hibernate.SQL` looks structurally like this — **this is an
illustration of the format, not captured output**:

```
DEBUG o.h.SQL : update orders set status=?,total_minor=? where id=?
TRACE o.h.orm.jdbc.bind : binding parameter (1:VARCHAR) <- [PAID]
```

### Proof 1 — `contains()` proves managed vs detached

The single most useful diagnostic in this topic.

```java
package com.orderflow.orders;

@Service
public class StateProbeService {

    @PersistenceContext
    private EntityManager em;

    private final OrderRepository orders;

    StateProbeService(OrderRepository orders) { this.orders = orders; }

    @Transactional
    public void probe(Long orderId) {
        Order order = orders.findById(orderId).orElseThrow();
        System.out.println("after findById : contains=" + em.contains(order));

        em.detach(order);
        System.out.println("after detach   : contains=" + em.contains(order));

        Order remerged = em.merge(order);
        System.out.println("after merge    : contains(original)=" + em.contains(order)
                         + " contains(returned)=" + em.contains(remerged)
                         + " same instance=" + (order == remerged));
    }
}
```

**What to look for:** the three lines.

| What you see | What it means |
|---|---|
| `true`, then `false` | Correct. `findById` returns a managed entity; `detach` evicts it. |
| `true`, then `true` | You detached a *different* instance, or you are not looking at the same `EntityManager` (check that `probe` is actually being called through the proxy — Topic 40). |
| `false` on the first line | There is no active transaction, so the repository ran in its own and closed it. Confirm `@Transactional` is on this method and that it was not called via self-invocation. |
| `contains(original)=false contains(returned)=true same instance=false` | The correct and important result: **`merge` returns a different object.** This is the single most misunderstood JPA method, proven on your own machine. |

### Proof 2 — the identity map

```java
@Transactional
public void identityProbe(Long orderId) {
    Order a = orders.findById(orderId).orElseThrow();
    Order b = orders.findById(orderId).orElseThrow();
    System.out.println("same instance = " + (a == b));
}
```

**What to look for:** the printed boolean, and the number of `select` lines in the
SQL log.

| What you see | What it means |
|---|---|
| `true`, and **one** `select` in the log | The identity map served the second call. This is the first-level cache, demonstrated. |
| `true`, and **two** `select`s | Unexpected — check whether one of them is a different query (a count, or a different entity). |
| `false` | The two calls were not in the same persistence context. Either `@Transactional` is missing, or you called through self-invocation and the proxy never applied. |

Now move the two `findById` calls into **separate** `@Transactional` methods and run
again. It should print `false` and issue two `select`s. That contrast is Trap 5,
proven.

### Proof 3 — flush ordering is not your ordering

```java
@Transactional
public void orderingProbe(Long orderId, Long productId) {
    Order order = orders.findById(orderId).orElseThrow();
    OrderLine existing = order.getLines().get(0);

    order.getLines().remove(existing);                       // you wrote DELETE first
    order.addLine(new OrderLine(productRef, 1, 999L));       // then INSERT
}
```

**What to look for:** the **order** of `insert` and `delete` lines in the SQL log.

| What you see | What it means |
|---|---|
| `insert into order_line ...` **before** `delete from order_line ...` | Expected. Hibernate's `ActionQueue` orders inserts before deletes regardless of your call order. |
| A `ConstraintViolationException` naming `uq_order_line_order_product` | The same fact, expressed as a failure. This is the production symptom. |
| `delete` before `insert` | Something forced a flush between them — an intervening query under `FlushMode.AUTO`, or an explicit `flush()`. Worth understanding *which*. |

Then add `entityManager.flush();` between the two lines and re-run. The delete should
now precede the insert. That is the fix, demonstrated.

### Proof 4 — `FlushMode.AUTO` flushes before a query

```java
@Transactional
public void autoFlushProbe(PlaceOrderCommand command) {
    Order order = new Order(command.idempotencyKey(), command.customerId(), Instant.now());
    orders.save(order);

    long count = orders.countByCustomerId(command.customerId());   // reads "orders"
    System.out.println("count = " + count);
}
```

**What to look for:** whether an `insert into orders` appears **before** the
`select count(...)`.

| What you see | What it means |
|---|---|
| `insert`, then `select count` | `FlushMode.AUTO` detected the overlap and flushed first. This is the documented default behaviour, observed. |
| `select count`, then `insert` | Either the flush mode is `COMMIT`/`MANUAL` (check for `readOnly = true`), or your ID generator is `SEQUENCE` and the insert was deferred while the count query touched no overlapping pending change — worth reading the log carefully. |
| `insert` appears immediately at `save()`, before anything else | Your ID generator is `IDENTITY`. Hibernate must execute the insert to get the key. Topic 53. |

### Proof 5 — `readOnly = true` turns dirty checking off

Take the `markPaid` method from Example 1. Change `@Transactional` to
`@Transactional(readOnly = true)`. Run it.

**What to look for:** the presence of an `update` statement.

| What you see | What it means |
|---|---|
| **No** `update` in the log, and no exception | Correct. `readOnly = true` sets Hibernate's flush mode to `MANUAL`, so dirty checking never runs. This is the safety net from Trap 2. |
| An `update` appears anyway | Something called `flush()` explicitly, or you are not on a Hibernate-backed `JpaTransactionManager`. Investigate before relying on `readOnly` as a guard. |
| An exception about a read-only transaction | Some drivers/configurations set the JDBC connection read-only and the database rejects the write. Also an acceptable outcome — it fails loudly instead of silently. |

**Do not conclude from this that `readOnly = true` is a security control.** It is a
correctness and performance measure. A native `@Modifying` query can still write.

---

## Failure drill

**This drill is mandatory.** It is the one from the master plan's failure-drill map:
*mutate a managed entity, never call `save()`, commit, observe the `UPDATE` in the
SQL log. Then `detach()` it first and observe nothing happens.*

Do it on `orderflow`, against real Postgres.

### Setup

```properties
# application-drill.properties
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE
spring.jpa.properties.hibernate.generate_statistics=true
spring.jpa.open-in-view=false
```

Seed one order with a known ID and status `PENDING`.

### The code

```java
package com.orderflow.orders;

@Service
public class DirtyCheckingDrill {

    private static final Logger log = LoggerFactory.getLogger(DirtyCheckingDrill.class);

    @PersistenceContext
    private EntityManager em;

    private final OrderRepository orders;
    private final SessionFactory sessionFactory;   // for Statistics

    DirtyCheckingDrill(OrderRepository orders, EntityManagerFactory emf) {
        this.orders = orders;
        this.sessionFactory = emf.unwrap(SessionFactory.class);
    }

    /** PART A: no save() anywhere. */
    @Transactional
    public void partA(Long orderId) {
        sessionFactory.getStatistics().clear();

        Order order = orders.findById(orderId).orElseThrow();
        log.info("PART A  managed? {}", em.contains(order));

        order.setStatus(OrderStatus.PAID);
        log.info("PART A  mutated in memory, calling NO save()");
        // method returns -> flush -> commit
    }

    /** PART B: identical, except the entity is detached first. */
    @Transactional
    public void partB(Long orderId) {
        sessionFactory.getStatistics().clear();

        Order order = orders.findById(orderId).orElseThrow();
        log.info("PART B  managed? {}", em.contains(order));

        em.detach(order);
        log.info("PART B  after detach, managed? {}", em.contains(order));

        order.setStatus(OrderStatus.PAID);
        log.info("PART B  mutated in memory, calling NO save()");
        // method returns -> flush -> commit
    }

    /** Read the counters after each part. */
    @Transactional(readOnly = true)
    public void report(String label) {
        Statistics s = sessionFactory.getStatistics();
        log.info("{} :: entityUpdateCount={} flushCount={} prepareStatementCount={}",
                 label, s.getEntityUpdateCount(), s.getFlushCount(), s.getPrepareStatementCount());
    }
}
```

### Run it

```bash
# reset the row between parts
psql -d orderflow -c "update orders set status='PENDING' where id = 1;"
# invoke partA (a test, a CommandLineRunner, or a temporary local-profile endpoint)
# then:
psql -d orderflow -c "select id, status from orders where id = 1;"
```

Repeat for `partB`, resetting the row first.

### What to capture

1. The full SQL log for each part, **in order**.
2. The `entityUpdateCount` from the statistics report after each part.
3. The row's `status` in Postgres after each part.

### How to read it

**Part A:**

| What you see | What it means |
|---|---|
| `managed? true`, an `update orders set ... where id=?` in the log, `entityUpdateCount=1`, row is `PAID` | **The expected result.** Dirty checking wrote the row with no `save()` anywhere. This is the mechanical statement, observed. |
| No `update`, row still `PENDING` | `@Transactional` is not applying. Almost always Topic 40's self-invocation trap — you called `partA` from another method in the same class. Call it from outside the bean. |
| `managed? false` | Same cause: no active transaction, so the repository ran in its own and closed it. |
| An `update` but `entityUpdateCount=0` | Statistics are off. Confirm `spring.jpa.properties.hibernate.generate_statistics=true`, and note the `spring.jpa.properties.` prefix — a common typo is omitting it. |

**Part B:**

| What you see | What it means |
|---|---|
| `managed? true` then `after detach, managed? false`, **no `update` in the log**, `entityUpdateCount=0`, row still `PENDING` | **The expected result.** Identical code, one extra line, opposite outcome, and — this is the point — **no error of any kind**. |
| An `update` appears anyway | You detached a different instance, or something re-attached it (a `merge`, or a cascade from another managed entity that references it). Trace it; this is a genuinely instructive investigation. |
| An exception on `detach` | You are calling it on a transient or already-detached instance. Check `contains` first. |

### What it proves

Three things, and you should be able to state each in one sentence.

1. **The write is caused by a diff, not by a call.** No `save()` appears anywhere in
   Part A, and the row changed.
2. **Managed and detached are the same object with different consequences.** The
   only difference between A and B is one line that mutates no data at all.
3. **The detached failure is silent.** Part B does not throw, warn or log. If this
   were a wallet debit, the customer would have received goods and kept their money,
   and nothing in your alerting would have fired.

That third point is why this is a drill and not a paragraph. You need the memory of
*nothing happening*.

### Extension — the `readOnly` variant

Run Part A again with `@Transactional(readOnly = true)`. You should get:
`managed? true`, no `update`, `entityUpdateCount=0`, row unchanged. Same silent
non-write, different cause: the flush mode is `MANUAL`, so dirty checking never ran.

Two different mechanisms produce identical observable behaviour. Being able to tell
them apart from a log — `contains()` is `true` under `readOnly`, `false` under
`detach` — is exactly the diagnostic skill this topic exists to build.

---

## Measurement

### Eyeballing the SQL log does not scale

Reading a SQL log works for a ten-line drill. It fails for anything real:

- One order-placement request issues 8–15 statements. At 60 rps that is 600+ lines
  per second, interleaved across 20 threads.
- You cannot assert on a log. A regression that adds 40 queries produces a longer
  log, which nobody reads.
- Log parsing is fragile: statement text changes with a Hibernate upgrade.

**Use counters. Assert on counters.**

### Turn on Hibernate statistics

```properties
spring.jpa.properties.hibernate.generate_statistics=true
```

Note the `spring.jpa.properties.` prefix — it forwards the key to Hibernate. Writing
`spring.jpa.generate_statistics` does nothing and is a common time-waster.

```java
SessionFactory sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);
Statistics stats = sessionFactory.getStatistics();
```

### The counters that matter for this topic

| Counter | What it tells you |
|---|---|
| `getEntityLoadCount()` | How many entity instances were materialised. **The number to watch for "am I loading 100,000 things".** |
| `getEntityInsertCount()` | `INSERT`s issued. |
| `getEntityUpdateCount()` | `UPDATE`s issued by dirty checking. **The counter for the failure drill.** |
| `getEntityDeleteCount()` | `DELETE`s issued. |
| `getFlushCount()` | How many times dirty checking ran. A surprisingly high number means `FlushMode.AUTO` is flushing before queries. |
| `getQueryExecutionCount()` | JPQL/HQL/criteria queries executed. |
| `getPrepareStatementCount()` | **Total JDBC statements.** The honest end-to-end number, and the one Topic 50 asserts on. |
| `getSessionOpenCount()` | Persistence contexts created. More than one per request means your transaction boundaries are not where you think. |

### The pattern to use in a test

```java
@Test
void markPaidIssuesExactlyOneUpdateAndNoSave() {
    Statistics stats = sessionFactory.getStatistics();
    stats.clear();

    orderStatusService.markPaid(seededOrderId);

    assertThat(stats.getEntityLoadCount()).isEqualTo(1);
    assertThat(stats.getEntityUpdateCount()).isEqualTo(1);
    assertThat(stats.getEntityInsertCount()).isZero();
}
```

That test fails if someone later adds an eager association that loads three extra
entities, or refactors the method into loading a collection. **A number in an
assertion is what stops a regression; a log line is not.** Topic 50 turns this into a
reusable rule and applies it to N+1.

`stats.clear()` before each measurement is not optional — the counters are
cumulative for the whole `SessionFactory`, so a shared test context accumulates
across tests. If your counts look absurdly high, this is why.

### Measuring memory honestly

The snapshot cost is real but do not quote a multiplier you have not measured.

```bash
# heap histogram while the transaction is open
jcmd <pid> GC.class_histogram | head -40
```

**What to look for:** instance counts for your entity classes, and for `Object[]` —
the snapshot arrays. If `Object[]` instance count tracks your entity count, you are
looking at the snapshots.

For a real answer, take a heap dump and open it in MAT:

```bash
jcmd <pid> GC.heap_dump /tmp/orderflow.hprof
```

...and look at what retains `StatefulPersistenceContext`. That is Topic 79's
technique; the point today is that the persistence context is a *retention root* for
everything it holds, for as long as the transaction runs.

### Do NOT time this with `System.nanoTime()`

```java
long start = System.nanoTime();
orderService.placeOrder(command);
long elapsed = System.nanoTime() - start;   // this number is not what you think
```

That measurement is wrong for at least five reasons, and they compound:

1. **JIT state.** The first invocations run interpreted. Your number is measuring
   compilation, not the operation.
2. **Connection acquisition.** The first call may create a pooled connection; later
   ones do not.
3. **Database cache state.** Postgres shared buffers are cold on the first read and
   warm afterwards, a difference of orders of magnitude.
4. **GC.** A young collection landing inside your window adds milliseconds that have
   nothing to do with the code.
5. **The persistence context is stateful.** The second call in the same context hits
   the identity map and issues no SQL at all, so you may be timing a map lookup.

Topic 77 is the full treatment: JMH forks a fresh JVM, warms up, uses `Blackhole` to
defeat dead-code elimination, and runs multiple forks. Until then, use **counters**
(deterministic, meaningful) rather than **timings** (noisy, misleading). A statement
count of 3 versus 300 is a real finding. A timing of 12 ms versus 14 ms is not.

### Where the counters go in production

Do not read `Statistics` by hand in production. Wire it to Micrometer and put
`entityLoadCount` and `prepareStatementCount` per endpoint on a dashboard next to
your RED metrics (Topic 118).

`[BOOT 3.x DELTA]` — Boot's auto-configured `hibernate.*` Micrometer metrics were
deprecated during the 3.x line in favour of Hibernate's own Micrometer integration,
and their availability differs by version. **Confirm on your build rather than
trusting this document:**

```bash
curl -s localhost:8080/actuator/metrics | jq -r '.names[]' | grep -i hibernate
```

If nothing comes back, you are on a version where the auto-configuration is gone;
register Hibernate's own Micrometer statistics implementation explicitly. Also note
`generate_statistics=true` has a measurable overhead — leave it on in test and
staging, and make it a toggle in production.

---

## Practice exercises

### 1 — Easy: name the state

Write a single `@Transactional` method that walks an `Order` through all four states,
printing `em.contains(order)` and the object's identity hash at each step:

1. `new Order(...)` — transient
2. after `em.persist(order)` — managed
3. after `em.detach(order)` — detached
4. after `Order managed = em.merge(order)` — note that `managed != order`
5. after `em.remove(managed)` — removed
6. after commit

For each step, write down **before running it**: what will `contains` return, and
what SQL (if any) will the log show at that point. Then run it and compare.

Where your prediction was wrong, write one sentence explaining the mechanism you had
wrong. That sentence is the actual output of this exercise.

### 2 — Medium: combining earlier topics (01–47)

Build a `WalletService.debit(Long customerId, long amountMinor)` for `orderflow`.

Requirements:

- `WalletRepository.findByCustomerId` returns `Optional<Wallet>` (Topic 26). Absence
  becomes a Topic 46 `ProblemDetail` 404 at the controller.
- `Wallet` uses `customerId` as the business key for `equals`/`hashCode` (Topics 13
  and 17). Write down why `customerId` is a valid business key here and what would
  make it invalid.
- `balanceMinor` is a `long` in pence, never `double` (Topic 01).
- `debit` is a method on the `Wallet` **entity**, not a setter called from the
  service. It throws `InsufficientFundsException` rather than allowing a negative
  balance.
- The service method calls `wallet.debit(amount)` and **no `save()`**. Add a comment
  explaining, in your own words, why the row still changes.
- Return a `record WalletBalance(Long customerId, long balanceMinor)` (Topic 27) —
  never the entity.

Then prove three things with the SQL log and Hibernate statistics:

1. A successful debit issues exactly one `UPDATE` and `entityUpdateCount == 1`.
2. Adding `@Transactional(readOnly = true)` makes the same code issue **zero**
   updates. Explain the mechanism in one sentence.
3. Adding `em.detach(wallet)` before `debit` **also** makes it issue zero updates.
   Explain how you would tell these two zero-update cases apart from a log alone.

Finally, add a JUnit test asserting `entityUpdateCount == 1`, and deliberately break
it by adding an eager association. Report which counter moved.

### 3 — Hard: production simulation

The nightly reconciliation job for `orderflow` must expire stale `PENDING` orders and
release their reserved inventory. There are roughly 120,000 stale orders on a bad
morning.

**Part A — write it the naive way.** Load all stale orders, iterate, call
`markCancelled()`, and release inventory by mutating the `Inventory` entities. One
transaction. Measure with `hibernate.generate_statistics`:
`entityLoadCount`, `entityUpdateCount`, `prepareStatementCount`, `flushCount`. Also
capture `jcmd <pid> GC.class_histogram` while it runs, and record peak heap from
`-Xlog:gc`.

**Part B — make it fail.** Run Part A with `-Xmx512m` and a concurrent load of 30 rps
against `GET /api/products`. Record two things: whether the batch OOMs, and what
happens to the *unrelated* catalogue endpoint's latency while the batch transaction
is open. Explain the second observation in terms of connection-pool occupancy, and
name the topic that covers it properly.

**Part C — chunk it.** Rewrite it to process 500 orders per transaction, calling
`entityManager.clear()` between chunks. Re-measure everything from Part A. Report the
change in peak heap and in the catalogue endpoint's latency.

**Part D — bulk it.** Rewrite it again as a single `@Modifying` JPQL update, with
`clearAutomatically` and `flushAutomatically`. Re-measure. Then write down
**everything you gave up**: at minimum `@PreUpdate` callbacks, `@Version` bumps
(Topic 52), domain events, and per-row business rules.

**Part E — decide.** Given your three sets of numbers, which implementation ships?
State the condition under which your answer flips. There is a defensible case for
each of B, C and D; a "chunked" answer that cannot say what it costs is not one of
them.

**Part F — the honest caveat.** Your wall-clock timings here are not trustworthy.
Write three sentences: why not, what they *are* useful for, and which counters you
would put in the pull-request description instead.

---

## Interview questions

### Q1 — "You loaded an entity, changed a field, never called save, and the row changed. Explain."

*This is the staple. Expect it in some form in every senior Java loop.*

**Mid-level answer:** "Spring/JPA saves it automatically at the end of the
transaction — that's dirty checking."

**Senior answer:** "When the entity was loaded, Hibernate put it in the persistence
context and took a snapshot of its field values — a `loadedState` array on the
`EntityEntry`. At flush, which the transaction commit triggers, Hibernate walks every
managed entity and compares each property against that snapshot. `status` differs, so
it queues an `EntityUpdateAction` in the `ActionQueue`, and the queue executes in a
fixed order — inserts, then updates, then deletes — which is not necessarily my call
order. `save()` would have been a no-op there: Spring Data's `save` calls `merge` for
a non-new entity, and merging an already-managed instance returns the same instance
and changes nothing. The cost of this is that the snapshot roughly doubles memory per
loaded entity and the dirty check is O(entities × properties) at every flush, which
is why loading 100,000 entities to change one field is a design error rather than
just slow."

**What separates them:** naming the snapshot as a distinct data structure, naming the
`ActionQueue` and its fixed ordering, explaining why `save()` would have been a
no-op, and volunteering the memory cost unprompted. The mid answer names the
behaviour; the senior answer names the mechanism and its price.

**Follow-up:** "How would you stop it from writing?" Three correct answers: detach
the entity, use `@Transactional(readOnly = true)` (which sets Hibernate's flush mode
to `MANUAL`), or do not load the entity at all — use a projection. The best answers
prefer the third and explain that the first two are guards on a design that already
loaded something it did not need.

---

### Q2 — "What is the difference between `persist`, `merge` and Spring Data's `save`?"

**Mid-level answer:** "`persist` inserts a new entity, `merge` updates an existing
one, and `save` does whichever is appropriate."

**Senior answer:** "`persist` takes a transient instance and makes *that instance*
managed; it throws if the entity already has an identifier that maps to a row.
`merge` takes a detached instance, finds or loads the managed copy, copies the state
onto it, and **returns the managed copy** — the instance you passed in stays
detached. That return value is the trap: `em.merge(order)` with the result discarded,
followed by more mutations to `order`, loses those mutations silently. Spring Data's
`save` inspects `isNew` — by default a null identifier, or a `@Version` field if
present — and calls `persist` or `merge` accordingly, returning the result. So for an
entity you just loaded inside the same transaction, `save` is a no-op with an extra
merge, and calling it is a lie about how the code works: it implies the write depends
on that call, which the next reader will believe."

**What separates them:** the return-value semantics of `merge`, the `isNew` check
behind `save`, and treating a redundant `save()` as a *readability* defect rather
than a harmless habit.

**Follow-up:** "When is a redundant `save()` actually harmful?" It can force an extra
`SELECT` on a detached entity, and with a `@Version` field the `isNew` heuristic
changes — so `save` on an entity with a non-null version takes the merge path.

---

### Q3 — "Your batch job OOMs at 120,000 rows. Walk me through the diagnosis."

**Mid-level answer:** "It's loading too many entities. I'd add pagination."

**Senior answer:** "Three compounding costs, and I'd confirm each. First: every
managed entity carries a snapshot array of its property values, so the memory is
roughly double the naive object size — more for primitive-heavy entities, because
the snapshot boxes them. I'd confirm with `jcmd GC.class_histogram` and look for
`Object[]` counts tracking my entity counts. Second: dirty checking is
O(entities × properties) at every flush, and under `FlushMode.AUTO` a flush happens
before any query touching those tables. `getFlushCount()` tells me how often.
Third: 120,000 individual `UPDATE`s, which batch poorly across different rows.
`getPrepareStatementCount()` gives me that number. The fix is either chunking —
process 500 per transaction with a `clear()` between chunks, which bounds both memory
and the dirty check — or a single bulk `@Modifying` update that loads nothing. Bulk
is faster and gives up `@PreUpdate` callbacks, version bumps and domain events, so
which one ships depends on whether those matter. And there's a fourth problem nobody
mentions: a multi-minute transaction pins a HikariCP connection for its whole
lifetime, so the batch degrades unrelated endpoints. That's usually the more urgent
bug."

**What separates them:** three named, separately-measurable costs; the specific
counter for each; the honest trade-off on the fix; and spotting the connection-pool
consequence, which is the one that causes the incident.

**Follow-up:** "How would you have caught this before production?" A test that
asserts on `entityLoadCount` and fails above a threshold. Coverage does not catch it;
a counter does.

---

### Q4 — "Why can't a record be a JPA entity?"

**Mid-level answer:** "Because records are immutable and JPA needs to set fields."

**Senior answer:** "Three separate blockers, any one of which is fatal. A record is
implicitly `final`, and Hibernate needs to generate a proxy subclass for lazy loading
— that's Topic 49's ByteBuddy proxy. A record has no no-arg constructor, and JPA
instantiates entities reflectively before populating them. And a record's fields are
`final`, so there is nowhere to write a generated identifier or a version. Records
are excellent as projections and DTOs — a JPQL constructor expression calls the
canonical constructor directly, which is exactly the right shape for a read model.
The rule I'd state is: records on the read side and the wire, mutable classes on the
entity side. That boundary is also where I stop lazy proxies from reaching Jackson."

**What separates them:** three distinct blockers rather than a vague "immutability",
naming the proxy requirement, and turning it into an architectural rule rather than a
limitation to work around.

**Follow-up:** "So how do you get immutability in an entity?" Honest answer: you
mostly do not, and that is a real cost of JPA. You approximate it — no public
setters, mutation only through named domain methods like `debit()` and `markPaid()`,
`updatable = false` on columns that must never change. Topic 17's guarantees do not
survive contact with JPA, and pretending otherwise is worse than admitting it.

---

### Q5 — "What does `@Transactional(readOnly = true)` actually do?"

**Mid-level answer:** "It optimises read queries and tells the database it's a read
transaction."

**Senior answer:** "Concretely, two things. Spring sets the Hibernate flush mode to
`MANUAL` for that transaction, so dirty checking never runs — no snapshot comparison,
no `UPDATE`s. That's both a performance saving on a hot read path and a genuine
correctness guard: it turns 'someone mutated a managed entity in a read method' from
a data-corruption bug into a no-op. Spring also marks the JDBC connection read-only,
which some drivers and some databases act on and others ignore. What it is *not* is a
security control — a native `@Modifying` query still writes, and a nested transaction
with different settings can write. I'd put it on every read-only service method as a
default, and I'd expect a reviewer to ask why if it were missing."

**What separates them:** naming `MANUAL` flush mode as the mechanism, framing it as a
correctness guard and not only an optimisation, and being explicit about the limit of
the guarantee.

**Follow-up:** "How would you prove the flush mode changed?" Run a method that
mutates a managed entity, once with and once without `readOnly`, and compare
`getEntityUpdateCount()`. Not a timing — a counter.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Hibernate cannot intercept a plain setter on your entity class, so dirty checking
   must scan everything at flush. Design an alternative that would not need the
   scan — and then say what you would have to give up to get it. (You are
   re-deriving bytecode enhancement.)

2. The `ActionQueue` orders inserts before deletes, always. Construct a case where
   that ordering is *necessary* for correctness, not merely inconvenient. Then
   construct the case where it breaks you. What does that tell you about why the
   order is fixed rather than configurable?

3. The identity map guarantees `==` for the same row within one context. Is that a
   feature you would keep if you were designing an ORM today? Argue both sides. What
   would break in `orderflow` if two loads of order 42 returned two objects?

4. `merge` returns a different instance than the one you passed. Why was it designed
   that way rather than mutating the argument in place? What invariant does the
   design protect?

5. A managed entity mutation writes; a detached one does not; neither logs anything.
   Design a development-time safety net that would make the detached case *loud*.
   What would it cost, and why do you think JPA does not ship one?

6. `@Transactional(readOnly = true)` prevents dirty checking. Given that, is there any
   remaining reason to prefer a projection over loading the entity on a read path?
   Give at least three.

7. You are told "we should just call `save()` everywhere, it's clearer". Steelman
   that position properly — there is a real argument in it — then give your rebuttal
   and say what you would actually enforce in review.

---

## Quick reference card

### States

| State | `em.contains` | Next flush does |
|---|---|---|
| Transient | false | nothing |
| Managed | **true** | dirty check → `INSERT`/`UPDATE` |
| Detached | false | **nothing — silently** |
| Removed | true | `DELETE` |

### Transitions

```java
em.persist(e)        // transient -> managed
em.find(C.class, id) // -> managed (identity map first, then SELECT)
em.merge(e)          // detached -> A DIFFERENT managed instance. USE THE RETURN VALUE.
em.remove(e)         // managed -> removed
em.detach(e)         // managed -> detached (one entity)
em.clear()           // managed -> detached (everything)
em.flush()           // dirty check + SQL now. Not a commit.
em.contains(e)       // the diagnostic. true == managed.
```

### Flush modes

| Mode | Flushes |
|---|---|
| `AUTO` (default) | at commit, and before a query touching a dirty table |
| `COMMIT` | at commit only |
| `MANUAL` | only on explicit `flush()` — **this is what `readOnly = true` sets** |

### Flush ordering (fixed, not yours)

```
orphan removals -> INSERTs -> UPDATEs -> collection ops -> DELETEs
```

### Entity conventions for `orderflow`

```java
@Entity @Table(name = "orders")                       // reserved word
@GeneratedValue(strategy = GenerationType.SEQUENCE)   // never IDENTITY - Topic 53
@SequenceGenerator(allocationSize = 50)
@Enumerated(EnumType.STRING)                          // never ORDINAL
@ManyToOne(fetch = FetchType.LAZY)                    // EAGER is the default - Topic 49
protected Order() { }                                 // no-arg, not public
equals/hashCode on a BUSINESS KEY, using instanceof   // never getClass(), never the generated id
money as long minor units                             // Topic 01
```

### Diagnostics

```properties
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE       # Hibernate 6+; CONFIRM on your version
spring.jpa.properties.hibernate.generate_statistics=true   # note the properties. prefix
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.open-in-view=false                          # Topic 49
```

```java
em.contains(entity)                    // managed or detached?
stats.getEntityLoadCount()             // am I loading too much?
stats.getEntityUpdateCount()           // did dirty checking write?
stats.getFlushCount()                  // is AUTO flushing more than I expect?
stats.getPrepareStatementCount()       // the honest total
stats.clear()                          // BEFORE every measurement
```

### Checklist

- [ ] `save()` only for transient entities. Never for loaded ones.
- [ ] Never pass an entity across a transaction boundary. Pass the ID.
- [ ] Every read-only service method is `@Transactional(readOnly = true)`.
- [ ] `equals`/`hashCode` use a business key and `instanceof`.
- [ ] Never `==` on entities.
- [ ] Bulk updates instead of loading tens of thousands of entities.
- [ ] `clear()` between chunks in any batch.
- [ ] Assert on counters in tests, not on log lines.
- [ ] `stats.clear()` before every measurement.
- [ ] Never time persistence with `System.nanoTime()`.

---

## When would I use this at work?

**1. Reviewing a pull request that touches a service method.**
Two questions answer most of it: is every entity in this method managed at the point
it is mutated, and is there a `save()` that implies a write the code does not
actually depend on. Both are ten-second reads once the model is in your head, and
both catch bugs that are invisible in testing.

**2. Debugging "the update didn't happen" — the most common Hibernate ticket there
is.**
`em.contains(entity)` at the point of mutation answers it in one line. `true` means
look at the flush mode (`readOnly`?) or an exception before commit; `false` means the
entity is detached and you need to find where the transaction boundary actually is.
This turns a half-day hunt into a five-minute one.

**3. Sizing a batch job before writing it.**
"We need to update 2 million rows nightly" has three implementations with three
different memory and connection-pool profiles. Being able to say "loading them costs
roughly double the row size in heap, plus a dirty check over every property at every
flush, plus one round trip per row, and it pins a connection for the whole run" turns
a guess into a design conversation — before the incident rather than after.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and boxing**: the snapshot array boxes every primitive property.
  Money as `long` minor units, never `double`.
- **13 — equals/hashCode contract**: a generated ID is null before persist, so a
  hash-based collection loses the entity. This is that drill, reproduced by JPA.
- **17 — Immutability**: JPA entities cannot be immutable. Understand what you are
  giving up rather than pretending you are not.
- **26 — Optional**: repository return type, and how absence becomes a 404.
- **27 — Records**: excellent projections, and — for three specific reasons —
  impossible as entities.
- **37 — Bean lifecycle**: `@PostConstruct` runs before AOP proxies exist for that
  bean, which is why a `@Transactional` call from it has no persistence context.
- **40 — Proxying**: `@Transactional` is a proxy. Self-invocation means no
  transaction, which means no persistence context, which means everything in this
  document silently does not apply.
- **46 — `ProblemDetail`**: rollback happens *before* your advice runs, so the advice
  never has a live persistence context. Never touch an entity in an exception
  handler.
- **47 — Spring Data JPA**: where the managed entities come from, and the
  first-level-cache staleness caused by `@Modifying`.

**This unlocks:**
- **49 — Associations and lazy loading**: what the proxy is, why `getClass()` lies,
  and why a detached entity with a lazy collection is a 500 in your serializer.
- **50 — N+1**: `getPrepareStatementCount()` as an assertion, which is the only
  honest proof of a fix.
- **51 — Caching**: the persistence context *is* the first-level cache. The
  second-level cache sits behind it and has completely different invalidation rules.
- **52 — Locking**: `@Version` is a field the dirty check bumps; optimistic locking
  is dirty checking with an extra `where version = ?`.
- **53 — Batching**: why `GenerationType.IDENTITY` forces an immediate insert and
  kills JDBC batching, and why you must `clear()` periodically.
- **54 / 55 — `@Transactional`**: the boundary that *creates* the persistence
  context, propagation, and why a transaction pins a connection for its lifetime.
- **61 — Testcontainers**: persistence tests where the write and the read happen in
  different transactions, which is the only kind that tests reality.
- **69 / 79 — Object layout and heap dumps**: measuring the snapshot cost properly
  instead of quoting a multiplier.
- **77 — JMH**: why the `System.nanoTime()` measurement in this document's
  Measurement section is wrong, in detail.
- **109 — HikariCP**: a long transaction is a held connection. The batch job in
  Exercise 3 Part B is that lesson in miniature.

---

*Java baseline 21, running on JDK 25, Spring Boot 4.1 / Framework 7.0,
`jakarta.persistence.*` throughout. Persistence-context semantics are JPA
specification behaviour and are identical across Boot 2.x, 3.x and 4.x — nothing in
this topic is version-dependent. What is version-dependent is the diagnostic
plumbing: the Hibernate bind-parameter logging category changed between Hibernate 5
and 6, and Boot's auto-configured Hibernate Micrometer metrics were deprecated during
the 3.x line. Confirm both on your own build with the commands given above rather
than trusting any document.*
