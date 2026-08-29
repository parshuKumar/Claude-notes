# 54 — `@Transactional` I — Proxy Semantics, Propagation, Rollback Rules, Read-Only

## Phase: 5 — Spring Boot & Persistence
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: transactional order placement — reserve inventory, debit wallet, create payment, publish event, all-or-nothing

---

## Mechanical statement

> `@Transactional` is Topic 40's proxy plus a `PlatformTransactionManager`.
> The proxy begins a transaction, invokes your method, and commits or rolls back.
> **It only ever sees calls that arrive *through* it.**

Read that third sentence again. Everything difficult about this topic is a consequence
of it. The annotation is not a compiler feature. It is not a keyword. It is metadata
that a *different object* reads at runtime, and if the call never reaches that other
object, the metadata is inert.

Three corollaries you should be able to state cold:

1. A call from inside your own class (`this.doSomething()`) never touches the proxy,
   so it is never transactional.
2. The proxy decides commit-versus-rollback from the exception that escapes your
   method. If nothing escapes, it commits. If a **checked** exception escapes, the
   default rules still say *commit*.
3. The transaction manager binds a database connection to the current thread for the
   whole duration. That binding is the subject of Topic 55.

---

## The bridge from what you know

### `@Transactional` has no Nest analogue worth leaning on

Be honest with yourself here, because pattern-matching from Nest will actively mislead
you.

In NestJS with TypeORM, a transaction is something you *hold*:

```ts
// Nest / TypeORM — the transaction is an object in your hand
async placeOrder(cmd: PlaceOrderCommand) {
  const queryRunner = this.dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();
  try {
    await queryRunner.manager.save(order);
    await queryRunner.manager.decrement(Inventory, { sku }, 'available', qty);
    await queryRunner.commitTransaction();
  } catch (e) {
    await queryRunner.rollbackTransaction();
    throw e;
  } finally {
    await queryRunner.release();
  }
}
```

Look at what that code gives you for free:

- The transaction is a **local variable**. You can see it.
- Every write goes through `queryRunner.manager`. If you accidentally used
  `this.orderRepo` instead, the write would obviously be outside the transaction —
  it is a different object, and it is right there in the source.
- `commit` and `rollback` are calls you wrote. There is no rule to learn about which
  exception types trigger which.

Spring's version:

```java
@Transactional
public OrderId placeOrder(PlaceOrderCommand cmd) {
    orderRepository.save(order);
    inventoryRepository.reserve(cmd.sku(), cmd.quantity());
    return order.id();
}
```

Shorter. Also: the transaction is invisible, the boundary is invisible, whether the
boundary *applied* is invisible, and the rollback rule is a default you did not choose.

**Say it plainly: the annotation is a promise made by an object you do not directly
hold.** Somebody else — the proxy — has to keep that promise on your behalf, and there
are several ordinary-looking ways to stop the proxy from ever being asked.

**Verdict: NO USEFUL ANALOGUE.** The nearest thing in your Nest experience is a
`@UseInterceptors` interceptor that opens a transaction around the handler — and even
that runs at the *route* boundary, where self-invocation cannot happen, so it does not
teach you the failure mode.

### What *does* transfer

| From your background | In Spring | Verdict |
|---|---|---|
| SQL `BEGIN` / `COMMIT` / `ROLLBACK` semantics | Identical. Spring emits exactly these. | **FULL** |
| Knowing a transaction holds locks until commit | Identical, and now it also holds a *pool connection* | **FULL** — Topic 55 |
| TypeORM `@Transaction()` decorator (deprecated) | Closest shape, same proxy hazard | **PARTIAL** |
| `dataSource.transaction(cb)` callback style | Spring's `TransactionTemplate` — the programmatic escape hatch | **FULL** |
| Nest interceptors run after routing, on the handler | Spring's proxy runs on *any* bean method call | **PARTIAL** — the surface is much larger |

If you ever want the Nest feel back, Spring gives you `TransactionTemplate`. It is not
a fallback for beginners; it is the correct tool whenever the boundary needs to be
narrower than a whole method. You will use it in Example 2.

### The one habit to unlearn

In Nest you ask "did I pass the query runner?" In Spring you must ask **"did this call
come through the proxy?"** — and the answer is not visible in the method you are
reading. It depends on the *caller*.

---

## What is this?

`@Transactional` is an annotation from `org.springframework.transaction.annotation`.
Putting it on a public method of a Spring bean tells Spring: wrap every call to this
method in a database transaction.

The machinery has four named parts. Learn the names; interviewers use them.

| Part | What it is |
|---|---|
| `@Transactional` | Metadata. Does nothing by itself. |
| `TransactionInterceptor` | The `MethodInterceptor` (Topic 41) sitting in the proxy's advice chain. This is the code that actually runs. |
| `PlatformTransactionManager` | The strategy that knows *how* to begin/commit/roll back for a given technology. `JpaTransactionManager` for JPA, `DataSourceTransactionManager` for plain JDBC. |
| `TransactionSynchronizationManager` | A holder of `ThreadLocal`s. This is where "the current transaction" actually lives. |

The interceptor's logic, in pseudocode, is roughly:

```
TransactionInterceptor.invoke(call):
    attrs = read @Transactional metadata for this method
    txInfo = transactionManager.getTransaction(attrs)     // may begin, join, or suspend
    try:
        result = call.proceed()                            // <- YOUR method body
    catch (Throwable t):
        if attrs.rollbackOn(t):
            transactionManager.rollback(txInfo)
        else:
            transactionManager.commit(txInfo)              // <- yes. commit. read on.
        throw t
    transactionManager.commit(txInfo)
    return result
```

That `else: commit` branch is the single most expensive line of Spring semantics in
this entire curriculum. It is Trap 2 below.

### The four things you configure

```java
@Transactional(
    propagation = Propagation.REQUIRED,      // what to do if a transaction already exists
    isolation   = Isolation.DEFAULT,         // Topic 55
    readOnly    = false,                     // a hint + a Hibernate flush-mode change
    timeout     = -1,                        // seconds; -1 = the manager's default
    rollbackFor = {},                        // extra exception types that cause rollback
    noRollbackFor = {}                       // exception types that must NOT roll back
)
```

Propagation, rollback rules and `readOnly` are this topic. Isolation is Topic 55.

---

## Why does it matter?

Three reasons, in increasing order of career impact.

**1. It is the most-asked Spring question in senior interviews.**
"Why didn't my `@Transactional` work?" has a short list of causes, and being able to
walk it in thirty seconds signals that you have debugged real Spring systems.

**2. Its failure mode is silent partial writes.**
When a `NullPointerException` escapes, you get a stack trace and an alert. When
`@Transactional` silently does not apply, you get an order row with no inventory
reservation, a wallet debited with no payment record, and no error anywhere. You find
it days later in a reconciliation report — which, given your SQL background, you will
recognise as the worst class of bug there is.

**3. Transaction boundaries are a capacity decision, not a correctness decision.**
Every second a transaction is open is a second a pool connection is unavailable to
anyone else. Topic 55 makes that the whole subject; you need the boundary rules first.

---

## Machine-level reality

This section is what separates you from someone who has memorised the annotation.

### Where "the current transaction" actually lives

There is no `Transaction` object passed around. There is a class of static
`ThreadLocal` fields:

```java
// org.springframework.transaction.support.TransactionSynchronizationManager
// (field names are from the framework; treat the exact set as illustrative of the
//  shape, and read the class source on your version to confirm)
private static final ThreadLocal<Map<Object, Object>> resources;
private static final ThreadLocal<Boolean> actualTransactionActive;
private static final ThreadLocal<String>  currentTransactionName;
private static final ThreadLocal<Integer> currentTransactionIsolationLevel;
private static final ThreadLocal<Boolean> currentTransactionReadOnly;
private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations;
```

The `resources` map is the important one. When a JPA transaction begins,
`JpaTransactionManager` does approximately this:

1. Ask the `EntityManagerFactory` for a new `EntityManager` (the persistence context —
   Topic 48).
2. Call `entityManager.getTransaction().begin()`, which pulls a JDBC `Connection` from
   HikariCP and issues `BEGIN`.
3. Wrap that in a holder and bind it:
   `TransactionSynchronizationManager.bindResource(entityManagerFactory, emHolder)`.
   For plain JDBC, `DataSourceTransactionManager` binds a `ConnectionHolder` keyed by
   the `DataSource`.
4. Set `actualTransactionActive = true`, plus the name/isolation/read-only flags.

Now, when your repository calls `EntityManagerFactoryUtils.doGetTransactionalEntityManager(...)`
— which every Spring Data repository does under the covers — it looks in that same
`ThreadLocal` map, finds the bound `EntityManager`, and uses it. That is the entire
mechanism by which "the repository joins the transaction" works.

**Three consequences fall straight out of this:**

- **Thread-bound means thread-fragile.** Hand work to another thread (`@Async`, a raw
  `ExecutorService`, a `CompletableFuture` continuation, a Reactor operator) and the
  new thread has an empty `TransactionSynchronizationManager`. Any repository call it
  makes runs in its own auto-commit transaction, or fails. Forward: Topics 90, 91, 101,
  119, 120 all rediscover this same `ThreadLocal` boundary from different angles.
- **You can interrogate it.** `TransactionSynchronizationManager.isActualTransactionActive()`
  is a plain static boolean read. It is the cleanest possible proof that a transaction
  exists at a given line of code. You will use it in the Hands-on proof.
- **One thread, one bound connection.** Which is Topic 55's mechanical statement.

### What each propagation level does to that binding

Assume method `outer()` calls (through a proxy) method `inner()`.

