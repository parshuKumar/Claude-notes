# 55 — `@Transactional` II — Isolation and the Transaction-to-Connection-Pool Interaction

## Phase: 5 — Spring Boot & Persistence
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: `orderflow` under a slow downstream — correct isolation on the contended writes, and transaction boundaries that survive a bad afternoon at the payment gateway

---

## Mechanical statement

> A transaction binds **one connection** to the current thread for its entire lifetime,
> from first statement to commit. Whatever else that thread does in between — including
> an HTTP call — happens while that connection is unavailable to everyone else.

That is the payload of this topic. You already know isolation levels from SQL. What you
do not yet know is the Java-specific fact that a transaction is also a **capacity
reservation**, and that reservation is held by a `ThreadLocal` binding you cannot see.

Two corollaries:

1. Transaction duration is not a latency concern. It is a **concurrency limit**. With a
   pool of `P` connections and an average transaction duration of `D` seconds, the
   service cannot exceed `P / D` transactions per second. That is Little's Law, and it
   is a hard ceiling.
2. When the pool is exhausted, **every** endpoint fails — including endpoints that touch
   no database — because the request threads that would serve them are parked waiting
   for a connection.

---

## The bridge from what you know

### Isolation levels: you already know these

You know isolation levels from Postgres. This doc is not going to re-teach the anomaly
table as theory. What it will do is nail down two things that people who "know isolation
levels" routinely still get wrong.

**(a) The Postgres reality, not the ANSI theory.**

The ANSI SQL standard defines four levels by which anomalies they forbid. Postgres does
not implement them that way — it implements MVCC, and the levels fall out of *which
snapshot you read*. Concretely:

| You ask for | Postgres gives you | The practical consequence |
|---|---|---|
| `READ_UNCOMMITTED` | **`READ COMMITTED`**. Silently. | There are **no dirty reads in Postgres at any level**. Asking for `READ_UNCOMMITTED` is a no-op, not a performance win. |
| `READ_COMMITTED` (default) | A **new snapshot per statement** | Two `SELECT`s in one transaction can return different data. Non-repeatable reads and phantoms are both possible. |
| `REPEATABLE_READ` | **Snapshot isolation** — one snapshot for the whole transaction | Stronger than ANSI requires: phantoms are also prevented. But a write conflict aborts with `40001 could not serialize access due to concurrent update`, which you must catch and retry. |
| `SERIALIZABLE` | Snapshot isolation **plus SSI** (Serializable Snapshot Isolation) with predicate locks | True serializability. Transactions can fail with `40001 could not serialize access due to read/write dependencies among transactions` even when no row was modified twice. **You must have a retry loop.** |

The two lines to burn in:

- **Postgres has no dirty reads.** Not at any level. If someone claims
  `READ_UNCOMMITTED` will speed things up, they are describing a different database.
- **`REPEATABLE_READ` and `SERIALIZABLE` on Postgres can fail your transaction with a
  serialization error at commit time, through no fault of your code.** These are not
  bugs. They are the mechanism. A retry is part of the contract, and a service that
  raises its isolation level without adding a retry loop has traded one bug class for
  a new 500-error class.

**(b) The Java-specific fact you do not get from SQL knowledge.**

In `psql`, a long transaction costs you a connection you already own. In a Spring
service, a long transaction costs you **one of `maximum-pool-size`**, which is shared
across every request the JVM is serving. That is the difference, and it is the entire
reason this is a separate topic.

### From Node: the pool you never thought about

In Node with `pg` or TypeORM, there is a pool too. But three things kept you from
meeting this failure mode:

| Node reality | Java reality | Verdict |
|---|---|---|
| One thread; a slow HTTP call yields the event loop and the process keeps serving other requests | A blocking HTTP call **parks the platform thread**, and if a transaction is open, its connection is parked too | **NO ANALOGUE** — this is the new failure mode |
| `await pool.query(...)` returns the connection between statements unless you explicitly hold a client | Spring holds the connection from `BEGIN` to `COMMIT`, across every line of your method | **PARTIAL** — the same is true if you hold a `pg` client, but you can *see* that you are holding it |
| Pool exhaustion is usually visible as a queue in `pool.waitingCount` | `hikaricp.connections.pending` — the same idea, and you must be exporting it | **FULL** |
| A slow downstream degrades that route | A slow downstream inside a transaction takes down **every** route | **NO ANALOGUE** |

That last row is what turns a two-second blip into an outage, and it is this doc's
failure drill.

### What transfers untouched

Everything you know about MVCC, snapshots, `xmin`/`xmax`, tuple visibility, vacuum and
bloat is directly applicable and you should use it aggressively. This doc's job is to
attach that knowledge to the JDBC connection and the thread.

---

## What is this?

Two mechanisms that meet at the same place.

