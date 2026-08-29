# 52 — Hibernate V — optimistic `@Version`, pessimistic locking, lost updates

## Phase: 5 — Spring Boot & Persistence
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: the two contended writes in `orderflow` — **inventory decrement** (`Inventory.available`, hammered by a flash sale on a hot product) and **wallet debit** (`Wallet.balanceMinor`, contended per customer). Everything else in the service is read-mostly. These two rows are where correctness is decided.

---

## Mechanical statement

**Optimistic locking** adds a version column to the `WHERE` clause of every `UPDATE`
Hibernate generates for that entity:

```sql
update inventory set available = ?, version = ? where id = ? and version = ?
```

If another transaction committed first, the version no longer matches, **zero rows are
affected**, and Hibernate raises the failure. Critically, it raises it at **flush or
commit** — not at the setter, not at the `save()`, but at the moment the SQL runs. In a
`@Transactional` method that is *after your method body has returned*.

That single fact determines where the retry has to live: **outside the transaction, not
inside the method.**

**Pessimistic locking** issues `SELECT … FOR UPDATE`. The database takes a row lock and
holds it **for the transaction's entire remaining life** — not until you finish reading,
not until you write, but until `COMMIT` or `ROLLBACK`. Other writers to that row block.
Readers do not (Postgres MVCC). Because the transaction also holds a pooled connection
for its whole life, a pessimistic lock converts row contention into connection-pool
contention, which is how a correct fix to one endpoint takes down every other endpoint.

**A single atomic conditional `UPDATE`** does the read and the write in one statement:

```sql
update inventory set available = available - ? where product_id = ? and available >= ?
```

No entity is loaded, no version is compared, no application-level read-modify-write
window exists, and the row lock is held from the moment the statement executes until
commit — which, if the statement is the last thing in the transaction, is microseconds.
The affected-row count is your answer: 1 means you got the stock, 0 means you did not.

Three mechanisms, three cost profiles. The senior skill is choosing between them with a
reason, and being honest that for a simple decrement the third usually wins.

---

## The bridge from what you know

### `@Version` is the row-versioning you already know. Say so and move on.

You have written this by hand in SQL:

```sql
UPDATE inventory
SET available = 4, version = 8
WHERE id = 77 AND version = 7;
-- check rowCount; if 0, someone else won
```

`@Version` is exactly that, generated for you on every `UPDATE` of the entity. **The
concept transfers completely.** You do not need it re-explained.

Here is what is genuinely Java-specific, and it is where every mistake lives:

**1. You never see the `WHERE version = ?`.** You write `inventory.setAvailable(4)`.
Hibernate's dirty checking (Topic 48) generates the `UPDATE` at flush. The version
predicate is added by the framework. There is no line of code to read.

**2. The failure arrives at a place you did not write.** In your hand-written SQL you
checked `rowCount` on the line after the `UPDATE`. In Hibernate the `UPDATE` runs at
flush — typically at transaction commit, which the `@Transactional` proxy performs
*after* your method returns. So the exception is thrown from the proxy, not from your
code. A `try/catch` inside the method **cannot see it.**

**3. The exception type depends on which layer you catch at.**

| Layer | Type |
|---|---|
| Hibernate | `org.hibernate.StaleObjectStateException` (a `StaleStateException`) |
| JPA | `jakarta.persistence.OptimisticLockException` |
| Spring | `org.springframework.orm.ObjectOptimisticLockingFailureException` (extends `OptimisticLockingFailureException`, extends `DataAccessException`) |

Spring's exception translation (the reason the whole data-access hierarchy is unchecked
— Topic 09) converts the provider exception at the repository boundary. **Catch the
Spring type.** It is stable across providers and it is what `@Retryable` should be
configured on.

**4. There is no `rowCount` for you to check.** Which means there is no place to write
"if 0, retry". The retry becomes a structural concern, not a line of code — and getting
its placement wrong is Trap 2, the most common bug in this topic.

### Pessimistic locking transfers cleanly too

`SELECT … FOR UPDATE` is `SELECT … FOR UPDATE`. You know what it does in Postgres. What
is new is only the coupling: **a JPA transaction holds a pooled connection for its
entire life**, and the lock lives exactly as long as the transaction. In a Node service
with a per-query connection you might hold a lock for one statement. Here you hold it
for everything the method does afterwards, including that HTTP call someone added last
sprint.

### The genuinely new Java problem: where does the retry go?

This has no analogue in your world because you never had an ORM commit for you at a
boundary you did not write. It is Topic 40 (proxies) plus Topic 54 (`@Transactional`)
plus this topic, and the interaction is where people lose the interview. Hold it in
mind through Trap 2.

---

## What is this?

### A lost update, stated precisely

Two transactions read the same row, both compute a new value from what they read, both
write. The second write overwrites the first, and the first transaction's effect
vanishes — with no error anywhere.

```
T1: SELECT available FROM inventory WHERE product_id=88   -> 1
T2: SELECT available FROM inventory WHERE product_id=88   -> 1
T1: if (1 >= 1) UPDATE inventory SET available = 0        -> commits
T2: if (1 >= 1) UPDATE inventory SET available = 0        -> commits
```

Two orders accepted. One unit of stock. **Oversell.** Both transactions committed
successfully; both were internally consistent; the database is in a state neither of
them would have produced alone.

This is Topic 92's check-then-act race, at the database instead of in a
`ConcurrentHashMap`. Same shape, same fix pattern: make the check and the act one
atomic operation, or make the act detect that the check is stale.

**Read committed does not prevent this.** Neither does repeatable read in the general
case for this pattern (though Postgres's implementation raises a serialization failure
for this specific shape — see Machine-level reality). Isolation levels are about what a
transaction can *see*; a lost update is about what happens when two transactions both
act on what they saw.

### Optimistic locking

```java
package com.orderflow.inventory;

import jakarta.persistence.*;

@Entity
public class Inventory {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "inventory_seq")
    private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", unique = true)
    private Product product;

    private int available;

    @Version
    private long version;          // Hibernate manages this. Never set it yourself.

    public void reserve(int quantity) {
        if (available < quantity) {
            throw new InsufficientStockException(product.getId(), quantity, available);
        }
        available -= quantity;
    }
}
```

`@Version` supports `int`, `Integer`, `long`, `Long`, `short`, `Short`, and timestamp
types (`Instant`, `LocalDateTime`, `java.sql.Timestamp`). **Use a `long`.** A timestamp
version has clock-resolution collisions — two updates in the same millisecond can both
appear to be "the same version" — and clock skew across instances makes it worse.
Numeric versions have neither problem.

Every `UPDATE` for a versioned entity becomes:

```sql
update inventory set available=?, version=? where id=? and version=?
```

Zero rows affected ⇒ Hibernate throws. The version is incremented in the same
statement, so there is no window.

### Pessimistic locking

```java
public interface InventoryRepository extends JpaRepository<Inventory, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select i from Inventory i where i.product.id = :productId")
    Optional<Inventory> findForUpdate(@Param("productId") Long productId);
}
```

or directly:

```java
Inventory inv = em.find(Inventory.class, id, LockModeType.PESSIMISTIC_WRITE);
```

The lock modes that matter:

| `LockModeType` | Postgres SQL (dialect-dependent — check your SQL log) | Meaning |
|---|---|---|
| `PESSIMISTIC_WRITE` | `select … for update` | exclusive row lock; other writers block |
| `PESSIMISTIC_READ` | `select … for share` | shared row lock; other readers may share, writers block |
| `PESSIMISTIC_FORCE_INCREMENT` | `for update` + version bump | lock **and** bump the version so optimistic readers elsewhere notice |
| `OPTIMISTIC` | none | re-check the version at commit even if you only *read* the entity |
| `OPTIMISTIC_FORCE_INCREMENT` | none | bump the version at commit even without a change — versions an aggregate when a child changes |

> The exact SQL Hibernate emits for each mode is decided by the dialect and has varied
> across Hibernate versions (Postgres has `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR
> SHARE` and `FOR KEY SHARE`, and which one a mode maps to is a dialect decision).
> **Do not memorise the mapping. Read it off `logging.level.org.hibernate.SQL=DEBUG`
> for your version** — it takes ten seconds and it is the only trustworthy answer.

Lock timeouts:

```java
em.find(Inventory.class, id, LockModeType.PESSIMISTIC_WRITE,
        Map.of("jakarta.persistence.lock.timeout", 3000));   // milliseconds
```

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
Optional<Inventory> findForUpdate(Long productId);
```

Two special values, both supported by Postgres and worth knowing by name:

- **`0` ⇒ `NOWAIT`** — fail immediately if the row is locked. Use when you would rather
  return "try again" than queue.
- **`-2` (`LockOptions.SKIP_LOCKED`) ⇒ `SKIP LOCKED`** — silently skip locked rows.
  This is the correct primitive for a job-queue table, and it is one of the concrete
  things H2 does not have, which is why Topic 61 insists on real Postgres.

**Set a lock timeout. Always.** The default is to wait forever, and "wait forever" while
holding a pooled connection is how a single hot row takes down the whole service. If
the hint is not honoured on your setup, fall back to the database:
`SET LOCAL lock_timeout = '3s'` at the start of the transaction.

### The atomic conditional update

```java
public interface InventoryRepository extends JpaRepository<Inventory, Long> {