| Propagation on `inner` | If no transaction exists | If `outer`'s transaction exists | Connections held |
|---|---|---|---|
| `REQUIRED` (default) | begin a new one | **join it** — same connection, same persistence context | 1 |
| `REQUIRES_NEW` | begin a new one | **suspend** `outer`'s (unbind its resources, stash them), begin a genuinely independent one, restore on exit | **2 simultaneously** |
| `NESTED` | begin a new one | keep the same transaction; take a **JDBC savepoint** | 1 |
| `SUPPORTS` | run with no transaction | join it | 0 or 1 |
| `NOT_SUPPORTED` | run with no transaction | suspend it, run non-transactionally, restore | 1 (held, idle) |
| `MANDATORY` | throw `IllegalTransactionStateException` | join it | 1 |
| `NEVER` | run with no transaction | throw `IllegalTransactionStateException` | 0 |

Two rows deserve elaboration.

**`REQUIRED` joining is why "rollback-only" exists.** When `inner` joins `outer`'s
transaction and then throws, the interceptor around `inner` cannot roll back — it does
not own the transaction. So it marks it **rollback-only**. If `outer` then catches the
exception and returns normally, `outer`'s interceptor tries to commit, discovers the
rollback-only flag, and throws `UnexpectedRollbackException`. That exception is not a
bug; it is Spring telling you that you swallowed something that had already doomed the
transaction.

**`NESTED` is not `REQUIRES_NEW`.** This distinction is a reliable senior/mid
discriminator.

| | `REQUIRES_NEW` | `NESTED` |
|---|---|---|
| Mechanism | Suspend outer, `BEGIN` a second physical transaction | `SAVEPOINT sp1` inside the *same* physical transaction |
| Connections | **two**, held at the same time | **one** |
| Inner commits, outer rolls back | Inner's work **survives** | Inner's work is **rolled back with the outer** |
| Inner rolls back | Outer unaffected | `ROLLBACK TO SAVEPOINT sp1`; outer continues |
| Sees outer's uncommitted writes | **No** — it is a separate transaction with its own snapshot | **Yes** — same transaction |
| Requires | Any transaction manager | JDBC savepoint support; on JPA, `JpaTransactionManager` does not support it by default |

The savepoint version emits, in order: `SAVEPOINT`, your statements, then either
`RELEASE SAVEPOINT` or `ROLLBACK TO SAVEPOINT`. Postgres supports savepoints fully.
The caveat is the JPA one: `JpaTransactionManager` reports that it does not support
savepoints unless you are on `DataSourceTransactionManager` or have configured nested
transaction support explicitly. If you ask for `NESTED` and get an exception at
runtime, that is why — not a typo.

### How the proxy is created, and why it can be absent