**Isolation** is the setting on `@Transactional` that Spring translates into a JDBC call
before your first statement runs:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
```

becomes, at the JDBC level, `Connection.setTransactionIsolation(TRANSACTION_REPEATABLE_READ)`,
which Postgres receives as `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ`.

**The connection pool** is HikariCP, which Spring Boot auto-configures. It holds a fixed
number of open JDBC connections. A transaction takes one on `BEGIN` and returns it on
`COMMIT` or `ROLLBACK`. Not before.

The interaction is the topic: **isolation determines correctness under concurrency;
transaction duration determines how much concurrency you can have at all.**

### `Isolation.DEFAULT` and what it means

```java
@Transactional(isolation = Isolation.DEFAULT)   // the default
```

`DEFAULT` means "do not call `setTransactionIsolation` at all — use whatever the
connection already has". For a Boot application against Postgres that is
`READ COMMITTED`, unless you changed `default_transaction_isolation` on the server or
set `spring.datasource.hikari.transaction-isolation`.

This matters more than it looks: a pooled connection is **reused**. If some code path
sets an isolation level on a connection and the pool does not reset it, the next
transaction inherits it. HikariCP does reset it — it records the state on checkout and
restores it on return — but this is a real class of bug in hand-rolled pools, and it is
worth knowing why `DEFAULT` is not the same as "READ COMMITTED".

---

## Why does it matter?

**1. The most common production Java outage has this shape.**
Not a memory leak, not a GC pause. A downstream service slows down, some transaction
somewhere is waiting on it, the pool drains, and a service that "doesn't even call that
downstream on this endpoint" returns 500s. If you can recognise this shape in an
incident channel in thirty seconds, you are visibly senior.

**2. Isolation is a correctness decision with a throughput price, and most teams pick
by cargo cult.**
"We use `READ_COMMITTED`" is usually true and usually unexamined. The interesting
question is which specific `orderflow` operations are *not* safe at `READ_COMMITTED`,
and whether the fix is a higher isolation level or a better-shaped statement. Usually it
is the latter.

**3. Raising the isolation level without a retry loop creates a new outage.**
`REPEATABLE_READ` and `SERIALIZABLE` on Postgres fail transactions with `40001`. If your
code does not retry, you have converted a rare data anomaly into a routine 500.

---

## Machine-level reality

### The exact lifecycle of one connection

Trace a single `POST /orders` request through the layers. This is the sequence you
should be able to draw on a whiteboard.

```
1.  Tomcat worker thread "http-nio-8080-exec-7" picks up the request.
2.  Controller calls placementService.place(cmd) -> hits the CGLIB proxy (Topic 40).
3.  TransactionInterceptor -> JpaTransactionManager.getTransaction(attrs)
4.      EntityManagerFactory.createEntityManager()      // persistence context created
5.      entityManager.getTransaction().begin()
6.          -> HikariDataSource.getConnection()          // <-- CONNECTION TAKEN HERE
7.             HikariPool.getConnection(30_000)          //     blocks up to connectionTimeout
8.          -> conn.setAutoCommit(false)
9.          -> conn.setTransactionIsolation(...)         // only if isolation != DEFAULT
10.     TransactionSynchronizationManager.bindResource(emf, emHolder)
11.     ...ALSO binds the ConnectionHolder for the DataSource...
12. YOUR METHOD BODY RUNS  <-- everything here happens with the connection held
13. TransactionInterceptor -> commit
14.     entityManager.flush()                            // Hibernate emits the writes
15.     conn.commit()                                    // Postgres writes WAL
16.     TransactionSynchronizationManager.unbindResource(emf)
17.     entityManager.close()
18.     conn.close()                                     // <-- RETURNED TO THE POOL
```

**Step 6 to step 18 is the reservation window.** Everything you write in your method
body sits inside it. There is no line of Spring code that releases the connection early
and reacquires it later; the JDBC transaction *is* the connection.

Two subtleties that come up in interviews:

- **The connection is taken lazily on some stacks.** `DataSourceTransactionManager` can
  be configured to defer acquisition, and Hibernate's
  `hibernate.connection.handling_mode` has a `DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION`
  option. Do not rely on this as a fix. The default with `JpaTransactionManager` is
  acquisition at `begin`, and the window you can shave off this way is the part *before*
  your first statement, which is not the part that hurts.
- **`conn.close()` does not close anything.** Hikari hands you a proxy; `close()` returns
  the physical connection to the pool. This is why `try (Connection c = ds.getConnection())`
  is correct and not wasteful.

### What the `ThreadLocal` binding means when threads change

From Topic 54: `TransactionSynchronizationManager` holds the bound `EntityManagerHolder`
and `ConnectionHolder` in `ThreadLocal`s. Now apply that here.

| Situation | What happens to the connection |
|---|---|
| Your method calls a repository | Finds the bound holder. **Same connection.** Correct. |
| Your method submits to an `ExecutorService` and the task calls a repository | The pool thread has an **empty** `TransactionSynchronizationManager`. It takes a **second, independent** connection from Hikari, with its own transaction. Your outer transaction's uncommitted writes are **invisible** to it. |
| Your method submits `N` tasks and waits for them | The request thread holds 1 connection while blocked; the `N` tasks hold up to `N` more. Peak `N+1` per request. |
| Your method calls an `@Async` method | Same as above, plus you have also lost the `SecurityContext` (Topic 56) and the MDC (Topic 120) |
| Virtual threads (`spring.threads.virtual.enabled=true`) | Each virtual thread has its own `ThreadLocal`s, so semantics are unchanged — but you can now have **100,000** threads competing for 20 connections. The pool becomes the bottleneck much sooner and much more visibly. Topic 101. |

That second row is a specific bug worth naming: **an async task inside a transaction
cannot see the transaction's uncommitted data.** People write "save the order, then
kick off an async job that reads the order", and the async job intermittently finds
nothing — because it raced the commit. The fix is `@TransactionalEventListener(AFTER_COMMIT)`,
which at least guarantees the data is durable before the listener runs (Topic 54), or
the outbox (Topic 115).

### How Postgres implements each isolation level

You know MVCC. This is the mapping, stated precisely enough to defend in an interview.

Every row version carries `xmin` (the transaction that created it) and `xmax` (the
transaction that deleted or superseded it). A **snapshot** is a set of transaction IDs
that were committed at a point in time. Visibility is: a row version is visible if its
`xmin` is committed-in-my-snapshot and its `xmax` is not.

| Level | Snapshot policy | Conflict detection | Failure mode |
|---|---|---|---|
| `READ COMMITTED` | **A new snapshot for every statement** | On an `UPDATE` that hits a row modified by a concurrent committed transaction, Postgres *re-evaluates* the row's `WHERE` clause against the new version and proceeds — "EvalPlanQual" | None. It just proceeds, which is why lost updates are possible in read-modify-write code. |
| `REPEATABLE READ` | **One snapshot for the whole transaction**, taken at the first statement | On an `UPDATE` to a row that a concurrent transaction committed after the snapshot: **abort** | `40001 could not serialize access due to concurrent update` |
| `SERIALIZABLE` | Same single snapshot, **plus SIREAD predicate locks** on everything read | Postgres builds a dependency graph between transactions and aborts one when it detects a dangerous structure (a pivot with an inbound and outbound rw-conflict) | `40001 could not serialize access due to read/write dependencies among transactions` — and this can fire on a transaction that only ever **read** the conflicting data |

The `SERIALIZABLE` line is the important one for interviews. SSI can abort a transaction
because of what it *read*, not what it wrote. This means:

- You cannot predict which transaction gets aborted; Postgres picks.
- Read-only transactions can be aborted too (though `SET TRANSACTION READ ONLY DEFERRABLE`
  avoids it by waiting for a safe snapshot instead).
- **Retry is mandatory, and the retry must re-read.** Retrying with the same in-memory
  data reproduces the same conflict.

### Why `READ COMMITTED` is not enough for a read-modify-write

This is the concrete thing to know about `orderflow`. Consider two concurrent order
placements against one unit of stock:

```java
@Transactional   // READ COMMITTED
public void reserve(String sku, int qty) {
    Inventory inv = inventoryRepository.findBySku(sku);   // SELECT available -> 1
    if (inv.getAvailable() >= qty) {                      // both see 1, both pass
        inv.setAvailable(inv.getAvailable() - qty);       // both compute 0
    }
}                                                          // both UPDATE ... SET available = 0
```

Both transactions read `available = 1` from their own statement-level snapshot, both
pass the check, both write `0`. You have sold two units of one. This is a **lost
update**, and `READ COMMITTED` does not prevent it — this is exactly Topic 52's
territory.

There are three fixes and they cost different amounts:

| Fix | Mechanism | Cost |
|---|---|---|
| `@Version` optimistic lock (Topic 52) | The `UPDATE` carries `WHERE version = ?`; the loser gets zero rows updated and Hibernate throws `OptimisticLockException` | A retry loop, and under high contention most attempts fail |
| `PESSIMISTIC_WRITE` (`SELECT ... FOR UPDATE`) | The second transaction **blocks** at the `SELECT` until the first commits | Serialised writers; throughput bounded by transaction duration — and now transaction duration is *everyone's* problem |
| Atomic conditional `UPDATE` | `UPDATE inventory SET available = available - ? WHERE sku = ? AND available >= ?` and check the row count | Cheapest and safest. **This is the right answer for `orderflow`'s inventory.** |
| Raise isolation to `REPEATABLE READ` | Postgres aborts the loser with `40001` | Correct, but you must add a retry loop, and you have paid for snapshot isolation across the whole transaction to fix one statement |

**The senior instinct:** reach for the correctly-shaped statement before reaching for a
higher isolation level. A single atomic `UPDATE` with a guard predicate solves the
inventory problem at `READ COMMITTED` with no retries and no extra locking. Isolation
levels are for when the invariant spans multiple statements and cannot be collapsed into
one.

### `PESSIMISTIC_WRITE` and lock waits inside the reservation window

One more mechanism to make explicit, because it compounds with the pool problem.
`SELECT ... FOR UPDATE` blocks. While it blocks, the waiting transaction still holds its
connection. So a row-lock queue on a hot SKU becomes a connection-pool queue:

- 50 concurrent orders for the same hot SKU.
- Each takes `FOR UPDATE` on that row, in turn.
- Transaction 1 runs for 40 ms; transaction 50 waits ~2 seconds.
- All 50 hold a connection the whole time.
- Pool size 20 → 30 of them never even got a connection.

Set `lock_timeout` (Postgres) or `@QueryHints` with `jakarta.persistence.lock.timeout`
so this fails fast rather than draining the pool. Failing fast with a 409 is a far
better outage than everything timing out.

---

## Example 1 — minimal

The whole topic in one comparison. Two methods, identical business logic, one
architectural difference.

```java
package com.orderflow.payments;

@Service
public class PaymentService {

    private final PaymentRepository payments;
    private final GatewayClient gateway;      // an HTTP client. p99 180 ms. Sometimes 30 s.

    // WRONG. The gateway call is inside the reservation window.
    @Transactional
    public void authorizeInline(long paymentId) {
        Payment payment = payments.findById(paymentId).orElseThrow();  // connection taken
        GatewayResult result = gateway.authorize(payment.reference()); //  <-- HTTP, holding it
        payment.applyResult(result);
    }                                                                   // connection released

    // RIGHT. Three phases, and only two of them are transactional.
    public void authorize(long paymentId) {
        PaymentRef ref = loadRef(paymentId);                  // tx 1: short
        GatewayResult result = gateway.authorize(ref);        // no transaction, no connection
        applyResult(paymentId, result);                       // tx 2: short
    }

    @Transactional(readOnly = true)
    PaymentRef loadRef(long paymentId) { ... }

    @Transactional
    void applyResult(long paymentId, GatewayResult result) { ... }
}
```

The difference in numbers, using `orderflow`'s constraints (pool 20, target 120 rps):

| | `authorizeInline` | `authorize` |
|---|---|---|
| Connection held per request | ~185 ms (5 ms DB + 180 ms HTTP) | ~10 ms (two short transactions) |
| Max throughput from Little's Law (`20 / D`) | **~108 rps** — below the 120 rps target | **~2000 rps** — the pool is not the constraint |
| When the gateway p99 goes to 2 s | ~10 rps. Total outage. | Unchanged. The pool never sees it. |

Note what the right-hand version costs: **it is no longer atomic**. The gateway can
succeed and `applyResult` can fail, leaving an authorised payment with no local record.
That is a real problem and it has a real answer — an idempotency key plus a
reconciliation job (Topic 116), or the outbox (Topic 115). What it is *not* is a reason
to put the HTTP call back inside the transaction, because atomicity across a network
boundary was never available to you in the first place. A transaction cannot roll back
someone else's database.

**`loadRef` and `applyResult` are called via `this`, so re-read Topic 54's Trap 1**: as
written above they are self-invoked and therefore not transactional. Either move them to
a collaborator bean, or use `TransactionTemplate`. This is deliberate — it is a live
reminder that these two topics compound.

---

## Example 2 — production scenario on the `orderflow` spine

### The situation

`orderflow` is running the Topic 65 baseline stack:

| Constraint | Value |
|---|---|
| Replicas | 6 |
| Tomcat threads per replica | 200 (Boot default) |
| Hikari `maximum-pool-size` | 20 per replica |
| Hikari `connection-timeout` | 30 s (default) |
| Postgres | 8 vCPU, `max_connections = 200` |
| Traffic mix | 70% catalogue read, 20% order read, 10% order placement |
| Peak | 1,200 rps total, so ~120 order placements/sec |
| SLO | catalogue p99 ≤ 80 ms; `POST /orders` p99 ≤ 400 ms; error rate ≤ 0.1% |

Total connections at full pool: 6 × 20 = 120 against `max_connections = 200`. That
leaves headroom for migrations, admin sessions and a batch job. This is a deliberate
number, not a default.

### The budget, computed

Little's Law for the pool: `throughput = pool_size / transaction_duration`.

Per replica, 20 orders/sec of placement traffic. To not be pool-bound at 2× headroom:

```
D_max = pool_size / (2 * throughput_per_replica)
      = 20 / (2 * 20)
      = 500 ms per transaction, worst case

Realistic target: p99 transaction duration <= 150 ms.
```

**150 ms is now a number you defend in code review.** Anything that could push a
transaction past it is a design change, not a detail.

### The three transaction shapes in `orderflow`

**Shape 1 — the contended write. Correct isolation via statement shape.**

```java
public interface InventoryRepository extends JpaRepository<Inventory, Long> {