    @Modifying
    @Query("""
           update Inventory i
              set i.available = i.available - :qty
            where i.product.id = :productId
              and i.available >= :qty
           """)
    int tryReserve(@Param("productId") Long productId, @Param("qty") int qty);
}
```

```java
int updated = inventory.tryReserve(productId, quantity);
if (updated == 0) {
    throw new InsufficientStockException(productId, quantity);
}
```

One statement. No entity loaded, no version compared, no persistence context, no
retry. The `WHERE` clause **is** the check and the `SET` **is** the act, and the
database performs both atomically. The return value is your `rowCount` — the thing
Hibernate took away from you in the optimistic case, handed back.

**This is the correct default for a simple counter decrement**, and the fact that it
sits outside the entity model is the reason people do not reach for it. Say so out
loud in an interview; it is a differentiating answer.

---

## Why does it matter?

**1. Oversell is a business event, not a bug.**
A flash sale on a hot `orderflow` product with 500 units and 3,000 concurrent attempts
in 60 seconds will oversell without a correct mechanism. Every oversold order becomes a
refund, a support contact, and a customer who does not come back. It does not throw. It
does not log. It looks exactly like a successful sale until fulfilment fails.

**2. The wrong fix creates a worse outage.**
Pessimistic locking on a hot row serialises every writer at the database while each one
holds a pooled connection. Ten connections, a 200 ms critical section, and one hot
product means a maximum of 50 orders per second for that product **and every other
endpoint queuing behind an exhausted pool.** You fixed correctness and created an
availability incident.

**3. Retry placement is the discriminating interview question.**
"Add `@Version` and retry" is the mid answer and it is *wrong as stated*, because the
naive placement of the retry does nothing. Knowing why — the exception fires at commit,
outside your method — is the thing that separates people who have read about optimistic
locking from people who have shipped it.

---

## Machine-level reality

### What a Postgres row lock actually does

Postgres is MVCC. Every row version carries the transaction id that created it and the
one that deleted it. Consequences:

**Readers never block writers, and writers never block readers.** A plain `SELECT`
against a row locked `FOR UPDATE` returns the last committed version immediately. This
is the single most important difference from a lock-based database, and it means a
pessimistic lock on `Inventory` does **not** slow down catalogue reads.

**Writers block writers, on the tuple.** A second `UPDATE` (or `SELECT … FOR UPDATE`)
against a locked row sleeps on the tuple lock until the holder commits or rolls back.
While it sleeps it holds its own connection, its own transaction, and its own snapshot.

**The wait is a queue with no fairness guarantee you should rely on.** Ten waiters means
the tenth waits for nine critical sections. Throughput for that row is
`1 / criticalSectionDuration`, full stop. If your critical section is 20 ms, that row
supports 50 writes per second no matter how many application instances you run.

**After the holder commits, `READ COMMITTED` re-evaluates.** This is the subtle and
genuinely useful bit. Under `READ COMMITTED`, a blocked `UPDATE … WHERE available >= 1`
does not just proceed with its stale snapshot. Postgres re-reads the newly committed row
version and **re-evaluates the `WHERE` clause against it** (the mechanism is called
EvalPlanQual). So the atomic conditional update is *correct* under `READ COMMITTED` —
the guard is checked against the fresh value, not the value from the snapshot.

**Under `REPEATABLE READ`, the same situation raises an error instead**: `could not
serialize access due to concurrent update`, SQLSTATE `40001`. Postgres refuses to
silently re-evaluate under a snapshot that promised repeatability. So the atomic
conditional update needs a retry loop at `REPEATABLE READ` and needs none at
`READ COMMITTED`. Spring's default is the database default, which for Postgres is
`READ COMMITTED`. **Know which one you are on before reasoning about any of this** —
Topic 55.

### Deadlock

Two transactions that lock the same two rows in opposite orders deadlock:

```
T1: lock product 88 ... then lock product 91
T2: lock product 91 ... then lock product 88
```

Postgres detects it after `deadlock_timeout` (default **1 second**) and kills one
transaction with SQLSTATE `40P01`. Note the cost: **the victim waited a full second
before being killed**, holding a connection the whole time.

`orderflow` produces this naturally. A multi-line order locks the inventory row for
each line. Two orders containing the same two products in different line orders
deadlock. The fix is **consistent lock ordering** — sort the product ids before
locking — and it is one line of code that eliminates an entire class of incident.

### The connection-pool coupling — the systemic cost

A transaction holds a pooled connection from `begin` to `commit`. A pessimistic lock is
held for the same interval. Therefore:

```
lockWaitTime  ⊆  transactionDuration  =  connectionHeldTime
```

With HikariCP at 10 connections, 10 concurrent requests waiting on one hot row means
**zero connections available for anything else.** Catalogue reads start timing out. The
health check starts timing out. Kubernetes restarts the pod, the in-flight transactions
roll back, and the retry storm begins.

This is Topic 109's failure, and Topic 55's, triggered from here. **The row lock is not
the risk. The connection you hold while waiting for it is.**

Three mitigations, all of which you should be able to name:

1. **Shrink the critical section.** Take the lock as late as possible, commit as soon
   as possible, and never do I/O between them.
2. **Bound the wait.** `lock.timeout` of 2–3 s, so a waiter gives up and returns the
   connection rather than queuing indefinitely.
3. **Do not take the lock at all** when a single conditional `UPDATE` will do — the
   lock is then held for the duration of one statement instead of the duration of your
   method.

### Why the atomic conditional `UPDATE` usually wins, mechanically

Compare the round trips and the lock-hold time for one inventory decrement:

| Approach | Statements | Lock held for | Retries under contention |
|---|---|---|---|
| `@Version` + retry | `SELECT` + `UPDATE` (+ full retry on conflict) | the `UPDATE` only | yes — and each retry is a whole new transaction |
| `PESSIMISTIC_WRITE` | `SELECT … FOR UPDATE` + `UPDATE` | from the `SELECT` to commit — **includes all your application logic in between** | no |
| Conditional `UPDATE` | **one** `UPDATE` | from the statement to commit | no |

Under **low** contention, `@Version` is cheapest: no lock wait, no conflicts, two
statements.

Under **high** contention, `@Version` degrades badly. Conflict probability rises with
concurrency, every conflict costs a full transaction rollback and replay, and retries
add load to an already-contended row. This is a livelock-shaped failure: throughput can
*decrease* as concurrency increases.

`PESSIMISTIC_WRITE` has stable throughput bounded by `1 / criticalSection` — but the
critical section includes everything between the `SELECT` and the commit, which is where
people accidentally put a wallet debit, a payment-gateway call, and an event publish.

The conditional `UPDATE` has the shortest possible critical section (one statement), no
retries, and one round trip. It scales until Postgres's own tuple-lock contention on
that single row becomes the limit — which is a much higher ceiling than any of the
above.

**The honest cost:** it bypasses the entity model. No dirty checking, no `@Version`
increment, no L2 invalidation (Topic 51, Trap 1 — a JPQL bulk update does invalidate;
a native one does not unless you declare the space), no domain method holding the
business rule. You have traded model expressiveness for throughput and correctness. For
`Inventory.available`, that is the right trade and you should say so plainly.

---

## Example 1 — minimal

Reproduce the lost update, then prevent it. Two threads, one row.

```java
package com.orderflow.inventory;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class NaiveInventoryService {

    private final InventoryRepository inventory;

    NaiveInventoryService(InventoryRepository inventory) { this.inventory = inventory; }

    /** Check-then-act. This is the bug. */
    @Transactional
    public void reserve(Long productId, int quantity) {
        Inventory inv = inventory.findByProductId(productId).orElseThrow();
        if (inv.getAvailable() >= quantity) {          // CHECK
            inv.setAvailable(inv.getAvailable() - quantity);   // ACT (dirty checking writes at flush)
        } else {
            throw new InsufficientStockException(productId, quantity);
        }
    }
}
```

Drive it from two threads against a row with `available = 1`:

```java
@Test
void twoConcurrentReservationsAgainstOneUnit() throws Exception {
    seedInventory(productId, 1);

    var barrier = new CyclicBarrier(2);
    var pool = Executors.newFixedThreadPool(2);
    var failures = new java.util.concurrent.atomic.AtomicInteger();

    Runnable attempt = () -> {
        try {
            barrier.await();                    // maximise the overlap
            naiveInventoryService.reserve(productId, 1);
        } catch (Exception e) {
            failures.incrementAndGet();
            System.out.println("rejected: " + e.getClass().getSimpleName());
        }
    };

    pool.submit(attempt); pool.submit(attempt);
    pool.shutdown();
    pool.awaitTermination(10, TimeUnit.SECONDS);

    int finalStock = jdbc.queryForObject(
        "select available from inventory where product_id = ?", Integer.class, productId);

    System.out.println("rejections = " + failures.get());
    System.out.println("final stock = " + finalStock);
}
```

**How to read what you get:**

| `rejections` | `final stock` | What it means |
|---|---|---|
| 0 | 0 | **Both succeeded. Oversell reproduced.** Two orders, one unit. This is the reading the drill wants. |
| 0 | −1 | Oversell, and the counter went negative — the guard did nothing at all. |
| 1 | 0 | One was rejected. Either the threads did not overlap, or something is already protecting the row. |
| 0 | 1 | Neither transaction committed. Check your transaction boundaries. |

If you get 1 rejection, **the threads did not actually race**. `CyclicBarrier` helps but
is not a guarantee; increase to 8 threads and run the test 20 times. A race that only
reproduces sometimes is still a race, and "it passed once" is not evidence.

Now add `@Version` to `Inventory` and re-run:

| `rejections` | `final stock` | What it means |
|---|---|---|
| 1 | 0 | **Correct.** One committed, one got `ObjectOptimisticLockingFailureException`. No oversell. |
| 0 | 0 | The version column is not being used. Check `@Version` is `jakarta.persistence.Version`, that the column exists in the schema, and that Hibernate generated `where id=? and version=?` — read the SQL log. |

Read the SQL log and confirm the shape:

```
DEBUG o.h.SQL : update inventory set available=?,version=? where id=? and version=?
```

*(That line is an illustration of the format, not captured output. What matters is that
the trailing `and version=?` is present. If it is not, `@Version` is not wired up.)*

---

## Example 2 — production scenario (on the project spine)

### The situation

`orderflow` order placement is 10% of the Topic 65 traffic mix. `POST /orders`
currently does, in one transaction:

1. load `Inventory` for each order line, check and decrement
2. load `Wallet` for the customer, check balance, debit
3. insert `Order` + `OrderLine` rows
4. insert a `Payment` row in `PENDING`
5. publish an `OrderPlaced` event

### The real constraints

- **A flash sale**: one hot product, 500 units, ~3,000 placement attempts in 60 seconds
  (peak ~120 concurrent). Two more products are moderately hot.
- **Wallets**: contention is low for consumer wallets (one customer, one device) but
  high for a handful of **corporate wallets** shared by a purchasing team — up to 15
  concurrent debits.
- 1,000,000 existing orders, 5,000,000 lines, 100,000 products.
- HikariCP pool size 20 (Topic 109's tuning).
- **SLO: `POST /orders` p99 < 300 ms; oversell rate exactly zero.**
- Postgres, default isolation `READ COMMITTED`.

Note that the two contended writes have **different contention shapes**, and that is
the whole design problem. One hot row hammered by strangers; many cool rows with a few
hot ones. The same mechanism is not right for both.

### Decision 1 — inventory: atomic conditional `UPDATE`

The inventory decrement is a bounded counter with a guard. It has no interesting domain
logic — "subtract if there is enough" is the entire rule, and SQL expresses it exactly.

```java
public interface InventoryRepository extends JpaRepository<Inventory, Long> {