Recap from Topic 40 rather than a re-teach: at
`BeanPostProcessor.postProcessAfterInitialization` (Topic 37's lifecycle step), the
`InfrastructureAdvisorAutoProxyCreator` inspects each bean. If any method carries
`@Transactional`, the bean is replaced in the container by a proxy — a CGLIB subclass
for a class-based bean, a JDK dynamic proxy for an interface-based one. Callers who
`@Autowired` that bean receive the proxy.

From Topic 40 you already know the consequences. Here they are, mapped onto this topic:

| Topic 40 fact | What it means for `@Transactional` |
|---|---|
| The proxy is a *different object* than your bean | The annotation is read by the proxy, not by your code |
| `this.method()` bypasses the proxy | Self-invocation is not transactional |
| CGLIB overrides non-final methods | `final` methods cannot be advised — silently |
| Proxies cannot override `private` | `@Transactional private` does nothing — silently |
| Proxies exist only after post-processing | A `@Transactional` call from `@PostConstruct` is not transactional |
| `getClass().getName()` shows `$$SpringCGLIB$$` | Your one-line proof that proxying happened at all |

**Spring Boot 4 / Framework 7 note:** Framework 6.0 added `@Transactional` support on
*interface default methods* and improved handling in some cases, and Boot 4 defaults to
CGLIB proxying (`spring.aop.proxy-target-class=true`) as it has since Boot 2. None of
that changes the self-invocation rule. Do not expect a version to rescue you from it.

### What `readOnly = true` actually does

This is misunderstood constantly, so be precise. `readOnly = true` does **three**
things, and *none* of them is "prevent writes".

1. **It sets a flag** in `TransactionSynchronizationManager.currentTransactionReadOnly`.
   Spring itself does not enforce anything with it.
2. **Hibernate sets `FlushMode.MANUAL`** on the session for that transaction. This is
   the real, measurable effect: dirty checking still records snapshots, but no
   automatic flush happens, so your modifications are **silently discarded**. Note
   carefully: discarded, not rejected. No exception.
3. **The JDBC connection is set read-only** — `Connection.setReadOnly(true)` — *if* the
   transaction manager is configured to propagate it. On Postgres this issues
   `SET TRANSACTION READ ONLY`, and Postgres **will** then reject a write with
   `ERROR: cannot execute INSERT in a read-only transaction`.

So whether a write throws or is silently swallowed depends on whether the read-only
flag reached the connection. That is exactly the wrong kind of ambiguity to rely on.

**The honest rule:** treat `readOnly = true` as a *performance and intent* hint —
skipping flushes on a read path is a genuine saving when the persistence context holds
thousands of entities — and never as a safety guarantee. If you want a guarantee, use a
database role that lacks write permission.

There is a secondary benefit worth knowing for interviews: with a replica-aware routing
`DataSource`, `readOnly = true` is the standard signal used to route the transaction to
a read replica.

### How Postgres implements the transaction underneath

Your MVCC knowledge transfers directly, so this is a mapping exercise rather than new
material:

- `BEGIN` assigns nothing yet. Postgres defers the transaction ID.
- The first *write* statement allocates an `xid` and stamps `xmin` on the new row
  version.
- `COMMIT` writes a commit record to WAL and marks the `xid` committed in `pg_xact`.
  From that instant every new snapshot sees the row.
- `ROLLBACK` marks the `xid` aborted. The dead tuples remain on the heap until
  `VACUUM`. This is why a high rollback rate is not free: it produces bloat.
- `SAVEPOINT` allocates a **subtransaction** with its own `xid`. Postgres tracks these
  in shared memory, and more than 64 subtransactions per transaction spills into
  `pg_subtrans` on disk — a documented performance cliff. Do not build a loop that
  takes one `NESTED` savepoint per order line.

---

## Example 1 — minimal

Two beans, one write, one deliberate failure. The point is to see the boundary, not to
model a domain.

```java
package com.orderflow.wallet;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.support.TransactionSynchronizationManager;

@Service
public class WalletService {

    @PersistenceContext
    private EntityManager em;

    @Transactional
    public void debit(long walletId, long amountMinor) {
        System.out.println("active tx? "
            + TransactionSynchronizationManager.isActualTransactionActive());
        System.out.println("tx name  : "
            + TransactionSynchronizationManager.getCurrentTransactionName());

        Wallet wallet = em.find(Wallet.class, walletId);
        wallet.debit(amountMinor);          // dirty checking will UPDATE at flush (Topic 48)

        if (amountMinor > wallet.balanceMinor()) {
            throw new InsufficientFundsException(walletId);   // extends RuntimeException
        }
    }
}
```

Reading it line by line, in the order things happen at runtime:

| Line | What actually happens |
|---|---|
| `walletService.debit(...)` from another bean | The call hits the **proxy**, not this object |
| interceptor before `debit` | `getTransaction()` → new `EntityManager`, `BEGIN`, bind to `ThreadLocal` |
| `isActualTransactionActive()` | reads the `ThreadLocal` — prints `true` if and only if the proxy was used |
| `getCurrentTransactionName()` | the fully-qualified method name Spring assigned; a *named* transaction is strong evidence |
| `em.find` | joins the bound persistence context; no new connection |
| `wallet.debit(...)` | mutates a managed entity; **no `save()` call needed** (Topic 48) |
| `throw new InsufficientFundsException` | unchecked → default rules → **rollback** |
| interceptor after | `rollback()`, unbind, close the `EntityManager` |

Now the same method with one word changed:

```java
    // InsufficientFundsException now extends Exception, not RuntimeException
    @Transactional
    public void debit(long walletId, long amountMinor) throws InsufficientFundsException {
        ...
        throw new InsufficientFundsException(walletId);       // CHECKED
    }
```

The debit **commits**. The exception propagates to your caller, your controller returns
a 4xx, your logs show the failure — and the money is gone. Nothing anywhere reports a
problem. This is Trap 2, and it is why this topic is a DIFFERENTIATOR.

---

## Example 2 — production scenario on the `orderflow` spine

### The requirement

`POST /orders` places an order. In one atomic step it must:

1. Reserve inventory for every line (conditional `UPDATE`, Topic 52).
2. Debit the customer's wallet.
3. Create a `Payment` row in `PENDING`.
4. Write an audit record that must survive **even if the order fails**.
5. Publish an `OrderPlaced` event that must fire **only if the order succeeds**.

Real constraints, so the design decisions have teeth:

| Constraint | Value |
|---|---|
| Peak order rate | 120 orders/sec |
| Average lines per order | 3.4 |
| Hot products | top 50 SKUs take 60% of volume |
| Hikari pool | `maximum-pool-size: 20` (Topic 109 will justify this) |
| Postgres | 8 vCPU, `max_connections = 200` across 6 service replicas |
| SLO | `POST /orders` p99 ≤ 400 ms, error rate ≤ 0.1% |

At 120 orders/sec with a 20-connection pool, Little's Law gives you a budget: a
transaction may hold a connection for at most `20 / 120 ≈ 166 ms` on average before the
pool is the bottleneck. That number is the reason every decision below goes the way it
does.

### The service

```java
package com.orderflow.orders;

import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderPlacementService {

    private final OrderRepository orders;
    private final InventoryService inventory;
    private final WalletService wallets;
    private final PaymentRepository payments;
    private final OrderAuditService audit;
    private final ApplicationEventPublisher events;

    OrderPlacementService(OrderRepository orders,
                          InventoryService inventory,
                          WalletService wallets,
                          PaymentRepository payments,
                          OrderAuditService audit,
                          ApplicationEventPublisher events) {
        this.orders = orders;
        this.inventory = inventory;
        this.wallets = wallets;
        this.payments = payments;
        this.audit = audit;
        this.events = events;
    }

    @Transactional(
        timeout = 3,                                   // seconds. See below.
        rollbackFor = InsufficientFundsException.class // CHECKED exception. See below.
    )
    public OrderId place(PlaceOrderCommand cmd) throws InsufficientFundsException {

        Order order = Order.newFor(cmd.customerId());

        for (OrderLineCommand line : cmd.lines()) {
            inventory.reserve(line.sku(), line.quantity());   // conditional UPDATE
            order.addLine(line.sku(), line.quantity(), line.unitPriceMinor());
        }

        wallets.debit(cmd.customerId(), order.totalMinor());  // may throw CHECKED

        Payment payment = Payment.pendingFor(order);
        payments.save(payment);
        orders.save(order);

        events.publishEvent(new OrderPlaced(order.id(), order.totalMinor()));
        return order.id();
    }
}
```

Five decisions in that method, each with a reason:

**1. `rollbackFor = InsufficientFundsException.class`.**
`InsufficientFundsException` is checked, because the caller has a genuine alternative
action — offer a top-up (Topic 09's rule for when checked is justified). Checked means
the default rules would **commit**. `rollbackFor` restores the behaviour any reader
would assume. Without this one attribute, an insufficient-funds order leaves the
inventory reserved forever.

**2. `timeout = 3`.**
The SLO is a 400 ms p99. A three-second cap is not a performance tuning knob; it is a
blast-radius cap. A transaction that runs long enough to breach it is already a failed
request, and holding the connection past that point only spreads the damage. Spring
enforces this via the JDBC statement timeout and by refusing to commit past the
deadline.

**3. Audit is `REQUIRES_NEW` — deliberately, and with the cost named.**

```java
@Service
public class OrderAuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordAttempt(OrderAttempt attempt) {
        auditRepository.save(attempt);      // must survive the outer rollback
    }
}
```

This is the *only* legitimate reason to use `REQUIRES_NEW`: the write must survive when
the surrounding transaction fails. And you pay for it — while `recordAttempt` runs, the
request thread holds **two** connections from a 20-connection pool. With 20 concurrent
placements, all 20 threads hold one connection each and all 20 wait for a second one.
Nobody can proceed. That is the pool-vs-pool deadlock of Topic 109, and this annotation
is how you build one.

At pool size 20 and 120 orders/sec, the safe bound is
`pool >= threads × (connections_per_thread − 1) + 1`. With 200 Tomcat threads and 2
connections per thread, that bound is 201 — which you cannot afford against Postgres.
So the honest resolution in `orderflow` is **not** `REQUIRES_NEW` at all: write the
audit row through the outbox pattern (Topic 115), or accept that audit is best-effort
and write it *after* the transaction. `REQUIRES_NEW` stays in the doc because you must
be able to recognise it in someone else's code and say why it is dangerous.

**4. `publishEvent` inside the transaction, consumed after commit.**

`ApplicationEventPublisher.publishEvent` inside a transaction is *synchronous by
default* — the listener runs on the same thread, inside the same transaction, before
`place` returns. That is almost never what you want for an event that triggers an email
or a Kafka publish. The fix is on the listener:

```java
@Component
public class OrderPlacedListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPlaced(OrderPlaced event) {
        notifier.sendConfirmation(event.orderId());
    }
}
```

`AFTER_COMMIT` registers a `TransactionSynchronization` callback on the
`TransactionSynchronizationManager` you met above, and fires it after the commit
completes. Two things to internalise:

- The listener runs on the **same thread**, but the transaction is already gone. A
  repository write inside it needs `REQUIRES_NEW` or it will run outside any
  transaction.
- There is still a crash window between the commit and the publish. If the JVM dies
  there, the order exists and the event never fires. That window is precisely what the
  transactional outbox (Topic 115) closes. Say this in an interview and you have
  demonstrated you know the difference between "works" and "is correct under failure".

**5. The read path is separate and read-only.**

```java
@Service
public class OrderQueryService {

    @Transactional(readOnly = true, timeout = 2)
    public OrderSummary summary(OrderId id) {
        return orders.findSummaryById(id)       // interface projection, Topic 47
                     .orElseThrow(() -> new OrderNotFoundException(id));
    }
}
```

`readOnly = true` here buys a skipped flush and, later, replica routing. It buys no
safety. And the two-second timeout exists for the same reason as above.

### What is deliberately outside the transaction

```java
@RestController
class OrderController {

    @PostMapping("/orders")
    ResponseEntity<OrderResponse> place(@Valid @RequestBody PlaceOrderRequest req) {
        FraudDecision decision = fraudClient.check(req);   // HTTP call — OUTSIDE the tx
        if (decision.isBlocked()) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
        }
        OrderId id = placementService.place(req.toCommand());   // tx starts HERE
        return ResponseEntity.created(...).body(...);
    }
}
```

The fraud check is a network call to another service. Its p99 is 180 ms on a good day
and unbounded on a bad one. Putting it inside `place` would mean a 20-connection pool
serving 120 req/sec while each request holds a connection for 180 ms plus database
time. That is a total outage waiting for one bad afternoon at the fraud provider, and
it is the entire subject of Topic 55's failure drill.

**The rule, stated once, to be applied forever: a transaction contains database work
and nothing else.**

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — self-invocation: a silently non-transactional write

**Wrong:**
```java
@Service
public class OrderPlacementService {

    public OrderId place(PlaceOrderCommand cmd) {
        validate(cmd);
        return doPlace(cmd);            // <-- plain call on `this`
    }

    @Transactional
    OrderId doPlace(PlaceOrderCommand cmd) {
        inventory.reserve(...);
        wallets.debit(...);             // throws
        orders.save(order);
    }
}
```

**Exact symptom:** `wallets.debit` throws, the request returns 500, and the inventory
row **still shows the reservation**. Re-run it ten times and you have leaked ten units
of stock. No exception mentions transactions. `logging.level.org.springframework.transaction=TRACE`
prints **no** "Creating new transaction" line for `doPlace`.

**Root cause:** `doPlace(cmd)` compiles to `this.doPlace(cmd)`. `this` is your raw
object, not the CGLIB subclass. The proxy is never consulted, so `TransactionInterceptor`
never runs. Each repository call then runs in its own auto-commit transaction, which is
why the inventory `UPDATE` is durable on its own.

This is Topic 40's trap. What is new here is the *shape of the damage*: not an error,
but a half-completed business operation.

**Fix — in order of preference:**

```java
// 1. BEST: move the boundary to the entry point that callers actually call.
@Transactional
public OrderId place(PlaceOrderCommand cmd) {
    validate(cmd);
    return doPlace(cmd);        // now inside an existing transaction; REQUIRED joins it
}
```

```java
// 2. Extract the transactional work into a separate bean, so the call crosses a proxy.
@Service
class OrderPlacementService {
    private final OrderWriter writer;                 // a different bean
    public OrderId place(PlaceOrderCommand cmd) {
        validate(cmd);
        return writer.doPlace(cmd);                   // goes THROUGH writer's proxy
    }
}
```

```java
// 3. Programmatic. Correct, explicit, and the closest thing to your Nest queryRunner.
private final TransactionTemplate tx;                  // inject TransactionTemplate

public OrderId place(PlaceOrderCommand cmd) {
    validate(cmd);
    return tx.execute(status -> doPlace(cmd));
}
```

```java
// 4. Self-injection. Works, and looks like a hack because it is one.
@Autowired private ObjectProvider<OrderPlacementService> self;
public OrderId place(PlaceOrderCommand cmd) {
    validate(cmd);
    return self.getObject().doPlace(cmd);              // through the proxy
}
```

Option 3 is underrated. When the atomic region is *narrower than a method* — you want
the transaction to cover three statements out of thirty —
`TransactionTemplate` is not a workaround, it is the right tool.

---

### Trap 2 — a checked exception commits the transaction

**Give this one full weight. It is the single most common silent data bug in Spring
codebases.**

**Wrong:**
```java
@Transactional
public void settle(PaymentId id) throws GatewayDeclinedException {   // CHECKED
    Payment payment = payments.findById(id).orElseThrow();
    payment.markSettled();                       // dirty check -> UPDATE at flush
    wallets.credit(payment.merchantId(), payment.amountMinor());

    GatewayResult result = gateway.confirm(id);
    if (!result.accepted()) {
        throw new GatewayDeclinedException(id);  // extends Exception
    }
}
```

**Exact symptom:** the API returns a 402. The logs contain a clean
`GatewayDeclinedException` with a full stack trace. And the payment row reads `SETTLED`
and the merchant wallet has been credited. Reconciliation catches it a week later, or a
customer does.

With `logging.level.org.springframework.transaction=TRACE` you will see a line whose
*shape* is like this — **this is an illustration of the log format, not captured
output**:

```
TRACE ... TransactionInterceptor : Completing transaction for
        [com.orderflow.payments.PaymentService.settle] after exception:
        com.orderflow.payments.GatewayDeclinedException: PAY-8831
TRACE ... JpaTransactionManager  : Initiating transaction commit
```

`after exception` immediately followed by `Initiating transaction commit` is the exact
signature of this bug. Learn those two lines as a pair.

**Root cause:** the default rollback rule is
`DefaultTransactionAttribute.rollbackOn(Throwable ex)`, which returns true for
`RuntimeException` and `Error` **only**. A checked exception falls into the `else`
branch of the interceptor pseudocode above: commit, then rethrow.

**Why is the default that way?** This is the interview follow-up, so have an answer.
The design predates Spring: EJB's `SessionContext` used the same rule. The rationale is
that a checked exception is, by Java's own contract (Topic 08), an *anticipated
alternative outcome* that the caller is expected to handle — not a defect. An
anticipated outcome should not unilaterally destroy work the caller may want to keep.
An unchecked exception, by the same contract, signals a programming defect, and after a
defect the transaction's state is untrustworthy, so rolling back is the safe default.

The rationale is coherent. It also does not match what any working engineer expects
from the word "transactional", which is why it produces so many bugs.

**Fix — three options, pick per situation:**

```java
// 1. Declare it. Explicit, local, survives someone changing the exception hierarchy.
@Transactional(rollbackFor = GatewayDeclinedException.class)
```

```java
// 2. Declare it for a whole family. Blunt but honest, and easy to review.
@Transactional(rollbackFor = Exception.class)
```

```java
// 3. Make the exception unchecked. Usually the right answer for a domain exception
//    the caller cannot recover from (Topic 09).
public class GatewayDeclinedException extends RuntimeException { ... }
```

**Team-level fix:** if your codebase uses checked domain exceptions, put
`rollbackFor = Exception.class` in a meta-annotation and use it everywhere:

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Transactional(rollbackFor = Exception.class)
public @interface AtomicCommand { }
```

Now `@AtomicCommand` on a method means what people think `@Transactional` means. Spring
reads `@Transactional` through meta-annotations, so this works with no extra code.

The mirror-image knob is `noRollbackFor`, for the rarer case where an unchecked
exception is a legitimate outcome you want committed — for example a
`DuplicateOrderException` where the correct behaviour is "keep the row we already
wrote".

---

### Trap 3 — `@Transactional` on a `private` or `final` method

**Wrong:**
```java
@Service
public class InventoryService {

    @Transactional
    private void reserveInternal(String sku, int qty) {     // private
        ...
    }

    @Transactional
    public final void reserve(String sku, int qty) {        // final
        ...
    }
}
```

**Exact symptom:** absolutely nothing. No warning at startup, no exception, no log
line. The methods run, the writes happen with auto-commit, and a mid-method failure
leaves partial state. If you have `logging.level.org.springframework.transaction=TRACE`
on, the absence of a "Creating new transaction" line is your only clue.

**Root cause:** a CGLIB proxy is a **subclass**. It advises by overriding. You cannot
override a `private` method (it is not inherited) and you cannot override a `final`
one. With a JDK dynamic proxy the situation is worse: only interface methods exist at
all. Topic 40 again — this is the same fact wearing different clothes.

**Fix:** make the method `public` (or at minimum non-private and non-final) and ensure
it is called from outside the bean. If it genuinely must stay private, extract it into
its own bean, or use `TransactionTemplate` inside it.

**Make the compiler catch it.** This class of bug is entirely preventable with static
analysis. SpotBugs, Sonar and IntelliJ's inspections all flag `@Transactional` on
non-overridable methods. Turn it on and fail the build. It costs an afternoon and
removes the trap permanently — which is a better answer in an interview than "you have
to remember".

---

### Trap 4 — catching the exception inside the transaction

**Wrong:**
```java
@Transactional
public OrderId place(PlaceOrderCommand cmd) {
    inventory.reserve(cmd.sku(), cmd.quantity());
    orders.save(order);

    try {
        wallets.debit(cmd.customerId(), order.totalMinor());
    } catch (InsufficientFundsException e) {
        log.warn("debit failed for order {}, continuing", order.id(), e);
        order.markPaymentPending();
    }
    return order.id();
}
```

**Exact symptom (two variants, and you must be able to tell them apart):**

*Variant A — `wallets.debit` has no `@Transactional` of its own, or has `REQUIRED` and
you swallowed an unchecked exception.* The method returns normally, so the interceptor
commits. You now have an order and an inventory reservation with no wallet debit.
Half-completed business operation, HTTP 201, no alert.

*Variant B — `wallets.debit` is `@Transactional(REQUIRED)` and threw.* The inner
interceptor could not roll back (it joined, it does not own the transaction), so it set
**rollback-only**. Your `catch` swallowed the exception; `place` returns normally; the
outer interceptor tries to commit and throws
`UnexpectedRollbackException: Transaction rolled back because it has been marked as
rollback-only`. The client gets a 500 pointing at a line of code that did nothing
wrong, and the stack trace names no business logic at all.

**Root cause:** the interceptor's *only* input for the commit/rollback decision is what
escapes your method. A `catch` block is you telling Spring "nothing went wrong". Trap 4
and Trap 2 are the same mistake from opposite directions: Trap 2 is Spring not seeing a
failure it should act on; Trap 4 is you hiding a failure Spring would have acted on.

**Fix — pick one deliberately:**

```java
// 1. Do not swallow. Let it out, let the transaction die. Usually correct.
wallets.debit(cmd.customerId(), order.totalMinor());
```

```java
// 2. If the fallback is genuinely valid business behaviour, the failing call must be
//    in a transaction you are allowed to lose independently.
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void tryDebit(...) { ... }
// ...and now re-read Trap 5, because you just took a second connection.
```

```java
// 3. If you must swallow, make the intent explicit and check the flag.
try {
    wallets.debit(...);
} catch (InsufficientFundsException e) {
    if (TransactionSynchronizationManager.isActualTransactionActive()
            && TransactionAspectSupport.currentTransactionStatus().isRollbackOnly()) {
        throw e;                    // it is already doomed; stop pretending
    }
    order.markPaymentPending();
}
```

**The diagnostic rule to remember:** `UnexpectedRollbackException` is never the real
bug. It is always a `catch` block somewhere upstream. Go find it.

---

### Trap 5 — `REQUIRES_NEW` taking a second connection while holding the first

**Wrong:**
```java
@Transactional
public OrderId place(PlaceOrderCommand cmd) {
    for (OrderLineCommand line : cmd.lines()) {      // 3.4 lines on average
        auditService.recordLineAttempt(line);        // REQUIRES_NEW, per line
        inventory.reserve(line.sku(), line.quantity());
    }
    ...
}
```

**Exact symptom:** fine in development, fine at low load. At about 20 concurrent
placements against a 20-connection pool, every request hangs and then fails with:

```
java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available,
request timed out after 30000ms
```

*(illustration of the message format)*

`hikaricp.connections.active` sits pinned at 20. `hikaricp.connections.pending` climbs
without bound. `hikaricp.connections.timeout` counts up. **Every endpoint in the
service fails**, including ones that touch no database, because Tomcat's threads are
all parked waiting on the pool. A thread dump (`jcmd <pid> Thread.print`) shows every
request thread parked inside `HikariPool.getConnection`.

**Root cause:** `REQUIRES_NEW` suspends the outer transaction but does **not** release
its connection — the outer connection stays bound in `TransactionSynchronizationManager`
with its locks and its snapshot intact, waiting to be resumed. So the thread holds two.
With `N` threads each holding one and requesting a second from a pool of `N`, there is
no ordering in which anybody finishes. It is a textbook resource deadlock, and it is
100% self-inflicted by an annotation attribute.

**Fix:**

```java
// 1. BEST: do not need a second transaction. Collect the audit rows and write them
//    inside the same transaction, or emit them via the outbox (Topic 115).
List<OrderAudit> auditRows = new ArrayList<>();
for (OrderLineCommand line : cmd.lines()) {
    auditRows.add(OrderAudit.attempt(line));
    inventory.reserve(line.sku(), line.quantity());
}
auditRepository.saveAll(auditRows);          // same transaction, one connection
```

```java
// 2. If REQUIRES_NEW is genuinely required, size the pool for it and prove the bound:
//    pool >= threads * (max_simultaneous_connections_per_thread - 1) + 1
//    With 200 Tomcat threads and 2 connections each: 201. Against 8 Postgres vCPUs
//    that is not viable, which means the design is wrong, not the pool.
```

```java
// 3. Move the independent write off the request path entirely — an outbox row plus a
//    relay, or an @Async handler with its own bounded pool (Topic 90).
```

**The rule:** count the maximum number of connections one request thread can hold at
once. If that number is greater than one, you have a capacity problem you must be able
to defend with arithmetic. Full treatment in Topic 109.

---

## Hands-on proof

Everything below is something **you** run. I have no JVM, no Spring context and no
Postgres, so I will not print output and call it real. What I can give you exactly is
the configuration, what to look for, and how to read every possible result.

### Setup — turn on the instruments

`src/main/resources/application-lab.yml`:

```yaml
logging:
  level:
    org.springframework.transaction: TRACE
    org.springframework.orm.jpa: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE     # shows bound parameter values
spring:
  jpa:
    properties:
      hibernate.format_sql: true
  datasource:
    hikari:
      maximum-pool-size: 20
      pool-name: orderflow-pool
```

Run with `--spring.profiles.active=lab`.

**What each logger tells you:**

| Logger | What appears | Why you need it |
|---|---|---|
| `org.springframework.transaction=TRACE` | `Creating new transaction with name [...]`, `Participating in existing transaction`, `Suspending current transaction`, `Initiating transaction commit/rollback`, `Completing transaction ... after exception` | The single most useful transaction instrument in Spring. Every trap in this doc is visible here. |
| `org.springframework.orm.jpa=DEBUG` | `Opened new EntityManager`, `Not closing pre-bound EntityManager`, `Closing JPA EntityManager` | Proves whether a persistence context was created or joined |
| `org.hibernate.SQL=DEBUG` | the SQL statements | Shows *what* was written and in what order |
| `org.hibernate.orm.jdbc.bind=TRACE` | bound parameters | Turns `?` into values. **Never enable in production** — it logs PII. |

> `[BOOT 3.x DELTA]` The parameter-binding logger was renamed. On Boot 3.x /
> Hibernate 6 it is `org.hibernate.orm.jdbc.bind`; on Boot 2.x / Hibernate 5 it was
> `org.hibernate.type.descriptor.sql.BasicBinder`. If you see no parameter values,
> you have the wrong logger name for your Hibernate version.

### Proof 1 — is there a transaction here at all?

The cleanest proof available. Put this in any method:

```java
package com.orderflow.support;

import org.springframework.transaction.support.TransactionSynchronizationManager;

public final class TxProbe {
    private TxProbe() {}

    public static void print(String where) {
        System.out.printf(
            "[%s] thread=%s active=%s name=%s readOnly=%s isolation=%s%n",
            where,
            Thread.currentThread().getName(),
            TransactionSynchronizationManager.isActualTransactionActive(),
            TransactionSynchronizationManager.getCurrentTransactionName(),
            TransactionSynchronizationManager.isCurrentTransactionReadOnly(),
            TransactionSynchronizationManager.getCurrentTransactionIsolationLevel());
    }
}
```

Call `TxProbe.print("place")` as the first line of any method you are investigating.

**How to read every possible result:**

| What you see | What it means |
|---|---|
| `active=true name=com.orderflow...place` | A real transaction exists and Spring named it after this method. The proxy ran. This method **is** the boundary. |
| `active=true` but `name` is a *different* method | You are participating in a caller's transaction (`REQUIRED` joined). Your own `@Transactional` attributes — including `readOnly` and `isolation` — were **ignored**. |
| `active=false name=null` | There is no transaction. If this method is annotated, the annotation did not apply: self-invocation, `private`/`final`, missing `@EnableTransactionManagement`, or the bean is not a Spring bean at all. |
| `readOnly=true` when you expected false | You joined a read-only transaction. Hibernate is in `FlushMode.MANUAL`; your writes will be **silently discarded**. |
| `thread` differs between two probes | You crossed a thread boundary. The `ThreadLocal` did not follow. Everything downstream is untransacted. |

### Proof 2 — reveal the proxy

```java
@Component
public class ProxyReport {

    private final OrderPlacementService placement;

    ProxyReport(OrderPlacementService placement) { this.placement = placement; }

    @EventListener(ApplicationReadyEvent.class)
    public void report() {
        System.out.println("injected class : " + placement.getClass().getName());
        System.out.println("is AOP proxy   : " + AopUtils.isAopProxy(placement));
        System.out.println("is CGLIB proxy : " + AopUtils.isCglibProxy(placement));
        System.out.println("target class   : " + AopUtils.getTargetClass(placement).getName());
    }
}
```

| What you see | What it means |
|---|---|
| `injected class` contains `$$SpringCGLIB$$` | Class-based proxy. `final`/`private`/`static` methods are not advised. Self-invocation bypasses everything. |
| `injected class` is `jdk.proxy2.$Proxy###` | Interface-based JDK proxy. Only interface methods are advised at all. |
| `injected class` equals `target class` and `isAopProxy` is `false` | **No proxy exists.** Nothing on this bean is transactional. Check `@EnableTransactionManagement` (Boot auto-configures it — its absence usually means a hand-rolled test context), and check that Spring actually manages this bean. |

`ApplicationReadyEvent` matters here: run this in a constructor or `@PostConstruct` and
you may be looking at a bean before proxying happened (Topic 37).

### Proof 3 — read the propagation decisions in the log

Call `place()` (which is `REQUIRED`) and have it call, through a proxy, a
`REQUIRES_NEW` method. With `org.springframework.transaction=TRACE` on, look for this
sequence of *log-line shapes* — **this is the format to expect, not captured output**:

```
TRACE ... : Creating new transaction with name [com.orderflow.orders...place]
TRACE ... : Suspending current transaction, creating new transaction with name [...recordAttempt]
TRACE ... : Initiating transaction commit
TRACE ... : Resuming suspended transaction after completion of inner transaction
TRACE ... : Initiating transaction commit
```

| Phrase to look for | What it proves |
|---|---|
| `Creating new transaction with name [X]` | X is a boundary. A `BEGIN` was issued and a connection taken. |
| `Participating in existing transaction` | `REQUIRED` joined. **No** new connection, and this method's own attributes were discarded. |
| `Suspending current transaction, creating new transaction` | `REQUIRES_NEW`. **Two connections held right now.** Count these lines under load. |
| `Creating nested transaction with name [X]` | `NESTED`. A savepoint, one connection. |
| `Participating transaction failed - marking existing transaction as rollback-only` | An inner method threw. The outer transaction is already doomed, whatever happens next. |
| `Completing transaction for [X] after exception` followed by `Initiating transaction commit` | **Trap 2 firing.** A checked exception committed. |
| *no line at all* for an annotated method | The proxy was bypassed. Trap 1 or Trap 3. |

### Proof 4 — count the connections a request actually holds

Expose Hikari metrics through Actuator:

```yaml
management:
  endpoints.web.exposure.include: metrics,health
  metrics.enable.hikaricp: true
```

Then, while a single request is in flight (put a breakpoint or a deliberate 10-second
sleep in the middle of the transaction — in your lab only):

```bash
curl -s localhost:8080/actuator/metrics/hikaricp.connections.active | jq
```

| What you see | What it means |
|---|---|
| `active` = 1 for a single in-flight request | One transaction, one connection. Correct. |
| `active` = 2 for a single in-flight request | Something took a second connection — `REQUIRES_NEW`, a nested `TransactionTemplate` with a new transaction, or a second `DataSource`. Trap 5 territory. |
| `active` = 0 while your code is clearly running | You are outside a transaction, or the work already finished. Cross-check with `TxProbe`. |

Cross-check on the database side:

```sql
SELECT pid, state, xact_start, now() - xact_start AS xact_age,
       wait_event_type, wait_event, left(query, 80) AS query
FROM pg_stat_activity
WHERE datname = 'orderflow' AND state <> 'idle'
ORDER BY xact_start;
```

| What you see | What it means |
|---|---|
| `state = 'idle in transaction'` with a growing `xact_age` | A transaction is open and the application is doing something that is not SQL. That "something" is your bug. Topic 55's drill is exactly this. |
| `xact_age` > your `@Transactional(timeout)` | The timeout did not apply. Check that the annotation is on the boundary method and that the call came through the proxy. |
| Many rows with `state = 'active'` and identical queries | Contention, not a transaction-boundary problem. Different investigation (Topics 52, 109). |

---

## Failure drill

**Mandatory.** Two parts. Do them in order; part (b) only makes sense once part (a) has
shown you that the proxy is real.

### Part (a) — the self-invocation trap

**Goal:** write a row, throw, and observe the row is still there. Then prove why.

**Step 1 — build the broken version.**

```java
package com.orderflow.inventory;

@Service
public class ReservationService {

    private final InventoryRepository inventory;
    private final ReservationRepository reservations;

    ReservationService(InventoryRepository inventory, ReservationRepository reservations) {
        this.inventory = inventory;
        this.reservations = reservations;
    }

    // NOT annotated. This is the method the controller calls.
    public void reserve(String sku, int quantity) {
        TxProbe.print("reserve(outer)");
        reserveInternal(sku, quantity);              // <-- self-invocation
    }

    @Transactional
    void reserveInternal(String sku, int quantity) {
        TxProbe.print("reserveInternal");
        reservations.save(new Reservation(sku, quantity));   // the write
        throw new IllegalStateException("deliberate failure after the write");
    }
}
```

**Step 2 — the config.** The `application-lab.yml` above, plus:

```yaml
spring:
  jpa:
    hibernate.ddl-auto: validate      # never let the drill hide a schema problem
```

**Step 3 — run it.**

```bash
curl -i -X POST localhost:8080/inventory/reserve \
     -H 'Content-Type: application/json' \
     -d '{"sku":"SKU-1001","quantity":2}'

psql -d orderflow -c "SELECT count(*) FROM reservation WHERE sku = 'SKU-1001';"
```

**Step 4 — what to capture.** Four things, saved to a file:

1. The HTTP status and body from `curl -i`.
2. Both `TxProbe` lines.
3. Every line from `org.springframework.transaction` during the request.
4. The `count(*)` from Postgres.

**Step 5 — how to read it.**

| What you see | What it means |
|---|---|
| `count(*)` = 1 after a 500 response | **The trap fired.** The write committed despite the exception. |
| `TxProbe` prints `active=false` in **both** places | Confirmed: `reserveInternal` ran with no transaction at all. |
| No `Creating new transaction` line for `reserveInternal` | The interceptor never ran. The proxy was bypassed. |
| A `Creating new transaction` line **does** appear and `count(*)` = 0 | The trap did *not* fire. Almost always because you called `reserveInternal` from outside the class, or you have AspectJ load-time weaving enabled (which advises the real object, not a proxy). Check how the controller calls in. |
| `count(*)` = 1 **and** `active=true` | Something else is wrong: an `@Transactional(noRollbackFor = ...)`, or the write went through a different `DataSource`. |

**Step 6 — reveal the cause.** Add the `ProxyReport` bean from Proof 2 and restart.

| What you see | What it means |
|---|---|
| `com.orderflow.inventory.ReservationService$$SpringCGLIB$$0` | The proxy exists and is a CGLIB subclass. The container has one. Your `this` call simply never asked it. **This is the proof.** |
| `ReservationService` with no marker, `isAopProxy=false` | There is no proxy at all — a different bug (bean not managed, or `@EnableTransactionManagement` missing). Fix that first, then redo the drill. |

**Step 7 — fix it three ways and state what each proves.**

| Fix | What it proves |
|---|---|
| Move `@Transactional` up to `reserve()` | The boundary belongs at the method callers actually call. `count(*)` becomes 0. |
| Extract `reserveInternal` into a `ReservationWriter` bean and call it | The call now crosses a proxy. Proves the mechanism is *crossing a bean boundary*, not the annotation's position. |
| Wrap the body in `TransactionTemplate.execute(...)` | Proves the interceptor was never the point — a transaction manager invoked directly works fine. This is the Nest `queryRunner` shape, and it removes the trap by removing the proxy. |

**Step 8 — the sentence you should be able to say afterwards.** "The annotation was
read by an object I never called."

### Part (b) — a checked exception commits

**Goal:** with a *correctly proxied* `@Transactional` method, throw a checked exception
and watch the write commit.

**Step 1 — the code.** Note that the call now comes from a controller, so the proxy
definitely runs. That is the point: this failure has nothing to do with part (a).

```java
public class GatewayDeclinedException extends Exception {          // CHECKED
    public GatewayDeclinedException(String message) { super(message); }
}

@Service
public class PaymentSettlementService {

    @Transactional                                   // no rollbackFor
    public void settle(long paymentId) throws GatewayDeclinedException {
        TxProbe.print("settle");
        Payment payment = payments.findById(paymentId).orElseThrow();
        payment.markSettled();                       // dirty checking -> UPDATE
        throw new GatewayDeclinedException("gateway said no");
    }
}
```

**Step 2 — run it.**

```bash
psql -d orderflow -c "SELECT id, status FROM payment WHERE id = 991;"
curl -i -X POST localhost:8080/payments/991/settle
psql -d orderflow -c "SELECT id, status FROM payment WHERE id = 991;"
```

**Step 3 — how to read it.**

| What you see | What it means |
|---|---|
| `status` goes `PENDING` → `SETTLED`, and the HTTP response is an error | **The trap fired.** A checked exception committed the transaction. |
| `TxProbe` shows `active=true name=...settle` | The transaction was completely real. This is not a proxy problem. Rule that out explicitly — it is the whole lesson. |
| A `Completing transaction for [...settle] after exception` line followed by `Initiating transaction commit` | The interceptor telling you, in plain English, that it chose to commit. |
| `status` stays `PENDING` | Either your exception extends `RuntimeException` (check the class), or a `rollbackFor` is inherited from a meta-annotation, or Hibernate never flushed because the transaction was `readOnly`. Check all three. |

**Step 4 — fix and re-run.** Change one thing at a time and re-run the whole sequence:

| Fix | Expected observation | What it proves |
|---|---|---|
| `@Transactional(rollbackFor = GatewayDeclinedException.class)` | `status` stays `PENDING`; log shows `Initiating transaction rollback` | The rule is per-exception-type and configurable |
| `@Transactional(rollbackFor = Exception.class)` | Same | The blunt team-wide fix |
| Make the exception extend `RuntimeException`, remove `rollbackFor` | Same | The default rule is about the *type hierarchy*, not about the annotation |
| Keep it checked, catch it inside the method, return normally | `status` becomes `SETTLED` and **no** exception reaches the client | Trap 4: swallowing is the same bug with the failure hidden as well |

**Step 5 — the sentence to be able to say.** "Spring's default rollback rule follows
EJB: unchecked means defect, so roll back; checked means an anticipated outcome, so
commit. I always set `rollbackFor` or make domain exceptions unchecked."

---

## Measurement

Claims about transaction boundaries must be falsifiable. Here is how.

### The metrics that matter

With `spring-boot-starter-actuator` and Micrometer on the classpath:

| Metric | What it tells you | The number to watch |
|---|---|---|
| `hikaricp.connections.active` | connections currently checked out | Sustained near `maximum-pool-size` means transactions are too long or too many |
| `hikaricp.connections.pending` | threads **blocked** waiting for a connection | Anything above zero for a sustained period is a live incident |
| `hikaricp.connections.usage` | timer: how long a connection is held once acquired | **This is your transaction-duration proxy.** Its p99 is the number to defend. |
| `hikaricp.connections.acquire` | timer: how long acquisition took | Rising means the pool is contended |
| `hikaricp.connections.timeout` | count of acquisition timeouts | Every increment is a failed request |
| `hikaricp.connections.max` / `.min` | configured bounds | Confirms your config actually applied |

```bash
curl -s localhost:8080/actuator/metrics/hikaricp.connections.usage | jq
curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending | jq
```

`hikaricp.connections.usage` is the metric to build a dashboard panel and an alert on.
It is close to "transaction duration" without you instrumenting anything, because for a
Spring JPA application a connection is checked out for almost exactly the transaction's
lifetime.

### Measuring transaction duration directly

If you want the real thing rather than the proxy metric, time it where the transaction
actually begins and ends — a `TransactionSynchronization` callback:

```java
@Component
public class TransactionTimer {

    private final MeterRegistry registry;

    TransactionTimer(MeterRegistry registry) { this.registry = registry; }

    public void startTiming() {
        if (!TransactionSynchronizationManager.isSynchronizationActive()) return;
        long startNanos = System.nanoTime();
        String name = TransactionSynchronizationManager.getCurrentTransactionName();
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                @Override public void afterCompletion(int status) {
                    registry.timer("orderflow.tx.duration",
                                   "tx", name == null ? "unknown" : name,
                                   "status", status == STATUS_COMMITTED ? "commit" : "rollback")
                            .record(System.nanoTime() - startNanos, TimeUnit.NANOSECONDS);
                }
            });
    }
}
```

Call `startTiming()` from an `@Around` aspect over your transactional entry points
(Topic 41), or from the first line of the boundary method. The tag on commit-vs-rollback
matters: a rollback rate that climbs is a business-logic signal, and a rollback rate
that is suspiciously *zero* on a path with checked exceptions is Trap 2 in production.

### Why a naive `System.nanoTime()` timing is wrong

You will be tempted to do this:

```java
long start = System.nanoTime();
orderService.place(cmd);
long micros = (System.nanoTime() - start) / 1_000;   // WRONG for four reasons
```

Four independent reasons this number is not what you think:

1. **You are timing the wrong interval.** The transaction begins inside the proxy
   *before* your method body and commits *after* it returns. A stopwatch around the
   call includes proxy dispatch and the commit; a stopwatch inside the method body
   excludes both — and the commit is often the expensive part, because that is when
   Hibernate flushes every pending write and Postgres writes to WAL.
2. **JIT compilation.** The first few thousand invocations run interpreted, then
   partially compiled, then optimised — possibly with on-stack replacement mid-loop. A
   single timing measures whichever tier you happened to hit. Topic 77.
3. **It measures one sample, and latency is a distribution.** Your SLO is a p99. One
   sample tells you nothing about a tail that is by definition 1 in 100. Coordinated
   omission (Topic 65) makes hand-rolled measurement under load worse still.
4. **Safepoints and GC pauses land inside your window arbitrarily.** A 40 ms young-GC
   pause between the two `nanoTime` calls is attributed entirely to your transaction.
   Topic 68 onwards.

**What to do instead:** use the Micrometer timer above (which records a distribution
and percentiles), read `hikaricp.connections.usage` for the pool-side view, and for
anything microbenchmark-shaped use JMH (Topic 77). Under load, take the numbers from
the Topic 65 baseline, not from a stopwatch.

### The falsifiable claims of this topic

Write these down and test each one. That is the difference between knowing and
believing.

| Claim | How to falsify it |
|---|---|
| "Self-invocation is not transactional" | Failure drill part (a): the row survives |
| "A checked exception commits" | Failure drill part (b): the status flips |
| "`REQUIRES_NEW` holds two connections" | `hikaricp.connections.active` = 2 with one request in flight |
| "`readOnly=true` skips the flush" | Modify a managed entity in a `readOnly` transaction; no `UPDATE` in the Hibernate SQL log |
| "A transaction holds a connection for its whole life" | `pg_stat_activity` shows `idle in transaction` for the duration — Topic 55 |

---

## Practice exercises

Write real code, run it, and keep the evidence.

### 1 — Easy: build a propagation truth table by observation

Create two beans, `OuterService` and `InnerService`. `OuterService.run()` is
`@Transactional` and calls `innerService.work()` through injection. Call `TxProbe.print`
in both.

Run the whole matrix, changing only the propagation on `InnerService.work()`:
`REQUIRED`, `REQUIRES_NEW`, `NESTED`, `SUPPORTS`, `NOT_SUPPORTED`, `MANDATORY`, `NEVER`.
Run each one twice: once called through `OuterService.run()`, and once called directly
from a controller with no surrounding transaction.

For each of the 14 cells record: `active`, the transaction `name`, and which
`org.springframework.transaction` TRACE lines appeared.

Then answer:
- Which levels produced a *different* transaction name in the inner method, and why is
  that the tell for "a new transaction was created"?
- Which one threw, and what was the exception class?
- Which one, if any, failed on `JpaTransactionManager` — and what exactly did the
  message say?

### 2 — Medium: the audit (combines Topics 08, 09, 37, 40, 48, 52)

The class below contains **six** distinct defects from this topic and earlier ones. For
each: name the defect, state the **exact observable symptom** an on-call engineer would
see (not "bad practice" — what appears in the logs, the response, or the database), and
write the fix.

```java
@Service
public class CheckoutService {

    @Autowired private OrderRepository orders;
    @Autowired private InventoryRepository inventory;
    @Autowired private WalletRepository wallets;
    @Autowired private AuditRepository audits;

    @PostConstruct
    void warmUp() {
        preloadHotProducts();
    }

    @Transactional
    public void preloadHotProducts() {
        inventory.findTop50ByOrderBySoldDesc();
    }

    public OrderId checkout(CheckoutCommand cmd) throws PaymentDeclinedException {
        return doCheckout(cmd);
    }

    @Transactional(readOnly = true)
    private final OrderId doCheckout(CheckoutCommand cmd) throws PaymentDeclinedException {

        Order order = new Order(cmd.customerId());
        for (var line : cmd.lines()) {
            Inventory inv = inventory.findBySku(line.sku());
            inv.setAvailable(inv.getAvailable() - line.quantity());
            order.addLine(line);
        }

        Wallet wallet = wallets.findById(cmd.customerId()).orElseThrow();
        try {
            wallet.debit(order.totalMinor());
        } catch (InsufficientFundsException e) {
            audits.save(Audit.of("debit failed", cmd.customerId()));
        }

        if (!gateway.authorize(order)) {
            throw new PaymentDeclinedException(order.id());   // extends Exception
        }

        orders.save(order);
        return order.id();
    }
}
```

Hints, one per earlier topic: Topic 37 (when does the proxy exist?), Topic 40 (which
methods can be advised?), Topic 48 (what does `readOnly` do to flush?), Topic 52 (is
that decrement safe under concurrency?), Topic 09 (should that exception be checked?),
and this topic (what does that `catch` block tell the interceptor?).

### 3 — Hard: production simulation — make `place()` correct and defensible

Advance the `orderflow` spine. Implement `OrderPlacementService.place()` to satisfy all
of the following, and produce evidence for each.

**Part A — correctness.**
Reserve inventory for every line, debit the wallet, create a `PENDING` payment, and
publish `OrderPlaced` after commit. Write integration tests (Testcontainers Postgres —
Topic 61) asserting: if the wallet debit fails, **zero** inventory rows changed and
**zero** order rows exist. Test it with the exception both checked and unchecked, and
show that the checked case fails until you add `rollbackFor`.

**Part B — the boundary is narrow.**
Move the fraud HTTP call outside the transaction. Prove the transaction contains only
database work by asserting on the SQL log, or by asserting
`TransactionSynchronizationManager.isActualTransactionActive()` is `false` at the point
of the HTTP call.

**Part C — count the connections.**
Add the `TransactionTimer` from the Measurement section. Run 200 order placements. Report
the p50/p95/p99 of `orderflow.tx.duration` and the maximum of
`hikaricp.connections.active`. State whether one request ever held two connections, and
how you know.

**Part D — break it deliberately.**
Change the audit write to `REQUIRES_NEW`. Set `maximum-pool-size: 5`. Run 10 concurrent
placements. Capture `jcmd <pid> Thread.print` and the Hikari metrics. Identify every
thread parked in `HikariPool.getConnection` and explain the cycle in one paragraph.
Then restore the fix and re-run to show `pending` returns to zero.

**Part E — argue the other side.**
Your team lead proposes `@Transactional(rollbackFor = Exception.class)` on every service
method via a meta-annotation. Write the strongest case *for* it, then the strongest case
*against* it. Name one concrete scenario in `orderflow` where committing on a specific
exception is the correct behaviour, and how you would express that.

> Part C's timings are still not a benchmark. You are measuring a running service under
> synthetic load, which is the right shape, but the numbers only become trustworthy
> against the Topic 65 baseline. Revisit this after Topic 65 and again after Topic 77.

---

## Interview questions

### Q1 — "Your `@Transactional` did nothing. Why?"

**Mid-level answer:** "Probably self-invocation — calling the annotated method from
another method in the same class bypasses the proxy."

**Senior answer:** "Self-invocation is the most common, but I'd run a checklist rather
than guess, because there are five causes and they look identical from the outside. One:
self-invocation — `this.method()` never touches the proxy. Two: the method is `private`,
`final` or `static`, so CGLIB can't override it. Three: the class isn't a Spring bean at
all — someone `new`ed it. Four: it's called from `@PostConstruct`, where the proxy
doesn't exist yet from that bean's point of view. Five: there's no transaction manager,
usually in a hand-rolled test context. I diagnose it in two moves: print
`TransactionSynchronizationManager.isActualTransactionActive()` inside the method, and
print `bean.getClass().getName()` to see whether there's a `$$SpringCGLIB$$` marker. Then
`logging.level.org.springframework.transaction=TRACE` tells me whether a 'Creating new
transaction' line appeared. And the real fix is usually architectural — the boundary
belongs at the method callers actually call, or the work belongs in a separate bean."

**What separates them:** an ordered checklist instead of one guess, a *diagnostic
procedure* rather than a fact, and treating the fix as a design question rather than an
annotation move.

**Follow-up:** "You moved the annotation to the public method and it works. Now a
colleague adds a second public method that calls the first internally. What happens, and
how do you stop it happening again?" *(Answer: it silently loses the boundary again;
prevent it with static analysis, or by extracting the transactional work into a
collaborator so a plain call cannot reach it.)*

---

### Q2 — "A checked exception was thrown and the row is still there. Explain."

**Mid-level answer:** "Spring only rolls back on runtime exceptions by default. You need
`rollbackFor`."

**Senior answer:** "That's the default rollback rule: `DefaultTransactionAttribute`
rolls back on `RuntimeException` and `Error` only, so a checked exception takes the
commit path and is then rethrown. The rationale comes from EJB — a checked exception is
by Java's own contract an anticipated alternative outcome the caller is expected to
handle, and an anticipated outcome shouldn't unilaterally destroy the caller's work,
whereas an unchecked exception signals a defect after which the state is untrustworthy.
The rationale is coherent and it still doesn't match what anyone expects from the word
'transactional', which is why it's the most common silent data bug in Spring codebases.
Three fixes: `rollbackFor` on the method, making the domain exception unchecked — usually
right, per Spring's own choice to make the whole data-access hierarchy unchecked — or a
team-wide meta-annotation carrying `rollbackFor = Exception.class` so the default matches
the expectation. In the log it's unmistakable: 'Completing transaction after exception'
immediately followed by 'Initiating transaction commit'."

**What separates them:** knowing the *rationale*, not just the rule; naming the exact log
signature; and proposing a systemic fix rather than a per-method one.

**Follow-up:** "Is there ever a case where committing on an exception is what you want?"
*(Yes — `noRollbackFor` on something like a duplicate-submission exception where the row
you already wrote is the correct final state.)*

---

### Q3 — "What's the difference between `REQUIRES_NEW` and `NESTED`?"

**Mid-level answer:** "`REQUIRES_NEW` starts a new transaction, `NESTED` starts a nested
one inside the current transaction."

**Senior answer:** "They differ in mechanism, in connection count and in failure
semantics. `REQUIRES_NEW` suspends the outer transaction — unbinds its resources from
`TransactionSynchronizationManager` and stashes them — and begins a genuinely independent
physical transaction on a **second** connection. So the thread holds two connections at
once, which at pool size N with N concurrent threads is a deadlock generator. Its
independence is real: it commits even if the outer rolls back, and it can't see the
outer's uncommitted writes. `NESTED` is a JDBC savepoint inside the *same* physical
transaction — one connection, it sees the outer's uncommitted writes, and if the outer
rolls back the nested work goes with it. It's a partial-rollback tool, not an
independence tool. Practical caveats: `JpaTransactionManager` doesn't support savepoints
by default, so `NESTED` often throws at runtime on a JPA stack; and Postgres tracks
subtransactions in shared memory with a documented cliff past 64 per transaction, so a
savepoint per order line is a real performance problem. I use `REQUIRES_NEW` only when a
write must survive the outer rollback — audit trails — and then I check the pool
arithmetic first."

**What separates them:** connection count, the savepoint mechanism, the JPA caveat, the
Postgres subtransaction cliff, and a stated usage rule.

**Follow-up:** "You need an audit row that survives a rollback, and you can't afford a
second connection. What do you do?" *(An outbox row in the same transaction plus a relay
— Topic 115 — or write the audit after the transaction and accept best-effort.)*

