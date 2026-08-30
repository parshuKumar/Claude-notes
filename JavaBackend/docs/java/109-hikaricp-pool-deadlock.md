# 109 — HikariCP Sizing and the Pool-vs-Thread-Pool Deadlock

## Phase: 11 — Distributed Systems & Production
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21
## Project spine: tune `orderflow`'s connection pool against the Topic 65 baseline and record the latency-versus-pool-size curve. This is the first Phase 11 topic because every later topic — outbox relays, Kafka consumers, tracing spans — sits on top of a connection pool, and a pool you have not measured is a pool you do not have.

---

## Mechanical statement

Read this twice. The rest of the document elaborates it and nothing else.

> **A request thread that holds one connection and asks for a second can deadlock
> the pool.**
>
> With a pool of N connections and N threads, where each thread has taken one
> connection and is now blocked waiting for a second one, no thread can make
> progress. Every thread needs a connection that only another blocked thread can
> release. Nothing times out until `connectionTimeout` fires — by default, thirty
> seconds later — and then every one of them fails at once.
>
> The safe bound is:
>
> ```
> pool >= threads * (simultaneous_connections_per_thread - 1) + 1
> ```
>
> For the common case of two simultaneous connections per thread (an outer
> transaction plus a nested `REQUIRES_NEW`), that reduces to `pool >= threads + 1`.
>
> **And the second half, which matters more in production:** raising the pool to
> escape a timeout usually makes latency worse for everyone. PostgreSQL runs a
> separate operating-system process per connection. A hundred connections against
> eight cores is a hundred processes competing for eight cores. You did not add
> capacity; you moved the queue from your application, where you could see it, into
> the database, where you cannot.

Two failure modes, one resource. The first is a correctness bug that hangs. The
second is a capacity mistake that degrades. They look identical from the outside —
"we are getting connection timeouts" — and the fixes are opposites.

---

## The bridge from what you know

### There is no Node analogue that matters. Say that out loud.

You have used `pg.Pool`. You have set `max: 20`. You may even have exhausted it once.
So your instinct will be that this is the same topic with different syntax.

It is not, and the reason is structural.

```ts
// Node — the shape you are used to
const pool = new pg.Pool({ max: 20 });

async function placeOrder(dto: PlaceOrderDto) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query('UPDATE inventory SET available = available - $1 WHERE sku = $2',
                       [dto.units, dto.sku]);
    await client.query('COMMIT');
  } finally {
    client.release();          // <-- YOU wrote this line. You know it exists.
  }
}
```

Three things are true about that code and false about the Java equivalent:

1. **The connection checkout is a variable you named.** `client` is right there. Its
   lifetime is visible in the source. You can see, by reading, exactly how long it is
   held.
2. **The event loop is not blocked while you hold it.** `await pool.connect()` yields.
   The single Node thread goes off and serves other requests. Ten thousand pending
   checkouts cost you ten thousand promises, not ten thousand threads.
3. **You cannot accidentally take a second connection inside the first.** You would
   have to write `await pool.connect()` again, on a visible line, with a second
   variable name. It would look obviously wrong in review.

Now the Java version:

```java
@Transactional
public void placeOrder(PlaceOrderCommand cmd) {
    inventory.reserve(cmd.sku(), cmd.units());
    wallet.debit(cmd.userId(), cmd.amountMinor());
    payments.record(cmd);
}
```

Where is the connection? Nowhere. It is not a variable. It is not a parameter. It is
not returned. It is bound to the **thread** by
`TransactionSynchronizationManager`, in a `ThreadLocal`, by the proxy that
`@Transactional` created (Topic 40), at the moment the proxy began the transaction.

That single fact generates the entire topic:

| | Node `pg.Pool` | Java + HikariCP + `@Transactional` |
|---|---|---|
| Where the connection lives | a local variable you named | a `ThreadLocal` you never see |
| Who releases it | you, in a `finally` | the transaction interceptor, at commit or rollback |
| Cost of waiting for one | one pending promise | **one blocked platform thread** |
| Can you take a second by accident | no — you would have to write the call | **yes — `REQUIRES_NEW` on a method three classes away does it** |
| Does a slow HTTP call inside hold it | only if you wrote the checkout around it | **yes, always, for the whole transaction** (Topic 55) |

**Verdict: NO ANALOGUE for the deadlock.** The Node pool can be exhausted — that is
the same. The Node pool cannot deadlock in this shape, because getting a second
connection inside the first requires you to write it down, and because a waiting
request costs a promise rather than a thread.

### The one analogue that *is* honest, and where it stops

Node's `pool.connect()` hanging forever when you forget `client.release()` is a real
leak, and it maps cleanly onto HikariCP's leak detection. If your instinct is "the
pool is empty because something is not giving connections back", that instinct is
correct and transfers.

Where it stops: in Node the missing `release()` is a line you can grep for. In Java
the equivalent is a transaction that stayed open — and transactions are opened by
annotations, not by calls. There is no `release()` to grep for. You find it from
`hikaricp.connections.usage` and `pg_stat_activity`, not from reading code.

### The thing your system-design knowledge already tells you, and which is right

You know that a bounded resource pool in front of a slow dependency is a queue, and
that queue depth times service time is wait time. You know Little's Law. You know
that a full queue is a signal, not a problem to be papered over.

All of that transfers exactly. The only new content is: **which resource, held by
whom, released by what, and observable through which instrument.** That is what this
document teaches. It does not re-teach queueing.

---

## What is this?

**HikariCP** is a JDBC connection pool. It is Spring Boot's default `DataSource`
implementation and has been since Boot 2.0. It keeps a set of open TCP connections
to PostgreSQL and hands them to threads that ask.

A **connection** is expensive to create. Opening one means a TCP handshake, a
PostgreSQL startup packet, authentication, and — this is the part people forget —
the PostgreSQL server **forking a new operating-system process** to serve it. That
fork costs milliseconds and the resulting process costs memory for as long as the
connection lives. Pooling exists so you pay that once per connection rather than
once per request.

**Checkout** is a thread asking the pool for a connection. **Return** (HikariCP calls
it `requite`) is giving it back. Between those two points, that connection belongs to
exactly one thread and no other thread can use it.

The three numbers that define pool behaviour:

| Property | Boot key | Default | What it actually controls |
|---|---|---|---|
| Maximum pool size | `spring.datasource.hikari.maximum-pool-size` | 10 | The hard ceiling on concurrent database work from this JVM |
| Connection timeout | `spring.datasource.hikari.connection-timeout` | 30000 ms | How long a thread blocks in checkout before throwing |
| Minimum idle | `spring.datasource.hikari.minimum-idle` | equal to maximum | How many connections are kept open when idle |

HikariCP's own documentation recommends leaving `minimumIdle` equal to
`maximumPoolSize` — a fixed-size pool. The reasoning is that a pool that shrinks and
regrows introduces connection-creation latency exactly when load arrives, which is
the worst possible time.

**Pool exhaustion** is every connection checked out and at least one thread waiting.
That is not automatically a bug. A pool at full utilisation with short waits is a
correctly sized pool. It becomes a bug when the wait exceeds your latency budget, and
it becomes a *deadlock* when the wait can never end.

---

## Why does it matter?

Four reasons, in ascending order of how much they will cost you.

**1. It is the single most common production Java outage shape.**
The symptom is that every endpoint fails, including endpoints that touch no database.
The reason is that the request threads are all blocked in checkout, so Tomcat's
thread pool is empty, so nothing can be served at all. A database problem becomes a
total outage because two pools are chained.

**2. The obvious fix makes it worse.**
"We are timing out on connections, raise the pool" is the reflex. On PostgreSQL that
reflex has a specific, measurable cost, described in Machine-level reality below.
Being the person who says "no, and here is the measurement that shows why" is a large
part of what senior means on a Java team.

**3. The deadlock variant does not appear in a deadlock report.**
`jcmd <pid> Thread.print` detects deadlocks on Java monitors and on
`java.util.concurrent` locks. It will happily tell you "Found one Java-level
deadlock" for a lock-ordering bug (Topic 94). It will print **nothing** for the pool
deadlock, because a connection pool is a counting resource, not a lock, and no
wait-for edge exists in any structure the JVM tracks. You have to read the stack
frames yourself and recognise the shape.

**4. Virtual threads do not fix it — they expose it faster.**
Topic 101 removed the thread ceiling. If `orderflow` runs on virtual threads and the
pool is 10, you now have ten thousand virtual threads queueing for ten connections
instead of two hundred platform threads queueing for ten connections. Throughput is
identical. Latency is worse, because the queue is longer. The bottleneck was never
the threads.

---

## Machine-level reality

### Part 1 — What `dataSource.getConnection()` actually executes

HikariCP's core data structure is `com.zaxxer.hikari.util.ConcurrentBag`. It is not a
queue and it is not a `BlockingQueue`. It is a purpose-built structure with three
parts:

**(a) A shared list.** A `CopyOnWriteArrayList` of every `PoolEntry` in the pool.
Copy-on-write is chosen because the list changes only when the pool grows or shrinks,
which is rare, while it is *read* on every checkout, which is constant. Reads are
therefore lock-free array traversals.

**(b) A thread-local list.** Each thread keeps a small list of the entries it has
recently used. On checkout, this is scanned **first**. This is the piece with real
consequences: HikariCP deliberately gives a thread back the connection it just had.

Why: cache locality of the `PoolEntry` and its `ProxyConnection` wrapper, and — more
importantly on the PostgreSQL side — the server-side prepared-statement cache and any
session state live per connection. Reusing the same connection means the statement
cache is warm.

> The thread-local list is bounded, so a thread cannot hoard the pool. The exact
> bound is a constant in `ConcurrentBag.java`. **I am not going to quote a number I
> cannot verify against your version.** Open the source for the HikariCP version on
> your classpath (`mvn dependency:tree | grep HikariCP` to find it) and read
> `ConcurrentBag.requite`. This is a one-minute check and it is the kind of check
> that makes the difference between believing a blog post and knowing.

**(c) A handoff queue.** A `SynchronousQueue`. A `SynchronousQueue` has no capacity at
all — an `offer` succeeds only if a consumer is already waiting to `poll`. It is a
rendezvous, not a buffer. This is the mechanism by which a thread returning a
connection hands it *directly* to a thread that is waiting for one, with no trip
through the shared list.

The checkout path, in order:

```
ConcurrentBag.borrow(timeout, unit):
  1. Walk this thread's thread-local list, most-recent-first.
     For each entry, CAS its state NOT_IN_USE -> IN_USE.
     First successful CAS wins: return that connection. No blocking, no allocation.

  2. No local hit. Increment the waiters counter.
     Walk the shared CopyOnWriteArrayList.
     For each entry, CAS NOT_IN_USE -> IN_USE. First success wins.

  3. Still nothing. Ask the pool's "addConnection" executor to create one,
     if the pool is below maximumPoolSize.

  4. Block: handoffQueue.poll(remainingTimeout).
     THIS is where the thread parks. This is the frame you will see in the dump.

  5. Timeout expires with nothing handed off:
     throw SQLTransientConnectionException.
```

The return path:

```
ConcurrentBag.requite(entry):
  1. Set state IN_USE -> NOT_IN_USE.
  2. Spin a bounded number of times: if waiters > 0, handoffQueue.offer(entry),
     then Thread.yield(). This is the direct handoff.
  3. Otherwise put it on this thread's thread-local list (if there is room)
     and return.
```

**The consequence you need:** a thread waiting for a connection is parked in
`SynchronousQueue.poll`, called from `ConcurrentBag.borrow`, called from
`HikariPool.getConnection`. Its `Thread.State` is `TIMED_WAITING`. It is not spinning,
not consuming CPU, and — crucially — **not holding a lock that the JVM knows about.**

### Part 2 — Why the JVM cannot detect this deadlock

The JVM's deadlock detector builds a wait-for graph over two edge types:

- Thread A is blocked entering a `synchronized` block on monitor M, and thread B owns
  M. (Edge A → B.)
- Thread A is blocked in `AbstractQueuedSynchronizer` acquisition of lock L, and
  thread B holds L. (Edge A → B.)

A cycle in that graph is a deadlock, and `jcmd Thread.print` prints it under a
`Found one Java-level deadlock:` heading.

Now look at the pool deadlock. Thread A holds `PoolEntry#3`. Is `PoolEntry#3` a
monitor? No. Is it an AQS lock? No. It is an object whose `state` field was moved from
`0` to `1` by a compare-and-swap. There is no ownership record anywhere the JVM
inspects. Thread A is parked in a `SynchronousQueue`, which the JVM sees as "parked on
a condition", with no owner attributed.

So the deadlock detector reports nothing, and the dump shows N threads all
`TIMED_WAITING (parking)` in the same frames. **Recognising that pattern is the
skill.** It is not automated.

Topic 98 gave you the four Coffman conditions. Check them against this:

| Coffman condition | Pool deadlock |
|---|---|
| **Mutual exclusion** | A checked-out connection is held by exactly one thread. Yes. |
| **Hold and wait** | The thread holds connection #1 while blocking for connection #2. Yes. |
| **No preemption** | Nothing takes a connection away from a thread. There is no such API. Yes. |
| **Circular wait** | Generalised form: the set of blocked threads collectively holds every instance of the resource, and each needs one more than it holds. Yes. |

**Topic 109 is Topic 94's deadlock with connections as the resource.** The only
difference is that the resource has N interchangeable instances instead of one named
lock, which turns a strict cycle into a counting argument — and which is exactly why
the automated detector misses it.

### Part 3 — Why PostgreSQL punishes a large pool

This is the half people skip, and it is the half that separates senior from mid.

PostgreSQL uses a **process-per-connection** model. `postmaster` accepts a connection
and `fork()`s a backend process to serve it. Not a thread. A process, with its own
address space, its own file descriptors, its own private memory for `work_mem`
sorting and hashing, and its own entry in the shared `ProcArray`.

Four costs scale with connection count:

**(a) Memory.** Each backend has private memory beyond the shared buffer pool. The
exact figure depends on your workload — a backend running a big sort with
`work_mem = 64MB` can spike far above baseline, and `work_mem` is *per sort node per
query*, not per query. Measure yours; do not use a number from a blog.

**(b) Scheduling.** Eight cores can run eight processes. A hundred runnable backends
means the OS scheduler is time-slicing them. Each context switch costs a TLB and
cache disturbance. The work does not get done faster; it gets done in more, smaller,
worse-localised slices. Total throughput falls and every individual query's wall time
rises.

**(c) Snapshot and ProcArray work.** Every transaction taking a snapshot needs to know
which transactions are in flight, which historically meant walking the ProcArray —
work proportional to the number of connections. PostgreSQL 14 substantially improved
this path. It is better than it was; it is not free.

**(d) Lock manager contention.** The shared lock table is partitioned, but more
backends means more contention on those partitions, and `LWLock` waits show up in
`pg_stat_activity.wait_event_type`.

The net effect, which you are going to measure yourself in the Spine exercise: as you
raise `maximumPoolSize` past the point where the database can actually execute that
much concurrency, **throughput plateaus and then declines, while p99 latency climbs
steadily.** There is a knee. Finding it is the job.

The widely-cited starting formula, from the PostgreSQL wiki and repeated in
HikariCP's own sizing guidance:

```
connections = (core_count * 2) + effective_spindle_count
```

`effective_spindle_count` came from an era of rotating disks and represents how many
concurrent I/O operations the storage can service. On NVMe or cloud block storage
that term is genuinely ambiguous. Treat `cores * 2` as a **starting point to measure
from**, never as an answer.

And the number that surprises people: **the database sees `instances × pool`.** Eight
`orderflow` pods with a pool of 20 each is 160 backends. Your pool size is a
per-instance decision with a cluster-wide consequence. This is why `max_connections`
tuning and pool tuning have to be done together, and why PgBouncer in transaction
pooling mode exists.

### Part 4 — Where the connection is bound, and for how long

`@Transactional` (Topic 54) makes the proxy call
`PlatformTransactionManager.getTransaction()`. For JDBC/JPA that acquires a connection
and binds it to the current thread inside
`TransactionSynchronizationManager`'s `ThreadLocal<Map<Object, Object>> resources`,
keyed by the `DataSource`.

From that moment until commit or rollback:

- Every `JdbcTemplate` call, every Hibernate flush, every repository method on that
  thread uses **that** connection. It does not check out a new one.
- The connection is not returned. Not between statements. Not while you are waiting
  on an HTTP call. Not while you are sleeping.
- If the method calls something annotated `@Transactional(propagation = REQUIRES_NEW)`
  through a proxy, the transaction manager **suspends** the current transaction —
  which means it stashes the current connection in the suspended-resources holder and
  goes to the pool for a second one. The first connection is still checked out. It is
  just parked.

That last bullet is the deadlock. Nothing else is needed.

Topic 55 taught you that a transaction pins a connection for its lifetime, and that
an HTTP call inside a transaction converts a downstream blip into pool exhaustion.
This topic is the other half: a *second checkout* inside a transaction converts
normal load into a hang.

---

## Example 1 — minimal

The smallest program that reproduces the deadlock. No Spring, no annotations, no
proxy — because the point is that this is a **pool** property, not a Spring property.
Spring only makes it easy to do by accident.

```java
package com.orderflow.lab;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

import java.sql.Connection;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class PoolDeadlockMinimal {

    private static final int POOL_SIZE = 10;
    private static final int THREADS   = 10;

    public static void main(String[] args) throws Exception {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/orderflow");
        config.setUsername("orderflow");
        config.setPassword("orderflow");
        config.setMaximumPoolSize(POOL_SIZE);
        config.setMinimumIdle(POOL_SIZE);
        config.setPoolName("orderflow-lab");
        // 15 seconds so the drill finishes while you are still watching it.
        config.setConnectionTimeout(15_000);

        try (HikariDataSource ds = new HikariDataSource(config)) {

            // A barrier so that ALL threads hold their first connection before
            // ANY thread asks for its second. Without this the deadlock is
            // probabilistic and you will chase a heisenbug.
            CountDownLatch allHoldOne = new CountDownLatch(THREADS);
            ExecutorService pool = Executors.newFixedThreadPool(THREADS);

            for (int i = 0; i < THREADS; i++) {
                final int id = i;
                pool.submit(() -> {
                    try (Connection outer = ds.getConnection()) {          // 1st checkout
                        outer.setAutoCommit(false);
                        outer.createStatement().execute("select 1");

                        allHoldOne.countDown();
                        allHoldOne.await();          // everyone now holds exactly one

                        System.out.println("thread " + id + " asking for a SECOND");
                        try (Connection inner = ds.getConnection()) {      // 2nd checkout
                            inner.createStatement().execute("select 1");
                            System.out.println("thread " + id + " GOT the second");
                        }
                        outer.rollback();
                    } catch (Exception e) {
                        System.out.println("thread " + id + " FAILED: "
                                + e.getClass().getSimpleName() + ": " + e.getMessage());
                    }
                });
            }

            pool.shutdown();
            pool.awaitTermination(2, TimeUnit.MINUTES);
        }
    }
}
```

What to reason about before you run it:

- Ten connections exist. Ten threads take one each. The pool now has zero available.
- Every thread then asks for a second. Every ask goes to step 4 of `borrow` and parks.
- No thread can reach `outer.rollback()`, because it is stuck on the line above.
- Therefore no connection is returned. Therefore no handoff. Therefore nothing.
- Fifteen seconds later, all ten throw `SQLTransientConnectionException`.

Now change one line:

```java
config.setMaximumPoolSize(11);        // THREADS * (2 - 1) + 1 = 11
```

Now one thread gets its second connection, does its work, releases both, and the
released connections unblock the next thread. The whole thing drains, serially. It
completes. It is also **ten times slower than it should be**, because you have turned
a parallel workload into a serial one — which is the honest cost of the safe bound,
and why the real fix is to stop taking two connections.

> Both variants must be run against a real PostgreSQL. `docker run --rm -p 5432:5432
> -e POSTGRES_PASSWORD=orderflow -e POSTGRES_USER=orderflow -e POSTGRES_DB=orderflow
> postgres:17` is enough. I have no database and am not going to print what this
> outputs.

---

## Example 2 — production scenario (on the project spine)

### The constraints, stated up front

From the Topic 65 baseline, `orderflow` runs as:

- Three container replicas, `--cpus=2 --memory=2g` each.
- Tomcat `server.tomcat.threads.max=200` (Boot's default) per replica.
- `spring.datasource.hikari.maximum-pool-size=10` (HikariCP's default — nobody chose
  it, which is the first problem).
- PostgreSQL 17 on 8 cores, `max_connections=100`.
- Dataset: 100k products, 1M orders, 5M order lines.
- Load mix: 70% catalogue read, 20% order read, 10% order placement.
- SLO under discussion: `POST /orders` p99 under 400 ms, error rate under 0.1%.

Three replicas × pool 10 = 30 backends. That is comfortable against
`max_connections=100`, with room for migrations, the load generator's own
connections, and a human with `psql` open.

### The change that introduces the bug

A requirement lands: **the audit record for a failed order placement must survive the
rollback.** If a customer's payment is declined and the order rolls back, compliance
still wants a row saying the attempt happened.

This is a completely legitimate requirement, and `REQUIRES_NEW` is the textbook Spring
answer to it. Here is the code that ships.

```java
package com.orderflow.orders;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private final InventoryService inventory;
    private final WalletService wallet;
    private final PaymentService payments;
    private final OrderAuditService audit;          // <-- new collaborator
    private final OrderRepository orders;

    public OrderService(InventoryService inventory, WalletService wallet,
                        PaymentService payments, OrderAuditService audit,
                        OrderRepository orders) {
        this.inventory = inventory;
        this.wallet = wallet;
        this.payments = payments;
        this.audit = audit;
        this.orders = orders;
    }

    @Transactional                                   // outer transaction: connection #1
    public Order place(PlaceOrderCommand cmd) {

        // A separate bean, so this call DOES go through the proxy.
        // That is why the REQUIRES_NEW actually takes effect —
        // and why the bug is real rather than a silent no-op (Topic 40).
        audit.recordAttempt(cmd.idempotencyKey(), cmd.userId());   // connection #2

        inventory.reserve(cmd.sku(), cmd.units());
        wallet.debit(cmd.userId(), cmd.amountMinor());
        Payment p = payments.authorise(cmd);
        return orders.save(Order.from(cmd, p));
    }
}
```

```java
package com.orderflow.orders;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderAuditService {

    private final OrderAuditRepository repo;

    public OrderAuditService(OrderAuditRepository repo) { this.repo = repo; }

    /**
     * REQUIRES_NEW so the audit row survives a rollback of the caller.
     * The requirement is right. The mechanism is a pool hazard.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordAttempt(String idempotencyKey, long userId) {
        repo.save(new OrderAttempt(idempotencyKey, userId, Instant.now()));
    }
}
```

### What happens at the Topic 65 baseline

Trace one request thread:

1. `place()` is called through the proxy. The transaction manager checks out
   connection **#1** from a pool of 10 and binds it to this thread.
2. `audit.recordAttempt()` is called through *its* proxy. Propagation is
   `REQUIRES_NEW`, so the manager **suspends** the outer transaction — connection #1
   is stashed, still checked out — and asks the pool for connection **#2**.
3. If a connection is available, this takes microseconds and nobody notices.

Now run ten of those concurrently on one replica.

- Ten request threads each complete step 1. The pool has ten connections. All ten are
  now checked out.
- All ten threads reach step 2. All ten call `HikariPool.getConnection()`. All ten
  park in `SynchronousQueue.poll`.
- Nothing can return a connection, because every connection's owner is parked.
- Thirty seconds later — the default `connection-timeout` — all ten throw.

And here is why this is an **outage**, not a slow endpoint:

Tomcat's 200 request threads are a pool too. Every thread stuck for thirty seconds in
`HikariPool.getConnection` is a Tomcat thread that cannot serve anything else. Under
the 10% order-placement mix at baseline throughput, the arrival rate of new order
placements consumes Tomcat threads faster than the thirty-second timeout frees them.

Within a minute, all 200 Tomcat threads on that replica are parked in pool checkout.
`GET /products/{sku}` — which needs one connection and would complete in single-digit
milliseconds — never gets a thread to run on. The catalogue endpoint, which is 70% of
your traffic and has nothing to do with orders, returns nothing at all.

**One pool blocking causes another pool to fill.** That is the pool-vs-thread-pool
part of the title, and it is why the k6 report shows every endpoint failing while the
database sits nearly idle.

### The instruments that name it in under two minutes

```bash
# 1. Is the app-side pool the bottleneck?
curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending
curl -s localhost:8080/actuator/metrics/hikaricp.connections.active
curl -s localhost:8080/actuator/metrics/hikaricp.connections.timeout
```

| What to look for | What it means |
|---|---|
| `pending` high and steady, `active` pinned at `max`, `timeout` climbing | Pool exhaustion confirmed. Now find out whether it is a deadlock or a slow query. |
| `pending` near zero, `active` well below `max` | The pool is not your problem. Look at Tomcat threads (`tomcat.threads.busy`) or upstream. |

```bash
# 2. Deadlock or slow queries? Ask the database what its backends are doing.
psql -h localhost -U orderflow -d orderflow -c "
  select state, count(*), max(now() - state_change) as longest
  from pg_stat_activity
  where datname = 'orderflow'
  group by state order by count desc;"
```

| What to look for | What it means |
|---|---|
| Most backends `idle in transaction`, `longest` growing without bound | **The deadlock.** The database is doing nothing; the application is holding transactions open while blocked in Java. This is the decisive observation. |
| Most backends `active`, `longest` in the seconds, queries visible | Not a deadlock. Slow queries are holding connections. Fix the query (Topic 50), do not raise the pool. |
| Backends `active` with `wait_event_type = 'Lock'` | Row-level lock contention — Topic 52's territory, not this topic's. |
| Backend count near `max_connections` | Someone already "fixed" this by raising the pool. Undo it. |

```bash
# 3. Confirm it in the JVM. This is the proof.
jcmd $(pgrep -f orderflow) Thread.print > /tmp/orderflow-dump.txt
grep -c 'HikariPool.getConnection' /tmp/orderflow-dump.txt
grep -c 'Found one Java-level deadlock' /tmp/orderflow-dump.txt
```

| What to look for | What it means |
|---|---|
| First count equals or approaches the pool size (or Tomcat thread count) | Every one of those threads is parked in checkout. Confirmed. |
| Second count is `0` | **Expected, and the point.** The JVM does not consider this a deadlock. Absence of a deadlock section is not absence of a deadlock. |
| First count is 0 | Threads are blocked somewhere else. Look at the top frames of the busiest thread group instead. |

### The fix, in the order you should consider it

**Fix 1 — do not take a second connection (correct, and usually possible).**

The audit row does not need a second transaction. It needs to be written *outside* the
order transaction. Restructure the boundary:

```java
@Service
public class OrderPlacementFacade {

    private final OrderAuditService audit;
    private final OrderService orders;

    public OrderPlacementFacade(OrderAuditService audit, OrderService orders) {
        this.audit = audit;
        this.orders = orders;
    }

    /** NOT transactional. Two sequential transactions, never nested. */
    public Order place(PlaceOrderCommand cmd) {
        audit.recordAttempt(cmd.idempotencyKey(), cmd.userId());  // txn 1: opens, commits, RELEASES
        return orders.place(cmd);                                  // txn 2: opens, commits, releases
    }
}
```

`recordAttempt` becomes `@Transactional` with default `REQUIRED` propagation. Because
no transaction is active when the facade calls it, it starts its own, commits, and
**returns its connection** before `orders.place` asks for one. Maximum simultaneous
connections per thread: **one**.

The audit row still survives a rollback of the order — better than before, in fact,
because it is committed before the order transaction even begins. And you can now
reason about connection lifetime by reading the facade.

**Fix 2 — if nesting is genuinely unavoidable, size for it.**

```
pool >= threads * (simultaneous - 1) + 1
```

With Tomcat at 200 threads and 2 simultaneous connections, that is `201`. Multiplied
by three replicas, that is 603 PostgreSQL backends. Against `max_connections=100` on
8 cores, that is not a plan; it is a different outage.

Which tells you something important: **the safe bound is usually a proof that the
design is wrong, not a configuration to apply.** When the bound gives you an
impossible number, that is the formula telling you to restructure.

The bound is applicable when the concurrency is small and bounded — a background
reconciliation job with a dedicated 4-thread executor, for instance. There,
`4 * (2-1) + 1 = 5` is a perfectly reasonable dedicated pool.

**Fix 3 — bound the concurrency that can reach the nested path.**

If you cannot restructure and cannot size the pool, cap how many threads can be inside
the nested region at once. This is Topic 97's `Semaphore` used as a bulkhead, and it
is the same shape as Resilience4j's bulkhead in Topic 111.

```java
@Service
public class OrderService {

    // At most 4 threads may be inside the nested-connection region.
    // 4 * (2 - 1) + 1 = 5 <= pool size of 10. Provably deadlock-free.
    private final Semaphore nestedBudget = new Semaphore(4);

    @Transactional
    public Order place(PlaceOrderCommand cmd) {
        try {
            // Bounded wait: fail fast rather than adding an unbounded queue.
            if (!nestedBudget.tryAcquire(200, TimeUnit.MILLISECONDS)) {
                throw new AuditBudgetExhaustedException(cmd.idempotencyKey());
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException(e);
        }
        try {
            audit.recordAttempt(cmd.idempotencyKey(), cmd.userId());
        } finally {
            nestedBudget.release();
        }
        // ... rest of the order placement
    }
}
```

Read the arithmetic in the comment carefully, because it is the whole justification.
Four permitted threads, each able to hold two connections, needs at most
`4 * 1 + 1 = 5` connections to guarantee one thread can always finish. The pool is 10.
Deadlock is now impossible by construction, not by hope.

The cost is honest and you should state it in review: under a burst you will reject
some order placements with a 503 after 200 ms rather than hanging all of them for 30
seconds. That is a better failure. It is not a free one.

**Fix 4 — a separate DataSource for the nested work.**

Two pools, so the nested checkout cannot compete with the outer one. This does remove
the deadlock. It also doubles your PostgreSQL backend count and adds a second thing to
size and monitor. Reach for it only when the nested work is genuinely a different
workload with a different latency profile — an outbox relay, say (Topic 115) — not as
a way to avoid restructuring.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — "We raised the pool to 100 to fix the timeouts"

**Wrong:**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 100      # was 10. "The timeouts stopped!"
```

**Exact symptom:** the `SQLTransientConnectionException` timeouts do stop. In their
place:

- p50 latency on **every** endpoint rises, including the catalogue reads that were
  fast before.
- p99 rises much more than p50 — the tail spreads.
- Throughput at the Topic 65 baseline is flat or lower than it was.
- On the database host, `%sys` CPU and context-switch rate (`vmstat 1`, the `cs`
  column) climb sharply.
- `pg_stat_activity` shows a large number of `active` backends, each individually slow.
- With three replicas at pool 100, `max_connections=100` is exceeded and new
  connections fail with `FATAL: sorry, too many clients already` — which also locks
  *you* out of `psql` during the incident.

**Root cause:** two of them, and you must name both.

First, the timeouts were never a signal that the pool was too small. They were a
signal that connections were being **held too long**. The three causes, in the order
you should check them: a transaction with a network call inside it (Topic 55), a
transaction with a second checkout inside it (this topic), or a genuinely slow query
(Topic 50). Raising the pool treats none of them; it just raises the number of
concurrent slow things.

Second, PostgreSQL's process-per-connection model means the pool size *is* a
concurrency limit on the database, and removing that limit does not create capacity.
Eight cores execute eight things. A hundred runnable backends means each query's work
is split across more, shorter scheduler slices, with cache and TLB disturbance at each
switch. Every query gets slower. Throughput does not rise.

**Fix:** put it back, then find the hold time.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 16                # 8 cores * 2 — a starting point to measure from
      minimum-idle: 16
      connection-timeout: 3000             # fail in 3s, not 30s. See Trap 5.
      leak-detection-threshold: 5000       # log any connection held over 5s
      pool-name: orderflow
```

Then measure `hikaricp.connections.usage` — the timer for how long connections are
held. If its p99 is in the hundreds of milliseconds, that is your bug and no pool size
fixes it.

**The sentence to say in the incident review:** "The pool size is a concurrency limit
on the database. Raising it moves the queue from a place we can measure into a place
we cannot."

---

### Trap 2 — a nested `REQUIRES_NEW` that nobody knows is nested

**Wrong:** a `@Transactional` service method that calls another bean, three layers
down, whose method is annotated `@Transactional(propagation = REQUIRES_NEW)`.

**Exact symptom:**

- The application hangs under load with no CPU usage and no database activity.
- `hikaricp.connections.active` equals `hikaricp.connections.max`; `pending` climbs
  and never falls.
- Exactly `connection-timeout` milliseconds later, a burst of
  `SQLTransientConnectionException` all at once.
- `pg_stat_activity` shows backends in state `idle in transaction`, not `active`. The
  database has nothing to do.
- A thread dump shows N threads all `TIMED_WAITING` in `HikariPool.getConnection`.
- **`jcmd Thread.print` reports no Java-level deadlock.**
- Load tests at low concurrency pass. It only appears when concurrent order placements
  approach the pool size.

**Root cause:** `REQUIRES_NEW` suspends the current transaction without releasing its
connection, then requests a second one. Hold-and-wait on a counting resource with no
preemption.

**Fix:** find every one of them first.

```bash
# Every explicit REQUIRES_NEW in the codebase.
grep -rn "REQUIRES_NEW\|Propagation.REQUIRES_NEW" --include=*.java src/

# Every @Transactional, so you can build the call graph by hand where it matters.
grep -rn "@Transactional" --include=*.java src/ | wc -l
```

Then restructure to sequential transactions (Fix 1 above), or bound the concurrency
with a semaphore (Fix 3), or size to the safe bound if the thread count is small.

Make it a permanent architectural rule and write it down: **no transaction may
contain a second connection checkout.** It is enforceable in review because
`REQUIRES_NEW` is greppable.

---

### Trap 3 — `open-session-in-view` holding a connection through view rendering

**Wrong:** leaving Spring Boot's default in place.

```yaml
# This is the DEFAULT if you say nothing. Boot logs a warning about it at startup.
spring:
  jpa:
    open-in-view: true
```

**Exact symptom:**

- `hikaricp.connections.usage` p99 is roughly equal to your **full HTTP request
  duration**, not your query duration. That equality is the tell.
- Pool exhaustion under a load level that the database is nowhere near saturated by.
- Slow *clients* cause database connection timeouts. A mobile client on a bad
  connection reading a response body slowly holds a PostgreSQL backend.
- The startup log contains `spring.jpa.open-in-view is enabled by default`.

**Root cause:** OSIV keeps the Hibernate `Session` — and therefore its JDBC connection
— open until the response is fully written. This was designed to stop
`LazyInitializationException` during JSON serialisation (Topic 49). It works. The
price is that connection hold time becomes request duration.

Topic 49's drill told you this would show up here. This is where it shows up.

**Fix:**

```yaml
spring:
  jpa:
    open-in-view: false
```

Then fix the `LazyInitializationException`s that appear — properly, with DTO
projections or explicit fetch joins (Topic 50), which is what you should have done in
the first place. Turning OSIV off will break some endpoints. That breakage is
information: each one is an endpoint that was serialising an entity graph out of the
persistence context.

---

### Trap 4 — a leaked connection from manual JDBC

**Wrong:**
```java
public int countPendingOrders() {
    Connection c = dataSource.getConnection();          // no try-with-resources
    PreparedStatement ps = c.prepareStatement(
        "select count(*) from orders where status = 'PENDING'");
    ResultSet rs = ps.executeQuery();
    rs.next();
    return rs.getInt(1);                                // returns; c is never closed
}
```

**Exact symptom:**

- `hikaricp.connections.active` rises monotonically over hours or days and never
  returns to baseline, including during quiet periods overnight.
- `hikaricp.connections.idle` falls to zero over the same period.
- Restarting the service "fixes" it for a while, which convinces people it is a memory
  leak. It is not.
- Eventually, total exhaustion, with a stack trace pointing at whichever unlucky
  endpoint asked next.

**Root cause:** `close()` on a HikariCP connection is what returns it to the pool.
HikariCP hands you a `ProxyConnection`, not the real driver connection, and its
`close()` calls `requite`. No `close()`, no return. The garbage collector will not
save you: the pool holds a strong reference to every `PoolEntry`, so the connection is
reachable forever.

**Fix — two parts, and do both.**

Part one, the code:
```java
public int countPendingOrders() {
    try (Connection c = dataSource.getConnection();
         PreparedStatement ps = c.prepareStatement(
             "select count(*) from orders where status = 'PENDING'");
         ResultSet rs = ps.executeQuery()) {
        rs.next();
        return rs.getInt(1);
    } catch (SQLException e) {
        throw new DataAccessResourceFailureException("counting pending orders", e);
    }
}
```

Part two, the instrument that finds the next one automatically:
```yaml
spring:
  datasource:
    hikari:
      leak-detection-threshold: 5000     # milliseconds; minimum permitted is 2000
```

When a connection is held longer than the threshold, HikariCP logs a warning
containing **the stack trace of the code that acquired it**. That is the single most
useful diagnostic in this entire topic, because it names the acquiring line rather
than the line that eventually failed.

Do not leave this on permanently at a low threshold in production if you have
legitimately long transactions — you will drown in warnings and stop reading them.
Set it just above your genuine p99 hold time.

---

### Trap 5 — a 30-second `connection-timeout` in a service with a 400 ms SLO

**Wrong:** leaving the default.

```yaml
# The default. Also, the value nobody ever changes.
spring:
  datasource:
    hikari:
      connection-timeout: 30000
```

**Exact symptom:**

- During a pool incident, requests do not fail — they **hang**, for thirty seconds
  each.
- Tomcat's `threads.busy` climbs to `threads.max` within a minute or two.
- Endpoints with no database access start failing, because there is no thread to run
  them on.
- Upstream clients have already timed out at, say, 5 seconds and retried, so the
  30-second wait is producing work whose result nobody will ever read — and the retry
  added *more* load (Topic 111).
- The Kubernetes liveness probe eventually fails and the pod restarts, dropping
  in-flight work (Topic 121).

**Root cause:** thirty seconds is longer than every timeout above it in the stack. A
resource wait must be shorter than the SLO of the request waiting on it, or the wait
is pure damage: the caller is gone, the thread is occupied, and the retry storm is
already under way.

**Fix:** make the timeout a fraction of the request SLO.

```yaml
spring:
  datasource:
    hikari:
      connection-timeout: 2000     # p99 SLO is 400ms; 2s means "definitely broken"
      validation-timeout: 1000
      max-lifetime: 1140000        # 19 min — must be shorter than any infra idle timeout
      keepalive-time: 300000       # 5 min
```

`max-lifetime` deserves a sentence of its own: it must be **shorter** than the
shortest connection-idle timeout anywhere between your JVM and PostgreSQL — the
database's own, a cloud load balancer's, a NAT gateway's. HikariCP's guidance is to
set it at least 30 seconds below that value. Otherwise the infrastructure silently
kills a connection that HikariCP still believes is good, and you get an apparently
random `connection reset` on a request that did nothing wrong.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM, no PostgreSQL, and no metrics
endpoint. What follows is the exact setup, the exact command, what to look for, and
how to read every result you might get.

### Setup

```bash
mkdir -p ~/java-lab/109 && cd ~/java-lab/109
java --version                 # expect 21 or 25

docker run -d --name pg109 \
  -e POSTGRES_USER=orderflow -e POSTGRES_PASSWORD=orderflow -e POSTGRES_DB=orderflow \
  -p 5432:5432 postgres:17

curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,postgresql,actuator \
  -d javaVersion=21 \
  -d groupId=com.orderflow -d artifactId=pool-lab \
  -d type=maven-project -o pool-lab.zip && unzip pool-lab.zip -d pool-lab
```

`src/main/resources/application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orderflow
    username: orderflow
    password: orderflow
    hikari:
      pool-name: orderflow
      maximum-pool-size: 10
      minimum-idle: 10
      connection-timeout: 15000
      leak-detection-threshold: 5000
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: update

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  endpoint:
    health:
      show-details: always

logging:
  level:
    com.zaxxer.hikari: DEBUG
    com.zaxxer.hikari.HikariConfig: DEBUG
```

Add Micrometer's Prometheus registry so the Hikari metrics are exported:

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
  <scope>runtime</scope>
</dependency>
```

> Boot binds HikariCP's metrics to Micrometer automatically when both are on the
> classpath and the pool has a name. If `hikaricp.connections.active` does not appear
> in `/actuator/metrics`, check that `pool-name` is set and that you have not defined
> your own `DataSource` bean that bypasses the auto-configuration. The precise
> auto-configuration class name moved during Boot 4's modularisation; if you need to
> read it, find it via `--debug` and the condition-evaluation report (Topic 42)
> rather than trusting a name from memory.

### Proof 1 — see the config the pool actually used

```bash
./mvnw spring-boot:run
```

With `com.zaxxer.hikari.HikariConfig` at `DEBUG`, HikariCP dumps its entire effective
configuration at startup, one property per line.

**What to look for:** `maximumPoolSize`, `minimumIdle`, `connectionTimeout`,
`maxLifetime`, `leakDetectionThreshold`, `poolName`.

| What you see | What it means |
|---|---|
| Values match your YAML | Your configuration is being read. Never assume this; a typo in a property name is silent. |
| `maximumPoolSize=10` when you set something else | Your property is not binding. Check for `spring.datasource.hikari.*` versus a custom `DataSource` `@Bean` that ignores the properties entirely. |
| `leakDetectionThreshold=0` | Leak detection is off. Anything under 2000 is rejected and treated as 0. |
| No config dump at all | The logger name is wrong, or a logging config file is overriding your YAML level. |

### Proof 2 — watch the pool's own stats line

The HikariCP house-keeping task logs pool statistics periodically at `DEBUG`. The
format is:

```
DEBUG com.zaxxer.hikari.pool.HikariPool : orderflow - Pool stats (total=<n>, active=<n>, idle=<n>, waiting=<n>)
```

*Illustration of the format, not captured output. `<n>` are placeholders.*

**How to read it:**

| Reading | What it means |
|---|---|
| `active` well below `total`, `waiting=0` | Healthy. The pool is oversized relative to demand, which is fine and cheap. |
| `active` equals `total`, `waiting=0` | Fully utilised, nobody queueing. This is a **good** state, not a problem. |
| `active` equals `total`, `waiting` > 0 and stable | Saturated with a steady queue. Latency now includes queue time. Compare against your SLO before acting. |
| `active` equals `total`, `waiting` climbing without bound | Exhaustion. Go to Proof 4 and find out whether the database is busy or idle. |
| `total` below `maximumPoolSize` while `waiting` > 0 | The pool is still growing, or connection creation is failing. Check for creation errors in the log. |

### Proof 3 — read the metrics through Actuator

```bash
curl -s localhost:8080/actuator/metrics/hikaricp.connections.active | jq
curl -s localhost:8080/actuator/metrics/hikaricp.connections.idle | jq
curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending | jq
curl -s localhost:8080/actuator/metrics/hikaricp.connections.timeout | jq
curl -s localhost:8080/actuator/metrics/hikaricp.connections.usage | jq
curl -s localhost:8080/actuator/metrics/hikaricp.connections.acquire | jq
```

The two timers are the ones that matter and the two nobody looks at:

| Metric | Question it answers | What a bad value looks like |
|---|---|---|
| `hikaricp.connections.acquire` | How long does a thread wait to *get* a connection? | Above single-digit milliseconds means you are queueing. |
| `hikaricp.connections.usage` | How long does a thread *hold* a connection? | Anything near your HTTP request duration means OSIV or a network call inside a transaction. |

**The diagnostic rule, and it is the most valuable line in this document:**

> If `usage` is high, the pool is not too small — something is holding connections too
> long. Raising the pool will not help and will hurt the database.
>
> If `usage` is low but `acquire` is high, you genuinely have more concurrent demand
> than pool capacity, and a modest increase is justified — after you check what the
> database can actually absorb.

For Prometheus scraping:
```bash
curl -s localhost:8080/actuator/prometheus | grep hikaricp
```

### Proof 4 — ask PostgreSQL what its backends are doing

```bash
docker exec -it pg109 psql -U orderflow -d orderflow -c "
  select pid, state, wait_event_type, wait_event,
         now() - xact_start  as xact_age,
         now() - state_change as in_state,
         left(query, 60)      as query
  from pg_stat_activity
  where datname = 'orderflow' and pid <> pg_backend_pid()
  order by xact_start nulls last;"
```

| What you see | What it means |
|---|---|
| Rows in state `idle in transaction` with a growing `xact_age` | **The pool deadlock, or a transaction blocked on something non-database.** The backend has nothing to do; the JVM is holding the transaction open. Go to the thread dump. |
| Rows in state `active`, `xact_age` in seconds, real queries visible | Slow queries. This is Topic 50's problem. Do not raise the pool. |
| Rows in state `active` with `wait_event_type = 'Lock'` | Row-lock contention. Topic 52. |
| Rows in state `idle` | Pooled connections doing nothing. Normal and expected. |
| Total row count near `max_connections` | Your combined pools are too large for the server. Check `show max_connections;`. |

A cheap guardrail worth having on every `orderflow` database:

```sql
alter database orderflow set idle_in_transaction_session_timeout = '30s';
```

It turns "silent hang forever" into "loud error after thirty seconds", which is
strictly better. It is a safety net, not a fix.

### Proof 5 — the thread dump, and how to read it

```bash
PID=$(jcmd -l | grep pool-lab | awk '{print $1}')
jcmd $PID Thread.print > dump-1.txt
sleep 5
jcmd $PID Thread.print > dump-2.txt

# How many threads are in checkout, in each dump?
grep -c 'HikariPool.getConnection' dump-1.txt dump-2.txt

# Are they the SAME threads? This is the deadlock test.
grep -B 30 'HikariPool.getConnection' dump-1.txt | grep '^"' | sort > t1.txt
grep -B 30 'HikariPool.getConnection' dump-2.txt | grep '^"' | sort > t2.txt
comm -12 t1.txt t2.txt | wc -l

# Does the JVM think this is a deadlock?
grep -A 40 'Found one Java-level deadlock' dump-1.txt
```

**The structure of the frames you are looking for**, so you can recognise it:

```
"http-nio-8080-exec-<n>" #<n> [<n>] daemon prio=5 os_prio=0 cpu=<n>ms elapsed=<n>s tid=<addr> nid=<addr> waiting on condition  [<addr>]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at jdk.internal.misc.Unsafe.park(java.base@21/Native Method)
        - parking to wait for  <<addr>> (a java.util.concurrent.SynchronousQueue$TransferStack)
        at java.util.concurrent.locks.LockSupport.parkNanos(java.base@21/LockSupport.java:<n>)
        at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(java.base@21/SynchronousQueue.java:<n>)
        at java.util.concurrent.SynchronousQueue$TransferStack.transfer(java.base@21/SynchronousQueue.java:<n>)
        at java.util.concurrent.SynchronousQueue.poll(java.base@21/SynchronousQueue.java:<n>)
        at com.zaxxer.hikari.util.ConcurrentBag.borrow(ConcurrentBag.java:<n>)
        at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:<n>)
        at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:<n>)
        at com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:<n>)
        at org.springframework.jdbc.datasource.DataSourceUtils.fetchConnection(...)
        at org.springframework.jdbc.datasource.DataSourceUtils.doGetConnection(...)
        ...
        at com.orderflow.orders.OrderAuditService.recordAttempt(OrderAuditService.java:<n>)
        ...
        at com.orderflow.orders.OrderService.place(OrderService.java:<n>)