    @Modifying
    @Query("""
           update Inventory i
              set i.available = i.available - :qty
            where i.product.id = :productId
              and i.available >= :qty
           """)
    int tryReserve(@Param("productId") Long productId, @Param("qty") int qty);

    @Modifying
    @Query("update Inventory i set i.available = i.available + :qty where i.product.id = :productId")
    int release(@Param("productId") Long productId, @Param("qty") int qty);
}
```

```java
@Service
public class InventoryService {

    private final InventoryRepository inventory;

    InventoryService(InventoryRepository inventory) { this.inventory = inventory; }

    /** Reserves all lines or throws. Lock order is by product id — see Trap 5. */
    void reserveAll(List<OrderLine> lines) {
        List<OrderLine> ordered = lines.stream()
                .sorted(Comparator.comparing(l -> l.getProduct().getId()))
                .toList();

        for (OrderLine line : ordered) {
            int updated = inventory.tryReserve(line.getProduct().getId(), line.getQuantity());
            if (updated == 0) {
                throw new InsufficientStockException(line.getProduct().getId(), line.getQuantity());
            }
        }
    }
}
```

Why this and not `@Version`: at 120 concurrent attempts on one row, optimistic conflicts
would be the common case, not the exception. Every conflict is a wasted transaction plus
a retry, and the retries pile onto the same row. Throughput would fall as load rose —
exactly the wrong shape for a flash sale.

Why this and not `PESSIMISTIC_WRITE`: a pessimistic lock would be held from the
`SELECT` through the wallet debit, the four inserts and the event publish. At ~40 ms of
work per order that caps the hot product at ~25 orders per second, and 120 concurrent
requests would each hold a connection while queuing — pool exhaustion at size 20.

The conditional `UPDATE` holds the tuple lock from the statement to commit. Keep the
inventory decrement **last** among the writes and that is a few milliseconds.

**The trade you are accepting, written down:** no `@Version` increment on `Inventory`,
so any *other* code path that loads `Inventory` as an entity and mutates it will not see
these decrements as version conflicts. The mitigation is that `Inventory.available` is
written **only** through this repository method. Enforce it with an ArchUnit rule, not
with a comment.

**And the L2 consequence (Topic 51):** a JPQL bulk update invalidates the entity region.
A *native* one would not. If you rewrite this as `nativeQuery = true` for any reason,
you must add `addSynchronizedEntityClass`. That is a real trap waiting six months out.

### Decision 2 — wallet: `@Version` with a retry, because the debit has domain logic

The wallet debit is not a bare decrement. It applies an overdraft policy, records a
ledger entry, and may trigger a low-balance notification. That logic belongs in the
domain model, and contention is low for the overwhelming majority of wallets.

```java
@Entity
public class Wallet {

    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "wallet_seq")
    private Long id;

    private String customerRef;
    private long balanceMinor;         // money as long minor units — Topic 01
    private long overdraftLimitMinor;

    @Version
    private long version;

    public void debit(long amountMinor) {
        long resulting = balanceMinor - amountMinor;
        if (resulting < -overdraftLimitMinor) {
            throw new InsufficientFundsException(customerRef, amountMinor, balanceMinor);
        }
        balanceMinor = resulting;
    }
}
```

### Decision 3 — where the retry goes. This is the part that is easy to get wrong.

The `OptimisticLockException` is raised when the transaction commits. The
`@Transactional` proxy commits **after** `placeOrder` returns. So the retry must wrap a
call that includes the commit — meaning it must sit on a **different bean**, outside the
transactional proxy.

```java
/** Outer bean. Retry lives here. NOT transactional. */
@Service
public class OrderPlacementFacade {

    private final OrderPlacementService placement;   // a different bean -> a real proxy call

    OrderPlacementFacade(OrderPlacementService placement) { this.placement = placement; }

    @Retryable(
        retryFor = ObjectOptimisticLockingFailureException.class,
        maxAttempts = 4,
        backoff = @Backoff(delay = 25, multiplier = 2.0, random = true))   // jitter matters
    public OrderId place(PlaceOrderCommand command) {
        return placement.placeOrder(command);        // the whole transaction, retried whole
    }

    @Recover
    public OrderId giveUp(ObjectOptimisticLockingFailureException e, PlaceOrderCommand command) {
        throw new OrderContentionException(
            "wallet contention exceeded retry budget for " + command.customerRef(), e);
    }
}
```

```java
/** Inner bean. Transaction lives here. NO retry annotation. */
@Service
public class OrderPlacementService {