    @Modifying
    @Query("""
           update Inventory i
              set i.available = i.available - :qty,
                  i.reserved  = i.reserved  + :qty
            where i.sku = :sku
              and i.available >= :qty
           """)
    int reserve(@Param("sku") String sku, @Param("qty") int qty);
}
```

```java
@Transactional(timeout = 3)                      // isolation = DEFAULT = READ COMMITTED
public void reserveAll(List<OrderLineCommand> lines) {
    for (OrderLineCommand line : lines) {
        int updated = inventory.reserve(line.sku(), line.quantity());
        if (updated == 0) {
            throw new OutOfStockException(line.sku());   // unchecked -> rolls back
        }
    }
}
```

Why this is the right answer and not `SERIALIZABLE`:

- The guard `and i.available >= :qty` is evaluated **by Postgres, under the row lock the
  `UPDATE` itself takes**. There is no window between the read and the write, because
  there is no separate read.
- Row count zero means "someone else got there first, or there was never enough". One
  branch, no retry loop.
- It works at `READ COMMITTED`. No serialization failures, no retries, no extra locking
  beyond the row lock the `UPDATE` needs anyway.
- Under a flash sale on a top-50 SKU, writers still serialise on that row — but each
  holds the lock for microseconds, not for the duration of the transaction's other work,
  *provided* the `UPDATE` is late in the transaction. Order your statements so contended
  writes happen last.

> Topic 52 covers `@Version` and pessimistic locking in full. The point here is the
> isolation-level decision: **the statement shape removed the need to raise the level.**

**Shape 2 — the multi-statement invariant that genuinely needs a higher level.**

Monthly settlement: compute a merchant's payout as the sum of settled payments in a
period, then write a `Payout` row asserting that total. If a payment settles between the
`SELECT SUM` and the `INSERT`, the payout is wrong and nothing detects it — a phantom
read.

```java
@Service
public class PayoutService {

    @Transactional(isolation = Isolation.REPEATABLE_READ, timeout = 30)
    public PayoutId createPayout(long merchantId, LocalDate period) {
        long totalMinor = payments.sumSettledFor(merchantId, period);   // snapshot fixed here
        return payouts.save(new Payout(merchantId, period, totalMinor)).id();
    }
}
```

And — non-negotiably — the retry:

```java
@Retryable(
    retryFor = { CannotAcquireLockException.class, CannotSerializeTransactionException.class },
    maxAttempts = 4,
    backoff = @Backoff(delay = 50, multiplier = 3.0, random = true))
public PayoutId createPayoutWithRetry(long merchantId, LocalDate period) {
    return payoutService.createPayout(merchantId, period);   // a DIFFERENT bean: proxy!
}
```

Four things to notice, each of which is a review comment you should be able to make:

1. **The retry must be outside the transaction.** Retrying inside a doomed transaction
   does nothing — the transaction is already marked for rollback. Hence the separate
   bean, and hence the Topic 54 proxy rule applying again.
2. **Spring translates the Postgres SQLSTATE.** `40001` becomes
   `CannotSerializeTransactionException` (a `ConcurrencyFailureException`), and `40P01`
   (deadlock) becomes `CannotAcquireLockException`. This translation is why Spring's
   `DataAccessException` hierarchy is unchecked — Topic 09's argument, made concrete.
3. **Jitter (`random = true`) is not optional.** Without it, two conflicting transactions
   retry in lockstep and conflict again.
4. **`timeout = 30` is deliberately long here** because this is a batch path, not a
   request path, and it does not share the request pool if you give it its own
   `DataSource`. If it *does* share the pool, 30 seconds of one connection at peak is a
   5% capacity hit for that duration — compute it, do not assume it.

**Shape 3 — the read path.**

```java
@Transactional(readOnly = true, timeout = 2)
public Page<OrderSummary> recentOrders(long customerId, Pageable page) {
    return orders.findSummariesByCustomerId(customerId, page);   // projection, Topic 47
}
```

`readOnly = true` here is a hint (see Topic 54 for exactly what it does and does not
guarantee) plus, in a replica-aware setup, the routing signal. `timeout = 2` caps the
blast radius. A projection rather than an entity keeps the persistence context small,
which keeps the flush cheap, which keeps the transaction short — every decision in this
service points back at the same 150 ms budget.

### What is deliberately *not* transactional

| Operation | Why it has no transaction |
|---|---|
| `GET /products/{id}` served from Redis | No database access at all. Do not annotate it — an empty transaction still takes a connection. |
| The fraud check | External HTTP. Runs in the controller, before the service call. |
| Sending the order-confirmation email | `@TransactionalEventListener(AFTER_COMMIT)` on a bounded async executor (Topic 90). Not the request thread, not the transaction. |
| The Kafka publish | An outbox row inside the transaction; a relay publishes it (Topic 115). |

That last row is the general pattern: **when something must be atomic with a database
write but is not itself a database write, make it a database write.**

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — an external HTTP call inside a transaction

**This is the one. Everything else in this doc is context for this trap.**

**Wrong:**
```java
@Transactional
public PaymentId capture(long orderId) {
    Order order = orders.findById(orderId).orElseThrow();      // connection taken
    GatewayResult result = gatewayClient.capture(order.ref()); // 180 ms... usually
    Payment payment = Payment.from(result);
    payments.save(payment);
    order.markPaid();
    return payment.id();
}                                                               // connection released
```

**Exact symptom, in the order you will observe it during the incident:**

1. The payment gateway's p99 goes from 180 ms to 2 s. Their status page says
   "investigating".
2. `hikaricp.connections.active` on every replica pins at 20 and stays there.
3. `hikaricp.connections.pending` climbs from 0 into the dozens, then the hundreds.
4. `hikaricp.connections.timeout` starts incrementing.
5. **`GET /products` starts returning 500.** This is the moment the incident becomes
   confusing, because the catalogue endpoint reads from Redis and does not call the
   gateway at all. Someone says "the cache must be down".
6. Health checks that hit the database fail. Kubernetes starts restarting pods.
   Restarting drops in-flight requests, the retries hit the remaining replicas, and the
   remaining replicas fall over faster. **A 2-second downstream blip has become a
   total outage.**
7. Errors are `SQLTransientConnectionException: HikariPool-1 - Connection is not
   available, request timed out after 30000ms` *(illustration of the message format)*.

**Root cause:** two compounding facts and one configuration mistake.

- The connection is held for the entire method, including the 2-second HTTP call. At 20
  orders/sec/replica and 2 s per transaction, Little's Law says you need 40 connections
  and you have 20. The pool is oversubscribed by 2× and the queue is unbounded.
- Every Tomcat thread that reaches this code path parks in `HikariPool.getConnection`.
  With 200 Tomcat threads and 20 connections, 180 threads are parked. Those threads are
  now unavailable to serve *any* request, which is why endpoints with no database access
  fail.
- The configuration mistake: `connection-timeout: 30000`. Thirty seconds is far longer
  than any request SLO. It converts "fail fast" into "hold a thread hostage for thirty
  seconds".

**Fix — in order:**

```java
// 1. THE FIX. Move the call out. Split the transaction in two.
public PaymentId capture(long orderId) {
    OrderRef ref = orderReader.loadRef(orderId);            // tx 1, ~5 ms
    GatewayResult result = gatewayClient.capture(ref);      // NO transaction, NO connection
    return paymentWriter.recordCapture(orderId, result);    // tx 2, ~10 ms
}
```

```yaml
# 2. Bound the damage even when someone gets it wrong again.
spring:
  datasource:
    hikari:
      connection-timeout: 2000      # fail fast, well inside the request SLO
      maximum-pool-size: 20
      leak-detection-threshold: 5000   # log a stack trace for any connection held > 5 s
```

```java
// 3. Bound the downstream itself. A client with no timeout is a bug regardless.
@Bean
RestClient gatewayRestClient(RestClient.Builder builder) {
    ClientHttpRequestFactorySettings settings = ClientHttpRequestFactorySettings.defaults()
        .withConnectTimeout(Duration.ofMillis(500))
        .withReadTimeout(Duration.ofSeconds(2));
    return builder.requestFactory(ClientHttpRequestFactories.get(settings)).build();
}
```

> I am confident about the *shape* here — a builder plus explicit connect and read
> timeouts — but the exact factory-settings API has moved between Framework 6.x and 7.0.
> Check the current `RestClient` / `ClientHttpRequestFactorySettings` reference for your
> version rather than trusting this method name. The principle does not change: **every
> outbound client gets an explicit connect timeout and read timeout, always.**

```java
// 4. Circuit-break the downstream so a sustained outage sheds load instead of queueing.
//    Topic 111.
```

**`leak-detection-threshold` deserves its own line.** Set it to something just above your
worst legitimate transaction (5 s is a reasonable start). Hikari will then log a warning
with the **full stack trace of the code that acquired the connection** whenever one is
held longer. That single setting turns "some transaction somewhere is slow" into a file
and line number. Enable it in staging permanently and in production during an incident.

**The general rule, stated so you can quote it in a design review:**

> A transaction may contain database work and nothing else. No HTTP calls, no message
> broker publishes, no file I/O, no `Thread.sleep`, no waiting on a lock or a queue, no
> user interaction. If it is not a query, it goes outside the boundary.

---

### Trap 2 — a long read transaction holding a connection

**Wrong:**
```java
@Transactional(readOnly = true)
public void exportAllOrders(OutputStream out) {
    try (Stream<Order> stream = orders.streamAllByStatus(OrderStatus.SETTLED)) {
        stream.forEach(order -> csvWriter.write(out, toRow(order)));   // 4M rows
    }
}
```

**Exact symptom:** the export takes 11 minutes and works fine in staging with 10,000
rows. In production it holds one connection for 11 minutes. During that time:

- `pg_stat_activity` shows a session in `idle in transaction` — or `active` running a
  cursor — with `xact_start` 11 minutes old.
- `hikaricp.connections.active` shows a persistent baseline of 1 that never drops.
- Autovacuum on the `orders` table cannot remove dead tuples newer than this
  transaction's snapshot, so the table bloats. `n_dead_tup` in `pg_stat_user_tables`
  climbs and never falls. Query plans on `orders` degrade for **everyone**.
- Run four exports concurrently at month-end and 4 of 20 connections are gone for the
  whole window.

**Root cause:** two separate costs that people conflate.

1. **The pool cost.** One connection unavailable for 11 minutes.
2. **The MVCC cost, which is worse.** An open transaction holds a snapshot. Postgres
   cannot vacuum any row version that this snapshot might still need — that is exactly
   what `xmin` horizon means. A long-running transaction is a *global* brake on vacuum,
   and this is one of the most common causes of unexplained Postgres bloat. Your SQL
   background should make this immediately familiar; the new part is that a Spring
   `@Transactional(readOnly = true)` on an innocuous-looking export method is what
   created it.

**Fix:**

```java
// 1. Keyset-paginate. Many short transactions instead of one long one.
public void exportAllOrders(OutputStream out) {
    long lastId = 0;
    List<OrderRow> batch;
    do {
        batch = orderReader.nextPage(lastId, 5_000);     // its own short @Transactional
        batch.forEach(row -> csvWriter.write(out, row));
        if (!batch.isEmpty()) lastId = batch.get(batch.size() - 1).id();
    } while (!batch.isEmpty());
}
```

Keyset (`WHERE id > :lastId ORDER BY id LIMIT 5000`) rather than `OFFSET`, because
`OFFSET 3_000_000` makes Postgres scan and discard three million rows every page.

```yaml
# 2. Give reporting its own DataSource and pool, so it can never starve the request path.
spring:
  datasource:
    hikari.maximum-pool-size: 20
  # plus a second, separately-configured DataSource for reporting, pool size 2,
  # pointed at a read replica.