```

*Illustration of the format, not captured output. Every `<n>` and `<addr>` is a
placeholder; exact line numbers vary by version and exact frames vary by whether you
are on JDBC, JPA, or a virtual thread.*

**How to read it, frame by frame:**

| Frame | What it tells you |
|---|---|
| `TIMED_WAITING (parking)` | The thread is asleep, using no CPU. It is not slow; it is stopped. |
| `SynchronousQueue$TransferStack` | The Hikari handoff queue. This is a *waiting-for-a-connection* park and nothing else. |
| `ConcurrentBag.borrow` | Step 4 of the checkout algorithm. Confirms the thread-local and shared-list scans both found nothing. |
| Two `HikariPool.getConnection` frames | Normal — the outer one applies the timeout, the inner one does the work. Not a duplicate. |
| **`OrderService.place` further down the same stack** | **The decisive frame.** This thread is *already inside* an outer transaction and is asking for a second connection. That is the deadlock, spelled out. |
| No `Found one Java-level deadlock` section | Expected. The JVM does not track connection ownership. |
| Same thread names in both dumps, five seconds apart | Nothing is progressing. Combined with the frame above, this is conclusive. |

If `orderflow` is running on virtual threads (Topic 101), `Thread.print` will not show
them. Use:

```bash
jcmd $PID Thread.dump_to_file -format=json /tmp/vthreads.json
jq -r '.threadDump.threadContainers[].threads[]
       | select(.stack != null)
       | select(.stack | join("") | contains("HikariPool.getConnection"))
       | .name' /tmp/vthreads.json | wc -l