    @Transactional
    public OrderId placeOrder(PlaceOrderCommand command) {
        inventoryService.reserveAll(command.lines());          // conditional UPDATE
        Wallet wallet = wallets.findByCustomerRef(command.customerRef()).orElseThrow();
        wallet.debit(command.totalMinor());                    // @Version checked at commit
        Order order = orders.save(Order.from(command));
        payments.save(Payment.pendingFor(order, command.totalMinor()));
        events.publish(new OrderPlaced(order.getId()));        // see the note below
        return new OrderId(order.getId());
    }
}
```

Four properties of that structure, each of which someone gets wrong:

1. **`@Retryable` and `@Transactional` are on different beans.** Two separate proxies,
   the retry outside the transaction. Putting both on the same method means the retry
   proxy and the transaction proxy are on the *same* call — and depending on
   `@Order` you can easily end up retrying inside a transaction that is already marked
   rollback-only. See Trap 2.

2. **The retry re-runs the whole transaction.** A fresh persistence context, a fresh
   read of the wallet, the current version. Retrying with the stale entity would fail
   identically forever.

3. **Backoff has jitter** (`random = true`). Without it, N conflicting transactions
   retry in lockstep and collide again. This is Topic 111's retry-storm lesson, and it
   applies at 4 concurrent requests exactly as it does at 400.

4. **`@Recover` converts exhaustion into a domain exception**, which Topic 46 maps to
   HTTP 409 with a `ProblemDetail`. A retry budget that ends in a stack trace is not a
   retry policy.

> **The event publish inside the transaction is a bug**, and it is Topic 116/117's
> subject: publishing before commit means you can publish an event for a transaction
> that then rolls back — and a retry publishes it again. The correct shape is an outbox
> row written in the same transaction. It is flagged here so it does not read as
> endorsed.

### The result you will verify

| | Inventory | Wallet |
|---|---|---|
| Mechanism | atomic conditional `UPDATE` | `@Version` + retry outside the transaction |
| Contention | one hot row, ~120 concurrent | mostly none; ~15 concurrent on corporate wallets |
| Failure mode | `updated == 0` ⇒ out of stock (a business answer) | `ObjectOptimisticLockingFailureException` ⇒ retry, then 409 |
| Lock hold | one statement to commit | the `UPDATE` at commit |
| Retries | none | up to 4, jittered |

Different mechanisms for two writes in the same transaction, each chosen from the
contention shape. **That is the answer to "how do you handle concurrency in
`orderflow`", and it is a much better answer than naming one technique.**

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — check-then-act with no locking at all

**Wrong:**

```java
@Transactional
public void reserve(Long productId, int qty) {
    Inventory inv = inventory.findByProductId(productId).orElseThrow();
    if (inv.getAvailable() >= qty) { inv.setAvailable(inv.getAvailable() - qty); }
}
```

**Exact symptom:**
- `select sum(quantity) from order_line where product_id = 88` exceeds the units that
  ever existed.
- `inventory.available` goes **negative** — the strongest possible evidence, and the
  thing to alert on.
- Fulfilment fails days later; the customer already has a confirmation email.
- **Zero errors, zero exceptions, zero log lines.** Both transactions committed cleanly.
- It reproduces only under concurrency, so it passes every sequential test and every
  manual QA pass.

**Root cause:** the `if` and the assignment are separate operations against a value that
another transaction can change in between. `READ COMMITTED` permits both transactions to
read the same committed value. This is Topic 92's check-then-act race, at the database.

**Fix:** any of the three mechanisms in this document. But note the *diagnostic*: a
`CHECK (available >= 0)` constraint on the column converts a silent oversell into a
loud constraint violation. **Add it regardless of which mechanism you choose.** It costs
nothing, it catches every future code path, and it turns an invisible business loss into
a visible error you can alert on. A database constraint is the only defence that
protects against code you have not written yet.

---

### Trap 2 — the retry inside the transaction: it does nothing

**This is the most important trap in the document.** It looks completely correct.

**Wrong — version A:**

```java
@Transactional
public void placeOrder(PlaceOrderCommand cmd) {
    for (int attempt = 0; attempt < 3; attempt++) {
        try {
            Wallet w = wallets.findByCustomerRef(cmd.customerRef()).orElseThrow();
            w.debit(cmd.totalMinor());
            return;                              // "success"
        } catch (ObjectOptimisticLockingFailureException e) {
            // never reached
        }
    }
}
```

**Wrong — version B:**

```java
@Transactional
@Retryable(retryFor = ObjectOptimisticLockingFailureException.class, maxAttempts = 3)
public void placeOrder(PlaceOrderCommand cmd) { ... }
```

**Exact symptom:**
- The `catch` block never executes. Add a log line in it and it never prints.
- The exception surfaces from the controller, from the transaction proxy, with a stack
  trace that contains none of your retry code.
- With `@Retryable` on the same method, you may instead see a
  `UnexpectedRollbackException` or `TransactionSystemException`, or retries that all
  fail identically and instantly, or — worst — retries that appear to succeed while
  nothing was written.
- Under load the 409 rate is exactly what it was before you added the retry.

**Root cause, in two parts:**

1. **The exception is thrown at flush/commit.** Your method body never triggers it. The
   `@Transactional` proxy commits after `placeOrder` returns, and the version-mismatch
   `UPDATE` runs there. A `try/catch` inside the method is lexically before the thing
   that throws.

   You can force a flush inside the method with `em.flush()`, which *does* move the
   exception inside your `try`. But that does not save version A, because —

2. **Once a transaction has failed, it is doomed.** After an optimistic-lock failure the
   persistence context is in an undefined state (the JPA specification says the
   transaction must be rolled back) and Spring marks it rollback-only. Retrying inside
   it cannot commit. You get `UnexpectedRollbackException` at the end, or a silent
   no-op.

   With `@Retryable` on the *same* method, both advices apply to the same proxied call.
   Whether the retry sits outside or inside the transaction depends on advisor ordering,
   and even when it sits outside, self-invocation and ordering surprises (Topic 40) make
   it fragile. **Do not rely on ordering for correctness.**

**Fix:** the retry goes on a **different bean**, calling the transactional bean through
its proxy, as in Example 2. The unit of retry is **the whole transaction**, including
the commit. Say that sentence in an interview.

Verify the structure holds:

```java
@Test
void retryHappensOutsideTheTransaction() {
    // A wallet whose version is bumped by a competing writer between attempts.
    // Assert the transaction ran more than once by counting a side effect, e.g.
    // an audit row written per attempt, or a Statistics transaction count.
    Statistics s = emf.unwrap(SessionFactory.class).getStatistics();
    s.clear();
    facade.place(command);
    assertThat(s.getTransactionCount()).isGreaterThan(1);
}
```

`getTransactionCount()` proves the transaction *restarted*, which is the property you
actually need. A test that only asserts "no exception escaped" passes for a broken
retry.

---

### Trap 3 — a pessimistic lock held across slow work

**Wrong:**

```java
@Transactional
public void placeOrder(PlaceOrderCommand cmd) {
    Inventory inv = inventory.findForUpdate(cmd.productId()).orElseThrow();  // FOR UPDATE
    inv.reserve(cmd.quantity());

    PaymentResult result = paymentGateway.authorize(cmd);   // 200 ms HTTP call. Or 2 s.
    payments.save(Payment.from(result));
}
```

**Exact symptom — a very specific and recognisable shape:**
- `POST /orders` p99 climbs to whole seconds while p50 stays flat. The distribution goes
  **bimodal**: uncontended requests are fast, contended ones wait a multiple of the
  gateway latency.
- `SELECT * FROM pg_stat_activity WHERE wait_event_type = 'Lock'` shows many sessions
  waiting on the same relation.
- HikariCP's `hikaricp_connections_pending` metric rises; `hikaricp_connections_idle`
  hits zero.
- **Endpoints that touch no inventory at all start failing** with connection-acquisition
  timeouts. The blast radius is the whole service.
- If the gateway degrades from 200 ms to 2 s, the service stops entirely.

**Root cause:** the row lock is held from `SELECT … FOR UPDATE` to `COMMIT`. Everything
between is inside the critical section, including the network call. And because the
transaction pins a pooled connection for the same interval, lock waiters consume the
pool. **Row-lock contention has become connection-pool contention**, which is not
contained to inventory.

**Fix, in order:**

1. **Get the external call out of the transaction.** Authorize first, then open a short
   transaction that takes the lock and writes. This is Topic 55's central rule and it is
   the primary fix, not a refinement.
2. **Take the lock as late as possible** and make it the last thing before commit.
3. **Bound the wait**: `jakarta.persistence.lock.timeout` of 2–3 s so waiters give up and
   return their connection.
4. **Prefer the conditional `UPDATE`**, which has no read-then-write window to hold at
   all.

**The rule to carry:** anything inside a transaction that holds a lock must be
CPU-and-database-only. No HTTP, no message publish that blocks, no file I/O, no
`Thread.sleep`, no retry with backoff.

---

### Trap 4 — `@Version` does nothing for bulk updates, native SQL, or the row you only read

**Wrong (three variants of the same misunderstanding):**

```java
// (a) bulk JPQL update — does not check or bump the version
@Modifying
@Query("update Wallet w set w.balanceMinor = w.balanceMinor - :amt where w.id = :id")
int debit(@Param("amt") long amt, @Param("id") Long id);

// (b) native SQL — same, and it also skips L2 invalidation (Topic 51)
jdbc.update("update wallet set balance_minor = balance_minor - ? where id = ?", amt, id);

// (c) reading a versioned entity and acting on it without writing it
Wallet w = wallets.findById(id).orElseThrow();
if (w.getBalanceMinor() >= amount) {
    payments.save(Payment.authorizedFor(order, amount));   // writes Payment, not Wallet
}
```

**Exact symptom:**
- (a) and (b): the version column stops advancing while balances change. Any *other*
  transaction that read the wallet earlier commits its own update successfully — because
  the version it holds still matches. **Silent lost update, on a balance.**
- (c): no exception ever, and a payment authorized against a balance that another
  transaction already spent. Overdraft beyond the limit. The wallet's version is
  untouched because nothing wrote to the wallet.

**Root cause:** `@Version` is checked and incremented only by Hibernate's **entity-level
`UPDATE`** for that entity. A bulk `UPDATE` bypasses the entity lifecycle entirely.
Variant (c) never writes the entity at all, so there is nothing for the version
predicate to guard — you made a decision based on a read and the read is not protected
by anything.

**Fix:**

- (a)/(b): if the entity is versioned, either write it through the persistence context,
  or include and bump the version explicitly in the bulk statement:
  `set w.balanceMinor = w.balanceMinor - :amt, w.version = w.version + 1 where w.id = :id and w.version = :expected` —
  and check the returned row count yourself. **If you are doing that, you have
  hand-rolled optimistic locking, which is fine as long as you know that is what you
  did.** Make the guard part of the `WHERE` clause and you have the conditional-update
  pattern, which is better.
- (c): use `LockModeType.OPTIMISTIC` to force a version re-check at commit for an entity
  you only read, or `OPTIMISTIC_FORCE_INCREMENT` to bump the parent's version when a
  child changes.

```java
Wallet w = em.find(Wallet.class, id, LockModeType.OPTIMISTIC);
// at commit, Hibernate re-checks the wallet's version even though we never wrote it
```

**`OPTIMISTIC_FORCE_INCREMENT` is the aggregate-consistency tool** and it is worth
naming: adding an `OrderLine` does not change `Order`'s columns, so `Order`'s version
would not move, so a concurrent transaction that read the whole order would not detect
the change. `OPTIMISTIC_FORCE_INCREMENT` on the `Order` makes the aggregate versioned as
a unit. If someone asks "how do you version an aggregate root when only children
change", this is the answer.

---

### Trap 5 — inconsistent lock ordering across a multi-line order

**Wrong:**

```java
for (OrderLine line : command.lines()) {          // in whatever order the client sent
    inventory.tryReserve(line.getProduct().getId(), line.getQuantity());
}
```

**Exact symptom:**
- Intermittent failures with Postgres SQLSTATE `40P01`, message `deadlock detected`.
  Spring surfaces it as `CannotAcquireLockException` (a `DataAccessException`).
- The rate is proportional to the square of concurrency and to how often two orders share
  two products — so it is **rare in staging and common during a promotion**.
- Postgres logs a `DETAIL` naming both processes and both waited-for locks. That log
  entry names the exact rows and is your fastest diagnosis.
- Latency shows a cluster of requests at just over **1 second** — `deadlock_timeout`.
  A tight band of failures at 1 s is a deadlock fingerprint, not a slow query.

**Root cause:** transaction A locks product 88 then 91; transaction B locks 91 then 88.
Each holds what the other needs. Postgres breaks it by killing one after
`deadlock_timeout`.

**Fix — one line, and it eliminates the class:**

```java
List<OrderLine> ordered = command.lines().stream()
        .sorted(Comparator.comparing(l -> l.getProduct().getId()))
        .toList();
```

**A total order on lock acquisition makes deadlock structurally impossible.** Any total
order works as long as every code path uses the same one — sort by primary key, which is
stable, unique and available everywhere.

Then make it a rule rather than a coincidence:

- Every path that locks multiple rows of the same table sorts by id first.
- Consider deduplicating lines by product before reserving, so one order never touches
  the same row twice.
- Retry `CannotAcquireLockException` with jittered backoff as a backstop, at the same
  outer boundary as the optimistic retry — deadlock victims are safe to retry because
  the transaction fully rolled back.
- Alert on the 40P01 rate. A non-zero rate means an unordered path exists somewhere.

This is the same lock-ordering discipline as Topic 96's deadlock avoidance in Java
monitors. The mechanism is a database instead of a JVM; the rule is identical.

---

## Hands-on proof

No output from me. Settings you apply, readings you take, and how to read each outcome.

### Instruments

```yaml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate:
        generate_statistics: true
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.springframework.orm.jpa: DEBUG
    org.springframework.transaction: DEBUG      # transaction begin/commit boundaries