---

### Q4 — "What does `readOnly = true` guarantee?"

**Mid-level answer:** "It makes the transaction read-only so you can't write, and it's
faster."

**Senior answer:** "It guarantees nothing about writes. It does three things: it sets a
flag in `TransactionSynchronizationManager`, it puts the Hibernate session into
`FlushMode.MANUAL` so dirty-checked changes are **silently discarded** rather than
rejected, and — only if the transaction manager propagates it — it calls
`Connection.setReadOnly(true)`, which on Postgres issues `SET TRANSACTION READ ONLY` and
*will* reject a write. So the behaviour of an accidental write ranges from 'silently
lost' to 'hard error' depending on configuration, which is the worst possible kind of
ambiguity. I treat it as a performance and intent hint — skipping the flush is a real
saving on a read path with a large persistence context — and never as a safety
mechanism. If I need a guarantee I use a database role without write permission. The
other real use is replica routing: `readOnly` is the standard signal a routing
`DataSource` uses to send a transaction to a read replica."

**What separates them:** naming all three effects, the silent-versus-hard-error
ambiguity, and the replica-routing use.

**Follow-up:** "A read endpoint is `readOnly = true`, someone mutated an entity in it,
and the change didn't persist — but no error. Where does that show up?" *(Nowhere, until
a user reports it. It is the argument for making the read path use projections, not
entities — Topic 47.)*