```

**What to look for:** a count in the thousands. That is the virtual-thread version of
the same picture, and it is worse, because there is no thread-count ceiling to limit
how many can pile up.

---

## Failure drill

**Mandatory.** Produce the failure yourself and write down what you saw before you
read the fix. The point is not the knowledge. It is the memory of watching ten threads
stop, seeing no deadlock reported, and having to work it out from the frames.

### The scenario

Pool size 10. Ten concurrent HTTP requests. Each request opens a transaction, then
calls a method annotated `REQUIRES_NEW` on a different bean.

### Setup

`src/main/java/com/orderflow/lab/OrderAttempt.java`:

```java
package com.orderflow.lab;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "order_attempt")
public class OrderAttempt {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "attempt_seq")
    @SequenceGenerator(name = "attempt_seq", sequenceName = "attempt_seq", allocationSize = 50)
    private Long id;

    @Column(nullable = false)
    private String idempotencyKey;

    @Column(nullable = false)
    private long userId;

    @Column(nullable = false)
    private Instant attemptedAt;

    protected OrderAttempt() { }

    public OrderAttempt(String idempotencyKey, long userId) {
        this.idempotencyKey = idempotencyKey;
        this.userId = userId;
        this.attemptedAt = Instant.now();
    }
}
```

`src/main/java/com/orderflow/lab/AuditService.java`:

```java
package com.orderflow.lab;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class AuditService {

    private final AttemptRepository repo;

    public AuditService(AttemptRepository repo) { this.repo = repo; }

    /**
     * SEPARATE BEAN — so the call from PlacementService goes through the proxy
     * and REQUIRES_NEW actually takes effect. If this were a method on
     * PlacementService itself, Topic 40's self-invocation trap would make the
     * annotation a no-op and this drill would silently pass.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordAttempt(String key, long userId) {
        repo.save(new OrderAttempt(key, userId));
    }
}
```

`src/main/java/com/orderflow/lab/PlacementService.java`:

```java
package com.orderflow.lab;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

@Service
public class PlacementService {

    private final AuditService audit;

    /**
     * A barrier so all ten threads hold their first connection before any asks
     * for a second. Without it the failure is probabilistic and you will waste
     * an hour deciding whether you reproduced it.
     */
    private final CountDownLatch allHoldOne = new CountDownLatch(10);

    public PlacementService(AuditService audit) { this.audit = audit; }

    @Transactional                                          // connection #1
    public String place(String key, long userId) throws InterruptedException {
        allHoldOne.countDown();
        if (!allHoldOne.await(20, TimeUnit.SECONDS)) {
            return "barrier timed out: fewer than 10 concurrent requests arrived";
        }
        audit.recordAttempt(key, userId);                   // connection #2 <-- HERE
        return "placed " + key;
    }
}
```

`src/main/java/com/orderflow/lab/DrillController.java`:

```java
package com.orderflow.lab;

import org.springframework.web.bind.annotation.*;

@RestController
public class DrillController {

    private final PlacementService placement;

    public DrillController(PlacementService placement) { this.placement = placement; }

    @PostMapping("/drill/place")
    public String place(@RequestParam String key, @RequestParam long userId) throws Exception {
        return placement.place(key, userId);
    }
}
```

Confirm the pool is exactly 10:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 10
      connection-timeout: 15000
      pool-name: orderflow
```

### Run it

Terminal 1:
```bash
./mvnw spring-boot:run
```

Terminal 2 — fire exactly ten concurrent requests:
```bash
for i in $(seq 1 10); do
  curl -s -X POST "localhost:8080/drill/place?key=ORD-$i&userId=$i" &
done
```

Terminal 3 — capture evidence **while it is hung**, not after:
```bash
PID=$(jcmd -l | grep pool-lab | awk '{print $1}')

jcmd $PID Thread.print > drill-dump-1.txt
sleep 5
jcmd $PID Thread.print > drill-dump-2.txt

curl -s localhost:8080/actuator/metrics/hikaricp.connections.active  | jq '.measurements'
curl -s localhost:8080/actuator/metrics/hikaricp.connections.idle    | jq '.measurements'
curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending | jq '.measurements'

docker exec pg109 psql -U orderflow -d orderflow -c \
  "select state, count(*) from pg_stat_activity where datname='orderflow' group by state;"
```

### What to capture

Write down these seven things before reading on:

1. `hikaricp.connections.active` while hung.
2. `hikaricp.connections.idle` while hung.
3. `hikaricp.connections.pending` while hung.
4. The count of `HikariPool.getConnection` occurrences in `drill-dump-1.txt`.
5. Whether `drill-dump-1.txt` contains `Found one Java-level deadlock`.
6. The `pg_stat_activity` state breakdown.
7. How long the curls took to fail, and what exception message came back.

### How to read it

| What you see | What it means |
|---|---|
| `active = 10`, `idle = 0`, `pending = 10` | **The drill has fired.** Every connection is held; every thread wants another. |
| Ten stacks parked in `SynchronousQueue.poll` under `ConcurrentBag.borrow` under `HikariPool.getConnection` | The checkout algorithm reached step 4 and blocked for all ten. |
| `PlacementService.place` appears **below** `HikariPool.getConnection` in those same stacks | The decisive frame. Each thread is inside an outer transaction while asking for a second connection. Hold-and-wait, in one screenful. |
| `Found one Java-level deadlock` is **absent** | Expected, and the lesson. Connections are not monitors. The JVM cannot see this. |
| `pg_stat_activity` shows ~10 backends `idle in transaction` and 0 `active` | The database is doing nothing at all. This is not a database problem. It is entirely a Java-side resource cycle. |
| After ~15 s, all ten curls return `SQLTransientConnectionException: orderflow - Connection is not available, request timed out after <n>ms` | The `connectionTimeout` expiring. All at once, because they all blocked at the same moment. |
| Some requests succeed | You did not get ten truly concurrent — the barrier timed out. Check the response text and re-fire. |
| Nothing hangs at all | Your `REQUIRES_NEW` is not taking effect. Confirm `AuditService` is a separate bean, and that `audit.getClass().getName()` contains `$$SpringCGLIB$$` (Topic 40). |
| `pg_stat_activity` shows backends `active` running real queries | You reproduced slow queries, not the deadlock. Different bug, different chapter. |