```

`org.springframework.transaction=DEBUG` is the one people forget. It prints where
transactions start and end, which is exactly the information you need to reason about
where the optimistic-lock exception fires.

### Proof 1 — confirm `@Version` is actually in the SQL

Load an `Inventory`, change `available`, commit, and read the SQL log.

**What to look for:** `update inventory set available=?,version=? where id=? and version=?`

| What you see | What it means |
|---|---|
| the trailing `and version=?` is present | `@Version` is working |
| no `version` in the `SET` or `WHERE` | wrong `@Version` import (`org.hibernate.annotations.Version` does not exist — it must be `jakarta.persistence.Version`), or the column is missing from the schema, or the field is `transient` |
| the `UPDATE` lists every column | normal — Hibernate updates all columns by default. `@DynamicUpdate` changes that, at the cost of a statement-cache miss per distinct column set. Do not add it reflexively. |

### Proof 2 — see the pessimistic lock in the SQL, and in Postgres

```java
@Transactional
public void holdLock(Long productId) throws InterruptedException {
    Inventory inv = inventory.findForUpdate(productId).orElseThrow();
    Thread.sleep(30_000);      // TEST ONLY. Never in application code.
}
```

**What to look for in the SQL log:** the `select` should end with `for update` (or a
Postgres variant — read what your dialect actually emits).

While it sleeps, from `psql`:

```sql
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE datname = 'orderflow' AND state <> 'idle';

SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks
WHERE NOT granted OR relation = 'inventory'::regclass;

SELECT pid, pg_blocking_pids(pid), query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

| What you see | What it means |
|---|---|
| a row in `pg_locks` with `mode = 'RowExclusiveLock'` or a tuple lock, `granted = true` | the holder |
| rows with `granted = false` | waiters, blocked on that row |
| `pg_blocking_pids` non-empty | that pid is blocked, and the array names the blockers — **the fastest possible diagnosis** |
| `wait_event_type = 'Lock'` in `pg_stat_activity` | confirmed lock wait, not a slow query |
| nothing at all | the lock was not taken. Check the SQL log for `for update`. |

Learn `pg_blocking_pids(pid)`. It answers "who is blocking whom" in one query and it is
the single most useful thing in this section during a real incident.

### Proof 3 — where exactly does the exception fire?

```java
@Transactional
public void probeWhereItThrows(Long walletId, long amount) {
    Wallet w = wallets.findById(walletId).orElseThrow();
    w.debit(amount);
    System.out.println("A: after debit, before flush");
    em.flush();                                   // force the UPDATE now
    System.out.println("B: after flush");
}
```

Have a second thread bump the wallet's version between the load and the flush.

| What you see | What it means |
|---|---|
| `A` prints, `B` does not, exception propagates | The `UPDATE` ran at `flush()` and failed there. **This is the proof that the failure is at flush, not at the setter.** |
| both `A` and `B` print, exception appears afterwards | flush did not include this entity, or the competing write landed later. |
| Remove the `em.flush()`: `A` and `B` both print, exception still propagates | The `UPDATE` ran at **commit**, after your method returned. **This is the proof that an in-method `try/catch` cannot catch it.** |

Run both variants. The difference between them is Trap 2, demonstrated in ten lines.

---

## Failure drill

**Mandatory.** Master plan, Topic 52: *two concurrent order placements against one unit
of stock. Observe oversell. Fix three ways (`@Version` + retry, `PESSIMISTIC_WRITE`,
atomic conditional `UPDATE`) and compare throughput under Topic 65's load generator.*

### Setup

Real Postgres via Testcontainers (Topic 61). **This drill is meaningless on H2** — H2's
locking, its `FOR UPDATE` semantics and its deadlock detection all differ from Postgres.

```java
@SpringBootTest
@Testcontainers
class OversellDrillTest {

    @Container @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17");

    @Autowired JdbcTemplate jdbc;
    @Autowired PlacementUnderTest placement;      // swap the implementation per phase

    Long productId;

    @BeforeEach
    void seed() {
        productId = seedProductWithStock(1);      // exactly one unit
    }

    record Outcome(int accepted, int rejected, int lockFailures, int finalStock) {}

    Outcome race(int threads) throws Exception {
        var start = new CountDownLatch(1);
        var done  = new CountDownLatch(threads);
        var accepted = new AtomicInteger();
        var rejected = new AtomicInteger();
        var lockFail = new AtomicInteger();

        try (var pool = Executors.newFixedThreadPool(threads)) {
            for (int i = 0; i < threads; i++) {
                pool.submit(() -> {
                    try {
                        start.await();
                        placement.place(productId, 1);
                        accepted.incrementAndGet();
                    } catch (InsufficientStockException e) {
                        rejected.incrementAndGet();
                    } catch (OptimisticLockingFailureException | CannotAcquireLockException e) {
                        lockFail.incrementAndGet();
                    } catch (Exception e) {
                        System.out.println("unexpected: " + e);
                    } finally { done.countDown(); }
                });
            }
            start.countDown();
            done.await(30, TimeUnit.SECONDS);
        }

        Integer stock = jdbc.queryForObject(
            "select available from inventory where product_id = ?", Integer.class, productId);
        return new Outcome(accepted.get(), rejected.get(), lockFail.get(), stock);
    }
}
```

`CountDownLatch` gives a much tighter start than a barrier at higher thread counts. Run
each phase at **2, 8 and 64 threads**, and run each **20 times**. A race that
reproduces 3 times in 20 is fully reproduced; "it passed" is not a result.

### Phase 0 — reproduce the oversell

Naive check-then-act, no locking.

**What to capture:** `accepted`, `rejected`, `finalStock`, over 20 runs.

| What you see | What it means |
|---|---|
| `accepted > 1`, `finalStock ≤ 0` | **Oversell reproduced.** The drill has started correctly. |
| `accepted = 1` in all 20 runs at 64 threads | The threads are not racing, or something is protecting the row. Check for a unique constraint, a `CHECK`, or an accidental `@Version`. |
| `finalStock` negative | The strongest evidence, and the reason to add `CHECK (available >= 0)` in production. |

**Write down the invariant you just violated**, in one sentence:
*units accepted across all orders must never exceed units ever stocked.* You are about
to fix it three ways, and this sentence is what all three must satisfy.

### Phase 1 — `@Version` plus a retry outside the transaction

Add `@Version` to `Inventory`; put `@Retryable` on the outer facade per Example 2.

**What to capture:** `accepted`, `rejected`, `lockFailures`, `finalStock`, plus
`Statistics.getTransactionCount()` (which reveals the retry amplification) and
`getOptimisticFailureCount()`.

| What you see | What it means |
|---|---|
| `accepted = 1`, `finalStock = 0` | Correct. |
| `transactionCount` ≫ threads | Retry amplification. At 64 threads each conflict replays a full transaction; this is the cost you are measuring. |
| `lockFailures > 0` | The retry budget was exhausted. That is a *correct* outcome, surfaced as HTTP 409 — but note the rate, it is the user-visible cost of this mechanism. |
| `accepted > 1` | `@Version` is not in the SQL. Go back to Proof 1. |

### Phase 2 — `PESSIMISTIC_WRITE`

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
@Query("select i from Inventory i where i.product.id = :productId")
Optional<Inventory> findForUpdate(@Param("productId") Long productId);
```

**What to capture:** the same four, plus — and this is the interesting one — the **wall
time of the whole race** at 64 threads, and a `pg_stat_activity` sample taken mid-race
from a separate connection.

| What you see | What it means |
|---|---|
| `accepted = 1`, `lockFailures = 0` | Correct, and no retries were needed — writers queued instead. |
| total wall time ≈ threads × criticalSection | **Serialisation, made visible.** This is the number to compare against Phase 3. |
| `lockFailures > 0` with a timeout message | Waiters exceeded the 3 s budget. Expected at high thread counts; it is the pressure valve doing its job. |
| the pool metrics show zero idle connections mid-race | The systemic cost. Note it — it is Trap 3 in miniature. |

### Phase 3 — the atomic conditional `UPDATE`

```java
@Modifying
@Query("update Inventory i set i.available = i.available - :qty " +
       "where i.product.id = :pid and i.available >= :qty")