---

### Q5 — "Where should the transaction boundary go?"

**Mid-level answer:** "On the service method, not the controller or the repository."

**Senior answer:** "Service layer is the right default, and the reason is that the
service method is the unit of business atomicity — a repository boundary makes every
statement its own transaction, and a controller boundary drags request parsing and
serialization inside it. But the more important rule is what goes *inside*: the
transaction contains database work and nothing else. No HTTP calls, no message publishes,
no file I/O, no `Thread.sleep`, no waiting on a lock, because a transaction pins a pool
connection for its entire lifetime and every one of those turns a downstream latency
problem into a connection-pool outage that takes down endpoints that touch no database at
all. Concretely, in `orderflow` at 120 orders per second on a 20-connection pool, Little's
Law gives me a budget of about 166 ms per transaction before the pool is the bottleneck —
so a 180 ms fraud check inside the transaction is not a slow endpoint, it's an outage. I
also set an explicit `timeout` as a blast-radius cap, and I use `TransactionTemplate` when
the atomic region is narrower than a whole method."

**What separates them:** an arithmetic budget rather than a rule of thumb, naming the
blast radius beyond the affected endpoint, and knowing that a narrower-than-a-method
boundary has a first-class tool.

**Follow-up:** "The fraud check has to happen before the reservation, and the product
owner insists it be atomic with it. What do you tell them?" *(That it cannot be — the
external call is not transactional. The design is: check, then transact, and make the
reservation reversible with a compensating action or a saga.)*

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Spring could have implemented `@Transactional` with bytecode weaving at build time,
   advising the real object so self-invocation worked. It chose runtime proxies. What
   does it gain, and what does the self-invocation trap actually cost the ecosystem?

