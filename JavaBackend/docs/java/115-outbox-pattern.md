# 115 — Dual-Write Failure and the Transactional Outbox

## Phase: 11 — Distributed Systems & Production
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21
## Project spine: `orderflow`'s order-placed events move out of the service method and into an `outbox` table written in the **same transaction** as the order. A relay claims batches with `SELECT ... FOR UPDATE SKIP LOCKED` and publishes them to Kafka. The `kill -9` drill proves the window exists before the outbox and is closed after it.

---

## Mechanical statement

Read this twice. Everything else is an elaboration of it.

> **Committing to Postgres and then publishing to Kafka is two non-atomic operations.**
>
> There is a window between them. A `kill -9`, an OOMKill, a node loss, a network
> partition, or an uncaught error inside the window means **the order exists and the
> event does not**. Nothing retries it, because the code that would have retried is
> gone. The loss is permanent and silent.
>
> Reversing the order does not help. Publish first and the crash leaves an **event for
> an order that does not exist**, and every downstream consumer acts on a fact that is
> not true.
>
> **The outbox makes the event a ROW, inserted in the SAME transaction as the order.**
> One commit, one atomic unit, no window. A separate **relay** process reads that table
> afterwards and publishes to Kafka.
>
> The relay can crash too — between publishing and marking the row sent. So the relay
> is **at-least-once by construction**, and consumers must be idempotent (Topic 116).
> You have not eliminated the problem. You have moved it from *silent permanent loss*
> to *visible duplicate delivery*, which is a trade you should make every time.

The critical corollary, stated now because it is the trap nearly everyone misses:

> **`@TransactionalEventListener(phase = AFTER_COMMIT)` has EXACTLY the same window as
> publishing after commit.** It is not a fix. It runs after the commit, in the same
> thread, and a crash between the commit and the listener body loses the event
> identically. Spring gives you a cleaner call site and zero additional durability.

---

## The bridge from what you know

You know the dual-write problem. You have drawn the diagram. So this section skips the
concept and goes straight to the four things that are different in Java and Spring, and
that decide whether your implementation is correct.

### Difference 1 — Spring hands you a tool that looks like the fix and is not

In Node you would write:

```ts
await db.transaction(async (tx) => {
  await tx.insert(orders).values(order);
});
await kafka.send({ topic: 'order-placed', messages: [{ value: JSON.stringify(event) }] });
```

The window is visible. It is on two adjacent lines and nobody is confused about it.

Spring offers this:

```java
@Transactional
public OrderId place(PlaceOrderCommand cmd) {
    var order = orders.insert(cmd);
    events.publishEvent(new OrderPlacedEvent(order.id()));   // deferred
    return order.id();
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onPlaced(OrderPlacedEvent e) {
    kafkaTemplate.send("orderflow.order-placed", e.orderId(), e);
}
```

This *reads* as though Spring is guaranteeing something. It is not. Mechanically:
`ApplicationEventPublisher.publishEvent` inside a transaction registers a
`TransactionSynchronization` on the current transaction. `AFTER_COMMIT` means the
synchronization's `afterCommit()` callback fires from
`TransactionSynchronizationManager` **after** `commit()` returns, on the same thread,
inside the same `try` block in `AbstractPlatformTransactionManager`.

So the sequence is: commit, return, invoke listener. The window between "commit
returned" and "listener body executed" is the identical window. It is smaller in
wall-clock terms and exactly as fatal. Worse, it is now invisible: it is not on two
adjacent lines any more, it is spread across two classes and an annotation, and code
review will not catch it.

**This is the single most important thing in the document.** Say it out loud once.

### Difference 2 — the flush is not the commit

From Topic 48: Hibernate's persistence context batches writes and flushes them at the
transaction boundary (or earlier, under `FlushMode.AUTO`, before a query that touches a
dirty table). So `orders.save(order)` returning does not mean anything reached
Postgres, and `entityManager.flush()` returning does not mean anything is committed.

Concretely, three distinct moments people conflate:

| Moment | What is true | What is not true |
|---|---|---|
| `repository.save(order)` returns | The entity is managed; an ID may have been allocated | Nothing has necessarily reached the database |
| `flush()` returns | The INSERT has reached Postgres | It is inside an open transaction and is invisible to everyone else, and can still roll back |
| the proxy's `commit()` returns | Durable, visible | Nothing about Kafka |

An outbox row inserted through the same `EntityManager` is flushed in the same flush
and committed in the same commit. That is what makes it atomic with the order. If you
insert it through a *different* `DataSource`, or with `REQUIRES_NEW`, or via a native
query on a separate connection, you have re-created the dual write inside your own
process. This is a real mistake and it is hard to see in review.

### Difference 3 — `SKIP LOCKED` is the concurrency primitive, and it is a database feature

Node engineers usually reach for a Redis list or a distributed lock to make a poller
safe with multiple instances. Postgres has a better answer that you already know from
your SQL background but may not have used as a queue primitive:

```sql
SELECT id, aggregate_id, topic, payload
FROM outbox
WHERE status = 'PENDING'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 100;
```

`FOR UPDATE` takes a row-level exclusive lock. `SKIP LOCKED` says: rows another
transaction already locked are **not waited for and not returned** — they are skipped
as though they did not match the predicate. So N relay instances running the identical
query get disjoint batches with no coordination, no leader election, and no
distributed lock.

The Java-side consequence: the locks live for the life of the **transaction**, which
means for the life of the Hikari connection that transaction holds (Topic 55, Topic
109). How long you hold that transaction is the central design decision in the relay,
and it is where most implementations go wrong.

### Difference 4 — no framework does this for you

There is no `@Outbox` annotation. Spring has no outbox abstraction. Spring Modulith
offers an event publication registry that is outbox-shaped and worth knowing about, and
Debezium offers an outbox event router as a Kafka Connect transform. Neither is
automatic and both have real operational surface. **Check the current documentation for
whichever you adopt; I am not going to assert their configuration keys or their exact
delivery semantics from memory.** Most teams write ~150 lines and own it, and that is a
defensible choice because the semantics matter more than the code.

### Verdict table

| What you know | Java/Spring mechanic | Verdict |
|---|---|---|
| Dual write is not atomic | Same problem, same shape | **TRANSFERS** |
| "Publish after commit" is the bug | `@TransactionalEventListener(AFTER_COMMIT)` is the same bug wearing a framework | **PARTIAL, and the delta is the lesson** |
| ORM writes go to the DB when you call save | Hibernate flush vs commit are distinct moments | **NO ANALOGUE** — Topic 48 |
| Poller needs a distributed lock | `FOR UPDATE SKIP LOCKED` needs nothing | **NO ANALOGUE** — and it is better |
| A library will handle it | No standard one; you own the semantics | **NO ANALOGUE** |

---

## What is this?

Three moving parts.

### Part 1 — the outbox table

```sql
create table outbox (
  id             bigserial     primary key,
  aggregate_type text          not null,        -- 'order', 'payment'
  aggregate_id   text          not null,        -- the Kafka message key
  event_type     text          not null,        -- 'OrderPlaced'
  topic          text          not null,
  payload        jsonb         not null,
  headers        jsonb         not null default '{}'::jsonb,
  status         text          not null default 'PENDING',   -- PENDING | SENT
  created_at     timestamptz   not null default now(),
  sent_at        timestamptz,
  attempts       int           not null default 0,
  last_error     text
);

-- The only index the relay needs. Partial, so it stays small as rows are marked SENT.
create index outbox_pending_idx
  on outbox (id)
  where status = 'PENDING';
```

Three deliberate choices to defend in review:

- **`status` column, not a cursor.** See Trap 4; a `WHERE id > last_seen` relay is
  broken, and the reason is a Postgres MVCC fact rather than a coding mistake.
- **Partial index.** Once a row is `SENT` it leaves the index. The index size tracks the
  backlog, not the history. On `orderflow`'s volumes this is the difference between an
  index that stays in cache and one that does not.
- **`aggregate_id` is the Kafka key.** Ordering per aggregate falls out of partitioning
  by key, and nothing else gives you ordering.

### Part 2 — the write path

```java
@Transactional
public OrderId place(PlaceOrderCommand cmd) {
    var order = orders.insert(cmd);
    inventory.reserve(cmd.sku(), cmd.units());
    wallet.debit(cmd.customerId(), cmd.totalMinorUnits());
    outbox.append("order", order.id().value(), "OrderPlaced",
                  "orderflow.order-placed", toJson(order));   // same transaction
    return order.id();
}
```

That is the entire change to the business path. One insert. Same transaction. No Kafka
client on this thread at all — which also removes a network call from inside a
transaction, which was a Topic 55 bug you may not have noticed you had.

### Part 3 — the relay

A separate loop that claims pending rows, publishes them, and marks them sent. It is
the part with all the design decisions, and **Machine-level reality** below is mostly
about it.

Two implementations, and you should be able to argue both:

- **Polling relay.** `SELECT ... FOR UPDATE SKIP LOCKED`, publish, `UPDATE ... SET
  status='SENT'`. Simple, in your language, in your deployment, debuggable with `psql`.
  Costs a poll interval of latency and a steady query load on Postgres.
- **CDC (Debezium).** Postgres logical decoding streams the WAL; a connector reads the
  outbox table's inserts and publishes them. Near-zero latency, no polling load, and
  strictly correct ordering — at the cost of a Kafka Connect cluster, a replication
  slot, and the failure mode where a stalled slot fills your disk and takes the database
  down.

---

## Why does it matter?

**1. It is the difference between "we lost some events" and "we did not".**

At `orderflow`'s baseline — 10% of the k6 mix is order placement — the window between
commit and publish is a few milliseconds. Over a rolling deploy that restarts three
pods, or one OOMKill, or one node preemption, some number of orders exist with no
event. Inventory never decrements for them. The customer is charged and the warehouse
never hears about it. Nothing errors.

**2. The failure is invisible by construction.**

There is no exception, no error metric, no DLQ entry, no lag. The event was never
created, so nothing can observe its absence except a reconciliation query that compares
`orders` to whatever the consumer wrote. Most teams do not have that query until after
the incident.

**3. It is the pattern that makes at-least-once honest.**

Topic 114 concluded: design for at-least-once and make sinks idempotent. That is only
true if the event is *produced* reliably in the first place. The outbox is the producer
side of that contract. Without it, "at-least-once" is actually "at-most-once with extra
steps".

**4. `@TransactionalEventListener` is in a very large number of Spring codebases,
written by people who believe it solved this.**

Being the person who can say precisely why it does not — in one sentence, with the
mechanism — is a senior signal that reliably lands.

---

## Machine-level reality

### `SELECT ... FOR UPDATE SKIP LOCKED`, precisely

Postgres row locks are stored in the tuple header (`xmax` plus infomask bits), not in a
lock table, which is why row-level locking scales. `FOR UPDATE` takes the strongest
row-level lock; a concurrent `FOR UPDATE` on the same row would normally **block**
until the first transaction ends.

`SKIP LOCKED` changes the scan: when the executor encounters a tuple that is locked by
another transaction, it does not wait and does not return it. It moves on. So:

- N relay workers running the same query get **disjoint** row sets.
- No worker ever waits for another.
- No lock table growth, no deadlock risk between workers.

Two properties you must understand before you rely on it:

1. **The locks are held until the transaction ends.** Commit or rollback, not "when you
   are done with the row". So the transaction's lifetime is the claim's lifetime, and
   a long transaction is a long claim on a Hikari connection.