int tryReserve(@Param("pid") Long productId, @Param("qty") int qty);
```

**What to capture:** the same, plus statement count per attempt.

| What you see | What it means |
|---|---|
| `accepted = 1`, `rejected = threads - 1`, `finalStock = 0`, `lockFailures = 0` | Correct, with **no retries and no exceptions** — the losers got a business answer (`0 rows`), not a technical failure. |
| statements per attempt = 1 | The whole mechanism, in one round trip. |
| total wall time noticeably below Phase 2 | Expected: the critical section is one statement instead of a method body. |
| `accepted > 1` | You are on `REPEATABLE READ` and something swallowed a serialization failure, or the `WHERE` guard is missing. Check your isolation level (Topic 55). |

### Phase 4 — throughput comparison under real load

Run all three against Topic 65's k6/Gatling setup: open-model arrival rate, the realistic
dataset, 60 seconds on one hot product with 500 units.

**Record, per mechanism:**

| Metric | Where from |
|---|---|
| successful orders / second | load generator |
| p50 / p95 / p99 of `POST /orders` | load generator |
| 409 rate (contention rejections) | load generator |
| `getOptimisticFailureCount()`, `getTransactionCount()` | Hibernate `Statistics` |
| `hikaricp_connections_pending`, `_idle` | Micrometer (Topic 118) |
| Postgres deadlocks, lock waits | `pg_stat_database.deadlocks`, `pg_stat_activity` |
| oversell count | **must be 0 for all three** |

**Expected shapes — predict them before you run, then check yourself:**

- **`@Version` + retry:** best latency at low contention, degrading as concurrency
  rises; `transactionCount` climbing faster than throughput; a rising 409 rate. The
  failure shape is livelock-ish: more load, less useful work.
- **`PESSIMISTIC_WRITE`:** flat, predictable throughput bounded by
  `1 / criticalSection`; p99 rises with queue depth; connections pending rises;
  **watch for unrelated endpoints degrading** — that is the systemic cost and it is the
  finding that matters.
- **Conditional `UPDATE`:** highest throughput, lowest p99, no retries, no lock-wait
  metric of note.

**Be honest about the result.** For a simple counter decrement the conditional `UPDATE`
usually wins on every axis, and if your numbers say otherwise you have found something
worth investigating rather than something to explain away. Report what you measured,
including the case where two mechanisms are indistinguishable — "within noise, here are
the numbers" is a better answer than a confident wrong one.

### Phase 5 — the regression barrier

```java
@RepeatedTest(20)
@DisplayName("concurrent placements never oversell")
void neverOversells() throws Exception {
    Outcome o = race(64);
    assertThat(o.finalStock()).isGreaterThanOrEqualTo(0);
    assertThat(o.accepted()).isEqualTo(1);
}
```

Plus the constraint that protects against code you have not written:

```sql
ALTER TABLE inventory ADD CONSTRAINT inventory_available_non_negative
  CHECK (available >= 0);
```

**And prove the test works** by reverting to the naive implementation on a scratch commit
and confirming it goes red. A concurrency test you have never seen fail is not evidence.

### What the fixes prove

Not that the code is faster. They prove an **invariant holds under concurrency**: units
accepted never exceeds units stocked, verified by 64 threads racing, 20 times, against
real Postgres. That is a correctness claim, it is reproducible, and it is the only kind
of claim worth making about a concurrency fix.

The throughput comparison is a separate, weaker claim — a measurement under one load
shape on one machine. Report it as such.

---

## Measurement

### Hibernate counters

```java
Statistics s = emf.unwrap(SessionFactory.class).getStatistics();
```

| Counter | Use it to |
|---|---|
| `getOptimisticFailureCount()` | **the contention rate.** Non-zero in production means `@Version` is doing work; a rising trend means contention is growing |
| `getTransactionCount()` / `getSuccessfulTransactionCount()` | the gap is rollbacks. Divided by request count, it is your retry amplification |
| `getEntityUpdateCount()` | updates that actually reached the database |
| `getPrepareStatementCount()` | statements per placement — 1 for the conditional update, 2 for the others |
| `getFlushCount()` | catches an unexpected mid-transaction flush moving where the exception fires |

`getOptimisticFailureCount()` is the counter to expose as a Micrometer gauge (Topic 118).
It is a leading indicator: it rises before the 409 rate does, because retries absorb the
early conflicts.

### Postgres instrumentation

```sql
-- Who is blocked, and by whom. The single most useful query in an incident.
SELECT pid, pg_blocking_pids(pid) AS blocked_by, wait_event_type, wait_event,
       now() - query_start AS waiting_for, left(query, 120)
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- Lock inventory, including ungranted (waiting) locks.
SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks WHERE relation = 'inventory'::regclass;

-- Deadlocks and conflicts since stats reset. Trend this.
SELECT datname, deadlocks, conflicts, xact_commit, xact_rollback
FROM pg_stat_database WHERE datname = 'orderflow';