```

```sql
-- 3. Set a server-side seatbelt so this can never happen again by accident.
ALTER ROLE orderflow_app SET idle_in_transaction_session_timeout = '10s';
ALTER ROLE orderflow_app SET statement_timeout = '5s';
ALTER ROLE orderflow_report SET statement_timeout = '15min';
```

`idle_in_transaction_session_timeout` is the single highest-value Postgres setting for a
Spring application. It kills exactly the pathology in Trap 1 — a transaction open with no
statement running — at the database, regardless of what the application does.

---

### Trap 3 — `readOnly = true` treated as a guarantee

**Wrong:**
```java
@Transactional(readOnly = true)
public OrderView view(long orderId) {
    Order order = orders.findById(orderId).orElseThrow();
    if (order.getDisplayName() == null) {
        order.setDisplayName(deriveName(order));   // "safe — it's readOnly"
    }
    return OrderView.from(order);
}
```

**Exact symptom:** the derived name is correct in the response. It is **not** in the
database. The next request derives it again. A `LEFT JOIN` on `display_name` in a report
returns nulls forever. Nothing in the logs, no exception, no failed test — the response
is right, so the endpoint's test passes.

*Or*, on a different configuration, an unhandled
`InvalidDataAccessResourceUsageException` wrapping
`ERROR: cannot execute UPDATE in a read-only transaction` at flush time, from a method
that "only reads".

**Root cause:** `readOnly = true` sets Hibernate's `FlushMode` to `MANUAL`, so dirty
checking still runs but the flush never happens and the change is **silently discarded**.
Separately, *if* the transaction manager propagated the flag to
`Connection.setReadOnly(true)`, Postgres issues `SET TRANSACTION READ ONLY` and rejects
the write hard. Which of the two you get depends on configuration. Full mechanics in
Topic 54.

**Fix:**

- Read paths return **projections or DTOs**, never managed entities. If the object is
  not managed, there is nothing to dirty-check and the whole question disappears
  (Topic 47).
- If a read path legitimately needs to write — a lazily-derived field, a `last_seen`
  timestamp — that is a separate, explicitly non-read-only transaction, or an async
  write after the response.
- If you want a real guarantee, use a database role without write permission.

**The mental correction:** `readOnly = true` means "I intend not to write, please skip
the flush and route me to a replica if you can". It does not mean "writes are
impossible".

---

### Trap 4 — raising the isolation level with no retry loop

**Wrong:**
```java
@Transactional(isolation = Isolation.SERIALIZABLE)   // "to be safe"
public void applyPromotion(long orderId, String promoCode) { ... }
```

**Exact symptom:** works perfectly in development and in every test, because there is no
concurrency. In production, at low load, a trickle of 500s — maybe 0.3% of requests.
Under a flash sale, 15% of requests fail. The logs show
`CannotSerializeTransactionException` wrapping
`PSQLException: ERROR: could not serialize access due to read/write dependencies among
transactions`. The failing requests are not correlated with any particular input, so
nobody can reproduce it.

**Root cause:** `SERIALIZABLE` on Postgres is SSI. It detects dangerous read/write
dependency structures and **aborts one of the participants**. That abort is not an error
condition; it is how the guarantee is delivered. A transaction that has done nothing
wrong is chosen and killed. Without a retry, that is a 500 for the user.

Worse: raising the level also raises the *transaction duration* (predicate locks and
tracking are not free) and therefore consumes more pool capacity, which increases
concurrency, which increases the conflict rate. It is a positive feedback loop.

**Fix:**

```java
// 1. First, ask whether the invariant can be expressed in one statement. It usually can.
//    An atomic conditional UPDATE at READ COMMITTED beats SERIALIZABLE almost always.

// 2. If a higher level is genuinely necessary, retry — outside the transaction,
//    with jitter, with a bounded attempt count, and with a metric.
@Retryable(retryFor = CannotSerializeTransactionException.class,
           maxAttempts = 4,
           backoff = @Backoff(delay = 50, multiplier = 3.0, random = true))
public void applyPromotionWithRetry(long orderId, String code) { ... }

// 3. Prefer REPEATABLE READ to SERIALIZABLE when snapshot isolation is enough.
//    In Postgres, REPEATABLE READ is genuine snapshot isolation and also prevents
//    phantoms; it only fails on write-write conflicts, which is a much narrower and
//    more predictable failure surface than SSI's read/write dependency graph.

// 4. Emit a metric on every retry. A rising serialization-failure rate is the earliest
//    warning that contention is growing. Silent retries hide a capacity problem.
```

**The rule:** never raise an isolation level without (a) naming the specific multi-
statement invariant it protects, and (b) shipping the retry loop in the same commit.

---

### Trap 5 — sizing the pool by "it timed out, make it bigger"

**Wrong:**
```yaml
spring.datasource.hikari.maximum-pool-size: 100    # was 20, we had timeouts
```

**Exact symptom:** the timeouts stop. Then p99 latency across every endpoint gets
**worse** — often 2–3× worse. Postgres CPU pins at 100%. `pg_stat_activity` shows 100
active backends against 8 vCPUs. Under a real spike, connection count across 6 replicas
reaches 600 against `max_connections = 200` and you now get
`FATAL: sorry, too many clients already` — an error that takes down *every* replica at
once, including ones that were healthy.

**Root cause:** two things.

1. **Postgres runs a process per connection.** 100 active connections on 8 cores means
   the OS is context-switching between 100 processes to do 8 cores' worth of work.
   Throughput does not increase past core count; latency increases for everyone. The
   well-established guidance is roughly `connections ≈ cores × 2` for a CPU-bound
   workload, adjusted upward for I/O wait.
2. **A pool timeout is almost never caused by a pool that is too small.** It is caused
   by transactions that are too long. Raising the pool size treats the symptom and
   makes the underlying problem harder to see, because the queue moves from Hikari
   (where you have a metric) into Postgres (where you do not, unless you look).

**Fix:**

```
1. Measure transaction duration first: hikaricp.connections.usage p99.
2. Apply Little's Law: required_pool = throughput * duration.
   At 20 rps and 150 ms: 20 * 0.15 = 3 connections. A pool of 20 is 6x headroom.
   If the arithmetic says you need 100, the duration is the problem, not the pool.
3. Find the long transactions: leak-detection-threshold, and pg_stat_activity ordered
   by xact_start.
4. Only then, size the pool from database capacity:
   total_connections_across_all_replicas < max_connections, with headroom for admin.
5. Add a connection-timeout well inside your request SLO so failure is fast and visible.
```

Full treatment, including the pool-vs-thread-pool deadlock and the latency-versus-pool-
size curve, is Topic 109. The instinct to build now: **a bigger pool is almost never the
fix, and "we raised the pool" in an incident review is a finding, not a resolution.**

---

## Hands-on proof

Commands **you** run. No JVM here, so no captured output — what follows is the exact
configuration, what to look for, and how to read every result.

### Setup

`application-lab.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orderflow
    hikari:
      pool-name: orderflow-pool
      maximum-pool-size: 5              # small, so the effect is visible on a laptop
      connection-timeout: 3000
      leak-detection-threshold: 2000
  jpa:
    properties:
      hibernate.format_sql: true
logging:
  level:
    org.springframework.transaction: TRACE
    org.springframework.orm.jpa: DEBUG
    org.hibernate.SQL: DEBUG
    com.zaxxer.hikari: DEBUG
    com.zaxxer.hikari.pool.HikariPool: DEBUG
management:
  endpoints.web.exposure.include: metrics,health
  metrics.tags.application: orderflow
```

`com.zaxxer.hikari.pool.HikariPool=DEBUG` makes Hikari log a periodic pool-state line.
Its *shape* is like `Pool stats (total=5, active=5, idle=0, waiting=12)` — **an
illustration of the format, not captured output**. `waiting` is the number you watch.

### Proof 1 — the connection is held for the whole method

Add a deliberate pause inside a transaction. **Lab only.**

```java
@Service
public class LabService {

