# 53 — Hibernate VI — Write Performance: Batching and the IDENTITY Trap

## Phase: 5 — Spring Boot & Persistence
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 (runtime JDK 25)
## Project spine: the two bulk-write paths `orderflow` cannot reach the Phase 7 gate without — the **seed loader** that must produce ≥100k products, ≥1M orders and ≥5M order lines for Topic 65's baseline, and the **nightly settlement writer** that updates ~200k `Payment` rows. Both are unusable at Hibernate's default settings, and the reason is not the settings you would first reach for.

---

## Mechanical statement

Read this twice. Every section below is an elaboration of it.

> **JDBC batching groups statements into one round trip.**
> `PreparedStatement.addBatch()` accumulates parameter sets in the driver; `executeBatch()`
> sends them together. One network round trip carries N rows instead of N round trips
> carrying one row each.
>
> **`GenerationType.IDENTITY` requires the database to return the key per row.**
> Hibernate's persistence context is a map keyed by entity identifier (Topic 48). It cannot
> put an entity in that map without an id. With `IDENTITY`, the id only exists after the
> `INSERT` has run. So `persist()` must execute the `INSERT` **immediately**, one row at a
> time, to read the generated key back.
>
> **Therefore `hibernate.jdbc.batch_size` does nothing on an `IDENTITY` entity.**
> No error. No warning. No log line. Hibernate detects the situation and silently disables
> insert batching for that entity. You set a property, restarted, and changed nothing.
>
> **`GenerationType.SEQUENCE` with a pooled optimiser is what makes batching possible**,
> because Hibernate can allocate ids in advance, in blocks, without executing the insert.
> Then `order_inserts` and `order_updates` keep same-table statements adjacent so a batch is
> not broken every time the entity type changes. And you must `flush()` and `clear()`
> periodically, or the persistence context's per-entity snapshot grows linearly until the
> heap does not hold it.

Four settings, one prerequisite. The prerequisite is the id generator, and it is the one
nobody changes.

---

## The bridge from what you know

### There is no Prisma analogue. This is genuinely new.

State this plainly because pattern-matching against your TypeScript experience will actively
mislead you here.

```ts
// Prisma
await prisma.orderLine.createMany({
  data: lines,          // 50,000 objects
});
```

`createMany` compiles to a **single multi-row `INSERT`** — `INSERT INTO order_line (...)
VALUES (...),(...),(...)` — chunked by the driver's parameter limit. One statement, a
handful of round trips, done. You have never had to think about it, because Prisma made the
efficient thing the obvious thing.

```ts
// The other Prisma shape people reach for, which is NOT batching
await prisma.$transaction(lines.map(l => prisma.orderLine.create({ data: l })));
```

That one *is* 50,000 statements — but they are wrapped in a transaction, and the atomicity
is what you were thinking about, not the round trips. Prisma does not expose a batch-size
knob, does not expose an identity-versus-sequence choice, and does not have a concept of
"batching was silently disabled".

**So three things do not transfer:**

1. **The idea that an ORM might issue one statement per row by default.** Hibernate does. It
   is the default and it is not flagged anywhere.
2. **The idea that your primary key strategy determines your write throughput.** In Prisma
   you write `@id @default(autoincrement())` and never think about it again. In Hibernate
   that exact choice — `IDENTITY` — is what caps your insert rate.
3. **The idea that a performance setting can be silently ignored.** You are used to a config
   value either working or erroring. `hibernate.jdbc.batch_size` on an `IDENTITY` entity does
   neither.

**One thing transfers perfectly, and it is the mental model to lean on:** you already
understand **N+1 on reads** from Topic 50 — one query for the parents, then one per child.
This is **N+1 on writes**, and unlike the read version, no ORM tool warns you about it and
no query log makes it obvious, because every one of those N statements is individually
correct and fast.

### What JDBC batching is *not*: a multi-row INSERT

This distinction matters and almost nobody states it.

```sql
-- (A) JDBC batching: ONE prepared statement, N parameter sets, sent together
PREPARE: insert into order_line (order_id, product_id, quantity, unit_price_minor, id)
         values (?, ?, ?, ?, ?)
BIND+ADD x 50
EXECUTE BATCH                       -- one round trip, 50 executions server-side
```

```sql
-- (B) Multi-row INSERT: ONE statement containing N tuples
insert into order_line (order_id, product_id, quantity, unit_price_minor, id)
values (?,?,?,?,?), (?,?,?,?,?), ... x 50
                                    -- one round trip, ONE execution server-side
```

(A) is what `hibernate.jdbc.batch_size` produces. (B) is what Prisma's `createMany`
produces, and it is faster still, because Postgres parses and plans once rather than
executing the same plan fifty times.

**The Postgres JDBC driver can turn (A) into (B) for you**, with a connection parameter:

```
jdbc:postgresql://localhost:5432/orderflow?reWriteBatchedInserts=true
```

This is the single highest-leverage line in this whole topic and it is off by default. It
only applies to `INSERT`s (not updates), and only when the batched statements are
rewriteable. Measure it — Phase 5 of the failure drill does exactly that.

**Verdict: NO ANALOGUE for the mechanism, HONEST ANALOGUE for the intuition** that round
trips dominate. Your Node experience taught you to fear the loop-of-awaits. This is the same
fear, applied to a place where the ORM hides the loop from you.

---

## What is this?

Two independent machines that have to agree before anything batches.

### Machine 1 — the JDBC batch API

```java
PreparedStatement ps = conn.prepareStatement(
        "insert into order_line (order_id, product_id, quantity, unit_price_minor, id) " +
        "values (?, ?, ?, ?, ?)");

for (OrderLine line : lines) {
    ps.setLong(1, line.orderId());
    ps.setLong(2, line.productId());
    ps.setInt(3, line.quantity());
    ps.setLong(4, line.unitPriceMinor());
    ps.setLong(5, line.id());          // <-- the id must ALREADY be known
    ps.addBatch();                     // buffered in the driver, no network traffic
}

int[] counts = ps.executeBatch();      // ONE round trip
```

Line 10 is the entire topic. `addBatch()` requires every parameter, including the primary
key. If the key comes from the database, you cannot bind it, so you cannot batch.

### Machine 2 — Hibernate's action queue

Between `persist()` and the actual SQL, Hibernate maintains an **action queue**: a list of
pending `EntityInsertAction`, `EntityUpdateAction` and `EntityDeleteAction` objects. At
flush, it walks that queue and emits SQL.

Two properties control how it walks:

| Property | Default | Effect |
|---|---|---|
| `hibernate.jdbc.batch_size` | unset (no batching) | how many statements to accumulate before `executeBatch()` |
| `hibernate.order_inserts` | `false` | sort pending inserts by entity type before emitting |
| `hibernate.order_updates` | `false` | sort pending updates by entity type and id |
| `hibernate.batch_versioned_data` | `true` on modern Hibernate | allow batching of `@Version`-carrying updates |