2. **`SKIP LOCKED` breaks total ordering.** Worker A takes rows 1–100, worker B takes
   101–200, and B may publish first. If two events for the *same aggregate* land in
   different batches, they can reach Kafka out of order. This is the ordering trap, and
   the fix is below.

`ORDER BY id ... FOR UPDATE SKIP LOCKED LIMIT n` also has a subtlety worth knowing: the
ordering is applied to the scan, but rows skipped because they were locked simply do not
appear, so what you get is "the first n unlocked rows in id order", which is what you
want.

Verify the behaviour yourself in two `psql` sessions:

```sql
-- session 1
BEGIN;
SELECT id FROM outbox WHERE status='PENDING' ORDER BY id FOR UPDATE SKIP LOCKED LIMIT 5;
-- do not commit yet

-- session 2
BEGIN;
SELECT id FROM outbox WHERE status='PENDING' ORDER BY id FOR UPDATE SKIP LOCKED LIMIT 5;
```

| What you see | What it means |
|---|---|
| Session 2 returns a **different** five ids, immediately | `SKIP LOCKED` is working. This is the whole concurrency mechanism, demonstrated in two commands. |
| Session 2 blocks | You wrote `FOR UPDATE` without `SKIP LOCKED`. Two relay workers will serialise, and your throughput is one worker's. |
| Session 2 returns the **same** five ids | You are in the same session, or session 1 has already committed. Check with `SELECT pg_backend_pid();` in both. |
| Session 2 returns nothing | Fewer than ten pending rows exist. Seed more. |
| `ERROR: FOR UPDATE is not allowed with aggregate functions` or similar | You added a `GROUP BY`/`DISTINCT`. Row locking needs a plain row scan. |

### The sequence-gap fact that breaks cursor relays

This one is pure Postgres internals and it is the most valuable thing in this section.

`bigserial` is backed by a sequence. `nextval()` is **non-transactional** — it hands out
a value immediately, does not lock, and does not roll back. So values are allocated in
call order, but transactions **commit in a different order**.

Timeline:

```
T1: nextval -> 1001      (long transaction: inventory contention, a retry, a slow flush)
T2: nextval -> 1002      (fast transaction)
T2: COMMIT               <- row 1002 becomes visible
                          relay polls, sees 1002, sets cursor = 1002
T1: COMMIT               <- row 1001 becomes visible NOW
                          relay polls WHERE id > 1002 -> never sees 1001
```

**Row 1001 is lost forever.** No error. The relay is healthy, the outbox has a
`PENDING` row that nothing will ever select, and the event is never published — which
is the exact failure the outbox was built to prevent, now hiding inside the fix.

The same reasoning kills `WHERE created_at > last_seen_timestamp`, because `now()` in
Postgres is the **transaction start time**, so a long transaction stamps an *earlier*
timestamp and becomes visible *later*. Same bug, different column.

The correct designs are: a `status` column the relay updates (what we use here), or
logical decoding, which reads the WAL in **commit order** and therefore has this problem
by construction solved.

Prove it to yourself:

```sql
-- session 1
BEGIN;
INSERT INTO outbox(aggregate_type, aggregate_id, event_type, topic, payload)
VALUES ('order','ORD-A','OrderPlaced','orderflow.order-placed','{}'::jsonb)
RETURNING id;                       -- note the id, do NOT commit

-- session 2
BEGIN;
INSERT INTO outbox(...) VALUES ('order','ORD-B',...) RETURNING id;
COMMIT;                             -- higher id, commits first

-- session 3
SELECT id, aggregate_id FROM outbox ORDER BY id;   -- ORD-A is absent
-- session 1
COMMIT;
-- session 3
SELECT id, aggregate_id FROM outbox ORDER BY id;   -- ORD-A appears, with the LOWER id
```

| What you see | What it means |
|---|---|
| The lower id appears in the table only after the second commit | **The gap is real.** Any relay with a monotonic id cursor has already skipped it. |
| Both ids appear immediately | Autocommit is on in your client. `BEGIN` explicitly. |
| The ids are not adjacent | Sequence caching, or other traffic. Irrelevant to the point. |

### Unique-constraint and index cost of the outbox row

The outbox insert adds, per order: one heap tuple, one index entry in the primary key,
one index entry in the partial pending index, and WAL for all of it. The `jsonb` payload
is the dominant size; if it exceeds roughly 2KB after compression it goes to TOAST,
which adds an out-of-line write and read. Keep payloads small — an event should be a
fact, not a document. Reference the aggregate by ID and let the consumer read what it
needs, unless you deliberately want a self-contained event for schema-decoupling
reasons (a real trade; state which side you picked and why).

### Vacuum and bloat — the operational failure nobody plans for

Two relay designs, two problems:

- **`UPDATE ... SET status='SENT'`.** In Postgres an UPDATE writes a **new tuple** and
  marks the old one dead. So every outbox row is written twice and leaves one dead
  tuple. At the baseline's placement rate that is a steady stream of dead tuples in one
  table, and autovacuum must keep up or the table and its indexes bloat.
- **`DELETE` after publish.** Same dead-tuple production, but the live set stays tiny,
  which keeps the indexes small.

Watch it:

```sql
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum, autovacuum_count
FROM pg_stat_user_tables WHERE relname = 'outbox';
```

| What you see | What it means |
|---|---|
| `n_dead_tup` climbing, `last_autovacuum` old | Autovacuum is not keeping up on this table. Set per-table `autovacuum_vacuum_scale_factor` low (a hot small table wants a scale factor near 0 and a small threshold) rather than raising global settings. |
| `n_live_tup` growing without bound | You are not deleting or archiving `SENT` rows. The partial index stays small but the heap and primary key do not. |
| `n_dead_tup` low and `autovacuum_count` rising steadily | Healthy. This is what a well-tuned outbox looks like. |

The design that avoids the problem entirely is **partitioning by day** and dropping old
partitions — `DROP TABLE` on a partition reclaims space instantly with no vacuum. For a
high-volume outbox this is the right answer and it is worth the schema complexity.

### CDC: logical decoding, and the replication slot that can kill your database

The alternative relay. `wal_level=logical`, a publication, a replication slot, and
Debezium's Postgres connector reading it.

What you get: events in **commit order**, no polling load, latency bounded by WAL flush
and connector lag rather than by a poll interval, and the sequence-gap problem
structurally impossible.

What you take on:

```sql
SELECT slot_name, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;
```

| What you see | What it means |
|---|---|
| `active = t`, `retained` small and stable | Healthy. The connector is keeping up. |
| `active = f` and `retained` growing | **The most dangerous state in this document.** The slot is holding WAL that Postgres may not delete. Left alone, `pg_wal` fills the disk and the database stops accepting writes — your outbox took down the system it was protecting. |
| `retained` growing while `active = t` | The connector is connected but falling behind. Investigate the connector, not Postgres. |
| No rows | No logical replication is configured. If you thought Debezium was running, it is not. |

Mitigation: set `max_slot_wal_keep_size` so Postgres will invalidate a runaway slot
rather than fill the disk — accepting that an invalidated slot means the connector must
re-snapshot. That is a deliberate trade between "lose the stream" and "lose the
database", and you should make it consciously, before the incident.

Debezium also offers an **outbox event router** single-message transform that maps
outbox table rows onto topics and keys. Useful, and one more component to operate.
**Check its current configuration keys in the Debezium documentation; I am not asserting
them here.**

### The relay's transaction boundary — the real design decision

Two shapes. This choice determines whether your relay is safe under load.

**Shape A — publish inside the claiming transaction.**

```
BEGIN
  SELECT ... FOR UPDATE SKIP LOCKED LIMIT 100
  for each row: kafkaTemplate.send(...).get()      <-- NETWORK CALL, TX OPEN
  UPDATE ... SET status='SENT'
COMMIT
```

Correct in the sense that no row is marked sent unless the publish returned. But it
holds a Hikari connection across a network call for the whole batch — precisely the
Topic 55 bug that Topic 109's drill turns into a full outage. If Kafka is slow, the
relay's connections are held, and at pool size 20 with a few relay threads you are
consuming a meaningful share of the pool. Under a Kafka partition, the relay pins
connections until `delivery.timeout.ms`.

**Shape B — claim, commit, publish, mark. The one to pick.**

```
TX1: BEGIN
       SELECT ... FOR UPDATE SKIP LOCKED LIMIT 100
       UPDATE ... SET status='CLAIMED', claimed_by=<id>, claimed_at=now()
     COMMIT                                        <-- connection released
publish the batch to Kafka (no DB connection held)
TX2: BEGIN
       UPDATE ... SET status='SENT', sent_at=now() WHERE id = ANY(...)
     COMMIT
```

The connection is held only for two short database transactions. A crash between TX1
and TX2 leaves rows `CLAIMED` and unpublished-or-published-unknown, which a **reaper**
returns to `PENDING` after a claim timeout:

```sql
UPDATE outbox SET status='PENDING', claimed_by=NULL
WHERE status='CLAIMED' AND claimed_at < now() - interval '2 minutes';
```

That reaper is what makes the relay at-least-once rather than at-most-once. It is
twenty lines and it is the difference between a correct outbox and a broken one.

**The honest cost of Shape B:** a crash after publishing and before TX2 republishes the
whole batch. Duplicates. Which is fine, and expected, and why Topic 116 exists.

### Ordering, precisely

You get **per-key ordering in Kafka** if and only if:

1. All events for one aggregate go to the same partition — true if you use
   `aggregate_id` as the Kafka key and do not change the partition count.
2. The relay sends them in the right order.
3. The producer does not reorder on retry — true when `enable.idempotence=true`, which
   caps in-flight requests at five and has the broker reject out-of-order sequences
   (Topic 114).

Condition 2 is the one `SKIP LOCKED` threatens: two workers, two batches, no ordering
between them. Three fixes, in increasing cost:

- **Accept it.** Many event types do not need ordering. Say so explicitly in the
  contract table rather than by omission.
- **Shard the relay by aggregate.** Add `WHERE hashtext(aggregate_id) % :workers =
  :worker_index` to the claim query. Each aggregate is owned by exactly one worker, so
  its events are ordered. Costs you a static worker count and a rebalance story on scale
  events.
- **One relay worker.** Trivially ordered, and the throughput ceiling is one worker.
  Perfectly reasonable at `orderflow`'s placement volumes; measure before dismissing it.

Or use CDC, which reads in commit order and gives you global ordering for free. That is
the strongest argument for Debezium and it is worth more than the latency argument.

---

## Example 1 — minimal

The bug, in the smallest form that can actually lose data.

```java
package com.orderflow.lab.outbox;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class NaiveDualWrite {

    private final JdbcTemplate jdbc;
    private final KafkaTemplate<String, String> kafka;

    public NaiveDualWrite(JdbcTemplate jdbc, KafkaTemplate<String, String> kafka) {
        this.jdbc = jdbc;
        this.kafka = kafka;
    }

    /**
     * The transaction commits when this method returns through the proxy (Topic 40).
     * The send happens AFTER that. Everything between is the window.
     */
    @Transactional
    public void placeOrder(String orderId) {
        jdbc.update("insert into orders(id, status) values (?, 'PLACED')", orderId);
    }

    public void placeAndPublish(String orderId) {
        placeOrder(orderId);                                       // commits here
        // <<<<<<<<<<<<<<<<  THE WINDOW  >>>>>>>>>>>>>>>>
        kafka.send("orderflow.lab.order-placed", orderId, orderId);
    }
}
```