### Now prove the boundary

Change **one number** and re-run everything, unchanged:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 11        # threads * (simultaneous - 1) + 1 = 10 * 1 + 1
      minimum-idle: 11
```

| What you see | What it means |
|---|---|
| All ten requests complete | The safe bound is not folklore. One spare connection lets one thread finish, and its two released connections cascade to unblock the rest. |
| They complete noticeably serially, not in parallel | **The honest cost.** You converted a deadlock into a serialisation. Throughput at the boundary is roughly one thread at a time. This is why sizing to the bound is a mitigation and not a fix. |
| Still hangs at 11 | Something is taking a *third* connection. Re-derive the bound: three simultaneous needs `10 * 2 + 1 = 21`. Find the third checkout. |

Then remove the `REQUIRES_NEW` entirely — restructure to sequential transactions as in
Example 2's Fix 1 — and re-run with the pool back at 10.

| What you see | What it means |
|---|---|
| All ten complete, and in parallel | The real fix. Maximum simultaneous connections per thread is now one, so the deadlock is impossible at any pool size, and the pool is doing its job instead of being a lock. |

### What the fix proves

Sizing to the bound removes the hang and destroys the concurrency. Removing the second
checkout removes the hang and keeps the concurrency. **Configuration mitigated the
symptom; structure removed the cause.** Being able to state that difference under
pressure — and to show the two runs that demonstrate it — is the whole point of the
drill.

---

## Measurement

### First: why the obvious measurement is wrong

You will be tempted to do this:

```java
// DO NOT DO THIS. Every number it produces is untrustworthy.
long start = System.nanoTime();
try (Connection c = dataSource.getConnection()) {
    // ...
}
System.out.println("acquire took " + (System.nanoTime() - start) + " ns");
```

Four reasons it lies, and you cannot tell which one is lying on any given run:

1. **The first N calls are not the steady state.** The pool is warming, the JIT is
   compiling `borrow` and `requite`, and the thread-local list is empty so every
   checkout falls through to the shared-list scan. You are measuring cold behaviour and
   calling it typical.
2. **The thread-local fast path makes the second call in a thread structurally
   different from the first.** A microbenchmark that loops in one thread measures the
   fast path exclusively and reports a number that no real request will ever see.
3. **`System.nanoTime()` around a blocking call measures wall time, which includes
   scheduler delay, safepoint pauses, and GC pauses that have nothing to do with the
   pool.** At the tail — which is the only part you care about — that noise dominates.
4. **Single-threaded checkout timing tells you nothing about contention**, and
   contention is the entire subject. A pool with zero waiters behaves nothing like a
   pool with fifty.

This is Topic 77's subject in full. Forward-reference it, and treat any published
"HikariCP is X nanoseconds" figure derived from a `nanoTime` loop as fiction.

**And for this topic specifically there is a second, sharper reason:** the number you
need is not "how long does one checkout take". It is **the shape of the latency curve
as pool size varies, under realistic concurrency, measured at the percentile your SLO
names.** A single-threaded timing cannot produce that shape even in principle.

### The measurement that is actually correct: a pool-size sweep

This is the Spine deliverable. It is a load test, not a microbenchmark.

**Method:**

1. Restore the Topic 65 baseline exactly: same dataset (100k products, 1M orders, 5M
   order lines), same scenario mix (70/20/10), same JVM flags, same container limits.
   If you cannot reproduce the baseline within ±10%, stop — Topic 65's gate rule
   applies and any number you produce next is noise.
2. Pick the sweep: `maximum-pool-size` in `{4, 8, 12, 16, 24, 32, 48, 64}`. Set
   `minimum-idle` equal to it each time so the pool is fixed-size and you are not
   measuring growth latency.
3. For each value: restart the service, run a warm-up at 30% of target rate for two
   minutes (to warm the JIT, the page cache, and PostgreSQL's shared buffers),
   discard it, then run the real measurement for at least ten minutes at a **fixed
   open-model arrival rate** — the same rate every time.
4. Record, per run:

| Source | What to record |
|---|---|
| k6 | p50 / p95 / p99 / p999 and error rate, per endpoint |
| k6 | achieved throughput, and whether the generator itself was saturated |
| Micrometer | `hikaricp.connections.acquire` p99 |
| Micrometer | `hikaricp.connections.usage` p99 |
| Micrometer | `hikaricp.connections.pending` maximum |
| Micrometer | `hikaricp.connections.timeout` total |
| Micrometer | `tomcat.threads.busy` maximum |
| PostgreSQL | `select count(*) from pg_stat_activity` peak, and the state breakdown |
| Host | database CPU: `%user`, `%sys`, and `vmstat`'s context-switch rate |

**Critical:** hold the arrival rate constant across the sweep. If you let the load
generator push as hard as it can, you are measuring throughput at saturation for each
configuration, and the latency numbers are not comparable — a classic coordinated-
omission trap that Topic 65 already warned you about.

### What the curve should look like, and what each shape means

You are producing a table with pool size on one axis. Before you run it, commit to a
prediction. Then compare.

| Shape you observe | What it means |
|---|---|
| p99 falls steeply from 4 to some value, then flattens | The flat point is the knee. Below it you were queueing in the application; above it you are not. Pick the knee, not the far end. |
| p99 flattens and then **rises** as pool size grows | You crossed into database saturation. Every added connection makes every query slower. This is the process-per-connection cost, measured on your own hardware. **Getting this graph is the deliverable.** |
| Throughput plateaus while p99 keeps rising | The classic overload signature. Extra concurrency buys no work, only queueing. |
| p99 is flat across the whole sweep | The pool was never the bottleneck. Something else is — profile it (Topic 78) before touching pool configuration again. |
| `usage` p99 is high and roughly constant across all pool sizes | Connections are held too long, and no pool size fixes that. Go find the long hold: OSIV, a network call in a transaction, or a slow query. |
| `acquire` p99 falls as the pool grows but `usage` p99 stays flat | The pool genuinely was undersized for the concurrency. A modest increase is justified — up to the point the database's own curve turns. |
| `pending` maximum is zero at every size | You never saturated the pool at all. Raise the arrival rate or the test proves nothing. |

### The second measurement: hold-time attribution

Once you know p99 `usage`, find out what the time is spent on. Add a timing aspect
(Topic 41) around your transactional boundaries and record:

- time inside the transaction total,
- time inside JDBC execution (from Hibernate `Statistics` or `datasource` metrics),
- the difference.

**That difference is time a PostgreSQL backend was checked out and idle.** It is the
number that tells you whether you have a pool problem or a transaction-boundary
problem, and it is the number that changes the argument in a design review from
opinion to evidence.

### What to write down in `/docs/java/baselines/`

Commit a file per sweep containing: the baseline hash, the JVM flags, the container
limits, `max_connections`, the arrival rate, and the full table. When someone proposes
raising the pool six months from now, that file is the answer, and it is the artefact
Topic 129's capacity model is built from.

---

## Practice exercises

### 1 — Easy: derive and verify the bound

Do the arithmetic before you touch a keyboard.

**Part A.** For each row, compute the minimum deadlock-free pool size using
`pool >= threads * (simultaneous - 1) + 1`:

| Threads | Simultaneous connections per thread | Minimum pool |
|---|---|---|
| 10 | 2 | ? |
| 200 | 2 | ? |
| 8 | 3 | ? |
| 50 | 1 | ? |
| 4 | 4 | ? |

**Part B.** For the `threads = 200, simultaneous = 2` row: you have three replicas.
How many PostgreSQL backends does the safe bound require in total? Compare that to
`max_connections = 100`. Write one sentence explaining what that comparison tells you
about the design.

**Part C.** Explain, in your own words, why the `simultaneous = 1` row needs a pool of
only 1 to be deadlock-free — and why a pool of 1 would nevertheless be a terrible
production choice. Name the distinction you just drew.

**Part D.** Modify `PoolDeadlockMinimal` from Example 1 to take the pool size and
thread count as command-line arguments. Run it across the table above and confirm each
predicted boundary empirically — the largest pool size that still hangs, and the
smallest that completes.

### 2 — Medium: the audit (combines Topics 01–108)

Below is a service from a Spring Boot 4.1 codebase. It contains **six** distinct
defects that will produce connection-pool symptoms in production. Find them all. For
each: name the exact symptom an on-call engineer observes (a metric, a log line, or a
`pg_stat_activity` state — not "it's bad practice"), name the topic it comes from, and
write the fix.

```java
package com.orderflow.settlement;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.client.RestClient;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.ResultSet;
import java.util.ArrayList;
import java.util.List;

@Service
public class SettlementService {

    private final DataSource dataSource;
    private final RestClient paymentGateway;
    private final LedgerService ledger;
    private final SettlementRepository repo;

    public SettlementService(DataSource dataSource, RestClient paymentGateway,
                             LedgerService ledger, SettlementRepository repo) {
        this.dataSource = dataSource;
        this.paymentGateway = paymentGateway;
        this.ledger = ledger;
        this.repo = repo;
    }

    @Transactional
    public void settleAll(java.time.LocalDate day) {
        List<Settlement> pending = repo.findByDayAndStatus(day, "PENDING");

        for (Settlement s : pending) {
            GatewayResult result = paymentGateway.get()
                    .uri("/settlements/{id}", s.getExternalId())
                    .retrieve()
                    .body(GatewayResult.class);

            ledger.recordSettlement(s.getId(), result.amountMinor());

            s.setStatus(result.settled() ? "SETTLED" : "FAILED");
        }
    }

    public List<Long> unreconciledIds(java.time.LocalDate day) {
        List<Long> ids = new ArrayList<>();
        try {
            Connection c = dataSource.getConnection();
            ResultSet rs = c.prepareStatement(
                    "select id from settlement where day = '" + day + "'"
                            + " and reconciled = false").executeQuery();
            while (rs.next()) {
                ids.add(rs.getLong("id"));
            }
        } catch (Exception e) {
            log.error("failed to load unreconciled ids");
        }
        return ids;
    }
}

@Service
class LedgerService {

    private final LedgerRepository ledgerRepo;