    @Transactional
    public void slowTransaction(long millis) throws InterruptedException {
        TxProbe.print("slowTransaction");                  // from Topic 54
        orders.count();                                     // forces the connection open
        Thread.sleep(millis);                               // stands in for an HTTP call
    }
}
```

Run it, and while it is sleeping, query both sides.

```bash
curl -s -X POST 'localhost:8080/lab/slow?millis=15000' &
sleep 3
curl -s localhost:8080/actuator/metrics/hikaricp.connections.active | jq '.measurements'
psql -d orderflow -c "SELECT pid, state, now()-xact_start AS age, left(query,40) FROM pg_stat_activity WHERE datname='orderflow' AND state <> 'idle';"
```

| What you see | What it means |
|---|---|
| `hikaricp.connections.active` = 1, and Postgres shows `state = 'idle in transaction'` with a growing `age` | **The proof.** The connection is checked out and the database is doing nothing. Every millisecond of that is capacity you are not using and nobody else can. |
| Postgres shows no session for you at all | The connection was not actually opened. Add a real query (`orders.count()`) before the sleep — Hibernate can defer acquisition until the first statement. |
| `active` = 0 while the request is clearly in flight | The `@Transactional` did not apply. Go back to Topic 54: check `TxProbe`, check for self-invocation. |
| A Hikari `Connection leak detection triggered` warning with a stack trace | `leak-detection-threshold` firing. The stack trace names the exact line that acquired the connection. **This is the single most useful diagnostic in this whole topic.** |

### Proof 2 — isolation is really being set

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void probeIsolation() {
    TxProbe.print("probeIsolation");       // prints the Spring-side isolation constant
    System.out.println("db says: " + jdbc.queryForObject(
        "SHOW transaction_isolation", String.class));
}
```

| What you see | What it means |
|---|---|
| `db says: repeatable read` | Spring called `setTransactionIsolation` and Postgres accepted it. |
| `db says: read committed` when you asked for `REPEATABLE_READ` | You are **participating in an outer transaction** (`REQUIRED` joined it), so your isolation attribute was ignored. Check `getCurrentTransactionName()` — if it names a different method, that is your answer. This is a real and common bug. |
| You asked for `READ_UNCOMMITTED` and get `read committed` | Expected. Postgres maps `READ UNCOMMITTED` to `READ COMMITTED`. There are no dirty reads in Postgres. |
| An exception at `begin` | Some pool or driver configurations reject an isolation change on a connection that already has one set. Check `spring.datasource.hikari.transaction-isolation`. |

### Proof 3 — see two snapshots disagree

Two `psql` sessions, no Java. This is the fastest way to make the level concrete.

```sql
-- session A
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT available FROM inventory WHERE sku = 'SKU-1001';   -- note the value

-- session B (a different terminal)
UPDATE inventory SET available = available - 1 WHERE sku = 'SKU-1001';
COMMIT;

-- session A again
SELECT available FROM inventory WHERE sku = 'SKU-1001';   -- ?
UPDATE inventory SET available = available - 1 WHERE sku = 'SKU-1001';   -- ?
COMMIT;
```

| What you see | What it means |
|---|---|
| A's second `SELECT` returns the **same** value as the first | Snapshot isolation. A is reading its transaction-start snapshot and cannot see B's commit. |
| A's `UPDATE` fails with `ERROR: could not serialize access due to concurrent update` | The write-write conflict detection. This is the `40001` your Java code must retry. |
| Repeat the whole thing with `BEGIN ISOLATION LEVEL READ COMMITTED`: A's second `SELECT` returns the **new** value and the `UPDATE` succeeds | Statement-level snapshots. And note: A's `UPDATE` silently applied to a row it never read at that value — that is how lost updates happen. |

Now run the equivalent in `orderflow` with two concurrent HTTP requests and confirm that
the atomic conditional `UPDATE` from Example 2 gives you row count 0 for the loser
instead of an exception. That is the whole argument for statement shape over isolation
level, demonstrated rather than asserted.

### Proof 4 — read the pool from both sides at once

```bash
# Application side
for m in active idle pending timeout usage acquire; do
  echo "== hikaricp.connections.$m"
  curl -s localhost:8080/actuator/metrics/hikaricp.connections.$m | jq -c '.measurements'
done
```

```sql
-- Database side
SELECT state, count(*), max(now() - xact_start) AS oldest_xact
FROM pg_stat_activity WHERE datname = 'orderflow' GROUP BY state;
```

| Application side | Database side | Diagnosis |
|---|---|---|
| `active` high, `pending` > 0 | many `active` | Genuine database load. Optimise queries or scale Postgres. |
| `active` high, `pending` > 0 | many **`idle in transaction`** | **Trap 1.** The application is holding connections while doing non-database work. Find it with `leak-detection-threshold`. |
| `active` low, `pending` = 0, requests still slow | few sessions | Not a pool problem. Look at the thread pool, GC, or the downstream. |
| `active` = `max`, `timeout` incrementing | many `idle in transaction` | Live outage, Trap 1 shape. Kill the long transactions to recover, then fix the boundary. |
| Total connections across replicas near `max_connections` | `FATAL: too many clients` in the Postgres log | Trap 5. Pool sizing was done by increment, not arithmetic. |

---

## Failure drill

**Mandatory.** You are going to take down `orderflow` with one `Thread.sleep`, watch
endpoints that touch no database fail, and then fix it with a refactor rather than a
setting.

### Setup

**Step 1 — the stack.** Topic 65's `docker compose` (app + Postgres + Redis), or a
laptop-scale version. Set the pool small so a laptop can reproduce it:

```yaml
spring:
  datasource.hikari:
    maximum-pool-size: 5
    connection-timeout: 3000
    leak-detection-threshold: 2000
    pool-name: orderflow-pool
management.endpoints.web.exposure.include: metrics,health
logging.level.com.zaxxer.hikari: DEBUG
logging.level.org.springframework.transaction: TRACE
```

**Step 2 — the slow downstream.** Do **not** point at a real third party. Stand up a stub
that sleeps 2 seconds:

```java
// A second tiny Boot app on port 9090, or a WireMock stub with a fixed delay.
@RestController
class SlowGatewayStub {
    @PostMapping("/authorize")
    Map<String, String> authorize() throws InterruptedException {
        Thread.sleep(2000);
        return Map.of("status", "APPROVED");
    }
}
```

**Step 3 — the broken code.** Put the call inside the transaction, exactly as a tired
engineer would:

```java
@Service
public class PaymentAuthorizationService {

    @Transactional                                    // <-- the bug
    public void authorize(long orderId) {
        Order order = orders.findById(orderId).orElseThrow();
        GatewayResponse response = gatewayClient.authorize(order.reference());  // 2 s
        order.markAuthorized(response.status());
    }
}
```

**Step 4 — an endpoint that touches no database at all.** This is the control, and it is
the point of the whole drill:

```java
@RestController
class HealthLikeController {
    @GetMapping("/api/version")
    Map<String, String> version() {
        return Map.of("version", "1.4.2");            // no DB. no cache. nothing.
    }
}
```

### Run the load

```bash
# baseline: confirm both endpoints are healthy and fast
curl -i localhost:8080/api/version
curl -i -X POST localhost:8080/orders/1001/authorize

# now generate load on the transactional endpoint (k6 from Topic 65)
k6 run --vus 30 --duration 90s authorize-load.js
```

`authorize-load.js`:
```javascript
import http from 'k6/http';
export default function () {
  http.post('http://localhost:8080/orders/1001/authorize');
}
```

While it runs, in a second terminal, every 5 seconds:

```bash
curl -s -o /dev/null -w 'version endpoint: %{http_code} in %{time_total}s\n' \
     localhost:8080/api/version

for m in active pending timeout; do
  printf '%s: ' "$m"
  curl -s localhost:8080/actuator/metrics/hikaricp.connections.$m | jq -c '.measurements[0].value'
done

psql -d orderflow -c \
  "SELECT state, count(*), max(now()-xact_start) FROM pg_stat_activity WHERE datname='orderflow' GROUP BY state;"
```

And once the pool is clearly saturated, capture the thread dump:

```bash
jcmd $(jcmd -l | grep orderflow | cut -d' ' -f1) Thread.print > /tmp/orderflow-threads.txt
grep -c 'http-nio-8080-exec' /tmp/orderflow-threads.txt
grep -A 8 'HikariPool.getConnection' /tmp/orderflow-threads.txt | head -60
```

### What to capture

1. `/api/version` status code and latency, sampled through the run.
2. `hikaricp.connections.active`, `.pending`, `.timeout` over time.
3. `pg_stat_activity` state counts.
4. The thread dump.
5. Any Hikari leak-detection warnings from the application log.

### How to read every result

| What you see | What it means |
|---|---|
| `hikaricp.connections.active` pinned at 5 (your `maximum-pool-size`) | The pool is fully checked out. Expected — this is the drill working. |
| `hikaricp.connections.pending` climbing above zero and staying there | Threads are **blocked** waiting for a connection. This is the number that predicts the outage. |
| `hikaricp.connections.timeout` incrementing | Requests are now failing outright. Each increment is a 500 to a user. |
| **`/api/version` returns 500 or times out** | **The headline result.** An endpoint with no database access has failed, because every Tomcat thread is parked waiting for a connection that a *different* endpoint is holding. Sit with this. It is the reason the rule exists. |
| `/api/version` stays fast | Your load is not high enough to saturate the Tomcat thread pool. Raise VUs, or lower `server.tomcat.threads.max`, until Tomcat threads are exhausted too. The pool alone is not the whole failure — thread starvation is what generalises it to every endpoint. |
| `pg_stat_activity` shows 5 sessions `idle in transaction`, ages ~2 s | **The smoking gun.** Postgres is completely idle. It is not the bottleneck. The application is holding connections to do nothing. |
| Postgres CPU near zero while the service is failing | Confirms the same thing from the other side, and is the fact that ends the "is it the database?" argument in an incident channel. |
| Thread dump: many `http-nio-8080-exec-N` threads with `HikariPool.getConnection` and `parkNanos` in the stack | Thread starvation confirmed, with the exact call site. |
| Thread dump: threads inside the HTTP client's socket read | These are the 5 that *have* a connection and are waiting on the gateway. Count them: they should equal your pool size. |
| Hikari `Connection leak detection triggered ... Stack trace:` naming `PaymentAuthorizationService.authorize` | The tool naming your bug, with a line number, without you having to reason about it. |