And the version that looks like a fix and is not:

```java
@Service
public class LooksLikeAFix {

    private final JdbcTemplate jdbc;
    private final ApplicationEventPublisher events;

    @Transactional
    public void placeOrder(String orderId) {
        jdbc.update("insert into orders(id, status) values (?, 'PLACED')", orderId);
        events.publishEvent(new OrderPlaced(orderId));   // registers a synchronization
    }
}

@Component
class Publisher {
    private final KafkaTemplate<String, String> kafka;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(OrderPlaced e) {
        // Runs after commit() returns, on the same thread.
        // <<<<<<<<<<<<<<<<  THE SAME WINDOW  >>>>>>>>>>>>>>>>
        kafka.send("orderflow.lab.order-placed", e.orderId(), e.orderId());
    }
}
```

Now the outbox:

```java
@Service
public class OutboxWrite {

    private final JdbcTemplate jdbc;

    @Transactional
    public void placeOrder(String orderId) {
        jdbc.update("insert into orders(id, status) values (?, 'PLACED')", orderId);
        jdbc.update("""
            insert into outbox(aggregate_type, aggregate_id, event_type, topic, payload)
            values ('order', ?, 'OrderPlaced', 'orderflow.lab.order-placed', ?::jsonb)
            """, orderId, "{\"orderId\":\"" + orderId + "\"}");
        // ONE commit. No window.
    }
}
```

**WHAT TO LOOK FOR** when you run all three under the drill below: whether `orders` and
the topic ever disagree.

| What you see | What it means |
|---|---|
| `orders` has a row, the topic has nothing, for the same id | The dual-write window fired. This is the failure. |
| `orders` has no row and the topic has the event | You published before committing. Downstream consumers are now acting on a fact that is false — worse than loss. |
| `orders` has a row and `outbox` has a `PENDING` row for it, after any kill | The outbox held. The event is not lost; it is queued and the relay will publish it. |
| `orders` has a row and no `outbox` row | Your outbox insert is not in the same transaction. Look for a second `DataSource`, a `REQUIRES_NEW`, or a self-invocation (Topic 40) that made the whole method non-transactional. |

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` at the Topic 65 baseline:

- 100k products, 1M orders, 5M order lines, containerised, k6 open-model load.
- 10% of the mix is `POST /orders`; placement is transactional across four writes.
- HikariCP pool sized from Topic 109's work — assume 20, and the relay shares it unless
  you give it its own.
- `orderflow.order-placed` has 12 partitions, keyed by `orderId`.
- Two consumer groups (Topic 113), both at-least-once with idempotent sinks (Topic 116).
- Rolling deploys of three pods are routine. That is the failure injector you already
  have in production.

### Schema

```sql
create table outbox (
  id             bigserial     primary key,
  aggregate_type text          not null,
  aggregate_id   text          not null,
  event_type     text          not null,
  topic          text          not null,
  payload        jsonb         not null,
  headers        jsonb         not null default '{}'::jsonb,
  status         text          not null default 'PENDING',
  claimed_by     text,
  claimed_at     timestamptz,
  created_at     timestamptz   not null default now(),
  sent_at        timestamptz,
  attempts       int           not null default 0,
  last_error     text,
  constraint outbox_status_ck check (status in ('PENDING','CLAIMED','SENT'))
);

create index outbox_pending_idx on outbox (id) where status = 'PENDING';
create index outbox_claimed_idx on outbox (claimed_at) where status = 'CLAIMED';

-- Per-table autovacuum tuning: this is a hot, small, high-churn table.
alter table outbox set (autovacuum_vacuum_scale_factor = 0.02,
                        autovacuum_vacuum_threshold = 500,
                        autovacuum_analyze_scale_factor = 0.05);
```

### The write path

```java
package com.orderflow.orders;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private final OrderRepository orders;
    private final InventoryService inventory;
    private final WalletService wallet;
    private final PaymentService payments;
    private final OutboxAppender outbox;

    public OrderService(OrderRepository orders, InventoryService inventory,
                        WalletService wallet, PaymentService payments,
                        OutboxAppender outbox) {
        this.orders = orders;
        this.inventory = inventory;
        this.wallet = wallet;
        this.payments = payments;
        this.outbox = outbox;
    }

    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {
        inventory.reserve(cmd.sku(), cmd.units());
        wallet.debit(cmd.customerId(), cmd.totalMinorUnits());
        var paymentId = payments.authorize(cmd);
        var order = orders.insert(cmd, paymentId);

        outbox.append(new OutboxEvent(
                "order",
                order.id().value(),
                "OrderPlaced",
                "orderflow.order-placed",
                OrderPlacedPayload.from(order),
                Map.of("traceparent", currentTraceparent())));   // Topic 119

        return order.id();
    }
}
```

```java
package com.orderflow.outbox;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

/**
 * Deliberately NOT annotated @Transactional. It must join the CALLER's transaction.
 * If this class ever gains REQUIRES_NEW, atomicity is gone and the outbox is a lie.
 */
@Component
public class OutboxAppender {

    private final JdbcTemplate jdbc;      // the SAME DataSource as the order write
    private final ObjectMapper mapper;

    public OutboxAppender(JdbcTemplate jdbc, ObjectMapper mapper) {
        this.jdbc = jdbc;
        this.mapper = mapper;
    }

    public void append(OutboxEvent e) {
        jdbc.update("""
            insert into outbox(aggregate_type, aggregate_id, event_type,
                               topic, payload, headers)
            values (?, ?, ?, ?, ?::jsonb, ?::jsonb)
            """,
            e.aggregateType(), e.aggregateId(), e.eventType(), e.topic(),
            writeJson(e.payload()), writeJson(e.headers()));
    }
}
```

> **Write a test that fails if this class becomes transactional.** Assert inside
> `append` that `TransactionSynchronizationManager.isActualTransactionActive()` is true
> and that `getCurrentTransactionName()` ends with `OrderService.place`. That single
> assertion catches the highest-consequence refactor in the whole pattern.

### The relay

```java
package com.orderflow.outbox;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.support.TransactionTemplate;

@Component
public class OutboxRelay {

    private static final int BATCH = 200;

    private final JdbcTemplate jdbc;
    private final TransactionTemplate tx;          // programmatic: boundaries are explicit
    private final KafkaTemplate<String, byte[]> kafka;
    private final MeterRegistry meters;            // Topic 118
    private final String workerId;                 // stable per pod: HOSTNAME

    // ... constructor omitted for brevity ...

    @Scheduled(fixedDelayString = "${orderflow.outbox.poll-ms:200}")
    public void drain() {
        List<OutboxRow> batch = claim();
        if (batch.isEmpty()) return;

        List<Long> published = new ArrayList<>(batch.size());
        for (OutboxRow row : batch) {
            try {
                var record = new ProducerRecord<>(
                        row.topic(), null, row.aggregateId(), row.payload());
                row.headers().forEach((k, v) ->
                        record.headers().add(k, v.getBytes(UTF_8)));   // trace context
                kafka.send(record).get(10, TimeUnit.SECONDS);          // ack before marking
                published.add(row.id());
            } catch (Exception e) {
                markFailed(row.id(), e);
                break;   // stop the batch: preserve per-worker ordering on failure
            }
        }
        markSent(published);
        meters.counter("orderflow.outbox.published").increment(published.size());
    }

    /** TX1: claim. Short. No network call inside. */
    private List<OutboxRow> claim() {
        return tx.execute(status -> {
            List<OutboxRow> rows = jdbc.query("""
                select id, topic, aggregate_id, payload, headers
                from outbox
                where status = 'PENDING'
                order by id
                for update skip locked
                limit ?
                """, OUTBOX_ROW_MAPPER, BATCH);
            if (rows.isEmpty()) return rows;
            jdbc.update("""
                update outbox
                set status = 'CLAIMED', claimed_by = ?, claimed_at = now()
                where id = any (?)
                """, workerId, idArray(rows));
            return rows;
        });
    }

    /** TX2: mark. Short. Runs after the Kafka acks. */
    private void markSent(List<Long> ids) {
        if (ids.isEmpty()) return;
        tx.executeWithoutResult(s -> jdbc.update("""
            update outbox set status = 'SENT', sent_at = now()
            where id = any (?)
            """, idArray(ids)));
    }

    /**
     * The reaper. THIS is what makes the relay at-least-once instead of at-most-once.
     * A pod killed between publish and markSent leaves CLAIMED rows; they come back.
     */
    @Scheduled(fixedDelayString = "${orderflow.outbox.reap-ms:30000}")
    public void reapStaleClaims() {
        int recovered = tx.execute(s -> jdbc.update("""
            update outbox
            set status = 'PENDING', claimed_by = null, claimed_at = null,
                attempts = attempts + 1
            where status = 'CLAIMED'
              and claimed_at < now() - (? || ' seconds')::interval
            """, claimTimeoutSeconds));
        if (recovered > 0) {
            meters.counter("orderflow.outbox.reclaimed").increment(recovered);
        }
    }

    /** Retention. Without this the table grows forever. */
    @Scheduled(cron = "${orderflow.outbox.purge-cron:0 */10 * * * *}")
    public void purgeSent() {
        jdbc.update("""
            delete from outbox
            where status = 'SENT' and sent_at < now() - interval '7 days'
            """);
    }
}
```

Six decisions in that code a reviewer should challenge, with the answers:

1. **Why `@Scheduled` and not a dedicated thread?** Because `@Scheduled` on the default
   single-threaded scheduler serialises `drain`, `reapStaleClaims` and `purgeSent`
   against each other, which is a hidden coupling: a slow purge delays drains. Give the
   relay its own `ThreadPoolTaskScheduler` with a named thread, or run it as a separate
   deployment. Note also that `@Scheduled` is proxy-based (Topic 40) — a self-called
   `drain()` gets no scheduling and no transaction.
2. **Why `TransactionTemplate` rather than `@Transactional`?** Because the boundaries
   are the point. Two explicit short transactions with a network call between them
   cannot be expressed with one annotation, and someone would eventually annotate the
   whole method and reintroduce Shape A.
3. **Why `break` on the first failure instead of continuing?** To preserve per-worker
   ordering. If row 5 fails and row 6 succeeds, an aggregate's events can be published
   out of order. Stopping the batch keeps the invariant at the cost of head-of-line
   blocking — which is the right trade for an ordered stream and the wrong one for an
   unordered one. Decide per topic and write it down.
4. **Why `.get(10, SECONDS)` instead of fire-and-forget?** Because "sent" must mean
   "the broker acked it". A fire-and-forget send that fails asynchronously marks the row
   `SENT` and loses the event. If you want throughput, batch the futures and await them
   all before `markSent` — do not remove the await.
5. **Why does the relay need its own connection pool?** Because at pool size 20, relay
   transactions compete with request threads. Give it a small dedicated `DataSource`
   (four connections is plenty) so a relay stall cannot starve `POST /orders`. This is
   Topic 109 applied.
6. **Why is `claimTimeoutSeconds` a configuration value?** Because it must exceed the
   worst-case publish time for a batch, or the reaper will return rows that are still
   being published and you will duplicate on purpose. Set it to several multiples of
   `delivery.timeout.ms`, and state the relationship in a comment so nobody "optimises"
   it later.

### What changed operationally

- `POST /orders` no longer makes a Kafka call on the request thread. One fewer network
  dependency in the request path, and one fewer source of p99 variance. Measure this
  against the Topic 65 baseline — it is usually an improvement, and it is a nice thing
  to be able to show.
- Placement now writes one extra row per order. Measure the write amplification.
- A new failure mode exists: the relay stops and the outbox backs up. That is
  **observable** — `SELECT count(*) FROM outbox WHERE status='PENDING'` and the age of
  the oldest pending row — which is the entire improvement. You traded an invisible
  failure for a visible one.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — publishing after the commit, including via `@TransactionalEventListener(AFTER_COMMIT)`

**Wrong (both forms):**

```java
@Transactional
public OrderId place(PlaceOrderCommand cmd) { ... }