2. A method annotated `@Transactional(readOnly = true)` calls, through a proxy, a method
   annotated `@Transactional(readOnly = false)` with `REQUIRED`. Which flag wins, and
   why is that answer inevitable given how propagation works?

3. `UnexpectedRollbackException` names no business logic in its stack trace. Explain why
   that is *structurally* unavoidable, and what you would add to the codebase to make
   the real cause findable in one step.

4. You are told "we'll use `NESTED` everywhere so partial failures don't lose the whole
   order." Give two independent reasons that is a bad plan — one from Spring, one from
   Postgres.

5. `@TransactionalEventListener(AFTER_COMMIT)` and the transactional outbox both solve
   "publish only if it committed". Precisely which failure does the outbox handle that
   `AFTER_COMMIT` does not, and how large is that window in practice?

6. In your Nest work, an explicit `queryRunner` made the transaction visible in the
   source. Spring's annotation makes it invisible. Argue that the invisible version is
   *better* for a large codebase, then make the strongest case against yourself.

7. A colleague proposes banning `@Transactional` entirely in favour of
   `TransactionTemplate`. As a code review rule, what does that gain, what does it cost,
   and under what team conditions would you accept it?

---

## Quick reference card

### The annotation

```java
@Transactional(
    propagation   = Propagation.REQUIRED,   // REQUIRED | REQUIRES_NEW | NESTED |
                                            // SUPPORTS | NOT_SUPPORTED | MANDATORY | NEVER
    isolation     = Isolation.DEFAULT,      // Topic 55
    readOnly      = false,                  // hint + Hibernate FlushMode.MANUAL
    timeout       = -1,                     // seconds; set it, as a blast-radius cap
    rollbackFor   = { Exception.class },    // SET THIS if you throw checked exceptions
    noRollbackFor = { }                     // commit despite this unchecked exception
)
```