-- Statement-level view: is the conditional UPDATE actually one statement?
SELECT calls, rows, mean_exec_time, left(query, 120)
FROM pg_stat_statements WHERE query ILIKE '%inventory%' ORDER BY calls DESC;
```

**How to read `pg_stat_statements` for the conditional update:** `calls` is the number of
attempts and `rows` is the number that succeeded. `rows / calls` is your success ratio —
the contention rate, measured by the database, with no application instrumentation at
all. That is a genuinely elegant measurement and worth remembering.

Trend `pg_stat_database.deadlocks`. Any non-zero rate means an unordered lock path
exists.

### Why `System.nanoTime()` is the wrong instrument here

Everything from Topic 50 applies — no warmup, one sample, dead-code elimination, cold
JIT — plus three that are specific to concurrency:

1. **You are timing a scheduler, not your code.** Thread start-up, `CountDownLatch`
   wake-up ordering, OS scheduling and core count dominate a short concurrent
   measurement. Two runs on the same machine can differ by an order of magnitude for
   reasons that have nothing to do with your locking strategy.
2. **The contention level is not controlled.** Throughput under contention depends on
   arrival *distribution*, not just concurrency count. A burst of 64 simultaneous
   threads and a steady 64-in-flight open model produce completely different conflict
   rates. Your `nanoTime` number describes one arrival pattern you did not choose
   deliberately.
3. **Coordinated omission.** A fixed thread pool that waits for each request before
   issuing the next *cannot* observe the latency of the requests it did not send while
   blocked. It systematically under-reports the tail — often by an order of magnitude —
   which is exactly the region where locking shows up.

**The correct instruments:**
- **Counts and invariants** for correctness. Exact, deterministic, CI-safe.
- **Topic 77 (JMH)** for anything below the service boundary, with proper forks and
  warmup — though note that JMH is a poor fit for measuring database contention, which
  is dominated by I/O and by the database's own scheduler.
- **Topic 65 (the load baseline)** for the real answer: open-model arrival rate,
  realistic dataset, p50/p95/p99/p999, coordinated omission accounted for. This is
  where the throughput comparison actually belongs.
- **Topic 118 (Micrometer)** for `getOptimisticFailureCount()`, the 409 rate and the
  pool gauges as continuous production signals.

---

## Practice exercises

### 1 — easy: see all three mechanisms in the SQL

With `logging.level.org.hibernate.SQL=DEBUG`, produce and record the exact statement
text for each:

- **(a)** load an `Inventory` with `@Version`, mutate `available`, commit
- **(b)** `em.find(Inventory.class, id, LockModeType.PESSIMISTIC_WRITE)`
- **(c)** `em.find(Inventory.class, id, LockModeType.PESSIMISTIC_READ)`
- **(d)** `em.find(Inventory.class, id, LockModeType.OPTIMISTIC_FORCE_INCREMENT)` with
  no mutation, then commit
- **(e)** the `@Modifying` conditional `UPDATE`

Then answer: **(d) issues an `UPDATE` even though you changed nothing. Why is that
useful?** Give a concrete `orderflow` case where you would want it.

### 2 — medium: combines Topics 13, 40, 48 and 54

**(a)** Put `@Retryable` and `@Transactional` on the **same** method and reproduce the
failure from Trap 2. Capture the exact exception and the stack trace. Then use Topic
40's technique — print `service.getClass().getName()` — to show which proxies wrapped
the bean and in what order. Explain the failure in terms of proxy ordering.

**(b)** Move `@Retryable` to a separate bean. Prove the transaction actually restarted by
asserting on `Statistics.getTransactionCount()`, not merely that no exception escaped.
State in one sentence why the weaker assertion passes for a broken retry.

**(c)** Give `Inventory` an `equals`/`hashCode` that includes the `@Version` field.
Load one, put it in a `HashSet`, mutate it, flush, and call `contains`. Explain the
result using Topic 13's contract, then state the rule for which fields may participate
in an entity's `hashCode`.

**(d)** Using Topic 48's dirty checking: load a versioned `Inventory`, call
`setAvailable` with **the value it already has**, and commit. Does the version
increment? Predict first, then check the SQL log. Explain the result in terms of the
snapshot diff, and say what it implies for a "touch to bump the version" idiom.

### 3 — hard: production simulation, advancing the spine

Bring `orderflow` order placement to a defensible concurrency design.

**Part A — the flash sale.** Seed one hot product with 500 units. Using Topic 65's load
generator with an **open-model** arrival rate, drive 3,000 placement attempts over 60
seconds, peaking at ~120 concurrent. Implement all three mechanisms and record the full
metric set from Failure drill Phase 4. **Oversell must be zero in all three or the
result is invalid.**

**Part B — the wallet, which is a different shape.** Mix in wallet debits: 95% of
requests hit distinct consumer wallets (no contention), 5% hit one of three corporate
wallets (up to 15 concurrent). Measure each mechanism against *this* distribution and
explain why the ranking differs from Part A. Write the paragraph you would put in an
ADR justifying **two different mechanisms in the same transaction.**

**Part C — the deadlock.** Build multi-line orders drawing 3 lines from a pool of 5 hot
products, with lines in random order. Run 64 concurrent. Capture the Postgres deadlock
log entries and the SQLSTATE. Record the deadlock rate and confirm the latency cluster
sits just above `deadlock_timeout`. Then sort by product id and confirm the rate goes to
exactly zero. Report both numbers.

**Part D — the systemic failure.** Add a 300 ms stub payment-gateway call *inside* the
transaction, after taking a `PESSIMISTIC_WRITE` lock. Run the Part A load. Record: p99
for `POST /orders`, p99 for `GET /products` (which touches no inventory at all),
`hikaricp_connections_pending`, and the number of `pg_stat_activity` rows with
`wait_event_type = 'Lock'`. **Write the incident summary you would post**, naming the
causal chain from row lock to pool exhaustion to unrelated-endpoint failure. Then move
the gateway call outside the transaction and re-measure everything.

**Part E — the recovery path.** A reservation succeeds, then payment authorization
fails. Implement compensation — release the reserved units — and prove it is correct
under concurrency: run 64 threads where half fail at payment, and assert final stock
exactly equals initial stock minus successful orders. Note where this becomes Topic
117's saga problem, and where an idempotency key (Topic 116) is required so a retried
release does not double-credit.

**Part F — argue against yourself.** You chose the conditional `UPDATE` for inventory.
Name two concrete future `orderflow` requirements that would make that the wrong choice
and force you back to an entity with `@Version`. (Consider: per-warehouse allocation
with business rules; an audit trail of every stock movement; a reservation that expires.)
State what you would need to see to make the switch.

---

## Interview questions

### Q1 — "Two customers order the last unit at the same time. How do you make sure only one succeeds?"

**Mid-level answer:** "Add `@Version` to the inventory entity. Hibernate will throw an
`OptimisticLockException` for the second one."

**Senior answer:** "Three options, and I'd pick from the contention shape.

**Optimistic `@Version`**: Hibernate appends `and version = ?` to the `UPDATE` and bumps
it; zero rows affected means someone else won. Cheap when conflicts are rare — no lock,
no waiting. But it fails at *commit*, so the retry has to wrap the whole transaction on
a different bean, and under real contention retry amplification means throughput can
fall as concurrency rises.

**`PESSIMISTIC_WRITE`**: `SELECT … FOR UPDATE`, writers serialise at the database, no
retries. Predictable throughput bounded by one over the critical-section duration. The
cost is that the lock is held for the transaction's whole life and the transaction pins
a pooled connection for the same interval, so lock contention becomes pool contention
and takes out unrelated endpoints.

**A single atomic conditional `UPDATE`**: `update inventory set available = available - ?
where product_id = ? and available >= ?`, and check the affected-row count. One
statement, one round trip, no entity loaded, no retry, and the row lock is held for
microseconds. For a flash sale on one hot row, this wins on every axis and it is what
I'd ship.

The honest trade on the third is that it bypasses the entity model — no `@Version`
increment, no domain method holding the rule, no L2 invalidation if you write it as
native SQL. I'd enforce with an ArchUnit rule that `available` is only ever written
through that one repository method.

And regardless of mechanism: `CHECK (available >= 0)` on the column. That protects
against every code path anyone writes later, which no application-level mechanism does."

**What separates them:** three mechanisms with cost profiles rather than one; naming the
retry-placement problem unprompted; the connection-pool coupling; choosing the
non-obvious answer with a stated trade-off; and the database constraint as a defence
against future code.

**Follow-up:** *"Under what contention level would you switch from the conditional
update back to `@Version`?"* They are testing whether the answer is a reflex or a
judgment. Good answer: contention isn't the trigger — *domain logic* is. When the
decrement grows business rules (per-warehouse allocation, expiring reservations, an
audit trail per movement), the entity model earns its cost back.

---

### Q2 — "Where do you put the retry for an `OptimisticLockException`?"

**Mid-level answer:** "Wrap the operation in a try/catch and retry a few times, or put
`@Retryable` on the service method."

**Senior answer:** "Outside the transaction, on a different bean — and both halves of
that matter.

The exception is thrown at flush or commit. In a `@Transactional` method, commit happens
in the proxy *after* my method returns, so a `try/catch` inside the method is lexically
before the thing that throws. It never fires. And even if I force the flush inside my
`try`, the transaction is already doomed — after an optimistic-lock failure JPA requires
rollback and Spring marks it rollback-only, so retrying inside cannot commit. You get
`UnexpectedRollbackException` or a silent no-op.

Putting `@Retryable` on the same method as `@Transactional` makes the correctness depend
on advisor ordering, which I don't want to rely on. So: a facade bean with `@Retryable`
calling a service bean with `@Transactional`, through the proxy. The unit of retry is
the whole transaction.

Details that matter: backoff with jitter, or N conflicting transactions retry in
lockstep and collide again. A bounded budget — three or four attempts — with `@Recover`
converting exhaustion into a domain exception mapped to 409, not a stack trace. And only
retry idempotent work: if the transaction sent an email or published an event before
failing, the retry does it twice, which is why the event belongs in an outbox row in the
same transaction.

I'd prove the retry actually works by asserting `Statistics.getTransactionCount()` is
greater than one, not just that no exception escaped — the weaker assertion passes for a
retry that never runs."

**What separates them:** knowing the exception fires at commit, knowing the transaction
is doomed even if you catch it, refusing to depend on advisor ordering, and the
assertion that distinguishes a working retry from a decorative one.

**Follow-up:** *"How many retries, and what backoff?"* They want a budget, jitter, and
an awareness that retries add load to an already-contended resource — Topic 111. A
number without a reason is the wrong answer.

---

### Q3 — "You added `PESSIMISTIC_WRITE` and now unrelated endpoints are timing out. Explain."

**Mid-level answer:** "The lock is blocking other requests."

**Senior answer:** "The lock is blocking other *writers to that row*, which is what I
asked for. The outage is second-order.

A JPA transaction pins a HikariCP connection from begin to commit. A pessimistic lock is
held for exactly the same interval. So every request waiting on that row is holding a
pool connection while it waits. At pool size 20, twenty waiters means zero connections
for anything else — catalogue reads, health checks, everything. Kubernetes then fails
the readiness probe and restarts the pod, the in-flight transactions roll back, and the
retry storm compounds it.

The proximate cause is almost always something slow inside the critical section. If
there's an HTTP call between the `SELECT … FOR UPDATE` and the commit, the lock hold
time is the downstream's latency, and a 200 ms gateway blip becomes a full outage. That's
the same failure as the transaction-plus-HTTP-call problem generally.

Diagnosis: `pg_stat_activity` filtered on `wait_event_type = 'Lock'`, and
`pg_blocking_pids(pid)` to see who blocks whom — that names the row and the holder in one
query. On the JVM side, `hikaricp_connections_pending` rising with `_idle` at zero
confirms it.

Fixes in order: get the external call out of the transaction; take the lock as late as
possible; set `jakarta.persistence.lock.timeout` so waiters give up and return their
connection rather than queuing forever; and prefer a conditional `UPDATE` so there's no
read-then-write window to hold at all."

**What separates them:** the causal chain from row lock to connection to pool to
unrelated endpoints; naming the specific diagnostic queries; and treating "no external
calls in a locked transaction" as the primary fix rather than a tuning tweak.

**Follow-up:** *"Would raising the pool size fix it?"* No — it moves the contention into
Postgres, which is process-per-connection, and makes latency worse for everyone. That is
Topic 109's answer, and volunteering it is the signal they want.

---

### Q4 — "Does `@Version` protect a bulk update? What about an entity you only read?"

**Mid-level answer:** "`@Version` protects the entity whenever it's updated."

**Senior answer:** "Neither case is protected, and both are silent.

A bulk JPQL or native `UPDATE` bypasses the entity lifecycle entirely — no dirty check,
no version predicate, no version increment. So the column stops advancing while the data
changes, and any transaction holding an older snapshot commits its own update
successfully because the version it holds still matches. That's a lost update on data
you thought was protected. If I must bulk-update a versioned entity, I put the version
in the `SET` and the `WHERE` myself and check the row count — at which point I've
hand-rolled optimistic locking, which is fine as long as I know that's what I did.

The read-only case is subtler and more dangerous. If I load a `Wallet`, check the
balance, and then write a `Payment` — not the wallet — the wallet's version is never
consulted, so nothing detects that another transaction spent the balance between my read
and my commit. I made a decision on a read and the read wasn't guarded. The fix is
`LockModeType.OPTIMISTIC`, which forces a version re-check at commit for an entity I only
read.

The related one is `OPTIMISTIC_FORCE_INCREMENT`, which bumps the version even with no
change. That's how you version an *aggregate*: adding an `OrderLine` doesn't change any
`Order` column, so `Order`'s version wouldn't move and a concurrent reader of the whole
order wouldn't detect the change. Forcing the increment makes the aggregate versioned as
a unit."

**What separates them:** knowing the read-then-decide hole exists at all, naming
`OPTIMISTIC` and `OPTIMISTIC_FORCE_INCREMENT` with the aggregate use case, and
recognising the hand-rolled bulk version as legitimate-if-conscious.

**Follow-up:** *"What else does a bulk update bypass?"* The second-level cache (Topic
51) — a JPQL bulk update invalidates the region, a native one does not unless you
declare the query space. Also entity listeners, `@PreUpdate`, and auditing. Bulk
operations bypass a lot, and the pattern is worth stating as a pattern.

---

### Q5 — "Optimistic or pessimistic? Give me your decision rule."

**Mid-level answer:** "Optimistic when conflicts are rare, pessimistic when they're
common."

**Senior answer:** "That's the right first cut but it's not sufficient, because the cost
of a conflict matters as much as its probability.

I'd ask four questions.

**How likely is a conflict?** Rare — optimistic, because you pay nothing when there's no
conflict. Common — optimistic degrades, because every conflict costs a full transaction
replay and the retries pile onto the same contended row. Throughput can *fall* as load
rises, which is the worst shape a system can have.

**What does a retry cost?** If the transaction is a pure read-modify-write of one row,
retrying is cheap. If it sent an email, called a payment gateway, or published an event,
the retry is either impossible or requires idempotency work. Expensive retries push me
toward pessimistic.

**How long is the critical section?** Pessimistic caps throughput at one over the
critical-section duration and holds a pooled connection the whole time. If there's
anything slow in there, pessimistic is not an option until that's fixed. This is usually
the question that decides it.

**Can I avoid loading the entity at all?** If the operation is expressible as a single
conditional `UPDATE` — decrement if sufficient, transition if in state X — that beats
both. One statement, no retry, the shortest possible lock hold, and the affected-row
count is the answer.

For our inventory decrement that fourth question wins. For a wallet debit with an
overdraft policy and a ledger entry, the logic belongs in the domain model, contention is
low for most wallets, so optimistic plus a bounded retry. Two mechanisms in the same
transaction, each chosen from its own contention shape — and I'd write that in an ADR so
the next person doesn't 'unify' it."

**What separates them:** four dimensions instead of one; naming retry *cost* separately
from conflict *probability*; the conditional update as a third option; and giving a
concrete split decision for a real system rather than a general principle.

**Follow-up:** *"What if the operation spans two services?"* Now it is a saga with
compensations (Topic 117), no shared transaction, and the reservation becomes a
first-class state with a timeout — and idempotency keys (Topic 116) become mandatory
because retries cross a network boundary.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Optimistic locking fails at commit, which is *after* your method returns. Design the
   alternative: what would Hibernate have to do to fail earlier, and what would that
   cost? Would you want that behaviour as a default?

2. `PESSIMISTIC_WRITE` holds the row lock until commit, not until you finish reading.
   Give the concrete reason a database cannot safely release it earlier. Then say what
   changes about your code layout once you accept that constraint.

3. Postgres re-evaluates a blocked `UPDATE`'s `WHERE` clause against the newly committed
   row under `READ COMMITTED`, but raises a serialization error under `REPEATABLE READ`.
   Explain why the second behaviour is *more* correct, and why the first is nonetheless
   what you want for the inventory decrement.

4. The atomic conditional `UPDATE` is the best mechanism for a bare decrement and gets
   worse as the operation grows business logic. Identify the exact point at which it
   stops being right — state it as a property of the operation, not as a rule of thumb.

5. You add a `CHECK (available >= 0)` constraint. Under which of the three mechanisms
   would that constraint ever actually fire, and what does that tell you about what the
   constraint is really defending against?

6. Retry with jitter fixes retry storms. Explain, without using the word "jitter", what
   goes wrong when N conflicting transactions retry after identical delays — then say
   whether the same argument applies at N = 3.

7. `@Version` on `Inventory` and a conditional `UPDATE` on the same column are mutually
   incompatible in one important way. Name it. Then say what would happen if a colleague
   added `@Version` to `Inventory` six months after you shipped the conditional update,
   and what would tell you it had happened.

---

## Quick reference card

### Optimistic

```java
@Entity
public class Wallet {
    @Version private long version;       // jakarta.persistence.Version. Use long, not a timestamp.
}
```

```
update wallet set balance_minor=?, version=? where id=? and version=?
-- 0 rows affected  ->  someone else won  ->  exception at FLUSH/COMMIT
```

| Layer | Exception |
|---|---|
| Hibernate | `StaleObjectStateException` |
| JPA | `jakarta.persistence.OptimisticLockException` |
| Spring | `ObjectOptimisticLockingFailureException` — **catch and retry on this one** |

### Lock modes

```java
em.find(Inventory.class, id, LockModeType.PESSIMISTIC_WRITE);