    LedgerService(LedgerRepository ledgerRepo) { this.ledgerRepo = ledgerRepo; }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordSettlement(Long settlementId, Long amountMinor) {
        ledgerRepo.save(new LedgerEntry(settlementId, amountMinor));
    }
}
```

Hints, one per defect, in no particular order: Topic 55, Topic 109, Topic 50, Topic
09, Topic 08, and one that is a security problem rather than a pool problem — name it
anyway, because you would flag it in the same review.

**Part B.** Rewrite `settleAll` so that the maximum number of simultaneous connections
held by one thread is one, the transaction never contains a network call, and a failure
partway through leaves the already-settled rows settled. State what property you gave
up to achieve that, and why it is acceptable. (Topics 115 and 116 are where this goes
next; you do not need them to answer.)

### 3 — Hard: production simulation on `orderflow` under load

This is the Spine deliverable. Budget a full session.

**Part A — reproduce the baseline.** Bring up the Topic 65 stack and re-run the
recorded baseline. Confirm p50/p95/p99 for all three endpoint groups land within ±10%
of the committed numbers. If they do not, fix that before continuing; everything below
is comparison against this.

**Part B — the sweep.** Run the pool-size sweep from the Measurement section:
`{4, 8, 12, 16, 24, 32, 48, 64}`, fixed arrival rate, ten-minute measurement windows,
two-minute discarded warm-ups. Produce one table with all the columns listed there.
Plot p99 for `POST /orders` against pool size. Mark the knee.

**Part C — find the database's limit independently.** Using `pgbench` or a direct SQL
driver, and **not** through `orderflow`, find the concurrency at which your PostgreSQL
container's throughput stops rising. Compare that number to the knee from Part B.
Write two sentences on whether they agree, and what it means if they do not.

**Part D — inject the deadlock at scale.** Add the `REQUIRES_NEW` audit write from
Example 2 to `orderflow`'s real order-placement path. Set the pool to your chosen
value from Part B. Run the baseline load. Capture:

- the time from load start to the first `SQLTransientConnectionException`,
- `tomcat.threads.busy` over time,
- the error rate on `GET /products/{sku}` — an endpoint that has nothing to do with
  orders,
- one thread dump taken during the failure.

Write the sentence that explains, to someone who has not read this document, why the
catalogue endpoint failed.

**Part E — fix it three ways and compare.** Implement all three: sequential
transactions, the semaphore bulkhead, and sizing to the safe bound. Run the baseline
against each. Produce a table of p99, throughput, and error rate for all three plus the
broken version. Recommend one, with the trade-off stated explicitly.

**Part F — the argument.** Write the paragraph you would put in a design document
arguing against a proposal to raise the pool to 100. Use your own numbers from Part B
and Part C. It must be persuasive to someone who does not know what a connection pool
costs, and it must not overstate what your data shows. If your sweep did **not** show
latency degrading at large pool sizes, say so honestly and explain what that means
about your test — an honest null result is worth more than a confident wrong one.

---

## Interview questions

### Q1 — "We were getting connection timeouts, so we raised the pool from 10 to 100. Thoughts?"

**Mid-level answer:** "That should fix it — more connections means fewer threads
waiting. Though you should check the database's `max_connections` can handle it."

**Senior answer:** "It will make the timeouts stop and make everything slower, and I
would want to undo it. Two reasons.

First, the diagnosis was wrong. A connection timeout means threads are waiting for
connections. That has three causes and only one of them is 'the pool is too small'.
The other two are far more likely: a transaction holding a connection across a network
call, or a transaction taking a second connection via `REQUIRES_NEW`. I would look at
`hikaricp.connections.usage` before touching pool size — if hold time is high, no pool
size helps.

Second, PostgreSQL is process-per-connection. A hundred connections is a hundred OS
processes. On eight cores that is context-switch thrashing, plus per-backend memory,
plus more contention on shared structures. And it is a hundred *per replica* — with
three replicas that is 300 backends against a default `max_connections` of 100, so new
connections start failing outright and you cannot even get a `psql` session during the
incident.

What I would actually do: put the pool back near `cores × 2` as a starting point,
turn on `leak-detection-threshold`, look at `pg_stat_activity` to see whether the
backends are `active` or `idle in transaction` — that single observation splits the
diagnosis — and then run a pool-size sweep against our load baseline to find the knee
empirically rather than arguing about it."

**What separates them:** the mid-level answer accepts the framing that the pool was
the problem. The senior answer rejects the framing, names the actual likely causes,
knows the specific database cost, catches the per-replica multiplication, and proposes
an experiment rather than a number.

**Follow-up:** "How would you tell, in one command, whether it is a slow query or a
held connection?" They want `pg_stat_activity` grouped by `state` —
`idle in transaction` means the application is holding it, `active` means the database
is working.

---

### Q2 — "The service is hung. No CPU, no database load, and `jcmd Thread.print` reports no deadlock. Where do you go?"

**Mid-level answer:** "If there is no deadlock in the dump, it is probably blocked on
I/O. I would look at what the threads are doing and check whether a downstream service
is slow."

**Senior answer:** "'No deadlock reported' does not mean no deadlock — it means no
deadlock on a monitor or an AQS lock, which are the only two things the JVM's detector
tracks. A connection pool deadlock is invisible to it, because a checked-out connection
is a CAS'd state field, not a lock with an owner.

So I would grep the dump for `HikariPool.getConnection` and count. If that count is
close to the pool size or the Tomcat thread count, that is my answer. Then I would look
*down* those same stacks: if I see a `@Transactional` service method below the
`getConnection` frame, the thread is inside a transaction asking for a second
connection, and that is hold-and-wait on a counting resource.

Two confirmations. Take a second dump five seconds later and check the thread names are
identical — no progress means deadlock rather than slowness. And check
`pg_stat_activity`: if the backends are `idle in transaction` while the application is
hung, the database is doing nothing and the cycle is entirely on our side.

Immediate mitigation is a restart, which is embarrassing but correct. The fix is
finding the second checkout — usually a `REQUIRES_NEW`, occasionally a manual
`dataSource.getConnection()` inside a transactional method — and restructuring so the
transactions are sequential rather than nested."

**What separates them:** knowing *why* the detector misses it, having a specific grep,
reading down the stack rather than only at the top, and using two dumps to distinguish
stuck from slow.

**Follow-up:** "What if the service runs on virtual threads?" They want:
`Thread.print` will not show virtual threads, so use
`jcmd Thread.dump_to_file -format=json`; and virtual threads make it worse, not better,
because there is no thread-count ceiling limiting the pile-up.

---

### Q3 — "Derive the safe pool size for a service with 200 request threads where each request may open a nested `REQUIRES_NEW` transaction."

**Mid-level answer:** "`pool >= threads * (simultaneous - 1) + 1`, so
`200 * 1 + 1 = 201`. I would set the pool to 201."

**Senior answer:** "The arithmetic gives 201, and the fact that it gives 201 is the
answer to a different question.

201 connections per replica, times however many replicas, against a PostgreSQL default
of 100 backends on maybe eight cores. That is not a configuration; it is a proposal to
replace a hang with a database meltdown. When the safe bound produces an impossible
number, the bound is telling you the design is wrong rather than giving you a setting.

So I would use it as a diagnostic and then not apply it. Options in order: remove the
nesting — usually the `REQUIRES_NEW` work can happen in its own transaction before or
after the main one, sequentially, which caps simultaneous connections at one and makes
the deadlock impossible at any pool size. If it genuinely cannot be removed, bound how
many threads may enter the nested region with a semaphore: cap it at four, and now the
bound is `4 * 1 + 1 = 5`, which fits comfortably in a pool of 16.

The formula is genuinely applicable where the thread count is small and fixed — a
background job with a four-thread executor and its own dedicated pool. It is not
applicable to a 200-thread web tier, and knowing which case you are in is the actual
skill."

**What separates them:** the mid-level answer applies the formula. The senior answer
applies it, notices the result is absurd, and treats that absurdity as the signal.

**Follow-up:** "Where does the `+ 1` come from?" They want: one spare guarantees at
least one thread can always acquire its last needed connection, complete, and release
everything — which cascades. It is the same argument as the Banker's algorithm's safe
sequence.

---

### Q4 — "`hikaricp.connections.pending` is consistently above zero. Is that a problem?"

**Mid-level answer:** "Yes — threads are waiting for connections, so the pool is too
small. I would increase it."

**Senior answer:** "Not on its own. A non-zero `pending` means the pool is fully
utilised and there is a queue, which for a bounded resource under load is a normal and
arguably desirable state. A pool that never queues is oversized, and an oversized pool
against PostgreSQL costs latency for everyone.

The question is whether the queue time fits the budget. `hikaricp.connections.acquire`
is a timer — its p99 is the actual wait. If our order SLO is 400 ms p99 and acquire p99
is 3 ms, a non-zero `pending` is fine and I would leave it alone.

What worries me is a *rising* `pending`, or `timeout` incrementing at all. Rising means
arrivals exceed service rate and the queue is unbounded in effect — Little's Law says
the wait grows without limit. Any `timeout` increment means we breached 30 seconds,
which at a 400 ms SLO means the caller left long ago.

And before adjusting the pool I would check `usage` p99. If connections are held for
hundreds of milliseconds, the queue is a symptom of hold time, not of capacity, and I
would go looking for OSIV or a network call inside a transaction rather than adding
connections."

**What separates them:** treating saturation as information rather than as an error,
knowing which metric answers which question, and checking hold time before capacity.

**Follow-up:** "What would you alert on?" Good answers: alert on `timeout` count and on
`acquire` p99 exceeding a fraction of the request SLO — not on `pending` being non-zero,
which would page you for normal operation.

---

### Q5 — "We are moving to virtual threads. Does that fix our connection pool problems?"

**Mid-level answer:** "It should help — virtual threads are cheap, so we can handle
many more concurrent requests without blocking platform threads."

**Senior answer:** "It fixes one problem and sharpens another, and the sharpened one is
usually the one that hurts.

What it fixes: a blocked platform thread was expensive — roughly a megabyte of stack
each, and a hard ceiling at Tomcat's `threads.max`. With virtual threads, a thread
parked in `HikariPool.getConnection` costs almost nothing, so the *thread pool* stops
being the constraint.

What it sharpens: the connection pool was always the real constraint and now nothing
hides it. Previously 200 platform threads queued for 16 connections. Now ten thousand
virtual threads queue for 16 connections. Throughput is identical — the database does
the same work — but the queue is fifty times longer, so queueing latency is far worse
and `connection-timeout` fires for many more requests. You converted a fast rejection
at the thread pool into a slow timeout at the connection pool.

Two consequences. First, with virtual threads you need an explicit concurrency limit in
front of the database — a semaphore or a bulkhead sized to the pool — because the
implicit limit that the thread pool used to provide is gone. Second, the deadlock is
unchanged: the bound still says `threads * (simultaneous - 1) + 1`, and `threads` is
now effectively unbounded, so nesting connections goes from dangerous to guaranteed
fatal.

One operational note: `jcmd Thread.print` does not show virtual threads. You need
`Thread.dump_to_file -format=json`, and if you have not changed your incident runbook,
your first dump during an outage will look empty."

**What separates them:** understanding that virtual threads move a bottleneck rather
than removing one, knowing the implicit limit that disappears, and having thought about
the operational tooling gap.

**Follow-up:** "What happens to the safe bound with virtual threads?" They want: it
becomes unsatisfiable, so nesting must be eliminated structurally rather than sized
around.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. HikariCP hands a thread back the connection it most recently used, via a
   thread-local list. Name one performance benefit and one *diagnostic* consequence —
   specifically, how does that behaviour change what you see when you profile checkout
   latency in a single-threaded microbenchmark versus under real concurrent load?

2. The JVM detects deadlocks on monitors and on AQS locks but not on connection pools.
   Describe, concretely, what HikariCP would have to change for `jcmd Thread.print` to
   report the pool deadlock. Then argue whether it should.

3. `pool >= threads * (simultaneous - 1) + 1` guarantees no deadlock. Prove to
   yourself why the `+ 1` is sufficient — what specifically can the thread holding the
   spare connection do that unblocks everyone else? Now: is the bound *tight*, or is
   there a smaller pool that also works for some workloads?

4. PostgreSQL uses a process per connection; MySQL uses a thread per connection. Does
   the "bigger pool is worse" argument still apply to MySQL? Argue both sides, and name
   the measurement that would settle it for a given deployment.

5. You have three replicas with a pool of 16 each. Autoscaling adds seven more replicas
   under load. What happens at the database, and what does that tell you about where
   pool sizing belongs as a *system* decision rather than a per-service one?

6. Topic 55 said a transaction pins a connection for its lifetime. Topic 101 said
   virtual threads make blocking cheap. Reconcile those: if blocking is cheap, why is
   pinning a connection still expensive? Name exactly what is scarce in each case.

7. Suppose you could preempt a connection — take it back from a thread mid-transaction.
   That would break one of the four Coffman conditions and make this deadlock
   impossible. Explain why no connection pool does this, in terms of what a transaction
   *is*.

---

## Quick reference card

### The two failure modes

| | Deadlock | Undersize / overhold |
|---|---|---|
| Symptom | Total hang, then a synchronised burst of timeouts | Rising latency, some timeouts |
| `pg_stat_activity` | `idle in transaction`, database idle | `active`, database busy |
| Progress between two dumps | None. Same threads, same frames. | Threads change |
| Fix | Remove the second checkout | Fix hold time, then size to the measured knee |
| Made worse by a bigger pool | No — a bigger pool masks it until concurrency rises | **Yes** |

### Configuration that matters

```yaml
spring:
  datasource:
    hikari:
      pool-name: orderflow                # required for readable metrics and logs
      maximum-pool-size: 16               # start at cores * 2; MEASURE from there
      minimum-idle: 16                    # equal to max: fixed-size pool
      connection-timeout: 2000            # a fraction of the request SLO, never 30s
      validation-timeout: 1000
      max-lifetime: 1140000               # >=30s BELOW the shortest infra idle timeout
      keepalive-time: 300000
      leak-detection-threshold: 5000      # min 2000; logs the ACQUIRING stack trace
  jpa:
    open-in-view: false                   # or hold time == request duration