### The fix, and what it proves

**Step 1 — move the call out of the transaction.**

```java
@Service
public class PaymentAuthorizationService {

    private final OrderReader orderReader;         // separate beans -> real proxies
    private final OrderWriter orderWriter;
    private final GatewayClient gatewayClient;

    public void authorize(long orderId) {
        OrderRef ref = orderReader.loadRef(orderId);                 // tx 1: ~3 ms
        GatewayResponse response = gatewayClient.authorize(ref);     // 2 s, NO connection
        orderWriter.markAuthorized(orderId, response.status());      // tx 2: ~5 ms
    }
}
```

**Step 2 — re-run the identical load and compare.**

| Measurement | Broken | Fixed | What it proves |
|---|---|---|---|
| `hikaricp.connections.active` | pinned at 5 | fluctuating near 0–1 | The connection is only held for real database work |
| `hikaricp.connections.pending` | climbing | 0 | Nothing is queueing for the pool |
| `hikaricp.connections.timeout` | incrementing | 0 | No request fails for want of a connection |
| `/api/version` | 500 / timeout | 200, fast | **Unrelated endpoints are no longer collateral damage** |
| `POST /authorize` latency | 30 s, then error | ~2.01 s | Still slow — the downstream is still slow. That is correct and honest. |
| `pg_stat_activity` `idle in transaction` | 5, ~2 s old | 0 | The database is no longer holding state for an application that is not talking to it |

**The crucial reading:** the fixed version is **still slow**. You did not make the
gateway faster; you cannot. What you changed is the **blast radius**. A slow downstream
now degrades exactly the endpoint that depends on it, and nothing else. That is the
entire goal, and it is a distinction many engineers never make explicit.

**Step 3 — add the seatbelts and prove they fire.**

```yaml
spring.datasource.hikari.connection-timeout: 2000       # fail fast, inside the SLO
spring.datasource.hikari.leak-detection-threshold: 2000
```
```sql
ALTER ROLE orderflow_app SET idle_in_transaction_session_timeout = '10s';
```

Re-introduce the bug deliberately and confirm: requests now fail in ~2 s instead of 30 s,
the leak detector logs the exact stack trace, and Postgres kills any session that goes
idle in transaction for 10 seconds. Defence in depth, at three layers.

### The general rule

> **A transaction contains database work and nothing else.**
> Transaction duration is not a latency metric — it is a concurrency limit shared by
> every request in the JVM. Anything that can be slow for reasons outside your database
> must be outside the boundary.

---

## Measurement

### The metrics that make the claims falsifiable

| Metric | Reads as | Alert on |
|---|---|---|
| `hikaricp.connections.active` | connections checked out right now | sustained > 80% of `maximum-pool-size` |
| `hikaricp.connections.pending` | threads **blocked** waiting | **> 0 for more than 30 s** — this is the leading indicator |
| `hikaricp.connections.usage` | timer: hold duration per connection | p99 above your Little's Law budget (150 ms for `orderflow`) |
| `hikaricp.connections.acquire` | timer: time spent waiting to acquire | p99 > 10 ms means contention |
| `hikaricp.connections.timeout` | count of acquisition failures | **any increment** — each one is a failed request |
| `hikaricp.connections.creation` | timer: time to open a new physical connection | spikes mean the database or network is struggling |
| `orderflow.tx.duration` (yours, Topic 54) | actual transaction duration, tagged commit/rollback | p99 breach, or a rollback-rate change |
| `spring.data.repository.invocations` | repository call timings | a query that got slower |

Alert on `pending` first. `active` at maximum is not necessarily a problem — a busy pool
is a working pool. `pending` above zero means somebody is *waiting*, and waiting scales
into an outage.

### Database-side measurement

```sql
-- 1. The single most important query in this topic.
SELECT pid, usename, state,
       now() - xact_start  AS xact_age,
       now() - state_change AS state_age,
       wait_event_type, wait_event,
       left(query, 60) AS query
FROM pg_stat_activity
WHERE datname = 'orderflow' AND state <> 'idle'
ORDER BY xact_start NULLS LAST;

-- 2. Serialization failures and deadlocks, cumulative.
SELECT datname, xact_commit, xact_rollback, deadlocks, conflicts
FROM pg_stat_database WHERE datname = 'orderflow';

-- 3. Bloat caused by long-running transactions (Trap 2).
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000 ORDER BY n_dead_tup DESC;

-- 4. Is a long transaction blocking vacuum right now?
SELECT max(now() - xact_start) AS oldest_open_transaction FROM pg_stat_activity
WHERE state <> 'idle' AND xact_start IS NOT NULL;
```

Query 4 belongs on a dashboard permanently. A steadily rising "oldest open transaction"
is the earliest possible warning of both pool exhaustion and table bloat, and it costs
nothing.

### Why a naive `System.nanoTime()` timing is wrong

You will be tempted to write:

```java
long start = System.nanoTime();
paymentService.authorize(orderId);
long ms = (System.nanoTime() - start) / 1_000_000;   // WRONG, for five reasons here
```