### Rollback rules

| Escapes your method | Default behaviour |
|---|---|
| `RuntimeException` and subclasses | **rollback** |
| `Error` and subclasses | **rollback** |
| Checked `Exception` | **COMMIT** ← the trap |
| Nothing (normal return) | commit |
| Nothing, but an inner `REQUIRED` method threw | `UnexpectedRollbackException` on commit |

### Propagation at a glance

| Level | With an existing transaction | Connections held |
|---|---|---|
| `REQUIRED` | join it | 1 |
| `REQUIRES_NEW` | suspend it, begin a new one | **2** |
| `NESTED` | savepoint in the same one | 1 |
| `SUPPORTS` | join it | 1 |
| `NOT_SUPPORTED` | suspend, run untransacted | 1 (idle) |
| `MANDATORY` | join it (throws if absent) | 1 |
| `NEVER` | throws | 0 |

### The five reasons `@Transactional` silently does nothing

- [ ] Self-invocation — `this.method()` never reaches the proxy
- [ ] `private`, `final` or `static` method — CGLIB cannot override it
- [ ] The object is not a Spring bean — somebody `new`ed it
- [ ] Called from `@PostConstruct` — the proxy does not exist yet (Topic 37)
- [ ] No transaction manager — usually a hand-rolled test context

### Diagnostics, in the order you should reach for them