public OrderId placeAndPublish(PlaceOrderCommand cmd) {
    var id = place(cmd);
    kafka.send("orderflow.order-placed", id.value(), event);    // form A
    return id;
}
```

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onPlaced(OrderPlacedEvent e) {
    kafka.send("orderflow.order-placed", e.orderId(), e);       // form B: IDENTICAL window
}
```

**Exact symptom:** orders exist in Postgres with no corresponding event and no
downstream effect. Concretely, this query returns rows:

```sql
SELECT o.id, o.created_at
FROM orders o
LEFT JOIN inventory_reservations r ON r.order_id = o.id
WHERE o.created_at > now() - interval '7 days'
  AND r.order_id IS NULL
ORDER BY o.created_at;
```

The rows cluster around deployment timestamps and pod restarts. No exception was ever
logged, no error metric moved, no consumer lag appeared, and no DLQ entry exists.
Customer support finds it first, as "I was charged and my order never shipped".

**Root cause:** two operations, no atomicity. For form B specifically:
`ApplicationEventPublisher.publishEvent` inside a transaction registers a
`TransactionSynchronization`; `AFTER_COMMIT` invokes it from
`TransactionSynchronizationManager` **after** `AbstractPlatformTransactionManager`
finishes committing, on the same thread. Commit returns, then the listener body runs.
A `kill -9` between them loses the event exactly as form A does. The annotation moved
the call site; it did not add durability.

**Two further facts about form B that make it worse than form A:**

- `AFTER_COMMIT` listeners that throw do **not** fail the caller — the transaction is
  already committed. By default the exception is logged and swallowed. So a Kafka
  outage produces a log line and silent loss, at whatever your logging level happens to
  be.
- `TransactionPhase.AFTER_COMPLETION` and `AFTER_ROLLBACK` exist and are easy to select
  by accident, and `@TransactionalEventListener` on a method invoked with **no active
  transaction** does nothing at all by default (`fallbackExecution=false`). A unit test
  without a transaction therefore passes while proving nothing.

**Fix:** the outbox. The event becomes a row in the same transaction. If you want to
keep `@TransactionalEventListener` as an internal call-site convenience, move it to
`TransactionPhase.BEFORE_COMMIT` and have the listener write the **outbox row** — then
it is inside the transaction and it is genuinely atomic. That is a legitimate and
elegant use of the annotation, and it is the opposite of how it is usually used.

```java
@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
public void onPlaced(OrderPlacedEvent e) {
    outbox.append(...);       // still inside the transaction. Atomic. Correct.
}
```

---

### Trap 2 — publishing before the commit

**Wrong:**

```java
@Transactional
public OrderId place(PlaceOrderCommand cmd) {
    var order = orders.insert(cmd);
    kafka.send("orderflow.order-placed", order.id().value(), event);  // inside the tx
    wallet.debit(cmd.customerId(), cmd.totalMinorUnits());   // may throw -> ROLLBACK
    return order.id();
}
```

Teams arrive here after being told "publish after commit is a bug", and reverse it.

**Exact symptom:** consumers act on orders that do not exist. The inventory projector
decrements stock for order `ORD-88213`; `SELECT * FROM orders WHERE id='ORD-88213'`
returns nothing. Downstream services log `OrderNotFoundException` when they try to
enrich the event, and — if they are written defensively — retry forever against a row
that will never appear, which turns into a poison-pill loop and a rising DLQ.

Phantom stock-outs are the business-visible form: inventory shows zero for a product
that has stock, because reservations exist for orders that rolled back.

**Root cause:** the send left the process. The rollback rolled back the database. There
is no mechanism that recalls a Kafka record. This is Topic 114's boundary again: the
Postgres transaction manager has no authority over Kafka, and Kafka's coordinator has
no authority over Postgres.

**Second-order cause worth naming:** it also puts a network call inside a transaction,
which pins a Hikari connection for the duration of the Kafka send. Under a broker blip
this is Topic 55's outage shape: every endpoint fails, including ones that touch no
Kafka.

**Fix:** the outbox. And note the asymmetry: publish-after-commit loses events;
publish-before-commit **manufactures false events**. The second is strictly worse,
because a lost event can sometimes be reconstructed from the database, while a false
event has already caused effects downstream that must be individually compensated
(Topic 117).

---

### Trap 3 — a relay that assumes at-most-once

**Wrong (three variants, all common):**

```java
// Variant A: delete first, then publish
var rows = jdbc.query("delete from outbox where id in (...) returning *", MAPPER);
rows.forEach(r -> kafka.send(r.topic(), r.key(), r.payload()));

// Variant B: mark sent first, then publish
jdbc.update("update outbox set status='SENT' where id = any(?)", ids);
rows.forEach(r -> kafka.send(...));

// Variant C: fire-and-forget send, then mark sent
rows.forEach(r -> kafka.send(...));              // no .get(), no callback check
jdbc.update("update outbox set status='SENT' where id = any(?)", ids);
```

**Exact symptom:** events vanish, and only under specific conditions — a relay pod
restart (A and B), or a Kafka broker unavailability window (C). The outbox table looks
perfectly healthy afterwards: zero `PENDING` rows, everything `SENT`. The reconciliation
query from Trap 1 still returns rows. This is the cruellest variant, because the outbox
was supposed to make loss impossible and the table itself testifies that everything was
sent.

Variant C is the worst because it fails **without a restart**: `KafkaTemplate.send`
returns a future immediately, the record sits in the producer's accumulator, the
delivery eventually fails after `delivery.timeout.ms`, and by then the row says `SENT`.
The only trace is a producer error log that nobody is alerting on.

**Root cause:** the relay marked completion before completion was durable. This is
exactly Trap 3 of Topic 114 — committing the offset before the work — in the producer's
clothing. There is no ordering of two non-atomic operations that gives you
exactly-once; you get to choose which side of the window you fail on. Marking first
chooses **loss**. Publishing first chooses **duplicates**.

**Fix:** publish first, await the ack, then mark. And add the reaper, because now a
crash between the ack and the mark leaves `CLAIMED` rows that must return to `PENDING`.

```java
kafka.send(record).get(10, TimeUnit.SECONDS);   // throws if not acked
published.add(row.id());
// ... then, in a separate short transaction:
markSent(published);
```

**Say the trade out loud in the design doc:** "The relay is at-least-once. Consumers
must be idempotent. See Topic 116." If that sentence is not written down, someone will
eventually build a non-idempotent consumer and be surprised.

---

### Trap 4 — a cursor relay: `WHERE id > last_seen_id`

**Wrong:**

```java
private final AtomicLong cursor = new AtomicLong(0);

@Scheduled(fixedDelay = 200)
public void drain() {
    var rows = jdbc.query(
        "select * from outbox where id > ? order by id limit 200",
        MAPPER, cursor.get());
    for (var r : rows) {
        kafka.send(r.topic(), r.key(), r.payload()).get();
        cursor.set(r.id());
    }
}
```

Attractive because it needs no `status` column, no UPDATE, no vacuum churn, and it is
obviously "efficient".

**Exact symptom:** a small percentage of events are never published — and the
percentage rises exactly when the system is under load, because that is when
transactions take longer and the interleaving becomes likely. The outbox row is still
in the table forever, `status` untouched, and nothing will ever select it because its id
is below the cursor. The symptom is the Trap 1 reconciliation query returning rows, but
now it is *not* correlated with restarts, which makes it much harder to diagnose.

Detection query — the one that reveals it:

```sql
SELECT id, aggregate_id, created_at FROM outbox
WHERE status = 'PENDING' AND created_at < now() - interval '10 minutes'
ORDER BY id;
```

Old `PENDING` rows with much newer rows already sent is the signature.

**Root cause:** Postgres sequences are non-transactional. `nextval()` allocates in call
order; transactions commit in a different order. A long-running transaction can allocate
id 1001, and a short one can allocate 1002 and commit first. The relay sees 1002, sets
the cursor to 1002, and row 1001 becomes visible afterwards, forever below the cursor.

The `created_at` variant is broken for the same reason with a different mechanism:
`now()` returns the **transaction start** timestamp, so a long transaction stamps an
earlier time and becomes visible later.

**Fix:** a `status` column the relay updates (a transactional fact, not a
non-transactional counter), or logical decoding, which reads the WAL in commit order and
cannot have this problem.

If you must keep a cursor for some reason, the only safe form is a cursor with a lag
window — `WHERE id > cursor - K` combined with a dedup table — which is strictly more
complex than the `status` column and buys nothing. Do not.

> **This is the trap to remember from this document if you remember only one.** It is
> not a Java mistake or a Spring mistake. It is a database-internals mistake, it is
> invisible in review, it produces no error, and the code looks better than the correct
> version.

---

### Trap 5 — holding the transaction across the Kafka publish

**Wrong:**

```java
@Transactional
@Scheduled(fixedDelay = 200)
public void drain() {
    var rows = jdbc.query("""
        select * from outbox where status='PENDING'
        order by id for update skip locked limit 500
        """, MAPPER);
    for (var r : rows) {
        kafka.send(r.topic(), r.key(), r.payload()).get();   // network call, TX OPEN
    }
    jdbc.update("update outbox set status='SENT' where id = any(?)", ids(rows));
}
```

**Exact symptom:** under a Kafka slowdown or partition, `POST /orders` and every other
endpoint starts failing with
`SQLTransientConnectionException: HikariPool-1 - Connection is not available, request
timed out after 30000ms`. Endpoints that touch no Kafka fail too. `GET /products`,
served from a cache, fails. It looks like a database outage; Postgres is completely
healthy.

Confirm with Topic 109's instruments: `hikaricp.connections.pending` climbing,
`hikaricp.connections.active` pinned at maximum, and a thread dump
(`jcmd <pid> Thread.print`) showing relay threads parked inside a Kafka send while
holding a connection, and request threads parked in `HikariPool.getConnection`.

**Root cause:** a transaction holds its connection for its entire lifetime (Topic 55).
Putting a network call inside converts the downstream's latency into connection-pool
occupancy. A batch of 500 with a slow broker holds one connection for the whole batch;
several relay threads doing it consume a meaningful fraction of a pool of 20.

**Fix, in three parts:**

1. **Split the transaction** (Shape B above): claim and commit, publish outside, mark in
   a second transaction.
2. **Give the relay its own small `DataSource`.** Even with Shape B, isolate it so a
   relay pathology cannot starve request threads.
3. **Bound the batch.** 500 rows times a 10-second publish timeout is an 83-minute
   worst case for one batch. 100–200 with a per-send timeout is a bounded claim.