1. **It measures the wrong interval.** The transaction begins inside the proxy before
   your method body and commits after it returns. The commit — flush plus WAL write —
   is often the largest single component, and a stopwatch inside the method misses it
   entirely. If you must time something, time it in a `TransactionSynchronization`
   `afterCompletion` callback (Topic 54's `TransactionTimer`), which is the only place
   that sees the real boundary.
2. **It includes queueing you did not intend to measure.** Under load, most of the
   elapsed time can be `HikariPool.getConnection` waiting for a free connection. Your
   number then describes pool contention, not your query, and the two demand completely
   different fixes. `hikaricp.connections.acquire` separates them; a stopwatch cannot.
3. **JIT tiering.** The first thousands of invocations run interpreted, then C1, then
   C2, possibly with on-stack replacement. A single measurement captures an arbitrary
   tier. Topic 77.
4. **Safepoints and GC pauses land in your window arbitrarily**, and get attributed
   entirely to your transaction. Topic 68 onward.
5. **One sample is not a distribution.** Your SLO is a p99. A single number cannot
   describe a tail, and under load, coordinated omission (Topic 65) makes hand-rolled
   measurement systematically *under*-report the tail — often by an order of magnitude,
   because the requests that would have been slowest were never issued.

**What to do instead:** Micrometer timers for distributions and percentiles;
`hikaricp.connections.usage` for the pool-side view of hold time;
`hikaricp.connections.acquire` to separate wait from work; `pg_stat_activity` for the
database's own account; and JMH (Topic 77) for anything microbenchmark-shaped. Under
load, take numbers from the Topic 65 baseline harness, never from a stopwatch.

### The falsifiable claims of this topic

| Claim | How to falsify it |
|---|---|
| "A transaction holds a connection for its whole lifetime" | Proof 1: `idle in transaction` with a growing age while Java sleeps |
| "An HTTP call inside a transaction takes down unrelated endpoints" | The failure drill: `/api/version` returns 500 |
| "Postgres has no dirty reads" | Ask for `READ_UNCOMMITTED`, run `SHOW transaction_isolation`, get `read committed` |
| "`REPEATABLE READ` can fail your write" | Proof 3, session A's `UPDATE` |
| "A bigger pool makes latency worse" | Topic 109: sweep pool size against p99 and plot the curve |
| "A long read transaction blocks vacuum" | Trap 2: hold a transaction open, watch `n_dead_tup` climb and `last_autovacuum` stall |

---

## Practice exercises

### 1 — Easy: prove the connection is pinned, and time-box it

Write `LabController` with a `@Transactional` endpoint that runs one query then sleeps
for a configurable number of milliseconds. Set `maximum-pool-size: 3`.

1. Call it 3 times concurrently with `millis=10000`. Call it a 4th time. Record what the
   4th caller sees and how long it takes to see it.
2. While they are sleeping, record `hikaricp.connections.active`, `.pending`, and the
   output of the `pg_stat_activity` query above.
3. Set `connection-timeout: 1000` and repeat. Report exactly what changed for the 4th
   caller, and argue whether that change is an improvement.
4. Set `idle_in_transaction_session_timeout = '3s'` on the role and repeat. What
   exception does the *application* see now, and at which line?

### 2 — Medium: the audit (combines Topics 08, 09, 47, 48, 52, 54)

The class below has **six** defects spanning this topic and earlier ones. For each: name
it, state the exact observable symptom (log line, HTTP response, database state, or
metric — not "bad practice"), and write the fix.

```java
@Service
public class SettlementService {

    @Autowired private PaymentRepository payments;
    @Autowired private WalletRepository wallets;
    @Autowired private LedgerClient ledgerClient;      // HTTP, p99 400 ms

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void settleAll(LocalDate day) throws LedgerException {

        List<Payment> due = payments.findAllByStatusAndDay(PaymentStatus.PENDING, day);

        for (Payment p : due) {                         // ~40,000 rows at month-end
            Wallet w = wallets.findById(p.merchantId()).orElseThrow();
            w.setBalanceMinor(w.getBalanceMinor() + p.amountMinor());

            LedgerReceipt receipt = ledgerClient.post(p.reference());
            p.setLedgerRef(receipt.id());
            p.setStatus(PaymentStatus.SETTLED);
        }

        if (due.isEmpty()) {
            throw new LedgerException("nothing to settle");
        }
    }
}
```

Hints by topic: 54 (what does a checked exception do here?), 52 (is that balance update
safe under concurrency?), 48 (what is the memory cost of 40,000 managed entities?), 47
(should this load entities at all?), this topic (isolation choice; the HTTP call; the
duration), and 09 (should `LedgerException` be checked?).

Then rewrite it. Your rewrite must state, in a comment, the maximum time any single
transaction can hold a connection, and how you would prove that number.

### 3 — Hard: production simulation — make `orderflow` survive a slow gateway

**Part A — establish the budget.** Using the Topic 65 stack, measure
`hikaricp.connections.usage` p99 for `POST /orders` at 120 rps. Apply Little's Law and
state, in writing, the maximum transaction duration your current pool size supports at
2× headroom. Commit that number to `/docs/java/baselines/` alongside the Topic 65
numbers.

**Part B — break it.** Introduce the 2-second gateway stub inside the transaction. Run
the full Topic 65 scenario mix (70% catalogue read, 20% order read, 10% placement).
Report: catalogue p99 (SLO 80 ms), order-read p99, error rate per endpoint, and the
Hikari metrics. Prove that catalogue reads — which touch only Redis — degrade, and
explain the causal chain in five sentences.

**Part C — fix and re-measure.** Restructure into read-transaction / HTTP / write-
transaction. Re-run the identical scenario. Produce a before/after table for all four
metrics. Confirm that `POST /orders` is still slow and explain why that is the correct
outcome.

**Part D — isolation.** Reproduce the oversell from Topic 52 with two concurrent orders
for one unit of stock. Fix it three ways: (i) `@Version` plus a retry loop,
(ii) `PESSIMISTIC_WRITE`, (iii) the atomic conditional `UPDATE`. Measure throughput and
p99 for each under a flash-sale profile (200 concurrent orders on one SKU). Then answer:
under what condition would you choose (ii) despite it being the slowest?

**Part E — raise the isolation level and pay for it.** Set the placement transaction to
`SERIALIZABLE`. Re-run the flash-sale profile. Report the rate of
`CannotSerializeTransactionException`, the retry rate after you add `@Retryable`, and the
effect on p99. Then write one paragraph arguing that the atomic `UPDATE` at
`READ COMMITTED` was the better engineering decision — or, if your numbers say otherwise,
say so and show them.

**Part F — the seatbelts.** Add `connection-timeout`, `leak-detection-threshold`,
`statement_timeout` and `idle_in_transaction_session_timeout`. Deliberately reintroduce
the Part B bug and show that each of the four fires, naming which one catches it first
and why that ordering is the one you want.

> Every timing here comes from the load harness, not a stopwatch. If you find yourself
> writing `System.nanoTime()`, re-read the Measurement section.

---

## Interview questions

### Q1 — "Your service is returning 500s on an endpoint that doesn't touch the database. Where do you look?"

**Mid-level answer:** "Check the logs and the health endpoint. Maybe the service is out
of memory or the database is down."

**Senior answer:** "My first hypothesis is connection-pool exhaustion causing thread
starvation, because that's the failure mode that makes *unrelated* endpoints fail. The
chain is: some transaction somewhere is holding a connection while waiting on something
slow — almost always an HTTP call inside a `@Transactional` method. The pool drains,
Tomcat threads park in `HikariPool.getConnection`, and once the thread pool is exhausted
there's nobody left to serve any request, including ones that need no connection. I
confirm it in three checks: `hikaricp.connections.pending` — if that's above zero,
threads are blocked; `pg_stat_activity` — if I see sessions `idle in transaction` with a
growing age while Postgres CPU is near zero, the database is not the problem and the
application is holding connections to do nothing; and a thread dump, where I'd expect
most request threads parked in `getConnection`. To recover I'd terminate the long
transactions and, if it's a downstream, circuit-break it. The fix is moving the external
call outside the transaction boundary, plus `leak-detection-threshold` and a
`connection-timeout` inside the request SLO so next time it fails fast and names itself."

**What separates them:** naming the specific failure mode from the *shape* of the
symptom, a three-signal confirmation procedure, separating recovery from fix, and
knowing that the pool metric and the database view answer different questions.

**Follow-up:** "You've confirmed pool exhaustion. Why not just raise the pool size?"
*(Because a pool timeout is caused by long transactions, not a small pool; Postgres is
process-per-connection so more connections make latency worse for everyone; and the
queue just moves from Hikari, where you have a metric, into Postgres, where you do not.)*

---

### Q2 — "Which isolation level do you use, and why?"

**Mid-level answer:** "`READ_COMMITTED`. It's the default and it prevents dirty reads."

**Senior answer:** "`READ COMMITTED`, which on Postgres is the default and is a
statement-level snapshot — and dirty reads aren't the reason, because Postgres has no
dirty reads at any level. `READ UNCOMMITTED` is silently mapped to `READ COMMITTED`. The
real question is which invariants aren't safe at statement-level snapshots, and for us
that's read-modify-write on inventory. My first move there isn't a higher isolation
level, it's a better-shaped statement: a single atomic `UPDATE ... SET available =
available - :qty WHERE sku = :sku AND available >= :qty`, checking the row count. That's
correct at `READ COMMITTED` with no retries, because the guard is evaluated by Postgres
under the row lock the update already takes. I'd only raise the level for an invariant
that genuinely spans statements — a monthly payout asserting a sum, where a phantom would
corrupt it — and then `REPEATABLE READ`, which on Postgres is real snapshot isolation and
also prevents phantoms. `SERIALIZABLE` adds SSI predicate locks and can abort a
transaction over what it *read*, which means a mandatory retry loop with jitter and a
metric. Any isolation increase ships with its retry loop in the same commit, or it's a
new source of 500s."

**What separates them:** knowing Postgres's actual implementation rather than the ANSI
table, preferring statement shape to isolation level, and treating retry as part of the
level rather than an afterthought.

**Follow-up:** "You raised a path to `SERIALIZABLE` and error rate went up under load.
Why, and what do you do?" *(SSI aborts increase with concurrency, and the higher
isolation also lengthens transactions, which increases concurrency — a feedback loop. Fix
by narrowing what the transaction reads, or by collapsing the invariant into one
statement.)*

---

### Q3 — "How long should a transaction be?"

**Mid-level answer:** "As short as possible."

**Senior answer:** "Short enough that the pool isn't the bottleneck, and I can compute
the number rather than assert it. Little's Law: max throughput equals pool size divided
by transaction duration. In `orderflow` that's a 20-connection pool per replica serving
20 placements a second, so at 2× headroom a transaction may hold a connection for at most
500 ms, and I target a p99 of 150 ms. That number is a review criterion — anything that
could push past it is a design change. The corollary is that transaction duration isn't a
latency concern, it's a concurrency limit shared by every request in the JVM, which is
why one slow endpoint takes down all of them. I measure it with
`hikaricp.connections.usage` p99 rather than a stopwatch, because a stopwatch around the
call includes pool queueing and misses the commit, and one sample can't describe a p99."

**What separates them:** arithmetic instead of a platitude, the "concurrency limit not
latency" reframe, and knowing which metric to trust and why.

**Follow-up:** "Your p99 transaction duration is 400 ms and Little's Law says you need
more connections than Postgres can serve. What now?" *(Reduce duration — move work out,
fix N+1s, batch — before touching the pool. If duration genuinely cannot come down,
that's a sharding or read-replica conversation, not a pool-size one.)*

---

### Q4 — "What does `REPEATABLE READ` mean on Postgres specifically?"

**Mid-level answer:** "Re-reading the same row inside the transaction gives you the same
value."

**Senior answer:** "It's snapshot isolation. One snapshot taken at the first statement
serves the whole transaction, so every read sees a consistent point-in-time view. That's
strictly stronger than the ANSI definition — ANSI's `REPEATABLE READ` still permits
phantoms and Postgres's doesn't, because the snapshot covers rows that appear as well as
rows that change. The price is on writes: if you update a row that a concurrent
transaction committed after your snapshot, Postgres aborts you with `40001 could not
serialize access due to concurrent update`. That's not a bug, it's the mechanism, and it
means every `REPEATABLE READ` transaction needs a retry that re-reads — retrying with the
same in-memory data reproduces the conflict. Spring translates the SQLSTATE into
`CannotSerializeTransactionException`, and the retry has to sit in a different bean or
it won't be outside the transaction, because of the proxy rule from Topic 54. What it
does *not* give you is serializability — for write skew you need `SERIALIZABLE` and SSI's
predicate locks."

**What separates them:** knowing Postgres exceeds the standard here, the write-conflict
failure mode, and the retry-must-re-read detail.

**Follow-up:** "Give me a concrete write skew that `REPEATABLE READ` doesn't prevent."
*(The classic: two transactions each check "at least one on-call engineer remains" and
each removes a different one. Both snapshots see two; both writes touch different rows,
so there is no write-write conflict; the invariant is violated. Needs `SERIALIZABLE` or
an explicit lock.)*

---

### Q5 — "Why doesn't `readOnly = true` stop writes?"

**Mid-level answer:** "It's just a hint to the database."

**Senior answer:** "It does three things, and none of them is 'prevent writes'. It sets a
flag in `TransactionSynchronizationManager`; it sets Hibernate's flush mode to `MANUAL`,
so dirty-checked changes to managed entities are **silently discarded** rather than
rejected; and if the transaction manager propagates it, it calls
`Connection.setReadOnly(true)`, which on Postgres issues `SET TRANSACTION READ ONLY` and
*will* reject the write hard. So the same mistake either vanishes silently or throws,
depending on configuration — the worst kind of ambiguity. I treat it as a performance and
intent hint: skipping the flush is a real saving on a read path with a large persistence
context, and it's the standard signal for routing to a read replica. For an actual
guarantee I use a database role without write permission. The structural fix is that read
paths return projections, not managed entities — then there's nothing to dirty-check and
the question doesn't arise."

**What separates them:** all three effects, the silent-versus-hard failure split, and a
structural fix rather than a rule to remember.

**Follow-up:** "How would you find every place in a codebase where this has already
happened?" *(You largely cannot from the logs, because the failure is silent — which is
the argument. What you can do is grep for entity mutation inside `readOnly` methods with
a static analysis rule, and enforce projections on read paths in review.)*

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Little's Law says throughput equals pool size over transaction duration. Both
   variables are under your control. Give a concrete `orderflow` situation where
   increasing pool size is right, and one where reducing duration is the only option —
   and state the signal that tells them apart.

2. Postgres maps `READ UNCOMMITTED` to `READ COMMITTED`. Why is that mapping *safe* for
   the standard, and what does it tell you about how MVCC differs from lock-based
   isolation?

3. A colleague says "we'll use `SERIALIZABLE` everywhere and stop thinking about
   concurrency." Give three independent reasons that fails — one about correctness, one
   about throughput, one about operations.

4. A read-only transaction can be aborted by SSI at `SERIALIZABLE`. Explain how a
   transaction that wrote nothing can be part of a dangerous dependency structure.

5. A transaction holds one connection. A `REQUIRES_NEW` inside it holds two. Under what
   pool size and concurrency does that become a deadlock rather than merely slow? Derive
   the bound, and then say why the derivation is usually the wrong thing to act on.

6. Your Node experience is that a slow downstream degrades one route. In Java it can
   degrade all of them. Name the two independent mechanisms that make this so — and say
   what virtual threads (Topic 101) do and do not change about each.

7. A long-running read transaction blocks vacuum and bloats a table. Explain why this is
   a *correctness-adjacent* problem and not merely a performance one, in terms of what
   bloat does to query plans.

---

## Quick reference card

### Isolation on Postgres

| `@Transactional(isolation = ...)` | Postgres gives you | Fails with |
|---|---|---|
| `DEFAULT` | whatever the connection has — `READ COMMITTED` for Boot + Postgres | — |
| `READ_UNCOMMITTED` | **`READ COMMITTED`** (mapped; no dirty reads exist) | — |
| `READ_COMMITTED` | new snapshot per statement | — (lost updates possible) |
| `REPEATABLE_READ` | snapshot isolation for the whole transaction; no phantoms | `40001` on write-write conflict |
| `SERIALIZABLE` | snapshot isolation + SSI predicate locks | `40001` on read/write dependency cycles |

Spring's exception translation: `40001` → `CannotSerializeTransactionException`;
`40P01` (deadlock) → `CannotAcquireLockException`. Both are unchecked
`ConcurrencyFailureException`s. Retry both.

### The connection lifecycle

```
BEGIN  -> connection checked out from Hikari, bound to the thread's ThreadLocal
   ... every line of your method runs with it held ...
COMMIT -> flush, WAL write, unbind, connection returned to the pool
```

### Configuration that matters

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20            # from Little's Law and Postgres capacity, not a guess
      connection-timeout: 2000         # inside your request SLO. NOT the 30 s default.
      leak-detection-threshold: 5000   # logs a stack trace for any long-held connection
      max-lifetime: 1800000            # shorter than any proxy/firewall idle timeout
      validation-timeout: 1000
```

```sql
ALTER ROLE orderflow_app SET statement_timeout = '5s';
ALTER ROLE orderflow_app SET idle_in_transaction_session_timeout = '10s';
ALTER ROLE orderflow_app SET lock_timeout = '2s';
```

### Diagnostics, in order

```bash
curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending   # blocked threads
curl -s localhost:8080/actuator/metrics/hikaricp.connections.active
curl -s localhost:8080/actuator/metrics/hikaricp.connections.usage     # hold duration
jcmd <pid> Thread.print | grep -A 8 'HikariPool.getConnection'
```

```sql
SELECT pid, state, now()-xact_start AS age, wait_event, left(query,60)
FROM pg_stat_activity WHERE datname='orderflow' AND state <> 'idle'
ORDER BY xact_start;
```

### Symptom → cause table

| Symptom | Most likely cause |
|---|---|
| Unrelated endpoints returning 500 | Pool exhaustion → thread starvation. Something slow inside a transaction. |
| `idle in transaction` with a growing age | The application is holding a connection while not talking to the database |
| `hikaricp.connections.pending` > 0 | Threads blocked on the pool. Leading indicator of an outage. |
| `40001` under load | `REPEATABLE READ` or `SERIALIZABLE` with no retry loop |
| Table bloat, autovacuum never completing | A long-running transaction holding an old snapshot |
| `FATAL: too many clients already` | Pool size × replicas exceeds `max_connections` |
| Latency worse after raising the pool | Postgres process-per-connection contention. Topic 109. |

### Rules

- The transaction contains database work and **nothing else**.
- Compute the duration budget from Little's Law; defend it in review.
- Prefer a correctly-shaped statement to a higher isolation level.
- Every isolation increase ships with its retry loop in the same commit.
- `readOnly = true` is a hint, never a guarantee.
- Alert on `pending`, not on `active`.
- A bigger pool is almost never the fix.

> `[BOOT 3.x DELTA]` HikariCP is the default pool and the metric names
> (`hikaricp.connections.*`) are unchanged from Boot 2.x through 4.x, so everything
> operational in this doc applies verbatim on 3.5. The differences are elsewhere:
> outbound HTTP client APIs moved (`RestTemplate` → `RestClient`, and
> `ClientHttpRequestFactorySettings` changed shape between Framework 6.x and 7.0), and
> Boot 3.5 left OSS support in June 2026, so a 3.x service gets no free CVE patches.
> Verify the exact client-builder API against your version's reference docs.

---

## When would I use this at work?

**1. In an incident, in the first two minutes.**
Alerts fire on endpoints that have nothing in common. You go straight to
`hikaricp.connections.pending` and `pg_stat_activity`. If Postgres CPU is low and
sessions are `idle in transaction`, you have both diagnosed the class of problem and
ruled out the database — which is usually where the first twenty minutes of an incident
goes. Recovery is terminating the long transactions or circuit-breaking the downstream;
the fix comes later.

**2. In design review, before the code exists.**
Somebody proposes "call the fraud service, then reserve inventory, in one transaction so
it's atomic". You do not say "that's bad practice". You say: the fraud service p99 is
180 ms, our pool is 20, we do 20 placements per second per replica, Little's Law gives us
a 500 ms budget and this design spends 180 ms of it on something that cannot participate
in the transaction anyway — and atomicity across a network boundary was never available,
so we need an idempotency key and a compensating action regardless. That is a different
conversation, and it ends differently.

**3. Choosing an isolation level for a new invariant.**
A new feature needs "the sum of these rows must match the total we write". Instead of
reaching for `SERIALIZABLE`, you ask whether the invariant can be expressed in one
statement. If it can, you keep `READ COMMITTED` and add no retry logic. If it cannot, you
choose `REPEATABLE READ`, ship the retry loop with jitter and a metric in the same
commit, and write down what the retry rate should be so a rise is detectable. Both
outcomes are defensible; the difference is that you decided rather than defaulted.

---

## Connected topics

**Prerequisites:**
- **54 — `@Transactional` I**: the proxy, propagation, rollback rules, and where the
  transaction boundary actually is. This topic assumes all of it.
- **48 — Persistence context**: flush timing determines when the writes actually hit the
  connection, which determines how long it is held.
- **52 — Locking**: `@Version`, `PESSIMISTIC_WRITE` and the atomic conditional `UPDATE`
  are the alternatives to raising the isolation level.
- **47 — Spring Data JPA**: projections keep read transactions short and keep entities
  out of read paths.
- **40 — Proxying**: still applies to every split-transaction refactor in this doc —
  splitting a method into two transactions only works if the calls cross a bean boundary.
- **09 — Exception API design**: Spring's `DataAccessException` hierarchy is unchecked,
  which is what makes `CannotSerializeTransactionException` catchable at the right layer.

**This unlocks:**
- **65 — Load baseline**: you cannot defend a duration budget without one. The failure
  drill in this topic uses Topic 65's harness.
- **77 — JMH**: the correct way to measure anything microbenchmark-shaped, and the full
  argument against the stopwatch.
- **90 — Executors**: bounded pools, and why handing transactional work to a pool thread
  breaks the `ThreadLocal` binding.
- **101 — Virtual threads**: raise the thread ceiling and the connection pool becomes the
  visible bottleneck immediately. Virtual threads do not create connections.
- **109 — HikariCP deadlock**: pool sizing arithmetic, the latency-versus-size curve, and
  the pool-vs-thread-pool deadlock in full.
- **111 — Circuit breakers**: the correct response to a downstream that is slow rather
  than down.
- **115 — Outbox**: how to make "publish an event atomically with a write" a database
  write, so it can live inside the boundary.
- **116 — Idempotency**: what you owe the system once you split one transaction into two
  around a network call.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0, Jakarta EE 11,
HikariCP, Postgres. The isolation semantics here are Postgres facts and are stable across
Spring versions; the Hikari metric names are stable from Boot 2.x onward. The one
genuinely version-sensitive item flagged inline is the outbound HTTP client builder API,
which you should verify against your Framework version's reference documentation rather
than trusting a method name from any document.*