```

### The formula

```
pool >= threads * (simultaneous_connections_per_thread - 1) + 1

threads=10,  simultaneous=2  ->  11
threads=200, simultaneous=2  ->  201   (absurd: fix the design, not the pool)
threads=4,   simultaneous=3  ->  9
threads=N,   simultaneous=1  ->  1     (deadlock-free at any size)
```

Sizing from demand instead, via Little's Law (Topic 90):
```
concurrency_needed = throughput_rps * mean_db_service_time_seconds
```
Then add headroom for burst, and cap by what the database can actually execute.

### Diagnostic commands

```bash
# Metrics
curl -s localhost:8080/actuator/metrics/hikaricp.connections.active
curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending
curl -s localhost:8080/actuator/metrics/hikaricp.connections.timeout
curl -s localhost:8080/actuator/metrics/hikaricp.connections.usage     # HOLD time
curl -s localhost:8080/actuator/metrics/hikaricp.connections.acquire   # WAIT time
curl -s localhost:8080/actuator/prometheus | grep hikaricp

# Thread dumps (platform threads)
jcmd <pid> Thread.print > d1.txt; sleep 5; jcmd <pid> Thread.print > d2.txt
grep -c 'HikariPool.getConnection' d1.txt d2.txt
grep 'Found one Java-level deadlock' d1.txt        # expect NOTHING for a pool deadlock

# Thread dumps (virtual threads, JDK 21+)
jcmd <pid> Thread.dump_to_file -format=json /tmp/v.json

# Database side
psql -c "select state, count(*), max(now()-state_change) from pg_stat_activity
         where datname='orderflow' group by state;"
psql -c "show max_connections;"
psql -c "select count(*) from pg_stat_activity;"

# Find every nested-transaction risk in the codebase
grep -rn "REQUIRES_NEW" --include=*.java src/
grep -rn "dataSource.getConnection" --include=*.java src/
```

### Logging

```properties
logging.level.com.zaxxer.hikari=DEBUG              # periodic pool stats line
logging.level.com.zaxxer.hikari.HikariConfig=DEBUG # full effective config at startup
```

### Gotchas checklist

- [ ] `maximum-pool-size` was chosen from a measurement, not from a default or a guess.
- [ ] `connection-timeout` is shorter than the request SLO, not 30 seconds.
- [ ] `max-lifetime` is at least 30 s below every infrastructure idle timeout in the path.
- [ ] `leak-detection-threshold` is set, above genuine p99 hold time.
- [ ] `spring.jpa.open-in-view` is explicitly `false`.
- [ ] No `@Transactional` method reaches a second connection checkout.
- [ ] No network call is inside a transaction (Topic 55).
- [ ] `replicas × pool` is comfortably below `max_connections`, with headroom for humans.
- [ ] `hikaricp.connections.usage` and `.acquire` are on a dashboard, not just `.active`.
- [ ] The alert is on `timeout` count and `acquire` p99 — not on `pending` being non-zero.
- [ ] The runbook says `Thread.dump_to_file -format=json` if you run virtual threads.

---

## When would I use this at work?

**1. An incident where every endpoint is failing and the database looks fine.**
You run three commands: `pg_stat_activity` grouped by state, a `grep -c` for
`HikariPool.getConnection` on a thread dump, and the same grep on a second dump five
seconds later. Those three answers tell you deadlock versus overhold versus slow
query, and they take ninety seconds. Without them, this is the incident that consumes
an afternoon and ends with a restart and no root cause.

**2. Reviewing a pull request that adds `REQUIRES_NEW`.**
The comment writes itself: "This takes a second connection while holding the first. Our
pool is 16 and Tomcat is 200 threads, so the safe bound is 201 — which we cannot have.
Can the audit write be a separate transaction *before* the order transaction instead?"
That review comment prevents an outage, and it is the single highest-leverage
application of this topic.

**3. Capacity planning before a launch.**
Product says traffic will triple. You already have the pool-size sweep from the Spine
exercise, so you know where the knee is and what the database's independent limit is.
You can say "we have headroom to 2.4× on the current pool, and beyond that we need
either PgBouncer in transaction mode or a read replica for the catalogue path" — with a
graph. That is Topic 129's capacity model, and this measurement is its foundation.

---

## Connected topics

**Prerequisites — you need these to read this topic:**

- **40 — Proxying**: `@Transactional` is a proxy. The nested `REQUIRES_NEW` only takes
  effect because the call crosses a bean boundary. Self-invocation would make this
  drill silently pass.
- **52 — Locking**: row-lock contention looks like pool exhaustion from the outside.
  `pg_stat_activity.wait_event_type = 'Lock'` is how you tell them apart.
- **54 — `@Transactional` I**: propagation semantics. `REQUIRES_NEW` suspends rather
  than releasing, and that word "suspends" is the whole bug.
- **55 — `@Transactional` II**: a transaction pins its connection for its lifetime.
  This topic is the second half of that sentence: it can pin *two*.
- **65 — The load baseline**: no measurement in this topic means anything without it.
- **90 — Executors and Little's Law**: the pool sizing arithmetic and the bounded-queue
  reasoning are the same reasoning applied to a different resource.
- **94 / 98 — Explicit locks and the bug taxonomy**: the four Coffman conditions.
  **Topic 109 is the same deadlock with connections as the resource** — generalised
  from one named lock to N interchangeable instances, which is exactly why the
  automated detector misses it.
- **97 — Coordination primitives**: `Semaphore` as a bulkhead, which is Fix 3.
- **101 — Virtual threads**: they do not fix a pool-bound bottleneck. They remove the
  implicit concurrency limit that was hiding it.
- **105 — Backpressure**: a full pool with a bounded timeout *is* backpressure. An
  unbounded wait is not.

**This unlocks:**

- **111 — Resilience4j**: the bulkhead pattern generalised, and the reason retries into
  a saturated pool make everything worse.
- **114 / 115 / 116 — Kafka delivery, outbox, idempotency**: the outbox relay is a
  second consumer of the same pool, and its polling loop must be sized against it.
- **118 — Micrometer**: `hikaricp.*` is the canonical USE (utilisation, saturation,
  errors) metric set. Utilisation is `active/max`, saturation is `pending`, errors is
  `timeout`.
- **119 — Tracing**: a span around connection acquisition turns "the request was slow"
  into "the request waited 2.1 s for a connection", which is the difference between a
  guess and an answer.
- **121 — Readiness probes**: a readiness check that opens a connection will fail
  during pool exhaustion and remove the pod from load balancing — sometimes correct,
  sometimes an amplifier. Decide deliberately.
- **129 — Capacity modelling**: the pool-size sweep and the database's independent
  concurrency limit are two of the inputs to the capacity model.

---

## `[BOOT 3.x DELTA]`

HikariCP has been Spring Boot's default `DataSource` since Boot 2.0, and every
`spring.datasource.hikari.*` property named in this document binds identically on
Boot 3.5 and Boot 4.1. The pool's behaviour, the deadlock, the formula, and every
diagnostic command are unchanged.

Three differences worth knowing:

1. **Auto-configuration class locations moved** in Boot 4 as part of the codebase
   modularisation. If you are reading Spring's source to understand how the
   `DataSource` or its metrics are wired, the package you remember from 3.x may not
   exist. Find the current one via `--debug` and the condition-evaluation report
   (Topic 42) rather than from memory or from a blog.

2. **`spring.jpa.open-in-view` still defaults to `true`** and still logs a startup
   warning on both lines. Turning it off is the same one-line change and produces the
   same set of newly-visible `LazyInitializationException`s.

3. **Observability packaging differs.** Boot 4 ships an OpenTelemetry starter and
   Micrometer's Hikari binding is wired through the standard metrics auto-configuration
   on both lines. If `hikaricp.*` metrics are missing on either version, the cause is
   almost always the same two things: no `pool-name` set, or a hand-rolled `DataSource`
   `@Bean` that bypasses auto-configuration entirely.

Boot 3.5 left OSS support in June 2026. Nothing in this topic is a reason to stay on
it; the pool behaviour is identical, so a migration here carries no risk from this
document's subject matter.

---

*Java baseline 21, verified against JDK 25 behaviour for the thread-dump commands.
Everything in this topic is a property of HikariCP, JDBC and PostgreSQL rather than of
a Java language version, so it has been true since Boot 2.0 and will remain true. The
one thing that changed materially is the virtual-thread dump command, which requires
JDK 21 or later — and the fact that virtual threads make the pool bottleneck sharper
rather than softer.*