**Do not fix it by raising the pool size.** Topic 109's argument applies: a bigger pool
moves the contention into Postgres, where a process per connection makes it worse. The
problem is the hold time, not the pool.

---

### Trap 6 — no retention, no vacuum tuning, no backlog alert

**Wrong:** the relay works. Nobody deletes `SENT` rows. Nobody tunes autovacuum on the
table. Nobody alerts on the backlog.

**Exact symptom, over weeks:** `POST /orders` p99 degrades slowly against the Topic 65
baseline with no code change. `pg_stat_user_tables` shows `outbox` with a large
`n_live_tup` and a large `n_dead_tup`. The primary key index no longer fits in shared
buffers, so each outbox insert costs additional random I/O. The relay's claim query,
which used to be an index scan over a tiny partial index, now spends its time on a bloated
heap.

**Exact symptom, in an incident:** the relay stops (a bad deploy, a Kafka outage, a
config error). Nothing alerts, because the service is up, the endpoints are green, and
consumer lag is *zero* — there is nothing to consume. Four hours later someone notices
inventory has not moved. The backlog is enormous and draining it saturates Kafka.

**Root cause:** an outbox is a queue, and every queue needs a depth metric, an age
metric, and a retention policy. Treating it as a table instead of as a queue is the
error.

**Fix:**

```sql
-- Retention
DELETE FROM outbox WHERE status='SENT' AND sent_at < now() - interval '7 days';

-- Better at volume: partition by day and drop partitions.
-- DROP TABLE on a partition reclaims space with no vacuum at all.

-- Autovacuum tuning for a hot, small, high-churn table
ALTER TABLE outbox SET (autovacuum_vacuum_scale_factor = 0.02,
                        autovacuum_vacuum_threshold = 500);
```

And the two alerts that matter (Topic 118):

```java
Gauge.builder("orderflow.outbox.pending", this,
              r -> jdbc.queryForObject(
                  "select count(*) from outbox where status='PENDING'", Long.class))
     .register(registry);

Gauge.builder("orderflow.outbox.oldest.pending.seconds", this,
              r -> jdbc.queryForObject(
                  "select coalesce(extract(epoch from now() - min(created_at)), 0) "
                + "from outbox where status='PENDING'", Double.class))
     .register(registry);
```

**Alert on the age, not the count.** A depth of ten thousand during a traffic spike is
fine if it is draining. A single row that is five minutes old means the relay is dead.
Age catches the failure that count does not.

Note the Micrometer detail: `Gauge.builder(name, obj, fn)` holds a **weak** reference to
`obj`. Register gauges against a long-lived bean, not a local object, or the gauge
reports nothing once it is collected. Topic 118 covers this trap properly.

---

## Hands-on proof

Every command is one **you** run against the Topic 65 stack. No output is reproduced
here as captured.

### Setup

```bash
cd ~/orderflow
docker compose up -d postgres kafka
psql -h localhost -U orderflow -d orderflow -f db/outbox.sql

docker compose exec kafka kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orderflow.lab.order-placed --partitions 3 --replication-factor 1 \
  --if-not-exists

alias kcli='docker compose exec -T kafka'
alias psq='psql -h localhost -U orderflow -d orderflow -qtAX -c'
```

### Proof 1 — `SKIP LOCKED` hands out disjoint batches

Two `psql` sessions, side by side. Use the table from **Machine-level reality**.

The extra step worth doing: while session 1 holds its lock, run

```sql
SELECT pid, locktype, mode, granted, relation::regclass
FROM pg_locks WHERE relation = 'outbox'::regclass;
```

| What you see | What it means |
|---|---|
| A `RowExclusiveLock`/tuple lock held by session 1's pid, `granted = t` | The claim is real and is a database lock, not application state. |
| Session 2's pid absent from the list while it runs its query | It never requested the locked rows. `SKIP LOCKED` skipped them in the scan; there is nothing to wait for. |
| Session 2's pid present with `granted = f` | You omitted `SKIP LOCKED`. It is queueing. Your relay workers will serialise. |

### Proof 2 — the sequence gap is real

Run the three-session script from **Machine-level reality**. Then, to make it concrete
for the relay:

```sql
-- while session 1's INSERT is uncommitted:
SELECT max(id) FROM outbox;             -- the cursor a relay would record
-- after session 1 commits:
SELECT id, aggregate_id FROM outbox WHERE id < (that max) AND status = 'PENDING';
```

| What you see | What it means |
|---|---|
| The second query returns a row | **That row is what a cursor relay loses.** You have just produced the invisible bug with three commands. |
| It returns nothing | Your two inserts did not interleave. Add `SELECT pg_sleep(5);` inside session 1's transaction before committing. |

### Proof 3 — the relay's SQL plan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, topic, aggregate_id, payload
FROM outbox WHERE status = 'PENDING'
ORDER BY id FOR UPDATE SKIP LOCKED LIMIT 200;
```

**WHAT TO LOOK FOR:** the scan node and the buffer counts.

| What you see | What it means |
|---|---|
| `Index Scan using outbox_pending_idx`, small `Buffers: shared hit` | Correct. The partial index is doing its job and the claim is cheap. |
| `Seq Scan on outbox` | The partial index is not being used — usually because your predicate does not match the index's `WHERE` clause exactly, or the planner thinks nearly everything is pending. Check with `\d+ outbox`. |
| Large `shared read` counts | The index or heap is not cached. Usually bloat (Trap 6) or a table that has grown because nothing purges `SENT` rows. |
| `LockRows` node present | Expected — that is `FOR UPDATE`. |

### Proof 4 — watch the outbox drain in real time

```bash
watch -n 1 "psql -h localhost -U orderflow -d orderflow -qtAX -c \
  \"select status, count(*), max(now() - created_at) from outbox group by 1 order by 1\""
```

In another terminal, run the Topic 65 load generator at a low rate.

| What you see | What it means |
|---|---|
| `PENDING` oscillating near zero, max age under a second or two | Healthy. The relay is keeping up at this rate. |
| `PENDING` growing monotonically, max age climbing | The relay cannot keep up, or it is dead. Check the relay's scheduler thread with `jcmd <pid> Thread.print \| grep -A5 outbox`. |
| `CLAIMED` rows with a growing age | A relay claimed rows and did not finish. If the reaper is working they return to `PENDING`; if they sit there, your reaper is not running. |
| `SENT` growing without bound | No retention policy. Trap 6. |

### Proof 5 — confirm the outbox insert is in the caller's transaction

```java
public void append(OutboxEvent e) {
    if (!TransactionSynchronizationManager.isActualTransactionActive()) {
        throw new IllegalStateException(
            "OutboxAppender called with no active transaction - the outbox guarantee is void");
    }
    // ...
}
```

Plus the assertion in a test:

```java
@Test
void outbox_row_and_order_row_share_one_transaction() {
    assertThatThrownBy(() -> orderService.place(commandThatFailsAtWalletDebit()))
        .isInstanceOf(InsufficientFundsException.class);

    assertThat(jdbc.queryForObject("select count(*) from orders where id=?",
                                   Integer.class, id)).isZero();
    assertThat(jdbc.queryForObject("select count(*) from outbox where aggregate_id=?",
                                   Integer.class, id)).isZero();
}
```

| What you see | What it means |
|---|---|
| Both counts zero | Atomic. The rollback took both. This is the property the whole pattern rests on, asserted. |
| Order zero, outbox one | The outbox insert is in a different transaction. Look for `REQUIRES_NEW`, a second `DataSource`, or a `@Transactional` on `OutboxAppender`. |
| Order one, outbox zero | The order insert is not covered by the same transaction, or `place` is being self-invoked (Topic 40) and is not transactional at all. |
| The `IllegalStateException` fires in production | Someone called the appender from outside a transaction. That is the guard doing its job — do not delete it. |

---

## Failure drill

**Mandatory.** Produce the failure yourself before reading the fix.

### The scenario

`kill -9` the service in the window between the Postgres commit and the Kafka publish.
Show the order exists with no event. Then implement the outbox, repeat the identical
kill, and show the event is eventually published.

### Part 1 — reproduce the loss

Instrument the window so you can hit it reliably. This sleep is the only artificial
thing in the drill; it widens a window that is genuinely there.

```java
package com.orderflow.lab.outbox;

@Service
public class DrillOrderService {

    private final JdbcTemplate jdbc;
    private final KafkaTemplate<String, String> kafka;

    @Transactional
    public void commitOnly(String orderId) {
        jdbc.update("insert into orders(id, status, created_at) values (?, 'PLACED', now())",
                    orderId);
        System.out.println("ABOUT TO COMMIT " + orderId);
    }

    public void placeAndPublish(String orderId) throws InterruptedException {
        commitOnly(orderId);                       // through the proxy: commits here
        System.out.println("COMMITTED, IN THE WINDOW " + orderId);
        Thread.sleep(5000);                        // <-- the window, widened
        kafka.send("orderflow.lab.order-placed", orderId, orderId).get();
        System.out.println("PUBLISHED " + orderId);
    }
}
```

```java
@Component
public class DrillRunner implements CommandLineRunner {
    private final DrillOrderService svc;
    public void run(String... args) throws Exception {
        for (int i = 1; i <= 5; i++) svc.placeAndPublish("DRILL-" + i);
    }
}
```

```bash
./mvnw spring-boot:run &
SERVICE_PID=$!

# Wait until you see "COMMITTED, IN THE WINDOW DRILL-3" in the logs, then:
kill -9 $SERVICE_PID

psq "select id, status from orders where id like 'DRILL-%' order by id"
kcli kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orderflow.lab.order-placed --from-beginning --timeout-ms 5000 \
  --property print.key=true
```

Then, to prove nothing recovers it:

```bash
./mvnw spring-boot:run &      # restart; it will place DRILL-1..5 again, not recover DRILL-3
sleep 30
psq "select id from orders where id like 'DRILL-%' order by id"
kcli kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orderflow.lab.order-placed --from-beginning --timeout-ms 5000 \
  --property print.key=true | sort -u
```

### What to capture — Part 1

1. The `orders` rows after the kill.
2. The topic contents after the kill.
3. The exact id that is in one and not the other.
4. Whether a restart recovers it.
5. Whether **any** log line, metric, or error indicated the loss.

| What you see | What it means |
|---|---|
| `orders` contains `DRILL-3`; the topic does not | **The drill has fired.** The order exists and the event does not. This is the dual-write window with a name and a row number. |
| The restart does not produce `DRILL-3` on the topic | Confirmed permanent. Nothing in the system knows the event was owed. There is no retry, because the code that would have retried died. |
| No error, no exception, no metric moved | The point of the drill. This failure produces **no signal at all**. Write that down. |
| Both contain `DRILL-3` | Your kill landed after the send. Increase the sleep, or kill on the exact log line rather than on a timer. |
| Neither contains `DRILL-3` | The kill landed before the commit — which is the *safe* failure. Try again; you need the kill inside the window. |

### Part 2 — repeat with `@TransactionalEventListener(AFTER_COMMIT)`

Rewrite using the listener form from Trap 1 and run the identical kill.

| What you see | What it means |
|---|---|
| Identical result: order present, event absent | **The annotation is not a fix.** You have now proven the most commonly believed false thing in Spring event handling, on your own machine. |
| The event *is* published | Your kill missed the window, which is now much narrower without the artificial sleep. Add `Thread.sleep(5000)` at the top of the listener body and repeat. The narrowness of the window is not the same as its absence. |

### Part 3 — implement the outbox and repeat the identical kill

```java
@Transactional
public void placeWithOutbox(String orderId) {
    jdbc.update("insert into orders(id, status, created_at) values (?, 'PLACED', now())",
                orderId);
    jdbc.update("""
        insert into outbox(aggregate_type, aggregate_id, event_type, topic, payload)
        values ('order', ?, 'OrderPlaced', 'orderflow.lab.order-placed', ?::jsonb)
        """, orderId, "{\"orderId\":\"" + orderId + "\"}");
}
```

Run with the relay enabled, kill at the same point (during the relay's sleep, or by
killing the relay pod specifically), then:

```bash
psq "select id from orders where id like 'DRILL-%' order by id"
psq "select id, aggregate_id, status, created_at from outbox order by id"
# restart, wait for the relay to run
sleep 15
kcli kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orderflow.lab.order-placed --from-beginning --timeout-ms 5000 \
  --property print.key=true | sort -u