> **Honest flag:** the defaults above are what I understand modern Hibernate to use, and
> `batch_versioned_data` in particular has changed default across Hibernate 4/5. **Settling
> command:** turn on `logging.level.org.hibernate.cfg=DEBUG` (or dump the
> `SessionFactory`'s `getSessionFactoryOptions()` at startup) and read the effective values
> your build resolved, rather than trusting this table.

In Spring Boot these are set through the pass-through prefix:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
        batch_versioned_data: true
        generate_statistics: true
```

**The prefix is load-bearing.** `spring.jpa.properties.*` is passed verbatim to Hibernate;
anything under it that Hibernate does not recognise is **silently ignored**. Anything you
put under `spring.jpa.hibernate.*` instead is a different (small, Boot-owned) namespace and
your setting never reaches Hibernate at all. That is Trap 5.

### The identifier generation strategies

| Strategy | How the id is obtained | Batchable inserts? | Notes |
|---|---|---|---|
| `IDENTITY` | database column default (Postgres `serial`/`GENERATED ... AS IDENTITY`); read back per row | **No** | `persist()` executes the INSERT immediately |
| `SEQUENCE` | `select nextval('...')` before the insert | **Yes** | with an optimiser, one `nextval` per N entities |
| `TABLE` | a row in a dedicated table, updated under a lock | Yes, but | serialises every id allocation. Do not use it. |
| `AUTO` | Hibernate picks | depends | on Hibernate 6+ against Postgres this resolves to a sequence strategy |
| `UUID` / application-assigned | generated in the JVM | **Yes** | no round trip at all; costs index locality |

> `[BOOT 3.x DELTA]` — worth knowing because it changes what `AUTO` means. On Hibernate 5
> (Boot 2.x), `GenerationType.AUTO` against Postgres commonly resolved to a **single shared**
> `hibernate_sequence` for every entity in the application. Hibernate 6 (Boot 3.x) and
> Hibernate 7 (Boot 4.x) moved to **per-entity sequences**, conventionally named
> `<table>_seq`. If you are migrating a Boot 2.x application, that is a schema change, and
> it is the kind that appears as `relation "order_seq" does not exist` on first insert after
> the upgrade. **I am confident about the direction of this change and less confident about
> the exact naming convention on Hibernate 7** — settle it by turning on
> `spring.jpa.show-sql=true` with `spring.jpa.hibernate.ddl-auto=create-drop` against a
> throwaway database and reading the generated DDL. Do not guess a sequence name into a
> Flyway migration.

**The recommendation for `orderflow`, stated once:** every entity uses
`GenerationType.SEQUENCE` with an explicit `@SequenceGenerator` and an `allocationSize`
you chose deliberately. Topic 48 already made this choice for `Order` and `OrderLine`
precisely so this topic is not a migration.

### The `flush()` / `clear()` obligation

From Topic 48: the persistence context holds, for every managed entity, **the entity plus a
snapshot array of its loaded field values**, so dirty checking can diff at flush. Insert
50,000 entities in one persistence context and you hold 50,000 entities, 50,000 snapshot
arrays, 50,000 `EntityEntry` objects and the map entries that index them — none of which you
will ever read again.

```java
for (int i = 0; i < lines.size(); i++) {
    em.persist(lines.get(i));
    if (i % BATCH_SIZE == 0) {
        em.flush();      // send the accumulated batch to the database
        em.clear();      // detach everything; release the entities and their snapshots
    }
}
```

`flush()` without `clear()` sends the SQL but keeps everything managed — memory still grows.
`clear()` without `flush()` **discards pending work**. They go together, in that order, and
the interval is normally the batch size.

---

## Machine-level reality

This is the section that makes the topic a differentiator. Everything here is checkable, and
where I am uncertain I say so and give you the command.

### Why `IDENTITY` forces a round trip per row — the exact mechanism

Hibernate's first-level cache is, structurally, a `Map<EntityKey, Object>` where `EntityKey`
is `(entityName, identifier)`. It is the thing that makes `em.find(OrderLine.class, 7L)`
return the same instance twice, and it is what dirty checking iterates at flush.

You cannot construct an `EntityKey` without an identifier.

So when you call `persist(orderLine)`:

- **With `SEQUENCE`**, Hibernate can obtain an id *without touching the row*: it reads
  `nextval` (or takes one from a pre-allocated block held in memory), assigns it to the
  entity, builds the `EntityKey`, puts the entity in the persistence context, and queues an
  `EntityInsertAction` for later. **Nothing is written yet.** At flush, it walks the queue
  and can bind every parameter — including the id — into a batch.

- **With `IDENTITY`**, the id does not exist until the row exists. Hibernate has exactly one
  way to get it: execute the `INSERT` now and read the generated key back. On Postgres that
  is `INSERT ... RETURNING id` or `getGeneratedKeys()`. Only then can it build the
  `EntityKey`. So `persist()` performs a synchronous, unbatchable database round trip, and
  the `EntityInsertAction` is executed immediately rather than queued.

Hibernate is explicit about this: it disables insert batching for `IDENTITY` entities. It
does so **silently**, because from Hibernate's point of view nothing is wrong — you asked for
a generation strategy that is incompatible with deferred execution and it honoured your
request.

**Two consequences beyond throughput, which are the ones that surprise people:**

1. **`persist()` writes immediately, which breaks Topic 48's mental model.** "Nothing happens
   until flush" is false for `IDENTITY` entities. Code that relied on being able to
   `persist()` and then change its mind before flush has a row in the database.
2. **Row and index locks are taken earlier and held longer.** Every `INSERT` takes locks that
   are held until commit. With `IDENTITY`, insert #1's locks are held for the entire duration
   of inserting rows #2 through #50,000. With `SEQUENCE` + batching, all the inserts happen at
   flush, near the end of the transaction, so the lock window is a fraction of the size. That
   is a Topic 52 consequence you get for free from a Topic 53 change.

### The arithmetic: round trips versus everything else

This is where the intuition should live. Use realistic numbers.

| Path | Round-trip time |
|---|---|
| Same-host / Unix socket | ~0.05–0.1 ms |
| Same availability zone | ~0.2–0.5 ms |
| Cross-AZ within a region | ~0.5–2 ms |
| Cross-region | 20–150 ms |

Take **0.3 ms**, a reasonable same-AZ figure, and 50,000 `OrderLine` inserts:

| Configuration | Round trips | Network time alone |
|---|---|---|
| `IDENTITY`, batching "enabled" | 50,000 | **15 seconds** |
| `SEQUENCE` `allocationSize=1`, batch 50 | 50,000 `nextval` + 1,000 batches = 51,000 | **15.3 seconds** — *worse* |
| `SEQUENCE` `allocationSize=50` pooled, batch 50 | 1,000 `nextval`/50 = 1,000 allocations… ≈ 1,000 + 1,000 | ~0.6 seconds |
| …with `reWriteBatchedInserts=true` | 1,000 batches, each one multi-row statement | ~0.3 seconds + less server CPU |

That second row is the one to internalise: **switching to `SEQUENCE` without setting
`allocationSize` makes things worse, not better.** You have replaced one round trip per row
with two. People make this change, measure, see no improvement, and conclude batching does
not work.

The third row assumes `allocationSize = 50` and a pooled optimiser: one `nextval` call yields
50 usable ids, so 50,000 rows need 1,000 `nextval` calls, and 50,000 rows at batch size 50
need 1,000 `executeBatch()` calls.

**And the honest caveat:** these are *network* numbers. They do not include Postgres's work —
index maintenance, WAL writes, constraint checks, and eventually autovacuum. For a 5M-row
seed load, the database's own work becomes the bottleneck long before the network does, which
is why the production example ends by telling you to use `COPY` instead. Batching removes the
round-trip cost. It does not make Postgres faster at inserting.

### Sequence optimisers, and the mismatch that produces duplicate keys

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_line_seq")
@SequenceGenerator(name = "order_line_seq",
                   sequenceName = "order_line_seq",
                   allocationSize = 50)
private Long id;
```

`allocationSize = 50` tells Hibernate: "one `nextval` gives me a block of 50 ids I may use
without asking again."

The three optimisers you will meet:

| Optimiser | How it uses the returned value | Sequence `INCREMENT BY` must be |
|---|---|---|
| `none` | one `nextval` per entity | 1 |
| `pooled` | `nextval` returns the **top** of the block; Hibernate uses `[value - allocationSize + 1, value]` | `allocationSize` |
| `pooled-lo` | `nextval` returns the **bottom** of the block; Hibernate uses `[value, value + allocationSize - 1]` | `allocationSize` |
| `hilo` | legacy multiplicative scheme | 1, but the ids are not database-friendly |

**`pooled` and `pooled-lo` are database-friendly**: another application (or a `psql` session)
calling `nextval` on the same sequence gets a value outside your block, so ids do not
collide. `hilo` is not — it multiplies, so a plain `nextval` from outside lands inside a
block Hibernate thinks it owns. Use `pooled`; it is the modern Hibernate default when
`allocationSize > 1`.

**The mismatch trap.** If your DDL says:

```sql
create sequence order_line_seq start with 1 increment by 1;     -- WRONG for allocationSize=50
```

and your entity says `allocationSize = 50`, Hibernate reserves 50 ids from a sequence that
only advanced by 1. Two application instances — or one instance across a restart — will hand
out overlapping ids and you get intermittent
`ERROR: duplicate key value violates unique constraint "order_line_pkey"`. Intermittent,
because it only fires when two blocks actually overlap on a row that gets inserted.

```sql
create sequence order_line_seq start with 1 increment by 50;    -- matches allocationSize
```

Hibernate can detect a mismatch and log a warning about it, but the warning is easy to miss
and does not stop startup. **Treat the sequence `INCREMENT BY` and the entity's
`allocationSize` as one value expressed in two files**, and put them next to each other in
the Flyway migration's comment.

**What `allocationSize` costs you:** ids are not gapless. A restart discards the unused
remainder of the current block, so `allocationSize = 50` can leave gaps of up to 49. If
anyone is treating the primary key as a countable business number — "we have processed
1,204,881 orders" — that breaks. It should not be, and the fix is a separate business
sequence, not a smaller `allocationSize`.

### Why `order_inserts` matters — a JDBC batch is per statement

A JDBC batch belongs to **one `PreparedStatement`**, which means one SQL string, which means
one table. The moment Hibernate needs to emit SQL for a different table, it must execute the
current batch and start a new one.

Now consider the natural way to write the seed loader:

```java
for (int i = 0; i < 10_000; i++) {
    Order order = new Order(...);
    em.persist(order);                                // action queue: INSERT orders
    for (int j = 0; j < 5; j++) {
        em.persist(new OrderLine(order, ...));        // action queue: INSERT order_line
    }
}
```

Without `order_inserts`, the action queue preserves your call order:

```
orders, order_line x5, orders, order_line x5, orders, order_line x5, ...
```

Every transition between `orders` and `order_line` flushes the current batch. Your effective
batch sizes are 1 and 5, no matter what `batch_size` says. You configured batching, you can
see batching in the code, and you get almost none of it.

With `hibernate.order_inserts=true`, Hibernate sorts the pending inserts by entity type
before emitting:

```
orders x10,000, order_line x50,000
```

Now the batches fill to `batch_size`. Same code, same property for `batch_size`, an order of
magnitude fewer round trips.

`order_updates` does the same for updates, additionally sorting by id — which has a second
benefit: **consistent update ordering reduces deadlocks** (Topic 52, Trap 5), because
concurrent transactions touch rows in the same order.

**The cost of `order_inserts`:** Hibernate must hold the whole action queue and sort it,
which means it cannot start emitting until flush. For a very large flush this is memory and
a sort. In practice it is dominated by the win, and the `flush()`/`clear()` discipline caps
the queue size anyway.

**One real correctness note:** sorting inserts by entity type can change the order in which
foreign-key parents and children are written. Hibernate is aware of FK dependencies and
orders types accordingly, but if you have a cycle in your FK graph or a deferred-constraint
scheme, verify. For `orderflow`'s tree-shaped schema (`Order` → `OrderLine` → `Product`),
this is not an issue.

### The memory shape of an unbounded persistence context

Per managed entity, in round terms:

- the entity object itself (header + fields — Topic 69's arithmetic)
- an `Object[]` snapshot with one slot per persistent attribute, **plus** the boxed values
  in it
- an `EntityEntry` holding the key, status, version and a reference to the snapshot
- entries in the `EntityKey` map and the identity map

For an `OrderLine` with five persistent attributes, a defensible estimate is **a few hundred
bytes per entity**, dominated by the snapshot's boxed values. I am not going to state a
precise number from memory — the honest instruction is: **measure it with JOL** (Topic 69,
`ClassLayout.parseInstance(...)`) or take a heap dump mid-load and look at the retained size
of the `StatefulPersistenceContext` (Topic 79).

What you can predict without measuring is the **shape**: linear in the number of entities
persisted since the last `clear()`, never released until then, and all of it in the young
generation initially, then promoted as the load continues — which is the worst thing you can
do to a generational collector (Topic 68). The observable symptom is not usually
`OutOfMemoryError` first; it is the GC log showing full collections that reclaim almost
nothing, with the loader's throughput collapsing as the JVM spends its time tracing a live
set that only grows.

### `StatelessSession` — the escape hatch, and what it costs

```java
StatelessSession session = sessionFactory.openStatelessSession();
```

A `StatelessSession` has **no persistence context**. No first-level cache, no snapshots, no
dirty checking, no cascade, no lifecycle events, no second-level cache interaction. You
issue `insert(entity)` / `update(entity)` explicitly and they map directly to SQL.

**When it wins:** a pure bulk load or a pure bulk update, where you never re-read what you
wrote and you do not need cascading. Memory is flat regardless of row count — no
`flush()`/`clear()` discipline required, because there is nothing to clear.

**What you give up:** everything Topics 48–52 are about. Cascades do not happen. Dirty
checking does not happen — an unwritten change is simply lost. `@Version` handling is manual.
Associations are not managed. It is a thin, honest layer over JDBC with entity mapping, and
treating it as a faster `EntityManager` will silently lose writes.

**And the honest comparison for a seed load:** `StatelessSession` with batching is a large
win over a stateful session. `COPY` via the Postgres driver's `CopyManager` is a much larger
win still, and for the specific job of "put 5 million rows into an empty table" it is simply
the right tool. Example 2 says so out loud.

---

## Example 1 — minimal

The smallest thing that demonstrates the trap. Two entities, identical in every respect
except the generation strategy.

```java
package com.orderflow.lab;

import jakarta.persistence.*;

@Entity
@Table(name = "line_identity")
public class LineIdentity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)      // the trap
    private Long id;

    @Column(nullable = false) private String sku;
    @Column(nullable = false) private int quantity;

    protected LineIdentity() {}
    public LineIdentity(String sku, int quantity) { this.sku = sku; this.quantity = quantity; }
}
```

```java
package com.orderflow.lab;

import jakarta.persistence.*;

@Entity
@Table(name = "line_sequence")
public class LineSequence {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "line_sequence_seq")
    @SequenceGenerator(name = "line_sequence_seq",
                       sequenceName = "line_sequence_seq",
                       allocationSize = 50)                  // MUST match the DDL
    private Long id;

    @Column(nullable = false) private String sku;
    @Column(nullable = false) private int quantity;

    protected LineSequence() {}
    public LineSequence(String sku, int quantity) { this.sku = sku; this.quantity = quantity; }
}
```

```sql
-- src/main/resources/schema.sql
create table if not exists line_identity (
  id bigint generated by default as identity primary key,
  sku varchar(32) not null,
  quantity int not null
);

create sequence if not exists line_sequence_seq start with 1 increment by 50;  -- == allocationSize
create table if not exists line_sequence (
  id bigint primary key,
  sku varchar(32) not null,
  quantity int not null
);
```

```java
package com.orderflow.lab;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class BatchProbe {

    @PersistenceContext private EntityManager em;
    private final SessionFactory sessionFactory;

    public BatchProbe(EntityManager em, SessionFactory sessionFactory) {
        this.sessionFactory = sessionFactory;
    }

    private Statistics stats() { return sessionFactory.getStatistics(); }

    @Transactional
    public void insertIdentity(int rows, int batchSize) {
        stats().clear();
        for (int i = 1; i <= rows; i++) {
            em.persist(new LineIdentity("SKU-" + i, 1));
            if (i % batchSize == 0) { em.flush(); em.clear(); }
        }
        em.flush();
        report("IDENTITY", rows);
    }

    @Transactional
    public void insertSequence(int rows, int batchSize) {
        stats().clear();
        for (int i = 1; i <= rows; i++) {
            em.persist(new LineSequence("SKU-" + i, 1));
            if (i % batchSize == 0) { em.flush(); em.clear(); }
        }
        em.flush();
        report("SEQUENCE", rows);
    }

    private void report(String label, int rows) {
        Statistics s = stats();
        System.out.println(label
            + " rows=" + rows
            + " entityInserts=" + s.getEntityInsertCount()
            + " prepareStatements=" + s.getPrepareStatementCount()
            + " flushes=" + s.getFlushCount());
    }
}
```

Run both with `rows = 1000` and `batch_size = 50`.

**What to reason about before running.** `entityInsertCount` will be 1,000 in both cases —
that counts entities, not statements. The number that separates them is
`prepareStatementCount`. Write down your prediction for each, then run it. The interpretation
table is in the Failure drill.

The point of the two-entity design is that **nothing else differs**. Same column count, same
data, same batch size, same transaction, same flush discipline. The only variable is the
annotation on the id field.

---

## Example 2 — production scenario (on the project spine)

### The requirement

Topic 65 is a hard gate: `orderflow` cannot enter Phase 8 without a reproducible load
baseline against a realistic dataset. The dataset specification from the master plan is:

- **≥100,000 products**
- **≥1,000,000 orders**
- **≥5,000,000 order lines**
- realistic cardinality and skew — a few hot products carrying a disproportionate share

Constraints:

- The loader runs in CI against a Testcontainers Postgres (Topic 61) and must fit the CI
  time budget. A loader that takes 40 minutes means nobody re-seeds, which means the baseline
  drifts and the gate is worthless.
- The CI container has a **1 GB heap**. The loader must not OOM.
- It must be re-runnable and deterministic, so two people get comparable baselines.

### Attempt 1 — the version everybody writes first

```java
@Service
public class NaiveSeedLoader {

    @PersistenceContext private EntityManager em;

    @Transactional
    public void load(int orderCount) {
        for (int i = 0; i < orderCount; i++) {
            Order order = new Order("seed-" + i, randomCustomerId(), Instant.now());
            em.persist(order);
            for (int j = 0; j < 5; j++) {
                order.addLine(new OrderLine(order, randomProduct(), 1, 1999L));
            }
        }
    }
}
```

With `Order` and `OrderLine` on `GenerationType.IDENTITY` this does the following, and every
part of it is bad:

1. **6,000,000 individual round trips** (1M orders + 5M lines), because `IDENTITY` disables
   batching. At 0.3 ms each that is **30 minutes of network time alone**, before Postgres
   does any work.
2. **6,000,000 managed entities in one persistence context**, because there is no
   `flush()`/`clear()`. On a 1 GB heap this OOMs somewhere in the low hundreds of thousands.
3. **One transaction for the entire load.** It pins a HikariCP connection for the whole run
   (Topic 55), holds every row and index lock it takes for the whole run (Topic 52), produces
   an enormous amount of WAL that cannot be checkpointed away, and fails all-or-nothing —
   twenty-five minutes in, one constraint violation and you start again.

Setting `hibernate.jdbc.batch_size=50` fixes **none** of it, and that is the observation this
topic exists for.

### Attempt 2 — the version that ships

**Step 1: the entities use `SEQUENCE` with a pooled block.** Topic 48 already established
this, deliberately, so there is no migration here.

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_line_seq")
@SequenceGenerator(name = "order_line_seq",
                   sequenceName = "order_line_seq",
                   allocationSize = 500)      // larger for the loader; see the note below
private Long id;
```

```sql
-- V1__sequences.sql  (Flyway)
-- allocationSize in OrderLine.java MUST equal INCREMENT BY here. Change them together.
create sequence order_line_seq start with 1 increment by 500;
create sequence order_seq      start with 1 increment by 500;
```

> **On `allocationSize = 500`.** For a 5M-row load this reduces `nextval` round trips from
> 5,000,000 (allocationSize 1) to 10,000. The cost is larger gaps in the id space after a
> restart — up to 499. For `orderflow` that is fine, because nothing treats the id as a
> countable business quantity; the order's business identity is its `idempotencyKey`
> (Topic 48), not its surrogate key. **State this trade in the migration's comment**, because
> the next person will want to know why it is not 50.

**Step 2: configuration in a dedicated profile**, so the loader's aggressive settings do not
apply to the serving path.

```yaml
# application-seed.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orderflow?reWriteBatchedInserts=true
    hikari:
      maximum-pool-size: 4          # the loader is one thread; a big pool buys nothing
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 100
        order_inserts: true
        order_updates: true
        batch_versioned_data: true
        generate_statistics: true   # SEED PROFILE ONLY — see the note
logging:
  level:
    org.hibernate.stat: DEBUG
```

> **`generate_statistics=true` is not free.** It adds counters on every session operation.
> Enable it in the seed and load-analysis profiles; do **not** leave it on in the profile
> that serves 1,200 rps. Turning it on to diagnose production and forgetting to turn it off
> is a classic way to make a service permanently slower for no visible reason.

**Step 3: chunked transactions, not one giant one.**

```java
package com.orderflow.seed;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class SeedLoader {

    private static final int ORDERS_PER_CHUNK = 1_000;      // -> ~6,000 rows per transaction

    private final SeedChunkWriter chunkWriter;              // SEPARATE BEAN. Topic 40.

    public SeedLoader(SeedChunkWriter chunkWriter) { this.chunkWriter = chunkWriter; }

    /** NOT transactional. Each chunk gets its own transaction. */
    public void load(int totalOrders) {
        for (int start = 0; start < totalOrders; start += ORDERS_PER_CHUNK) {
            int end = Math.min(start + ORDERS_PER_CHUNK, totalOrders);
            chunkWriter.writeChunk(start, end);             // through the PROXY
            if (start % 100_000 == 0) {
                System.out.println("seeded orders " + start + "/" + totalOrders);
            }
        }
    }
}
```

```java
package com.orderflow.seed;

@Service
public class SeedChunkWriter {

    private static final int FLUSH_EVERY = 100;             // == hibernate.jdbc.batch_size

    @PersistenceContext private EntityManager em;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void writeChunk(int fromIndex, int toIndex) {
        int persisted = 0;
        for (int i = fromIndex; i < toIndex; i++) {
            Order order = new Order("seed-" + i, customerFor(i), timestampFor(i));
            em.persist(order);
            persisted++;

            for (int j = 0; j < linesFor(i); j++) {
                em.persist(new OrderLine(order, skewedProduct(i, j), qtyFor(i, j), priceFor(i, j)));
                persisted++;
            }

            if (persisted >= FLUSH_EVERY) {
                em.flush();
                em.clear();
                persisted = 0;
            }
        }
        em.flush();
        em.clear();
    }
}
```

Five decisions worth defending:

**1. `SeedChunkWriter` is a separate bean.** `writeChunk` must go through a proxy for
`@Transactional` to apply at all — calling it from a sibling method in `SeedLoader` would be
Topic 40's self-invocation trap, and the entire chunking would silently become one
transaction (or no transaction). A separate bean makes the boundary real. The class name
states the guarantee.

**2. `flush()` + `clear()` every 100 entities, matching `batch_size`.** They must be the same
number, or you are either flushing a partly-filled batch or letting the queue grow past the
batch size for no benefit.

**3. `em.clear()` detaches `order` too**, which is why the `OrderLine`s are constructed and
persisted *before* the flush boundary for that order. If a chunk's flush landed between
persisting an `Order` and its lines, the lines would reference a detached parent. The
`persisted >= FLUSH_EVERY` check after the inner loop is what keeps each order's lines with
their order. **This is the kind of detail that only shows up as an intermittent
`TransientObjectException` at 3am**, so it is worth a comment in the real file.

**4. Chunked transactions mean partial progress survives.** A failure at 60% leaves 60%
loaded and tells you where to resume. It also caps WAL growth, caps lock hold time, and
returns the pool connection between chunks (Topics 55, 109).

**5. Skew is deliberate.** `skewedProduct(i, j)` draws from a Zipf-like distribution so a
handful of products carry most of the lines. A uniformly random dataset makes every index
lookup equally cheap and every cache equally useless, and the Topic 65 baseline it produces
does not resemble production. Realistic skew is what makes the Phase 8 profiling work find
anything.

### The honest recommendation: for a pure seed load, do not use Hibernate at all

Everything above is correct and it is the right tool when you need entity semantics —
cascades, `@Version`, listeners, computed fields. **For "put five million rows into an empty
table", it is the wrong tool by about an order of magnitude.**

```java
package com.orderflow.seed;

import org.postgresql.copy.CopyManager;
import org.postgresql.core.BaseConnection;

@Component
public class CopySeedLoader {

    private final DataSource dataSource;

    public CopySeedLoader(DataSource dataSource) { this.dataSource = dataSource; }

    public long copyOrderLines(Reader csvRows) throws SQLException, IOException {
        try (Connection c = dataSource.getConnection()) {
            CopyManager copy = new CopyManager(c.unwrap(BaseConnection.class));
            return copy.copyIn(
                "COPY order_line (id, order_id, product_id, quantity, unit_price_minor) "
                + "FROM STDIN WITH (FORMAT csv)",
                csvRows);
        }
    }
}
```

`COPY` streams rows into Postgres with no per-row statement, no per-row parse, and minimal
protocol overhead. For a bulk load into a table you control, it is what the database is
built for.

**The judgement to carry into an interview:** the right answer to "how do I insert five
million rows fast" is not "tune the batch size". It is "why is this going through an ORM at
all". Knowing batching is what lets you make that argument with a number attached instead of
a preference. And knowing batching is still required for the case `COPY` cannot serve — the
next section.

### The second path: the nightly settlement writer, where `COPY` does not apply

Every night, ~200,000 `Payment` rows move from `AUTHORIZED` to `SETTLED`, each with a
settlement reference from the acquirer's file. These are **updates**, not inserts, on rows
that carry a `@Version` for optimistic locking (Topic 52).

```java
@Service
public class SettlementWriter {

    private static final int CHUNK = 500;

    @PersistenceContext private EntityManager em;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void settleChunk(List<SettlementRecord> records) {
        for (int i = 0; i < records.size(); i++) {
            SettlementRecord r = records.get(i);
            Payment p = em.find(Payment.class, r.paymentId());
            if (p == null || p.getStatus() != PaymentStatus.AUTHORIZED) {
                continue;                       // idempotent: already settled or unknown
            }
            p.settle(r.externalRef(), r.settledAt());   // dirty checking, no save() — Topic 48
            if ((i + 1) % CHUNK == 0) { em.flush(); em.clear(); }
        }
        em.flush();
        em.clear();
    }
}
```

Three things differ from the insert path and each is worth knowing:

1. **`order_updates=true` matters more here than `order_inserts` did.** Updates sorted by
   entity type *and id* mean concurrent settlement workers touch rows in the same order,
   which reduces the deadlock rate (Topic 52, Trap 5). That is a correctness benefit falling
   out of a performance setting.
2. **`batch_versioned_data` must be on** for these updates to batch at all. Optimistic
   locking needs the per-statement row count to detect a stale version; batching returns an
   array of counts, and Hibernate needs the driver to report real values rather than
   `Statement.SUCCESS_NO_INFO`. The Postgres driver does. With the property off, versioned
   updates are not batched and your 200k updates are 200k round trips.
3. **`em.find` per row is a read round trip.** Batching does nothing for reads. If the read
   is the bottleneck, the fix is a single query loading the whole chunk by id
   (`where id in (:ids)`) and working from that — which is Topic 50's thinking applied to a
   write path.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `hibernate.jdbc.batch_size` does nothing because the id generator is `IDENTITY`

**This is the topic's headline. Everything else is a supporting act.**

**Wrong:**

```java
@Entity
public class OrderLine {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)      // <-- the actual bug
    private Long id;
    ...
}
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50                                     # <-- has no effect
        order_inserts: true                                  # <-- has no effect either
```

**Exact symptom.** Nothing changes. Not "improves slightly" — **nothing changes at all**.

Concretely, on the `orderflow` seed loader inserting 50,000 `OrderLine` rows:

- `Statistics.getPrepareStatementCount()` is **~50,000**, identical before and after adding
  `batch_size`.
- Wall-clock time is identical within noise.
- Postgres `pg_stat_statements` shows the `insert into order_line ...` entry with `calls`
  ≈ 50,000.
- With datasource-proxy logging, every logged statement shows `batchSize=1`.
- There is **no warning in the log**, at any level, saying batching was disabled.

The second symptom, which is the one that confirms the diagnosis in five seconds: with
`logging.level.org.hibernate.SQL=DEBUG`, the `insert` statement appears in the log
**immediately after each `persist()` call**, interleaved with your own log lines — not in a
burst at flush. With `SEQUENCE`, nothing appears until the flush.

**Root cause.** The persistence context is keyed by entity identifier. With `IDENTITY`, the
identifier does not exist until the row does. Hibernate must therefore execute the `INSERT`
synchronously inside `persist()` and read the generated key back, one row at a time. A JDBC
batch requires every parameter — including the primary key — to be bound before
`addBatch()`, which is impossible. Hibernate detects this and disables insert batching for
the entity. It does so silently, because from its perspective you asked for an incompatible
combination and it honoured the stricter half.

**Fix:**

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_line_seq")
@SequenceGenerator(name = "order_line_seq",
                   sequenceName = "order_line_seq",
                   allocationSize = 50)
private Long id;
```

```sql
create sequence order_line_seq start with 1 increment by 50;   -- MUST match allocationSize
```

**Migrating an existing table** is the part that takes planning, and you should be able to
describe it:

1. Create the sequence, seeded above the current maximum id:
   `select setval('order_line_seq', (select coalesce(max(id),0) + 1000 from order_line));`
2. Set `INCREMENT BY` to your `allocationSize`.
3. Change the entity annotation and deploy.
4. Drop the column's identity/default **last**, only once nothing writes without an id.

Steps 3 and 4 must not be the same deploy, because during the rollout both the old
(`IDENTITY`) and new (`SEQUENCE`) code are live. This is a Topic 129-shaped migration
problem, and getting the ordering wrong produces duplicate keys in production.

**The regression barrier** — and this is the part that makes the fix stick:

```java
@Test
@Transactional
void inserting_1000_lines_uses_far_fewer_than_1000_statements() {
    Statistics stats = sessionFactory.getStatistics();
    stats.clear();

    for (int i = 1; i <= 1000; i++) {
        em.persist(new OrderLine(...));
        if (i % 50 == 0) { em.flush(); em.clear(); }
    }
    em.flush();

    assertThat(stats.getEntityInsertCount()).isEqualTo(1000);
    assertThat(stats.getPrepareStatementCount())
        .as("batching must be active; ~1000 statements means IDENTITY or a lost setting")
        .isLessThan(100);
}
```

That assertion fails the moment someone changes the generation strategy, moves a property to
the wrong prefix, or adds an entity that breaks the batch. There is **no other automated
signal** — the code compiles, the tests pass, and the only symptom is a number in a
dashboard.

---

### Trap 2 — the persistence context grows linearly without `flush()` + `clear()`

**Wrong:**

```java
@Transactional
public void load(List<OrderLine> lines) {          // 5,000,000 lines
    for (OrderLine line : lines) {
        em.persist(line);
    }
}                                                   // one flush, at commit
```

**Exact symptom.** Not an immediate crash — a **slow collapse**, which is what makes it hard
to diagnose:

1. The first 50,000 rows load at a reasonable rate.
2. Throughput degrades steadily. By 500,000 it is a fraction of the starting rate.
3. The GC log (`-Xlog:gc*`) shows full collections every few seconds, each reclaiming almost
   nothing — the classic "GC thrash" signature: high pause frequency, near-zero reclaimed
   bytes, live set climbing monotonically.
4. Eventually: `java.lang.OutOfMemoryError: Java heap space`, with a stack trace pointing at
   whatever unlucky allocation happened last — often something completely unrelated, which
   sends people investigating the wrong code.
5. A heap dump shows `StatefulPersistenceContext` (or its maps) retaining the vast majority
   of the live heap.

**Root cause.** Every `persist()` adds the entity to the persistence context and records a
snapshot of its field values so dirty checking can diff at flush (Topic 48). Nothing is
released until `clear()` or transaction end. The persistence context is a **cache with no
eviction policy**, holding objects you will never read again.

**Fix:**

```java
@Transactional
public void load(List<OrderLine> lines) {
    for (int i = 0; i < lines.size(); i++) {
        em.persist(lines.get(i));
        if ((i + 1) % 50 == 0) {          // == hibernate.jdbc.batch_size
            em.flush();                    // emit the accumulated batch
            em.clear();                    // detach everything; free the snapshots
        }
    }
    em.flush();
    em.clear();
}
```

**Three things to get right, all of which people get wrong:**

1. **Order matters.** `clear()` before `flush()` discards the pending work — the rows are
   never written and nothing tells you.
2. **`clear()` detaches everything, including entities you still hold references to.**
   Touching a lazy association on one afterwards throws `LazyInitializationException`
   (Topic 49). If you need the parent after a clear, re-`find()` it or restructure so the
   flush boundary does not fall inside a parent/child group — see Example 2, decision 3.
3. **`flush()` + `clear()` does not bound the transaction.** You still hold one connection
   and accumulate WAL for the whole run. Chunked transactions are a separate fix for a
   separate problem, and you usually need both.

**How to confirm it is this and not something else:**

```bash
# while the load is running
jcmd <pid> GC.heap_info
jcmd <pid> GC.class_histogram | head -30
```

Look for a large and growing count of your entity class and of `Object[]`. Then take a heap
dump (`jcmd <pid> GC.heap_dump /tmp/load.hprof`) and check the retained size of the
persistence context in Eclipse MAT. That is Topic 79's workflow; this is one of its cleanest
applications.

---

### Trap 3 — `allocationSize` does not match the database sequence's `INCREMENT BY`

**Wrong:**

```java
@SequenceGenerator(name = "order_line_seq", sequenceName = "order_line_seq",
                   allocationSize = 50)
```

```sql
create sequence order_line_seq start with 1 increment by 1;    -- MISMATCH
```

**Exact symptom.** Intermittent, environment-dependent, and it will not reproduce on your
laptop:

```
org.postgresql.util.PSQLException: ERROR: duplicate key value violates unique
  constraint "order_line_pkey"
  Detail: Key (id)=(<n>) already exists.
```

*illustration of the message shape, not captured output*

It fires when two id blocks overlap on a row that actually gets inserted. On a single
instance with light traffic that can take days. With **two replicas** — which is every real
deployment — both instances call `nextval`, get values one apart, each reserves 50 ids
starting from there, and the blocks overlap by 49. The failure rate is then high and the
symptom looks like a race condition in your business code.

The confirmation query, which settles it immediately:

```sql
select sequencename, increment_by, last_value from pg_sequences
where sequencename = 'order_line_seq';
```

If `increment_by` is not equal to your `allocationSize`, that is the bug. No further
investigation is needed.

**Root cause.** The `pooled` optimiser interprets the value returned by `nextval` as the
**top of a reserved block** of `allocationSize` values, and assumes it may use every value
below it down to `value - allocationSize + 1`. That assumption is only safe if the sequence
advances by `allocationSize` on each call. When it advances by 1, the "reserved" block
overlaps the next caller's block almost entirely.

**Fix:**

```sql
create sequence order_line_seq start with 1 increment by 50;
-- or, for an existing sequence:
alter sequence order_line_seq increment by 50;
```

**And the process fix, which matters more than the SQL:** put the two values next to each
other and comment the coupling.

```sql
-- V3__order_line_sequence.sql
-- INCREMENT BY must equal OrderLine.id's @SequenceGenerator(allocationSize).
-- Currently 50. Change both or neither.
create sequence order_line_seq start with 1 increment by 50;
```

**The escape hatch if you cannot change the DDL:** set `allocationSize = 1`. It is correct
and safe. It also costs one `nextval` round trip per row, which puts you back roughly where
`IDENTITY` had you for the sequence half of the work — though inserts still batch, so it is
strictly better than `IDENTITY`. Know that this is a real fallback and know what it costs.

Hibernate can log a warning when it detects a mismatch during schema validation. Do not build
a process on noticing it: it is one line among hundreds at startup, it is not an error, and
it does not appear at all if `ddl-auto` is `none`, which it should be in production.

---

### Trap 4 — interleaved entity types silently reduce the batch size to almost nothing

**Wrong:**

```java
for (int i = 0; i < 10_000; i++) {
    Order order = new Order(...);
    em.persist(order);                              // table: orders
    for (int j = 0; j < 5; j++) {
        em.persist(new OrderLine(order, ...));      // table: order_line
    }
    if (i % 100 == 0) { em.flush(); em.clear(); }
}
```

with `batch_size: 50` and **`order_inserts` not set** (default `false`).

**Exact symptom.** Batching is technically working — the ids are sequence-generated, the
property is set correctly — and yet:

- `Statistics.getPrepareStatementCount()` is far higher than `total_rows / batch_size`. For
  60,000 rows at batch size 50 you expect ~1,200; you observe several thousand.
- With datasource-proxy, the logged `batchSize` values are `1` and `5`, alternating, never
  `50`.
- Wall-clock improvement over the unbatched version is real but disappointing — maybe 2×
  where you expected 20× — which is the worst outcome, because it looks like batching just
  is not very effective and people stop investigating.

**Root cause.** A JDBC batch belongs to one `PreparedStatement` and therefore one SQL string
and one table. Hibernate's action queue preserves the order you called `persist()` in, so the
queue reads `orders, order_line×5, orders, order_line×5, …`. Every switch between the two
statements forces `executeBatch()` on the current one. Your effective batch sizes are the
run lengths in your call pattern: 1 and 5.

**Fix:**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        order_inserts: true
        order_updates: true
```

With `order_inserts=true`, Hibernate sorts the pending inserts by entity type before
emitting, producing `orders×N` then `order_line×5N`, and the batches fill.

**Two notes on the fix:**

- **`order_updates` deserves the same treatment for update-heavy paths**, and it additionally
  sorts by id — which reduces deadlocks between concurrent writers, because they touch rows
  in the same order (Topic 52, Trap 5). A performance flag buying a correctness property is
  rare enough to remember.
- **The alternative fix is to restructure the loop**: persist all the orders, then all the
  lines. This works without the property and is sometimes clearer. It also means you must
  keep the orders around until you build the lines, which reintroduces the memory problem
  from Trap 2. The property is usually the better trade.

---

### Trap 5 — the property is in the wrong namespace, and nothing tells you

**Wrong — any of these:**

```yaml
spring:
  jpa:
    hibernate:
      jdbc:
        batch_size: 50            # WRONG namespace: spring.jpa.hibernate.* is Boot's, not Hibernate's
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc.batchSize: 50        # WRONG name: it is batch_size, snake_case
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        order-inserts: true       # WRONG: Hibernate wants order_inserts, not kebab-case
```

**Exact symptom.** The application starts cleanly. No warning, no error, no hint. The
setting simply does not exist as far as Hibernate is concerned, and statement counts are
identical to having set nothing. You spend an afternoon concluding that batching "doesn't
work on our setup".

This trap is nastier than it looks because **Boot's relaxed binding does not apply here**.
Elsewhere in Spring Boot, `my-property`, `myProperty` and `MY_PROPERTY` all bind to the same
thing. Under `spring.jpa.properties.*`, the keys are passed to Hibernate **verbatim**, and
Hibernate wants its own exact names. Your instincts from every other Boot property are
actively wrong here.

**Root cause.** `spring.jpa.properties.*` is a pass-through map into the JPA
`PersistenceUnitInfo` properties. Hibernate reads the keys it recognises and **ignores
everything else without comment** — which is the correct behaviour for a properties map that
may legitimately contain vendor-specific keys, and is exactly why a typo is invisible.

**Fix — verify, do not assume.** Three ways, in increasing order of confidence:

1. **Read the effective configuration back at startup:**

```java
@Component
class HibernateSettingsProbe implements ApplicationRunner {
    private final EntityManagerFactory emf;
    HibernateSettingsProbe(EntityManagerFactory emf) { this.emf = emf; }

    @Override public void run(ApplicationArguments args) {
        var props = emf.getProperties();
        for (String key : List.of("hibernate.jdbc.batch_size",
                                  "hibernate.order_inserts",
                                  "hibernate.order_updates",
                                  "hibernate.batch_versioned_data",
                                  "hibernate.generate_statistics")) {
            System.out.println("HIBERNATE-SETTING " + key + " = " + props.get(key));
        }
    }
}
```

2. **Assert on behaviour**, with the statement-count test from Trap 1. A property that took
   effect changes a number; a property that did not, does not.

3. **Watch it at the wire** with datasource-proxy, which logs the actual batch size per
   execution. If the property is not applied, every logged batch size is 1.

**The general lesson, which applies well beyond Hibernate:** a configuration setting you have
not *observed changing something* is a setting you have not made. This is the same discipline
as Topic 42's condition-evaluation report and Topic 50's query-count assertion. Configuration
is a hypothesis until a counter moves.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM, no Postgres and no captured output,
and I will not print numbers and call them measured. What follows is the exact configuration,
the exact command, what to look for, and how to read every result you might get.

### Setup

`orderflow` is on Postgres from Topic 47, and this topic is **untestable on H2** — H2's
sequence behaviour, batch reporting and `reWriteBatchedInserts` support all differ. Use
Testcontainers (Topic 61) or plain Docker.

```bash
mkdir -p ~/java-lab/53 && cd ~/java-lab/53
java --version           # expect 21 or 25

docker run --name pg-53 -e POSTGRES_PASSWORD=orderflow \
  -e POSTGRES_DB=orderflow -p 5432:5432 -d postgres:17

curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,postgresql \
  -d javaVersion=21 \
  -d groupId=com.orderflow -d artifactId=batch-lab \
  -d type=maven-project -o batch-lab.zip && unzip batch-lab.zip -d batch-lab
cd batch-lab
```

`src/main/resources/application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orderflow
    username: postgres
    password: orderflow
  jpa:
    hibernate:
      ddl-auto: none
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
        generate_statistics: true
  sql:
    init:
      mode: always
logging:
  level:
    org.hibernate.stat: DEBUG
```

Add the `schema.sql` and the two entities from Example 1.

### Instruments — what each one actually tells you

| Instrument | Tells you | Does **not** tell you |
|---|---|---|
| `Statistics.getPrepareStatementCount()` | how many JDBC statements Hibernate prepared | how many network round trips happened |
| `Statistics.getEntityInsertCount()` | how many entities were inserted | anything about batching |
| `logging.level.org.hibernate.SQL=DEBUG` | the SQL text, and **when** it was emitted | the batch size — this is its key limitation |
| datasource-proxy / p6spy | the actual batch size per execution | — this is the right tool for this topic |
| `pg_stat_statements.calls` | how many times Postgres executed each statement | how they were grouped on the wire |

**`org.hibernate.SQL=DEBUG` is a poor batching instrument and it is the one everyone reaches
for.** It shows statement text, not grouping. Its genuine value here is *timing*: with
`IDENTITY` the insert appears immediately after each `persist()`; with `SEQUENCE` nothing
appears until flush. That difference alone diagnoses Trap 1.

Add datasource-proxy for the real answer:

```xml
<dependency>
  <groupId>net.ttddyy</groupId>
  <artifactId>datasource-proxy</artifactId>
  <version><!-- pick the current release; not managed by the Boot BOM --></version>
</dependency>
```

> **Honest note:** datasource-proxy is not in the Spring Boot BOM, so you must supply a
> version (Topic 32). Check the current release rather than copying a version from any
> document, including this one.

### Proof 1 — confirm the settings actually reached Hibernate

Add the `HibernateSettingsProbe` from Trap 5.

```bash
./mvnw spring-boot:run 2>&1 | grep HIBERNATE-SETTING
```

| What you see | What it means |
|---|---|
| `hibernate.jdbc.batch_size = 50` and `order_inserts = true` | The properties reached Hibernate. You may now interpret statement counts. |
| `hibernate.jdbc.batch_size = null` | **Trap 5.** Wrong prefix or wrong spelling. Everything downstream in this drill is meaningless until you fix it. |
| The values are present but statement counts do not change | The property took effect and something else is disabling batching — go to Proof 2 and check the id generator. |

**Do this proof first, every time.** Interpreting a statement count when the setting never
applied is how people conclude batching does not work.

### Proof 2 — the `IDENTITY` versus `SEQUENCE` statement count

Run both `BatchProbe` methods from Example 1 with `rows = 1000, batchSize = 50`.

```bash
./mvnw spring-boot:run 2>&1 | grep -E "IDENTITY|SEQUENCE"
```

| What you see | What it means |
|---|---|
| `IDENTITY rows=1000 entityInserts=1000 prepareStatements=~1000` | **Trap 1, confirmed.** One statement per row. `batch_size=50` had no effect. |
| `SEQUENCE rows=1000 entityInserts=1000 prepareStatements=~40` | Batching is active. Roughly `1000/50` insert batches plus `1000/50` `nextval` calls, give or take how Hibernate counts allocations. |
| `SEQUENCE` also shows ~1000 | Either the property did not apply (Proof 1), or `allocationSize` is 1 so you are paying a `nextval` round trip per row, or something else interleaved. |
| `SEQUENCE` shows ~1020 and you expected ~20 | The extra ~1000 are `nextval` calls. Set `allocationSize = 50` and re-run — this is the "worse before better" step from the arithmetic table. |
| `entityInserts` is not 1000 in either case | Your loop or your flush discipline is wrong; fix that before reading anything else. |

### Proof 3 — see the timing difference in the SQL log

```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG
```

Add a marker print inside the loop:

```java
System.out.println("--- persisted row " + i);
```

| What you see | What it means |
|---|---|
| `insert into ...` lines interleaved between every `--- persisted row <n>` | `IDENTITY`. The insert executes inside `persist()`. This is the five-second diagnosis. |
| All the `--- persisted row` lines, then a burst of `insert into ...` at the flush | `SEQUENCE`. Inserts are queued and emitted at flush. Batching is possible. |
| A burst of inserts, but far more of them than `rows / batch_size` | Batching is partially working — check `order_inserts` (Trap 4) and whether more than one entity type is interleaved. |

### Proof 4 — observe the actual batch size at the wire

With datasource-proxy configured to log batch information:

```bash
./mvnw spring-boot:run 2>&1 | grep -oE "batch=true, batchSize=[0-9]+" | sort | uniq -c
```

*The grep pattern above matches datasource-proxy's default log format; adjust it to whatever
your configured formatter emits.*

| What you see | What it means |
|---|---|
| Almost all lines `batchSize=50` | Batching is fully effective. This is the goal state. |
| All lines `batchSize=1` | No batching. `IDENTITY`, or the property did not apply. |
| A mix of `batchSize=1` and `batchSize=5` | **Trap 4.** Interleaved entity types with `order_inserts=false`. |
| `batchSize` mostly 50 with occasional smaller values | Normal — the last partial batch of each flush, and any type transition. Nothing to fix. |

### Proof 5 — the Postgres side

```sql
create extension if not exists pg_stat_statements;   -- requires shared_preload_libraries
select calls, rows, query
from pg_stat_statements
where query like 'insert into line_%'
order by calls desc;
```

| What you see | What it means |
|---|---|
| `calls` ≈ number of rows, `rows` ≈ number of rows | One execution per row. Either no batching, or batching without `reWriteBatchedInserts` — Postgres still executes the plan once per parameter set even inside a batch. |
| `calls` ≈ rows/batch_size, `rows` ≈ number of rows | `reWriteBatchedInserts=true` collapsed each batch into one multi-row `INSERT`. Fewer parses and plans server-side. |
| The extension is unavailable | Fall back to `log_statement = 'all'` in `postgresql.conf` and count lines in the container log. Noisy, but definitive. |

**This is the only instrument that measures the database's view.** Hibernate's counters
measure Hibernate's view, and the two differ in exactly the way `reWriteBatchedInserts`
exploits.

### Proof 6 — watch the persistence context grow

```bash
./mvnw spring-boot:run \
  -Dspring-boot.run.jvmArguments="-Xmx512m -Xlog:gc*:file=/tmp/load-gc.log:time,uptime,level,tags"
```

Run the loader with the `flush()`/`clear()` lines commented out, then restored.

```bash
jcmd <pid> GC.class_histogram | head -20
grep -c "Pause Full" /tmp/load-gc.log
```

| What you see | What it means |
|---|---|
| Without clear: instance count of your entity class grows monotonically; frequent `Pause Full` entries reclaiming little | **Trap 2, reproduced.** The persistence context is a cache with no eviction. |
| Without clear: `OutOfMemoryError: Java heap space` | Same trap, further along. Note where it happened relative to the row count — that ratio is your per-entity memory cost. |
| With clear: instance count oscillates around the batch size; few or no full collections | Correct. Memory is now bounded by `batch_size`, not by row count. |
| With clear, memory still grows | Something else holds references — commonly the `List` you are iterating, or a parent entity accumulating children in a `@OneToMany`. `clear()` cannot free what your own code retains. |

---

## Failure drill

**Mandatory.** Do not read past the interpretation tables until you have produced each
phase's numbers yourself and written them down. The value is not the knowledge. It is the
memory of setting a property, restarting, and seeing the count not move.

### The scenario

Insert 50,000 `OrderLine` rows into Postgres. Count statements. Change one annotation. Count
again.

### Phase 0 — the harness

```java
package com.orderflow.drill;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class DrillWriter {

    @PersistenceContext private EntityManager em;
    private final SessionFactory sessionFactory;

    public DrillWriter(SessionFactory sessionFactory) { this.sessionFactory = sessionFactory; }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void insertChunk(int from, int to, int flushEvery) {
        for (int i = from; i < to; i++) {
            em.persist(newLine(i));
            if ((i - from + 1) % flushEvery == 0) { em.flush(); em.clear(); }
        }
        em.flush();
        em.clear();
    }

    public Statistics stats() { return sessionFactory.getStatistics(); }
}
```

```java
@Component
public class DrillRunner implements ApplicationRunner {

    private static final int ROWS = 50_000;
    private static final int CHUNK = 5_000;
    private static final int FLUSH_EVERY = 50;

    private final DrillWriter writer;

    public DrillRunner(DrillWriter writer) { this.writer = writer; }

    @Override public void run(ApplicationArguments args) {
        var stats = writer.stats();
        stats.clear();

        for (int start = 0; start < ROWS; start += CHUNK) {
            writer.insertChunk(start, Math.min(start + CHUNK, ROWS), FLUSH_EVERY);
        }

        System.out.println("DRILL rows=" + ROWS
            + " entityInserts=" + stats.getEntityInsertCount()
            + " prepareStatements=" + stats.getPrepareStatementCount()
            + " flushes=" + stats.getFlushCount()
            + " transactions=" + stats.getSuccessfulTransactionCount());
    }
}
```

Note the structure: `DrillRunner` is a separate bean from `DrillWriter`, so
`@Transactional(REQUIRES_NEW)` on `insertChunk` goes through the proxy (Topic 40). Chunked
transactions keep the WAL and lock windows bounded. `ApplicationRunner` rather than
`@PostConstruct`, for the Topic 37 reason.

**Before running anything, write down your prediction for `prepareStatements` in each of the
six phases below.** Predicting first is what converts the drill from an observation into a
model correction.

### Phase 1 — `IDENTITY` with batching "on"

Entity: `GenerationType.IDENTITY`. Config: `batch_size: 50`, `order_inserts: true`.

```bash
./mvnw spring-boot:run 2>&1 | grep DRILL
```

| What you see | What it means |
|---|---|
| `prepareStatements` ≈ 50,000 | **The drill has fired.** You set `batch_size=50` and got one statement per row. Write the number down. This is the number that does not move in Phase 2. |
| `prepareStatements` far below 50,000 | Check the entity really is `IDENTITY` and that a stale class is not on the classpath (`./mvnw clean spring-boot:run`). |
| `entityInserts` is not 50,000 | Your loop is wrong. Fix it before continuing; every later phase compares against this. |

Now **change nothing but `batch_size`** — set it to 500, restart, re-run.

| What you see | What it means |
|---|---|
| `prepareStatements` unchanged at ≈ 50,000 | **This is the moment the topic exists for.** You changed the setting by 10×. Nothing moved. No warning was logged. Sit with that for a second. |

### Phase 2 — `SEQUENCE` with `allocationSize = 1`

Switch the entity to `GenerationType.SEQUENCE`, `allocationSize = 1`, and create the sequence
with `increment by 1`.

| What you see | What it means |
|---|---|
| `prepareStatements` ≈ 51,000 — **higher** than Phase 1 | Expected, and it is the step that makes people give up. Inserts now batch (≈1,000 statements) but you added 50,000 `nextval` round trips. The arithmetic table predicted this. |
| `prepareStatements` ≈ 1,000 | Your Hibernate version is not counting `nextval` as a prepared statement. The counter is a proxy, not ground truth — go to Proof 5 and read `pg_stat_statements` instead. |
| Wall-clock time roughly the same as Phase 1 | Consistent with the above. You have moved the round trips, not removed them. |

### Phase 3 — `SEQUENCE` with `allocationSize = 50`, pooled

`allocationSize = 50` on the entity, `increment by 50` on the sequence. **Change both.**

| What you see | What it means |
|---|---|
| `prepareStatements` ≈ 2,000 (≈1,000 batches + ≈1,000 allocations) | **Batching is working.** A 25× reduction in statements from Phase 1. This is the headline result. |
| `prepareStatements` ≈ 51,000 | The sequence's `INCREMENT BY` was not changed, or `allocationSize` did not apply. Check `select increment_by from pg_sequences where sequencename='...'`. |
| `duplicate key value violates unique constraint` | **Trap 3.** The two values disagree. This is the good version of that failure — in a lab, on purpose. |
| Wall-clock time drops sharply | Expected, and see the Measurement section before you quote the number to anyone. |

Now push `allocationSize` to 500 (and the sequence with it) and re-run.

| What you see | What it means |
|---|---|
| `prepareStatements` ≈ 1,100 | Allocation round trips drop from ~1,000 to ~100. Diminishing returns are visible: the batches themselves are now the floor. |
| No further wall-clock improvement | You have moved the bottleneck off id allocation. Further gains must come from the batch mechanism or the database, not from the sequence. |

### Phase 4 — break it with interleaving, then fix it with `order_inserts`

Change the loop to persist an `Order` and five `OrderLine`s per iteration. Set
`order_inserts: false`. Run. Then set it to `true` and run again.

| What you see | What it means |
|---|---|
| `order_inserts=false`: `prepareStatements` far above `rows / batch_size` | **Trap 4, reproduced.** Every entity-type transition ends a batch. |
| `order_inserts=true`: the count drops sharply | The action queue was sorted by type before emission. Same code, one property. |
| datasource-proxy shows `batchSize=1` and `batchSize=5` in the first run, `batchSize=50` in the second | The cleanest possible evidence. This is why Proof 4 is worth setting up. |

### Phase 5 — `reWriteBatchedInserts`

Add `?reWriteBatchedInserts=true` to the JDBC URL. Re-run Phase 3's configuration.

| What you see | What it means |
|---|---|
| Hibernate's `prepareStatements` **unchanged**, but `pg_stat_statements.calls` drops by roughly `batch_size` | Correct. The rewrite happens in the driver, below Hibernate. Each batch became one multi-row `INSERT`, so Postgres parses and plans once instead of `batch_size` times. |
| No change in `pg_stat_statements` either | The statements were not rewriteable, or the parameter did not apply. Confirm the URL reached the driver: `select * from pg_stat_activity` will not tell you, but printing `dataSource.getConnection().getMetaData().getURL()` will. |
| Wall-clock improves further | Expected. Note that this is the one lever in the whole drill that reduces *server* CPU rather than round trips. |

### Phase 6 — remove `flush()`/`clear()` and watch it die

Comment out the flush/clear lines. Set `-Xmx512m`. Run with GC logging.

| What you see | What it means |
|---|---|
| Throughput degrades progressively; frequent full GCs reclaiming little; eventually `OutOfMemoryError` | **Trap 2, reproduced.** Note the row count at which it collapsed — that is your per-entity memory budget, empirically. |
| It completes without OOM | Your chunk size is small enough that each transaction's context stays bounded. Raise `CHUNK` to 50,000 (one transaction) and try again — that is the version people actually write. |

### Phase 7 — the regression barrier

The drill is not finished until the fix cannot silently regress:

```java
@Test
@Transactional
void order_line_inserts_are_batched() {
    Statistics stats = sessionFactory.getStatistics();
    stats.clear();

    for (int i = 1; i <= 1000; i++) {
        em.persist(newLine(i));
        if (i % 50 == 0) { em.flush(); em.clear(); }
    }
    em.flush();

    assertThat(stats.getEntityInsertCount()).isEqualTo(1000);
    assertThat(stats.getPrepareStatementCount())
        .as("~1000 statements means batching is off: check the id generator, "
          + "the property namespace, and order_inserts")
        .isLessThan(120);
}
```

Run it against Testcontainers Postgres, not H2 (Topic 61). Then deliberately change the
entity back to `IDENTITY` and confirm the test **fails**. A regression test you have not seen
fail is a regression test you do not have.

### What the drill proves

1. A configuration property can be applied correctly and do absolutely nothing, with no
   diagnostic anywhere.
2. The prerequisite for batching is a **modelling decision** (the id generator), not a
   tuning knob — which is why it is not in the tuning documentation you would have read.
3. The intermediate step (Phase 2) is *worse* than the starting point, which is exactly the
   shape of change that makes teams abandon a correct direction.
4. Statement counts are the instrument. Wall-clock time is the outcome you care about and
   the number you must not trust naively — which is the next section.

---

## Measurement

### The counters, and what each one actually counts

```java
Statistics stats = sessionFactory.getStatistics();   // requires generate_statistics=true

stats.getEntityInsertCount();           // entities inserted. NOT statements.
stats.getEntityUpdateCount();
stats.getEntityDeleteCount();
stats.getPrepareStatementCount();       // JDBC statements PREPARED. The key number here.
stats.getFlushCount();                  // how many flushes occurred
stats.getSuccessfulTransactionCount();
stats.getQueryExecutionCount();         // reads, for the settlement path's find()
stats.getSecondLevelCacheHitCount();    // Topic 51
```

**The metric for this topic is `getPrepareStatementCount()` relative to
`getEntityInsertCount()`.**

```
statements / entities  ≈ 1.0    -> no batching at all
statements / entities  ≈ 1/batch_size + allocation overhead  -> batching working
```

**Three honest limitations of this counter**, which you should state before quoting it:

1. **It counts statements prepared, not network round trips.** A batch of 50 is one round
   trip but the counter's relationship to that is a Hibernate implementation detail. Use it
   as a *ratio*, comparing configurations, not as an absolute round-trip count.
2. **`generate_statistics=true` has a cost.** It is fine in a drill and in a seed profile. It
   is not fine left on in the profile serving 1,200 rps.
3. **It says nothing about what Postgres did.** `reWriteBatchedInserts` changes the
   database's work and not this counter at all (Phase 5). For the database's view you need
   `pg_stat_statements`.

The authoritative pairing is therefore: **Hibernate `Statistics` for "did my configuration
take effect", `pg_stat_statements` for "what did the database actually execute", and
datasource-proxy for "what was the batch size on the wire".** Three instruments, three
questions, and none of them substitutes for another.

### Why a naive `System.nanoTime()` timing is wrong here

You are about to write this. Do not.

```java
long t0 = System.nanoTime();
loader.load(50_000);
long t1 = System.nanoTime();
System.out.println("took " + (t1 - t0) / 1_000_000 + " ms");     // WRONG
```

It is wrong for at least seven independent reasons, and they push in different directions so
they do not cancel out:

1. **JIT compilation state.** The first thousand iterations run interpreted; the method is
   compiled somewhere in the middle, possibly deoptimised and recompiled. You are timing a
   mixture of interpreted and compiled code in unknown proportions. This is Topic 74's
   territory.
2. **Class loading.** The first pass through Hibernate's insert path loads and initialises
   dozens of classes. That cost lands entirely in your first measurement.
3. **Connection pool warm-up.** HikariCP establishes connections lazily. The first
   transaction may pay a TCP handshake, TLS negotiation and Postgres authentication that
   later ones do not (Topic 109).
4. **Postgres plan caching.** The first execution of a statement parses and plans; later ones
   may reuse a cached plan. Your first batch is more expensive per row than your thousandth.
5. **OS page cache and index state.** Inserting into an empty table with a small index is
   nothing like inserting into a table with 5 million rows and a warm index. The rate changes
   *during* the run, so a single total tells you nothing about the shape.
6. **Autovacuum and WAL checkpoints.** Postgres does background work on its own schedule.
   A checkpoint landing inside your measurement window adds hundreds of milliseconds that
   have nothing to do with your code.
7. **One sample is not a measurement.** You have a single number with no distribution, no
   variance, and no way to know whether a 15% "improvement" is real.

**What to do instead, in order of how much you should trust it:**

- **For "did my configuration take effect": statement counts.** Deterministic, immune to all
  seven problems above, and assertable in a test. This is why the drill is built around
  counts and not times.
- **For "how long does the seed load take": run it end-to-end, repeatedly, against a
  realistic dataset size, from a cold database each time, and report a distribution.** Drop
  the first run entirely. Report median and spread, never a single number.
- **For the serving path: the Topic 65 load harness**, with an open-model arrival rate and
  proper percentiles — not a stopwatch around a loop.

**Forward reference: Topic 77 (JMH)** explains in full why a `System.nanoTime()` loop
measures dead-code elimination, constant folding, on-stack replacement and cold-JIT state in
unknown proportions, and how a correct harness defeats each.

**And the honest extension of that argument, which Topic 52 also makes:** JMH is the right
tool for JVM-level microbenchmarks and it is **not** the right tool for this. A benchmark of
database write throughput is dominated by the database, the network and the operating
system — none of which JMH's warm-up, forking and `Blackhole` machinery addresses. For this
topic, JMH would give you a beautifully rigorous measurement of the wrong thing. The right
instrument is: **statement counts for correctness of configuration, and a repeated
end-to-end run with a reported distribution for time.**

### What to record for the Topic 65 baseline

When you commit the seed loader's numbers to `/docs/java/baselines/`, record:

| Field | Why |
|---|---|
| rows loaded, by table | the denominator for everything else |
| `batch_size`, `allocationSize`, `order_inserts`, `reWriteBatchedInserts` | the configuration that produced the number |
| `entityInsertCount` and `prepareStatementCount` | the deterministic, reproducible evidence that batching was active |
| median and spread of wall-clock across ≥5 cold runs | the outcome, honestly reported |
| Postgres version, container CPU and memory limits, whether the volume is a bind mount or a Docker volume | I/O characteristics dominate, and bind mounts on macOS are dramatically slower |
| JVM heap size and collector | Trap 2's behaviour depends on both |

A number without that context is not reproducible, which by the Topic 65 gate rule means it
does not count.

---

## Practice exercises

### 1 — easy: predict the statement counts

For a loop persisting 10,000 `OrderLine` entities with `flush()`/`clear()` every 50, predict
`getPrepareStatementCount()` for each configuration **before running anything**. Write the
numbers down.

| # | Configuration | Prediction | Actual |
|---|---|---|---|
| 1 | `IDENTITY`, `batch_size` unset | | |
| 2 | `IDENTITY`, `batch_size=50` | | |
| 3 | `IDENTITY`, `batch_size=500` | | |
| 4 | `SEQUENCE` `allocationSize=1`, `batch_size=50` | | |
| 5 | `SEQUENCE` `allocationSize=50`, `batch_size=50` | | |
| 6 | `SEQUENCE` `allocationSize=50`, `batch_size=50`, `reWriteBatchedInserts=true` | | |

**Done when:** you predicted rows 2 and 3 as identical to row 1, predicted row 4 as *worse*
than rows 1–3, and can explain row 6 being identical to row 5 in Hibernate's counter while
differing in `pg_stat_statements`.

### 2 — medium: combines Topics 40, 48, 52 and 55

Build the `orderflow` seed loader from Example 2 and answer each of these with evidence, not
reasoning alone:

1. Put `@Transactional(REQUIRES_NEW)` on `writeChunk` and call it from a **sibling method in
   the same class**. Predict the effect on transaction count and on memory, then measure both
   with `getSuccessfulTransactionCount()` and a heap histogram. Explain the result in Topic
   40's terms.
2. Set `CHUNK` to the full row count so there is one transaction. Measure how long a single
   HikariCP connection is held (Topic 55) and what `pg_stat_activity` shows for that backend's
   `state` and `xact_start`. State what would happen to the serving path if this ran against
   production.
3. Add a `@Version` field to `OrderLine`, set `batch_versioned_data=false`, and measure the
   statement count for a bulk **update** path. Then set it to `true` and re-measure. Explain
   the difference in terms of what optimistic locking needs from the batch result array
   (Topic 52).
4. With `order_updates=true`, argue — with reference to Topic 52's Trap 5 — why sorting
   updates by id reduces deadlocks between two concurrent settlement workers.

**Done when:** you can state, without measuring, which of steps 1–3 changes a *count* and
which changes only *time*, and why that distinction determines which instrument you reach
for.

### 3 — hard: production simulation — make the Topic 65 seed load fit the CI budget

**Scenario.** `orderflow` needs its Phase 7 dataset: ≥100k products, ≥1M orders, ≥5M order
lines, with realistic skew. The CI runner gives you a 1 GB heap and a Testcontainers
Postgres.

**Requirements:**

1. Implement the loader three ways: (a) stateful `EntityManager` with full batching,
   (b) `StatelessSession`, (c) `CopyManager`/`COPY`.
2. For each, record: statement counts, peak heap (from `-Xlog:gc*` or `jcmd GC.heap_info`),
   and the median of five cold runs. Report a distribution, not a single number.
3. The dataset must have realistic skew — a small number of hot products carrying a large
   share of the lines. Justify your distribution choice in two sentences.
4. Chunked transactions with resumability: killing the loader at 60% and restarting must not
   duplicate rows and must not start from zero.
5. **Write the decision memo.** Two paragraphs: which loader you would ship, what it costs
   you (what does `COPY` give up? what does `StatelessSession` give up?), and under what
   change of requirements you would switch. Name a specific requirement that would flip the
   decision.
6. Add the Phase 7 regression test asserting batching is active, and verify it fails when you
   set the entity back to `IDENTITY`.
7. **The honesty check.** Somewhere in your report, state one number you are *not* confident
   in and say why. If every number in a performance report is presented as certain, the
   report is wrong.

**Done when:** the memo would survive a senior engineer asking "why not just use `COPY` for
everything?" — which means you have an answer involving cascades, `@Version`, listeners, or
computed fields, not a preference.

---

## Interview questions

### Q1 — "We set `hibernate.jdbc.batch_size` and saw no improvement. What's wrong?"

**Mid-level answer:** "Maybe the property isn't being picked up, or you need to flush
periodically. Check that it's under `spring.jpa.properties.hibernate`."

**Senior answer:** "The first thing I'd check is the id generator, because
`GenerationType.IDENTITY` disables insert batching entirely and does so silently. The
persistence context is keyed by identifier, so Hibernate cannot register the entity without
an id — and with `IDENTITY` the id only exists once the row does. So `persist()` executes the
`INSERT` immediately to read the generated key back, one row at a time, and a JDBC batch
needs every parameter including the key bound before `addBatch()`. Hibernate detects the
incompatibility and turns batching off for that entity with no warning at any log level. The
fix is `SEQUENCE` with a `@SequenceGenerator` and an `allocationSize` — and critically, the
database sequence's `INCREMENT BY` has to equal that `allocationSize` or you get intermittent
duplicate-key errors from overlapping blocks. After that I'd check `order_inserts`, because
interleaving two entity types breaks the batch at every transition, and I'd verify the
property actually reached Hibernate by reading it back from
`EntityManagerFactory.getProperties()`. And I'd prove all of it with
`Statistics.getPrepareStatementCount()`, not a stopwatch, because a count is deterministic
and a single timing is not a measurement."

**What separates them:** going straight to the id generator, and being able to explain *why*
from the persistence context's structure rather than as a memorised rule. The `INCREMENT BY`
detail and the "prove it with a counter, not a timer" discipline are what make it a senior
answer rather than a well-read one.

**Interviewer's follow-up:** *"How would you migrate an existing `IDENTITY` table?"* —
Create the sequence with `setval` above the current max plus headroom, set `INCREMENT BY` to
the `allocationSize`, deploy the annotation change, and only then drop the column default —
in separate deploys, because during a rolling rollout both versions are writing.

---

### Q2 — "What is `allocationSize` and what does it cost you?"

**Mid-level answer:** "It's how many ids Hibernate fetches at once from the sequence, so you
make fewer `nextval` calls."

**Senior answer:** "It's a block reservation. With the pooled optimiser, one `nextval` returns
the top of a block and Hibernate uses every value below it down to `value - allocationSize +
1`, without going back to the database. That turns 50,000 `nextval` round trips into 1,000 at
`allocationSize` 50. The non-obvious part is that it is a **contract with the database**: the
sequence's `INCREMENT BY` must equal `allocationSize`, or two instances get overlapping
blocks and you get intermittent duplicate-key violations that only reproduce under
concurrency. So it lives in two files and has to change in both. What it costs is gaps: a
restart discards the unused remainder, so with `allocationSize` 500 you can lose up to 499
ids. That is fine unless someone is treating the surrogate key as a business count — 'we've
processed 1.2 million orders' — in which case you need a separate business sequence, not a
smaller allocation. The other thing worth saying is that switching from `IDENTITY` to
`SEQUENCE` with `allocationSize` 1 makes things *worse*, not better: you have replaced one
round trip per row with two. People make that change, measure nothing, and conclude batching
doesn't work."

**What separates them:** the two-files contract, the gap trade-off framed as a business
question, and the "worse before better" observation — which shows they have actually done the
migration rather than read about it.

**Interviewer's follow-up:** *"What if you can't change the DDL?"* — Set `allocationSize = 1`.
Correct and safe, costs a `nextval` per row, but inserts still batch, so it is strictly better
than `IDENTITY`. Know that it is a real fallback and know its price.

---

### Q3 — "You're inserting five million rows. Walk me through your approach."

**Mid-level answer:** "Use JDBC batching with a batch size of 50, flush and clear
periodically, and wrap it in a transaction."

**Senior answer:** "My first question is whether it should go through the ORM at all. For a
pure bulk load into a table I control, `COPY` via the Postgres driver's `CopyManager` beats
anything Hibernate can do by roughly an order of magnitude — no per-row statement, no
per-row parse, minimal protocol overhead. So I'd default to `COPY` and only move up the stack
if I need something it can't give me: cascades, `@Version`, entity listeners, computed
fields. If I do need entity semantics, the next step down is `StatelessSession` — no
persistence context, so memory is flat regardless of row count — accepting that I lose dirty
checking and cascades entirely. Only if I need the full stateful session do I get into
`SEQUENCE` with a pooled allocation, `batch_size`, `order_inserts`, and `flush()`/`clear()`
at the batch boundary. And independently of all of that, I'd chunk the transactions — one
transaction for five million rows pins a pool connection for the whole run, holds every lock
it takes, generates WAL that can't be checkpointed away, and fails all-or-nothing, so a
constraint violation at 90% costs you the whole load. Chunking gives me partial progress and
resumability. Then I'd verify with statement counts, not a stopwatch."

**What separates them:** starting with "should this use the ORM at all", and treating
transaction chunking as a separate concern from batching. Most candidates conflate the two.
The tool ladder — `COPY`, `StatelessSession`, stateful session — shows judgement rather than
recall.

**Interviewer's follow-up:** *"When would you not use `COPY`?"* — When the write path needs
cascades, optimistic locking, entity listeners, or computed columns the ORM populates — which
is most *application* write paths and almost no *seed* paths.

---

### Q4 — "Why does `order_inserts` matter, and what does it cost?"

**Mid-level answer:** "It groups inserts by table so they batch better."

**Senior answer:** "A JDBC batch belongs to one `PreparedStatement`, which means one SQL
string, which means one table. Hibernate's action queue preserves the order you called
`persist()` in, so the natural loop — persist an order, then its five lines, repeat — produces
an alternating queue, and every switch between the two tables forces `executeBatch()` on the
current one. Your effective batch sizes become 1 and 5 regardless of what `batch_size` says.
`order_inserts=true` sorts the queue by entity type before emission, so the batches fill.
`order_updates` does the same for updates and additionally sorts by id, which has a second
benefit I care about more than the throughput: consistent update ordering across concurrent
transactions reduces the deadlock rate, because everyone touches rows in the same order. The
cost is that Hibernate must hold and sort the entire action queue before emitting anything,
so it's memory plus a sort — which the `flush()`/`clear()` discipline bounds anyway. And the
one thing to verify rather than assume is that sorting by type doesn't reorder a foreign-key
dependency in a way your schema can't take; Hibernate orders types by dependency, but with a
cycle or deferred constraints I'd check."

**What separates them:** the "why is a batch per statement" mechanism, the deadlock benefit
from `order_updates`, and naming the FK-ordering caveat. The symptom "your batch size is
silently 1 and 5" is the concrete detail that shows real experience.

**Interviewer's follow-up:** *"How would you notice this in production?"* — Statement count
far above `rows / batch_size`, or datasource-proxy showing small alternating batch sizes.
Wall-clock alone would show a modest improvement over unbatched, which reads as "batching
just isn't very effective" and stops the investigation.

---

### Q5 — "You measured a 10× improvement with a timer around the loop. Convince me."

**Mid-level answer:** "I ran it several times and took an average, and the difference was
consistent."

**Senior answer:** "I wouldn't ask you to believe the timer, and I'd lead with the number
that isn't a timer: `Statistics.getPrepareStatementCount()` went from about 50,000 to about
1,000 for the same 50,000 entities, and `pg_stat_statements` shows the corresponding drop in
`calls`. Those are deterministic and reproducible — a count doesn't depend on JIT state,
connection warm-up, plan caching, page cache, or whether an autovacuum ran during the window.
The timing I'd report as a distribution across at least five cold runs with the first
discarded, and I'd say what varied: heap size, collector, Postgres version, whether the
container volume is a bind mount. I'd also be explicit that JMH is the wrong tool here even
though it's the right tool for JVM microbenchmarks — this measurement is dominated by the
database, the network and the OS, none of which JMH's warm-up and dead-code-elimination
machinery addresses. So: counts for whether the configuration took effect, an end-to-end
distribution for how long it takes, and no single number presented as a fact."

**What separates them:** separating "did my change take effect" from "how fast is it", and
knowing that JMH — the obvious rigorous answer — is wrong for this specific measurement.
Naming which environmental variables matter shows they have been burned by a number that did
not reproduce.

**Interviewer's follow-up:** *"What single number would you put in the ticket?"* — The
statement count ratio, because it is reproducible and it is the thing that actually changed.
The wall-clock number goes in with its distribution and its environment, or not at all.

---

## Mental model checkpoint

Answer these out loud, without looking above.

1. Explain, from the structure of the persistence context, why `GenerationType.IDENTITY`
   makes JDBC insert batching impossible. Do not say "Hibernate needs the id" — say *what for*.

2. Beyond throughput, name two other things that change when an entity moves from `IDENTITY`
   to `SEQUENCE`. One is about Topic 48's mental model; one is about Topic 52.

3. Why is `SEQUENCE` with `allocationSize = 1` *worse* than `IDENTITY` on round-trip count,
   and better on something else?

4. A JDBC batch and a multi-row `INSERT` are different things. State the difference and name
   the one connection parameter that converts one into the other.

5. `order_inserts` is off. You persist a parent and five children per iteration with
   `batch_size = 50`. What are your actual batch sizes, and why?

6. You set `spring.jpa.hibernate.jdbc.batch_size: 50` and nothing changed. Give the reason,
   and give the command that would have told you in ten seconds.

7. You want to report "batching made this 20× faster". Name the one instrument you would
   quote, the one you would not, and the reason the second one is untrustworthy here.

---

## Quick reference card

### The prerequisite

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "x_seq")
@SequenceGenerator(name = "x_seq", sequenceName = "x_seq", allocationSize = 50)
private Long id;
```

```sql
create sequence x_seq start with 1 increment by 50;   -- MUST equal allocationSize
```

**`GenerationType.IDENTITY` silently disables insert batching. Nothing else matters until
this is fixed.**

### The configuration

```yaml
spring:
  datasource:
    url: jdbc:postgresql://host:5432/db?reWriteBatchedInserts=true
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
        batch_versioned_data: true
        generate_statistics: true      # DIAGNOSTIC PROFILES ONLY
```

Prefix must be `spring.jpa.properties.hibernate.*`. Keys are verbatim, snake_case, **no
relaxed binding**.

### The loop

```java
for (int i = 0; i < n; i++) {
    em.persist(entity(i));
    if ((i + 1) % BATCH_SIZE == 0) { em.flush(); em.clear(); }   // flush THEN clear
}
em.flush(); em.clear();
```

Plus: **chunked transactions**, in a separate bean so the proxy applies (Topic 40).

### The tool ladder for bulk writes

```
COPY / CopyManager        fastest; no entity semantics at all
StatelessSession          fast; no persistence context, no cascade, no dirty checking
EntityManager + batching   full semantics; needs SEQUENCE + batch_size + order_inserts + flush/clear
```

### Instruments

| Question | Instrument |
|---|---|
| Did the property apply? | `EntityManagerFactory.getProperties()` |
| Is batching active? | `Statistics.getPrepareStatementCount()` vs `getEntityInsertCount()` |
| What was the batch size on the wire? | datasource-proxy / p6spy |
| What did Postgres execute? | `pg_stat_statements.calls` |
| Is memory growing? | `jcmd <pid> GC.class_histogram`, `-Xlog:gc*` |
| When did the insert fire? | `logging.level.org.hibernate.SQL=DEBUG` (timing only, not batch size) |

### Gotchas checklist

- `IDENTITY` → no insert batching, silently.
- `allocationSize` != sequence `INCREMENT BY` → intermittent duplicate keys.
- `allocationSize = 1` → a `nextval` round trip per row; worse than you started.
- `order_inserts` off + interleaved types → effective batch size is your run length.
- No `flush()`/`clear()` → linear memory growth → GC thrash → OOM.
- `clear()` before `flush()` → pending writes silently discarded.
- `clear()` detaches everything → `LazyInitializationException` on a parent you still hold.
- Wrong property prefix or spelling → silently ignored, no relaxed binding.
- One transaction for the whole load → connection pinned, WAL growth, all-or-nothing failure.
- `generate_statistics=true` left on in production → permanent unexplained slowdown.
- `reWriteBatchedInserts` is off by default and only affects inserts.
- H2 cannot test any of this.

---

## When would I use this at work?

**1. Any bulk import path, on day one of designing it.** Partner catalogue imports, CSV
uploads, backfills, data migrations, test-data seeding. The decision that matters is made
when the entity is written — the id generator — and changing it later is a multi-deploy
migration with a duplicate-key failure mode. Choosing `SEQUENCE` with a pooled allocation at
the start costs nothing and removes the problem permanently. This is why Topic 48 made that
choice for `orderflow` before this topic existed.

**2. Diagnosing a batch job that "got slower".** The shape is recognisable: the job's runtime
grows super-linearly with the dataset, or it starts OOMing at a size it used to handle. Three
checks in order — is the persistence context being cleared, is the id generator `SEQUENCE`,
did somebody add an entity type to the loop that broke `order_inserts`' grouping. All three
are answerable from `Statistics` counters and a heap histogram in under twenty minutes,
without a profiler.

**3. Pushing back on "let's just increase the batch size".** This topic's real value at work
is the ability to say "that property is currently doing nothing, and here is the count that
proves it" — then redirect the conversation to the id generator, or to whether the job should
use `COPY` at all. That is a very different conversation from tuning a number, and it is the
one that actually fixes the job. The counter is what makes it a conversation about evidence
rather than opinion.

---

## Connected topics

**Prerequisites:**

- **01 — Primitives and wrappers**: the persistence-context snapshot holds boxed values, which
  is a meaningful share of the per-entity memory in Trap 2.
- **32 — Dependency resolution and BOMs**: datasource-proxy is not in the Boot BOM, so you
  supply a version; the Postgres driver's is managed.
- **40 — Proxying**: chunked transactions require `@Transactional` on a **separate bean**, or
  the self-invocation trap silently makes the whole load one transaction.
- **47 — Spring Data JPA**: `save()` on a new entity delegates to `persist()`, which is where
  the `IDENTITY` round trip happens; the repository abstraction hides it completely.
- **48 — Persistence context**: the identifier-keyed map is *why* `IDENTITY` cannot defer, and
  the field-value snapshot is *why* memory grows linearly. This topic is unreadable without
  Topic 48.
- **50 — N+1**: the same instrument (a query/statement counter in an assertion) applied to
  the write path. The discipline transfers exactly.
- **52 — Locking**: `batch_versioned_data` is what lets `@Version` updates batch at all; and
  `order_updates`' id sorting reduces deadlocks between concurrent writers.

**This unlocks / is used by:**

- **54 / 55 — `@Transactional`**: transaction chunking is a Topic 54 decision; the connection
  held for the whole load is a Topic 55 failure; the two are independent of batching and both
  are required.
- **61 — Testcontainers**: this entire topic is untestable on H2 — different sequence
  semantics, different batch reporting, no `reWriteBatchedInserts`.
- **65 — GATE, load baseline**: the seed loader is a Phase 7 deliverable. Without batching it
  does not complete in a usable time, and without a completed dataset the gate does not open.
- **68 — Heap generations and TLABs**: Trap 2's failure is a promotion problem — an ever-growing
  live set is the worst input to a generational collector.
- **77 — JMH**: the full argument for why a `System.nanoTime()` loop is not a measurement —
  and, from this topic, why JMH is nonetheless the wrong tool for a database benchmark.
- **79 — Heap dumps**: finding `StatefulPersistenceContext` retaining most of the heap is the
  canonical worked example of Trap 2.
- **109 — HikariCP**: a single long transaction holds one connection for its entire duration;
  a chunked loader returns it between chunks.
- **113 — Kafka**: the outbox pattern's publisher is a bulk-insert path with exactly these
  characteristics.
- **116 — Idempotency**: a chunked, resumable loader must not duplicate rows on restart,
  which is the same unique-constraint-arbitrates-it argument at a different scale.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0, Hibernate via
`spring-boot-starter-data-jpa`, `jakarta.persistence.*`, Postgres. Batching behaviour,
sequence optimisers and the `IDENTITY` restriction are Hibernate semantics and have been
stable in kind across Hibernate 5, 6 and 7, but exact **defaults** — notably
`batch_versioned_data` and what `GenerationType.AUTO` resolves to — have changed between
major versions, and I have flagged both in the text. Read the effective values off
`EntityManagerFactory.getProperties()` and the generated DDL on your build rather than
trusting this document. Every number in this topic is an arithmetic estimate from stated
assumptions, never a captured measurement; the drill exists so you produce your own.*