@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
Optional<Inventory> findForUpdate(Long productId);
```

| Mode | Effect |
|---|---|
| `PESSIMISTIC_WRITE` | `for update` — exclusive row lock |
| `PESSIMISTIC_READ` | `for share` — shared row lock |
| `PESSIMISTIC_FORCE_INCREMENT` | lock **and** bump the version |
| `OPTIMISTIC` | re-check the version at commit for an entity you only **read** |
| `OPTIMISTIC_FORCE_INCREMENT` | bump the version with no change — versions an aggregate |

Timeout values: `3000` = 3 s · `0` = `NOWAIT` · `-2` (`LockOptions.SKIP_LOCKED`) = `SKIP LOCKED`.
**Always set one.** Fallback: `SET LOCAL lock_timeout = '3s'`.

### Atomic conditional update

```java
@Modifying
@Query("update Inventory i set i.available = i.available - :qty " +
       "where i.product.id = :pid and i.available >= :qty")
int tryReserve(@Param("pid") Long productId, @Param("qty") int qty);
// returns 1 = reserved, 0 = insufficient. No retry. No lock held across your logic.
```

### Retry — the placement is the whole point

```java
@Service class Facade {                       // NOT @Transactional
    @Retryable(retryFor = ObjectOptimisticLockingFailureException.class,
               maxAttempts = 4,
               backoff = @Backoff(delay = 25, multiplier = 2.0, random = true))
    Result run(Cmd c) { return service.doIt(c); }   // different bean -> real proxy call

    @Recover Result giveUp(ObjectOptimisticLockingFailureException e, Cmd c) {
        throw new ContentionException(e);           // -> HTTP 409 via Topic 46
    }
}

@Service class Service {
    @Transactional Result doIt(Cmd c) { ... }       // NO @Retryable here
}
```

### Diagnose

```sql
SELECT pid, pg_blocking_pids(pid), wait_event_type, left(query,120)
FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;

SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks WHERE relation = 'inventory'::regclass;

SELECT deadlocks, xact_rollback FROM pg_stat_database WHERE datname = 'orderflow';
```

```java
stats.getOptimisticFailureCount();   // contention rate — expose as a gauge
stats.getTransactionCount();         // retry amplification
```

### Gotchas checklist

- [ ] The optimistic failure fires at **flush/commit**, after your method returns.
- [ ] Retry goes on a **different bean**, outside the transaction. Whole transaction, retried whole.
- [ ] Backoff needs jitter, even at N = 3.
- [ ] Bulk and native `UPDATE`s bypass `@Version` entirely — and L2 invalidation.
- [ ] Reading a versioned entity and writing a *different* one is unprotected. Use `OPTIMISTIC`.
- [ ] Pessimistic lock is held to **commit**. No HTTP, no I/O, nothing slow inside.
- [ ] Always set a lock timeout. "Wait forever" plus a pooled connection is an outage.
- [ ] Sort by id before locking multiple rows. Deadlock becomes structurally impossible.
- [ ] A 1-second latency cluster is a `deadlock_timeout` fingerprint.
- [ ] Use `long` for `@Version`, never a timestamp.
- [ ] Add `CHECK (available >= 0)`. It defends against code that does not exist yet.
- [ ] Test on real Postgres. H2 has different locking, and no `SKIP LOCKED`.

---

## When would I use this at work?

**1. Reviewing any read-modify-write on a shared row.**
The pattern is recognisable at a glance: a `findBy…`, an `if`, and a setter. The review
comment is *"two requests, same row, same millisecond — walk me through it."* Half the
time the author has not considered it, and the fix is a one-line conditional `UPDATE`.
This is the highest-frequency application of this topic.

**2. Diagnosing a bimodal latency distribution.**
p50 flat, p99 in whole seconds, and a suspicious cluster just above one second. That
shape is lock contention, and the one-second cluster is `deadlock_timeout`.
`pg_blocking_pids` names the rows and the holders in one query, and you go from "the
database is slow" to "these two code paths lock two products in different orders" in
about two minutes.

**3. Designing a feature before it is built.**
Product wants a limited-edition drop: 500 units, a scheduled start time, marketing
driving a spike. Before any code exists you can say: this is one hot row, optimistic
locking will livelock, pessimistic will exhaust the pool, so it is a conditional
`UPDATE` with a `CHECK` constraint and a 409 contract on the API. Then you specify the
load test that proves it. That conversation, held in planning rather than during the
incident, is what senior means here.

---

## Connected topics

**Prerequisites:**
- **13 — equals/hashCode contract**: the `@Version` field must not participate in an
  entity's `hashCode`, for the same reason a generated id must not.
- **40 — Proxying and self-invocation**: why `@Retryable` and `@Transactional` on the
  same method is fragile, and why the retry needs a separate bean.
- **48 — Persistence context and dirty checking**: the flush that generates the
  versioned `UPDATE`, and why a no-op setter does not bump the version.
- **49 — Lazy proxies**: loading `Inventory` through a lazy `@OneToOne` from `Product`
  changes when the lock is actually taken.
- **50 — N+1**: statement counts are how you verify the conditional update is genuinely
  one statement.
- **51 — Caching**: L2's `READ_WRITE` soft lock and `@Version` solve overlapping
  problems at different layers, and a bulk update bypasses both.

**This unlocks / is used by:**
- **54 — `@Transactional` I**: propagation decides which transaction the lock belongs
  to; `REQUIRES_NEW` takes a second connection while holding the first, which is a
  deadlock generator on top of everything here.
- **55 — Isolation and the connection pool**: the isolation level decides whether a
  blocked update re-evaluates or errors, and the pool is the mechanism by which lock
  contention becomes a service-wide outage.
- **61 — Testcontainers**: this entire topic is untestable on H2. Different locking,
  different deadlock detection, no `SKIP LOCKED`.
- **65 — GATE, load baseline**: the throughput comparison between the three mechanisms
  is a load-test result with an open-model arrival rate, not a stopwatch.
- **77 — JMH**: why the timing you were about to write is not a measurement — and why
  even JMH is the wrong tool for database contention.
- **92 — Check-then-act races**: the identical bug in a `ConcurrentHashMap`, with the
  identical fix pattern (`compute`/`merge` instead of `containsKey` + `put`).
- **96 — Deadlock avoidance**: lock ordering, the same discipline applied to JVM
  monitors instead of database rows.
- **109 — HikariCP**: the arithmetic that turns a lock wait into pool exhaustion.
- **111 — Resilience4j**: retry budgets, jitter and why retries amplify an overload.
- **116 — Idempotency**: what a retried transaction must not do twice, and why the event
  publish belongs in an outbox.
- **117 — Sagas**: what happens to all of this when the reservation and the payment live
  in different services and there is no shared transaction.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0,
`jakarta.persistence.*`, Postgres. The SQL each `LockModeType` produces is decided by the
dialect and has varied across Hibernate versions — read it off
`logging.level.org.hibernate.SQL=DEBUG` on your version rather than trusting any table,
including the one above. Isolation behaviour described here is Postgres's; other engines
differ in ways that change the conclusions.*