```

| What you see | What it means |
|---|---|
| After the kill: order present, outbox row `PENDING`, no event | **The window is closed.** The event is not lost; it is durably queued. This is the entire point of the pattern, visible as a row. |
| After the restart: the event appears on the topic, outbox row `SENT` | The relay recovered it. No human intervention, no reconciliation script. |
| After the kill: outbox row stuck in `CLAIMED` and never published | Your reaper is missing or its timeout is too long. This is the at-most-once relay bug (Trap 3) hiding in your fix. |
| The event appears **twice** on the topic | Correct and expected: the relay published, crashed before marking, and the reaper returned the row. **This is at-least-once working.** Do not "fix" it by marking before publishing — go make the consumer idempotent (Topic 116). |
| No outbox row at all after the kill | The outbox insert was not in the same transaction. Proof 5's guard would have caught this. |

### Part 4 — the honest closing question

Write down the answer before moving on:

> After the outbox, what have you actually guaranteed, and what have you not?

The answer you should have arrived at: you have guaranteed that **an event exists for
every committed order**. You have not guaranteed that it is published exactly once, or
promptly, or in order with respect to other aggregates. You converted a silent permanent
loss into a visible, bounded, retryable duplicate. That is the trade, and it is the
right one — but only if the consumer is idempotent, which is a promise you have not yet
kept.

---

## Measurement

### The standing rule first

> **A naive `System.nanoTime()` measurement of the outbox's cost is wrong.** JIT
> warm-up, dead-code elimination and on-stack replacement apply to any in-JVM timing
> (Topic 77), and on top of them the outbox insert's real cost is dominated by WAL
> flush, index maintenance and page-cache state — none of which a microbenchmark
> reproduces. Measure the outbox's cost as an end-to-end delta against the Topic 65
> baseline with the k6 open-model generator, and measure the database side with
> `pg_stat_statements`. Never with a stopwatch around a method.

### The four things to measure

**1. Backlog depth and, more importantly, backlog age.**

```sql
SELECT count(*) FILTER (WHERE status='PENDING')   AS pending,
       count(*) FILTER (WHERE status='CLAIMED')   AS claimed,
       coalesce(extract(epoch FROM now() - min(created_at))
                FILTER (WHERE status='PENDING'), 0) AS oldest_pending_seconds
FROM outbox;
```

Expose both as Micrometer gauges (Topic 118) and **alert on age**. Depth alone is
ambiguous — a large draining backlog is fine, a single stale row is an outage. Age is
the SLI that maps to "how late is a downstream effect", which is what a customer
experiences.

Set the alert threshold from the downstream requirement, not from a round number. If
inventory must reflect an order within 30 seconds, alert at 10.

**2. Relay throughput and reclaim rate.**

```java
meters.counter("orderflow.outbox.published").increment(published.size());
meters.counter("orderflow.outbox.reclaimed").increment(recovered);
meters.counter("orderflow.outbox.failed").increment();
Timer.builder("orderflow.outbox.batch").register(meters);   // one batch, claim to mark
```

Tag by `topic`. **Never** tag by `aggregate_id` or `event_id` — that is Topic 118's
cardinality bomb, and an outbox relay is the single most tempting place to do it.

The reclaim counter is the interesting one: it is nonzero exactly when the relay
crashed mid-batch, so it is a direct measure of how often your at-least-once behaviour
is actually exercised. Correlate it with duplicate counts at the consumer (Topic 114's
`duplicates_absorbed`). Those two numbers should move together; if they do not, one of
the two instruments is lying.

**3. Write amplification and vacuum health.**

```sql
SELECT relname, seq_scan, idx_scan, n_tup_ins, n_tup_upd, n_tup_del,
       n_live_tup, n_dead_tup, last_autovacuum, autovacuum_count
FROM pg_stat_user_tables WHERE relname IN ('orders','outbox');

SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
WHERE query ILIKE '%outbox%' ORDER BY total_exec_time DESC LIMIT 10;
```

| What you see | What it means |
|---|---|
| `outbox.n_tup_ins` roughly equal to `orders.n_tup_ins` | One event per order. Expected. If it is a multiple, you are emitting more events than you think. |
| `n_tup_upd` roughly 2x `n_tup_ins` | Expected with the claim/mark design (PENDING to CLAIMED to SENT). If it is much higher, the reaper is churning — your claim timeout is too short. |
| The claim query dominating `pg_stat_statements` by `calls` | Your poll interval is too aggressive for the backlog. A 200ms poll on an empty outbox is a lot of pointless queries; back off adaptively when a batch comes back empty. |
| `n_dead_tup` high with an old `last_autovacuum` | Trap 6. Tune per-table autovacuum before it becomes a latency regression. |

**4. Endpoint latency against the Topic 65 baseline.**

```bash
k6 run --out json=results-before-outbox.json load/orderflow-mix.js
# deploy the outbox change
k6 run --out json=results-after-outbox.json load/orderflow-mix.js
```

Compare p50/p95/p99/p999 per endpoint against `/docs/java/baselines/`. The gate rule
applies: if the unchanged baseline no longer reproduces within ±10%, fix the
environment before drawing a conclusion.

**Two effects to look for, and they point in opposite directions:**

- `POST /orders` may get **faster**, because the Kafka send left the request path. If
  it does, that is a real win and worth reporting — it also means you previously had a
  network call on the hot path that nobody had counted.
- The database gets one more insert per placement, plus the relay's query load. Under
  the baseline that is usually small, but "usually" is not a measurement. Look at p999
  and at Postgres CPU, not at the mean.

### What "proof" looks like at the end of this topic

- The `kill -9` drill reproduced, before and after, with the artefacts saved.
- The backlog-age gauge on a dashboard with an alert wired to the downstream requirement.
- A test that fails if `OutboxAppender` ever gains its own transaction.
- A latency comparison against the committed baseline, both directions accounted for.
- One sentence in the design doc: "the relay is at-least-once; consumers must be
  idempotent."

---

## Practice exercises

### 1 — Easy: find every dual write in `orderflow`

Without writing code, grep the codebase for every place a database transaction and an
external effect happen in the same logical operation:

```bash
grep -rn "kafkaTemplate\|KafkaTemplate\|restClient\|RestClient\|WebClient\|@TransactionalEventListener" \
  src/main/java --include="*.java"
```

For each hit, produce a row: file, method, is it inside `@Transactional`, does the
external effect happen before or after the commit, and what specifically is lost or
falsified by a `kill -9` at the worst moment.

Then answer:

1. Which hits are genuine dual writes and which are safe (a read, or an effect whose
   loss is acceptable and documented)?
2. For each genuine one, is the outbox the right fix, or is the right fix to remove the
   external call from that path entirely?
3. Which of them would a code reviewer plausibly catch, and which are invisible? What
   does that tell you about where to spend review effort versus where to spend a test?

### 2 — Medium: the audit (combines Topics 40, 48, 54, 55, 90, 109, 114)

This relay has **seven** defects. Three are from this topic. Four are from earlier
topics. For each: name the topic, state the **observable** symptom in production, and
write the fix.

```java
package com.orderflow.outbox;

@Service
public final class BrokenRelay {

    @Autowired private JdbcTemplate jdbc;
    @Autowired private KafkaTemplate<String, String> kafka;

    private long cursor = 0;
    private final ExecutorService pool = Executors.newCachedThreadPool();