```java
TransactionSynchronizationManager.isActualTransactionActive()   // is there a tx?
TransactionSynchronizationManager.getCurrentTransactionName()   // whose tx is it?
TransactionSynchronizationManager.isCurrentTransactionReadOnly()
service.getClass().getName()                                    // $$SpringCGLIB$$ ?
AopUtils.isAopProxy(service)
TransactionAspectSupport.currentTransactionStatus().isRollbackOnly()
```

```yaml
logging.level.org.springframework.transaction: TRACE
logging.level.org.springframework.orm.jpa: DEBUG
logging.level.org.hibernate.SQL: DEBUG
```

```sql
SELECT pid, state, now() - xact_start AS age, left(query,60)
FROM pg_stat_activity WHERE state = 'idle in transaction';
```

### Log phrases and what they mean

| Phrase | Meaning |
|---|---|
| `Creating new transaction with name [X]` | X is a boundary; a connection was taken |
| `Participating in existing transaction` | joined; this method's attributes were ignored |
| `Suspending current transaction` | `REQUIRES_NEW`; two connections now held |
| `marking existing transaction as rollback-only` | an inner call failed; the outer is doomed |
| `Completing transaction ... after exception` + `Initiating transaction commit` | **a checked exception just committed** |
| *nothing at all* | the proxy was bypassed |

### Rules to apply without thinking

- The transaction contains database work and **nothing else**.
- Set `rollbackFor` whenever a checked exception can escape.
- Count the connections one thread can hold. If it is more than one, justify it.
- Put the boundary on the method callers actually call.
- `readOnly = true` is a hint, never a guarantee.
- Set an explicit `timeout`.
- Never `catch` inside a transaction unless you have decided what the interceptor should
  conclude from that.

> `[BOOT 3.x DELTA]` Nothing in this topic's *semantics* changed between Boot 3.x and
> 4.x — propagation, rollback rules and the proxy model are the same. Two operational
> differences: the Hibernate parameter-binding logger is
> `org.hibernate.type.descriptor.sql.BasicBinder` on Boot 2.x versus
> `org.hibernate.orm.jdbc.bind` on Boot 3.x and 4.x; and Boot 3.x is out of OSS support
> as of June 2026, so a 3.x codebase gets no free CVE patches. If you interview at a
> shop on 3.5, everything in this doc still applies verbatim.

---

## When would I use this at work?

**1. Reviewing a pull request that adds a service method.**
You look for three things in ten seconds: is the boundary on the method callers actually
call; does anything non-database happen inside it; can a checked exception escape without
`rollbackFor`. That review habit prevents more production incidents than any test suite,
because these bugs are invisible to tests written by the same person who wrote the bug.

**2. Debugging a reconciliation break at 2am.**
Finance reports 14 orders this month with a wallet debit and no payment row. You do not
read the whole service. You check whether the write path crosses a bean boundary, whether
the exception that fires on that path is checked, and whether there is a `catch` between
the write and the return. One of those three is the answer, essentially always.

**3. Capacity planning before a launch.**
Marketing forecasts 5× traffic for a flash sale. You do not guess. You take
`hikaricp.connections.usage` p99 as your transaction duration, apply Little's Law against
your pool size, and produce a number: "at 600 orders/sec with a 166 ms transaction we
need 100 connections, Postgres cannot serve that from 8 vCPUs across 6 replicas, so we
must cut the transaction to 60 ms or shard." That conversation is what Topics 55, 65 and
109 exist to make you capable of.

---

## Connected topics

**Prerequisites — this topic assumes them, and re-read them if any of this was shaky:**
- **08 — Exceptions**: the `Throwable`/`Error`/checked/unchecked hierarchy *is* the
  rollback rule. Spring did not invent a new taxonomy; it consumed Java's.
- **09 — Exception API design**: why Spring made the entire data-access hierarchy
  unchecked, which is the same argument as "make your domain exceptions unchecked".
- **37 — Bean lifecycle**: proxies are created at
  `BeanPostProcessor.postProcessAfterInitialization`, which is why a `@Transactional`
  call from `@PostConstruct` is not transactional.
- **40 — Proxying**: the whole mechanism. This topic is Topic 40 applied to one specific
  interceptor; if the self-invocation trap surprised you here, go back.
- **41 — AOP**: `TransactionInterceptor` is one `MethodInterceptor` in an ordered chain.
  Ordering matters when `@Cacheable` and `@Transactional` are both present.
- **48 — Persistence context**: dirty checking is why `place()` writes without calling
  `save()`, and flush timing is why `readOnly` changes behaviour.
- **52 — Locking**: `@Version` and pessimistic locks live *inside* the boundary this
  topic defines. A lost update is a transaction-scope question first.

**This unlocks:**
- **55 — Isolation and the connection pool**: the immediate sequel. A transaction pins a
  connection for its lifetime; this topic told you where the lifetime starts and stops.
- **65 — Load baseline**: you cannot defend a transaction-duration budget without one.
- **90 — Executors**: hand a transaction's work to a pool thread and the `ThreadLocal`
  binding does not follow.
- **101 — Virtual threads**: the same `ThreadLocal` question, plus a much larger number
  of threads competing for the same fixed pool.
- **109 — HikariCP deadlock**: Trap 5 in full, with the sizing arithmetic and a thread
  dump.
- **115 — Outbox**: closes the crash window that `@TransactionalEventListener(AFTER_COMMIT)`
  leaves open.
- **119 — Tracing / 120 — MDC**: the same `ThreadLocal`-does-not-propagate lesson,
  arrived at from observability rather than persistence.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0, Jakarta EE 11,
Postgres. Everything in this topic — propagation, rollback rules, the proxy model — has
been stable since Spring 3 and is unchanged in Framework 7. The version-sensitive parts
are logger names and the Hibernate flush behaviour, both flagged inline.*