    @Scheduled(fixedDelay = 100)
    @Transactional
    public void drain() {
        var rows = jdbc.query(
            "select id, topic, aggregate_id, payload from outbox where id > ? order by id",
            ROW_MAPPER, cursor);

        for (var row : rows) {
            pool.submit(() -> {
                kafka.send(row.topic(), row.aggregateId(), row.payload());
                markSent(row.id());
            });
            cursor = row.id();
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    private void markSent(long id) {
        jdbc.update("update outbox set status='SENT' where id=?", id);
    }
}
```

Hints, in the order to think about them. One defect makes both `@Transactional`
annotations completely inert — find it first because it changes the analysis of two
others. One is the sequence-gap bug and is the highest-consequence defect here. One
makes the relay at-most-once. One is an unbounded batch with no `LIMIT`. One is Topic
90's unbounded thread pool. One is Topic 55/109's connection held across a network
call, made worse by Topic 109's nested-transaction deadlock shape. One is a data race
on a plain `long` field read and written from a scheduler thread (Topic 86/87).

For each, also say whether it fails **loudly** or **silently**, and rank them by how
long they would survive in production before anyone noticed.

### 3 — Hard: production simulation on `orderflow` under load

**Part A — measure the window.** Instrument `place()` to record the timestamp
immediately after the commit and immediately after the Kafka ack, and export the delta
as a Micrometer timer. Run the Topic 65 load for ten minutes. Record the p50, p99 and
max of that delta. That is the width of the window you are exposed to, measured rather
than assumed. Compute the expected number of lost events per rolling deploy from your
placement rate and the p99 width, and state your assumptions.

**Part B — reproduce under load, not in isolation.** With the load running, `kill -9`
one of three `orderflow` pods. Run the reconciliation query. Record how many orders
exist with no event. Repeat three times and report the spread, not the mean — the
variance is the interesting part, and one run proves nothing.

**Part C — implement the outbox properly.** Schema, appender, relay with claim/publish/
mark, reaper, retention, and the two gauges. Constraints you must satisfy:

- The relay must not be able to starve `POST /orders` of connections. Prove it by
  running the load with Kafka network-partitioned (`docker network disconnect`) and
  showing that `POST /orders` still succeeds.
- Per-order event ordering must hold. Design a test that would detect a violation, and
  say honestly whether your relay guarantees it or merely usually achieves it.
- The claim timeout must exceed the worst-case publish time. State the relationship
  between `claimTimeoutSeconds` and `delivery.timeout.ms` and enforce it at startup.

**Part D — repeat Part B.** Same kill, same load. Record: orders with no event (should
be zero), duplicate events on the topic (should be non-zero at least once), and the
reclaim counter. Explain the relationship between the last two.

**Part E — the cost.** Compare against the Topic 65 baseline: per-endpoint latency
percentiles, Postgres CPU, WAL generation rate (`pg_stat_wal` or
`pg_current_wal_lsn()` sampled over time), and outbox table plus index size after a
sustained run. Then answer: at what placement rate does the polling relay stop being
adequate, and what is the specific resource that saturates first? Show the evidence.

**Part F — argue for CDC.** Write the case for replacing the polling relay with
Debezium. Include: what it fixes that your relay does not (name the ordering property
precisely), what it costs operationally, the specific failure mode of a stalled
replication slot and how you would detect it before it fills the disk, and what would
have to be true about your team's on-call capacity for the trade to be worth it. Then
argue the other side and say which you would actually ship.

---

## Interview questions

### Q1 — "We publish the event after committing the transaction. Is that safe?"

**Mid-level answer:** "It's mostly fine — the transaction is committed so the data is
consistent. If the publish fails you can retry it, or catch the exception and log it so
someone can replay the event."

**Senior answer:** "No, and the reason is that it is two non-atomic operations with a
window between them. The commit succeeds, then the process dies — a `kill -9`, an
OOMKill, a node preemption, a rolling deploy — and the order exists with no event.
Nothing retries, because the code that would have retried is gone with the process. And
crucially there is no signal: no exception, no error metric, no consumer lag, no DLQ
entry. You find it in a reconciliation query weeks later.

Catching the exception does not help, because the failure I care about is the one where
no exception is thrown — the process simply stops between the two lines.

The variant I would raise even if nobody asked is
`@TransactionalEventListener(AFTER_COMMIT)`. Most Spring codebases use it believing it
solves this. It does not. It registers a `TransactionSynchronization` whose
`afterCommit` callback runs after `commit()` returns, on the same thread. It is exactly
the same window with a cleaner call site, and it is worse in one respect: an exception
thrown from an after-commit listener does not fail the caller, so it is logged and
swallowed by default.

Reversing the order is not the fix either — publish first and a rollback leaves an event
for an order that does not exist, and downstream consumers act on something false. That
is strictly worse than loss, because false effects have to be individually compensated.

The fix is the transactional outbox: the event becomes a row inserted in the same
transaction as the order, so there is one commit and no window. A relay publishes it
afterwards — polling with `SELECT ... FOR UPDATE SKIP LOCKED`, or Debezium reading the
WAL. The relay is at-least-once by construction, because it can crash between publishing
and marking the row sent, so consumers must be idempotent. I have not eliminated the
problem; I have converted silent permanent loss into visible bounded duplication, and
that is a trade I would make every time."

**What separates them:** the mid answer treats it as an error-handling problem. The
senior answer identifies it as an atomicity problem, names the specific crash modes,
emphasises the *absence of signal*, volunteers the `AFTER_COMMIT` equivalence unasked,
explains why the reversal is worse rather than merely also-wrong, and states what the
outbox does and does not guarantee. The last part is the strongest signal: someone who
says "and consumers must still be idempotent" has implemented one.

**Follow-up:** "How would you detect that this has been happening in a system you just
inherited?" — A left join from `orders` to whatever the consumer writes, over a window,
looking for orders with no downstream effect. Then correlate the timestamps with
deployment and restart events; clustering around restarts is the signature. If they do
not cluster, look for the cursor-relay bug instead.

---

### Q2 — "Walk me through your outbox relay. What are the failure modes?"

**Mid-level answer:** "It polls the outbox table for unsent rows, publishes them to
Kafka, and marks them as sent. If it crashes it picks up where it left off next time."

**Senior answer:** "The relay claims a bounded batch with
`SELECT ... FOR UPDATE SKIP LOCKED LIMIT n`, which lets multiple instances run with no
coordination — locked rows are skipped rather than waited for, so workers get disjoint
batches with no leader election and no distributed lock.

The transaction structure is the part I would spend review time on. I claim and commit
in one short transaction, publish outside any transaction, and mark sent in a second
short transaction. The alternative — holding the transaction across the publish — pins a
Hikari connection for the whole batch, so a Kafka slowdown becomes connection-pool
exhaustion and every endpoint fails, including ones that touch no Kafka. I would also
give the relay its own small `DataSource` so it cannot starve request threads at all.

That structure creates the failure mode I have to handle: a crash between publishing and
marking leaves rows `CLAIMED` forever. So there is a reaper that returns `CLAIMED` rows
older than a timeout to `PENDING`. That reaper is what makes the relay at-least-once
rather than at-most-once, and its timeout must exceed the worst-case publish time or it
manufactures duplicates on purpose.

Failure modes, ranked by how badly they bite:

The worst one is a relay that marks rows sent before the broker acks — including
fire-and-forget `send()` without awaiting the future, which loses events with no crash
at all. That is at-most-once wearing an outbox costume, and the table testifies that
everything was sent.

Second is a cursor-based relay using `WHERE id > last_seen`. Postgres sequences are
non-transactional, so a long transaction can allocate a lower id and commit *after* a
short one with a higher id. The cursor has already moved past it and that row is never
selected again. The `created_at` variant is broken identically, because `now()` is the
transaction start time.

Third is ordering. `SKIP LOCKED` gives disjoint batches with no ordering between them,
so two events for the same aggregate in different batches can be published out of order.
I either accept it and say so per topic, shard the relay by a hash of the aggregate id,
run a single worker, or use CDC, which reads the WAL in commit order.

Fourth is operational: no retention policy, no autovacuum tuning on a hot high-churn
table, and — the one that actually causes the incident — no alert on backlog *age*. If
the relay dies, every endpoint is green and consumer lag is zero, because there is
nothing to consume. Age is the only signal."

**What separates them:** the mid answer describes the happy path. The senior answer
leads with the transaction boundary and its connection-pool consequence, names the
reaper as the thing that establishes the delivery semantics, and knows the sequence-gap
bug — which is a database-internals fact, not a Java fact, and is the single strongest
signal in the answer. Alerting on age rather than depth is the operational signal.

**Follow-up:** "You said `SKIP LOCKED` breaks ordering. Does that matter for order
placement?" — It depends on whether two events for the *same order* can be in flight at
once. For a single `OrderPlaced` per order it cannot happen. For an order lifecycle —
placed, paid, shipped, cancelled — it absolutely can, and a `cancelled` arriving before
`placed` breaks a consumer that builds state. That is the case where I shard the relay
or use CDC.

---

### Q3 — "Why not just use a two-phase commit between Postgres and Kafka?"

**Mid-level answer:** "XA transactions are slow and hard to configure, and most people
avoid them. Kafka doesn't really support XA anyway."

**Senior answer:** "Three reasons, in increasing order of how fundamental they are.

The practical one: Kafka's transaction protocol is not XA. There is no XA resource
adapter to enlist. Postgres supports prepared transactions, but there is nothing to
coordinate them *with*. So the option does not physically exist for this pair, whatever
one thinks of XA.

The architectural one: even where XA exists, it makes availability strictly worse. A
2PC coordinator that dies after PREPARE and before COMMIT leaves resources holding locks
in an in-doubt state until a human or a recovery process resolves them. In Postgres that
means a prepared transaction holding locks and, worse, pinning the transaction horizon
so vacuum cannot clean up — one forgotten prepared transaction degrades the whole
database. You have traded a rare lost event for a rare total stall, which is a bad
trade for an e-commerce write path.

The fundamental one: it does not generalise. The next sink is an HTTP call to a payment
gateway, and there is no two-phase commit interface for `POST /charges`. Any design
whose correctness depends on 2PC breaks at the first integration that does not offer it.
The outbox works for every sink, because it only requires that the *event* be durable
and that the *sink* be idempotent — and both of those are things I can always arrange.

There is a version of the question worth taking seriously, which is Spring's old
`ChainedKafkaTransactionManager`. It looked like a distributed transaction and was
removed in Spring for Apache Kafka 3.0 precisely because it was read that way. It
synchronised two commits so one began after the other, leaving a window in between. If I
find it in a codebase, that codebase has this bug with a reassuring class name on top."

**What separates them:** the mid answer says XA is unfashionable. The senior answer
knows the protocol does not exist for this pair, knows the specific operational damage
of an in-doubt prepared transaction in Postgres (vacuum horizon, not just locks), and
makes the generalisation argument, which is the one that actually decides the design.
Naming `ChainedKafkaTransactionManager` unprompted signals real Spring experience.

**Follow-up:** "Is there any situation where you would use 2PC?" — Between two resources
that both genuinely support XA, where the coordinator is highly available, where the
business truly cannot tolerate an intermediate state, and where the transaction is
short. Two databases inside one bounded context, maybe. Never across a service boundary
you do not control, and never with an HTTP call involved.

---

### Q4 — "Your outbox table is 40 million rows and `POST /orders` p99 has doubled over three months with no code change. What happened?"

**Mid-level answer:** "The table got too big. We should add an index, or archive old
rows, or partition it."

**Senior answer:** "I would confirm the causal chain before changing anything, because
'the table is big' is a hypothesis, not a finding.

First, is the outbox actually the cause? `pg_stat_statements` ordered by
`total_exec_time`, filtered to statements touching `outbox` and `orders`. If the outbox
insert is not near the top, I am looking in the wrong place.

Assuming it is: 40 million rows means nothing is purging `SENT` rows. The direct cost is
that the primary key index no longer fits in shared buffers, so each insert costs random
I/O for the index page instead of a buffer hit. That is a real per-insert regression on
the hot path, and it grows with the table, which matches 'degraded slowly over months'.

Second thing I would check is `pg_stat_user_tables` for `n_dead_tup` and
`last_autovacuum`. The claim/mark design updates every row twice, and in Postgres an
UPDATE writes a new tuple and leaves a dead one. On a high-churn table, default
autovacuum settings — which are scale-factor based and therefore trigger later as the
table grows — fall behind. Bloat then makes both the heap and the indexes larger than
the live data justifies, which compounds the first problem.

The fixes, in order of how much they buy:

Partition by day and drop old partitions. `DROP TABLE` on a partition reclaims space
instantly with no vacuum at all, which sidesteps the whole bloat problem rather than
managing it. For a high-volume outbox this is the right answer.

If partitioning is too invasive right now, a retention job deleting `SENT` rows older
than the replay window, plus per-table autovacuum settings with a low scale factor and
a small threshold — a hot small table wants aggressive vacuuming, and tuning it globally
would be wrong for the rest of the schema.

Make sure the relay's index is partial: `WHERE status = 'PENDING'`. Then the index size
tracks the backlog rather than the history, and the claim query stays cheap regardless
of how much history the heap holds.

And I would fix the missing alert, because this should never have been discovered by a
latency graph. Table size and backlog age both belong on a dashboard.

Finally I would re-run the Topic 65 baseline after the fix. 'p99 doubled' is only
meaningful against a recorded baseline, and if the unchanged baseline no longer
reproduces, the environment drifted and the whole comparison is suspect."

**What separates them:** the mid answer lists remedies. The senior answer establishes
causality first, explains the *mechanism* by which size becomes latency (index no longer
cached, per-insert random I/O), knows Postgres UPDATE semantics and the scale-factor
behaviour that makes autovacuum fall behind precisely as the table grows, prefers
partition-drop over delete-and-vacuum for the right reason, and closes the loop on the
missing observability. Insisting on a baseline comparison is what a Topic 65 graduate
does.

**Follow-up:** "How long should the retention window be?" — Longer than the longest
replay you might need and longer than the maximum time a consumer group can be down and
still resume. Shorter than that and you have deleted the only durable record of an event
whose delivery you cannot confirm. I would derive it from the topic's `retention.ms`
and from the incident-response window, not pick a round number.

---

### Q5 — "When would you choose CDC over a polling relay?"

**Mid-level answer:** "CDC is lower latency and doesn't put load on the database from
polling. Debezium is the standard tool. Polling is simpler to start with."

**Senior answer:** "The latency and polling-load arguments are real but they are not the
one that decides it for me.

The decisive property is **ordering**. Logical decoding reads the WAL, which is in
commit order, so events come out in exactly the order the database committed them —
globally, not just per aggregate. A polling relay with `SKIP LOCKED` gives disjoint
batches with no ordering between workers, so I have to shard by aggregate or run a
single worker to get even per-aggregate ordering. If my consumers build state from an
event stream — an order lifecycle, say, where `cancelled` before `placed` is a bug — CDC
gives me for free something I would otherwise have to engineer and then continuously
prove.

Second, CDC makes the sequence-gap class of bug structurally impossible. There is no
cursor over a non-transactional sequence, because the WAL is the cursor and it is in
commit order by construction.

What it costs: a Kafka Connect cluster or a Debezium Server to operate, `wal_level =
logical` on the primary, and a replication slot — which is the failure mode that
actually matters. An inactive slot holds WAL that Postgres will not delete. If the
connector is down long enough, `pg_wal` fills the disk and the database stops accepting
writes. The outbox mechanism you added to protect the system takes the system down. I
would monitor `pg_replication_slots` for `active` and the retained WAL size, and set
`max_slot_wal_keep_size` so Postgres invalidates a runaway slot rather than filling the
disk — accepting that invalidation means a re-snapshot. That is a conscious choice
between losing the stream and losing the database, and it should be made before the
incident, not during it.

So: polling for a service where per-topic ordering is not required and the team has no
Connect capacity. CDC where ordering matters, where the volume makes polling load
material, or where I already run Connect for other reasons. I would start with polling
in `orderflow`, because it is debuggable with `psql` at 2am and the volume does not
demand more — and I would write down the specific trigger that would make me switch."

**What separates them:** the mid answer repeats the marketing. The senior answer picks
ordering as the deciding property and explains why, knows CDC eliminates a whole bug
class structurally, and — most importantly — leads with the replication-slot failure
mode and its mitigation. Anyone who has run Debezium has had that page. Naming a
specific trigger for switching, rather than a preference, is the principal-track move.

**Follow-up:** "You said you would write down the trigger. What is it?" — Something
measurable: when per-aggregate ordering becomes a requirement for any consumer, or when
the relay's claim query enters the top five of `pg_stat_statements` by total execution
time, or when backlog age p99 exceeds the downstream requirement at peak. A trigger that
is not measurable is a preference with a schedule attached.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `@TransactionalEventListener(AFTER_COMMIT)` runs after `commit()` returns, on the
   same thread. From that one fact alone, derive both why it does not fix the dual write
   *and* why an exception thrown from the listener cannot fail the caller.

2. Postgres sequences are non-transactional. Derive the cursor-relay bug from that fact
   without recalling it from the text. Then derive the equivalent bug for a
   `created_at` cursor, and name the different mechanism that causes it.

3. `SKIP LOCKED` gives relay workers disjoint batches. Name the property you gave up to
   get that, and design the smallest change that recovers it for one aggregate without
   giving up concurrency entirely.

4. The outbox converts silent permanent loss into visible duplication. Construct the
   scenario in which that trade is **wrong** — where a duplicate is worse than a loss —
   and say what you would do instead.

5. Suppose Postgres offered a hook that ran your code atomically with the commit, so you
   could publish to Kafka there. Explain precisely why that still would not give you
   exactly-once, and name the failure that remains.

6. Your relay marks rows `SENT` only after the broker acks. A colleague proposes marking
   before publishing "to avoid duplicates". Steelman their argument. Then say what
   measurement would settle it, and what you would need to know about the consumers.

7. You have an outbox and an idempotent consumer. Argue that the outbox is now
   redundant — that at-least-once delivery from a retry loop plus an idempotent consumer
   is equivalent. Find the flaw in your own argument.

---

## Quick reference card

### The window, stated three ways

```
commit(); send();                       <- window between them: EVENT LOST
send();   commit();                     <- rollback leaves a FALSE EVENT (worse)
@TransactionalEventListener(AFTER_COMMIT) <- SAME window as the first
insert order + insert outbox in ONE tx  <- no window; relay publishes later
```

### The outbox table

```sql
create table outbox (
  id bigserial primary key, aggregate_type text, aggregate_id text,
  event_type text, topic text, payload jsonb, headers jsonb,
  status text default 'PENDING',              -- PENDING | CLAIMED | SENT
  claimed_by text, claimed_at timestamptz,
  created_at timestamptz default now(), sent_at timestamptz,
  attempts int default 0, last_error text
);
create index outbox_pending_idx on outbox (id) where status = 'PENDING';
```

### The relay, in the correct order

```
TX1: SELECT ... FOR UPDATE SKIP LOCKED LIMIT n; UPDATE -> CLAIMED; COMMIT
     publish, AWAIT THE ACK                      (no DB connection held)
TX2: UPDATE -> SENT; COMMIT
reaper: CLAIMED older than timeout -> PENDING    (this makes it at-least-once)
purge:  SENT older than retention -> deleted / partition dropped
```

### Diagnostic SQL

```sql
-- backlog depth and age (alert on AGE)
select status, count(*), max(now() - created_at) from outbox group by 1;

-- rows a cursor relay would have skipped
select * from outbox where status='PENDING' and created_at < now() - interval '10 min';

-- bloat and vacuum health
select relname, n_live_tup, n_dead_tup, last_autovacuum, autovacuum_count
from pg_stat_user_tables where relname='outbox';

-- SKIP LOCKED behaviour, two sessions
begin; select id from outbox where status='PENDING'
       order by id for update skip locked limit 5;

-- CDC slot health (the one that can kill the database)
select slot_name, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
from pg_replication_slots;

-- reconciliation: orders with no downstream effect
select o.id from orders o
left join inventory_reservations r on r.order_id = o.id
where r.order_id is null and o.created_at > now() - interval '7 days';
```

### Gotchas checklist

- [ ] The outbox insert must be in the **caller's** transaction. No `REQUIRES_NEW`, no
      second `DataSource`. Assert it.
- [ ] `@TransactionalEventListener(AFTER_COMMIT)` is not a fix. `BEFORE_COMMIT` writing
      the outbox row is.
- [ ] Publish, **await the ack**, then mark. Never mark first. Never fire-and-forget.
- [ ] No cursor over `id` or `created_at`. Sequences and `now()` are not commit-ordered.
- [ ] Never hold the claiming transaction across the publish.
- [ ] Give the relay its own small connection pool.
- [ ] The reaper is not optional; without it the relay is at-most-once.
- [ ] Claim timeout must exceed worst-case publish time.
- [ ] Alert on backlog **age**, not depth.
- [ ] Retention policy plus per-table autovacuum tuning, from day one.
- [ ] Write down: "the relay is at-least-once; consumers must be idempotent."
- [ ] `@Scheduled` is proxy-based (Topic 40). A self-called `drain()` does nothing.

---

## When would I use this at work?

**1. The first design review of any service that owns data and emits events.**

Someone will have written `save(); publish();`. You ask one question — "what happens if
the process dies between those two lines?" — and the answer is always the same silence.
Then you show the reconciliation query that would find the damage in the existing
system. This converts an abstract objection into a number, in one meeting, and it is the
single highest-leverage thing in this document.

**2. Diagnosing "some orders never ship" with no errors anywhere.**

Every dashboard is green. No exceptions, no lag, no DLQ. You run the left-join
reconciliation, see the missing rows cluster around deploy timestamps, and you have the
diagnosis in ten minutes. If they do *not* cluster around deploys, you look for the
cursor-relay bug instead — old `PENDING` rows sitting below the cursor while newer rows
are `SENT`. Both diagnoses are five minutes with this document and a week without it.

**3. Reviewing a relay someone else wrote.**

You have four questions and they find almost every real bug: Is the mark after the ack?
Is there a cursor over `id`? Is the transaction held across the publish? Is there a
reaper? A relay that answers all four correctly is probably fine; one that fails any of
them has a specific, nameable, reproducible failure. Being able to ask those four
questions in a review is what makes this topic worth the day it takes.

---

## Connected topics

**Prerequisites:**

- **48 — Hibernate persistence context**: flush is not commit. The outbox insert must go
  through the same `EntityManager`/`DataSource` and be flushed in the same flush, or
  atomicity is an illusion.
- **54 — `@Transactional` semantics**: the outbox depends entirely on one transaction
  covering both writes. Self-invocation (Topic 40), `REQUIRES_NEW`, and the
  checked-exception rollback default each silently break it.
- **55 — Isolation and the connection pool**: why the relay must not hold its
  transaction across the Kafka publish, and why the naive relay causes a full outage.
- **65 — The load baseline**: every latency claim about the outbox's cost is a delta
  against `/docs/java/baselines/`.
- **90 — Executors and bounded queues**: the relay's batch size *is* a bound. Unbounded
  batches are unbounded work.
- **109 — HikariCP**: give the relay its own pool; a relay stall must not starve
  `POST /orders`.
- **111 — Resilience4j**: the relay's publish is the natural place for a circuit breaker,
  and the retry semantics must be idempotent — which they are, because the row is still
  `CLAIMED`.
- **114 — Kafka delivery semantics**: the relay is at-least-once by construction, and
  the producer must be idempotent so retries do not reorder or duplicate inside Kafka.

**This unlocks:**

- **116 — Idempotency**: the promise the outbox makes to consumers. Without it, the
  duplicates the relay produces are a new bug rather than an absorbed one.
- **117 — Sagas**: every saga step's command and reply travel through an outbox. A saga
  built on a dual write is a saga that silently stops halfway.
- **118 — Micrometer**: the backlog-depth and backlog-age gauges, the published and
  reclaimed counters, and the rule against tagging any of them with an aggregate ID.
- **119 — Tracing**: the trace context must be stored in the outbox row's headers at
  write time and re-injected by the relay, or the consumer's span becomes a new trace —
  the outbox is a thread and a process boundary.
- **120 — MDC and structured logging**: the outbox row id and aggregate id belong in
  every relay log line, or a duplicate is not greppable.
- **121 — Readiness probes**: a pod whose relay cannot reach Kafka is still ready to
  serve HTTP. Deciding that honestly is Topic 121's subject.
- **123 — Graceful shutdown**: the relay must finish or safely abandon its in-flight
  batch on `SIGTERM`, and readiness must go false first. This is called out explicitly
  in Topic 123's mastery line.
- **129 — Capacity**: outbox write amplification, WAL generation and table growth all
  belong in the capacity model.
- **130 — SLOs**: backlog age is a genuine SLI — "an order's event is published within N
  seconds" is a user-visible promise.
- **133 — Postmortems**: the `kill -9` drill is a ready-made incident to write up.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0. Three things in
this document are deliberately hedged rather than asserted: Spring Modulith's event
publication registry semantics and configuration, Debezium's outbox event router
configuration keys, and the exact `spring-kafka` version resolved by the Boot BOM —
check each in current documentation rather than trusting a recalled version number. The
Postgres facts — non-transactional sequences, `now()` as transaction start time, UPDATE
producing a dead tuple, `SKIP LOCKED` semantics, and replication-slot WAL retention —
are stable across every supported Postgres release and are the load-bearing parts of the
document. The mechanical statement has been true since the first time anyone wrote
`save()` on one line and `publish()` on the next.*
